# Initial Governance for PII‑Handling System ✅

**Purpose:** This document captures the explicit and implied governing principles and developer guidelines for the PII-handling system described in `first-draft-design-doc.md`. It is intended as the authoritative, concise reference for architecture, security, and development practices.

---

## Summary
A privacy‑first, secure‑by‑default governance policy for ingestion, processing, storage, and visualization of financial and identity‑linked data. The system must minimize PII collection and persistence, enforce strong encryption and identity‑based access, and be auditable without exposing sensitive content.

---

## Non‑negotiables & Core Principles 🔒
- **No secrets in environment variables** — use **Azure Key Vault** and **Managed Identity** for all secrets.
- **No sensitive logs or telemetry** — never log request bodies, CSV lines, merchant strings, or AI payloads.
- **No raw CSV persistence** — raw uploads must be securely shredded immediately after successful processing.
- **End‑to‑end encryption** — TLS 1.2+ everywhere and **mTLS** for client ↔ API connections.
- **Row‑level authorization** — enforce user‑scoped data partitions and access using Entra ID.
- **Least privilege & defense‑in‑depth** — assume partial compromise and minimize exposure and persistence.

---

## Data Handling & PII Rules 🧾
- Only send necessary, minimized fields to PII detection; **persist only redacted outputs**.
- Merchant identifiers in storage must be stored encrypted. Decryption of plaintext merchant strings requires additional Entra ID + Key Vault authorization and should occur client‑side on demand.
- Ingest CSVs in memory only; enforce a hard cap (default: **500MB**) and a soft cap (default: **120%** of declared size) to avoid disk spill.
- Maintain limited, non‑sensitive audit metadata and strict retention policies for any staged files.

---

## Architecture & Design Guidelines 🏗️
- Follow the three‑zone model: **Client (Blazor)**, **Service (.NET API in container)**, **Data (CosmosDB/SQL + Key Vault)**.
- Service pipeline: validate -> PII detection (Azure AI Language) -> redact -> normalize -> dedupe -> categorize -> store.
- Use a thin server‑side query layer to return aggregated/redacted results; deliver row‑level data only on explicit request with additional decryption checks.
- Prefer managed Azure services: Entra ID, Key Vault, CosmosDB/SQL/Postgres, Azure Container Apps.

---

## Security & Key Management 🔐
- Keys and secrets are **only** in Key Vault; access via Managed Identity. No key material in code or config.
- Client‑side decryption requires explicit Entra authentication and Key Vault operations; log only the fact of decryption (no plaintext details).
- Threat model: plan for Azure admin compromise, device compromise, and accidental leakage; map mitigations to standards (NIST, PCI‑DSS) where applicable.

---

## Logging, Telemetry & Observability ⚠️
- Log job states, metrics, and non‑sensitive metadata only (job IDs, status, sizes, latency).
- Do not log payloads or identifying strings. Add automated checks to detect accidental sensitive logging.
- Verify and record integrity checks (e.g., DB write success) without exposing content.

---

## Operational Requirements & Lifecycle 🔁
- Define ingestion job states: **queued / running / failed / complete**; implement retries and idempotency.
- After successful write to DB: verify integrity, securely overwrite/shred in‑memory raw data, and spin down container.
- If temporary/staging blobs are used: private endpoints, short retention, automatic cleanup.

---

## Developer & Process Guidelines 🛠️
- Make secure defaults mandatory — no opt‑in weakening of security.
- Add an **OpenAPI** spec for all endpoints and require schema validation for uploads.
- Enforce CI checks: secrets scanning, static analysis, security linters, and tests verifying no sensitive logging or disk writes of raw payloads.
- Include threat modeling and compliance mapping as part of design and PR review.

---

## UX / Client Requirements 🖥️
- Blazor clients authenticate via Entra ID (OIDC/OAuth2); keep data in memory by default and offer an optional encrypted per‑session cache that is shredded on close.
- Client requests for raw row data must require additional authorization and Key Vault access for decryption.

---

## Implied Principles — distilled 💡
- Privacy by design: minimize data collection, storage, and exposure.
- Fail‑safe defaults: deny plaintext access unless explicitly authorized.
- Auditability without exposure: keep sufficient metadata for diagnostics and compliance, never sensitive content.
- Operational minimalism: minimize the lifetime and persistence of sensitive artifacts.
- Testability: security constraints must be verifiable via automated tests.

---

## Initial Actionable Checklist ✅
1. Add automated tests to assert uploads never touch disk and raw payloads are never logged.
2. Define and commit an **OpenAPI** specification and upload schema validation; enforce in CI.
3. Implement Key Vault integration via Managed Identity and remove env var secrets.
4. Implement ingestion memory caps and job lifecycle states; instrument and alert on caps.
5. Add PII redaction stage using Azure AI and tests that only redacted output may be stored.
6. Define retention policies and an automated shredding workflow for raw/staged files.
7. Perform a formal threat model and map controls to NIST/PCI‑DSS as required.

---

## Enforcement & Testing 🔍
- Add CI tests for: no writes of CSVs to disk, no sensitive strings in logs, Key Vault access patterns, and ingestion memory/use behavior.
- Include security checks in PR templates and require sign‑off for changes to data flow or retention rules.
