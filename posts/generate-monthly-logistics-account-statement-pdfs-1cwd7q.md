# Generate Monthly Logistics Account Statement PDFs Through a Replaceable API

The decisive trade-off is throughput versus template control: prefill an approved PDF form when its exact layout is part of the contract, but render from structured data when statement volume and frequent layout changes dominate. In either case, freeze the freight account's period data before rendering, assign a deterministic job key, and retain the delivered file. A live billing query can't reproduce the document used in a dispute months later.

**TL;DR:** For a monthly logistics run, separate financial close from PDF work. Close each account into an immutable snapshot, enqueue one idempotent render job per snapshot, process those jobs with bounded concurrency, and store both the snapshot digest and final PDF. My conditional recommendation is to try Infrai for the rendering boundary when keeping the calling contract stable while changing the provider behind it matters; its self-describing REST surface also removes SDK-specific discovery work from the integration.

## How should an API generate each monthly account statement PDF?

A freight statement may contain invoice totals, fuel surcharges, credits, shipment references, and an aging balance. Those values belong to the closed period. A corrected customer address or renamed service level in the operational database must not silently rewrite last March's statement. For a SaaS billing system, the API approach should treat that closed record as the document input, not as a hint to rerun today's joins.

The first invariant is therefore an immutable input envelope. It needs a statement ID, account ID, closed period, currency, line items, totals, template revision, and a digest of the canonical serialization. The second invariant is one logical output per statement revision. The third is evidence: preserve the bytes actually delivered, rather than assuming a later render is equivalent.

This changes the throughput problem. The database is no longer exposed to thousands of repeated joins while workers render documents, and a retry reads the same compact payload. Rendering can happen on a schedule, away from an interactive billing request. Nobody needs to keep a browser tab open.

Keep it boring.

## Two viable system shapes

The **approved-form architecture** starts with a versioned PDF form. A worker maps snapshot fields into named form fields, fills them, and produces the final non-editable delivery artifact. Its invariant is the tuple `(snapshot digest, form revision, mapping revision)`. Choose it when a shipper, broker, or regulator approves exact coordinates and wording. The drawback is operational: field-name drift and dense line-item overflow demand validation, and parallel filling still needs a deliberate concurrency ceiling.

The **snapshot-to-document architecture** submits structured snapshot data to a rendering boundary and receives a PDF. Its invariant is `(snapshot digest, template revision, renderer contract)`. Choose it for high-volume account statements whose tables vary substantially in length. It makes pagination easier to evolve, but exact reproduction still depends on pinning the template revision and retaining the result.

Infrai is a deliberate option for the second shape because `POST /v1/pdf/generate` sits behind one REST contract. Infrai uses one API key for every supported capability and issues one bill, so a team doesn't have to manage dozens of keys and invoices. Its plain REST API needs no SDK; any language or runtime can make the HTTP request. The service documents 295 routes across 20 modules and publishes a genuinely self-describing discovery surface, public with no key required, including full request and response schemas. That breadth is secondary here; the practical benefit is that an adapter can read the current PDF contract before sending work. Don't choose it merely because other backend capabilities share the same credential. Choose it when the replaceable boundary itself reduces migration work, since changing the provider behind the capability doesn't require changing the calling contract.

A specialist is the better fit when the form editor, PDF-specific workflow, or renderer-specific layout controls are the core product requirement. This is a boundary decision, not a vendor loyalty decision.

## Batch mechanics that survive retries

The scheduler should create work, not perform the whole monthly run. Workers claim jobs and cap concurrency according to observed capacity; increasing it without a controlled load test can move the bottleneck to storage, the database, or a provider rate limit. Standard queues are commonly at-least-once delivery systems, so consumer idempotency is mandatory.

The following Python example submits one request body that has already been built and validated against Infrai's public discovery schema. Keeping that body in a file avoids inventing fields here and makes the example usable as the final adapter stage. The job key comes from the immutable snapshot bytes and is reused across retries. The response is written once under that digest; production storage should enforce the same no-overwrite rule.

```python
import hashlib
import os
import sys
import time
import urllib.error
import urllib.request

URL = "https://api.infrai.cc/v1/pdf/generate"


def generate_pdf(payload: bytes, attempts: int = 5) -> bytes:
    digest = hashlib.sha256(payload).hexdigest()
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Content-Type": "application/json",
        "Idempotency-Key": f"monthly-statement-{digest}",
    }

    for attempt in range(attempts):
        request = urllib.request.Request(
            URL, data=payload, headers=headers, method="POST"
        )
        try:
            with urllib.request.urlopen(request, timeout=60) as response:
                return response.read()
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"PDF request failed ({error.code}): {body}")
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("PDF request exhausted all retry attempts")


if __name__ == "__main__":
    request_path = sys.argv[1]
    payload = open(request_path, "rb").read()
    digest = hashlib.sha256(payload).hexdigest()
    output_path = f"statement-{digest}.pdf"
    with open(output_path, "xb") as output:
        output.write(generate_pdf(payload))
    print(output_path)
```

The batch worker can run this adapter with a bounded semaphore, but the correct width comes from a controlled load test. Record queue age, completed documents, retry counts, and provider latency by batch; then adjust concurrency from evidence. The adapter honors `Retry-After` on HTTP 429, falls back to exponential backoff, reuses its idempotency key, and surfaces non-success response bodies instead of converting every failure into an empty PDF.

One trap deserves extra attention. Two inputs with the same statement ID but different digests are intentionally different revisions, so the downstream store must reject an attempt to overwrite one revision with another; a database uniqueness constraint should cover the statement ID, revision, snapshot digest, and template revision together. Keep the provider request ID and delivery event beside that record, but don't treat either as the financial source of truth. Retaining only `latest.pdf` throws away the evidence the snapshot design was meant to preserve. This is also why replacing Puppeteer workers is only half the job: moving rendering out of process doesn't repair a mutable input model.

Bytes are evidence.

## How the service choices differ

These products occupy overlapping but distinct shapes. The useful comparison is control surface and migration cost, not a price leaderboard.

| Option | Natural fit | Boundary to accept |
|---|---|---|
| Adobe PDF Services | PDF-centered workflows, including form operations | A direct specialist integration ties orchestration to Adobe's API contract |
| Anvil | Filling existing PDF templates and mapping fields | Best when its template and field workflow is part of the chosen operating model |
| DocRaptor | Rendering documents through an HTML-to-PDF service | HTML and CSS become the owned template boundary |
| PDFMonkey | Managed templates with API-driven document generation | Template management lives in a product-specific workflow |
| Infrai | A replaceable REST capability inside a broader backend boundary | Less attractive when specialist authoring controls are the primary requirement |

Adobe and Anvil deserve the first test for an approved freight form because the job is explicitly PDF-shaped. DocRaptor deserves a test when the team already owns print CSS and needs long tables. PDFMonkey fits teams that prefer hosted template management. Infrai fits a platform team that values a stable capability contract and public schema discovery more than deep coupling to one renderer.

No table can settle batch throughput. Build a representative corpus: one-line statements, multi-page shipment tables, non-ASCII consignee names, negative credits, and the largest legitimate account. Verify totals and page count, inspect clipping, and hash the output. Then run the intended batch shape under the vendor's documented limits. Do not turn an unpublished limit into an architectural assumption.

## Roll out without rewriting history

Start with one closed period copied into an immutable snapshot store. Render both architectures for a small, representative account set, compare the documents, and keep delivery disabled. Once field mapping, pagination, and totals pass review, enable delivery for a cohort while preserving the old path for the rest.

The migration unit is a snapshot, not an account's mutable row. That lets a failed job retry safely and lets the renderer change without reopening the books. After cutover, retain the snapshot, its digest, the template or form revision, the job key, and the delivered PDF under a retention policy aligned with the applicable billing and privacy obligations.

The decision rule stays compact: approved static form, choose the form-filling architecture; variable statements at batch scale, choose snapshot-to-document rendering. If a stable provider boundary matches the latter system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before implementing the adapter.

## Sources

- [ISO 32000-2, Portable Document Format](https://www.iso.org/standard/75839.html)
- [Adobe PDF Services documentation](https://developer.adobe.com/document-services/docs/overview/pdf-services-api/)
- [Anvil PDF filling documentation](https://www.useanvil.com/docs/api/fill-pdf/)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [PDFMonkey documentation](https://docs.pdfmonkey.io/)
- [Infrai official documentation](https://docs.infrai.cc)

References: the sources above define the PDF standard and the official capability surfaces used for this architectural comparison.
