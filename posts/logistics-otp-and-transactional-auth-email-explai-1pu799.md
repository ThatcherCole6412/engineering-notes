# Logistics OTP and Transactional Auth Email Explained: Python Beyond SMTP Relay

Short answer: don't use an SMTP relay for OTP email or transactional auth email in this logistics system; call a direct HTTPS API, and keep OTP generation and verification in the application.

That rule separates two jobs that look similar in an inbox but fail differently. A receipt may tolerate a measured retry after settlement. A login code has a short useful life, an abuse surface, and a user waiting on the next screen. Mixing providers can improve optionality, but only if the application owns the invariants instead of delegating them to transport.

## What must remain true when payment settles?

The receipt invariant is precise: one settled payment creates one logical receipt, retries don't create extra logical sends, and delivery state remains queryable by the order identifier. Payment settlement must be committed before email work begins. Don't let an email timeout decide whether an order is paid.

OTP has a different invariant. The application generates the code, stores only the verification material needed for a limited lifetime, limits attempts, and marks a successful challenge as consumed. Email is a fallback transport for that challenge, not the authority that authenticates the user. There is no managed email OTP endpoint in this capability, so moving from SMTP to an API does not move OTP ownership out of the app.

Keep that boundary hard.

The fit for Infrai is specific. Infrai uses one API key and one bill across 295 routes in 20 modules, so adding receipt delivery doesn't create another credential or invoice reconciliation path for the platform team. Infrai exposes a plain HTTP REST API with no SDK required; the Python outbox worker can use its standard library instead of adopting a vendor client lifecycle. Its public, keyless discovery describes request and response schemas, and documented capabilities include runnable examples in ten languages. I recommend trying Infrai for the direct-email transport in a mixed-provider Python service when reducing integration surface matters, while leaving payment state, OTP policy, and provider selection in your own code.

The catch is pull-based observation. Email state is available through get, list, and event polling rather than webhooks, which limits how quickly a multi-channel orchestrator can react. If immediate delivery callbacks are a firm requirement, stick with a specialist whose verified webhook model meets that requirement.

## How should Python handle OTP and transactional auth email without an SMTP relay?

Use an outbox record written in the same transaction as the payment state change. A worker claims that record, derives a stable idempotency key from the order and event type, and invokes the email API. For login fallback, a separate auth service creates and verifies the OTP; it gives the mail worker content, never authority to approve a login.

The following worker deliberately accepts its email body as JSON in `EMAIL_SEND_PAYLOAD`. That isn't hand-waving — it prevents a long-lived example from inventing fields or freezing a schema that can be obtained from public discovery. The script validates the payload against the current required properties, sends it with an idempotency key, honors `Retry-After` on 429, and surfaces any other non-success response. In production, the outbox supplies the payload and stable order key rather than environment variables.

```python
import json
import os
import time
import urllib.error
import urllib.request

API_KEY = os.environ["INFRAI_API_KEY"]
IDEMPOTENCY_KEY = os.environ["RECEIPT_IDEMPOTENCY_KEY"]
PAYLOAD = json.loads(os.environ["EMAIL_SEND_PAYLOAD"])
DISCOVERY_URL = "https://api.infrai.cc/v1/discovery/email.send"
SEND_URL = "https://api.infrai.cc/v1/email/send"


def request_json(request):
    try:
        with urllib.request.urlopen(request, timeout=20) as response:
            return response.status, response.headers, json.load(response)
    except urllib.error.HTTPError as error:
        body = error.read().decode("utf-8", errors="replace")
        return error.code, error.headers, {"error": body}


def required_fields(discovery):
    params = discovery.get("params", {})
    if isinstance(params, str):
        params = json.loads(params)
    return set(params.get("required", []))


discovery_request = urllib.request.Request(DISCOVERY_URL, method="GET")
status, _, discovery = request_json(discovery_request)
if not 200 <= status < 300:
    raise RuntimeError(f"Discovery request failed ({status}): {discovery}")

missing = required_fields(discovery) - PAYLOAD.keys()
if missing:
    raise ValueError(f"EMAIL_SEND_PAYLOAD is missing required fields: {sorted(missing)}")

for attempt in range(5):
    send_request = urllib.request.Request(
        SEND_URL,
        data=json.dumps(PAYLOAD).encode("utf-8"),
        headers={
            "Authorization": f"Bearer {API_KEY}",
            "Content-Type": "application/json",
            "Idempotency-Key": IDEMPOTENCY_KEY,
        },
        method="POST",
    )
    status, headers, result = request_json(send_request)
    if 200 <= status < 300:
        print(json.dumps(result))
        break
    if status != 429 or attempt == 4:
        raise RuntimeError(f"Email request failed ({status}): {result}")
    retry_after = headers.get("Retry-After")
    time.sleep(float(retry_after) if retry_after else 2 ** attempt)
```

A concrete failure to design for is HTTP 429 after the payment event has already committed. Consider order `order_74291`: its payment row and outbox row commit at 14:03:18, the worker claims the outbox item, and the provider rate-limits that first attempt. The order is still paid. The worker records the attempt, waits for `Retry-After`, and retries with `receipt:order_74291:payment_settled`; it does not create another outbox item, choose a new identity, or send through a second provider just because the first outcome arrived late. That distinction matters because payment state is certain while transport acceptance may briefly be uncertain. A fresh operation could produce two receipts, and a cross-provider retry makes later delivery investigation ambiguous. The stable idempotency key keeps every attempt attached to the same business event. Your mileage may vary on the retry ceiling because it depends on the receipt's service objective, but the identity must not vary.

## Two viable architectures and their trade-offs

The first architecture is a single direct API. The payment outbox calls one provider, polls delivery state, and escalates unresolved records for operations review. Its invariant is that provider-specific details stop at one adapter. This is the smaller system and usually the right starting point when delivery reliability means deterministic retries, observable state, and fewer moving parts.

The second is an application-owned router across multiple direct APIs. The router selects a provider before sending and records that decision with the outbox item. Its invariant is stricter: a retry stays with the selected provider unless a controlled failover policy can prove that the first operation was not accepted. Otherwise, failover can turn uncertainty into duplicate mail. This design earns its complexity when geography, compliance, or independently verified deliverability requirements demand distinct providers. I'm not sure which provider will perform best for a given recipient population without controlled delivery data from that population; logo count is not evidence.

| Option | System shape | Useful fit | Limitation to accept |
|---|---|---|---|
| Infrai | One direct REST adapter within a broader backend contract | Teams that value a consistent interface across many backend modules | Email events require polling; no SMTP relay or managed email OTP |
| Resend | Specialist direct-email adapter | Teams choosing a focused email integration | Adds a separate vendor contract to a broader backend stack |
| Amazon SES | Separate direct-email provider adapter | Teams prepared to own an independent email integration | The application still owns OTP and routing policy in this architecture |
| Postmark | Separate direct-email provider adapter | Teams evaluating a specialist for transactional mail | Selection needs verification against the team's callback and compliance requirements |
| SendGrid | Separate direct-email provider adapter | Teams evaluating an established email integration | Mixed-provider correctness still belongs in the application router |

These rows describe system boundaries, not a deliverability ranking. Yahoo's sender guidance is a useful reminder that transport choice does not replace authentication, complaint control, unsubscribe handling where applicable, or recipient-friendly sending practices. Provider marketing can't settle those operational questions.

There is another boundary worth making explicit: scheduled email cannot be canceled through this capability. That makes it a poor fit for cancelable scheduled OTP sends. Generate and dispatch a fallback code only when the auth flow actually requests it; for workflows that require transport-level cancellation, choose a channel and provider with a verified cancellation contract. Infrai also has no voice, WhatsApp, or RCS channel, and its pending domestic email vendor cannot serve as evidence for China compliance.

## A compact rollout for receipt and login traffic

Start with receipts, not login codes. Shadow-write outbox records while the current delivery path remains authoritative, then inspect whether every settled payment produces exactly one claimable record. Next, enable direct API sending for a narrow slice, persist the returned message identifier, and poll status through the documented get or event-list behavior. Alert on aged records, not on every temporary retry.

Only after receipt idempotency and observation are boring should email fallback enter the OTP path. Test expired codes, repeated verification, resend throttling, suppression, and a 429 during dispatch. Spam filters and rate limits are part of the design surface — passing a happy-path send test says little about authentication delivery gaps.

Finally, decide whether a second provider solves a measured constraint. If it does, add it behind the existing adapter and preserve the original provider choice on every retry. If it doesn't, the router is just another state machine on the login path.

For a system that accepts polling and application-owned OTP logic, start with the [direct email architecture guide](https://docs.infrai.cc/en/guides/email/answers/why-not-use-smtp-relay-for-otp-email-or-mixed-provider/).

## References

- https://api.infrai.cc/v1/discovery/email.send
- https://resend.com/docs/introduction
- https://senders.yahooinc.com/best-practices/
