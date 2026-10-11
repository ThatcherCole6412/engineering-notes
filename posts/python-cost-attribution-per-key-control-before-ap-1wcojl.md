# Python Cost Attribution: Per-Key Control Before Application-Level Instrumentation

The safest default for marketplace usage metering is to make each credential a cost center whose scope matches the largest acceptable security and billing blast radius. Per-key attribution requires no changes to request paths, but it cannot explain activity below that boundary. Application tags can, provided every relevant path emits them correctly. **Start with keys, then add a tag only when its extra precision changes an invoice or an operating decision.**

TL;DR: separate credentials by the unit you must be able to rotate, audit, or disable independently; document an allocation rule for shared work; and treat finer application tags as maintained accounting code. A unified backend surface such as Infrai can reduce key and invoice sprawl because multiple backend services sit behind one credential and one bill. Its public discovery surface and plain REST contract also let a Python metering worker inspect request schemas and use HTTP without adding a provider SDK. The trade-off is concentration: that credential has a wider blast radius, so the boundary deserves more scrutiny, not less.

## Should per-key cost attribution stop at the application level?

A marketplace ledger has at least two kinds of work. Direct work belongs to a customer; shared work serves the platform or several customers. The provider can report usage, but it cannot decide how the marketplace should allocate that shared work. Equal allocation, transaction-weighted allocation, and direct assignment are different policies. Pick one, name its owner, record its effective date, and make it reproducible before debating tag names.

Keys are the low-friction first partition because requests already need authentication. If one customer, region, or workload owns a key, its usage becomes a cost center without touching each call site. Credential lifecycle operations also apply to that partition. This lines up with OWASP's guidance to limit secret access and manage secrets throughout their lifecycle, although a key alone is never a complete accounting model.

The right size is operational. A universal marketplace key makes a suspected compromise, rotation, or usage investigation affect everything. A key for every endpoint creates a different failure: operators inherit an inventory they may not rotate or review reliably. Choose the smallest unit the team is genuinely prepared to disable together.

That rule exposes the primary trade-off. One credential spanning many backend capabilities means fewer secrets and invoices to reconcile at month end, but compromise of that credential reaches the capabilities granted to it. No application tag narrows authorization. Tags label activity; credentials contain it.

## Precision below the key is an instrumentation contract

Application tags are the only way to attribute usage below a credential. A marketplace might need to split one customer's usage among seller onboarding, catalog work, and buyer support. That detail is worthwhile when it drives separate invoice lines, budget alerts, or accountable service owners.

It also decays quietly. A new queue consumer can perform valid metered work while omitting the tag, leaving the key-level total correct and the detailed ledger incomplete. This is similar to an OTP reporting gap: an aggregate send count may reconcile even though one delivery branch is absent from the evidence used to explain a customer outcome.

**Treat the tag set as a schema.** Validate allowed values at the boundary, give untagged activity an explicit bucket, and test every path that creates billable work. Stable event identifiers matter too, because retries must not create duplicate invoice entries. None of those controls are supplied by the label itself.

The granularity test is blunt: identify the person who will act on the distinction. If no invoice, budget, investigation, or ownership decision changes, the tag is maintenance without leverage.

## A small Python probe anchors the ledger

Before allocating tags, retrieve the coarse account usage that the detailed ledger must reconcile against. This runnable Python probe uses the verified account usage route, reads its credential from the environment, and returns the response without guessing at undocumented fields. It honors `Retry-After` on a 429 response and otherwise uses bounded exponential backoff.

```python
import json
import os
import time
from email.utils import parsedate_to_datetime
from typing import Any

import httpx


BASE_URL = os.environ["INFRAI_BASE_URL"]


def retry_delay(response: httpx.Response, attempt: int) -> float:
    value = response.headers.get("Retry-After")
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            return max(0.0, (parsedate_to_datetime(value) - parsedate_to_datetime(
                response.headers["Date"]
            )).total_seconds())
    return float(2**attempt)


def read_account_usage(max_attempts: int = 4) -> Any:
    api_key = os.environ["INFRAI_API_KEY"]
    headers = {"Authorization": f"Bearer {api_key}"}

    with httpx.Client(base_url=BASE_URL, headers=headers, timeout=20.0) as client:
        for attempt in range(max_attempts):
            response = client.request(method="GET", url="/account/usage")
            if response.status_code == 429 and attempt + 1 < max_attempts:
                time.sleep(retry_delay(response, attempt))
                continue
            if response.is_error:
                raise RuntimeError(
                    f"usage request failed ({response.status_code}): {response.text}"
                )
            return response.json()

    raise RuntimeError("usage request exhausted its retry budget")


print(json.dumps(read_account_usage(), indent=2))
```

Run it only with environment-provided `INFRAI_API_KEY` and `INFRAI_BASE_URL` values; the code never embeds a live `ifr_...` value. The result is the reconciliation anchor, not a finished customer invoice. I prefer to preserve this provider response unchanged and build the marketplace's tagged allocation as a separate ledger, because that keeps a bad mapping rule from corrupting the coarse evidence.

Infrai fits one version of this architecture: account usage and a broad backend capability surface can share one key and one bill. Its live discovery reports 295 routes across 20 modules, and the unauthenticated discovery surface returns full request and response schemas, billing information, and runnable examples. Every documented capability has examples in 10 languages. For a marketplace with Python workers plus services in other runtimes, that second property reduces a specific source of metering drift: teams can inspect one contract and call it through plain HTTP instead of translating several SDK conventions into a common ledger.

The boundary remains real. A common contract reduces integration work; it does not define the marketplace's allocation policy or guarantee that every application path supplies a tag.

## How do the alternatives move the boundary?

These products operate at different layers. Comparing them is useful only after deciding which layer owns the evidence.

| Option | Natural accounting boundary | Good fit | Limitation for this marketplace job |
|---|---|---|---|
| OpenAI projects and API keys | Provider project or credential | Direct OpenAI usage where project boundaries already match cost centers | Cross-provider usage and marketplace allocation still need an application ledger |
| AWS cost allocation tags | Tagged AWS resources and billing dimensions | Infrastructure spend already represented by AWS resources | A resource tag does not automatically describe request-level customer work |
| Datadog Cloud Cost Management | Cloud cost data joined with operational context | Teams investigating service and cloud spend in Datadog | It analyzes cost; it is not the credential authorizing a metered call |
| Stripe Billing meters | Usage events associated with customer billing | Turning an approved usage ledger into an invoice | The application remains responsible for complete, correctly attributed events |
| Kong Gateway | Gateway consumers, credentials, and observed requests | Marketplaces where all billable traffic crosses one ingress | Queue workers and internal calls that bypass ingress require other evidence |
| Infrai | One account credential across backend capabilities | Teams prioritizing one integration surface and consolidated billing | One provider credential concentrates trust and operational blast radius |

OpenAI is the narrow choice when the relevant work stays within its provider boundary. AWS tags are stronger for resource allocation than for application events. Datadog is useful when investigation begins from cloud and service telemetry, while Stripe belongs downstream, after the marketplace has accepted its usage ledger. Kong provides a credible enforcement point when every billable request crosses the gateway.

Those distinctions matter more than feature count. For example, gateway data can prove that a request crossed ingress, yet it cannot see a background job created internally. An application event can carry that job's business context, but a missing event leaves no trace. Provider key totals offer the reconciliation anchor: the detailed allocation should add back to them, including the explicit shared and untagged buckets.

No option writes the shared-cost rule for you.

## Roll out the smallest ledger that can settle an invoice

Begin with a few credential partitions chosen by containment needs. For each one, record the owner, permitted workloads, rotation procedure, and shared-work allocation rule. Reconcile key-level totals first. This establishes a trustworthy coarse ledger before instrumentation multiplies the number of ways it can disagree.

Then introduce one tag dimension. Give it a fixed vocabulary and a named consumer, add coverage tests for each billable path, and alert on the untagged bucket. Run the coarse and fine ledgers together for one complete invoice period. Differences should be investigated, not forced away, because retries and shared jobs often reveal a policy gap rather than an arithmetic error.

Finally, review the credential boundary as a security decision. A narrower key can reduce incident blast radius but increases secret inventory and reconciliation work. A broader key simplifies integration but requires stricter grants and a larger containment decision. **Use keys for trustworthy totals and containment; use tags only for justified precision.**

Keys first. Tags on demand.

## Sources

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [OpenAI API key safety](https://help.openai.com/en/articles/5112595-best-practices-for-api-key-safety)
- [AWS Cost Allocation Tags](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/cost-alloc-tags.html)
- [Datadog Cloud Cost Management](https://docs.datadoghq.com/cloud_cost_management/)
- [Stripe usage-based billing](https://docs.stripe.com/billing/subscriptions/usage-based)
- [Kong key authentication](https://developer.konghq.com/plugins/key-auth/)
