# GitOps layout (Argo CD) - App of Apps, single-cluster

## App of Apps pattern

A single **root Application** (`root-app.yaml`) points at `gitops/argocd/`. Argo CD syncs that directory and automatically creates all **AppProjects** and child **Applications** defined inside it. After the initial bootstrap, every change is driven by **Git push only**.

```
                       kubectl apply (one-time)
                              |
                              v
                     +----------------+
                     |  root (Application)  |
                     |  path: gitops/argocd |
                     +--------+-------+
                              |  syncs kustomization.yaml
                              v
            +-----------------+-----------------+
            |                                   |
     projects/                         infrastructure/
   infrastructure.yaml                 envoy-gateway-crds.yaml  (wave -50)
   apps.yaml                           cert-manager.yaml        (wave -40)
                                       envoy-gateway.yaml       (wave -30)
                                       envoy-edge.yaml          (wave -20)
                                       cert-manager-issuers.yaml(wave -10)
                                       envoy-edge/
                                         gatewayclass.yaml
                                         gateway.yaml
                                       cert-manager/
                                         clusterissuers.yaml
```

## Directory structure

```
gitops/
  root-app.yaml                        # Root Application (bootstrap entry point)
  README.md
  argocd/
    kustomization.yaml                 # Aggregates projects + infrastructure
    projects/
      infrastructure.yaml              # AppProject "infrastructure"
      apps.yaml                        # AppProject "apps"
    infrastructure/                    # infrastructure project: Applications + manifests
      envoy-gateway-crds.yaml          # Application  wave -50
      cert-manager.yaml                # Application  wave -40
      envoy-gateway.yaml               # Application  wave -30
      envoy-edge.yaml                  # Application  wave -20
      cert-manager-issuers.yaml        # Application  wave -10
      envoy-edge/                      # GatewayClass + Gateway manifests
        gatewayclass.yaml
        gateway.yaml
        kustomization.yaml
      cert-manager/                    # ClusterIssuer manifests
        clusterissuers.yaml
        kustomization.yaml
    applications/                      # apps project: Applications (added later)
```

## Bootstrap (one-time)

Only one manual command is needed:

```bash
kubectl apply -n argocd -f gitops/root-app.yaml
```

The root Application syncs `gitops/argocd/`, which creates all AppProjects and child Applications. Child Applications then sync their respective Helm charts and manifests in wave order.

After this, **adding or modifying infrastructure/apps requires only a Git push**. The root Application detects changes and reconciles automatically.

## AppProjects

| Project | Purpose |
|---------|---------|
| **`infrastructure`** | Cluster-wide infra: CRDs, cert-manager, Envoy Gateway controller, edge Gateway, ClusterIssuers. Allowed to create cluster-scoped resources. |
| **`apps`** | Application workloads. Restricted to namespaced resources in `prod`, `dev`, `staging`. |

## Sync-wave strategy

All infrastructure Applications use **negative** waves so they sync before any application workload (wave >= 0). Waves are spaced by **10** to leave room for future additions.

| Wave | Application | What it does |
|------|-------------|--------------|
| -50 | `envoy-gateway-crds` | Gateway API CRDs (standard channel) + Envoy Gateway CRDs |
| -40 | `cert-manager` | cert-manager controller + its CRDs with Gateway API support |
| -30 | `envoy-gateway` | Envoy Gateway controller (skips CRDs, relies on wave -50) |
| -20 | `envoy-edge` | `GatewayClass` `eg` + `Gateway` `eg` in `default` namespace |
| -10 | `cert-manager-issuers` | Let's Encrypt `ClusterIssuer` (prod + staging) using Gateway API HTTP-01 |
| 0+ | _(apps project)_ | Application workloads added later |

## Pre-push checklist

1. Set `ACME_EMAIL_PLACEHOLDER` in `gitops/argocd/infrastructure/cert-manager/clusterissuers.yaml`.
2. Pin Helm chart versions in `targetRevision`:
   - [envoyproxy/gateway-helm tags](https://hub.docker.com/r/envoyproxy/gateway-helm/tags)
   - [envoyproxy/gateway-crds-helm tags](https://hub.docker.com/r/envoyproxy/gateway-crds-helm/tags)
   - [cert-manager releases](https://github.com/cert-manager/cert-manager/releases)
3. Confirm `targetRevision` (branch) in `root-app.yaml` and Git-sourced Applications matches your default branch (`master` or `main`).

## Helm install vs edge Gateway vs upstream quickstart

- **`envoy-gateway-crds` + `envoy-gateway`** (Helm) install the **controller** (Deployments, RBAC, webhooks).
- The controller alone does **not** create listeners. **`envoy-edge`** applies `GatewayClass` + `Gateway` from `gitops/argocd/infrastructure/envoy-edge/`.
- If you prefer the upstream quickstart (includes a demo HTTPRoute + backend), use that instead and remove `envoy-edge`. Do **not** apply both with the same names and different specs.

## Gateway API in short

- **Ingress** uses `Ingress` + an ingress controller (e.g. ingress-nginx).
- **Gateway API** uses `GatewayClass`, `Gateway`, `HTTPRoute`, `GRPCRoute`. It is the Kubernetes SIG Network standard direction.
- Gateway API is only a specification. A concrete **implementation** must be deployed. **This repo uses Envoy Gateway as that implementation**, so no separate install is needed.

References:

- [Gateway API](https://gateway-api.sigs.k8s.io/)
- [Envoy Gateway install (Helm)](https://gateway.envoyproxy.io/latest/install/install-helm/)

## OCI Helm and Argo CD

Argo CD must be able to pull `oci://docker.io/envoyproxy/...`. No extra credentials are needed for public charts. If pulls fail, upgrade Argo CD and confirm the `infrastructure` AppProject lists those OCI URLs under `sourceRepos`.

## Apps project

Register your application Git URLs under `spec.sourceRepos` in `projects/apps.yaml`. Add Application YAML files under `applications/` and reference them in `kustomization.yaml`. The root Application will pick them up on next sync.

## Minikube / local clusters

- Use `minikube tunnel` to expose LoadBalancer services, or
- `kubectl port-forward` to the Envoy Service created for your Gateway.
