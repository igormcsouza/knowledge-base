---
tags:

- aws
- lambda
- serverless
- compute

---

# AWS Lambda

Lambda runs code in response to events without provisioning or managing any
server — upload a function, tell it what triggers it, and AWS handles
scaling the number of concurrent executions up and down automatically. You
pay per invocation and per execution duration, not for idle capacity.

## The Handler

A Lambda function is a single entry-point function receiving the triggering
event and a context object:

```python
def handler(event: dict, context) -> dict:
    body = event["body"]
    return {"statusCode": 200, "body": f"Processed: {body}"}
```

What `event` looks like depends entirely on the trigger — an API Gateway
request, an S3 notification, and an SQS message all populate `event` with
completely different shapes.

## Cold Starts vs. Warm Invocations

The first invocation of a function (or the first after it's been idle)
requires AWS to provision a fresh execution environment — initializing the
runtime, running any module-level code, before the handler itself runs. This
is a **cold start**, and it's measurably slower than a **warm** invocation,
which reuses an already-initialized environment.

```python
# runs once per cold execution environment, reused across warm invocations
db_client = create_db_client()


def handler(event: dict, context) -> dict:
    return db_client.query(event["id"])  # warm invocations skip client setup
```

Anything expensive to set up (DB connections, SDK clients) belongs at module
level, outside the handler, specifically so warm invocations reuse it instead
of paying that cost every time.

## Execution Environment Is Not Persistent State

Two invocations of the same function may run in the same warm environment,
or in two entirely separate ones — there's no guarantee which, and no way to
rely on it. Anything a function needs to persist between invocations has to
go somewhere external (a database, [S3](s3.md), [SQS](sqs.md)) — never in a
local variable or file, the same "don't rely on process-local memory"
principle covered for [Kubernetes pods](../kubernetes.md#self-healing).

## Configuration: Memory, Timeout, Concurrency

- **Memory** — configured directly; **CPU scales proportionally with
  memory**, so a CPU-bound function can sometimes get *cheaper* by
  increasing memory, because it finishes faster.
- **Timeout** — a hard cap (up to 15 minutes) on how long a single
  invocation may run before it's forcibly terminated.
- **Reserved concurrency** — caps how many concurrent executions a function
  can use, protecting downstream systems (like a database) from being
  overwhelmed by a burst of invocations.
- **Provisioned concurrency** — keeps a set number of execution environments
  pre-initialized and warm at all times, trading cost for eliminating cold
  starts on latency-sensitive paths.

## IAM Execution Role

Every Lambda function assumes an IAM role granting it exactly the
permissions it needs to touch other AWS resources (read this S3 bucket,
write to that SQS queue). Least-privilege here matters more than it might
seem — a function's blast radius if compromised is bounded by its role, not
by what it happens to call.

## Common Triggers

- **API Gateway** — synchronous HTTP requests; the function's return value
  becomes the HTTP response.
- **SQS** — asynchronous, batched; Lambda polls the queue and invokes the
  function with a batch of messages. Since SQS delivers **at-least-once**,
  the same message can trigger the function more than once — the handler
  must be [idempotent](../../databases/idempotency.md). For partial
  failures within a batch, `ReportBatchItemFailures` lets only the failed
  messages be retried instead of the whole batch.
- **S3 events** — object created/deleted notifications, useful for
  processing uploads as they land.
- **EventBridge** — scheduled (cron-like) invocations, or routing custom
  application events.

## Deployment

Functions ship either as a **zip** package (code + dependencies) or as a
**container image** (using Lambda's container runtime) — useful when
dependencies are large or need OS-level packages a zip can't carry. Either
way, deployment is almost always driven by infrastructure-as-code (see
[CDK](cdk.md)) rather than manual console uploads, so function code and its
trigger wiring stay versioned together.

## Summary

- Lambda scales invocations automatically; you pay per invocation/duration,
  not for idle capacity.
- Cold starts pay initialization cost; keep expensive setup at module level
  so warm invocations skip it.
- Never rely on the execution environment persisting state between
  invocations — externalize anything that needs to survive.
- Memory configuration also controls CPU; timeout, reserved concurrency, and
  provisioned concurrency are the main dials for cost/latency/protection
  trade-offs.
- SQS-triggered functions must be idempotent — at-least-once delivery means
  duplicate invocations will happen.

## Related Articles

- [SQS](sqs.md) — the most common asynchronous trigger, and the source of
  the at-least-once delivery guarantee Lambda handlers need to design around.
- [Idempotency](../../databases/idempotency.md) — the general principle
  behind writing a safe-to-retry Lambda handler.
- [AWS CDK](cdk.md) — the usual way a Lambda function and its trigger get
  deployed and versioned together.
- [FastMCP: Building and Deploying an MCP Server](../../machine-learning/fastmcp-knowledge-base-server.md) —
  running an MCP server on Lambda, and why response streaming pushes that
  toward the AWS Lambda Web Adapter over a built-in ASGI adapter.
