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
