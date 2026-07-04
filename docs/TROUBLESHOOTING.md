# Troubleshooting

Common errors and how to resolve them. Sections are ordered from "most
likely on first run" to "rare but worth knowing".

---

## 1. `make setup` hangs forever

**Symptom:** `make setup` runs for 20+ minutes and never finishes.

**Cause:** the build step compiles gems and Node modules inside Docker
and is slow on first run. The default `docker compose build` has no
parallelism, so the cold build is sequential.

**Fix:** wait. Subsequent builds are cached. If you cancel and restart,
`docker compose build` will resume from the cache.

---

## 2. `bind: address already in use`

**Symptom:** one of the services fails to start with `bind: address
already in use` for port 3000, 3001, 5173, 5555, 8000, or 8080.

**Fix:**

```bash
# Linux/macOS: see who owns the port
sudo lsof -i :3000
# Windows:
netstat -ano | findstr :3000

# Stop the offending process, or change the port in docker-compose.yml
# and .env, then `make restart`.
```

---

## 3. Postgres connection refused after `make setup`

**Symptom:** CRM/Auth/Core/Processor logs show:

```
FATAL:  password authentication failed for user "postgres"
```
or
```
dial tcp 127.0.0.1:5432: connect: connection refused
```

**Fix:**

1. Confirm the container is up:
   ```bash
   make status
   ```
2. If `postgres` is unhealthy, check the volume:
   ```bash
   docker volume inspect evo-crm-community_postgres_data
   ```
3. Wipe and re-init (⚠ destroys all data):
   ```bash
   make clean
   make setup
   ```
4. If using a managed Postgres (Supabase, RDS, …), confirm
   `POSTGRES_SSLMODE=require` and that the security group allows the
   IP of the VPS.

---

## 4. `pgvector` extension missing

**Symptom:** the CRM service fails to start with:

```
PG::UndefinedFile: ERROR: extension "vector" is not available
```

**Cause:** the database was provisioned without the pgvector extension.
Evo CRM uses the `pgvector/pgvector:pg16` image which ships the
extension, but it must be enabled per database:

```sql
\c evo_community
CREATE EXTENSION IF NOT EXISTS vector;
```

For managed Postgres, see the provider's pgvector guide (Supabase,
Neon, RDS, Cloud SQL, Azure Database for PostgreSQL all support it).

---

## 5. Auth login fails with "Login error"

**Symptom:** submitting the default credentials returns
`Error: Login error` on the login page.

**Cause:** the auth service has not been seeded, or the seed ran
against a different database than the one the auth service is now
talking to.

**Fix:**

```bash
# Re-run the auth seed
make seed-auth

# Verify the user exists
make shell-auth
bundle exec rails runner 'puts User.pluck(:email)'
```

You should see `support@evo-auth-service-community.com`.

---

## 6. CRM seed fails: "User not found"

**Symptom:**

```
db/seeds.rb: User with email 'support@evo-auth-service-community.com' not found.
Have you run rails db:seed in evo-auth-service-community first?
```

**Fix:** the CRM seed depends on the auth seed. Run them in order:

```bash
make seed-auth
make seed-crm
```

Or simply `make seed`, which calls both.

---

## 7. Frontend shows "Network Error" on every API call

**Symptom:** the SPA loads, but every action returns a CORS or
connection error in the browser console.

**Cause:** the `VITE_*` build-time variables in the frontend image point
at the wrong host, or the CORS allowlist in the backend is too narrow.

**Fix:**

1. Confirm `VITE_API_URL`, `VITE_AUTH_API_URL`, `VITE_EVOAI_API_URL`,
   `VITE_AGENT_PROCESSOR_URL` in `.env` point at reachable URLs.
2. The browser hits these URLs directly, so they must be reachable
   from the user's machine, **not** from inside Docker. In dev, use
   `http://localhost:<port>`.
3. Update `CORS_ORIGINS` in `.env` to include the frontend origin.
4. Rebuild the frontend: `docker compose build evo-frontend`.

---

## 8. `EVOAI_CRM_API_TOKEN` mismatch between services

**Symptom:** logs in any service show 401/403 errors when calling
another service in the stack.

**Cause:** the shared `EVOAI_CRM_API_TOKEN` is different in `.env` for
two services, or one service was started before the token was rotated.

**Fix:**

1. Confirm every service in `.env` has the same value for
   `EVOAI_CRM_API_TOKEN`.
2. Restart everything:
   ```bash
   make restart
   ```

The service token is also exposed as `EVOAI_AUTH_API_TOKEN` in the auth
service — the auth service uses whichever is set; they should match.

---

## 9. Processor migrations don't run

**Symptom:** the processor container exits immediately and logs:

```
sqlalchemy.exc.ProgrammingError: (psycopg2.errors.UndefinedTable) ...
```

**Fix:**

1. The processor uses Alembic. The `CMD` in the Dockerfile runs
   `alembic upgrade head` before starting uvicorn. If the DB is
   unhealthy, the migration step will fail. Check Postgres first (see
   section 3).
2. Manually run migrations:
   ```bash
   make shell-processor
   alembic upgrade head
   ```

---

## 10. Port 5173 already in use (Vite dev server)

**Symptom:** the `evo-frontend` container exits because port 5173 is
already taken by a local Vite dev server.

**Fix:** stop the local Vite dev server, or change the host port in
`docker-compose.yml`:

```yaml
evo-frontend:
  ports:
    - "9000:80"   # was 5173:80
```

---

## 11. Submodule out of date

**Symptom:** `make setup` uses an old version of a service that lacks
features you expected.

**Fix:**

```bash
git submodule update --remote --merge
docker compose build
make restart
```

---

## 12. Still stuck?

- Read the upstream docs: <https://docs.evolutionfoundation.com.br>
- Search the issue tracker: <https://github.com/EvolutionAPI/evo-ai-services/issues>
- Open an issue using the bug report template in this repo. Include:
  - output of `make status`
  - the full log of the failing service: `make logs SERVICE=<name>`
  - the contents of `.env` (redact secrets!)
