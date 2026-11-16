# Changelog

All notable changes to this add-on fork are documented here.
This is a maintained fork of the archived [`hassio-addons/addon-influxdb`](https://github.com/hassio-addons/addon-influxdb); fork baseline is upstream commit `43f4a6d` ("Mark add-on EOL").

## 5.0.13 - 2026-11-16

### Synced with upstream (up to `bed4a9e`)

- InfluxDB `1.13.1-1` → **`1.13.2-1`**
- Chronograf `1.11.6` → **`1.11.7`**
- Kapacitor `1.8.8-1` → **`1.8.9-1`**
- Base image `debian-base` `9.5.1` → **`9.5.2`**

## 5.0.12 - 2026-10-25

### Synced with upstream (first full sync, up to `87f33da`)

- Adopted the upstream restructure to **s6-rc services** and **`.gtpl` templated configurations** for nginx, InfluxDB and Kapacitor.
- New ports: `8089/udp` (InfluxDB UDP listener, enable via envvars) and `9092/tcp` (Kapacitor HTTP API).
- `nofile` ulimit raised to 65535.
- Base image `debian-base` `9.5.0` → **`9.5.1`**, Chronograf → **`1.11.6`**, Kapacitor → **`1.8.8-1`**, nginx pinned to `1.26.3-3+deb13u10`.

### Changed

- Fork identity preserved: slug `influxdb-vistalba`, own GHCR images, own OCI labels, and the fork's **self-contained CI/CD workflows** (kept automatically on upstream merges).

## 5.0.11 - 2026-09-28

### Changed

- Reverted to the v5.0.9 state: the first upstream-merge attempt (5.0.10) was abandoned.
- Documentation aligned with the bundled component versions.

## 5.0.9 - 2026-09-28

### Changed

- Base image `debian-base` `9.4.0` → **`9.5.0`** in `build.yaml` (the value the build actually uses).

## 5.0.8 - 2026-09-15

### Changed

- InfluxDB `1.12.4-1` → **`1.13.1-1`**.
- `config.yaml` version aligned with the release tag.

## 5.0.7 - 2026-09-04

### Changed

- README refinements (fork header, component table, migration guide wording).

## 5.0.6 - 2026-09-04

### Added

- README: fork header, component version table, armv7-drop note, and an InfluxDB data **migration guide** (moving data between add-on instances with an atomic rename).

## 5.0.5 - 2026-09-02

### Changed

- Unique add-on **slug `influxdb-vistalba`** so the fork can be installed alongside the original add-on.

## 5.0.4 - 2026-09-01

### Fixed

- Robust InfluxDB download URL: the release subdirectory is derived by stripping the build suffix (`%-*`), so only the version argument changes on a bump.

## 5.0.3 - 2026-09-01

### Removed

- **armv7 (32-bit) support dropped** — the `debian-base` 9.4.0 base image only ships `amd64` and `aarch64`.

## 5.0.2 and earlier

- Upstream history of the original `hassio-addons/addon-influxdb` (archived/EOL upstream). See the [original repository](https://github.com/hassio-addons/addon-influxdb) for releases up to and including upstream `43f4a6d`.
