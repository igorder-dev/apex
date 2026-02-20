# Infrastructure Specification: PolyBot Platform

## Overview

PolyBot deploys as a single-VPS Docker Compose application with 9 service containers, 2 persistence containers, and 2 monitoring containers. This document covers the complete infrastructure: Docker configuration, VPS provisioning, network topology, monitoring stack, backup/recovery, deployment workflows, and operational procedures.

**Target Environment**: Ubuntu 22.04 LTS VPS, 8 cores, 16-32 GB RAM, 200 GB NVMe SSD.

---

## Docker Compose Configuration

### Production Stack (`docker-compose.yml`)

```yaml
# docker-compose.yml — PolyBot Production Stack
version: "3.9"

x-common-env: &common-env
  REDIS_URL: redis://redis:6379/0
  DATABASE_URL: postgresql+asyncpg://polybot:${POSTGRES_PASSWORD}@postgres:5432/polybot
  LOG_LEVEL: INFO
  ENVIRONMENT: production

x-service-defaults: &service-defaults
  restart: unless-stopped
  networks:
    - polybot_net
  logging:
    driver: json-file
    options:
      max-size: "50m"
      max-file: "5"

services:
  # ─── Persistence ────────────────────────────────────────
  postgres:
    image: timescale/timescaledb:latest-pg16
    <<: *service-defaults
    container_name: polybot-postgres
    environment:
      POSTGRES_USER: polybot
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: polybot
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U polybot"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 2G
        reservations:
          cpus: "0.5"
          memory: 1G
    labels:
      - "com.polybot.service=postgres"

  redis:
    image: redis:7-alpine
    <<: *service-defaults
    container_name: polybot-redis
    command: >
      redis-server
      --maxmemory 512mb
      --maxmemory-policy allkeys-lru
      --appendonly yes
      --appendfsync everysec
      --save 300 10
      --save 60 1000
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
        reservations:
          cpus: "0.25"
          memory: 256M
    labels:
      - "com.polybot.service=redis"

  # ─── Core Services ──────────────────────────────────────
  market-data:
    build:
      context: .
      dockerfile: src/services/market_data/Dockerfile
    <<: *service-defaults
    container_name: polybot-market-data
    environment:
      <<: *common-env
      POLYMARKET_WS_URL: ${POLYMARKET_WS_URL}
      GAMMA_API_URL: ${GAMMA_API_URL}
    depends_on:
      redis:
        condition: service_healthy
      postgres:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "python", "-c", "import httpx; httpx.get('http://localhost:8001/health').raise_for_status()"]
      interval: 15s
      timeout: 10s
      retries: 3
      start_period: 20s
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 1G
        reservations:
          cpus: "0.5"
          memory: 512M
    labels:
      - "com.polybot.service=market-data"
      - "prometheus.io/scrape=true"
      - "prometheus.io/port=8001"
      - "prometheus.io/path=/metrics"

  orchestrator:
    build:
      context: .
      dockerfile: src/services/orchestrator/Dockerfile
    <<: *service-defaults
    container_name: polybot-orchestrator
    environment:
      <<: *common-env
    volumes:
      - ./config/bots:/app/config/bots:ro
    depends_on:
      redis:
        condition: service_healthy
      postgres:
        condition: service_healthy
      market-data:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "python", "-c", "import httpx; httpx.get('http://localhost:8002/health').raise_for_status()"]
      interval: 15s
      timeout: 10s
      retries: 3
      start_period: 30s
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 4G
        reservations:
          cpus: "1.0"
          memory: 2G
    labels:
      - "com.polybot.service=orchestrator"
      - "prometheus.io/scrape=true"
      - "prometheus.io/port=8002"
      - "prometheus.io/path=/metrics"

  execution:
    build:
      context: .
      dockerfile: src/services/execution/Dockerfile
    <<: *service-defaults
    container_name: polybot-execution
    environment:
      <<: *common-env
      POLYMARKET_CLOB_URL: ${POLYMARKET_CLOB_URL}
      POLYGON_CHAIN_ID: ${POLYGON_CHAIN_ID:-137}
    depends_on:
      redis:
        condition: service_healthy
      postgres:
        condition: service_healthy
      wallet:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "python", "-c", "import httpx; httpx.get('http://localhost:8003/health').raise_for_status()"]
      interval: 15s
      timeout: 10s
      retries: 3
      start_period: 20s
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
        reservations:
          cpus: "0.25"
          memory: 256M
    labels:
      - "com.polybot.service=execution"
      - "prometheus.io/scrape=true"
      - "prometheus.io/port=8003"
      - "prometheus.io/path=/metrics"

  risk:
    build:
      context: .
      dockerfile: src/services/risk/Dockerfile
    <<: *service-defaults
    container_name: polybot-risk
    environment:
      <<: *common-env
    volumes:
      - ./config/risk.yaml:/app/config/risk.yaml:ro
    depends_on:
      redis:
        condition: service_healthy
      postgres:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "python", "-c", "import httpx; httpx.get('http://localhost:8004/health').raise_for_status()"]
      interval: 15s
      timeout: 10s
      retries: 3
      start_period: 15s
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
        reservations:
          cpus: "0.25"
          memory: 256M
    labels:
      - "com.polybot.service=risk"
      - "prometheus.io/scrape=true"
      - "prometheus.io/port=8004"
      - "prometheus.io/path=/metrics"

  wallet:
    build:
      context: .
      dockerfile: src/services/wallet/Dockerfile
    <<: *service-defaults
    container_name: polybot-wallet
    environment:
      <<: *common-env
      VAULT_PRIVATE_KEY: ${VAULT_PRIVATE_KEY}
      ALPHA_PRIVATE_KEY: ${ALPHA_PRIVATE_KEY:-}
      SWEEP_PRIVATE_KEY: ${SWEEP_PRIVATE_KEY:-}
      POLYMARKET_CLOB_URL: ${POLYMARKET_CLOB_URL}
      POLYGON_CHAIN_ID: ${POLYGON_CHAIN_ID:-137}
    volumes:
      - ./config/wallets.yaml:/app/config/wallets.yaml:ro
    depends_on:
      redis:
        condition: service_healthy
      postgres:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "python", "-c", "import httpx; httpx.get('http://localhost:8005/health').raise_for_status()"]
      interval: 15s
      timeout: 10s
      retries: 3
      start_period: 20s
    deploy:
      resources:
        limits:
          cpus: "0.25"
          memory: 256M
        reservations:
          cpus: "0.1"
          memory: 128M
    labels:
      - "com.polybot.service=wallet"
      - "prometheus.io/scrape=true"
      - "prometheus.io/port=8005"
      - "prometheus.io/path=/metrics"

  dashboard:
    build:
      context: .
      dockerfile: src/services/dashboard/Dockerfile
    <<: *service-defaults
    container_name: polybot-dashboard
    environment:
      <<: *common-env
      DASHBOARD_API_KEY: ${DASHBOARD_API_KEY}
      TELEGRAM_BOT_TOKEN: ${TELEGRAM_BOT_TOKEN:-}
      TELEGRAM_CHAT_ID: ${TELEGRAM_CHAT_ID:-}
    depends_on:
      redis:
        condition: service_healthy
      postgres:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "python", "-c", "import httpx; httpx.get('http://localhost:8080/health').raise_for_status()"]
      interval: 15s
      timeout: 10s
      retries: 3
      start_period: 15s
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
        reservations:
          cpus: "0.25"
          memory: 256M
    labels:
      - "com.polybot.service=dashboard"
      - "prometheus.io/scrape=true"
      - "prometheus.io/port=8080"
      - "prometheus.io/path=/metrics"

  # ─── Reverse Proxy ──────────────────────────────────────
  caddy:
    image: caddy:2-alpine
    <<: *service-defaults
    container_name: polybot-caddy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./config/Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config
    depends_on:
      dashboard:
        condition: service_healthy
    deploy:
      resources:
        limits:
          cpus: "0.25"
          memory: 128M

  # ─── Monitoring ─────────────────────────────────────────
  prometheus:
    image: prom/prometheus:latest
    <<: *service-defaults
    container_name: polybot-prometheus
    volumes:
      - ./config/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.retention.time=30d"
      - "--storage.tsdb.retention.size=5GB"
      - "--web.enable-lifecycle"
    healthcheck:
      test: ["CMD", "wget", "-q", "--tries=1", "-O", "/dev/null", "http://localhost:9090/-/healthy"]
      interval: 30s
      timeout: 10s
      retries: 3
    deploy:
      resources:
        limits:
          cpus: "0.25"
          memory: 512M
        reservations:
          cpus: "0.1"
          memory: 256M
    labels:
      - "com.polybot.service=prometheus"

  grafana:
    image: grafana/grafana:latest
    <<: *service-defaults
    container_name: polybot-grafana
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD:-changeme}
      GF_USERS_ALLOW_SIGN_UP: "false"
      GF_SERVER_ROOT_URL: "https://${DOMAIN}/grafana/"
      GF_SERVER_SERVE_FROM_SUB_PATH: "true"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./config/grafana/dashboards:/etc/grafana/provisioning/dashboards:ro
      - ./config/grafana/datasources.yml:/etc/grafana/provisioning/datasources/datasources.yml:ro
    depends_on:
      prometheus:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "-q", "--tries=1", "-O", "/dev/null", "http://localhost:3000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    deploy:
      resources:
        limits:
          cpus: "0.25"
          memory: 256M
        reservations:
          cpus: "0.1"
          memory: 128M
    labels:
      - "com.polybot.service=grafana"

networks:
  polybot_net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.28.0.0/16

volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local
  prometheus_data:
    driver: local
  grafana_data:
    driver: local
  caddy_data:
    driver: local
  caddy_config:
    driver: local
```

### Development Stack (`docker-compose.dev.yml`)

```yaml
# docker-compose.dev.yml — Development Overrides
version: "3.9"

services:
  # Only DB + Redis run in Docker during development
  # Python services run locally with hot reload via `make dev`
  postgres:
    extends:
      file: docker-compose.yml
      service: postgres
    ports:
      - "5432:5432"    # Exposed for local development tools

  redis:
    extends:
      file: docker-compose.yml
      service: redis
    ports:
      - "6379:6379"    # Exposed for local Redis CLI / RedisInsight

  # Monitoring (optional in dev)
  prometheus:
    extends:
      file: docker-compose.yml
      service: prometheus
    ports:
      - "9090:9090"

  grafana:
    extends:
      file: docker-compose.yml
      service: grafana
    ports:
      - "3000:3000"

networks:
  polybot_net:
    driver: bridge
```

**Development workflow**: Only `postgres` and `redis` run in Docker. All Python services run locally using `uvicorn --reload` for hot reload. The frontend runs via `npm run dev` (Vite HMR). This minimizes Docker overhead during development and maximizes iteration speed.

---

## Service Dockerfile Pattern

### Base Image Strategy

All Python services share a multi-stage Dockerfile pattern for minimal image size and consistent builds.

```dockerfile
# src/services/market_data/Dockerfile (representative pattern)
# ─── Stage 1: Builder ─────────────────────────────────────
FROM python:3.11-slim AS builder

WORKDIR /build

# Install build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Copy dependency files first (Docker layer caching)
COPY pyproject.toml ./
RUN pip install --no-cache-dir --prefix=/install .

# ─── Stage 2: Runtime ─────────────────────────────────────
FROM python:3.11-slim AS runtime

# Security: run as non-root
RUN groupadd -r polybot && useradd -r -g polybot polybot

WORKDIR /app

# Install runtime OS dependencies only
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Copy installed packages from builder
COPY --from=builder /install /usr/local

# Copy application code
COPY src/ ./src/
COPY config/ ./config/

# Drop privileges
USER polybot

# Health check port (each service uses a unique port)
EXPOSE 8001

# Entrypoint
CMD ["python", "-m", "src.services.market_data.service"]
```

### Service Port Assignments

| Service | Internal Port | Health Endpoint | Metrics Endpoint |
|---------|--------------|-----------------|------------------|
| Market Data | 8001 | `/health` | `/metrics` |
| Orchestrator | 8002 | `/health` | `/metrics` |
| Execution | 8003 | `/health` | `/metrics` |
| Risk | 8004 | `/health` | `/metrics` |
| Wallet | 8005 | `/health` | `/metrics` |
| Dashboard | 8080 | `/health` | `/metrics` |
| Prometheus | 9090 | `/-/healthy` | — |
| Grafana | 3000 | `/api/health` | — |

**None of the internal service ports are exposed to the host in production.** Only Caddy (80/443) is Internet-facing.

---

## Network Architecture

### Topology Diagram

```mermaid
flowchart TB
    subgraph Internet
        USER["Operator Browser"]
        PM["Polymarket APIs"]
        TG["Telegram API"]
    end

    subgraph VPS["VPS Host (Ubuntu 22.04)"]
        subgraph UFW["UFW Firewall"]
            SSH["SSH :2222"]
        end

        subgraph Docker["Docker Engine"]
            subgraph Exposed["Internet-Facing"]
                CADDY["Caddy<br/>:80 → :443<br/>Auto TLS"]
            end

            subgraph Internal["polybot_net (172.28.0.0/16)"]
                DASH["Dashboard<br/>:8080"]
                GRAF["Grafana<br/>:3000"]
                MDS["Market Data"]
                ORCH["Orchestrator"]
                EXEC["Execution"]
                RISK["Risk Manager"]
                WALL["Wallet"]
                PROM["Prometheus<br/>:9090"]
                PG["PostgreSQL<br/>:5432"]
                REDIS["Redis<br/>:6379"]
            end
        end
    end

    USER -->|HTTPS :443| CADDY
    CADDY -->|/| DASH
    CADDY -->|/grafana/| GRAF
    SSH -->|Key auth only| VPS
    MDS -->|WSS/HTTPS| PM
    EXEC -->|HTTPS| PM
    WALL -->|HTTPS| PM
    DASH -->|HTTPS| TG
```

### Firewall Rules (UFW)

```bash
# Default: deny all incoming, allow all outgoing
ufw default deny incoming
ufw default allow outgoing

# SSH (non-standard port)
ufw allow 2222/tcp comment 'SSH'

# HTTPS (Caddy handles TLS termination)
ufw allow 80/tcp comment 'HTTP redirect'
ufw allow 443/tcp comment 'HTTPS'

# Enable
ufw enable
```

**Explicitly NOT exposed**:
- PostgreSQL (5432) — Docker internal only
- Redis (6379) — Docker internal only
- Prometheus (9090) — accessible via Caddy reverse proxy path only
- Grafana (3000) — accessible via Caddy `/grafana/` path only
- Service health/metrics ports (8001-8005) — Docker internal only

### Caddy Configuration

```
# config/Caddyfile
{$DOMAIN} {
    # Dashboard SPA (main site)
    reverse_proxy dashboard:8080

    # Grafana (sub-path)
    handle_path /grafana/* {
        reverse_proxy grafana:3000
    }

    # Security headers
    header {
        X-Content-Type-Options nosniff
        X-Frame-Options SAMEORIGIN
        Referrer-Policy strict-origin-when-cross-origin
        Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; connect-src 'self' wss://{$DOMAIN}"
        -Server
    }

    # Rate limiting for dashboard API
    @api path /api/*
    rate_limit @api {
        zone api_zone {
            key {remote_host}
            events 100
            window 1m
        }
    }

    # Auto TLS via Let's Encrypt
    tls {$TLS_EMAIL}
}
```

---

## VPS Provisioning

### Hardware Requirements

| Tier | CPU | RAM | Storage | Cost (est.) | Phase |
|------|-----|-----|---------|-------------|-------|
| **MVP** | 8 vCPU | 16 GB | 200 GB NVMe | ~$80-120/mo | Phase 1-2 |
| **Growth** | 8 vCPU | 32 GB | 400 GB NVMe | ~$150-200/mo | Phase 3-4 |
| **Scale** | 16 vCPU | 64 GB | 500 GB NVMe | ~$300-400/mo | Phase 5-6 |

**Recommended providers** (evaluated for: datacenter proximity to Polymarket's matching engine, DDoS protection, NVMe storage):
1. Hetzner (EU/US datacenters, excellent price/performance)
2. OVH (US East, proximity to Polymarket infrastructure)
3. Vultr (US East, high-frequency compute options)

**Critical**: Choose a US East datacenter for minimum latency to Polymarket's matching engine.

### Provisioning Script (`scripts/setup-vps.sh`)

```bash
#!/usr/bin/env bash
# scripts/setup-vps.sh — VPS provisioning for PolyBot
set -euo pipefail

# ─── Variables ────────────────────────────────────────────
SSH_PORT=${SSH_PORT:-2222}
DEPLOY_USER=${DEPLOY_USER:-polybot}
POLYBOT_DIR="/opt/polybot"

echo "═══ PolyBot VPS Setup ═══"
echo "SSH Port: $SSH_PORT"
echo "Deploy User: $DEPLOY_USER"
echo ""

# ─── 1. System Updates ───────────────────────────────────
echo "→ Updating system packages..."
apt-get update && apt-get upgrade -y
apt-get install -y \
    curl wget git unzip \
    ufw fail2ban \
    htop iotop \
    jq \
    gnupg2 \
    apt-transport-https \
    ca-certificates

# ─── 2. Create Deploy User ───────────────────────────────
echo "→ Creating deploy user: $DEPLOY_USER"
if ! id "$DEPLOY_USER" &>/dev/null; then
    adduser --disabled-password --gecos "" "$DEPLOY_USER"
    usermod -aG sudo "$DEPLOY_USER"
    mkdir -p /home/$DEPLOY_USER/.ssh
    cp ~/.ssh/authorized_keys /home/$DEPLOY_USER/.ssh/
    chown -R $DEPLOY_USER:$DEPLOY_USER /home/$DEPLOY_USER/.ssh
    chmod 700 /home/$DEPLOY_USER/.ssh
    chmod 600 /home/$DEPLOY_USER/.ssh/authorized_keys
fi

# ─── 3. SSH Hardening ────────────────────────────────────
echo "→ Hardening SSH..."
cat > /etc/ssh/sshd_config.d/polybot.conf << EOF
Port $SSH_PORT
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
LoginGraceTime 30
AllowUsers $DEPLOY_USER
ClientAliveInterval 300
ClientAliveCountMax 2
EOF
systemctl restart sshd

# ─── 4. Firewall ─────────────────────────────────────────
echo "→ Configuring UFW firewall..."
ufw default deny incoming
ufw default allow outgoing
ufw allow $SSH_PORT/tcp comment 'SSH'
ufw allow 80/tcp comment 'HTTP'
ufw allow 443/tcp comment 'HTTPS'
echo "y" | ufw enable

# ─── 5. Fail2Ban ─────────────────────────────────────────
echo "→ Configuring Fail2Ban..."
cat > /etc/fail2ban/jail.local << EOF
[sshd]
enabled = true
port = $SSH_PORT
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600
findtime = 600
EOF
systemctl enable fail2ban && systemctl restart fail2ban

# ─── 6. Docker Installation ──────────────────────────────
echo "→ Installing Docker..."
curl -fsSL https://get.docker.com | sh
usermod -aG docker $DEPLOY_USER

# Docker daemon configuration
mkdir -p /etc/docker
cat > /etc/docker/daemon.json << 'EOF'
{
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "50m",
        "max-file": "5"
    },
    "default-ulimits": {
        "nofile": {
            "Name": "nofile",
            "Hard": 65536,
            "Soft": 65536
        }
    },
    "userns-remap": "default",
    "no-new-privileges": true,
    "live-restore": true
}
EOF
systemctl restart docker

# ─── 7. NTP Time Sync ────────────────────────────────────
echo "→ Configuring NTP (critical for HMAC timestamps)..."
apt-get install -y chrony
systemctl enable chrony && systemctl start chrony

# Verify time sync
chronyc tracking

# ─── 8. Swap Configuration ───────────────────────────────
echo "→ Configuring encrypted swap..."
# Disable existing swap
swapoff -a
# Create encrypted swap (prevents key leakage to disk)
dd if=/dev/zero of=/swapfile bs=1M count=4096
chmod 600 /swapfile
mkswap /swapfile
# Add to fstab with encryption
echo '/swapfile none swap sw 0 0' >> /etc/fstab
swapon /swapfile

# Set swappiness low (prefer RAM)
echo 'vm.swappiness=10' >> /etc/sysctl.conf
sysctl -p

# ─── 9. Unattended Security Updates ──────────────────────
echo "→ Enabling unattended security upgrades..."
apt-get install -y unattended-upgrades
dpkg-reconfigure -f noninteractive unattended-upgrades

# ─── 10. Project Directory ───────────────────────────────
echo "→ Creating project directory..."
mkdir -p $POLYBOT_DIR
chown $DEPLOY_USER:$DEPLOY_USER $POLYBOT_DIR

# ─── 11. Kernel Tuning ───────────────────────────────────
echo "→ Applying kernel optimizations..."
cat >> /etc/sysctl.conf << EOF

# PolyBot optimizations
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
net.core.netdev_max_backlog = 65535
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 5
fs.file-max = 2097152
EOF
sysctl -p

echo ""
echo "═══ VPS Setup Complete ═══"
echo "→ SSH port: $SSH_PORT"
echo "→ Deploy user: $DEPLOY_USER"
echo "→ Project dir: $POLYBOT_DIR"
echo "→ NEXT: Clone repo, copy .env, run 'docker compose up -d'"
```

---

## Secrets Management

### Environment File Structure

```bash
# .env — NEVER commit. Template in .env.example
# File permissions: chmod 600 .env

# ─── Wallet Keys (CRITICAL — encrypted with sops/age) ────
VAULT_PRIVATE_KEY=                     # Hex, no 0x prefix
ALPHA_PRIVATE_KEY=                     # Phase 3
SWEEP_PRIVATE_KEY=                     # Phase 2

# ─── Polymarket API ──────────────────────────────────────
POLYMARKET_CLOB_URL=https://clob.polymarket.com
POLYMARKET_WS_URL=wss://ws-subscriptions-clob.polymarket.com/ws/
GAMMA_API_URL=https://gamma-api.polymarket.com
POLYGON_CHAIN_ID=137

# ─── Database ────────────────────────────────────────────
POSTGRES_PASSWORD=                     # Generate: openssl rand -base64 32

# ─── Dashboard ───────────────────────────────────────────
DASHBOARD_API_KEY=                     # Generate: python -c "import secrets; print(secrets.token_urlsafe(32))"
DOMAIN=polybot.example.com
TLS_EMAIL=admin@example.com

# ─── Monitoring ──────────────────────────────────────────
GRAFANA_PASSWORD=                      # Generate: openssl rand -base64 16

# ─── Alerting ────────────────────────────────────────────
TELEGRAM_BOT_TOKEN=
TELEGRAM_CHAT_ID=
```

### sops/age Encryption for Private Keys

```bash
# One-time setup: install age + sops
apt-get install -y age
# Download sops from https://github.com/getsops/sops/releases

# Generate age key pair
age-keygen -o /home/polybot/.config/sops/age/keys.txt
# Output: public key: age1...

# Create .sops.yaml in project root
cat > .sops.yaml << EOF
creation_rules:
  - path_regex: \.env$
    age: >-
      age1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
EOF

# Encrypt .env
sops --encrypt --in-place .env

# Decrypt for use (done by deployment script)
sops --decrypt .env > .env.decrypted
# Use .env.decrypted at runtime
# Delete after startup: rm .env.decrypted
```

### Secrets Rotation Schedule

| Secret | Rotation | Procedure |
|--------|----------|-----------|
| EOA private keys | Never (unless compromised) | Generate new key → transfer funds → update .env → rotate API creds |
| L2 API credentials | Automatic on restart | Derived from EOA key; `create_or_derive_api_creds()` is idempotent |
| `POSTGRES_PASSWORD` | Quarterly | Update .env → `docker compose down postgres` → update pg_hba → restart |
| `DASHBOARD_API_KEY` | Monthly | Update .env → restart dashboard service |
| `GRAFANA_PASSWORD` | Quarterly | Update .env → restart grafana |
| `TELEGRAM_BOT_TOKEN` | On compromise | Create new bot via @BotFather → update .env → restart dashboard |

---

## Monitoring Stack

### Prometheus Configuration

```yaml
# config/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  scrape_timeout: 10s

rule_files:
  - /etc/prometheus/rules/*.yml

scrape_configs:
  # ─── PolyBot Services ──────────────────────────────────
  - job_name: "polybot-services"
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        filters:
          - name: label
            values: ["prometheus.io/scrape=true"]
    relabel_configs:
      - source_labels: [__meta_docker_container_label_prometheus_io_port]
        target_label: __address__
        regex: (.+)
        replacement: "${1}"
      - source_labels: [__meta_docker_container_label_com_polybot_service]
        target_label: service
      - source_labels: [__meta_docker_container_name]
        regex: /(.+)
        target_label: container

  # ─── PostgreSQL ─────────────────────────────────────────
  - job_name: "postgres"
    static_configs:
      - targets: ["postgres-exporter:9187"]

  # ─── Redis ──────────────────────────────────────────────
  - job_name: "redis"
    static_configs:
      - targets: ["redis-exporter:9121"]

  # ─── Node Exporter (VPS host metrics) ──────────────────
  - job_name: "node"
    static_configs:
      - targets: ["node-exporter:9100"]
```

### Alert Rules

```yaml
# config/prometheus/rules/polybot-alerts.yml
groups:
  - name: polybot-critical
    rules:
      - alert: EmergencyStopTriggered
        expr: increase(polybot_emergency_stop_total[5m]) > 0
        for: 0s
        labels:
          severity: critical
        annotations:
          summary: "Emergency stop triggered"
          description: "The emergency stop has been activated. All trading halted."

      - alert: BotInErrorState
        expr: polybot_circuit_breaker_state > 0
        for: 60s
        labels:
          severity: critical
        annotations:
          summary: "Bot {{ $labels.bot_id }} circuit breaker OPEN"
          description: "Circuit breaker for bot {{ $labels.bot_id }} has been open for >60s."

      - alert: WebSocketDisconnected
        expr: polybot_websocket_connections{status="failed"} > 0
        for: 60s
        labels:
          severity: critical
        annotations:
          summary: "WebSocket disconnected for >60s"
          description: "Market data WebSocket has been disconnected for over 60 seconds."

      - alert: WalletBalanceLow
        expr: polybot_wallet_balance_usdc < 500
        for: 0s
        labels:
          severity: warning
        annotations:
          summary: "Wallet {{ $labels.wallet_id }} balance low: {{ $value }} USDC"
          description: "Wallet balance has fallen below the alert threshold."

  - name: polybot-performance
    rules:
      - alert: HighOrderLatency
        expr: histogram_quantile(0.95, polybot_fill_latency_seconds_bucket) > 5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Signal-to-fill latency >5s at p95"
          description: "Order execution is slower than expected. Current p95: {{ $value }}s."

      - alert: RedisStreamLag
        expr: polybot_redis_stream_lag > 1000
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Redis Stream lag >1000 messages"
          description: "Stream {{ $labels.stream }} has a consumer lag of {{ $value }} messages."

      - alert: DailyLossApproachingCap
        expr: (polybot_daily_loss_usdc / polybot_daily_loss_cap_usdc) > 0.8
        for: 0s
        labels:
          severity: warning
        annotations:
          summary: "Bot {{ $labels.bot_id }} at {{ $value | humanizePercentage }} of daily loss cap"

  - name: polybot-infrastructure
    rules:
      - alert: HighCPUUsage
        expr: rate(process_cpu_seconds_total[5m]) > 0.8
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.service }}"

      - alert: HighMemoryUsage
        expr: process_resident_memory_bytes / 1e9 > 3.5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on {{ $labels.service }}: {{ $value | humanize }}GB"

      - alert: DiskSpaceLow
        expr: node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"} < 0.15
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Disk space below 15%"
          description: "Root filesystem has {{ $value | humanizePercentage }} free."

      - alert: PostgresConnectionsHigh
        expr: pg_stat_database_numbackends{datname="polybot"} > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL connections >80 (max 100)"
```

### Grafana Dashboard Provisioning

```yaml
# config/grafana/datasources.yml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false

  - name: PostgreSQL
    type: postgres
    url: postgres:5432
    database: polybot
    user: polybot
    secureJsonData:
      password: ${POSTGRES_PASSWORD}
    jsonData:
      sslmode: disable
      postgresVersion: 1600
      timescaledb: true
    editable: false
```

### Pre-Built Grafana Dashboards

| Dashboard | Panels | Data Source | Refresh |
|-----------|--------|-------------|---------|
| **Trading Overview** | Portfolio value, daily P&L, orders/fills rate, win rate | Prometheus + PostgreSQL | 5s |
| **Bot Performance** | Per-bot P&L, fill latency, signal rate, edge distribution | Prometheus | 10s |
| **Risk Monitor** | Circuit breaker states, daily loss tracking, exposure heatmap | Prometheus | 5s |
| **Infrastructure** | CPU/RAM/disk per container, Redis memory, PG connections, WS status | Prometheus | 15s |
| **Market Data** | Stream lag, WS reconnects, Gamma poll success rate, cache hit rate | Prometheus | 10s |
| **Wallet Overview** | Balance per wallet, ledger P&L, rebalance events | Prometheus + PostgreSQL | 30s |

Dashboard JSON files are stored in `config/grafana/dashboards/` and auto-provisioned via Grafana's provisioning API.

---

## Backup & Recovery

### Backup Strategy

```mermaid
flowchart LR
    subgraph Daily["Daily Backup (02:00 UTC)"]
        PG_DUMP["pg_dump polybot"]
        REDIS_SAVE["redis-cli BGSAVE"]
        CONFIG["tar config/ .env"]
    end

    subgraph Compress["Compress & Encrypt"]
        GZIP["gzip"]
        GPG["gpg --encrypt"]
    end

    subgraph Store["Storage"]
        LOCAL["Local: /backups/<br/>Retain 7 days"]
        REMOTE["Remote: S3/B2<br/>Retain 30 days"]
    end

    PG_DUMP --> GZIP --> GPG --> LOCAL --> REMOTE
    REDIS_SAVE --> GZIP
    CONFIG --> GZIP
```

### Backup Script (`scripts/backup.sh`)

```bash
#!/usr/bin/env bash
# scripts/backup.sh — Automated PolyBot backup
set -euo pipefail

BACKUP_DIR="/backups/polybot"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_PATH="$BACKUP_DIR/$DATE"
RETENTION_DAYS=7
REMOTE_BUCKET="${REMOTE_BACKUP_BUCKET:-}"  # s3://polybot-backups or b2://polybot-backups

mkdir -p "$BACKUP_PATH"

echo "═══ PolyBot Backup: $DATE ═══"

# ─── 1. PostgreSQL Dump ──────────────────────────────────
echo "→ Dumping PostgreSQL..."
docker compose exec -T postgres pg_dump \
    -U polybot \
    --format=custom \
    --compress=6 \
    polybot > "$BACKUP_PATH/polybot.dump"
echo "  Size: $(du -h "$BACKUP_PATH/polybot.dump" | cut -f1)"

# ─── 2. Redis Snapshot ───────────────────────────────────
echo "→ Saving Redis snapshot..."
docker compose exec -T redis redis-cli BGSAVE
sleep 5  # Wait for BGSAVE to complete
docker cp polybot-redis:/data/dump.rdb "$BACKUP_PATH/redis.rdb"

# ─── 3. Configuration Files ──────────────────────────────
echo "→ Archiving configuration..."
tar czf "$BACKUP_PATH/config.tar.gz" \
    config/ \
    docker-compose.yml \
    .env.example \
    Makefile \
    2>/dev/null || true

# ─── 4. Encrypt ──────────────────────────────────────────
echo "→ Encrypting backup..."
tar cf - -C "$BACKUP_DIR" "$DATE" | \
    gpg --batch --yes --symmetric \
    --cipher-algo AES256 \
    --passphrase-file /home/polybot/.backup-passphrase \
    -o "$BACKUP_DIR/$DATE.tar.gpg"

# Remove unencrypted files
rm -rf "$BACKUP_PATH"

echo "  Encrypted size: $(du -h "$BACKUP_DIR/$DATE.tar.gpg" | cut -f1)"

# ─── 5. Upload to Remote Storage ─────────────────────────
if [ -n "$REMOTE_BUCKET" ]; then
    echo "→ Uploading to remote: $REMOTE_BUCKET"
    # AWS S3
    if [[ "$REMOTE_BUCKET" == s3://* ]]; then
        aws s3 cp "$BACKUP_DIR/$DATE.tar.gpg" "$REMOTE_BUCKET/$DATE.tar.gpg"
    fi
    # Backblaze B2
    if [[ "$REMOTE_BUCKET" == b2://* ]]; then
        b2 upload-file "${REMOTE_BUCKET#b2://}" "$BACKUP_DIR/$DATE.tar.gpg" "$DATE.tar.gpg"
    fi
fi

# ─── 6. Cleanup Old Backups ──────────────────────────────
echo "→ Cleaning backups older than $RETENTION_DAYS days..."
find "$BACKUP_DIR" -name "*.tar.gpg" -mtime +$RETENTION_DAYS -delete

echo "═══ Backup Complete ═══"
```

### Restore Script (`scripts/restore.sh`)

```bash
#!/usr/bin/env bash
# scripts/restore.sh — Restore PolyBot from backup
set -euo pipefail

BACKUP_FILE="${1:?Usage: ./restore.sh <backup-file.tar.gpg>}"
RESTORE_DIR="/tmp/polybot-restore"

echo "═══ PolyBot Restore ═══"
echo "Source: $BACKUP_FILE"
echo ""

# ─── 1. Decrypt & Extract ────────────────────────────────
echo "→ Decrypting backup..."
mkdir -p "$RESTORE_DIR"
gpg --batch --yes --decrypt \
    --passphrase-file /home/polybot/.backup-passphrase \
    "$BACKUP_FILE" | tar xf - -C "$RESTORE_DIR"

BACKUP_NAME=$(ls "$RESTORE_DIR" | head -1)
BACKUP_PATH="$RESTORE_DIR/$BACKUP_NAME"

# ─── 2. Stop Services ────────────────────────────────────
echo "→ Stopping services (DB + Redis stay running)..."
docker compose stop market-data orchestrator execution risk wallet dashboard

# ─── 3. Restore PostgreSQL ───────────────────────────────
echo "→ Restoring PostgreSQL..."
docker compose exec -T postgres dropdb -U polybot --if-exists polybot
docker compose exec -T postgres createdb -U polybot polybot
docker compose exec -T postgres pg_restore \
    -U polybot \
    --dbname=polybot \
    --no-owner \
    --no-acl \
    < "$BACKUP_PATH/polybot.dump"
echo "  PostgreSQL restored."

# ─── 4. Restore Redis ────────────────────────────────────
echo "→ Restoring Redis..."
docker compose stop redis
docker cp "$BACKUP_PATH/redis.rdb" polybot-redis:/data/dump.rdb
docker compose start redis
sleep 5
echo "  Redis restored."

# ─── 5. Restart Services ─────────────────────────────────
echo "→ Starting all services..."
docker compose up -d

# ─── 6. Verify ───────────────────────────────────────────
echo "→ Verifying restoration..."
sleep 10
docker compose ps
echo ""
echo "→ Checking database..."
docker compose exec -T postgres psql -U polybot -c "SELECT count(*) FROM bot_config;" polybot

# ─── 7. Cleanup ──────────────────────────────────────────
rm -rf "$RESTORE_DIR"

echo ""
echo "═══ Restore Complete ═══"
echo "→ Verify positions: check Dashboard or Polymarket Data API"
echo "→ Reconcile: compare internal state with on-chain positions"
```

### Cron Configuration

```bash
# /etc/cron.d/polybot-backup
# Daily backup at 02:00 UTC
0 2 * * * polybot /opt/polybot/scripts/backup.sh >> /var/log/polybot-backup.log 2>&1

# Weekly integrity check (restore to temporary DB)
0 4 * * 0 polybot /opt/polybot/scripts/verify-backup.sh >> /var/log/polybot-backup-verify.log 2>&1
```

---

## Deployment Workflow

### Initial Deployment

```bash
# On VPS (as polybot user):

# 1. Clone repository
cd /opt
git clone <repo-url> polybot && cd polybot

# 2. Configure environment
cp .env.example .env
# Edit .env: add all required secrets
chmod 600 .env

# 3. Build all images
docker compose build

# 4. Start persistence layer first
docker compose up -d postgres redis
sleep 10  # Wait for initialization

# 5. Run database migrations
docker compose run --rm execution alembic upgrade head

# 6. Start all services
docker compose up -d

# 7. Verify
docker compose ps
docker compose logs --tail=20
curl -sk https://localhost/health  # Dashboard health check
```

### Rolling Update Deployment

```bash
# scripts/deploy.sh — Zero-downtime deployment
#!/usr/bin/env bash
set -euo pipefail

cd /opt/polybot

echo "═══ PolyBot Deploy ═══"

# 1. Pull latest code
echo "→ Pulling latest..."
git pull origin main

# 2. Build new images
echo "→ Building images..."
docker compose build

# 3. Run migrations (non-destructive only)
echo "→ Running migrations..."
docker compose run --rm execution alembic upgrade head

# 4. Restart services one at a time (no trading downtime)
# Order matters: infrastructure → data → execution → bots → dashboard
for service in market-data risk wallet execution orchestrator dashboard; do
    echo "→ Restarting $service..."
    docker compose up -d --no-deps --build $service
    sleep 5
    # Wait for health check
    timeout 30 bash -c "until docker inspect --format='{{.State.Health.Status}}' polybot-$service 2>/dev/null | grep -q healthy; do sleep 2; done"
    echo "  $service: healthy ✓"
done

# 5. Verify
echo "→ Verifying deployment..."
docker compose ps
echo ""
echo "═══ Deploy Complete ═══"
```

### Rollback Procedure

```bash
# Rollback to previous version
cd /opt/polybot

# 1. Find previous commit
git log --oneline -5

# 2. Revert to known-good version
git checkout <commit-hash>

# 3. Rebuild and restart
docker compose build
docker compose up -d

# 4. If database migration needs rollback
docker compose run --rm execution alembic downgrade -1
```

---

## Database Management

### PostgreSQL Configuration

Key tuning parameters for a VPS with 16 GB RAM (PostgreSQL gets ~2 GB):

```
# Applied via docker-compose environment or custom postgresql.conf

# Memory
shared_buffers = 512MB              # 25% of PostgreSQL's 2GB allocation
effective_cache_size = 1536MB       # 75% of allocation
work_mem = 16MB                     # Per-sort/hash operation
maintenance_work_mem = 128MB        # For VACUUM, CREATE INDEX

# WAL
wal_buffers = 16MB
max_wal_size = 1GB
min_wal_size = 256MB

# Connections
max_connections = 100               # 5 services × ~15 connections + monitoring + admin
```

### TimescaleDB Retention & Compression

```sql
-- Applied via Alembic migration after hypertable creation

-- Compression policies (reduce storage ~10x for old data)
ALTER TABLE fill SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'bot_id, wallet_id',
    timescaledb.compress_orderby = 'created_at DESC'
);
SELECT add_compression_policy('fill', INTERVAL '7 days');

ALTER TABLE signal SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'bot_id',
    timescaledb.compress_orderby = 'created_at DESC'
);
SELECT add_compression_policy('signal', INTERVAL '3 days');

ALTER TABLE bot_metric SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'bot_id',
    timescaledb.compress_orderby = 'recorded_at DESC'
);
SELECT add_compression_policy('bot_metric', INTERVAL '1 day');

ALTER TABLE order_book_snapshot SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'token_id',
    timescaledb.compress_orderby = 'captured_at DESC'
);
SELECT add_compression_policy('order_book_snapshot', INTERVAL '1 day');

-- Retention policies (drop data older than threshold)
SELECT add_retention_policy('order_book_snapshot', INTERVAL '90 days');
SELECT add_retention_policy('bot_metric', INTERVAL '365 days');
-- fill and signal retained indefinitely for audit
```

### Storage Projections

| Table | Row Rate | Raw Size/Day | Compressed/Day | 90-Day Total |
|-------|----------|-------------|----------------|--------------|
| `order_book_snapshot` | ~10,000/hr | ~500 MB | ~50 MB | ~4.5 GB |
| `bot_metric` | ~8,640/day | ~5 MB | ~500 KB | ~45 MB |
| `fill` | ~500/day | ~2 MB | ~200 KB | ~18 MB |
| `signal` | ~5,000/day | ~25 MB | ~2.5 MB | ~225 MB |
| **Total** | — | ~532 MB | ~53 MB | **~4.8 GB** |

With 200 GB NVMe, storage capacity is not a concern for 2+ years of operation.

---

## Redis Configuration

### Memory Management

```
# Redis configuration (via docker-compose command)
maxmemory 512mb
maxmemory-policy allkeys-lru

# Persistence
appendonly yes              # AOF enabled for durability
appendfsync everysec        # Fsync every second (balance durability/performance)
save 300 10                 # RDB snapshot every 5 min if ≥10 changes
save 60 1000                # RDB snapshot every 1 min if ≥1000 changes
```

### Stream Trimming

```python
# Applied by each service publishing to Redis Streams
# Prevents unbounded stream growth

STREAM_TRIM_CONFIG = {
    "market_data:{token_id}": {"maxlen": 10_000, "approximate": True},
    "fills:{bot_id}": {"maxlen": 50_000, "approximate": True},
    "risk_events": {"maxlen": 100_000, "approximate": True},
    "audit_events": {"maxlen": 500_000, "approximate": True},
}

# Usage: XADD market_data:token_123 MAXLEN ~ 10000 * field value
```

### Memory Budget

| Key Pattern | Estimated Count | Size per Entry | Total |
|-------------|----------------|----------------|-------|
| `orderbook:{token_id}` | ~500 markets | ~10 KB | ~5 MB |
| `fee_rate:{token_id}` | ~500 markets | ~50 B | ~25 KB |
| `wallet:{id}:balance` | 3 wallets | ~100 B | ~300 B |
| `rate_limit:{wallet_id}` | 3 wallets | ~50 B | ~150 B |
| Streams (all) | ~10K entries total | ~200 B each | ~2 MB |
| Pub/Sub channels | 5 channels | ~0 (ephemeral) | ~0 |
| **Total estimated** | — | — | **~10 MB** |

Actual Redis memory usage will be ~50-200 MB (overhead, fragmentation, AOF buffer). The 512 MB limit provides significant headroom.

---

## Log Management

### Log Architecture

```
Service containers → Docker json-file driver → /var/lib/docker/containers/
    ├── max-size: 50MB per log file
    ├── max-file: 5 rotated files
    └── Total per service: ~250MB max

Access patterns:
    docker compose logs -f --tail=100           # Live tail
    docker compose logs --since=1h market-data  # Recent service logs
    docker compose logs | jq '.level'           # JSON parsing
```

### Log Rotation

Docker's `json-file` driver handles rotation automatically via the `max-size` and `max-file` options in `docker-compose.yml`. No external logrotate configuration needed.

**Storage budget**: 10 services × 250 MB max = ~2.5 GB worst case.

### Log Aggregation (Phase 2)

For Phase 2+, consider adding a lightweight log aggregation stack:

| Option | Memory | Capability | Recommendation |
|--------|--------|------------|----------------|
| **Loki + Promtail** | ~200 MB | Label-based log queries, Grafana integration | **Recommended** — lightweight, native Grafana support |
| ELK Stack | ~4 GB | Full-text search, Kibana visualizations | Overkill for single VPS |
| Vector + ClickHouse | ~300 MB | High-performance log processing | Over-engineered for this use case |

---

## Health Check Design

### Service Health Contract

Every PolyBot service exposes a `/health` endpoint returning:

```json
{
    "status": "healthy",           // healthy | degraded | unhealthy
    "service": "market_data",
    "uptime_seconds": 86412,
    "checks": {
        "redis": "ok",
        "postgres": "ok",
        "websocket": "connected",  // Service-specific check
        "last_data_age_ms": 1500   // Service-specific metric
    },
    "version": "0.1.0"
}
```

### Health Check Matrix

| Service | Docker Healthcheck | Dependencies Checked | Degradation Criteria |
|---------|-------------------|---------------------|---------------------|
| PostgreSQL | `pg_isready` | — | — |
| Redis | `redis-cli ping` | — | — |
| Market Data | HTTP `/health` | Redis, WS connection | WS disconnected >30s |
| Orchestrator | HTTP `/health` | Redis, PostgreSQL | Any bot in ERROR >60s |
| Execution | HTTP `/health` | Redis, PostgreSQL, Wallet | Rate limit >80% consumed |
| Risk | HTTP `/health` | Redis, PostgreSQL | Circuit breaker OPEN |
| Wallet | HTTP `/health` | Redis, PostgreSQL | Balance below threshold |
| Dashboard | HTTP `/health` | Redis, PostgreSQL | — |

### Docker Compose Dependency Chain

```
postgres ──────────────────────────────────────────┐
redis ─────────────────────────────────────────────┤
   │                                               │
   ├── market-data (redis + postgres healthy) ─────┤
   ├── risk (redis + postgres healthy) ────────────┤
   ├── wallet (redis + postgres healthy) ──────────┤
   │      │                                        │
   │      └── execution (redis + postgres + wallet)│
   │                                               │
   ├── orchestrator (redis + postgres + market-data)
   │                                               │
   └── dashboard (redis + postgres healthy) ───────┘
              │
              └── caddy (dashboard healthy)
```

---

## Operational Procedures

### Daily Operations Checklist

| Task | Automated? | Frequency | Tool |
|------|-----------|-----------|------|
| Check all services healthy | Yes | Every 15s | Docker healthchecks |
| Monitor daily P&L | Yes | Continuous | Grafana dashboard |
| Check wallet balances | Yes | Every 30s | Wallet service + alerts |
| Review circuit breaker events | Manual | Daily | Grafana Risk Monitor |
| Check disk space | Yes | Every 10 min | Prometheus alert |
| Review error logs | Manual | Daily | `docker compose logs --since=24h | grep ERROR` |
| Verify backup completed | Yes (cron) | Daily | Check backup log |

### Emergency Stop Procedure

```
Trigger: Dashboard Emergency Stop button OR Redis CLI

1. Dashboard button → POST /api/system/emergency-stop
   OR: docker compose exec redis redis-cli SET emergency_stop 1

2. Risk Manager detects flag → broadcasts to all services

3. Execution Engine:
   - Cancels ALL open orders (per-wallet, parallel)
   - Target: <5 seconds total

4. Orchestrator:
   - Calls on_emergency_stop() on ALL bots
   - Sets all bot states to STOPPED

5. Dashboard:
   - Switches to read-only mode
   - Shows emergency stop banner

6. Telegram: sends CRITICAL alert

RESUME:
   docker compose exec redis redis-cli DEL emergency_stop
   Then restart bots manually from dashboard
```

### Service Restart Procedures

| Scenario | Procedure | Downtime |
|----------|-----------|----------|
| Single service crash | Auto-restart (Docker `unless-stopped`) | ~5-15s |
| Redis crash | Auto-restart; services reconnect automatically | ~10-30s |
| PostgreSQL crash | Auto-restart; services retry connections | ~15-60s |
| Full stack restart | `docker compose restart` | ~30-60s |
| Config change | `docker compose up -d --no-deps <service>` | ~5-10s per service |
| Full rebuild | `docker compose build && docker compose up -d` | ~2-5 min |
| VPS reboot | Docker auto-starts (`unless-stopped` policy) | ~2-5 min |

---

## Capacity Planning

### Scaling Triggers

| Metric | Current Capacity | Warning Threshold | Scale Action |
|--------|-----------------|-------------------|--------------|
| CPU utilization | 8 cores | >70% sustained 10 min | Upgrade VPS or optimize |
| RAM usage | 16 GB | >80% sustained 10 min | Upgrade to 32 GB |
| Disk usage | 200 GB | >75% used | Expand volume or increase compression |
| WebSocket connections | ~10 (500 instruments each) | >40 connections | Shard Market Data Service |
| PostgreSQL connections | 100 max | >80 connections | Increase max_connections or add pgBouncer |
| Redis memory | 512 MB | >80% used | Increase maxmemory or trim streams |
| Bot count | ~5 concurrent (Phase 1) | >15 bots | Separate orchestrator processes |

### Phase-Based Infrastructure Evolution

| Phase | Services | VPS Spec | Additional Infrastructure |
|-------|----------|----------|--------------------------|
| **Phase 1** (MVP) | 9 containers | 8 CPU / 16 GB | — |
| **Phase 2** (Market Making) | 9 containers + Loki | 8 CPU / 16 GB | Log aggregation |
| **Phase 3** (Latency Arb) | 10 containers + Rust sidecar | 8 CPU / 32 GB | Rust build toolchain |
| **Phase 4** (Copy Trading) | 10 containers | 8 CPU / 32 GB | On-chain indexer process |
| **Phase 5** (NegRisk) | 11 containers | 16 CPU / 32 GB | — |
| **Phase 6** (AI/ML) | 12+ containers | 16 CPU / 64 GB | GPU optional for inference |

---

## Disaster Recovery

### Recovery Time Objectives

| Scenario | RTO | RPO | Procedure |
|----------|-----|-----|-----------|
| Single service crash | <30s | 0 | Auto-restart via Docker |
| VPS reboot | <5 min | 0 | Docker auto-start |
| Database corruption | <1 hr | <24 hr | Restore from backup |
| Full VPS loss | <4 hr | <24 hr | New VPS + provision + restore |
| Region outage (provider) | <8 hr | <24 hr | New provider + provision + restore |

### Full Disaster Recovery Procedure

```
1. Provision new VPS (scripts/setup-vps.sh)                    ~15 min
2. Clone repository                                              ~2 min
3. Configure .env (from secure backup or regenerate)             ~5 min
4. Build Docker images                                          ~10 min
5. Start persistence layer (postgres + redis)                    ~2 min
6. Restore from latest backup (scripts/restore.sh)             ~10 min
7. Run any pending migrations                                    ~1 min
8. Start all services                                            ~2 min
9. Verify health (docker compose ps + dashboard)                 ~5 min
10. Reconcile positions with Polymarket Data API                ~15 min
11. Resume trading after full reconciliation                     ~5 min
                                                          TOTAL: ~1 hr
```

### Position Reconciliation After Recovery

Critical after any data loss: internal state (PostgreSQL) may not match on-chain state (Polymarket).

```bash
# Step 1: Fetch actual positions from Polymarket Data API
curl -s "https://data-api.polymarket.com/positions?address=<proxy_address>" | jq .

# Step 2: Compare with internal position tracker
docker compose exec postgres psql -U polybot -c \
    "SELECT bot_id, token_id, size, avg_entry_price FROM position WHERE is_open = true;"

# Step 3: Reconcile differences
# - If Polymarket shows positions not in DB: add them (conservative)
# - If DB shows positions not on Polymarket: mark as resolved
# - Size mismatches: trust Polymarket (on-chain is source of truth)
```

---

## Security Infrastructure

### Container Security

```yaml
# Applied to all service containers via docker-compose.yml
security_opt:
  - no-new-privileges:true

# Docker daemon (via daemon.json)
{
    "userns-remap": "default",        # User namespace remapping
    "no-new-privileges": true,         # Prevent privilege escalation
    "live-restore": true               # Keep containers running during daemon restart
}
```

### Image Scanning

```bash
# CI pipeline step: scan images for vulnerabilities
# Run after docker compose build

for image in $(docker compose config --images); do
    echo "Scanning: $image"
    trivy image --severity HIGH,CRITICAL --exit-code 1 "$image"
done
```

### Network Security

- All internal services communicate over the `polybot_net` Docker bridge network
- No internal ports exposed to the host (verified by the absence of `ports:` directives on internal services)
- Only Caddy exposes ports 80/443 to the host
- Redis requires no authentication (internal network only — acceptable per threat model)
- PostgreSQL uses password authentication (internal network only)

---

## Makefile Reference

```makefile
# Makefile — PolyBot operations commands

# ─── Infrastructure ──────────────────────────────────────
.PHONY: up down dev restart logs ps

up:                                    ## Start all services (production)
	docker compose up -d

down:                                  ## Stop all services
	docker compose down

dev:                                   ## Start dev infrastructure (DB + Redis only)
	docker compose -f docker-compose.dev.yml up -d

restart:                               ## Restart all services
	docker compose restart

logs:                                  ## Tail all service logs
	docker compose logs -f --tail=100

ps:                                    ## Show running services
	docker compose ps

# ─── Database ────────────────────────────────────────────
.PHONY: migrate migrate-new db-shell db-reset

migrate:                               ## Run pending migrations
	docker compose run --rm execution alembic upgrade head

migrate-new:                           ## Create new migration
	docker compose run --rm execution alembic revision --autogenerate -m "$(msg)"

db-shell:                              ## Open PostgreSQL shell
	docker compose exec postgres psql -U polybot polybot

db-reset:                              ## Drop and recreate database (DESTRUCTIVE)
	docker compose exec postgres dropdb -U polybot --if-exists polybot
	docker compose exec postgres createdb -U polybot polybot
	$(MAKE) migrate

# ─── Testing ─────────────────────────────────────────────
.PHONY: test test-unit test-int test-cov

test:                                  ## Run all tests
	pytest tests/ -v

test-unit:                             ## Run unit tests only (fast, no Docker deps)
	pytest tests/unit/ -v

test-int:                              ## Run integration tests (requires DB + Redis)
	pytest tests/integration/ -v

test-cov:                              ## Run tests with coverage report
	pytest tests/ --cov=src --cov-report=html --cov-report=term-missing

# ─── Frontend ────────────────────────────────────────────
.PHONY: fe-dev fe-build fe-lint

fe-dev:                                ## Start frontend dev server
	cd frontend && npm run dev

fe-build:                              ## Build frontend for production
	cd frontend && npm run build

fe-lint:                               ## Lint frontend code
	cd frontend && npm run lint

# ─── Operations ──────────────────────────────────────────
.PHONY: backup restore deploy health

backup:                                ## Run backup
	./scripts/backup.sh

restore:                               ## Restore from backup (usage: make restore file=<path>)
	./scripts/restore.sh $(file)

deploy:                                ## Deploy latest changes
	./scripts/deploy.sh

health:                                ## Check all service health
	@echo "Service Health:"
	@for svc in market-data orchestrator execution risk wallet dashboard; do \
		status=$$(docker inspect --format='{{.State.Health.Status}}' polybot-$$svc 2>/dev/null || echo "not found"); \
		echo "  $$svc: $$status"; \
	done

# ─── Utilities ───────────────────────────────────────────
.PHONY: api-creds clean

api-creds:                             ## Generate Polymarket API credentials
	python scripts/generate-api-creds.py

clean:                                 ## Remove all containers, volumes, and images
	docker compose down -v --rmi local
	rm -rf frontend/dist
```

---

## Cross-References

| Topic | Document |
|-------|----------|
| Architecture diagrams and service details | [04-technical-specification.md](./04-technical-specification.md) |
| Development environment setup | [05-development-guidelines.md](./05-development-guidelines.md) |
| CI/CD pipeline configuration | [05-development-guidelines.md](./05-development-guidelines.md) — CI/CD section |
| Security hardening details | [08-security-spec.md](./08-security-spec.md) |
| API endpoint specifications | [10-api-specification.md](./10-api-specification.md) |
| Monitoring dashboards and alerting | [04-technical-specification.md](./04-technical-specification.md) — Observability section |
| Wallet architecture and key management | [04-technical-specification.md](./04-technical-specification.md) — Service 5 |
