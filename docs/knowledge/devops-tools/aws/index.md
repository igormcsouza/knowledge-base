---
title: AWS
tags:

- aws
- cloud
- overview

---

# AWS

Notes on the AWS services used day to day, as a subsection of
[DevOps & Tools](../index.md) — practical mechanics rather than a full
service catalog.

## Articles

- [AWS Lambda](lambda.md) — serverless compute: handlers, cold starts,
  execution-environment statelessness, and common triggers.
- [AWS CDK](cdk.md) — defining infrastructure as real code that synthesizes
  to CloudFormation.
- [AWS Cognito](cognito.md) — User Pools (authentication) vs. Identity Pools
  (temporary AWS credentials), and when to use which.
- [Amazon S3](s3.md) — object storage: keys vs. real directories, presigned
  URLs, event notifications, lifecycle rules.
- [Amazon SQS](sqs.md) — batching, long polling, dead-letter queues, and the
  Lambda integration specifics.
- [Amazon CloudWatch](cloudwatch.md) — Logs and Logs Insights, Metrics and
  custom metrics, Alarms, and Lambda's automatic structured logging.

## Contributing

Picked up something about another AWS service worth keeping? Add it here as
its own file. See [Contributing](../../../contributing.md) for the how-to.
