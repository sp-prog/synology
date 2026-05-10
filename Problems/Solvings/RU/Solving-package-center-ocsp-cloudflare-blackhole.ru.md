---
title: (DEPRECATED) Починка зависающих обновлений Package Center через blackhole для Cloudflare
date: 2026-05-09
status: deprecated
superseded_by: "[[Solving-package-center-external-proxy.ru]]"
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
  - "[[ADR-0001-package-center-hangs-on-sectigo-ocsp.ru]]"
  - "[[ADR-0003-package-center-via-external-proxy.ru]]"
  - "[[Solving-package-center-external-proxy.ru]]"
---

# (DEPRECATED) Blackhole для Cloudflare

> [!danger] Решение помечено как deprecated
> На сетях с **DPI-фильтрацией** (ТСПУ и аналогах) этот подход даёт **частичный и нестабильный** эффект:
> - CLI `synopkg checkupdateall` иногда работает, иногда нет;
> - GUI Package Center либо «Сбой подключения», либо «крутится»;
> - после ребута маршруты приходится восстанавливать вручную, keepalive не покрывает все случаи.
>
> **Рабочее решение** — внешний HTTP-прокси на VPS вне зоны DPI: см. [[Solving-package-center-external-proxy.ru]].
>
> Этот документ оставлен ради:
> 1. Истории и шагов **отката** (если уже применили и хотите вернуть всё назад).
> 2. Будущих читателей, чтобы они не повторяли тот же путь.

## Когда этот подход в принципе мог бы сработать

Если у вас **нет DPI** на трафике к Cloudflare, но Cloudflare-IP блокируются по факту установления соединения (например, конкретные подсети не маршрутизируются провайдером), blackhole с узкими /32-исключениями **может** помочь. На сетях с активным DPI — нет.

## Почему не работает на DPI

Симптом-сигнатура DPI: `curl -sI` (HEAD ~700 байт) проходит, `curl GET` со списком пакетов обрезается **ровно на 16384 байт** (`/tmp/getlist.out` ровно 16 КБ). Это типовое поведение фильтра, который пропускает TCP-handshake и первое окно зашифрованных данных, потом дропает молча. Маршрутизация и MTU тут не помогают — фильтрация по содержимому, а не по адресу.

Подробный разбор — [[ADR-0001-package-center-hangs-on-sectigo-ocsp.ru#Почему решение не работает в условиях DPI]].

## Что делает (делал) этот подход — кратко

1. Обновляет URL в `/etc/synoinfo.conf` (это полезно само по себе → отдельная процедура [[Solving-fix-stale-update-urls-in-synoinfo.ru]]).
2. Чистит сторонние feeds в `/usr/syno/etc/packages/feeds`.
3. Добавляет `ip route add blackhole` для 4 подсетей Cloudflare (`104.18.0.0/16`, `104.26.0.0/16`, `172.64.0.0/16`, `172.67.0.0/16`).
4. Поднимает /32-исключения через шлюз для IP `pkgautoupdate.synologyupdate.com`.
5. Закрепляет всё через DSM Task Scheduler (Boot-up + Scheduled keepalive).
6. Опционально — снижает MTU до 1400 (попытка обойти PMTU black hole, бесполезна при DPI).

## Откат

Если ранее применялись шаги этого подхода и нужно вернуть систему в исходное состояние — выполните перечисленные ниже действия вручную. Сверяйтесь с тем, что реально было сделано на конкретной системе; пропускайте пункты, которые не применялись.

### Через DSM GUI

1. **Control Panel → Task Scheduler** — удалить все задачи, относящиеся к этому подходу (имена, которые могли использоваться: `Cloudflare blackhole for OCSP`, `Cloudflare blackhole — boot`, `Cloudflare blackhole — keepalive`).
2. **Control Panel → Network → Network Interface → LAN 1 → Edit → MTU value** — если значение менялось вручную, вернуть на `Auto` (или `1500`).
3. **Control Panel → Network → General → DNS** — если меняли DNS на публичный, при желании вернуть на адрес роутера.
4. **Package Center → Settings → Package Sources** — если в ходе диагностики удалялись сторонние источники (например, SynoCommunity), при необходимости добавить их обратно через **Add**.

### Через SSH (при необходимости, если правились системные файлы)

Бэкапы создавались с расширением `.bak.<метка>` рядом с оригиналом. Восстановление из бэкапа в общем случае:

```
ls -t <файл>.bak.*
cp <файл>.bak.<метка> <файл>
```

Файлы, которые могли быть изменены в ходе этого подхода:
- `/etc/synoinfo.conf` — правка URL обновлений (см. также [[Solving-fix-stale-update-urls-in-synoinfo.ru]]).
- `/usr/syno/etc/packages/feeds` — список сторонних репозиториев.
- `/etc/hosts` — заглушки для OCSP/CRL-доменов Sectigo.

Маршруты ядра (`ip route blackhole ...`, `ip route add IP/32 ...`) и `ip link set eth0 mtu ...` не персистят и автоматически уходят после перезагрузки NAS.

После любых изменений системных файлов — перезапустить пакетный демон:
```
synosystemctl restart synosmpkgd
synosystemctl restart synoscgi
```

Для полной чистоты состояния — перезагрузить NAS.

## Что делать вместо этого

Перейти на рабочее решение через внешний прокси: [[Solving-package-center-external-proxy.ru]].

## Что осталось валидным из этой ветки расследования

- Восстановление актуальных URL в `/etc/synoinfo.conf` — отдельная задача, не зависит от blackhole. См. [[Solving-fix-stale-update-urls-in-synoinfo.ru]].
- Чистка / возврат сторонних feeds через GUI или SSH — стандартная процедура, не требует никаких костылей.

## Ссылки

- ADR (с разбором, почему подход провалился): [[ADR-0001-package-center-hangs-on-sectigo-ocsp.ru]]
- Замещающее ADR: [[ADR-0003-package-center-via-external-proxy.ru]]
- Замещающий Solving: [[Solving-package-center-external-proxy.ru]]
- Связанная (валидная) проблема URL: [[Solving-fix-stale-update-urls-in-synoinfo.ru]]
