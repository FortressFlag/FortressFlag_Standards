# Client contract v2 — the value union

> **Owned by `FortressFlag_Backend`** (Founding CLAUDE.md §5), decided in its
> `docs/adr/0008-multivariate-flags-rollouts-and-segments.md`; canonically published here since
> ADR-0013. This document is a DELTA over
> [`contract-v1.md`](contract-v1.md): everything v1 specifies — transport, headers, the SDK key,
> tags, the envelope, signing, the binding checks, cache semantics — holds unchanged unless this
> document says otherwise. v1 remains served for as long as v1 SDKs exist; **we cannot recall a
> shipped SDK.**

## What v2 is

Flags have an immutable **kind** — `boolean`, `string` or `number` — and multivariate kinds
(string, number) serve named **variants**. v2 is how those values reach a device:

- The request gains `?v=2`. An absent `v` (or `v=1`) is a v1 request, forever.
- The payload's `flags` becomes a **value union**: each value is a bare JSON boolean, string or
  number — the flag's kind. Never an object, never an array, never null (backend ADR-0008:
  evaluated payloads are readable by anyone holding the customer's binary, and scalars are the
  whole vocabulary).
- The payload's `v` echoes the version **that was served**, and the SDK verifies it as before.

```
GET /v1/client/flags?environment=prod&v=2
```

```json
{ "v": 2, "tenant": "…", "environment": "prod", "device": "dev_…",
  "issuedAt": "…", "expiresAt": "…",
  "flags": { "dark-mode": true, "checkout-cta": "buy-now", "retry-limit": 3 } }
```

Variant **names** (`control`, `treatment`) are management vocabulary and never appear in a
payload — only the value a variant names. The `device` field echoes what was sent, exactly as
in v1 — including a `sim_`-prefixed simulator identity (device metering, M5: served flags,
never billed).

## What v1 clients see

A v1 request's payload **omits every non-boolean flag**. That filter is the compatibility
mechanism: a string value inside a `[String: Bool]` payload would be a decode failure on every
installed v1 SDK — fail-closed to cache, silently stale forever after. Omission is the
fail-safe instead: v1's own cache semantics already treat an absent key as gone, so a v1 client
simply never learns a multivariate flag exists and resolves its compiled-in default. Boolean
flags are byte-identical under both versions.

A `v` the server does not know answers `400`; the SDK treats it as a failed fetch and serves
its cache.

## A flag with nothing to serve is omitted

A multivariate flag whose environment has **no default variant chosen** and whose rules did not
match is absent from the v2 payload — not `null`, not a guessed value. The SDK resolves its
developer default (Founding §8.4), which is exactly what a brand-new flag should do until
someone decides what it serves.

## SDK surface

- `isEnabled(_:default:)` keeps its exact v1 behaviour for boolean flags. Reading a string or
  number flag through it answers the caller's default (else `false`) — a kind projection, never
  a coercion and never an error.
- `stringValue(_:default:)` and `numberValue(_:default:)` are the multivariate reads, with the
  identical fallback cascade: last received value, then the durable cache, then the default.
  Numbers are float64 on the wire.
- `allFlags()` / `Resolution.value` carry the union (`FlagValue`).

## Upgrade compatibility — the cache

The durable cache stores the raw envelope, and the first launch after an SDK upgrade reads a
**v1** file. The SDK therefore accepts payload versions `1...2` (`supportedContractVersions`):
a v1 payload's booleans decode into the union unchanged, so no updating device ever boots into
the `false` fallback because its cache spoke the old dialect. Pinned by
`PublicAPITests.v1CacheLoadsAfterUpgrade`.

## Percentage-rollout bucketing (published for cross-SDK use)

Rollouts (backend ADR-0008) gate a rule on a device's **bucket**, computed as a pure function
of values already in the request and **persisting nothing** (the roadmap-M7 privacy
constraint — no record of which devices are in a rollout may exist):

```
bucket = uint64_be(SHA-256(deviceID + ":" + flagKey)[0..8]) mod 100
```

— that is: UTF-8 bytes of the device identifier, one ASCII colon, UTF-8 bytes of the flag key;
SHA-256 over those bytes; the first 8 bytes of the digest read as a big-endian unsigned 64-bit
integer; modulo 100. A rule with `rollout percentage p` matches only when `bucket < p`.

Today the **server** computes this — devices never see rules (backend ADR-0004) — and the
algorithm is published here because it is a contract, not an implementation detail: a future
server-side SDK evaluating locally must produce the identical bucket for the identical
`(deviceID, flagKey)`, or a device would flip cohorts depending on which component evaluated
it. The function is deliberately per-flag (the `:` + flag key suffix): one device is in
different cohorts for different flags, so a 10% rollout is not always the same unlucky 10% of
devices.

## Versioning rules

v1's rules hold: additive payload fields are compatible within a version; removing or retyping
one is a new version served alongside the old. `v: 3`, when it comes, will follow this
document's own precedent.
