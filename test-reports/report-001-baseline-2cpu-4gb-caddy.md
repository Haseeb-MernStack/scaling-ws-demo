# Load Test Report #001 — Baseline

**Date:** 2026-03-23
**Test ID:** 001-baseline

## Infrastructure

| VM | Role | Specs | Notes |
|----|------|-------|-------|
| VM 1 | Caddy (reverse proxy) | 2 CPU, 4GB RAM | Bottleneck identified |
| VM 2 | NATS | 2GB RAM | No issues |
| VM 3 | ws-server-1 | 4GB RAM | No issues |
| VM 4 | ws-server-2 | 4GB RAM | No issues |
| VM 5 | ws-server-3 | 4GB RAM | No issues |
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
WS_URL=wss://chat.codersgyan.com/ws
```

**Test type:** Synthetic — all users send messages every 2s (not realistic)

## Results

### Connection Phase
| Metric | Value | Status |
|--------|-------|--------|
| Target connections | 10,000 | |
| Opened | 9,291 | |
| Failed | 1,418 | ISSUE |
| Success rate | 92.9% | ISSUE (<95%) |
| Connect latency (avg) | 5,451ms | |
| Connect latency (P50) | 3,827ms | |
| Connect latency (P99) | 24,446ms | ISSUE (>5s) |

### Messaging Phase
| Metric | Value | Notes |
|--------|-------|-------|
| Messages sent | 650,881 | |
| Messages received | 260,884,850 | High due to room fan-out |
| Cross-server deliveries | 173,501,225 | NATS working correctly |
| Delivery ratio | 40,081.8% | Expected with ~1000 users/room |

### Thundering Herd Phase
| Metric | Value | Status |
|--------|-------|--------|
| Clients disconnected | 7,432 (80%) | |
| Reconnected | 7,056 | |
| Reconnect failed | 752 | ISSUE |
| Reconnect success rate | 94.9% | |
| Reconnect latency (avg) | 7,217ms | |
| Reconnect latency (P50) | 6,284ms | |
| Reconnect latency (P99) | 18,446ms | ISSUE |

### Message Replay Phase
| Metric | Value | Status |
|--------|-------|--------|
| Clients that got replay | 7,056 | PASS |
| Messages replayed | 7,078,329 | PASS |

### Infrastructure
| Metric | Value |
|--------|-------|
| Unique servers seen | ws-server-1, ws-server-2, ws-server-3 |
| Server reconnect signals | 0 |
| Load test memory (RSS) | 1,478 MB |

## Verdict

**ISSUES FOUND:**
- Low connection success rate: 92.9%
- High P99 connect latency: 24,446ms
- Too many connection failures: 1,418
- Thundering herd: 752 reconnects failed

## Root Cause Analysis

All issues traced to **Caddy VM being under-resourced** (2 CPU, 4GB RAM). Every WebSocket connection passes through Caddy twice (client→Caddy→server), requiring TLS termination on each. At ~3,500 connections, Caddy was hitting 141% CPU (maxing both cores).

App servers (ws-server-1/2/3), NATS, and MySQL showed no signs of stress.

## Action Items

1. Upgrade Caddy VM to 4 CPU, 4-8GB RAM
2. Rerun test with upgraded Caddy
3. Switch to realistic traffic patterns (variable room sizes, 5-10% active senders)
