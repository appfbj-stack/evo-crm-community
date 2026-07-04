# Setup Guide

This guide walks you through installing **Evo CRM Community** on a local
machine and (optionally) deploying it to a single-VPS production stack.

The platform is a monorepo of 6 microservices. All commands are run from
the repository root unless noted otherwise.

---

## 1. Prerequisites

| Tool | Version | Notes |
|------|---------|-------|
| Git | 2.30+ | With submodule support |
| Docker | 24+ | Docker Engine + Compose v2 (`docker compose`) |
| Make | any | Optional but recommended |
| 4 GB free RAM | — | For the dev stack (Postgres + Redis + 5 services) |
| Ports free | 3000, 3001, 5173, 5555, 8000, 8080, 8025 | Frontend, CRM, Auth, Core, Processor, Bot Runtime, Mailhog |

Linux/macOS: Docker and Make usually come preinstalled or via your package manager.
Windows: use [Docker Desktop](https://docs.docker.com/desktop/) and [Make for Windows](https://gnuwin32.sourceforge.net/packages/make.htm).

---

## 2. Quick start (local dev)

```bash
# Clone with all submodules
git clone --recurse-submodules https://github.com/appfbj-stack/evo-crm-community.git
cd evo-crm-community

# Bring every service up, build images, seed the database
make setup
```

That single command will:
1. Copy `.env.example` → `.env` (only if `.env` does not already exist)
2. Initialize all git submodules
3. Build all Docker images (5–15 min on first run)
4. Start Postgres, Redis and Mailhog
5. Wait for the database to be ready
6. Seed the auth service (creates the two system roles + account)
7. Seed the CRM service (depends on the auth seed)
8. Start every service in the background

When the banner appears, the stack is ready. Open the frontend:

```
http://localhost:5173
```

Default credentials created by `make setup` (auth seed):

```
Email:    support@evo-auth-service-community.com
Password: Password@123
```

> Rotate the default password immediately after the first login.

### Useful daily commands

```bash
make status        # which containers are running
make logs          # tail logs of all services
make logs SERVICE=evo-crm
make restart       # restart everything
make stop          # stop (data preserved)
make clean         # ⚠ DESTROY all data volumes
make shell-crm     # bash inside the CRM container
```

---

## 3. Database options

### Option A — bundled Docker Postgres (default)

No extra setup. Postgres is started by `docker compose` using the
`pgvector/pgvector:pg16` image and the `evo_community` database is
created automatically.

### Option B — local PostgreSQL 16+

Install PostgreSQL 16+ with the pgvector extension:

| OS | Install |
|----|---------|
| Ubuntu/Debian | `apt install postgresql-16 postgresql-16-pgvector` |
| macOS | [Postgres.app](https://postgresapp.com/) (pgvector is bundled) |
| Windows | [EDB installer](https://www.postgresql.org/download/windows/) + pgvector |

Create the database and enable the extension:

```sql
CREATE DATABASE evo_community;
\c evo_community
CREATE EXTENSION IF NOT EXISTS vector;
```

Then update `.env`:

```env
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_USERNAME=postgres
POSTGRES_PASSWORD=your_password
POSTGRES_DATABASE=evo_community
```

### Option C — managed Postgres (Supabase, Neon, RDS, …)

Any provider that supports PostgreSQL 16+ and pgvector works. After
provisioning, update `.env` with the connection details and add
`POSTGRES_SSLMODE=require` if the provider requires SSL.

---

## 4. Required environment variables

`make setup` creates a working `.env` automatically. For production, you
**must** replace the placeholder secrets with strong random values.

| Variable | Purpose | Generate with |
|----------|---------|---------------|
| `SECRET_KEY_BASE` | Rails secret (auth, crm) | `bundle exec rails secret` |
| `JWT_SECRET_KEY` | JWT signing (auth, crm, core) | `openssl rand -hex 64` |
| `DOORKEEPER_JWT_SECRET_KEY` | Doorkeeper signing | `openssl rand -hex 64` |
| `ENCRYPTION_KEY` | Fernet for API keys (core, processor) | `openssl rand -base64 32` |
| `EVOAI_CRM_API_TOKEN` | Inter-service service token | `uuidgen` |
| `BOT_RUNTIME_SECRET` | Bot runtime HMAC | `openssl rand -hex 32` |
| `POSTGRES_PASSWORD` | DB admin password | `openssl rand -hex 24` |
| `REDIS_PASSWORD` | Redis auth | `openssl rand -hex 24` |

Do **not** reuse the dev values from `.env.example` in production.

---

## 5. Updating submodules

The monorepo pins specific commits of each submodule. To bring them
in line with the latest:

```bash
git submodule update --remote --merge
git add . && git commit -m "chore: bump submodules"
```

To pin a specific version:

```bash
cd evo-auth-service-community
git checkout v1.2.3
cd ..
git add evo-auth-service-community
git commit -m "chore: pin auth to v1.2.3"
```

---

## 6. Production (single VPS, Docker Swarm)

Use the reference `evo_crm.yml` Swarm stack:

1. Provision a VPS (Ubuntu 22.04+ recommended, 4 vCPU / 8 GB RAM minimum).
2. Install Docker Engine + the Swarm plugin.
3. Create the shared external resources:

   ```bash
   docker network create --driver=overlay network_public
   docker volume create evocrm_redis
   docker volume create evocrm_processor_logs
   docker volume create pgvector
   ```

4. Provision a managed Postgres (Supabase, Neon, RDS, …) with pgvector.
5. Create a Traefik stack with a `websecure` entrypoint and a
   `letsencryptresolver` certificate resolver.
6. Copy `evo_crm.yml` to the VPS and customize:
   - replace `seudominio.com.br` with your domain
   - replace SMTP credentials
   - replace the `CHANGE_ME` secrets with the generated values
7. Deploy:

   ```bash
   docker stack deploy -c evo_crm.yml evocrm
   ```

8. After the stack is up, visit `https://crm.<your-domain>` and follow
   the setup wizard to create the first admin user.

---

## 7. Smoke test

After `make setup`, run these checks:

```bash
# Auth health
curl -fsS http://localhost:3001/health

# CRM health
curl -fsS http://localhost:3000/health/live

# Core health
curl -fsS http://localhost:5555/api/v1/health

# Processor health
curl -fsS http://localhost:8000/health

# Frontend health (via nginx)
curl -fsS http://localhost:5173/health
```

If all five return 200, the stack is healthy.

---

## 8. Where to go next

- See [`TROUBLESHOOTING.md`](./TROUBLESHOOTING.md) for common errors.
- See [`CONTRIBUTING.md`](./CONTRIBUTING.md) to send patches.
- See the upstream documentation at <https://docs.evolutionfoundation.com.br>
  for product-level details.
