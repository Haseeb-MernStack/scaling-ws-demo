# Load Test Report #002 — Realistic Traffic, Upgraded Caddy

**Date:** 2026-03-23
**Test ID:** 002-realistic-4cpu-8gb-caddy

## Infrastructure

| VM | Role | Specs | Notes |
|----|------|-------|-------|
| VM 1 | Caddy (reverse proxy) | 4 CPU, 8GB RAM | Upgraded from 2 CPU / 4GB |
| VM 2 | NATS | 2GB RAM | No issues |
| VM 3 | ws-server-1 | 2 CPU, 2GB RAM | CPU bottleneck (106%) |
| VM 4 | ws-server-2 | 2 CPU, 2GB RAM | Under-utilized (3% CPU) |
| VM 5 | ws-server-3 | 2 CPU, 2GB RAM | Moderate (71% CPU) |
| VM 6 | MySQL 8 (self-hosted) | - | No issues |
| VM 7 | Load test runner | 8GB RAM | AWS |

## Test Configuration

```
TOTAL_CONNECTIONS=10000
RAMP_RATE=100/sec
MESSAGE_INTERVAL_MS=2000
TEST_DURATION_SEC=120
NUM_ROOMS=10
THUNDERING_HERD=true (80%)
ACTIVE_SENDERS=10% (realistic)
PASSIVE_READERS=90%
ROOMS=23 (weighted distribution)
BURST_CHANCE=30%
WS_URL=wss://chat.codersgyan.com/ws
```

**Test type:** Realistic — 10% active senders, 90% passive readers, burst messaging, weighted room distribution

## Results

### Connection Phase
| Metric | Value | Status | vs Report #001 |
|--------|-------|--------|----------------|
| Target connections | 10,000 | | Same |
| Opened | 6,557 | | Worse (was 9,291) |
| Failed | 6,886 | ISSUE | Worse (was 1,418) |
| Success rate | 65.6% | ISSUE | Worse (was 92.9%) |
| Connect latency (avg) | 6,709ms | | Worse (was 5,451ms) |
| Connect latency (P50) | 4,213ms | | Similar (was 3,827ms) |
| Connect latency (P99) | 24,318ms | ISSUE | Similar (was 24,446ms) |

### Messaging Phase
| Metric | Value | Notes |
|--------|-------|-------|
| Messages sent | 854,527 | More than #001 (650,881) despite fewer connections |
| Messages received | 410,053,138 | Massive fan-out |
| Cross-server deliveries | 272,941,883 | NATS working correctly |
| Delivery ratio | 47,986.0% | Expected with large rooms |

### Thundering Herd Phase
| Metric | Value | Status | vs Report #001 |
|--------|-------|--------|----------------|
| Clients disconnected | 5,245 (80%) | | |
| Reconnected | 5,245 | | |
| Reconnect failed | 0 | PASS | Improved! (was 752) |
| Reconnect success rate | 100.0% | PASS | Improved! (was 94.9%) |
| Reconnect latency (avg) | 4,646ms | | Improved (was 7,217ms) |
| Reconnect latency (P50) | 4,347ms | | Improved (was 6,284ms) |
| Reconnect latency (P99) | 10,799ms | | Improved (was 18,446ms) |

### Message Replay Phase
| Metric | Value | Status |
|--------|-------|--------|
| Clients that got replay | 5,245 | PASS (100%) |
| Messages replayed | 5,133,780 | PASS |

### Infrastructure
| Metric | Value |
|--------|-------|
| Unique servers seen | ws-server-1, ws-server-2, ws-server-3 |
| Server reconnect signals | 0 |
| Load test memory (RSS) | 1,153 MB |

### Server Load During Test
| Server | CPU | RAM | Notes |
|--------|-----|-----|-------|
| Caddy | 255% / 400% | 758 MB / 8GB | CPU is the limit |
| ws-server-1 | 106% / 200% | 219 MB / 2GB | Maxed out |
| ws-server-2 | 3% / 200% | 163 MB / 2GB | Under-utilized |
| ws-server-3 | 71% / 200% | 177 MB / 2GB | Moderate |

## Verdict

**ISSUES FOUND:**
- Low connection success rate: 65.6%
- High P99 connect latency: 24,318ms
- Too many connection failures: 6,886

**IMPROVEMENTS vs #001:**
- Thundering herd: 100% reconnect success (was 94.9%)
- Reconnect P99 latency halved: 10.8s (was 18.4s)
- Message replay: 100% of clients got replay

## Root Cause Analysis

### 1. Connection failures worse than #001
Despite upgrading Caddy (4 CPU / 8GB), connection success dropped from 92.9% to 65.6%. The reason: **WS servers are now the bottleneck, not Caddy.** Server 1 was at 106% CPU (maxed), server 2 at only 3%. The `least_conn` load balancing strategy caused severely uneven distribution.

### 2. Uneven load distribution (least_conn problem)
`least_conn` routes new connections to the server with fewest active connections. For long-lived WebSocket connections, this creates a feedback loop:
- Server gets slightly slow → Caddy routes new connections elsewhere → Server's existing connections still generate fan-out work → Imbalance compounds

### 3. WS servers CPU-bound at 2 cores
With 2 CPUs per server, the combination of WebSocket handling + JSON serialization + NATS pub/sub + MySQL writes saturates the CPU well before memory is an issue. Memory usage stayed under 220 MB even at peak.

## Observations

1. **CPU-bound, not memory-bound** — All servers used <220 MB RAM but hit CPU limits. WebSocket message fan-out (JSON.stringify × room_size × messages/sec) is the dominant CPU cost.

2. **Thundering herd improved significantly** — 100% reconnect success (vs 94.9% in #001). The upgraded Caddy handled the reconnection storm much better. The lower connection count at disconnect time (5,245 vs 7,432) also helped.

3. **Message replay is solid** — Every reconnected client received missed messages. The MySQL-based replay mechanism works reliably.

4. **The system plateaus at ~5,500 active connections** — After hitting 5,200 connections, failures spike. This is the effective capacity ceiling with current WS server specs.

## Action Items

1. Switch from `least_conn` to `round_robin` in Caddy and retest — should give more even distribution
2. Consider upgrading WS servers from 2 CPU to 4 CPU
3. Rerun test after changes for comparison
