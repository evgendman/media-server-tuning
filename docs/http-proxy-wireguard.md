# HTTP proxy through WireGuard on MINI-Ubuntu / HTTP-прокси через WireGuard на MINI-Ubuntu

**Status / Статус:** implemented and operational; follow-up checks are recorded below / реализовано и работает; оставшиеся проверки перечислены ниже  
**Last verified / Последняя проверка:** 2026-10-16  
**Purpose / Назначение:** record the motivation, architecture, deployed configuration, observed behavior, and safe usage of the LAN HTTP proxy / зафиксировать причины создания, архитектуру, действующую конфигурацию, проверенное поведение и правила использования HTTP-прокси в локальной сети.

> **Public-documentation note / Примечание для публичной документации:** concrete public and LAN host addresses are intentionally redacted. Replace the placeholders with the actual values only in local, uncommitted configuration. / Конкретные публичные и локальные адреса узлов намеренно скрыты. Подставлять фактические значения следует только в локальные конфигурации, не публикуемые в репозитории.

---

# Русская версия

## 1. Назначение и исходная проблема

Цель — предоставить устройствам домашней сети один явный HTTP-прокси, через который выбранные приложения или отдельные функции приложений могут обращаться к внешним сайтам с публичного IP-адреса VPS. Остальной трафик этих же устройств продолжает идти обычным путём через домашний маршрутизатор. Это не глобальный VPN для всей сети, а выборочный сетевой выход, который приложение включает в собственных настройках.

Необходимость возникла из-за неоднородной маршрутизации внешних запросов. На домашнем роутере использовались принудительные маршруты для отдельных доменов и сервисов. В результате разные компоненты одного рабочего процесса — например, Seerr, Radarr, Sonarr и внешние API, к которым они обращаются, — могли выходить в Интернет разными путями. Некоторые конечные адреса или запросы через Cloudflare в таком сценарии периодически блокировались, поэтому операции добавления фильмов и сериалов могли работать непоследовательно: одни запросы проходили, другие завершались ошибкой.

Обратный пример — синхронизация базы JacRed. Для неё туннелированный или принудительно маршрутизированный путь к серверу синхронизации JacRed приводил к ответу Cloudflare 403, тогда как прямой путь мог работать. Поэтому прокси не должен принудительно переводить в VPN весь трафик домашней сети. Он нужен именно там, где выбранному приложению требуется альтернативный исходящий маршрут.

Прокси позволяет выбирать маршрут на уровне приложения или компонента приложения:
- приложение, которому подходит обычный маршрут, не использует прокси;
- приложение, которому нужен выход через VPS, указывает LAN-адрес прокси;
- если плагин позволяет отдельно настраивать поиск и воспроизведение, прокси можно использовать только для поиска, оставив поток воспроизведения прямым.

Смена маршрута не гарантирует, что Cloudflare или другой удалённый сервис обязательно разрешит запрос: сервис может блокировать и адрес VPS. Прокси делает локальный путь выбранного потока более предсказуемым, но не гарантирует обход любой внешней блокировки.

## 2. Архитектура

~~~text
Любой компьютер или приложение в домашней сети
                |
                | HTTP proxy: LAN-IP-МИНИ-ПК:8888
                v
MINI-Ubuntu (LAN-IP-МИНИ-ПК)
  Docker container: proxy-gateway
  Image: qmcgaw/gluetun:v3.41.3
  ├─ встроенный HTTP-прокси, TCP/8888
  ├─ WireGuard client interface: tun0
  ├─ Gluetun firewall / kill switch
  └─ DNS под управлением Gluetun
                |
                | WireGuard / UDP 51820
                v
VPS (IP-АДРЕС-VPS)
  существующий сервер wg-easy
  отдельный peer для клиента прокси
                |
                | выход в Интернет через VPS
                v
Внешний сайт / API
~~~

HTTP-прокси и WireGuard-клиент работают в одном контейнере Gluetun и используют одну сетевую среду. Поэтому исходящие соединения самого прокси защищены правилами Gluetun и направляются через WireGuard, а не через обычный маршрут хоста MINI-Ubuntu.

На мини-ПК не устанавливался отдельный WireGuard-сервис на уровне хоста и не менялся маршрут по умолчанию Ubuntu. Обычный трафик хоста и других приложений сохраняет свой маршрут через домашний шлюз. Сетевой интерфейс WireGuard существует внутри контейнера; при проверке он назывался **tun0**.

На VPS используется существующий **wg-easy**. В его веб-интерфейсе был создан отдельный клиент **MINI-Ubuntu-proxy**. Параметры сервера, его порт, подсеть, NAT и существующие клиенты не перенастраивались. Отдельный WireGuard-сервер не создавался.

## 3. Фактически установленная реализация

### Docker и файлы

| Параметр | Фактическое значение |
|---|---|
| Хост | **MINI-Ubuntu** |
| LAN-адрес хоста | скрыт в публичной документации; используется LAN-IP-МИНИ-ПК |
| Каталог проекта | **/opt/proxy-gateway** |
| Compose-файл | **/opt/proxy-gateway/compose.yaml** |
| Docker Compose service | **gluetun** |
| Имя контейнера | **proxy-gateway** |
| Образ | **qmcgaw/gluetun:v3.41.3** |
| Загруженный digest | **sha256:fa19cc76b2af13d57a8d3dc3066f2ada061b1c761b8aecf989b3877c0486e027** |
| Рабочая копия WireGuard-профиля | **/opt/proxy-gateway/wireguard/wg0.conf** |
| Права на рабочий профиль | **0600**, владелец **root:root** |
| Монтирование профиля | только для чтения (**:ro**) |
| Опубликованный порт | **LAN-IP-МИНИ-ПК:8888 → контейнер TCP/8888** |
| Аутентификация HTTP-прокси | отключена: имя пользователя и пароль не заданы |
| Политика перезапуска | **unless-stopped** |

Образ был загружен и проверен по digest. Контейнер **proxy-gateway** перешёл в состояние **healthy**. Docker показывает порты 8000/tcp и 8388/tcp/udp как доступные внутри образа, но они не опубликованы на хост. Из хоста опубликован только TCP/8888, привязанный к LAN-адресу мини-ПК.

Ниже — санитарно отредактированный фрагмент Compose-конфигурации. Вместо строки **LAN-IP-МИНИ-ПК** в реальном локальном файле используется фактический LAN-адрес; placeholder не предназначен для буквального копирования в рабочий Compose-файл.

~~~yaml
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
      FIREWALL_INPUT_PORTS: "8888"
      LOG_LEVEL: "info"
    ports:
      - "LAN-IP-МИНИ-ПК:8888:8888/tcp"
~~~

Встроенный HTTP-прокси Gluetun используется вместо отдельной установки Tinyproxy. Отдельные Nginx, stunnel или публичный HTTP-прокси на VPS не нужны.

### WireGuard-клиент

Новый peer был создан в действующем **wg-easy** на VPS. В контейнер передана отдельная рабочая копия профиля; приватный и pre-shared ключи в документ не включаются.

Параметры, не содержащие секретов:
- Endpoint: **IP-АДРЕС-VPS:51820**;
- адрес клиента — отдельный адрес в существующей VPN-подсети;
- рабочий **AllowedIPs**: **0.0.0.0/0**;
- **PersistentKeepalive**: **25**;
- IPv6-маршрут **::/0** удалён из рабочей копии профиля, так как IPv6-forwarding на VPS был выключен;
- исходная строка DNS из WireGuard-профиля удалена из рабочей копии; DNS обслуживается механизмами Gluetun.

Исходный скачанный профиль находился во временном файле **/tmp/proxy-gateway.conf**; рабочая копия хранится в каталоге, доступном только root, и подключена к контейнеру read-only. После окончательного подтверждения, что исходная копия больше не нужна, её следует удалить, поскольку она содержит приватный ключ клиента.

### Кратко о Gluetun

Gluetun — контейнерный VPN-клиент с собственным управлением маршрутизацией и firewall. В данном развёртывании он выполняет две связанные задачи: устанавливает WireGuard-туннель к VPS и предоставляет встроенный HTTP-прокси. Оба компонента находятся в одном сетевом окружении, поэтому правила firewall Gluetun ограничивают исходящие соединения прокси.

### DNS

DNS передан под управление Gluetun, а не оставлен настройкой из исходного WireGuard-профиля. В сводке запуска Gluetun были указаны внутренний DNS-адрес, DNS-forwarder и Cloudflare в качестве upstream resolver с типом DoT (DNS over TLS); DNS-кэширование включено, IPv6 в DNS-настройках Gluetun отключён.

**Оговорка:** в том же журнале запуска также появилась строка, указывающая на plaintext DNS по адресу публичного резолвера. Поэтому фактическую схему upstream DNS нельзя считать окончательно подтверждённой только по этим сообщениям. При обслуживании нужно проверить актуальную конфигурацию и документацию установленной версии Gluetun и выяснить, какой upstream реально используется. Не следует без этой проверки утверждать, что каждый DNS-запрос гарантированно зашифрован по DoT.

## 4. Как движется трафик

### Входящий путь

Клиентское устройство подключается к HTTP-прокси по адресу **LAN-IP-МИНИ-ПК**, порт **8888**. Docker публикует порт только на LAN IPv4-адресе мини-ПК, а не на всех адресах хоста. На VPS порт HTTP-прокси не открывался. Доступность с другого компьютера домашней сети проверена успешно.

Это явный (explicit) HTTP-прокси, а не прозрачная маршрутизация. Устройство или приложение должно поддерживать настройку HTTP-прокси и использовать этот адрес. Прокси не перехватывает автоматически весь трафик программ, которые не настроены на его использование.

### Исходящий путь и kill switch

Прокси работает внутри сетевого пространства Gluetun, где активен WireGuard-интерфейс **tun0**. Firewall Gluetun имеет политику **OUTPUT DROP**; из просмотренных правил разрешены loopback, существующие соединения, необходимые обращения к локальной Docker-подсети, UDP к WireGuard endpoint и выход через **tun0**. Это kill switch: если туннель недоступен, новые внешние соединения не должны уходить напрямую через обычный Docker gateway.

При этом таблица маршрутизации контейнера показывала default route через Docker bridge. Это не означает, что прикладной трафик свободно выходит напрямую: Gluetun управляет маршрутизацией и ограничивает её firewall. Для безопасности важны одновременно маршрутизация и правила kill switch.

## 5. Проверенные результаты

| Тест | Результат |
|---|---|
| Старт Gluetun | Контейнер запущен; журнал сообщил о настройке WireGuard, готовности DNS и HTTP-прокси |
| Состояние контейнера | **healthy** |
| HTTP proxy | слушает **:8888** внутри контейнера |
| HTTPS-запрос через прокси с мини-ПК | **https://api.ipify.org** вернул IP-адрес VPS, HTTP 200 |
| HTTPS-запрос через прокси с другого ПК в домашней сети | вернул IP-адрес VPS, HTTP 200 |
| Прямой запрос с самого мини-ПК без прокси | вернул другой публичный IP; основной маршрут остался через домашний шлюз |
| Тест отказа туннеля | peer **MINI-Ubuntu-proxy** временно отключали в **wg-easy**; запрос через прокси завершился timeout, HTTP 000, домашний IP не вернулся |
| Восстановление | после повторного включения peer запрос снова вернул IP-адрес VPS, HTTP 200 |
| Nginx, маршруты хоста, остальные Docker-сервисы | для установки прокси не перенастраивались |

Тест с отключением peer показывает, что внешний запрос через прокси перестал успешно выполняться, а после восстановления WireGuard снова заработал. Вместе с политикой **OUTPUT DROP** это подтверждает ожидаемое fail-closed поведение в проверенном сценарии. Это не был полный аудит с захватом пакетов; если потребуется более строгая проверка отсутствия утечек, можно отдельно провести тест с packet capture.

На VPS **wg-easy** остался существующим сервером. Добавлен только отдельный peer для прокси-клиента; настройки сервера и других клиентов не менялись.

## 6. Использование в приложениях

Общие настройки прокси:

~~~text
Proxy type: HTTP
Host:       LAN-IP-МИНИ-ПК
Port:       8888
Username:   not set
Password:   not set
~~~

В настройках приложения обычно выбираются HTTP proxy, hostname с LAN-IP мини-ПК, port **8888**, SSL/TLS к самому proxy — выключен, username/password — пустые.

### Seerr, Radarr и Sonarr

Прокси можно задать в сетевых настройках Seerr, Radarr и Sonarr, если внешние запросы соответствующего сервиса периодически блокируются или идут по нежелательному маршруту. Настройка прокси в одном сервисе не меняет маршрут другого сервиса: каждый процесс должен быть настроен отдельно.

Для Seerr в Docker адрес **127.0.0.1** не подходит: внутри контейнера Seerr он обозначает сам контейнер, а не хост MINI-Ubuntu. Нужно использовать LAN-IP мини-ПК и порт **8888**. При этом обращения Seerr к внутренним API Radarr/Sonarr — отдельный тип соединений, который не следует без необходимости направлять через внешний прокси.

Прокси полезен как выборочный альтернативный выход, когда Cloudflare или другой внешний поставщик по-разному обрабатывает запросы с разных исходящих IP. Но он не гарантирует доступность сервиса, если IP-адрес VPS тоже заблокирован.

### Kodi и Elementum

Если установленная версия Elementum позволяет настраивать прокси в параметрах самого плагина, можно направлять через него только запросы поиска, оставляя воспроизведение и передачу потока прямыми. Такое разделение задаётся настройками и логикой самого плагина: общий HTTP-прокси не может самостоятельно определить, является запрос поиском или воспроизведением. Точные настройки следует сверить с установленной версией Elementum; эта функциональная детализация не проверялась в рамках развёртывания самого прокси.

### JacRed

Для синхронизации JacRed может потребоваться прямой маршрут, если запросы к серверу синхронизации через альтернативный выход получают Cloudflare 403. Не нужно автоматически настраивать этот прокси для каждого сервиса. Задача архитектуры — направлять через VPN только нужные потоки, оставляя заведомо рабочие или требующие прямого выхода запросы на обычном маршруте.

## 7. Шифрование и безопасность

Для HTTPS-сайта TLS-сеанс устанавливается между клиентским приложением и конечным сайтом. При использовании HTTP-прокси клиент отправляет **CONNECT**, после чего TLS-трафик к сайту проходит через прокси без расшифровки самим прокси. На участке от MINI-Ubuntu до VPS этот поток дополнительно переносится внутри WireGuard-туннеля. Поэтому для HTTPS-запроса на участке мини-ПК—VPS есть два слоя защиты: TLS конечного сайта внутри WireGuard.

Это **не** означает, что соединение от клиентского устройства до прокси тоже зашифровано. В текущей конфигурации LAN-участок использует обычный HTTP-прокси без TLS. Для обычного HTTP-сайта внутреннего TLS-слоя нет: WireGuard защищает путь до VPS, но соединение от VPS до такого HTTP-сайта не зашифровано HTTPS.

Аутентификация прокси пока отключена по принятому решению, поскольку он предназначен для доверенной локальной сети. Любое устройство или процесс, имеющий сетевой доступ к LAN-IP мини-ПК и TCP/8888, потенциально может использовать его. Поэтому:
- не настраивать проброс TCP/8888 с Интернет-порта роутера;
- не считать отсутствие пароля заменой сетевому ограничению;
- если доступ расширится за пределы доверенной LAN, включить **HTTPPROXY_USER** и **HTTPPROXY_PASSWORD** и отдельно рассмотреть TLS между клиентом и прокси;
- помнить, что Basic-аутентификация сама по себе не шифрует логин и пароль.

**HTTPPROXY_STEALTH=on** отключает добавление некоторых HTTP-заголовков прокси к запросам. Это не шифрование и не полноценная система контроля доступа.

## 8. Обслуживание и оставшиеся задачи

Проверить контейнер:

~~~bash
sudo docker ps --filter 'name=^/proxy-gateway$'
sudo docker logs --tail 100 proxy-gateway
~~~

Посмотреть настройки Compose:

~~~bash
sudo cat /opt/proxy-gateway/compose.yaml
~~~

Проверить внешний адрес через прокси с MINI-Ubuntu или другого компьютера:

~~~bash
curl -sS --connect-timeout 5 --max-time 20 \
  -x http://LAN-IP-МИНИ-ПК:8888 \
  https://api.ipify.org \
  -w '\nHTTP=%{http_code}\n'
~~~

В публичном документе LAN-IP мини-ПК оставлен как placeholder: для выполнения команды замените его фактическим адресом в локальном терминале. Успешный ответ должен содержать публичный IP-адрес VPS и HTTP 200. Для проверки обычного маршрута хоста следует выполнить запрос без параметра **-x**: он не должен автоматически переходить в туннель.

Оставшиеся эксплуатационные задачи:

1. Закрепить LAN-IP за MINI-Ubuntu в DHCP роутера либо иным способом гарантировать стабильность адреса: Docker публикует прокси на конкретном LAN-адресе.
2. Уточнить неоднозначность DNS-логов Gluetun и подтвердить фактический upstream resolver.
3. При необходимости выполнить более строгую проверку IPv6 и kill switch с наблюдением сетевых пакетов.
4. Удалить **/tmp/proxy-gateway.conf** после окончательного подтверждения, что он больше не нужен. Файл содержит приватный ключ клиента, хотя сейчас закрыт правами **0600**.
5. При обновлении Gluetun проверять release notes и выполнять контролируемое обновление. Compose закреплён за тегом **v3.41.3**; загруженный образ был идентифицирован digest, указанным выше.

---

# English version

## 1. Purpose and original problem

The goal is to provide one explicit HTTP proxy for devices on the home LAN. Selected applications—or individual features within an application—can use the proxy to reach external sites through the VPS public IP, while other traffic from the same devices continues to use the normal home-router path. This is not a network-wide VPN. It is an optional egress path selected by each proxy-aware application.

The requirement arose from inconsistent routing of external requests. The home router had forced routes for selected domains and services. As a result, components in one workflow—for example, Seerr, Radarr, Sonarr and the external APIs they call—could reach the Internet through different paths. Some endpoints or requests behind Cloudflare were intermittently blocked. Consequently, requests to add movies or series could behave inconsistently: some succeeded while others failed.

JacRed synchronization is the inverse case. A forced or tunneled route to the JacRed synchronization server could receive Cloudflare HTTP 403, while the direct route could work. The proxy must therefore not force all home-network traffic into the VPN. It is intended for specific applications that need an alternative egress route.

This design allows route selection at application or feature level:
- an application that works correctly through the normal route does not use the proxy;
- an application that needs the VPS egress configures the LAN proxy address;
- if a plugin supports separate routing for search and playback, the proxy can be used for search only while playback remains direct.

Changing the egress route does not guarantee that Cloudflare or any other remote service will accept a request; the service may block the VPS address too. The proxy makes the selected path more predictable but is not a guarantee against every external block.

## 2. Architecture

~~~text
Any home-LAN computer or application
                |
                | HTTP proxy: LAN-IP-OF-MINI-PC:8888
                v
MINI-Ubuntu (LAN-IP-OF-MINI-PC)
  Docker container: proxy-gateway
  Image: qmcgaw/gluetun:v3.41.3
  ├─ embedded HTTP proxy, TCP/8888
  ├─ WireGuard client interface: tun0
  ├─ Gluetun firewall / kill switch
  └─ DNS managed by Gluetun
                |
                | WireGuard / UDP 51820
                v
VPS (VPS-IP-ADDRESS)
  existing wg-easy server
  dedicated peer for the proxy client
                |
                | Internet egress through VPS
                v
Destination site / API
~~~

The HTTP proxy and WireGuard client run in the same Gluetun container and share its network namespace. As a result, the proxy's outbound connections are subject to Gluetun's firewall and routed through WireGuard rather than through MINI-Ubuntu's normal host route.

No separate host-level WireGuard service was installed on MINI-Ubuntu, and Ubuntu's default route was not changed. Ordinary host traffic and other applications retain their normal route through the home gateway. The WireGuard interface exists inside the container and was shown as **tun0** during inspection.

The VPS uses the existing **wg-easy** instance. A dedicated client named **MINI-Ubuntu-proxy** was created in its web UI. The server parameters, listening port, VPN subnet, NAT and existing clients were not reconfigured. No second WireGuard server was created.

## 3. Deployed implementation

### Docker and files

| Setting | Actual value |
|---|---|
| Host | **MINI-Ubuntu** |
| Host LAN address | redacted in public documentation; use LAN-IP-OF-MINI-PC |
| Project directory | **/opt/proxy-gateway** |
| Compose file | **/opt/proxy-gateway/compose.yaml** |
| Docker Compose service | **gluetun** |
| Container name | **proxy-gateway** |
| Image | **qmcgaw/gluetun:v3.41.3** |
| Pulled image digest | **sha256:fa19cc76b2af13d57a8d3dc3066f2ada061b1c761b8aecf989b3877c0486e027** |
| Working WireGuard profile | **/opt/proxy-gateway/wireguard/wg0.conf** |
| Working profile permissions | **0600**, owned by **root:root** |
| Profile mount | read-only (**:ro**) |
| Published listener | **LAN-IP-OF-MINI-PC:8888** → container TCP/8888 |
| HTTP proxy authentication | disabled; no username or password configured |
| Restart policy | **unless-stopped** |

The image was downloaded and identified by digest. The **proxy-gateway** container reached the **healthy** state. Docker reports ports 8000/tcp and 8388/tcp/udp as exposed by the image, but they are not published on the host. Only TCP/8888 is published, bound to the mini-PC's LAN address.

The following is a sanitized excerpt of the Compose configuration. Replace **LAN-IP-OF-MINI-PC** with the actual address in the local configuration; the placeholder is not intended to be copied literally into a running Compose file.

~~~yaml
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
      FIREWALL_INPUT_PORTS: "8888"
      LOG_LEVEL: "info"
    ports:
      - "LAN-IP-OF-MINI-PC:8888:8888/tcp"
~~~

Gluetun's built-in HTTP proxy is used instead of installing Tinyproxy separately. No separate Nginx, stunnel, or public HTTP proxy on the VPS is required.

### WireGuard client

A new peer was created in the existing **wg-easy** instance on the VPS. A separate working copy of the client profile is mounted into the container. No private or pre-shared key is included in this document.

Non-secret parameters:
- Endpoint: **VPS-IP-ADDRESS:51820**;
- client address: a dedicated address in the existing VPN subnet;
- working **AllowedIPs**: **0.0.0.0/0**;
- **PersistentKeepalive**: **25**;
- the IPv6 route **::/0** was removed from the working profile because IPv6 forwarding was disabled on the VPS;
- the original DNS line from the WireGuard profile was removed from the working copy; DNS is managed by Gluetun.

The downloaded original profile was temporarily stored at **/tmp/proxy-gateway.conf**. The working copy is stored in a root-only directory and mounted read-only in the container. Once it is confirmed that the original copy is no longer needed, it should be removed because it contains the client's private key.

### Gluetun in brief

Gluetun is a containerized VPN client with routing and firewall management. In this deployment it performs two related tasks: it establishes the WireGuard tunnel to the VPS and provides the built-in HTTP proxy. Both components share one network namespace, allowing Gluetun's firewall to restrict the proxy's outbound connections.

### DNS

DNS is managed by Gluetun rather than using the original WireGuard profile's DNS setting. The Gluetun startup settings summary showed an internal resolver address, a DNS forwarder and Cloudflare as the upstream resolver with **DoT (DNS over TLS)**; caching was enabled and IPv6 was disabled in the DNS settings.

**Qualification:** the same startup log also contained a message indicating plaintext DNS at a public resolver address. Therefore, the actual upstream DNS behavior should not be considered conclusively verified from those messages alone. During maintenance, check the configuration and documentation for the installed Gluetun version and confirm which upstream resolver is actually used. Do not claim that every DNS request is guaranteed to use DoT until the discrepancy is resolved.

## 4. Traffic flow

### Inbound path

A client device connects to the HTTP proxy at **LAN-IP-OF-MINI-PC**, port **8888**. Docker publishes the port only on the mini-PC's LAN IPv4 address, not on every host address. No HTTP-proxy port was opened on the VPS. Access from another computer on the home LAN was successfully tested.

This is an explicit HTTP proxy, not transparent routing. A device or application must support proxy configuration and use this address. It does not automatically intercept traffic from programs that are not configured to use it.

### Outbound path and kill switch

The proxy runs in Gluetun's network namespace, where the WireGuard interface **tun0** is active. Gluetun's firewall has an **OUTPUT DROP** policy. The inspected rules allow loopback, established connections, necessary access to the local Docker subnet, UDP to the WireGuard endpoint, and traffic through **tun0**. This is the kill switch: if the tunnel becomes unavailable, new external connections should not fall back to the normal Docker gateway.

The container's ordinary routing table still showed a default route through the Docker bridge. That does not by itself mean application traffic can freely escape directly: Gluetun manages routing and restricts it with the firewall. Both routing and kill-switch rules matter to the security of this design.

## 5. Verified results

| Test | Result |
|---|---|
| Gluetun startup | Container started; logs reported WireGuard setup, DNS readiness and the HTTP proxy listener |
| Container health | **healthy** |
| HTTP proxy | listening on **:8888** inside the container |
| HTTPS request through proxy from MINI-Ubuntu | **https://api.ipify.org** returned the VPS public IP, HTTP 200 |
| HTTPS request through proxy from another home-LAN PC | returned the VPS public IP, HTTP 200 |
| Direct request from MINI-Ubuntu without proxy | returned a different public IP; the host default route remained through the home gateway |
| Tunnel-failure test | peer **MINI-Ubuntu-proxy** was temporarily disabled in **wg-easy**; the request through the proxy timed out with HTTP 000 and did not return the home connection's IP |
| Recovery | after the peer was enabled again, the same request returned the VPS public IP, HTTP 200 |
| Nginx, host routing and other Docker services | not reconfigured as part of the proxy deployment |

The peer-disable test showed that the external request through the proxy stopped succeeding and worked again after WireGuard was restored. Together with **OUTPUT DROP**, this supports the expected fail-closed behavior for the scenario tested. It was not a full packet-capture audit; a stricter leak test can be performed with packet capture if required.

The existing **wg-easy** server remained in place. Only a dedicated peer for the proxy client was added; server settings and other peers were not changed.

## 6. Using the proxy in applications

Common proxy settings:

~~~text
Proxy type: HTTP
Host:       LAN-IP-OF-MINI-PC
Port:       8888
Username:   not set
Password:   not set
~~~

In a typical application, select HTTP proxy, set the hostname to the mini-PC's LAN address and port to **8888**, disable SSL/TLS to the proxy itself, and leave username/password empty.

### Seerr, Radarr and Sonarr

The proxy can be configured in Seerr, Radarr and Sonarr network settings when a service's own external requests are intermittently blocked or need a different egress route. Configuring a proxy in one service does not change the route used by another; each process must be configured separately.

For Seerr running in Docker, **127.0.0.1** is not the correct proxy host: inside the Seerr container it refers to that container, not the MINI-Ubuntu host. Use the mini-PC's LAN address and port **8888**. Calls from Seerr to internal Radarr/Sonarr APIs are a separate traffic path and should not be sent through the external proxy unnecessarily.

The proxy is useful when Cloudflare or another external provider treats requests differently based on source IP. It cannot guarantee availability if the VPS egress IP is blocked too.

### Kodi and Elementum

If the installed version of Elementum supports proxy settings within the plugin, search requests can be routed through this proxy while playback and stream transfer remain direct. This separation is controlled by the plugin's own settings and behavior: the generic HTTP proxy cannot identify whether a request represents search or playback. Exact Elementum settings need to be verified against the installed version; that feature-level configuration was not tested as part of deploying the proxy itself.

### JacRed

JacRed synchronization may need a direct route if requests to the synchronization service receive Cloudflare 403 through the alternative egress. Do not configure every service to use the proxy by default. The purpose is to route only selected traffic through the VPN while leaving known-good or direct-route-dependent requests on their normal path.

## 7. Encryption and security

For an HTTPS site, TLS is established between the client application and the destination site. When an HTTP proxy is used, the client sends **CONNECT**, after which the TLS stream to the site passes through the proxy without being decrypted by the proxy itself. Between MINI-Ubuntu and the VPS, that stream is additionally carried inside WireGuard. HTTPS requests therefore have two protection layers on the mini-PC-to-VPS segment: the destination site's TLS inside WireGuard.

This does **not** mean that the connection from the client device to the proxy is also encrypted. The LAN segment currently uses an ordinary HTTP proxy without TLS. For a plain-HTTP destination, there is no inner TLS layer: WireGuard protects the path to the VPS, but the connection from the VPS to the plain-HTTP site is not encrypted by HTTPS.

Proxy authentication is disabled by design because the proxy is intended for a trusted LAN. Any device or process that can reach the mini-PC's LAN address on TCP/8888 may be able to use it. Therefore:
- do not configure router port forwarding for TCP/8888 from the public Internet;
- do not treat the lack of a password as a substitute for network access restrictions;
- if access is extended beyond the trusted LAN, set **HTTPPROXY_USER** and **HTTPPROXY_PASSWORD** and separately consider TLS between clients and the proxy;
- remember that Basic authentication alone does not encrypt the username and password.

**HTTPPROXY_STEALTH=on** disables certain proxy HTTP headers from being added to requests. It is not encryption or a full access-control system.

## 8. Operations and follow-up tasks

Inspect the container:

~~~bash
sudo docker ps --filter 'name=^/proxy-gateway$'
sudo docker logs --tail 100 proxy-gateway
~~~

Inspect the Compose settings:

~~~bash
sudo cat /opt/proxy-gateway/compose.yaml
~~~

Check the egress address through the proxy from MINI-Ubuntu or another computer:

~~~bash
curl -sS --connect-timeout 5 --max-time 20 \
  -x http://LAN-IP-OF-MINI-PC:8888 \
  https://api.ipify.org \
  -w '\nHTTP=%{http_code}\n'
~~~

In the public document, the mini-PC's LAN address is a placeholder. Replace it with the actual address when running the command locally. A successful response should contain the VPS public IP and HTTP 200. To verify the host's normal route, run a request without the **-x** parameter; it should not automatically be redirected through the tunnel.

Remaining operational tasks:

1. Reserve the mini-PC's LAN address in the router's DHCP settings or otherwise guarantee it remains stable: Docker publishes the proxy on a specific LAN address.
2. Resolve the ambiguity in Gluetun's DNS logs and confirm the actual upstream resolver.
3. If a stricter security audit is required, verify IPv6 behavior and the kill switch while observing packets.
4. Remove **/tmp/proxy-gateway.conf** after confirming it is no longer needed. It contains the client's private key, even though its current permissions are **0600**.
5. Review release notes before updating Gluetun and perform updates in a controlled manner. Compose is pinned to **v3.41.3**; the downloaded image was identified by the digest listed above.
