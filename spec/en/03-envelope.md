# 03. Message Envelope

English · [Русский](../ru/03-envelope.md)

v1 draft, derived from Shum-iOS `5b8efc8` and Bitchat revision `9b84b361225facd8e623f25d76f889d3dc54a879` in Upstreams/versions.json.

Sources under Shum-iOS/ShumiOS/:

- Vendor/Messaging/ShumConversationStore.swift: ShumEnvelope, ShumPlaintext, ShumReplyReference, ShumConversation.identifier.
- Vendor/Messaging/ShumMessageStore.swift: send, receive, accept, carry, routeOutbox, routeRelay, makeReceipt.
- Vendor/Messaging/ShumIdentityService.swift: ShumCryptoService, ShumCoding.
- Vendor/Bluetooth/Services/BLE/BLEService.swift: sealShumPayload, openShumPayload, sendShumPacket.
- Vendor/Bluetooth/Services/NoiseEncryptionService.swift: sealCourierPayload, openCourierPayload.
- Vendor/Bluetooth/Noise/NoiseProtocol.swift: NoiseHandshakeState, NoiseSymmetricState, NoiseCipherState, NoisePattern.messagePatterns.

## 1. Layers

```text
ShumPlaintext JSON
  → Noise X to the recipient's permanent Noise key
  → ciphertext in a signed ShumEnvelope
  → envelope in ShumPacket
  → Bluetooth Noise XX or encrypted Nostr layers
```

A message inside Nostr is encrypted both as a Shum envelope and as Nostr content. A courier verifies the envelope signature but cannot read the text. The same signed contents are retransmitted over both transports; outer transport wrappers may change.

## 2. ShumEnvelope fields

All fields are required. JSON follows section 02: UTF-8, sorted keys, no whitespace, Data in standard padded base64 and integers without fractions.

| Field | Type | Value / validation |
| :--- | :--- | :--- |
| version | Integer | Exactly 1 |
| id | String | UUID accepted by UUID(uuidString:) |
| conversationID | String | Derived from both participants' IDs, below |
| sender | Card | Full signed sender card |
| recipient | Card | Signed recipient card |
| timestamp | Int64 | Unix milliseconds, at least zero |
| expiresAt | Int64 | Unix milliseconds, expiration time |
| hopLimit | Integer | 1 through 4 inclusive, created as 4 |
| ciphertext | Data | Nonempty, at most 12 000 decoded bytes |
| signature | Data | Sender's Ed25519 signature |

Key order: ciphertext, conversationID, expiresAt, hopLimit, id, recipient, sender, signature, timestamp, version. Nested cards are fully encoded under section 02, including their signatures.

```text
conversationID = hex(SHA256(UTF8(sort([sender.id, recipient.id]).join(":"))))
digest         = hex(SHA256(ciphertext))
```

Sort the two Shum ID strings. They contain 64 ASCII hex characters, so binary lexicographic ordering matches Swift. The separator is exactly one byte, 3a. digest hashes raw ciphertext bytes, not base64, and is not an envelope field.

Creation uses UUID().uuidString, usually uppercase. Preserve case when reading: the textual ID participates in signatures, inner-message matching and deletion/deduplication keys.

## 3. Signature and lifetime

```text
unsigned = envelope with signature = empty Data, encoded as JSON string ""
signing_bytes = canonical_json(unsigned)
signature = Ed25519.sign(sender.signing_private_key, signing_bytes)
```

The envelope signature has **no domain label**. Do not add one in v1. It covers both cards, ciphertext, ID, times and hopLimit.

validate(at:) first validates both cards, then requires:

1. version == 1, id is a UUID, sender.id != recipient.id.
2. conversationID equals the derived value.
3. timestamp >= 0 and timestamp <= now_ms + 300_000.
4. expiresAt > now_ms and expiresAt > timestamp.
5. expiresAt - timestamp <= 86_400_000, 24 hours.
6. hopLimit is 1…4; ciphertext length is 1…12 000.
7. The signature verifies with sender.signingKey over signing_bytes.

Time boundaries are inclusive only where written as <=. An envelope expiring exactly now is invalid. Creation sets expiresAt = timestamp + 86_400_000. Stored history survives expiration; lifetime limits reception and delivery, not history.

validate does not decrypt. An arbitrary signed 12 000-byte array may pass it but must fail opening at the recipient. Do not combine these distinct checks into an incorrect test.

## 4. Inner message, ShumPlaintext

| Field | Type | Contents |
| :--- | :--- | :--- |
| protocolName | String | Created as shum.message.v1; spotchat.message.v1 also accepted |
| id | String | Exact envelope.id |
| conversationID | String | Exact envelope.conversationID |
| senderID | String | Exact envelope.sender.id |
| recipientID | String | Current recipient's ID |
| timestamp | Int64 | Exact envelope.timestamp |
| expiresAt | Int64 | Exact envelope.expiresAt |
| text | String | Nonempty, at most 4096 UTF-8 bytes |
| reply | Optional object | Omitted when nil, below |

reply requires string fields messageID, senderID and text. messageID is nonempty and at most 255 UTF-8 bytes; it **need not be a UUID**. text is nonempty and at most 4096 bytes. senderID identifies either participant. This is a quotation snapshot; existence of the original message is not checked.

Sending trims text with .whitespacesAndNewlines and checks 4096 bytes. Receiving does not trim: a nonempty whitespace-only string passes the text check. Signatures and encryption use original UTF-8 bytes without Unicode normalization.

## 5. Noise X courier wrapping

### 5.1. Algorithms

| Parameter | Exact bytes / algorithm |
| :--- | :--- |
| Protocol name | ASCII Noise_X_25519_ChaChaPoly_SHA256, 31 bytes |
| Prologue | ASCII bitchat-courier-v1, 18 bytes, **without a zero byte** |
| DH | X25519, 32-byte static and ephemeral keys |
| AEAD | ChaCha20-Poly1305, 32-byte key, 12-byte nonce, 16-byte tag |
| Hash and HKDF | SHA-256, HMAC-SHA256 |
| Premessage | Recipient's static public key rs |
| Only message | → e, es, s, ss, payload |

sealCourierPayload creates a fresh NoiseHandshakeState for each envelope, using the sender's static Noise key, known recipient Noise key and random ephemeral X25519 key. openCourierPayload creates a new responder with the recipient's permanent key and returns plaintext together with the authenticated static sender key.

### 5.2. Initialization and HKDF

```text
h  = UTF8(protocol_name) || 00       # pad the name to 32 bytes
ck = h
h  = SHA256(h || UTF8("bitchat-courier-v1"))
h  = SHA256(h || rs)
```

Shared implementation operations:

```text
MixHash(data): h = SHA256(h || data)
HKDF2(ck, ikm):
    prk = HMAC_SHA256(key=ck, data=ikm)
    t1  = HMAC_SHA256(key=prk, data=01)
    t2  = HMAC_SHA256(key=prk, data=t1 || 02)
    return (t1, t2)
MixKey(ikm): (ck, k) = HKDF2(ck, ikm); nonce_counter = 0
```

There is no string info in HKDF. This is Noise HKDF with byte counters 01 and 02, not an arbitrary HKDF invocation with another domain label.

### 5.3. Construction order

1. Write the 32-byte public ephemeral key e; MixHash(e).
2. Compute es = X25519(ephemeral_private_sender, rs); MixKey(es).
3. Encrypt the sender's static public key s with AEAD key k, current h as AAD, nonce `00 00 00 00 || LE64(0)`. Write encrypted_s || tag, 48 bytes; MixHash those bytes.
4. Compute ss = X25519(static_private_sender, rs); MixKey(ss). This resets the nonce to zero again.
5. Encrypt ShumPlaintext JSON with the new k, current h as AAD and nonce `00 00 00 00 || LE64(0)`. Write encrypted_payload || tag; MixHash those last bytes.

### 5.4. Output format

| Offset | Size | Field |
| :--- | :--- | :--- |
| 0 | 32 | Public ephemeral X25519 key |
| 32 | 32 | Encrypted sender static public key |
| 64 | 16 | Static-key Poly1305 tag |
| 80 | P | Encrypted payload |
| 80+P | 16 | Payload Poly1305 tag |

Total: P + 96 bytes. There is no version byte, length, nonce or type prefix. Unlike transport Noise XX, there is **no four-byte counter**. Even an empty payload produces 96 bytes.

Opening reads e from the first 32 bytes, computes es using the recipient's static private key, opens the 48-byte s, computes ss and opens the remaining payload. Both AEAD checks are mandatory. Recovered s must equal envelope.sender.noiseKey.

### 5.5. X25519 key validation

NoiseHandshakeState.validatePublicKey requires exactly 32 bytes and rejects these raw representations, comparing every byte:

- `00` × 32.
- `01 || 00` × 31.
- `00` × 31 `|| 01`.
- `e0eb7a7c3b41b8ae1656e3faf19fc46ada098deb9c32b1fd866205165f49b800`.
- `5f9c95bca3508c24b1d0b1559c83ef5b04445cc4581c8e86d8224eddd09f1157`.
- `ff` × 32.
- `da || ff` × 31.
- `db || ff` × 31.

It then constructs a CryptoKit key and performs DH. A compatible client must also reject an all-zero shared secret. Transport validation is stricter than card noiseKey validation, which explicitly excludes only the all-zero key. An accepted card is not proof of DH suitability.

## 6. Recipient, forwarding and duplicates

Before decryption, the recipient validates the envelope and all three public keys of its own card. For Nostr, the authenticated outer sender must match envelope.sender.nostrKey.

hopCount is in ShumPacket and is **not covered** by the envelope signature. The originating node sets it to 1 for the first Bluetooth transmission; each courier increments it. The BLE recipient requires 1 <= hopCount <= hopLimit; a courier stores only 1 <= hopCount < hopLimit. Nostr reception skips hopCount checking and records zero in history.

The courier validates the depositor's authenticated BLE identity. Limits are 64 courier envelopes per device, eight per depositor and 4000 seenRelay IDs. A node offers an envelope to at most three distinct couriers in one chain. Section 05 details queues and retries.

An already stored inbound message with the same ID, sender and digest need not be decrypted again. Repeat its ACK if the contact remains permitted to message. Reject a different digest with the same ID. An ID with an active deletion record is not accepted. Relay acceptance is not recipient delivery.

## Test vectors

Generator: ShumTests/Protocol/ShumProtocolVectorTests.swift, courierEnvelopes and messageEnvelopes.

| File | Contents |
| :--- | :--- |
| vectors/03-courier.json | Both directions; empty, Unicode and binary payloads; fixed ephemeral key; real randomized sealing; Swift opening; corruption and wrong recipient |
| vectors/03-envelope.json | Exact JSON/signing bytes, conversation ID, digest, time/length/hopLimit boundaries and corrupted signature |

Fixed ephemeral private keys are only for vectors. Every new client envelope requires fresh random bytes. Ed25519 signatures need verification, not bytewise comparison.

## Questions

1. **No forward secrecy for courier envelopes.** Compromising the recipient's permanent Noise key reveals previously intercepted envelopes. Wrapping in BLE XX does not change Noise X's own properties. Bitchat has prekey variants, but sealShumPayload uses the permanent key. Do not add prekeys without v2.
2. **Unsigned forwarding counter.** A courier can change hopCount. v1 checks its range, not a proven path length.
3. **X25519 validation compatibility.** Swift's blacklist includes unusual extra representations, including `00` × 31 `|| 01`. Preserve and vector-test them during migration. Do not silently replace the blacklist with only a shared-secret check.
4. **Sending and receiving text differ.** Sending trims; receiving only checks nonempty text and byte length. This records current behavior and does not permit rewriting received text.
