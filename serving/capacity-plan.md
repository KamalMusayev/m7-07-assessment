# Capacity Plan

## Purpose

This document explains the serving capacity plan for the `retail-home-recommender` recommendation service.

The system must support personalized recommendations for a B2C retail mobile app with the following production targets:

| Item | Target |
|---|---:|
| Peak traffic | 800 RPS |
| End-to-end p95 latency budget | 120 ms |
| Recommendation Service p95 target | 60 ms |
| Availability SLO | 99.9% monthly |
| Successful response SLO | 99.5% |
| Model size estimate | ~250 MB |

The goal is to choose a serving configuration that is simple, scalable, and consistent with the SLOs, load test plan, monitoring alerts, and rollback runbook.

---

## Serving Assumptions

The recommendation service runs as a containerized CPU-based API service.

| Setting | Value |
|---|---:|
| CPU per replica | 4 vCPU |
| Memory per replica | 8 GB |
| Initial replicas | 8 |
| Autoscaling range | 8–30 replicas |
| Expected safe throughput per replica | ~120 RPS |
| Container port | 8080 |
| Model packaging | Baked into container image |
| Production model version | `recsys-2026.06.07-001` |

The model is assumed to be a lightweight ranking model suitable for CPU inference. GPU serving is not used because the expected model size and inference target can be handled with CPU replicas.

---

## Traffic Profile

The main online endpoint is:

```text
POST /v1/recommendations
```

Expected traffic:

| Traffic Type | Assumption |
|---|---:|
| Average traffic | 300 RPS |
| Peak traffic | 800 RPS |
| Load test peak | 900 RPS |
| Batch endpoint traffic | Internal, not part of mobile critical path |
| Async endpoint traffic | Internal, lower priority |

The capacity plan is based on the peak online serving path, not the async or batch path.

---

## Replica Calculation

Expected safe throughput per replica:

```text
120 RPS per replica
```

Required replicas at peak:

```text
800 RPS / 120 RPS per replica = 6.67 replicas
```

Rounded up:

```text
7 replicas
```

Chosen initial production replicas:

```text
8 replicas
```

Reasoning:

- 7 replicas is the mathematical minimum.
- 8 replicas provides safety margin.
- 8 replicas prevents the service from running at the edge of capacity.
- 8 replicas matches the minimum autoscaling setting used in production.

Autoscaling range:

```text
8–30 replicas
```

This gives the service room to handle traffic spikes, uneven load distribution, or temporary slowdowns in dependencies.

---

## Latency Budget

The system has a strict **120 ms end-to-end p95 latency budget**.

| Step | Target p95 |
|---|---:|
| API Gateway and network overhead | 20 ms |
| Online feature cache lookup | 20 ms |
| Model scoring | 35 ms |
| Product catalog filtering and business rules | 15 ms |
| Response serialization | 10 ms |
| Safety buffer | 20 ms |
| **Total** | **120 ms** |

The Recommendation Service itself targets:

```text
60 ms p95
```

This service target includes:

- feature cache lookup;
- model scoring;
- catalog filtering;
- response preparation.

The remaining time is reserved for gateway, network, and safety buffer.

---

## Dependency Capacity

The Recommendation Service depends on several systems during online serving.

| Dependency | Expected Behavior | Capacity Concern |
|---|---|---|
| Online Feature Cache | Fast user feature lookup | Must keep p95 lookup ≤ 20 ms |
| Model Runtime | Scores candidate products | Must keep p95 scoring ≤ 35 ms |
| Product Catalog Service | Filters unavailable products | Must keep p95 filtering ≤ 15 ms |
| Experiment Service | Assigns A/B variant | Must be low latency and cacheable |
| Monitoring backend | Receives metrics/logs | Must not block user response |

The service must degrade safely if some dependencies are slow or unavailable.

Example:

- if feature cache is missing user history, use fallback recommendations;
- if experiment assignment fails, default to control variant;
- if product catalog is partially unavailable, use cached safe product filters when possible.

---

## Memory Plan

Each replica has:

```text
8 GB RAM
```

Estimated memory usage:

| Component | Estimate |
|---|---:|
| Python runtime and API service | ~500 MB |
| Runtime dependencies | ~700 MB |
| Loaded model artifact | ~250 MB |
| Feature processing buffers | ~500 MB |
| Request concurrency overhead | ~1 GB |
| Metrics/logging overhead | ~250 MB |
| Safety buffer | ~4.8 GB |
| **Total reserved** | **8 GB** |

The memory allocation is intentionally conservative. It leaves enough headroom for traffic bursts, dependency retries, and model warmup.

---

## CPU Plan

Each replica has:

```text
4 vCPU
```

The expected safe throughput is:

```text
~120 RPS per replica
```

This assumes the model scoring path remains under:

```text
35 ms p95
```

CPU autoscaling should use both:

- CPU utilization;
- request latency.

Recommended autoscaling signals:

| Signal | Scale Out When |
|---|---|
| CPU utilization | > 65% for 5 minutes |
| Recommendation Service p95 latency | > 60 ms for 5 minutes |
| Request queue depth | Increasing for 5 minutes |
| RPS per replica | > 120 RPS |

Latency-based scaling is important because CPU alone may not catch dependency-driven slowdowns.

---

## Autoscaling Policy

Recommended production autoscaling:

| Setting | Value |
|---|---:|
| Minimum replicas | 8 |
| Maximum replicas | 30 |
| Target CPU utilization | 65% |
| Target RPS per replica | 120 |
| Scale-out stabilization | 2 minutes |
| Scale-in stabilization | 10 minutes |

Scale-out should be faster than scale-in. This protects the service during traffic spikes and avoids removing capacity too quickly.

---

## Load Test Target

The staging load test should exceed expected production peak.

| Item | Value |
|---|---:|
| Expected production peak | 800 RPS |
| Load test peak | 900 RPS |
| Test duration at peak | 30 minutes |
| Max end-to-end p95 latency | 120 ms |
| Max Recommendation Service p95 latency | 60 ms |
| Max error rate | 0.5% |

Using **900 RPS** gives a 12.5% safety margin above the expected 800 RPS peak.

---

## Cost Estimate

Estimated monthly cost for the serving layer:

| Component | Estimate |
|---|---:|
| 8 CPU serving replicas | ~$950/month |
| Online feature cache | ~$350/month |
| Monitoring/logging/tracing | ~$200/month |
| Image registry and storage | ~$50/month |
| Network and operational buffer | ~$50/month |
| **Estimated monthly total** | **~$1,600/month** |

This is a planning estimate, not a vendor quote. The purpose is to show that the design is operationally reasonable for the expected traffic.

---

## Failure and Degraded Mode Capacity

The system should continue serving valid responses during partial failures.

| Situation | Behavior |
|---|---|
| Feature cache miss | Use fallback recommendations |
| Feature cache latency high | Use timeout and fallback path |
| Model runtime error | Return fallback recommendations |
| Experiment service unavailable | Default to control variant |
| Product catalog slow | Use cached safe filters if available |
| Traffic exceeds 800 RPS | Autoscale up to 30 replicas |

Fallback responses still count as successful responses if the API returns a valid recommendation list. However, fallback usage is tracked separately because a spike may indicate a production issue.

Important metric:

```text
recommendation_fallback_rate
```

A fallback increase of more than **25% over baseline** should trigger investigation and may trigger rollback if caused by a new model or feature schema.

---

## Capacity Risks

| Risk | Mitigation |
|---|---|
| Feature cache becomes slow | Timeout quickly, use fallback, alert on cache latency |
| Model scoring exceeds 35 ms p95 | Block promotion during load test or rollback |
| Traffic exceeds expected peak | Autoscale to 30 replicas |
| Image pull is slow during deploy | Keep minimum replicas available during rolling update |
| New model increases memory usage | Enforce model size gate in registry |
| Batch jobs compete with online traffic | Keep batch/async lower priority than sync endpoint |

---

## Capacity Gates

A model version cannot be promoted to production unless staging passes these gates:

| Gate | Required Result |
|---|---:|
| Staging peak load | 900 RPS |
| End-to-end p95 latency | ≤ 120 ms |
| Recommendation Service p95 latency | ≤ 60 ms |
| Error rate | < 0.5% |
| Feature cache lookup p95 | ≤ 20 ms |
| Model scoring p95 | ≤ 35 ms |
| Model artifact size | ≤ 300 MB |
| Test duration at peak | ≥ 30 minutes |

These gates must match the lifecycle, SLO, load test, monitoring, and rollback documents.

---

## Summary

The service starts with **8 replicas** because the peak capacity calculation requires at least **7 replicas** and one extra replica gives safety margin.

The capacity plan is based on:

```text
800 RPS peak traffic
120 ms end-to-end p95 latency
60 ms Recommendation Service p95 latency
120 RPS safe throughput per replica
8 initial replicas
8–30 autoscaling range
```

This plan keeps the recommendation system simple, scalable, and aligned with the rest of the MLOps design.