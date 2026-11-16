# Home Assistant Community App: InfluxDB v1

> **ℹ️ Maintained fork.** This is a maintained fork of
> [`hassio-addons/app-influxdb`](https://github.com/hassio-addons/app-influxdb)
> (formerly `hassio-addons/addon-influxdb`). Upstream was archived and marked
> EOL when this fork was created, but has since been revived and is actively
> maintained again; InfluxDB 1.x itself remains end-of-lifed by InfluxData.
> Current components: InfluxDB `1.13.2-1`, Chronograf `1.11.7`,
> Kapacitor `1.8.9-1`, base image `ghcr.io/hassio-addons/debian-base:9.5.2`.
> 32-bit (`armv7`) is **not supported** — the base image ships `amd64` and
> `aarch64` only. Release notes are shown in the **Changelog** tab of this
> add-on's dialog.

InfluxDB is an open source time series database optimized for high-write-volume.
It's useful for recording metrics, sensor data, events,
and performing analytics. It exposes an HTTP API for client interaction and is
often used in combination with Grafana to visualize the data.

This app provides the InfluxDB **1.x** series (InfluxQL, the 1.x HTTP API and
the classic database, user and retention policy model). It is not InfluxDB 2
or InfluxDB 3; those have a different API, data model and configuration. If
you are integrating with Home Assistant, use the `influxdb` integration with
`api_version: 1`, which is the default.

This app comes with Chronograf and Kapacitor pre-installed, which gives you
a nice InfluxDB admin interface for managing your users, databases and data
retention settings, and lets you peek inside the database using the Data
Explorer.

## Installation

The installation of this app is pretty straightforward and not different in
comparison to installing any other Home Assistant app.

1. Click the Home Assistant My button below to open the app on your Home
   Assistant instance.

   [![Open this app in your Home Assistant instance.][addon-badge]][addon]

1. Click the "Install" button to install the app.
1. Start the "InfluxDB v1" app.
1. Check the logs of the "InfluxDB v1" app to see if everything went well.
1. Click the "OPEN WEB UI" button!

## Configuration

**Note**: _Remember to restart the app when the configuration is changed._

Example app configuration:

```yaml
log_level: info
auth: true
reporting: false
ssl: true
certfile: fullchain.pem
keyfile: privkey.pem
envvars:
  - name: INFLUXDB_HTTP_LOG_ENABLED
    value: "true"
```

**Note**: _This is just an example, don't copy and paste it! Create your own!_

### Option: `log_level`

The `log_level` option controls the level of log output by the app and can
be changed to be more or less verbose, which might be useful when you are
dealing with an unknown issue. Possible values are:

- `trace`: Show every detail, like all called internal functions.
- `debug`: Shows detailed debug information.
- `info`: Normal (usually) interesting events.
- `warning`: Exceptional occurrences that are not errors.
- `error`: Runtime errors that do not require immediate action.
- `fatal`: Something went terribly wrong. App becomes unusable.

Please note that each level automatically includes log messages from a
more severe level, e.g., `debug` also shows `info` messages. By default,
the `log_level` is set to `info`, which is the recommended setting unless
you are troubleshooting.

### Option: `auth`

Enable or disable InfluxDB user authentication.

**Note**: _Turning this off is NOT recommended!_

### Option: `reporting`

Enables or disables the reporting of anonymous usage data by InfluxDB,
Chronograf and Kapacitor to InfluxData. It is disabled by default.

**Note**: _No data from user databases is ever transmitted!_

### Option: `ssl`

Enables/Disables SSL (HTTPS) on the Chronograf web interface when it is
exposed directly on port 80. Set it `true` to enable it, `false` otherwise.

**Note**: _This does NOT activate SSL for the InfluxDB API on port 8086, just
the web interface. See the known issues and limitations section below._

### Option: `certfile`

The certificate file to use for SSL.

**Note**: _The file MUST be stored in `/ssl/`, which is the default_

### Option: `keyfile`

The private key file to use for SSL.

**Note**: _The file MUST be stored in `/ssl/`, which is the default_

### Option: `envvars`

This allows the setting of Environment Variables to control InfluxDB
configuration as documented at:

<https://docs.influxdata.com/influxdb/v1/administration/config/>

**Note**: _Changing these options can possibly cause issues with your instance.
USE AT YOUR OWN RISK!_

These are case sensitive.

#### Sub-option: `name`

The name of the environment variable to set which must start with `INFLUXDB_`

#### Sub-option: `value`

The value of the environment variable to set, see the InfluxDB documentation
for full details. Values should always be entered as a string (even true/false
values).

For example, to enable the InfluxDB UDP listener (exposed on port 8089 in the
network settings of the app):

```yaml
envvars:
  - name: INFLUXDB_UDP_0_ENABLED
    value: "true"
  - name: INFLUXDB_UDP_0_BIND_ADDRESS
    value: ":8089"
  - name: INFLUXDB_UDP_0_DATABASE
    value: "udp"
```

### Option: `leave_front_door_open`

Adding this option to the app configuration allows you to disable the Home
Assistant authentication on the Chronograf web interface, when it is exposed
directly on port 80, by setting it to `true`.

**Note**: _We STRONGLY suggest, not to use this, even if this app is
only exposed to your internal network. USE AT YOUR OWN RISK!_

## Integrating into Home Assistant

The `influxdb` integration of Home Assistant makes it possible to transfer all
state changes to an InfluxDB database.

You need to do the following steps in order to get this working:

- Click on "OPEN WEB UI" to open the admin web-interface provided by this app.
- On the left menu click on the "InfluxDB Admin".
- Create a database for storing Home Assistant's data in, e.g., `homeassistant`.
- Go to the users tab and create a user for Home Assistant,
  e.g., `homeassistant`.
- Add "ALL" to "Permissions" of the created user, to allow writing to your
  database.

Now we've got this in place, add the following snippet to your Home Assistant
`configuration.yaml` file.

```yaml
influxdb:
  api_version: 1
  host: a0d7b954-influxdb
  port: 8086
  database: homeassistant
  username: homeassistant
  password: <yourpassword>
  max_retries: 3
  default_measurement: state
```

Restart Home Assistant.

You should now see the data flowing into InfluxDB by visiting the web-interface
and using the Data Explorer.

Full details of the Home Assistant integration can be found here:

<https://www.home-assistant.io/integrations/influxdb/>

## Network ports

Besides the Chronograf web interface (port 80, not needed when using
Ingress), the app can expose the following ports in its network settings:

- `8086/tcp`: The InfluxDB HTTP API. Used by the Home Assistant `influxdb`
  integration, Grafana and other clients.
- `8088/tcp`: The InfluxDB RPC service, used by the `influxd backup` and
  `influxd restore` tools.
- `8089/udp`: The InfluxDB UDP listener, disabled by default. See the
  `envvars` option above on how to enable it.
- `9092/tcp`: The Kapacitor HTTP API, for managing Kapacitor tasks from
  outside of Home Assistant. Kapacitor only listens on this port for external
  connections when it is exposed here.

**Note**: _Kapacitor does not have authentication, only expose its API on a
trusted network._

## Backup and restore

The InfluxDB databases, the Chronograf settings and the Kapacitor data are
all stored in the data directory of the app, so a regular Home Assistant
backup of this app contains everything.

For a database level backup, the app exposes the InfluxDB backup and restore
RPC service on port 8088 (disable the port if you do not need it), which can
be used with the `influxd backup` and `influxd restore` tools from another
machine. The `/share` folder of Home Assistant is available inside the app as
well, for example to import or export data using the `influx` CLI.

Full details on backing up and restoring InfluxDB 1.x can be found here:

<https://docs.influxdata.com/influxdb/v1/administration/backup_and_restore/>

## Migrating InfluxDB data between add-ons

Move InfluxDB 1.x data from an old add-on to a new one on Home Assistant OS
without extra disk space (uses an atomic rename, not a copy).

> **⚠️ BACK UP FIRST — the move is destructive!** Step 3 uses an atomic `mv`,
> so the data is **removed from the old add-on** and there is **no fallback** if
> anything goes wrong. Before you start, create a full backup — and ideally
> **not only** a Home Assistant snapshot:
>
> - A **full system / VM image snapshot** (e.g. Proxmox, ESXi, or a disk image
>   of the SD card / SSD) so you can roll back the entire HA OS instance.
> - Or an off-box copy of `/mnt/data/supervisor/apps/data/<OLD_SLUG>` to
>   external storage.
>
> A Home Assistant snapshot alone may be insufficient (large InfluxDB data is
> often excluded or too big), so a VM/disk-image level backup is strongly
> recommended.

### Prerequisites

- Access via the **Advanced SSH & Web Terminal** add-on with **Protection Mode
  disabled**.
- Both add-ons installed. Note their slugs (folder names) under
  `/mnt/data/supervisor/apps/data/`.
- **A verified full backup / VM image (see warning above).**

### 1. Identify the container names and data paths

```bash
docker ps | grep -i influx

docker inspect <OLD_CONTAINER> --format '{{ range .Mounts }}{{ .Source }} -> {{ .Destination }}{{"\n"}}{{ end }}'
docker inspect <NEW_CONTAINER> --format '{{ range .Mounts }}{{ .Source }} -> {{ .Destination }}{{"\n"}}{{ end }}'
```

The data volume is mapped to `/data`, e.g.
`/mnt/data/supervisor/apps/data/<slug>`.

### 2. Stop BOTH add-ons

In the Home Assistant UI:
`Settings -> Add-ons -> <add-on> -> Stop`

Stopping flushes the WAL and closes all files, ensuring consistent data.
Note: On HA OS, stopping an add-on removes its container (so `docker cp`
won't work afterwards - that's why we use a helper container below).

### 3. Move the data (atomic rename, no extra space)

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

### 4. Verify the move

```bash
docker run --rm \
  -v /mnt/data/supervisor/apps/data/<NEW_SLUG>:/dst \
  alpine sh -c "ls -la /dst /dst/influxdb"
```

Expected in `/dst/influxdb`: `data/`, `meta/`, `wal/`.

### 5. Start the new add-on

Start the new add-on in the HA UI, then check the logs:

```bash
docker logs --tail 30 <NEW_CONTAINER>
```

### 6. Reconfigure the connectors

Both add-ons use port `8086`, but the **hostname** and **credentials** may
differ. In Home Assistant, add-ons are reachable by their **slug** as a
hostname (slugs contain hyphens, e.g. `influxdb-vistalba`). Use `<NEW_SLUG>`
(the new add-on's slug) as the hostname, or use the HA host IP instead.

#### Home Assistant -> InfluxDB (`configuration.yaml`)

```yaml
influxdb:
  host: <NEW_SLUG>   # or HA host IP
  port: 8086
  database: homeassistant
  username: <USER>
  password: <PASS>
  ssl: false
  verify_ssl: false
```

Get `username` / `password` from the new add-on's Configuration tab.
Restart Home Assistant (or reload the InfluxDB integration) afterwards.

#### Grafana -> InfluxDB (Data source)

`Connections -> Data sources -> <your InfluxDB source>`:

| Field           | Value                                        |
|-----------------|----------------------------------------------|
| URL             | `http://<NEW_SLUG>:8086`        |
| Database        | `homeassistant`                              |
| User / Password | credentials from the new add-on              |
| HTTP Method     | GET or POST (as before)                      |

Click **Save & Test** - it must report "datasource is working".

### 7. Post-migration tests

#### a) Databases and data present

```bash
# Add -username / -password if authentication is enabled
docker exec -it <NEW_CONTAINER> influx -username <USER> -password <PASS> \
  -execute "SHOW DATABASES"

docker exec -it <NEW_CONTAINER> influx -username <USER> -password <PASS> \
  -database homeassistant -execute "SHOW MEASUREMENTS" | head
```

#### b) Continuous Queries migrated

```bash
docker exec -it <NEW_CONTAINER> influx -username <USER> -password <PASS> \
  -execute "SHOW CONTINUOUS QUERIES"
```

Confirm the CQ service is enabled:

```bash
docker exec -it <NEW_CONTAINER> sh -c "grep -A3 continuous /etc/influxdb/influxdb.conf"
# expected: enabled = true
```

Verify a CQ is actually firing (the CQ target should hold recent timestamps).
Replace `<CQ_TARGET>` with your actual continuous-query target name (list them
with `SHOW CONTINUOUS QUERIES` above):

```bash
docker exec -it <NEW_CONTAINER> influx -username <USER> -password <PASS> \
  -database homeassistant \
  -execute 'SELECT * FROM "<CQ_TARGET>"./.*/ ORDER BY time DESC LIMIT 5'
```

#### c) Connector smoke tests

- Home Assistant: confirm new points arrive (check the logs, or run a query
  filtered to a recent time range).
- Grafana: open a dashboard and confirm panels render current data.

### 8. Clean up

Remove the old (deprecated) add-on only after all tests pass **and** you have
confirmed your backup is intact.

### Notes / Troubleshooting

- **Timezone error in Grafana** (`unable to find time zone CET`): Newer Debian
  images split legacy zones into the `tzdata-legacy` package, so `CET` is
  missing. Replace `tz('CET')` with `tz('Europe/Zurich')` in your queries
  (always included in standard `tzdata`), or set the timezone in Grafana's
  dashboard settings instead.

## Known issues and limitations

- While the Chronograf interface supports SSL, the app does not support
  enabling SSL on the InfluxDB API itself. This limitation is caused by
  Chronograf, and we are still looking into a proper solution for this.

## Changelog & Releases

The changelog is shown in the **Changelog** tab of this add-on's dialog
(maintained in [`changelog.md`](changelog.md) in the repository). Release
notes are also published via [GitHub's releases][releases] functionality.

Releases are based on [Semantic Versioning][semver], and use the format
of `MAJOR.MINOR.PATCH`. In a nutshell, the version will be incremented
based on the following:

- `MAJOR`: Incompatible or major changes.
- `MINOR`: Backwards-compatible new features and enhancements.
- `PATCH`: Backwards-compatible bugfixes and package updates.

## Support

Got questions?

You have several options to get them answered:

- The [Home Assistant Community Apps Discord chat server][discord] for app
  support and feature requests.
- The [Home Assistant Discord chat server][discord-ha] for general Home
  Assistant discussions and questions.
- The Home Assistant [Community Forum][forum].
- Join the [Reddit subreddit][reddit] in [/r/homeassistant][reddit]

You could also [open an issue here][issue] on GitHub.

## Authors & contributors

The original setup of this repository is by [Franck Nijhof][frenck].

For a full list of all authors and contributors,
check [the contributor's page][contributors].

## License

MIT License

Copyright (c) 2018-2026 Franck Nijhof

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

[addon-badge]: https://my.home-assistant.io/badges/supervisor_addon.svg
[addon]: https://my.home-assistant.io/redirect/supervisor_addon/?addon=influxdb-vistalba&repository_url=https%3A%2F%2Fgithub.com%2Fvistalba%2Faddon-influxdb
[contributors]: https://github.com/vistalba/addon-influxdb/graphs/contributors
[discord-ha]: https://discord.gg/c5DvZ4e
[discord]: https://discord.me/hassioaddons
[forum]: https://community.home-assistant.io/t/home-assistant-community-add-on-influxdb/54491?u=frenck
[frenck]: https://github.com/frenck
[issue]: https://github.com/vistalba/addon-influxdb/issues
[reddit]: https://reddit.com/r/homeassistant
[releases]: https://github.com/vistalba/addon-influxdb/releases
[semver]: https://semver.org/spec/v2.0.0.html
