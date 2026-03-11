# Demo Review Report – Work Item 2647905

**Work Item:** 2647905
**Title:** PME-484256 - EXPEDITE | PME DRIVER | Camden Thornton Park - Applicant Shanell Paraison - Unable to Apply Reservation to Unit in New Onesite
**Type:** User Story
**State:** Closed
**Assigned To:** Suvarnamadhuri Batchu
**Sprint:** PI 26.Q1 (Sprint 26.05)
**Story Points:** 8
**Area:** PropertyManagement\OneSite Platform\Koopalings
**Tags:** 25.26; AI_Analysis_Done; CAMDEN DEVELOPMENT INC.; Expedite; External PME; Multiple Escalations; Top 50 PMC
**Created:** 2025-12-17
**Escalation ID:** PME-484256
**Feature Flag:** reservation_commit_decoupled_from_quote (Off)
**Functional Area:** Accounts - Prospects / Quotes
**Domain:** OSPM: Lead Mgmt
**Review Date:** 2026-03-11
**Reviewed By:** AI Demo Review Agent v1
**Guidelines Reference:** os-demo-readiness.guidelines.md (v2, Mar 02, 2026)
**Data Source:** Azure DevOps REST API (verified work item fields)

---

## 1. Business Context & Value

**Assessment: Fail**

### Evidence

The presenter opens with:

> *"0:01 Hi everyone as a part of the Sprint we delivered this item."*
> *"0:05 So for the reserved one and the expired code we are not able to do the select unit process in new one site experience."*

### Analysis

- **Was the business problem clearly stated?** No. The presenter describes the technical symptom ("not able to do the select unit process") but does not explain the underlying business problem — that a gap exists between Classic OneSite and New OneSite where onsite teams cannot apply a reservation to a unit when the quote has expired but the reservation is still valid.
- **Was customer impact explained?** No. There is no mention of the affected customer (Camden Development, Inc. / Camden Thornton Park), no reference to the PME (PME-484256), and no mention that this is an Expedite-tagged item with multiple escalations (2 PME; 3 SF Cases) from a Top 50 PMC.
- **Was "why" clearly answered?** No. The demo does not explain why this matters: onsite leasing teams were unable to complete the leasing process when a quote expired within the 48-hour reservation window, forcing them to fall back to Classic OneSite as a workaround.
- **Was the work item number or title stated?** No. Neither the work item number (2647905) nor the PME reference (PME-484256) is mentioned.

### Recommendations

- Open with the business context: *"This story addresses PME-484256, reported by Camden Development (Camden Thornton Park), a Top 50 PMC. In Classic OneSite, when a quote expires but the reservation is still valid within its 48-hour window, users can still select the unit via the reservation. This capability was missing in New OneSite, causing multiple escalations."*
- Explicitly state the work item number and title.
- Mention the escalation history (2 PMEs, 3 SF Cases) and the Expedite priority to convey urgency.
- Explain who benefits and what changes for the leasing user.

---

## 2. Logical Completion of Work

**Assessment: Partial**

### Evidence

The demo walks through two scenarios:

**Scenario 1 — Expired quote with active reservation (positive case):**

> *"0:13 We will login to the new one site experience, select a prospect and now get to the codes tab, select the status of expired and verify the action select unit is displayed."*
> *"0:27 So when we click on select unit."*
> *"0:42 Unit is selected successfully."*
> *"0:54 Now this is with an expired status and still the reserve is active."*

**Scenario 2 — Expired quote with inactive reservation (negative case):**

> *"1:00 The second scenario when it is expired and the reserve status is not active then we are not displaying the select unit action."*

### Analysis

- **Is the demo showing end-to-end behavior?** Partially. The presenter demonstrates:
  1. Navigating to a prospect's quotes tab
  2. Selecting an expired quote with an active reservation
  3. Verifying the "Select Unit" action is displayed
  4. Clicking "Select Unit" and confirming success
  5. A second scenario where the reservation is inactive and "Select Unit" is NOT shown

  This covers the core happy path and one negative scenario. However:
  - **No "before" state shown** — The demo does not first demonstrate the broken behavior (expired quote with active reservation NOT showing "Select Unit") to establish the baseline.
  - **No verification of unit assignment result** — After "Unit is selected successfully" at 0:42, there is no demonstration of the resulting state (e.g., unit now showing as assigned, prospect status updated).
  - **No demonstration from the "Reserved" dropdown** — The acceptance criteria specifically references the *"Quote tab dropdown labeled 'Reserved'"*. The presenter references *"select the status of expired"* rather than the "Reserved" dropdown, creating ambiguity about whether the exact AC navigation path was followed.

### Risk Flags

- Missing before/after comparison to demonstrate the fix.
- No verification of the downstream effect of selecting the unit.
- Navigation path described ("status of expired") does not clearly match the AC's "Reserved" dropdown.
- No demonstration of Classic OneSite parity behavior for comparison.
- "codes tab" mentioned at 0:13 — likely "quotes tab" but unclear narration.

---

## 3. Acceptance Criteria Coverage

**Assessment: Partially Covered**

**Actual Acceptance Criteria from AZD Work Item 2647905:**

> *"Need to select the Unit for the reserved quote from the Quote tab dropdown labeled 'Reserved.' — GAP between Classic OneSite and NEW OneSite"*

| # | Criterion | Demonstrated? | Evidence |
|---|-----------|---------------|----------|
| 1 | Select Unit action is available for a reserved quote when the quote has expired but reservation is still active | **Yes** | Presenter shows "Select Unit" action displayed for expired quote with active reservation (0:13–0:42). |
| 2 | Action accessible from the Quote tab dropdown labeled "Reserved" | **Unclear** | Presenter says *"select the status of expired"* (0:13) — does not explicitly reference or navigate via the "Reserved" dropdown as specified in the AC. |
| 3 | Unit can be successfully selected using this action | **Yes** | Presenter confirms *"Unit is selected successfully"* (0:42). |
| 4 | GAP between Classic and New OneSite is closed | **Not Verified** | No side-by-side or sequential comparison with Classic OneSite behavior is shown. |
| 5 | Select Unit NOT shown when reservation is inactive/expired | **Yes** | Presenter demonstrates: *"when it is expired and the reserve status is not active then we are not displaying the select unit action"* (1:00). |

### Gaps

- **Critical gap:** The AC specifically mentions the *"Quote tab dropdown labeled 'Reserved'"* — the presenter's navigation path ("select the status of expired") does not clearly demonstrate this specific dropdown interaction.
- **Minor gap:** No Classic OneSite comparison to verify the behavior gap is closed.
- **Minor gap:** No demonstration of the 48-hour reservation window behavior or boundary conditions.

### Recommendations

- Explicitly navigate to and narrate the "Reserved" dropdown on the Quotes tab as referenced in the AC.
- Consider briefly showing the Classic OneSite behavior as a reference point for the gap closure.
- Clarify the relationship between the expired quote status and the active reservation status during narration.

---

## 4. Validation & Edge Case Demonstration

**Assessment: Moderate**

### Analysis

- **Were negative scenarios shown?** Yes — one negative scenario was demonstrated. The presenter showed that when the quote is expired and the reservation is not active, the "Select Unit" action is not displayed (1:00). This is a meaningful validation.
- **Were validation messages demonstrated?** No. No success messages, confirmation dialogs, or error states were narrated.
- **Were system responses validated?** Partially. The presenter confirms "Unit is selected successfully" (0:42) but does not show what the system displays after selection (e.g., unit assignment confirmation, prospect status change).

### Positive Notes

- The inclusion of a second scenario (expired quote + inactive reservation = no Select Unit) is a notable strength. This demonstrates that the fix is conditional and does not blindly enable the action.

### Risk Flags

- No edge case testing demonstrated:
  - What happens if the reservation expires during the selection process?
  - What happens if another user reserves the same unit concurrently?
  - What happens if the reservation has exactly reached its expiration boundary?
- No error handling or validation message demonstration.
- No verification of downstream state after unit selection.
- Feature flag behavior not demonstrated (feature flag `reservation_commit_decoupled_from_quote` is documented in the work item but not mentioned in the demo).

---

## 5. Presentation Quality

**Assessment: Needs Improvement**

### Evaluate

- **Clarity of narration:** The narration is minimal and contains several unclear phrases:
  > *"0:05 So for the reserved one and the expired code we are not able to do the select unit process"* — "expired code" likely means "expired quote" but is ambiguous. "Select unit process" is vague for an audience unfamiliar with the feature.
  > *"0:13 ...select a prospect and now get to the codes tab"* — "codes tab" appears to be a mispronunciation or transcription error for "quotes tab."

- **Logical structure:** The demo follows a basic flow (brief intro → scenario 1 → scenario 2 → end) but lacks formal structure. No business context, no scope statement, no summary.

- **Presence of summary:** None. The demo ends abruptly:
  > *"1:13 Thank you."*
  No recap of what was demonstrated, no reinforcement of business value, no mention of deployment plan.

- **Background disturbance:** Cannot be assessed from transcript alone.

- **Confident vs uncertain phrasing:** No explicitly uncertain phrases ("I think," "probably," "should work") were detected. The presenter uses declarative statements, which is positive.

- **Demo duration:** Very short (~1:13). For an 8-point story with multiple escalations from a Top 50 PMC tagged as Expedite, the demonstration is extremely brief.

### Flagged Phrases

| Timestamp | Phrase | Concern |
|-----------|--------|---------|
| 0:05 | *"the expired code"* | "code" likely means "quote" — unclear terminology |
| 0:13 | *"get to the codes tab"* | "codes tab" likely means "quotes tab" — mispronunciation or transcription error |
| 0:01 | *"we delivered this item"* | Does not specify what "this item" is — no work item number, no title, no PME reference |

---

## 6. Structured Flow Evaluation

**Did the demo follow a logical flow?**

| # | Expected Component | Present? | Notes |
|---|-------------------|----------|-------|
| 1 | **Introduction** | Minimal | *"Hi everyone as a part of the Sprint we delivered this item"* — no work item number, no title, no presenter identification. |
| 2 | **Business Context** | Missing | No customer name, no PME reference, no escalation history, no explanation of the Classic vs New OneSite gap. |
| 3 | **Scope of Change** | Missing | No mention of what was changed (UI logic, API, business rules), what the feature flag covers, or what was NOT changed. |
| 4 | **Walkthrough of Feature** | Present | Two scenarios demonstrated: (1) expired quote + active reservation → Select Unit shown and successful; (2) expired quote + inactive reservation → Select Unit not shown. |
| 5 | **Validation / Edge Cases** | Partial | One negative scenario shown (inactive reservation hides the action). No edge cases, no error scenarios, no boundary testing. |
| 6 | **Summary & Impact** | Missing | Demo ends with "Thank you." — no recap, no impact statement, no deployment information. |

### Missing Components

- **Business Context** — Should describe the customer (Camden Development, Top 50 PMC), the PME (PME-484256), the gap between Classic and New OneSite, and the impact on leasing teams.
- **Scope of Change** — Should explain what code was modified, the feature flag (`reservation_commit_decoupled_from_quote`), and the conditions under which Select Unit is displayed.
- **Summary & Impact** — Should restate what was accomplished, which customers benefit, and the deployment/release plan (scheduled release date: 2026-05-06 per work item).

---

## 7. Overall Readiness Assessment

**Classification: Needs Minor Refinement**

### Justification

1. **Business context is absent** — The demo does not explain who is affected (Camden Development / Top 50 PMC), the escalation severity (2 PMEs, 3 SF Cases, Expedite tag), or why this gap between Classic and New OneSite matters to leasing teams. A viewer unfamiliar with the story would not understand the urgency or impact.

2. **Core functionality is demonstrated** — The two key scenarios (expired quote + active reservation → Select Unit available; expired quote + inactive reservation → Select Unit hidden) are both shown. This covers the primary AC requirement and includes a meaningful negative case.

3. **Acceptance criteria navigation path unclear** — The AC specifically references the *"Quote tab dropdown labeled 'Reserved'"* but the presenter's narration refers to *"select the status of expired"*. It is unclear whether the exact AC path was followed.

4. **Structured flow is incomplete** — The demo is missing business context, scope of change, and summary sections. Only the walkthrough and partial validation are present.

5. **Narration quality needs improvement** — Unclear terminology ("expired code," "codes tab") and no identification of the work item or PME reduce comprehension.

6. **Demo duration is very short (~1:13)** — For an 8-point Expedite story with multiple escalations from a Top 50 PMC, the demonstration is brief. However, the core requirement is a focused UI behavior change, so brevity alone is not disqualifying.

7. **Positive: Negative scenario included** — The inclusion of the second scenario (inactive reservation) is a strength that many demos lack. This demonstrates awareness of the conditional logic.

### Required Actions Before Sprint Recap

| Priority | Action |
|----------|--------|
| **High** | Add business context: customer name (Camden Development, Inc.), PME reference (PME-484256), escalation history (2 PME; 3 SF Cases), Top 50 PMC designation, and description of the Classic vs New OneSite gap |
| **High** | Explicitly navigate and narrate via the "Reserved" dropdown on the Quotes tab as specified in the acceptance criteria |
| **Medium** | Add a before/after or Classic vs New comparison to visually demonstrate the gap closure |
| **Medium** | Add a proper summary recapping what was demonstrated, who benefits, and deployment timeline |
| **Medium** | Clarify terminology: use "quotes tab" not "codes tab," "expired quote" not "expired code" |
| **Low** | Show or mention the feature flag state (`reservation_commit_decoupled_from_quote`) and its purpose |
| **Low** | Demonstrate the downstream result after unit selection (e.g., unit assignment confirmation, prospect status update) |

---

## Limitations

- This review is based solely on the transcript text. Visual UI correctness (actual dropdown navigation, button visibility, unit assignment result) cannot be confirmed.
- Audio quality and background noise cannot be assessed from transcript alone.
- "codes tab" appears in the transcript — it cannot be determined whether this is a mispronunciation by the presenter or a transcription error.
- BA/Product Owner sign-off is not mentioned in the transcript and cannot be confirmed.
- This review acts as a structured pre-check and does not replace BA validation.

---

*Generated by AI Demo Review Agent v1 per os-demo-review.agent.md and os-demo-readiness.guidelines.md*
*Work item data verified via Azure DevOps REST API on 2026-03-11*
