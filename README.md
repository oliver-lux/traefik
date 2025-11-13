# Traefik Docker Setup

Simple Traefik v3.0 reverse proxy for local development and production.

## Quick Start

### Development
```bash
cp .env.local .env
docker compose --profile dev up -d
# Dashboard: http://localhost:8080/dashboard/
```

### Production
```bash
cp .env.production.example .env
nano .env

# Set these values:
# - TRAEFIK_DASHBOARD_HOST=traefik.yourdomain.com
# - TRAEFIK_LETSENCRYPT_EMAIL=your-email@example.com
# - TRAEFIK_DASHBOARD_AUTH_USERS=admin:$(htpasswd -nbB admin PASSWORD | cut -d: -f2)

# Ensure DNS points to server and ports 80/443 are open
docker compose up -d
# Dashboard: https://traefik.yourdomain.com/dashboard/
```

## Configuration

| Environment | TLS | Dashboard | Auth | Logging |
|-------------|-----|-----------|------|---------|
| Dev | No | :8080 | None | Debug |
| Prod | Let's Encrypt | Via routing | BasicAuth | Warn |

## Connect Your Services

Add to your service's `docker-compose.yml`:

```yaml
services:
  myapp:
    image: myapp:latest
    networks:
      - traefik-proxy
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.myapp.rule=Host(`myapp.localhost`)"
      - "traefik.http.routers.myapp.entrypoints=web"
      - "traefik.http.services.myapp.loadbalancer.server.port=80"

networks:
  traefik-proxy:
    external: true
```

**Production with HTTPS:**
```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.myapp.rule=Host(`myapp.yourdomain.com`)"
  - "traefik.http.routers.myapp.entrypoints=websecure"
  - "traefik.http.routers.myapp.tls.certresolver=letsencrypt"
  # Optional: HTTP to HTTPS redirect
  - "traefik.http.routers.myapp-http.rule=Host(`myapp.yourdomain.com`)"
  - "traefik.http.routers.myapp-http.entrypoints=web"
  - "traefik.http.routers.myapp-http.middlewares=https-redirect"
```

## Custom Middlewares

Available middlewares defined in `docker-compose.yml`:
- `security-headers` - Security headers (HSTS, frame-deny, XSS filter)
- `dashboard-auth` - BasicAuth for dashboard

Reference in labels: `traefik.http.routers.myapp.middlewares=security-headers`

## Common Commands

```bash
docker compose --profile dev up -d     # Start with dev profile
docker compose logs -f                 # View logs
docker compose down                    # Stop
docker compose restart                 # Restart
docker volume rm traefik_letsencrypt-data  # Delete certificates (when stopped)
```
