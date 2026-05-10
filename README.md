# Synology — Diagnostic Notes and Solutions

[🇬🇧 English](README.md) · [🇷🇺 Русский](README.ru.md)

A collection of **Architecture Decision Records (ADRs)** and **step-by-step solving guides** for issues encountered while running a Synology DS218play on DSM 7.x.

Documents reference each other via wiki-style `[[links]]` and are also readable on GitHub. Each document is short and focused on a single problem.

## Repository layout

```
Problems/
├── ADR/                    # why we decided what we decided
│   ├── RU/                 # Russian-language records (*.ru.md)
│   └── EN/                 # English-language records (*.en.md)
└── Solvings/               # how to actually do it
    ├── RU/                 # Russian-language guides (*.ru.md)
    └── EN/                 # English-language guides (*.en.md)
```

Each translated document has a language suffix in the file name (`.ru.md` / `.en.md`), so wiki-links inside the documents resolve unambiguously to the same-language counterpart.

## Architecture Decision Records

| ID | Title | Status | EN | RU |
|---|---|---|---|---|
| ADR-0001 | Bypass OCSP check for `pkgupdate7.synology.com` via blackhole routes | superseded by ADR-0003 | [EN](Problems/ADR/EN/ADR-0001-package-center-hangs-on-sectigo-ocsp.en.md) | [RU](Problems/ADR/RU/ADR-0001-package-center-hangs-on-sectigo-ocsp.ru.md) |
| ADR-0002 | Reconcile stale update URLs in `/etc/synoinfo.conf` after a DSM upgrade | accepted | [EN](Problems/ADR/EN/ADR-0002-stale-update-urls-in-synoinfo-after-dsm-upgrade.en.md) | [RU](Problems/ADR/RU/ADR-0002-stale-update-urls-in-synoinfo-after-dsm-upgrade.ru.md) |
| ADR-0003 | Route Synology DSM service traffic through an external HTTP proxy | accepted (supersedes ADR-0001) | [EN](Problems/ADR/EN/ADR-0003-package-center-via-external-proxy.en.md) | [RU](Problems/ADR/RU/ADR-0003-package-center-via-external-proxy.ru.md) |

## Solving Guides

| Guide | Status | EN | RU |
|---|---|---|---|
| Fix stale update URLs in `/etc/synoinfo.conf` | applied | [EN](Problems/Solvings/EN/Solving-fix-stale-update-urls-in-synoinfo.en.md) | [RU](Problems/Solvings/RU/Solving-fix-stale-update-urls-in-synoinfo.ru.md) |
| Package Center via external HTTP proxy | applied | [EN](Problems/Solvings/EN/Solving-package-center-external-proxy.en.md) | [RU](Problems/Solvings/RU/Solving-package-center-external-proxy.ru.md) |
| (deprecated) Package Center via Cloudflare blackhole | deprecated | [EN](Problems/Solvings/EN/Solving-package-center-ocsp-cloudflare-blackhole.en.md) | [RU](Problems/Solvings/RU/Solving-package-center-ocsp-cloudflare-blackhole.ru.md) |

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
