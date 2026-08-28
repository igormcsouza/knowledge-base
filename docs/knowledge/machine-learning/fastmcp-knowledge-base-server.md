---
tags:

- mcp
- fastmcp
- ai
- agents
- aws-lambda
- python

---

# FastMCP: Building and Deploying an MCP Server

The [Model Context Protocol](https://modelcontextprotocol.io) (MCP) is a standard
way for an LLM client (Claude Desktop, Claude Code, or any custom agent) to
discover and call **tools**, read **resources**, and use **prompts** exposed by an
external server — instead of every integration inventing its own bespoke
function-calling glue. **FastMCP** is the Python framework for building that
server side: plain decorated functions in, a fully spec-compliant MCP server out.

## How FastMCP Works

FastMCP wraps three MCP primitives as decorators on ordinary Python functions:

- **`@mcp.tool`** — an action the model can invoke, with arguments. FastMCP
  reads the function's type hints and docstring to generate the JSON Schema the
  client needs, so there's no schema to hand-write or keep in sync.
- **`@mcp.resource`** — read-only data the client can fetch by URI, similar in
  spirit to a GET endpoint.
- **`@mcp.prompt`** — a reusable, parameterized prompt template the client can
  pull and fill in.

The server object itself just needs a name, and running it is a single call:

```python
from fastmcp import FastMCP

mcp = FastMCP("Knowledge Base")

# ... @mcp.tool-decorated functions go here ...

if __name__ == "__main__":
    mcp.run()
```

What actually changes between local use and a production deployment is the
**transport** — how requests and responses cross the wire — covered below.

## Example: Querying This Knowledge Base

A natural first tool for a docs site like this one is "let an agent search and
read the articles." Both tools below only ever touch files under `docs/knowledge`
— `read_article` resolves the path and rejects anything that escapes that root,
which matters the moment `path` is attacker- or model-controlled input rather
than a trusted constant.

```python
import re
from pathlib import Path

from fastmcp import FastMCP

mcp = FastMCP("Knowledge Base")

DOCS_ROOT = Path(__file__).parent / "docs" / "knowledge"


def _iter_articles():
    for path in DOCS_ROOT.rglob("*.md"):
        if path.name != "index.md":
            yield path


@mcp.tool
def search_articles(query: str) -> list[dict]:
    """Search knowledge base articles by keyword in their title or body."""
    query = query.lower()
    results = []
    for path in _iter_articles():
        text = path.read_text()
        if query in text.lower():
            title_match = re.search(r"^#\s+(.+)$", text, re.MULTILINE)
            results.append({
                "path": str(path.relative_to(DOCS_ROOT)),
                "title": title_match.group(1) if title_match else path.stem,
            })
    return results


@mcp.tool
def read_article(path: str) -> str:
    """Return the full markdown content of one article, given its relative path."""
    article_path = (DOCS_ROOT / path).resolve()
    if not article_path.is_relative_to(DOCS_ROOT) or not article_path.exists():
        raise ValueError(f"Unknown article: {path}")
    return article_path.read_text()


if __name__ == "__main__":
    mcp.run()
```

An agent can now ask `search_articles("idempotency")`, get back a path, and
follow up with `read_article(...)` to pull the full text into context — no
custom retrieval pipeline needed for a docs corpus this size.

!!! tip "Try it locally first"
    `fastmcp dev server.py` spins up the MCP Inspector, a browser UI for calling
    tools by hand — the fastest way to sanity-check a tool's schema and output
    before wiring it into any client.

## Deployment

### stdio — local clients

For a client running on the same machine (Claude Desktop, Claude Code), the
default `mcp.run()` uses **stdio**: the client spawns the server as a
subprocess and talks to it over stdin/stdout. No networking, no auth — the
process boundary *is* the security boundary. This is the right choice for a
personal tool like the knowledge-base server above.

### Streamable HTTP — remote clients

To let a server be reached over a network instead, switch transports:

```python
mcp.run(transport="http", host="0.0.0.0", port=8000)
```

This serves MCP over **Streamable HTTP**, the spec's HTTP-based transport —
a single endpoint that responds with a normal JSON body for quick calls, or
switches to a chunked/SSE stream when the server needs to push multiple
messages back (progress updates on a long-running tool call, for instance).
`mcp.http_app()` returns the underlying ASGI app directly, so it can be run
with `uvicorn` or mounted inside a larger Starlette/FastAPI app like any other
route.

## Deploying to AWS Lambda: Web Adapter vs. a Built-In Handler

Once the server speaks HTTP, [Lambda](../devops-tools/aws/lambda.md) is a
natural host for it — pay per request, no server to keep patched. There are
two ways to get an ASGI app like `mcp.http_app()` running inside Lambda, and
for an MCP server specifically they aren't equivalent.

**A built-in ASGI adapter (e.g. Mangum)** translates the incoming Lambda
event (API Gateway or Function URL) into an ASGI `scope` in-process, calls
the app, and translates the response back:

```python
from mangum import Mangum

app = mcp.http_app()
handler = Mangum(app)
```

This ships as a plain zip package, needs no extra infrastructure, and is the
simplest path when a server's tools are fast, synchronous request/response
calls. Its limitation is that it buffers the **entire** response before
handing it back to Lambda — there's no way to stream chunks out as they're
produced, because the ASGI-to-Lambda-event translation only has a complete
response object to work with.

**The [AWS Lambda Web Adapter](https://github.com/awslabs/aws-lambda-web-adapter)**
takes the opposite approach: it doesn't translate the app at all. It runs the
app exactly as it would run anywhere else —

```dockerfile
FROM public.ecr.aws/awsguru/aws-lambda-adapter:0.8 AS adapter
FROM python:3.12-slim
COPY --from=adapter /lambda-adapter /opt/extensions/lambda-adapter
COPY . /app
WORKDIR /app
RUN pip install -r requirements.txt
CMD ["uvicorn", "server:app", "--host=0.0.0.0", "--port=8080"]
```

The adapter is a small Rust binary, added as a Lambda extension (layer or, as
above, baked into a container image). It receives the actual Lambda
invocation, forwards it to `uvicorn` over `localhost:8080` as a real HTTP
request, and — when the Function URL is configured with
`InvokeMode: RESPONSE_STREAM` — streams the response back chunk by chunk
instead of buffering it.

| | Built-in adapter (Mangum) | Lambda Web Adapter |
|---|---|---|
| Code changes needed | Wrap the app in `Mangum(app)` | None — app runs unmodified |
| Response streaming | No — buffers the full response | Yes — true chunked/SSE streaming |
| Deploys as | Zip package | Container image (or layer) |
| Portability | Lambda-specific handler | Same image runs on Fargate, App Runner, or a laptop unchanged |
| Cold start | Slightly lighter — no extra process | Slightly heavier — spins up a real HTTP server process |

!!! note "Why this matters specifically for MCP"
    Streamable HTTP relies on being able to push more than one message per
    request when a tool call is long-running. A buffering adapter silently
    degrades that to "wait for the whole thing," which defeats the point for
    any tool that reports progress or streams partial results. For a
    production MCP server, the Web Adapter's real streaming is worth the extra
    container-image setup; Mangum is fine for a low-traffic server whose tools
    are all quick, one-shot calls — like the knowledge-base search above.

## Integrating Into Projects

- **Claude Desktop / Claude Code** — point the client's MCP config at either a
  local command (stdio) or a deployed URL (HTTP):

```json
{
  "mcpServers": {
    "knowledge-base": {
      "command": "uv",
      "args": ["run", "server.py"]
    }
  }
}
```

  or, for the deployed Lambda version, a `"url"` entry pointing at the Function
  URL instead of a `"command"`.

- **From another service or agent framework** — use `fastmcp.Client` to call
  tools programmatically, without a chat client in the loop:

```python
from fastmcp import Client

async with Client("server.py") as client:
    hits = await client.call_tool("search_articles", {"query": "idempotency"})
```

  The same `Client` works against a stdio script or an `http://...` URL —
  swapping deployment target doesn't change calling code.

- **Deploy the same way other infrastructure gets deployed** — drive the
  container build and Lambda/Function URL creation through
  [CDK](../devops-tools/aws/cdk.md) (or whatever IaC tool the project already
  uses) rather than a manual `docker push` + console click, for the same
  reasons any other resource belongs in version control.

## Summary

- FastMCP turns decorated Python functions into MCP tools/resources/prompts,
  generating their JSON Schema from type hints automatically.
- stdio is for local, same-machine clients; Streamable HTTP is for anything
  reached over a network.
- On Lambda, a built-in ASGI adapter (Mangum) is simplest but buffers
  responses; the AWS Lambda Web Adapter runs the app unmodified and supports
  true streaming — the better fit for MCP servers with long-running tools.
- `fastmcp.Client` calls tools the same way regardless of whether the server
  is a local script or a deployed HTTP endpoint, so switching deployment
  target doesn't touch calling code.

## Related Articles

- [AWS Lambda](../devops-tools/aws/lambda.md) — the execution model (cold
  starts, statelessness, triggers) that the Lambda deployment options above
  build on.
- [AWS CDK](../devops-tools/aws/cdk.md) — how to define and deploy the
  container image and Function URL as code rather than by hand.
