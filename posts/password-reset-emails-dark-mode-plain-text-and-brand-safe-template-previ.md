# Password Reset Emails: Dark Mode, Plain Text, and Brand-Safe Template Previews

## The call, and the two invariants it protects

Use a stored password reset email template with a real HTML part and a hand-written plain text part when the reset link has to survive Outlook, dark mode inversion and a screen reader on the same afternoon; otherwise reach for your provider's stock transactional template and leave it alone. I've shipped this flow at four companies now, and the version that caused the fewest support tickets was always the boring one: one template, one preview step in CI, no marketing copy anywhere near it.

Everything below hangs off two invariants.

The first: a user who never sees the HTML part must still be able to reset. That means the full reset URL appears as literal text — not behind "click here", not shortened, not wrapped in a tracking redirect that a corporate mail gateway will rewrite into something unrecognisable. Screen readers, text-only clients and the plain-text preview pane in older Outlook builds all land on the same path.

The second is duller: nothing ships without a render preview on real sample data.

Where these flows actually break is narrower than people expect. Gmail's dark mode inverts colours it thinks are backgrounds, so a navy button with white label text can come out as near-black on near-black. A logo PNG with a white matte turns into a bright rectangle floating in a grey message. Some clients strip `<style>` blocks entirely, which kills any `prefers-color-scheme` rule you wrote and leaves whatever inline styles survived. And the expiry sentence — the one piece of brand-safe copy that genuinely matters — gets edited by a designer who doesn't know it's load-bearing.

## How should a password reset email template handle dark mode without breaking the plain text part?

Treat dark mode as a rendering variant of the HTML part only, and never let it touch the text alternative. The text part has no styling to lose, which is exactly why it's the fallback that always works.

Concretely: inline every colour, put a `prefers-color-scheme` block in `<head>` as an enhancement rather than a requirement, and pick colours that stay legible after a forced inversion. I use a mid-tone brand colour for the button (not `#ffffff`, not `#000000`), an SVG or transparent PNG logo with real padding, and a border on the CTA so it still reads as a button when the fill gets mangled. Contrast should clear the WCAG 4.5:1 minimum in both directions — check it inverted too, which most people skip. Alt text on the logo, a real `<a>` element instead of an image link, `lang` on the `<html>` element, and a font size no smaller than 14px round out the accessibility side.

Then the copy. Say what happened, say when the link expires with an actual duration, and say what to do if the request wasn't theirs. NIST's guidance on authenticators is the reason the expiry window belongs in the message at all — a reset link is a short-lived authenticator, and the user needs to know its lifetime.

One paragraph, no promotions, no unsubscribe footer on a transactional message.

Here's my war story, and it's an unglamorous one. In my setup at a fintech, the reset sender wrapped every call in a retry loop that caught the transport exception and moved on. During a migration burst we pushed about 4,200 resets in ten minutes, the provider started returning 429, and the loop swallowed every one of them — no log line above debug, no metric. I spent most of a morning staring at delivery dashboards that showed nothing wrong because nothing had been sent. Roughly 60 users got no reset mail at all, and we only found out from support. I'm not sure why the client library defaulted to swallowing that status; either way, the fix was to honour `Retry-After` and alert on retry exhaustion, not to send harder.

## Which delivery layer fits which team

Four real options, plus one worth knowing about. No prices here on purpose — everyone's rate card moves, and the integration shape is what you actually live with.

| Option | How you integrate | Template + preview story | Main limitation |
| --- | --- | --- | --- |
| SendGrid | SDK or REST | Handlebars dynamic templates, versioned, preview in the UI | Template editing lives away from your repo |
| Postmark | SDK or REST | Layouts + templates, pushable from the CLI | Transactional only, by design |
| Resend | SDK-first, React Email | JSX components rendered from your codebase | Wants a JavaScript toolchain in the loop |
| Amazon SES | AWS SDK, SigV4 | Bare template store, minimal rendering | You build the rest yourself |
| Infrai | Plain REST, one key | Template create, update and preview over the same API | No SMTP relay; delivery events are pull-based, not pushed |

The Infrai row is the one I'd flag for polyglot teams: the email surface is a plain REST API with no SDK to install, so the same request works from a Node.js worker, a Python task queue or a Go cron job without three different client libraries drifting apart. If you want webhook-style push notifications on bounces, though, it doesn't support that — you poll the event list — and if your infrastructure is built around handing SMTP credentials to a legacy app, stick with SES or Mailgun.

## The critical path, end to end

Two calls: render the stored template against sample data, eyeball it, then send. The preview call is what I wire into CI, so a copy change that breaks the layout gets caught before a real user sees it.

```python
import json
import os
import time
import uuid

import requests

# Point this at your provider's REST base (the one ending in /v1) - no SDK needed.
BASE = os.environ["EMAIL_API_BASE"]
HEADERS = {
    "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
    "Content-Type": "application/json",
}
TEMPLATE_ID = os.environ["RESET_TEMPLATE_ID"]

SAMPLE = {"first_name": "Alice", "reset_url": "https://app.example.com/r/demo", "expires_in": "15 minutes"}


def preview(template_id: str, variables: dict) -> dict:
    """Render the stored template so CI can diff it before anyone sends it."""
    r = requests.post(
        f"{BASE}/email/template/preview/{template_id}",
        json=variables,
        headers=HEADERS,
        timeout=20,
    )
    if r.status_code >= 400:
        raise RuntimeError(f"preview rejected: {r.status_code} {r.text[:200]}")
    return r.json()


def send_reset(to: str, subject: str, html: str, request_id: str) -> dict:
    """POST the reset mail. Idempotency-Key makes a retry safe to repeat."""
    headers = {**HEADERS, "Idempotency-Key": request_id}
    delay = 1.0
    for _ in range(5):
        r = requests.post(
            f"{BASE}/email/send",
            json={"to": to, "subject": subject, "html": html},
            headers=headers,
            timeout=20,
        )
        if r.status_code == 429:
            time.sleep(float(r.headers.get("Retry-After", delay)))
            delay *= 2
            continue
        if r.status_code >= 400:
            raise RuntimeError(f"send rejected: {r.status_code} {r.text[:200]}")
        return r.json()
    raise RuntimeError("rate limited after 5 attempts - alert, do not drop silently")


if __name__ == "__main__":
    rendered = preview(TEMPLATE_ID, SAMPLE)
    print(json.dumps(rendered, indent=2)[:400])
    print(send_reset("alice@example.com", "Reset your password", "<p>Hi Alice</p>", str(uuid.uuid4())))
```

Note the `Idempotency-Key`: a reset request that gets retried by your own queue must not produce two live links. And send reset mail immediately rather than scheduling it — scheduled sends here don't support cancellation, so a delayed reset job is a design you can't take back.

Domain setup does more for inbox placement than any of this. Verified domain, aligned DKIM and SPF, a From address on a subdomain you actually control — Google's sender guidelines are the checklist, and password reset mail that lands in spam is a support cost, not a deliverability metric.

## The option I rejected, and when it's still right

I rejected build-time rendering — MJML or React Email compiled into the app, HTML shipped inline with each send, no server-side template store. It's a genuinely good pattern: templates get code review, diffs are readable, and the preview is a local build step with no API call at all.

The catch is who can change the copy. Every wording tweak becomes a deploy, and in two of the four teams I mentioned, that meant legal-approved copy sat in a branch for a week. A stored template with an update endpoint lets compliance change the expiry sentence without a release train.

So: if your reset copy is stable and your deploys are fast, compile it. If copy changes come from outside engineering, store the template server-side and version it there. Your mileage may vary on that one — it's an org-shape decision more than a technical one.

## References

- [Google: Email sender guidelines](https://support.google.com/a/answer/81126)
- [NIST SP 800-63B, Digital Identity Guidelines: Authenticators](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [MDN: prefers-color-scheme](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-color-scheme)
- [Can I email: prefers-color-scheme support](https://www.caniemail.com/features/css-at-media-prefers-color-scheme/)
- [WCAG 2.1: Understanding contrast (minimum)](https://www.w3.org/WAI/WCAG21/Understanding/contrast-minimum.html)
- [Postmark transactional email templates](https://postmarkapp.com/transactional-email-templates)
