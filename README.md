# AstroLumina-Helm

Helm chart for the AstroLumina platform (frontend + astrology / booking /
payment APIs). One chart, one release per environment — the differences
between dev, staging and production live in the values files. Rendered
output is functionally identical to the former Kustomize manifests in
`AstroLumina-Kubernetes` (verified by an automated diff, see Validation).

## Layout

```text
charts/astrolumina/
  Chart.yaml
  values.yaml                  # shared service catalog (images, resources, env wiring)
  values-dev.yaml              # development (NodePort, no Traefik)
  values-staging.yaml          # staging (blue/green, HTTP Traefik)
  values-production.yaml       # production (blue/green, HTTPS + dashboard auth)
  templates/
    namespace.yaml             # Namespace
    ghcr-secret.yaml           # ghcr placeholder (real secret is imperative, see below)
    placeholder-secrets.yaml   # first-boot values until Doppler syncs
    doppler-secrets.yaml       # DopplerSecret sync wiring
    deployments.yaml           # one per service (x blue/green on staging/prod)
    services.yaml              # NodePorts on dev, *-live ClusterIP on staging/prod
    hpa.yaml                   # 70
    networkpolicy.yaml         # 60 (permissive on dev, strict otherwise)
    middlewares.yaml           # 51 (staging/prod only)
    ingressroute.yaml          # 52 (staging/prod only)
    NOTES.txt                  # post-install hints
```

## Usage

One release per environment, each with its own values file:

```bash
helm install astrolumina-dev ./charts/astrolumina \
  -f charts/astrolumina/values-dev.yaml

helm install astrolumina-staging ./charts/astrolumina \
  -f charts/astrolumina/values-staging.yaml

helm install astrolumina-prod ./charts/astrolumina \
  -f charts/astrolumina/values-production.yaml
```

Dry-run against the old manifests any time:

```bash
helm template astrolumina-dev ./charts/astrolumina \
  -f charts/astrolumina/values-dev.yaml
```

Blue-green switch (staging/production -- flips one `*-live` selector,
exactly like editing `53-live-services.yaml` by hand; repeat per service,
each app flips independently):

```bash
helm upgrade astrolumina-staging ./charts/astrolumina \
  -f charts/astrolumina/values-staging.yaml \
  --set blueGreen.liveColors.astrology-api=green
```

Image tags are pinned per color in `values-staging.yaml` /
`values-production.yaml` (`imageTags.blue` / `imageTags.green` — source of
truth in git, mirrors the Compose `versions.env` files, Doppler no longer
carries any `*_DOCKER_IMAGE_TAG` keys). Dev falls back to `image.tag`
(`latest`). To promote a build, bump the tags in the values file and upgrade
(still overridable per run with `--set`):

```bash
helm upgrade astrolumina-prod ./charts/astrolumina \
  -f charts/astrolumina/values-production.yaml \
  --set imageTags.blue.frontend=<new-tag> --set imageTags.green.frontend=<new-tag>
```

The same bumps go through the manual `.github/workflows/deploy.yml` (env
choice + per-service versions, branch + PR): staging/production edit the
idle `imageTags` entries (live read from `blueGreen.liveColors`, never
flipped by the workflow); dev writes the per-service `devTags` entries
(one version per service, no blue/green on dev).

## Chart releases (OCI registry)

Every merge to main touching `charts/**` packages the chart and pushes it to
GitHub Container Registry as an OCI artifact
(`oci://ghcr.io/astrolumina-by-carmen-ilie/charts/astrolumina`) via
`.github/workflows/release-chart.yml` — GHCR *is* the Helm repository here,
no extra server needed (OCI references are lowercase, hence the lowercase
org name). Pull without cloning the repo:

```bash
helm pull oci://ghcr.io/astrolumina-by-carmen-ilie/charts/astrolumina --version 1.0.0
```

Release discipline: bump `version:` in `charts/astrolumina/Chart.yaml` to
cut a release. Re-pushing an existing version fails (OCI tags are
immutable) — bump instead of retrying. `appVersion` tracks the default app
images; the actual pins stay in the values files, not in the release.
ArgoCD keeps consuming the git path (`charts/astrolumina` + values files);
the OCI artifact is for portability (other clusters, `helm pull`
inspection), not a second source of truth.

## First boot on an empty cluster (3 phases, same as before)

1. Install the release — pods boot on the placeholder Secrets.
2. Create the one-time token Secret and let the Doppler operator sync
   real values (token name per env: `doppler-token-dev` / `-stg` / `-prd`;
   see `NOTES.txt` output after install for the exact commands).
3. After the sync is verified, phase the placeholders out without
   uninstalling:

```bash
helm upgrade <release> ./charts/astrolumina -f charts/astrolumina/<values-file> \
  --set placeholderSecretsEnabled=false
```

## NOT in this chart (imperative, once per cluster rebuild)

- The `doppler-token-*` ServiceToken Secret in `doppler-operator-system`.
- The real `ghcr-secret` docker-registry Secret (chart ships a placeholder
  only — a pull secret cannot be synced by the Doppler operator).
- The `traefik-dashboard-auth` Secret in `astrolumina-prod` (htpasswd
  `users` line for the dashboard basicAuth middleware).
- The RKE2 `HelmChartConfig` enabling Traefik + the `letsencrypt`
  certResolver (production prerequisite).

## Deliberate normalizations vs the Kustomize originals

- Placeholder Secrets and Deployments use `app.kubernetes.io/component`
  everywhere; the old dev Secrets used `app.kubernetes.io/name` instead.
- Rendered `stringData` values are always quoted; parsed values are equal.
- The stale `# 3 replicas` comment on staging Deployments (which actually
  run 1 replica) was dropped; replicas come from values.

## Environment variable scheme

Mirrors the application repos (the APIs validate their full env set at
startup via zod and exit 1 if anything is missing) and the central
`AstroLumina-DockerCompose` repo. 100% of variables come from Doppler —
there are no ConfigMaps.

- **App-specific variables first, shared endpoint pairs last.** Every
  service lists its own variables first (`NODE_ENV`, its `*_SERVER_PORT`,
  `*_SENTRY_DSN`, plus its own keys: astrologer/CalCom/Stripe/`_API_URL`),
  then the 16 shared `DC`/`K8S` endpoint pairs in canonical order
  (`ASTROLOGY` / `BOOKING` / `PAYMENT` / `FRONTEND` × `DC_PORT`, `DC_DNS`,
  `K8S_PORT`, `K8S_DNS`). The APIs build their CORS origins from these
  pairs; the browser never talks to them directly.
- **Frontend URL mapping (K8s logic).** The frontend image reads the
  app-facing names `ASTROLOGY_API_URL` / `PAYMENT_API_URL` /
  `BOOKING_API_URL` (injected into `public/env.js` by the entrypoint —
  `VITE_*` names are NOT read). In Kubernetes these take the K8s-side
  values, so each Deployment wires `name: *_API_URL` from
  `secretKeyRef key: *_API_K8S_URL` (in Compose they take the `_API_DC_URL`
  values instead). The unreferenced plain `_API_URL` / `_API_DC_URL` keys
  are NOT synced — only the 24 / 23 / 27 / 28 keys each service actually
  reads (frontend / astrology / booking / payment).
- **Service token.** The Doppler operator authenticates with the
  `DOPPLER_SERVICE_TOKEN` value (one token per Doppler config:
  `dev` / `stg` / `prd`), stored in the imperative
  `doppler-token-dev` / `-stg` / `-prd` Secrets referenced by
  `doppler.tokenSecret`. It is never committed to git.

## Validation

`helm lint` passes for all three values files, and every rendered object
was compared field-by-field against the original manifests: resource
inventory, replicas, images, full `secretKeyRef` env wiring (name/key/
optional), probes, resources, affinity, selectors, labels, service ports,
HPA bounds/metrics/targets, Doppler sync key sets, placeholder values,
NetworkPolicy, middlewares and IngressRoutes — all equal.
