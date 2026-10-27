# Retrieval contract

The AI container runs retrieval over one job, then generation. Nothing in this path calls a hosted model API. The index exists only in the worker process for that job and is discarded when the job returns.

## Corpus

The corpus is `messages` from the extraction job: text a member typed, including replies. Forwards, captions, service notices, and the bot are not in the job. One message is one chunk. A message longer than 512 tokens is split into windows of 512 with an overlap of 64 tokens; every window keeps the same author. Retrieval returns the parent message once.

## Queries

These six queries are embedded once at process start and reused. They are not taken from chat text.

| Category | Query |
| --- | --- |
| Family | семейное событие: рождение ребёнка, свадьба, родственники |
| Health | здоровье: болезнь, больница, операция, выздоровление |
| Relationship | личные отношения: помирились, расстались, начали встречаться |
| Move | переезд в другой город или в новое жильё |
| Job | работа: новая работа, увольнение, повышение |
| Achievement | личное достижение: диплом, победа, награда |

## Selection

Embed every chunk with the local embedding model. A message's score is the maximum cosine similarity of its chunks against the six query vectors. Keep a message when that score is greater than or equal to `RAG_SCORE_THRESHOLD`.

The shipped threshold is the highest value that, on `digest-cases.json` with the real embedder, returns every `retrieved_texts` entry and none of the `ignored_texts` entries. Until that calibration is recorded, the starting value is `0.35`.

After the threshold, keep at most 40 messages, the highest scores first. If eligible messages were dropped only because of this cap, or because embedding stopped at its time limit, the result sets `context_truncated` to true. The backend then treats the day as partly visible. Leaving out small talk that fell below the threshold is not truncation.

Surviving messages are passed to the generator in their original order.

## Generator skip

If no message meets the threshold, the generator does not run. `narrative_ru` is null, `event_count` is 0, and `usage.output_tokens` is 0. That is a successful extraction, not a failure note.

## What the generator receives

The system instructions in `telegram-copy.md`, plus the retrieved messages and their author labels. It does not receive silence counts, the queries, scores, or messages that were filtered out.

## Local models

| Role | Model | Loaded from |
| --- | --- | --- |
| Embedder | `BAAI/bge-m3` | `/models/embedder` inside the image |
| Generator | `Qwen3.5-4B` GGUF `Q4_K_M` | `/models/generator.gguf` inside the image |

Both files are copied into the image at build time. The running container sets `HF_HUB_OFFLINE=1` and does not open a connection to a model host. There is no generation API key and no generation base URL.

The generator disables thinking. Any `<think>` block is stripped before `narrative_ru` is set. If nothing remains after stripping, the result is `model_rejected` and `narrative_ru` is null.

Generation settings: temperature `0.2`, at most 800 output tokens, context wide enough for the 40 retrieved messages and no wider. One generation at a time. Embedding may use the rest of the 90-second job budget. A job that exceeds 90 seconds returns `model_timeout`.
