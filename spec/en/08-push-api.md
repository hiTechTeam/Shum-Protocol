# 08. Push Server API

English · [Русский](../ru/08-push-api.md)

Sources: iOS 5b8efc8, ShumiOS/App/Services/Notifications/ShumPushService.swift; Shum-Push-Server/internal/api/server.go, internal/identity/auth.go, internal/apns/client.go, internal/store/. Generator pushAPI() intercepts real ShumPushService requests through a local URLProtocol. vectors/08-requests.json uses no internet, APNs or real user keys.

## 1. Purpose and address

Push wakes the recipient to read Nostr. It contains no message, ciphertext, reaction, card or chat keys. APNs delivery is not a ShumReceipt. Only section 04 confirms delivery/read.

iOS reads its HTTPS base URL from bundle ShumPushAPIBaseURL. Empty values or unresolved $(...) substitutions disable requests. Paths below are relative; hostname is not signed. CLI must offer a configurable URL. The protocol does not bind to a network interface.

## 2. HTTP API card

This is a separate DTO, not Shum's network-card JSON:

```json
{"version":1,"noise_key":"base64url32","signing_key":"base64url32",
 "nostr_key":"lowercase-hex32","name":"Имя","bio":"",
 "signature":"base64url64"}
```

Keys and legacy signature use unpadded Base64URL. Seed, avatarVersion, profileRevision and profileSignature extensions are not transmitted. The server verifies the legacy section 02 signature by reconstructing a camelCase object with standard Base64 keys and signature:"", sorting fields, leaving HTML and slash unescaped, and removing Go encoder's trailing LF. There is no additional domain label.

Server requirements: version 1; noise key 32 bytes and not all zero; signing key 32; signature 64; Nostr exactly 64 **lowercase ASCII hex** characters. Name is nonempty after Go TrimSpace, with original name at most 64 UTF-8 bytes. Bio is at most 72 Unicode **runes**. ID is lowercase SHA-256 hex of the raw Noise key. Runes differ from Swift Characters/graphemes.

## 3. Signing every HTTP request

| Header | Value |
| :--- | :--- |
| Content-Type | application/json |
| X-Shum-Public-Key | Unpadded Base64URL, 32-byte Ed25519 public key |
| X-Shum-Timestamp | Decimal integer Unix seconds |
| X-Shum-Nonce | Fresh lowercase UUID in iOS |
| X-Shum-Signature | Unpadded Base64URL, 64-byte Ed25519 signature |

Exact signed UTF-8 bytes:

```text
SHUM1\nMETHOD\nPATH\nTIMESTAMP\nNONCE\nBODY_SHA256_HEX
```

Here \n denotes one LF byte 0a. There is **no trailing LF**. SHUM1 has no NUL. METHOD is uppercase. PATH starts with slash, excluding hostname/query; the server uses URL.EscapedPath(). The three fixed paths have no percent escapes. Timestamp/nonce enter as original header text; the server does not reserialize timestamp before verifying. BODY_SHA256_HEX is 64 lowercase hex characters over **original HTTP body bytes**.

iOS encodes the body with .sortedKeys/.withoutEscapingSlashes JSONEncoder. Signing uses the resulting JSON, not a server-parsed object. Changing whitespace or field order breaks the signature. Header key must equal the validated card key.

Server accepts Int64 timestamp within five minutes of its clock. Nonce is 16–128 bytes and need not be a UUID. Store a successfully verified nonce for ten minutes per public key; reject reuse. This in-memory cache disappears on restart.

## 4. Device registration and removal

```text
POST /v1/devices
DELETE /v1/devices
```

Both use a signed JSON body:

```json
{"card":{},"device_token":"64-hex","environment":"production",
 "app_version":"1.0"}
```

device_token is the 32-byte APNs token as 64 hex characters; server accepts either case and stores lowercase. POST environment must be sandbox or production. iOS DEBUG selects sandbox, Release production. DELETE does not validate environment. app_version is optional on the server; iOS sends bundle version.

POST saves a device under card ID and returns 201 {"status":"registered"}. DELETE removes that identity's token and returns 204 with an empty body. CLI must not invent an APNs token for a computer.

iOS registers after configure(card,signer) and receipt of the token. Deduplication key: card.id:token:environment. Registration retry delays are sequential 0, 15, 60, 300 seconds. Card changes reset the registration key.

## 5. Notification request

```text
POST /v1/notifications
```

```json
{"card":{},"recipient_id":"64-hex","event_id":"uuid-or-other-id",
 "kind":"message"}
```

kind is exactly message, invitation or reaction. recipient_id is 64 ASCII hex characters; uppercase is accepted but lookup uses the original text. Send lowercase for compatibility. event_id is 16–64 ASCII characters from `A-Z a-z 0-9 - _`. It identifies a Shum packet, not necessarily a 64-character Nostr wrapper ID; a 36-character UUID is valid.

After verification, attempt push to every recipient device. Response is **always 202 {"status":"accepted"}**, even with no devices or every APNs attempt failing. sent/failed/removed counts are only in server logs. Remove invalid APNs tokens from storage.

iOS deduplicates in memory by eventID alone, not kind/recipientID, with one concurrent task per ID. Sequential retry delays: 0, 2, 5, 15, 30, 60 seconds. Any 2xx completes successfully. 4xx except 429 stops; retry 429, 5xx and network failures. Identity changes/removal cancel tasks. Section 05 delays push for BLE delivery: eight seconds for messages, five for reactions.

## 6. Limits and error responses

Body limit: 16 KiB. Go ignores unknown JSON fields. Invalid JSON/fields: 400. Card/signature/time/nonce failure: 401. More than 60 requests per minute per IP or invalid gateway secret: 429. Persistence failure: 500. Without APNs configuration, notify returns 503 before reading/authenticating body.

When configured, gateway secret requires X-Shum-Gateway; iOS does not send that secret itself. A gateway is expected before the service.

Every response has Cache-Control: no-store and X-Content-Type-Options: nosniff. JSON uses application/json and a final LF. GET /healthz returns 200 {"status":"ok"}. GET /readyz returns ready plus identities/devices counts, or 503 waiting_for_apns_credentials. GET /invite serves an invitation page outside the signed API.

## 7. APNs payload and wake-up

```json
{"aps":{"alert":{"title":"Имя","loc-key":"PUSH_NEW_MESSAGE_BODY"},
 "sound":"default","content-available":1},
 "shum":{"event_id":"id","kind":"message"}}
```

Reaction loc-key is PUSH_NEW_REACTION_BODY; invitation uses the literal Russian text `Новое приглашение`. Derive title from name: TrimSpace, remove control characters, U+202A–202E and U+2066–2069, take the first 64 runes, then trim again. An empty result becomes Shum. Message text and selected reaction are absent.

APNs uses HTTP/2 POST /3/device/TOKEN to production or sandbox, push-type alert, priority 10, topic bundle ID, expiration now+24h, collapse-id eventID. Server authorization is ES256 JWT cached up to 50 minutes. 410, BadDeviceToken, DeviceTokenNotForTopic and Unregistered remove the token. CLI never needs APNs secrets.

On push, iOS passes eventID to a background handler. Before it is installed, queue at most 16 events with a ten-second timeout. Remote dedup caps at 1000 IDs. Real content is fetched from Nostr.

## 8. Vectors and questions

08-requests.json records registration, all three notification types and DELETE, original bodies/headers/signing bytes and Swift verification. Timestamp/nonce are recorded as generated; replaying server-time checks requires fixture time, not the client's current clock.

CryptoKit randomizes valid Ed25519 signatures. Rust transport tests compare body/signing bytes exactly and verify both signatures with the public key. CryptoKit/Dalek signature byte equality is unnecessary.

1. Server bio counts runes; iOS counts graphemes. Valid iOS emoji/combining-mark bios can receive 401.
2. Go escapes U+2028/U+2029 even with SetEscapeHTML(false). These names need separate Swift canonical-byte compatibility checks.
3. 202 does not prove successful APNs submission. CLI must display only what it actually confirmed, without implying recipient delivery.
4. IP limiter trusts the first X-Forwarded-For. Public backend must sit behind a trusted gateway replacing this header.
5. API requires legacy card signature even with a profile signature. Do not discard it when creating the push DTO.
