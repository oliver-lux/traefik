# Traefik

Shared edge proxy: terminates TLS, obtains Let's Encrypt certificates, and
routes to app containers by their Docker labels.

## Setup

```bash
cp .env.example .env
# Generate the dashboard hash into TRAEFIK_DASHBOARD_AUTH, with the $ doubled
# (Compose interpolates .env, so a single $ reads as a variable reference and
# the bcrypt hash silently arrives truncated):
htpasswd -nbB admin 'your-password' | sed 's/[$]/$$/g'

docker compose up -d
```

The stack creates the Docker network `traefik-proxy`. App stacks join it as external,
so **start Traefik first**.

A certificate requires:

- `DOMAIN` in the app's `.env` resolving (A/AAAA) to this host - including
  `www.`, which the app also routes.
- Ports 80 and 443 reachable from the internet (HTTP-01 answers on 80).

## Verifying

```bash
docker compose logs -f traefik            # watch ACME + router setup
docker compose exec traefik traefik healthcheck --ping
curl -I http://<your-domain>              # expect 301 to https
curl -I https://<your-domain>
```

Dashboard: `https://$TRAEFIK_DOMAIN` (basic auth). It is never exposed via
`api.insecure`, so there is no unauthenticated port to forget about.

## Notes

- **Test with the staging CA first** (uncomment `ACME_CA_SERVER` in `.env`) -
  Let's Encrypt allows only 5 duplicate certs/week. Delete
  `letsencrypt/acme.json` before switching to production, or staging certs
  get reused.
- Back up `letsencrypt/acme.json` (0600, created by Traefik), or renewal
  starts from scratch after a rebuild.
- `logs/access.log` is **not** rotated by Traefik - add a logrotate entry with
  `copytruncate` for real traffic.
- The Docker socket is mounted read-only; anything that can read it can
  enumerate containers, so keep the dashboard behind its basic auth.
- TLS floor is 1.2, forward-secret AEAD ciphers only
  (`config/dynamic/tls.yaml`, hot-reloaded).
- `docker compose config` prints the hash with `$$` still doubled - that's
  YAML round-trip escaping, not a bug. Check what the container actually got:
  `docker inspect traefik --format '{{index .Config.Labels "traefik.http.middlewares.dashboard-auth.basicauth.users"}}'`
- App-specific middlewares (compression, www redirect, rate limits) belong on
  the app's own labels - this stack stays app-agnostic.
