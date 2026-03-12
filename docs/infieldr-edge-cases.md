# Infieldr — FSM Edge Cases & Unhappy Path Scenarios
*Compiled from Jobber, Housecall Pro, ServiceTitan patterns*
*Generated: 2026-03-12*

---

## 1. SCHEDULING & DISPATCHING

### Booking / Slot Conflicts
- Two dispatchers book the same tech at the same time (race condition)
- Customer books online while dispatcher is mid-edit of same slot
- Job runs over — next job on same tech is now conflicting
- Tech has overlapping jobs from recurring job expansion
- Job created without assigning a tech (unassigned queue)
- Job assigned to tech who has called out sick after assignment
- Double-booking same customer address on same day (merge or warn?)
- Emergency job needs insertion into full schedule

### Time Windows
- Customer wants 8–10 AM window; tech can't arrive until 10:15 AM
- Job estimated at 2h, tech finishes in 45 min (early completion)
- Job estimated at 1h, takes 4h (overtime, next jobs cascade)
- All-day job blocks a tech from receiving any same-day dispatch
- Job start time passes with no check-in (auto-alert dispatcher?)
- Tech checks in late — was stuck in traffic
- Job needs to start before business hours (e.g., 6 AM commercial)
- Job spans midnight (overnight service window)

### Recurring Jobs
- Recurring job falls on a holiday
- Recurring job falls on a weekend (business closed, reschedule to Monday?)
- Recurring job end date is in the past (should still exist or auto-cancel?)
- Customer cancels one instance vs. cancels entire recurring series
- Customer pauses recurring service for 3 months
- Recurring job expands 6 months ahead and tech leaves the company
- Same customer has overlapping recurring cadences (monthly + quarterly)
- Recurring job edit: apply to "this one" vs. "this and future" vs. "all"

### Multi-Technician Jobs
- Lead tech cancels — helper tech shows up alone
- One tech of two arrives; other is running late
- Partial check-in — only 1 of 3 techs checked in
- Tech added to job mid-progress
- Helper tech tries to complete job without lead
- Disagreement on labor time between two techs on same job

---

## 2. JOBS / WORK ORDERS

### Status Transitions (State Machine)
```
DRAFT → SCHEDULED → EN_ROUTE → ON_SITE → IN_PROGRESS → COMPLETE → INVOICED → PAID
                                                          ↓
                                                      INCOMPLETE / ON_HOLD / CANCELLED
```
**Edge cases:**
- Tech marks complete but no invoice generated (forgot)
- Job marked complete but customer disputes quality
- Job cancelled after tech already en route
- Job cancelled after materials already ordered/used
- Job moved back to IN_PROGRESS after COMPLETE (rework)
- Job put ON_HOLD mid-work (waiting for part, customer approval)
- Job marked complete with zero labor time logged
- Admin marks job complete on behalf of tech (tech offline)
- Job reactivated after 30+ days in COMPLETE state

### Job Details
- Job address is unverifiable (no Google Maps match)
- Service address differs from billing address
- Job notes contain special characters / emoji that break rendering
- Very long job descriptions (truncation vs. scroll)
- Attachments: zero, max (storage limit), corrupted file
- Job has no line items (is invoicing possible?)
- Required fields missing at time of creation (force draft vs. block save)
- Job created for a deleted/inactive customer
- Job created for a customer with a past-due balance

### Cancellation / No-Show
- Customer cancels within 1h of appointment (cancellation fee?)
- Customer not home when tech arrives (no-show fee?)
- Tech marks "customer not home" — auto-reschedule or manual?
- Tech cancels (sick/emergency) after customer already notified
- Job cancelled but deposit was collected — refund workflow
- Partial work done before cancellation (partial invoice?)
- Job cancelled by dispatcher vs. cancelled by customer (audit trail)

---

## 3. CUSTOMER MANAGEMENT

### Account State
- Duplicate customers (same name, different phone/email)
- Customer has two service addresses (home + rental)
- Customer merges: which address, payment method, history wins?
- Customer deactivated but has open jobs
- Customer deactivated but has unpaid invoices
- Customer changes phone number — old number still on job notifications
- Customer opts out of SMS — how do you reach them day-of?
- Customer email bounces on invoice send
- Customer deletes their portal account (jobs/invoices still exist)
- Customer with $0 credit vs. customer with negative balance (overpaid)

### Customer Portal
- Customer books job but no account exists yet (auto-create or require registration?)
- Customer logs in from new device — session/auth handling
- Customer books after business hours (queued or auto-confirmed?)
- Customer tries to reschedule within lock window (e.g., <24h)
- Customer cancels from portal — dispatcher not notified (notification failure)
- Two family members managing same property under different accounts

### Contact Info
- No phone number on file — can't send SMS reminders
- No email on file — can't send invoice
- International phone number format (+44...) in US-focused system
- Customer has multiple contacts (office + site contact)
- Emergency contact vs. billing contact vs. on-site contact

---

## 4. TECHNICIAN / EMPLOYEE MANAGEMENT

### Availability
- Tech has no schedule set (always available vs. never available?)
- Tech availability changes mid-week
- Tech on PTO — jobs still assigned to them
- Tech works in multiple service zones (which zone on which day?)
- Tech exceeds daily hour cap
- Tech has a certification required for job type — but certification expired
- Tech assigned job outside their skill set

### Check-In / Check-Out
- Tech forgets to check in (manual override by dispatcher)
- Tech checks in from wrong location (GPS mismatch)
- Tech checks out but job is unfinished
- Tech loses cell signal mid-job — sync on reconnect
- Tech phone dies mid-job — data recovery
- Tech checks in/out multiple times for same job
- Tech clock-in gap > 8h (flag for review)
- Tech checks out before customer signs off

### Payroll / Commissions
- Tech paid hourly vs. flat rate vs. commission — which applies?
- Job billable time vs. drive time vs. unpaid break
- Overtime calculation when tech works across two jobs in one day
- Tech works partial day then calls out — how is pay calculated?
- Commission on job that later gets refunded
- Two techs split a job — how is commission divided?

---

## 5. INVOICING & PAYMENTS

### Invoice Generation
- Invoice sent to wrong email (customer updated email after job)
- Invoice generated with $0 total (all line items $0?)
- Invoice with duplicate line items
- Invoice for cancelled job (should auto-void or warn)
- Invoice with negative total (credit/discount > charges)
- Invoice sent before job is actually complete
- Multi-job invoice (bundling multiple jobs into one bill)
- Invoice for recurring job — auto-generate or manual trigger?

### Payment Processing
- Card declined — which retry strategy?
- Card expired since original booking
- Partial payment (customer pays half, what's the status?)
- Overpayment — credit to account or refund?
- Payment via cash/check recorded manually vs. card on file
- Payment captured twice (double-charge — critical)
- Refund processed but job not marked as disputed
- Chargeback filed — job revenue reversal
- Payment processing outage (Stripe/Square down)
- Customer pays after job goes to collections

### Tax
- Tax rate changes mid-invoice period
- Job spans two tax jurisdictions (dispatch from VA, job in MD)
- Customer claims tax-exempt status — needs certificate on file
- Tax calculated on parts but not labor (or vice versa)
- Tax-exempt customer charged tax by mistake — credit memo

### Estimates / Quotes
- Estimate accepted after expiry date
- Estimate modified after customer accepted (version control)
- Estimate converted to job — original estimate amounts vs. actuals differ
- Multiple estimates sent to same customer for same job
- Estimate requires customer signature but customer hasn't signed
- Estimate with optional add-ons — customer selects some, not others

---

## 6. NOTIFICATIONS & COMMUNICATIONS

### SMS/Email Delivery Failures
- SMS undelivered (carrier error, invalid number)
- Email bounced (mailbox full, domain invalid)
- Notification sent to old phone number after update
- Customer reply to automated SMS (goes where?)
- Tech never acknowledges job assignment notification
- Duplicate notifications sent (idempotency failure)
- Notification for cancelled job still in send queue

### Reminder Timing
- Reminder sent for job already completed
- Reminder sent for job that was cancelled
- Appointment in 30 min but reminder fires (too late to be useful)
- Customer in different timezone — reminder sent at wrong local time
- Multiple reminders for same job (dedup logic)

### Customer-Facing Portal Notifications
- Customer gets "tech is on the way" but tech hasn't started driving
- ETA is shown as 5 min but tech is 45 min away (stale GPS)
- Customer tracking link expired
- Tech GPS location shared after job complete (privacy)

---

## 7. ROUTE & GPS

### Route Optimization
- Tech location unknown (GPS off or no app)
- Last known location is 2h stale
- Traffic causes ETA to blow past scheduled window
- Route optimization called with 0 jobs (no-op or error?)
- Two jobs at exact same address (multi-unit building)
- Job location outside service zone (admin override or block?)
- Route optimization ignores manual ordering set by dispatcher

### GPS / Location
- Tech GPS permission denied on device
- GPS signal lost in rural area
- Geofence check-in at wrong job (tech is at right address, different unit)
- Two techs at same address — both trigger check-in
- Auto check-in fires when tech drives past location without stopping

---

## 8. INVENTORY & PARTS

### Parts / Materials
- Part used on job not in inventory (untracked part)
- Part quantity goes negative (oversold)
- Part added to job but not used — needs to be returned
- Part ordered but never received — job on hold
- Part ordered from supplier mid-job — lead time > job schedule
- Part serial number tracked for warranty — not captured on job
- Two techs use same part on same job (dedup?)

### Trucks / Stock
- Tech van restocked after job — inventory updated in system?
- Truck inventory vs. warehouse inventory tracking
- Part transferred between trucks
- Part low-stock alert — threshold not set

---

## 9. FORMS & CHECKLISTS

- Required checklist not completed before job marked done
- Checklist completed but photos not attached (photos required)
- Form submitted with blank required fields (frontend vs. backend validation)
- Customer signature required but customer refused to sign
- Signature captured on broken screen (illegible)
- Form submitted offline — sync conflict when back online
- Old form version vs. new version mid-job (tech started on v1, form updated to v2)
- Checklist items: pass/fail — what happens on fail? (block completion or flag only?)

---

## 10. TIME TRACKING

- Tech logs 0 hours on a 4-hour job
- Tech logs more hours than job duration allows
- Time entry overlaps another job
- Break time not deducted (labor law compliance)
- Tech manually edits time after the fact (audit log needed)
- Time tracked in wrong timezone
- Time entry for job tech wasn't assigned to

---

## 11. OFFLINE / SYNC

- App goes offline mid-job update
- Conflicting edits: dispatcher edits job while tech edits same job offline
- Offline form submission, job marked complete offline — sync fails
- Offline payment captured — sync creates duplicate
- Last-write-wins vs. conflict resolution strategy
- Large media upload (video/photo) fails on slow connection
- Partial sync — some fields updated, others not

---

## 12. AUTHENTICATION & PERMISSIONS

- Tech tries to view/edit another tech's jobs
- Admin accidentally deletes their own account
- Role changed while user is mid-session (cached permissions)
- Dispatcher books job for customer admin has restricted
- Password reset link expired before use
- Multi-device login — session invalidation
- Company account suspended (non-payment) — all users locked out
- New employee added but invite email never received
- Tech deactivated but still logged in on device

---

## 13. INTEGRATIONS (QuickBooks, Stripe, Google Calendar, etc.)

- QuickBooks sync fails mid-batch (partial sync)
- Invoice exported to QB but then edited in FSM — version drift
- Stripe webhook not received (delayed payment status)
- Google Calendar event deleted externally but job still in FSM
- Third-party API rate limit hit during bulk operation
- OAuth token expired — integration silently stops syncing
- Integration disconnected — jobs stop syncing, no alert

---

## 14. MULTI-TENANT / COMPANY-LEVEL

- Company has multiple locations — job assigned to wrong location
- Tech assigned to wrong branch
- Invoice sent from wrong company email/logo
- Two company admins conflict on settings (last-write-wins?)
- Company timezone changes (DST, relocation)
- Company deleted — what happens to all data?
- Free trial expired mid-active-job

---

## 15. CRITICAL FINANCIAL EDGE CASES

| Scenario | Correct Behavior |
|---|---|
| Refund after job marked paid | Reverse payment, reopen invoice |
| Invoice sent, never paid, 90 days | Auto-flag for collections |
| Discount > invoice total | Prevent or clamp to $0 |
| Tax-exempt customer charged tax | Issue credit memo |
| Duplicate charge | Immediate refund + alert |
| Subscription payment fails | Grace period, then suspend |
| Cash payment, no receipt | Manual record + audit log |

---

## 16. VALIDATION RULES (Field-Level)

| Field | Validation |
|---|---|
| Phone | E.164 format, US/Canada or international |
| Email | RFC 5322, MX record check optional |
| Address | Geocodable, unit number separate field |
| Job date | Cannot be in past unless backdating allowed |
| Duration | > 0, reasonable max (e.g., <24h single job) |
| Price | Non-negative, max value cap |
| Tax rate | 0–100%, no more than 3 decimal places |
| Discount | Cannot exceed subtotal |
| Notes | Max length, sanitize HTML/scripts |
| Part qty | Integer, >= 0 |
| GPS coords | Valid lat/lng range |

---

## Priority Tiers for Implementation

### P0 — Block app use if wrong
- Double-booking same tech
- Duplicate charge / double payment
- Job completion without required photos/signature when enforced
- Permission violations (viewing other company's data)

### P1 — Warn but allow
- No-show fee trigger
- Missing required checklist items
- Customer email bounce on invoice send
- Schedule conflict on assignment

### P2 — Log and alert
- Stale GPS location
- Integration sync failure
- Offline conflict resolution
- Partial payment

### P3 — Nice to have
- Estimate expired
- Low stock threshold crossed
- Tech exceeds estimated job duration
