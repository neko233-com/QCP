# QCP Protocol Specification v2.0

**新时代 UDP 可靠协议 — 超低延迟 · 高保证 · 游戏 / IoT 专用**

---

## 1. 设计哲学

QCP 是 **2026+ 独立设计的新时代 UDP 可靠协议**，专为**游戏、IoT** 等需要**超低延迟且可靠**的场景。

**不是 KCP 补丁，不是 KCP 超集。** KCP 仅作 `qcp-benchmark` 压测基线（见 [BASELINE.md](BASELINE.md)）。

生产级模块结构、wire format、状态机、安全、扩展和压测矩阵见 [PRODUCTION_ARCHITECTURE.md](PRODUCTION_ARCHITECTURE.md)。

```
Send(msg, stream, deadline) → deadline 内按语义交付，超时则放弃
```

| 对比 | KCP (2012) | QCP (2026+) |
|------|-----------|-------------|
| 定位 | UDP 可靠传输 | **游戏 / IoT 专用** |
| 丢包 | 等 RTO 8-20ms | **Fast NACK ~5ms** |
| 状态数据 | 可靠重传 | **REALTIME 最新覆盖** |
| 并发 | ~10K | **100K+ Lock-Free** |

**按消息语义选择机制，而非全局统一纠错。**

| 对比 | WebRTC / 传统可靠 UDP | QCP |
|------|----------------------|-----|
| 可靠性模型 | 字节级全可靠 | 消息级语义可靠 |
| 延迟保证 | 无 deadline | 每条消息有 deadline |
| 多路径 | 弱 | 5G + WiFi 原生 Race |

---

## 2. TLB 语义交付

### 2.0 协议层责任边界

QCP 只做传输协议层，不承载业务语义。业务层负责玩家、房间、匹配、战斗判定、登录鉴权、业务重放校验；QCP 负责把业务消息按声明的传输语义低延迟送达。

| 能力 | QCP 协议层负责 | 业务层负责 |
|------|----------------|------------|
| 身份 | SessionID、ConnectionID、Resume Token、防重放窗口 | 玩家账号、角色 ID、房间 ID、登录态 |
| 重连 | 0-RTT resume、序号连续性、未确认 CRITICAL/BATCH 恢复 | 是否允许玩家回到对局、状态快照补发 |
| 切网 | PathID、多路径探测、地址迁移、主路径切换 | 是否提示玩家网络变化 |
| 丢包 | ACK/NACK、deadline、ARQ、按需冗余、拥塞控制 | 哪类消息走 REALTIME/CRITICAL/BATCH |
| 可靠性 | 消息级顺序、去重、去旧、交付确认 | 业务幂等、业务事务、最终状态校验 |
| 安全 | 握手密钥、token 绑定、防重放、防伪造包 | 账号认证、权限、风控 |
| 扩展 | TLV/extension frame、feature negotiation | 自定义 payload schema |

协议层必须提供足够通用的机制，但不内置任何游戏业务概念。应用只声明：

```
Send(payload, stream, deadline, options)
```

QCP 根据 stream/deadline/path 状态决定如何发送、恢复、切换路径或放弃过期包。

### 2.1 三通道语义

| Stream | 语义 | 默认机制 | 典型用途 |
|--------|------|----------|----------|
| **REALTIME** | 最新状态覆盖 | 不可靠 + SeqID 去旧 | 游戏移动、IoT 遥测 |
| **CRITICAL** | 有界可靠事件 | deadline 内 Fast NACK | 射击、设备控制指令 |
| **BATCH** | 强可靠 | ARQ + ACK | 聊天、OTA 固件 |

```go
// API 示例
Send(payload, StreamCRITICAL, Deadline(8*time.Millisecond))
Send(payload, StreamREALTIME,  LatestWins())
Send(payload, StreamBATCH,      Reliable())
```

### 2.2 有界延迟（TLB）

```
deadline 内:
  REALTIME  → 收到最新 tick 即用，旧包丢弃
  CRITICAL  → Fast NACK → 最多 1-2 次重传（5G 上 1 RTT ≈ 5-10ms）
  BATCH     → 标准 ARQ，延迟不敏感

deadline 外:
  → 放弃恢复，应用层走预测 / 插值 / 状态外推
  → 绝不为「字节级可靠」无限阻塞
```

### 2.3 Recovery Policy（按需恢复，非 FEC-first）

纠删 / 编码是 **Recovery Policy**，仅在策略引擎判定需要时激活：

| 条件 | 策略 |
|------|------|
| 双路径可用（WiFi + 5G） | **多路径 Race**，优先于任何纠删 |
| 单路径 + 丢包 < 1% | 无冗余 |
| 单路径 + CRITICAL + burst 丢包 | 临时 Network Coding |
| 丢包 > 20% 或 deadline 不足 | Fast ARQ，关闭固定冗余 |
| 弱网 + 带宽敏感 | 仅 BATCH 走 ARQ，REALTIME 纯覆盖 |

**FEC / Network Coding 是工具，不是默认策略。**

---

## 3. vs KCP — 完全吊打（压测基线）

KCP 是 2012 年的 legacy 协议，**不是 QCP 组件**。量化对照见 [BASELINE.md](BASELINE.md)。

| 维度 | KCP | QCP |
|------|-----|-----|
| 设计年代 | 2012 | **2026+** |
| 目标场景 | 通用 UDP 可靠 | **游戏 / IoT** |
| 丢包恢复 | RTO 8-20ms | **Fast NACK ~5ms** |
| 状态/遥测 | 可靠重传浪费 | **REALTIME 覆盖** |
| 通道 | 单一 FIFO | **三通道语义** |
| 多路径 | 无 | **WiFi + 5G** |
| 并发 | ~10K | **100K+** |
| 包头 | 24B | **9B** |

---

## 4. 5G / 6G 设计原则

### 4.1 为什么 5G/6G 改变协议选型

```
4G 时代:  RTT 30-80ms  → FEC 用带宽换延迟，合理
5G SA:    RTT 10-20ms  → 1-RTT 重传成本骤降
5G + MEC: RTT 3-8ms    → ARQ 与 FEC 延迟差距缩小
6G 目标:  RTT ~1ms     → 空口可靠 + 多路径 > 应用层 FEC
```

固定 20% FEC 在 5G 上是**双重付费**：空口 URLLC 已提供高可靠，应用层再常驻冗余浪费带宽。

### 4.2 5G / 6G 原生能力对接

| 能力 | QCP 对接方式 |
|------|-------------|
| **URLLC 切片** | PathID 映射网络切片，CRITICAL 走低延迟切片 |
| **MEC 边缘锚点** | Session Cache 绑定最近 MEC，缩短 RTT |
| **双连接 (EN-DC)** | Multi-Path Manager：5G 主路径 + WiFi 副路径 Race |
| **5G mmWave / Sub-6** | 实时评估路径 RTT/丢包，动态切换 |
| **6G AI-native RAN** | 拥塞控制输入基站侧 QoS 预测（未来） |

### 4.3 多路径优先于纠删

```
策略 A — Race:     双发，先到先用（延迟最优）
策略 B — Diversity: 关键包走质量最优路径
策略 C — Fallback:  主路径超时立刻切副路径

当 WiFi + 5G 同时可用时，Race/Fallback 通常优于 FEC。
```

---

## 5. 核心组件

### 5.1 Multi-Path Manager

- 同时管理 WiFi 6E/7、5G SA、5G mmWave 等接口
- 实时 RTT / 丢包率 / 抖动评估
- 0-RTT 路径切换（Session Cache）

### 5.2 Fast NACK + 有界 ARQ

- CRITICAL 通道：收到 gap 立刻 NACK，不等 RTO
- 5G 上 1-RTT 重传 ≈ 5-10ms，多数场景足够
- 重传次数上限由 deadline 决定，非无限重试

### 5.3 Network Coding（按需）

- 仅在单路径 + burst 丢包 + deadline 允许时启用
- k 个数据包编码为 n 个（n > k），接收任意 k 个可解码
- 带宽效率优于固定 FEC（90-95% vs 60-80%）

### 5.4 AI-Native 拥塞控制

```
输入: RTT 序列、丢包趋势、带宽利用率、多路径状态、切片 QoS
输出: 发送速率、Recovery Policy 选择、路径建议、冗余率（若启用）
```

### 5.5 0-RTT Session

- 缓存连接参数与路径质量
- 重连 / 切换路径时直接发送数据，无需握手等待

### 5.6 生产级协议能力清单

以下能力应落在协议库中，业务只通过配置和回调感知结果：

| 能力 | 协议层落地要求 | 性能要求 |
|------|----------------|----------|
| Session Resume | Resume Token 绑定 ConnectionID、密钥 epoch、最大 seq、过期时间 | 0-RTT 恢复首包发送；无堆分配热路径 |
| Connection Migration | 同一 SessionID 可接受 NAT 端口变化、WiFi/蜂窝地址变化 | 路径切换期间 CRITICAL 不阻塞 REALTIME |
| Multi-Path Manager | PathID 独立 RTT/loss/jitter 统计，支持 active/probing/standby | O(路径数) 调度，路径数默认上限 4 |
| Recovery Policy | Race、Fast NACK、ARQ、按需 Coding 可插拔 | 策略选择不进入 payload 解析热路径 |
| Congestion Control | 每路径 pacing、拥塞窗口、丢包退避、deadline-aware 发送 | 单连接常数级状态更新 |
| Anti-Replay | 窗口化 packet number 校验，token 与地址/密钥 epoch 绑定 | 位图窗口，避免 map 热路径 |
| Extension Frame | TLV 扩展帧，未知扩展可跳过 | 不破坏老版本解析 |
| Observability | RTT、P50/P99、loss、retransmit、path switch、resume latency 指标 | 指标采样可关闭，默认低开销 |

推荐的最小 API 面：

```go
type SendOptions struct {
    Stream   StreamType
    Deadline time.Duration
    Priority uint8
    Flags    SendFlags
}

type SessionOptions struct {
    ResumeToken []byte
    MaxPaths    int
    Features    FeatureSet
}

type PathStats struct {
    PathID PathID
    RTT    time.Duration
    Loss   float64
    State  PathState
}
```

业务不应该手写重连、NAT 迁移、ACK/NACK、路径切换、重传队列；这些是协议层的职责。业务只根据 `OnResume`、`OnPathChange`、`OnTimeout` 等事件决定是否补状态快照或降级玩法表现。

---

## 6. 协议架构

```
┌─────────────────────────────────────────────────────────────┐
│                    游戏应用层                                │
├─────────────────────────────────────────────────────────────┤
│                    QCP 2.0 API                              │
│  Send(data, stream, deadline)                               │
│  Recv() → {data, stream, path, seq}                        │
├─────────────────────────────────────────────────────────────┤
│              TLB Semantic Scheduler                         │
│  REALTIME: LatestWins  │  CRITICAL: BoundedARQ             │
│  BATCH: FullARQ        │  deadline 超时 → 放弃 + 预测      │
├─────────────────────────────────────────────────────────────┤
│              Recovery Policy Engine（按需）                  │
│  Multi-Path Race → Fast NACK → Network Coding → ARQ       │
├─────────────────────────────────────────────────────────────┤
│  Multi-Path │ AI Congestion │ 0-RTT Session │ Lock-Free    │
├─────────────────────────────────────────────────────────────┤
│  QCP Packet (8-12 bytes)                                    │
│  Type │ Stream │ SeqID │ PathID │ Payload                   │
├─────────────────────────────────────────────────────────────┤
│  WiFi 6E/7  │  5G SA  │  5G mmWave  │  ...  │  6G         │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. 性能对比

| 指标 | KCP | WebRTC DC | QCP 2.0 |
|------|-----|-----------|---------|
| P50 延迟 (5G) | 97ms | 20-50ms | **<5ms** |
| 语义分层 | 无 | 无 | **REALTIME/CRITICAL/BATCH** |
| 有界延迟 | 无 | 无 | **deadline 驱动** |
| 多路径 | 不支持 | 弱 | **Race / Fallback 原生** |
| 常驻带宽开销 | 低 | 中（栈 + 可选 FEC） | **低（按需 Recovery）** |
| 0-RTT | 不支持 | 不支持 | **支持** |
| 5G 切片对接 | 不支持 | 不支持 | **PathID 映射** |

---

## 8. 语言绑定与游戏集成

**Go（`qcp-lib-go`）是 QCP 的参考实现与协议验证语言。** 新特性先在 Go 中实现并通过 `qcp-benchmark` 验证，再移植到 C/C++ 及其他主流绑定。

仅维护主流生态绑定（不含 Nim / Zig / Elixir）：

| 分类 | 仓库 | 说明 |
|------|------|------|
| **参考实现** | qcp-lib-go | 协议验证、API 规范、首选开发 |
| 系统 / 引擎 | qcp-lib-c, qcp-lib-cpp, qcp-lib-rust | C / C++ / Rust |
| 游戏客户端 | qcp-unity-lib, qcp-lib-csharp, qcp-lib-swift, qcp-lib-typescript | Unity / iOS / WebGL |
| 移动端 | qcp-lib-java, qcp-lib-kotlin | Android / JVM |
| 服务端 | qcp-lib-erlang | 分布式游戏服务器 |
| 脚本 | qcp-lib-lua, qcp-lib-verse | Lua / UE6 Verse |

| 引擎 / 平台 | 绑定仓库 | 接入方式 |
|-------------|----------|----------|
| Unity | qcp-unity-lib | UPM 包 → MonoBehaviour / NetworkBehaviour / 平台 Transport |
| Unity 底层绑定 | qcp-lib-csharp | C# 协议绑定 → Unity 包或自定义工程复用 |
| Unreal Engine | qcp-lib-cpp | C++ 库 → UE Module + Blueprint |
| UE6 Verse | qcp-lib-verse | Verse 原生 API |
| Unity WebGL / 微信小游戏 | qcp-unity-lib | `.jslib` 平台桥；微信小游戏替换 JS 桥内部实现 |
| Web / Node.js | qcp-lib-typescript | TypeScript / WASM |
| Android | qcp-lib-java / qcp-lib-kotlin | JNI / 原生 SDK |
| iOS | qcp-lib-swift | Swift Package |
| 游戏服务端 | qcp-lib-erlang | Erlang/OTP |

---

## 9. 适用场景

```
QCP 2.0 适合:
✓ 5G SA / 5G + MEC / 未来 6G
✓ WiFi 7 + 5G 双路径设备
✓ 高频战斗（MOBA / FPS / 格斗）
✓ 延迟敏感 + 消息语义可分层
✓ 2026+ 新游戏

QCP 2.0 不适合:
✗ 需要 WebRTC 互通的语音/视频（请用 WebRTC 媒体通道）
✗ 纯文件传输 / 非实时 bulk 数据
✗ 无法做语义分层的 legacy 协议迁移
```

---

## 10. 版本路线

| 版本 | 状态 | 核心特性 |
|------|------|----------|
| QCP 1.0 | 已废弃叙事 | ~~FEC-First~~（已被 TLB 取代） |
| QCP 2.0 | **当前规范** | TLB 语义交付、多路径优先、按需 Recovery、0-RTT |
| QCP 2.x | 规划中 | 5G 切片 PathID、MEC Session 锚定、6G QoS 对接 |

---

## 11. 技术演进

```
2010: 纯 ARQ (KCP/TCP)
2015: FEC 固定冗余 (WebRTC FlexFEC)
2020: Network Coding + QUIC
2024: 多路径 (MPQUIC) + 0-RTT
2025: 语义可靠 / 部分可靠 (游戏行业共识)
2026: QCP — TLB 语义交付 + 5G/6G 原生多路径
```

---

*QCP 2.0: 不是更快地恢复每一个字节，而是在 deadline 内交付正确的语义。*
