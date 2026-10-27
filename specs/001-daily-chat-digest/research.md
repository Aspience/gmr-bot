# Research: Daily Chat Digest

## Service split

**Decision**: Two independently deployable services. `backend` (NestJS, PostgreSQL) collects Telegram updates, owns membership, silence counts, attempt state, and action logs, and publishes the recap. `ai` (Python) only pulls extraction jobs from Redis and returns a personal-news narrative plus a structured work log. Neither service reads or writes the other's database.

**Rationale**: This is the requested shape, and it matches the constitution rule that services couple only through a published contract. The recap body is produced for Telegram and is not a database record.

**Alternatives considered**: One NestJS process that calls the model in-process (rejected: the AI runtime, test suite, and deploy cycle must stay separate). A public HTTP API on the Python service (rejected: the service would be reachable outside the chat pipeline).

## What the model is allowed to do

**Decision**: The generator writes the Russian, person-centered narrative and sees only messages the retriever kept. The backend does the rest without a model: drop forwards, media captions, service notices, and the bot's own messages; update silence counts; add silence lines and the partial-day sentence; decide to publish, skip, or fail; retry publication.

A job is enqueued only when at least one eligible message remains. Inside that job the generator runs only when retrieval returns at least one message. A day of silence notes, a partial-day notice, an empty retrieval, or an empty day never starts generation. A repeated job id reuses the Redis result and does not embed or generate again.

**Rationale**: Silence counts, message eligibility, attempt limits, and the partial-day sentence are deterministic. Retrieval, not the generator, decides which typed lines are personal news. Principle IX still applies to a local model: do not spend a generation when a cheaper step already knows there is nothing to write.

**Alternatives considered**: Sending the whole day, including forwards and the member roster, straight to the generator (rejected: the generator would re-read small talk, and a silence count would depend on the model). A hosted chat-completions API (rejected: the model must run inside the AI container).

## Retrieval

**Decision**: In-process RAG over the current job only, specified in `contracts/rag.md`. Each eligible message is a chunk. `BAAI/bge-m3` embeds the chunks and six fixed Russian category queries. Messages at or above the calibrated cosine threshold are the generator context, capped at 40. The index is memory inside the worker and is dropped when the job finishes. No vector database, no `pgvector`, and no Redis key holds embeddings.

**Rationale**: A chat day is a small corpus. Retrieval is what keeps plans, jokes, greetings, and small talk out of the generator. Persisting vectors would keep a lasting representation of message text, which FR-009 does not allow. The corpus cannot be a history of earlier days because that text is not retained.

**Alternatives considered**: Qdrant or `pgvector` as a standing knowledge base (rejected: it would outlive the day and store message meaning). Embedding through a hosted API (rejected: same outbound dependency as a hosted generator). Keyword rules instead of embeddings (rejected: the user asked for RAG, and Russian phrasing of personal news is not a closed word list).

## Model

**Decision**: Both models are files inside the AI image. The embedder is `BAAI/bge-m3` at `/models/embedder`. The generator is `Qwen/Qwen3.5-4B` as GGUF `Q4_K_M` (about 3 GB) at `/models/generator.gguf`, loaded with llama.cpp in the worker process. The image build uses a llama.cpp new enough for Qwen3.5 hybrid attention (b9222 or newer). Thinking is off. Temperature is 0.2. Output is capped at 800 tokens. One generation runs at a time. The job budget is 90 seconds. The container runs with `HF_HUB_OFFLINE=1` and has no model API key, no model base URL, and no published port.

If the golden local run misses 90% inclusion or includes a forbidden phrase, the only in-plan step up is swapping the GGUF for `Qwen3.5-9B` `Q4_K_M` in the same image. That swap is recorded with the eval. It is not a call to a hosted model.

**Rationale**: The acceptance bar is factual retelling of Russian personal news (SC-002). Qwen3.5-4B is the smallest instruct model in the current Qwen3.5 family that is aimed at this kind of extraction, Russian is in its training mix, and `Q4_K_M` fits a CPU container. The official weights are Apache-2.0. Loading them in-process keeps inference off the network. A 4B model can finish a short retrieved prompt inside the five-minute midnight window; retrieval is what makes the prompt short. There is no per-token bill. Principle IX is met by skipping generation when retrieval is empty, bounding output, and refusing a second run while the cached result exists.

**Alternatives considered**:

- Alibaba Model Studio, including `qwen3.7-flash` (rejected: the generator must not call an outside server).
- Any other hosted generator or embedder, including YandexGPT, GigaChat, Gemini, and OpenAI (rejected: same outbound dependency).
- `LLiserginov/Luqwen3.5-4B` (rejected: a creative Russian fine-tune whose Russian data was machine-translated and which was not preference-tuned; a weaker fit for factual extraction than the official instruct model).
- `Qwen3.5-0.8B` or `Qwen3.5-2B` as the default (rejected: below the size that can be expected to hold the 90% retell bar).
- `Qwen3.5-9B` as the default (rejected until a recorded local eval shows the 4B model misses the bar).
- Ollama as a second published service (rejected: an extra listening port; llama.cpp stays inside the worker).

## Where text lives

**Decision**: PostgreSQL stores chats, users, silence counts, digest-run state, and action logs. It never stores message text or the recap body. Eligible message text for the open Minsk day lives in a Redis buffer encrypted with AES-256-GCM. The buffer is deleted when the run settles, and its TTL ends at the close of the following Minsk day either way. The extraction result is cached in Redis under the job id, encrypted, until the run settles or that TTL fires, so a retry does not call the model again. The backend sends the narrative to Telegram and does not insert it.

**Rationale**: The product needs the day's text until the recap is produced, and principle I forbids a lasting plaintext copy. The user requirement is that the recap itself is not stored in the database. Redis is the backend's short-lived buffer, not a store the AI service scans; the AI service sees text only inside a signed job.

**Alternatives considered**: Message rows in PostgreSQL deleted after publish (rejected: a database backup would still be a lasting copy of chat text). Keeping the narrative in `logs` for debugging (rejected: FR-009 and principle VIII).

## Silence count

**Decision**: Each chat member has `consecutive_silent_days` and `silence_counted_through` on `users`. After a fully visible day is read, once per member per that date: typed eligible text sets the count to 0; no eligible text increments it by 1; `silence_counted_through` becomes that date. A later attempt on the same date does not change the count. A partly visible day does not increment or reset. A day that could not be read does not change the count. The published line uses the stored count. A count of 2 or more adds the long-silence sentence.

**Rationale**: The user asked for the absence count to be the stored value. The date column is the minimum extra state that keeps retries and partial days from corrupting that value, which FR-011 and FR-012 require. Per-day message text is not stored.

**Alternatives considered**: Recomputing the count by scanning stored messages (rejected: message text is not retained). A single integer with no date (rejected: a retry would count the same day twice).

## Member roster

**Decision**: The bot is a group administrator so Telegram delivers `chat_member` updates. `users` rows are created from those updates and from any member message the bot sees. Silence notes include only people with a `users` row who were members at some point during that Minsk day. The bot's own user id is never a silence subject. Members who were already in the chat before the bot could see them, and who never speak and never change status, are absent from the roster until Telegram reveals them. The recap does not invent extra people or a longer count.

**Rationale**: The Bot API has no method that lists every member of a group. Administrator status is the supported way to observe joins and leaves.

**Alternatives considered**: `getChatAdministrators` only (rejected: misses ordinary members). Guessing a full roster (rejected: the spec forbids invented silence counts).

## Authentication

**Decision**: Both services refuse unauthenticated work.

- Telegram webhook updates are accepted only with the webhook secret token. Commands and recaps run only for a chat whose `chats.bot_is_member` is true. `/logs` also requires `getChatMember` to show the sender as `creator`, `administrator`, `member`, or `restricted`. A private chat, a non-member, or a chat the bot has left receives one Russian refusal and no action rows.
- Redis requires a password and an ACL. The backend user can use the message buffer, the job list, and the result list. The AI user can only pop jobs and push results. Redis is not bound to a public interface.
- Job and result bodies are encrypted with `REDIS_PAYLOAD_KEY` and signed with HMAC-SHA256 (`INTERNAL_HMAC_KEY`). The AI service drops any payload that fails decryption or signature checks and returns no narrative. The backend drops any result that fails those checks.
- The AI service does not listen on a public port. Liveness is the process plus a Redis key `ai:heartbeat` that contains no chat data.

**Rationale**: The user required that neither service be usable from outside, and that product behavior exist only in a chat the bot belongs to. Network authentication and chat membership are different checks; both are required. Principle II keeps every secret in the environment.

**Alternatives considered**: IP allow-lists alone (rejected: a forwarded port would still accept an unsigned job). A public AI HTTP endpoint with a bearer token (rejected: an unused public route is an extra entry point).

## Backend modules

**Decision**: NestJS feature modules, each owning its types and persistence:

| Module | Owns |
| --- | --- |
| `ConfigModule` | Environment schema |
| `DatabaseModule` | PostgreSQL connection |
| `AuthModule` | Webhook secret, chat membership, HMAC, payload crypto |
| `TelegramModule` | Bot API, update intake, outbound messages |
| `QueueModule` | Redis buffer, job list, result list, result cache |
| `UsersModule` | `users` and silence counts |
| `LogsModule` | `logs` and `/logs` rendering |
| `DigestModule` | Schedule, attempt limits, eligibility, composition, publish decision |

`DigestModule` orchestrates. It does not embed Bot API calls or SQL for another module's tables.

**Rationale**: The user asked for a modular backend. Feature modules keep the Telegram edge, the queue contract, and the tables independently testable.

**Alternatives considered**: A single `AppService` (rejected: it would mix transport, storage, and recap rules). Prisma as the module boundary (rejected: one global schema client is a weaker fit for per-feature repositories than TypeORM entities registered by each module).

## Tests and the golden set

**Decision**: `contracts/digest-cases.json` is the synthetic dataset. Each case names `retrieved_texts` and `ignored_texts` for the retriever, plus the publish and silence expectations. Backend tests run every case with a stub narrative and assert publish behavior, silence lines, count updates, and that PostgreSQL never receives message text or narrative. Default AI tests stub the embedder and the generator: retrieval must return the required texts and drop the ignored ones, the generator must see only the retrieved texts, thinking stays off, and `work_log` contains no message text and no narrative. A `local_eval` mark loads the real GGUF and `bge-m3` from `/models` and is off in the default run. It passes only when at least 90% of cases that expect events include every required phrase, no case includes a forbidden phrase, and the retriever matches `retrieved_texts` and `ignored_texts`.

**Rationale**: Principle VII requires automated tests. Loading a 3 GB model on every commit is slow, and principle IX forbids a second generation while a cached result exists. The stub suite locks the combinations. The local suite is the SC-002 check and does not leave the container.

**Alternatives considered**: Exact full-text match against the generator (rejected: wording varies). Calling a hosted model for the eval (rejected: no outside model server). Scoring retrieval only by hand (rejected: the golden file is the regression set).

## Attempt timing

**Decision**: The scheduler fires at 00:00 `Europe/Minsk`. Attempt 1 runs then. If that attempt fails, attempt 2 runs 15 minutes later and attempt 3 runs 30 minutes after attempt 1, still inside the same new Minsk day. Three is the maximum. A settled day (recap published, empty day skipped, or three failures) does not run again. Lateness without a failed attempt posts no failure note. After the new Minsk day ends, nothing further is posted for that recap day.

**Rationale**: SC-001 allows the successful midnight path to finish by 00:05. Spacing failed attempts avoids burning all three tries during a short outage, which FR-013 still caps at three.

**Alternatives considered**: Three immediate retries (rejected: a one-minute outage would consume the whole day). Hourly retries past midnight of the following day (rejected: the spec closes the window at the next midnight).
