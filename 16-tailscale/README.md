# 16-tailscale

Access cluster services over Tailscale using your real `mourahouse.com`
hostnames. Built on the official Tailscale Kubernetes operator
(<https://tailscale.com/kb/1236/kubernetes-operator>).

## How it works

```
tailnet client ──▶ split-DNS: *.mourahouse.com ─▶ house-dns (CoreDNS, tailnet IP)
                                                    └▶ answers A = Traefik LB IP
              ──▶ subnet route 192.168.1.0/24 (Connector) ─▶ Traefik ─▶ service
```

- **tailscale-operator/** – the operator (Helm) + the `Connector` subnet router
  advertising `192.168.1.0/24` (where the Traefik MetalLB LB lives).
- **dns/** – a wildcard CoreDNS that answers every `*.mourahouse.com` with the
  Traefik LB IP, exposed to the tailnet via `loadBalancerClass: tailscale`.
- **secrets/** – sealed `values-secret` holding the operator OAuth credentials.

A wildcard resolver is used (not k8s_gateway) because services are split across
standard `Ingress` and Traefik `IngressRoute` CRDs but all share one Traefik LB —
so a single `*.mourahouse.com → LB IP` answer covers everything.

## One-time setup

### 1. Fill in the Traefik LB IP

Edit `dns/coredns.yaml` and replace `REPLACE_TRAEFIK_LB_IPV4` with the Traefik
LoadBalancer's LAN IPv4 (the same `loadBalancerIPs` value used in 05-traefik).

### 2. Tailscale admin console

- **OAuth client** (admin console → Settings → Trust credentials): `write` scope for
  all three, each tagged `tag:k8s-operator`:
  - `General/Services`
  - `Devices/Core`
  - `Keys/Auth Keys`

  Put the id/secret in the sealed secret (step 3).
- **ACL policy** — two-tag convention (operator tag owns the proxy tag) plus route
  auto-approval:
  ```jsonc
  "tagOwners": {
    "tag:k8s-operator": ["autogroup:admin"],
    "tag:k8s":          ["tag:k8s-operator"]
  },
  "autoApprovers": { "routes": { "192.168.1.0/24": ["tag:k8s"] } }
  ```
  The OAuth client / operator device is `tag:k8s-operator`; the operator then stamps
  `tag:k8s` onto the Connector and proxy nodes (matches `connector.yaml`).

### 3. Seal the OAuth secret

```bash
cp 02-secrets/templates/16-tailscale/tailscale/values-secret.yaml \
   02-secrets/bundles/16-tailscale/tailscale/values-secret.yaml
# edit the bundle copy, fill TS_OAUTH_CLIENT_ID / TS_OAUTH_CLIENT_SECRET
cd 02-secrets && ./install.sh      # generates 16-tailscale/secrets/seal-secret-values-secret.yaml
```

### 4. After deploy: point split-DNS at house-dns

```bash
kubectl get svc -n house-dns house-dns      # note the tailnet IP (100.x.y.z)
```
In the admin console → DNS → **Split DNS**, add nameserver `100.x.y.z` restricted
to domain `mourahouse.com`.

Then from any tailnet device: `https://jellyfin.mourahouse.com` resolves to the
Traefik LB over the subnet route, with your existing cert-manager TLS.
