# Synology — Diagnostic Notes and Solutions

[🇬🇧 English](README.md) · [🇷🇺 Русский](README.ru.md)

A collection of **Architecture Decision Records (ADRs)** and **step-by-step solving guides** for issues encountered while running a Synology DS218play on DSM 7.x.

Documents reference each other via wiki-style `[[links]]` and are also readable on GitHub. Each document is short and focused on a single problem.

## Repository layout

```
Problems/
├── ADR/                    # why we decided what we decided
│   └── RU/                 # Russian-language records
└── Solvings/               # how to actually do it
    └── RU/                 # Russian-language guides
```

Sub-folders by language ID (`RU/`, future `EN/`, …) leave room for translations without breaking existing wiki-links.

## Architecture Decision Records

| ID | Title | Status |
|---|---|---|
| [ADR-0001](Problems/ADR/RU/ADR-0001-package-center-hangs-on-sectigo-ocsp.md) | Bypass OCSP check for `pkgupdate7.synology.com` via blackhole routes | superseded by ADR-0003 |
| [ADR-0002](Problems/ADR/RU/ADR-0002-stale-update-urls-in-synoinfo-after-dsm-upgrade.md) | Reconcile stale update URLs in `/etc/synoinfo.conf` after a DSM upgrade | accepted |
| [ADR-0003](Problems/ADR/RU/ADR-0003-package-center-via-external-proxy.md) | Route Synology DSM service traffic through an external HTTP proxy | accepted (supersedes ADR-0001) |

## Solving Guides

| Guide | Status |
|---|---|
| [Fix stale update URLs in `/etc/synoinfo.conf`](Problems/Solvings/RU/Solving-fix-stale-update-urls-in-synoinfo.md) | applied |
| [Package Center via external HTTP proxy](Problems/Solvings/RU/Solving-package-center-external-proxy.md) | applied |
| [(deprecated) Package Center via Cloudflare blackhole](Problems/Solvings/RU/Solving-package-center-ocsp-cloudflare-blackhole.md) | deprecated |

## Cross-document map

```
ADR-0001 (superseded) ──── superseded_by ───► ADR-0003 (accepted)
   │                                              │
   └── points to ──► Solving-…-blackhole          └── points to ──► Solving-…-external-proxy
                     (deprecated, with rollback notes)              (current applied solution)

ADR-0002 (accepted) ─── points to ──► Solving-fix-stale-update-urls
                       (independent of the ADR-0001 / ADR-0003 thread)
```

## How to read these documents

- **In a wiki-link-aware editor.** When the repository folder is opened in any tool that resolves `[[wiki-links]]` between markdown files, all cross-references between documents become clickable.
- **On GitHub.** Wiki-style `[[…]]` links render as plain bracketed text. Use the tables above to navigate between documents.

## Disclaimer

> [!warning] No warranty — use at your own risk
> The materials in this repository describe changes to a Synology NAS running DSM: edits to system configuration files, kernel routing, third-party packages, network settings, external proxy services. Some procedures (especially those marked `deprecated`) may not apply to a specific environment and may make it worse.
>
> The authors and contributors:
> - provide the information **"as is"**, with no warranty of any kind, express or implied, including correctness, completeness, fitness for any particular purpose, or non-infringement;
> - accept **no liability** for any data loss, hardware damage, voided warranty, network outage, security incident, downtime, or any other consequence — direct or indirect — that may result from following these documents;
> - do not guarantee that any procedure will continue to work after a future DSM update or a change in third-party services (Synology, Cloudflare, certificate authorities, VPN providers, etc.).
>
> Anyone applying these documents does so **entirely at their own risk**. Always make a backup before any change. Verify every command against the actual system. When in doubt, consult Synology's official documentation and support channels.

## License

[MIT](LICENSE).
