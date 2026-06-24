# 18-owncloud — ownCloud Infinite Scale (OCIS)

Self-hosted file sync/cloud, deployed as the **single-binary** OCIS server.
OCIS keeps **all** data — file blobs *and* metadata — on one volume, so there is
no database. rclone works against it via WebDAV.

## Bundles

| Bundle | Type | What it does |
|--------|------|--------------|
| `01-storage` | Kustomize | `longhorn-owncloud` StorageClass (2 replicas, hard node anti-affinity) + `ocis-data` PVC (100Gi, RWO). |
| `02-ocis` | Helm (inline) | OCIS Deployment + Service + Traefik IngressRoute. Mounts `ocis-data` at `/var/lib/ocis` (data, subPath `data`) and `/etc/ocis` (config, subPath `config`). |
| `secrets` | Kustomize | SealedSecret `ocis-secrets` holding the admin password. |

`02-ocis` `dependsOn` both `01-storage` and `secrets`, so it only rolls out once
the volume and the password exist.

## URL

`https://owncloud.<cloudflare_zone>` (TLS terminated at Traefik; OCIS proxy runs
plain HTTP on `:9200`). Admin user: `admin`.

## One manual step before first deploy: seal the admin password

The admin password is the only secret you must provide.

```bash
cd 02-secrets
cp -r templates/18-owncloud bundles/18-owncloud   # then edit the real password:
#   bundles/18-owncloud/owncloud/ocis-secrets.yaml  -> set stringData.admin-password
./install.sh                                       # seals it into 18-owncloud/secrets/
cd ..
git add 18-owncloud/secrets
```

`install.sh` writes `18-owncloud/secrets/seal-secret-ocis-secrets.yaml` and the
matching `kustomization.yaml`. Commit those; Fleet does the rest.

## Notes

- **Image** is pinned to `owncloud/ocis:8.0.5`. Bump it deliberately; OCIS
  config can change across major versions, so read release notes before jumping
  majors.
- **rclone** (`webdav`, `vendor = owncloud`): create an *app token* in the OCIS
  web UI (top-right settings), then point rclone at
  `https://owncloud.<zone>/remote.php/webdav/`.
- Capacity = the `ocis-data` PVC size. Grow it by raising `storage:` in
  `01-storage/storage/pvc.yaml` (the StorageClass allows expansion).
