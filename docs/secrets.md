# Secrets on bee

Three places hold credentials, and only one of them is a source of truth.

| Place | Holds | Managed by |
| --- | --- | --- |
| OpenBao (`openbao` namespace) | `secret/cert-manager/cloudflare`, `secret/epi-trainer/basic-auth`, `secret/monitoring/*`, database and PKI engines | Argo CD installs the server; the mounts, auth roles and policies are a one-time bootstrap |
| Kubernetes Secrets | copies synced by External Secrets Operator from OpenBao | ESO, `manifests/openbao/externalsecrets.yaml` and `manifests/observability-secrets/eso.yaml` |
| `/var/lib/secrets` on the host | bootstrap credentials and host-only consumers | root, out of band |

OpenBao is a single-node standalone server (pebbledb) with a 2 GiB data PVC
and a 1 GiB audit PVC. It is not exposed outside the cluster; ESO reaches it at
`http://openbao.openbao.svc:8200`. In-cluster traffic is HTTP, so the cluster
network is the trust boundary and no NetworkPolicy currently narrows it.

## Initialise OpenBao (once, after the `openbao` app is Synced)

The chart installs the server only. The mounts, Kubernetes auth and ESO role
below are bootstrap state created with the root token; they are not reconciled
from this repository. Do this once, and record the unseal key and the orphan
admin token in `/var/lib/secrets` before revoking the root token.

```bash
# 1. Initialise with a single share and keep the JSON off the terminal.
ssh bee 'kubectl -n openbao exec openbao-0 -- bao operator init \
  -key-shares=1 -key-threshold=1 -format=json' > openbao-init.json
# unseal_keys_b64[0] -> /var/lib/secrets/openbao-unseal (0600 root)
# root_token          -> a root-only file used only for this bootstrap

# 2. Keep the key root-only on the host. The NixOS timer in nixos-config
#    (hosts/oracle) reads it and retries every two minutes; no Kubernetes
#    Secret holds it.

# 3. Unseal. The CLI refuses a non-TTY, so feed it from a file inside the pod.
ssh bee 'kubectl -n openbao exec -i openbao-0 -- sh -c "umask 077; cat > /tmp/unseal.key"' < unseal.key
ssh bee 'kubectl -n openbao exec openbao-0 -- sh -c '\''bao operator unseal "$(cat /tmp/unseal.key)"; rm -f /tmp/unseal.key'\'''
```

With the root token copied into the pod as `/tmp/root.token`, create the
engines and the ESO identity:

```bash
bao secrets enable -path=secret kv-v2
bao secrets enable database
bao secrets enable pki
bao secrets tune -max-lease-ttl=87600h pki
bao write pki/root/generate/internal common_name="bee internal CA" ttl=87600h
bao write pki/roles/internal allowed_domains="svc.cluster.local,internal" \
  allow_subdomains=true max_ttl=72h

bao auth enable kubernetes
bao write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

# One identity per consumer. ESO mints a token for the eso-openbao
# ServiceAccount in each namespace; each role reads only its own prefix.
bao policy write eso-cert-manager - <<'EOF'
path "secret/data/cert-manager/*"     { capabilities = ["read"] }
path "secret/metadata/cert-manager/*" { capabilities = ["read", "list"] }
EOF
bao policy write eso-epi-trainer - <<'EOF'
path "secret/data/epi-trainer/*"     { capabilities = ["read"] }
path "secret/metadata/epi-trainer/*" { capabilities = ["read", "list"] }
EOF
bao write auth/kubernetes/role/eso-cert-manager \
  bound_service_account_names=eso-openbao \
  bound_service_account_namespaces=cert-manager \
  policies=eso-cert-manager ttl=1h
bao write auth/kubernetes/role/eso-epi-trainer \
  bound_service_account_names=eso-openbao \
  bound_service_account_namespaces=epi-trainer \
  policies=eso-epi-trainer ttl=1h

# The database engine uses the application's Postgres user, which owns the
# database and can create roles. Read both values from the ticket-queue-db
# Secret; they are not repeated here.
bao write database/config/ticket-queue \
  plugin_name=postgresql-database-plugin \
  allowed_roles="ticket-queue-readonly" \
  connection_url="postgresql://{{username}}:{{password}}@ticket-queue-db.ticket-queue.svc.cluster.local:5432/relay?sslmode=disable" \
  username="<POSTGRES_USER>" password="<POSTGRES_PASSWORD>"
bao write database/roles/ticket-queue-readonly db_name=ticket-queue \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT CONNECT ON DATABASE relay TO \"{{name}}\"; GRANT USAGE ON SCHEMA public TO \"{{name}}\"; GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl=1h max_ttl=24h

# Break-glass admin credential. -orphan is required: revoking the root token
# would otherwise revoke this child too.
bao policy write platform-admin - <<'EOF'
path "*" {
  capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}
EOF
bao token create -orphan -policy=platform-admin -ttl=720h \
  -renewable=true -display-name=platform-admin -format=json
# store auth.client_token at /var/lib/secrets/openbao-admin-token (0600 root)

# Seed the two KV entries. Their sources are the host files described in the
# rotation sections below.
bao kv put secret/cert-manager/cloudflare api-token=@/tmp/cf
bao kv put secret/epi-trainer/basic-auth users=@/tmp/epi.users

# Revoke the root token only after the orphan token and both KV entries exist.
bao token revoke -self
```

## Bootstrap credentials

These cannot live in OpenBao, because OpenBao needs them to start.

- `/var/lib/secrets/openbao-unseal` (0600 root) holds the single unseal key.
  The `openbao-unseal.timer` in `nixos-config` (`hosts/oracle`) reads it and
  retries every two minutes after a pod or node restart, so the key never
  enters Kubernetes and cluster-admin alone cannot unseal. Losing the file
  (and its offline backup) means the data is unrecoverable.
- `/var/lib/secrets/openbao-admin-token` (0600 root) is an **orphan** token
  with the `platform-admin` policy and a 30-day TTL. It is the credential for
  `bao` operations and the database/PKI recipes below. Renew it before it
  expires. Create it with `-orphan`: revoking a parent token cascades to its
  non-orphan children, which is how an otherwise correct bootstrap loses its
  admin token when the root token is revoked.
- `/var/lib/secrets/cloudflare.env` is read by the host's ACME client for the
  `mail.khong.dev` certificate. The same token is also in OpenBao for
  cert-manager; a rotation updates both.

Stalwart receives its secrets through systemd `LoadCredential=`, so they are
read from `/var/lib/secrets` at unit start rather than copied into the Nix
store.

## Rotating the Cloudflare DNS token

1. Create a replacement token with `Zone:DNS:Edit` and `Zone:Read` on
   `khong.dev`, an expiry of at most one year, and a descriptive name.
2. Replace the value in `/var/lib/secrets/cloudflare.env` (0600 root).
3. Put the same value into OpenBao. Copy it into the pod over stdin, then use
   the admin token from the host:

   ```bash
   ssh bee "sudo -n cat /var/lib/secrets/openbao-admin-token" \
     | ssh bee 'kubectl -n openbao exec -i openbao-0 -- sh -c "umask 077; cat > /tmp/admin.token"'
   ssh bee "printf '%s' '<new-token>' | kubectl -n openbao exec -i openbao-0 -- sh -c 'umask 077; cat > /tmp/cf'"
   ssh bee 'kubectl -n openbao exec openbao-0 -- sh -c '\''BAO_TOKEN=$(cat /tmp/admin.token) bao kv put secret/cert-manager/cloudflare api-token=@/tmp/cf; rm -f /tmp/admin.token /tmp/cf'\'''
   ```

   `/tmp` does not survive a pod restart, so repeat the copy in a new pod
   rather than assuming the files are still there.
4. The ExternalSecret refreshes within an hour; force it with
   `kubectl -n cert-manager annotate externalsecret cloudflare-api-token force-sync=$(date +%s) --overwrite`.
5. Prove DNS-01 with a throwaway `Certificate` before deleting the old token.
   Cert-manager and the host ACME client both read the replacement; the old
   token stays valid until deleted, which is the rollback.

## Rotating the epi-trainer basic-auth password

The plaintext is in `/var/lib/secrets/epi-trainer-basic-auth.env`. Regenerate
the hash at bcrypt cost 12, store the line in OpenBao, and confirm the ingress
still enforces it:

```bash
nix shell nixpkgs#apacheHttpd -c htpasswd -niB -C 12 krishna   # reads the password on stdin
# write the resulting krishna:$2y$12$... line to a file, copy it into the pod,
# then:
bao kv put secret/epi-trainer/basic-auth users=@/tmp/epi.users
kubectl -n epi-trainer annotate externalsecret epi-trainer-basic-auth \
  force-sync=$(date +%s) --overwrite
```

`curl -u krishna:<password> https://epi-trainer-auth.khong.dev/` must still
return 302, and a wrong password must return 401. Update
`/var/lib/secrets/epi-trainer-basic-auth.users` as well; it is the source for
the next rebuild.

## Grafana and Alertmanager credentials

Grafana's admin password and Alertmanager's SMTP password are synced into
`monitoring` by External Secrets, using the `eso-monitoring` role and the
`monitoring/grafana` and `monitoring/alertmanager` KV entries. Bootstrap once,
after the `openbao` app is Synced (copy the admin token into the pod first, as
in the incantation above):

```bash
# Inside openbao-0, after copying the admin token to /tmp/admin.token as above.
export BAO_TOKEN=$(cat /tmp/admin.token)
bao policy write eso-monitoring - <<'EOF'
path "secret/data/monitoring/*"     { capabilities = ["read"] }
path "secret/metadata/monitoring/*" { capabilities = ["read", "list"] }
EOF
bao write auth/kubernetes/role/eso-monitoring \
  bound_service_account_names=eso-openbao \
  bound_service_account_namespaces=monitoring \
  policies=eso-monitoring ttl=1h
```

Generate the Grafana admin password without printing it, then store both
values (the commands run on your machine; the password files stay in the pod's
`/tmp`, which does not survive a restart):

```bash
head -c 32 /dev/urandom | base64 | tr -dc 'A-Za-z0-9' | head -c 32 \
  | base64 -w0 \
  | kubectl -n openbao exec -i openbao-0 -- sh -c 'umask 077; base64 -d > /tmp/grafana.pass'
ssh bee 'sudo cat /var/lib/secrets/stalwart-krishna' \
  | base64 -w0 \
  | kubectl -n openbao exec -i openbao-0 -- sh -c 'umask 077; base64 -d > /tmp/smtp.pass'

kubectl -n openbao exec openbao-0 -- sh -c '
  export BAO_TOKEN=$(cat /tmp/admin.token)
  bao kv put secret/monitoring/grafana admin-user=admin admin-password=@/tmp/grafana.pass
  bao kv put secret/monitoring/alertmanager smtp-password=@/tmp/smtp.pass
  rm -f /tmp/grafana.pass /tmp/smtp.pass'
```

Read the admin password with:

```bash
kubectl -n monitoring get secret grafana-admin \
  -o jsonpath='{.data.admin-password}' | base64 -d; echo
```

Rotating either value is a `bao kv put`, then
`kubectl -n monitoring annotate externalsecret <name> force-sync=$(date +%s) --overwrite`.
Grafana reads its admin password only at start, so restart the deployment after
a rotation; Alertmanager reloads its config from the Secret on its own.

Alertmanager delivers through Stalwart at `mail.khong.dev:587` as the
`krishna@khong.dev` account; the recipient and routing policy live in
`manifests/observability/eso.yaml`, not in OpenBao.

## Rotating the OCI SMTP credential

The relay credential is a long-lived OCI IAM SMTP credential. OCI takes a few
minutes to propagate a new one, so keep the old credential until a message has
been delivered.

```bash
oci iam smtp-credential create --user-id <user-ocid> --description stalwart-relay-<date>
# write .data.username and .data.password to
# /var/lib/secrets/stalwart-relay-oci-user and -pass (0600 root)
sudo systemctl restart stalwart
# send a test through Stalwart, confirm "Message delivered", then:
oci iam smtp-credential delete --user-id <user-ocid> --smtp-credential-id <old-id> --force
```

`oci-smtp.json` was a CLI artefact that duplicated the password; do not recreate
it.

## Database credentials

OpenBao's database engine has a read-only role against the Relay Desk database:

```bash
# Copy the admin token into the pod first; /tmp does not survive a restart.
kubectl -n openbao exec openbao-0 -- sh -c \
  'BAO_TOKEN=$(cat /tmp/admin.token) bao read database/creds/ticket-queue-readonly'
```

The role is created with the application's own Postgres user, which is the
container superuser, and the grants allow `CONNECT`, `USAGE` and `SELECT` on
the existing tables. Credentials live for one hour (maximum 24 hours) and
`bao lease revoke <lease>` drops the Postgres role immediately. The application
still consumes the static `ticket-queue/ticket-queue-db` Secret; moving it to a
dynamic credential plus a pod restart on rotation is the next step.

## PKI

`pki/issue/internal` issues certificates under `bee internal CA` for
`*.svc.cluster.local` and `*.internal`, with a maximum TTL of 72 hours. The
public certificates stay with cert-manager and Let's Encrypt; this PKI is for
in-cluster identities.

## Residual risks

- The 2 GiB pebbledb PVC is the only copy of OpenBao's state. The two KV
  entries can be rebuilt from `/var/lib/secrets/cloudflare.env` and
  `…/epi-trainer-basic-auth.users`, but the PKI and database configuration
  would be recreated from the commands above. Add a volume snapshot before
  relying on it.
- `openbao-config` manages resources in the cert-manager and epi-trainer
  namespaces. Pruning that Application deletes the ExternalSecrets and
  SecretStores, and their `ownerReferences` then garbage-collect
  `cert-manager/cloudflare-api-token` and `epi-trainer/epi-trainer-basic-auth`
  until ESO recreates them.
- The in-cluster listener is HTTP with no NetworkPolicy; every pod can reach
  `openbao:8200`, though a request without a token is rejected.
