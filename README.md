# QCP

**Q**uick **C**onnection **P**rotocol — Ultra-high-performance UDP protocol for 2026 distributed gaming.

## Why QCP?

QCP is designed to replace KCP for modern large-scale distributed games:

- **Faster** — Lower latency than KCP with optimized congestion control
- **Less bandwidth** — Smarter FEC and compression reduces traffic by 30-50%
- **Multi-platform** — Native support for Go, Java, C++, C#
- **Game-engine ready** — Unity, Unreal Engine, WebGL, IoT

## Repositories

| Repository | Description |
|------------|-------------|
| [qcp-core](https://github.com/neko233-com/qcp-core) | Core protocol implementation (C/C++) |
| [qcp-lib-go](https://github.com/neko233-com/qcp-lib-go) | Go binding |
| [qcp-lib-java](https://github.com/neko233-com/qcp-lib-java) | Java binding |
| [qcp-lib-csharp](https://github.com/neko233-com/qcp-lib-csharp) | C# binding |
| [qcp-engine-unity](https://github.com/neko233-com/qcp-engine-unity) | Unity plugin |
| [qcp-engine-ue](https://github.com/neko233-com/qcp-engine-ue) | Unreal Engine plugin |
| [qcp-engine-webgl](https://github.com/neko233-com/qcp-engine-webgl) | WebGL/WebAssembly binding |
| [qcp-iot](https://github.com/neko233-com/qcp-iot) | IoT integration |

## Performance Comparison

| Metric | KCP | QCP |
|--------|-----|-----|
| Latency (RTT) | ~50ms | ~30ms |
| Bandwidth usage | 100% | 60-70% |
| Packet loss recovery | 2x retransmit | Adaptive FEC |
| Max connections | ~10K | ~100K |

## Quick Start

```bash
# Go
go get github.com/neko233-com/qcp-lib-go

# Java (Maven)
<dependency>
    <groupId>com.neko233</groupId>
    <artifactId>qcp-lib-java</artifactId>
    <version>1.0.0</version>
</dependency>

# C# (NuGet)
dotnet add package QcpLib.CSharp
```

## Architecture

```
┌─────────────────────────────────────────────┐
│                 Application                 │
├─────────────────────────────────────────────┤
│  Language Bindings (Go/Java/C++/C#/etc.)   │
├─────────────────────────────────────────────┤
│           QCP Core Protocol                │
│  ┌─────────┐ ┌─────────┐ ┌─────────────┐  │
│  │  FEC    │ │Congestion│ │  Reliable   │  │
│  │ Engine  │ │ Control  │ │  Transfer   │  │
│  └─────────┘ └─────────┘ └─────────────┘  │
├─────────────────────────────────────────────┤
│              UDP Transport                  │
└─────────────────────────────────────────────┘
```

## License

MIT License

## Contributing

See individual repositories for contribution guidelines.
