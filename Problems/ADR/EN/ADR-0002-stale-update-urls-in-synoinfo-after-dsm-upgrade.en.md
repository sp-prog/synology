---
id: ADR-0002
title: Reconcile stale update URLs in /etc/synoinfo.conf after a DSM upgrade
status: accepted
date: 2026-05-09
deciders: home-admin
tags:
  - synology
  - ds218play
  - dsm-7.3
  - synoinfo
  - upgrade
  - configuration
  - adr
related:
  - "[[Solving-fix-stale-update-urls-in-synoinfo.en]]"
  - "[[ADR-0001-package-center-hangs-on-sectigo-ocsp.en]]"
---

# ADR-0002 — Reconcile stale update URLs in `/etc/synoinfo.conf` after a DSM upgrade

> [!summary]
> On a DS218play (DSM 7.3.2-86009 Update 3) the active `/etc/synoinfo.conf` retains **legacy** package-server URLs (`pkgupdate.synology.com`, `update.synology.com`, `small_info_path` without the `7`) after a major-version DSM upgrade, while the factory defaults in `/etc.defaults/synoinfo.conf` are already updated to `pkgupdate7.synology.com` / `update7.synology.com`. The decision is to selectively update only the diverging keys with `synosetkeyvalue`, leaving user settings untouched.

## Context

### Symptom
While diagnosing a hanging Package Center (see [[ADR-0001-package-center-hangs-on-sectigo-ocsp.en]]), it was found that `/etc/synoinfo.conf` carries URLs like `pkgupdate.synology.com` and `update.synology.com`, whereas the current DSM 7.x ones include the digit `7` in the host name.

### Divergence found on the actual system
DSM 7.3.2-86009 Update 3, DS218play.

| Key | `/etc/synoinfo.conf` (live) | `/etc.defaults/synoinfo.conf` (factory) |
|---|---|---|
| `pkgupdate_server` | `https://pkgupdate.synology.com` ❌ | `https://pkgupdate7.synology.com` ✅ |
| `rss_server` | `http://update.synology.com/autoupdate/genRSS.php` ❌ | `http://update7.synology.com/autoupdate/genRSS.php` ✅ |
| `rss_server_ssl` | `https://update.synology.com/autoupdate/genRSS.php` ❌ | `https://update7.synology.com/autoupdate/genRSS.php` ✅ |
| `update_server` | `http://update.synology.com/` ❌ | `http://update7.synology.com/` ✅ |
| `small_info_path` | `http://update.synology.com/smallupdate` ❌ | `https://update7.synology.com/smallupdate` ✅ |

Less critical (cosmetic) divergences were also found:

| Key | `/etc/synoinfo.conf` | `/etc.defaults/synoinfo.conf` |
|---|---|---|
| `online_help_base_url` | `http://www.synology.com/help/uihelp/` | `http://help.synology.com/` |
| `printer_driver_host` | `http://download.synology.com/airprint/DSM4.1` | `https://global.download.synology.com/airprint/DSM7.3/latest` |
| `frame_options_built_in_allow_url` | (missing) | `gofile.me/,find.synology.com/` |

And **legacy keys** that are no longer in the factory defaults:

| Key | `/etc/synoinfo.conf` | Factory |
|---|---|---|
| `pushservice_server_1` | `https://sns1.synology.com:8089/api/` | (missing) |
| `pushservice_server_2` | `https://sns2.synology.com:8089/api/` | (missing) |

### Why this happens
On a major-version upgrade (6.x → 7.x, and in some cases between 7.x minors) DSM updates **`/etc.defaults/synoinfo.conf`** but does **not** rewrite the active `/etc/synoinfo.conf` wholesale — to avoid clobbering user settings. Per-key migration is implemented inside upgrade scripts; for some keys the migration is either not provided or did not run.

In the case at hand the upgrade path likely started from DSM 6.x (the DS218play shipped with DSM 6.x), so `pkgupdate_server` / `update_server` were carried over with their old values.

### Impact
- `synopkg` and `synoupgrade` query stale hosts. On DSM 7.3 the legacy `pkgupdate.synology.com` (without `7`) returns **HTTP 404** from CloudFront in milliseconds — that is, fast but with no useful payload. Package Center then sees an empty/incorrect response rather than a hardware failure.
- The fix surfaced during the diagnosis of [[ADR-0001-package-center-hangs-on-sectigo-ocsp.en]] and was applied to remove noise from further diagnostics. Whether stale URLs contribute to the main hang (which is caused by a separate factor — DPI filtering of Cloudflare, see [[ADR-0003-package-center-via-external-proxy.en]]) was not investigated. The fix is correct as a stand-alone task: on DSM 7.x all update URLs are expected to contain `7`.

## Options considered

### Option A — overwrite `/etc/synoinfo.conf` entirely from `/etc.defaults/synoinfo.conf`
- **Pros:** one command synchronises everything.
- **Cons:** **wipes user settings** (UPnP model, SMB parameters, UI customisations, ActiveInsight telemetry, regional settings). The file holds hundreds of keys, not all of them URLs.
- **Decision:** rejected as destructive.

### Option B — selectively update the diverging URLs via `synosetkeyvalue` ✅
- **Pros:**
  - Only the needed keys are changed.
  - Uses the native Synology utility — preserves quoting and format correctly.
  - Backup is a single `cp` command.
  - Rollback by restoring the backup.
- **Cons:**
  - The list of diverging keys must be assembled manually (via `diff`).
  - Does not cover legacy keys that **exist in the live config but are removed from the defaults** (they need separate cleanup or can be ignored as harmless).
- **Decision:** **accepted.**

### Option C — Reset Network/Services via DSM GUI
- **Pros:** official Synology mechanism, safe for user data.
- **Cons:** resets a large bundle of settings at once (network, firewall, authentication). Excessive for a URL issue.
- **Decision:** rejected as overkill.

### Option D — ignore
- **Pros:** do nothing.
- **Cons:** Package Center and `synoupgrade` keep misbehaving. A core NAS function stays broken.
- **Decision:** rejected.

## Decision

Apply option **B**: selectively update the URL keys via `synosetkeyvalue`, taking values from the **same system's** `/etc.defaults/synoinfo.conf` (not from external sources — this avoids mismatches with DSM version/region).

Mandatory keys to update (when divergent):
1. `pkgupdate_server`
2. `rss_server`
3. `rss_server_ssl`
4. `update_server`
5. `small_info_path`

Optional (cosmetic):
6. `online_help_base_url`
7. `printer_driver_host`
8. `frame_options_built_in_allow_url`

Legacy keys (`pushservice_server_1/2`) — **leave alone**: re-adding them is not what DSM expects, and removing may be risky. They are unused by the current system and harmless.

A backup of `/etc/synoinfo.conf` is mandatory before any change.

Detailed procedure: [[Solving-fix-stale-update-urls-in-synoinfo.en]].

## Consequences

### Positive
- `synopkg checkupdate*` and `synoupgrade --check` start hitting the current servers.
- Removes one class of phantom Package Center hangs.
- Less noise in the logs — 404s and timeouts to the old hosts disappear.

### Neutral
- A `/etc/synoinfo.conf.bak.<timestamp>` backup file is created — a few kilobytes.

### Negative / risks
- **May recur after the next major DSM update.** If DSM 8 moves from `update7` to `update8`, the config will diverge again. Mitigation: run the diff check on every major DSM upgrade (see the verification step in the Solving guide).
- **`synosetkeyvalue` does not validate URLs.** A typo in the command yields a broken config. Mitigation: backup + a `grep` check after every change.

## Verification cues

- Before the fix: `curl -sI https://pkgupdate.synology.com/` responds (404), but `synopkg` uses the legacy host and behaves unpredictably.
- After the fix: `grep -E '^(pkgupdate_server|rss_server|update_server|small_info_path)' /etc/synoinfo.conf` shows all lines containing `update7` / `pkgupdate7`.
- A `diff` against `/etc.defaults/synoinfo.conf` on the URL keys is empty (apart from the deliberately retained legacy keys).

## Related decisions

- [[ADR-0001-package-center-hangs-on-sectigo-ocsp.en]] — root cause of Package Center hangs; the ADR-0002 fix is necessary but not sufficient.
- [[Solving-fix-stale-update-urls-in-synoinfo.en]] — step-by-step guide for applying this ADR.
- [[Solving-package-center-ocsp-cloudflare-blackhole.en]] — related (deprecated) OCSP-fix guide.
