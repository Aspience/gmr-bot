# Queue and authentication contract

Services exchange extraction work only through Redis. The plaintext schemas are `digest-job.schema.json` and `digest-result.schema.json`. On the wire, that JSON is never stored as plaintext.

## Envelope

Every job, result, buffer entry, and cached result uses the same envelope:

1. Serialize the JSON body with sorted keys and UTF-8.
2. Encrypt with AES-256-GCM. The 32-byte key is `REDIS_PAYLOAD_KEY` (base64). A fresh 12-byte nonce is generated per message.
3. Sign `nonce || ciphertext || tag` with HMAC-SHA256 and `INTERNAL_HMAC_KEY`.
4. Push the nonce, ciphertext, tag, and signature.

The receiver verifies the signature with a constant-time compare, then decrypts, then validates the schema. A failed check drops the payload. The AI service answers an invalid job with `status: error` and `error_code: invalid_job` or `bad_signature`, `narrative_ru: null`, and a `work_log` entry that contains no source text. The backend answers an invalid result by inserting `rejected_result` and treating the attempt as failed. It does not publish a narrative from that payload.

## Lists

| List | Producer | Consumer |
| --- | --- | --- |
| `digest:jobs` | backend | ai |
| `digest:results` | ai | backend |

The backend waits up to 120 seconds for a result with the same `job_id`. A timeout is attempt failure `model_timeout`. The AI worker, if it finishes later, still pushes the signed result; the backend ignores a result for a job it no longer considers open, and it does not store the narrative.

A second push of the same `job_id` is answered from `digest:cache:{job_id}` and does not embed or generate again.

## Who may use Redis

Redis is bound to a private interface and requires `REDIS_PASSWORD`.

| ACL user | Allowed |
| --- | --- |
| backend | Read and write `buffer:*`, push `digest:jobs`, pop `digest:results`, read `ai:heartbeat` |
| ai | Pop `digest:jobs`, push `digest:results`, read and write `digest:cache:*`, set `ai:heartbeat` |

The AI user cannot read `buffer:*`. The backend user is the only writer of that buffer.

## Chat gate

Authentication of the queue does not replace the chat gate.

- The webhook handler rejects any update whose `X-Telegram-Bot-Api-Secret-Token` does not match `TELEGRAM_WEBHOOK_SECRET`.
- The handler then drops the update unless `chats.bot_is_member` is true for that chat. `my_chat_member` is the exception used to maintain that flag.
- `/logs` in a chat the bot has not joined, in a private chat, or from a sender `getChatMember` does not report as `creator`, `administrator`, `member`, or `restricted`, gets the refusal in `telegram-copy.md` and no rows from `logs`.

No HTTP route accepts a chat id and returns a recap or a log. The AI process opens no listening socket.

## Rate limits

The webhook validates the secret and the update shape before any other work, then acknowledges Telegram.

| Entry | Limit | Over the limit |
| --- | --- | --- |
| Updates in one chat | 30 accepted per 10 seconds | Extra updates are not stored. That Minsk day is partly visible. |
| `/logs` | 5 per sender per 10 minutes | The rate-limit reply is sent. No log row is written. |
| Local generation | 1 at a time in the AI process | Further jobs stay on `digest:jobs`. |

One chat's model call must not be awaited inside the webhook handler.

## Local inference

The worker does not send the job to a model host. It embeds and generates with the files in `/models`, as specified in `rag.md`.

| Bound | Value |
| --- | --- |
| Embedder | `BAAI/bge-m3` on local disk |
| Generator | `Qwen3.5-4B` GGUF `Q4_K_M` on local disk |
| Thinking | off; a `<think>` block is stripped and never becomes `narrative_ru` |
| Temperature | `0.2` |
| Retrieved messages | at most 40 |
| Output tokens | at most 800 |
| Job time | 90 seconds, then `model_timeout` |
| Runtime network | `HF_HUB_OFFLINE=1`; no generation URL and no generation API key |

The generator prompt contains the system instructions and the retrieved messages only. It does not contain silence counts, forwards, captions, service text, or messages the retriever dropped. `usage` is returned to the backend and is not written to `logs`. `usage.model` is `qwen3.5-4b-q4_k_m`.
