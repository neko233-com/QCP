# QCP Production Protocol Architecture

QCP is a transport protocol for latency-sensitive games and real-time systems. It must stay below business logic: no player model, room model, match state, combat rules, inventory, chat moderation, or account logic belongs in the protocol. The protocol provides generic delivery semantics, path management, recovery, congestion control, security, and observability.

This document defines the production-oriented structure that QCP implementations should converge on.

---

## 1. Design Goals

| Goal | Requirement |
|------|-------------|
| Low tail latency | Optimize P50 and P99, not only average latency. |
| Semantic delivery | REALTIME, CRITICAL, and BATCH must use different recovery rules. |
| No global head-of-line blocking | Loss in BATCH must not block REALTIME or CRITICAL. |
| Network migration | WiFi / cellular / NAT port changes must be handled by the protocol. |
| Production security | Packets require authentication, anti-replay, key epoch, and token binding. |
| Extensibility | Unknown extensions must be skippable by older implementations. |
| Hot-path performance | No heap allocation on packet send/receive hot path after warmup. |
| Operational visibility | RTT, loss, retransmit, migration, resume, P50/P99 metrics must be exported. |

---

## 2. Layering

```
Game / App Protocol
  - player, room, battle, command schema, snapshots, permissions

QCP API
  - Send(payload, stream, deadline, priority, options)
  - Recv() -> Message{payload, stream, seq, path, flags}
  - Events: OnResume, OnPathChange, OnTimeout, OnDegraded

Semantic Scheduler
  - REALTIME: latest-wins, drop old state
  - CRITICAL: bounded reliable, deadline-aware Fast NACK
  - BATCH: ordered reliable, background ARQ

Recovery Policy Engine
  - choose Race / Fast NACK / ARQ / coding based on stream, deadline, path stats

Session + Path Manager
  - SessionID, ConnectionID, Resume Token, PathID, migration, multi-path probing

Packet Engine
  - packet number, ACK/NACK ranges, anti-replay, crypto, pacing, congestion

UDP Socket Layer
```

The business layer owns message meaning. QCP owns how the bytes are transported.

---

## 3. Core Modules

| Module | Responsibility | Notes |
|--------|----------------|-------|
| `session` | SessionID, ConnectionID, token issue/verify, resume state | No account or player concepts. |
| `packet` | Header encode/decode, packet number, frame parsing, checksum/auth tag | Stable binary ABI. |
| `stream` | REALTIME / CRITICAL / BATCH queues and delivery state | Independent queues prevent cross-stream blocking. |
| `scheduler` | Deadline-aware send selection and priority | CRITICAL can preempt BATCH. |
| `recovery` | ACK/NACK, retransmit queue, optional coding, loss recovery | Policy-driven, not FEC-first. |
| `path` | PathID, address validation, migration, RTT/loss/jitter tracking | Default max paths: 4. |
| `cc` | Pacing, congestion window, bandwidth estimate, backoff | Per-path state. |
| `crypto` | Key epoch, packet protection, token binding, anti-replay | Must be enabled in production. |
| `metrics` | P50/P95/P99, loss, retransmit, path switch, resume latency | Low-overhead sampling. |
| `buffer` | Packet pools, ring buffers, slab allocator | No allocation in hot path. |

---

## 4. Wire Format

The production header should be compact but explicit enough for migration, anti-replay, and future extension.

| Field | Size | Purpose |
|-------|------|---------|
| `flags` | 1 byte | packet type, ack presence, extension presence |
| `stream` | 1 byte | REALTIME / CRITICAL / BATCH / CONTROL |
| `conn_id` | varint or 32-bit | connection routing without 5-tuple dependency |
| `packet_no` | varint or 32-bit | anti-replay and ACK ranges |
| `stream_seq` | 16-bit or 32-bit | stream-local ordering / latest-wins |
| `path_id` | 1 byte | multi-path and migration |
| `payload_len` | varint | frame payload length |
| `frames` | variable | DATA / ACK / NACK / PATH / TOKEN / EXT |
| `auth_tag` | 8-16 bytes | packet authentication |

Recommended packet types:

| Type | Use |
|------|-----|
| `INITIAL` | first handshake, token request, feature negotiation |
| `HANDSHAKE` | key confirmation and transport params |
| `SHORT` | normal encrypted data packet |
| `CONTROL` | path challenge, path response, close, drain |
| `RETRY` | stateless retry / anti-amplification |

Recommended frame types:

| Frame | Use |
|-------|-----|
| `DATA` | payload with stream, seq, deadline class |
| `ACK_RANGE` | compressed ACK ranges |
| `NACK_RANGE` | explicit gaps for Fast NACK |
| `PATH_CHALLENGE` | validate new address/path |
| `PATH_RESPONSE` | confirm path ownership |
| `TOKEN` | resume / retry token transport |
| `MAX_DATA` | flow control credit |
| `PING` | keepalive / RTT probe |
| `EXT` | skippable extension frame |

---

## 5. Stream Semantics For All Game Protocols

QCP should expose generic stream semantics that can cover all game business protocols without knowing the business schema.

| Business protocol class | QCP stream | Reliability | Ordering | Deadline | Example |
|-------------------------|------------|-------------|----------|----------|---------|
| Movement / transform / aim | REALTIME | latest-wins | stream-local seq, drop old | 1 tick | position, velocity, rotation |
| AOI snapshot / entity state | REALTIME | latest-wins | newest snapshot wins | 1-2 ticks | nearby entity state |
| Combat command | CRITICAL | bounded reliable | ordered per command stream | 5-20 ms | fire, skill cast, interrupt |
| Hit / damage event | CRITICAL | bounded reliable | ordered / deduped | 5-20 ms | hit confirm, damage apply event |
| Inventory / economy / trade | BATCH | reliable | ordered | no tight deadline | item update, shop result |
| Chat / social | BATCH | reliable | ordered | relaxed | chat, friend event |
| Match / room control | CRITICAL or BATCH | reliable | ordered | depends on UX | ready, start, leave |
| Telemetry / debug | BATCH or REALTIME | configurable | optional | relaxed | perf trace, client stats |

Business logic chooses the row. QCP applies the transport behavior.

---

## 6. State Machines

### 6.1 Session

```
NEW
  -> HANDSHAKING
  -> ESTABLISHED
  -> MIGRATING
  -> RESUMING
  -> DRAINING
  -> CLOSED
```

Rules:

| Transition | Protocol requirement |
|------------|----------------------|
| `HANDSHAKING -> ESTABLISHED` | keys confirmed, transport params negotiated |
| `ESTABLISHED -> MIGRATING` | new address/path observed or probed |
| `MIGRATING -> ESTABLISHED` | path validated by challenge/response |
| `ESTABLISHED -> RESUMING` | resume token accepted after disconnect |
| `RESUMING -> ESTABLISHED` | key epoch and packet number window restored |
| `* -> DRAINING` | close sent, retransmit only required close/control |

### 6.2 Path

```
UNKNOWN -> PROBING -> VALIDATED -> ACTIVE
                         |          |
                         v          v
                      STANDBY <-> DEGRADED -> CLOSED
```

Rules:

| Path state | Behavior |
|------------|----------|
| `PROBING` | send PATH_CHALLENGE and low-rate RTT probes |
| `VALIDATED` | eligible for CRITICAL fallback |
| `ACTIVE` | primary path for most traffic |
| `STANDBY` | keep warm with low-rate probes |
| `DEGRADED` | avoid BATCH, allow emergency CRITICAL only |

---

## 7. Recovery Policy

Recovery is selected per stream and deadline.

| Stream | Default policy | Loss response |
|--------|----------------|---------------|
| REALTIME | send newest only | do not recover old state; replace with newer seq |
| CRITICAL | ACK/NACK + bounded retransmit | Fast NACK immediately; stop after deadline |
| BATCH | ARQ + ACK ranges | recover until delivered or connection closes |

Policy order:

1. Use best validated path.
2. If CRITICAL and multiple paths are healthy, use short Race window.
3. If gap detected, send NACK immediately.
4. If burst loss and deadline allows, enable temporary coding.
5. If deadline expired, stop recovery and report timeout.

Hard rule: BATCH recovery must not block REALTIME delivery.

---

## 8. Congestion And Pacing

Production QCP should use per-path congestion state.

| Mechanism | Requirement |
|-----------|-------------|
| Pacing | send packets on pacing timer, not burst all queues |
| cwnd | maintain per-path congestion window |
| RTT estimate | srtt, rttvar, min_rtt per path |
| loss reaction | reduce sending rate on loss; do not globally stall all streams |
| deadline scheduling | CRITICAL with near deadline can preempt BATCH |
| anti-bufferbloat | drop obsolete REALTIME before queue grows |

Queue policy:

| Queue | Bound |
|-------|-------|
| REALTIME | latest value per key / channel, not unbounded FIFO |
| CRITICAL | bounded ring, deadline-ordered |
| BATCH | backpressure-capable FIFO |

---

## 9. Security Baseline

Production mode must not ship unauthenticated packets.

| Feature | Requirement |
|---------|-------------|
| Packet protection | AEAD or equivalent authenticated encryption |
| Token binding | resume token bound to ConnectionID, key epoch, expiry, and server secret |
| Anti-replay | sliding packet number window per session and path |
| Address validation | path challenge before trusting migrated path |
| Amplification control | limit bytes sent before validation |
| Key rotation | support key epoch and rekey |
| Downgrade protection | feature negotiation must be authenticated |

Business auth remains outside QCP. QCP only authenticates transport continuity and packet integrity.

---

## 10. Extension Model

Extensions must be negotiated and skippable.

| Extension area | Examples |
|----------------|----------|
| recovery | coding schemes, redundancy hints |
| path | network slice hint, carrier path class |
| metrics | trace id, sampled diagnostics |
| scheduling | priority classes beyond default |
| platform | console/mobile network hints |

Rules:

1. Unknown `EXT` frames are skipped if marked optional.
2. Required extensions must be negotiated during handshake.
3. Extension parsing cannot allocate in the receive hot path.
4. Extension data cannot change core security checks.

---

## 11. Performance Engineering

Implementation requirements:

| Area | Requirement |
|------|-------------|
| packet buffers | pooled fixed-size buffers |
| queues | ring buffers or intrusive lists |
| timers | timer wheel or batched timers, not one timer per packet |
| ACK state | range compression, no per-packet map on hot path |
| retransmit | indexed ring by packet number |
| metrics | sampling counters, no formatting in hot path |
| locks | avoid global locks; per-session or sharded state |
| memory | bounded per-session memory budget |

Default production limits:

| Setting | Default |
|---------|---------|
| max paths | 4 |
| max packet size | path MTU bounded, default 1200-1400 bytes |
| REALTIME queue | latest only per key |
| CRITICAL queue | 256-1024 messages per session |
| BATCH queue | configurable backpressure |
| anti-replay window | 1024-8192 packets |
| idle timeout | configurable, default 15-30 s |

---

## 12. API Shape

The API should keep protocol concerns explicit without leaking business concepts.

```go
type Conn interface {
    Send(payload []byte, opts SendOptions) error
    Recv(buf []byte) (Message, error)
    Close(code CloseCode) error

    Paths() []PathStats
    Metrics() ConnMetrics
    ExportResumeToken() ([]byte, error)
}

type SendOptions struct {
    Stream   StreamType
    Deadline time.Duration
    Priority uint8
    Key      uint32 // optional latest-wins key for REALTIME coalescing
    Flags    SendFlags
}

type Message struct {
    Payload []byte
    Stream  StreamType
    Seq     uint64
    PathID  uint8
    Flags   MessageFlags
}

type Events interface {
    OnPathChange(PathStats)
    OnResume(ResumeInfo)
    OnTimeout(TimeoutInfo)
    OnDegraded(DegradeInfo)
}
```

Business protocols sit above `Payload`.

---

## 13. Production Test Matrix

Every implementation should pass the following protocol-level tests.

| Test | Acceptance |
|------|------------|
| 0% loss LAN | QCP P50/P99 better than TCP/KCP baseline |
| 1-5% random loss | CRITICAL delivered before deadline target |
| 10-20% burst loss | REALTIME remains latest-wins; BATCH eventually delivers |
| NAT port change | connection migrates without new business session |
| WiFi -> cellular switch | CRITICAL continues on validated fallback path |
| cellular -> WiFi switch | primary path changes after RTT/loss improves |
| resume after idle gap | resume token restores session without full handshake |
| replayed packet | dropped and counted |
| unknown optional extension | skipped without connection failure |
| unknown required extension | handshake fails cleanly |
| BATCH flood | REALTIME and CRITICAL P99 remain bounded |

Required benchmark outputs:

1. `性能对比表格.md`: QCP / TCP / KCP only.
2. `migration-report.md`: path switch latency and packet loss during migration.
3. `resume-report.md`: reconnect / resume latency and success rate.
4. `loss-report.md`: random and burst loss delivery behavior per stream.

---

## 14. Implementation Order

Recommended order for a production implementation:

| Phase | Deliverable |
|-------|-------------|
| 1 | stable packet format, stream scheduler, ACK/NACK ranges |
| 2 | CRITICAL bounded retransmit and BATCH ARQ isolation |
| 3 | REALTIME coalescing by key and latest-wins receive path |
| 4 | session token, anti-replay, key epoch |
| 5 | path validation and connection migration |
| 6 | multi-path probing, standby path, CRITICAL fallback/race |
| 7 | per-path congestion and pacing |
| 8 | optional coding policy for burst loss |
| 9 | metrics, reports, production soak tests |

Do not start with coding/FEC. Start with correct stream isolation, deadline-aware recovery, session continuity, and path migration. Those are the production foundation for game battle traffic.
