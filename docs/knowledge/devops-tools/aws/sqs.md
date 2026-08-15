---
tags:

- aws
- sqs
- messaging
- queues

---

# Amazon SQS

SQS is AWS's managed message queue. The general shape — Standard vs. FIFO
queues, at-least-once delivery, visibility timeouts, and how SQS compares to
Kafka/RabbitMQ — is covered in
[Event-Driven Architecture](../../architecture/event-driven-architecture.md);
this article focuses on the practical mechanics of actually using it.

## Batching

Both sending and receiving support batches of up to 10 messages per call
(`SendMessageBatch`, `ReceiveMessage` with `MaxNumberOfMessages=10`) —
batching is significantly cheaper and faster than one API call per message,
and is the default assumption for any production-grade producer/consumer.

## Long Polling

`ReceiveMessage` with `WaitTimeSeconds > 0` enables **long polling** — the
call blocks (up to 20 seconds) waiting for a message to become available,
instead of returning immediately with an empty result. Long polling
dramatically reduces the number of empty API calls (and their cost) compared
to **short polling** (`WaitTimeSeconds=0`), which returns immediately
whether or not there's a message. There's essentially never a reason to
prefer short polling for a production consumer.

## Dead-Letter Queues

A **redrive policy** on a queue routes messages to a separate **dead-letter
queue (DLQ)** after they've been received and failed to be deleted (i.e.
failed to process) a configured number of times. Without a DLQ, a
"poison" message that always fails processing gets redelivered forever,
endlessly consuming consumer capacity; with one, it's set aside after N
attempts for manual inspection instead of looping indefinitely. A DLQ is
close to mandatory for any consumer where processing can genuinely fail.

## Lambda Integration Specifics

When SQS triggers [Lambda](lambda.md), a few settings matter beyond the
general Lambda trigger behavior:

- **Batch size** — how many messages Lambda pulls per invocation; larger
  batches amortize invocation overhead but mean one failing message can
  affect the whole batch's retry behavior.
- **`ReportBatchItemFailures`** — without it, a single failed message in a
  batch causes the *entire batch* to be retried, including messages that
  already succeeded; with it enabled, the handler explicitly reports which
  message IDs failed, and only those are retried.

```python
def handler(event: dict, context) -> dict:
    failures = []
    for record in event["Records"]:
        try:
            process(record["body"])
        except Exception:
            failures.append({"itemIdentifier": record["messageId"]})

    return {"batchItemFailures": failures}
```

## Idempotency Is Not Optional

SQS's at-least-once delivery means a consumer **will** see duplicate
messages sooner or later — after a Lambda invocation times out mid-processing,
after a visibility timeout expires before a slow consumer finishes, or from
retries after a partial batch failure. Processing has to be safe to run
twice on the same message; see [Idempotency](../../databases/idempotency.md)
for the general techniques (idempotency keys, upserts, conditional updates).
A message ID or an application-level sequence number in the message body is
usually what a consumer keys deduplication on.

## Summary

- Batch sends and receives (up to 10 messages) instead of one call per
  message.
- Always use long polling (`WaitTimeSeconds > 0`) — short polling wastes
  API calls for essentially no benefit.
- Attach a dead-letter queue so a permanently-failing message gets set
  aside instead of retried forever.
- For Lambda triggers, enable `ReportBatchItemFailures` so one bad message
  doesn't force a retry of an entire successful batch.
- Design every consumer assuming duplicate delivery will happen — idempotent
  processing is the baseline requirement, not an edge case.

## Related Articles

- [Event-Driven Architecture](../../architecture/event-driven-architecture.md)
  — the general Kafka/RabbitMQ/SQS comparison this builds on.
- [AWS Lambda](lambda.md) — the most common SQS consumer in a serverless
  setup.
- [Idempotency](../../databases/idempotency.md) — the general principle
  behind designing for SQS's at-least-once delivery.
