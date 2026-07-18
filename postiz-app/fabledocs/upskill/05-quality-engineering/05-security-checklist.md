# Security Checklist

Each item mapped to real code. Ends with a pre-merge checklist you can actually run.

## Authentication
- JWT in cookie/header, verified in [auth.middleware.ts L33-L48](../../../apps/backend/src/services/auth/auth.middleware.ts#L33-L48). **No expiry** ([signJWT L42-L44](../../../libraries/helpers/src/auth/auth.service.ts#L42-L44)) — no revocation, stale claims. bcrypt for passwords ([hashPassword L36-L40](../../../libraries/helpers/src/auth/auth.service.ts#L36-L40)) ✓.
- Cookie flags gated by `NOT_SECURED` ([L14-L20](../../../apps/backend/src/services/auth/auth.middleware.ts#L11-L25)); self-hosters on http must set it or lose their session (and secure cookies won't set) — a misconfig, not a vuln, but a support-ticket generator.
- Super-admin impersonation via cookie ([L50-L76](../../../apps/backend/src/services/auth/auth.middleware.ts#L50-L76)) — powerful; ensure it's audited.

## Authorization / IDOR (the weak area)
- Tenant isolation by convention (`orgId` in where-clauses); **confirmed IDOR** at [editTag L804-L814](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L804-L814). Run the IDOR audit in [03-validation-auth](../03-architecture-and-patterns/03-validation-auth-and-permissions.md).
- Plan gates can be **path-bypassed** ([permissions.guard L24-L31](../../../apps/backend/src/services/auth/permissions/permissions.guard.ts#L24-L31)); fail-open controller list ([api.module L48-L109](../../../apps/backend/src/api/api.module.ts#L48-L109)).
- `functionIntegration` dispatches an arbitrary `body.name` method on a provider ([integrations.controller L341-L352](../../../apps/backend/src/api/routes/integrations.controller.ts#L341-L352)) — *investigate*: is `body.name` constrained to safe method names? A provider method name reachable from user input is a potential over-reach.

## Input validation
- DTO + class-validator for typed bodies; **but** the flagship `POST /posts` takes `any` ([posts.controller L174](../../../apps/backend/src/api/routes/posts.controller.ts#L170-L179)) and re-validates manually. Missed a spot = unvalidated input.
- Content sanitized on transform ([sanitizePostContent](../../../libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts#L31-L36)) — check its rules for the XSS surface below.

## XSS
- Post content is HTML (tiptap editor), sanitized via [sanitizePostContent](../../../libraries/helpers/src/utils/sanitize.post.content.ts) + `isomorphic-dompurify` (dep). Verify every render path uses sanitized content; drafts skip settings validation but content is still transformed.
- Frontend: audit any `dangerouslySetInnerHTML` in preview components ([components/preview](../../../apps/frontend/src/components/preview)).

## SSRF
- Outbound fetches to user-controlled URLs: webhooks ([sendWebhooks](../../../apps/orchestrator/src/activities/post.activity.ts#L310-L336)), the connect-time `webhookUrl` POST ([no.auth… L269-L284](../../../apps/backend/src/api/routes/no.auth.integrations.controller.ts#L269-L284)), media fetch-by-URL. The **public API** uses an `ssrfSafeDispatcher` ([public.integrations.controller L37](../../../apps/backend/src/public-api/routes/v1/public.integrations.controller.ts#L37)) — good; check whether the *internal* webhook sender has the same protection (it appears not — *investigate*, likely a gap).

## CSRF
- Cookie auth + `credentials: 'include'` ([custom.fetch.func L47](../../../libraries/helpers/src/utils/custom.fetch.func.ts#L46-L74)); CORS origin allowlist ([main.ts L42-L47](../../../apps/backend/src/main.ts#L22-L48)). State-changing GETs? Audit — most mutations are POST/PUT/DELETE ✓. The `auth`-header alternative to cookies mitigates classic CSRF for API clients.

## Injection
- Prisma parameterizes queries — SQL injection surface is low. Watch any raw queries (grep `$queryRaw`).
- The `(post:id)` reference resolver builds a `RegExp` from ids ([updateTags L568-L570](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L566-L572)) — ids are uuids so ReDoS is unlikely, but note the pattern.

## Secrets at rest (top risk)
- **Social OAuth tokens plaintext** ([schema L322-L325](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L314-L356)); third-party apiKeys and custom instance details *are* encrypted (`fixedEncryption`). Inconsistent, and the wrong thing is unprotected. R1.
- Encryption uses a **legacy fixed-IV AES** ([helpers auth.service L22-L34](../../../libraries/helpers/src/auth/auth.service.ts#L9-L34)) derived from `JWT_SECRET` — fixed IV means identical plaintexts produce identical ciphertexts (pattern leakage); acceptable for opaque keys, weak in general. *Investigate* whether a modern AEAD (GCM + random IV) is warranted.

## Dependency risk
- pnpm `onlyBuiltDependencies: [bcrypt]` ([package.json L330-L332](../../../package.json#L330-L332)) blocks arbitrary install scripts ✓. Large surface (200+ deps in one list); `pnpm audit` in CI recommended.

## Uploads
- Magic-byte validation ([CustomFileValidationPipe](../../../libraries/nestjs-libraries/src/upload/custom.upload.validation.ts#L21-L41)) — doesn't trust mimetype ✓; size caps per type. Public API additionally MIME-allowlists ([L42-L52](../../../apps/backend/src/public-api/routes/v1/public.integrations.controller.ts#L42-L52)).

## Webhooks (incoming vs outgoing)
- Incoming Stripe: signature-verified against raw body ([stripe.controller L19-L25](../../../apps/backend/src/api/routes/stripe.controller.ts#L18-L25)) ✓.
- Outgoing app webhooks: **unsigned** — receivers can't verify us (R9). Add HMAC + timestamp.

## Rate limiting
- Global 90 req/h/key, Redis-backed, proxy-aware ([app.module L35-L43](../../../apps/backend/src/app.module.ts#L35-L43), [ThrottlerBehindProxyGuard](../../../libraries/nestjs-libraries/src/throttler/throttler.provider.ts)) ✓. Per-provider concurrency caps protect social APIs.

## Cookies
- httpOnly + secure + sameSite='none' when secured ([auth.middleware removeAuth L11-L23](../../../apps/backend/src/services/auth/auth.middleware.ts#L11-L25)); domain from `FRONTEND_URL`.

---

## Pre-merge security checklist (run this on every PR)

- [ ] New repository query? Does its `where` include `organizationId` (or is it a documented safe-internal call)?
- [ ] New controller? Is it in `authenticatedController` (or intentionally public with its own auth)?
- [ ] New `@Body()` — is it a typed DTO, not `any`?
- [ ] New outbound `fetch` to a user-influenced URL — SSRF-guarded?
- [ ] New rendered user content — sanitized before render?
- [ ] New secret stored — encrypted? (match the third-party pattern, not the social-token gap)
- [ ] New endpoint mutating state — POST/PUT/DELETE, not GET?
- [ ] New env-gated behavior — documented, and safe default?
- [ ] New external call in a retried activity — idempotent / signed?

Interview angle: "walk me through securing a multi-tenant SaaS" → tenant isolation (with the IDOR example), secrets at rest (the token gap), webhook signing (in vs out), upload validation (magic bytes). This checklist *is* a strong answer to "what do you look for in a security review?"
