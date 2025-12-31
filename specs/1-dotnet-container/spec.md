# Feature Specification: .NET Container

**Feature Branch**: `1-dotnet-container`
**Created**: 2025-12-31
**Status**: Draft
**Input**: Create a secure, privacy-first .NET container image and runtime pattern for services in this repository that fully complies with the Plumb Dotnet Constitution (Privacy-First, Secret Hygiene, Minimize Data Lifetime, Safe Logging, Least Privilege, and CI security gates).

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Containerized Service Runtime (Priority: P1)
A platform operator wants to run .NET-based services in a containerized environment that enforces privacy, secret hygiene, and safe logging.

**Why this priority**: Deployable .NET workloads are required to run production jobs and perform ingestion tasks. Without a secure container pattern, services risk leaking PII or secrets.

**Independent Test**: Build the container image using CI, run it in an ephemeral environment, and verify the image passes the security and privacy checks (no secrets, no raw PII on disk, safe logging, image scan results within policy).

**Acceptance Scenarios**:
1. **Given** a service project and CI pipeline, **When** the container image is built and scanned, **Then** it passes the configured vulnerability and policy checks and is published to the registry.
2. **Given** the container is started in an ephemeral environment, **When** it processes sample input, **Then** there is no raw PII saved to disk and logs contain only non-sensitive metadata.

---

### User Story 2 - Developer Feedback & Reproducible Builds (Priority: P2)
A developer wants to build and test the container locally with reproducible results that match CI builds.

**Why this priority**: Ensures developer productivity and consistency between local and CI environments.

**Independent Test**: Developer can run `make build-image` (or `docker build`) and the produced image matches CI-produced image digest for the same commit.

**Acceptance Scenarios**:
1. **Given** a local developer build and the CI image built from the same commit, **When** both digests are compared, **Then** digests match.

---

### User Story 3 - Secure Configuration & Secret Access (Priority: P2)
An operator wants containers to access secrets securely without embedding credentials in images or environment variables that persist in logs or artifacts.

**Why this priority**: Prevents secret leakage and supports rotation and auditing.

**Independent Test**: Provision a Key Vault secret and verify the running container obtains it via an injected runtime-managed identity or sidecar mechanism; secret material is never committed to disk or recorded in application logs.

**Acceptance Scenarios**:
1. **Given** a managed identity/secret provider configured, **When** the container requests a secret at runtime, **Then** the response succeeds and secret material is never written to disk or logs.

---

## Constitution Alignment 🏛️
This feature MUST adhere to the Plumb Dotnet Constitution (`constitution.md`). The following verifications are mandatory and must be codified into CI and acceptance tests where applicable:
- **No secrets in environment variables** — enforce Key Vault + Managed Identity for all runtime secrets; CI must block changes that introduce long-lived env vars or embedded secrets.
- **No sensitive logs or telemetry** — add static analysis, log-scanning, and CI gates that fail on sensitive-string matches (PII patterns, raw payloads).
- **No raw payload persistence to disk** — ingestion must process uploads in-memory with enforceable caps and automatic shredding; include integration tests that assert no raw payload writes to filesystem.
- **OpenAPI/schema validation & auth** — require schema validation for uploads and additional authorization checks/audit trails for any row-level data or decryption endpoints.
- **Retention & shredding policies** — transient artifacts must have short TTLs and automatic cleanup enforced by platform or CI checks.

---

### Edge Cases

- How does the container behave if the secret provider is unavailable? (Retry/backoff and fail-fast with non-sensitive error codes.)
- What happens when large inputs exceed memory caps? (Enforce caps and fail early with non-sensitive error messages.)
- How to handle malformed inputs that might contain PII? (Input validation and immediate redaction of PII before any persistence.)

## Requirements *(mandatory)*

### Functional Requirements
- **FR-001**: Provide a reproducible Docker build that produces a content-addressable image digest for a given commit.
- **FR-002**: Images MUST run as a non-root user by default.
- **FR-003**: Runtime MUST support retrieving secrets via managed identity/Key Vault and MUST NOT require embedded plaintext secrets in the image.
- **FR-004**: The container MUST not persist raw PII to disk; any persisted artifacts must be redacted, encrypted, or otherwise rendered non-identifying.
- **FR-005**: Application logging MUST follow safe-logging rules (no PII/raw payloads); logging traces MUST be limited to non-sensitive metadata (job IDs, statuses, sizes, latency, error codes).
- **FR-006**: The build pipeline MUST include static analysis and image scanning (vulnerability scan, license checks) and fail on policy violations.
- **FR-007**: The deployed container MUST enforce memory/disk caps configurable by environment and/or orchestration platform.
- **FR-008**: The container MUST include health and readiness probes and clear failure modes with non-sensitive diagnostics.

### PII & Security Requirements (MANDATORY)
- **PII-001**: Any feature that processes PII MUST declare exact fields considered PII and how they are handled (redacted, encrypted, discarded). For this container pattern: raw uploads MUST be handled in-memory and explicitly discarded after processing; persistent artifacts MUST never contain raw PII.
- **PII-002**: The container image MUST NOT contain secret material (keys, tokens, certificates). Secrets must be accessed at runtime via Key Vault and retrieved using managed identity with least privilege.
- **PII-003**: All logs and telemetry MUST be scanned in CI for accidental sensitive strings before publishing (automated checks in CI pipeline).
- **SEC-001**: The container build and runtime MUST be included in CI security gates: vulnerability scanning, SBOM generation, license checks, and a security sign-off for changes that affect data flows.

## Key Entities *(include if feature involves data)*
- **Container Image**: Built artifact (digest + tags) containing the application and only non-secret configuration.
- **Runtime Configuration**: Non-sensitive configuration (timeouts, caps). Secrets are referenced by identity and not embedded.
- **Secret**: Any key or credential stored in Key Vault and retrieved at runtime via managed identity; never present in image or persisted logs.
- **Log/Telemetry**: Structured events that include metadata only (job id, status, duration); no raw payloads.

## Assumptions
- The orchestration environment (Kubernetes/AKS or similar) will supply managed identity support and runtime secret injection patterns (e.g., CSI drivers, token exchange).
- The codebase is using a supported LTS .NET runtime; images will adopt the latest LTS base and apply security patches regularly.
- CI has image-scan and SBOM generation capability (or will be configured as part of the implementation).

## Success Criteria *(mandatory)*

### Measurable Outcomes
- **SC-001**: 100% of container images built from mainline commits generate SBOM and pass configured vulnerability policy in CI (no critical/high vulnerabilities unmitigated).
- **SC-002**: No raw PII is persisted to disk in more than 100 integration runs (validated by tests that assert file system writes contain no PII patterns).
- **SC-003**: 0 secrets committed in images or repository (validated by automated secrets scanning in CI and pre-commit hooks).
- **SC-004**: Container startup time under 15s in typical environment (measured in CI integration test).
- **SC-005**: Logs contain no PII in 100% of automated test runs (validated by automated log scanning tests).

## Acceptance Tests (examples)
- Build and scan pipeline test: Trigger CI build, assert SBOM generated, image scan passes policy, and image published to staging registry.
- Runtime secret access test: Deploy to ephemeral environment with Key Vault/restricted secrets; assert that container can retrieve secret and that secret never appears in logs or persisted files.
- Redaction test: Feed input with PII; assert no PII found on disk or in logs after processing.
- Ingestion persistence test: Upload representative payloads and assert the service performs in-memory-only processing and never writes raw payloads to disk (add filesystem assertions to integration tests).
- CI log-scan test: Add a CI step that scans build and runtime logs for sensitive patterns/PII and fails the pipeline when matches are detected.
- Failure mode tests: Simulate secret provider failure and large input memory spike; assert predictable fail-fast behavior and non-sensitive diagnostics.

## Dependencies
- CI image scanning and SBOM tooling
- Key Vault or equivalent secret management
- Orchestration platform support for managed identities and resource limits

## Notes & Non-goals
- This specification does not prescribe a specific orchestration platform or image registry; those are environment-specific and belong in the implementation plan.
- Implementation details (which base image tag or specific tools for scanning) belong in `/speckit.plan` and tasks.

## Assumptions & Decisions to Document
- Default to in-memory processing of sensitive inputs and avoid writing raw payloads to disk whenever possible.
- Default to non-root execution user in containers and smallest necessary set of packages in images.


---

**Spec status**: Draft — ready for quality checklist validation and planning (`/speckit.plan`) once validated.
