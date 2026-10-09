# 02. Contact Cards and Invitations

English · [Русский](../ru/02-contact-card.md)

Draft. Sources: `ShumIdentityService.swift` (`ShumContactCard`, `ShumContactLocator`, `ShumInvitationPayload`, `ShumCoding`, `ShumIdentityService.card(...)`), `docs/seed-profile-protocol.md`.

A contact card is signed public information about a person: keys, name, bio and pixel avatar. It contains no private data.

## 1. Fields

| JSON field | Type | Required | Constraints |
| :--- | :--- | :--- | :--- |
| `version` | Integer | Yes | Exactly `1` |
| `noiseKey` | Bytes, JSON base64 | Yes | 32 bytes, not all zero; public Noise key |
| `signingKey` | Bytes, JSON base64 | Yes | 32 bytes; public Ed25519 key |
| `nostrKey` | String | Yes | 64 lowercase hex characters; x-only public Nostr key |
| `name` | String | Yes | Nonempty after trimming whitespace and newlines; at most 64 UTF-8 bytes |
| `bio` | String | Yes | At most 72 characters (question 3) and 640 UTF-8 bytes; may be empty |
| `signature` | Bytes, JSON base64 | Yes, may be empty | Card signature, 64 bytes or empty |
| `avatarSeed` | Unsigned 64-bit integer | No | Pixel avatar seed |
| `avatarVersion` | Integer | No | Avatar generator version, currently `1` |
| `avatarSeedSignature` | Bytes, JSON base64 | No | Seed signature, 64 bytes |
| `profileRevision` | Unsigned 64-bit integer | No | Profile revision, greater than zero |
| `profileSignature` | Bytes, JSON base64 | No | Profile signature, 64 bytes |

Derived values **not** stored in the card:

- `id`: Shum ID, `hex(SHA-256(noiseKey))`, section 01.
- `peerID`: `hex(noiseKey)`, section 01.
- `profileID`: `hex(SHA-256(profile_bytes))`, below.

## 2. Canonical JSON

All three signatures use a JSON representation of the card. Verification requires byte-for-byte identical JSON, making this format mandatory.

**Current iOS behavior:** Apple's `JSONEncoder`, configured with `.sortedKeys` and `.withoutEscapingSlashes` by `ShumCoding.encode`, produces:

1. An object without spaces or newlines.
2. Keys in ascending order. For card fields: `avatarSeed`, `avatarSeedSignature`, `avatarVersion`, `bio`, `name`, `noiseKey`, `nostrKey`, `profileRevision`, `profileSignature`, `signature`, `signingKey`, `version`.
3. Absent (`nil`) fields are **omitted** entirely.
4. Bytes are standard padded base64 strings. Empty bytes become `""`.
5. Integers are unquoted decimal numbers without a fractional part, including values above 2^53, common for avatarSeed.
6. Strings are emitted as UTF-8. Only the following are escaped:

   | Character | Encoding |
   | :--- | :--- |
   | `"` | `\"` |
   | `\` | `\\` |
   | U+0008, U+000C, U+000A, U+000D, U+0009 | `\b`, `\f`, `\n`, `\r`, `\t` |
   | Other U+0000–U+001F | `\u00XX`, lowercase hex, such as `\u001f` |

Everything else is unescaped, including `/`, U+007F, U+2028, U+2029, Cyrillic and emoji. This is confirmed by `control-and-separators` in `vectors/02-canonical-json.json`.

Example shape, with abbreviated values; example string contents are preserved:

```json
{"bio":"Барба это жызн!","name":"Игорь Загоев","noiseKey":"q2…=","nostrKey":"7f3a…","signature":"","signingKey":"Zx…=","version":1}
```

## 3. Three signatures

All signatures use the Ed25519 `signingKey`.

### 3.1. Card signature, `signature`

The oldest signature covers only keys, name and bio.

```text
card_bytes = canonical_json( card with
                 signature            = empty ("")
                 profileRevision      = absent
                 profileSignature     = absent
                 avatarSeed           = absent
                 avatarVersion        = absent
                 avatarSeedSignature  = absent )
signature  = Ed25519.sign( signing_private_key, card_bytes )
```

There is no domain label. This is retained for older Shum versions and the push server (section 08). Keys in card_bytes are ordered `bio`, `name`, `noiseKey`, `nostrKey`, `signature`, `signingKey`, `version`. Code: `signedBytes()`.

### 3.2. Avatar seed signature, `avatarSeedSignature`

```text
seed_bytes = "shum.avatar-seed.v1\0"            // 20 bytes
             || noiseKey                         // 32 bytes
             || avatarSeed as UInt64 little-endian  // 8 bytes
avatarSeedSignature = Ed25519.sign( signing_private_key, seed_bytes )
```

seed_bytes exists only with avatarSeed and avatarVersion == 1. Otherwise there is nothing to sign or verify. Code: `avatarSeedBytes()`.

### 3.3. Profile signature, `profileSignature`

The newer signature covers the complete public profile: keys, name, bio, avatar seed, generator version and revision.

```text
profile_bytes = "shum.profile.v1\0"          // 16 bytes
                || canonical_json( card with
                       signature            = empty ("")
                       avatarSeedSignature  = absent
                       profileSignature     = absent )
profileSignature = Ed25519.sign( signing_private_key, profile_bytes )
profileID        = hex( SHA-256( profile_bytes ) )
```

profileID is independent of signature randomness and compares profile contents. Code: `profileBytes()`, `profileID`.

## 4. Card validation

A compatible client **must** validate every received card using these rules (`validate()`). Any failure makes the card invalid.

Common rules:

1. version == 1.
2. noiseKey is 32 bytes and not all zero; signingKey is 32 bytes.
3. nostrKey is exactly 64 `[0-9a-f]` characters (question 1).
4. name is nonempty after trimming whitespace and newlines, and no longer than 64 UTF-8 bytes.
5. bio is no longer than 72 characters and 640 UTF-8 bytes.
6. signingKey is accepted as an Ed25519 public key.

Then choose one of two branches.

**A: profileRevision is present**, a modern card:

1. profileRevision > 0, avatarSeed is present, avatarVersion == 1.
2. profileSignature is 64 bytes and valid for profile_bytes.
3. If signature is nonempty, it must verify against card_bytes.
4. If avatarSeedSignature is present, it must be 64 bytes and valid for seed_bytes.

An empty signature and absent avatarSeedSignature are allowed: this is the card decoded from a c4 QR code.

**B: profileRevision is absent**, a legacy card:

1. profileSignature is absent.
2. signature is 64 bytes and valid for card_bytes.
3. If avatarSeed is present, avatarVersion == 1 and avatarSeedSignature is 64 bytes and valid for seed_bytes.
4. Without avatarSeed, avatarVersion and avatarSeedSignature must also be absent.

## 5. Creating the local card

`ShumIdentityService.card(name:bio:avatarSeed:revision:)`:

1. Fill keys, name, bio trimmed to 72 characters, and version = 1.
2. Sign card_bytes and store signature.
3. Set avatarSeed to the selected seed or the default derived from the Noise key (section 01); avatarVersion = 1.
4. Sign seed_bytes and store avatarSeedSignature.
5. Set profileRevision to the provided revision, default `1`.
6. Sign profile_bytes and store profileSignature.
7. Validate under subsection 4.

A card created by the app always contains all six optional fields.

## 6. Choosing the newer card

When a card arrives for someone already known, `preferred(over:)` chooses a winner:

1. The new card must pass validation.
2. Both cards' noiseKey, signingKey and nostrKey must match. Otherwise it is a different identity and the new card is rejected.
3. Compare profileRevision, treating absence as `0`. The larger revision wins.
4. If equal and greater than zero:
   - Different profileID values: the larger hex string wins.
   - Equal profileID: the card with more supplementary signatures, signature and avatarSeedSignature, wins. On a tie, retain the existing card.
5. If both revisions are zero, the new card wins.

Rule 4 lets two devices restored from backups converge without negotiation.

## 7. Invitations

Cards are shared as links or QR codes. Every link is at most 4096 bytes.

### 7.1. `shum://c4/<data>`: primary format

Currently created for the profile QR code and Share. Data is an unpadded base64url binary string:

| Offset | Size | Field | Byte order |
| :--- | :--- | :--- | :--- |
| 0 | 1 | Format = `4` | |
| 1 | 1 | Card version | |
| 2 | 32 | noiseKey | |
| 34 | 32 | signingKey | |
| 66 | 32 | nostrKey as bytes | |
| 98 | 1 | name byte length, N | |
| 99 | N | name, UTF-8 | |
| 99+N | 2 | bio byte length, M | Big-endian |
| 101+N | M | bio, UTF-8 | |
| 101+N+M | 8 | avatarSeed | Little-endian |
| 109+N+M | 1 | avatarVersion | |
| 110+N+M | 8 | profileRevision | Little-endian |
| 118+N+M | 64 | profileSignature | |

c4 omits signature and avatarSeedSignature because the profile signature already covers everything. The recipient reconstructs a card with empty signature and absent avatarSeedSignature, validation branch A.

Without a seed, revision or profile signature, creation falls back to legacy c. QR codes use error correction level M to remain readable with a central logo.

### 7.2. `https://<server>/invite#c4/<data>`: shareable HTTPS link

The same c4 string wrapped as an HTTPS link recognized by messengers. Data is in the **fragment** after `#`, so opening the page does not send it to the server.

Parsing requires scheme https and exact path `/invite`. Read the fragment as `shum://<fragment>` and parse it again. The server comes from `ShumPushAPIBaseURL`.

### 7.3. Legacy formats clients must read

| Link | Format | Contents |
| :--- | :--- | :--- |
| `shum://c/<data>` | Binary, first byte `1` | Same as c4 through bio, then signature (64 bytes); no seed or revision |
| `shum://c3/<data>` | Binary, first byte `3` | Same as c4 through bio, then signature (64), avatarSeed (8, LE), avatarVersion (1), profileRevision (8, LE), avatarSeedSignature (64), profileSignature (64) |
| `shum://c2/<43 characters>` | Key only | base64url of 32 Nostr key bytes; discover the card through Nostr, section 07 |
| `shum://contact?data=<base64>` | JSON | Card JSON, standard base64 in the data parameter |

New clients **need not** create these formats except c as the fallback in 7.1.

### 7.4. Binary parsing rules

1. Scheme shum; host c, c3 or c4.
2. Strip leading and trailing `/` from the path. It must be nonempty and at most 2048 bytes.
3. Replace `-` with `+` and `_` with `/`, add `=` to a multiple of four, and decode base64. Decoded data must be at most 1536 bytes.
4. The first byte must match the host: c4 → 4, c3 → 3, c → 1.
5. Read fields according to the format table. Insufficient data makes the link invalid.
6. **No bytes may remain** after the last field.
7. name and bio must be valid UTF-8.
8. The reconstructed card must pass subsection 4 validation.

For c2, the path is exactly 43 characters. Adding one `=` must decode to 32 bytes yielding 64 lowercase hex characters.

For contact, data decodes to at most 2048 bytes and is parsed as card JSON, with byte fields in base64, then validated under subsection 4.

## Test vectors

| File | Coverage |
| :--- | :--- |
| `vectors/02-canonical-json.json` | Ten cards: exact card_bytes, profile_bytes, seed_bytes and profileID; Cyrillic, empty bio, quotes, backslash, slash and URL, newline/tab, emoji, controls, U+2028/U+2029; avatarSeed 0, 2^53+1 and 2^64−1 |
| `vectors/02-validate.json` | 33 cards and Swift validation outcomes; invalid cards are re-signed to violate only one rule |
| `vectors/02-invite.json` | c4, c, https://…/invite#c4/…, c3 assembled from this table, c2, contact and two corrupted links, with Swift parsing results |
| `vectors/02-merge.json` | Ten card pairs and the Swift winner |

All vectors use fixed keys in `Shum-iOS/ShumTests/Protocol/ShumProtocolVectorTests.swift`.

## Questions

1. **nostrKey hex characters.** Swift's Character.isHexDigit returns true for fullwidth digits and letters (`０`–`９`, `ａ`–`ｆ`). Such a card could pass but later fail byte conversion. Confirmed by nostr-key-fullwidth-a in vectors/02-validate.json. The specification requires ASCII `[0-9a-f]`. **Fixed** in iOS commit `5b8efc8`; the regenerated vector has swift_valid: false.
2. **Apple-dependent canonical JSON.** Signed bytes depend on JSONEncoder. Escaping and key ordering are now documented and vector-tested, but an Apple behavior change could break verification of older cards. A binary v2 format, CBOR or Protobuf (ROADMAP section 2), would remove this risk.
3. **Bio character count.** The 72 limit counts Swift grapheme clusters (String.count). Counts depend on the Unicode tables, especially for new emoji. A Rust client using another Unicode version could disagree. Proposed v2 rule: count bytes only.
4. **Name trimming.** Swift uses CharacterSet.whitespacesAndNewlines. Enumerating every Unicode scalar on the permitted simulator yielded U+0009–000D, U+0020, U+0085, U+00A0, U+1680, U+2000–200B, U+2028–2029, U+202F, U+205F, U+3000. Rust records this set explicitly, including U+200B, absent from Rust is_whitespace. Fixture: 02-foundation-ed25519.json.
5. **Ed25519 strictness.** CryptoKit and Rust libraries, including normal and strict ed25519-dalek, differ on edge keys/signatures. Swift boundary vectors confirm acceptance of identity public/R with S=0 and zero public/R with S=0 for the recorded message; noncanonical public keys and S are rejected. Rust verify_strict accepts every genuine signature and rejects those degenerate variants. **Owner decision, 8 October 2026:** use verify_strict in Rust. The two degenerate vectors are rejected as an explicit approved compatibility exception. Normal iPhone signatures are accepted; wire format and iOS code are unchanged.
6. **Card signature without a domain label.** signature signs JSON without a label. Its leading `{` currently distinguishes it from other signatures; v2 should add a label.
7. **Safety-code mockup and iPhone 1.0.** CLI mockup 07 shows a shared numeric code for a pair. Inspected iOS AppCoordinator.identityFingerprint instead displays the first 12 SHA-256(noiseKey) hex characters, uppercase, in three groups of four. No confirmed shared numeric algorithm exists in v1. CLI keys verify shows this iPhone fingerprint and full Noise/Ed25519 SHA-256 fingerprints. Agreement on an unverified numeric code is not promised. A shared code needs a separately agreed client contract, without changing v1 packets for it.
