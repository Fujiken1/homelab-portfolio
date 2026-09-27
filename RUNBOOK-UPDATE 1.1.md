# Fujiken Server — Infrastructure Runbook

Self-hosted game-server infrastructure running behind CGNAT via a cloud relay.
This document is the single source of truth for operating and maintaining the
system. Keep it updated when anything changes.

Last reviewed: 2026-09-27

> **What changed on 2026-09-27 (sanity-check review):** crontab now matches the live
> server (shutdown moved to 03:00, `@reboot daily` removed); Pterodactyl stop schedule
> marked decommissioned; Telegram bot (`fujiken-bot.service`) documented; Tailscale
> added to the stack; NVMe link confirmed at 5 Gbps; backup gaps + no-rotation flagged;
> Azure egress measured; new troubleshooting rows (Windows .pem permissions, Wings
> console spinner, router swap); electricity cost corrected.

---

## 1. What this is

A private Minecraft (and general game) server that runs on a home laptop behind
CGNAT, made reachable to friends over the internet through a cheap cloud VM acting
as a reverse-tunnel relay. The home network is never exposed; the laptop only ever
makes an *outbound* connection to the relay, which is what defeats CGNAT.

The same pattern (relay + outbound tunnel) is what Cloudflare Tunnel, ngrok, and
Tailscale Funnel productize. This is a self-hosted version of it.

Because both `frpc` and Tailscale dial **out**, the setup does not depend on the home
public IP, ISP, or router. Swapping ISPs (e.g. Globe → PLDT backup) needs no DDNS and
no relay changes — only the LAN IP reservation matters (see Section 10).

### Traffic paths

- Friends:  `client -> Azure relay (Jakarta) -> tunnel -> laptop frpc -> container`  (~85-100 ms)
- You/LAN:  `gaming PC -> laptop socat :25564 -> container`  (~1 ms, never touches Azure)
- Remote admin:  `device -> Tailscale -> laptop` (Immich, Portainer, SSH)

Both game paths terminate at the same Minecraft container, so it's one world with two doors.

> **Scope note:** Fujiken has grown past just the game server — it now also runs a
> self-hosted photo stack (Immich + Syncthing) and Docker management (Portainer).
> This runbook is the operational source of truth for **all** services on the box
> (ports, security, boot, DR). The detailed photo-stack architecture and 3-2-1
> backup design live in the companion **`RUNBOOK-HOMELAB.md`**. Modpack details live
> in **`aegle-frathouse-runbook.md`**.

---

## 2. Component inventory

### Cloud relay (Azure)

| Item | Value |
| --- | --- |
| Provider | Microsoft Azure for Students |
| Region | Indonesia Central (Jakarta) |
| VM size | B2ats v2 (2 vCPU, 1 GiB) — free-tier eligible |
| OS | Ubuntu Server 24.04 LTS |
| Public address (DNS) | `fujiken.indonesiacentral.cloudapp.azure.com` |
| Role | Runs `frps` (reverse-proxy server). Forwards only. |
| Auth into VM | SSH key only (`.pem`), root login disabled |
| Uptime | Always on (24/7). Last boot: 2026-06-11 |

### Home server (laptop)

| Item | Value |
| --- | --- |
| Hardware | Lenovo IdeaPad Gaming 3 15ARH7, Ryzen 5 6600H, 24 GB DDR5, dGPU disabled |
| OS disk | External NVMe (~476 GB, expanded via LVM) running Ubuntu Server 24.04 |
| NVMe connection | USB-A cable into a 10 Gbps port — link negotiates **5 Gbps** (`uas`, `5000M` in `lsusb -t`), ~500 MB/s. Sufficient for all Minecraft workloads |
| Internal disk | Windows — untouched, used when laptop is portable |
| Local IP | `192.168.254.122` — **DHCP reservation on the router** (not set on the laptop) |
| Timezone | Asia/Manila (PST, UTC+8) — all cron times are PH time. Verified 2026-09-27 |
| Battery | Lenovo Conservation Mode ON (caps ~80%) — persists into Linux |
| UPS | CyberPower CP1350EPFCLCD — laptop + router plugged in, ~2hr brownout protection |
| CPU freq cap | 4.0 GHz max (set via crontab @reboot) — prevents thermal runaway under Chunky |
| Role | Runs the actual game servers + Pterodactyl panel + photo stack |

### Software stack (on the laptop)

| Service | Purpose | Notes |
| --- | --- | --- |
| `frpc` | Tunnel client → dials Azure `:7000` | The CGNAT-bypass |
| `wings` | Pterodactyl daemon, runs game containers in Docker | |
| `nginx` | Serves the Pterodactyl panel (HTTP only, LAN only) | |
| `php8.3-fpm` | PHP backend for the panel | |
| `mariadb` | Panel database | Secured; bound to `127.0.0.1` |
| `redis-server` | Panel cache / sessions | |
| `fail2ban` | SSH brute-force protection | |
| `minecraft-local` | `socat` LAN forward `:25564` → container | Gives you ~1 ms local play |
| `docker` | Container runtime for Wings, Immich, Portainer | |
| `tailscaled` | Tailscale — remote admin access (outbound, NAT-proof) | Node `fujikenserver`, `100.78.160.65` |
| `fujiken-bot` | Telegram command listener (`fujiken-report listen`) | Runs as root, whitelisted commands only — see Section 5 |
| `fujiken-monitor` | Custom Python web dashboard + `/api/metrics` JSON | systemd service, port 5050, LAN-only, no auth (accepted risk) |
| `immich` | Self-hosted photo library (Google Photos replacement) | Docker Compose at `/home/fujiken/immich`, port 2283. Library on external NVMe |
| `syncthing@fujiken` | Syncs Immich library → main PC (local redundancy) | GUI on `0.0.0.0:8384`; config at `~/.local/state/syncthing/config.xml` |
| `portainer` | Docker management UI (Portainer CE) | Port 9000. Tailscale-only, never publicly exposed |

### Tailscale devices (as of 2026-09-27)

| Device | Tailscale IP | Status |
| --- | --- | --- |
| `fujikenserver` (laptop) | `100.78.160.65` | Online |
| `iphone183` | `100.80.90.127` | Offline ~72 days |
| `tab-s9-fe` | `100.96.60.25` | Offline ~106 days |

Main PC is **not** on the tailnet. No node sharing with friends has been set up.
If remote access or Immich mobile backup away from home matters, reconnect the phone.

### Networking reference

| Port | Where | Purpose |
| --- | --- | --- |
| 22 | Azure + laptop | SSH |
| 7000 | Azure | frp control channel (laptop dials in) |
| 25565 | Azure | Minecraft, public (friends connect here) |
| 25564 | Laptop LAN | socat local forward (you connect here) |
| 80 | Laptop LAN | Pterodactyl panel (do NOT expose) |
| 2022 | Laptop LAN | Wings daemon (panel ↔ Wings, do NOT expose) |
| 2283 | Laptop LAN / Tailscale | Immich photo library |
| 8384 | Laptop LAN | Syncthing web GUI |
| 9000 | Laptop LAN / Tailscale | Portainer CE (do NOT expose) |
| 5050 | Laptop LAN | fujiken-monitor dashboard (no auth — do NOT expose) |
| Container | `172.18.0.2:25565` | Docker internal — frpc + socat both point here |

> Note: Immich, Portainer, and the monitor dashboard are reachable remotely **only**
> via Tailscale, never through the frp tunnel. The tunnel forwards the Minecraft
> port only. Adding any of these to `frpc.toml` would expose them publicly — don't.

> Note: the Docker container IP (`172.18.0.2`) is stable as long as one server
> runs at a time. If it ever changes, both `frpc` and `socat` configs must be
> updated to match, or nothing connects.

### frp version

Both ends run **frp v0.69.1**. They must match. If you upgrade one, upgrade both.

---

## 3. Secrets & credentials (record these in a password manager — NOT here)

Do not write actual secrets in this file. Keep them in a password manager. The
list of what exists:

- [ ] Azure account login
- [ ] SSH private key (`.pem`) for the Azure VM — currently stored at `F:\azure key\` on the main PC (NTFS). **Also put a copy in the password manager** (file attachment) so losing the drive doesn't lock you out
- [ ] SSH credentials / key for the laptop
- [ ] frp shared token (in `/etc/frp/frps.toml` and `/etc/frp/frpc.toml` — must match)
- [ ] Pterodactyl admin login (panel) — 2FA/TOTP available, enable it
- [ ] MariaDB root password
- [ ] MariaDB panel-user password
- [ ] Portainer admin login (was reset June 2026 — record the new one)
- [ ] Immich admin login (was reset July 2026 — record the new one)
- [ ] BitLocker recovery key for the internal Windows drive
- [ ] Telegram **bot token** (in `/usr/local/bin/fujiken-report`). The chat ID is not a secret; the token is. If the token leaks: `/revoke` in @BotFather, then update the script and `sudo systemctl restart fujiken-bot`
- [ ] Azure Storage account access key (once Blob backup is configured)

---

## 4. Daily operation

### Normal daily cycle

Fujiken runs **on demand**: you boot it when you want to use it, and it shuts itself
down at night. All times are **Philippine Standard Time (PST, UTC+8)**.

- Boot manually whenever you want to play / use it; everything auto-starts
- 03:00 — laptop shuts down (crontab)
- Typical runtime: ~15-16 hours/day

> ⚠️ **The Pterodactyl stop schedule is decommissioned.** Nothing stops the game
> server before the 03:00 shutdown, so the OS kills the container directly (Docker
> gives it ~10 s). A heavy modded world may not finish saving in that time →
> rollback or chunk corruption. Either stop the server manually before 3AM, or
> re-add a panel schedule (Power: Stop at 02:50, with a warning a few minutes before).

On boot, these come up automatically (all `systemctl enable`d, verified 2026-09-27):
`frpc`, `wings`, `nginx`, `php8.3-fpm`, `mariadb`, `redis-server`, `fail2ban`,
`minecraft-local`, `docker`, `tailscaled`, `fujiken-bot`, `fujiken-monitor`,
`syncthing@fujiken`.
The game server itself is started from the Pterodactyl panel.

### Connecting

- Friends: `fujiken.indonesiacentral.cloudapp.azure.com`
- You (lag-free, at home): `192.168.254.122:25564`

### Switching which game/server is live

Only one server runs at a time, all sharing the same public port. To switch:

1. Pterodactyl → stop current server
2. Pterodactyl → start the other server
3. Done — same address, no config changes needed

`frpc` always points at `172.18.0.2:25565`, so it doesn't care which server is behind it.

### Mod / config changes (Minecraft)

Any mod or config change must go to the **server and every client at the same time**
(Pterodactyl Files → config, and each friend's Prism instance). A mismatch crashes
clients on join. Details in `aegle-frathouse-runbook.md`.

### Taking the laptop portable

1. Stop servers in Pterodactyl
2. `sudo shutdown now` (let it power off fully)
3. Unplug the external NVMe
4. Boot → Windows loads normally

To return to server mode: plug NVMe back in, boot, everything auto-starts.
**Never unplug the NVMe while Ubuntu is running** — risk of world corruption.

---

## 5. Health checks & monitoring

### Tools

| Command | Where | What it does |
| --- | --- | --- |
| `healthcheck` | Laptop | Full status: services, container, tunnel, RAM/disk/temp, fail2ban, backups |
| `healthcheck` | Azure VM | Relay status: frps, ports, tunnel connections, bandwidth **since VM boot** (not per month) |
| `fujimon` | Laptop | Live terminal dashboard (1s refresh): CPU temp/load, RAM, NVMe temp, battery, network, container. Does **not** show CPU power or frequency |
| `fujiken-monitor` | Laptop | Web dashboard on port 5050 (2s refresh) + `/api/metrics` JSON. Chart.js history graphs. systemd service, LAN-only, no auth. Reachable remotely via Tailscale only |

These four are **read-only** and safe to run anytime, even mid-game. (The Telegram
bot below is **not** read-only — it can restart services and reboot.)

### fujiken-report (script) + Telegram bot

`/usr/local/bin/fujiken-report` does both the scheduled reports and the bot.
The copy of `fujiken-report.sh` in this project is **out of date** — the live
script is the source of truth.

**CLI modes (run by cron / systemd):**

| Command | Run by | What |
| --- | --- | --- |
| `fujiken-report log` | Cron, every minute | Silently logs PPT to `/var/log/fujiken-ppt.log` (keeps a 48h tail; monthly kWh tracked separately in the script) |
| `fujiken-report status` | Cron, every 2 hours | CPU temp, power, RAM, disk, battery, server/tunnel status |
| `fujiken-report report` / `daily` | On demand only | Electricity report since the last report (first run: last 24h) |
| `fujiken-report listen` | `fujiken-bot.service` | Telegram command listener |

**Bot commands (send in Telegram):**

| Command | Does |
| --- | --- |
| `status` | Send a status report now |
| `report` / `daily` / `electricity` | Send the electricity report |
| `restart-wings` | Restart Wings |
| `restart-frpc` | Restart the tunnel client |
| `restart-panel` | Restart the panel services |
| `backup-now` | Run a world backup now; sends ❌ alert if it fails (error log in `/tmp/backup-err.log`) |
| `reboot` | Reboot the laptop |

**Bot security:** runs as root by design, but only dispatches a fixed whitelist —
no shell/eval command (see the comment in `fujiken-bot.service`). Messages from any
chat other than the configured `CHAT_ID` are ignored. **Never add a generic
shell/eval command** — that turns the bot into a root backdoor gated only by the token.

Power calculation: `(PPT + 13W overhead) / 0.87 PSU efficiency = wall draw`
(13W overhead confirmed by measurement — charger plugged in adds ~20-22W).
Meralco rate: update `MERALCO_RATE` monthly in the script.

### Quick one-offs

```bash
sensors | grep Tctl                        # CPU temp only
sudo fail2ban-client status sshd           # how many IPs banned
sudo journalctl -u frpc -n 20 --no-pager  # tunnel log
sudo journalctl -u fujiken-bot -n 20 --no-pager  # Telegram bot log
tail -50 /var/log/fujiken-ppt.log          # PPT power log
lsusb -t                                   # NVMe link speed (look for uas, 5000M)
tailscale status                           # tailnet devices
```

### Expected healthy values

| Metric | Idle | Normal play | Danger |
| --- | --- | --- | --- |
| CPU temp (Tctl) | ~40°C | ~56°C | 90°C+ |
| CPU power (PPT) | ~7W | ~12-20W | ~50W at full all-core load — safe; see note |
| CPU freq | ~1.1 GHz | ~2-3 GHz | capped at 4.0 GHz |
| NVMe temp | ~45°C | ~47°C | 70°C+ |
| RAM | ~25% | ~35-40% | 90%+ |
| Battery | ~80% (Not charging) | — | below 20% |

> **On CPU power:** the "danger" column above is for everyday gameplay. Heavy
> all-core background work (Chunky pre-gen, or Immich photo indexing) legitimately
> pushes PPT to ~50W and temps to ~75°C — this is **normal and safe** at the
> 4.0 GHz cap. With the dGPU disabled and the freq cap in place, power is
> self-limiting; CPU *temperature* (90°C+) is the real ceiling to watch, not wattage.

---

## 6. Crontab reference (sudo crontab -l)

Matches the live server as of 2026-09-27. All times are **Philippine Standard Time
(PST, UTC+8)**. The system timezone is `Asia/Manila` — cron runs in local time.

```
# Pterodactyl task scheduler (required, do not remove)
* * * * * php /var/www/pterodactyl/artisan schedule:run >> /dev/null 2>&1

# Weekly world backup — Sunday 2AM PH
0 2 * * 0 tar -czf /home/fujiken/backups/world-$(date +\%F).tar.gz /var/lib/pterodactyl/volumes/

# Nightly shutdown — 3AM PH
0 3 * * * /sbin/shutdown -h now

# CPU frequency cap on boot — 4.0 GHz max
@reboot echo 4000000 | tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_max_freq

# Log PPT every minute
* * * * * /usr/local/bin/fujiken-report log

# Status report every 2 hours
0 */2 * * * /usr/local/bin/fujiken-report status
```

Removed since the last review: `@reboot sleep 60 && fujiken-report daily` — the
electricity report is now on demand via the bot.

> **Important:** If you ever reinstall Ubuntu, it defaults to UTC timezone. Always
> run `sudo timedatectl set-timezone Asia/Manila` immediately after setup, or all
> cron times will be off by 8 hours. After a timezone change, also run
> `sudo systemctl restart wings` (see Section 8).

---

## 7. Maintenance tasks

### Recurring

| Task | Frequency | How |
| --- | --- | --- |
| Apply OS security patches | Automatic | `unattended-upgrades` (verify it's enabled) |
| Reboot Azure VM if needed | Monthly check | On the VM: `ls /var/run/reboot-required` — if it exists, reboot when nobody's playing; `frpc` reconnects on its own |
| Apply Pterodactyl/Wings updates | When released | Follow panel upgrade docs; apply security fixes promptly |
| Update Immich | When wanted (optional) | Snapshot Postgres dir first, check release notes, then `docker compose pull && docker compose up -d` from `/home/fujiken/immich`; watch migration logs, verify web UI + iOS app compatibility after |
| Verify photo backup (Syncthing → PC) | Weekly | Confirm Syncthing shows the library in sync on both ends |
| Update Meralco rate in fujiken-report | Monthly | Edit `MERALCO_RATE` in `/usr/local/bin/fujiken-report` |
| Verify backups exist | Weekly | `ls -lh /home/fujiken/backups` — check the newest date is < 7 days old |
| Prune old backups | Monthly (manual for now) | Keep the newest ~4, delete the rest — nothing auto-deletes (see Backups) |
| Test a backup restore | Monthly | Extract a backup to a scratch dir, confirm world loads |
| Copy a backup offline (USB) | Weekly/monthly | Protects against NVMe failure |
| Check Azure cost / bandwidth | Monthly | Azure Portal → Cost Management → Cost analysis (source of truth). VM `healthcheck` shows egress **since boot only** |
| Renew Azure student credit | Yearly | Re-verify student status for fresh $100 |
| Blow dust from laptop vents | Every 3-4 months | Compressed air |

### Backups

A weekly cron job tars the game volumes (Sunday 02:00). The bot's `backup-now`
command runs the script's own backup function, which also alerts on failure.

Known problems (found 2026-09-27):

- **Skipped weeks.** Cron only fires if the laptop is on at 02:00 Sunday. Since
  2026-06-28 only 7 of ~14 Sundays got a backup (missing: Jul 12, Aug 2/9/16/23,
  Sep 6/13). Until this is automated, hit `backup-now` in Telegram if the newest
  backup is more than a week old.
- **No rotation.** Nothing deletes old backups. Each is ~5.5 GB now (26 GB total and
  growing), on the same NVMe as the server. Prune manually for now.
- **Raw `tar` in cron has no failure alert.** Only `backup-now` alerts.
- **Server is not stopped first.** `tar` runs on a live world, so a backup taken
  mid-save can be inconsistent. Prefer backing up with the game server stopped.

Backups live on the **same NVMe** as the server. This protects against griefing
and corruption, NOT drive failure. Keep an offline USB copy too.

### Disk space (already expanded)

The Ubuntu installer left most of the NVMe unallocated under LVM. It has been
expanded to use the full disk:

```bash
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
```

If you ever reinstall, remember to redo this — it's a classic Ubuntu LVM gotcha.

### Memory tuning (NeoForge)

Heavy modpacks: 14 GB heap inside the 16 GB container limit (leaves headroom).
Too much heap makes Java's garbage collector lazy, then it does one big
stop-the-world pause = lag spike. Aikar's flags are set in the server startup to
keep GC pauses small and predictable.

### CPU frequency cap

Capped at 4.0 GHz max to prevent thermal runaway during heavy all-core work.
Draw depends on the workload: Chunky pre-gen sits around ~35W PPT / ~60°C, while
fully saturating all 12 threads (e.g. Immich photo indexing) reaches ~50W PPT /
~75°C. Both are safe at this cap — the frequency limit (not a hard power limit)
is what keeps temperatures in check.
Applied automatically on every boot via crontab. To change the cap:

```bash
# Temporary (until reboot)
echo 4000000 | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_max_freq

# Permanent — edit the crontab value (in kHz: 4000000 = 4.0 GHz)
sudo crontab -e
```

### NVMe connection (5 Gbps over USB-A)

The external NVMe is plugged into a **10 Gbps** port, but the link negotiates at
**5 Gbps** (~500 MB/s) — the USB-A cable/enclosure is the limit. Confirmed with
`lsusb -t` (`Mass Storage, Driver=uas, 5000M`, on a `10000M` root hub). The original
USB-C cable failed earlier.

This is effectively SATA 3 speeds, but more than sufficient for all Minecraft
workloads — the bottleneck is always CPU or RAM, never disk bandwidth. A proper
USB-C 3.2 Gen 2 cable would unlock the full 10 Gbps if ever needed.

Keep a **spare USB-A cable** with the server hardware at all times.

---

## 8. Troubleshooting (issues we actually hit)

| Symptom | Cause | Fix |
| --- | --- | --- |
| Friends can't connect, `frpc` log says `i/o timeout` on `:7000` | Azure NSG not allowing the port | Add inbound rules for 7000 + 25565 in Azure NSG |
| `frpc` log: `token in login doesn't match` | frp tokens differ between ends | Make `frps.toml` and `frpc.toml` tokens identical, restart both |
| `frpc` log: `connection refused` to `172.18.0.2:25565` | Game server stopped, or wrong container IP | Start server in panel; confirm container IP with `docker inspect`, update `frpc.toml` |
| Panel: "CSRF token mismatch" | `SESSION_SECURE_COOKIE=true` on plain HTTP, or wrong `APP_URL` | Set `SESSION_SECURE_COOKIE=false`, fix `APP_URL`, clear config cache (as `www-data`) |
| Panel console stuck on loading spinner | Wings cached a JWT before a system clock jump (e.g. after the UTC → Asia/Manila timezone fix) → JWT denylist error | `sudo systemctl restart wings` |
| nginx won't start: `server_tokens duplicate` | Directive in both nginx.conf and site conf | Remove the duplicate line from nginx.conf |
| nginx won't start: `cannot load certificate ...pem` | Let's Encrypt failed, config expects SSL | Rewrite the site conf to plain HTTP (port 80) |
| Panel upload fails | Multiple upload limits | Raise `client_max_body_size` (nginx), `upload_max_filesize`/`post_max_size` (php.ini), `PHP_VALUE` (site conf), and `upload_limit` (Wings `/etc/pterodactyl/config.yml`) — the Wings one is the real limit |
| Can't connect via LAN | socat not running | `sudo systemctl status minecraft-local` → restart if needed |
| Local stuff breaks after router/ISP swap (LAN connect, healthcheck) | New router gave the laptop a different LAN IP | Recreate the DHCP reservation for `.122` on the new router (subnet `192.168.254.x`, laptop MAC from `ip link`). Friends via the tunnel are unaffected |
| Client crashes on join after a mod/config change | Server and client configs don't match | Push the same mod/config files to the server and every client |
| Vanilla commands missing despite op | Username mismatch (online-mode off is case/spelling sensitive) | `op ExactUsername` exactly as it appears in-game |
| CPU at 90°C | Running Chunky / world pre-gen | Expected; limit Chunky threads to keep it cooler |
| Chunky crashes with OOM | Chunk buffer outpaces disk writes at the heap ceiling | Pause/limit Chunky; raise heap only within the container limit |
| Telegram report not sending | Token/chat ID wrong, or network issue | Run `fujiken-report status` manually and check for curl errors |
| Telegram bot not responding to commands | `fujiken-bot` stopped/crashed, or token revoked | `sudo systemctl status fujiken-bot`; check `journalctl -u fujiken-bot` |
| Battery showing blank in fujimon/report | Wrong battery path | Run `ls /sys/class/power_supply/` to find correct name (BAT0 or BAT1) |
| Shutdown not firing / Telegram reports overnight | System timezone is UTC instead of Asia/Manila | Run `sudo timedatectl set-timezone Asia/Manila`, then restart Wings |
| Boot failure / OS not detecting NVMe | USB cable failure (USB-C is fragile) | Switch to a USB-A cable; `ls /dev/sd*` in initramfs to confirm drive is visible |
| Wings pinned at ~150% CPU long after boot, never settles | A resource-polling routine from the previous session didn't clean up | `sudo systemctl restart wings` — safe when no one is playing |
| fujimon shows "Server: running" when no game server is actually up | `fujimon.sh` counts **all** Docker containers, incl. Immich/Portainer | `healthcheck.sh` and `fujiken-monitor.py` filter to Wings (UUID-named) containers. `fujimon.sh` still doesn't — apply the same filter, or check the panel |
| Boot hangs on `systemd-networkd-wait-online.service` | Ethernet unplugged (laptop taken portable), or networkd waiting on Docker interfaces | Plug the cable back in. Permanent fix: `optional: true` on the interface in netplan, then `sudo netplan apply` |
| SSH to Azure VM from Windows: `UNPROTECTED PRIVATE KEY FILE` then `Permission denied (publickey)` | The `.pem` inherits read access for other groups (`Users`, `Authenticated Users`, `BUILTIN\BUILTIN`), so OpenSSH ignores the key | In PowerShell: `icacls "<key>.pem" /inheritance:r`, `/grant:r "$($env:USERNAME):(R)"`, then `/remove` each extra group until only **you (R)**, **SYSTEM (F)**, **Administrators (F)** remain. Check with `icacls "<key>.pem"` |
| Lost the Azure VM `.pem` entirely | — | Azure Portal → VM → Help → Reset password → **Reset SSH public key** (not "reset password", which would enable password auth). Use the **existing** username. Save the new key in the password manager |

Always-useful first step: run `healthcheck` on the laptop and `healthcheck` on the
VM. Between them they pinpoint which side is broken.

---

## 9. Security posture

### In place

- Home network: zero inbound exposure, real IP hidden behind relay, no port forwarding
- Dual firewall: Azure NSG + UFW (only 22, 7000, 25565 open externally)
- SSH: key-only, root login disabled, fail2ban active (laptop + VM)
- Docker container isolation for each game server
- MariaDB secured; strong frp token; panel HTTP is LAN-only
- MariaDB bound to `127.0.0.1` only (was `0.0.0.0` — fixed during June 2026 pentest, F-001)
- Telegram bot: fixed command whitelist, no shell/eval, ignores any chat but yours
- Remote admin over Tailscale only (outbound, no exposed ports)
- UPS protects against ungraceful shutdown during brownouts
- Backups + monitoring scripts + Telegram alerts

### Pentest (June 25, 2026)

A read-only recon assessment was run from Kali (WSL2) against the LAN and the
public Azure VM. Full write-up: `fujiken-pentest-report.docx`. Summary:

- **1 medium finding, fixed same session:** MariaDB was bound to all interfaces
  (`0.0.0.0:3306`), reachable by any LAN device. Fixed to `127.0.0.1`, verified
  closed with a follow-up nmap scan.
- **4 informational / accepted-risk findings** (see Standing tradeoffs below).
- External surface confirmed minimal: only ports **22** and **7000** are
  internet-facing on the Azure VM; 998/1000 filtered by the NSG. Home LAN stays
  fully hidden behind CGNAT.
- Re-run this recon whenever the stack changes materially (new exposed service,
  new host, tunnel config change). Note: the Telegram bot was added after this
  pentest and was not in scope.

### Standing tradeoffs (known, accepted)

- `online-mode=false` (cracked clients). The whitelist is a username string check,
  not real authentication. **Mitigation: keep the server address private.** If
  everyone gets legitimate accounts, flip to `online-mode=true` and the whitelist
  becomes airtight.
- Panel on plain HTTP, LAN-only. Fine at home; never log into it over public Wi-Fi.
- Single node, single admin (you). No redundancy, no uptime guarantee.
- Telegram token stored in plaintext in script — acceptable for single-user home server.
  Because the bot runs as root and can reboot/restart, a leaked token = someone can
  run the whitelisted commands. Revoke immediately if leaked.
- **fujiken-monitor dashboard (5050) has no auth** (F-002). LAN-only, trusted network.
  Cheap fix available (HTTP basic auth or UFW IP restriction) if ever wanted. Never
  add 5050 to the frp tunnel.
- **nginx discloses its version** in HTTP headers (F-003). One-line fix
  (`server_tokens off;`) if wanted; LAN-only so low priority.
- **Pterodactyl panel missing HTTP security headers** — CSP, Permissions-Policy,
  HSTS (F-004). LAN-only HTTP, so low practical impact.
- **Azure VM SSH (22) is publicly exposed** (F-005). Intentional, for remote admin.
  Mitigated by key-only auth + fail2ban. Could restrict to home IP via NSG, or move
  admin to Tailscale and close 22 entirely — noted as a future improvement.
- **Portainer (9000)** had no auth at pentest time (credentials had just been reset).
  Confirm an admin password is set and stored in the password manager. Tailscale-only.

### Do not

- Do not expose the panel (port 80) through the tunnel without HTTPS + auth.
- Do not post the server address publicly.
- Do not add a shell/eval command to the Telegram bot.
- Do not install mods/plugins from untrusted sources — they run with full server
  privileges inside the container (supply-chain risk, e.g. fractureiser 2023).

---

## 10. Disaster recovery

| If this dies | Impact | Recovery |
| --- | --- | --- |
| Game server world corrupts/griefed | Lost progress | Restore latest backup from `/home/fujiken/backups` (or USB) |
| External NVMe fails | Whole server gone | Restore from offline USB backup onto a new drive; reinstall stack |
| USB cable fails | Boot failure / OS unreachable | Swap to spare USB-A cable; drive data is fine |
| Laptop unavailable (you're out) | Server offline | None — it's intentionally tied to the laptop. Tell friends. |
| Router replaced / backup ISP router used | LAN IP may change → local connect + healthcheck break | Recreate the DHCP reservation for `192.168.254.122` (laptop MAC via `ip link`). Tunnel + Tailscale keep working regardless |
| Azure VM down / IP changed | Friends can't connect | DNS name auto-updates on VM restart; `frpc` reconnects automatically. If region/VM lost, redeploy relay + reinstall `frps` with same token |
| Lost Azure VM SSH key | Can't admin the relay (tunnel still runs) | Portal → Reset SSH public key (see Section 8) |
| Azure credit exhausted | Relay disabled | Renew student credit, or move relay to LightNode Manila / paid tier |
| Brownout longer than ~2hrs | UPS dies, ungraceful shutdown | World may need rollback from latest backup |

The relay is disposable — it holds no state. The only irreplaceable thing is the
world save, which is why backups (and an offline copy) matter most.

---

## 11. Costs & limits

| Item | Cost |
| --- | --- |
| Azure VM compute | $0 (free tier, 750 hrs/mo covers 24/7) |
| Azure public IP | ~$3-4/mo (from $100 student credit) |
| Azure egress | First ~100 GB/mo free. Measured: ~52 GB out from 2026-06-11 to 2026-09-27 ≈ **~15 GB/month** average. Heavy sessions spike (e.g. 10.5 GB in ~11 hrs) but stay well under the limit |
| Electricity | ~₱170-185/mo (~26W avg wall draw, ~15-16 hrs/day) |
| Azure Blob (photo backup) | ~$2/mo (~₱114-116) once live — 500GB Cold tier, LRS. Not yet configured |
| **Effective total** | **~₱170-185/mo now; +~₱115/mo when Blob backup goes live** |

Azure ingress (data into the VM) is always free. Egress is the metered direction.
The VM `healthcheck` bandwidth counter only resets on VM reboot — it is **total since
boot**, not per month. Use Azure Cost analysis (or vnStat, see Section 12) for monthly
numbers.

Post-graduation options: keep Azure (~$8-11/mo), switch to LightNode Manila
(~₱450/mo, much lower ping for PH players), or move everything to a dedicated
home-lab machine and keep a relay.

---

## 12. Future improvements (optional, not required)

**Backup / DR (highest priority — closes the drive-failure gap):**
- Switch the weekly cron to the script's backup function (gets failure alerts)
- Catch-up on boot: if the newest backup is older than 7 days, run one on boot
  (fixes skipped Sundays when the laptop was off)
- Auto-rotation: keep the newest ~4 backups, delete older ones
- Stop the game server before backing up (or re-add the panel stop schedule and
  back up after it)
- Offline USB backup rotation for the Minecraft world
- Offsite Minecraft world backup via rclone
- Test a backup restore end-to-end (an untested backup is just a hope)
- **Azure Blob Cold tier for Immich photos** — DECIDED (500GB, Cold, LRS, Indonesia
  Central), replaces Google Photos as the offsite leg before it expires Oct 2026.
  Config still pending: storage account → container → access key → rclone remote.

**Reliability / hardware:**
- Re-add the Pterodactyl stop schedule (warn ~02:45, Power: Stop at 02:50)
- Tailscale: **disable key expiry** for `fujikenserver` (admin console → Machines →
  ⋯ → Disable key expiry) so the server doesn't silently drop off after 180 days
- vnStat on the Azure VM (`sudo apt install vnstat`, `vnstat -m`) for real monthly
  egress; optionally add it to the VM `healthcheck`
- Install CyberPower PowerPanel on Ubuntu for automatic graceful shutdown during long brownouts
- `optional: true` in netplan so a missing ethernet cable doesn't hang boot (see Section 8)
- USB-C 3.2 Gen 2 cable to unlock the full 10 Gbps NVMe link (nice-to-have)
- Disable unused default services to trim boot time: `open-vm-tools`, `ModemManager`,
  `cloud-init*`, `multipathd`, `open-iscsi` (low priority)
- Dedicated server box (mini PC, e.g. Ryzen 7 7840HS) so the laptop is freed and uptime improves — planned ~1-2 years
- Proxmox on the dedicated box (1-2 year goal)
- Oracle Cloud Free Tier (24GB ARM) as post-graduation Azure replacement
- LightNode Manila relay for ~20-40 ms ping if PH latency matters
- Full boot automation (Wake-on-LAN, smart plug) — only feasible on a proper home lab box

**Documentation:**
- Refresh the project copy of `fujiken-report.sh` from the live script (secrets redacted)
- Apply the UUID container filter to `fujimon.sh`

**Security hardening (low-effort pentest follow-ups, see Section 9):**
- Add auth to fujiken-monitor (F-002); `server_tokens off;` (F-003); panel security headers (F-004); restrict Azure SSH to home IP or Tailscale-only (F-005)
- Enable 2FA/TOTP on the Pterodactyl panel
- Back up the Azure `.pem` into the password manager

**Services being explored (no commitment):**
- Pi-hole for network-wide ad blocking — wants a dedicated always-on device (not
  Fujiken, which shuts down nightly) with a fallback DNS so the network survives the shutdown
- Wazuh HIDS — high career value for cloud security; planned post-graduation on the
  Oracle Cloud Free Tier ARM instance as the manager (not on Fujiken or the 1GiB Azure VM)
- Personal tempmail server — must live on the Azure VM (not Fujiken, due to nightly
  shutdown) and needs a purchased domain for MX records (free DDNS won't work). No domain bought yet
- HTTPS + auth on the panel if remote admin access is ever needed
- Tailscale node-sharing for a trusted friend to access the panel remotely (idea only, not set up)
- Velocity proxy for seamless `/server` switching between multiple worlds
- Grafana + Prometheus if you want graphs instead of `fujimon`/`fujiken-monitor`

---

## 13. Quick command reference

```bash
# Status
healthcheck                         # full report (laptop or VM)
fujimon                             # live dashboard (laptop)
sensors | grep Tctl                 # CPU temp only
tailscale status                    # tailnet devices

# Timezone (verify after any reinstall)
timedatectl                         # should show Asia/Manila (PST, +0800)
sudo timedatectl set-timezone Asia/Manila  # fix if wrong, then restart wings

# Telegram reports (manual trigger from the laptop)
fujiken-report status               # send status now
fujiken-report daily                # send electricity report now
# Or in Telegram: status, report, restart-wings, restart-frpc, restart-panel, backup-now, reboot

# Services (laptop)
sudo systemctl status frpc wings nginx minecraft-local fujiken-bot
sudo systemctl restart frpc         # if tunnel drops
sudo systemctl restart wings        # panel console spinner / Wings CPU stuck
sudo systemctl restart minecraft-local  # if LAN forward drops

# Services (Azure VM)
sudo systemctl status frps
ls /var/run/reboot-required         # reboot pending?

# SSH into the Azure VM (Windows PowerShell)
ssh -i "F:\azure key\<keyname>.pem" <user>@fujiken.indonesiacentral.cloudapp.azure.com

# Tunnel logs
sudo journalctl -u frpc -n 20 --no-pager   # laptop
sudo journalctl -u frps -n 20 --no-pager   # VM

# Container IP (if connections break)
sudo docker inspect $(sudo docker ps -q) | grep '"IPAddress"' | grep -v '""'

# Backups
ls -lh /home/fujiken/backups
sudo crontab -l                     # confirm all crontab jobs

# Hardware
lsusb -t                            # NVMe link: uas, 5000M = 5 Gbps
ip link                             # laptop MAC (for router reservation)

# Power log
tail -50 /var/log/fujiken-ppt.log

# Graceful shutdown (before unplugging NVMe)
sudo shutdown now
```
