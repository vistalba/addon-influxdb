# CLAUDE.md — InfluxDB Home Assistant Add-on

Project context for AI developer agents working in this repository.

## What this repository is

- A **Home Assistant community add-on** for **InfluxDB** (a time-series database), plus **Chronograf** (admin UI) and **Kapacitor** (stream processing).
- This is a **maintained successor/fork** of the archived `hassio-addons/addon-influxdb` (the original org is archived; InfluxDB 1.x is end-of-lifed by InfluxData).
- The add-on bundles: InfluxDB `1.13.2-1`, Chronograf `1.11.7`, Kapacitor `1.8.9-1`.

## Repository layout

```
addon-influxdb/
├── CLAUDE.md                  # this file
├── repository.json            # HA add-on store metadata (makes the repo an installable HA repository)
├── influxdb/                  # the add-on definition + build context
│   ├── Dockerfile             # builds InfluxDB + Chronograf + Kapacitor
│   ├── build.yaml            # per-arch base image (debian-base)
│   ├── config.yaml           # HA add-on manifest (name, version, slug, image, arch, ports, options)
│   ├── rootfs/              # files copied into the container (s6-rc.d services, templated nginx/influxdb/kapacitor configs via .gtpl)
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

- **Base image:** `ghcr.io/hassio-addons/debian-base` (multi-arch tag). Latest is **9.5.2**, which is published **only for `amd64` and `aarch64`** (no armv7). Declared per-arch in `influxdb/build.yaml`.
- **Dockerfile** (`influxdb/Dockerfile`):
  - Installs pinned `nginx` only (configs are now rendered from `.gtpl` templates at startup via s6-overlay; no lua/luarocks anymore).
  - Downloads + installs the three `.deb` packages from `dl.influxdata.com`.
  - `COPY rootfs /` copies the s6-rc service scripts and templates.
  - Sets `io.hass.*` and OCI image labels from build args (fork maintainer/vendor/url labels kept).
- **Supported architectures:** `amd64`, `aarch64`. **armv7 (32-bit) was dropped** because the 9.5.x base image no longer ships an armv7 variant.

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

## Staying in sync with upstream

The upstream repo is `hassio-addons/app-influxdb`, configured as the `upstream` git remote. `main` carries upstream ancestry (merge commits), so syncing is a plain merge — **no PR against upstream needed**.

### One-time setup per clone (REQUIRED)

The workflow protection relies on a local git config value that does **not** travel with the clone:

```bash
git config merge.ours.driver true
```

Without it, `.gitattributes` (`merge=ours` for `.github/workflows/**`) is inert and workflow files will conflict on every merge.

### Sync routine (run whenever upstream has new commits)

```bash
git fetch upstream
git checkout main
git merge upstream/main     # workflows auto-kept via merge=ours
git push origin main
```

Then: CI validates the merge → cut a `vX.Y.Z` release branch per the releasing convention above.

### What is protected / what to watch

- **Auto-kept (no conflict):** any file under `.github/workflows/` that both sides modified — our self-contained workflows always win.
- **Not protected — review after every merge:**
  - Upstream *adds* a new workflow file → it lands in the repo; delete it in the merge commit if we don't want it.
  - Upstream *deletes* a workflow we never modified → it is deleted silently; restore with `git checkout HEAD~1 -- <file>` if needed.
  - Non-workflow conflicts (e.g. `influxdb/Dockerfile`, `influxdb/config.yaml`, `README.md`, `influxdb/build.yaml`) must be resolved by hand.
- **When resolving conflicts, always keep the fork identity:**
  - `config.yaml`: `slug: influxdb-vistalba`, `image: ghcr.io/vistalba/influxdb/{arch}`, `url: https://github.com/vistalba/addon-influxdb`, our `version:`
  - `Dockerfile`: vistalba `maintainer`/`vendor`/`authors`/`url` OCI labels, `INFLUXDB_VERSION` build-suffix form (`1.13.1-1` + `%-*` URL trick), `KAPACITOR_VERSION` `-1` suffix
  - `README.md`: the fork header block ("maintained fork", component table, armv7 note, migration guide) on top of upstream content
  - Take upstream's side for everything else (rootfs/s6-rc layout, `.gtpl` templates, DOCS.md, base image and component version bumps).
- **After merging:** update the component versions in the `README.md` table and in this file ("What this repository is" + "Current state") to match the `Dockerfile`/`build.yaml`.
- Reference resolutions: full sync merge `31912fe` (upstream up to `87f33da`, v5.0.12) and version-bump merge to `bed4a9e` (v5.0.13); earlier attempts are on the `merge-upstream` / `sync-upstream` branches.

## Home Assistant repository integration

- `repository.json` (repo root) makes this repo a valid **HA community add-on repository**. Its `url` must point to this repo.
- `influxdb/config.yaml` carries the critical field **`image: ghcr.io/vistalba/influxdb/{arch}`** — HA substitutes the user's architecture and pulls that image.
- Users add this repo in HA: **Settings → Add-ons & Integrations → Add-on Store → gear icon → Repositories → add** `https://github.com/vistalba/addon-influxdb`.
- The `config.yaml` `version` must match the GHCR image tag (keep them in lockstep when releasing).

## Gotchas / known issues

- **GHCR package visibility:** the `vistalba/influxdb` package was created via a PAT and defaults to **private**. It must be set to **Public** (GitHub → Packages → package → Settings → Visibility: Public), otherwise HA users can't pull it without auth and it won't appear in the public packages list.
- **armv7 dropped** because base image 9.5.x only ships amd64/aarch64.
- **InfluxDB 1.x is EOL** — a future major change would be migrating to InfluxDB 2.x (different config, different ports).
- **InfluxDB download URL changed in 1.11+**: newer InfluxDB `.deb`s live under a `v{base}/` subdirectory, where the base version has no build suffix while the filename carries the build suffix (`-1`, `-2`, `-3`, …). The Dockerfile keeps the build suffix in the `INFLUXDB_VERSION` arg (like Kapacitor's `1.8.6-1`) and derives the subdirectory by stripping everything from the last hyphen: `https://dl.influxdata.com/influxdb/releases/v${INFLUXDB_VERSION%-*}/influxdb_${INFLUXDB_VERSION}_${ARCH}.deb`. The `%-*` strips any trailing build suffix (`-1`, `-2`, `-3`, …), so only the version arg changes on a bump. Chronograf and Kapacitor still use the flat `/{tool}/releases/{tool}_{ver}_{arch}.deb` layout.
- **InfluxDB 1.13.0** was listed in InfluxData's release notes but its `.deb` was a 404 at the time; the add-on now ships `1.13.1-1`, which downloads fine via the `v1.13.1/` subdirectory pattern above.
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

## Current state (as of v5.0.13)

- `main` is synced with **`upstream/main` up to `bed4a9e`** (hassio-addons/app-influxdb). The v5.0.12 merge adopted the upstream restructure to **s6-rc services + `.gtpl` templated configs** (nginx/influxdb/kapacitor), new ports (8089/udp, 9092/tcp), and ulimits; v5.0.13 is a routine upstream version-bump merge.
- Fork identity preserved: `slug: influxdb-vistalba`, `image: ghcr.io/vistalba/influxdb/{arch}`, vistalba OCI labels, fork README header + migration guide, and **all self-contained `.github/workflows` files** (auto-kept via `merge=ours`).
- Bundled components: InfluxDB `1.13.2-1`, Chronograf `1.11.7`, Kapacitor `1.8.9-1`, base image `debian-base:9.5.2`, nginx `1.26.3-3+deb13u10`.
- The `ghcr.io/vistalba/influxdb` package is set to **Public**.
