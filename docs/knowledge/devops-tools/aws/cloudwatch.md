---
tags:

- aws
- cloudwatch
- observability
- monitoring

---

# Amazon CloudWatch

CloudWatch is AWS's native observability service: it collects logs, metrics,
and alarms across almost every AWS service, and is where anything running on
AWS ends up being watched from by default. Most other AWS services write to
it automatically — no separate agent to install — which makes it the default
starting point for observability even on teams that later layer a dedicated
APM tool on top.

## Logs: Log Groups and Log Streams

CloudWatch Logs organizes log data in two levels:

- **Log group** — a named container for related logs, typically one per
  application or per Lambda function (e.g. `/aws/lambda/my-function`). Log
  group settings (retention, encryption, subscription filters) apply to
  every stream inside it.
- **Log stream** — a sequence of log events from a single source within that
  group — for a Lambda function, one stream per execution environment; for
  an ECS service, typically one stream per task.

```bash
aws logs create-log-group --log-group-name /my-app/api
aws logs describe-log-streams --log-group-name /my-app/api
aws logs tail /my-app/api --follow
```

### Retention

By default, log groups **retain events forever** — which quietly becomes a
real cost line item on high-volume applications. Set an explicit retention
policy on every log group instead of relying on the default:

```bash
aws logs put-retention-policy \
  --log-group-name /my-app/api \
  --retention-in-days 30
```

!!! warning "Forgotten log groups are a real cost"
    A Lambda function or ECS service that's been redeployed hundreds of
    times leaves behind hundreds of log streams inside one group, all kept
    indefinitely unless retention is set. This is one of the most common
    AWS bill surprises on a growing account — check retention on log groups
    the same way you'd check S3 lifecycle rules.

### Logs Insights

CloudWatch Logs Insights is a query language for ad-hoc searching and
aggregating across a log group without exporting anything first:

```text
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 20
```

```text
fields @timestamp, duration
| filter ispresent(duration)
| stats avg(duration), max(duration), count() by bin(5m)
```

The first example is a straightforward text-search-and-tail; the second
parses structured fields out of the log lines and aggregates them into
5-minute buckets — useful for spotting a latency regression without shipping
logs anywhere else first. Insights queries can also span multiple log groups
at once, which matters for a service split across several Lambda functions.

## Metrics

A CloudWatch **metric** is a time-ordered set of data points identified by a
**namespace** (a logical grouping, e.g. `AWS/Lambda`, `AWS/EC2`, or a custom
namespace like `MyApp/Orders`) and a set of **dimensions** (key/value pairs
that further identify the metric, e.g. `FunctionName=my-function`). The same
metric name in different namespaces or with different dimensions is a
completely separate time series.

### Standard vs. Custom Metrics

- **Standard metrics** — emitted automatically by AWS services. Lambda
  publishes `Invocations`, `Errors`, `Duration`, `Throttles`; EC2 publishes
  `CPUUtilization`; an ALB publishes `RequestCount`, `TargetResponseTime`.
  No code required.
- **Custom metrics** — published explicitly by application code via
  `put_metric_data`, for anything domain-specific AWS has no way to know
  about (orders processed, queue depth by tenant, a business KPI):

```python
import boto3

cloudwatch = boto3.client("cloudwatch")


def record_order_processed(amount: float) -> None:
    cloudwatch.put_metric_data(
        Namespace="MyApp/Orders",
        MetricData=[
            {
                "MetricName": "OrdersProcessed",
                "Dimensions": [{"Name": "Environment", "Value": "production"}],
                "Value": 1,
                "Unit": "Count",
            },
            {
                "MetricName": "OrderAmount",
                "Dimensions": [{"Name": "Environment", "Value": "production"}],
                "Value": amount,
                "Unit": "None",
            },
        ],
    )
```

!!! tip "Batch custom metrics"
    `put_metric_data` accepts up to 1000 metric/value pairs per call, and
    CloudWatch also supports the embedded metric format (structured JSON in
    a regular log line, parsed into metrics automatically) as a cheaper
    alternative to one API call per data point in a hot path.

### Granularity: Standard vs. Detailed Monitoring

Most AWS services publish metrics at **1-minute** granularity by default
(some, like basic EC2 monitoring, default to 5-minute granularity unless
**detailed monitoring** is enabled, which raises EC2 metrics to 1-minute
resolution for an added cost). Custom metrics can go finer still — down to
1-second resolution as **high-resolution metrics** — at a higher cost per
data point, useful for an alarm that needs to react faster than a minute.
The trade-off is always the same: finer granularity means faster detection
and faster alarm reaction, at higher storage/API cost.

## Alarms

A CloudWatch **alarm** watches one metric (or a math expression over
several) and changes state based on whether it crosses a threshold:

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name high-error-rate \
  --namespace AWS/Lambda \
  --metric-name Errors \
  --dimensions Name=FunctionName,Value=my-function \
  --statistic Sum \
  --period 60 \
  --evaluation-periods 3 \
  --threshold 5 \
  --comparison-operator GreaterThanThreshold \
  --treat-missing-data notBreaching \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:ops-alerts
```

Key knobs:

- **Period** — the size of each evaluation window in seconds (here, 60
  seconds — one data point per minute).
- **Evaluation periods** — how many consecutive periods must breach the
  threshold before the alarm actually fires. `evaluation-periods 3` with a
  60-second period means 3 consecutive minutes of breaching data before
  going to `ALARM` — this is what keeps a single noisy spike from paging
  someone at 3am.
- **Treat missing data** — what to do when a period has no data points at
  all (`missing`, `notBreaching`, `breaching`, or `ignore`). This matters
  more than it looks: a metric that only emits on errors (no data = no
  errors) should typically be `notBreaching`, while a metric that should
  always have traffic (no data = something's actually broken, like the
  process crashed and stopped reporting) should often be `breaching`.

### Alarm Actions

An alarm's state transition (`OK` → `ALARM`, `ALARM` → `OK`) can trigger an
action, most commonly:

- **SNS notification** — fan out to email, Slack (via a subscribed Lambda),
  or a paging system like PagerDuty/Opsgenie.
- **Auto Scaling action** — directly scale an Auto Scaling group up or down
  in response to a metric (e.g. high `CPUUtilization` triggers a scale-out
  policy) — the mechanism behind EC2 target-tracking scaling policies.
- **EC2 action** — stop, terminate, or recover an instance automatically.

!!! note "Composite alarms"
    A **composite alarm** combines several other alarms with `AND`/`OR`
    logic (e.g. only page if *both* error rate and latency are elevated at
    once) — useful for cutting down alert noise from correlated failures
    that would otherwise fire as several separate pages for one incident.

## Structured Logging From Lambda

CloudWatch Logs is where a [Lambda](lambda.md) function's stdout/stderr
lands automatically — there's nothing to configure to get logs flowing; AWS
creates the `/aws/lambda/<function-name>` log group and a stream per
execution environment for you. What's worth deliberately setting up is the
**shape** of what gets logged:

```python
import json
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)


def handler(event: dict, context) -> dict:
    logger.info(json.dumps({
        "event": "order_processed",
        "order_id": event["order_id"],
        "request_id": context.aws_request_id,
    }))
    return {"statusCode": 200}
```

Logging structured JSON instead of free-text messages is what makes Logs
Insights queries like `filter order_id = "abc123"` or
`stats count() by event` actually work — an unstructured `f"processed order
{order_id}"` string can only be text-searched, not filtered or aggregated on
a specific field. Including `context.aws_request_id` in every log line ties
every log statement from one invocation back together, which matters once
concurrent invocations are interleaving log lines in the same log stream.

!!! tip "AWS Lambda Powertools"
    For anything beyond a handful of log lines, the AWS Lambda Powertools
    library handles structured JSON logging, request-ID correlation, and
    X-Ray tracing integration with a decorator instead of hand-rolling the
    `json.dumps` calls above in every handler.

## Relationship to X-Ray

CloudWatch Logs and Metrics answer "what happened and how much" per
service; **AWS X-Ray** answers "which service in the call chain was slow"
for a single request that crosses multiple services (API Gateway → Lambda →
DynamoDB, for example) — distributed tracing rather than aggregated
logs/metrics. X-Ray traces and CloudWatch data are complementary and
cross-linked in the console: an elevated `Duration` metric or an error log
line points you at *that something is wrong*, and the corresponding X-Ray
trace shows *where in the request's path* the time actually went.

## Summary

- Log groups hold retention/encryption settings; log streams are the
  individual sequences of events inside them — set retention explicitly, the
  default is "forever."
- Logs Insights queries logs ad hoc with a purpose-built syntax (`fields`,
  `filter`, `stats ... by bin(...)`) without exporting data first.
- Metrics are namespace + dimensions + name; standard metrics are automatic,
  custom metrics go through `put_metric_data` (or embedded metric format).
- Most metrics are 1-minute resolution by default; detailed monitoring (EC2)
  and high-resolution custom metrics trade cost for finer granularity.
- Alarms fire after N consecutive breaching evaluation periods, not on a
  single data point — tune `evaluation-periods` and `treat-missing-data`
  deliberately rather than leaving the defaults.
- Lambda's stdout/stderr lands in CloudWatch Logs automatically; structuring
  those log lines as JSON is what makes them queryable, not just
  text-searchable.
- X-Ray complements CloudWatch with per-request distributed tracing across
  service boundaries, rather than aggregated logs/metrics.

## Related Articles

- [AWS Lambda](lambda.md) — the compute service whose logs land in
  CloudWatch automatically, and the source of the structured-logging example
  above.
- [Kubernetes Fundamentals](../kubernetes.md) — liveness/readiness probes are
  the K8s-native equivalent of "pull an unhealthy thing out of rotation,"
  the same instinct behind a CloudWatch alarm driving an Auto Scaling
  action.
- [Idempotency](../../databases/idempotency.md) — relevant background for
  interpreting duplicate `Errors`/`Invocations` metrics from at-least-once
  triggers.
