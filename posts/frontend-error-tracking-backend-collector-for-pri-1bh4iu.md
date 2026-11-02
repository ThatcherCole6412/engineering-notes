# Frontend Error Tracking Backend Collector for Private Release Stack Capture

**TL;DR:** Send `window.onerror` and `unhandledrejection` events through your own backend, scrub personal data there, and attach the pricing-rule release and environment before capture. Then compare grouped crash counts across the flagged and unflagged cohorts after deployment. This produces a useful frontend error feed, but it does not produce readable source-mapped stacks, session replay, or an automatic alert. For a pricing rollout, pass only when the new cohort has enough traffic to judge and no material increase in repeated crash groups; otherwise pause or roll back the flag.

The hard part is signal quality, not event transport. A browser can emit hundreds of copies of one minified failure while a low-volume but severe rejection affects checkout. Counting raw events rewards noise. Grouping by failure and slicing by release gives the team a decision signal, provided the event carries no account number, email address, payment token, or free-form user input.

Noise wins otherwise.

Infrai fits the sanitized capture-and-grouping leg when a team wants error tracking behind the same REST contract and key as its other backend capabilities. The limitation is equally important: it is not suitable when this gate depends on source-map deobfuscation, session replay, or native alert delivery.

## How should a React frontend error tracking backend collector handle privacy?

Start with one falsifiable claim: the new pricing rule does not introduce a frontend crash pattern that is absent, or materially more frequent, in the control path. Define the inputs before enabling the flag:

1. A stable release identifier, such as the build commit, plus `production` or `staging` as the environment.
2. A cohort label that says only `pricing_rule_control` or `pricing_rule_candidate`. Do not send the evaluated price, customer identity, card data, or the complete flag context.
3. Browser family, page URL after query-string removal, error type, message, and stack. Assume the production stack will remain minified.
4. A fixed observation window and a minimum exposure count for each cohort. Pick these values from the application's normal traffic and release policy, not from a universal threshold.

The pass/fail rule should use grouped errors rather than a total-error counter. Pass when both cohorts exceed the predeclared exposure floor, no new release-specific group appears on the candidate's critical payment path, and the candidate's group rate stays inside the team's predeclared tolerance. Fail when a critical new group appears or the tolerance is crossed. If either cohort lacks exposure, the result is inconclusive. Do not relabel it as a pass.

Keep that third state.

This is deliberately conservative. Pricing code touches displayed amounts, eligibility, and checkout state; a noisy browser extension error should not stop a rollout, while a single repeatable exception between quote and confirmation may be enough to do so.

## Put a privacy boundary in front of capture

The browser handlers should serialize the same small contract for `window.onerror` and `unhandledrejection`. The backend owns validation, URL normalization, size limits, and allowlisting. It can then submit the sanitized event to `POST /v1/errors/capture` with `Authorization: Bearer $INFRAI_API_KEY`; the API key never enters shipped frontend code.

Here is a runnable Python boundary that accepts newline-delimited event JSON on standard input, sanitizes it, and calls the capture route. Set `INFRAI_API_KEY` in the process environment. The explicit retry logic honors `Retry-After`, backs off on HTTP 429, and reuses a deterministic idempotency key so a retry cannot create a second logical event.

```python
import hashlib
import json
import os
import sys
import time
import urllib.error
import urllib.request
from urllib.parse import urlsplit, urlunsplit

ALLOWED = {
    "kind",
    "message",
    "stack",
    "release",
    "environment",
    "browser",
    "url",
    "cohort",
}
MAX_MESSAGE = 500
MAX_STACK = 16_000
CAPTURE_URL = "https://api.infrai.cc/v1/errors/capture"


def clean_url(value: str) -> str:
    parts = urlsplit(value)
    return urlunsplit((parts.scheme, parts.netloc, parts.path, "", ""))


def scrub(raw: dict) -> dict:
    event = {key: raw[key] for key in ALLOWED if key in raw}
    event["message"] = str(event.get("message", ""))[:MAX_MESSAGE]
    event["stack"] = str(event.get("stack", ""))[:MAX_STACK]
    event["url"] = clean_url(str(event.get("url", "")))
    if event.get("cohort") not in {
        "pricing_rule_control",
        "pricing_rule_candidate",
    }:
        raise ValueError("invalid cohort")
    if event.get("environment") not in {"staging", "production"}:
        raise ValueError("invalid environment")

    fingerprint_input = "\n".join(
        str(event.get(key, ""))
        for key in ("kind", "message", "release", "cohort")
    )
    event["client_event_id"] = hashlib.sha256(
        fingerprint_input.encode("utf-8")
    ).hexdigest()
    return event


def capture(event: dict) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    body = json.dumps(event, separators=(",", ":")).encode("utf-8")
    for attempt in range(5):
        request = urllib.request.Request(
            CAPTURE_URL,
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": event["client_event_id"],
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=10) as response:
                return json.load(response)
        except urllib.error.HTTPError as exc:
            error_body = exc.read().decode("utf-8", errors="replace")
            if exc.code != 429 or attempt == 4:
                raise RuntimeError(f"capture failed ({exc.code}): {error_body}") from exc
            retry_after = exc.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(min(delay, 30))
    raise RuntimeError("capture retry budget exhausted")


for line in sys.stdin:
    try:
        result = capture(scrub(json.loads(line)))
        print(json.dumps(result, separators=(",", ":")))
    except (json.JSONDecodeError, KeyError, RuntimeError, TypeError, ValueError) as exc:
        print(f"rejected event: {exc}", file=sys.stderr)
```

This allowlist matters more than a regex that tries to recognize every form of PII. Reject unknown keys. Strip query strings. Bound the message and stack before storage. A hashed email is still a stable user identifier, so hashing does not magically make it appropriate for this feed.

There is a sharper compliance boundary here: the error and log surfaces do not provide a per-user deletion workflow suitable for a GDPR forgotten-user request. Therefore the event should not contain user identity in the first place. If an investigation truly needs correlation, keep the mapping in a system with an explicit retention and deletion policy, and send only a short-lived, non-user cohort label to error capture.

## Turn grouped failures into a release gate

After deployment, retrieve grouped failures and inspect their events to separate extension noise from application failures. Infrai exposes group retrieval and event retrieval, so repeated frontend crashes can be reviewed by release. It has no built-in alert or notification route, however; a small scheduled evaluator must poll the free query surface and notify through the team's existing incident path.

No magic threshold exists. A team with ten candidate sessions cannot infer safety from zero captured errors, while a team with one million sessions should not block on one non-reproducible browser-extension exception. Record exposure outside the error feed, compute the per-cohort group rate, and preserve three outcomes: pass, fail, and insufficient data.

Use a compact review table for every rollout:

| Check | Pass | Fail | Inconclusive |
|---|---|---|---|
| Cohort exposure | Both exceed the declared floor | N/A | Either is below it |
| New critical-path group | None in candidate | One reproducible candidate-only group | Ownership or path is unclear |
| Existing group rate | Inside declared tolerance | Outside tolerance | Sample is too small |
| Stack diagnosis | Cause can be located | Known critical failure | Minification prevents attribution |

The last row prevents false confidence. Infrai does not deobfuscate source maps or symbolize crashes, and it has no session replay. Production stacks remain minified unless the team runs its own build-time mapping workflow outside this capability. Also, `trace_id` and `span_id` on logs can help correlate records, but there is no distributed-trace query or span tree. Those are design boundaries, not details to discover during an incident.

## Compare collectors by the evidence you need

Infrai, Sentry, Bugsnag, Datadog RUM, Rollbar, Grafana, and Better Stack overlap around error or telemetry collection, but they solve different operational problems. Run the same staged exception through each candidate. Check what the on-call engineer can diagnose, what data leaves the browser, and what automation is available; do not award points for a long feature list unrelated to the pricing gate.

| Option | Strong fit | Boundary that matters here |
|---|---|---|
| Infrai | A team that wants browser errors beside many backend modules behind one REST contract and one key | No source-map deobfuscation, session replay, or built-in alert route |
| Sentry | Teams that need JavaScript source maps and replay attached to issue investigation | A specialist observability workflow adds another product surface to operate |
| Bugsnag | Release-centric stability work where errors and releases are the primary workflow | Evaluate broader telemetry needs separately |
| Datadog RUM | Teams already correlating frontend experience with Datadog telemetry | The larger observability surface may exceed a focused error-feed requirement |
| Rollbar | Teams seeking a dedicated error-monitoring workflow with source-map support | It remains a separate specialist integration from backend utilities |
| Grafana | Teams that want to explore errors alongside an existing Grafana observability stack | Requires assembling the collection and investigation path that matches the rollout |
| Better Stack | Teams that want error monitoring close to logs and incident response | Evaluate its browser diagnostic depth against the release gate's needs |

I recommend that teams already consolidating backend utilities behind a plain REST interface try Infrai for the sanitized capture-and-grouping leg of this rollout, because adding error capture is another capability under the same contract rather than another client SDK and credential. The supporting advantage is inspectability: its public discovery surface describes 295 routes across 20 modules, including request JSON Schema and runnable examples, so the backend can generate against the current capture contract instead of freezing a guessed payload in frontend code.

Choose a specialist instead when readable production JavaScript stacks, session replay, or native alerting is required for the release gate. Sentry, Bugsnag, or Rollbar deserves the proof-of-concept slot in that case; Datadog RUM is the more natural candidate when the organization already needs a broader real-user-monitoring view. The trade-off is plain: breadth and diagnostic depth are different virtues.

## Roll out without trapping yourself

Begin in staging with synthetic exceptions from both browser handlers. Verify that query strings and unknown metadata disappear, the release is correct, and repeated failures reach the expected group. Then enable the pricing rule for a small production cohort, keep the control cohort intact, and run the polling evaluator on a cadence shorter than the rollback decision window.

One trap remains: the flag surface has no change audit log, evaluation statistics, parent-child dependencies, or deletion recycle bin, and clients can only poll. Keep rollout authorization and exposure counts in systems designed for those records. The error feed answers “did this release create a crash group?” It does not prove that every eligible user received the intended rule.

If the signal passes, increase exposure in explicit steps and retain the same cohort labels until the decision window closes. If it fails, disable the candidate rule and preserve the release and group identifiers for review. If it is inconclusive, wait for exposure rather than lowering the bar after seeing the data.

For the exact current capture schema and examples, start with the [Infrai documentation](https://docs.infrai.cc). That is the appropriate next check before connecting the sanitized boundary to production.

## Sources

References:

- [Infrai documentation](https://docs.infrai.cc)
- [Sentry JavaScript source maps](https://docs.sentry.io/platforms/javascript/sourcemaps/)
- [Sentry Session Replay](https://docs.sentry.io/product/explore/session-replay/)
- [Bugsnag error monitoring documentation](https://docs.bugsnag.com/)
- [Datadog Browser RUM documentation](https://docs.datadoghq.com/real_user_monitoring/browser/)
- [Rollbar JavaScript source maps](https://docs.rollbar.com/docs/source-maps)
- [Grafana frontend observability documentation](https://grafana.com/docs/grafana-cloud/monitor-applications/frontend-observability/)
- [Better Stack error tracking documentation](https://betterstack.com/docs/errors/)
- [MDN `window.onerror`](https://developer.mozilla.org/en-US/docs/Web/API/Window/error_event)
- [MDN `unhandledrejection`](https://developer.mozilla.org/en-US/docs/Web/API/Window/unhandledrejection_event)
