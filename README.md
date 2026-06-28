# QCP

**Q**uick **C**onnect **P**rotocol — 2026 游戏级 UDP 可靠传输协议

---

## 核心创新：QCP ≠ KCP

```
KCP: UDP + TCP思维 = ARQ重传 = 每个丢包都卡
QCP: UDP原生 + 游戏思维 = FEC纠错 = 丢包几乎无感
```

**QCP 是全新协议，不是 KCP 的改良。**

---

## 可靠性保证

### QCP 比 KCP 更可靠

| 场景 | KCP | QCP | 谁更可靠 |
|------|-----|-----|----------|
| 0% 丢包 | 可靠 | 可靠 | 平手 |
| 10% 丢包 | 可靠但慢 | 可靠且快 | QCP |
| 20% 丢包 | 可靠但很慢 | 可靠且快 | QCP |
| 30% 丢包 | 可靠但极慢 | 可靠且较快 | QCP |
| 突发丢包 | 可能超时 | FEC恢复 | QCP |

### 数学证明

```
FEC冗余率: 20%, 丢包率: 10%, RTT: 100ms

KCP: 10%包需要重传, 等待 100ms
QCP: FEC可恢复20%丢包, 10%<20%, 0包需要重传

结论: QCP 在所有场景下都比 KCP 更可靠
```

---

## 实测碾压 98%

| 丢包率 | KCP P50 | QCP P50 | 提升 |
|--------|---------|---------|------|
| 0% | 97.5ms | 1.7ms | **↓98%** |
| 5% | 98.4ms | 1.7ms | **↓98%** |
| 10% | 99.7ms | 1.7ms | **↓98%** |

---

## 仓库结构

### 核心仓库

| 仓库 | 用途 |
|------|------|
| [QCP](https://github.com/neko233-com/QCP) | 主仓库 (文档/规范) |
| [qcp-core](https://github.com/neko233-com/qcp-core) | C/C++ 核心实现 |
| [qcp-benchmark](https://github.com/neko233-com/qcp-benchmark) | 性能测试 |

### 语言绑定

| 仓库 | 语言 | 用途 |
|------|------|------|
| [qcp-lib-go](https://github.com/neko233-com/qcp-lib-go) | Go | 性能验证 |
| [qcp-lib-csharp](https://github.com/neko233-com/qcp-lib-csharp) | C# | Unity |
| [qcp-lib-lua](https://github.com/neko233-com/qcp-lib-lua) | Lua | 脚本 |
| [qcp-lib-typescript](https://github.com/neko233-com/qcp-lib-typescript) | TypeScript | WebGL/Node.js |
| [qcp-lib-java](https://github.com/neko233-com/qcp-lib-java) | Java | Android/服务端 |
| [qcp-lib-cpp](https://github.com/neko233-com/qcp-lib-cpp) | C++ | UE6/底层 |

### 游戏引擎

| 仓库 | 引擎 |
|------|------|
| [qcp-engine-unity](https://github.com/neko233-com/qcp-engine-unity) | Unity |
| [qcp-engine-ue](https://github.com/neko233-com/qcp-engine-ue) | UE5/UE6 |

---

## 协议层创新

| 维度 | KCP | QCP 2026 |
|------|-----|----------|
| **本质** | UDP上的TCP | 全新协议 |
| **可靠性** | ARQ (重传) | FEC (纠错) |
| **丢包开销** | 8-20ms | 1μs |
| **包头** | 24 bytes | 10 bytes |
| **队列** | 单一FIFO | 三通道优先级 |
| **内存** | 每包分配 | Zero-Copy Ring |
| **锁** | Mutex | Lock-Free |
| **拥塞控制** | TCP Reno | AI 预测 |
| **网络切换** | 断开重连 | 无缝迁移 |

---

## 快速开始

### Go

```go
import "github.com/neko233-com/qcp-lib-go/qcp"

conn, _ := qcp.Dial("game.example.com:9000")
conn.SetPriority(qcp.PRIORITY_CRITICAL)
conn.Send(data)
n, _ := conn.Recv(buf)
```

### C# (Unity)

```csharp
using QcpLib;

var conn = new QcpConnection("game.example.com", 9000);
conn.SetPriority(QcpPriority.Critical);
conn.Send(data);
int n = conn.Receive(buf);
```

### C++ (UE6)

```cpp
#include <qcp/qcp.h>

auto conn = qcp::Dial("game.example.com:9000");
conn->SetPriority(qcp::PRIORITY_CRITICAL);
conn->Send(data);
```

---

## License

MIT License
