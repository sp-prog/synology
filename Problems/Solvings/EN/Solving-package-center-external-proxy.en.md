---
title: Reach Package Center via an external HTTP proxy
date: 2026-05-09
status: applied
hardware: DS218play
dsm: 7.3.2-86009 Update 3
tags:
  - synology
  - dsm-7
  - package-center
  - proxy
  - dpi
  - howto
  - solving
related:
  - "[[ADR-0003-package-center-via-external-proxy.en]]"
  - "[[Solving-package-center-ocsp-cloudflare-blackhole.en]]"
  - "[[Solving-fix-stale-update-urls-in-synoinfo.en]]"
---

# Reach Package Center via an external HTTP proxy

> [!info] In short
> On networks with DPI filtering (TSPU and equivalents) NAS traffic to Cloudflare is cut by content, so Package Center and `synopkg` hang.
> **Fix:** take a ready-made HTTP proxy from a VPN provider and configure it in **DSM → Control Panel → Network → Connectivity → Proxy**. DSM forwards all outgoing package-layer traffic through the proxy.
> The whole procedure is performed in the DSM GUI; no terminal work is required.
> **Decision:** [[ADR-0003-package-center-via-external-proxy.en]].

## When to apply

✅ Applicable when:
- The network between the NAS and the internet performs **DPI** (typical signature: responses from Cloudflare resources are cut by content; observed during [[ADR-0001-package-center-hangs-on-sectigo-ocsp.en]]).
- Symptom: Package Center hangs / shows "Connection failed".
- A subscription to a commercial VPN service exists with an HTTP/HTTPS proxy mode, or any equivalent ready-made proxy outside the DPI zone.

❌ Not applicable / overkill when:
- The network has no DPI and the problem is something else (see [[Solving-fix-stale-update-urls-in-synoinfo.en]] — a URL fix may be enough).

## What the proxy covers

| What | Through the proxy? |
|---|---|
| Package Center (GUI) | ✅ |
| DSM Update, Synology Account, push notifications, etc. | ✅ — by the principle "the system HTTP proxy in DSM applies to all outgoing connections of DSM services" |
| File shares (SMB/AFP/NFS) to LAN clients | ❌ — direct |
| DSM web UI to LAN clients | ❌ — direct |
| Hyper Backup to a cloud target | depends on the destination and the task's proxy support |
| Docker containers (if running) | ❌ — proxy must be configured per container |

In other words, the local network and how clients use the NAS are unaffected. The proxy applies only to **outgoing** traffic of DSM services.

## Step 1. Obtain HTTP-proxy parameters from the VPN provider

Most paid VPN services expose, in addition to the regular VPN tunnel, an **HTTP/HTTPS proxy**. It is a separate mode, usually labelled "HTTP/SOCKS proxy", "Proxy servers", "Browser proxy", or similar in the provider's dashboard. Any such service is suitable as long as it:

- offers an **HTTP CONNECT proxy** (SOCKS alone is not enough — DSM's setting does not support SOCKS);
- has its server **outside the DPI zone** (a country with an open internet);
- supports **login/password authentication** (or IP whitelisting).

What is needed from the VPN provider's dashboard:
- proxy server address: `host` or `IP`;
- port (typically `8080`, `3128`, `8888`, `1080`);
- login and password.

> [!note] Alternative — a self-hosted proxy on a VPS
> Instead of a ready-made VPN-provider proxy, a self-hosted HTTP proxy (`tinyproxy`, `3proxy`, `squid`) can be set up on a cheap VPS outside the DPI zone. Setup documentation is on those projects' websites; it is not reproduced here. The parameters (host / port / login / password) for Step 2 are identical in either case.

## Step 2. Configure the proxy in DSM

DSM in the browser → **Control Panel → Network → Connectivity → Proxy** (tab):

1. Tick **Use proxy server**.
2. **Address:** the proxy host or IP (no `http://`).
3. **Port:** the proxy port (from the VPN provider's dashboard).
4. **Use the same proxy server for all protocols:** ✅ ticked.
5. **Authentication required:** ✅ ticked → enter login/password.
6. **Bypass proxy server for local addresses:** ✅ (so LAN does not go through the proxy).
7. **Apply** / **OK**.

DSM picks up the settings immediately, no restart needed.

## Step 3. Roll back artefacts of the previous approach

If the blackhole/Cloudflare fix from [[Solving-package-center-ocsp-cloudflare-blackhole.en]] was applied earlier — go through the rollback checklist in [[Solving-package-center-ocsp-cloudflare-blackhole.en#Rollback]]: remove the related Task Scheduler tasks, set MTU back to Auto, restore system files from backups.

Otherwise leftover blackhole routes and overrides may conflict with the proxy and produce symptoms similar to the original problem.

If the previous approach was never applied — this step can be skipped.

## Step 4. Verify via the Package Center GUI

In the browser → close the DSM tab entirely → open DSM → **Package Center**.

Verification cues:
- The window opens **without** the "Connection failed" banner.
- The **All Packages** tab loads the Synology catalogue.
- The **Installed** tab shows the correct state of installed packages (with update markers, if any).
- **Settings (gear icon) → Auto Updates → Check Now** completes without error.

## Reset MTU and DNS if they were changed

If earlier experiments altered any of the following:

- **MTU on eth0** — DSM → Control Panel → Network → Network Interface → LAN 1 → Edit → **MTU value** = `Auto` (or explicitly `1500`).
- **DNS** — DSM → Control Panel → Network → General → **Manually configure DNS server** → restore the router address `192.168.1.1`. Public DNS (`1.1.1.1` + `8.8.8.8`) also works and does not affect the proxy.

## Rollback

To disable the proxy and restore direct DSM connections:

DSM → **Control Panel → Network → Connectivity → Proxy** → **untick** Use proxy server → **Apply**.

That is all. No file edits on the NAS are needed.

## What to do if the proxy stops working

Symptom: Package Center hangs again or returns "Connection failed".

First — **through the DSM GUI**:
1. **Control Panel → Network → Connectivity → Proxy** — confirm the proxy setting is still in place (the **Use proxy server** tick is on, address and port match what the VPN provider currently issues).
2. **The VPN provider's dashboard** — check subscription status, the current proxy address, and server availability. If the provider has rotated the address, update it in DSM.
3. If the VPN provider exposes multiple servers, try another one (another country/city) and update the address in DSM. This is especially relevant if the previous IP has been DPI-blocked.

After any DSM-side change — re-open Package Center in the browser (close the tab and open it again) and repeat Step 4.

## Technical notes

### Why the proxy works where blackhole did not

The principle: the DSM proxy setting is applied **at the application layer** — DSM services (including Package Center and the package daemon) make their outgoing HTTP/HTTPS requests by first establishing a TLS tunnel to the proxy server, and then talk to the Synology / Cloudflare servers through it.

- The ISP DPI sees only the encrypted tunnel `<NAS> → <proxy>:<port>`.
- The SNI to Cloudflare and the actual exchange with Synology servers happen **from the proxy**, which is outside the DPI zone.
- Content-based DPI does not help — the content inside the HTTPS tunnel is unavailable for analysis.

This is fundamentally different from local-side attempts to override routes or /etc/hosts (see [[ADR-0001-package-center-hangs-on-sectigo-ocsp.en]] — that document explains why such approaches fall short against DPI applied to the TLS payload).

## Links

- ADR for this guide: [[ADR-0003-package-center-via-external-proxy.en]]
- Previous (deprecated) guide via blackhole + its rollback: [[Solving-package-center-ocsp-cloudflare-blackhole.en]]
- Adjacent (still applicable) task — URLs in `synoinfo.conf`: [[Solving-fix-stale-update-urls-in-synoinfo.en]]
- DSM Help, Proxy: Control Panel → Network → Connectivity → ? (built-in help).
