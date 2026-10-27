# Implementation Plan: Daily Chat Digest

**Branch**: `001-daily-chat-digest` | **Date**: 2026-10-27 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-daily-chat-digest/spec.md`

## Summary

Each chat the bot has joined gets one Russian recap of the previous Minsk day: notable personal events members typed themselves, plus a stored count of consecutive fully visible silent days. Two services do the work. The NestJS backend collects Telegram updates, keeps members and silence counts in PostgreSQL, and publishes the recap. The Python service pulls an extraction job from Redis and returns a short Russian narrative of personal news, plus a work log the backend stores. The recap body and the message text are not stored in PostgreSQL. Both services authenticate every call, and the bot acts only in chats where it is a member.

The AI container runs retrieval over that day's eligible messages and then a local generator. The embedder is `bge-m3`. The generator is `Qwen3.5-4B` as a GGUF file loaded in the worker. No request goes to Alibaba or any other model host. Silence lines, eligibility filters, and the publish decision stay deterministic. A synthetic set of 24 cases in `contracts/digest-cases.json` drives the tests, including which lines retrieval must keep and which it must drop.

## Technical Context

**Language/Version**: Backend Node.js 22 and NestJS 11. AI service Python 3.12.

**Primary Dependencies**: NestJS feature modules, TypeORM, Telegraf (`nestjs-telegraf`), ioredis, class-validator. Python: pydantic, redis, cryptography, structlog, pytest, fastembed, llama-cpp-python. Models are local files: `BAAI/bge-m3` and `Qwen3.5-4B` GGUF `Q4_K_M`.

**Storage**: PostgreSQL 16 for chats, users, silence counts, digest-run state, and action logs. Redis 7 for the encrypted day buffer, the job and result lists, and the same-day result cache. Message text and the recap narrative are not PostgreSQL columns.

**Testing**: Jest in `backend`. pytest in `ai`. Shared golden cases in `contracts/digest-cases.json`. Default runs stub the embedder and the generator. A separate `local_eval` mark loads the GGUF and `bge-m3` from the image.

**Target Platform**: Linux services on a private network. Telegram is the only user-facing channel. The day boundary is `Europe/Minsk`.

**Project Type**: Two backend services (Telegram orchestration and an extraction worker).

**Performance Goals**: When the 00:00 Minsk run can publish, the recap is in the chat by 00:05 (SC-001). The webhook acknowledges Telegram without waiting for the model. One local job has 90 seconds for embedding and generation. The backend waits up to 120 seconds for the result.

**Constraints**: At most 3 publication attempts per chat per recap day, all inside the new Minsk day. Retrieval keeps at most 40 messages. Generator output cap 800 tokens, thinking off, one generation at a time. Logs and errors carry allowlisted codes only. Secrets come from the environment. The AI service opens no listening port and does not call a model host at runtime.

**Scale/Scope**: A small number of group chats, hundreds of members each, one recap per chat per day.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Gate | Requirement | Plan | Status |
| --- | --- | --- | --- |
| I. Security and privacy | Validate input. No plaintext secrets or lasting message text. Authorized components only. | Webhook secret, schema checks, AES-256-GCM on Redis payloads, digest and message text excluded from PostgreSQL, chat membership required. | Pass |
| II. Configuration and secrets | No hardcoded or committed credentials. | Tokens, Redis password, payload key, and HMAC key are environment variables. Model weights are image files, not secrets, and are not committed as source. | Pass |
| III. Architecture and contracts | Loose coupling. No cross-database access. | NestJS and Python meet only through the signed Redis job and result. | Pass |
| IV. Resilience and UX | A failed dependency does not kill the bot and does not stay silent. | AI and model failures become one Russian failure note. A later success still posts the one recap. | Pass |
| V. Rate limiting and performance | Every public entry point is limited. One request cannot stall the rest. | Per-chat update cap, `/logs` cap, webhook returns before the model call, two-call model pool. | Pass |
| VI. Documentation | English docs for behavior and contracts. | This plan, research, data model, contracts, and quickstart are English. User-facing Telegram copy is Russian, as the spec requires. | Pass |
| VII. Testing | Automated tests for the new behavior. | Golden cases cover the recap combinations. Both services test against them. | Pass |
| VIII. Observability | Structured logs without message content, credentials, or the recap body. | Allowlisted `logs` actions and error codes. AI `work_log` has the same rule. | Pass |
| IX. AI cost | Paid calls only when a deterministic step cannot answer. Smallest sufficient model. Bounded input, output, and retries. No repeat while a result exists. | No paid model API. Local `Qwen3.5-4B` runs only on retrieved messages. Empty retrieval skips generation. Output capped. Cache by `job_id`. | Pass |

Post-design re-check: the data model, queue contract, retrieval contract, copy contract, and golden cases introduce no new violation. PostgreSQL still has no message or narrative column. Embeddings are not stored. The result cache holds the narrative only until the run settles, encrypted, outside PostgreSQL. No gate needs an exception.

## Project Structure

### Documentation (this feature)

```text
specs/001-daily-chat-digest/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   ├── digest-job.schema.json
│   ├── digest-result.schema.json
│   ├── digest-cases.json
│   ├── queue.md
│   ├── rag.md
│   └── telegram-copy.md
└── tasks.md              # /speckit-tasks, not created here
```

### Source Code (repository root)

```text
backend/
├── src/
│   ├── app.module.ts
│   ├── config/
│   ├── database/
│   ├── auth/
│   ├── telegram/
│   ├── queue/
│   ├── users/
│   ├── logs/
│   └── digest/
└── test/

ai/
├── src/
│   ├── worker.py
│   ├── auth.py
│   ├── rag.py
│   ├── generate.py
│   └── logging.py
├── Dockerfile
└── tests/
```

**Structure Decision**: Two deployable services at the repository root. `backend` is a NestJS modular application; each directory under `src` is a feature module that owns its types and, where it has tables, its TypeORM entities. `ai` is a queue worker with no HTTP server. Its image contains the embedding model and the GGUF generator, and the worker loads both in-process. Tests in each service read `specs/001-daily-chat-digest/contracts/digest-cases.json`.

## Complexity Tracking

No constitution violations require justification. The second service is the requested AI boundary and the constitution's microservice rule, not an extra project beyond that rule.

## Design

### Flow

1. Telegram updates hit the backend webhook. The handler checks the webhook secret, the update shape, and that the bot is a member of the chat. It returns immediately.
2. Member updates create or refresh `users`. Eligible and ineligible messages for the open Minsk day go into the encrypted Redis buffer. Forwards, captions, service notices, and bot messages are kept only long enough to know they do not count as typing, and their text is not forwarded to the model.
3. At 00:00 `Europe/Minsk` the digest module opens a run for the previous Minsk day. It decides whether the day was fully readable.
4. If any member typed eligible text, the backend pushes one signed job. The AI worker embeds those messages, retrieves the ones that match the personal-news queries in `contracts/rag.md`, and runs the local generator only on that set. It returns `narrative_ru` plus `work_log`. If retrieval keeps nothing, it returns an empty narrative and does not generate. If no eligible text exists, the backend does not push a job.
5. The backend updates `consecutive_silent_days` using the rules in `data-model.md`, composes the Telegram message from `contracts/telegram-copy.md`, and sends it. Empty fully visible days send nothing. Unreadable days send the one failure sentence.
6. The backend inserts `work_log` and its own actions into `logs`. It discards the narrative after send. It deletes the buffer and the result cache when the run settles.
7. A member of that chat can send `/logs`. The reply lists that chat's action rows in Russian and includes no message text and no narrative.

Failed attempts retry at +15 minutes and +30 minutes, at most three times, and only during the same new Minsk day. A published recap or an empty skip settles the run.

### Authentication

Documented in `contracts/queue.md`. The backend rejects Telegram updates that fail the webhook secret or come from a chat the bot is not in. `/logs` also checks that the sender is a current member. Redis uses a password, a private bind, and two ACL users. Payloads are encrypted and signed. The AI service has no public socket and drops unsigned or invalid jobs.

### Retrieval and model

Documented in `research.md` and `contracts/rag.md`. The worker loads `bge-m3` and `Qwen3.5-4B` `Q4_K_M` from `/models`. The index for a job is discarded with the job. Thinking is off. Output is capped at 800 tokens. A repeated `job_id` reuses the encrypted cache and does not embed or generate again. The running container does not call Alibaba or any other model host.

### Tests

`contracts/digest-cases.json` holds 24 synthetic combinations: each personal-news category, plans and jokes and greetings, small talk, forwards, captions, replies, duplicates, five people at once, partial days, an unreadable day, service notices, silence counts of 1, 2, and 5, a reset, a retry, bot text, an English fact recapped in Russian, and one mixed day. Each case also lists the eligible lines retrieval must keep and the eligible lines it must drop. `quickstart.md` is the runbook. The local eval is the SC-002 check and is not part of the default test run.

### Module boundaries

`DigestModule` decides and composes. `TelegramModule` speaks to Telegram. `QueueModule` speaks to Redis. `UsersModule` owns silence counts. `LogsModule` owns action rows and `/logs` text. `AuthModule` owns the secret checks and the payload crypto. No module queries another module's tables except through that module's service.
