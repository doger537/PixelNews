# PixelNews

FastAPI backend for real news ingestion, database-backed feed/search, parser logs, deduplication, ranking, personalization, and a minimal frontend at `/`.

## Local Run

```bash
py -m pip install -r requirements.txt
py -m uvicorn app.main:app --host 127.0.0.1 --port 8000
```

Open `http://127.0.0.1:8000`.

The local `.env` uses SQLite by default. PostgreSQL is supported through `DATABASE_URL`.

## Production Deploy

Server deployment is documented in [README_DEPLOY.md](README_DEPLOY.md).

```bash
docker compose -f docker-compose.prod.yml up -d --build --scale backend=2
docker compose -f docker-compose.prod.yml logs -f backend
docker compose -f docker-compose.prod.yml logs -f article-worker
docker compose -f docker-compose.prod.yml logs -f scheduler
docker compose -f docker-compose.prod.yml logs -f feed-cache
docker compose -f docker-compose.prod.yml logs -f postgres
```

Production uses PostgreSQL, Redis, nginx, remote feed/user cache services, fanout, one scheduler container for parsing, one article worker, and a background-only AI editor. Scale backend replicas with:

```bash
docker compose -f docker-compose.prod.yml up -d --scale backend=4
```

## Sources

Add an RSS source:

```bash
curl -X POST http://127.0.0.1:8000/sources ^
  -H "Content-Type: application/json" ^
  -d "{\"name\":\"Example RSS\",\"title\":\"Example RSS\",\"type\":\"rss\",\"rss_url\":\"https://example.com/rss\",\"is_active\":true}"
```

Add a website source:

```bash
curl -X POST http://127.0.0.1:8000/sources ^
  -H "Content-Type: application/json" ^
  -d "{\"name\":\"Example Site\",\"title\":\"Example Site\",\"type\":\"website\",\"url\":\"https://example.com/news\",\"is_active\":true}"
```

Telegram sources require Telethon credentials and authorization through `/auth/telegram/start` and `/auth/telegram/confirm`.

## Parser

Run parser for today:

```bash
curl -X POST http://127.0.0.1:8000/parser/run ^
  -H "Content-Type: application/json" ^
  -d "{\"period\":\"today\"}"
```

Run parser for yesterday:

```bash
curl -X POST http://127.0.0.1:8000/parser/run ^
  -H "Content-Type: application/json" ^
  -d "{\"period\":\"yesterday\"}"
```

Run parser for the day before yesterday:

```bash
curl -X POST http://127.0.0.1:8000/parser/run ^
  -H "Content-Type: application/json" ^
  -d "{\"period\":\"day_before_yesterday\"}"
```

Other supported periods are `last_24h`, `last_7d`, and `custom`:

```bash
curl -X POST http://127.0.0.1:8000/parser/run ^
  -H "Content-Type: application/json" ^
  -d "{\"period\":\"custom\",\"date_from\":\"2026-04-28T00:00:00+03:00\",\"date_to\":\"2026-04-29T00:00:00+03:00\"}"
```

The parser saves only real parsed articles. If a source fails, the error is saved in `parser_errors`; no fallback news are generated.

## Search

Search real articles from the database:

```bash
curl "http://127.0.0.1:8000/articles/search?q=киберспорт&period=today&limit=10"
```

Search supports `q`, `period`, `topic`, `source_id`, `limit`, and `offset`. Empty `q` returns a normal filtered feed from the database. Empty results stay empty.

## Parser Logs

```bash
curl "http://127.0.0.1:8000/parser/runs?limit=10"
curl "http://127.0.0.1:8000/parser/errors?limit=50"
curl "http://127.0.0.1:8000/parser/errors?parser_run_id=1"
```

## Main API

- `GET /sources`
- `POST /sources`
- `PATCH /sources/{id}`
- `POST /parser/run`
- `GET /parser/runs`
- `GET /parser/errors`
- `GET /articles`
- `GET /articles/search`
- `GET /articles/{id}`
- `GET /debug/articles/{id}/media`
- `POST /articles/{id}/media/select`

## Optional AI Filtering

Default filtering is local and rule based. To connect a neural provider, set:

```env
AI_FILTER_PROVIDER=ollama
AI_FILTER_BASE_URL=http://127.0.0.1:11434
AI_FILTER_MODEL=llama3.1
```

or an OpenAI-compatible endpoint:

```env
AI_FILTER_PROVIDER=openai_compatible
AI_FILTER_BASE_URL=https://api.openai.com/v1
AI_FILTER_API_KEY=
AI_FILTER_MODEL=gpt-4o-mini
```

## Background AI Editor

For production, keep live requests fast and warm edited posts in the background:

```env
AI_POST_EDITOR_ENABLED=false
AI_POST_EDITOR_BACKGROUND_ENABLED=true
AI_POST_EDITOR_BACKGROUND_BATCH_SIZE=5
AI_POST_EDITOR_SUMMARY_SIZES=short
```

Run one batch manually:

```powershell
py -3 -m app.commands.ai_editor_worker --once --force --limit 5 --summary-sizes short
```

## 24/7 Parsing

Local service mode can keep collecting posts while the computer is on:

```env
PARSER_SCHEDULER_ENABLED=true
PARSER_SCHEDULE_MINUTES=15
```

Then restart:

```powershell
.\scripts\start_service_mode.ps1
```

## Tests

```bash
py -m pytest app/tests
```

## Load Test

```bash
py scripts/load_test.py --url http://127.0.0.1:8000 --requests 200 --concurrency 20
```
