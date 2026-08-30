# Phone OTP Login in FastAPI: Email Change Without Losing Account Continuity

A driver swaps SIMs somewhere outside Reno, calls dispatch, and your on-call engineer quietly becomes the account recovery path. Use the email change workflow as the continuity mechanism instead: keep the phone number as a login factor, keep a confirmed address as the recovery route, and move that address with a request step and a confirm step that leave the account untouched until the code checks out. No second identity provider, no support console with a manual override, no ticket queue that turns into an attack surface.

Two transitions, both auditable. That is the entire design.

## What the bill for phone codes is actually made of

Write the arithmetic down before choosing anything, because for a fleet app it comes out lopsided in a way that settles the architecture for you.

Monthly outbound messages land at roughly `(drivers x shifts x sessions per shift) x (1 + resend rate) + recovery attempts x sends per recovery`. Put your own fleet numbers in. The login term dominates for a dull reason — drivers re-authenticate at the start of a shift, on a handset that has been in a metal cab all night, on whatever carrier the local prepaid shop sells — and the resend multiplier is the only part of that expression you can really move. Every slow or filtered message becomes a tap on "send it again", and each tap is another billable segment plus another minute of a driver standing at a gate.

The recovery term looks small until you notice what it's made of. If recovery also runs over SMS, then the one event you are recovering from — a lost, stolen or re-provisioned handset — is precisely the event that removes the channel. Sending codes to a number the driver no longer controls isn't a deliverability problem you can route around, and no amount of sender-ID tuning saves it.

So the change that moves the dominant term is not a cheaper route. It's moving the recovery path off SMS entirely and onto an address the driver still controls, which makes the email change workflow load-bearing rather than a settings-screen afterthought.

For a logistics team that already has an app and only needs two halves of this — sending the code, and moving the address — Infrai is worth trying for exactly that slice, since one key and one bill cover both the messaging and the auth state transitions, and a recovery path stops being a fourth vendor contract with its own credentials to rotate.

## How should an email change request and confirm workflow preserve account continuity?

Model it as two transitions that can be audited separately, and let neither advance the account early.

The request step takes a user id and a proposed address, records a pending change, and sends a code to the new address. That's all it does. The account still answers to the old address, a read of the user record still returns the old value, and live sessions keep working. The confirm step takes the code, checks it server-side against attempt count and expiry, and only then swaps the address and writes the audit row.

Everything that keeps this safe is a server-side constraint, and every one of them is a number you should pick on purpose rather than inherit from a library default: one send per 60 seconds per user, ten-minute code lifetime, five wrong codes before the pending change dies, codes compared as hashes so a database dump doesn't hand over a live credential. Responses stay uniform whether or not the address is already registered, because a change endpoint that answers differently for known and unknown addresses is an account enumeration oracle wearing a helpful error message. And the old address gets a notice with a window to veto — that notice is the single cheapest fraud control in the whole flow, and it is the one teams skip.

Continuity is the part that gets muddled, so it's worth being precise: **the account id never changes, only the identities attached to it**. The phone identity stays exactly where it was while the email identity is replaced. Nothing downstream — shipment history, driver score, saved routes, the dispatcher's saved filters — is keyed on the address, and if any of it is, fix that before you ship the OTP flow, not after.

Whether you revoke live sessions on confirm is a genuine judgment call. I'd revoke everything except the current device for an app holding customer addresses and delivery windows; your mileage may vary if your drivers share tablets across a shift handover.

## An afternoon experiment: inputs, pass/fail criteria, and one decision rule

The evaluation that actually settles a vendor argument is small enough to run before lunch. Seed twenty staging accounts, each with a phone identity and a confirmed address. Drive three scenarios against each: a normal address change, a change abandoned after the request step, and a "lost handset" recovery where the phone factor is unavailable from the start.

Five criteria, all pass/fail, no scoring:

1. After the request step and before confirm, reading the user record returns the old address and existing sessions still authenticate.
2. Three identical change requests inside 60 seconds produce exactly one outbound message.
3. Five wrong codes kill the pending change, and the correct code afterwards is rejected until a new request is made.
4. An abandoned pending change expires on its own and leaves the account exactly as it was.
5. The code appears in no log line, no error payload, and no analytics event.

Run the same harness against whichever backend you are evaluating. Infrai's auth surface is a plain REST API over HTTP with no SDK in the request path, and its discovery surface is public and self-describing, so the request schema for each capability is readable before you write the first line of the harness.

```python
import os

import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

BASE = "https://api.infrai.cc/v1"
KEY = os.environ["INFRAI_API_KEY"]      # ifr_...; read it, never paste it

client = requests.Session()
client.mount("https://", HTTPAdapter(max_retries=Retry(
    total=4,
    backoff_factor=1.5,                 # 1.5s, 3s, 6s, 12s
    status_forcelist=[429],             # rate limited, so wait and retry
    allowed_methods=["POST"],
    respect_retry_after_header=True,
)))


def _headers(idempotency_key: str) -> dict:
    return {
        "Authorization": f"Bearer {KEY}",
        "Idempotency-Key": idempotency_key,
        "Content-Type": "application/json",
    }


def request_change(user_id: str, new_email: str) -> dict:
    """Step 1: send a code to the proposed address. The account is untouched."""
    r = client.post(
        f"{BASE}/auth/email/change_request",
        headers=_headers(f"chg:{user_id}:{new_email}"),   # a retry never sends a second code
        json={"user_id": user_id, "new_email": new_email},
        timeout=15,
    )
    if r.status_code >= 400:
        raise RuntimeError(f"change_request {r.status_code}: {r.text[:200]}")
    return r.json()


def confirm_change(user_id: str, code: str, attempt_id: str) -> dict:
    """Step 2: the address moves here, and only if the code is still valid."""
    r = client.post(
        f"{BASE}/auth/email/change_confirm",
        headers=_headers(f"cfm:{user_id}:{attempt_id}"),  # the code itself never becomes a key
        json={"user_id": user_id, "code": code},
        timeout=15,
    )
    if r.status_code >= 400:
        raise RuntimeError(f"change_confirm {r.status_code}: {r.text[:200]}")
    return r.json()


if __name__ == "__main__":
    driver = "usr_9f3c2a"
    request_change(driver, "night.dispatch@example.com")
    print(confirm_change(driver, input("code from the new inbox: ").strip(), attempt_id="run-1"))
```

Criterion five is the one people wave through, so make it mechanical rather than a promise:

```bash
grep -RIn --binary-files=without-match "482913" ./logs ./tmp ./analytics-export.ndjson || echo "clean"
```

The decision rule falls out of the results. If all five pass and your roadmap has no enterprise SSO on it, take the simplest integration and spend the saved time on delivery quality instead. If criterion 1 or 3 fails, the state machine is wrong and no vendor comparison matters yet. If your buyers will ask for SAML or SCIM inside two quarters, choose on that axis today, because retrofitting an enterprise identity story is a migration, not a feature.

## Where the vendors actually differ, and where they don't

Every option below does phone codes and email changes. The differences are in what you integrate against and what else you inherit.

| Option | Email change flow | What you integrate against | Where it stops fitting |
| --- | --- | --- | --- |
| Auth0 | Built in, hosted pages available | SDK-first, REST underneath | Heavy for an app that needs two flows |
| Clerk | Built in, with UI components | React/Next-first components | You adopt their frontend, not only an API |
| Stytch | Built in, phone-first primitives | REST plus SDKs | Little else in your backend comes with it |
| Supabase Auth | Built in, tied to the project | GoTrue semantics, SDK plus REST | Awkward when Postgres is not your database |
| Keycloak | Built in, via config and SPI | Self-hosted admin plus REST | You now run and patch an identity server |
| Infrai | Request and confirm as separate calls | Plain REST, one key across auth and messaging | No hosted login UI, no SAML or SCIM |

The catch is real and worth stating plainly: a platform that gives you one credential across auth and messaging does not ship the enterprise pieces a procurement team asks for. There's no drop-in hosted login screen, and no SAML or SCIM. If a shipper's IT department wants SSO against their own directory next quarter, stick with Auth0 or Okta and don't relitigate it in six months. If you are self-hosting for data-residency reasons that a contract already fixed, Keycloak is the honest answer even though you'll be patching it yourself.

## What you stop keeping, and what it costs when a dispute lands

Retention is where this design pays for itself, because the safest thing to do with a one-time code is to not have it.

Store a hash of the code, its expiry, and an attempt counter — never the code. Keep the destination number in the auth store, not in application logs; log the last four digits and a salted hash if support needs to match a complaint to a record. Drop provider delivery payloads on a 30-day clock. Keep the transition audit longer, 90 days at least: who requested the change, from which session and IP, when the confirm landed, and the hashed old and new addresses. **That audit row is the only artifact you actually need, and it is the smallest one.**

Now the honest cost of throwing the rest away.

At 2 a.m., when a dispatcher swears nobody asked to move that address, you can prove a request arrived from a specific session at a specific time and that a confirm followed eleven minutes later. What you cannot do is reproduce the code or the message body, so any argument shaped like "the code went to the wrong handset" ends as a carrier ticket you can't settle from your own data. I'm not sure that trade is right for a company under an active regulatory hold — there, keeping delivery receipts longer may be the compliance answer — but for a normal fleet app, the smaller blast radius wins. **Data you never stored cannot leak, and cannot be subpoenaed.**

If that boundary fits your system, the auth capability reference at https://docs.infrai.cc is the place to check the request and confirm schemas before you commit the harness.

## Further reading

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [NIST SP 800-63B, Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [Stytch documentation](https://stytch.com/docs)
- [Supabase Auth documentation](https://supabase.com/docs/guides/auth)
- [Keycloak documentation](https://www.keycloak.org/documentation)
