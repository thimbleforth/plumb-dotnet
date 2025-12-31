# Example Dockerfile for 1-dotnet-container
# - Multi-stage build
# - Build with official .NET SDK
# - Run with Chainguard nightly runtime image (configurable via ARG)
# - Non-root user, minimal final image

FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

# Copy csproj(s) and restore first for caching
COPY *.sln .
COPY src/ ./src/
RUN dotnet restore

# Build and publish
RUN dotnet publish -c Release -o /app --no-restore

# Final stage: use Chainguard nightly .NET runtime (configurable)
ARG CHAINGUARD_RUNTIME_IMAGE=ghcr.io/chainguard/dotnet:nightly
FROM ${CHAINGUARD_RUNTIME_IMAGE} AS runtime

# Create non-root user & group (explicit uid/gid for reproducibility)
RUN addgroup -S appgroup && adduser -S -G appgroup -u 10001 appuser
WORKDIR /app

# Copy published app from build stage
COPY --from=build /app/ ./

# Ensure proper ownership and minimal permissions
RUN chown -R appuser:appgroup /app && chmod -R 750 /app

# Switch to non-root user
USER appuser:appgroup

ENV DOTNET_RUNNING_IN_CONTAINER=true \
    DOTNET_USE_POLLING_FILE_WATCHER=false

EXPOSE 8080

# Lightweight healthcheck (replace with your app's probe)
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 CMD curl -f http://localhost:8080/health || exit 1

ENTRYPOINT ["dotnet", "YourApp.dll"]

# Notes:
# - Replace 'YourApp.dll' with the actual assembly produced by 'dotnet publish'.
# - Keep final image small and immutable; don't install package managers in final stage.
# - Consider running final image with '--read-only' and a tmpfs for writable directories if supported by your platform.
