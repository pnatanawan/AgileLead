# Work Item Update Content for AZD-2404320 (PME-451110)
## Date Range Filtering in Task Summary Widget

This document contains the detailed content for each field in the Azure DevOps work item. Copy and paste these sections into the corresponding fields in AZD-2404320.

---

## Field: Product Vision

```
Enable date range filtering in the Task Summary widget to support modern multi-day task analysis, unblocking client migration from Classic OneSite and improving property management visibility into workflow tasks across custom date ranges rather than single-day snapshots.
```

---

## Field: Objectives

```
1. Replace hardcoded single-day filtering logic with configurable date range support across all 16 task workflows
2. Implement feature flag-based gradual rollout to validate behavior with pilot customer (Lindemann) before organization-wide release
3. Maintain 100% backward compatibility with existing single-date filtering for non-pilot customers
4. Achieve <2 second query performance for date ranges up to 365 days
5. Provide complete API and UI implementation that aligns with Classic OneSite date range capabilities
```

---

## Field: Key Results

```
- KR1: All 7 affected stored procedures (uspGetTaskSummaryTileSummary_UPM, uspTaskSummaryDashboardCount_UPM, and 5 supporting SPs) updated to support BETWEEN clause date range filtering - Target 100% coverage
- KR2: UI displays date range picker to all users when feature flag enabled; single date picker for disabled users - Target 0 validation errors in UAT
- KR3: API validates and rejects invalid date ranges (endDate < startDate, range > 365 days) - Target 100% pass rate on validation tests
- KR4: Feature flag toggleable without code deployment; supports gradual rollout from 0% to 100% - Target deploy to 10% by Day 3, 100% by Day 16
- KR5: Query execution completes in <2 seconds for typical datasets (100K+ records over 90-day range) - Target 95th percentile <2000ms
- KR6: Lindemann property manager successfully filters tasks by date range and migrates from Classic OneSite - Target migration complete by [DATE]
- KR7: Zero production incidents post-deployment; complete rollback capability via feature flag - Target zero escalations, <1 minute rollback time
- KR8: Complete technical documentation including code snippets, test scenarios, and customer migration guide - Target 100% documentation coverage
```

---

## Field: Assumptions

```
1. Client's date picker component (rp-daterangepicker) is functionally equivalent to Classic OneSite date range picker
2. Task workflow data includes task_date column across all 16 workflows (MIN, MOU, RFAS, ApplicantFAS, FAS-READY-APP, FAS-REJECTED, NRFAS, TRAN, TRANSOUT, GFR, Appointment, FUP, APPINPROG, PVR, FVR, CWL)
3. LaunchDarkly SDK (v7.2.0 or higher) will be successfully integrated into application and configured with multi-context evaluation
4. Current hardcoded logic in uspGetTaskSummaryTileSummary_UPM (SET @LV_ENDDate = @LV_StartDate) is the primary blocker preventing date range filtering
5. Lindemann customer property ID will be identified for controlled feature flag targeting during phased rollout
6. Database query performance is acceptable for existing SPs and will scale linearly with date range expansion
7. No breaking changes required to existing API contracts; date range parameters can be added additively
8. Feature flag will be managed by product/release team; no customer-facing configuration required
9. All team members will have access to Azure DevOps, GitHub, LaunchDarkly admin console, and development SQL Server instance
10. Testing environment mirrors production data volume and structure for accurate performance validation
```

---

## Field: Background

```
PROBLEM STATEMENT
The Dashboard "Task Summary" widget displays a date picker UI component but enforces single-day task queries due to hardcoded SQL logic. This fundamental limitation prevents Lindemann Multifamily Management LLC (a strategic customer) from migrating away from Classic OneSite, where they actively use date range filtering.

ROOT CAUSE ANALYSIS
Investigation of the data layer revealed the bug in uspGetTaskSummaryTileSummary_UPM stored procedure:
- Line: SET @LV_ENDDate = @LV_StartDate
- Effect: Hardcodes the query end date to equal the start date, defeating any date range logic
- Impact: Users see a date picker UI but can only retrieve data for a single day, exactly like the legacy behavior

CUSTOMER IMPACT
- Lindemann Multifamily Management LLC cannot migrate to modern OneSite without losing date range filtering capability
- Adoption is blocked on a core workflow feature
- Competitive disadvantage vs. Classic OneSite

RELATED WORKFLOWS AFFECTED
16 task workflows rely on date filtering:
1. MIN (Move-in): Entry task for move-in process
2. MOU (Move-out): Entry task for move-out process
3. RFAS (Resident FAS): Financial accounting statement
4. ApplicantFAS: Applicant financial accounting
5. FAS-READY-APP: FAS ready for application
6. FAS-REJECTED: Rejected FAS
7. NRFAS: Non-resident FAS
8. TRAN (Transfer): Resident transfer
9. TRANSOUT: Transfer out
10. GFR (Guarantor Form): Guarantor form requests
11. Appointment: Service appointments
12. FUP (Follow-up): Required follow-ups
13. APPINPROG: Applications in progress
14. PVR (Pending Virtual Refunds): Virtual refund requests pending
15. FVR (Failed Virtual Refunds): Virtual refund failures
16. CWL (Conventional Waitlist): Resident waitlist

TECHNICAL CONTEXT
- Stack: Angular (UI), C# .NET (API), SQL Server (Database)
- Architecture: Layered (UI → Controller → Manager → DAO → Database)
- Feature flagging platform: LaunchDarkly (Enterprise feature flag service)
- Existing API endpoint: POST /api/dashboard/v1/dashboardTileExecution/Execute/TaskSummary/{TileID}
- Current parameter structure: single dateFilter epoch milliseconds

BUSINESS CONTEXT
- Lindemann Multifamily Management LLC is strategic customer / reference account
- Classic OneSite has mature date range filtering (used actively by customer)
- Date range filtering is documented feature in modern OneSite, just not functional
- Feature flag framework already exists in codebase for gradual rollouts
- LaunchDarkly infrastructure already provisioned and configured

PRIOR WORK
- Spike WI-2380310 analyzed current architecture and identified root cause
- Technical analysis identified all 7 affected stored procedures
- Spike enumerated all 16 workflows that depend on date filtering
- LaunchDarkly integration pattern already validated in another feature
```

---

## Field: Acceptance Criteria

```
FUNCTIONAL ACCEPTANCE CRITERIA

[ ] AC1: Date Range Picker Displays
  - Given: User navigates to dashboard when feature flag is ON
  - When: Task Summary widget renders
  - Then: Date range picker component displays (not single date picker)
    AND: Default range set to last 30 days from current date
    AND: User can select any date range

[ ] AC2: Date Range Validation (Client-Side)
  - Given: User has date range picker visible
  - When: User selects startDate > endDate
  - Then: Error message displays: "Start date must be less than or equal to end date"
    AND: No API request is made
    AND: Filter is not applied

[ ] AC3: Date Range Validation (Client-Side) - Maximum Days
  - Given: User has date range picker visible
  - When: User selects range > 365 days
  - Then: Error message displays: "Date range cannot exceed 365 days"
    AND: No API request is made

[ ] AC4: Valid Date Range - API Execution
  - Given: User selects valid date range (e.g., Jan 1 - Jan 31, 2024)
  - When: User clicks "Apply Filter" or equivalent
  - Then: API POST /dashboardTileExecution/Execute/TaskSummary/{tileID} called with:
    {
      "filterBy": {
        "dateFilterFrom": "1704067200000",
        "dateFilterTo": "1704672000000",
        ...
      }
    }
    AND: API returns Task Summary widget data with all 16 workflow counts for date range
    AND: Results display in <2 seconds

[ ] AC5: Database Query - All Workflows Included
  - Given: Valid date range specified
  - When: uspTaskSummaryDashboardCount_UPM executed
  - Then: Results include counts for all 16 workflows:
    MIN, MOU, RFAS, ApplicantFAS, FAS-READY-APP, FAS-REJECTED, NRFAS, TRAN, TRANSOUT, GFR, Appointment, FUP, APPINPROG, PVR, FVR, CWL
    AND: Each count represents tasks within date range (inclusive start, inclusive end)

[ ] AC6: Hardcoded Bug Fixed
  - Given: uspGetTaskSummaryTileSummary_UPM receives @dateFilterFrom and @dateFilterTo
  - When: Stored procedure executes
  - Then: Does NOT execute: SET @LV_ENDDate = @LV_StartDate (hardcoded logic removed)
    AND: Instead uses: SET @LV_ENDDate from @dateFilterTo parameter
    AND: WHERE clauses use: BETWEEN @LV_StartDate AND @LV_ENDDate

[ ] AC7: Backward Compatibility - Single Date
  - Given: Feature flag is OFF or user's organization not yet enabled
  - When: User navigates to Task Summary widget
  - Then: Single date picker displays (not date range picker)
    AND: Existing filterBy.dateFilter parameter used
    AND: Behavior identical to current product release
    AND: No breaking changes to API or UI

[ ] AC8: Backward Compatibility - API
  - Given: Legacy API request with only dateFilter parameter
  - When: DashboardTileExecutionV1Controller.ExecuteTileTaskSummary() called
  - Then: Request succeeds (no error)
    AND: dateFilter treated as single-day query
    AND: All 16 workflows returned for that single day

[ ] AC9: Feature Flag OFF Behavior
  - Given: dashboards-daterangefilter-enabled = false for user's property
  - When: API receives request with dateFilterFrom and dateFilterTo
  - Then: Parameters are silently removed from request
    AND: Legacy single-date behavior used
    AND: No error returned to client

[ ] AC10: Feature Flag ON Behavior
  - Given: dashboards-daterangefilter-enabled = true for user's property
  - When: DashboardTileExecutionV1Controller validates request
  - Then: Date range parameters processed
    AND: Validation applied (dateFrom <= dateTo, range <= 365 days)
    AND: Full date range query executed

[ ] AC11: Data Parity with Classic OneSite
  - Given: Lindemann customer migrating from Classic OneSite
  - When: Same date range queried in new OneSite vs Classic OneSite
  - Then: Results match exactly
    AND: Workflow counts identical
    AND: Customer confirms parity before cutover

[ ] AC12: Performance Target Met
  - Given: Database query over 90-day date range with 100K+ task records
  - When: uspTaskSummaryDashboardCount_UPM executes
  - Then: Execution completes in <2000ms (95th percentile)
    AND: No timeout errors

[ ] AC13: Error Handling - Invalid Format
  - Given: API receives dateFilterFrom with non-numeric value
  - When: ValidateDateRangeParameters() called
  - Then: HTTP 400 response returned with error:
    "Invalid dateFilterFrom format. Expected epoch milliseconds as string."

[ ] AC14: Error Handling - Invalid Range
  - Given: API receives dateFilterFrom > dateFilterTo
  - When: ValidateDateRangeParameters() called
  - Then: HTTP 400 response returned with error:
    "dateFilterFrom must be less than or equal to dateFilterTo"

[ ] AC15: Error Handling - Range Too Large
  - Given: API receives range spanning > 365 days
  - When: ValidateDateRangeParameters() called
  - Then: HTTP 400 response returned with error:
    "Date range cannot exceed 365 days"

[ ] AC16: Unit Tests Pass
  - Given: Code implementation complete
  - When: Test suite executed
  - Then: All 20+ unit tests pass
    AND: Code coverage >= 90%

[ ] AC17: Integration Tests Pass
  - Given: Full stack deployed to staging
  - When: Integration test suite executed with real database
  - Then: All 5+ integration tests pass
    AND: End-to-end flow validated (UI → API → Database)

[ ] AC18: SQL Tests Pass
  - Given: Stored procedures deployed
  - When: SQL test scripts execute
  - Then: All 4+ SQL test scenarios pass
    AND: Performance benchmarks met
    AND: Legacy parameter compat verified

[ ] AC19: UAT Complete - Lindemann Customer
  - Given: Feature ready for UAT
  - When: Lindemann property manager uses new feature in staging
  - Then: Date range filtering works as designed
    AND: Results match Classic OneSite
    AND: Customer signoff obtained
    AND: No critical bugs reported

[ ] AC20: Phased Rollout Complete
  - Given: Feature validated in UAT
  - When: Phased rollout executed:
    - Day 1-3: 10% of properties (including Lindemann)
    - Day 4-8: 25% of properties
    - Day 9-12: 50% of properties
    - Day 13-16: 100% of properties
  - Then: No production incidents (zero escalations)
    AND: Performance metrics remain in target
    AND: Custom events tracked in LaunchDarkly

NON-FUNCTIONAL ACCEPTANCE CRITERIA

[ ] NFR-AC1: Performance - Query Time
  - 90-day date range query with 100K+ records: <2 seconds (95th percentile)
  - Single-day date range query: <500ms (99th percentile)

[ ] NFR-AC2: Performance - API Response
  - API response time (excluding DB): <500ms (99th percentile)
  - Validation overhead: <50ms

[ ] NFR-AC3: Backward Compatibility
  - Existing single-date API requests: 100% compatible
  - No breaking changes to TileData response contract
  - Legacy customers see no difference

[ ] NFR-AC4: Security
  - Input validation prevents SQL injection
  - Date range validation prevents denial of service (>365 day ranges rejected)
  - No new authentication or authorization logic required (existing controls suffice)

[ ] NFR-AC5: Reliability
  - Feature flag failure: defaults to OFF (safe state)
  - Database unavailability: graceful error handling
  - Rollback via feature flag: <1 minute to complete

[ ] NFR-AC6: Maintainability
  - All code includes comments explaining date range logic
  - SQL parameter naming consistent (@dateFilterFrom, @dateFilterTo)
  - Complete technical documentation included
```

---

## Field: Additional Information

```
RELATED ARTIFACTS
- Impact Analysis: AZD_WI_2404320_IMPACT_ANALYSIS.md (800+ lines)
- Implementation Plan: AZD_WI_2404320_IMPLEMENTATION_PLAN.md (900+ lines)
- Technical Specification: AZD_WI_2404320_FORMAL_TECHNICAL_SPECIFICATION.md (2000+ lines)
- LaunchDarkly Guide: AZD_WI_2404320_LAUNCHDARKLY_GUIDE.md (1100+ lines)

RELATED SPIKE
- WI-2380310: Analysis and Technical Specification for Date Range Support
  - Identified root cause: hardcoded SQL logic in uspGetTaskSummaryTileSummary_UPM
  - Enumerated all affected workflows (16 total)
  - Identified all affected stored procedures (7 total)
  - Validated architectural feasibility

DEPENDENCIES
- LaunchDarkly SDK (v7.2.0 or higher): Feature flagging platform
- RealPage DateRangePicker component: UI component library
- SQL Server 2019+: Database platform
- .NET 6.0+: API framework

DEVELOPMENT PHASES & EFFORT ESTIMATES
Phase 1: Frontend UI Changes
  - Components: tile-filters.component (HTML, TypeScript)
  - Effort: 13 hours
  - Deliverables: Date range picker, validation, epoch conversion pipe
  
Phase 2: API/Controller Layer
  - Components: DashboardTileExecutionV1Controller, LaunchDarklyFeatureFlagService
  - Effort: 8.5 hours
  - Deliverables: Feature flag evaluation, parameter validation, error handling
  
Phase 3: Manager/DAO Layer
  - Components: TileTemplateManager, TileTemplateDAO
  - Effort: 6 hours
  - Deliverables: Parameter extraction, SQL command building
  
Phase 4: Database & SQL
  - Components: 7 Stored Procedures (uspGetTaskSummaryTileSummary_UPM, etc.)
  - Effort: 12 hours
  - Deliverables: Date range logic fix, BETWEEN clause implementation
  
Phase 5: Testing
  - Components: Unit, Integration, SQL, UAT tests
  - Effort: 15 hours
  - Deliverables: 20+ test cases, 90%+ code coverage
  
Phase 6: Release & Documentation
  - Components: Release notes, migration guide, developer guide
  - Effort: 6 hours
  - Deliverables: Complete documentation, customer communication

TOTAL EFFORT: 60.5 hours (~3 weeks at 20 hrs/week)

TEST STRATEGY
- Unit Tests: 20+ test cases covering validation, parameter handling, legacy compat
- Integration Tests: 5+ scenarios covering full end-to-end flow
- SQL Tests: 4+ scenarios covering stored procedures, performance, backward compat
- UAT: 3+ scenarios with Lindemann customer
- Performance Tests: Verify <2 second query times

ROLLOUT STRATEGY
- Feature Flag: dashboards-daterangefilter-enabled
- Default: OFF (safe state)
- Phased Rollout:
  * Day 1-3: 10% (include Lindemann for controlled validation)
  * Day 4-8: 25%
  * Day 9-12: 50%
  * Day 13-16: 100%
- Kill Switch: Can disable instantly via feature flag toggle
- Rollback: Complete rollback in <1 minute if needed

DEFINITION OF DONE
- All code changes implemented per specification
- All unit/integration/SQL tests passing (90%+ coverage)
- Code reviewed and approved by tech lead
- UAT completed with customer signoff
- Technical documentation complete
- Customer migration guide provided
- Release notes published
- Feature flag configured in LaunchDarkly
- Rollout plan validated
- Monitoring/alerts configured
- Deployment runbook created
- Post-deployment support plan documented

KNOWN ISSUES / CONSIDERATIONS
- Database query performance dependent on proper indexes on task_date column
  Action: Verify indexes exist on all 7 stored procedures' base tables before deployment
  
- Epoch millisecond conversion requires careful handling to avoid off-by-one errors
  Action: Use standardized conversion function across all layers
  
- Feature flag targeting by property requires accurate PropertyID context
  Action: Validate PropertyID context builder in LaunchDarkly service
  
- Backward compatibility must be maintained for non-pilot customers
  Action: Feature flag ON only for Lindemann initially, gradual rollout

CUSTOMER COMMUNICATION TIMELINE
- Week 1 (Dev): Internal team kickoff, development begins
- Week 2 (Dev): UAT staging activation, Lindemann customer notified of UAT opportunity
- Week 3 (UAT): Lindemann customer validates in staging, provides feedback
- Week 4 (Deploy): Phased production rollout begins (10% Lindemann day 1)
- Week 5 (Deploy/Monitor): Monitor metrics, complete phased rollout to 100%
- Customer Success: Schedule post-deployment review with Lindemann

SUCCESS METRICS
- KPI1: Lindemann successfully migrates from Classic OneSite (by [TARGET DATE])
- KPI2: Migration occurs without security incidents, data loss, or performance degradation
- KPI3: Feature flag enables gradual rollout without production incidents (target: 0 escalations)
- KPI4: Query performance maintained <2 seconds (95th percentile)
- KPI5: All 16 workflows report correct counts for specified date ranges
- KPI6: Feature adopted by 80% of active properties within 30 days of 100% rollout
```

---

## Field: Risk Comment

```
TECHNICAL RISKS

Risk TR1: SQL Performance Degradation
  Probability: Medium | Impact: High | Severity: HIGH
  Description: Adding BETWEEN logic to date range queries may degrade performance if task_date column lacks proper indexes
  Mitigation: 
    - Verify indexes exist on task_date column in all 7 SPs before deployment
    - Load test with 100K+ records in 90-day range to validate <2s performance
    - Have index creation script ready as emergency action
  Contingency: Revert to hardcoded single-day logic via feature flag if performance unacceptable

Risk TR2: Epoch Millisecond Conversion Errors
  Probability: Medium | Impact: Medium | Severity: MEDIUM
  Description: Off-by-one errors in epoch millisecond to datetime conversion could cause queries to miss tasks on boundary dates
  Mitigation:
    - Implement comprehensive test coverage for epoch conversion (especially midnight boundary)
    - Use standardized conversion function across all layers
    - Include SQL test cases for edge dates (e.g., Jan 1 00:00:00, Jan 1 23:59:59)
  Contingency: Add 1-day buffer to date ranges if conversion issues discovered post-deploy

Risk TR3: Feature Flag Configuration Complexity
  Probability: Low | Impact: Medium | Severity: MEDIUM
  Description: LaunchDarkly context builders (user, property, organization) misconfigured, flag not evaluated correctly
  Mitigation:
    - Have LaunchDarkly SME validate context builder logic before deployment
    - Test flag evaluation with multiple context combinations in staging
    - Document context builder behavior clearly in code comments
  Contingency: Disable feature for specific properties via flag rules if targeting fails

Risk TR4: Backward Compatibility Broken
  Probability: Low | Impact: High | Severity: HIGH
  Description: New dateFilterFrom/dateFilterTo parameters interfere with legacy dateFilter for non-pilot customers
  Mitigation:
    - API validation logic prioritizes dateFilterFrom/dateFilterTo only if BOTH present
    - Legacy dateFilter processed if date range params absent
    - Feature flag OFF by default, requiring explicit enablement
  Contingency: Can revert dateFilterFrom/dateFilterTo removal in controller if compat issues emerge

BUSINESS RISKS

Risk BR1: Lindemann Customer Migration Delay
  Probability: Medium | Impact: Critical | Severity: CRITICAL
  Description: Feature not deployed in time for scheduled Lindemann migration, customer remains on Classic OneSite
  Mitigation:
    - Assign dedicated dev team to 60.5-hour effort (3 weeks)
    - Daily standup for status tracking
    - Senior dev on-call for blocking issues
    - UAT scheduled in week 2, allowing week 3 for fixes
  Contingency: Negotiate extension with Lindemann if critical blocker; prioritize bug fixes over polish

Risk BR2: Adoption Slower Than Expected
  Probability: Low | Impact: Medium | Severity: MEDIUM
  Description: After 100% rollout, customers don't adopt date range filtering (stick with single-day queries)
  Mitigation:
    - Include customer communication highlighting new capability
    - Provide How-to guide and training materials
    - Monitor adoption metrics via custom events tracked in LaunchDarkly
  Contingency: Gather feedback from non-adopters to identify usability issues; iterate on UI/UX

OPERATIONAL RISKS

Risk OR1: Production Hotfix Needed During Rollout
  Probability: Low | Impact: High | Severity: HIGH
  Description: Critical bug discovered during phased rollout (e.g., data corruption in calculations)
  Mitigation:
    - Feature flag provides instant kill switch (all 16 workflows revert to legacy logic)
    - Rollback time: <1 minute
    - Have emergency on-call engineer available during phased rollout days
  Contingency: Disable feature flag immediately, revert code in next deploy cycle

Risk OR2: Database Lock Contention
  Probability: Low | Impact: Medium | Severity: MEDIUM
  Description: New stored procedures cause lock contention on task_date index during peak usage
  Mitigation:
    - Load test during peak hours in staging
    - Monitor blocking sessions during first week of production rollout
    - Have DBA on standby first week
  Contingency: Adjust stored procedure isolation level or split large queries if blocking occurs

CHANGE MANAGEMENT RISKS

Risk CM1: Insufficient User Communication
  Probability: Medium | Impact: Low | Severity: MEDIUM
  Description: Customers unaware of new feature capability, see unfamiliar UI and contact support
  Mitigation:
    - Release notes published before 100% rollout
    - Customer success team briefed on feature
    - FAQ document published
    - In-app tooltips explain new date range picker
  Contingency: Post announcement in customer portal; offer webinar training session

Risk CM2: Staging Environment Data Mismatch
  Probability: Low | Impact: Medium | Severity: MEDIUM
  Description: Staging data doesn't match production volume/patterns, performance issues not caught until production
  Mitigation:
    - Load staging with production data subset before UAT
    - Include >100K task records in staging test data
    - Run performance tests during peak hour simulation
  Contingency: Priority UAT on production if staging insufficient; be ready for quick rollback

TOTAL RISK EXPOSURE: MEDIUM
Overall risk is managed through feature flag kill switch, phased rollout, comprehensive testing, and customer preparation.
```

---

## Field: Non-Functional Requirements

```
PERFORMANCE REQUIREMENTS

NFR-P1: Query Execution Time
  Requirement: Date range queries shall complete in <2 seconds for typical datasets
  Details: 
    - Typical dataset: 100K+ task records, 90-day date range
    - Measured at: 95th percentile (not worst-case outliers)
    - Single-day queries: <500ms (99th percentile)
  Acceptance Test: Load test with 100K records, 90-day range, <2000ms result

NFR-P2: API Response Time
  Requirement: API response time shall not exceed 500ms (excluding database)
  Details:
    - Includes: Request validation, feature flag evaluation, response serialization
    - Excludes: Database query time
    - Measured at: 99th percentile
  Acceptance Test: API response time monitoring in staging, <500ms for 99% of requests

NFR-P3: Validation Latency
  Requirement: Parameter validation shall add <50ms overhead
  Details:
    - Date format validation, range validation, legacy compat checks
  Acceptance Test: Measure validation time separately from query execution

SCALABILITY REQUIREMENTS

NFR-S1: Concurrent User Load
  Requirement: System shall support 1000+ concurrent users querying Task Summary with date ranges
  Details: No degradation beyond single-user performance at 1000 users
  Acceptance Test: Load test with 1000 VUs, measure response time and error rate

NFR-S2: Date Range Maximum
  Requirement: Date ranges up to 365 days shall be supported without degradation
  Details: Performance for 365-day range shall match 30-day range (within 10%)
  Acceptance Test: Compare 30-day vs 365-day performance metrics

NFR-S3: Workflow Scale
  Requirement: All 16 workflows shall return counts in single query
  Details: No workflow omitted, all included in response
  Acceptance Test: Verify all 16 workflows present in response

AVAILABILITY REQUIREMENTS

NFR-A1: Graceful Degradation
  Requirement: If date range filtering unavailable, single-day filtering shall still work
  Details: Feature flag OFF → legacy behavior, no errors
  Acceptance Test: Disable feature flag, verify legacy queries succeed

NFR-A2: Feature Flag Kill Switch
  Requirement: Feature flag toggle shall disable date range filtering within <1 minute
  Details: All users revert to legacy single-day logic
  Acceptance Test: Toggle feature flag, verify behavior change within 60 seconds

NFR-A3: Error Recovery
  Requirement: Invalid date range requests shall not crash system; return clear error message
  Details: HTTP 400 with descriptive error, transaction rolled back
  Acceptance Test: Send invalid date ranges, verify graceful error handling

SECURITY REQUIREMENTS

NFR-SEC1: SQL Injection Prevention
  Requirement: Date parameters shall be parameterized (not concatenated into SQL)
  Details: @dateFilterFrom, @dateFilterTo as typed parameters
  Acceptance Test: Code review confirms parameterized queries, OWASP SQL injection testing

NFR-SEC2: Input Validation
  Requirement: All date inputs shall be validated for format and range
  Details:
    - Format: Epoch milliseconds as string
    - Range: 0 <= value <= 9999999999999 (year 5138)
    - Logic: dateFilterFrom <= dateFilterTo, range <= 365 days
  Acceptance Test: Unit tests for all validation rules

NFR-SEC3: Denial of Service Prevention
  Requirement: System shall reject date ranges exceeding 365 days
  Details: Prevents resource exhaustion through large range queries
  Acceptance Test: Attempt 1000-day range, verify rejection with HTTP 400

NFR-SEC4: Data Access Control
  Requirement: Users shall only see tasks for properties they have access to
  Details: Existing property-level authorization unchanged
  Acceptance Test: Verify access control audit logs

RELIABILITY REQUIREMENTS

NFR-R1: Backward Compatibility
  Requirement: Existing APIs using single-date filtering shall work unchanged
  Details: No breaking changes, new parameters optional
  Acceptance Test: Run existing integration tests against new code, all pass

NFR-R2: Legacy Parameter Support
  Requirement: legacy @dateFilter parameter shall work indefinitely
  Details: Not deprecated, functional feature flag OFF
  Acceptance Test: Query with dateFilter only, verify results correct

NFR-R3: Transaction Integrity
  Requirement: Database transactions shall not be corrupted by date range logic
  Details: ACID properties maintained, all or nothing execution
  Acceptance Test: Rollback tests for concurrent updates during date range query

MAINTAINABILITY REQUIREMENTS

NFR-M1: Code Documentation
  Requirement: All date range logic shall include inline comments explaining behavior
  Details: Explain epoch conversion, BETWEEN logic, legacy compat
  Acceptance Test: Code review confirms >80% of logic has comments

NFR-M2: Test Coverage
  Requirement: Code coverage >= 90% for all modified/new code
  Details: Unit + integration + SQL tests
  Acceptance Test: Code coverage report shows >=90%

NFR-M3: Consistent Naming
  Requirement: Date parameters shall use consistent naming across all layers
  Details: @dateFilterFrom, @dateFilterTo, dateFilterFrom, dateFilterTo
  Acceptance Test: Search codebase confirms consistent naming

NFR-M4: Technical Documentation
  Requirement: Complete technical documentation provided for future maintainers
  Details: Specification, architecture, code examples, edge cases, troubleshooting
  Acceptance Test: Technical spec document (2000+ lines) complete and reviewed

INTEROPERABILITY REQUIREMENTS

NFR-I1: API Contract Compatibility
  Requirement: Response format unchanged; new data fields optional
  Details: TileData structure accepts optional dateRange fields
  Acceptance Test: Old clients ignore dateRange, new clients use it

NFR-I2: UI Component Compatibility
  Requirement: rp-daterangepicker must be compatible with existing tile-filters module
  Details: No breaking changes to module structure
  Acceptance Test: Existing tests pass, no module dependency issues

NFR-I3: LaunchDarkly SDK Compatibility
  Requirement: LaunchDarkly SDK v7.2.0 or higher must be integrated
  Details: No conflicts with existing SDK versions
  Acceptance Test: Dependency check, SDK version validation

COMPLIANCE REQUIREMENTS

NFR-C1: Data Privacy
  Requirement: Date filtering shall not change data privacy/access controls
  Details: No new PII exposure, existing GDPR/CCPA controls unchanged
  Acceptance Test: Security audit confirms no privacy violations

NFR-C2: Audit Trail
  Requirement: Custom events tracked in LaunchDarkly for compliance/debugging
  Details: Events: task-summary-flag-evaluated, task-summary-executed, task-summary-error
  Acceptance Test: Verify events in LaunchDarkly dashboard

NFR-C3: Monitoring & Alerting
  Requirement: Performance metrics and errors monitored; alerts configured
  Details: Dashboard, threshold alerts, on-call escalation
  Acceptance Test: Verify alerts trigger on threshold breach in staging

USABILITY REQUIREMENTS

NFR-U1: UI Consistency
  Requirement: Date range picker UX matches existing RealPage design patterns
  Details: Layout, styling, interactions consistent with product
  Acceptance Test: Design review with UX team, user testing with 2+ users

NFR-U2: Error Messages
  Requirement: Error messages shall be clear and actionable
  Details: Specify what went wrong and how to fix it
  Acceptance Test: UAT users can understand and act on error messages

NFR-U3: Default Values
  Requirement: Default date range shall be 30 days (not confusing empty state)
  Details: Start: 30 days ago, End: today
  Acceptance Test: Verify default range displays on component init

NFR-U4: Accessibility
  Requirement: Date range picker shall be accessible (WCAG 2.1 AA)
  Details: Keyboard navigation, screen reader compatible
  Acceptance Test: Accessibility audit, keyboard testing

DEPLOYMENT REQUIREMENTS

NFR-D1: Zero-Downtime Deployment
  Requirement: Feature deployable without customer-facing downtime
  Details: Feature flag enables gradual rollout; old code path available
  Acceptance Test: Deploy to production during business hours, monitor uptime

NFR-D2: Automated Rollback
  Requirement: Feature can be disabled instantly via feature flag
  Details: <1 minute to complete rollback if critical issue
  Acceptance Test: Test flag toggle during load, verify behavior change

NFR-D3: Database Migration Safety
  Requirement: SQL changes deployable without data loss or locking
  Details: Add new parameters to SPs without removing old ones
  Acceptance Test: SQL deployment tested in staging, verify backward compat

NFR-D4: Deployment Validation
  Requirement: Automated tests run post-deployment to validate functionality
  Details: Smoke tests, sanity checks for all 16 workflows
  Acceptance Test: Post-deployment test suite completes with 100% pass rate
```

---

## Instructions for Work Item Update

Copy each section above into the corresponding field in Azure DevOps work item AZD-2404320:

1. **Product Vision** → Click Edit, paste vision statement
2. **Objectives** → Paste 5 objectives (numbered 1-5)
3. **Key Results** → Paste 8 key results (KR1-KR8)
4. **Assumptions** → Paste 10 assumptions (numbered 1-10)
5. **Background** → Paste entire background section
6. **Acceptance Criteria** → Paste all 20 functional + NFR ACs with checkboxes
7. **Additional Information** → Paste all sections (artifacts, phases, test strategy, etc.)
8. **Risk Comment** → Paste risk matrix with all identified risks
9. **Non-Functional Requirements** → Paste all NFR categories with detailed requirements

**Alternative Method:** If copy/paste exceeds field character limits, you can:
- Link to this document (AZD_WI_2404320_WORKITEM_UPDATE_CONTENT.md) in the work item
- Create child work items for each major section
- Update fields incrementally over multiple edits
