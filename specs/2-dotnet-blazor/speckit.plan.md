# speckit.plan — 2-dotnet-blazor

## Summary ✅
A secure, privacy-first cross-platform .NET Blazor frontend that targets mobile and desktop via Blazor Hybrid (.NET MAUI + Blazor). Blazor Hybrid is the primary and required approach for this feature to ensure robust secure token storage and access to platform security APIs. The UI follows Microsoft Fluent Design using the `fluentui-blazor` family (or equivalent approved Fluent Blazor library) and minimizes third-party dependencies: prefer first-party .NET, Azure, and Blazor libraries only. The plan aligns with the `1-dotnet-container` plan for auth, telemetry, secrets, CI gates (SBOM/supply-chain checks), and safe-logging / PII scanning.

---

## Goals (short)
- Ship a secure Blazor-based client that stores secrets only in platform secure storage and minimizes persisted PII
- Use Microsoft Fluent Design via an approved Blazor Fluent UI library for a consistent UX
- Reuse the same security & CI gates as `1-dotnet-container` (SBOM, dependency/license checks, telemetry scrubbing) where applicable
- Keep dependency surface minimal (official Microsoft/Azure packages only where possible)
- Provide a skeleton app, acceptance tests, and CI workflows (build, UI tests, telemetry/payload scanning)

---

## High-level Tech Stack 🔧
- Runtime / Host: .NET 10 (follow repo baseline; prefer latest LTS supported by server)
- UI framework: .NET Blazor (Blazor Hybrid using .NET MAUI for native mobile/desktop)
  - Tradeoff: Blazor Hybrid (MAUI) is the recommended and required implementation for secure token storage & platform APIs. Blazor WebAssembly (WASM/PWA) is not recommended for this project due to weaker secure-storage guarantees and is out-of-scope unless a future, low-sensitivity use case is explicitly approved.
- Fluent UI styling: `fluentui-blazor` (or approved equivalent). Ensure the chosen library is audited and included in SBOM.
- Authentication: MSAL.NET (system/browser integrated flows where applicable) + prefer server-side token exchange when possible to avoid long-lived refresh tokens on client
- Secure storage abstraction: `SecureStorage` (MAUI/Platform Keychain/Keystore/Windows Credential Manager) behind an `ISecureStorage` interface
- Networking: HttpClient with TLS 1.2+ and certificate validation; use authenticated API calls with bearer tokens (prefer short-lived tokens)
- Telemetry: Azure Application Insights or an approved provider supporting payload scrubbing and offline encrypted queue; ensure telemetry is scrubbed client-side before transmission
- CI/CD: GitHub Actions with Roslyn analyzers, static analysis for unsafe logging patterns, dependency/license checks, and release gating (SBOM generation and third-party SDK audits)

---

## Recommended Minimal Libraries (official first-party only) 📦
- Microsoft.Extensions.Logging
- Microsoft.Extensions.Configuration
- Microsoft.Identity.Client (MSAL)
- Microsoft.Maui.* (when using Blazor Hybrid)
- System.Text.Json
- Azure.Identity (for any direct Key Vault interactions, prefer server-side Key Vault usage)
- An approved `fluentui-blazor` package (audit & pin versions)

Rationale: minimize attack surface and license surface by favoring official Microsoft/Azure libraries.

---

## Security & Privacy Considerations 🔒
- Follow the Plumb Dotnet Constitution: Privacy-first, Secret Hygiene, Minimize Data Lifetime, Safe Logging, Least Privilege.
- Token strategy: prefer short-lived access tokens and server-side token exchange where feasible. If refresh tokens are required on client, store them in platform secure storage and apply strict rotation policies.
- Local persistence: encrypted at rest using platform mechanisms; ephemeral caches pruned by short retention policy; offline queues encrypted and auto-shredded on sync.
- Logging & telemetry: add Roslyn analyzers to detect unsafe log statements; all telemetry and crash payloads MUST be scrubbed client-side before send and validated by CI scanning as part of release gate.
- Third-party libraries: audit license and security posture, pin versions, include in SBOM. CI must fail on disallowed licenses.
- Consent & permissions: explicit user consent for telemetry; request runtime permissions only when required and provide localized justification text in UX.

## Constitution Alignment 🏛️
This plan explicitly maps to `constitution.md` and mandates the following verifications and behaviors:
- **No secrets in environment variables** — clients must rely on secure platform storage and server-side Key Vault access; CI must block changes that introduce env‑var secrets.
- **Client defaults to in-memory data**; any local persistence must be encrypted, have short retention, and be prunable on demand (e.g., on logout or remote revoke).
- **No sensitive telemetry or logs** — Roslyn analyzers, CI telemetry scans, and pre-send scrubbing must be part of the release gate.
- **Explicit authorization for raw data** — any request returning raw row-level data requires additional auth checks and Key Vault-based decryption with audit trails.
- **Retention & shredding** — offline queues and local caches must be auto‑shredded per policy; CI simulations should validate shredding behavior.

---

## UI Design Guidelines & Fluent UI Usage ✨
- Use Microsoft Fluent Design language for controls, spacing, and responsive layouts via the approved Blazor Fluent UI library.
- Create a small design system wrapper (local component library) around Fluent components for:
  - Consistent safe-logging wrappers (no accidental logging of bound sensitive fields)
  - Input components that support built-in masking, redaction, and client-side validators
  - Consent and permissions components (standardized prompt and accessible copy)
- Accessibility: ensure WCAG AA compliance for primary flows.
- Minimize custom CSS and prefer theme tokens from Fluent library to reduce maintenance.

---

## Architecture & Integration with `1-dotnet-container` 🔗
- API contract: backend services (containers) must offer token-exchange endpoints that allow clients to trade ephemeral auth for short-lived service credentials where possible.
- Shared expectations:
  - Server side enforces PII minimization and does final redaction/validation
  - Both client & server must run PII scans in CI and enforce safe-logging policies
  - Use the same telemetry policy: redact fields at the source and validate in CI
- Deployment: client releases go through GitHub Actions, produce SBOM for dependencies, and must pass the same third-party license checks used for container workloads.

---

## CI Pipeline (GitHub Actions) 🛠️
- Steps (per branch/build):
  1. checkout + restore + dotnet build + unit tests
  2. run Roslyn analyzers and custom analyzers for unsafe logging
  3. run UI tests (Playwright / Appium for hybrid apps) in CI (smoke flows)
  4. generate SBOM (Syft or dotnet-sbom) and run license checks
  5. run dependency vulnerability checks (GitHub Dependabot alerts + SCA tools)
  6. run telemetry & crash payload scanning tool on sample payloads to assert no PII
  7. sign artifacts (optional) and publish release assets
- Additional gating: require security sign-off for adding new third-party SDKs or altering data flows

Notes:
- For Blazor Hybrid builds: CI must produce signed platform artifacts (where appropriate).

---

## Tests & Acceptance Criteria (mapping to spec) ✅
- Telemetry Scan Test: generate sample telemetry/crash payloads in test harness and run CI scanning; assert no PII patterns found.
- Secure Storage Test: end-to-end auth flow using test tenant and assert tokens exist only in platform secure storage and cannot be read from FS.
- Permission/Consent Test: UI tests that validate permission prompts only appear when features are invoked and that denial is safe.
- Offline Sync Test: simulate offline queue, persist encrypted items, ensure retention pruning after policy period, and assert no PII persisted in plaintext.
- Persistence test: assert client never writes raw sensitive inputs to disk; any local persistence must be encrypted and prunable (add automated UI/integration assertions).
- CI telemetry/log-scan test: add a CI step that scans client logs and telemetry payloads for sensitive patterns and fails the pipeline when matches are found.
- SDK Audit Test: CI fails when new SDKs are introduced without an approved security and license review.

---

## Edge Cases & Operational Concerns
- Lost device: provide a token revocation flow and ensure client can delete local caches on remote revoke command from server.
- Long uploads & memory: process in streaming manner and apply memory caps; fail-fast with non-sensitive diagnostics.
- Third-party SDKs: if a crash SDK cannot guarantee scrubbing, do not use it or wrap its payloads with pre-send scrubbing.

---

## Files to add (minimal)
- `specs/2-dotnet-blazor/speckit.plan.md` (this document)
- `src/Plumb.Client/` (Blazor Hybrid skeleton project)
- `src/Plumb.Client/Components/Fluent/` (design system wrappers)
- `src/Plumb.Client/Infrastructure/Security/ISecureStorage.cs` and platform implementations
- `.github/workflows/client-build-and-scan.yml` (build + analyzers + SBOM + telemetry scan)
- `tests/ui/` (UI automation tests for auth / consent / offline / telemetry)
- `checklists/client-security.md` (review checklist)
- `tests/acceptance/telemetry-scan.test.md` (manifest for telemetry scanning)

---

## Tasks (mapping to TODOs)
1. Draft speckit.plan for `2-dotnet-blazor` (in repo) — DONE ✅
2. Add CI workflows & Roslyn analyzers + telemetry scanning — NEXT
3. Create Blazor Hybrid skeleton and Fluent UI wrappers — NEXT
4. Implement MSAL auth flows & `ISecureStorage` abstractions — NEXT
5. Add telemetry scrubber, encrypted offline queue, and tests — NEXT
6. Add acceptance tests, SBOM generation, and third-party SDK audit checks — NEXT
7. Security review & official sign-off — NEXT

---

## Cost & Tradeoffs 💲
- Blazor Hybrid (MAUI) increases binary size and build complexity but gives strong secure storage and platform APIs — recommended for high-security use cases.
- Blazor WebAssembly (WASM/PWA) is not recommended for this project due to weaker guarantees for secure storage and token safety; if considered in future it must be limited to low-sensitivity features and undergo an explicit security review.
- Keep third-party SDK use minimal; prefer server-side processing for sensitive operations to minimize client surface area.

---

## Dependencies
- Backend token exchange & server-side PII minimization
- CI capabilities for Roslyn analysis, SBOM generation, dependency/license checks, and telemetry scanning
- Approved crash/telemetry provider that supports payload scrubbing and offline encryption
- Platform secure storage support (MAUI/Keychain/Keystore/Windows Credential Manager)

---

## Next Steps (immediate)
- Add `client-build-and-scan.yml` CI workflow and Roslyn analyzers to the repo
- Create the Blazor Hybrid skeleton repository and basic Fluent UI wrapper components
- Implement an `ISecureStorage` abstraction and an MSAL auth POC

If you want, I can implement the CI workflow and the Blazor skeleton next — tell me which to prioritize and I’ll start with that (I can also open PR templates and the initial GitHub Actions workflow for review).