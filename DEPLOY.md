# Deploy Invidious on Fly.io

Invidious is a Crystal/Kemal long-running server with a PostgreSQL dependency.
It cannot run on Vercel (no Crystal runtime, no long-running processes).
Fly.io runs the official Docker image directly.

## Prerequisites

- `flyctl` installed: https://fly.io/docs/hands-on/install-flyctl/
- A Fly.io account (`fly auth login`)

## 1. Provision a Postgres cluster

```bash
fly pg create \
  --name invidious-db \
  --region fra \
  --initial-cluster-settings vm=shared-cpu-1x,volumeSize=1
```

Attach it to the app (creates `DATABASE_URL` and wires private networking):

```bash
fly pg attach invidious-db --app invidious
```

## 2. Create the app (do not deploy yet)

```bash
fly launch --no-deploy
```

When prompted, choose "No" for creating a Postgres (we made our own) and let it
use the existing `fly.toml` + `docker/Dockerfile`.

## 3. Set secrets / config

Edit `fly.toml` and replace the placeholders, or (recommended) inject the full
config as a secret so it overrides the file:

```bash
fly secrets set INVIDIOUS_CONFIG='
db:
  dbname: invidious
  user: kemal
  password: <PASSWORD_FROM_pg_attach_OUTPUT>
  host: invidious-db.flycast
  port: 5432
check_tables: true
domain: invidious.fly.dev
https_only: true
hmac_key: <RUN: openssl rand -hex 32>
' --app invidious
```

Notes:
- `host` is the Fly private DNS name of your Postgres app (`<pg-app-name>.flycast`).
- `check_tables: true` auto-creates missing tables/columns on boot, so you do
  NOT need to run `config/sql/*.sql` manually. If you prefer manual init, see
  step 4.
- `hmac_key` must be a random secret; keep it stable across restarts.

## 4. (Optional) Manual DB schema init

Only needed if you don't trust `check_tables`:

```bash
fly ssh proxy --app invidious-db
# in another shell, from repo root:
for f in config/sql/*.sql; do
  psql "postgres://kemal:<password>@localhost:5432/invidious" -f "$f"
done
```

## 5. Deploy

```bash
fly deploy --app invidious
```

First build is slow (compiles Crystal + OpenSSL from source, ~10-15 min).
Subsequent builds reuse the Crystal dependency cache.

## 6. Open

```bash
fly apps open --app invidious
```

## Operational notes

- Keep `min_machines_running = 1` — Invidious is not designed for scale-to-zero
  (DB pool warmup, in-memory caches). `auto_start_machines = true` is fine as a
  safety net.
- For a custom domain: `fly certs add yourdomain.com --app invidious` and set
  `domain:` in the config to match.
- YouTube rate-limits datacenter IPs aggressively. If you hit 429s, consider:
  - a region closer to your users
  - running behind a residential proxy (Invidious supports `http_proxy` config)
  - the Invidious companion (see `config.example.yml`)
- Backups: `fly pg snapshot` for the Postgres volume.
