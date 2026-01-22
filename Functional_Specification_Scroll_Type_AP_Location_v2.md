# Functional Specification Document
## Scroll Type and AP Location Code Determination Enhancement (Dynamic Configuration)

**Document Version:** 2.0  
**Date:** 2025-01-27  
**Author:** Development Team  
**Status:** Draft

---

## 1. Executive Summary

This document outlines the functional requirements for enhancing the Scroll Type determination logic and AP Location Code determination in the SAP ABAP Function Module `Z_SCME_CREATE_SCROLL` and the method `GET_AP_LOC` of class `ZCL_SCME_SCROLL_CREATE_VALID` using **dynamic configuration** from maintenance table `ZLOG_EXEC_VAR`.

### 1.1 Purpose
The purpose of this enhancement is to:
1. Implement enhanced scroll type determination based on transport mode (Marine/Air) and movement direction (Inbound/Outbound) using **dynamic configuration**
2. Enhance AP location code determination logic to support Import/Export scenarios and Marine-specific requirements using **dynamic SType values**
3. Eliminate hardcoded values and enable business users to maintain scroll types and SType values through configuration table

### 1.2 Scope
- **Function Module:** `Z_SCME_CREATE_SCROLL`
- **Class Method:** `ZCL_SCME_SCROLL_CREATE_VALID=>GET_AP_LOC`
- **Maintenance Table:** `ZLOG_EXEC_VAR`
- **Impact Areas:** Scroll creation process, AP location determination, configuration management

---

## 2. Business Requirements

### 2.1 Requirement 1: Dynamic Scroll Type Determination

#### 2.1.1 Current Behavior
Currently, scroll type is determined solely based on movement direction:
- **Inbound (I):** Scroll Type = `INRO`
- **Outbound (O):** Scroll Type = `OBRO`

#### 2.1.2 Required Behavior
The system shall determine scroll type **dynamically** from configuration table `ZLOG_EXEC_VAR` based on:
- **Transport Mode:** Marine (MR), Air (AR), Road (RD), Rail (RL)
- **Movement Direction:** Inbound (I) or Outbound (O)

**Dynamic Configuration Logic:**
1. Query `ZLOG_EXEC_VAR` table with:
   - `NAME = 'ZSCM_SCROLLTYPE_FETCH'`
   - `VSART = gw_trbl_det-transport_mode` (Transport Mode: MR, AR, RD, RL)
   - `FUNCTION = <gfs_scrl>-movt_dir` (Movement Direction: I or O)
   - `ACTIVE = 'X'` (Active indicator)
2. Retrieve `REMARKS` field which contains the **Scroll Type** value
3. If no record found, fallback to existing logic (INRO/OBRO based on movement direction)

**Configuration Table Structure:**
| Field | Value | Description |
|-------|-------|-------------|
| NAME | ZSCM_SCROLLTYPE_FETCH | Configuration name for scroll type |
| VSART | MR/AR/RD/RL | Transport mode (Shipping type) |
| FUNCTION | I/O | Movement direction (Inbound/Outbound) |
| REMARKS | INSF/OBSF/IASF/OASF/INRO/OBRO | Scroll type value |
| ACTIVE | X | Active indicator |

#### 2.1.3 Business Rules
1. System shall query `ZLOG_EXEC_VAR` table for scroll type configuration
2. Query criteria: `NAME = 'ZSCM_SCROLLTYPE_FETCH'` AND `VSART = transport_mode` AND `FUNCTION = movement_direction` AND `ACTIVE = 'X'`
3. Movement direction (`<gfs_scrl>-movt_dir`) is passed in `FUNCTION` field of `ZLOG_EXEC_VAR` table
4. If configuration found, use `REMARKS` field value as scroll type
5. If configuration not found, use existing fallback logic:
   - Inbound (I) → `INRO`
   - Outbound (O) → `OBRO`
6. Business users can maintain scroll type values through `ZLOG_EXEC_VAR` table without code changes

#### 2.1.4 Data Sources
- **Configuration Table:** `ZLOG_EXEC_VAR`
- **Transport Mode:** Available in `gw_trbl_det-transport_mode` (set from `i_marin`, `i_air` flags)
- **Movement Direction:** Available in `<gfs_scrl>-movt_dir` (values: 'I' or 'O')
- **Scroll Type Value:** Retrieved from `ZLOG_EXEC_VAR-REMARKS` field

#### 2.1.5 Configuration Examples
| Transport Mode | Movement Direction | Configuration (NAME/VSART/FUNCTION) | REMARKS Value | Result |
|----------------|-------------------|-----------------------------------|---------------|---------|
| MR | I | ZSCM_SCROLLTYPE_FETCH / MR / I | INSF | INSF |
| MR | O | ZSCM_SCROLLTYPE_FETCH / MR / O | OBSF | OBSF |
| AR | I | ZSCM_SCROLLTYPE_FETCH / AR / I | IASF | IASF |
| AR | O | ZSCM_SCROLLTYPE_FETCH / AR / O | OASF | OASF |
| RD | I | ZSCM_SCROLLTYPE_FETCH / RD / I | INRO | INRO |
| RD | O | ZSCM_SCROLLTYPE_FETCH / RD / O | OBRO | OBRO |

---

### 2.2 Requirement 2: Dynamic AP Location Code Determination

#### 2.2.1 Current Behavior
Currently, AP location is fetched from table `YSCM_APLOC` with:
- **Type:** `SER` (constant `gc_ser`) - **Hardcoded**
- **SType:** `LOGI` (constant `gc_logi`)

#### 2.2.2 Required Behavior
The system shall determine AP location **dynamically** using Type and SType values from configuration table `ZLOG_EXEC_VAR` for EXIM (Import/Export) and Road scenarios.

**Step 0: Determine Type for Data Fetching**
- Query `ZLOG_EXEC_VAR` with:
  - `NAME = 'ZSCM_APCODETYPE_FETCH'`
  - `ACTIVE = 'X'`
  - Retrieve `REMARKS` field for Type value
- If configuration not found, use existing logic: **Type = 'SER'** (constant `gc_ser`)

**Step 1: Determine SType for Data Fetching (EXIM Scenario)**
- If scenario is **Import** (`i_import = 'X'`) OR **Export** (`i_export = 'X'` OR `i_rexport = 'X'`):
  - **Primary:** Query `ZLOG_EXEC_VAR` with:
    - `NAME = 'ZSCM_SERTYPE_EXIM_FETCH'`
    - `ACTIVE = 'X'`
    - Retrieve `REMARKS` field for SType value
  - **Fallback:** If not found, query `ZLOG_EXEC_VAR` with:
    - `NAME = 'ZSCM_SERTYPE_QWIK_FETCH'`
    - `ACTIVE = 'X'`
    - Retrieve `REMARKS` field for SType value
  - **Final Fallback:** If still not found, use existing logic: `SType = 'LOGI'`
- Otherwise (non-EXIM scenario):
  - Use existing logic: **SType = 'LOGI'**

**Step 2: Post-Fetch Determination (Marine Scenario)**
After fetching AP location data:
- If transport mode is **Marine (MR)** AND (movement direction is **Inbound (I)** OR **Outbound (O)**):
  - Use SType determined from EXIM configuration (from Step 1) for AP location determination
- Otherwise:
  - Use the SType used for fetching

#### 2.2.3 Business Rules

**Rule 0: Type Configuration**
1. Query `ZLOG_EXEC_VAR` where:
   - `NAME = 'ZSCM_APCODETYPE_FETCH'`
   - `ACTIVE = 'X'`
   - Use `REMARKS` field value as Type
2. If configuration not found, use existing constant: `Type = 'SER'` (`gc_ser`)

**Rule 1: EXIM SType Configuration (Priority Order)**
1. First attempt: Query `ZLOG_EXEC_VAR` where:
   - `NAME = 'ZSCM_SERTYPE_EXIM_FETCH'`
   - `ACTIVE = 'X'`
   - Use `REMARKS` field value as SType
2. Second attempt (if first fails): Query `ZLOG_EXEC_VAR` where:
   - `NAME = 'ZSCM_SERTYPE_QWIK_FETCH'`
   - `ACTIVE = 'X'`
   - Use `REMARKS` field value as SType
3. Final fallback: Use `SType = 'LOGI'` (existing constant)

**Rule 2: Marine Scenario Check**
- After data fetch, if transport mode = 'MR' (Marine) AND movement direction is 'I' or 'O':
  - Use the SType value retrieved from EXIM configuration (Step 1)
- Else:
  - Use the SType used for fetching

**Rule 3: Non-EXIM Scenarios**
- For non-Import/Export scenarios, use existing logic: `SType = 'LOGI'`

#### 2.2.4 Data Sources
- **Configuration Table:** `ZLOG_EXEC_VAR`
- **Import Flag:** `i_import` (CHAR1)
- **Export Flag:** `i_export` (CHAR1)
- **Re-Export Flag:** `i_rexport` (CHAR1)
- **Transport Mode:** `gw_trbl_det-transport_mode` (values: 'MR', 'AR', 'RD', 'RL')
- **Movement Direction:** `<gfs_scrl>-movt_dir` (values: 'I' or 'O')
- **AP Location Table:** `YSCM_APLOC`
- **Type Value:** Retrieved from `ZLOG_EXEC_VAR-REMARKS` field (NAME = 'ZSCM_APCODETYPE_FETCH')
- **SType Value:** Retrieved from `ZLOG_EXEC_VAR-REMARKS` field

#### 2.2.5 Configuration Examples

**Type Configuration:**
| Field | Value | Description |
|-------|-------|-------------|
| NAME | ZSCM_APCODETYPE_FETCH | Configuration for AP Location Type |
| REMARKS | SER | Type value (default: SER if not maintained) |
| ACTIVE | X | Active indicator |

**EXIM Configuration:**
| Field | Value | Description |
|-------|-------|-------------|
| NAME | ZSCM_SERTYPE_EXIM_FETCH | Primary configuration for EXIM SType |
| REMARKS | IMPORTS | SType value for Import/Export scenarios |
| ACTIVE | X | Active indicator |

**Fallback Configuration:**
| Field | Value | Description |
|-------|-------|-------------|
| NAME | ZSCM_SERTYPE_QWIK_FETCH | Fallback configuration for SType |
| REMARKS | IMPORTS | SType value (fallback) |
| ACTIVE | X | Active indicator |

---

## 3. Functional Flow

### 3.1 Dynamic Scroll Type Determination Flow

```
START
  ↓
Get Transport Mode (gw_trbl_det-transport_mode)
Get Movement Direction (<gfs_scrl>-movt_dir)
  ↓
Query ZLOG_EXEC_VAR
  WHERE NAME = 'ZSCM_SCROLLTYPE_FETCH'
    AND VSART = transport_mode
    AND FUNCTION = movement_direction
    AND ACTIVE = 'X'
  ↓
Is Record Found?
  ├─ YES → Use REMARKS field value as Scroll Type
  │         Set: gw_yinvhead_i-scrtype = REMARKS
  │              <gfs_scrl>-scroll_type = REMARKS
  │
  └─ NO → Use Fallback Logic
          ├─ Movement Direction = 'I' → Set Scroll Type = 'INRO'
          └─ Movement Direction = 'O' → Set Scroll Type = 'OBRO'
END
```

### 3.2 Dynamic AP Location Determination Flow

```
START
  ↓
Query ZLOG_EXEC_VAR for Type
  WHERE NAME = 'ZSCM_APCODETYPE_FETCH' AND ACTIVE = 'X'
  ↓
Is Type Configuration Found?
  ├─ YES → Use REMARKS as Type
  │
  └─ NO → Use Type = 'SER' (fallback to gc_ser)
  ↓
Is EXIM Scenario? (i_import='X' OR i_export='X' OR i_rexport='X')
  ├─ YES → Query ZLOG_EXEC_VAR for SType
  │         ├─ Try: NAME = 'ZSCM_SERTYPE_EXIM_FETCH' AND ACTIVE = 'X'
  │         │         └─ Found? → Use REMARKS as SType
  │         │
  │         └─ Not Found? → Try: NAME = 'ZSCM_SERTYPE_QWIK_FETCH' AND ACTIVE = 'X'
  │                           └─ Found? → Use REMARKS as SType
  │                           └─ Not Found? → Use SType = 'LOGI' (fallback)
  │
  └─ NO → Use SType = 'LOGI' (existing)
  ↓
Fetch AP Location from YSCM_APLOC
  WHERE TYPE = determined_type AND STYPE = determined_stype
  ↓
After Data Fetch
  ↓
Is Transport Mode = 'MR' (Marine) AND (Movement Direction = 'I' OR 'O')?
  ├─ YES → Use SType from EXIM configuration for AP Location determination
  │
  └─ NO → Use fetched SType for AP Location determination
END
```

---

## 4. User Stories

### 4.1 Dynamic Scroll Type Configuration
**As a** business administrator  
**I want** to configure scroll types in `ZLOG_EXEC_VAR` table for different transport modes  
**So that** scroll types can be changed without code modifications

### 4.2 Dynamic SType Configuration for EXIM
**As a** business administrator  
**I want** to configure SType values for EXIM scenarios in `ZLOG_EXEC_VAR` table  
**So that** AP location determination can be adjusted for Import/Export scenarios without code changes

### 4.3 Scroll Type - Marine Inbound (Dynamic)
**As a** business user  
**I want** the system to assign scroll type from configuration table for Marine transport with Inbound movement  
**So that** the scroll is correctly categorized using maintainable configuration

### 4.4 AP Location - EXIM Scenario (Dynamic)
**As a** business user  
**I want** the system to fetch AP location using SType from configuration table for Import/Export scenarios  
**So that** the correct AP location is determined using configurable values

---

## 5. Acceptance Criteria

### 5.1 Requirement 1 Acceptance Criteria
1. ✅ System queries `ZLOG_EXEC_VAR` table with `NAME = 'ZSCM_SCROLLTYPE_FETCH'`, `VSART = transport_mode`, and `FUNCTION = movement_direction`
2. ✅ Movement direction (`<gfs_scrl>-movt_dir`) is passed in `FUNCTION` field of the query
3. ✅ Scroll type is retrieved from `REMARKS` field when configuration exists
4. ✅ Fallback to existing logic (INRO/OBRO) when configuration not found
5. ✅ Scroll type is correctly populated in both `gw_yinvhead_i-scrtype` and `<gfs_scrl>-scroll_type`
6. ✅ Configuration can be maintained through `ZLOG_EXEC_VAR` table without code changes
7. ✅ Only active configurations (`ACTIVE = 'X'`) are considered

### 5.2 Requirement 2 Acceptance Criteria
1. ✅ System queries `ZLOG_EXEC_VAR` table with `NAME = 'ZSCM_APCODETYPE_FETCH'` for Type value
2. ✅ If Type configuration not found, system uses `Type = 'SER'` (gc_ser) as fallback
3. ✅ For EXIM scenarios, system queries `ZLOG_EXEC_VAR` with `NAME = 'ZSCM_SERTYPE_EXIM_FETCH'` first
4. ✅ If primary SType configuration not found, system queries with `NAME = 'ZSCM_SERTYPE_QWIK_FETCH'` as fallback
5. ✅ If both SType configurations not found, system uses `SType = 'LOGI'` as final fallback
6. ✅ Type and SType values from configuration are used for AP location fetching
7. ✅ For Marine scenarios, configured SType is used for AP location determination
8. ✅ Configuration can be maintained through `ZLOG_EXEC_VAR` table without code changes
9. ✅ Only active configurations (`ACTIVE = 'X'`) are considered

---

## 6. Configuration Maintenance

### 6.1 Scroll Type Configuration
**Table:** `ZLOG_EXEC_VAR`  
**Key Fields:**
- `NAME = 'ZSCM_SCROLLTYPE_FETCH'`
- `VSART = <Transport Mode>` (MR, AR, RD, RL)
- `FUNCTION = <Movement Direction>` (I or O)
- `REMARKS = <Scroll Type Value>` (INSF, OBSF, IASF, OASF, INRO, OBRO, etc.)
- `ACTIVE = 'X'`

**Example Records:**
```
NAME                    | VSART | FUNCTION | REMARKS | ACTIVE
------------------------|-------|----------|---------|--------
ZSCM_SCROLLTYPE_FETCH  | MR    | I        | INSF    | X
ZSCM_SCROLLTYPE_FETCH  | MR    | O        | OBSF    | X
ZSCM_SCROLLTYPE_FETCH  | AR    | I        | IASF    | X
ZSCM_SCROLLTYPE_FETCH  | AR    | O        | OASF    | X
ZSCM_SCROLLTYPE_FETCH  | RD    | I        | INRO    | X
ZSCM_SCROLLTYPE_FETCH  | RD    | O        | OBRO    | X
```

### 6.2 Type Configuration
**Table:** `ZLOG_EXEC_VAR`  
**Key Fields:**
- `NAME = 'ZSCM_APCODETYPE_FETCH'`
- `REMARKS = <Type Value>` (e.g., SER)
- `ACTIVE = 'X'`

**Example Records:**
```
NAME                    | REMARKS | ACTIVE
------------------------|---------|--------
ZSCM_APCODETYPE_FETCH  | SER     | X
```

**Notes:**
- Configuration name: `ZSCM_APCODETYPE_FETCH`
- `REMARKS` contains the Type value to be used (typically 'SER')
- `ACTIVE` must be 'X' for configuration to be active
- If not maintained, system uses `gc_ser` ('SER') as fallback

### 6.3 SType Configuration for EXIM
**Table:** `ZLOG_EXEC_VAR`  
**Key Fields:**
- `NAME = 'ZSCM_SERTYPE_EXIM_FETCH'` (Primary)
- `REMARKS = <SType Value>` (e.g., IMPORTS)
- `ACTIVE = 'X'`

**Fallback Configuration:**
- `NAME = 'ZSCM_SERTYPE_QWIK_FETCH'` (Fallback)
- `REMARKS = <SType Value>` (e.g., IMPORTS)
- `ACTIVE = 'X'`

**Example Records:**
```
NAME                      | REMARKS  | ACTIVE
--------------------------|----------|--------
ZSCM_SERTYPE_EXIM_FETCH  | IMPORTS  | X
ZSCM_SERTYPE_QWIK_FETCH  | IMPORTS  | X
```

---

## 7. Out of Scope

1. Changes to `ZLOG_EXEC_VAR` table structure
2. UI/UX changes for configuration maintenance
3. Changes to other transport modes beyond configuration support
4. Changes to other methods in the class
5. Historical data migration for existing scrolls

---

## 8. Dependencies

1. **Configuration Table:** 
   - `ZLOG_EXEC_VAR` - Must exist and be accessible
   - Proper authorization for reading configuration

2. **Data Structures:**
   - `ZSCME_SS_SCROLL` - Scroll details structure
   - `YSCM_APLOC` - AP location table

3. **Function Module Parameters:**
   - `i_marin`, `i_air` - Transport mode flags
   - `i_import`, `i_export`, `i_rexport` - Import/Export flags

4. **Configuration Data:**
   - Scroll type configurations must be maintained in `ZLOG_EXEC_VAR`
   - Type configurations must be maintained in `ZLOG_EXEC_VAR` (optional, defaults to SER)
   - SType configurations must be maintained in `ZLOG_EXEC_VAR`

---

## 9. Assumptions

1. `ZLOG_EXEC_VAR` table is accessible and contains required configuration records
2. `VSART` field in `ZLOG_EXEC_VAR` corresponds to transport mode values (MR, AR, RD, RL)
3. `REMARKS` field contains valid scroll type and SType values
4. Configuration records are maintained with `ACTIVE = 'X'` flag
5. Fallback logic (INRO/OBRO, LOGI) remains as default when configuration not found
6. Transport mode values are: 'MR' (Marine), 'AR' (Air), 'RD' (Road), 'RL' (Rail)
7. Business users have access to maintain `ZLOG_EXEC_VAR` table

---

## 10. Risks and Mitigation

| Risk | Impact | Mitigation |
|------|--------|------------|
| Missing configuration in ZLOG_EXEC_VAR | High | Implement fallback logic to existing values |
| Invalid REMARKS values in configuration | Medium | Validate configuration values, provide error messages |
| Performance impact of additional table reads | Medium | Optimize queries, consider caching if needed |
| Configuration data inconsistency | Medium | Provide validation checks, documentation for maintenance |
| Impact on existing functionality | High | Thorough testing of existing scenarios with fallback logic |

---

## 11. Testing Requirements

### 11.1 Unit Testing
- Test scroll type determination with valid configuration in `ZLOG_EXEC_VAR`
- Test scroll type determination with missing configuration (fallback)
- Test SType determination with primary configuration (EXIM)
- Test SType determination with fallback configuration (QWIK)
- Test SType determination with no configuration (final fallback)

### 11.2 Integration Testing
- Test end-to-end scroll creation with dynamic scroll types
- Test AP location determination with dynamic SType values
- Test error handling when configuration is missing
- Test Marine scenario with dynamic SType

### 11.3 Regression Testing
- Verify existing Road and Rail transport mode functionality
- Verify existing AP location logic for non-EXIM scenarios
- Verify consignment category logic remains intact
- Verify fallback logic works correctly

### 11.4 Configuration Testing
- Test with various transport mode configurations
- Test with different SType values
- Test with inactive configurations (should be ignored)
- Test configuration changes take effect immediately

---

## 12. Sign-off

| Role | Name | Signature | Date |
|------|------|-----------|------|
| Business Analyst | | | |
| Functional Lead | | | |
| Technical Lead | | | |
| Project Manager | | | |
| Configuration Administrator | | | |

---

**Document End**

