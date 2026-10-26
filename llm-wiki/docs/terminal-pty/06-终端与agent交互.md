# 第 6 章 · 终端与 Agent 的交互:观测一个黑盒进程

> agent CLI 是跑在 PTY 里的黑盒。Orca 不修改它们,只靠四层信号观测:
> ready 门、字节流 transcript、OSC 序列、进程表。本章讲观测面与归属关系。
> agent 状态的 store 模型见 [agent 系列](../agent-lifecycle/01-概览与架构.md),本章只讲终端侧的生产者。
> 引用为仓库相对路径。

---

## 6.1 readiness:四个信号,一道门

daemon 什么时候才能往 shell 里"键入"startup command?四路独立信号:

| # | 信号 | 机制 | 出处 |
|---|---|---|---|
| 1 | **OSC 777 marker** | wrapper 在首个 prompt 发 `\x1b]777;orca-shell-ready` | `src/main/shell-ready-marker-scanner.ts:1` |
| 2 | **line-editor 输出** | `?2004h`(bracketed paste)/ `?1034h` / `133;A` 任一出现 | `src/main/line-editor-ready-output-scanner.ts:1` |
| 3 | **进程表 probe** | PTY slave line discipline + 进程状态,settle 50ms、最多 4 次 | `src/main/shell-prompt-readiness-probe.ts:19-40` |
| 4 | **startup identity marker** | wrapper 烧入的身份序列 | `src/main/shell-startup-identity-scanner.ts` |

全部汇入一道门(`src/main/daemon/session-shell-ready-barrier.ts:40-41`):

> "**The startup gate every byte of PTY output passes through**: it strips the shell-ready marker, holds stdin until the shell can accept it, and owns DA1 authority for as long as the gate is closed."

- marker 被剥离——用户终端上看不到它;
- 门关上时 stdin 被 hold——startup command 不会丢进还没准备好的 shell;
- 超时 15s(`SHELL_READY_TIMEOUT_MS`,:22),**Codex 特例 300ms**(`CODEX_SHELL_READY_TIMEOUT_MS`,:23)——Codex 自己接管输入很快,等满 15s 是纯粹的用户可感知延迟;
- daemon 无 console,readiness 事件只写 NDJSON 文件日志(:29-31 区域):"the daemon runs detached with stdio 'ignore', so console output goes nowhere — the only durable sink is the daemon's NDJSON file log"。

fish 不走这道门的原因见第 4 章 §4.6(STA-3417)。

---

## 6.2 transcript:字节流里的事实

主进程为每个 PTY 维护规范化文本 tail(`src/main/runtime/terminal/output/pty-output-ingestion.ts:120-160`):`tailTranscriptBuffer`(normalized 文本、partial line、redraw cursor)。派生事实:

- **OSC title / agent-status 提取**:经共享 per-PTY title tracker,按字节序应用(:128-133,#7880/#7852 语义由 tracker 保留);
- **命令码检测、OSC7 cwd 提取**:`terminal-derived-agent-facts.ts`(`extractLastOsc7CwdForPty:35-60`);
- **agent 状态 OSC**:OSC 9999(`src/shared/agent-status-osc.ts:5`)——PTY agent 状态的唯一生产者(见 agent 系列第 2 章 §2.3)。

> AGENTS.md 的规则在此落地:任何"读 agent CLI 画面"的规则(readiness、blocked 提示、idle)**必须基于捕获的 transcript,不能基于记忆里的屏幕**——采集工具 `config/scripts/capture-agent-pty-transcript.mjs` 保留转义与换行原貌。

### headless 镜像的水合顺序

renderer mount 后首字节触发 `maybeHydrateHeadlessFromRenderer`(`pty-output-ingestion.ts:120-127` 区域):注释强调 **MUST run before** `trackHeadlessTerminalData`——eager-state 模式(set state + writeChain head = seedPromise)必须先就位,否则 live 字节的 chain link 排队后,迟到的 seed-resolve 会把它覆盖,"dropping the live byte"。又一个"await 后世界变了"的排序防护。

---

## 6.3 title:手工权威,合成帧旁路

`src/main/runtime/terminal/output/terminal-title-tracker.ts`:

- **手工 title 是权威**(:25-33):"a manual title is authoritative until explicitly cleared with null"——用户改的标题不被程序更新覆盖;
- **合成 agent title 帧不走 `pty:data`**(:36-39):"synthetic agent title frames no longer ride pty:data, so neither renderer xterm nor the headless emulator observes them"——agent 状态合成的标题(如 spinner 文案)是独立通道,不污染终端字节流,快照里也就不会固化一个瞬间的 spinner 文本。

配套:`synthetic-title-frame-routing.ts`、`synthetic-title-spinner.ts`。title 的快照恢复见第 5 章 §5.6。

---

## 6.4 归属:终端是谁的

### spawn 时的身份(`src/main/providers/pty-provider-contract.ts`)

```ts
/** Orca worktree identity. When present, the local provider scopes shell
 *  history to this worktree so ArrowUp only surfaces local commands. */
worktreeId?: string
/** Stable terminal pane identity. Remote providers use this as PTY metadata
 *  even when it must not be exported into the spawned shell environment. */
paneKey?: string
```

- `worktreeId`(:65-67):**shell history 按 worktree 隔离**——在 worktree A 按 ↑ 不会翻出 worktree B 的命令(`src/main/terminal-history.ts`、`fish-history-session.ts`、`zsh-scoped-histfile`);
- `paneKey`(:69-71):稳定 pane 身份;注意注释里的分寸——remote provider 拿它当元数据,但**不许泄进 spawned shell 的环境变量**。

### agent 识别与启动

- `recognizeAgentProcessFromCommandLine(opts.command)`(`shell-launch-plan.ts:66`;`src/shared/agent-process-recognition.ts`):从命令行认出"这是个 agent 启动";
- agent cwd 安全断言 `assertSafeAgentStartupCwd`(:68-70);
- agent session 认领:`terminal-host-agent-session-claim.ts`(`ClaimedAgentPtyOwnerRegistry`)+ `terminal-host-agent-session-generations.ts`——认领与代际分离,防旧代迟到事件(与 agent 系列的 fence 三件套同族);
- spawn 准入与台账:`terminal-spawn-gate.ts`、`host-terminal-ledger.ts`(256 上限见第 3 章 §3.1)。

### 孤儿收养

app 重启后,pane 还没恢复,daemon 里已有活 PTY:`runtime/terminal/adoption/adopt-terminal-orphans-from-inventory.ts:28+` 从 daemon inventory 收养它们,含 worktree 竞争校验(:41-58)——另一个 worktree 也在等这个终端时,归属要有裁判。

---

## 6.5 与 agent 生命周期子系统的接口

终端侧产出的三类事实,分别流向 agent 状态 store 的不同通道:

| 终端侧事实 | 通道 | agent 系列章节 |
|---|---|---|
| OSC 9999 状态序列 | `ingestTerminalStatus`(通道③) | 第 2 章 §2.3 |
| PTY 存活/退出 | 探针与扫尾的证据源 | 第 4 章 §4.3 |
| paneKey / launchToken | 状态行作用域与 fence | 第 5 章 §5.2 |

**边界**:终端子系统只负责"忠实地产生证据",判定(working/blocked/exited)全在 agent hook server。两个子系统的词汇表在边界上对齐——daemon liveness 报告直接用 `live / unverifiable / exited`(`src/main/daemon/daemon-liveness-report.ts:2-7`,第 7 章)。

---

**上一章**:[05-scrollback 与恢复](./05-scrollback与恢复.md) · **下一章**:[07-平台专章与设计原则](./07-平台专章与设计原则.md)
