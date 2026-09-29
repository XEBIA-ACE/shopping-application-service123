## svc-user-account

### Purpose
Define the functional behavior of User Account Service to register a new user via email and trigger email confirmation.

### Scope
Covers email-based user registration within User Account Service, including input validation, account creation, and requesting a confirmation email via an email delivery capability.

### Non-Goals
- Social login/registration (Google, Apple, etc.)
- Phone/SMS-based registration
- Password reset and recovery flows
- Profile editing after registration
- JWT issuance and login behavior
- Rate-limiting/throttling policy definition
- Email template authoring and branding
- Admin user provisioning tools

### Key Entities
UserAccount: user_id (string), email (string), status (string), created_at (datetime), updated_at (datetime)  
EmailConfirmation: confirmation_id (string), user_id (string), email (string), token (string), expires_at (datetime), sent_at (datetime), confirmed_at (datetime, nullable), status (string)  
RegistrationRequest: email (string), password (string) [NEEDS CLARIFICATION: Is password required at registration?] (Assumed: password is required)  
Relationships: UserAccount related EmailConfirmation (0..many); EmailConfirmation related UserAccount (1)

### Assumptions Propagation
A-001: Registration is email + password (no social), and password is required at registration. (Affects FR-002, FR-004, FR-006)  
A-002: Newly created accounts require email confirmation before being considered “active.” (Affects FR-005, FR-009, FR-010)  
A-003: An external Email Delivery Service exists and can be requested to send confirmation messages. (Affects FR-011, FR-012)

### Acceptance Criteria Priorities
AC-001 (P1): Given a new user, when they provide a valid email, then their account is created and a confirmation email is sent.

### Functional Requirements
FR-001: User Account Service SHALL accept an email registration request from an API client.  
FR-002: User Account Service MUST require an email value in the registration request.  
FR-003: User Account Service MUST validate the email format for syntactic correctness.  
FR-004: User Account Service MUST require a password in the registration request.  
FR-005: User Account Service SHALL create a new UserAccount for a previously unregistered email.  
FR-006: User Account Service MUST enforce password policy rules.  
FR-007: User Account Service MUST reject registration when the email is already registered.  
FR-008: User Account Service SHOULD return a user-facing outcome that does not disclose whether an email exists. [NEEDS CLARIFICATION: Is anti-enumeration required?] (Assumed: yes)  
FR-009: User Account Service SHALL set newly registered accounts to a non-confirmed status.  
FR-010: User Account Service MUST generate an EmailConfirmation token with an expiry.  
FR-011: User Account Service SHALL request Email Delivery Service to send a confirmation email to the registered address.  
FR-012: User Account Service MUST record the confirmation send attempt outcome for audit/troubleshooting.  
FR-013: User Account Service MUST respond with a registration result that allows the client to proceed (e.g., “check your email”).  
FR-014: User Account Service MUST reject requests missing required fields with a clear validation reason.  
FR-015: User Account Service MUST handle transient Email Delivery Service failures without creating duplicate user accounts.

### Business Rules and Validations
- Email MUST be normalized consistently for uniqueness comparisons [NEEDS CLARIFICATION: case-folding and dot/plus rules?]. (Assumed: case-insensitive comparison)  
- Password MUST meet minimum length and complexity requirements [NEEDS CLARIFICATION: exact policy]. (Assumed: min 8 chars)  
- Duplicate registration attempts for the same email MUST NOT create multiple UserAccounts.  
- Confirmation token MUST be single-use and expire after a defined period [NEEDS CLARIFICATION: expiry duration]. (Assumed: 24 hours)

### Edge Cases and Error Handling
EC-001: Given missing email, When registration is submitted, Then the service rejects it citing email required. (FR-002, FR-014)  
EC-002: Given invalid email syntax, When registration is submitted, Then the service rejects it citing invalid email format. (FR-003, FR-014)  
EC-003: Given weak password, When registration is submitted, Then the service rejects it citing password policy failure. (FR-006, FR-014)  
EC-004: Given email already registered, When registration is submitted, Then the service rejects or responds generically per anti-enumeration rule. (FR-007, FR-008)  
EC-005: Given Email Delivery Service is unavailable, When registration succeeds, Then the account is created once and the send attempt is recorded as failed. (FR-011, FR-012, FR-015)  
EC-006: Given a retry of the same request, When processed, Then no duplicate account is created and confirmation may be re-sent per policy. [NEEDS CLARIFICATION: allow resend on retry?] (FR-005, FR-015)

### Integration Points
- Email Delivery Service: receives a request to send a confirmation email containing a confirmation link/token; returns delivery acceptance/failure for recording. (FR-011, FR-012)

### Independent Testability (Minimum Viable Test Scenario)
Preconditions: (1) Email is not registered, (2) Email Delivery Service is reachable, (3) Password meets policy.  
User action: Submit an email registration request.  
Observable outcome: A new UserAccount exists in non-confirmed status and a confirmation email send request is recorded as successful.

### Success Criteria
SC

---

## svc-bff

### Purpose
Define the Backend-for-Frontend Layer (svc-bff) behavior to register a new user with an email address and trigger a confirmation email via downstream services.

### Scope
Covers email-based registration orchestration in svc-bff, including validation, user creation delegation, confirmation email triggering, and user-facing responses; excludes social registration and user profile completion flows.

### Non-Goals
- Social login/registration (Google/Apple/Facebook)
- Passwordless magic-link sign-in
- Password policy definition beyond basic validation
- Email template design and branding
- User profile fields beyond email
- CAPTCHA/bot mitigation flows
- Rate-limit configuration and tuning
- Admin user provisioning tools

### Key Entities
RegistrationRequest: email (string), locale (string), clientContext (string)  
RegistrationResult: userId (string), email (string), status (string), confirmationRequired (boolean)  
UserAccount: userId (string), email (string), state (string), createdAt (datetime)  
ConfirmationDispatch: email (string), dispatchStatus (string), correlationId (string)  
Relationships: RegistrationResult -> UserAccount (1:1); UserAccount -> ConfirmationDispatch (1:0..1)

### Assumptions Propagation
A-001: User creation and confirmation email dispatch are performed by downstream services, with svc-bff orchestrating. (Applies to FR-003, FR-006, FR-011)  
A-002: Registration only requires email; password is not part of this story. (Applies to FR-002, FR-008)  
A-003: Email confirmation is required to activate the account. (Applies to FR-010, FR-012)  
A-004: svc-bff can provide localized messaging if locale is supplied; otherwise default locale is used. (Applies to FR-009, FR-013)  
A-005: “Account created and confirmation email sent” may be acknowledged even if email delivery is asynchronous. (Applies to FR-011, FR-012)  
[NEEDS CLARIFICATION: Should repeated registration attempts with an existing email return a generic success to prevent enumeration, or a specific error?] (Assumed: generic success) (Applies to FR-007, FR-014)

### Acceptance Criteria Priorities
AC-001 (P1): Given a new user, when they provide a valid email, then their account is created and a confirmation email is sent.

### Functional Requirements
FR-001: svc-bff SHALL expose an email registration capability to frontend clients.  
FR-002: svc-bff MUST accept a registration request containing an email value.  
FR-003: svc-bff SHALL delegate account creation to the User Account Service.  
FR-004: svc-bff MUST validate that the email is syntactically valid before delegation.  
FR-005: svc-bff MUST reject requests with missing or blank email input.  
FR-006: svc-bff SHALL trigger confirmation email dispatch via downstream capability after account creation.  
FR-007: svc-bff SHOULD return a generic outcome for already-registered emails to reduce account enumeration risk.  
FR-008: svc-bff MUST NOT require social account identifiers for registration.  
FR-009: svc-bff MAY accept an optional locale to influence user-facing messaging.  
FR-010: svc-bff SHALL indicate that confirmation is required for newly created accounts.  
FR-011: svc-bff MUST provide an observable result indicating whether registration processing was accepted or completed.  
FR-012: svc-bff SHALL correlate account creation and confirmation dispatch outcomes in its response.  
FR-013: svc-bff SHOULD return user-consumable error messages for validation failures.  
FR-014: svc-bff MUST handle downstream service failures without exposing internal error details to clients.

### Edge Cases
EC-001: Given an empty email, When registration is submitted, Then svc-bff rejects it with a validation message. (FR-005)  
EC-002: Given an invalid email format, When registration is submitted, Then svc-bff rejects it before calling downstream services. (FR-004)  
EC-003: Given an email with leading/trailing spaces, When submitted, Then svc-bff normalizes or rejects per validation rules. [NEEDS CLARIFICATION: normalize by trimming?] (Assumed: trim) (FR-004)  
EC-004: Given an email already registered, When registration is submitted, Then svc-bff returns a generic success outcome. (FR-007)  
EC-005: Given downstream account creation is unavailable, When registration is submitted, Then svc-bff returns a non-specific failure outcome and suggests retry. (FR-014)  
EC-006: Given account creation succeeds but confirmation dispatch is delayed, When registration is submitted, Then svc-bff indicates account created and confirmation pending/required. (FR-011)  
EC-007: Given confirmation dispatch fails after account creation, When registration completes, Then svc-bff indicates confirmation required and provides retry guidance. (FR-012)  
EC-008: Given locale is unsupported, When provided, Then svc-bff falls back to default locale behavior. (FR-009)

### Integration Points
svc-bff interacts with User Account Service to create accounts and retrieve resulting user identifiers/state. svc-bff interacts with a Confirmation Email Dispatch capability [NEEDS CLARIFICATION: is this part of User Account Service or a separate Notification Service?] to request sending a confirmation email and obtain dispatch status/correlation data.

### Independent Testability
Preconditions: (1) A previously unseen email is chosen; (2) downstream account creation capability is available; (3) downstream confirmation dispatch capability is available.  
User action: Submit an email registration request with a valid email.  
Observable outcome: svc-bff returns

---

## svc-background-worker

### Purpose
This specification defines how the Background Task Worker supports email-based user registration by sending confirmation emails after an account is created.

### Scope
Covers Background Task Worker responsibilities to accept, queue, execute, track, and report “send registration confirmation email” tasks triggered by user account creation.

### Non-Goals
- Creating user accounts or storing user profiles
- Validating email ownership beyond sending confirmation email
- Supporting social login registrations
- Rendering UI for registration or confirmation flows
- Managing email templates’ content lifecycle beyond selecting a version
- Handling inbound confirmation link clicks
- Guaranteeing immediate delivery by external email providers
- Performing synchronous email sending during user sign-up
- Sending SMS or push notifications for registration

### Key Entities
TaskRequest: task_type (string), idempotency_key (string), payload (object), requested_by (string), requested_at (datetime)
EmailConfirmationTask: task_id (string), user_id (string), email (string), locale (string), template_id (string), correlation_id (string), created_at (datetime)
TaskStatus: task_id (string), state (string), attempts (integer), last_error (string), next_retry_at (datetime), updated_at (datetime)
EmailSendResult: task_id (string), provider_message_id (string), accepted_at (datetime), outcome (string)
Relationships: TaskRequest 1..1 -> EmailConfirmationTask; EmailConfirmationTask 1..1 -> TaskStatus; EmailConfirmationTask 0..1 -> EmailSendResult

### Functional Requirements
FR-001: Background Task Worker SHALL accept enqueue requests for a registration confirmation email task. (P1)
FR-002: Background Task Worker MUST validate that the task payload includes user_id and email. (P2)
FR-003: Background Task Worker MUST reject tasks with syntactically invalid email addresses. (P2)
FR-004: Background Task Worker SHALL persist task status transitions for each confirmation email task. (P1)
FR-005: Background Task Worker MUST execute the task by requesting email delivery from the Email Delivery Provider. (P1)
FR-006: Background Task Worker SHALL include a confirmation token or confirmation link data provided in the task payload when sending. (P1) [NEEDS CLARIFICATION: Who generates the confirmation token/link—upstream service or the worker?] (Assumed: upstream provides token/link data)
FR-007: Background Task Worker SHOULD apply idempotency using idempotency_key to avoid duplicate emails for the same registration event. (P2)
FR-008: Background Task Worker MUST retry failed sends according to configured retry policy and stop after max attempts. (P2) [NEEDS CLARIFICATION: What are retry intervals and max attempts?] (Assumed: bounded retries with backoff)
FR-009: Background Task Worker MUST route permanently failing tasks to a dead-letter handling capability and expose failure reason to operators. (P2)
FR-010: Background Task Worker SHALL expose task state and outcome for operational visibility and user-support diagnostics. (P2)

Acceptance Criteria Priority Mapping:
- AC-001 (Account created triggers confirmation email send via worker): P1

Assumptions Propagation
A-001: Upstream account service creates the user account before enqueuing email task. (Affected FR-001, FR-005)
A-002: Upstream provides confirmation token/link data in the task payload. (Affected FR-006)
A-003: Email Delivery Provider is available via an integration the worker can call. (Affected FR-005, FR-008)
A-004: A unique idempotency_key can be provided per registration event. (Affected FR-007)

### Edge Cases
EC-001: Given a task missing user_id, When enqueued, Then the worker rejects it with a validation error and does not attempt sending. (FR-002)
EC-002: Given a task with email “invalid@”, When enqueued, Then the worker rejects it as invalid email format. (FR-003)
EC-003: Given two enqueue requests with same idempotency_key, When processed, Then only one email send is performed and the second is marked duplicate. (FR-007)
EC-004: Given the Email Delivery Provider times out, When executing, Then the worker marks attempt failed and schedules a retry until max attempts. (FR-008)
EC-005: Given the provider rejects the email as non-existent domain, When executing, Then the worker marks as permanent failure and routes to dead-letter handling. (FR-009)
EC-006: Given a task is canceled before execution, When the worker sees it, Then it does not send and records state as canceled. (FR-004)
EC-007: Given a task completed successfully, When queried by operators, Then status shows completed with provider_message_id (if available). (FR-010)

### Integration Points with Other Services
- Account/Registration Service (upstream event source): Provides user_id, email, locale, and confirmation token/link data; requests enqueue.
- Event Bus (optional trigger path): Publishes/consumes “UserRegistered” domain events that result in enqueueing the task. [NEEDS CLARIFICATION: Is event-driven trigger required or admin enqueue only?]
- Email Delivery Provider: Receives send request and returns acceptance/rejection and optional provider message identifier.
- Operations/Monitoring Systems: Consume worker-exposed status/metrics to monitor failures, retries, and dead-letter volume.

### Independent Testability
Preconditions: (1) Worker is running, (2) Email Delivery Provider test double is available, (3) Valid payload contains user_id, email, and confirmation link data.  
User action: Enqueue one registration confirmation email task.  
Observable outcome: Task status transitions to completed and exactly one send request is recorded by the provider test double.

---

## SVC-01

### Purpose
This specification defines the user-visible behavior for email-based user registration in the Shopping Application Service, including account creation and sending a confirmation email.

### Scope
Covers creation of a new user account using an email address and initiating delivery of an email confirmation message. Applies only to the Shopping Application Service user-registration capability.

### Non-Goals
- Social login or federated identity registration
- Password reset and account recovery flows
- User profile completion beyond email identity
- Login/authentication token issuance
- Email template design and branding details
- Admin user creation and backoffice tooling
- Marketing opt-in management [NEEDS CLARIFICATION]
- Multi-factor authentication enrollment
- Mobile number registration

### Key Entities
UserAccount: user_id (string), email (string), status (string), created_at (datetime), updated_at (datetime)  
EmailConfirmation: confirmation_id (string), user_id (string), email (string), token (string), expires_at (datetime), sent_at (datetime), confirmed_at (datetime, nullable), status (string)  
RegistrationRequest: email (string)  
Relationships: UserAccount to EmailConfirmation (1 to 0..many), EmailConfirmation to UserAccount (many to 1)

### Assumptions Propagation
A-001: Registration is initiated by a shopper providing only an email (no password collected in this story). (Affects: FR-001, FR-002, FR-003)  
A-002: Account starts in an unconfirmed state until the shopper confirms via email link/code. (Affects: FR-004, FR-006, FR-009)  
A-003: A separate Email Delivery service exists and is reachable for sending confirmation emails. (Affects: FR-006, FR-010)  
A-004: Email uniqueness is enforced for user accounts. (Affects: FR-003, FR-008)

### Functional Requirements
FR-001: Shopping Application Service SHALL accept a registration request containing an email address. (P1)  
FR-002: Shopping Application Service SHALL validate the email address format and reject invalid formats. (P2)  
FR-003: Shopping Application Service SHALL reject registration when the email is already associated with an existing account. (P2)  
FR-004: Shopping Application Service SHALL create a new UserAccount in an unconfirmed status for a valid, new email. (P1)  
FR-005: Shopping Application Service SHALL create an EmailConfirmation record bound to the new UserAccount. (P1)  
FR-006: Shopping Application Service SHALL request delivery of a confirmation email containing a confirmation token or code. (P1)  
FR-007: Shopping Application Service SHALL return a user-facing result indicating that confirmation email delivery was initiated. (P1)  
FR-008: Shopping Application Service SHOULD avoid revealing whether an email is registered in user-facing responses. [NEEDS CLARIFICATION: should responses be identical for existing vs new email?] (Assumed: yes) (P2)  
FR-009: Shopping Application Service SHALL set an expiration time for the confirmation token/code and mark it unusable after expiration. (P2)  
FR-010: Shopping Application Service MUST record the outcome of the email delivery request for audit/troubleshooting (e.g., requested, failed). (P2)

Acceptance Criteria Priorities:  
AC-001 (Given new user provides valid email, account created and confirmation email sent): P1

### Edge Cases
EC-001: Given an empty email, When registration is submitted, Then the request is rejected with a reason indicating email is required. (FR-001, FR-002)  
EC-002: Given an email with invalid format, When registration is submitted, Then the request is rejected due to invalid email format. (FR-002)  
EC-003: Given an email already registered, When registration is submitted, Then the system rejects creation and responds without confirming account existence. (FR-003, FR-008)  
EC-004: Given a valid new email, When registration is submitted twice rapidly, Then at most one active unconfirmed account is created for that email. [NEEDS CLARIFICATION: should duplicate attempts be idempotent?] (Assumed: yes) (FR-004, FR-005)  
EC-005: Given account creation succeeds but email delivery request fails, When registration completes, Then the user receives a generic “check your email” style result and the failure is recorded. (FR-006, FR-007, FR-010)  
EC-006: Given a confirmation token is expired, When it is later presented for confirmation, Then the system treats it as invalid and requires a new confirmation to be issued. (FR-009)

### Integration Points with Other Services
The service MUST integrate with an Email Delivery service to send confirmation messages (FR-006) and MUST capture delivery-request outcomes for operational visibility (FR-010). If an event/messaging capability is used for email dispatch, the service behavior MUST remain consistent: initiating delivery without requiring synchronous confirmation of inbox receipt. [NEEDS CLARIFICATION: is an event bus required or optional for dispatch?]

### Independent Testability
Minimum viable test scenario:  
Preconditions (1-5): (1) No existing account uses email test@example.com; (2) Email Delivery service is available or simulated; (3) System is able to persist accounts.  
User action (exactly one): Shopper submits registration with email test@example.com.  
Observable outcome (exactly one): System returns a result indicating a confirmation email was initiated for delivery.

### Success Criteria
SC-001: confirmation email initiation rate equals 100% for valid new-email registrations in controlled test runs