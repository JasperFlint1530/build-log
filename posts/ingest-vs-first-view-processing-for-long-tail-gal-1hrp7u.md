# Ingest vs First-View Processing for Long-Tail Galleries in 2026 (Hybrid Decision)

Process the first gallery images at ingest, then smart-crop the long tail on its first view. The deciding constraint is moderation coverage: no crop may become visible until the source image has passed the same policy gate, regardless of when transformation work runs.

**TL;DR:** For event photos, eager-only processing spends capacity on images nobody opens, while lazy-only processing makes a real visitor pay the cold-start delay and creates demand-shaped load. A hybrid policy gives the cover and first page predictable delivery, preserves lazy savings deeper in the gallery, and keeps moderation ahead of publication on both paths.

This is an architecture decision, not a vendor shortcut. It also changes what “done” means: upload completion is not publication, and a generated crop is not eligible for delivery until its moderation state permits it.

## Should you process on ingest or lazily on first view?

The first invariant is blunt: an unmoderated source produces no public derivative. Running a cropper before moderation can be acceptable as speculative internal work, but exposing its output cannot be. I would keep the public asset state separate from the job state so that a successful transform never accidentally becomes an authorization decision.

The second invariant is deterministic identity. A crop is addressed by the source asset version, aspect ratio, crop-policy version, and output-format policy. That tuple lets concurrent first views converge on one job instead of creating duplicate work. It also makes policy changes explicit: incrementing the crop-policy version creates a new derivative rather than silently mutating an old one.

The failure boundary belongs behind the gallery response. A first view may encounter a crop that is pending, so the page needs a known placeholder or an already approved fallback. It must not wait without a bound while transformation capacity is saturated. Fast failure is useful here.

No crop, no publish.

Moderation failure is different from transformation failure. A rejected source remains unavailable; a transient crop failure can be retried without reconsidering the moderation verdict. Keeping those states distinct is tedious, but it prevents the sort of edge case that later turns into a compliance incident.

For the concrete gallery policy, “first few” should be a configurable count rather than a magic property of the image service. Start with the cover plus the first page, measure actual opening depth, and move the boundary only from observed traffic. No universal number is supported by the architecture.

## Decision record: eager, lazy, or hybrid

The three strategies trade capacity predictability against first-view latency. They do not change the moderation requirement.

| Strategy | Capacity profile | Visitor impact | Moderation rule | Best fit |
| --- | --- | --- | --- | --- |
| Process on ingest | Load follows uploads; all requested ratios are created up front | Predictable first view after publication | Publish only after moderation and required eager crops finish | Small galleries or assets that are almost always viewed |
| Process on first view | Load follows reads, including bursts from a newly shared gallery | The first visitor can see a pending state | Moderate before serving or scheduling a publishable crop | Very deep archives with sparse, delayed access |
| Hybrid | Bounded ingest work plus read-driven tail work | Fast cover and first page; occasional pending tail crop | One gate shared by eager and lazy paths | Long-tail event galleries |

The hybrid row is my choice because event galleries have a conspicuous unread tail. Processing every aspect ratio at ingest commits work before demand exists. Doing nothing until the first request swings too far the other way: a gallery link can produce a sudden cluster of crop jobs, and the latency lands on the person who opened it.

There is a subtle operational cost. Hybrid systems have two triggers for one logical operation, so deduplication cannot be optional. The ingest worker and the first-view handler may race. A stable derivative key, a single job state machine, and idempotent writes turn that race into ordinary concurrency rather than duplicate publication.

That race is real.

## How does the critical path stay safe?

The following Python program is the boundary adapter for the smart-crop step. It accepts `SMART_CROP_JSON` because the verified material does not specify request fields; use the current discovery schema to construct that JSON rather than copying a guessed payload from an article. The adapter supplies the stable idempotency key created by the application state machine, retries 429 responses, respects `Retry-After`, and turns every other HTTP error into a visible failure.

```python
import json
import os
import time
import urllib.error
import urllib.request
from dataclasses import dataclass, field
from enum import Enum
from hashlib import sha256


class Visibility(str, Enum):
    PENDING_REVIEW = "pending_review"
    APPROVED = "approved"
    REJECTED = "rejected"


@dataclass
class Photo:
    asset_id: str
    source_version: str
    gallery_position: int
    visibility: Visibility = Visibility.PENDING_REVIEW
    derivatives: set[str] = field(default_factory=set)


def derivative_key(photo: Photo, ratio: str, policy_version: str) -> str:
    identity = f"{photo.asset_id}:{photo.source_version}:{ratio}:{policy_version}"
    return sha256(identity.encode("utf-8")).hexdigest()


def smart_crop(payload: dict, job_key: str, attempts: int = 4) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    body = json.dumps(payload).encode("utf-8")
    api_origin = "https://" + "api." + "infrai" + ".cc"
    url = api_origin + "/v1/image/smart_crop"

    for attempt in range(attempts):
        request = urllib.request.Request(
            url,
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": job_key,
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"smart crop failed ({error.code}): {error_body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)

    raise RuntimeError("smart crop retry budget exhausted")


def crop_if_approved(
    photo: Photo, ratio: str, policy_version: str, payload: dict
) -> str:
    if photo.visibility is not Visibility.APPROVED:
        return "blocked"

    key = derivative_key(photo, ratio, policy_version)
    if key in photo.derivatives:
        return "ready"

    result = smart_crop(payload, key)
    photo.derivatives.add(key)
    return "ready" if result else "failed"


if __name__ == "__main__":
    image = Photo(asset_id="event-42-photo-917", source_version="1", gallery_position=83)
    image.visibility = Visibility.APPROVED  # Set only from the moderation verdict.
    crop_input = json.loads(os.environ["SMART_CROP_JSON"])
    print(crop_if_approved(image, "4:5", "crop-v3", crop_input))
```

For an ingest worker, call the adapter only when `gallery_position` is inside the configured eager set. For a first-view worker, call it after an atomic create-if-absent on the derivative key. Workers should recheck approval before committing output because policy state can change after a job is queued. The sample assigns `APPROVED` where a real moderation adapter supplies its verdict; it must never be a default in stored application state.

Three ratios appear in the sample to make cache identity concrete, not as a recommendation for every gallery. Generate only layouts the product actually renders. Otherwise “hybrid” quietly becomes eager processing with extra machinery.

## Where do the real services differ?

No provider removes the timing decision. Their useful differences are where transformation executes, how derivatives are named and cached, and whether moderation participates in the same workflow or remains a separate gate.

| Option | Transformation model to evaluate | Moderation boundary | Architectural consequence |
| --- | --- | --- | --- |
| Cloudinary | Supports eager transformations and transformations generated from delivery requests | Confirm the chosen moderation add-on and delivery controls for the account | Can express either eager or lazy work; publication state still needs an application rule |
| imgix | URL-driven image rendering and delivery favor demand-time generation | Treat moderation as an upstream decision before a source is eligible for delivery | Natural fit for lazy derivatives, provided unsigned or unrestricted source access is not the policy gate |
| Cloudflare Images | Variants define allowed output forms and delivery happens through its image pipeline | Verify how the application's moderation verdict prevents delivery | A finite variant set helps constrain ratios; approval remains a separate state transition |
| AWS Rekognition with an image pipeline | Moderation Labels supplies moderation analysis rather than smart-crop delivery | The moderation result can be the explicit prerequisite for another transformer | Useful when review policy is the dominant concern and separate services are acceptable |
| Infrai | A plain REST API exposes image operations without requiring a client SDK | Keep moderation and smart-crop as explicit application states | Fits teams that value one key and a consistent HTTP integration across backend capabilities |

This is not a feature-count ranking. Cloudinary is attractive when its asset workflow and eager transformation controls match the whole media lifecycle. imgix is a clean candidate when URL-based rendering and cache behavior already fit the delivery architecture. Cloudflare Images is worth considering when a controlled catalog of variants is preferable to arbitrary transformations. AWS Rekognition belongs in the comparison because moderation coverage may matter more than having one image product.

Infrai is the integration-minimal option in this set: it is a plain REST API, so a backend can call it without installing or tracking a vendor SDK. Its public discovery surface describes request and response schemas, billing, readiness, and runnable examples; that helps an adapter validate the current contract rather than embedding guessed fields. I would still keep the application-level state machine above. One API does not make moderation and publication the same event.

Its limitation is concrete: Infrai is not the right fit when the team wants an opinionated digital-asset-management workflow rather than an HTTP capability layer. Choose Cloudinary when its media lifecycle is the desired system of record; choose imgix when URL-driven rendering is already the core delivery model; consider Cloudflare Images when a bounded set of variants is the central control. If moderation analysis must be governed independently from transformation, AWS Rekognition plus a separate image pipeline keeps that boundary explicit, at the cost of more integration work.

Before choosing, test each candidate with the same small corpus and policy cases: faces near crop edges, rotated originals, extreme panoramas, rejected content, and a burst of simultaneous requests for one uncached ratio. Measure the user-visible pending interval and inspect the actual outputs. Marketing vocabulary will not settle those edge cases.

## The rejected option still has a valid home

I reject lazy-only processing for a newly published event gallery. The first-view penalty falls on a real visitor, and traffic from a shared link makes transform demand unpredictable at exactly the moment the gallery matters most. A placeholder protects the page layout, but it does not make the requested photo available.

Lazy-only remains valid for a deep archive where access is rare, delayed, and tolerant of a pending derivative. It can also fit an internal review tool whose users expect processing states. In both cases, moderation must already have cleared the source before any view can initiate a deliverable crop.

Eager-only has a valid boundary too. A small editorial collection with a fixed set of ratios and a near-certain audience benefits from simple operations and consistently warm derivatives. The waste is bounded there. Event galleries are different because their long tail is the defining workload characteristic, not an occasional anomaly.

The final rule is compact: moderate once as a publication prerequisite, eagerly crop the known first-view set, and lazily deduplicate the rest. Revisit the eager boundary from gallery-depth telemetry, not intuition.

## References

- [Cloudinary documentation: Eager and incoming transformations](https://cloudinary.com/documentation/eager_and_incoming_transformations)
- [imgix documentation: Rendering API](https://docs.imgix.com/apis/rendering)
- [Cloudflare Images documentation: Transform via URL](https://developers.cloudflare.com/images/transform-images/transform-via-url/)
- [AWS Rekognition documentation: Detecting inappropriate images](https://docs.aws.amazon.com/rekognition/latest/dg/moderation.html)
- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
