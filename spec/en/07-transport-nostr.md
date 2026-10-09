# 07. Nostr Transport

English · [Русский](../ru/07-transport-nostr.md)

Derived from iOS 5b8efc8. This is the deployed private Bitchat/Shum format using Nostr events. Kinds 1059/13/14 and the v2: prefix do not imply NIP-17, NIP-44 or NIP-59 compatibility. Do not substitute those NIPs' encryption for the format below.

## 1. Sources

Under Shum-iOS/ShumiOS/:

- Vendor/Nostr/NostrProtocol.swift: createPrivateMessage, decryptPrivateMessage, encryptContent, decryptContent, NostrEvent.
- Vendor/Nostr/NostrRelayManager.swift: connections, filters, acknowledgments.
- Vendor/Nostr/: NostrIdentity, NostrTransport.
- Vendor/Messaging/ShumNostrService.swift: Shum reception, card lookup, queue limits, handled-event persistence.
- Vendor/Messaging/ShumConversationStore.swift: ShumWireProtocol.
- Vendor/: XChaCha20Poly1305Compat, Base64URLCoding.

Generator: ShumTests/Protocol/ShumProtocolVectorTests.swift, nostrEnvelopes(). Fixtures: 07-events.json, 07-private-envelopes.json.

## 2. Event and signature

Object fields: id, pubkey, created_at, kind, tags, content, optional sig. created_at is integer Unix seconds. Public key is 32-byte x-only secp256k1, 64 hex characters. BIP-340 signature is 64 bytes, 128 hex characters; id is SHA-256, 64 hex characters.

SHA-256 input is compact UTF-8 array JSON:

```text
[0,pubkey,created_at,kind,tags,content]
```

Element order is mandatory. Slash is unescaped; strings are not NFC-normalized. This uses JSONSerialization array encoding, not section 02 card object canonicalization. Fixtures record exact bytes, including Cyrillic, backslash, U+2028/U+2029 and controls. Sign the hash, not JSON directly. iOS supplies 32 random auxiliary bytes to Schnorr. Verification recomputes id, checks sizes and verifies against pubkey.

Decoder limits: 64 tags, 16 values per tag, 1024 UTF-8 bytes per tag string. Tag/value ordering is signed.

## 3. Three nested layers

| Layer | kind | Author | tags | content | Signature |
| :--- | ---: | :--- | :--- | :--- | :--- |
| rumor | 14 | Actual sender | [] | Original string | Absent |
| seal | 13 | Actual sender | [] | Encrypted rumor JSON | BIP-340 |
| gift wrap | 1059 | Fresh random key | [["p",recipient]] | Encrypted seal JSON | BIP-340 |

rumor uses current time. seal and wrap independently choose random times 1–300 seconds earlier. Each wrap gets a new secp256k1 identity. Both encryptions target the same recipient key. Nested IDs use ordinary derivation.

Receive strictly in this order:

1. Outer kind is 1059, tags exactly contain the sole p tag for the local key, and outer signature verifies.
2. Decrypt content and decode seal: kind 13, empty tags, required valid signature.
3. Decrypt seal content and decode rumor: kind 14, no signature, pubkey exactly matches seal signer.
4. rumor tags are empty or exactly [["p",localKey]], retained for older Android compatibility. Reject every extra tag.
5. Return rumor content, **seal author**, and rumor time.

The outer wrapper author is not the contact. Section 05 binds the authenticated author to its card.

## 4. Private v2: encryption

Uses the identity's same 32-byte secret secp256k1 key. Lift the recipient x-only key to even Y with prefix 02 plus 32 X bytes. Sending code also falls back to 03 if public-key construction fails.

Swift ECDH uses format: .compressed. Its result is **33 bytes**: 02 or 03 plus the X coordinate of private × public. This is neither libsecp256k1's usual 32-byte ECDH hash nor only X.

```text
shared = compressedSEC1(private × public)       # 33 bytes
key = HKDF-SHA256(ikm=shared, salt=empty,
                  info=UTF8("nip44-v2"), length=32)
nonce = random(24)
(ciphertext,tag) = XChaCha20-Poly1305(key,nonce,plaintext,AAD=empty)
content = "v2:" + base64url_no_padding(nonce || ciphertext || tag)
```

info has no trailing zero; salt is not the string nip44-v2. There is no version, padding or MAC outside AEAD. Tag is 16 bytes. XChaCha uses HChaCha20 with the first 16 nonce bytes, then ChaCha20-Poly1305 with nonce `00000000 || nonce[16..24]`.

Decrypt by trying the author's x-only key with prefix 02, then 03: original secret-key parity affects the shared point. AEAD verification determines success. Base64URL decoding also accepts standard + and / and restores padding. Sending uses URL-safe unpadded encoding.

Limit encrypted content to 64 KiB **before** base64 decoding. Combined data must exceed 40 bytes: nonce24, nonempty ciphertext, tag16. Decrypted nested JSON is also limited to 64 KiB and must be valid UTF-8. Repeat checks for both layers.

## 5. Shum content

rumor string:

```text
"shum-v1:" + standard_base64(ShumCoding.encode(ShumPacket))
```

Also accept spotchat-v1:. This is standard Base64, not outer-cipher Base64URL. Packet JSON is limited to 24 000 bytes; the prefixed input string to 33 000 bytes. After decoding apply sections 03–05. Nostr does not carry outer BLE hopCount as proof of a path.

ShumNostrService has no separate general rumor-age limit. Type handlers check packet expiry. Do not import the 900-second window used by neighboring Bitchat handlers.

## 6. Relays, subscriptions and acknowledgments

Built-in list at this revision:

```text
wss://nostr.oxtr.dev
wss://soloco.nl
wss://relay.snort.social
wss://nostr.bitcoiner.social
```

User relays are added separately. This records iOS configuration, not external-server availability.

Subscription shum-private-v1:

```json
{"kinds":[1059],"#p":["свой-x-only-ключ"],"since":0,"limit":100}
```

In practice, since is now minus three days in Unix seconds. NIP-01 request: ["REQ",subscriptionID,filter]; publication: ["EVENT",event]. Restore subscriptions after reconnect. EOSE means stored results ended, not user delivery.

sendShumPacket sends immediately. Success requires at least one relay OK(eventID,true,...) within ten seconds. This gives forwarding; only signed ShumReceipt gives delivered/read. Sections 05 and 08 define queues, retries and push. Shum status needs a connected DM relay, not merely any geographic relay.

## 7. Deduplication and receive limits

At most four concurrent handlers and 128 queued events. Queue overflow drops an event without marking it handled. Fast seen set caps at 1000. Persistent handled retains four days: target 30 000 entries; above 31 000, keep the 30 000 newest. Loading prunes expired entries.

File format is unordered 36-byte records: eventID[32] || handledUnixSeconds[4,BE]. Save is deferred two seconds. This unencrypted event index is separate from SQLite. Decryption failures/invalid events are also marked handled. ShumEnvelope deduplication is independent of wrapper eventID, section 05.

## 8. c2 card lookup

Extract the Nostr key from c2 under section 02. Send an ordinary private message using the same three layers. String formats:

| Prefix | JSON after standard Base64 | Decoded limit |
| :--- | :--- | ---: |
| shum-contact-request-v1: | {"id":"uuid-lowercase"} | 256 |
| shum-contact-manifest-v1: | {"id":...,"card":...,"profile":...} | 4096 |
| shum-contact-chunk-v1: | id, hash, offset, data | 6144 |

At most eight concurrent requests; 20-second timeout; self-lookup forbidden. id must parse as UUID; creation uses 36 characters. Lookup uses sendEvent without waiting for relay OK as message sending does. Responses to one key are spaced at least three seconds. At 1000 limiter keys, discard entries older than 300 seconds.

Accept a response only for a pending id. The target contact key, authenticated seal author and card.nostrKey must match. Validate card and manifest. Manifest name, bio and avatarSeed must exactly match card. Render avatar locally from seed. Current handlers do not accept photos/chunks; packet names alone do not establish working Nostr photo transfer.

## 9. Vectors and questions

07-events.json records hashing bytes, BIP-340 and subscription filter. 07-private-envelopes.json contains a real wrapper, recipient keys, both intermediate shared/key/plaintext values and negative cases. Historical single-p-tag rumor is accepted; extra tag, wrong recipient, invalid signatures and mismatched rumor author are rejected. Tests do not connect to live relays.

1. Private encryption named v2: is easily confused with NIP-44. Reproduce it in v1; migration to the standard requires a separate v2.
2. The plaintext persistent event index exposes IDs and receive times. CLI storage can protect it without changing network v1.
3. A seedless card and manifest using fallback seed may fail strict matching. Add a separate legacy card test without extensions.
4. Public-relay availability needs operational checking. The list records code, not current network state.
