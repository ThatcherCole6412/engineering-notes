# Order Receipts and Incident Alerts: Picking a Bulk SMS API Without Monthly Minimums

Pick the least complex path that can prove a message arrived: one queue per channel, an idempotency key on every send, and a stored provider message id you can reconcile against an invoice later. That rule decides more than any price page for a B2B SaaS that emails an order receipt the moment a payment settles and pages on-call by SMS when the receipt worker stalls. Delivery reliability, not the headline per-message rate, is what should put a bulk SMS alerts API into your incident path.

Rates move. Evidence doesn't.

A per-destination price list takes a minute to read. Finding out whether a route into Ireland or Ohio actually lands your traffic, and whether you can prove it a month later, takes a controlled drill and one reconciled invoice. So this walks the constraint first — the receipt has to arrive, the page has to wake somebody — and only then looks at what genuinely varies between vendors.

## What delivery reliability means when the receipt is the product

For an order receipt, "sent" is a worthless state. The customer's expectation is set by the payment confirmation screen, so a receipt that lands nine minutes later reads as a billing incident, and support gets the ticket either way.

Three failure modes dominate this path and none of them appear on a price page. The payment webhook is delivered twice, so the customer receives two receipts for one charge. The receipt worker stalls behind a slow PDF render, so nothing goes out at all — and no alarm fires, because queue depth is graphed as a five-minute average and the spike hides inside it. Or the alert path itself gets throttled: the gateway accepts the first few pages, answers the rest with 429 and a retry-after, and the European half of the rotation finds out about the stalled receipts forty minutes late.

Only the third one is a vendor question.

The other two are yours, and they stay yours no matter whose API you sign up with. That matters when reading comparisons, because a lot of what gets attributed to a provider is really an application that never stored a message id, never deduplicated a webhook, and had no way to tell "accepted by the gateway" apart from "delivered to the handset".

On the delivery side there are two details I'd check before anything else, because they quietly break alert copy. Sender identity comes first: US A2P traffic over standard long codes needs brand and campaign registration, alphanumeric sender IDs are not available for that traffic, and several European countries want sender IDs pre-registered before they will pass them. Encoding comes second. A plain GSM-7 body fits 160 characters, or 153 per part once a message is concatenated; a single smart quote, an em dash pasted from a runbook, or a `€` sign flips the whole body to UCS-2, where the limits drop to 70 and 67. An alert template that was one segment in staging becomes three in production, which changes both the bill and the time to deliver.

Compliance sits underneath all of it. The FTC's CAN-SPAM guidance is the primary source for how commercial email content is treated in the US, and the line between a transactional receipt and a promotional message is thinner than most templates assume — a cross-sell block at the bottom of a receipt can move it across that line. SMS consent rules are a separate question again, and country-specific. I'm not a lawyer and this isn't legal advice; have counsel read the actual template and the actual destination list.

## How should a SaaS team compare bulk SMS alerts APIs for incidents when there is no monthly minimum?

Define the workload before opening anyone's pricing page. Sending country, destination mix, sender type, peak recipients per incident, repeat-page policy, and the deadline at which SMS escalates to a phone call. Without those six numbers, every quote is unfalsifiable.

"No monthly minimum" is a real and useful property — it means the floor is set by usage rather than by a contract — but the floor is rarely zero. Number rental, registration fees for US A2P, and per-destination rates all keep running whether or not you send anything, and surcharges differ by operator inside the same country. Telnyx, Bandwidth, Twilio and Sinch each publish per-destination rate lists and each run their own registration flow for US traffic; those lists change often enough that any number quoted in an article like this one would be stale before you read it. Get current quotes, then verify them against your own drill.

The drill is the actual comparison. Send a fixed set of synthetic alerts through each finalist, to the same destinations, in the same hour, and record one row per attempt.

| What to measure | Where it comes from | Why it decides the shortlist |
| --- | --- | --- |
| Delivery receipt rate per destination | DLR callbacks or status polling, grouped by country and operator | Acceptance is free; delivery is the thing you're buying |
| Throttle behaviour | HTTP 429 responses, retry-after values, observed burst ceiling | An incident is exactly when your send rate spikes |
| Sender identity support | Registration flow, alphanumeric sender availability per country | A blocked sender ID is a silent 100% failure in one market |
| Reconciliation | Provider message id present on both the API response and the invoice | Without it, cost analysis is guesswork |
| Escalation latency | Timestamp of accept, of DLR, and of the on-call acknowledgement | This is the number the incident review will ask about |

Averaging US and EU results into one row destroys the comparison, since the operators, the registration rules and the rates are all different. Keep the destination on every row and aggregate late.

## Two queues, one ledger, and an idempotency key

Receipts and pages have opposite requirements, so give them separate queues. The receipt queue is ordered and patient: it can retry for an hour, and duplicate suppression matters more than latency. The alert queue is impatient and lossy-by-design: if a page can't go out in thirty seconds it should already be escalating to a second channel rather than retrying quietly.

Both write to the same ledger table before the request leaves your process. Row first, send second — if the row isn't there, you can't tell a lost message from a lost log line.

The idempotency key is what keeps a webhook replay or a redeployed worker from double-sending. Derive it from something the business already owns (the payment id for a receipt, the incident id plus escalation step for a page), not from a timestamp or a random value generated at call time.

```python
import hashlib
import os

import requests

GATEWAY = os.environ["SMS_GATEWAY_URL"]      # provider base URL or your own fan-out service
TOKEN = os.environ["SMS_GATEWAY_TOKEN"]


def segments(body: str) -> int:
    """Rough SMS segment count: 160 GSM-7 chars, 70 for UCS-2 (153/67 when concatenated).

    The exact rule is the GSM 03.38 character table; this approximation is close
    enough to catch a template that silently doubled in cost.
    """
    limit, chained = (70, 67) if any(ord(c) > 0x7F for c in body) else (160, 153)
    return 1 if len(body) <= limit else -(-len(body) // chained)


def alert_key(incident_id: str, msisdn: str, step: int) -> str:
    """Stable across retries and redeploys, unique per escalation step."""
    raw = f"{incident_id}:{msisdn}:{step}".encode()
    return hashlib.sha256(raw).hexdigest()[:32]


def page(incident_id: str, msisdn: str, body: str, step: int, ledger) -> dict:
    key = alert_key(incident_id, msisdn, step)
    ledger.record_attempt(key=key, incident=incident_id, destination=msisdn,
                          parts=segments(body), state="pending")

    reply = requests.post(
        f"{GATEWAY}/messages",
        headers={"Authorization": f"Bearer {TOKEN}", "Idempotency-Key": key},
        json={"to": msisdn, "text": body, "reference": key},
        timeout=8,
    )

    if reply.status_code == 429:
        # Keep the throttle visible: an incident timeline that only shows the
        # successful retry is a comforting lie.
        ledger.record_state(key, "throttled", retry_after=reply.headers.get("Retry-After"))
        return {"key": key, "state": "throttled"}

    reply.raise_for_status()
    provider_id = reply.json()["id"]
    ledger.record_state(key, "accepted", provider_id=provider_id)
    return {"key": key, "state": "accepted", "provider_id": provider_id}
```

Three things in that function are the whole point, and they survive a change of provider. That idempotency key is derived, not generated. The throttle is a recorded state rather than an exception swallowed by a retry wrapper. And the provider's own id goes into the ledger the instant it exists, which is what makes the invoice line, the DLR callback and the incident timeline joinable afterwards. Everything else — the header names, the path, the shape of the JSON body — is a thin adapter you can rewrite in an afternoon, which is why I'd argue about the ledger schema in review and not about the client library.

## What the invoice can and cannot tell you

An invoice tells you what you were charged and roughly for what. It will not tell you which incident, which team, or which retry policy produced the charge, and no dashboard will reconstruct that after the fact.

So carry your own tags through to the ledger and join on the provider message id at the end of the month. That join is the only honest cost comparison anyone in this thread is going to get.

## Migrating one route at a time

Run the new provider in parallel rather than cutting over. Route ten percent of alerts and a duplicate copy of the drill traffic through it for a week, keep the old credentials warm, and compare DLR rates per destination rather than in aggregate. Set the exit condition in advance: what delivery rate, over how many attempts, in which countries, before the old route is retired.

The catch is that parallel running costs real money on both accounts and doubles the on-call surface while it lasts, so it isn't a good fit for a two-person team paging a single rotation in one country. Stick with the incumbent when the measured difference is smaller than the migration risk — an existing integration that reconciles cleanly is worth more than a slightly better rate card. The trade-off flips once you have a market where delivery is measurably failing, because at that point the second route is redundancy, not procurement.

One more boundary worth naming: none of this makes SMS a good primary channel for receipts. It's the escalation channel. If a receipt genuinely has to arrive on a handset, that's a product decision with its own consent and compliance work, and it should not be smuggled in through the alerting stack.

## References

- https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
- https://www.itu.int/rec/T-REC-E.164
- https://datatracker.ietf.org/doc/html/rfc7489
- https://resend.com/docs/introduction

## Further reading

- https://www.rfc-editor.org/rfc/rfc5321
- https://www.rfc-editor.org/rfc/rfc8058
