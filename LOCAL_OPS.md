# Local Operations — Office Mac Self-Hosted Deployment

This file documents the Office Mac deployment of Multica. It is not part of the upstream repo
and will not conflict with `git pull` from `origin`.

---

## Architecture on This Machine

The self-hosted Multica stack runs as **Docker containers** managed by Docker Compose.
The `multica` CLI and daemon are installed via Homebrew and run natively on the host.

| Component | How it runs | Port |
|-----------|------------|------|
| Backend API | Docker container (`multica-backend:dev`) | `100.90.144.113:8090` → container 8080 |
| Frontend | Docker container (`multica-web:dev`) | `100.90.144.113:3000` |
| PostgreSQL | Docker container (`multica-postgres-1`) | internal only |
| Agent daemon | Native macOS process (LaunchAgent) | `localhost:19514` |

Compose project: `/Users/joey/dev/multica`
Compose files: `docker-compose.selfhost.yml` + `docker-compose.selfhost.build.yml` + `docker-compose.selfhost.local.yml`

---

## Critical: Docker Desktop Must Be Running

**The Multica backend goes down whenever Docker Desktop is not running.**

The containers have restart policies, so they come back up automatically the moment Docker
Desktop starts — no manual `docker compose up` needed. But Docker Desktop itself does not
auto-start on macOS by default.

### Symptoms when Docker is down
- `multica issue list` times out or returns "Request timed out"
- `morning-brief` cron fails with `multica issue list --assignee joey (workspace)` error
- Port 8090 is unreachable (connection refused or timeout)

### Fix
```bash
open /Applications/Docker.app
```

Wait ~15 seconds for the daemon to start. The containers come up automatically.
Verify:
```bash
/Applications/Docker.app/Contents/Resources/bin/docker ps
# should show multica-backend-1 and multica-web-1 as "Up"

PATH=/opt/homebrew/bin:/usr/bin:/bin multica issue list
# should return results
```

### Permanent fix — auto-start Docker at login
1. Open Docker Desktop
2. Go to **Settings → General**
3. Enable **Start Docker Desktop when you log in**

This prevents the outage from recurring after reboots.

---

## Common Operations

### Check status
```bash
# Docker containers
/Applications/Docker.app/Contents/Resources/bin/docker ps

# Agent daemon
PATH=/opt/homebrew/bin:/usr/bin:/bin multica daemon status

# Backend health
curl http://100.90.144.113:8090/api/health
```

### Restart the stack (if needed)
```bash
cd /Users/joey/dev/multica
/Applications/Docker.app/Contents/Resources/bin/docker compose \
  -f docker-compose.selfhost.yml \
  -f docker-compose.selfhost.build.yml \
  -f docker-compose.selfhost.local.yml \
  up -d
```

### Restart just the backend
```bash
/Applications/Docker.app/Contents/Resources/bin/docker restart multica-backend-1
```

### View backend logs
```bash
/Applications/Docker.app/Contents/Resources/bin/docker logs multica-backend-1 --tail 50
```

---

## The Desktop App Is Not the Server

`/Applications/Multica.app` is an **Electron GUI client** — it contains only the `multica`
CLI binary and a web renderer. It does **not** run the server. The backend lives entirely
in Docker containers.

You do not need `Multica.app` running for the CLI or agents to work. You only need:
1. Docker Desktop running
2. The Multica containers up
3. The `multica` daemon running (managed by the `dev.worksuite.multica-daemon` LaunchAgent)

---

## Tailscale Binding Note

The containers are bound to the Tailscale IP `100.90.144.113` (hostname
`office-mac.tail96dfb.ts.net`), not `0.0.0.0`. This is intentional — binding to `0.0.0.0`
would bypass the macOS firewall and expose Multica with default secrets.

The `docker-compose.selfhost.local.yml` override handles this binding. If Tailscale
reassigns the IP, update that file and recreate the stack.
