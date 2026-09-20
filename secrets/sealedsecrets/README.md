# Sealed Secrets

This folder holds **SealedSecret** manifests. They are encrypted with the
cluster's Sealed Secrets public key and are safe to commit to a public repo.
The Sealed Secrets controller running in the cluster is the only thing that can
decrypt them back into real Kubernetes `Secret` objects.

## What lives here

- `akk-db-credentials-sealed.yaml` — the MS SQL connection string for the
  `ws-akk-api` backend (namespace `akk-api`, database `akk_kercheval`).

  > Not committed until you generate it (see below). The plaintext Secret must
  > **never** be committed.

## Generate the DB SealedSecret

Run on a machine that has `kubeseal` and `kubectl` access to the cluster (or use
the Kubeseal VS Code extension, as done for bccmusic). The `akk-api` namespace
must exist first (it is created by `apps/namespaces.yaml`).

```bash
# 1. Build a plaintext Secret WITHOUT applying it to the cluster.
#    Replace <MSSQL-LXC-IP> and the SQL login/password with the real values.
kubectl create secret generic akk-db-credentials \
  --namespace=akk-api \
  --from-literal=connectionString="Server=<MSSQL-LXC-IP>,1433;Database=akk_kercheval;User Id=wsakkapi;Password=CHANGE_ME;TrustServerCertificate=True;Encrypt=False" \
  --dry-run=client -o yaml > /tmp/akk-db-credentials.yaml

# 2. Seal it against the running controller.
kubeseal \
  --controller-namespace sealed-secrets \
  --controller-name sealed-secrets \
  --format yaml \
  < /tmp/akk-db-credentials.yaml \
  > secrets/sealedsecrets/akk-db-credentials-sealed.yaml

# 3. Delete the plaintext file immediately.
rm /tmp/akk-db-credentials.yaml

# 4. Commit ONLY the sealed file.
git add secrets/sealedsecrets/akk-db-credentials-sealed.yaml
git commit -m "Add sealed DB credentials for akk-api"
git push
```

Argo CD applies the SealedSecret, the controller decrypts it into a `Secret`
named `akk-db-credentials`, and the `ws-akk-api` Deployment reads
`connectionString` from it.

## Rotating the credentials later

See the root `README.md` and the reusable runbook in
`My Web Sites/Onboarding a New Website to the CI-CD Pipeline.md` for the full
clean-rotation procedure.
