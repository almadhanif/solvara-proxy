# Solvara server — ops runbook

Single source of truth for the `82.197.68.92` server. Read this before touching
deployment, routing, or DNS.

## Server access

- **SSH:** `ssh almadhani@82.197.68.92` (hostname `vmi3449545`)
- **sudo:** the `almadhani` user has **no passwordless sudo** — host-level
  firewall/iptables changes need the sudo password (do them interactively).

## Directory layout

All apps live under `/opt/apps/solvara/`, each its own git clone + docker
compose project:

| Path | Repo | Container | Exposes |
|---|---|---|---|
| `/opt/apps/solvara/proxy` | `almadhanif/solvara-proxy` | `solvara-caddy` | `80/443` (the only thing on these ports) |
| `/opt/apps/solvara/solvara-frontend` | `almadhanif/solvara-frontend` | `solvara-frontend` | `3001` (internal) |
| `/opt/apps/solvara/logique-test` | `almadhanif/logique-test` | `logique-motors` | `3000` (internal) |

## Architecture (one universal proxy)

```
internet :80/:443 ──▶ solvara-caddy  (this repo — binds 80/443, terminates TLS)
                          │   external docker network "web"
                          ├── solvara-tech.com / www.solvara-tech.com ─▶ solvara-frontend:3001
                          └── logique-test.solvara-tech.com            ─▶ logique-motors:3000
```

- **Only `solvara-caddy` binds 80/443.** No app project binds host ports.
- Apps talk to the proxy over the **external docker network `web`** and are
  referenced by **container name**.
- Caddy **auto-provisions and renews Let's Encrypt** certs per hostname once
  its DNS A record points here and 80/443 are reachable.
- The `web` network is created once on the host: `docker network create web`.

## Domains & DNS

All domains are under `solvara-tech.com`. A records → `82.197.68.92`:

| Domain | App |
|---|---|
| `solvara-tech.com` (apex) | solvara-frontend |
| `www.solvara-tech.com` | solvara-frontend |
| `logique-test.solvara-tech.com` | logique-motors (used-car app) |

Manage DNS at your registrar. After changing an A record, Caddy picks up the
cert automatically within ~1–2 min (it retries on a loop).

## Firewall

Only **80 and 443** are open inbound (provider security group + the host). Any
other port (e.g. a raw app port) is **not reachable from the internet** — and
that's intentional. Always front an app through Caddy rather than publishing its
port.

## Add a new app

1. **App compose** — in the new app's `docker-compose.yml`, attach it to `web`
   and give it a `container_name`. Do **not** publish a host port; use `expose`:
   ```yaml
   services:
     myapp:
       container_name: myapp            # the proxy references THIS
       # ...
       expose: ["8080"]
       networks: [web]
   networks:
     web:
       external: true
   ```
2. **Route it** — add a block in this repo's `Caddyfile`:
   ```
   myapp.solvara-tech.com {
       reverse_proxy myapp:8080
   }
   ```
3. **DNS** — add an A record `myapp.solvara-tech.com` → `82.197.68.92`.
4. **Deploy proxy change** — on the server:
   ```bash
   cd /opt/apps/solvara/proxy && git pull
   docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
   ```
   The cert is provisioned automatically.

## Deploy / update an app

Apps are independent — updating one never affects the others (only the proxy
touches both).

```bash
ssh almadhani@82.197.68.92
cd /opt/apps/solvara/<app>          # e.g. logique-test
git pull
docker compose up -d --build        # rebuild image, recreate container, run entrypoint
```

- `logique-motors` runs `prisma migrate deploy` + first-run seed on start; its
  SQLite DB lives on the `db-data` volume (survives redeploys).
- Secrets (DB, API keys, passwords) live in each project's **`.env`** (gitignored,
  `chmod 600`) — never in the image.

## Update the proxy itself

```bash
cd /opt/apps/solvara/proxy
git pull
docker compose up -d                 # recreate caddy (e.g. after compose change)
# or, after editing ONLY the Caddyfile, zero-downtime reload:
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

Before applying a Caddyfile change, validate it:
```bash
docker run --rm -v "$PWD/Caddyfile:/etc/caddy/Caddyfile:ro" \
  caddy:2-alpine caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
```

## Logs & troubleshooting

```bash
docker compose -f /opt/apps/solvara/proxy/docker-compose.yml logs -f caddy   # proxy + cert
docker logs logique-motors --tail 50 -f                                       # car app
docker logs solvara-frontend --tail 50 -f                                     # frontend
docker ps                                                                     # what's running
docker network inspect web --format '{{range .Containers}}{{.Name}} {{end}}'  # who's on web
```

- **502 from a domain** → the target container isn't on `web`, or isn't
  listening on the expected port. Check `docker network inspect web` and that
  the app's compose declares `networks: [web]` with `external: true`.
- **Cert not provisioning** → DNS for that host isn't pointing at the server
  yet, or 80/443 got blocked. Check `docker logs solvara-caddy | grep -i acme`.
- **Port conflict on 80/443 at startup** → another container is still bound;
  only `solvara-caddy` may bind 80/443.

## Rollback

Each project is git-tracked, so revert + redeploy:
```bash
cd /opt/apps/solvara/<app>
git log --oneline -5          # find the last good commit
git checkout <sha> -- .       # or git revert
docker compose up -d --build
```

## Port-map (quick reference)

| Container | Internal port | Host port | Reachable as |
|---|---|---|---|
| `solvara-caddy` | 80, 443 | 80, 443 | (public entry point) |
| `solvara-frontend` | 3001 | — | `solvara-frontend:3001` on `web` |
| `logique-motors` | 3000 | — | `logique-motors:3000` on `web` |
