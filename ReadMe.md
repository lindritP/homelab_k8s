# homelab-gitops

Argo CD GitOps repo for the homelab AKS cluster, using the **app-of-apps** pattern.
Argo CD itself is installed by the Terraform stack (`../terraform`); this repo holds
everything it deploys onto the cluster.

## Layout

```
k8s/
├── root-app.yaml          # app-of-apps root; recurse:true, prune + selfHeal
└── infrastructure/        # sync-wave -1 (deploys first)
    ├── ingress-nginx.yaml # Helm; Service type LoadBalancer -> Azure LB
    └── cert-manager.yaml  # Helm (jetstack); crds.enabled: true
```

Every file under `infrastructure/` is an Argo CD `Application`. The root (`path: .`,
`recurse: true`) adopts them all and re-applies itself, so the root manages itself.
`prune: true` is set everywhere — **deleting a file from this repo tears down the
workload**.

Application workloads will live under an `apps/` directory (sync-wave `0`, so they land
after the platform pieces). None are defined yet.

## Bootstrap (once)

```sh
kubectl apply -f root-app.yaml
```

After that, adding or removing a workload is just a commit:

- **Add** a workload → add an `Application` file under `infrastructure/` (or a new `apps/`).
- **Remove** a workload → delete its file (prune tears it down).

## Conventions

- `infrastructure/` = sync-wave `-1` (ingress, cert-manager, …) so platform pieces land
  before app workloads (`apps/`, sync-wave `0`).
- Helm-backed apps create their own namespace via `CreateNamespace=true`.
- Apps that source from this repo use `repoURL: https://github.com/lindritP/homelab_k8s.git`.

## Before first sync — verify

- **Chart versions** (`targetRevision`) are pinned but should be checked/bumped:
  `helm repo add … && helm search repo <chart> --versions`.
