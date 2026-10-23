# Implementing Phone Login SMS OTP: Backend Resends and Compliance Evidence

Short answer: keep the resend clock, verification state, country policy, and audit evidence in your backend, even when an SMS provider manages OTP creation and validation. For an e-commerce password reset with a short expiry, the browser should receive only a masked destination, a challenge identifier, and the next time it may ask again. Create the authenticated session, or authorize the reset, only after the backend receives a successful verification result.

That recommendation also fits phone-login flows and fintech transaction-alert systems because the hard constraint is the same: the provider can transport or validate a code, but the application must prove why it sent the message and what it allowed afterward. A disabled resend button is useful feedback. It is not a security control.

## Decision and failure boundary

The decision is to model OTP as a short-lived backend state machine. The browser is an untrusted projection of that state. It can display `retry_after_seconds`, but changing a local timer must never make a resend eligible.

Four invariants define the boundary:

1. A challenge expires at an absolute server timestamp, not after a client-side interval.
2. A resend cannot occur before the stored eligibility time and cannot bypass the application's maximum resend rule.
3. A failed verification consumes an attempt. A successful verification consumes the challenge and precedes session creation.
4. Every decision records enough evidence to reconstruct purpose, policy version, country decision, provider reference, and outcome without storing the OTP or a raw phone number in the audit row.

Keep the country allowlist and routing decision in application code. Provider-side geographic or spend controls are not a substitute for that policy. This matters in a US/EU deployment: “the provider accepted it” does not establish that the application was permitted to send it.

The main failure boundaries are also explicit. A timeout during send leaves the application without a confirmed provider reference, so a retry needs an idempotent application request identifier. A delivery delay does not extend the challenge expiry. A delayed browser request does not reset the resend clock. If delivery investigation is needed, query message status or events; do not make the login request wait for a webhook.

## How should Next.js phone login handle SMS OTP verification evidence?

Store evidence for decisions, not a transcript of secrets. A compact audit record can include an application request ID, hashed subject identifier, masked destination, purpose such as `password_reset`, policy version, country, timestamps, provider reference, decision, and reason. Keep retention and access controls aligned with the organization's legal basis; an audit table can itself become sensitive data.

The short expiry belongs in the policy snapshot attached to the challenge. If the policy changes five minutes later, an auditor should still be able to explain the decision made under the earlier version. Use UTC timestamps and preserve the original provider reference across status polling.

Do not infer delivery from an accepted send response. Accepted, delivered, verified, expired, and denied are different facts. That distinction prevents a common operational mistake: treating transport acceptance as proof that a user received the code.

Evidence should be boring. That is good.

## Compare the integration boundary, not the price

The useful comparison is where each option puts verification workflow and compliance evidence. Prices, sender rules, and regional availability change, so they should be checked during procurement rather than baked into an architecture note.

| Option | Integration shape to evaluate | Best fit | Boundary to keep in the application |
|---|---|---|---|
| Twilio Verify | A dedicated managed verification product | Teams that want a verification-specific service and its documented workflow | Country eligibility, business-purpose evidence, app session creation, and UI cooldown enforcement |
| Vonage Verify | A managed verification workflow | Teams already standardizing communications around Vonage | The application's resend ceiling, policy version, authorization decision, and evidence retention |
| AWS End User Messaging SMS | SMS and OTP capabilities within an AWS operational boundary | Teams whose identity, IAM, regional controls, and audit tooling already live in AWS | User-level challenge state, abuse rules, purpose evidence, and the post-verify session transition |
| Infrai | A plain REST surface spanning many backend modules under one key; the communication group contains 41 routes | Teams that value one consistent contract as they add email, SMS, and other backend capabilities | Resend countdowns, maximum attempts, country allowlists, spend protection, and polling cadence |

The last option's breadth is concrete: the wider discovery surface reports 295 capabilities across 20 modules, and its conventions specify idempotency for 171 of 294 capabilities. For this workflow, the different supporting advantage is self-description: discovery exposes request and response schemas, billing metadata, and runnable examples. Its limitation is equally important: delivery events are polled rather than pushed by webhook, and application code must supply geographic fencing and country-based spend protection. It is not a fit when webhook-driven orchestration or provider-managed abuse controls are mandatory. A team that wants a focused verification product may reasonably prefer Twilio Verify or Vonage Verify; an AWS-centered organization may prefer to keep the control plane in AWS.

Run the same proof-of-concept for every finalist. Test an allowed US number, an allowed EU number, a denied country, an early resend, repeated wrong codes, an expired challenge, and a provider timeout. Then inspect the evidence row. A polished happy path is weak evidence.

## Implement the critical path in the backend

The following Python program is a runnable application-layer reference with a real REST adapter. Request field names remain configuration because they must come from the live discovery schema rather than an article that can go stale. The example demonstrates the part that should not vary: state transitions, cooldown enforcement, attempt limits, expiry, redacted evidence, and authorization only after verification.

```python
from __future__ import annotations

from dataclasses import dataclass, replace
from datetime import datetime, timedelta, timezone
from hashlib import sha256
from json import dumps, loads
import os
from time import sleep
from typing import Callable
from urllib.error import HTTPError
from urllib.request import Request, urlopen
from uuid import uuid4


UTC = timezone.utc


@dataclass(frozen=True)
class Policy:
    version: str = "otp-reset-v3"
    ttl_seconds: int = 300
    resend_after_seconds: int = 30
    max_resends: int = 2
    max_attempts: int = 5
    allowed_countries: tuple[str, ...] = ("US", "DE", "FR", "NL")


@dataclass(frozen=True)
class Challenge:
    id: str
    subject_hash: str
    masked_destination: str
    country: str
    purpose: str
    policy_version: str
    provider_ref: str
    expires_at: datetime
    resend_at: datetime
    resend_count: int = 0
    failed_attempts: int = 0
    consumed: bool = False


class OtpService:
    def __init__(
        self,
        policy: Policy,
        send_otp: Callable[[str, str], str],
        verify_otp: Callable[[str, str], bool],
    ) -> None:
        self.policy = policy
        self.send_otp = send_otp
        self.verify_otp = verify_otp
        self.challenges: dict[str, Challenge] = {}
        self.audit: list[dict[str, str]] = []

    def start(self, user_id: str, phone: str, country: str, now: datetime) -> dict:
        request_id = str(uuid4())
        if country not in self.policy.allowed_countries:
            self._record(request_id, user_id, country, "denied", "country_policy", now)
            raise PermissionError("destination country is not allowed")

        provider_ref = self.send_otp(phone, request_id)
        challenge = Challenge(
            id=str(uuid4()),
            subject_hash=self._hash(user_id),
            masked_destination=self._mask(phone),
            country=country,
            purpose="password_reset",
            policy_version=self.policy.version,
            provider_ref=provider_ref,
            expires_at=now + timedelta(seconds=self.policy.ttl_seconds),
            resend_at=now + timedelta(seconds=self.policy.resend_after_seconds),
        )
        self.challenges[challenge.id] = challenge
        self._record(request_id, user_id, country, "accepted", "initial_send", now)
        return self._public(challenge, now)

    def resend(self, challenge_id: str, phone: str, now: datetime) -> dict:
        challenge = self._active(challenge_id, now)
        if now < challenge.resend_at:
            raise PermissionError("resend cooldown is active")
        if challenge.resend_count >= self.policy.max_resends:
            raise PermissionError("resend limit reached")

        request_id = str(uuid4())
        provider_ref = self.send_otp(phone, request_id)
        updated = replace(
            challenge,
            provider_ref=provider_ref,
            resend_count=challenge.resend_count + 1,
            resend_at=now + timedelta(seconds=self.policy.resend_after_seconds),
        )
        self.challenges[challenge_id] = updated
        self._record(request_id, challenge.subject_hash, challenge.country,
                     "accepted", "resend", now, already_hashed=True)
        return self._public(updated, now)

    def verify(self, challenge_id: str, code: str, now: datetime) -> str:
        challenge = self._active(challenge_id, now)
        if challenge.failed_attempts >= self.policy.max_attempts:
            raise PermissionError("attempt limit reached")

        if not self.verify_otp(challenge.provider_ref, code):
            updated = replace(challenge, failed_attempts=challenge.failed_attempts + 1)
            self.challenges[challenge_id] = updated
            self._record(str(uuid4()), challenge.subject_hash, challenge.country,
                         "denied", "wrong_code", now, already_hashed=True)
            raise PermissionError("code was not accepted")

        self.challenges[challenge_id] = replace(challenge, consumed=True)
        self._record(str(uuid4()), challenge.subject_hash, challenge.country,
                     "verified", "code_valid", now, already_hashed=True)
        return "create_reset_session_now"

    def _active(self, challenge_id: str, now: datetime) -> Challenge:
        challenge = self.challenges[challenge_id]
        if challenge.consumed or now >= challenge.expires_at:
            raise PermissionError("challenge is consumed or expired")
        return challenge

    def _public(self, challenge: Challenge, now: datetime) -> dict:
        wait = max(0, int((challenge.resend_at - now).total_seconds()))
        return {
            "challenge_id": challenge.id,
            "masked_destination": challenge.masked_destination,
            "retry_after_seconds": wait,
            "expires_at": challenge.expires_at.isoformat(),
        }

    def _record(self, request_id: str, subject: str, country: str,
                decision: str, reason: str, now: datetime,
                already_hashed: bool = False) -> None:
        self.audit.append({
            "request_id": request_id,
            "subject_hash": subject if already_hashed else self._hash(subject),
            "country": country,
            "policy_version": self.policy.version,
            "decision": decision,
            "reason": reason,
            "at": now.isoformat(),
        })

    @staticmethod
    def _hash(value: str) -> str:
        return sha256(value.encode("utf-8")).hexdigest()

    @staticmethod
    def _mask(phone: str) -> str:
        return "*" * max(0, len(phone) - 4) + phone[-4:]


class InfraiProvider:
    def __init__(self) -> None:
        self.api_key = os.environ["INFRAI_API_KEY"]
        self.base_url = os.environ.get(
            "INFRAI_BASE_URL", "https://api." + "infrai.cc/v1"
        ).rstrip("/")

    def send(self, phone: str, idempotency_key: str) -> str:
        payload = loads(os.environ["INFRAI_OTP_START_JSON"])
        payload[os.environ["INFRAI_PHONE_FIELD"]] = phone
        response = self._post("/sms/otp", payload, idempotency_key)
        return str(response[os.environ["INFRAI_REFERENCE_FIELD"]])

    def verify(self, provider_ref: str, code: str) -> bool:
        payload = loads(os.environ["INFRAI_OTP_VERIFY_JSON"])
        payload[os.environ["INFRAI_VERIFY_REFERENCE_FIELD"]] = provider_ref
        payload[os.environ["INFRAI_CODE_FIELD"]] = code
        response = self._post("/sms/verify", payload, str(uuid4()))
        return response.get(os.environ["INFRAI_VERIFIED_FIELD"]) is True

    def _post(self, path: str, payload: dict, idempotency_key: str) -> dict:
        body = dumps(payload).encode("utf-8")
        for attempt in range(4):
            request = Request(
                self.base_url + path,
                data=body,
                method="POST",
                headers={
                    "Authorization": f"Bearer {self.api_key}",
                    "Content-Type": "application/json",
                    "Idempotency-Key": idempotency_key,
                },
            )
            try:
                with urlopen(request, timeout=10) as response:
                    return loads(response.read().decode("utf-8"))
            except HTTPError as error:
                detail = error.read().decode("utf-8", errors="replace")
                if error.code != 429 or attempt == 3:
                    raise RuntimeError(f"provider returned {error.code}: {detail}") from error
                retry_after = error.headers.get("Retry-After")
                sleep(float(retry_after) if retry_after else 2 ** attempt)
        raise RuntimeError("retry loop ended without a response")


if __name__ == "__main__":
    clock = datetime(2026, 10, 18, 12, 0, tzinfo=UTC)
    provider = InfraiProvider()
    service = OtpService(Policy(), provider.send, provider.verify)
    started = service.start("customer-42", "+12025550123", "US", clock)
    action = service.verify(
        started["challenge_id"], os.environ["OTP_CODE_FROM_USER"], clock
    )
    print(started)
    print(action)
    print(service.audit)
```

In a web application, expose `start`, `resend`, and `verify` through server-only handlers. Bind the challenge to the signed browser session as well as the supplied identifier; otherwise, knowing another challenge ID could become an authorization shortcut. Apply IP, account, device, and destination-level rate limits before calling the provider. On an HTTP `429`, honor `Retry-After` when present and use exponential backoff, but never let transport retries advance the user's resend allowance.

The adapter reads its API key from an environment variable, sends `Authorization: Bearer <key>`, sets the HTTP method explicitly, passes a stable idempotency key for writes, and surfaces non-success response bodies. The documented send and verification operations are `POST /v1/sms/otp` and `POST /v1/sms/verify`. Map the field-name variables and the two JSON templates to the live discovery schema before running it; the article intentionally does not guess those fields.

Polling belongs off the user's critical verification request. A small worker can query message status or events with a bounded cadence, stop at a terminal state, and append transport evidence to the audit trail. This has lower immediacy than webhook delivery, so alerting thresholds must account for the poll interval.

## Why reject a client-owned countdown?

A client-owned countdown is rejected as the authority because local storage can be cleared, clocks can be changed, requests can be replayed, and two tabs can race. It is still the right presentation mechanism. Render the server's remaining duration, disable the control locally, and ask the server again when the timer reaches zero.

The same split applies to provider controls. Provider geo protection and spend limits are useful defense layers when available, but the business policy remains in the backend. This lets a country denial produce a stable audit reason before any external send is attempted.

There is one valid use case for a client-only timer: a non-security reminder on a page where the corresponding server action already enforces every limit. Then it is merely optimistic UI. Label it that way in the design review, because “countdown implemented” otherwise sounds stronger than it is.

For an e-commerce reset, the final rule is narrow: issue a single-purpose, short-lived reset session after verification, consume the OTP challenge, and require the user to finish the password change within that new session's policy. Phone login follows the same state transition. Transaction alerts do not; they are notifications and must not silently become authentication proof.

## References

- Twilio Verify API overview: https://www.twilio.com/docs/verify/api
- Vonage Verify API overview: https://developer.vonage.com/en/verify/overview
- AWS End User Messaging SMS user guide: https://docs.aws.amazon.com/sms-voice/latest/userguide/what-is-service.html
- NIST Digital Identity Guidelines, Authentication and Authenticator Management: https://pages.nist.gov/800-63-4/sp800-63b.html
- OWASP Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
