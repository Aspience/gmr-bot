# Data Model: Daily Chat Digest

PostgreSQL is owned by `backend`. `ai` has no database. Message text and the recap narrative are not columns in any table.

Telegram ids are stored as `bigint`. Dates that bound the recap are Minsk calendar dates (`date`), not timestamps.

## Chat

A Telegram group the bot has joined. One recap belongs to one chat.

| Field | Type | Rules |
| --- | --- | --- |
| `telegram_chat_id` | bigint, primary key | Required |
| `bot_is_member` | boolean, not null | True only while the bot status is `member` or `administrator`. False after `left` or `kicked`. |
| `updated_at` | timestamptz, not null | Set on every membership change |

No chat title is stored. Updates for a chat with `bot_is_member = false` are ignored.

## User

One person in one chat. The bot's user id is never inserted. A row is created when a `chat_member` update or a member message shows that person in the chat.

| Field | Type | Rules |
| --- | --- | --- |
| `id` | uuid, primary key | Generated |
| `telegram_chat_id` | bigint, not null | References `chats.telegram_chat_id` |
| `telegram_user_id` | bigint, not null | Unique together with `telegram_chat_id` |
| `display_name` | text, not null | Telegram first and last name, otherwise username, otherwise `участник` plus the id. Updated when Telegram sends a newer name. Used in the recap, not in `logs`. |
| `in_chat` | boolean, not null | Current membership |
| `present_since` | timestamptz, not null | Start of the current membership observation |
| `left_at` | timestamptz, null | Set when the member leaves; cleared on rejoin by setting `in_chat` true and refreshing `present_since` |
| `consecutive_silent_days` | integer, not null, default 0 | Stored absence count. Never negative. |
| `silence_counted_through` | date, null | Last fully visible Minsk day already applied to the count. Null means no day has been applied. |
| `updated_at` | timestamptz, not null | |

A member is in the recap roster for Minsk day D when `present_since` is before the end of D and (`left_at` is null or `left_at` is at or after the start of D).

### Silence transition

Applied only when day D was fully readable, and only if `silence_counted_through` is null or earlier than D.

| Eligible text from this member on D | New `consecutive_silent_days` | `silence_counted_through` |
| --- | --- | --- |
| Yes (own text or reply) | 0 | D |
| No | previous + 1 | D |

A forward, a media caption, a service notice, or the bot's own message is not eligible text.

If D was only partly readable, or not readable, both fields stay as they were. A second pass for the same D does not change them.

The recap states the stored count after this transition. The long-silence sentence is added only when that count is 2 or more.

## Digest run

Publication state for one chat and one recap day. Holds no narrative and no message text.

| Field | Type | Rules |
| --- | --- | --- |
| `id` | uuid, primary key | |
| `telegram_chat_id` | bigint, not null | |
| `recap_day` | date, not null | The Minsk day being described (the day that just ended) |
| `attempt_count` | integer, not null, default 0 | 0 through 3 |
| `recap_published` | boolean, not null, default false | True after the recap is sent |
| `failure_published` | boolean, not null, default false | True after the one failure note is sent |
| `settled` | boolean, not null, default false | True when the recap was published, the day was an empty skip, attempts reached 3, or the new Minsk day ended |
| `updated_at` | timestamptz, not null | |

Unique (`telegram_chat_id`, `recap_day`).

### Run transitions

| State | Event | Next |
| --- | --- | --- |
| Open, attempts < 3, new Minsk day | Eligible text exists, model and send succeed | `recap_published`, `settled` |
| Open | No events, no silence lines, fully visible | `settled` without a Telegram recap |
| Open | No events, silence lines or a partial notice, send succeeds | `recap_published`, `settled` |
| Open, no failure note yet | Attempt cannot produce a recap | `failure_published`, `attempt_count + 1` |
| Failure note already sent | Later attempt produces the recap | `recap_published`, `settled`; no second failure note |
| Recap already sent | Later trigger | No send |
| `attempt_count` reaches 3 without a recap | Third failure | `settled` |
| New Minsk day ends | Window closes | `settled` even if attempts remain |

## Log

One action, successful or not. `/logs` reads this table for the current chat only.

| Field | Type | Rules |
| --- | --- | --- |
| `id` | uuid, primary key | |
| `telegram_chat_id` | bigint, not null | |
| `action` | text, not null | Allowlist: `collect_day`, `extract_events`, `publish_recap`, `publish_failure`, `skip_empty`, `show_logs`, `rejected_result` |
| `recap_day` | date, not null | |
| `success` | boolean, not null | |
| `attempt_number` | integer, null | 1–3 when the action is an attempt; null otherwise |
| `source` | text, not null | `backend` or `ai` |
| `error_code` | text, null | Allowlist in `contracts/digest-result.schema.json`. No free text. |
| `created_at` | timestamptz, not null | |

Index (`telegram_chat_id`, `created_at`). No column exists for message text, display names, or the recap narrative. AI work-log entries are inserted here by the backend after signature checks. An invalid result inserts `rejected_result` and does not insert a narrative.

## Redis structures

These are not PostgreSQL tables. They are the only place message text and the narrative exist before publish.

| Key | Writer | Reader | Contents | Lifetime |
| --- | --- | --- | --- | --- |
| `buffer:{chat}:{minsk_day}` | backend | backend | Encrypted eligible and not-yet-filtered message envelopes for that day | Deleted when the run settles. TTL at the end of the next Minsk day. |
| `digest:jobs` | backend | ai | Encrypted, signed extraction job | Consumed by the worker |
| `digest:results` | ai | backend | Encrypted, signed result | Consumed by the backend |
| `digest:cache:{job_id}` | ai | ai | Encrypted prior result for that job | Same TTL as the buffer. A cache hit does not call the model. |
| `ai:heartbeat` | ai | backend | Timestamp only | Short TTL, refreshed by the worker |

The AI service never reads `buffer:*`. The backend copies eligible text into a job.

The retrieval index is not one of these keys. The worker builds it in memory from the job and drops it before returning. Embeddings are not written to Redis or PostgreSQL.

## Relationships

```text
chats 1──* users
chats 1──* digest_runs
chats 1──* logs
```

`users.telegram_chat_id`, `digest_runs.telegram_chat_id`, and `logs.telegram_chat_id` reference `chats.telegram_chat_id`.

## Validation

- Every inbound Telegram update is checked against an allowlist of update types (`message`, `edited_message`, `chat_member`, `my_chat_member`) before use.
- Message text longer than 4096 characters is truncated for the buffer and the day is marked partly visible.
- Job and result JSON must validate against `contracts/digest-job.schema.json` and `contracts/digest-result.schema.json` after decryption. Invalid JSON is rejected.
- `consecutive_silent_days` is written only by the transition above.
- `logs.action`, `logs.source`, and `logs.error_code` reject values outside their allowlists.
