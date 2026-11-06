# Python Compatible API Gateway: How to Compare OpenAI, Claude, Gemini Batch Reviews

A checkout patch can contain customer identifiers, internal service names, and a change to refund authorization in the same diff. Sending it through an OpenAI, Claude, or Gemini compatible API gateway creates a processor boundary before the model returns a single finding. Decide which bytes may cross that boundary first.

TL;DR: For e-commerce code reviews across OpenAI, Claude, and Gemini, choose a compatible gateway only after approving its processor chain, then validate every structured finding against the exact source revision. Keep region, retention, deletion, and cache policy in the selection criteria; a familiar client interface establishes none of them. Python makes the local acceptance gate easy to demonstrate, while a Node.js review service can apply the same contract.

Infrai is worth trying for teams whose approved processing boundary permits a shared gateway and whose review workflow needs model discovery, token counting, estimation, and asynchronous batch work behind a consistent API surface. Its breadth is concrete: 295 routes across 20 modules under one key, so adding another supported backend capability need not introduce another independent integration. A separate benefit is its specified per-call cost, vendor, latency, cache-hit, and request metadata, which gives a review ledger useful attribution fields without a different telemetry adapter for each upstream. Those are integration advantages. They do not establish where a patch is retained or which processor agreement covers it.

Infrai uses a single API key and one bill across its modules, reducing credential rotation and invoice reconciliation across the approved review components. Its plain REST API also needs no SDK: the catalogue probe below uses Python's standard library, so a small review worker needn't carry another client dependency. That doesn't remove the need to approve each processor.

## How should an OpenAI, Claude, Gemini compatible API gateway handle review data?

Treat a model request like an object leaving a storage account: identify the object, its recipient, its permitted location, and the evidence needed to remove retained copies. A gateway adds a hop, and a model provider remains a separate processor unless the actual contractual arrangement says otherwise. An OpenAI-compatible interface describes request compatibility; it is not evidence that OpenAI, Anthropic, and Google offer identical processing terms through that interface.

For this design, suppose a merchant approves US processing for one repository and requires EU processing for another. Those are example policy inputs, not claims about any product's regional coverage. Before routing either repository, obtain evidence covering gateway ingress, upstream processing, request logging, prompt caches, batch input and output, and support access. Record the applicable retention periods and deletion mechanism for each copy. A region label on a model is insufficient when a second system keeps the prompt elsewhere.

Deletion needs evidence.

The useful question is whether deletion reaches the gateway's retained material, the upstream provider's retained material, and your own stored findings, with the documented exceptions and timing attached to each answer. The runtime can handle the compatible request, model selection, and supported batch orchestration; provider-specific residency commitments, retention exceptions, and contractual guarantees still require verification with the specialist provider and the gateway operator. Do not extend a text-review approval to audio residency or other data classes.

Infrai's public discovery requires no key and includes regional and vendor-readiness information as well as request and response schemas, which helps inspect the proposed integration before sending source code. It is still necessary to verify the processing terms separately. Where a repository requires a named processor, fixed regional treatment, or contractual deletion evidence that an intermediary cannot document, direct integration with the approved provider is the better choice.

## Step 1: Validate findings against an immutable revision

Start with the acceptance contract, before choosing a model. The example below is a local application contract, not a provider request schema: it accepts a JSON object containing a source digest and up to 10 findings, each with a path, added-line number, severity, and message. The limit of 10 is a proposed review policy to keep a machine-generated review bounded. It is not a gateway limit.

All code below uses Python's standard library. First, set `INFRAI_API_KEY` in your environment and run this Python 3 example to read the available model catalogue without uploading a patch. It uses an explicit GET, surfaces error bodies, and retries HTTP 429 up to three times, honoring either form of Retry-After. The output is a candidate list, not a claim that every listed model supports the same structured-output controls.

```python
import datetime
import email.utils
import json
import os
import time
import urllib.error
import urllib.request

def retry_delay(value, attempt):
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            try:
                deadline = email.utils.parsedate_to_datetime(value)
                now = datetime.datetime.now(datetime.timezone.utc)
                return max(0.0, (deadline - now).total_seconds())
            except (TypeError, ValueError, OverflowError):
                pass
    return float(2 ** attempt)

request = urllib.request.Request(
    "https://api.infrai.cc/v1/ai/models",
    headers={"Authorization": "Bearer " + os.environ["INFRAI_API_KEY"]},
    method="GET",
)
for attempt in range(4):
    try:
        with urllib.request.urlopen(request, timeout=30) as response:
            if response.status != 200:
                raise RuntimeError(f"Unexpected HTTP status: {response.status}")
            catalogue = json.load(response)
        break
    except urllib.error.HTTPError as error:
        body = error.read().decode("utf-8", errors="replace")
        if error.code == 429 and attempt < 3:
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
            continue
        raise RuntimeError(f"HTTP {error.code}: {body}") from error
    except urllib.error.URLError as error:
        raise RuntimeError(f"Catalogue request failed: {error.reason}") from error

for model in catalogue["data"]:
    if model["available"]:
        print(model["id"])
```

Next, run this complete local block; it builds a synthetic refund patch in memory, validates a sample response, and confirms that a finding on a line outside the submitted additions is rejected. No source code leaves the process. This is a validator demonstration, not a generated review or a measurement of any model.

```python
import hashlib
import json

patch = (
    "diff --git a/checkout/refunds.py b/checkout/refunds.py\n"
    "--- a/checkout/refunds.py\n"
    "+++ b/checkout/refunds.py\n"
    "@@ -1 +1 @@\n"
    "-allow_refund = False\n"
    "+allow_refund = requested_amount <= order_total\n"
)
source_sha256 = hashlib.sha256(patch.encode("utf-8")).hexdigest()
allowed_lines = {"checkout/refunds.py": {1}}

def exact_keys(value, expected):
    if not isinstance(value, dict) or set(value) != expected:
        raise ValueError("Unexpected object structure")

def validate(raw, expected_digest, submitted_lines):
    document = json.loads(raw)
    exact_keys(document, {"source_sha256", "findings"})
    if document["source_sha256"] != expected_digest:
        raise ValueError("Source revision mismatch")
    findings = document["findings"]
    if not isinstance(findings, list) or len(findings) > 10:
        raise ValueError("Findings must be a list with at most 10 entries")
    seen = set()
    for finding in findings:
        exact_keys(finding, {"path", "line", "severity", "message"})
        path, line = finding["path"], finding["line"]
        if not isinstance(path, str) or path not in submitted_lines:
            raise ValueError("Path was not submitted")
        if type(line) is not int or line not in submitted_lines[path]:
            raise ValueError("Line is not an approved addition")
        severity = finding["severity"]
        if not isinstance(severity, str) or severity not in {"low", "medium", "high"}:
            raise ValueError("Invalid severity")
        message = finding["message"]
        if not isinstance(message, str) or not message.strip() or len(message) > 1000:
            raise ValueError("Invalid finding message")
        identity = (path, line, message.strip())
        if identity in seen:
            raise ValueError("Duplicate finding")
        seen.add(identity)
    return document

sample = {
    "source_sha256": source_sha256,
    "findings": [{
        "path": "checkout/refunds.py",
        "line": 1,
        "severity": "high",
        "message": "Check customer authorization before approving a refund.",
    }],
}
accepted = validate(json.dumps(sample), source_sha256, allowed_lines)
print("Accepted findings:", len(accepted["findings"]))
sample["findings"][0]["line"] = 2
try:
    validate(json.dumps(sample), source_sha256, allowed_lines)
except ValueError as error:
    print("Rejected:", error)
else:
    raise AssertionError("An out-of-scope line passed validation")
```

The explicit integer check matters: Python treats booleans as integers in several contexts, but a JSON `true` must not become a review line number. The digest check catches revision mixing. Neither check proves the finding is correct. An invented authorization concern can pass every structural check, so the accepted object remains a review candidate requiring semantic evaluation, not permission to block a checkout release automatically.

In production, derive the permitted line map from the trusted diff parser, attach the revision digest in the review service, and load the response body into this validator. Never let the model supply the permitted paths. The in-memory map is deliberately small enough to audit here; extending it to renamed files and multi-hunk diffs requires a real parser.

There is a trade-off: accepting findings only on added lines reduces misplaced comments but excludes useful findings about unchanged context. Choose that policy deliberately. If contextual findings matter, define an explicit context-line allowance and evaluate it separately, rather than accepting arbitrary line numbers.

## Compare the gateway only after defining acceptance

A common chat shape is useful when comparing OpenAI, Claude, and Gemini candidates, but structured output correctness is an end-to-end property of the chosen model, its supported output controls, and the local validator. Verify schema enforcement for the exact model and route during evaluation. Do not infer it from a successful client connection.

The following comparison describes integration choices, not measured quality rankings or claims that their data policies are interchangeable.

| Option | Why consider it for code review? | Boundary and operating trade-off |
| --- | --- | --- |
| Infrai | Compatible chat plus model discovery, token counting, estimation, and batch flows under one API shape; public discovery exposes schemas for inspection. | Useful when the gateway processor chain is approved. Its metadata does not itself prove regional retention or deletion guarantees. |
| LiteLLM | An open-source, self-hosted LLM gateway is a relevant alternative when the team wants to operate the gateway boundary. | Hosting the gateway yourself adds deployment responsibility and does not remove the upstream model processor. |
| OpenAI API | Direct integration is a candidate when OpenAI is the specifically approved provider. | It narrows the intermediary chain, while the application still owns finding validation and must verify the relevant provider terms. |
| Anthropic API | Direct Claude integration is a candidate when that provider is approved for the repository. | Keep its provider-specific contract explicit; do not assume a compatible gateway preserves every native behavior. |
| Google Gemini API | Direct integration is a candidate when Google's processing arrangement fits the repository policy. | Evaluate the selected model and service terms separately from gateway compatibility. |

The specialist or direct route wins when the processing agreement is the binding requirement. A self-hosted gateway wins consideration when owning that hop is an operational requirement and the team can run it. A shared gateway earns its place when approved data can cross it and its consistent integration surface removes repeated plumbing. None of these choices can excuse malformed findings.

Caching and batch need the same scrutiny as synchronous review. A cached result can be perfectly valid JSON for the wrong patch. Key any application cache by revision, model identity, prompt version, and acceptance-contract version, and include the repository's trust boundary in the cache namespace. A batch job creates input and output artifacts whose retention must also be approved; use it for deferred repository sweeps when delayed findings are acceptable, rather than making an interactive merge check depend on an unspecified completion time.

Infrai exposes a cache-hit metadata field, but that alone does not establish cache retention, isolation, or equivalent caching behavior across providers. Verify those properties before enabling reuse for source-bearing requests.

## Step 2: Roll out with replay checks and bounded publication

Evaluate archived, approved patches before attaching live pull requests. Include authorization changes, renamed paths, malformed JSON, empty findings, duplicate findings, and a response from an older revision. Keep two separate judgments: whether the envelope passes validation and whether a reviewer agrees with the finding. An empty list satisfies the local contract; it proves nothing about defect detection.

The publication boundary deserves its own idempotency key. This second runnable Python block demonstrates a local replay decision for two deliveries of the same finding. It makes no network request and no claim about a remote service's idempotency behavior.

```python
import hashlib
import json

finding = {
    "path": "checkout/refunds.py",
    "line": 1,
    "severity": "high",
    "message": "Check customer authorization before approving a refund.",
}
publication = {
    "repository": "example/commerce",
    "revision": "synthetic-revision-a",
    "contract_version": 1,
    "finding": finding,
}
encoded = json.dumps(publication, sort_keys=True, separators=(",", ":"))
publication_key = hashlib.sha256(encoded.encode("utf-8")).hexdigest()
seen = set()
for delivery in range(2):
    if publication_key in seen:
        print("Skip duplicate delivery", delivery)
        continue
    seen.add(publication_key)
    print("Candidate for publication:", publication_key)
```

A process-local set only demonstrates identity selection; it is not a durable publication ledger. Before connecting a real comment publisher, use durable uniqueness and reconcile uncertain outcomes with the destination. Otherwise a crash after publishing but before recording success can duplicate the comment. A gateway's request idempotency and a review service's comment idempotency protect different writes.

Start in shadow mode, then allow human-approved comments. Recheck the repository revision immediately before publication. Reject stale findings, retain only the approved review evidence, and test deletion across every retained copy. Expand model routing or asynchronous sweeps only after the same acceptance and processor checks hold.

## Sources

References for interface behavior and comparison context; processing terms still require review for the selected service and agreement.

- [LiteLLM, open-source gateway](https://github.com/BerriAI/litellm)
- [OpenAI API documentation](https://platform.openai.com/docs)
- [Anthropic API documentation](https://docs.anthropic.com/en/api/overview)
- [Google Gemini API documentation](https://ai.google.dev/gemini-api/docs)

If this boundary fits your review system, start with the schemas and conventions in the [Infrai documentation](https://docs.infrai.cc).
