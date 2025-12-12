# Jellycat Draft UI Helm Chart

Helm chart for deploying the **Jellycat Draft UI** (Go microservice) as a root-level application in your Kubernetes cluster.

> **Note:** The Jellycat Draft UI has been migrated from Next.js/React to Go. This chart has been updated to support the new Go-based architecture. See the [Migration Notes](#migration-from-nextjs-to-go) section for details.

---

## TL;DR

```bash
helm repo add jellycat-draft https://billy-davies-2.github.io/jellycat-draft-helm-chart
helm install jellycat-ui jellycat-draft/jellycat-ui \
  --namespace jellycat --create-namespace
```

> Adjust the repo URL and release name to match how you publish this chart.

---

## Requirements

- Kubernetes cluster
- Helm 3.x
- PostgreSQL database (or use memory/SQLite for development)
- NATS server with JetStream (for real-time messaging)
- Authentik OAuth2 provider (for authentication)
- (Optional) ClickHouse (for analytics)
- (Optional) Ingress / Gateway + HTTPRoute configured to route traffic to the `jellycat-ui` Service.

---

## Installing the Chart

From the chart root (this directory):

```bash
helm upgrade --install jellycat-ui . \
  --namespace jellycat --create-namespace
```

To uninstall:

```bash
helm uninstall jellycat-ui -n jellycat
```

---

## Image Configuration

The chart is preconfigured to use the public GHCR image for the UI (Go-based microservice):

```yaml
image:
  repository: ghcr.io/billy-davies-2/jellycat-draft-ui
  tag: latest
  pullPolicy: IfNotPresent
  tagOverride: ""
```

- `image.repository`: container image repository
- `image.tag`: default tag (usually `latest` or a version)
- `image.tagOverride`: optional tag that takes precedence over `image.tag` (useful for CI deploys)

Example overrides:

```bash
# Deploy a specific version
helm upgrade --install jellycat-ui . \
  -n jellycat \
  --set image.tag=v0.2.0

# One-off deployment using tagOverride (keeps values.yaml at latest)
helm upgrade --install jellycat-ui . \
  -n jellycat \
  --set image.tagOverride=sha-1234567
```

If your registry requires authentication, create a pull secret and reference it:

```bash
kubectl create secret docker-registry ghcr-creds \
  -n jellycat \
  --docker-server=ghcr.io \
  --docker-username=<USERNAME> \
  --docker-password=<TOKEN>
```

Then in `values.yaml`:

```yaml
imagePullSecrets:
  - name: ghcr-creds
```

---

## Service & Probes

Key service-related values in `values.yaml`:

```yaml
service:
  type: ClusterIP
  port: 3000      # HTTP server port
  grpcPort: 50051 # gRPC server port
  annotations: {}

healthPath: /api/health    # Legacy health check
livenessPath: /healthz     # Kubernetes liveness probe
readinessPath: /readyz     # Kubernetes readiness probe
```

- The Deployment exposes the container on both HTTP (`service.port`, default `3000`) and gRPC (`service.grpcPort`, default `50051`)
- **gRPC is required** for NATS-based real-time chat and event streaming
- Liveness probe uses `/healthz` to check if the application is running
- Readiness probe uses `/readyz` to check if the application is ready to serve traffic (checks database connectivity)

---

## gRPC and NATS

The application uses gRPC for real-time features:

- **gRPC Server** (port 50051): Provides API endpoints and real-time event streaming
- **NATS Integration**: gRPC's `StreamEvents()` method provides real-time updates via NATS pub/sub
- **Chat Interface**: Chat messages use NATS for distributed messaging through gRPC

**Important:** gRPC must be enabled when NATS is enabled. The chart will fail validation if you try to enable NATS without gRPC.

---

## Basic Scaling & Resources

```yaml
replicaCount: 1

resources:
  requests:
    cpu: 50m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi
```

Scale the app:

```bash
helm upgrade --install jellycat-ui . -n jellycat \
  --set replicaCount=3
```

---

## Auth, Database, and NATS

This chart exposes flexible configuration for:

- **Auth** (Authentik OAuth2) via `auth.*` values
- **Database** (Memory, SQLite, or PostgreSQL) via `db.*`
- **NATS** (JetStream for real-time messaging) via `nats.*`
- **gRPC** (API server) via `grpc.*`
- **ClickHouse** (optional analytics) via `db.clickhouse.*`

These are wired through environment variables in the `Deployment` template and can be sourced from Vault-synced Secrets (via CSI) or explicit values. See `values.yaml` for detailed examples and comments.

### Secrets: choose ExternalSecrets OR SecretProviderClass

You can wire secrets using one of two patterns. Do not enable both at the same time — the chart will fail the render if it detects both are enabled.

- ExternalSecrets (external-secrets.io):
  - Enable `externalSecrets.enabled=true`
  - Point `externalSecrets.secretStoreRef` and `externalSecrets.remoteRef.basePath`
  - Optionally set `externalSecrets.secretName` (defaults to `auth.vault.secretName` or `<release>-jellycat-ui-auth`).
  - The `Deployment` reads envs from this Secret using the configured keys.

- SecretProviderClass (CSI driver):
  - Enable `auth.vault.enabled=true`
  - Provide `auth.vault.secretProviderClassName` (name of the SPC)
  - Optionally set `auth.vault.secretName` (name of the synced Kubernetes Secret if your SPC defines `secretObjects`)
  - Provide the raw SPC spec under `auth.vault.spec` (provider-specific). The chart renders a `SecretProviderClass` resource using this spec and mounts it at `/vault/secrets` when enabled.

Example ExternalSecrets values:

```yaml
externalSecrets:
  enabled: true
  secretStoreRef:
    name: my-cluster-store
    kind: ClusterSecretStore
  remoteRef:
    basePath: secret/data/jellycat/ui
  secretName: ui-auth-secrets
```

Example SecretProviderClass values (Vault provider — spec is provider-specific):

```yaml
auth:
  vault:
    enabled: true
    secretProviderClassName: ui-auth-spc
    secretName: ui-auth-secrets
    spec:
      provider: vault
      parameters:
        vaultAddress: https://vault.example.com
        roleName: jellycat-ui
        objects: |
          - objectName: clientSecret
            secretPath: secret/data/jellycat/ui
            secretKey: clientSecret
          - objectName: NEXTAUTH_SECRET
            secretPath: secret/data/jellycat/ui
            secretKey: NEXTAUTH_SECRET
      secretObjects:
        - secretName: ui-auth-secrets
          type: Opaque
          data:
            - key: clientSecret
              objectName: clientSecret
            - key: NEXTAUTH_SECRET
              objectName: NEXTAUTH_SECRET
```

### Routing: Ingress or HTTPRoute

Choose one routing method:

- Ingress:
  - Enable with `ingress.enabled=true`
  - Configure `ingress.className`, `ingress.annotations`, `ingress.hosts[]`, and `ingress.tls[]`

- HTTPRoute (Gateway API):
  - Enable with `httpRoute.enabled=true`
  - Reference a `Gateway` via `httpRoute.gateway.{name,namespace}` or provide `parentRefs`
  - Configure `httpRoute.hostnames[]` and `httpRoute.rules[]`

Example (Ingress):

```yaml
ingress:
  enabled: true
  className: nginx
  hosts:
    - host: ui.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - hosts: [ui.example.com]
      secretName: jellycat-ui-tls
```

Example (HTTPRoute):

```yaml
httpRoute:
  enabled: true
  gateway:
    name: public-gateway
    namespace: gateway-system
  hostnames:
    - ui.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
```

### Metrics: ServiceMonitor

If using Prometheus Operator, enable a ServiceMonitor to scrape the UI metrics endpoint:

```yaml
serviceMonitor:
  enabled: true
  labels:
    release: prometheus
  interval: 30s
  scrapeTimeout: 10s
  portName: http
  path: /api/metrics
```

---

## Example: Development Deployment (In-Memory Database)

For local development or testing:

```bash
helm upgrade --install jellycat-ui . \
  -n jellycat --create-namespace \
  --set environment=development \
  --set db.driver=memory \
  --set auth.enabled=false \
  --set nats.enabled=false
```

This configuration:
- Uses in-memory database (no persistence)
- Uses mock NATS (no NATS server required)
- Uses mock ClickHouse (no ClickHouse server required)
- Disables authentication

## Example: Production Deployment

The chart includes production-ready defaults. Simply configure your secrets:

```bash
# Create a secret with your credentials
kubectl create secret generic jellycat-ui-secrets -n jellycat \
  --from-literal=clientSecret=your-authentik-client-secret \
  --from-literal=DATABASE_URL=postgres://user:pass@postgres:5432/jellycat \
  --from-literal=NATS_URL=nats://nats.nats.svc:4222 \
  --from-literal=NATS_USERNAME=jellycat \
  --from-literal=NATS_PASSWORD=secret \
  --from-literal=NATS_CREDS="" \
  --from-literal=CLICKHOUSE_ADDR=clickhouse:9000 \
  --from-literal=CLICKHOUSE_DB=default \
  --from-literal=CLICKHOUSE_USER=default \
  --from-literal=CLICKHOUSE_PASSWORD=secret

# Install the chart with production defaults
helm upgrade --install jellycat-ui . \
  -n jellycat --create-namespace \
  --set auth.baseURL=https://auth.example.com \
  --set auth.clientID=your-client-id \
  --set auth.redirectURL=https://ui.example.com/auth/callback
```

---

## Observability & Rollout

Check rollout status:

```bash
kubectl -n jellycat rollout status deploy/jellycat-ui
```

View logs:

```bash
kubectl -n jellycat logs deploy/jellycat-ui -f
```

---

## Migration from Next.js to Go

The Jellycat Draft UI application has been completely rewritten from Next.js/React to Go. This helm chart has been updated accordingly.

### Breaking Changes

**Environment Variables Removed:**
- `NODE_ENV` - No longer needed for Go application
- `NEXTAUTH_URL`, `NEXTAUTH_SECRET` - Replaced by Authentik OAuth2
- `AUTH_URL`, `AUTH_SECRET`, `BETTER_AUTH_URL`, `BETTER_AUTH_SECRET` - Not used in Go
- `AUTH_TRUST_HOST`, `NEXTAUTH_TRUST_HOST` - Not needed
- `NEXT_PUBLIC_APP_URL`, `PUBLIC_BASE_URL`, `APP_URL`, `BASE_URL` - Frontend specific
- `AUTHENTIK_ISSUER` - Not used in Go OAuth2 implementation
- `NATS_WS_URL` - Go app uses server-side NATS only (no browser WebSocket)
- `CLICKHOUSE_URL` - Split into `CLICKHOUSE_ADDR` and `CLICKHOUSE_DB`
- `CLICKHOUSE_POINTS_TABLE`, `CLICKHOUSE_POINTS_QUERY` - Not used

**Environment Variables Added:**
- `GRPC_PORT` (default: 50051) - gRPC server port
- `ENVIRONMENT` (default: production) - Controls use of mocks vs real services
- `NATS_SUBJECT` (default: draft.events) - NATS JetStream subject
- `AUTHENTIK_REDIRECT_URL` - OAuth2 redirect URL
- `SQLITE_FILE` (default: dev.sqlite) - SQLite database file path

**Environment Variables Renamed:**
- `AUTHENTIK_URL` → `AUTHENTIK_BASE_URL`
- `CLICKHOUSE_URL` → `CLICKHOUSE_ADDR` (address only, e.g., `localhost:9000`)
- Added `CLICKHOUSE_DB` for database name (separate from address)

**Service Changes:**
- gRPC port 50051 now exposed alongside HTTP port 3000
- **gRPC is required for NATS**: The chart will fail validation if NATS is enabled without gRPC
- gRPC provides real-time event streaming and chat functionality via NATS pub/sub
- Health probes updated: `/api/health` → `/healthz` (liveness), `/readyz` (readiness)

**Resource Changes:**
- Memory requirements reduced (Go uses ~50% less memory than Node.js)
- Requests: 64Mi (was 128Mi)
- Limits: 128Mi (was 256Mi)

### Migration Steps

If you're upgrading from the old Next.js-based chart:

1. **Update your secrets** to remove NextAuth/Better Auth secrets and ensure Authentik OAuth2 credentials are present
2. **Update environment variable names** in your values override file
3. **Review database configuration** - the chart now defaults to `postgres` instead of no driver
4. **Review NATS configuration** - remove `wsUrl` if you have it set
5. **Review ClickHouse configuration** - split `url` into `addr` and `database` if using ClickHouse
6. **Test in development** first with `environment: development` to use mocks
7. **Deploy to production** with the new configuration

### What Stayed the Same

- PostgreSQL database support (still the recommended production database)
- SQLite support for development
- Memory driver for testing
- Authentik OAuth2 integration (now simplified)
- NATS JetStream for real-time messaging
- ClickHouse for analytics (optional)
- Ingress/HTTPRoute configuration
- ServiceMonitor support for Prometheus
- Vault CSI and ExternalSecrets support

---

## Values Reference

For a complete list of configurable values, see `values.yaml`. The file is heavily commented to serve as reference documentation.
