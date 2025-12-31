---
description: "Generated tasks for feature: .NET Container"
---

# Tasks: .NET Container

**Input**: `specs/1-dotnet-container/spec.md`, `specs/1-dotnet-container/speckit.plan.md`

## Phase 1: Setup (Shared Infrastructure)

- [ ] T001 Create `specs/1-dotnet-container/Dockerfile` with multi-stage build, non-root user, and Chainguard runtime final stage
- [ ] T002 Add `Makefile` target `make build-image` and `make local-build` that reproduces CI build steps (`docker build --label=git-commit` etc.)
- [ ] T003 Add GitHub Actions workflow at `.github/workflows/ci-build-and-scan.yml` to: restore, build, test, generate SBOM (Syft), run Trivy scan, and push to ACR via OIDC
- [ ] T004 Add repository-level `.github/actions/` utility scripts for SBOM and image-scan orchestration used by CI
- [ ] T005 Add `checklists/container-security.md` under `specs/1-dotnet-container/checklists/` (review checklist for reviewers and acceptance criteria)
- [ ] T006 Add a short `quickstart.md` in `specs/1-dotnet-container/` describing local build, local run, and comparison steps for reproducible builds

---

## Phase 2: Foundational (Blocking Prerequisites)

- [ ] T007 [P] Add `scripts/ci/generate-sbom.sh` and `scripts/ci/run-trivy.sh` and ensure output is machine-parsable for CI gates
- [ ] T008 [P] Create RBAC/infra docs: `specs/1-dotnet-container/infra.md` describing Managed Identity usage and Key Vault access patterns
- [ ] T009 [P] Add `src/health/check-health.sh` and minor app health endpoint example `src/health/health.cs` (or sample program) and add health/readiness probe guidance in `Dockerfile`
- [ ] T010 [P] Add a short `tests/contract/test_image_scan.yml` CI manifest (or make target) that asserts SBOM present and Trivy exit code policy checks
- [ ] T011 Implement a static log-scan CI job: `.github/workflows/ci-log-scan.yml` that runs a sensitive-string scan against build and runtime logs
- [ ] T012 [P] Add pre-commit hooks and CI secrets scanning via `.github/` config and `pre-commit` hooks to detect committed secrets

---

## Phase 3: User Story 1 - Containerized Service Runtime (Priority: P1) 🎯 MVP

**Goal**: Provide a reproducible, secure, non-root container runtime with safe logging and in-memory-only sensitive handling

**Independent Test**: Build image in CI, run ephemeral container, process sample payload, verify no raw PII on disk and logs contain only non-sensitive metadata

### Tests for User Story 1

- [ ] T013 [P] [US1] Add integration test `tests/integration/test_runtime_no_pii.cs` that runs container, sends sample PII payloads, and asserts no disk writes and logs are PII-free
- [ ] T014 [US1] Add contract test `tests/contract/test_container_health.cs` that verifies health and readiness endpoints

### Implementation for User Story 1

- [ ] T015 [US1] Implement `specs/1-dotnet-container/Dockerfile` (finalize recommended Dockerfile from plan) and add example `src/Program.cs` minimal app to run in container
- [ ] T016 [US1] Ensure app runs as non-root: add Dockerfile steps to create non-root user and set ownership of `/app`
- [ ] T017 [US1] Implement in-memory-only upload handling in `src/services/upload_service.cs` and add unit tests in `tests/unit/test_upload_in_memory.cs`
- [ ] T018 [US1] Add structured logging configuration in `src/logging/logging.cs` to redact or omit payload fields and add Roslyn analyzer rules to flag unsafe logging patterns
- [ ] T019 [US1] Add resource limit guidance and runtime config in `specs/1-dotnet-container/runtime-config.md` and CI test `tests/integration/test_memory_limit.yml` to assert memory/disk caps
- [ ] T020 [US1] Add health/readiness endpoints to `src/health/` and wire into `Dockerfile` HEALTHCHECK

**Checkpoint**: User Story 1 should be fully functional and testable independently

---

## Phase 4: User Story 2 - Developer Feedback & Reproducible Builds (Priority: P2)

**Goal**: Developers can reproduce CI image builds locally and verify digest parity with CI

**Independent Test**: `make build-image` locally produces the same image digest as CI for the same commit

### Tests for User Story 2

- [ ] T021 [P] [US2] Add reproducible-build test `tests/integration/test_reproducible_build.sh` that compares local vs CI-produced digest for a commit

### Implementation for User Story 2

- [ ] T022 [US2] Add `scripts/repro/build_and_label.sh` to ensure deterministic build arguments and labels (`git-commit` and build metadata) are applied
- [ ] T023 [US2] Add documentation in `specs/1-dotnet-container/reproducible-build.md` describing steps to verify digests and reproducibility
- [ ] T024 [US2] Add `Makefile` target `make compare-digest` that fetches CI artifact digest and compares against local build

**Checkpoint**: Local builds match CI digests for the same commit

---

## Phase 5: User Story 3 - Secure Configuration & Secret Access (Priority: P2)

**Goal**: Containers access secrets securely via Managed Identity / Key Vault without embedding secrets or leaking them to logs or disk

**Independent Test**: Deploy an ephemeral container with a managed identity that retrieves a secret; assert secret never appears in logs or filesystem

### Tests for User Story 3

- [ ] T025 [P] [US3] Add integration test `tests/integration/test_keyvault_access.cs` that deploys to a local emulator (or uses CI environment) and asserts secret retrieval succeeds and no secret material is persisted or logged

### Implementation for User Story 3

- [ ] T026 [US3] Add `src/services/keyvault_client.cs` example using `DefaultAzureCredential` and document how to configure a managed identity in `specs/1-dotnet-container/infra.md`
- [ ] T027 [US3] Add CI job to validate `No secrets in env vars` by scanning workflow and generated images for long-lived secrets
- [ ] T028 [US3] Add runtime behavior to safely handle secret provider unavailability (`src/services/retry_policy.cs`) and tests to assert fail-fast and non-sensitive error reporting

**Checkpoint**: Secrets are retrieved at runtime with least-privilege and not persisted

---

## Phase N: Polish & Cross-Cutting Concerns

- [ ] T029 Documentation: Add `docs/container-quickstart.md` with full instructions for reviewers and operators
- [ ] T030 [P] Add `specs/1-dotnet-container/tests/ci_log_scan_fixture.sh` to reuse the log-scan step across tests
- [ ] T031 [P] Add acceptance checklist items to PR template to ensure reviewers verify PII, secrets, SBOMs, and CI gates
- [ ] T032 [P] Add additional unit tests and refactoring tasks as needed in `tests/unit/`
- [ ] T033 [P] Run quick demo: Build in CI and deploy to ephemeral environment to validate end-to-end (document steps in `specs/1-dotnet-container/quickstart.md`)
- [ ] T034 [P] Add image signing & verification step using `cosign` in CI (`scripts/ci/sign-image.sh` and verify in `.github/workflows/ci-build-and-scan.yml`)
- [ ] T035 [US1] Add integration test `tests/integration/test_memory_cap_enforcement.sh` to run container under memory limits, trigger large input, and assert fail-fast behavior with non-sensitive diagnostics

---

## Dependencies & Execution Order

- Foundation (Phase 2) MUST be complete before User Stories (Phase 3+)
- MVP suggestion: Complete Phase 1 and Phase 2, then implement User Story 1 (T015-T020)

---

## Summary

- **Total tasks**: 35
- **Tasks by story**: US1: 9 (T013-T020,T035), US2: 4 (T021-T024), US3: 4 (T025-T028), Setup/Foundation/Polish: 18 (T001-T012, T029-T034)
- **Parallel opportunities**: Foundational [P] tasks (T007-T012) and many polish tasks (T030-T034)
- **MVP scope**: Phase 1 + Phase 2 + User Story 1 (T001-T020, T035)
- **Path**: `specs/1-dotnet-container/tasks.md`

---
