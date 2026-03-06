# User Story Breakdown — Feature 2380310 (De‑Identification Queue + UI Indicator)

- **Feature:** 2380310 — UI Indicator and Dedicated Page for Accounts Scheduled for De‑Identification (Onesite De Identification Rollout)
- **Source of truth:** Feature acceptance criteria + discussion notes (PMC vs property vs guest card; exemption applies across associated properties)
- **Audience:** PM/Eng planning + AI project management ingestion

---

## 0) Key Decisions / Assumptions to Confirm (from discussions)
These items materially affect scope and should be explicitly confirmed with Product/Legal/Architecture before implementation locks:

1) **“Restore” expectation**
   - The feature description states: provide a **90‑day “coming” window**; after DE‑ID, **no rollback**.
   - Confirm messaging to users: “irreversible after 90 days” vs any actual restore mechanism.

2) **Entity + eligibility evaluation rules (Nabil/Leelakrishna summary)**
   - DE‑ID is **driven at PMC level**, evaluated across **all associated properties** for a guest card/resident.
   - A property may DE‑ID if eligible, even if PMC cannot.
   - **PMC DE‑ID occurs only when all associated properties qualify**.

3) **Exemption semantics**
   - Exempting a **guest card** exempts **all linked property records**.

4) **Pages: separate vs single page**
   - Current feature AC includes a dedicated queue page **and** a separate exemption list page.
   - Some discussion suggested merging pages via filtering.
   - If Product wants “single page”, update feature AC to avoid building both.

---

## 1) Existing Linked User Stories (already in AZD)
These are linked to the feature and likely represent the core breakdown already started:

- **2394913** — UI indicator on Account Overview (AC1)
- **2394914** — Dedicated DE‑ID queue page + export (AC2 + parts of AC2.5)
- **2394915** — Auto‑queue accounts older than 10 years (AC5.1, AC5.2)
- **2394916** — Batch processing every 90 days (AC5.4, AC5.5)
- **2394917** — Exempt flag + display (AC2.2, AC4.1)
- **2394918** — Exempted accounts reappearance rules (AC4.2, AC4.4 + exemption persistence)

**Gaps vs Feature AC / discussions:** RBAC rights + default role mapping (AC3), auditability/timestamping (AC5.6), and the clarified PMC/property/guest-card gating rules are not explicitly represented in the linked stories (based on their titles).

---

## 2) Proposed Story Backlog (recommended sequencing)
This keeps stories small enough for parallel work and makes dependencies explicit.

### Phase A — Rules, data, and batch mechanics (backend‑first)
1. **US-A1: Define & document business rules for DE‑ID eligibility** (non-code deliverable)
2. **US-A2: Implement eligibility evaluation + queue fill (10+ years) with PMC/property/guest-card gating**
3. **US-A3: Assign scheduled DE‑ID date + batch run timestamping (auditability)**
4. **US-A4: Batch generation every 90 days with “fixed list” / no rolling updates**
5. **US-A5: Exemption write model (exempt guest card across linked properties) + re-queue behavior**

### Phase B — API surface for UI
6. **US-B1: DE‑ID queue read API** (pagination/sorting/search/filters)
7. **US-B2: Export API (CSV/Excel)**
8. **US-B3: Exemption list read API** (if separate exemption page is kept)

### Phase C — UI + RBAC
9. **US-C1: RBAC rights + default role mapping**
10. **US-C2: Account Overview indicator**
11. **US-C3: DE‑ID queue page**
12. **US-C4: Exemption list page** (if separate page is kept)

---

## 3) Detailed Story Definitions

### US-A1 — Document DE‑ID rules + signoff (Product/Legal)
**Type:** User Story (non-code) / Task

**Description**
Document and obtain sign-off for:
- 90‑day “coming” window semantics and irreversibility messaging
- Eligibility definition for “10+ years” (creation vs last active date)
- PMC vs property vs guest card gating rules
- Exemption semantics (guest card exemption applies across linked properties)

**Acceptance Criteria**
- A written policy doc exists and is reviewed/approved by Product + Legal.
- The doc defines: inputs, outputs, and examples for edge cases (multi-property mixed statuses).

**Dependencies**
- Product, Legal, Architecture.

---

### US-A2 — Eligibility evaluation + queue fill (10+ years) with gating rules
**Maps to Feature AC:** AC5.1, AC5.2 (+ discussion gating rules)
**Likely aligns with existing:** 2394915 (extend/clarify)

**User Story**
As a system, I want to identify and queue accounts eligible for de-identification so that the DE‑ID process can be reviewed and executed on a predictable schedule.

**Acceptance Criteria**
- System identifies accounts that meet the 10+ year rule (per policy-defined date basis).
- Queue is filled during scheduled maintenance windows.
- Accounts from exempt sites are excluded.
- Evaluation respects PMC/property/guest-card rules:
  - Queue population begins at PMC scope.
  - Per-property eligibility is evaluated and stored/derivable.
  - PMC eligibility requires all linked properties qualify.

**Notes**
- Ensure the data model supports showing “# associated properties” and “per-property eligibility” in the UI.

---

### US-A3 — Scheduled DE‑ID date assignment + audit fields
**Maps to Feature AC:** AC5.3, AC5.6

**User Story**
As a system, I want each queued item to have an assigned scheduled DE‑ID date and traceable batch metadata so that the queue is auditable and supports a 90‑day review window.

**Acceptance Criteria**
- Each queued record has:
  - Queueing date
  - Scheduled DE‑ID date exactly 90 days after queueing date
  - Batch run date (or batch identifier)
- Metadata is queryable for UI display and audit.

---

### US-A4 — Batch generation every 90 days (fixed list / no rolling updates)
**Maps to Feature AC:** AC5.4, AC5.5
**Likely aligns with existing:** 2394916 (extend/clarify)

**User Story**
As a system, I want DE‑ID candidates to be processed in fixed 90‑day batches so that the set of accounts under review remains stable for the full window.

**Acceptance Criteria**
- Batch job runs every 90 days.
- Once a batch is generated, its list remains unchanged until the next cycle.
- Newly eligible accounts after batch generation go to the next batch (if that is the agreed rule).

---

### US-A5 — Exemption write model + re-queue rules
**Maps to Feature AC:** AC4.1, AC4.2, AC4.4
**Likely aligns with existing:** 2394917, 2394918 (extend to guest-card semantics)

**User Story**
As an authorized user, I want to exempt a guest card from DE‑ID (and remove exemptions later) so that we can prevent DE‑ID for special cases.

**Acceptance Criteria**
- Exempt action sets `Exempted = Yes`.
- Exempt action applies to all records linked to the guest card across properties.
- Remove exemption sets `Exempted = No`.
- After removing exemption, if eligible (10+ years), the record becomes eligible for the next 90-day batch.

---

### US-B1 — DE‑ID queue read API (list/search/filter/sort/page)
**Maps to Feature AC:** AC2.1, AC2.3, AC2.4

**User Story**
As a user, I want to view a DE‑ID queue list that supports searching, filtering, sorting, and pagination so that I can quickly find and manage queued accounts.

**Acceptance Criteria**
- API returns the required columns:
  - Account Name, Site Name, Account Type, Scheduled DE‑ID Date, Exempted Yes/No
- Supports:
  - Search by any column
  - Filters: Exemption (Y/N), Account Type, Source (Find Accounts)
  - Sorting by any column
  - Pagination: 20 per page (default)
- Response reflects latest scheduled DE‑ID date (real-time accuracy).

**Discussion-driven additions (confirm)**
- Include guest card view elements: guest cards, associated property count/list, per-property 10-year eligibility.

---

### US-B2 — Export API (CSV/Excel)
**Maps to Feature AC:** AC2.5

**User Story**
As a user, I want to export the DE‑ID queue to CSV/Excel so that I can review and share the list offline.

**Acceptance Criteria**
- Export supports CSV and Excel.
- Export respects current filters/search (if that is desired) or is clearly defined.

---

### US-B3 — Exemption list read API
**Maps to Feature AC:** AC4.3 (if separate exemption page remains)

**User Story**
As a user with edit rights, I want to view the list of exempted accounts so that I can manage exemptions.

**Acceptance Criteria**
- API returns all currently exempted accounts.
- Supports basic paging/sorting (at minimum consistent with queue list).

---

### US-C1 — RBAC rights + default role mapping
**Maps to Feature AC:** AC3.1, AC3.2, AC3.3

**User Story**
As a system administrator, I want role-based access control for DE‑ID queue features so that only authorized users can view or manage the queue/exemptions.

**Acceptance Criteria**
- Users without “View the identification queue” cannot access the queue UI.
- Only users with “Edit the de-identification queue” can exempt/unexempt.
- Edit right is non-distributable to custom roles.
- Default roles listed in feature AC have both view and edit rights.

---

### US-C2 — Account Overview indicator for scheduled DE‑ID
**Maps to Feature AC:** AC1.1–AC1.3
**Likely aligns with existing:** 2394913

**User Story**
As a user viewing an account, I want a clear indicator when the account is scheduled for DE‑ID (with date) so that I understand upcoming changes.

**Acceptance Criteria**
- Show indicator on account overview page only when:
  - Account is actively queued
  - Site is not exempt
- Display scheduled DE‑ID date.
- Indicator uses latest DE‑ID date from the system.

**Note**
- AC1.4 (“irreversibility notice”) may be best implemented as copy/text adjacent to the indicator or in the queue UI—confirm desired placement.

---

### US-C3 — DE‑ID queue page
**Maps to Feature AC:** AC2.1–AC2.5
**Likely aligns with existing:** 2394914

**User Story**
As an authorized user, I want a dedicated DE‑ID queue page so that I can review upcoming de-identifications and manage exemptions.

**Acceptance Criteria**
- Displays required columns.
- Has ellipsis actions per row:
  - Exempt from queue
  - Remove exemption
- Supports search/filters/pagination/sorting.
- Supports export.

---

### US-C4 — Exemption list page
**Maps to Feature AC:** AC4.3

**User Story**
As a user with edit rights, I want a page to view all exempted accounts so that I can remove exemptions when appropriate.

**Acceptance Criteria**
- Displays all exempted accounts.
- Provides a remove exemption action.

**Alternative**
- If Product chooses “single page”, this becomes a filter view within the main queue page; update feature AC accordingly.

---

## 4) Estimate Pointers (from discussions)
- Feature field: **XL (4+ sprints)**
- Comment sizing (81 SP total) suggested buckets:
  - Queue management: 21
  - Exemption handling: 34
  - Scheduled batch processing: 13
  - UI enhancements: 13

Use these buckets to validate the story breakdown, but keep final estimates owned by the delivery team.
