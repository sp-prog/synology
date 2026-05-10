---
id: ADR-0001
title: Bypass the OCSP check on pkgupdate7.synology.com via blackhole routes
status: superseded
superseded_by: ADR-0003
date: 2026-05-09
deciders: home-admin
tags:
  - synology
  - ds218play
  - dsm-7.3
  - networking
  - tls
  - ocsp
  - cloudflare
  - dpi
  - adr
  - superseded
related:
  - "[[Solving-package-center-ocsp-cloudflare-blackhole]]"
  - "[[ADR-0003-package-center-via-external-proxy]]"
  - "[[Solving-package-center-external-proxy]]"
---

# ADR-0001 — Bypass the OCSP check on `pkgupdate7.synology.com` via blackhole routes

> [!warning] Superseded
> The solution is **partial and unstable** on networks with DPI filtering (such as TSPU and equivalents). See [[#Why the solution does not work under DPI]].
> Replaced by: [[ADR-0003-package-center-via-external-proxy]].

> [!summary]
> On a Synology DS218play (DSM 7.3.2-86009 Update 3) `synopkg checkupdate*` and Package Center hang for 30+ seconds. Initial hypothesis: the Sectigo OCSP responder (`ocsp.sectigo.com` / `crl.sectigo.com`) is hosted on Cloudflare, the ISP throttles TCP data to Cloudflare, so the TLS stack waits for an OCSP timeout.
> Decision: install blackhole routes on the NAS for the relevant Cloudflare subnets so OCSP queries fail immediately (soft-fail).
> **The solution does not fully work** — DPI cuts more than just OCSP, and it does so by inspecting the TLS payload, not by routing.

## Context

### Symptom
- `synopkg checkupdate <pkg>` and `synopkg checkupdateall` hang until they time out without returning a result.
- In the DSM browser UI Package Center spins on its loading indicator; package updates never arrive.
- Networking in general works (ping/curl to `update7.synology.com` answers quickly, SSL is valid).

### Environment
- **Model:** Synology DS218play (Realtek RTD1296, ARMv8, 1 GB RAM).
- **DSM:** 7.3.2-86009 Update 3.
- **ISP:** regional carrier with active DPI (at the time of diagnosis Cloudflare is partially throttled at the TCP-stream level).

### What the initial diagnosis showed
1. `synopkg list` returns instantly — the binary itself is alive.
2. `curl https://pkgupdate7.synology.com/` answers HTTP/2 404 in milliseconds (CloudFront `18.165.140.x` is reachable).
3. The SSL chain is valid (`Verify return code: 0 (ok)`), `issuer=Sectigo Public Server Authentication CA DV R36`.
4. Through `/proc/<pid>/fd` and `/proc/net/tcp`, **two TCP connections from synopkg were captured during the hang**:
   - `18.165.140.89:443` — CloudFront (Synology, response received).
   - `172.67.75.147:443` / `104.26.5.55:443` — **Cloudflare**, `ESTABLISHED` but with `retr=3` (TCP retransmissions without ACK).
5. `nslookup` of the key OCSP/CRL hosts:
   - `ocsp.sectigo.com` → `172.64.149.23` (Cloudflare).
   - `crl.sectigo.com` → `104.18.38.233` (Cloudflare).
   - `ocsp.usertrust.com`, `ocsp.comodoca.com` — also on Cloudflare.
6. `synopkg` uses **its own networking stack** through `libsynocore.so.7` / `libsynow3.so.7` / Synology's bundled libcurl, and this stack **ignores `/etc/hosts`** (an asynchronous resolver bypassing glibc). This is visible from the discrepancy: Python's `socket.gethostbyname` returns `127.0.0.1` (sees the hosts overrides), while `synopkg` still talks to the real Cloudflare IPs.

### Initially-assumed root cause
The TLS handshake from `synopkg` to `pkgupdate7.synology.com` triggers an **OCSP revocation check on the Sectigo certificate**. The Sectigo OCSP responder lives on Cloudflare → the request hangs in TCP retransmissions → `synopkg` blocks on the OCSP timeout for every update query.

### Side issues (resolved separately)
- `/etc/synoinfo.conf` retained legacy update-server URLs after the DSM upgrade. See [[ADR-0002-stale-update-urls-in-synoinfo-after-dsm-upgrade]] — that decision remains valid independently of ADR-0001.
- `/usr/syno/etc/packages/feeds` carried the third-party `packages.synocommunity.com` repository (on Fastly, working); cleared during diagnosis.

## Options considered

### Option A — `/etc/hosts` overrides for the OCSP hosts
- **Pros:** minimal change, leaves routing alone.
- **Cons:** **does not work** — `synopkg` uses its own resolver, bypassing glibc/`/etc/hosts`.
- **Decision:** rejected.

### Option B — disable the OCSP check inside the TLS library
- **Pros:** targeted fix.
- **Cons:** Synology exposes no OCSP setting in `synopkg`/`synosmpkgd`. Patching the binaries would break on every DSM update.
- **Decision:** rejected.

### Option C — VPN/WARP for all NAS traffic
- **Pros:** fully bypasses the regional throttling.
- **Cons:** slows down all communication with Synology, adds dependencies (VPN client and its upkeep); DS218play resources are tight already (1 GB RAM).
- **Decision:** rejected as overkill **at the time**. **Eventually re-opened in [[ADR-0003-package-center-via-external-proxy]] in the form of an external HTTP proxy.**

### Option D — `iptables -j REJECT --reject-with tcp-reset` on the Cloudflare subnets
- **Pros:** works, returns a TCP RST → instant rejection.
- **Cons:** iptables rules in DSM 7 may be rewritten by the built-in firewall, so hooks are required.
- **Decision:** rejected in favour of option E (simpler and more stable).

### Option E — `ip route add blackhole` for the Cloudflare subnets ✅ (initially accepted, later superseded)
- **Pros:**
  - Routes live in the kernel and trigger **before** the SYN packet is sent, returning `ENETUNREACH` immediately.
  - Independent of the DSM iptables/firewall state.
  - Easy to anchor in the DSM Task Scheduler on Boot-up.
- **Cons (discovered later):**
  - The Synology Package Center pipeline uses **`pkgupdate7.synology.com` → 302 redirect → `pkgautoupdate.synologyupdate.com` (Cloudflare)**. That domain is **legitimately required** — the GUI does not function without it. A `/16` blackhole blocks it too.
  - Narrow `/32` exceptions via the gateway help only as long as Cloudflare does not rotate the domain's IPs. With rotation in mind, a keepalive every ≤5 minutes is needed.
  - **Does not help against DPI** — the ISP cuts the TCP stream by content, not by the fact of connection establishment.
- **Decision:** **initially accepted**, in operation it proved unstable, **superseded**.

### Option F — same thing on the router
- **Pros:** centralised, applies to the whole LAN.
- **Cons:** requires router access; does not solve the DPI issue.
- **Decision:** rejected by the operator as off-topic (and it would not have helped anyway — the root cause is DPI).

## The (then-)accepted decision

Apply option **E**: install four blackhole routes for Cloudflare subnets:

| Subnet | Covers |
|---|---|
| `104.18.0.0/16` | `crl.sectigo.com` (104.18.38.233), `ocsp.usertrust.com` |
| `104.26.0.0/16` | Captured during the hang (104.26.4.55, 104.26.5.55) |
| `172.64.0.0/16` | `ocsp.sectigo.com` (172.64.149.23), `ocsp.comodoca.com` |
| `172.67.0.0/16` | Captured during the hang (172.67.75.147) |

Persistence — through the DSM Task Scheduler: a Triggered Task on `Boot-up` plus a Scheduled Task every 10 minutes, both running as `root`.

## Why the solution does not work under DPI

Operational testing surfaced the following:

### 1. The Package Center GUI talks to a Cloudflare domain
A Package Center request to `pkgupdate7.synology.com/packagecenter/v3/getList?...` returns an **HTTP 302 redirect** to `https://pkgautoupdate.synologyupdate.com/getList/...`. That domain:
- lives **entirely on Cloudflare** (`172.67.75.147`, `104.26.4.55`, `104.26.5.55`, IPv6 `2606:4700::/32`),
- has **no AWS fallback**.

A `/16` blackhole on Cloudflare cuts it as well, which produces an immediate "Connection failed" error in the GUI instead of a long hang. The symptom changes shape; the underlying problem stays.

### 2. Narrow `/32` exceptions are unstable
Pulling specific IPs of `pkgautoupdate.synologyupdate.com` out of the blackhole via `ip route add IP/32 via $GW dev eth0`:
- **works** for IPs already known (CLI `synopkg checkupdateall` returns `exit=0`);
- **does not cover** fresh IPs that Cloudflare rotates per DNS lookup;
- even on covered `/32` IPs, Synology **still hangs intermittently** (`retr=3` in `/proc/net/tcp` on ESTABLISHED connections).

The last point is the crucial one.

### 3. DPI cuts the TCP stream, not the connection
The TCP handshake to a Cloudflare IP completes, ICMP answers in 39 ms, `curl -sI` (a ~700-byte HEAD) returns `HTTP 404` in 0.8 s. But a `curl GET` for the larger JSON response is truncated **at exactly 16384 bytes** (`/tmp/getlist.out` after timeout contains those 16 KB and nothing more — the TCP socket is empty afterwards).

This is the signature of **TLS-stream-level DPI**: the device lets the TCP handshake and the first ~16 KB of encrypted data through, then silently drops further packets without sending ICMP. No routing change or MTU/PMTU adjustment can help — the filtering is by content, not by address.

Lowering eth0 MTU to 1400 and 1280 made no difference.

### 4. DNS resolution of the critical domains is flaky
The local DNS `192.168.1.1` (the router) intermittently returns an empty answer for `pkgautoupdate.synologyupdate.com` and the Sectigo OCSP hosts. The keepalive script that depends on a fresh DNS lookup misses every other run, so `/32` routes are not refreshed.

### Conclusion
The blackhole approach addresses only the sub-problem of "make TLS soft-fail on OCSP" — and even that unstably under IP rotation. The actual root cause is **content-based DPI filtering of Cloudflare traffic**. Reaching `pkgautoupdate.synologyupdate.com` is impossible without routing the entire channel through an external proxy/VPN.

## Consequences (current)

### Positive
- The ADR is useful as a map of dead ends: anyone hitting "Package Center hangs" later does not have to retrace the `/etc/hosts` → blackhole path.
- The companion work on [[ADR-0002-stale-update-urls-in-synoinfo-after-dsm-upgrade|stale URLs]] remains valid and needed.

### Negative
- The NAS retains artefacts that need rolling back (Task Scheduler tasks, blackhole routes, `/32` exceptions, modified MTU). The rollback checklist lives in [[Solving-package-center-ocsp-cloudflare-blackhole#Rollback|the deprecated Solving guide]].

## Verification cues (historical)

- Before the fix: `timeout 30 synopkg checkupdate FileStation` → `exit=124` (timeout).
- After applying blackhole + `/32`: `synopkg checkupdateall` → `exit=0` in < 2 s **at the moment of application**, yet the Package Center GUI showed "Connection failed", and after a reboot the CLI would time out again with `exit=124`.

## Links

- Replacement decision: [[ADR-0003-package-center-via-external-proxy]] and [[Solving-package-center-external-proxy]].
- Adjacent URL issue: [[ADR-0002-stale-update-urls-in-synoinfo-after-dsm-upgrade]].
- Deprecated guide (with the rollback steps): [[Solving-package-center-ocsp-cloudflare-blackhole]].
- Cloudflare IP ranges: <https://www.cloudflare.com/ips-v4/>
- Sectigo OCSP/CRL endpoints: <https://sectigo.com/resource-library/cps-and-cp>
