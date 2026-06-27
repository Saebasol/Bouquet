# Kubernetes migration for Bouquet

This directory contains a Kubernetes baseline converted from `docker-compose.yaml`.

## What is included

- Namespace: `bouquet`
- Stateful data services with PVCs:
  - PostgreSQL via **CloudNativePG** (`postgres`) — 2-instance HA cluster
  - MongoDB (`mongodb`) — StatefulSet
- Application services:
  - Heliotrope (`heliotrope`)
  - Sunflower (`sunflower`)
  - Hibiscus (`hibiscus`)
- IngressRoute (Traefik CRD) for Traefik + TLS + `www` redirect middlewares
- Secret and config templates

## Prerequisites

- Kubernetes cluster (k3s 1-control + 1-worker tested)
- **CloudNativePG Operator** installed:
  ```bash
  kubectl apply -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.25/releases/cnpg-1.25.1.yaml
  ```
- Traefik Ingress Controller installed
- Traefik must have ACME cert resolver named `cfresolver`
  - Example static args:
    - `--certificatesresolvers.cfresolver.acme.dnschallenge.provider=cloudflare`
    - `--certificatesresolvers.cfresolver.acme.email=<your_email>`
    - `--certificatesresolvers.cfresolver.acme.storage=/data/acme.json`

## 1) Fill placeholders

Edit `secret.example.yaml` and replace all placeholder values.

Edit `ingress.yaml` and replace domains:

- `example.com` (heliotrope + sunflower)
- `hibiscus.example.com` (hibiscus)

`www` subdomains are automatically redirected to the root domain
by the `redirect-to-root` middleware.

Create `traefik-cloudflare.example.yaml` with your Cloudflare API token,
then apply it to Traefik namespace (default example: `traefik`).

## 2) Apply resources

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/secret.example.yaml
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/postgres.yaml
kubectl apply -f k8s/mongodb.yaml
kubectl apply -f k8s/heliotrope.yaml
kubectl apply -f k8s/sunflower.yaml
kubectl apply -f k8s/hibiscus.yaml
kubectl apply -f k8s/middlewares.yaml
kubectl apply -f k8s/ingress.yaml

# apply this to Traefik namespace after replacing token value
kubectl apply -f k8s/traefik-cloudflare.example.yaml
```

## Notes

- `hibiscus` image is set to `ghcr.io/saebasol/hibiscus:latest`.
  - If this image does not exist in your registry, build and push it first, then change the image field in `hibiscus.yaml`.
- In the original compose, Sunflower labels and ports looked inconsistent.
  - This migration exposes Sunflower via IngressRoute for `/` and `/api/status`
    on `example.com`, with Heliotrope handling all other paths as a catch-all.
- DB services:
  - PostgreSQL uses **CloudNativePG** `Cluster` CRD with 2 instances
    (primary + replica) for high availability and automatic failover.
  - MongoDB uses `StatefulSet` with `volumeClaimTemplates`
    and liveness/readiness probes.
- Application services include liveness/readiness probes.
- For production reliability, also consider:
  - enabling CNPG backups (S3-compatible storage)
  - backup strategy for MongoDB PVCs
  - using explicit image tags instead of `:latest`
  - adding resource requests/limits based on your node capacity

## PostgreSQL backend selection

Two PostgreSQL configurations are provided:

| File | Mode | HA | Requires | DB host in secret |
|------|------|----|----------|--------------------|
| `postgres.yaml` | **CNPG** (recommended) | ✅ 2 instances + auto failover | CNPG Operator | `postgres-pooler-rw` |
| `postgres-statefulset.yaml` | **StatefulSet** | ❌ single instance | None | `postgres` |

### CNPG mode (default)

```bash
# 1. Install CNPG Operator (once)
kubectl apply -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.25/releases/cnpg-1.25.1.yaml

# 2. Apply with postgres.yaml (CNPG)
kubectl apply -f k8s/postgres.yaml
```

`secret.example.yaml` DB URLs use `postgres-pooler-rw:5432`.

### StatefulSet mode (fallback)

If you don't want to install CNPG Operator, use the StatefulSet version:

1. Edit `kustomization.yaml`:
   ```yaml
   resources:
     # - postgres.yaml          # CNPG version (comment out)
     - postgres-statefulset.yaml  # StatefulSet version (uncomment)
   ```

2. Edit `secret.example.yaml` DB URLs:
   ```yaml
   HELIOTROPE_GALLERYINFO_DB_URL: "postgresql+asyncpg://...@postgres:5432/heliotrope"
   SUNFLOWER_GALLERYINFO_DB_URL:  "postgresql+asyncpg://...@postgres:5432/heliotrope"
   ```
   (change `postgres-pooler-rw` → `postgres`)

3. `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` are already
   included in `secret.example.yaml` for StatefulSet mode.

4. Apply:
   ```bash
   kubectl apply -f k8s/postgres-statefulset.yaml
   ```

## CNPG notes

- CNPG automatically creates `postgres-rw` (read-write, primary) and
  `postgres-ro` (read-only, replica) services.
- A `Pooler` CRD (PgBouncer) is deployed alongside the cluster.
  Applications connect to `postgres-pooler-rw:5432` — the pooler
  proxies connections to the primary, preventing connection exhaustion.
  `secret.example.yaml` DB URLs point to `postgres-pooler-rw`.
- To enable backups, uncomment the `backup` section in `postgres.yaml`
  and create a `postgres-backup-creds` secret with S3 credentials.
- Check cluster status: `kubectl cnpg status postgres -n bouquet`
- CNPG operator must be installed before applying `postgres.yaml`.
