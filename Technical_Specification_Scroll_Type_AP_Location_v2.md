# Technical Specification Document
## Scroll Type and AP Location Code Determination Enhancement (Dynamic Configuration)

**Document Version:** 2.0  
**Date:** 2025-01-27  
**Author:** Development Team  
**Status:** Draft

---

## 1. Overview

This document provides technical specifications for implementing enhanced scroll type determination and AP location code determination in the SAP ABAP Function Module `Z_SCME_CREATE_SCROLL` and the method `GET_AP_LOC` of class `ZCL_SCME_SCROLL_CREATE_VALID` using **dynamic configuration** from `ZLOG_EXEC_VAR` table.

---

## 2. Technical Requirements

### 2.1 Configuration Table: ZLOG_EXEC_VAR

#### 2.1.1 Table Structure
**Table:** `ZLOG_EXEC_VAR`  
**Key Fields Used:**
- `MANDT` - Client (CLNT, 3)
- `NAME` - Variant Variable Name (CHAR, 30) - Key field
- `VSART` - Shipping Type (CHAR, 2) - Used for scroll type determination (Transport Mode)
- `FUNCTION` - Function Code (CHAR, 4) - Used for scroll type determination (Movement Direction)
- `REMARKS` - Text field (CHAR, 72) - Contains configuration values
- `ACTIVE` - Active Indicator (CHAR, 1) - Filter condition

#### 2.1.2 Configuration Names
- **Scroll Type:** `ZSCM_SCROLLTYPE_FETCH`
- **AP Location Type:** `ZSCM_APCODETYPE_FETCH`
- **SType EXIM (Primary):** `ZSCM_SERTYPE_EXIM_FETCH`
- **SType QWIK (Fallback):** `ZSCM_SERTYPE_QWIK_FETCH`

---

## 3. Requirement 1: Dynamic Scroll Type Determination

### 3.1 Location of Changes
**File:** `Z_SCME_CREATE_SCROLL.txt`  
**Section:** Lines 579-585 (Scroll Type Determination Logic)

### 3.2 Current Code
```abap
IF <gfs_scrl>-movt_dir = 'I' .      "Inbound Case
  gw_yinvhead_i-scrtype  =  gc_inro.
  <gfs_scrl>-scroll_type =  gc_inro.
ELSEIF <gfs_scrl>-movt_dir = 'O'.   "Outbound Case
  gw_yinvhead_i-scrtype  =  gc_obro.
  <gfs_scrl>-scroll_type =  gc_obro.
ENDIF.
```

### 3.3 Proposed Code Changes

#### 3.3.1 Data Declaration
Add data declaration before the scroll type determination logic:

```abap
"BOC - Dynamic Scroll Type Determination from ZLOG_EXEC_VAR
DATA: lv_scroll_type_config TYPE textr,  "REMARKS field from ZLOG_EXEC_VAR
      lv_config_found       TYPE abap_bool.
"EOC - Dynamic Scroll Type Determination
```

#### 3.3.2 Enhanced Logic with Dynamic Configuration
Replace the existing IF-ELSEIF block with the following enhanced logic:

```abap
"BOC - Dynamic Scroll Type Determination from ZLOG_EXEC_VAR
CLEAR: lv_scroll_type_config, lv_config_found.

"Query ZLOG_EXEC_VAR for scroll type configuration
IF gw_trbl_det-transport_mode IS NOT INITIAL AND <gfs_scrl>-movt_dir IS NOT INITIAL.
  SELECT SINGLE remarks
    FROM zlog_exec_var CLIENT SPECIFIED
    INTO lv_scroll_type_config
    WHERE mandt   = sy-mandt
      AND name    = 'ZSCM_SCROLLTYPE_FETCH'
      AND vsart   = gw_trbl_det-transport_mode
      AND function = <gfs_scrl>-movt_dir
      AND active  = 'X'.
  IF sy-subrc = 0 AND lv_scroll_type_config IS NOT INITIAL.
    lv_config_found = abap_true.
  ENDIF.
ENDIF.

"Use configured scroll type if found, otherwise use fallback logic
IF lv_config_found = abap_true.
  "Use scroll type from configuration
  gw_yinvhead_i-scrtype  = lv_scroll_type_config.
  <gfs_scrl>-scroll_type = lv_scroll_type_config.
ELSE.
  "Fallback to existing logic based on movement direction
  IF <gfs_scrl>-movt_dir = 'I'.      "Inbound Case
    gw_yinvhead_i-scrtype  = gc_inro.
    <gfs_scrl>-scroll_type = gc_inro.
  ELSEIF <gfs_scrl>-movt_dir = 'O'.  "Outbound Case
    gw_yinvhead_i-scrtype  = gc_obro.
    <gfs_scrl>-scroll_type = gc_obro.
  ENDIF.
ENDIF.
"EOC - Dynamic Scroll Type Determination
```

### 3.4 Technical Details

#### 3.4.1 Data Flow
1. **Input Variables:**
   - `gw_trbl_det-transport_mode` - Transport mode ('MR', 'AR', 'RD', 'RL')
   - `<gfs_scrl>-movt_dir` - Movement direction ('I' or 'O')

2. **Configuration Query:**
   - Table: `ZLOG_EXEC_VAR`
   - Key: `NAME = 'ZSCM_SCROLLTYPE_FETCH'`, `VSART = transport_mode`, `FUNCTION = movement_direction`, `ACTIVE = 'X'`
   - Output: `REMARKS` field value

3. **Output Variables:**
   - `gw_yinvhead_i-scrtype` - Scroll type for invoice header
   - `<gfs_scrl>-scroll_type` - Scroll type in scroll details

#### 3.4.2 Error Handling
- If configuration not found, fallback to existing logic (INRO/OBRO)
- If `transport_mode` is initial, skip configuration query and use fallback
- If `movement_direction` is initial, skip configuration query and use fallback
- If `REMARKS` is initial after query, use fallback logic

---

## 4. Requirement 2: Dynamic AP Location Code Determination

### 4.1 Location of Changes

#### 4.1.1 Method: GET_AP_LOC
**File:** `zcl_scme_scroll_create_valid_get_ap_loc.txt`  
**Class:** `ZCL_SCME_SCROLL_CREATE_VALID`  
**Method:** `GET_AP_LOC`

#### 4.1.2 Function Module: Z_SCME_CREATE_SCROLL
**File:** `Z_SCME_CREATE_SCROLL.txt`  
**Section:** 
- Line 462: Method call to `get_ap_loc`
- Lines 536-569: AP location determination logic after fetch

### 4.2 Current Code - Method GET_AP_LOC

```abap
METHOD get_ap_loc.

  SELECT srno
         type
         stype
         werks
         ekgrp
         apcode
         scrldept
    FROM yscm_aploc CLIENT SPECIFIED
    INTO TABLE et_aploc
     WHERE mandt = sy-mandt
       AND type  = gc_ser
       AND stype = gc_logi.
  IF sy-subrc = 0.
    SORT et_aploc BY type stype.
  ELSE.
    CLEAR : gw_return.
    gw_return-type    = gc_e.
    gw_return-message = 'AP Locations not found'(001).
    APPEND gw_return TO et_return.
  ENDIF.

ENDMETHOD.
```

### 4.3 Proposed Code Changes - Method GET_AP_LOC

#### 4.3.1 Method Signature Enhancement
The method signature needs to be updated to accept Import/Export flags:

**Current Signature:**
```abap
METHOD get_ap_loc
  EXPORTING
    et_aploc TYPE ...
    et_return TYPE ...
  IMPORTING
    im_scroll_details TYPE ...
```

**Proposed Signature:**
```abap
METHOD get_ap_loc
  EXPORTING
    et_aploc TYPE ...
    et_return TYPE ...
  IMPORTING
    im_scroll_details TYPE ...
    im_import TYPE char1 OPTIONAL
    im_export TYPE char1 OPTIONAL
    im_rexport TYPE char1 OPTIONAL
```

#### 4.3.2 Enhanced Method Implementation with Dynamic SType

```abap
METHOD get_ap_loc.

  DATA: lv_type         TYPE char10,  "Local variable for Type determination
        lv_stype        TYPE char10,  "Local variable for SType determination
        lv_type_config  TYPE textr,   "REMARKS field from ZLOG_EXEC_VAR for Type
        lv_stype_config TYPE textr,   "REMARKS field from ZLOG_EXEC_VAR for SType
        lv_config_found TYPE abap_bool.

  "Determine Type dynamically from configuration
  CLEAR: lv_type_config, lv_type.
  SELECT SINGLE remarks
    FROM zlog_exec_var CLIENT SPECIFIED
    INTO lv_type_config
    WHERE mandt = sy-mandt
      AND name  = 'ZSCM_APCODETYPE_FETCH'
      AND active = 'X'.
  
  IF sy-subrc = 0 AND lv_type_config IS NOT INITIAL.
    lv_type = lv_type_config.
  ELSE.
    "Fallback to existing constant if configuration not found
    lv_type = gc_ser.
  ENDIF.

  "Determine SType based on Import/Export scenario with dynamic configuration
  IF im_import = 'X' OR im_export = 'X' OR im_rexport = 'X'.
    "EXIM Scenario - Try primary configuration first
    CLEAR: lv_stype_config, lv_config_found.
    
    "Primary: Query ZSCM_SERTYPE_EXIM_FETCH
    SELECT SINGLE remarks
      FROM zlog_exec_var CLIENT SPECIFIED
      INTO lv_stype_config
      WHERE mandt = sy-mandt
        AND name  = 'ZSCM_SERTYPE_EXIM_FETCH'
        AND active = 'X'.
    
    IF sy-subrc = 0 AND lv_stype_config IS NOT INITIAL.
      lv_stype = lv_stype_config.
      lv_config_found = abap_true.
    ELSE.
      "Fallback: Query ZSCM_SERTYPE_QWIK_FETCH
      SELECT SINGLE remarks
        FROM zlog_exec_var CLIENT SPECIFIED
        INTO lv_stype_config
        WHERE mandt = sy-mandt
          AND name  = 'ZSCM_SERTYPE_QWIK_FETCH'
          AND active = 'X'.
      
      IF sy-subrc = 0 AND lv_stype_config IS NOT INITIAL.
        lv_stype = lv_stype_config.
        lv_config_found = abap_true.
      ENDIF.
    ENDIF.
    
    "Final fallback if no configuration found
    IF lv_config_found = abap_false.
      lv_stype = gc_logi.  "Use existing 'LOGI' as final fallback
    ENDIF.
  ELSE.
    "Non-EXIM Scenario - Use existing logic
    lv_stype = gc_logi.
  ENDIF.

  "Fetch AP location using determined Type and SType
  SELECT srno
         type
         stype
         werks
         ekgrp
         apcode
         scrldept
    FROM yscm_aploc CLIENT SPECIFIED
    INTO TABLE et_aploc
     WHERE mandt = sy-mandt
       AND type  = lv_type   "Use determined Type
       AND stype = lv_stype.  "Use determined SType
  IF sy-subrc = 0.
    SORT et_aploc BY type stype.
  ELSE.
    CLEAR : gw_return.
    gw_return-type    = gc_e.
    gw_return-message = 'AP Locations not found'(001).
    APPEND gw_return TO et_return.
  ENDIF.

ENDMETHOD.
```

### 4.4 Function Module Changes

#### 4.4.1 Method Call Update
**Location:** Line 462

**Current Code:**
```abap
CALL METHOD zcl_scme_scroll_create_valid=>get_ap_loc
  EXPORTING
    im_scroll_details = gw_trbl_det
  IMPORTING
    et_aploc          = gt_aploc
    et_return         = et_return.
```

**Proposed Code:**
```abap
CALL METHOD zcl_scme_scroll_create_valid=>get_ap_loc
  EXPORTING
    im_scroll_details = gw_trbl_det
    im_import         = i_import
    im_export         = i_export
    im_rexport        = i_rexport
  IMPORTING
    et_aploc          = gt_aploc
    et_return         = et_return.
```

#### 4.4.2 AP Location Determination Logic Enhancement
**Location:** Lines 536-569

**Proposed Code Enhancement:**

Add logic to determine SType for Marine scenarios using dynamic configuration:

```abap
"BOC - Enhanced AP Location Determination with Dynamic SType
DATA: lv_stype_marine      TYPE char10,
      lv_stype_exim_config  TYPE textr,
      lv_exim_config_found  TYPE abap_bool.

"Determine SType for Marine scenario or use EXIM configuration
IF gw_trbl_det-transport_mode = 'MR' AND 
   ( <gfs_scrl>-movt_dir = 'I' OR <gfs_scrl>-movt_dir = 'O' ).
  "Marine scenario - Use EXIM configuration
  CLEAR: lv_stype_exim_config, lv_exim_config_found.
  
  "Try primary EXIM configuration
  SELECT SINGLE remarks
    FROM zlog_exec_var CLIENT SPECIFIED
    INTO lv_stype_exim_config
    WHERE mandt = sy-mandt
      AND name  = 'ZSCM_SERTYPE_EXIM_FETCH'
      AND active = 'X'.
  
  IF sy-subrc = 0 AND lv_stype_exim_config IS NOT INITIAL.
    lv_stype_marine = lv_stype_exim_config.
    lv_exim_config_found = abap_true.
  ELSE.
    "Try fallback QWIK configuration
    SELECT SINGLE remarks
      FROM zlog_exec_var CLIENT SPECIFIED
      INTO lv_stype_exim_config
      WHERE mandt = sy-mandt
        AND name  = 'ZSCM_SERTYPE_QWIK_FETCH'
        AND active = 'X'.
    
    IF sy-subrc = 0 AND lv_stype_exim_config IS NOT INITIAL.
      lv_stype_marine = lv_stype_exim_config.
      lv_exim_config_found = abap_true.
    ENDIF.
  ENDIF.
  
  "If no configuration found, use based on Import/Export scenario
  IF lv_exim_config_found = abap_false.
    IF i_import = 'X' OR i_export = 'X' OR i_rexport = 'X'.
      "Try to get from method fetch result or use default
      lv_stype_marine = gc_imports.  "Or use value from method if available
    ELSE.
      lv_stype_marine = gc_logi.
    ENDIF.
  ENDIF.
ELSE.
  "Non-Marine scenario - Use SType based on Import/Export scenario
  IF i_import = 'X' OR i_export = 'X' OR i_rexport = 'X'.
    "Get from EXIM configuration
    CLEAR: lv_stype_exim_config, lv_exim_config_found.
    SELECT SINGLE remarks
      FROM zlog_exec_var CLIENT SPECIFIED
      INTO lv_stype_exim_config
      WHERE mandt = sy-mandt
        AND name  = 'ZSCM_SERTYPE_EXIM_FETCH'
        AND active = 'X'.
    
    IF sy-subrc = 0 AND lv_stype_exim_config IS NOT INITIAL.
      lv_stype_marine = lv_stype_exim_config.
    ELSE.
      SELECT SINGLE remarks
        FROM zlog_exec_var CLIENT SPECIFIED
        INTO lv_stype_exim_config
        WHERE mandt = sy-mandt
          AND name  = 'ZSCM_SERTYPE_QWIK_FETCH'
          AND active = 'X'.
      
      IF sy-subrc = 0 AND lv_stype_exim_config IS NOT INITIAL.
        lv_stype_marine = lv_stype_exim_config.
      ELSE.
        lv_stype_marine = gc_logi.
      ENDIF.
    ENDIF.
  ELSE.
    lv_stype_marine = gc_logi.
  ENDIF.
ENDIF.

"BOC by Ravi M For Cd-8056401 Tr-RD2K9A3HA0 on 20.07.2021
READ TABLE lt_stlmntvr INTO lw_stlmntvr WITH KEY name = lc_consign_cat 
                                                 remarks = <gfs_scrl>-consignment_category BINARY SEARCH.
IF sy-subrc = 0.
  READ TABLE gt_aploc INTO gw_aploc WITH KEY type  = gc_ser
                                             stype = lv_stype_marine  "Use determined SType
                                             ekgrp = gw_ekgrp
                                             werks = <gfs_scrl>-werks BINARY SEARCH.
  IF sy-subrc = 0.
    gw_yinvhead_i-apcode   =  gw_aploc-apcode.
    gw_yinvhead_i-scrldept =  gw_aploc-scrldept.
    <gfs_scrl>-apcode      =  gw_aploc-apcode.
  ENDIF.
ELSE.
  "EOC by Ravi M For Cd-8056401 Tr-RD2K9A3HA0 on 20.07.2021
  READ TABLE gt_aploc INTO gw_aploc WITH KEY type  = gc_ser
                                             stype = lv_stype_marine  "Use determined SType
                                             ekgrp = gw_ekgrp.
  IF sy-subrc = 0.
    gw_yinvhead_i-apcode   =  gw_aploc-apcode.
    gw_yinvhead_i-scrldept =  gw_aploc-scrldept.
    <gfs_scrl>-apcode      =  gw_aploc-apcode.

  ELSE.   """" SOC by sagar on 20-03-2023

    READ TABLE gt_aploc INTO gw_aploc WITH KEY type  = gc_ser
                                               stype = lv_stype_marine  "Use determined SType
                                               werks = <gfs_scrl>-werks.
    IF sy-subrc = 0.
      gw_yinvhead_i-apcode   =  gw_aploc-apcode.
      gw_yinvhead_i-scrldept =  gw_aploc-scrldept.
      <gfs_scrl>-apcode      =  gw_aploc-apcode.
    ENDIF.                                      """" EOC by sagar on 20-03-2023
  ENDIF.
ENDIF.
"EOC - Enhanced AP Location Determination with Dynamic SType
```

---

## 5. Data Dictionary Considerations

### 5.1 Table: ZLOG_EXEC_VAR
- Ensure proper indexes exist on:
  - `NAME`, `VSART`, `FUNCTION`, `ACTIVE` (for scroll type query)
  - `NAME`, `ACTIVE` (for Type and SType queries)
- Verify field `REMARKS` (TEXTR, CHAR 72) can accommodate required values
- Verify field `VSART` (VSARTTR, CHAR 2) matches transport mode values
- Verify field `FUNCTION` (YSTATS, CHAR 4) can accommodate movement direction values

### 5.2 Table: YSCM_APLOC
- Ensure data exists for dynamically determined SType values
- Verify field length for STYPE field can accommodate values from configuration

### 5.3 Constants
- Existing constants (`gc_inro`, `gc_obro`, `gc_ser`, `gc_logi`) remain for fallback
- No new constants required (values come from configuration)

---

## 6. Implementation Steps

### 6.1 Step 1: Prepare Configuration Data
1. Create configuration records in `ZLOG_EXEC_VAR`:
   - Scroll type configurations for each transport mode and movement direction combination
   - Type configuration (ZSCM_APCODETYPE_FETCH) - optional, defaults to SER
   - SType EXIM configuration
   - SType QWIK fallback configuration
2. Verify all configurations have `ACTIVE = 'X'`
3. Test configuration queries independently

### 6.2 Step 2: Update Function Module - Scroll Type Logic
1. Add data declarations for configuration variables
2. Implement dynamic scroll type determination logic
3. Test with and without configuration records
4. Verify fallback logic works correctly

### 6.3 Step 3: Update Method GET_AP_LOC
1. Update method signature to include Import/Export parameters
2. Implement dynamic SType determination with primary and fallback queries
3. Update SELECT statement to use determined SType
4. Test method independently with various scenarios

### 6.4 Step 4: Update Function Module - Method Call
1. Update method call to pass Import/Export flags
2. Test method call with various flag combinations

### 6.5 Step 5: Update Function Module - AP Location Determination
1. Add dynamic SType determination logic for Marine scenarios
2. Update all READ TABLE statements to use determined SType
3. Test AP location determination in various scenarios

### 6.6 Step 6: Testing
1. Unit testing for each component
2. Integration testing for end-to-end scenarios
3. Regression testing for existing functionality
4. Configuration testing with various data combinations

---

## 7. Code Review Checklist

- [ ] Configuration queries use proper WHERE clauses
- [ ] Fallback logic is implemented correctly for Type (gc_ser) and SType (gc_logi)
- [ ] Method signature is updated correctly
- [ ] All SELECT and READ TABLE statements use determined Type and SType
- [ ] Error handling is maintained
- [ ] Code comments are added for new logic
- [ ] No hardcoded configuration values (use table queries)
- [ ] Type determination is implemented with fallback to gc_ser
- [ ] Existing functionality is not broken
- [ ] Performance considerations (indexes, single record queries)
- [ ] Client-specific queries use CLIENT SPECIFIED

---

## 8. Performance Considerations

### 8.1 Database Access
- Use `SELECT SINGLE` for configuration queries (single record expected)
- Ensure proper indexes exist on `ZLOG_EXEC_VAR`:
  - `NAME + VSART + FUNCTION + ACTIVE` (for scroll type)
  - `NAME + ACTIVE` (for Type and SType)
- Consider caching configuration values if performance is critical
- Type configuration query is executed once per method call

### 8.2 Table Operations
- `gt_aploc` table should be sorted appropriately before READ TABLE operations
- Use BINARY SEARCH where applicable
- Minimize number of database queries (use single SELECT where possible)

---

## 9. Error Handling

### 9.1 Scroll Type Determination
- If configuration not found, default to existing logic (INRO/OBRO)
- If `transport_mode` is initial, skip configuration query and use fallback
- If `movement_direction` is initial, skip configuration query and use fallback
- If `REMARKS` is initial after query, use fallback logic
- No exceptions raised for missing configuration

### 9.2 AP Location Determination
- If Type configuration not found, use `gc_ser` ('SER') as fallback
- If primary EXIM configuration not found, try fallback QWIK configuration
- If both SType configurations not found, use `gc_logi` as final fallback
- If AP location not found with determined Type and SType, existing error handling applies
- Error message: 'AP Locations not found' (existing)

---

## 10. Testing Scenarios

### 10.1 Scroll Type Testing
| Test Case | Transport Mode | Movement Direction | Configuration Exists | REMARKS Value | Expected Scroll Type |
|-----------|---------------|-------------------|---------------------|--------------|---------------------|
| TC-001 | MR | I | Yes | INSF | INSF |
| TC-002 | MR | O | Yes | OBSF | OBSF |
| TC-003 | MR | I | No | - | INRO (fallback) |
| TC-004 | AR | I | Yes | IASF | IASF |
| TC-005 | AR | O | Yes | OASF | OASF |
| TC-006 | AR | I | No | - | INRO (fallback) |
| TC-007 | RD | I | Yes | INRO | INRO |
| TC-008 | RD | O | Yes | OBRO | OBRO |
| TC-009 | RD | I | No | - | INRO (fallback) |
| TC-010 | Initial | I | N/A | - | INRO (fallback) |
| TC-011 | MR | Initial | N/A | - | INRO/OBRO (fallback) |
| TC-012 | MR | I | Yes (ACTIVE='') | - | INRO (fallback) |

### 10.2 AP Location Testing
| Test Case | Type Config | Import/Export | Primary Config | Fallback Config | Transport Mode | Expected Type | Expected SType |
|-----------|------------|--------------|----------------|-----------------|---------------|---------------|----------------|
| TC-011 | Yes | Yes | Yes | - | MR | From Config | From Primary Config |
| TC-012 | Yes | Yes | No | Yes | MR | From Config | From Fallback Config |
| TC-013 | Yes | Yes | No | No | MR | From Config | LOGI (final fallback) |
| TC-014 | No | No | - | - | MR | SER (fallback) | LOGI |
| TC-015 | Yes | Yes | Yes | - | RD | From Config | From Primary Config |
| TC-016 | No | No | - | - | RD | SER (fallback) | LOGI |
| TC-017 | Yes | No | - | - | MR | From Config | LOGI |

---

## 11. Configuration Maintenance Guide

### 11.1 Scroll Type Configuration
**Transaction:** SE16N or custom maintenance transaction  
**Table:** `ZLOG_EXEC_VAR`

**Required Records:**
```
MANDT | NAME                    | VSART | FUNCTION | REMARKS | ACTIVE
------|-------------------------|-------|----------|---------|--------
XXX   | ZSCM_SCROLLTYPE_FETCH  | MR    | I        | INSF    | X
XXX   | ZSCM_SCROLLTYPE_FETCH  | MR    | O        | OBSF    | X
XXX   | ZSCM_SCROLLTYPE_FETCH  | AR    | I        | IASF    | X
XXX   | ZSCM_SCROLLTYPE_FETCH  | AR    | O        | OASF    | X
XXX   | ZSCM_SCROLLTYPE_FETCH  | RD    | I        | INRO    | X
XXX   | ZSCM_SCROLLTYPE_FETCH  | RD    | O        | OBRO    | X
XXX   | ZSCM_SCROLLTYPE_FETCH  | RL    | I        | INRO    | X
XXX   | ZSCM_SCROLLTYPE_FETCH  | RL    | O        | OBRO    | X
```

**Notes:**
- `VSART` must match transport mode values exactly (MR, AR, RD, RL)
- `FUNCTION` must match movement direction values exactly (I for Inbound, O for Outbound)
- `REMARKS` contains the scroll type value to be used
- `ACTIVE` must be 'X' for configuration to be active
- Separate records are required for each combination of transport mode and movement direction

### 11.2 Type Configuration
**Transaction:** SE16N or custom maintenance transaction  
**Table:** `ZLOG_EXEC_VAR`

**Required Records:**
```
MANDT | NAME                    | REMARKS | ACTIVE
------|-------------------------|---------|--------
XXX   | ZSCM_APCODETYPE_FETCH  | SER     | X
```

**Notes:**
- `NAME` must be exactly 'ZSCM_APCODETYPE_FETCH'
- `REMARKS` contains the Type value to be used (typically 'SER')
- `ACTIVE` must be 'X' for configuration to be active
- If not maintained, system uses `gc_ser` ('SER') as fallback

### 11.2 SType Configuration
**Transaction:** SE16N or custom maintenance transaction  
**Table:** `ZLOG_EXEC_VAR`

**Required Records:**
```
MANDT | NAME                      | REMARKS  | ACTIVE
------|---------------------------|----------|--------
XXX   | ZSCM_SERTYPE_EXIM_FETCH  | IMPORTS  | X
XXX   | ZSCM_SERTYPE_QWIK_FETCH  | IMPORTS  | X
```

**Notes:**
- Primary configuration: `ZSCM_SERTYPE_EXIM_FETCH`
- Fallback configuration: `ZSCM_SERTYPE_QWIK_FETCH`
- `REMARKS` contains the SType value to be used
- `ACTIVE` must be 'X' for configuration to be active

---

## 12. Deployment Considerations

### 12.1 Pre-Deployment
1. Create configuration records in `ZLOG_EXEC_VAR` table in all systems:
   - Scroll type configurations
   - Type configuration (optional, defaults to SER)
   - SType configurations
2. Verify configuration data is correct
3. Ensure `YSCM_APLOC` table has data for configured Type and SType values
4. Backup existing code
5. Document configuration requirements

### 12.2 Deployment
1. Deploy class method changes
2. Deploy function module changes
3. Activate all objects
4. Verify configuration records exist
5. Test with sample data

### 12.3 Post-Deployment
1. Verify functionality in test system
2. Monitor for errors
3. Validate data consistency
4. Verify configuration changes take effect
5. Update configuration documentation

---

## 13. Rollback Plan

### 13.1 Rollback Steps
1. Revert function module changes
2. Revert class method changes
3. Verify existing functionality works
4. Configuration data can remain (will not be used after rollback)

### 13.2 Rollback Testing
- Test that existing scroll creation works after rollback
- Verify AP location determination returns to original behavior
- Verify fallback logic still works

---

## 14. Appendix

### 14.1 Code Locations Summary
| Component | File | Lines | Description |
|-----------|------|-------|-------------|
| Scroll Type Logic | Z_SCME_CREATE_SCROLL.txt | 579-585 | Dynamic scroll type determination |
| Method Call | Z_SCME_CREATE_SCROLL.txt | 462 | Updated method call with new parameters |
| AP Location Logic | Z_SCME_CREATE_SCROLL.txt | 536-569 | Dynamic AP location determination |
| Method Implementation | zcl_scme_scroll_create_valid_get_ap_loc.txt | 1-24 | Dynamic GET_AP_LOC method |

### 14.2 Variable Reference
| Variable | Type | Usage |
|----------|------|-------|
| `gw_trbl_det-transport_mode` | CHAR2 | Transport mode ('MR', 'AR', 'RD', 'RL') - Used in VSART field |
| `<gfs_scrl>-movt_dir` | CHAR1 | Movement direction ('I', 'O') - Passed in FUNCTION field of ZLOG_EXEC_VAR |
| `i_import` | CHAR1 | Import flag |
| `i_export` | CHAR1 | Export flag |
| `i_rexport` | CHAR1 | Re-export flag |
| `lv_scroll_type_config` | TEXTR | Scroll type from configuration (REMARKS field) |
| `lv_type` | CHAR10 | Determined Type for AP location (from config or gc_ser) |
| `lv_type_config` | TEXTR | Type from configuration (ZSCM_APCODETYPE_FETCH, REMARKS field) |
| `lv_stype_marine` | CHAR10 | Determined SType for Marine scenario |
| `lv_stype_exim_config` | TEXTR | SType from EXIM configuration (REMARKS field) |
| `lv_config_found` | ABAP_BOOL | Flag indicating configuration found |

**Notes:**
- The movement direction (`<gfs_scrl>-movt_dir`) is passed in the `FUNCTION` field of `ZLOG_EXEC_VAR` table when querying for scroll type configuration.
- The Type value is fetched from `ZLOG_EXEC_VAR` table with `NAME = 'ZSCM_APCODETYPE_FETCH'`, and if not found, defaults to `gc_ser` ('SER').

### 14.3 Configuration Query Examples

**Scroll Type Query:**
```abap
SELECT SINGLE remarks
  FROM zlog_exec_var CLIENT SPECIFIED
  INTO lv_scroll_type_config
  WHERE mandt   = sy-mandt
    AND name    = 'ZSCM_SCROLLTYPE_FETCH'
    AND vsart   = gw_trbl_det-transport_mode
    AND function = <gfs_scrl>-movt_dir
    AND active  = 'X'.
```

**SType EXIM Query (Primary):**
```abap
SELECT SINGLE remarks
  FROM zlog_exec_var CLIENT SPECIFIED
  INTO lv_stype_exim_config
  WHERE mandt = sy-mandt
    AND name  = 'ZSCM_SERTYPE_EXIM_FETCH'
    AND active = 'X'.
```

**SType QWIK Query (Fallback):**
```abap
SELECT SINGLE remarks
  FROM zlog_exec_var CLIENT SPECIFIED
  INTO lv_stype_exim_config
  WHERE mandt = sy-mandt
    AND name  = 'ZSCM_SERTYPE_QWIK_FETCH'
    AND active = 'X'.
```

**Type Query:**
```abap
SELECT SINGLE remarks
  FROM zlog_exec_var CLIENT SPECIFIED
  INTO lv_type_config
  WHERE mandt = sy-mandt
    AND name  = 'ZSCM_APCODETYPE_FETCH'
    AND active = 'X'.
```

---

**Document End**

