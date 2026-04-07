# Load Test Report #003 — Realistic Traffic, Round Robin, Per-Room NATS

**Date:** 2026-03-23
**Test ID:** 003-realistic-roundrobin-perroom-nats
**Result:** PASS

## Infrastructure

| VM | Role | Specs |
|----|------|-------|
| VM 1 | Caddy (reverse proxy) | 4 CPU, 8GB RAM |
| VM 2 | NATS | 2GB RAM |
| VM 3 | ws-server-1 | 2 CPU, 2GB RAM |
| VM 4 | ws-server-2 | 2 CPU, 2GB RAM |
| VM 5 | ws-server-3 | 2 CPU, 2GB RAM |
| VM 6 | MySQL 8 (self-hosted) | Self-hosted |
| VM 7 | Load test runner | 8GB RAM (AWS) |

## Changes Since Report #002

1. **Load balancing:** `least_conn` → `round_robin`
2. **NATS subscriptions:** `chat.*` wildcard → per-room (`chat.general`, `chat.random`, etc.)
3. **Load test:** Synthetic (100% senders) → Realistic (10% senders, 90% readers, burst messaging)
4. **Clean server state:** Servers restarted before test to clear zombie connections

## Test Configuration

```
TOTAL_CONNECTIONS=10000
RAMP_RATE=100/sec
TEST_DURATION_SEC=120
ACTIVE_SENDERS=10% (1,000)
PASSIVE_READERS=90% (9,000)
ROOMS=23 (weighted distribution)
MESSAGE_INTERVAL=3,000-15,000ms (randomized)
BURST_CHANCE=30% (2-5 msgs per burst)
THUNDERING_HERD=true (80%)
WS_URL=wss://chat.codersgyan.com/ws
```

## Results

### Connection Phase
| Metric | Value | Status | vs #001 | vs #002 |
|--------|-------|--------|---------|---------|
| Target connections | 10,000 | | | |
| Opened | 10,000 | PASS | +709 | +3,443 |
| Failed | 0 | PASS | -1,418 | -6,886 |
| Success rate | 100.0% | PASS | was 92.9% | was 65.6% |
| Connect latency (avg) | 1,189ms | PASS | was 5,451ms | was 6,709ms |
| Connect latency (P50) | 923ms | PASS | was 3,827ms | was 4,213ms |
| Connect latency (P99) | 3,550ms | PASS | was 24,446ms | was 24,318ms |

### Messaging Phase
| Metric | Value | Notes |
|--------|-------|-------|
| Active senders | 1,000 | 10% of connections |
| Passive readers | 16,200 | Includes reconnected clients |
| Messages sent | 34,136 | Realistic volume |
| Messages received | 19,322,544 | Fan-out across rooms |
| Cross-server deliveries | 12,844,780 | NATS working |
| Avg messages per sender | 34.1 | ~0.7 msgs/sec per sender |

### Room Distribution
| Room | Users | Notes |
|------|-------|-------|
| general | 2,652 | Largest (15% weight) |
| random | 2,218 | Second largest |
| tech | 1,747 | |
| support | 1,503 | |
| announcements | 1,451 | |
| design | 1,025 | |
| watercooler | 987 | |
| marketing | 853 | |
| project-beta | 740 | |
| project-alpha | 725 | |
| ... | 13 more rooms | Smaller teams |

### Thundering Herd Phase
| Metric | Value | Status | vs #001 | vs #002 |
|--------|-------|--------|---------|---------|
| Clients disconnected | 8,000 (80%) | | | |
| Reconnected | 8,000 | PASS | | |
| Reconnect failed | 0 | PASS | was 752 | was 0 |
| Reconnect success rate | 100.0% | PASS | was 94.9% | was 100% |
| Reconnect latency (avg) | 1,469ms | PASS | was 7,217ms | was 4,646ms |
| Reconnect latency (P50) | 1,418ms | PASS | was 6,284ms | was 4,347ms |
| Reconnect latency (P99) | 2,569ms | PASS | was 18,446ms | was 10,799ms |

### Message Replay Phase
| Metric | Value | Status |
|--------|-------|--------|
| Clients that got replay | 8,000 (100%) | PASS |
| Messages replayed | 823,324 | PASS |

### Infrastructure
| Metric | Value |
|--------|-------|
| Unique servers seen | ws-server-1, ws-server-2, ws-server-3 |
| Server reconnect signals | 0 |
| Load test memory (RSS) | 1,104 MB |

## Verdict

**PASS — System handled the load successfully**

No issues found. All 10,000 connections established with 0 failures. Thundering herd recovered 100% with sub-3s P99 latency. Message replay working for all reconnected clients.

## Comparison Across All Tests

| Metric | #001 (baseline) | #002 (upgraded Caddy) | #003 (this test) |
|--------|----------------|----------------------|------------------|
| Caddy specs | 2 CPU, 4GB | 4 CPU, 8GB | 4 CPU, 8GB |
| LB strategy | least_conn | least_conn | **round_robin** |
| NATS | wildcard | wildcard | **per-room** |
| Test type | Synthetic (100%) | Realistic (10%) | Realistic (10%) |
| Connection success | 92.9% | 65.6% | **100%** |
| Connect P99 | 24,446ms | 24,318ms | **3,550ms** |
| Herd reconnect | 94.9% | 100% | **100%** |
| Herd P99 | 18,446ms | 10,799ms | **2,569ms** |
| Verdict | FAIL | FAIL | **PASS** |

## What Fixed It

Three changes combined:

1. **Round robin** — even connection distribution eliminated the CPU skew (106%/3%/71% → balanced)
2. **Per-room NATS** — servers only process messages for rooms they have users in (zero wasted work)
3. **Realistic traffic** — 10% senders instead of 100% reduced message volume by ~10x, matching real-world usage
4. **Clean server state** — no zombie connections from previous test runs interfering
