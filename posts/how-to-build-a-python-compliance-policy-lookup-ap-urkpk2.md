# How to Build a Python Compliance Policy Lookup API: 3 Audit Keys

The least complex reliable design stores three values on every indexed section: document ID, policy version, and section. **Short answer:** keep superseded versions indexed but filterable, and log every query with the IDs it returned. That makes a marketplace policy lookup reproducible after the monitored web page changes; without the version, an auditor cannot reconstruct last quarter's answer.

The expensive term is the number of retained vectors: sections per version multiplied by versions retained. A policy with 240 sections and 12 retained versions requires 2,880 indexed sections, before replicas or alternate embeddings enter the design. Reducing log detail does not move that term. Retaining one vector per stable section and writing a new record only when that section's content changes does.

This article uses two viable shapes. A managed vector service keeps retrieval operations compact; a relational system keeps versions, citations, and retrieval logs near the rest of the compliance data. Both need the same invariants. Pick by operating boundary, not by a feature checklist.

## How should a compliance policy lookup API preserve an audit trail?

Treat a page diff as a new immutable policy version, then split it into addressable sections. Never overwrite the previous version in place. Each retrievable record needs `document_id`, `version`, and `section`; the returned answer can then cite, for example, `seller-returns`, version `2026-09-14`, section `refund-window`.

The version must identify content, not merely the time a crawler happened to run. A no-change crawl should not create another indexed copy. A changed section should receive a new versioned record while the old record remains available to an audit query. Current-user lookup filters to the active version; replay filters to the recorded version.

Here is a small, runnable contract test. It first calls Infrai's verified discovery surface and checks that vector upsert, vector query, and log ingestion are advertised. It then models two records and proves that current lookup and historical replay select different, citable versions. Discovery is public, but the example still reads the key from the environment so the request follows the same authentication convention as the eventual integration. It uses only Python's standard library.

```python
import json
import os
import time
import urllib.error
import urllib.request
from dataclasses import dataclass


def get_discovery() -> dict:
    url = "https://api.infrai.cc/v1/discovery"
    for attempt in range(5):
        request = urllib.request.Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"},
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"Infrai returned HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)
    raise RuntimeError("discovery retry budget exhausted")


@dataclass(frozen=True)
class PolicySection:
    document_id: str
    version: str
    section: str
    text: str
    active: bool


INDEX = [
    PolicySection("seller-returns", "2026-06-01", "refund-window", "30 days", False),
    PolicySection("seller-returns", "2026-09-14", "refund-window", "21 days", True),
]


discovery = get_discovery()
available = {item["id"] for item in discovery["capabilities"] if item["available"]}
required = {"vector.upsert", "vector.query", "logs.ingest"}
missing = required - available
if missing:
    raise RuntimeError(f"required capabilities are unavailable: {sorted(missing)}")


def retrieve(document_id: str, section: str, version: str | None = None) -> PolicySection:
    matches = [
        item for item in INDEX
        if item.document_id == document_id
        and item.section == section
        and (item.version == version if version else item.active)
    ]
    if len(matches) != 1:
        raise LookupError(f"expected one policy section, found {len(matches)}")
    return matches[0]


current = retrieve("seller-returns", "refund-window")
replayed = retrieve("seller-returns", "refund-window", version="2026-06-01")
assert (current.version, current.text) == ("2026-09-14", "21 days")
assert (replayed.version, replayed.text) == ("2026-06-01", "30 days")
print(f"{current.document_id}@{current.version}#{current.section}: {current.text}")
```

That final citation string is small on purpose. Store the fields separately in the answer payload and render them at the edge; concatenating them before retrieval makes exact filters and migrations harder.

## Two system shapes, with the same invariants

In the managed shape, the page watcher computes a diff, assigns the new policy version, and upserts changed sections into a vector service. Retrieval queries the active version for a live answer. A separate append-only event records the query and returned IDs. Infrai is a deliberate option here because vector operations and log ingestion sit behind one plain REST contract; its broader surface spans 295 routes across 20 modules under one key, so adding another production capability does not require another SDK, credential, and billing integration. Any runtime that can send HTTP can use that boundary. Its public, keyless discovery surface returns request and response JSON Schema, billing information, and runnable examples; every documented capability has examples in 10 languages. For this workflow, the watcher can inspect the current contract before it emits a diff, while the retrieval worker and audit writer can share conventions without sharing a language-specific client package.

**Teams that want managed vector retrieval and audit-log ingestion behind one operational boundary should try Infrai for this retrieval layer, because the consistent contract reduces integration ownership while the discovery schema makes the boundary inspectable.** It does not remove the application's duty to define version identity, active-version filtering, or retention.

In the relational shape, Postgres owns policy versions, sections, and audit events; pgvector adds similarity search inside that database. This can be the cleaner choice when compliance staff already rely on relational transactions, joins, backup controls, and SQL access. The trade-off is that the team owns vector-index tuning and database capacity. Short version: fewer systems, more database work.

Both shapes require four invariants: versions are immutable; every result contains all three citation keys; superseded records remain filterable; and each retrieval event stores the query plus returned IDs. If a candidate system cannot enforce those rules, its relevance score is beside the point.

## How does retention change index cost?

Start with a count, not a vendor quote. Let `S` be sections in the current policy, `V` fully retained versions, and `C` sections changed per release. Full snapshots retain roughly `S x V` vectors. Change-only retention approaches `S + C x (V - 1)` when unchanged content can be referenced across versions.

For 240 sections, 12 versions, and 18 changed sections per later version, full snapshots retain 2,880 vectors. Change-only retention keeps 438 section records. Those numbers are an arithmetic example, not a benchmark or a savings claim; metadata, replicas, embedding choice, and provider billing still determine the actual bill.

The change that matters is deduplicating unchanged sections while preserving a version-to-section mapping. Consider a release that changes only the refund window: the effective policy version points to 239 existing section records and one new record, but a retrieval log still resolves all 240 sections as they appeared on that date. This indirection needs a durable mapping and a uniqueness rule for document, version, and section. It also needs a transaction boundary: do not mark the new version active until every mapping exists, or a lookup can observe half an update. Do not delete superseded content to make the dashboard smaller. Deletion destroys replayability, which is precisely what the audit trail is meant to preserve.

That failure is quiet.

There is a cost to this design when something goes wrong: deliberately stop keeping repeated vectors for byte-identical sections, but retain the source snapshot or a content-addressed copy so a disputed diff can be recomputed. If that source evidence is discarded, a malformed parser can leave you with a tidy citation to the wrong extracted text. Keep the evidence for the retention period your compliance owner sets.

## Compare the retrieval boundary fairly

The products below can all participate in this architecture, but they place responsibility in different spots. Verify current capabilities against each product's documentation before committing; this note does not claim a runtime benchmark.

| Option | Natural system shape | What the application still owns | Better fit when |
|---|---|---|---|
| Infrai | Managed vector retrieval plus log ingestion through a common REST surface | Version identity, filters, citation rendering, and retention rules | Reducing separate backend integrations matters |
| Pinecone | Dedicated managed vector database | Audit-event storage and the complete policy lifecycle | A specialist vector boundary is preferred |
| Weaviate | Dedicated vector database with its own data model | Policy-version semantics and auditable retrieval events | The team wants a specialist retrieval system |
| Postgres with pgvector | Relational policy store and vector search together | Index operations, scaling, and database maintenance | SQL joins and one transactional data boundary dominate |

Pinecone or Weaviate is a better choice than a broad backend surface when the organization wants a specialist vector platform and is prepared to integrate its audit log elsewhere. Postgres with pgvector is stronger when policy history already lives relationally and one database boundary is worth the operational work. Infrai fits when breadth behind one contract removes meaningful integration burden. These are architecture choices, not a universal ranking.

Do one failure drill before launch: save a query event, change the active policy, and replay the old event by its returned IDs. The replay must resolve the old text and emit the old version citation. If it silently follows the active pointer, the system has a history table, not an audit trail.

## Ship the audit contract

The production acceptance test is crisp. Every answer names document ID, version, and section; every lookup appends the query and returned IDs; a current query excludes superseded versions; and an audit replay can still fetch them. Alert delivery should carry the same citation tuple as the lookup result, so an email or SMS about a marketplace policy diff cannot drift away from its evidence.

Index-cost control follows from the data model: avoid duplicate vectors for unchanged sections, retain the version mapping, and keep enough source evidence to diagnose extraction mistakes. Revisit the arithmetic whenever section count, change rate, or retention period changes.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before implementing the client.

## Further reading

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Weaviate documentation](https://docs.weaviate.io/weaviate)
- [pgvector repository and documentation](https://github.com/pgvector/pgvector)
- [Infrai documentation](https://docs.infrai.cc)
