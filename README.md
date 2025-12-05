# Jellycat Draft UI Helm Chart

Helm chart for deploying the **Jellycat Draft UI** (Next.js) as a root-level application in your Kubernetes cluster.

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

The chart is preconfigured to use the public GHCR image for the UI:

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
  port: 3000
  annotations: {}

healthPath: /api/health
```

- The Deployment exposes the container on `service.port` (default `3000`).
- Both liveness and readiness probes use `healthPath` on the named `http` port.

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

- **Auth** (Authentik / NextAuth / Better Auth) via `auth.*` values
- **Database** (Postgres or ClickHouse) via `db.*`
- **NATS** (server and optional WebSocket) via `nats.*`

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

## Example: Minimal Auth-Disabled Deployment

```bash
helm upgrade --install jellycat-ui . \
  -n jellycat --create-namespace \
  --set auth.enabled=false \
  --set nats.enabled=false
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

## Values Reference

For a complete list of configurable values, see `values.yaml`. The file is heavily commented to serve as reference documentation.
