# Checkpoint — CVE-2026-98432 kernel patch (2026-11-10)

## Trigger
Provider (ParsPack) urgent notice: two kernel flaws.
- **CVE-2026-98432** — Linux IPv4 TCP use-after-free (`inet_opt`/`ireq_opt`), CVSS **9.8**, remote + unauthenticated.
  Ubuntu fix for 26.04 LTS (resolute): **7.0.0-43.43** — <https://ubuntu.com/security/CVE-2026-98432>
- KSMBD RCE (CVSS 9.0) — **not applicable on this host**: `ksmbd` module not loaded, no listeners on 139/445.

## Host
`cloudservice-037076` · 62.3.42.228 · Ubuntu 26.04 LTS · nginx → Next.js (`silentshift.service`) → SQLite.

## Before
| Item | Value |
|---|---|
| Running kernel | `7.0.0-38-generic` (**vulnerable**) |
| Installed kernels | `-38`, `-42` |
| Pending upgrades | 67 (kernel meta held by phasing) |
| reboot-required | present |

## Action
1. Backups (root-only):
   - DB: `/root/backups/20261110-233725/app.db` — sha256 `4f9ec431cfc2fe5072522f17151cfb5c0cc395afd5057414771547f9c481f4e2`
   - Configs: `/root/backups/20261110-233725/configs.tar.gz` — sha256 `5c359f7b3e5cbcff13fa9ca68bfad1d4827cdec46d70cee46d76b62881e5d9d2`
     (nginx, systemd unit, `.env`, letsencrypt renewal/archive, iptables rules, sshd drop-ins)
   - `root-crontab.txt` — sha256 `37027f733acc9318c3ca76d5c1cd56864431bdcbfd2d576c14e8bd236157d4e5`
   - Not backed up on purpose: `public/uploads` (assets) — noted, not tarred.
2. `apt-get full-upgrade` with `DEBIAN_FRONTEND=noninteractive`, `NEEDRESTART_SUSPEND=1`,
   `--force-confold`, `APT::Get::Always-Include-Phased-Updates=true` → **exit 0**
   Log: `/root/patch-kernel-20261110-233725.log`
3. Reboot (approved window). GRUB keeps `-42` and `-38`; nothing autoremoved.

## After (verified)
| Check | Result |
|---|---|
| Running kernel | **`7.0.0-43-generic`** (≥ fixed `43.43`) |
| reboot-required | cleared |
| silentshift / nginx / ssh | active **and** enabled |
| `nginx -t` | ok |
| `/api/health` | `ok:true, db:true` |
| `https://silentshift.ir` | 200 (valid TLS) · `www` 301 · `.life` 301 → `.ir` |
| DDL row counts | podcasts 7 · articles 7 · categories 3 · tags 8 · faqs 5 · content 5 · messages 1 · subscribers 1 · admins 1 |
| Pending upgrades | 0 |
| errors since boot (project units) | none |
| Failed units | none |
| Port 3000 from internet | closed (iptables DROP persisted) |

**Boot note:** the app needs ~19 s after boot (`seed` → `next start`), so probes inside that window return 502. Not a failure; consider a longer `start_period` if a healthcheck is ever added.

## Rollback
1. **Kernel:** GRUB → *Advanced options for Ubuntu* → `7.0.0-42-generic` (or `-38`). Both remain installed.
   Persist: `grub-reboot "1>2"` (pick the entry) or edit `GRUB_DEFAULT`; `update-grub`.
2. **Data:** `systemctl stop silentshift` → copy `/root/backups/20261110-233725/app.db` over `/opt/silentshift/data/app.db` (remove `-wal`/`-shm` first) → `systemctl start silentshift`.
3. **Config:** `tar xzf /root/backups/20261110-233725/configs.tar.gz -C /` → `nginx -t && systemctl reload nginx`.
4. **Verify:** same table as above.

## Residual risk
- `7.0.0-42` (vulnerable) is still installed for rollback; remove `linux-image-7.0.0-42-generic` once `-43` has proven stable for a few days.
- No provider VM snapshot was taken or confirmed — recovery rests on the dumps above (they live on the same disk; copy them off-host when convenient).
- `uploads/` assets have no backup (only this app's DB/config are covered).
- Unrelated: the old OVH VPS (51.255.167.150) is still «مسدود» in the IranServer panel and refuses SSH (host-key mismatch) — untouched.
