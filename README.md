# heli-naik-expense-config

GitOps repo for `expense-api`. Argo CD clones **this** repo; the application
repo [`heli-naik-expense-tracking`](https://github.com/AI-Native-2026-08-05-Intuit/heli-naik-expense-tracking)
holds the Java source, Dockerfile, and CI. Nothing here is applied by hand
except the `argocd/` and `argocd-system/` bootstrap objects.

## Layout

| Path | Owner | What it is |
| --- | --- | --- |
| `base/` | app team | The W5 D3 Kubernetes manifests, identical across envs |
| `overlays/dev`, `overlays/staging`, `overlays/prod` | app team | Kustomize overlays: namespace, replicas, log level, image tag |
| `argocd/projects/expense.yaml` | platform team (app team proposes via PR) | `AppProject expense` — repo allow-list, destinations, RBAC, sync windows |
| `argocd/applications/expense-api-dev.yaml` | app team | Single-env `Application` for `overlays/dev`; the readable anchor. The ApplicationSet owns the live `expense-api-dev` |
| `argocd/applicationsets/expense-api-envs.yaml` | app team | Matrix generator producing `expense-api-{dev,staging,prod}` |
| `argocd-system/notifications-cm.yaml` | platform team | `argocd-notifications-cm` — Slack webhook service, triggers, templates |

`base/kustomization.yaml` deliberately does **not** render `00-namespace.yaml`
or the ServiceMonitor: the AppProject's `clusterResourceWhitelist` is empty and
the k3d cluster has no Prometheus Operator CRDs. Both files stay in `base/` so
they can be re-enabled once the cluster supports them.

## Reconcile loop

1. CI in the application repo builds and pushes `uptimecrew/expense-api:<sha>`.
2. Its `_bump-config` job opens a PR **against this repo** rewriting the image
   tag in `overlays/dev/kustomization.yaml`. That job holds a PAT with
   `contents: write` on this repo only — never cluster credentials.
3. A human merges the bump PR.
4. Argo CD polls this repo (~3 min) and syncs the affected Applications.

Cluster drift is reverted by `selfHeal`. Do not `kubectl edit` a fix into the
cluster; commit it here.

## Bootstrap

The `argocd/` and `argocd-system/` objects are applied once, in order:

```bash
kubectl apply -f argocd/projects/expense.yaml          # AppProject first -- Applications reference it
kubectl apply -f argocd-system/notifications-cm.yaml   # Slack wiring
kubectl apply -f argocd/applicationsets/expense-api-envs.yaml
```

`argocd-notifications-secret` (keys `slack-webhook-url`, optional `slack-token`)
is created out-of-band by `scripts/slack-webhook-secret.sh` in the application
repo and is **never** committed here.

## Render locally

```bash
kustomize build overlays/dev
kustomize build overlays/staging
kustomize build overlays/prod
```

## Repo allow-list

`AppProject.spec.sourceRepos` lists exact URLs, never `*`:

- `https://github.com/AI-Native-2026-08-05-Intuit/heli-naik-expense-config.git` — this repo
- `https://github.com/uptimecrew/expense-config.git` — curriculum reference

An Application pointing anywhere else fails sync with
`application repo <url> is not permitted in project 'expense'`.
