# IAG5 Helm Chart — Claude Reference

## Project Overview

Helm chart for **Itential Automation Gateway 5 (IAG5)**, a gRPC-based automation execution platform. Chart version 1.0.5, app version 5.1.1, targeting Kubernetes 1.31+ and Helm v3.15.0+.

## Repository Layout

```
iag5-helm/
├── README.md                                   # User-facing docs: requirements, secrets, architectures
├── .cr.yaml                                    # chart-releaser config (owner=Itential, branches)
├── charts/iag5/
│   ├── Chart.yaml                              # Chart metadata + dependencies
│   ├── values.yaml                             # Default values (only values file tracked in git)
│   ├── .helmignore                             # Excludes .git, IDE files, backups from chart package
│   ├── templates/
│   │   ├── _helpers.tpl                        # Named templates
│   │   ├── NOTES.txt                           # Post-install deployment summary shown to user
│   │   ├── deployment-server.yaml
│   │   ├── deployment-runner.yaml              # Generates N deployments via loop
│   │   ├── service.yaml                        # LoadBalancer for servers
│   │   ├── service-runner.yaml                 # ClusterIP per runner
│   │   ├── issuer.yaml                         # cert-manager Issuer
│   │   ├── certificate.yaml                    # cert-manager Certificate
│   │   └── tests/                              # Helm test jobs
│   └── tests/                                  # helm-unittest unit tests
└── .github/workflows/
    ├── helm-ci.yaml                            # Lint + unit tests on PR and push to main
    └── release.yaml                            # chart-releaser + GitHub Pages on charts/** to main
```

## Optional Dependencies

All are conditional and disabled by default unless toggled via their respective `enabled` key:
- `etcd` (Bitnami ^11.3.0) — condition: `etcd.enabled` — in-cluster etcd for distributed mode
- `cert-manager` (Jetstack ^1.12.3) — condition: `certManager.enabled` — TLS certificate management
- `external-dns` (kubernetes-sigs ^1.17.0) — condition: `external-dns.enabled` — DNS record automation

## Deployment Modes

| Mode | `serverSettings.replicaCount` | `runnerSettings.replicaCount` | `storeBackend` |
|------|-------------------------------|-------------------------------|----------------|
| Simple (dev) | 1 | 0 | `memory` |
| HA Server | >1 | 0 | `memory` or `etcd` |
| Distributed | 1 | >0 | `etcd` or `dynamodb` |
| Distributed HA | >1 | >0 | `etcd` or `dynamodb` |

Storage backends: `memory`, `etcd`, `dynamodb`, `local`

## Key Templates

### deployment-server.yaml
- Renders only if `serverSettings.replicaCount > 0`
- Deployment replicas = `serverSettings.replicaCount`
- Sets `GATEWAY_DISTRIBUTED_EXECUTION` and `GATEWAY_HA` env vars derived from replica counts
- Mounts: `itential-gateway-secrets` → `/etc/gateway/keys/encryption.key`, TLS certs → `/etc/ssl/gateway/`, etcd certs → `/etc/ssl/etcd/`

### deployment-runner.yaml
- Uses `range $i := until $replicaCount` loop to generate **one independent Deployment per runner index** (0 to N-1)
- Negative `runnerSettings.replicaCount` is handled gracefully (treated as 0)
- Each runner gets `GATEWAY_RUNNER_ANNOUNCEMENT_ADDRESS` set to its own ClusterIP service DNS

### service.yaml / service-runner.yaml
- Main service: LoadBalancer, selects only `app.kubernetes.io/component: server` pods
- Runner services: ClusterIP, one per runner index, enables stable DNS for announcement addresses
- Service name configurable via `service.name` (default: `iag5-service`)

### certificate.yaml
- Generates DNS SANs for all Kubernetes DNS variants of each service:
  - `name`, `name.namespace`, `name.namespace.svc`, `name.namespace.svc.cluster.local`
- Also generates all four variants per runner service index
- Additional SANs via `certificate.dnsNames` and `certificate.additionalServices`
- Deduplicates with `uniq` + `sortAlpha` filters

### issuer.yaml
- Renders only if `issuer.enabled: true`
- Kind is configurable: `Issuer` or `ClusterIssuer`
- CA-based: references the `issuer.caSecretName` secret for the signing CA

### NOTES.txt
- Displayed to user after `helm install` / `helm upgrade`
- Shows: server/runner counts, distributed/HA flags, TLS status, store backend, etcd/DynamoDB settings

### _helpers.tpl
Defines: `iag5.name`, `iag5.fullname`, `iag5.chart`, `iag5.labels`, `iag5.selectorLabels`, `iag5.serviceAccountName`, `iag5.annotations`

## Required Pre-existing Secrets

| Secret Name | Required When | Contents |
|---|---|---|
| (user-defined) | Always | Image pull credentials — name set in `imagePullSecrets[0].name` |
| `itential-ca` | TLS enabled | `tls.crt` + `tls.key` — CA cert/key for cert-manager Issuer |
| `itential-gateway-secrets` | Always | `gatewayEncryptionKey` (base64, 256-char) mounted at `/etc/gateway/keys/encryption.key` |
| `dynamodb-aws-secrets` | DynamoDB backend | AWS env vars: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`, `AWS_REGION` |
| `etcd-tls-secret` (configurable via `etcdTlsSecretName`) | etcd + TLS | `ca.crt`, `tls.crt`, `tls.key` for etcd client connections |

## Key values.yaml Sections

```yaml
hostname: "iag5.example.com"
port: 50051
useTLS: true

image:
  repository: ""              # Must be set — ECR image repo
  tag: "5.1.1-amd64"         # Defaults to appVersion
  pullPolicy: IfNotPresent

imagePullSecrets:
  - name: ""                  # Must be set — name of pull secret

serverSettings:
  replicaCount: 1
  connectEnabled: true        # Connect to Gateway Manager (IAP)
  connectHosts: "itential.example.com:8080"
  connectInsecureEnabled: false
  env: []                     # Extra env vars for server pods only

runnerSettings:
  replicaCount: 0
  env: []                     # Extra env vars for runner pods only

applicationSettings:
  clusterId: "cluster_1"
  logLevel: "DEBUG"           # INFO | DEBUG | WARN | ERROR
  storeBackend: "memory"      # memory | etcd | dynamodb | local
  etcdHosts: "etcd.default.svc.cluster.local:2379"
  etcdUseTLS: true
  etcdUseClientCertAuth: true
  etcdTlsSecretName: "etcd-tls-secret"
  dynamodbTableName: ""
  env: []                     # Extra env vars for ALL pods

resources:
  enabled: true
  servers:  { limits: {cpu: "2", memory: "4Gi"}, requests: {cpu: "1", memory: "2Gi"} }
  runners:  { limits: {cpu: "6", memory: "10Gi"}, requests: {cpu: "4", memory: "8Gi"} }

livenessProbe:
  enabled: true
  initialDelaySeconds: 10
  periodSeconds: 10

readinessProbe:
  enabled: true
  initialDelaySeconds: 5
  periodSeconds: 10

issuer:
  enabled: true
  kind: "Issuer"              # Issuer | ClusterIssuer
  name: "iag5-ca-issuer"
  caSecretName: "itential-ca"

certificate:
  enabled: true
  duration: "2160h"           # 90 days
  renewBefore: "48h"
  dnsNames: []
  additionalServices: []
  ipAddresses: []
  subject:
    organizations: ["Itential"]
    countries: ["US"]
    localities: ["Atlanta"]
    provinces: ["Georgia"]

service:
  type: LoadBalancer
  name: "iag5-service"

tests:
  enabled: true
  processCount:
    server: { min: 1 }
    runner: { min: 1 }

# Pod-level overrides (all default to empty/disabled)
podAnnotations: {}
podLabels: {}
podSecurityContext: {}
securityContext: {}
nodeSelector: {}
tolerations: []
affinity: {}
volumes: []
volumeMounts: []
```

## Testing

### Unit Tests (helm-unittest)
```bash
helm unittest --strict charts/iag5
```
Test files in `charts/iag5/tests/`:
- `deployment-server_test.yaml` — server deployment rendering, env vars, probes, resources
- `deployment-runner_test.yaml` — runner loop, announcement addresses, negative replicaCount
- `service_test.yaml` — selectors, ports, type
- `service-runner_test.yaml` — per-runner service creation, selectors
- `certificate_test.yaml` — DNS SAN generation, deduplication, subjects, IP addresses
- `issuer_test.yaml` — enable/disable, kind, CA secret ref
- `test-values.yaml` — shared test fixture (2 servers, 3 runners, etcd backend)

### Integration Tests (Helm Tests)
```bash
helm test <release-name>
```
Jobs in `templates/tests/`:
- `test-connection.yaml` — TCP connect (`nc -zv`) on port 50051
- `test-version.yaml` — `iagctl version` in each pod compared against `image.tag`
- `test-processes.yaml` — confirms minimum process counts via `ps aux | grep iagctl`

### CI
- **helm-ci.yaml**: `helm lint` + unit tests on every PR and push to main (Helm v3.15.0)
- **release.yaml**: chart-releaser + GitHub Pages deploy on `charts/**` changes to main (Helm v3.13.0)

## Environment Variable Hierarchy (all pods)

1. Hardcoded defaults in templates
2. `applicationSettings.*` values
3. `applicationSettings.env` — all pods
4. `serverSettings.env` — server pods only
5. `runnerSettings.env` — runner pods only

## Probes

Both liveness and readiness use `exec: pgrep iagctl` — process presence check, not network connectivity. Both have an `enabled` flag in values.yaml.

## Annotations Helper

`iag5.annotations` injects an Itential copyright/license comment into every resource's annotations block, plus the source template filename.

## Notes

- Chart hosted on GitHub Pages (`gh-pages` branch) via chart-releaser
- Only `values.yaml` is tracked in git; all `values-*.yaml` files are gitignored (dev/env-specific)
- Negative `runnerSettings.replicaCount` is handled gracefully (treated as 0)
- `itential-ca.yml` in the chart directory is a reference CA Secret manifest — not deployed by the chart; user must create the real secret manually
