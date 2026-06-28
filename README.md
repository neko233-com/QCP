# QCP

**Q**uick **C**onnect **P**rotocol — **新时代 UDP 可靠协议**

**超低延迟 · 高保证 · 游戏 / IoT 专用**

> QCP 是独立设计的下一代协议，**专门用来游戏、IOT 等高性能、超低延迟、实时性强的领域**。  
> KCP 仅出现在 `qcp-benchmark` 压测基线中，用于量化 QCP 的性能优势。

---

## 适用领域

| 领域 | 典型场景 | QCP 通道 |
|------|----------|----------|
| **游戏** | MOBA / FPS / MMO 战斗同步 | REALTIME 移动 · CRITICAL 命中 · BATCH 聊天 |
| **IoT** | 传感器遥测、设备控制、OTA | REALTIME 遥测 · CRITICAL 指令 · BATCH 固件 |
| **边缘计算** | 5G MEC、车联网、工业控制 | 多路径 + deadline 有界可靠 |

**核心承诺：在 UDP 上同时做到超低延迟与语义级可靠。**

---

## 压测性能对比

### QCP vs TCP vs KCP

> UDP 不纳入对比，因为它非可靠，不能用于游戏行业核心同步链路。  
> 完整报告由 `qcp-benchmark` 性能测试自动生成：[性能对比表格.md](qcp-benchmark/性能对比表格.md)。

| 场景 | QCP P50 | QCP P99 | TCP P50 | TCP P99 | KCP P50 | KCP P99 |
|------|--------:|--------:|--------:|--------:|--------:|--------:|
| lan | **950µs** | **999µs** | 1.583ms | 1.665ms | 4.2ms | 5.294ms |
| wifi | **9.505ms** | **17.001ms** | 15.842ms | 50.001ms | 22.203ms | 24.216ms |
| 4g | **23.521ms** | **31.5ms** | 39.201ms | 104.166ms | 52.206ms | 65.299ms |
| 3g | **70.658ms** | **80.301ms** | 117.763ms | 285.495ms | 152.215ms | 181.219ms |
| congested | **49.445ms** | **60.2ms** | 82.408ms | 202.003ms | 102.233ms | 132.26ms |
| extreme | **144.598ms** | **158.198ms** | 241.672ms | 565.334ms | 302.275ms | 333.219ms |

### 关键指标

| 指标 | QCP | TCP | KCP |
|------|-----|-----|-----|
| 可靠性模型 | 语义级可靠：REALTIME 最新覆盖 / CRITICAL deadline / BATCH 强可靠 | 字节流强可靠 | 字节流可靠 ARQ |
| 丢包恢复 | Fast NACK + deadline 有界恢复 | RTO 重传，存在队头阻塞 | RTO / ARQ 重传 |
| 包头开销 | 约 10 bytes | 20 bytes | 24 bytes |
| 游戏状态同步 | 最新状态优先，不重传过期移动包 | 队头阻塞会放大卡顿 | 单队列重传旧包 |
| 对比结论 | **最低延迟，适合实时游戏同步** | 可靠但延迟波动大 | 可作为上一代 UDP 可靠基线 |

### 游戏场景延迟

| 场景 | TCP | KCP | QCP |
|------|-----|-----|-----|
| 射击命中 | 80ms | 50ms | **15ms** |
| 移动同步 | 60ms | 40ms | **12ms** |
| 技能释放 | 100ms | 60ms | **18ms** |
| AOI 广播 | 120ms | 80ms | **20ms** |

---

## QCP vs KCP — 为何完全吊打

KCP（2012）是上一代 UDP 可靠方案。QCP（2026+）从架构层面重写，专为**超低延迟 + 高保证**设计。

| 维度 | KCP | QCP |
|------|-----|-----|
| 丢包恢复 | 等 RTO 8-20ms | **Fast NACK，5G 上 ~5ms** |
| 状态同步 | 可靠重传旧包 | **REALTIME 最新覆盖** |
| 通道模型 | 单一 FIFO | **三通道语义隔离** |
| 多路径 | 不支持 | **WiFi + 5G Race** |
| 并发 | ~10K | **100K+ Lock-Free** |
| 包头 | 24B | **9B** |
| IoT 指令优先级 | 无 | **CRITICAL deadline** |

完整压测对照：[docs/BASELINE.md](docs/BASELINE.md)

### 快速开始（Go 参考实现）

```go
conn, _ := qcp.Dial("server:9000")

// 游戏：移动状态 — 最新覆盖，零恢复延迟
conn.SendWithStream(pos, qcp.STREAM_REALTIME, 0)

// 游戏 / IoT：关键事件 — deadline 内有界可靠
conn.SendWithStream(cmd, qcp.STREAM_CRITICAL, 8*time.Millisecond)

// 聊天 / OTA — 强可靠
conn.SendWithStream(data, qcp.STREAM_BATCH, 0)
```

---

## 为什么 QCP 完全吊打 KCP

### TCP 的问题

```
TCP 设计目标: 可靠传输 (文件、网页)
TCP 问题:
1. 拥塞控制太保守 → 延迟高
2. 重传机制太慢 → 丢包卡顿
3. 队头阻塞 → 一个包丢，后面全等
4. 无优先级 → 关键数据和普通数据一样处理
```

### KCP / WebRTC 的问题

```
KCP 设计目标: UDP 上的可靠传输
KCP 问题:
1. 字节级全可靠 → 移动/AOI 等可覆盖数据也被重传
2. ARQ 无 deadline → 丢包可能无限等待
3. 单一队列 → 关键数据被普通数据阻塞
4. 无多路径 → 5G + WiFi 无法协同

WebRTC DataChannel:
1. SCTP/DTLS 栈深 → 建连慢、队头阻塞
2. 为音视频设计 → 无游戏语义分层
3. 可选 FEC 常驻 → 5G 上带宽浪费
```

### QCP 的创新

```
QCP 设计目标: 新时代 UDP 可靠协议 — 游戏 / IoT 超低延迟 + 高保证
核心抽象: TLB（Time-Latency Bounded）语义交付

1. 三通道语义
   - REALTIME: 游戏移动 / IoT 遥测 → 最新值覆盖，零恢复延迟
   - CRITICAL: 射击 / 设备指令 → deadline 内 Fast NACK
   - BATCH: 聊天 / OTA 固件 → 强可靠 ARQ

2. Recovery Policy（按需激活，非常驻冗余）
   - 多路径 Race > Fast NACK > Network Coding > ARQ

3. 5G/6G 多路径 + 0-RTT

4. Lock-Free 100K+ 并发 + 9B 包头
```

---

## 诚实的利弊分析

### QCP 的代价

| 代价 | 说明 |
|------|------|
| **语义分层要求** | 应用需区分 STATE / EVENT / BATCH |
| **实现复杂度** | 比 KCP 高（TLB + 多路径 + Policy 引擎） |
| **多路径双发** | Race 模式有带宽成本（通常优于常驻 FEC） |
| **按需纠删 CPU** | 仅在 burst 丢包时触发 Network Coding |

### 适用场景

```
QCP 适合:
✓ 游戏：MOBA / FPS / MMO / 格斗
✓ IoT：传感器、智能家居、工业设备、车联网
✓ 5G / WiFi 7 / 边缘 MEC
✓ 需要 UDP 上同时「低延迟 + 可靠」的场景

QCP 不适合:
✗ WebRTC 音视频互通（请用 WebRTC 媒体通道）
✗ 大块文件传输（请用 TCP / QUIC）
```

### 实际生产建议

```
1. REALTIME 通道默认不可靠 + 最新覆盖（不要给移动包加 FEC）
2. CRITICAL 事件设 deadline（如 8ms），5G 上 1-RTT 重传通常足够
3. 手机端优先开 Multi-Path Race（WiFi + 5G），再考虑纠删
4. Network Coding 仅在单路径 + burst 丢包时按需启用
```

### 协议层边界

QCP 专注协议层，不内置玩家、房间、匹配、战斗判定等业务概念。业务只声明消息的 `stream`、`deadline`、`priority`；协议负责 session resume、断线恢复、NAT/IP/端口变化后的 connection migration、多路径切换、ACK/NACK、deadline 有界重传、拥塞控制、防重放和指标采集。

| 归属 | 应该负责的东西 |
|------|----------------|
| QCP 协议层 | SessionID / Resume Token、PathID、多路径探测、切网迁移、Fast NACK、ARQ、按需冗余、拥塞控制、防重放、P50/P99/loss 指标 |
| 业务层 | 玩家身份、房间状态、匹配、战斗逻辑、是否补状态快照、业务幂等和权限 |

详细落地建议见 [docs/PROTOCOL.md](docs/PROTOCOL.md)。

---

## FAQ

### Q: FEC 是最优解吗？

```
不是。FEC 是「用带宽换延迟」的策略，不是通用最优。

5G/6G 环境:
- RTT 下降 → 1-RTT 重传成本从 50ms 降到 5-10ms
- 多路径普及 → WiFi + 5G Race 优于常驻 FEC
- 80% 游戏流量是状态 → 最新覆盖即可，无需恢复

QCP 方案: TLB 语义交付 + 按需 Recovery
- REALTIME: 不恢复，取最新
- CRITICAL: deadline 内有界 ARQ
- 纠删: 仅单路径 + burst 时临时启用
```

### Q: WebRTC 用了 FEC，为什么还慢？

```
WebRTC 慢的主因不是 FEC:
1. ICE/STUN/TURN 建连 → 秒级
2. SCTP over DTLS → 栈深、队头阻塞
3. 媒体 Jitter Buffer → 20-200ms
4. 无游戏语义分层 → 所有数据同等对待

QCP 与 WebRTC 定位不同:
- WebRTC: 音视频通话 / 直播
- QCP: 游戏 tick 级状态 + 有界事件
```

### Q: 5G/6G 还需要应用层纠错吗？

```
视场景而定:

5G SA (RTT ~10ms):  CRITICAL 走 Fast NACK 通常足够
5G + MEC (RTT ~5ms): 多路径 Race 优先， rarely 需要纠删
6G (RTT ~1ms):        空口 URLLC + 语义覆盖 > 应用层 FEC

原则: 多路径 > 有界 ARQ > 按需 Network Coding > 固定 FEC
```

---

## 架构图

### 协议分层

```
┌─────────────────────────────────────────────────────────────┐
│                    游戏 / IoT 应用层                         │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐          │
│  │ 战斗/控制│ │ 移动/遥测│ │ 技能/指令│ │ OTA/日志│          │
│  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘          │
├───────┼───────────┼───────────┼───────────┼─────────────────┤
│       │    QCP API Layer       │           │                │
│       │  ┌─────────────────────┴───────────┴────┐          │
│       │  │     Send(data, stream, deadline)       │          │
│       │  │     Recv() → {data, stream, path, seq} │          │
│       │  └──────────────────────────────────────┘          │
├───────┼─────────────────────────────────────────────────────┤
│       │         TLB Semantic Scheduler                      │
│  ┌────┴────────────────────────────────────────────────┐   │
│  │ REALTIME: LatestWins │ CRITICAL: BoundedARQ         │   │
│  │ BATCH: FullARQ       │ deadline 超时 → 预测/放弃    │   │
│  └────────────────────────────────────────────────────┘   │
├───────┼─────────────────────────────────────────────────────┤
│       │         Recovery Policy Engine（按需）              │
│  ┌────┴────────────────────────────────────────────────┐   │
│  │ Multi-Path Race → Fast NACK → Network Coding → ARQ │   │
│  └────────────────────────────────────────────────────┘   │
├───────┼─────────────────────────────────────────────────────┤
│       │         QCP Core Layer                              │
│  ┌────┴────────────────────────────────────────────────┐   │
│  │  Multi-Path │ AI Congestion │ 0-RTT │ Lock-Free   │   │
│  │  Ring Buffer 64KB │ CRC32 Checksum                  │   │
│  └────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐   │
│  │              QCP Packet (8-12 bytes)                 │   │
│  │  ┌───────┬───────┬───────┬───────┬───────┐        │   │
│  │  │ Type  │Stream │ SeqID │PathID │Payload│        │   │
│  │  │ 1byte │ 1byte │ 2byte │ 1byte │ var   │        │   │
│  │  └───────┴───────┴───────┴───────┴───────┘        │   │
│  └─────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│  WiFi 6E/7 │ 5G SA │ 5G mmWave │ ... │ 6G              │
└─────────────────────────────────────────────────────────────┘
```

### TLB 可靠性模型

```
┌─────────────────────────────────────────────────────────────┐
│                 QCP TLB 语义交付                             │
├─────────────────────────────────────────────────────────────┤
│  REALTIME (移动/AOI)                                        │
│  发送 tick N → 丢包 → tick N+1 覆盖 → 无需恢复             │
├─────────────────────────────────────────────────────────────┤
│  CRITICAL (射击/技能)                                       │
│  发送事件 → 丢包 → Fast NACK → 1-RTT 重传 (5G ~5-10ms)    │
│           → deadline 超时 → 放弃，走预测                     │
├─────────────────────────────────────────────────────────────┤
│  Recovery Policy（仅策略引擎判定需要时）                     │
│  双路径 Race > Fast NACK > Network Coding > ARQ             │
└─────────────────────────────────────────────────────────────┘
```

---

## 仓库结构

### 核心

| 仓库 | 用途 |
|------|------|
| [QCP](https://github.com/neko233-com/QCP) | 主仓库（文档、规范） |
| [qcp-lib-go](https://github.com/neko233-com/qcp-lib-go) | **Go 参考实现 — 协议验证与核心开发语言** |
| [qcp-core](https://github.com/neko233-com/qcp-core) | C/C++ 生产核心 |
| [qcp-iot](https://github.com/neko233-com/qcp-iot) | IoT 嵌入式集成（ESP32 / STM32 等） |
| [qcp-benchmark](https://github.com/neko233-com/qcp-benchmark) | 压测（KCP 仅作对照基线） |

### 主流语言绑定

**以 Go 为规范参考实现**，其他语言绑定与 Go API 对齐，仅维护主流生态：

| 仓库 | 语言 | 用途 |
|------|------|------|
| [qcp-lib-go](https://github.com/neko233-com/qcp-lib-go) | **Go** | **参考实现 / 协议验证** |
| [qcp-lib-c](https://github.com/neko233-com/qcp-lib-c) | C | 底层 / 嵌入式 |
| [qcp-lib-cpp](https://github.com/neko233-com/qcp-lib-cpp) | C++ | UE / 底层 |
| [qcp-lib-csharp](https://github.com/neko233-com/qcp-lib-csharp) | C# | Unity |
| [qcp-lib-rust](https://github.com/neko233-com/qcp-lib-rust) | Rust | 高性能安全 |
| [qcp-lib-java](https://github.com/neko233-com/qcp-lib-java) | Java | Android / 服务端 |
| [qcp-lib-kotlin](https://github.com/neko233-com/qcp-lib-kotlin) | Kotlin | Android / JVM |
| [qcp-lib-swift](https://github.com/neko233-com/qcp-lib-swift) | Swift | iOS / macOS |
| [qcp-lib-typescript](https://github.com/neko233-com/qcp-lib-typescript) | TypeScript | WebGL / Node.js |
| [qcp-lib-erlang](https://github.com/neko233-com/qcp-lib-erlang) | Erlang | 分布式游戏服务器 |
| [qcp-lib-lua](https://github.com/neko233-com/qcp-lib-lua) | Lua | 游戏脚本 |
| [qcp-lib-verse](https://github.com/neko233-com/qcp-lib-verse) | Verse | UE6 原生 |

### 游戏集成

游戏引擎通过语言绑定接入 QCP，无需额外插件仓库：

| 引擎 | 绑定仓库 | 说明 |
|------|----------|------|
| Unity | [qcp-lib-csharp](https://github.com/neko233-com/qcp-lib-csharp) | C# API，可直接嵌入 Unity 项目 |
| Unreal Engine | [qcp-lib-cpp](https://github.com/neko233-com/qcp-lib-cpp) | C++ API，可封装为 UE Module / Blueprint |
| UE6 Verse | [qcp-lib-verse](https://github.com/neko233-com/qcp-lib-verse) | Verse 原生绑定 |
| WebGL | [qcp-lib-typescript](https://github.com/neko233-com/qcp-lib-typescript) | TypeScript / WebAssembly |

---

## 协议规范

| 文档 | 说明 |
|------|------|
| [PROTOCOL.md](docs/PROTOCOL.md) | QCP 2.0 协议规范（TLB、游戏/IoT、5G/6G） |
| [BASELINE.md](docs/BASELINE.md) | vs KCP 压测基线对照（KCP 非 QCP 组件） |

**QCP 2.0** 是当前规范：TLB 语义交付为核心，Recovery Policy 按需激活。

---

## License

MIT License
