---
name: gordon
description: Use when deploying containers, managing routes, configuring services, or operating any application with the Gordon CLI. Use when the user mentions gordon, push-to-deploy, or self-hosted container deployment.
---

# Gordon - Deployment Guide

Gordon is a self-hosted container deployment platform. Private Docker registry + push-to-deploy + HTTP reverse proxy in a single binary.

**Mental model:** Build locally, push an image to Gordon's registry, Gordon deploys it, routes traffic, and handles zero-downtime updates.

> **Current CLI note:** Use `gordon version` to check the installed version.

> **First step:** Before running any Gordon command, check if `~/.config/gordon/remotes.toml` exists and has a remote configured. If not, the user needs to set one up with `gordon remotes add` before any remote operation will work.

## How Deployment Works

1. Push image to Gordon's registry (`gordon push` or `docker push`)
2. Gordon starts a new container from the image
3. Gordon waits for readiness (healthcheck, HTTP probe, or delay)
4. Gordon switches proxy traffic to the new container
5. Gordon drains in-flight requests and stops the old container

Designed for zero-downtime when readiness succeeds. Push only triggers deploy if a route matches the pushed image.

## Command Locality

| Command | Local | Remote | Notes |
|---------|:-----:|:------:|-------|
| `serve` | Y | - | Start server |
| `auth token generate/list/revoke` | Y | - | Token management |
| `auth login` | - | Y | Authenticate with remote |
| `bootstrap` | Y | Y | First deploy from Dockerfile |
| `routes list/show/add/remove/deploy` | Y | Y | Route management |
| `config show` | Y | Y | Server configuration |
| `networks list` | Y | Y | Gordon-managed networks |
| `attachments list/add/remove` | Y | Y | Service dependencies |
| `attachments push [name]` | - | Y | Build and push attachment images |
| `secrets list/set/remove` | Y | Y | Environment/secrets |
| `deploy <domain>` | Y | Y | Trigger deploy |
| `restart <domain>` | Y | Y | Restart container |
| `status` | Y | Y | Show server/container status |
| `logs [domain]` | Y | Y | View logs (remote needs file logging enabled on server) |
| `backups list/run/detect/status` | Y | Y | Database backups |
| `push [image]` | - | Y | Build, push, deploy |
| `version` | Y | Y | Show CLI version |
| `rollback <domain>` | - | Y | Roll back to previous tag |
| `images list/prune/tags` | - | Y | Registry image management |
| `reload` | Y | - | Hot-reload config (SIGUSR1) |
| `remotes add/remove/use/list` | Y | - | Manage saved remotes |

## First-Time Setup

### 1. Install
```bash
curl -fsSL https://gordon.bnema.dev/install | bash
```

### 2. Initialize config
```bash
gordon serve    # Generates default config, then Ctrl+C
```

### 3. Configure `~/.config/gordon/gordon.toml`
```toml
[server]
port = 8080
registry_port = 5000
gordon_domain = "gordon.mydomain.com"

[auth]
enabled = true
secrets_backend = "pass"
token_secret = "gordon/auth/token_secret"

[network_isolation]
enabled = true

[routes]
"app.mydomain.com" = "myapp:latest"
```

### 4. DNS
Point `gordon.mydomain.com` and app domains to the server IP. Wildcard recommended.

### 5. Generate tokens
```bash
# Deploy token (push/pull only)
gordon auth token generate --subject deploy --scopes push,pull --expiry 0

# Admin token (full remote CLI access)
gordon auth token generate --subject admin --scopes "push,pull,admin:*:*" --expiry 0
```

### 6. Save remote on local machine
```bash
gordon remotes add prod https://gordon.mydomain.com --token-env PROD_TOKEN
gordon remotes use prod
```

## Bootstrap (First Deploy)

For first-time deployments when no image exists in the registry yet:

```bash
gordon bootstrap app.example.com myapp:latest
gordon push app.example.com --build --no-confirm
```

First step creates the route and registers the image name. Second step builds, pushes, and deploys. Skips if route+image already exist (idempotent).

## Deploying

### Option A: gordon push (recommended)
```bash
gordon push myapp --build --no-confirm
```
Builds locally, pushes to registry, triggers deploy. Takes an **image name**, not a domain.

Recent releases also add a Gordon-native upload path for large image pushes. If the registry TLS chain is not trusted locally, `--insecure` or an insecure remote config may be required.

Flags:
- `--build` - build with docker buildx first
- `--platform linux/arm64` - cross-platform build
- `-f Dockerfile.prod` - custom Dockerfile
- `--build-arg KEY=VALUE` - build arguments (repeatable)
- `--tag v1.2.0` - override version tag
- `--no-confirm` - skip deploy confirmation
- `--no-deploy` - push only, don't deploy

### Option B: Standard docker push
```bash
docker build -t gordon.mydomain.com/myapp:v1.0.0 .
docker push gordon.mydomain.com/myapp:v1.0.0
```
Auto-deploys if a route matches the pushed image.

### Option C: Config change + reload
Edit `gordon.toml` routes section, then save (auto-reloads) or run `gordon reload`.

## Routes

```bash
gordon routes list
gordon routes list --json
gordon routes show app.example.com         # Single route detail + health
gordon routes show app.example.com --json
gordon routes add app.example.com myapp:v2
gordon routes remove app.example.com
gordon routes deploy app.example.com       # Redeploy
gordon deploy app.example.com              # Also works (top-level)
```

Config:
```toml
[routes]
"app.example.com" = "myapp:latest"         # HTTPS (TLS terminated upstream)
"http://internal.local" = "myapi:latest"   # HTTP-only
```

Gordon also supports native TLS (`server.tls_enabled = true`) if not using an upstream proxy.

## Attachments (Service Dependencies)

**Requires** `network_isolation.enabled = true`.

```bash
gordon attachments add app.example.com postgres:18
gordon attachments add app.example.com redis:latest
gordon attachments list
gordon attachments remove app.example.com postgres:18
```

Config:
```toml
[attachments]
"app.example.com" = ["postgres:18", "redis:latest"]
```

Services reachable **by name**: `postgres:5432`, `redis:6379`.

### Network groups (shared services)
```toml
[network_groups]
"backend" = ["app.example.com", "api.example.com"]

[attachments]
"backend" = ["postgres:18"]
```

### Attachment secrets
```bash
gordon secrets set app.example.com --attachment postgres POSTGRES_PASSWORD "secret"
```

### Pushing attachment images
```bash
gordon attachments push pitlane-pgsql --build --tag v18 -f infra/postgres/Dockerfile
```

Use this when an attachment image must exist in Gordon's registry before deploy.

## Secrets and Environment Variables

### Managing secrets via CLI
```bash
gordon secrets set app.example.com KEY=value
gordon secrets set app.example.com DB_URL="postgres://..." API_KEY="sk-..."
gordon secrets list app.example.com
gordon secrets remove app.example.com API_KEY
```

### Backend behavior

**`pass`** (recommended for production): Secrets stored in GPG-encrypted pass store at `gordon/env/<domain>/<KEY>`. Existing `.env` files are migrated on startup. Use `gordon secrets set/remove` to manage.

**`sops`**: Env files remain source of truth. Use `${sops:file.yaml:key.path}` syntax.

**`unsafe`**: Plain text `.env` files. Development only.

Env file location configured via `[env] dir` (default `~/.gordon/env/`). Filename pattern: dots replaced with underscores + `.env` (e.g., `app.example.com` -> `app_example_com.env`).

Current releases debounce route or attachment secret changes into an automatic redeploy, and redundant deploy checks now also compare env drift via an env hash.

## Health Checks and Readiness

Dockerfile labels:
```dockerfile
LABEL gordon.health="/api/health"
LABEL gordon.port=3000
HEALTHCHECK --interval=10s --timeout=3s CMD curl -f http://localhost:3000/api/health || exit 1
```

`gordon.health` drives Gordon's own HTTP probe. Docker `HEALTHCHECK` is a separate signal. Both can coexist.

**Readiness modes** (`deploy.readiness_mode`):
- `auto` (default) - Docker healthcheck if present, else falls back to `readiness_delay`. Max wait: `health_timeout` (90s).
- `docker-health` - Requires Docker HEALTHCHECK. Fails deploy if absent.
- `delay` - Fixed `readiness_delay` (default 5s), no probing.

**Drain modes** (`deploy.drain_mode`):
- `auto` (default) - Wait for in-flight requests, fall back to `drain_delay`. Max: `drain_timeout` (30s).
- `inflight` - Strict wait for zero in-flight requests.
- `delay` - Fixed `drain_delay` (default 2s).

## Status, Logs, Rollback, Restart

```bash
gordon status                              # Server info + container states
gordon logs                                # Server process logs
gordon logs app.example.com -f -n 100      # Follow container logs
gordon rollback app.example.com            # Interactive tag selection
gordon rollback app.example.com --tag v1.0 # Specific tag
gordon rollback list app.example.com       # List available tags
gordon restart app.example.com             # Restart container
gordon restart app.example.com --with-attachments  # Remote mode only
```

## Image Management

```bash
gordon images list
gordon images list --json
gordon images tags myapp                   # List registry tags for a repo
gordon images tags myapp --json
gordon images prune --keep-releases 3      # Keep last 3 tags per repo
gordon images prune --dangling             # Remove dangling images
gordon images prune --registry             # Prune registry images
gordon images prune --dry-run              # Preview without deleting
gordon images prune --no-confirm           # Skip confirmation
```

Auto-prune config:
```toml
[images.prune]
enabled = true
schedule = "daily"
keep_last = 3
```

## Config and Networks

```bash
gordon config show                         # Full server config snapshot
gordon config show --json
gordon networks list                       # Gordon-managed Docker networks
gordon networks list --json
```

## Database Backups

```toml
[backups]
enabled = true
schedule = "daily"
storage_dir = "~/.gordon/backups"
[backups.retention]
daily = 7
weekly = 4
```

```bash
gordon backups detect app.example.com      # Find supported databases
gordon backups run app.example.com         # Trigger backup
gordon backups run app.example.com --db mydb
gordon backups list [domain]               # List backups
gordon backups status                      # Backup health
```

## Auth and Tokens

```bash
gordon auth token generate --subject ci-bot --scopes push,pull --expiry 0
gordon auth token generate --subject admin --scopes "push,pull,admin:*:*" --expiry 30d
gordon auth token list
gordon auth token revoke <id> [--all]
gordon auth password hash
gordon auth login --remote prod
```

**Scopes:** `push`, `pull`, `admin:*:*`, `admin:routes:read`, `admin:routes:write`, `admin:config:read`, `admin:config:write`, `admin:status:read`, `admin:secrets:read`, `admin:secrets:write`.

## Remote Management

```bash
gordon remotes add prod https://gordon.mydomain.com --token-env PROD_TOKEN
gordon remotes add staging https://staging.mydomain.com --token $TOKEN [--insecure]
gordon remotes list
gordon remotes use prod
gordon remotes set-token prod $NEW_TOKEN
gordon remotes remove prod [--force]
```

Override per-command:
```bash
gordon routes list --remote https://gordon.mydomain.com --token $TOKEN
# Or via env: GORDON_REMOTE, GORDON_TOKEN
```

## CI/CD (GitHub Actions)

```bash
# Generate CI token on server
gordon auth token generate --subject github-actions --scopes push,pull --expiry 0
```

```yaml
name: Deploy to Gordon
on:
  push:
    tags: ['v*']
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: bnema/gordon/.github/actions/deploy@main
        with:
          registry: ${{ secrets.GORDON_REGISTRY }}
          username: ${{ secrets.GORDON_USERNAME }}
          password: ${{ secrets.GORDON_TOKEN }}
          # Optional: image, tag, dockerfile, context, platforms, build-args, push-latest
```

## Configuration Reference

Config: `~/.config/gordon/gordon.toml`. Env overrides: `GORDON_SECTION_KEY=value`.

**Hot-reloaded:** routes, attachments, network_groups, logging level, backups, prune schedules.
**Requires restart:** server.port, registry_port, data_dir, auth settings, deploy timing.

### Key Defaults
| Setting | Default | Note |
|---------|---------|------|
| `server.port` | 80 | |
| `server.registry_port` | 5000 | |
| `server.tls_enabled` | false | Set true for native TLS |
| `auth.secrets_backend` | `unsafe` | **Use `pass` in production** |
| `deploy.pull_policy` | `if-tag-changed` | |
| `deploy.readiness_mode` | `auto` | |
| `deploy.health_timeout` | 90s | |
| `deploy.readiness_delay` | 5s | |
| `deploy.drain_mode` | `auto` | |
| `deploy.drain_timeout` | 30s | |
| `deploy.drain_delay` | 2s | |
| `network_isolation.enabled` | false | **Enable for attachments** |

## Dockerfile Labels

```dockerfile
LABEL gordon.port=3000                     # HTTP port to proxy to
LABEL gordon.health="/healthz"             # Health check endpoint
LABEL gordon.domains="app.example.com"     # Auto-route (needs auto_route.enabled)
```

## Common Patterns

### Multi-service
```toml
[routes]
"app.example.com" = "app:latest"
"api.example.com" = "api:latest"

[network_groups]
"core" = ["app.example.com", "api.example.com"]

[attachments]
"core" = ["postgres:18", "redis:latest"]
```

### Behind Cloudflare/nginx
```toml
[server]
port = 8080
```

### Native TLS
```toml
[server]
tls_enabled = true
tls_port = 443
```

## Troubleshooting

- **Container not starting?** `gordon logs <domain>` for container output.
- **Attachments not reachable?** `network_isolation.enabled` must be `true`.
- **Route not working?** Check DNS + `gordon routes list`.
- **Push didn't deploy?** Push only deploys if a route matches the image. Check `gordon routes list`.
- **Large or failing pushes?** Recent releases can use Gordon's native image upload path; if local trust is the issue, use `--insecure` or an insecure remote config.
- **Auth failing?** Regenerate token with correct scopes. Admin commands need `admin:*:*`.
- **Config not applied?** Some settings need restart. Try `gordon reload` first.
- **Disk full?** `gordon images prune --keep-releases 3 --registry --dangling`.
- **Remote logs empty?** File logging must be enabled on the server (`[logging.file] enabled = true`).
- **Secrets not working with pass?** Use `gordon secrets set`, not manual `.env` files.

## Server Signals

- `SIGTERM`/`SIGINT` - Graceful shutdown
- `SIGUSR1` - Reload config (`gordon reload`)
- `SIGUSR2` - Manual deploy (`gordon deploy` in local mode)
