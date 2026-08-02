# Proxmox Manager (Docker deployment)

Deploy **Proxmox Manager** with Docker Compose using published images — **no application source code required**.

| Image | Docker Hub |
|-------|------------|
| Backend | [`dridilotfi/proxmox-manager-backend`](https://hub.docker.com/r/dridilotfi/proxmox-manager-backend) |
| Frontend | [`dridilotfi/proxmox-manager-frontend`](https://hub.docker.com/r/dridilotfi/proxmox-manager-frontend) |

This repository only contains:

- `docker-compose.yml` — Postgres, Redis, backend, and frontend
- `.env.example` — configuration template
- this guide

## What you get

A web UI to manage **Proxmox VE (PVE)** and **Proxmox Backup Server (PBS)**:

- Multi-datacenter connections (password or API token)
- VM list and power actions (start / stop / restart)
- PBS datastores, jobs, and task monitoring
- Local + LDAP/Active Directory login, MFA (TOTP), and roles (Admin / Operator / Viewer)
- Activity logs with export

## Prerequisites

- [Docker Engine](https://docs.docker.com/engine/install/) **or** [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- Docker Compose **v2** (`docker compose version`)
- Free host ports (defaults): **3000** (UI), **5000** (API)
- Outbound access to Docker Hub to pull images
- Network reachability from this host to your Proxmox API (usually TCP **8006**)

## Quick start

### 1. Clone this repository

```bash
git clone https://github.com/lotfidridii/proxmox.manager.git
cd proxmox.manager
```

> Rename the repo as you like when you create it on GitHub. Update the clone URL accordingly.

### 2. Create your environment file

```bash
cp .env.example .env
```

Windows (PowerShell):

```powershell
copy .env.example .env
```

Edit `.env` and set **real secrets** (never leave `change-me-*` values):

```bash
# Linux / macOS / Git Bash
openssl rand -hex 32   # use for JWT_SECRET
openssl rand -hex 32   # use for ENCRYPTION_KEY
openssl rand -hex 32   # use for SODIUM_SECRET_KEY
```

Also set a strong `POSTGRES_PASSWORD` and matching `DATABASE_URL`.

**Important:** characters like `@ : / # ? &` in the DB password break the connection string unless URL-encoded in `DATABASE_URL` (e.g. `@` → `%40`). Simplest: use a password without those characters, or set:

```env
POSTGRES_PASSWORD=Pr0xm0x@Pp
DATABASE_URL=postgresql://proxmox-manager:Pr0xm0x%40Pp@db:5432/proxmox-manager?schema=public
```

| Variable | Required | Description |
|----------|----------|-------------|
| `POSTGRES_DB` | yes | Database name |
| `POSTGRES_USER` | yes | Database user |
| `POSTGRES_PASSWORD` | yes | Database password (same value Postgres was initialized with) |
| `DATABASE_URL` | yes | Full URL for Prisma; encode special chars in the password |
| `FRONTEND_URL` | yes | Public UI URL (CORS), e.g. `http://localhost:3000` |
| `NEXT_PUBLIC_API_URL` | recommended | Browser-facing API base, e.g. `http://localhost:5000/api` |
| `JWT_SECRET` | yes | Signs auth tokens |
| `ENCRYPTION_KEY` | yes | 64-char hex (32 bytes) for encrypting stored secrets |
| `SODIUM_SECRET_KEY` | yes | 64-char hex string for crypto helpers |
| `FRONTEND_PORT` | no | Host port for the UI (default `3000`) |
| `BACKEND_PORT` | no | Host port for the API (default `5000`) |
| `IMAGE_TAG` | no | Image tag (default `latest`) |

### 3. Pull and start

```bash
docker compose pull
docker compose up -d
```

First start runs database migrations inside the backend container. Check status:

```bash
docker compose ps
docker compose logs -f backend
```

Healthy API: [http://localhost:5000/api/health](http://localhost:5000/api/health)

### 4. Create the admin user

```bash
docker compose exec backend node prisma/seed.js
```

Default login (change immediately after first sign-in):

| Field | Value |
|-------|-------|
| Username | `admin` |
| Password | `useradm` |

### 5. Open the UI

- **Web UI:** [http://localhost:3000](http://localhost:3000)
- **API:** [http://localhost:5000/api/health](http://localhost:5000/api/health)

## How to use the application

### First-time setup

1. Sign in as `admin` / `useradm`.
2. Open **Datacenters** → **Add Datacenter**.
3. Enter your Proxmox host, port (often `8006`), and credentials (user/password **or** API token).
4. Save and wait for the connection to succeed.
5. *(Optional)* **Settings → LDAP** for Active Directory login.
6. Under **Users**, create accounts or assign roles/datacenter access for AD users.
7. Enable **MFA** on the admin account.

### Day-to-day

| Area | What you can do |
|------|-----------------|
| **Dashboard** | Overview of VMs / PBS resources; filter PVE vs PBS |
| **VMs** | Inspect resources; start, stop, restart |
| **PBS** | Datastores, backup jobs, verification, GC / task history |
| **Activity logs** | Filter and export CSV / JSON / HTML |
| **External links** | Admin-defined shortcuts (IPAM, monitoring, …) |

### Roles

| Role | Access |
|------|--------|
| **Admin** | Full control (users, datacenters, LDAP, MFA, links) |
| **Operator** | Manage VMs on assigned datacenters |
| **Viewer** | Read-only on assigned datacenters |

## Architecture (this compose stack)

```text
Browser
   │
   ├─ :3000 ─► frontend  (dridilotfi/proxmox-manager-frontend)
   │              │
   │              └─ /api/* ─► backend:5000  (Docker network)
   │
   └─ :5000 ─► backend   (dridilotfi/proxmox-manager-backend)
                  │
                  ├─► db (postgres:15-alpine)
                  ├─► redis (redis:7-alpine)
                  └─► your Proxmox VE / PBS APIs
```

Postgres and Redis are **not** published on the host by default (only reachable inside the Compose network).

## Updates

```bash
docker compose pull
docker compose up -d
```

Pin a specific tag in `.env` if you prefer not to float on `latest`:

```env
IMAGE_TAG=1.0.0
```

(Only if those tags exist on Docker Hub.)

## Useful commands

```bash
# Logs
docker compose logs -f backend
docker compose logs -f frontend

# Stop (keep data volume)
docker compose down

# Stop and delete the database volume (destructive)
docker compose down -v

# Re-seed admin (upserts admin / useradm)
docker compose exec backend node prisma/seed.js
```

## Deploy behind a reverse proxy / HTTPS

If users open the app as `https://proxmox-manager.example.com`:

1. Set in `.env`:
   ```env
   FRONTEND_URL=https://proxmox-manager.example.com
   NEXT_PUBLIC_API_URL=https://proxmox-manager.example.com/api
   ```
   Or point `NEXT_PUBLIC_API_URL` at your public API URL if the API is on a separate hostname.
2. Proxy to the frontend (and optionally the backend). For realtime Socket.IO, proxy WebSocket upgrades to the **backend** (path `/socket.io`).
3. Keep `INTERNAL_BACKEND_URL` as `http://backend:5000` inside Compose (already set in `docker-compose.yml`).

## Troubleshooting

| Symptom | What to try |
|---------|-------------|
| Backend unhealthy / restarting | `docker compose logs backend` — often bad DB password, migrations, or missing secrets |
| `P1000: Authentication failed` | Password special chars (especially `@`) not URL-encoded in `DATABASE_URL`, **or** Postgres volume was created with an older password — fix `.env`, then `docker compose down -v` and `up -d` again (wipes DB data) |
| Cannot log in on a fresh install | Run `docker compose exec backend node prisma/seed.js` |
| UI loads but API calls fail | Confirm backend health; check `FRONTEND_URL` / CORS; ensure ports `3000`/`5000` match how you browse |
| Cannot reach Proxmox | From the host, verify TCP to Proxmox (`8006`). Containers must route to that network |
| LDAP errors | Check bind DN, base DN, TLS options, and that the backend container can resolve the LDAP host |
| Permission denied in UI | Check the user role and datacenter assignments |
| Pull errors from Docker Hub | `docker login` if the images are private; otherwise check network / rate limits |

## Security checklist

- [ ] Replaced every `change-me-*` value in `.env`
- [ ] Changed the default `admin` / `useradm` password (or disabled that account after creating another admin)
- [ ] Enabled MFA for admins
- [ ] Did **not** commit `.env` to Git
- [ ] Restricted who can reach ports 3000/5000 (firewall / VPN / reverse proxy)
- [ ] Using HTTPS in any non-local deployment

## Support / images

- Backend image: https://hub.docker.com/r/dridilotfi/proxmox-manager-backend
- Frontend image: https://hub.docker.com/r/dridilotfi/proxmox-manager-frontend

## License

MIT — see [LICENSE](./LICENSE).
