# Deploying to GKE Autopilot

## Prerequisites

1. Install [gcloud](https://cloud.google.com/sdk/docs/install) and [werf](https://werf.io/documentation/v2/quickstart.html).

2. Configure gcloud:

```sh
gcloud auth login
gcloud auth configure-docker <your_region>-docker.pkg.dev
gcloud config set project <your_project_id>
gcloud config set compute/region <your_region>
gcloud container clusters get-credentials primary
```

## Configure Helm Values

Before deploying, create a private values file from the example and fill in the
deployment-specific settings:

```sh
cp .helm/values.example.yaml .helm/values.yaml
```

Do not commit `.helm/values.yaml`; it contains API keys, OAuth secrets, database
URLs, and cookie/encryption secrets.

The chart creates separate Kubernetes secrets from `main.envVars` for the app
and `zero.envVars` for Zero cache. Use `.helm/values.example.yaml` as the source
of truth for the required keys.

For `baseDomain: example.com`, the chart exposes these public hostnames:

- `https://example.com` for the marketing site.
- `https://www.example.com` as a redirect to the marketing site.
- `https://ktb.example.com` for the main app.
- `https://zero.ktb.example.com` for Zero cache.

## Deploy

```sh
werf converge
```

## Setting up a new GKE Autopilot cluster

1. Create the cluster in the GCP console.

2. Configure gcloud:

```sh
gcloud container clusters get-credentials <your_cluster_name>
```

3. Create managed database and cache instances.

Create a Cloud SQL for PostgreSQL instance, database, and user for the app. Make
sure it is reachable from the GKE cluster, preferably through private IP, then
use its connection string in your Helm values.

Create a Memorystore for Redis instance reachable from the same cluster/network,
then use its Redis endpoint in your Helm values for the app's queue backing
store.

4. Configure Cloud SQL for Zero.

Zero requires PostgreSQL 15+ with logical replication enabled. For Cloud SQL for
PostgreSQL, enable logical decoding with the `cloudsql.logical_decoding`
database flag. See Zero's
[Google Cloud SQL notes](https://zero.rocicorp.dev/docs/connecting-to-postgres#google-cloud-sql).

```sh
gcloud sql instances patch <cloud_sql_instance_name> \
  --database-flags=cloudsql.logical_decoding=on
```

Cloud SQL may restart the instance after the flag change. Afterward, use direct
Cloud SQL connection strings for `ZERO_UPSTREAM_DB`, `ZERO_CHANGE_DB`, and
`ZERO_CVR_DB`. The example values point all three at the same PostgreSQL
instance and database.

5. Choose a TLS setup.

The Helm chart references a Kubernetes TLS secret by default. You can create that
secret yourself, or enable the optional cert-manager `Certificate` resource and
point it at an issuer that you manage separately.

The chart serves Zero at `zero.ktb.<base_domain>` so the app's
`ktb.<base_domain>` auth cookie is sent to both the app and Zero cache. Make
sure your TLS certificate covers that nested Zero hostname; the chart's
cert-manager `Certificate` includes it explicitly because `*.example.com` does
not cover `zero.ktb.example.com`.

To bring your own certificate, create a TLS secret in the app namespace:

```sh
kubectl create secret tls <tls_secret_name> \
  --namespace <app_namespace> \
  --cert <path_to_tls_crt> \
  --key <path_to_tls_key>
```

Configure the chart to use it:

```yaml
tls:
  enabled: true
  secretName: <tls_secret_name>

certManager:
  enabled: false
```

6. Optionally install cert-manager:

```sh
# Request values copied from https://oneuptime.com/blog/post/2026-01-17-helm-cert-manager-tls-certificates/view
# Note: GKE Autopilot will adjust requests to meet its supported minimums.
helm upgrade --install cert-manager cert-manager \
  --repo https://charts.jetstack.io \
  --create-namespace \
  --namespace cert-manager \
  --set resources.requests.cpu=50m \
  --set resources.requests.memory=64Mi \
  --set webhook.resources.requests.cpu=25m \
  --set webhook.resources.requests.memory=32Mi \
  --set cainjector.resources.requests.cpu=25m \
  --set cainjector.resources.requests.memory=64Mi \
  --set startupapicheck.resources.requests.cpu=25m \
  --set startupapicheck.resources.requests.memory=32Mi \
  --set crds.enabled=true \
  --set crds.keep=true \
  --set global.leaderElection.namespace=cert-manager
```

7. Verify cert-manager install:

```sh
kubectl get pods -n cert-manager
```

You should see three pods running: cert-manager, cert-manager-cainjector, and cert-manager-webhook.

For GKE with CloudDNS DNS-01 validation, create the CloudDNS credentials secret
in cert-manager's cluster resource namespace, which is `cert-manager` by default:

```sh
kubectl create secret generic clouddns-dns01-solver-svc-acct \
  --namespace cert-manager \
  --from-file=key.json=<path_to_service_account_key_json>
```

Then create a cluster-scoped issuer for your cluster:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-dns01
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: you@example.com
    privateKeySecretRef:
      name: letsencrypt-dns01-account-key
    solvers:
      - dns01:
          cloudDNS:
            project: your-gcp-project-id
            serviceAccountSecretRef:
              name: clouddns-dns01-solver-svc-acct
              key: key.json
```

Prefer Workload Identity over a static service account key for long-lived GKE
clusters. If you do use a static key, keep it out of Helm values.

Enable the chart's cert-manager `Certificate` resource after the issuer exists:

```yaml
tls:
  enabled: true
  secretName: <tls_secret_name>

certManager:
  enabled: true
  issuerRef:
    kind: ClusterIssuer
    name: letsencrypt-dns01
```

8. Create a regional static IP in the GCP console.

Point DNS for `base_domain`, `www.<base_domain>`, `ktb.<base_domain>`, and
`zero.ktb.<base_domain>` at this static IP. Zero uses the nested hostname so the
app's `ktb.<base_domain>` auth cookie is shared with the Zero cache origin.

9. Install ingress-nginx:

```sh
# Request values copied from the ingress-nginx helm chart.
# Note: GKE Autopilot will adjust requests to meet its supported minimums.
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --create-namespace \
  --namespace ingress-nginx \
  --set controller.admissionWebhooks.createSecretJob.resources.requests.cpu=10m \
  --set controller.admissionWebhooks.createSecretJob.resources.requests.memory=20Mi \
  --set controller.admissionWebhooks.patchWebhookJob.resources.requests.cpu=10m \
  --set controller.admissionWebhooks.patchWebhookJob.resources.requests.memory=20Mi \
  --set controller.service.loadBalancerIP=<your_static_ip> \
  --set controller.allowSnippetAnnotations=true \
  --set controller.config.annotations-risk-level=Critical
```
