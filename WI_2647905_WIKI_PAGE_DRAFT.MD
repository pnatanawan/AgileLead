# WI 2647905: Enable Applicant Reservation Commitment for Expired Quotes

**Work Item ID:** 2647905  
**PME ID:** PME-484256  
**Status:** Active (Updated Feb 28, 2026)  
**Priority:** 3 (Medium) — **EXPEDITE** flag active  
**Related WI:** PME-461308 (Cancel Reservation for Expired Quotes)  
**Outstanding Since:** November 2023 (2+ years)

---

## Executive Summary

Currently, when a quotation expires (standard 48-hour window) but the applicant has an active reservation on the unit (separate 48-hour window from quote date), the **New OneSite system incorrectly blocks selecting that unit**, while the **Classic OneSite system correctly allows it**. This creates a blocking issue for applicants at Cambridge Thornton Park property and other sites.

**Root Cause:** Quote expiration check happens before reservation validity check in the business logic layer.

**Solution:** Decouple quote validity from reservation validity, allowing "Select Unit" action to be available for expired quotes that have valid reservations. All changes will be gated behind a LaunchDarkly feature flag for safe rollout and instant rollback.

**Effort Estimate:** 2-3 working days development + 1 day testing = 3-5 story points  
**Architect Confidence:** 80-85%

---

## Problem Statement

### Business Context

In OneSite, when prospects apply for units, they interact with two time-bound concepts:

| Concept | Window | Purpose |
|---------|--------|---------|
| **Quote Expiration** | 2 days (configurable per PMC) | Quotation validity period |
| **Reservation Hold** | 3 days (configurable per PMC) | Unit hold period after reservation |

#### The Divergence Scenario

1. **Hour 0:** Prospect receives quotation, reserves unit
   - Quote expires at: Hour 48 (2 days)
   - Reservation expires at: Hour 72 (3 days) — **8-hour divergence begins**

2. **Hour 48:** Quote expires
   - Classic OneSite: "Select Unit" still available (reservation is valid)
   - **New OneSite (BUG):** "Select Unit" hidden/disabled (quote expired)

3. **Hour 72:** Reservation expires
   - Both systems: Unit hold released

**Impact:** Applicants cannot apply their valid reservation to the unit once the quote expires, even though both the quote-as-communication and the unit-hold are separately valid concepts.

---

## Acceptance Criteria

### Updated Functional Requirements

✅ **UI Layer:** "Select Unit" action visible for expired quotes with valid reservations
- Condition 1: `quote.Status = "expired"` OR `quote.Status = "reserved"`
- Condition 2: `quote.QuoteReserved = true` (active reservation bit)
- Condition 3: User has `C_MOVEIN_DATE_UNIT_RIGHT` permission
- Condition 4: Unit is not on hold (`!unit.UnitHold`)
- Condition 5: Feature flag enabled: `reservation_commit_decoupled_from_quote`

✅ **Backend Validation:** Backend processing allows unit commitment when reservation is valid
- Method: `CommitUnitManager.ValidateApplicantCommitUnit(applicantId, unitId)`
- Logic: Check reservation validity independently from quote status
- Falls back to legacy behavior when feature flag is OFF

✅ **Data Retrieval:** Reservation validity determination
- Source: Database stored procedure `uspCheckReservationValidity`
- Validation: Check if `Reservations.ExpirationDate > GETUTCDATE()` (or equivalent column)
- Result: Boolean (true = valid, false = expired)

### Updated Non-Functional Requirements

✅ **Feature Flag:** All behavioral changes gated behind LaunchDarkly
- Flag Name: `reservation_commit_decoupled_from_quote`
- Flag Type: Boolean
- Default Value: `false` (legacy behavior)
- Scope: Per-applicant context

✅ **Configuration:** Respect PMC-configured time windows
- Quote expiration: PMC Config setting (default: 2 days)
- Reservation hold: PMC Config setting (default: 3 days)
- Behavior: Independent validation of each expiration

✅ **Backward Compatibility:** No impact to existing workflows
- Disabled by default (flag OFF = legacy OneSite behavior)
- Gradual rollout to specific properties/applicants via LD rules
- Instant rollback via flag toggle (no deployment needed)

✅ **Data Integrity:** No schema changes required
- All data fields already exist and are populated
- `QuoteReserved` bit already mapped in DAO layer
- Reservation expiration concept already in use (per PME-461308)

---

## Technical Architecture

### 4-Tier Implementation

#### 1. **Presentation Layer (UI)**

**File:** `ONESITE WEB/RP/RealPage/OneSite/Prospects/.../BusinessLogic/ProspectListManager.cs`

**Method:** `SetQuotationsAction()` (line ~1015-1030)

**Current Behavior:**
```csharp
foreach (var q in quotationList)
{
    if (q.Status == "reserved" && q.SiteId == propertyId)
    {
        // Show actions: viewPrint, email, cancelReservation
        q.Actions.Add("cancelReservation");
    }
    else
    {
        // Show: viewPrint, email ONLY
    }
    
    // Missing: selectUnit for expired quotes with valid reservations
}
```

**Required Change:** Add condition for expired quotes with valid reservations

```csharp
if (q.Status == "reserved" && q.SiteId == propertyId)
{
    // Existing behavior
    q.Actions.AddRange(new[] { "viewPrint", "email", "cancelReservation" });
}
else if (IsFeatureFlagEnabled("reservation_commit_decoupled_from_quote")
    && q.Status == "expired"
    && q.QuoteReserved
    && q.SiteId == propertyId)
{
    // NEW: Show selectUnit for expired but reserved quotes
    if (HasRight(C_MOVEIN_DATE_UNIT_RIGHT) && !q.UnitHold)
    {
        q.Actions.AddRange(new[] { "viewPrint", "email", "selectUnit" });
    }
}
```

**Note:** The UI component `quotes-grid-comp.js` already handles both `cancelReservation` and `selectUnit` actions — no UI framework changes needed.

#### 2. **Business Logic Layer (Validation)**

**File:** `RealPage/OneSite/Applicants/.../BusinessLogic/CommitUnitManager.cs`

**Method:** `ValidateApplicantCommitUnit()` (line ~285)

**Current Behavior:**
```csharp
public ValidationResult ValidateApplicantCommitUnit(int applicantId, int unitId)
{
    // Only checks quote expiration, not reservation validity
    if (IsQuoteExpired(applicantId))
        return ValidationResult.Failure("Quote has expired.");
    
    return ValidationResult.Success();
}
```

**Required Changes:** Add reservation validity check

```csharp
public ValidationResult ValidateApplicantCommitUnit(int applicantId, int unitId)
{
    // Check if feature flag enabled
    if (IsFeatureFlagEnabled("reservation_commit_decoupled_from_quote"))
    {
        // NEW: Check reservation validity independently
        if (IsReservationValid(applicantId, unitId))
            return ValidationResult.Success("Reservation is valid.");
        
        // Only fail if BOTH quote AND reservation are expired
        if (IsQuoteExpired(applicantId) && !IsReservationValid(applicantId, unitId))
            return ValidationResult.Failure("Quote expired and no valid reservation.");
    }
    else
    {
        // Legacy behavior: quote must not be expired
        if (IsQuoteExpired(applicantId))
            return ValidationResult.Failure("Quote has expired.");
    }
    
    return ValidationResult.Success();
}

// NEW METHOD
private bool IsReservationValid(int applicantId, int unitId)
{
    return _commitUnitDao.CheckReservationValidity(applicantId, unitId);
}
```

**Key Logic:**
- When flag is ON: Reservation validity is checked independently
- When flag is OFF: Legacy behavior (quote must be valid)
- Allows unit commitment even if quote expired, so long as reservation is valid

#### 3. **Data Access Layer (DAO)**

**File:** `RealPage/OneSite/Applicants/.../DAO/CommitUnitDAO.cs`

**New Method:** `CheckReservationValidity()`

```csharp
/// <summary>
/// Checks if the reservation for an applicant on a specific unit is still valid.
/// </summary>
/// <param name="applicantId">Applicant ID</param>
/// <param name="unitId">Unit ID</param>
/// <returns>true if reservation is valid (not expired); false if expired or not found</returns>
public bool CheckReservationValidity(int applicantId, int unitId)
{
    return ExecuteScalar<bool>(
        "uspCheckReservationValidity",
        new { ApplicantId = applicantId, UnitId = unitId }
    );
}
```

**Implementation Pattern:**
- Uses existing DAO pattern: `ExecuteScalar<T>(sprocName, parameters)`
- Calls new stored procedure `uspCheckReservationValidity`
- Returns Boolean directly (true/false)
- No schema changes needed — uses existing tables/columns

#### 4. **Database Layer (Stored Procedure)**

**New Stored Procedure:** `uspCheckReservationValidity`

**Location:** SQL Server (Realpage database)

**Implementation:**

```sql
CREATE PROCEDURE [dbo].[uspCheckReservationValidity]
    @ApplicantId INT,
    @UnitId INT 
AS 
BEGIN
    SET NOCOUNT ON;
    
    -- Check if reservation exists and has not expired
    SELECT CASE
        WHEN EXISTS (
            SELECT 1
            FROM Reservations
            WHERE ApplicantId = @ApplicantId
              AND UnitId = @UnitId
              AND ExpirationDate > GETUTCDATE()
              AND IsActive = 1  -- Check if reservation is still active
        )
        THEN 1 
        ELSE 0 
    END;
END
```

**Alternative Implementation** (if Reservations table doesn't exist):

If the schema uses `QUOTATION` table with `quoteReservationDate`:

```sql
CREATE PROCEDURE [dbo].[uspCheckReservationValidity]
    @ApplicantId INT,
    @UnitId INT 
AS 
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @ReservationWindowDays INT = 3;  -- From PMC Config
    
    SELECT CASE
        WHEN EXISTS (
            SELECT 1
            FROM Quotation
            WHERE LeaseId = @ApplicantId
              AND UnitId = @UnitId
              AND quoteReserveBit = 1
              AND DATEADD(DAY, @ReservationWindowDays, quoteReservationDate) > GETUTCDATE()
        )
        THEN 1 
        ELSE 0 
    END;
END
```

**Schema Verification Needed:**
- Confirm: Does `Reservations` table exist?
- Confirm: Column name for reservation start date (`quoteReservationDate`? `reservationCreatedDate`?)
- Confirm: Column name for reservation flag/status (`qoteReserveBit`? `IsReservationActive`?)
- Confirm: UTC vs local time handling in database

---

## LaunchDarkly Feature Flag Configuration

### Flag Definition

| Property | Value |
|----------|-------|
| **Flag Name** | `reservation_commit_decoupled_from_quote` |
| **Flag Type** | Boolean |
| **Default Value** | `false` (Legacy behavior) |
| **Description** | Allow applicants to commit units via valid reservations even when quotes are expired |
| **Owner** | Engineering / Product (RealPage Real Estate) |
| **Related Flags** | `cancel-reservation-expired-quotes` (PME-461308) |
| **Rollback Plan** | Toggle flag OFF (instant, no deployment) |

### Rollout Strategy

**Phase 1: Validation** (Day 1)
- Enable for: QA environment only
- Scope: All applicants
- Duration: Full day UAT

**Phase 2: Pilot** (Day 2-3)
- Enable for: Production — Camden Thornton Park property (specific PMC)
- Scope: Applicant "Shanell Paraison" + 5-10 additional test applicants
- Duration: 24-48 hours monitoring
- Success criteria: No reported issues, validation passes

**Phase 3: Rollout** (Day 4+)
- Enable for: Production — All affected PMCs
- Scope: Gradual rollout (10% → 50% → 100%)
- Duration: Monitored over 1 week
- Rollback: Available instantly if issues detected

### Code Integration

**C# Implementation:**

```csharp
public class CommitUnitValidator
{
    private readonly ILaunchDarklyClient _ldClient;
    private const string FLAG_NAME = "reservation_commit_decoupled_from_quote";
    
    public bool IsReservationDecoupledEnabled(int applicantId)
    {
        var context = Context.New(applicantId.ToString());
        return _ldClient.BoolVariation(FLAG_NAME, context, false);
    }
}
```

---

## Testing Strategy

### Unit Tests

| Test ID | Scenario | Conditions | Expected Result | Status |
|---------|----------|-----------|-----------------|--------|
| **UT-01** | Valid reservation + expired quote | LD ON, res valid, quote expired | ✅ Pass validation | Pending |
| **UT-02** | Expired reservation + expired quote | LD ON, res expired, quote expired | ❌ Fail validation | Pending |
| **UT-03** | Valid reservation + valid quote | LD ON, res valid, quote valid | ✅ Pass validation | Pending |
| **UT-04** | Reserved quote (legacy) | LD OFF, res valid, quote valid | ✅ Pass validation | Pending |
| **UT-05** | Feature flag OFF | LD OFF, res valid, quote expired | ❌ Fail validation (legacy) | Pending |
| **UT-06** | Null reservation | LD ON, no reservation record | ❌ Fail validation | Pending |
| **UT-07** | Permission check failure | LD ON, res valid, no rights | ❌ UI hidden, no validation | Pending |
| **UT-08** | Unit on hold | LD ON, res valid, unit held | ❌ UI hidden, no validation | Pending |

### Integration Tests

| Test ID | Scenario | System Flow | Expected Outcome |
|---------|----------|-------------|------------------|
| **IT-01** | **End-to-end: Quote expires, reservation valid, select unit** | Prospect reserves unit (T0) → Quote expires (T+48h) → Reservation still valid (T+48-72h) → UI shows "Select Unit" → Backend validates → Unit committed | ✅ Success: Unit committed via valid reservation |
| **IT-02** | **Quote expires while processing** | Quote approaching expiration → User clicks "Select Unit" at T+47h → Validation completes at T+48:30h | ✅ Success: Time check is at validation, not at UI render |
| **IT-03** | **Concurrent reservations** | Two applicants have reservations on same unit → First commits at T+48h → Second attempts commit at T+49h | ✅ Success: First wins, second fails (unit no longer available) |
| **IT-04** | **Feature flag toggle during session** | Applicant in middle of checkout → Flag toggled OFF → Next validation call | ✅ Success: Uses updated flag value (legacy behavior) |

### Manual Tests (UAT)

| Test ID | User Story | Steps | Expected Result |
|---------|-----------|-------|-----------------|
| **MT-001** | **Camden Thornton Park - Shanell Paraison** | 1. Log in as Shanell 2. View quotes 3. Verify "Select Unit" visible on expired-but-reserved quote 4. Click "Select Unit" 5. Complete move-in process | ✅ Unit committed successfully |
| **MT-002** | **Expired quote actions dropdown** | 1. Navigate to prospects 2. Find expired quote with active reservation 3. Click actions dropdown 4. Verify actions: [viewPrint, email, selectUnit] | ✅ All actions visible |
| **MT-003** | **No reservation behavior** | 1. Find expired quote WITHOUT reservation 2. Verify "Select Unit" NOT visible 3. Verify only [viewPrint, email] available | ✅ Correct actions for non-reserved |
| **MT-004** | **Permission-based visibility** | 1. Test as applicant WITH C_MOVEIN_DATE_UNIT_RIGHT 2. Test as applicant WITHOUT right 3. Verify visibility matches | ✅ Permissions enforced in UI |
| **MT-005** | **Unit hold constraint** | 1. Find expired quote with valid reservation 2. Place hold on unit 3. Verify "Select Unit" hidden | ✅ Hold constraints honored |
| **MT-006** | **Multiple PMCs** | 1. Test with PMC Config: Quote 2d, Reservation 3d 2. Test with PMC Config: Quote 1d, Reservation 2d | ✅ Each PMC honored correctly |
| **MT-007** | **Mobile/Responsive** | 1. Access quotes via mobile browser 2. Verify "Select Unit" action accessible 3. Complete unit selection on mobile | ✅ Mobile workflow works |
| **MT-008** | **Classic OneSite parity** | 1. Replicate same scenario in Classic OneSite 2. Verify New OneSite gives same result | ✅ Behavior matching |

---

## Implementation Checklist

### Pre-Implementation (Database Team)

- [ ] **Verify Reservations table structure**
  - [ ] Confirm table exists: `Reservations` or `QUOTATION`
  - [ ] Identify reservation start date column: `quoteReservationDate`? `reservationCreatedDate`?
  - [ ] Identify expiration column: `ExpirationDate`? Column exists?
  - [ ] Confirm primary key structure: `(ApplicantId, UnitId)`
  - [ ] Verify indexing on frequently-queried columns

- [ ] **Verify data population**
  - [ ] Reservation records created when quote is reserved
  - [ ] Expiration dates calculated correctly (creation date + PMC config window)
  - [ ] Soft delete handling (IsActive flag or hard delete?)

- [ ] **Verify PMC configuration**
  - [ ] Locate table: `PMC_CONFIG` or `PROPERTIES`
  - [ ] Fields for: `Quote_Expiration_Days`, `Reservation_Hold_Days`
  - [ ] Default values: 2 days quote, 3 days reservation
  - [ ] Any overrides per site (`SITE_SETTINGS`)?

### Development (Backend)

- [ ] **ProspectListManager.cs changes**
  - [ ] Add feature flag check in `SetQuotationsAction()` method
  - [ ] Add condition for expired + reserved quotes
  - [ ] Verify permission checks still enforced
  - [ ] Verify unit hold checks still enforced
  - [ ] Test with feature flag ON/OFF

- [ ] **CommitUnitManager.cs changes**
  - [ ] Add `IsReservationValid()` method
  - [ ] Modify `ValidateApplicantCommitUnit()` logic
  - [ ] Add feature flag conditional
  - [ ] Implement fallback to legacy behavior when flag OFF
  - [ ] Add error message clarity: "Quote expired but reservation is valid"

- [ ] **CommitUnitDAO.cs changes**
  - [ ] Add `CheckReservationValidity(applicantId, unitId)` method
  - [ ] Call stored procedure with correct parameters
  - [ ] Handle null/empty results (return false by default)
  - [ ] Add logging for troubleshooting

- [ ] **Stored Procedure (SQL)**
  - [ ] Create `uspCheckReservationValidity` or verify existence
  - [ ] Implement CASE/WHEN/EXISTS logic
  - [ ] Handle UTC time comparisons
  - [ ] Add parameter validation
  - [ ] Test with boundary cases (exact expiration time)

- [ ] **LaunchDarkly Integration**
  - [ ] Create flag: `reservation_commit_decoupled_from_quote`
  - [ ] Set default to `false`
  - [ ] Add context builder for applicant ID
  - [ ] Test flag toggle on/off behavior
  - [ ] Document flag in team wiki/documentation

### Testing (QA)

- [ ] **Unit Tests**
  - [ ] Run UT-01 through UT-08 (see Testing Strategy section)
  - [ ] Achieve 85%+ code coverage on new methods
  - [ ] Mock stored procedure calls
  - [ ] Test feature flag ON/OFF branch coverage

- [ ] **Integration Tests**
  - [ ] Run IT-01 through IT-04 (see Testing Strategy section)
  - [ ] Test database integration with real stored procedure
  - [ ] Verify concurrent reservation handling
  - [ ] Verify flag toggle during active sessions

- [ ] **Manual UAT**
  - [ ] Run MT-001 through MT-008 (see Testing Strategy section)
  - [ ] Reproduce Shanell Paraison's original issue
  - [ ] Test across multiple PMCs with different configs
  - [ ] Test on mobile/responsive views
  - [ ] Compare behavior with Classic OneSite

- [ ] **Regression Testing**
  - [ ] Verify flag OFF behavior matches legacy
  - [ ] Test other quote actions (cancelReservation, email, print)
  - [ ] Verify no impact to non-expired quotes
  - [ ] Test permission checks still enforced

### Post-Implementation

- [ ] **LaunchDarkly Rollout**
  - [ ] Phase 1: QA environment validation
  - [ ] Phase 2: Production pilot (Camden Thornton Park)
  - [ ] Phase 3: Gradual rollout (10% → 50% → 100%)
  - [ ] Monitor error logs and success metrics

- [ ] **Documentation**
  - [ ] Update API documentation with new validation rules
  - [ ] Document feature flag behavior in wiki
  - [ ] Create troubleshooting guide
  - [ ] Document PMC configuration requirements

- [ ] **Stakeholder Communication**
  - [ ] Notify Shanell Paraison of fix (Feb 28, 2026 — 2+ years outstanding!)
  - [ ] Update product team on deployment status
  - [ ] Provide runbooks for support team
  - [ ] Schedule training if needed

---

## Related Work Items

### Parent Work Item
- **WI #2596920:** Feature request: Applicant unit selection improvements

### Related/Blocked By
- **WI #2721524:** Reservation hold configuration
- **WI #2722354:** Quote expiration handling
- **WI #2496779:** Classic OneSite parity

### Parallel/Reference
- **PME-461308:** Cancel Reservation for Expired Quotes (related, shares same pattern)
  - Uses identical LaunchDarkly feature flag approach
  - Modifies same `ProspectListManager.cs` file
  - Addresses complementary scenario (cancel vs select for expired+reserved)
  - Recommended: Implement both in same release if possible

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        USER INTERFACE LAYER                      │
│  quotes-grid-comp.js (AngularJS component)                      │
│  ├─ Actions: [viewPrint, email, selectUnit, cancelReservation]  │
│  └─ Renders actions returned from backend                        │
└────────────────────────────┬────────────────────────────────────┘
                             │
        ┌────────────────────┴────────────────────┐
        │                                         │
┌───────▼──────────────────────┐    ┌────────────▼─────────────────┐
│  PRESENTATION LAYER (C#)     │    │   LaunchDarkly (Feature Gate) │
│  ProspectListManager         │    │                               │
│  ├─ SetQuotationsAction()    │    │ Flag:                         │
│  │  ├─ Check: quote.Status   │    │   reservation_commit_         │
│  │  ├─ Check: quote.Reserved │◄───┤   decoupled_from_quote        │
│  │  └─ Build action list     │    │ Default: false               │
│  └─ Calls: CommitUnitManager │    │ Type: Boolean                │
└───────┬──────────────────────┘    └──────────────────────────────┘
        │
        │ ValidateApplicantCommitUnit(applicantId, unitId)
        │
┌───────▼──────────────────────────────────────────────────────────┐
│         BUSINESS LOGIC LAYER (CommitUnitManager)                 │
│                                                                   │
│  ┌─ Legacy Path (Flag OFF) ──────────────────┐                  │
│  │ if (IsQuoteExpired()) return FAILURE      │                  │
│  │                                            │                  │
│  └─ New Path (Flag ON) ────────────────────┐ │                  │
│    if (IsReservationValid()) return SUCCESS ├─┤                  │
│    if (IsQuoteExpired()) return FAILURE     │ │                  │
│                              ┌──────────────┘ │                  │
│                              ▼                 │                  │
│              Calls: CheckReservationValidity() │                  │
│              Returns: Boolean (valid/expired)  │                  │
│                                                │                  │
└────────────┬─────────────────────────────────┬┘                  │
             │                                 │
             └────────────────┬────────────────┘
                              │
        ┌─────────────────────┴──────────────────┐
        │                                        │
┌───────▼──────────────────┐      ┌─────────────▼────────────────┐
│   DATA ACCESS LAYER (DAO)│      │  DATABASE LAYER (SQL Server) │
│   CommitUnitDAO          │      │                              │
│   ├─ New Method:         │      │ Stored Procedure:            │
│   │ CheckReservation     │◄─────┤   uspCheckReservationValidity│
│   │ Validity()           │      │                              │
│   │                      │      │ Logic:                       │
│   │ Calls: ExecuteScalar │      │   SELECT CASE WHEN EXISTS    │
│   │   stored procedure   │      │     FROM Reservations        │
│   │ Returns: bool        │      │     WHERE ExpirationDate > ? │
│   └──────────────────────┘      │   THEN 1 ELSE 0             │
│                                  │                              │
└──────────────────────────────────┴──────────────────────────────┘
```

---

## Effort & Timeline

### Development Effort

| Component | Estimated Effort | Notes |
|-----------|-----------------|-------|
| ProspectListManager.cs | 2-3 hours | UI action visibility logic |
| CommitUnitManager.cs | 3-4 hours | Core validation logic, feature flag integration |
| CommitUnitDAO.cs | 1-2 hours | New method wrapper |
| uspCheckReservationValidity SP | 1-2 hours | SQL logic (or field verification if exists) |
| LaunchDarkly Configuration | 1 hour | Flag setup and context builder |
| **Total Development** | **8-12 hours (1-2 days)** | |

### Testing Effort

| Phase | Estimated Effort | Notes |
|-------|-----------------|-------|
| Unit Tests (UT-01 to UT-08) | 3-4 hours | Mock testing |
| Integration Tests (IT-01 to IT-04) | 3-4 hours | Database integration |
| Manual UAT (MT-001 to MT-008) | 4-5 hours | End-to-end, multi-browser |
| Regression Testing | 2-3 hours | Ensure no regressions |
| **Total Testing** | **12-16 hours (1-2 days)** | |

### Pre-Implementation (Database Verification)

| Task | Estimated Effort | Critical Path? |
|------|-----------------|----------------|
| Schema verification | 1-2 hours | YES — blocks development |
| Data population review | 1-2 hours | YES — impacts SP design |
| **Total Pre-Implementation** | **2-4 hours** | YES |

### **Total Project Timeline: 3-5 working days**
- **Day 1:** Database verification + Development setup
- **Day 2:** Full development (8-12 hours) 
- **Day 3:** Integration testing + bug fixes
- **Day 4-5:** Manual UAT + feature flag rollout

---

## Risk Assessment

### High Confidence Factors ✅
1. **Architecture Pattern Proven:** PME-461308 uses identical LaunchDarkly approach
2. **Data Fields Exist:** `QuoteReserved` field already populated and in use
3. **UI Component Ready:** quotes-grid-comp.js already handles the actions
4. **Feature Flag Platform:** LaunchDarkly already integrated in codebase
5. **Schema Likely Exists:** Quotation table has reservation concept (PME-461308 reference)

### Medium Confidence Factors ⚠️
1. **Exact Column Names:** Minor variation in Reservations vs QUOTATION table structure
2. **Stored Procedure Existence:** uspCheckReservationValidity may already exist (needs verification)
3. **PMC Configuration:** Exact location of 2-day/3-day configuration values

### Low Risk Issues 🟢
1. **Breaking Changes:** No — feature is behind flag, defaults to legacy behavior
2. **Schema Migration:** No — all data fields already exist
3. **API Changes:** No — only backend validation logic changes
4. **Concurrent Requests:** Low risk — database handles unit locks

### Rollback Strategy 🔄
- **Very Simple:** Toggle LaunchDarkly flag `reservation_commit_decoupled_from_quote` from `true` → `false`
- **No Code Deployment Required:** Instant rollback at runtime
- **Impact:** Immediate reversion to legacy behavior (quote must not be expired)
- **Rollback Time:** < 1 minute

---

## Success Criteria

### Functional Success ✅
1. ✅ Shanell Paraison (Camden Thornton Park) can apply valid reservation to unit after quote expires
2. ✅ "Select Unit" action visible for quotes with Status="expired" + QuoteReserved=true
3. ✅ Backend validation accepts unit commitment via valid reservation
4. ✅ System matches Classic OneSite behavior for expired+reserved quotes
5. ✅ Feature flag OFF reverts to legacy behavior (quote must be valid)

### Test Success ✅
1. ✅ All unit tests pass (UT-01 to UT-08)
2. ✅ All integration tests pass (IT-01 to IT-04)
3. ✅ All manual UAT tests pass (MT-001 to MT-008)
4. ✅ No regression in existing quote functionality
5. ✅ Permission checks enforced (C_MOVEIN_DATE_UNIT_RIGHT)

### Production Success ✅
1. ✅ Gradual rollout succeeds (10% → 50% → 100%)
2. ✅ No increase in error rates during rollout
3. ✅ Feature flag toggle works instantly
4. ✅ Monitoring shows expected usage patterns
5. ✅ Zero support escalations related to this change

---

## Architect Sign-Off

**Technical Specification Review:** 2647905_Technical_Specification.docx  
**Review Status:** ✅ **APPROVED**  
**Architect Confidence:** **80-85%**  
**Comments:** "Implementation is straightforward with proven architectural patterns from PME-461308. Main dependency is database schema verification (Reservations table structure), which is low-risk given existing similar work."

---

## Notes & References

### Related Files in Repository
- **2647905.plan.md** — Original bug fix and implementation plan (attached to WI)
- **2647905_Technical_Specification.docx** — Architect-reviewed specification
- **PME-461308 Technical Spec** — Related work item with identical pattern for Cancel Reservation

### External References
- Classic OneSite source code (behavior comparison)
- LaunchDarkly SDK documentation (feature flag integration)
- ProspectListDAO schema reference (QuoteReserved mapping at index 14)

### Stakeholders
- **Product Owner:** Leelakrishna Balu (Acceptance Criteria Sign-Off)
- **Architecture Review:** Completed (80-85% Confidence)
- **Database Team:** Schema Verification Required (Blocking)
- **QA Lead:** Testing Strategy Defined
- **Applicant Impacted:** Shanell Paraison (Camden Thornton Park)

---

## Change Log

| Date | Change | Reason |
|------|--------|--------|
| Feb 28, 2026 | Initial Technical Analysis | Extracted from 2647905_Technical_Specification.docx |
| Feb 28, 2026 | Correlated with PME-461308 | Identified parallel work patterns |
| Feb 28, 2026 | Complete Wiki Page Draft | Synthesized all findings for publication |

---

**Last Updated:** February 28, 2026  
**Status:** Ready for Product Team Review and Development Phase

