# Node.js Consent Snapshots vs Live Checks — Choose Runtime Enforcement for Health Exports

A health data export can outlive the permission that started it. A patient may withdraw consent after requesting an archive but before a worker gathers records, renders files, or sends the download notification. **Choose a live consent check at every consequential boundary; use a snapshot only as evidence of what authorized the request at that moment.** The boundary is important: live checking is appropriate when consent is the lawful or policy basis for the export, while statutory access rights and other legal bases need their own rules rather than being forced into a consent model.

TL;DR: a consent record explains who agreed to what, under which notice, and when. It is audit evidence, not a reusable bearer token. For a healthtech app adding phone one-time-code login, successful OTP verification proves control of a phone number at one point in time. It does not prove that the patient still permits a particular export, nor that the requested dataset and recipient remain within scope.

## What does a consent record actually prove?

A useful record ties a subject to a specific purpose, scope, notice version, collection channel, and decision timestamp. It also needs lifecycle facts: withdrawal time, expiration if the policy defines one, and the authority that changed the state. The record should be immutable as evidence; a separate current-state projection can answer authorization queries quickly.

That separation prevents a subtle audit failure. Overwriting `granted` with `withdrawn` preserves today's answer but destroys yesterday's explanation. Keeping only an append-only event log has the opposite operational problem: every worker must interpret history correctly. Store the events, then derive a versioned decision view from them.

Consent is not authentication. OWASP recommends generic authentication error responses because account-specific responses can enable user enumeration. The same caution applies to an OTP entry point: do not reveal whether a phone number belongs to a patient, and apply throttling against automated attempts. After authentication, authorization still has to evaluate the export action and its current consent scope.

The distinction matters in healthcare because the data is sensitive and exports often become asynchronous. The request handler, queue consumer, object-storage writer, and notification sender do not share one instant in time. Treating the login session or the original checkbox as permanent authorization lets a delayed job cross a policy change unnoticed.

## Why must the check happen live?

Consent can change while work is queued. Under GDPR Article 7(3), withdrawing consent must be as easy as giving it, and withdrawal does not retroactively invalidate processing that occurred before withdrawal. That temporal rule points to two separate questions: "Was the request valid when accepted?" and "May this next processing step happen now?"

They need separate answers.

Consider one concrete race. At 10:00, a patient requests an export covering lab results and visit summaries. The web process records consent decision `c-1842` and queues the work. At 10:03, the patient withdraws permission while the worker is still waiting. At 10:05, the worker starts. A snapshot-only design sees a valid 10:00 decision and proceeds; a live evaluation sees the 10:03 withdrawal and cancels before reading or releasing more data. The exact times are illustrative, but the ordering is the test case that matters. Repeat it with withdrawal during rendering and again immediately before download, because moving the race by one boundary often exposes an unchecked path.

A snapshot is valuable for the first question. It records the decision ID and policy version used when the export was requested. A live policy read answers the second. Before data leaves its protected system, the worker should resolve the current subject, purpose, requested categories, destination, and consent status. A cached answer is still a snapshot; its acceptable age must follow the risk of the action, and final release deserves a fresh authoritative read.

Retries make this more than a theoretical distinction. A queue may redeliver a job after a timeout, or an operator may replay a failed stage. Idempotency prevents duplicate side effects, but it cannot make yesterday's permission current. Each retry must reauthorize before it reads additional health data or publishes an artifact.

## Separate phone proof from export authority

The OTP flow should produce an authenticated principal, not an export grant. Bind the verified phone challenge to a short-lived authentication transaction, limit attempts, consume successful challenges once, and avoid logging the code. Then let a policy service make the export decision from server-side records.

For bot and abuse resistance, apply controls at several dimensions rather than trusting one counter: phone number, account, network risk, device/session signals, and export frequency. Those signals can slow or reject suspicious authentication and export requests. They must not silently manufacture consent. Keep the fraud decision and the consent decision explainable as different inputs, because a low-risk request can still lack permission and a high-risk request may require step-up verification rather than a rewritten consent history.

The worker contract can stay small and vendor-neutral:

```python
from dataclasses import dataclass
from datetime import datetime, timezone
from typing import Protocol


@dataclass(frozen=True)
class ExportIntent:
    subject_id: str
    purpose: str
    categories: frozenset[str]
    recipient: str
    consent_decision_id: str


class ConsentPolicy(Protocol):
    def authorize_now(self, intent: ExportIntent, at: datetime) -> bool: ...


def release_export(intent: ExportIntent, policy: ConsentPolicy) -> None:
    allowed = policy.authorize_now(intent, datetime.now(timezone.utc))
    if not allowed:
        mark_export_cancelled(intent.consent_decision_id)
        return

    artifact = build_encrypted_export(intent)
    publish_once(intent.consent_decision_id, artifact)
```

The example deliberately passes identifiers and scope instead of a boolean captured by the web request. `authorize_now` can detect withdrawal, scope reduction, expiration, or a subject mismatch. The cancellation path should expose a neutral status to the patient and a precise internal reason code to authorized operators; do not leak sensitive account state through a public endpoint.

## Snapshot comparison vs live policy evaluation

| Decision point | Consent snapshot | Live policy evaluation |
| --- | --- | --- |
| Request acceptance | Strong audit evidence | Confirms current eligibility |
| Delayed worker start | Can be stale | Detects withdrawal or scope change |
| Retry after partial failure | Repeats the old decision | Re-evaluates before the next side effect |
| Historical investigation | Preserves the original context | Explains current state only |
| Service dependency | No synchronous policy read | Requires an available, consistent authority |

The clear choice for releasing a health export is live evaluation. Keep the snapshot too, but do not let it authorize future work by itself. A fail-open policy turns a consent-service outage into unauthorized disclosure risk, so high-impact release steps should stop and retry when the authority is unavailable. Earlier, reversible preparation may continue only if it does not expand access or disclose data and the threat model permits it.

There is a real trade-off.

The main limitation of live evaluation is dependency coupling: if the policy authority is slow or unavailable, release waits even when the original grant was valid. Snapshot-only authorization can be reasonable for an instantaneous, fully completed action whose governing rule cannot change during execution, or for preserving evidence after processing has ended. It is the wrong choice for a queued health export with multiple disclosure boundaries. Teams choosing live checks accept more availability engineering because stopping is safer than releasing under an unverifiable grant.

This design has a cost: the authorization service becomes part of the release path. Reduce that cost with bounded timeouts, idempotent jobs, backoff, and explicit states such as `waiting_for_policy`, `cancelled`, and `released`. Measure decision latency, denied releases, stale queued jobs, retry counts, and the time from withdrawal event to enforcement. Do not put phone numbers, OTPs, or exported clinical fields in those metrics.

## Roll it out without trusting the old checkbox

Start by defining purposes and data categories narrowly enough to evaluate. Backfill legacy grants as `unknown` unless their notice version, subject, scope, and provenance can be demonstrated; inventing precision during migration weakens the audit trail. Run the new evaluator in observe-only mode first, comparing its decision with the legacy path while the legacy path remains authoritative.

Next, enforce live checks on newly requested exports, then on retries and final downloads. Test the awkward sequences: withdrawal between queueing and execution, scope reduction during rendering, duplicate delivery, policy timeout, OTP replay, and two concurrent release attempts. A synthetic patient account is enough for these tests; production health data is not.

Finally, make rollout reversible at the execution level without making consent reversible by operators. A deployment flag may send jobs back to a waiting state, but it must never convert a denial into approval. Retain decision IDs, policy versions, timestamps, and outcome reason codes according to the organization's documented retention rules.

The rule stays compact: authenticate the person, evaluate the action, and re-check immediately before disclosure. Keep the original snapshot for proof. Use the live record for permission.

## Sources

- OWASP, "Authentication Cheat Sheet": https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- European Union, GDPR Article 7, "Conditions for consent": https://eur-lex.europa.eu/eli/reg/2016/679/art_7/oj
- NIST SP 800-63B, "Digital Identity Guidelines: Authentication and Authenticator Management": https://pages.nist.gov/800-63-4/sp800-63b.html
