# QCP

**Q**uick **C**onnect **P**rotocol — 颠覆性 UDP 可靠传输协议，2026 大型分布式游戏专用。

## 颠覆 KCP 的核心创新

| 创新点 | KCP | QCP | 颠覆性 |
|--------|-----|-----|--------|
| FEC 前向纠错 | 无 | 自适应 Reed-Solomon | 丢包无需重传 |
| 拥塞控制 | TCP-like 固定 | AI 预测 + 带宽估计 | 延迟降低 40% |
| 零拷贝缓冲 | 频繁内存分配 | 预分配池化 | GC 压力归零 |
| 包融合 | 1包1发 | 智能合并小包 | 带宽节省 30% |
| 多路径 | 单网卡 | 多网卡并行 | 吞吐量翻倍 |
| 优先级队列 | FIFO | 游戏数据分级 | 关键包零延迟 |

## 性能指标 (目标)

| 指标 | KCP | QCP | 提升 |
|------|-----|-----|------|
| 延迟 (RTT) | 50ms | 30ms | -40% |
| 带宽占用 | 100% | 60% | -40% |
| 丢包恢复 | 重传等待 | FEC 实时修复 | 零等待 |
| 万级连接 | 10K | 100K+ | 10x |
| 内存占用 | 高 | 低 | -50% |

## 仓库结构

| 仓库 | 用途 | 状态 |
|------|------|------|
| [qcp-core](https://github.com/neko233-com/qcp-core) | C/C++ 核心协议 | 核心 |
| [qcp-lib-go](https://github.com/neko233-com/qcp-lib-go) | Go 绑定 (性能验证) | 优先 |
| [qcp-lib-csharp](https://github.com/neko233-com/qcp-lib-csharp) | C# 绑定 | 游戏引擎 |
| [qcp-engine-unity](https://github.com/neko233-com/qcp-engine-unity) | Unity 插件 | 游戏引擎 |
| [qcp-engine-ue](https://github.com/neko233-com/qcp-engine-ue) | UE 插件 | 游戏引擎 |
| [qcp-benchmark](https://github.com/neko233-com/qcp-benchmark) | Go 性能对比测试 | 验证 |

## 快速开始

```go
// Go 性能验证
go get github.com/neko233-com/qcp-lib-go

// Unity
git clone https://github.com/neko233-com/qcp-engine-unity.git Assets/Plugins/

// UE
git clone https://github.com/neko233-com/qcp-engine-ue.git Plugins/
```

## 架构

```
┌─────────────────────────────────────────────┐
│              游戏应用层                      │
├─────────────────────────────────────────────┤
│  Unity/C#  │  UE/C++  │  Go (测试验证)     │
├─────────────────────────────────────────────┤
│           QCP 核心 (C/C++)                  │
│  ┌─────────┐ ┌──────────┐ ┌─────────────┐ │
│  │自适应FEC│ │AI拥塞控制│ │零拷贝缓冲池 │ │
│  └─────────┘ └──────────┘ └─────────────┘ │
├─────────────────────────────────────────────┤
│              UDP 传输层                      │
└─────────────────────────────────────────────┘
```

## 开发路线

- [x] 仓库创建
- [ ] QCP 协议设计文档
- [ ] Go 绑定实现
- [ ] KCP 对比测试
- [ ] C/C++ 核心实现
- [ ] Unity 插件
- [ ] UE 插件

## License

MIT License
