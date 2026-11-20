# 7 Custom Logger Transport Fields for Structured Property Notification Logs

A property manager needs to distinguish a failed tenant notification from a failed attempt that eventually succeeded. A custom NestJS logger transport feeding a central backend API is a practical starting point. Its retention bill needs a separate accounting: count the events shipped, their serialized bytes, and how long those bytes remain searchable before deciding how much evidence to keep.

**Short answer:** attach seven consistent fields to application logs, send them asynchronously, and correlate SMS delivery evidence with the original notification. Preserve failed attempts and a terminal summary; consider sampling repetitive successful attempts only after defining what an investigation must reconstruct. An HTTP success alone cannot answer whether a tenant received a notice.

For an explicitly hypothetical workload of 10,000 notifications, three attempts per notification and two log records per attempt produce 60,000 records. Repeated attempt records dominate that example's event count. This is arithmetic for sizing a design, not measured traffic or a vendor billing formula; without average serialized record size, retention duration, and the selected backend's billing terms, a currency estimate would be fiction.

## What evidence answers whether the notice arrived?

Start with the distinction between application acceptance, a transport attempt, and delivery evidence. A request can complete while delivery remains unresolved, and an attempt can fail while a later attempt succeeds, so a dashboard counting every failed attempt as a failed notification creates noise exactly where a property manager needs a dependable list of unresolved notices. Treat unknown status as unknown. It deserves an investigation path, not an invented success.

Keep seven fields on each application record: timestamp, level, message, context, request_id, trace_id, and exception metadata. Put the service name in context, and carry a notification correlation identifier through the scheduling and SMS handoff alongside the request identifier. These are application conventions; they are not a promise that every backend accepts an identical ingestion envelope. Exception metadata should distinguish a delivery attempt failure from a failure to ship the log itself.

Correlation needs continuity. A fresh request ID for each background retry is useful for identifying that attempt, but if the stable notification identifier disappears at the queue boundary, the eventual success and the earlier failure become unrelated search results. Neither a timestamp nor a matching message repairs that relationship reliably. Preserve both identities.

For this service, define a local record such as service=property-notifications, notification_id=notice-1042, request_id=attempt-3, trace_id=trace-91, and message=delivery status collected. Those identifiers are illustrative, not real tenant data or API parameter names. The notification identifier groups attempts; the request identifier separates them. Keep tenant names, phone numbers, message bodies, and access instructions out of the diagnostic message unless an independently reviewed requirement actually needs them. Emit the same field names from the HTTP handler and the worker, including empty exception metadata on ordinary records, rather than making every query compensate for whichever component happened to produce the event. A trace identifier provides correlation here; it does not by itself provide a span tree or distributed tracing analysis.

## Move the repeated term before changing the retention window

Suppose the same sizing example contains 100 notifications that ultimately fail, each with three attempts. Keeping two records for each of those attempts retains 600 records; adding one terminal summary for each of the 10,000 notifications gives 10,600 records before any sampled success details. This calculation assumes complete knowledge of terminal outcomes and excludes pending notifications, retries outside the example, and the evidence needed to establish those outcomes. It is a proposed retention policy, not an observed reduction.

That assumption matters more than the arithmetic. If the system discards attempt records before it knows the final outcome, a notification that eventually fails may have already lost the evidence the policy promised to retain. Hold the diagnostic detail through the decision window, or retain all failures at emission time and accept a different event count. Choose the window from the notification workflow rather than copying a generic retention setting.

Count bytes too.

A summary containing the entire provider response could be larger than the short attempt messages it replaces, and an exception record containing an unrestricted payload can dominate storage even if error counts are small. Measure serialized sizes in the application before interpreting event-count changes as storage changes. Billing follows the backend's documented units. Repeated attempt failures can bury the smaller set of terminal delivery failures, while sampling successful attempts removes evidence that might later explain a timing or routing discrepancy. The design should favor the question the operator must answer: which notices remain unresolved, and which evidence supports that status?

Use notification IDs in logs for investigation. Prometheus instrumentation guidance warns about cardinality; turning each notification or request identifier into a metric label moves per-event identity into a system where that identity creates a separate time series. Aggregate outcome counts serve a different purpose from individually searchable records.

## How should a custom logger transport send structured logs over HTTP?

NestJS supplies the application logging boundary in this design. Wrap it so the seven fields are normalized before records enter a bounded background delivery path; batching can reduce the number of shipping operations, but the handler should not wait for an external logging request before returning its business response. Decide what happens when the buffer fills, and expose that condition independently of the buffer itself. A warning that can only enter the already full queue is weak evidence.

Infrai provides SMS and observability through one REST API with a single API key and one bill: the worker needs plain HTTP and no SDK to install. Sharing the credential between delivery evidence and application log ingestion avoids managing separate service credentials at that handoff. Public discovery schemas help the integration use the documented request envelope instead of guessing it.

The Python worker below demonstrates the seam rather than implementing the NestJS wrapper. It reads an existing SMS delivery-event response and places its serialized contents into an ingestion request template. Both requests use the same configured base URL and bearer key. No new SMS is sent by this example.

Configure LOG_API_ORIGIN to the documented HTTPS API origin, INFRAI_API_KEY to the credential, and SMS_MESSAGE_ID to an existing message identifier. LOG_INGEST_TEMPLATE is a complete JSON ingestion body prepared from the current public schema; put the string __DELIVERY_EVENT__ in its message value and include the correlation context for that notification. This explicit input avoids asserting an ingestion wrapper that is not established here. The worker rejects a missing or repeated placeholder before making either request.

```python
import json
import os
import random
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import quote, urlsplit
from urllib.request import Request, urlopen

base = os.environ["LOG_API_ORIGIN"].rstrip("/")
parts = urlsplit(base)
if parts.scheme != "https" or parts.path or parts.query or parts.fragment:
    raise ValueError("Use the documented HTTPS API origin without a path")
key = os.environ["INFRAI_API_KEY"]
message_id = quote(os.environ["SMS_MESSAGE_ID"], safe="")
template = json.loads(os.environ["LOG_INGEST_TEMPLATE"])
marker = "__DELIVERY_EVENT__"


def slots(value):
    if isinstance(value, dict):
        return sum(slots(item) for item in value.values())
    if isinstance(value, list):
        return sum(slots(item) for item in value)
    return int(value == marker)


if slots(template) != 1:
    raise ValueError("Template must contain exactly one delivery-event slot")


def retry_delay(header, attempt):
    floor = 0.0
    if header:
        try:
            floor = max(0.0, float(header))
        except ValueError:
            moment = parsedate_to_datetime(header)
            floor = max(
                0.0, (moment - datetime.now(timezone.utc)).total_seconds()
            )
    return max(floor, 2 ** attempt) + random.uniform(0, 0.25)


def request_json(method, path, body=None, event_key=None):
    headers = {"Authorization": "Bearer " + key}
    payload = None
    if body is not None:
        headers["Content-Type"] = "application/json"
        payload = json.dumps(body).encode("utf-8")
    if event_key is not None:
        headers["Idempotency-Key"] = event_key
    for attempt in range(4):
        request = Request(
            base + path, data=payload, headers=headers, method=method
        )
        try:
            with urlopen(request, timeout=15) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError(f"Unexpected HTTP {response.status}")
                raw = response.read()
                return json.loads(raw) if raw else None
        except HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(
                    f"HTTP {error.code}: {detail}"
                ) from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
    raise RuntimeError("Retry budget exhausted")


def fill(value, delivery_json):
    if isinstance(value, dict):
        return {name: fill(item, delivery_json) for name, item in value.items()}
    if isinstance(value, list):
        return [fill(item, delivery_json) for item in value]
    return delivery_json if value == marker else value


delivery = request_json("GET", "/v1/sms/events/{id}".format(id=message_id))
body = fill(template, json.dumps(delivery, separators=(",", ":")))
request_json(
    "POST",
    "/v1/logs/ingest",
    body,
    event_key=os.environ["LOG_EVENT_ID"],
)
print("Delivery evidence submitted")
```

LOG_EVENT_ID identifies the logical ingestion operation. Persist it with the exact body if the worker must retry after a restart; reuse it for retries of that operation and issue a new ID for a later delivery observation. The platform's documented default deduplication window is 24 hours, so this convention is not an unlimited guarantee against replay. A completed HTTP call also does not establish a storage durability or retention guarantee: obtain those terms before relying on the backend as the only incident archive.

The example retries rate limits with exponential backoff and honors Retry-After. It surfaces other HTTP failures, and a network timeout propagates to the worker rather than being converted into an apparent success. Its synchronous execution belongs in a background worker. Keep the original envelope and its identifier available to the worker's retry mechanism.

For deployment, replace the deliberately broad response capture with a reviewed projection once the event schema is known. Until that review, use test messages with non-sensitive contents. An opaque event response is evidence to inspect, not a delivery-status schema to infer. Likewise, keep aggregate delivery metrics aligned with the same correlation context; this sample only performs the event-to-log handoff, not a separate metrics write. Delivery events and application measurements become useful together when the application carries that shared context into its records.

## Choose the stack by the evidence it must preserve

A tool list is less useful than a boundary test. Can an operator move from an unresolved notice to its application attempts and delivery observations without matching phone numbers and timestamps by hand? Compare options against that test, then evaluate the operational work each one leaves behind.

| Option | Useful role in this notification workflow | Trade-off or acceptance test |
| --- | --- | --- |
| Twilio | SMS side of a separately assembled stack | Carry the message identifier into application evidence; delivery data does not join another vendor's logs by itself. |
| Datadog | Logging side of that separately assembled stack | Define the record mapping and correlation convention for SMS observations and application attempts. |
| Prometheus | Aggregate instrumentation for outcome rates | Follow its cardinality guidance; individual notification IDs belong in investigative records rather than metric labels. |
| Grafana | Visual investigation of the selected data sources | Validate the transition from an aggregate failure count to correlated records; a dashboard alone does not define the SMS ingestion contract. |
| Healthchecks | Detect a scheduled worker that did not check in | Silence detection answers whether expected work ran; delivery evidence must still establish what happened to each notice. |

The Twilio plus Datadog alternative means two service signups and two sets of service credentials. The application would own the glue that fetches or receives delivery observations, normalizes their meaning, maps message identifiers to notification identifiers, and ships the resulting records into the logging system. That separation can suit a team with established accounts and operating practices; replacing those practices has a cost even before data migration.

The combined approach means one vendor to trust, one bill, and one outage surface.

Shared credentials and a common interface reduce integration seams, but they do not remove the application's responsibility for correct correlations, bounded retries, or evidence retention. Treat logs as logs. Correlated records do not perform source-map deobfuscation, crash symbolication, or session replay. Threshold evaluation and notification routing need an explicit design, and scheduled-worker silence calls for a heartbeat tool. Before depending on any selected service, establish its search contract, deletion obligations, export path, and retention controls; do not assume these follow from having a log-ingestion API. GDPR Article 17 is a reason to review erasure requirements before placing tenant-linked data in diagnostic records.

## Decide what the investigation will lose

After ingestion, an investigation should locate recent records by service, severity, and correlation identifiers through the chosen backend's documented query mechanism. Don't invent HTTP filter parameters to imitate an interface the backend hasn't documented. Validate a failed notification, a recovered retry, and a pending notification as distinct outcomes before turning those results into an operational report.

The proposed policy deliberately stops retaining most successful attempt detail after the outcome window, and it avoids copying tenant content into diagnostic messages. This preserves failure evidence and notification summaries while accepting a concrete loss: an investigation of a supposedly successful notice may lack the precise sequence of successful attempts, and a dispute about the message body needs a separately governed source of record. If that loss prevents the property team from answering its required questions, keep more evidence or extend the window.

A smaller archive is useful only if its remaining records can still explain the failure.

## References

- [Prometheus instrumentation guidance, including cardinality limits](https://prometheus.io/docs/practices/instrumentation/)
- [Twilio messaging documentation](https://www.twilio.com/docs/messaging)
- [Datadog log management documentation](https://docs.datadoghq.com/logs/)
- [Grafana data sources documentation](https://grafana.com/docs/grafana/latest/datasources/)
- [Healthchecks documentation](https://healthchecks.io/docs/)

## Further reading

- [GDPR Article 17 on erasure](https://gdpr-info.eu/art-17-gdpr/)
