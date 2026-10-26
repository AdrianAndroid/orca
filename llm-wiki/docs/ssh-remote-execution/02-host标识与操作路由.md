# 第 2 章 · Host 标识与操作路由

> "这个操作该在哪台主机执行"——标识的语法、哨兵值、以及从渲染端到主进程的路由决策。
> 核心规则:**主机身份只在一个地方解析;解析不了的一律按远端处理(fail closed)**。引用为仓库相对路径。

---

## 2.1 ExecutionHostId:语法与禁区

核心类型(`src/shared/execution-host.ts:5-9`):

```ts
export const LOCAL_EXECUTION_HOST_ID = 'local'
export type ExecutionHostId = 'local' | `ssh:${string}` | `runtime:${string}`
```

三种主机:本机、本客户端直拨的 SSH target、配对 server 环境。URI 组件经 `encodeURIComponent` 编码(:61-74)。

### WSL 不是 ExecutionHostId

WSL 是 local 主机**内部**的一个维度:`wslDistro` 字段(:14-19),`isNativeLocalLocation` = `executionHostId === 'local' && wslDistro === null`。`wsl:<distro>` 只作为 UNC 路由键出现在 `src/shared/wsl-paths.ts:208`——**标识体系里多一种拼写,路由就多一倍判错的面**。

### `|` 禁令(:99-101)

> "'|' must stay out of a host id: `composeWorktreeHostIdentity` uses it as its delimiter and splits at the first one, so **an unencoded pipe would rebind an alias to a different host**."

分隔符字符一旦漏进标识,别名会被重绑到别的主机——不是解析错误,是**张冠李戴**。所以解析器直接拒绝。

### 哨兵:UNRESOLVED_OWNER_HOST_ID(:138-150)

```ts
export const UNRESOLVED_OWNER_HOST_ID = 'runtime:unresolved-owner'
```

所有者出版物冲突时的簿记哨兵:"no environment has this id and it must never be dialed"。路由专用解析器 `parseRoutableExecutionHostId` 把它拒掉——**哨兵可以存在,但永远不可拨号**。

---

## 2.2 同一行上的两种拼写:`connectionId` vs `executionHostId`

`getRepoSshConnectionId` 的长 Why(`execution-host.ts:216-236`)记录了路由层最阴险的一类 bug:

> "SSH ownership has two spellings on a repo row — the legacy `connectionId` field and the unified `executionHostId`. **Routing that reads the raw field answers 'local' for a row that only carries `ssh:<target>`, which runs a remote operation on the client.**"

直接读原始字段的行,在只有 `ssh:<target>` 拼写时会答出 "local"——于是远端操作在客户端本地执行。解法:**先解析主机,再从主机上读连接**。注释还区分了两种非 SSH 主机的不同处置:

- `local` 行上残存 `connectionId` = 自相矛盾(`contradictory`),不能据此发 SSH 连接;
- `runtime:<env>` 是另一台机器,它有自己的 SSH targets,可以按 `(environmentId, targetId)` 对寻址。

---

## 2.3 HostAuthority:端点 × 落点二元组

`ssh:t` 一个字符串表达不了 "server E 背后的 SSH target t"——它读起来像"本客户端自己拨的 t"。`src/shared/host-authority.ts:1-9`:

> "An `ExecutionHostId` alone cannot say 'SSH target t behind server E': `ssh:t` reads as a target this client dials itself. The pair can."

于是引入 `{endpoint: self|environment, at: local|ssh:*}` 二元组,`authorityKey` 保留所有既有拼写——**持久化键一个都不用改**。

嵌套键 `nested:<env>/<target>`(:32-40)是刻意设计的 fail closed:

> "In-memory only, never persisted. The nested spelling is **deliberately not an `ExecutionHostId`: every host-id parser rejects it, so a leaked key fails closed** instead of collapsing to `runtime:E`."

泄漏到标识解析器的嵌套键会被所有 parser 拒绝——宁可报错,不可静默坍缩成错误主机。

---

## 2.4 路由决策:谁把操作送到执行主机

| 决策点 | 位置 | 规则 |
|---|---|---|
| 渲染端 worktree 操作 | `src/renderer/src/lib/worktree-operation-route.ts` | `WorktreeOperationRoute{executionHostId, runtimeEnvironmentId}`;浮动 workspace 永远 local(:28-34) |
| 终端权威(三值) | `src/renderer/src/lib/workspace-terminal-host-authority.ts:21-44` | `live / unverifiable / none`;门控两个高危行为:初次播种(STA-4658)与睡眠 agent 恢复(STA-3500) |
| 主进程文件系统授权 | `src/main/ipc/remote-filesystem-owner.ts:1-25` | 他机持有的路径不得授权本地读;"Fails closed on a stamp the parser rejects" |
| RPC 入口 | `src/main/runtime/rpc/core.ts:58-73` | `RpcRequest.executionHost?: unknown`——phone 给的路由目标,**在读取处校验** |

终端权威那条注释值得全文(:36-41):

> "...forks a second `claude --resume` onto a transcript the host is still writing, **which no later evidence can undo**."

路由错误在"读"方向是显示错误,在"写"方向是**不可逆的重复执行**——这是路由必须 fail closed 的根本原因。

对 `ssh:` 行,权威看**连接状态**而非 host 快照(:47-54):relay 移除后(#27214)再没有任何东西回答 host workspace 快照,等快照就是等一封不会来的信。

---

## 2.5 被路由到执行主机的操作清单

| 操作域 | 路径 |
|---|---|
| 终端创建/订阅/输入/缩放/快照 | runtime RPC `terminal.*`(`src/main/runtime/rpc/methods/terminal/`) |
| 文件读/写/列目录 | `files.*` RPC(SFTP/exec 只用于部署与维护) |
| git worktree 创建、状态、fetch | 远端 runtime 的 git 服务(**不是**本地 exec over SSH pipe) |
| agent 会话 launch/send/stop/status | `agentSession.*`;远端 hook server 拥有状态(见 [agent 系列第 5 章](../agent-lifecycle/05-并发护栏与跨主机边界.md)) |
| 端口检测/转发 | `workspace-ports.v1` capability |
| AI vault 会话搜索 | `src/main/ipc/ai-vault-search-all-hosts.ts`(按 host 合并游标) |
| VS Code Remote-SSH 移交 | `src/main/ssh/vscode-ssh-authority.ts` |

搜索那条的注释(`src/main/ai-vault-search/session-search-source-presence.ts:16`)再次复诵主题:"loss of contact is never evidence of absence"——不可 stat 的源保行不删。

---

**上一章**:[01-概览与拓扑](./01-概览与拓扑.md) · **下一章**:[03-连接生命周期与重连](./03-连接生命周期与重连.md)
