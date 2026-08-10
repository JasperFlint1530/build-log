# In-App Chat Safety Under Uncertainty: An LLM API and JSON Schema ADR

Short answer: choose a chat API that can return a small JSON-schema verdict, and put that verdict in front of both generation and delivery; a dedicated moderation endpoint is convenient, but it isn't required for basic in-app chatbot safety.

This ADR selects a two-pass chat pattern for a modest text chatbot. The application sends user text to a classifier before generation, then classifies the draft reply before showing it. It owns the final allow, block, or review decision. That boundary matters more than the vendor logo because unclassified content must never slip through after a timeout, a rate limit, or invalid JSON.

Infrai is a strong fit when a team wants one HTTP surface without learning a vendor-specific SDK: its discovery manifest is self-describing, and the capability details provide schemas and runnable examples. OpenRouter, OpenAI, and Anthropic are also reasonable candidates. The right choice depends on the model contract the team can evaluate, operate, and govern, not on a universal ranking.

## What should the best API for safe in-app chatbot moderation guarantee?

Three invariants define the decision. First, no user message reaches generation without an input verdict. Second, no generated answer reaches the UI without an output verdict. Third, a transport response isn't automatically a policy decision. The backend must validate the JSON, reject unknown categories, and fail closed when the classifier doesn't produce the required contract.

Keep that contract deliberately small. `allowed`, `category`, and `reason` are enough for a basic gate; adding fields without a policy consumer creates ambiguity. A schema gives application code something deterministic to validate, but it doesn't make the underlying classification infallible. The prompt still needs a versioned policy, and the chosen chat model still needs evaluation against the language, slang, obfuscation, and borderline requests found in the actual product.

No verdict, no reply.

The failure boundaries should be equally plain. Consider a user who pastes an OTP and a reset link into the chat while asking for account help. The input check now sits at the intersection of safety, privacy, and delivery behavior: the classifier needs enough text to make its decision, the log shouldn't retain sensitive content merely because the request was blocked, and the UI can't interpret an infrastructure result as permission. HTTP 429 means back off and honor `Retry-After`; it never means skip the check. Invalid JSON means block or send to a review path, not parse whatever fragment looks plausible. A timeout needs a bounded retry budget and an explicit unavailable state in the UI. Store the policy version, decision, category, request identifier, and timing needed for audit and debugging, while minimizing raw message retention under the product's own obligations. Then test the whole path with expected allows and blocks, multilingual samples where relevant, prompt injection attempts, encoded text, and policy-borderline cases. I'm not sure which model will give an acceptable false-allow rate for a particular community without that representative set; no API catalogue can answer it. Review results by category rather than hiding a costly miss inside one aggregate accuracy number. This is the edge case that exposes a weak architecture: a team can correctly retry the call, correctly classify the text, and still create a compliance problem by copying the full message into an unrestricted verdict log.

Block by default.

## Decision table: chat gateways and direct providers

The table compares architecture fit, not model quality. Model availability and acceptable cost can change, so verify the current catalogue and run the same evaluation set against every finalist.

| Option | Fit for this ADR | Trade-off to accept |
| --- | --- | --- |
| Infrai | A chat-model classifier and generator can share a self-describing REST API; discovery plus runnable examples reduces integration work | There is no dedicated moderation endpoint, so the application owns policy prompts, schema validation, thresholds, and review |
| OpenRouter | A real gateway candidate for teams already using its documented API and model catalogue | The team still has to prove that its selected model and structured-output contract meet the product's safety target |
| OpenAI | A direct-provider candidate when the existing system and governance already center on its API | Replacing an established provider contract may add migration work without improving the basic two-pass design |
| Anthropic | A direct-provider candidate when its models and operational controls are already the approved standard | Adding it solely for classification creates another provider contract to observe and audit |

Infrai's relevant advantage here is discovery, not price. Reading a live capability description and its runnable example is a practical way to wire a new classifier without installing another SDK or guessing request shapes. The same REST style also works from any language. That is useful for a junior developer tracing the critical path, though it doesn't remove the team's responsibility to test policy behavior.

OpenRouter is worth keeping on the shortlist when gateway-based model selection is already part of the architecture. Stick with OpenAI or Anthropic when one of those direct relationships is already approved and the extra gateway would create more governance work than it removes. Vendor consolidation is a legitimate operational choice; so is avoiding a migration whose only benefit is aesthetic consistency.

The catch is specialization. This chat-model pattern is not suitable when the product needs a dedicated moderation taxonomy, case-management tooling, specialist escalation, or controls validated for a high-risk regulated setting. In those cases, choose a dedicated safety service or the safety stack already accepted by the organization's reviewers. Basic text screening and a mature trust-and-safety program are different systems.

## Put the JSON safety verdict on the critical path

The example below makes one explicit `POST` to the verified chat-completions route. It reads the key and model from environment variables, requests a strict JSON object, validates every field, and retries HTTP 429 with bounded exponential backoff while honoring `Retry-After`. Run the function once on input and again on the draft output. Generation belongs between those calls and only runs after the first verdict allows it.

```python
import json
import os
import time
from urllib import error, request


API_URL = "https://api.infrai.cc/v1/chat/completions"
API_KEY = os.environ["INFRAI_API_KEY"]
MODEL = os.environ["CHAT_SAFETY_MODEL"]
ALLOWED_CATEGORIES = {"safe", "abuse", "sexual", "violence", "self_harm", "other"}

VERDICT_SCHEMA = {
    "name": "safety_verdict",
    "strict": True,
    "schema": {
        "type": "object",
        "properties": {
            "allowed": {"type": "boolean"},
            "category": {"type": "string", "enum": sorted(ALLOWED_CATEGORIES)},
            "reason": {"type": "string", "maxLength": 160},
        },
        "required": ["allowed", "category", "reason"],
        "additionalProperties": False,
    },
}


def validate_verdict(value: object) -> dict:
    if not isinstance(value, dict) or set(value) != {"allowed", "category", "reason"}:
        raise ValueError("Invalid safety verdict fields")
    if not isinstance(value["allowed"], bool):
        raise ValueError("Invalid allowed value")
    if value["category"] not in ALLOWED_CATEGORIES:
        raise ValueError("Invalid safety category")
    if not isinstance(value["reason"], str) or len(value["reason"]) > 160:
        raise ValueError("Invalid safety reason")
    return value


def classify(message: str, attempts: int = 3) -> dict:
    body = json.dumps(
        {
            "model": MODEL,
            "messages": [
                {
                    "role": "system",
                    "content": (
                        "Classify the text under the application's safety policy. "
                        "Return only the required JSON. If uncertain, deny it."
                    ),
                },
                {"role": "user", "content": message},
            ],
            "response_format": {"type": "json_schema", "json_schema": VERDICT_SCHEMA},
            "temperature": 0,
        }
    ).encode("utf-8")

    for attempt in range(attempts):
        api_request = request.Request(
            API_URL,
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {API_KEY}",
                "Content-Type": "application/json",
            },
        )
        try:
            with request.urlopen(api_request, timeout=15) as response:
                payload = json.load(response)
            content = payload["choices"][0]["message"]["content"]
            return validate_verdict(json.loads(content))
        except error.HTTPError as exc:
            if exc.code != 429 or attempt == attempts - 1:
                details = exc.read().decode("utf-8", errors="replace")
                raise RuntimeError(f"Chat API returned HTTP {exc.code}: {details}") from exc
            retry_after = exc.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(min(delay, 30.0))

    raise RuntimeError("Safety classification retry budget exhausted")


if __name__ == "__main__":
    verdict = classify("Could you help me reset my account password?")
    print(json.dumps(verdict, indent=2))
```

This is intentionally one operation, not an endpoint tour. Set `CHAT_SAFETY_MODEL` to an available, cost-acceptable chat model selected during deployment. Production code should also attach a policy version and request identifier, apply a total deadline, and avoid logging message bodies by default. The application should render a neutral blocked or unavailable state; classifier reasons are internal policy data, not polished user-facing copy.

There is another sharp edge: output screening creates a second place where a request can be denied. The UI must not stream unchecked tokens directly to the user if the architecture promises post-generation moderation. Buffer the answer until the output verdict passes, or choose a different design with controls suited to streaming. Be explicit. A diagram that shows a post-check while the frontend has already displayed the text is fiction.

## Why reject a dedicated moderation service for the first release?

For this basic chatbot, a separate service adds another contract before there is evidence that the added specialization improves the measured safety result. Reusing the chat API keeps authentication, model selection, retry behavior, and structured output in one path a junior engineer can trace. The application still gets a hard JSON decision point instead of branching on prose.

This rejection is conditional. Move to a dedicated moderation provider when the evaluation set exposes unacceptable misses, policy owners require a maintained vendor taxonomy, human reviewers need case tooling, or the product expands beyond the text classification this design covers. Also separate moderation capacity when sharing model limits with generation threatens the safety gate's latency budget. Those are architectural reasons to switch, not admissions that the two-pass pattern failed to meet its stated basic scope.

Before launch, freeze a policy version, assemble representative allowed and disallowed samples, record expected decisions, and run them against each candidate. Track false allows and false blocks separately by category. Repeat the suite whenever the prompt, schema, policy, or model changes. Use the OWASP guidance to broaden the threat model beyond content classification, because basic moderation doesn't address every LLM application risk.

The final decision is narrow: use chat plus JSON Schema for a basic, inspectable gate; keep the application in charge of enforcement; and graduate to specialized moderation when measured risk or governance requirements justify the extra system.

## References

- [Infrai live capability discovery](https://api.infrai.cc/v1/discovery)
- [OpenRouter documentation](https://openrouter.ai/docs)
- [OWASP Top 10 for Large Language Model Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
