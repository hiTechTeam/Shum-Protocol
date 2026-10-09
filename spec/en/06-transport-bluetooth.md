# 06. Bluetooth Transport

English · [Русский](../ru/06-transport-bluetooth.md)

v1 draft, Shum-iOS 5b8efc8, Bitchat upstream 9b84b361225facd8e623f25d76f889d3dc54a879. Current Swift, including Shum's upstream modifications, takes precedence. Sources in Shum-iOS/:

- ShumiOS/Vendor/Bluetooth/Services/BLE/BLEService.swift, BLEService+LinkLayerCentralRole.swift, BLEService+LinkLayerPeripheralRole.swift, BLERadioController.swift, BLEAnnounceHandler.swift, BLEProximityStore.swift.
- BLEOutboundPacketPolicy, BLEOutboundFragmentPlanner, BLEFragmentAssemblyBuffer, BLEFragmentHandler, BLEIngressPacketGuard in the same directory.
- ShumiOS/Vendor/Bluetooth/Noise/NoiseProtocol.swift, NoiseSession.swift, Services/NoiseEncryptionService.swift, Services/RelayController.swift, Services/TransportConfig.swift, Protocols/Packets.swift, Models/NoisePayload.swift.
- Local dependency localPackages/BitFoundation/Sources/BitFoundation/: BinaryProtocol, BitchatPacket, PeerID, MessageType, MessagePadding, CompressionUtil. ShumiOS.xcodeproj specifies the path; these are iOS sources.

## 1. GATT and discovery

| Object | UUID |
| :--- | :--- |
| Primary service | F85FC602-7866-4F4C-90AB-4F14E107792B |
| Single characteristic | 8F85918D-468D-4FBE-825A-CBD890B74D10 |

This Shum namespace is identical in Debug/Release, not ordinary Bitchat UUIDs. The characteristic supports notify, write, writeWithoutResponse and read, with readable/writeable permissions. A password or BLE pairing is not Shum authentication; Noise establishes trust. The peripheral publishes the service before advertising.

Advertising contains only the service UUID, no Local Name, keys or Shum ID. The central scans by UUID and permits foreground duplicates. After connecting, it discovers the characteristic and enables notify. Central-to-peripheral uses ATT writes; peripheral-to-central uses notifications. Both participants can take either role; both roles are needed for mutual discovery.

ATT success means the stack accepted a write, not Shum delivery. iOS replies successfully before inspecting data. For long writes, it assembles by offset per central, up to 1 000 000 buffered bytes. Notifications use updateValue/readyToUpdateSubscribers backpressure. Do not drop remaining fragments when buffers temporarily fill.

The local maximum BLE frame budget is 512; actual limits come from maximumWriteValueLength/maximumUpdateValueLength and must not be assumed always 512. Fragment pauses: 25 ms directed, 30 ms broadcast; at most two large concurrent transfers. Local radio policy: at most six central links, eight-second connect timeout, baseline -90 dBm RSSI threshold, relaxed to -95/-100 when isolated. These settings are not wire fields.

## 2. Full identity and short ID

```text
Shum ID / fingerprint = lowerhex(SHA256(noise_static_public_key))   # 64 chars
full contact peerID   = lowerhex(noise_static_public_key)           # 64 chars
routing peerID        = fingerprint[0:16]                           # 8 bytes
```

PeerID(hexData:) only encodes bytes as hex. PeerID(publicKey:) derives a short hash ID. ShumContactCard.peerID is full-length; toShort()/routingData and BLEService use the short eight-byte value. Do not interchange these representations in a binary header.

Current BLEService derives the local routing ID from the permanent Noise key. It changes with identity/profile/panic, not periodically during ordinary reconnects. Experimental announceV2 0x2c and rotating-ID code exist, but shipping mesh neither sends them nor accepts them as a new discovery system. Do not enable them automatically.

## 3. BitchatPacket binary format

All multibyte integers are big-endian. Timestamp is UInt64 Unix milliseconds.

| Offset | Size | Field |
| ---: | ---: | :--- |
| 0 | 1 | version, 1 or 2 |
| 1 | 1 | type |
| 2 | 1 | ttl |
| 3 | 8 | timestamp |
| 11 | 1 | flags |
| 12 | 2 for v1, 4 for v2 | payloadLength |
| 14/16 | 8 | senderID |
| Following | 8 if flag 01 | recipientID |
| Following | 1 + 8*n, v2 and flag 08 only | Route count and n IDs |
| Following | payloadLength | Payload including compression preamble |
| Following | 64 if flag 02 | signature |
| Following | Optional | Padding |

Minimum 22 bytes for v1, 24 for v2. Upstream comments about 21 bytes and header=13 are outdated; executable code uses header=14/16.

Flags: 01 hasRecipient, 02 hasSignature, 04 compressed, 08 hasRoute, 10 isRSR. Ignore route in v1. Decoder does not reject other bits. Encoder truncates IDs to eight bytes or right-pads zeroes. Clients must supply correctly derived eight-byte IDs, not truncate full Noise keys instead. Route count <=255; an empty hop is invalid.

Absent recipient or eight ff bytes means broadcast; other recipients are directed. payloadLength excludes IDs, route, signature and padding. Decoder permits trailing bytes after the declared body and separately checks field presence/sizes. Binary decoding alone does not verify signatures.

Required Shum types: announce=01, leave=03, noiseHandshake=10, noiseEncrypted=11, fragment=20. requestSync=21 belongs to Bitchat gossip. Bitchat courierEnvelope=04 is another outer protocol. Section 05 Shum relay carries ShumPacket through Noise payload 41, not type04.

### 3.1. BitchatPacket signature

```text
unsigned = packet(signature=nil, ttl=0, isRSR=false)
signing_bytes = BinaryProtocol.encode(unsigned, padding=true)
signature = Ed25519.sign(signing_private_key, signing_bytes)
```

TTL and RSR may change and are unsigned. Route is signed when present. Compression and padding apply to signing_bytes exactly as in the encoder. announce is signed; ordinary Shum noiseEncrypted is created with signature=nil because Noise authenticates it.

### 3.2. Compression

Compress the **payload** before adding the binary header. Threshold is 100 bytes, not the old comment's 256. A candidate with count>=100 qualifies if uniqueBytes / min(count,256) < 0.9 and compression actually shortens it. Apple Compression COMPRESSION_ZLIB produces **raw DEFLATE**, with no RFC1950 zlib header or Adler-32, as confirmed by Swift vectors. Independent zlib.decompress(bytes, -15) succeeds; mode 15 rejects it.

With flag04, payload is originalSize (UInt16/UInt32 BE by version) followed by compressed bytes. Header payload length includes the originalSize preamble. Decompression must produce exactly originalSize. Enforce ratio limit 50 000 and FileTransferLimits.maxFramedFileBytes before allocating. Uncompressed wire is also allowed. The heuristic usually avoids compressing Noise ciphertext. Rust must decode the exact 06-frames stream; arbitrary JSON gzip is unsuitable.

### 3.3. Padding

BLE policy enables padding only for noiseHandshake/noiseEncrypted. MessagePadding chooses the first bucket in 256,512,1024,2048 that fits encodedLength + 16; for larger frames target=encodedLength. Pad only when the difference is 1…255, each byte equal to that difference, PKCS#7-style. Differences above 255 leave the frame unpadded. Padding applies to the whole outer binary frame, not Noise payload.

Decoder first reads as-is, then attempts unpadding on failure. It normally treats padding as an allowed tail. Never feed that tail to Noise AEAD: ciphertext length comes from payloadLength; a separate flag defines signature presence.

## 4. Announce and binding

TLV: UInt8 type, UInt8 length, then length bytes. Encoder order:

| TLV | Value |
| :--- | :--- |
| 01 | UTF-8 nickname, <=255 bytes |
| 02 | Public Noise key, 32 bytes |
| 03 | Public Ed25519 key, 32 bytes |
| 04 | Optional up to ten eight-byte neighbors |
| 05 | Optional capabilities, minimally encoded little-endian UInt64 |
| 06 | Optional bridge geohash, 1…12 UTF-8 bytes |

01–03 are required; skip unknown TLVs. Structural decoding is not trust. The handler binds senderID to the Noise-key hash, verifies announce signature and stored signing-key pin, and rejects stale/self packets. TTL=7 signals directness but is not cryptographic evidence of physical proximity. Noise must prove key ownership before accepting cards or Shum content.

After XX, send authenticatedPeerState, Noise payload21: version01 || TLV01 capabilities || TLV02 signing key32. Skip unknown TLVs, reject known-field duplicates and noncanonical capabilities. This binds signing key to proven Noise key; a public announce capability alone does not.

## 5. Noise XX between nodes

Noise_XX_25519_ChaChaPoly_SHA256 is exactly 32 ASCII bytes; prologue is empty, no premessage. Initialize h=protocol_name, ck=h, then MixHash(prologue) even for empty Data. HKDF/MixHash/MixKey/AEAD follow section 03. Pattern:

```text
→ e                  + encrypted_or_clear_payload
← e, ee, s, es       + encrypted_payload
→ s, se              + encrypted_payload
```

Ordinary handshake payloads are empty. Frames contain respectively 32, 96 and 64 bytes, including tags for empty encrypted payloads. Each e is 32 bytes and encrypted s is 48. MixKey resets nonce; EncryptAndHash uses current h as AAD and updates h with ciphertext. Do not omit an empty payload's tag once a cipher key exists.

Handshake bytes go directly into directed Bitchat noiseHandshake10 payload. On completion, split=HKDF2(ck, empty): initiator sends c1/receives c2; responder sends c2/receives c1. Retain transcript hash for channel binding. Match remote static key to known contact/subsequent cards. If needed, queue payload before starting handshake to avoid losing a fast response. Noise service resolves simultaneous-handshake conflict; support responder regardless of who established the GATT link.

### 5.1. Transport cipher

```text
nonce12 = 00 00 00 00 || LE64(counter)
wire = BE32(counter) || ChaCha20Poly1305(ciphertext || tag)
AAD = empty
```

Each direction starts counter=0. Sending stops at UInt32.max-1; beyond that establish a new session. Receiver extracts BE32, constructs the same 12-byte nonce, verifies AEAD and updates replay state only after success. The declared window is 1024 counters and allows out-of-order packets. Transport wire is at least 20 bytes. Handshake cipher instead has no four-byte counter prefix and uses transcript h as AAD.

### 5.2. Shum typed payload

```text
plaintext = 41 || ShumPacket JSON          # up to 24000 JSON bytes
plaintext = 40 || ShumProfilePacket JSON   # up to 6144 JSON bytes
```

One type byte without a length, followed by Data. After encryption, use binary noiseEncrypted11, signature=nil, version1, ttl7 and recipient=routing ID. Noise X ciphertext inside the envelope remains another encryption layer. A Shum courier opens its own XX session with the depositor, then stores the envelope still sealed for the recipient.

## 6. Fragment wire and assembly

First encode the original packet fully, including padding/signature, then split its bytes into chunks. Each type20 fragment has payload:

```text
fragmentID[8] || BE16(index) || BE16(total) || originalType[1] || chunk
```

Index starts at zero; total is 1…10000; index<total. Fragment header is 13 bytes. Assembly key: senderID8 plus fragmentID8. Sender, timestamp, ttl and recipient come from original; signature=nil. A directed override can select another link peer. Originals with route use fragment version2 and a copied route.

Default chunk=469. For a real link budget use max(64, limit - 42). Route sizing includes v2 header, IDs, count+8*n route, fragment header and 16 bytes of margin. Explicit maxChunk is at least 64. Reverse arrival is supported; concatenate by index and decode the original frame again.

At most 128 assemblies; evict the oldest at capacity. Lifetime is 30 seconds from assembly start, not last duplicate. A duplicate index replaces bytes but does not reset stall time. Broadcast stalls after five seconds may request the missing stream through requestSync, no more often than every ten seconds; directed transfers do not recover this way. Assembly size limit depends on originalType; file/noise permit the framed file limit. Reconstructed data still needs Noise/Shum validation; unverified fragment originalType is not trusted.

## 7. Mesh and two separate counters

Outer Bitchat ttl starts at seven and decrements per mesh relay; it is unsigned. v2 source routing permits intermediate eight-byte IDs. This is the binary transport version, **not Shum's version or v2 devices**.

BLEIngressPacketGuard requires timestamp skew<=120000 ms except an allowed RSR sync response. Do not set RSR without a real request. Reject loopback of the local ID.

Separate Shum hopCount starts at one for courier-to-courier envelope transmission and is bounded by envelope.hopLimit<=4. Several physical mesh hops within one XX session do not become several Shum courier hops. Bitchat gossip deduplication does not replace section 05 receipts, queues or tombstones.

## 8. Distance and nearby list

Distance comes only from local RSSI of a physical BLE peripheral bound to a peer. Relayed packets are not distance measurements. BLEProximityStore accepts -127…-1, ignores 127 and zero, retains up to 30 recent samples per peripheral and 256 peripherals. Samples qualify only at age>=0 and <30 seconds.

```text
readings = sorted(all current RSSI samples of peripherals bound to peer)
median = readings[count / 2]             # upper median for an even count
distance = max(1, round(10^((-59 - median) / 20)))  # integer meters
```

No valid RSSI means nil, not zero meters. This estimate assumes -59 dBm at one meter and exponent two; walls and device placement affect RSSI. Shum nearby membership also requires a card with a Noise key matching the established session. Seeing the service UUID alone is insufficient.

## Test vectors

Generator: bluetoothFrames and bluetoothNoise, without radio.

| File | Contents |
| :--- | :--- |
| 06-frames.json | v1/v2 binary, padding, compression, route, signing bytes, announce, both peerIDs |
| 06-fragments.json | Fixed stream ID, real frame bytes, reverse-order assembly |
| 06-noise-xx.json | Three deterministic handshake frames, transcript hash, bidirectional transport, replay check |
| 06-proximity.json | RSSI median, rounding, absent signal, exact 30-second boundary |

## Questions

1. **Section 01 short-ID change clarification.** Shipping BLE derives it from permanent Noise key. “Changes over time” does not mean periodic rotation in iPhone 1.0; experimental rotating announceV2 is unused. CLI must follow shipping derivation.
2. **Compression and obsolete comments.** Actual threshold is 100; header is 14/16. Do not use comments claiming 256/21/13 for compatibility. Apple output is fixture-confirmed and independently checked as raw DEFLATE.
3. **Confirmed replay-window defect.** Swift shifts the bitmap in the wrong direction as counter increases. After successful counter0 and counter1, replayed counter0 authenticates and is accepted again: swift_replay_zero_after_one_accepted=true in 06-noise-xx. Fixing iOS is outside this work. **Owner decision, 8 October 2026:** Rust uses a correct 1024-counter window and rejects replay after AEAD verification. This approved exception covers the Swift defect; fresh packets, in-window out-of-order delivery and wire format remain compatible.
4. **Fragment metadata.** Completion uses incoming total without comparing total/originalType to the first fragment, and duplicate indices replace data. Evaluate separate anti-substitution limits. Final AEAD remains mandatory; incomplete assembly is untrusted.
5. **Some v1 signatures lack a domain label.** Section 01's blanket statement does not apply to legacy card, envelope, controls or binary announce. Use each type's actual signing bytes as defined in its section.
