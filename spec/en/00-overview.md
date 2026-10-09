# 00. Overview

English · [Русский](../ru/00-overview.md)

v1 draft: the protocol as implemented by the current iOS app. The protocol has not been released. The first stable version is described in [section 12](12-v1-stable.md).

## Shum from the protocol's perspective

Shum is a messenger without phone numbers or a central messaging server. Each person has an **identity**: keys created on their device. People exchange **contact cards** through QR codes, links or nearby Bluetooth, then communicate using **envelopes** encrypted for the recipient.

One of two **transports** delivers an envelope:

| Transport | When | Contents |
| :--- | :--- | :--- |
| [Bluetooth LE](06-transport-bluetooth.md) | Nearby, without internet | Bitchat mesh: direct connections, Noise sessions, forwarding through other peers |
| [Nostr](07-transport-nostr.md) | Internet is available | Events on public relays, with encrypted contents |

A **[push server](08-push-api.md)** additionally wakes the recipient's app when an event is waiting on a relay. It cannot see the conversation.

## Entities

| Entity | Section | Summary |
| :--- | :--- | :--- |
| Identity | [01](01-keys-identity.md) | Three key pairs: Noise (X25519), signing (Ed25519), Nostr (secp256k1) |
| Shum ID | [01](01-keys-identity.md) | SHA-256 of the Noise key, 64 hex characters |
| Contact card | [02](02-contact-card.md) | Public keys, name, bio, pixel avatar seed and signatures |
| Invitation | [02](02-contact-card.md) | A card packed into a `shum://c4/...` or `https://.../invite#c4/...` link |
| Envelope | [03](03-envelope.md) | A message sealed to the recipient's permanent Noise key |
| Packet | [04](04-packets.md) | Payload type: message, receipt, invitation, typing, presence, profile, retraction or reaction |
| Rules | [05](05-rules.md) | Admission, deduplication and conflict resolution |

## Profile devices

The v1 draft, as currently implemented on iOS, supports **one writer device** per identity. Concurrent use from several devices is unsupported (`docs/seed-profile-protocol.md`).

In v1 stable, a profile works across multiple devices. Each device has its own key; the profile confirms its devices; devices are peers; QR linking works in both directions. See sections [10](10-devices.md), [11](11-relays.md) and [12](12-v1-stable.md).

## Code origins

Bluetooth mesh, Noise sessions, courier wrapping and packet models originate from **Bitchat** (Unlicense) and were adapted for Shum. The Bitchat revision is recorded in `Shum-iOS/Upstreams/versions.json`. Even where behavior matches Bitchat, this specification describes it fully so a compatible client does not depend on another repository.

## Conventions

- Byte order is specified explicitly for every multibyte integer in binary formats: some use little-endian, others big-endian.
- `hex` means lowercase hexadecimal without an `0x` prefix.
- `base64` means the standard alphabet with `=` padding. `base64url` uses `-` and `_` instead of `+` and `/`, **without** padding.
- Text is UTF-8 without Unicode normalization. Received bytes are preserved as received.
- Strings ending in a zero byte, such as `"shum.profile.v1\0"`, are **domain labels**. They prefix signed data so a signature for one purpose cannot be presented as a signature for another.

## Versions

| Item | Defined by | Current value |
| :--- | :--- | :--- |
| Card version | Card's `version` field | `1`; other values are rejected |
| Invitation format | First format byte | `1`, `3`, `4`; see [section 02](02-contact-card.md) |
| Pixel avatar generator | `avatarVersion` field | `1` |
| Domain labels | `.v1` suffix | `v1` |

The rule that new fields must not break older clients is part of v1 stable: [section 12, item 5](12-v1-stable.md#5-extension-rules).
