# Implementation Plan — Feature 2476930
# PME-451065: Minimum Rental History / Minimum Employment History Enforcement

- **Feature:** [2476930](https://tfs.realpage.com/tfs/Realpage/PropertyManagement/_workitems/edit/2476930)
- **Version:** 1.0 (Draft)
- **Date:** 2026-03-13
- **Derived from:** [Full Technical Specification](AZD_WI_2476930_FULL_TECHNICAL_SPECIFICATION.md)
- **Iteration:** PI 26.Q2
- **Estimated Effort:** 211.25 dev hours + 126.5 test hours (API + UI combined)

---

## Table of Contents

1. [Implementation Strategy](#1-implementation-strategy)
2. [Prerequisite Checks](#2-prerequisite-checks)
3. [Dependency Graph](#3-dependency-graph)
4. [Phase Breakdown](#4-phase-breakdown)
5. [Sprint Allocation](#5-sprint-allocation)
6. [Integration Checkpoints](#6-integration-checkpoints)
7. [Risk Mitigations](#7-risk-mitigations)
8. [Environment & Testing Strategy](#8-environment--testing-strategy)
9. [Code Review Strategy](#9-code-review-strategy)
10. [Rollout Plan](#10-rollout-plan)

---

## 1. Implementation Strategy

### Core Principle: Bottom-Up, API-First

Build in dependency order: **shared rules → API load → API validation → API save → UI data layer → UI step rendering → integration**.

The API layer must be complete and testable before UI work begins on each domain. This prevents the UI team from building against speculative contracts.

### Parallelization Opportunities

Two independent domain pairs exist:
- **Rental History** (residence) — can proceed independently from Employment History
- **Employment History** — mirrors rental history but in separate files

Within each domain, the Prospects and Leased Accounts changes share models but touch different controllers/managers, so they can be developed by different engineers once the shared validation rules are defined.

### Feature Flag Discipline

All new behavior is gated behind `accounts-employment-and-rental-length`. Every code change must preserve the existing behavior when the flag is OFF. This means:
- No method signature changes that break existing callers
- New methods/overloads alongside existing ones
- UI branches: `if (flagOn) { newPath } else { legacyPath }`

---

## 2. Prerequisite Checks

Before implementation begins, these must be confirmed or resolved:

| # | Prerequisite | Owner | Status |
|---|---|---|---|
| P1 | Bug 2621966 (PME-481270) — Required Fields Prospect bug | TBD | Must be completed before Waitlist validation work |
| P2 | Mandatory Fields LD flag state — confirm test environment strategy | QA Lead | Open — flag is disabled for all customers |
| P3 | Open Questions 1–3 from tech spec (duration aggregation rules) | Product | Must be answered before Phase 1 coding |
| P4 | Feature branch created from `Development` | Dev Lead | Not started |
| P5 | AOLR alignment confirmed (Feature 2741850 separate scope) | Product | Verbal — needs written confirmation |

**Hard blockers:** P3 must be resolved before any validation code is written. P1 must be resolved before Phase 3 Waitlist work.

---

## 3. Dependency Graph

```
P3 (Duration rules confirmed)
│
├──► Phase 1: Foundation
│    ├── 1A: Duration aggregation rules (shared)
│    └── 1B: Feature flag wiring (UI)
│         │
│    ┌────┴────────────────────────────────────┐
│    │                                         │
├──► Phase 2A: API — Rental History    Phase 2B: API — Employment History
│    ├── Prospects Load (§9.1)         ├── Prospects Load (§9.2)
│    ├── Prospects Validation (§9.3)   ├── Prospects Validation (§9.4)
│    ├── Prospects Save XML (§9.5)     ├── Prospects Save XML (§9.6)
│    ├── Leased Accts Validation       ├── Leased Accts Validation
│    │   (§10.4)                       │   (§10.5)
│    └── Leased Accts PUT (§10.2)      └── Leased Accts PUT (§10.3)
│         │                                    │
│    ┌────┴────────────────────────────────────┘
│    │
├──► Phase 3: Cross-Domain API
│    ├── Prospects Required Fields (§9.7)
│    ├── Waitlist Required Fields (§11.1) ◄── requires Bug 2621966
│    └── Select Unit multi-node XML (§12.1)
│         │
│    ┌────┴────────────────────────────────────┐
│    │                                         │
├──► Phase 4A: UI — Residence         Phase 4B: UI — Employment
│    ├── Data/Load/Save layer (§13.2)  ├── Data/Load/Save layer (§13.3)
│    ├── Step UI repeater (§13.4)      ├── Step UI repeater (§13.5)
│    └── Prospects UI (§14.1)          └── Prospects UI (§14.1)
│         │                                    │
│    ┌────┴────────────────────────────────────┘
│    │
├──► Phase 5: Shared UI
│    ├── Save orchestration (§13.6)
│    ├── Service endpoint contracts (§13.7)
│    ├── Wizard model refactor (§13.8)
│    └── Required Fields validation UI (§13.9)
│         │
├──► Phase 6: MergeFields (§15) — independent, can start after Phase 2
│         │
└──► Phase 7: Integration Testing & Rollout
```

### Critical Path

The longest chain is: **P3 → Phase 1A → Phase 2A → Phase 3 → Phase 4A → Phase 5 → Phase 7**

Phase 2A and 2B can run in parallel. Phase 4A and 4B can run in parallel. Phase 6 is independent after Phase 2. This creates opportunities to shorten the calendar timeline with two developers.

---

## 4. Phase Breakdown

### Phase 1: Foundation (Pre-Sprint or Sprint 1, Day 1-2)

**Goal:** Establish shared rules and feature flag wiring before any domain-specific work.

| Task | Description | Files | Est. Hours |
|---|---|---|---:|
| 1A | Define and document duration aggregation rules — exact calculation logic, edge cases | Spec update + unit tests | 6.5 |
| 1B | Add `accounts-employment-and-rental-length` activity key | `activity-name.js` | 3.25 |
| 1C | Create feature branch, set up LD flag in dev environment | DevOps | 2 |

**Exit criteria:** Aggregation rules codified in unit tests. Feature flag toggleable in SAT environment.

---

### Phase 2A: API — Rental History (Sprint 1)

**Goal:** All rental history backend changes complete and API-testable.

| Task | Description | Files | Est. Hours | Depends On |
|---|---|---|---:|---|
| 2A-1 | Prospects Load — `SelectSingleNode` → `SelectNodes` for Previous rental | ProspectListManager.cs (L2698, L9444) | 6.5 | 1A |
| 2A-2 | Prospects Save Validation — aggregate diffDays, indexed errors | ProspectListManager.cs (L10932) | 13 | 1A |
| 2A-3 | Prospects Save XML — move node removal, handle list | ProspectListManager.cs (L3443) | 6.5 | 2A-1 |
| 2A-4 | Leased Accts Validation — refactor `IsRentalHistoryDurationSufficient` to list | PreviousResidenceValidator.cs, Field.cs | 13 | 1A |
| 2A-5 | Leased Accts Save — new PUT endpoint (array + deleteIds, hard delete) | PreviousResidenceController.cs, PrevResidenceManager.cs, PrevResidenceDAO.cs | 13 | — |

**Parallelization:** 2A-1 and 2A-4 can start simultaneously (different files). 2A-5 is independent.

**Exit criteria:** All rental history API changes pass unit tests. GET returns N records. PUT accepts array. Validation returns indexed errors and aggregate insufficiency key.

---

### Phase 2B: API — Employment History (Sprint 1, parallel with 2A)

**Goal:** Mirror all rental history changes for employment domain.

| Task | Description | Files | Est. Hours | Depends On |
|---|---|---|---:|---|
| 2B-1 | Prospects Load — `SelectSingleNode` → `SelectNodes` for Previous employment | ProspectListManager.cs (L3001) | 6.5 | 1A |
| 2B-2 | Prospects Save Validation — aggregate diffDays, indexed errors | ProspectListManager.cs (L11248) | 13 | 1A |
| 2B-3 | Prospects Save XML — mirror rental pattern | ProspectListManager.cs (L3965) | 6.5 | 2B-1 |
| 2B-4 | Leased Accts Validation — refactor employment duration check | EmploymentHistoryValidator.cs, Field.cs | 13 | 1A |
| 2B-5 | Leased Accts Save — new PUT endpoint (array + deleteIds, soft delete) | EmploymentController.cs + new manager/DAO | 13 | — |

**Exit criteria:** Same as Phase 2A but for employment domain.

---

### Phase 3: Cross-Domain API (Sprint 2, first half)

**Goal:** Required fields and Select Unit flows handle N records.

| Task | Description | Files | Est. Hours | Depends On |
|---|---|---|---:|---|
| 3-1 | Prospects Required Fields — multi-record completion check | ProspectListManager.cs (L7308+) | 6.5 | 2A-1, 2B-1 |
| 3-2 | Waitlist Required Fields — list-based `FillRentalHistory` / `FillEmploymentHistory` | PeopleManager.cs (L1918, L1977) | 13 | Bug 2621966, 1A |
| 3-3 | Select Unit — ContactsManager multi-node XML | ContactsManager.cs (L475, L542, L644, L709) | 13 | 1A |

**Parallelization:** 3-1, 3-2, and 3-3 are in different files and can be developed in parallel. 3-2 is blocked if Bug 2621966 is not yet resolved; if so, defer to Sprint 3.

**Exit criteria:** Household member completion status correctly reflects N records. ContactsManager parses all XML nodes. WaitlistManager (already looping) handles them automatically.

---

### Phase 4A: UI — Residence (Sprint 2, second half → Sprint 3)

**Goal:** Residence step supports list rendering, add/delete, and list-based save.

| Task | Description | Files | Est. Hours | Depends On |
|---|---|---|---:|---|
| 4A-1 | Data layer refactor — list-based load/model mapping | previous-residence-svc.js, hh-member-previous-residence-model.js, household-member-manage-ctrl.js | 13 | 2A-5 (PUT endpoint available) |
| 4A-2 | Step UI refactor — repeater template, add/delete buttons, form-config | index.html, hh-member-previous-residence.js, hh-member-previous-residence-form-config.js | 16 | 4A-1 |
| 4A-3 | Prospects UI — refactor response mapping + buildRentalAdrress | prospect-rental-history.js, prospect-household-list-comp.js | 6 | 2A-1, 4A-1 |

**Exit criteria:** User can load, add, edit, delete multiple previous residence entries. Save sends single PUT. Legacy path works when flag OFF.

---

### Phase 4B: UI — Employment (Sprint 2–3, parallel with 4A)

**Goal:** Employment step supports list rendering and list-based save.

| Task | Description | Files | Est. Hours | Depends On |
|---|---|---|---:|---|
| 4B-1 | Data layer refactor — list-based employment model | hh-member-employment-model.js, household-member-manage-ctrl.js | 6.5 | 2B-5 (PUT endpoint available) |
| 4B-2 | Step UI refactor — repeater template, add/delete | index.html, hh-member-employment-ctrl.js, hh-member-employment-form-config.js | 16 | 4B-1 |

**Exit criteria:** Same as 4A but for employment domain.

---

### Phase 5: Shared UI Integration (Sprint 3)

**Goal:** Tie together save orchestration, service contracts, wizard model, and validation UI.

| Task | Description | Files | Est. Hours | Depends On |
|---|---|---|---:|---|
| 5-1 | Save orchestration — single PUT per domain | household-member-manage-ctrl.js | 8 | 4A-1, 4B-1 |
| 5-2 | Service contracts — PUT helpers + legacy POST preservation | household-member-manage-svc.js | 4 | 5-1 |
| 5-3 | Wizard model refactor — collection state + compatibility adapters | household-member-wizard-model.js | 6 | 4A-1, 4B-1 |
| 5-4 | Validation UI — indexed keys, aggregate keys, row-level error mapping | live-form-validator-svc.js, form-validator-dir.js, hh-member-previous-residence-model.js, hh-member-employment-model.js | 8 | 2A-4, 2B-4, 5-1 |
| 5-5 | Localization + regression hardening | Resource files, all modified components | 4 | 5-1 through 5-4 |

**Exit criteria:** Full add/edit/delete/save cycle works E2E for both domains. Validation errors map to correct repeater rows. Flag OFF preserves all legacy behavior.

---

### Phase 6: MergeFields (Sprint 2–3, independent track)

**Goal:** Repeater mergefields for multi-history records.

| Task | Description | Files | Est. Hours | Depends On |
|---|---|---|---:|---|
| 6-1 | Determine which mergefields are needed (product decision) | — | 2 | Product input |
| 6-2 | Create SQL insert scripts per mergefield | DATABASESETUP/.../ServerManager/ | 3.25/ea | 6-1 |
| 6-3 | Validate mergefield rendering in letter generation | E2E test | 3 | 6-2 |

**Note:** This is on a separate track. Can be done by a third engineer or deferred to a later sprint without blocking the main feature.

---

### Phase 7: Integration Testing & Rollout (Sprint 3–4)

**Goal:** End-to-end verification, regression, and controlled rollout.

| Task | Description | Est. Hours | Depends On |
|---|---|---|---:|---|
| 7-1 | E2E test plan creation (LD-gating aware) | 8 | All API + UI phases |
| 7-2 | Execute E2E tests — Prospects flow | 13 | 7-1 |
| 7-3 | Execute E2E tests — Leased Accounts flow | 13 | 7-1 |
| 7-4 | Execute E2E tests — Waitlist/Select Unit flow | 6.5 | 7-1, 3-2 |
| 7-5 | Flag-OFF regression (all flows unchanged) | 6.5 | 7-1 |
| 7-6 | Performance check (N records load/save timing) | 4 | 7-2 |
| 7-7 | Controlled rollout (flag ON for staging → production) | 2 | 7-2 through 7-6 |

---

## 5. Sprint Allocation

Assuming **2-week sprints**, **6.5 dev-hours/day**, and **2 API + 1 UI developer** available:

### Sprint 1 (Weeks 1–2): API Foundation + Domain APIs

| Dev | Week 1 | Week 2 |
|---|---|---|
| **API Dev 1** | Phase 1A (rules) → 2A-1 (Prospects load rental) → 2A-2 (Prospects validate rental) | 2A-3 (Prospects save XML rental) → 2A-4 (Leased accts validate rental) |
| **API Dev 2** | 2A-5 (Leased accts PUT rental) → 2B-1 (Prospects load employment) | 2B-2 (Prospects validate employment) → 2B-3 (Prospects save XML employment) |
| **UI Dev** | Phase 1B (flag wiring) → UI prep/prototype | 2B-4 (Leased accts validate employment) → 2B-5 (Leased accts PUT employment) |

**Sprint 1 output:** All Phase 2A + 2B APIs complete and testable. Feature flag wired.

### Sprint 2 (Weeks 3–4): Cross-Domain API + UI Begins

| Dev | Week 3 | Week 4 |
|---|---|---|
| **API Dev 1** | 3-1 (Prospects Required Fields) → 3-2 (Waitlist Required Fields) | 3-3 (Select Unit) → Code reviews + bug fixes from Sprint 1 |
| **API Dev 2** | Phase 6 (MergeFields, if product confirmed) | Support UI devs + code reviews |
| **UI Dev** | 4A-1 (Residence data layer) → 4A-2 start (Residence step UI) | 4A-2 finish → 4B-1 (Employment data layer) |

**Sprint 2 output:** All API work complete. Residence UI functional. Employment UI data layer ready.

### Sprint 3 (Weeks 5–6): UI Completion + Integration

| Dev | Week 5 | Week 6 |
|---|---|---|
| **API Dev 1** | Support UI integration + fix API issues found during UI work | 7-1 (test plan) → start E2E testing |
| **API Dev 2** | 4A-3 (Prospects UI residence) | 5-4 (Validation UI) → 5-5 (hardening) |
| **UI Dev** | 4B-2 (Employment step UI) → 5-1 (save orchestration) | 5-2 (service contracts) → 5-3 (wizard model) |

**Sprint 3 output:** Full feature functional E2E. Test plan created.

### Sprint 4 (Week 7–8): Testing + Rollout

| Dev | Week 7 | Week 8 |
|---|---|---|
| **QA** | 7-2, 7-3 (E2E tests Prospects + Accounts) | 7-4, 7-5, 7-6 (Waitlist + regression + perf) |
| **Dev 1** | Bug fixes from QA | 7-7 (rollout) |
| **Dev 2** | Bug fixes from QA | Documentation cleanup |

**Sprint 4 output:** Feature tested, verified, rolled out behind flag.

---

## 6. Integration Checkpoints

These are mandatory verification gates before proceeding to the next phase.

### Checkpoint 1 — After Phase 1 (Foundation)
- [ ] Duration aggregation rules codified in unit tests with edge cases
- [ ] Feature flag toggleable in SAT environment
- [ ] Feature branch created and CI pipeline green

### Checkpoint 2 — After Phase 2 (Domain APIs)
- [ ] Prospects GET returns N rental history records from multi-node XML
- [ ] Prospects GET returns N employment history records
- [ ] Prospects PUT validates aggregated duration and returns indexed errors
- [ ] Leased Accounts PUT /rentalHistoryMulti accepts array, inserts/updates/hard-deletes correctly
- [ ] Leased Accounts PUT /employmentDetails accepts array, inserts/updates/soft-deletes correctly
- [ ] Leased Accounts validation returns `previousResidenceLengthOfStay` / `employmentHistoryLengthOfEmployment` in missingFields
- [ ] All existing unit tests still pass
- [ ] Flag OFF: all endpoints behave identically to pre-change

### Checkpoint 3 — After Phase 3 (Cross-Domain API)
- [ ] Prospects household members completion check validates N records
- [ ] Waitlist `FillRentalHistory` / `FillEmploymentHistory` handles N records
- [ ] ContactsManager parses multi-node XML; WaitlistManager loops correctly
- [ ] API regression suite green

### Checkpoint 4 — After Phase 4+5 (UI Complete)
- [ ] User can load household member with 4+ previous residences — all render
- [ ] User can add new empty row, fill it, save → record persisted
- [ ] User can delete unsaved row → immediate removal, no API call
- [ ] User can delete saved row → confirmation, deleteIds sent, record removed
- [ ] Employment: same as above, soft-delete confirmed
- [ ] Single PUT per domain per save (network tab verified)
- [ ] Indexed validation errors highlight correct repeater row
- [ ] Aggregate insufficiency message displayed when total < minimum
- [ ] Flag OFF: legacy two-entry UI and POST save methods used
- [ ] Prospects flow: N records load, save, validate correctly
- [ ] No regressions in unrelated household member steps (billing, custom fields, etc.)

### Checkpoint 5 — After Phase 7 (E2E + Rollout)
- [ ] Full E2E test plan executed and passing
- [ ] Flag-OFF regression passing
- [ ] Performance acceptable with 10+ records
- [ ] Rollout approved for staging → production

---

## 7. Risk Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| **Mandatory Fields flag stays disabled** — feature validation cannot be tested E2E | High | High | Create test-specific LD override in SAT/QA environment. Document that production enablement requires both flags. |
| **household-member-manage-ctrl.js merge conflicts** | High | Medium | Coordinate with other teams on change windows. Make focused, minimal edits. Consider branching strategy per-phase. |
| **Duration aggregation rules unclear** — Product hasn't answered overlaps/gaps/rounding | Medium | High | **Block implementation until answered.** If delayed >3 days, escalate to PO. Provide default rules (simple sum, no dedup) for team to start with, marked as assumption. |
| **Bug 2621966 not resolved** — blocks Waitlist validation work | Medium | Medium | Phase 3-2 can be deferred to Sprint 4 without blocking main feature. Track separately. |
| **Leased accounts PUT partially succeeds** — row-level errors in bulk save | Medium | Medium | Design error response contract in Phase 2A-5/2B-5 to return per-row success/failure. UI must handle partial save state. |
| **Performance degradation with N records** | Low | Medium | Load-test with 20 records in Phase 7-6. If slow, add pagination or lazy-load. |
| **AOLR feature (2741850) creates conflicts** | Low | High | Confirm separate scope in P5. If shared contracts emerge, joint design review before Phase 2. |

---

## 8. Environment & Testing Strategy

### Test Environment

| Environment | Purpose | Flag State |
|---|---|---|
| Local dev | Unit testing, manual debugging | ON (hardcoded override) |
| SAT | API integration testing, UI validation | ON (LD override for test property) |
| SAT (regression) | Flag-OFF regression | OFF (default) |
| Staging | Pre-production E2E | ON (controlled rollout property) |
| Production | Rollout | OFF initially → ON per-customer |

### Test Data Requirements

- At least one property with minimum rental history configured (X months)
- At least one property with minimum employment history configured
- Household members with 0, 1, 2, 3, 5+ previous rental/employment records
- Edge case: records with overlapping date ranges
- Edge case: records with gaps between date ranges
- Edge case: open-ended current record (no move-out / end date)

### Mandatory Fields LD Workaround

Since the Mandatory Fields flag is disabled globally, the testing strategy is:
1. **Unit tests** — test validation methods directly without LD dependency
2. **SAT override** — enable Mandatory Fields flag via LD targeting for specific test property/user
3. **Integration tests** — verify via API calls with LD override context
4. **Document** — clearly state that production activation requires both flags enabled

---

## 9. Code Review Strategy

### Review Assignments

Given the high-risk nature of `ProspectListManager.cs` (10,000+ lines) and `household-member-manage-ctrl.js` (orchestration hub):

| Phase | Primary Reviewer | Focus |
|---|---|---|
| Phase 1A (rules) | Tech Lead | Correctness of aggregation logic, edge cases |
| Phase 2A/2B (API) | Senior API dev (cross-review between Dev 1 ↔ Dev 2) | XPath changes, validation chain integrity, feature flag gating |
| Phase 3 (cross-domain) | Tech Lead | PeopleManager fill refactor, backward compatibility |
| Phase 4A/4B (UI step) | UI Lead | Repeater rendering, ng-repeat patterns, form state management |
| Phase 5 (shared UI) | Tech Lead + UI Lead (joint) | Save orchestration, backward compatibility, regression surface |

### Review Checklist (per PR)

- [ ] Feature flag gating present on all new behavior
- [ ] Existing unit tests still pass
- [ ] No method signature changes that break existing callers (additive only)
- [ ] Legacy path (flag OFF) is unchanged
- [ ] New code has unit test coverage
- [ ] Indexed error key format consistent (`fieldKey_0`, `fieldKey_1`)
- [ ] No hardcoded record limits (supports N records)

---

## 10. Rollout Plan

### Stage 1: Internal (Flag ON for dev/QA properties)
- Enable `accounts-employment-and-rental-length` flag for 2-3 test properties
- Enable `addEditHouseholdMemberMandatoryFieldsCheckEnabled` for same properties
- Execute full E2E test plan
- Duration: 1 sprint

### Stage 2: Beta (Flag ON for pilot customer)
- Identify willing pilot customer (preferably from Barrett and Stokely — original requester per tags)
- Monitor for errors, performance, user feedback
- Duration: 2 weeks

### Stage 3: General Availability
- Enable flag for all customers
- Monitor error rates and support tickets
- Rollback trigger: >5% increase in save errors or support tickets mentioning rental/employment history

### Rollback Procedure
1. Turn `accounts-employment-and-rental-length` OFF in LaunchDarkly
2. Existing records created with N entries remain in database
3. UI reverts to showing only primary + other previous (first 2 records)
4. Save path reverts to legacy POST (one record at a time)
5. No data loss — extra records are still in DB, just not visible until re-enabled

---

## Appendix: Task Dependency Quick Reference

```
1A ──► 2A-1 ──► 2A-3
 │      │
 │      └──► 2A-2
 │
 ├──► 2A-4
 │
 ├──► 2B-1 ──► 2B-3
 │      │
 │      └──► 2B-2
 │
 ├──► 2B-4
 │
 ├──► 3-1 (needs 2A-1 + 2B-1)
 │
 ├──► 3-2 (needs Bug 2621966)
 │
 └──► 3-3

2A-5 (independent) ──► 4A-1 ──► 4A-2 ──► 5-1
                                           │
2B-5 (independent) ──► 4B-1 ──► 4B-2 ──► 5-1
                                           │
                                           └──► 5-2 ──► 5-3 ──► 5-4 ──► 5-5 ──► 7-1

1B (UI flag wiring, independent)

6-1 ──► 6-2 ──► 6-3 (independent track)
```

**Shortest possible calendar with 2 API + 1 UI devs: ~7 weeks (3.5 sprints)**

**Conservative estimate with buffer: 4 sprints (8 weeks)**
