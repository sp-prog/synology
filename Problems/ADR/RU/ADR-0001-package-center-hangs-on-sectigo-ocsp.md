---
id: ADR-0001
title: Обход OCSP-проверки сертификата pkgupdate7.synology.com через blackhole-маршруты
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

# ADR-0001 — Обход OCSP-проверки `pkgupdate7.synology.com` через blackhole-маршруты

> [!warning] Superseded
> Решение **частично и нестабильно** на сетях с DPI-фильтрацией (ТСПУ и аналогах). См. [[#Почему решение не работает в условиях DPI]].
> Замещено: [[ADR-0003-package-center-via-external-proxy]].

> [!summary]
> На Synology DS218play (DSM 7.3.2-86009 Update 3) `synopkg checkupdate*` и Package Center виснут на 30+ секунд. Изначальная гипотеза: OCSP-responder Sectigo (`ocsp.sectigo.com` / `crl.sectigo.com`) хостится на Cloudflare, провайдер режет TCP-данные к Cloudflare, поэтому TLS-стек ждёт OCSP-таймаут.
> Принятое решение: blackhole-маршруты к подсетям Cloudflare на NAS, чтобы OCSP-запросы получали мгновенный отказ (soft-fail).
> **Решение работает не в полной мере** — DPI режет не только OCSP, и не на маршрутном уровне, а по содержимому TLS-потока.

## Контекст

### Симптом
- `synopkg checkupdate <pkg>` и `synopkg checkupdateall` виснут до таймаута, не возвращая результат.
- В Package Center в DSM-браузере «крутится» индикатор загрузки, апдейты пакетов не приходят.
- Сеть в целом работает (ping/curl до `update7.synology.com` отвечает быстро, SSL-валидно).

### Окружение
- **Модель:** Synology DS218play (Realtek RTD1296, ARMv8, 1 ГБ RAM).
- **DSM:** 7.3.2-86009 Update 3.
- **Провайдер:** региональный оператор с активным DPI (на момент диагностики Cloudflare частично режется на уровне TCP-потока).

### Что показала первичная диагностика
1. `synopkg list` отрабатывает мгновенно — сам бинарник живой.
2. `curl https://pkgupdate7.synology.com/` отвечает HTTP/2 404 за миллисекунды (CloudFront `18.165.140.x` доступен).
3. SSL-цепочка валидна (`Verify return code: 0 (ok)`), `issuer=Sectigo Public Server Authentication CA DV R36`.
4. Через `/proc/<pid>/fd` и `/proc/net/tcp` поймано **два TCP-соединения synopkg в момент зависания**:
   - `18.165.140.89:443` — CloudFront (Synology, ответ есть).
   - `172.67.75.147:443` / `104.26.5.55:443` — **Cloudflare**, состояние `ESTABLISHED`, но `retr=3` (TCP-ретрансмиссии без ACK).
5. `nslookup` ключевых OCSP/CRL-серверов:
   - `ocsp.sectigo.com` → `172.64.149.23` (Cloudflare).
   - `crl.sectigo.com` → `104.18.38.233` (Cloudflare).
   - `ocsp.usertrust.com`, `ocsp.comodoca.com` — тоже на Cloudflare.
6. `synopkg` использует **собственный сетевой стек** через `libsynocore.so.7` / `libsynow3.so.7` / libcurl-внутри-Synology, который **игнорирует `/etc/hosts`** (асинхронный резолвер мимо glibc). Это видно из расхождения: Python через `socket.gethostbyname` отдаёт `127.0.0.1` (видит hosts-заглушки), а `synopkg` всё равно стучится на реальные Cloudflare-IP.

### Изначально предполагаемая корневая причина
TLS-handshake `synopkg` к `pkgupdate7.synology.com` инициирует **OCSP-проверку отзыва сертификата Sectigo**. OCSP-responder Sectigo хостится на Cloudflare → запрос виснет в TCP-ретрансмиссиях → `synopkg` ждёт OCSP-таймаут на каждый запрос обновлений.

### Сопутствующие проблемы (исправлены отдельно)
- В `/etc/synoinfo.conf` после апгрейда DSM остались устаревшие URL пакетных серверов. См. [[ADR-0002-stale-update-urls-in-synoinfo-after-dsm-upgrade]] — это решение остаётся валидным независимо от ADR-0001.
- В `/usr/syno/etc/packages/feeds` был сторонний репозиторий `packages.synocommunity.com` (на Fastly, работает), очищен на время диагностики.

## Рассмотренные варианты

### Вариант A — заглушки в `/etc/hosts` для OCSP-доменов
- **Плюсы:** минимальное изменение, не трогает маршрутизацию.
- **Минусы:** **не работает** — `synopkg` использует собственный резолвер мимо glibc/`/etc/hosts`.
- **Решение:** отвергнут.

### Вариант B — отключить OCSP-проверку в TLS-библиотеке
- **Плюсы:** точечное решение.
- **Минусы:** Synology не предоставляет настройки OCSP в `synopkg`/`synosmpkgd`. Пришлось бы патчить бинарники, что слетит при апдейте DSM.
- **Решение:** отвергнут.

### Вариант C — VPN/WARP для всего трафика NAS
- **Плюсы:** полностью обходит региональную блокировку.
- **Минусы:** замедляет весь обмен с Synology, увеличивает зависимости (VPN-клиент, его поддержка), на DS218play ресурсов и так впритык (1 ГБ RAM).
- **Решение:** отвергнут как избыточный (на тот момент). **В итоге переоткрыт в [[ADR-0003-package-center-via-external-proxy]] в форме внешнего HTTP-прокси.**

### Вариант D — `iptables -j REJECT --reject-with tcp-reset` на Cloudflare-подсети
- **Плюсы:** работает, отдаёт TCP RST → мгновенный отказ.
- **Минусы:** правила iptables в DSM 7 могут перезаписываться при изменении настроек встроенного firewall, нужны хуки.
- **Решение:** отвергнут в пользу варианта E (проще и стабильнее).

### Вариант E — `ip route add blackhole` на Cloudflare-подсети ✅ (изначально принят, потом superseded)
- **Плюсы:**
  - Маршруты на уровне ядра, отрабатывают **до** отправки SYN-пакета, отдают `ENETUNREACH` мгновенно.
  - Не зависят от состояния iptables/firewall DSM.
  - Легко закрепляются через Task Scheduler на Boot-up.
- **Минусы (выявленные позже):**
  - Synology Package Center использует **`pkgupdate7.synology.com` → 302 redirect → `pkgautoupdate.synologyupdate.com` (Cloudflare)**. Этот домен **легитимно нужен** — без него GUI не работает. /16-blackhole его блокирует.
  - Узкие /32-исключения через шлюз помогают только пока Cloudflare не ротирует IP домена. С учётом ротации требуется keepalive каждые ≤5 минут.
  - **Не помогает против DPI** — провайдер режет TCP-поток по содержимому, не по факту установления соединения.
- **Решение:** **первоначально принят**, в эксплуатации показал себя нестабильным, **superseded**.

### Вариант F — то же самое на роутере
- **Плюсы:** централизованно, работает для всей LAN.
- **Минусы:** требует доступа к роутеру; не решает проблему с DPI.
- **Решение:** отвергнут пользователем как не относящийся к делу (и в любом случае не помог бы — корень в DPI).

## Принятое (на тот момент) решение

Применить вариант **E**: добавить четыре blackhole-маршрута к подсетям Cloudflare:

| Подсеть | Покрывает |
|---|---|
| `104.18.0.0/16` | `crl.sectigo.com` (104.18.38.233), `ocsp.usertrust.com` |
| `104.26.0.0/16` | Поймано в момент зависания (104.26.4.55, 104.26.5.55) |
| `172.64.0.0/16` | `ocsp.sectigo.com` (172.64.149.23), `ocsp.comodoca.com` |
| `172.67.0.0/16` | Поймано в момент зависания (172.67.75.147) |

Закрепление — через DSM Task Scheduler, Triggered Task на `Boot-up` + Scheduled Task каждые 10 минут, под `root`.

## Почему решение не работает в условиях DPI

При тестовой эксплуатации обнаружено:

### 1. Package Center в GUI ходит на Cloudflare-домен
Запрос Package Center к `pkgupdate7.synology.com/packagecenter/v3/getList?...` возвращает **HTTP 302 redirect** на `https://pkgautoupdate.synologyupdate.com/getList/...`. Этот домен:
- сидит **полностью на Cloudflare** (`172.67.75.147`, `104.26.4.55`, `104.26.5.55`, IPv6 `2606:4700::/32`),
- **не имеет AWS-fallback'а**.

Blackhole для `/16` Cloudflare-подсетей резал и его, что приводило к моментальной ошибке «Сбой подключения» в GUI вместо длительного зависания. Симптом изменился, корневая проблема не ушла.

### 2. Узкие /32-исключения нестабильны
Попытка вытащить конкретные IP `pkgautoupdate.synologyupdate.com` из blackhole через `ip route add IP/32 via $GW dev eth0`:
- **работала** для уже известных IP (CLI `synopkg checkupdateall` отдавал `exit=0`);
- **не покрывала** свежие IP, которые Cloudflare ротирует на каждый DNS-резолв;
- даже на покрытых /32-маршрутами IP **синология всё равно периодически виcла** (`retr=3` в `/proc/net/tcp` на ESTABLISHED-соединениях).

Последнее — самое важное.

### 3. DPI режет TCP-поток, а не соединение
TCP-handshake к Cloudflare-IP проходит, ICMP отвечает за 39 мс, `curl -sI` (HEAD-запрос ~700 байт) возвращает `HTTP 404` за 0.8 сек. Но `curl GET` с большим JSON-ответом обрезается **ровно на 16384 байт** (`/tmp/getlist.out` после таймаута содержит ровно эти 16 КБ, дальше TCP пустой).

Это сигнатура **DPI на уровне TLS-потока**: оборудование пропускает TCP-handshake и первые ~16 КБ зашифрованных данных, потом начинает дропать пакеты молча, не присылая ICMP. Никакая правка маршрутов или MTU/PMTU помочь не может — фильтрация по содержимому, не по адресу.

Снижение MTU на eth0 до 1400 и 1280 не помогло.

### 4. DNS-резолв критичных доменов нестабилен
Локальный DNS `192.168.1.1` (роутер) периодически возвращает пустой ответ для `pkgautoupdate.synologyupdate.com` и OCSP-доменов Sectigo. Keepalive-скрипт с DNS-резолвом срабатывал «через раз», и /32-маршруты не обновлялись.

### Вывод
Blackhole-подход решает только подзадачу «сделать TLS soft-fail на OCSP» — и то нестабильно при ротации IP. Корневая проблема — **DPI-фильтрация Cloudflare-трафика на содержимом**. Без обхода всего канала через сторонний прокси/VPN доступ к `pkgautoupdate.synologyupdate.com` не получить.

## Последствия (актуальные)

### Положительные
- ADR полезен как карта тупиков: следующий, кто столкнётся с симптомом «Package Center виснет», не будет повторять путь от `/etc/hosts` к blackhole.
- Сопутствующая работа по [[ADR-0002-stale-update-urls-in-synoinfo-after-dsm-upgrade|устаревшим URL]] остаётся валидной и нужной.

### Отрицательные
- В DSM на NAS остались артефакты, требующие отката (Task Scheduler-задачи, blackhole-маршруты, /32-исключения, изменённый MTU). Скрипт отката — в [[Solving-package-center-ocsp-cloudflare-blackhole#Откат|deprecated Solving]].

## Метрики проверки (исторические)

- До фикса: `timeout 30 synopkg checkupdate FileStation` → `exit=124` (таймаут).
- После применения blackhole + /32: `synopkg checkupdateall` → `exit=0` за < 2 сек **в момент применения**, но GUI Package Center показывал «Сбой подключения», и через ребут CLI снова отдавал `exit=124`.

## Ссылки

- Замещающее решение: [[ADR-0003-package-center-via-external-proxy]] и [[Solving-package-center-external-proxy]].
- Сопутствующая проблема URL: [[ADR-0002-stale-update-urls-in-synoinfo-after-dsm-upgrade]].
- Deprecated-инструкция (с шагами отката): [[Solving-package-center-ocsp-cloudflare-blackhole]].
- Cloudflare IP ranges: <https://www.cloudflare.com/ips-v4/>
- Sectigo OCSP/CRL endpoints: <https://sectigo.com/resource-library/cps-and-cp>
