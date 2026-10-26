# 第 3 章 · PTY 层:node-pty、job object 与 kill 语义

> 真正创建和杀死进程的地方。核心是 `node-pty@1.1.0` 加一份 1466 行的 Orca 补丁
> (`config/patches/node-pty@1.1.0.patch`)——**这份补丁就是 Orca 的 PTY 语义本体**,本章逐条拆解。
> Windows 是主战场;POSIX 的对偶机制在章末。引用为仓库相对路径。

---

## 3.0 补丁总览

补丁新增四个原生导出(patch:1235-1238):

```ts
terminateJob            // 杀整个 job(=整个终端进程树)
listJobProcessIds       // 枚举 job 成员(内核视角,无 pid 复用问题)
assignCurrentProcessToJob  // 把 daemon 自己放进 kill-on-close job
setActiveProcessLimit   // 每终端进程数上限
```

动态加载(`pty-subprocess/native-pty-spawn.ts:10-12`,`import('node-pty')`);Windows 强制 `useConptyDll: true`——"Why: bundled ConPTY has the wrap-marker behavior xterm expects"(:47-57)。

---

## 3.1 每-PTY job:让显式 teardown 精确

### 问题:pid 走查不可信

`src/main/windows/windows-pty-job.ts:5-26` 给出动机(issue #9045/#10475/#10087):按 pid 遍历进程树杀终端后代,撞在 **pid 复用**上——杀错进程,或漏杀 reparent 过的后代,孤儿 agent 树把 worktree 目录占着不放。**job object 用内核维护的成员关系替代用户态推断**。

### CREATE_SUSPENDED:先入 job,再让跑(patch:882-884)

```c
// Orca: CREATE_SUSPENDED so the shell is inside its job before it can
// spawn anything. Assigning after the fact leaves a window in which a
// fast child escapes the job and outlives the pane.
EXTENDED_STARTUPINFO_PRESENT | CREATE_UNICODE_ENVIRONMENT | CREATE_SUSPENDED
```

顺序:`CreateProcess`(挂起)→ `CreateJobObjectW` → `AssignProcessToJobObject` → `ResumeThread`。先跑再入队的老写法有一个竞态窗口:**快子进程会在入 job 之前逃走,活得比 pane 久**。

### 刻意不加 KILL_ON_JOB_CLOSE(patch:899-903)

```c
// Deliberately WITHOUT JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE. Measured on
// Windows 11: with that flag, closing the handle when the shell exits also
// kills whatever the user left running, so typing `exit` in a pane reaped a
// `start /b` server that used to survive. This job exists to make an
// EXPLICIT teardown exact, not to redefine what a clean exit means.
```

这是全补丁最精髓的一条注释。直觉上"job 关闭即杀全家"最省事,但实测发现:用户在 pane 里 `exit` 会连带杀死他 `start /b` 起的后台服务器。**job 的用途被精确限定为"显式 teardown 时杀得干净",不改变"正常退出"的语义**——干净退出留下什么是用户的事。

### breakaway 策略:对 MSYS 全家桶收紧(patch:811-822, 907-914)

```c
// Cygwin and MSYS request breakaway for every child whenever the job allows it,
// so their shells need one that does not.
```

检测方式是看 shell exe 旁边有没有 `msys-2.0.dll`/`cygwin1.dll`(含 `..\usr\bin\` 布局,Git for Windows 的 bash.exe 在 `bin\`):

- **MSYS/Cygwin shell** → job **不带** `JOB_OBJECT_LIMIT_BREAKAWAY_OK`(严格成员制),否则它们的每个子进程都自动 breakaway,job 形同虚设——MSYS/Git Bash pane 的孩子就这样逃出 per-PTY job;
- **原生前缀 shell**(powershell/cmd)→ 保留显式 breakaway,安装程序/更新程序需要它;
- 代价也被诚实记录:"Explicit breakaway requests inside such a pane are consequently denied; ordinary backgrounding and clean shell exit remain supported."

> AGENTS.md 特别提醒:不含此修复的旧构建 `conpty.node` 能通过所有现存门检——改这块必须重建并实测,不能靠 CI 绿灯。

### 宿主 job:daemon 自杀式保险(patch:1166-1203)

per-PTY job 不能带 KILL_ON_JOB_CLOSE(§3.1),那 daemon 自己崩了怎么办?把 **daemon 进程自己**放进一个 kill-on-close job:

> "a per-pty job cannot carry KILL_ON_JOB_CLOSE, because its handle is released when the shell exits... **This job's handle is released only when the process itself dies**, so it reaps a crashed host without changing what a clean shell exit means. Children inherit job membership, so every pty the caller later spawns is covered without further work, and the per-pty jobs simply nest inside this one."

宿主 job 带 BREAKAWAY_OK(理由同前),且必须在首个 Windows PTY 之前建好——"children inherit job membership, so the host job must exist before the first Windows PTY"(`native-pty-spawn.ts:44-47`)。两层 job 嵌套:**per-PTY 管"显式杀一个终端",宿主管"daemon 死了全带走"**。

### 进程数上限(patch:1215-1222)

`setActiveProcessLimit` → `JOB_OBJECT_LIMIT_ACTIVE_PROCESS`,fork 超限只在**该终端的树内**失败。默认 256(`host-terminal-ledger.ts:21-23`):

> "Why 256: half of macOS's machine-wide pty limit (kern.tty.ptmx_max = 511), so other terminal programs and the ptys agents open inside Orca terminals keep room; five times the deleted SSH relay's 50, which users with many worktrees did reach."

每个数字都有出处:511 的一半(留余量),旧 relay 上限 50 的 5 倍(用户真打到过)。

---

## 3.2 kill 升级链

`pty-subprocess/subprocess-handle.ts:137-172`:

| 方法 | 语义 |
|---|---|
| `kill()` | 走 node-pty 常规 kill;**被拒不是死亡证明**——"A rejected native kill is not proof of exit; keep the wrapper live for a retry"(:144-147) |
| `forceKill()` | win32 且已发过 nodePtyKill → 升级走 `terminatePtyJob`(整个 job 终结,:155 "Escalate a ConPTY kill through the job without double-closing node-pty's shell handle");POSIX → `forceKillPosixPtyProcessGroups` |
| `terminateOwnedTree()` | 直接 job 终结 |

Windows 进程树杀的独立实现:`src/main/windows-process-tree-kill.ts:23` = `taskkill /pid /T /F`,**5 秒超时**(:38,#10004——taskkill 自己会卡死),并经 `admitSelfInitiatedTreeKill` 防误杀自家 Chromium 树。

POSIX 对偶:`src/main/pty-descendant-termination.ts`(进程表快照 + pgid 扫描,`DESCENDANT_KILL_GRACE_MS = 2000`)、`src/main/pty/posix-pty-process-groups.ts`。SIGWINCH 只发前台进程组(`subprocess-handle.ts:186-196`)——resize 不该惊动后台进程组。

---

## 3.3 ConPTY 预热:把 2.7 秒预付掉

`src/main/daemon/windows-conpty-warmup.ts:15-21`:

> "Pays the one-time cost of the first ConPTY spawn (conpty native module load, bundled conpty.dll + OpenConsole.exe first launch, **Defender scans** of those binaries) at daemon boot instead of on the user's first terminal. Measured **~2.7s** on a Windows dev profile for the first spawn vs **~70ms** after."

首个 ConPTY spawn 的成本大头是 Defender 实时扫描新出现的二进制——这是 Windows 特有的"第一个请求付全款"现象。`setImmediate` 保证 ready 握手优先于预热(:26-28),预热子进程挂进宿主 job(:31)。

---

## 3.4 shell 解析:`--shell` vs `--command`

CLI 文档即规格(`src/cli/specs/core.ts:223`):

> "--shell picks the shell the terminal **IS** on a Windows host (cmd.exe, powershell.exe, pwsh.exe, wsl.exe, bash.exe, git-bash); **--command is typed into whatever shell the host started**, so `--command cmd.exe` leaves a cmd running INSIDE the default shell and exiting it drops back to that shell."

落码(`pty-subprocess/shell-launch-plan.ts`):

- **shellOverride(= --shell)**:`:62`——`shellPath = resolvedWslContext ? 'wsl.exe' : opts.shellOverride || resolvePtyShellPath(env)`。`resolvePtyShellPath`(`daemon/shell-ready.ts:99-104`):Windows 读 `ORCA_TERMINAL_WINDOWS_SHELL` 否则 powershell.exe;POSIX 读 `env.SHELL` 否则 /bin/zsh;
- **command(= --command)**:`:199-226`——wrapper 启动 shell,命令经 startup ingress 在 shell ready 后"键入"(Codex 右键粘贴一次修复 #29647 就是这条路径上的事);cmd.exe 的 startup command 不能走 `/K`,因为 node-pty 的 argv 转义会弄坏引号(#7978),改走 stdin(`powershell-osc133-bootstrap.ts:35-38`);
- Windows shell 解析链(`:74-188`):git-bash 路径解析、powershell/pwsh 家族探测、PowerShell→cmd fallback 链(`native-pty-spawn.ts:21` "Walks the Windows PowerShell -> cmd.exe fallback chain when ConPTY rejects the primary shell")、WSL distro 改写、`CHERE_INVOKING`;
- Unix 侧:`resolveUnixShellPath` 不可用则回退并改写 `env.SHELL`(:191-198);无 command/agent 时透传用户 `terminalShellArgs`,兜底 `['-l']`(:208-225);
- **--shell 拒绝策略**(specs/core.ts:224 区域):主机能不了就**拒绝创建**而不是静默退化成默认 shell——又一例 fail-closed。

spawn 预检(`pty-subprocess/spawn-preflight.ts`)提前验证 node-pty helper 与 cwd 存在,给出可读的错误文案。

---

## 3.5 Linux/macOS 的落位

- **Linux**:经 `systemd-run` 进每终端 scope(`wrapPtySpawnInTerminalScope`,`native-pty-spawn.ts:41-43`;`daemon-cgroup-scope.ts`),与 daemon 自己的 scope 并列;
- **macOS**:`wrapShellSpawnForMacosTccAttribution`(`src/main/providers/macos-tcc-login-shell.ts`)包装 spawn,spawn 策略回报 `onMacosTccSpawnStrategy`——TCC 归因的完整故事见第 7 章。

---

**上一章**:[02-daemon 生命周期](./02-daemon生命周期.md) · **下一章**:[04-shell wrapper 体系](./04-shell-wrapper体系.md)
