# clearops-gitops

The **config repo** for ClearOps. What is in this repo is what runs: ArgoCD watches it and keeps the cluster matching it, and its `git log` is the deploy history.

Application code lives in [clearops-app](https://github.com/vivianokose/clearops-app). Jenkins builds images there, then commits only a new image tag here. It never touches the cluster.

Built in Module 12.5 (GitOps with ArgoCD) of [nexaops-operations-lab](https://github.com/vivianokose/nexaops-operations-lab/tree/main/12.5-gitops-argocd), which has the full write-up, architecture diagram and screenshots.

> **Status:** the demo cluster has been torn down. The two SealedSecrets here were encrypted for that cluster's key and can no longer be decrypted. On a rebuild, reseal them (see below).

## Layout

```
apps/clearops/
├── base/                     frontend, API (+HPA), Redis StatefulSet, NetworkPolicy,
│                             ConfigMap, PostSync smoke test, with sync waves
└── overlays/
    ├── dev/                  namespace dev, HPA 1-3, debug logs; Jenkins bumps tags here
    └── staging/              namespace staging, HPA 2-4; changed only by pull request
argocd/
├── root.yaml                 App of Apps root (applied once, by hand)
├── apps/clearops-appset.yaml ApplicationSet: one Application per overlay folder
├── projects/                 AppProjects: clearops (workloads), platform (the root)
└── config/                   notifications: GitHub commit status on every deploy
```

## How changes flow

| Change | Who | How |
|---|---|---|
| New release to dev | Jenkins | `chore(dev): bump clearops to X.Y.Z` |
| Promote to staging | A person | Pull request changing the staging overlay's tags |
| Roll back | A person | `git revert` the bump commit (see the [runbook](https://github.com/vivianokose/nexaops-operations-lab/blob/main/12.5-gitops-argocd/runbooks/gitops-rollback.md)) |
| New environment | A person | Add an overlay folder, and allow its namespace in `argocd/projects/clearops.yaml` |

A green tick on a commit means ArgoCD deployed it and it is healthy (`argocd/clearops-<env>`).

## Bootstrap a new cluster

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.5.6/manifests/install.yaml
# install the Sealed Secrets controller, then reseal redis-auth for dev and staging

kubectl apply -f argocd/projects/clearops.yaml -f argocd/projects/platform.yaml
kubectl patch configmap argocd-notifications-cm -n argocd --patch-file argocd/config/argocd-notifications-cm-patch.yaml
kubectl apply -f argocd/root.yaml
```

Everything else is created from this repo.

## Rules

- No plain Secrets, ever. Only SealedSecrets.
- No hand edits in the cluster. Self-heal undoes them in seconds.
- Image tags are immutable. The pipeline never reuses a published tag.
