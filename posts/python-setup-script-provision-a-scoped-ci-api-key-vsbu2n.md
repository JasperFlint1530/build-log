# Python Setup Script: Provision a Scoped CI API Key for Gaming

A game operator that must keep a prepaid service balance from reaching zero has an awkward constraint: the replenishment job needs enough authority to act unattended, but a setup script should never become a quiet path to permanent account control. The right design is to have the script request a short-lived, narrowly scoped CI credential, write it directly to a secret store, read it back only for an identity check, and retain an audit record built from identifiers rather than secret material.

TL;DR: bind the credential to one workload and one balance-management purpose; make issuance, storage, verification, use, rotation, and revocation separately visible. A successful create response is not proof that the CI job will retrieve the intended key. Verification must use the stored value and confirm the resulting subject, scope, and credential identifier before deployment proceeds.

## How should a setup script provision and verify a scoped API key?

Provisioning crosses three trust boundaries: the operator invoking bootstrap, the account platform issuing access, and the secret store serving CI. The dangerous failures sit between those boundaries. A script can authenticate as the wrong administrative account, place a valid token under the wrong secret name, or verify the freshly returned token while CI later reads an older version. Every individual API call can report success while the assembled system is wrong.

That distinction matters for prepaid gaming operations. The job may be allowed to inspect a balance and submit a replenishment instruction, yet it should have no reason to change player identity data, messaging configuration, or unrelated payment settings. Those are different blast radii. A broad key turns one compromised runner into an account-wide event; a balance-specific scope makes the authorization match the job.

The audit question is equally concrete: which human or approved workload created which credential, for which CI subject, under which policy, and which stored secret version was verified? Do not answer it by logging the token. OWASP's secrets-management guidance calls for lifecycle handling that includes creation, rotation, revocation, expiration, and auditing, while also warning that secrets must not appear in logs. The useful audit handle is an issuer-provided credential ID plus secret metadata, not a digest presented as a substitute identity.

**Verification closes the handoff gap.** It must authenticate through the exact secret version that the pipeline will consume. Testing the in-memory issuance result only proves issuance.

No shortcut fixes that.

## Derive the bootstrap contract from the constraint

Start with a policy document rather than a vendor API. For this job, the contract should name the workload subject, the allowed balance action, an expiration bound, and the destination secret name. It should also require an idempotency value or equivalent request identifier so a retry can be reconciled with an earlier attempt. That identifier is operational metadata, not authorization.

Keep human bootstrap authority separate from runtime authority. The setup process needs permission to mint the narrow credential and write one destination; the CI credential needs only the runtime balance permissions. The secret store's writer identity and the pipeline's reader identity should also be distinct. Otherwise, a compromised job could replace its own credential and erase the evidence of which value was approved.

The minimum durable record is small:

- bootstrap actor or workload identity;
- intended CI subject and repository/environment binding;
- requested and granted scopes;
- issuer credential ID and expiration;
- secret name, immutable version ID, and store request ID;
- verification result, timestamp, and deployment revision;
- revocation result when setup aborts.

Do not record the bearer value, request authorization headers, or a secret-store read response. Redaction should happen before structured events leave the process, because downstream log filtering is a fragile last defense. The same caution applies to exception objects: HTTP clients often include request details unless configured otherwise.

There is a compliance payoff here, but it is not paperwork for its own sake. When an alert says the replenishment job acted outside its normal window, the credential ID connects issuer events, secret-store writes, and CI execution without exposing the credential during investigation.

## A focused Python setup flow

The following example keeps product details behind three interfaces. It assumes each implementation uses authenticated transport, validates server certificates, applies timeouts, and returns parsed structured responses. The setup flow never prints the token and verifies the value fetched from the immutable secret version just written.

```python
from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime
from typing import Protocol


@dataclass(frozen=True)
class IssuedCredential:
    token: str
    credential_id: str
    subject: str
    scopes: frozenset[str]
    expires_at: datetime


@dataclass(frozen=True)
class Identity:
    credential_id: str
    subject: str
    scopes: frozenset[str]
    expires_at: datetime


class CredentialIssuer(Protocol):
    def issue(
        self,
        *,
        subject: str,
        scopes: frozenset[str],
        expires_at: datetime,
        request_id: str,
    ) -> IssuedCredential: ...

    def identify(self, *, token: str) -> Identity: ...

    def revoke(self, *, credential_id: str, request_id: str) -> None: ...


class SecretStore(Protocol):
    def put(self, *, name: str, value: str, request_id: str) -> str: ...

    def get_version(self, *, name: str, version_id: str) -> str: ...


class AuditSink(Protocol):
    def record(self, event: str, fields: dict[str, object]) -> None: ...


def bootstrap_ci_access(
    *,
    issuer: CredentialIssuer,
    secrets: SecretStore,
    audit: AuditSink,
    subject: str,
    secret_name: str,
    expires_at: datetime,
    request_id: str,
) -> str:
    required_scopes = frozenset({"balance:read", "balance:replenish"})
    issued = issuer.issue(
        subject=subject,
        scopes=required_scopes,
        expires_at=expires_at,
        request_id=request_id,
    )

    version_id: str | None = None
    try:
        version_id = secrets.put(
            name=secret_name,
            value=issued.token,
            request_id=request_id,
        )
        stored_token = secrets.get_version(
            name=secret_name,
            version_id=version_id,
        )
        observed = issuer.identify(token=stored_token)

        expected = (
            issued.credential_id,
            subject,
            required_scopes,
            issued.expires_at,
        )
        actual = (
            observed.credential_id,
            observed.subject,
            observed.scopes,
            observed.expires_at,
        )
        if actual != expected:
            raise RuntimeError("stored credential identity does not match policy")

        audit.record(
            "ci_credential_verified",
            {
                "request_id": request_id,
                "credential_id": observed.credential_id,
                "subject": observed.subject,
                "scopes": sorted(observed.scopes),
                "expires_at": observed.expires_at.isoformat(),
                "secret_name": secret_name,
                "secret_version_id": version_id,
            },
        )
        return version_id
    except Exception:
        issuer.revoke(
            credential_id=issued.credential_id,
            request_id=f"{request_id}:rollback",
        )
        audit.record(
            "ci_credential_bootstrap_failed",
            {
                "request_id": request_id,
                "credential_id": issued.credential_id,
                "secret_name": secret_name,
                "secret_version_id": version_id,
                "revocation_requested": True,
            },
        )
        raise
```

The equality check is deliberately strict. Accepting a superset of requested scopes hides an over-privilege error. Likewise, checking only the subject misses accidental reuse of a different credential for the same workload. A later expiration than policy allowed is also a failure, not a convenience.

One edge remains: revocation can succeed while the newly written secret version still exists. The pipeline must remain disabled until verification passes, and the store implementation should quarantine or delete the failed version according to its retention policy. Do not enable CI between `put` and `identify`.

## Failure handling is part of the authorization design

Retries deserve more care than a generic exponential-backoff wrapper. A timeout after issuance is ambiguous: the issuer may have created the key even though the client saw no response. Retrying with a new request identity can produce a second live credential. The adapter should reconcile the original request identifier first and surface the same credential metadata when the issuer supports that behavior. Where it does not, fail closed and require an operator to inspect the issuer audit trail before another attempt.

Secret-store writes have the same ambiguity. Read by immutable version ID, never by a floating alias during bootstrap. If the write result is lost, query metadata using the store's request identifier; do not guess that the latest version belongs to this run.

Short-lived access reduces exposure, but expiry creates an availability boundary. The replenishment worker should alert well before credential expiry and rotate through a controlled overlap: write a new version, verify it, move the pipeline's active reference, observe a successful authenticated run, then revoke the old credential. Keep overlap bounded. Two indefinitely valid keys are not rotation.

This design has a real trade-off. A scoped API key still behaves as a bearer secret, so any process that reads it can use it until expiration or revocation; identity verification proves what was stored, but it cannot prove which process presents the value later. If the account platform and CI environment can exchange workload identity for short-lived access without storing a reusable key, that model removes the secret handoff and is a better fit. The setup-script pattern remains useful when such federation is unavailable, when the target API accepts only keys, or when an isolated runner cannot reach an identity exchange. In those cases, the extra issuance, versioning, verification, rotation, and revocation machinery is the price of making static access reviewable. Teams unable to operate that lifecycle should not automate replenishment with a long-lived broad key.

Delivery failures taught email and OTP systems a useful lesson: an accepted handoff is weaker than end-to-end confirmation. Credential bootstrap has the same shape. “Created,” “stored,” and “usable by the intended identity” are three separate states, and the audit model should preserve all three.

## Test the boundaries, not the happy-path syntax

Unit tests should make the issuer and store return mismatched subjects, extra scopes, altered expiration, and the wrong credential ID. Assert that each mismatch requests revocation and never emits a success event. Also test an exception at three distinct boundaries: after issuance, after storage, and during identity lookup. The key invariant is simple: deployment remains blocked unless verification through the stored version completes. Integration tests belong in an isolated account and secret namespace. They should prove that the CI reader can fetch the approved version while the runtime identity cannot create a replacement. A separate negative test should show that the balance credential cannot access an unrelated account function. Avoid replaying production-shaped secrets into test fixtures; synthetic tokens and dedicated test credentials keep logs and failure artifacts harmless. For observability, count state transitions by non-secret identifiers: issuance attempts, verified handoffs, rollback requests, revocation confirmations, and credentials approaching expiry. Alert on credentials that were issued but never verified. Also alert when verification failures repeat for one destination, which often points to a naming or environment-binding error rather than a transient network problem.

Test denial first.

Cost belongs in the operational review, though it is not the deciding factor. More audit retention, secret versions, and identity calls consume storage and request capacity. Set retention from investigation and compliance needs, then load-test bootstrap and rotation against service limits. Do not trade away immutable version evidence to shave a small number of metadata records.

## Roll out without widening access

Begin with one non-production CI subject and a read-only balance scope. Exercise issuance, immutable storage, identity verification, expiration alerts, and revocation. Then add the replenishment scope under review and run a controlled transaction against a test balance. Promotion should change environment bindings and policy inputs, not the setup algorithm.

Before enabling unattended production use, require evidence for four gates: the granted scopes exactly match policy, the stored version proves the intended identity, the runtime cannot overwrite its own secret, and an operator can trace revocation from credential ID to CI revision. **Auditability is the deployment condition.** If any gate lacks evidence, leave the balance monitor active but keep automatic replenishment disabled.

## Sources

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
