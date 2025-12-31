# Feature Specification: .NET MAUI Frontend

**Feature Branch**: `2-dotnet-maui`
**Created**: 2025-12-31
**Status**: Draft
**Input**: Design and implement a cross-platform .NET MAUI frontend that follows the Plumb Dotnet Constitution (Privacy-First, Secret Hygiene & Key Vault First, Minimize Data Lifetime, Safe Logging & Telemetry, Least Privilege & Auditable Access, and CI security gates).

## Summary
A secure, privacy-first mobile/desktop client built on .NET MAUI that interacts with backend services without exposing secrets or PII, minimizes local data lifetime, and includes CI checks for safe logging, telemetry, and PII handling.

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 – Minimal Data Collection (Priority: P1)
A user authenticates and uses the app; the app must only collect and persist the minimal fields required for the feature and must never persist raw PII unnecessarily.

**Independent Test**: Automated UI and unit tests assert fields persisted locally or sent to telemetry are only the declared, minimized set; CI runs log/telemetry scans to assert no PII strings are present.

**Acceptance Scenarios**:
1. **Given** an app workflow that needs user profile data, **When** profile information is stored or transmitted, **Then** only approved fields (declared in the feature PII table) are used and any sensitive fields are redacted or tokenized.

---

### User Story 2 – Secure Authentication & Token Storage (Priority: P1)
The app uses secure OAuth/OpenID (MSAL or equivalent) flows; refresh/access tokens and credentials must be stored only in platform secure storage (Keychain/Keystore/Windows Credential Manager) and never in plaintext files or logs.

**Independent Test**: Integration test flows authenticate against an identity test tenant; verify tokens are stored in secure storage and are not present in any build artifacts or logs.

**Acceptance Scenarios**:
1. **Given** a successful sign-in, **When** a token is issued, **Then** the token is persisted only in platform secure storage and never written to logs or files accessible by other apps.

---

### User Story 3 – Safe Telemetry and Crash Reporting (Priority: P1)
Telemetry must never contain PII or raw payloads; crash reports must be scrubbed before transmission and must not contain sensitive fields.

**Independent Test**: End-to-end test injects a crash and inspects the crash report payload for PII patterns; CI asserts telemetry export does not contain disallowed fields.

**Acceptance Scenarios**:
1. **Given** an error that triggers a crash report, **When** the report is uploaded, **Then** it contains only non-sensitive metadata (error codes, stack traces without request/PII content, app version) and passes automated sensitive-string checks.

---

### Edge Cases
- Device offline sync: ensure queued items are encrypted, have retention limits, and are pruned after successful sync.
- Lost device: ensure local caches and tokens can be revoked remotely and that local caches are auto-evicted after configurable inactivity or policy expiration.
- Large uploads: enforce memory caps for in-memory processing and fail-fast with non-sensitive diagnostics.
- Third-party SDKs: verify SDKs do not leak PII in logs or telemetry and are pinned to approved versions with license checks.

---

## Requirements *(mandatory)*

### Functional Requirements
- **FR-001**: App MUST collect and store only the minimal data required for the user task; any additional data collection requires explicit justification and a change to the spec.
- **FR-002**: Authentication MUST use OAuth/OpenID flows (e.g., MSAL) or delegated tokens; client secrets MUST NOT be embedded in the app.
- **FR-003**: All sensitive tokens and secrets MUST be stored using platform secure storage APIs (`SecureStorage`, Keychain/Keystore/Windows Credential Manager); where backend token exchange can avoid refresh token storage on client, prefer token exchange.
- **FR-004**: Network communication MUST use TLS 1.2+ with certificate validation; mission-critical flows SHOULD consider certificate pinning or mTLS where feasible.
- **FR-005**: The app MUST avoid persisting raw PII to disk; transient in-memory processing is required for uploads and raw payloads must be zeroed/cleared as soon as feasible.
- **FR-006**: Logging and telemetry MUST adhere to safe-logging rules (no PII, no raw request/response bodies); logging libraries must be configured to strip or redact sensitive fields.
- **FR-007**: The build and CI pipelines MUST include static analyzers and Roslyn analyzers that detect unsafe logging patterns and flag them as failures.
- **FR-008**: The app MUST implement explicit user consent surfaces for any telemetry or feature that collects data beyond basic functionality.
- **FR-009**: Offline storage (cache, queue) MUST be encrypted at rest using platform mechanisms and must enforce retention/automatic shredding policies.
- **FR-010**: Permissions (camera, photos, etc.) MUST be requested only when needed and explanations for usage MUST be present in UX text.

### PII & Security Requirements (MANDATORY)
- **PII-001**: Any feature that processes PII MUST declare the exact fields that are PII and provide handling instructions (redacted, tokenized, encrypted, or discarded).
- **PII-002**: The app MUST NOT store sensitive payment data (PAN, CVV) unless a certified payment SDK is used and PCI constraints are followed; where feasible, use tokenized payment flows via backend.
- **PII-003**: Crash reports and telemetry MUST be scanned in CI for sensitive strings before allowing release builds to be published.
- **PII-004**: Local logs and debugging artifacts included in builds MUST NOT contain PII; developer builds should have stricter safeguards and automatic redaction in place.
- **PII-005**: Secrets (API keys, client secrets, certificates) MUST NOT be embedded in source or build artifacts; secrets for integrations MUST be acquired via secure backend exchange or injected through protected CI variables and server-side provisioning.
- **SEC-001**: The app MUST use least privilege for requested permissions and document why each permission is needed.
- **SEC-002**: A security sign-off is required for changes that alter data flows, increase data collection, or add third-party SDKs that access user data.
- **SEC-003**: All third-party libraries must be reviewed for license compatibility and security posture and be included in SBOM for app releases.

---

## PII Handling Table *(example — required when processing PII)*
- **Email**: considered PII — Handling: tokenized when sent to backend; only hashed email may be stored locally for display if necessary.
- **Full name**: considered PII — Handling: stored ephemeral for session; persisted only if user profile requires it and stored encrypted.
- **Phone number**: considered PII — Handling: encrypted at rest; minimized use in telemetry.
- **Payment data**: **not** stored on device; use tokenization via backend.

> Note: Every feature that will store or transmit any of the above MUST include a specific PII field declaration in its feature spec and a short risk/mitigation note.

---

## Key Entities
- **User Credentials/Token**: OAuth access/refresh tokens; stored only in platform secure storage.
- **Local Cache/Queue**: Ephemeral offline items stored encrypted with strict retention (default short retention policy documented in implementation plan).
- **Telemetry/Crash Event**: Structured events without payloads or PII, annotated with non-sensitive metadata only.
- **Third-party SDKs**: Analytics, crash reporting, or payment SDKs; must be audited and version-pinned.

---

## Assumptions
- Backend APIs perform server-side validation, token exchange, and PII-minimization logic; client is not the source of truth for policy enforcement.
- CI is capable of running static analyzers, Roslyn analyzers, secrets scanning, and telemetry payload scanning during release pipelines.
- Platform secure storage APIs are available on target platforms (.NET MAUI-supported OSes) and behave as documented.
- User consent and privacy text will be part of the UX (localization to be handled by product/UX teams).

---

## Success Criteria *(mandatory / measurable outcomes)*
- **SC-001**: 0 occurrences of PII strings in release artifacts and telemetry across 100 automated test runs (validated by CI-sensitive-string checks).
- **SC-002**: Authentication flows succeed and tokens are only present in secure storage for >99% of integration test runs.
- **SC-003**: 100% of third-party SDKs included in releases are listed in SBOM and pass license/security checks in CI.
- **SC-004**: Telemetry and crash payloads pass automated redaction validator in CI for all test crashes and events.
- **SC-005**: Offline queue retention policy enforced in >99% of runs during integration tests (items pruned according to policy).

---

## Acceptance Tests (examples)
- **Telemetry Scan Test**: Simulate app flows that generate telemetry; run CI scanning tool against telemetry artifacts; assert no PII patterns.
- **Secure Storage Test**: End-to-end test to authenticate, store tokens, and assert tokens exist only in platform secure storage and are inaccessible via regular FS reads.
- **Permission Flow Test**: UI tests to validate permission prompts only appear when feature invoked and that denial flows are handled gracefully without leaking data.
- **Offline Sync Test**: Simulate network loss and re-sync; assert queued items are encrypted, pruned per retention policy, and do not contain PII in logs.
- **Third-party SDK Audit Test**: CI checks for allowed SDK versions/licenses and runs static checks to ensure SDKs are not configured to send PII.

---

## Dependencies
- Backend token exchange endpoints and server-side PII minimization
- CI tools for static analysis, secrets scanning, telemetry scanning, and SBOM generation
- Approved crash/telemetry providers that support payload scrubbing and privacy controls
- Platform secure storage support for target OSes

---

## Notes & Non-goals
- Non-goal: Attempting to implement server-side compliance solely in the client; server controls are required for full compliance.
- Non-goal: Embedding long-term secrets or client secrets in the application.
- Note: Where possible prefer server-side token exchange patterns to keep client surface minimal.

---

## Assumptions & Decisions to Document
- Choose `MSAL` for authentication unless a compelling reason to use another provider is documented.
- Use `SecureStorage` abstraction with platform-specific providers and include tests that verify plat-specific behavior.
- Decide on runtime telemetry provider during planning and ensure it supports payload scrubbing and offline queue encryption.

---

**Spec status**: Draft — ready for review and quality checklist validation (`/speckit.plan`) once validated by security and product stakeholders.
