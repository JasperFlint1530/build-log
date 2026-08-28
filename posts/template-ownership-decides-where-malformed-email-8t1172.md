# Template Ownership Decides Where Malformed Email and SMS Notification Payloads Fail

Pick the owner of every notification template before writing a line of validation code. If support operations edit the copy, the variables that copy references are their contract, and the service has to check each event payload against the current template revision at ingestion. If engineers own the copy in the repository, the same check belongs in the build instead. An invalid phone number, a missing template variable, an event no JSON schema ever described — the debugging story for all three depends on that one ownership decision, because it decides which side of the boundary the truth lives on.

## The constraint: a fintech contact form, three support queues, one rendering boundary

The system is small and unglamorous. A contact form on a consumer lending product collects a name, an email address, an optional mobile number, a masked account reference, and a category: billing, fraud, or general. Each submission becomes one event on a queue. Two notifications come out of it — an email to the mailbox that owns the category, and, for fraud only, an SMS to the analyst on call.

The transport isn't the hard part. The constraint is that support operations rewrite the wording of those messages every few weeks, after a compliance reviewer signs off, and neither of them ever looks at the JSON moving through the queue.

Here is how that bites. A reviewer asks for "account reference" to be spelled out in the fraud SMS; support edits the template and, while editing, renames the placeholder from `account_ref` to `account_reference` so the variable reads the way the sentence does. Nothing in the pipeline objects. The payload still validates against the JSON schema it was written for, the address is still a real mailbox, the number is still E.164, and the renderer substitutes an empty string for a name it doesn't recognize. So the analyst on call gets an SMS at 02:00 about a suspected fraud case with a hole in the middle of it, and the investigation starts from the wrong end, because the payload is the artifact engineers can actually see.

The payload was never malformed. The contract moved.

## What does a Node.js worker check in an email or SMS notification payload before it renders the template?

Four checks, in this order, and none of them is the provider's job.

Envelope shape comes first, and JSON Schema is the right tool as long as you know what `format` does. Under the 2020-12 draft, `format` is an annotation by default: a validator implementing only the Format-Annotation vocabulary accepts `"not-an-address"` for `"format": "email"` without complaint. In Node.js that usually means a validator such as Ajv with `ajv-formats` registered, or an explicit assertion mode — otherwise the schema documents an intent it never enforces, which is worse than no schema at all, since everyone downstream now believes addresses were checked.

Recipients come second, per channel. Address syntax under RFC 5322 is far wider than the regex most services ship, so the useful boundary check is narrow syntax plus a resolvable domain, with real deliverability judged later by bounce handling rather than guessed at ingestion. Phone numbers need E.164 at the boundary: a leading `+`, a country code, at most fifteen digits, no separators. Never infer the country code from server locale. In a lending product, `07946 0018` and `+44 7946 0018` and `+1 7946 0018` are three different claims about who receives the message, and only one of them is the customer.

Third, the template variable set, which is where this article started.

Fourth, channel limits that the copy owner has no reason to think about: GSM-7 versus UCS-2 segmentation for SMS, so a curly apostrophe pasted from a review document doesn't silently triple the segment count, and a `List-Unsubscribe` header with one-click support on the email side, which Google's bulk sender guidelines require of senders above their volume threshold along with authenticated domains and a spam rate held under 0.3%.

Debugging is a matter of what you kept. A diagnostic record for a rejected event needs the event id, the category, the channel, the template key, the template revision, and the sorted JSON pointers of the fields that were rejected. It should not carry the rendered body, the full mobile number, or the account reference — this is fintech, and a log store is not a place to build a second copy of customer data. Keep the payload and the revision id together so the case can be replayed exactly; a report that says only "render failed" cannot tell you which of the two contracts drifted, and that ambiguity is the entire cost of the incident.

Log the pointer, not the value.

## Deriving the variable contract from the copy support edits, with a runnable example

A hand-maintained list of required variables is a third contract, and it drifts from the other two. Read the requirement out of the template revision instead:

```python
"""Contact-form notification guard: payload checked against the live template revision."""
import json
import re
from string import Formatter

E164 = re.compile(r"^\+[1-9]\d{6,14}$")            # ITU-T E.164: 15 digits maximum
ADDRESS = re.compile(r"^[^@\s]+@[^@\s.]+(?:\.[^@\s.]+)+$")

# Copy owned by support ops, pulled from the template store with its revision id.
TEMPLATES = {
    ("fraud", "sms"): {
        "revision": "2026-03-11",
        "body": "Fraud case {case_id} on account {account_reference}. Reply STOP to opt out.",
    },
    ("fraud", "email"): {
        "revision": "2026-03-11",
        "body": "Queue: fraud\nCase {case_id}\nAccount {account_ref}\nReply-to {contact_email}",
    },
}


def required_variables(body: str) -> set:
    """The contract is whatever the current copy references."""
    return {name for _, name, _, _ in Formatter().parse(body) if name}


def check(event: dict) -> list:
    key = (event.get("category"), event.get("channel"))
    template = TEMPLATES.get(key)
    if template is None:
        return [f"/category,/channel: no template registered for {key}"]

    problems = []
    revision = template["revision"]
    recipient = event.get("recipient", "")
    if key[1] == "sms" and not E164.fullmatch(recipient):
        problems.append("/recipient: not E.164 (+countrycode then subscriber number)")
    if key[1] == "email" and not ADDRESS.fullmatch(recipient):
        problems.append("/recipient: not a routable address")

    variables = event.get("variables") or {}
    required = required_variables(template["body"])
    for name in sorted(required - variables.keys()):
        problems.append(f"/variables/{name}: required by template revision {revision}")
    for name in sorted(variables.keys() - required):
        problems.append(f"/variables/{name}: unused by template revision {revision}")
    return problems


event = {
    "event_id": "cf_20260311_0042",
    "category": "fraud",
    "channel": "sms",
    "recipient": "+442079460018",
    "variables": {"case_id": "FR-4471", "account_ref": "****8813"},
}
print(json.dumps({"event_id": event["event_id"], "problems": check(event)}, indent=2))
```

Two lines come back: `account_reference` is required and absent, `account_ref` is present and unused. The second line is the one that earns its keep, because a validator that only reports missing variables tells you the producer is broken when the copy is what changed. Run the same function over every stored revision in CI, with a fixture event per category, and the rename gets caught by the support team's own save action rather than by an analyst at 02:00.

## Three ownership models, and how each one behaves in operations

| Template ownership | Who edits the copy | Where the contract is enforced | Failure mode to watch |
| --- | --- | --- | --- |
| Repository | Engineers | Build, plus unit tests over fixtures | Every wording fix waits for a deploy |
| Console or template store | Support and compliance | Ingestion, against the stored revision | Placeholder drift between revisions |
| Split: copy in the store, manifest in the repo | Support writes text, engineers own the manifest | Both sides, and they have to agree | Two sources of truth to reconcile |

The catch with repository ownership is organizational, not technical: a compliance-driven wording change turns into a pull request, a review, and a release, and the reviewer who asked for it has no way to see the result until it ships. Console ownership inverts the trade-off — copy changes land in minutes, and the variable contract becomes a runtime concern that only ingestion checks can defend.

Stick with repository ownership when the message text is itself regulated output that a compliance officer must sign off in a versioned artifact, such as a statutory notice. Move to a store with revisions when the copy changes faster than your release train, and accept that you now owe the system a contract check on every event. The split model reads like the best of both, though I'd be careful with it: it only works if the manifest is generated from the copy rather than typed next to it, and I'm not sure any team keeps two hand-written lists aligned for long.

## Rollout on a queue that is already moving

Ship the checker in observe-only mode first, recording rejected pointers and template revisions without dropping anything. A week of that data tells you whether the drift is in the producer, the copy, or a category nobody has sent since launch.

Then pin the template revision into each queued event at enqueue time, so a message that waits an hour renders against the revision it was validated for. Enforce at ingestion for new events, keep the worker-side check for anything already in flight, and drain old revisions before retiring them. Build the fixture corpus from realistic shapes with no live recipients or account data, and include the mutations that actually happen: a renamed placeholder, a number in national format, an address with a trailing space, a category with no template, an empty variables object.

Fraud alerts first, since the on-call analyst is the one who notices silence. Then the rest.

## References

- [Google: Email sender guidelines](https://support.google.com/a/answer/81126)
- [NIST SP 800-63B: Digital Identity Guidelines, authenticators](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [ITU-T Recommendation E.164: the international public telecommunication numbering plan](https://www.itu.int/rec/T-REC-E.164)
- [JSON Schema 2020-12: validation vocabularies, including format](https://json-schema.org/draft/2020-12/json-schema-validation)
- [RFC 5322: Internet Message Format](https://datatracker.ietf.org/doc/html/rfc5322)
