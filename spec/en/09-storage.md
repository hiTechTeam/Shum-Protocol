# 09. Encrypted SQLite Storage

English · [Русский](../ru/09-storage.md)

Sources: iOS 5b8efc8, ShumiOS/Vendor/Messaging/ShumSQLitePersistence.swift, ShumConversationStore.swift, ShumIdentityService.swift. This is a local storage format, not a database-transfer protocol. encryptedSQLite() uses real ShumSQLitePersistence with test keys. vectors/09-storage.json contains the complete SQLite file in Base64, rows before/after decryption, state and failure checks.

## 1. Key and owner

Storage key is independently generated random 32 bytes, not derived from Noise, signing or Nostr keys. iOS stores shumStorageKey in Keychain and reads spotchatStorageKey for migration. ownerID is lowercase SHA-256 hex of the Noise public key, section 01. Each database belongs to one ownerID. A wrong owner or state version other than 1 gives unavailableIdentity.

Do not use a network-encryption key as the storage key. Identity secrets are not automatically written to records.

## 2. SQLite schema and connection

```sql
CREATE TABLE records (
  bucket TEXT NOT NULL,
  id TEXT NOT NULL,
  position INTEGER NOT NULL,
  payload BLOB NOT NULL,
  PRIMARY KEY(bucket,id)
) WITHOUT ROWID;
PRAGMA user_version=1;
```

Open READWRITE | CREATE | FULLMUTEX. WAL, synchronous FULL, secure_delete ON, busy_timeout 1000 ms. Serialize all writers/checkpoints for one standardized pathname. Read transactions use BEGIN DEFERRED; writes BEGIN IMMEDIATE. close performs WAL checkpoint TRUNCATE.

bucket, hidden id, position, sizes and SQLite structure are unencrypted. State JSON, actual IDs and text are inside AEAD payload. This is record encryption, not SQLCipher. WAL carries the same ciphertext. Moving an open database must account for WAL; copying only the main file before checkpoint is insufficient.

## 3. Hidden index, AAD and packing

For an ordinary row:

```text
indexID = lowerhex(HMAC-SHA256(storageKey, UTF8(bucket + NUL + realID)))
AAD = UTF8(bucket + NUL + indexID + NUL + decimal(position))
packed = lengthBE16(realID.utf8) || realID.utf8 || payloadJSON
cipher = ChaCha20-Poly1305(storageKey, randomNonce12, packed, AAD)
SQLite.payload = nonce12 || ciphertext || tag16
```

NUL is one 00 byte. Decimal position uses ordinary Int notation without leading zeroes. HMAC uses raw key, not hex/base64. Length covers only realID UTF-8, in range 1–65 535. Following JSON must be nonempty; ID must be valid UTF-8. ChaChaPoly combined matches CryptoKit: nonce12, ciphertext, tag16, no version/extra prefix. Row payload must exceed 28 bytes.

Special row bucket="header", id="root", position 0: indexID is literal root without HMAC; packed realID is also root. AAD hex: 68656164657200726f6f740030.

Reader decrypts with column-derived AAD, unpacks realID, recomputes HMAC and compares with id column. Altering bucket, position, indexID or ciphertext must fail, not skip a row.

## 4. Header

Decrypted JSON:

```json
{"state":{"version":1,"ownerID":"..."},"counts":{"contacts":0}}
```

state uses the full snapshot's Codable ShumDatabase, with row-extracted arrays/dictionaries cleared. Preserve the distinction between absent optional fields and empty arrays/dictionaries. counts includes **every** bucket below, even zero counts, but excludes header.

State retains ownerID/version, ownProfileCard, unviewedEncounterIDs, pinnedDirectoryEntries and legacyHistory metadata without its messages array. Do not replace it with a minimal object and lose unknown metadata. The root row is required with position zero.

## 5. Array buckets

| bucket | Row JSON type | realID |
| :--- | :--- | :--- |
| contacts | ShumContact | card.id |
| requests | ShumContactCard | card.id |
| conversations | ShumConversation | id |
| messages | ShumStoredMessage | envelope.id |
| relay | ShumRelayCopy | envelope.id |
| receipts | ShumStoredReceipt | receipt.key |
| encounters | ShumEncounter | card.id |
| savedProfiles | ShumSavedProfile | card.id |
| invitationOutbox | ShumStoredInvitationControl | control.id |
| profileOutbox | ShumProfileDelivery | recipientID |
| retractOutbox | ShumStoredRetract | control.id |
| reactionOutbox | ShumStoredReaction | control.id |
| legacyMessages | ShumLegacyMessage | id |

Read ORDER BY bucket,position. Within arrays, positions strictly increase and the first is > -1. Gaps are allowed; consecutive numbering from zero is unnecessary. Reject duplicate realIDs on writing. Deletion retains other positions; new tail elements use lastPosition+1. Reordering or inserting before existing elements renumbers the array and requires new AEAD for rows whose positions change.

## 6. Dictionary buckets

| bucket | JSON value | realID |
| :--- | :--- | :--- |
| seenRelay | Date | Dictionary key |
| blocked | ShumContactCard | Blocked identity ID |
| deletedMessageIDs | Date | Message ID |
| invitationStates | ShumInvitationState | Contact ID |
| reactions | [String: ShumReactionMark] | Message ID |

Every dictionary row has position exactly zero. Value is separate from its key, already packed in realID. reactions stores the entire inner personID-to-mark dictionary as one row per messageID.

## 7. JSON and values

Payload uses ShumCoding.encode: sortedKeys, withoutEscapingSlashes, Data in **standard padded Base64**. Optional nil is usually omitted. Set becomes an array with unspecified, meaningless order. Do not require exact forwardedTo/sentTo/unviewedEncounterIDs ordering in snapshot comparisons.

Swift Date defaults to seconds since 2001-01-01T00:00:00Z, not Unix milliseconds:

```text
unixSeconds = jsonDate + 978307200
```

Numbers can be fractional or negative; .distantPast is valid. Wire timestamp/expiresAt/updatedAt remain Int64 Unix milliseconds. Both time representations coexist within one stored message.

ShumStoredMessage stores envelope, decrypted text, outgoing, status (queued, forwarding, delivered, read, expired, cancelled), unread, hopCount, deliveryTransport, attempts, lastAttempt, forwardedTo, nostrAccepted and optional lastNostrAttempt/nostrAttempts/deliveredAt/readAt/reply. Receipt/control outboxes keep their own attempt/transport metadata. Exact Codable keys and nil/default values are recorded in fixture state_json and Swift types; these are not wire packets.

## 8. Full read and normalization

Read one snapshot:

1. Decrypt root and decode Header.
2. Read every row; validate size, AEAD, packed structure and HMAC.
3. Compare each bucket's count; reject extra buckets.
4. Reconstruct arrays in position order and dictionaries at position zero. Fill optional collections only if their field existed in header.state.
5. Total encrypted payload must not exceed 100 000 000 bytes.
6. Check version/owner and apply legacy normalization.

Never publish partial state on failure. Normalization creates absent invitationStates as accepted for existing contacts, updatedAt=addedAt in Unix ms, eventID=`legacy-`+contactID; absent invitationOutbox becomes []; savedProfiles always becomes nil. If normalization changes state, iOS writes it transactionally.

## 9. Writing and atomicity

Commit compares previous/next, writes changed rows/root and deletes removed rows. Unchanged rows retain ciphertext/nonce. An identical-state commit creates no transaction. Every changed row gets a fresh random nonce even for the same ID. All changes/counts share one SQLite transaction; failure ROLLBACKs. Ciphertext sum after commit is at most 100 000 000 bytes.

ShumConversationStore makes the durable commit before publishing memory or allowing ACK. CLI must preserve this order and never acknowledge before successful persistence.

First creation uses a neighboring temporary database, complete commit/checkpoint/close, then atomic rename. iOS excludes the directory from backup and applies completeUntilFirstUserAuthentication protection. Other OSes need equivalent local access restrictions.

## 10. Legacy snapshots and migration

Detect SQLite by its first 16 bytes, SQLite format 3\0. Otherwise iOS tries legacy v1 snapshot: ChaChaPoly combined, empty AAD, complete ShumDatabase JSON, at most 100 000 000 bytes. After decrypt/normalize, create temporary SQLite, commit/checkpoint/close, copy the old file to <path>.v1-recovery, then atomic rename. Failure must not destroy the original.

Legacy Bitchat history migrates separately in ShumConversationStore.prepare: AES-GCM combined with a separate legacy key, JSON ShumLegacyArchive, limit 100 000 000. This differs from storage-key ChaChaPoly. snapshot(at:key:) reads either v1 backup or validates every SQLite row before returning a portable snapshot.

## 11. Vectors and questions

Fixture contains header, contacts, requests, conversation, text message, relay, receipt, encounter, dictionaries, profileOutbox and profile metadata. Swift created the database and closed it after checkpoint; Rust opens the original file. Each row records indexID, realID, AAD, packed data, JSON and combined ciphertext. Negative checks cover wrong key/owner, altered position, contacts deleted without counts update, and corrupted ciphertext. No-op commit is checked.

09-rust-roundtrip.json is generated by Rust from the Swift database with changed stored text, reordered contacts, fractional Dates and a tombstone. The real Swift reader on the permitted simulator checks the complete database/portable snapshot in rustSQLiteRoundTrip(). This tests storage: altered stored text is not a newly signed wire message. 09-legacy-cipher.json comes from CryptoKit AES-256-GCM with separate legacy key and fixed nonce; it tests encryption, not historical Bitchat archive Codable schema.

1. Individual JSON payloads have no schema version. Rust additions must preserve unknown fields and optional values during round trips.
2. Counts, sizes and buckets are visible. Do not claim whole-database encryption.
3. Legacy restore intentionally clears savedProfiles. A separate product decision is needed about preserving this bucket.
4. Invalid rows in an optional bucket hidden by header nil are still checked cryptographically and against counts, but may not enter state. New writers should not silently create such states.
5. SQLite transactions protect rows atomically, but correct diffing requires previous snapshot. Independent writers with stale previous values can lose changes. CLI needs one database owner/daemon with IPC or a profile lock.
