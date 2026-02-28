# Gap Analysis: WI 2647905 - 48-Hour Reservation Validity Storage

## Summary
There is a **significant gap** between the User Story acceptance criteria and the technical implementation plan regarding how the **48-hour reservation validity window** should be stored, tracked, and validated.

---

## Current Acceptance Criteria (from WI 2647905)

```
"Need to select the Unit for the reserved quote from the Quote tab dropdown 
labeled "Reserved."--GAP between Classic OneSite and NEW OneSite"
```

### What's Missing from Acceptance Criteria

The acceptance criteria focuses **ONLY** on the UI behavior (ability to see and select the "Select Unit" option) but makes **NO mention** of:

1. **How the 48-hour reservation window is stored** in the system
2. **Where reservation creation timestamps should be persisted**
3. **How to calculate/track reservation expiration**
4. **Database schema requirements** for reservation tracking
5. **Configuration or settings** for the 48-hour default value
6. **Validation rules** for what constitutes a "valid" reservation

---

## Technical Plan Requirements (from 2647905.plan.md)

The attached technical implementation plan **explicitly requires:**

### 1. Database Stored Procedure Creation
```sql
CREATE PROC [dbo].[uspCheckReservationValidity]
    @InternalEntityID int,
    @InternalUserID int,
    @InternalSiteID int,
    @IPVC_LEASEID int,
    @IPVC_UNITID int
AS
BEGIN
    -- Check if a reservation exists for the given lease/unit combination
    -- Validates that the reservation is within the 48-hour window
    -- Returns 1 if valid reservation exists, 0 otherwise
END
```

### 2. Validation Points Mentioned in Plan

From the "Variables to Inspect" section:
- **"Reservation expiration timestamp vs current time"** - Implies timestamp is stored
- **"Reservation status flags"** - Implies status tracking needed
- Checking "if reservation is within the 48-hour window" - Implies calculation needed

### 3. Data Requirements Not Specified

The technical plan assumes the following exist but **DOESN'T clarify where they come from:**

- ✓ Reservation creation timestamp - **WHERE IS THIS STORED?**
- ✓ Reservation validity duration (48 hours) - **WHERE IS THIS CONFIGURED?**
- ✓ Reservation status - **WHICH TABLE/COLUMN?**
- ✓ Lease/Unit/Reservation relationship - **ALREADY EXISTS OR NEEDS CREATION?**

---

## Database Schema Questions

To properly implement the `uspCheckReservationValidity` stored procedure, we need clarity on:

### Question 1: Does a "Reservation" Table Exist?
**Current Status:** ❓ UNCLEAR

If Yes:
- What columns does it have?
- Is there a `ReservationCreatedDate` or `ReservationTimestamp`?
- Is there a `ReservationExpiryDate` or calculated?
- How is it linked to Leases and Units?

If No:
- Do we need to create a RESERVATION table?
- Or store reservation data in existing tables (LEASE, UNIT, or QUOTATION)?
- How is "reserved" status currently tracked? (Currently appears to be only a Quote.Status = "reserved")

### Question 2: The 48-Hour Default Value
**Current Status:** ❓ UNCLEAR

Where should this be stored?
- **Option A:** Hardcoded in stored procedure (NOT RECOMMENDED - hard to change)
- **Option B:** Configuration table (RECOMMENDED - allows site-specific overrides)
- **Option C:** Site settings (RECOMMENDED - per-site customization)
- **Option D:** System constant (REASONABLE - but still hardcoded)

Current findings from code review:
- Description mentions: "The quote is good for 48 hours; and if the applicant reserves the unit within those 48 hours, the reservation starts another 48-hour time frame"
- This is a **BUSINESS RULE** that needs to be captured in acceptance criteria
- But **WHERE IS IT STORED?** - Not mentioned in acceptance criteria

### Question 3: Reservation vs Quote Expiration
**Current Status:** ❓ UNCLEAR

Is there a separate table/schema for:
- **Quote expiration** (already exists - Quote.Status = "active"/"expired")
- **Reservation expiration** (UNCLEAR - how is this tracked independently?)

When a Quote.Status = "reserved", is there an associated:
- Reservation record with its own expiration date?
- Or is expiration calculated from creation timestamp + 48 hours?

---

## Impact on Development

### Current Scenario (Without Clarity)
Developer (Leela) would need to:
1. ❓ Investigate whether RESERVATION data structure exists
2. ❓ Determine where 48-hour default is stored
3. ❓ Make assumptions about data model
4. ❓ Potentially spend time reworking if assumptions are wrong
5. 🚫 Accept criteria doesn't validate their implementation is correct

### Recommended Approach
Product team should clarify BEFORE development starts:

1. **Schema Requirement:** Confirm database structure for reservation tracking
2. **Configuration Requirement:** Where is the 48-hour default stored/configured?
3. **Business Rules:** Document all rules for "valid reservation"
4. **Acceptance Criteria Update:** Include these non-functional requirements

---

## Recommended Communication to Product Team (Leela)

---

### **Communication to Product Team**

**To:** Leelakrishna Balu (Product Owner, WI 2647905)  
**From:** Engineering  
**Subject:** WI 2647905 - Acceptance Criteria Update Needed  
**Date:** 2026-02-28

Hi Leela,

We reviewed WI 2647905 in preparation for implementation and identified a gap in the acceptance criteria that we need to clarify before development starts.

**The Issue:**

The current acceptance criteria focuses on UI functionality:
> "Need to select the Unit for the reserved quote from the Quote tab dropdown labeled 'Reserved.'"

However, the technical implementation plan requires a new stored procedure `uspCheckReservationValidity` that validates **whether a reservation is within the 48-hour validity window**. This validation depends on data that is not currently specified in the work item.

**Questions for Product:**

1. **Reservation Data Storage:**
   - Does a RESERVATION record exist in the database (linked to LEASE + UNIT)?
   - Or is reservation status only tracked as Quote.Status = "reserved"?
   - How is the reservation creation timestamp currently stored?

2. **48-Hour Validity Configuration:**
   - Where should the "48-hour" default validity period be stored?
   - Should this be site-configurable, or is it always 48 hours globally?
   - Should it be stored in a configuration table, system settings, or hardcoded?

3. **Reservation Expiration Calculation:**
   - Should expiration be: `ReservationCreatedDate + 48 hours`?
   - Or is there a separate `ReservationExpiryDate` field?
   - How should we handle time zones (UTC, site timezone)?

4. **Business Rules Clarity:**
   - "Valid reservation" = within 48 hours of creation? 
   - Or are there other conditions (e.g., applicant status, site policies)?
   - Should applicant status affect reservation validity?

**Recommended Acceptance Criteria Update:**

Once clarified, we should update the acceptance criteria to include:

```
UPDATED ACCEPTANCE CRITERIA:

Functional Requirements:
✅ "Select Unit" option visible under "Reserved" quotes in Prospects > Quotes tab
✅ Permission checks enforced (user must have C_MOVEIN_DATE_UNIT_RIGHT)
✅ Backend ValidateApplicantCommitUnit() allows unit commitment via valid reservation

Non-Functional Requirements:
☐ Reservation validity is checked against creation timestamp + 48-hour window
☐ 48-hour default value is stored in [LOCATION TO BE DETERMINED]
☐ Reservation status is tracked in [TABLE TO BE DETERMINED]
☐ System validates reservation before allowing unit selection
☐ Clear error messages if reservation has expired
```

**Next Steps:**

Once you clarify these points, we can:
1. Update acceptance criteria formally in the work item
2. Ensure all technical details are captured
3. Give developer clear implementation requirements
4. Avoid rework due to mismatched assumptions

Please let me know your thoughts on these questions, and we can schedule a brief sync if needed.

Thanks!

---

## Attachment: Technical Plan References

### From 2647905.plan.md - Lines about "48-hour window":

**In Reproduction Steps:**
> "Allow the quote to expire (after 48 hours) while keeping the reservation active"

**In Root Cause Analysis:**
> "Recognizes that reservation creates a new 48-hour window independent of quote expiration"

**In Variables to Inspect:**
> "Reservation expiration timestamp vs current time"

**In Database Stored Procedure Comment:**
> "Validates that the reservation is within the 48-hour window"
> "Returns 1 if valid reservation exists, 0 otherwise"

**In Testing Edge Cases:**
> "Boundary testing at 48-hour mark for both quote and reservation expiry"
> "Test exactly at 48-hour mark for both quote and reservation expiration"

---

## Summary Table

| Aspect | Acceptance Criteria | Technical Plan | Status |
|--------|-------------------|-----------------|--------|
| UI "Select Unit" visibility | ✅ Specified | ✅ Covered | **CLEAR** |
| Permission validation | ❌ Not specified | ✅ Covered | **PARTIAL** |
| Backend validation logic | ❌ Not specified | ✅ Covered | **PARTIAL** |
| Reservation table/schema | ❌ Not specified | ❓ Assumed | **UNCLEAR** |
| 48-hour default storage | ❌ Not specified | ❌ Not specified | **MISSING** |
| Reservation timestamp tracking | ❌ Not specified | ✅ Assumed | **UNCLEAR** |
| Configuration/settings | ❌ Not specified | ❌ Not specified | **MISSING** |

**Overall Status:** ⚠️ **PROCEED WITH CAUTION** - Missing critical non-functional requirements
