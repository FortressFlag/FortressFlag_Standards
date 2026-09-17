# Server contract v1

> **Owned by `FortressFlag_Backend`** (Founding CLAUDE.md §5). This is the canonical
> publication of the SERVER data plane (backend ADR-0015): the endpoint a customer's own
> backend calls to download the full evaluable ruleset for one project + environment and
> evaluate flags locally, in-process. Changes go through an ADR in
> `FortressFlag_Backend/docs/adr/` and must be backward compatible — **we cannot recall a
> shipped SDK**, and server SDKs deployed in customers' fleets are no easier to recall than
> apps.
>
> This contract versions **independently** of the client contract: it is requested with
> `?sv=` (server version), its payload echoes `sv`, and nothing about
> [`contract-v1.md`](contract-v1.md)/[`contract-v2.md`](contract-v2.md) changed when it was
> introduced. The Go, Node, Python and Java server SDKs
> (ADR-0016, ADR-0020) are built against this document.

## Backend status per feature

| Feature | Backend |
|---|---|
| `GET /v1/server/ruleset` with server-key auth | ✅ Shipped (ADR-0015) |
| `If-None-Match` / `304` conditional requests | ✅ Shipped — the M3 validator with an `sv1` suffix |
| `sig` / envelope signing | ✅ Shipped (ADR-0025) — the same pure-Ed25519 detached signature as the client plane, over the ruleset's exact payload bytes; the size question that blocked the algorithm decision was settled by signing in-process rather than through KMS. A local dev backend without `FF_SIGNING_*` still omits `sig`, and a signature-requiring consumer fails closed exactly as the client SDKs do. |

## Request

```
GET /v1/server/ruleset?sv=1
Authorization: Bearer ffs_prod_<random>
Accept: application/json
If-None-Match: "<etag>"        (when the SDK holds a cached ruleset)
```

`sv` absent means 1. A version the server does not speak is a `400` — the SDK treats it as a
failed fetch and keeps serving its cached ruleset, never guesses at a payload whose meaning may
have changed.

There is no device header and no tags header: this plane has no devices. The key is the entire
request identity.

### The server key IS a secret

This is the **inversion** of contract-v1's "[the SDK key is not a
secret](contract-v1.md#the-sdk-key-is-not-a-secret)" section, and it deserves the same
prominence. An `ffc_` client key ships inside every copy of a customer's app, so its design
assumes it is public. An `ffs_` server key lives in a customer's server configuration and can
download the full ruleset — **targeting rules included** — for its project + environment. Treat
it exactly like a database password:

- store it in a secret manager or an environment variable, never in client-side code, never in
  a repository, never in a log
- prefix `ffs_<env>_` so secret-scanning classifies a leak as a **genuine secret that should
  page somebody** — the `ffc_` rationale with the outcome inverted, and the reason the two
  classes share no prefix
- scoped to **one tenant, one project, one environment**; read-only; it cannot reach the
  management API, and it cannot be used on the client data plane (nor a client key here — the
  planes reject each other's credentials)
- rate-limited server-side per key (an abuse ceiling, not a usage limit)
- revocable without a deploy: revocation takes effect on the key's next poll. A revoked key's
  `401` is answered like any failed fetch — the SDK keeps evaluating with the last ruleset it
  downloaded, indefinitely, until it is given a new key

Rotation is the sdk-key discipline: several live keys per environment are legal, so issue the
new key, roll it out, revoke the old one.

### What this endpoint discloses, and to whom

Rule conditions — tag keys, operators, condition values — leave FortressFlag and reach every
server holding the key. That is the point of the plane (local evaluation needs the rules), and
it is a **different trust boundary** than an end user's device: the customer's own
infrastructure, behind a secret credential the customer minted (backend ADR-0004 — devices
never see rules — is unchanged and still enforced on the client plane). The corollary is the
customer's to hear: **do not put secrets or end-user personal data in rule conditions.**
Anything a rule contains is readable by every process holding the key — which Founding §2.1
already required back when the only reader was FortressFlag itself.

What still never leaves, on either plane: flag names, flag descriptions, variant names, segment
names, and management timestamps. The export carries keys, kinds, defaults and rules — nothing
a dashboard renders as prose.

## Response

`200`, `application/json`. The **client envelope design, reused**:

```json
{
  "payload": "<unpadded base64url of the JSON document below>",
  "sig": "ed25519:prod-2026-09-k1:<unpadded base64url signature>"
}
```

`sig` is a detached signature over the payload's exact bytes, for contract-v1's canonical-JSON
reasons ([why a detached signature](contract-v1.md#why-a-detached-signature-over-opaque-bytes)),
with the same algorithm, key grammar and trust-store rules
([signing keys](contract-v1.md#signing-keys)). Only a local development backend with no signing
key configured omits it.

The decoded payload:

```json
{
  "sv": 1,
  "tenant": "00000000-0000-4000-8000-000000000001",
  "project": "default",
  "environment": "dev",
  "issuedAt": "2026-08-21T10:00:00Z",
  "expiresAt": "2026-08-21T10:30:00Z",
  "flags": {
    "dark-mode": {
      "kind": "boolean",
      "default": false,
      "rules": [
        {
          "conditions": [
            { "tagKey": "cohort", "operator": "eq", "value": "beta" }
          ],
          "serve": true,
          "rolloutPercentage": 50
        }
      ]
    },
    "checkout-cta": {
      "kind": "string",
      "default": "buy-now",
      "rules": [
        { "conditions": [], "serve": "treatment-value" }
      ]
    }
  }
}
```

Field by field:

- **`sv`** echoes the served contract version.
- **`tenant`**, **`project`**, **`environment`** name the scope the KEY resolved to — never
  anything the request asked for. A key cannot choose its scope.
- **`issuedAt` / `expiresAt`** — thirty minutes, and expiry governs **freshness, not
  validity**: the client contract's asymmetry
  ([the expiry asymmetry](contract-v1.md#the-expiry-asymmetry)) applies verbatim. An SDK must
  refuse an expired payload arriving on a live response (staleness where fresh data was
  expected) and must NOT refuse its own durable cache for being old — a service that restarts
  offline keeps evaluating with what it last saw.
- **`flags`** — one entry per live flag in the project + environment. Absence means the flag
  does not exist (or was archived): evaluate it as the SDK's compiled-in default.
- **`kind`** — `"boolean"`, `"string"` or `"number"`. Immutable per flag (ADR-0008).
- **`default`** — the value served when no rule matches: the on/off boolean for boolean kinds;
  the default **variant's value** for multivariate kinds. **Omitted** when a multivariate flag
  has no default variant chosen — no rule match then means the SDK's compiled-in default, the
  fail-safe.
- **`rules`** — the ordered targeting rules, always present (empty list for a ruleless flag).
- **`conditions`** — the ANDed tests, always present. An **empty list holds vacuously**: the
  rule always matches (the terminal "everyone else" shape).
- **`serve`** — what a match serves, following the kind: a boolean, or the served variant's
  **value** (never its name).
- **`rolloutPercentage`** — omitted for none, else 0–100: the rule matches only contexts whose
  bucket (below) is under it.

### Evaluation semantics — by reference

A server SDK must evaluate **exactly** as the backend evaluates for devices, or the same user
would see different cohorts depending on which component answered. The semantics are already
published and are incorporated here by reference, not restated (one prose copy per fact):

- **Rule walking**: contract-v1's
  [evaluation semantics](contract-v1.md#x-ff-tags-the-devices-tags-m8-backend-adr-0004) —
  ordered list, first match wins, a condition whose tag key is absent from the evaluation
  context does not hold, no match serves the default, and the `semver_*` operators' exact
  parsing rules (dotted numerics, missing components read as 0, an unparseable context value
  makes the condition not hold — never an error, and an unknown operator fails closed to the
  default). These semantics are pinned by machine-readable cases in
  [`vectors/evaluation.json`](../vectors/evaluation.json), ported as unit tests by every
  evaluating SDK (backend ADR-0016).
- **Bucketing**: contract-v2's
  [published bucketing](contract-v2.md#percentage-rollout-bucketing-published-for-cross-sdk-use)
  — `uint64_be(SHA-256(id + ":" + flagKey)[0..8]) mod 100`, matching when `bucket < p`, with
  the pinned vectors in [`vectors/buckets.json`](../vectors/buckets.json) ported as unit
  tests. **The bucket input is the context identifier string the evaluating SDK was given**:
  on a device that is the device ID; server-side it is whatever context key the customer's
  code passes for the evaluation (a user id, a session id — the customer's choice, made
  consistently). Identical bytes in, identical bucket out, whoever computes it.

### Segments are flattened — deliberately absent from this contract

The management model has segments (named, reusable AND-lists). The export does not: a rule
referencing a segment arrives with the segment's conditions **inlined into the rule's own
condition list**, in position order. This is semantics-preserving — a rule is an AND, a segment
is an AND, and AND(a, AND(b, c)) = AND(a, b, c) — so a server SDK implements conditions once
and never learns segments exist. Segment names never appear. (Backend ADR-0015 records the
rejection of structured segment export: contract surface and reference-resolution work in every
SDK, for zero behavioural difference.)

### Size and shape bounds

The backend bounds what the management API accepts, so a consumer may rely on: at most **20
rules per flag + environment** and **5 condition slots per rule** — but each slot may have
been a segment reference holding up to **10 conditions of its own**, so a flattened rule as
exported here carries at most **50 conditions**. Tag keys are ≤ 64 chars of
`[a-zA-Z0-9._-]`, condition values ≤ 256 chars, flag keys ≤ 64 chars of lowercase
`[a-z0-9-]`. Payload size therefore scales with the customer's own flag count; there is no
server-imposed flag-count cap in v1.

### Conditional requests

The `200` carries a strong `ETag` (opaque to the SDK), `Cache-Control: private, max-age=60`
and `Vary: Authorization`. Replaying with `If-None-Match` answers **`304`** with the same
headers and no body when nothing changed — the steady state of a polling fleet. Any
payload-changing management write (toggle, rules, variants, schedule falling due) changes the
validator; the next poll is a full `200`. The polling cadence is the SDK's; the client
contract's default (300 s, floor 30 s) is a sane starting point for a per-process poller.

`private`: the payload is scoped to one key's project + environment and must not be served
from a shared cache to a different key's holder.

## Versioning rules

Additive changes (new optional payload fields, new operators announced by kind) ride `sv=1`.
Anything a shipped consumer would misread is `sv=2`, served alongside `sv=1` indefinitely —
the client contract's rule, applied to the second plane. The `ETag` embeds the server contract
version, so an SDK upgrading its `sv` naturally refetches in full.
