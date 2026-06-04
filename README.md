# Qdrant GitOps — Kustomize + ArgoCD

Self-host Qdrant via the official Helm chart, wrapped by Kustomize overlays,
deployed by ArgoCD. Mirrors a base/overlays GitOps layout.

## Layout

```
base/                    # provides the namespace only
  kustomization.yaml
  namespace.yaml
overlays/
  local/                 # OrbStack: 1 replica, tiny resources, no API key
    kustomization.yaml   # references ../../base + inflates chart
    values-local.yaml    # self-contained values (base + local merged)
  prod/                  # HA: 3 replicas, clustered, API key from secret
    kustomization.yaml
    values-prod.yaml
argocd/
  application-local.yaml # ArgoCD Application pointing at overlays/local
```

Note: each overlay carries a complete `values-*.yaml` and references the base
as a *directory* (`../../base`). kustomize's load-restrictor blocks an overlay
from pointing at a *file* inside another directory (e.g. `../../base/x.yaml`),
which is why values aren't split into a shared base file.

## Requirements

- **kustomize v5.0+** — `additionalValuesFiles` in `helmCharts` needs v5.
- **helm** on PATH — kustomize shells out to it for `--enable-helm`.

## Try it locally on OrbStack

### Option 1: render + apply directly (fastest sanity check)

```bash
# Render the local overlay (no ArgoCD involved)
kustomize build --enable-helm overlays/local | kubectl apply -f -

kubectl -n qdrant rollout status statefulset/qdrant
kubectl -n qdrant port-forward svc/qdrant 6333:6333 6334:6334
curl http://localhost:6333/collections
```

### Option 2: through ArgoCD (mirrors the real flow)

```bash
# 1. Install ArgoCD on the OrbStack cluster
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 2. Allow Helm inflation in kustomize builds
kubectl -n argocd patch configmap argocd-cm --type merge \
  -p '{"data":{"kustomize.buildOptions":"--enable-helm"}}'
kubectl -n argocd rollout restart deploy argocd-repo-server

# 3. Push this directory to your Git repo, then update repoURL in
#    argocd/application-local.yaml to match.

# 4. Create the Application
kubectl apply -f argocd/application-local.yaml

# 5. Watch it sync
kubectl -n argocd get application qdrant-local -w
```

Get the ArgoCD admin password and UI:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
kubectl -n argocd port-forward svc/argocd-server 8080:443
# https://localhost:8080  (user: admin)
```

## Notes

- **Local overlay** omits `storageClassName` so OrbStack's default provisioner
  handles the PVC, and runs without an API key for quick poking.
- **Prod overlay** expects a secret `qdrant-api-key` (key `api-key`) to exist —
  wire it via External Secrets / 1Password rather than committing it.
- ArgoCD needs `--enable-helm` (step 2) or the build fails with a
  "helm chart inflation requires --enable-helm" error.
- The `ServerSideApply=true` syncOption avoids last-applied-config annotation
  bloat on the larger rendered manifests.
- **The `StatefulSet` needs `ServerSideDiff=true`** (set as the
  `argocd.argoproj.io/compare-options` annotation on the Application). The
  Helm-rendered manifest omits every field Kubernetes defaults onto the live
  object — `revisionHistoryLimit`, `dnsPolicy`, `volumeMode`, `storageClassName`
  (null on the local overlay), `persistentVolumeClaimRetentionPolicy`, and the
  pod-template defaults. ArgoCD's client-side diff does not cancel these, so the
  StatefulSet sits permanently `OutOfSync` (Healthy, but never green) and
  `selfHeal` re-syncs it on every cycle. `ServerSideDiff` runs the desired
  manifest through an API-server dry-run so the same defaults apply to both
  sides and cancel. Symptom if you ever drop the annotation: `kubectl diff`
  shows no drift but the ArgoCD UI does.
- **The Application manifest is not self-managed.** `argocd/application-local.yaml`
  lives in this repo for reference, but ArgoCD does not reconcile its own
  `Application` object — you apply it with `kubectl apply -f`. So edits to that
  file (the annotations above, `syncPolicy`, `repoURL`, …) only take effect
  after a manual `kubectl apply -f argocd/application-local.yaml`. ArgoCD
  auto-reconciles only what lives under the Application's `path` (`overlays/local`).
  To make the Application itself GitOps-managed, adopt an app-of-apps pattern or
  an `ApplicationSet`.

## Cleanup

```bash
# Option 1
kustomize build --enable-helm overlays/local | kubectl delete -f -
# Option 2
kubectl delete -f argocd/application-local.yaml   # prune removes Qdrant too
```
