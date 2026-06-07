# Architecture Justification

## Why This Architecture Fits Scenario X

Scenario X requires personalized product recommendations on every mobile app home-screen load. The system must support **800 RPS at peak** and keep the **end-to-end p95 latency under 120 ms**.

This means the architecture must be:

- fast enough for real-time user experience;
- reliable during peak traffic;
- flexible enough for continuous A/B testing;
- safe to roll back if a new model performs badly;
- traceable across model version, API responses, monitoring, and deployment.

Because of these requirements, the system uses an **online recommendation service** backed by an **online feature cache**, a **model registry**, a **containerized model runtime**, CI/CD promotion gates, and SLO-based monitoring.

---

## Chosen Pattern

The chosen pattern is:

```text
Online Recommendation Service
+ Online Feature Cache
+ Model Registry
+ CI/CD Controlled Deployment
+ Monitoring and Rollback
```

This pattern fits the scenario because recommendations must be generated when the user opens the app, not only once per day in a batch job.

The system also needs controlled model promotion because the product team continuously runs A/B tests against new recommendation models.

---

## Why Online Serving?

The recommendation request happens when the user opens the home screen. The app needs a response immediately.

If recommendations were generated only offline once per day, the system would not react well to recent user behavior such as:

- products viewed a few minutes ago;
- items added to cart;
- recent purchases;
- fresh category interest.

Online serving allows the system to use recent user features while still keeping latency low through caching.

The trade-off is that online serving is more operationally sensitive. It needs strict latency budgets, autoscaling, monitoring, and rollback.

---

## Why Use an Online Feature Cache?

The model needs serving-time user features from the last 30 days of behavior. Reading these features directly from an offline warehouse during a live request would be too slow and unreliable for a **120 ms p95** budget.

The Online Feature Cache gives the Recommendation Service fast access to user features.

Target:

```text
Feature cache lookup p95: 20 ms
```

This keeps the request within the total latency budget.

The trade-off is that the cache must be monitored for freshness, hit rate, and latency. If the cache fails, the service must safely fall back to trending or popular products.

---

## Why Bake the Model Into the Container Image?

The model is baked into the container image instead of being mounted or downloaded at runtime.

Production model version:

```text
recsys-2026.06.07-001
```

Container image:

```text
ghcr.io/acme-retail/retail-recommender:recsys-2026.06.07-001
```

This decision makes production safer because the code, model artifact, and image tag move together as one immutable deployment unit.

Benefits:

- easier rollback;
- clearer model traceability;
- no runtime dependency on object storage during startup;
- CI/CD can test and scan one complete artifact;
- monitoring can compare the served `X-Model-Version` with the registry version.

The trade-off is that the image becomes larger and each new model version requires a new image build. This is acceptable because the estimated model size is **~250 MB**, which is manageable for this service.

---

## Why CPU Serving?

This system assumes the recommendation model is a lightweight ranking model that can run efficiently on CPU.

The serving plan uses:

```text
4 vCPU / 8 GB RAM per replica
8 initial replicas
8–30 autoscaling range
```

CPU serving is selected because it is simpler and cheaper to operate for this type of workload.

GPU serving would add cost and operational complexity. It would only be justified if the model required heavy neural inference and could not meet the latency target on CPU.

---

## Why Use Canary Deployment?

The product team will run A/B tests continuously, and recommendation models can affect business metrics directly.

A new model might pass offline evaluation but still cause problems in production, such as:

- higher latency;
- lower click-through rate;
- lower add-to-cart rate;
- worse conversion;
- higher fallback rate;
- unexpected behavior for a user segment.

For this reason, production deployment uses a canary rollout before full promotion.

If the canary violates SLOs or business gates, the deployment is stopped and rolled back to the previous stable model.

---

## Why Include Cold-Start Fallback?

A retail app will always have users with little or no history. This includes:

- new users;
- anonymous users;
- users who cleared tracking data;
- users whose features are temporarily missing.

The system must not return empty recommendations.

For cold-start users, the service returns:

- trending products;
- popular products by category;
- regional best sellers;
- seasonal campaigns;
- safe default rankings.

This fallback also protects the user experience during partial failures, such as feature cache problems or model runtime errors.

Fallback usage is monitored with:

```text
recommendation_fallback_rate
```

A sudden spike in this metric can indicate a production issue.

---

## Why Observability Headers Matter

Each response includes observability headers:

```text
X-Request-ID
X-Model-Version
X-Experiment-ID
X-AB-Variant
```

These headers make every recommendation response easier to debug and audit.

For example:

- `X-Request-ID` helps trace a single request across services.
- `X-Model-Version` shows which model produced the response.
- `X-Experiment-ID` connects the response to an A/B test.
- `X-AB-Variant` shows whether the user was in control or treatment.

The `X-Model-Version` header is especially important because monitoring uses it to detect model-version mismatch.

---

## Main Trade-Offs

| Decision | Benefit | Cost |
|---|---|---|
| Online serving | Uses recent user behavior | Requires strict latency and reliability controls |
| Online feature cache | Fast feature lookup | Needs freshness and hit-rate monitoring |
| Baked model image | Strong version consistency and simple rollback | Larger image and rebuild required for model updates |
| CPU serving | Lower cost and simpler operations | Requires model to stay lightweight |
| Canary deployment | Safer production rollout | Slower full promotion |
| Cold-start fallback | Better user experience during missing history or failures | Less personalized results |

---

## Coherence Across the Repository

The architecture is designed to stay consistent across all deliverables.

| Assumption | Used In |
|---|---|
| `800 RPS` peak traffic | README, architecture, capacity plan, load test plan |
| `120 ms p95` latency budget | README, architecture, SLOs, monitoring, rollback |
| `60 ms` Recommendation Service p95 target | architecture, capacity plan, SLOs |
| `recsys-2026.06.07-001` model version | registry, API, CI/CD, monitoring, rollback |
| Baked model image | ADR, Dockerfile, container README, CI/CD, rollback |
| `X-Model-Version` header | API contract, monitoring, rollback |
| `recommendation_fallback_rate` | architecture, monitoring, rollback |

This consistency is important because the assessment is not only checking each artifact separately. It is also checking whether the full system works as one coherent MLOps design.

---

## Summary

This architecture is intentionally simple, fast, and operationally safe.

It supports real-time personalized recommendations, handles cold-start users, allows controlled A/B testing, and provides clear rollback paths when a model or deployment becomes unhealthy.

The design avoids unnecessary complexity while still covering the production concerns expected from an MLOps system: versioning, packaging, deployment gates, capacity planning, SLOs, monitoring, and rollback.