# speckit.plan — 1-dotnet-container

## Summary ✅
A minimal-cost, security-first tech stack for running .NET 10 LTS containerized services on Azure.
The plan uses the Chainguard nightly .NET runtime in the final runtime image, a Dockerfile multi-stage build, and lean Azure services (ACR + Container Apps or AKS depending on scale). CI uses GitHub Actions with open-source scanning tools (Syft for SBOM, Trivy for vulnerability scans) and OIDC to authenticate to ACR (no long-lived CI secrets).

---

## Goals (short)
- Reproducible, content-addressable images (digest stability per commit)
- Minimal extra dependencies (use built-in .NET and official Azure libraries only)
- Least-cost Azure services with strong security posture
- Image scanning, SBOM, and CI gates as part of the build pipeline
- Images run as non-root, with read-only filesystem and strict process limits

---

## High-level Tech Stack
- Runtime: .NET 10 LTS
- Base runtime image: Chainguard "nightly" .NET runtime image (final stage)
  - Use a build ARG to make the image tag configurable so teams can pin or change nightly tags easily
- Build/SDK image: Official Microsoft .NET SDK image (for restore/build/publish)
- Container store: Azure Container Registry (ACR) — Basic tier for cost savings
- Orchestration / runtime platform:
  - Primary recommendation: Azure Container Apps (ACA) — lowest operational cost for microservices / small workers; supports managed identities and resource limits
  - Alternative for heavier workloads: AKS with node pools and managed identities (higher operational cost)
- Secrets: Azure Key Vault with Managed Identity (prefer user-assigned MI for least privilege)
- CI/CD: GitHub Actions (workload identity federation / OIDC for ACR), use standard GitHub runners or self-hosted if available
- Logging & Telemetry: Azure Monitor (Log Analytics) with structured logs, minimal retention, and PII redaction
- Vulnerability scanning & SBOM generation: Syft (SBOM), Trivy (scan). Optionally sign artifacts with cosign.

---

## Recommended Minimal Libraries
- Microsoft.Extensions.Logging
- Microsoft.Extensions.Configuration
- Azure.Identity (DefaultAzureCredential to pick Managed Identity in production / developer credential locally)
- Azure.Security.KeyVault.Secrets (for secret retrieval)
- System.Text.Json (for structured logging and small payloads)

Rationale: These are official Microsoft/Azure libraries and keep third-party dependencies at a minimum.

---

## Security & Cost Tradeoffs 🔒💲
- Least-cost choices: ACR Basic + Azure Container Apps. Avoid always-on VM/AKS clusters for low throughput jobs.
- Security: Use managed identity + Key Vault; use private ACR with firewall + ACR Tasks or GitHub Actions push using OIDC.
- Scanning: Run SBOM generation and vulnerability scanning in CI (Trivy + Syft). These tools are free and effective.
- Runtime hardening: run as non-root, make filesystem mostly read-only, set resource limits via platform (CPU, memory), configure probes.

---

## Dockerfile (recommended, minimal and secure)
- Usage: multi-stage build with SDK stage and Chainguard runtime final stage
- Enforce non-root user, copy only published artifacts, set PATH and healthcheck, use a configurable runtime image ARG

Example:

```dockerfile
# Build stage (use official .NET SDK)
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
WORKDIR /src
COPY *.sln .
COPY src/ ./src/
RUN dotnet restore
RUN dotnet publish -c Release -o /app --no-restore

# Final stage (Chainguard nightly runtime)
ARG CHAINGUARD_RUNTIME_IMAGE=ghcr.io/chainguard/dotnet:nightly
FROM ${CHAINGUARD_RUNTIME_IMAGE} AS runtime

# Create non-root user & app dir (user ids are explicit for reproducibility)
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --from=build /app/ ./

# Use non-root user
RUN chown -R appuser:appgroup /app
USER appuser:appgroup

# runtime environment hardening
ENV DOTNET_RUNNING_IN_CONTAINER=true \
    DOTNET_USE_POLLING_FILE_WATCHER=false

EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 CMD ["/bin/check-health"]

ENTRYPOINT ["dotnet", "YourApp.dll"]

# NOTE: Replace check-health with a small script that checks readiness or HTTP probe.
```

Notes:
- Use an explicit user to avoid root; Chainguard images are minimal and limit attack surface.
- Keep the final image free of SDKs and package managers.
- Consider `--read-only` and a small writable tmp over overlay during runtime if platform supports it.

---

## CI Pipeline (GitHub Actions minimal example)
- Steps: restore/build/test -> publish -> generate SBOM (Syft) -> scan (Trivy) -> sign (cosign optional) -> push to ACR via OIDC

Key points:
- Use OIDC (workload identity federation) to authenticate to ACR (prevents storing long-lived secrets in CI)
- Fail pipeline on policy violations (critical/high CVEs, license issues)

Snippets and recommended actions to add to repo:
- `ci/build-and-scan.yml` with steps:
  - `actions/checkout`
  - `dotnet restore && dotnet test`
  - `docker build` (with `--label=git-commit` and `--label=git-digest` for reproducibility)
  - `syft` to produce SBOM (spdx/json)
  - `trivy` to scan and exit non-zero if policy fails
  - `docker push` to ACR (login via OIDC)

---

## Secrets & Runtime Access
- Use Azure Managed Identity and Key Vault:
  - Grant the container's assigned identity only `get` permissions for secrets required.
  - Access secrets using `DefaultAzureCredential` which will use Managed Identity in production and developer-local credentials when available.
- Never write secrets to disk or logs; use streaming patterns and hold secrets only in memory for the minimum time.

Sample usage in code:
- Use `DefaultAzureCredential` to instantiate `SecretClient` from `Azure.Security.KeyVault.Secrets`.
- Always catch and redact exceptions that may leak secret names/values.

---

## Operational Considerations
- Resource limits: Configure CPU and memory caps from the platform (ACA or AKS). Add settings for timeouts and fail-fast.
- Health & readiness: Provide HTTP endpoints for both, and ensure they do not expose secret or PII.
- Logging: Use structured logs, redact PII at the source, and scan logs in CI for accidental PII pre-release.
- SBOM & image signing: Generate SBOMs and sign images with cosign to assert provenance.

---

## Acceptance Tests to Add (mapping to spec)
- Build and scan test: CI runs and asserts SBOM created, Trivy scan passed, image pushed to ACR.
- Runtime secret access: Deploy ephemeral to Container Apps with managed identity + Key Vault; retrieve and assert secret not written to disk or logs.
- Redaction test: Feed PII input; assert no PII exists on disk or in logs.
- Reproducible build test: Build the same commit locally vs in CI and compare digest.
- Failure mode tests: Simulate Key Vault downtime and large input to assert fail-fast behavior and safe logging.

---

## Files to add (minimal)
- `specs/1-dotnet-container/speckit.plan.md`  (this document)
- `specs/1-dotnet-container/Dockerfile` (example)
- `.github/workflows/build-and-scan.yml` (CI)
- `tests/integration/secret-access.test.md` (test manifest for secret runtime test)
- `checklists/container-security.md` (checklist for reviewers)

---

## Cost-saving tips
- Use ACR Basic and Azure Container Apps for low-traffic workloads.
- Reuse user-assigned managed identities and RBAC roles across environments where possible.
- Use Syft and Trivy in CI (free OSS tools) rather than paid scanners if budget constrained.

---

If you'd like, I can:
- Add the `Dockerfile` example as a file in the repo,
- Add the `build-and-scan` GitHub Actions workflow,
- Add a small C# example function showing Key Vault access with `DefaultAzureCredential`.

Tell me which of the above you'd like me to implement next.
