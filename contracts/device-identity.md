# The cross-platform device-identity contract

> **Owned by `FortressFlag_Backend`** (Founding CLAUDE.md §5, §6.1); changes go through an ADR in
> `FortressFlag_Backend/docs/adr/`. This is the canonical statement of the identity every client
> SDK mints — the billing primitive (§6.1: one device = one seat) and a GDPR pseudonymous
> identifier at the same time. Promoted here from `FortressFlag_SDK_ios/CLAUDE.md` by ADR-0013,
> when Android became the second implementer and ratified what §12 of the founding document had
> held as a proposal. The Web (browser) column was added by ADR-0014, when the third SDK made
> "vendor-scoped secure storage" a phrase a platform could not honour and the table had to say
> so honestly.

## The identifier

**`dev_` + unpadded base64url of 16 bytes of CSPRNG output** — or **`sim_`** with the identical
body when the build targets a simulator or emulator. 128 bits of randomness, 22 base64url
characters, no padding. Random, never derived.

- `dev_` — a physical device; billable.
- `sim_` — a simulator/emulator; the server serves it flags normally and **excludes it from
  seat metering** (a customer's simulators are never billed), tallying sim traffic server-side
  to spot builds that lie. Simulator detection is **compile-time-exact on iOS**
  (`#if targetEnvironment(simulator)`) and **best-effort heuristics on Android** (a wrong
  heuristic is a billing skew, not an outage). On the web, `sim_` is minted when
  **`navigator.webdriver === true`** (ADR-0014) — the browser's own declaration that it is
  driven by automation (Playwright, Selenium, Puppeteer, Cypress), so a customer's E2E suite
  is served but never billed; best-effort like Android, since stealth plugins that hide the
  flag bill automation as `dev_`, a skew in our favour.

**Validation accepts BOTH prefixes regardless of which prefix the running build mints.** A
device that stored a `dev_` id and later runs in a simulator keeps it: identity stability wins
over prefix purity. Anything else — wrong prefix, wrong length, non-base64url, empty body — is
treated as absent rather than trusted: a corrupt or foreign value must not be adopted, because
sending a malformed identifier upstream fails every request and adopting one puts junk in the
billing path. The accept/reject table is machine-readable in
[`../vectors/device-ids.json`](../vectors/device-ids.json); every SDK ports it as a unit test.

## Forbidden derivations, and why

Not IDFV, not IDFA, not `ANDROID_ID`, not a serial number, not any hardware identifier or
end-user PII — and specifically **not a hash of any of those**, which is the tempting wrong
answer: the input space is small enough to enumerate, so the "one-way" function is reversible
in practice and the result is not a pseudonym at all. Random has no preimage.

## Storage semantics

Vendor-scoped secure storage, **never synced across devices** — an identity riding a cloud
backup onto a second physical device breaks "one device, one seat" *and* silently turns a
per-device pseudonym into a cross-device one. The identity must be readable before first
unlock's full protection would allow (SDKs resolve flags during background launches).

| | iOS (as shipped) | Android (per ADR-0013) | Web (per ADR-0014) |
|---|---|---|---|
| Mechanism | Keychain, `kSecClassGenericPassword`, service `com.fortressflag.sdk.device`, account `v1` | App-private storage encrypted with a Keystore-held AES key | `localStorage`, key `fortressflag.device.v1`, value the bare identifier |
| Availability class | `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly` | `setUserAuthenticationRequired(false)` on the Keystore key (the `AfterFirstUnlock` analogue) | Whenever the page runs (browsers have no first-unlock concept) |
| Sync | `kSecAttrSynchronizable = false` | Excluded from auto-backup (`android:allowBackup` rules) | Origin-scoped by the browser; never leaves the origin |
| Cross-app sharing | Optional shared keychain access group (`com.acme.fortressflag`), resolved at runtime | Per-app in v0.1; app-family sharing (signature-gated ContentProvider, automatic discovery) is a designed follow-up | n/a — the origin IS the scope |
| Uninstall survival | Keychain persists across uninstall | Does **not** survive uninstall (Keystore-backed app data is wiped); reinstall = new identity = a new billable device — a platform fact, documented rather than papered over | User clears site data / a private window closes = new identity — the platform's analog of uninstall, documented rather than papered over |

### Browser honesty notes (ADR-0014)

Browsers have no vendor-scoped secure storage, so the Web row states the mechanism honestly
rather than pretending. The identity is **JS-readable on the page** — it is pseudonymous, not
secret, the same class as the SDK key that ships beside it in every page source. Origin
scoping is privacy-positive (no cross-site linkage by construction) and billing-relevant at
once: one person on a customer's website, iOS app and Android app is up to **three** billable
devices, one per platform. **Private/incognito windows mint throwaway identities** that each
count as a device for the session, and a user clearing site data is a new device — platform
consequences, priced in and stated rather than hidden.

## Concurrency: mint-then-adopt-on-conflict

Two of a customer's apps (or two racing threads) launching together will both find no identity
and both mint. The loser must **adopt the winner's value, never overwrite** — otherwise one
device holds two identities and bills as two, permanently. On iOS this is add-then-read on
`errSecDuplicateItem`; on Android, file-lock/atomic-write semantics with the same outcome: all
participants converge on one identity.

## Reset

Every SDK exposes a reset (`resetIdentity()`): delete and re-mint. This is the GDPR erasure
path for the pseudonymous identifier, and its documented consequence is honest: this device
will be counted as a new one.
