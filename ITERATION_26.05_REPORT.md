# Koopalings - Sprint 26.05 Iteration Report

**Team:** Koopalings  
**Project:** PropertyManagement  
**Iteration:** Sprint 26.05 (Feb 25 – Mar 10, 2026)  
**Report Date:** February 27, 2026  

---

## Team Composition

| Role | Members |
|------|---------|
| **Developers** | Paolo Natanawan, Edgar Fortin, JayAlvin Co, Aljun Christian Luna |
| **QA** | GraceMarie Garcia, Jomar Bingco, Suvarnamadhuri Batchu |
| **BA** | Leelakrishna Balu |

---

## Work Item Summary Table

### User Stories

| Work Item ID | Title | Type | State | Assigned To | STATUS | STATUSREASON |
|--------------|-------|------|-------|-------------|--------|--------------|
| 2647905 | PME-484256 - EXPEDITE \| Camden Thornton Park - Unable to Apply Reservation to Unit in New OneSite | User Story | Active | Paolo Natanawan | Active | Dev task (Development) is Active; tech spec approved by Maurice; development in progress. |
| 2496779 | PME-461308 - BARRIER TO ADOPT \| NOS: Does Not Have Cancel Reservation when a quote is expired | User Story | Active | Aljun Christian Luna | Active | Dev task (Unit Testing) is Active; Development and VM Setup tasks are Closed; Code Review and Demo tasks pending. |
| 1708591 | PME-352510: [ENHANCEMENT] ALL RESIDENTS Modify Reports to honor UK Date Formatting - PME DRIVER | User Story | Active | Edgar Fortin | Active | Dev task (Development) is Active; tech spec approved by Maurice; pending Leelakrishna Balu's sign-off for RTW confirmation. |
| 2253142 | (Data Clean-up & Replication) PME-429593 - Unable to add HH lease signers error | User Story | Active | Edgar Fortin | Active conflict | DEV task 2750235 is Closed but work item remains in Active state with no active DEV tasks. QA replication task (2750125) is New. Pending QA verification and Affordable team coordination for data cleanup. |

### Bugs

| Work Item ID | Title | Type | State | Assigned To | STATUS | STATUSREASON |
|--------------|-------|------|-------|-------------|--------|--------------|
| 2564870 | PME-472177 - BARRIER TO ADOPT \| New Experience Quote Settings - Not Honoring Grace Period Setting | Bug | Test in Progress | Jomar Bingco | Testing in progress conflict | Assigned to Sprint 26.04 but has tasks in 26.05. QA Functional Testing task (2700231) is Active. However, build has not yet been applied to the test environment; no dev unit testing evidence found in comments/attachments; API demo not conducted (Edgar on sick leave). Additional UI changes may be needed beyond original scope. |

### Spikes

| Work Item ID | Title | Type | State | Assigned To | STATUS | STATUSREASON |
|--------------|-------|------|-------|-------------|--------|--------------|
| 2749658 | [UI] Tech analysis for Min rental history/min emp history enforcement and merge fields repeater | Spike | Active | JayAlvin Co | Active | Task (2750298) is Active; analysis work in progress. |
| 2750240 | Additional Effort - User Story 2170733: PME-414179 - Screening History not Available in New OneSite | Spike | Ready To Work | Suvarnamadhuri Batchu | Ready to work | No active tasks. QA additional effort spike; task (2750287) is New. |
| 2750361 | Analysis/Make RTW - Feature 2005358: PME-400546 - Add "Make Prospects Inactive" Option to Operations Menu | Spike | Ready To Work | Aljun Christian Luna | Ready to work | No active tasks; task (2750378) is New. Ready for dev pickup. |
| 2750405 | UI Analysis/Make RTW - Feature 2404320: PME-451110 - Need forward looking date filter in "Task Summary" widget | Spike | Ready To Work | JayAlvin Co | Ready to work | No active tasks; task (2750520) is New. Ready for dev pickup. |
| 2750436 | API Analysis/Make RTW - Feature 2404320: PME-451110 - Need forward looking date filter in "Task Summary" widget | Spike | Ready To Work | Paolo Natanawan | Ready to work | No active tasks; task (2751674) is New. Ready for dev pickup. |

### Non Functional Stories

| Work Item ID | Title | Type | State | Assigned To | STATUS | STATUSREASON |
|--------------|-------|------|-------|-------------|--------|--------------|
| 2749657 | Demo Recording | Non Functional Story | Closed | Jomar Bingco | Closed | All tasks (2749659, 2749958) are Closed. Completed successfully. |
| 2749866 | March release regression testing | Non Functional Story | Active | Jomar Bingco | Active | Task 2749963 (JB) is Closed; task 2749975 (SB) is Active. Regression testing in progress. |
| 2749870 | CRM Support - March release regression testing | Non Functional Story | Active | Jomar Bingco | Active conflict | No active tasks. Task 2750172 (JB) is New and task 2753891 (SB - KT on Required fields) is Closed. Work item state does not match task activity. |
| 2749894 | March release pre-prod testing | Non Functional Story | Ready To Work | Jomar Bingco | Ready to work | All tasks (2749966, 2750059) are New. Ready for QA pickup. |
| 2754073 | Smoke & Sanity Automation Test Suite Execution and Monitoring | Non Functional Story | Ready To Work | Jomar Bingco | Ready to work conflict | Task 2754081 (JB) is Active, but work item state is Ready To Work. Work item should be in Active state. |
| 2750512 | Online Leasing KT - Quotes Reservation | Non Functional Story | Active | Suvarnamadhuri Batchu | Active conflict | No active tasks. Task 2750525 (SB) is New. Work item state does not match task activity. |

### Admin

| Work Item ID | Title | Type | State | Assigned To | STATUS | STATUSREASON |
|--------------|-------|------|-------|-------------|--------|--------------|
| 2750268 | Q2 Feature Readiness & Estimation Refinement (Sprint 26.05) | Admin | Active | Paolo Natanawan | Active | Task 2749946 (ACL - Contingency) is Active; task 2749947 (ACL - Sprint Planning) is Closed. Contingency tasks for all team members in New state. |
| 2751814 | Standardize Backlog Grooming Criteria & Level-Heading Alignment (Cross-Team) | Admin | Ready To Work | Paolo Natanawan | Ready to work | No tasks created. Feedback item. |
| 2751828 | Review or generate prompt for AI to review demo | Admin | Ready To Work | Paolo Natanawan | Ready to work | No tasks created. Feedback item. |
| 2751832 | Establish Centralized AI Prompt Repository & Usage Standards (Koopalings) | Admin | Active | Paolo Natanawan | Active | No tasks created. Active administrative initiative. |
| 2751855 | Ensure item handover or demo have PR approvals prior to setting item to resolve and demo | Admin | Ready To Work | Paolo Natanawan | Ready to work | No tasks created. Feedback/process improvement item. |

---

## State Conflict Summary

| Work Item ID | Title | Current State | Expected State | Issue |
|--------------|-------|---------------|----------------|-------|
| 2253142 | PME-429593 - Unable to add HH lease signers | Active | Resolved or Active (with active tasks) | DEV task is Closed with no remaining active DEV tasks. If dev work is complete, should transition to Resolved and be assigned to QA. |
| 2564870 | PME-472177 - Grace Period Setting | Test in Progress | Test in Progress (with conditions) | Build not applied to test environment; no dev unit testing evidence; API demo not conducted. State prerequisites not fully met. |
| 2749870 | CRM Support - March release regression testing | Active | Ready To Work or Active (with active tasks) | No active tasks. One task Closed, one task New. |
| 2754073 | Smoke & Sanity Automation Test Suite | Ready To Work | Active | Task 2754081 is in Active state but parent work item remains Ready To Work. |
| 2750512 | Online Leasing KT - Quotes Reservation | Active | Ready To Work | Task 2750525 is New (not Active). No active tasks to justify Active state. |

---

## Achievements

1. **Demo Recording (2749657)** - Completed and Closed. Demo recordings for PME-463082 (Create a Quote error fix) and Screening History delivered by Jomar Bingco and Suvarnamadhuri Batchu.

2. **US 2647905 Tech Spec Approved** - Technical specification for the EXPEDITE item (Camden Thornton Park - Apply Reservation) completed by Paolo Natanawan and approved by Maurice Gollayan. Architecture approach includes LaunchDarkly governance and QuoteSettings table for configurability.

3. **US 1708591 Tech Spec Approved** - Regional Date Formatting specification completed and approved by Maurice Gollayan ("The spec looked like functional code already"). Awaiting BA sign-off.

4. **US 2496779 Development Complete** - Cancel Reservation development and VM setup tasks completed by Aljun Christian Luna. Currently in unit testing phase.

5. **Bug 2564870 QA In Progress** - Grace Period bug (from Sprint 26.04) has active QA functional testing underway by Jomar Bingco, with API demo completed.

6. **March Release Regression Testing Started** - Regression testing (2749866) in progress with Suvarnamadhuri Batchu actively testing and Jomar Bingco's portion completed.

---

## Blockers & Areas Requiring Attention

### Critical Blockers

1. **Bug 2564870 - Build Not Applied**: The build for the Grace Period fix has not been applied to the test environment, hindering QA functional testing. Additionally, the API demo was not conducted due to Edgar Fortin's sick leave. Additional UI changes may be needed beyond the original scope, putting this item at risk.

2. **US 2253142 - Affordable Data Cleanup Dependency**: Analysis by Edgar Fortin reveals the majority of problematic data (4,785 No-HOH + 1,917 Duplicate-HOH leases) exists in Affordable housing sites with complex compliance table dependencies. Recommendation is to defer Affordable cleanup to the Affordable Housing team. Conventional-only cleanup is feasible but requires cross-team coordination.

### Process Concerns

3. **US 1708591 - Pending Sign-off**: Tech spec approved by Maurice but awaiting Leelakrishna Balu's (BA) and GraceMarie Garcia's (QA) sign-off to formally transition to Ready To Work. Paolo has followed up twice (Feb 20, Feb 25) with no response noted.

4. **US 2496779 - CCR/Justification Needed**: Paolo requested CCR and Justification for WMU package inclusion. This administrative requirement must be addressed for the item to proceed through the release pipeline.

5. **5 State Conflicts Identified**: Five work items have state mismatches between the work item state and their child task states (see State Conflict Summary above). Recommend reviewing and correcting states to maintain accurate sprint tracking.

### Team Capacity Notes

6. **GraceMarie Garcia** (QA) has no work items or tasks assigned in Sprint 26.05. Verify if capacity is intentionally allocated elsewhere or if assignments are needed.

7. **Leelakrishna Balu** (BA) has no work items or tasks assigned in Sprint 26.05 but is referenced as a dependency for sign-offs on US 1708591 and US 2496779.

---

## Overall Progress

| Metric | Count |
|--------|-------|
| **Total Work Items** | 21 |
| **User Stories** | 4 |
| **Bugs** | 1 |
| **Spikes** | 5 |
| **Non Functional Stories** | 6 |
| **Admin** | 5 |

| State | Count | Items |
|-------|-------|-------|
| **Closed** | 1 | 2749657 |
| **Active** | 11 | 2647905, 2496779, 1708591, 2253142, 2749658, 2749866, 2749870, 2750512, 2750268, 2751832, 2564870 (Test in Progress) |
| **Ready To Work** | 9 | 2750240, 2750361, 2750405, 2750436, 2749894, 2754073, 2751814, 2751828, 2751855 |

| Validation | Count |
|------------|-------|
| **Valid States** | 15 |
| **State Conflicts** | 5 |
| **Admin Conflicts** | 0 |

Sprint 26.05 is in its early days (Day 3 of 14). The team has active development work on 3 user stories (2647905, 2496779, 1708591), 1 active spike analysis (2749658), and QA regression testing in progress. Key risks include the Bug 2564870 build deployment blocker and the pending BA/QA sign-offs for US 1708591. The 5 state conflicts should be addressed to maintain accurate sprint tracking. Cross-team coordination is needed for US 2253142 (Affordable data cleanup).
