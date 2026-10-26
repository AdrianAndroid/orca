# 第 2 章 · Daemon 生命周期:启动、端点、死亡与重生

> terminal daemon 是一个"活得比 app 久"的进程,这带来一串独特的生命周期问题:
> 怎么启动、怎么证明"那个 socket 后面有个活 daemon"、什么时候该自杀、崩了怎么换。
> 全章最难的部分是**端点所有权协议**——23 个缺陷换来的一页规则。引用为仓库相对路径。

---

## 2.1 启动:四级 rung 与 10 秒 ready

`createOutOfProcessLauncher`(`daemon-out-of-process-launcher.ts:80`)→ `launchDaemonChild`(`daemon-launched-child.ts:37`):

- fork `daemon-entry.js`,**10 秒 ready 超时**(`DAEMON_CHILD_STARTUP_TIMEOUT_MS`,`daemon-launched-child.ts:16`);
- AppImage 有 extracted / mount / in-dir 三级 rung(:40-74)——每种打包形态一条路径,逐级降级;
- Linux 上经 `systemd-run` 进独立 scope(包装器让 daemon 在 `systemctl restart` 合并单元下存活,`daemon-launched-child.ts:78-81`;`loginctl enable-linger` 见 `daemon-user-manager-lifetime.ts:82`);
- Windows 走 relocated 副本,macOS 走 launchd 签名副本(第 7 章);
- 全部失败 → 60 秒硬上限后 fail-open 到主进程本地 PTY(第 1 章 §1.4 ③)。

daemon 入口契约写在用法注释里(`daemon-entry.ts:1-7`):`node daemon-entry.js --socket … --token …`,fork 场景经 IPC 上报 `{type:'ready'}`,SIGTERM 干净退出。`loginSessionWatch` 参数只给 GUI daemon——"GUI-spawned daemons only — **headless serve/SSH daemons must survive session loss**"(:43)。

---

## 2.2 端点所有权协议:23 个缺陷换来的一页规则

`src/main/daemon/AGENTS.md` 整篇是这个子系统浓度最高的设计文档,直接升格为本节。问题的根源:

> "`net.Server.close()` unlinks the pathname it bound with no ownership check, so a departing daemon deleted whichever socket then sat at the canonical path — **including a live replacement's**."

旧设计("launcher 回收死进程的名字")经七轮 review 产出 **23 个缺陷,全是同一个交错**:第三方在 T 时刻观察活性、T+1 时刻操作目录项。于是立了两条不变量:

> **"Only a daemon publishing itself onto the canonical endpoint may mutate that directory entry, and only by replacing an entry it has itself just proven dead."**
>
> **"No actor removes a name it did not create."**

### 发布协议(`daemon-endpoint-ownership.ts`)

```
bind 私有名 .p<hex>  →  尝试排他 link  ──EEXIST──►  连接探测 incumbent
       ▲                                            │ 只有 refused/missing 算死
       │                                            ▼
  验证没丢  ◄── 单次 rename 原子发布  ◄──  复查目录项没易手 → 再探一次
```

### 陷阱清单(每条都有事故史)

| 规则 | 为什么 |
|---|---|
| **"Never collapse 'can't tell' into 'dead.'"** | 只有 `connected` 证明占用,只有 `refused`/`missing` 证明死亡;timeout/`EPERM` 必须放弃——把它当死亡会删掉一台还在为全机终端服务的 endpoint |
| **`link` 先行,永不无条件 `rename`** | `rename` 会替换掉它找到的一切,包括一个健康 daemon;`link` 响亮地失败,强迫回答活性问题 |
| **`rename`,永不 `unlink`-then-`link`** | 后者在两次调用间让名字消失;"measured across a live handover it gapped on essentially every observation, where `rename` gapped on none in ~14,500 probes" |
| **不用 `birthtimeMs` 识别目录项** | Node 文档承认它有时是 ctime,无 birth time 的文件系统报 epoch,粒度常粗于要区分的事件;绕它打过三个补丁,产出三个新缺陷 |
| **不加 sweeper** | "判断别人的残留能不能删"正是这个设计退役掉的问题;上一个 sweeper 产出 5 个缺陷,包括删掉活 listener 唯一的路径名 |
| **关闭时绝不删 endpoint** | 离开的 daemon 留下死目录项,下一个发布者一次 rename 替换它 |

残留风险也被诚实记录:最后一次探测与 `rename` 是两次系统调用,POSIX 没有 rename-if-target-is-inode-X——但危害独立不可达,因为 daemon 绝不在自己不再持有的 endpoint 上建 session。

这套协议与 agent 子系统的三态语义(`live / unverifiable / exited`,见 [agent 系列第 3 章](../agent-lifecycle/03-状态词表与三态语义.md))是同一哲学在文件系统上的投影:**"说不清"必须保持"说不清"**。

---

## 2.3 保活与自杀:空闲退出 + 2 分钟收养窗口

daemon 没有主人的时候不该永远活着。`DaemonServer.isIdle()`(`daemon-server.ts:256-263`):

```ts
private isIdle(): boolean {
  if (this.admission.inFlight > 0 || this.host.hasLiveSessions()) {
    return false
  }
  if (this.endpoint.lost) {
    return true
  }
  return this.connections.transportCount === 0 && this.connections.size === 0
}
```

三个条件:没有在途 spawn、没有活 session、没有连接(或端点已丢)→ `beginIdleShutdown` → `onIdleShutdown` 打日志 `process.exit(0)`(`daemon-entry.ts:323-329`)。

新 daemon 有 **2 分钟初始收养窗口**(`INITIAL_ADOPTION_TIMEOUT_MS`,`daemon-server.ts:35`):fork 出来没有客户端来认领就自杀——防"启动竞态产生孤儿 daemon"。macOS 另有登录会话死亡监视(`macos-login-session-death-watch.ts`)。

---

## 2.4 重连与 respawn

### 连接层:generation 防旧事件拆新连接

`DaemonClient`(`client.ts:46-57`):

> "after a disconnect + reconnect (daemon respawn), a stale 'close' event from the old sockets can fire. Without a generation check, that event would tear down the fresh connection."

每次 `doConnect()` 递增 `connectionGeneration`,旧世代的 close 事件直接忽略。另有连接锁(`connectingPromise`)防并发 `ensureConnected()` 各自建连互相覆盖——注释记录的现场:"multiple concurrent spawn() calls from simultaneous pane mounts … overwriting sockets and triggering 'Connection lost'"。

### 重连后的状态重放

`DaemonPtyConnectionLifecycle.ensureConnected`(`daemon-pty-connection-lifecycle.ts:27-48`):重连后重放事件路由、checkpoint 计时器、颜色配置;**fresh connect**("no listener yet" 判定)还要 `resyncBackgroundedSessions()`——新 daemon 丢掉了旧 daemon 的后台 session 集合。

### respawn:崩溃后的换人

`DaemonPtyDaemonRecovery.withDaemonRetry`(`daemon-pty-daemon-recovery.ts:19-45`)在 `isDaemonGoneError` 时触发 `doRespawn`(:266-281:断开旧 client → 调 `respawnFn` → 持 adoption lease)。

写失败路径更讲究(`reconnectAfterWriteFailure:47-69`):

> "the dead endpoint took down every session on this daemon. **Signal all active panes now — while they are still in activeSessionIds**"

先向所有活跃 pane 广播 write-unavailable,让后台 pane 立即重挂载,而不是冻结着等超时。

### crash-loop 熔断:滑动窗口,不是永久跳闸

`DaemonRespawnThrottle`(`daemon-respawn-throttle.ts:32`):60 秒窗口内 5 次(`:27-28`)。注释说明为什么是滑动窗口(:14-17 区域):

> "Generous on purpose: a user manually restarting the daemon a few times, or a laptop waking to a stale socket, must never trip this. **Five failures inside a minute is a daemon that cannot start, not a daemon having a bad moment.**"

拒绝时返回 `{allowed: false, reason: 'crash_loop', retryAfterMs}`——可恢复的拒绝,不是拉黑。

### 健康驱动的替换(新 PTY 之前)

三个入口(`daemon-pty-daemon-recovery.ts:120/159/213`):macOS resolver 不健康、bundle 过期、TCC 断裂——都在**新建 PTY 之前**先换掉病 daemon,且都先核实 live session 数:有活 session 就保留 daemon 只发警告。**换新不能牺牲存量**,与 crash-loop 熔断的宽容一脉相承。

---

## 2.5 多代 daemon:router 与 history 移交

app 更新后,旧 daemon(旧协议版本)可能还持有活 session。`DaemonPtyRouter`(`daemon-pty-router.ts:25-28`)持有 current + 多个 legacy adapter,按 `sessionAdapters` 归属逐 session 路由——**session 死绝的 legacy adapter 才退役**。

history 跨代移交有门槛(`daemon-history-handoff.ts:4-14`):

```ts
return (
  keepHistory === true &&
  owner !== current &&
  owner.protocolVersion < HISTORY_SEED_TRANSFER_PROTOCOL_VERSION &&   // v30
  current.protocolVersion >= HISTORY_SEED_TRANSFER_PROTOCOL_VERSION
)
```

旧 adapter 太老(不会 seed 传输)而新的会,才移交 history 种子;两个都会或都不会,各留各的。seed 传输本身分块(512K code units/块,1MB 内联阈值,surrogate 安全,`terminal-history-seed-chunks.ts:3-4`)——细节见第 5 章。

---

**上一章**:[01-概览与架构](./01-概览与架构.md) · **下一章**:[03-PTY 层](./03-pty层.md)
