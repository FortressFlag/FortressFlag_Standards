<!--
Keep the PR description honest and specific. State non-obvious tradeoffs and which pillar they
serve: Security → Compliance → Efficiency → Cost (Founding CLAUDE.md §2).
-->

## What & why

## Compliance & security review

<!--
This block is not paperwork. SOC 2 Type II is graded by sampling real changes and asking for
evidence that review happened. This repository is the canonical publication of the wire
contracts: its risks are exactly two — changing what every SDK must implement, and changing the
vectors those SDKs pin as tests. Both are backward-compatibility sacred (Founding §5, §8.3):
we cannot recall a shipped SDK.

Tick every box that applies — or tick the last one. A block with nothing ticked is an
unfinished PR, not a PR with nothing to declare: that is the whole point of the last box.
-->

- [ ] **Contract change** — changes what a contract document states an SDK or the backend must
      do. *Requires a backing ADR in `FortressFlag_Backend/docs/adr/` — cite it. Additive
      within a version; anything breaking is a new version served alongside the old.*
- [ ] **Test-vector change** — adds, removes, or alters an entry in `vectors/`. *A vector
      change is a wire-contract change, never a test fix. Cite the ADR.*
- [ ] **None of the above.** I checked, and this change touches neither.

<!-- For every box ticked above: name the ADR, and state the compatibility argument. -->

## Testing

<!-- What you checked — at minimum, that the `Check` job's assertions still hold. -->

## Tradeoffs

<!-- What this gives up, and why that is the right call. Delete if genuinely none. -->
