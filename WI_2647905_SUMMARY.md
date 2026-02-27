# Azure DevOps Work Item 2647905 - Summary

## Basic Information

| Field | Value |
|-------|-------|
| **Work Item ID** | 2647905 |
| **PME ID** | PME-484256 |
| **Type** | User Story |
| **Status** | Defined (Ready for Engineering) |
| **Priority** | 3 (Medium) |
| **Severity** | 3 - Medium |
| **Assigned To** | Leelakrishna Balu |
| **Created** | 2025-12-17 by svc-mulesoft |
| **Last Updated** | 2026-02-19 by Paolo Natanawan (TODAY) |
| **Business Value** | 20 |
| **Functional Area** | Accounts - Prospects (Quotes) |
| **Sprint** | PI 26.Q1 (Sprint 26.05) |
| **Comments** | 7 |

---

## The Problem

**Title:** PME-484256 - EXPEDITE | Camden Thornton Park - Applicant Shanell Paraison - Unable to Apply Reservation to Unit in New OneSite

### Issue Summary
Camden Thornton Park encountered a critical gap between **Classic OneSite** and **New OneSite** regarding applicant unit reservations. When a quote expires but the reservation remains valid, New OneSite does not allow applicants to be applied to units, whereas Classic OneSite does.

### Specific Scenario
- **Applicant:** Shanell Paraison
- **Customer:** Camden Thornton Park (CAMDEN DEVELOPMENT, INC. | PMC ID: 1051412)
- **Unit:** Apartment #644
- **What Happened:**
  1. Applicant reserved apartment #644 (created a quote)
  2. Reservation put apartment in reserve status within 48-hour quote window
  3. Quote expired after 48 hours
  4. **Reservation remains valid** for another 48 hours per business rules
  5. In **Classic OneSite**: Team can "Select Unit" from the eclipse dropdown under Quotes/Reserved section
  6. In **New OneSite**: "Select Unit" option is **NOT AVAILABLE**, blocking unit commitment

### Root Cause
The business logic in New OneSite has **hardcoded coupling between quotes and reservations**, whereas Classic OneSite treats them as independent entities with separate expiration timelines:

1. **Classic Logic:** Reservation creates a new independent 48-hour window after applicant reserves the unit
2. **New OneSite Logic:** Tightly couples quote and reservation validation, blocking unit selection when quote expires regardless of reservation validity

---

## Technical Analysis & Solution Plan

### Files Involved
- `RealPage\OneSite\Applicants\Realpage.OneSite.Applicants\BusinessLogic\Class\CommitUnitManager.cs` (line 82+)
- `RealPage\OneSite\Applicants\Realpage.OneSite.Applicants\DAO\Class\CommitUnitDAO.cs` (line 124+)
- UnifiedQuotes controller
- Database stored procedures for unit/reservation validation

### Expected Fix Implementation

**Primary Change:** Modify `CommitUnitManager.ValidateApplicantCommitUnit()` to check reservation validity independently from quote expiration.

**Key Changes Required:**
1. Add new method `IsReservationValid()` in CommitUnitManager
2. Modify validation logic to allow unit selection if:
   - Unit is available, OR
   - Unit is unavailable BUT a valid reservation exists (within 48-hour window)
3. Add new DAO method `CheckReservationValidity()` in CommitUnitDAO
4. Create database stored procedure `uspCheckReservationValidity` to validate reservation status
5. Update UI dropdown to show "Select Unit" option for valid reservations even with expired quotes
6. Add comprehensive logging to distinguish reservation-based vs quote-based selections

---

## Related Work Items

### Linked Items (Relations)
- **Parent (Hierarchy):** WI#2596920
- **Predecessors (Dependencies):** WI#2721524, WI#2722354
- **Related:** WI#2496779
- **Pull Request:** 1 attached
- **Attachment:** 2647905.plan.md (contains detailed bug fix plan)

---

## Latest Activity & Discussions

### Recent Status Changes
- **Created:** December 17, 2025 (by svc-mulesoft automation)
- **Last Activity:** February 19, 2026 (TODAY - by Paolo Natanawan)
- **Total Comments:** 7 discussions

### Key Discussion Points
The attached plan document indicates discussions around:
1. **Business Logic Discrepancy** between Classic and New OneSite
2. **Technical Deep Dive** into quote vs reservation validation
3. **Implementation Strategy** with detailed code changes needed
4. **Testing Approach** including edge cases:
   - Boundary testing at 48-hour expiration marks
   - Multiple reservations from same applicant
   - Unit availability changes after reservation
   - Concurrent user access
5. **Risk Mitigation** including:
   - Audit trail tracking
   - Feature flag considerations for rollout
   - Impact on reporting and integrations

---

## Key Takeaways

1. **Bug Type:** Functional Gap (New OneSite missing Classic OneSite feature)
2. **Customer Impact:** HIGH - Shows as "EXPEDITE" - 2+ PMEs and 2+ Salesforce cases
3. **Outstanding Duration:** 2+ years (since November 2023)
4. **Priority:** Medium (Standard priority but business impact is significant)
5. **Current State:** Ready for engineering work with detailed technical plan attached
6. **Complexity:** Moderate - Requires C# code changes, DAO updates, and DB stored procedure

---

## Next Steps
The work item is in **"Defined"** state, which means it's approved and ready for development. The attached plan document provides comprehensive guidance for implementation, testing, and risk mitigation.
