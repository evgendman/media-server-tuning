# Apprise API на MINI-Ubuntu

**Статус:** контейнер запущен 2026-10-21; `GET /` и `/status` ответили HTTP 200, состояние — `running`, `RestartCount=0`. Настройка исходящего HTTP-прокси и проверка доставки Telegram ещё ожидают выполнения.  
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
| Адрес публикации на хосте | `127.0.0.1:8000` |
| Persistent config | `/opt/apprise-api/config` |
| Вложения | `/opt/apprise-api/attach` |
| Дополнительные плагины | `/opt/apprise-api/plugin` |
| Stateful mode | `simple` — конфигурации сохраняются на диске в `/config` |
| Workers | `1` — достаточно для небольшого домашнего сервера |
| UID:GID процесса | `1000:1000` — соответствует основному пользователю MINI-Ubuntu |
| Автоперезапуск | `unless-stopped` |
| Часовой пояс контейнера | `Asia/Yekaterinburg` — часовой пояс MINI-Ubuntu на предварительной проверке 2026-10-21 |
| Proxy исходящих HTTP(S)-запросов | `http://<MINI-LAN-IP>:8888` — адрес показан как заполнитель, поскольку LAN-адреса не публикуются |
| Proxy environment | `HTTP_PROXY`, `HTTPS_PROXY`, `http_proxy`, `https_proxy`; bypass для loopback через `NO_PROXY` / `no_proxy` |

### Граница доступа

Порт намеренно публикуется только на loopback-интерфейсе хоста. Так API и веб-интерфейс не становятся доступными всей локальной сети. Для начального локального развёртывания встроенная HTTP-аутентификация не включается; любой локальный процесс с доступом к `127.0.0.1:8000` сможет использовать API. Доступ к веб-интерфейсу с другого компьютера через SSH-туннель следует разрешать только доверенному SSH-пользователю.

Для браузера на другом компьютере можно использовать SSH-туннель. Если впоследствии понадобится прямой доступ из Docker-контейнеров или других узлов, способ подключения нужно отдельно спроектировать и защитить; не следует просто менять привязку на `0.0.0.0:8000`.

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
      - "127.0.0.1:8000:8000"
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

Перед запуском замените все шесть вхождений `http://<MINI-LAN-IP>:8888` локальным адресом действующего HTTP-прокси (в конфигурации MINI-Ubuntu — адрес самого мини-ПК и TCP/8888). Заполнитель `<MINI-LAN-IP>` предназначен только для этого публичного документа и не является готовым рабочим адресом. Используется `TZ: Asia/Yekaterinburg`, соответствующий часовому поясу MINI-Ubuntu. Установить ожидаемые права:

```bash
sudo chown 1000:1000 /opt/apprise-api/docker-compose.yml
sudo chmod 0640 /opt/apprise-api/docker-compose.yml
```

Использовать основной файл Compose из каталога приложения. Не клонировать репозиторий разработчика для обычного production-развёртывания и не подключать его `docker-compose.override.yml`: override предназначен для разработки.

## 4. Загрузка и запуск

```bash
sudo docker compose -f /opt/apprise-api/docker-compose.yml config
sudo docker compose -f /opt/apprise-api/docker-compose.yml pull
sudo docker compose -f /opt/apprise-api/docker-compose.yml up -d
```

Проверить фактическое состояние и API health:

```bash
sudo docker ps --filter name=apprise
sudo docker inspect -f 'Image={{.Config.Image}} ImageID={{.Image}} Status={{.State.Status}} RestartCount={{.RestartCount}}' apprise
sudo docker logs --tail 80 apprise
curl -sS -o /dev/null -w 'GET / -> HTTP %{http_code}\n' http://127.0.0.1:8000/
curl -sS --max-time 15 -w '\nHTTP %{http_code}\n' http://127.0.0.1:8000/status
```

После изменения переменных proxy Compose должен пересоздать контейнер, чтобы они появились в окружении нового экземпляра. Проверить их внутри контейнера без вывода секретов:

```bash
sudo docker exec apprise python -c 'import os,urllib.request; print("HTTPS_PROXY=" + os.environ.get("HTTPS_PROXY", "<unset>")); print("proxies=" + repr(urllib.request.getproxies()))'
```
При проверке зафиксировать фактическую версию образа/digest, состояние контейнера, HTTP-код и отсутствие ошибок инициализации. Успешный статус не отмечать до выполнения этих проверок.

## 5. Открытие веб-интерфейса

На самом MINI-Ubuntu открыть:

`http://127.0.0.1:8000/`

Если браузер запускается на другом компьютере, открыть SSH-туннель на этом компьютере (подставить фактический LAN-адрес мини-ПК):

```bash
ssh -N -L 8000:127.0.0.1:8000 dmitriev-ev@<MINI-LAN-IP>
```

Затем открыть на компьютере-клиенте `http://127.0.0.1:8000/`. Окно терминала с туннелем должно оставаться открытым.

В веб-интерфейсе создать конфигурацию с идентификатором, например, `arr-stack`, и добавить Telegram-адрес в формате, который определён в документации Apprise:

`tgram://<BOT_TOKEN>/<CHAT_ID>/`

Вместо заполнителей использовать собственные значения. Не публиковать этот URL в репозитории: он содержит токен бота. Убедиться, что бот создан через BotFather и пользователь ранее открыл с ним диалог/нажал Start.

## 6. Тестовое уведомление

После того как конфигурация `arr-stack` сохранена и проверена в UI:

```bash
curl -fsS -X POST \
  -H 'Content-Type: application/json' \
  -d '{"title":"MINI-Ubuntu: Apprise API","body":"Тестовая проверка доставки уведомлений.","type":"success"}' \
  http://127.0.0.1:8000/notify/arr-stack
```

Проверить сообщение в Telegram и логи:

```bash
sudo docker logs --tail 80 apprise
```

API-путь `/notify/{KEY}` принимает `body`, необязательный `title`, тип `info`, `success`, `warning` или `failure`, а также параметры адресации вроде `tag`. После первичной проверки нужно согласовать единые идентификаторы и теги для адаптеров, импортёров и systemd-задач.

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
- Следующий этап: добавить env-переменные существующего HTTP-прокси и проверить HTTPS-запрос из контейнера через него; затем настроить Telegram и подтвердить тестовую доставку. На момент последней проверки исходящие уведомления ещё не тестировались.
