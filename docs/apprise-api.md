# Apprise API на MINI-Ubuntu

**Статус на 2026-10-21:** контейнер запущен и проверен; `GET /` и `/status` ответили HTTP 200. Исходящий HTTP(S)-прокси настроен, запрос к Telegram API получил HTTP 200, а тестовые сообщения успешно доставлены в четыре Telegram-темы. Настройка Telegram, теги и результаты тестов вынесены в отдельный документ: [Apprise API: Telegram и маршрутизация уведомлений](apprise-notifications.md).  
**Проект разработчика:** https://github.com/caronc/apprise-api  
**Способ установки:** официальный опубликованный Docker-образ `caronc/apprise:latest` с Docker Compose; сборка из исходников не требуется.  
**Назначение:** единый локальный HTTP API для доставки уведомлений адаптеров, импортёров и заданий systemd в Telegram и другие поддерживаемые Apprise каналы.

## 1. Выбранные настройки

| Параметр | Значение |
|---|---|
| Каталог развёртывания | `/opt/apprise-api` |
| Compose-файл | `/opt/apprise-api/docker-compose.yml` |
| Образ | `caronc/apprise:latest` |
| Контейнер | `apprise` |
| HTTP-порт контейнера | `8000` |
| Адрес публикации на хосте | `<MINI-LAN-IP>:8000` — привязка только к LAN-интерфейсу, не ко всем интерфейсам |
| Persistent config | `/opt/apprise-api/config` |
| Вложения | `/opt/apprise-api/attach` |
| Дополнительные плагины | `/opt/apprise-api/plugin` |
| Stateful mode | `simple` — конфигурации сохраняются на диске в `/config` |
| Workers | `1` — достаточно для небольшого домашнего сервера |
| UID:GID процесса | `1000:1000` — соответствует основному пользователю MINI-Ubuntu |
| Автоперезапуск | `unless-stopped` |
| Часовой пояс контейнера | `Asia/Yekaterinburg` — часовой пояс MINI-Ubuntu на предварительной проверке 2026-10-21 |
| Proxy исходящих HTTP(S)-запросов | Настроен через `http://<MINI-LAN-IP>:8888`; публичный документ скрывает конкретный LAN-адрес |
| Proxy environment | `HTTP_PROXY`, `HTTPS_PROXY`, `http_proxy`, `https_proxy`; bypass для loopback через `NO_PROXY` / `no_proxy` |

### Граница доступа

Веб-интерфейс и API публикуются на LAN-адресе самого MINI-Ubuntu (`<MINI-LAN-IP>:8000`), а не на `0.0.0.0:8000`: Docker слушает только выбранный интерфейс. Это позволяет домашним устройствам открывать `http://<MINI-LAN-IP>:8000/`, не публикуя порт на всех интерфейсах хоста.

На данном этапе встроенная HTTP-аутентификация Apprise API намеренно отключена. Граница доступа — доверенная домашняя сеть: любой узел, который может подключиться к `192.168.0.14:8000`, сможет обращаться к API, включая отправку сообщений и управление конфигурациями. Поэтому не настраивать перенаправление TCP/8000 на роутере из Интернета и не считать LAN-привязку полноценной аутентификацией.

Токен Telegram идентифицирует бота при отправке сообщений в Telegram, но не защищает входящий Apprise API. При необходимости защиты API позже включить встроенную аутентификацию либо ограничить доступ межсетевым экраном по источникам. Если понадобится доступ вне доверенной сети, сначала спроектировать VPN или HTTPS с подходящими сетевыми ограничениями.

Токен Telegram и другие секреты нельзя добавлять в Git. Их следует вводить в веб-интерфейсе Apprise API на хосте; конфигурационный каталог содержит секретные данные и должен оставаться локальным.

### Исходящий трафик через домашний HTTP-прокси

Apprise должен отправлять внешние уведомления через уже существующий HTTP-прокси MINI-Ubuntu, иначе прямой выход к Telegram может быть заблокирован провайдером. Фактический адрес прокси хранится в локальном Compose-файле на хосте; в этом публичном репозитории он обозначен как `http://<MINI-LAN-IP>:8888`.

Контейнеру задаются стандартные переменные `HTTP_PROXY` и `HTTPS_PROXY`, а также их варианты в нижнем регистре для совместимости с разными HTTP-клиентами. Значения для HTTPS используют схему `http://`, поскольку соединение TLS к целевому сайту создаётся через HTTP proxy. `NO_PROXY`/`no_proxy` исключают loopback-адреса. Python Requests поддерживает эти стандартные переменные окружения ([документация](https://requests.readthedocs.io/en/stable/user/advanced/#proxies)). Конкретный плагин уведомлений всё равно нужно проверить фактической отправкой тестового сообщения.

Если адрес/порт существующего прокси изменится, обновить переменные в локальном Compose-файле и пересоздать контейнер. Не публиковать конкретный LAN-адрес или токены в этом репозитории.

## 2. Предварительная проверка хоста

Перед изменениями проверить, что Docker Compose доступен, порт 8000 свободен и каталог назначения не содержит прежнюю установку:

```bash
docker --version
docker compose version
timedatectl show -p Timezone --value
sudo ss -ltnp 'sport = :8000'
sudo ls -ld /opt/apprise-api 2>/dev/null || true
```

Если каталог `/opt/apprise-api` уже существует, не перезаписывать его вслепую: сначала изучить содержимое и сохранить резервную копию.

## 3. Создание каталогов и Compose-файла

После проверки (если это новая установка) создать каталоги с доступом UID/GID 1000:

```bash
sudo install -d -o 1000 -g 1000 -m 0750 \
  /opt/apprise-api \
  /opt/apprise-api/config \
  /opt/apprise-api/plugin \
  /opt/apprise-api/attach
```

Создать `/opt/apprise-api/docker-compose.yml`:

```yaml
services:
  apprise:
    image: caronc/apprise:latest
    container_name: apprise
    restart: unless-stopped
    user: "1000:1000"
    ports:
      - "<MINI-LAN-IP>:8000:8000"
    environment:
      APPRISE_STATEFUL_MODE: simple
      APPRISE_WORKER_COUNT: "1"
      APPRISE_DEFAULT_FORMAT: "text"
      APPRISE_ADMIN: "yes"
      TZ: Asia/Yekaterinburg
      HTTP_PROXY: "http://<MINI-LAN-IP>:8888"
      HTTPS_PROXY: "http://<MINI-LAN-IP>:8888"
      http_proxy: "http://<MINI-LAN-IP>:8888"
      https_proxy: "http://<MINI-LAN-IP>:8888"
      NO_PROXY: "localhost,127.0.0.1,::1"
      no_proxy: "localhost,127.0.0.1,::1"
    volumes:
      - ./config:/config
      - ./plugin:/plugin
      - ./attach:/attach
```

При изменении системного часового пояса обновить параметр `TZ` и пересоздать контейнер.

Перед запуском замените `<MINI-LAN-IP>` в привязке порта фактическим LAN-адресом MINI-Ubuntu. На дату развёртывания это `192.168.0.14`; конкретный адрес можно заменить заполнителем при публикации документа. Также замените четыре значения `http://<MINI-LAN-IP>:8888` адресом существующего HTTP-прокси (TCP/8888). В рабочем Compose-файле на MINI-Ubuntu используется фактический LAN-адрес для порта и прокси. Файл `.env` для Apprise на данном этапе не требуется.

```bash
sudo chown 1000:1000 /opt/apprise-api/docker-compose.yml
sudo chmod 0640 /opt/apprise-api/docker-compose.yml
```

Использовать основной файл Compose из каталога приложения. Не клонировать репозиторий разработчика для обычного production-развёртывания и не подключать его `docker-compose.override.yml`: override предназначен для разработки.

## 4. Загрузка и запуск

```bash
sudo docker compose -f /opt/apprise-api/docker-compose.yml config >/dev/null
sudo docker compose -f /opt/apprise-api/docker-compose.yml pull
sudo docker compose -f /opt/apprise-api/docker-compose.yml up -d
```



Проверить фактическое состояние и API health:

```bash
sudo docker ps --filter name=apprise
sudo docker inspect -f 'Image={{.Config.Image}} ImageID={{.Image}} Status={{.State.Status}} RestartCount={{.RestartCount}}' apprise
sudo docker logs --tail 80 apprise
curl -sS -o /dev/null -w 'GET / -> HTTP %{http_code}\n' http://<MINI-LAN-IP>:8000/
curl -sS --max-time 15 -w '\nHTTP %{http_code}\n' http://<MINI-LAN-IP>:8000/status
```

После изменения переменных proxy Compose должен пересоздать контейнер, чтобы они появились в окружении нового экземпляра. Проверить их внутри контейнера без вывода секретов:

```bash
sudo docker exec apprise python -c 'import os,urllib.request; print("HTTPS_PROXY=" + os.environ.get("HTTPS_PROXY", "<unset>")); print("proxies=" + repr(urllib.request.getproxies()))'
```
При проверке зафиксировать фактическую версию образа/digest, состояние контейнера, HTTP-код и отсутствие ошибок инициализации. Успешный статус не отмечать до выполнения этих проверок.

## 5. Открытие веб-интерфейса

С компьютера в домашней LAN открыть в браузере:

`http://<MINI-LAN-IP>:8000/`

На MINI-Ubuntu фактический LAN-адрес на дату настройки — `192.168.0.14`; адрес скрыт в публичной документации. На этом этапе логин и пароль не запрашиваются. Не использовать прежнюю инструкцию с `127.0.0.1:18000`: SSH-туннель не нужен при прямом доступе в LAN.

В веб-интерфейсе создана конфигурация `arr-stack`. Telegram-адреса, теги, ID тем и правила отправки документированы отдельно в [Apprise API: Telegram и маршрутизация уведомлений](apprise-notifications.md). Не публиковать Telegram URL в репозитории: он содержит токен бота. Секреты хранятся только в локальной конфигурации Apprise.

## 6. Тестовое уведомление

После того как конфигурация `arr-stack` сохранена и проверена в UI:

```bash
curl -fsS -X POST \
  -H 'Content-Type: application/json' \
  -d '{"title":"MINI-Ubuntu: Apprise API","body":"Тестовая проверка доставки уведомлений.","type":"success"}' \
  http://<MINI-LAN-IP>:8000/notify/arr-stack
```

Проверить сообщение в Telegram и логи:

```bash
sudo docker logs --tail 80 apprise
```

Первичная доставка подтверждена. Для отправителей API `/notify/{KEY}` принимает `body`, необязательный `title`, тип `info`, `success`, `warning` или `failure`, а также `tag` для выбора назначения. Согласованные теги и примеры запросов приведены в [документе о маршрутизации уведомлений](apprise-notifications.md).

## 7. Обновление и управление

```bash
# Состояние
sudo docker compose -f /opt/apprise-api/docker-compose.yml ps

# Логи
sudo docker logs --tail 100 apprise

# Обновить официальный образ и пересоздать контейнер
sudo docker compose -f /opt/apprise-api/docker-compose.yml pull
sudo docker compose -f /opt/apprise-api/docker-compose.yml up -d

# Остановить, не удаляя настройки
sudo docker compose -f /opt/apprise-api/docker-compose.yml down
```

Каталоги `config`, `plugin` и `attach` являются постоянными данными на хосте. Не удалять их при обычном обновлении/пересоздании контейнера.

## 8. Ссылки разработчика

- Репозиторий и инструкции по установке: https://github.com/caronc/apprise-api
- Официальный файл Docker Compose: https://github.com/caronc/apprise-api/blob/master/docker-compose.yml
- Документация Telegram-плагина: https://appriseit.com/services/telegram/
- Веб-документация Apprise API: https://appriseit.com/getting-started/

## История развёртывания

- 2026-10-21: предварительная проверка MINI-Ubuntu подтвердила Ubuntu 26.04.1 LTS, `Asia/Yekaterinburg`, Docker Engine `29.8.1`, Docker Compose `v5.5.1`; TCP/8000 был свободен, `/opt/apprise-api` отсутствовал.
- 2026-10-21: официальный образ `caronc/apprise:latest` загружен и запущен. Проверено: `GET /` → HTTP 200; `GET /status` → тело `OK`, HTTP 200; контейнер `running`, `RestartCount=0`; ImageID `sha256:0b2913b15c8eb47c25345722985ffbfae701326edde21909c205d296d62dd52c`. В журнале есть успешный запуск Gunicorn, Nginx и prune-loop.
- 2026-10-21: в Compose добавлены `HTTP_PROXY`, `HTTPS_PROXY` и их варианты в нижнем регистре, указывающие на существующий прокси MINI-Ubuntu по TCP/8888; для loopback задан `NO_PROXY`. Проверено внутри контейнера: окружение подхвачено, HTTPS-запрос к `https://api.telegram.org/` получил HTTP 200; `/status` после пересоздания контейнера вернул `OK`, HTTP 200. Конкретный LAN-адрес прокси не публикуется.
- 2026-10-21: выбран доступ без встроенной HTTP-аутентификации на этапе первоначальной настройки; граница доступа — доверенная домашняя LAN. Фактическая публикация сервиса и доступ через `http://192.168.0.14:8000/` подтверждены пользователем. Не публиковать TCP/8000 в Интернет.
- 2026-10-21: HTTP-прокси настроен в контейнере; проверка HTTPS к Telegram API прошла с HTTP 200.
- 2026-10-21: конфигурация `arr-stack` создана; тестовые уведомления доставлены в темы `critical`, `warnings`, `run-info` и `library`. Подробности и ID трёх новых тем приведены в [apprise-notifications.md](apprise-notifications.md). Подключение Seerr к теме `library` остаётся отдельным следующим шагом.
