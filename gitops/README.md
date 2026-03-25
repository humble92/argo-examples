# GitOps layout (Argo CD) - App of Apps, single-cluster

## App of Apps pattern

A single **root Application** (`root-app.yaml`) points at `gitops/argocd/`. Argo CD syncs that directory and automatically creates all **AppProjects** and child **Applications** defined inside it. After the initial bootstrap, every change is driven by **Git push only**.

```
                       kubectl apply (one-time)
                              |
                              v
                     +----------------------+
                     |  root (Application)  |
                     |  path: gitops/argocd |
                     +--------+-------------+
                              |  syncs kustomization.yaml
                              v
            +-----------------+-----------------+
            |                                   |
     projects/                         infrastructure/           applications/
   infrastructure.yaml                 envoy-gateway-crds  -50   n8n-prod      10
   apps.yaml                           cert-manager        -40
                                       envoy-gateway       -30
                                       envoy-edge          -20
```

## Directory structure

```
gitops/
  root-app.yaml                        # Root Application (bootstrap entry point)
  README.md

  argocd/                              # Argo CD resource definitions ONLY
    kustomization.yaml                 # Aggregates projects + all Application CRs
    projects/
      infrastructure.yaml              # AppProject "infrastructure"
      apps.yaml                        # AppProject "apps"
    infrastructure/                    # Application CRs for infra (source + destination)
      envoy-gateway-crds.yaml          # wave -50
      cert-manager.yaml                # wave -40 (multi-source)
      envoy-gateway.yaml               # wave -30
      envoy-edge.yaml                  # wave -20
    applications/                      # Application CRs for workloads
      n8n-prod.yaml                    # wave 10

  infrastructure/                      # Infra manifests referenced by Application paths
    cert-manager/                      # ClusterIssuers (Kustomize)
      kustomization.yaml
      clusterissuers.yaml
    envoy-edge/                        # GatewayClass + Gateway (Kustomize)
      kustomization.yaml
      gatewayclass.yaml
      gateway.yaml

  applications/                        # Workload manifests referenced by Application paths
    n8n-prod/                          # n8n production (Kustomize)
      kustomization.yaml
      namespace.yaml
      serviceaccount.yaml
      pvc.yaml
      deployment.yaml
      service.yaml
      httproute.yaml
```

## Bootstrap (one-time)

Only one manual command is needed:

```bash
kubectl apply -n argocd -f gitops/root-app.yaml
```

The root Application syncs `gitops/argocd/`, which creates all AppProjects and child Applications. Child Applications then sync their respective Helm charts and manifests in wave order.

After this, **adding or modifying infrastructure/apps requires only a Git push**. The root Application detects changes and reconciles automatically.

## AppProjects

| Project              | Purpose                                                                                                                                     |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| **`infrastructure`** | Cluster-wide infra: CRDs, cert-manager, Envoy Gateway controller, edge Gateway, ClusterIssuers. Allowed to create cluster-scoped resources. |
| **`apps`**           | Workloads in allowed namespaces. `Namespace` is whitelisted so Git can include `namespace.yaml`; `destinations` still limits namespace names. |

## Sync-wave strategy

All infrastructure Applications use **negative** waves so they sync before any application workload (wave >= 0). Waves are spaced by **10** to leave room for future additions.

| Wave | Application          | What it does                                                                                                                                         |
| ---- | -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| -50  | `envoy-gateway-crds` | Gateway API CRDs (standard channel) + Envoy Gateway CRDs                                                                                             |
| -40  | `cert-manager`       | Multi-source: Jetstack Helm (default internal wave **0**) + Git `infrastructure/cert-manager/` for ClusterIssuers (**wave 10**) in the **same** sync |
| -30  | `envoy-gateway`      | Envoy Gateway controller (skips CRDs, relies on wave -50)                                                                                            |
| -20  | `envoy-edge`         | `GatewayClass` `eg` + `Gateway` `eg` in `default` namespace                                                                                          |
| 10   | `n8n-prod`           | n8n in namespace `n8n-prod` (HTTPRoute to Gateway `eg`)                                                                                              |
| 10+  | _(other apps)_       | Add more Application CRs with wave `10` or higher                                                                                                    |

### `cert-manager` Application (multi-source)

`cert-manager.yaml` uses `spec.sources`: Helm chart + Git directory `gitops/infrastructure/cert-manager`. Argo CD merges rendered manifests and applies by `argocd.argoproj.io/sync-wave`. Chart objects stay at default wave **0**; `ClusterIssuer` resources use wave **10**, so they apply **after** CRDs, webhook, and controller from the chart in a single Application sync.

## Pre-push checklist

1. Set ACME email in `gitops/infrastructure/cert-manager/clusterissuers.yaml`.
2. Pin Helm chart versions in `targetRevision`:
   - [envoyproxy/gateway-helm tags](https://hub.docker.com/r/envoyproxy/gateway-helm/tags)
   - [envoyproxy/gateway-crds-helm tags](https://hub.docker.com/r/envoyproxy/gateway-crds-helm/tags)
   - [cert-manager releases](https://github.com/cert-manager/cert-manager/releases)
3. Confirm `targetRevision` (branch) in `root-app.yaml` and Git-sourced Applications matches your default branch (`master` or `main`).

## Helm install vs edge Gateway vs upstream quickstart

- **`envoy-gateway-crds` + `envoy-gateway`** (Helm) install the **controller** (Deployments, RBAC, webhooks).
- The controller alone does **not** create listeners. **`envoy-edge`** applies `GatewayClass` + `Gateway` from `gitops/infrastructure/envoy-edge/`.
- If you prefer the upstream quickstart (includes a demo HTTPRoute + backend), use that instead and remove `envoy-edge`. Do **not** apply both with the same names and different specs.

## Gateway API in short

- **Ingress** uses `Ingress` + an ingress controller (e.g. ingress-nginx).
- **Gateway API** uses `GatewayClass`, `Gateway`, `HTTPRoute`, `GRPCRoute`. It is the Kubernetes SIG Network standard direction.
- Gateway API is only a specification. A concrete **implementation** must be deployed. **This repo uses Envoy Gateway as that implementation**, so no separate install is needed.

References:

- [Gateway API](https://gateway-api.sigs.k8s.io/)
- [Envoy Gateway install (Helm)](https://gateway.envoyproxy.io/latest/install/install-helm/)

## OCI Helm and Argo CD

Envoy Gateway charts are pulled from Docker Hub OCI (`docker.io/envoyproxy`). If Argo CD gets a 403 Forbidden, create an individual repository Secret (`argocd.argoproj.io/secret-type: repository`) with `url: docker.io/envoyproxy`, `enableOCI: true`, and Docker Hub credentials. Do **not** use `oci://` prefix in Application `repoURL` or AppProject `sourceRepos` per upstream issue [#25513](https://github.com/argoproj/argo-cd/issues/25513). Credential templates (`repo-creds`) do not work reliably with OCI registries.

## Apps project

Register your application Git URLs under `spec.sourceRepos` in `projects/apps.yaml`. Add Application YAML files under `argocd/applications/` and reference them in `argocd/kustomization.yaml`. Put workload manifests in `gitops/applications/<app>/` and point the Application `path` there.

## n8n production (`n8n-prod`)

Dedicated namespace `n8n-prod`. Pinned image tag. PVC for `/home/node/.n8n`. Probes (`/healthz`, `/healthz/readiness`). Recreate strategy (single replica + RWO volume). No `--tunnel` (use Gateway API + Service). Restricted container security context. HTTPRoute to Envoy Gateway `eg` in `default` with hostname `n8n.local`.

### Required Secret (not in Git)

Create this **before** the Deployment can run (Argo CD will sync other objects; the Pod stays in `CreateContainerConfigError` until the Secret exists):

```bash
kubectl create namespace n8n-prod --dry-run=client -o yaml | kubectl apply -f -
kubectl -n n8n-prod create secret generic n8n-prod-secrets \
  --from-literal=N8N_ENCRYPTION_KEY="$(openssl rand -hex 16)"
```

Keep this key stable across restarts and backups; loss of the key means stored credentials cannot be decrypted.

Argo CD treats that Secret as **orphaned** because it is not in the Git manifest set. The **`apps` AppProject** lists `Secret/n8n-prod-secrets` under `spec.orphanedResources.ignore` so the warning is suppressed. For long-term GitOps, replace this with Sealed Secrets, External Secrets, or SOPS.

### Access (local)

1. Expose the Envoy dataplane Service (name like `envoy-default-eg-<hash>` in `envoy-gateway-system`):
   - **`minikube tunnel`** (leave the terminal open). On some Windows setups, LoadBalancer **port 80** needs an elevated shell; see [minikube: ports \< 1024 on Windows](https://minikube.sigs.k8s.io/docs/handbook/accessing/#access-to-ports-1024-on-windows-requires-root-permission).
   - Or **`kubectl port-forward`** (no admin): `kubectl port-forward -n envoy-gateway-system svc/envoy-default-eg-<hash> 8080:80` then open `http://n8n.local:8080/`.
2. Point **`n8n.local`** at the address you actually hit:
   - After tunnel: `kubectl -n envoy-gateway-system get svc` and set hosts to the **`EXTERNAL-IP`** (not always `127.0.0.1`).
   - After port-forward to localhost: `127.0.0.1 n8n.local` is fine if you use the forwarded port in the URL.
3. Open **http://n8n.local** (HTTP on port 80; **https://** will fail until you add TLS on the Gateway).
4. The Deployment sets **`N8N_SECURE_COOKIE=false`** so the UI works over HTTP with hostname `n8n.local` (n8n 2.x otherwise requires HTTPS or `localhost`). For real HTTPS, remove that env and align `N8N_PROTOCOL` / `WEBHOOK_URL` with your public URL.

### Hardening for real production

- Use **PostgreSQL** (or another external DB) instead of the default SQLite on PVC for availability and backup story; set the [database env vars](https://docs.n8n.io/hosting/configuration/environment-variables/database/) accordingly.
- Terminate **HTTPS** at the Gateway (cert-manager + `HTTPRoute` TLS) and set `N8N_PROTOCOL`, `WEBHOOK_URL`, and editor base URL to match public URLs. Remove **`N8N_SECURE_COOKIE=false`** from the Deployment when using HTTPS.
- Review resource **requests/limits** and **PVC** size for your workload.
- Bump the image tag in `gitops/applications/n8n-prod/deployment.yaml` deliberately on upgrades (avoid floating `:latest`).

## Minikube / local clusters

- Use `minikube tunnel` to expose LoadBalancer services (Envoy Gateway creates a Service such as `envoy-default-eg-*`), or
- `kubectl port-forward -n envoy-gateway-system svc/<envoy-service> <local-port>:80` if tunnel or port 80 on Windows is awkward.

## Orphan warnings for runtime Secrets

cert-manager and Envoy Gateway create TLS and runtime **Secrets** that are not in the Helm/Git manifest set. Argo CD may list them as orphaned.

The **`infrastructure` AppProject** sets `spec.orphanedResources.ignore` for those Secret names (by `group` / `kind` / `name`). The **`apps` AppProject** ignores the bootstrap Secret **`n8n-prod-secrets`** (created with `kubectl`, not Git). Per-Application `spec.orphanedResources` is not used here because the installed **Application** CRD OpenAPI schema in common Argo CD installs does not include that field, so it would be dropped by the API and keep the **root** Application permanently **OutOfSync** against Git.

## WSL: `argocd` CLI and `connection refused`

The CLI talks to **Argo CD API** (usually via `kubectl port-forward` to `argocd-server` on the Windows host where Minikube runs).

1. On **Windows** (PowerShell), start port-forward and bind all interfaces so WSL can reach it:

   ```powershell
   kubectl port-forward -n argocd svc/argocd-server 8080:443 --address 0.0.0.0
   ```

2. In **WSL**, use the Windows host IP toward the default route, not always `/etc/resolv.conf` `nameserver` (that can be a stub and refuse TCP):

   ```bash
   WIN_HOST=$(ip route | awk '/^default/ {print $3; exit}')
   argocd login ${WIN_HOST}:8080 --grpc-web --insecure
   ```

3. There is **no** `argocd status` subcommand. Use for example `argocd app list`, `argocd app get root`, `argocd app resources n8n-prod`.

Alternatively run `argocd` from **Windows** (same machine as port-forward to `127.0.0.1:8080`) to avoid cross-OS networking.

## cert-manager: `ClusterIssuer` Missing / OutOfSync

If desired `ClusterIssuer` objects show **Missing** in the UI:

1. Confirm Helm source sync finished (controller + **webhook** Deployments healthy). The validating webhook must admit `ClusterIssuer` creates.
2. Run `kubectl get clusterissuer` and check `kubectl -n argocd describe application cert-manager` for sync errors.
3. After fixing Helm `values` (for example schema errors), use **Hard Refresh** on the Application so repo-server re-runs `helm template`.

## Troubleshooting: `argocd-repo-server` not ready (init `copyutil`)

If `kubectl -n argocd get pods` shows `argocd-repo-server` as `0/1` and init container `copyutil` is in `CrashLoopBackOff`, check:

```bash
kubectl -n argocd logs deploy/argocd-repo-server -c copyutil --tail=20
```

If you see **`/bin/ln: Already exists`**, the init script already created `/var/run/argocd/argocd-cmp-server` on a previous attempt. The `emptyDir` volume is kept across init retries, so later retries fail on `ln -s`.

**Fix:** delete the pod so Kubernetes creates a new pod with a fresh `emptyDir`:

```bash
kubectl -n argocd delete pod -l app.kubernetes.io/name=argocd-repo-server
```

After this, the Deployment should recreate the pod and it should reach `1/1 Running`. A broken repo-server also blocks manifest generation, so the **root** Application may show sync errors and no child apps until this is fixed.
