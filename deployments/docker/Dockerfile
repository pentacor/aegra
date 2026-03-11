ARG PY_VERSION=3.12
FROM python:${PY_VERSION}-slim-bookworm AS base
ARG PY_VERSION

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PIP_NO_CACHE_DIR=off \
    PIP_DISABLE_PIP_VERSION_CHECK=on \
    PIP_DEFAULT_TIMEOUT=100 \
    AEGRA_CONFIG=/app/aegra.json

WORKDIR /app

# Create non-root user for runtime
RUN addgroup --system app && adduser --system --ingroup app app

# -----------------------------
# Builder stage
# -----------------------------
FROM base AS builder

# Retrieve the uv binary directly from the official image.
COPY --from=ghcr.io/astral-sh/uv:0.9.26 /uv /bin/uv

# Install system build dependencies required for compiling Python extensions.
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Copy workspace root (needed for uv export) and both packages
COPY pyproject.toml uv.lock ./
COPY libs/aegra-api/pyproject.toml libs/aegra-api/README.md ./libs/aegra-api/
COPY libs/aegra-cli/pyproject.toml libs/aegra-cli/README.md ./libs/aegra-cli/

# Install dependencies from lockfile
RUN uv export --frozen --package aegra-cli --no-emit-project --format=requirements-txt > requirements.txt && \
    uv pip install --system --compile-bytecode -r requirements.txt && \
    rm requirements.txt

# Copy source code for both packages
COPY libs/aegra-api/src/ ./libs/aegra-api/src/
COPY libs/aegra-api/alembic.ini ./libs/aegra-api/alembic.ini
COPY libs/aegra-api/alembic/ ./libs/aegra-api/alembic/
COPY libs/aegra-cli/src/ ./libs/aegra-cli/src/

# Install both packages (aegra-cli depends on aegra-api)
RUN cd libs/aegra-api && uv pip install --system --compile-bytecode --no-deps . && \
    cd ../aegra-cli && uv pip install --system --compile-bytecode --no-deps .

# -----------------------------
# Final, minimal runtime image
# -----------------------------
FROM base AS final

# Install only minimal runtime libs
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    libpq5 \
    && rm -rf /var/lib/apt/lists/*

# Copy installed Python packages from the builder stage.
COPY --from=builder /usr/local/lib/python${PY_VERSION}/site-packages/ /usr/local/lib/python${PY_VERSION}/site-packages/
COPY --from=builder /usr/local/bin/ /usr/local/bin/

# No aegra.json or graph code baked in — users mount or extend via FROM
EXPOSE 8000

# Run as non-root
USER app

# Use aegra CLI for proper config discovery, .env loading, and startup
CMD ["aegra", "serve"]
