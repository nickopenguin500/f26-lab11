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
| 1 | Room validation & conflict checking | Synchronous | Synchronous | Caller needs immediate confirmation that the room is valid and not double-booked. |
| 2 | Persisting booking (`store.save`) | Synchronous | Synchronous | Core state must be durably saved before returning success to the user. |
| 3 | Updating search index (`searchIndex.indexBooking`) | Synchronous | Event-driven | Search indexing is secondary; search cluster downtime should not fail room reservations. |
| 4 | Confirmation email (`EmailConfirmationHandler`) | Event-driven | Event-driven | Mail delivery latency and SMTP glitches should not hold up the booking response. |
| 5 | Receipt generation (`ReceiptGenerator`) | Event-driven | Event-driven | Receipts can be rendered in the background and downloaded on demand. |
| 6 | Reminder scheduling (`ReminderScheduler`) | Event-driven | Event-driven | Scheduling a future reminder does not need to block the booking request path. |
| 7 | Dashboard stats update (`StatsUpdater`) | Event-driven | Event-driven | Metric aggregation is internal telemetry that the booking caller does not depend on. |
| 8 | Refund processing (`RefundLedger` / payment gateway) | Event-driven | Event-driven | Third-party payment gateway calls are slow and failure-prone; decoupling allows retries. |

### Two calls in detail

Pick two rows and give each a paragraph rather than a line. One of them has to
be the operation `TICKET.md` implicates.

**First operation and your call.** Updating the search index (`searchIndex.indexBooking`) — My call: Event-driven.
The caller (a student or staff member booking via kiosk or web) needs an immediate guarantee that the room is held for their requested time slot. They do not need the room-finder search index to be updated at that exact millisecond; eventual consistency within a few minutes is completely sufficient. When wired synchronously (as it is today), an outage in the search cluster throws `SearchIndexUnavailableError` and aborts the entire transaction before the booking can be saved. If wired as an event subscriber to `booking.confirmed`, search outages would only dead-letter or defer index updating while allowing bookings to succeed uninterrupted.

**Second operation and your call.** Confirmation email (`EmailConfirmationHandler`) — My call: Event-driven.
The caller needs immediate confirmation on their screen that their booking went through; while they expect an email confirmation for their calendar and records, receiving it within a few minutes is acceptable. When the email provider or SMTP relay is unavailable, because this operation is wired as an event subscriber on the bus, the bus catches the handler error and dead-letters it, so the caller still sees an immediate successful booking confirmation on screen. If it were wired synchronously in the request path, any mail server downtime or network delay would cause the booking request to hang or crash for the user.

### The incident

**The call behind `TICKET.md`.**
- **File:** `src/service/bookingService.ts`
- **Method:** `BookingService.book`
- **Line:** Line 94 (`this.searchIndex.indexBooking(booking, room, now);`)
- **What the caller gets back:** When the search cluster was down, `searchIndex.indexBooking` threw `SearchIndexUnavailableError` (`search index is unavailable`). Because this call happens before `this.store.save(booking)` at line 95, execution aborted immediately. The booking was never stored, no events were emitted, and the caller (kiosk/web page) received an uncaught exception (error screen) rather than a booking confirmation.

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
