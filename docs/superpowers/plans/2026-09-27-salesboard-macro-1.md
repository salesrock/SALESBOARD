# Salesboard Macro Fase 1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Entregar a fundação segura e multi-tenant do Salesboard com Cloudflare Workers/D1, autenticação, RBAC, auditoria, CRUD básico de clientes, UI interna, CI e documentação, sem iniciar a Macro 2.

**Architecture:** Aplicação Next.js 16 App Router executada em Cloudflare Workers por vinext, hoje o caminho recomendado pela Cloudflare para projetos Next.js novos. O backend acessa D1 exclusivamente por Drizzle, Better Auth mantém credenciais e sessões, e todas as rotas/mutações passam por autenticação, permissões centralizadas e escopo de cliente antes de consultar dados.

**Tech Stack:** TypeScript strict, Next.js 16, React, vinext, Cloudflare Workers, Wrangler, D1, Drizzle ORM/Kit, Better Auth, Zod, Tailwind CSS, Vitest, Testing Library e Playwright.

**Spec:** Requisitos combinados de `SALESBOARD_SPEC.md` e `SALESBOARD_CODEX_START.md`, fornecidos pelo usuário; esta entrega termina ao concluir a Macro Fase 1.

## Global Constraints

- Implementar somente Fundação + Auth/RBAC + CRUD básico de clientes; não implementar Google Sheets, sync, dashboards de mídia, conteúdos, IA, LinkedIn funcional, PDF, R2 ou subdomínios.
- Usar Cloudflare Workers, D1, Turnstile, DNS, GitHub Actions e Wrangler; não usar Vercel.
- Usar vinext para Next.js em Workers, conforme a recomendação oficial vigente da Cloudflare em 27/09/2026.
- Usar TypeScript strict, Drizzle ORM, Zod e Better Auth com e-mail/senha e sessões revogáveis.
- Validar autenticação, autorização e vínculo de tenant no servidor; slug nunca autoriza acesso.
- Não permitir cadastro público, senha padrão, segredo em seed, cliente hard-coded ou dados reais em fixtures.
- Usar IDs internos aleatórios e strings; persistir datas em UTC.
- Validar Turnstile no servidor e aplicar rate limit progressivo ao login.
- Criar migrations versionadas; nenhuma alteração manual de schema em produção.
- Criar commits pequenos após cada tarefa aprovada e testada.
- Executar lint, typecheck, testes e build antes do relatório final; parar antes da Macro 2.

## Review Focus

- Usuário `disabled` com cookie ainda válido deve perder acesso imediatamente; cobrir em teste de integração de sessão.
- Usuário interno não atribuído não pode abrir cliente por ID ou slug; cobrir tentativa IDOR em teste de autorização.
- Slug duplicado, inválido ou alterado deve gerar erro amigável sem corromper o cadastro; cobrir no domínio e na integração.
- Falha/ausência do Turnstile em produção deve bloquear login, enquanto a chave oficial de teste deve viabilizar testes locais; cobrir no adaptador.
- Desativação/arquivamento deve ser soft delete, preservar auditoria e nunca apagar vínculos; cobrir no serviço de clientes.

---

### Task 1: Scaffold Next.js/vinext and quality gates

**Files:**
- Replace: `index.html`
- Create: `package.json`, `package-lock.json`, `next.config.ts`, `vite.config.ts`, `wrangler.jsonc`, `tsconfig.json`, `eslint.config.mjs`, `postcss.config.mjs`
- Create: `src/app/layout.tsx`, `src/app/page.tsx`, `src/app/globals.css`, `src/app/api/health/route.ts`
- Create: `src/lib/env.ts`, `src/lib/env.test.ts`, `.gitignore`, `.dev.vars.example`

**Interfaces:**
- Produces: `getServerEnv(source?: Record<string, string | undefined>): ServerEnv` and a Workers-ready build/deploy pipeline.
- Produces: `GET /api/health -> { status: "ok", environment: string }` without leaking secrets.

- [ ] **Step 1: Scaffold the current official Next.js/vinext project shape**

Run the Cloudflare-recommended scaffold in a temporary sibling directory, inspect its generated versions/configuration, then copy only the required generated files into this repository. Preserve `.git` and do not deploy.

- [ ] **Step 2: Write failing environment tests**

```ts
import { describe, expect, it } from "vitest";
import { getServerEnv } from "./env";

describe("getServerEnv", () => {
  it("rejects production without required secrets", () => {
    expect(() => getServerEnv({ APP_ENV: "production" })).toThrow();
  });

  it("accepts a complete development configuration", () => {
    expect(getServerEnv({
      APP_ENV: "development",
      APP_URL: "http://localhost:3000",
      AUTH_SECRET: "12345678901234567890123456789012",
      BOOTSTRAP_ADMIN_EMAIL: "admin@example.test",
      TURNSTILE_SITE_KEY: "1x00000000000000000000AA",
      TURNSTILE_SECRET_KEY: "1x0000000000000000000000000000000AA",
    }).APP_ENV).toBe("development");
  });
});
```

- [ ] **Step 3: Run the environment test and verify RED**

Run: `npm.cmd test -- src/lib/env.test.ts`

Expected: FAIL because `src/lib/env.ts` does not exist.

- [ ] **Step 4: Implement strict environment parsing and health route**

Use a server-only Zod schema with `APP_ENV`, `APP_URL`, `AUTH_SECRET`, `BOOTSTRAP_ADMIN_EMAIL`, Turnstile variables, and optional Google/e-mail variables. The health route returns only application status and verifies the D1 binding with `SELECT 1` once the database helper exists.

- [ ] **Step 5: Verify scaffold quality gates**

Run: `npm.cmd test -- src/lib/env.test.ts`, `npm.cmd run lint`, `npm.cmd run typecheck`, `npm.cmd run build`.

Expected: all commands exit 0.

- [ ] **Step 6: Commit**

```bash
git add .
git commit -m "chore: scaffold salesboard cloudflare app"
```

### Task 2: D1 schema, migrations, and repository boundary

**Files:**
- Create: `src/db/schema/auth.ts`, `src/db/schema/clients.ts`, `src/db/schema/audit.ts`, `src/db/schema/index.ts`
- Create: `src/db/client.ts`, `src/db/ids.ts`, `src/db/ids.test.ts`
- Create: `drizzle.config.ts`, `migrations/0001_initial.sql`, `migrations/0002_indexes.sql`
- Modify: `wrangler.jsonc`, `src/app/api/health/route.ts`, `package.json`

**Interfaces:**
- Produces: `createDb(binding: D1Database)` returning the typed Drizzle client.
- Produces: `createId(): string` using `crypto.randomUUID()`.
- Produces tables required by Macro 1: Better Auth tables, `clients`, `client_channels`, `client_users`, `internal_client_assignments`, `audit_logs`, `dashboard_preferences`, `invitations`, and `login_attempts`.

- [ ] **Step 1: Write failing ID and schema-shape tests**

```ts
it("creates non-sequential UUID identifiers", () => {
  const first = createId();
  const second = createId();
  expect(first).not.toBe(second);
  expect(first).toMatch(/^[0-9a-f-]{36}$/i);
});
```

Add a migration smoke test that applies every SQL file to an isolated local D1 database and checks the required table names through `sqlite_master`.

- [ ] **Step 2: Run database tests and verify RED**

Run: `npm.cmd test -- src/db/ids.test.ts tests/integration/migrations.test.ts`

Expected: FAIL because IDs, schema and migration runner are absent.

- [ ] **Step 3: Define the Drizzle schema and versioned SQL**

Use text UUID primary keys, explicit foreign keys, unique `(client_id, channel)` and `(client_id, user_id)` constraints, UTC integer timestamps, status checks, and indexes for client/user/audit lookups. Configure D1 as binding `DB` and `migrations_dir: "migrations"`.

- [ ] **Step 4: Apply and verify local migrations**

Run: `npm.cmd run db:migrate:local` and the focused tests.

Expected: both migration files apply once; the second run reports no pending migrations; tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/db drizzle.config.ts migrations wrangler.jsonc package.json package-lock.json tests
git commit -m "feat: add d1 schema and migrations"
```

### Task 3: Better Auth, Turnstile, rate limiting, and bootstrap invitation

**Files:**
- Create: `src/lib/auth/config.ts`, `src/lib/auth/server.ts`, `src/lib/auth/client.ts`, `src/lib/auth/session.ts`
- Create: `src/lib/auth/turnstile.ts`, `src/lib/auth/rate-limit.ts`, `src/lib/auth/bootstrap.ts`
- Create: `src/lib/mail/provider.ts`, `src/lib/mail/development.ts`
- Create: `src/app/api/auth/[...all]/route.ts`, `src/app/api/bootstrap/route.ts`
- Create: `src/features/auth/login-schema.ts`, `src/features/auth/login-schema.test.ts`
- Create: `tests/integration/auth.test.ts`

**Interfaces:**
- Produces: `getAuth(db, env)`, `requireSession(request)`, `assertActiveUser(sessionUser)`.
- Produces: `verifyTurnstile(token, ip, env): Promise<boolean>` and `checkLoginRateLimit(key, now): Promise<RateLimitDecision>`.
- Produces: `MailProvider.sendInvite()` and a local-only implementation that never exposes tokens in production.

- [ ] **Step 1: Write failing auth-domain tests**

Test 12-character minimum passwords, normalized e-mail, generic invalid-credential response, Turnstile rejection, progressive lockout, session revocation and immediate denial for disabled users.

```ts
it("blocks a disabled user even when the session has not expired", async () => {
  const response = await protectedRequest({ userStatus: "disabled", validSession: true });
  expect(response.status).toBe(403);
});
```

- [ ] **Step 2: Run auth tests and verify RED**

Run: `npm.cmd test -- src/features/auth/login-schema.test.ts tests/integration/auth.test.ts`

Expected: FAIL because auth services do not exist.

- [ ] **Step 3: Configure Better Auth for Workers/D1**

Use the Drizzle SQLite adapter, e-mail/password enabled with public signup disabled, minimum password length 12, secure `HttpOnly`/`SameSite=Lax` cookies, trusted origins from `APP_URL`, session expiration/rotation, and the Better Auth captcha plugin with provider `cloudflare-turnstile`. Mount the official Next.js handler under `/api/auth/[...all]`.

- [ ] **Step 4: Implement safe bootstrap**

Create an idempotent CLI/API flow restricted to local or authenticated deployment administration that reads `BOOTSTRAP_ADMIN_EMAIL`, creates a hashed one-use invitation expiring in 48 hours, and sends it through `MailProvider`. It must never create or print a production password/token.

- [ ] **Step 5: Verify auth GREEN**

Run focused tests and the complete test suite. Expected: login/session/revocation/disabled/Turnstile/rate-limit tests pass.

- [ ] **Step 6: Commit**

```bash
git add src/lib/auth src/lib/mail src/features/auth src/app/api/auth src/app/api/bootstrap tests package.json package-lock.json
git commit -m "feat: add secure authentication foundation"
```

### Task 4: Centralized RBAC and tenant authorization

**Files:**
- Create: `src/lib/auth/permissions.ts`, `src/lib/auth/permissions.test.ts`
- Create: `src/lib/auth/authorize.ts`, `tests/integration/authorization.test.ts`

**Interfaces:**
- Produces: `Role`, `PermissionContext`, `canViewClient`, `canManageClient`, `canManageUsers`, `canCreateContent`, `canPublishContent`, `canTriggerSync`, `canViewAudit`.
- Produces: `authorizeClientAccess(db, session, clientId, permission)` that returns a scoped client or throws a sanitized 403/404.

- [ ] **Step 1: Write a failing permission matrix test**

```ts
it.each([
  ["super_admin", true],
  ["manager", true],
  ["analyst", false],
  ["client_admin", false],
  ["client_viewer", false],
] as const)("canManageClient(%s) is %s", (role, allowed) => {
  expect(canManageClient({ role, assigned: true, sameClient: true })).toBe(allowed);
});
```

Add cross-tenant tests proving that changing path ID, slug, query or payload cannot escape assignments.

- [ ] **Step 2: Run permission tests and verify RED**

Run: `npm.cmd test -- src/lib/auth/permissions.test.ts tests/integration/authorization.test.ts`.

- [ ] **Step 3: Implement pure permissions and server authorization**

Keep role decisions in one immutable permission map; compose tenant checks from session state plus `client_users` or `internal_client_assignments`. Return not-found for resources outside the caller's scope where existence disclosure is unsafe.

- [ ] **Step 4: Run tests and commit**

```bash
git add src/lib/auth tests/integration/authorization.test.ts
git commit -m "feat: add role based access control"
```

### Task 5: Client domain, repository, audit, and server mutations

**Files:**
- Create: `src/features/clients/domain.ts`, `src/features/clients/schemas.ts`, `src/features/clients/schemas.test.ts`
- Create: `src/features/clients/repository.ts`, `src/features/clients/service.ts`
- Create: `src/features/audit/service.ts`, `src/features/audit/service.test.ts`
- Create: `src/app/api/clients/route.ts`, `src/app/api/clients/[clientId]/route.ts`
- Create: `tests/integration/clients.test.ts`

**Interfaces:**
- Produces: `Channel = "META" | "GOOGLE" | "LINKEDIN"`, `ClientStatus`, `CreateClientInput`, `UpdateClientInput`.
- Produces: `listClients`, `getClient`, `createClient`, `updateClient`, `disableClient`, `archiveClient` with actor context.
- Produces: audit events `client_created`, `client_updated`, `client_disabled` in the same mutation boundary.

- [ ] **Step 1: Write failing validation and service tests**

Cover slug normalization, uniqueness, IANA timezone, ISO currency, hex accent color, nullable URL, disabled LinkedIn UI state, create/update/disable/archive, channels replacement, audit metadata allowlist and unauthorized mutation rejection.

```ts
it("preserves the client and writes audit data when disabling", async () => {
  await service.disableClient(actor, client.id);
  expect(await repo.get(client.id)).toMatchObject({ status: "inactive" });
  expect(await audit.last()).toMatchObject({ eventType: "client_disabled", clientId: client.id });
});
```

- [ ] **Step 2: Run focused tests and verify RED**

Run: `npm.cmd test -- src/features/clients src/features/audit tests/integration/clients.test.ts`.

- [ ] **Step 3: Implement the client service boundary**

Parse all input with Zod, call centralized permission functions, use Drizzle parameter binding, update channels transactionally, soft-delete only, and sanitize conflict/not-found responses. Never branch on a client name or Deliziare.

- [ ] **Step 4: Verify GREEN and commit**

```bash
git add src/features src/app/api/clients tests/integration/clients.test.ts
git commit -m "feat: add client management domain"
```

### Task 6: Login and protected internal interface

**Files:**
- Create: `src/app/(auth)/login/page.tsx`, `src/components/auth/login-form.tsx`, `src/components/auth/turnstile-widget.tsx`
- Create: `src/app/(internal)/interno/layout.tsx`, `src/app/(internal)/interno/page.tsx`
- Create: `src/components/layout/internal-sidebar.tsx`, `src/components/layout/internal-header.tsx`, `src/components/layout/user-menu.tsx`
- Create: `src/components/ui/button.tsx`, `src/components/ui/input.tsx`, `src/components/ui/select.tsx`, `src/components/ui/status-badge.tsx`, `src/components/ui/empty-state.tsx`, `src/components/ui/error-state.tsx`, `src/components/ui/skeleton.tsx`
- Create: `src/app/loading.tsx`, `src/app/error.tsx`, `src/app/not-found.tsx`
- Create: `src/components/auth/login-form.test.tsx`

**Interfaces:**
- Consumes: Better Auth client, `requireSession`, centralized permissions.
- Produces: accessible `/login` and protected `/interno` shell with responsive drawer/sidebar.

- [ ] **Step 1: Generate and review the Salesboard Macro 1 design system**

Run the local `ui-ux-pro-max` design-system and stack searches for a sober B2B SaaS admin product, then persist only verified tokens relevant to this interface. Use semantic CSS variables required by the master specification.

- [ ] **Step 2: Write failing UI tests**

Test labels, password-manager-compatible fields, generic login errors, disabled submit, keyboard focus, loading/error states, Turnstile token forwarding and accessible mobile navigation.

- [ ] **Step 3: Run UI tests and verify RED**

Run: `npm.cmd test -- src/components/auth/login-form.test.tsx`.

- [ ] **Step 4: Implement the accessible shell**

Use server components by default; isolate only interactive forms/menu as client components. Apply semantic tokens for background, surface, foreground, muted, border, accent, success, warning and danger; preserve 4.5:1 text contrast and reduced-motion behavior.

- [ ] **Step 5: Verify GREEN and commit**

```bash
git add src/app src/components src/app/globals.css
git commit -m "feat: add login and internal application shell"
```

### Task 7: Client list, create, edit, detail, and status UI

**Files:**
- Create: `src/app/(internal)/interno/clientes/page.tsx`
- Create: `src/app/(internal)/interno/clientes/novo/page.tsx`
- Create: `src/app/(internal)/interno/clientes/[clientId]/page.tsx`
- Create: `src/app/(internal)/interno/clientes/[clientId]/editar/page.tsx`
- Create: `src/components/clients/client-table.tsx`, `src/components/clients/client-form.tsx`, `src/components/clients/client-status-actions.tsx`
- Create: `src/components/clients/client-form.test.tsx`
- Create: `tests/e2e/macro-1.spec.ts`

**Interfaces:**
- Consumes: client service DTOs and authenticated server mutations.
- Produces: list/create/edit/detail/disable/archive UI for `super_admin`; LinkedIn is visible as future/disabled and is never enabled.

- [ ] **Step 1: Write failing component and E2E tests**

The E2E scenario signs in as a seeded test-only super admin, opens `/interno`, creates a fictional company, edits it, disables it, logs out, and confirms an unauthorized test user receives 403/404 for direct client access.

- [ ] **Step 2: Run tests and verify RED**

Run: `npm.cmd test -- src/components/clients/client-form.test.tsx` and `npm.cmd run test:e2e -- tests/e2e/macro-1.spec.ts`.

- [ ] **Step 3: Implement responsive client management**

Use server-side pagination/search parameters, progressive-enhancement forms, inline validation, confirmation dialog for status changes, desktop table and compact mobile cards, plus loading/empty/error/success states.

- [ ] **Step 4: Verify GREEN and commit**

```bash
git add src/app src/components/clients tests/e2e
git commit -m "feat: add internal client management interface"
```

### Task 8: Security headers, CORS, error boundaries, and audit completion

**Files:**
- Create: `src/lib/security/headers.ts`, `src/lib/security/headers.test.ts`
- Create: `src/lib/security/origin.ts`, `src/lib/security/origin.test.ts`
- Create: `src/lib/logging/logger.ts`, `src/lib/logging/logger.test.ts`
- Create: `src/proxy.ts`
- Modify: auth/client mutations and audit hooks created earlier

**Interfaces:**
- Produces: `securityHeaders(nonce?)`, `assertTrustedOrigin(request, allowedOrigins)`, `safeLog(event)`.
- Produces audit events `login_success`, `login_failure`, `logout`, `user_created`, `user_disabled`, `user_role_changed`, plus client events already implemented.

- [ ] **Step 1: Write failing security tests**

Cover CSP, frame denial, MIME sniff prevention, referrer policy, noindex headers, strict same-origin mutation checks, CORS allowlist, redaction of passwords/tokens/cookies/private keys, generic browser errors and complete required audit events.

- [ ] **Step 2: Run tests and verify RED**

Run: `npm.cmd test -- src/lib/security src/lib/logging tests/integration`.

- [ ] **Step 3: Implement defense-in-depth**

Apply route matching only for headers/early redirects; keep authorization inside every server handler/action. Redact structured logs by key allowlist and include `request_id`, `user_id`, `client_id`, `operation`, `status`, `duration_ms`, and `error_code`.

- [ ] **Step 4: Run full tests and commit**

```bash
git add src/lib/security src/lib/logging src/proxy.ts src/lib/auth src/features/audit tests
git commit -m "feat: harden macro one security controls"
```

### Task 9: CI, operations documentation, and final verification

**Files:**
- Create: `.github/workflows/ci.yml`
- Create: `README.md`, `docs/ARCHITECTURE.md`, `docs/DEPLOYMENT.md`, `docs/SECURITY.md`
- Modify: `.dev.vars.example`, `package.json`, `wrangler.jsonc`

**Interfaces:**
- Produces: reproducible local setup, local D1 migrations, test/build commands, staging/production deployment steps and documented architectural decisions.

- [ ] **Step 1: Add CI workflow**

Pin Node to the supported LTS range selected by the scaffold, run `npm ci`, `npm run lint`, `npm run typecheck`, `npm test -- --run`, and `npm run build`; never embed Cloudflare credentials in the workflow.

- [ ] **Step 2: Write required documentation**

Document prerequisites, `.dev.vars`, D1 creation/binding/migrations, bootstrap invitation, local test keys for Turnstile, commands, staging/production separation, Cloudflare secrets, custom domains, security model, vinext decision, Better Auth decision, recovery/rollback and explicit Macro 2 exclusions.

- [ ] **Step 3: Run secret and scope scans**

Run searches for private keys, tokens, real Spreadsheet IDs, Deliziare, Google Sheets imports, sync jobs, media dashboards, `R2Bucket`, and unsafe role checks outside `permissions.ts`. Expected: no secret, real-client fixture, or Macro 2 implementation is present.

- [ ] **Step 4: Run fresh complete verification**

Run in order:

```text
npm.cmd ci
npm.cmd run db:migrate:local
npm.cmd run lint
npm.cmd run typecheck
npm.cmd test -- --run
npm.cmd run test:e2e
npm.cmd run build
npm.cmd run preview:smoke
```

Expected: every command exits 0; report exact test counts and any environmental limitation instead of inferring success.

- [ ] **Step 5: Commit documentation and CI**

```bash
git add .github README.md docs .dev.vars.example package.json package-lock.json wrangler.jsonc
git commit -m "docs: add macro one operations and deployment guides"
```

- [ ] **Step 6: Deliver the required Macro 1 report and stop**

Report exactly: implementation summary, project structure, migrations, routes, roles/permissions, verification results, local commands, missing variables, Cloudflare deployment steps, risks/pending points, and an exact Macro 2 recommendation. Do not implement Macro 2.
