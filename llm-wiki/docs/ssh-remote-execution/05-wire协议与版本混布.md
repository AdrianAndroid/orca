# 第 5 章 · Wire 协议与版本混布

> 客户端与远端 server 各自独立更新,**版本混布是常态**。本章讲两层协议的帧格式、版本窗口、capability 协商,以及"新 opcode 为什么不能随便加"。
> 引用为仓库相对路径。

---

## 5.1 两层协议

| 层 | 状态 | 帧格式 | 载荷 |
|---|---|---|---|
| **SSH relay 私有协议** | legacy,解码端仍维护 | 13 字节头(VS Code PersistentProtocol 兼容) | JSON-RPC 2.0 |
| **Orca runtime RPC** | 现行,所有 remote 执行走它 | transport 解耦(Unix socket / WebSocket / named pipe) | Zod schema 化 JSON,流式可选 |

### relay 协议(legacy)

`src/main/ssh/relay-protocol.ts:1-3`:

> "13-byte framing header matching VS Code's PersistentProtocol wire format."

头布局(:76-84):`[0]=TYPE, [1-4]=ID BE, [5-8]=ACK BE, [9-12]=LENGTH`;消息类型 Regular=1 / KeepAlive=9(:34-37),keepalive 5s / 超时 20s(:40-41)。握手哨兵 `RELAY_SENTINEL = 'ORCA-RELAY v0.1.0 READY\n'`(:28-31,10s 超时)。版本失配以 **exit 42** 上报(`ssh-relay-version-mismatch-error.ts`),`RelayVersionMismatchError` 是终态错误:跳过指数退避,提示用户手动重连。帧解码预算在 `src/shared/relay-frame-decoder.ts:11-17`(13 字节头、16MiB 上限、每 turn 64 帧/4ms)。

### runtime RPC(现行)

- **方法声明三合一**:`src/main/runtime/rpc/core.ts:1`——"single boundary between raw RPC frames and OrcaRuntimeService; keeps schema, handler, and result type on one object"(Zod schema + handler + 类型同体);
- **信封**(:37-73):`{id, ok, result|error{code,message,data}, _meta.runtimeId}`,流式 `streaming?: true`;
- **每连接握手**(`src/shared/remote-runtime-request-socket.ts:140-170`):E2EE 握手 → `sealAuth({kind:'token',deviceToken}, {clientCapabilities})` → `runtime.status` 预检 → 真正 RPC;
- **WebSocket transport**(`ws-transport.ts`):每设备 token 鉴权、15s ping/pong 收半开连接(:28-31,STA-3320)、pre-auth 10s、128 连接上限;端口策略含 #8535、STA-1511、#26603(:46-54);
- **宽容参数**:`src/shared/rpc-contract/rpc-param-primitives.ts:1-30`——"Preserve that forgiving behavior so CLI callers passing stringified numbers or Infinity still reach the runtime";
- **E2EE 与 RPC 分层**:`e2ee-channel.ts:1`——"keeps E2EE framing out of RPC handlers, which consume plaintext across transports"。

---

## 5.2 版本窗口与 bump 守则(混布宪法)

`src/shared/protocol-version.ts:133-154`,守则全文引用:

> "Bump RUNTIME_PROTOCOL_VERSION when:
> - You remove an RPC method or required parameter that clients use.
> - You change the meaning (units, nullability) of an existing field clients read.
> - You change encrypted framing, terminal stream framing, or auth.
> Do NOT bump for:
> - Adding new RPC methods.
> - Adding new optional fields on existing methods.
> - Adding new ignorable event types."

当前窗口:`RUNTIME_PROTOCOL_VERSION = 3`,双向 MIN 兼容版本 = 2(:152-154)。"Exact app-version equality is never required; these numbers define the supported compatibility window"——**对等的不是 app 版本,是协议窗口**。

判定器(`src/shared/protocol-compat.ts:20-54`)的关键策略:缺字段视为 protocol 0——"New clients can give old servers a clear 'update server' error instead of attempting partially-supported RPCs"(:26-29)。还有 kill-switch 优先级(:89-91):"if desktop says 'I refuse this mobile build', that wins";反向(:82-85):"absent fields → 0 lets mobile keep talking to pre-PR desktops. Bumping minCompatibleDesktopVersion above 0 will fence those older desktops... which is the intended kill-switch behavior"——**版本门槛是有意的熔断器**。

### capability 清单

`protocol-version.ts:172-273` `RUNTIME_CAPABILITIES`(100+ 字符串):`'terminal.binary-stream.v1'`、`'terminal.multiplex.v1'`、agent session host 权威(:231)、SSH target 管理(:271)……客户端 capabilities 解析有界(64 条、每条 ≤128 字符,`runtime-client-capabilities.ts:3-12`)。

### COMPAT 惯例

仓库级标记(:277-279):

> "COMPAT(mobileProtocolAliases): added 2026-05-15 for mobile builds that still read desktop/mobile names; remove once mobile reads runtime names."

临时兼容代码一律挂 `COMPAT(name): added/revisit <date>`——legacy bridge 的 revisit 是 2027-01-15。**兼容代码必须有日落日期,否则它会活成协议本身**。

---

## 5.3 终端二进制流:帧格式与 opcode 治理

### 帧格式

`src/shared/terminal-stream-protocol.ts:3-10,72-107`:16 字节头(kind=0x74、version=1、opcode、reserved、streamId LE u32、seq LE u48 拆两段)+ payload。JSON 载荷限 8MiB、256k 结构 token、深 32。

**解码器对未知 opcode 返回 null**(:95-98)——这就是 AGENTS.md "decoders drop unknown opcodes silently" 的落码,也是 opcode 治理存在的理由:**你加的 opcode,旧端不是拒绝,是看不见**。

### opcode 兼容注释逐条(本章核心素材)

| opcode | 注释 | 兼容策略 |
|---|---|---|
| Ack=13(:25-26) | "Metadata=12 shipped to mobile clients in v1.4.120; Ack... renumbers to stay wire-compatible" | 新功能**让号**给已发布的旧号 |
| ClaimViewport=14(:28-30) | "Ack already occupies 13 on current clients; older runtimes ignore this opcode and still receive the compatibility Resize frame behind it" | 新帧在前,**兼容 Resize 帧垫底** |
| SetOutputPaused=16(:32) | "Negotiated per stream; older hosts reject unknown opcodes, so clients send only after capability confirmation" | capability 确认后才发 |
| WriteUnavailable=17(:34-35) | "Negotiated per stream because older clients reject unknown opcodes" | 同上 |
| InputAck=18(:36-37) | host→client 累计输入序号,"Sent only after `inputAck` negotiation" | 协商后才发 |
| PausedOutputDropped=19(:38-39) | 每次暂停至多一次,"Sent only after `outputPauseDropNotice` negotiation" | 协商后才发 |

三种策略对应三类新 opcode:**可以静默忽略的**(ClaimViewport——旧端忽略,靠垫底的 Resize 达到近似效果)、**必须双向确认的**(16-19——旧端会拒绝或行为错误)、**让号保序的**(Ack——新分支重编号迁就已发布的线格式)。

发送端的配套实现(`remote-runtime-terminal-multiplexer-implementation.ts:111-119`):"Claim first keeps new-runtime ownership precise and leaves a backwards-compatible resize immediately behind it"。

**协商是 per-stream 的**:订阅帧携带 `capabilities` 对象(`ackOutput`、`outputPause`、`writeUnavailable`、`inputAck`……,`terminal-multiplex-stream-initialization.ts:31-41` 每条流独立记录)——同一 host 上新旧客户端并存,各谈各的。

### InputAck 语义

第 4 章 §4.3 已列三态表;协议侧定义在 `terminal-stream-protocol.ts:42-63`:`applied`(空 payload)/ `delivery-unknown`("may have reached; must not resend")/ `resend`("everything after `seq`, in order")。

---

## 5.4 宿主侧版本准入:orcad 的反降级

wire 兼容管"协议",版本准入管"状态"——`src/main/ssh/orcad-host-version-admission.ts:1-21`:

> "The downgrade rule every orcad activation on a shared host applies: **a build may not start over state a newer Orca's orcad may already have migrated**."

新版本可能迁移过远端状态,旧版本在其上启动 = 用旧代码读新格式。但有一个刻意缺口(:15-17):

> "Why an unnamed version is admitted: every Stop before the field existed omits it, so **refusing would strand those hosts for good**; only a provably newer host is refused."

字段出现之前停止的 host 没有版本记录——拒绝它们 = 永久放逐。**只有"可证明更新"才拒绝,"不知道"一律放行**。这与失联语义同构:fail 的方向永远偏向"不造成不可逆后果"。

配套:`orcad-running-version-adoption.ts`——host 被别人的版本占着时先 pair 上在跑版本,由 linked update check 决定替换;fence 持有者无法被普查时只在 fence 清除后才 defer(#28833)。

---

**上一章**:[04-失联语义与对账](./04-失联语义与对账.md) · **下一章**:[06-部署与上传链](./06-部署与上传链.md)
