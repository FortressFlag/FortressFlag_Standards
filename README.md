# FortressFlag_Standards

Canonical publication of the FortressFlag SDK wire contracts and cross-SDK test vectors.

| What | Where |
|---|---|
| Client data-plane contract, v1 | [`contracts/contract-v1.md`](contracts/contract-v1.md) |
| Client data-plane contract, v2 (value union delta) | [`contracts/contract-v2.md`](contracts/contract-v2.md) |
| Server data-plane contract, v1 (ruleset export — ADR-0015) | [`contracts/server-contract-v1.md`](contracts/server-contract-v1.md) |
| Cross-platform device-identity contract | [`contracts/device-identity.md`](contracts/device-identity.md) |
| Device-ID accept/reject vectors | [`vectors/device-ids.json`](vectors/device-ids.json) |
| Rollout-bucketing vectors | [`vectors/buckets.json`](vectors/buckets.json) |
| Evaluation-semantics vectors (ADR-0016) | [`vectors/evaluation.json`](vectors/evaluation.json) |
| Envelope-signature vectors (ADR-0025) | [`vectors/signing.json`](vectors/signing.json) |

**Ownership:** the contract is owned by
[`FortressFlag_Backend`](https://github.com/FortressFlag/FortressFlag_Backend) — changes go
through an ADR there (see ADR-0013 for this repo's creation) and are published here. SDK repos
(`FortressFlag_SDK_ios`, `FortressFlag_SDK_android`, `FortressFlag_SDK_web`,
`FortressFlag_SDK_go`, `FortressFlag_SDK_node`, `FortressFlag_SDK_python`,
`FortressFlag_SDK_java`) implement what this repo states and port the
vectors as unit tests.

Docs and vectors only — no implementation code lives here.

## Implementation status

| Consumer | Status |
|---|---|
| `FortressFlag_SDK_ios` | Shipped |
| `FortressFlag_SDK_android` | Shipped (v0.1.0, ADR-0013) |
| `FortressFlag_SDK_web` | Shipped (v0.1.0, ADR-0014) |
| `FortressFlag_SDK_go` | Shipped (v0.1.0, ADR-0016) |
| `FortressFlag_SDK_node` | Shipped (v0.1.0, ADR-0020) |
| `FortressFlag_SDK_python` | Shipped (v0.1.0, ADR-0020) |
| `FortressFlag_SDK_java` | Shipped (v0.1.0, ADR-0020) |
