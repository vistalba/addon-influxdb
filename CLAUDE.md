# CLAUDE.md — InfluxDB Home Assistant Add-on

Project context for AI developer agents working in this repository.

## What this repository is

- A **Home Assistant community add-on** for **InfluxDB** (a time-series database), plus **Chronograf** (admin UI) and **Kapacitor** (stream processing).
- This is a **maintained successor/fork** of the archived `hassio-addons/addon-influxdb` (the original org is archived; InfluxDB 1.x is end-of-lifed by InfluxData).
- The add-on bundles: InfluxDB `1.12.4`, Chronograf `1.11.4`, Kapacitor `1.8.6-1`.

## Repository layout

```
addon-influxdb/
├── CLAUDE.md                  # this file
├── repository.json            # HA add-on store metadata (makes the repo an installable HA repository)
├── influxdb/                  # the add-on definition + build context
│   ├── Dockerfile             # builds InfluxDB + Chronograf + Kapacitor
│   ├── build.yaml            # per-arch base image (debian-base)
│   ├── config.yaml           # HA add-on manifest (name, version, slug, image, arch, ports, options)
│   ├── rootfs/              # files copied into the container (services.d, nginx, cont-init.d, etc.)
│   ├── DOCS.md
│   ├── icon.png / logo.png
│   └── .README.j2
├── .github/workflows/
│   ├── ci.yaml              # build-only CI (no push)
│   ├── deploy.yaml          # build + push to GHCR + multi-arch manifest
│   ├── release-drafter.yaml, pr-labels.yaml, labels.yaml, lock.yaml, stale.yaml
└── images/
```

## Build system

- **Base image:** `ghcr.io/hassio-addons/debian-base/{arch}`. Latest is **9.4.0**, which is published **only for `amd64` and `aarch64`** (no armv7). Declared per-arch in `influxdb/build.yaml`.
- **Dockerfile** (`influxdb/Dockerfile`):
  - Installs `libnginx-mod-http-lua`, `luarocks`, `nginx`, `procps` (unpinned — pinned versions broke the build on the new base image).
  - Downloads + installs the three `.deb` packages from `dl.influxdata.com`.
  - `COPY rootfs /` copies the service scripts.
  - Sets `io.hass.*` and OCI image labels from build args.
- **Supported architectures:** `amd64`, `aarch64`. **armv7 (32-bit) was dropped** because the 9.4.0 base image no longer ships an armv7 variant.

## CI/CD workflows (self-contained)

The workflows were rewritten to be **self-contained** — they no longer call the archived `hassio-addons/workflows` shared workflows.

- **`ci.yaml` (CI):**
  - Triggers on `push`, `pull_request`, `workflow_dispatch`.
  - Builds each arch in a matrix (`amd64`, `aarch64`) with `docker/build-push-action` and `push: false` (build-only validation).
  - `aarch64` runs on `ubuntu-24.04-arm`; others on `ubuntu-latest`.
  - Reads the base image per-arch from `influxdb/build.yaml` via `yq`.
- **`deploy.yaml` (Deploy):**
  - Triggers on `release: published` **or** after a successful CI run on `main` (`workflow_run`, gated on `conclusion == 'success'`).
  - Builds **and pushes** per-arch images to `ghcr.io/vistalba/influxdb/{arch}:{version}` + `:latest`, then creates a **multi-arch manifest** (`ghcr.io/vistalba/influxdb:{version}`).
  - Version = release tag (with `v` stripped) when triggered by a release; otherwise the short commit SHA.
  - **Registry auth:** uses the `REGISTRY_TOKEN` repo secret (a PAT with `packages:write`), falling back to `GITHUB_TOKEN` if the secret is absent.

## Releasing process (version branch)

The team's convention: **version bumps happen on a dedicated version branch** (e.g. `v5.0.3`), then merged to `main`, then a GitHub Release is published.

1. Create a version branch `vX.Y.Z` from `main`.
2. On that branch: bump `influxdb/config.yaml` `version:` to `X.Y.Z` (and any base-image / feature changes).
3. Push the branch; CI validates the build.
4. **Merge the branch into `main`** (fast-forward when possible).
5. **Create a GitHub Release** with tag `vX.Y.Z`. Publishing it triggers the Deploy workflow, which builds + pushes `ghcr.io/vistalba/influxdb/{arch}:X.Y.Z` + the multi-arch manifest.

## Home Assistant repository integration

- `repository.json` (repo root) makes this repo a valid **HA community add-on repository**. Its `url` must point to this repo.
- `influxdb/config.yaml` carries the critical field **`image: ghcr.io/vistalba/influxdb/{arch}`** — HA substitutes the user's architecture and pulls that image.
- Users add this repo in HA: **Settings → Add-ons & Integrations → Add-on Store → gear icon → Repositories → add** `https://github.com/vistalba/addon-influxdb`.
- The `config.yaml` `version` must match the GHCR image tag (keep them in lockstep when releasing).

## Gotchas / known issues

- **GHCR package visibility:** the `vistalba/influxdb` package was created via a PAT and defaults to **private**. It must be set to **Public** (GitHub → Packages → package → Settings → Visibility: Public), otherwise HA users can't pull it without auth and it won't appear in the public packages list.
- **Pinned apt versions** in the Dockerfile caused build failures on the newer base image; the fix was to install packages **unpinned**.
- **armv7 dropped** because base image 9.4.0 only ships amd64/aarch64.
- **InfluxDB 1.x is EOL** — a future major change would be migrating to InfluxDB 2.x (different config, different ports).
- **InfluxDB download URL changed in 1.11+**: newer InfluxDB `.deb`s live under a `v{base}/` subdirectory, where the base version has no build suffix while the filename carries the `-1` suffix. The Dockerfile keeps the `-1` in the `INFLUXDB_VERSION` arg (like Kapacitor's `1.8.6-1`) and derives the subdirectory by stripping it: `https://dl.influxdata.com/influxdb/releases/v${INFLUXDB_VERSION%-1}/influxdb_${INFLUXDB_VERSION}_${ARCH}.deb`. This way only the version arg changes on a bump. Chronograf and Kapacitor still use the flat `/{tool}/releases/{tool}_{ver}_{arch}.deb` layout.
- **InfluxDB 1.13.0** is listed in InfluxData's release notes, but its `.deb` was not downloadable (404) at build time; the latest available 1.x is `1.12.4`.
- **Node.js 20 deprecation warnings** in Actions are non-blocking (actions run on Node 24).

## Useful commands

```bash
# Local smoke test (outside HA) — provide the env vars + volume HA would supply:
mkdir -p ./influxdb-data
docker run --rm -it \
  --add-host=supervisor:127.0.0.1 \
  -e SUPERVISOR_TOKEN=dummy \
  -e INFLUXDB_AUTH=false \
  -v "$PWD/influxdb-data:/data" \
  -p 8086:8086 \
  ghcr.io/vistalba/influxdb:latest
```

```bash
# Git: create + push a version branch
git checkout -b v5.0.4 main
# ...bump config.yaml version, push...
git push -u origin v5.0.4

# Merge a version branch into main (fast-forward)
git checkout main
git merge --ff-only v5.0.4
git push origin main
```

## Current state (as of v5.0.4)

- `main` tip: `d74be6d` (CLAUDE.md added), then the `v5.0.4` component bump (InfluxDB 1.12.4, Chronograf 1.11.4, Kapacitor 1.8.6-1) is merged to `main`.
- `v5.0.4` release published; the Deploy workflow pushed `ghcr.io/vistalba/influxdb/{amd64,aarch64}:5.0.4` + `:latest` + multi-arch manifest.
- Open item: set the GHCR package to **Public** so HA users can pull it.
