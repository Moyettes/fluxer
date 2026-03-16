# Fluxer Self-Hosting Plan

## Overview

This document outlines a comprehensive plan for self-hosting a Fluxer instance. Fluxer ships a **unified Docker container** (`fluxer_server`) that bundles the API server, WebSocket gateway, media proxy, admin dashboard, and web client into a single image — making self-hosting straightforward.

---

## 1. Prerequisites

### Hardware (minimum)
| Component | Minimum | Recommended |
|-----------|---------|-------------|
| CPU | 2 cores | 4+ cores |
| RAM | 2 GB | 4+ GB |
| Disk | 10 GB | 40+ GB (depends on media storage) |
| Network | 100 Mbps | 1 Gbps (especially if using voice/video) |

### Software
- **Docker** 24+ and **Docker Compose** v2
- A domain name with DNS control (e.g. `chat.example.com`)
- A reverse proxy with TLS termination (Caddy, nginx, or Traefik)
- (Optional) S3-compatible storage for large-scale media

### Ports to open
| Port | Protocol | Service | Required? |
|------|----------|---------|-----------|
| 80 | TCP | HTTP → HTTPS redirect | Yes |
| 443 | TCP | HTTPS (reverse proxy) | Yes |
| 3478 | UDP | TURN (voice/video) | Only for voice |
| 7881 | TCP | ICE-TCP fallback | Only for voice |
| 50000-50100 | UDP | RTP media | Only for voice |

---

## 2. Architecture

```
                    ┌─────────────────┐
                    │  Reverse Proxy  │
                    │  (Caddy/nginx)  │
                    │  :443 TLS       │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  fluxer_server  │   ← Unified container
                    │  :8080          │
                    │                 │
                    │  ┌─ API Server  │
                    │  ├─ Gateway (WS)│
                    │  ├─ Media Proxy │
                    │  ├─ Admin Panel │
                    │  ├─ Web Client  │
                    │  └─ Marketing   │
                    └──┬──────────┬───┘
                       │          │
              ┌────────▼──┐  ┌───▼──────────┐
              │  Valkey    │  │  Meilisearch  │  (optional)
              │  :6379     │  │  :7700        │
              └───────────┘  └──────────────┘

              ┌─────────────┐
              │  LiveKit    │  (optional, for voice/video)
              │  :7880      │
              └─────────────┘
```

### Core services (required)
| Service | Image | Purpose |
|---------|-------|---------|
| `fluxer_server` | `ghcr.io/fluxerapp/fluxer-server:stable` | All backend + frontend in one container |
| `valkey` | `valkey/valkey:8.0.6-alpine` | In-memory cache, pub/sub, session store |

### Optional services
| Service | Image | Purpose |
|---------|-------|---------|
| `meilisearch` | `getmeili/meilisearch:v1.14` | Full-text search (messages, users, communities) |
| `elasticsearch` | `elasticsearch:8.19.11` | Alternative search backend (heavier, more features) |
| `livekit` | `livekit/livekit-server:v1.9.11` | Voice & video calls, screen sharing |

### Database options
| Backend | Best for | Notes |
|---------|----------|-------|
| **SQLite** (default) | Small-medium instances (<1000 users) | Zero config, single file, bundled |
| **Cassandra** | Large/distributed instances | Requires separate cluster, migration scripts provided |

---

## 3. Step-by-Step Deployment

### Step 1: Prepare the host

```bash
# Create a directory for your Fluxer instance
mkdir -p /opt/fluxer/config
cd /opt/fluxer

# Download the compose file
curl -O https://raw.githubusercontent.com/fluxerapp/fluxer/canary/compose.yaml

# Download the production config template
curl -o config/config.json \
  https://raw.githubusercontent.com/fluxerapp/fluxer/canary/config/config.production.template.json
```

### Step 2: Generate secrets

Generate all required secrets before editing config:

```bash
# Generate 64-char hex secrets (run once per secret needed)
openssl rand -hex 32

# Generate VAPID keys for web push notifications
npx web-push generate-vapid-keys
```

You'll need secrets for:
- `services.media_proxy.secret_key`
- `services.admin.secret_key_base`
- `services.admin.oauth_client_secret`
- `services.marketing.secret_key_base`
- `services.gateway.admin_reload_secret`
- `auth.sudo_mode_secret`
- `auth.connection_initiation_secret`
- `auth.vapid.public_key` / `auth.vapid.private_key`

### Step 3: Configure `config/config.json`

Edit the config file with your domain and secrets:

```jsonc
{
  "env": "production",
  "domain": {
    "base_domain": "chat.example.com",   // ← Your domain
    "public_scheme": "https",
    "public_port": 443
  },
  "database": {
    "backend": "sqlite",
    "sqlite_path": "./data/fluxer.db"    // ← Persisted via Docker volume
  },
  "internal": {
    "kv": "redis://valkey:6379/0",
    "kv_mode": "standalone"
  },
  "services": {
    "server": { "port": 8080, "host": "0.0.0.0" },
    "media_proxy": { "secret_key": "<generated>" },
    "admin": {
      "secret_key_base": "<generated>",
      "oauth_client_secret": "<generated>"
    },
    "marketing": {
      "enabled": false,                  // ← Disable if not needed
      "secret_key_base": "<generated>"
    },
    "gateway": {
      "port": 8082,
      "admin_reload_secret": "<generated>",
      "media_proxy_endpoint": "http://127.0.0.1:8080/media"
    },
    "nats": {
      "core_url": "nats://nats:4222",
      "jetstream_url": "nats://nats:4222",
      "auth_token": "<generated>"
    }
  },
  "auth": {
    "sudo_mode_secret": "<generated>",
    "connection_initiation_secret": "<generated>",
    "vapid": {
      "public_key": "<generated>",
      "private_key": "<generated>"
    }
  },
  "integrations": {
    "search": {
      "engine": "meilisearch",
      "url": "http://meilisearch:7700",
      "api_key": "<your-meilisearch-master-key>"
    }
  }
}
```

### Step 4: Set up the reverse proxy

**Option A: Caddy (recommended — automatic TLS)**

```
# /opt/fluxer/Caddyfile
chat.example.com {
    reverse_proxy fluxer_server:8080
}
```

**Option B: nginx**

```nginx
server {
    listen 443 ssl http2;
    server_name chat.example.com;

    ssl_certificate     /etc/letsencrypt/live/chat.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/chat.example.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

> **Important:** WebSocket support (`Upgrade` / `Connection` headers) is required for the real-time gateway.

### Step 5: Start the services

```bash
cd /opt/fluxer

# Start core services (Fluxer + Valkey)
docker compose up -d

# With search enabled
MEILI_MASTER_KEY=<your-key> docker compose --profile search up -d

# With voice/video enabled (also configure livekit.yaml first)
docker compose --profile voice up -d

# With everything
MEILI_MASTER_KEY=<your-key> docker compose --profile search --profile voice up -d
```

### Step 6: Verify the deployment

```bash
# Check service health
docker compose ps

# Check Fluxer health endpoint
curl -s http://localhost:8080/_health

# Check logs
docker compose logs -f fluxer_server
```

Visit `https://chat.example.com` — you should see the Fluxer web client. Register the first account; it will become the instance owner.

---

## 4. Optional: Voice & Video Setup (LiveKit)

### Create `config/livekit.yaml`

```yaml
port: 7880

keys:
  '<api-key>': '<api-secret>'

rtc:
  tcp_port: 7881

turn:
  enabled: true
  udp_port: 3478

room:
  auto_create: true
  max_participants: 100
  empty_timeout: 300
```

### Add voice config to `config.json`

```jsonc
{
  "integrations": {
    "voice": {
      "enabled": true,
      "api_key": "<same-api-key-as-livekit.yaml>",
      "api_secret": "<same-api-secret-as-livekit.yaml>",
      "url": "ws://livekit:7880",
      "webhook_url": "http://fluxer_server:8080/api/webhooks/livekit",
      "default_region": {
        "id": "default",
        "name": "Default",
        "emoji": "🌐",
        "latitude": 0.0,
        "longitude": 0.0
      }
    }
  }
}
```

### Firewall rules for voice

```bash
# TURN server (required for NAT traversal)
ufw allow 3478/udp

# ICE-TCP fallback
ufw allow 7881/tcp

# RTP media ports
ufw allow 50000:50100/udp
```

---

## 5. Optional: Email (SMTP)

For email verification, password resets, and notifications:

```jsonc
{
  "integrations": {
    "email": {
      "enabled": true,
      "provider": "smtp",
      "from_email": "noreply@chat.example.com",
      "smtp": {
        "host": "smtp.example.com",
        "port": 587,
        "username": "your-smtp-user",
        "password": "your-smtp-password",
        "secure": true
      }
    }
  }
}
```

---

## 6. Optional: S3 Storage

By default, Fluxer stores media on the local filesystem. For production, you may want S3-compatible storage (AWS S3, MinIO, Cloudflare R2, Backblaze B2):

```jsonc
{
  "s3": {
    "access_key_id": "YOUR_ACCESS_KEY",
    "secret_access_key": "YOUR_SECRET_KEY",
    "endpoint": "https://s3.us-east-1.amazonaws.com",
    "bucket": "fluxer-media",
    "region": "us-east-1"
  }
}
```

For a self-hosted MinIO instance, add it to your compose file:

```yaml
  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    volumes:
      - minio_data:/data
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
```

---

## 7. Backups

### SQLite backup

```bash
# Stop writes briefly and copy the database file
docker compose exec fluxer_server sqlite3 /usr/src/app/data/db/fluxer.db ".backup '/tmp/backup.db'"
docker compose cp fluxer_server:/tmp/backup.db ./backups/fluxer-$(date +%Y%m%d).db
```

### Valkey backup

```bash
# Valkey is configured with AOF persistence; copy the dump
docker compose exec valkey valkey-cli BGSAVE
docker compose cp valkey:/data/dump.rdb ./backups/valkey-$(date +%Y%m%d).rdb
```

### Media files

```bash
# If using local storage, back up the Docker volume
docker run --rm -v fluxer_fluxer_data:/data -v $(pwd)/backups:/backup \
  alpine tar czf /backup/media-$(date +%Y%m%d).tar.gz -C /data .
```

### Automate with cron

```bash
# /etc/cron.d/fluxer-backup
0 3 * * * root /opt/fluxer/backup.sh >> /var/log/fluxer-backup.log 2>&1
```

---

## 8. Upgrading

```bash
cd /opt/fluxer

# Pull latest images
docker compose pull

# Restart with new images
docker compose up -d

# Verify health
docker compose ps
curl -s http://localhost:8080/_health
```

> **Tip:** Pin to `stable` for production. Use `nightly` only for testing new features.

### Before upgrading
1. **Back up** your database and config
2. **Read** the release notes for breaking changes
3. **Test** on a staging instance first if possible

---

## 9. Monitoring & Observability

### Health checks

The container has a built-in healthcheck at `/_health`. Docker will automatically restart unhealthy containers with `restart: unless-stopped`.

### Logs

```bash
# Follow all logs
docker compose logs -f

# Follow specific service
docker compose logs -f fluxer_server

# Last 100 lines
docker compose logs --tail 100 fluxer_server
```

### OpenTelemetry (advanced)

Fluxer supports full OpenTelemetry instrumentation. Add to your config:

```jsonc
{
  "telemetry": {
    "enabled": true,
    "service_name": "fluxer",
    "exporter": {
      "endpoint": "http://otel-collector:4318"
    }
  }
}
```

Pair with Grafana + Tempo/Loki/Prometheus or SignOZ for full observability.

---

## 10. Security Hardening

### Checklist

- [ ] Use HTTPS everywhere (TLS 1.2+)
- [ ] Generate unique, strong secrets for every config field
- [ ] Don't expose Valkey, Meilisearch, or LiveKit ports publicly
- [ ] Set up a firewall — only expose 80, 443, and voice ports if needed
- [ ] Enable hCaptcha or Cloudflare Turnstile to prevent spam registrations
- [ ] Keep images updated (`docker compose pull` regularly)
- [ ] Back up database and media on a schedule
- [ ] Consider running Fluxer behind Cloudflare or similar DDoS protection
- [ ] Use Docker's `read_only` filesystem where possible
- [ ] Set resource limits in compose (CPU/memory)

### Rate limiting

Rate limiting is enabled by default in production. No action needed.

### CAPTCHA

```jsonc
{
  "integrations": {
    "captcha": {
      "provider": "hcaptcha",
      "site_key": "YOUR_SITE_KEY",
      "secret_key": "YOUR_SECRET_KEY"
    }
  }
}
```

---

## 11. Scaling Considerations

### Small instance (<500 users)
- SQLite + Valkey + single `fluxer_server` container
- Meilisearch optional
- 2 CPU / 2 GB RAM sufficient

### Medium instance (500–5000 users)
- SQLite or Cassandra
- Meilisearch recommended
- LiveKit for voice
- 4 CPU / 4-8 GB RAM
- S3 storage for media

### Large instance (5000+ users)
- Cassandra (distributed)
- Elasticsearch for search
- Multiple fluxer_server replicas behind a load balancer
- Dedicated LiveKit cluster
- S3 storage (required)
- 8+ CPU / 16+ GB RAM
- Consider NATS for inter-service communication

---

## Quick Reference

| Task | Command |
|------|---------|
| Start all services | `docker compose up -d` |
| Stop all services | `docker compose down` |
| View logs | `docker compose logs -f` |
| Check health | `curl http://localhost:8080/_health` |
| Update to latest | `docker compose pull && docker compose up -d` |
| Backup database | See [Backups](#7-backups) section |
| Restart a service | `docker compose restart fluxer_server` |
