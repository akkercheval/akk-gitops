# cloudflared (akk tunnel)

The Cloudflare Tunnel connector for `akk.kercheval.cc`, managed by Argo CD.

- `cloudflared-akk-token-sealed.yaml` — the tunnel token, sealed (safe to
  commit). Decrypts into a `Secret` named `cloudflared-akk-token` in the
  `cloudflared` namespace.
- `deployment.yaml` — the `cloudflared-akk` Deployment that runs the connector
  and reads the token from that secret.

Both live in the **`cloudflared`** namespace, alongside the existing bccmusic
tunnel (`cloudflared`). The two tunnels are separate deployments/secrets in the
same namespace and do not conflict.

## Rotating the tunnel token

If you recreate the tunnel in Cloudflare (new token):

```bash
kubectl create secret generic cloudflared-akk-token -n cloudflared \
  --from-literal=token=NEW_TOKEN --dry-run=client -o yaml \
  | kubeseal --controller-namespace sealed-secrets --controller-name sealed-secrets --format yaml \
  > infra/cloudflared/cloudflared-akk-token-sealed.yaml
# commit + push; Argo applies it, then:
kubectl rollout restart deployment cloudflared-akk -n cloudflared
```
