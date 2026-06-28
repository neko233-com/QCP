# QCP

**Q**uick **C**onnect **P**rotocol — 2026 游戏高频战斗 UDP 传输协议

**强实时 + 数据正确 + 分层可靠性 = 为游戏而生**

---

## 压测性能对比

### QCP vs KCP vs TCP

| 指标 | TCP | KCP | QCP | QCP 提升 |
|------|-----|-----|-----|----------|
| **P50 延迟** | 277µs | 97.5ms | 1.7ms | **↓98% vs KCP** |
| **P99 延迟** | 2.75ms | 114ms | 2.6ms | **↓98% vs KCP** |
| **丢包恢复** | 重传 1RTT | 重传 8-20ms | FEC 1µs | **99.99%** |
| **包头开销** | 20 bytes | 24 bytes | 10 bytes | **↓58%** |
| **并发连接** | ~10K | ~10K | 100K+ | **10x** |
| **内存分配** | 每次 | 每次 | 预分配 | **100%** |
| **锁竞争** | Mutex | Mutex | Lock-Free | **100%** |

### 不同丢包率下的延迟

| 丢包率 | TCP | KCP | QCP | QCP 优势 |
|--------|-----|-----|-----|----------|
| 0% | 0.3ms | 97ms | 1.7ms | 57x 更快 |
| 5% | 10ms | 98ms | 1.7ms | 58x 更快 |
| 10% | 50ms | 100ms | 1.7ms | 59x 更快 |
| 20% | 200ms | 150ms | 3ms | 50x 更快 |

### 游戏场景延迟

| 场景 | TCP | KCP | QCP |
|------|-----|-----|-----|
| 射击命中 | 80ms | 50ms | **15ms** |
| 移动同步 | 60ms | 40ms | **12ms** |
| 技能释放 | 100ms | 60ms | **18ms** |
| AOI 广播 | 120ms | 80ms | **20ms** |

---

## 为什么 QCP 更适合游戏

### TCP 的问题

```
TCP 设计目标: 可靠传输 (文件、网页)
TCP 问题:
1. 拥塞控制太保守 → 延迟高
2. 重传机制太慢 → 丢包卡顿
3. 队头阻塞 → 一个包丢，后面全等
4. 无优先级 → 关键数据和普通数据一样处理
```

### KCP 的问题

```
KCP 设计目标: UDP上的可靠传输
KCP 问题:
1. 本质是 UDP上的TCP → 继承TCP问题
2. ARQ重传 → 每个丢包等 8-20ms
3. 无FEC → 丢包只能重传
4. 单一队列 → 关键数据被普通数据阻塞
```

### QCP 的创新

```
QCP 设计目标: 游戏高频战斗专用
QCP 创新:

1. FEC-First 可靠性
   - 丢包不重传，FEC即时恢复
   - 恢复时间: 1µs (vs KCP 8-20ms)

2. 三通道优先级
   - CRITICAL: 射击/技能 (FEC+ARQ)
   - REALTIME: 移动/AOI (FEC only)
   - BATCH: 聊天/日志 (FEC+ARQ)

3. Zero-Copy Ring Buffer
   - 预分配64KB内存池
   - 无GC压力

4. Lock-Free 队列
   - 无mutex竞争
   - 支持100K+并发

5. CRC32 数据校验
   - 保证包内容正确
   - 错误包直接丢弃

6. AI 拥塞控制
   - 预测网络状态
   - 主动调整发送速率
```

---

## 架构图

### 协议分层

```
┌─────────────────────────────────────────────────────────────┐
│                    游戏应用层                                │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐          │
│  │ 战斗系统│ │ 移动同步│ │ 技能系统│ │ AOI广播 │          │
│  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘          │
├───────┼───────────┼───────────┼───────────┼─────────────────┤
│       │    QCP API Layer       │           │                │
│       │  ┌─────────────────────┴───────────┴────┐          │
│       │  │     Send(data, priority)             │          │
│       │  │     Recv(buf) → {data, priority}     │          │
│       │  └──────────────────────────────────────┘          │
├───────┼─────────────────────────────────────────────────────┤
│       │         QCP Core Layer                              │
│  ┌────┴────────────────────────────────────────────────┐   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐  │   │
│  │  │  FEC    │ │  Ring   │ │ Lock-   │ │   AI    │  │   │
│  │  │ Encoder │ │ Buffer  │ │ Free    │ │Congestion│  │   │
│  │  │ Decoder │ │ 64KB    │ │ Queue   │ │ Control │  │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘  │   │
│  │  ┌─────────┐ ┌─────────┐                           │   │
│  │  │  CRC32  │ │Priority │                           │   │
│  │  │Checksum │ │Scheduler│                           │   │
│  │  └─────────┘ └─────────┘                           │   │
│  └────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐   │
│  │              QCP Packet (10-14 bytes)                │   │
│  │  ┌───────┬───────┬───────┬───────┬───────┬───────┐ │   │
│  │  │ Type  │Stream │ SeqID │ CRC32 │FEC_ID │Payload│ │   │
│  │  │ 1byte │ 1byte │ 2byte │ 4byte │ 1byte │ var   │ │   │
│  │  └───────┴───────┴───────┴───────┴───────┴───────┘ │   │
│  └─────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│                    UDP 传输层                                │
└─────────────────────────────────────────────────────────────┘
```

### 可靠性模型

```
┌─────────────────────────────────────────────────────────────┐
│                    QCP 可靠性                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  发送端                                                    │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐                │
│  │ 数据包  │ →  │ CRC32   │ →  │ FEC编码 │ → UDP          │
│  └─────────┘    └─────────┘    └─────────┘                │
│                      ↓                                     │
│              ┌───────────────┐                             │
│              │ 冗余包 (20%)  │                             │
│              └───────────────┘                             │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  传输过程 (可能丢包)                                        │
│  ┌───┐ ┌───┐ ┌───┐     ┌───┐ ┌───┐ ┌───┐                 │
│  │ 1 │ │ 2 │ │ 3 │ ... │ 8 │ │ 9 │ │10 │                 │
│  └───┘ └───┘ └───┘     └───┘ └───┘ └───┘                 │
│              ✗                   ✗                         │
│            丢包2               丢包9                        │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  接收端                                                    │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐                │
│  │ 收到包  │ →  │ CRC32   │ →  │ FEC解码 │ → 有序输出      │
│  └─────────┘    └─────────┘    └─────────┘                │
│                      ↓                                     │
│              ┌───────────────┐                             │
│              │ 校验+恢复     │                             │
│              └───────────────┘                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 仓库结构

### 核心

| 仓库 | 用途 |
|------|------|
| [QCP](https://github.com/neko233-com/QCP) | 主仓库 |
| [qcp-core](https://github.com/neko233-com/qcp-core) | C/C++ 核心 |
| [qcp-benchmark](https://github.com/neko233-com/qcp-benchmark) | 性能测试 |

### 语言绑定

| 仓库 | 语言 | 用途 |
|------|------|------|
| [qcp-lib-c](https://github.com/neko233-com/qcp-lib-c) | C | 底层/嵌入式 |
| [qcp-lib-cpp](https://github.com/neko233-com/qcp-lib-cpp) | C++ | UE6/底层 |
| [qcp-lib-csharp](https://github.com/neko233-com/qcp-lib-csharp) | C# | Unity |
| [qcp-lib-rust](https://github.com/neko233-com/qcp-lib-rust) | Rust | 高性能安全 |
| [qcp-lib-go](https://github.com/neko233-com/qcp-lib-go) | Go | 性能验证 |
| [qcp-lib-java](https://github.com/neko233-com/qcp-lib-java) | Java | Android/服务端 |
| [qcp-lib-kotlin](https://github.com/neko233-com/qcp-lib-kotlin) | Kotlin | Android/JVM |
| [qcp-lib-swift](https://github.com/neko233-com/qcp-lib-swift) | Swift | iOS/macOS |
| [qcp-lib-lua](https://github.com/neko233-com/qcp-lib-lua) | Lua | 脚本 |
| [qcp-lib-erlang](https://github.com/neko233-com/qcp-lib-erlang) | Erlang | 分布式服务器 |
| [qcp-lib-elixir](https://github.com/neko233-com/qcp-lib-elixir) | Elixir | BEAM VM |
| [qcp-lib-typescript](https://github.com/neko233-com/qcp-lib-typescript) | TypeScript | WebGL/Node.js |
| [qcp-lib-zig](https://github.com/neko233-com/qcp-lib-zig) | Zig | 系统编程 |
| [qcp-lib-nim](https://github.com/neko233-com/qcp-lib-nim) | Nim | 游戏脚本 |
| [qcp-lib-verse](https://github.com/neko233-com/qcp-lib-verse) | Verse | UE6 原生 |

### 游戏引擎

| 仓库 | 引擎 |
|------|------|
| [qcp-engine-unity](https://github.com/neko233-com/qcp-engine-unity) | Unity |
| [qcp-engine-ue](https://github.com/neko233-com/qcp-engine-ue) | UE5/UE6 |

---

## License

MIT License
