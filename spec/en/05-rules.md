# 05. Receive Rules, State and Queues

English · [Русский](../ru/05-rules.md)

v1 draft, Shum-iOS 5b8efc8. Sources under Shum-iOS/ShumiOS/:

- Vendor/Messaging/ShumMessageStore.swift: receive, accept, carry, applyReceipt, receiveInvitation, receiveTyping, receivePresence, receiveReaction, receiveRetract, receiveProfile, send, tick, all route* methods, deleteConversation, setBlocked, retire.
- Vendor/Messaging/ShumConversationStore.swift: ShumContactsService, ShumConversationStore.transaction, local state models.
- Vendor/Messaging/ShumIdentityService.swift: ShumContactCard.preferred.

## 1. Trust and ingress

Decoding, signature verification and message acceptance are distinct stages. Raw BLE ShumPacket is at most 24 000 bytes. JSON must decode with version=1 and exactly one primary field from section 04.

Before dispatch, reject:

- A retired profile.
- A BLE source whose authenticated Noise key produces a blocked ID.
- A Nostr source matching a blocked card's nostrKey.
- A blocked card or either participant in envelope, receipt, invitation, typing, presence, retract or reaction. ProfileSync checks sender separately in its handler.

Then rate-limit by source and class: 80 packets per 60 seconds. Source key is peer.id ?? nostrSender ?? "unknown", followed by ":" + trafficClass. Classes are ephemeral for typing/presence, receipt for receipts, and content for everything else. At most 1000 distinct limiter keys; existing keys continue functioning at capacity. Reset the window when elapsed >=60 seconds. Remove old entries every 30 ticks. Structurally invalid packets do not spend quota; structurally valid packets with bad signatures do.

Untrusted-input errors are silently dropped without modal UI. Local storage/send failures are reported through onError. Persist message/receipt changes before an ACK or network action.

## 2. Pinned keys and cards

Shum ID = SHA-256(noiseKey). An existing identity's signingKey and nostrKey must not be replaced. Merge via preferred(over:) under section 02, then update the winning profile in contacts, encounters and requests together. A seed avatar removes an older stored PNG.

Accept BLE card only when session.noiseKey == card.noiseKey. It updates nearby and encounter; the first card triggers sending the local card. It does not automatically add an address-book contact. A Nostr card requires outer-authenticated sender == card.nostrKey; it updates known profiles and creates a legacy request for unknown senders in phase ready.

QR addition creates a verified contact and conversation in phase ready but **does not permit messaging**. canMessage == (phase == accepted). Address-book membership (metadata.addressBook != "false") and messaging permission are independent. The source="preview" exception is internal: fixtures immediately receive accepted; normal CLI does not use it.

Limits: 2000 contacts, 20 requests, 2000 blocked identities. Encounters remain visible for 24 hours after lastSeen; this is discovery history, not current connectivity evidence. Nearby is cleared against the actual connected-peer set.

## 3. Receiving a message addressed locally

1. Validate envelope under section 03. For Nostr, sender.nostrKey must match the authenticated seal author.
2. Match recipient to ownCard across all three keys. BLE requires 1 <= hopCount <= hopLimit; Nostr ignores it and records zero.
3. Active deletedMessageIDs[id] rejects even an otherwise valid replay.
4. For an existing inbound message with the same id and sender.id, check pinned signingKey/nostrKey, digest and accepted. If they match, repeat ACK without decrypting or changing text/history order.
5. Otherwise open Noise X; the returned static sender must exactly match envelope.sender.noiseKey.
6. Decode ShumPlaintext; compare current/legacy protocolName, id, conversationID, senderID, recipientID, timestamp and expiresAt to envelope. Validate text/reply under section 03.
7. For an unknown author, create a request but do not save the message. For a known contact, check pinned keys and accepted, then merge profile. Reject a different ciphertext with an existing id.
8. Create the conversation if needed; persist delivered, outgoing=false. unread=false only when the app is foreground and this exact chat is open; otherwise true. Transport is nostr, mesh or ble.
9. Create a signed receipt, with read=true only for a visible active chat. If a BLE route to the sender exists, issue immediate ACK after persistence.

Receipts/profile signals never reorder history. Replays do not create another bubble. IDs are case-sensitive strings even if two UUID spellings represent the same 16 bytes.

## 4. Courier storage

An envelope for another recipient is accepted only from a verified nearby BLE depositor whose session Noise key matches. Nostr cannot create courier ingress. Require 1 <= hopCount < hopLimit, no id in seenRelay, no existing relay-copy and no sufficient receipt. Limits: 64 envelopes, eight per depositor, 4000 seenRelay IDs. A stored seenRelay expires with the envelope; deleting its copy does not allow the known event to be accepted again.

Couriers neither decrypt ciphertext nor alter signed envelopes. Forwarding increments external hopCount by one. Retry a direct recipient no more often than every 30 seconds. Without a direct recipient, offer to at most three new neighbors, excluding original author and recipient, sorted by peer.id. At most eight transmissions per routeRelay. A genuine recipient receipt removes the relay-copy and may itself propagate through the graph.

## 5. Invitations

Phases: ready, outgoingPending, incomingPending, accepted, declinedByPeer, declinedLocally. Without a state record, a requests entry implies incomingPending; otherwise ready.

| Local action | Allowed phase | New phase |
| :--- | :--- | :--- |
| sendInvitation | ready | outgoingPending |
| acceptInvitation | incomingPending or declinedLocally | accepted |
| declineInvitation | incomingPending | declinedLocally |

Create contact/conversation if needed; commit the signed control and phase transition together. Retain the latest local invitation per recipient in outbox. Local time is strictly above previous updatedAt and, when needed, the peer signal time.

Ingress must validate and match all three recipient keys to local keys. Bind sender to BLE/Nostr as in section 04. Merge sender profile before phase processing. Ignore an exact replay of the last eventID. Avatar attachments are currently ignored and stored as nil.

| Incoming action | Allowed current phase | Result |
| :--- | :--- | :--- |
| request | ready, outgoingPending, declinedByPeer, legacy incomingPending | incomingPending, requests <=20; remove local request from outbox |
| accept | outgoingPending, declinedByPeer, accepted, incomingPending with evidence of local request | accepted; create contact/conversation; remove requests and local request |
| decline | outgoingPending, declinedByPeer, incomingPending with evidence of local request | declinedByPeer; remove requests and local request |

Exceptions:

- Simultaneous request: when already outgoingPending and own.id < sender.id, ignore the peer request. The smaller stable ID remains initiator.
- While accepted, a new request with timestamp > current updatedAt triggers another local accept with a time above both sides. An older request has no effect. This recovers from earlier local clearing.
- Legacy incomingPending is identified by the legacy- eventID prefix. For accept, an incorrectly reversed legacy request is also recovered if contact metadata.source == "chat-invitation".
- Other branches have **no general (timestamp, eventID) comparison**. Do not import reaction newest-wins ordering here. An older admissible accept may overwrite updatedAt in an already accepted state; see questions.

A legacy card creates a request only from ready, at most 20. updatedAt=now, eventID=`legacy-` plus UUID; remove local requests. A normal nearby card does not accept an invitation.

## 6. Typing, presence and reactions

All require correct recipient across three keys, transport-bound sender, a known contact with pinned keys, and accepted. Typing/presence additionally compare pinned nostrKey. Reaction explicitly checks noise/signing; authenticated Nostr author was checked separately earlier.

Newness: greater timestamp, or equal timestamp with lexicographically greater string id. Ignore equal/older signals. Compare UUID strings without conversion to 16 bytes. Typing/presence retain ordering in process memory; reactions persist it in the database.

- Typing=true sets typing deadline=expiresAt and extends presence to now+40 seconds. false clears typing; tick clears expiry.
- Presence=true sets deadline=expiresAt; false clears presence.
- Reactions may arrive before messages. If the message exists, conversationID must match these participants' conversation. At most 20 000 distinct messageID entries. Persist nil with timestamp/eventID so an old reaction cannot reappear. receiveReaction has no separate deletedMessageIDs check.
- Locally toggling the same reaction removes it; another replaces it. timestamp strictly increases, and outbox retains the last choice per messageID. Push only for an inbound message and only after the choice has persisted for five seconds.

## 7. Retraction and local clearing

cancelSending applies only to one's own queued/forwarding message. If it has left the device, indicated by attempts>0, nostrAccepted, lastNostrAttempt or forwardedTo, create retract until the original expiresAt. Otherwise local deletion suffices. Delete the message and related reactions together.

Receiving retract checks recipient, transport and known noise/signing keys. **accepted is not required**. An existing message must be inbound from this sender; without a message, a tombstone is allowed. Delete message, receipts and reactions; set deletedMessageIDs[id]=expiresAt. A late old retract may replace a farther expiry with a shorter one: this branch does not use max or newest-wins.

Local clear retains contact and accepted phase. For known unexpired messages, create tombstones until envelope.expiresAt. Delete conversation history, reactions, recipient reactionOutbox, legacy history, participant receipts and conversation. Send no peer-control. A new ID from the retained accepted contact can recreate a conversation. removeContact additionally removes contact, requests, invitationState/outbox, profileOutbox and pins.

## 8. Blocking and profile retirement

Blocking stores a verified card in blocked. Remove requests; invitation/profile/reaction outbox; relay entries involving the participant or depositor; participant receipts and directory pins. Mark own queued/forwarding messages to that contact cancelled. Retain existing history and invitationState. Remove the nearby card. Unblocking removes only the blocked entry.

retire sends offline presence, stops internet, drains its persistent records, and clears callbacks and ephemeral caches. Every late callback checks retired and must not restore removed profile data.

## 9. Receipts and delivery order

First validate receipt under section 04. For a local outbound message, check digest, original recipient.id/signingKey and destination.id==own.id. An unrelated node's signature does not authorize deleting a courier envelope.

Match proof to the original message or relay-copy by digest, recipient.id/signingKey and original sender.id==receipt.destination.id. If the original no longer exists, allow only a read upgrade of an existing identical verified receipt. Reject unsolicited receipts lacking both original and previous proof.

At most 2000 receipts, except updates to existing keys. Repeated false does not replace false/true; repeated true does not replace true. Read is monotonic: later false cannot downgrade it. Persisting receipt and removing relay are atomic; outbound deliveredAt/readAt come from timestamp. MarkRead updates flags and creates batch ACKs while retaining original message order. After message expiry, create no new receipts; clear local unread.

## 10. Queues and retries

Persist every attempt counter/time **before** transmission. Nostr uses separate lastNostrAttempt/nostrAttempts. Its general delay:

```text
attempts nil/0: 0 seconds
attempts >=1: min(60, 5 * 2^min(attempts - 1, 4))
# 5, 10, 20, 40, 60, 60, ...
```

| Queue | Per pass | BLE | Nostr / completion |
| :--- | ---: | :--- | :--- |
| Own messages | 4 | min(60, 10*max(1,attempts)) seconds; direct recipient or up to three distinct couriers | Until relay OK, then await receipt |
| relay | 8 | Direct recipient every 30 seconds; otherwise up to three new neighbors | Courier does not forward through Nostr |
| receipts | 8 | Every 30 seconds, direct destination and up to eight new neighbors | Own ACK only, until OK |
| invitation | First 4 | 10, 20, 40, then 60 seconds; replies at most six offers | Until OK/expiry; request has no six-attempt BLE limit |
| retract/reaction | First 8 | No more often than ten seconds, at most six offers | Until OK/expiry |
| profileOutbox | 4 | When a direct peer is available | Retry until signed profileID ACK |

Invitation BLE delay is min(60, 10 * 2^min(max(attempts - 1, 0), 3)) seconds. request continues beyond six BLE retries; accept/decline stop at six. This is routeInvitationControls, distinct from generic route.

profileOutbox delay is min(3600, 5 * 2^min(attempts,10)) seconds; attempts caps at 20. Relay acceptance does not complete the profile queue.

After BLE/courier transmission an own message becomes forwarding with ble/mesh transport; relay acceptance gives forwarding/nostr. **Delivered** requires a recipient receipt. With BLE also available, delay push eight seconds awaiting ACK; otherwise trigger after relay OK. ACK before the delay ends cancels push.

tick deletes expired relay/receipts/controls and marks queued/forwarding messages expired. delivered/read history is not deleted by expiration. Resending expired creates a new message at the end and deletes the old one after successful send. tick also clears seenRelay/deleted tombstones. The normal own-undelivered queue caps at 200 messages; trimmed text is <=4096 bytes.

## 11. Profile synchronization

All transports use one head: section 02 revision/profileID. Changing local name/bio/seed increments UInt64 revision by one, with an error at max, saves ownProfileCard and replaces profileOutbox for every unblocked contact. Initialization restores the saved head while checking all local keys.

Incoming sync requires recipientID==own.id, a known unblocked contact and sender matching BLE Noise or authenticated Nostr. accepted is not required for profiles. Merge sender through pinned-key preferred.

If knownRecipient describes the same local profileID, remove the corresponding pending ACK. If a peer retained a newer/preferred local head after backup rollback, iOS retains local name/bio/seed choices and re-signs them with revision > peer.revision. requestsReply triggers a reply with reply=false over the incoming transport. ProfileSync has no lifetime.

## Test vectors

The generator uses real ShumMessageStore, Noise X and Swift signatures, fixed time and in-memory keychain/store/transport. Radio and Nostr connections are not started. Random outbound UUIDs are recorded in fixtures as concrete Rust inputs, not expected to be regenerated.

| File | Scenarios |
| :--- | :--- |
| 05-delivery.json | accepted/pending/blocked, sender pinning, legacy/plaintext/reply boundaries, duplicates, conflicting digest, clear/replay |
| 05-invitations.json | 18 phase transitions, simultaneous-request tie by stable ID |
| 05-newest-signals.json | Reaction ordering/removal tombstones; typing/presence with equal timestamps and differing UUIDs |
| 05-invitation-retries.json | request/decline BLE schedule, six-reply limit, continuing requests |
| 05-outbox.json | Real retry intervals, relay OK versus receipt, delivered-to-read without rollback |

## Questions

During Rust implementation an earlier draft error was corrected: invitations had been documented as eight packets at constant ten-second intervals, but current Swift takes the first four with exponential BLE delay. iOS behavior did not change. 05-invitation-retries.json verifies the actual schedule.

1. **No general newest-wins for invitation.** Except the special accepted/request branch and exact eventID, timestamps do not filter old accept/decline. This may roll back state metadata. Port actual v1 branches; coordinate algorithm changes separately with iOS.
2. **Retraction can shorten a tombstone.** A late valid retract overwrites expiry without max. Do not silently fix this in the v1 description.
3. **Reaction after deletion.** Without a message, a reaction may be accepted in advance even when its ID is in deletedMessageIDs. Message history stays removed, but reactions may accumulate again.
4. **Ephemeral ordering is not persistent.** Restart permits an older still-unexpired typing/presence signal. Evaluate UX without changing v1 wire formats.
5. **No cross-device local-clear sync.** v1 retains accepted and deletes only local history. New devices and clear-all require an agreed v2.
