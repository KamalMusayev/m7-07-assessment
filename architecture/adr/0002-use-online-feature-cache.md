# ADR 0002: Use an Online Feature Cache for Serving-Time Features

## Status

Accepted

## Context

The recommendation model uses the user's last 30 days of browsing and purchase behavior. These features are needed during every home-screen recommendation request.

The system must support:

| Requirement | Target |
|---|---:|
| Peak traffic | 800 RPS |
| End-to-end p95 latency | 120 ms |
| Recommendation Service p95 target | 60 ms |
| Feature lookup p95 target | 20 ms |

Reading user features directly from an offline warehouse during a live request would be too slow and unreliable for this latency budget.

The main design question is where the Recommendation Service should read serving-time features from.

---

## Decision

The Recommendation Service will read serving-time user features from a **Redis-compatible Online Feature Cache**.

The cache will store features such as:

- recently viewed product categories;
- recent purchases;
- preferred brands;
- preferred price range;
- last activity timestamp;
- cold-start indicator.

The target p95 lookup time is:

```text
20 ms
```

If the cache does not contain enough user history, or if the user is new, the service will use the cold-start fallback.

---

## Alternatives Considered

### Option 1: Read features from the offline warehouse during request time

Benefits:

- Simple architecture.
- No separate online cache infrastructure.
- Uses the same source as training features.

Costs:

- Too slow for a 120 ms p95 latency budget.
- Higher risk of request timeouts.
- Not designed for high-QPS online serving.
- Poor fit for 800 RPS peak traffic.

### Option 2: Precompute all recommendations offline

Benefits:

- Very fast online serving if recommendations already exist.
- Lower online model scoring cost.
- Simpler runtime logic.

Costs:

- Recommendations may become stale.
- Harder to react to recent user behavior.
- Less flexible for continuous A/B testing.
- Requires fallback when precomputed recommendations are missing.

### Option 3: Use an online feature cache

Benefits:

- Low-latency feature reads.
- Supports recent user behavior.
- Fits the 120 ms p95 latency budget.
- Works well with online model serving.
- Allows graceful fallback when features are missing.

Costs:

- Requires cache freshness monitoring.
- Requires feature ingestion and backfill logic.
- Cache failures must be handled safely.
- Adds another production dependency.

---

## Decision Outcome

We choose **Option 3: use an Online Feature Cache**.

This decision keeps the online recommendation path fast enough for the latency target while still allowing the model to use recent personalization signals.

The Online Feature Cache is part of the serving path:

```text
Mobile App
→ API Gateway
→ Recommendation Service
→ Online Feature Cache
→ Model Runtime
→ Product Catalog
→ Ranked Recommendations
```

The offline warehouse remains part of the training and lifecycle path, not the live request path.

---

## Fallback Behavior

If the Online Feature Cache is unavailable or missing user features, the service must not fail the user experience.

Instead, it returns fallback recommendations based on:

- trending products in the user's region;
- popular products by category;
- seasonal campaigns;
- safe default business rankings.

Fallback usage is tracked with:

```text
recommendation_fallback_rate
```

A sudden increase in this metric may indicate:

- feature cache degradation;
- broken feature ingestion;
- event stream issues;
- a model runtime problem;
- unexpected cold-start traffic.

---

## Monitoring Impact

This decision requires monitoring for the feature cache and fallback path.

Important metrics:

| Metric | Purpose |
|---|---|
| `feature_cache_lookup_latency_ms` | Measures feature cache read latency |
| `feature_cache_hit_rate` | Measures how often user features are found |
| `feature_freshness_lag_seconds` | Measures how fresh serving features are |
| `recommendation_fallback_rate` | Measures how often fallback recommendations are used |
| `recommendation_latency_ms` | Measures total recommendation service latency |

The feature cache must support the target:

```text
feature_cache_lookup_latency_ms p95 <= 20 ms
```

If cache latency increases, it can cause the Recommendation Service to miss its **60 ms p95** target and the overall system to miss the **120 ms p95** end-to-end budget.

---

## Operational Impact

### Deployment

The Recommendation Service depends on the Online Feature Cache at runtime. The service must be configured with cache connection settings through environment variables or deployment configuration.

### Failure Handling

The Recommendation Service must treat cache failures as degraded mode, not total failure.

Expected behavior:

1. Try to read user features from the Online Feature Cache.
2. If features are missing or the cache call fails, use fallback recommendations.
3. Emit fallback and cache error metrics.
4. Continue returning a valid recommendation response.

### Rollback

If a new model or feature schema causes fallback rate to spike, rollback should return the system to the previous stable model and feature schema recorded in the model registry.

---

## Consequences

This decision means:

1. The latency budget must reserve time for feature cache lookup.
2. The capacity plan must include cache dependency assumptions.
3. The SLO file must track recommendation latency and success rate.
4. Monitoring must include cache hit rate, cache latency, and fallback rate.
5. The rollback runbook must include fallback spike and cache degradation as investigation points.
6. The API response should remain successful when fallback recommendations are returned, unless the whole service is unavailable.

---

## Links to Other Artifacts

This decision must stay consistent with:

- `architecture/architecture.md`
- `serving/capacity-plan.md`
- `serving/slos.yaml`
- `api/openapi.yaml`
- `monitoring/alerts.yaml`
- `runbooks/rollback.md`