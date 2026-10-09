# Shum Protocol

English · [Русский](README.ru.md)

Protocol specification and compatibility test vectors for Shum. Identities use keys created on the device; encrypted messages travel over Bluetooth or Nostr relays.

## Status

The protocol has not reached a stable release. Sections 01–09 document the current v1 draft used by iOS and the Rust core. Multiple devices, sync and profile relays are planned for v1 stable; their wire formats still need work.

All specification sections are available in English and Russian, in `spec/en/` and `spec/ru/`. Literal UTF-8 strings in data examples are preserved across translations.

## Specification

| Topic | Document |
| :--- | :--- |
| Overview | [00](spec/en/00-overview.md) |
| Keys and identity | [01](spec/en/01-keys-identity.md) |
| Contact cards and invitations | [02](spec/en/02-contact-card.md) |
| Message envelopes | [03](spec/en/03-envelope.md) |
| Packet types | [04](spec/en/04-packets.md) |
| Receive and merge rules | [05](spec/en/05-rules.md) |
| Bluetooth | [06](spec/en/06-transport-bluetooth.md) |
| Nostr | [07](spec/en/07-transport-nostr.md) |
| Push API | [08](spec/en/08-push-api.md) |
| Local storage | [09](spec/en/09-storage.md) |
| Devices, planned | [10](spec/en/10-devices.md) |
| Profile relays, planned | [11](spec/en/11-relays.md) |
| v1 stable scope and open decisions | [12](spec/en/12-v1-stable.md) |

“Must” and “must not” define compatibility requirements. Descriptions of current iOS behavior record what the code does. Proposed changes and unresolved questions are listed in the specification; they are not implemented features.

## Test vectors

[JSON vectors](vectors/) contain Swift inputs and expected results. For randomized encryption or signatures, clients verify or decrypt the recorded output. [Shum Core](https://github.com/hiTechTeam/Shum-Core) pins a revision of this repository and checks compatibility against these vectors.

The generator is `ShumTests/Protocol/ShumProtocolVectorTests.swift` in [Shum iOS](https://github.com/hiTechTeam/Shum-iOS). Last full check: 21 tests passed at iOS revision `5b8efc8`. Regenerating vectors changes random signatures and nonces.

```sh
xcodebuild test -project ShumiOS.xcodeproj -scheme Shum \
  -destination 'platform=iOS Simulator,name=iPhone 17 Pro' \
  -parallel-testing-enabled NO \
  -only-testing:ShumTests/ShumProtocolVectorTests \
  -disableAutomaticPackageResolution -skipPackageUpdates
```

Run from Shum-iOS. Bluetooth mesh and Noise originate from [Bitchat](https://github.com/permissionlesstech/bitchat); revisions and licenses are recorded in the iOS repository.

## Related projects

[Shum Core](https://github.com/hiTechTeam/Shum-Core) · [Shum CLI](https://github.com/hiTechTeam/Shum-CLI) · [Shum iOS](https://github.com/hiTechTeam/Shum-iOS) · [Issues](https://github.com/hiTechTeam/Shum-Protocol/issues)

## License

[MIT](LICENSE), copyright 2026 hiTechTeam.
