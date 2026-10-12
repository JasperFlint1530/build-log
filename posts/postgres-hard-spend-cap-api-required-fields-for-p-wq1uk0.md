# Postgres Hard Spend Cap API: Required Fields for Property Access Reviews

Set the ceiling before issuing or enabling a credential, reject work atomically when the period's reserved spend reaches that ceiling, and read the stored policy back before an approver signs the access review. An alert threshold is a warning, not the control. In a property-management system, this rule keeps a leasing integration from continuing to send email, SMS, or OTP traffic after its approved budget is exhausted.

**Short answer:** the write contract needs a subject identifier, ceiling amount, currency, period definition, effective boundary, and alert threshold. The read-back response must expose the normalized values plus policy identity and revision. Store money in integer minor units, bind each reservation to an idempotency key, and make admission a single database transaction. This favors refused traffic over an overspend whenever the system cannot prove capacity remains.

## How should an API set a hard spend cap?

An access review should answer more than “who has a key?” It should show what the credential can spend, when that allowance resets, and what happens at the boundary. For a property portfolio, the subject might be `leasing-notices-west`, while the period might be a calendar month in `America/Los_Angeles`. That timezone is part of the policy because a bare `monthly` label leaves the reset instant ambiguous.

The control has four invariants. First, accepted plus reserved spend never exceeds the ceiling for a subject and period. Second, a retry with the same idempotency key cannot reserve twice. Third, changing a ceiling creates a new revision rather than rewriting the evidence an approver saw. Fourth, a missing or unreadable policy fails closed for billable traffic.

No silent fallback.

That last choice hurts availability. It is still the defensible default for this decision axis: the stated ceiling is hard, so uncertainty must become refused traffic. Emergency communications should use a separately approved policy and credential rather than a hidden bypass. OTP delivery gaps are already difficult to diagnose; mixing an undocumented exemption into the same counter makes both the spend record and the delivery record less credible.

Treat secrets separately from the budget document. The OWASP Secrets Management Cheat Sheet recommends lifecycle controls including creation, rotation, revocation, expiration, and auditing. A review record can reference a credential identifier, but it should not contain the secret value.

## Decision and failure boundaries

The selected design is a relational policy row plus a per-period usage row, updated in one Postgres transaction. The policy API is an internal contract, not a vendor-specific route. Its required write fields are:

- `subject_id`: the stable billing and access principal.
- `ceiling_minor`: a positive integer in the currency's minor unit.
- `currency`: the currency code used by both policy and metering.
- `period_kind`: an explicit supported value such as `calendar_month`.
- `period_timezone`: the timezone used to calculate calendar boundaries.
- `effective_from`: the instant after which the revision governs new reservations.
- `alert_at_minor`: an integer below or equal to the ceiling.

The service should reject unknown fields, unsupported periods, mixed currencies, nonpositive ceilings, alert thresholds above the ceiling, and effective times that would retroactively alter settled usage. Validation is only the front door. The database transaction remains authoritative when two workers race.

The trade-off is concrete: row locking adds contention, the database sits on the admission path, and a database outage refuses otherwise valid messages. This design is not suitable when uninterrupted traffic matters more than a strict ceiling. In that case, asynchronous metering with a deliberately documented overspend tolerance is the honest choice. For the access review here, however, allowing traffic without a confirmed reservation would contradict the control being signed.

| Option | Ceiling guarantee | Behavior during dependency failure | Review evidence | Operational cost |
|---|---|---|---|---|
| Transactional policy and usage rows | Strong when all reservations use the transaction | Refuse when the database cannot confirm capacity | Policy revision and reservation ledger share one authority | More write contention; clear failure semantics |
| Alert from an asynchronous meter | None at admission time | Traffic may continue while events lag | Good for notification, weak for enforcement | Low request-path overhead |
| In-process counter | Per process at best | Restarts and replicas lose a common decision point | Difficult to reconcile | Simple only for a single disposable worker |

The alert should fire on the transition from below `alert_at_minor` to at or above it, with deduplication by subject, period, and policy revision. Delivery failure must not roll back the spend reservation. Otherwise a temporary notification outage changes admission behavior, and retries can create the very overspend the alert was meant to flag.

Alerts can arrive late. Enforcement cannot.

## Critical path in Python

The critical path is short because it has one job: reserve capacity or refuse. The surrounding API can authenticate the caller and validate the request, but the transaction owns the race.

```python
from dataclasses import dataclass
from decimal import Decimal, ROUND_HALF_UP


@dataclass(frozen=True)
class Reservation:
    subject_id: str
    period_start: str
    idempotency_key: str
    amount_minor: int


def to_minor_units(amount: Decimal, scale: int = 2) -> int:
    quantum = Decimal(10) ** -scale
    normalized = amount.quantize(quantum, rounding=ROUND_HALF_UP)
    return int(normalized * (10**scale))


def reserve(conn, request: Reservation) -> dict:
    with conn.transaction():
        existing = conn.execute(
            """
            SELECT amount_minor
            FROM spend_reservations
            WHERE subject_id = %s AND idempotency_key = %s
            """,
            (request.subject_id, request.idempotency_key),
        ).fetchone()
        if existing is not None:
            return {"accepted": True, "amount_minor": existing[0], "replayed": True}

        usage = conn.execute(
            """
            SELECT u.reserved_minor, p.ceiling_minor, p.alert_at_minor, p.revision
            FROM spend_usage AS u
            JOIN spend_policy AS p ON p.id = u.policy_id
            WHERE p.subject_id = %s AND u.period_start = %s
            FOR UPDATE OF u
            """,
            (request.subject_id, request.period_start),
        ).fetchone()
        if usage is None:
            return {"accepted": False, "reason": "policy_or_period_unavailable"}

        reserved, ceiling, alert_at, revision = usage
        if request.amount_minor <= 0 or reserved + request.amount_minor > ceiling:
            return {"accepted": False, "reason": "ceiling_exceeded"}

        conn.execute(
            """
            INSERT INTO spend_reservations
                (subject_id, period_start, idempotency_key, amount_minor, policy_revision)
            VALUES (%s, %s, %s, %s, %s)
            """,
            (
                request.subject_id,
                request.period_start,
                request.idempotency_key,
                request.amount_minor,
                revision,
            ),
        )
        new_total = reserved + request.amount_minor
        conn.execute(
            """
            UPDATE spend_usage SET reserved_minor = %s
            WHERE subject_id = %s AND period_start = %s
            """,
            (new_total, request.subject_id, request.period_start),
        )
        return {
            "accepted": True,
            "reserved_minor": new_total,
            "alert_due": reserved < alert_at <= new_total,
            "policy_revision": revision,
        }
```

One trap deserves attention: an idempotency key should be scoped to the subject, and a replay carrying a different amount should be rejected as a conflict rather than accepted with the old amount. The compact example returns the prior reservation; a production boundary must also compare the immutable request fingerprint. Small detail. Large audit consequence.

Reservations also need a settlement story. If the downstream message is never submitted, release or expire the reservation through a recorded transition. If actual cost arrives later, reconcile it without deleting the original decision. Monitoring should separate ceiling refusals, unavailable-policy refusals, duplicate replays, alert-delivery failures, and reconciliation drift; combining them into one error count hides whether the control protected spend or merely broke traffic.

Keep the evidence.

## What must the read-back prove?

Immediately after a policy write, read it through the same interface used by the review exporter. Compare every normalized control field, not just the ceiling. A useful response includes `policy_id`, `revision`, `subject_id`, `ceiling_minor`, `currency`, `period_kind`, `period_timezone`, `effective_from`, `alert_at_minor`, and status. It may also include creation metadata for audit, but it must never echo a secret.

The reviewer should receive the policy revision beside the credential identifier and scope. For example, the evidence can say that `leasing-notices-west` is governed by revision 7 for the calendar month defined in Los Angeles time. The exporter should fail the review job if the read-back differs from the requested normalized policy. Silence is not confirmation.

Test the boundary with 9,999 units already reserved against a 10,000-unit ceiling: a request for 1 is accepted, a subsequent request for 1 is refused, and replaying the accepted request changes nothing. Then race two reservations against one remaining unit. Exactly one may commit. Also test the timezone boundary, policy revision during an active period, alert retry, missing policy, database timeout, and a duplicate key with altered payload.

## Rejected option and its valid use

The rejected design is asynchronous metering followed by an alert. It cannot enforce a hard ceiling because work can be admitted before the meter consumes its events, and an alert consumer can lag or fail independently. It is valid when the business accepts bounded reporting delay and wants visibility rather than admission control, such as forecasting portfolio-wide communication demand.

An in-process limiter is also useful for smoothing bursts before they hit a downstream service. It should remain a traffic-shaping layer. It cannot serve as the signed spend control once multiple workers, restarts, and delayed reconciliation enter the picture.

The durable decision is therefore plain: keep the policy versioned, reserve atomically, refuse on uncertainty, and make read-back equality a prerequisite for approval. This produces evidence an access reviewer can sign without pretending that an alert is a ceiling.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
