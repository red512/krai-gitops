# krai-gitops

Helm charts and ArgoCD application manifests for the krai platform. ArgoCD watches this repo and auto-syncs all applications to GKE.

Part of the [krai](https://github.com/red512/krai) project.

## Directory Structure

```
krai-gitops/
├── argocd/apps/
│   ├── eso-app.yaml                    # ArgoCD App → external-secrets namespace
│   ├── keda-app.yaml                   # ArgoCD App → keda namespace
│   ├── krai-app.yaml                   # ArgoCD App → krai-backend namespace
│   └── krai-frontend-app.yaml          # ArgoCD App → krai-frontend namespace
├── helm/
│   ├── eso/
│   │   ├── Chart.yaml                  # Wrapper chart for external-secrets v0.12.1
│   │   └── values.yaml                 # ESO operator resource limits
│   ├── keda/
│   │   ├── Chart.yaml                  # Wrapper chart for KEDA v2.16.1
│   │   └── values.yaml                 # KEDA operator config + Workload Identity
│   ├── krai-helm-chart/
│   │   ├── Chart.yaml
│   │   ├── values.yaml                 # Image tag, env vars, resource limits
│   │   └── templates/
│   │       ├── deployment.yaml          # API server pods (HPA-scaled)
│   │       ├── worker-deployment.yaml   # Worker pods (KEDA-scaled)
│   │       ├── service.yaml             # LoadBalancer service
│   │       ├── serviceaccount.yaml      # krai-app SA with Workload Identity
│   │       ├── hpa.yaml                 # CPU-based HPA for API pods
│   │       ├── worker-hpa.yaml          # KEDA ScaledObject for worker pods
│   │       ├── keda-trigger-auth.yaml   # TriggerAuthentication (GCP podIdentity)
│   │       ├── cluster-secret-store.yaml # ClusterSecretStore → GCP Secret Manager
│   │       ├── external-secret.yaml     # ExternalSecret → K8s Secret krai-secrets
│   │       └── eso-serviceaccount.yaml  # krai-eso SA with Workload Identity
│   └── krai-frontend-chart/
│       ├── Chart.yaml
│       ├── values.yaml                  # Image tag, env vars, resource limits
│       └── templates/
│           ├── deployment.yaml          # Frontend pods (HPA-scaled)
│           ├── service.yaml             # LoadBalancer service
│           └── hpa.yaml                 # CPU-based HPA for frontend pods
```

## How It Works

1. CI/CD pipelines in krai-backend and krai-frontend push new Docker images to Artifact Registry
2. The CD pipeline updates the image tag in the respective `values.yaml` in this repo
3. ArgoCD detects the commit and syncs the new image to GKE

All ArgoCD applications are configured with `automated.selfHeal: true` and `automated.prune: true`.

## Namespace Layout

```
external-secrets    → ESO operator + webhook + cert-controller
keda                → KEDA operator + metrics server
krai-backend        → API pods + Worker pods + LoadBalancer + ExternalSecret + ClusterSecretStore
krai-frontend       → React pods + LoadBalancer
```

## Secrets Management

The API key flows through: **GCP Secret Manager** → ESO `ClusterSecretStore` → `ExternalSecret` → K8s Secret `krai-secrets` → pod env var via `secretKeyRef`. No plaintext secrets in Helm values.

## Worker Autoscaling

Worker pods scale via KEDA based on Pub/Sub queue depth (`krai-jobs-sub` subscription). The `TriggerAuthentication` uses GCP pod identity via Workload Identity — no static credentials.
