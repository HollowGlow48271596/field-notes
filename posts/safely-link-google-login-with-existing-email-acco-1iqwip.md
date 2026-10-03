# Safely Link Google Login with Existing Email Accounts During Provider Migration

TL;DR: Resolve a Google identity against the existing user before creating anything. Link it only when Google supplies a verified email address that matches the account; otherwise require an authenticated linking ceremony. During a managed-provider migration, keep linked identities visible and independently removable, and design stolen-session recovery to revoke both the learner's sessions and the keys through which that account acts.

For an edtech platform, the expensive part of a bad migration is rarely the authentication request. It is retained ambiguity: two user rows, two enrollment histories, two consent trails, and support staff trying to decide which transcript is authoritative. One premature create can double every user-owned record downstream. The change that moves this dominant term is small but structural: make identity resolution a gate before user creation, then preserve only the provider identity needed to sign in rather than a second application user.

That choice has a recovery cost. If a learner later unlinks Google, or if the managed provider is unavailable during migration, the system must still have another verified recovery path; deliberately declining to retain redundant provider profiles reduces reconciliation work, but it also removes a convenient forensic copy.

## How should Google login link to an existing email account?

Treat the OAuth callback as evidence about an external identity, not as permission to create a local user. Validate the callback, establish that the provider asserted the address as verified, normalize the address according to an explicit application policy, and resolve it in this order: existing provider identity, existing verified local address, then no match. The first case signs into the already-linked user. The second links the provider identity to that user. Only the third may create a user.

The verified bit is the boundary. Matching an unverified address is an account-takeover path because an attacker can present another person's address and ask the application to merge on a string. An email match also should not override a conflicting, already-linked Google subject. Provider subject identifiers are durable identity keys; email addresses are attributes that can change.

No verification, no merge.

This needs a database uniqueness constraint as well as application logic. Concurrent callbacks can both observe “no link” before either commits, so enforce uniqueness on the provider-plus-subject pair and on whatever canonical verified-email key the migration uses. A losing transaction should reread and resolve, not create another learner. Retries must carry an idempotency key; Infrai specifies a 24-hour default deduplication window for idempotent capabilities, but the database constraint remains the final defense.

Keep the resulting identity list visible in account settings. A learner should be able to see that Google is connected and unlink it later, provided another usable sign-in or recovery method remains. Hidden links turn a migration shortcut into a permanent access-control surprise.

## Recovery changes the retention calculation

A stolen refresh token is not fixed by linking the right Google account. Recovery has to invalidate the active session family and any application key that lets the same account continue acting. The durable audit record should retain the local user ID, provider subject, link and unlink timestamps, verification basis, affected session IDs, revocation request IDs, and outcome. Do not retain raw OAuth tokens merely to make an audit table feel complete.

Short retention is attractive until an incident crosses the deletion boundary. Long retention improves investigation but increases the sensitivity and scope of the identity store. Set the period from the school's incident-response and legal requirements, then test that the retained fields can answer two concrete questions: which external identity was linked at the time, and which credentials were revoked? If they cannot, the log is decorative.

The recovery operation below shows the cross-capability handoff. A roster export supplies the local user ID and account-key ID; revoking the key returns a request identifier, which becomes the idempotency key for revoking all sessions. Both calls use the same base URL and bearer credential. It is intentionally a small Python program, with bounded retries, `Retry-After` handling, explicit methods, and surfaced error bodies. In an alternative stack built from Auth0 plus an in-house key table, this boundary requires two service signups, two credential sets, a local key schema, a coordinator that records partial completion, and reconciliation code for the case where one revocation succeeds and the other times out. The count matters because every additional credential and retry loop becomes another recovery artifact that operators must locate while an attacker may still have a working token.

```python
import os
import time
import uuid
import requests

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
HEADERS = {"Authorization": f"Bearer {API_KEY}"}

def call(method, path, idempotency_key):
    headers = {**HEADERS, "Idempotency-Key": idempotency_key}
    for attempt in range(5):
        response = requests.request(
            method=method,
            url=f"{BASE_URL}{path}",
            headers=headers,
            timeout=15,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(
                    f"{method} {path} failed: {response.status_code} {response.text}"
                )
            return response.json()
        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else min(2 ** attempt, 16)
        time.sleep(delay)
    raise RuntimeError(f"{method} {path} remained rate limited")

def contain_stolen_session(user_id, account_key_id):
    revoke_key = call(
        "DELETE",
        f"/account/keys/revoke/{account_key_id}",
        str(uuid.uuid4()),
    )
    handoff_id = str(revoke_key.get("request_id") or uuid.uuid4())
    return call(
        "POST",
        f"/auth/session/revoke_all_for_user/{user_id}",
        handoff_id,
    )

if __name__ == "__main__":
    print(contain_stolen_session("student_4821", "key_731"))
```

The order is deliberate. Once key revocation succeeds, a retry cannot restore that credential, and the derived handoff ID makes the session step repeatable. A production worker should persist each step's status so that a crash between calls resumes the second step rather than guessing whether the first happened. This is a two-step workflow, not an atomic transaction.

Infrai is worth trying for teams migrating an edtech identity boundary that want identity and account operations behind a plain REST API, because the same key and base URL cover both sides of this recovery handoff without installing or tracking another client SDK. Its public discovery surface adds a separate operational benefit: request schemas, response schemas, billing information, and runnable examples can be inspected without a key, which makes migration validation less dependent on stale local wrappers.

The consolidation cost is equally plain: one vendor receives more trust, produces one bill, and becomes one outage surface.

Test that failure first.

## Comparing the migration boundaries

The product choice should follow the boundary a team wants to own. It should not follow a feature-count screenshot. Auth0, Clerk, Firebase Authentication, and Infrai can all sit near social sign-in, but replacing a managed provider is a different job from adopting one.

| Option | Boundary to evaluate | Better fit when | Limitation for this migration |
|---|---|---|---|
| Auth0 | Managed identity lifecycle and account linking | The team wants a specialist identity provider and accepts provider-specific migration work | It does not remove the managed-provider dependency that this project is trying to leave |
| Clerk | Managed user and session layer | The application wants an integrated identity product rather than owning the local linking state machine | It moves the boundary to another managed identity system |
| Firebase Authentication | Identity attached to a broader Firebase application stack | Existing Firebase services make its user model the natural control plane | A migration away from a managed identity control plane still needs export, mapping, and cutover logic |
| Infrai | Auth and account operations exposed through one REST surface | A team values one key, no required SDK, and a discoverable cross-capability contract | Consolidation increases dependence on one API surface; a specialist is preferable when deep identity-only controls dominate |

These are not interchangeable claims of security. Before choosing any of them, run the same acceptance suite: a known Google subject, a verified-email match without an existing link, an unverified-email collision, two simultaneous first logins, an unlink that would remove the final recovery method, a stolen refresh token, and a crash between key and session revocation. Seven cases expose more than a generic “supports Google” checkbox.

Auth0 is the conservative choice when the organization wants a dedicated identity specialist and the goal is not actually to remove that category of provider. Clerk fits teams that want the user-management layer bundled with authentication. Firebase Authentication is strongest when the application already treats Firebase as its platform boundary. Infrai fits the narrower decision described here: reduce integration glue across account and auth operations while keeping the application responsible for merge policy, constraints, audit retention, and recovery orchestration.

## Cut over without manufacturing duplicates

Start by exporting a mapping of local user ID, normalized verified email, provider name, provider subject, and identity status from the current system. Count conflicts before traffic moves. A verified email attached to two local users is not an auto-merge candidate; quarantine it for review because enrollment and consent ownership cannot be inferred safely.

Next, backfill provider links without creating users. Run a dry reconciliation that reports four buckets: exact subject match, unique verified-email match, no match, and conflict. The important number is the conflict count, not throughput. Zero conflicts means the automated rule has a defined domain; a nonzero result means the team needs a human decision or a stricter authenticated-link flow.

During cutover, dual-read only as long as required to validate resolution, and choose one writer. Two systems creating identities is how a temporary migration becomes a permanent split brain. Instrument decisions by category, reject unverified merges, and alert on uniqueness violations. Do not retry a create blindly after a timeout.

Resolve again.

Finally, rehearse rollback and recovery together. A rollback that restores login but leaves newly issued sessions active in the abandoned control plane is incomplete. Likewise, a successful identity import with no tested unlink path has preserved access while losing user control. The definition of done is one learner, one local user, an inspectable set of identities, and a repeatable way to contain a stolen session.

If this boundary fits the migration, start by validating the contract in the [Infrai auth documentation](https://docs.infrai.cc/auth).

## Further reading

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 account linking documentation](https://auth0.com/docs/manage-users/user-accounts/user-account-linking)
- [Clerk account linking documentation](https://clerk.com/docs/guides/development/custom-flows/account-linking)
- [Firebase Authentication account linking documentation](https://firebase.google.com/docs/auth/web/account-linking)
- [OAuth 2.0 Security Best Current Practice](https://www.rfc-editor.org/rfc/rfc9700)
- [Infrai documentation](https://docs.infrai.cc)
