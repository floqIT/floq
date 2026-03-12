# Infieldr — FSM Edge Cases, Prevention & AI-Specific Scenarios
*Version 2 — Extended with prevention strategies + AI edge cases*
*Generated: 2026-03-12*

---

## HOW TO READ THIS DOC

Each edge case follows this pattern:
- **Scenario** — what happens
- **Root cause** — why it happens
- **Prevention** — architecture/code/UX fix
- **Fallback** — what to do when it still happens

---

# PART 1: CORE FSM EDGE CASES (DETAILED)

---

## 1. SCHEDULING & DISPATCHING

---

### 1.1 Double-Booking a Technician

**Scenario:** Two dispatchers assign the same tech to overlapping jobs simultaneously.

**Root cause:** No atomic lock on tech availability during assignment.

**Prevention:**
```
DB: Use SELECT FOR UPDATE on tech_schedule row during assignment transaction
API: Optimistic locking — include schedule_version in request, reject if stale
UI: Disable the "Assign" button while request is in-flight (debounce + spinner)
Real-time: Broadcast slot "reserved" state via WebSocket when dispatcher opens it
```
**Fallback:** On conflict, return HTTP 409 with "Tech already booked for this window — here are 3 available alternatives." Never silently overwrite.

---

### 1.2 Job Runs Over — Cascading Schedule Collapse

**Scenario:** Tech's 10 AM job takes 4h instead of 2h. All subsequent jobs for that tech are now late.

**Root cause:** No real-time schedule ripple detection.

**Prevention:**
```
Monitor: When tech's ON_SITE duration exceeds estimate by >30 min, trigger alert
Auto-recalculate: Recompute ETAs for all downstream jobs for that tech
Dispatcher UI: Show red "cascading delay" banner with affected jobs listed
Customer notify: Auto-send "running behind" SMS to next customer(s) with new ETA
```
**Fallback:** If dispatcher doesn't act within 15 min, escalate to manager alert. Offer one-click "redistribute jobs to other available techs."

---

### 1.3 Recurring Job Falls on Holiday

**Scenario:** Weekly pool cleaning auto-generates a job on July 4th. Tech is off. Customer isn't expecting anyone.

**Root cause:** Recurring job engine doesn't check company holiday calendar.

**Prevention:**
```
Holiday calendar: Company admin defines holiday list (+ federal holidays auto-loaded)
Recurrence engine: Skip or shift logic — "skip" vs "move to next business day" per customer preference
Preview UI: Show next 90 days of generated jobs before saving recurring rule
Warning on save: "3 upcoming jobs land on holidays. Choose: Skip / Move to next weekday / Keep"
```
**Fallback:** 48h before a holiday job fires, send dispatcher alert "Holiday conflict — confirm or reschedule."

---

### 1.4 "Edit This Recurring Job" — Wrong Scope Applied

**Scenario:** Dispatcher edits one instance (time change) but accidentally updates all future occurrences. Customer's next 6 months of appointments silently shift.

**Root cause:** No clear scope selector; default scope too broad.

**Prevention:**
```
Always show modal on recurring job edit:
  ○ Just this job
  ○ This and all future jobs
  ○ All jobs in this series
Never default to "all" — default must be "just this job"
Audit log: Record who changed what scope, with before/after snapshot
Undo window: 5-minute rollback available after bulk edit
```
**Fallback:** Email confirmation to customer when series-level changes are made. Include "If this is wrong, call us" link.

---

### 1.5 Tech Calls Out After Jobs Already Notified

**Scenario:** 7 AM — tech calls in sick. Customers already received "Your tech is on the way at 9 AM" notifications.

**Root cause:** No automated reassignment + customer re-notification flow.

**Prevention:**
```
Tech sick-out workflow:
  1. Tech or dispatcher marks tech "Out — Sick" for day
  2. System shows all affected jobs sorted by time
  3. One-click reassignment suggestions (nearest available tech, right skills)
  4. Batch re-notify customers: "Your appointment is still on — different tech, same time"
  5. If no tech available: offer customer reschedule options via SMS link
```
**Fallback:** Dispatcher manually calls customers. System logs each customer contact attempt.

---

## 2. JOBS / WORK ORDERS

---

### 2.1 Job State Machine — Invalid Transitions

**Valid flow:**
```
DRAFT → SCHEDULED → EN_ROUTE → ON_SITE → IN_PROGRESS → COMPLETE → INVOICED → PAID
```

**Invalid transitions to explicitly block:**

| Attempted Transition | Block Reason | Error Message |
|---|---|---|
| COMPLETE → SCHEDULED | Job already done | "Cannot reschedule a completed job. Create a follow-up instead." |
| INVOICED → IN_PROGRESS | Invoice exists | "Reopen job first — this will void the invoice." |
| PAID → any | Payment received | "Cannot modify a paid job. Issue a credit memo or create a new job." |
| DRAFT → COMPLETE | Steps skipped | "Job must be scheduled and worked before completion." |
| CANCELLED → IN_PROGRESS | Without audit | "Reactivating a cancelled job requires manager approval." |

**Prevention:**
```typescript
// State machine enforcement
const VALID_TRANSITIONS: Record<JobStatus, JobStatus[]> = {
  DRAFT: ['SCHEDULED', 'CANCELLED'],
  SCHEDULED: ['EN_ROUTE', 'ON_HOLD', 'CANCELLED'],
  EN_ROUTE: ['ON_SITE', 'CANCELLED'],
  ON_SITE: ['IN_PROGRESS', 'ON_HOLD', 'INCOMPLETE'],
  IN_PROGRESS: ['COMPLETE', 'ON_HOLD', 'INCOMPLETE'],
  COMPLETE: ['INVOICED'],
  INVOICED: ['PAID'],
  ON_HOLD: ['SCHEDULED', 'CANCELLED'],
  INCOMPLETE: ['SCHEDULED', 'CANCELLED'],
  PAID: [], // terminal — no transitions
  CANCELLED: [], // terminal unless manager override
}

function transitionJob(job, newStatus, actor) {
  if (!VALID_TRANSITIONS[job.status].includes(newStatus)) {
    throw new InvalidTransitionError(job.status, newStatus);
  }
  // log audit trail
  auditLog.write({ jobId: job.id, from: job.status, to: newStatus, actor, ts: now() });
  job.status = newStatus;
}
```

---

### 2.2 Cancellation After Materials Already Used

**Scenario:** Job is 60% done, customer cancels. Tech already used $280 in parts.

**Root cause:** No mid-job cancellation workflow accounting for sunk costs.

**Prevention:**
```
Cancellation modal (mid-job):
  - Show: Materials used so far (auto-pulled from job line items)
  - Show: Labor logged so far
  - Options:
    ○ Issue partial invoice for work completed
    ○ Waive charges (manager approval required)
    ○ Apply cancellation fee per contract
  - Require manager PIN for zero-charge cancellation
Parts return: Flag used parts as "potentially returnable" → prompt tech to bring back
```
**Fallback:** If cancelled without completing this flow, job moves to CANCELLED_DISPUTED status for manual review.

---

### 2.3 Completion Without Required Checklist

**Scenario:** Tech taps "Complete" without filling out required safety checklist or taking mandatory before/after photos.

**Prevention:**
```
Hard block (P0): If checklist has required items, Complete button is disabled
Visual progress: "3 of 5 required steps complete" bar on job screen
Soft block: If photos required, show "Add at least 1 photo to complete"
Bypass: Manager override with reason logged (audit trail)
```
**Fallback:** If tech completes offline and skips checklist, flag job as "COMPLETE_UNVERIFIED" — dispatcher must review before invoice generates.

---

## 3. PAYMENTS (MOST CRITICAL)

---

### 3.1 Duplicate Charge — Catastrophic

**Scenario:** Network timeout causes tech to tap "Charge card" twice. Customer charged twice.

**Root cause:** No idempotency on payment request.

**Prevention:**
```
EVERY payment request must include an idempotency key:
  key = sha256(jobId + customerId + amount + date)
Stripe: Pass Idempotency-Key header — Stripe returns same result for duplicate keys
DB: Store payment_intent_id before charging; check for existence before new charge
UI: Disable "Charge" button immediately on tap; show spinner; only re-enable on explicit error
```
**Fallback:** Automated duplicate detection job — runs hourly, checks for same customer + same amount within 10 minutes → auto-refund + alert finance.

---

### 3.2 Payment Captured on Cancelled Job

**Scenario:** Job gets cancelled but payment was already captured. Customer never gets refund.

**Root cause:** Cancellation and payment are not transactionally linked.

**Prevention:**
```
Cancellation flow checks:
  - IF payment_status = CAPTURED → show "Refund required" step before confirming cancel
  - Auto-initiate refund via Stripe refund API
  - Log: cancelled_by, reason, refund_id, refund_amount
  - Notify customer: "Your job was cancelled. Refund of $X issued — 3–5 business days."
```

---

### 3.3 Partial Payment — Ambiguous State

**Scenario:** Job total = $500. Customer pays $200. What's the status?

**Prevention:**
```
Invoice statuses must include:
  UNPAID | PARTIALLY_PAID | PAID | OVERPAID | VOIDED | DISPUTED

amount_due = total - sum(payments)
Display: "$300 remaining balance"
Alert: After 30 days of partial payment, auto-flag for follow-up
Payment plan: Optionally split into scheduled installments
```

---

### 3.4 Overpayment

**Scenario:** Customer pays $520 on a $500 invoice (sent check for wrong amount).

**Prevention:**
```
Overpayment detection: IF payment > amount_due, prompt:
  ○ Apply $20 as account credit (toward next job)
  ○ Issue $20 refund
  ○ Keep as tip (if tip feature enabled)
Never silently absorb overpayment
Customer gets notification of credit or refund either way
```

---

### 3.5 Tax Edge Cases

**Scenario:** Plumbing job in Virginia — labor is tax-exempt, parts are taxable. System applies one tax rate to everything.

**Prevention:**
```
Line item tax flags:
  each_line_item.taxable = boolean
  tax_rule = { laborTaxable: false, partsTaxable: true, rate: 0.06 }

Tax jurisdiction lookup:
  - Use job address (not company address) for tax rate
  - Integrate TaxJar or Avalara for multi-state accuracy

Tax-exempt customers:
  - Store exemption_certificate on customer record
  - Cert expiry date — alert 30 days before expiry
  - Block tax application when valid cert exists
  - Log: who applied exemption, when, cert ID
```

---

## 4. TECHNICIAN MANAGEMENT

---

### 4.1 Expired Certification Assigned to Specialized Job

**Scenario:** Tech's EPA 608 refrigerant handling cert expired last month. Gets assigned to an HVAC job requiring it.

**Prevention:**
```
Certifications table:
  tech_id, cert_type, cert_number, expiry_date, verified_by

Job type requirements:
  job_type.required_certs = ['EPA_608', 'NATE']

Assignment validation:
  IF job.required_certs not subset of tech.active_certs:
    WARN dispatcher: "Tech missing: EPA 608 (expired Jan 15)"
    BLOCK assignment if cert is hard-required

Proactive alerts:
  30 days before expiry → email tech + manager
  7 days before → Slack/push alert
  On expiry → auto-remove cert, re-evaluate all future assignments
```

---

### 4.2 GPS Check-In from Wrong Location

**Scenario:** Tech is at 123 Main St Unit 4B but geofence is set for Unit 1A. Auto check-in fails or fires at wrong unit.

**Prevention:**
```
Geofence radius: configurable per job (default 150ft, commercial = 300ft)
Manual override: Tech can tap "I'm at the job" with photo verification
Mismatch alert: "GPS shows you 400ft from job address. Confirm location?"
Address detail: Unit number as separate field — store both in geofence polygon
Multi-unit: For apartment complexes, require manual check-in always
```

---

### 4.3 Offline Sync Conflict

**Scenario:** Tech updates job notes offline. Dispatcher updates same notes online. Tech reconnects — whose version wins?

**Prevention:**
```
Conflict resolution strategy:
  - Last-write-wins for non-critical fields (notes, status)
  - Merge strategy for structured fields (line items — append, don't overwrite)
  - For critical fields (payment, completion): server always wins
  - On reconnect: show diff UI — "These changes conflict. Keep yours / Keep server / Merge"

Implementation:
  - Each record has updated_at + device_id
  - Offline queue stores operations (not snapshots)
  - Apply operations in timestamp order on sync
  - Flag conflicts for dispatcher review in "Sync Issues" queue
```

---

## 5. NOTIFICATIONS

---

### 5.1 Reminder Fires for Cancelled Job

**Scenario:** Job cancelled at 3 PM. Automated 24h-before reminder fires at 9 PM for tomorrow's now-cancelled job.

**Prevention:**
```
Notification queue: Store scheduled notifications with job_id + send_at
On cancellation: Purge all pending notifications for that job_id
Check at send time: Verify job.status != CANCELLED before sending (belt AND suspenders)
SMS/email provider: Implement cancel-send API (Twilio message cancellation)
```

---

### 5.2 "Tech On The Way" with Stale GPS

**Scenario:** Customer gets "James is 10 min away" but James is actually 45 min away. GPS position is 2h old.

**Prevention:**
```
GPS freshness check before sending ETA notification:
  IF last_gps_ping > 15 min ago:
    Do NOT send ETA — send generic "On the way" instead
    Alert dispatcher: "Tech GPS stale — ETA unreliable"

ETA calculation:
  Use real-time traffic (Google Maps Distance Matrix or HERE API)
  Recalculate every 5 min while tech is EN_ROUTE
  Only show ETA in customer portal if confidence > threshold

Customer portal:
  Show "Updating..." instead of stale ETA
  Show last-updated timestamp
```

---

## 6. INTEGRATIONS

---

### 6.1 QuickBooks Sync Partial Failure

**Scenario:** Batch of 50 invoices syncing to QB — 37 succeed, 13 fail silently. Financial records now inconsistent.

**Prevention:**
```
Sync architecture:
  - Atomic per-invoice sync with retry
  - Never mark "synced" until QB returns success response
  - Store: qb_invoice_id, synced_at, sync_status per invoice
  - Failed invoices go to "Sync Queue" — visible in admin dashboard
  - Daily reconciliation job: compare FSM invoices vs. QB invoices

On failure:
  - Log full error from QB API
  - Retry with exponential backoff (1min, 5min, 30min, 2h)
  - After 3 failures: page finance admin
```

---

### 6.2 OAuth Token Expired — Silent Stop

**Scenario:** QuickBooks/Google Calendar integration stops syncing because OAuth token expired. No alert. Data silently diverges for days.

**Prevention:**
```
Token monitoring:
  - Store token expiry for each integration
  - 3 days before expiry: email admin "QuickBooks integration needs re-auth"
  - On 401 from provider: immediately alert + disable integration gracefully
  - Health check: ping each integration API hourly, surface status in dashboard
  - Integration status page: Green/Yellow/Red with last-synced timestamp
```

---

# PART 2: AI-SPECIFIC EDGE CASES

---

## AI-1. VOICE AI (Scheduling via Phone/WhatsApp)

### AI-1.1 Intent Ambiguity

**Scenario:** Customer says "I need someone to come fix my AC next week." AI doesn't know: Which AC unit? Which day? Morning or afternoon? What's wrong with it?

**Prevention:**
```
Slot-filling dialogue: AI must collect all required slots before booking:
  Required: service_type, preferred_date, preferred_time_window, address, problem_description
  Optional: unit_age, last_serviced, warranty_status

Clarification strategy:
  - Ask ONE question at a time (not a barrage)
  - Confirm back: "So you'd like AC repair at 456 Oak Lane on Tuesday the 17th, morning. Is that right?"
  - Max 3 clarification turns before escalating to human

Edge cases:
  - "Next week" → resolve to specific date range, ask "Which day works best?"
  - "As soon as possible" → show next available slot, don't just say "I'll check"
  - "Same as last time" → pull previous job type/address from customer history
```

---

### AI-1.2 Mishearing / Speech Recognition Errors (Riva STT)

**Scenario:** Customer says "leaking pipe" — Riva transcribes "leading pipe." AI books wrong service type.

**Prevention:**
```
Confidence thresholds:
  - If STT confidence < 0.85, ask for clarification: "Just to confirm, did you say [X]?"
  - For critical fields (address, phone number): always read back and confirm
  - For numbers: "You said your address is 4-5-6 Oak Lane — correct?"

Noise handling (HVAC/field environment):
  - Use NVIDIA Riva with noise-suppression model
  - Detect background noise level; if high, ask to move somewhere quieter
  - Fallback: "I'm having trouble hearing you — I'll connect you to our team"

Post-call review:
  - Flag low-confidence bookings for human review before confirmation sent
  - Log transcription + confidence score for every booking
```

---

### AI-1.3 Customer Rage / Emotional State

**Scenario:** Customer is furious about a botched previous job. AI tries to book a new appointment. Customer is yelling.

**Prevention:**
```
Sentiment detection:
  - Monitor for anger keywords + tone (volume, speech rate)
  - Trigger: IF sentiment = ANGRY or FRUSTRATED for 2+ turns:
    → Stop trying to complete booking task
    → Empathy response: "I understand this is frustrating. Let me get a manager for you."
    → Route to human immediately
    → Log: customer_id, job_id, sentiment_score, escalation_reason

Never:
  - Keep trying to upsell to an angry customer
  - Repeat the same question after customer expresses frustration
  - Say "I'm sorry you feel that way"
```

---

### AI-1.4 AI Books Job It Can't Fulfill

**Scenario:** AI confirms "We can be there Wednesday at 2 PM" — but there are no available techs Wednesday afternoon.

**Root cause:** AI checks availability in real-time but schedule changed between confirmation and commit.

**Prevention:**
```
Two-phase booking:
  Phase 1: AI proposes slot → checks availability (READ)
  Phase 2: Customer confirms → atomic slot reservation (WRITE with lock)
  
Hold mechanism:
  - On Phase 1: place soft hold for 5 min
  - If not confirmed in 5 min, release hold
  - On Phase 2: convert to hard booking

Failure path:
  - If slot taken between phases: "That slot just got taken — here's the next available: [3 options]"
  - Never confirm without successful DB write
```

---

### AI-1.5 AI Gives Wrong Pricing

**Scenario:** Customer asks "How much for a furnace tune-up?" AI says "$89." Actual price is $129 because it changed last month.

**Prevention:**
```
Pricing source of truth:
  - AI MUST pull pricing from live catalog (not trained-in hardcoded values)
  - Price catalog version tracked; AI sees only current_prices where active = true
  - If pricing is variable/quote-based: AI says "Our tech will assess and quote on-site — typical range is $X-$Y"
  - Never let AI confirm a specific price without authorization flag on that service
  - Audit: log every price quoted by AI for reconciliation
```

---

## AI-2. INTELLIGENT DISPATCHING

### AI-2.1 AI Assigns Wrong Skill Match

**Scenario:** AI dispatches HVAC tech to a plumbing job because "they were closest."

**Prevention:**
```
Dispatch scoring must weight skills above distance:
  score = (skill_match * 0.5) + (availability * 0.3) + (proximity * 0.2)

Hard constraint: skill_match = 0 → tech is ineligible (not just lower score)
Fallback: If no skilled tech available in zone, escalate to dispatcher
AI explains: "Assigned James — NATE-certified, 4.2 mi away, next available 10 AM"
```

---

### AI-2.2 AI Routing Creates Impossible Schedule

**Scenario:** AI optimizer assigns tech 8 jobs across 60 miles with 30-min windows. Physically impossible.

**Prevention:**
```
Feasibility check after optimization:
  - Simulate full day route with realistic travel times (Google Maps API)
  - Flag if any job has <5 min buffer after travel + service time
  - Cap jobs per tech per day based on average job duration
  - If infeasible: remove lowest-priority job, re-optimize, alert dispatcher

AI must surface confidence:
  "This schedule has tight windows — 2 jobs have <10 min buffers. Approve or adjust?"
```

---

### AI-2.3 AI Learns Bad Patterns from Bad Data

**Scenario:** AI dispatch model trained on historical data where certain techs were always assigned certain jobs — not because of skill, but bias. AI perpetuates unfair assignment.

**Prevention:**
```
Training data audit:
  - Remove tech_id from dispatch model features (use skills, certs, ratings only)
  - Regular fairness audit: check workload distribution across techs monthly
  - Alert if any tech receives <70% or >130% of average assignments

Explainability:
  - Every AI dispatch decision must log: "Why this tech?" with factors
  - Dispatcher can override + reason logged → feeds back into model
```

---

## AI-3. DOCUMENT & INVOICE PROCESSING

### AI-3.1 AI Misreads Handwritten Work Order

**Scenario:** Tech fills out paper work order. AI OCR reads "3 hours" as "8 hours." Invoice is wrong.

**Prevention:**
```
OCR confidence gate:
  - If confidence < 0.90 on any numeric field → flag for human review
  - Show extracted values with highlighting: "AI read this as '8' — correct?"
  - Never auto-post invoice from AI extraction without human confirmation
  - Track OCR accuracy per tech (some write more legibly) — learn over time
```

---

### AI-3.2 AI Auto-Generates Invoice with Wrong Line Items

**Scenario:** AI generates invoice from job notes. Notes say "replaced capacitor" — AI adds capacitor part at wrong price/part number.

**Prevention:**
```
Line item resolution:
  - AI maps natural language to catalog items (fuzzy match)
  - If match confidence < 0.95: show options — "Did you mean: [A] $45 capacitor or [B] $78 dual capacitor?"
  - If no catalog match: create "custom line item" flagged for review
  - Never auto-generate invoice without tech sign-off screen
  - Tech sees: "AI generated this invoice — review and approve"
```

---

## AI-4. CUSTOMER COMMUNICATION AI

### AI-4.1 AI Sends Personalized Message to Wrong Customer

**Scenario:** AI sends "Hi John, your tech James is on the way to 123 Main St" to a different customer named John.

**Root cause:** Template rendering bug — wrong customer context injected.

**Prevention:**
```
Message generation safety:
  - Always include customer_id in notification context
  - Render template with customer_id-scoped data only (no ambient state)
  - Before send: assert message.contains(customer.phone_last4) or customer.first_name
  - Test: unit test every notification template with mismatched contexts
  - Audit log: store rendered message + recipient for every send
```

---

### AI-4.2 AI Responds to Customer Complaint Inappropriately

**Scenario:** Customer texts "Your tech scratched my floor!!!" — AI auto-responds "Thanks for your feedback! Would you like to schedule your next appointment?"

**Prevention:**
```
Complaint classification:
  Intent categories: COMPLAINT, DISPUTE, PRAISE, BOOKING, INFO_REQUEST, CANCELLATION
  
  IF intent = COMPLAINT:
    - NEVER continue to task completion
    - NEVER respond with generic positive language
    - Response: "I'm so sorry to hear this. I'm flagging this for our manager who will call you within [X hours]."
    - Create internal ticket: type=COMPLAINT, urgency=HIGH
    - Notify manager immediately
    - Log: message, customer_id, job_id (if inferable), timestamp

Escalation SLA:
  - COMPLAINT → manager notified in <15 min
  - DAMAGE_CLAIM → owner notified immediately
```

---

### AI-4.3 AI Falls Into Infinite Clarification Loop

**Scenario:** Customer gives vague answers. AI keeps asking clarifying questions. Customer abandons after 10 back-and-forths.

**Prevention:**
```
Turn limit: After 3 failed clarification attempts for same slot:
  - Collect what you have
  - Say: "Let me have our team follow up to confirm the details — expect a call within 30 min"
  - Create lead with partial info + flag for human follow-up
  - Never loop more than 3 times on same missing field

Fallback escalation:
  Turn 1–3: AI collects slots
  Turn 4: "Let me get a team member to help"
  Immediately: Route to live dispatcher or send callback request
```

---

## AI-5. AI PRICING & ESTIMATION

### AI-5.1 AI Gives Confident Quote for Unusual Job

**Scenario:** Customer describes a job with unusual complexity. AI gives a flat quote. Tech arrives and job is 3x more complex. Customer fights invoice.

**Prevention:**
```
Confidence-based quoting:
  IF job_complexity_score > threshold:
    "Based on what you've described, typical range is $X–$Y. 
     Our tech will assess on-site and confirm before starting work."
  
  Require customer acknowledgment: "Prices may vary based on site conditions"
  
  On-site: Tech must get customer signature on estimate before work begins
  If actual > estimate by >20%: Require customer approval before proceeding
```

---

### AI-5.2 AI Upsell Goes Wrong

**Scenario:** AI notices customer's water heater is old (from job history) and proactively recommends replacement. Customer feels "spied on" and complains.

**Prevention:**
```
Upsell rules:
  - Only surface recommendations when directly relevant to current job
  - Never reference data the customer didn't explicitly share in THIS conversation
  - Frame as: "Based on your service history with us..." (transparent data use)
  - Allow customer to opt out: "Don't send me recommendations"
  - Upsell timing: Post-job completion only, not mid-service
  - Log all AI-generated recommendations for compliance review
```

---

## AI-6. PREDICTIVE MAINTENANCE AI

### AI-6.1 False Positive — Unnecessary Service Dispatch

**Scenario:** AI predicts AC unit needs servicing based on usage patterns. Dispatches tech. Tech finds nothing wrong. Customer loses half a day.

**Prevention:**
```
Confidence thresholds:
  < 0.70: No action
  0.70–0.85: Send customer a notification ("Consider scheduling a checkup")
  > 0.85: Suggest scheduling (customer must confirm, never auto-book)
  
Never auto-dispatch based on prediction alone.
Always require human (customer or dispatcher) confirmation.

Feedback loop:
  Log prediction outcome (was issue found? what was found?)
  Use to retrain model and improve precision
```

---

### AI-6.2 AI Prediction Bias Toward Certain Equipment Brands

**Scenario:** AI trained on data where Brand X equipment was always serviced on shorter cycles (company preference). Model now over-recommends service for Brand X, even when not needed.

**Prevention:**
```
Feature audit: Remove brand as a predictive feature
Cross-brand validation: Test model performance per brand — flag if >15% variance
Human override: Technician feedback on prediction quality feeds back as label
Regular model refresh with de-biased datasets
```

---

# PART 3: VALIDATION RULES (COMPREHENSIVE)

## Field-Level Validations

```typescript
const validations = {
  phone: {
    format: /^\+?[1-9]\d{9,14}$/,  // E.164
    normalize: (p) => parsePhoneNumber(p, 'US').formatE164(),
    error: "Phone must be a valid US or international number"
  },
  email: {
    format: /^[^\s@]+@[^\s@]+\.[^\s@]+$/,
    maxLength: 254,
    sanitize: (e) => e.toLowerCase().trim(),
    error: "Invalid email address"
  },
  jobDate: {
    rule: (d) => d >= today() || isBackdatingAllowed(),
    error: "Job date cannot be in the past"
  },
  jobDuration: {
    min: 15,   // minutes
    max: 1440, // 24 hours
    error: "Duration must be between 15 minutes and 24 hours"
  },
  price: {
    min: 0,
    max: 999999.99,
    decimals: 2,
    error: "Price must be a positive number"
  },
  taxRate: {
    min: 0,
    max: 30,  // % — flag if > 20% as likely error
    decimals: 3,
    error: "Tax rate must be between 0% and 30%"
  },
  discount: {
    rule: (d, subtotal) => d <= subtotal,
    error: "Discount cannot exceed invoice subtotal"
  },
  notes: {
    maxLength: 5000,
    sanitize: stripHtml,  // prevent XSS
    error: "Notes cannot exceed 5,000 characters"
  },
  gpsCoords: {
    lat: [-90, 90],
    lng: [-180, 180],
    error: "Invalid GPS coordinates"
  },
  partQty: {
    type: 'integer',
    min: 0,
    error: "Quantity must be a whole number, 0 or greater"
  }
}
```

## Business Rule Validations

```
BEFORE saving a job:
  ✓ Customer exists and is active
  ✓ Service address is geocodable
  ✓ Assigned tech is available in that window
  ✓ Assigned tech has required certifications
  ✓ At least one line item exists (if status > DRAFT)
  ✓ Job date is not on company holiday (warn)
  ✓ Customer has no invoice >90 days past due (warn or block per config)

BEFORE generating invoice:
  ✓ Job status is COMPLETE or INCOMPLETE
  ✓ At least one line item with price > $0
  ✓ Customer has valid email or phone for delivery
  ✓ Tax rate is set (even if 0)
  ✓ No duplicate invoice already exists for this job

BEFORE processing payment:
  ✓ Invoice exists and is not VOID
  ✓ Payment amount ≤ remaining balance (unless overpayment handling enabled)
  ✓ Card is valid and not expired
  ✓ Idempotency key is unique for this transaction
  ✓ Not a duplicate within last 10 minutes for same customer + amount
```

---

# PART 4: ERROR CODES (API Design)

```
4xx — Client Errors (fix the request)
  400 VALIDATION_ERROR        — field validation failed (include field + message)
  409 SCHEDULE_CONFLICT       — tech already booked for this time
  409 DUPLICATE_PAYMENT       — payment already processed for this idempotency key
  409 INVALID_TRANSITION      — job status change not allowed
  422 MISSING_REQUIRED_FIELDS — required fields empty
  423 JOB_LOCKED              — job is being edited by another user
  429 RATE_LIMITED            — too many requests

5xx — Server Errors (retry or escalate)
  500 PAYMENT_PROVIDER_ERROR  — Stripe/Square returned error
  502 INTEGRATION_UNAVAILABLE — QuickBooks/Google Calendar down
  503 OPTIMIZATION_TIMEOUT    — route optimization took too long
  507 STORAGE_QUOTA_EXCEEDED  — attachment storage full

AI-specific:
  AI_001 LOW_CONFIDENCE_STT   — speech recognition below threshold
  AI_002 INTENT_UNCLEAR       — could not determine customer intent
  AI_003 SLOT_UNAVAILABLE     — AI-proposed slot taken during hold
  AI_004 PRICE_UNRESOLVABLE   — service not in live catalog
  AI_005 ESCALATE_TO_HUMAN    — AI cannot handle this case
```

---

# PRIORITY IMPLEMENTATION ORDER

## Sprint 1 (P0 — Before Launch)
1. Job state machine enforcement (no invalid transitions)
2. Double-booking prevention (optimistic lock)
3. Idempotency on payments (no duplicate charges)
4. Required field validation at API layer (not just frontend)
5. AI booking two-phase hold (propose → confirm → commit)
6. Notification purge on cancellation

## Sprint 2 (P1 — First 30 Days)
7. Cascading delay detection + customer re-notification
8. Recurring job holiday handling
9. Offline sync conflict resolution
10. AI sentiment detection → human escalation
11. GPS freshness check before ETA notification
12. OAuth token expiry monitoring for integrations

## Sprint 3 (P2 — Post-Launch Hardening)
13. AI dispatch fairness audit
14. Predictive maintenance confidence thresholds
15. QuickBooks reconciliation job
16. Certification expiry proactive alerts
17. Tax jurisdiction lookup by job address
18. AI upsell compliance rules
