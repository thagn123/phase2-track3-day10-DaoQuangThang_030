# Day 10 Reliability Report

## 1. Architecture summary

The architecture consists of a reliability layer sitting in front of the provider API. It uses an in-memory/Redis cache to avoid redundant downstream calls. If a cache miss occurs, the gateway routes the request through a Circuit Breaker pointing to the primary provider. If the primary circuit is open or it fails, the fallback provider is used. If all providers fail, a static fallback message is returned.

```text
User Request
    |
    v
[Gateway] ---> [Cache check] ---> HIT? return cached
    |                                 |
    v                                 v MISS
[Circuit Breaker: Primary] -------> Provider A
    |  (OPEN? skip)
    v
[Circuit Breaker: Backup] --------> Provider B
    |  (OPEN? skip)
    v
[Static fallback message]
```

## 2. Configuration

| Setting | Value | Reason |
|---|---:|---|
| failure_threshold | 3 | Low enough to detect failures fast but high enough to ignore minor jitters. |
| reset_timeout_seconds | 2 | Gives the provider enough time to recover without waiting excessively. |
| success_threshold | 1 | Ensures one successful probe in HALF_OPEN resets the circuit. |
| cache TTL | 300 | 5 minutes balances freshness with cache hit rate for typical interactions. |
| similarity_threshold | 0.92 | Optimized to prevent false hits while capturing high overlap inputs. |
| load_test requests | 100 | Sufficient to test circuit breaker oscillation and cache filling. |

## 3. SLO definitions

| SLI | SLO target | Actual value | Met? |
|---|---|---:|---|
| Availability | >= 99% | 1.000 | Yes |
| Latency P95 | < 2500 ms | 313.0 | Yes |
| Fallback success rate | >= 95% | 1.0 | Yes |
| Cache hit rate | >= 10% | 0.7875 | Yes |
| Recovery time | < 5000 ms | N/A | (Testing ended before recovery elapsed) |

## 4. Metrics

| Metric | Value |
|---|---:|
| availability | 1.0 |
| error_rate | 0.0 |
| latency_p50_ms | 0.0 |
| latency_p95_ms | 313.0 |
| latency_p99_ms | 515.01 |
| fallback_success_rate | 1.0 |
| cache_hit_rate | 0.7875 |
| estimated_cost_saved | 0.315 |
| circuit_open_count | 4 |
| recovery_time_ms | null |

## 5. Cache comparison

| Metric | Without cache | With cache | Delta |
|---|---:|---:|---|
| latency_p50_ms | 266.0 | 0.0 | -100% |
| latency_p95_ms | 516.0 | 313.0 | -39% |
| estimated_cost | 0.183796 | 0.037884 | -79% |
| cache_hit_rate | 0.0 | 0.7875 | +0.7875 |

## 6. Redis shared cache

- Why in-memory cache is insufficient for multi-instance deployments: In-memory cache is restricted to the specific gateway process. Multi-instance load balancing results in low cache hit rates since each instance maintains its own cache state.
- How `SharedRedisCache` solves this: Externalizing the cache to Redis ensures all gateway instances lookup from a centralized store.

### Evidence of shared state

A `SharedRedisCache` instance connects to the dockerized Redis database and sets values. Any other Gateway instance requesting the same `key` from the Redis Database immediately retrieves the hit.

### Redis CLI output

```bash
# docker compose exec redis redis-cli KEYS "rl:cache:*"
1) "rl:cache:8baa2cfa11fa"
2) "rl:cache:9e413fd814eb"
3) "rl:cache:095946136fea"
4) "rl:cache:b2a52f7dc795"
```

### In-memory vs Redis latency comparison (optional)

| Metric | In-memory cache | Redis cache | Notes |
|---|---:|---:|---|
| latency_p50_ms | 0.0 | ~2.0 | Redis adds network latency. |
| latency_p95_ms | 313.0 | 315.0 | Overall performance relies heavily on provider. |

## 7. Chaos scenarios

| Scenario | Expected behavior | Observed behavior | Pass/Fail |
|---|---|---|---|
| primary_timeout_100 | All traffic fallback to backup, circuit opens | Primary failed, traffic fell back to backup, circuit opened. | pass |
| primary_flaky_50 | Circuit oscillates, mix of primary and fallback | Circuit transitioned between open and close, mixing routes. | pass |
| all_healthy | All requests via primary, no circuit opens | All non-cached requests routed to primary without circuit errors. | pass |
| cache_stale_candidate | Test false hits on cache queries | Successfully dodged false hits using the year validation logic. | pass |

## 8. Failure analysis

- What could still go wrong? 
Circuit breaker state is per-instance. If an instance restarts or traffic routes to a new instance, the new gateway won't know the primary provider is failing until it tries and fails multiple times itself.
- What would you change?
Store the Circuit Breaker transition and probe counts in the shared Redis Cache with an appropriate TTL so all instances react instantaneously to a downed provider.

## 9. Next steps

1. Persist circuit breaker states in Redis.
2. Add per-user rate-limiting to restrict abuse and high costs.
3. Apply cost-aware budget routing rules to automatically transition to cheaper fallback providers once the threshold reaches >80%.
