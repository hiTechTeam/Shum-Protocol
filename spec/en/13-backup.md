# 13. Profile Backup

English · [Русский](../ru/13-backup.md)

Part of v1 stable. Owner decision, 10 October 2026: every client can create a backup and restore a profile from it, and a backup made in one client restores in any other. The format is implemented once in Rust Shum-Core (section 12, subsection 3): `shum-core` encrypts and decodes the file, `shum-store` exports and imports the profile. Clients only provide the interface: choosing a file, entering the password, reminders.

The current iOS backup (subsection 8) is a snapshot of the app's internal files. It is not portable and stays readable only for migration.

## 1. What a backup is for

A backup moves one profile to a new device or brings it back after the device is lost or the app is deleted. It holds the profile's private keys, so the file is protected by a password the person chooses.

A backup is not a way to use one profile on two devices at the same time. Restoring the same profile on a second device while the first one is still in use creates two devices with the same keys and no device keys of their own. To use a profile on several devices, link them (section 10). A client restoring a backup **must** say this before restoring.

## 2. File

| Property | Value |
| :--- | :--- |
| Extension | `.shumbackup` |
| Media type | `application/octet-stream` |
| Suggested name | `Shum-<profile name>-<YYYY-MM-DD>.shumbackup` |
| Maximum size | 192 MiB |

Layout: the 14 ASCII bytes `SHUM-BACKUP-2` followed by a line feed (`0x0A`), then one UTF-8 JSON object, the header. Nothing follows the header.

```json
{
  "version": 2,
  "createdAt": "2026-10-10T18:04:00Z",
  "profileName": "Руслан",
  "kdf": { "name": "PBKDF2-HMAC-SHA256", "salt": "<base64, 16 bytes>", "iterations": 210000 },
  "cipher": "AES-256-GCM",
  "nonce": "<base64, 12 bytes>",
  "ciphertext": "<base64, ciphertext followed by the 16-byte tag>"
}
```

- `createdAt` is RFC 3339 in UTC with whole seconds.
- `profileName` and `createdAt` in the header are shown before the password is entered, so the person can pick the right file. They are not authenticated. After decryption the client **must** use the values from the payload (subsection 4).
- base64 is standard with padding (RFC 4648, section 4).
- A reader **must** reject unknown `version`, `kdf.name` or `cipher`, a salt other than 16 bytes, a nonce other than 12 bytes, iterations outside 100 000–1 000 000, and a file larger than the maximum.

## 3. Password and encryption

1. **Password.** At least 10 characters (Unicode extended grapheme clusters) and at most 256 UTF-8 bytes after normalization. The password is normalized to Unicode NFC and encoded as UTF-8. Clients **must not** store the password.
2. **Key.** `key = PBKDF2-HMAC-SHA256(password, salt, iterations, 32 bytes)`. Writers use a fresh random salt and 210 000 iterations. PBKDF2 is chosen because browsers provide it through WebCrypto.
3. **Encryption.** AES-256-GCM with a fresh random 12-byte nonce and additional authenticated data equal to the 13 ASCII bytes `SHUM-BACKUP-2`. The plaintext is the payload (subsection 4) as UTF-8 JSON.
4. A wrong password and a damaged file are indistinguishable: both fail the GCM tag. Clients show one message, for example “Wrong password or damaged file”.

## 4. Payload

```json
{
  "version": 2,
  "createdAt": "2026-10-10T18:04:00Z",
  "profile": { "name": "Руслан", "ownerID": "<Shum ID, 64 hex>" },
  "keys": { "noise": "<base64, 32>", "signing": "<base64, 32>", "nostr": "<base64, 32>" },
  "state": { "version": 1, "ownerID": "<Shum ID>", "...": "..." }
}
```

- `keys` holds the three private keys of section 01: Noise X25519, Ed25519 seed (RFC 8032) and Nostr secp256k1.
- The storage key of section 09 is **not** included. It belongs to one device; restoring creates a new random storage key.
- `state` is the database state of section 09 (`version` 1) with every bucket the profile owns: own profile card and its metadata, contacts, requests, conversations, messages, receipts, encounters, saved profiles, blocked contacts, deleted message IDs, invitation states, reactions, outboxes and legacy history.
- Excluded, because they belong to the device and not to the profile: `relay` (envelopes carried for other people) and `seenRelay` (relay deduplication).
- Unknown fields **must** be kept on export and import (section 12, subsection 5).
- Device records and device keys of section 10 are not part of the profile backup. After restoring, the device creates its own device key; until section 10 formats are final, a restored device acts as the only device of the profile.

## 5. Restoring

A client **must** check, before writing anything:

1. `ownerID` equals the Shum ID of `keys.noise` (section 01, subsection 2), and `state.ownerID` equals it too.
2. The own card in `state` is signed by `keys.signing` and contains the public keys of all three private keys.
3. `keys.nostr` is a valid secp256k1 private key.
4. `state` passes the checks of section 09, subsection 8.

Then:

- If a profile with the same `ownerID` already exists on the device, the client **must not** merge silently. It asks whether to replace that profile; the default is to cancel.
- The profile is created as a new local profile (section 12, subsection 6) with the name from the payload. A new storage key encrypts the database; private keys go to the platform key store as for a new profile.
- Writing is atomic: either the whole profile appears, or nothing changes.
- After restoring, the client publishes nothing automatically except what an ordinary start does: relay subscriptions, push registration, outboxes.

## 6. Creating

- A backup always contains the full current state; there are no incremental backups.
- The client creates the file locally and gives it to the person: the system save dialog, a download in the browser, a path in the CLI. Clients **must not** upload backups anywhere on their own.
- Clients may remind the person when the last backup is old. Only the date of the last backup may be stored, never the password or the file.

## 7. Client interface

| Client | Create | Restore |
| :--- | :--- | :--- |
| CLI | `shum backup create <file>` | `shum backup restore <file>`, and as the first choice of `shum init` |
| Desktop and browser | Profile → Backup | “Restore from backup” on the first screen |
| iOS and Android | Settings → Backup | “Restore from backup” during onboarding |

## 8. Current iOS backup, version 1 (migration)

The draft iOS app writes `SHUM-BACKUP-1` followed by a line feed and a JSON header `{version: 1, createdAt, profileName, salt, iterations, ciphertext}`. Differences from version 2:

| Field | Version 1 |
| :--- | :--- |
| `createdAt` | Seconds since 1 January 2001 (Foundation `Date` default) |
| `salt`, `ciphertext` | base64; `ciphertext` is nonce, ciphertext and tag together (CryptoKit `combined`) |
| Additional data | ASCII `hiTeam.ShumiOS.backup.v1` |
| Key | PBKDF2-HMAC-SHA256 as in subsection 3, 210 000 iterations |

The payload is a snapshot of internal iOS files: `LocalCards-v1/own.json`, `ShumProfiles/own.json`, `ShumConversations/state.enc` (database state encrypted with ChaCha20-Poly1305 by the storage key), and Keychain values `noiseStatic`, `noiseSigning`, `localCardSigning`, `nostrIdentity`, `shumStorage` and others.

- Clients **must** write only version 2.
- Shum-Core **may** read version 1 to migrate draft profiles (section 12, subsection 8): take the keys, decrypt `state.enc` with `shumStorage`, then continue as in subsection 5.
- Open question: which of `noiseSigning` and `localCardSigning` is the Ed25519 key of section 01. To be confirmed against the iOS source and the vectors before version 1 import is implemented.

## 9. Vectors

`vectors/13-backup.json`, added with the Shum-Core implementation: a fixed password, salt and nonce with the expected key, file and payload; a file with a wrong password; a damaged tag; a payload whose `ownerID` does not match the keys; a version 1 file created by iOS.
