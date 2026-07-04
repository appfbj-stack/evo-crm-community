# Evo CRM Community — Estado Atual da Plataforma

> Mapeamento consolidado do monorepo `evo-crm-community` (fork em `appfbj-stack`).
> 6 submódulos + 2 dependências third-party + 1 monorepo orquestrador.
> Data: 2026-07-02

---

## 1. Visão Geral

| Serviço | Stack | Porta | Status |
|---|---|---|---|
| `evo-auth-service-community` | Ruby 3.4 / Rails 7.1 (API-only) | 3001 | Funcional, com débitos |
| `evo-ai-crm-community` | Ruby 3.4 / Rails 7.1 (API-only) | 3000 | Funcional, com débitos |
| `evo-ai-frontend-community` | React 19 / Vite 6 / TypeScript 5.7 | 5173 → 80 (nginx) | Funcional, com muitos stubs |
| `evo-ai-processor-community` | Python 3.11 / FastAPI 0.115 | 8000 | Funcional, com débitos |
| `evo-ai-core-service-community` | Go 1.24 / Gin / GORM | 5555 | Funcional, sem testes |
| `evo-bot-runtime` | Go 1.24 / Gin / redsync | 8080 | Funcional, fora do release |
| `evolution-api` (third-party) | Node / TypeScript | (variável) | Dependência |
| `evolution-go` (third-party) | Go | (variável) | Dependência |

**Versões ativas dos submódulos:**
- `evo-ai-crm-community` → `v1.0.0-rc3~10^2`
- `evo-ai-frontend-community` → `v1.0.0-rc3~14^2`
- `evo-ai-core-service-community` → `v1.0.0-rc1~1`
- `evo-auth-service-community` → `v1.0.0-rc1~1^2`
- `evo-ai-processor-community` → commit `9050693` (sem tag)
- `evo-bot-runtime` → `v1.0.0-rc1`
- `evolution-api` → `2.3.7` (modificado localmente)
- `evolution-go` → `0.6.1`

---

## 2. Arquitetura de Comunicação

```
[Frontend :5173 / :80]
        │
        ▼ Bearer JWT / api_access_token
[evo-auth :3001] ──── validate_token ──── [evo-crm :3000] ──── HTTParty ──── [evo-core :5555]
[evo-crm :3000] ──── X-Service-Token ──── [evo-processor :8000]
[evo-processor :8000] ── A2A / x-api-key ── [evo-bot-runtime :8080]
[evo-bot-runtime] ── redlock ── [redis]
```

- **Auth inbound**: Bearer JWT (Doorkeeper + JWT HS256) **ou** `api_access_token` UUID header **ou** `X-Service-Token` (server-to-server, `EVOAI_CRM_API_TOKEN`).
- **Encryption compartilhada**: Fernet com `ENCRYPTION_KEY` (mesma chave usada em Core e Processor).
- **DB compartilhado**: 1 Postgres único (`evo_community`) com 5 schemas/services escrevendo nele.
- **Cache/Queue**: Redis único, namespaces por serviço.

---

## 3. Débitos Críticos (por serviço)

### evo-auth-service-community
- `Dockerfile:38` deleta `bin/docker-entrypoint` mas não define `CMD` — imagem não roda sem comando explícito.
- `RolePermission` model referencia tabela `role_permissions` que **não existe** (só `role_permissions_actions`) — vai explodir ao carregar.
- `setup_controller.rb:114` expõe secrets `_SECRET` ao frontend em plaintext.
- Testes: 2 specs cobrindo 2 modelos apenas. README afirma 95% coverage — **falso**.
- `Licensing::SetupGate` middleware bloqueia todas as requests; bypass é hardcoded.
- `tier = 'evo-ai-crm-community'` hardcoded.
- `Obfuscated method names` (`_2s`, `_9ps`) em `licensing/activation.rb` — anti-tamper que pode crashar.
- Default `EVOAI_CRM_API_TOKEN=6e10e689-…` em `.env.example` — risco se vazar.

### evo-ai-crm-community
- 60 modelos, 100+ controllers, **apenas 48 spec files** — cobertura baixíssima.
- **Campaign** foi removido e substituído por Pipeline mas referências mortas em 12+ arquivos.
- `Message` default scope marcado como TODO para remover.
- `installation_config.rb:30` tem TODO para remover default scope.
- `dashboard_controller.rb:114` expõe `_SECRET` em plaintext (segurança).
- `pipelines` têm custom_fields (jsonb) + position + hierarchy de tasks com `depth`/`position`.
- Integração com AI via `EvoAiCoreService` (HTTParty) + Processor via `session_sync_service`.
- **pgVector image** está no compose mas extensão `vector` **não está habilitada** no schema.
- Whatsapp suporta 7 providers (default, whatsapp_cloud, baileys, evolution, evolution_go, notificame, zapi).
- Sem Dockerfile `HEALTHCHECK` — health tem que ser sondado externamente.
- Botão de campanhas ainda aparece no menu lateral mas sem backend.

### evo-ai-core-service-community
- **Zero testes** (nenhum `*_test.go`).
- "Sync Evolution bot" endpoint mencionado no README **não existe** (método `SyncAgentBot` existe, mas não tem rota).
- CORS middleware faz `Access-Control-Allow-Origin` echo do request (bypass do `isAllowedOrigin`).
- Module `pkg/mcp_server/module.go:1` declara `package folderShare` (bug de nome).
- Permission fallback permissivo: se 404 do auth, libera tudo.
- `user_admin` middleware nunca seta `c.IsAdmin`.
- Folder-share `SharedWithUserID` sempre = `SharedByUserID` (bug).

### evo-ai-processor-community
- **Zero testes** (config existe, diretório `tests/` ausente).
- `scripts/run_seeders.py` chamado no boot com `all_seeders = {}` — no-op.
- `task_push_configs = {}` global (em memória) — push notifications perdidos em restart.
- Tasks A2A são "ephemeral" — `tasks/cancel`, `tasks/get`, `tasks/resubscribe` retornam mock.
- A2A é "hand-rolled JSON-RPC" apesar do claim de 100% compliance.
- Docs/Swagger desabilitados (`docs_url=None, openapi_url=None`).
- CORS `allow_origins=["*"]`.
- Dependências de vector stores (Pinecone/Qdrant/OpenSearch) declaradas mas não usadas.
- Migrations + seeders rodam em **todo boot** (desperdício).
- Alembic tem tabela `evo_agent_processor_execution_metrics` mas ORM diz `evo_ai_agent_processor_execution_metrics` — mismatch.
- `KNOWLEDGE_SERVICE_API_TOKEN` referenciado mas não declarado em settings.

### evo-ai-frontend-community
- **Múltiplas páginas stub**: `/bots`, `/reports`, `/settings/integrations/:integrationId`, várias integrações específicas.
- ThemeContext é **no-op** (dark mode hardcoded, toggle não funciona).
- Vários botões críticos com `console.log` apenas (ver contato, retry mensagem, deletar MCP server, etc).
- 100+ usos de `any` em TypeScript.
- Arquivos monolíticos: `Widget/index.tsx` (71 kB), `Profile.tsx` (46 kB), `PipelineKanban.tsx` (47 kB).
- Sem React Query/SWR — toda fetch é service manual.
- `StrictMode` removido do `main.tsx` (comentário "para evitar duplicação").
- GA4 ID hardcoded em vez de env.
- i18n bundleia todos os 6 locales × 45 namespaces no bundle inicial.
- 3 lockfiles (npm, pnpm, yarn) — inconsistência.
- 10 spec files — testes só em admin settings e utils.
- Sem Cypress/Playwright.

### evo-bot-runtime
- Sem README próprio.
- **Não está no `release.yml`** do monorepo — image só é publicada manualmente.
- Default porta no `.env.example` é 8090, mas compose usa 8080 — inconsistência.
- `k8s/` existe mas compose assume Swarm — duas estratégias de deploy.

### Raiz do monorepo
- `bsuid.html` (135 kB) commitado sem razão — material de marketing órfão.
- `docs/SETUP-GUIDE.md`, `docs/TROUBLESHOOTING.md`, `CONTRIBUTING.md` referenciados mas **não existem** (404).
- `docker-compose.prod-test.yaml` tem segredos de dev inlineados.
- `docker-compose.swarm.yaml` tem `placement.constraints: [node.hostname == manager1]` — single point of failure.
- `release.yml` não publica `evo-bot-runtime` nem gateway via GHCR.
- Dois registries: GHCR (`ghcr.io/evolutionapi/*`) e Docker Hub (`evoapicloud/*`) sem sync.
- Submódulos só por SSH — clones HTTPS precisam rewrite.
- `_evo/` é o scaffolding interno (gitignored) com workflows tipo BMad Method.
- 0 testes rodando em CI.
- Sem `CODEOWNERS`, sem `SECURITY.md`, sem `CHANGELOG.md` raiz.

---

## 4. Funcionalidades por Status

### Implementadas e funcionais
- Auth (Bearer JWT + DeviseTokenAuth + api_access_token + service token)
- MFA (TOTP + Email OTP + backup codes, lockout 5 tentativas)
- RBAC (custom permission_key system, 24+ resources, 250+ permission keys)
- OAuth 2.0 (Doorkeeper provider, RFC 7591 client registration)
- 11 canais (WhatsApp × 7 providers, Email, Facebook, Instagram, Line, Telegram, Twitter, SMS, Twilio, API, Web Widget)
- Pipelines (kanban com stages, items, tasks hierárquicas, drag-and-drop)
- AI Agents (LLM, Sequential, Parallel, Loop, A2A, Workflow, Crew_AI, Task, External)
- Tools custom (HTTP), MCP servers (registrados + custom), 12+ integrações OAuth
- A2A protocol (hand-rolled, parcialmente compliant)
- ActionCable realtime (chat, presence, typing, status)
- CSAT surveys
- Macros, Canned Responses, Custom Attributes, Custom Filters
- Scheduled Actions com recorrência
- Reports v2 (summary, live, by agent/team/inbox)
- Public widget (embeddable)
- Webhooks (in/out)
- ActionCable + WebSocket chat
- Action Mailbox
- RAG (via knowledge service, sem vector store local)
- Memory (via HTTP para knowledge service)
- Bot pipeline (debounce + Redis locks + dispatch)
- Product tours (22 React-Joyride)
- Impersonation
- Multi-idioma (6 locales: en, pt-BR, pt, es, fr, it)
- Dark mode (hardcoded)
- i18n com 45 namespaces
- Setup wizard + onboarding
- GDPR/LGPD (data privacy consents, export, deletion)
- Migrations automatizadas
- Cron jobs (12 jobs, sidekiq-cron)
- Prometheus metrics + OpenTelemetry traces (opt-in)
- Multi-arch Docker images (amd64+arm64)
- nginx API gateway com path-routing e rate limits

### Implementadas parcialmente / com débitos
- Multi-tenant (single-tenant na verdade, `account` é JSON blob)
- Campaigns (removido, referências mortas)
- Audit logs (removido)
- Custom roles (removido)
- SLA system (removido)
- Portal system (removido)
- Platform apps (removido)
- pgVector (imagem ok, extensão não habilitada)
- Vector stores (Pinecone/Qdrant/OpenSearch) — deps sem uso
- MCP live protocol (não implementado, só config storage)
- A2A task persistence (in-memory only)

### Stubs / placeholders
- /bots (frontend)
- /reports (frontend)
- /settings/integrations/:integrationId (frontend)
- Vários MCP server actions (frontend, console.log)
- Reportes v1 (commented out)
- Plans / Features (auth service — community stubs)
- Custom MCP server actions (frontend)

---

## 5. Gaps Arquiteturais

1. **Sem testes end-to-end** em nenhum serviço.
2. **Sem CI de testes** — só lint de Dockerfile + `docker compose config`.
3. **`release.yml` incompleto** — não publica `evo-bot-runtime`.
4. **Dois registries** sem sync automático.
5. **`_evo/` e `_evo-output/`** gitignored mas referenciados em paths hardcoded.
6. **Sem tradução de "sync"** entre Evo CRM e o fork `appfbj-stack` — submódulos apontam para upstream `EvolutionAPI`, não para o fork.
7. **Alembic mismatch** entre ORM e migration no processor.
8. **No `worker.ts` pattern** para SSE/streaming em JSX — done ad-hoc.
9. **Auth service quebra** se tentar usar o `RolePermission` model.
10. **CRM expõe secrets** ao frontend no `dashboard_controller.rb`.

---

## 6. Próximos Passos (propostos, não validados)

Decidir com o usuário **qual problema resolver primeiro**:

1. **Consertar débitos críticos** (Dockerfile auth, CORS core, frontend stubs, secrets leak)
2. **Adicionar testes** (auth + CRM mínimo viável)
3. **Endurecer CI** (rodar testes por submódulo, hadolint, security scan)
4. **Migrar de LICENSE-gated** para um modelo open source puro
5. **Atualizar submódulos** para as últimas versões upstream
6. **Documentar arquitetura** (decisões ADRs)
7. **Reescrever frontend** com Shadcn/Radix + remover dependência de `@evoapi/design-system` privado
8. **Migrar de pgVector** se for usar RAG de verdade (ou remover deps de vector stores)
9. **Implementar Evolution bot sync endpoint** que falta
10. **Refatorar AI processor** para usar A2A SDK em vez de hand-rolled
