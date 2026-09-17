# Client contract v1

> **Owned by `FortressFlag_Backend`** (Founding CLAUDE.md §5). This is the canonical
> publication of what the client SDKs consume and the backend produces (ADR-0013); the SDK
> repos hold pointer stubs. Changes go through an ADR in `FortressFlag_Backend/docs/adr/` and
> must be backward compatible — **we cannot recall a shipped SDK.**
>
> **Contract v2 exists** ([`contract-v2.md`](contract-v2.md), backend ADR-0008): the multivariate
> value union, requested with `?v=2`. v1 stays served indefinitely and sees only boolean flags —
> the filter and its reasoning are in the v2 document. Everything below remains the v1 truth.

## Backend status per feature

The contract below describes the full v1 shape, and since backend ADR-0025 all of it is live end
to end. The table stays so the history of what shipped when is recorded in one place, and so the
rest of the document can describe the contract rather than hedge it:

| Feature | SDK | Backend |
|---|---|---|
| `GET /v1/client/flags` with key auth, device header, tags | ✅ Shipped | ✅ Shipped |
| `If-None-Match` / `304` conditional requests | ✅ Shipped — sends the header, handles `304` | ✅ Shipped (roadmap M3, environment versioning). The server answers an unchanged poll with `304` and no body; the ETag is a strong validator the SDK treats as opaque. |
| `sig` / envelope signing | ✅ Shipped (ADR-0025) — every SDK verifies **fail-closed** with the production key baked in (`SignaturePolicy.required` is the default and rejects an unsigned envelope; `.disabled` is the explicit opt-out for local dev against an unsigned backend) | ✅ Shipped (ADR-0025). Every `200` carries `sig`: pure Ed25519 over the exact payload bytes, one key per environment, held in AWS Secrets Manager and signed in-process. A local dev backend without `FF_SIGNING_*` still omits `sig`. See [Signing keys](#signing-keys). |

## Why this is not the management API

The dashboard reads the project-scoped flag routes
(`GET /v1/projects/{projectKey}/flags`), which return each flag's `key`, `name`, `description`,
`enabled`, `environment` and `updatedAt`. None of that can be served to an end user's device:

- `name` and `description` are internal prose. "Q3 checkout redesign, legal sign-off pending" is
  not something a customer wants on every phone that runs their app (Founding §2.1).
- It has no notion of a device, so per-device billing (§6.1) has nothing to count.
- It is authenticated by a session plus a tenant header that is a claim checked against the
  memberships table — a person's credential, not something an app can ship with.
- It is `Cache-Control: no-store`, which is right for tenant-scoped management data and wrong for
  a data plane whose entire cost model is CDN caching (§6).

So the client data plane is a separate contract, described here.

## Request

```
GET /v1/client/flags?environment=prod
Authorization: Bearer ffc_prod_<random>
X-FF-Device: dev_<22 chars base64url>     (sim_ from simulator builds — same body shape)
X-FF-SDK: ios/0.1.0
X-FF-Tags: <unpadded base64url of a JSON object>   (when the device has tags to report)
Accept: application/json
If-None-Match: "<etag>"        (when the SDK holds a cached payload)
```

### The SDK key is not a secret

It ships in every copy of the customer's app and `strings` recovers it. The design assumes it is
public:

- read-only, scoped to **one tenant and one environment**; it cannot reach the management API
- prefix `ffc_<env>_` so secret-scanning can classify it as a *client* key and not page anyone
- rate-limited server-side per key and per device
- revocable and rotatable without an app release — a revoked key makes the SDK fall back to its
  cache, not break

What a key-holder learns is the tenant's flag keys for one environment, which they could already
read out of the app binary. That is accepted, and stated rather than hand-waved.

### `X-FF-Tags`: the device's tags (M8, backend ADR-0004)

An **additive request field within v1** (see the versioning rules below): a server that ignores
it serves defaults, a pre-M8 SDK that never sends it keeps working forever. The header carries
the tags the server evaluates targeting rules against, and it changes nothing about the
response shape — rules never reach a device; the payload stays a boolean map.

The value is **unpadded base64url of a JSON object of string values**, keys sorted:

```json
{ "appBuild": "421", "appVersion": "2.1", "cohort": "beta", "osVersion": "26.0",
  "platform": "ios", "sdkVersion": "0.1.0" }
```

Base64url because tag values are free text and header values are not; one header keeps the
request a GET, mirroring `X-FF-Device`. **Sorted keys are required of SDKs**, not decoration:
the same tag set must produce the same bytes on every request, or the ETag design (M3) would
see a different request per launch for an unchanged device.

The SDK sends five **built-in tags** automatically — `appVersion`
(CFBundleShortVersionString), `appBuild` (CFBundleVersion), `osVersion`, `platform` (`ios`),
`sdkVersion` — all mechanically derived, none user-identifying. Their names are reserved:
customer tags that collide are dropped. A missing Info.plist value omits its tag. Customer
tags come from `Configuration.tags` at start and `FortressFlag.setTags(_:)` at runtime.

**Caps, enforced on both ends:** decoded document ≤ 4 KB; ≤ 32 tags; key 1–64 chars matching
`^[a-zA-Z0-9._-]+$`; value ≤ 256 bytes. The SDK drops violating entries with a logged warning
before sending, so a request the server sees over the caps came from a broken client — the
server answers **400**, and the SDK treats it like any other failed refresh: the cache keeps
answering. An **absent header means no tags**: only default states serve.

**Evaluation semantics** (the server's, stated here because SDKs document what customers
observe): rules are an ordered list per flag + environment, first match wins; a rule whose tag
key the request did not send is skipped; no match serves the flag's default. `eq`/`neq` are
exact string comparison and `contains` (ADR-0023) is substring match — all three
case-sensitive, untrimmed. The `semver_*` operators split on `.`, compare numeric components
left to right, read missing components as 0 (`2.0` == `2.0.0`), and a device tag that does
not parse (`2.0-beta`) makes the rule not match — never an error. No pre-release or
build-metadata support in v1. An operator an evaluator does not recognise makes its rule not
match — fail closed to the later rules and the default, which is what lets the set grow
additively.

**Tags are stateless by decision** (backend ADR-0004): evaluated for the duration of one
request, never persisted, and never logged — a custom tag value may carry anything the
customer's app puts in it. Nothing about a device's tags is stored anywhere on FortressFlag's
side, and the SDK exposes only the tag *keys* on its diagnostics surface.

### Reserved: `attestation`

v1 sends no attestation. The field name is reserved now so that App Attest can be added later
without a contract break — see the metering note in `FortressFlag_SDK_ios/docs/threat-model.md`.

## Response

`200 OK`, `Cache-Control: private, max-age=60`:

```json
{
  "payload": "eyJ2IjoxLCJ0ZW5hbnQiOiIxMTExMTExMS0…",
  "sig": "ed25519:prod-2026-09-k1:<unpadded base64url signature>"
}
```

`payload` is unpadded base64url of:

```json
{
  "v": 1,
  "tenant": "11111111-1111-1111-1111-111111111111",
  "environment": "prod",
  "device": "dev_8KqW3nR2vT7yLp0aZxQmBg",
  "issuedAt": "2026-07-28T10:00:00Z",
  "expiresAt": "2026-07-28T10:30:00Z",
  "flags": { "new-checkout": true, "dark-mode": false }
}
```

The `device` field echoes the identifier the request sent, verbatim — `sim_`-prefixed when a
simulator build sent one (device metering, M5): the server serves simulator identities flags
normally and excludes them from seat metering, so a customer's simulators are never billed.

Other statuses: `304` (ETag matched), `401`/`403` (key rejected or revoked), `429` with
`Retry-After` in delta-seconds, `5xx`. Error bodies reuse the backend's existing envelope
(`internal/httpx/respond.go`) so the SDK has one error shape to handle.

### Open tension: this payload is not shared-cacheable

Founding §6 leans on "cheap, globally-distributed reads" from a CDN, and §2.4 makes that the cost
argument for the whole product. A **per-device** payload cannot be shared between devices, so the
usual CDN win — one origin fetch serving many clients — does not apply here. `Cache-Control` has to
be `private`.

What the data plane actually buys us in v1 is therefore narrower, and worth stating so nobody
plans a budget on the wrong model:

- **`304` on ETag** — the common case once a device is warm. Cheap, but still a request.
- **TLS termination and connection reuse at the edge**, which is latency, not origin cost.
- **No per-evaluation server compute** — evaluation is local in the SDK, so origin cost scales with
  *poll rate × devices*, not with flag reads. That is the real §6 win and it holds.

The version that *does* get shared caching is a per-*segment* payload: the edge groups devices that
resolve identically and caches one artifact per group. That is a larger design and it changes the
contract, so it belongs in an ADR rather than being smuggled into v1. Recorded here so the
mismatch between this document and the founding cost model is a known open question, not something
someone discovers from a bill.

## Why a detached signature over opaque bytes

The obvious design is an inline JSON object with a `sig` field alongside the data. It requires the
Go signer and every SDK verifier to agree, byte for byte, on a canonical serialisation — key
ordering, number formatting, unicode escaping, how `+00:00` versus `Z` is emitted. Any divergence
shows up as a signature that fails in production on *some* payloads and not others, on devices we
cannot debug and cannot recall.

Signing the literal transmitted bytes removes that class of bug entirely. It also gives a rule
worth stating plainly:

> **Nothing lives outside the signature.** There is no field to read before verification, so there
> is no field that can be trusted before verification.

The cost is that a response is not readable in a terminal. Decode one with:

```sh
curl -s -H "Authorization: Bearer $KEY" -H "X-FF-Device: $DEVICE" \
  "$BASE/v1/client/flags?environment=dev" \
  | jq -r .payload | tr '_-' '/+' | base64 -d | jq
```

## What the signature protects, and what it does not

It stops **the network and the CDN** from lying to an honest device: an active MITM with a trusted
root installed, or a compromised edge node, can replace the bytes but cannot sign them.

It does **not** stop the device's owner. Anyone who can patch the app can patch the SDK out. Flag
values are not a security boundary — see `FortressFlag_SDK_ios/docs/threat-model.md`.

## Fields the SDK checks, and why each one

| Check | Failure means | Why it exists |
|---|---|---|
| `sig` verifies against a trusted key ID | `badSignature` / `unknownKeyID` | Integrity, independent of TLS |
| `v == 1` | `unsupportedContractVersion` | A future dialect must be refused, not guessed at |
| `environment` matches | `environmentMismatch` | A prod payload replayed at a dev build |
| `device` matches | `deviceMismatch` | Another device's values served to this one |
| `issuedAt` not far in the future | `issuedInTheFuture` | A pre-signed payload held back and released later |
| `expiresAt` not passed — **live responses only** | `expired` | Bounds the replay window |

Unknown *top-level* fields in the payload are ignored, so the server can add fields additively.
Unknown keys inside `flags` are simply flags this app build does not know about.

### A key disappearing from the payload

A payload names every flag the tenant currently has in the requested environment. A key that was
present in one response and absent from the next means the flag **no longer exists for devices** —
archived or purged in the control plane, not merely "unmentioned".

The SDK replaces both its live values and its durable cache with each accepted payload, so a
disappeared key stops resolving from either tier on the same poll: it resolves to the caller's
`default:` if one was supplied, otherwise `false`. With this endpoint's `Cache-Control:
private, max-age=60`, the practical statement is:

> **A key that leaves the payload turns off on every online device within about a minute, with no
> grace period.** The durable cache does not protect it — deliberately.

This is load-bearing product behaviour, not an accident of the cache design: archiving a flag in
the dashboard is how a customer turns a feature off *for good*, and a cache that "helpfully"
preserved removed keys would make deletion unable to end a rollout
(`FortressFlag_Backend/docs/adr/0003-flag-deletion-is-reversible.md`, option D, rejected). The
second consumer exists: `FortressFlag_SDK_android` implements the same overwrite semantics
(ADR-0013), pinned by its wholesale-overwrite tests — and so does the third,
`FortressFlag_SDK_web` (ADR-0014), whose client mirrors every accepted payload into both of
its `localStorage` tiers the same way.

### The expiry asymmetry

`expiresAt` is enforced on a live response and **deliberately not enforced when loading the
durable cache.**

Founding §8.4 makes the last recorded value the primary fallback. A device that has been offline
for a month still has to serve what it last saw. Expiring the cache would silently revert every
flag on that device to `false` — turning an outage into a feature regression, which is exactly the
failure the cascade exists to prevent.

So: **expiry governs freshness, not validity.**

## Key rotation

`sig` names a key ID. The SDK holds a set of trusted keys, so the backend can begin signing with
key *N+1* as soon as SDKs in the wild trust it — rotation is a publish, not an app release. Retire
a key only once telemetry shows no live SDK still needs it; a shipped SDK that trusts only the old
key will reject everything and fall back to cache until its app is updated.

## Signing keys

Decided in backend ADR-0025; the SDKs implement exactly this.

- **Algorithm: pure Ed25519 (RFC 8032), never Ed25519ph or any prehashed variant.** The
  signature is over the base64url-decoded `payload` bytes exactly as transmitted. The
  algorithm token in `sig` is the literal `ed25519`; a verifier rejects any other token as
  `unsupportedSignatureAlgorithm` without looking further.
- **`sig` is `algorithm:keyID:signature`, split at the first two colons.** The signature is
  unpadded base64url of the 64 raw signature bytes. A verifier that cannot decode it, or finds
  fewer than three parts, rejects as `malformedSignature`.
- **Key IDs are `<env>-<yyyy>-<mm>-k<n>`** — `prod-2026-09-k1`, `staging-2026-09-k1` —
  lowercase `[a-z0-9-]+` and **never a colon**: the two-colon split means the *signature*
  may contain extra colons (it cannot, being base64url, but the parse allows it) and the key
  ID never can. The backend refuses to load a key ID outside that grammar.
- **Public keys are raw 32 bytes**, supplied to a verifier as unpadded base64url (or the same
  bytes as a hex literal in a constant). No SPKI, no PEM: a verifier that needs a wrapped
  form (Node's `crypto.verify`, Java's `KeyFactory`) prepends the 12-byte SPKI prefix
  `302a300506032b6570032100` itself. A key in the trust store that is not exactly 32 bytes is
  treated as `unknownKeyId`, so one bad entry cannot disable a rotation set.
- **One key per environment.** `FORTRESSFLAG_PRODUCTION` (or the platform's spelling) holds
  the production key only; a build that targets staging passes the staging key explicitly.
  The current keys are published in exactly three places — the SDK constants, the customer
  docs page `concepts/payload-signing`, and ADR-0025 — and nowhere else: there is no
  key-discovery endpoint, because a fetch-and-trust endpoint would invert the pinned-key model
  above.
- **Vectors:** [`vectors/signing.json`](../vectors/signing.json) — two envelopes that must
  verify (client v2, server sv1) and four that must fail with a named code, all signed with the
  RFC 8032 §7.1 TEST 1 key pair, whose seed is published in the RFC and deliberately absent
  from the file. Every SDK ports it as a unit test and feeds each `envelope` to its verifier
  unchanged.
- **Platform notes.** Web verifies with WebCrypto (`crypto.subtle` `Ed25519`): Safari 17+,
  Chrome 137+, Firefox 130+, Node 22+. An insecure non-localhost origin has no `crypto.subtle`
  at all, and an older browser throws on import; both are `signatureUnverifiable` — a
  rejection that keeps the cache and serves defaults — never `badSignature`. Android and
  Python vendor a verify-only RFC 8032 implementation (no platform provider, one code path on
  every API level); Go, Node and Java use their platform crypto.

## Versioning rules

- Adding a payload field is backward compatible. Removing or retyping one is not.
- A breaking change means `v: 2`, served alongside `v: 1` for as long as v1 SDKs are in use.
- `flags` is `[String: Bool]` in v1. Multivariate values arrive as a value union under `v: 2`.
