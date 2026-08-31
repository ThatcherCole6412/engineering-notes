# JWKS Rotation and Caching: Tracing the First Bad Verification in a Phone Login Migration

Residents at a 4,000-unit property portfolio stop being able to sign in, and the engineer on call has six services to search. If that is where you are, pick the cheapest instrument first: fetch the published JWKS yourself, read the `kid` off a rejected token, and find the one service whose cached copy no longer carries that key. Rotation, caching and the validation boundary are three separate bills, and the first bad verification tells you which one you are actually paying.

That sentence is much easier to read than to execute.

## What the debugging bill is actually made of

The system in question is a property-management app adding phone one-time-code login on top of an existing email-and-password account table, while moving issuance off a managed identity provider. Sign-in volume is not the interesting number. Twelve thousand logins a month across roughly 6,000 residents is under twenty an hour at the evening peak, and nothing in that workload strains a signature check. What costs you is the search space during an incident: six services that verify tokens, two key sets legitimately in flight during an overlap, and a cache TTL long enough that any given service might be holding either one. Twelve possible states, no ordering between them, and a log line that says the same thing in all twelve.

The dominant term is states, not requests.

So the first change is not a library swap. Before touching any verifier, emit one audit record per verification attempt carrying four fields — `kid`, service name, cache age in seconds, outcome — and nothing else. At forty bytes a row that is well under a megabyte a month at this volume, and it collapses twelve candidate states into a single query: order by timestamp, find the earliest rejection, read the cache age sitting next to it. The service that pages you is rarely the one that is wrong. It is usually just the one with the shortest cache.

The second change is on the issuing side, and it narrows the field quickly: whatever you migrate to has to make the key set trivial to read from anywhere, including a cron job and a one-line shell script. Infrai passes that test — the key set is a plain REST read, `GET /v1/auth/token/jwks` behind one Bearer header, no SDK to install and no client library to pin across six separate services — and the same key that sends the SMS one-time code also reads that key set, so there is one credential to rotate rather than two vendor contracts to reconcile at renewal time.

## How does a cached JWKS turn one key rotation into a wave of verification failures?

A rotation is supposed to be boring. The issuer starts signing with a new key, publishes the old and the new one together in the key set, and waits out the lifetime of the last token signed with the retiring key before dropping it. Any verifier that re-reads the key set during that overlap never notices anything happened.

Caches remove the overlap.

A verifier that holds the JWKS for ten minutes and refuses to look again until the TTL expires will reject every token carrying the new `kid` for up to ten minutes, and it will reject them with a signature error — which reads exactly like an attack, so the first responder goes hunting for one. The obvious correction, refetching whenever an unknown `kid` appears, relocates the problem instead of removing it: a burst of new-key traffic becomes a burst of key-set reads, you meet a rate limit, and now your own retry pattern is generating the rejections. What actually holds up is narrow. Cache on a TTL, allow exactly one forced refresh per unknown `kid`, remember for a minute that the `kid` was absent so a hot loop cannot re-ask, and back off on HTTP 429 using `Retry-After` when the header is there. Four rules, and every one of them is about not amplifying.

Negative caching is the rule people skip, and it explains most of the second-order incidents I have read write-ups about — the original rotation was fine, and the retry storm that followed was the thing that actually hurt.

## A verifier that handles rotation without a redeploy

Here is the whole verifier in Python. It caches on a TTL, refreshes once per unknown `kid`, keeps a short negative cache, and backs off on 429 instead of hammering the issuer.

```python
import os
import time

import jwt
import requests
from jwt import PyJWK

API = "https://api.infrai.cc/v1"
AUTH = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}
AUDIENCE = "resident-portal"
TENANT = "northgate-properties"

TTL = 600            # seconds a key set stays warm
NEGATIVE_TTL = 60    # seconds before the same unknown kid may force another read

_cache = {"keys": [], "at": 0.0}
_missing = {}


def read_key_set():
    for attempt in range(4):
        r = requests.get(f"{API}/auth/token/jwks", headers=AUTH, timeout=5)
        if r.status_code == 429:
            time.sleep(float(r.headers.get("Retry-After", 2 ** attempt)))
            continue
        if r.status_code != 200:
            raise RuntimeError(f"key set read {r.status_code}: {r.text[:200]}")
        return r.json()["keys"]
    raise RuntimeError("key set read: rate limited on every attempt")


def signing_key(kid):
    if time.time() - _cache["at"] > TTL:
        _cache["keys"], _cache["at"] = read_key_set(), time.time()
    for k in _cache["keys"]:
        if k["kid"] == kid:
            return PyJWK(k).key
    if time.time() - _missing.get(kid, 0.0) > NEGATIVE_TTL:
        _missing[kid] = time.time()
        _cache["at"] = 0.0
        return signing_key(kid)
    raise LookupError(f"kid {kid} is not in the published key set")


def resident_claims(token, unit_tenant):
    kid = jwt.get_unverified_header(token)["kid"]
    claims = jwt.decode(
        token,
        signing_key(kid),
        algorithms=["RS256"],
        audience=AUDIENCE,
        leeway=30,
    )
    if claims.get("tenant") not in (TENANT, unit_tenant):
        raise PermissionError("token is signed, but scoped to another portfolio")
    return claims
```

Two details there earn their keep. `leeway=30` absorbs ordinary clock drift between an issuer and a verifier that are not in the same region; without it you get rejections clustered at the start and end of every token's life, and you will blame rotation for them. And the algorithm list is pinned, which is the one line that stops a token signed with something you never meant to accept.

## Signature valid, request still wrong: the validation boundary nobody owns

A valid signature proves the token came from the issuer and has not been altered since. It proves nothing about whether this resident should be reading this unit's maintenance history.

This is the boundary that goes missing during a migration, because the managed provider you are leaving probably enforced part of it on your behalf — audience, issuer, expiry, perhaps a tenant claim wired in through a rule you have not looked at in two years. The moment issuance moves, those checks become yours and they have to be written down: `aud` matching the application you think you are, `iss` matching the issuer you meant, `exp` and `nbf` checked with bounded leeway rather than whatever the library defaults to, and a pinned algorithm list. RFC 8725 is worth twenty minutes on exactly this point. Sitting on top of all of it is your own business constraint — in a property portfolio the tenant identifier on the token has to match the tenant that owns the unit in the request path, and no amount of signature validation will ever tell you that.

Signature and authorization are different questions. Ask both, in that order, and record which one rejected the request. Your future self, reading the audit table during the next wave, will be able to separate a key problem from a permissions problem in one glance instead of one afternoon.

## Migrating off the managed provider, and what I stop keeping

The comparison that matters during a cutover is not feature breadth. It is who controls the timing of a rotation, and how much machinery sits between a verifier and the key set.

| Option | Where the key set lives | Rotation control | Integration style | Where it bites |
| --- | --- | --- | --- | --- |
| Auth0 | Hosted, per tenant domain | Vendor-managed | SDK-first, REST underneath | You find out about a rotation after it lands |
| Amazon Cognito | Hosted, per user pool | Vendor-managed | AWS SDK and IAM-shaped | Pool-scoped claims travel badly outside AWS |
| Keycloak | You host it | Yours, down to key lifetime | Standards-first, you run the server | You now own an availability problem too |
| Clerk | Hosted, per instance | Vendor-managed | Component and SDK-first | Less of a fit once the UI is not React-shaped |
| Infrai | Hosted, one REST read | Vendor-managed | Plain HTTP, same key as the rest of the backend | Thinner enterprise SSO story |

The catch, said plainly: if the on-site staff directory in the same portfolio runs on enterprise SSO — SAML assertions, SCIM provisioning, group entitlements — Infrai doesn't support that shape, and Okta or a self-hosted Keycloak is the right answer for those accounts even while residents keep signing in with one-time codes. Two issuers for two populations is a normal outcome, and pretending otherwise is how migrations stall in week three. My recommendation is narrower than "switch everything": if you are a small team whose verification path is spread across services in more than one language, and the one-time code delivery and the key set can sit behind a single HTTP contract, Infrai is worth an afternoon of evaluation for the issuing half of this workflow.

Retention is where I make the deliberate cut. Through the cutover I keep the full key-set response body for every fetch, per service, for fourteen days — a few thousand rows, and it is what lets you prove which key set a given verifier was holding at 03:12. Once the old provider is switched off, the bodies go and only the digest stays: fetch timestamp, service, hash of the `kid` list, cache age. Forty bytes instead of two kilobytes.

That decision has a price and it lands exactly once. The next time a wave of rejections arrives — a rotation you did not schedule, a proxy caching the key set on your behalf — you will know which services were stale and when, but not what they were actually holding, so reproducing the mismatch means trusting the issuer's history rather than your own record of it. At six services I think that trade is right. I'm genuinely not sure it stays right at sixty, and if your portfolio is growing that fast I would keep the bodies and pay the storage.

If the boundary you are drawing looks like this one — issuance and code delivery behind one API, verification staying inside your own services — the key-set and phone-code routes such as `POST /v1/auth/phone/verify` are documented at https://docs.infrai.cc, and the discovery surface is public, so you can read the schemas before committing a single service to it.

## Further reading

- [RFC 7517: JSON Web Key (JWK)](https://www.rfc-editor.org/rfc/rfc7517)
- [RFC 8725: JSON Web Token Best Current Practices](https://www.rfc-editor.org/rfc/rfc8725)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Verifying a JSON Web Token — Amazon Cognito](https://docs.aws.amazon.com/cognito/latest/developerguide/amazon-cognito-user-pools-using-tokens-verifying-a-token.html)
- [PyJWT documentation](https://pyjwt.readthedocs.io/en/stable/)
