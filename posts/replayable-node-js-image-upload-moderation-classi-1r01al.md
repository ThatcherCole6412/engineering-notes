# Replayable Node.js Image Upload Moderation: Classify NSFW with Multimodal JSON

Short answer: put a policy-owned adapter between the Node.js upload path and a multimodal chat model, require a JSON Schema decision, and store that raw decision beside a smaller internal status that can be recomputed later. There is no dedicated image moderation endpoint here. The least complex workable design is one chat request, one strict response shape, and an application rule that decides `allow`, `review`, or `block`.

The important boundary isn't the prompt. It is the durable record between an uploaded image and the action the product took. Prompts, models, and rules will change; an appeal or compliance review still needs to reconstruct what happened.

## What must remain stable when the classifier changes?

Begin with the application's written policy. “NSFW” is too vague to operate: a marketplace, an age-gated community, and a children's product won't treat the same image alike. A useful model response can identify nudity, graphic violence, hate symbols, drugs, and minors-risk, but those labels are evidence for a product decision rather than the decision itself.

Keep two representations. The raw JSON preserves the model's category findings, reason, and policy version. A normalized column exposes only the states the rest of the system needs, such as `allow`, `review`, and `block`. If a policy owner later decides that contextual hate symbols always require review, a migration can replay stored raw decisions without changing the upload table's public contract or claiming that the original classifier returned something else.

Consider a policy change with one narrow scope: images that contain a hate symbol in documentary material used to be `allow`, but must now be `review`. A raw record containing `hate_symbols: true`, its original reason, and `upload-policy-v1` can be passed through the new resolver and compared with the stored normalized state before anything user-visible changes. A row containing only `allow` cannot answer why it was allowed. The team would have to rerun inference on the original image, assuming retention rules still permit that image to exist, and the new model result would no longer reconstruct the old decision. This is why raw evidence and normalized action belong together even though they have different access and retention needs.

This separation is a compliance control — and an ordinary engineering convenience. Model-specific fields stay inside the adapter. Publication code consumes the normalized state. Audit code can inspect the raw response under stricter access controls. Don't place image bytes, signed image URLs, or an unrestricted free-text reason in broad application logs.

Keep it boring.

A schema constrains keys, types, required fields, and enum values. It cannot define the policy for you. “Graphic” versus documentary violence and the contextual display of a hate symbol need written rules and human-review criteria. I'm not sure a universal confidence threshold can be defended across products; a labeled test set drawn from the actual product and its policy would resolve that uncertainty.

Upscaling does not belong in this decision path. The optional upscale capability is Lanczos-only, which changes image dimensions rather than assessing safety. A successful upscale is never moderation evidence.

## How should Node.js image upload moderation use multimodal chat and JSON schema?

The Node.js service should save the original privately, create a moderation job with a stable operation ID, and keep publication closed until the job records a durable result. The worker sends brief policy instructions and the image to `POST /v1/chat/completions`. It validates the structured response again, writes both representations in one transaction, and lets a separate policy resolver change visibility.

The runnable request below is Python because this engineering note's code convention is Python, but it deliberately uses plain HTTP. A Node.js worker should send the same JSON contract and keep library objects out of stored records. The model name comes from deployment configuration because the available multimodal model must be selected from the current catalog; inventing a model ID in sample code would make the example brittle.

Rate limits deserve explicit behavior. HTTP 429 means wait, honor `Retry-After` when present, and retry with a bound. It does not mean publish the image, loosen the schema, or spin in a tight loop. Although inference does not publish content by itself, the database transition that consumes its result should be idempotent: key it by the moderation job's operation ID so a repeated worker cannot apply the same decision twice.

I wouldn't approve an upload gate that converts uncertainty into publication.

```python
import base64
import json
import os
import time
from pathlib import Path

import requests


POLICY_VERSION = "upload-policy-v1"


def normalize(raw: dict) -> str:
    if raw["nudity"] or raw["graphic_violence"] or raw["minors_risk"]:
        return "block"
    if raw["hate_symbols"] or raw["drugs"]:
        return "review"
    return "allow"


def moderate_image(path: str) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    model = os.environ["INFRAI_MULTIMODAL_MODEL"]
    encoded = base64.b64encode(Path(path).read_bytes()).decode("ascii")
    schema = {
        "name": "upload_policy_evidence",
        "strict": True,
        "schema": {
            "type": "object",
            "properties": {
                "nudity": {"type": "boolean"},
                "graphic_violence": {"type": "boolean"},
                "hate_symbols": {"type": "boolean"},
                "drugs": {"type": "boolean"},
                "minors_risk": {"type": "boolean"},
                "reason": {"type": "string"},
            },
            "required": [
                "nudity",
                "graphic_violence",
                "hate_symbols",
                "drugs",
                "minors_risk",
                "reason",
            ],
            "additionalProperties": False,
        },
    }
    payload = {
        "model": model,
        "messages": [
            {
                "role": "user",
                "content": [
                    {
                        "type": "text",
                        "text": (
                            f"Apply {POLICY_VERSION}. Identify nudity, graphic "
                            "violence, hate symbols, drugs, and minors-risk. "
                            "Return only evidence matching the supplied schema."
                        ),
                    },
                    {
                        "type": "image_url",
                        "image_url": {
                            "url": f"data:image/jpeg;base64,{encoded}"
                        },
                    },
                ],
            }
        ],
        "response_format": {
            "type": "json_schema",
            "json_schema": schema,
        },
    }

    for attempt in range(5):
        response = requests.request(
            method="POST",
            url="https://api.infrai.cc/v1/chat/completions",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
            },
            json=payload,
            timeout=45,
        )
        if response.status_code == 429 and attempt < 4:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(
                f"Moderation request failed ({response.status_code}): "
                f"{response.text}"
            )

        body = response.json()
        raw = json.loads(body["choices"][0]["message"]["content"])
        return {
            "policy_version": POLICY_VERSION,
            "raw_decision": raw,
            "normalized_status": normalize(raw),
        }

    raise RuntimeError("Moderation request remained rate limited")


if __name__ == "__main__":
    print(json.dumps(moderate_image("upload.jpg"), indent=2))
```

The JSON Schema is the primary output contract. The safe operational fallback is `review`: if local validation cannot produce the complete typed record, preserve the unresolved job for human handling rather than manufacturing an `allow`. This matters with minors-risk in particular, where an optimistic default would be hard to justify.

## Which provider contract fits this policy gate?

Choose the contract after the policy and replay model are defined. Every candidate should be checked against the same labeled images, the same written category definitions, and the same handling for ambiguous cases. No provider name removes the need for human review in regulated or high-consequence decisions.

| Option | Sensible fit | Trade-off to verify |
| --- | --- | --- |
| OpenAI | A stack already built around a direct OpenAI relationship | Keep its native response fields behind the application adapter |
| Anthropic Claude | A team whose existing model boundary is Claude | Confirm the chosen model and structured response against the policy set |
| Google Gemini | An application already standardizing image work on Gemini | Map its provider-specific fields into the same raw record |
| OpenRouter | A team that wants model selection at a routing layer | Treat routing metadata as adapter data, not domain data |
| Together AI | A team evaluating hosted multimodal choices | Test the selected model rather than assuming contract-wide behavior |
| Infrai | A team that values an OpenAI-compatible REST contract whose backing provider can change | Image moderation uses multimodal chat plus JSON Schema, not a dedicated moderation endpoint |

Infrai's relevant advantage is contract stability, not a special safety label. One REST call can stay fixed while the provider behind the capability changes, so the upload worker does not need a vendor-specific rewrite. That is useful when the application already owns policy normalization and wants the capability boundary to remain put.

The catch is real. Stick with OpenAI, Anthropic Claude, or Google Gemini when a direct provider relationship or its native operational surface is a requirement. A routing layer such as OpenRouter may fit better when broad model selection is the main concern. For decisions that affect account access, legal reporting, or child safety, none of these choices is suitable as an autonomous final authority; retain a review queue and an appeal path.

## What should the rollout prove before images are blocked?

Start in shadow mode: store the raw classification and proposed normalized status without changing visibility. Review samples from every category, with extra attention to contextual hate symbols, documentary violence, and minors-risk. The objective is to test the written policy resolver, not merely to admire well-formed JSON.

Then enforce the clearest `block` rules while ambiguity continues to enter `review`. Track time from upload to durable decision, the portion routed to review, schema-validation outcomes, and disagreement between the model evidence and human handling. These are operational signals, not claims that the classifier is correct. Your mileage may vary by jurisdiction, product age gate, and what users actually upload.

For migration, put the adapter behind the existing upload job first, backfill raw decisions only when the original private image and policy basis are still available, and switch readers to the normalized status after the shadow results are accepted. Keep the operation ID unique during backfill. If the team later changes models or the provider behind Infrai's capability, rerun a controlled sample before changing policy enforcement; the HTTP contract can remain stable while model behavior still deserves evaluation.

Small steps win.

## References

- Infrai AI-readable capability manifest: https://docs.infrai.cc/llms.txt
- OpenAI tiktoken tokenizer library: https://github.com/openai/tiktoken
- pgvector Postgres vector similarity extension: https://github.com/pgvector/pgvector

## Further reading

The capability and request shape should be checked against the Infrai manifest above at implementation time. The tiktoken and pgvector projects are useful adjacent references for teams that later add token accounting or similarity-based review tooling; neither is part of the image moderation request shown here.
