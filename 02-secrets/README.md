#Variables

## ./templates/files/monitoring/values-secret.yaml
- **GRAFANA_ADMIN_USER** - grafana admin username
- **GRAFANA_ADMIN_PASSWORD** - grafana admin password

## ./templates/files/cattle-system/values-secret.yaml
- **CLOUDFLARE_ZONE** - normally domain
- **RANCHER_ADMIN_PASSWORD** - password for the rancher admin

## ./templates/files/cert-manager/helper-values-secret.yaml
- **CLOUDFLARE_EMAIL** - email associated with cloudflare_domain
- **LETS_ENCRYPT_EMAIL** - email associated with let's encrypt

## ./templates/files/traefik/helper-values-secret.yaml
- **CLOUDFLARE_ZONE** - normally domain
- **CLOUDFLARE_ZONE_HYPHEN** - domain, but instead of . subtituted with - (I know
there was probably a better way)

## ./templates/files/traefik/values-secret.yaml
- **LOAD_BALANCER_IP** - local ip of the load balancer

## ./templates/secrets/cloudflare-token-secret.yaml
- **CLOUDFLARE_TOKEN** - generated token in cloudflare (to automaticly setup the ip
addresss, probably?? too much time passed since I made this)

## ./templates/secrets/dashboard-secret.yaml
- **TRAEFIK_USERS** - users and passwords of the traefik (probably better to check,
but value coded base64, was a string from command `htpasswd -nbm user password`)

## ./templates/16-tailscale/tailscale/values-secret.yaml
- **TS_OAUTH_CLIENT_ID** - Tailscale OAuth client id (admin console → Settings → Trust
credentials; `write` scope for `General/Services`, `Devices/Core`, `Keys/Auth Keys`,
each tagged `tag:k8s-operator`)
- **TS_OAUTH_CLIENT_SECRET** - the matching OAuth client secret

# Create sealed secret

ls seal-secrets | xargs -I {} rm -f seal-secrets/{}
find files -type f -name '*.yaml' | while read file; do namespace=$(basename $(dirname "$file")); name=$(basename "$file" .yaml); kubectl create secret generic $name --namespace="$namespace" --from-file=values.yaml="$file" --dry-run=client -o yaml | yq eval '.metadata.annotations += {"reflector.v1.k8s.emberstack.com/reflection-allowed": "true", "reflector.v1.k8s.emberstack.com/reflection-allowed-namespaces": ""}' > "secrets/${namespace}-${name}.yaml"; done

```bash
find files -type f -name '*.yaml' | while read file; do namespace=$(basename $(dirname "$file")); name=$(basename "$file" .yaml); kubectl create secret generic $name --namespace="$namespace" --from-file=values.yaml="$file" --dry-run=client -o yaml > "secrets/${namespace}-${name}.yaml"; done
ls secrets | xargs -I {} sh -c 'kubeseal -f secrets/$1 -o yaml > seal-secrets/seal-$1' -- {}
echo "resources:" > seal-secrets/kustomization.yaml
ls secrets | xargs -I {} sh -c 'echo "- seal-$1" >> seal-secrets/kustomization.yaml' -- {}
```

# Create secrets

```bash
kubectl create secret generic {NAME} --namespace={NAMESPACE} --from-literal={KEY}={VALUE} --dry-run=client -o yaml > {NAME}-secret.yaml
```
