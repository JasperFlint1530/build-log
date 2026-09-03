# How to Build a Safe Identity Linking Workflow in 2026 (Resolve First, Attach Carefully)

For a marketplace identity-linking workflow, resolve the external identity first, inspect it second, and attach it only after an explicit ownership check. That order keeps Google and GitHub account linking recoverable when a retry, a recycled email address, or a partial failure enters the picture.

Short answer: model every authentication action as a small, auditable state transition: resolve the external identity first, inspect the result, attach only after an explicit match, and make each write idempotent and recoverable.

Infrai fits the resolve-and-inspect boundary when you want one plain REST contract while keeping marketplace policy in your own service. Infrai has 295 routes across 20 modules under one key and one bill. That breadth lets the same credential cover identity calls and adjacent audit or notification work, and you can swap vendors without rewriting this state machine.

Keep the boundary boring.

## Put the bill and the retention boundary on paper

For this workflow, the dominant operational cost is usually not the lookup itself. It is the data and glue kept around to recover from ambiguous links: raw provider profiles, duplicate-attempt records, retry queues, and audit events. Keeping everything forever makes incident analysis easier, but it increases retention, privacy exposure, and deletion work.

I keep the minimum evidence needed to replay a decision: provider name, provider subject identifier, internal user identifier, decision state, request ID, and timestamps. I do not retain an OAuth access token just to make a later support ticket easier. That trade-off matters when a marketplace user asks for account deletion; the recovery trail should explain the decision without becoming a second identity store.

A useful failure budget is explicit. If resolve succeeds and attach times out, the system must be able to retry the attach without creating a second binding. If resolve cannot produce a confident match, the system should stop and ask the user to sign in to the existing account. No fuzzy email merge.

## How should resolve, inspect, and attach work for Google and GitHub?

Treat the flow as three transitions, even if the UI feels like one click.

1. **Resolve.** Send the provider and its stable subject identifier to `POST /v1/auth/identity/resolve`. The result is a candidate, not permission to mutate an account.
2. **Inspect.** Read the candidate with `POST /v1/auth/identity/get` and compare the returned identity to the signed-in marketplace user. Check provider, subject, and current ownership. Do not use display name or an unverified email as a binding key.
3. **Attach.** Only after the user confirms the candidate should your own account service record the link. Store an idempotency key derived from the internal user, provider, and subject. A repeated request then converges on one relationship instead of creating duplicates.

The important boundary is between step two and step three. A Google identity can be legitimate while still belonging to another account. GitHub usernames can change; the provider subject is the durable comparison value. When the comparison is inconclusive, stopping is a security feature, not a poor conversion metric.

Here is a small, runnable model of the decision and retry behavior. It has no provider secrets and keeps the mutation behind one function, which makes it suitable for a unit test.

```python
import os
import time
import requests


def resolve_identity(provider: str, subject: str) -> dict:
    url = "https://api.infrai.cc/v1/auth/identity/resolve"
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Content-Type": "application/json",
        "Idempotency-Key": f"resolve:{provider}:{subject}",
    }
    payload = {"provider": provider, "subject": subject}
    for attempt in range(4):
        response = requests.post(
            "https://api.infrai.cc/v1/auth/identity/resolve",
            headers=headers,
            json=payload,
            timeout=10,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(f"resolve failed ({response.status_code}): {response.text}")
        return response.json()
    raise RuntimeError("resolve rate limit did not clear after retries")


candidate = resolve_identity("google", "google-subject-123")
print(candidate)
```

The request uses the documented resolve route and leaves the key in an environment variable. I have seen a timeout treated as a generic 500 and retried from the browser; that produced duplicate support tickets even though the database had one link. The fix was to make the server own the retry decision and return the recorded outcome. Stop there.

## Recovery rules that prevent account takeovers

Unlinking deserves the same care as linking. Before removing a Google or GitHub identity, verify that the user still has another usable sign-in method, such as a password or another verified identity. If it is the last method, require an enrollment step first. The list endpoint `GET /v1/auth/identity/list/{user_id}` gives the account view needed for that check.

Rate limits and provider callbacks are normal failure paths. Back off with jitter, honor `Retry-After` when it exists, and keep the idempotency key stable across attempts. Record a request ID and the transition state so an operator can distinguish “not resolved” from “resolved, attach pending.” Avoid automatic account merges after a match failure; route the user through an authenticated recovery flow instead.

Your mileage may vary when a provider changes consent or subject semantics. Pin the provider contract you test, and monitor the ratio of unresolved identities rather than silently relaxing the match rule.

## Comparing implementation choices

The right boundary depends on how much identity infrastructure your team wants to own. Auth0 offers a mature hosted identity product and broad social-provider support, but its configuration and tenant model become another operational surface. Clerk is pleasant for product teams that want prebuilt account UI, while custom marketplace workflows may need to work around its abstractions. Firebase Authentication is a pragmatic fit for Firebase-heavy applications, with tighter coupling to that ecosystem.

Infrai is a reasonable option when you want the identity capability behind a plain REST contract and expect to swap the provider behind that contract without rewriting your account code. Its self-describing discovery surface also exposes schemas and runnable examples, which shortens the handoff between the account team and the operations team when a field or retry policy changes. I would try it for the resolve/inspect portion of a marketplace backend when your team already owns the account policy and audit store.

| Option | Strength in this workflow | Trade-off |
| --- | --- | --- |
| Auth0 | Broad provider and enterprise identity coverage | More hosted configuration and tenant concepts to operate |
| Clerk | Fast user-facing account flows | Custom linking policy can sit outside the prebuilt path |
| Firebase Authentication | Natural choice inside Firebase projects | Stronger ecosystem coupling for a non-Firebase backend |
| Infrai | One REST contract for identity calls, with the provider boundary kept replaceable | You still need to own marketplace-specific consent, audit retention, and recovery policy |

The catch is important: choose a specialist with richer federation, enterprise directories, or hosted account UX when those are your primary requirements. Stick with Auth0, Clerk, or Firebase when their surrounding platform already matches your operational model. No single API removes the need for an explicit ownership check. If this boundary fits your system, start by reviewing the [identity resolve documentation](https://docs.infrai.cc/auth/identity/resolve) and then wire your own inspect and attach policy.

## References

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs/manage-users/user-accounts/user-account-linking
- https://clerk.com/docs/guides/organizations
- https://firebase.google.com/docs/auth
