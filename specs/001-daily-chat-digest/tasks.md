---
description: "Task list for Daily Chat Digest implementation"
---

# Tasks: Daily Chat Digest

**Input**: Design documents from `/specs/001-daily-chat-digest/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/, quickstart.md, `.specify/memory/constitution.md`

**Tests**: Required. Constitution VII and the plan's Testing section require automated tests for this feature. Write the test tasks in each user-story phase first and confirm they fail before the implementation tasks in that phase. Default runs stub the embedder and the generator. The `local_eval` mark loads local weights and is not part of the default run.

**Organization**: Tasks are grouped by user story so each story can be implemented and tested on its own.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies on incomplete tasks in the same phase)
- **[Story]**: Which user story this task belongs to (US1, US2)
- Setup, Foundational, and Polish tasks have no story label

## Path Conventions

- `backend/` — NestJS 11 service (Node.js 22), Jest tests under `backend/test/`
- `ai/` — Python 3.12 queue worker, pytest under `ai/tests/`
- Contracts and golden cases stay in `specs/001-daily-chat-digest/contracts/`
- PostgreSQL is owned by `backend`. `ai` has no database. Message text and `narrative_ru` are never PostgreSQL columns.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Create the two empty services and the local data stores the plan names

- [ ] T001 Create the NestJS 11 application under `backend/` for Node.js 22 with `backend/package.json`, `backend/tsconfig.json`, `backend/nest-cli.json`, `backend/src/main.ts`, and `backend/src/app.module.ts`. Depend on `@nestjs/core`, `@nestjs/common`, `@nestjs/typeorm`, `typeorm`, `pg`, `nestjs-telegraf`, `telegraf`, `ioredis`, `class-validator`, `class-transformer`, and `ajv`. Configure Jest so `npm test --prefix backend` runs `backend/test/**/*.spec.ts`. Add ESLint for `backend/`. Do not commit secrets.
- [ ] T002 [P] Create the Python 3.12 package under `ai/` with `ai/pyproject.toml` so `pip install -e ai` works and `python -m pytest ai/tests -m "not local_eval"` is the default command. Depend on `pydantic`, `redis`, `cryptography`, `structlog`, `pytest`, `fastembed`, `llama-cpp-python`, and `jsonschema`. Set the pytest `pythonpath` to `ai` and register marker `local_eval`. Add Ruff configuration in `ai/pyproject.toml`.
- [ ] T003 [P] Add `docker-compose.yml` and `docker/redis/users.acl` for PostgreSQL 16 and Redis 7. Redis requires `REDIS_PASSWORD`, binds to a private interface, and defines two ACL users from `specs/001-daily-chat-digest/contracts/queue.md`: `backend` may read and write `buffer:*`, push `digest:jobs`, pop `digest:results`, and read `ai:heartbeat`; `ai` may pop `digest:jobs`, push `digest:results`, read and write `digest:cache:*`, and set `ai:heartbeat`. The `ai` user cannot read `buffer:*`.
- [ ] T004 [P] Add `ai/Dockerfile` that copies untracked build inputs `models/embedder` (`BAAI/bge-m3`) to `/models/embedder` and `models/generator.gguf` (`Qwen3.5-4B` GGUF `Q4_K_M`) to `/models/generator.gguf`. Use a llama.cpp build new enough for Qwen3.5 hybrid attention (b9222 or newer). Set `HF_HUB_OFFLINE=1`. Publish no port. Do not set a generation API key or a generation base URL. Do not commit the weight files.
- [ ] T005 [P] Add `.env.example` and gitignore rules for `.env` and `models/`. Document `TELEGRAM_BOT_TOKEN`, `TELEGRAM_WEBHOOK_SECRET`, `DATABASE_URL`, `REDIS_URL`, `REDIS_PASSWORD`, `REDIS_PAYLOAD_KEY` (32 bytes, base64), `INTERNAL_HMAC_KEY`, and `RAG_SCORE_THRESHOLD` (starting point `0.35`). Put no secret values in the example or in git.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Shared config, persistence, crypto, chat gate, and the action-log table. No user-story work starts before this phase.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [ ] T006 [P] Implement `backend/src/config/env.schema.ts` and `backend/src/config/config.module.ts`. Require `TELEGRAM_BOT_TOKEN`, `TELEGRAM_WEBHOOK_SECRET`, `DATABASE_URL`, `REDIS_URL`, `REDIS_PASSWORD`, `REDIS_PAYLOAD_KEY`, and `INTERNAL_HMAC_KEY`. `REDIS_PAYLOAD_KEY` must decode to 32 bytes. Refuse to start when any value is missing. Do not hardcode credentials.
- [ ] T007 [P] Implement `backend/src/database/database.module.ts` with a TypeORM connection to PostgreSQL 16 through `DATABASE_URL` only. Load migrations from `backend/src/database/migrations/`.
- [ ] T008 [P] Implement the queue envelope in `backend/src/auth/payload-crypto.ts` per `specs/001-daily-chat-digest/contracts/queue.md`: serialize JSON with sorted keys and UTF-8; encrypt with AES-256-GCM using `REDIS_PAYLOAD_KEY` and a fresh 12-byte nonce per message; sign `nonce || ciphertext || tag` with HMAC-SHA256 and `INTERNAL_HMAC_KEY`; verify the signature with a constant-time compare, then decrypt.
- [ ] T009 [P] Implement the same envelope in `ai/src/auth.py` (AES-256-GCM, fresh 12-byte nonce, HMAC-SHA256 over `nonce || ciphertext || tag`, constant-time compare before decrypt) so `ai` can open jobs and seal results without reading PostgreSQL.
- [ ] T010 [P] Implement structured logging in `backend/src/logging/app-logger.ts` and `ai/src/logging.py` using structlog on the Python side. Log allowlisted action and error codes only. Logs must not contain message content, credentials, tokens, display names, or `narrative_ru`.
- [ ] T011 [P] Add the Chat entity in `backend/src/telegram/chat.entity.ts` and migration `backend/src/database/migrations/1740000000001-create-chats.ts` for table `chats`: `telegram_chat_id` bigint primary key, required; `bot_is_member` boolean not null, true only while the bot status is `member` or `administrator`, false after `left` or `kicked`; `updated_at` timestamptz not null, set on every membership change. Store no chat title.
- [ ] T012 [P] Add the Log entity in `backend/src/logs/log.entity.ts`, `backend/src/logs/logs.module.ts`, and `backend/src/logs/logs.service.ts`, plus migration `backend/src/database/migrations/1740000000002-create-logs.ts` for table `logs`: `id` uuid primary key; `telegram_chat_id` bigint not null referencing `chats.telegram_chat_id`; `action` text not null allowlist `collect_day`, `extract_events`, `publish_recap`, `publish_failure`, `skip_empty`, `show_logs`, `rejected_result`; `recap_day` date not null; `success` boolean not null; `attempt_number` integer null, 1–3 when the action is an attempt and null otherwise; `source` text not null, `backend` or `ai`; `error_code` text null allowlist `model_timeout`, `model_unavailable`, `model_rejected`, `invalid_job`, `bad_signature`, with no free text; `created_at` timestamptz not null. Index (`telegram_chat_id`, `created_at`). Reject `action`, `source`, and `error_code` values outside those allowlists. Add no column for message text, display names, or the recap narrative.
- [ ] T013 [P] Implement `backend/src/queue/queue.module.ts` and `backend/src/queue/redis.service.ts` with ioredis using `REDIS_URL` and `REDIS_PASSWORD`. The backend may use `buffer:{chat}:{minsk_day}`, push `digest:jobs`, pop `digest:results`, and read `ai:heartbeat`. It does not read `digest:cache:*`.
- [ ] T014 [P] Add `backend/test/support/clock.ts` pinning "today" to `2026-10-27 00:00 Europe/Minsk` so `recap_day` is `2026-10-26`, and `backend/test/support/cases.ts` loading `specs/001-daily-chat-digest/contracts/digest-cases.json` and asserting 24 unique case ids.
- [ ] T015 Implement `backend/src/telegram/update-rate-limit.ts` and `backend/src/telegram/webhook.controller.ts`. Reject an update whose `X-Telegram-Bot-Api-Secret-Token` does not match `TELEGRAM_WEBHOOK_SECRET` and write nothing. Accept only update types `message`, `edited_message`, `chat_member`, and `my_chat_member`. Acknowledge Telegram before any digest or model work. Allow 30 accepted updates per 10 seconds per chat; do not store extras, and mark that Minsk day partly visible in the encrypted buffer metadata. `my_chat_member` maintains `chats.bot_is_member`. Drop every other update unless `bot_is_member` is true. Expose no HTTP route that accepts a chat id and returns a recap or a log.
- [ ] T016 Register `ConfigModule`, `DatabaseModule`, `AuthModule` (`backend/src/auth/auth.module.ts`), `TelegramModule` (`backend/src/telegram/telegram.module.ts`), `QueueModule`, and `LogsModule` in `backend/src/app.module.ts`. Export only each module's own service.

**Checkpoint**: Config, PostgreSQL, the chat gate, Redis, payload crypto, and the `logs` table are ready. User stories can start.

---

## Phase 3: User Story 1 - Read the day's personal news (Priority: P1) 🎯 MVP

**Goal**: At 00:00 Europe/Minsk, each joined chat receives one Russian recap of the previous Minsk day: notable personal events members typed themselves, plus stored full-day silence counts. Forwards, captions, plans, jokes, greetings, and small talk stay out. A failed attempt posts one Russian failure sentence, with at most three attempts during that new Minsk day.

**Independent Test**: Leave a fully visible day that includes a notable personal event a member typed, a notable event that appears only as a forward or a media caption, plus some plans and small talk, and one member who typed nothing. At midnight Minsk time the chat receives a Russian recap of the typed notable events only, the silent member is named with a count of 1 full day, and the recap can be retold without the day's messages. The golden file `specs/001-daily-chat-digest/contracts/digest-cases.json` is the automated form of this test.

### Tests for User Story 1 ⚠️

> **NOTE: Write these tests FIRST, ensure they FAIL before implementation**

- [ ] T017 [P] [US1] Add a schema contract test in `backend/test/contract/digest-schemas.spec.ts` that validates sample job and result bodies against `specs/001-daily-chat-digest/contracts/digest-job.schema.json` and `specs/001-daily-chat-digest/contracts/digest-result.schema.json`. Enforce `additionalProperties: false`; job `messages` `maxItems` 2000; message `text` `minLength` 1 `maxLength` 4096; `author_label` `minLength` 1 `maxLength` 128; `author_user_id` pattern `^[0-9]+$`; `chat_id` pattern `^-?[0-9]+$`; `v` const 1; `narrative_ru` `maxLength` 4000; `work_log` `minItems` 1 `maxItems` 10 with `action` enum `extract_events`; `error_code` enum `model_timeout`, `model_unavailable`, `model_rejected`, `invalid_job`, `bad_signature`; `usage.model` const `qwen3.5-4b-q4_k_m`. Status `ok` requires `narrative_ru`, `event_count`, `context_truncated`, and `usage`, with `error_code` null. Status `error` requires a string `error_code` and `narrative_ru` null.
- [ ] T018 [P] [US1] Add `backend/test/digest/golden.spec.ts` driving all 24 cases in `specs/001-daily-chat-digest/contracts/digest-cases.json` with the clock pinned to `2026-10-27 00:00 Europe/Minsk` and a stub narrative. Case ids: `family-and-one-silent-day`, `health-only`, `relationship-only`, `move-only`, `job-only`, `achievement-only`, `plans-jokes-greetings-no-recap`, `small-talk-and-first-silent-day`, `second-silent-day-is-long`, `fifth-silent-day-plural`, `forward-and-caption-do-not-count-as-typed`, `reply-kept-duplicate-forward-once`, `same-event-typed-three-times`, `five-people-grouped`, `partial-day-does-not-change-counts`, `unreadable-day-is-failure`, `service-notice-plus-real-event`, `typing-resets-a-long-run`, `no-earlier-mark-starts-at-one`, `news-plus-long-silence`, `retry-does-not-double-count`, `english-fact-russian-recap`, `bot-message-ignored`, `mixed-combination`. For every case assert: a job is pushed only when `enqueue_model` is true and the body contains only `text` and `reply` messages from members other than `bot_user_id` `900`; `publish: recap` sends the stub narrative, then silence lines, then partial lines; `publish: nothing` sends no Telegram message; `publish: failure` sends exactly `Не удалось подготовить выжимку за этот день.` and no member name; `silence` and `visible_without_count` match `specs/001-daily-chat-digest/contracts/telegram-copy.md`, including `полный день`, `полных дня`, `полных дней`, and `Молчит уже давно.` only when `long` is true; the partial-day sentence is present only when `partial_notice` is true; `count_after` matches `users.consecutive_silent_days` and a second evaluation does not change it; PostgreSQL contains no message text and no `stub_narrative_ru`; `logs.action` stays on the allowlist. The test must fail before the digest implementation exists.
- [ ] T019 [P] [US1] Add `backend/test/digest/attempt-rules.spec.ts` for the rules outside the golden assertions: a webhook with the wrong secret writes nothing; a result with a bad signature inserts `rejected_result` and is not published as a recap; `context_truncated: true` adds `Остальная часть дня была не видна.`; attempt 2 runs 15 minutes after attempt 1 and attempt 3 runs 30 minutes after attempt 1; at most 3 attempts, all inside the same new Minsk day; a settled run (recap published, empty day skipped, or three failures) sends nothing further; a later success after the one failure note posts the recap once and does not post a second failure note; lateness alone does not post a failure note; after the new Minsk day ends, that recap day is not published.
- [ ] T020 [P] [US1] Add `ai/tests/test_retrieval.py` with a stub embedder. For each golden case with `enqueue_model: true`, the retrieval input is eligible member text only (no forwards, captions, service text, or bot messages). Every `retrieved_texts` entry is kept and every `ignored_texts` entry is dropped. A message scores the maximum cosine similarity of its chunks against the six query vectors and is kept when that score is greater than or equal to `RAG_SCORE_THRESHOLD` (starting value `0.35`). After the threshold, keep at most 40 messages, highest scores first. A message longer than 512 tokens is split into windows of 512 with an overlap of 64 tokens, every window keeps the same author, and retrieval returns the parent message once. If no message meets the threshold, the generator is not called, `narrative_ru` is null, `event_count` is 0, and `usage.output_tokens` is 0.
- [ ] T021 [P] [US1] Add `ai/tests/test_worker.py` with a stub generator. The generator runs only when retrieval kept at least one message, and its prompt contains those messages and no ignored message. Thinking is off, the output cap is 800 tokens, temperature is `0.2`, and `usage.model` is `qwen3.5-4b-q4_k_m`. `work_log` validates against `specs/001-daily-chat-digest/contracts/digest-result.schema.json` and contains no message text and no `narrative_ru`. A second job with the same `job_id` does not embed or generate again. A job that fails the signature or the schema returns `status: error`, `error_code` `invalid_job` or `bad_signature`, and `narrative_ru: null`. A `<think>` block is stripped and never becomes `narrative_ru`; if nothing remains, the result is `model_rejected` and `narrative_ru` is null. A job that exceeds 90 seconds returns `model_timeout`. The worker opens no connection to a model host and no listening socket.
- [ ] T022 [P] [US1] Add `ai/tests/test_local_eval.py` marked `local_eval`. It loads `BAAI/bge-m3` from `/models/embedder` and `Qwen3.5-4B` GGUF `Q4_K_M` from `/models/generator.gguf`. Retrieval must match `retrieved_texts` and `ignored_texts` on every case. Score generation only when `distinct_events` is greater than 0. A case passes when the narrative contains every `must_mention` phrase and none of the `must_not_mention` phrases. The run passes when retrieval matches all cases, at least 90% of the scored cases pass, and no scored case contains a forbidden phrase. Silence lines are not part of this score. Do not select this mark in the default run. If it fails, the only allowed model change is replacing the GGUF with `Qwen3.5-9B` `Q4_K_M` in the same image and recording that run. Do not point the worker at a hosted API.

### Implementation for User Story 1

- [ ] T023 [P] [US1] Add the User entity in `backend/src/users/user.entity.ts` and migration `backend/src/database/migrations/1740000000003-create-users.ts` for table `users`: `id` uuid primary key, generated; `telegram_chat_id` bigint not null referencing `chats.telegram_chat_id`; `telegram_user_id` bigint not null, unique together with `telegram_chat_id`; `display_name` text not null (Telegram first and last name, otherwise username, otherwise `участник` plus the id; updated when Telegram sends a newer name; used in the recap, not in `logs`); `in_chat` boolean not null; `present_since` timestamptz not null; `left_at` timestamptz null (set when the member leaves; cleared on rejoin by setting `in_chat` true and refreshing `present_since`); `consecutive_silent_days` integer not null default 0, never negative; `silence_counted_through` date null (null means no day has been applied); `updated_at` timestamptz not null. Never insert the bot's user id.
- [ ] T024 [P] [US1] Add the DigestRun entity in `backend/src/digest/digest-run.entity.ts` and migration `backend/src/database/migrations/1740000000004-create-digest-runs.ts` for table `digest_runs`: `id` uuid primary key; `telegram_chat_id` bigint not null referencing `chats.telegram_chat_id`; `recap_day` date not null (the Minsk day being described); `attempt_count` integer not null default 0, values 0 through 3; `recap_published` boolean not null default false; `failure_published` boolean not null default false; `settled` boolean not null default false (true when the recap was published, the day was an empty skip, attempts reached 3, or the new Minsk day ended); `updated_at` timestamptz not null. Unique (`telegram_chat_id`, `recap_day`). No narrative column and no message-text column.
- [ ] T025 [P] [US1] Implement retrieval in `ai/src/rag.py`. Embed these six queries once at process start and reuse them: family `семейное событие: рождение ребёнка, свадьба, родственники`; health `здоровье: болезнь, больница, операция, выздоровление`; relationship `личные отношения: помирились, расстались, начали встречаться`; move `переезд в другой город или в новое жильё`; job `работа: новая работа, увольнение, повышение`; achievement `личное достижение: диплом, победа, награда`. Embed chunks with local `BAAI/bge-m3` from `/models/embedder`. Keep a message when its max cosine similarity is greater than or equal to `RAG_SCORE_THRESHOLD` (starting value `0.35`), then at most 40, highest scores first, in original order. Set `context_truncated` true only when eligible messages were dropped because of that cap or because embedding stopped at its time limit. Leaving out small talk below the threshold is not truncation. Build the index in memory and drop it before returning. Do not write embeddings to Redis or PostgreSQL. Run with `HF_HUB_OFFLINE=1` and do not open a connection to a model host.
- [ ] T026 [P] [US1] Implement generation in `ai/src/generate.py`. Load `Qwen3.5-4B` GGUF `Q4_K_M` from `/models/generator.gguf`. Thinking off, temperature `0.2`, at most 800 output tokens, one generation at a time. The system prompt states, in English: write Russian prose centered on people; include only notable personal events (family, health, relationships, moves, jobs, and achievements, and who was involved); leave out plans, jokes, greetings, and small talk; tell each distinct event once and group events about the same person; do not add silence notes, counts, or a remark about missing messages; if nothing qualifies, return an empty narrative; do not invent events or people who are not in the input. Pass only the retrieved messages and their author labels. Strip any `<think>` block before setting `narrative_ru`. If nothing remains, return `model_rejected` and `narrative_ru` null. `narrative_ru` is at most 4000 characters. `usage.model` is `qwen3.5-4b-q4_k_m`. A job that exceeds 90 seconds returns `model_timeout`. If retrieval kept nothing, do not generate: `narrative_ru` null, `event_count` 0, `usage.output_tokens` 0, and that is a successful extraction.
- [ ] T027 [P] [US1] Implement Russian composition in `backend/src/digest/compose.ts` from `specs/001-daily-chat-digest/contracts/telegram-copy.md`. Order: non-empty `narrative_ru`; one blank line when both a narrative and a following section exist; full-day silence lines in display-name order; partial-visibility member lines in display-name order; the partial-day sentence when the day was not fully visible and something was read, and also when `context_truncated` is true. Unit for count `n`: `n % 10 == 1` and `n % 100 != 11` → `полный день`; `n % 10` in `2, 3, 4` and `n % 100` not in `12, 13, 14` → `полных дня`; otherwise `полных дней`. Line for `n == 1`: `{label}: 1 полный день без своих сообщений.` Line for `n >= 2`: `{label}: {n} {unit} без своих сообщений подряд. Молчит уже давно.` Partial member line: `{label}: в видимой части дня своих сообщений не было.` Partial sentence: `Остальная часть дня была не видна.` Failure sentence: `Не удалось подготовить выжимку за этот день.` A fully visible day with an empty narrative and no silence lines produces no message. Split on line boundaries at 3500 characters.
- [ ] T028 [P] [US1] Implement `backend/src/digest/result.consumer.ts`. Wait up to 120 seconds for a result with the same `job_id`. A timeout is attempt failure `model_timeout`. Ignore a result for a job that is no longer open and do not store its narrative. After a constant-time signature check and schema validation, insert each `work_log` item into `logs` (`action` `extract_events`, `source` `ai`, allowlisted `error_code` only). An invalid result inserts `rejected_result` and fails the attempt. Do not insert `narrative_ru` or `usage` into PostgreSQL. Document each public method in English with its purpose and contract.
- [ ] T029 [US1] Implement `backend/src/users/users.module.ts` and `backend/src/users/users.service.ts`. Create or refresh a row from a `chat_member` update or a member message. A member is on the roster for Minsk day D when `present_since` is before the end of D and (`left_at` is null or `left_at` is at or after the start of D). Apply the silence transition only when D was fully readable and `silence_counted_through` is null or earlier than D: eligible own text or a reply sets `consecutive_silent_days` to 0 and `silence_counted_through` to D; no eligible text sets `consecutive_silent_days` to previous + 1 and `silence_counted_through` to D. A forward, a media caption, a service notice, or the bot's own message is not eligible text. If D was only partly readable, or not readable, both fields stay as they were. A second pass for the same D does not change them. `consecutive_silent_days` is written only by this transition. The bot's user id is never a silence subject. Depends on T023.
- [ ] T030 [US1] Implement `backend/src/digest/buffer.service.ts`. Store that day's envelopes in encrypted `buffer:{chat}:{minsk_day}`. Keep forwards, captions, service notices, and bot messages only as not-typed markers, and do not copy their text into a job. An `edited_message` replaces that message's buffered text. Truncate message text longer than 4096 characters and mark the day partly visible. Set the TTL at the end of the next Minsk day. Delete the buffer when the run settles. The AI service never reads `buffer:*`. Depends on T013 and T029.
- [ ] T031 [US1] Implement `backend/src/digest/job.builder.ts`. Push one signed job to `digest:jobs` only when at least one eligible message remains. Validate the plaintext against `specs/001-daily-chat-digest/contracts/digest-job.schema.json` before sealing: `v` const 1, `job_id` uuid stable for one chat, recap day, and attempt, `messages` `maxItems` 2000, each `text` `minLength` 1 `maxLength` 4096, `author_label` `minLength` 1 `maxLength` 128. The job contains only `text` and `reply` from members other than the bot. If no eligible text exists, do not push a job. Depends on T030.
- [ ] T032 [US1] Implement `ai/src/worker.py`. Pop `digest:jobs`, verify the signature, decrypt, then validate the schema. An invalid job returns `status: error`, `error_code` `invalid_job` or `bad_signature`, `narrative_ru: null`, and a `work_log` entry (`action` `extract_events`, `minItems` 1, `maxItems` 10) that contains no source text. Answer a repeated `job_id` from encrypted `digest:cache:{job_id}` with the same TTL as the buffer, and do not embed or generate again. Push the signed result to `digest:results`. Refresh `ai:heartbeat` with a timestamp only. Open no listening socket and call no model host. Further jobs stay on `digest:jobs` while one generation is running. Depends on T009, T025, and T026.
- [ ] T033 [US1] Implement `backend/src/digest/digest.service.ts` and `backend/src/digest/digest.scheduler.ts`. The scheduler fires at 00:00 `Europe/Minsk` for the previous Minsk day. Attempt 1 runs then; a failed attempt retries at +15 minutes and +30 minutes, at most 3 times, and only during that new Minsk day. Apply these run transitions: eligible text with model and send success sets `recap_published` and `settled`; no events and no silence lines on a fully visible day sets `settled` without a Telegram recap and logs `skip_empty`; no events with silence lines or a partial notice, when send succeeds, sets `recap_published` and `settled`; an attempt that cannot produce a recap, when no failure note exists yet, sets `failure_published` and increments `attempt_count`; a later attempt that produces the recap sets `recap_published` and `settled` and does not post a second failure note; a later trigger after the recap was sent sends nothing; `attempt_count` reaching 3 without a recap sets `settled`; the new Minsk day ending sets `settled` even if attempts remain. A missing `ai:heartbeat` or a model error uses `model_unavailable` or the result `error_code` and the one failure sentence `Не удалось подготовить выжимку за этот день.` Lateness alone does not post that sentence. Discard the narrative after send. Delete the buffer and the result cache when the run settles. Log `collect_day`, `publish_recap`, `publish_failure`, `skip_empty`, and `rejected_result` with `attempt_number` 1–3 when the action is an attempt. Do not await a model call inside the webhook handler. Depends on T024, T027, T028, T029, T031, and T032.
- [ ] T034 [US1] Add `backend/src/digest/digest.module.ts` and register it in `backend/src/app.module.ts`. Route `message`, `edited_message`, and `chat_member` from `backend/src/telegram/webhook.controller.ts` into the buffer and `UsersService`. `DigestModule` decides and composes by calling `TelegramModule`, `QueueModule`, `UsersModule`, and `LogsModule`. It does not embed Bot API calls or SQL for another module's tables. Depends on T015 and T033.
- [ ] T035 [US1] Run `npm test --prefix backend -- --testPathPattern=digest` and `python -m pytest ai/tests/test_retrieval.py ai/tests/test_worker.py -m "not local_eval"` and fix `backend/src/digest/` and `ai/src/` until those User Story 1 tests pass. Leave `local_eval` unselected. Do not load the GGUF on this run and do not point the worker at a hosted model.

**Checkpoint**: User Story 1 is independently functional. A joined chat can receive the midnight recap, the one failure sentence, or a deliberate empty skip, and PostgreSQL still has no message text and no narrative.

---

## Phase 4: User Story 2 - Read what the bot did (Priority: P2)

**Goal**: A member of a joined chat sends `/logs` and receives, in that same chat, a Russian list of every recorded action for this feature, successful and unsuccessful, with no message text and no recap body.

**Independent Test**: Insert one successful `publish_recap` and one failed attempt for a chat, send `/logs` there as a member, and confirm the Russian reply lists both outcomes and contains no message text. A `/logs` from a private chat or a non-member returns the refusal and no rows.

### Tests for User Story 2 ⚠️

> **NOTE: Write these tests FIRST, ensure they FAIL before implementation**

- [ ] T036 [P] [US2] Add `backend/test/logs/logs.command.spec.ts`. Seed a successful `publish_recap` and a failed attempt in one chat and another chat's row. `/logs` in the first chat replies in Russian with both of that chat's actions and continues in the same chat when the list passes 3500 characters, split on line boundaries. Each line is `{label} — {recap_day} — {outcome} — попытка {attempt_number}`, with `outcome` `успех` when `success` is true and `неудача` otherwise. Omit the attempt segment when `attempt_number` is null. Map `collect_day` to `Сбор дня`, `extract_events` to `Подготовка выжимки`, `publish_recap` to `Публикация выжимки`, `publish_failure` to `Сообщение об ошибке`, `skip_empty` to `Пустой день`, `show_logs` to `Просмотр журнала`, and `rejected_result` to `Отклонение ответа`. Order rows by `created_at` ascending. The reply contains no message text and no `narrative_ru`. No rows yields `Пока ничего не записано.` A read failure yields `Не удалось показать журнал.` A private chat, a chat the bot has not joined, or a sender `getChatMember` does not report as `creator`, `administrator`, `member`, or `restricted` yields `Эта команда работает только для участника чата, в котором состоит бот.` and no rows. A sixth `/logs` from the same sender within 10 minutes yields `Слишком много запросов. Попробуйте чуть позже.` and writes no log row. The `show_logs` row for the request being answered is inserted after the reply is sent, so it appears on the next `/logs`. The test must fail before the command exists.

### Implementation for User Story 2

- [ ] T037 [US2] Implement `/logs` in `backend/src/logs/logs.service.ts` and `backend/src/telegram/logs.update.ts`. Enforce the rate limit of 5 requests per sender per 10 minutes before reading `logs`. Reply in the same chat using the templates and action labels in `specs/001-daily-chat-digest/contracts/telegram-copy.md`. Read only that chat's rows. On a read failure, reply `Не удалось показать журнал.` and do not stay silent. Insert `show_logs` only after a successful send. Document each public method in English with its purpose and contract. Depends on T012, T015, and T036.

**Checkpoint**: User Stories 1 and 2 both work. `/logs` lists the actions User Story 1 recorded, and it can also be tested from seeded rows alone.

---

## Phase 5: Polish & Cross-Cutting Concerns

**Purpose**: Operator documentation and the quickstart path across both stories

- [ ] T038 [P] Document the running system in English in `README.md`: the two services, the environment variables from `specs/001-daily-chat-digest/quickstart.md`, `docker compose up -d postgres redis`, `npm ci --prefix backend`, `pip install -e ai`, the default test commands, and that `python -m pytest ai/tests -m local_eval` is the SC-002 check and is not part of the default run. State that user-facing Telegram copy is Russian and that PostgreSQL stores no message text and no recap narrative.
- [ ] T039 [P] Add `backend/test/digest/quickstart-path.spec.ts` for the quickstart end-to-end path with the embedder and generator stubbed and the clock pinned to `2026-10-27 00:00 Europe/Minsk`. Mark a chat `bot_is_member`, insert two users, push one eligible Russian message and one forward, run the midnight handler, and confirm one Russian recap, the silent member's stored count is 1, `logs` has a successful `publish_recap`, and neither the message nor the recap body is in PostgreSQL. Run the handler again and confirm nothing new is posted and the count is unchanged. Send `/logs` in that chat as a member and confirm the Russian list has no recap body. Send `/logs` in a private chat and confirm the refusal and that the group rows are absent.
- [ ] T040 Run the quickstart checks from the repository root and fix failures in `backend/` and `ai/` without calling Telegram or a hosted model: `python -c "import json; p='specs/001-daily-chat-digest/contracts/digest-cases.json'; d=json.load(open(p,encoding='utf-8')); ids=[c['id'] for c in d['cases']]; assert len(ids)==len(set(ids))==24; print(len(ids), 'cases')"`, `npm test --prefix backend`, and `python -m pytest ai/tests -m "not local_eval"`. Expected contract output is `24 cases`.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies. Start immediately.
- **Foundational (Phase 2)**: Depends on Setup. Blocks both user stories.
- **User Story 1 (Phase 3)**: Depends on Foundational. Does not depend on User Story 2.
- **User Story 2 (Phase 4)**: Depends on Foundational (`logs`, the webhook, and the chat gate). It does not depend on the digest pipeline; seeded `logs` rows are enough to test it. It shows live recap actions once User Story 1 writes them.
- **Polish (Phase 5)**: Depends on the user stories being present.

### User Story Dependencies

- **User Story 1 (P1)**: Starts after Phase 2. No dependency on User Story 2.
- **User Story 2 (P2)**: Starts after Phase 2. Independently testable with inserted `logs` rows.

### Within Each User Story

- Write that story's tests and confirm they fail before its implementation tasks.
- Entities before the services that write them.
- `UsersService` (T029) before the buffer (T030). The buffer before the job builder (T031).
- Retrieval (T025) and generation (T026) before the worker (T032).
- Composer (T027) and result consumer (T028) before the digest service (T033).
- Digest service before webhook wiring (T034).

### Parallel Opportunities

- Phase 1: T002, T003, T004, and T005 can run with T001 (separate trees).
- Phase 2: T006 through T014 can run together. T015 waits for the chat entity, crypto, and Redis. T016 waits for T015.
- User Story 1 tests T017 through T022 can run together.
- After those tests are written: T023, T024, T025, T026, T027, and T028 can run together.
- User Story 2's test (T036) can run in parallel with User Story 1 once Phase 2 is complete.
- Polish: T038 and T039 can run together. T040 runs last.

---

## Parallel Example: User Story 1

```bash
# Write the failing User Story 1 tests together:
# T017 backend/test/contract/digest-schemas.spec.ts
# T018 backend/test/digest/golden.spec.ts
# T019 backend/test/digest/attempt-rules.spec.ts
# T020 ai/tests/test_retrieval.py
# T021 ai/tests/test_worker.py
# T022 ai/tests/test_local_eval.py

# Then implement the independent pieces together:
# T023 backend/src/users/user.entity.ts
# T024 backend/src/digest/digest-run.entity.ts
# T025 ai/src/rag.py
# T026 ai/src/generate.py
# T027 backend/src/digest/compose.ts
# T028 backend/src/digest/result.consumer.ts
```

## Parallel Example: User Story 2

```bash
# User Story 2 has one test file, then one implementation step.
# T036 can be written while User Story 1 implementation is in progress:
# T036 backend/test/logs/logs.command.spec.ts
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup.
2. Complete Phase 2: Foundational (blocks every story).
3. Complete Phase 3: User Story 1 tests, then implementation, then T035.
4. **STOP and VALIDATE**: Run the golden digest suite and the stubbed AI suite. Confirm a fully visible day with one typed personal event and one silent member publishes one Russian recap, and that PostgreSQL has no message text and no narrative.
5. Demo the midnight recap before starting `/logs`.

### Incremental Delivery

1. Setup + Foundational → chat gate, crypto, and `logs` storage.
2. User Story 1 → midnight recap, silence counts, and the one failure sentence (MVP).
3. User Story 2 → `/logs` for that chat's actions.
4. Polish → README and the quickstart path.

### Parallel Team Strategy

1. The team finishes Setup and Foundational together.
2. After Phase 2:
   - One person: User Story 1 backend (`users`, buffer, compose, digest).
   - Another person: User Story 1 AI worker (`rag`, `generate`, `worker`).
   - A third person: User Story 2 `/logs`, using seeded rows.
3. The stories meet when User Story 1 writes the `logs` rows User Story 2 already knows how to render.

---

## Notes

- [P] tasks use different files and do not depend on incomplete tasks in the same phase.
- [US1] and [US2] map to the two stories in `specs/001-daily-chat-digest/spec.md`.
- Default tests stub `bge-m3` and the GGUF generator. `local_eval` is the 90% retell check (SC-002) and stays out of the default run.
- The day boundary in every test that cares about "today" is `2026-10-27 00:00 Europe/Minsk`, so `recap_day` is `2026-10-26`.
- Commit after each task or logical group. Stop at the checkpoints and validate that story alone.
