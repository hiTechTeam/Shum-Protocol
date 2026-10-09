# 12. First Stable Version, v1 Stable

English · [Русский](../ru/12-v1-stable.md)

Owner decision, 9 October 2026. The protocol has not been released. Sections 01–09 describe iOS behavior as the **v1 draft**. The first stable version combines the draft, sections 10/11 and the corrections below.

## 1. What v1 stable means

A complete product on which later versions build:

- **All clients interoperate:** iOS, Android, macOS/Windows/Linux apps, browser and CLI.
- **One profile synchronized across its devices:** bidirectional QR linking, messages on all devices, read/clear/settings/relay synchronization, sections 10 and 11.
- **Independent profiles on one device**, subsection 6.
- **Extension rules** so later versions do not break v1, subsection 5.

Draft compatibility is not required; formats may change before release. Draft profiles must migrate without loss, subsection 8.

## 2. Scope

| Part | Section | Status |
| :--- | :--- | :--- |
| Keys/identity | 01 | Draft with vectors |
| Card/invitations | 02 | Draft; corrections in subsection 4 |
| Envelope/packets/rules | 03–05 | Draft; corrections in subsection 4 |
| Bluetooth/Nostr/push | 06–08 | Draft; corrections in subsection 4 |
| Local storage | 09 | Draft |
| Multiple devices | 10 | Decisions agreed, formats needed |
| Profile relays/networks | 11 | Decisions agreed, formats needed |
| Profile isolation | 12, subsection 6 | Needs specification |
| Extension rules | 12, subsection 5 | Needs specification |

## 3. One implementation

Implement the protocol once in Rust Shum-Core. Clients use the core rather than reimplementing the protocol.

| Client | Core integration |
| :--- | :--- |
| CLI | Direct Rust |
| iOS | UniFFI, Swift |
| Android | UniFFI, Kotlin |
| macOS/Windows/Linux desktop | Direct, Tauri |
| Browser | WebAssembly |

iOS migration to the core is part of v1 stable. Until then, Swift remains the vector source for sections 01–09.

## 4. Corrections before stable release

The draft deferred these matters to a future protocol version. Resolve them before release; each needs an owner decision.

| # | Issue | Reference | Proposal |
| ---: | :--- | :--- | :--- |
| 1 | Card signature lacks domain label | 02, question 6 | Add one like other signatures |
| 2 | Signed bytes depend on Apple's JSONEncoder | 02, question 2 | Specify an independent canonical format |
| 3 | Bio grapheme count depends on Unicode version | 02, question 3 | Count scalars or bytes |
| 4 | Private v2: encryption resembles NIP-44 | 07, question 1 | Adopt standard NIP-44 |
| 5 | Swift replay-window bug | 06, question 3 | Fix; core migration removes the Swift defect |
| 6 | Courier lacks forward secrecy; no key rotation | 03 question 1; 01 question 3 | Decide whether ratchet belongs in v1 stable or a later version |
| 7 | Unsigned forwarding counter | 03, question 2 | Decide whether signing is needed |
| 8 | Several invitation formats, c2/c3/c4/contact | 02 | Create only c4/HTTPS, read the rest |
| 9 | Clear on all devices | 04; 05 question 5 | Included in section 10 |
| 10 | Different safety codes across clients | 02, question 7 | One shared pair-code algorithm |

## 5. Extension rules

Specify before release, at minimum:

- Preserve unknown card/packet fields without breaking signature verification.
- Ignore unknown packet kinds without error.
- Exchange capabilities; enable a new feature only if both sides understand it.
- Every signed structure has a format version.

## 6. Profile isolation

Multiple profiles on one device must not reveal that they belong to the same person:

- Each has separate keys, relays and Bluetooth address; only the active profile advertises nearby.
- No shared identifiers in cards, packets or Nostr events.
- **Push:** APNs issues one token per app installation. Registering it for all profiles lets the push server link them to the same phone. Options: honestly document this limit, or register under separate one-time identifiers, which only partially hides the link. Owner decision required.

## 7. Open decisions

From sections 10 and 11:

1. Multi-device recipient encryption: profile key or per-device, 10 question 4.
2. Linking channel: Bluetooth or relay with QR key, 10 question 2.
3. Nearby behavior with multiple profile devices, 10 question 3.
4. Linking a camera-less computer, 10 question 1.
5. Profile Relays format/equal-time merge, 11 questions 1–2.
6. Push/profile isolation, subsection 6.

## 8. Draft-profile migration

If a draft build ships, migrate profiles to v1 stable without loss. Three draft keys become profile keys, section 10 option B; Shum ID stays unchanged; SQLite migrates under section 09. Until updated, a draft device may not understand new formats.

## 9. Readiness criteria

Verify between real clients, not tests alone:

1. Create a profile on any client and link the others in both directions.
2. Deliver a message to every profile device. Synchronize read state, local/all-device clearing, settings and relays.
3. An unlinked device receives no new messages.
4. Profiles on one device are not linked, subsection 6.
5. Nearby people see one card per profile.
6. Every client passes the same vectors/ test set.
7. A newer client does not break v1 conversations, subsection 5.
