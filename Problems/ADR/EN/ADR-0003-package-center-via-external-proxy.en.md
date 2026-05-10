---
id: ADR-0003
title: Reach the Synology back-end for Package Center via an external HTTP proxy
status: accepted
supersedes: ADR-0001
date: 2026-05-09
deciders: home-admin
tags:
  - synology
  - ds218play
  - dsm-7.3
  - networking
  - proxy
  - dpi
  - cloudflare
  - adr
related:
  - "[[ADR-0001-package-center-hangs-on-sectigo-ocsp.en]]"
  - "[[ADR-0002-stale-update-urls-in-synoinfo-after-dsm-upgrade.en]]"
  - "[[Solving-package-center-external-proxy.en]]"
---

# ADR-0003 — Reach Package Center via an external HTTP proxy

> [!summary]
> The root cause of the hanging Package Center on a DS218play is **DPI filtering of Cloudflare traffic** by the ISP (presumably TSPU). Synology has moved part of its infrastructure (`pkgautoupdate.synologyupdate.com`) onto Cloudflare with no AWS fallback. Local-side workarounds (blackhole routes, `/32` exceptions, MTU) yield only partial effect — DPI cuts the TLS stream by content.
> Decision: route the outgoing traffic of DSM services through a ready-made **HTTP proxy** outside the DPI zone (provided by a commercial VPN service). Configured in **DSM → Control Panel → Network → Connectivity → Proxy**.

## Context

### Limits of the previous approach
[[ADR-0001-package-center-hangs-on-sectigo-ocsp.en]] assumed that the `synopkg`/Package Center hang was caused by an OCSP check on the `update7.synology.com` certificate — Sectigo's OCSP responder lives on Cloudflare, the ISP throttles Cloudflare, TLS waits for an OCSP timeout. The `ip route blackhole` solution removed the OCSP symptom but:

1. Did not cover `pkgautoupdate.synologyupdate.com` — that domain is **legitimately required** by the Package Center GUI (302 redirect from `pkgupdate7.synology.com`) and lives **entirely on Cloudflare**.
2. Narrow `/32` exceptions for individual IPs do not survive Cloudflare IP rotation.
3. Most importantly — DPI cuts the TLS stream **by content**, not by address: short HEAD requests to Cloudflare resources go through, larger responses (over ~16 KB) get truncated. Lowering MTU does not help.

Routing-level workarounds do not work against content-based filtering.

### What needs to be solved
- `synopkg` (CLI) and the DSM Package Center (GUI) must reach the Synology servers, including the Cloudflare-hosted resources (`pkgautoupdate.synologyupdate.com` and the Sectigo OCSP/CRL).
- The solution must survive IP rotation, reboots and DSM updates.
- It must not require ongoing maintenance of iptables / routes / `/etc/hosts`.

### Constraints
- The NAS has no native WireGuard, Shadowsocks or Tor support.
- Docker does not run on DS218play (RTD1296 is unsupported).
- DSM **does** ship a built-in HTTP CONNECT proxy setting that covers all outgoing traffic of its own services (Package Center, DSM Update, Synology Account, push). It is just a setting, no installation needed.
- A ready-made HTTP proxy is available from a commercial VPN service (most paid VPNs expose this mode in their dashboard). Alternatively — a self-run proxy on a VPS outside the DPI zone.

## Options considered

### Option A — `ip route blackhole` + `/32` for Cloudflare IPs
- **Pros:** local-only, no external infrastructure.
- **Cons:** does not cover `pkgautoupdate.synologyupdate.com`, IP rotation, no defense against content-based DPI.
- **Decision:** **superseded**, see [[ADR-0001-package-center-hangs-on-sectigo-ocsp.en]].

### Option B — VPN client on the NAS (OpenVPN / L2TP)
- **Pros:** DSM supports it out of the box (Network Interface → Create VPN profile).
- **Cons:**
  - Run as **default route through VPN** — **all** NAS traffic goes through the VPN, slowing local operations (backups, media server, file shares) and burning the VPS quota.
  - Run with **partial routing** (only Cloudflare/Sectigo via VPN) — DSM's GUI does not allow that; routing tables would have to be edited by hand on every reconnect.
  - DSM 7.3 has no native WireGuard — only OpenVPN/L2TP, both heavier.
  - Holding a permanent VPN tunnel on a DS218play (1 GB RAM, ARMv8) is unnecessary load.
- **Decision:** rejected as overkill.

### Option C — HTTP CONNECT proxy declared in DSM settings ✅
- **Pros:**
  - **Built into DSM:** Control Panel → Network → Connectivity → Proxy. No scripts, routes or files.
  - Applies only to outgoing DSM-service traffic (Package Center, DSM Update, Synology Account, Active Insight). Local shares/backups/media server stay direct.
  - The proxy source can be an off-the-shelf service — the HTTP-proxy mode included in a commercial VPN subscription. No infrastructure to deploy.
  - Indifferent to Cloudflare IP rotation, OCSP infrastructure, and changes in Synology infrastructure.
  - Survives reboots — the DSM proxy setting persists like any other network setting.
- **Cons:**
  - Dependent on the proxy service availability and the VPN subscription term.
  - If the proxy IP is filtered, a different server has to be picked (VPN services usually have a pool of addresses).
- **Alternative within the same option:** run a self-hosted HTTP proxy (`tinyproxy`, `3proxy`, `squid`) on a cheap VPS outside the DPI zone. The DSM-side parameters (host/port/login/password) remain identical.
- **Decision:** **accepted** in the off-the-shelf VPN-provider proxy form.

### Option D — Cloudflare WARP in Docker
- **Pros:** free.
- **Cons:** Docker does not run on DS218play; WARP is itself often filtered under active DPI; running a second network stack on 1 GB RAM is heavy.
- **Decision:** rejected.

### Option E — obfuscated VPN (AmneziaWG, XRay/XTLS-Reality)
- **Pros:** designed specifically to bypass DPI.
- **Cons:** on the NAS only via Docker / Entware build, hard to maintain, heavier than a plain HTTP proxy.
- **Decision:** deferred. Should be revisited if the proxy option ever stops working (i.e. DPI starts cutting the VPS IPs themselves).

## Decision

Apply option **C** with a **ready-made HTTP proxy from a commercial VPN service**.

The alternative implementation (a self-hosted HTTP proxy on a VPS, e.g. `tinyproxy`) is functionally equivalent — the DSM-side configuration steps are identical.

Steps:
1. Obtain HTTP-proxy parameters from the VPN provider (host, port, login, password).
2. Configure the proxy in **DSM → Control Panel → Network → Connectivity → Proxy**, with authentication.
3. Roll back the ADR-0001 artefacts (blackhole routes, `/32`, MTU, Task Scheduler tasks) — see [[Solving-package-center-ocsp-cloudflare-blackhole.en#Rollback]].

Detailed procedure — [[Solving-package-center-external-proxy.en]].

## Consequences

### Positive
- Package Center in the GUI reaches the Synology servers (including Cloudflare resources) through the proxy tunnel.
- The solution does not depend on Synology infrastructure (Cloudflare/AWS, OCSP servers, IP rotation).
- No scripts, Task Scheduler tasks, or periodic IP-list updates required.
- By design DSM's system-proxy setting is applied to all outgoing connections of DSM services (DSM Update, Synology Account, Active Insight, push notifications).

### Neutral
- Recurring cost of the VPN subscription (typically already paid for other purposes).
- A persistent setting becomes visible in **Control Panel → Network → Connectivity → Proxy**.

### Negative / risks
- **Dependence on the external proxy:** if the VPN service goes down or the subscription ends, Package Center breaks again. Mitigation: keep a backup proxy (a second VPN service).
- **Dependence on the proxy IP not being DPI-blocked.** If it ends up on a blocklist, the address has to be swapped. VPN providers usually offer a pool of addresses.
- **The proxy sees all DSM HTTPS traffic** (as encrypted bytes — it cannot decrypt without MitM). Still, the proxy provider can technically log destination domains through SNI. Acceptable risk for a home NAS sending public Synology API queries.
- **The proxy does not cover Docker containers** (irrelevant on DS218play — Docker does not run) or third-party processes that read proxy settings **outside** the DSM config. For example, direct HTTPS calls from a Python script in a Task Scheduler job require a separate proxy declaration.

### Out of scope
- Access to torrent trackers / social networks / other restricted resources: this configuration is for DSM only. The rest needs a router-level VPN or dedicated clients.
- Access from apps that ignore the system proxy (some Synology mobile apps such as DS file may bypass it).

## Verification cues

After the proxy is applied:
- Package Center in the GUI opens without the "Connection failed" banner.
- The "All Packages" and "Installed" tabs load the catalogue and the state of installed packages.
- **Settings → Auto Updates → Check Now** completes without error.

Symptoms typical of the unfixed problem (for "before/after" comparison):
- In the GUI — "Connection failed" banner, the package list does not load.
- In the CLI — `synopkg checkupdateall` times out (`exit=124`, ~30 s wait).

## Links

- Procedure: [[Solving-package-center-external-proxy.en]]
- Rollback of the previous approach: [[Solving-package-center-ocsp-cloudflare-blackhole.en#Rollback]]
- Related (still applicable) task: [[ADR-0002-stale-update-urls-in-synoinfo-after-dsm-upgrade.en]]
- DSM: Control Panel → Network → Connectivity → Proxy (built-in setting).
