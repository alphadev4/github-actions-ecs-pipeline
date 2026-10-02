# syntax=docker/dockerfile:1
FROM python:3.12-slim AS builder

# Pinned uv — bump deliberately, in lockstep with the team's uv version.
COPY --from=ghcr.io/astral-sh/uv:0.11.13 /uv /uvx /bin/

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    # Compile .pyc at build time for faster cold start.
    UV_COMPILE_BYTECODE=1 \
    # Copy (not hardlink) from the cache mount across filesystems.
    UV_LINK_MODE=copy \
    # Use the base image's Python 3.12; never download a managed interpreter,
    # so the venv references /usr/local/bin/python which also exists in runtime.
    UV_PYTHON_DOWNLOADS=0 \
    UV_PYTHON=/usr/local/bin/python3.12 \
    # Fixed venv path, shared with the runtime stage.
    UV_PROJECT_ENVIRONMENT=/app/.venv

RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && apt-get install -y --no-install-recommends \
        gcc \
        g++ \
        libpq-dev \
        libgdal-dev \
        libgeos-dev \
        libproj-dev \
        tesseract-ocr \
        poppler-utils

WORKDIR /app

# Only the dependency manifests — keeps this layer cached unless deps change.
COPY pyproject.toml uv.lock ./

# --frozen: fail if uv.lock is out of date with pyproject.toml (no silent drift).
# --no-dev: production install only.
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-dev

# ─── Runtime ─────────────────────────────────────────────────────────────────
FROM python:3.12-slim AS runtime

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    # Put the venv on PATH so `python`/`daphne`/`celery` resolve to it —
    # existing entrypoint.sh and compose commands work unchanged.
    PATH="/app/.venv/bin:$PATH"

RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && apt-get install -y --no-install-recommends \
        netcat-openbsd \
        supervisor \
        libpq5 \
        libgdal36 \
        libgeos-c1t64 \
        libproj25 \
        tesseract-ocr \
        poppler-utils

WORKDIR /app

# The built virtualenv (deps only) from the builder stage.
COPY --from=builder /app/.venv /app/.venv

# Application source.
COPY . .

# RUN python manage.py makemigrations

# RUN python manage.py migrate

EXPOSE 8000

CMD ["daphne", "-b", "0.0.0.0", "-p", "8000", "core.asgi:application"]