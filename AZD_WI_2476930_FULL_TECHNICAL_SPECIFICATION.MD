# Full Technical Specification — Feature 2476930
# PME-451065: Minimum Rental History / Minimum Employment History Enforcement

- **Feature:** [2476930](https://tfs.realpage.com/tfs/Realpage/PropertyManagement/_workitems/edit/2476930)
- **Version:** 1.0 (Draft)
- **Date:** 2026-03-13
- **Author:** Engineering (consolidated from domain specs + codebase verification)
- **Sources:**
  - [API Spec (wiki)](https://newwiki.realpage.com/display/PM/API+-+Multiple+previous+rental+and+employment+history) — by Maurice Gollayan, last modified 2026-03-02
  - [UI Spec (wiki)](https://newwiki.realpage.com/display/PM/UI+-+Multiple+previous+rental+and+employment+history) — by JayAlvin Co, last modified 2026-03-04
  - Codebase verification — traced and verified against `Development` branch, 2026-03-13
- **Status:** DRAFT — Pending team review

---

## Table of Contents

1. [Purpose](#1-purpose)
2. [Scope](#2-scope)
3. [Feature Flag](#3-feature-flag)
4. [Dependencies & Risks](#4-dependencies--risks)
5. [Terminology](#5-terminology)
6. [Configuration Source](#6-configuration-source)
7. [Duration Aggregation Rules](#7-duration-aggregation-rules)
8. [Affected Workflows](#8-affected-workflows)
9. [Prospects Domain — API Changes](#9-prospects-domain--api-changes)
10. [Leased Accounts Domain — API Changes](#10-leased-accounts-domain--api-changes)
11. [Waitlist Domain — API Changes](#11-waitlist-domain--api-changes)
12. [New OneSite Select Unit — API Changes](#12-new-onesite-select-unit--api-changes)
13. [Leased Accounts — UI Changes](#13-leased-accounts--ui-changes)
14. [Prospects — UI Changes](#14-prospects--ui-changes)
15. [MergeField Support](#15-mergefield-support)
16. [Acceptance Criteria](#16-acceptance-criteria)
17. [LOE Summary](#17-loe-summary)
18. [Open Questions](#18-open-questions)
19. [Spec Errata & Codebase Corrections](#19-spec-errata--codebase-corrections)

---

## 1. Purpose

Enable **multiple previous rental and employment history entries** across all OneSite household member workflows (Prospects, Leased Accounts, Waitlist, Applicants).

Currently, the system assumes a **fixed two-record model** (current + one previous) for both rental history and employment history. This feature refactors the data model, APIs, validation, and UI to support **N records** so that applicants can satisfy company-configured minimum duration requirements.

Minimum history enforcement is based on the **aggregated duration** of current, previous, and additional previous records.

---

## 2. Scope

### In Scope
- Prospect flows (load, save, validate rental + employment history)
- Leased account flows (load, save, validate rental + employment history)
- Waitlist flows (required fields validation)
- New OneSite Select Unit flow (ContactsManager multi-node XML)
- Required Fields validation framework updates
- MergeField repeater support for multi-history records
- UI repeater/list rendering for Add/Edit Household Members

### Out of Scope
- Classic OneSite
- UI behavior outside validation status display
- AOLR / Apply Now flows (covered by Feature 2741850)
- Styling/theme redesign unrelated to repeater support

### Limitations
- Classic OneSite is out-of-scope
- Prospect → Waitlist workflow is not present in New OneSite UI
- Feature behavior may be unobservable if Mandatory Fields remains disabled via LaunchDarkly

---

## 3. Feature Flag

| Property | Value |
|---|---|
| **Flag name** | `accounts-employment-and-rental-length` |
| **Description** | When enabled, validation against the total length of employment and total length of residence is enforced |
| **When disabled** | Existing single-history validation behavior and fixed two-entry UI remain unchanged |
| **UI activity key** | To be added to `activity-name.js` (does not currently exist) |

### Dependency Flag
| Property | Value |
|---|---|
| **Flag name** | `addEditHouseholdMemberMandatoryFieldsCheckEnabled` (`HHM_MANDATORY_FIELDS_CHECK`) |
| **Relationship** | This feature depends on the Mandatory Fields framework; if that flag is disabled, this feature's validation behavior is also inactive |
| **Current state** | Mandatory Fields flag is disabled for all customers as of 2026-03-13 |

---

## 4. Dependencies & Risks

### Dependencies
1. **Mandatory Fields validation framework** — controlled by LaunchDarkly flag `2406950_Add_Edit_Household_Member_Mandatory_Fields_Check_Enabled`. This feature's validation behavior is inactive when that flag is disabled.
2. **Bug 2621966** (PME-481270) — Required Fields Validation section is dependent on this bug's completion. "New Experience Issue – Prospect Required Fields show as Incomplete."
3. **Feature 2741850** — AOLR Application: Support More Than 2 Records (alignment needed, separate scope).

### Risks
1. **LaunchDarkly gating** — Mandatory Fields may remain disabled, preventing end-to-end verification.
2. **High-risk file** — `household-member-manage-ctrl.js` is a frequent merge-conflict hotspot and regression point.
3. **Cross-team coordination** — Multiple features touch the same validation/save pipelines.
4. **Repeater key backward compatibility** — Indexed `_<index>` suffix keys must coexist with non-indexed field keys.
5. **Bulk save partial failure** — PUT endpoints can partially succeed; row-level error handling is required.

---

## 5. Terminology

| Term | Definition |
|---|---|
| **Primary previous residence** | Previous residence record with `sDisplay = 1` (or `rthDisplayBit = 1`) |
| **Other previous residence** | Additional previous residence records with `sDisplay = 0` (or `rthDisplayBit = 0`) |
| **Current employment** | Employment record with `RecurrentBit = 1` (or `reicurrentbit = 1`) |
| **Previous employment** | Employment records with `RecurrentBit = 0` (or `reicurrentbit = 0`) |
| **Residence identity key** | `rthid` — insert/update discriminator for rental history save |
| **Employment identity key** | `reiid` — insert/update discriminator for employment history save |
| **Repeater** | List-based rendering of homogeneous form rows (Angular `ng-repeat` pattern) |
| **Save-progress XML** | Prospects-domain inflight storage mechanism using XML nodes in Site DB |

---

## 6. Configuration Source

Minimum duration (in months) is configured at:
> **New OneSite → My Company → Company Details → Applicants → Applicants History**

The minimum is **not always 12 months** — it is **X months** (company-level setting). The applicant must be able to enter **as many records as needed** to meet the configured minimum.

The configured value is accessed via `IDemographic` settings:
- `demographicSettings.RentalHistoryMinimumDays` — for rental history
- Employment equivalent — for employment history

---

## 7. Duration Aggregation Rules

### Current Behavior (pre-feature)
- Validates **only** the current residence/employment record duration against minimum
- `IsRentalHistoryDurationSufficient` checks `daysSinceMoveIn >= RentalHistoryMinimumDays` for **single** record
- No aggregation across multiple previous records

### Target Behavior (post-feature)
- **Rental history**: `diffDays = (current residence days) + SUM(all previous residence days)`
- **Employment history**: `diffDays = SUM(all previous employment days)`
- If aggregated total < configured minimum: validation fails with aggregate error
- Individual record validation still applies (required fields per record)

### Open Questions on Duration Calculation
- **Overlaps/gaps**: Are overlapping date ranges deduplicated or simply summed?
- **Partial months**: Exact day-count conversion vs calendar month rounding?
- **Open-ended current record**: End date = today for duration calc?

---

## 8. Affected Workflows

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AFFECTED WORKFLOWS MATRIX                        │
├─────────────────┬──────────┬──────────┬──────────┬─────────────────┤
│ Change Type     │ Prospects│ Leased   │ Waitlist  │ Select Unit    │
│                 │          │ Accounts │          │ (Classic NOS)   │
├─────────────────┼──────────┼──────────┼──────────┼─────────────────┤
│ Load API        │    ✓     │  reuse   │   n/a    │      ✓         │
│ Save API        │    ✓     │    ✓     │   n/a    │      ✓         │
│ Validation      │    ✓     │    ✓     │    ✓     │     n/a        │
│ UI Changes      │    ✓     │    ✓     │   n/a    │     n/a        │
│ Required Fields │    ✓     │   n/a    │    ✓     │     n/a        │
│ MergeFields     │   n/a    │    ✓     │   n/a    │     n/a        │
└─────────────────┴──────────┴──────────┴──────────┴─────────────────┘
```

---

## 9. Prospects Domain — API Changes

### 9.1 Prospects Load — Rental History

**Endpoint:** `GET /api/prospects/v1/rentalhistory/company/{companyId}/property/{propertyId}?customerId={}&prospectId={}`

**Controller:** `ProspectsController.GetProspectRentalHistory` (L627)
```
ONESITE WEB/RP/RealPage/OneSite/Prospects/Realpage.OneSite.Prospects/Controllers/ProspectsController.cs
```

**Manager:** `ProspectListManager.GetCustomerRentalAddress(gCid, customerId)` (L2551)
```
ONESITE WEB/RP/RealPage/OneSite/Prospects/Realpage.OneSite.Prospects/BusinessLogic/Class/ProspectListManager.cs
```

**Current behavior:**
1. Site DB → Load save-progress XML via `LoadSaveProgressByGuestCardId(gCid, "SP_RentalHistory")`
2. PMC DB → Load Customer/Address via `LoadRentalAddressByGuestId(customerId)` → `SELECT FROM Customer C LEFT JOIN CustomerAddress CA WHERE C.custID = @CustomerID`
3. XML XPath for previous: `SelectSingleNode("//CurrentRentalHistory/Row[@CustomerID='" + customerId + "' and @HistoryType='Previous']")` at **L2698**
4. Merges 16+ override fields from XML into address object
5. Returns `ArrayList [currentRentalList, previousRentalHistory]`

**Required change:** Modify `SelectSingleNode` at L2698 to `SelectNodes` — iterate and build list of all previous rental history nodes instead of selecting only the first matching node.

**Async variant:** `GetCustomerRentalAddressAsync` (L9558) — called by `CheckHouseholdMemberCompleteApplication` (L7500) for validation. Same `SelectSingleNode` at L9444 needs the same multi-node refactor.

**DAO:** `ProspectListDAO.LoadRentalAddressByGuestId(customerId)` (L2185) via `ProspectListDB.SelectAddressQuery` (L665)
```
SELECT AddressID, TypeID, CustomerID, Address1, Address2, City, StateCode, Zip,
       CountryCode, RentOrOwnFlag
FROM Customer C LEFT JOIN CustomerAddress CA ON CA.caddrID = C.caddrHomeId
WHERE C.custID = @CustomerID
```

**Models:**
- `IRentalHistoryAddress.cs` / `RentalHistoryAddress.cs` (Prospects BusinessObject)
- Base 10 fields + 16 override fields from save-progress XML

### 9.2 Prospects Load — Employment History

**Endpoint:** `GET /api/prospects/v1/employment/company/{companyId}/property/{propertyId}?customerId={}&prospectId={}`

**Manager:** `ProspectListManager.GetCustomerEmployment(gCid, customerId)`

**Current behavior:** `SelectSingleNode("//CustomerEmployment/Row[@CustomerID='" + customerId + "' and @CurrentEmploymentFlag='0']")` at **L3001**

**Required change:** Same `SelectSingleNode` → `SelectNodes` pattern. Mirror #9.1 for employment nodes.

### 9.3 Prospects Save — Rental History Validation

**Endpoint:** `PUT /api/prospects/v1/company/{companyId}/property/{propertyId}/household/member`

**Validation chain (in ProspectListManager.cs):**
```
SaveHouseholdMember
 └─ ValidateHouseHoldMemberData (L10302)
     └─ ValidateRentalHistory (L10791)
         ├─ ValidateNotRequiredMoveIn (L10904)
         │   └─ ValidateMoveInDate (L10932)
         └─ ValidateMoveInDate (L10932)
```

**Required changes:**
1. `ValidateMoveInDate` (L10932): Refactor to iterate multiple previous rental addresses, aggregate `diffDays` across Previous Residence + SUM(Other Previous Residences)
2. Add indexed error messages: `AddApiError` with element position prefix
3. If aggregated length of stay is insufficient: `AddApiError("Insufficient total for length of stay"); return false;`

### 9.4 Prospects Save — Employment History Validation

**Validation chain (in ProspectListManager.cs):**
```
ValidateHouseHoldMemberData (L10306)
 └─ ValidateHouseHoldEmploymentHistory (L11069)
     ├─ ValidateNotRequiredStartDate (L11219)
     │   └─ ValidatePreviousStartDate (L11248)
     └─ ValidatePreviousStartDate (L11248)
```

**Required changes:** Mirror rental history validation — aggregate `diffDays = SUM(Previous Employment)`, indexed error messages.

### 9.5 Prospects Save — Rental History XML Data

**Method:** `SaveCustomerRentalAddress(trans, rentalAddress, rentalCheck)` (L3443)

**Current behavior:** The `else` block for `if (addr.HistoryType.ToLower() == "current")` handles Previous type. Node removal at L3560 uses `SelectSingleNode` and is inside the else block.

**Required changes:**
1. Move the node removal **before** the `if (addr.HistoryType == "current")` block: Remove all nodes under `//CurrentRentalHistory/Row[@CustomerID='...'` regardless of type, then rebuild
2. Remove duplicate removal in the else block (no longer needed once moved before the condition)

### 9.6 Prospects Save — Employment History XML Data

**Method:** `SaveCustomerEmploymentProgress(trans, lstEmploymentHistory)` (L3965)

**Required changes:** Mirror #9.5 — move node removal before `if (custEmpl.CurrentEmploymentFlag == 1)` block. Remove duplicate removal in else block.

### 9.7 Prospects — Required Fields Validation

**Controller:** `ProspectsController` → `ProspectListManager.CheckHouseholdMembersCompleteApplicationStatusAsync` (L7308)

**Endpoint:** `GET /api/prospects/v{api-version}/company/{companyId}/property/{propertyId}/householdMembers`

**Required changes:** Update to validate multiple rental/employment history entries in completion status check. Uses `GetCustomerRentalAddressAsync` (L9558) which itself needs the multi-node refactor from #9.1.

---

## 10. Leased Accounts Domain — API Changes

### 10.1 Existing Load APIs (Reusable — No Changes Needed)

**Rental History:**
- `GET /api/accounts/PreviousResidence/rentalHistoryMulti?ResmID={resmId}&sDisplay=1` (primary previous)
- `GET /api/accounts/PreviousResidence/rentalHistoryMulti?ResmID={resmId}&sDisplay=0` (other previous)

**Controller:** `PreviousResidenceController.GetMaxRentalHistoryMulti` (L84)
```
ONESITE WEB/RP/api-src/OneSite/Accounts/RealPage.API.OneSite.Accounts/Controllers/PreviousResidenceController.cs
```
**Manager:** `PrevResidenceManager.GetMaxRentalHistoryMulti(ResmId, sDisplay)` (L76)
**DAO:** `PrevResidenceDAO.GetMaxRentalHistoryMulti` (L71) → `PrevResidenceDB.GetMaxRentalHistoryMulti` → `EXEC DBO.uspSelectMaxRentalHistoryMulti`
- Returns `IList<IRentalHistoryMulti>` with 24+ fields
- **Already multi-record capable** — no load changes needed

**Employment History:**
- `GET /api/accounts/Employment/residentEmployer?RecurrentBit=1&ResmId={resmId}` (current)
- `GET /api/accounts/Employment/residentEmployer?RecurrentBit=0&ResmId={resmId}` (previous)

**Controller:** `EmploymentController` (L255)
```
ONESITE WEB/RP/api-src/OneSite/Accounts/RealPage.API.OneSite.Accounts/Controllers/EmploymentController.cs
```

### 10.2 Leased Accounts Save — Rental History (New PUT Endpoint)

**Current endpoint (to deprecate):** `POST /api/accounts/PreviousResidence/rentalHistoryMulti`
- Controller: `PreviousResidenceController.AddRentalHistoryMulti` (L139)
- UI currently calls this multiple times (once per record)

**New endpoint:**
- `PUT /api/accounts/PreviousResidence/rentalHistoryMulti`
- Accept array of addresses regardless of insert or update
- Insert/update determined by `rthid` property:
  - `rthid = null/0` → INSERT
  - `rthid > 0` → UPDATE
- Request body includes IDs for deletion
- **Delete is hard-delete** from `RentalHistory` table given `(leaseid, resmid, rthid)`
- Requires new controller action + manager method + DAO method

### 10.3 Leased Accounts Save — Employment History (New PUT Endpoint)

**Current endpoint (to deprecate):** `POST /api/accounts/Employment/employmentDetails`
- Controller: `EmploymentController` (L52-53)
- UI currently calls this multiple times

**New endpoint:**
- `PUT /api/accounts/Employment/employmentDetails`
- Accept array of employments regardless of insert or update
- Insert/update determined by `reiid` property
- Request body includes IDs for deletion
- **Delete is soft-delete** — update `ResidentEmployerInfo.reiDisplayBit = 0` given `(leaseid, resmid, reiid)`
- Requires new controller action + manager method + DAO method

### 10.4 Leased Accounts — Rental History Validation

> **⚠️ CODEBASE CORRECTION**: The wiki API spec references methods in `WaitingListWizardSettingManager.cs`. These methods have been **refactored** into separate validator classes via a factory pattern. The manager now delegates at L1401-1402.

**Endpoint:** `POST /api/settings/v1/company/{}/property/{}/relationship/H/fields/validate`

**Actual file (verified 2026-03-13):**
```
ONESITE WEB/RP/RealPage/OneSite/Settings/Realpage.OneSite.Settings/BusinessLogic/Class/Validators/PreviousResidenceValidator.cs
```

**NOT** in `WaitingListWizardSettingManager.cs` (wiki spec references the old location).

**Delegation path:**
```
WaitingListWizardSettingManager.cs (L1401-1402)
 └─ pageValidatorFactory.CreatePreviousResidenceValidator(validateRequestParameters)
     └─ PreviousResidenceValidator
         ├─ ValidateCurrentRentalHistory (L118)
         │   └─ ValidatePreviousRentalHistoryIfRequired (L161)
         │       └─ IsRentalHistoryDurationSufficient (L204)
         └─ ... per-field validators
```

**Current method signature:**
```csharp
private bool IsRentalHistoryDurationSufficient(
    RentalHistory currentRentalAddress,    // ← SINGLE record, not list
    IDemographic demographicSettings)
```
Checks: `daysSinceMoveIn >= demographicSettings.RentalHistoryMinimumDays`

**Required changes:**
1. Modify `IsRentalHistoryDurationSufficient` to accept the **whole list** of rental addresses
2. Aggregate `diffDays = (current residence days) + SUM(previous + other previous residence days)`
3. If aggregated length of stay is insufficient:
   - Introduce new constant `previousResidenceLengthOfStay` in `Field.cs`
   - Add `missingFields.Add(newConstant)`
4. Add underscore + sequence suffix for indexed error keys:
   - e.g., `missingFields.Add("otherPreResMoveIn_0")`, `missingFields.Add("otherPreResMoveIn_1")`
   - `_{value}` = element position of the errored record
5. Refactor to expect multiple previous rental addresses in `ValidatePreviousRentalHistoryIfRequired` (L161)
6. Instantiate new `List<string>` for per-record validation (do not pass main `missingFields` into sub-validators). After validation, iterate and add `_index` suffix, then consolidate into main list.

**New constant location:**
```
ONESITE WEB/RP/RealPage/OneSite/Settings/Realpage.OneSite.Settings/Constants/WaitingListWizardSetting/Field.cs
```

### 10.5 Leased Accounts — Employment History Validation

> **⚠️ CODEBASE CORRECTION**: Same refactoring as rental history.

**Actual file (verified 2026-03-13):**
```
ONESITE WEB/RP/RealPage/OneSite/Settings/Realpage.OneSite.Settings/BusinessLogic/Class/Validators/EmploymentHistoryValidator.cs
```

**Delegation path:**
```
WaitingListWizardSettingManager.cs (L1401)
 └─ pageValidatorFactory.CreateEmploymentHistoryValidator(validateRequestParameters)
     └─ EmploymentHistoryValidator
         ├─ ValidateCurrentStartDateAndHistory (L336)
         │   └─ ValidatePreviousEmploymentIfRequired (L411)
         └─ ... per-field validators
```

**Required changes:**
1. Aggregate `diffDays = SUM(Previous Employment)`
2. If insufficient: introduce new constant `employmentHistoryLengthOfEmployment` in `Field.cs`
3. Indexed error keys with `_<index>` suffix (same pattern as rental history)
4. Instantiate separate `List<string>` per record, add index suffix, then consolidate

---

## 11. Waitlist Domain — API Changes

### 11.1 Waitlist — Required Fields Validation

**Controller:** `WaitlistApplicantController`

**Endpoint:** `GET /api/waitlistaccounts/v1/WaitlistApplicant/company/{companyId}/property/{propertyId}/leaseID/{leaseID}/checkhouseholdmemberscompleteapplication`

**Manager:** `PeopleManager.CheckHouseholdMemberRequiredFieldStatusAsync` (L3198)
```
ONESITE WEB/RP/RealPage/OneSite/ResidentAccounts/Realpage.OneSite.ResidentAccounts/BusinessLogic/Class/PeopleManager.cs
```

**Current behavior:**
- `FillRentalHistory(RentalHistoryResult)` (L1918) — uses fixed properties `.CurrentResidence` + `.PreviousResidence` only (max 2 records)
- `FillEmploymentHistory(EmploymentHistoryResult)` (L1977) — same fixed 2-record pattern
- Comments state "zero, one, or two entries"

**Required changes:** Refactor `FillRentalHistory` and `FillEmploymentHistory` from fixed-property access to list iteration to support N records.

**Dependency:** Bug 2621966 (PME-481270) must be completed first.

---

## 12. New OneSite Select Unit — API Changes

### 12.1 ContactsManager — Multi-Node XML

**File:**
```
ONESITE WEB/Onesite/DotNet/RealPage/OneSite/LRC/Waitlist/BusinessLogic/ContactsManager.cs
```

**Method:** `GetCustomerRentalHistoryWithProgress` (L475)
- `SelectSingleNode("//CurrentRentalHistory/Row[@CustomerID='" + custid + "' and @HistoryType='Previous']")` at **L542**
- **Required change:** `SelectSingleNode` → `SelectNodes` + iterate and build list

**Method:** `GetCustomerEmploymentWithProgress` (L644)
- `SelectSingleNode("//CustomerEmployment/Row[@CustomerID='" + custid + "' and @CurrentEmploymentFlag='0']")` at **L709**
- **Required change:** Same multi-node pattern

### 12.2 WaitlistManager — No Changes Needed

**File:**
```
ONESITE WEB/Onesite/DotNet/RealPage/OneSite/LRC/Waitlist/BusinessLogic/WaitlistManager.cs
```

Already iterates via `foreach` loops at L769 (rental) and L955 (employment). No changes required — will automatically handle multiple records once `ContactsManager` provides them.

### 12.3 Legacy Save (LRCWebService.asmx.cs)

**Entry point:** `SaveWaitlistTC` in `LRCWebService.asmx.cs`
- Calls `ContactsManager.GetCustomerRentalHistoryWithProgress` and `GetCustomerEmploymentWithProgress`
- `WaitlistManager.SaveWaitlist` already loops through rental/employment history collections

**Note:** There is no Prospect → Waitlist workflow in New OneSite yet. The Select Unit flow uses the legacy LRC path.

---

## 13. Leased Accounts — UI Changes

### 13.1 Feature Flag Wiring

**File:** `ONESITE WEB/RP/ui-src/accounts/_app/js/config/activity-name.js`

**Current state:** Contains `HHM_MANDATORY_FIELDS_CHECK` and other activity keys. Does NOT include `accounts-employment-and-rental-length`.

**Change:** Add activity key for `accounts-employment-and-rental-length`. Gate repeater/list behavior and bulk endpoint usage behind this key. Preserve legacy fixed-two rendering when disabled.

### 13.2 Previous Residence — Data/Load/Save Layer

**Components:**
```
ONESITE WEB/RP/ui-src/accounts/_common/household-member/household-member-aside/js/controllers/household-member-manage-ctrl.js
ONESITE WEB/RP/ui-src/accounts/_common/household-member/previous-residence/js/services/previous-residence-svc.js
ONESITE WEB/RP/ui-src/accounts/_common/household-member/previous-residence/js/models/hh-member-previous-residence-model.js
```

**Current state:**
- Load orchestration in `household-member-manage-ctrl.js` makes 3 parallel `$resource` calls (L300-302): checks, otherPrevRes (`sDisplay=0`), prevRes (`sDisplay=1`)
- Maps at L378-380: `setRentalHistoryForm`, `setPreviousResidenceForm` (x2)
- Model stores fixed objects: `previousResidence` and `otherPreviousResidence`
- Save at L1011-1013: 3 separate save calls
- `previous-residence-svc.js` (L12): `svc.getPreviousResidence = $resource("/api/accounts/PreviousResidence/rentalHistoryMulti")`

**Changes:**
- Replace dual-call/fixed-object assumptions with list-based load and save mapping
- Add list-based fetch helpers normalizing API records into repeater-friendly model objects
- Add/replace bulk save helper to submit `upsertItems[]` + `deleteIds[]` when feature flag ON
- Reuse existing load APIs (no new load endpoint required)
- Keep existing service methods for legacy path

### 13.3 Employment — Data/Load/Save Layer

**Components:**
```
ONESITE WEB/RP/ui-src/accounts/_common/household-member/household-member-aside/js/controllers/household-member-manage-ctrl.js
ONESITE WEB/RP/ui-src/accounts/_common/household-member/employment/js/models/hh-member-employment-model.js
ONESITE WEB/RP/ui-src/accounts/_common/household-member/_base/js/services/household-member-manage-svc.js
```

**Current state:**
- Loads `RecurrentBit=1` and `RecurrentBit=0` separately
- Model stores fixed objects: `form.current` and `form.previous`
- Employment save handled by `household-member-manage-svc.js`: `svc.saveEmploment()` → POST `/api/accounts/Employment/employmentDetails`

**Changes:**
- Shift to list-based employment hydration and save
- Keep current-employment semantics, allow N previous employment entries
- Preserve legacy two-record path when feature flag OFF

### 13.4 Previous Residence Step UI

**Components:**
```
ONESITE WEB/RP/ui-src/accounts/_common/household-member/previous-residence/index.html
ONESITE WEB/RP/ui-src/accounts/_common/household-member/previous-residence/js/controllers/hh-member-previous-residence.js
ONESITE WEB/RP/ui-src/accounts/_common/household-member/previous-residence/js/models/hh-member-previous-residence-form-config.js
```

**Current state:**
- Template has explicit sections for `previousResidence` and `otherPreviousResidence`
- Date validation methods split by fixed sections
- Validation payload builder returns two items only
- Controller branches on `isProspectRentalHistory()` at L40, L179, L351

**Changes:**
- Convert fixed blocks to repeater/list rendering
- Refactor validators to accept index/item context
- Update payload builder to emit all entries in deterministic order

**Add/Delete UX:**
- Add button: bottom of previous residence list, labeled "Add Previous Residence"
- New row: empty editable, `id = null`, `sDisplay = 0`, marked unsaved
- Delete unsaved: remove from client list only
- Delete saved: remove from UI + append `rthid` to `deleteIds[]`
- Delete semantics: **hard delete** from RentalHistory table by `(leaseid, resmid, rthid)`
- No hard UI cap on max entries

### 13.5 Employment Step UI

**Components:**
```
ONESITE WEB/RP/ui-src/accounts/_common/household-member/employment/index.html
ONESITE WEB/RP/ui-src/accounts/_common/household-member/employment/js/controllers/hh-member-employment-ctrl.js
ONESITE WEB/RP/ui-src/accounts/_common/household-member/employment/js/models/hh-member-employment-form-config.js
```

**Current state:** Two hardcoded sections (current + previous). Controller/configs assume one previous employment.

**Changes:**
- Single current employment block + repeater/list for previous employment rows
- Refactor date/address/amount validation helpers for record collections

**Add/Delete UX:**
- Same pattern as residence. Uses `reiid` for delete tracking.
- Delete semantics: **soft delete** (`ResidentEmployerInfo.reiDisplayBit = 0`)

### 13.6 Save Orchestration

**Component:** `household-member-manage-ctrl.js`

**Current:** `onSaveSuccess` builds multiple independent save calls for fixed records.

**Change:**
- Replace fixed multi-call saves with one list-based request per domain:
  - Residence: single PUT with `upsertItems[]` + `deleteIds[]`
  - Employment: single PUT with `upsertItems[]` + `deleteIds[]`
- Use HTTP PUT for multi-record path when feature flag ON
- Preserve unrelated save calls (billing/custom fields)

### 13.7 Service Endpoint Contracts

**Component:** `household-member-manage-svc.js`

| Domain | Legacy (flag OFF) | New (flag ON) |
|---|---|---|
| Residence | `POST /api/accounts/PreviousResidence/rentalHistoryMulti` | `PUT /api/accounts/PreviousResidence/rentalHistoryMulti` |
| Employment | `POST /api/accounts/Employment/employmentDetails` | `PUT /api/accounts/Employment/employmentDetails` |

PUT requests accept: `{ upsertItems: [...], deleteIds: [...] }`
- Each item in `upsertItems` identified by `rthid`/`reiid` (null = insert, >0 = update)
- `deleteIds` triggers hard/soft delete per domain

### 13.8 Wizard Model Refactor

**Component:** `household-member-wizard-model.js`

**Current:** Discrete forms (`currentEmploymentForm`, `previousEmploymentForm`, `previousResidenceForm`, `otherPreviousResidenceForm`).

**Change:** Introduce collection-based state. Provide compatibility adapters for existing consumers.

### 13.9 Required Fields Validation UI

**Components:**
```
ONESITE WEB/RP/ui-src/accounts/_common/household-member/_base/js/services/live-form-validator-svc.js
ONESITE WEB/RP/ui-src/accounts/_common/household-member/_base/js/directives/form-validator-dir.js
ONESITE WEB/RP/ui-src/accounts/_common/household-member/previous-residence/js/models/hh-member-previous-residence-model.js
ONESITE WEB/RP/ui-src/accounts/_common/household-member/employment/js/models/hh-member-employment-model.js
```

**Changes:**
- Submit full array payloads for rental/employment validation
- Map indexed missing-field keys from API (`_<index>` suffix) to repeater rows
- Preserve non-indexed key handling for backward compatibility
- Support new aggregate insufficiency keys: `previousResidenceLengthOfStay`, `employmentHistoryLengthOfEmployment`

---

## 14. Prospects — UI Changes

### 14.1 Prospects Load UI

**Components:**
```
ONESITE WEB/RP/ui-src/accounts/_common/household-member/previous-residence/js/services/prospect-svc.js
ONESITE WEB/RP/ui-src/accounts/_common/prospect-rental-history/prospect-rental-history.js
ONESITE WEB/RP/ui-src/accounts/_common/prospect-household-list/prospect-household-list-comp.js
```

**Current state:**
- `prospect-svc.js` (L37): `svc.prospectRentalHistory = $resource(baseUrl + "/rentalhistory/company/:companyId/property/:propertyId")`
- `prospect-rental-history.js` (L38): calls `prospectSvc.prospectRentalHistory.get(params)`
  - Response mapping (L76-82): `resp.model[0][0]` → previousResidence, `resp.model[0][1]` → otherPreviousResidence, `resp.model[1][0]` → rentalHistoryChecks
  - Sets `previousResidenceModel.setProspectRentalHistoryFlag(true)` (L26)
- `prospect-household-list-comp.js` (L123): `buildRentalAdrress()` — builds fixed `[prevRes, otherPrevRes]` array — **KEY constraint for multi-record feature**

**Changes:**
- Refactor `prospect-rental-history.js` response mapping from fixed 2-element arrays to dynamic list
- Refactor `buildRentalAdrress()` in `prospect-household-list-comp.js` to support N entries
- Update `hh-member-previous-residence-model.js` prospect path:
  - `setProspectPreviousResidenceForm(formData, property)` (L200) — extend for list
  - `buildProspectResData(property)` (L373) — build save payload for N entries

### 14.2 Shared Model (Both Domains)

**Component:** `hh-member-previous-residence-model.js`

Key integration points:
- `prospectRentalHistory` flag (L34) — determines domain branching
- `isProspectRentalHistory()` (L438) — branch predicate
- `setProspectPreviousResidenceForm()` (L200) — Prospect setter (16 fields)
- `setPropspectRentalHistoryForm()` (L224) — eviction/sued/property/brokenLease flags
- `buildProspectResData()` (L373) — Prospect save payload
- `setPreviousResidenceForm()` — Accounts setter
- `getRentalHistoryFormData()` / `getPrevResFormData()` / `getOtherPrevResFormData()` — Accounts save payloads

**Controller:** `hh-member-previous-residence.js` — branches on `isProspectRentalHistory()` at L40, L179, L351. All branch points need list-aware refactoring.

---

## 15. MergeField Support

### 15.1 Current State (Verified)

**Rental History mergefields:**
- Only "Previous Residence" has mergefields (which maps to current residence during prospect stage)
- "Other Previous Residence" data does NOT appear in any current mergefields
- No existing mergefields need modification

**Employment mergefields:**
- Only "Current Employment" has mergefields
- No previous employment mergefields exist

### 15.2 Required Changes — Repeater MergeFields

If repeater mergefields are required, create using SQL insert scripts.

**Reference script:** `DATABASESETUP/100/static/ServerManager/152166_RentableNTVDate_MergeField.sql`

**Database queries for verification:**
```sql
-- Rental history
SELECT * FROM mergefield WHERE name LIKE '%previous%'
SELECT * FROM mergefield WHERE name LIKE '%residence%'

-- Employment
SELECT * FROM mergefield WHERE name LIKE '%employment%'
SELECT * FROM MergeFieldDetail WHERE helptext LIKE '%employment%'
```

**Data sources for repeater mergefields:**

| Domain | Table | Filters | Order |
|---|---|---|---|
| Rental History | `RentalHistory` | `resmID`, `rthDisplayBit` (1=previous during leased/current during prospect, 0=other previous) | `rthMoveInDate DESC` |
| Employment | `ResidentEmployerInfo` | `resmID`, `reicurrentbit` (1=current, 0=previous) | `reiStartDate DESC` |

**Known placeholders:** `$leaid` = lease id, `$resmid` = resident member id

---

## 16. Acceptance Criteria

### 16.1 API Acceptance Criteria

1. **Multi-record load (Prospects):** Given a prospect with 3 previous rental addresses saved in XML, when GET rental history is called, then all 3 previous records are returned in the response array.
2. **Multi-record load (Employment):** Same as above for employment.
3. **Aggregated validation (Prospects):** Given combined rental history < configured minimum, when household member is saved, then API returns indexed error messages identifying insufficiency.
4. **Aggregated validation (Leased Accounts):** Given combined rental history < configured minimum, when fields/validate is called, then API returns `previousResidenceLengthOfStay` in missingFields plus indexed per-record errors.
5. **Bulk save (Leased Accounts Rental):** Given PUT with array of 4 addresses (1 update, 2 inserts, 1 delete), then records are correctly inserted/updated/hard-deleted.
6. **Bulk save (Leased Accounts Employment):** Same pattern — soft-delete via `reiDisplayBit = 0`.
7. **Required Fields (Prospects):** Given 3 previous rental entries, when household members completion check runs, then all entries are validated for required fields.
8. **Required Fields (Waitlist):** Given multiple records, when `CheckHouseholdMemberRequiredFieldStatus` runs via PeopleManager, then list-based fill methods handle N records.
9. **Select Unit (Classic):** Given multi-node XML for rental/employment, when ContactsManager loads history, then all nodes are parsed into list.
10. **Feature flag OFF:** Given flag disabled, then all existing single-history validation and two-record behavior is unchanged.

### 16.2 UI Acceptance Criteria

1. **Load (Residence):** Given 4 previous residences, when step loads with flag ON, then all 4 render with correct primary/other labeling.
2. **Load (Employment):** Given 1 current + 3 previous, when step loads, then current renders in current block and 3 previous in repeater.
3. **Add behavior:** Given flag ON, when user clicks Add, then new empty editable row appended and included in `upsertItems[]` on save.
4. **Delete unsaved:** Given unsaved row, when deleted, then removed immediately with no `deleteIds[]` entry.
5. **Delete saved:** Given persisted row, when confirmed delete, then row removed and ID added to `deleteIds[]`.
6. **Delete semantics:** Residence = hard delete, Employment = soft delete.
7. **Bulk save:** Given valid rows, then exactly one residence PUT and one employment PUT per save.
8. **Validation HTTP 200:** Given validation errors with HTTP 200, then response body is inspected and errors shown.
9. **Flag OFF backward compatibility:** Given flag OFF, then legacy two-entry UI and POST save methods used.
10. **Indexed validation mapping:** Given indexed field keys from API, then errors highlight matching repeater row controls.
11. **Regression safety:** Unrelated household fields remain unaffected.

---

## 17. LOE Summary

### 17.1 API LOE (from wiki spec, 2026-03-02)

| # | Activity | Dev (hrs) | Testing (hrs) |
|---|---|---:|---:|
| 1 | Prospects Load — Rental history | 6.5 | 6.5 |
| 2 | Prospects Load — Employment history | 6.5 | 6.5 |
| 3 | Prospects Save — Rental history validation | 13 | 6.5 |
| 4 | Prospects Save — Employment history validation | 13 | 6.5 |
| 5 | Prospects Save — Rental history xmldata | 6.5 | 6.5 |
| 6 | Prospects Save — Employment history xmldata | 6.5 | 6.5 |
| 7 | Prospects — People Required Fields | 6.5 | 6.5 |
| 8 | Leased accounts validation — Rental history | 13 | 6.5 |
| 9 | Leased accounts validation — Employment history | 13 | 6.5 |
| 10 | Leased accounts saving — Rental history | 13 | 6.5 |
| 11 | Leased accounts saving — Employment history | 13 | 6.5 |
| 12 | Waitlist — People Required Fields | 13 | 6.5 |
| 13 | New OneSite — Select unit | 13 | 13 |
| 14 | Each repeater mergefield | 3.25 | 3.25 |
| | **20% Contingency** | 18 | |
| | **Gross Total (excl mergefields)** | **136.5** | **91** |

### 17.2 UI LOE (from wiki spec, 2026-03-04)

| Phase | Activity | Dev (hrs) | QA (hrs) |
|---|---|---:|---:|
| Foundation | Feature flag wiring | 3.25 | 1.5 |
| Residence | Data layer refactor | 13.0 | 4.0 |
| Residence | Step UI refactor | 16.0 | 8.0 |
| Employment | Data layer refactor | 6.5 | 4.0 |
| Employment | Step UI refactor | 16.0 | 8.0 |
| Shared | Save orchestration update | 8.0 | 4.0 |
| Shared | Live validation mapping | 8.0 | 4.0 |
| Shared | Localization + regression | 4.0 | 2.0 |
| | **Gross Total** | **74.75** | **35.5** |

### 17.3 Combined Totals

| Category | Dev Hours | Test Hours |
|---|---:|---:|
| API | 136.5 | 91.0 |
| UI | 74.75 | 35.5 |
| **Grand Total** | **211.25** | **126.5** |

---

## 18. Open Questions

| # | Question | Context | Status |
|---|---|---|---|
| 1 | How should overlaps/gaps in date ranges be handled? | Duration aggregation: simple sum or deduplicated intervals? | Open |
| 2 | Partial months: day-count or calendar-month rounding? | Affects minimum threshold comparison | Open |
| 3 | Open-ended current record: end date = today? | For calculating current residence/employment duration | Open |
| 4 | Maximum record count cap? | AC says "as many as needed" but is there a practical DB/perf limit? | Open |
| 5 | LaunchDarkly test strategy? | Mandatory Fields flag is disabled — how to verify E2E? | Open |
| 6 | Prompting UX: on submit only vs progressive? | AC says "continue prompting" — clarify trigger | Open |
| 7 | AOLR alignment (Feature 2741850)? | Separate feature for AOLR "more than 2 records" — shared contracts? | Open |
| 8 | Repeater mergefields: which fields exactly? | LOE has per-field effort but no definitive list | Open |

---

## 19. Spec Errata & Codebase Corrections

These corrections were identified during codebase verification on 2026-03-13, comparing the wiki API spec against the current `Development` branch.

### 19.1 Stale File References (Critical)

The wiki API spec references leased accounts validation methods as being in `WaitingListWizardSettingManager.cs`. These have been **refactored** into separate validator classes:

| Wiki Says | Actual Location (Verified) |
|---|---|
| `WaitingListWizardSettingManager.cs` → `IsRentalHistoryDurationSufficient` | `BusinessLogic/Class/Validators/PreviousResidenceValidator.cs` (L204) |
| `WaitingListWizardSettingManager.cs` → `ValidatePreviousRentalHistoryIfRequired` | `BusinessLogic/Class/Validators/PreviousResidenceValidator.cs` (L161) |
| `WaitingListWizardSettingManager.cs` → `ValidateCurrentRentalHistory` | `BusinessLogic/Class/Validators/PreviousResidenceValidator.cs` (L118) |
| `WaitingListWizardSettingManager.cs` → `ValidatePreviousEmploymentIfRequired` | `BusinessLogic/Class/Validators/EmploymentHistoryValidator.cs` (L411) |
| `WaitingListWizardSettingManager.cs` → `ValidateCurrentStartDateAndHistory` | `BusinessLogic/Class/Validators/EmploymentHistoryValidator.cs` (L336) |

**Root cause:** Wiki links reference branch `GBDevelopment/Chargers/Mandatory_Fields_Validation_APIs` where validation was inside the manager. The current codebase uses a factory/validator pattern (delegates at L1401-1402).

### 19.2 Line Number Drift (Non-Critical)

All Prospects domain line numbers in the wiki spec show ~30–70 line drift from the current codebase due to code evolution since the wiki was written:

| Method | Wiki Line | Current Line |
|---|---|---|
| `SelectSingleNode` (Previous rental) | ~2631 | 2698 |
| `SelectSingleNode` (Previous employment) | ~2934 | 3001 |
| `SaveCustomerRentalAddress` | ~3419 | 3443 |
| `SaveCustomerEmploymentProgress` | ~3924 | 3965 |
| `ValidateMoveInDate` | ~10872 | 10932 |
| `ValidatePreviousStartDate` | ~11173 | 11248 |

### 19.3 Method Signature Discrepancy

`IsRentalHistoryDurationSufficient` currently takes a **single** `RentalHistory currentRentalAddress` (not a list). The wiki correctly identifies this needs to change to accept the full list, but the description could imply it already accepts a list.

### 19.4 ContactsManager Line Numbers (Exact Match)

The ContactsManager references in the wiki are still exact:
- `GetCustomerRentalHistoryWithProgress` at L475 ✓
- SelectSingleNode Previous rental at L542 ✓
- `GetCustomerEmploymentWithProgress` at L644 ✓
- SelectSingleNode Previous employment at L709 ✓

---

## Appendix A — File Inventory (All Affected Files)

### API / Backend Files

| File | Domain | Change Type |
|---|---|---|
| `Prospects/.../BusinessLogic/Class/ProspectListManager.cs` | Prospects | Load, Save, Validate |
| `Prospects/.../Controllers/ProspectsController.cs` | Prospects | Controller routes |
| `Prospects/.../DAO/Class/ProspectListDAO.cs` | Prospects | DAO (no change expected) |
| `Accounts/.../Controllers/PreviousResidenceController.cs` | Accounts | New PUT endpoint |
| `Accounts/.../Controllers/EmploymentController.cs` | Accounts | New PUT endpoint |
| `Accounts/.../BusinessLogic/Class/PrevResidenceManager.cs` | Accounts | New manager method |
| `Accounts/.../DAO/Class/PrevResidenceDAO.cs` | Accounts | New DAO method |
| `Settings/.../Validators/PreviousResidenceValidator.cs` | Settings | Multi-record validation |
| `Settings/.../Validators/EmploymentHistoryValidator.cs` | Settings | Multi-record validation |
| `Settings/.../Constants/WaitingListWizardSetting/Field.cs` | Settings | New constants |
| `ResidentAccounts/.../BusinessLogic/Class/PeopleManager.cs` | Waitlist | List-based fill |
| `LRC/Waitlist/.../ContactsManager.cs` | Select Unit | Multi-node XML |
| `LRC/Waitlist/.../WaitlistManager.cs` | Select Unit | No change (already loops) |

### UI Files

| File | Domain | Change Type |
|---|---|---|
| `accounts/_app/js/config/activity-name.js` | Shared | Feature flag key |
| `household-member-aside/.../household-member-manage-ctrl.js` | Shared | Save orchestration |
| `_base/.../household-member-manage-svc.js` | Shared | PUT endpoints |
| `_base/.../household-member-wizard-model.js` | Shared | Collection state |
| `_base/.../live-form-validator-svc.js` | Shared | Indexed keys |
| `_base/.../form-validator-dir.js` | Shared | Directive updates |
| `previous-residence/.../previous-residence-svc.js` | Residence | List helpers |
| `previous-residence/.../hh-member-previous-residence.js` | Residence | Controller refactor |
| `previous-residence/.../hh-member-previous-residence-model.js` | Shared | Model refactor |
| `previous-residence/.../hh-member-previous-residence-form-config.js` | Residence | Form config |
| `previous-residence/index.html` | Residence | Template repeater |
| `employment/.../hh-member-employment-ctrl.js` | Employment | Controller refactor |
| `employment/.../hh-member-employment-model.js` | Employment | Model refactor |
| `employment/.../hh-member-employment-form-config.js` | Employment | Form config |
| `employment/index.html` | Employment | Template repeater |
| `prospect-rental-history/prospect-rental-history.js` | Prospects | Response mapping |
| `prospect-household-list/prospect-household-list-comp.js` | Prospects | buildRentalAdrress |
| `prospect-svc.js` | Prospects | Service (no change expected) |
| `print-application-details-svc.js` | Accounts | Print (tertiary consumer) |

### Database Scripts

| Script | Purpose |
|---|---|
| MergeField insert scripts (TBD) | Repeater mergefields for rental/employment history |

---

## Appendix B — Rollback Strategy

1. **Feature flag OFF** → keeps legacy fixed-two rendering and legacy POST save methods active
2. New bulk PUT endpoints are backward-compatible with two-record submissions
3. Legacy save path ignores extra records created while flag was ON (or trims with warning log)
4. No destructive migration required
5. If rollback post-release:
   - Turn feature flag OFF
   - Continue reads from canonical backend source
   - Restrict writes to legacy path while retaining historical records server-side
