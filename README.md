# Traefik

Shared edge proxy for the sites on this host. Terminates TLS, obtains
Let's Encrypt certificates, and routes to app containers by their Docker labels.

## Setup

```bash
cp .env.example .env
# Generate the dashboard hash into TRAEFIK_DASHBOARD_AUTH, with the $ doubled:
# Compose interpolates .env values, so a single $ is read as a variable
# reference and the bcrypt hash silently arrives truncated.
htpasswd -nbB admin 'your-password' | sed 's/[$]/$$/g'

docker compose up -d
```

The stack creates the Docker network `proxy`. Every app stack joins it as an
external network, so **start Traefik before the app stacks**.

```

Requirements for a certificate to be issued:

- `DOMAIN` in the app's `.env` resolves (A/AAAA) to this host — including the
  `www.` name, which the app also routes.
- Ports 80 and 443 reach this host from the internet. The HTTP-01 challenge is
  answered on port 80, so it cannot be firewalled off.

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

- **Test with the staging CA first.** Uncomment `ACME_CA_SERVER` in `.env`;
  Let's Encrypt allows only 5 duplicate certificates per week and a
  misconfigured DNS record burns through that quickly. Delete
  `letsencrypt/acme.json` before switching back to production, otherwise the
  staging certificates are reused.
- Certificates live in `letsencrypt/acme.json` (created 0600 by Traefik). Back
  it up, or renewal starts from scratch after a host rebuild.
- Access logs go to `logs/access.log` and are **not** rotated by Traefik. Add a
  logrotate entry with `copytruncate` if this host serves real traffic.
- The Docker socket is mounted read-only. Anything that can read it can enumerate
  containers, so keep the dashboard behind its basic auth.
- TLS floor is 1.2 with forward-secret AEAD ciphers only
  (`config/dynamic/tls.yaml`, hot-reloaded — no restart needed).
- `docker compose config` prints the hash with $$ still doubled — that is its
  YAML round-trip escaping, not a bug. To see what the container really got:
  `docker inspect traefik --format '{{index .Config.Labels "traefik.http.middlewares.dashboard-auth.basicauth.users"}}'`
- App-specific middlewares (compression, www redirect, form rate limits) are
  declared on the app's own labels, not here. This stack stays app-agnostic.
