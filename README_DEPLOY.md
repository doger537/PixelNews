# PixelNews Production Deployment

This production setup runs PixelNews behind nginx with PostgreSQL, Redis, feed/user cache services, fanout, a scheduled parser worker, an article worker, and a background-only AI editor.

## 1. Prepare

```bash
git clone <repo-url> pixelnews
cd pixelnews
cp .env.production.example .env.production
mkdir -p data/media data/telegram_sessions data/queues
```

Edit `.env.production` before public launch:

- set `POSTGRES_PASSWORD`
- set the same password inside `DATABASE_URL`
- set a long random `SECRET_KEY`
- set `FRONTEND_URL`
- add Telegram credentials
- replace Turnstile test keys with real Cloudflare Turnstile keys

## 2. Start

Run 2 backend containers first:

```bash
docker compose -f docker-compose.prod.yml up -d --build --scale backend=2
```

Scale to 4 backend containers when CPU/RAM allow it:

```bash
docker compose -f docker-compose.prod.yml up -d --scale backend=4
```

The `migrator` service runs `alembic upgrade head` once. Backend replicas do not run migrations, so they can start safely in parallel.

## 3. Services

- `postgres`: main database
- `redis`: feed cache and queue backend
- `feed-cache`: remote cache for feed responses
- `user-cache`: remote cache for user/session-adjacent data
- `fanout`: receives parser-completed jobs
- `fanout-worker`: warms cache after parser runs
- `backend`: FastAPI app, scaled with `--scale backend=2` or `--scale backend=4`
- `article-worker`: processes raw articles/media in the background
- `scheduler`: runs the parser on `PARSER_SCHEDULE_MINUTES`
- `ai-editor-worker`: warms AI-edited post text in the background
- `nginx`: public entrypoint and load balancer

AI/Ollama is intentionally not used while opening the feed:

```env
AI_POST_EDITOR_ENABLED=false
AI_POST_EDITOR_BACKGROUND_ENABLED=true
```

## 4. Seed And Login

```bash
docker compose -f docker-compose.prod.yml exec backend python -m app.commands.seed_sources
docker compose -f docker-compose.prod.yml exec backend python -m app.commands.telegram_login
```

Manual parser run:

```bash
docker compose -f docker-compose.prod.yml exec backend python -m app.commands.run_parser --period last_24h
```

## 5. Checks

```bash
curl http://your-server/health
curl http://your-server/debug/db-status
curl http://your-server/debug/parser-status
curl http://your-server/debug/telegram-status
curl http://your-server/articles?period=last_24h
```

Logs:

```bash
docker compose -f docker-compose.prod.yml logs -f nginx backend scheduler article-worker feed-cache redis postgres
```

## 6. Load Test

Local check:

```bash
python scripts/load_test.py --url http://127.0.0.1:8000 --requests 200 --concurrency 20
```

Server check through nginx:

```bash
python scripts/load_test.py --url http://your-server --requests 1000 --concurrency 100
```

For a first VPS, start with `--concurrency 20`, then increase to `50` and `100`. Watch CPU, RAM, PostgreSQL, Redis, and nginx/backend logs while testing.
