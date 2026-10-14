# Property Event Notifications — 3 Email-to-SMS Delayed Status Recovery Boundaries

TL;DR: Send each property report once by email, keep the attachment in durable object storage, and poll that email attempt until it reaches a terminal state or a fixed deadline. If the deadline expires without confirmed delivery, enqueue one SMS linking to an authenticated report page. A channel-independent notification record and stable idempotency keys prevent a late email result or worker retry from producing duplicate texts.

The bill is shaped by three quantities: attachment bytes retained, status checks performed, and secondary messages sent. For `R` reports, average attachment size `B`, retention window `D`, poll count `P`, and fallback rate `F`, the controllable workload is `R x B x D` byte-days, `R x P` status reads, and `R x F` SMS submissions. Photographic property reports can make attachment retention the dominant term; aggressive polling can dominate request volume for small PDFs. Measure both.

The useful change is to retain one report object rather than a copy per attempt or channel. Email gets a short-lived attachment read. SMS gets an authenticated link, not another binary. Delivery history stores identifiers and timestamps, never report bytes.

Keep one copy.

## What should the system keep?

Use three retention boundaries. Keep the report object according to the property's document policy. Keep delivery events long enough to investigate disputes and satisfy the applicable recordkeeping policy. Keep transient polling leases only until an attempt becomes terminal or crosses its deadline. These are different data classes; giving all three the longest lifetime quietly multiplies cost and exposure.

A compact state record is enough: notification ID, report object ID, recipient reference, email attempt ID, current state, next poll time, deadline, SMS idempotency key, and event timestamps. Do not store the recipient's phone number in every event row. Resolve it through the protected contact record when the fallback job runs, and put only a redacted recipient reference in logs.

Deletion has a price. Once detailed responses age out, an operator may know that a message timed out but not why it remained delayed. Once the report object expires, a recipient following an old SMS link must request a regenerated report. Accept those limits explicitly instead of retaining attachments and raw responses forever.

## How should delayed email event notifications trigger SMS fallback?

A delayed status is an observation, not permission to resend. The email service may still be processing the message, or the final result may be temporarily unavailable. Retrying email submission while its outcome is unknown can create two valid messages. Poll the existing attempt first.

Use exponential backoff with jitter and a hard deadline. Polls near 15, 45, 105, and 225 seconds, for example, give the remote system time to settle without making a request every few seconds. Those numbers are an operating-policy example, not a protocol guarantee; tune them from observed terminal-status latency and report urgency. A property damage report may justify a shorter window than a routine monthly statement.

The state machine should distinguish `email_delivered`, `email_failed`, `email_unknown`, `sms_queued`, and `closed`. Only one atomic transition may claim the fallback. A worker waking after the deadline performs a compare-and-set from `email_unknown` to `sms_queued`; if the update affects zero rows, another worker or a late result already won.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
import hashlib
import random

@dataclass(frozen=True)
class PollDecision:
    action: str
    next_poll_at: datetime | None = None

def decide(status: str, attempts: int, deadline: datetime,
           now: datetime) -> PollDecision:
    if status == "delivered":
        return PollDecision("close")
    if status == "failed" or now >= deadline:
        return PollDecision("claim_sms_fallback")

    base_seconds = min(15 * (2 ** attempts), 240)
    jitter = random.uniform(0.8, 1.2)
    return PollDecision(
        "poll_again",
        now + timedelta(seconds=base_seconds * jitter),
    )

def sms_key(notification_id: str) -> str:
    value = f"{notification_id}:report-ready:sms".encode()
    return hashlib.sha256(value).hexdigest()

now = datetime.now(timezone.utc)
decision = decide("delayed", 2, now + timedelta(minutes=5), now)
```

The example separates decisions from side effects. The database claim, SMS submission, and result recording belong in a transactional outbox workflow. The idempotency key must remain stable across worker crashes and HTTP timeouts. Generating a new key on every retry defeats the protection.

## Timeout recovery without duplicate notices

There are two ambiguous timeouts. A status poll can time out before any response arrives. An SMS submission can time out after the receiving service accepted it. Both require retrying the same operation identity, not creating a new notification.

HTTP clients need an explicit request deadline. Fetch-style APIs return a promise, but application code still defines cancellation and response handling; the MDN Fetch documentation describes that request and response model. In a backend worker, apply the same principle with the language's HTTP client: cap connection and read time, classify transport errors separately from terminal delivery states, and record the next eligible attempt.

Recovery should replay durable outbox rows first, then resume due polls, then reconcile attempts stuck beyond their lease time. This order prevents a restart from sending a fallback before an already committed email attempt has been submitted. Keep the queue message disposable and the database record authoritative. Late evidence needs a policy. If email reports delivery after SMS has been claimed, record the event and stop; do not attempt to retract a text. If SMS is still queued, cancellation is safe only when the queue supports an atomic state change. Otherwise, favor a possible two-channel notice over losing the report, and word the SMS to acknowledge that email may also arrive. This recovery design also has a limitation: it is a poor fit for emergency notifications whose safety requirement demands immediate simultaneous channels. It deliberately trades speed for fewer duplicate and unnecessary messages. Use concurrent channel submission for that different requirement, with its higher message volume and recipient-noise cost documented up front.

Ambiguity remains.

## Why opens cannot govern fallback

An open pixel does not establish that a recipient received or read a property report. Apple Mail Privacy Protection can download remote content in the background and prevent senders from learning whether a recipient opened a message. An absent open is weak evidence, and an apparent open can also mislead.

That is a hard boundary.

Use transport status to operate the pipeline, then use an authenticated report-page view as a separate business event if acknowledgment matters. Do not collapse `delivered`, `opened`, and `report_viewed` into one state. They answer different questions.

This distinction matters for compliance too. The SMS should contain minimal context, avoid report details, and point to access-controlled content. Consent, quiet hours, opt-out handling, and message classification must be evaluated for the jurisdiction and purpose before fallback is enabled. A technically successful send can still be the wrong send.

## Operate the policy, not the happy path

Track counts and age distributions for pending email attempts, poll errors, deadline expirations, fallback claims, duplicate-claim conflicts, and late terminal results. Alert on the oldest pending attempt and sudden changes in fallback rate. A single success-rate average hides a queue that has stopped advancing.

Test with deterministic failure injection: time out a poll, return a delayed state beyond the deadline, crash after the atomic fallback claim, time out after SMS acceptance, and deliver a late email result. The invariant is short: one report object, at most one logical submission per channel, and a complete state history. Run the cases during deployment because worker version skew can apply two retry policies to one record.

The decision rule is conservative: poll known email attempts until terminal status or deadline, claim fallback exactly once, and retry transport operations under stable identities. Delete polling leases promptly, age out raw responses on a documented schedule, and retain only evidence the organization can justify. The cost is less forensic detail after expiration. The benefit is predictable storage and retry volume.

## Further reading

- Apple, Mail Privacy Protection: https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
- MDN Web Docs, Fetch API: https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API
- IETF, RFC 5321, Simple Mail Transfer Protocol: https://www.rfc-editor.org/rfc/rfc5321
- IETF, RFC 9110, HTTP Semantics: https://www.rfc-editor.org/rfc/rfc9110
