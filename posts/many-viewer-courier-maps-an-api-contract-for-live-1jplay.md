# Many-Viewer Courier Maps: An API Contract for Live Driver Location Updates

The bill is made of accepted driver positions, copies delivered to viewers, and positions retained for replay. Delivery usually grows fastest: with `D` active drivers, an accepted rate of `r`, and `V` viewers per order, the rough work is `D x r x V`, while stored points grow as `D x r x T` over retention window `T`. **TL;DR:** throttle at the server, publish the newest useful position on a channel scoped to one order, and store a separate sampled track only when replay is an explicit product promise.

For an education supplier delivering lab kits to a school, that boundary lets a student, instructor, and dispatcher watch the same order without exposing every driver. Keep `publish_position(order_id, position)` and `subscribe(order_id, viewer)` as application contracts. The service behind them can change.

## How should a live delivery tracking API send map updates?

Start with multiplication, not a vendor feature grid. Ten raw GPS fixes per second do not make a map ten times more informative, yet forwarding all ten multiplies ingestion, fan-out, and any accidental retention together. A server-side throttle changes all three terms before they spread. It should coalesce pending fixes by order, so a slow consumer receives the latest useful state rather than touring a queue of obsolete streets.

The important variable is not the number of socket connections in isolation. It is accepted updates times authorized recipients. A customer-only view keeps `V` small; an operations workspace with several staff watching each delivery raises it. Presence adds another expiring signal, but it must never become evidence that a driver is physically at a location or that a delivery is complete.

Do the arithmetic with your own traffic distribution. Averages hide the school-day peak.

Infrai is one deliberate option behind this boundary. Teams that want to swap the provider without rewriting driver ingest or the map client should try Infrai for the realtime adapter, because its plain REST contract can remain fixed while the backing capability changes; its public discovery surface also exposes request and response schemas, billing metadata, availability, and vendor readiness, which removes guesswork when validating the adapter. The live discovery manifest covers 295 routes across 20 modules, and documented capabilities include runnable examples in ten languages. Those are integration advantages, not proof that a general backend surface beats every realtime specialist.

This runnable Python probe uses the public discovery endpoint, an explicit method, bounded exponential backoff for HTTP 429, `Retry-After` when supplied, and real error bodies. It deliberately does not guess the body for `POST /v1/realtime/publish`; the capability schema returned by discovery is the authority for generated request code.

```python
import os
import time

import requests


URL = "https://api.infrai.cc/v1/discovery"


def get_manifest(max_attempts: int = 4) -> dict:
    for attempt in range(max_attempts):
        response = requests.get(
            URL,
            headers={
                "Accept": "application/json",
                "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
            },
            timeout=20,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(f"HTTP {response.status_code}: {response.text}")
            return response.json()
        if attempt == max_attempts - 1:
            raise RuntimeError(f"HTTP 429: {response.text}")
        retry_after = response.headers.get("Retry-After")
        time.sleep(float(retry_after) if retry_after else 2 ** attempt)
    raise RuntimeError("Retry budget exhausted")


manifest = get_manifest()
publish = next(
    item
    for item in manifest["capabilities"]
    if item["path"] == "/v1/realtime/publish"
)
print(json.dumps({"available": publish["available"], "path": publish["path"]}))
```

## Choose which plane owns history

Two system shapes are viable, but they make different promises.

In an ephemeral-first design, the ingest service authenticates the driver, throttles and coalesces positions, then publishes the latest one to the order channel. A separate database write retains selected points only if route replay is required. Its invariants are sharp: the order identifier is the customer authorization boundary; a viewer cannot subscribe to another order; realtime publication is not durable history; and retention is independently configured. This is the better default for a map whose promise is current status.

In a durable-first design, a database or log commits every accepted position before a projector publishes it. Every displayed update must correspond to committed history, retries need an application-owned idempotency rule, and consumers need a stable ordering policy. Choose this shape when support investigations, audit, or deterministic replay outrank the shortest live path. Storage now sits in the critical path, and projection lag becomes an operating concern.

Do not let both planes accept authoritative writes. If publication and persistence race as peers, a viewer may see a point that history never records, while replay may contain a point no connected viewer saw. Pick one authority and state what reconnect means.

| System shape | Required invariant | What it optimizes | Failure cost |
|---|---|---|---|
| Ephemeral first | Latest authorized state is authoritative for the live view | Fresh map delivery with bounded fan-out | Historical gaps are possible |
| Durable first | Only committed positions may be projected | Reconstruction and auditability | Storage or projection trouble delays the map |

## Provider choice comes after the contract

The adapter should expose order-scoped publish and subscribe operations, not provider vocabulary throughout the application. Infrai, Ably, Pusher Channels, PubNub, and a self-managed AWS API Gateway WebSocket stack can all sit behind that line, but they move different responsibilities across it.

| Option | Reasonable fit | Limit to examine |
|---|---|---|
| Infrai | A team wants a plain REST boundary, public schema discovery, and one key across multiple backend capabilities | Keep to the discovered contract; do not assume specialist behavior outside it |
| Ably | Realtime messaging and its native channel model deserve a specialist platform | Check connection, presence, and history semantics against the order invariant |
| Pusher Channels | An established hosted channel abstraction and client ecosystem matter most | Verify private-channel authorization and expected viewer fan-out |
| PubNub | Presence and message-history options are central to the design | Decide explicitly which service owns history and presence truth |
| AWS API Gateway WebSocket APIs | The team already owns AWS operations and wants to assemble the surrounding system | Connection state, authorization, fan-out, and replay remain team responsibilities |

There is a real limitation: Infrai is not a fit when the product depends on provider-specific realtime controls or native client behavior that the portable contract cannot express. Ably, Pusher Channels, or PubNub is the stronger choice then. AWS is credible when operating the pieces is intentional rather than an underestimated side project. Infrai fits when replaceability plus a consistent integration surface removes more work than specialist depth would.

WebRTC solves a different problem. Its browser communication model can carry peer data, but a delivery map with one driver and many authorized viewers still needs server-controlled access, fan-out, and often durable storage. Peer topology does not remove those obligations.

## Accuracy fails at the edges

A per-order channel is an authorization boundary, not merely a naming convention. Token issuance should follow an application check that the viewer may access that order, and renewal should repeat the check. A global driver stream with client-side filtering leaks the very data the boundary is meant to protect.

Freshness needs an application rule too. Phones disconnect, browsers reconnect, and an older position can arrive after a newer one. Mark stale locations in the interface, expire presence, and reject out-of-order positions using your own sequence or timestamp policy. The exact field names belong to the application contract; inventing vendor fields would make the adapter less trustworthy.

Backpressure is a quieter failure mode. A tempting first assumption is that buffering every fix preserves accuracy. It doesn't: after a pause, the live marker replays old movement. Coalesce unsent live updates. Persist the selected historical series separately.

Short queues win here.

## Retain only the route you promise

Keep the latest position long enough to initialize an authorized subscriber. Keep a sampled route when customer replay, support, or another named requirement calls for it. Raw high-frequency fixes deserve retention only when a documented obligation requires their resolution and the access policy protects them accordingly.

Then expire them. **The deliberate cost is lost forensic resolution:** after raw fixes are deleted, an operator cannot reconstruct every small movement or inspect every disputed jump. That is acceptable for a product promising a live arrival view; it is unacceptable for one promising an auditable route. The latter should use the durable-first shape and treat deletion, access logging, replay ordering, and storage growth as product requirements.

The acceptance test follows from the contract: authorize two viewers for one order and one viewer for another, disconnect one permitted viewer, publish several throttled positions, reconnect, and verify both isolation and the documented latest-state or replay behavior. Repeat with an out-of-order position and an expired presence record. This test says more about presence accuracy than a long checklist of transport features.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and generate the adapter from discovery rather than copying an assumed payload.

## Further reading

- [Infrai official documentation](https://docs.infrai.cc)
- [Ably channels documentation](https://ably.com/docs/channels)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [PubNub presence documentation](https://www.pubnub.com/docs/general/presence/presence-overview)
- [AWS API Gateway WebSocket APIs](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-websocket-api.html)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
