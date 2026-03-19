## Set up Guide

### 1. Create a Virtual Environment

```bash
python -m venv venv
```

*Note:* Make sure you have Python 3.12.3 or compatible version installed.

### 2. Activate the Virtual Environment

#### Mac/Linux:

```bash
source venv/bin/activate
```

#### Windows:

```bash
venv\Scripts\activate
```

### 3. Install the Required Libraries using poetry

```bash
poetry install
```

### 4. Create a .env and add the environment variables present in the .env.example file

```bash
touch .env
```

```bash
cp .env.example .env
```

### 5. Apply Database Migrations

```bash
poetry run python manage.py migrate
```

### 6. Run the Server

#### Using Uvicorn (Recommended for ASGI support)

```bash
poetry run uvicorn myapp.asgi:application --host 0.0.0.0 --port 8000 --reload
```

#### Using Django Development Server (Alternative)

```bash
poetry run python manage.py runserver
```

*Note:* Uvicorn is recommended as it provides ASGI support for async features and WebSockets.

### 7. Install Redis

#### Windows:
Useful Links: 
  - [https://naveenrenji.medium.com/install-redis-on-windows-b80880dc2a36](https://naveenrenji.medium.com/install-redis-on-windows-b80880dc2a36)
  - [https://redis.io/docs/latest/operate/oss_and_stack/install/install-redis/install-redis-on-windows/](https://redis.io/docs/latest/operate/oss_and_stack/install/install-redis/install-redis-on-windows/)
  - [https://github.com/tporadowski/redis](https://github.com/tporadowski/redis)

#### Mac:
```bash
brew install redis
```

#### Linux (Ubuntu):
```bash
sudo apt update
sudo apt install redis-server
```

### 8. Run Celery Worker

(Before running Celery, make sure Redis is properly set up on your machine.)

#### Windows:

```bash
celery -A myapp worker --loglevel=info --concurrency=5 --pool=solo
```

#### Mac/Linux (Ubuntu):

```bash
celery -A myapp worker --loglevel=info --concurrency=5
```

*Note:* The `--pool=solo` flag is required on Windows but not necessary on Mac/Linux.

After installation, start Redis using:
```bash
redis-server
```

### 9. Run Tornado Server (For Realtime Features)

If you need realtime features like WebSocket support, run the Tornado server:

```bash
poetry run python tornado_server.py
```

*Note:* The Tornado server runs on port 8888 by default and handles realtime subscriptions and events.

### 10. Run Docker locally

```bash
# Copy .env.example to .env.local
cp .env.example .env.local

docker compose -f docker-compose.dev.yml --env-file .env.local up

# To run in detached mode:
docker compose -f docker-compose.dev.yml --env-file .env.local up -d
```

You can now access the server at [http://localhost:8000/](http://localhost:8000/) and API documentation at [http://localhost:8000/api/docs/](http://localhost:8000/api/docs/).


# SciCommons Backend — GSoC 2026 PoC

Django-Ninja backend for SciCommons, an open peer review 
platform. This branch (`gsoc-2026-poc`) adds an AI-powered 
editorial workflow on top of the existing codebase.

## What this PoC adds

### Problem
`CommunityArticle` already defines seven status values 
(`submitted → under_review → accepted → published`) but 
no API endpoints or UI used any of them. Reviewer 
assignment was random — no expertise matching existed.

### Solution
Two new Django models and four new endpoints implementing 
an editorial workflow informed by studying Kotahi 
(Coko Foundation) and OJS (PKP).

## New models

**ReviewerInvitation**
Tracks Kotahi's reviewer status flow:
`Invited → Accepted → In Progress → Completed`

Fields: `article`, `community`, `reviewer`, `invited_by`,
`status`, `similarity_score`, `invited_at`, `responded_at`,
`reviewer_opted_out`

**EditorialDecision**
Implements Kotahi's Decision tab verdict logic:
- Accept → enables Publish, moves to production
- Revise → author submits new version, new round begins  
- Reject → ends active review cycle

Fields: `community_article`, `editor`, `decision`,
`comments`, `submitted_to_author`, `published`, `decided_at`

## New endpoints

All registered on `articles_parent_router`:

| Method | URL | Description |
|--------|-----|-------------|
| GET | `/api/articles/{id}/suggest-reviewers` | AI reviewer ranking via sentence-transformers |
| POST | `/api/articles/{id}/invite-reviewer` | Create ReviewerInvitation record |
| PATCH | `/api/articles/reviewer-invitations/{id}/respond` | Accept or decline invitation |
| POST | `/api/articles/community-articles/{id}/decision` | Submit editorial decision |
| GET | `/api/articles/community-articles/{id}/editorial-status` | Full workflow status |

## AI matching

Uses `sentence-transformers` (`all-MiniLM-L6-v2`) loaded 
at module level. Computes cosine similarity between paper 
abstract and reviewer bio. Locally hosted — zero API costs.

In testing with a neuroscience paper:
- fMRI specialist: 64% match
- Botanist (unrelated field): 5% match

## Architecture notes

Informed by studying:
- **Kotahi** (Coko Foundation) — Team tab reviewer flow, 
  Decision tab verdict logic, reviewer status progression
- **OJS** (PKP) — reviewer interest matching, 
  submission tracking metrics

Not yet implemented (full GSoC scope):
- Multiple review rounds with versioning
- Email notifications via Celery
- Reviewer anonymization controls
- pgvector for persistent embeddings

## Setup
```bash
# Copy env file
cp .env.example .env.local

# Start with Docker
docker compose -f docker-compose.dev.yml --env-file .env.local up --build

# Seed test data
docker exec -it my_scicommons-backend-web-1 python manage.py seed_poc_data

# API docs
open http://localhost:8000/api/docs/
```

## Stack

- Django + Django-Ninja
- PostgreSQL
- Redis + Celery
- Tornado (WebSocket)
- sentence-transformers (CPU)
