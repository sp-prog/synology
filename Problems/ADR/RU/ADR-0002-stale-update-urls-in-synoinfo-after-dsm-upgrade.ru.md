---
id: ADR-0002
title: Синхронизация устаревших URL обновлений в /etc/synoinfo.conf после апгрейда DSM
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
  - "[[Solving-fix-stale-update-urls-in-synoinfo.ru]]"
  - "[[ADR-0001-package-center-hangs-on-sectigo-ocsp.ru]]"
---

# ADR-0002 — Синхронизация устаревших URL обновлений в `/etc/synoinfo.conf` после апгрейда DSM

> [!summary]
> На DS218play (DSM 7.3.2-86009 Update 3) активный конфиг `/etc/synoinfo.conf` после апгрейда между мажорными версиями DSM сохранил **старые** URL пакетных серверов (`pkgupdate.synology.com`, `update.synology.com`, `small_info_path` без `7`), в то время как заводские дефолты в `/etc.defaults/synoinfo.conf` уже обновлены до `pkgupdate7.synology.com` / `update7.synology.com`. Принято решение точечно обновить расходящиеся ключи через `synosetkeyvalue`, не трогая пользовательские настройки.

## Контекст

### Симптом
При диагностике зависающего Package Center (см. [[ADR-0001-package-center-hangs-on-sectigo-ocsp.ru]]) обнаружено: `/etc/synoinfo.conf` содержит URL вида `pkgupdate.synology.com`, `update.synology.com`, тогда как актуальные на DSM 7.x — с цифрой `7` в имени.

### Расхождение, найденное на конкретной системе
DSM 7.3.2-86009 Update 3, DS218play.

| Ключ | `/etc/synoinfo.conf` (живой) | `/etc.defaults/synoinfo.conf` (заводской) |
|---|---|---|
| `pkgupdate_server` | `https://pkgupdate.synology.com` ❌ | `https://pkgupdate7.synology.com` ✅ |
| `rss_server` | `http://update.synology.com/autoupdate/genRSS.php` ❌ | `http://update7.synology.com/autoupdate/genRSS.php` ✅ |
| `rss_server_ssl` | `https://update.synology.com/autoupdate/genRSS.php` ❌ | `https://update7.synology.com/autoupdate/genRSS.php` ✅ |
| `update_server` | `http://update.synology.com/` ❌ | `http://update7.synology.com/` ✅ |
| `small_info_path` | `http://update.synology.com/smallupdate` ❌ | `https://update7.synology.com/smallupdate` ✅ |

Также найдены менее критичные расхождения (косметические):

| Ключ | `/etc/synoinfo.conf` | `/etc.defaults/synoinfo.conf` |
|---|---|---|
| `online_help_base_url` | `http://www.synology.com/help/uihelp/` | `http://help.synology.com/` |
| `printer_driver_host` | `http://download.synology.com/airprint/DSM4.1` | `https://global.download.synology.com/airprint/DSM7.3/latest` |
| `frame_options_built_in_allow_url` | (отсутствует) | `gofile.me/,find.synology.com/` |

И **легаси-ключи**, которых уже нет в дефолтах:

| Ключ | `/etc/synoinfo.conf` | Заводские |
|---|---|---|
| `pushservice_server_1` | `https://sns1.synology.com:8089/api/` | (отсутствует) |
| `pushservice_server_2` | `https://sns2.synology.com:8089/api/` | (отсутствует) |

### Почему это произошло
DSM при апгрейде между мажорными версиями (6.x → 7.x, и в некоторых случаях между минорными 7.x) обновляет **`/etc.defaults/synoinfo.conf`**, но **не перезаписывает** активный `/etc/synoinfo.conf` целиком — чтобы не затереть пользовательские настройки. Миграция конкретных ключей реализуется в скриптах апгрейда; для части ключей миграция не предусмотрена либо не сработала.

В рассматриваемом случае апгрейд, по-видимому, шёл с DSM 6.x (DS218play выпускалась с DSM 6.x), и ключи `pkgupdate_server` / `update_server` остались с прежними значениями.

### Влияние
- `synopkg` и `synoupgrade` ходят на устаревшие хосты. На DSM 7.3 устаревший `pkgupdate.synology.com` (без `7`) отвечает **HTTP 404** от CloudFront за миллисекунды — то есть быстро, но без полезной нагрузки. Поведение Package Center в этом случае — пустой/некорректный ответ, а не отказ оборудования.
- Правка обнаружена в ходе диагностики [[ADR-0001-package-center-hangs-on-sectigo-ocsp.ru]] и применена для устранения «шума» при дальнейшей диагностике. Связь устаревших URL с основной проблемой зависания (она вызвана отдельным фактором — DPI-фильтрацией Cloudflare, см. [[ADR-0003-package-center-via-external-proxy.ru]]) не исследовалась. Правка корректна как самостоятельная задача: на DSM 7.x все URL обновлений должны быть с `7`.

## Рассмотренные варианты

### Вариант A — перезаписать `/etc/synoinfo.conf` из `/etc.defaults/synoinfo.conf` целиком
- **Плюсы:** одной командой синхронизирует всё.
- **Минусы:** **затрёт пользовательские настройки** (модель UPnP, SMB-параметры, кастомизации интерфейса, телеметрию ActiveInsight, региональные настройки). В файле сотни ключей, не все из них URL.
- **Решение:** отвергнут как разрушительный.

### Вариант B — точечно обновить расходящиеся URL через `synosetkeyvalue` ✅
- **Плюсы:**
  - Меняются только нужные ключи.
  - Используется родная утилита Synology — корректно сохраняет формат и кавычки.
  - Бэкап делается одной командой `cp`.
  - Откат — восстановлением бэкапа.
- **Минусы:**
  - Нужно вручную составить список расхождений (через `diff`).
  - Не покрывает легаси-ключи, которые **есть в живом конфиге, но удалены из дефолтов** (их нужно либо удалять отдельно, либо игнорировать как безопасные).
- **Решение:** **принят.**

### Вариант C — Reset Network/Services через DSM GUI
- **Плюсы:** официальный механизм Synology, безопасен для пользовательских данных.
- **Минусы:** сбрасывает сразу много настроек (сеть, файервол, аутентификация). Чрезмерная мера для проблемы с URL.
- **Решение:** отвергнут как избыточный.

### Вариант D — игнорировать
- **Плюсы:** ничего не делать.
- **Минусы:** Package Center и `synoupgrade` продолжат работать криво. Ломается базовая функция NAS.
- **Решение:** отвергнут.

## Принятое решение

Применить вариант **B**: точечно обновить URL-ключи через `synosetkeyvalue`, ориентируясь на значения из `/etc.defaults/synoinfo.conf` той же системы (а не из внешних источников — это исключает несовпадения по версии DSM/региону).

Перечень ключей для обязательного обновления (если расходятся):
1. `pkgupdate_server`
2. `rss_server`
3. `rss_server_ssl`
4. `update_server`
5. `small_info_path`

Опциональные (косметика):
6. `online_help_base_url`
7. `printer_driver_host`
8. `frame_options_built_in_allow_url`

Легаси-ключи (`pushservice_server_1/2`) — **не трогать**: добавление их обратно DSM не предполагает, удаление может быть рискованно. Они не используются актуальной системой, безвредны.

Перед изменениями — обязательный бэкап `/etc/synoinfo.conf`.

Подробная инструкция: [[Solving-fix-stale-update-urls-in-synoinfo.ru]].

## Последствия

### Положительные
- `synopkg checkupdate*` и `synoupgrade --check` начинают ходить на актуальные серверы.
- Снимается часть фантомных «зависаний» Package Center.
- Меньше шума в логах — исчезают 404 и таймауты к старым хостам.

### Нейтральные
- Создаётся бэкап `/etc/synoinfo.conf.bak.<метка времени>` — занимает несколько килобайт.

### Отрицательные / риски
- **Может повторяться после следующего крупного апдейта DSM.** Если Synology в DSM 8 переедет с `update7` на `update8` — конфиг снова разойдётся. Митигация: при каждом крупном апгрейде DSM запускать diff-проверку (см. шаг проверки в Solving).
- **`synosetkeyvalue` не валидирует URL.** Опечатка в команде → битый конфиг. Митигация: бэкап + проверка `grep` после каждого изменения.

## Метрики проверки

- До фикса: `curl -sI https://pkgupdate.synology.com/` → отвечает (404), но `synopkg` использует устаревший хост и поведение непредсказуемо.
- После фикса: `grep -E '^(pkgupdate_server|rss_server|update_server|small_info_path)' /etc/synoinfo.conf` показывает все строки с `update7` / `pkgupdate7`.
- `diff` против `/etc.defaults/synoinfo.conf` по URL-ключам — пустой (кроме намеренно оставленных легаси-ключей).

## Связанные решения

- [[ADR-0001-package-center-hangs-on-sectigo-ocsp.ru]] — корневая причина зависаний Package Center; фикс из ADR-0002 необходим, но недостаточен.
- [[Solving-fix-stale-update-urls-in-synoinfo.ru]] — пошаговая инструкция применения этого ADR.
- [[Solving-package-center-ocsp-cloudflare-blackhole.ru]] — связанная инструкция по OCSP-фиксу.
