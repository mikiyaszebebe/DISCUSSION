# AI Agent Discussion Platform (Python)

Opinionated plan for a discussion platform with AI assistance. Stack: FastAPI, Hugging Face Inference API for LLM calls, PostgreSQL for storage.

## Core Goals
- Threaded discussions with messages and attachments.
- AI helper to summarize threads, propose replies, and suggest routing/labels.
- Moderation guardrails (rate limits, toxicity check hook).
- Webhook/API-first so UI clients can be built separately.

## High-Level Architecture
- FastAPI app: REST + WebSocket for live updates; authentication via JWT or provider SSO.
- Background worker (Celery/Arq/RQ) for LLM calls, summarization, async indexing.
- Hugging Face Inference endpoint: completion/summarization; swap to local models later.
- PostgreSQL: users, discussions, messages, agent jobs, labels.
- Cache/queue: Redis (for rate limits, job queue, websocket fan-out).

## Data Model (minimum viable)
- users: id, email, display_name, role, created_at.
- discussions: id, title, status (open/closed), created_by, created_at, updated_at.
- messages: id, discussion_id, author_id, body, metadata (JSON), created_at.
- agent_jobs: id, discussion_id, type (summary, draft_reply, classify), status, result (JSON), created_at, completed_at.
- labels: id, name; discussion_labels: discussion_id, label_id.

## API Surface (sketch)
- POST /auth/login, POST /auth/register (or SSO stub).
- GET/POST /discussions; GET/PATCH /discussions/{id}.
- GET/POST /discussions/{id}/messages.
- POST /discussions/{id}/agent: {type: summary|draft_reply|classify} → job id.
- GET /agent-jobs/{id} → status/result.
- WebSocket /ws/discussions/{id}: stream new messages and agent events.

## AI Workflows
- Summarize: worker pulls latest N messages, calls HF endpoint, stores summary in agent_jobs, emits event.
- Draft reply: use system prompt with thread context and style guide; return candidate reply with rationale.
- Classify/route: predict labels or assignees; store in agent_jobs and discussion_labels.
- Safety: pass content through moderation hook before LLM; block or redact on hit.

## Config & Secrets
- HF_API_TOKEN: Hugging Face Inference token.
- DATABASE_URL: postgres connection string.
- REDIS_URL: queue/cache backend.
- APP_SECRET_KEY: signing key for JWT/sessions.
- ALLOWED_ORIGINS: CORS whitelist.

## Local Dev Outline
- Python 3.11+; package manager: uv or pip + venv.
- FastAPI + Uvicorn for API; pick a worker (Celery/Arq/RQ) and keep it minimal.
- Migrations: Alembic.
- Tests: pytest + httpx; add VCR-like fixtures for HF calls.

## Milestones
1) Scaffold FastAPI project, Postgres schema, basic auth.
2) CRUD for discussions/messages; WebSocket broadcast.
3) Hook Hugging Face endpoint for summaries and draft replies via background jobs.
4) Add labels/classification and moderation hook.
5) Ship minimal UI or API client examples; add observability (logging, metrics).
