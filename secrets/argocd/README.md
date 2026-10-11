# Argo CD namespace SealedSecrets

SealedSecrets that must live in the **`argocd`** namespace. Kept separate from
`secrets/sealedsecrets/` (which targets `akk-api`) because a SealedSecret is
encrypted for one specific namespace and cannot be moved between them.

## Status: created

Both secrets below are generated, committed, and syncing. Image Updater has
been confirmed reading GHCR and committing tag bumps back to this repo. If you
are setting up a fresh cluster, regenerate them (see
`infra/image-updater/README.md`); otherwise there is nothing to do here.

## What belongs here

- `ghcr-creds-sealed.yaml` — GHCR read token (`read:packages`) so Argo CD Image
  Updater can scan `ghcr.io/akkercheval/akk-frontend` for new tags.
- `git-creds-sealed.yaml` — GitHub token with write access to
  `akkercheval/akk-gitops` so Image Updater can commit the tag bump back.

Both are generated with `kubeseal` and sealed for namespace `argocd`. See
`infra/image-updater/README.md` for the exact commands. **Never** commit a
plaintext `Secret`.

The `image-updater-secrets` Argo Application
(`infra/argocd/applications/image-updater-secrets.yaml`) syncs this folder.

## Troubleshooting

Symptoms and fixes for the two failure modes we have actually hit. The place to
look is always the Image Updater log:

```bash
kubectl logs -n argocd deploy/argocd-image-updater --tail=50 -f
```

### "A build is green in GitHub Actions but Argo CD never redeploys"

This is the normal shape of every failure here: Actions only builds and pushes
the image. The chain that actually deploys it is

```
GHCR has new sha- tag  ->  Image Updater scans (needs ghcr-creds)
  ->  commits new tag to apps/<app>/kustomization.yaml (needs git-creds)
  ->  Argo CD syncs that Git commit  ->  pod rolls to the new image
```

If Argo CD shows the app Healthy/Synced but on an old tag, the break is upstream
of Argo CD, in Image Updater. The log tells you which link failed.

### `could not fetch secret "ghcr-creds" from namespace "argocd" ... not found`

Image Updater cannot authenticate to GHCR, so it never sees new builds
(`images_updated=0 errors=2`). Causes, in order of likelihood:

- The secret does not exist in the cluster. Check:
  `kubectl get secret ghcr-creds git-creds -n argocd`
- **A PAT expired.** Both tokens are long-lived GitHub PATs. When one lapses the
  same "not found"/auth error returns. Fix: regenerate the PAT on GitHub, then
  re-create + re-seal + commit the affected `*-sealed.yaml` (full commands in
  `infra/image-updater/README.md`). A SealedSecret is bound to its namespace, so
  always re-seal for `argocd`.
- The `image-updater-secrets` Argo app is syncing an **empty folder.** If this
  directory has only `README.md`, there is nothing to apply and the app will
  still report Synced/Healthy — a green status here does not mean the secrets
  exist. Confirm with `git ls-files secrets/argocd`.

### `could not find kustomization in .../apps/<app>/apps/<app>` (doubled path)

Credentials are fine (the log shows `Setting new image to ...`), but the
write-back fails because the path is doubled. The
`write-back-target` annotation is resolved **relative to the Application's
`spec.source.path`**, not from the repo root. Since the kustomization sits at
the source path, the target must be `kustomization:.` — not
`kustomization:apps/<app>`, which doubles to `apps/<app>/apps/<app>`. Fixed on
both apps in `infra/argocd/applications/{akk-frontend,ws-akk-api}.yaml`.

### Verifying a healthy run

A good scan logs `Setting new image to ghcr.io/.../<app>:sha-xxxxxxx`, then
`Committing ... to branch main`, and ends with
`Processing results: ... images_updated=1 errors=0`. Confirm the write-back
landed with `git log --oneline -- apps/akk-frontend/kustomization.yaml` — a
commit authored by the Image Updater is the proof the whole loop works.
