# jellyfin snap

A strictly confined snap of [Jellyfin](https://jellyfin.org) (the free
software media system), translated from Umbrel's
[docker-compose.yml](https://github.com/getumbrel/umbrel-apps/blob/master/jellyfin/docker-compose.yml)
using the `docker-compose-to-snap` skill. No Docker involved — the server
runs as a single snapd daemon.

## Architecture

| Compose service | In this snap |
|---|---|
| `app_proxy` (Umbrel glue) | Dropped; port 8096 exposed directly via `network-bind` |
| `server` (`linuxserver/jellyfin:10.11.11` image) | **Not** extracted from the Docker image. Jellyfin publishes its own self-contained portable build (server binary + bundled `jellyfin-web` client) at repo.jellyfin.org — that's the real upstream release artifact, so it's used directly (`snap/snapcraft.yaml` part `jellyfin`), pinned by tag and sha256. `jellyfin-ffmpeg` (upstream's own portable ffmpeg build, same one the Docker image ships) is bundled the same way rather than staging Ubuntu archive ffmpeg. |
| `PUID`/`PGID` env vars | Dropped — those are a linuxserver.io/Docker convention for remapping the container's internal uid to match the host. Irrelevant to a snap daemon (which already runs as its invoking system's root); Jellyfin itself has no uid requirement (unlike e.g. Postgres), so no `system-usernames` drop-privileges dance was needed here. |
| `${APP_DATA_DIR}/data/config:/config` | Split by Jellyfin's own `--configdir`/`--datadir`/`--cachedir`/`--logdir` flags: config (small, versioned) → `$SNAP_DATA/config`; library DB + metadata images (bulk) → `$SNAP_COMMON/data`; cache → `$SNAP_COMMON/cache`; logs → `$SNAP_COMMON/log`. The linuxserver image collapses all four under one `/config` mount; this snap keeps them separate because `$SNAP_DATA` is what `snap save` snapshots per revision — you don't want gigabytes of metadata cache in every snapshot. |
| `${UMBREL_ROOT}/data/storage/downloads:/downloads` | No single equivalent — see "Media libraries" below. |
| `hostname: "${DEVICE_HOSTNAME}"` | Dropped (cosmetic container hostname; not meaningful for a snap daemon). |
| `7359:7359/udp` (auto-discovery) | Bound by the daemon via `network-bind`; no separate config needed. |
| `restart: on-failure` | `restart-condition: on-failure` |

No `ports.http` / other config options are exposed via `snapctl`. Unlike
Immich, Jellyfin has no supported CLI flag or env var to override its HTTP
port — that's set through Jellyfin's own Dashboard → Networking after first
login (this matches how every other Jellyfin install works, container or
not), so duplicating it as a snap option would just be a second source of
truth. Default port stays at Jellyfin's own default, 8096, matching the
compose file's `APP_PORT`.

## Media libraries and confinement

Strict confinement means Jellyfin can only read paths it's been granted:

- `$SNAP_COMMON/media` is created by the install hook as a ready-to-use
  default library location.
- The `home` and `removable-media` plugs (both auto-connect) additionally
  let Jellyfin read anything under users' home directories or mounted
  removable media (`/media`, `/mnt`, `/run/media`).
- A library on some *other* host path (e.g. a manually-mounted NAS share
  under `/srv`) needs the `system-files` interface, which requires a manual
  connection (and, for a Store upload, human review — it's not
  auto-connecting): `sudo snap connect jellyfin:system-files`. This snap
  does not declare that plug by default to keep the confinement surface
  minimal; add it in `snapcraft.yaml` if your library layout needs it.

## Build

```bash
snapcraft pack --verbose
sudo snap install ./jellyfin_*.snap --dangerous
snap services jellyfin
curl -fsS http://localhost:8096/health   # expect "Healthy"
```

Then open http://localhost:8096, run the first-run setup wizard, and add a
library pointed at `/var/snap/jellyfin/common/media` (or a home/removable
path).

## Status / not yet verified

Built to the skill's Step 3 (snapcraft.yaml + wrappers + install hook), but
**Step 4 (build/install/verify) has not been run yet** — `snapcraft pack`
has not been executed in this environment. Before trusting this:

- Confirm `snapcraft pack` succeeds and both dump parts land their payload
  where expected (`$SNAP/jellyfin/jellyfin`, `$SNAP/jellyfin/jellyfin-web`,
  `$SNAP/usr/bin/ffmpeg`).
- Confirm the daemon starts and stays active (`snap services jellyfin`),
  and check `sudo snap logs jellyfin -n 100` on any failure.
- Confirm playback actually transcodes (proves the bundled `jellyfin-ffmpeg`
  is found and runs under confinement — AppArmor denials on `exec` are the
  most likely first failure; check `dmesg | grep DENIED`).
- Confirm `sudo snap remove --purge jellyfin` + reinstall gives a clean
  first-run wizard (proves the install hook is idempotent and no state
  leaked outside `$SNAP_DATA`/`$SNAP_COMMON`).
- The pinned build is amd64-only (`platforms: [amd64]`); arm64 users would
  need the arm64 tarball URLs from repo.jellyfin.org substituted in, plus a
  multi-arch part setup.
