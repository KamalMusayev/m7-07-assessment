# Load Test Plan

## Purpose

This document defines the load test plan for the `retail-home-recommender` recommendation service.

The goal is to prove that the service can handle the expected production peak of **800 RPS** while staying within the **120 ms end-to-end p95 latency budget**.

The staging load test uses **900 RPS** to provide safety margin above the expected production peak.

---

## Scope

The load test covers the main online recommendation endpoint:

```text
POST /v1/recommendations
```

This is the critical mobile home-screen path.

The batch and async endpoints are tested separately for correctness, but they are not part of the critical low-latency mobile path.

---

## Production Targets

| Item | Target |
|---|---:|
| Expected production peak | 800 RPS |
| Load test peak | 900 RPS |
| End-to-end p95 latency | ≤ 120 ms |
| Recommendation Service p95 latency | ≤ 60 ms |
| Feature cache lookup p95 | ≤ 20 ms |
| Model scoring p95 | ≤ 35 ms |
| Error rate | < 0.5% |
| Test duration at peak | 30 minutes |
| Initial replicas | 8 |
| Autoscaling range | 8–30 replicas |
| Model version | `recsys-2026.06.07-001` |

---

## Test Environment

Load tests run in staging before production canary.

Staging must match production as closely as possible:

| Setting | Value |
|---|---:|
| CPU per replica | 4 vCPU |
| Memory per replica | 8 GB |
| Initial replicas | 8 |
| Autoscaling range | 8–30 replicas |
| Container image | `ghcr.io/acme-retail/retail-recommender:recsys-2026.06.07-001` |
| Model packaging | Baked into container image |
| Feature cache | Staging cache with production-like data volume |
| Product catalog | Staging catalog with production-like product count |

---

## Test Data

The test must include a realistic mix of users.

| User Type | Share |
|---|---:|
| Returning users with feature cache hits | 75% |
| Cold-start users | 15% |
| Users with partial feature history | 8% |
| Users with simulated cache miss | 2% |

This mix is important because the service must handle both personalized and fallback recommendation paths.

---

## Request Shape

Each request should follow the OpenAPI contract.

Example:

```json
{
  "request_id": "req_load_000001",
  "user_id": "user_12345",
  "session_id": "sess_abc123",
  "context": {
    "region": "AZ-BA",
    "locale": "en-US",
    "device_type": "ios",
    "placement": "home_screen"
  },
  "limit": 10,
  "include_explanations": false
}
```

Cold-start requests may omit `user_id` and rely on `session_id`.

---

## Load Stages

| Stage | Duration | Target RPS | Purpose |
|---|---:|---:|---|
| Warmup | 5 minutes | 100 RPS | Warm service and caches |
| Ramp 1 | 10 minutes | 300 RPS | Simulate average traffic |
| Ramp 2 | 10 minutes | 600 RPS | Simulate high traffic |
| Peak | 30 minutes | 900 RPS | Validate above production peak |
| Spike | 5 minutes | 1,100 RPS | Short stress spike |
| Recovery | 10 minutes | 300 RPS | Confirm service stabilizes |

The official pass/fail decision is based mainly on the **30-minute 900 RPS peak stage**.

---

## Pass Criteria

The model version can continue toward production canary only if all required gates pass.

| Gate | Required Result |
|---|---:|
| End-to-end p95 latency | ≤ 120 ms |
| Recommendation Service p95 latency | ≤ 60 ms |
| Feature cache lookup p95 | ≤ 20 ms |
| Model scoring p95 | ≤ 35 ms |
| Error rate | < 0.5% |
| Fallback rate increase | ≤ 25% over baseline |
| No model-version mismatch | Required |
| Service availability during test | ≥ 99.9% |
| Response schema validity | 100% of sampled responses valid |
| Required headers present | 100% of sampled responses |

Required response headers:

```text
X-Request-ID
X-Model-Version
X-Experiment-ID
X-AB-Variant
```

Expected model version header:

```text
X-Model-Version: recsys-2026.06.07-001
```

---

## Fail Criteria

The load test fails if any of these happen during the 900 RPS peak stage:

- end-to-end p95 latency exceeds 120 ms;
- Recommendation Service p95 latency exceeds 60 ms;
- error rate reaches or exceeds 0.5%;
- feature cache p95 latency exceeds 20 ms;
- model scoring p95 latency exceeds 35 ms;
- fallback rate increases more than 25% over baseline;
- any response serves the wrong `X-Model-Version`;
- the service becomes unavailable;
- response schema validation fails.

A failed load test blocks production canary.

---

## Metrics to Collect

| Metric | Purpose |
|---|---|
| `http_requests_total` | Request count by endpoint and status |
| `http_request_duration_ms` | End-to-end request latency |
| `recommendation_service_duration_ms` | Internal service latency |
| `feature_cache_lookup_duration_ms` | Feature cache latency |
| `model_scoring_duration_ms` | Model scoring latency |
| `recommendation_fallback_total` | Fallback usage |
| `feature_cache_hit_rate` | Cache effectiveness |
| `model_version_info` | Active served model version |
| `container_cpu_usage` | CPU saturation |
| `container_memory_usage` | Memory pressure |
| `pod_restart_count` | Stability signal |

---

## Response Validation

A sampled set of responses must be validated against `api/openapi.yaml`.

Validation checks:

- response is valid JSON;
- response matches the OpenAPI schema;
- `recommendations` is an array;
- product ranks are positive integers;
- scores are between 0 and 1;
- `model_version` equals `recsys-2026.06.07-001`;
- `X-Model-Version` header equals `recsys-2026.06.07-001`;
- fallback responses still contain valid recommendations.

---

## Dependency Behavior

The load test should confirm that dependency behavior is safe.

| Dependency | Expected Test Behavior |
|---|---|
| Online Feature Cache | p95 lookup ≤ 20 ms |
| Model Runtime | p95 scoring ≤ 35 ms |
| Product Catalog | filtering remains within latency budget |
| Experiment Service | assignment does not block response |
| Monitoring backend | metrics emission does not block response |

If dependencies slow down, the service should prefer bounded timeouts and fallback logic instead of hanging requests.

---

## Canary Readiness

Passing the load test does not automatically promote the model to full production. It only allows the model to move to production canary.

Canary still requires:

- approval from Product Owner;
- approval from ML Lead;
- no critical SLO regression;
- no critical business KPI regression;
- no `ModelVersionMismatch` alert.

---

## Result Template

After the test, the team should record results in this format:

```text
Test date:
Model version:
Container image:
Peak RPS reached:
End-to-end p95 latency:
Recommendation Service p95 latency:
Feature cache p95 latency:
Model scoring p95 latency:
Error rate:
Fallback rate:
Model-version mismatch detected:
OpenAPI response validation:
Pass/fail:
Reviewer:
Notes:
```

Example expected passing result:

```text
Test date: 2026-06-07
Model version: recsys-2026.06.07-001
Container image: ghcr.io/acme-retail/retail-recommender:recsys-2026.06.07-001
Peak RPS reached: 900
End-to-end p95 latency: 112 ms
Recommendation Service p95 latency: 56 ms
Feature cache p95 latency: 18 ms
Model scoring p95 latency: 31 ms
Error rate: 0.2%
Fallback rate: +8% over baseline
Model-version mismatch detected: no
OpenAPI response validation: passed
Pass/fail: pass
Reviewer: ML Platform Engineer
Notes: Safe to proceed to production canary.
```

---

## Summary

The load test proves that the recommendation service can handle more than the expected production peak.

The key gates are:

```text
900 RPS for 30 minutes
p95 end-to-end latency ≤ 120 ms
Recommendation Service p95 ≤ 60 ms
error rate < 0.5%
X-Model-Version = recsys-2026.06.07-001
```

If these gates pass, the model can move to production canary. If any gate fails, production rollout is blocked.