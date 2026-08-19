# civitas-goat-addon

[CIVITAS/CORE](https://docs.core.civitasconnect.digital/) Ansible addon that installs the [GOAT](https://github.com/plan4better/goat) Helm chart and wires it to civitas's own Postgres (central-db), MinIO (deployed by this addon) and Keycloak realm.

## Status

Installs the GOAT Helm chart (`oci://ghcr.io/plan4better/charts/goat` v0.4.x) into civitas with:

- `<env>-goat-stack` namespace + civitas CA mirroring
- `goat` + `windmill` databases provisioned in civitas's central-db (Zalando `preparedDatabases` patch)
- Keycloak `goat-web` OIDC client in the civitas realm (idempotent)
- MinIO deployment + bucket + DuckLake catalog bootstrap
- Helm install of GOAT (core, web, geoapi, processes, windmill server + 4 workers, redis)
- All GOAT images pinned to a single release tag (`inv_addons.goat.release`) — see [Versioning](#versioning)

## Routing

The whole stack is served from **one hostname** — `inv_addons.goat.web.public_url` (default `https://goat.<DOMAIN>`). Path prefixes route to services; the ingress class is `inv_k8s.ingress_class`, TLS is one shared secret (`goat-<DOMAIN>-tls`) since a single hostname can only be served by one certificate.

| Path prefix     | Service          | Mechanism                                                                                |
|-----------------|------------------|------------------------------------------------------------------------------------------|
| `/`             | goat-web         | host root, no rewrite (Next.js `basePath` is build-time)                                 |
| `/core`         | goat-core        | `API_V2_STR="/core/api/v2"` carries the prefix, no rewrite                               |
| `/geoapi`       | geoapi           | prefix stripped via ingress profile (nginx: `rewrite-target`; traefik: `stripPrefix` MW) |
| `/processes`    | processes        | prefix stripped via ingress profile (nginx: `rewrite-target`; traefik: `stripPrefix` MW) |
| `/<bucket>`     | MinIO            | path prefix equals bucket name (SigV4 constraint) — no rewrite                           |
| —               | windmill         | no public ingress; `processes` reaches it in-cluster                                     |

### Supported ingress controllers

Selected by `inv_k8s.ingress_class`; a startup `assert` fails cleanly if the value has no matching profile.

| `inv_k8s.ingress_class` | Prefix-strip for geoapi/processes | Extra operator setup |
|---|---|---|
| `nginx`   | `nginx.ingress.kubernetes.io/rewrite-target` annotation | none |
| `traefik` | Per-service `stripPrefix` `Middleware` CRD (`tasks/04_ingress_middleware.yml`) | HTTPS-redirect must be configured at the Traefik entrypoint (or via a `redirectScheme` middleware) — the `nginx.ingress.kubernetes.io/ssl-redirect` annotation is silently ignored on Traefik |

Traefik Middleware API group defaults to `traefik.io/v1alpha1` (Traefik v3). For Traefik v2 clusters, override:

```yaml
inv_addons:
  goat:
    ingress:
      traefik_api_group: "traefik.containo.us/v1alpha1"
```

Adding a third controller (e.g. APISIX) is additive: add a profile block under `goat_addon.ingress.profiles.<class>` in `vars/default.yml`, wire whatever per-service CRDs it needs, and the assert stops failing. Do not add elif branches in the template.

### Strategic note — this abstraction is temporary

The profile map exists **only** because geoapi and processes do not support FastAPI's `root_path`. If the upstream `ROOT_PATH` change lands in `plan4better/goat`, both services can serve under their own prefix like goat-core already does — at which point the profile map, the Middleware CRDs, and every controller-conditional annotation can be deleted in favour of plain `pathType: Prefix` paths with no annotations on any controller. Treat any additions to this layer as debt.

**Accepted trade-offs.** geoapi/processes emit OGC/HATEOAS/TileJSON absolute URLs from `request.base_url` that omit the `/geoapi` or `/processes` prefix. The GOAT UI does not consume those URLs (it composes every backend URL itself from `NEXT_PUBLIC_*`), so this only affects external clients pointing directly at those endpoints. Similarly, `/geoapi/api/docs` and `/processes/api/docs` cannot fetch their spec (`openapi_url` is hardcoded to `/api/openapi.json`), though the raw JSON stays reachable.

## Versioning

Every image built from the `plan4better/goat` monorepo pins to a single release tag. Set it via inventory:

```yaml
inv_addons:
  goat:
    release: "v2.4.58"   # applies to core / web / geoapi / processes /
                         # windmill-server / windmill-worker-{default,tools,print}
```

**Why they move together.** `geoapi`, `processes` and `windmill-worker-tools` share a DuckLake catalog through a Postgres schema; the DuckDB extension baked into each image writes the catalog's on-disk format, so a version skew across services produces `DuckLake catalog version mismatch` at attach time. `core` does not use DuckDB (delegates DuckLake to geoapi over HTTP), but is pinned for API compatibility. Non-GOAT images (`redis`, `minio`, `minio_mc`) are pinned independently in `vars/software_references.yml`.

The DuckLake bootstrap Job (`tasks/05_minio.yml`) runs from the `processes` image and uses `AUTOMATIC_MIGRATION TRUE` on ATTACH, so it can upgrade an older catalog to the current release's format. This is the *only* place a catalog upgrade can happen — goatlib's runtime attach hardcodes its option list.

## Inventory

Full schema for the operator's `cc_cli_inventory.yml` under `inv_addons.goat`:

```yaml
inv_addons:
  goat:
    enable: true
    namespace: "{{ ENVIRONMENT }}-goat-stack"
    release: "v2.4.58"
    chart:
      ref: "oci://ghcr.io/plan4better/charts/goat"
      version: "0.4.0"
    db:
      # Optional — Postgres database names. Defaults shown.
      # Override only if you need to co-tenant multiple goat installs in
      # one Postgres cluster. NB windmill's *role* names (windmill_admin,
      # windmill_user) are hardcoded by its migrations regardless.
      goat_db_name: "goat"
      windmill_db_name: "windmill"
    keycloak:
      client_id: "goat-web"
    web:
      # Public-facing URL of the whole goat stack. Everything is served
      # from this hostname and routed by path prefix — see Routing table.
      public_url: "https://goat.{{ DOMAIN }}"
    minio:
      # Bucket name. Also the path prefix on the MinIO ingress: presigned
      # URLs are path-style, so any deviation from `/<bucket>` breaks
      # SigV4 signatures.
      bucket: "goat"
```

## How to use

In your civitas-core fork:

```sh
# Civitas-core's .gitignore excludes `core_platform/addons/`, so the addon
# is cloned directly (not added as a tracked submodule). The Ansible
# playbook reads files from the path regardless of git state.
git clone https://github.com/plan4better/civitas-goat-addon.git \
          core_platform/addons/goat_addon
```

Register the addon's `tasks.yml` in inventory:

```yaml
inv_addons:
  import: true
  addons:
    - "addons/goat_addon/tasks.yml"
  goat:
    # ...see Inventory schema above...
```

Run the civitas playbook scoped to addons:

```sh
ansible-playbook -i cc_cli_inventory.yml core_platform/playbook.yml \
  --tags "addons"
```

## Compatibility

| addon | civitas-core | GOAT chart | GOAT release |
|---|---|---|---|
| **current** | v1.7.x+ | v0.4.x | v2.4.58 |
| v0.2.0      | v1.5.x+ | v0.3.x | mixed (`latest` / `f59d1e3` / `v2.4.36`) |
| v0.1.x      | v1.5.x+ | v0.1.x |  |

Addon version is independent of civitas-core's — tag the addon on its own semver.

## Image references (mirror-friendly)

Every container image this addon spins up — both chart-deployed services and addon-deployed ones (MinIO, mc) — is enumerated in `vars/software_references.yml` under `goat_addon_software.images.*`. Override `registry:` per image in your inventory to point at a private mirror.

```sh
yq '.goat_addon_software.images[] | "\(.registry)/\(.repository):\(.tag)"' \
   vars/software_references.yml
```

**Caveat.** Civitas-core's own `tools/extract-images` and `tools/harbor` playbooks currently only enumerate images under a top-level `software:` key, whereas this addon uses `goat_addon_software:`. As a result the addon's images are not picked up by that tooling today. Track this as a known integration gap.

## Known caveats

- **Postgres-operator stale-password**: if civitas's Zalando postgres-operator has lost its in-memory state (K8s secret rotated but DB password unchanged), `01_db.yml` will hang waiting for the goat secret to appear. Restart the operator and retry:
  ```sh
  kubectl -n cc-loc-operation-stack delete pod -l app.kubernetes.io/name=postgres-operator
  ```

- **`storage_class` is a dict, not a string**: civitas-core's `inv_k8s.storage_class` is `{loc, rwo, rwx}`. The chart wants a single string; the rendered values file picks `loc` by default. Override `inv_k8s.storage_class.loc` (or edit `templates/goat_values.yml` in a fork) for multi-node setups that need `rwx` storage.

- **`goat-core` has no curl**: the image is stripped to essentials. Don't write tasks that `kubernetes.core.k8s_exec` shell-out into the container; rely on Kubernetes probes instead.

- **Ansible Jinja2 in dict keys**: if you ever need to add a third `preparedDatabases` entry with a name derived from a variable, note that Ansible does NOT evaluate Jinja inside YAML dict keys. Workaround in `01_db.yml`: a `set_fact` + a single Jinja dict-literal expression.

## License

[EUPL-1.2](LICENSE) — same as civitas-core.
