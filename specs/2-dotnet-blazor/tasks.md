---
description: "Generated tasks for feature: .NET Blazor Frontend"
---

# Tasks: .NET Blazor Frontend

**Input**: `specs/2-dotnet-blazor/spec.md`

## Phase 1: Setup (Shared Infrastructure)

- [ ] T001 Create project skeleton `src/BlazorApp/` and sample pages demonstrating minimal data collection
- [ ] T002 Add GitHub Actions workflow `.github/workflows/ci-blazor.yml` to run build, unit tests, static analyzers, and SBOM generation for client assemblies
- [ ] T003 Add `specs/2-dotnet-blazor/quickstart.md` documenting local development, secure storage behaviors, and test flows
- [ ] T004 Add Roslyn analyzer config and sample analyzer rules in `tools/roslyn/` to detect unsafe logging patterns in client code
- [ ] T005 Add `specs/2-dotnet-blazor/checklists/client-security.md` for reviewer checklist

---

## Phase 2: Foundational (Blocking Prerequisites)

- [ ] T006 [P] Add secure storage abstraction `src/BlazorApp/Services/ISecureStorage.cs` and examples for Keychain/Keystore integration (platform-binding guidance in `specs/2-dotnet-blazor/secure-storage.md`)
- [ ] T007 [P] Add telemetry safe-scrubbing utilities `src/BlazorApp/Telemetry/Scrubber.cs` and unit tests in `tests/unit/test_scrubber.cs`
- [ ] T008 [P] Add CI telemetry/log-scan step `.github/workflows/ci-telemetry-scan.yml` to run sensitive-string scanning against telemetry payloads
- [ ] T009 [P] Add static analyzer rules to CI that fail on unsafe logging and build if Roslyn analyzer reports PII logging
- [ ] T010 Add platform secure storage test harness `tests/integration/test_secure_storage.md` documenting how to validate secure storage on CI or manual runs

---

## Phase 3: User Story 1 - Minimal Data Collection (Priority: P1) 🎯 MVP

**Goal**: App collects and persists only minimal required fields and enforces redaction or encryption where required

**Independent Test**: Automated UI/unit tests assert only declared fields are persisted and telemetry scans report zero PII

### Tests for User Story 1

- [ ] T011 [P] [US1] Add unit tests `tests/unit/test_minimal_persistence.cs` verifying only approved fields are persisted
- [ ] T012 [P] [US1] Add telemetry scan test integration `tests/integration/test_telemetry_scan.sh` that simulates telemetry events and asserts scrubbing

### Implementation for User Story 1

- [ ] T013 [US1] Implement local persistence layer `src/BlazorApp/Storage/LocalStore.cs` with encryption and TTL enforcement
- [ ] T014 [US1] Add UI validation and consent surfaces in `src/BlazorApp/Pages/Consent.razor` and tests in `tests/ui/test_consent_flow.cs` or `playwright/` script
- [ ] T015 [US1] Add PII field declaration file `specs/2-dotnet-blazor/pii-declaration.md` listing fields and handling rules

**Checkpoint**: US1 independently testable

---

## Phase 4: User Story 2 - Secure Authentication & Token Storage (Priority: P1)

**Goal**: Secure OAuth/OpenID flows with tokens stored only in platform secure storage

**Independent Test**: Integration test that authenticates against test tenant and asserts tokens stored only in secure storage

### Tests for User Story 2

- [ ] T016 [P] [US2] Add integration test `tests/integration/test_secure_token_flow.sh` to validate token storage and absence of tokens in filesystem/logs

### Implementation for User Story 2

- [ ] T017 [US2] Integrate MSAL or chosen auth library in `src/BlazorApp/Services/AuthService.cs` and include tests in `tests/unit/test_auth_service.cs`
- [ ] T018 [US2] Add secure storage adapter implementations for supported platforms in `src/BlazorApp/Platform/` and a fallback for desktop/web where applicable
- [ ] T019 [US2] Add CI job that runs the secure token flow against a test tenant or mocked identity provider and fails on leaks

**Checkpoint**: Auth flows validated and secure storage verified

---

## Phase 5: User Story 3 - Safe Telemetry and Crash Reporting (Priority: P1)

**Goal**: Telemetry/crash reports are scrubbed of PII and pass automated CI checks

**Independent Test**: Simulate crash and assert telemetry payloads are PII-free

### Tests for User Story 3

- [ ] T020 [P] [US3] Add integration test `tests/integration/test_crash_reporting_scrub.sh` to simulate error and assert scrubbed payloads

### Implementation for User Story 3

- [ ] T021 [US3] Integrate crash reporting provider adapter `src/BlazorApp/Telemetry/CrashAdapter.cs` with pre-send scrub and add unit tests to validate scrub behavior
- [ ] T022 [US3] Add CI step that runs the crash-scrub integration test in a controlled environment

**Checkpoint**: Crash reports and telemetry pass scrubbing CI checks

---

## Phase N: Polish & Cross-Cutting Concerns

- [ ] T023 Documentation: Add `docs/blazor-privacy.md` covering minimal-data and retention policy
- [ ] T024 [P] Add SBOM generation for client artifacts and include in release pipeline
- [ ] T025 [P] Add checklist items in PR template for secure storage, telemetry, and PII declaration verification
- [ ] T026 [P] Add final demo: end-to-end scenario demonstrating sign-in, consent, telemetry, and offline sync without PII leakage
- [ ] T027 [P] [US1] Add integration test `tests/integration/test_offline_sync.sh` to simulate offline sync, assert encrypted queue, retention pruning, and no PII leakage
- [ ] T028 [P] [US1] Add UI test `tests/ui/test_permission_flow.cs` (or Playwright script) to validate permission prompts appear only when needed and denial flows do not leak PII
- [ ] T029 [P] Add `specs/2-dotnet-blazor/third-party-sdks.md` and CI check `scripts/ci/verify-sdks.sh` to audit third-party SDK versions, licenses, and SBOM inclusion

---

## Dependencies & Execution Order

- Foundation (Phase 2) MUST be complete before User Stories begin
- MVP suggestion: Setup + Foundational + US1 (T001-T014)

---

## Summary

- **Total tasks**: 26
- **Tasks by story**: US1: 5 (T011-T015), US2: 4 (T016-T019), US3: 3 (T020-T022), Setup/Foundation/Polish: 14 (T001-T010, T023-T026)
- **Parallel opportunities**: Foundational tasks marked [P] (T006-T009, T011, T016, T020, T024-T026)
- **MVP scope**: Phase 1 + Phase 2 + User Story 1 (T001-T015)
- **Path**: `specs/2-dotnet-blazor/tasks.md`

---
