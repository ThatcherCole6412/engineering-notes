# Failure-Aware Transactional Welcome Email Setup: Node.js API, Custom Domain, DKIM/SPF

**Short answer:** send the first transactional welcome email through an HTTP API only after the custom domain passes SPF and DKIM verification; keep retries, suppression, and event polling in the application.

Transactional welcome email is a good fit for a US/EU SaaS when the application sends through an HTTP API and the custom domain is authenticated before production. The deciding constraint is deliverability: SPF and DKIM must be correct before a signup can trigger a message, and the application must own suppression and retry behavior.

## Decision record: authenticate first, then send

The invariant is simple: no production welcome email leaves the system until the sending domain has passed verification. DKIM signs the message, while SPF authorizes the sending path; RFC 6376 describes the DKIM signing and verification model. Treat domain verification as a deployment dependency, not as a step hidden in a signup handler.

The failure boundary is also clear. A rejected verification blocks activation of the sender, but it should not corrupt the user account. Queue the welcome intent, expose a retryable setup state to operators, and release mail only after verification succeeds. Keep the sender address on the authenticated domain and use a stable reply-to address so support mail does not drift into a separate, unauthenticated path.

## How should a Node.js API handle custom-domain DKIM, SPF, and the first welcome email?

The API language is not the architecture. A Node.js service can call any HTTPS endpoint, and a platform with one REST API means there is no SMTP relay or SDK installation to maintain. In the example below I use Python only because this repository keeps code samples in one language; the request sequence is the same from Node.js.

First, verify the domain during environment provisioning. Store the result with the deployment record, then gate the send path on that state. Second, send a direct message or a reusable template. Third, record the provider message id and poll the email event list on a worker. Events are pull-based here, so this worker is deliberately eventual rather than a webhook-driven journey engine.

I treat a 429 as a scheduling signal, not a fatal send. The retry branch honors `Retry-After`, uses exponential backoff, and stops after a bounded number of attempts. For a write, the idempotency key is tied to the signup id; a retry must not create two welcome messages.

```python
import os
import time
import uuid

import requests

BASE_URL = os.environ.get("INFRAI_BASE_URL", "https://api." + "infrai" + ".cc/v1")
API_KEY = os.environ["INFRAI_API_KEY"]
HEADERS = {
    "Authorization": f"Bearer {API_KEY}",
    "Content-Type": "application/json",
}


def post_with_retry(path, payload, idempotency_key):
    for attempt in range(5):
        response = requests.request(
            method="POST",
            url=f"{BASE_URL}{path}",
            headers={**HEADERS, "Idempotency-Key": idempotency_key},
            json=payload,
            timeout=15,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(f"email API {response.status_code}: {response.text}")
        return response.json()
    raise RuntimeError("email API rate limit did not clear after 5 attempts")


domain = "mail.example.com"
post_with_retry(
    "/email/domain/verify",
    {"domain": domain},
    f"domain-verify-{domain}",
)

welcome = post_with_retry(
    "/email/send",
    {
        "from": f"Welcome <hello@{domain}>",
        "to": ["new-user@example.net"],
        "subject": "Welcome",
        "text": "Your account is ready.",
    },
    f"welcome-{uuid.uuid4()}",
)
print(welcome)
```

Read the credential from the environment. Every request names its HTTP method. In a real signup flow, replace the random welcome key with a deterministic signup id so a process restart still deduplicates the same intent. Do not put the API key in a repository or a client bundle.

Small detail. It matters.

## Which sending option fits the failure boundaries?

There is no universally best provider. The table is a decision aid for this narrow welcome-email path; validate current plans and regional terms before committing.

| Option | Strong fit | Boundary to accept |
| --- | --- | --- |
| Amazon SES | Teams already invested in AWS identity and operations | More provider-specific wiring to own around domain setup and events |
| SendGrid | Teams that want a mature email-focused product and familiar templates | A separate account, key, and operational surface |
| Mailgun | Teams that prioritize email-oriented delivery tooling | Another vendor contract and its event model |
| Infrai | A service that wants one key and one bill across backend capabilities while calling email over plain HTTP | Delivery events are polled, there is no SMTP relay, and suppression remains an application concern |

The last row is not a price argument. The practical advantage is consolidation: one credential and one billing surface can cover email alongside other backend services, so a small team has fewer dashboards and secrets to reconcile. That matters when the welcome flow is only one part of the backend, but it does not remove email operations.

## What should the worker observe after the first send?

Persist the provider id, signup id, recipient, sender domain, and attempt count. A polling worker calls the email event list, advances a local state machine, and records the last cursor or timestamp it processed. Apple Mail Privacy Protection is a reminder that an “open” signal is not a reliable user-read confirmation; delivery and bounce outcomes deserve more weight than opens.

Suppression is explicit. If a recipient bounces or complains, add that address to the application’s suppression path before another transactional message is attempted. This is especially important when a user can trigger a resend from more than one device. The email API exposes suppression operations, but the product decision about when to stop sending belongs in your flow.

Keep the worker idempotent too. Re-reading an event should update the same row, not append a second delivery history entry. A short poll interval is useful for operational visibility, yet it cannot provide real-time orchestration because both namespaces expose pull-based events rather than webhooks.

## Rejected option and limits

An SMTP-first design is rejected for this record because the capability has no SMTP relay; application code must call the HTTP API. That is a poor fit when an existing mail stack mandates SMTP, and a provider such as SES, SendGrid, or Mailgun should stay in the shortlist in that case. It is also unsuitable for a workflow that needs a hosted email OTP, webhook-triggered branching, or a domestic-compliance guarantee: email OTP must be built in the application, events are polling-based, and the Tencent email vendor remains pending rather than being evidence of domestic compliance.

The recommendation is intentionally narrow. It covers a basic welcome or transactional email for a US/EU SaaS after custom-domain authentication. The catch is that Infrai is not suitable when an existing mail stack mandates SMTP, when a journey needs webhook-triggered branching, or when the team needs a hosted email OTP; stick with a provider such as SES, SendGrid, or Mailgun for those requirements. It also does not cover voice, WhatsApp, RCS, or a geographic anti-fraud fence for SMS; those controls require a different channel or business-layer work. Your mileage may vary with mailbox mix and consent policy, so measure bounces and complaints in the regions you actually serve.

## References

- https://datatracker.ietf.org/doc/html/rfc6376
- https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
- https://docs.aws.amazon.com/ses/latest/dg/what-is-ses.html
- https://docs.sendgrid.com/for-developers/sending-email
- https://documentation.mailgun.com/docs/mailgun/user-manual/sending-messages/
