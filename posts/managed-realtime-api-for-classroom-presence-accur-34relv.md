# Managed Realtime API for Classroom Presence — Accurate Hand Raises Without Servers

Presence accuracy matters more than transport novelty when a property management team runs a virtual classroom for resident orientation or staff training. The session needs a roster and hand raises, while its side conversation needs typing indicators and read receipts. **Use a managed realtime API for fan-out, but keep durable message state and authorization in the application backend.** Treat typing and a raised hand as expiring hints, a read receipt as a monotonic fact, and a roster entry as a lease rather than proof that a person is looking at the screen. That approach removes the need to operate a socket server without pretending that every connection event represents human attention.

TL;DR: issue short-lived, classroom-scoped credentials from the backend; publish ephemeral typing, hand-raise, and heartbeat events through the realtime layer; commit read progress through an authenticated backend command; and rebuild the visible roster from leases with explicit expiry. A Node.js application can own those commands without hosting the long-lived socket fleet. WebRTC is useful when a class, showing, or maintenance call needs peer media, but its specification does not make it the source of truth for message reads or application presence.

## Can a virtual classroom realtime API keep an accurate roster?

A browser can remain connected in a background tab. A phone can suspend an app without delivering a clean disconnect. A tenant can open the same classroom on two devices, while a leasing agent has three tabs open. Network transitions can also produce a brief overlap between an old connection and a replacement. Counting connections therefore answers a narrow question: how many transport sessions appear live? It does not reliably answer who is attentive, who belongs on the roster, or whose hand is still raised.

This distinction has consequences in property management. During a resident orientation, a hand raise must vanish after the participant leaves; in the related chat, a typing indicator must disappear if an agent closes a laptop mid-sentence. A read receipt attached to a maintenance access notice carries more weight: moving it backward after an out-of-order event would misrepresent what the recipient has seen. Compliance-aware systems should also avoid turning presence into a hidden activity log. Retain the durable event that the workflow needs, not every heartbeat emitted by every tab. The awkward edge case is a resident who raises a hand on a phone, opens the class on a laptop, then loses mobile coverage: collapsing state only by user can preserve a raise that belonged to the vanished phone, while showing every device can duplicate the resident. Keep the event scoped to its originating session, but fold the roster by user.

Three clocks are involved: transport liveness, user activity, and durable read progress.

They drift apart.

Design for that.

Use a lease for each active device session, keyed by user and session identifiers. A heartbeat renews the lease; missing renewals let it expire. The displayed roster folds multiple unexpired sessions into one user entry. Visibility and recent interaction can refine a label such as `active` or `away`, but those client claims remain advisory. They must never grant access.

Typing and hand raises are softer. Publish `typing_started` or `hand_raised` with a short expiry and let the UI clear the state either on an explicit stop/lower event or when the deadline passes. The expiry handles lost terminal events. Avoid persisting keystroke cadence, draft text, or an indefinite hand flag; none is required to render the feature.

## Separate hints from committed facts

The clean boundary is a small state model, not a collection of callbacks. Ephemeral events may be duplicated, delayed, or dropped without corrupting the record. Durable commands require authentication, membership checks, idempotency, and monotonic updates.

| Signal | Authority | Storage | Merge rule |
| --- | --- | --- | --- |
| Connection | Realtime transport | None or short operational log | Count distinct live session leases |
| Typing or hand raise | Current client session | Expiring memory | Latest unexpired state per session and classroom |
| Roster activity | Backend-issued session lease | Expiring store | Fold sessions by authorized user |
| Read progress | Application backend | Durable database | Keep the greatest authorized message position |

The read marker should reference a server-assigned message position, not a client timestamp. Client clocks disagree, and retries can arrive out of order. If a conversation has positions 40 through 43 and the backend has accepted 43, a delayed request for 41 is a successful no-op. It must not roll the marker back.

Here is the core behavior in Python. The storage interfaces are deliberately generic; the important part is where trust and ordering live.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone


@dataclass(frozen=True)
class SessionLease:
    conversation_id: str
    user_id: str
    session_id: str
    expires_at: datetime


def renew_presence(store, membership, *, conversation_id, user_id, session_id):
    if not membership.can_view(user_id, conversation_id):
        raise PermissionError("conversation membership required")

    now = datetime.now(timezone.utc)
    lease = SessionLease(
        conversation_id=conversation_id,
        user_id=user_id,
        session_id=session_id,
        expires_at=now + timedelta(seconds=45),
    )
    store.put_lease(lease, ttl_seconds=45)
    return lease


def mark_read(db, membership, *, conversation_id, user_id, message_position):
    if not membership.can_view(user_id, conversation_id):
        raise PermissionError("conversation membership required")
    if not db.message_exists(conversation_id, message_position):
        raise ValueError("unknown message position")

    return db.advance_read_position(
        conversation_id=conversation_id,
        user_id=user_id,
        minimum_position=message_position,
    )
```

The 45-second lease is an example policy, not a universal constant. Set it from the product's tolerance for stale names, background throttling, and heartbeat traffic, then test the setting on the devices the audience actually uses. Accuracy has two failure modes: a long lease leaves departed users visible; a short lease makes present users flicker during suspension or a weak cellular handoff. There is no honest setting that eliminates both.

## Choose the transport after defining the contract

Once the state boundaries are explicit, several managed transport shapes can carry the ephemeral stream. A broadcast channel is enough for typing changes and hand-raise events. Presence-aware channels may also report joins and leaves, but their member list should still be interpreted as transport state. Server-sent events provide a standardized one-way stream from server to browser through `EventSource`; client commands continue over ordinary authenticated requests. WebSockets, standardized by RFC 6455, provide a two-way channel over one connection. WebRTC supplies peer-connection facilities suited to media and data exchange, but signaling and application authorization remain separate concerns. In a Node.js backend, these choices affect the adapter and credential issuer; they should not alter the classroom state contract.

The choice is operational. Check whether the service can mint credentials restricted to one classroom, revoke or naturally expire them, resume after interruption, expose connection metrics, and define ordering boundaries clearly. Ask what happens when a subscriber reconnects after missing events. For ephemeral typing or a hand raise, accepting loss is often correct. For a read marker, fetch the durable current value again.

There is a real limitation: a managed realtime API is a poor fit when policy requires every connection and payload to stay on infrastructure the organization directly controls, or when the session must continue on an isolated network. Operating a self-hosted WebSocket server is then the more appropriate trade-off. Server-sent events are also unsuitable when the design specifically requires one bidirectional transport, while WebRTC adds peer and signaling complexity that text-only classroom state does not need.

**Do not use the realtime channel as an authorization oracle.** The application backend decides that a tenant, owner, contractor, or staff member may enter a conversation, and it issues a short-lived capability limited to that scope. Every durable mutation repeats the membership check. This contains a leaked credential and prevents a stale channel subscription from extending access after a role changes.

Payloads should carry opaque user and conversation identifiers, an event type, a server-understood schema version, and the minimum state needed to render. Email addresses, phone numbers, unit access notes, and message bodies do not belong in presence events. That rule reduces accidental exposure in logs as well as on the wire.

## Failure tests reveal the honest user experience

Happy-path testing barely touches presence. Open two tabs for one instructor, close one, and verify that the instructor remains once in the classroom roster. Raise a hand from one session and disconnect that session while another remains; the hand should clear without removing the participant. Suspend a tenant's phone beyond the lease, resume it, and confirm that the UI tolerates expiration and re-entry. Deliver `typing_stopped` before a delayed `typing_started`; the expiry must still clear the indicator. Reconnect after several missed events and ensure a snapshot replaces guesses based on an incomplete stream.

Then attack the durable path. Submit read positions 43, 41, and 43 again. The stored value should remain 43, and retries should not create multiple audit records for the same transition. Try a valid credential against another property's conversation. Reject it at both subscription issuance and command handling. Remove a contractor from the work order while a tab is open, then verify that credential expiry and backend authorization close the window without relying on a client disconnect.

Observe the system in terms users feel: lease expirations, reconnect duration, rejected unauthorized commands, duplicate event rate, read-marker lag, and snapshot failures. Do not label connection count as "online users." It is connection count.

A useful deploy test deliberately drops connectivity between publish and receipt. The UI should age out typing, mark its roster view as uncertain if a fresh snapshot cannot be obtained, and preserve the last confirmed read position. Quiet degradation is better than confident fiction.

## Roll out with reversible semantics

Start with read-only roster display for an internal property team, using leases and aggregate metrics but no durable activity history. Next enable typing for a narrow conversation type and confirm that stale indicators expire under suspension. Add read receipts last, after the monotonic database update, authorization audit, and privacy retention rule are settled.

Keep each phase reversible through server-controlled feature flags. Older clients should ignore unknown event fields, while newer clients must be able to rebuild from a snapshot rather than require a perfect event history. During migration, dual-reading the old and new read positions is safer than dual-writing ambiguous semantics; define one authoritative value and compare the other for discrepancies before switching readers.

The final decision rule is compact: **lease what may disappear, commit what users may rely on, and authorize both at the backend boundary.** A managed realtime API can remove socket fleet operations. It cannot decide what "present" means for a property conversation; that definition belongs in the application contract.

## Sources

- https://www.w3.org/TR/webrtc/
- https://www.rfc-editor.org/rfc/rfc6455
- https://html.spec.whatwg.org/multipage/server-sent-events.html
- https://developer.mozilla.org/en-US/docs/Web/API/Page_Visibility_API
