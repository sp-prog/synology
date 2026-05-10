# Synology — записи диагностики и решения

[🇬🇧 English](README.md) · [🇷🇺 Русский](README.ru.md)

Коллекция **Architecture Decision Records (ADR)** и **пошаговых инструкций (Solving)** по проблемам, обнаруженным при эксплуатации Synology DS218play на DSM 7.x.

Документы связаны между собой wiki-ссылками `[[…]]` и читаются как локально, так и на GitHub. Каждый документ короткий и посвящён одной проблеме.

## Структура репозитория

```
Problems/
├── ADR/                    # почему принято такое решение
│   ├── RU/                 # документы на русском (*.ru.md)
│   └── EN/                 # документы на английском (*.en.md)
└── Solvings/               # как именно это сделать
    ├── RU/                 # инструкции на русском (*.ru.md)
    └── EN/                 # инструкции на английском (*.en.md)
```

У каждого переведённого документа в имени файла стоит языковой суффикс (`.ru.md` / `.en.md`), поэтому wiki-ссылки внутри документов однозначно резолвятся на одноязычную копию.

## Architecture Decision Records

| ID | Название | Статус | RU | EN |
|---|---|---|---|---|
| ADR-0001 | Обход OCSP-проверки `pkgupdate7.synology.com` через blackhole-маршруты | superseded by ADR-0003 | [RU](Problems/ADR/RU/ADR-0001-package-center-hangs-on-sectigo-ocsp.ru.md) | [EN](Problems/ADR/EN/ADR-0001-package-center-hangs-on-sectigo-ocsp.en.md) |
| ADR-0002 | Синхронизация устаревших URL обновлений в `/etc/synoinfo.conf` после апгрейда DSM | accepted | [RU](Problems/ADR/RU/ADR-0002-stale-update-urls-in-synoinfo-after-dsm-upgrade.ru.md) | [EN](Problems/ADR/EN/ADR-0002-stale-update-urls-in-synoinfo-after-dsm-upgrade.en.md) |
| ADR-0003 | Доступ Package Center к Synology через внешний HTTP-прокси | accepted (supersedes ADR-0001) | [RU](Problems/ADR/RU/ADR-0003-package-center-via-external-proxy.ru.md) | [EN](Problems/ADR/EN/ADR-0003-package-center-via-external-proxy.en.md) |

## Solving — пошаговые инструкции

| Инструкция | Статус | RU | EN |
|---|---|---|---|
| Починка устаревших URL в `/etc/synoinfo.conf` | applied | [RU](Problems/Solvings/RU/Solving-fix-stale-update-urls-in-synoinfo.ru.md) | [EN](Problems/Solvings/EN/Solving-fix-stale-update-urls-in-synoinfo.en.md) |
| Package Center через внешний HTTP-прокси | applied | [RU](Problems/Solvings/RU/Solving-package-center-external-proxy.ru.md) | [EN](Problems/Solvings/EN/Solving-package-center-external-proxy.en.md) |
| (deprecated) Package Center через blackhole для Cloudflare | deprecated | [RU](Problems/Solvings/RU/Solving-package-center-ocsp-cloudflare-blackhole.ru.md) | [EN](Problems/Solvings/EN/Solving-package-center-ocsp-cloudflare-blackhole.en.md) |

## Карта связей между документами

```
ADR-0001 (superseded) ──── superseded_by ───► ADR-0003 (accepted)
   │                                              │
   └── ссылается на ─► Solving-…-blackhole        └── ссылается на ─► Solving-…-external-proxy
                       (deprecated, с чек-листом отката)              (текущее применённое решение)

ADR-0002 (accepted) ─── ссылается на ─► Solving-fix-stale-update-urls
                        (не зависит от ветки ADR-0001 / ADR-0003)
```

## Как читать эти документы

- **В редакторе с поддержкой wiki-ссылок.** Если открыть папку репозитория в инструменте, который умеет резолвить `[[wiki-ссылки]]` между markdown-файлами, все взаимные переходы между документами становятся кликабельными.
- **На GitHub.** `[[…]]` рендерятся как обычный текст в квадратных скобках. Для навигации используйте таблицы выше.

## Ограничение ответственности

> [!warning] Без гарантий — применение на свой страх и риск
> Материалы этого репозитория описывают изменения на Synology NAS под управлением DSM: правки системных конфигурационных файлов, маршрутизации ядра, сторонних пакетов, сетевых настроек, внешних прокси-сервисов. Некоторые процедуры (особенно помеченные `deprecated`) могут быть неприменимы в конкретной среде и способны ухудшить ситуацию.
>
> Авторы и контрибьюторы:
> - предоставляют информацию **«как есть»**, без каких-либо гарантий, явных или подразумеваемых, включая корректность, полноту, пригодность для конкретной задачи и соблюдение прав третьих лиц;
> - **не несут ответственности** за потерю данных, выход оборудования из строя, потерю гарантии, перебои в работе сети, инциденты безопасности, простой и любые другие последствия — прямые или косвенные, — которые могут возникнуть из-за следования этим документам;
> - не гарантируют, что описанные процедуры продолжат работать после очередного обновления DSM или изменений на стороне сторонних сервисов (Synology, Cloudflare, удостоверяющие центры, VPN-провайдеры и т.д.).
>
> Любое применение этих материалов производится **полностью на свой страх и риск**. Перед любым изменением — обязательный бэкап. Каждая команда сверяется с реальной системой. В случае сомнений — официальная документация и поддержка Synology.

## Лицензия

[MIT](LICENSE).
