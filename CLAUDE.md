# Scrutiny HDD Monitor — deployment repo

Deploys [Scrutiny](https://github.com/AnalogJ/scrutiny) (SMART monitoring + web UI)
to a Raspberry Pi 4 homelab via Docker Compose. This repo is authored on a
workstation and then `git clone`d onto the Pi; the app runs on the Pi only.

## Deployment target

- Host: Raspberry Pi 4, hostname `raspberrypi`, user `daniel`
- Arch: `aarch64` (arm64) — any images pulled must have an arm64 tag
- Access: over a Tailscale tailnet, interface `tailscale0`. LAN is `eth0`.
- Docker + Docker Compose already installed and in use
- Existing neighbour on this Pi: Jellyfin, running via Compose with
  `network_mode: host`, UFW-scoped to `eth0` only
- UFW is hardened — do NOT open ports globally. Scrutiny's web port should be
  reachable **on the tailnet only** → `ufw allow in on tailscale0 to any port <n>`

## The drive being monitored

- Model: WD Scorpio Blue 320 GB (`WDC WD3200BPVT-24A0RT0`)
- Serial: `WD-WXC1A8026620`
- Kernel device: `/dev/sdb` (single partition `/dev/sdb1`)
- Stable path (prefer this in configs):
  `/dev/disk/by-id/ata-WDC_WD3200BPVT-24A0RT0_WD-WXC1A8026620`
- Filesystem: ext4, mounted at `/srv/media`
- Transport: USB enclosure

## USB bridge — smartctl device type

- Bridge: Realtek `0x0bda:0x9201` — reported by smartctl as "Unknown USB bridge"
- **Required smartctl flag: `-d sat`.** Plain `smartctl` calls fail. This must
  be plumbed into Scrutiny's collector config as the device type (`type: sat`
  in `collector.yaml`).

## Spindown — the non-negotiable constraint

The drive is configured for aggressive power saving and this must NOT be
defeated by the monitoring:

- APM level 254, 20-minute idle spindown
- Configured via `hdparm` in a udev rule at `/etc/udev/rules.d/69-hdparm.rules`
  on the Pi (owned by the OS, not this repo)
- The Scrutiny collector polls `smartctl` on a schedule. A naive poll spins the
  drive up. **The collector must invoke smartctl with `-n standby`** so it
  skips the read when the drive is in standby (smartctl exits 2, no ATA
  commands sent, drive stays asleep).

### Verified 2026-07-04 on the Pi

- `sudo smartctl -n standby -a -d sat /dev/sdb` while drive was in standby →
  printed *"Device is in STANDBY mode, exit(2)"* and `hdparm -C` afterwards
  still reported `standby`. The USB bridge honours the ATA CHECK POWER MODE
  passthrough correctly. `-n standby` is safe on this hardware.

If the bridge behaviour changes (firmware update, enclosure swap), re-run:
```
sudo hdparm -y /dev/sdb            # force standby
sudo hdparm -C /dev/sdb            # confirm: standby
sudo smartctl -n standby -a -d sat /dev/sdb   # must exit 2, no SMART table
sudo hdparm -C /dev/sdb            # must still be standby
```

## Scrutiny collector config essentials

Verified against upstream `example.collector.yaml` (master, fetched 2026-07-04).
Scrutiny has NO built-in "skip standby" mode. Workaround: override the raw
smartctl arg strings so `-n standby` is included on every disk-touching
invocation. Both keys below run on every poll cycle, so both need the flag.

```yaml
version: 1
host:
  id: ""

devices:
  - device: /dev/disk/by-id/ata-WDC_WD3200BPVT-24A0RT0_WD-WXC1A8026620
    type: 'sat'
    commands:
      metrics_info_args:  '--info --json -n standby'
      metrics_smart_args: '--xall --json -n standby'

allow_listed_devices:
  - /dev/disk/by-id/ata-WDC_WD3200BPVT-24A0RT0_WD-WXC1A8026620
```

`allow_listed_devices` prevents Scrutiny from also trying to poll the SD card,
eMMC, or zram device that `smartctl --scan` may return.

### Open risk — verify empirically on first deploy

When the drive is in standby, `smartctl -n standby` exits **2** with no
JSON output. It is unknown whether Scrutiny's collector:
- (good) treats exit(2) as "skip this cycle, retain last-known values", or
- (bad) treats exit(2) as a hard error and drops/hides the device.

First-deploy verification steps:
1. Deploy with the config above.
2. Force standby: `sudo hdparm -y /dev/sdb`
3. Wait for one collector cycle (default schedule is `0 0 * * *` — hourly in
   omnibus, cron string in newer versions; check docker logs for the tick).
4. Check `docker logs scrutiny` for the collector run. Expected: a log line
   noting the device is in standby and the cycle was skipped.
5. Confirm `sudo hdparm -C /dev/sdb` still says `standby` after the tick.
6. Confirm the drive still appears in the web UI with previous SMART data.

If Scrutiny drops the device on exit(2), fallback plan: wrap smartctl in a
small script that intercepts exit 2 and returns an empty-but-valid JSON
document + exit 0. Point `metrics_smartctl_bin` at the wrapper.

## Web UI exposure — Phase 1 (now)

- **Tailnet-only, no reverse proxy, no auth.**
- **Port: 8080** (confirmed free on the Pi 2026-07-05 via `ss -tlnp`).
- `network_mode: host` on the Scrutiny container — matches the existing
  Jellyfin pattern on this Pi. Scrutiny binds to `0.0.0.0:8080` and Influx
  binds to `0.0.0.0:8086`; UFW's default-deny keeps 8086 unreachable
  externally, and only 8080 is opened, only on tailscale0.
- UFW rule on the Pi:
  `sudo ufw allow in on tailscale0 to any port 8080 proto tcp comment 'scrutiny web (tailnet only)'`
- Accepted risk: partner is on the tailnet and Scrutiny's UI is fully
  writable (thresholds, notification endpoints, device labels — no
  read-only mode exists in Scrutiny). Interim only until Phase 2.

## Web UI exposure — Phase 2 (deferred)

Add a Caddy reverse proxy in front of Scrutiny with:
- HTTP Basic Auth (bcrypt-hashed password in Caddyfile).
  **Basic Auth is only safe over HTTPS** — TLS is not optional here.
- TLS via one of:
  - **Tailscale-issued Let's Encrypt certs** for
    `raspberrypi.<tailnet>.ts.net` — clean, no browser warnings on tailnet
    devices, but does not cover eth0/LAN access.
  - **Caddy `tls internal`** — Caddy generates a local CA and issues certs;
    covers every hostname/interface, but browsers warn until the Caddy CA
    root cert is installed on each client device.
- Expose Caddy on `tailscale0` + `eth0`; keep Scrutiny bound to `127.0.0.1`
  (proxy is the only path in).

Trigger to do Phase 2: whenever the current no-auth situation feels wrong,
or when access from a non-tailnet LAN device becomes desirable.

## Repo layout

```
scrutiny-HDD-monitor/
├── CLAUDE.md                   this file
├── docker-compose.yml          omnibus, pinned to v0.9.2-omnibus, host net
├── config/
│   └── collector.yaml          type: sat + -n standby injection
├── data/                       git-ignored, InfluxDB persistence
└── .gitignore                  ignores data/
```

The Pi clones this repo into `~daniel/` (or wherever the user chose) and
runs `docker compose up -d` from that directory. Config lives in git;
InfluxDB data does not.

## Deployment procedure

Run on the Pi after cloning/pulling:

```bash
cd ~/scrutiny-HDD-monitor
docker compose up -d
docker compose logs -f scrutiny          # sanity check; Ctrl-C when steady

sudo ufw allow in on tailscale0 to any port 8080 proto tcp \
  comment 'scrutiny web (tailnet only)'
sudo ufw status verbose | grep 8080      # confirm rule active

# Trigger a collector run to populate the dashboard
docker exec scrutiny /opt/scrutiny/bin/scrutiny-collector-metrics run

# Load http://raspberrypi:8080 from a tailnet device
```

### The load-bearing verification — MUST pass before we call this done

```bash
sudo hdparm -y /dev/sdb                                    # force standby
sudo hdparm -C /dev/sdb                                    # expect: standby
docker exec scrutiny /opt/scrutiny/bin/scrutiny-collector-metrics run
docker logs --tail=50 scrutiny                             # look for standby skip
sudo hdparm -C /dev/sdb                                    # MUST still be: standby
```

If that final `hdparm -C` reports anything other than `standby`, the whole
`-n standby` approach failed inside Scrutiny's collector and we go to the
smartctl-wrapper fallback documented above.

## Working style (from the user)

- Real deployment, not a tutorial. Correctness > completeness of explanation.
- Verify assumptions on real hardware before committing to a design. The
  standby check above is the template: state hypothesis, run the test on the
  Pi, decide from the result.
- Ask for scoping decisions (e.g. UFW rules) rather than picking a permissive
  default.
- Be concise. No trailing "here's what I did" summaries when the diff shows it.
