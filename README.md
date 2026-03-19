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
