# Security Policy

FortressFlag holds the switches that turn our customers' production behavior on and off. A
compromise of FortressFlag is a compromise of every customer that trusts us (Founding CLAUDE.md
§2.1). We would rather hear about a problem early and awkwardly than late and publicly.

## Reporting a vulnerability

**Use GitHub private vulnerability reporting:** open the repository's **Security** tab and choose
**Report a vulnerability**. This creates a private advisory that only maintainers can see, so a
report never sits in a public issue while it is still exploitable.

Please do not open a public issue, pull request, or discussion for a suspected vulnerability.

Helpful reports usually include: what you found, how to reproduce it, which component and version,
and what an attacker gets out of it. A rough report you send today beats a polished one you send
next month.

## What to expect

| Stage | Target |
|---|---|
| Acknowledgement that a human has read it | 3 business days |
| Initial assessment and severity | 10 business days |
| Fix or documented mitigation for critical/high findings | 30 days from assessment |

These are targets for a small team, stated so you know when to chase us rather than assume
silence means indifference. If a target slips, we will tell you where the work stands.

## Scope

This repository is the **canonical publication of the FortressFlag SDK wire contracts and
cross-SDK test vectors** — documents and JSON, no executable code. Findings we especially want
to hear about:

- **A contract statement that lets an SDK accept what it should refuse.** The binding checks,
  the signature grammar, the cache and expiry rules — anything an implementer could follow
  faithfully and end up insecure.
- **A vector that pins the wrong behaviour.** Every SDK ports `vectors/*.json` as unit tests; a
  wrong expected value propagates to every implementation at once.
- **Ambiguity two implementations could resolve differently** in a way that matters for
  security or billing (device identity, rollout bucketing).

One note specific to this repo:

- **We cannot recall a shipped SDK.** A contract mistake lives in every SDK built against it
  until each is fixed and each customer rebuilds, on their schedule. That makes findings here
  the longest-lived in the system, and worth reporting even when they look minor.

Other components live in their own repositories, each with this policy: the seven SDKs
(`FortressFlag_SDK_ios`, `FortressFlag_SDK_android`, `FortressFlag_SDK_web`,
`FortressFlag_SDK_go`, `FortressFlag_SDK_node`, `FortressFlag_SDK_python`,
`FortressFlag_SDK_java`), `FortressFlag_Backend` (control plane), `FortressFlag_Frontend`
(dashboard), `FortressFlag_Infra` (infrastructure).

## Safe harbour

If you make a good-faith effort to follow this policy, we will not pursue legal action against you
for your research. Good faith means: you do not access, modify, or retain data belonging to anyone
but yourself; you do not degrade service for others; you stop when you have proven the issue rather
than exploring how far it goes; and you give us a reasonable chance to fix it before disclosing.

## Disclosure

We will credit reporters who want credit, and coordinate timing on a public advisory once a fix is
available. If a finding affects customer data, our obligations under GDPR — including the 72-hour
notification window for a personal-data breach — take precedence over any disclosure timeline
agreed here.
