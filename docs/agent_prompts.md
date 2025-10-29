# Agentic Prompts for ZBV Identity & KYC Platform

> **Note**: These prompts are intended for use with an agentic-mode workspace that can generate production-grade Laravel code. They capture the global conventions and individual phase objectives that drive the build-out of the ZBV platform.

## 0. Global System Prompt (use once per session)

```
You are a senior Laravel architect. Follow SOLID, DDD-ish boundaries, and Laravel 11 best practices.
Stack: PHP 8.3, MySQL 8, Redis, Horizon, Octane optional. Use strict types, request validation, policies, form requests, resources, feature tests, Pest, and factories.
Security: TLS assumed, HMAC signatures for API, Argon2id passwords, encryption for sensitive fields. Log only non-sensitive metadata. Implement maker-checker for sensitive flows.

Project: “ZBV Identity & KYC Platform” supporting Individuals (ZBV IDs), Companies (ZBC IDs), AML/PEP screening, Holds/Watchlist, API Credits billing, and multi-tenant access (banks/fintechs/corporates).

Coding rules:
- Use PHP 8.3 readonly properties where helpful and typed properties everywhere.
- Use route model binding, API Resources, and Form Requests.
- Use Enum classes for statuses/reason codes.
- Use Policies for authorization. Gate sensitive endpoints by role + tenant.
- Create migrations, seeders, models, factories, observers, tests.
- Add docblocks and PHPStan-friendly types. Target PHPStan level 6+.

Deliverables always include: changed files, migration names, sample .env keys, and ✅ tests.
```

## 1. Project Bootstrap & Core Scaffolding

```
Objective: Bootstrap a fresh Laravel 11 project “zbv-platform” with API-first setup.

Scope:
- Install Laravel 11, Pest, PHPStan, Laravel Pint, Sanctum or Laravel Passport (choose Sanctum) for session auth; custom HMAC middleware for API clients; Laravel Horizon.
- Create modules (folders/namespaces) for: Identity, Companies, Screening, Watchlist, Holds, Billing, ApiAuth, Admin, Audit, Reporting.
- Configure multi-tenancy (simple: tenant_id on core tables + middleware that resolves tenant from API key header `X-ZBV-API-KEY` or from authenticated user).
- Add base enums: SubjectType {INDIVIDUAL, COMPANY}, RiskLevel {LOW, MEDIUM, HIGH, CRITICAL}, HoldType {SOFT, HARD}, HoldStatus, WatchlistStatus, ReasonCode {PEP_MATCH, SANCTION_HIT, ADVERSE_MEDIA, …}.
- Add global exception handler returning RFC 7807 problem+json.

Acceptance:
- `php artisan test` passes initial smoke tests
- Routes: `/health`, `/v1/ping` return json
- HMAC middleware stub compiled and unit-tested

Deliverables:
- Commands to run
- composer.json updates
- config/zbv.php with toggles (tenancy, hmac, rate limits)
- tests/Feature/Bootstrap/…
```

## 2. Data Model & Migrations (Phase 1 Core Entities)

```
Objective: Implement migrations, models, factories for:
- Individual (zbv_id, names, dob, nat_id [encrypted], phone, email, address json, status enum, risk_level, risk_score, last_screened_at, tenant_id)
- BiometricTemplate (individual_id FK, type enum, template_hash, liveness_score, vendor_ref, encrypted_blob nullable)
- Company (zbc_id, rc_no, tin [encrypted], legal_name, trade_name, address json, industry, status, risk_level, risk_score, last_screened_at, tenant_id)
- CompanyOfficer (company_id, individual_id, role enum [DIRECTOR, SIGNATORY, UBO], ownership_pct, kyc_status)
- WatchlistEntry (subject_type, subject_id, status, reason_code, evidence_url, maker_id, checker_id, tenant_id)
- Hold (subject_type, subject_id, hold_type, reason_code, severity int, scope enum [GLOBAL,TENANT], notes encrypted, placed_by, approved_by, expires_at, next_review_at, status)
- ScreeningResult (subject_type, subject_id, pep_tier, sanction_matches int, sanction_sources json, adverse_media_count int, aml_score int, risk_level, provider, raw_ref, created_at)
- ApiClient (tenant_type enum [BANK, FINTECH, CORPORATE], name, contact_email, status)
- ApiKey (api_client_id, key_ref, public_key, secret_hash, scopes json, ip_allowlist json, status)
- CreditWallet (api_client_id, balance decimal(18,2), currency)
- CreditTransaction (wallet_id, type enum [TOPUP,DEBIT,ADJUST], amount decimal(18,2), product_code, ref, metadata json)
- ApiUsage (api_client_id, endpoint, credits_debited, status_code, duration_ms, request_hash, created_at)
- WebhookEndpoint (api_client_id, url, secret, status)
- Invoice (api_client_id, amount, currency, status, due_date, pdf_url)
- AuditLog (actor_id, actor_role, action, entity, entity_id, ip, user_agent, hash, created_at)
- StatusChange (subject_type, subject_id, from_status, to_status, reason, actor_id, notes, created_at)

Rules:
- Use UUIDs for primary keys except for pivot-like tables; add indexes for lookups.
- Use soft deletes where sensible.
- Add foreign keys with cascade rules.
- Encrypt national IDs, TIN, notes with Laravel’s Eloquent encryption casts.

Acceptance:
- Migrations run clean.
- Factories + seeders for samples.
- Pest tests for schema integrity.

Deliverables:
- migrations/, app/Models/, database/factories/, database/seeders/, tests/Feature/DatabaseSchemaTest.php
```

## 3. API Authentication: API Keys + HMAC + Rate Limits

```
Objective: Implement API key issuance, HMAC verification, and rate limiting.

Scope:
- Endpoint: POST /v1/auth/api-keys → create key pair (key_ref, public_key, secret shown once), assign scopes, tenant binding; Admin/ TenantAdmin only.
- Middleware: VerifyApiKey (checks key status, scopes, tenant), VerifyHmacSignature (headers: X-ZBV-Key, X-ZBV-Signature, X-ZBV-Timestamp; signature over canonical request).
- Throttling: per API key + per tenant configurable (config/zbv.php).
- Usage logging: ApiUsage model on success/failure.

Acceptance:
- Invalid signature rejected (403).
- Expired timestamp rejected (skew > 5m).
- Rate limit headers present.

Deliverables:
- Controllers, middleware, tests (happy/negative cases).
- Example client snippet (PHP + curl) in /docs/clients.md
```

## 4. Credits & Billing Engine

```
Objective: Implement credit wallets, products, burn rules, top-ups.

Scope:
- Configurable products & prices: VERIFY_BASIC=1, VERIFY_BIOMETRIC=3, VERIFY_COMPANY=2, WATCHLIST_CHECK=1, RESCREEN=1
- Debit-on-accept semantics; auto-refund on provider failure.
- Endpoints:
  - GET /v1/credits/wallet
  - POST /v1/credits/topup (returns invoice/payment-intent stub)
  - GET /v1/credits/usage?from&to&endpoint
- Auto-recharge rules (tenant settings): threshold & target.

Acceptance:
- Debits are atomic + idempotent (request_hash).
- Prevent negative balance unless grace_mode enabled.
- Tests cover concurrency (parallel requests).

Deliverables:
- Services/CreditService.php
- Policies to restrict credit ops
- Feature tests for debit/refund/threshold
```

## 5. Individual Enrolment & Verification Endpoints

```
Objective: Implement core KYC flows for individuals.

Scope:
- POST /v1/individuals/enrol (FormRequest validation; issue unique zbv_id; create enrolment receipt JSON signed)
- POST /v1/individuals/verify (supports ZBV ID + OTP OR ZBV ID + biometric template hash placeholder; returns match score, status, risk_level, watchlist summary)
- GET /v1/individuals/{zbv_id} (resource with policy checks)

Acceptance:
- Duplicate detection check (nat_id + dob) flagged.
- Enrolment receipt is signed (HMAC over payload) and verifiable.
- Credit burn: VERIFY_BASIC or VERIFY_BIOMETRIC as configured.

Deliverables:
- Controllers, Requests, Resources, Policies
- Example requests/responses in tests and /docs/api.md
```

## 6. Company Registration, Officers, and Verification

```
Objective: Implement company lifecycle and linkage to signatories.

Scope:
- POST /v1/companies (create ZBC ID, store RC/TIN, docs placeholders)
- POST /v1/companies/{zbc_id}/officers (link existing ZBV IDs as DIRECTOR/SIGNATORY/UBO)
- POST /v1/companies/verify (RC/TIN → company profile, linked signatories summary, risk)

Acceptance:
- Officer roles validated; duplicates prevented.
- Company verify burns VERIFY_COMPANY credits.
- Policies ensure tenant isolation.

Deliverables:
- Controllers/Requests/Resources/Policies + tests
```

## 7. Holds, Watchlist, and Maker–Checker

```
Objective: Implement AML/PEP flagging with soft/hard holds and dual control.

Scope:
- POST /v1/watchlist (maker) → pending watchlist entry
- POST /v1/watchlist/{id}/approve (checker) → active
- POST /v1/holds (maker) → create hold (soft/hard) with reason_code, scope, expires_at, next_review_at
- POST /v1/holds/{id}/approve | POST /v1/holds/{id}/release
- GET /v1/holds?subject_type&subject_id
- StatusChange records on every transition.
- Notifications to tenant admins (queue + mail stub).

Acceptance:
- Maker cannot approve own item; checker required.
- Holds enforce behavior (policy: blocked endpoints when HARD; limited when SOFT).
- Complete audit logs for each action (hash-chained note entries).

Deliverables:
- Controllers/Policies/Events/Listeners
- Tests for workflows, permissions, and blocked flows
```

## 8. Screening Integration (PEP/Sanctions/Adverse Media)

```
Objective: Create a provider abstraction and basic “mock provider” for screenings.

Scope:
- Contracts/ScreeningProvider.php (methods: screenIndividual, screenCompany, rescreen)
- Implement MockProvider that returns deterministic results based on name hash.
- Service: ScreeningService orchestrates calls, maps to ScreeningResult, updates risk_level/score.
- Endpoints:
  - POST /v1/individuals/{zbv_id}/rescreen
  - POST /v1/companies/{zbc_id}/rescreen
- Nightly job: queued re-screen for subjects with due `next_review_at`.

Acceptance:
- Results persisted and surfaced via verification responses.
- Credit burn: WATCHLIST_CHECK or RESCREEN.
- Failover strategy: if provider down → auto-refund debit, return 503 with remediation.

Deliverables:
- Contracts, Services, Jobs, Console kernel schedule
- Tests for mapping logic and failure handling
```

## 9. Webhooks & Notifications

```
Objective: Deliver signed webhooks and email/SMS events.

Scope:
- WebhookEndpoint registration & verification
- Dispatcher job with retries (exponential backoff), HMAC header `X-ZBV-Signature`
- Events: VerificationCompleted, HoldPlaced, HoldReleased, LowCreditWarning, InvoiceCreated
- /v1/webhooks/test endpoint

Acceptance:
- Signature verified by a sample consumer (include example code)
- Dead-letter queue for failed webhooks

Deliverables:
- Events, Listeners, Jobs, config/webhooks.php, tests
```

## 10. Reporting & Analytics

```
Objective: Implement initial reports + CSV export.

Scope:
- Endpoints:
  - GET /v1/reports/operations?from&to (enrolments by day, verifications, match rates)
  - GET /v1/reports/compliance?from&to (watchlist adds/removals, holds, SLA timings)
  - GET /v1/reports/billing?from&to (credit usage by endpoint/client)
- CSV export using Laravel’s built-in streams.

Acceptance:
- All reports scoped by tenant unless SuperAdmin.
- Large ranges stream without memory blowups.

Deliverables:
- Controllers/Services, tests, sample CSVs in tests/fixtures
```

## 11. Admin & RBAC

```
Objective: Role system and policy enforcement.

Scope:
- Roles: SUPER_ADMIN, TENANT_ADMIN, COMPLIANCE, OPERATOR, AUDITOR
- Policies for each entity; Gate checks; seeds for demo users.
- Admin UI (API-first but basic Blade or Inertia for internal ops is OK).

Acceptance:
- Each protected endpoint requires correct role + tenant.
- Policy test matrix covers allow/deny.

Deliverables:
- Policies, Seeders, tests/Feature/AuthorizationTest.php
```

## 12. Observability & Audit

```
Objective: Centralized auditing and metrics.

Scope:
- AuditLog writes for sensitive actions (holds, watchlist, status changes) with request id + hash chain.
- Prometheus metrics: request count, duration, credits debit count, provider failures.
- Structured logs (JSON) with correlation ids.

Acceptance:
- Metrics endpoints exposed; basic Grafana dashboard JSON generated.
- Audit logs tamper-evident (hash chain test).

Deliverables:
- Middleware for correlation id
- ObservabilityService + tests
```

## 13. OpenAPI & SDK Stubs

```
Objective: Generate OpenAPI 3.1 spec and basic SDK clients.

Scope:
- openapi/zbv-v1.yaml covering all endpoints, security headers, examples.
- Autogenerate minimal PHP and TypeScript SDKs (simple classes with HMAC signing).

Acceptance:
- `swagger-cli validate` passes
- Example snippets compile and pass basic integration test against app

Deliverables:
- openapi/zbv-v1.yaml
- sdk/php/, sdk/ts/, tests for SDK smoke
```

## 14. CI/CD & Quality Gates

```
Objective: Reliable pipeline with quality checks.

Scope:
- GitHub Actions: run composer install, Pint, PHPStan, Pest, build, artifact.
- Security: dependabot, php-security-checker (or symfony CLI audit).
- Container build (Dockerfile), multi-stage, healthcheck.
- .env.example with all required keys.

Acceptance:
- Pipeline green.
- Docker image runs: `php artisan migrate --force` and `php artisan horizon`.

Deliverables:
- .github/workflows/ci.yml
- Dockerfile, docker-compose.yml for local
```
