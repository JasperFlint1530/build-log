# Identity Linking: Why Duplicate Accounts Happen in Password-Recovery Migrations

The constraint is auditability: for a media publisher leaving a managed provider, identity linking means preserving each reader's stable subject without creating duplicate accounts. Those duplicates happen when a new login credential creates a new subject instead of resolving to the existing one. The practical choice is to treat email addresses and external-provider identifiers as changeable evidence, never as the account itself.

**TL;DR:** Identity linking is the controlled association of multiple verified login identifiers with one local subject. Duplicate accounts happen when registration or recovery creates a new subject before checking whether trustworthy evidence already points to an existing one. Do not merge on a matching email alone. Make linking an authenticated, logged state transition, and make password recovery restore access to a subject rather than create one.

## What does identity linking mean, and why do duplicate accounts happen?

Suppose a publisher has one subscriber record, `reader_4821`, with newsletter preferences, paid archive access, and an audit trail. The reader originally used a social login, then later enters the same email address into a password-based flow. If the new flow understands only its own credential table, it creates `reader_9014`. One human now has two subjects, two consent histories, and perhaps two different authorization states.

The key distinction is easy to miss: an account is not an email address. An email address can be reassigned, mistyped, changed, or shared. A provider-specific identifier is scoped to that provider. A local subject identifier is the durable key that the publisher's authorization and content systems should use.

Recovery makes this distinction urgent. Email delivery can be delayed, filtered, or requested repeatedly, so possession of a recovery link should prove control of a challenge, not invite the application to infer a new identity. A request for `alex@example.com` should produce the same public response whether or not that address exists, while the server performs its lookup and delivery work privately. OWASP recommends consistent messages and timing for existing and nonexistent accounts, single-use expiring reset tokens, and no account change until a valid token is presented.

## Derive the model from the audit constraint

Keep three concepts separate: the subject, the login identifier, and the linking event. The subject owns subscriptions and permissions. An identifier records a credential namespace plus the issuer-scoped identifier or normalized login name. A linking event records who initiated the change, which proof was accepted, when it happened, and the result.

A useful invariant is `UNIQUE(namespace, issuer, external_subject)`. For local email login, the namespace might be `password`; for federated login, the issuer must be part of the key. Email can be stored as a contact point and lookup aid, but it is not a safe universal join key.

The write path needs a transaction. Lock the candidate subject and identifier rows, re-check uniqueness inside the transaction, then attach the identifier and append the audit event together. If two callbacks race, one wins and the other receives a conflict that can be resolved through an authenticated flow. No orphan subject should be created as a side effect.

```python
def link_identifier(actor_subject, candidate, proof, repository, audit):
    if proof.subject_id != actor_subject.id or not proof.is_fresh():
        raise PermissionError("fresh authentication is required")

    with repository.transaction():
        current = repository.find_identifier_for_update(candidate.key)
        if current and current.subject_id != actor_subject.id:
            audit.record(
                action="identity_link_rejected",
                actor_id=actor_subject.id,
                reason="identifier_already_linked",
            )
            raise ValueError("identifier is already linked")

        repository.attach_identifier(actor_subject.id, candidate)
        audit.record(
            action="identity_linked",
            actor_id=actor_subject.id,
            identifier_namespace=candidate.namespace,
        )
```

Notice what is absent: an automatic merge based on equal email strings. Linking should normally require a current authenticated session plus fresh proof for the identifier being added. If the user cannot authenticate to either side, the case belongs in a recovery or reviewed support process with stricter evidence and a recorded decision.

This design has a real trade-off. It rejects some convenient automatic joins and sends ambiguous cases to review, which adds operational work and delays access for a small set of readers. It is also a poor fit for a low-risk service with no durable entitlements or support capacity. For a publisher carrying paid access and consent history, however, that limitation is preferable to an irreversible false merge: staff can resolve a held case, but they may not be able to separate two people's mixed audit records with confidence.

## Put password recovery on the same subject graph

During migration, import the subject and its identifier relationships before enabling the new recovery entry point. A reset request looks up an existing subject through a verified, eligible recovery address, creates a random single-use token, stores only an appropriate protected representation, applies an expiry, and sends a link. After redemption, rotate the password credential for that same subject and invalidate affected sessions according to policy.

Do not create an account from a reset request. Ever.

The outward response can remain neutral: if the address is eligible, instructions will be sent. Internally, rate limits should cover the source, destination, and subject so an attacker cannot turn the endpoint into an email flood. Delivery status is operational telemetry, not proof that the intended person received the message. Log the request outcome without putting reset tokens or unnecessary personal data into logs.

For a media company, the audit record should connect the migration mapping, reset challenge issuance, successful redemption, credential rotation, and session action through opaque event identifiers. Keep the security log append-oriented and access-controlled. Retention follows the publisher's legal and operational policy; there is no honest universal number.

## Failure modes worth testing

The happy path proves very little. Test two concurrent sign-ins that try to attach the same external identifier. Test an address change followed by a reset to the old address. Test a delayed message after a newer reset request has invalidated the first token. Test Unicode and case handling according to the exact identifier policy rather than sprinkling lowercase calls through handlers.

Then test the uncomfortable states:

- one legacy subject has two login methods, and both must land on the same subscription;
- two legacy subjects share a contact address but have different entitlements, so migration flags them instead of merging them;
- a federated login returns the same display email but a different issuer or external subject;
- a user unlinks the last usable login method, which the service must reject unless another recovery path is established;
- a reset token is redeemed twice, from two processes, with only one transaction succeeding.

Observe counts of link conflicts, duplicate-subject detections, recovery requests, token redemptions, expirations, and delivery outcomes. Alert on changes in rates, but keep high-cardinality addresses and tokens out of metric labels. Correlation IDs let operators trace a flow without turning dashboards into a personal-data index.

## A compact migration rollout

Start by exporting a mapping from each legacy subject to its identifiers and downstream entitlements. Validate uniqueness constraints offline, then quarantine ambiguous rows for review. Dual-read during a bounded transition can help locate a subject in either store, but choose one authority for writes; competing writers make the audit story incoherent.

Next, migrate a small cohort and verify that every sign-in and recovery resolves to the expected stable subject. Compare aggregate link-conflict and recovery-completion signals, inspect audit continuity, and rehearse rollback before widening the cohort. Rollback should restore routing, not manufacture another identity mapping.

Finally, disable legacy writes, retain the mapping evidence for the approved retention period, and remove transitional lookup paths after reconciliation. **The success criterion is continuity of subject, authorization, and evidence**, not merely a high login-success rate. A login that succeeds into the wrong profile is a security failure wearing a green status code.

## Sources

References:

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- https://www.rfc-editor.org/rfc/rfc7519
- https://openid.net/specs/openid-connect-core-1_0.html
