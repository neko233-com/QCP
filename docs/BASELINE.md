# QCP vs KCP — 压测基线对照

> **KCP 不是 QCP 的组件，也不是 QCP 的依赖。**  
> `qcp-benchmark/baseline_kcp.go` 中的 KCP 代码**仅用于压测对照**，模拟 legacy 协议行为，方便量化 QCP 的延迟优势。

---

## 定位差异

| | KCP (2012) | QCP (2026+) |
|---|-----------|-------------|
| 定位 | UDP 上的可靠传输 | **新时代 UDP 可靠协议** |
| 场景 | 通用 | **游戏 / IoT 专用** |
| 延迟模型 | ARQ 等 RTO (8-20ms) | **TLB + Fast NACK (<10ms)** |
| 语义 | 字节级全可靠 | **REALTIME / CRITICAL / BATCH** |
| 多路径 | 无 | **5G + WiFi 原生** |
| 并发 | ~10K Mutex | **100K+ Lock-Free** |
| 包头 | 24B | **9B** |

**QCP 不是 KCP 超集——是完全不同的协议架构，在延迟、语义、扩展性上全面超越。**

---

## 压测结论（qcp-benchmark）

| 指标 | KCP (基线) | QCP |
|------|-----------|-----|
| P50 延迟 | ~97ms | **~1.7ms** |
| P99 延迟 | ~114ms | **~2.6ms** |
| 丢包恢复 | RTO 8-20ms | **Fast NACK + 1-RTT** |
| 游戏移动包 | 可靠重传浪费 | **REALTIME 最新覆盖** |
| IoT 遥测 | 单一队列阻塞 | **CRITICAL 指令优先** |

运行压测：

```bash
cd qcp-benchmark
go run . -mode all -duration 5s -loss 0.05
```

---

## 从 KCP 项目迁移

如果你正在维护基于 KCP 的老项目，QCP 提供原生 API，无需引入 KCP 库：

```go
conn, _ := qcp.Dial("server:9000")
conn.ConfigureARQ(qcp.ARQConfig{NoDelay: true, Interval: 10, MTU: 1400})
conn.Send(data)
conn.Recv(buf)
```

迁移后建议按语义拆分通道，释放 QCP 的全部延迟优势（见 [PROTOCOL.md](PROTOCOL.md)）。

---

*KCP 留在 2012。QCP 为 2026+ 游戏与 IoT 而生。*
