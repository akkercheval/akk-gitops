# Argo CD Image Updater

Automatically rolls out new container builds **without** the old `:latest`
problem (new image in GHCR, but the running pod never re-pulls it).

## How it works here

1. You push app code to `akkercheval/akk-frontend`. GitHub Actions builds and
   pushes an image tagged `sha-<short>` (and `latest`) to
   `ghcr.io/akkercheval/akk-frontend`.
2. Image Updater (running in the `argocd` namespace) scans GHCR every couple of
   minutes. For the `akk-frontend` Argo Application it considers only tags
   matching `^sha-[0-9a-f]{7,}$` and picks the one with the **newest build
   time** (`newest-build` strategy — a git SHA is not sortable by name).
3. When it finds a newer build, it **commits the new tag back to this repo** by
   updating the `images:` block in `apps/akk-frontend/kustomization.yaml` on
   `main`.
4. Argo CD sees the Git change and syncs it, which recreates the pod on the new,
   immutable tag. Git stays the source of truth, so a rollback is just a
   `git revert` of that commit.

The configuration lives as annotations on
`infra/argocd/applications/akk-frontend.yaml`.

## One-time setup: the two secrets

Image Updater needs two credentials in the `argocd` namespace. Both should be
created as **SealedSecrets** (same pattern as `secrets/sealedsecrets/`) so no
plaintext token is ever committed.

### 1. GHCR pull secret (`ghcr-creds`) — read registry tags

A GitHub Personal Access Token (classic) with **`read:packages`** scope.

```bash
# Build a docker-registry secret WITHOUT applying it.
kubectl create secret docker-registry ghcr-creds \
  --namespace=argocd \
  --docker-server=ghcr.io \
  --docker-username=akkercheval \
  --docker-password='<PAT-with-read:packages>' \
  --dry-run=client -o yaml > /tmp/ghcr-creds.yaml

# Seal it against the cluster controller.
kubeseal \
  --controller-namespace sealed-secrets \
  --controller-name sealed-secrets \
  --format yaml \
  < /tmp/ghcr-creds.yaml \
  > secrets/argocd/ghcr-creds-sealed.yaml

rm /tmp/ghcr-creds.yaml
```

> The annotation `frontend.pull-secret: pullsecret:argocd/ghcr-creds` tells
> Image Updater to read registry credentials from this secret.

### 2. Git write-back token (`git-creds`) — commit the new tag

A GitHub PAT with **`repo`** (or fine-grained **Contents: read/write**) scope on
`akkercheval/akk-gitops`, so Image Updater can push the tag-bump commit.

```bash
kubectl create secret generic git-creds \
  --namespace=argocd \
  --from-literal=username=akkercheval \
  --from-literal=password='<PAT-with-repo-write>' \
  --dry-run=client -o yaml > /tmp/git-creds.yaml

kubeseal \
  --controller-namespace sealed-secrets \
  --controller-name sealed-secrets \
  --format yaml \
  < /tmp/git-creds.yaml \
  > secrets/argocd/git-creds-sealed.yaml

rm /tmp/git-creds.yaml
```

> The annotation `write-back-method: git:secret:argocd/git-creds` references
> this secret. The committer identity defaults to the Image Updater; set
> `argocd-image-updater.argoproj.io/git-committer` annotations later if you want
> a custom author.

Write both sealed files into **`secrets/argocd/`** (not
`secrets/sealedsecrets/`, which targets the `akk-api` namespace). A SealedSecret
is encrypted for a specific namespace, so these must be sealed for and applied
to `argocd`:

```bash
# ...the kubeseal commands above, but write the output here instead:
#   > secrets/argocd/ghcr-creds-sealed.yaml
#   > secrets/argocd/git-creds-sealed.yaml
```

The `image-updater-secrets` Argo Application
(`infra/argocd/applications/image-updater-secrets.yaml`) syncs that folder into
the `argocd` namespace. Commit **only** the two `*-sealed.yaml` files.

## Verifying it works

```bash
# Controller is running:
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-image-updater

# Watch what it decides each scan:
kubectl logs -n argocd deploy/argocd-image-updater -f

# After a new frontend build, confirm the commit landed:
git log --oneline -- apps/akk-frontend/kustomization.yaml
```

A successful run logs something like "Setting new image to
ghcr.io/akkercheval/akk-frontend:sha-xxxxxxx" followed by a commit to `main`.

## Backend (ws-akk-api)

The backend is wired up the same way as the frontend:

- `apps/ws-akk-api/kustomization.yaml` holds the `images:` override (alias
  `api`), pinned to the current `sha-<short>` build. Image Updater bumps it on
  each newer build.
- `infra/argocd/applications/ws-akk-api.yaml` carries the Image Updater
  annotations (image alias `api`, same `newest-build` + `^sha-` filter, write-
  back to `kustomization:apps/ws-akk-api`).
- The `ghcr-creds` and `git-creds` secrets are **shared** with the frontend —
  no new secrets needed.

The `ws-akk-api` CI (in the `AkkWebService` repo) already tags images with
`docker/metadata-action` `type=sha,format=short`, matching the frontend, so the
`^sha-[0-9a-f]{7,}$` filter works without any workflow change.

## Pinning / upgrading the controller

The controller version is pinned in `infra/image-updater/kustomization.yaml`
(remote base at tag `v0.15.2`). Bump that tag to upgrade.
