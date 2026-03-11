# Demo Review Report – Work Item 1708591

**Work Item:** 1708591
**Title:** PME-352510: [ENHANCEMENT] ALL RESIDENTS Modify Reports to honor UK Date Formatting – PME DRIVER
**Type:** User Story
**State:** Closed
**Assigned To:** Jomar Bingco
**Sprint:** PI 26.Q1 (Sprint 26.05)
**Story Points:** 8
**Area:** PropertyManagement\OneSite Platform\Koopalings
**Tags:** External PME; Multiple Escalations; OUTPOST MANAGEMENT LIMITED; Reporting Audit
**Created:** 2023-11-14
**Review Date:** 2026-03-11
**Reviewed By:** AI Demo Review Agent v1
**Guidelines Reference:** os-demo-readiness.guidelines.md (v2, Mar 02, 2026)
**Data Source:** Azure DevOps REST API (verified work item fields)

---

## 1. Business Context & Value

**Assessment: Partial**

### Evidence

The presenter briefly states the purpose at the start:

> *"0:10 The fix that we made here is to ensure that new asset user will belongs to a UK TNC or site."*
> *"0:16 The dates in the old resident Excel report should appear in UA format which is date, month and year."*
> *"0:23 And right now they show up in the US format."*
> *"0:25 So this is we need to fix that."*

### Analysis

- **Was the business problem clearly stated?** Partially. The presenter mentions dates show up in US format and need to show in UK format, but never names the affected customer (Outpost PMC), never references the PME (PME-352510), and does not mention the 2+ year escalation history.
- **Was customer impact explained?** No. There is no mention of customer confusion, manual interpretation burden, or escalation frequency. The "why" is limited to "we need to fix that."
- **Was "why" clearly answered?** Minimally. The value proposition — reducing customer confusion and addressing long-standing PME escalations — is absent.

### Recommendations

- Open with the business context: *"This enhancement addresses PME-352510, reported by Outpost PMC (UK-based). UK sites have been seeing US-formatted dates in the All Residents Excel report since November 2023, causing confusion and multiple escalations."*
- Explicitly state customer impact and who benefits from the change.
- Reference the work item number and story title clearly at the start (the title was partially stated but mumbled).

---

## 2. Logical Completion of Work

**Assessment: Partial**

### Evidence

The demo walks through:

1. Navigating to Unified Reports → Reports tab → All Residents Excel report (0:41–0:59)
2. Selecting a property (Cooper Station) with the flag enabled (1:02)
3. Generating the report (1:14–1:16)
4. Viewing the completed report (1:31–1:34)
5. Observing the date columns in the report:

> *"1:40 View the completed reports and as you can see in the Move in Expires notice for Date and Move Out column, the date format is in that date, month and year."*

### Analysis

- **Is the demo showing end-to-end behavior?** Partially. The basic happy path (generate report → view dates in UK format) is shown. However, the following are missing:
  - **No "before" comparison** — the presenter did not first show the existing US-formatted output to establish the baseline, making it impossible to confirm the change purely from the transcript.
  - **All 4 required date columns mentioned** — the work item description specifies 4 date columns: Move-In, Expires, Notice for Date, and Move-Out. The presenter names "Move in, Expires, notice for Date and Move Out" — which covers all 4. *(Note: The AI-generated technical specification incorrectly added Birth Date and Property Date as columns 5 and 6 — these are NOT in the actual AZD work item and were AI slop.)*
  - **No demonstration with a US site** — there is no verification that US sites remain unaffected (regression check).
  - **No feature flag toggle demonstration** — the tech spec emphasizes LaunchDarkly feature flag integration, but the demo does not show the flag state or demonstrate toggling behavior.

### Risk Flags

- Missing before/after comparison weakens the demonstration of the fix.
- No regression scenario shown (US site unchanged).
- No feature flag behavior demonstrated.

---

## 3. Acceptance Criteria Coverage

**Assessment: Partially Covered**

**Actual Acceptance Criteria from AZD Work Item 1708591:**

> *"UK: All Residents-Excel Report - Change dates to UK date format. This report currently only shows U.S. date format."*

**Work Item Description specifies 4 date columns:**
- Move-In – Resident's move-in date
- Expires – Lease end date
- Notice for Date – Scheduled move-out/transfer date (if notice is given)
- Move-Out – Actual move-out date for former residents

| # | Criterion | Demonstrated? | Evidence |
|---|-----------|---------------|----------|
| 1 | All Residents Excel report displays dates in UK format (DD/MM/YYYY) for UK sites | **Yes** | Presenter shows report with dates in "date, month and year" format for Cooper Station (UK-enabled property). |
| 2 | All 4 required date columns correctly transformed (Move-In, Expires, Notice for Date, Move-Out) | **Yes** | Presenter explicitly names all 4 columns at 1:40. |
| 3 | No impact on US or other regional sites | **Not Covered** | No US site comparison shown. |
| 4 | Feature flag prerequisite mentioned | **Partial** | Presenter mentions flag must be "enabled" at 0:29 but does not demonstrate toggling or disabled state. |

### Gaps

- **Notable gap:** No US site comparison to verify no regression.
- **Minor gap:** Feature flag toggle not demonstrated (only mentioned as prerequisite).
- **Minor gap:** No edge-case or negative-scenario testing shown.

### Recommendations

- Add a before/after comparison: Show the report for a US site first (US dates), then for the UK site (UK dates).
- Demonstrate the feature flag: Show the flag enabled in LaunchDarkly, then optionally show the flag disabled to confirm fallback behavior.

---

## 4. Validation & Edge Case Demonstration

**Assessment: Weak**

### Analysis

- **Were negative scenarios shown?** No. No demonstration of malformed dates, empty date fields, or error handling.
- **Were validation messages demonstrated?** No. No error states or graceful fallback behavior was shown.
- **Were system responses validated?** Partially. The presenter visually confirms the report output shows dates in the expected format, but there is no systematic validation.

### Risk Flags

- No edge case testing (empty dates, malformed dates, null values).
- No regression testing against US sites.
- No verification of large dataset performance.
- No feature flag on/off validation.
- No demonstration of what happens if regional settings retrieval fails.

---

## 5. Presentation Quality

**Assessment: Needs Improvement**

### Evaluate

- **Clarity of narration:** The narration is understandable but contains several grammatical and phrasing issues that reduce clarity:
  > *"0:10 The fix that we made here is to ensure that new asset user will belongs to a UK TNC or site."* — Phrasing is unclear; "new asset user" likely means "when a site user belongs to a UK TNC" but this is ambiguous.
  > *"0:16 The dates in the old resident Excel report should appear in UA format"* — "UA format" appears to be a mispronunciation or transcription error for "UK format."
  > *"1:02 Then for this I'm going to use Cooper station that has a debug enabled and you read it format enabled."* — "debug enabled and you read it format enabled" is unclear; likely means "that has the UK date format flag enabled."

- **Logical structure:** The demo follows a basic flow (introduction → setup → walkthrough → result), but lacks a formal structure with business context, scope statement, and summary.

- **Presence of summary:** Minimal. The demo ends abruptly:
  > *"2:05 And that's all for this demo. 2:07 Thank you."*
  No recapping of what was demonstrated, no reinforcement of business value.

- **Background disturbance:** Cannot be assessed from transcript alone.

- **Confident vs uncertain phrasing:** No explicitly uncertain phrases like "I think," "probably," or "should work" were detected. However, the narration lacks confident, definitive statements about the change's completeness and correctness.

### Flagged Phrases

| Timestamp | Phrase | Concern |
|-----------|--------|---------|
| 0:16 | *"should appear in UA format"* | Uses "should" — ambiguous whether describing the requirement or the actual behavior |
| 0:29 | *"make sure that the NT flag and UK date format is enabled"* | "NT flag" is unclear — likely "feature flag" but not explicitly stated |

---

## 6. Structured Flow Evaluation

**Did the demo follow a logical flow?**

| # | Expected Component | Present? | Notes |
|---|-------------------|----------|-------|
| 1 | **Introduction** | Partial | Work item number stated, but title is paraphrased unclearly. No presenter self-identification. |
| 2 | **Business Context** | Weak | Problem stated minimally ("dates show in US format, need UK"). No customer impact or escalation history. |
| 3 | **Scope of Change** | Missing | No explicit scope statement. No mention of which service layer was modified, what files changed, or what the feature flag covers. |
| 4 | **Walkthrough of Feature** | Present | Basic happy path demonstrated: navigate to report → generate → view dates. |
| 5 | **Validation / Edge Cases** | Missing | No validation scenarios, no negative tests, no before/after comparison. |
| 6 | **Summary & Impact** | Missing | Demo ends with "And that's all for this demo. Thank you." — no recap, no impact reinforcement. |

### Missing Components

- **Scope of Change** — Should describe what was changed (service layer transformation), what was NOT changed (stored procedure), and what protections are in place (feature flag).
- **Validation / Edge Cases** — At minimum, a US site comparison and feature flag toggle should be demonstrated.
- **Summary & Impact** — Should restate what was accomplished, which customers it benefits, and deployment plan.

---

## 7. Overall Readiness Assessment

**Classification: Needs Minor Refinement**

### Justification

1. **Business context is insufficient** — The demo does not explain customer impact (Outpost PMC), escalation history (2+ years, tagged "Multiple Escalations"), or value proposition. A viewer unfamiliar with the story would not understand why this matters.

2. **Core acceptance criteria are covered** — All 4 required date columns are mentioned and shown in UK format. The actual AC from AZD (*"Change dates to UK date format"*) is functionally demonstrated. However, no regression testing (US site) is shown.

3. **Structured flow is incomplete** — The demo is missing scope of change, validation, and summary sections. It follows only a minimal walkthrough.

4. **No before/after comparison** — The core value of the fix (dates changing from US to UK format) is not visually contrasted. There is no baseline shown.

5. **Narration quality needs improvement** — Multiple unclear phrases ("new asset user," "UA format," "debug enabled and you read it format enabled") reduce comprehension.

6. **Demo duration is very short (~2 minutes)** — For an 8-point story with feature flag integration and multi-region impact, the demonstration is brief but covers the core requirement.

### Required Actions Before Sprint Recap

| Priority | Action |
|----------|--------|
| **High** | Add business context: customer name (Outpost PMC), PME reference (PME-352510), escalation history, customer impact |
| **High** | Show a before/after comparison (US site → UK site) to clearly demonstrate the change |
| **Medium** | Add a proper summary recapping what was demonstrated and deployment plan |
| **Medium** | Clarify narration — avoid ambiguous terms ("NT flag," "debug enabled") |
| **Low** | Show or mention feature flag state and its purpose |
| **Low** | Consider demonstrating feature flag toggle (enabled vs disabled behavior) |

---

## AI Slop Correction Notice

During this review, the AI-generated technical specification ([1708591-TECHNICAL-SPECIFICATION-SERVICE-LEVEL.md](1708591-TECHNICAL-SPECIFICATION-SERVICE-LEVEL.md)) was found to contain inaccuracies compared to the actual AZD work item:

| Field | Tech Spec (AI-Generated) | Actual AZD Work Item |
|-------|--------------------------|---------------------|
| Date columns | 6 columns (Move-In, Expires, Notice for Date, Move-Out, **Birth Date**, **Property Date**) | 4 columns (Move-In, Expires, Notice for Date, Move-Out) |
| Acceptance Criteria | 8 detailed success criteria | Single line: *"UK: All Residents-Excel Report - Change dates to UK date format. This report currently only shows U.S. date format."* |

**Birth Date and Property Date were fabricated by the AI** and are not part of the actual work item scope. This review has been corrected to reflect the actual AZD data.

---

## Limitations

- This review is based solely on the transcript text. Visual UI correctness (actual date format rendering in the Excel file) cannot be confirmed.
- Audio quality and background noise cannot be assessed from transcript alone.
- BA/Product Owner sign-off is not mentioned in the transcript and cannot be confirmed.
- This review acts as a structured pre-check and does not replace BA validation.

---

*Generated by AI Demo Review Agent v1 per os-demo-review.agent.md and os-demo-readiness.guidelines.md*
*Work item data verified via Azure DevOps REST API on 2026-03-11*
