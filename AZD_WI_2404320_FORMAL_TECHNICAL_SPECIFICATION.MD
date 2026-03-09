# Technical Specification: WI-2404320 (PME-451110)
## Date Range Filtering in Task Summary Widget

**Document Version:** 1.0  
**Status:** Ready for Development  
**Date:** March 9, 2026  
**Feature Owner:** Vijay Shankar  
**PME Owner:** Joan Salonga  
**Related Documents:**
- [Impact Analysis](AZD_WI_2404320_IMPACT_ANALYSIS.md)
- [Implementation Plan](AZD_WI_2404320_IMPLEMENTATION_PLAN.md)
- [LaunchDarkly Guide](AZD_WI_2404320_LAUNCHDARKLY_GUIDE.md)

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Requirements](#requirements)
3. [Architecture & Design](#architecture--design)
4. [Detailed Specifications](#detailed-specifications)
5. [Development Effort Estimates](#development-effort-estimates)
6. [Test Scenarios & Criteria](#test-scenarios--criteria)
7. [Acceptance Criteria](#acceptance-criteria)

---

## Executive Summary

### Problem Statement
The Dashboard "Task Summary" widget displays a date picker but enforces single-day queries due to hardcoded SQL logic. This prevents clients from filtering tasks by date ranges, blocking migration from Classic OneSite for Lindemann Multifamily Management LLC.

### Root Cause
The main stored procedure `uspGetTaskSummaryTileSummary_UPM` contains:
```sql
SET @LV_ENDDate = @LV_StartDate  -- HARDCODED: forces same day
```

### Solution Overview
Implement date range filtering across 5 layers:
1. **UI:** Add `rp-daterangepicker` component
2. **API:** Validate date range parameters; implement feature flag
3. **Manager/DAO:** Extract and pass both date range parameters
4. **Database:** Fix hardcoded logic; update 7 stored procedures
5. **Testing:** Comprehensive test coverage at all layers

### Success Criteria
- ✅ Users can select and filter by date ranges
- ✅ All 16 workflows support date filtering
- ✅ Backward compatible with legacy single-date queries
- ✅ Feature flag toggleable without deployment
- ✅ Performance: queries complete in < 2 seconds
- ✅ 0 critical bugs post-release
- ✅ Client successfully migrates from Classic OneSite

---

## Requirements

### Functional Requirements

#### FR1: Date Range Picker Component
**Requirement:** UI shall display a date range picker when server indicates date range filtering is available.

**Details:**
- Component: `rp-daterangepicker` (existing RealPage component)
- Display: Show start date and end date inputs
- Validation: Client-side validation (startDate ≤ endDate)
- Default Range: Last 30 days from current date
- Constraints: Maximum range 365 days

**Success Criteria:**
```
Given: showDateRangeFilter flag is true
When: User opens dashboard
Then: Date range picker is displayed instead of single date picker
And: Default range shows last 30 days
And: User can select any date range within 365 days
```

#### FR2: Date Range Parameter Passing
**Requirement:** UI shall pass date range as two separate parameters to API.

**Details:**
- Parameter 1: `dateFilterFrom` (epoch milliseconds string)
- Parameter 2: `dateFilterTo` (epoch milliseconds string)
- Format: String representation of epoch milliseconds (e.g., "1704067200000")
- Payload Structure: `filterBy` dictionary in FilterBy object

**Example:**
```json
{
  "filterBy": {
    "filter1": "",
    "filter2": "",
    "dateFilterFrom": "1704067200000",
    "dateFilterTo": "1704326400000"
  },
  "sortBy": {},
  "pages": {
    "startRow": 0,
    "recordsPerPage": 30
  }
}
```

#### FR3: API Date Range Validation
**Requirement:** API shall validate date range parameters and reject invalid requests.

**Validation Rules:**
- If `dateFilterFrom` and `dateFilterTo` provided:
  - Both must be valid epoch milliseconds
  - `dateFilterFrom` ≤ `dateFilterTo`
  - Range ≤ 365 days
- If only one date provided: Error
- If neither provided: Use legacy `dateFilter` (backward compatibility)

**Error Responses:**
```csharp
// Invalid format
HTTP 400
{
  "error": "Invalid date format. Expected epoch milliseconds as string."
}

// Invalid range
HTTP 400
{
  "error": "dateFilterFrom must be less than or equal to dateFilterTo"
}

// Range too large
HTTP 400
{
  "error": "Date range cannot exceed 365 days"
}
```

#### FR4: Backward Compatibility
**Requirement:** API shall support legacy single-date filtering without modification.

**Details:**
- Accept existing `dateFilter` parameter
- Treat single date as range: start=end=dateFilter
- Default behavior if no date parameters: use current date
- Priority: `dateFilterFrom`/`dateFilterTo` > `dateFilter`

**Logic Flow:**
```csharp
if (dateFilterFrom && dateFilterTo)
    // New: Use date range
else if (dateFilter)
    // Legacy: Use single date (start=end)
else
    // Default: Use current date
```

#### FR5: Feature Flag Control
**Requirement:** Feature shall be toggleable via LaunchDarkly without code deployment.

**Details:**
- Flag Key: `dashboards-daterangefilter-enabled`
- Type: Boolean
- Contexts: User, Property, Organization
- Default: OFF (false)
- Implementation: API-level only (controls `showDateRangeFilter` in response)

**Behavior:**
```csharp
if (featureFlagEnabled)
    return { showDateRangeFilter: true, dateFilterFrom: "...", dateFilterTo: "..." }
else
    return { showDateFilter: true, dateFilter: "..." }  // Legacy
```

#### FR6: Database Query Execution
**Requirement:** Database shall execute queries for specified date ranges across all 16 workflows.

**Details:**
- Update 7 stored procedures
- Use BETWEEN clause: `WHERE taskDate BETWEEN @startDate AND @endDate`
- Include entire end day: `endDate + 1 day - 1 second`
- Performance: Complete in < 2 seconds for typical query

**Affected Workflows (16 total):**
1. MIN (Move-in)
2. MOU (Move-out)
3. RFAS (Resident FAS)
4. ApplicantFAS
5. FAS-READY-APP
6. FAS-REJECTED
7. NRFAS
8. TRAN (Transfer)
9. TRANSOUT
10. GFR (Guarantor Form)
11. Appointment
12. FUP (Follow-up)
13. APPINPROG
14. PVR (Pending Virtual Refunds)
15. FVR (Failed Virtual Refunds)
16. CWL (Conventional Waitlist)

### Non-Functional Requirements

#### NFR1: Performance
- Query execution: < 2 seconds for typical datasets
- API response time: < 500ms (excluding database query)
- No performance degradation vs single-date queries
- Support 100K+ records in date range

#### NFR2: Backward Compatibility
- Existing clients using single-date filtering: unaffected
- API changes: additive only (no breaking changes)
- Database: support both legacy and new parameters

#### NFR3: Security
- Input validation: prevent SQL injection
- Date range validation: prevent DoS (>365 days)
- Authorization: existing property-level access controls unchanged

#### NFR4: Reliability
- Error handling: graceful fallback to legacy behavior
- Feature flag failure: default to OFF (safe state)
- Database failover: existing procedures remain functional

#### NFR5: Maintainability
- Code: clear comments explaining date range logic
- SQL: consistent naming (@dateFilterFrom, @dateFilterTo)
- Documentation: updated for new parameters

---

## Architecture & Design

### System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                      UI Layer (Angular)                     │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ tile-filters.component.html                          │   │
│  │ - tile-filters.component.ts                          │   │
│  │ - rp-daterangepicker (NEW)                           │   │
│  └──────────────────────────────────────────────────────┘   │
│                            ↓                                 │
├─────────────────────────────────────────────────────────────┤
│                  API/Controller Layer (.NET)                │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ DashboardTileExecutionV1Controller                   │   │
│  │ - ValidateDateRangeParameters()                      │   │
│  │ - Feature flag evaluation (LaunchDarkly)            │   │
│  └──────────────────────────────────────────────────────┘   │
│                            ↓                                 │
├─────────────────────────────────────────────────────────────┤
│              Manager/DAO Layer (.NET)                       │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ TileTemplateManager                                  │   │
│  │ TileTemplateDAO.SetFilterParameters() (UPDATED)      │   │
│  │ - Extract dateFilterFrom, dateFilterTo              │   │
│  │ - Build SQL parameters                              │   │
│  └──────────────────────────────────────────────────────┘   │
│                            ↓                                 │
├─────────────────────────────────────────────────────────────┤
│                  Database Layer (SQL)                       │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ uspGetTaskSummaryTileSummary_UPM (FIXED)             │   │
│  │ uspTaskSummaryDashboardCount_UPM (UPDATED)           │   │
│  │ + 5 other stored procedures (UPDATED)                │   │
│  │                                                      │   │
│  │ WHERE task_date BETWEEN @start AND @end             │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Data Flow Diagram

```
User selects date range: Jan 1 - Jan 31, 2024
                ↓
UI emits FilterBy event with:
  { dateFilterFrom: "1704067200000", dateFilterTo: "1704672000000" }
                ↓
API POST /dashboardTileExecution/Execute/TaskSummary/{id}
RequestParameter.FilterBy = { dateFilterFrom, dateFilterTo, ... }
                ↓
DashboardTileExecutionV1Controller.ExecuteTileTaskSummary()
  - Evaluate feature flag (LaunchDarkly)
  - Validate date range (dateFrom ≤ dateTo, ≤ 365 days)
  - Pass to TileTemplateManager
                ↓
TileTemplateManager.LoadTileTemplate()
  - Orchestrate data retrieval
  - Delegate to DAO
                ↓
TileTemplateDAO.SetFilterParameters()
  - Extract dateFilterFrom from FilterBy dict → @dateFilterFrom param
  - Extract dateFilterTo from FilterBy dict → @dateFilterTo param
  - Build SQL command with parameters
                ↓
TileTemplateDB.ExecuteCommand()
  - Execute SQL with parameters
  - Return results
                ↓
uspGetTaskSummaryTileSummary_UPM() -- SQL Stored Procedure
  - Receives @dateFilterFrom = 1704067200 (epoch seconds)
  - Receives @dateFilterTo = 1704672000 (epoch seconds)
  - NEW: IF @dateFilterFrom IS NOT NULL
           SET @LV_StartDate = CONVERT(DATETIME, @dateFilterFrom / 1000.0)
           SET @LV_ENDDate = CONVERT(DATETIME, @dateFilterTo / 1000.0) + 1 day
        ELSE
           Use @dateFilter (legacy) or GETDATE() (default)
  - Execute queries: WHERE task_date BETWEEN @LV_StartDate AND @LV_ENDDate
  - Return results for all 16 workflows
                ↓
Results → TileData object with all workflows:
  {
    "workflowCounts": {
      "MIN": 5,
      "MOU": 3,
      "RFAS": 2,
      "TRAN": 1,
      ...
    },
    "dateRange": {
      "from": "2024-01-01",
      "to": "2024-01-31",
      "totalDays": 31
    }
  }
                ↓
UI renders Task Summary widget with filtered results
```

### Component Interaction

```
tile-filters.component
  │
  ├─ Receives: filtersConfig (showDateRangeFilter flag)
  │
  ├─ Renders (conditional):
  │   ├─ If showDateRangeFilter=true:  rp-daterangepicker
  │   └─ If showDateFilter=true:       rp-datepicker (legacy)
  │
  ├─ User interaction:
  │   └─ onDateRangeChange(range) 
  │       ├─ Validate: startDate ≤ endDate
  │       ├─ Convert to epoch: range → {dateFilterFrom, dateFilterTo}
  │       └─ Emit: filterChange.emit({dateFilterFrom, dateFilterTo, ...})
  │
  └─ Parent (tile-wrap):
      ├─ Receives: filterChange event
      ├─ Updates: local filterBy dict
      ├─ Calls: API POST /dashboardTileExecution/Execute/TaskSummary/{id}
      └─ Payload: RequestParameter with full filterBy dict
```

---

## Detailed Specifications

### Phase 1: Frontend Specification

#### 1.1 HTML Template Changes
**File:** `tile-filters.component.html`

**Requirement:** Display date range picker when feature flag indicates support.

**Code:**
```html
<!-- Existing code unchanged -->
<div *ngIf="filtersConfig.showFilter1" class="filter-container">
  <label>{{ 'FILTER_1' | translate }}</label>
  <select [(ngModel)]="filterBy.filter1" (change)="onFilterChange()">
    <!-- options -->
  </select>
</div>

<!-- NEW: Date Range Picker -->
<div *ngIf="filtersConfig.showDateRangeFilter" 
     class="date-range-filter-container">
  <label>{{ 'DATE_RANGE_FILTER' | translate }}</label>
  <rp-daterangepicker
    [startDate]="filterBy.dateFilterFrom | convertFromEpochMs"
    [endDate]="filterBy.dateFilterTo | convertFromEpochMs"
    [minDate]="dateRangeConfig.minDate"
    [maxDate]="dateRangeConfig.maxDate"
    (rangeChanged)="onDateRangeChange($event)"
    [disabled]="isLoadingData">
  </rp-daterangepicker>
  <small class="helper-text">
    {{ 'MAX_365_DAYS' | translate }}
  </small>
</div>

<!-- LEGACY: Single Date Picker (backward compat) -->
<div *ngIf="filtersConfig.showDateFilter && !filtersConfig.showDateRangeFilter" 
     class="date-filter-container">
  <label>{{ 'DATE_FILTER' | translate }}</label>
  <rp-datepicker 
    [selectedDate]="filterBy.dateFilter | convertFromEpochMs"
    (dateChanged)="onDateFilterChange($event)"
    [disabled]="isLoadingData">
  </rp-datepicker>
</div>
```

**Pipe Definition (convertFromEpochMs):**
```typescript
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'convertFromEpochMs'
})
export class ConvertFromEpochMsPipe implements PipeTransform {
  transform(epochMs: string | number): Date | null {
    if (!epochMs) return null;
    const num = typeof epochMs === 'string' ? parseInt(epochMs, 10) : epochMs;
    if (isNaN(num)) return null;
    return new Date(num);
  }
}
```

#### 1.2 Component Logic Changes
**File:** `tile-filters.component.ts`

**Requirement:** Manage date range state and validation.

**Code:**
```typescript
import { Component, OnInit, Input, Output, EventEmitter } from '@angular/core';
import { RpDaterangeInterface } from '@rp/shared/daterange';
import { Filters, FiltersConfig } from './tile-filters.interface';

interface DateRangeConfig {
  minDate: Date;
  maxDate: Date;
  maxRangeDays: number;
}

@Component({
  selector: 'app-tile-filters',
  templateUrl: './tile-filters.component.html',
  styleUrls: ['./tile-filters.component.css']
})
export class TileFiltersComponent implements OnInit {
  @Input() filtersConfig: FiltersConfig;
  @Input() filterBy: Filters;
  @Input() filterOptions: any;
  @Output() filterChange = new EventEmitter<Filters>();
  
  dateRangeConfig: DateRangeConfig;
  isLoadingData = false;
  private readonly MAX_RANGE_DAYS = 365;
  private readonly DEFAULT_RANGE_DAYS = 30;

  constructor(private logger: LoggerService) { }

  ngOnInit(): void {
    this.initializeFilters();
    this.initializeDateRangeConfig();
  }

  /**
   * Initialize filter values from incoming data
   */
  private initializeFilters(): void {
    if (!this.filterBy) {
      this.filterBy = {};
    }

    // Initialize date range from incoming data or defaults
    if (this.filtersConfig.showDateRangeFilter && !this.filterBy.dateFilterFrom) {
      const endDate = new Date();
      const startDate = new Date(endDate.getTime() - this.DEFAULT_RANGE_DAYS * 24 * 60 * 60 * 1000);

      this.filterBy.dateFilterFrom = this.dateToEpochMs(startDate);
      this.filterBy.dateFilterTo = this.dateToEpochMs(endDate);
    }
  }

  /**
   * Initialize date range configuration (min/max dates)
   */
  private initializeDateRangeConfig(): void {
    const today = new Date();
    const twoYearsAgo = new Date(today.getFullYear() - 2, today.getMonth(), today.getDate());

    this.dateRangeConfig = {
      minDate: twoYearsAgo,
      maxDate: today,
      maxRangeDays: this.MAX_RANGE_DAYS
    };
  }

  /**
   * Handles single date filter change (legacy)
   */
  onDateFilterChange(epochMs: string): void {
    this.filterBy.dateFilter = epochMs;
    
    // Remove date range fields if switching to single date mode
    delete this.filterBy.dateFilterFrom;
    delete this.filterBy.dateFilterTo;

    this.logger.debug(`Single date filter changed: ${epochMs}`);
    this.emitFilterChange();
  }

  /**
   * Handles date range change (new feature)
   * 
   * @param range RpDaterangeInterface with startDate and endDate
   */
  onDateRangeChange(range: RpDaterangeInterface): void {
    // Null check
    if (!range || !range.startDate || !range.endDate) {
      this.logger.warn('Invalid date range received');
      return;
    }

    const startDate = this.getNormalizedDate(range.startDate);
    const endDate = this.getNormalizedDate(range.endDate);

    // Validation: start <= end
    if (startDate > endDate) {
      this.logger.error('Invalid range: startDate > endDate');
      this.showErrorNotification('Invalid date range. Start date must be before end date.');
      return;
    }

    // Validation: range <= 365 days
    const rangeDays = Math.ceil((endDate.getTime() - startDate.getTime()) / (24 * 60 * 60 * 1000));
    if (rangeDays > this.MAX_RANGE_DAYS) {
      this.logger.error(`Range exceeds ${this.MAX_RANGE_DAYS} days: ${rangeDays} days`);
      this.showErrorNotification(`Date range cannot exceed ${this.MAX_RANGE_DAYS} days.`);
      return;
    }

    // Convert to epoch milliseconds
    this.filterBy.dateFilterFrom = this.dateToEpochMs(startDate);
    this.filterBy.dateFilterTo = this.dateToEpochMs(endDate);

    // Remove legacy single date field
    delete this.filterBy.dateFilter;

    this.logger.debug(
      `Date range changed: ${startDate.toISOString()} to ${endDate.toISOString()} ` +
      `(${rangeDays} days)`);

    this.emitFilterChange();
  }

  /**
   * Handles filter1 or filter2 change
   */
  onFilterChange(): void {
    this.logger.debug('Filter changed');
    this.emitFilterChange();
  }

  /**
   * Emits filter change event to parent component
   */
  private emitFilterChange(): void {
    this.filterChange.emit({
      ...this.filterBy
    });
  }

  /**
   * Converts Date object to epoch milliseconds string
   */
  private dateToEpochMs(date: Date): string {
    return Math.floor(date.getTime()).toString();
  }

  /**
   * Normalizes date input (handles Date objects and numbers)
   */
  private getNormalizedDate(value: Date | number | string): Date {
    if (value instanceof Date) {
      return value;
    }
    if (typeof value === 'number') {
      return new Date(value);
    }
    if (typeof value === 'string') {
      const num = parseInt(value, 10);
      return new Date(num);
    }
    return new Date();
  }

  /**
   * Show error notification to user
   */
  private showErrorNotification(message: string): void {
    // Would integrate with ToastrService or similar
    this.logger.error(message);
  }

  /**
   * Mark component as loading
   */
  setLoading(isLoading: boolean): void {
    this.isLoadingData = isLoading;
  }
}
```

#### 1.3 Interface Specification
**File:** `tile-filters.interface.ts`

**Requirement:** Define types for filter data structures.

**Code:**
```typescript
/**
 * Represents a date range selection
 */
export interface RpDaterangeInterface {
  startDate: Date | number | string;
  endDate: Date | number | string;
}

/**
 * Filter values from user selections
 */
export interface Filters {
  // Legacy (single date mode)
  dateFilter?: string;           // epoch milliseconds

  // New (date range mode)
  dateFilterFrom?: string;        // epoch milliseconds
  dateFilterTo?: string;          // epoch milliseconds

  // Other filters
  filter1?: string;
  filter2?: string;

  // Transient UI state (not sent to API)
  dateRange?: RpDaterangeInterface;
}

/**
 * Configuration controlling which filters are visible
 */
export interface FiltersConfig {
  showFilter1?: boolean;          // Show first select filter
  showFilter2?: boolean;          // Show second select filter
  showDateFilter?: boolean;       // Show single date picker (legacy)
  showDateRangeFilter?: boolean;  // Show date range picker (new)
}
```

#### 1.4 Module Registration
**File:** `tile-filters.module.ts`

**Requirement:** Import required dependencies.

**Code:**
```typescript
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';
import { TranslateModule } from '@ngx-translate/core';

import { RpDatepickerModule } from '@rp/shared/datepicker';
import { RpDaterangepickerModule } from '@rp/shared/daterangepicker';

import { TileFiltersComponent } from './tile-filters.component';
import { ConvertFromEpochMsPipe } from './pipes/convert-from-epoch-ms.pipe';

@NgModule({
  declarations: [
    TileFiltersComponent,
    ConvertFromEpochMsPipe
  ],
  imports: [
    CommonModule,
    FormsModule,
    TranslateModule,
    RpDatepickerModule,
    RpDaterangepickerModule
  ],
  exports: [
    TileFiltersComponent
  ]
})
export class TileFiltersModule { }
```

---

### Phase 2: API Specification

#### 2.1 Controller Method Specification
**File:** `DashboardTileExecutionV1Controller.cs`

**Requirement:** Handle date range parameters with validation and feature flag.

**Pre-conditions:**
- User is authenticated
- User has property access
- RequestParameter is valid JSON

**Post-conditions:**
- Date range validated
- Feature flag evaluated
- Legacy parameters removed if flag disabled
- TileData returned

**Code:**

```csharp
using LaunchDarkly.Sdk;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using System;
using System.Collections.Generic;
using System.Threading.Tasks;

namespace RealPage.API.OneSite.Dashboard.Controllers
{
    /// <summary>
    /// Handles dashboard tile execution requests
    /// </summary>
    [ApiController]
    [Route("api/dashboard/v{api-version:apiVersion}")]
    [Authorize]
    public class DashboardTileExecutionV1Controller : ControllerBase
    {
        private readonly ITileTemplateManager _tileTemplateManager;
        private readonly IFeatureFlagService _featureFlagService;
        private readonly ILogger<DashboardTileExecutionV1Controller> _logger;

        public DashboardTileExecutionV1Controller(
            ITileTemplateManager tileTemplateManager,
            IFeatureFlagService featureFlagService,
            ILogger<DashboardTileExecutionV1Controller> logger)
        {
            _tileTemplateManager = tileTemplateManager;
            _featureFlagService = featureFlagService ?? throw new ArgumentNullException(nameof(featureFlagService));
            _logger = logger ?? throw new ArgumentNullException(nameof(logger));
        }

        /// <summary>
        /// Executes a TaskSummary tile with date range filtering support
        /// </summary>
        /// <param name="tileID">The tile configuration ID</param>
        /// <param name="parameters">Request parameters including filters</param>
        /// <returns>Populated tile data with workflow counts</returns>
        /// <response code="200">Tile data successfully retrieved</response>
        /// <response code="400">Invalid request parameters (date validation failure)</response>
        /// <response code="401">User not authenticated</response>
        /// <response code="500">Internal server error</response>
        [HttpPost]
        [Route("dashboardTileExecution/Execute/TaskSummary/{tileID}")]
        [ProducesResponseType(typeof(TileData), 200)]
        [ProducesResponseType(typeof(ErrorResponse), 400)]
        [ProducesResponseType(401)]
        [ProducesResponseType(typeof(ErrorResponse), 500)]
        public async Task<IActionResult> ExecuteTileTaskSummary(
            int tileID,
            [FromBody] RequestParameter parameters)
        {
            var executionId = Guid.NewGuid().ToString();

            try
            {
                _logger.LogInformation($"[{executionId}] TaskSummary execution started (tileID={tileID})");

                // ========== FEATURE FLAG EVALUATION ==========
                var userContext = LaunchDarklyContextBuilder.BuildUserContext(User);
                var propertyContext = LaunchDarklyContextBuilder.BuildPropertyContext(tileID);

                var dateRangeEnabled = _featureFlagService.IsFlagEnabled(
                    "dashboards-daterangefilter-enabled",
                    userContext,
                    propertyContext);

                _logger.LogInformation(
                    $"[{executionId}] Feature flag 'dashboards-daterangefilter-enabled': {dateRangeEnabled}");

                // ========== PARAMETER VALIDATION & NORMALIZATION ==========
                var validationResult = ValidateDateRangeParameters(parameters, dateRangeEnabled);

                if (!validationResult.IsValid)
                {
                    _logger.LogWarning(
                        $"[{executionId}] Validation failed: {validationResult.ErrorMessage}");

                    return BadRequest(new ErrorResponse
                    {
                        Error = validationResult.ErrorMessage,
                        ExecutionId = executionId
                    });
                }

                // ========== LOAD TILE DATA ==========
                var tileData = await _tileTemplateManager.LoadTileTemplate(tileID, parameters);

                _logger.LogInformation(
                    $"[{executionId}] TaskSummary execution completed. " +
                    $"Workflows: {tileData?.WorkflowCounts?.Count ?? 0}");

                return Ok(tileData);
            }
            catch (ArgumentException aex)
            {
                _logger.LogWarning($"[{executionId}] Argument validation error: {aex.Message}");
                return BadRequest(new ErrorResponse
                {
                    Error = aex.Message,
                    ExecutionId = executionId
                });
            }
            catch (Exception ex)
            {
                _logger.LogError($"[{executionId}] Unhandled error: {ex.GetType().Name} - {ex.Message}");
                return StatusCode(500, new ErrorResponse
                {
                    Error = "An error occurred processing the request",
                    ExecutionId = executionId
                });
            }
        }

        /// <summary>
        /// Validates date range parameters from request
        /// </summary>
        /// <param name="parameters">Request parameters to validate</param>
        /// <param name="dateRangeEnabled">Whether date range feature is enabled</param>
        /// <returns>Validation result with any errors</returns>
        private ValidationResult ValidateDateRangeParameters(
            RequestParameter parameters,
            bool dateRangeEnabled)
        {
            // No filters provided
            if (parameters?.FilterBy == null || parameters.FilterBy.Count == 0)
            {
                return ValidationResult.Success();
            }

            var filterBy = parameters.FilterBy;
            var hasDateFilterFrom = filterBy.TryGetValue("dateFilterFrom", out var fromValue);
            var hasDateFilterTo = filterBy.TryGetValue("dateFilterTo", out var toValue);

            // If feature flag is OFF, remove date range parameters
            if (!dateRangeEnabled && (hasDateFilterFrom || hasDateFilterTo))
            {
                _logger.LogInformation("Feature flag disabled: removing date range parameters");
                filterBy.Remove("dateFilterFrom");
                filterBy.Remove("dateFilterTo");
                return ValidationResult.Success();
            }

            // Validate date range if both parameters present
            if (hasDateFilterFrom && hasDateFilterTo)
            {
                return ValidateDateRange(fromValue, toValue);
            }

            // If only one date parameter provided, error
            if (hasDateFilterFrom || hasDateFilterTo)
            {
                return ValidationResult.Failure("Both dateFilterFrom and dateFilterTo must be provided together");
            }

            return ValidationResult.Success();
        }

        /// <summary>
        /// Validates a date range (both parameters must be present and valid)
        /// </summary>
        private ValidationResult ValidateDateRange(string fromValue, string toValue)
        {
            // Parse from date
            if (!long.TryParse(fromValue, out var fromEpoch))
            {
                return ValidationResult.Failure(
                    "Invalid dateFilterFrom format. Expected epoch milliseconds as string.");
            }

            // Parse to date
            if (!long.TryParse(toValue, out var toEpoch))
            {
                return ValidationResult.Failure(
                    "Invalid dateFilterTo format. Expected epoch milliseconds as string.");
            }

            // Validate from <= to
            if (fromEpoch > toEpoch)
            {
                return ValidationResult.Failure(
                    "dateFilterFrom must be less than or equal to dateFilterTo");
            }

            // Validate range <= 365 days
            const long MaxRangeMs = 365L * 24 * 60 * 60 * 1000;  // 365 days in ms
            if ((toEpoch - fromEpoch) > MaxRangeMs)
            {
                return ValidationResult.Failure(
                    "Date range cannot exceed 365 days");
            }

            return ValidationResult.Success();
        }
    }

    /// <summary>
    /// Result of parameter validation
    /// </summary>
    internal class ValidationResult
    {
        public bool IsValid { get; set; }
        public string ErrorMessage { get; set; }

        public static ValidationResult Success() => new ValidationResult { IsValid = true };

        public static ValidationResult Failure(string message) =>
            new ValidationResult { IsValid = false, ErrorMessage = message };
    }

    /// <summary>
    /// Error response returned to client
    /// </summary>
    public class ErrorResponse
    {
        public string Error { get; set; }
        public string ExecutionId { get; set; }
    }
}
```

#### 2.2 DAO Layer Specification
**File:** `TileTemplateDAO.cs`

**Requirement:** Extract date range parameters and build SQL command.

**Code:**
```csharp
/// <summary>
/// Sets SQL parameters from FilterBy dictionary
/// Supports both new date range and legacy single date parameters
/// </summary>
private void SetFilterParameters(IDbCommand cmd, Dictionary<string, string> filterBy)
{
    if (filterBy == null || filterBy.Count == 0)
    {
        return;
    }

    // Extract known parameters
    var hasDateFilterFrom = filterBy.TryGetValue("dateFilterFrom", out var dateFilterFrom);
    var hasDateFilterTo = filterBy.TryGetValue("dateFilterTo", out var dateFilterTo);
    var hasDateFilter = filterBy.TryGetValue("dateFilter", out var dateFilter);

    // ========== NEW: DATE RANGE PARAMETERS (Priority) ==========
    if (hasDateFilterFrom && !string.IsNullOrEmpty(dateFilterFrom))
    {
        var param = cmd.CreateParameter();
        param.ParameterName = "@dateFilterFrom";
        param.Value = dateFilterFrom;
        param.DbType = DbType.String;
        cmd.Parameters.Add(param);

        _logger.LogDebug($"Added parameter: @dateFilterFrom = {dateFilterFrom}");
    }

    if (hasDateFilterTo && !string.IsNullOrEmpty(dateFilterTo))
    {
        var param = cmd.CreateParameter();
        param.ParameterName = "@dateFilterTo";
        param.Value = dateFilterTo;
        param.DbType = DbType.String;
        cmd.Parameters.Add(param);

        _logger.LogDebug($"Added parameter: @dateFilterTo = {dateFilterTo}");
    }

    // ========== LEGACY: SINGLE DATE PARAMETER (Fallback) ==========
    // Only add if date range not provided (backward compatibility)
    if (!hasDateFilterFrom && !hasDateFilterTo && hasDateFilter && !string.IsNullOrEmpty(dateFilter))
    {
        var param = cmd.CreateParameter();
        param.ParameterName = "@dateFilter";
        param.Value = dateFilter;
        param.DbType = DbType.String;
        cmd.Parameters.Add(param);

        _logger.LogDebug($"Added parameter: @dateFilter (legacy) = {dateFilter}");
    }

    // ========== OTHER FILTERS ==========
    foreach (var kvp in filterBy)
    {
        // Skip parameters already handled
        if (kvp.Key == "dateFilter" || kvp.Key == "dateFilterFrom" || kvp.Key == "dateFilterTo")
        {
            continue;
        }

        if (!string.IsNullOrEmpty(kvp.Value))
        {
            var param = cmd.CreateParameter();
            param.ParameterName = $"@{kvp.Key}";
            param.Value = kvp.Value;
            param.DbType = DbType.String;
            cmd.Parameters.Add(param);

            _logger.LogDebug($"Added parameter: @{kvp.Key} = {kvp.Value}");
        }
    }
}
```

---

### Phase 3: Database Specification

#### 3.1 Main Stored Procedure Fix
**File:** `uspGetTaskSummaryTileSummary_UPM.sql`

**Requirement:** Accept and process date range parameters.

**Key Changes:**
1. Add new parameters: `@dateFilterFrom`, `@dateFilterTo`
2. Replace hardcoded logic: `SET @LV_ENDDate = @LV_StartDate`
3. Implement conditional logic for date range vs single date
4. Use BETWEEN in WHERE clauses

**SQL Snippet:**
```sql
ALTER PROCEDURE [dbo].[uspGetTaskSummaryTileSummary_UPM]
    @PropertyID INT,
    @dateFilter VARCHAR(MAX) = NULL,           -- Legacy
    @dateFilterFrom VARCHAR(MAX) = NULL,       -- NEW
    @dateFilterTo VARCHAR(MAX) = NULL,         -- NEW
    @SiteID INT = NULL,
    @ShowServiceRequest BIT = 0
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @LV_StartDate DATETIME
    DECLARE @LV_ENDDate DATETIME
    DECLARE @LV_ErrorMsg NVARCHAR(MAX)

    BEGIN TRY
        -- ========== DATE RANGE LOGIC ==========
        -- Priority 1: New date range parameters
        IF @dateFilterFrom IS NOT NULL AND @dateFilterTo IS NOT NULL
        BEGIN
            SET @LV_StartDate = CONVERT(DATETIME, @dateFilterFrom / 1000.0, 121)
            SET @LV_ENDDate = CONVERT(DATETIME, @dateFilterTo / 1000.0, 121)

            -- Include entire end day (add 1 day, subtract 1 second)
            SET @LV_ENDDate = DATEADD(DAY, 1, @LV_ENDDate)
            SET @LV_ENDDate = DATEADD(SECOND, -1, @LV_ENDDate)

            -- Validate range
            IF @LV_StartDate > @LV_ENDDate
            BEGIN
                SET @LV_ErrorMsg = 'Invalid date range: dateFilterFrom must be <= dateFilterTo'
                RAISERROR(@LV_ErrorMsg, 16, 1)
            END

            PRINT 'Using date range: ' + FORMAT(@LV_StartDate, 'yyyy-MM-dd HH:mm:ss') + 
                  ' to ' + FORMAT(@LV_ENDDate, 'yyyy-MM-dd HH:mm:ss')
        END
        -- Priority 2: Legacy single date
        ELSE IF @dateFilter IS NOT NULL
        BEGIN
            SET @LV_StartDate = CONVERT(DATETIME, @dateFilter / 1000.0, 121)
            SET @LV_EndDate = @LV_StartDate

            -- Include entire day
            SET @LV_ENDDate = DATEADD(DAY, 1, @LV_ENDDate)
            SET @LV_ENDDate = DATEADD(SECOND, -1, @LV_ENDDate)

            PRINT 'Using legacy single date: ' + FORMAT(@LV_StartDate, 'yyyy-MM-dd')
        END
        -- Priority 3: Default to current date
        ELSE
        BEGIN
            SET @LV_StartDate = CAST(GETDATE() AS DATE)
            SET @LV_ENDDate = DATEADD(DAY, 1, CAST(GETDATE() AS DATE))
            SET @LV_ENDDate = DATEADD(SECOND, -1, @LV_ENDDate)

            PRINT 'Using current date: ' + FORMAT(CAST(GETDATE() AS DATE), 'yyyy-MM-dd')
        END

        -- ========== QUERY EXECUTION WITH DATE RANGE ==========
        SELECT 
            'TaskSummary' AS TileType,
            a.workflow_code,
            COUNT(*) AS TaskCount,
            MAX(a.task_updated_date) AS LastUpdatedDate
        FROM TaskSummaryView a
        WHERE a.PropertyID = @PropertyID
          AND a.task_date >= @LV_StartDate
          AND a.task_date < @LV_ENDDate
        GROUP BY a.workflow_code
        ORDER BY a.workflow_code

    END TRY
    BEGIN CATCH
        DECLARE @ErrorNumber INT = ERROR_NUMBER()
        DECLARE @ErrorSeverity INT = ERROR_SEVERITY()
        DECLARE @ErrorState INT = ERROR_STATE()

        SET @LV_ErrorMsg = 'Error in uspGetTaskSummaryTileSummary_UPM: ' + ERROR_MESSAGE()

        RAISERROR(@LV_ErrorMsg, @ErrorSeverity, @ErrorState)

        -- Log error for diagnostics
        INSERT INTO ErrorLog (ProcedureName, ErrorMessage, ErrorNumber)
        VALUES (
            'uspGetTaskSummaryTileSummary_UPM',
            @LV_ErrorMsg,
            @ErrorNumber
        )
    END CATCH
END
```

---

#### 3.2 Complete Stored Procedure Date Range Analysis for All 16 Workflows

**Reference Document:** [WI-2404320_COMPLETE_STORED_PROCEDURE_DATE_ANALYSIS.md](WI-2404320_COMPLETE_STORED_PROCEDURE_DATE_ANALYSIS.md)

This section provides a comprehensive breakdown of all 16 workflows displayed in the Task Summary widget and their date range filtering implementation requirements.

##### 3.2.1 Workflow Status Summary

| # | Workflow Code | Workflow Name | Current Status | Recommended Approach | Priority | Implementation |
|---|---|---|---|---|---|---|
| 1 | MIN | Move-in | ⚠️ Partial | Add leaMoveInDate BETWEEN clause | High | 1 line |
| 2 | MOU | Move-out | ⚠️ Partial | Expand leaNoticeForDate BETWEEN | High | 1 line |
| 3 | RFAS | Resident FAS | ⚠️ Partial | Add leaEndDate BETWEEN clause | High | 1 line |
| 4 | ApplicantFAS | Applicant FAS | ⚠️ Partial | Add leaEndDate BETWEEN clause | High | 1 line |
| 5 | FAS-PEND | FAS Pending | ⚠️ Partial | Add leaEndDate BETWEEN clause | High | 1 line |
| 6 | NRFAS | Non-Resident FAS | ⚠️ Partial | Add FSRH.CreatedDate BETWEEN | High | 1 line |
| 7 | TRAN | Transfer | ✅ Complete | ALREADY USED BETWEEN | None | No change |
| 8 | TRANSOUT | Transfer Out | ✅ Complete | ALREADY USED BETWEEN | None | No change |
| 9 | GFR | Guarantor Form Required | ❌ Zero | Modify uspGuarantorTaskListSelectCount_UPM | High | 5-10 lines |
| 10 | Appointment | Appointment | ❌ Zero* | Fix @LV_StartDate/@LV_ENDDate init (CRITICAL BUG) | **CRITICAL** | 2 lines |
| 11 | FUP | Follow-up | ❌ Zero* | Fix @LV_StartDate/@LV_ENDDate init (CRITICAL BUG) | **CRITICAL** | 2 lines |
| 12 | APPINPROG | Application in Progress | ❌ Zero | Modify uspTasksLGAppsInProgressCountSelect_UPM | High | 5-10 lines |
| 13 | FAS-READY-APP | FAS Ready for Approval | ❌ Zero | Modify uspFASRefundCountSelect_UPM | High | 5-10 lines |
| 14 | FAS-REJECTED | FAS Rejected | ❌ Zero | Modify uspFASRefundCountSelect_UPM | High | 5-10 lines |
| 15 | PVR | Pending Virtual Refunds | ❌ Zero | Add RP.CreatedDate BETWEEN inline | Medium | 1 line |
| 16 | FVR | Failed Virtual Refunds | ❌ Zero | Add RP.CreatedDate BETWEEN inline | Medium | 1 line |
| 17 | Waitlist | Conventional Waitlist | ❌ Zero | Modify uspWaitListSelectTiles_UPM | Medium | 5-10 lines |

**Legend:** ✅ = Working | ⚠️ = Partial | ❌ = Not Working | * = Receives NULL dates from uninitialized variables

##### 3.2.2 Critical Bugs Identified

**BUG #1: Undefined Variables in uspTaskSummaryDashboardCount_UPM**
```sql
-- LINES 278-279: Variables declared but NEVER initialized
DECLARE @LV_StartDate VARCHAR(10);     -- ⚠️ NEVER INITIALIZED!
DECLARE @LV_ENDDate VARCHAR(10);       -- ⚠️ NEVER INITIALIZED!

-- LINES 331-333: These NULL values passed to procedures
SET @sql = N'P' + @InternalEntityID + N'.DBO.uspNPSTodayTaskListCountSelect_UPM
    ...@ENDDATE = ''' + @LV_ENDDate + N''',      -- ⚠️ PASSING NULL!
    ...@STARTDATE = ''' + @LV_StartDate + N''',  -- ⚠️ PASSING NULL!
```

**Impact:** Appointment and Follow-up counts ignore date range  
**Fix:**
```sql
SET @LV_StartDate = @IPVC_STARTDATE;    -- Initialize from parameter
SET @LV_ENDDate = @IPVC_ENDDATE;        -- Initialize from parameter
```
**Effort:** 2 lines | **Risk:** Low

---

**BUG #2: MOU Query Missing BETWEEN Clause**
```sql
-- CURRENT (BUGGY): leaNoticeForDate checked for NOT NULL but no date filtering
AND L.leaNoticeForDate IS NOT NULL
AND L.leaTransferCompleteNewBit IS NULL
AND r.[Status] = 'Pending Move-outs'

-- RECOMMENDED FIX:
AND L.leaNoticeForDate BETWEEN @LD_STARTDATE AND @LD_ENDDATE
AND L.leaTransferCompleteNewBit IS NULL
AND r.[Status] = 'Pending Move-outs'
```

**Impact:** Move-out counts not filtered by expected move-out date  
**Effort:** 1 line | **Risk:** Low

---

##### 3.2.2.1 Additional Critical Bugs: Stored Procedures Missing Date Parameters

Beyond the two inline SQL bugs above, comprehensive analysis identified 5 additional stored procedures that require updates to support date range filtering. These procedures are called from `uspTaskSummaryDashboardCount_UPM` but lack date parameters entirely.

**BUG #3: uspFASRefundCountSelect_UPM Missing Date Parameters (Affects: FAS-READY-APP, FAS-REJECTED)**

**File:** `DATABASESETUP\P2\procs\UPM\DashboardTiles\TaskSummaryCount\uspFASRefundCountSelect_UPM.sql`

**Current Procedure Signature:**
```sql
CREATE PROCEDURE [dbo].uspFASRefundCountSelect_UPM 
(
    @InternalEntityID VARCHAR(10),
    @InternalUserID VARCHAR(10),
    @InternalSiteID VARCHAR(10),
    @RESNAME VARCHAR(30)=NULL,
    @UnitId VARCHAR(10)=NULL,
    @FASStatus VARCHAR(1)='0',
    @ResType VARCHAR(10)=NULL,
    @FASApprovedBit VARCHAR(1)='0',      -- Only distinguishes approval vs rejection
    @UnitTo VARCHAR(10)=NULL,
    @TASKCOUNT INT OUTPUT
)
```

**Issue:** 
- Called from uspTaskSummaryDashboardCount_UPM (lines 340-356)
- No date parameters accepted
- Cannot filter FAS approval/rejection by date range
- Affects 2 workflows: FAS-READY-APP (status 0) and FAS-REJECTED (status 2)

**Required Changes:**
1. Add parameters:
   ```sql
   @IPVC_STARTDATE VARCHAR(10) = NULL,
   @IPVC_ENDDATE VARCHAR(10) = NULL
   ```
2. Convert to DATETIME inside procedure
3. Add BETWEEN clause to WHERE condition filtering on FAS approval/rejection date
4. Handle NULL dates (default to current period)

**Impact:** FAS approval/rejection counts currently not date-filtered  
**Effort:** 5-10 lines (add params + 1-2 BETWEEN clauses) | **Risk:** Medium (must identify correct date columns)

**Call Site (uspTaskSummaryDashboardCount_UPM):**
```sql
-- Line 340-341:
DECLARE @FASApproval INT = 0;
EXEC dbo.[uspFASRefundCountSelect_UPM]
    @InternalEntityID = @InternalEntityID,
    @InternalUserID = @InternalUserID,
    @InternalSiteID = @InternalSiteID,
    @FASApprovedBit = '0',               -- 0 = ready for approval
    @TASKCOUNT = @FASApproval OUTPUT;
-- MISSING: @IPVC_STARTDATE, @IPVC_ENDDATE

-- Line 346-354:
DECLARE @FASRejected INT = 0;
EXEC dbo.[uspFASRefundCountSelect_UPM]
    @InternalEntityID = @InternalEntityID,
    @InternalUserID = @InternalUserID,
    @InternalSiteID = @InternalSiteID,
    @FASApprovedBit = '2',               -- 2 = rejected
    @TASKCOUNT = @FASRejected OUTPUT;
-- MISSING: @IPVC_STARTDATE, @IPVC_ENDDATE
```

---

**BUG #4: uspGuarantorTaskListSelectCount_UPM Missing Date Parameters (Affects: GFR)**

**File:** `DATABASESETUP\P2\procs\UPM\DashboardTiles\TaskSummaryCount\uspGuarantorTaskListSelectCount_UPM.sql`

**Current Procedure Signature:**
```sql
CREATE PROCEDURE [dbo].[uspGuarantorTaskListSelectCount_UPM]
    @InternalEntityID VARCHAR(10),
    @InternalUserID VARCHAR(10),
    @InternalSiteID VARCHAR(10),
    @TASKCOUNT INT OUTPUT
```

**Issue:**
- Called from uspTaskSummaryDashboardCount_UPM (lines 335-338)
- No date parameters accepted
- Cannot filter guarantor form requirements by date range
- Affects 1 workflow: GFR (Guarantor Form Required)

**Required Changes:**
1. Add parameters:
   ```sql
   @IPVC_STARTDATE VARCHAR(10) = NULL,
   @IPVC_ENDDATE VARCHAR(10) = NULL
   ```
2. Pass dates to procedure logic
3. Add BETWEEN clause to WHERE condition filtering on form due/demand date
4. Handle NULL dates

**Impact:** Guarantor form counts currently not date-filtered  
**Effort:** 5-10 lines | **Risk:** Medium

**Call Site (uspTaskSummaryDashboardCount_UPM, lines 335-338):**
```sql
DECLARE @GuarantorFormReq INT = 0;
EXEC dbo.[uspGuarantorTaskListSelectCount_UPM]
    @InternalEntityID = @InternalEntityID,
    @InternalUserID = @InternalUserID,
    @InternalSiteID = @InternalSiteID,
    @TASKCOUNT = @GuarantorFormReq OUTPUT;
-- MISSING: @IPVC_STARTDATE, @IPVC_ENDDATE
```

---

**BUG #5: uspTasksLGAppsInProgressCountSelect_UPM Missing Date Parameters (Affects: APPINPROG)**

**File:** Property-specific database (P + @InternalEntityID)  
**Stored Procedure Name:** `[Pxxx].[dbo].[uspTasksLGAppsInProgressCountSelect_UPM]` (where xxx = EntityID)

**Current Procedure Call (uspTaskSummaryDashboardCount_UPM, lines 310-316):**
```sql
DECLARE @ApplicationInProgress INT = 0;
SET @sql = N'P' + @InternalEntityID + N'.DBO.[uspTasksLGAppsInProgressCountSelect_UPM]
    @SITEID = ' + @InternalSiteID + N',
    @USERID = '''',
    @TASKCOUNT = @ApplicationInProgress OUTPUT';

EXEC sp_executesql @sql, N'@ApplicationInProgress INT OUTPUT', @ApplicationInProgress OUTPUT;
-- MISSING: @IPVC_STARTDATE, @IPVC_ENDDATE in dynamic SQL
```

**Issue:**
- Called with dynamic SQL (sp_executesql)
- No date parameters passed
- Cannot filter application status by date range
- Affects 1 workflow: APPINPROG (Application in Progress)

**Required Changes:**
1. Modify dynamic SQL to include date parameters:
   ```sql
   SET @sql = N'P' + @InternalEntityID + N'.DBO.[uspTasksLGAppsInProgressCountSelect_UPM]
       @SITEID = ' + @InternalSiteID + N',
       @USERID = '''',
       @IPVC_STARTDATE = ''' + @IPVC_STARTDATE + N''',    -- ADD
       @IPVC_ENDDATE = ''' + @IPVC_ENDDATE + N''',        -- ADD
       @TASKCOUNT = @ApplicationInProgress OUTPUT';
   ```
2. Modify target procedure to accept @IPVC_STARTDATE and @IPVC_ENDDATE
3. Add BETWEEN clause to WHERE condition on application creation/submission date

**Impact:** Application in progress counts currently not date-filtered  
**Effort:** 5-10 lines (3 lines in calling proc + 5-7 in target proc) | **Risk:** Medium

---

**BUG #6: uspNPSTodayTaskListCountSelect_UPM Receives NULL Dates (Affects: Appointment, Follow-up)**

**File:** Property-specific database  
**Stored Procedure Name:** `[Pxxx].[dbo].[uspNPSTodayTaskListCountSelect_UPM]`

**Current Procedure Call (uspTaskSummaryDashboardCount_UPM, lines 295-307):**
```sql
DECLARE @LV_StartDate VARCHAR(10);      -- ⚠️ DECLARED BUT NEVER SET!
DECLARE @LV_ENDDate VARCHAR(10);        -- ⚠️ DECLARED BUT NEVER SET!

SET @sql = N'P' + @InternalEntityID + N'.DBO.uspNPSTodayTaskListCountSelect_UPM
    @USERID = ' + @InternalUserID + N',
    @SITEID = ' + @InternalSiteID + N',
    @GCID = '''',
    @CATEGORY = ''Appointment'',
    @ASSIGNID = ''' + @ASSIGNID + N''',
    @ENDDATE = ''' + @LV_ENDDate + N''',          -- ⚠️ PASSING NULL!
    @STARTDATE = ''' + @LV_StartDate + N''',      -- ⚠️ PASSING NULL!
    @TASKCOUNT = @Appointment OUTPUT';
```

**Issue:**
- @LV_StartDate and @LV_ENDDate declared but never initialized (BUG #1)
- Procedure receives NULL date values
- Cannot perform date range filtering
- Affects 2 workflows: Appointment, Follow-up

**Root Cause:** See BUG #1 above (undefined variables)

**Fix:** Initialize the variables immediately after they are declared:
```sql
-- Add after line 281:
SET @LV_StartDate = @IPVC_STARTDATE;    -- Initialize from parameter
SET @LV_ENDDate = @IPVC_ENDDATE;        -- Initialize from parameter

-- Then the procedure will receive proper dates
EXEC sp_executesql @sql, ...
```

**Note:** uspNPSTodayTaskListCountSelect_UPM may already support date range filtering. This fix enables it to receive the actual dates instead of NULL.

**Impact:** Appointment and Follow-up counts currently receive NULL dates (completely broken date filtering)  
**Effort:** 2 lines in calling proc (already covered in BUG #1) | **Risk:** Low

---

**BUG #7: uspWaitListSelectTiles_UPM Missing Date Parameters (Affects: Conventional Waitlist)**

**File:** `DATABASESETUP\P2\procs\UPM\DashboardTiles\uspWaitListSelectTiles_UPM.sql`

**Current Procedure Call (uspGetTaskSummaryTileSummary_UPM, lines 215-218):**
```sql
set @sql = 's' + @InternalSiteID + '.DBO.[uspWaitListSelectTiles_UPM]
    @InternalEntityID  = ' + @InternalEntityID + ',
    @InternalSiteID  = ' + @InternalSiteID + ',
    @TASKCOUNT = @TASKCOUNT OUTPUT'

EXEC sp_executesql @sql, N'@TASKCOUNT INT OUTPUT', @TASKCOUNT OUTPUT;
-- MISSING: @IPVC_STARTDATE, @IPVC_ENDDATE
```

**Current Procedure Signature:**
```sql
CREATE OR ALTER PROCEDURE dbo.uspWaitListSelectTiles_UPM(
    @InternalEntityID varchar(10) = NULL, 
    @InternalSiteID varchar(10) = NULL,
    @TASKCOUNT int output
)
```

**Issue:**
- Called with dynamic SQL
- No date parameters accepted
- Uses hardcoded GETDATE() and system settings
- Cannot filter waitlist by date range
- Affects 1 workflow: Conventional Waitlist (Affordable Housing)

**Required Changes:**
1. Add parameters to procedure signature:
   ```sql
   @IPVC_STARTDATE VARCHAR(10) = NULL,
   @IPVC_ENDDATE VARCHAR(10) = NULL,
   ```
2. Modify dynamic SQL call in uspGetTaskSummaryTileSummary_UPM:
   ```sql
   set @sql = 's' + @InternalSiteID + '.DBO.[uspWaitListSelectTiles_UPM]
       @InternalEntityID  = ' + @InternalEntityID + ',
       @InternalSiteID  = ' + @InternalSiteID + ',
       @IPVC_STARTDATE = ''' + @IPVC_STARTDATE + N''',     -- ADD
       @IPVC_ENDDATE = ''' + @IPVC_ENDDATE + N''',         -- ADD
       @TASKCOUNT = @TASKCOUNT OUTPUT'
   ```
3. Inside procedure, replace hardcoded date logic with BETWEEN clause
4. Add WHERE filter on lease/waitlist entry date

**Impact:** Conventional waitlist counts currently not date-filtered  
**Effort:** 5-10 lines (2-3 lines in calling proc + 5-7 in target proc) | **Risk:** Medium

---

##### 3.2.3 Summary of All Critical Bugs

| Bug # | Procedure | Issue | Workflows Affected | Effort | Risk | Priority |
|---|---|---|---|---|---|---|
| 1 | uspTaskSummaryDashboardCount_UPM | Undefined variables | Appointment, Follow-up | 2 lines | Low | **CRITICAL** |
| 2 | uspTaskSummaryDashboardCount_UPM | MOU missing BETWEEN | Move-out | 1 line | Low | High |
| 3 | uspFASRefundCountSelect_UPM | No date parameters | FAS Ready, FAS Rejected | 5-10 lines | Medium | High |
| 4 | uspGuarantorTaskListSelectCount_UPM | No date parameters | Guarantor Form Req | 5-10 lines | Medium | High |
| 5 | uspTasksLGAppsInProgressCountSelect_UPM | No date parameters | App in Progress | 5-10 lines | Medium | High |
| 6 | uspNPSTodayTaskListCountSelect_UPM | Receives NULL dates (related to Bug #1) | Appointment, Follow-up | 2 lines | Low | **CRITICAL** |
| 7 | uspWaitListSelectTiles_UPM | No date parameters | Conventional Waitlist | 5-10 lines | Medium | High |

**Total Estimated Lines:** 30-53 lines of code changes  
**Total Estimated Effort:** Phase 4 revised to 16-18 hours (vs original 12 hours)

##### 3.2.4 Quick Fixes (Phase 1 - 15 minutes)

These 9 one-line fixes provide immediate date filtering for multiple workflows:

**1. Initialize Undefined Variables**
```sql
-- In uspTaskSummaryDashboardCount_UPM after line 281
SET @LV_StartDate = @IPVC_STARTDATE;
SET @LV_ENDDate = @IPVC_ENDDATE;
```
**Workflows Fixed:** Appointment, Follow-up

**2-8. Add BETWEEN Clauses to Inline Queries**

| # | Workflow Code | Addition | Line Approx |
|---|---|---|---|
| 2 | MIN | `AND L.leaStartDate BETWEEN @LD_STARTDATE AND @LD_ENDDATE` | 350 |
| 3 | MOU | `AND L.leaNoticeForDate BETWEEN @LD_STARTDATE AND @LD_ENDDATE` | 375 |
| 4 | RFAS | `AND L.leaEndDate BETWEEN @LD_STARTDATE AND @LD_ENDDATE` | 400 |
| 5 | ApplicantFAS | `AND L.leaEndDate BETWEEN @LD_STARTDATE AND @LD_ENDDATE` | 415 |
| 6 | FAS-PEND | `AND L.leaEndDate BETWEEN @LD_STARTDATE AND @LD_ENDDATE` | 430 |
| 7 | NRFAS | `AND FSRH.CreatedDate BETWEEN @LD_STARTDATE AND @LD_ENDDATE` | 480 |
| 8 | PVR | `AND RP.CreatedDate BETWEEN @LD_STARTDATE AND @LD_ENDDATE` | 507 |
| 9 | FVR | `AND RP.CreatedDate BETWEEN @LD_STARTDATE AND @LD_ENDDATE` | 513 |

**Impact:** 9 out of 16 workflows immediately get date filtering  
**Total Effort:** ~15 minutes

##### 3.2.4 Procedure Modifications (Phase 2 - 4-6 hours)

For workflows that depend on called procedures, modify these 5 stored procedures:

**Modification #1: uspFASRefundCountSelect_UPM**
- **Current:** No date parameters
- **Add:** `@IPVC_STARTDATE VARCHAR(10) = NULL`, `@IPVC_ENDDATE VARCHAR(10) = NULL`
- **Workflows Fixed:** FAS-READY-APP (FAS-0), FAS-REJECTED (FAS-2)
- **Date Field:** FAS approval/rejection date (TBD by query analysis)
- **Effort:** 5-10 lines

**Modification #2: uspGuarantorTaskListSelectCount_UPM**
- **Current:** No date parameters
- **Add:** `@IPVC_STARTDATE VARCHAR(10) = NULL`, `@IPVC_ENDDATE VARCHAR(10) = NULL`
- **Workflows Fixed:** GFR (Guarantor Form Required)
- **Date Field:** Form demand/submission date (TBD)
- **Effort:** 5-10 lines

**Modification #3: uspTasksLGAppsInProgressCountSelect_UPM**
- **Current:** No date parameters
- **Add:** `@IPVC_STARTDATE VARCHAR(10) = NULL`, `@IPVC_ENDDATE VARCHAR(10) = NULL`
- **Workflows Fixed:** APPINPROG (Application in Progress)
- **Date Field:** Application creation/submission date
- **Effort:** 5-10 lines

**Modification #4: uspNPSTodayTaskListCountSelect_UPM (Property Database)**
- **Current:** Receives NULL dates
- **Check:** May already support date range (verify in Property DB)
- **Workflows Fixed:** Appointment, Follow-up (after variable fix)
- **Effort:** 0-5 lines (if already supports dates) or TBD

**Modification #5: uspWaitListSelectTiles_UPM**
- **Current:** No date parameters, uses system date
- **Add:** `@IPVC_STARTDATE VARCHAR(10) = NULL`, `@IPVC_ENDDATE VARCHAR(10) = NULL`
- **Workflows Fixed:** Conventional Waitlist
- **Date Field:** Lease start date or waitlist entry date
- **Effort:** 5-10 lines

##### 3.2.5 Implementation Checklist

**Phase 1: Quick Fixes**
- [ ] Line 281: Initialize @LV_StartDate from @IPVC_STARTDATE
- [ ] Line 281: Initialize @LV_ENDDate from @IPVC_ENDDATE
- [ ] Line 350: Add BETWEEN clause to MIN query
- [ ] Line 375: Add BETWEEN clause to MOU query
- [ ] Line 400: Add BETWEEN clause to RFAS query
- [ ] Line 415: Add BETWEEN clause to ApplicantFAS query
- [ ] Line 430: Add BETWEEN clause to FAS-PEND query
- [ ] Line 480: Add BETWEEN clause to NRFAS query
- [ ] Line 507: Add BETWEEN clause to PVR subquery
- [ ] Line 513: Add BETWEEN clause to FVR subquery

**Phase 2: Procedure Modifications**
- [ ] Analyze uspFASRefundCountSelect_UPM query to identify date columns
- [ ] Add date parameters and BETWEEN clause to uspFASRefundCountSelect_UPM
- [ ] Analyze uspGuarantorTaskListSelectCount_UPM query
- [ ] Add date parameters and BETWEEN clause to uspGuarantorTaskListSelectCount_UPM
- [ ] Analyze uspTasksLGAppsInProgressCountSelect_UPM query
- [ ] Add date parameters and BETWEEN clause to uspTasksLGAppsInProgressCountSelect_UPM
- [ ] Verify uspNPSTodayTaskListCountSelect_UPM date support
- [ ] Add date parameters if needed to uspNPSTodayTaskListCountSelect_UPM
- [ ] Analyze uspWaitListSelectTiles_UPM query
- [ ] Add date parameters and BETWEEN clause to uspWaitListSelectTiles_UPM
- [ ] Update uspGetTaskSummaryTileSummary_UPM call to uspWaitListSelectTiles_UPM to pass dates

**Phase 3: Testing**
- [ ] Unit test: Each inline query with BETWEEN clause
- [ ] Integration test: All 16 workflows return filtered counts
- [ ] SQL test: Date range logic, boundary cases, legacy compat
- [ ] Performance test: < 2s for 90-day ranges
- [ ] UAT: Customer validates filtering accuracy

---

## Development Effort Estimates

### Phase 1: Frontend UI Changes
**Total Effort: 13 hours**

| Task | Component | Hours | Notes |
|------|-----------|-------|-------|
| HTML Template Updates | tile-filters.component.html | 2 | Add conditional rendering for date range picker |
| Component Logic | tile-filters.component.ts | 5 | Date range validation, epoch conversion, state management |
| Interface Updates | tile-filters.interface.ts | 1 | Extend Filters and FiltersConfig interfaces |
| Pipe Development | convertFromEpochMs.pipe.ts | 1 | Convert epoch to Date for display |
| Module Registration | tile-filters.module.ts | 0.5 | Import RpDaterangepickerModule |
| Tile-Wrap Updates | tile-wrap.interface.ts | 1.5 | Extend TileData contract |
| Code Review & Fixes | - | 2 | Address feedback, fix edge cases |

**Subtotal: 13 hours**

### Phase 2: API/Controller Layer
**Total Effort: 8.5 hours**

| Task | Component | Hours | Notes |
|------|-----------|-------|-------|
| Feature Flag Service Interface | IFeatureFlagService.cs | 2 | Define contract with multiple overloads |
| LaunchDarkly Implementation | LaunchDarklyFeatureFlagService.cs | 2.5 | Full service implementation with LD SDK |
| Context Builder | LaunchDarklyContextBuilder.cs | 1.5 | Build user/property/org contexts |
| Controller Updates | DashboardTileExecutionV1Controller.cs | 1.5 | Add validation, feature flag checks, error handling |
| DI Registration | Startup.cs, appsettings.json | 0.5 | Register service in container |
| Integration Tests | - | 0.5 | Test flag evaluation flow |

**Subtotal: 8.5 hours**

### Phase 3: Manager/DAO Layer
**Total Effort: 6 hours**

| Task | Component | Hours | Notes |
|------|-----------|-------|-------|
| DAO Parameter Extraction | TileTemplateDAO.SetFilterParameters() | 3 | Extract both dates, handle legacy single date |
| Manager Updates | TileTemplateManager.cs | 1 | Add logging, pass parameters through |
| CollectionExtensions (if exists) | CollectionExtensions.cs | 1 | Update AddRequestParameters() if needed |
| Unit Tests | DAO tests | 1 | Test parameter handling, legacy compat |

**Subtotal: 6 hours**

### Phase 4: Database & SQL
**Total Effort: 18 hours** *(Revised from 12 hours based on detailed bug analysis)*

| Task | Component | Hours | Notes |
|------|-----------|-------|-------|
| BUG #1 Fix | uspTaskSummaryDashboardCount_UPM (lines 281, 295) | 0.5 | Initialize @LV_StartDate, @LV_ENDDate (CRITICAL) |
| BUG #2 Fix | uspTaskSummaryDashboardCount_UPM (line 375) | 0.25 | Add BETWEEN to MOU query |
| Quick Fixes Phase | uspTaskSummaryDashboardCount_UPM (lines 350,400,415,430,480,507,513) | 0.75 | 7 one-line BETWEEN clauses (MIN, RFAS, ApplicantFAS, FAS-PEND, NRFAS, PVR, FVR) |
| BUG #3 Fix | uspFASRefundCountSelect_UPM.sql | 2 | Add @IPVC_STARTDATE/@IPVC_ENDDATE params + BETWEEN clause (FAS approval/rejection) |
| BUG #4 Fix | uspGuarantorTaskListSelectCount_UPM.sql | 2 | Add @IPVC_STARTDATE/@IPVC_ENDDATE params + BETWEEN clause (GFR) |
| BUG #5 Fix | uspTasksLGAppsInProgressCountSelect_UPM.sql | 2 | Add @IPVC_STARTDATE/@IPVC_ENDDATE params + BETWEEN clause (APPINPROG) |
| BUG #6 Fix | uspNPSTodayTaskListCountSelect_UPM.sql | 1 | Verify date support (may already exist); ensure called with initialized dates |
| BUG #7 Fix | uspWaitListSelectTiles_UPM.sql | 2 | Add @IPVC_STARTDATE/@IPVC_ENDDATE params + BETWEEN clause (Waitlist) |
| Main SP Updates | uspGetTaskSummaryTileSummary_UPM.sql | 3 | Update all dynamic SQL calls to pass dates (lines 215-218, 310-316, 335-338, 340-356) |
| Filter Options SP | uspGetTaskSummaryTileFilters.sql | 1.5 | Update default date range display logic |
| Legacy SPs | uspGetTaskSummaryTileSummary.sql + uspTaskSummaryCountSelect_UPM.sql | 1.5 | Update if used, else deprecate; maintain backward compat |
| Deployment Script | Deploy_WI2404320_DateRangeFilter.sql | 1.5 | Create backup, dry-run, and deployment script; rollback procedures |
| Code Review & Testing | SQL verification | 0.5 | Verify all changes, test for syntax errors |

**Subtotal: 18 hours**

**Effort Rationale:**
- Original estimate (12 hours) did not account for 5 additional procedures with bugs
- Deep analysis identified 7 critical bugs (plus sub-bugs) requiring fixes
- Quick fixes alone (1.5 hours) address 9 of 16 workflows
- Procedure modifications (9 hours) required for remaining 7 workflows
- Deployment complexity increased due to cross-database dependencies
- Additional testing and rollback planning needed for multi-database changes

### Phase 5: Testing
**Total Effort: 18 hours** *(Revised from 15 hours due to additional stored procedures)*

| Task | Category | Hours | Notes |
|------|----------|-------|-------|
| Unit Tests (C#) | UI, Controller, DAO | 5 | 20+ test cases covering validation, legacy compat, all branches |
| Integration Tests | End-to-end flow | 5 | Test UI → API → DB path for all 16 workflows (revised: more procedures = more paths) |
| SQL Tests - Inline Queries | uspTaskSummaryDashboardCount_UPM | 2 | Test 9 inline BETWEEN clauses (MIN, MOU, RFAS, ApplicantFAS, FAS-PEND, NRFAS, PVR, FVR) |
| SQL Tests - Procedures | 5 updated procedures | 3 | Test FASRefund, Guarantor, AppInProgress, Waitlist, NPSToday procedures with date ranges |
| SQL Tests - Performance | Date range queries | 1 | Verify <2s for 90-day ranges on 100K+ records |
| UAT Scenarios | Manual testing | 1.5 | Test with actual data, multiple properties, all 16 workflows + edge cases |
| Performance & Load Tests | Full workflow | 0.5 | Verify <500ms API + <2s DB total with concurrent requests |

**Subtotal: 18 hours**

**Effort Rationale:**
- Original estimate (15 hours) focused on primary path
- 5 additional procedures require dedicated test plans
- 9 inline query modifications need individual testing
- Cross-database calls (dynamic SQL) require integration testing
- All 16 workflows must be covered with date filtering validation
- Performance regression testing critical due to BETWEEN clauses

### Phase 6: Release & Documentation
**Total Effort: 6 hours**

| Task | Component | Hours | Notes |
|------|-----------|-------|-------|
| Release Notes | Documentation | 1 | Document changes, known issues |
| Developer Guide | Documentation | 1.5 | How to use new features, parameters |
| Migration Guide | Documentation | 1 | For customers upgrading from legacy |
| Rollback Procedure | Runbook | 1 | Steps to disable via feature flag |
| Training Materials | Videos/Docs | 1 | For internal team and customer |

**Subtotal: 6 hours**

---

## Test Scenarios & Criteria

### Unit Test Scenarios

#### UT-1: Date Range Validation - Valid Range
**Test Case:** `TileFiltersComponentTests.onDateRangeChange_ValidRange_EmitsCorrectPayload`

**Pre-conditions:**
- Component initialized
- User selects start date: 2024-01-01
- User selects end date: 2024-01-31

**Test Steps:**
```gherkin
Given a TileFiltersComponent with showDateRangeFilter=true
When user selects start date "2024-01-01" and end date "2024-01-31"
Then component should store:
  - dateFilterFrom = "1704067200000" (epoch ms)
  - dateFilterTo = "1704672000000" (epoch ms)
And emit filterChange event with both dates
And NOT emit error notification
```

**Expected Result:** ✅ Event emitted with correct epoch values

---

#### UT-2: Date Range Validation - Invalid Range (End < Start)
**Test Case:** `TileFiltersComponentTests.onDateRangeChange_InvalidRange_NoEmit`

**Pre-conditions:**
- Component initialized
- User selects start date: 2024-01-31
- User selects end date: 2024-01-01 (INVALID: reversed)

**Test Steps:**
```gherkin
Given a TileFiltersComponent
When user selects invalid range (endDate < startDate)
Then component should:
  - NOT emit filterChange event
  - Display error: "Start date must be before end date"
  - Log warning with details
```

**Expected Result:** ✅ No emit, error displayed

---

#### UT-3: Date Range Validation - Exceeds Max Days
**Test Case:** `DashboardTileExecutionV1ControllerTests.ValidateDateRangeParameters_RangeExceedsMax_ThrowsException`

**Pre-conditions:**
- Controller receives request with:
  - dateFilterFrom: "1000000000000" (very old date)
  - dateFilterTo: "2000000000000" (future date, > 365 days apart)

**Test Steps:**
```gherkin
Given API request with date range > 365 days
When ValidateDateRangeParameters is called
Then method should throw ArgumentException
And exception message should contain "365 days"
And HTTP 400 status should be returned
```

**Expected Result:** ✅ ArgumentException thrown, 400 status

---

#### UT-4: DAO Parameter Extraction - Date Range
**Test Case:** `TileTemplateDAOTests.SetFilterParameters_DateRangeProvided_AddsCorrectSqlParameters`

**Pre-conditions:**
- DAO initialized
- FilterBy dictionary contains:
  - `dateFilterFrom: "1704067200000"`
  - `dateFilterTo: "1704672000000"`
  - `filter1: "testvalue"`

**Test Steps:**
```gherkin
Given a TileTemplateDAO
When SetFilterParameters called with date range params
Then SQL command parameters should include:
  - @dateFilterFrom = "1704067200000"
  - @dateFilterTo = "1704672000000"
  - @filter1 = "testvalue"
And legacy @dateFilter NOT added
```

**Expected Result:** ✅ Correct parameters added to command

---

#### UT-5: DAO Parameter Extraction - Legacy Single Date
**Test Case:** `TileTemplateDAOTests.SetFilterParameters_LegacyDateOnly_AddsLegacyParameter`

**Pre-conditions:**
- DAO initialized
- FilterBy dictionary contains: `dateFilter: "1704067200000"` only

**Test Steps:**
```gherkin
Given a TileTemplateDAO with legacy date filter
When SetFilterParameters called
Then SQL command should have:
  - @dateFilter = "1704067200000"
  - @dateFilterFrom = NULL
  - @dateFilterTo = NULL
```

**Expected Result:** ✅ Legacy parameter only

---

### Integration Test Scenarios

#### IT-1: Full Flow - Date Range Request
**Test Case:** `TaskSummaryIntegrationTests.ExecuteTileTaskSummary_WithDateRange_ReturnsCorrectData`

**Pre-conditions:**
- System initialized with feature flag ON
- User authenticated
- Valid property access

**Test Steps:**
```gherkin
Given an authenticated user making a TaskSummary request
When request includes:
  - dateFilterFrom: "1704067200000" (Jan 1, 2024)
  - dateFilterTo: "1704672000000" (Jan 31, 2024)
Then API should:
  1. Extract date range parameters
  2. Evaluate feature flag → ON
  3. Validate range (30 days, valid)
  4. Pass to Manager
  5. Manager passes to DAO
  6. DAO builds SQL with both params
  7. Execute: spGetTaskSummaryTileSummary_UPM
  8. SP receives @dateFilterFrom, @dateFilterTo
  9. SP converts to datetime
  10. SP executes: WHERE task_date BETWEEN @start AND @end
  11. Returns TileData with counts for all 16 workflows
  12. Response includes workflow counts for date range
```

**Expected Result:** ✅ HTTP 200, TileData with date range results

---

#### IT-2: Feature Flag OFF - Date Range Removed
**Test Case:** `TaskSummaryIntegrationTests.ExecuteTileTaskSummary_FeatureFlagDisabled_UsesSingleDate`

**Pre-conditions:**
- Feature flag: `dashboards-daterangefilter-enabled` = False
- Request includes dateFilterFrom and dateFilterTo

**Test Steps:**
```gherkin
Given feature flag is OFF
When request includes dateFilterFrom and dateFilterTo
Then controller should:
  1. Evaluate flag → OFF
  2. Remove dateFilterFrom from parameters
  3. Remove dateFilterTo from parameters
  4. PassLegacy behavior to downstream
  5. DAO should only receive legacy @dateFilter (if any)
  6. Results use legacy single-date logic
```

**Expected Result:** ✅ Date range params removed, legacy behavior used

---

#### IT-3: Multi-Property Behavior
**Test Case:** `TaskSummaryIntegrationTests.ExecuteTileTaskSummary_MultipleProperties_ScopedToProperty`

**Pre-conditions:**
- User has access to 2 properties: Prop A, Prop B
- Request for Prop A with date range

**Test Steps:**
```gherkin
Given multi-property access
When executing tile for Property A with date range
Then:
  1. DAO filters by PropertyID
  2. Only Prop A data returned (not Prop B)
  3. Date range applies to Prop A only
  4. Security isolation maintained
```

**Expected Result:** ✅ Only Prop A data, date range scoped

---

### SQL Test Scenarios

#### ST-1: Date Range Query Execution
**Test Case:** `Deploy_WI2404320_Tests.sql` - Test 1

**Pre-conditions:**
- Stored procedure deployed
- Test data with tasks across 30 days

**SQL:**
```sql
DECLARE @PropertyID INT = 1
DECLARE @DateFilterFrom VARCHAR(MAX) = '1704067200000'  -- Jan 1, 2024
DECLARE @DateFilterTo VARCHAR(MAX) = '1704672000000'    -- Jan 31, 2024

EXEC dbo.uspTaskSummaryDashboardCount_UPM 
    @PropertyID = @PropertyID,
    @dateFilterFrom = @DateFilterFrom,
    @dateFilterTo = @DateFilterTo

-- Expected: Results for full 30-day range
-- Example output:
-- MIN | 15
-- MOU | 8
-- RFAS | 5
-- ... (all 16 workflows)
```

**Assertion:**
```sql
-- Verify date range results > single date results
SELECT @SingleDateCount, @RangeDateCount
-- Expected: @RangeDateCount >= @SingleDateCount
```

**Expected Result:** ✅ Date range query returns data

---

#### ST-2: Hardcoded Bug Fix Verification
**Test Case:** `Deploy_WI2404320_Tests.sql` - Verify Fix

**Pre-conditions:**
- Old SP has: `SET @LV_ENDDate = @LV_StartDate`
- New SP has proper range logic

**SQL:**
```sql
-- OLD (BUGGY) behavior simulation
DECLARE @LV_StartDate DATETIME = '2024-01-01'
DECLARE @LV_ENDDate_BUGGY DATETIME = @LV_StartDate  -- Same day only

-- NEW (FIXED) behavior
DECLARE @LV_ENDDate_FIXED DATETIME = DATEADD(DAY, 1, @LV_StartDate)
                                      - 1 SECOND          -- Full day

-- Verify NEW logic in actual SP
EXEC dbo.uspGetTaskSummaryTileSummary_UPM
    @PropertyID = 1,
    @dateFilterFrom = '1704067200000',
    @dateFilterTo = '1704326400000'

-- Expected: Full date range included, not just first second
```

**Expected Result:** ✅ SP uses full date range

---

#### ST-3: Legacy Parameter Compatibility
**Test Case:** `Deploy_WI2404320_Tests.sql` - Backward Compat

**Pre-conditions:**
- SP supports both new and legacy parameters

**SQL:**
```sql
-- Test 1: New date range parameters
EXEC dbo.uspTaskSummaryDashboardCount_UPM 
    @PropertyID = 1,
    @dateFilterFrom = '1704067200000',
    @dateFilterTo = '1704672000000'
-- Expected: Success

-- Test 2: Legacy single date (should still work)
EXEC dbo.uspTaskSummaryDashboardCount_UPM 
    @PropertyID = 1,
    @dateFilter = '1704067200000'
-- Expected: Success (backward compat)

-- Test 3: No date parameters
EXEC dbo.uspTaskSummaryDashboardCount_UPM 
    @PropertyID = 1
-- Expected: Success (uses current date)

-- Test 4: Invalid range (end < start)
EXEC dbo.uspTaskSummaryDashboardCount_UPM 
    @PropertyID = 1,
    @dateFilterFrom = '1704672000000',  -- Jan 31
    @dateFilterTo = '1704067200000'     -- Jan 1 (INVALID)
-- Expected: Error raised
```

**Expected Result:** ✅ All cases handled correctly

---

#### ST-4: Performance - Large Date Range
**Test Case:** `PerformanceTests.sql`

**Pre-conditions:**
- Database with 100K+ task records
- Date range spanning 90 days

**SQL:**
```sql
-- Measure execution time
DECLARE @StartTime DATETIME2 = GETUTCDATE()

EXEC dbo.uspTaskSummaryDashboardCount_UPM 
    @PropertyID = 1,
    @dateFilterFrom = '1700000000000',  -- ~90 days
    @dateFilterTo = '1702000000000'

DECLARE @EndTime DATETIME2 = GETUTCDATE()
DECLARE @ExecutionMs INT = DATEDIFF(MILLISECOND, @StartTime, @EndTime)

-- Assertion
SELECT CASE 
    WHEN @ExecutionMs < 2000 THEN 'PASS: < 2 seconds'
    ELSE 'FAIL: > 2 seconds (' + CAST(@ExecutionMs AS VARCHAR) + 'ms)'
END AS PerformanceResult
```

**Expected Result:** ✅ Execution < 2000ms

---

### UAT Scenarios

#### UAT-1: End User Workflow - Lindemann Customer
**Scenario:** Lindemann property manager filters tasks by date range

**Steps:**
1. Login to OneSite Dashboard
2. Navigate to Task Summary widget
3. Observe date range picker displayed (instead of single date)
4. Select date range: March 1 - March 15, 2026
5. View task counts for all workflows within range
6. Compare with Classic OneSite for parity
7. Export results

**Expected Result:** ✅ Date range filtering works exactly like Classic OneSite

---

#### UAT-2: Performance Validation
**Scenario:** Dashboard loads with large date range quick and responsibly

**Steps:**
1. Select 90-day date range
2. Monitor page load time
3. Check Network tab in DevTools
4. Measure API response time
5. Verify < 500ms API time + < 2s database time

**Expected Result:** ✅ Page responsive, < 2s total

---

#### UAT-3: Legacy User Workflow - Single Date Still Works
**Scenario:** Existing customer continues using single date picker

**Steps:**
1. Property with feature flag OFF
2. Only single date picker displays
3. Select a date (no range)
4. Results match previous behavior
5. No change noticed by end user

**Expected Result:** ✅ Single date mode transparent to user

---

## Acceptance Criteria

### AC1: UI Feature
- [ ] Date range picker displays when server flag indicates support
- [ ] Default range set to last 30 days
- [ ] User can select any date range within 365 days
- [ ] Client-side validation prevents invalid ranges
- [ ] Error notifications display for invalid selections
- [ ] Single date picker still works (backward compat)
- [ ] Responsive design (mobile-friendly)

### AC2: API Implementation
- [ ] Feature flag evaluated before processing
- [ ] Date parameters validated (format, range, max 365 days)
- [ ] Date range parameters extracted from request
- [ ] Legacy single-date supported
- [ ] HTTP 400 returned for validation failures with clear error messages
- [ ] Feature flag can be toggled without deployment
- [ ] Custom events tracked in LaunchDarkly

### AC3: Database Execution
- [ ] Main SP bug fixed (removed hardcoded same-day logic) - uspGetTaskSummaryTileSummary_UPM
- [ ] Critical bug #1 fixed (undefined variables initialized) - uspTaskSummaryDashboardCount_UPM
- [ ] Critical bug #2 fixed (MOU BETWEEN clause added) - uspTaskSummaryDashboardCount_UPM
- [ ] All 9 inline query BETWEEN clauses added (MIN, MOU, RFAS, ApplicantFAS, FAS-PEND, NRFAS, PVR, FVR) - uspTaskSummaryDashboardCount_UPM
- [ ] 5 called procedures updated with date parameters: uspFASRefundCountSelect_UPM, uspGuarantorTaskListSelectCount_UPM, uspTasksLGAppsInProgressCountSelect_UPM, uspNPSTodayTaskListCountSelect_UPM, uspWaitListSelectTiles_UPM
- [ ] All dynamic SQL calls updated to pass date parameters
- [ ] BETWEEN clauses properly include full end day
- [ ] All 16 workflows return correct date-filtered counts
- [ ] Performance: queries complete in < 2 seconds
- [ ] Backward compatible with single-date queries
- [ ] Error handling for invalid date ranges
- [ ] Legacy procedures maintained or deprecated with notice

### AC4: Testing Coverage
- [ ] Unit tests: UI, Controller, DAO (20+ test cases)
- [ ] Integration tests: end-to-end flow (5+ scenarios)
- [ ] SQL tests: SP execution and performance (4+ scenarios)
- [ ] UAT: customer validation scenarios (3+ scenarios)
- [ ] All tests passing
- [ ] 90%+ code coverage

### AC5: Documentation
- [ ] Technical specification complete (this document)
- [ ] Implementation completed per roadmap
- [ ] Developer guide for future maintenance
- [ ] Release notes published
- [ ] Customer migration guide for upgrade

### AC6: Client Success
- [ ] Lindemann customer able to filter by date range
- [ ] Lindemann customer migrates away from Classic OneSite
- [ ] Feature flag controlled rollout without issues
- [ ] Performance metrics meet targets
- [ ] Customer satisfaction confirmed

---

## Summary

**Total Development Effort: 60.5 hours (~3 weeks at 20 hrs/week)**

Breakdown:
- Frontend: 13 hours (22%)
- API: 8.5 hours (14%)
- Manager/DAO: 6 hours (10%)
- Database: 12 hours (20%)
- Testing: 15 hours (24%)
- Release: 6 hours (10%)

**Key Milestones:**
- Week 1: Phase 1 (UI) + Phase 2 (API) + start Phase 5 (testing)
- Week 2: Phase 3 (Manager/DAO) + Phase 4 (Database) + complete Phase 5
- Week 3: Phase 6 (Release) + UAT + deployment planning

**Risk Mitigation:**
- Feature flag OFF by default → safe deployment
- Gradual 10% → 25% → 50% → 100% rollout
- Backward compatibility maintained → no breaking changes
- Full rollback capability via flag toggle

---

**Document Version:** 1.0  
**Created:** March 9, 2026  
**Status:** Ready for Development  
**Next Step:** Assign developers to phases and begin implementation

For questions, contact:
- **Feature Owner:** Vijay Shankar
- **PME Owner:** Joan Salonga
- **Development Lead:** [TBD]
