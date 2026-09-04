# vistalba/addon-influxdb — maintained fork

This is a **maintained fork** of the archived [`hassio-addons/addon-influxdb`](https://github.com/hassio-addons/addon-influxdb).

## Current components

| Component | Version |
|---|---|
| InfluxDB | 1.12.4 |
| Chronograf | 1.11.4 |
| Kapacitor | 1.8.6-1 |
| Base image | `ghcr.io/hassio-addons/debian-base/{arch}:9.4.0` |

> **32-bit (`armv7`) support dropped:** the 9.4.0 base image only ships `amd64` and `aarch64`.

## Migrating InfluxDB Data Between Home Assistant OS Add-ons
Move InfluxDB 1.x data from an old add-on to a new one on Home Assistant OS
without extra disk space (uses an atomic rename, not a copy).

> **BACK UP FIRST - the move is destructive!**
> Step 3 uses an atomic `mv`, so the data is **removed from the old add-on**
> and there is **no fallback** if anything goes wrong. Before you start,
> create a full backup - and ideally **not only** a Home Assistant snapshot:
> - **Full system / VM image snapshot** (e.g. Proxmox, ESXi, or a disk image
>   of the SD card / SSD) so you can roll back the entire HA OS instance.
> - Or an off-box copy of `/mnt/data/supervisor/apps/data/<OLD_SLUG>` to
>   external storage.
>
> A Home Assistant snapshot alone may be insufficient (large InfluxDB data
> is often excluded or too big), so a VM/disk-image level backup is strongly
> recommended.

## Prerequisites

- Access via the **Advanced SSH & Web Terminal** add-on with **Protection Mode disabled**.
- Both add-ons installed. Note their slugs (folder names) under
  `/mnt/data/supervisor/apps/data/`.
- **A verified full backup / VM image (see warning above).**

## 1. Identify the container names and data paths

```bash
docker ps | grep -i influx

docker inspect <OLD_CONTAINER> --format '{{ range .Mounts }}{{ .Source }} -> {{ .Destination }}{{"\n"}}{{ end }}'
docker inspect <NEW_CONTAINER> --format '{{ range .Mounts }}{{ .Source }} -> {{ .Destination }}{{"\n"}}{{ end }}'
```

The data volume is mapped to `/data`, e.g.
`/mnt/data/supervisor/apps/data/<slug>`.

## 2. Stop BOTH add-ons

In the Home Assistant UI:
`Settings -> Add-ons -> <add-on> -> Stop`

Stopping flushes the WAL and closes all files, ensuring consistent data.
Note: On HA OS, stopping an add-on removes its container (so `docker cp`
won't work afterwards - that's why we use a helper container below).

## 3. Move the data (atomic rename, no extra space)

Both add-on folders live under the same partition, so a `mv` is an instant
rename. A helper Alpine container mounts the shared parent directory:

```bash
docker run --rm \
  -v /mnt/data/supervisor/apps/data:/apps \
  alpine sh -c "mv /apps/<OLD_SLUG>/influxdb \
                   /apps/<OLD_SLUG>/chronograf.db \
                   /apps/<OLD_SLUG>/kapacitor \
                   /apps/<OLD_SLUG>/secret \
                   /apps/<NEW_SLUG>/"
```

Do NOT move `options.json` (managed by each add-on).

## 4. Verify the move

```bash
docker run --rm \
  -v /mnt/data/supervisor/apps/data/<NEW_SLUG>:/dst \
  alpine sh -c "ls -la /dst /dst/influxdb"
```

Expected in `/dst/influxdb`: `data/`, `meta/`, `wal/`.

## 5. Start the new add-on

Start the new add-on in the HA UI, then check the logs:

```bash
docker logs --tail 30 <NEW_CONTAINER>
```

## 6. Reconfigure the connectors

Both add-ons use port `8086`, but the **hostname** and **credentials** may differ.
For add-on-to-add-on communication, the hostname is the add-on slug with hyphens
(e.g. `<NEW_SLUG_WITH_HYPHENS>`); alternatively use the HA host IP.

### Home Assistant -> InfluxDB (`configuration.yaml`)

```yaml
influxdb:
  host: <NEW_SLUG_WITH_HYPHENS>   # or HA host IP
  port: 8086
  database: homeassistant
  username: <USER>
  password: <PASS>
  ssl: false
  verify_ssl: false
```

Get `username` / `password` from the new add-on's Configuration tab.
Restart Home Assistant (or reload the InfluxDB integration) afterwards.

### Grafana -> InfluxDB (Data source)

`Connections -> Data sources -> <your InfluxDB source>`:

| Field           | Value                                        |
|-----------------|----------------------------------------------|
| URL             | `http://<NEW_SLUG_WITH_HYPHENS>:8086`        |
| Database        | `homeassistant`                              |
| User / Password | credentials from the new add-on              |
| HTTP Method     | GET or POST (as before)                      |

Click **Save & Test** - it must report "datasource is working".

## 7. Post-migration tests

### a) Databases and data present

```bash
# Add -username / -password if authentication is enabled
docker exec -it <NEW_CONTAINER> influx -username <USER> -password <PASS> \
  -execute "SHOW DATABASES"

docker exec -it <NEW_CONTAINER> influx -username <USER> -password <PASS> \
  -database homeassistant -execute "SHOW MEASUREMENTS" | head
```

### b) Continuous Queries migrated

```bash
docker exec -it <NEW_CONTAINER> influx -username <USER> -password <PASS> \
  -execute "SHOW CONTINUOUS QUERIES"
```

Confirm the CQ service is enabled:

```bash
docker exec -it <NEW_CONTAINER> sh -c "grep -A3 continuous /etc/influxdb/influxdb.conf"
# expected: enabled = true
```

Verify a CQ is actually firing (target should hold recent timestamps):

```bash
docker exec -it <NEW_CONTAINER> influx -username <USER> -password <PASS> \
  -database homeassistant \
  -execute 'SELECT * FROM "inf_5m"./.*/ ORDER BY time DESC LIMIT 5'
```

### c) Connector smoke tests

- Home Assistant: confirm new points arrive (check the logs, or run a query
  filtered to a recent time range).
- Grafana: open a dashboard and confirm panels render current data.

## 8. Clean up

Remove the old (deprecated) add-on only after all tests pass **and** you have
confirmed your backup is intact.

## Notes / Troubleshooting

- **Timezone error in Grafana** (`unable to find time zone CET`): Newer Debian
  images split legacy zones into the `tzdata-legacy` package, so `CET` is
  missing. Replace `tz('CET')` with `tz('Europe/Zurich')` in your queries
  (always included in standard `tzdata`), or set the timezone in Grafana's
  dashboard settings instead.

---

# Home Assistant Community Add-on: InfluxDB

[![GitHub Release][releases-shield]][releases]
![Project Stage][project-stage-shield]
[![License][license-shield]](LICENSE.md)

![Supports aarch64 Architecture][aarch64-shield]
![Supports amd64 Architecture][amd64-shield]
![Supports armhf Architecture][armhf-shield]
![Supports armv7 Architecture][armv7-shield]
![Supports i386 Architecture][i386-shield]

[![Github Actions][github-actions-shield]][github-actions]
![Project Maintenance][maintenance-shield]
[![GitHub Activity][commits-shield]][commits]

[![Discord][discord-shield]][discord]
[![Community Forum][forum-shield]][forum]

[![Sponsor Frenck via GitHub Sponsors][github-sponsors-shield]][github-sponsors]

[![Support Frenck on Patreon][patreon-shield]][patreon]

Scalable datastore for metrics, events, and real-time analytics.

## Deprecation warning

**This add-on is in a deprecated state!**

This add-on is built on InfluxDB 1.x, which InfluxData has end-of-lifed and
no longer supports. That makes this add-on obsolete; it will not receive any
updates anymore and has been removed from our add-on store.

## About

InfluxDB is an open source time series database optimized for high-write-volume.
It's useful for recording metrics, sensor data, events,
and performing analytics. It exposes an HTTP API for client interaction and is
often used in combination with Grafana to visualize the data.

![Chronograf in the Home Assistant Frontend](images/screenshot.png)

This add-on comes with Chronograf & Kapacitor pre-installed. These provide a
nice InfluxDB admin interface for managing your users, databases, data
retention settings, and let you peek inside the database using the Data
Explorer.

[:books: Read the full add-on documentation][docs]

## Support

Got questions?

You have several options to get them answered:

- The [Home Assistant Community Add-ons Discord chat server][discord] for add-on
  support and feature requests.
- The [Home Assistant Discord chat server][discord-ha] for general Home
  Assistant discussions and questions.
- The Home Assistant [Community Forum][forum].
- Join the [Reddit subreddit][reddit] in [/r/homeassistant][reddit]

You could also [open an issue here][issue] GitHub.

## Contributing

This is an active open-source project. We are always open to people who want to
use the code or contribute to it.

We have set up a separate document containing our
[contribution guidelines](.github/CONTRIBUTING.md).

Thank you for being involved! :heart_eyes:

## Authors & contributors

The original setup of this repository is by [Franck Nijhof][frenck].

For a full list of all authors and contributors,
check [the contributor's page][contributors].

## We have got some Home Assistant add-ons for you

Want some more functionality to your Home Assistant instance?

We have created multiple add-ons for Home Assistant. For a full list, check out
our [GitHub Repository][repository].

## License

MIT License

Copyright (c) 2018-2025 Franck Nijhof

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

[aarch64-shield]: https://img.shields.io/badge/aarch64-yes-green.svg
[amd64-shield]: https://img.shields.io/badge/amd64-yes-green.svg
[armhf-shield]: https://img.shields.io/badge/armhf-no-red.svg
[armv7-shield]: https://img.shields.io/badge/armv7-yes-green.svg
[commits-shield]: https://img.shields.io/github/commit-activity/y/hassio-addons/addon-influxdb.svg
[commits]: https://github.com/hassio-addons/addon-influxdb/commits/main
[contributors]: https://github.com/hassio-addons/addon-influxdb/graphs/contributors
[discord-ha]: https://discord.gg/c5DvZ4e
[discord-shield]: https://img.shields.io/discord/478094546522079232.svg
[discord]: https://discord.me/hassioaddons
[docs]: https://github.com/hassio-addons/addon-influxdb/blob/main/influxdb/DOCS.md
[forum-shield]: https://img.shields.io/badge/community-forum-brightgreen.svg
[forum]: https://community.home-assistant.io/t/home-assistant-community-add-on-influxdb/54491?u=frenck
[frenck]: https://github.com/frenck
[github-actions-shield]: https://github.com/hassio-addons/addon-influxdb/workflows/CI/badge.svg
[github-actions]: https://github.com/hassio-addons/addon-influxdb/actions
[github-sponsors-shield]: https://frenck.dev/wp-content/uploads/2019/12/github_sponsor.png
[github-sponsors]: https://github.com/sponsors/frenck
[i386-shield]: https://img.shields.io/badge/i386-no-red.svg
[issue]: https://github.com/hassio-addons/addon-influxdb/issues
[license-shield]: https://img.shields.io/github/license/hassio-addons/addon-influxdb.svg
[maintenance-shield]: https://img.shields.io/maintenance/yes/2025.svg
[patreon-shield]: https://frenck.dev/wp-content/uploads/2019/12/patreon.png
[patreon]: https://www.patreon.com/frenck
[project-stage-shield]: https://img.shields.io/badge/project%20stage-%20!%20DEPRECATED%20%20%20!-ff0000.svg
[reddit]: https://reddit.com/r/homeassistant
[releases-shield]: https://img.shields.io/github/release/hassio-addons/addon-influxdb.svg
[releases]: https://github.com/hassio-addons/addon-influxdb/releases
[repository]: https://github.com/hassio-addons/repository
