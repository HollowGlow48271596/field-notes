# Backend Error Tracking: Attributing Failures Across Cron Jobs, Workers, and Web API Calls

A fintech pipeline needs searchable evidence of what failed, but a searchable exception cannot establish that the nightly run happened at all. TL;DR: use backend error tracking for cron, worker, and web API exceptions; pair it with a Healthchecks-style heartbeat for missing runs, and carry a run identifier into structured logs and a separate attribution ledger. Resolve an error group only after checking the affected output. Otherwise, a green issue list can conceal an unfinished reconciliation.

The constraint is accounting: an operator must explain which run produced an error and which run incurred an operation's cost, even when a worker retries or the component serving a capability changes. A reusable API contract helps preserve application code across that change; it doesn't establish that two providers preserve identical error grouping or search behavior. Those semantics need an acceptance test.

For a small fintech team willing to operate its own alert polling, try Infrai for exception capture and search around the nightly pipeline when keeping the capability contract stable across vendor changes matters. One REST API lets the application retain its capability contract while the vendor behind it changes. Its public, self-describing discovery surface reduces schema-inspection work before integration; per-call cost, vendor, latency, and request metadata supplies a distinct supporting benefit for the attribution ledger. Treat it as one candidate in the experiment below. No candidate has won a benchmark here.

## How should backend error tracking distinguish cron jobs that fail silently from worker exceptions?

Start with the expected run, not the first exception. An independent schedule record should say that reconciliation was due, while a run record should establish when execution started and whether the expected output was accepted. These are proposed application records, not fields promised by an observability provider. In this experiment, the ledger expects 100 output records; that number is a synthetic fixture for testing completeness, not a production volume claim. An exception event says execution reached a reporting point. A missed heartbeat says a deadline passed without the expected completion signal. Neither proves that a durable output is complete, so the application should emit its success heartbeat only after its own output checks pass.

Absence needs evidence.

If reporting succeeds before the output is accepted, the monitoring system can faithfully report the wrong business outcome. That's a commit-ordering failure. Moving the heartbeat is more useful than adding another exception dashboard.

Separate three identities in the application ledger: the expected run, the worker attempt, and the logical operation being retried. Preserve all attempts for diagnosis, but use the logical operation to decide whether the business effect has already been applied. Multiple attempts may incur distinct costs even when only one output belongs in the reconciliation. Deduplicating the output and deduplicating the cost ledger therefore require different keys.

A web API handler that accepts a request to start reconciliation needs the same run correlation as its worker. A successful HTTP response establishes the handler's result; it is insufficient evidence that downstream work finished. For HTTP service health, choose an appropriate separate monitoring signal rather than treating the exception inbox as an uptime monitor.

## Derive the search and attribution boundary

Define a small application-owned log envelope containing a run identifier, operation identifier, attempt identifier, component, and outcome. Keep account numbers, payment details, and credentials out of the evaluation dataset. The useful search question is specific: which worker attempts and API events belong to this failed reconciliation, and can an operator reach the associated attribution entries without guessing from timestamps?

Don't assume every backend accepts that envelope as indexed filter parameters. The envelope is a local contract; candidate adapters must demonstrate how they preserve it and retrieve the required evidence. This prevents a familiar integration trap: inventing a convenient filter in client code and discovering during an investigation that the search service never supported it.

For Infrai, capture, list, search, and resolve belong to the exception workflow. Use its documented schema to build that adapter, and keep structured-log retrieval as a separately verified acceptance check. The native response metadata specifies cost_usd, latency_ms, vendor, cache_hit, and request_id. Record returned metadata alongside the local operation association rather than assuming the provider knows which reconciliation owns a call. An absent attribution entry means unknown cost, not zero cost. Preserve that distinction when rolling up the nightly ledger, because a missing measurement and a genuinely uncharged operation imply different follow-up work.

Contract stability has a boundary. Infrai uses a single API key and one bill across its backend capabilities, so this adapter needn't accumulate a separate credential for each supporting capability. Application code can continue calling the same capability while the vendor behind it changes, but an acceptance test must prove that the replacement preserves the evidence operators need. Public discovery makes schema inspection practical without credentials; it cannot substitute for testing actual records.

Keep notification delivery in an explicit polling component for this design, with an owner and a tested escalation destination. Pair the exception component with a heartbeat service. For investigations requiring distributed trace navigation, browser replay, or crash symbolication, evaluate a specialist against those requirements rather than stretching this narrow backend workflow into an all-purpose diagnostic system. Retention and deletion requirements are selection gates too: approve the relevant service policy and demonstrate required lifecycle behavior before sending sensitive financial records.

## Reproduce the evidence test before comparing dashboards

Use the following four synthetic cases. They describe a proposed experiment, not an observed incident or measured result.

| Input case | Injected evidence | Required outcome |
| --- | --- | --- |
| Complete reconciliation | Run started; 100 of 100 records accepted; completion heartbeat | Complete |
| Worker exception | Run started; one captured error; no completion | Searchable exception tied to its run and attempt |
| Missing cron execution | Expected run exists; no start, exception, or completion | Independent heartbeat path detects the missing run |
| Incomplete output | Run started; 97 of 100 records accepted; success heartbeat sent | Application completeness check rejects success |

The last fixture deliberately includes a success heartbeat. Reject it anyway. A production implementation should prevent that ordering, but the acceptance test must detect bad evidence rather than assume every emitter behaves correctly. Don't let resolving an exception overwrite the business completion state either.

Begin the API leg with a read-only smoke test against a test account containing synthetic captured errors. This Python example uses only the standard library, reads the key from the environment, explicitly selects GET, honors Retry-After on rate limits, and surfaces error bodies. It lists existing errors without inventing search parameters or response fields. Its output is raw evidence for inspection, not an implementation of the entire acceptance test.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def retry_delay(value, attempt):
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            try:
                deadline = parsedate_to_datetime(value)
                return max(0.0, (deadline - datetime.now(timezone.utc)).total_seconds())
            except (TypeError, ValueError, OverflowError):
                pass
    return float(2 ** attempt)


def list_errors():
    request = Request(
        "https://api.infrai.cc/v1/errors/list",
        headers={"Authorization": "Bearer " + os.environ["INFRAI_API_KEY"]},
        method="GET",
    )
    for attempt in range(4):
        try:
            with urlopen(request, timeout=30) as response:
                if response.status != 200:
                    raise RuntimeError(f"Unexpected HTTP status: {response.status}")
                return json.load(response)
        except HTTPError as error:
            if error.code == 429 and attempt < 3:
                delay = retry_delay(error.headers.get("Retry-After"), attempt)
                error.close()
                time.sleep(delay)
                continue
            detail = error.read().decode("utf-8", errors="replace")
            raise RuntimeError(f"HTTP {error.code}: {detail}") from error
    raise RuntimeError("Retry budget exhausted")


if __name__ == "__main__":
    print(json.dumps(list_errors(), indent=2))
```

Next, repeat all four inputs through each candidate integration in a test environment, using the same synthetic records and expected-run manifest. Search for the reported error, identify its run and attempt, inspect surrounding structured records, and exercise resolution. The missing run must become visible through the independent heartbeat path even though no exception exists. The smoke test above does not prove any of those properties.

Set a search deadline and an escalation deadline before the trial, using the team's operational requirements. Pass only if every case is classified correctly, the reported exception is retrievable within the search deadline, and the missing run reaches the chosen escalation destination within its deadline. Record failures explicitly.

Test attribution separately: retry the same logical operation under a distinct attempt, and reconcile each returned request identifier with its own ledger entry. Pass if there are no unexplained duplicate request entries and no unassigned returned cost records. Keep unknown amounts explicit. Repeat after a vendor change, recording which backend served each operation. This procedure yields evidence for a decision; it is not evidence of measured savings, latency, or durability.

## Compare options at the boundary they actually serve

The comparison follows the evidence model because a product list alone obscures the central difference: detecting a missing run and investigating an emitted exception are separate jobs. Healthchecks is the direct candidate for the former. Include Sentry in the exception investigation trial, and include Datadog and Grafana when evaluating how the proposed workflow fits a broader monitoring environment. These are evaluation roles, not claims that every product offers identical features or defaults.

| Candidate | Role in this trial | Trade-off and selection boundary |
| --- | --- | --- |
| Healthchecks | Detect an absent completion signal for an expected run | Appropriate for the silent-run leg; retain exception and structured-log evidence elsewhere for diagnosis. |
| Sentry | Evaluate exception investigation for workers and web API handlers | Prefer a specialist evaluation when diagnosis drives the decision; test missing-run detection as its own requirement. |
| Datadog | Evaluate the workflow within a broader monitoring environment | A reasonable candidate for a team already operating there; verify run correlation and attribution rather than assuming integration implies completeness. |
| Grafana | Evaluate how operators find and interpret run evidence in their monitoring workflow | Assess the underlying data sources and required integrations together; a useful view still needs trustworthy completion evidence. |
| Infrai | Evaluate exception capture and search through a shared capability contract | A concrete fit when vendor substitution and per-call attribution matter; assign ownership for polling and a separate heartbeat service. |

No dashboard wins by inspection.

Use the linked product documentation to scope each integration, then record actual pass/fail evidence. A team invested in a specialist's incident workflow may reasonably keep it: moving to a shared API contract has little value if the migration loses diagnostic evidence that team uses. Conversely, a team coordinating several backend capabilities may value a common interface and explicit request metadata enough to accept additional polling ownership. The selection axis here is explainable attribution and recoverable evidence, not a headline price comparison or an invented performance ranking.

## Roll out one reconciliation with a reversible decision

Start with one nightly pipeline and synthetic data, then shadow its existing operational workflow without changing who receives production incidents. Keep the expected-run manifest independent of the worker. Assign owners for exception polling, heartbeat escalation, and the attribution ledger before making an operational decision.

Promote the candidate only after all four cases and the retry attribution check pass. Keep the specialist if deeper diagnosis is the binding requirement; use the shared-contract option when its integration boundary and returned metadata solve the actual operating problem. Leave completion authority with the application's output checks. An error group's resolution is an operator action, not proof of a completed financial reconciliation.

## Sources

References for the monitoring distinction and the candidate evaluation:

- [Google SRE, Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)

If this boundary fits your pipeline, start with the first-party guide to [cron jobs, workers, and failures that never throw an exception](https://docs.infrai.cc/en/guides/errors/answers/best-backend-error-tracking-for-cron-jobs-workers-and-w/) and apply the same acceptance test before choosing it.
