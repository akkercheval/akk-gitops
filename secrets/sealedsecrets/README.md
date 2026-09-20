# Sealed Secrets

This folder holds **SealedSecret** manifests. They are encrypted with the
cluster's Sealed Secrets public key and are safe to commit to a public repo.
The Sealed Secrets controller running in the cluster is the only thing that can
decrypt them back into real Kubernetes `Secret` objects.

## What lives here

- `akk-app-secrets-sealed.yaml` — all runtime secrets for the `ws-akk-api`
  backend (namespace `akk-api`): the MS SQL connection string, the JWT signing
  key, and the admin password hash. These mirror the sensitive values in the
  service's (git-ignored) `appsettings.json`.

  > Not committed until you generate it (see below). The plaintext Secret must
  > **never** be committed.

## Generate the app-secrets SealedSecret

Run on a machine that has `kubeseal` and `kubectl` access to the cluster (or use
the Kubeseal VS Code extension, as done for bccmusic). The `akk-api` namespace
must exist first (Argo creates it via `CreateNamespace=true`, or run
`kubectl create namespace akk-api`).

Pull the real values from the service's local `appsettings.json`
(`ConnectionStrings:DefaultConnection`, `Jwt:Key`, `AdminUser:PasswordHash`).

```bash
# 1. Build a plaintext Secret WITHOUT applying it to the cluster.
kubectl create secret generic akk-app-secrets \
  --namespace=akk-api \
  --from-literal=connectionString="Server=<MSSQL-LXC-IP>,1433;Database=akk_kercheval;User Id=wsakkapi;Password=CHANGE_ME;Encrypt=True;TrustServerCertificate=True;" \
  --from-literal=jwtKey="<JWT-KEY-FROM-APPSETTINGS>" \
  --from-literal=adminPasswordHash='<BCRYPT-HASH-FROM-APPSETTINGS>' \
  --dry-run=client -o yaml > /tmp/akk-app-secrets.yaml

# 2. Seal it against the running controller.
kubeseal \
  --controller-namespace sealed-secrets \
  --controller-name sealed-secrets \
  --format yaml \
  < /tmp/akk-app-secrets.yaml \
  > secrets/sealedsecrets/akk-app-secrets-sealed.yaml

# 3. Delete the plaintext file immediately.
rm /tmp/akk-app-secrets.yaml

# 4. Commit ONLY the sealed file.
git add secrets/sealedsecrets/akk-app-secrets-sealed.yaml
git commit -m "Add sealed app secrets for akk-api"
git push
```

Argo CD applies the SealedSecret, the controller decrypts it into a `Secret`
named `akk-app-secrets`, and the `ws-akk-api` Deployment reads
`connectionString`, `jwtKey`, and `adminPasswordHash` from it (mapped to
`ConnectionStrings__DefaultConnection`, `Jwt__Key`, `AdminUser__PasswordHash`).

> The BCrypt hash starts with `$2a$`, which shells may treat specially — wrap it
> in single quotes as shown so it is stored verbatim.

## Rotating the credentials later

See the root `README.md` and the reusable runbook in
`My Web Sites/Onboarding a New Website to the CI-CD Pipeline.md` for the full
clean-rotation procedure.
