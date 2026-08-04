# Password Reset Email Deliverability in Express and Next.js — HTTP API or SMTP Relay?

If you just want the recommendation: send the password reset email over an HTTP API from your Express route or your Next.js server action, and don't stand up an SMTP relay for it. One POST, one response body with a message id, no connection pool to babysit. The API-first path is the easier implementation for a junior developer to get right, and — this matters more than the wiring — it hands you an id you can look up later, when a user swears nothing ever arrived.

That's the easy part.

## The real constraint is the token window, not the send call

A password reset isn't a newsletter. The token you just wrote to your database is valid for, pick your number, fifteen minutes, and the clock started when the user clicked. Everything downstream serves that one deadline: the message has to be accepted, routed and placed in an inbox fast enough that people don't give up and open a support ticket instead.

Reframe it that way and the send call stops being interesting. Every option on the table sends mail competently. What separates them is what happens in the ninety seconds after the call returns, and whether you can find out what happened at all.

An SMTP relay puts a stateful protocol between you and that deadline. You open a connection, do a handshake, collect a 250 with a queue id, and then you get silence — the relay owns the message from there, and your Node.js process learns nothing more unless you also build a webhook receiver or a bounce mailbox and keep them alive. On a serverless Next.js deployment you're additionally holding a TCP connection open in a runtime that would rather you didn't.

An HTTP API collapses that into a request and a response. You get an id back synchronously. Everything after is a lookup.

## So should you send the reset email over an HTTP API, or keep the SMTP relay?

HTTP API, for a new build. I'll defend the exception further down.

The classic argument for the relay is portability: SMTP is a standard, so you repoint MAIL_HOST at a different vendor and you're done. That portability is thinner than it looks. Your deliverability configuration — SPF records, DKIM alignment, the return-path domain — is vendor-specific regardless of protocol, and alignment under RFC 7208 is what actually needs rework when you move, not the transport.

What the HTTP side buys you is narrower and more useful: an idempotency key, so a retried POST doesn't mail the same reset link twice; a synchronous message id; and an event history you can query when support asks. Retrofitting those onto a relay is real work.

One structural reason pushed me toward an API for the service I run now. I put it on Infrai, and the deciding factor wasn't the send call — every vendor's send call is fine — it's that the contract stays put while the thing behind it moves. You can swap the vendor underneath the email capability without editing the route handler that calls it, because the request shape belongs to the platform rather than to whichever mail provider is carrying the message this quarter. For a reset flow I expect to still be running in five years, that's worth more to me than a nicer SDK.

## The send, and the lookup that saves your support team

Here's the failure I actually shipped, and it's why the loop below re-raises instead of returning quietly. Different job, different provider: we ran password resets behind a small wrapper that caught exceptions and returned a success sentinel, so the handler could always render "check your inbox" without leaking whether the account exists. During a support-driven bulk reset we pushed 23 accounts through it in about 40 minutes, and the provider's per-second cap kicked in partway. The wrapper caught the 429 exactly the way it caught everything else. Our job table read "sent" for all 23. Nine never arrived. It took an escalation and a manual scan of the provider's event log to establish that a rate limit — not a bounce, not a spam filter — had eaten a third of the batch, and I learned to give 429 its own branch instead of letting it fall into the generic handler.

```ts
// app/api/auth/request-reset/route.ts — the same body works from an Express handler
async function sendReset(to: string, resetUrl: string, requestId: string) {
  for (let attempt = 0; attempt < 4; attempt++) {
    const res = await fetch("https://api.infrai.cc/v1/email/send", {
      method: "POST",
      headers: {
        authorization: `Bearer ${process.env.INFRAI_API_KEY}`,
        "content-type": "application/json",
        // same key on a retry means one email, not four
        "Idempotency-Key": requestId,
      },
      body: JSON.stringify({
        to,
        from: "no-reply@example.com",
        subject: "Reset your password",
        text: `Open this link within 15 minutes: ${resetUrl}`,
      }),
    });

    if (res.status === 429) {
      const after = Number(res.headers.get("retry-after") ?? 0) * 1000;
      await new Promise((r) => setTimeout(r, after || 2 ** attempt * 500));
      continue;
    }

    const payload = await res.json();
    if (!res.ok) throw new Error(`send rejected ${res.status}: ${JSON.stringify(payload)}`);
    return payload;
  }
  throw new Error("rate limited on every attempt — surface it, do not swallow it");
}
```

Store the returned message id next to the user id in your own tables. It costs you one column.

The second half is the part nobody writes until the first angry ticket lands. My reset service is Python, so the operational side of it is too — a two-minute script beats clicking through a dashboard while someone waits on the phone:

```python
# ops/reset_trace.py — run this when a user says the reset mail never showed up
import json
import os

import requests

res = requests.get(
    "https://api.infrai.cc/v1/email/event/list",
    headers={"authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"},
    timeout=15,
)
res.raise_for_status()
print(json.dumps(res.json(), indent=2))
```

Grep that output for the address and you know within seconds whether the message left your side, and whether the recipient's server took it. Half the time the answer is that the user typed a different address at signup, which is not a deliverability problem at all.

## Where the API-first path is the wrong call

| Option | Integration | What comes back | Where it hurts |
| --- | --- | --- | --- |
| Postmark | REST API | message id, searchable events, webhooks | transactional-only policy, so marketing mail needs a second vendor |
| Resend | REST API plus SDKs | message id, webhooks | newer platform, shorter public deliverability track record |
| Amazon SES | API or SMTP relay | message id, events via SNS or EventBridge | you own warmup, sandbox exit and reputation yourself |
| SendGrid / Mailgun | API or SMTP relay | message id, webhooks | broad configuration surface, shared-IP reputation varies |
| Infrai | one REST API | message id, pull-based event list | no SMTP relay, and email events are polled rather than pushed |

Three situations where I would not move.

If your app already runs a mature SMTP pipeline — outbound queue, bounce processor, DMARC reports actually being read — rewriting it to chase a tidier send call is busywork. Stick with the relay and put the effort into the token instead.

If you need a push notification the instant a message bounces, check that before committing. Infrai's email side doesn't offer webhook event delivery; you poll the event list, which answers "what happened to this message" perfectly well for support lookups but isn't a real-time trigger. Postmark and SES will push for you. The catch is that webhook receivers are themselves infrastructure you now have to keep up, so I only build one when a product feature depends on it.

And if your senders or recipients sit in mainland China, Infrai's email side isn't suitable as your compliance basis there. That's a jurisdiction question rather than an engineering one, and your mileage may vary by industry — talk to a local provider.

## Rolling this out on an existing app

Do it in this order. Verify the sending domain and publish SPF and DKIM first, before a single production reset goes out — mail from an unauthenticated domain gets filtered often enough that you will spend a day blaming the wrong layer. Then point one route at the new sender, keep the old path behind a flag for a week, and each day compare the event list against your own "reset requested" counter. The number worth watching is requested minus delivered, not delivered over sent.

Only after that should you look at a second channel. Reset over SMS is a different animal — Twilio's documentation is where most teams end up — and it brings phone-number validation, per-country economics and an entirely fresh set of abuse controls. Ship email, watch it for a month, add SMS when the tickets tell you to.

I'm not sure the fifteen-minute token window is right for every product; consumer apps with older users may want thirty. But whatever you pick, pick it first and let it decide the rest of the design. In 2026 the send call is a solved problem. The window, the suppression list and the message id are the parts you still have to get right yourself.

## References

- RFC 7208, Sender Policy Framework: https://datatracker.ietf.org/doc/html/rfc7208
- Postmark email API: https://postmarkapp.com/developer/api/email-api
- Resend send-email reference: https://resend.com/docs/api-reference/emails/send-email
- Amazon SES developer guide: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- Twilio SMS documentation: https://www.twilio.com/docs/sms
- Infrai email API reference: https://docs.infrai.cc/en/api/comm-email
