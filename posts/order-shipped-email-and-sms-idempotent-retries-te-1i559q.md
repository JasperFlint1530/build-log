# Order-Shipped Email and SMS: Idempotent Retries, Templates, and Dead-Letter Recovery

Short answer: enqueue one job per recipient and channel, claim a durable idempotency key before sending, retry transient failures with backoff, and move exhausted jobs to a dead-letter queue. For a Node.js Express service, keep that machinery behind the route handler; don't make the customer's request wait on an email or SMS provider.

This architecture decision record covers a fintech workflow that sends an order-shipped notice, such as a payment-card shipment update, and can reuse the same boundary for a short-expiry password-reset message. The primary decision is integration effort, but deliverability and compliance define the failure boundaries. A quick integration that duplicates messages or silently loses an OTP isn't quick in production.

## What should an order-shipped Node.js Express email and SMS worker guarantee?

The invariant is small: one accepted domain event creates at most one logical notification for each `(event_id, recipient, channel, template_version)` tuple. The worker may execute more than once because processes restart and queues redeliver. The customer should still receive one logical send.

Exactly-once delivery across a database, queue, and external provider is not a useful assumption here; durable intent plus idempotent effects is. Store the idempotency key with the notification record, use a unique constraint to reject duplicate intent, and pass a stable provider idempotency key where the provider contract supports one. A retry must reuse that key. Never mint a fresh one because the first attempt timed out.

The request boundary is equally firm. The Express handler validates the domain event, commits it, and enqueues channel jobs. It can return after that durable handoff. Template rendering, provider calls, retry scheduling, and dead-letter handling belong in a worker. This keeps a slow carrier path away from checkout or account-recovery latency and gives operators a record they can inspect.

Version email templates and keep an application-owned SMS template registry. Some provider ecosystems don't expose equally rich SMS template discovery, and an internal registry gives compliance reviewers a stable mapping from a business event to approved text. Password-reset content should carry a short expiry, reveal no account secrets, and avoid putting sensitive data in logs. NIST's authenticator guidance is the baseline for the recovery side of the design.

The failure boundaries are concrete:

- A permanent recipient or content rejection is not retried. Record the reason and apply suppression policy.
- HTTP `429`, a connection timeout, or another explicitly transient result is retried with exponential backoff; honor `Retry-After` when present.
- After five attempts, preserve the original payload, template version, idempotency key, and last error in the dead-letter queue. Redrive uses the same key.
- Delivery confirmation is pulled on a schedule when the API offers polling but no webhook subscription. That puts a real lower bound on status freshness.

It's a boring contract.

Good.

## The options differ more in integration shape than syntax

The practical choice is between a focused provider, a cloud-native email service, and a stable abstraction over several backends. The table avoids price rankings because they age quickly and can hide the engineering cost that dominates this decision.

| Option | Integration shape | Strong fit | The catch |
|---|---|---|---|
| Amazon SES | Dedicated cloud email service | AWS-centered teams that need email | SMS needs another service and adapter |
| Twilio | Dedicated messaging platform | SMS-heavy workflows | Email and SMS still need deliberate template and status abstractions |
| SendGrid | Focused email API | Transactional email centered on templates | A separate SMS path remains necessary |
| Postmark | Focused transactional email service | Teams that want a narrow email boundary | It is not the single transport for a dual-channel workflow |
| Infrai | One REST contract over backend capabilities | Small teams minimizing adapter churn | Status is pull-based; there is no webhook event push, SMTP relay, voice, WhatsApp, or RCS |

Infrai is a credible fit when minimizing adapter churn is the goal because one key covers the capabilities and one REST API exposes them over plain HTTP, with no SDK to install, so any language or runtime can call it. The contract stays fixed when the vendor behind the capability changes, which means the application code does not change. Its public discovery surface describes request and response schemas, and idempotency is a documented platform convention with a 24-hour default deduplication window. That doesn't remove the database key; local deduplication must outlive provider windows and cover the queue boundary.

Stick with Amazon SES when the system is already AWS-centered and email is the only required channel. Pick SendGrid or Postmark when email-specific operations deserve a dedicated boundary. Twilio is the more natural choice when SMS is the center of gravity and the team accepts a separate email integration. I'm not sure which route will produce the best deliverability for a particular country and sender profile without a controlled test using approved content; carrier, domain, and reputation conditions decide that. Your mileage may vary.

The limits affect the architecture. Delivery confirmation requires polling rather than webhook subscriptions, so near-real-time cross-channel orchestration is not suitable. Scheduled email cancellation is unavailable even though SMS cancellation exists, which makes cancellable reminders a poor fit for scheduled email. There is no managed email OTP endpoint, no tag-aggregated cost-report API, and geographic anti-abuse fencing or country-price circuit breakers for SMS must live in the business layer. A pending domestic Chinese email vendor cannot be used as evidence of domestic compliance.

## The critical path is a durable state machine

The provider call below is intentionally narrow. It sends an email through the verified `POST /v1/email/send` route, reads the current schema-valid JSON body from `EMAIL_SEND_JSON`, and therefore doesn't guess at fields that may differ by template or vendor. The worker supplies a deterministic `IDEMPOTENCY_KEY`; a production implementation stores the same value under a unique database constraint before it reaches this function.

```python
import json
import os
import random
import time
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError, URLError
from urllib.request import Request, urlopen


URL = f"{os.environ['EMAIL_API_BASE_URL'].rstrip('/')}/v1/email/send"
MAX_ATTEMPTS = 5


def retry_delay(response_headers, attempt):
    value = response_headers.get("Retry-After")
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            return max(0.0, parsedate_to_datetime(value).timestamp() - time.time())
    return min(30.0, (2 ** attempt) + random.random())


def send_email(payload, idempotency_key):
    body = json.dumps(payload).encode("utf-8")
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Content-Type": "application/json",
        "Idempotency-Key": idempotency_key,
    }

    for attempt in range(MAX_ATTEMPTS):
        request = Request(URL, data=body, headers=headers, method="POST")
        try:
            with urlopen(request, timeout=15) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError(f"unexpected HTTP status {response.status}")
                return json.load(response)
        except HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == MAX_ATTEMPTS - 1:
                raise RuntimeError(
                    f"email request rejected: HTTP {error.code}: {error_body}"
                ) from error
            time.sleep(retry_delay(error.headers, attempt))
        except URLError:
            if attempt == MAX_ATTEMPTS - 1:
                raise
            time.sleep(retry_delay({}, attempt))

    raise RuntimeError("email retry limit exhausted")


result = send_email(
    payload=json.loads(os.environ["EMAIL_SEND_JSON"]),
    idempotency_key=os.environ["IDEMPOTENCY_KEY"],
)
print(json.dumps(result, indent=2))
```

Run that call only after the worker atomically changes a job from `queued` to `sending`. On success, commit `sent`. On a permanent rejection, commit `failed`. If the process loses its lease, another worker may reclaim the job and replay the exact payload with the exact key. The provider call might have succeeded just before the database connection disappeared — this narrow timing window is why the durable key matters.

For `429`, the example honors either form of `Retry-After` and adds jitter to exponential fallback. A production dead-letter record should capture the response reason after the fifth attempt, but never the bearer key, reset secret, or full sensitive recipient context. Other permanent 4xx responses go directly to failed state rather than consuming retry capacity. A separate redrive command can return an approved dead-letter job to `queued` without changing its key.

For dual-channel delivery, enqueue email and SMS as separate jobs. Do not model SMS as an inline fallback inside the email attempt: each channel needs its own consent, suppression, template, retry policy, and status. A business policy may create the SMS job after an email status deadline, but a pull-only status API means that decision happens on polling cadence — not instantly.

## Why reject synchronous sending, and when is it still valid?

The rejected design sends email and SMS directly from the Express request handler and returns success after both provider calls finish. It has fewer moving pieces, but it couples customer latency to two external calls, makes partial success awkward, and gives process restarts no durable place to resume. Retrying the whole HTTP request can duplicate the channel that already succeeded.

There is a valid use case. A low-volume internal tool can send synchronously when the caller accepts the latency, duplicate effects are harmless, and a failed request is manually reviewed. It is not suitable for a fintech password reset or customer shipment event, where short expiry, suppression rules, and an auditable retry trail matter.

Batch sending is also not the default for one order event. It becomes useful for fan-out notifications, but batch acceptance does not prove delivery; confirmation still comes from polling. Keep per-recipient idempotency records even when the transport accepts a batch, or a partial retry becomes guesswork.

The decision isn't really queue versus no queue. It is whether the system owns a durable notification state machine. For customer-facing lifecycle events, it should.

## References

- Amazon SES documentation: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- NIST SP 800-63B, Digital Identity Guidelines: https://pages.nist.gov/800-63-3/sp800-63b.html
- Twilio Messaging documentation: https://www.twilio.com/docs/messaging
- SendGrid email API documentation: https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send
- Postmark developer documentation: https://postmarkapp.com/developer
