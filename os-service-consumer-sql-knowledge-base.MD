# os-service-consumer SQL Knowledge Base

This page is the canonical, team-shared SQL knowledge base for the `os-service-consumer` agent. It provides SQL query patterns for building valid API request payloads, verifying API responses against the database, and understanding OneSite data models.

**Agent:** `os-service-consumer` (see [Agent Catalog](https://newwiki.realpage.com/display/PM/Agent+Catalog))
**Last Updated:** 2026-03-14
**Default Environment:** SAT — Server: `RCAONECLNSQL017`, Database: `S4476658`, Company: `4341841`, Property: `4476658`

---

## How This Page Is Used

The `os-service-consumer` agent checks this page at the start of any session that needs SQL data:

1. Fetches this wiki page via `fetch_webpage`
2. Searches for SQL patterns matching the needed data (by domain keyword, table name, or field name)
3. If a pattern is found → executes the query to get real values for API payloads
4. If no pattern is found → prompts the user for a SQL query
5. New queries provided by users should be added here for team-wide reuse

---

## Table of Contents

1. [Prospects Domain](#prospects-domain)
2. [Leased Accounts Domain](#leased-accounts-domain)
3. [Waitlist Domain](#waitlist-domain)
4. [Move-In / Move-Out Domain](#move-in--move-out-domain)
5. [Common Lookups](#common-lookups)
6. [Field Definitions Reference](#field-definitions-reference)

---

## Prospects Domain

### Get Prospect/Guest Card by Customer ID

```sql
use S4476658
SELECT TOP 10 c.custID, c.custFirstName, c.custLastName, c.custEmail,
       c.custPhone1, c.custSSN, c.custBirthDate,
       ca.caAddr1, ca.caCity, ca.caState, ca.caZip
FROM Customer c
LEFT JOIN CustomerAddress ca ON ca.caddrID = c.caddrHomeId
WHERE c.custType = 'P'  -- P = Prospect
ORDER BY c.custID DESC
```
- **Use when:** Need a valid `customerId` for prospect API calls
- **Key columns:** `custID` → API `customerId`, `custFirstName`/`custLastName` → name fields

### Get Guest Card ID (Prospect ID)

```sql
use S4476658
SELECT TOP 10 g.gCID, g.gCCustomerID, g.gCDateofContact, g.gCStatus,
       c.custFirstName, c.custLastName
FROM GuestCard g
JOIN Customer c ON c.custID = g.gCCustomerID
ORDER BY g.gCID DESC
```
- **Use when:** Need a valid `prospectId` (gCID) for prospect load/save API calls
- **Key columns:** `gCID` → API `prospectId`, `gCCustomerID` → API `customerId`

### Get Rental History for a Customer

```sql
use S4476658
SELECT ca.caddrID, ca.caAddr1, ca.caAddr2, ca.caCity, ca.caState, ca.caZip,
       ca.caCountry, ca.caTypeID
FROM CustomerAddress ca
WHERE ca.caCustomerID = @CustomerID
```
- **Use when:** Verifying rental history data returned by the rental history GET endpoint
- **Related endpoint:** `GET /api/prospects/v1/rentalhistory/company/{companyId}/property/{propertyId}`

### Get Employment History for a Customer

```sql
use S4476658
SELECT ce.ceID, ce.ceCustomerID, ce.ceEmployerName, ce.ceJobTitle,
       ce.ceSupervisorName, ce.ceSupervisorPhone, ce.ceAnnualIncome,
       ce.ceStartDate, ce.ceEndDate, ce.ceCurrentEmploymentFlag
FROM CustomerEmployment ce
WHERE ce.ceCustomerID = @CustomerID
```
- **Use when:** Verifying employment data returned by the employment GET endpoint
- **Related endpoint:** `GET /api/prospects/v1/employment/company/{companyId}/property/{propertyId}`

### Get Save-Progress XML for a Guest Card

```sql
use S4476658
SELECT sp.spID, sp.spGuestCardID, sp.spSaveType, sp.spXMLData
FROM SaveProgress sp
WHERE sp.spGuestCardID = @GuestCardID
  AND sp.spSaveType = 'SP_RentalHistory'  -- or 'SP_Employment'
```
- **Use when:** Understanding the XML override data that the load endpoint merges with DB records
- **Notes:** The Prospects API load endpoints merge fields from this XML into the DB records. `SelectSingleNode` → `SelectNodes` is a known refactor area.

---

## Leased Accounts Domain

### Get Resident by Customer ID

```sql
use S4476658
SELECT TOP 10 c.custID, c.custFirstName, c.custLastName, c.custEmail,
       c.custPhone1, l.leaseID, l.leaseStartDate, l.leaseEndDate,
       l.leaseRent, u.unitNumber, u.unitTypeID
FROM Customer c
JOIN Lease l ON l.leaseCustomerID = c.custID
JOIN Unit u ON u.unitID = l.leaseUnitID
WHERE c.custType = 'R'  -- R = Resident
  AND l.leaseStatus = 'A'  -- A = Active
ORDER BY c.custID DESC
```
- **Use when:** Need a valid `customerId` for resident/leased account API calls

### Get Lease Details

```sql
use S4476658
SELECT l.leaseID, l.leaseCustomerID, l.leaseUnitID, l.leaseStartDate,
       l.leaseEndDate, l.leaseTerm, l.leaseRent, l.leaseMarketRent,
       l.leaseDeposit, l.leaseStatus, l.leaseMoveInDate, l.leaseMoveOutDate
FROM Lease l
WHERE l.leaseCustomerID = @CustomerID
ORDER BY l.leaseStartDate DESC
```
- **Use when:** Need lease data for leased account API calls

### Get Resident Employer (Leased Accounts)

```sql
use S4476658
SELECT re.reID, re.reCustomerID, re.reEmployerName, re.reJobTitle,
       re.reSupervisorName, re.rePhone, re.reIncome,
       re.reStartDate, re.reEndDate, re.reCurrentFlag
FROM ResidentEmployer re
WHERE re.reCustomerID = @CustomerID
```
- **Use when:** Verifying leased accounts employment data
- **Related endpoint:** `GET /api/leasedaccounts/v1/residentemployer/company/{companyId}/property/{propertyId}`

---

## Waitlist Domain

### Get Waitlist Entries

```sql
use S4476658
SELECT TOP 10 w.wlID, w.wlCustomerID, w.wlUnitTypeID, w.wlMoveInDate,
       w.wlDateAdded, w.wlPosition, w.wlPriority, w.wlStatus,
       c.custFirstName, c.custLastName
FROM Waitlist w
JOIN Customer c ON c.custID = w.wlCustomerID
ORDER BY w.wlPosition
```
- **Use when:** Need waitlist data for waitlist API calls

---

## Move-In / Move-Out Domain

### Get Move-In Records

```sql
use S4476658
SELECT TOP 10 l.leaseID, l.leaseCustomerID, l.leaseMoveInDate,
       l.leaseStartDate, l.leaseEndDate, l.leaseRent, l.leaseDeposit,
       u.unitNumber, c.custFirstName, c.custLastName
FROM Lease l
JOIN Unit u ON u.unitID = l.leaseUnitID
JOIN Customer c ON c.custID = l.leaseCustomerID
WHERE l.leaseMoveInDate IS NOT NULL
ORDER BY l.leaseMoveInDate DESC
```

### Get Move-Out Records

```sql
use S4476658
SELECT TOP 10 l.leaseID, l.leaseCustomerID, l.leaseMoveOutDate,
       l.leaseMoveOutReason, l.leaseForwardAddress,
       u.unitNumber, c.custFirstName, c.custLastName
FROM Lease l
JOIN Unit u ON u.unitID = l.leaseUnitID
JOIN Customer c ON c.custID = l.leaseCustomerID
WHERE l.leaseMoveOutDate IS NOT NULL
ORDER BY l.leaseMoveOutDate DESC
```

---

## Common Lookups

### Get Lease Terms

```sql
use S4476658
SELECT * FROM leaseterms
```
- **Columns:**
  - `ltermdisplaybit` — Whether the lease term is displayed (1) or not (0)
  - `NewLease` — Available for new leases (1/0)
  - `Renewal` — Available for renewals (1/0)
  - `Transfer` — Available for transfers (1/0)

### Get Unit Types

```sql
use S4476658
SELECT ut.unitTypeID, ut.unitTypeName, ut.unitTypeBedrooms,
       ut.unitTypeBathrooms, ut.unitTypeSqFt
FROM UnitType ut
ORDER BY ut.unitTypeName
```

### Get Units by Unit Type

```sql
use S4476658
SELECT u.unitID, u.unitNumber, u.unitTypeID, u.unitStatus,
       u.unitMarketRent, u.unitSqFt
FROM Unit u
WHERE u.unitTypeID = @UnitTypeID
ORDER BY u.unitNumber
```

### Get Charge Codes

```sql
use S4476658
SELECT cc.ccID, cc.ccCode, cc.ccDescription, cc.ccType, cc.ccAmount
FROM ChargeCode cc
WHERE cc.ccActiveFlag = 1
ORDER BY cc.ccCode
```

### Get Demographic Settings (Mandatory Fields Config)

```sql
use S4476658
SELECT ds.dsID, ds.dsSettingName, ds.dsSettingValue
FROM DemographicSettings ds
```
- **Use when:** Need to check configured minimum days for rental/employment history validation
- **Key settings:** `RentalHistoryMinimumDays`, employment equivalent

### Get Sources and Referrals

```sql
use S4476658
SELECT s.srcID, s.srcName, s.srcType, s.srcActiveFlag
FROM Source s
WHERE s.srcActiveFlag = 1
ORDER BY s.srcName
```

---

## Field Definitions Reference

The field definitions below are sourced from [OneSite Leasing & Rents Help](https://onesitehelp.realpage.com/OneSiteHTMLHelp/leasing%20and%20rents/40/292151.htm) and map UI field names to their business meanings. Use these to understand what data an API expects or returns.

### Prospects — Prospect Tile & Account-Level Fields

| UI Field | Definition | Likely DB Column |
|----------|-----------|-----------------|
| First/Middle/Last | Prospect's full name | `Customer.custFirstName`, `custMiddleName`, `custLastName` |
| Phone Number | Primary phone number | `Customer.custPhone1` |
| Email Address | Primary email | `Customer.custEmail` |
| Community | Property of interest | `GuestCard.gCPropertyID` |
| Unit Type | Floor plan of interest | `GuestCard.gCUnitTypeID` |
| Move-In Date | Desired move-in date | `GuestCard.gCMoveInDate` |
| Lease Term | Desired lease term (months) | `GuestCard.gCLeaseTerm` |
| Agent | Assigned leasing agent | `GuestCard.gCAgentID` |
| Source/Referral | How prospect found community | `GuestCard.gCSourceID` |
| Account Status | Current status (Active, Lost, etc.) | `GuestCard.gCStatus` |

### Prospects — Guest Card Tab (Personal Information)

| UI Field | Definition | Likely DB Column |
|----------|-----------|-----------------|
| First Name | Legal first name | `Customer.custFirstName` |
| Middle Name | Middle name | `Customer.custMiddleName` |
| Last Name | Legal last name | `Customer.custLastName` |
| Suffix | Name suffix (Jr., Sr., III) | `Customer.custSuffix` |
| Preferred Name | Nickname | `Customer.custPreferredName` |
| Birthdate | Date of birth | `Customer.custBirthDate` |
| SSN/Tax ID | Social Security Number (masked) | `Customer.custSSN` |
| Government ID Type | ID type (Driver's License, Passport) | `Customer.custGovIDType` |
| Government ID Number | ID number | `Customer.custGovIDNumber` |
| Government ID Exp Date | ID expiration date | `Customer.custGovIDExpDate` |
| Issuing Authority | State/agency that issued ID | `Customer.custGovIDIssuer` |

### Prospects — Guest Card Tab (Contact Information)

| UI Field | Definition | Likely DB Column |
|----------|-----------|-----------------|
| Primary Phone | Primary phone | `Customer.custPhone1` |
| Primary Phone Type | Phone type (Cell, Home, Work) | `Customer.custPhone1Type` |
| Secondary Phone | Secondary phone | `Customer.custPhone2` |
| Email | Primary email | `Customer.custEmail` |
| Contact Preference | Preferred contact method | `Customer.custContactPreference` |

### Prospects — Qualification Tab (Employment/Income)

| UI Field | Definition | Likely DB Column |
|----------|-----------|-----------------|
| Employer Name | Employer name | `CustomerEmployment.ceEmployerName` |
| Employer Phone | Employer phone for verification | `CustomerEmployment.cePhone` |
| Job Title | Job title/position | `CustomerEmployment.ceJobTitle` |
| Annual Income | Annual gross income | `CustomerEmployment.ceAnnualIncome` |
| Monthly Income | Monthly gross income | `CustomerEmployment.ceMonthlyIncome` |
| Employment Status | Current or previous | `CustomerEmployment.ceCurrentEmploymentFlag` |
| Hire Date | Employment start date | `CustomerEmployment.ceStartDate` |
| End Date | Employment end date (previous) | `CustomerEmployment.ceEndDate` |
| Supervisor Name | Supervisor name for verification | `CustomerEmployment.ceSupervisorName` |
| Supervisor Phone | Supervisor phone | `CustomerEmployment.ceSupervisorPhone` |

### Prospects — Qualification Tab (Rental History)

| UI Field | Definition | Likely DB Column |
|----------|-----------|-----------------|
| Residence Type | Current or previous | `historyType` (from save-progress XML) |
| Address Line 1 | Street address | `CustomerAddress.caAddr1` |
| Address Line 2 | Apt/suite number | `CustomerAddress.caAddr2` |
| City | City | `CustomerAddress.caCity` |
| State | State/province | `CustomerAddress.caState` |
| Zip Code | ZIP/postal code | `CustomerAddress.caZip` |
| Landlord/Mgmt Company | Landlord name | `residenceName` (from save-progress XML) |
| Landlord Phone | Landlord phone | `managerPhoneNumber` (from save-progress XML) |
| Monthly Rent | Monthly rent amount | `rentAmount` (from save-progress XML) |
| Move-In Date | Moved in date | `moveInDate` (from save-progress XML) |
| Move-Out Date | Moved out date (previous) | `moveOutDate` (from save-progress XML) |
| Reason for Leaving | Reason for leaving | `leavingReason` (from save-progress XML) |

### Prospects — Qualification Tab (Screening)

| UI Field | Definition | Likely DB Column |
|----------|-----------|-----------------|
| Broken Lease | History of breaking lease (Y/N) | `brokenLeaseFlag` |
| Eviction | History of eviction (Y/N) | `evictedFlag` |
| Felony | Felony conviction (Y/N) | screening result |
| Bankruptcy | Filed for bankruptcy (Y/N) | screening result |
| Refused to Pay Rent | History of refusing rent (Y/N) | `suedForRentFlag` |
| Credit Score | Credit score if available | screening result |
| Screening Status | Background check status | screening result |

### Applications — Applicant Tile

| UI Field | Definition | Likely DB Column |
|----------|-----------|-----------------|
| First Name, Last Name | Applicant name | `Customer.custFirstName`, `custLastName` |
| Status | Current applicant status | application status field |
| Move-in Date | Scheduled move-in | lease/application move-in date |
| Unit | Assigned unit | `Unit.unitNumber` |
| Floorplan | Unit floorplan | `UnitType.unitTypeName` |
| Applied Date | Application submission date | application date field |
| Approved Date | Approval date | application approval date |
| Lease Sign Date | Lease signing date | lease sign date field |

### Applications — Qualification Tab

| UI Field | Definition | Likely DB Column |
|----------|-----------|-----------------|
| ID Type | Identification type (DL, Passport) | `Customer.custGovIDType` |
| ID Number | Identification number | `Customer.custGovIDNumber` |
| Issuing State/Province | ID issuing state | `Customer.custGovIDIssuer` |
| Expiration Date | ID expiration | `Customer.custGovIDExpDate` |
| SSN | Social Security Number (masked) | `Customer.custSSN` |
| Employer | Current employer name | `CustomerEmployment.ceEmployerName` |
| Title | Job title | `CustomerEmployment.ceJobTitle` |
| Supervisor | Supervisor name | `CustomerEmployment.ceSupervisorName` |
| Monthly Income | Monthly gross income | `CustomerEmployment.ceMonthlyIncome` |

### Applications — Lease Tab

| UI Field | Definition | Likely DB Column |
|----------|-----------|-----------------|
| Unit | Assigned unit | `Unit.unitNumber` |
| Floorplan | Unit floorplan | `UnitType.unitTypeName` |
| Move-in Date | Scheduled move-in | `Lease.leaseMoveInDate` |
| Lease Start Date | Lease start | `Lease.leaseStartDate` |
| Lease End Date | Lease end | `Lease.leaseEndDate` |
| Lease Term | Term in months | `Lease.leaseTerm` |
| Market Rent | Market rent for unit | `Unit.unitMarketRent` |
| Lease Rent | Agreed lease rent | `Lease.leaseRent` |
| Concession | Rent concession | concession field |
| Deposit | Security deposit amount | `Lease.leaseDeposit` |
| Pro-Rated Rent | Prorated first month rent | calculated |
| Recurring Charges | Other recurring charges (pet rent, parking) | charge codes |

### Residents — Resident Tile

| UI Field | Definition | Likely DB Column |
|----------|-----------|-----------------|
| First Name, Last Name | Resident name | `Customer.custFirstName`, `custLastName` |
| Status | Current status (Current, Notice, Past) | resident status field |
| Unit | Currently occupied unit | `Unit.unitNumber` |
| Floorplan | Unit floorplan | `UnitType.unitTypeName` |
| Move-in Date | Date moved in | `Lease.leaseMoveInDate` |
| Lease Start/End Date | Current lease dates | `Lease.leaseStartDate`, `leaseEndDate` |
| Lease Rent | Current lease rent | `Lease.leaseRent` |
| Balance | Account balance | calculated from transactions |

### Residents — Lease Tab (Notice/Renewal)

| UI Field | Definition | Likely DB Column |
|----------|-----------|-----------------|
| Notice Date | Date notice given to vacate | notice date field |
| Move-out Date | Scheduled move-out | `Lease.leaseMoveOutDate` |
| Move-out Reason | Reason for moving out | `Lease.leaseMoveOutReason` |
| Forwarding Address | Post-vacate address | `Lease.leaseForwardAddress` |
| Renewal Status | Renewal status (Offered, Accepted, Declined) | renewal status field |
| New Lease Start/End/Term/Rent | Renewal lease details | renewal lease fields |

### Waitlist Fields

| UI Field | Definition | Likely DB Column |
|----------|-----------|-----------------|
| Unit Type | Preferred unit type | `Waitlist.wlUnitTypeID` |
| Bedrooms/Bathrooms | Preferred bed/bath count | unit type attributes |
| Min/Max Rent | Rent range | waitlist preferences |
| Move-In Date | Desired move-in date | `Waitlist.wlMoveInDate` |
| Date Added | Date added to waitlist | `Waitlist.wlDateAdded` |
| Position | Waitlist position | `Waitlist.wlPosition` |
| Priority | Priority level (High, Medium, Low) | `Waitlist.wlPriority` |
| Status | Waitlist status (Active, Contacted, Removed) | `Waitlist.wlStatus` |

### Move-In Fields

| UI Field | Definition | Likely DB Column |
|----------|-----------|-----------------|
| Move-In Date | Actual move-in date | `Lease.leaseMoveInDate` |
| Lease Start/End Date | Lease term dates | `Lease.leaseStartDate`, `leaseEndDate` |
| Unit / Unit Type | Assigned unit and floorplan | `Unit.unitNumber`, `UnitType.unitTypeName` |
| Market Rent | Current market rent | `Unit.unitMarketRent` |
| Lease Rent | Lease rent amount | `Lease.leaseRent` |
| Deposit | Security deposit collected | `Lease.leaseDeposit` |
| Prorate Amount | Prorated rent for partial first month | calculated |
| Move-In Charges | Total charges due at move-in | calculated from charge codes |
| Lease Term | Lease length in months | `Lease.leaseTerm` |

### Move-Out Fields

| UI Field | Definition | Likely DB Column |
|----------|-----------|-----------------|
| Move-Out Date | Actual vacate date | `Lease.leaseMoveOutDate` |
| Notice Date | Date notice given | notice date field |
| Reason | Move-out reason | `Lease.leaseMoveOutReason` |
| Forwarding Address | Post-vacate address | `Lease.leaseForwardAddress` |
| Deposit Refund | Deposit amount to refund | calculated |
| Deposit Charges | Charges deducted from deposit | calculated |
| Balance Due | Remaining balance owed | calculated from transactions |
| Early Termination Fee | Fee for early lease break | charge code |
| FAS | Final Account Statement completed (Y/N) | FAS status field |

---

## Contributing

When the `os-service-consumer` agent discovers a new SQL pattern during a session, it will suggest adding it here. To contribute:

1. Add the SQL query under the appropriate domain section
2. Include a `use S4476658` prefix (default test database)
3. Add a "Use when" description explaining when this query is needed
4. Note the related API endpoint if applicable
5. Map key column names to API field names

**Note:** The "Likely DB Column" mappings in the Field Definitions Reference are best-guess mappings based on naming conventions. They should be verified against the actual schema during implementation. Tag verified mappings with ✅.
