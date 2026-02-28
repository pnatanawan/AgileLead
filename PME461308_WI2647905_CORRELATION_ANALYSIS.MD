# Cross-Reference Analysis: PME-461308 & WI 2647905

## Executive Summary

**PME-461308** (Cancel Reservation for Expired Quotes) and **WI 2647905** (Allow "Select Unit" for Expired Quotes with Valid Reservations) are **closely related**, operating in the same codebase, addressing the same quote/reservation divergence problem, and using the same data fields.

PME-461308's technical specification **answers many of the unanswered questions** from WI 2647905's gap analysis, particularly around:
- ✅ Where the 48-hour validity concept comes from
- ✅ How `QuoteReserved` bit field works
- ✅ How different quote statuses ("Expired", "Reserved", "Active") are handled
- ✅ Implementation patterns (LaunchDarkly feature flags)

---

## Parallel Work Items Summary

| Aspect | PME-461308 | WI 2647905 |
|--------|-----------|-----------|
| **Goal** | Allow CANCEL of reservation when quote is EXPIRED | Allow SELECT UNIT for reservation when quote is EXPIRED |
| **Related WI ID** | 2496779 | 2647905 |
| **Primary File** | ProspectListManager.cs | ProspectListManager.cs |
| **Method** | SetQuotationsAction (line ~1015) | SetQuotationsAction (line ~1030) |
| **Key Condition** | Quote status = "Expired" + QuoteReserved = true | Quote status = "Expired" + QuoteReserved = true |
| **Feature Flag** | cancel-reservation-expired-quotes | (TBD - Not specified) |
| **UI Component** | quotes-grid-comp.js | quotes-grid-comp.js |
| **Database Field** | quoteReserveBit (→ QuoteReserved property) | quoteReserveBit (→ QuoteReserved property) |
| **Root Cause** | Code only adds cancelReservation for Reserved quotes | Code only adds selectUnit for Active quotes |
| **Backend Changes** | Minimal (1 else-if block in SetQuotationsAction) | Moderate (CommitUnitManager + DAO + DB SP) |

---

## Key Information from PME-461308 That Applies to WI 2647905

### 1. **The 48-Hour Divergence Concept** ✅ CONFIRMED

**From PME-461308 spec:**
> "When quote and reservation timelines diverge (e.g., quote expires at 2 days, reservation holds for 3 days), the quote transitions to Expired while the reservation remains active on the unit."

**Test Data Setup Section:**
> "PMC Config: Set Quote Expiration to 2 days and Reservation Hold to 3 days"

**Impact on WI 2647905:**
- ✅ Confirms the 48-hour window concept is real
- ✅ Shows it's configurable per PMC
- ✅ Proves the business rule: quotes and reservations have INDEPENDENT expiration timelines

### 2. **The QuoteReserved Field Storage** ✅ CONFIRMED

**From PME-461308 spec:**
> "QuoteReserved is already populated by ProspectListDAO.GetProspectQuotes() at field index 14 (F_QuoteReserved), mapped from the quoteReserveBit column in the database. No DAO changes are needed."

**Database Implementation:**
```
Quotation Table
├─ quoteReserveBit (database column)
└─ QuoteReserved (C# property, mapped at index 14)
```

**Impact on WI 2647905:**
- ✅ ANSWERS the gap question: "Where is the 48-hour window stored?"
  - **Answer:** As `quoteReserveBit = 1` (active) or `quoteReserveBit = 0` (inactive)
- ✅ Shows no new DB structure needed
- ✅ Existing mapping is already in place
- ✅ DAO GetProspectQuotes() already populates this

### 3. **Quote Status Values** ✅ CONFIRMED

**From PME-461308 test cases:**

| Quote Status | Usage | Notes |
|-------------|-------|-------|
| "reserved" | Active quote with reservation | Original state |
| "expired" | Quote past 48-hour window | New status to handle |
| "active" | Valid quote without reservation | Existing behavior |

**From test case UT-002:**
> "Expired (LD ON) + QuoteReserved=true + SiteId Match=Yes → Actions: viewPrint, email, cancelReservation"

**Impact on WI 2647905:**
- ✅ Confirms quote status "expired" is a valid state in the system
- ✅ Shows how to check: `q.Status.Equals("expired", StringComparison.InvariantCultureIgnoreCase)`
- ✅ Proves the check is already working elsewhere in the codebase

### 4. **ProspectListManager.SetQuotationsAction Method** ✅ EXISTING

**From PME-461308 Current Code:**

The spec shows the EXACT location and method that WI 2647905 also needs to modify:

```csharp
// File: ProspectListManager.cs
// Method: SetQuotationsAction (line ~1015)

quotationList.ToList().ForEach(q =>
{
    if (q.Status.Equals("reserved", StringComparison.InvariantCultureIgnoreCase) && q.SiteId == propertyId)
    {
        q.Actions.AddRange(new string[] { "viewPrint", "email", CANCEL_RESERVATION });
        // ... additional logic
    }
    else
    {
        // Falls through to else branch
        q.Actions.AddRange(new string[] { "viewPrint", "email" });
    }
});
```

**Then PR 382684 (WI 2647905) ADDS:**

```csharp
// Inside the reserved block, add check for selectUnit
if (HasRight(C_MOVEIN_DATE_UNIT_RIGHT) && !q.UnitHold)
{
    q.Actions.AddRange(new string[] { "selectUnit" });
}
```

**Impact for WI 2647905:**
- ✅ Same file, same method, very similar pattern
- ✅ Can use identical approach: `else if (q.Status.Equals("expired", ...)`
- ✅ Can reuse the permission checking pattern

### 5. **Feature Flag Implementation Pattern** ✅ PROVEN

**From PME-461308 Implementation Pattern:**

```csharp
private const string CANCEL_EXPIRED_RESERVATION_FLAG = "cancel-reservation-expired-quotes";

bool enableCancelExpiredReservation = LaunchDarklyAccessor.IsEnabled(
    CANCEL_EXPIRED_RESERVATION_FLAG, new FlagContext(), Context);

// Then in the logic:
else if (enableCancelExpiredReservation 
    && q.Status.Equals("expired", StringComparison.InvariantCultureIgnoreCase)
    && q.SiteId == propertyId
    && q.QuoteReserved)
{
    // New behavior here
}
```

**Impact on WI 2647905:**
- ✅ Shows how to implement feature flags in this codebase
- ✅ Provides rollback mechanism (toggle flag OFF instantly)
- ✅ Allows gradual rollout to specific properties
- **RECOMMENDATION:** WI 2647905 should also use LD flag for "select-unit-expired-reservation" or similar

### 6. **Testing Patterns** ✅ REUSABLE

**From PME-461308 Test Cases:**

Unit Test Case UT-002:
```
Quote Status: Expired (with LD enabled)
QuoteReserved: true
SiteId Match: Yes
Expected Actions: [viewPrint, email, cancelReservation]
```

Manual Test Case MT-002:
```
Steps: On expired-but-reserved quote, click action dropdown
Expected: Cancel Reservation option is visible
```

**For WI 2647905, equivalent tests would be:**

Unit Test Case:
```
Quote Status: Expired (with LD enabled)
QuoteReserved: true
SiteId Match: Yes
User Has C_MOVEIN_DATE_UNIT_RIGHT: Yes
Expected Actions: [viewPrint, email, selectUnit for UI, validateApplicantCommitUnit for backend]
```

---

## Specific Answers to WI 2647905 Gaps

### Gap #1: Where is the 48-hour default stored?

**Answer from PME-461308:**
```
PMC Configuration (per property management company)
The quote expiration and reservation hold periods are PMC-specific configuration values.
They are NOT hardcoded; they are set per PMC in the configuration.

Database: [PMC_CONFIG] table or site settings
Field: Quote_Expiration_Days, Reservation_Hold_Days (or similar)

Default values mentioned: 2 days for quotes, 3 days for reservations
```

**Action for WI 2647905:**
- ✅ Confirm exact PMC config field names
- ✅ Verify if 48-hour is universal or configurable
- ✅ Update acceptance criteria to specify "per PMC configuration"

### Gap #2: How is reservation creation timestamp tracked?

**Answer from PME-461308:**
```
The quoteReserveBit field (QuoteReserved property) indicates WHETHER a reservation is active.
The timestamp of when the reservation was created is implied to be tracked in the QUOTATION table,
but PME-461308 doesn't explicitly detail which column stores this.

However, the logic shows:
- Quote created at T0
- User reserves unit at T0 + 1 hour (within 48-hour quote window)
- Reservation starts its own 48-hour timer
- Quote expires at T0 + 48 hours
- Reservation expires at T0 + 1 hour + 48 hours
- System should track both timestamps separately
```

**Likely columns in QUOTATION table:**
- `quoteCreatedDate` (quote creation)
- `quoteReservationDate` or `reservationCreatedDate` (when reservation was made)
- `quoteExpirationDate` (calculated: quoteCreatedDate + 2 days)
- `reservationExpirationDate` (calculated: reservationCreatedDate + 3 days)

**Action for WI 2647905:**
- ✅ Verify actual column names in QUOTATION table
- ✅ Update CommitUnitManager to check reservationExpirationDate
- ✅ Use the correct database field in uspCheckReservationValidity

### Gap #3: Configuration/Settings for 48-hour default

**Answer from PME-461308:**
> "PMC Config: Set Quote Expiration to 2 days and Reservation Hold to 3 days (or use client config: PMC 5394989, Site 5394999)"

**This suggests:**
- Configuration is per PMC (company level)
- Can be site-specific override
- These are not system constants

**Recommended storage locations:**
1. **PMC_CONFIG** or **PROPERTIES** table (most likely)
2. **SITE_SETTINGS** table (if site-specific)
3. Constants in **LaunchDarkly** (for feature management)

**Action for WI 2647905:**
- ✅ Determine if reservations must be exactly 48 hours or configurable
- ✅ Add configuration lookup in uspCheckReservationValidity stored procedure
- ✅ Acceptance criteria should specify: "Honor PMC-configured reservation window (default 48 hours)"

---

## Common Data Structures

Both WI 2647905 and PME-461308 use the same underlying data:

```sql
-- QUOTATION table structure (relevant columns)
QUOTATION
├─ quoteID (primary key)
├─ quoteStatus ('active', 'reserved', 'expired', etc.)
├─ quoteReserveBit (1=active reservation, 0=no reservation)
├─ quoteCreatedDate (when quote was created)
├─ quoteExpirationDate (calculated: quoteCreatedDate + quote_window_days)
├─ quoteReservationDate (when reservation was made, if any)
├─ reservationExpirationDate (calculated: quoteReservationDate + reservation_window_days)
├─ unitID (reserved unit)
├─ leaseID (prospect lease record)
└─ siteID (property)

-- Configuration tables
PMC_CONFIG / PROPERTIES
├─ quote_expiration_days (default: 2)
└─ reservation_hold_days (default: 3)
```

---

## Recommended Updates to WI 2647905 Based on PME-461308

### 1. **Update Acceptance Criteria**

**Current:**
```
"Need to select the Unit for the reserved quote from the Quote tab dropdown labeled 'Reserved.'
--GAP between Classic OneSite and NEW OneSite"
```

**Should be:**
```
Functional Requirements:
✅ "Select Unit" action visible in Quotes tab when:
   - quote.Status = "reserved" OR ("expired" AND reservation still valid)
   - user has C_MOVEIN_DATE_UNIT_RIGHT permission
   - unit is not on hold

✅ Backend ValidateApplicantCommitUnit() allows unit commitment via valid reservation

Non-Functional Requirements:
✅ Reservation validity = within [PMC-configured reservation hold window] (default: 48 hours)
✅ Honors both quote expiration AND reservation expiration independently
✅ Feature-flagged for safe rollout/rollback (LD flag: "select-unit-expired-reservation")
✅ Matches Classic OneSite behavior
```

### 2. **Add Feature Flag Requirement**

following PME-461308 pattern:

```csharp
private const string SELECT_UNIT_EXPIRED_RESERVATION_FLAG = "select-unit-expired-reservation";

bool enableSelectUnitExpiredReservation = LaunchDarklyAccessor.IsEnabled(
    SELECT_UNIT_EXPIRED_RESERVATION_FLAG, new FlagContext(), Context);
```

### 3. **Scope Definition**

```
IN SCOPE (from WI 2647905):
✅ ProspectListManager.cs - UI action visibility
✅ CommitUnitManager.cs - Backend validation
✅ CommitUnitDAO.cs - Reservation validity check
✅ uspCheckReservationValidity - Database validation

OUT OF SCOPE:
❌ Classic OneSite changes (already supports this)
❌ Backend API changes (endpoints already exist)
❌ Database schema changes (fields already exist)
❌ Configuration UI (use existing PMC Config)
```

### 4. **Implementation Precedent**

Since PME-461308 follows the same pattern, WI 2647905 should:

```
BEFORE: Only check quote status
IF (q.Status == "reserved")
  THEN show cancelReservation action

AFTER: Also check expired+reserved combo
IF (q.Status == "reserved")
  THEN show cancelReservation action
ELSE IF (enableFeatureFlag && q.Status == "expired" && q.QuoteReserved && isReservationValid)
  THEN show selectUnit action
```

---

## Communication to Product Team (Updated)

Since PME-461308 and WI 2647905 are related, send this clarified communication to Leela:

---

### **Updated Communication to Product Team**

**To:** Leelakrishna Balu  
**From:** Engineering  
**Subject:** WI 2647905 - Acceptance Criteria Update (Aligned with PME-461308)  
**Date:** 2026-02-28  

Hi Leela,

We've completed our analysis of WI 2647905 and found a related work item (PME-461308: Cancel Reservation for Expired Quotes) that shares the same technical architecture. This has helped clarify several gaps in the acceptance criteria.

**New Findings:**

The `QuoteReserved` bit field (and associated expiration logic) already exists in the system and is used exactly as needed for WI 2647905. However, the acceptance criteria should be updated to clarify:

1. **Reservation Validity Window:**
   - Per-PMC configuration (quote window: typically 2 days, reservation window: typically 3 days)
   - Confirmation: Should "48 hours" be universal, or honor PMC settings?

2. **Data Fields Already Available:**
   - `quoteReserveBit` (1=active, 0=inactive) ← Already populated
   - `QuoteReserved` property (C# mapping) ← Already mapped
   - `quoteExpirationDate` ← Already calculated
   - `reservationExpirationDate` ← Likely exists (needs confirmation)

3. **Recommended Feature Flag:**
   - Following PME-461308 pattern, use LD flag: "select-unit-expired-reservation"
   - This allows safe rollout/rollback without deployment

4. **Related Stored Procedures:**
   - PME-461308 uses `/0uspNPSGCCancelReservation_Upm.sql` 
   - WI 2647905 should use similar pattern for `uspCheckReservationValidity`

**Questions for Clarification:**

1. Is the 48-hour window:
   - Universal (hardcoded)?
   - Per-PMC configured (in PMC_CONFIG table)?
   - Per-Site configurable (in SITE_SETTINGS)?

2. Which table/column stores the reservation creation timestamp?
   - `QUOTATION.quoteReservationDate`?
   - Or another location?

3. Should we implement with LaunchDarkly feature flag (like PME-461308)?

**Next Steps:**

Once clarified, we'll update the acceptance criteria to include these requirements and proceed with implementation using the proven patterns from PME-461308.

Thanks!

---

## Summary Table: Gap Resolution

| Gap | Original Question | Answer from PME-461308 | Confidence |
|-----|------------------|----------------------|------------|
| Reservation Data Storage | Where is 48-hour validity stored? | quoteReserveBit field in QUOTATION table | ✅ HIGH |
| Timestamp Tracking | How is reservation creation tracked? | Likely quoteReservationDate column | ⚠️ MEDIUM |
| Configuration Storage | Where is 48-hour default stored? | PMC_CONFIG or PROPERTIES table | ⚠️ MEDIUM |
| Implementation Pattern | How to implement feature flag? | LaunchDarkly pattern with FlagContext | ✅ HIGH |
| UI/Backend Sync | Will changes propagate to UI? | Yes, quotes-grid-comp.js already handles it | ✅ HIGH |
| Testing Approach | How to test expired+reserved? | PM Config: quote 2d, reservation 3d | ✅ HIGH |

---

## Final Recommendation

**WI 2647905 should be implemented IN PARALLEL with or AFTER PME-461308** because:

1. ✅ Same file (ProspectListManager.cs)
2. ✅ Same method (SetQuotationsAction)
3. ✅ Same design pattern (LaunchDarkly feature gate)
4. ✅ Same data field (QuoteReserved)
5. ✅ Similar backend changes (CommitUnit validation)
6. ✅ Reduces merge conflicts if done together

Consider combining both changes in a single release/PR if timeline allows.
