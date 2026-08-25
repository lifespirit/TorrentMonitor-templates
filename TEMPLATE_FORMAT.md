# Формат шаблона TorrentMonitor

Файл имеет формат YAML или JSON. Текущая версия схемы — `1`.

## Минимальный каркас

```yaml
schema_version: 1
id: example.org
name: Example Tracker
kind: forum_topic
mode: http

domains:
  - example.org

defaults:
  access_mode: native
  timezone: Europe/Moscow
  page_timeout_seconds: 30

encoding:
  page: utf-8
  target: utf-8
  form: utf-8

urls:
  base: https://example.org
  login: https://example.org/login.php
  auth_check: https://example.org/index.php
  topic: https://example.org/viewtopic.php?t={{ item.torrent_id }}
  download: https://example.org/dl.php?t={{ item.torrent_id }}

http:
  proxy: inherit

auth:
  type: form
  check:
    method: GET
    url: https://example.org/index.php
    success:
      contains: logout.php
  login_form:
    method: POST
    url: https://example.org/login.php
    form:
      username: "{{ credentials.login }}"
      password: "{{ credentials.password }}"

topic:
  ready:
    url:
      contains: viewtopic.php?t={{ item.torrent_id }}
    dom_contains:
      - dl.php?t={{ item.torrent_id }}
  title:
    selector: title
  updated_at:
    selector: body
    regex: "([0-9]{2}\.[0-9]{2}\.[0-9]{4} [0-9]{2}:[0-9]{2})"
    layout: "02.01.2006 15:04"
    timezone: Europe/Moscow
  closed:
    contains_any:
      - Тема закрыта

download:
  url: https://example.org/dl.php?t={{ item.torrent_id }}
  validate:
    bencode_torrent: true
    max_size_mb: 5
    reject_if_starts_with:
      - "<!DOCTYPE html"
      - "<html"
```

## Верхний уровень

| Поле | Обязательное | Описание |
| --- | --- | --- |
| `schema_version` | да | Версия схемы, сейчас `1`. |
| `id` | да | Стабильный идентификатор, обычно основной домен. По нему внешний шаблон переопределяет встроенный. |
| `name` | желательно | Читаемое название трекера. |
| `kind` | да | Для форумных раздач используйте `forum_topic`. |
| `mode` | да | Сейчас движок шаблона — `http`. Это не выбор транспорта: Native/Chromium задаётся в `defaults.access_mode`. |
| `domains` | да | Все домены, по которым registry должен находить шаблон. |

## `defaults`

| Поле | Значения | Описание |
| --- | --- | --- |
| `access_mode` | `native`, `chromium` | Рекомендуемый режим для новой записи учётных данных. |
| `timezone` | IANA timezone | Часовой пояс дат сайта, например `Europe/Moscow`. |
| `page_timeout_seconds` | целое число | Время ожидания страницы, особенно важно для Chromium. |

Выбирайте `native`, если авторизация и загрузка работают обычными HTTP-запросами. `chromium` нужен для JavaScript, Cloudflare или CAPTCHA.

## `encoding`

| Поле | Описание |
| --- | --- |
| `response` | Кодировка HTTP-ответов общего назначения. |
| `page` | Кодировка HTML страницы темы. |
| `target` | Целевая кодировка после преобразования, обычно `utf-8`. |
| `form` | Кодировка полей формы авторизации. |

Для современных сайтов используйте `utf-8`. Для старых phpBB-трекеров может потребоваться `windows-1251`.

## `urls`

| Поле | Описание |
| --- | --- |
| `base` | Базовый URL для разрешения относительных ссылок. |
| `login` | Страница входа, также используется как цель интерактивной Chromium-сессии. |
| `auth_check` | Страница проверки существующей сессии. |
| `topic` | Страница темы. |
| `download` | URL загрузки `.torrent`. |

## Переменные

В строках URL, заголовках, cookies и формах можно использовать:

| Переменная | Значение |
| --- | --- |
| `{{ item.tracker }}` | идентификатор трекера |
| `{{ item.name }}` | текущее название темы |
| `{{ item.url }}` | исходный URL темы |
| `{{ item.torrent_id }}` | ID темы/раздачи |
| `{{ credentials.login }}` | логин |
| `{{ credentials.password }}` | пароль |
| `{{ credentials.passkey }}` | passkey |
| `{{ credentials.cookie }}` | сохранённый cookie header |

Не помещайте реальные значения credentials в репозиторий.

## HTTP-запрос

Одинаковая структура используется в `auth.check`, `auth.login_form` и `download.request`:

| Поле | Описание |
| --- | --- |
| `method` | `GET` или `POST`; пустое значение означает `GET`. |
| `url` | URL с шаблонными переменными. |
| `form` | Map полей POST-формы. |
| `form_encoding` | Кодировка формы, если отличается от `encoding.form`. |
| `headers` | Дополнительные HTTP-заголовки. |
| `cookies` | Cookies запроса как map `имя: значение`. |
| `success` | Правила успешного ответа. |

Глобальный User-Agent для Native HTTP добавляет приложение. Явный заголовок `User-Agent` в `headers` имеет приоритет.

## Правила совпадения

В `success`, `closed` и других проверках доступны:

| Поле | Условие |
| --- | --- |
| `contains` | присутствует одна строка |
| `contains_all` | присутствуют все строки |
| `contains_any` | присутствует хотя бы одна строка |
| `regex` | совпадает регулярное выражение Go/RE2 |

Используйте устойчивые признаки: ссылку выхода, имя профиля, ссылку скачивания. Не используйте слишком общие слова вроде `forum` или HTTP 200 как единственный признак.

## `auth`

```yaml
auth:
  type: form
  check:
    method: GET
    url: https://example.org/index.php
    success:
      contains: logout.php
  login_form:
    method: POST
    url: https://example.org/login.php
    form:
      username: "{{ credentials.login }}"
      password: "{{ credentials.password }}"
```

В Native HTTP сначала проверяется сохранённая cookie-сессия, затем при необходимости отправляется `login_form`, после чего `check` выполняется повторно. В Chromium пользователь проходит вход и CAPTCHA вручную, а `check.success` подтверждает авторизацию профиля.

## `topic.ready`

Описывает положительные признаки того, что Chromium/HTTP получил именно ожидаемую страницу:

```yaml
ready:
  url:
    contains: viewtopic.php?t={{ item.torrent_id }}
  dom_contains:
    - dl.php?t={{ item.torrent_id }}
```

Поддерживаются URL-поля `exact`, `contains`, `contains_any`, DOM-поля `dom_contains`, `dom_contains_all`, `dom_contains_any`, а также `dom_regex`.

Не используйте признаки страницы CAPTCHA как успешные: runner должен продолжать ждать, пока пользователь не пройдёт проверку.

## Извлечение данных

### Заголовок

```yaml
title:
  selector: title
  cleanup:
    - trim_suffix: " :: Example"
```

`selector: title` извлекает содержимое HTML `<title>`. Полноценные CSS-селекторы пока не реализованы: при другом значении `selector` источником остаётся всё тело страницы, после чего применяется `regex`.

### Время обновления

```yaml
updated_at:
  selector: body
  regex: "Обновлено: ([0-9]{2}\.[0-9]{2}\.[0-9]{4} [0-9]{2}:[0-9]{2})"
  layout: "02.01.2006 15:04"
  timezone: Europe/Moscow
```

Первая capture group регулярного выражения должна содержать дату. `layout` использует формат времени Go. Можно задать `layouts` со списком альтернатив и `locale` для локализованных названий месяцев.

`cleanup` поддерживает `trim_prefix`, `trim_suffix`, `replace` + `with`, а также `regex` + `with`.

## Закрытая тема

```yaml
closed:
  contains_any:
    - Тема закрыта
    - раздача закрыта
```

Выбирайте текст, который однозначно означает закрытие раздачи.

## `download`

| Поле | Описание |
| --- | --- |
| `url` | Прямой URL загрузки. |
| `url_from_page` | Извлечение ссылки из HTML, если прямого шаблона недостаточно. |
| `before.headers` | Заголовки перед загрузкой, например `Referer`. |
| `before.set_cookie` | Cookies, которые нужно установить перед загрузкой. |
| `request` | Расширенное описание HTTP-запроса. |
| `validate` | Проверка полученного файла. |

Пример cookie:

```yaml
before:
  set_cookie:
    - name: bb_dl
      value: "{{ item.torrent_id }}"
      domain: example.org
      path: /
```

## Проверка загрузки

```yaml
validate:
  bencode_torrent: true
  max_size_mb: 5
  reject_content_types:
    - text/html
  reject_if_starts_with:
    - "<!DOCTYPE html"
    - "<html"
```

`bencode_torrent: true` проверяет, что ответ похож на bencoded `.torrent`. Ограничение размера защищает от ошибочной загрузки большой HTML-страницы или другого файла.

## Контрольный список

- YAML загружается без ошибок;
- `id` совпадает с основным доменом;
- URL темы распознаёт `torrent_id`;
- Native-авторизация сохраняет рабочую cookie либо Chromium позволяет пройти CAPTCHA;
- `ready` отвергает страницу входа/Cloudflare;
- заголовок и дата извлекаются стабильно;
- закрытая тема определяется однозначно;
- загрузка возвращает bencoded `.torrent`, а не HTML;
- шаблон не содержит секретов.
