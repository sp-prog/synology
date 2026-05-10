---
title: (DEPRECATED) Fix hanging Package Center via Cloudflare blackhole routes
date: 2026-05-09
status: deprecated
superseded_by: "[[Solving-package-center-external-proxy]]"
hardware: DS218play
dsm: 7.3.2-86009 Update 3
tags:
  - synology
  - ds218play
  - dsm-7.3
  - package-center
  - synopkg
  - networking
  - tls
  - ocsp
  - cloudflare
  - howto
  - solving
  - deprecated
related:
  - "[[ADR-0001-package-center-hangs-on-sectigo-ocsp]]"
  - "[[ADR-0003-package-center-via-external-proxy]]"
  - "[[Solving-package-center-external-proxy]]"
---

# (DEPRECATED) Cloudflare blackhole

> [!danger] This guide is deprecated
> On networks with **DPI filtering** (TSPU and equivalents) the approach delivers a **partial and unstable** result:
> - CLI `synopkg checkupdateall` works intermittently;
> - Package Center GUI either shows "Connection failed" or just spins;
> - after a reboot routes have to be re-applied by hand; keepalive does not cover every case.
>
> **Working solution** — an external HTTP proxy outside the DPI zone: see [[Solving-package-center-external-proxy]].
>
> This document is kept for:
> 1. History and **rollback** steps (in case the approach was already applied and needs to be undone).
> 2. Future readers, so they do not retrace the same path.

## When the approach could in principle work

If the network does **not** have DPI on Cloudflare traffic but Cloudflare IPs are blocked by the act of establishing a connection (e.g. specific subnets are not routed by the ISP), blackhole with narrow `/32` exceptions **might** help. On networks with active DPI it does not.

## Why it fails under DPI

The DPI signature: `curl -sI` (a ~700-byte HEAD) succeeds; a `curl GET` for the package list is truncated **at exactly 16384 bytes** (`/tmp/getlist.out` is exactly 16 KB). Typical behaviour for a filter that lets through the TCP handshake and the first window of encrypted data, then silently drops further packets. Routing and MTU do not help here — the filter is content-based, not address-based.

Detailed analysis — [[ADR-0001-package-center-hangs-on-sectigo-ocsp#Why the solution does not work under DPI]].

## What the approach does (did) — in short

1. Updates URLs in `/etc/synoinfo.conf` (useful by itself — see the standalone [[Solving-fix-stale-update-urls-in-synoinfo]]).
2. Cleans third-party feeds in `/usr/syno/etc/packages/feeds`.
3. Installs `ip route add blackhole` for 4 Cloudflare subnets (`104.18.0.0/16`, `104.26.0.0/16`, `172.64.0.0/16`, `172.67.0.0/16`).
4. Adds `/32` exceptions via the gateway for the IPs of `pkgautoupdate.synologyupdate.com`.
5. Persists everything through the DSM Task Scheduler (Boot-up + Scheduled keepalive).
6. Optionally lowers the MTU to 1400 (an attempt to bypass PMTU black holes — useless under DPI).

## Rollback

If the approach was previously applied and the system needs to return to its original state — perform the actions below by hand. Cross-check against what was actually done on the specific system; skip items that were not applied.

### Through the DSM GUI

1. **Control Panel → Task Scheduler** — remove every task related to this approach (likely names: `Cloudflare blackhole for OCSP`, `Cloudflare blackhole — boot`, `Cloudflare blackhole — keepalive`).
2. **Control Panel → Network → Network Interface → LAN 1 → Edit → MTU value** — if it was set manually, restore `Auto` (or `1500`).
3. **Control Panel → Network → General → DNS** — if a public DNS was set, optionally revert to the router address.
4. **Package Center → Settings → Package Sources** — if third-party sources were removed during diagnosis (e.g. SynoCommunity), add them back via **Add** if needed.

### Through SSH (only if system files were edited)

Backups were created with the `.bak.<tag>` suffix next to the original. The general restore pattern:

```
ls -t <file>.bak.*
cp <file>.bak.<tag> <file>
```

Files that may have been modified by this approach:
- `/etc/synoinfo.conf` — update-URL edits (see also [[Solving-fix-stale-update-urls-in-synoinfo]]).
- `/usr/syno/etc/packages/feeds` — list of third-party repositories.
- `/etc/hosts` — overrides for the Sectigo OCSP/CRL hosts.

Kernel routes (`ip route blackhole ...`, `ip route add IP/32 ...`) and `ip link set eth0 mtu ...` do not persist and disappear automatically on reboot.

After any system-file change — restart the package daemon:
```
synosystemctl restart synosmpkgd
synosystemctl restart synoscgi
```

For a fully clean state — reboot the NAS.

## What to do instead

Switch to the working solution via an external proxy: [[Solving-package-center-external-proxy]].

## What remains valid from this branch of the investigation

- Restoring up-to-date URLs in `/etc/synoinfo.conf` — a stand-alone task independent of blackhole. See [[Solving-fix-stale-update-urls-in-synoinfo]].
- Cleaning / restoring third-party feeds through the GUI or SSH — a standard procedure, no workarounds required.

## Links

- ADR (with the analysis of why the approach failed): [[ADR-0001-package-center-hangs-on-sectigo-ocsp]]
- Replacement ADR: [[ADR-0003-package-center-via-external-proxy]]
- Replacement Solving: [[Solving-package-center-external-proxy]]
- Related (valid) URL task: [[Solving-fix-stale-update-urls-in-synoinfo]]
