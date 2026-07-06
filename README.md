# Cloudflare Helm Charts

### About
A convenient location to publish Cloudflare helm charts.

### Setup
```bash
helm repo add cloudflare https://cloudflare.github.io/helm-charts
helm repo update
```

### Discovery
```bash
helm search repo cloudflare
```

### Contents

- **`charts/cloudflare-tunnel-remote`** — Deploys a **remotely-managed** Cloudflare
  tunnel (one already provisioned in the Cloudflare Zero Trust dashboard) using a
  tunnel token. This is the chart currently in use.
- `charts/cloudflare-tunnel` — Locally-managed tunnel using a `credentials.json`
  file and a `config.yaml`. **Not currently in use / unmaintained** — prefer
  `cloudflare-tunnel-remote` for new deployments.

---

## cloudflare-tunnel-remote

Runs the `cloudflare/cloudflared` image as a Deployment that connects to a
tunnel you have already created in the Cloudflare dashboard. Authentication is
done entirely with the tunnel token — all ingress/routing configuration lives in
Cloudflare, not in the chart.

### How authentication works

`cloudflared` reads its token from the `TUNNEL_TOKEN` environment variable. The
Deployment wires that variable to a key in a Kubernetes Secret:

```text
TUNNEL_TOKEN  <-  secretKeyRef(name: <secret>, key: <secretKey>)
```

You can supply the token two ways:

1. **Chart-managed Secret (default).** Set `cloudflare.tunnel_token` and the chart
   creates a Secret containing the token under the `tunnelToken` key. A
   `checksum/secret` pod annotation rolls the Deployment whenever the token
   changes. If the token is empty, templating fails with a clear error rather
   than deploying a broken pod.

2. **Pre-existing Secret.** Set `cloudflare.existingSecret` to the name of a
   Secret you manage yourself (e.g. via External Secrets, Sealed Secrets, or
   SOPS) so the raw token never lives in your Helm values. The chart creates **no**
   Secret in this mode and reads the token from that Secret instead. Use
   `cloudflare.existingSecretKey` if the token is stored under a key other than
   `tunnelToken`.

### Installation

Using a token directly:
```bash
helm install my-tunnel cloudflare/cloudflare-tunnel-remote \
  --set cloudflare.tunnel_token=<YOUR_TUNNEL_TOKEN>
```

Using a Secret you already manage:
```bash
# Your Secret must contain the token under the key "tunnelToken"
# (or set cloudflare.existingSecretKey to match your key).
helm install my-tunnel cloudflare/cloudflare-tunnel-remote \
  --set cloudflare.existingSecret=my-cloudflared-secret
```

### Configuration

| Key | Description | Default |
| --- | --- | --- |
| `cloudflare.tunnel_token` | Token for the remotely-managed tunnel. Required unless `existingSecret` is set; ignored when it is. | `""` |
| `cloudflare.existingSecret` | Name of a pre-existing Secret to read the token from. When set, the chart creates no Secret. | `""` |
| `cloudflare.existingSecretKey` | Key within the Secret (chart-created or existing) that holds the token. | `tunnelToken` |
| `image.repository` | cloudflared image repository. | `cloudflare/cloudflared` |
| `image.tag` | Image tag; overrides `appVersion` when set. | `""` (→ `latest`) |
| `image.pullPolicy` | Image pull policy. | `IfNotPresent` |
| `replicaCount` | Number of cloudflared replicas. | `2` |
| `imagePullSecrets` | Image pull secrets. | `[]` |
| `nameOverride` / `fullnameOverride` | Override generated names. | `""` |
| `serviceAccount.name` | Service account name; generated if empty. | `""` |
| `serviceAccount.annotations` | Annotations for the service account. | `{}` |
| `podAnnotations` / `podLabels` | Extra pod metadata. | `{}` |
| `metricsService.annotations` / `.labels` | Metadata for the metrics Service. | `{}` |
| `serviceMonitor.enabled` | Create a Prometheus `ServiceMonitor`. | `false` |
| `serviceMonitor.labels` / `.annotations` | Extra metadata for the ServiceMonitor. | `{}` |
| `serviceMonitor.namespace` | Namespace for the ServiceMonitor; defaults to release namespace. | `""` |
| `serviceMonitor.interval` / `.scrapeTimeout` / `.path` | Scrape settings. | `30s` / `10s` / `/metrics` |
| `podSecurityContext` | Pod-level security context. | runs as non-root (`65532`) |
| `securityContext` | Container-level security context. | drops all caps, read-only rootfs |
| `resources` | Container resource requests/limits. | `{}` |
| `nodeSelector` / `tolerations` / `affinity` | Scheduling controls. | `{}` / `[]` / `{}` |

### Metrics

cloudflared exposes metrics on `0.0.0.0:2000`, which the liveness probe uses via
`/ready`. Set `serviceMonitor.enabled=true` to scrape `/metrics` with the
Prometheus Operator.

---

## cloudflare-tunnel (not currently in use)

A locally-managed tunnel chart that authenticates with a `credentials.json` file
(mounted from a Secret containing `AccountTag`, `TunnelID`, and `TunnelSecret`)
and a `config.yaml` ConfigMap defining ingress rules. It is retained for
reference but is **not currently in use** — new deployments should use
`cloudflare-tunnel-remote`.
