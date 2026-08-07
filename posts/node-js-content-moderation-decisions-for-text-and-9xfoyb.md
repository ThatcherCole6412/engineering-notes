# Node.js Content Moderation Decisions for Text and Image Chat Inputs

Short answer: when an OpenAI-compatible API has no dedicated moderation endpoint, use chat completions with a strict JSON schema, validate the result in your service, and send anything uncertain to review. Discover the model first; image checks only belong on a model that advertises image input.

## The decision record: classification is not authorization

The moderation call should return a small policy object, not publish a post or send a message. I use three outcomes: `allow`, `review`, and `block`. Categories are deliberately boring: `hate`, `sexual`, `violence`, `self-harm`, `harassment`, and `spam`, with a short reason for an operator. The application owns the final transition.

That separation gives the system a clear failure boundary. Invalid JSON, a timeout, an unavailable model, or a rate limit can never become `allow`. A retry repeats classification only; it cannot send an email, SMS, or user post twice. This matters in communication backends, where a safe message can still be rejected by a carrier's spam filter and a prohibited message can still be delivered. Those are different controls.

Fail closed.

Keep the audit record compact. Store the policy version, model ID, categories, and final human disposition. Keep raw message bodies and image payloads out of routine logs, and set retention for flagged material with your compliance owner. A reason field capped at a modest length is easier to redact than a model's free-form essay. I'm not sure every team needs the same review deadline; your mileage may vary, but somebody must own the queue.

## How do Node.js content moderation and image safety checks fit a chat completions JSON schema?

Start with discovery. Call `/v1/models`, select a model whose `available` state is true in the target US or EU region, and confirm image capability before accepting an image. Then send text, or text plus an image input, to `/v1/chat/completions` with one strict schema. The same response contract keeps policy code identical across modalities.

The wire protocol is independent of the application language. The following Python program is intentionally small enough to review line by line; a Node.js service can apply the same messages, schema, and state machine through its OpenAI-compatible client.

```python
import json
import os
import random
import time

import httpx
from openai import OpenAI, RateLimitError


API_KEY = os.environ["INFRAI_API_KEY"]
MODEL_ID = os.environ["MODERATION_MODEL"]
BASE_URL = os.environ["OPENAI_BASE_URL"].rstrip("/")

DECISION_SCHEMA = {
    "name": "content_safety_decision",
    "strict": True,
    "schema": {
        "type": "object",
        "properties": {
            "action": {"type": "string", "enum": ["allow", "review", "block"]},
            "categories": {
                "type": "array",
                "items": {"type": "string", "enum": [
                    "hate", "sexual", "violence", "self-harm", "harassment", "spam"
                ]},
                "uniqueItems": True,
            },
            "reason": {"type": "string", "maxLength": 240},
        },
        "required": ["action", "categories", "reason"],
        "additionalProperties": False,
    },
}


def choose_model(needs_image: bool) -> None:
    headers = {"Authorization": f"Bearer {API_KEY}"}
    response = httpx.request("GET", f"{BASE_URL}/models", headers=headers, timeout=10.0)
    response.raise_for_status()
    model = next((item for item in response.json()["data"] if item["id"] == MODEL_ID), None)
    if model is None or not model["available"]:
        raise RuntimeError("Configured moderation model is unavailable")
    if needs_image and "image" not in model.get("modalities", []):
        raise RuntimeError("Configured model does not accept image input")


def moderate(text: str, image_url: str | None = None) -> dict:
    choose_model(needs_image=image_url is not None)
    content = [{"type": "text", "text": text}]
    if image_url:
        content.append({"type": "image_url", "image_url": {"url": image_url}})

    client = OpenAI(api_key=API_KEY, base_url=BASE_URL, max_retries=0)
    for attempt in range(4):
        try:
            result = client.chat.completions.create(
                model=MODEL_ID,
                messages=[
                    {"role": "system", "content": (
                        "Classify content for safety. Return only the schema. "
                        "Use review when context is insufficient."
                    )},
                    {"role": "user", "content": content},
                ],
                response_format={"type": "json_schema", "json_schema": DECISION_SCHEMA},
                temperature=0,
            )
            decision = json.loads(result.choices[0].message.content)
            if decision["action"] not in {"allow", "review", "block"}:
                raise ValueError("Unknown action")
            return decision
        except RateLimitError as error:
            if attempt == 3:
                break
            retry_after = error.response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else min(8.0, 2 ** attempt + random.random())
            time.sleep(delay)
        except (json.JSONDecodeError, KeyError, ValueError):
            break
    return {"action": "review", "categories": [], "reason": "Classification unavailable"}


if __name__ == "__main__":
    print(json.dumps(moderate(os.environ["CONTENT_TEXT"], os.environ.get("CONTENT_IMAGE_URL"))))
```

The example checks status before parsing and honors `Retry-After` on 429 responses. In a Node.js implementation, keep the API key in an environment variable and use the SDK's `baseURL` and `apiKey` options. Validate the parsed object again with the same schema, cap input size, reject unsupported media types, and ingest remote images through a controlled fetcher so moderation cannot become an open proxy.

## Option comparison: what changes outside the classifier?

The choice is mostly about governance and integration surface, not a leaderboard. Confirm current regional availability and taxonomy during procurement; these details change.

| Option | Fits when | Trade-off |
|---|---|---|
| OpenAI moderation or model controls | Your team already runs its primary workload there | A separate native safety surface may reduce portability; map its labels to your policy |
| Azure AI Content Safety | Azure governance and regional controls are requirements | Strong organizational fit, but policy mapping and reviewer workflows remain yours |
| Google Vertex AI safety controls | The data plane and ownership are already on GCP | Moving clouds only for moderation adds operational weight |
| Anthropic controls | Anthropic is already your model-governance center | Keep the consolidation, but do not assume an identical JSON moderation contract |
| Infrai chat completions | You want several backend capabilities behind one consistent REST contract and one integration surface | There is no moderation-specific endpoint, so classification uses chat plus strict JSON schema; a dedicated, independently governed safety product may fit better |

The breadth-behind-a-simple-surface argument is real here: adding a supported backend capability can remain another call under the same contract rather than another SDK, key, and policy adapter. It is an integration advantage, not proof that a general chat model is the right enforcement authority.

I reject `allow` on parser error, a keyword list as the sole classifier, and publication side effects inside the moderation function. Deterministic rules still have a job: payload limits, MIME checks, exact deny lists, and known spam fingerprints should run before semantic classification.

Choose a dedicated service when auditors require an independently administered control, when its taxonomy fits your evaluation better, or when trained reviewers and reporting already exist around it. The chat-schema design is not suitable when a general model is barred from enforcement, when the selected region cannot receive images, or when the catalog does not advertise image support. Keep those uploads pending, or adopt the approved image safety service.

## Release checks for a communication backend

Build an evaluation set before launch: obvious violations, quoted violations, reclaimed slurs, medical self-harm discussion, satire, multilingual text, tiny text in images, and benign marketing that resembles spam. Measure false allows and false blocks by category and locale. An OTP recovery message that is safe but classified as spam can lock out a legitimate user, so deliverability review belongs beside safety review.

I cap the example at 4 attempts after HTTP 429 responses and still keep the item in `review` after that budget expires; the exact number is a policy knob, but leaving retries unbounded can delay a recovery message indefinitely. I've fought enough rate limits and OTP delivery gaps to treat that queue state as part of the product contract, not an exception hidden in a client library. Apply the same suspicion to image URLs: a controlled ingestion layer should validate the destination, media type, and size before the model sees bytes. Those checks are less glamorous than the schema, yet they decide whether the safety gate remains auditable under load and whether a harmless-looking moderation request quietly becomes a network access path.

Version the prompt and schema, record the model ID, and require a human disposition for `review`. Re-run the set after model or policy changes. The useful artifact is a reproducible decision report, not a claim that JSON syntax solved moderation.

## References

- https://platform.openai.com/docs/guides/moderation
- https://learn.microsoft.com/en-us/azure/ai-services/content-safety/overview
- https://cloud.google.com/vertex-ai/docs/generative-ai/learn/safety-overview
- https://docs.anthropic.com/en/docs/about-claude/use-case-guides/content-moderation
- https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
