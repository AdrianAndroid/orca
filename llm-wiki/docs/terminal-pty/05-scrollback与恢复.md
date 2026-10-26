# 第 5 章 · Scrollback 与恢复:两套互补的持久化

> 终端的内容有两条生存线:daemon 的增量 history(让 session 活过 app/daemon 重启)与
> workspace-session 快照(让 tab/pane 布局恢复时有东西可画)。
> 本章讲两套机制的格式、写盘时机、恢复管线与防腐蚀设计。引用为仓库相对路径。

---

## 5.0 两套机制对照

| | daemon history | workspace-session 快照 |
|---|---|---|
| 目的 | session 跨 app/daemon **重启**复活 | tab/pane 布局恢复时**画面**复原 |
| 内容 | checkpoint(全量快照)+ output.log(增量帧) | SerializeAddon 序列化的单 buffer |
| 写盘方 | daemon(主进程 adapter 调度) | renderer(shutdown/park 时) |
| 存储 | `userData/terminal-history/<enc(sessionId)>/` | `userData/terminal-scrollback/<ref>.bin` |
| 上限 | checkpoint 200MB / log 5MB | 存 5MB / replay 512KB |
| 恢复路径 | cold restore(checkpoint+log replay) | pane mount 时写回 xterm |

---

## 5.1 daemon history:三件套与帧格式

### 目录与文件

`userData/terminal-history/<encodeURIComponent(sessionId)>/`(`src/main/terminal-history-paths.ts:5`;`daemon/history-paths.ts:1-7` 解释 URI 编码——session id 内含 worktree identity 的 `:`/`/`)。WSL 另树 `terminal-history-wsl/<distro>`。三件套(`daemon/terminal-history-session-files.ts:9-16`):

- `checkpoint.json`:全量快照,上限 **200MB**(`terminal-history-file-limits.ts:4`);
- `output.log`:增量输出日志,上限 **5MB**(:2);
- `meta.json`:cwd/cols/rows/startedAt/endedAt/exitCode,上限 64KB。

### output.log 帧格式(`daemon/terminal-history-log.ts:3-16`)

```
header = magic 'OCKL' + u8 formatVersion + u32le generation
frame  = u8 kind + u32le payloadLength + payload
  0x01 batch  — u32le seq(每次 take 追加一批)
  0x02 output — utf8 bytes
  0x03 resize — u16le cols + u16le rows
  0x04 clear  — 空
```

**为什么带帧而不是裸字节流**(:13-16):

> "Why framing instead of raw bytes: **a crash can tear the final append**. Length prefixes make the torn tail detectable so restore truncates at the last complete frame instead of replaying half an escape sequence ('reading a corrupt checkpoint is worse than reading a slightly stale one')."

两条防腐蚀规则:

- **撕裂检测**:文件结尾半帧 → `truncatedTail`,只重放完整前缀(:33-38);
- **洞检测**:batch seq 不连续说明 take 与 append 之间崩过,字节流有洞,**整份日志作废**回退 checkpoint-only(:103-107)——"replaying it would corrupt the restored terminal"。有洞的日志比旧快照更危险。

### checkpoint 内容(`daemon/terminal-snapshot.ts:5-25`)

`TerminalSnapshot`:`snapshotAnsi`(当前画面)、`scrollbackAnsi`(alt-screen 时的 normal buffer,分开存)、`rehydrateSequences`(模式重放序列)、`pendingEscapeTailAnsi`(解析器尾巴,**必须最后恢复**)、`oscLinks`、cwd/modes/cols/rows/scrollbackLines、`lastTitle`、`outputSequence`、`terminalOwner`(快照边界的 shell 生命周期证据)。

### 5MB 触发的全量重置(`terminal-history-session-writer.ts:19-21` 区域)

> "Why 5MB: bounds cold-restore replay time and per-session disk; hitting the cap triggers one checkpoint that resets the log."

日志到顶不做轮转截断,而是写一次全量 checkpoint 把日志清零——**恢复时长与磁盘占用同时有界**。

---

## 5.2 写盘时机:dirty 门控的 5 秒心跳

主进程 adapter 侧,`DaemonPtyCheckpointScheduler`:

- 每个 data 事件 `markSessionDirty`;
- **dirty-gate 定时器**(`daemon-pty-checkpoint-scheduler.ts:29-31`):"a permanent 5s interval woke the main process for idle terminals with nothing to write"——没脏就不醒;
- **防重叠**(:33-36):"concurrent tmp-file writes can lose a rename and disable future history writes"——在途 checkpoint 没完成就把下一轮重新排期;
- 版本过滤(:58-60):太老的 daemon 不支持的不发。

写盘本体 `DaemonPtyCheckpointPersistence.writeSessionCheckpoint`(`daemon-pty-checkpoint-persistence.ts:8-53`),三种情形都有 Why:

1. **take-with-snapshot**(:26-28):全量 checkpoint 在同一回合清掉 pending 记录——"so a warm reattach won't re-append records the checkpoint already contains (**double-replay on cold restore**)";
2. **deferred**(:8-9):cooldown 内的全量请求推迟、session 保持 dirty——"keeps the on-disk log a consistent (stale) prefix **instead of punching a hole**";
3. **overflow**(:45-47):take 溢出 = 丢记录 = 日志有洞 → 只有全量快照能重新锚定。

teardown/final checkpoint 另有 `onFinalCheckpoint` 钩子与 deadline(daemon 关机时最后写一次)。

---

## 5.3 cold restore:把终端从磁盘复活

### 快照来源:HeadlessEmulator

daemon 内为每个 PTY 维护一个 `@xterm/headless` + SerializeAddon 的屏幕模型(`headless-emulator.ts:1-30`):绝对光标序列化、partial escape tail 推进。所有 checkpoint 由它产出。

### 恢复管线

`HistoryReader` 读 checkpoint+log → `ColdRestoreInfo` → `coldRestorePayloadFromInfo`(`cold-restore-payload-cache.ts:16-35`):

- **alt-screen 时不加 rehydrate**(:20-21):"its `?1049h` would paint the TUI frame on the alt buffer, which the appended ground's `?1049l` then discards"——alt 帧画了也白画,直接用 scrollback/snapshot;
- 末尾追加 `PROCESS_BOUNDARY_GROUND`(:29):把终端模式归到已知基态,"main's mirror seeds from this payload, not the renderer's reset";
- 主进程缓存上限 16MB(:39)。

### replay 写入的两个保护(`cold-restore-replay-writer.ts`)

- **每 turn 64K 字符 / 1024 操作**(:1-4):恢复几百 MB 的 scrollback 不能卡死事件循环;
- **绝不切断 UTF-16 surrogate pair**(:17-31):"xterm must receive UTF-16 surrogate pairs together when a replay slice lands between them"——切片落在 pair 中间就退一格(或进一格,offset+1 的边界情形也处理了)。

renderer 侧 replay 后施加 `POST_REPLAY_REATTACH_RESET`(链路见 `repro-7329-remote-snapshot-corruption.test.ts:4-13`——远端快照腐蚀事故的回归测试)。

### 已知陷阱:mouse tracking 复活(#12101)

被杀的 TUI 没来得及发 DECRST,checkpoint 会把 latch 住的 mouseTracking 模式复活进裸 shell——鼠标事件全被吞。修复在 `terminal-mode-rehydrate-sequences.ts`/`terminal-modes.ts`,事故记录在 repro 测试头(`repro-12101-….test.ts:11-18`)。**模式恢复是快照最危险的部分**,因为模式是"看不见的界面状态"。

---

## 5.4 跨代 seed 转移与崩溃保护

- **跨代移交**:v30 门槛见第 2 章 §2.5;传输分块 512K code units/块、1MB 内联阈值、surrogate 安全迭代器(`terminal-history-seed-chunks.ts:3-4,20+`);
- **quarantine**:读不动的 checkpoint 挪进 `.recovery-quarantine/` 并留 `.unreadable-recovery` marker(`terminal-history-recovery-quarantine.ts:7-8`)——损坏不等于删除;
- **权限收紧**:目录 0700/文件 0600(`daemon-private-file-modes.ts`、补修 `terminal-history-permission-repair.ts`);
- **两阶段删除**:"rename live history out of the way first so a quit mid-rm still leaves a durable tombstone GC can finish"(`terminal-history-paths.ts:8-9`,`.pending-delete` + `terminal-history-session-tombstone.ts`);
- **保留上限**:可恢复 session 最多 10k,min-heap 保留最新(`terminal-history-restorable-retention.ts:1`)。

---

## 5.5 workspace-session 快照:第二套机制

### ref 命名与落盘

`src/main/terminal-scrollback-snapshots.ts`:ref = `v1-<sha256(tabId\0leafId)[:32]>`(:45-48),tmp+rename 写、目录 0700/文件 0600(:144-158)。存上限 5MB、读/replay 上限 512KB(`src/shared/terminal-scrollback-limits.ts:1-3`)。

尾部截断对齐 UTF-8 continuation byte(:97-107):截断点扫到非 continuation 字节为止——**不制造半个字符**(与 surrogate pair 保护是同族心思,一个在 UTF-16 层一个在 UTF-8 层)。

### 双 home 与冲突裁决

- `terminalLayoutsByTabId[tab].buffersByLeafId`:共享,随远端投影;
- `localOnlyScrollbackByTabId`:本机私有。

读取唯一入口 `resolveLeafScrollbackBuffers`(`src/renderer/src/components/terminal-pane/leaf-scrollback-resolution.ts:20-22`):

> "Why local-only wins a conflict: it is written by the ordinary park, and every shared capture clears it."

本机 park 写的最新,共享捕获每次都会清掉它——所以冲突时 local-only 赢是安全的。

### 捕获与恢复

- **捕获**:shutdown/force-park 时 renderer 用 SerializeAddon 序列化(`terminal-shutdown-layout-capture.ts:1-60`;超限的 buffer 用二分/插值探测逼近上限,`MAX_SCROLLBACK_FIT_PROBES = 4`);
- **恢复**:IPC `session:read-terminal-scrollback-sync`(`src/main/ipc/session.ts:72-75`)→ renderer 在 pane mount/layout replay 时写回 xterm(`pane-lifecycle.ts:88-98` replay 链);
- **迁移**:旧格式(session JSON 内嵌 buffer)落盘成 ref(`terminal-scrollback-snapshots.ts:245-292` 及异步版);
- **GC**:refs 收集 `collectTerminalScrollbackSnapshotRefs:235-243` + workspace 快照剪枝。

---

## 5.6 title 与 tab 恢复

- 快照带 `lastTitle`(`terminal-snapshot.ts:20`),cold-restore payload 只 seed title 记录;
- 运行期 title 跟踪:`runtime/terminal/output/terminal-title-tracker.ts`——**手工 title 是权威**(:25-33),合成 agent title 帧不走 `pty:data`(:36-39);
- pane↔PTY 归属落盘:`persistence/loading-store/pty-binding-*.ts`,配 10+ 个 reconciliation 测试;
- tab/layout 拓扑:`persistence/terminal-topology/` + renderer `terminal-pane-layout-restore.ts`;
- 孤儿收养:app 重启后从 daemon inventory 收养无 pane 的 PTY(`runtime/terminal/adoption/adopt-terminal-orphans-from-inventory.ts:28+`,含 worktree 竞争校验)。

---

**上一章**:[04-shell wrapper 体系](./04-shell-wrapper体系.md) · **下一章**:[06-终端与 agent 交互](./06-终端与agent交互.md)
