---
tags:

- aws
- cdk
- infrastructure-as-code
- cloudformation

---

# AWS CDK

The Cloud Development Kit (CDK) defines cloud infrastructure using a real
programming language (TypeScript, Python, Java, etc.) instead of a static
YAML/JSON template. It compiles ("synthesizes") that code down to a
CloudFormation template, and CloudFormation is what actually provisions the
resources — CDK is a layer of abstraction over CloudFormation, not a
replacement for it.

## Why Infrastructure as Code

Manually clicking through the AWS console to create resources ("ClickOps")
leaves no record of what was done, no review process, and no way to
reproduce the same setup in another environment. Defining infrastructure as
code fixes all three: changes go through the same version control and code
review as application code, and the same stack definition can be deployed
identically to dev, staging, and prod.

## Core Concepts

- **App** — the root of a CDK program; contains one or more Stacks.
- **Stack** — a deployable unit, mapping 1:1 to a CloudFormation stack —
  a logical grouping of resources (e.g. "the API stack," "the data stack").
- **Construct** — a reusable definition of one or more resources, at three
  levels of abstraction:
   - **L1** — a direct, low-level wrapper around a single CloudFormation
     resource (`CfnBucket`, `CfnFunction`).
   - **L2** — a higher-level, intent-based wrapper around an L1 (`Bucket`,
     `Function`) with sensible defaults and convenience methods.
   - **L3 (patterns)** — pre-built combinations of several resources
     solving a common use case (e.g. an API Gateway + Lambda pair wired
     together).

## A Minimal Stack

```python
from aws_cdk import Stack, aws_s3 as s3, aws_lambda as lambda_
from constructs import Construct


class ApiStack(Stack):
    def __init__(self, scope: Construct, id: str, **kwargs) -> None:
        super().__init__(scope, id, **kwargs)

        uploads_bucket = s3.Bucket(
            self, "UploadsBucket",
            versioned=True,
        )

        handler = lambda_.Function(
            self, "ProcessUpload",
            runtime=lambda_.Runtime.PYTHON_3_12,
            handler="process_upload.handler",
            code=lambda_.Code.from_asset("lambda"),
            environment={"BUCKET_NAME": uploads_bucket.bucket_name},
        )

        uploads_bucket.grant_read(handler)
```

`grant_read` is an L2 convenience method — it adds exactly the IAM policy
statement needed for `handler` to read from `uploads_bucket`, without
hand-writing the policy JSON.

## Workflow

```bash
cdk synth    # render the CloudFormation template, without deploying
cdk diff     # show what would change against what's currently deployed
cdk deploy   # deploy the stack (creates a CloudFormation change set, applies it)
cdk destroy  # tear the stack down
```

`cdk diff` before `cdk deploy` is the equivalent of reviewing a `git diff`
before merging — it's what catches an unintended resource replacement (which
can mean downtime or data loss) before it actually happens.

## Environments and Context

A stack can be deployed to different AWS accounts/regions by parameterizing
the `env` passed to it, keeping the same stack definition reusable across
dev/staging/prod instead of duplicating code per environment. `cdk.json`
holds CDK CLI configuration and **context** values — cached lookups (like
"the default VPC's ID") that would otherwise require an API call on every
synth.

## CDK vs. Raw CloudFormation vs. Terraform

- **Raw CloudFormation** — the same guarantees CDK ultimately relies on, but
  authored directly as YAML/JSON — verbose, no loops/conditionals beyond
  CloudFormation's own limited intrinsic functions.
- **CDK** — real programming language on top of CloudFormation: loops,
  functions, reusable constructs, type checking — but AWS-only, since it
  synthesizes to CloudFormation specifically.
- **Terraform** — a different provisioning engine entirely (not
  CloudFormation-based), multi-cloud, using its own HCL language (or CDK for
  Terraform for a similar real-language experience). The choice between CDK
  and Terraform mostly comes down to whether the infrastructure is AWS-only
  and whether the team prefers a general-purpose language over HCL.

## Best Practices

- One stack per logical, independently-deployable unit — not one giant stack
  for the whole account, which makes every deploy touch everything.
- Prefer L2/L3 constructs over L1 whenever one exists — they encode AWS's
  own best-practice defaults (encryption, sane IAM scoping) that an L1
  wrapper leaves entirely up to you.
- Avoid hardcoding account IDs/regions in a stack meant to be reused across
  environments — parameterize via `env` and context instead.

## Summary

- CDK is real code that synthesizes to CloudFormation — CloudFormation still
  does the actual provisioning.
- Constructs range from low-level (L1, one resource) to pattern-level (L3,
  several resources wired together for a common use case).
- `cdk diff` before `cdk deploy` is the safety check that catches
  unintended replacements before they happen.
- Parameterize environment (account/region) rather than duplicating stacks
  per environment.

## Related Articles

- [AWS Lambda](lambda.md) — one of the most common resources defined and
  deployed via a CDK stack.
- [S3](s3.md) — another resource type shown in the example stack above.
