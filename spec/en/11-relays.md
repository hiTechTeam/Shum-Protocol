# 11. Profile Relays and Relay Networks

English · [Русский](../ru/11-relays.md)

**Part of v1 stable**, [section 12](12-v1-stable.md). Not implemented in the v1 draft or current iOS. Decisions agreed on 8 October 2026. [Section 10](10-devices.md) defines devices, keys and the self-channel.

## 1. Current v1 behavior

- All clients embed the same four public relays, section 07. Sender/recipient share that list, enabling delivery.
- Custom relays are local to each device and are not transmitted.
- Contact cards contain no relays.
- Push server only wakes the device, section 08, independently of relays.

If a user keeps only a private relay, messages stop arriving: contacts publish to their own relays rather than that user's relay.

## 2. Three settings levels

| Level | Contents | Synchronization |
| :--- | :--- | :--- |
| **Profile** | Mailbox for receiving messages; relay networks/passes; network chosen for each contact | Automatic to every profile device |
| **Device** | Direct/Tor/proxy connection; extra device-only relay; do not use this relay here | None |
| **Contact** | Contact mailbox and usable networks | Received in contact card |

All profile devices share the mailbox; only connection methods differ, avoiding receive-configuration drift.

A device may disable a mailbox relay only if at least one other mailbox relay remains reachable. Otherwise the client refuses and explains why.

## 3. Profile Relays record

One record per profile, signed by its signing key, with a revision like profileRevision.

Contents:

- **Mailbox:** relay addresses, active/read-only mode and read-only transition time.
- **Networks:** relay addresses, network key and pass, subsection 7.
- **Routes:** chosen network per contact.
- Private-relay passwords/tokens stored only here, encrypted to own device keys. Contacts do not receive them.

Each item records added/removed state, change time and originating device.

**Merge:** disconnected concurrent edits merge per item; the later change wins for that item. Assign the next revision to the result. Equal-time resolution is open, question 2.

## 4. Propagating changes to own devices

Example: a profile links iPhone, MacBook, Mac mini CLI and browser; mailbox has four public relays, revision 7.

1. **Edit:** iPhone adds wss://my.relay, creates revision 8 and signs with the profile key.
2. **Send over the old path:** publish the self-channel revision to **all** relays, four old and one new. Other devices still listen only to old relays, so the change must arrive there.
3. **Online devices:** MacBook/Mac mini verify the profile signature and 8 > 7, apply, connect to my.relay using NIP-42/profile Nostr key, and reply “revision 8 applied, my.relay reachable” or “unreachable.”
4. **Offline device:** iOS subscriptions fetch only three days, section 07, and public relays may delete old data. Therefore repeat until all devices acknowledge; carry revision numbers in every service message so stale devices request the latest record; offer manual Bluetooth sync as fallback.
5. **Devices UI:** every device shows who applied the revision, who has not, and which relays each cannot reach.

The weak case is a long-offline device after all old relays disappear with no Bluetooth recovery. The transition period addresses this, subsection 6.

## 5. Informing contacts

- Mailbox becomes a contact-card field; changes use ordinary profileRevision/profileOutbox updates, section 05.
- Send to every recipient mailbox relay and duplicate to own relays.
- Draft-v1 clients do not know the field and publish to embedded relays. While such clients remain in circulation, retain at least one embedded public relay in the mailbox, question 4.

## 6. Switching relays without loss

A removed relay first becomes read-only: devices keep reading it while the card marks it obsolete. Remove it after every contact acknowledges the new card, or after 30 days at the latest.

## 7. Relay networks

A user can operate one or more relays and invite others to communicate through them.

1. **Create:** choose network relays and create a network key. Store in Profile Relays and propagate to all own devices.
2. **Invite:** link/QR like a contact invitation, carrying relay addresses and a network-key-signed pass.
3. **Join:** add to the invitee's **profile**, not device; it appears on all devices.
4. **Relay admission:** NIP-42 authentication checks profile Nostr key against network membership. Because devices share that key, section 10, no per-device enrollment is needed.
5. **Route:** a shared network is selected for two members' conversation; other contacts use the ordinary mailbox.
6. **Remove member:** owner removes its key and issues new passes to the others.
7. **Unlink a member's device:** after profile Nostr rotation, section 10 item 6, the member updates its key in networks.

## 8. Push

Unchanged: server wakes the device, which reads messages from its relays itself.

## Questions

1. **Format:** Profile Relays fields, signature domain label such as shum.profile-relays.v1\0, mailbox card field.
2. **Equal-time changes:** choose winner by device ID, or let removed beat added?
3. **Mailbox visibility:** everyone receiving a card sees it. Hide some relays, such as closed-network relays, from contacts?
4. **Draft clients:** must a mailbox retain an embedded public relay after v1 stable? If no draft build ships, this issue disappears.
5. **Network invitation format** and pass lifetime.
