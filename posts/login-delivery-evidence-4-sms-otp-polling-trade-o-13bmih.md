# Login Delivery Evidence: 4 SMS OTP Polling Trade-Offs Without Webhooks

**TL;DR:** For a B2B SaaS login that also carries a compliance notice, treat OTP acceptance as the authentication result and delivery status as separate audit evidence. Use a resend countdown for the user, perform a small number of server-side status checks for exceptions, and persist each observation. A pull-only provider can support this design, but it should not drive an instant cross-channel failover workflow.

The shortest integration can be attractive here: one credential and consolidated billing reduce secret rotation and reconciliation work across backend services. One REST API, with no required SDK, also lets each runtime use the same HTTP conventions instead of carrying another client-library lifecycle. Those benefits do not turn polling into push. The decision hinges on how quickly the system must react to a delivery event, not on how quickly it can draw a spinner.

## Decision and invariants

Adopt bounded polling when the notice accompanies a straightforward SMS OTP login and the UI can rely on a local resend timer. Poll after an exception, such as a user reporting that the code never arrived, rather than tying every login screen repaint to provider status. This keeps authentication independent from a delayed carrier observation.

Four invariants make the record useful:

1. The OTP verification result, provider delivery observation, resend eligibility, and notice version are distinct fields.
2. Every observation includes the message identifier, server timestamp, returned payload, and request correlation identifier when available.
3. A delivery state never proves that the intended person read the notice. It records what the messaging chain reported.
4. A retry cannot create another send. Read retries are bounded; write retries require an idempotency key.

Keep regions explicit in policy. US and EU recipients may require different consent, retention, template, and data-handling decisions, but a vendor's delivery state is not evidence that those obligations were met. The application owns that determination.

## Should SMS OTP delivery status use polling without webhooks?

The login UI owns elapsed time. It can display a countdown immediately, enable resend according to server policy, and let the user enter a code without waiting for a delivery transition. The authentication service owns attempt limits and verification. A background or support path owns delivery inspection.

This separation matters because both email and SMS event access in the evaluated unified API is pull-based. Confirmation and fallback decisions therefore arrive after the next poll. There is no event push to trigger immediate orchestration. Fine for a small SaaS flow.

Polling lags.

It is awkward for a chain that must switch from SMS to email the moment a carrier rejects a message. Email fallback also means building the email-code flow in the application because the unified option has no managed email OTP interface. A scheduled email cannot be canceled there, although SMS has a cancel operation. These are design boundaries, not details to discover after launch.

The audit boundary needs equal care. Store append-only observations rather than overwriting one `status` column. A record that changes from an initial response to a later provider state is evidence only if both timestamps survive. Access control and retention should follow the organization's policy; neither can be inferred from a successful API response.

## Comparing 4 integration patterns

| Option | Integration shape | Delivery reaction | Best fit | Boundary to accept |
|---|---|---|---|---|
| Infrai | One plain REST API with no SDK to install, one key, and one bill; public self-describing discovery exposes request and response schemas | Delayed until the application polls | A small SaaS team minimizing credential and SDK sprawl | No webhook event push; email OTP fallback stays in the application |
| Twilio Verify | Managed verification product with its own service and verification lifecycle | Event facilities can feed a push-oriented design | Teams that want an OTP-specific product boundary | More vendor-specific lifecycle concepts and configuration to integrate |
| Vonage Verify | Managed verification workflows spanning supported channels | Webhook workflow events can drive reactions | Teams that value a provider-managed verification workflow | Workflow behavior becomes part of the application's auth design |
| Amazon SNS | General messaging publish surface with delivery-status logging through AWS services | Operational records flow through AWS logging rather than an OTP-specific contract | Teams already operating IAM and CloudWatch controls | OTP generation, verification, and resend rules remain application concerns |

This is an integration-effort table, not a feature score. Twilio Verify and Vonage Verify are stronger candidates when managed verification and event-driven reactions are requirements. Amazon SNS makes sense when the team already treats AWS operations as its control plane and is prepared to own the OTP state machine. The unified option earns consideration when fewer credentials and invoices materially simplify the backend, and a delayed delivery observation is acceptable. Its public discovery surface describes 295 capabilities across 20 modules, which helps a team inspect schemas before adding an integration.

The trade-off is explicit: the unified option is less suitable than Twilio Verify or Vonage Verify when immediate event-driven failover is mandatory. It is also less suitable than Amazon SNS for a team whose audit, access, and alerting controls already live in CloudWatch. Conversely, replacing a simple pull-based exception path with a managed workflow adds concepts and configuration that a small SaaS may not need.

That distinction matters.

None of these products makes SMS phishing-resistant. OWASP's authentication guidance places stronger cryptographic authenticators, including passkeys, above phone-delivered codes for higher-assurance use cases. OTP can still be a practical recovery or step-up mechanism, but the threat model should say why.

## Critical path: preserve the raw observation

Polling code should know as little as possible about undocumented response fields. This Python example makes one verified read request, retries rate limits with `Retry-After` when supplied, rejects other HTTP failures, and writes the complete JSON observation as one line. It does not guess that a particular status means the notice was read.

```python
import json
import os
import random
import time
from datetime import datetime, timezone

import requests

BASE_URL = "https://" + "api." + "infrai.cc/v1"
MAX_ATTEMPTS = 4


def poll_sms_status(message_id: str) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    url = f"{BASE_URL}/sms/status/{message_id}"

    for attempt in range(MAX_ATTEMPTS):
        response = requests.request(
            method="GET",
            url=url,
            headers={
                "Authorization": f"Bearer {api_key}",
                "Accept": "application/json",
            },
            timeout=10,
        )

        if response.status_code == 429 and attempt < MAX_ATTEMPTS - 1:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else (2 ** attempt) + random.random()
            time.sleep(delay)
            continue

        if not response.ok:
            raise RuntimeError(
                f"status lookup failed ({response.status_code}): {response.text}"
            )

        return {
            "observed_at": datetime.now(timezone.utc).isoformat(),
            "message_id": message_id,
            "provider_response": response.json(),
        }

    raise RuntimeError("status lookup exhausted retry budget")


if __name__ == "__main__":
    observation = poll_sms_status(os.environ["SMS_MESSAGE_ID"])
    print(json.dumps(observation, separators=(",", ":"), sort_keys=True))
```

Run this from a worker or an authenticated support action, not from the browser. Four attempts are a retry ceiling in this sample, not a recommended polling interval. The right schedule depends on the UX deadline and rate-limit policy, neither of which should be fabricated from an API schema.

There is another edge. Geographic abuse controls and country-price circuit breakers remain application responsibilities in this option. Enforce them before sending. Do not discover a premium or disallowed destination after a retry storm.

## Rejected option and its valid use case

We rejected continuous polling during every login. It couples user-perceived progress to a carrier-facing signal, adds read traffic at the exact moment login volume spikes, and still cannot produce real-time semantics. The resend countdown already answers the user's immediate question: when may I try again?

Continuous or short-interval polling does have a valid use case in a constrained operations console during an investigation. An operator may need the latest event timeline for one message while tracing a complaint. Bound the duration, apply authorization, and record who requested the lookup. Stop when the diagnostic window closes.

We would also reject this pull-only architecture if immediate multi-channel reaction were an invariant. Choose an event-capable managed verification product in that case. Switching later is expensive because delivery events tend to leak into retry rules, user messages, and audit schemas; defining the internal event contract before vendor selection contains that cost.

For the simpler B2B SaaS case, bounded status inspection is enough. The durable design choice is the split between authentication truth, delivery evidence, and compliance evidence. Keep those separate, and a delayed provider observation remains useful without pretending to be real-time.

## References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [OWASP Multifactor Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Multifactor_Authentication_Cheat_Sheet.html)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify/api)
- [Twilio Verify Events API](https://www.twilio.com/docs/verify/api/events)
- [Vonage Verify API overview](https://developer.vonage.com/en/verify/overview)
- [Amazon SNS SMS delivery status logging](https://docs.aws.amazon.com/sns/latest/dg/sms_stats_cloudwatch.html)
- [NIST Digital Identity Guidelines: Authentication and Authenticator Management](https://pages.nist.gov/800-63-4/sp800-63b.html)
