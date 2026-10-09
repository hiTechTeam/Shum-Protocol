# 04. Packet Types

English · [Русский](../ru/04-packets.md)

v1 draft, Shum-iOS 5b8efc8. Sources under Shum-iOS/ShumiOS/:

- Vendor/Messaging/ShumConversationStore.swift: packet structures and validate.
- Vendor/Messaging/ShumIdentityService.swift: ShumCoding.
- Vendor/Messaging/ShumMessageStore.swift: receive, receiveProfile, transitionInvitation, receiveInvitation, setTyping, sendPresence, receiveRetract, receiveReaction, clearDirectoryEntry.
- Vendor/Messaging/ShumProfiles.swift: ShumProfile, ShumProfileManifest, ShumProfilePacket, ShumProfiles.receive.

## 1. ShumPacket wrapper

UTF-8 JSON with required integer version = 1. Other fields are optional. Encoding omits nil; decoding JSON null also yields nil. Codable ignores unknown fields. Missing required fields do not take Swift declaration defaults: missing version is an error.

Primary fields: card, envelope, receipt, invitation, typing, presence, profileSync, retract, reaction. **Exactly one** must be present. ShumMessageStore.receive checks this, not JSONDecoder or ShumPacket itself.

Auxiliary hopCount (integer) and invitationAvatar (object) are not counted. hopCount is used only with envelope. invitationAvatar reaches the invitation handler, which currently ignores it. An unrelated auxiliary field does not itself invalidate a packet. Clients must create meaningful combinations while retaining actual read compatibility.

ShumCoding key order: card, envelope, hopCount, invitation, invitationAvatar, presence, profileSync, reaction, receipt, retract, typing, version, omitting absent fields. Nested objects are also sorted under section 02. There is no whole-packet signature: contents are signed separately. The decoded packet limit is **24 000 bytes**, both for BLE and Shum content within Nostr.

| Field | Contents | Signer |
| :--- | :--- | :--- |
| card | ShumContactCard, section 02 | Card owner |
| envelope | ShumEnvelope, section 03 | Message sender |
| receipt | ShumReceipt | Original message recipient |
| invitation | ShumInvitationControl | Action sender |
| typing | ShumTypingControl | Person typing |
| presence | ShumPresenceControl | Person opening/closing the chat |
| profileSync | ShumProfileSync | Transmitted profile owner |
| retract | ShumRetractControl | Retracted message author |
| reaction | ShumReactionControl | Reaction author |

Cards inside a signed object enter its signing bytes in full, including their own signatures. Section 05 describes binding to authenticated transport senders and pinned keys.

## 2. Common control format

invitation, typing, presence, retract and reaction require:

| Field | Type |
| :--- | :--- |
| version | Integer, exactly 1 |
| id | UUID string accepted by UUID(uuidString:) |
| sender | Sender card |
| recipient | Recipient card |
| timestamp | Int64, Unix milliseconds |
| expiresAt | Int64, Unix milliseconds |
| signature | Data, standard base64 Ed25519 signature |

Signature: Ed25519(sender.signingKey, ShumCoding(object with signature = "")). None of these five types has a domain label. Sort fields by name, not table declaration order. Preserve UUID case.

All five validate both cards, differing sender.id/recipient.id, timestamp >= 0, timestamp <= now_ms + allowed_future, expiresAt > now_ms, expiresAt > timestamp, the maximum lifetime below and sender signature.

| Type | Maximum future offset | Maximum lifetime from timestamp | Created by iOS |
| :--- | ---: | ---: | ---: |
| invitation | 300 000 ms | 2 592 000 000 ms, 30 days | 30 days |
| typing | 30 000 ms | 10 000 ms | 8 000 ms |
| presence | 30 000 ms | 90 000 ms | 40 000 ms |
| retract | 300 000 ms | 86 400 000 ms | Until original message expiry |
| reaction | 300 000 ms | 86 400 000 ms | 24 hours |

now_ms = Int64(now.timeIntervalSince1970 * 1000). Maxima are inclusive. A valid signature does not make an action valid in the current state.

### 2.1. Invitation and response

Additional required action: request, accept or decline. Other strings fail enum decoding. Key order: action, expiresAt, id, recipient, sender, signature, timestamp, version.

Creation chooses a time after the previous local change: max(now_ms, previous.updatedAt + 1, afterTimestamp + 1). Section 05 covers phases and simultaneous invitations. Own-card packets are also sent for older-build compatibility; this is an additional signal, not another invitation signature format.

### 2.2. Typing

Additional required isTyping: JSON bool. Order: expiresAt, id, isTyping, recipient, sender, signature, timestamp, version.

Sent over available BLE and Nostr without a persistent queue. Repeat true no more often than every four seconds. An initial false without a preceding true is not sent. Received true lasts until expiresAt; false clears it.

### 2.3. Chat presence

Additional required isOnline: JSON bool. Order: expiresAt, id, isOnline, recipient, sender, signature, timestamp, version.

This means the chat with this contact is open, not global network presence. Positive signals refresh every 25 seconds with a 40-second lifetime; negative signals clear presence. Delivery is ephemeral without a historical queue. Backgrounding waits at most four seconds for relay flush.

### 2.4. Retraction

Additional required messageID: UUID string. Order: expiresAt, id, messageID, recipient, sender, signature, timestamp, version.

iOS uses this to cancel one's own undelivered outbound message. The recipient may store a tombstone before the original arrives. Retraction does not clear a whole chat. The control queue uses its own id, distinct from messageID.

### 2.5. Reaction

Required messageID (UUID); optional reaction, one of heart, like, dislike, laugh, fire, coffin, hundred, horror. An absent or null reaction removes it. Swift omits the field when creating a removal.

Order: expiresAt, id, messageID, reaction if present, recipient, sender, signature, timestamp, version. One current value per messageID/author pair. Removal is retained as a tombstone with the signal time and ID. Ordering rules are in section 05.

## 3. Delivery/read receipt, ShumReceipt

This type has **no version field or independent UUID id**.

| Field | Type / validate rule |
| :--- | :--- |
| envelopeID | Original message UUID |
| digest | String of 64 Swift Characters; hex is not checked here |
| sender | Original recipient's card |
| destination | Original author's card |
| read | Bool: false delivered, true read |
| timestamp | Int64, >= 0 and <= now_ms + 300 000 |
| expiresAt | Int64, > now_ms and <= now_ms + 86 700 000 |
| signature | Sender Ed25519, canonical JSON with empty signature |

Order: destination, digest, envelopeID, expiresAt, read, sender, signature, timestamp. No domain label. Both cards are validated.

validate does not check expiresAt > timestamp, lifetime relative to creation, or differing sender/destination. When applying a receipt to an outbound message, the handler additionally compares actual ciphertext SHA-256, ID and participant keys. This matters: 64 z characters pass validate but do not match a real digest.

Storage key: envelopeID + ":" + digest + ":" + sender.id. A read receipt upgrades delivery. Recipient ACK is distinct from a Nostr relay OK, which proves neither delivery nor reading.

## 4. Signed profile synchronization, ShumProfileSync

| Field | Type |
| :--- | :--- |
| sender | Full card with profileRevision |
| recipientID | Recipient Shum ID string |
| knownRecipient | Optional recipient card |
| requestsReply | Bool |
| signature | Data |

No version, id, timestamp or expiresAt. Order: knownRecipient if present, recipientID, requestsReply, sender, signature.

```text
signing_bytes = UTF8("shum.profile-sync.v1\0")
             || canonical_json(sync with signature = "")
```

The label includes a trailing 00. Requires a valid sender card, profileRevision and a valid 64-byte signature. If knownRecipient exists, validate its card and require knownRecipient.id == recipientID. Without it, validate itself does not check recipientID format or nonemptiness. Routing requires the local recipient ID and pinned contact keys.

A reply contains the local card and known peer card. It ACKs a particular profileID, not relay acceptance. requestsReply prevents endless responses; merge details are in section 05.

## 5. Separate invitation avatar

Optional ShumInvitationAvatar in invitationAvatar, with required data and signature Data fields, both base64.

```text
signing_bytes = UTF8("shum.invitation-avatar.v1\0")
             || invitation.signature
             || data
```

Validation requires action != decline, size <= 8192 bytes and ShumProfile.validAvatar: one decodable image, each dimension 1…360 pixels, overall 40 KiB limit. An empty image is invalid. Verify Ed25519 with invitation.sender. The invitation signature enters signing bytes as 64 raw bytes, not base64.

Current receiveInvitation explicitly ignores the attachment and stores avatar=nil; transitionInvitation creates invitations without it. Structural verification support does not imply the current iPhone displays the supplied photo.

## 6. Separate BLE profile channel

ShumProfilePacket is not nested in ShumPacket: it has its own Noise payload type and **6144-byte** limit. It is JSONEncoder JSON, without a canonical wire-encoding requirement or signature. Trust comes from the authenticated BLE session.

Required fields: version (1), kind, request (string, <=64 UTF-8 bytes). Optional: manifest, hash, offset (integer), data (Data).

| kind | Created fields / current receive action |
| :--- | :--- |
| query | Nonempty request; reply with manifest using the same request |
| changed | Usually empty request; issue new query, hints limited to every two seconds |
| manifest | Match pending request; valid manifest |
| chunkRequest | hash, offset; handler returns immediately |
| chunk | hash, offset, data; handler returns immediately |

Declared chunk size is 3072 bytes, but chunk transfer is not implemented. The service permits at most 100 peers and 20 packets per second per peer.

### 6.1. ShumProfileManifest

Required name and bio strings and integer avatarBytes. Optional avatarHash string, avatarSeed UInt64 and avatarVersion integer. Canonical order: avatarBytes, avatarHash, avatarSeed, avatarVersion, bio, name, omitting nil.

name must exactly equal InputValidator.validateNickname(name): nonempty after Foundation trim, at most 50 graphemes, no controlCharacters, in NFC; additionally at most 64 bytes. bio is at most 72 graphemes and 640 UTF-8 bytes, with no control scalars except CharacterSet.newlines. This is stricter than section 02 cards.

- With seed: avatarVersion=1, avatarHash absent, avatarBytes=0. PNG is generated locally; its size/hash do not enter the manifest.
- Without seed: avatarVersion absent. If avatarHash exists, require exactly 64 ASCII [0-9a-f] and avatarBytes 1…40960; without hash, avatarBytes=0.

```text
revision = hex(SHA256(UTF8(name + "\0" + bio + "\0"
    + (avatarHash ?? "") + "\0" + (decimal(avatarSeed) ?? "") + "\0"
    + (decimal(avatarVersion) ?? ""))))
```

Four NUL separators, no trailing NUL. UInt64 is decimal without rounding. This revision differs from card profileRevision and profileID. Receiving a seedless manifest currently saves only name and bio; it does not fetch the photo by hash. ShumProfile contains name, bio, optional avatar Data and optional avatarSeed; it is a local model, not an independently signed network packet.

## 7. Clearing a chat

v1 has no clear packet. clearDirectoryEntry calls local deleteConversation, deletes history and receipts, retains contact and invitation phase, and creates temporary tombstones for known messages. The other participant receives nothing. Removing a contact is another local mode of the same function. Clearing on all devices requires v2.

## Test vectors

Generator: ShumProtocolVectorTests.controlPackets and profilePackets.

| File | Coverage |
| :--- | :--- |
| 04-controls.json | All six signed controls, exact bytes/packets, actions, reactions, removals, lifetime/signature boundaries |
| 04-profile-sync.json | Signed profileSync with/without knownRecipient, wrong recipient, weak recipientID validation |
| 04-invitation-avatar.json | Real PNG, separate signing bytes, validation and decline prohibition |
| 04-profile-packets.json | Every profile kind, manifest, revision, UInt64.max, photo and invalid name |

## Questions

1. **BLE photo avatars are disabled.** chunk/chunkRequest are ignored; a photo manifest does not trigger download. Rust alone cannot make CLI profile avatar --photo deliver a photo to the current iPhone. A separately agreed iOS change is needed.
2. **Clear is local only.** v1 has no peer-clear. iPhone acceptance must verify local deletion and replay protection, not claim synchronized clearing for the peer.
3. **Receipt.validate is weaker than other controls.** It lacks hex digest validation and expiresAt > timestamp checking. Preserve compatibility behavior; actual-message checks remain mandatory.
4. **ProfileSync has no expiry.** Revision rules, not expiration, prevent rollback by an old correctly signed profile. recipientID without knownRecipient can be arbitrary; the handler must validate addressing.
5. **Profile/card validators differ.** Profile.valid rejects some control scalars and requires normalized validateNickname output. Card.validate has different rules. Do not substitute one validator for the other.
