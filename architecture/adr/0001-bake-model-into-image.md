# ADR 0001: Bake the Model Into the Container Image

## Status

Accepted

## Context

The recommendation service must serve personalized product recommendations at **800 RPS peak** while keeping the **end-to-end p95 latency under 120 ms**.

The active production model must be traceable across the full MLOps system:

- model registry;
- container image tag;
- CI/CD deployment;
- API response headers;
- monitoring alerts;
- rollback runbook.

The current production model is:

| Field | Value |
|---|---|
| Model name | `retail-home-recommender` |
| Model version | `recsys-2026.06.07-001` |
| Container image | `ghcr.io/acme-retail/retail-recommender:recsys-2026.06.07-001` |
| Estimated model size | ~250 MB |

The main packaging question is whether the model should be baked into the container image or loaded from external storage at runtime.

---

## Decision

The model artifact will be **baked into the container image** during the CI/CD build.

The deployed image tag will include the model version:

```text
ghcr.io/acme-retail/retail-recommender:recsys-2026.06.07-001
```

The service will expose the active model version in every recommendation response through:

```text
X-Model-Version: recsys-2026.06.07-001
```

---

## Alternatives Considered

### Option 1: Bake the model into the image

The container image contains both the application code and the model artifact.

Benefits:

- The application code and model version are deployed as one immutable artifact.
- Rollback is simple because the previous image contains the previous stable model.
- CI/CD can test, scan, tag, and promote one complete artifact.
- The service does not depend on object storage during startup.
- Monitoring can compare the served `X-Model-Version` with the expected registry version.

Costs:

- The image becomes larger.
- Every new model version requires a new image build.
- Pulling images may take longer than using a very small app-only image.

### Option 2: Load the model from external storage at runtime

The container starts first, then downloads or mounts the model artifact from object storage or a shared volume.

Benefits:

- The application image is smaller.
- The model can be updated without rebuilding the application image.
- Multiple services could theoretically reuse the same model artifact.

Costs:

- It is harder to prove exactly which model is running in each pod.
- Rollback has more moving parts.
- Startup can fail if object storage or mounted storage is unavailable.
- Cold starts may be slower.
- CI/CD cannot fully validate the final runtime artifact as one unit.

---

## Decision Outcome

We choose **Option 1: bake the model into the container image**.

This is the safer choice for this assessment because it gives strong consistency between:

- the model registry version;
- the image tag;
- the deployed runtime;
- the `X-Model-Version` API header;
- monitoring alerts;
- rollback steps.

The image size increase is acceptable because the model is estimated to be **~250 MB**, which is manageable for this service.

---

## Consequences

This decision means:

1. The Dockerfile must copy the model artifact into the final runtime image.
2. The model registry must store the exact image tag for each model version.
3. CI/CD must tag images using the same model version format.
4. The service must return the active model version in the `X-Model-Version` response header.
5. Monitoring must raise `ModelVersionMismatch` if the served version differs from the expected production version.
6. Rollback must redeploy the previous stable image, not only switch a config value.

---

## Operational Impact

### Deployment

Each new model version creates a new container image.

Example:

```text
ghcr.io/acme-retail/retail-recommender:recsys-2026.06.07-001
```

This image is deployed first to staging, then to canary, and finally to full production after gates pass.

### Rollback

Rollback is simple and predictable.

If the current model causes errors, latency regression, fallback spikes, or business KPI regression, the on-call engineer redeploys the previous stable image recorded in the model registry.

### Monitoring

The service exposes the active model version through the API and metrics.

Expected production version:

```text
recsys-2026.06.07-001
```

If production serves a different version, the monitoring rule `ModelVersionMismatch` fires.

---

## Links to Other Artifacts

This decision must stay consistent with:

- `container/Dockerfile`
- `container/README.md`
- `lifecycle/model-registry.yaml`
- `api/openapi.yaml`
- `cicd/.github/workflows/deploy-model.yml`
- `monitoring/alerts.yaml`
- `runbooks/rollback.md`