# solvara-proxy

Shared Caddy reverse proxy for the Solvara server. This is the single service
that binds ports **80/443** and terminates HTTPS for every app on the host.

## One-time host setup

```bash
docker network create web
```

## Deploy / update

```bash
git pull
# Recreate (NOT reload) so the new Caddyfile inode is picked up:
# git replaces the file with a new inode, and the bind-mounted container keeps
# seeing the old one until it is recreated.
docker compose up -d --force-recreate caddy
```

## Add a new app

1. In the app's own `docker-compose.yml`, attach it to the external `web` network:
   ```yaml
   networks:
     web:
       external: true
   ```
   and reference it here by its `container_name`.
2. Add a site block in `Caddyfile`:
   ```
   your-domain.com {
       reverse_proxy your-container:PORT
   }
   ```
3. Point the domain's DNS A record at this server.
4. Recreate Caddy (`docker compose up -d --force-recreate caddy`). The cert is auto-provisioned.

## What lives where

- `logique-motors` (used-car app) → `logique-test.solvara-tech.com`
- `solvara-frontend` (landing) → `solvara-tech.com`

Other projects must **not** bind 80/443 — only this proxy does.
