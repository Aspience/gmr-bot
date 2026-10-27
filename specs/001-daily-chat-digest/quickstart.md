# Quickstart: Daily Chat Digest

Validation guide for the two-service recap. Implementation details belong in the tasks that follow this plan. Cases and copy live in the contracts; this file only says how to prove them.

## Prerequisites

- Node.js 22 and Python 3.12
- PostgreSQL 16 and Redis 7, private to the machine
- Environment variables, none of them committed:

| Name | Used by |
| --- | --- |
| `TELEGRAM_BOT_TOKEN` | backend |
| `TELEGRAM_WEBHOOK_SECRET` | backend |
| `DATABASE_URL` | backend |
| `REDIS_URL` | both |
| `REDIS_PASSWORD` | both |
| `REDIS_PAYLOAD_KEY` | both, 32 bytes, base64 |
| `INTERNAL_HMAC_KEY` | both |
| `RAG_SCORE_THRESHOLD` | ai, calibrated value; starting point `0.35` |

The AI image already contains `/models/embedder` (`BAAI/bge-m3`) and `/models/generator.gguf` (`Qwen3.5-4B` `Q4_K_M`). The running container sets `HF_HUB_OFFLINE=1`. There is no model API key and no model base URL.

The default test commands do not call Telegram and do not load the local weights.

## Setup

From the repository root, after the services exist:

```bash
docker compose up -d postgres redis
```

Apply the backend migrations, then install dependencies:

```bash
npm ci --prefix backend
pip install -e ai
```

## Contract checks

Confirm the golden file loads and every case id is unique:

```bash
python -c "import json; p='specs/001-daily-chat-digest/contracts/digest-cases.json'; d=json.load(open(p,encoding='utf-8')); ids=[c['id'] for c in d['cases']]; assert len(ids)==len(set(ids))==24; print(len(ids), 'cases')"
```

Expected: `24 cases`.

## Backend suite

```bash
npm test --prefix backend
```

The suite reads `specs/001-daily-chat-digest/contracts/digest-cases.json` and `contracts/telegram-copy.md` templates.

For every case it must show:

| Expectation | Result |
| --- | --- |
| `enqueue_model` | A job is pushed only when true, and the job body contains only `text` and `reply` messages from members other than `bot_user_id`. |
| `publish: recap` | The chat receives the composed Russian message: stub narrative, then silence lines, then partial lines. |
| `publish: nothing` | No Telegram message. |
| `publish: failure` | Exactly the failure sentence, with no member name. |
| `silence` and `visible_without_count` | Lines match the templates, including `полный день`, `полных дня`, `полных дней`, and the long-silence sentence only when `long` is true. |
| `partial_notice` | The partial-day sentence is present only when true. |
| `count_after` | `users.consecutive_silent_days` matches, and a second evaluation of the same case does not change it. |
| Persistence | PostgreSQL contains no message text and no `stub_narrative_ru`. `logs` rows use the action allowlist only. |

Additional cases outside the golden file:

- Webhook with a wrong secret is rejected and writes nothing.
- `/logs` from a private chat or a non-member returns the refusal and no other chat's rows.
- A result with a bad signature is stored as `rejected_result` and is not published as a recap.
- A result with `context_truncated: true` adds the partial-day sentence.

## AI suite

```bash
python -m pytest ai/tests -m "not local_eval"
```

Using the same golden file, a stub embedder, and a stub generator, each case with `enqueue_model: true` must show:

- The retrieval input is eligible text only. Forwards, captions, service text, and the bot message are absent.
- Every `retrieved_texts` entry is kept and every `ignored_texts` entry is dropped.
- The generator is called only when retrieval kept at least one message, and its prompt contains those messages and no ignored message.
- Thinking is off, the output cap is 800 tokens, and `usage.model` is `qwen3.5-4b-q4_k_m`.
- The returned `work_log` validates against `contracts/digest-result.schema.json` and does not contain message text or `narrative_ru`.
- A second job with the same `job_id` does not embed or generate again.
- The worker opens no connection to a model host.

Cases with `enqueue_model: false` are not sent to the worker by the backend suite. The AI suite still rejects a job that fails signature or schema checks with `narrative_ru: null`.

## Local model check

This loads the weights inside the AI image. It does not call a hosted model. Run it when the image is built or the GGUF changes, not on every commit.

```bash
python -m pytest ai/tests -m local_eval
```

Retrieval must match `retrieved_texts` and `ignored_texts` on every case. Score generation only for cases whose `distinct_events` is greater than 0. A case passes when the narrative contains every `must_mention` phrase and none of the `must_not_mention` phrases. The run passes when retrieval matches all cases, at least 90% of the scored cases pass, and no scored case contains a forbidden phrase. Silence lines are not part of this score.

If this fails, the only allowed change is to replace the GGUF with `Qwen3.5-9B` `Q4_K_M` in the same image and record the new run. Do not point the worker at a hosted API.

## End-to-end path

With both processes running against local Redis and PostgreSQL, and with the embedder and generator stubbed:

1. Mark a chat as `bot_is_member` and insert two users.
2. Push one eligible Russian message and one forward into the day's buffer.
3. Run the midnight handler for `Europe/Minsk`.
4. Confirm the chat mock received one Russian recap, the silent member's stored count is 1, `logs` has a successful `publish_recap`, and neither the message nor the recap body is in PostgreSQL.
5. Run the handler again. Confirm nothing new is posted and the count is unchanged.
6. Send `/logs` in that chat as a member. Confirm both outcomes that exist are listed in Russian and the recap body is absent.
7. Send `/logs` in a private chat. Confirm the refusal and that the group rows are absent.

## Clock

Tests that care about "today" pin the clock to `2026-10-27 00:00 Europe/Minsk`, so `recap_day` is `2026-10-26`, matching the golden file.
