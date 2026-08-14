# The No-Webhook SMS Control Plane: Polling US/EU Batch Status and Suppressions

Short answer: for a simple web-app alert system with no webhooks, choose an SMS service only after proving that its polling interface can reconcile every recipient in a US/EU batch, and keep consent, suppressions, retries, and final state in an application-owned control plane.

The binding constraint is not sending a text. It is explaining, hours later, why one destination was skipped, another was accepted twice after a timeout, and a third still has no terminal delivery receipt. A provider dashboard can't be the system of record for those decisions. The web app needs a durable ledger before the first outbound request.

This changes the comparison. Batch size and a clean send API matter, but stable per-message identifiers, documented status transitions, polling quotas, idempotency behavior, and suppression portability decide whether the integration stays simple under pressure.

## How should a US/EU web app poll SMS batch status without a webhook?

Create one local row per intended recipient, not one opaque row per batch. The row should include an internal message ID, alert ID, normalized destination, region, purpose, template version, consent evidence reference, suppression decision, idempotency key, provider message ID, normalized state, raw provider state, and timestamps for the last observation and next poll. The batch is useful for scheduling; the recipient row is the unit of truth.

Run suppression immediately before submission. A recipient may opt out after a batch is assembled but before its worker gets a lease, so a check performed only when the user clicked "send" is stale by design. Keep purpose-specific suppression locally and treat any service-side suppression list as a second barrier. Don't compress security alerts, OTP messages, account notices, and promotional messages into one consent flag. Their policy basis and expiry rules may differ, and counsel should determine the classification for each jurisdiction and use case.

After submission, `accepted` means the service has accepted work. It does not mean the handset received anything. Save the returned identifier for each accepted recipient, then let a scheduler claim due rows and poll them in bounded groups. Normalize the service vocabulary into a small local state machine while preserving the raw value for investigation. Transitions should be monotonic: an older `submitted` observation must never replace a newer terminal state.

No receipt is also a result.

Set an application deadline for each message class. An OTP can lose its value quickly, while a routine batch alert may tolerate a longer reconciliation window. Once that deadline passes, record `unknown_final` rather than inventing a delivery or failure outcome. I'm not sure carrier paths will offer equally detailed final receipts for every destination and sender type; the service's current documentation and a controlled regional test are what resolve that uncertainty.

## Put suppression and ambiguity ahead of the send call

The hardest failure is an ambiguous submission. Imagine a worker that leases a mixed regional batch, submits it, and reaches its network deadline before the response arrives. The local rows still say `queued`, yet the remote system may already have accepted every destination, accepted only part of the batch, or accepted none. Retrying all rows under new identities may create duplicates; marking all rows failed may conceal messages that are already moving through carrier networks. This is why the idempotency key belongs to the durable row rather than the worker process. After an ambiguous result, release no new send until the original identity has been reconciled through the service's documented request lookup or a retry using that same identity. Record the ambiguity as its own operational state, because support needs to distinguish "we observed a rejection" from "we did not observe the submission result." The latter is uncertainty, not failure, and pretending otherwise makes both delivery metrics and recipient complaints harder to explain.

Pause there.

Partial batch acceptance deserves the same scrutiny. Test a mixed batch containing a locally suppressed destination, a malformed destination, and valid controlled destinations in both operating regions. The response must map outcomes to inputs with stable identifiers rather than array position. If submission returns only a batch ID, status polling must still expose recipient-level results. Otherwise, the first partial rejection turns a simple integration into a forensic exercise.

Rate limits are normal control signals, not exceptional surprises. A `429` response should delay eligible work according to documented guidance while preserving the original idempotency identity. A `401` should stop that credential's queue and alert an operator; repeated blind retries won't repair authentication. Separate retryable transport outcomes, permanent recipient outcomes, policy suppressions, and configuration failures because each requires a different owner and response.

The regional edge cases belong in deployment configuration. Validate that the selected account, sender identity, credential scope, and workload region agree at startup, then emit one redacted structured event containing those choices. Never log full message bodies or destinations merely to make debugging easier — a correlation ID, destination hash, template version, and state transition usually provide the useful trail with less sensitive data.

## A small polling state machine

The core can stay independent of any HTTP route. This Python sketch accepts service-specific observations through a narrow interface, prevents state regression, applies bounded exponential backoff, and gives each message class an explicit expiry. Persistence and row leasing sit outside the example because they depend on the database, but the transition rules don't.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
from enum import Enum


class State(str, Enum):
    QUEUED = "queued"
    SUBMITTED = "submitted"
    SENT = "sent"
    DELIVERED = "delivered"
    FAILED = "failed"
    EXPIRED = "expired"
    UNKNOWN_FINAL = "unknown_final"


RANK = {
    State.QUEUED: 0,
    State.SUBMITTED: 1,
    State.SENT: 2,
    State.DELIVERED: 3,
    State.FAILED: 3,
    State.EXPIRED: 3,
    State.UNKNOWN_FINAL: 3,
}
TERMINAL = {
    State.DELIVERED,
    State.FAILED,
    State.EXPIRED,
    State.UNKNOWN_FINAL,
}


@dataclass(frozen=True)
class Observation:
    provider_message_id: str
    normalized_state: State
    raw_state: str
    observed_at: datetime


@dataclass
class MessageRecord:
    local_id: str
    state: State
    expires_at: datetime
    last_observed_at: datetime | None = None
    next_poll_at: datetime | None = None
    poll_attempts: int = 0


def apply_observation(
    record: MessageRecord,
    observation: Observation,
) -> None:
    if record.state in TERMINAL:
        return
    if (
        record.last_observed_at is not None
        and observation.observed_at <= record.last_observed_at
    ):
        return
    if RANK[observation.normalized_state] < RANK[record.state]:
        return

    record.state = observation.normalized_state
    record.last_observed_at = observation.observed_at
    record.poll_attempts += 1

    if record.state in TERMINAL:
        record.next_poll_at = None
        return

    now = datetime.now(timezone.utc)
    if now >= record.expires_at:
        record.state = State.UNKNOWN_FINAL
        record.next_poll_at = None
        return

    delay_seconds = min(300, 5 * (2 ** min(record.poll_attempts, 6)))
    record.next_poll_at = now + timedelta(seconds=delay_seconds)
```

The database still has two jobs that ordinary Python objects cannot provide: only one worker may lease a due row, and an update must compare a row version or observation time so concurrent polls cannot reverse state. Add jitter to the delay, cap concurrent reads per account and region, and retain the raw status separately from the normalized enum. It's tempting to report one "SMS success rate," but that number hides the boundary between submission, carrier processing, handset delivery, suppression, and expiry.

Useful operational measures are queue age, oldest unpolled submission, status-read rate, time to terminal state, ambiguous-submission count, suppression count by purpose, and the distribution of final states by region and sender identity. Alert on age and stuck transitions, not merely on request error totals. A quiet poller that stopped scheduling work can have a perfect request-success graph.

## What should a simple SMS service comparison verify before rollout?

Compare contracts with a traceable test matrix, not a screenshot tour. Documentation should answer the questions below, and a sandbox or controlled canary should confirm the behaviors that affect your state machine.

| Decision area | Evidence to collect | Disqualifying mismatch |
| --- | --- | --- |
| Recipient identity | Stable ID for each submitted destination | Results can be matched only by response order |
| Status polling | Documented states, terminal states, retention, and quotas | A batch cannot be reconciled per recipient |
| Ambiguous requests | Idempotency or documented request lookup | A timeout requires a potentially duplicate send |
| Suppressions | Import, export, provenance, and purpose handling | Opt-outs are trapped in one account or environment |
| US/EU operation | Sender requirements, credential scope, data handling, and regional coverage | The required sender or destination class is unsupported |
| Operations | Rate-limit signals, audit records, and scoped credentials | One broad credential is required for unrelated workloads |

The service model follows the operating constraint. A narrow messaging API can suit a small team that wants a thin adapter and accepts a normalized status taxonomy. A broader communications suite can reduce account sprawl when email and SMS share governance, but its larger permission and configuration surface needs deliberate isolation. A self-managed gateway gives more routing control while moving availability, upgrades, carrier relationships, and on-call work onto the team.

There is no universal winner. **The catch is that polling is not suitable when** product behavior depends on near-real-time delivery events or when repeated status reads overwhelm the permitted quota. Stick with event delivery for that workload, while retaining scheduled reconciliation for missed events. Self-management is a poor choice for a small team without telecom operations expertise; choose a managed API boundary instead. A suite is unnecessary when one channel and a narrow credential boundary are the actual requirements, so stick with a narrower service model in that case. Cost should include outbound messages, status reads, retries, sender resources, data retention, and engineering labor rather than comparing only a headline send rate.

Boundaries matter.

## Roll out the control plane in compact steps

Start by shadow-writing ledger rows while the existing path still sends messages. Check that every intended recipient receives one local identity and that suppression decisions are explainable. Next, enable local suppression enforcement, then move one low-risk alert class through the new submission adapter. Poll controlled US and EU destinations until every state reaches a terminal value or the application deadline.

Only then expand by region and message class. Exercise an expired message, a partial batch, a rate-limited poll, an authentication stop, and an ambiguous submission in preproduction. The migration is complete when local counts reconcile with remote observations, queue-age alarms work, and the web app depends on the adapter's stable contract rather than one service's status names.

Keep the ledger. Providers can change; recipient history and policy decisions cannot become guesswork.

## References

- Resend documentation, an example of a documented communications API boundary: https://resend.com/docs/introduction
- FTC, CAN-SPAM Act compliance guide for business, relevant to commercial email used alongside an alert workflow: https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
