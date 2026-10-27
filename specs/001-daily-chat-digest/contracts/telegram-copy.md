# Telegram copy contract

User-facing text for this feature is Russian. Tests compare against these templates. Internal codes stay English; this file is the only mapping.

## Recap composition order

1. `narrative_ru` from the extraction result, when it is non-empty.
2. One blank line, when both a narrative and a following section exist.
3. Full-day silence lines, in display-name order.
4. Partial-visibility member lines, in display-name order.
5. The partial-day sentence, when the day was not fully visible and something was read.

A fully visible day with an empty narrative and no silence lines sends no message.

Messages are split on line boundaries at 3500 characters and sent, in order, to the same chat. Telegram's 4096-character limit stays above that cut.

## Silence lines

Let `n` be the stored `consecutive_silent_days` after the day's transition.

Unit for `n`:

- `n % 10 == 1` and `n % 100 != 11` → `полный день`
- `n % 10` in `2, 3, 4` and `n % 100` not in `12, 13, 14` → `полных дня`
- otherwise → `полных дней`

| Condition | Line |
| --- | --- |
| `n == 1` | `{label}: 1 полный день без своих сообщений.` |
| `n >= 2` | `{label}: {n} {unit} без своих сообщений подряд. Молчит уже давно.` |

`{label}` is the member `display_name`.

A member with no eligible text in the visible part of a partly read day is not given the lines above. They get:

`{label}: в видимой части дня своих сообщений не было.`

## Partial day and failures

| Situation | Text |
| --- | --- |
| Part of the day was not readable | `Остальная часть дня была не видна.` |
| An attempt cannot produce the recap, and no failure note exists yet | `Не удалось подготовить выжимку за этот день.` |
| `/logs` has no rows for this chat | `Пока ничего не записано.` |
| `/logs` cannot be read | `Не удалось показать журнал.` |
| Sender or chat is not allowed | `Эта команда работает только для участника чата, в котором состоит бот.` |
| `/logs` rate limit | `Слишком много запросов. Попробуйте чуть позже.` |

The failure note is posted at most once per chat per recap day. A later successful attempt posts the recap and does not post a second failure note.

## `/logs` lines

One log row becomes one line:

`{label} — {recap_day} — {outcome} — попытка {attempt_number}`

`outcome` is `успех` when `success` is true, otherwise `неудача`. The attempt segment is omitted when `attempt_number` is null.

| `action` | `{label}` |
| --- | --- |
| `collect_day` | Сбор дня |
| `extract_events` | Подготовка выжимки |
| `publish_recap` | Публикация выжимки |
| `publish_failure` | Сообщение об ошибке |
| `skip_empty` | Пустой день |
| `show_logs` | Просмотр журнала |
| `rejected_result` | Отклонение ответа |

Rows are ordered by `created_at` ascending. Rows from other chats are not included. The reply does not contain message text or `narrative_ru`.

The `show_logs` row for the request being answered is inserted after the reply is sent, so it appears on the next `/logs`.

## Extraction instructions

The generator system prompt states all of the following, in English. The prompt's messages are only the lines retrieval kept:

- Write Russian prose centered on people.
- Include only notable personal events: family, health, relationships, moves, jobs, and achievements, and who was involved.
- Leave out plans, jokes, greetings, and small talk.
- Tell each distinct event once. Group events about the same person.
- Do not add silence notes, counts, or a remark about missing messages.
- If nothing qualifies, return an empty narrative.
- Do not invent events or people who are not in the input.

The backend, not the generator, appends silence lines and the partial-day sentence. A true `context_truncated` flag on the result forces that partial-day sentence even when the backend itself could read the day.
