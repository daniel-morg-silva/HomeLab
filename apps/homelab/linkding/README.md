# Linkding

[Linkding](https://github.com/sissbruecker/linkding) is a self-hosted bookmark manager. It runs as a single container with a SQLite database.

## Components

| File | Kind | Description |
|------|------|-------------|
| `namespace.yaml` | Namespace | Creates the `linkding` namespace |
| `deployment.yaml` | Deployment | Runs the `sissbruecker/linkding` container |
| `service.yaml` | Service | Exposes the pod on port 9090 within the cluster |
| `ingress.yaml` | Ingress | Routes `danielmorgsilva.net/linkding` to the service |
| `secrets.yaml` | Secret (SOPS) | Superuser password, age-encrypted |

## Access

```
https://danielmorgsilva.net/linkding
```

## Key Configuration

**Subpath deployment:** Linkding is served at `/linkding` (not the root). This requires setting `LD_CONTEXT_PATH=linkding` (no leading slash) in the deployment. The app prepends its own `/` internally — setting it to `/linkding` results in `//linkding` and breaks routing.

**Mergeable Ingress:** Because this cluster uses the NGINX Inc controller, the Ingress here is a `minion` that adds the `/linkding` path to the `master` Ingress in the `default` namespace. Without this, the controller rejects the Ingress with `host is taken by another resource`. See [architecture.md](../../../docs/architecture.md).

```yaml
annotations:
  nginx.org/mergeable-ingress-type: "minion"
```

## Problems Solved During Setup


**Ingress rejected across namespaces:** The NGINX Inc controller (unlike the community ingress-nginx) does not merge Ingress resources across namespaces for the same host. Creating a separate Ingress in `linkding` with `host: danielmorgsilva.net` caused the error `host is taken by another resource`. Fixed by using Mergeable Ingresses (master/minion pattern).

**Master Ingress cannot have paths:** When converting to Mergeable Ingresses, the master Ingress must define only the host — no `http.paths` block. Having paths on the master causes it to be rejected.

**Double slash in URL paths:** Setting `LD_CONTEXT_PATH=/linkding` (with leading slash) caused linkding to internally generate paths like `//linkding/static`, breaking all assets. The correct value is `linkding` without the slash.

**SOPS secrets not decrypted:** The `apps` Flux Kustomization was missing the `decryption` block, so secrets were applied with their encrypted SOPS values as the literal password. The pod started but the superuser was created with an unusable password. Fixed by adding to `clusters/homelab/apps.yaml`:
```yaml
spec:
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```
After adding this, the pod needed a restart (`kubectl rollout restart deployment/linkding -n linkding`) because the superuser is only created on first startup.
