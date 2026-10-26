# 第 2 章 · Hook 摄取管线:五条通道,一个 store

> 状态怎么进 store:本地 hook POST、SSH relay 转发、OSC 9999 字节流、structured session 投影、进程探针回写。
> 贯穿全章的哲学:**fail-open——hook 坏了绝不许卡住 agent**。引用为仓库相对路径。

---

## 2.0 通道全景

| # | 通道 | 入口 | 信任级别 |
|---|---|---|---|
| ① | 本地 hook HTTP POST | `server-lifecycle.ts` handleRequest | loopback + token |
| ② | SSH relay 转发 | `server-ingest-remote.ts:31` `ingestRemote` | **不可信**,入境再归一化 |
| ③ | OSC 9999(PTY 带内) | `server-ingest-terminal.ts:11` `ingestTerminalStatus` | 本机 PTY |
| ④ | structured session 投影 | `ingestStructuredStatus` 等 | 本机 host 进程 |
| ⑤ | 进程探针回写 | `reconcileEndedProcessForPaneKeys` | 本机 `/proc` 证据 |

五条通道写入同一个 store,所以"哪条通道来的"只在摄取层有意义;下游读者看到的是统一视图(第 1 章 §1.3)。

---

## 2.1 通道①:loopback HTTP listener

`src/main/agent-hooks/server/server-lifecycle.ts:28-213`,POST-only,监听 127.0.0.1。三个防守点,每个都有 Why 注释:

### 先鉴权,再读 body(:70-73)

```ts
// Why: authenticate before spending work reading an untrusted body.
if (req.headers['x-orca-agent-hook-token'] !== this.token) {
  res.writeHead(403)
  res.end()
```

顺序是刻意的:body 是不可信输入,**不验证身份就一个字节都不读**。

### slowloris 上限(:79-86)

```ts
// Why: bound request time so a stalled client can't hold a socket open (slowloris).
// Why: track our own destroy so the slowloris cap can't be misread as outside interference.
let destroyedBySlowlorisCap = false
req.setTimeout(HOOK_REQUEST_SLOWLORIS_MS, () => { ... })
```

自己 destroy 的要打标记——否则会把"我们掐的"误判成"对端 RST"(后者是 #11217 的传输干扰指纹,见 §2.5)。

### fail-open(:208 附近)

```ts
// Why: fail open — return success on malformed payloads so a broken hook never blocks the agent.
res.writeHead(204)
res.end()
```

畸形 payload 也回 204。这是整个子系统最重要的取向:**agent 的工作流是主,状态上报是仆**——一个坏掉的 hook 配置绝不能让用户的 agent 卡在等 HTTP 响应上。

### 端点发布:agent 怎么找到 server

`src/shared/agent-hook-listener/endpoint-publication.ts:30-50`:原子写 `endpoint.env` / `endpoint.cmd` 两个文件,字段 `ORCA_AGENT_HOOK_PORT/TOKEN/ENV/VERSION/TRANSPORT/OPENCODE_TUI`。agent CLI 的 hook 配置 source 这些文件拿到地址与令牌。**原子写**是为了让正在 source 的脚本永远读到完整文件。

### hook 安装与对账

hook 不是 agent 自带的,是 Orca 注入的:`managed-agent-hook-startup.ts` + `managed-agent-hook-registry.ts` 负责安装,并**对账**(reconcile)——设置变了就重写 hook 配置。orcad 侧同款:`src/main/orcad/orcad-managed-agent-hooks.ts:15-19` 的头注释记录了为什么必须有启动对账:

> "without it a freshly managed SSH host only got hooks when its hook settings changed, so its agents never reported status"

新纳管的 SSH 主机如果只在设置变更时写 hook,那它的 agent 永远沉默。

---

## 2.2 通道②:SSH relay 与信任边界

远端 agent 的 hook POST 打到 **远端自己的 hook server**(orcad 宿主,`orcad-entry.ts:195`),归一化后经 relay 以 `agent.hook` JSON-RPC notification 上行。线协议 invariants 写在 `src/shared/agent-hook-relay.ts:13-24`,是跨主机设计的缩影:

> - "The relay normalizes; Orca routes." —— relay 侧已经跑过 `normalizeHookPayload`
> - "Orca's `ingestRemote` re-runs the canonical payload normalizer at the SSH trust boundary before caching or persisting, **so relay skew or a buggy remote process cannot poison main-process state**"
> - "The wire `connectionId` is **always `null`**: a `connectionId` is Orca's local handle on an `ssh2` connection, not a wire identity. Orca stamps the real value on receive from `mux` identity inside `ingestRemote`."

三条 invariant 对应三条原则:

1. **双重归一化**:relay 归一化一次(早失败),Orca 入境再跑一遍(不信任)。远端版本漂移或有 bug 都毒不了主进程。
2. **线身份 ≠ 本地句柄**:`connectionId` 是本地 ssh2 连接的句柄,上线恒 null,由接收方盖章。旧版 peer 不会因为不认识这个字段而出错(wire 兼容,见第 5 章)。
3. **诊断透传**:`version`/`env` 原样转发,Orca 的 warn-once 协议诊断对远端 agent 照常工作。

---

## 2.3 通道③:OSC 9999——PTY 字节流里的带内状态

有些状态不走 HTTP:agent(或 shell wrapper)把 `\x1b]9999;...` 序列直接写进终端输出(`src/shared/agent-status-osc.ts:5`)。main 的 PTY 解析器截获后调 `ingestTerminalStatus`。

接线处的注释定死了它的地位(`src/main/agent-hooks/agent-hook-runtime-deps.ts:14-16`):

> "main's OSC parse is the only producer for a PTY agent, and this store is the only place `worktree.ps` and the mobile projection read from."

**唯一生产者 + 唯一读取处**——单写者原则让 `worktree ps` 和 mobile 不可能看到互相矛盾的两份真相(第 6 章)。

---

## 2.4 通道④:structured session——没有 hook 的 agent

native chat(structured session)没有 PTY、没有 hook 脚本,状态由 host 进程直接投影(`agent-hook-runtime-deps.ts:30-37`):

> "Why: structured chats have no hooks, so the host writes their projections into the same store."

注意 **into the same store**——不是没有 hook 就另起一套状态系统。sink 接口(`publish / forget / publishChildWork / readChildWork`)只是摄取层适配;落点与 hook agent 完全相同。这是"新生产者写入同一 store,读者只保留呈现策略"规则的直接体现。

---

## 2.5 spool:进程不在时的 hook 缓冲

agent 可能在 Orca 没跑的时候发 hook(比如 Orca 重启中)。hook 脚本把 POST 失败的事件落盘到 spool:`src/shared/agent-hook-spool.ts:14-16`,上限 **5MB / 1024 文件 / 7 天**,Orca 启动时 drain 重放(`server-status-hook-lifecycle.ts`)。

三个上限各防一种失控:字节数防磁盘,文件数防 inode,年龄防考古。spool 也是 fail-open 哲学的延伸——hook 脚本永远可以"先记下,回头再说",不阻塞 agent。

### 传输干扰检测(#11217)

企业 IDS 会中途 RST"可疑"连接。`src/shared/agent-hook-listener/request-body.ts:105` 的注释给出指纹:

> "Why (#11217): a body cut short of its own Content-Length is the fingerprint of an IDS resetting the connection mid-inspection."

但 listener 本身是 fail-open 的——被掐的 POST 表现为 204 返回,**唯一的症状是所有 runtime 的 agent 状态悄悄变 stale**。所以专门开了一条告警通道:`setTransportInterferenceListener`(`server-listeners.ts:82-91`),把"传输被截断"作为独立信号上报,而不是混进状态流。

---

## 2.6 归一化:provider 方言 → 统一词汇

各 agent CLI 的事件名、payload 形状都不一样。归一化分两层:

1. **wire 层**:`parseAgentStatusPayload` → `normalizeAgentStatusObject`(shared,relay 与 Orca 跑同一份代码);
2. **provider 层**:`src/shared/agent-hook-listener/providers/` 下的 per-provider normalizer,把 Claude/Codex/Grok/OpenCode 的事件映射到统一的行状态。

`version`/`env` 字段随包透传,`warnedVersions`/`warnedEnvs` 两个 warn-once 集合(`listener-state.ts:34-35`)保证协议升级提示只吵一次。

Windows 侧的 hook 启动器有一段"为什么命令长这样"的考古注释(`src/main/agent-hooks/windows-powershell-hook-launcher.ts:1-57`,涉及 #14815/#16003/#16576/#18875 等),是 Windows EDR/编码坑的浓缩史,推荐阅读原文。

---

**上一章**:[01-概览与架构](./01-概览与架构.md) · **下一章**:[03-状态词表与三态语义](./03-状态词表与三态语义.md)
