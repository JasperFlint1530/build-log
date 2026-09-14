# PDF Endpoints a US/EU SaaS Can Use — Customer Identity Evidence That Holds Up

Short answer: a US/EU SaaS handling customer identity verification should use separate PDF endpoints to fill, render-check, flatten, and finally sign each form; keep the signed result only for the period its policy can justify, while storing a smaller append-only audit record for longer-lived evidence. This ordering protects signature validity, catches visual damage before commitment, and keeps privacy risk from growing with an undifferentiated document archive.

The bill has four parts: processing calls, transient transfer, retained document bytes, and the operational labor needed to explain a disputed verification. For a steady workload, retained bytes are approximately `documents per day × average signed bytes × retention days`, plus replicas and backups. That equation should be calculated from production histograms, not a brochure's sample file. If a planning case moves raw-PDF retention from 30 days to 7, its steady-state primary-document footprint falls by 23/30, or about 76.7%; call volume and audit-event volume do not. This is a capacity example, not a claimed benchmark.

**Choose the evidence boundary before choosing endpoints.** The PDF is sensitive input and, sometimes, a signed business record. It shouldn't automatically become the permanent system of record.

## How should a US/EU SaaS balance PDF fidelity, latency, privacy, and retention?

Treat the workflow as a state machine with four capability boundaries: field filling, visual rendering, flattening, and digital signing. They may be delivered by one service or several components, but their contracts should remain separate. A fill operation maps validated values into named form fields. A render operation produces page images for inspection. A flatten operation resolves form appearances into fixed page content and removes the application's dependence on interactive widgets. A signing operation covers the final bytes and returns signature metadata.

Order matters here.

Fill first, then render and inspect, then flatten, then render once more, and sign last. Any byte-level modification after signing can invalidate the signature or make validation ambiguous, so post-signing optimization is out. Keep the unsigned intermediate in short-lived encrypted workspace storage, not beside completed evidence.

Fidelity belongs in an automated acceptance test, not in a human spot-check queue. Compare page count and dimensions, confirm that every required field produced a visible appearance, reject text that exceeds its field boundary, and render with the same font assets used by the processor. A pixel comparison can be useful, but dynamic timestamps and anti-aliasing make a zero-difference rule brittle. Define masks and tolerances from controlled fixtures, and route threshold breaches to review without pretending that OCR alone proves layout correctness.

Latency has two clocks. The customer-facing clock ends when the verification UI can safely advance; the evidence clock ends when the final signed artifact and audit event are durably recorded. Don't hold an interactive request open through large-document rendering and signing. Return an opaque job identifier, process the document asynchronously, and expose a status resource or signed callback whose authenticity and replay rules are specified. A delivery mindset helps here: retries without idempotency are duplicates waiting to happen, much like resending an OTP without knowing whether the first message was accepted.

The privacy control is purpose limitation translated into system behavior. Under GDPR Article 5, personal data should be adequate, relevant, limited to what is necessary, and kept in identifiable form no longer than necessary. US obligations vary by sector, state, contract, and the type of identity evidence, so there isn't one defensible universal retention number. Counsel and the data owner must resolve that uncertainty; engineering should provide distinct, enforceable policies rather than a hard-coded default.

## Make the signature cover the artifact users actually saw

A digital signature and an audit trail answer different questions. The signature can show whether the final PDF changed after signing and who controlled the signing credential under the chosen trust model. The audit trail explains how the artifact came to exist: which template revision was used, which subject and tenant initiated the job, what policy approved it, which transformations ran, and when the retention clock started. One cannot substitute for the other.

For an identity-verification form, record hashes rather than extra copies wherever a hash is sufficient. A useful event includes a randomly generated correlation identifier, template version, input-object digest, final-object digest, ordered transition names, UTC timestamps, policy version, signer certificate reference, and outcome. Avoid raw field values, document images, access tokens, callback secrets, and full government identifiers in logs. Even an email address may be unnecessary if the event can refer to an internal subject identifier.

The signature profile must match the relying party's legal and archival needs. The EU eIDAS framework distinguishes electronic-signature levels, while ETSI's PAdES specifications define PDF advanced electronic signature profiles. A US/EU service shouldn't label every visible signature image a digital signature, and it shouldn't promise a legal effect merely because a library produced a signature dictionary. Document the certificate chain, timestamping and revocation-validation expectations, supported PDF profile, and validation horizon with legal and security owners.

Signing comes last.

Run it only after the second render check passes. Then validate the returned signature independently before publishing the artifact. That's the commitment point.

The code below keeps provider details behind a narrow interface and makes the sequencing testable. Its methods are capability names, not claims about any commercial route.

```python
from dataclasses import dataclass
from hashlib import sha256
from typing import Mapping, Protocol


class PdfProcessor(Protocol):
    def fill_form(self, template: bytes, fields: Mapping[str, str]) -> bytes: ...
    def render_pages(self, document: bytes) -> list[bytes]: ...
    def flatten_form(self, document: bytes) -> bytes: ...
    def sign_document(self, document: bytes, profile: str) -> bytes: ...
    def validate_signature(self, document: bytes) -> bool: ...


@dataclass(frozen=True)
class EvidenceResult:
    signed_pdf: bytes
    input_digest: str
    final_digest: str


def build_evidence(
    processor: PdfProcessor,
    template: bytes,
    fields: Mapping[str, str],
    profile: str,
) -> EvidenceResult:
    filled = processor.fill_form(template, fields)
    assert_visual_contract(processor.render_pages(filled))

    flattened = processor.flatten_form(filled)
    assert_visual_contract(processor.render_pages(flattened))

    signed = processor.sign_document(flattened, profile)
    if not processor.validate_signature(signed):
        raise ValueError("signature validation failed")

    return EvidenceResult(
        signed_pdf=signed,
        input_digest=sha256(template).hexdigest(),
        final_digest=sha256(signed).hexdigest(),
    )


def assert_visual_contract(pages: list[bytes]) -> None:
    if not pages or any(not page for page in pages):
        raise ValueError("rendered pages are incomplete")
```

The caller still needs durable idempotency. Bind a client-generated key to the tenant, template revision, normalized field digest, and signing profile. A repeated request with the same key and same digest returns the existing job; the same key with different input is a conflict. Use explicit terminal states such as `signed`, `rejected`, and `expired`, and make every transition emit exactly one logical audit event even when message delivery repeats.

## The endpoint contract should expose failure boundaries

A single synchronous "make PDF" call looks simple until a font is missing, a field value overflows, consent is withdrawn, or a callback arrives twice. The endpoint surface should expose a job submission operation, a status lookup, controlled artifact retrieval, and deletion or expiry semantics. Internally, keep fill, render, flatten, sign, and validate independently observable. Externally, don't leak processor-specific stages that clients would have to orchestrate forever.

| Boundary | Contract to require | Failure policy |
| --- | --- | --- |
| Submission | Idempotency key, template version, validated field schema, policy identifier | Reject conflicting reuse before processing |
| Rendering | Page count, dimensions, font set, overflow report, fixture tolerance | Stop before signing and send the job to review |
| Signing | Named profile, credential reference, timestamp policy, final digest | Publish only after independent validation |
| Retrieval | Short-lived authorization, tenant binding, content digest | Deny cross-tenant or expired access |
| Expiry | Policy-derived timestamp and deletion evidence | Remove raw and derived objects across replicas |

Be precise about errors. Schema mistakes and unsupported PDFs are permanent failures; transient capacity limits can be retried with bounded exponential backoff and jitter. A `409`-style idempotency conflict needs caller correction. A `429`-style rate limit needs delay. Neither should be hidden beneath a generic "processing failed" status, because operators will otherwise retry the wrong class and create a storm.

Logs are data.

Observability must stay useful without becoming a second document store. Track queue age, processing duration by stage, rendered page count, bytes by lifecycle class, validation outcomes, retry count, and deletion lag. Use bounded labels. A document identifier, tenant identifier, or government-ID fragment does not belong in a metric label, and dumping field payloads into traces is indefensible.

This is also where operational complexity becomes measurable. Separate services add network hops, credentials, version negotiation, and failure combinations; one bundled processor reduces those integration points but increases dependency concentration. The sensible choice depends on whether the team can operate font packages, signing keys, PDF parsers, security patches, and regional execution with the required response time. I'm not sure a generic scorecard can settle that choice: a threat model, two representative templates, and a restore drill will reveal more than a feature matrix.

## Retain proof, not every intermediate byte

Delete deliberately.

Start with data classes, not one retention period. The uploaded identity document, filled unsigned form, rendered previews, flattened unsigned PDF, final signed PDF, audit events, and operational logs serve different purposes. Give each an owner, lawful or contractual purpose, region, encryption rule, access role, deletion trigger, and backup treatment. Then test deletion.

The catch is that shorter raw-document retention reduces the material available for visual reinspection during a dispute. Longer retention can make investigation easier, but it increases exposure, access-review scope, backup cleanup, and the consequences of a credential mistake. Keep the final signed form only when the relying party needs the form itself; otherwise retain the minimum audit evidence and digests that prove processing state without reproducing identity data. A litigation hold or regulated record duty must override routine deletion through an explicit, audited policy path, not an engineer's manual copy.

Seven days versus 30 days is only a planning comparison. The production value should come from the verification purpose, appeal window, fraud model, contracts, and applicable law. NIST SP 800-63A provides identity-proofing guidance, but the service still has to map its evidence handling to its own assurance level and privacy assessment. GDPR Article 32 also requires security measures appropriate to risk; encryption is important, yet it doesn't turn unnecessary collection into necessary collection.

Stop keeping render previews as soon as the signed artifact passes validation. Stop keeping unsigned filled and flattened intermediates after the job's recovery window closes. Stop putting document bodies in application logs at all. What you give up is the ability to reconstruct every visual intermediate after deletion, so preserve processor version, template version, digest chain, validation report, and policy decision in the audit record. This compromise is honest: less sensitive material remains available, and a later investigation may explain the chain without being able to replay every pixel.

No architecture removes the need for access drills and erasure drills. Test that a support role cannot fetch another tenant's artifact, that an expired URL stays expired, that object deletion reaches replicas and documented backup expiry, and that an audit export omits forbidden payload fields. Run the signature validator against fixtures after every processor or font change. Small tests catch expensive evidence failures.

## A decision rule that survives procurement

Select an implementation only after it passes the same acceptance pack: representative AcroForm documents, unusual fonts, long names, non-ASCII addresses, rotated pages, empty optional fields, and the required signature profile. Measure p50 and p95 stage latency with your files and deployment region; published numbers from someone else's corpus don't answer this workload. Verify data location, subprocessors, deletion behavior, key custody, audit export, incident obligations, and exit mechanics in the contract as well as the API.

A managed processor is not suitable when policy requires signing keys and all plaintext processing to remain inside infrastructure the team exclusively controls; use a self-operated component and budget for parser patching, font management, sandboxing, and signing operations. A self-operated stack is a poor fit when the team cannot staff those duties or demonstrate deletion and key-handling controls; use a managed boundary whose contractual and technical evidence passes review. Split processing and signing when key custody requires it, accepting another authenticated hop and another retry boundary.

The final decision is therefore conditional, not a vendor ranking: prefer the smallest number of capability boundaries that still satisfies key custody, regional processing, signature validation, visual fidelity, deletion verification, and recovery objectives. Keep the adapter replaceable, store portable hashes and event records, and test export before signing a long contract.

## References

- https://developer.mozilla.org/en-US/docs/Web/API/Blob
- https://eur-lex.europa.eu/eli/reg/2016/679/oj
- https://pages.nist.gov/800-63-4/sp800-63a.html
- https://eur-lex.europa.eu/eli/reg/2014/910/oj
- https://www.etsi.org/deliver/etsi_en/319100_319199/31914201/
- https://www.iso.org/standard/75839.html
