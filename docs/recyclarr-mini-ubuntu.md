# Recyclarr на MINI-Ubuntu

**Последнее подтверждённое состояние: 2026-11-06.** На эту дату на MINI-Ubuntu был установлен Recyclarr **v8.8.1**. В документе разделены установленный исполняемый файл, рабочие материалы проекта и runtime-конфигурация. Секретные значения API-ключей намеренно не приводятся.

## 1. Установленный Recyclarr

| Назначение | Путь / значение |
|---|---|
| Версия | `8.8.1` |
| Исполняемый файл | `/var/tmp/arr-quality-policy/bin/recyclarr` |
| Каталог runtime-конфигурации | `/var/tmp/arr-quality-policy-runtime/config` |
| Каталог runtime-данных | `/var/tmp/arr-quality-policy-runtime/data` |

Recyclarr запускается локальным бинарным файлом. Для этой установки используются переменные окружения:

```bash
export RECYCLARR_CONFIG_DIR=/var/tmp/arr-quality-policy-runtime/config
export RECYCLARR_DATA_DIR=/var/tmp/arr-quality-policy-runtime/data
```

`RECYCLARR_CONFIG_DIR` и `RECYCLARR_DATA_DIR` — актуальные для установленной версии переменные. Не использовать устаревшее имя `RECYCLARR_APP_DATA_DIR`.

## 2. Runtime-конфигурация на сервере

В `/var/tmp/arr-quality-policy-runtime/config/` находится рабочая конфигурация Recyclarr, используемая при Sync:

| Файл | Назначение |
|---|---|
| `radarr.yml` | Экземпляр Radarr 1080p, адрес API, ссылка на secret и назначения Custom Formats |
| `sonarr.yml` | Экземпляр Sonarr 1080p, адрес API, ссылка на secret и назначения Custom Formats |
| `settings.yml` | Общие настройки Recyclarr и локальные Resource Providers |
| `secrets.yml` | Реальные API-ключи Arr, подключаемые через `!secret` |

`secrets.yml` — только локальный файл сервера; на дату проверки он имел права `600`. Его содержимое нельзя копировать в Git, отчёты, команды или публичные документы. Для восстановления имён секретов есть безопасный пример `recyclarr/config/secrets.yml.example` в репозитории `arr-quality-policy`.

## 3. Рабочие материалы на MINI-Ubuntu

Рабочий каталог проекта: `/var/tmp/arr-quality-policy/`.

### Исходная политика и снимок

| Файл | Назначение |
|---|---|
| `master/universal-policy.yml` | Универсальная политика отбора релизов для Radarr и Sonarr |
| `master/radarr-1080p-live-snapshot.json` | Снимок конфигурации Radarr 1080p, использованный при подготовке каталога |

Снимок Radarr — не полный экспорт дерева Quality Profile: актуальные группы, вложенные качества и порядок профиля позднее сохранены отдельно в `arr-quality-policy/recyclarr/profiles/`.

### Сгенерированные ресурсы и YAML

| Каталог / файл | Состав |
|---|---|
| `generated/radarr/custom-formats/` | 56 отдельных JSON Custom Formats Radarr |
| `generated/radarr/recyclarr.yml` | YAML Recyclarr для Radarr |
| `generated/radarr/settings.yml` | Пример/фрагмент настроек providers для Radarr |
| `generated/radarr/secrets.yml.example` | Пример имён секретов без реальных ключей |
| `generated/sonarr/custom-formats/` | 55 отдельных JSON Custom Formats Sonarr |
| `generated/sonarr/recyclarr.yml` | YAML Recyclarr для Sonarr |
| `generated/sonarr/settings.yml` | Пример/фрагмент настроек providers для Sonarr |
| `generated/sonarr/secrets.yml.example` | Пример имён секретов без реальных ключей |

Именно каталоги `custom-formats/` содержат отдельные JSON-определения, которые подключаются к Recyclarr как локальные Resource Providers. YAML назначает оценки этим форматам в конкретном Quality Profile. JSON-каталоги Radarr и Sonarr имеют разные идентификаторы и не должны смешиваться.

### Отчёты

В `reports/` на момент проверки находились, в частности:
- `policy-audit-2026-11-06.md` — аудит универсальной политики;
- `sonarr-preview-2026-11-06.md` — результат Preview для Sonarr;
- `sonarr-sync-2026-11-06.md` — отчёт фактической синхронизации Sonarr.

## Резервные копии Arr

Штатные резервные копии Radarr и Sonarr хранятся только локально на медиа-сервере. Они не являются частью этого репозитория и не должны загружаться в GitHub.

Такие архивы могут содержать API-ключи и учётные данные интеграций в `config.xml` или базе данных. Считать их конфиденциальными; не публиковать, не прикладывать к отчётам и не копировать в общедоступные сетевые каталоги.

В Git разрешено хранить только документацию, обезличенные отчёты и безопасные шаблоны конфигурации без реальных секретов.
## 4. Материалы в GitHub

Публичный репозиторий с источником политики, JSON-каталогами и Recyclarr YAML:

[`evgendman/arr-quality-policy`](https://github.com/evgendman/arr-quality-policy)

Основные пути в репозитории:

| Путь | Назначение |
|---|---|
| `docs/policy-model.md` | Универсальная модель Quality Profile и Custom Formats |
| `docs/custom-formats/` | Описание Voice, Src, Format и Extra |
| `docs/implementations/radarr.md` | Особенности каталога Radarr |
| `docs/implementations/sonarr.md` | Особенности каталога Sonarr |
| `recyclarr/catalogs/radarr-custom-formats-json.tar.gz` | Архив JSON-каталога Radarr (56 форматов) |
| `recyclarr/catalogs/sonarr-custom-formats-json.tar.gz` | Архив JSON-каталога Sonarr (55 форматов) |
| `recyclarr/config/radarr.yml` | Конфигурация экземпляра Radarr 1080p и оценки Custom Formats |
| `recyclarr/config/sonarr.yml` | Конфигурация экземпляра Sonarr 1080p и оценки Custom Formats |
| `recyclarr/config/settings.yml` | Фрагмент локальных Resource Providers |
| `recyclarr/config/secrets.yml.example` | Безопасный образец имён секретов |
| `recyclarr/profiles/radarr-1080p-ru.yml` | Include-шаблон Quality Profile Radarr 1080p RU |
| `recyclarr/profiles/sonarr-1080p-ru.yml` | Include-шаблон Quality Profile Sonarr 1080p RU |

В `arr-quality-policy` также документированы правила повторного использования профилей для будущих 4K-экземпляров. Include-шаблоны профилей пока не подключены к активным экземплярам.

## 5. Текущие экземпляры Arr и охват Recyclarr

На MINI-Ubuntu работают четыре отдельных экземпляра Arr:

| Сервис | Порт | Назначение |
|---|---:|---|
| Radarr 1080p | `7878` | Обычная библиотека фильмов |
| Radarr 4K | `7879` | Библиотека фильмов 2160p |
| Sonarr 1080p | `8989` | Обычная библиотека сериалов |
| Sonarr 4K | `8988` | Библиотека сериалов 2160p |

На дату последней проверки runtime-конфигурация Recyclarr в этом документе описана для экземпляров 1080p. Наличие 4K-экземпляров Arr не означает, что они уже подключены к Recyclarr: для каждого 4K-экземпляра по-прежнему нужна отдельная проверенная конфигурация Recyclarr, собственный API secret и назначения Custom Formats на соответствующий 4K Quality Profile.

**Seerr:** все четыре экземпляра должны быть отмечены серверами по умолчанию — отдельно обычный и 4K Radarr, а также отдельно обычный и 4K Sonarr. Seerr поддерживает по одному Default-серверу для обычного режима и по одному Default 4K-серверу для каждого типа приложения. Эти назначения независимы. Подробная таблица, логика маршрутизации и проверка приведены в [документе Seerr](seerr-arr-routing.md).

Важно: отметка Default выбирает целевой экземпляр для заявки соответствующего режима, но не дублирует одну заявку одновременно в обычный и 4K Arr. Для получения обоих вариантов создаются две отдельные заявки — обычная и 4K.

## 6. Безопасный порядок обслуживания

1. Проверить версию бинарного файла и доступность нужного экземпляра Arr.
2. Перед любым Sync сделать штатную резервную копию соответствующего Arr.
3. Проверить YAML и локальные пути Resource Providers.
4. Запустить Preview только для конкретного сервиса/экземпляра и изучить изменения профиля, форматов и удаления.
5. Выполнять Sync только после проверки Preview.
6. После Sync проверить дерево Quality Profile, cutoff, баллы Custom Formats и отсутствие нежелательных удалений.

Обычный Sync не следует запускать для всех сервисов сразу, если в общей конфигурации присутствуют несколько экземпляров. Указывать конкретный сервис и при необходимости `--instance`.

## Примечание о достоверности

Пути и версия в документе отражают последнее подтверждённое состояние MINI-Ubuntu от 2026-11-06. При следующем подключении к серверу следует сверить их с фактической файловой системой, особенно если временные каталоги `/var/tmp/` были очищены или рабочий каталог переносился.
