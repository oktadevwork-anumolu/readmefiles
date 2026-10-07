# Copilot Instructions — OutSystems → Next.js Rewrite

> Place this file at `.github/copilot-instructions.md` in the repo root.
> Copilot Chat, Copilot code review, and the Copilot coding agent read it automatically.

## 1. Project context

- We are rewriting an internal OutSystems application as a **Next.js (App Router) + React + TypeScript** web app.
- Authentication is **Okta OIDC**.
- The Next.js server is the **Backend-for-Frontend (BFF)**. The browser never holds OAuth tokens.
- The UI calls internal backend REST APIs **only through server-side code** in this app.
- Deployment targets are **AWS Amplify Hosting** or **AWS ECS Fargate (container behind an ALB)**. All code must run on both targets.
- **Next.js version: pin to 15.x** while Amplify is a target. Amplify Hosting documents SSR support through Next.js 15. Use `middleware.ts`, not `proxy.ts` (the Next.js 16 name). Do not use Edge runtime features: no `export const runtime = 'edge'`.
- Migration is incremental, screen by screen. OutSystems serves all routes not yet migrated.

## 2. Non-negotiable security rules

Copilot must follow these in every suggestion.

1. **No tokens in the browser.**
   - Never put Okta access, refresh, or ID tokens in `localStorage`, `sessionStorage`, JS-readable cookies, React state, props passed to Client Components, or `NEXT_PUBLIC_*` variables.
2. **Session model.**
   - The browser holds only an opaque session ID in cookie `__Host-sid`, with attributes `HttpOnly; Secure; SameSite=Lax; Path=/`.
   - Tokens live server-side in the session store (DynamoDB with TTL, or ElastiCache Redis), encrypted.
   - Session access is only through `@app/auth`.
3. **Server-only modules.**
   - Any module that touches secrets, tokens, the session store, or backend APIs must start with `import 'server-only';`.
4. **Middleware is not the security boundary.**
   - `middleware.ts` may do only these things:
     - set the request ID;
     - set security headers and the CSP nonce;
     - redirect to login when there is no session cookie.
   - **Authorization is enforced server-side in every Server Component data load, Route Handler, and Server Action**, via `requireUser()` / `requirePermission()` from `@app/auth`.
   - Reason: middleware-only auth has been bypassed before (CVE-2025-29927).
5. **Server Actions and Route Handlers are public endpoints.**
   - Each one must:
     - (a) authenticate;
     - (b) authorize the specific action and resource;
     - (c) validate all input with Zod;
     - (d) return typed, non-leaky errors.
6. **CSRF protection.**
   - Server Actions rely on Next.js origin checks. Configure `serverActions.allowedOrigins` explicitly.
   - Custom state-changing Route Handlers (POST/PUT/PATCH/DELETE) must verify the `Origin` header against the allowlist.
7. **Content Security Policy.**
   - Use a strict, nonce-based CSP generated in middleware.
   - No `unsafe-inline` or `unsafe-eval` in production.
   - Also set:
     - `frame-ancestors 'none'`;
     - HSTS;
     - `X-Content-Type-Options: nosniff`;
     - `Referrer-Policy: strict-origin-when-cross-origin`.
8. **Secrets.**
   - Never hardcode secrets.
   - Load runtime secrets (Okta client credentials/private key, session encryption key) from AWS Secrets Manager via `@app/config`.
   - Amplify SSR environment variables are written into the build output, so **do not** put secrets there.
9. **Input and output handling.**
   - Never use `dangerouslySetInnerHTML` unless the content is sanitized (DOMPurify on the server).
   - Never build URLs or SQL from unvalidated input.
   - Never redirect to a user-supplied URL without allowlist validation (open-redirect protection, e.g. `returnTo`).
10. **Logging hygiene.**
    - Never log tokens, cookies, `Authorization` headers, secrets, or full request bodies.
    - The logger redacts these by default; do not bypass it.

## 3. Okta integration (implemented in `packages/auth`)

- **Library:** `openid-client` (v6). Do not add other auth libraries without an ADR.
- **Flow:** Authorization Code + PKCE (S256), with `state` and `nonce`.
  - Store the transaction data in a short-lived encrypted `__Host-oidc-tx` cookie: `HttpOnly; Secure; SameSite=Lax; Max-Age=600`.
- **Client type:** confidential web client.
  - Prefer `private_key_jwt` client authentication over a client secret.
- **Authorization server:** Okta **custom authorization server**.
  - Request only the scopes needed (`openid profile email offline_access` plus API scopes).
  - The API audience must match the backend's expected `aud`.
- **Routes:**
  - `GET /auth/login?returnTo=` — validate `returnTo` (relative path only).
  - `GET /auth/callback` — exchange the code, validate the ID token (`iss`, `aud`, `nonce`, expiry), create the session, rotate the session ID.
  - `POST /auth/logout` — delete the server session, clear the cookie, redirect to the Okta end-session endpoint with `id_token_hint` and `post_logout_redirect_uri`.
- **Token refresh:**
  - Server-side, shortly before access-token expiry.
  - Single-flight per session, so concurrent requests don't double-refresh.
  - Refresh-token rotation is enabled in Okta.
  - On refresh failure, destroy the session and send the user back to login.
- **Authorization model:**
  - Map Okta group or role claims to app permissions in `packages/auth/src/permissions.ts`.
  - UI checks are for UX only. **Backend APIs re-validate the access token and scopes.**
- **Session timeouts:**
  - Idle timeout and absolute lifetime are configurable via `@app/config`.

## 4. Repository layout (pnpm workspaces + Turborepo)

```
apps/
  web/                         # Next.js app (BFF + UI)
    src/app/                   # App Router routes
      (public)/                # unauthenticated pages (login landing, error)
      (app)/                   # authenticated area; layout calls requireUser()
      auth/                    # login/callback/logout route handlers
      api/
        health/route.ts        # unauthenticated liveness probe, returns no data
        client-logs/route.ts   # authenticated, rate/size-limited browser error intake
    src/features/<feature>/    # one folder per migrated OutSystems module
      components/              # feature UI (Client/Server Components)
      server/                  # server-only loaders and actions for this feature
      schemas.ts               # Zod schemas for this feature
      *.test.ts(x)             # co-located unit tests
    src/middleware.ts
    instrumentation.ts         # OpenTelemetry bootstrap (server)
    e2e/                       # Playwright tests
    Dockerfile                 # ECS image (Next.js standalone output)
    amplify.yml                # Amplify build spec
packages/
  auth/          # server-only: OIDC, session store, requireUser/requirePermission
  api-client/    # server-only: typed backend client generated from OpenAPI
  logger/        # structured logging (server: pino; client: error reporter)
  config/        # Zod-validated env + Secrets Manager loader (server-only parts split)
  utils/         # pure, framework-agnostic helpers (dates, formatting, guards, Result type)
  ui/            # design system (shadcn/ui on Radix + Tailwind); no data fetching
  test-utils/    # Vitest setup, RTL render helpers, MSW handlers, fixtures, factories
  eslint-config/ # shared ESLint config incl. import-boundary rules
  tsconfig/      # shared tsconfig bases
infra/
  cdk/           # AWS CDK (TypeScript): AmplifyStack, EcsStack, shared (WAF, DynamoDB sessions, secrets)
docs/
  adr/           # Architecture Decision Records
  migration-tracker.md  # OutSystems screen → route mapping and status
```

### Dependency rules (enforced by ESLint)

- `packages/utils` depends on nothing internal and has no React, Node-only APIs, or I/O.
- `packages/ui` may depend on `utils` only. It must not import `auth`, `api-client`, `config`, or `logger/server`.
- `packages/auth` and `packages/api-client` are **server-only**.
  - Never import them from a file with `'use client'`.
  - May depend on `config`, `logger`, and `utils`.
- `apps/web/src/features/*` may import from any package, but **must not import from another feature**. Shared code moves to a package.
- Client Components receive plain serializable data only, never clients, sessions, or tokens.

## 5. Calling backend APIs

- Use only `@app/api-client`.
  - Types are generated from the backend OpenAPI specs with `openapi-typescript`.
  - Calls go through `openapi-fetch`.
- Never call backend URLs with raw `fetch` from features.
- The client wrapper:
  - attaches `Authorization: Bearer <access token>` from the session (server-side);
  - propagates `X-Request-Id`;
  - applies timeouts via `AbortSignal.timeout`;
  - retries **only idempotent GETs** with backoff;
  - maps non-2xx responses to a typed `ApiError` (`status`, `code`, `requestId`) without leaking backend messages to the UI.
- Data loading:
  - Prefer Server Components for reads.
  - Use Server Actions (or Route Handlers) for mutations.
  - Use TanStack Query in Client Components only when interactive client-side refetching is needed, pointed at our own Route Handlers, never at backend APIs.
- Regenerate types with `pnpm --filter @app/api-client generate` when specs change. Never hand-edit generated files.

## 6. Logging and observability (`packages/logger`)

- **Server:** `pino` JSON logs to stdout. Use only `getLogger(module)`; never `console.*` in committed code.
  - Mandatory fields: `level`, `time`, `msg`, `requestId`, `module`, `userId` (opaque subject only, no PII), `route`.
  - The redaction list includes: `authorization`, `cookie`, `set-cookie`, `*.token`, `*.access_token`, `*.refresh_token`, `*.id_token`, `password`, `secret`, `client_secret`.
  - **Levels:**
    - `error` — needs action.
    - `warn` — degraded or unexpected but handled.
    - `info` — business or lifecycle events.
    - `debug` — off in production.
- **Request correlation:**
  - Middleware creates or forwards `X-Request-Id` (UUID v4).
  - The logger reads it via `AsyncLocalStorage`.
  - `api-client` forwards it downstream.
- **Client:**
  - React error boundaries and `reportError()` send minimal, sanitized error events to `/api/client-logs`.
  - No PII, and no full URLs with query strings.
- **Destinations:**
  - ECS: stdout → CloudWatch Logs (awslogs driver or FireLens).
  - Amplify: SSR runtime logs go to CloudWatch automatically.
- **Tracing:** OpenTelemetry via `instrumentation.ts`.
  - On ECS, export to an ADOT collector sidecar (X-Ray/CloudWatch).
  - The tracing code must no-op cleanly when no exporter is configured (Amplify).
- **Audit events** (login, logout, permission denied, privileged mutations) use `logger.audit()` with a fixed event schema.

## 7. Configuration (`packages/config`)

- All environment variables are declared and validated with Zod in `packages/config/src/env.ts`. The app fails fast at startup on invalid config.
- Only variables explicitly marked public may be `NEXT_PUBLIC_*`, and none may be sensitive.
- Secrets are fetched from Secrets Manager at runtime using the task role (ECS) or the SSR compute role (Amplify), then cached in memory with a TTL.

## 8. Testing

- **Unit and component tests:** Vitest + React Testing Library + `@testing-library/user-event`.
  - Co-locate as `*.test.ts` / `*.test.tsx` next to the source.
- **API mocking:** MSW handlers live in `packages/test-utils`. Tests never hit real backends or Okta.
- **Server code:** test loaders, Server Actions, and Route Handlers as plain async functions with mocked `@app/auth` and `@app/api-client`.
- **Mandatory test cases for every Server Action and Route Handler:**
  - (a) unauthenticated → 401 or redirect;
  - (b) authenticated but unauthorized → 403;
  - (c) invalid input → 400 with Zod errors;
  - (d) happy path.
- **End-to-end:** Playwright in `apps/web/e2e`.
  - Uses a dedicated Okta test org/app and a test user from CI secrets. Never use real user credentials.
  - Covers login, logout, session expiry, and one critical flow per migrated feature.
- **Coverage gates (CI):**
  - 80% lines on `packages/utils`, `packages/auth`, `packages/api-client`, `packages/logger`.
  - 70% on features.
- **Test style:** test behavior, not implementation. Use factories from `test-utils`; avoid snapshot tests except for small, stable UI primitives.

## 9. Coding conventions

- TypeScript in `strict` mode, plus `noUncheckedIndexedAccess`. No `any`; use `unknown` and narrow.
- Default to Server Components. Add `'use client'` only when state, effects, or browser APIs are needed, and keep client components leaf-level.
- Forms use React Hook Form + Zod resolver. The same Zod schema is re-validated on the server.
- Errors:
  - Each route segment has `error.tsx` and `not-found.tsx`.
  - User-facing messages are generic; details go to logs with the `requestId`.
- Accessibility: WCAG 2.1 AA. Use Radix primitives and semantic HTML; every interactive element is keyboard-reachable and labeled.
- Naming:
  - kebab-case files;
  - PascalCase components;
  - `camelCase` functions;
  - Zod schemas suffixed `Schema`;
  - server modules under `server/`.
- Use Conventional Commits. Every PR touching auth, session, CSP, or logging redaction requires security review.

## 10. Build and deployment

- `next.config.ts`:
  - `output: process.env.BUILD_TARGET === 'container' ? 'standalone' : undefined`
  - `poweredByHeader: false`
  - explicit `serverActions.allowedOrigins`
  - `images.remotePatterns` allowlist only
- **ECS:**
  - Multi-stage `Dockerfile` with a Node 22 slim base, running as non-root.
  - Copy `.next/standalone`, `.next/static`, and `public`.
  - `HEALTHCHECK` targets `/api/health`.
  - The ALB terminates TLS. Use an internal ALB if the app must not be internet-facing.
  - AWS WAF is attached.
- **Amplify:**
  - `amplify.yml` builds with pnpm/Turborepo.
  - Artifacts base directory is `apps/web/.next`.
  - Node 22 runtime.
  - Non-secret env vars only.
  - AWS WAF is attached.
- **CI (GitHub Actions):** lint → typecheck → unit tests → build → Playwright (preview) → dependency audit, CodeQL, and secret scanning → deploy.

## 11. Migration workflow (per OutSystems module)

1. Add the module to `docs/migration-tracker.md`, listing its screens, entities, server actions, roles, and integrations.
2. Make sure the backend exposes the needed logic as an OpenAPI-described REST API, then regenerate `api-client`.
3. Create `src/features/<module>/` with schemas, server loaders and actions, components, and tests.
4. Port every authorization check from the OutSystems server actions into `requirePermission()` calls. Missing checks are a blocking defect.
5. Switch CloudFront/ALB path routing for that module's routes from OutSystems to the new app behind a feature flag. Run in parallel, verify parity per role, then retire the OutSystems screens.

## 12. What Copilot should NOT generate

- Client-side Okta SDK usage (`@okta/okta-react`, `@okta/okta-auth-js`) in the browser.
- `fetch` calls from Client Components to backend services or Okta.
- Auth decisions made only in middleware or only in UI components.
- `console.log`, ad-hoc loggers, or logging of request/response bodies.
- Edge runtime code, `proxy.ts` (Next.js 16 only), or other Next.js 16-only APIs while pinned to 15.x.
- New dependencies for things already covered by `packages/utils`, `packages/ui`, or the platform.
