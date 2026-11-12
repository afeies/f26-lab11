# EVENTS.md

One section per milestone. Fill each one in order as you go.

The prediction in milestone 2 should be written before you run the replay. That is the
part a TA will ask about.

Keep it short and specific. Point at operations, call sites, and what a caller
gets back.

---

## Milestone 1: what runs in the request path, and what runs after it

### The operations table

One row per operation. Read the code, not the file
names. "As wired today" is what the code does now. "Your call" is what you
would do if this were your service. List up to eight operations, and remove
the rows you don't use.

| # | Operation | As wired today | Your call | Why (in one line) |
|---|---|---|---|---|
| 1 | Validate, conflict-check, and save the booking (`BookingService.book`) | Synchronous | Synchronous | The caller needs a yes/no and a booking id before the page can respond. |
| 2 | Index the booking for the room-finder (`searchIndex.indexBooking`) | Synchronous, in `book()`, before the save | Event-driven (subscriber on `booking.confirmed`) | The room-finder can lag by minutes; a booking must not fail because search is down. |
| 3 | Audit record (`audit.record`) | Synchronous | Synchronous | Local, cheap, and must exist exactly when the write does. |
| 4 | Confirmation / release email (`EmailConfirmationHandler`) | Event-driven | Event-driven | The organizer needs it soon, not before the response; a mail relay hiccup should not undo a booking. |
| 5 | Receipt rendering (`ReceiptGenerator`) | Event-driven | Event-driven | Downloaded later; it can be rendered after the booking exists. |
| 6 | Reminder scheduling (`ReminderScheduler`) | Event-driven | Event-driven | Fires 30 minutes before start, so there is plenty of slack. |
| 7 | Dashboard stats (`StatsUpdater`) | Event-driven | Event-driven | Recomputed from the store; staleness is harmless. |
| 8 | Refund: ledger entry + payment provider (`RefundLedger`) | Event-driven | Event-driven, with an idempotent handler | The caller needs the amount owed now, not the money moved; a provider outage must not block cancelling. |

### Two calls in detail

**Row 2, indexing the booking: event-driven.** The caller of `book()` (the
reservations page or the kiosk) needs to know, right now, whether it holds the
room. It does not need the room-finder to show the booking, which only matters
to someone searching later. Today the index write runs inside `book()` before
the booking is saved, so when the search cluster is down the whole booking
fails: the caller gets a `SearchIndexUnavailableError`, nothing is saved, and
no `booking.confirmed` is published. Moved to a subscriber on
`booking.confirmed`, the booking succeeds during an outage, the index
delivery dead-letters, and someone redelivers it (or a backfill reindexes)
when the cluster returns. The cost is that the room-finder is stale for the
outage window, which the search team already announces as blank anyway.
Re-indexing is a keyed `set`, so redelivering is harmless.

**Row 8, the refund: event-driven.** The caller of `cancel()` needs the room
released and to be told how much they will get back; `cancel()` already
computes that amount synchronously and returns it. Actually writing the
ledger entry and calling the payment provider can happen after the response.
If it were synchronous, a provider outage would turn into "you cannot cancel,"
which is the same mistake as the ticket, aimed at money. The price of making
it an event is that the bus delivers at least once, so the handler must be
safe when it sees the same refund message twice (milestone 2), and a failure
mid-delivery needs a recovery path (milestone 3).

### The incident

**The call behind `TICKET.md`.** `src/service/bookingService.ts`, method
`BookingService.book`, line 94:
`this.searchIndex.indexBooking(booking, room, now);`. When the cluster is down
it throws `SearchIndexUnavailableError`. Nothing in `book()` catches it, and
it runs before `this.store.save(booking)` (line 95), so the caller gets that
exception instead of a `Booking`: no booking is saved, no audit record is
written, and `booking.confirmed` is never published. `cancel()` never touches
the index (the nightly sweep removes rows and defers on outage), which is why
cancellations and mail for older bookings kept working.

---

## Milestone 2: what a second delivery does

### Your prediction (before you run anything)

**What you expect `redeliver` on a refund message to do.** Name what you expect
to change in the ledger, at the payment provider, and in the bus log. Write it
down, then run the demo.

### Before your fix

**The run.** Paste what `npm run demo:replay` printed without editing it. A TA
may ask you about it in milestone 3.

**Your prediction against the run.** Where they agree and where they do not.

### Your fix

**The diff.** Paste it.

**What your fix uses to decide what to do on a second delivery, and what it
does not protect.** One or two lines.

### After your fix

**The run.** Paste what `npm run demo:replay` printed on the fixed code.

**What stays the same across its three states.** Name the numbers you
compared.

---

## Milestone 3: a delivery that stops halfway

### The failure

**The run.** Run `npm run demo:failure` and paste what the delivery left
behind (the ledger, the provider, and the dead-letter list).

### Your recovery

**What, if anything, the failed delivery left undone.**

**The option you took and its evidence.** If you redelivered the
message, paste the state after the redelivery and the bus log line for that
message. If you left the message for a person to handle, write the runbook line
they would follow, and name which fields of the dead-lettered message they read to
follow it and what each one tells them.

### The other option

**What the option you did not take would have cost.** One line, in terms of who
does the work, when the organizer gets their money, and what has to be true for
it to be safe.
