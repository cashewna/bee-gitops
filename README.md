# bee-gitops

GitOps manifests for the single-node k3s cluster on **bee** (OCI, aarch64).

Argo CD reconciles this public repository from `main`. The cluster runs
Traefik, local-path storage and cert-manager; applications are exposed on
`*.khong.dev` with Let's Encrypt certificates issued by DNS-01.

## Layout

```
argocd/root-app.yaml          # app-of-apps entry point
argocd/apps/*.yaml            # Argo CD Application resources
infra/cert-manager/           # Let's Encrypt ClusterIssuer
manifests/portfolio/          # portfolio.khong.dev
manifests/ticket-queue/       # apps.khong.dev (Relay Desk)
```

## Bootstrap

The cluster already has k3s, Traefik, local-path and metrics-server. Argo CD and
cert-manager are installed once with pinned upstream manifests:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.6.1/manifests/install.yaml

kubectl create namespace cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.21.3/cert-manager.yaml
```

Create the Cloudflare API token Secret out of band (the token is root-only on
bee at `/var/lib/secrets/cloudflare.env` and must never enter this repository):

```bash
kubectl -n cert-manager create secret generic cloudflare-api-token \
  --from-literal=api-token="$CLOUDFLARE_DNS_API_TOKEN"
```

Then apply the app-of-apps:

```bash
kubectl apply -f argocd/root-app.yaml
kubectl -n argocd get applications
```

Argo CD's UI is not exposed publicly. Reach it through an SSH tunnel:

```bash
ssh -L 8080:127.0.0.1:8080 bee \
  'sudo k3s kubectl -n argocd port-forward svc/argocd-server 8080:443'
# https://127.0.0.1:8080 — user admin
# password: sudo k3s kubectl -n argocd get secret argocd-initial-admin-secret \
#   -o jsonpath='{.data.password}' | base64 -d
```

## Secrets

No secret values live here. Create these out of band:

- `cert-manager/cloudflare-api-token` — Cloudflare DNS edit token.
- `ticket-queue/ticket-queue-db` — `POSTGRES_USER`, `POSTGRES_DB`,
  `POSTGRES_PASSWORD`, `DATABASE_URL`.
- The Relay Desk agent account is created once with the app's
  `create-user-cli.js`; the password is kept root-only on bee at
  `/var/lib/secrets/ticket-queue-admin.env`.

## Images

Images are built natively on bee (aarch64) with podman and imported into k3s
containerd. They are referenced by immutable tag and `imagePullPolicy:
IfNotPresent`. Rebuild and bump the tag in the manifest for each release; do
not use `latest`.

```bash
# ticket-queue
git clone https://github.com/cashewna/ticket-queue.git /srv/build/ticket-queue
cd /srv/build/ticket-queue
sudo nix shell nixpkgs#podman --command podman build \
  -t docker.io/cashewna/ticket-queue:sha-<commit> .
sudo podman save docker.io/cashewna/ticket-queue:sha-<commit> \
  | sudo k3s ctr -n k8s.io images import -

# portfolio
git clone https://github.com/cashewna/portfolio.git /srv/build/portfolio
cd /srv/build/portfolio
sudo nix shell nixpkgs#podman --command podman build \
  -t docker.io/cashewna/portfolio:sha-<commit> .
sudo podman save docker.io/cashewna/portfolio:sha-<commit> \
  | sudo k3s ctr -n k8s.io images import -
```

## DNS

Cloudflare zone `khong.dev`. Create DNS-only A records to `137.23.18.86`:

- `portfolio.khong.dev`
- `apps.khong.dev`

The apex `khong.dev` stays on Porkbun parking and is not touched. Mail records
(MX, SPF, DKIM) are not touched.

## Certificates

`infra/cert-manager/clusterissuer.yaml` defines `letsencrypt-dns`, a
ClusterIssuer using the Cloudflare DNS-01 solver. Ingresses request
certificates with `cert-manager.io/cluster-issuer: letsencrypt-dns`. DNS-01 is
used so Cloudflare proxying can be enabled later without switching challenge
types.
