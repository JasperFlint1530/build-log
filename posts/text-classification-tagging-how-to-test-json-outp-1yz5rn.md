# Text Classification Tagging: How to Test JSON Output Accuracy and API Cost

For OpenAI, Claude, or Gemini text classification, keep the validated rubric result, the source-text hash, and a short decision trace; do not keep every prompt and raw model response forever. That is the least complex way to make logistics-candidate scoring reviewable without letting storage and repeated inference dominate the bill. **The first selection criterion is schema-valid output under your own workload, not a provider name.**

TL;DR: measure OpenAI, Claude, and Gemini as interchangeable candidates behind one adapter. Reject malformed or semantically impossible scores before they reach hiring workflows. Cache only approved results keyed by rubric version and normalized-input hash, retain sampled redacted failures long enough to diagnose drift, and delete the bulk of raw payloads on a declared schedule.

## What is the bill actually made of?

For this backend, the useful cost equation is broader than an API invoice:

`total = inference + retries + evaluation runs + retained bytes + review time`

Start by measuring each term per 1,000 candidate documents. Do not invent a percentage split before collecting it. A cheap call that produces an unparseable tag set can cost more after retries and human review than a call with a higher nominal price. Conversely, perfect retention of every request, response, and resume creates an expanding privacy and compliance burden even when object storage looks inexpensive.

The dominant term depends on traffic and failure behavior. Record input and output units when the selected interface reports them, retry count, validation outcome, latency, retained byte count, and manual-review minutes. Keep monetary rates in deployment configuration rather than article code because rates change. This produces a comparable ledger for all candidates without turning procurement into a feature checklist.

Measure first.

The change that usually moves the avoidable term is deterministic reuse. Normalize the candidate text, hash it with the rubric version, and reuse a previously validated result. Never reuse across rubric revisions: changing `forklift_experience` from optional evidence to a required score changes the decision even when the resume does not.

Retention is part of that calculation. Keep the final validated JSON and its decision trace for the period required by the hiring policy. Keep a small, redacted failure sample for diagnosis. Deliberately stop keeping bulk raw prompts and rejected responses after the declared window. When something goes wrong later, that choice means some failures can be reconstructed only from hashes, aggregate metrics, and the sample, not replayed byte for byte. That lost forensic detail is the real price of deletion.

## How do you make rubric output safe to consume?

Treat generated JSON as hostile input. Parsing is only gate one. The application must also reject unknown tags, missing evidence, out-of-range scores, duplicate identifiers, and totals that disagree with component scores. A candidate should go to review when evidence is absent; the backend must not convert uncertainty into a zero or a pass.

Use one internal contract for every external adapter. The following Python program runs without third-party packages and demonstrates the boundary. In production, `call_model` is the only function that should know which service received the request.

```python
import hashlib
import json
from dataclasses import dataclass
from typing import Callable


RUBRIC_VERSION = "logistics-ops-v3"
ALLOWED_CRITERIA = {"route_planning", "warehouse_safety", "shift_handover"}


@dataclass(frozen=True)
class Score:
    criterion: str
    points: int
    evidence: str


def cache_key(candidate_text: str) -> str:
    normalized = " ".join(candidate_text.split()).casefold()
    material = f"{RUBRIC_VERSION}\n{normalized}".encode("utf-8")
    return hashlib.sha256(material).hexdigest()


def validate(raw: str) -> dict:
    payload = json.loads(raw)
    if set(payload) != {"candidate_id", "scores", "total", "review_required"}:
        raise ValueError("unexpected top-level fields")
    if not isinstance(payload["candidate_id"], str) or not payload["candidate_id"]:
        raise ValueError("candidate_id must be a non-empty string")
    if not isinstance(payload["review_required"], bool):
        raise ValueError("review_required must be boolean")
    if not isinstance(payload["scores"], list):
        raise ValueError("scores must be a list")

    scores: list[Score] = []
    for item in payload["scores"]:
        if set(item) != {"criterion", "points", "evidence"}:
            raise ValueError("unexpected score fields")
        score = Score(**item)
        if score.criterion not in ALLOWED_CRITERIA:
            raise ValueError(f"unknown criterion: {score.criterion}")
        if type(score.points) is not int or not 0 <= score.points <= 4:
            raise ValueError("points must be an integer from 0 through 4")
        if not score.evidence.strip():
            raise ValueError("every score needs quoted or located evidence")
        scores.append(score)

    criteria = [score.criterion for score in scores]
    if len(criteria) != len(set(criteria)) or set(criteria) != ALLOWED_CRITERIA:
        raise ValueError("criteria must appear exactly once")
    expected_total = sum(score.points for score in scores)
    if type(payload["total"]) is not int or payload["total"] != expected_total:
        raise ValueError("total does not equal component scores")
    return payload


def classify(
    candidate_id: str,
    candidate_text: str,
    call_model: Callable[[str], str],
) -> dict:
    request = json.dumps(
        {
            "rubric_version": RUBRIC_VERSION,
            "candidate_id": candidate_id,
            "allowed_criteria": sorted(ALLOWED_CRITERIA),
            "candidate_text": candidate_text,
        },
        separators=(",", ":"),
    )
    return validate(call_model(request))


def local_example(_: str) -> str:
    return json.dumps(
        {
            "candidate_id": "cand-1042",
            "scores": [
                {"criterion": "route_planning", "points": 3,
                 "evidence": "Planned daily routes for 18 drivers."},
                {"criterion": "warehouse_safety", "points": 2,
                 "evidence": "Completed monthly safety inspections."},
                {"criterion": "shift_handover", "points": 4,
                 "evidence": "Maintained a signed handover log."},
            ],
            "total": 9,
            "review_required": False,
        }
    )


result = classify("cand-1042", "Example candidate record", local_example)
print(cache_key("Example candidate record"), result["total"])
```

One subtle Python edge case matters here: `bool` is a subclass of `int`. The explicit `type(...) is int` checks stop `true` from being accepted as one point. Small boundary bugs like this are exactly why a syntactically valid response cannot be trusted as a hiring decision.

Return a typed error from the adapter for transport failure, parse failure, contract failure, and policy failure. Retry only the categories that can plausibly succeed unchanged. A missing evidence field is not a network glitch, so blind retries waste calls and can conceal an unstable instruction.

## How should an app backend compare text classification tagging API JSON output accuracy?

OpenAI, Claude, and Gemini belong in the same test harness, not in three application branches. Their objective difference in this architecture is the result they produce for the fixed corpus through their respective adapters. Do not infer a winner from a clean demonstration or from prose about capabilities.

Build a frozen evaluation set with ordinary resumes, sparse work histories, contradictory dates, multilingual fragments, copied job-description text, prompt-injection attempts, and documents that contain no rubric evidence. Remove direct identifiers before the test when policy permits. Then run the same rubric version, validation code, retry ceiling, and scoring rules against each candidate.

This method has a clear limitation: it cannot tell you which candidate service is universally best, and it is not suitable for choosing from a handful of happy-path examples. It answers a narrower, useful question about the workload in front of you. The trade-off is maintenance: every rubric revision requires a fresh labeled run, reviewer time, and a new cache namespace. In return, the team gets evidence tied to its own three-criterion, 0-through-4 scoring contract instead of borrowing somebody else's benchmark.

| Measure | Count as success | Count as failure |
|---|---|---|
| Contract validity | One complete object passes every validator | Parse, field, type, range, or total error |
| Evidence grounding | Each score points to relevant source text | Invented, missing, or unrelated evidence |
| Abstention | Ambiguous input is sent to review | Unsupported certainty |
| Repeatability | Repeated trials preserve the decision | Material tag or score movement |
| Operational fit | The call meets the region's measured budget | Timeout, excessive retrying, or policy breach |

Use enough repeated trials to expose inconsistent outputs, then report the sample size beside every rate. The task materials provide no verified comparative accuracy results, so there is no honest universal ranking to repeat here. **Choose only after the same acceptance suite passes in the intended US and European deployment paths.**

Accuracy also needs two layers. Contract accuracy is mechanical and can block a response immediately. Rubric accuracy requires labeled examples and reviewer agreement. Keep those numbers separate; combining them hides whether a service misunderstood the resume or merely returned the wrong shape.

They are different failures.

## Operate the decision path across regions

Region is a data-flow property, not a dropdown captured during signup. Before launch, map where candidate text, logs, caches, backups, support artifacts, and evaluation exports travel. Record the approved processing path in configuration, fail closed when it is unavailable, and avoid silently moving a European workload to a US path to rescue latency.

Observability should exclude resume bodies. Emit a request identifier, provider-neutral adapter name, rubric version, hashed input key, validation category, retry count, timing, and retained-byte count. Put direct identifiers in the hiring system of record, then join operational events through a restricted correlation key. This mirrors a lesson from OTP systems: delivery telemetry is useful; message content in general-purpose logs is a liability.

Deploy rubric changes as versioned releases. Run the old and new versions on a redacted evaluation set, review decision changes, and warm a separate cache namespace. Roll back the rubric and validator together. Never reinterpret an old JSON object under a new rubric merely because its fields still parse.

For incident handling, alert on validation-failure rate, review-queue age, retry amplification, and decision distribution shifts. A single malformed response is contained by the gate. A sustained rise is an adapter or evaluation incident, not an invitation to loosen validation.

## Keep the final choice reversible

The durable design decision is the contract around the model call. Store provider-neutral results, preserve rubric versions, and keep adapter-specific metadata out of downstream hiring tables. Reversibility gives the team room to respond when evaluation quality, regional constraints, latency, or commercial terms change.

Do not automate rejection from a tag alone. Structured output correctness makes data consumable; it does not prove that a rubric is fair, that evidence was interpreted correctly, or that the decision deserves no human review. The backend's job is narrower: enforce the contract, surface uncertainty, retain enough evidence for accountable review, and delete sensitive bulk data when its declared purpose expires.

## Further reading

- [OpenAI Embeddings guide](https://platform.openai.com/docs/guides/embeddings)
- [openai/whisper: open-source speech recognition](https://github.com/openai/whisper)
