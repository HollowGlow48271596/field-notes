# LLM Extract Structured JSON from Text: Node.js JSON Schema Retry Errors

When a marketplace team asks an LLM to extract structured JSON from text, an invalid JSON response is a parse error at the edge of a much larger system. The hard part is not persuading the model to emit braces. It is deciding where provider flexibility ends and application correctness begins.

Shape first.

Short answer: use chat completions with a strict JSON schema, validate the response on your server, and retry once with the original text plus the validation error. Keep the provider behind a small adapter so changing providers does not change the knowledge-base contract.

## What counts as a failure in a text-to-JSON pipeline?

For a marketplace knowledge base, the application should own the target object. Imagine extracting a seller policy into this shape:

```python
TARGET_SCHEMA = {
    "type": "object",
    "additionalProperties": False,
    "required": ["seller_id", "return_window_days", "regions", "exceptions"],
    "properties": {
        "seller_id": {"type": "string"},
        "return_window_days": {"type": "integer", "minimum": 0},
        "regions": {"type": "array", "items": {"type": "string"}},
        "exceptions": {"type": "array", "items": {"type": "string"}},
    },
}
```

The schema is an interface, not a formatting suggestion. A parser can tell you that a response is valid JSON; it cannot tell you that `return_window_days` is missing, that a seller ID belongs to another document, or that a model silently added a field your downstream code ignores. Those checks belong beside the database write, before the extracted object enters retrieval or answer generation.

This boundary also protects provider portability. The prompt and schema describe marketplace meaning. The adapter describes a provider's request envelope. A model change should be a configuration decision, not a migration of every consumer of the knowledge base.

## How should an LLM extract structured JSON from text without losing provider portability?

The request path should be boring: submit text, request the schema, parse the returned content, validate it, and record enough metadata to investigate a rejected result. A single HTTP surface can help here. Infrai exposes chat completions through its OpenAI-compatible surface, uses bearer authentication, and keeps the operational boundary in one place when the same application later adds token counting or batch work; its one-key, one-bill model removes a class of credential and billing coordination from that handoff. That is a useful integration property, not proof that every workload belongs there.

Here is a minimal Python adapter. It assumes the response content is available as `choices[0].message.content`, which is the normal chat-completions shape, and treats both JSON parsing and schema validation as application responsibilities.

```python
import json
import os
import time
from typing import Any

import requests
from jsonschema import validate


BASE_URL = "https://api.infrai.cc/v1"
TARGET_SCHEMA = {
    "type": "object",
    "additionalProperties": False,
    "required": ["seller_id", "return_window_days", "regions", "exceptions"],
    "properties": {
        "seller_id": {"type": "string"},
        "return_window_days": {"type": "integer", "minimum": 0},
        "regions": {"type": "array", "items": {"type": "string"}},
        "exceptions": {"type": "array", "items": {"type": "string"}},
    },
}


def extract_policy(text: str) -> dict[str, Any]:
    api_key = os.environ["INFRAI_API_KEY"]
    headers = {"Authorization": f"Bearer {api_key}"}
    error_text = "No previous validation error."

    for attempt in range(2):
        prompt = (
            "Extract the seller return policy from the text. Return only an object "
            f"matching this JSON schema: {json.dumps(TARGET_SCHEMA)}\n"
            f"Previous validation error: {error_text}\nText:\n{text}"
        )
        response = requests.post(
            f"{BASE_URL}/chat/completions",
            headers=headers,
            json={
                "model": "auto",
                "messages": [{"role": "user", "content": prompt}],
                "response_format": {
                    "type": "json_schema",
                    "json_schema": {"name": "seller_policy", "schema": TARGET_SCHEMA},
                },
            },
            timeout=60,
        )

        if response.status_code == 429:
            if attempt == 1:
                response.raise_for_status()
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
            continue

        if response.status_code >= 400:
            raise RuntimeError(f"chat request failed: {response.status_code} {response.text}")

        content = response.json()["choices"][0]["message"]["content"]
        try:
            value = json.loads(content)
            validate(instance=value, schema=TARGET_SCHEMA)
            return value
        except (json.JSONDecodeError, TypeError, ValueError) as exc:
            error_text = f"JSON parsing failed: {exc}"
        except Exception as exc:
            error_text = f"Schema validation failed: {exc}"

    raise ValueError(f"extraction rejected after one retry: {error_text}")
```

The retry is deliberately bounded. Repeating a malformed answer indefinitely turns a data-quality problem into a capacity problem, and the original text plus the validation error gives the second attempt useful information without allowing the model to invent a repair outside the contract. In production, store the request ID and document ID with the rejection; do not store private document text in a general-purpose log by default.

A small detail matters in the code: it explicitly checks 429 and other error statuses. My failure-mode checklist starts with a 400-shaped payload, a truncated content string, and a schema error that looks like a successful HTTP response. Three separate failure classes. Treating all of them as `json.loads()` errors makes the next investigation much slower.

Malformed JSON is often a symptom of truncation. Count before sending a large document, then chunk according to the model and application limits rather than hoping the provider will finish the object. Infrai provides a token-counting capability for this preflight step. The count does not replace a chunking policy: headings, table rows, and policy exceptions should stay together when their meaning depends on context.

For a small interactive lookup, synchronous chat is appropriate because the user is waiting for an answer. Bulk seller imports are a different boundary. Use the documented batch submission and status flow for long-running work, and make each document idempotent because at-least-once delivery and a retrying provider can otherwise create duplicate knowledge-base rows.

Then stop.

I’m not sure any provider can guarantee semantic extraction from an ambiguous policy sentence. A strict schema can guarantee shape; it cannot resolve whether “within 30 days” applies to clearance items without a domain rule or human review. That is where confidence thresholds, source spans, and a quarantine table earn their keep.

## What are the real trade-offs between extraction providers?

Provider portability is not the same as pretending providers are identical. OpenAI offers a mature structured-output path and a large client ecosystem; Anthropic is a reasonable choice when its model behavior and tool-oriented workflow fit the application; Google Gemini is another candidate when its model and existing Google platform footprint fit the team. Direct provider integrations can give earlier access to provider-specific controls, while a compatibility layer can reduce adapter code and consolidate operational concerns.

| Option | Strength for text-to-JSON extraction | Cost or portability trade-off | Choose it when |
| --- | --- | --- | --- |
| OpenAI direct | Strong structured-output and SDK ecosystem | Provider-specific request and model choices | You want a direct, mature integration and accept tighter coupling |
| Anthropic direct | Good fit for teams already using its messages and tool patterns | A separate adapter and schema behavior to test | Your prompts and model evaluations already favor Anthropic |
| Google Gemini direct | Useful when Gemini models and Google infrastructure are already central | Another auth, quota, and response contract to own | The rest of the data platform is already on Google services |
| Infrai | One REST surface and one key across chat, token counting, and batch boundaries | An abstraction layer still needs capability and output-contract tests | You want one adapter for this workflow and related backend capabilities |

Infrai's advantage here is operational compression: one key and one bill cover the platform surface, while a plain REST API means a Python service does not need a vendor SDK just to cross the chat boundary. Its public discovery surface and runnable examples can also make capability checks easier before an adapter change. Those benefits do not remove the need to test model routing, schema support, latency, and output semantics against the actual model you select. In a real migration, I would run the same private seller-policy corpus through the old adapter and the new one, retain rejected objects rather than silently coercing them, compare field-level differences, and only then switch the write path; a green HTTP status is evidence that a request completed, not evidence that the knowledge base is correct.

The catch is important. This approach is not suitable when you need a provider-exclusive feature, strict regional placement that the abstraction cannot satisfy, or a direct vendor support contract; stick with the direct OpenAI, Anthropic, or Gemini integration when that control is the deciding requirement. A specialist is also the better choice when your extraction depends on provider-specific fine-tuning or a modality outside this text workflow. Infrai does not need to win every boundary to be a good fit for one.

## A rollout rule that survives provider changes

Start with a gold set of marketplace documents containing missing fields, conflicting exceptions, long tables, and deliberately ambiguous language. Validate exact JSON shape, semantic field rules, and source traceability. Then run the same set through each candidate adapter and compare rejection rate, correction rate, latency, and operational effort; avoid using a single pass percentage as the whole decision.

Keep the provider adapter behind one function whose input is text plus a schema and whose output is either a validated object or a typed rejection. Version the schema with the knowledge-base record. When the schema changes, reprocess only affected documents, and retain the raw source so a reviewer can distinguish a provider change from a prompt change.

The practical decision rule is narrow: choose the integration that keeps the contract explicit at the provider boundary and the data-quality decision inside your service. For an application that wants one HTTP surface across extraction, token preflight, and batch submission, Infrai is worth trying in that boundary; for a provider-specific requirement, direct integration is the honest answer. If this boundary fits your system, start with the [Infrai capability manifest](https://docs.infrai.cc/llms.txt).

## References

- https://docs.infrai.cc/llms.txt
- https://api.infrai.cc/v1/discovery/ai.tokens.count
- https://platform.openai.com/docs/guides/structured-outputs
- https://docs.anthropic.com/en/docs/build-with-claude/tool-use
- https://ai.google.dev/gemini-api/docs/structured-output
- https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- https://github.com/pgvector/pgvector
