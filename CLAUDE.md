# FortressFlag_Standards — Agent & Contributor Guide

> **This repo inherits the FortressFlag founding principles.** The canonical, source-of-truth
> document lives in the backend repo. Read it before making architectural or design decisions:
>
> - GitHub: <https://github.com/FortressFlag/FortressFlag_Backend/blob/development/CLAUDE.md>
> - Local clone: `~/Workspace/FortressFlag_Backend/CLAUDE.md`
>
> When anything here conflicts with the founding document, the founding document wins.
> Priority order when in doubt: **Security → Compliance → Efficiency → Cost.**

## This repo

The **canonical publication of the FortressFlag SDK wire contracts and cross-SDK test vectors**
(founding §5.2's promotion rule, executed by ADR-0013 when Android became the second SDK):

- `contracts/contract-v1.md`, `contracts/contract-v2.md` — the client data-plane contract every
  SDK implements and the backend serves.
- `contracts/device-identity.md` — the cross-platform device-identity contract (the billing
  primitive, founding §6.1).
- `vectors/*.json` — machine-readable test vectors every SDK ports as unit tests: the device-ID
  accept/reject table and the pinned rollout-bucketing vectors.

## Who owns changes

**`FortressFlag_Backend` owns the contract; this repo is its publication.** A contract change
starts as an ADR in `FortressFlag_Backend/docs/adr/`, and only then lands here. Editing a
contract or a vector in this repo without a backing ADR is the failure mode this sentence
exists to prevent — a vector change is a wire-contract change, never a test fix.

**Docs and vectors only — no implementation code lives here.** SDKs implement the contract in
their own repos; the backend serves it from its.

## Workflow

- Default branch: `development`. Changes go via PR with review (founding §7.5).
- **Commits and PRs are authored as FortressFlag, never a personal identity.** Local commits
  carry `FortressFlag <noreply@fortressflag.com>` (the `~/Workspace/FortressFlag_*` gitconfig
  include); PRs are opened and merged via the `fortressflag` GitHub App, because GitHub
  authors a squash commit as the PR opener's account regardless of branch authorship.
- Backward compatibility on the published contract is **sacred** — we cannot recall a shipped
  SDK (founding §5, §8.3). Additive changes within a version; anything breaking is a new
  version served alongside the old.
