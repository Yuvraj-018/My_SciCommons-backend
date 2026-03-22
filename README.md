# SciCommons Backend — GSoC 2026 PoC

Django-Ninja backend for SciCommons, an open peer review 
platform built by INCF over three GSoC cycles.

## What is SciCommons

SciCommons is an open-source academic publishing platform 
where researchers submit papers, join communities (acting 
as open journals), get reviewed, and discuss research 
publicly — without paywalls or corporate intermediaries.

Three years of development produced solid infrastructure: 
community management, article submission, pseudonymous 
reviews, discussion threads, and real-time WebSocket 
support. The platform has been in alpha testing because 
one critical layer was never built.

## The Problem

`CommunityArticle` defines seven status values:
```
submitted → approved → under_review → accepted → 
rejected → published → unpublished
```

Zero API endpoints use them. Zero UI surfaces them.

A paper gets submitted. Reviewers are assigned randomly — 
a neuroscience paper can land with a botanist. No editor 
can formally accept or reject. No community can declare 
a paper published. The post-submission pipeline does not 
exist.

This is why the platform cannot leave alpha.

## What This PoC Adds

### Layer 1 — AI Reviewer Matching

`articles/reviewer_recommendation.py`

Uses `sentence-transformers` (`all-MiniLM-L6-v2`, locally 
hosted, zero API costs) to compute cosine similarity 
between the paper abstract and each community member's 
bio. Returns top 5 matches ranked by score.

| Reviewer | Bio focus | Match score |
|---|---|---|
| dr_fmri_expert | fMRI, prefrontal cortex, attention | 64% |
| clinical_neurologist | Frontal lobe lesions, attention networks | 53% |
| eeg_researcher | EEG, temporal attention dynamics | 41% |
| computational_modeler | Neural network modeling | 32% |
| botanist_user | Plant biology, photosynthesis | 5% |

Random assignment treats all five identically.

### Layer 2 — Editorial Workflow

Informed by studying Kotahi (Coko Foundation) and 
OJS (PKP).

**New model: ReviewerInvitation**

Tracks Kotahi's reviewer status flow exactly:
```
Invited → Accepted → In Progress → Completed
```

Declined reviewers tracked separately with opt-out flag.

**New model: EditorialDecision**

Implements Kotahi's Decision tab verdict logic:

| Decision | System action |
|---|---|
| Accept | Enables Publish, status → accepted |
| Minor Revision | Status → under_review, new round |
| Major Revision | Status → under_review, new round |
| Reject | Ends review cycle, status → rejected |

**New endpoints**

All registered on `articles_parent_router`:

| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/articles/{id}/suggest-reviewers` | AI-ranked reviewer list |
| POST | `/api/articles/{id}/invite-reviewer` | Create ReviewerInvitation |
| PATCH | `/api/articles/reviewer-invitations/{id}/respond` | Accept or decline |
| POST | `/api/articles/community-articles/{id}/decision` | Submit verdict |
| GET | `/api/articles/community-articles/{id}/editorial-status` | Full workflow state |

## Architecture
```
articles/
├── reviewer_recommendation.py  ← AI matching endpoint
├── editorial_workflow.py       ← 4 workflow endpoints
├── models.py                   ← ReviewerInvitation, EditorialDecision
└── management/commands/
    └── seed_poc_data.py        ← test data with realistic bios
myapp/
└── api.py                      ← router registration
```

## How the AI Matching Works
```python
# Module-level load — not per request
model = SentenceTransformer("all-MiniLM-L6-v2")

# Per request
article_text = f"{article.title}. {article.abstract}"
article_embedding = model.encode(article_text)

for member in community_members:
    user_text = f"{user.username} {user.bio}"
    user_embedding = model.encode(user_text)
    score = util.cos_sim(article_embedding, user_embedding)
```

Submitter is excluded from recommendations — 
authors cannot be suggested to review their own paper.

## Quick Start
```bash
# Copy env
cp .env.example .env.local

# Start all services
docker compose -f docker-compose.dev.yml \
  --env-file .env.local up --build

# Seed test data
docker exec -it my_scicommons-backend-web-1 \
  python manage.py seed_poc_data

# API docs
open http://localhost:8000/api/docs/
```

## Test the Endpoints
```bash
# Get AI reviewer suggestions
curl "http://localhost:8000/api/articles/1/suggest-reviewers?community_id=1"

# Invite a reviewer
curl -X POST "http://localhost:8000/api/articles/1/invite-reviewer" \
  -H "Content-Type: application/json" \
  -d '{"community_id":1,"reviewer_user_id":1,"similarity_score":0.64}'

# Get full editorial status
curl "http://localhost:8000/api/articles/community-articles/1/editorial-status"
```

## Known Limitations

- Embeddings recomputed on every request — production 
  version needs pgvector for persistent storage
- No email notifications on invitation or decision — 
  Celery tasks not yet implemented
- Single review round only — Kotahi supports multiple 
  rounds with versioning, not yet built
- No reviewer anonymization controls
- Auth not enforced on PoC endpoints — any user can 
  invite reviewers or submit decisions

## Stack

- Django 5 + Django-Ninja
- PostgreSQL
- Redis + Celery
- Tornado (WebSocket/realtime)
- sentence-transformers (CPU, all-MiniLM-L6-v2)

## Related

Frontend: https://github.com/Yuvraj-018/my_SciCommons-frontend/tree/gsoc-2026-poc

GSoC 2026 — INCF, Project #18
Mentors: Armaan Alam, Mohd Faisal Ansari, Suresh Krishna
