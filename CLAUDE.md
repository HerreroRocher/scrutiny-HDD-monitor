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
  `network_mode: host`, UFW-scoped to `eth0`

### Network posture (verified 2026-07-05)

- **UFW allows ALL ports inbound on `tailscale0`.** No per-port rule needed
  for tailnet-facing services; they're reachable the moment they bind.
- **UFW default-deny on `eth0`**, with per-port exceptions (Jellyfin is the
  current one).
- **Tailscale ACLs are the effective access control on the tailnet side** —
  who on the tailnet can reach which services is enforced there, not in UFW.
- Practical consequence for Scrutiny Phase 1: nothing to add to UFW. It
  binds via `network_mode: host` on `:8080`, tailscale0 already permits
  that, and every tailnet-enrolled device (including partner's) reaches it.
  This is why Phase 2 (auth via Caddy) matters — it's the boundary that
  keeps trusted tailnet devices from tweaking Scrutiny settings without
  fiddling with Tailscale ACLs.

## The drive being monitored

- Model: WD Scorpio Blue 320 GB (`WDC WD3200BPVT-24A0RT0`)
- Serial: `WD-WXC1A8026620`
- Kernel device: `/dev/sda` (single partition `/dev/sda1`) as of the current
  boot. **This is unstable and about to change:** we're moving the configs to
  reference the drive by its stable `/dev/disk/by-id/...` path (see below), so
  treat every `/dev/sda` in this doc as "whatever the kernel currently
  enumerates it as," not a fixed name.
- Stable path (prefer this in configs — migration in progress):
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
defeated by nightly monitoring:

- APM level 254, 20-minute idle spindown
- Configured via `hdparm` in a udev rule at `/etc/udev/rules.d/69-hdparm.rules`
  on the Pi (owned by the OS, not this repo)
- User preference (2026-07-05): infrequent daytime spin-ups are ACCEPTABLE
  (once/day at a chosen time). Repeated nightly spin-ups are NOT.

### Verified 2026-07-04 on the Pi

- `sudo smartctl -n standby -a -d sat /dev/sda` while drive was in standby →
  printed *"Device is in STANDBY mode, exit(2)"* and `hdparm -C` afterwards
  still reported `standby`. The USB bridge honours the ATA CHECK POWER MODE
  passthrough correctly. `-n standby` is safe on this hardware.

If the bridge behaviour changes (firmware update, enclosure swap), re-run:
```
sudo hdparm -y /dev/sda            # force standby
sudo hdparm -C /dev/sda            # confirm: standby
sudo smartctl -n standby -a -d sat /dev/sda   # must exit 2, no SMART table
sudo hdparm -C /dev/sda            # must still be standby
```

## Scrutiny collector config — final shape

Verified against upstream `example.collector.yaml` (master, fetched 2026-07-04).

The strategy that survived contact with reality:
1. **`-n standby` on NEITHER arg string.** Both breakage modes we hit
   (first-run device drop from info, false "SMART failed" from smart) came
   from Scrutiny treating exit 2 as a hard failure, not a "skip cycle".
2. **Change the cron schedule to run once a day at 20:00.** This is the
   power-saving lever, not the smartctl args. Set via env var
   `COLLECTOR_CRON_SCHEDULE: "0 20 * * *"` in docker-compose.yml.
3. **Manual runs (`docker exec ... scrutiny-collector-metrics run`) will
   wake the drive.** That is expected and acceptable — the user chose that.

```yaml
version: 1
host:
  id: "raspberrypi"

allow_listed_devices:
  - /dev/sda

devices:
  - device: /dev/sda
    type: 'sat'
    commands:
      metrics_info_args:  '--info --json'
      metrics_smart_args: '--xall --json'
```

`allow_listed_devices` prevents Scrutiny from also polling the SD card,
eMMC, or zram device that `smartctl --scan` may return.

### Gotcha: Scrutiny lowercases device paths — verified 2026-07-05

Scrutiny's collector YAML parser silently **lowercases the value** of the
`device:` field before passing it to smartctl. Docs mention it lowercases
config *keys* — it also mangles values.

The WD's stable by-id symlink is uppercase:
`/dev/disk/by-id/ata-WDC_WD3200BPVT-24A0RT0_WD-WXC1A8026620`

Configuring that path caused Scrutiny to invoke:
```
smartctl --info --json --device sat /dev/disk/by-id/ata-wdc_wd3200bpvt-24a0rt0_wd-wxc1a8026620
```
That lowercase path does not exist → smartctl exit 2 (device open failed) →
Scrutiny "has no scrutiny UUID; skipping (no data association possible)".
When the same command is run manually via `docker exec smartctl` with the
correct uppercase path, it succeeds with exit 0. That was the whole bug.

Workaround: use `/dev/sda` (already lowercase, no mangling). Trade-off is
that kernel names aren't as stable as by-id if the enclosure order changes,
but on this single-USB-disk Pi it's fine. If a second USB disk is added,
options are (a) create a lowercase udev symlink in `/dev/disk/by-id/`, or
(b) filter by-path/by-uuid instead.

### History: why we abandoned `-n standby` entirely — 2026-07-05

First attempt put `-n standby` on both args. It failed:
```
level=info  msg="Executing command: smartctl --info --json -n standby --device sat /dev/disk/by-id/..."
level=error msg="Could not retrieve device information for ...: exit status 2"
level=error msg="Device ... has no scrutiny UUID; skipping (no data association possible)."
```
Scrutiny needs a successful `--info` on every cycle to associate the device
with a UUID. Exit 2 → "unknown device" → whole device dropped for the cycle.
No SMART poll is attempted after info fails.

Follow-up test on the Pi:
```
sudo hdparm -y /dev/sda                                 # force standby
sudo smartctl --info --json -d sat /dev/sda ; echo $?   # exit: 0
sudo hdparm -C /dev/sda                                 # still: standby
```
The WD's controller answers IDENTIFY DEVICE from cache without spinning up.
So `--info` alone is safe to run against a sleeping drive on this specific
disk+bridge. This is drive/bridge-specific behavior; if the drive is ever
swapped, re-run that test before assuming it holds.

### Then: `-n standby` on `metrics_smart_args` produced a false SMART failure

With info fixed, the collector still marked the drive as "SMART failed" in
the UI on every standby cycle. Log:
```
level=info  msg="Executing command: smartctl --xall --json -n standby --device sat /dev/sda"
level=error msg="smartctl returned an error code (2) while processing sda"
level=error msg="smartctl could not open device"
level=info  msg="Publishing smartctl results for <uuid>"
```
Scrutiny does NOT distinguish exit-2-standby from exit-2-real-failure — it
publishes a failure record either way. That is a persistent false alarm on
the dashboard, which trains the user to ignore Scrutiny alerts, which
defeats the point of running Scrutiny.

Wrapper script (return synthetic empty JSON with exit 0) would work but is
brittle. Instead we solved this at the scheduling layer.

### Final approach: schedule instead of skip — 2026-07-05

- Remove `-n standby` from both arg strings. Let smartctl behave normally.
- Change `COLLECTOR_CRON_SCHEDULE` from the default `0 0 * * *` (midnight)
  to `0 20 * * *` (20:00 — prime Jellyfin evening use, drive likely awake).
- Manual `scrutiny-collector-metrics run` calls will always wake the drive
  if asleep; user is fine with that ("at my command" is acceptable).

Net effect on power: at worst, one drive spin-up per day at 20:00 followed
by the 20-min idle timer → ~20 min/day of extra active time. In practice
much less because Jellyfin usually has the drive awake at that hour anyway.

### Non-bug: dashboard shows 4 hourly temperature points per day — verified 2026-07-15

The temperature graph shows data at ~18:00, 19:00, 20:00 and 21:00 every day,
which looks like the collector is running four times (four spin-ups). **It is
not.** The collector runs exactly once, at 20:00 — confirmed against the
container's only cron entry (`0 20 * * *` in `/etc/cron.d/scrutiny`) and
against `docker logs scrutiny`, which shows a single `smartctl --info` +
`smartctl --xall` pair per day.

The extra points come from the drive's own onboard **SCT temperature history
log** (`ata_sct_temperature_history` in the `--xall` JSON): a 128-slot
circular buffer the drive's firmware samples every minute
(`sampling_period_minutes: 1`), so at collection time it holds roughly the
last ~2 hours of readings. Scrutiny parses that table and backdates each
sample into InfluxDB, spreading them across the hours preceding the single
20:00 poll — hence the 18:00–21:00 hourly buckets. Zero extra smartctl calls,
zero extra spin-ups; the spindown constraint is intact.

If you ever need to confirm the poll count again, `docker logs scrutiny |
grep "Executing command"` is the source of truth, not the dashboard graph.

## Web UI exposure — Phase 1 (now)

- **Tailnet-only, no reverse proxy, no auth.**
- **Port: 8080** (confirmed free on the Pi 2026-07-05 via `ss -tlnp`).
- `network_mode: host` on the Scrutiny container — matches the existing
  Jellyfin pattern on this Pi. Scrutiny binds to `0.0.0.0:8080` and Influx
  binds to `0.0.0.0:8086`.
- **No UFW change needed** — tailscale0 already permits all inbound ports
  by policy. eth0 default-deny keeps 8080/8086 unreachable from the LAN.
- Accepted risk: every tailnet device (including partner's) can reach and
  modify Scrutiny's UI (thresholds, notification endpoints, device labels
  — no read-only mode exists in Scrutiny). Phase 2's Caddy auth layer is
  the boundary that fixes this without touching Tailscale ACLs.

## Web UI exposure — Phase 2 (deferred)

Bundle these together — the proxy is a natural home for both:

### 2a. Caddy reverse proxy with auth + TLS

- HTTP Basic Auth (bcrypt-hashed password in Caddyfile).
  **Basic Auth is only safe over HTTPS** — TLS is not optional here.
- TLS via one of:
  - **Tailscale-issued Let's Encrypt certs** for
    `raspberrypi.<tailnet>.ts.net` — clean, no browser warnings on tailnet
    devices, but does not cover eth0/LAN access.
  - **Caddy `tls internal`** — Caddy generates a local CA and issues certs;
    covers every hostname/interface, but browsers warn until the Caddy CA
    root cert is installed on each client device.
- Expose Caddy on `tailscale0` + `eth0`; take Scrutiny off `network_mode:
  host` and bind it to a private Docker network (or bind its port to
  `127.0.0.1`) so the proxy is the only path in.

### 2b. Live drive-state widget

Scrutiny does not display current spindown state (standby/active). User
asked for this and we chose to add it as a tiny companion service behind
the same Caddy auth rather than a separate exposure:

- Small service on the Pi (systemd timer or a lightweight sidecar
  container) that runs `hdparm -C /dev/sda` every ~30s and writes state to
  a file, plus a minimal HTML/text endpoint that serves it.
- Caddy proxies e.g. `/state` (same host) to that endpoint, protected by
  the same basic auth as Scrutiny.
- Zero SMART impact — hdparm's power-state check is the same lightweight
  ATA command Scrutiny already uses safely.

### Trigger

Do Phase 2 when: the current no-auth situation feels wrong, access from a
non-tailnet LAN device becomes desirable, or the drive-state visibility
becomes actively useful (not just curiosity).

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

# No UFW rule needed — tailscale0 is blanket-allowed on this host.

# Trigger a collector run to populate the dashboard
docker exec scrutiny /opt/scrutiny/bin/scrutiny-collector-metrics run

# Load http://raspberrypi:8080 from a tailnet device
```

### The load-bearing verification — MUST pass before we call this done

```bash
sudo hdparm -y /dev/sda                                    # force standby
sudo hdparm -C /dev/sda                                    # expect: standby
docker exec scrutiny /opt/scrutiny/bin/scrutiny-collector-metrics run
docker logs --tail=50 scrutiny                             # look for standby skip
sudo hdparm -C /dev/sda                                    # MUST still be: standby
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
