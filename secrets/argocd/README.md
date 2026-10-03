# Argo CD namespace SealedSecrets

SealedSecrets that must live in the **`argocd`** namespace. Kept separate from
`secrets/sealedsecrets/` (which targets `akk-api`) because a SealedSecret is
encrypted for one specific namespace and cannot be moved between them.

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
