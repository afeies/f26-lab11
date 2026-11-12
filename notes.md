# Synchronous vs. event-driven

## Synchronous

The caller does the work itself, inside the request, and waits for it.

- The caller gets the result (or the error) before it returns.
- If the work fails, the whole operation fails.
- Caller and callee are coupled: the caller is only as available as
  everything it calls.

Example: `book()` calls `searchIndex.indexBooking(...)` directly
(`src/service/bookingService.ts:94`). Search down → booking fails.

## Event-driven

The caller does its core work, publishes an event ("this happened"), and
returns. Subscribers react to the event separately.

- The caller doesn't know or care who handles the event.
- A subscriber failing does not fail the caller (the bus dead-letters it).
- The side effect can happen later, or more than once (at-least-once
  delivery), so handlers must be idempotent.

Example: `book()` publishes `booking.confirmed`; email, receipt, reminders,
and stats each handle it on their own.

## How to choose

Ask: **does the caller need this done before it gets its answer?**

- Yes → synchronous (e.g. conflict check, saving the booking).
- No, it can lag or retry → event-driven (e.g. emails, search index,
  refunds).

Also ask: **if this fails, should the main operation fail too?** If not, it
shouldn't be synchronous.

## Trade-offs

| | Synchronous | Event-driven |
|---|---|---|
| Result | Immediate, known | Eventual |
| Failure | Fails the caller | Isolated; needs retry/dead-letter handling |
| Coupling | Tight | Loose |
| Duplicates | Not an issue | Possible; handlers must be idempotent |
| Debugging | Simple stack trace | Follow the event log |



 book() should check the room is free, save the booking, and publish booking.confirmed. Indexing should be a subscriber that reacts to that event, the same way email, receipts and reminders already do.

The problem here is that indexing happens in the book() method.

All that the method should do is check if the room is free and save the booking.

Ideally we would use the observer pattern (publisher, subscriber) which is a form of event driven programming

event-driven programming: instead of one funcrtion calling every piece of work directly, a component announces that something happened, and other components that care react to it on their own

- BookingService does core job, publishes events (booking.confirmed, booking.canceled)
- handlers subscribe to those events and do the follow-up work