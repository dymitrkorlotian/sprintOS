# The sync protocol: identity, order and conflict rules

Supports [ADR-0001](../decisions/0001-founding-architecture.md) → section 3. Written 2026-10-02 in answer to the red-team review of PR #1. This is the contract spike S1 must implement and test. It is a draft until S1 passes.

## 1. Goals

- Every device that has the same set of events computes the same tables, byte for byte, **whatever its wall clock says and whenever the events arrived**.
- An event the user was told is saved is never lost and never silently changed.
- Every user intent either takes effect or shows up as a visible, restorable conflict.
- A blind hub or a stolen device can't rewrite history, and withholding is detected.

## 2. The event

```
header (plaintext to a blind hub, signed):
  workspace_id, device_id, seq          -- seq: gap-free per device, starts at 1
  prev_hash                             -- hash of this device's previous event: a per-device hash chain
  epoch_id                              -- which workspace key encrypts the body
body (encrypted, signed together with the header):
  id            UUIDv7, unique per event
  hlc           hybrid logical clock (wall ms, counter), as written by the author
  deps          the author's version vector when it wrote the event (device → seq)
  type, schema_version
  actor         user / system / ai / automation
  logical_key   optional: (kind, key) for "one per thing" artefacts, e.g. ("retro_draft", 2026-W41)
  payload
signature: Ed25519 by the device key over header + body
```

- **Hashing and signing** cover the whole event, so nothing in it can be changed by a relay. Receivers never rewrite an event, so the sort key is never local.
- **Causal delivery:** a device applies an event only after it has every event named in `deps`. Sync sends events in an order that respects this.

## 3. Order

- **The order is a pure function of signed fields:** `(hlc, device_id, seq)`. Receivers never clamp or adjust it, so two devices that receive the same events at different wall-clock times sort them the same way.
- **HLC rules on the author's side.**
  - A device's HLC never goes backwards. An event whose `hlc` isn't greater than its own previous event's (`prev_hash`) is invalid everywhere: it goes to the conflicts table, deterministically, because the rule only reads the log.
  - When merging remote clocks, a device advances its own HLC from a remote event by at most 1 hour beyond its own wall clock. That is a local rule, and it only affects events it writes later, so it can't cause divergence. It stops one wrong clock spreading to every device.
- **Clock-skew detection.** Devices compare wall clocks on every direct contact, and through signed heads (section 7). A device more than 5 minutes off is shown a warning, and its events are marked in history as "from a device with a wrong clock". We don't try to repair the past automatically: a wrong clock is a user-visible problem, as it is in every system that uses wall time.
- **Domain time.** Which sprint week an action belongs to is decided by its `hlc`, converted with the workspace time zone in effect at that point in the log. UTC instants compared with a boundary computed from log state are unambiguous, whatever zone the device is in.

## 4. Rituals and other derived facts are not events

- **The week close is derived.** The reducer keeps a **log time**, the highest `hlc` it has applied. When log time passes the start of week W+1 (Monday 00:00 in the workspace zone in effect), the reducer applies the close of W: completed, carried over, missed. This is a pure function of the log, so there is no event to duplicate and no fixed-id collision.
- **Advancing log time.** When a device's clock passes a boundary and its log time hasn't, it writes a tiny `tick` event. Ticks from several devices are harmless: they only advance log time. This keeps Sprint's lazy, idempotent behaviour: nothing happens while nobody uses the app, and the first device that does use it closes the week.
- **Recurring occurrences are derived the same way.** They appear when log time reaches their scheduled moment, with a deterministic id derived from the series and the occurrence time. Edits to an occurrence are ordinary events that reference that id.
- **AI and human artefacts are events with a `logical_key`** (a retro draft, a summary, a plan). If two devices both write one, both events are kept, each signed by its author and each a normal step in its device's seq. The reducer shows the first in total order and keeps the rest as alternatives. No seq gaps, no dedupe by id.
- **A late event** whose `hlc` is before a close that has already been applied makes the reducer rewind to the checkpoint before it and replay. Checkpoints are kept at least at every week boundary.

## 5. Conflicts: every intent takes effect or becomes visible

**Invariant (tested in S1):** for every user-authored event there is either its effect in the tables or a row in `conflicts` holding the original payload, a reason, and a one-click restore or dismiss.

| Case | Rule |
|---|---|
| Cardinality (one active `in_sprint` per task and sprint; one parent; one project; a custom relation's *one* side) | In total order, the later edge wins. The earlier one is ended with `outcome = superseded` and gets a conflict row |
| A cardinality setting changed from many to one while new links arrive | After the change, the reducer ends all but the latest active edge per source and records each as a conflict |
| Tree move that would create a cycle | Skipped (Kleppmann's move algorithm), with a conflict row |
| Sibling order in the page tree | Fractional-index positions, ties broken by event id |
| A field's type changed (e.g. number → select) while a late value arrives | The value goes through the same conversion as existing values; if it can't convert, it becomes a conflict row with the raw value |
| A write to an archived field, option, type or object | Applied: archive is soft and nothing is hard-deleted. The object shows "edited after it was archived" |
| A page body edited on one device while the page is archived on another | The body merges (Loro); the page stays archived, with a notice |
| A recurrence rule edited while occurrences exist | Occurrences already reached in log time keep their ids and edits; the new rule applies from its own place in the order |
| An event of a type or version this device doesn't understand | Kept in the log, not applied, and shown as "update the app" (section 6) |

## 6. App versions

- **Event shapes only grow.** Old versions are upcast. A type or field key is never reused.
- **`min_reducer_version` events.** A device that sees one above its own reducer version:
  - becomes read-only, except for plain field and body edits, which commute;
  - gives up any job lease, so it can't run rituals or AI on the wrong state;
  - asks for an update.
- **After upgrading**, it rebuilds its tables from the log.
- **Convergence is required after both devices upgrade**, not while they run different reducers. The S1 property is: "after every device upgrades, the tables are byte-identical, and no write made by the old version broke an invariant".

## 7. Durability, identity and tamper evidence

- **Nothing is sent before it is durable.**
  - The events table and the projection commit in one transaction, with `synchronous=FULL`. On macOS, `fullfsync` is also on (`F_FULLFSYNC`), because a plain `fsync` there doesn't flush the drive cache.
  - An event leaves the device only after its commit has returned.
  - Spike S3 measures the cost on a real Mac SSD. If a commit is too slow, commands are group-committed (batched within about 50 ms), never made less durable.
- **Restore and migration.** The device key is a `ThisDeviceOnly` Keychain item, so it doesn't follow a backup restore, Migration Assistant or Time Machine. On open, the core checks that the key matches the device record in the database. If it doesn't, or if peers report a higher seq for this device than the local log holds, the core:
  1. fetches the old device's events from peers;
  2. freezes the old device id at the last seq peers agree on;
  3. mints a **new device id**.

  Device keys are never synced through iCloud Keychain. Only the workspace key may be, and only as an opt-in.
- **Equivocation is an alarm.** Two different events with the same `(device, seq)` can only come from a cloned or compromised device. Peers freeze that device at the last seq where the hash chain still agrees, record the event as a security alert, and ask the user.
- **Revocation by seq cutoff.**
  - "Revoke device D after seq N, hash H" is a signed event from another device.
  - N is the highest seq any surviving device has seen from D.
  - Every peer, and the blind hub (which can check header signatures), rejects D's events after N, whatever their `hlc`. So a stolen device can't backdate writes.
  - The key rotates at the same time (section 8).
- **Withholding is detected.**
  - Each device regularly signs a **head**: device id, max seq, the hash of that event and its wall time.
  - Heads travel with sync and through the mailbox.
  - A device compares the heads it gets from authors with what the hub delivered. A hub that hides B's last events from A is caught on A's next direct contact with B, or through any channel that carries B's head.
  - The per-device hash chain means a gap can't be papered over.

## 8. Keys and epochs

- **Epochs.** An epoch id is `hash(parent epoch id ‖ commitment to the new key)`.
- **Rotation** (on revoking a device, or untrusting a hub) is a signed event that wraps the new key to every remaining member with HPKE.
- **Forks.** If two devices rotate offline, there are two child epochs.
  - Both stay readable to their members.
  - The next device that sees both performs one more rotation from the fork, excluding every device revoked in either branch. If that also forks, the same rule repeats.
  - The revoked set only grows, so this converges.
  - Until then, devices write under the fork that comes first in total order.
- **Libraries:** RustCrypto for XChaCha20-Poly1305, Ed25519, X25519 and Argon2, plus the `hpke` crate. Audit coverage is partial: an older external audit of `chacha20poly1305`, none for `hpke`. Our key-handling module gets an external review before the hosted hub launches. aws-lc-rs is not a drop-in: it has no XChaCha20-Poly1305 and no public HPKE.

## 9. Undo

- Undo is an intent: `revert(event_id)`.
- The reducer resolves it at its own place in the order. A field goes back only if it still holds the value the reverted event set; otherwise the revert becomes a conflict row. Reverting a cardinality move works the same way.
- The UI offers undo for the device's own recent events. Older history is restored through "restore this version", which is also an ordinary event.

## 10. What S1 must test (in addition to the ADR's list)

- Events received at different wall-clock times sort the same.
- Clock skew of ±1 day in both directions, including a device whose HLC tries to go backwards.
- Two devices produce the same `logical_key` artefact offline, and no seq gap appears.
- Seq reuse after power loss, a backup restore and a Migration Assistant clone: detected, with a new device id and no lost events.
- A revoked device's backdated events are rejected by every peer and by the blind hub.
- A hub that hides one device's last 10 events from another is detected within one direct contact.
- Concurrent key rotation on two offline devices converges, and revoked devices can't read the result.
- `revert` racing a concurrent edit gives a conflict, not a silent overwrite.
- Every user-authored event leads to an effect or a conflict row.

**Prior art to read before building:**
- Evolu (MIT): SQLite with end-to-end encryption and a self-hostable blind relay.
- LiveStore (event-sourced SQLite with rebase).
- Kleppmann et al., "A highly-available move operation for replicated trees" (IEEE TPDS 2021).
- Kulkarni et al. (2014) for HLCs.
- Agent 6's prototype (sprint#57 handoff, 2026-10-02): a reducer converged in every seeded run *given one order from a sequencer*.

**Caveat on that prototype:** it doesn't cover the hard cases here, which come from having no sequencer.
