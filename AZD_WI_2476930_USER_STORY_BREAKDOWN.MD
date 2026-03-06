# User Story Breakdown — Feature 2476930 (Minimum Rental/Employment History Enforcement)

- **Feature:** 2476930 — PME-451065 - Minimum Rental History/ Minimum Employment history Enforcement
- **Source of truth:** Feature AC + description + discussion/LOE notes captured in [AZD_WI_2476930_SUMMARY.md](AZD_WI_2476930_SUMMARY.md)
- **Primary UX surface:** New OneSite → Add/Edit Household Members (prospects/waitlist/applicants/residents)
- **Key dependency:** Mandatory Fields validation framework + LaunchDarkly state (validation may be unobservable while disabled)

---

## 0) Key Decisions / Assumptions to Confirm
These affect scope and should be explicitly confirmed before implementation locks:

1) **Config-driven minimum duration (X months)**
   - Minimum Rental/Employment History duration is configured at:
     - New OneSite → My Company → Company Details → Applicants → Applicants History
   - Confirm the exact source-of-truth field(s) + caching behavior.

2) **How to compute “combined duration”**
   - Confirm calculation rules for:
     - Overlaps/gaps between records
     - Partial months vs exact day-count conversion
     - Open-ended “current” residence/employment (end date = today?)

3) **Prompting UX behavior**
   - AC says “continue prompting until requirement is fulfilled.”
   - Confirm whether prompting occurs:
     - On submit only
     - As user adds entries (progress indicator?)
     - When user edits existing entries

4) **Record model and max record count**
   - AC implies “as many as needed.” Confirm any practical cap.

5) **LaunchDarkly / Mandatory Fields gating**
   - If Mandatory Fields is disabled, define how we validate during testing and how the feature behaves.

---

## 1) Existing Related Work (from links/discussion)
These items shape dependencies and cross-team alignment:

- **2480887** — Spike (Closed) — [API] tech analysis for multi-history + mergefield repeater
- **2749658** — Spike (Active) — [UI] tech analysis for multi-history + mergefield repeater
- **2741850** — Feature (Defined) — AOLR Application: Support More Than 2 Records of Employment/Rental History (alignment needed)
- **2483194** — Feature (New) — LOFT AOLR - Add new Additional Previous Address Fields (OneSite)
- Additional related bug(s) exist (e.g., 2428658 removed; 2716191 defined)

---

## 2) Proposed Story Backlog (recommended sequencing)
This mirrors how the work was discussed (validation + API changes largely known; UI remains key).

### Phase A — Validation + domain rules (shared logic)
1. **US-A1: Define duration aggregation rules** (spec clarification + examples)
2. **US-A2: Update validation engine to support N previous records (rental + employment)**
3. **US-A3: Update required-fields validation messaging for indexed/multiple entries**

### Phase B — Data shaping + API behavior
4. **US-B1: Prospects load APIs support multiple rental/employment nodes**
5. **US-B2: Prospects save APIs accept/persist multiple rental/employment entries**
6. **US-B3: Leased accounts endpoints for multi-record save (PUT) + persistence rules**
7. **US-B4: New OneSite “Select unit” retrieval supports multi-node XML**

### Phase C — UI changes (core end-to-end experience)
8. **US-C1: Household Members UI — “Additional Previous Residence” repeater**
9. **US-C2: Household Members UI — “Additional Previous Employment” repeater**
10. **US-C3: UI validation + tooltip/messaging for X-month requirement**

### Phase D — Merge fields + reporting integration
11. **US-D1: MergeField repeater support (DB script + validation)**

### Phase E — Testing, rollout, and coordination
12. **US-E1: End-to-end test plan (LD gating aware) + regression coverage**
13. **US-E2: Cross-feature alignment with AOLR/OLL (verify contract + shared behavior)**

---

## 3) Detailed Story Definitions

### US-A1 — Define duration aggregation rules (rental + employment)
**Type:** User Story / Spec task

**Description**
Clarify how combined rental/employment history duration is computed when more than two records exist.

**Acceptance Criteria**
- Documented examples for:
  - Multiple records where total is < X then becomes ≥ X
  - Overlapping date ranges (if allowed)
  - Missing dates / partial entries
  - Current record with no end-date
- Agreed conversion rules (days→months) used consistently across validation + UI prompts.

**Dependencies**
- Product + Engineering agreement; aligns with Mandatory Fields behavior.

---

### US-A2 — Multi-record validation: rental history duration sufficiency
**Maps to Feature AC:** Rental scenario (prompt until meets X)

**User Story**
As the system, I want to validate combined rental history across current + all previous residences so that users are required to provide enough history to meet the configured minimum.

**Acceptance Criteria**
- Validation checks combined duration against configured minimum X months.
- Supports N previous records (not limited to 2).
- Produces actionable validation errors when insufficient.

**Implementation notes (from LOE discussion)**
- PriorResidence validation refactor: accept lists, aggregate duration, add indexed suffixes on missing fields.

---

### US-A3 — Multi-record validation: employment history duration sufficiency
**Maps to Feature AC:** Employment scenario (prompt until meets X)

**User Story**
As the system, I want to validate combined employment history across current + all previous employments so that users are required to provide enough history to meet the configured minimum.

**Acceptance Criteria**
- Same as rental validation behavior, mirrored for employment.
- Indexed validation messaging supported.

---

### US-B1 — Prospects load: return multiple rental/employment history records
**User Story**
As a user, I want the Household Members screen to load all existing rental/employment history records so I can review/edit them.

**Acceptance Criteria**
- Load includes all rental/employment history entries (not only current + one previous).
- Ordering rules are defined (e.g., newest→oldest) and consistent.

**Implementation notes (from LOE discussion)**
- Replace single-node XPath selection with multi-node iteration.

---

### US-B2 — Prospects save: validate + persist multiple rental/employment entries
**User Story**
As a user, I want to save multiple rental/employment history records so that the system can meet the configured minimum history requirement.

**Acceptance Criteria**
- Save accepts a list of entries.
- Save persists all entries and removes appropriately deleted entries.
- Validation occurs before commit and returns clear errors per entry.

**Implementation notes (from LOE discussion)**
- Validation functions refactored to iterate multiple addresses/employments and aggregate duration.

---

### US-B3 — Leased accounts saving: multi-record PUT endpoints + persistence rules
**User Story**
As an API consumer, I want endpoints that support creating/updating multiple rental/employment history records for leased accounts so that leased-account flows remain consistent with multi-record requirements.

**Acceptance Criteria**
- Provide/confirm PUT endpoints for rental/employment history saving.
- Supports insert/update detection by record IDs.
- Deletion rules align with system requirements (hard delete vs soft delete per entity).

---

### US-B4 — New OneSite “Select unit” retrieval supports multi-record XML
**User Story**
As a user, I want the “Select unit” and related flows to correctly retrieve multi-record rental/employment history so the UI and downstream validation are consistent.

**Acceptance Criteria**
- Retrieval logic supports multi-node XML for rental/employment history.
- No regressions for waitlist flows already looping.

---

### US-C1 — UI: Additional Previous Residence section (repeater)
**Maps to Feature AC:** Additional Previous Residence section + tooltip

**User Story**
As a user, when my combined rental history is below the configured minimum, I want the UI to prompt me to add additional previous residences until I meet the requirement.

**Acceptance Criteria**
- UI displays section titled **“Additional Previous Residence”** when needed.
- Section includes fields listed in the feature AC.
- Tooltip displays: “Required to meet the minimum X months rental history requirement.”
- UI continues prompting until combined duration ≥ X.

---

### US-C2 — UI: Additional Previous Employment section (repeater)
**Maps to Feature AC:** Additional Previous Employment section + tooltip

**User Story**
As a user, when my combined employment history is below the configured minimum, I want the UI to prompt me to add additional previous employments until I meet the requirement.

**Acceptance Criteria**
- UI displays section titled **“Additional Previous Employment”** when needed.
- Section includes fields listed in the feature AC.
- Tooltip displays: “Required to meet the minimum X-month employment history requirement.”
- Continues prompting until combined duration ≥ X.

---

### US-C3 — UI validation + messaging for indexed/multiple entries
**User Story**
As a user, I want validation errors to clearly identify which entry is missing/invalid so I can correct it efficiently.

**Acceptance Criteria**
- Validation messages map to specific entry indices (e.g., first/second additional history item).
- Errors are consistent across prospects/waitlist/applicants/residents.

---

### US-D1 — MergeField repeater support for multi-history
**User Story**
As a user generating documents/exports, I want merge fields to support repeating sets of rental/employment history so communications remain accurate.

**Acceptance Criteria**
- DB scripts added to MergeField/MergeFieldDetail for repeaters as needed.
- Validation exists for new merge fields.

---

### US-E1 — Test plan + rollout verification (LD gating aware)
**User Story**
As a team, we want a test plan that validates multi-history enforcement end-to-end across flows and handles LaunchDarkly gating so we can release confidently.

**Acceptance Criteria**
- Test scenarios for:
  - X months set to values other than 12
  - Multi-record aggregation meets/doesn’t meet threshold
  - Save + reload retains all records
  - Role/state permutations: prospects, waitlist, applicants, residents
- Explicit validation strategy when Mandatory Fields is disabled via LaunchDarkly.

---

### US-E2 — Cross-feature alignment (AOLR/OLL)
**User Story**
As the organization, we want OneSite and AOLR/OLL to stay in sync for multi-record history requirements so applicants have consistent behavior across products.

**Acceptance Criteria**
- Validate contract compatibility with Feature 2741850.
- Ensure no duplicate or conflicting implementations.

---

## 4) Estimate Pointers (from discussions)
- Work item field: **L (3–4 sprints)**
- Comment-based LOE snapshots exist (detailed API tasks, QA days, UI TBD). Treat as planning artifacts unless captured in formal estimate fields.

---

## 5) Suggested “Definition of Done” for the Feature
- Users can enter **N** rental/employment history entries until total ≥ configured minimum.
- Validation prompts and messaging work end-to-end on Add/Edit Household Members.
- Prospects + leased accounts + select-unit retrieval are consistent.
- Mergefields (if in scope) support repeaters.
- Verified under expected LaunchDarkly/Mandatory Fields state(s).
