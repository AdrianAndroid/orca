# Orca LLM Wiki

深度学习用 wiki,每篇聚焦一个子系统。所有结论附 `文件:行号` 出处,并大量引用代码中的 "Why" 注释原文(英文)+ 中文解读——这个仓库的设计意图很大程度固化在注释里。

生成于 2026-10-16(commit ebb9e3243a,分支 learn);agent-lifecycle / ssh-remote-execution / terminal-pty 三篇增补于 2026-10-17。代码持续演进,行号可能漂移,以符号名为准。

## 篇章

### [worktree-orchestration/](./worktree-orchestration/01-概览与架构.md) · Worktree 编排(6 章,1654 行)

| 章 | 内容 |
|---|---|
| [01-概览与架构](./worktree-orchestration/01-概览与架构.md) | 系统定位、术语表、分层图、执行主机路由元规则、Git 兼容层、创建/删除对照图 |
| [02-创建管线](./worktree-orchestration/02-创建管线.md) | `performLocalWorktreeCreate` 17 个阶段逐拆、命名循环 8 道检查、`git worktree-add.ts` 全解、准入层机制 |
| [03-预备池](./worktree-orchestration/03-预备池.md) | 投机执行核心:claim 协议(同步代替锁)、retarget 分歧预算、finalize 6 次验锁、burst 门控 re-arm |
| [04-删除管线与安全护栏](./worktree-orchestration/04-删除管线与安全护栏.md) | 请求合并 optionsKey、archive hook 硬前置、后台删除 WAL、home 守卫 unverifiable 论证、孤儿证明链 |
| [05-身份命名与持久化](./worktree-orchestration/05-身份命名与持久化.md) | instanceId 防 ABA、生成名退休双写注册表、扫描世代号、剪枝证据门控(#17775) |
| [06-设计模式与原则](./worktree-orchestration/06-设计模式与原则.md) | 15 个并发设计模式、5 条贯穿原则、10 个刻意不做的反模式、三遍式阅读路径 |

建议从 06 章的 §6.4 阅读路径开始,按三遍式顺序读源码,wiki 各章当讲义。

### [agent-lifecycle/](./agent-lifecycle/01-概览与架构.md) · Agent 生命周期(6 章,882 行)

| 章 | 内容 |
|---|---|
| [01-概览与架构](./agent-lifecycle/01-概览与架构.md) | hook server 单 store 双通道模型、26 层 mixin 剖面、术语表、四条摄取通道全景 |
| [02-hook摄取管线](./agent-lifecycle/02-hook摄取管线.md) | loopback listener 防守三连、fail-open 哲学、spool、SSH 信任边界再归一化、OSC 9999、structured 投影 |
| [03-状态词表与三态语义](./agent-lifecycle/03-状态词表与三态语义.md) | 行状态四态、进程三态 `live/unverifiable/exited`、"失联不是死亡"引文集、30min 陈旧窗口与时钟规则 |
| [04-生命周期状态机](./agent-lifecycle/04-生命周期状态机.md) | 启动注册(launchToken)、事件租约、退出判定四通道(均需正面证据)、state claim 清理清单 |
| [05-并发护栏与跨主机边界](./agent-lifecycle/05-并发护栏与跨主机边界.md) | generation/launchToken/retired 三 fence、迟到探针防护、execution host owns status、unverifiableTerminalCount 兼容读取 |
| [06-读者呈现与演进史](./agent-lifecycle/06-读者呈现与演进史.md) | sidebar/CLI/mobile/dashboard 呈现策略、Why 注释编目(#11217/#7970/STA-4658……)、10 个设计模式、阅读路径 |

### [ssh-remote-execution/](./ssh-remote-execution/01-概览与拓扑.md) · SSH / 远程执行边界(7 章,768 行)

| 章 | 内容 |
|---|---|
| [01-概览与拓扑](./ssh-remote-execution/01-概览与拓扑.md) | 三个 "relay" 正名、SSH 边界总图、orcad 即执行主机、幽灵引用声明 |
| [02-host标识与操作路由](./ssh-remote-execution/02-host标识与操作路由.md) | ExecutionHostId 语法与哨兵、HostAuthority 二元组、路由决策表 |
| [03-连接生命周期与重连](./ssh-remote-execution/03-连接生命周期与重连.md) | 连接池、双传输、三层 fencing 纪元、重连阶梯与 flap grace 上限 |
| [04-失联语义与对账](./ssh-remote-execution/04-失联语义与对账.md) | 判定词表宪法注释、排队 vs 拒绝矩阵(kill 重放 #16970、InputAck)、重连对账七步 |
| [05-wire协议与版本混布](./ssh-remote-execution/05-wire协议与版本混布.md) | 双层协议、bump 守则全文、opcode 逐条兼容注释、capability 协商、orcad 反降级准入 |
| [06-部署与上传链](./ssh-remote-execution/06-部署与上传链.md) | bundle 清单、安装锁(BUG-23)、SFTP/exec-stdin 回退、平台特例、relay→orcad 迁移与日落 |
| [07-测试地图与故障案例库](./ssh-remote-execution/07-测试地图与故障案例库.md) | 设计原则 7 条、12 个故障案例、hostile-hosts 测试矩阵、三遍式阅读路径 |

### [terminal-pty/](./terminal-pty/01-概览与架构.md) · 终端 / PTY 子系统(7 章,966 行)

| 章 | 内容 |
|---|---|
| [01-概览与架构](./terminal-pty/01-概览与架构.md) | 三层拓扑、daemon 化动机(app 可以死,终端不能死)、协议版本门、60s fail-open |
| [02-daemon生命周期](./terminal-pty/02-daemon生命周期.md) | 启动 rungs、端点所有权协议(23 缺陷同一交错)、空闲退出、respawn 熔断、多代路由 |
| [03-pty层](./terminal-pty/03-pty层.md) | node-pty Orca 补丁逐条(job/breakaway/不加 KILL_ON_JOB_CLOSE)、kill 升级链、ConPTY 预热、`--shell` vs `--command` |
| [04-shell-wrapper体系](./terminal-pty/04-shell-wrapper体系.md) | 内容寻址树、正向 allowlist、zsh/bash/fish/PowerShell/cmd 五种注入(#11044、MDE 答辩、STA-3417、#7978) |
| [05-scrollback与恢复](./terminal-pty/05-scrollback与恢复.md) | OCKL 帧格式、撕裂/洞检测、dirty-gate checkpoint、cold restore、两套快照机制 |
| [06-终端与agent交互](./terminal-pty/06-终端与agent交互.md) | readiness 四信号一道门、transcript、title 权威、归属与收养 |
| [07-平台专章与设计原则](./terminal-pty/07-平台专章与设计原则.md) | Windows 三道保命工程、macOS TCC 归因、8 条设计原则、Why 注释编目、阅读路径 |

## 系列间的关系

四篇不是孤立的:**同一套设计语言在不同域的复现**。`live / unverifiable / exited` 三态词表由 SSH 边界(ssh-remote-execution 第 4 章)定义,被 agent 状态(agent-lifecycle 第 3 章)与终端 daemon(terminal-pty 第 2 章)共用;"await 后重读世界""世代号防伪""空是危险值"等并发模式在 worktree 与 agent 两个子系统里逐条对照(agent-lifecycle 第 5 章 §5.4)。建议先读 worktree-orchestration 第 6 章的模式汇总,再带着模式清单读其余三篇。

## 待写(候选)

- Orchestration / fleet 子系统(`orchestration-legacy-*` 兼容层、`ORCHESTRATION_CONTRACT_VERSION`)
- Mobile 配对与云 relay(E2EE 通道、`@orca-cloud/relay`)
- 持久化与 session 恢复(loading-store、pty-binding、terminal-topology)
