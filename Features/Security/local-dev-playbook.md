# Local Dev Playbook -- Design Spec

> **Status:** SPECCED
> **Source:** Security overview.md (Q30-Q37, A6), TechStack spec
> **Resolves:** A6 (cost ceiling)

## Summary

Step-by-step guide for running the entire Airlock stack on a single developer machine with Docker Compose and zero mandatory SaaS dependencies. A developer following this document should go from a fresh clone to a running, seeded, TLS-enabled Airlock instance in under 15 minutes.

This spec answers security overview questions Q30 through Q37 and resolves ambiguity A6 by establishing the MVP cost ceiling at **$0/month** with optional API spend up to ~$20/month for higher-quality LLM output.

---

## 1. Prerequisites

### Operating System

| OS | Minimum Version | Notes |
|----|----------------|-------|
| macOS | 13.0 (Ventura) | Apple Silicon (M1+) or Intel. Rosetta 2 not required -- all images are multi-arch. |
| Ubuntu | 22.04 LTS | amd64. WSL2 on Windows 11 also works with Docker Desktop. |

### Required Software

| Tool | Version | Purpose | Install |
|------|---------|---------|---------|
| **Docker Desktop** | 4.25+ | Container orchestration | [docker.com/products/docker-desktop](https://docs.docker.com/get-docker/) |
| **Node.js** | 20 LTS (20.x) | Next.js 14 dev server with Turbopack hot reload | `nvm install 20` or [nodejs.org](https://nodejs.org/) |
| **Python** | 3.11+ | FastAPI backend development | `pyenv install 3.11` or system package manager |
| **Git** | 2.40+ | Source control | Pre-installed on macOS; `apt install git` on Ubuntu |
| **mkcert** | 1.4.4+ | Local TLS certificates for OAuth callbacks | `brew install mkcert` / `apt install mkcert` |
| **openssl** | 3.x | Secret generation | Pre-installed on both platforms |

**Podman alternative:** If using Podman instead of Docker Desktop, install `podman-compose` and alias `docker` to `podman`. All compose files are Podman-compatible. Ensure Podman machine has at least 16GB RAM allocated.

### Docker Desktop Configuration

Open Docker Desktop > Settings > Resources and set:

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| Memory | 8 GB | 16 GB (32 GB if using Ollama) |
| CPUs | 4 | 8 |
| Disk | 20 GB | 50 GB (Ollama models consume 4-30 GB) |

---

## 2. Hardware Requirements (Answers Q33)

### Per-Service Resource Allocation

| Service | RAM | CPU | Disk | Notes |
|---------|-----|-----|------|-------|
| PostgreSQL 16 | 512 MB | 0.5 core | 1 GB | With `pgvector` and `pgcrypto` extensions |
| Redis 7 | 256 MB | 0.25 core | 100 MB | BullMQ job queues + Pub/Sub + session cache |
| FastAPI (2 workers) | 512 MB | 1 core | 50 MB | Uvicorn with `--workers 2`, cipher middleware loaded |
| Next.js 14 (dev) | 1 GB | 1 core | 500 MB | Turbopack dev server with hot reload |
| LiteLLM Proxy | 256 MB | 0.25 core | 50 MB | Routes to Ollama or external API |
| Novu API | 384 MB | 0.25 core | 100 MB | Notification server |
| Novu Worker | 128 MB | 0.25 core | 100 MB | Notification job processor |
| **TOTAL (without Ollama)** | **~3 GB** | **3.5 cores** | **~2 GB** | Fits comfortably on 8 GB machine |
| Ollama (Llama 3.1 8B) | 8 GB | 4 cores | 5 GB | Quantized Q4_K_M |
| Ollama (Llama 3.1 70B) | 16 GB | 4 cores | 30 GB | Requires 32+ GB total system RAM |
| **TOTAL (with Ollama 8B)** | **~11 GB** | **7.5 cores** | **~7 GB** | Needs 16 GB machine |
| **TOTAL (with Ollama 70B)** | **~19 GB** | **7.5 cores** | **~32 GB** | Needs 32 GB machine |

### Minimum Viable Configurations

| Tier | RAM | CPU | Disk | LLM Strategy |
|------|-----|-----|------|---------------|
| **Minimal** | 8 GB | 4 cores | 20 GB | API only (Claude/GPT) or Otto disabled |
| **Standard** | 16 GB | 8 cores | 30 GB | Ollama 8B for basic tasks, API for complex |
| **Full** | 32 GB | 8+ cores | 50 GB | Ollama 70B for air-gapped, high-quality local LLM |

---

## 3. Docker Compose Service Topology (Answers Q30, Q31)

### Decision: Docker Compose Is Sufficient for MVP (Q30)

Docker Compose is the sole orchestration tool for MVP. Rationale:

- All services fit on a single machine (see hardware table above).
- Docker Compose v2 supports health checks, restart policies, resource limits, and profiles -- covering all MVP needs.
- Kubernetes (even k3s) adds operational complexity with no benefit for single-machine deployment.
- Secrets management is handled via `.env` file with documented safeguards (see Section 4).
- Rolling updates are unnecessary for local dev -- `docker compose up -d --build` is sufficient.

Kubernetes consideration deferred to enterprise phase when multi-node deployment, horizontal scaling, and managed secrets (via Sealed Secrets or External Secrets Operator) become requirements.

### Service Topology

```
                  +---------------------------+
                  |     Host Machine          |
                  |                           |
    Browser ----->| :3000 (web)               |
                  | :8000 (api)               |
                  +-----|--------|------------+
                        |        |
         +--------------+--------+--------------+
         |         airlock-internal network      |
         |                                       |
    +----+----+  +-------+  +--------+  +------+ |
    |   web   |  |  api  |  | litellm|  | novu | |
    | Next.js |  |FastAPI|  | proxy  |  | api  | |
    +---------+  +---+---+  +---+----+  +--+---+ |
                     |          |           |     |
              +------+------+  |     +-----+---+ |
              |  postgres   |  |     |  novu   | |
              | +pgvector   |  |     |  worker | |
              +-------------+  |     +---------+ |
              +-------------+  |                  |
              |    redis    |  |                  |
              +-------------+  |                  |
         +---------------------+------------------+
         |        airlock-llm network             |
         |  (isolated: LLM traffic only)          |
         |                                        |
         |  +--------+     +--------+             |
         |  | litellm| --- | ollama |             |
         |  +--------+     +--------+             |
         +----------------------------------------+
```

### Full `docker-compose.yml` Structure

```yaml
version: "3.9"

# ─── Networks ────────────────────────────────────────────────────────
networks:
  airlock-internal:
    driver: bridge
    # All application services communicate here
  airlock-llm:
    driver: bridge
    internal: true
    # Isolated network: only LiteLLM and Ollama
    # No direct internet access from Ollama

# ─── Volumes ─────────────────────────────────────────────────────────
volumes:
  pg-data:          # PostgreSQL persistent data
  redis-data:       # Redis AOF persistence
  ollama-models:    # Downloaded LLM model weights
  novu-data:        # Novu state
  file-storage:     # Encrypted file blobs (local S3 substitute)

# ─── Services ────────────────────────────────────────────────────────
services:

  # ── PostgreSQL 16 with pgvector + pgcrypto ──────────────────────
  postgres:
    image: pgvector/pgvector:pg16
    container_name: airlock-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: airlock
      POSTGRES_USER: airlock
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    ports:
      - "127.0.0.1:5432:5432"          # Host-only, not exposed to LAN
    volumes:
      - pg-data:/var/lib/postgresql/data
      - ./scripts/init-db.sql:/docker-entrypoint-initdb.d/01-init.sql:ro
    networks:
      - airlock-internal
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "0.5"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U airlock -d airlock"]
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 10s

  # ── Redis 7 ─────────────────────────────────────────────────────
  redis:
    image: redis:7-alpine
    container_name: airlock-redis
    restart: unless-stopped
    command: >
      redis-server
      --requirepass ${REDIS_PASSWORD}
      --appendonly yes
      --maxmemory 256mb
      --maxmemory-policy allkeys-lru
    ports:
      - "127.0.0.1:6379:6379"          # Host-only
    volumes:
      - redis-data:/data
    networks:
      - airlock-internal
    deploy:
      resources:
        limits:
          memory: 256M
          cpus: "0.25"
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 5s

  # ── FastAPI Backend ─────────────────────────────────────────────
  api:
    build:
      context: ./server
      dockerfile: Dockerfile
    container_name: airlock-api
    restart: unless-stopped
    command: >
      uvicorn main:app
      --host 0.0.0.0
      --port 8000
      --workers 2
      --ssl-keyfile /certs/localhost-key.pem
      --ssl-certfile /certs/localhost.pem
    environment:
      DATABASE_URL: postgresql+asyncpg://airlock:${POSTGRES_PASSWORD}@postgres:5432/airlock
      REDIS_URL: redis://:${REDIS_PASSWORD}@redis:6379/0
      AIRLOCK_MASTER_KEY: ${AIRLOCK_MASTER_KEY}
      JWT_SECRET: ${JWT_SECRET}
      LITELLM_BASE_URL: http://litellm:4000
      NOVU_API_URL: http://novu-api:3000
      NOVU_API_KEY: ${NOVU_API_KEY}
      GOOGLE_CLIENT_ID: ${GOOGLE_CLIENT_ID}
      GOOGLE_CLIENT_SECRET: ${GOOGLE_CLIENT_SECRET}
      AIRLOCK_DEV_MODE: ${AIRLOCK_DEV_MODE:-false}
      FILE_STORAGE_ROOT: /data/files
    ports:
      - "127.0.0.1:8000:8000"          # Host-only
    volumes:
      - ./server:/app:cached                # Hot reload in dev
      - ./certs:/certs:ro                   # TLS certificates
      - file-storage:/data/files            # Encrypted file storage
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - airlock-internal
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "1.0"
    healthcheck:
      test: ["CMD", "curl", "-fsk", "https://localhost:8000/health"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 15s

  # ── Next.js 14 Frontend ─────────────────────────────────────────
  web:
    build:
      context: ./web
      dockerfile: Dockerfile.dev
    container_name: airlock-web
    restart: unless-stopped
    command: npx next dev --turbo --port 3000
    environment:
      NEXT_PUBLIC_API_URL: https://localhost:8000
      NEXT_PUBLIC_WS_URL: wss://localhost:8000/ws
      NODE_TLS_REJECT_UNAUTHORIZED: "0"   # Accept mkcert certs in dev
    ports:
      - "3000:3000"                         # Exposed to host for browser
    volumes:
      - ./web:/app:cached                   # Hot reload in dev
      - /app/node_modules                   # Prevent host node_modules override
      - ./certs:/certs:ro                   # TLS certificates
    depends_on:
      api:
        condition: service_healthy
    networks:
      - airlock-internal
    deploy:
      resources:
        limits:
          memory: 1G
          cpus: "1.0"
    healthcheck:
      test: ["CMD", "curl", "-fsk", "https://localhost:3000"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 20s

  # ── LiteLLM Proxy ──────────────────────────────────────────────
  litellm:
    image: ghcr.io/berriai/litellm:main-latest
    container_name: airlock-litellm
    restart: unless-stopped
    environment:
      LITELLM_MASTER_KEY: ${LITELLM_MASTER_KEY:-sk-litellm-dev}
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY:-}
      OPENAI_API_KEY: ${OPENAI_API_KEY:-}
    volumes:
      - ./config/litellm.yaml:/app/config.yaml:ro
    command: ["--config", "/app/config.yaml", "--port", "4000"]
    networks:
      - airlock-internal
      - airlock-llm                         # Bridge to Ollama
    deploy:
      resources:
        limits:
          memory: 256M
          cpus: "0.25"
    healthcheck:
      test: ["CMD", "curl", "-fs", "http://localhost:4000/health"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s

  # ── Novu Notification Server ────────────────────────────────────
  novu-api:
    image: ghcr.io/novuhq/novu/api:latest
    container_name: airlock-novu-api
    restart: unless-stopped
    environment:
      NODE_ENV: local
      MONGO_URL: mongodb://novu-mongo:27017/novu
      REDIS_HOST: redis
      REDIS_PORT: 6379
      REDIS_PASSWORD: ${REDIS_PASSWORD}
      API_ROOT_URL: http://localhost:3001
    ports:
      - "127.0.0.1:3001:3000"              # Novu API on 3001 to avoid clash
    depends_on:
      redis:
        condition: service_healthy
    networks:
      - airlock-internal
    deploy:
      resources:
        limits:
          memory: 384M
          cpus: "0.25"
    healthcheck:
      test: ["CMD", "curl", "-fs", "http://localhost:3000/v1/health-check"]
      interval: 15s
      timeout: 5s
      retries: 5
      start_period: 20s

  novu-worker:
    image: ghcr.io/novuhq/novu/worker:latest
    container_name: airlock-novu-worker
    restart: unless-stopped
    environment:
      NODE_ENV: local
      MONGO_URL: mongodb://novu-mongo:27017/novu
      REDIS_HOST: redis
      REDIS_PORT: 6379
      REDIS_PASSWORD: ${REDIS_PASSWORD}
    depends_on:
      - novu-api
    networks:
      - airlock-internal
    deploy:
      resources:
        limits:
          memory: 128M
          cpus: "0.25"

  # ── MongoDB for Novu (Novu requires MongoDB) ───────────────────
  novu-mongo:
    image: mongo:7
    container_name: airlock-novu-mongo
    restart: unless-stopped
    volumes:
      - novu-data:/data/db
    networks:
      - airlock-internal
    deploy:
      resources:
        limits:
          memory: 256M
          cpus: "0.25"
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s

  # ── Ollama (optional, local LLM) ───────────────────────────────
  ollama:
    image: ollama/ollama:latest
    container_name: airlock-ollama
    restart: unless-stopped
    profiles:
      - local-llm                           # Only starts with --profile local-llm
    volumes:
      - ollama-models:/root/.ollama
    networks:
      - airlock-llm                         # Isolated: only LiteLLM can reach it
    deploy:
      resources:
        limits:
          memory: 16G                       # Adjust per model size
          cpus: "4.0"
    healthcheck:
      test: ["CMD", "curl", "-fs", "http://localhost:11434/api/version"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 30s
```

### Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Only ports 3000, 8000 exposed to all interfaces** | Web and API are the only user-facing services. All others bind to `127.0.0.1` or use container networking. |
| **Ollama on `airlock-llm` network with `internal: true`** | Prevents Ollama from making outbound internet requests. LiteLLM bridges the two networks. |
| **Ollama behind `profiles: [local-llm]`** | Does not start by default. Activated with `docker compose --profile local-llm up`. Saves 8-16 GB RAM when using API. |
| **MongoDB added for Novu** | Novu requires MongoDB. This is isolated to Novu's use and carries no application data. |
| **`unless-stopped` restart policy** | Services restart on crash but not after explicit `docker compose stop`. Suitable for dev. |
| **Host volume mounts for `server/` and `web/`** | Enables hot reload without container rebuilds. `:cached` flag for macOS performance. |

---

## 4. Secret Generation and Management (Answers Q32)

### Decision: `.env` File for MVP

For single-developer local development, a `.env` file is the pragmatic choice. The alternatives and why they were rejected:

| Option | Verdict | Reason |
|--------|---------|--------|
| `.env` file | **Selected for MVP** | Simple, Docker Compose reads it natively, sufficient for local dev |
| Docker Swarm secrets | Rejected | Requires Swarm mode, adds orchestration complexity with no local-dev benefit |
| Infisical / Vault | Deferred to enterprise | Adds a service dependency and operational overhead beyond MVP needs |

### Security Boundaries for `.env`

1. `.env` MUST be listed in `.gitignore` -- this is enforced by a pre-commit hook.
2. `.env` is readable only by the current user (`chmod 600 .env`).
3. `.env.example` is committed with placeholder values so developers know what to fill in.
4. No secrets appear in `Dockerfile` or `docker-compose.yml` -- only `${VARIABLE}` references.
5. For CI/CD (post-MVP), secrets transition to GitHub Actions secrets or equivalent.

### `.env.example` Template

```bash
# ─── Airlock Local Dev Environment ────────────────────────────────
# Copy to .env and fill in values: cp .env.example .env
# Generate secrets: ./scripts/generate-secrets.sh

# ── Core Secrets (auto-generated by generate-secrets.sh) ──────────
AIRLOCK_MASTER_KEY=<run: openssl rand -base64 32>
JWT_SECRET=<run: openssl rand -base64 32>
POSTGRES_PASSWORD=<run: openssl rand -base64 16>
REDIS_PASSWORD=<run: openssl rand -base64 16>
LITELLM_MASTER_KEY=<run: openssl rand -base64 16>

# ── Novu ──────────────────────────────────────────────────────────
NOVU_API_KEY=<generated during first Novu boot>

# ── OAuth (required for production auth flow) ─────────────────────
# Create at: https://console.cloud.google.com/apis/credentials
GOOGLE_CLIENT_ID=your-client-id.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=your-client-secret

# ── Dev Mode (bypasses OAuth, uses seed users) ────────────────────
# WARNING: Never enable in production
AIRLOCK_DEV_MODE=true

# ── LLM API Keys (optional -- omit to use Ollama only) ───────────
ANTHROPIC_API_KEY=
OPENAI_API_KEY=

# ── Feature Flags ─────────────────────────────────────────────────
AIRLOCK_FEATURE_OTTO=true
AIRLOCK_FEATURE_NOTIFICATIONS=true
```

### Secret Generation Script

`scripts/generate-secrets.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

ENV_FILE=".env"

if [[ -f "$ENV_FILE" ]]; then
  echo "WARNING: .env already exists. Overwrite? (y/N)"
  read -r confirm
  [[ "$confirm" != "y" ]] && echo "Aborted." && exit 0
fi

cat > "$ENV_FILE" <<EOF
# ─── Auto-generated by generate-secrets.sh on $(date -u +%Y-%m-%dT%H:%M:%SZ) ───
AIRLOCK_MASTER_KEY=$(openssl rand -base64 32)
JWT_SECRET=$(openssl rand -base64 32)
POSTGRES_PASSWORD=$(openssl rand -base64 16)
REDIS_PASSWORD=$(openssl rand -base64 16)
LITELLM_MASTER_KEY=$(openssl rand -base64 16)

# ── Novu (placeholder -- update after first boot) ──
NOVU_API_KEY=placeholder-update-after-novu-init

# ── OAuth (fill in manually or set dev mode) ──
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=

# ── Dev Mode (bypasses OAuth) ──
AIRLOCK_DEV_MODE=true

# ── LLM API Keys (optional) ──
ANTHROPIC_API_KEY=
OPENAI_API_KEY=

# ── Feature Flags ──
AIRLOCK_FEATURE_OTTO=true
AIRLOCK_FEATURE_NOTIFICATIONS=true
EOF

chmod 600 "$ENV_FILE"
echo "Generated $ENV_FILE with fresh secrets."
echo "IMPORTANT: Do not commit this file. It is in .gitignore."
```

### `.gitignore` Entries (Security-Critical)

```gitignore
# Secrets -- NEVER commit
.env
.env.local
.env.*.local

# TLS certificates
certs/*.pem
certs/*.key

# Ollama model cache (large binary files)
ollama-models/
```

---

## 5. Quick Start Commands (Answers Q37)

### Zero to Running in 15 Minutes

```bash
# ─── Step 1: Clone and enter repo ────────────────────────────────
git clone git@github.com:<org>/airlock.git
cd airlock

# ─── Step 2: Generate secrets ────────────────────────────────────
chmod +x scripts/generate-secrets.sh
./scripts/generate-secrets.sh
# Creates .env with cryptographically random values

# ─── Step 3: Setup local TLS (one-time) ──────────────────────────
mkcert -install                            # Install local CA (prompts for sudo)
mkdir -p certs
mkcert -cert-file certs/localhost.pem \
       -key-file certs/localhost-key.pem \
       localhost 127.0.0.1 ::1
# Certificates valid for localhost, used by both Next.js and FastAPI

# ─── Step 4: Start all services ──────────────────────────────────
docker compose up -d
# Without Ollama. Add --profile local-llm for Ollama.

# ─── Step 5: Wait for healthy state ──────────────────────────────
echo "Waiting for services..."
sleep 10
docker compose ps
# Verify all services show "healthy" in STATUS column
# If any show "starting", wait another 10-15 seconds and re-check

# ─── Step 6: Run database migrations ─────────────────────────────
docker compose exec api python -m alembic upgrade head
# Creates all tables, enables pgvector + pgcrypto extensions,
# sets up RLS policies, creates append-only triggers

# ─── Step 7: Seed demo data ──────────────────────────────────────
docker compose exec api python -m scripts.seed_demo
# Populates workspace with synthetic data (see Section 7)

# ─── Step 8: Open Airlock ─────────────────────────────────────────
open https://localhost:3000
# On Linux: xdg-open https://localhost:3000
# Browser will warn about self-signed cert -- click "Advanced > Proceed"
# With AIRLOCK_DEV_MODE=true, login page shows seed user buttons
```

### Common Lifecycle Commands

```bash
# Stop everything (preserves data)
docker compose down

# Stop everything and destroy volumes (fresh start)
docker compose down -v

# Rebuild after code changes to Dockerfiles
docker compose up -d --build

# View logs for a specific service
docker compose logs -f api          # Follow FastAPI logs
docker compose logs -f web          # Follow Next.js logs
docker compose logs postgres        # One-shot PostgreSQL logs

# Shell into a running container
docker compose exec api bash        # FastAPI container
docker compose exec postgres psql -U airlock -d airlock  # Database shell

# Start with Ollama for local LLM
docker compose --profile local-llm up -d

# Check resource usage
docker compose stats
```

---

## 6. Local TLS with mkcert (Answers Q34)

### Decision: Local TLS Required for MVP Dev

Local HTTPS is required because:

1. **Google OAuth requires HTTPS** for redirect URIs (even on localhost in production configurations).
2. **WebSocket security** requires `wss://` -- browsers block mixed content (HTTPS page connecting to `ws://`).
3. **Cookie security** attributes (`Secure`, `SameSite=None`) require HTTPS context.
4. **Parity with production** -- dev environment should mirror production TLS behavior to catch issues early.

The dev mode bypass (`AIRLOCK_DEV_MODE=true`) eliminates the OAuth requirement for day-to-day development, but TLS is still needed for WebSocket and cookie handling.

### mkcert Installation

```bash
# macOS
brew install mkcert
brew install nss        # Required for Firefox support

# Ubuntu / Debian
sudo apt install libnss3-tools
curl -JLO "https://dl.filippo.io/mkcert/latest?for=linux/amd64"
chmod +x mkcert-v*-linux-amd64
sudo mv mkcert-v*-linux-amd64 /usr/local/bin/mkcert
```

### Certificate Generation

```bash
# Install the local CA into system trust store (one-time, requires sudo)
mkcert -install

# Generate certificate for localhost
mkdir -p certs
mkcert -cert-file certs/localhost.pem \
       -key-file certs/localhost-key.pem \
       localhost 127.0.0.1 ::1

# Verify
openssl x509 -in certs/localhost.pem -noout -dates
# Should show valid dates spanning ~2 years
```

### Next.js TLS Configuration

In `web/next.config.js`:

```javascript
const fs = require('fs');
const path = require('path');

/** @type {import('next').NextConfig} */
const nextConfig = {
  // TLS for dev server
  ...(process.env.NODE_ENV === 'development' && {
    server: {
      https: {
        key: fs.readFileSync(path.join(__dirname, '../certs/localhost-key.pem')),
        cert: fs.readFileSync(path.join(__dirname, '../certs/localhost.pem')),
      },
    },
  }),
};

module.exports = nextConfig;
```

### FastAPI TLS Configuration

Uvicorn reads the cert files directly via CLI flags (specified in `docker-compose.yml`):

```
uvicorn main:app --ssl-keyfile /certs/localhost-key.pem --ssl-certfile /certs/localhost.pem
```

This enables both HTTPS for the REST API and `wss://` for WebSocket connections on the same port.

### CORS Configuration

In FastAPI `main.py`:

```python
from fastapi.middleware.cors import CORSMiddleware

ALLOWED_ORIGINS = [
    "https://localhost:3000",       # Next.js dev server
    "https://127.0.0.1:3000",      # Alternative localhost
]

app.add_middleware(
    CORSMiddleware,
    allow_origins=ALLOWED_ORIGINS,
    allow_credentials=True,         # Required for cookie-based auth
    allow_methods=["*"],
    allow_headers=["*"],
    expose_headers=["X-Request-Id"],
)
```

### Google OAuth Redirect URI

When configuring Google Cloud Console credentials:

```
Authorized JavaScript origins:  https://localhost:3000
Authorized redirect URIs:       https://localhost:3000/api/auth/callback/google
                                https://localhost:8000/auth/google/callback
```

---

## 7. Database Seeding (Answers Q35)

### Design Principles

1. **All data is synthetic** -- no real PII, no real contract text. Names like "Acme Inc", "Jane Builder", "John Gatekeeper".
2. **All sensitive fields encrypted** with the dev `AIRLOCK_MASTER_KEY` -- the seed script exercises the cipher pipeline.
3. **Vault hierarchy is realistic** -- Parent > Division > Counterparty > Item structure mirrors production topology.
4. **Seed is idempotent** -- running it twice does not duplicate data (upsert on workspace ID).

### Seed Data Manifest

| Entity | Count | Details |
|--------|-------|---------|
| Workspaces | 1 | "Airlock Demo" (`ws_DEMO_000000000000000000000000`) |
| Users | 3 | Admin (`admin@airlock.dev`), Gatekeeper (`gatekeeper@airlock.dev`), Builder (`builder@airlock.dev`) |
| Roles | 3 | Owner, Gatekeeper, Builder -- with full permission matrices |
| Accounts (L1 Parents) | 5 | "Acme Inc", "Globex Corp", "Initech", "Umbrella Music", "Stark Entertainment" |
| Divisions (L2) | 10 | 2 per parent (e.g., "Acme Records", "Acme Publishing") |
| Counterparties (L3) | 20 | Distributed across divisions |
| Items (L4 Vaults) | 30 | Contract Vaults with sample extraction data |
| Contacts | 15 | Distributed across accounts, with roles and communication preferences |
| Deals | 8 | Across pipeline stages: Discover (2), Build (2), Review (2), Ship (2) |
| Communications | 30 | Mix of types: iMessage (10), email (12), call logs (8) |
| Activity Events | 50 | Extraction results, patch proposals, approvals, comments |
| Tasks | 10 | Across statuses: Open (4), In Progress (3), Blocked (1), Done (2) |
| PDF Contracts | 2 | Synthetic distribution and publishing agreements (generated, no real legal text) |
| Notifications | 8 | Mix of read/unread, different priority levels |

### Dev Mode Auth Bypass

When `AIRLOCK_DEV_MODE=true`, the login page renders three "Quick Login" buttons:

```
┌─────────────────────────────────────┐
│           Airlock Login             │
│                                     │
│   [  Sign in with Google  ]        │  <- Standard OAuth (requires creds)
│                                     │
│   ── Dev Mode ──────────────────   │
│   [ Admin ]  [ Gatekeeper ]  [ Builder ]  <- Instant login as seed user
│                                     │
└─────────────────────────────────────┘
```

These buttons call a dev-only endpoint (`POST /auth/dev-login`) that:

1. Verifies `AIRLOCK_DEV_MODE=true` in the environment.
2. Issues a standard JWT for the requested seed user.
3. Sets the same `HttpOnly`, `Secure`, `SameSite=Lax` cookie as production OAuth.
4. **This endpoint does not exist when `AIRLOCK_DEV_MODE` is `false` or unset** -- the route is conditionally registered.

### Seed Script Structure

```
scripts/seed_demo.py
├── create_workspace()           # Workspace record + master DEK
├── create_users()               # 3 seed users with role assignments
├── create_vault_hierarchy()     # L1-L4 hierarchy (5 parents, 10 divisions, 20 CPs, 30 items)
├── create_contacts()            # 15 contacts across accounts
├── create_deals()               # 8 deals across chambers
├── create_communications()      # 30 comms (iMessage, email, calls)
├── create_activity_events()     # 50 events in vault feeds
├── create_tasks()               # 10 tasks across statuses
├── upload_sample_contracts()    # 2 synthetic PDFs, encrypted at rest
└── create_notifications()       # 8 sample notifications
```

Each function:
- Uses the cipher middleware to encrypt sensitive fields before insert.
- Generates deterministic UUIDs (seeded from workspace ID) for idempotency.
- Logs progress to stdout for verification.

---

## 8. Ollama Setup for Air-Gapped Dev (Answers Q36)

### Starting with Local LLM

```bash
# Start full stack including Ollama
docker compose --profile local-llm up -d

# Wait for Ollama to become healthy
docker compose logs -f ollama
# Look for: "Listening on [::]:11434"

# Pull a model (one-time download)
docker compose exec ollama ollama pull llama3.1:8b
# Downloads ~4.7 GB (Q4_K_M quantization)

# Verify model is available
docker compose exec ollama ollama list
```

### LiteLLM Configuration for Ollama

`config/litellm.yaml`:

```yaml
model_list:
  # ── Primary: External API (used when API keys are set) ──────────
  - model_name: claude-sonnet
    litellm_params:
      model: anthropic/claude-sonnet-4-20250514
      api_key: os.environ/ANTHROPIC_API_KEY
    model_info:
      description: "High-quality, used for complex extraction and generation"

  - model_name: gpt-4o
    litellm_params:
      model: openai/gpt-4o
      api_key: os.environ/OPENAI_API_KEY
    model_info:
      description: "Alternative high-quality model"

  # ── Fallback: Ollama (used when API keys are empty) ─────────────
  - model_name: local-llama
    litellm_params:
      model: ollama/llama3.1:8b
      api_base: http://ollama:11434
    model_info:
      description: "Local Llama 3.1 8B -- free, good for basic tasks"

  - model_name: local-mistral
    litellm_params:
      model: ollama/mistral:7b
      api_base: http://ollama:11434
    model_info:
      description: "Local Mistral 7B -- free, good for extraction"

# ── Routing: try external first, fall back to local ───────────────
router_settings:
  routing_strategy: "simple-shuffle"  # Load balance across available models
  num_retries: 2
  fallbacks:
    - claude-sonnet: [local-llama]
    - gpt-4o: [local-llama]
```

### Cost Comparison (Answers Q36, Resolves A6)

**A6 Resolution: MVP cost ceiling is $0/month. Optional API spend capped at $20/month.**

| Option | Monthly Cost | Quality | Latency | Use Case |
|--------|-------------|---------|---------|----------|
| Ollama Llama 3.1 8B | $0 | Good for basic tasks, summaries, chat | 2-5 s/response | Day-to-day dev, air-gapped environments |
| Ollama Mistral 7B | $0 | Good for extraction, structured output | 1-3 s/response | Field extraction testing |
| Ollama Llama 3.1 70B | $0 | Very good, near-API quality | 5-15 s/response | Quality testing (requires 32 GB RAM) |
| Claude Sonnet API | ~$5-20/month | Excellent | 1-2 s/response | Complex extraction, contract generation |
| GPT-4o API | ~$5-15/month | Excellent | 1-2 s/response | Alternative to Claude |
| No LLM (Otto disabled) | $0 | N/A | N/A | Frontend/CRUD development only |

**Budget guideline for API usage:**
- Solo developer: $0-10/month (mostly Ollama, API for edge cases).
- Team of 3: $10-20/month (shared API key, budget alerts via LiteLLM).
- LiteLLM tracks token usage per model. Set `max_budget: 20.0` in `litellm.yaml` to enforce ceiling.

### Air-Gapped Mode

For environments with no internet access (e.g., secure facilities):

1. Pre-download Ollama models on a connected machine:
   ```bash
   ollama pull llama3.1:8b
   ollama pull mistral:7b
   ```
2. Export the model directory (`~/.ollama/models/`).
3. Copy to the air-gapped machine and mount into the `ollama-models` Docker volume.
4. Remove all `ANTHROPIC_API_KEY` and `OPENAI_API_KEY` values from `.env`.
5. LiteLLM routes all requests to Ollama automatically.

---

## 9. External Services and Local Fallbacks (Answers Q37)

### Decision: No Mandatory SaaS

**Q37 Answer:** "No mandatory SaaS" means every external service has a local fallback that provides equivalent functionality (possibly at lower quality). The ONE exception is Google OAuth, which requires Google's servers -- but dev mode bypasses this entirely.

| Service | External Option | Local Fallback | Quality Delta |
|---------|----------------|----------------|---------------|
| **LLM** | Claude API, GPT-4o API | Ollama (Llama 3.1, Mistral) | Local models handle 80% of dev tasks; API needed for quality validation |
| **Authentication** | Google OAuth | Dev mode bypass (`AIRLOCK_DEV_MODE=true`) | Functionally equivalent for development; OAuth tested manually |
| **Email Delivery** | SendGrid, Postmark | Console logger (`AIRLOCK_EMAIL_BACKEND=console`) | Emails printed to stdout; sufficient for verifying content and triggers |
| **Notifications** | Novu Cloud | Novu self-hosted (in Docker Compose) | Full parity -- self-hosted Novu is identical to cloud |
| **File Storage** | AWS S3 | Local filesystem volume (`file-storage`) | Full parity for dev; S3 compatibility via MinIO if needed later |
| **Search** | Algolia, Typesense Cloud | PostgreSQL `tsvector` + `pg_trgm` | Adequate for dev data volumes; dedicated search deferred to enterprise |
| **Vector Search** | Pinecone, Weaviate Cloud | `pgvector` (in PostgreSQL) | Full parity for MVP scale; dedicated vector DB deferred to enterprise |
| **Monitoring** | Datadog, New Relic | Docker Compose logs + `docker stats` | Sufficient for local dev; APM deferred to production deployment |

### Service Availability Matrix

| Internet Status | Services Running | Otto (AI) Status | Auth Method |
|----------------|-----------------|------------------|-------------|
| **Connected, API keys set** | All | Full quality (Claude/GPT via LiteLLM) | Google OAuth or dev bypass |
| **Connected, no API keys** | All except external LLM | Ollama fallback | Google OAuth or dev bypass |
| **Air-gapped (no internet)** | All local services | Ollama only | Dev bypass only |
| **Minimal (no Docker)** | Next.js + FastAPI on host | Disabled | Dev bypass only |

---

## 10. Troubleshooting

### Port Conflicts

| Port | Service | Resolution |
|------|---------|------------|
| 3000 | Next.js (web) | Kill existing process: `lsof -ti:3000 \| xargs kill` or change port in `docker-compose.yml` |
| 3001 | Novu API | `lsof -ti:3001 \| xargs kill` |
| 5432 | PostgreSQL | Stop local PostgreSQL: `brew services stop postgresql` (macOS) or `sudo systemctl stop postgresql` (Ubuntu) |
| 6379 | Redis | Stop local Redis: `brew services stop redis` or `sudo systemctl stop redis` |
| 8000 | FastAPI (api) | `lsof -ti:8000 \| xargs kill` |

### Ollama Out of Memory

**Symptom:** Ollama container exits with OOM or becomes unresponsive during inference.

**Solutions:**
1. Use a smaller model: `ollama pull mistral:7b` (5 GB) instead of `llama3.1:70b` (40 GB).
2. Increase Docker Desktop memory allocation (Settings > Resources).
3. Stop Ollama and use API: remove `--profile local-llm` from your `docker compose up` command.
4. Set `OLLAMA_NUM_PARALLEL=1` in Ollama environment to prevent concurrent inference.

### OAuth Redirect Errors

**Symptom:** Google OAuth returns "redirect_uri_mismatch" error.

**Checklist:**
1. Verify mkcert is installed and CA is trusted: `mkcert -install`.
2. Confirm certs exist: `ls certs/localhost.pem certs/localhost-key.pem`.
3. Check Google Cloud Console redirect URIs match exactly: `https://localhost:3000/api/auth/callback/google`.
4. Ensure `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET` are set in `.env`.
5. Fallback: use `AIRLOCK_DEV_MODE=true` to bypass OAuth entirely during development.

### Database Migration Failures

**Symptom:** `alembic upgrade head` fails with extension or permission errors.

**Solutions:**
1. Check PostgreSQL is healthy: `docker compose ps postgres`.
2. Verify `pgvector` extension: `docker compose exec postgres psql -U airlock -d airlock -c "CREATE EXTENSION IF NOT EXISTS vector;"`.
3. Verify `pgcrypto` extension: `docker compose exec postgres psql -U airlock -d airlock -c "CREATE EXTENSION IF NOT EXISTS pgcrypto;"`.
4. Check migration logs: `docker compose exec api python -m alembic history --verbose`.
5. Nuclear option (destroys data): `docker compose down -v && docker compose up -d` to reset all volumes.

### WebSocket Connection Refused

**Symptom:** Browser console shows `WebSocket connection to 'wss://localhost:8000/ws' failed`.

**Checklist:**
1. Verify FastAPI is running with TLS: `curl -sk https://localhost:8000/health`.
2. Check Redis is running (WebSocket uses Redis Pub/Sub): `docker compose ps redis`.
3. Confirm CORS allows the WebSocket origin: check `ALLOWED_ORIGINS` in FastAPI config.
4. Check browser is not blocking mixed content (HTTPS page must connect to `wss://`, not `ws://`).
5. Inspect FastAPI logs: `docker compose logs -f api | grep -i websocket`.

### Container Build Failures

**Symptom:** `docker compose up -d --build` fails during image build.

**Solutions:**
1. Clear Docker build cache: `docker builder prune -f`.
2. Pull fresh base images: `docker compose pull`.
3. Check disk space: `docker system df` -- prune if needed: `docker system prune -a`.
4. On macOS with Apple Silicon, ensure `platform: linux/amd64` is not set (or use `linux/arm64`).

### Full Reset Procedure

When everything is broken and you need a clean start:

```bash
# Stop and remove all containers, networks, volumes
docker compose --profile local-llm down -v

# Remove any orphaned containers
docker compose rm -f

# Prune Docker system (optional, frees disk)
docker system prune -f

# Re-generate secrets
./scripts/generate-secrets.sh

# Regenerate TLS certs (if expired or corrupted)
rm -rf certs && mkdir certs
mkcert -cert-file certs/localhost.pem \
       -key-file certs/localhost-key.pem \
       localhost 127.0.0.1 ::1

# Fresh start
docker compose up -d
sleep 15
docker compose exec api python -m alembic upgrade head
docker compose exec api python -m scripts.seed_demo
open https://localhost:3000
```

---

## Related Specs

- [overview.md](./overview.md) -- Security architecture documentation plan (parent spec, Q30-Q37 defined here)
- [data-cipher-model.md](./data-cipher-model.md) -- Encryption at rest spec (cipher pipeline used by seed script)
- [../TechStack/overview.md](../TechStack/overview.md) -- Locked library choices (Docker, PostgreSQL, Redis, Next.js, FastAPI)
- [../AIAgent/overview.md](../AIAgent/overview.md) -- Otto AI agent spec (LiteLLM routing, Ollama integration)
- [../Notifications/overview.md](../Notifications/overview.md) -- Novu notification system
- [../RealTime/overview.md](../RealTime/overview.md) -- WebSocket + Redis Pub/Sub architecture
- [../Roles/overview.md](../Roles/overview.md) -- RBAC system (seed user roles and permissions)
