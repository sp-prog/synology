---
title: Починка устаревших URL обновлений в /etc/synoinfo.conf после апгрейда DSM
date: 2026-05-09
status: applied
hardware: DS218play (применимо к любой Synology)
dsm: 7.3.2-86009 Update 3 (применимо ко всем DSM 7.x после апгрейда с 6.x)
tags:
  - synology
  - dsm-7
  - synoinfo
  - upgrade
  - configuration
  - howto
  - solving
related:
  - "[[ADR-0002-stale-update-urls-in-synoinfo-after-dsm-upgrade.ru]]"
  - "[[ADR-0001-package-center-hangs-on-sectigo-ocsp.ru]]"
  - "[[Solving-package-center-ocsp-cloudflare-blackhole.ru]]"
---

# Починка устаревших URL обновлений в `/etc/synoinfo.conf`

> [!info] Кратко
> После апгрейда DSM 6.x → 7.x (и иногда между минорами 7.x) активный `/etc/synoinfo.conf` сохраняет старые URL пакетных серверов (`pkgupdate.synology.com`, `update.synology.com`, `small_info_path` без `7`). Заводские дефолты в `/etc.defaults/synoinfo.conf` уже обновлены до `pkgupdate7.synology.com` / `update7.synology.com`.
> **Лечение:** пройтись `diff`-ом по двум файлам и точечно перезаписать расходящиеся URL-ключи через `synosetkeyvalue`.
> **Решение принято в:** [[ADR-0002-stale-update-urls-in-synoinfo-after-dsm-upgrade.ru]]

## Симптомы

- В Package Center не приходят обновления, либо приходят медленно/частично.
- `synopkg checkupdate*` ведёт себя непредсказуемо — иногда отвечает, иногда виcит.
- `synoupgrade --check` не находит обновлений DSM, хотя они есть.
- В логах `/var/log/messages`, `/var/log/synolog/synopkg.log` встречаются ошибки с обращением к `update.synology.com` / `pkgupdate.synology.com` (без `7`).
- `curl -I https://pkgupdate.synology.com/` отвечает 404 от CloudFront — старый хост существует, но не обслуживает API DSM 7.x.

## Когда применять

✅ DSM 7.x на любой модели Synology, особенно если NAS обновлялся с DSM 6.x.

❌ Не применять без diff-проверки на DSM 6.x — там URL без `7` правильные.

❌ Не применять «вслепую» по чужому списку URL — всегда сверяйтесь с **вашим** `/etc.defaults/synoinfo.conf`. На разных версиях DSM значения могут отличаться (например, при выходе DSM 8 заводской URL станет `update8.synology.com`, а не `update7`).

## Шаг 0. SSH-доступ

DSM → **Control Panel → Terminal & SNMP → Enable SSH service**.

```bash
ssh admin@<IP-NAS>
sudo -i
```

## Шаг 1. Проверить, есть ли расхождения

```bash
echo "=== синология-домены в живом конфиге ==="
grep -E '\.synology\.com' /etc/synoinfo.conf | sort

echo
echo "=== синология-домены в заводских дефолтах ==="
grep -E '\.synology\.com' /etc.defaults/synoinfo.conf | sort

echo
echo "=== РАЗНИЦА (живой vs заводской) ==="
diff <(grep -E '\.synology\.com' /etc/synoinfo.conf | sort) \
     <(grep -E '\.synology\.com' /etc.defaults/synoinfo.conf | sort)
```

В выводе `diff`:
- **`<` строки** — то, что есть в живом, но отличается от заводского.
- **`>` строки** — то, что в заводском, и должно быть в живом.

Если diff пустой — расхождений нет, фикс не требуется.

Если есть расхождения — перейдите к шагу 2.

## Шаг 2. Бэкап

```bash
cp /etc/synoinfo.conf /etc/synoinfo.conf.bak.$(date +%Y%m%d-%H%M)
ls -la /etc/synoinfo.conf*
```

Бэкап остаётся в той же директории. На откат — `cp ... /etc/synoinfo.conf` обратно.

## Шаг 3. Обновить URL-ключи

> [!warning]
> Значения берите из вывода `diff` шага 1 — конкретно из строк с `>` (это значения из `/etc.defaults/synoinfo.conf` **вашей** системы). Не копируйте значения из этой инструкции вслепую — они могут устареть.

Шаблон команды:
```bash
synosetkeyvalue /etc/synoinfo.conf <ключ> "<новое_значение>"
```

### Обязательно обновляемые ключи (URL обновлений)

```bash
synosetkeyvalue /etc/synoinfo.conf pkgupdate_server  "https://pkgupdate7.synology.com"
synosetkeyvalue /etc/synoinfo.conf rss_server        "http://update7.synology.com/autoupdate/genRSS.php"
synosetkeyvalue /etc/synoinfo.conf rss_server_ssl    "https://update7.synology.com/autoupdate/genRSS.php"
synosetkeyvalue /etc/synoinfo.conf update_server     "http://update7.synology.com/"
synosetkeyvalue /etc/synoinfo.conf small_info_path   "https://update7.synology.com/smallupdate"
```

Эти пять ключей — критически важны для работы Package Center и DSM Update.

### Опциональные ключи (косметика)

Они не влияют на обновления, но почему бы не подровнять заодно:

```bash
synosetkeyvalue /etc/synoinfo.conf online_help_base_url             "http://help.synology.com/"
synosetkeyvalue /etc/synoinfo.conf printer_driver_host              "https://global.download.synology.com/airprint/DSM7.3/latest"
synosetkeyvalue /etc/synoinfo.conf frame_options_built_in_allow_url "gofile.me/,find.synology.com/"
```

`online_help_base_url` — куда ведёт «?» в DSM. `printer_driver_host` — драйверы AirPrint. `frame_options_built_in_allow_url` — список доменов, которым разрешено эмбедить DSM в iframe.

> [!note]
> На разных версиях DSM `printer_driver_host` указывает на свою версию (`DSM7.0/latest`, `DSM7.2/latest`, `DSM7.3/latest`). Берите значение из своего `/etc.defaults/synoinfo.conf`.

### Легаси-ключи — НЕ трогать

В живом конфиге могут остаться ключи, которых **уже нет** в заводских:

```
pushservice_server_1=https://sns1.synology.com:8089/api/
pushservice_server_2=https://sns2.synology.com:8089/api/
```

Их **не нужно** ни добавлять в дефолты, ни удалять из живого конфига. Synology давно не использует их (есть единый `pushservice_server`), и они безвредно лежат как мёртвый груз. Удалять через `synosetkeyvalue` нельзя — у неё нет операции delete; пришлось бы редактировать файл руками, что рискованно.

## Шаг 4. Проверить, что записалось

```bash
echo "=== обязательные ключи ==="
grep -E '^(pkgupdate_server|rss_server|update_server|small_info_path)=' /etc/synoinfo.conf

echo
echo "=== опциональные ключи ==="
grep -E '^(online_help_base_url|printer_driver_host|frame_options_built_in_allow_url)=' /etc/synoinfo.conf

echo
echo "=== повторный diff (должен сократиться) ==="
diff <(grep -E '\.synology\.com' /etc/synoinfo.conf | sort) \
     <(grep -E '\.synology\.com' /etc.defaults/synoinfo.conf | sort)
```

В обязательных ключах должны быть `update7` / `pkgupdate7`. В diff остаются только легаси-ключи (`pushservice_server_1/2`) — это нормально.

## Шаг 5. Перечитать конфиг и проверить эффект

```bash
synosystemctl restart synosmpkgd
sleep 3

# проверка ходит ли пакетный слой на новый хост
curl -m 10 -sI https://pkgupdate7.synology.com/ ; echo "exit=$?"

# проверка обновлений
timeout 30 synopkg checkupdateall
echo "exit=$?"
```

Должны быть:
- `curl ... pkgupdate7.synology.com` → HTTP/2 404 (это нормально для корня) **за миллисекунды**, `exit=0`.
- `synopkg checkupdateall` → `exit=0` за разумное время.

> [!warning]
> Если `synopkg checkupdateall` всё ещё виcит — это **отдельная проблема**, не связанная с устаревшими URL: DPI-фильтрация трафика к Cloudflare на стороне провайдера. Текущая правка URL её не лечит. См. [[Solving-package-center-external-proxy.ru]] — рабочее решение через внешний HTTP-прокси.

## Откат

```bash
ls /etc/synoinfo.conf.bak.*
cp /etc/synoinfo.conf.bak.<нужная_метка> /etc/synoinfo.conf
synosystemctl restart synosmpkgd
```

После отката `diff` снова покажет старые расхождения — это ожидаемо.

## Дополнительно: автоматизация diff-проверки после апдейтов DSM

Чтобы не пропускать новые расхождения после крупных апгрейдов DSM, можно повесить периодический check через Task Scheduler.

DSM → **Control Panel → Task Scheduler → Create → Scheduled Task → User-defined script**:

| Поле | Значение |
|---|---|
| Task | `Check synoinfo URL drift` |
| User | `root` |
| Schedule | First day of month, 03:00 |
| Run command | см. ниже |

```bash
DRIFT=$(diff <(grep -E '\.synology\.com' /etc/synoinfo.conf | sort) \
             <(grep -E '\.synology\.com' /etc.defaults/synoinfo.conf | sort) \
             | grep -vE 'pushservice_server_[12]=')
if [ -n "$DRIFT" ]; then
  echo "$DRIFT" | mail -s "Synology synoinfo URL drift detected" you@example.com
fi
```

(Замените `you@example.com` или закомментируйте `mail` и логируйте в файл — на NAS почта по умолчанию не настроена.)

## Технические детали

### Почему именно `synosetkeyvalue`, а не `sed -i`

`synosetkeyvalue` — родная утилита Synology, которая:
- корректно обрабатывает кавычки (`"value"`, без них или с одинарными — формат сохраняется);
- атомарно переписывает файл (через временный + `mv`), не оставляя битый промежуточный файл;
- не путается с многострочными значениями.

`sed -i` на конфиге Synology может неожиданно сломать файл (DSM использует свой парсер, чувствительный к пробелам и кавычкам).

### Почему изменения не теряются при перезагрузке

`/etc/synoinfo.conf` лежит на системном томе `/dev/md0` — это нормальный диск, а не tmpfs. Изменения пережёвают ребут. Они **могут** быть перезаписаны только при крупном апгрейде DSM, и то — Synology обычно мигрирует ключи, а не затирает файл.

### `/etc/synoinfo.conf` vs `/etc.defaults/synoinfo.conf`

- `/etc.defaults/synoinfo.conf` — заводские настройки текущей версии DSM. Обновляется при каждом апдейте системы. **Не редактировать руками.**
- `/etc/synoinfo.conf` — активный пользовательский конфиг. Создаётся копированием из дефолтов при первой установке, потом живёт самостоятельно. Редактируется через `synosetkeyvalue` или DSM-морду.

`diff` между ними при свежей установке — пустой. По мере апгрейдов и пользовательских настроек начинает расходиться. Цель этого Solving — синхронизировать только URL-подмножество ключей.

## Что было найдено на конкретной системе (для иллюстрации)

DS218play, DSM 7.3.2-86009 Update 3 (после апгрейда c DSM 6.x):

| Ключ | Было | Стало |
|---|---|---|
| `pkgupdate_server` | `https://pkgupdate.synology.com` | `https://pkgupdate7.synology.com` |
| `rss_server` | `http://update.synology.com/autoupdate/genRSS.php` | `http://update7.synology.com/autoupdate/genRSS.php` |
| `rss_server_ssl` | `https://update.synology.com/autoupdate/genRSS.php` | `https://update7.synology.com/autoupdate/genRSS.php` |
| `update_server` | `http://update.synology.com/` | `http://update7.synology.com/` |
| `small_info_path` | `http://update.synology.com/smallupdate` | `https://update7.synology.com/smallupdate` |
| `online_help_base_url` | `http://www.synology.com/help/uihelp/` | `http://help.synology.com/` |
| `printer_driver_host` | `http://download.synology.com/airprint/DSM4.1` | `https://global.download.synology.com/airprint/DSM7.3/latest` |

`pushservice_server_1`, `pushservice_server_2` — оставлены как есть (легаси).

## Ссылки

- ADR этого решения: [[ADR-0002-stale-update-urls-in-synoinfo-after-dsm-upgrade.ru]]
- Связанная проблема (OCSP+Cloudflare): [[ADR-0001-package-center-hangs-on-sectigo-ocsp.ru]] и [[Solving-package-center-ocsp-cloudflare-blackhole.ru]]
- Synology Knowledge Center, `synoinfo.conf`: внутри DSM специальной публичной документации нет — конфиг считается «черным ящиком»; обращайтесь к `/etc.defaults/synoinfo.conf` как к источнику истины.
