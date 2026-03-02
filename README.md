# home-k8s-gitops-flux

GitOps repository for managing Kubernetes clusters using [FluxCD](https://fluxcd.io/). Supports multiple clusters and multiple environments (dev, prod) with a clean separation between platform infrastructure and in-house application workloads.

---

## Repository Structure

```
.
├── clusters/                        # Per-cluster Flux entry points
│   ├── dev/
│   │   └── home-cluster/
│   │       ├── flux-instance.yaml  # Applied once manually to bootstrap Flux via the Flux Operator
│   │       ├── infrastructure.yaml  # Flux Kustomization → infrastructure/overlays/dev
│   │       └── apps.yaml           # Flux Kustomization → apps/overlays/dev
│   └── prod/
│       └── cloud-cluster/
│           ├── flux-instance.yaml  # Applied once manually to bootstrap Flux via the Flux Operator
│           ├── infrastructure.yaml  # Flux Kustomization → infrastructure/overlays/prod
│           └── apps.yaml           # Flux Kustomization → apps/overlays/prod
│
├── infrastructure/                  # Platform/ops tooling (cert-manager, ingress, MetalLB, NFS storage, etc.)
│   ├── base/
│   │   ├── sources/                 # HelmRepository and OCIRepository definitions
│   │   ├── cert-manager/
│   │   ├── traefik/
│   │   ├── metallb/
│   │   └── nfs-csi/                 # csi-driver-nfs — NFS-backed PersistentVolumes
│   └── overlays/
│       ├── dev/                     # Dev-specific values and patches
│       │   └── patches/
│       └── prod/                    # Prod-specific values and patches
│           └── patches/
│
└── apps/                            # In-house application workloads
    ├── base/
    │   └── example-app/             # Base manifests (deployment, service, ingress)
    └── overlays/
        ├── dev/                     # Dev overrides: namespace, replicas, image tags
        │   └── example-app/
        │       └── patches/
        └── prod/                    # Prod overrides: namespace, replicas, HPA, resource limits
            └── example-app/
                └── patches/
```

---

## How It Works

### Cluster Entry Points (`clusters/`)

Each cluster has its own directory under `clusters/<env>/<cluster-name>/` containing three files:

- **`flux-instance.yaml`** — a `FluxInstance` CR for the [Flux Operator](https://github.com/controlplaneio-fluxcd/flux-operator). This is the only file applied manually — it installs the Flux controllers and tells Flux to sync from this directory. Everything else is driven by Flux after this is applied.
- **`infrastructure.yaml`** — points Flux at the appropriate infrastructure overlay for that environment. Runs first with `wait: true` so all platform components are ready before apps deploy.
- **`apps.yaml`** — points Flux at the appropriate apps overlay. Has `dependsOn: infrastructure` to enforce ordering.

There is no `flux-system/` directory committed to the repo. The Flux Operator manages the controllers directly — nothing is generated into the repository.

Adding a new cluster means creating a new directory here with a `flux-instance.yaml` and choosing which environment overlays it targets.

### Infrastructure (`infrastructure/`)

Platform tooling that the cluster itself needs to function: ingress controllers, certificate management, load balancer IP allocation, persistent storage, monitoring, etc. These are typically Helm charts installed via FluxCD `HelmRelease` resources.

- **`base/`** — environment-agnostic definitions. `HelmRelease` objects with sensible defaults, `HelmRepository` sources, and namespace declarations.
- **`overlays/<env>/`** — environment-specific patches applied on top of base. Examples: different MetalLB IP pools, cert-manager replica counts, Traefik log levels.

Infrastructure components are installed **once per cluster**. If a cluster hosts both dev and prod workloads, it gets a single infrastructure install (typically prod-grade config).

### Apps (`apps/`)

In-house developed workloads. Kept strictly separate from infrastructure so teams can manage their own services without touching platform config.

- **`base/<app-name>/`** — the canonical manifest set for an app: Deployment, Service, Ingress, etc. Environment-neutral.
- **`overlays/<env>/<app-name>/`** — environment-specific overrides applied via Kustomize patches. Dev and prod apps are deployed into distinct namespaces (e.g., `my-app-dev` vs `my-app-prod`) so both can coexist on a shared cluster.

---

## Environment Separation

| Concern | How it's handled |
|---|---|
| Infrastructure config | Kustomize patches in `infrastructure/overlays/<env>/patches/` |
| App namespace | Kustomize `namespace:` field in `apps/overlays/<env>/<app>/kustomization.yaml` |
| App image tags | Patch in `apps/overlays/<env>/<app>/patches/` |
| App replicas / HPA | Patch or additional resource in `apps/overlays/<env>/<app>/` |
| Ingress hostnames | Patch in `apps/overlays/<env>/<app>/patches/` |

---

## Bootstrapping a Cluster

The Flux Operator must be installed on the cluster before applying the `FluxInstance`. This is a one-time manual step per cluster — everything after is GitOps-driven.

```bash
# 1. Install the Flux Operator
helm install flux-operator oci://ghcr.io/controlplaneio-fluxcd/charts/flux-operator \
  --namespace flux-system --create-namespace

# 2. Create the Git credentials secret
kubectl create secret generic flux-system \
  --namespace flux-system \
  --from-literal=username=<user> \
  --from-literal=password=<token>

# 3. Apply the FluxInstance — this bootstraps Flux and starts syncing the repo
kubectl apply -f clusters/<env>/<cluster-name>/flux-instance.yaml
```

After step 3, Flux installs its controllers and begins reconciling `infrastructure.yaml` and `apps.yaml` from the same cluster directory.

## Adding a New Cluster

1. Create `clusters/<env>/<cluster-name>/`
2. Add `flux-instance.yaml` with `sync.path` pointing at the new cluster directory
3. Add `infrastructure.yaml` pointing at `./infrastructure/overlays/<env>`
4. Add `apps.yaml` pointing at `./apps/overlays/<env>` with `dependsOn: infrastructure`
5. Follow the bootstrap steps above on the new cluster

If the cluster needs environment config that differs from an existing overlay, create a new overlay under `infrastructure/overlays/<new-env>/`.

---

## Adding a New Infrastructure Component

1. Add a `HelmRepository` source to `infrastructure/base/sources/` if needed
2. Create `infrastructure/base/<component>/` with `namespace.yaml`, `helmrelease.yaml`, and `kustomization.yaml`
3. Add the component to `infrastructure/overlays/dev/kustomization.yaml` and `prod/kustomization.yaml`
4. Add any environment-specific value patches to `infrastructure/overlays/<env>/patches/`

---

## Adding a New App

1. Create `apps/base/<app-name>/` with Deployment, Service, Ingress, and `kustomization.yaml`
2. Create `apps/overlays/dev/<app-name>/kustomization.yaml` — set `namespace: <app-name>-dev`, reference base, add dev patches
3. Create `apps/overlays/prod/<app-name>/kustomization.yaml` — set `namespace: <app-name>-prod`, reference base, add prod patches
4. Add the app directory to `apps/overlays/dev/kustomization.yaml` and `prod/kustomization.yaml`

---

## Secrets Management

Secrets are managed with [SOPS](https://github.com/getsops/sops) + age keys. Encrypted secret files are committed to the repository. Flux decrypts them at apply time using a key stored as a Kubernetes secret in the `flux-system` namespace.

See [FluxCD SOPS docs](https://fluxcd.io/flux/guides/mozilla-sops/) for setup instructions.

---

## Dependency Chain

```
kubectl apply flux-instance.yaml
     └── Flux Operator installs controllers
               └── infrastructure.yaml    (Flux Kustomization CR, wait: true)
                         └── apps.yaml   (Flux Kustomization CR, dependsOn: infrastructure)
```
