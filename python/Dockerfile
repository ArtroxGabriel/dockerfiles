# syntax=docker/dockerfile:1

# ==============================================================================
# Python Multi-Stage Dockerfile Template (Powered by Astral uv)
#
# Highlights:
# 1. Multi-Stage Build: uv builds the virtualenv; final image contains no uv.
# 2. BuildKit Cache & Bind Mounts: Caches uv package downloads across builds.
# 3. Lean Runtime: Based on official python:*-slim matching the builder interpreter.
# 4. Security: Runs as an unprivileged non-root user (UID 10001).
# 5. Production Best Practices: PYTHONUNBUFFERED=1, bytecode pre-compilation.
# ==============================================================================

# ------------------------------------------------------------------------------
# Global Build Arguments
# IMPORTANT: The Python version and Debian suite in builder and runtime must match
# so that the system interpreter and virtualenv remain compatible.
# ------------------------------------------------------------------------------
ARG PYTHON_VERSION=3.12
ARG DEBIAN_SUITE=trixie

# ==============================================================================
# Stage 1: Build virtual environment with uv
# ==============================================================================
FROM ghcr.io/astral-sh/uv:python${PYTHON_VERSION}-${DEBIAN_SUITE}-slim AS builder

# Compile Python bytecode for faster cold starts
ENV UV_COMPILE_BYTECODE=1 \
    UV_LINK_MODE=copy \
    UV_NO_DEV=1 \
    UV_PYTHON_DOWNLOADS=0

WORKDIR /app

# Step 1: Install dependencies only (cached until pyproject.toml or uv.lock changes)
RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --locked --no-install-project

# Step 2: Copy application code and install project
COPY . /app
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --locked


# ==============================================================================
# Stage 2: Lean Production Runtime (without uv)
# ==============================================================================
FROM python:${PYTHON_VERSION}-slim-${DEBIAN_SUITE} AS runtime

ARG APP_USER=appuser
ARG APP_UID=10001
ARG PORT=8000

WORKDIR /app

# Create dedicated unprivileged system user without login shell or home directory
RUN groupadd --system --gid ${APP_UID} ${APP_USER} && \
    useradd --system --uid ${APP_UID} --gid ${APP_USER} --no-create-home --shell /usr/sbin/nologin ${APP_USER}

# Copy the built application and virtualenv from the builder
COPY --from=builder --chown=${APP_USER}:${APP_USER} /app /app

# Put virtualenv binaries at the front of PATH
ENV PATH="/app/.venv/bin:$PATH" \
    PYTHONUNBUFFERED=1 \
    PORT=${PORT}

USER ${APP_UID}:${APP_UID}
EXPOSE ${PORT}

STOPSIGNAL SIGTERM

# Default: FastAPI application
CMD ["fastapi", "run", "--host", "0.0.0.0", "--port", "8000"]

# ------------------------------------------------------------------------------
# ALTERNATIVE ENTRYPOINTS:
#
# Uvicorn:
# CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
#
# Gunicorn with Uvicorn workers:
# CMD ["gunicorn", "-w", "4", "-k", "uvicorn.workers.UvicornWorker", "app.main:app", "-b", "0.0.0.0:8000"]
#
# Standard Python script / worker:
# CMD ["python", "-m", "app.main"]
# ------------------------------------------------------------------------------
