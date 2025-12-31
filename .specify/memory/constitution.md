<!--
Sync Impact Report
- Version change: template -> 1.0.0
- Modified principles: (new adoption from initial governance)
  - [IMPLIED PRIVACY PRINCIPLE] -> Privacy-First (new explicit)
  - [SECRETS] -> Secret Hygiene & Key Vault First (new explicit)
  - [DATA LIFETIME] -> Minimize Data Lifetime (new explicit)
  - [LOGGING] -> Safe Logging & Telemetry (new explicit)
  - [PRIVILEGE] -> Least Privilege & Auditable Access (new explicit)
- Added sections: Security & Compliance; Development Workflow & Quality Gates
- Removed sections: none (template placeholders replaced)
- Templates updated: 
  - .specify/templates/plan-template.md ✅ updated (Constitution Check aligned)
  - .specify/templates/spec-template.md ✅ updated (mandatory PII & security requirements added)
  - .specify/templates/tasks-template.md ✅ updated (security tasks added)
- Follow-up TODOs: 
  - TODO(RATIFICATION_DATE): confirm formal ratification date if different from commit date
  - Ensure CI test skeletons exist and are wired to PR checks (⚠ pending)
-->

# Plumb Dotnet Constitution

## Core Principles

### Privacy-First (NON-NEGOTIABLE)
System design MUST minimize the collection, retention, and exposure of personally identifiable
information (PII). Data collected MUST be limited to the minimum fields necessary for the
business purpose, and only redacted derivatives or aggregated results may be persisted by default.
Where raw or identifiable values are present, they MUST be redacted or stored encrypted with
explicit access controls. Rationale: Protecting user privacy is the primary objective for this
project and is the foundation for all other controls.

### Secret Hygiene & Key Vault First
All secrets, keys, and certificates MUST be stored in an enterprise secret store (Azure Key Vault)
and accessed via Managed Identity. Secrets MUST NOT be stored in environment variables, source
repositories, or build artifacts. Client-side decryption for protected strings MUST require explicit
user authentication and least-privilege Key Vault actions. Rationale: Centralized secret
management reduces exposure and supports rotation/auditability.

### Minimize Data Lifetime
Raw uploads and staging artifacts MUST be ephemeral: raw CSVs and unredacted payloads MUST NOT be
persisted to disk, and memory-based ingestion with strict caps (default hard cap 500MB, soft cap
~120% of declared size) MUST be enforced to avoid spills to disk. Retention of any sensitive data
MUST follow documented retention policies and automated shredding workflows. Rationale: Reduce
attack surface by shortening the lifespan of sensitive artifacts.

### Safe Logging & Telemetry
Logs and telemetry MUST never contain sensitive content (request bodies, merchant strings, PII
fields, unredacted AI payloads). Only non-sensitive metadata (job IDs, statuses, sizes, latency,
errors codes) MAY be recorded. Automated checks for accidental sensitive logging MUST be part
of CI. Rationale: Observability is essential, but MUST be implemented without exposing secrets.

### Least Privilege & Auditable Access
Access control MUST be enforced at row-level where applicable and use identity-based authorization
(Entra ID). Privileges MUST follow least-privilege principles; decryption/cleartext access MUST
require elevated, auditable steps (Entra + Key Vault). All access to sensitive operations MUST
produce non-sensitive audit records sufficient for compliance reviews. Rationale: Limits blast
radius and supports forensic analysis.

## Security & Compliance

This constitution establishes mandatory security and compliance controls that all designs and
implementations MUST follow. Controls include: end-to-end TLS (TLS 1.2+), mTLS for client↔API where
required, Key Vault-only key storage, automated secrets scanning in CI, and explicit mappings to
relevant standards (e.g., NIST, PCI‑DSS) when handling regulated data. Threat models MUST be
maintained for major features and updated in PRs that change data flows.

## Development Workflow & Quality Gates

All changes that affect data handling, authentication, authorization, encryption, or logging MUST
include: an OpenAPI contract (where endpoints exist), automated tests verifying PII redaction and
no-disk persistence of raw payloads, secrets scanning, static analysis for logging violations,
and a security sign-off in PR review. CI pipelines MUST include at least the following gates:
- Secrets scanning and license checks
- Static analysis and security linters
- Tests that assert no raw CSVs are written to disk and no sensitive strings appear in logs
- Contract and integration tests for the API surface

## Enforcement & Amendment Procedure

- The Constitution is the authoritative governance document for all repository activity related to
  PII handling and related components. Code and plan-level exceptions MUST be explicitly documented
  in PR descriptions and require an explicit approval from the Security Owner (or delegated
  reviewer) and one architect.
- Amendments MUST be proposed as a PR that updates this document and provides a short migration
  plan and automated checks that validate the change. Versioning follows semantic versioning:
  - MAJOR: Backwards-incompatible governance or principle removals/changes
  - MINOR: Additive changes (new principle, new mandatory requirement)
  - PATCH: Clarifications, wording fixes, non-semantic refinements
- The PR must include: (a) rationale, (b) change impact assessment (affected templates/specs/tasks),
  and (c) test updates. After approval and merge, **Last Amended** is set to the merge date.

**Version**: 1.0.0 | **Ratified**: 2025-12-31 | **Last Amended**: 2025-12-31

