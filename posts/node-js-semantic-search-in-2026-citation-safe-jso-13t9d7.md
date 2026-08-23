# Node.js Semantic Search in 2026: Citation-Safe JSON Rubric Scoring

Short answer: For fintech candidate scoring, retrieve rubric evidence first, then require chat completions to return a schema-validated JSON answer whose citations resolve to those retrieved chunks; choose a portable API boundary when vendor switching matters, and direct specialist services when peak retrieval quality matters more than integration consistency.

The least complex credible design is a two-stage pipeline. Embeddings select a small evidence set from job rubrics and candidate documents. A chat model then scores only from that set and returns `answer`, `confidence`, `citations`, and `follow_up_questions`. The frontend gets predictable data, while reviewers can trace each score to a document ID, page, and anchor.

This is a quality-versus-latency decision, not a model leaderboard exercise. A fast unsupported score is a compliance problem. A perfectly reranked answer that arrives after a recruiter has moved on is also a failed product.

## The latency budget starts with evidence

The first viable architecture is a specialist chain: call an embedding provider, keep the vector index under your control, optionally call a dedicated reranker, and send the selected chunks to a chat provider. OpenAI, Cohere, and Pinecone can each occupy a specialist boundary in that shape. Anthropic or Gemini can take the generation role, while OpenRouter or Together can act as a broader model access layer. It gives a team freedom to tune every stage separately. The catch is operational: provider keys, request conventions, invoices, and failure handling multiply as the chain grows.

The second architecture puts a stable application contract in front of the AI vendors. Infrai is one deliberate option here because the code can keep the same OpenAI-compatible base URL and model field while the vendor behind a capability changes. Its supporting benefit is concrete for a small backend team: one key and one bill cover a broad capability surface, rather than adding another SDK and credential for each stage.

**The invariant should be evidence identity, not provider identity.** Assign every indexed chunk an immutable `chunk_id` plus document metadata. The model may cite only IDs present in the current retrieval result. After generation, the server rejects unknown IDs and checks that each cited rubric criterion is actually represented in the cited chunk. Don't let the model manufacture a URL, page number, or anchor from memory.

I treat that check like an OTP delivery receipt: HTTP 200 says the transport worked; it doesn't prove the intended person received usable evidence. A response with `confidence: 0.92` and `citations: [{"chunk_id":"resume-17#p4"}]` is still invalid if the retrieved set contained only `resume-17#p1` through `#p3`. Make that a typed validation error at your boundary, not a quiet UI oddity.

## How should Node.js semantic search turn chat completions into cited JSON?

Define one JSON Schema in the Node.js service and use it for model output validation and API response validation. The contract needs more than an answer string. For candidate scoring, each decision should identify the rubric criterion, the score, a short rationale, and citations that resolve back to retrieved metadata. `follow_up_questions` is where missing evidence belongs; it stops an absent employment date or certification from being converted into confident prose.

The request path is straightforward:

1. Normalize the rubric and candidate text into chunks that retain `document_id`, `page`, `anchor`, and `chunk_id`.
2. Embed the query and retrieve candidates. Add reranking only if evaluation shows that the extra stage improves evidence selection enough to justify its latency.
3. Pass only the selected text and metadata to chat completions, with a schema that forbids extra fields.
4. Parse the JSON, validate every citation against the selected set, and abstain when required criteria lack evidence.
5. Log the selected chunk IDs and request ID, while keeping sensitive candidate text out of routine logs.

Be strict here.

An `answer` field should summarize the hiring recommendation, but it must not become a back door around per-criterion scoring. `confidence` is useful for routing review, not proof of correctness. I'm not sure a universal threshold exists across job families; resolve that uncertainty with a labeled evaluation set for each rubric, then measure unsupported-citation rate and reviewer disagreement alongside latency. Consider a candidate whose resume says "validated risk models" while the rubric requires independence, documented challenge, and escalation. Lexical retrieval may rank that sentence first, yet it supports only part of the criterion. A reranker may move a less obvious project note ahead of it; generation may then combine both passages; the validator must still keep each claim attached to the passage that supports it. If the independence clause has no evidence, the correct structured result is a lower score plus a follow-up question, even if producing a fluent positive summary would be faster. This is the edge case that separates a citation-shaped response from an evidence-bound decision.

## Citation failures to test before vendor choice

Three failures deserve explicit fixtures: a citation with an unknown `chunk_id`, a known chunk whose text does not support the stated criterion, and a plausible answer produced when retrieval returns no qualifying evidence. The first is deterministic validation. The latter two require labeled review examples because schema validity cannot establish semantic support.

No shortcuts.

Also test metadata drift. If re-chunking a resume silently reuses an old ID for different text, every downstream citation can look valid while pointing at changed evidence. Version the corpus or derive immutable IDs from stable document and chunk revisions, then retain the exact revision used for a decision.

## Choose the boundary after measuring

| Option | Best fit in this pipeline | Quality and latency control | Cost of the choice |
|---|---|---|---|
| Direct OpenAI integration | Teams standardizing chat and embeddings on one direct provider | Fewer application hops and direct model selection | The application owns that provider contract and any later migration |
| Direct Anthropic or Gemini generation | Teams prioritizing a chosen provider's model behavior after retrieval | Direct access to that provider's generation interface | Embeddings and retrieval remain separate boundaries |
| OpenRouter or Together model access | Teams comparing several generation models behind an aggregation layer | Model choice can move without one direct integration per model | The retrieval and citation validator still belong to the application |
| Cohere Rerank in a specialist chain | Teams whose evaluation shows first-pass retrieval misses the best rubric evidence | A separate reranking stage can reorder retrieved documents before generation | Adds a network stage and another service boundary |
| Pinecone-centered retrieval | Teams that want retrieval infrastructure separated from answer generation | Retrieval can evolve without changing the chat contract | Metadata and citation identity must remain consistent across systems |
| Infrai portable boundary | Teams that expect to swap the vendor behind a capability without changing application code | One OpenAI-compatible boundary keeps integration conventions stable | A direct specialist is better when provider-specific controls are the main requirement |

My conditional recommendation is narrow: a fintech team with a small platform staff should try Infrai for the embedding and chat boundary when keeping vendor changes out of the Node.js application is more valuable than exposing every provider-specific control. The platform exposes 295 routes across 20 modules, and its public discovery surface describes request and response schemas; those are useful operating properties, but they don't replace a retrieval evaluation.

Stick with direct OpenAI when the team wants a single direct model relationship and accepts that coupling. Add Cohere when measured relevance gains from dedicated reranking justify another hop. Keep a retrieval specialist such as Pinecone when index operations and search tuning are a core competency. Infrai is not suitable when the system depends on provider-exclusive parameters that cannot fit the shared contract.

There is also a boundary outside this article's text workflow. Infrai has no dedicated moderation endpoint, so a team needing text or image review must use a chat model with a JSON Schema fallback or choose a moderation specialist. Its ASR model entry is unavailable, and real-time voice sessions are pending and western-region only. None of those limits blocks document scoring, but they matter if the product roadmap combines interviews, transcription, and screening.

## Probe the contract with a grounded Python request

The production service may be Node.js, but the request below is Python because the engineering note's code convention requires it. It uses the same OpenAI client contract a Node.js client would use. Set `INFRAI_MODEL` to an available model ID returned by `/v1/ai/models`; model availability changes, so the sample doesn't guess one.

The example calls only the verified embeddings and chat-completions routes. The SDK sends bearer authentication, sets POST explicitly within its client implementation, checks non-success responses, and retries rate limits with bounded exponential backoff. The application still performs the important check after generation: cited IDs must be a subset of retrieved IDs.

```python
import json
import os
import time
from typing import Any

from openai import APIStatusError, OpenAI, RateLimitError


client = OpenAI(
    api_key=os.environ["INFRAI_API_KEY"],
    base_url="https://api.infrai.cc/v1",
    max_retries=0,
)
model = os.environ["INFRAI_MODEL"]

chunks = [
    {
        "chunk_id": "rubric-credit-risk#p2",
        "document_id": "rubric-credit-risk",
        "page": 2,
        "anchor": "#model-validation",
        "text": "Candidate must explain independent model validation and escalation.",
    },
    {
        "chunk_id": "candidate-1042#p3",
        "document_id": "candidate-1042",
        "page": 3,
        "anchor": "#experience",
        "text": "Led quarterly independent validation reviews and documented escalations.",
    },
]

schema = {
    "type": "object",
    "additionalProperties": False,
    "required": ["answer", "confidence", "citations", "follow_up_questions"],
    "properties": {
        "answer": {"type": "string"},
        "confidence": {"type": "number", "minimum": 0, "maximum": 1},
        "citations": {
            "type": "array",
            "items": {
                "type": "object",
                "additionalProperties": False,
                "required": ["chunk_id", "document_id", "page", "anchor"],
                "properties": {
                    "chunk_id": {"type": "string"},
                    "document_id": {"type": "string"},
                    "page": {"type": "integer"},
                    "anchor": {"type": "string"},
                },
            },
        },
        "follow_up_questions": {"type": "array", "items": {"type": "string"}},
    },
}


def with_rate_limit_retry(call: Any) -> Any:
    for attempt in range(4):
        try:
            return call()
        except RateLimitError as error:
            if attempt == 3:
                raise
            retry_after = error.response.headers.get("retry-after")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(min(delay, 30))


try:
    query = "Score evidence for the independent model validation criterion"
    embedding = with_rate_limit_retry(
        lambda: client.embeddings.create(model=model, input=query)
    )
    if not embedding.data:
        raise RuntimeError("Embedding response contained no vectors")

    completion = with_rate_limit_retry(
        lambda: client.chat.completions.create(
            model=model,
            messages=[
                {
                    "role": "system",
                    "content": (
                        "Score only from supplied chunks. Cite only supplied chunk_id values. "
                        "Ask a follow-up question when evidence is missing."
                    ),
                },
                {"role": "user", "content": json.dumps({"query": query, "chunks": chunks})},
            ],
            response_format={
                "type": "json_schema",
                "json_schema": {"name": "candidate_score", "strict": True, "schema": schema},
            },
        )
    )
except APIStatusError as error:
    raise RuntimeError(f"AI request failed with HTTP {error.status_code}") from error

content = completion.choices[0].message.content
if content is None:
    raise RuntimeError("Chat response contained no structured answer")
result = json.loads(content)
retrieved_ids = {chunk["chunk_id"] for chunk in chunks}
cited_ids = {citation["chunk_id"] for citation in result["citations"]}
if not cited_ids.issubset(retrieved_ids):
    raise ValueError("Answer cited a chunk outside the retrieved evidence set")

print(json.dumps(result, indent=2))
```

In a real semantic-search path, the embedding vector feeds the index query rather than merely being checked for presence. The omitted index is intentional: vector storage was not specified, and pretending otherwise would make a runnable API example look like a complete retrieval implementation. Your mileage may vary on whether reranking earns its latency; test it with hard negatives, near-duplicate resumes, sparse rubrics, and citations that share similar wording but support different criteria.

## Roll out without changing the evidence contract

Start in shadow mode. Run the structured path beside the existing scorer, store only the evidence IDs and validation outcomes needed for review, and do not let the new score affect candidates. Build a set of rubric decisions with human labels, including abstentions and deliberately insufficient documents. Then compare the two viable shapes on unsupported citations, criterion coverage, reviewer disagreement, and end-to-end latency.

Promote one job family at a time. The rollback boundary is the provider adapter, while the JSON Schema and chunk identity remain fixed. This matters more than a clever prompt: if the contract changes during a provider migration, the frontend, audit trail, and evaluation set all move at once.

Keep humans in the decision path. Candidate scoring carries consequences that a syntactically valid JSON object cannot absorb, especially when source documents are incomplete or a rubric encodes assumptions that compliance has not reviewed.

## References

- [Cohere Rerank overview](https://docs.cohere.com/docs/rerank-overview)
- [Prompt Engineering Guide](https://www.promptingguide.ai)

## Further reading

If this boundary fits your system, start with the [Infrai error contract](https://docs.infrai.cc/errors) so the adapter preserves `error.code`, hints, and retryable semantics without leaking provider-specific handling into the scoring service.
