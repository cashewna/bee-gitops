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
manifests/ticket-queue/       # demo.ticket-queue.khong.dev (Relay Desk)
manifests/epi-trainer/        # epi-trainer read-only public + auth host
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

## Demo data

`demo.ticket-queue.khong.dev` is a public demo of Relay Desk. The deployment sets
`SEED_DEMO=true` with the app's documented non-production password, so the login
page shows:

- `requester@example.com`
- `agent@example.com`
- password `demo-pass-123`

Anyone can change the queue. Do not put real data there. A CronJob
(`manifests/ticket-queue/demo-reset-cronjob.yaml`) truncates the demo tables and
reseeds every 30 minutes, then recreates the private maintenance account from
`ticket-queue-admin`. The seeded accounts are created only when the database has
no users; to reset manually, drop the schema and restart the deployment.

## Secrets

No secret values live here. Create these out of band:

- `cert-manager/cloudflare-api-token` — Cloudflare DNS edit token.
- `ticket-queue/ticket-queue-db` — `POSTGRES_USER`, `POSTGRES_DB`,
  `POSTGRES_PASSWORD`, `DATABASE_URL`.
- `epi-trainer/epi-trainer-basic-auth` — htpasswd `users` entry for the
  unsandboxed Java judge. The plaintext credentials are root-only on bee at
  `/var/lib/secrets/epi-trainer-basic-auth.env`.
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

# epi-trainer (source is /home/kk/Projects/epi-trainer on the laptop)
rsync -a --exclude node_modules --exclude backend/target --exclude dist \
  --exclude .git --exclude .cache --exclude tests \
  ~/Projects/epi-trainer/ bee:/home/kk/build/epi-trainer/
cd /home/kk/build/epi-trainer
sudo nix shell nixpkgs#podman --command podman build \
  -t docker.io/cashewna/epi-trainer:sha-<commit> .
sudo podman save docker.io/cashewna/epi-trainer:sha-<commit> \
  | sudo k3s ctr -n k8s.io images import -
```

## DNS

Cloudflare zone `khong.dev`. Create DNS-only A records to `137.23.18.86`:

- `portfolio.khong.dev`
- `demo.ticket-queue.khong.dev`
- `apps.khong.dev` (redirects to the demo host)
- `epi-trainer.khong.dev` (read-only public)
- `epi-trainer-auth.khong.dev` (basic auth; full editor/judge)

The apex `khong.dev` stays on Porkbun parking and is not touched. Mail records
(MX, SPF, DKIM) are not touched.

## Certificates

`infra/cert-manager/clusterissuer.yaml` defines `letsencrypt-dns`, a
ClusterIssuer using the Cloudflare DNS-01 solver. Ingresses request
certificates with `cert-manager.io/cluster-issuer: letsencrypt-dns`. DNS-01 is
used so Cloudflare proxying can be enabled later without switching challenge
types.
