---
name: email-label-apply
description: Apply Gmail labels in batch with retry and quota safety. Use as a sub-skill of email-classify.
---

# Email Label Apply

Take a map of `category → list of message IDs` and apply the corresponding `AI/<category>` label to each message efficiently.

## Constants

```
BATCH_SIZE = 25
MAX_RETRIES = 3
INITIAL_BACKOFF_SECONDS = 2
RATE_LIMIT_BACKOFF_SECONDS = 30
```

## Procedure

1. Receive `{category: [message_ids]}` and the in-memory `{label_name: label_id}` map from the parent skill.
2. For each category, resolve the label ID for `AI/<category>`. If the label ID is missing, **stop and surface the error** — do not silently skip.
3. Chunk the message ID list into groups of `BATCH_SIZE`.
4. For each chunk, call `batch_modify_gmail_message_labels` with:
   - `messageIds`: the chunk
   - `addLabelIds`: `[label_id]`
   - `removeLabelIds`: `[]`  (we never strip existing labels)
5. On success, count the chunk size into the running total.
6. On failure, apply the retry policy below.

## Retry policy

- **HTTP 429 / rate-limit error** → sleep `RATE_LIMIT_BACKOFF_SECONDS`, then retry the same chunk. Up to `MAX_RETRIES`.
- **Transient 5xx** → exponential backoff starting at `INITIAL_BACKOFF_SECONDS`, doubling each time. Up to `MAX_RETRIES`.
- **4xx other than 429** (auth, bad input) → do **not** retry. Stop the run, report which chunk failed and why.
- **Permanent failure after MAX_RETRIES** → record the failed message IDs, continue with the next chunk, and include the failed list in the final summary.

## Hard rules

- Never call `batch_modify_gmail_message_labels` with `removeLabelIds` populated. This skill is additive only.
- Never include label IDs not in the in-memory map. If a category somehow has no resolved ID, fail loudly.
- Do not parallelize chunks. Gmail's batch endpoint already covers `BATCH_SIZE` messages per call; firing many in parallel just trips rate limits.

## Output

Return a structured result:

```json
{
  "applied": { "contract": 3, "sales": 9, "support": 5, ... },
  "failed":  { "sales": ["18f2ab...", "18f2ac..."] }
}
```

Empty `failed` map is the happy path.
