# Vivamen Online Pharmacy — Full Backend Flowchart

This document describes the complete end-to-end process from a patient clicking a product CTA on the website through to final delivery. It is written for a developer team and intended to be used with LucidChart AI to generate a visual flowchart.

---

## SWIMLANES

The following swimlanes (actors/systems) should appear across the top:

1. **Patient (Frontend)**
2. **Website System (Backend)**
3. **ID Verification (LexisNexis API)**
4. **Clinical Team / Prescriber (Backend Dashboard)**
5. **Prescription System (SignatureRx API)**
6. **Order Processing & Dispatch**
7. **Royal Mail (API)**

---

## FLOW START

### PHASE 1: Product Selection & Consultation Entry

1. [START] Patient is on a product page (e.g. Regaine, Finasteride)
2. Patient clicks "Start Your Consultation" or "Start Your Hair Loss Assessment" CTA
3. → System routes patient to the consultation form for that product category (e.g. Hair Loss)
4. → Consultation form loads immediately — no login or account required at this stage
   - This reduces friction: anyone can start and complete a consultation without signing up first
   - Account creation / login is only required later at checkout (Phase 3, Step 9a)
   - → proceed to Step 5

---

### PHASE 2: Consultation Form

5. System loads the consultation form for the relevant category
   - Forms are configurable in the backend (questions, order, logic rules)
   - Each question is displayed one at a time or in sections

6. Patient answers each question in the consultation form
   - Questions cover: medical history, current medications, allergies, symptoms, previous treatments, lifestyle factors, etc.
   - Patient must answer all required fields to proceed

7. → System runs auto-check logic on each answer in real time
   - Decision: Does any answer trigger an automatic disqualification rule?
     - YES → go to Step 8 (AUTO-DENIAL)
     - NO → continue to next question until form is complete → go to Step 7a

7a. Patient confirms consent before submission
   - Must tick: acceptance of T&Cs + agreement to remote consultation
   - Consent is also required at reorder — patient confirms they can continue taking the treatment
   - After subscription ends or the **6-month hard limit**, a full new consultation form is required
   - → go to Step 9

---

### PHASE 2a: Auto-Denial Path

8. [AUTO-DENIAL] System flags the consultation as ineligible based on form logic
   - Example: patient selects a medical condition that is a contraindication for the treatment
   - → Patient sees a message: "Based on your answers, this treatment may not be suitable for you. We recommend speaking with your GP." 
   - → System logs the denial reason, timestamp, and patient ID
   - → System sends an email to the patient confirming the outcome
   - → Clinical team is notified in the backend dashboard (for awareness / follow-up if needed)
   - → [END — Auto-Denied]
   - Note: Patient may resubmit a new consultation. See Step 28 (repeat submission detection).

**Abandonment recovery (logged-in users only):**
   - If a logged-in patient starts the consultation form but does not complete it, the system logs the incomplete session
   - After a configurable timeframe (e.g. 1 hour / 24 hours), the system sends a reminder email: "You started a consultation but didn't finish — pick up where you left off"
   - Only works for logged-in users (we need their email to send the reminder)
   - Non-logged-in abandonments cannot be recovered (no contact details available)

---

### PHASE 3: Consultation Submitted & Payment

9. Patient completes all questions and submits the consultation form
10. → System saves the full consultation record:
    - All questions and answers (clearly mapped)
    - Timestamp of submission
    - Product/category requested
    - Consultation unique reference number

9a. → Decision: Is the patient logged in?
    - **YES** → proceed to Step 10a (payment)
    - **NO** → Patient is prompted to log in or create an account
      - New account collects: full name, date of birth, email, phone, delivery address
      - Account is created / patient logs in
      - → Consultation record is linked to the patient account
      - → proceed to Step 10a (payment)

10a. → Patient is taken to the payment screen
    - Payment is taken in full at this point (not a pre-auth hold — full capture)
    - → Decision: Payment successful?
      - **YES** → System stores payment reference against the consultation → Patient sees confirmation: "Payment received — your consultation is now being reviewed" → go to Step 11
      - **NO** → Patient shown a payment error and can retry
    - Note: If the order is later rejected (ID rejection in Phase 4, or clinical denial in Phase 5b), the payment is refunded in full automatically

**Patient cancellation (pre-dispatch):**
   - Patient has a "Request Cancellation" button in their account — available for a limited time window after payment (before dispatch)
   - Button sends a message/email to the pharmacy team via the messaging centre
   - Team manually processes the cancellation and issues a refund
   - This is a manual process — not an instant auto-cancel

11. → Decision: What type of product is being requested?
    - **POM (Prescription Only Medicine)** — e.g. Finasteride, Oral Minoxidil → go to Step 12 (ID Check required before clinical review)
    - **P (Pharmacy Medicine)** — e.g. Regaine, Viagra Connect → go to Step 12 (ID Check still runs, but a pharmacist has the option to review rather than a prescriber)

---

### PHASE 4: ID Verification

12. → System triggers ID verification via LexisNexis IDU API
    - Sends: patient full name, date of birth, address
    - LexisNexis runs identity and address verification checks

13. → Decision: Does the ID check pass?
    - **PASS** → System records: ID check status = Verified, timestamp, reference number → go to Step 16
    - **FAIL** → go to Step 14
    - **REFER (inconclusive)** → go to Step 14

14. [ID CHECK FAILED / REFERRED]
    - → System flags the order in the backend dashboard as "ID Check — Action Required"
    - → System sends the patient an email/message: "We need to verify your identity. Please upload a photo ID and proof of address (Driving License can account for both)"
    - → Patient can upload documents via:
      - The patient messaging centre on their account, OR
      - A secure upload link sent via email

15. → Decision: Does the clinical team manually verify the uploaded documents?
    - **YES** → Team manually marks ID status = Verified in the backend → go to Step 16
    - **NO** → Team marks ID status = Failed → Order status changes to "Rejected — Awaiting Refund" → Team issues refund via payment processor → Payment processor sends refund confirmation webhook to website → Order status changes to "Rejected & Refunded" → Patient is emailed that the order cannot proceed and a refund has been issued → [END — ID Rejected & Refunded]
    - Note: The patient rejection email is only sent AFTER the refund is confirmed. If the refund hasn't been processed yet, the order remains visible in the "Rejected — Awaiting Refund" queue.
    - Note: The backend should also allow clinical staff to manually trigger a new LexisNexis check if the patient updates their details.

---

### PHASE 5: Prescriber Notification & Clinical Review Queue

16. Consultation enters the clinical review queue in the backend dashboard
    - The dashboard shows:
      - Patient name and details
      - Consultation reference number
      - Product/category requested
      - Full consultation form (all questions and answers, clearly displayed)
      - ID check status (must show Verified before review can proceed)
      - Repeat submission flag (see Step 28) — if the patient has submitted multiple forms recently, this is highlighted
    - → Decision: Order cannot be progressed unless ID check = Verified

16a. → **System sends a notification email to the relevant clinician/prescriber**
    - → Decision: What type of product?
      - **POM item** → email is sent to the assigned prescriber(s)
      - **P item** → email is sent to the assigned pharmacist(s)
    - The notification email contains LIMITED, non-identifiable details only:
      - Consultation reference number (e.g. VM-HL-20260319-A3F2)
      - Product category (e.g. "Hair Loss")
      - Product type (POM or P)
      - Date/time submitted
      - A direct link to review the case in the backend dashboard
    - The email does NOT contain:
      - Patient name or date of birth
      - Consultation answers or medical details
      - Any clinical information
    - The link in the email points to the specific case (e.g. /admin/consultations/{id}/review)
    - → When the prescriber clicks the link:
      - If already logged in → they land directly on that patient's case
      - If not logged in → they are redirected to the login page, and after successful login, forwarded to the case
    - Note: The prescriber must also be logged into SignatureRx separately to issue prescriptions (see Phase 6). The website backend shows the treatment review and patient notes — the prescription itself is created and signed inside SignatureRx.
    - → Notification preferences: prescribers can opt for immediate email per order, or a batched summary (e.g. every 2 hours, or a daily digest at a set time). This is configurable per prescriber account in the backend.

17. → Decision: Who reviews?
    - **POM items** → must be reviewed by a registered prescriber (prescriber role account)
    - **P items** → can be reviewed by a pharmacist (pharmacist role account)

18. Clinician / Prescriber opens the patient case and reviews:
    - All consultation answers
    - ID verification status
    - Any previous consultation submissions from this patient
    - Any notes or messages from the patient

19. → Decision: Is the clinician satisfied that the treatment is appropriate?
    - **APPROVE** → go to Step 20
    - **REQUEST MORE INFO** → go to Step 19a
    - **DENY** → go to Step 19b

---

### PHASE 5a: More Information Requested

19a. Clinician clicks "Request More Information"
   - Clinician types a message/question for the patient
   - → System sends the message to the patient via the messaging centre and email notification
   - → Consultation status changes to "Awaiting Patient Response"
   - → Patient replies via the messaging centre
   - → Clinician is notified of the reply
   - → Clinician reviews the response and returns to Step 19 (Approve / Deny / Request More Info again)

**Consultation timeout / expiry:**
   - System flags consultations pending review for more than 7 days — visible in dashboard
   - If clinician requested more info and patient hasn't replied — re-flagged after 7 days
   - After 30 days of inactivity, order is deemed "abandoned" — pharmacy team can manually reject and refund
   - Consultations are flagged, NOT auto-expired — the team makes the final call

---

### PHASE 5b: Clinical Denial

19b. Clinician clicks "Deny Treatment"
   - → System requires the clinician to:
     - Select a denial reason (from a predefined list)
     - Add free-text clinical notes (mandatory)
   - → System records: denial decision, reason, notes, clinician ID, timestamp
   - → Order status changes to "Rejected — Awaiting Refund"
   - → Team issues refund via payment processor (e.g. Stripe dashboard)
   - → Payment processor sends refund confirmation webhook to website
   - → Order status changes to "Rejected & Refunded"
   - → System sends the patient an email: "After reviewing your consultation, we're unable to supply this treatment. [Reason summary]. A full refund has been issued. We recommend speaking with your GP."
   - → Clinical team can optionally send a more detailed message via the messaging centre
   - → [END — Clinically Denied & Refunded]
   - Note: The patient rejection email is only sent AFTER the refund is confirmed. If the refund hasn't been processed yet, the order remains visible in the "Rejected — Awaiting Refund" queue.

---

### PHASE 6: Prescription Generation & E-Signing (POM items only)

Note: For P (Pharmacy) medicines, skip to Step 24 (pharmacist approval is recorded differently — no formal prescription is generated, but the approval decision, pharmacist ID, and timestamp are still logged).

**Important — where things happen:**
- **Treatment review and patient notes** → happen on the Vivamen website backend. The prescriber reviews the consultation answers, ID check, patient history, and makes their clinical decision here.
- **The prescription itself** → is created, written, and signed INSIDE SignatureRx. The website backend sends the patient/product data to SignatureRx via API, and the prescriber completes the prescription within SignatureRx's system. The signed prescription details are then sent back to the website via a callback/webhook.

20. Prescriber clicks "Approve Treatment" for a POM item on the Vivamen backend
    - → The website system gathers the prescription data to send to SignatureRx:
      - Patient full name
      - Patient date of birth
      - Medication name
      - Strength
      - Form (tablet, solution, etc.)
      - Dose
      - Frequency
      - Quantity or duration
      - Directions
      - Date of issue
      - Prescriber identity (auto-filled from their account)
    - → Prescriber can review and adjust any field except patient name and date of birth before it's sent
    - → System validates: all required fields must be completed before proceeding

21. Prescriber clicks "Generate Prescription"
    - → **API CALL OUT — Website calls the SignatureRx API: CREATE PRESCRIPTION**
      - Sends: all prescription fields, prescriber ID, patient ID, consultation reference
      - SignatureRx creates the prescription record in its system
      - SignatureRx returns: a prescription reference ID

22. → **Prescriber completes and signs the prescription inside SignatureRx**
    - Everything that happens at this step (review, MFA, signing) is handled entirely within SignatureRx — not our system
    - The prescriber finalises the prescription and signs it within the SignatureRx platform

23. → **API CALL BACK — SignatureRx sends a webhook/callback TO the Vivamen website with:**
      - Signed prescription reference ID
      - Full prescription details (all fields as signed)
      - Signature status = Signed
      - Cryptographic hash of the prescription content (SHA-256)
      - Timestamp of signing
      - Prescriber details as recorded
    - → Vivamen website receives the callback and stores:
      - The full signed prescription record
      - The hash (for tamper-evidence — if any field changes, the hash won't match)
      - Signature timestamp
      - Prescriber ID
      - Prescription version number
      - SignatureRx reference ID (links the website record to the SignatureRx record)
    - → System logs the signing event in the audit trail:
      - Who signed
      - When they signed
      - What they signed (exact content, referenced by hash)
    - → Prescription is now LOCKED on both systems — no further edits allowed on this version
    - → Consultation status changes to "Approved — Prescription Signed"
    - → go to Step 24

---

### PHASE 6a: Prescription Void & Replacement (error correction)

If the prescriber realises the prescription contains an error (wrong drug, wrong dosage, incomplete fields, etc.), the following flow applies. This can happen at any point after the prescription has been signed.

23a. Prescriber identifies an error in a signed prescription
    - → Prescriber logs into SignatureRx and voids/cancels the incorrect prescription within SignatureRx
    - → **SignatureRx sends a webhook/callback to the Vivamen website with:**
      - Prescription reference ID
      - New status = **Voided / Cancelled**
      - Reason for void (if provided)
      - Timestamp of cancellation
      - Prescriber ID who voided it

23b. → Vivamen website receives the void callback and updates its records:
    - The original prescription record is marked as **Voided** (not deleted — kept for audit)
    - The voided prescription remains viewable in the backend but is clearly flagged as "Voided / Cancelled"
    - The consultation status changes back to "Approved — Awaiting New Prescription"
    - → The order is blocked from progressing to dispatch until a valid signed prescription is in place

23c. → Prescriber creates a corrected prescription within SignatureRx
    - This follows the same flow as Steps 21–23:
      - → Website calls SignatureRx API — CREATE PRESCRIPTION (with corrected details)
      - → Prescriber reviews and signs the new prescription within SignatureRx
      - → SignatureRx sends a callback/webhook to the website with the new signed prescription
    - → Website stores the new prescription as a new version, linked to the same consultation
    - → The new prescription becomes the **active valid prescription** for this order
    - → Consultation status changes back to "Approved — Prescription Signed"
    - → Order can now proceed to dispatch

23d. The backend must clearly show the full prescription history for any consultation:
    - Version 1: [Voided] — original Rx, signed on [date], voided on [date], reason: [reason]
    - Version 2: [Active] — corrected Rx, signed on [date]
    - (And any further versions if additional corrections were needed)
    - → Only the most recent **non-voided** prescription is treated as the valid one for dispatch purposes
    - → All versions are permanently stored and retrievable for audit/inspection

---

### PHASE 7: Payment & Order Confirmation

24. Payment was already taken in full at consultation submission (Phase 3, Step 10a). No further payment action is needed at this stage — the system links the existing payment reference to the order record.

25. → The system:
    - Generates the order record with a unique order number
    - Links the order to: consultation, prescription (if POM), patient account, clinician approval record
    - → **Sends approval email to patient:** "Your consultation has been approved and your order is being prepared"
    - → Triggers GP letter generation (see Step 26)
    - → Pushes patient data to TitanPMR (see Step 27)
    - → Moves order to dispatch queue (see Step 29)

---

### PHASE 8: GP Letter Generation

26. System auto-generates a GP notification letter (PDF) using a predefined template
    - Contains: patient name, DOB, medication prescribed/supplied, date, prescriber/pharmacist details
    - → The letter is stored against the patient record in the backend
    - Note: GP letter is generated for ALL approved orders (POM and P items)

26a. → Decision: Is a GP email address available and valid?
    - **YES (collected during consultation)** → System auto-emails the GP letter to the GP
      - → Decision: Did the email bounce or fail delivery?
        - **NO** → GP letter sent successfully
        - **YES** → System flags the order in the backend as "GP Email Failed"
          - → Clinical team can see this flag on the order screen
          - → Clinical team can input a corrected GP email address on that screen
          - → Inputting the email triggers the GP letter to be resent
    - **NO (not collected during consultation)** → System flags the order in the backend as "GP Email Missing"
      - → Clinical team can input the GP email address directly on the order/GP letter screen
      - → Inputting the email triggers the GP letter to be sent

26b. In all cases:
    - The GP letter is available for the clinical team to review, print, or email at any time
    - The GP email send status is visible on the order screen: Sent / Failed / Pending / Not Collected
    - The clinical team always has a manual input field on the order screen to enter or correct the GP email and trigger delivery

---

### PHASE 9: PMR Integration & Order Print Slip

**Function 1 — TitanPMR sync:**

27. → System pushes patient and order data to TitanPMR via API (if integration is available)
    - Sends: patient details, medication supplied, prescriber details, date
    - Note: TitanPMR integration scope to be confirmed with Titan — see https://www.titanpmr.com/sectors/online-pharmacies
    - If API integration is not available, the backend should allow clinical staff to manually export/enter data into TitanPMR

**Function 2 — Order print slip:**

27a. The Approved Orders screen in the backend includes a "Print Order Slip" button on each order
    - This button is ONLY available on the Approved Orders screen — not visible on pending, denied, or draft orders
    - When clicked, the system generates a hardcoded A4/A5 print slip template populated with:
      - Patient full name
      - Patient date of birth
      - Patient address
      - Delivery address (if different)
      - Item / medication name
      - Pack size
      - Quantity
      - Dosage / directions
      - Approved by: prescriber or pharmacist name
      - Approval date and timestamp
      - Order reference number
      - Consultation reference number
    - The print slip is sent directly to the printer
    - This is for internal pharmacy use — it accompanies the physical order during picking, packing, and dispatch as a paper summary
    - The template is hardcoded (not configurable by staff) — it always pulls the same fields in the same layout

**Stock management (future feature — not required for initial build):**
   - Ability to input and maintain stock levels per product in the backend
   - System checks stock before allowing dispatch — blocks order if out of stock
   - Optional: low-stock alerts to pharmacy team

---

### PHASE 10: Repeat Submission Detection

28. The system must flag when a patient submits multiple consultation forms within a defined time window (e.g. 7 days)
    - → When a new consultation is submitted, the system checks:
      - Has this patient submitted another consultation for the same category recently?
      - If YES → flag the new submission with a warning: "Repeat submission detected — [X] submissions in the last [Y] days"
    - → This flag is visible to the clinician when they open the case (Step 18)
    - → The system should also check if answers differ significantly between submissions (optional advanced feature)
    - → Clinicians can view all previous submissions from the patient in a history panel

---

### PHASE 11: Order Processing & Dispatch

29. Order appears in the dispatch queue in the backend
    - Order shows: order number, patient details, product(s), delivery address, approval status, prescription reference (if POM)
    - → Decision: All checks complete?
      - ID Verified = YES
      - Clinical approval = YES
      - Prescription signed = YES (if POM)
      - Payment confirmed = YES
      - If ANY of these = NO → order is blocked from dispatch and flagged

30. Staff picks and packs the order
    - Medication is dispensed and labelled
    - Order is marked as "Packed" in the system

31. → **System calls the Royal Mail API — CREATE SHIPMENT**
    - Sends: delivery address, package weight/dimensions, service type (Tracked 24/48)
    - Royal Mail API returns: tracking number, shipping label (PDF)

32. → System stores the tracking number against the order
33. → System generates and prints the shipping label
34. → System sends the patient an email: "Your order has been dispatched" with the tracking number and a link to track delivery

35. Package is collected by Royal Mail / dropped at post office
36. → Patient receives the package in plain, discreet, unbranded packaging

37. [END — Order Complete]

---

### PHASE 12: Post-Dispatch

38. System receives Royal Mail tracking update
    - → Decision: Delivery status?
      - **DELIVERED** → go to Step 39
      - **FAILED / RETURNED** → go to Step 38a

38a. [DELIVERY FAILED / RETURNED]
    - Order status changes to "Delivery Failed"
    - Order appears in a dedicated **"Delivery Failed"** screen in the backend
    - Pharmacy team is notified via email
    - → Decision: Team action?
      - **Re-dispatch** → Team updates address if needed, re-sends via Royal Mail API
      - **Contact patient** → Team messages patient via messaging centre to resolve (e.g. confirm address)
      - **Refund** → Team issues refund via payment processor → order moves to "Refunded"

39. Order status updated to "Delivered"
    - → Patient can access their order history, prescription records, and GP letters in their account

**Reorder flow:**

40. → Decision: Is the patient eligible to reorder?
    - Reorder eligibility is based on a **configurable timeframe** per product (e.g. 1-month supply → reorder available after ~2 weeks)
    - **YES (within active subscription / under 6-month limit):**
      - Patient clicks "Reorder" in their account
      - Patient must confirm consent: agree they can continue taking the treatment
      - Reorder skips the consultation form — goes straight to payment → dispatch
    - **NO (subscription ended OR 6-month hard limit reached):**
      - Patient must complete a full new consultation form (→ Phase 1)
      - The 6-month limit is a hard reset — ensures clinical review happens periodically for ongoing treatments

41. → Clinical team can view the full audit trail for the order at any time:
    - Consultation form and answers
    - ID check result
    - Clinical decision (approve/deny), reason, notes, clinician ID
    - Prescription record and signature details (hash, timestamp, prescriber)
    - Payment record
    - Dispatch and tracking info

---

## ERROR / EXCEPTION PATHS SUMMARY

| Scenario | What happens | Refund? | Status |
|----------|-------------|---------|--------|
| Auto-denial from form logic | Patient told treatment not suitable, advised to see GP. Clinical team notified. | No — payment not yet taken (before Phase 3 payment) | Auto-Denied |
| ID check fails | Patient asked to upload documents. If manual review also fails, order rejected. | Yes — team refunds via payment processor, webhook confirms | Rejected — Awaiting Refund → Rejected & Refunded |
| Clinician requests more info | Patient messaged via messaging centre. Consultation paused until reply. | No — order still active | Awaiting Patient Response |
| Clinician denies treatment | Reason logged, patient notified after refund confirmed, advised to see GP. | Yes — team refunds via payment processor, webhook confirms | Rejected — Awaiting Refund → Rejected & Refunded |
| MFA fails during e-sign | Signing blocked, event logged. Prescriber can retry or contact admin. | No — order still active, signing can be retried | Signing Failed |
| Payment fails at checkout | Patient shown error, can retry. | N/A — payment never captured | Awaiting Payment |
| Repeat submission detected | Flag shown to clinician during review. Previous submissions visible. | No — informational flag only | Flagged for Review |
| Post-sign prescription edit needed | New version created, old version locked. New version requires fresh signature. | No — order still active | New Version Required |
| Prescription voided in SignatureRx | Webhook received, website marks Rx as voided, order blocked until replacement Rx is signed and synced back. Old Rx kept for audit. | No — order still active, awaiting corrected Rx | Voided — Awaiting Replacement |
| Consultation abandoned (logged in) | Reminder email sent after configurable delay. Incomplete session logged. | N/A — before payment | Abandoned (Draft) |
| Patient requests cancellation | Message sent to pharmacy team. Team manually cancels and refunds. | Yes — manual refund | Cancelled → Refunded |
| Consultation timeout (30 days) | Flagged at 7 days, re-flagged if no reply. Team can reject/refund after 30 days. | Yes — manual refund | Abandoned → Refunded |
| Delivery failed / returned | Order enters Delivery Failed screen. Team notified via email. Can re-dispatch, contact patient, or refund. | Depends on team action | Delivery Failed |
| API unavailable (LexisNexis) | Order tagged "ID Check — Pending (API Unavailable)". System retries. | No — still active | Pending (API Down) |
| API unavailable (SignatureRx) | Order tagged "Rx — Pending (API Unavailable)". System retries. | No — still active | Pending (API Down) |
| API unavailable (Royal Mail) | Order tagged "Dispatch — Pending (API Unavailable)". System retries. | No — still active | Pending (API Down) |

---

## KEY API INTEGRATIONS

| System | API | Purpose | When it's called |
|--------|-----|---------|-----------------|
| LexisNexis IDU | Identity verification API | Verify patient name, DOB, address | After consultation is submitted (Step 12) |
| SignatureRx | Create Prescription endpoint | Send prescription data, receive Rx reference ID | When prescriber clicks "Sign Prescription" (Step 21) |
| SignatureRx | Webhook/callback TO website (Signed) | Returns signed Rx details, hash, timestamp back to the website | After prescriber signs inside SignatureRx (Step 23) |
| SignatureRx | Webhook/callback TO website (Voided) | Returns voided Rx reference, cancellation reason, timestamp | When prescriber voids a prescription in SignatureRx (Step 23a) |
| Royal Mail | Create Shipment endpoint | Generate shipping label + tracking number | After order is packed (Step 31) |
| Royal Mail | Tracking webhook (optional) | Receive delivery status updates | Post-dispatch (Step 38) |
| TitanPMR | Patient data push (TBC) | Sync patient + medication records to pharmacy PMR system | After order is confirmed (Step 27) |
| Payment Gateway | Full capture at checkout | Handle patient payment | Phase 3, Step 10a — after consultation submitted |
| Payment Gateway | Refund confirmation webhook | Confirms refund processed, moves order to "Rejected & Refunded" | When team issues refund from payment processor |

---

## OPTIONAL EXTRAS

Features we ideally want but can be removed or deferred if not practical to build in the initial release. These are **not blockers** — the core flow works without them. Each has a manual fallback already designed into the spec.

| Priority | Feature | What it does | Where in flow | Fallback if not built |
|----------|---------|-------------|---------------|----------------------|
| **HIGH** | **Stock / inventory management** | Input and maintain stock levels per product. System checks stock before dispatch and blocks if out of stock. Optional low-stock email alerts. | Phase 11 — Dispatch | Team manually tracks stock offline. No system-level block on out-of-stock orders. |
| **HIGH** | **TitanPMR API integration** | Automatically push patient + medication data to TitanPMR after order confirmation. Requires Titan enterprise plan. | Phase 9 — PMR | Staff manually exports/enters data into TitanPMR. Works without the API. |
| **MEDIUM** | **Royal Mail tracking webhook** | Automatic delivery status updates pushed from Royal Mail to the website. Updates order status to "Delivered" or "Failed" automatically. | Phase 12 — Post-Dispatch | Staff manually updates order status based on Royal Mail tracking portal. |
| **MEDIUM** | **Prescriber notification batching** | Configurable notification digest per prescriber: immediate, batched every 2 hours, or daily digest. | Phase 5 — Clinical Review | Send immediate email for every new consultation. Simpler, still functional. |
| **MEDIUM** | **Repeat submission answer-diff detection** | Compare answers between repeat submissions and highlight significant differences. Key indicator of system gaming. | Phase 10 — Repeat Detection | Clinician still sees the repeat flag and can manually view all previous submissions side by side. |
| **LOW** | **Low-stock email alerts** | Automated email to pharmacy team when stock drops below configurable threshold. | Phase 11 — Dispatch | Only relevant if stock management is built. Without it, team monitors stock manually. |

**Priority key:**
- **HIGH** = strong business value, build if timeline allows
- **MEDIUM** = nice to have, can defer to v2
- **LOW** = dependent on another optional feature being built first

---

## LUCIDCHART NOTES

- Use swimlanes for: Patient, Website System, LexisNexis API, Clinical Team / Prescriber, SignatureRx API, Dispatch, Royal Mail API
- Diamond shapes for all decision points (Steps 4, 7, 11, 13, 15, 17, 19, 22, 24, 29)
- Red paths for denial/rejection flows (Steps 8, 14→15 NO, 19b)
- Orange paths for "needs action" flows (Steps 14, 19a, payment fail)
- Green path for the happy path through to delivery
- Dashed lines for API calls between swimlanes (especially the SignatureRx create → callback flow)
- The SignatureRx flow should show two distinct API calls: one outbound (create Rx), one inbound (callback with signed Rx data)
- Group Phases 1–12 as labelled sections/containers in the chart
