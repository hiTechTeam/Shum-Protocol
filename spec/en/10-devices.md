# 10. Multiple Devices

English · [Русский](../ru/10-devices.md)

**Part of v1 stable**, [section 12](12-v1-stable.md). Not implemented in the v1 draft or current iOS. Decisions below were agreed on 8 October 2026 and define the direction. Byte formats, domain labels and vectors will follow when this section becomes a draft. Relay settings/networks are in [section 11](11-relays.md).

## 1. Principles

1. **Devices are peers.** No primary device. A profile can be created on any client: phone, desktop app, browser or CLI.
2. **Any linked device can link another**, not only a phone.
3. **QR linking works in both directions:** computer to phone, phone to computer, phone to phone, computer to browser. Direction depends on camera availability, not where the profile was created; subsection 4.
4. **Unlinking actually revokes access** to new messages; subsection 6.

## 2. Keys, option B

Two key groups:

| Keys | Location | Purpose |
| :--- | :--- | :--- |
| **Profile keys:** Noise, signing and Nostr from section 01 | Every full profile device | Identity, Shum ID, signing profile/device registry, relay address, authority to link devices |
| **Device keys:** own Noise and signing keys | Only that device | Self-channel, subsection 8; device sessions; revocation on unlink |

- Shum ID stays SHA-256 of the profile Noise key. Draft profiles migrate without ID changes or new contact verification; existing keys become profile keys.
- The **profile device registry** is signed by the profile signing key and has a revision. It stores device keys, name, client type and linking date.
- Profile Nostr key is shared by all devices. A relay checking NIP-42 admits them without a separate device list; section 11.

## 3. Linking roles

| Role | Participant | Action |
| :--- | :--- | :--- |
| **With profile** | Any linked device | Approves linking and transfers profile |
| **New** | Device without profile | Receives profile |

Approval always happens on the device **with the profile**. The new device only displays a code and waits.

## 4. QR in both directions

The device without a camera, or for which scanning is inconvenient, displays the code. The camera-equipped device scans it.

| Profile exists on | New device | Displays QR | Scans | UI |
| :--- | :--- | :--- | :--- | :--- |
| Phone | Computer, browser, CLI | New device | Phone | Computer: “Link to another device”, shum link; phone: Devices › Link device › Scan QR |
| Computer, browser, CLI | Phone | Computer | New phone | Computer: Devices › Link device › Show QR, shum devices add; phone start screen: Sign in from another device |
| Phone | Another phone | Existing phone | New phone | Show QR and Sign in from another device |
| Computer | Another computer/browser | Either | Question 1 | Short-code entry instead of camera |

The flow is identical in both directions:

1. One displays QR; the other scans. QR contains a one-time linking-channel key and rendezvous address, **never profile keys**.
2. Establish an encrypted channel and show the **same six-digit confirmation code** on both devices, derived from channel keys.
3. The user compares it and approves **on the device with the profile**.
4. The new device creates device keys and sends public keys.
5. The existing device signs the updated registry and transfers profile keys, registry, Profile Relays record (section 11), contacts, chats and the last 30 days of history.
6. The new device acknowledges receipt. The registry propagates to all profile devices through the self-channel, subsection 8.

QR and confirmation code are short-lived; mockups refresh them once a minute.

## 5. Nearby

Current CLI mockups show only the phone advertising after linking: one nearby card per profile.

Proposed v1 stable behavior: several profile devices may advertise; receivers merge them into one card by Shum ID. This avoids choosing a primary device and requires a decision, question 3.

## 6. Unlinking

1. Any profile device removes a device and signs a new registry revision.
2. Distribute it to own devices and contacts.
3. The removed device knows profile keys, so **rotate the profile Nostr key and relay-network passes** after unlinking; section 11. Shum ID/contact verification stay unchanged.
4. If stolen-device compromise may expose profile keys, the user can **rotate all profile keys**. This creates a new Shum ID and resets contact verification. Shum must explain this honestly.

## 7. What synchronizes

| Item | Method |
| :--- | :--- |
| Profile device registry | Automatic self-channel |
| Name, bio, avatar | Automatic self-channel and contact profile update |
| Profile settings, including relays/networks, section 11 | Automatic self-channel |
| Chat history | **Manual** shum sync / Synchronize in device card, over Bluetooth without QR |
| Clearing chats | Choose this device or all devices; locally cleared history returns through manual sync |
| Device connectivity, Tor, proxy | Not synchronized, section 11 |

## 8. Self-channel

Envelopes addressed to own devices and encrypted to their device keys. Publish to every profile mailbox relay, including transitional old relays under section 11.

Every service message carries registry and Profile Relays revision numbers. A device observing a newer revision requests the latest record itself.

Message types:

- New registry/settings revision.
- Revision applied acknowledgment with its number.
- Device state: reachable relays and last-online time.
- Request for the latest record.

Repeat a new revision until every device acknowledges it.

## Questions

1. **Computer to computer.** Laptops have cameras, Mac mini/servers may not. Is short-code entry needed instead of QR, and with what length?
2. **Link channel.** Transfer by Bluetooth, as desktop mockups require nearby devices, or via relay with QR key? Relay is convenient; Bluetooth leaves no server traces.
3. **Multi-device nearby.** Keep phone-only advertising or merge by Shum ID, subsection 5?
4. **Envelope encryption.** Encrypt to profile key as in v1, or separately to each device? Per-device encryption gives real revocation without rotating Nostr key but needs an up-to-date recipient device registry.
5. **Draft-profile migration.** If a draft build ships, migrate its profiles under section 12 item 8. Test database/key migration.
