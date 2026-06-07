# Container Image Plan

## Purpose

This directory describes the container packaging plan for the `retail-home-recommender` inference service.

The container image is responsible for running the online recommendation API in production. It serves the active production model version:

```text
recsys-2026.06.07-001
```

The image is expected to be deployed as:

```text
ghcr.io/acme-retail/retail-recommender:recsys-2026.06.07-001
```

This image tag matches the production model version stored in the model registry.

---

## Image Strategy

The model is **baked into the container image**.

This means the final runtime image contains:

- the application code;
- the Python runtime dependencies;
- the production model artifact;
- the startup command;
- healthcheck configuration.

The model is not downloaded from object storage at container startup.

---

## Why Bake the Model Into the Image?

Baking the model into the image makes the deployment easier to reason about.

The same immutable image contains both:

```text
application code
+
model artifact
```

This gives the system stronger consistency between:

- model registry entry;
- image tag;
- deployed runtime;
- `X-Model-Version` API header;
- monitoring alerts;
- rollback runbook.

If the model needs to be rolled back, the on-call engineer redeploys the previous stable image:

```text
ghcr.io/acme-retail/retail-recommender:recsys-2026.05.31-004
```

This is simpler and safer than changing a runtime model mount or downloading a different model during startup.

---

## Trade-Off: Bake vs Mount

| Strategy | Benefits | Costs |
|---|---|---|
| Bake model into image | Strong version consistency, simple rollback, one artifact for CI/CD | Larger image, rebuild needed for every model version |
| Mount/download model at runtime | Smaller app image, model can change independently | Harder traceability, more startup failure modes, rollback is more complex |

For this scenario, baking the model into the image is preferred because the estimated model size is manageable and production traceability is more important than minimizing image size.

---

## Base Image

The Dockerfile uses:

```text
python:3.11-slim-bookworm
```

Reasoning:

- smaller than the full Python image;
- stable Debian base;
- enough for a FastAPI-style inference service;
- compatible with common ML runtime libraries;
- easier to scan and patch than a large general-purpose image.

The Dockerfile uses a multi-stage build:

1. **Builder stage** — builds dependency wheels.
2. **Runtime stage** — installs only runtime dependencies, application code, and the baked model artifact.

This keeps the final image cleaner and easier to review.

---

## Expected Runtime Layout

The final runtime image uses this layout:

```text
/app
  app/
    main.py

/opt/model
  retail-home-recommender/
    model artifact files
```

Important environment variables:

| Variable | Value |
|---|---|
| `SERVICE_NAME` | `retail-recommender` |
| `MODEL_NAME` | `retail-home-recommender` |
| `MODEL_VERSION` | `recsys-2026.06.07-001` |
| `MODEL_PATH` | `/opt/model/retail-home-recommender` |
| `PORT` | `8080` |

The application must use `MODEL_VERSION` when returning the `X-Model-Version` response header.

---

## Image Size Estimate

Estimated image size:

| Layer / Component | Estimate |
|---|---:|
| Python slim base | ~80 MB |
| Runtime dependencies | ~180 MB |
| Application code | ~20 MB |
| Model artifact | ~250 MB |
| OS metadata and overhead | ~70 MB |
| **Estimated total** | **~600 MB** |

A ~600 MB image is acceptable for this system because:

- the model is not extremely large;
- deployments are controlled through CI/CD;
- rollback safety is more important than minimizing image size;
- production uses long-running pods, not high-frequency cold starts.

The image size limit in the model registry gate is:

```text
300 MB maximum model artifact size
```

This keeps the final container image within a reasonable operational range.

---

## Security Plan

The Dockerfile follows basic production security practices:

- uses a slim runtime image;
- separates builder and runtime stages;
- removes build tools from the final runtime image;
- runs as a non-root `app` user;
- avoids writing `.pyc` files;
- avoids pip cache;
- exposes only port `8080`;
- includes a healthcheck;
- keeps model and application version explicit.

The CI/CD pipeline is expected to run:

- Dockerfile linting;
- dependency scanning;
- container image vulnerability scanning;
- OpenAPI contract validation;
- staging deployment tests.

---

## Healthcheck

The image defines a healthcheck against:

```text
GET /health
```

Expected behavior:

- returns healthy if the service process is alive;
- does not require the model to be fully warmed;
- should be lightweight and safe to call frequently.

Readiness is handled separately by:

```text
GET /ready
```

The readiness endpoint should confirm that:

- the model artifact is loaded;
- the feature cache connection is reachable or safely degraded;
- the service can accept recommendation traffic.

---

## Runtime Command

The container starts the API service with:

```text
uvicorn app.main:app --host 0.0.0.0 --port 8080
```

This assumes a FastAPI-style service implementation.

The assessment does not require model training or a full application implementation, but the Dockerfile is written so another team could implement the service behind the documented API contract.

---

## Version Consistency

The container version must stay consistent with the rest of the repository.

| Artifact | Expected Value |
|---|---|
| Model registry version | `recsys-2026.06.07-001` |
| Docker image tag | `ghcr.io/acme-retail/retail-recommender:recsys-2026.06.07-001` |
| Runtime `MODEL_VERSION` | `recsys-2026.06.07-001` |
| API `X-Model-Version` header | `recsys-2026.06.07-001` |
| Monitoring expected version | `recsys-2026.06.07-001` |
| Rollback previous stable version | `recsys-2026.05.31-004` |

If the served model version differs from the expected registry version, monitoring must raise:

```text
ModelVersionMismatch
```

---

## Operational Notes

The image should be deployed behind the API Gateway and should not be exposed directly to public traffic.

Recommended production configuration:

| Setting | Value |
|---|---:|
| CPU per replica | 4 vCPU |
| Memory per replica | 8 GB |
| Initial replicas | 8 |
| Autoscaling range | 8–30 replicas |
| Container port | 8080 |
| Model artifact size | ~250 MB |
| Expected safe throughput per replica | ~120 RPS |

These values match the capacity plan and SLO assumptions used elsewhere in the repository.

---

## Summary

The container plan uses a simple and safe production packaging strategy:

```text
multi-stage Dockerfile
+ slim Python runtime
+ non-root user
+ baked model artifact
+ explicit model version
+ healthcheck
+ immutable image tag
```

This keeps the model-serving deployment traceable, testable, and easy to roll back.