# Rollback Runbook

## Purpose

This runbook explains how to roll back the `retail-home-recommender` service when a production deployment becomes unhealthy.

Current production model:

```text
recsys-2026.06.07-001
```

Previous stable rollback target:

```text
recsys-2026.05.31-004
```

Previous stable image:

```text
ghcr.io/acme-retail/retail-recommender:recsys-2026.05.31-004
```

---

## When to Use This Runbook

Use this runbook when one or more rollback-related alerts fire after a deployment, canary, or model promotion.

Rollback-related alerts:

| Alert | Rollback Trigger |
|---|---|
| `HighLatencyBurnRate` | End-to-end p95 latency > 120 ms |
| `HighErrorRateBurnRate` | Error rate > 0.5% |
| `RecommendationFallbackSpike` | Fallback rate increases > 25% over baseline |
| `ModelScoringLatencyHigh` | Model scoring p95 > 35 ms |
| `ModelVersionMismatch` | Served model version is not `recsys-2026.06.07-001` |
| `CanaryLatencyRegression` | Canary p95 latency > 120 ms |
| `CanaryBusinessKPIRegression` | Canary CTR drops more than 5% vs control |

---

## Severity

Treat rollback as urgent if:

```text
severity = page
```

or if customer-facing recommendation quality is clearly degraded.

Fallback responses count as successful API responses, but a large fallback spike still requires investigation because users are receiving less personalized recommendations.

---

## Pre-Rollback Checklist

Before rolling back, confirm these items quickly:

- [ ] Identify the active alert name.
- [ ] Confirm the affected endpoint is `POST /v1/recommendations`.
- [ ] Confirm the active model version.
- [ ] Check whether the issue started after deploying `recsys-2026.06.07-001`.
- [ ] Check whether canary or full production traffic is affected.
- [ ] Confirm the previous stable version is `recsys-2026.05.31-004`.
- [ ] Notify Product Owner, ML Lead, and SRE channel.

Expected production header:

```text
X-Model-Version: recsys-2026.06.07-001
```

Rollback target header after rollback:

```text
X-Model-Version: recsys-2026.05.31-004
```

---

## Rollback Decision Guide

| Situation | Action |
|---|---|
| Canary only is unhealthy | Stop canary and route traffic back to previous stable version |
| Full production is unhealthy after promotion | Roll back all production traffic |
| `ModelVersionMismatch` fires | Redeploy expected image or roll back to previous stable image |
| Error rate or latency regression is caused by dependency outage | Degrade safely first; rollback only if new model/deployment is involved |
| Fallback spike is linked to new model or feature schema | Roll back model and feature schema |
| Feature drift only, without acute SLO impact | Do not immediately roll back; investigate and consider retraining |

---

## Rollback Steps

### 1. Pause rollout

If a canary or progressive rollout is still running:

```text
Pause rollout immediately.
Do not increase traffic percentage.
```

Expected result:

```text
No additional traffic is sent to the unhealthy version.
```

---

### 2. Set rollback image

Rollback image:

```text
ghcr.io/acme-retail/retail-recommender:recsys-2026.05.31-004
```

Replace the active production image with the previous stable image.

Example deployment action:

```bash
kubectl set image deployment/retail-recommender \
  retail-recommender=ghcr.io/acme-retail/retail-recommender:recsys-2026.05.31-004 \
  --namespace production
```

---

### 3. Verify rollout status

Check that the deployment completed:

```bash
kubectl rollout status deployment/retail-recommender \
  --namespace production
```

Expected result:

```text
deployment "retail-recommender" successfully rolled out
```

---

### 4. Verify health and readiness

Check health:

```bash
curl -s https://api.acme-retail.example.com/health
```

Check readiness:

```bash
curl -s https://api.acme-retail.example.com/ready
```

Expected readiness model version:

```text
recsys-2026.05.31-004
```

---

### 5. Verify recommendation response header

Send a test recommendation request and confirm the model version header:

```bash
curl -i -X POST https://api.acme-retail.example.com/v1/recommendations \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $API_KEY" \
  -d @api/examples/single-request.json
```

Expected header after rollback:

```text
X-Model-Version: recsys-2026.05.31-004
```

---

### 6. Watch metrics

Observe production metrics for at least 30 minutes.

Required recovery targets:

| Metric | Expected After Rollback |
|---|---:|
| End-to-end p95 latency | ≤ 120 ms |
| Recommendation Service p95 latency | ≤ 60 ms |
| Error rate | < 0.5% |
| Model scoring p95 | ≤ 35 ms |
| Fallback rate | Returns near baseline |
| Model-version mismatch | 0 mismatches |

If metrics do not recover, continue incident investigation. The issue may be outside the model deployment.

---

### 7. Close or update rollout

After rollback is confirmed:

- [ ] Mark `recsys-2026.06.07-001` as rolled back in the model registry.
- [ ] Keep `recsys-2026.05.31-004` as production.
- [ ] Block further promotion of `recsys-2026.06.07-001`.
- [ ] Open a follow-up investigation ticket.
- [ ] Record the alert, timeline, cause, and decision.

---

## Post-Rollback Notes

Record the incident summary:

```text
Incident date:
Detected alert:
Bad model version:
Rollback version:
Rollback image:
Start time:
End time:
Primary symptom:
Metrics before rollback:
Metrics after rollback:
Decision owner:
Follow-up ticket:
```

---

## Do Not Do

- Do not promote the canary to 100% traffic while a rollback-related alert is active.
- Do not change the model version manually without updating the registry.
- Do not ignore `ModelVersionMismatch`.
- Do not treat fallback spikes as harmless just because responses are HTTP 200.
- Do not roll back repeatedly without checking whether the issue is caused by a dependency outage.

---

## Summary

Rollback target:

```text
recsys-2026.05.31-004
```

Rollback image:

```text
ghcr.io/acme-retail/retail-recommender:recsys-2026.05.31-004
```

Rollback is required when latency, error rate, fallback rate, model scoring latency, canary KPIs, or model-version consistency violates the thresholds defined in `monitoring/alerts.yaml` and `serving/slos.yaml`.