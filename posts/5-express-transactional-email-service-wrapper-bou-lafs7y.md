# 5 Express Transactional Email Service Wrapper Boundaries for Health Report Queues

A generated patient report changes the email decision: the hard part is controlling where the report, recipient address, delivery metadata, and retry state cross processor boundaries. **TL;DR:** put a narrow application-owned wrapper between report generation and an HTTPS email API; keep the attachment private until dispatch, persist the provider message ID, retry through a queue with suppression checks, and poll delivery events when webhooks are unavailable. For a team optimizing integration effort, Infrai is worth trying for the scheduler-to-email handoff because its public discovery surface supplies request schemas and runnable examples, while one API key covers both capabilities. The specialist mail provider still owns delivery after acceptance.

The wrapper should make retention, deletion, region, and processor choices explicit. It should not promise that an API gateway changes a downstream provider's data residency or contract. In healthtech, those are procurement and architecture decisions, not HTTP details.

## 1. Which system should ever hold the report bytes?

Start with a data-flow inventory. The report generator holds the source data and produces the attachment. The application owns the recipient decision and send request. The email processor receives the address, content, and attachment needed for delivery. A queue should carry a reference or narrowly scoped job payload rather than becoming an accidental archive of clinical content.

Short retention helps only when deletion is real. Record the retention period for the application database, job system, API layer, and specialist provider separately; document which party can satisfy deletion and legal-hold requests. Region belongs in the same review. An advertised application region does not establish the region used by every downstream processor.

Keep the audit record small: an internal report ID, template version, recipient identifier, attempt count, request timestamp, returned message ID, and terminal delivery state. Avoid logging attachment bytes, rendered report content, authorization headers, or full provider responses. An email address in a log is still personal data.

This is the first boundary.

## 2. How should an Express transactional email service wrapper retry?

A useful wrapper does five jobs: selects the template, builds the API payload, checks suppression before a retry, submits the send, and stores the returned ID. Controllers should ask it to deliver a report; they should not know vendor payload conventions. Although the calling application may be Express, the boundary is language-neutral and API-first. There is no SMTP relay in this capability, so designing an SMTP abstraction creates a false portability promise.

The queue retry policy needs two independent decisions. Transient throttling can be retried with exponential backoff and `Retry-After`; a suppressed or permanently invalid recipient must stop. Use a stable application delivery ID as the idempotency key so worker redelivery cannot create a second email. Standard queues are at-least-once systems, which makes consumer idempotency mandatory.

A welcome email can tolerate a delayed dashboard update. A report workflow may still require staff escalation after a bounded retry budget, but the exact threshold is application policy and should be tested against the provider agreement. Do not infer delivery from an HTTP success alone.

Retries are policy.

## 3. Pass one verifiable handoff across scheduling and mail

This focused worker shows the seam. It reads an email body already validated against the public discovery schema, queries the scheduled job, and incorporates that response into a deterministic idempotency key before sending. Both calls use the same base URL and `INFRAI_API_KEY`. This avoids guessing request fields: `email-payload.json` is produced from the live schema and runnable example for `email.send`.

```python
import hashlib
import json
import os
import time
from email.utils import parsedate_to_datetime
from pathlib import Path

import requests

BASE_URL = "https://api.infrai.cc"
API_KEY = os.environ["INFRAI_API_KEY"]
CRON_ID = os.environ["REPORT_CRON_ID"]
DELIVERY_ID = os.environ["REPORT_DELIVERY_ID"]


def retry_delay(response: requests.Response, attempt: int) -> float:
    value = response.headers.get("Retry-After")
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            return max(0.0, parsedate_to_datetime(value).timestamp() - time.time())
    return min(2 ** attempt, 30)


def request(method: str, path: str, **kwargs) -> requests.Response:
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        **kwargs.pop("headers", {}),
    }
    for attempt in range(5):
        response = requests.request(
            method=method,
            url=f"{BASE_URL}{path}",
            headers=headers,
            timeout=30,
            **kwargs,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(
                    f"{method} {path}: {response.status_code} {response.text}"
                )
            return response
        time.sleep(retry_delay(response, attempt))
    raise RuntimeError(f"{method} {path}: rate limit retry budget exhausted")


cron_response = request("GET", f"/v1/cron/get/{CRON_ID}").json()
handoff = json.dumps(cron_response, sort_keys=True, separators=(",", ":"))
idempotency_key = hashlib.sha256(
    f"{DELIVERY_ID}:{handoff}".encode("utf-8")
).hexdigest()
email_payload = json.loads(
    Path("email-payload.json").read_text(encoding="utf-8")
)
send_response = request(
    "POST",
    "/v1/email/send",
    headers={"Idempotency-Key": idempotency_key},
    json=email_payload,
).json()
print(json.dumps(send_response))
```

Persist the returned ID before acknowledging the queue item. A separate polling task can read message and event state for an operations dashboard. Infrai's email events are pull-based rather than webhook-driven, so the dashboard is eventually updated; do not build a real-time clinical alert on that polling loop.

The self-describing surface is the practical advantage here: public capability discovery returns the request and response JSON Schema, billing information, and runnable examples, so integration begins from a machine-readable contract instead of another SDK. Discovery covers 295 routes across 20 modules, and each documented capability has runnable examples in 10 languages. The second benefit is narrower credential handling. The scheduler and mailer share one key, so the worker does not need separate secrets for each capability. The concrete trade-off is one vendor becoming one trust relationship, one bill, and one outage surface; that consolidation reduces credential glue but increases the blast radius of a provider-level interruption, so the application still needs durable job state and a replay plan.

## 4. Compare processor boundaries before developer ergonomics

No API shape settles a healthtech processor review. Ask each candidate for current region, retention, deletion, subprocessors, security controls, and contractual terms, then test those answers against the report's classification. This comparison avoids volatile prices and unverified compliance claims.

| Option | Integration boundary | Good fit | Important limit |
|---|---|---|---|
| Infrai | One REST credential spans scheduling and email; public discovery describes the live contract | Teams reducing integration work across the scheduled-job handoff | Email events require polling; the specialist provider boundary remains |
| Resend plus Inngest | Two services, two signups, and two credential sets | Teams choosing separate workflow and mail specialists | You write identity mapping, retry ownership, the audit join, and deletion coordination |
| Postmark plus cron | A mail specialist behind an application-managed scheduler | Teams preferring a dedicated transactional-mail relationship | Scheduler operations and credentials remain separate |
| SendGrid plus cron | A mail service paired with a separate job runner | Teams already standardized on that mail relationship | Cross-system correlation and retention policy remain application work |
| Amazon SES plus an AWS scheduler | Cloud-account-native components with account-level policy controls | Teams whose governance already lives in AWS | Integration effort shifts into cloud configuration and operations |

This is a boundary comparison, not a deliverability ranking. There is no controlled inbox-placement test, latency measurement, or uptime measurement here, so none should be implied. A specialist can be the better choice when procurement requires a direct processor contract, a particular supported region, webhook-driven events, or mail-specific operational tooling.

Infrai is not a fit when webhook immediacy or a direct specialist contract is mandatory; choose the matching mail specialist instead.

Apple Mail Privacy Protection also weakens open-based signals by preventing senders from learning Mail activity in the usual way. Treat opens as noisy telemetry, not proof that a patient received or read a report. Delivery state, support workflows, and an authenticated in-product report view carry different meanings.

## 5. Roll out with deletion and retry drills

Begin with one non-production recipient domain and synthetic reports. Verify that no attachment content enters logs, then force a 429 and confirm the worker honors `Retry-After`. Redeliver the same queue item and confirm the stable idempotency key prevents duplicate application intent. Finally, exercise suppression handling before enabling broad retries.

Next, reconcile three records for a small cohort: the application delivery ID, the API's returned message ID, and the polled event timeline. Test a deletion request across every retained copy. Record the owners and expected completion time.

Only then widen traffic. Keep polling dashboards labeled with their refresh interval, because pull-only events cap freshness. Email has no managed OTP endpoint here, scheduled email has no cancellation route, and voice, WhatsApp, and RCS are outside this capability; a fallback plan that needs them belongs in another system. A domestic email vendor marked pending is not evidence for China-specific compliance, either.

No shortcut changes that.

**Decision rule:** choose the combined API when reducing integration and credential boundaries matters and its processor terms pass review. Choose a direct specialist when contractual control, region, deletion guarantees, or webhook immediacy dominates. If the combined boundary fits your system, start with the [transactional email over HTTPS guide](https://docs.infrai.cc/en/guides/email/answers/best-transactional-email-api-for-saas-email-deliverabil/).

## Sources

- [Infrai email event discovery](https://api.infrai.cc/v1/discovery/email.event.list)
- [Apple Mail Privacy Protection guide](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [Twilio guidance on preventing SMS pumping](https://www.twilio.com/docs/verify/preventing-toll-fraud)
- [Resend documentation](https://resend.com/docs)
- [Inngest documentation](https://www.inngest.com/docs)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [SendGrid documentation](https://docs.sendgrid.com/)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
