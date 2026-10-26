# 第 4 章 · Shell Wrapper 体系:把用户的 shell 变成可观测的

> daemon 需要知道"shell 什么时候真的能收输入了"、"一条命令什么时候开始/结束"——
> 这些 shell 不会主动告诉你。wrapper 是 Orca 生成的一套 shell 启动文件,在用户自己的配置之后注入钩子。
> 本章讲它的存放方式(内容寻址)与五种 shell 各自的注入手法。引用为仓库相对路径。

---

## 4.1 wrapper 提供什么

| 能力 | 机制 | 消费方 |
|---|---|---|
| shell-ready 标记 | OSC 777 `orca-shell-ready` | startup command 投递门(第 6 章) |
| 命令生命周期标记 | OSC 133 A/B/C/D | agent 状态机、transcript 分段 |
| 启动命令投递 | startup ingress(ready 后"键入") | `--command`、agent launch |
| agent 函数 | `codex`/`claude` shell 函数 | 用户在终端里直接调 agent |
| 环境 overlay | ORCA_CODEX_HOME、OPENCODE_CONFIG_DIR 等恢复 | agent 配置隔离 |
| 启动身份 | startup identity marker | 终端归属识别 |

为什么必须等 ready:daemon 只有在 shell 真正可交互时才能投递 startup command——慢 prompt(如 Starship)下提前写入的字节会丢。

---

## 4.2 内容寻址树:目录名即内容哈希

### 问题:Last writer won

`src/main/shell-wrapper-content-address.ts:4-14` 记录了旧形态的事故:

> "every writer that shares a userData dir — main's local PTY path, the daemon fork, and the daemons of other builds that outlive the app that spawned them — wrote one fixed `shell-ready/` tree. **Last writer won**, and the guard only re-checked that the files were present, never that they were this build's... the ready marker never fired and **every startup command waited out the full readiness timeout, silently**."

共享 userData 的写者有三类:main 的本地 PTY 路径、daemon fork、以及**比生成它的 app 活得久的其他 build 的 daemon**(第 1 章的多代共存)。它们共写一个固定目录,旧 daemon 读到新 wrapper 就不认得,ready marker 永不触发——症状是每条 startup command 静默等满超时。

### 解法:存在即本 build 所写

目录名 = 内容 sha256 前 16 hex(`resolveShellWrapperRoot:53-63`)——**不同字节就是不同目录,"存在"重新等于"本 build 写的"**。两个细节:

- **占位根** `HASH_PROBE_ROOT = '/__orca_shell_wrapper_root__'`(:29-43):`.zshenv` 里烧进了树路径,直接对真实路径哈希会让摘要依赖它自己要选的路径(自指)。占位符也是摘要的一部分——改名会 re-key 所有树,代价只是每个 install 多一个 48KB 目录,"not worth normalizing out";
- **hash 在 `shell-ready` 叶之上**(:45-50):ZDOTDIR 自引用 guard(TS 里和脚本里的 `*/shell-ready/zsh` glob)按这个后缀匹配;缺了它 wrapper 会 source 自己,zsh 死于 "job table full or recursion limit exceeded"。`<base>/<hash>/shell-ready/zsh` 既保住 guard,又让旧 build 认得新 build 的目录。

### 为什么不回收旧树(:16-20)

> "a tree is ~48KB and a new one appears only when the wrapper templates themselves change, so an install accrues a couple of megabytes a year against a userData directory that runs to tens of gigabytes. **Reclaiming that is not worth putting an `rm -rf` on the terminal-spawn path.**"

每年几 MB vs 在终端创建路径上放一把 `rm -rf`——不划算。这是"破坏性操作的代价/收益"权衡的范本。

### 原子写与失败回退

`shell-wrapper-file-writer.ts:1-9`:写失败(只读 FS、磁盘满、EACCES)时**必须回退到未包装 shell**——否则 ZDOTDIR 指向空目录,zsh 找不到 `.zshrc`,用户静默丢失全部配置。temp+rename(:23-29)保证并发读到的永远是完整文件。

---

## 4.3 feature 通道:正向 allowlist,阅后即焚

`src/main/shell-startup-features.ts:1-13`:

> "Why a positive allowlist the wrapper destroys before anything else runs: every earlier switch was a negative, exported one (`ORCA_SHELL_READY_MARKER=0`...). Those live in the pane's PTY env, so **every child inherits them — a pane launched with a feature suppressed suppressed it for an Orca started from that pane too**."

旧设计是负向开关(`=0` 关闭),会随环境变量继承:用户从一个"关了 ready marker"的 pane 里启动另一个 Orca,新 Orca 的终端也被传染。正向 allowlist 下,继承来的脏值只可能意味着**更少**的功能,不会更多;且 wrapper 在任何用户配置运行前就 unset 它。feature 集:overlay/history/markers/ready/identity/startup(:16-23);选择是纯函数(`selectShellStartupFeatures`),刻意不读继承来的 `ORCA_SHELL_FEATURES`(:51-58)。

---

## 4.4 zsh:ZDOTDIR 归还与延迟初始化

zsh 是唯一用 `ZDOTDIR` 注入的 shell(`daemon/shell-ready.ts:152-170`)。`src/main/zsh-startup-wrapper-builder.ts:7-23` 记录了旧形态的"bug 家族之根":

> "The old shape bought that by keeping ZDOTDIR pointed at Orca's wrapper dir for the whole of startup... `/etc/zshrc` assigns `HISTFILE=${ZDOTDIR:-$HOME}/.zsh_history` unconditionally, so **history landed inside Orca's own dir (#11044)**; zsh's `sourcehome()` ignores ZDOTDIR once the shell enters sh/ksh emulation...; and one wrapper dir shared by two installed builds could mix files from both."

新形态:

1. `.zshenv` **首行归还 ZDOTDIR**——之后 `/etc/zshrc`、`.zshrc` 全部从用户自己目录读,与未包装 shell 完全一致;
2. Orca 的工作推迟到**首个 prompt 的 `precmd` 钩子**(`__orca_deferred_init`,:146-210),配一次性 line-editor 兜底(防用户配置替换 hook 数组);
3. **函数必须先于 source 用户 `.zshenv` 定义**(:25-30)——"ORDER IS LOAD-BEARING":用户文件末尾的 `emulate sh` 会把后续内容置于 sh 解析规则下,zsh 专有语法直接解析失败带走整个 wrapper;函数体在定义时解析,先定义即免疫;
4. OSC 133 永久钩子的处理(:147-153):"Verified on zsh 5.9 that self-removal mid-sweep skips no later hook"——自删钩子要换名占位而不是摘除,保持用户其他钩子的数组位置;
5. `#11044 becomes unreachable rather than repaired`(:193-197)——不修 bug,让 bug 在结构上不可达。

文件集上,只有 `.zshenv` 是 Orca 写的被读文件(`daemon/daemon-shell-ready-wrapper-fileset.ts:24-26`):hook 首行归还 ZDOTDIR,其余启动文件全从用户目录来。

---

## 4.5 bash:DEBUG trap 驱动的 OSC 133

注入方式:`--rcfile <root>/bash/rcfile`(`shell-ready.ts:172-183`)。rcfile(`daemon/daemon-bash-shell-ready-rcfile.ts:9-149`)的要点:

- **feature channel 先消费即销毁**(:13-18):"the channel is consumed and destroyed in these first lines, so nothing this shell later spawns can see or inherit the selection";
- 然后 source `/etc/profile`、`~/.bash_profile` 等(:19-26)——用户环境照旧;
- **强制 bracketed paste**(:27-31):多行 startup prompt 作为单次 paste 投递(ESC[200~…ESC[201~);否则老 readline 把内嵌换行当 Enter,prompt 被拆进 PS2 continuation;
- **DEBUG trap 驱动 OSC 133 C/D**(:50-53):bash 用户没有 Stop/SessionEnd hook 时,没有它 spinner 会卡死;与 bash-preexec/starship 框架的 trap 争夺处理见 :65-114,epilogue 每个 prompt 重夺 trap(:115-125);
- **最后才 arm DEBUG**(:143-146):否则 bash 把 rcfile 自己的命令当前台命令,在首个 prompt 前发出假的 C/D;
- ready marker 经 PS1 注入 `\[\e]777;orca-shell-ready\a\]`(:121-124)。

---

## 4.6 fish:不走 marker 的投递

plain fish 走 env-only:prepend Orca 数据目录到 `XDG_DATA_DIRS`,fish source `vendor_conf.d/orca-shell-integration.fish`,snippet 第一件事是撤销该 env(`fish-xdg-data-dirs-handoff.ts:1-5`)。需要 ready/startup 时用 `fish -l -C '<init>'`。

为什么 fish 的 startup command 不走 ready-marker 门(`shell-ready.ts:120-122`):

> "Why fish: markerless, its startup command is written before fish's reader owns the PTY and **the launch is lost under slow prompts like Starship (STA-3417)**."

`-C` init 在 fish 的 reader 接管 PTY 之前写入启动命令——marker 路径反而会在慢 prompt 下丢命令。`-N/--no-config` 检测见 `fish-xdg-data-dirs-handoff.ts:26-31`。

---

## 4.7 PowerShell:-EncodedCommand 的 MDE 答辩

注入:`-NoLogo -NoExit -EncodedCommand <base64>`(`shell-ready.ts:185-197`)。`powershell-osc133-bootstrap.ts:7-39` 是一整段"为什么坚持 base64"的安全答辩——一次 MDE(Microsoft Defender for Endpoint)报告点名 base64 是"可疑 PowerShell"信号,并拿 VS Code 当反例。回应里有张实测表:

> ```
>   policy        dot-source .ps1   -Command inline   -EncodedCommand
>   Restricted    blocked           runs              runs
>   AllSigned     blocked           runs              runs
>   RemoteSigned  runs              runs              runs
> ```

论证三层:

1. **VS Code 的 dot-source 方案在 Restricted/AllSigned 下被静默阻断**——"its `catch {}` is that failure being swallowed",而 MDE 跑的恰恰是锁死的舰队;OSC 133 一丢,前台进程与退出码跟踪全没;
2. **inline `-Command` 技术上可行但信号更糟**:命令行里会出现 `$ExecutionContext.SessionState.LanguageMode`、`function Global:prompt` override、`[char]27` 组装的控制序列明文——"higher-signal for command-line heuristics than an opaque token with no `Bypass` beside it"。没有 `-ExecutionPolicy Bypass`,AMSI 和 script-block logging 反正能解 base64——"nothing is hidden from MDE today";
3. **`-EncodedCommand` 是 quoting-proof 的**:对照 #7978——cmd.exe 的 startup command 不得不从 `/K` 挪到 stdin,因为 node-pty 的 argv 转义弄坏引号;PowerShell 从不需要这种变通。

bootstrap 本体(:40-103):包装 `prompt` 函数发 133;A/D/B、包装 `PSConsoleHostReadLine` 发 133;C、UTF-8 编码保 CJK。Windows CLM(ConstrainedLanguage)变体在 `powershell-osc133-bootstrap-windows-clm.ts`。**cmd.exe 无 wrapper**,startup command 走 stdin(#7978)。

---

## 4.8 协议面小结

```
shell 启动
  │  .zshenv 首行归还 ZDOTDIR / rcfile source 用户文件 / fish -C init
  ▼
OSC 777 "orca-shell-ready" ──► daemon:ready 门开,startup ingress 开始投递
  │
  ▼ 每个 prompt
OSC 133;A (prompt 开始)
OSC 133;C (命令执行前,preexec/DEBUG trap)
  │  命令输出
OSC 133;D;exitcode (命令结束)
```

OSC 133 让 daemon 不靠猜就知道命令边界——agent 状态机(见 [agent 系列](../agent-lifecycle/01-概览与架构.md))与 transcript 分段(第 6 章)都建立在它上面。

---

**上一章**:[03-PTY 层](./03-pty层.md) · **下一章**:[05-scrollback 与恢复](./05-scrollback与恢复.md)
