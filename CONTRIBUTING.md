# Contributing

Thanks for considering a contribution to **Evo CRM Community**. This
project is a monorepo of six microservices plus a Gateway image; each
service is also maintained as its own upstream repository.

This document covers how to send pull requests to this **monorepo** (the
orchestrator). For service-internal changes, please open a PR in the
relevant upstream repository instead — the monorepo only tracks pinned
submodule SHAs.

---

## 1. Where to send your change

| Change | Target |
|--------|--------|
| Anything in `evo-auth-service-community/`, `evo-ai-crm-community/`, `evo-ai-frontend-community/`, `evo-ai-processor-community/`, `evo-ai-core-service-community/`, `evo-bot-runtime/` | upstream repo of that service |
| `docker-compose*.yml`, `Makefile`, `nginx/`, `.github/workflows/`, root docs, `evo_crm.yml` example | this monorepo |
| `evolution-api/`, `evolution-go/` | upstream `EvolutionAPI/*` — do not modify directly |

If you are not sure, open a discussion issue first.

---

## 2. Local development setup

```bash
# Clone with submodules
git clone --recurse-submodules https://github.com/appfbj-stack/evo-crm-community.git
cd evo-crm-community

# Bring up the stack
make setup

# After your changes, rebuild only the affected service
make shell-<service>     # open a shell inside the service container

# Run the tests
cd evo-ai-processor-community
make test
```

---

## 3. Coding standards

- **Indentation:** match the existing style per service. Ruby uses 2
  spaces, TypeScript/React uses 2 spaces, Python uses 4 spaces, Go uses
  tabs (gofmt).
- **Commits:** `feat: …`, `fix: …`, `chore: …`, `docs: …`,
  `refactor: …`, `test: …`. One logical change per commit.
- **Tests:** add a test for every bug fix. New features ship with tests
  in the relevant service. The monorepo CI runs the lightweight suites;
  full integration tests live in each service.
- **No secrets in git.** `.env` is git-ignored by design. Use the
  generation commands in [`SETUP-GUIDE.md`](./SETUP-GUIDE.md) §4 to
  produce strong secrets.
- **Submodule bumps:** one commit per submodule bump. Reference the
  upstream release notes in the commit body.

---

## 4. Pull request checklist

Before opening a PR, confirm:

- [ ] Branched from `main`.
- [ ] `git submodule status` is clean (no stray `+` or `-` markers).
- [ ] `docker compose config --quiet` passes locally.
- [ ] `make test` passes for any service whose code you changed.
- [ ] Updated the relevant docs (`SETUP-GUIDE.md`, `TROUBLESHOOTING.md`,
      service README).
- [ ] No new TODOs without a linked issue.
- [ ] No secrets, tokens or `.env` files in the diff.

---

## 5. CI

The CI pipeline (`.github/workflows/ci.yml`) runs on every PR to `main`:

1. `docker compose config --quiet` — validates the compose files.
2. Hadolint on the community Dockerfiles.

The release pipeline (`.github/workflows/release.yml`) runs on tags
matching `v*.*.*` and publishes six images to GHCR
(`ghcr.io/evolutionapi/*`).

If you change CI behaviour, also test it on a fork before opening a PR.

---

## 6. Reporting bugs

Use the **bug report** issue template. Include:

1. Steps to reproduce.
2. Affected service (auth / CRM / frontend / processor / core / docker / unknown).
3. Expected vs. actual behaviour.
4. Environment: OS, Docker version, browser.
5. Relevant logs (`make logs SERVICE=<name>`) with secrets redacted.

---

## 7. License

By contributing, you agree that your contributions will be licensed
under the [Apache License 2.0](../LICENSE).
