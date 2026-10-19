# Node.js Property Password Resets: Improve Custom Domain Email Deliverability

Use a dedicated custom-domain mail stream for property-management password resets, require aligned authentication before enabling it, and rotate DKIM keys with an overlap rather than a flag day. **Short answer:** Node.js should own the reset workflow and a narrow mail interface; DNS verification, signing, retries, and delivery events belong behind that interface. This keeps integration effort bounded without pretending that an accepted API request means a tenant received mail.

The decision hinges on four invariants. Reset responses must not reveal whether an account exists. Tokens must be random, single-use, stored securely, and expire after a short configured interval. The visible `From` domain must align with an authenticated domain. Finally, a signing-key change must leave old signatures verifiable while queued mail drains. Those boundaries matter more than the choice of transport provider.

## How can Node.js improve custom domain email deliverability?

Domain verification proves control of a DNS name; it does not establish a good sending history, correct authentication alignment, or useful message behavior. SPF authorizes sending infrastructure. DKIM attaches a domain signature to the message. DMARC evaluates alignment between the visible `From` domain and SPF or DKIM authentication, then publishes policy and reporting instructions. A green verification badge can therefore coexist with broken alignment.

No shortcut.

Check the received message, not the control panel. The `Authentication-Results` header should show which identities actually passed. A common edge case is signing with an organizational domain while placing a different tenant-facing subdomain in `From`. Relaxed DMARC alignment may accept related subdomains; strict alignment will not. Pick the intended relationship deliberately and encode it in DNS policy rather than discovering it during an urgent lockout.

Password-reset mail also has a different risk profile from a welcome campaign. It is user-triggered, time-sensitive, and security-sensitive. Mixing it into a promotional stream couples its reputation and queue latency to bulk traffic. Keep complaint handling, suppression, and sending cadence separate. Do not bypass a legitimate suppression blindly: repeated delivery attempts to an invalid mailbox damage reputation and still do not recover the resident's account.

Short expiry creates pressure, but it should not create ambiguity. The application should define the expiry once and use the same value for token validation and message copy. A delayed message whose token has expired is a failed reset, even if every SMTP hop behaved correctly.

## Record the invariants and failure boundaries

The critical boundary is between creating reset state and attempting notification. Return the same outward response for existing and nonexistent accounts, as OWASP recommends, and keep response timing consistent enough to avoid an obvious enumeration channel. For an existing account, create only a cryptographically random, single-use token; store a hash rather than the raw token; associate it with the user and an expiry; then enqueue a message containing the raw token in a TLS-protected URL. Rate-limit requests per account and per network source.

Once queued, delivery becomes an asynchronous state machine. `queued`, `accepted`, `delivered`, `deferred`, `bounced`, and `complained` are materially different states. The exact event vocabulary varies by implementation, so normalize it at the adapter. Never mark a reset message delivered from the synchronous send response alone.

The failure boundaries should be explicit:

- A database failure means no token and no message.
- A queue publication failure must not leave an untracked usable token; use a transactional outbox or an equivalent atomic handoff.
- A transient transport failure may be retried with bounded backoff and an idempotency key.
- A permanent bounce ends delivery attempts and opens an account-recovery path that does not disclose account existence.
- A stale delivery event may update telemetry, but it must never extend or revive a token.

This is the part teams underestimate. Email is not the source of truth for authentication state. The reset-token record is.

## Compare the integration shapes

Integration effort includes DNS ownership, event ingestion, retry semantics, testability, and the operational burden after launch. The options below are architectural shapes, not product rankings.

| Shape | Application integration | Authentication control | Main failure boundary | Appropriate use |
|---|---|---|---|---|
| SMTP relay behind a local adapter | Small send surface, but delivery events need separate work | Custom `From`, SPF, DKIM, and DMARC still require DNS coordination | SMTP acceptance can be mistaken for delivery | Existing systems with mature SMTP telemetry |
| Transactional mail API behind an adapter | One request path plus signed event ingestion | Often automates parts of verification, but DNS remains authoritative | Provider-specific statuses can leak into domain logic | Teams that want managed queues and event callbacks |
| Self-operated mail transfer agent | Largest setup and on-call surface | Full control over selectors, signing, queues, and policy | Reputation, queue health, and abuse response become internal duties | Organizations with specialist mail operations or regulatory constraints |

The narrow adapter is the common denominator. It accepts a stable message intent and returns an opaque submission identifier. It does not expose a provider's template IDs throughout the codebase, and it does not let a controller invent retry behavior.

This design has limitations. An adapter and outbox add a worker, another persisted state transition, and reconciliation work; for a low-risk internal application, direct SMTP may be the better trade-off. Self-operation is also a poor fit when nobody owns abuse response, DNS changes, and queue monitoring. In that case, choose a managed transport behind the same port, accepting less control over its event vocabulary in exchange for a smaller operational surface.

## Put the critical path behind one interface

The application may be Node.js, but the contract is language-neutral; this Python reference makes the state transitions compact without tying the design to a commercial SDK. Production code should place the outbox insert in the same database transaction as the token-hash insert.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
from hashlib import sha256
from secrets import token_urlsafe
from typing import Protocol


RESET_TTL = timedelta(minutes=15)


@dataclass(frozen=True)
class ResetMail:
    recipient: str
    reset_url: str
    expires_at: datetime
    idempotency_key: str


class MailPort(Protocol):
    def enqueue(self, message: ResetMail) -> str: ...


def begin_reset(account, mail: MailPort, token_store, public_base_url: str) -> None:
    # The caller returns the same response even when account is None.
    if account is None:
        return

    raw_token = token_urlsafe(32)
    token_hash = sha256(raw_token.encode("utf-8")).hexdigest()
    expires_at = datetime.now(timezone.utc) + RESET_TTL
    reset_id = token_store.insert_single_use(
        account_id=account.id,
        token_hash=token_hash,
        expires_at=expires_at,
    )

    message = ResetMail(
        recipient=account.email,
        reset_url=f"{public_base_url}/reset-password?token={raw_token}",
        expires_at=expires_at,
        idempotency_key=f"password-reset:{reset_id}",
    )
    mail.enqueue(message)
```

Do not log `reset_url`, the raw token, or a complete recipient address. Log the reset record ID, opaque submission ID, selected sending domain, selector label, attempt count, normalized event, and timestamps. That is enough to answer the operational question: was a valid reset created, submitted once, and observed before expiry?

Test the adapter with fixtures for timeout, duplicate callback, out-of-order event, hard bounce, and a callback whose signature is invalid. Test DNS separately by resolving the actual selector and DMARC records from more than one resolver. Deployment should fail closed for a new sending domain until ownership and authentication checks pass.

For DKIM rotation, publish a new selector first. Confirm it resolves, begin signing new messages with it, and retain the old public key while messages bearing the old signature may still be in transit or under investigation. Remove the old record only after the chosen overlap window and queue evidence support removal. Rotation is a DNS and signer rollout, not an in-place overwrite of one selector.

## Rejected option: send inline from the reset request

Sending inline looks like the lowest-effort Node.js integration: generate a token, call a mail endpoint, and return. It was rejected because network latency then affects the reset response, retries can create duplicate submissions, and a transport timeout leaves the application unsure whether the remote system accepted the message. That uncertainty is especially awkward with a short-lived credential.

Inline sending still has a valid use case in a small internal tool where mail loss is tolerable, account enumeration is not exposed to untrusted users, and an operator can recover failures manually. A public resident portal does not fit that envelope.

The durable decision is therefore modest: keep provider details behind one port, atomically hand reset intent to an outbox, and treat delivery telemetry as evidence rather than authentication state. Verify alignment from real messages, rotate selectors with overlap, and keep transactional reputation isolated from bulk mail. Those controls reduce integration surprises without making unverifiable promises about inbox placement.

## References

- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [FTC CAN-SPAM Act compliance guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)
- [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://www.rfc-editor.org/rfc/rfc6376)
- [RFC 7208: Sender Policy Framework (SPF)](https://www.rfc-editor.org/rfc/rfc7208)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc7489)
