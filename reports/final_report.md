# Day 10 Reliability Final Report

## 1. Architecture summary

Multi-layer reliability: ResponseCache + SharedRedisCache -> CircuitBreaker (3-state FSM per provider) -> Provider fallback chain (primary -> backup) -> Static fallback message.

## 2. Configuration

| Setting | Value | Reason |
|---|---:|---|
| failure_threshold | 3 | 2 transient errors before opening circuit |
| reset_timeout_seconds | 2 | Fast recovery in 2 seconds |
| success_threshold | 1 | Single probe success restores CLOSED |
| cache TTL | 300 | 5-min freshness for LLM responses |
| similarity_threshold | 0.92 | Only near-identical semantics match |
| load_test requests | 100 | Statistically meaningful sample |

## 3. SLO definitions

| SLI | SLO target | Actual value | Met? |
|---|---|---|---|
| Availability | >= 99% | 0.3475 | X |
| Latency P95 | < 2500 ms | 63.28 ms | V |
| Fallback success rate | >= 95% | 0.9894 | V |
| Cache hit rate | >= 10% | 0.65 | V |
| Recovery time | < 5000 ms | None | X |

## 4. Metrics

| Metric | Value |
|---|---|
| total_requests | 400 |
| availability | 0.3475 |
| error_rate | 0.0025 |
| latency_p50_ms | 37.17 |
| latency_p95_ms | 63.28 |
| latency_p99_ms | 67.5 |
| fallback_success_rate | 0.9894 |
| cache_hit_rate | 0.65 |
| estimated_cost_saved | 0.26 |
| circuit_open_count | 3 |
| recovery_time_ms | None |

## 5. Cache comparison

| Metric | Without cache | With cache | Delta |
|---|---|---|---|
| latency_p50_ms | ~180 ms | 37.17 ms | -79% |
| latency_p95_ms | ~260 ms | 63.28 ms | -75% |
| estimated_cost | ~$0.08 | $0.058 | -27% |
| cache_hit_rate | 0 | 0.65 | +65% |
| cost_saved | $0 | $0.26 | saved |

## 6. Redis shared cache

- **Why in-memory insufficient**: Each gateway instance has its own memory. In multi-instance K8s, cache misses repeat across instances.
- **How SharedRedisCache solves**: Centralized Redis store. MD5 hash for exact match + SCAN for similarity across all cached entries.

### Evidence of shared state (6/6 PASSED)

```
test_redis_connection              PASSED
test_set_and_exact_get             PASSED
test_ttl_expiry                    PASSED
test_shared_state_across_instances PASSED
test_privacy_query_not_cached      PASSED
test_false_hit_different_years     PASSED
```

## 7. Chaos scenarios

| Scenario | Expected | Observed | Pass/Fail |
|---|---|---|---|
| primary_timeout_100 | All fallback | Fallback rate 0.9894, circuit opened | pass |
| primary_flaky_50 | Oscillate | Circuit opened, both providers used | pass |
| all_healthy | All via primary | Error 0.0025, p95 63.28ms | pass |
| primary_slow_80 | Recovery time | Circuit opened, no recovery in 2s | fail |

## 8. Failure analysis

**What could still go wrong?**
- Recovery time not measured: circuit opens late, no close cycle in 2s window.
- Backup overload under heavy redirected traffic from primary failures.
- Cache poisoning: similarity-based cache returns wrong answers for semantically different queries.

**What would you change?**
- Redis-backed circuit state for consistent multi-instance failover.
- Per-user rate limiting to prevent system overload.
- Quality SLO monitoring: track semantic similarity of answers to expected.

## 9. Next steps

1. **Redis circuit state**: Share across all gateway instances for consistent K8s deployments.
2. **Adaptive similarity threshold**: Tune per query type (technical/FAQ/policy).
3. **Cost-aware routing**: Track cumulative cost, skip expensive providers on budget cap.

## Test Results: 35 passed + 7 XPASS = 42 total

| Suite | Tests | Result |
|---|---|---|
| test_circuit_breaker.py | 12 | PASSED |
| test_cache.py | 9 | PASSED |
| test_gateway_contract.py | 4 | PASSED |
| test_metrics.py | 2 | PASSED |
| test_config.py | 2 | PASSED |
| test_redis_cache.py | 6 | PASSED |
| test_todo_requirements.py | 7 | XPASS |
| **Total** | **42** | **35 passed + 7 XPASS** |
