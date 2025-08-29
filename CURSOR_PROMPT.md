Отлично. Ниже — готовое **ТЗ-промпт для Cursor** (вставь целиком в Cursor IDE в виде System/Instruction сообщения или в начале диалога в файле `CURSOR_PROMPT.md`). Он спроектирует и напишет код парсера/агрегатора под ключевые слова, теги, ссылки и т. д.

---

# ТЗ для Cursor: “Нейро-парсер/агрегатор по ключевым словам, тегам и ссылкам”

## Цель

Создать кроссплатформенный инструмент, который:

1. собирает данные из заданных источников (URL, домены, RSS, sitemap, поисковая выдача),
2. фильтрует по ключевым словам/фразам/тегам,
3. структурирует сущности (заголовок, дата, автор, теги, язык, краткое резюме),
4. сохраняет в БД и экспортирует в CSV/JSONL,
5. имеет CLI и минимальный Web API.

## Технологии (по умолчанию)

* Язык: **Python 3.11+**
* Веб-драйвер: **Playwright** (рендеринг JS при необходимости)
* HTTP: `httpx` с таймаутами/ретраями
* Парсинг HTML: `selectolax` или `lxml` + CSS/XPath
* Текст/Язык/Ключи: `langdetect` или `fasttext` (если доступно), RAKE/KeyBERT (по возможности), простой частотный анализ n-грамм
* Хранилище: **SQLite** (по умолчанию) + опционально PostgreSQL
* Миграции: `alembic` (если Postgres) или автосоздание схемы для SQLite
* Логи: `structlog`
* Тесты: `pytest`
* API: **FastAPI**
* CLI: `typer`
* Настройки: `.env` + `pydantic-settings`
* Контейнер: **Dockerfile + docker-compose** (Playwright с браузерами)

## Функциональные требования

### Источники

* Ввод через `sources.yaml`:

  * `seed_urls`: список конкретных URL
  * `domains`: базовые домены для обхода (глубина 1–2)
  * `rss`: списки RSS-лент
  * `sitemaps`: ссылки на sitemap.xml
  * `search`: запросы и провайдеры (заглушка/интерфейс: подключить позже)
* Учитывать `robots.txt` и `crawl-delay`. Предусмотреть флаг `--ignore-robots` (по умолчанию — выключен).

### Фильтрация и извлечение

* Конфиг `filters.yaml`:

  * `include_keywords`: список ключей/фраз (регистронезависимо)
  * `exclude_keywords`: список стоп-слов
  * `tags_map`: словарь нормализации тегов (например, «web3», «defi», «крипта» → `crypto`)
  * `languages_allowed`: список языков (например, `["ru","en"]`)
* На каждой странице извлечь:

  * `url`, `final_url`, `http_status`
  * `title`, `author` (если найдено), `published_at` (ISO8601, попытка нормализации), `description`/`summary` (сгенерировать короткое резюме 1–2 предложения), `tags` (явные + выведенные по ключам), `lang`, `content_text` (очищенный)
* Дедупликация: по `final_url` и по хешу `content_text` (MinHash/sha256).
* Ограничения по размеру текста (например, до 100k символов), обрезать с логом.

### Краулинг

* Конфиги `crawler.yaml`:

  * `concurrency`: по умолчанию 8
  * `request_timeout_sec`: 20
  * `max_depth`: 2
  * `respect_robots`: true
  * `retry`: 3 попытки с экспоненциальной паузой
  * `user_agent`: настраиваемый
  * `proxies`: опционально
* Рендеринг JS включать точечно (например, по списку доменов `render_js_domains`).
* Обработка пагинации (общие паттерны: `?page=`, `&p=`, `next` rel-link).

### Хранилище и модель данных

Таблица `items`:

* `id` (PK), `url`, `final_url`, `domain`, `title`, `author`, `published_at` (datetime, nullable),
* `summary`, `tags` (array/json), `lang`,
* `content_text` (text), `hash` (uniq),
* `created_at` (UTC now), `source_type` (`seed|rss|sitemap|crawl|search`), `http_status`, `metadata` (json).
  Индексы: по `final_url`, `hash`, `published_at`, `lang`.

### Экспорт

* CLI флаги `--export csv` и `--export jsonl` в папку `./exports/YYYY-MM-DD/`.
* Фильтры экспорта: по датам, языкам, тегам, доменам, ключевым словам.

### API (FastAPI)

* `GET /health`
* `POST /crawl` — запускает сбор по конфигам (асинхронный таск, вернёт job_id)
* `GET /items` — пагинированная выдача с параметрами фильтрации: `q`, `tags`, `lang`, `date_from`, `date_to`, `domain`
* `GET /items/{id}` — детально
* `GET /export` — инициирует экспорт по параметрам, отдаёт файл

### CLI (Typer)

* `crawler init` — создаёт пример конфигов и БД
* `crawler run --sources sources.yaml --filters filters.yaml --crawler crawler.yaml`
* `crawler export --format csv --tags crypto,ai --lang ru,en --date_from 2025-01-01`
* `crawler api --host 0.0.0.0 --port 8000`

### Качество и DX

* Покрыть ключевые части `pytest`:

  * парсинг HTML-шаблонов (title/date/author)
  * фильтрацию по ключам/языку
  * дедупликацию
* `pre-commit` с `ruff`/`black`.
* Подробные структурированные логи (уровни: info/debug/warn/error).
* Документация в `README.md`:

  * установка, .env, примеры конфигов
  * примеры CLI команд
  * запуск API, Docker инструкции
  * советы по лимитам/антибану

### Юридическое/этичное

* По умолчанию **соблюдать robots.txt**, не обходить paywall, не собирать PII.
* В `README.md` добавить раздел “Legal & Ethics”.
* Все домены — только явно указанные пользователем в `sources.yaml`.

## Нефункциональные требования

* Производительность: 1000+ страниц/час при `concurrency=8` и без рендеринга JS.
* Надёжность: ретраи, бэкофф, логирование ошибок с контекстом.
* Расширяемость: архитектура через провайдеры источников и экстракторы.

## Архитектура каталогов (сгенерировать)

```
neurocrawler/
  app/
    api/
      __init__.py
      routes.py
    cli/
      __init__.py
      main.py
    core/
      config.py
      logging.py
      utils_text.py
      filters.py
    crawl/
      runner.py
      fetcher.py
      parser_generic.py
      parser_rules/
        common.py
        domain_example_com.py   # примеры доменных правил
      dedupe.py
    db/
      models.py
      session.py
      repo.py
    export/
      exporter.py
    schemas/
      items.py
  tests/
    test_parser.py
    test_filters.py
    test_dedupe.py
  configs/
    sources.yaml
    filters.yaml
    crawler.yaml
  .env.example
  requirements.txt
  README.md
  Dockerfile
  docker-compose.yml
  pyproject.toml
```

## Мини-правила реализации

* Сначала scaffolding проекта и конфигов.
* Затем минимальный happy-path: загрузка URL → парсинг title/summary → запись в SQLite → экспорт CSV → API `/items`.
* После — добавить язык/ключевые слова/теги/дедупликацию → RSS/Sitemap → рендеринг JS.
* Покрыть базовые тесты, положить фикстуры HTML.

## Примеры начальных конфигов (сгенерировать)

`configs/sources.yaml`:

```yaml
seed_urls:
  - https://example.com/news
domains:
  - example.com
rss:
  - https://example.com/feed.xml
sitemaps:
  - https://example.com/sitemap.xml
search:
  enabled: false
  queries: []
```

`configs/filters.yaml`:

```yaml
include_keywords:
  - "крипта"
  - "web3"
  - "AI"
exclude_keywords:
  - "вакансия"
tags_map:
  crypto: ["крипта","web3","bitcoin","defi"]
  ai: ["ai","искусственный интеллект","нейросеть"]
languages_allowed: ["ru","en"]
```

`configs/crawler.yaml`:

```yaml
concurrency: 8
request_timeout_sec: 20
max_depth: 2
respect_robots: true
retry: 3
user_agent: "NeuroCrawler/1.0 (+contact@example.com)"
render_js_domains:
  - "some-js-site.com"
proxies: []
```

`.env.example`:

```
DB_URL=sqlite+aiosqlite:///./neurocrawler.db
API_HOST=0.0.0.0
API_PORT=8000
```

## Acceptance Criteria (что считать “готово”)

1. `crawler init` создаёт БД и пример конфигов.
2. `crawler run` собирает минимум 50 документов с `example.com`/RSS, применяет фильтры, сохраняет в БД, логирует статистику.
3. `crawler export --format csv` создаёт файл с колонками: url, title, published_at, tags, lang, summary.
4. `crawler api` поднимает сервис; `GET /items` возвращает список (пагинация, фильтры работают).
5. Тесты `pytest` проходят локально и в контейнере.

---

## Что нужно сделать прямо сейчас (Cursor, выполни по шагам)

1. Создай проект с указанной структурой, зависимостями и базовыми файлами.
2. Реализуй минимальный функционал: загрузка страницы → извлечение title → сохранение в SQLite → API `/items`.
3. Добавь фильтры include/exclude keywords, распознавание языка, дедупликацию по URL/контенту.
4. Реализуй экспорт CSV/JSONL.
5. Напиши тесты `test_parser.py`, `test_filters.py`, `test_dedupe.py`.
6. Сгенерируй `Dockerfile` и `docker-compose.yml` (Playwright + FastAPI + БД).
7. Подготовь `README.md` с инструкциями и примерами команд.

---

### Подсказки для меня (пользователя)

После генерации кода я буду запускать:

```bash
python -m pip install -r requirements.txt
python -m neurocrawler.app.cli.main init
python -m neurocrawler.app.cli.main run --sources configs/sources.yaml --filters configs/filters.yaml --crawler configs/crawler.yaml
python -m neurocrawler.app.cli.main export --format csv
python -m neurocrawler.app.cli.main api --host 0.0.0.0 --port 8000
```

