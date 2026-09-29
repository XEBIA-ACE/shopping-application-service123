## svc-user-account

Contracts & Interfaces

Affected services and API changes:
- svc-user-account: POST /api/v1/users/register (EXISTING) SHALL be implemented/updated to support email+password registration and confirmation triggering (FR-001..FR-015).
- Email Delivery Service (external): new/used contract `POST /email/v1/send` (or existing equivalent) to request confirmation delivery (FR-011, FR-012). [NEEDS CLARIFICATION: exact endpoint/path and auth mechanism?] (Assumed: HTTP API with service-to-service auth)

API contract (svc-user-account):
- POST /api/v1/users/register
  - Request body: `{ "email": string, "password": string }`
  - Validation: email REQUIRED (FR-002), syntactically valid (FR-003), password REQUIRED (FR-004), password policy (FR-006; assumed minLen=8).
  - Response:
    - 202 Accepted with body `{ "result": "CHECK_EMAIL" }` for both success and already-registered (anti-enumeration) (FR-008, FR-013).
    - 400 Bad Request with field-level errors ONLY for missing/invalid fields (FR-014): e.g. `{ "errors": [{"field":"email","code":"REQUIRED"}] }`.
    - MUST NOT return 409 for existing email if anti-enumeration is enabled (assumption).

Data schemas (DB):
- Table `user_accounts`:
  - columns: `user_id` (pk, uuid/ulid string), `email` (original), `email_normalized` (lowercased), `password_hash`, `status` (ENUM: PENDING_CONFIRMATION, ACTIVE, DISABLED), `created_at`, `updated_at`.
  - indexes: UNIQUE(`email_normalized`) to prevent duplicates (FR-005, FR-007).
- Table `email_confirmations`:
  - columns: `confirmation_id` (pk), `user_id` (fk -> user_accounts.user_id), `email`, `token_hash`, `expires_at`, `sent_at` (nullable), `confirmed_at` (nullable), `status` (ENUM: PENDING_SEND, SENT, CONFIRMED, EXPIRED, SEND_FAILED), `last_send_error` (nullable), `created_at`.
  - indexes: (`user_id`,`status`), UNIQUE(`token_hash`).
[NEEDS CLARIFICATION: token storage requirements—store raw token or hash only?] (Assumed: hash only)

Email Delivery Service interface:
- `EmailDeliveryClient.sendEmail(SendEmailCommand): SendEmailResult`
  - command: `to`, `templateId`/`type` = `EMAIL_CONFIRMATION`, `params` includes `confirmationToken` or confirmationLink, `idempotencyKey`.
  - result: `accepted` boolean, `providerMessageId`, `errorCode`, `errorMessage`.

Test Strategy

Contract tests (HTTP):
- T1 Valid registration returns 202 and CHECK_EMAIL; validates request schema, 202 behavior (FR-001, FR-013, AC-001).
- T2 Missing email returns 400 with email REQUIRED; validates field errors (FR-002, FR-014, EC-001).
- T3 Invalid email returns 400 with INVALID_FORMAT (FR-003, FR-014, EC-002).
- T4 Weak password returns 400 with PASSWORD_POLICY (FR-006, FR-014, EC-003).
- T5 Existing email returns 202 CHECK_EMAIL (no disclosure); validates anti-enumeration response parity (FR-007, FR-008, EC-004).

DB/integration tests:
- T6 On success, one user row with status PENDING_CONFIRMATION and one email_confirmation row with expiry set; validates persistence + expiry (FR-005, FR-009, FR-010).
- T7 Email delivery success marks confirmation SENT with sent_at and stores provider id; validates audit outcome (FR-011, FR-012).
- T8 Email delivery failure keeps user single-created, marks SEND_FAILED with error; validates no duplicate users and failure recording (FR-012, FR-015, EC-005).
- T9 Retry same email does not create second user; confirmation resend behavior [NEEDS CLARIFICATION: resend policy] (Assumed: create new confirmation and attempt send if no unexpired SENT/PENDING exists) (FR-015, EC-006).

Implementation Approach

Core classes/methods (svc-user-account):
- `RegistrationController.register(RegisterUserRequest): ResponseEntity<RegisterUserResponse>` (Spring/WebMVC-like) SHALL map POST /api/v1/users/register.
- `RegistrationService.registerByEmail(String email, String password): RegistrationResult`
  - Steps (single DB transaction):
    1) `EmailNormalizer.normalize(email)` -> lowercased (assumption) (Business rule).
    2) Validate with `EmailValidator` and `PasswordPolicyValidator(minLen=8)` (FR-003, FR-006).
    3) `UserAccountRepository.findByEmailNormalized(...)`.
       - If exists: return generic result; MAY enqueue resend per policy without revealing existence (FR-007, FR-008).
       - If not: create `UserAccount(status=PENDING_CONFIRMATION, password_hash=PasswordHasher.bcrypt(password))` (FR-005, FR-009).
    4) Create `EmailConfirmation(status=PENDING_SEND, token_hash=TokenService.hash(rawToken), expires_at=now+24h)` (FR-010).
    5) Persist both; commit.
  - Post-commit async send: publish `EmailConfirmationSendRequested(confirmationId)` to outbox.

Async/send pattern:
- MUST use Transactional Outbox to prevent duplicates and handle transient email failures without

---

## svc-bff

Contracts & Interfaces
- Public API (svc-bff)
  - POST /bff/v1/registrations/email
    - Request body: RegistrationRequest { email: string, locale?: string, clientContext?: string }
      - svc-bff MUST trim email before validation and downstream calls [NEEDS CLARIFICATION: confirm trimming is acceptable for all clients?] (Assumed: trim).
      - svc-bff MUST treat blank/absent email as validation error (FR-005).
      - svc-bff MUST validate email syntax (e.g., RFC 5322-lite) before delegation (FR-004).
      - locale MAY be provided; if unsupported, svc-bff MUST fall back to default locale (FR-009).
    - 200 Response (generic success for new or existing email; FR-007):
      - RegistrationResult { userId?: string, email: string, status: string, confirmationRequired: boolean, confirmation?: { dispatchStatus: string, correlationId?: string } }
      - status MUST be one of: ACCEPTED, COMPLETED, FAILED_VALIDATION, FAILED_RETRYABLE.
      - confirmationRequired MUST be true when account exists/created (FR-010).
    - 400 Response: validation failure with user-consumable message (FR-013)
      - Error { code: "VALIDATION_ERROR", message: string, fieldErrors?: [{ field: "email", message: string }] }
    - 503 Response: downstream unavailable; no internal details (FR-014)
      - Error { code: "TEMPORARY_UNAVAILABLE", message: string }
- Downstream contracts
  - User Account Service (REST)
    - POST /v1/users { email: string } -> 201 { userId: string, email: string, state: "PENDING_CONFIRMATION"|"ACTIVE" }
    - If email exists: 409 or 200 [NEEDS CLARIFICATION: actual behavior?] (Assumed: 409 conflict).
  - Confirmation dispatch capability
    - [NEEDS CLARIFICATION: separate Notification Service vs user service endpoint] (Assumed: separate Notification Service)
    - POST /v1/confirmations/email { userId: string, email: string, locale?: string, clientContext?: string } -> 202 { correlationId: string, dispatchStatus: "QUEUED"|"SENT" }

Test Strategy
- Contract tests (Spring MockMvc/WebTestClient)
  - T1 validates FR-005/FR-013: POST with missing/blank email returns 400, code=VALIDATION_ERROR, fieldErrors contains “email”.
  - T2 validates FR-004/EC-002: invalid email returns 400 and MUST NOT call downstream (verify no HTTP mocks invoked).
  - T3 validates EC-003: email with spaces is trimmed; downstream receives trimmed email; response email is normalized.
  - T4 validates FR-003/FR-006/AC-001: new email -> User Service 201 then Notification 202; response 200 with confirmationRequired=true and confirmation.correlationId present.
  - T5 validates FR-007/EC-004: existing email (User Service 409) returns 200 generic success (status=ACCEPTED or COMPLETED) and MUST NOT reveal existence.
  - T6 validates FR-014/EC-005: User Service timeout/5xx -> 503 TEMPORARY_UNAVAILABLE; message generic.
  - T7 validates FR-011/EC-006: notification returns 202 queued; response status=ACCEPTED with dispatchStatus=QUEUED.
  - T8 validates FR-012/EC-007: notification 5xx after account creation -> response 200 with confirmationRequired=true, status=ACCEPTED, confirmation.dispatchStatus=FAILED (no internal detail).
  - T9 validates FR-009/EC-008: unsupported locale -> default used; downstream receives default locale.

Implementation Approach
- Data model changes (svc-bff)
  - Add table bff_registration_attempt
    - id (uuid pk), email_hash (char(64)), email_domain (varchar(255)), locale (varchar(16)), client_context (varchar(64)), user_id (varchar(64) null), status (varchar(24)), confirmation_correlation_id (varchar(64) null), created_at (timestamptz)
    - Indexes: idx_bra_email_hash(created_at, email_hash) for diagnostics; idx_bra_created_at(created_at).
  - Rationale: store observable correlation and outcomes (FR-011/FR-012) without storing raw email (minimize PII).

- Core components (Spring Boot, existing patterns)
  - Controller: RegistrationController#createEmailRegistration(@RequestBody RegistrationRequestDto)
  - Validator: EmailRegistrationValidator#validateAndNormalize(dto) -> NormalizedRegistration(email, locale, clientContext)
  - Service: RegistrationOrchestrator#registerByEmail(normalized) -> RegistrationResultDto
    - Algorithm:
      1) validate/normalize; on error throw ValidationException mapped to 400.
      2) call UserAccountClient#createUser(email). If 201: capture userId/state. If 409: treat as generic success (FR-007).
      3) if userId present, call ConfirmationClient#sendEmailConfirmation(userId,email,locale,clientContext).
      4) Persist bff_registration_attempt with status mapping:
         - COMPLETED when both steps succeed and dispatchStatus=SENT
         - ACCEPTED when dispatch queued or email exists or dispatch fails after creation
         - FAILED_RETRYABLE on user service unavailable (mapped to 503)
      5) Return 200 with confirmationRequired=true, include correlationId when available.
  - HTTP clients: UserAccountClient (WebClient

---

## svc-background-worker

## 1) Contracts & Interfaces

### Affected Services and API Changes
- **svc-background-worker** (this change)
- **Account/Registration Service** (caller; no code change required but MUST conform to enqueue contract)
- **Email Delivery Provider** (callee; existing integration)

API changes (svc-background-worker):
- **POST /admin/v1/tasks:enqueue**: EXTEND to accept `task_type=registration_confirmation_email` and validate payload (FR-001/002/003/006/007).
- **GET /admin/v1/tasks/{taskId}**: EXTEND response to include registration-email specific fields and send result (FR-010).
- **POST /admin/v1/tasks/{taskId}:cancel**: EXTEND to cancel registration-email tasks (FR-004, EC-006).

[NEEDS CLARIFICATION: Is event-driven trigger required or is admin/API enqueue sufficient?] (Assumed: enqueue via admin endpoint is sufficient; event consumption MAY be added later.)

### Enqueue Contract (POST /admin/v1/tasks:enqueue)
Request body (new task type):
- `task_type`: MUST equal `registration_confirmation_email`
- `idempotency_key`: SHOULD be provided; if present MUST be unique per registration event (FR-007)
- `requested_by`: string
- `payload` object:
  - `user_id`: MUST be non-empty string (FR-002)
  - `email`: MUST be syntactically valid email (FR-003)
  - `locale`: MAY be string (default `en-US`)
  - `template_id`: MAY be string (default `registration-confirmation:v1`)
  - `correlation_id`: MAY be string
  - `confirmation`: MUST be provided by upstream (FR-006) with either:
    - `token`: string, or
    - `link_url`: string

Responses:
- `202 Accepted` with `{task_id, state}` when enqueued
- `409 Conflict` when `idempotency_key` already exists; MUST return existing `{task_id, state, duplicate_of_task_id}` (EC-003)
- `400 Bad Request` on missing/invalid `user_id/email/confirmation` (EC-001/EC-002)

### Data Model (new tables)
1) `task_requests`
- `task_id` (PK, UUID)
- `task_type` (varchar, index)
- `idempotency_key` (varchar, UNIQUE NULLABLE)
- `payload_json` (jsonb)
- `requested_by` (varchar)
- `requested_at` (timestamptz)
Indexes: `ux_task_requests_idempotency_key` (unique), `ix_task_requests_task_type`

2) `email_confirmation_tasks`
- `task_id` (PK/FK -> task_requests.task_id)
- `user_id` (varchar, index)
- `email` (varchar, index)
- `locale` (varchar)
- `template_id` (varchar)
- `correlation_id` (varchar, index)
- `created_at` (timestamptz)

3) `task_status`
- `task_id` (PK/FK)
- `state` (varchar; values: `queued|in_progress|retry_scheduled|completed|failed_permanent|canceled|duplicate`)
- `attempts` (int)
- `last_error` (text)
- `next_retry_at` (timestamptz, index)
- `updated_at` (timestamptz)

4) `email_send_results`
- `task_id` (PK/FK)
- `provider_message_id` (varchar, nullable)
- `accepted_at` (timestamptz)
- `outcome` (varchar; `accepted|rejected|timeout|error`)

## 2) Test Strategy

- **Contract validation tests** (POST /admin/v1/tasks:enqueue):
  - Missing `payload.user_id` => `400` (validates FR-002, EC-001).
  - Invalid `payload.email="invalid@"` => `400` (validates FR-003, EC-002).
  - Missing `payload.confirmation.token/link_url` => `400` (validates FR-006 assumption).
- **Idempotency test**:
  - Two enqueues same `idempotency_key` => first `202`, second `409` with same `task_id` (validates FR-007, EC-003).
- **Execution/retry tests** (worker loop + provider stub):
  - Provider timeout => status transitions `in_progress -> retry_scheduled`, increments attempts, sets `next_retry_at` (validates FR-008, EC-004).
  - Provider hard reject (e.g., NXDOMAIN) => `failed_permanent` + DLQ marker (validates FR-009, EC-005).
- **Cancel test**:
  - Cancel before execution => `canceled` and provider not called (validates FR-004, EC-006).
- **Operational visibility test**:
  - After success, GET task shows `completed` and `provider_message_id` (validates FR-010, EC-007).

## 3) Implementation Approach

### Core classes/modules
- `TaskEnqueueController`
  - `enqueue(TaskEnqueueRequest req): Response`
  - MUST route `registration_confirmation_email` to `RegistrationEmailTaskValidator`.
- `RegistrationEmailTaskValidator`
  - `validatePayload(payload): void` (email regex, required fields).
- `TaskRepository`
  - `createTaskRequest(...)`, `getByIdempotencyKey(...)`, `getTaskDetail(taskId)`
- `EmailConfirmationTaskRepository`
  - `insert(taskId, userId, email, locale,

---

## SVC-01

Contracts & Interfaces

Affected services: Shopping Application Service (SVC-01); Email Delivery Service (external).

API changes (Shopping Application Service):
- ADD POST /v1/registrations/email
  - Request body: { "email": string }
  - Responses:
    - 202 Accepted: { "result": "confirmation_initiated" } (MUST be returned for both new and already-registered emails to satisfy FR-008 [NEEDS CLARIFICATION] (Assumed: identical))
    - 400 Bad Request: { "error": "email_required" | "email_invalid" } for EC-001/EC-002
    - 429 Too Many Requests MAY be returned by gateway/rate limiter (out of scope but compatible)
  - Validation: email MUST be non-empty and match RFC 5322-lite regex (documented in code); invalid MUST yield 400 (FR-002).

Data model (relational DB):
- Table user_accounts
  - user_id VARCHAR(36) PK
  - email CITEXT NOT NULL UNIQUE
  - status VARCHAR(24) NOT NULL (values: UNCONFIRMED, CONFIRMED, DISABLED)
  - created_at TIMESTAMPTZ NOT NULL
  - updated_at TIMESTAMPTZ NOT NULL
  - Index: ux_user_accounts_email UNIQUE(email)
- Table email_confirmations
  - confirmation_id VARCHAR(36) PK
  - user_id VARCHAR(36) NOT NULL FK -> user_accounts(user_id)
  - email CITEXT NOT NULL
  - token_hash BYTEA NOT NULL
  - expires_at TIMESTAMPTZ NOT NULL
  - sent_at TIMESTAMPTZ NULL
  - confirmed_at TIMESTAMPTZ NULL
  - status VARCHAR(24) NOT NULL (PENDING, SENT, DELIVERY_FAILED, CONFIRMED, EXPIRED)
  - delivery_request_id VARCHAR(64) NULL
  - delivery_error TEXT NULL
  - Indexes: ix_email_conf_user(user_id), ix_email_conf_tokenhash(token_hash), ix_email_conf_expires(expires_at), ix_email_conf_status(status)

Email Delivery Service contract:
- Either HTTP: POST /v1/emails/confirmation { to, templateKey, parameters{token, expiresAt}, idempotencyKey } -> 202 with {requestId}
- Or event bus topic email.send.confirmation with same payload. [NEEDS CLARIFICATION: is event bus required or optional?] (Assumed: optional; implement HTTP first with adapter interface)

Test Strategy

Contract tests:
- T1 Valid new email returns 202 and body.result=confirmation_initiated; validates POST /v1/registrations/email response contract (FR-001, FR-007).
- T2 Empty email -> 400 error=email_required; validates request validation (EC-001, FR-002).
- T3 Invalid format -> 400 error=email_invalid; validates format rule (EC-002, FR-002).
- T4 Existing email -> 202 with same body as T1; validates non-disclosure behavior (EC-003, FR-008).
- T5 Double-submit same email concurrently -> only one user_accounts row; validates uniqueness/idempotency behavior (EC-004, FR-004) and that multiple email_confirmations MAY exist but only latest PENDING is used (implementation-defined).
Integration tests:
- T6 Email delivery success: email_confirmations.status transitions PENDING->SENT, sent_at set, delivery_request_id recorded (FR-006, FR-010).
- T7 Email delivery failure: still 202 to client; email_confirmations.status=DELIVERY_FAILED, delivery_error recorded (EC-005, FR-010).
DB tests:
- T8 Token expiry: expires_at in past causes status EXPIRED on validation routine; token unusable (FR-009, EC-006). [Note: confirmation endpoint not in scope; expiry enforcement via scheduled job + on-read helper.]

Implementation Approach

Core flow (synchronous request, async-ish delivery):
- Controller: RegistrationController#createEmailRegistration(Request req)
- Service: RegistrationService#registerByEmail(String email, String requestId)
  - MUST normalize email (trim, lower) and validate.
  - MUST attempt insert into user_accounts with status=UNCONFIRMED; handle unique violation by treating as “already registered” and returning success response (FR-003, FR-008).
  - MUST create email_confirmations row with token_hash = SHA-256(token + per-record salt) and expires_at = now + 24h [NEEDS CLARIFICATION: TTL?] (Assumed: 24h), status=PENDING (FR-005, FR-009).
  - MUST call EmailDeliveryClient#sendConfirmation(to, token, expiresAt, idempotencyKey=confirmation_id) (FR-006).
  - MUST persist outcome: on 202 set status=SENT, sent_at, delivery_request_id; on exception set status=DELIVERY_FAILED, delivery_error (FR-010).
- Interfaces/classes:
  - EmailDeliveryClient (port) with implementations HttpEmailDeliveryClient and (optional) EventBusEmailDeliveryPublisher.
  - UserAccountRepository, EmailConfirmationRepository with methods insertUserAccount(...), insertEmailConfirmation(...), markSent(...), markDeliveryFailed(...).
Async patterns:
- Default: synchronous HTTP call to Email Delivery; failures MUST NOT fail registration response (EC-005).
- Optional enhancement: outbox table + background dispatcher to guarantee delivery initiation; not required by FRs.

Operational jobs:
- ExpirySweeperJob runs every hour: EmailConfirmationRepository#markExpired(now) where status in (PENDING,SENT,DELIVERY_FAILED) and expires_at < now (FR-009).

ADRs
- ADR-001: Use POST /v