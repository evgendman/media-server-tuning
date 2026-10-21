# Apprise API на MINI-Ubuntu

**Статус:** процедура и целевая конфигурация подготовлены; выполнение на хосте и успешный запуск ещё не подтверждены. Не считать этот документ свидетельством уже установленного сервиса.  
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
| Часовой пояс контейнера | Изначально `Etc/UTC`; перед запуском предпочтительно заменить значением из `timedatectl show -p Timezone --value` на хосте |

### Граница доступа

Порт намеренно публикуется только на loopback-интерфейсе хоста. Так API и веб-интерфейс не становятся доступными всей локальной сети без дополнительной защиты. На начальном этапе предполагаются отправители, работающие непосредственно на MINI-Ubuntu.

Для браузера на другом компьютере можно использовать SSH-туннель. Если впоследствии понадобится прямой доступ из Docker-контейнеров или других узлов, способ подключения нужно отдельно спроектировать и защитить; не следует просто менять привязку на `0.0.0.0:8000`.

Токен Telegram и другие секреты нельзя добавлять в Git. Их следует вводить в веб-интерфейсе Apprise API на хосте; конфигурационный каталог содержит секретные данные и должен оставаться локальным.

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
      APPRISE_ADMIN: "y"
      TZ: Etc/UTC
    volumes:
      - ./config:/config
      - ./plugin:/plugin
      - ./attach:/attach
```

Перед запуском, если системный часовой пояс MINI-Ubuntu отличается от UTC, заменить `TZ: Etc/UTC` в этом файле на значение, полученное командой `timedatectl show -p Timezone --value`.

Файл можно создать редактором или безопасно записать через heredoc. Установить ожидаемые права:

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

Проверить фактическое состояние:

```bash
sudo docker ps --filter name=apprise
sudo docker inspect -f 'Image={{.Config.Image}} Status={{.State.Status}} StartedAt={{.State.StartedAt}}' apprise
sudo docker logs --tail 80 apprise
curl -sS -o /dev/null -w 'HTTP %{http_code}\n' http://127.0.0.1:8000/
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

- Процедура и безопасная исходная конфигурация подготовлены 2026-10-21.
- На момент подготовки документа выполнение команд на MINI-Ubuntu и успешная доставка тестового сообщения ещё не подтверждены. После реального развёртывания дополнить этот раздел фактическими данными: образом/digest, датой проверки, HTTP-кодом и результатом теста Telegram.
