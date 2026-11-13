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

Because `RefundLedger.handle` currently does not perform any idempotency check or tracking of previously processed messages/bookings, calling `redeliver` on the refund message will blindly re-execute the handler logic. Specifically:
- **In the ledger:** Each redelivery will append a duplicate entry to `entries` (new `entryId` like `led-2`, `led-3`), causing the ledger size and total refunded cents to increase by the refund amount ($170.00) on each delivery (e.g., from 1 entry / $170 to 2 entries / $340, then 3 entries / $510).
- **At the payment provider (`PaymentGateway`):** Each redelivery will call `gateway.refund()` again, creating additional settlements (`stl-2`, `stl-3`) and returning duplicate money to the organizer (total settled increasing from $170.00 to $340.00, then $510.00).
- **In the bus log:** Each call to `app.bus.redeliver` invokes `deliver`, which increments `attempts` for the message. In the bus log, the refund message will show an increasing delivery count (`deliveries=1`, then `deliveries=2`, then `deliveries=3`), while other published messages remain at `deliveries=1`.

### Before your fix

**The run.** Paste what `npm run demo:replay` printed without editing it. A TA
may ask you about it in milestone 3.

```
booked bk-1001 for 8500 cents
cancelled bk-1001, refund 8500 cents, refund message msg-3

=== after the cancellation (one delivery of the refund message) ===
ledger entries: 1
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 1
gateway settlements: 1
  stl-1 bk-1001 $85.00
money returned in total: $85.00
dead letters: 0

redelivering msg-3

=== after one redelivery ===
ledger entries: 2
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
  led-2 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 2
gateway settlements: 2
  stl-1 bk-1001 $85.00
  stl-2 bk-1001 $85.00
money returned in total: $170.00
dead letters: 0

redelivering msg-3 again

=== after two redeliveries ===
ledger entries: 3
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
  led-2 bk-1001 $85.00 (cancelled more than two hours ahead)
  led-3 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 3
gateway settlements: 3
  stl-1 bk-1001 $85.00
  stl-2 bk-1001 $85.00
  stl-3 bk-1001 $85.00
money returned in total: $255.00
dead letters: 0

=== bus log ===
  msg-1 booking.confirmed deliveries=1
  msg-2 booking.cancelled deliveries=1
  msg-3 refund.requested deliveries=3
```

**Your prediction against the run.** Where they agree and where they do not.
- **Where they agree:**
  - In the ledger: Each redelivery appended a new duplicate entry (`led-1`, `led-2`, `led-3`), causing the ledger entries count and total refunded cents to increase on every delivery.
  - At the payment provider: Every delivery called `gateway.refund()` again, creating duplicate settlements (`stl-1`, `stl-2`, `stl-3`) and returning duplicate money ($85.00 -> $170.00 -> $255.00).
  - In the bus log: Delivery count for `msg-3` incremented on each redelivery (`deliveries=3`), while `booking.confirmed` and `booking.cancelled` remained at `deliveries=1`.
- **Where they do not:**
  - The prediction had guessed $170.00 based on standard 2-hour pricing, whereas the sample booking refund was $85.00 (scaling by $85.00 per redelivery to $170 and $255). All qualitative behaviors and architectural side effects matched the prediction.

### Your fix

**The diff.** Paste it.

```diff
diff --git a/src/handlers/refundLedger.ts b/src/handlers/refundLedger.ts
index 600ffca..e5ecb2f 100644
--- a/src/handlers/refundLedger.ts
+++ b/src/handlers/refundLedger.ts
@@ -24,6 +24,7 @@ export class RefundLedger implements Subscriber<RefundRequested> {
   readonly name = 'RefundLedger';
 
   private readonly entries: LedgerEntry[] = [];
+  private readonly processedMessageIds = new Set<string>();
   private readonly gateway: PaymentGateway;
 
   constructor(gateway: PaymentGateway) {
@@ -31,16 +32,22 @@ export class RefundLedger implements Subscriber<RefundRequested> {
   }
 
   handle(message: Message<RefundRequested>): void {
+    if (this.processedMessageIds.has(message.id)) {
+      return;
+    }
     const event = message.payload;
-    this.entries.push({
-      entryId: `led-${this.entries.length + 1}`,
-      bookingId: event.bookingId,
-      organizer: event.organizer,
-      amountCents: event.amountCents,
-      reason: event.reason,
-      recordedAtMinute: message.publishedAtMinute,
-    });
+    if (this.forBooking(event.bookingId).length === 0) {
+      this.entries.push({
+        entryId: `led-${this.entries.length + 1}`,
+        bookingId: event.bookingId,
+        organizer: event.organizer,
+        amountCents: event.amountCents,
+        reason: event.reason,
+        recordedAtMinute: message.publishedAtMinute,
+      });
+    }
     this.gateway.refund(event.bookingId, event.amountCents);
+    this.processedMessageIds.add(message.id);
   }
 
   /** Every ledger entry, oldest first. */
```

**What your fix uses to decide what to do on a second delivery, and what it
does not protect.** One or two lines.

My fix uses the bus message ID (`message.id`) as an idempotency key to return early if the message was already fully processed, and checks `this.forBooking(event.bookingId)` before pushing to prevent duplicate ledger entries if a prior attempt failed mid-delivery at the gateway.
It does not protect against upstream publishing duplicate refund messages with different IDs for the same booking, nor does it persist across process restarts (in-memory state).

### After your fix

**The run.** Paste what `npm run demo:replay` printed on the fixed code.

```
booked bk-1001 for 8500 cents
cancelled bk-1001, refund 8500 cents, refund message msg-3

=== after the cancellation (one delivery of the refund message) ===
ledger entries: 1
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 1
gateway settlements: 1
  stl-1 bk-1001 $85.00
money returned in total: $85.00
dead letters: 0

redelivering msg-3

=== after one redelivery ===
ledger entries: 1
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 1
gateway settlements: 1
  stl-1 bk-1001 $85.00
money returned in total: $85.00
dead letters: 0

redelivering msg-3 again

=== after two redeliveries ===
ledger entries: 1
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 1
gateway settlements: 1
  stl-1 bk-1001 $85.00
money returned in total: $85.00
dead letters: 0

=== bus log ===
  msg-1 booking.confirmed deliveries=1
  msg-2 booking.cancelled deliveries=1
  msg-3 refund.requested deliveries=3
```

**What stays the same across its three states.** Name the numbers you
compared.

The following numbers stay identical across all three states (after the cancellation, after one redelivery, and after two redeliveries):
- Ledger entries count: `1`
- Gateway calls count: `1`
- Gateway settlements count: `1`
- Total money returned: `$85.00`
- Dead letters count: `0`
Only the delivery count for `msg-3` in the bus log changes (`deliveries=1` -> `deliveries=2` -> `deliveries=3`).

---

## Milestone 3: a delivery that stops halfway

### The failure

**The run.** Run `npm run demo:failure` and paste what the delivery left
behind (the ledger, the provider, and the dead-letter list).

```
armed: call 1 to the payment provider will throw
booked bk-1001 for 8500 cents
cancelled bk-1001, refund 8500 cents, refund message msg-3
the cancel call itself returned: booking is cancelled

=== after the failed delivery ===
ledger entries: 1
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 1
gateway settlements: 0
money returned in total: $0.00
dead letters: 1
  msg-3 refund.requested attempt 1 in RefundLedger: payment provider did not answer on call 1 for bk-1001
    failedAtMinute: 480
    payload: {"bookingId":"bk-1001","organizer":"H. Ahmad","organizerEmail":"hahmad@example.edu","amountCents":8500,"reason":"cancelled more than two hours ahead","requestedAtMinute":480}

=== bus log ===
  msg-1 booking.confirmed deliveries=1
  msg-2 booking.cancelled deliveries=1
  msg-3 refund.requested deliveries=1
```

### Your recovery

**What, if anything, the failed delivery left undone.**

The failed delivery recorded the refund entry in `RefundLedger` (`led-1` for $85.00), but when invoking `this.gateway.refund()`, the payment provider threw an error (`payment provider did not answer on call 1 for bk-1001`). Consequently, the payment gateway never completed a settlement (`gateway settlements: 0`, `money returned in total: $0.00`), the message was placed onto the dead-letter list, and the organizer never actually received their $85.00 refund despite the ledger entry existing.

**The option you took and its evidence.** If you redelivered the
message, paste the state after the redelivery and the bus log line for that
message. If you left the message for a person to handle, write the runbook line
they would follow, and name which fields of the dead-lettered message they read to
follow it and what each one tells them.

Option chosen: **Redelivery via `app.bus.redeliver(cancellation.refundMessageId)`**.
Because of the Milestone 2 fix (checking `this.forBooking(event.bookingId).length === 0`), the handler skips writing a duplicate ledger entry on redelivery and proceeds directly to re-invoking `gateway.refund()`, which succeeds on attempt 2.

Output after redelivery:
```
recovering: redelivering msg-3

=== after recovery ===
ledger entries: 1
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 2
gateway settlements: 1
  stl-1 bk-1001 $85.00
money returned in total: $85.00
dead letters: 1
  msg-3 refund.requested attempt 1 in RefundLedger: payment provider did not answer on call 1 for bk-1001
    failedAtMinute: 480
    payload: {"bookingId":"bk-1001","organizer":"H. Ahmad","organizerEmail":"hahmad@example.edu","amountCents":8500,"reason":"cancelled more than two hours ahead","requestedAtMinute":480}

=== bus log ===
  msg-1 booking.confirmed deliveries=1
  msg-2 booking.cancelled deliveries=1
  msg-3 refund.requested deliveries=2
```

Bus log line for the refund message showing delivery count 2:
```
  msg-3 refund.requested deliveries=2
```

*Triage alternative details (had manual triage been used):*
- **Runbook line:** "Inspect dead-letter queue for `RefundLedger` entries; check external payment gateway portal to confirm if charge for `bookingId` settled; if unsettled, issue manual refund via gateway terminal or invoke manual redelivery for `message.id`."
- **Fields read:**
  - `message.id` (`msg-3`): Unique bus message identifier to target for redelivery or query in system logs.
  - `subscriberName` (`RefundLedger`): Identifies which component failed, directing the operator to the refund subsystem.
  - `reason` (`payment provider did not answer on call 1 for bk-1001`): Explains the failure mode (transient network or gateway error).
  - `attempt` (`1`): Shows delivery attempt count at the time of failure.
  - `failedAtMinute` (`480`): Timestamp of failure (08:00) to correlate with external payment provider logs.
  - `message.payload` (`{bookingId, organizer, amountCents, ...}`): Contains business data needed to verify the booking, organizer, and refund amount ($85.00).

### The other option

**What the option you did not take would have cost.** One line, in terms of who
does the work, when the organizer gets their money, and what has to be true for
it to be safe.

Manual triage shifts the work to an on-call human operator, delays the organizer's refund until manual investigation (hours or days rather than instant recovery), and is only safe if the operator independently verifies the payment gateway's settlement state and the internal ledger before issuing money to avoid duplicate payouts.
