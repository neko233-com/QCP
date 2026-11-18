# QCP

English · [简体中文](README-CN.md)

QCP is a UDP-based transport protocol for latency-sensitive applications. It supports both client-to-server connections and server-to-server links. [The Go library](https://github.com/neko233-com/qcp-lib-go) is the reference implementation.

## Where to use QCP

| Topology | Common uses | Delivery requirements |
| --- | --- | --- |
| **Client → server** | Player input, position snapshots, telemetry | Use latest-state delivery when a new update can replace an older one. |
| **Server ↔ server** | Game gateway backhaul, cross-region game RPC, live event streams, state replication | Use reliable ordered delivery for RPC and business events. For replaceable snapshots, latest-state delivery avoids retrying obsolete updates. |

QCP is particularly useful for server-to-server links where you control both endpoints, can enable UDP, and can choose delivery semantics per traffic type. On a stable LAN, WebSocket/TCP is a strong baseline. On lossy paths, compare QCP using the same payload, delivery guarantees, and concurrency. For reliable RPC, track timeouts and delivery rates alongside P99. Application-level retries still need request IDs and idempotency.

### Delivery semantics

| Mode | Intended behavior |
| --- | --- |
| **REALTIME** | Deliver newer state when available. Older or lost updates may be skipped. Keep each update below the datagram payload limit. |
| **CRITICAL** | Deadline-aware recovery in the protocol design. Check deadline support in the binding before depending on it. |
| **BATCH** | Reliable, ordered delivery. Use for commands and events that the receiver must process in sequence. |

The raw Go transport exposes `Dial`, `Listen`, `Send`, and `RecvWait`. Reliable payloads larger than the MTU arrive as ordered chunks. Applications needing message boundaries should use a length prefix or another message-framing layer.

For Go server-to-server links, use `qcp.ListenStream` and `qcp.DialStreamContext` to obtain a `net.Listener` and `net.Conn`. Wrap the stream in TLS, then use your existing RPC or message framing. For service identity, require client certificates on the listener and verify the server certificate on the dialer.

## Network and deployment considerations

- QCP requires a UDP-capable runtime and a UDP path between peers. Native runtimes such as Go can open UDP sockets. Standard browser JavaScript and browser-hosted Unity WebGL have no raw UDP socket API. WeChat Mini Games use the platform UDP API through the Unity binding where available.
- Use a private network or authenticated tunnel for backhaul. Use TLS on an ordered byte stream or another authenticated encryption layer for sensitive traffic. Keep certificate verification enabled and restrict the UDP listener to trusted networks.
- Provision UDP forwarding and monitor loss, retransmissions, timeouts, and reconnects. Keep a WebSocket/TCP configuration available where UDP is blocked or its tail latency regresses.
- A single reliable ordered stream can still block behind a missing packet. Latest-state delivery helps only when the application can safely skip the skipped update.

## Latency comparison

The following values come from a **simulated model**. They describe modeled delivery latency under selected assumptions. Use real socket measurements to compare WebSocket, TCP, KCP, and QCP for a deployment. [Benchmark methodology and report](https://github.com/neko233-com/qcp-benchmark/blob/main/%E6%80%A7%E8%83%BD%E5%AF%B9%E6%AF%94%E8%A1%A8%E6%A0%BC.md).

| Profile | **`QCP` P50** | **`QCP` P99** | TCP P50 | TCP P99 | KCP P50 | KCP P99 |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| LAN | **950 µs** | **999 µs** | 1.583 ms | 1.665 ms | 4.2 ms | 5.294 ms |
| Wi-Fi | **9.505 ms** | **17.001 ms** | 15.842 ms | 50.001 ms | 22.203 ms | 24.216 ms |
| 4G | **23.521 ms** | **31.5 ms** | 39.201 ms | 104.166 ms | 52.206 ms | 65.299 ms |
| 3G | **70.658 ms** | **80.301 ms** | 117.763 ms | 285.495 ms | 152.215 ms | 181.219 ms |
| Congested | **49.445 ms** | **60.2 ms** | 82.408 ms | 202.003 ms | 102.233 ms | 132.26 ms |
| Extreme | **144.598 ms** | **158.198 ms** | 241.672 ms | 565.334 ms | 302.275 ms | 333.219 ms |

## Repositories and specifications

| Component | Purpose |
| --- | --- |
| [qcp-lib-go](https://github.com/neko233-com/qcp-lib-go) | Go reference implementation and transport tests |
| [qcp-lib-unity](https://github.com/neko233-com/qcp-lib-unity) | Unity package and WeChat Mini Game UDP bridge |
| [qcp-benchmark](https://github.com/neko233-com/qcp-benchmark) | Benchmark harness and modeled comparison |
| [`.gitmodules`](.gitmodules) | Tracked bindings and component repositories. Check each component for implementation and interoperability status. |

- [Protocol specification](docs/PROTOCOL.md)
- [Production architecture and roadmap](docs/PRODUCTION_ARCHITECTURE.md)
- [KCP baseline methodology](docs/BASELINE.md)

## License

MIT.
