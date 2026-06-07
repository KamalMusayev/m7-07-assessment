# syntax=docker/dockerfile:1.7

ARG PYTHON_VERSION=3.11

# -----------------------------------------------------------------------------
# Builder stage
# Builds Python dependency wheels so the runtime image can stay smaller and cleaner.
# -----------------------------------------------------------------------------
FROM python:${PYTHON_VERSION}-slim-bookworm AS builder

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1

WORKDIR /build

RUN apt-get update \
    && apt-get install --no-install-recommends -y build-essential \
    && rm -rf /var/lib/apt/lists/*

# In a real implementation this file would contain runtime dependencies such as:
# fastapi, uvicorn, pydantic, numpy, scikit-learn/onnxruntime, prometheus-client.
COPY container/requirements.txt ./requirements.txt

RUN python -m pip install --upgrade pip \
    && pip wheel --wheel-dir /wheels -r requirements.txt


# -----------------------------------------------------------------------------
# Runtime stage
# Contains only the app, installed wheels, and the baked model artifact.
# -----------------------------------------------------------------------------
FROM python:${PYTHON_VERSION}-slim-bookworm AS runtime

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PATH="/venv/bin:$PATH" \
    SERVICE_NAME="retail-recommender" \
    MODEL_NAME="retail-home-recommender" \
    MODEL_VERSION="recsys-2026.06.07-001" \
    MODEL_PATH="/opt/model/retail-home-recommender" \
    PORT="8080"

WORKDIR /app

# Create a dedicated non-root user for the service.
RUN groupadd --system app \
    && useradd --system --gid app --home-dir /app app \
    && python -m venv /venv

COPY --from=builder /wheels /wheels
COPY container/requirements.txt ./requirements.txt

RUN pip install --no-cache-dir --no-index --find-links=/wheels -r requirements.txt \
    && rm -rf /wheels

# Application source.
# The assessment does not require implementation code, but this shows the expected
# production layout for the inference service.
COPY --chown=app:app app/ /app/app/

# Model is baked into the image.
# This must match lifecycle/model-registry.yaml and the rollback runbook.
COPY --chown=app:app models/retail-home-recommender/recsys-2026.06.07-001/ /opt/model/retail-home-recommender/

USER app

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=20s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8080/health', timeout=2).read()" || exit 1

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8080"]