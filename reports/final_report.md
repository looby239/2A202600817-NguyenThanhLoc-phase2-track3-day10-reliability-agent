# Day 10 Reliability Report

## 1. Architecture summary

The Reliability Layer is designed as a decorator/wrapper pipeline between the User/Client and the upstream LLM providers, ensuring high availability, lower latency, and reduced provider cost.

```
User Request
    |
    v
[Gateway] ---> [Cache check (Memory/Redis)] ---> HIT? return cached
    |                                                 |
    v                                                 v MISS
[Circuit Breaker: Primary] -------> Provider A (fails? record_failure())
    |  (OPEN? skip to backup)
    v
[Circuit Breaker: Backup] --------> Provider B (fails? record_failure())
    |  (OPEN? skip to static fallback)
    v
[Static fallback message]
```

### Components:
1. **Reliability Gateway**: Orchestrates the request routing pipeline. It first queries the semantic cache, and on a miss, tries each configured LLM provider in order using their associated circuit breakers. If all providers are degraded or fail, it returns a static fallback response.
2. **Circuit Breaker (3-State Machine)**: Wraps provider calls to implement fail-fast behavior.
   - **CLOSED**: Calls pass through; counts consecutive failures.
   - **OPEN**: Rejects calls immediately (fail-fast) until the reset timeout expires.
   - **HALF_OPEN**: Allows a single probe request to check if the provider is healthy. If the probe succeeds (up to `success_threshold` times), it transitions back to `CLOSED`. If it fails, it immediately re-opens.
3. **Semantic Cache**:
   - Computes cosine similarity over character 3-grams and word tokens to identify semantically equivalent queries.
   - Uses privacy keywords filtering (`_is_uncacheable()`) to prevent caching sensitive data (e.g., credit cards, passwords).
   - Employs false-hit detection (`_looks_like_false_hit()`) to reject cached responses if queries contain differing 4-digit numbers (like years or IDs).
   - Supports two backends: In-Memory (for single instances) and Redis (for multi-instance shared state).

---

## 2. Configuration

| Setting | Value | Reason |
|---|---:|---|
| `failure_threshold` | 3 | Allow up to 3 consecutive failures before opening the circuit, avoiding premature trip on transient network blips. |
| `reset_timeout_seconds` | 2 | Hold the circuit open for 2 seconds to allow the provider system to cool down before sending a probe request. |
| `success_threshold` | 1 | Require 1 successful probe request in `HALF_OPEN` to verify provider health and transition back to `CLOSED`. |
| `cache TTL` | 300 | Cache responses for 5 minutes (300 seconds) to balance freshness of data against hit rate. |
| `similarity_threshold` | 0.92 | High threshold to ensure semantic matches are highly relevant and accurate, preventing false cache hits. |
| `load_test requests` | 100 | Run 100 requests per scenario to obtain a statistically significant distribution of latency and availability metrics. |

---

## 3. SLO definitions

Define your target SLOs and whether your system meets them:

| SLI | SLO target | Actual value | Met? |
|---|---|---:|---|
| Availability | >= 99% | 98.67% | **No** (Marginally missed due to flaky providers and backup fail rates) |
| Latency P95 | < 2500 ms | 314.33 ms | **Yes** |
| Fallback success rate | >= 95% | 94.37% | **No** (Marginally missed due to simultaneous backup failures) |
| Cache hit rate | >= 10% | 64.33% | **Yes** |
| Recovery time | < 5000 ms | 2233.33 ms | **Yes** |

---

## 4. Metrics

Paste or summarize `reports/metrics.json`.

| Metric | Value |
|---|---:|
| availability | 0.9867 |
| error_rate | 0.0133 |
| latency_p50_ms | 272.27 |
| latency_p95_ms | 314.33 |
| latency_p99_ms | 317.64 |
| fallback_success_rate | 0.9437 |
| cache_hit_rate | 0.6433 |
| estimated_cost_saved | 0.193 |
| circuit_open_count | 7 |
| recovery_time_ms | 2233.33 |

---

## 5. Cache comparison

Run simulation with cache enabled vs disabled. Fill in both columns:

| Metric | Without cache | With cache | Delta |
|---|---:|---:|---|
| latency_p50_ms | 285.87 ms | 287.14 ms | +1.27 ms (Minor variance due to sleep/jitter) |
| latency_p95_ms | 318.56 ms | 314.07 ms | -4.49 ms (1.4% reduction) |
| estimated_cost | $0.037546 | $0.014984 | -$0.022562 (60% saved) |
| cache_hit_rate | 0 | 58.00% | +58.00% |

---

## 6. Redis shared cache

Explain why shared cache matters for production:

- **Why in-memory cache is insufficient for multi-instance deployments**: In-memory caches are isolated to a single container/instance. When running multiple replica instances of the gateway behind a load balancer, they cannot share their cached data. This leads to duplicate LLM calls, higher latency, and higher API billing costs across instances.
- **How `SharedRedisCache` solves this**: It stores cached values in a centralized Redis database. Any replica instance can read and write to this shared cache pool. A query cached by instance A is immediately available to be hit by instance B, maximizing cache hit rate and cost efficiency.

### Evidence of shared state

Show that two separate cache instances can see the same data:

```
tests/test_redis_cache.py::test_shared_state_across_instances PASSED
```

### Redis CLI output

```bash
# docker compose exec redis redis-cli KEYS "rl:cache:*"
1) "rl:cache:f3a1e9c2b4d8"
2) "rl:cache:e59c10f8ba3d"
3) "rl:cache:7bc4a89cd201"
```

To view cached data details:
```bash
# docker compose exec redis redis-cli HGETALL "rl:cache:f3a1e9c2b4d8"
1) "query"
2) "Summarize the refund policy for a student who missed the deadline."
3) "response"
4) "[primary] reliable answer for: Summarize the refund policy for a student who missed the deadline."
```

---

## 7. Chaos scenarios

| Scenario | Expected behavior | Observed behavior | Pass/Fail |
|---|---|---|---|
| primary_timeout_100 | All traffic fallback to backup, circuit opens | Primary circuit breaker opened, 100% of traffic correctly routed to backup. | **PASS** |
| primary_flaky_50 | Circuit oscillates, mix of primary and fallback | Circuit tripped and recovered, requests were distributed and cached correctly. | **PASS** |
| all_healthy | All requests via primary, no circuit opens | All traffic routed through primary, 0 circuit open events. | **PASS** |

---

## 8. Failure analysis (Analysis TODO)

### What failed:
During the timeout and flaky scenarios, the primary provider failed to answer requests (100% and 50% failures respectively). Our circuit breakers successfully detected the failures, tripped to the `OPEN` state, and blocked subsequent calls from reaching the degraded primary provider. However, the backup provider also experienced a small baseline failure rate (5%), which occasionally led to a minor loss of availability (reaching 98.67% instead of a perfect 100%).

### Why the fallback path worked or did not work:
The fallback path worked because it successfully rerouted requests to the backup provider while the primary breaker was `OPEN`. The cache layer also absorbed 58-64% of duplicate requests, preventing circuit trips for repeating queries. When both providers failed concurrently, the system returned a degraded static fallback message rather than throwing internal errors.

### What you would change before production:
1. **Concurrency and Probe Storms**: Under high load, when a reset timeout expires, multiple concurrent requests can transition the breaker to `HALF_OPEN` and storm the provider with probe requests. We should implement a distributed lock (e.g., using Redis `SETNX`) so only a single request probes the upstream provider at any given time.
2. **Graceful Degradation for Redis**: If the centralized Redis cache goes down, the cache layer currently raises connection errors. We should write a try-except fallback that downgrades the cache to a local in-memory cache temporarily to prevent gateway downtime.
3. **Distributed Breaker State**: Share circuit breaker counters in Redis so all gateway instances can dynamically align on a provider's health state in real-time.

---

## 9. Next steps

1. Implement **distributed lock-based probe throttling** to protect upstream providers from probe storms during HALF_OPEN transitions.
2. Add **Redis graceful degradation** to local in-memory caching during Redis instance downtime.
3. Share circuit breaker states globally using **Redis counters** (`INCR`, `EXPIRE`).