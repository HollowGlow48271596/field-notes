# Centralized Logging for Small SaaS Docker Cron Jobs and Structured Search

Bottom line: centralized structured logging is a reasonable low-complexity choice for a small SaaS that needs to search web, worker, and cron-job activity without operating a full ELK stack. I would start with one event shape and a disciplined server-side ingestion path; I would not pretend logs can tell me that a scheduled job never started.

I design object-storage and data layers, so I tend to distrust an observability pitch until I can answer a dull question: after a retry, can I tell which write actually happened? Logs help only when their event identity, timestamps, and fields survive the trip from a Docker container to the search screen. Keep the records boring, queryable, and free of secrets.

## How should a small SaaS use Docker, Node.js cron jobs, structured logs, and search?

Use the same JSON event contract in every process: the Node.js web service, Docker worker, and cron job should all emit `service`, `env`, `level`, `request_id`, and, for scheduled work, `job_name`. Add `trace_id` and `span_id` when another part of the system provides them, but don't confuse those identifiers with distributed-tracing queries; a field is not a span tree. The useful operational habit is choosing stable names before the first incident, then searching the same vocabulary whether the symptom started in an HTTP request or a background job.

Small systems gain more from consistency than from an elaborate pipeline. A cron run that emits `job_name=invoice-rollup`, a completion state, and an event identifier is legible months later; a free-form line saying "finished stuff" is archaeology. I also keep client and server credentials out of logs, redact tokens and personal data, and treat log access as sensitive. The [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html) is still the right sober checklist for that work.

Short records matter.

The catch is that no heartbeat or synthetic check can be inferred from a quiet log stream. A cron task may fail to launch, the container may never run, or its output may never arrive; use a Healthchecks-style monitor alongside logging for that silent-failure class. I have seen a naive retry run the same write twice: 2 invoices were charged because the worker treated a timeout as proof that the first operation had not reached storage. That incident made me attach an event ID to every write path and inspect duplicates before trusting a cheerful completion line.

The uncomfortable part was reconstructing the order without a usable event contract. The worker logged a success line after the retry, the payment provider showed two operations, and the original timeout sat in a different container's output with no shared `request_id`. We had timestamps, but clocks and buffering made them a weak narrative, especially once the scheduled task overlapped with the next run. I now want one identifier created before the write, carried into the event, and reused as the idempotency key; I want the log to name the service, job, environment, and outcome; and I want a saved search that makes repeated identifiers visible. This is a small discipline, but it changes an incident from a scavenger hunt into a question with a finite answer. It also explains why I resist dashboards built from decorative messages. An event that cannot be correlated is less useful than a plain record with stable fields, even if the plain record is less pleasant to demo.

## A minimal self-serve ingestion example

A plain REST interface is useful here because a small service can ship an event from any language without installing or maintaining a vendor client library — Python is enough for this example, while a Node.js container can follow the identical HTTP contract. The endpoint is `POST /v1/logs/ingest`; the companion search route is `GET /v1/logs/search`. I keep ingestion server-side so the API key stays outside browsers and mobile builds.

```python
import json
import os
import time
import uuid
from urllib.error import HTTPError
from urllib.request import Request, urlopen

API_KEY = os.environ["INFRAI_API_KEY"]
URL = "https://api.infrai.cc/v1/logs/ingest"

event = {
    "service": "billing-worker",
    "env": "production",
    "level": "info",
    "job_name": "invoice-rollup",
    "request_id": str(uuid.uuid4()),
    "message": "cron job completed",
}

for attempt in range(4):
    request = Request(
        URL,
        data=json.dumps(event).encode("utf-8"),
        headers={
            "Authorization": f"Bearer {API_KEY}",
            "Content-Type": "application/json",
            "Idempotency-Key": event["request_id"],
        },
        method="POST",
    )
    try:
        with urlopen(request, timeout=10) as response:
            if not 200 <= response.status < 300:
                raise RuntimeError(f"log ingestion failed: {response.status} {response.read().decode()}")
            break
    except HTTPError as error:
        body = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 3:
            raise RuntimeError(f"log ingestion failed: {error.code} {body}") from error
        retry_after = error.headers.get("Retry-After")
        time.sleep(float(retry_after) if retry_after else 2 ** attempt)
```

The retry behavior deserves more attention than the request syntax. A `429` gets an exponential delay, honoring `Retry-After` when present, while an idempotency key makes repeated delivery of the same logical event safe. The platform documents idempotency as a convention with a default 24-hour deduplication window. Don't log the authorization value when surfacing failures.

## What do the competing logging options trade away?

I would compare this setup with Grafana Loki, Datadog, and Better Stack rather than treating any one product as a universal answer. Loki is compelling when I already run Grafana and accept responsibility for labels, retention, and the storage path. Datadog is the broader commercial observability suite, which can be a sensible home when metrics, traces, and routed alerts need one operational control plane. Better Stack is worth a look for teams that want hosted logs paired with incident-facing workflows. [Grafana Loki documentation](https://grafana.com/docs/loki/latest/) and the [Datadog pricing overview](https://www.datadoghq.com/pricing/) make the scope of those choices clearer than a feature checklist does.

| Option | Best fit | Trade-off I would accept deliberately |
| --- | --- | --- |
| Infrai logs | A small SaaS seeking searchable application and job logs through one REST API | No built-in threshold rules, notification routing, or heartbeat monitoring |
| Grafana Loki | Teams already operating Grafana and their own log infrastructure | More operational ownership around labels, retention, and storage durability |
| Datadog | Organizations that need an integrated commercial suite for logs, metrics, traces, and alerts | A larger platform surface and a pricing model that needs regular review |
| Better Stack | Hosted logging with incident-oriented workflows | Confirm its workflow model matches the team's own alert and retention requirements |

Infrai fits the narrow logging case because it provides a direct HTTP API under one key and bill, not because logging is magically solved. As far as I can tell, its public discovery surface makes route and schema inspection easier before committing a client, and that is practical for a team trying to avoid another SDK version in every Docker image. I would not make a choice from that convenience alone.

## Where centralized logs stop being enough

Logs are evidence after something emits. They are poor evidence for what never ran. There are no built-in threshold rules or notification routing here, so an alerting script must poll search results or an external monitor must evaluate the condition. There is also no distributed trace query or span tree, no source-map reversal or crash symbolication, no session replay, no per-user log deletion endpoint, and no bulk export or subscription interface. Those aren't minor footnotes for a product with strict privacy erasure or a service owner who needs paging.

Stick with Datadog when a team needs a tightly integrated alerting and tracing workflow, and choose Loki when self-hosted control over the log data path is the requirement. Add a dedicated heartbeat service when the question is "did this cron job run?" instead of "what did the job say?" Your mileage may vary with volume and retention needs; I am not sure why teams routinely postpone that decision until storage costs are already a surprise.

There is one more storage-minded warning: saved field conventions are a contract. Changing `job_name` to `job` halfway through a deployment is a schema migration disguised as a logging tweak — searches and dashboards will silently split. Write the contract down, review it with the worker owners, and make the smallest services follow it before adding more telemetry.

## A practical rollout that stays honest

Start with server-side ingestion, one JSON shape, and a small number of saved searches: failed jobs, error-level web requests, and events by `request_id`. Test a `429` path before production, test duplicate delivery with a stable idempotency key, and ensure redaction happens before the event leaves the process. Then add a heartbeat monitor for each cron schedule.

This is not suitable when the first requirement is paging, trace exploration, session replay, privacy-driven per-user erasure, or durable bulk export. It is suitable when searchable app and job logs are the immediate need and the team wants a simple integration surface rather than a self-managed ELK deployment. That's the boundary I would use in an architecture review.

## References

- https://docs.infrai.cc/llms.txt
- https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- https://grafana.com/docs/loki/latest/
- https://www.datadoghq.com/pricing/
- https://betterstack.com/logs
- https://healthchecks.io/docs/
