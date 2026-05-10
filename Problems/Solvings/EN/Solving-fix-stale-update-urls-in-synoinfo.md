---
title: Fix stale update URLs in /etc/synoinfo.conf after a DSM upgrade
date: 2026-05-09
status: applied
hardware: DS218play (applies to any Synology)
dsm: 7.3.2-86009 Update 3 (applies to any DSM 7.x after an upgrade from 6.x)
tags:
  - synology
  - dsm-7
  - synoinfo
  - upgrade
  - configuration
  - howto
  - solving
related:
  - "[[ADR-0002-stale-update-urls-in-synoinfo-after-dsm-upgrade]]"
  - "[[ADR-0001-package-center-hangs-on-sectigo-ocsp]]"
  - "[[Solving-package-center-ocsp-cloudflare-blackhole]]"
---

# Fix stale update URLs in `/etc/synoinfo.conf`

> [!info] In short
> After a DSM 6.x → 7.x upgrade (and sometimes between 7.x minors) the active `/etc/synoinfo.conf` retains legacy package-server URLs (`pkgupdate.synology.com`, `update.synology.com`, `small_info_path` without the `7`). The factory defaults in `/etc.defaults/synoinfo.conf` already point to `pkgupdate7.synology.com` / `update7.synology.com`.
> **Fix:** diff the two files and selectively rewrite the diverging URL keys with `synosetkeyvalue`.
> **Decision:** [[ADR-0002-stale-update-urls-in-synoinfo-after-dsm-upgrade]]

## Symptoms

- Updates fail to arrive in Package Center, or arrive slowly / partially.
- `synopkg checkupdate*` behaves unpredictably — sometimes responds, sometimes hangs.
- `synoupgrade --check` finds no DSM updates even though some exist.
- Logs in `/var/log/messages` and `/var/log/synolog/synopkg.log` reference `update.synology.com` / `pkgupdate.synology.com` (without `7`).
- `curl -I https://pkgupdate.synology.com/` returns 404 from CloudFront — the legacy host exists but does not serve the DSM 7.x API.

## When to apply

✅ DSM 7.x on any Synology model, especially if the NAS was upgraded from DSM 6.x.

❌ Do not apply on DSM 6.x without a diff check — there the URLs without `7` are the correct ones.

❌ Do not apply blindly from a third-party URL list — always cross-check against **your own** `/etc.defaults/synoinfo.conf`. DSM versions can carry different values (for instance, DSM 8 will likely default to `update8.synology.com`, not `update7`).

## Step 0. SSH access

DSM → **Control Panel → Terminal & SNMP → Enable SSH service**.

```bash
ssh admin@<NAS-IP>
sudo -i
```

## Step 1. Check whether divergences exist

```bash
echo "=== synology.com hosts in the live config ==="
grep -E '\.synology\.com' /etc/synoinfo.conf | sort

echo
echo "=== synology.com hosts in the factory defaults ==="
grep -E '\.synology\.com' /etc.defaults/synoinfo.conf | sort

echo
echo "=== DIFF (live vs factory) ==="
diff <(grep -E '\.synology\.com' /etc/synoinfo.conf | sort) \
     <(grep -E '\.synology\.com' /etc.defaults/synoinfo.conf | sort)
```

Reading the `diff`:
- **`<` lines** — present in the live file but different from the factory copy.
- **`>` lines** — what the factory copy has, and what the live file should match.

If the diff is empty — there are no divergences and no fix is needed.

If divergences exist — proceed to Step 2.

## Step 2. Backup

```bash
cp /etc/synoinfo.conf /etc/synoinfo.conf.bak.$(date +%Y%m%d-%H%M)
ls -la /etc/synoinfo.conf*
```

The backup stays in the same directory. To roll back — `cp ... /etc/synoinfo.conf` over the live file.

## Step 3. Update the URL keys

> [!warning]
> Take the values from the Step 1 `diff` output — specifically the `>` lines (these are the values from **your** `/etc.defaults/synoinfo.conf`). Do not blindly copy values from this guide — they may be stale.

Command template:
```bash
synosetkeyvalue /etc/synoinfo.conf <key> "<new_value>"
```

### Mandatory keys (update URLs)

```bash
synosetkeyvalue /etc/synoinfo.conf pkgupdate_server  "https://pkgupdate7.synology.com"
synosetkeyvalue /etc/synoinfo.conf rss_server        "http://update7.synology.com/autoupdate/genRSS.php"
synosetkeyvalue /etc/synoinfo.conf rss_server_ssl    "https://update7.synology.com/autoupdate/genRSS.php"
synosetkeyvalue /etc/synoinfo.conf update_server     "http://update7.synology.com/"
synosetkeyvalue /etc/synoinfo.conf small_info_path   "https://update7.synology.com/smallupdate"
```

These five keys are critical for Package Center and DSM Update.

### Optional keys (cosmetic)

They do not affect updates, but it makes sense to align them at the same time:

```bash
synosetkeyvalue /etc/synoinfo.conf online_help_base_url             "http://help.synology.com/"
synosetkeyvalue /etc/synoinfo.conf printer_driver_host              "https://global.download.synology.com/airprint/DSM7.3/latest"
synosetkeyvalue /etc/synoinfo.conf frame_options_built_in_allow_url "gofile.me/,find.synology.com/"
```

`online_help_base_url` — destination of the "?" link in DSM. `printer_driver_host` — AirPrint drivers. `frame_options_built_in_allow_url` — the list of domains permitted to embed DSM in an iframe.

> [!note]
> Across DSM versions `printer_driver_host` points to its own version path (`DSM7.0/latest`, `DSM7.2/latest`, `DSM7.3/latest`). Use the value from your own `/etc.defaults/synoinfo.conf`.

### Legacy keys — leave them alone

The live config may keep keys that are **no longer present** in the factory defaults:

```
pushservice_server_1=https://sns1.synology.com:8089/api/
pushservice_server_2=https://sns2.synology.com:8089/api/
```

They should **neither** be added back to the defaults **nor** removed from the live config. Synology has long stopped using them (a single `pushservice_server` is in effect now), and they sit there as harmless dead weight. `synosetkeyvalue` cannot delete keys; manual editing of the file is risky.

## Step 4. Verify what was written

```bash
echo "=== mandatory keys ==="
grep -E '^(pkgupdate_server|rss_server|update_server|small_info_path)=' /etc/synoinfo.conf

echo
echo "=== optional keys ==="
grep -E '^(online_help_base_url|printer_driver_host|frame_options_built_in_allow_url)=' /etc/synoinfo.conf

echo
echo "=== diff again (should shrink) ==="
diff <(grep -E '\.synology\.com' /etc/synoinfo.conf | sort) \
     <(grep -E '\.synology\.com' /etc.defaults/synoinfo.conf | sort)
```

The mandatory keys must contain `update7` / `pkgupdate7`. Only the legacy keys (`pushservice_server_1/2`) should remain in the diff — that is fine.

## Step 5. Reload the config and check the effect

```bash
synosystemctl restart synosmpkgd
sleep 3

# does the package layer hit the new host?
curl -m 10 -sI https://pkgupdate7.synology.com/ ; echo "exit=$?"

# update check
timeout 30 synopkg checkupdateall
echo "exit=$?"
```

Expected:
- `curl ... pkgupdate7.synology.com` → HTTP/2 404 (normal for the root) **in milliseconds**, `exit=0`.
- `synopkg checkupdateall` → `exit=0` in reasonable time.

> [!warning]
> If `synopkg checkupdateall` still hangs — that is a **separate problem** unrelated to stale URLs: ISP-side DPI filtering of Cloudflare traffic. The URL fix does not address it. See [[Solving-package-center-external-proxy]] — the working solution via an external HTTP proxy.

## Rollback

```bash
ls /etc/synoinfo.conf.bak.*
cp /etc/synoinfo.conf.bak.<wanted-tag> /etc/synoinfo.conf
synosystemctl restart synosmpkgd
```

After rollback the `diff` will show the old divergences again — that is expected.

## Optional: automate the diff check after DSM updates

To avoid missing new divergences after major DSM upgrades, schedule a periodic check via Task Scheduler.

DSM → **Control Panel → Task Scheduler → Create → Scheduled Task → User-defined script**:

| Field | Value |
|---|---|
| Task | `Check synoinfo URL drift` |
| User | `root` |
| Schedule | First day of month, 03:00 |
| Run command | see below |

```bash
DRIFT=$(diff <(grep -E '\.synology\.com' /etc/synoinfo.conf | sort) \
             <(grep -E '\.synology\.com' /etc.defaults/synoinfo.conf | sort) \
             | grep -vE 'pushservice_server_[12]=')
if [ -n "$DRIFT" ]; then
  echo "$DRIFT" | mail -s "Synology synoinfo URL drift detected" you@example.com
fi
```

(Replace `you@example.com` or comment out `mail` and log to a file — DSM does not have outgoing mail configured by default.)

## Technical notes

### Why `synosetkeyvalue` and not `sed -i`

`synosetkeyvalue` is the native Synology utility, which:
- handles quoting correctly (preserves `"value"`, unquoted, or single-quoted forms);
- rewrites the file atomically (temp file + `mv`), so a crash mid-write does not leave a corrupt file;
- handles multi-line values cleanly.

`sed -i` against the Synology config can quietly break the file — DSM uses its own parser sensitive to whitespace and quoting.

### Why edits survive a reboot

`/etc/synoinfo.conf` lives on the system volume `/dev/md0` — a real disk, not tmpfs. Edits persist across reboots. They **can** be overwritten only on a major DSM upgrade, and even then Synology typically migrates keys rather than wiping the file.

### `/etc/synoinfo.conf` vs `/etc.defaults/synoinfo.conf`

- `/etc.defaults/synoinfo.conf` — factory settings of the current DSM version. Updated on every system update. **Do not edit by hand.**
- `/etc/synoinfo.conf` — the live user config. Created by copying from the defaults on first install, then leads its own life. Edited via `synosetkeyvalue` or the DSM UI.

The `diff` between them is empty on a fresh install. It diverges over time with upgrades and user changes. The goal of this guide is to align only the URL subset of keys.

## Concrete example (for illustration)

DS218play, DSM 7.3.2-86009 Update 3 (after an upgrade from DSM 6.x):

| Key | Before | After |
|---|---|---|
| `pkgupdate_server` | `https://pkgupdate.synology.com` | `https://pkgupdate7.synology.com` |
| `rss_server` | `http://update.synology.com/autoupdate/genRSS.php` | `http://update7.synology.com/autoupdate/genRSS.php` |
| `rss_server_ssl` | `https://update.synology.com/autoupdate/genRSS.php` | `https://update7.synology.com/autoupdate/genRSS.php` |
| `update_server` | `http://update.synology.com/` | `http://update7.synology.com/` |
| `small_info_path` | `http://update.synology.com/smallupdate` | `https://update7.synology.com/smallupdate` |
| `online_help_base_url` | `http://www.synology.com/help/uihelp/` | `http://help.synology.com/` |
| `printer_driver_host` | `http://download.synology.com/airprint/DSM4.1` | `https://global.download.synology.com/airprint/DSM7.3/latest` |

`pushservice_server_1`, `pushservice_server_2` — left as is (legacy).

## Links

- ADR for this guide: [[ADR-0002-stale-update-urls-in-synoinfo-after-dsm-upgrade]]
- Related problem (OCSP + Cloudflare): [[ADR-0001-package-center-hangs-on-sectigo-ocsp]] and [[Solving-package-center-ocsp-cloudflare-blackhole]]
- `synoinfo.conf` reference: there is no official public documentation — the config is treated as a black box; treat `/etc.defaults/synoinfo.conf` as the source of truth.
