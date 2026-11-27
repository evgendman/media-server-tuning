# Telegram MTProto Proxy on MINI-Ubuntu / Telegram MTProto-прокси на MINI-Ubuntu

**Status / Статус:** installed; Telegram client operation verified; network-namespace recovery verified 2026-11-27; media-transfer follow-up remains open / установлено; работа Telegram-клиента подтверждена; восстановление сетевой привязки проверено 2026-11-27; проверка передачи медиа остаётся открытой  
**Last verified / Последняя проверка:** 2026-11-27  
**Purpose / Назначение:** document the MTProxy deployment, its integration with the existing Gluetun/WireGuard gateway, the daily restart timer, and operational checks without publishing real endpoints or credentials / описать развёртывание MTProxy, интеграцию с существующим шлюзом Gluetun/WireGuard, таймер ежедневного перезапуска и эксплуатационные проверки без публикации реальных адресов и секретов.

> **Public-documentation safety / Безопасность публичной документации:** all host addresses, public endpoints, VPN client addresses, MTProxy secrets, WireGuard keys and other credentials are omitted. Placeholders such as \`MINI_LAN_IP\`, \`VPS_PUBLIC_ENDPOINT\`, \`WG_CLIENT_IP\`, \`VPS_EGRESS_IP\` and \`MTProxy_SECRET\` are pseudonyms, not literal values. Do not replace them with real values in this public file. / Все адреса узлов, публичные endpoint-адреса, адреса VPN-клиентов, секреты MTProxy, ключи WireGuard и другие учётные данные исключены. Обозначения-псевдонимы не являются реальными значениями и не должны заменяться фактическими данными в этом публичном файле.

---

# Русская версия

## 1. Назначение и текущая схема

На MINI-Ubuntu установлен классический Telegram MTProto Proxy из официального Docker-образа \`telegrammessenger/proxy:1.4\`. Использована та же версия и тот же опубликованный registry digest, что и на действующем VPS. Это сделано намеренно: ранее проверенная реализация на VPS работала стабильнее альтернатив, поэтому при переносе не меняли программную реализацию.

Прокси предназначен для клиентов в домашней локальной сети. Внешний доступ через Дубль/VPS в рамках этой установки **не настраивался**.

Архитектура:

\`\`\`text
Telegram-клиент в доверенной LAN
        |
        | MTProto / TCP
        | MINI_LAN_IP:8443
        v
MINI-Ubuntu
  Docker: proxy-gateway (Gluetun)
  ├─ HTTP proxy :8888 (существующий сервис)
  ├─ WireGuard client interface: tun0
  ├─ Gluetun firewall / kill switch
  └─ опубликованный TCP :8443 -> namespace TCP :443
          |
          | общий network namespace
          |
       Docker: mtproto-proxy
       ├─ telegrammessenger/proxy:1.4
       ├─ MTProxy listener :443
       └─ секрет из /data/secret
          |
          | исходящие соединения через tun0
          v
      VPS / WireGuard peer
          |
          v
      Telegram DC endpoints
\`\`\`

## 2. Как MTProxy связан с существующим HTTP-прокси

Это важное архитектурное уточнение: MTProxy **не настроен как HTTP-клиент и не отправляет запросы CONNECT на локальный HTTP proxy :8888**. В MTProxy нет штатного параметра для такого upstream HTTP proxy.

Вместо этого используется Docker network namespace sharing:

\`\`\`yaml
network_mode: "service:gluetun"
\`\`\`

Сервисы \`mtproxy\` и \`gluetun\` используют одно сетевое пространство. Поэтому исходящий трафик MTProxy проходит через сетевой стек Gluetun и его WireGuard-интерфейс \`tun0\`, то есть использует тот же VPN-маршрут и внешний egress, что и встроенный HTTP-прокси Gluetun.

Практические последствия:

- MTProxy не имеет отдельного Docker bridge и собственного независимого маршрута по умолчанию.
- Отдельный redsocks, proxychains или второй WireGuard-клиент для MTProxy не устанавливался.
- Gluetun firewall/kill switch применяется к сетевому пространству, в котором работает MTProxy.
- При отказе WireGuard исходящие соединения MTProxy должны блокироваться правилами Gluetun, а не уходить напрямую через домашний шлюз.
- Перезапуск MTProxy не требует перезапуска Gluetun и WireGuard.
- HTTP proxy :8888 остаётся самостоятельным explicit proxy для приложений, которые явно настроены его использовать. MTProxy им не пользуется на уровне HTTP CONNECT.

Подробная документация существующего шлюза: [HTTP proxy through WireGuard on MINI-Ubuntu](http-proxy-wireguard.md).

## 3. Версия и идентификация образа

| Параметр | Значение |
|---|---|
| Docker image | \`telegrammessenger/proxy:1.4\` |
| Registry digest | \`sha256:73210d43c8f8e4c888ba4e30d6daf7742528e9134252a1cd538caabf5e24a597\` |
| Platform | \`linux/amd64\` |
| Engine в startup log | \`mtproxy-0.01\`, build 2020-02-28 |
| Container name | \`mtproto-proxy\` |
| Compose service | \`mtproxy\` |
| Restart policy | \`unless-stopped\` |

При переносе на MINI-Ubuntu registry digest, архитектура и все шесть root filesystem layers были сверены с образом на VPS. Совпадение подтверждено. Важно отличать registry digest от локального Docker image/config ID: это разные идентификаторы; для проверки идентичности здесь использовались digest, платформа и список слоёв.

Образ закреплён по digest, а не только по изменяемому тегу.

## 4. Docker Compose: фактическая структура

Compose-файл шлюза находится в:

- \`/opt/proxy-gateway/compose.yaml\`

Данные MTProxy:

- каталог: \`/opt/mtproto-proxy/data\`;
- файл секрета: \`/opt/mtproto-proxy/data/secret\`;
- каталог принадлежит \`root:root\`, режим \`0700\`;
- файл секрета принадлежит \`root:root\`, режим \`0600\`.

Ниже — санитарно отредактированная схема текущего Compose. \`MINI_LAN_IP\` — псевдоним фактического LAN-адреса; его нельзя копировать буквально вместо реального адреса в рабочую конфигурацию.

\`\`\`yaml
services:
  gluetun:
    image: qmcgaw/gluetun:v3.41.3
    container_name: proxy-gateway
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    volumes:
      - /opt/proxy-gateway/wireguard/wg0.conf:/gluetun/wireguard/wg0.conf:ro
    environment:
      VPN_SERVICE_PROVIDER: "custom"
      VPN_TYPE: "wireguard"
      HTTPPROXY: "on"
      HTTPPROXY_LISTENING_ADDRESS: ":8888"
      HTTPPROXY_STEALTH: "on"
      FIREWALL_INPUT_PORTS: "8888,443"
      LOG_LEVEL: "info"
    ports:
      - "MINI_LAN_IP:8888:8888/tcp"
      - "MINI_LAN_IP:8443:443/tcp"

  mtproxy:
    image: telegrammessenger/proxy:1.4@sha256:73210d43c8f8e4c888ba4e30d6daf7742528e9134252a1cd538caabf5e24a597
    container_name: mtproto-proxy
    restart: unless-stopped
    network_mode: "service:gluetun"
    depends_on:
      gluetun:
        condition: service_healthy
    environment:
      WORKERS: "2"
    volumes:
      - /opt/mtproto-proxy/data:/data
\`\`\`

Публикация портов выполняется на сервисе Gluetun, потому что именно его сетевое пространство разделяет MTProxy:

- \`MINI_LAN_IP:8888 -> container TCP/8888\` — существующий HTTP proxy;
- \`MINI_LAN_IP:8443 -> shared namespace TCP/443\` — новый MTProxy listener.

Переменная Gluetun \`FIREWALL_INPUT_PORTS\` расширена с \`8888\` до \`8888,443\`, чтобы разрешить входящий порт MTProxy внутри сетевого пространства Gluetun. Остальные параметры VPN-клиента, WireGuard-профиль и его mount read-only сохранены.

### Жизненный цикл Gluetun и MTProxy

Сервис mtproxy использует Docker Compose `network_mode: "service:gluetun"`. Docker разрешает эту ссылку в сетевое пространство конкретного экземпляра контейнера Gluetun; в `docker inspect` фактическая привязка отображается как `NetworkMode=container:<GLUETUN_CONTAINER_ID>`.

Если `proxy-gateway` пересоздан (например, после замены WireGuard-профиля), новый контейнер получает другой ID. Уже созданный MTProxy продолжает ссылаться на старый ID. Обычный `docker restart mtproto-proxy` не меняет эту привязку и не устраняет ошибку `joining network namespace ... No such container`.

После пересоздания Gluetun:
1. Убедиться, что `proxy-gateway` запущен и имеет состояние `healthy`.
2. Пересоздать только зависимый сервис MTProxy:

```bash
sudo docker compose -f /opt/proxy-gateway/compose.yaml up -d --no-deps --force-recreate mtproxy
```

3. Проверить, что `NetworkMode=container:<ID>` у MTProxy содержит актуальный ID Gluetun:

```bash
sudo docker inspect proxy-gateway --format 'ID={{.Id}} Status={{.State.Status}} Health={{if .State.Health}}{{.State.Health.Status}}{{end}}'
sudo docker inspect mtproto-proxy --format 'Status={{.State.Status}} NetworkMode={{.HostConfig.NetworkMode}}'
```

Команда с `--no-deps` применяется только после проверки здоровья Gluetun и не пересоздаёт сам шлюз. Если в будущем появятся другие сервисы с `network_mode: "service:gluetun"`, их также нужно пересоздавать после замены Gluetun.

Повседневный перезапуск MTProxy при неизменном контейнере Gluetun остаётся обычным; принудительно пересоздавать MTProxy каждый день не нужно. Этот порядок восстановления был применён 2026-11-27 после пересоздания `proxy-gateway`; новый контейнер MTProxy получил актуальное сетевое пространство, а работа прокси была подтверждена в Telegram.

## 5. Секрет и параметры MTProxy

Сохранён существующий MTProxy secret с VPS, чтобы не менять конфигурацию Telegram-клиентов. Сам секрет, его хеш, base64-представление, строка подключения и содержимое файлов с ключами в репозиторий не добавлялись.

Docker image использует стандартный startup script \`/run.sh\`. Он читает секрет из \`/data/secret\`, загружает актуальную конфигурацию Telegram DC из официального endpoint \`core.telegram.org/getProxyConfig\`, определяет исходящий адрес и запускает MTProxy.

Наблюдавшиеся параметры запуска:

| Параметр | Значение / назначение |
|---|---|
| HTTP listener \`-H\` | \`443\` внутри общего network namespace |
| Statistics/RPC port \`-p\` | \`2398\`, не публикуется на LAN |
| Workers \`-M\` | \`2\` |
| Connection limit \`-C\` | \`60000\` |
| AES config file | image-managed file under \`/etc/telegram/\` |
| Backend DC config | \`/etc/telegram/backend.conf\`, downloaded at container start |
| Skip DH check | \`--allow-skip-dh\` |
| NAT information | generated by image startup script from the current namespace route and observed egress address |
| Secret source | \`/data/secret\`, mounted from the host data directory |

The effective NAT information is dynamic; it is not hardcoded in Compose. The startup script derives the namespace source address and external egress address. No literal values are published here.

**Shared secret note:** both the VPS and MINI-Ubuntu proxies temporarily use the same secret. Clients distinguish the proxies by server address and port, not by a unique secret per host. This is operationally valid, but the secret is a shared credential: rotate it on both proxies and update clients if it is ever exposed.

## 6. Порт, область доступности и безопасность

На MINI-Ubuntu MTProxy предназначен только для доверенной локальной сети:

- listener внутри контейнерного network namespace: TCP/443;
- опубликованный host endpoint: \`MINI_LAN_IP:8443/TCP\`;
- HTTP proxy Gluetun: \`MINI_LAN_IP:8888/TCP\`;
- UDP для MTProxy не публиковался;
- внешний проброс MTProxy через Дубль/VPS на этом этапе не настраивался.

Не следует публиковать HTTP proxy :8888 в Интернет. Для будущего удалённого доступа к MTProxy можно использовать существующую схему перенаправления TCP через WireGuard на Дубле, но это отдельная задача и требует отдельной проверки firewall, NAT и клиентского адреса. Не нужно открывать домашний Gluetun HTTP proxy наружу только ради удалённого MTProxy.

## 7. Systemd: ежедневный перезапуск MTProxy

Для периодического перезапуска используется отдельная systemd-служба и монотонный таймер. Это повторяет рабочую схему, ранее применявшуюся на VPS.

Файл \`/etc/systemd/system/mtproto-proxy-restart.service\`:

\`\`\`ini
[Unit]
Description=Restart Telegram MTProto Proxy
After=docker.service
Requires=docker.service

[Service]
Type=oneshot
ExecStart=/usr/bin/docker restart mtproto-proxy
\`\`\`

Файл \`/etc/systemd/system/mtproto-proxy-restart.timer\`:

\`\`\`ini
[Unit]
Description=Daily restart of Telegram MTProto Proxy

[Timer]
OnBootSec=1h
OnUnitActiveSec=24h
Unit=mtproto-proxy-restart.service

[Install]
WantedBy=timers.target
\`\`\`

Таймер включён:

\`\`\`bash
sudo systemctl daemon-reload
sudo systemctl enable --now mtproto-proxy-restart.timer
\`\`\`

Расписание:

- первая активация — через один час после загрузки MINI-Ubuntu;
- дальнейшие активации — через 24 часа после предыдущего запуска службы;
- используется монотонный таймер; \`Persistent=true\` не требуется и не задан;
- если таймер включается уже после того, как \`OnBootSec=1h\` истёк в текущей загрузке, systemd может выполнить первый запуск сразу. Это наблюдалось при первоначальном включении таймера после установки.

Служба выполняет только \`docker restart mtproto-proxy\`. Она не перезапускает \`proxy-gateway\`, HTTP proxy или WireGuard. Во время перезапуска MTProxy текущие клиентские сессии прерываются и должны переподключиться.
Таймер намеренно выполняет обычный перезапуск, а не пересоздание контейнера: при неизменном Gluetun это правильно и не создаёт лишних контейнеров. Этот таймер **не исправляет** устаревшую сетевую привязку после пересоздания Gluetun; в таком случае требуется процедура из раздела «Жизненный цикл Gluetun и MTProxy» выше.

Проверка таймера и журнала:

\`\`\`bash
systemctl list-timers mtproto-proxy-restart.timer
systemctl status mtproto-proxy-restart.timer
systemctl status mtproto-proxy-restart.service
journalctl -u mtproto-proxy-restart.service
\`\`\`

Ручной перезапуск через ту же systemd-службу:

\`\`\`bash
sudo systemctl start mtproto-proxy-restart.service
\`\`\`

## 8. Развёртывание и резервная копия

Установка была выполнена из отдельного сценария:

- \`/home/dmitriev-ev/install-mtproxy-mini.sh\`

Сценарий проверял host name, состояние и health Gluetun, формат и SHA-256 временно перенесённого секрета, отсутствие занятого host TCP/8443 и валидность Compose. Образ проверялся по registry digest, платформе \`linux/amd64\` и всем шести слоям перед изменением Compose.

Перед изменением был создан локальный backup:

- \`/opt/proxy-gateway/compose.yaml.before-mtproxy.<UTC_TIMESTAMP>\`

Фактическое резервное имя содержит timestamp; приведённый путь — шаблон. Секрет сначала находился в staging-каталоге пользователя, затем был установлен в root-only data directory. После установки временная копия удалена. Реальные backup-файлы, ключи и секреты в GitHub не публикуются.

## 9. Проверенное состояние на 2026-11-26

| Проверка | Результат |
|---|---|
| Compose deployment | оба контейнера созданы |
| Gluetun health | \`healthy\` |
| MTProxy container | \`running\` |
| MTProxy network mode | \`container:<GLUETUN_CONTAINER_ID>\` |
| Host TCP/8443 | локальная TCP-проверка успешна |
| Secret permissions | \`root:root\`, \`0600\` |
| Systemd timer | \`enabled\`, \`active (waiting)\` |
| Timer service execution | первый запуск завершился \`status=0/SUCCESS\` |
| Telegram client | подключение к LAN proxy подтверждено; текстовые сообщения приходили |
| Photos/videos | на момент последней проверки загрузка медиа оставалась проблемной; причина не установлена |

### Выборочная проверка Telegram DC endpoints

Временная проверка 19 уникальных адресов из загруженного \`backend.conf\` выполнялась из network namespace MTProxy с TCP connect timeout 5 секунд:

- 15 endpoint-проверок завершились успешным TCP connect;
- 4 endpoint-проверки завершились timeout.

Для публичной документации адреса заменены псевдонимами:

| DC / endpoint alias | Port | MINI-Ubuntu via Gluetun/WireGuard | VPS direct control |
|---|---:|---|---|
| DC1 primary | TCP/8888 | timeout | TCP connect succeeded |
| DC3 primary | TCP/8888 | timeout | timeout |
| DC203 endpoint | TCP/443 | timeout | timeout |
| DC5 alternate B | TCP/8888 | timeout | TCP connect succeeded |
| DC5 alternate A | TCP/8888 | TCP connect succeeded | timeout |

Остальные проверенные адреса в том тесте отвечали. Результаты относятся только к моменту проверки и не являются постоянным списком заблокированных Telegram-узлов. Тот же endpoint может вести себя по-разному с разных точек выхода и в разное время. Отдельные тайм-ауты на VPS показывают, что нельзя объяснять все неудачи только Gluetun/WireGuard.

Маршрутизация к трём контрольным Telegram IP из сетевого пространства MTProxy показывала интерфейс \`tun0\` и источник \`WG_CLIENT_IP\`. Это подтверждает выбор маршрута через WireGuard, но само по себе не доказывает доступность каждого удалённого endpoint.

Фильтрация последних 10 минут логов по словам \`connection\`, \`disconnected\`, \`error\`, \`failed\` и \`timeout\` не выявила строк. Это не является полным аудитом сетевых ошибок: стандартный уровень логирования не гарантирует подробное журналирование каждого исходящего соединения.

### Открытые проверки

1. Повторить TCP-проверку endpoint-адресов позднее: конфигурация \`backend.conf\` загружается при запуске контейнера и может обновляться Telegram.
2. Проверить отправку и загрузку фото/видео через LAN MTProxy на нескольких клиентах и в разное время.
3. При сохранении проблемы сопоставить время попытки загрузки медиа с журналом MTProxy и активными TCP-соединениями в namespace Gluetun.
4. Отдельно проверить fail-closed поведение: временно отключить только peer \`MINI-Ubuntu-proxy\` в wg-easy, убедиться, что исходящие подключения MTProxy прекращаются, затем восстановить peer. Такой тест требует согласованного окна, так как одновременно перестанет работать и HTTP proxy Gluetun.
5. Не менять \`backend.conf\` вручную и не добавлять фиксированные Telegram IP без отдельного подтверждения.

---

# English version

## 1. Purpose and deployed topology

The classic Telegram MTProto Proxy was installed on MINI-Ubuntu using the official Docker image \`telegrammessenger/proxy:1.4\`. The same image version and registry digest as the existing VPS deployment were retained to preserve the previously tested implementation.

The current deployment is intended for clients on the trusted home LAN. Public access through the Duble/VPS forwarding path has **not** been configured as part of this deployment.

\`\`\`text
Telegram client on trusted LAN
        |
        | MTProto / TCP
        | MINI_LAN_IP:8443
        v
MINI-Ubuntu
  Docker: proxy-gateway (Gluetun)
  ├─ existing HTTP proxy :8888
  ├─ WireGuard client interface: tun0
  ├─ Gluetun firewall / kill switch
  └─ published TCP :8443 -> shared namespace TCP :443
          |
          | shared network namespace
          |
       Docker: mtproto-proxy
       ├─ telegrammessenger/proxy:1.4
       ├─ MTProxy listener :443
       └─ secret loaded from /data/secret
          |
          | outbound through tun0
          v
      VPS / WireGuard peer
          |
          v
      Telegram DC endpoints
\`\`\`

## 2. Relationship with the existing HTTP proxy

MTProxy is **not configured as an HTTP client and does not send HTTP CONNECT requests to the local HTTP proxy on port 8888**. The MTProxy implementation has no native upstream HTTP proxy option in this deployment.

Instead, Docker network namespace sharing is used:

\`\`\`yaml
network_mode: "service:gluetun"
\`\`\`

The \`mtproxy\` and \`gluetun\` services share one network namespace. Consequently, MTProxy outbound traffic uses Gluetun's network stack and WireGuard interface \`tun0\`. It therefore follows the same VPN route and VPS egress path as Gluetun's embedded HTTP proxy.

Consequences:

- MTProxy has no separate Docker bridge or independent default route.
- No separate redsocks, proxychains or second WireGuard client was installed.
- Gluetun's firewall/kill switch applies to the shared network namespace.
- If WireGuard fails, Gluetun should block MTProxy outbound traffic rather than allowing a direct fallback through the home gateway.
- Restarting MTProxy does not require restarting Gluetun or WireGuard.
- The HTTP proxy on TCP/8888 remains an explicit proxy for applications configured to use it. MTProxy does not use it as an HTTP CONNECT upstream.

See [HTTP proxy through WireGuard on MINI-Ubuntu](http-proxy-wireguard.md) for the existing gateway deployment.

## 3. Image version and identity

| Setting | Value |
|---|---|
| Docker image | \`telegrammessenger/proxy:1.4\` |
| Registry digest | \`sha256:73210d43c8f8e4c888ba4e30d6daf7742528e9134252a1cd538caabf5e24a597\` |
| Platform | \`linux/amd64\` |
| Engine in startup log | \`mtproxy-0.01\`, build 2020-02-28 |
| Container name | \`mtproto-proxy\` |
| Compose service | \`mtproxy\` |
| Restart policy | \`unless-stopped\` |

During deployment, the registry digest, architecture and all six root filesystem layers were compared with the image on the VPS and matched. Registry digest and local Docker image/config ID are different identifiers; digest, platform and layer list were used for this comparison.

The image is pinned by digest rather than relying only on a mutable tag.

## 4. Docker Compose structure

The Gluetun Compose file is:

- \`/opt/proxy-gateway/compose.yaml\`

MTProxy data:

- directory: \`/opt/mtproto-proxy/data\`;
- secret file: \`/opt/mtproto-proxy/data/secret\`;
- directory owner/mode: \`root:root\`, \`0700\`;
- secret file owner/mode: \`root:root\`, \`0600\`.

The following is a sanitized representation of the current Compose structure. \`MINI_LAN_IP\` is a pseudonym for the actual LAN address and must not be copied literally into a working Compose file.

\`\`\`yaml
services:
  gluetun:
    image: qmcgaw/gluetun:v3.41.3
    container_name: proxy-gateway
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    volumes:
      - /opt/proxy-gateway/wireguard/wg0.conf:/gluetun/wireguard/wg0.conf:ro
    environment:
      VPN_SERVICE_PROVIDER: "custom"
      VPN_TYPE: "wireguard"
      HTTPPROXY: "on"
      HTTPPROXY_LISTENING_ADDRESS: ":8888"
      HTTPPROXY_STEALTH: "on"
      FIREWALL_INPUT_PORTS: "8888,443"
      LOG_LEVEL: "info"
    ports:
      - "MINI_LAN_IP:8888:8888/tcp"
      - "MINI_LAN_IP:8443:443/tcp"

  mtproxy:
    image: telegrammessenger/proxy:1.4@sha256:73210d43c8f8e4c888ba4e30d6daf7742528e9134252a1cd538caabf5e24a597
    container_name: mtproto-proxy
    restart: unless-stopped
    network_mode: "service:gluetun"
    depends_on:
      gluetun:
        condition: service_healthy
    environment:
      WORKERS: "2"
    volumes:
      - /opt/mtproto-proxy/data:/data
\`\`\`

Ports are published on Gluetun because MTProxy shares its network namespace:

- \`MINI_LAN_IP:8888 -> container TCP/8888\` — existing HTTP proxy;
- \`MINI_LAN_IP:8443 -> shared namespace TCP/443\` — new MTProxy listener.

Gluetun's \`FIREWALL_INPUT_PORTS\` was extended from \`8888\` to \`8888,443\` to permit incoming MTProxy traffic in the shared namespace. The existing VPN client settings, WireGuard profile and read-only profile mount were preserved.

### Gluetun and MTProxy lifecycle

The mtproxy service uses Docker Compose `network_mode: "service:gluetun"`. Docker resolves this reference to the network namespace of a specific Gluetun container instance; in `docker inspect` the effective binding appears as `NetworkMode=container:<GLUETUN_CONTAINER_ID>`.

When `proxy-gateway` is recreated (for example, after replacing the WireGuard profile), the replacement receives a new ID. An existing MTProxy container continues to reference the old ID. A plain `docker restart mtproto-proxy` does not update that binding and cannot fix an error such as `joining network namespace ... No such container`.

After recreating Gluetun:
1. Confirm that `proxy-gateway` is running and `healthy`.
2. Recreate only the dependent MTProxy service:

```bash
sudo docker compose -f /opt/proxy-gateway/compose.yaml up -d --no-deps --force-recreate mtproxy
```

3. Verify that MTProxy's `NetworkMode=container:<ID>` contains the current Gluetun container ID:

```bash
sudo docker inspect proxy-gateway --format 'ID={{.Id}} Status={{.State.Status}} Health={{if .State.Health}}{{.State.Health.Status}}{{end}}'
sudo docker inspect mtproto-proxy --format 'Status={{.State.Status}} NetworkMode={{.HostConfig.NetworkMode}}'
```

Use `--no-deps` only after confirming Gluetun is healthy; this command does not recreate the gateway itself. If other services using `network_mode: "service:gluetun"` are added later, they must also be recreated after Gluetun is replaced.

For routine maintenance, an ordinary MTProxy restart is sufficient while the Gluetun container remains unchanged. MTProxy does not need to be force-recreated every day. This recovery procedure was used on 2026-11-27 after `proxy-gateway` had been recreated; the replacement MTProxy container received the current network namespace and proxy operation was confirmed in Telegram.

## 5. MTProxy secret and runtime parameters

The existing MTProxy secret from the VPS was retained so Telegram clients would not need to be reconfigured. The secret itself, its hash, base64 representation, connection URI and any key-file contents were not committed to the repository.

The Docker image uses its standard startup script \`/run.sh\`. It reads the secret from \`/data/secret\`, downloads the current Telegram DC configuration from the official \`core.telegram.org/getProxyConfig\` endpoint, determines the outbound address and starts the MTProxy engine.

Observed runtime parameters:

| Parameter | Value / purpose |
|---|---|
| HTTP listener \`-H\` | \`443\` inside the shared network namespace |
| Statistics/RPC port \`-p\` | \`2398\`, not published on the LAN |
| Workers \`-M\` | \`2\` |
| Connection limit \`-C\` | \`60000\` |
| AES config file | image-managed file under \`/etc/telegram/\` |
| Backend DC config | \`/etc/telegram/backend.conf\`, downloaded at container start |
| Skip DH check | \`--allow-skip-dh\` |
| NAT information | generated by the image startup script from the current namespace route and observed egress address |
| Secret source | \`/data/secret\`, mounted from the host data directory |

NAT information is dynamic and is not hardcoded in Compose. The startup script derives the namespace source address and external egress address. No literal values are published here.

**Shared-secret note:** the VPS and MINI-Ubuntu proxies temporarily use the same secret. Clients distinguish the proxies by server address and port, not by a unique secret per host. This is operationally valid, but the secret is a shared credential: rotate it on both proxies and update clients if it is exposed.

## 6. Listener scope and security

The MINI-Ubuntu MTProxy is currently intended only for the trusted home LAN:

- listener inside the container network namespace: TCP/443;
- published host endpoint: \`MINI_LAN_IP:8443/TCP\`;
- Gluetun HTTP proxy: \`MINI_LAN_IP:8888/TCP\`;
- no MTProxy UDP port is published;
- external MTProxy forwarding through Duble/VPS has not been configured.

Do not expose HTTP proxy :8888 to the public Internet. Remote access to MTProxy may later use the existing TCP-forwarding path through WireGuard on Duble, but this is a separate task requiring its own firewall, NAT and client-reachability checks. The home Gluetun HTTP proxy does not need to be exposed for remote MTProxy access.

## 7. systemd daily restart timer

A dedicated systemd service and monotonic timer restart only the MTProxy container. This follows the previously used VPS pattern.

\`/etc/systemd/system/mtproto-proxy-restart.service\`:

\`\`\`ini
[Unit]
Description=Restart Telegram MTProto Proxy
After=docker.service
Requires=docker.service

[Service]
Type=oneshot
ExecStart=/usr/bin/docker restart mtproto-proxy
\`\`\`

\`/etc/systemd/system/mtproto-proxy-restart.timer\`:

\`\`\`ini
[Unit]
Description=Daily restart of Telegram MTProto Proxy

[Timer]
OnBootSec=1h
OnUnitActiveSec=24h
Unit=mtproto-proxy-restart.service

[Install]
WantedBy=timers.target
\`\`\`

The timer is enabled with:

\`\`\`bash
sudo systemctl daemon-reload
sudo systemctl enable --now mtproto-proxy-restart.timer
\`\`\`

Schedule:

- first activation: one hour after MINI-Ubuntu boots;
- subsequent activations: every 24 hours after the previous service activation;
- a monotonic timer is used; \`Persistent=true\` is neither needed nor configured;
- if the timer is enabled after \`OnBootSec=1h\` has already elapsed during the current boot, systemd may run the service immediately. This occurred when the timer was first enabled after installation.

The service runs only \`docker restart mtproto-proxy\`. It does not restart \`proxy-gateway\`, the HTTP proxy or WireGuard. Existing Telegram client sessions are interrupted during the restart and must reconnect.
The timer intentionally performs a normal restart rather than recreating the container: that is correct while Gluetun remains the same container and avoids unnecessary recreation. The timer **does not repair** a stale network-namespace reference after Gluetun is recreated; use the “Gluetun and MTProxy lifecycle” procedure above in that case.

Inspect the timer and journal:

\`\`\`bash
systemctl list-timers mtproto-proxy-restart.timer
systemctl status mtproto-proxy-restart.timer
systemctl status mtproto-proxy-restart.service
journalctl -u mtproto-proxy-restart.service
\`\`\`

Trigger a manual restart through the same unit:

\`\`\`bash
sudo systemctl start mtproto-proxy-restart.service
\`\`\`

## 8. Deployment and backup

The installation was performed by a dedicated script:

- \`/home/dmitriev-ev/install-mtproxy-mini.sh\`

The script checked the host name, Gluetun state and health, format and SHA-256 of the temporarily transferred secret, whether host TCP/8443 was already occupied, and Compose validity. Before modifying Compose, it checked the image registry digest, \`linux/amd64\` platform and all six image layers.

A local Compose backup was created before editing:

- \`/opt/proxy-gateway/compose.yaml.before-mtproxy.<UTC_TIMESTAMP>\`

The actual backup filename contains a timestamp; the path above is a template. The secret was first held in a user staging directory, then installed in the root-only data directory. The temporary copy was removed after installation. Actual backup files, keys and secrets are never committed to GitHub.

## 9. Verified state on 2026-11-26

| Check | Result |
|---|---|
| Compose deployment | both containers created |
| Gluetun health | \`healthy\` |
| MTProxy container | \`running\` |
| MTProxy network mode | \`container:<GLUETUN_CONTAINER_ID>\` |
| Host TCP/8443 | local TCP connection check succeeded |
| Secret permissions | \`root:root\`, \`0600\` |
| systemd timer | \`enabled\`, \`active (waiting)\` |
| Timer service execution | first run completed with \`status=0/SUCCESS\` |
| Telegram client | connection to the LAN proxy confirmed; text messages arrived |
| Photos/videos | media loading was still problematic at the latest check; root cause not established |

### Selective Telegram DC endpoint test

A one-time TCP check of 19 unique destinations from the downloaded \`backend.conf\` was performed inside the MTProxy network namespace with a 5-second connect timeout:

- 15 endpoint checks established TCP connections;
- 4 endpoint checks timed out.

Public endpoint addresses are replaced with aliases:

| DC / endpoint alias | Port | MINI-Ubuntu via Gluetun/WireGuard | VPS direct control |
|---|---:|---|---|
| DC1 primary | TCP/8888 | timeout | TCP connect succeeded |
| DC3 primary | TCP/8888 | timeout | timeout |
| DC203 endpoint | TCP/443 | timeout | timeout |
| DC5 alternate B | TCP/8888 | timeout | TCP connect succeeded |
| DC5 alternate A | TCP/8888 | TCP connect succeeded | timeout |

The other tested endpoints responded at that time. These observations are time-specific and do not establish a permanent block list. The same endpoint may behave differently from different egress locations and at different times. Timeouts observed from the VPS as well mean that the failures cannot all be attributed to Gluetun/WireGuard.

Route lookups for three representative Telegram IPs from the MTProxy network namespace showed interface \`tun0\` and source \`WG_CLIENT_IP\`. This confirms route selection through WireGuard but does not prove that every remote endpoint is reachable.

Filtering the last 10 minutes of logs for \`connection\`, \`disconnected\`, \`error\`, \`failed\` and \`timeout\` did not show matching lines. This is not a complete network error audit: the default log level does not guarantee per-destination outbound connection logging.

### Open follow-up checks

1. Repeat endpoint TCP checks later; \`backend.conf\` is downloaded at container startup and may change when Telegram updates it.
2. Test photo and video upload/download through the LAN MTProxy from multiple clients and at different times.
3. If media problems remain, correlate the test time with MTProxy logs and active TCP connections in the Gluetun namespace.
4. Perform a separate fail-closed test: temporarily disable only the \`MINI-Ubuntu-proxy\` peer in wg-easy, verify MTProxy outbound connections stop, then restore the peer. This also interrupts Gluetun's HTTP proxy, so schedule the test accordingly.
5. Do not edit \`backend.conf\` manually or add fixed Telegram IP addresses without separate verification.
