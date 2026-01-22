# Implementation Guide
## Scroll Type and AP Location Code Determination Enhancement

**Document Version:** 1.0  
**Date:** 2025-01-27  
**Author:** Development Team  
**Status:** Draft

---

## 1. Overview

This guide provides step-by-step instructions for implementing the dynamic scroll type determination and AP location code determination enhancements in the SAP system.

**Components to be Modified:**
- Function Module: `Z_SCME_CREATE_SCROLL`
- Class Method: `ZCL_SCME_SCROLL_CREATE_VALID=>GET_AP_LOC`
- Configuration Table: `ZLOG_EXEC_VAR` (data setup)

---

## 2. Prerequisites

### 2.1 System Requirements
- [ ] SAP ECC 6.0 EHP 6 or higher
- [ ] ABAP 731 SP0008 or higher
- [ ] Access to development system
- [ ] Transport request created
- [ ] Backup of existing code completed

### 2.2 Authorization Requirements
- [ ] SE80 - ABAP Workbench (for class method changes)
- [ ] SE37 - Function Builder (for function module changes)
- [ ] SE16N - Data Browser (for configuration data setup)
- [ ] SE11 - ABAP Dictionary (if table structure review needed)
- [ ] Transport Organizer access

### 2.3 Pre-Implementation Checklist
- [ ] Review Technical Specification document
- [ ] Review Functional Specification document
- [ ] Backup existing function module code
- [ ] Backup existing class method code
- [ ] Identify transport request number
- [ ] Notify stakeholders of implementation schedule
- [ ] Prepare test environment
- [ ] Prepare configuration data

---

## 3. Implementation Steps

### 3.1 Step 1: Prepare Configuration Data

#### 3.1.1 Access Configuration Table
**Transaction:** SE16N  
**Table:** `ZLOG_EXEC_VAR`

#### 3.1.2 Create Scroll Type Configuration Records

**Record 1: Marine Inbound**
```
MANDT: [Your Client, e.g., 100]
NAME: ZSCM_SCROLLTYPE_FETCH
VSART: MR
FUNCTION: I
REMARKS: INSF
ACTIVE: X
NUMB: 0001
```

**Record 2: Marine Outbound**
```
MANDT: [Your Client]
NAME: ZSCM_SCROLLTYPE_FETCH
VSART: MR
FUNCTION: O
REMARKS: OBSF
ACTIVE: X
NUMB: 0002
```

**Record 3: Air Inbound**
```
MANDT: [Your Client]
NAME: ZSCM_SCROLLTYPE_FETCH
VSART: AR
FUNCTION: I
REMARKS: IASF
ACTIVE: X
NUMB: 0003
```

**Record 4: Air Outbound**
```
MANDT: [Your Client]
NAME: ZSCM_SCROLLTYPE_FETCH
VSART: AR
FUNCTION: O
REMARKS: OASF
ACTIVE: X
NUMB: 0004
```

**Record 5: Road Inbound**
```
MANDT: [Your Client]
NAME: ZSCM_SCROLLTYPE_FETCH
VSART: RD
FUNCTION: I
REMARKS: INRO
ACTIVE: X
NUMB: 0005
```

**Record 6: Road Outbound**
```
MANDT: [Your Client]
NAME: ZSCM_SCROLLTYPE_FETCH
VSART: RD
FUNCTION: O
REMARKS: OBRO
ACTIVE: X
NUMB: 0006
```

**Record 7: Rail Inbound**
```
MANDT: [Your Client]
NAME: ZSCM_SCROLLTYPE_FETCH
VSART: RL
FUNCTION: I
REMARKS: INRO
ACTIVE: X
NUMB: 0007
```

**Record 8: Rail Outbound**
```
MANDT: [Your Client]
NAME: ZSCM_SCROLLTYPE_FETCH
VSART: RL
FUNCTION: O
REMARKS: OBRO
ACTIVE: X
NUMB: 0008
```

**Verification:**
- Execute query: `SELECT * FROM zlog_exec_var WHERE name = 'ZSCM_SCROLLTYPE_FETCH' AND active = 'X'`
- Verify 8 records are returned (or as per your transport modes)

#### 3.1.3 Create Type Configuration Record

**Record: AP Location Type**
```
MANDT: [Your Client]
NAME: ZSCM_APCODETYPE_FETCH
REMARKS: SER
ACTIVE: X
NUMB: 0001
```

**Note:** This is optional. If not created, system will use `gc_ser` ('SER') as fallback.

**Verification:**
- Execute query: `SELECT * FROM zlog_exec_var WHERE name = 'ZSCM_APCODETYPE_FETCH' AND active = 'X'`
- Verify 1 record is returned (if created)

#### 3.1.4 Create SType Configuration Records

**Record 1: Primary EXIM Configuration**
```
MANDT: [Your Client]
NAME: ZSCM_SERTYPE_EXIM_FETCH
REMARKS: IMPORTS
ACTIVE: X
NUMB: 0001
```

**Record 2: Fallback QWIK Configuration**
```
MANDT: [Your Client]
NAME: ZSCM_SERTYPE_QWIK_FETCH
REMARKS: IMPORTS
ACTIVE: X
NUMB: 0002
```

**Verification:**
- Execute query: `SELECT * FROM zlog_exec_var WHERE name IN ('ZSCM_SERTYPE_EXIM_FETCH', 'ZSCM_SERTYPE_QWIK_FETCH') AND active = 'X'`
- Verify 2 records are returned

#### 3.1.5 Verify AP Location Data

**Transaction:** SE16N  
**Table:** `YSCM_APLOC`

**Action:**
- Verify data exists for Type = 'SER' and SType = 'LOGI'
- Verify data exists for Type = 'SER' and SType = 'IMPORTS' (if using EXIM scenarios)
- If data doesn't exist, create test data as per business requirements

---

### 3.2 Step 2: Backup Existing Code

#### 3.2.1 Backup Function Module

**Transaction:** SE37  
**Function Module:** `Z_SCME_CREATE_SCROLL`

**Steps:**
1. Open function module in SE37
2. Go to Source Code tab
3. Copy entire source code
4. Save to backup file: `Z_SCME_CREATE_SCROLL_BACKUP_[DATE].txt`
5. Note the line numbers where changes will be made:
   - Line ~462: Method call
   - Line ~579-585: Scroll type determination
   - Line ~539-648: AP location determination

#### 3.2.2 Backup Class Method

**Transaction:** SE80  
**Class:** `ZCL_SCME_SCROLL_CREATE_VALID`  
**Method:** `GET_AP_LOC`

**Steps:**
1. Navigate to class in SE80
2. Open method GET_AP_LOC
3. Copy entire method code
4. Save to backup file: `GET_AP_LOC_BACKUP_[DATE].txt`

---

### 3.3 Step 3: Update Class Method GET_AP_LOC

#### 3.3.1 Open Class Method

**Transaction:** SE80  
**Path:** Class `ZCL_SCME_SCROLL_CREATE_VALID` → Methods → `GET_AP_LOC`

#### 3.3.2 Update Method Signature

**Current Signature:**
```abap
METHOD get_ap_loc
  EXPORTING
    et_aploc TYPE ...
    et_return TYPE ...
  IMPORTING
    im_scroll_details TYPE ...
```

**Updated Signature:**
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

**Steps:**
1. Go to method definition (not implementation)
2. Add new IMPORTING parameters:
   - `im_import TYPE char1 OPTIONAL`
   - `im_export TYPE char1 OPTIONAL`
   - `im_rexport TYPE char1 OPTIONAL`
3. Save and activate

#### 3.3.3 Update Method Implementation

**Replace the entire method implementation with:**

```abap
METHOD get_ap_loc.

  DATA: lv_type         TYPE char10,  "Local variable for Type
        lv_stype        TYPE char10,  "Local variable for SType
        lv_type_config  TYPE textr,   "REMARKS field for Type
        lv_stype_config TYPE textr,   "REMARKS field for SType
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

  "Determine SType based on Import/Export scenario with dynamic config
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

**Steps:**
1. Open method implementation
2. Replace entire method code with new code above
3. Check syntax (Ctrl+F2)
4. Save and activate
5. Verify no syntax errors

**Verification:**
- [ ] Method compiles without errors
- [ ] Method signature updated correctly
- [ ] All variables declared
- [ ] Syntax check passes

---

### 3.4 Step 4: Update Function Module Z_SCME_CREATE_SCROLL

#### 3.4.1 Open Function Module

**Transaction:** SE37  
**Function Module:** `Z_SCME_CREATE_SCROLL`

#### 3.4.2 Update Method Call (Line ~462)

**Locate the method call:**
```abap
CALL METHOD zcl_scme_scroll_create_valid=>get_ap_loc
  EXPORTING
    im_scroll_details = gw_trbl_det
  IMPORTING
    et_aploc          = gt_aploc
    et_return         = et_return.
```

**Replace with:**
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

**Steps:**
1. Navigate to line ~462
2. Locate the method call
3. Add the three new EXPORTING parameters
4. Save

#### 3.4.3 Update Scroll Type Determination (Line ~579-585)

**Locate the existing code:**
```abap
IF <gfs_scrl>-movt_dir = 'I' .      "Inbound Case
  gw_yinvhead_i-scrtype  =  gc_inro.
  <gfs_scrl>-scroll_type =  gc_inro.
ELSEIF <gfs_scrl>-movt_dir = 'O'.   "Outbound Case
  gw_yinvhead_i-scrtype  =  gc_obro.
  <gfs_scrl>-scroll_type =  gc_obro.
ENDIF.
```

**Replace with:**
```abap
"BOC - Dynamic Scroll Type Determination from ZLOG_EXEC_VAR
DATA: lv_scroll_type_config TYPE textr,  "REMARKS field
      lv_config_found       TYPE abap_bool.

CLEAR: lv_scroll_type_config, lv_config_found.

"Query ZLOG_EXEC_VAR for scroll type configuration
IF gw_trbl_det-transport_mode IS NOT INITIAL AND
   <gfs_scrl>-movt_dir IS NOT INITIAL.
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

**Steps:**
1. Navigate to line ~579
2. Locate the IF-ELSEIF block for scroll type determination
3. Replace with new code above
4. Ensure DATA declarations are placed before the logic
5. Save

**Verification:**
- [ ] Code replaced correctly
- [ ] DATA declarations added
- [ ] Query syntax correct
- [ ] Fallback logic intact

#### 3.4.4 Update AP Location Determination (Line ~539-648)

**Locate the existing code block starting with:**
```abap
"BOC by Ravi M For Cd-8056401 Tr-RD2K9A3HA0 on 20.07.2021
READ TABLE lt_stlmntvr INTO lw_stlmntvr WITH KEY name = lc_consign_cat
                                                 remarks = <gfs_scrl>-consignment_category BINARY SEARCH.
```

**Add the following code BEFORE the existing READ TABLE:**

```abap
"BOC - Enhanced AP Location Determination with Dynamic Type/SType
DATA: lv_type_marine       TYPE char10,
      lv_type_config       TYPE textr,
      lv_stype_marine      TYPE char10,
      lv_stype_exim_config  TYPE textr,
      lv_exim_config_found  TYPE abap_bool.

"Determine Type dynamically from configuration
CLEAR: lv_type_config, lv_type_marine.
SELECT SINGLE remarks
  FROM zlog_exec_var CLIENT SPECIFIED
  INTO lv_type_config
  WHERE mandt = sy-mandt
    AND name  = 'ZSCM_APCODETYPE_FETCH'
    AND active = 'X'.

IF sy-subrc = 0 AND lv_type_config IS NOT INITIAL.
  lv_type_marine = lv_type_config.
ELSE.
  "Fallback to existing constant if configuration not found
  lv_type_marine = gc_ser.
ENDIF.

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

  "If no configuration found, use fallback
  IF lv_exim_config_found = abap_false.
    lv_stype_marine = gc_logi.
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
```

**Then update the existing READ TABLE statements:**

**Find:**
```abap
READ TABLE gt_aploc INTO gw_aploc WITH KEY type  = gc_ser
                                           stype = gc_logi
```

**Replace all occurrences (3 places) with:**
```abap
READ TABLE gt_aploc INTO gw_aploc WITH KEY type  = lv_type_marine
                                           stype = lv_stype_marine
```

**Add at the end of the AP location determination block:**
```abap
"EOC - Enhanced AP Location Determination with Dynamic SType
```

**Steps:**
1. Navigate to line ~539 (before existing AP location logic)
2. Add the new DATA declarations and Type/SType determination logic
3. Find all three READ TABLE statements with `type = gc_ser` and `stype = gc_logi`
4. Replace `gc_ser` with `lv_type_marine`
5. Replace `gc_logi` with `lv_stype_marine` (in READ TABLE statements)
6. Add closing comment at the end
7. Save

**Verification:**
- [ ] All three READ TABLE statements updated
- [ ] Type determination logic added
- [ ] SType determination logic added
- [ ] No hardcoded values remain (except in fallback)

---

### 3.5 Step 5: Syntax Check and Activation

#### 3.5.1 Check Syntax

**For Function Module:**
1. In SE37, press **Ctrl+F2** or click **Check** button
2. Verify no syntax errors
3. Fix any errors if found

**For Class Method:**
1. In SE80, open method
2. Press **Ctrl+F2** or click **Check** button
3. Verify no syntax errors
4. Fix any errors if found

#### 3.5.2 Activate Objects

**Function Module:**
1. In SE37, click **Activate** button (or Ctrl+F3)
2. Verify activation successful
3. Note any warnings (review if needed)

**Class Method:**
1. In SE80, activate the class
2. Verify activation successful
3. Note any warnings (review if needed)

**Verification:**
- [ ] Function module activates without errors
- [ ] Class method activates without errors
- [ ] No activation warnings (or warnings reviewed and accepted)

---

### 3.6 Step 6: Unit Testing

#### 3.6.1 Test Scroll Type Determination

**Test 1: Marine Inbound with Configuration**
1. Ensure configuration exists in ZLOG_EXEC_VAR
2. Call function module with:
   - `i_marin = 'X'`
   - `im_trbl_det-transport_mode = 'MR'`
   - `im_trbl_det-movt_dir = 'I'`
3. Verify scroll type = 'INSF'

**Test 2: Configuration Not Found**
1. Delete or deactivate configuration
2. Call function module with same parameters
3. Verify scroll type = 'INRO' (fallback)

#### 3.6.2 Test AP Location Type Determination

**Test 1: Type Configuration Found**
1. Ensure Type configuration exists
2. Set breakpoint in method GET_AP_LOC
3. Call function module
4. Verify `lv_type = 'SER'` (from configuration)

**Test 2: Type Configuration Not Found**
1. Delete Type configuration
2. Set breakpoint in method
3. Call function module
4. Verify `lv_type = gc_ser` (fallback)

#### 3.6.3 Test AP Location SType Determination

**Test 1: EXIM Scenario - Primary Config**
1. Ensure primary EXIM configuration exists
2. Call function module with `i_import = 'X'`
3. Set breakpoint in method
4. Verify `lv_stype = 'IMPORTS'` (from primary config)

**Test 2: EXIM Scenario - Fallback Config**
1. Delete primary config, keep fallback
2. Call function module with `i_import = 'X'`
3. Verify `lv_stype = 'IMPORTS'` (from fallback config)

**Test 3: EXIM Scenario - Final Fallback**
1. Delete both configurations
2. Call function module with `i_import = 'X'`
3. Verify `lv_stype = gc_logi` (final fallback)

---

### 3.7 Step 7: Integration Testing

#### 3.7.1 End-to-End Test

**Test Scenario: Marine Inbound with All Configurations**

**Prerequisites:**
- All configurations exist in ZLOG_EXEC_VAR
- AP location data exists in YSCM_APLOC
- Service PO exists
- All validations pass

**Steps:**
1. Call function module `Z_SCME_CREATE_SCROLL` with complete data:
   ```
   i_marin = 'X'
   i_import = 'X'
   im_trbl_det-transport_mode = 'MR'
   im_trbl_det-movt_dir = 'I'
   im_trbl_det-tr_billno = 'TEST001'
   im_trbl_det-lifnr = '0000001001'
   im_trbl_det-werks = '1000'
   im_trbl_det-bukrs = '1000'
   [Other required fields]
   ```

2. Verify results:
   - Scroll type = 'INSF'
   - AP location Type = 'SER'
   - AP location SType = 'IMPORTS'
   - AP code assigned correctly
   - Scroll created successfully

**Verification Points:**
- [ ] Scroll type determined correctly
- [ ] AP location Type determined correctly
- [ ] AP location SType determined correctly
- [ ] AP code assigned
- [ ] Scroll number generated
- [ ] No errors in return table

---

### 3.8 Step 8: Transport to Quality System

#### 3.8.1 Release Transport Request

**Steps:**
1. Go to SE10 (Transport Organizer)
2. Open your transport request
3. Verify all objects are included:
   - Function module: `Z_SCME_CREATE_SCROLL`
   - Class: `ZCL_SCME_SCROLL_CREATE_VALID`
4. Release transport request
5. Note transport request number

#### 3.8.2 Import to Quality System

**Steps:**
1. Import transport request to Quality system
2. Verify objects are imported successfully
3. Activate objects in Quality system
4. Verify no activation errors

#### 3.8.3 Setup Configuration Data in Quality

**Steps:**
1. Create same configuration records in Quality system
2. Verify data matches Development system
3. Test functionality in Quality system

---

### 3.9 Step 9: User Acceptance Testing (UAT)

#### 3.9.1 Prepare UAT Test Cases

**Provide to Business Users:**
- Test scenarios document
- Test data setup guide
- Expected results for each scenario

#### 3.9.2 Execute UAT

**Steps:**
1. Business users execute test cases
2. Document test results
3. Report any issues
4. Sign off on successful tests

---

### 3.10 Step 10: Production Deployment

#### 3.10.1 Pre-Deployment Checklist

- [ ] All UAT tests passed
- [ ] Configuration data prepared for Production
- [ ] Backup of Production code completed
- [ ] Deployment window scheduled
- [ ] Rollback plan prepared
- [ ] Support team notified

#### 3.10.2 Deploy to Production

**Steps:**
1. Import transport request to Production
2. Activate objects
3. Create configuration records in Production
4. Verify configuration data
5. Execute smoke test
6. Monitor for errors

#### 3.10.3 Post-Deployment Verification

**Steps:**
1. Execute test transaction
2. Verify scroll creation works
3. Verify scroll types are correct
4. Verify AP locations are correct
5. Monitor application logs
6. Verify no errors in system logs

---

## 4. Rollback Procedure

### 4.1 Rollback Steps

**If issues occur after deployment:**

1. **Revert Code Changes:**
   - Restore function module from backup
   - Restore class method from backup
   - Activate objects

2. **Remove Configuration Data (Optional):**
   - Configuration data can remain (won't be used after rollback)
   - Or delete test configurations if needed

3. **Verify Rollback:**
   - Test existing functionality
   - Verify original behavior restored

### 4.2 Rollback Verification

- [ ] Function module restored to original code
- [ ] Class method restored to original code
- [ ] Existing functionality works correctly
- [ ] No errors in system

---

## 5. Post-Implementation Tasks

### 5.1 Documentation Updates

- [ ] Update technical documentation
- [ ] Update user documentation
- [ ] Document configuration maintenance procedures
- [ ] Update change log

### 5.2 Knowledge Transfer

- [ ] Conduct training session for support team
- [ ] Document configuration maintenance process
- [ ] Provide troubleshooting guide
- [ ] Share test scenarios with support team

### 5.3 Monitoring

**First Week After Deployment:**
- [ ] Monitor scroll creation transactions
- [ ] Check for errors in application logs
- [ ] Verify scroll types are correct
- [ ] Verify AP locations are correct
- [ ] Collect user feedback

---

## 6. Troubleshooting Guide

### 6.1 Common Issues

#### Issue 1: Configuration Not Found
**Symptom:** System uses fallback values

**Solution:**
1. Verify configuration exists in ZLOG_EXEC_VAR
2. Check NAME field is exactly correct (case-sensitive)
3. Verify ACTIVE = 'X'
4. Verify MANDT matches current client
5. Check VSART and FUNCTION values match exactly

#### Issue 2: Syntax Error in Function Module
**Symptom:** Function module won't activate

**Solution:**
1. Check all DATA declarations are placed correctly
2. Verify variable names match
3. Check query syntax
4. Verify CLIENT SPECIFIED is used correctly
5. Check for missing ENDIF or ENDIF statements

#### Issue 3: Method Signature Error
**Symptom:** Method call fails with parameter error

**Solution:**
1. Verify method signature is updated in class definition
2. Verify method implementation matches signature
3. Check function module method call includes all parameters
4. Verify parameter types match

#### Issue 4: AP Location Not Found
**Symptom:** AP location not determined

**Solution:**
1. Verify Type and SType are determined correctly
2. Check YSCM_APLOC table has data for determined values
3. Verify WERKS and EKGRP values match
4. Check consignment category logic (if applicable)

---

## 7. Implementation Checklist

### Pre-Implementation
- [ ] Review Technical Specification
- [ ] Review Functional Specification
- [ ] Prepare transport request
- [ ] Backup existing code
- [ ] Prepare configuration data
- [ ] Schedule implementation window

### Implementation
- [ ] Create configuration data in ZLOG_EXEC_VAR
- [ ] Update class method signature
- [ ] Update class method implementation
- [ ] Update function module method call
- [ ] Update function module scroll type logic
- [ ] Update function module AP location logic
- [ ] Syntax check all code
- [ ] Activate all objects

### Testing
- [ ] Unit test scroll type determination
- [ ] Unit test AP location Type determination
- [ ] Unit test AP location SType determination
- [ ] Integration test end-to-end flow
- [ ] Regression test existing scenarios
- [ ] User acceptance testing

### Deployment
- [ ] Release transport request
- [ ] Import to Quality system
- [ ] Test in Quality system
- [ ] UAT sign-off obtained
- [ ] Deploy to Production
- [ ] Verify in Production
- [ ] Monitor post-deployment

### Post-Implementation
- [ ] Update documentation
- [ ] Knowledge transfer to support team
- [ ] Monitor first week
- [ ] Collect feedback
- [ ] Close implementation

---

## 8. Support and Maintenance

### 8.1 Configuration Maintenance

**Adding New Scroll Type:**
1. Create new record in ZLOG_EXEC_VAR:
   - NAME = 'ZSCM_SCROLLTYPE_FETCH'
   - VSART = transport mode
   - FUNCTION = movement direction
   - REMARKS = scroll type value
   - ACTIVE = 'X'

**Changing Scroll Type:**
1. Update REMARKS field in existing record
2. Verify ACTIVE = 'X'
3. Test immediately

**Adding New SType:**
1. Create record in ZLOG_EXEC_VAR:
   - NAME = 'ZSCM_SERTYPE_EXIM_FETCH' (primary)
   - REMARKS = SType value
   - ACTIVE = 'X'

### 8.2 Support Contacts

**Technical Issues:**
- Development Team: [Contact]
- Technical Lead: [Contact]

**Configuration Issues:**
- Functional Team: [Contact]
- Business Analyst: [Contact]

**Production Issues:**
- Support Team: [Contact]
- On-Call: [Contact]

---

## 9. Appendix

### 9.1 Code Change Summary

| Component | Location | Change Type | Description |
|-----------|----------|-------------|-------------|
| Class Method | Signature | Modified | Added Import/Export parameters |
| Class Method | Implementation | Replaced | Dynamic Type and SType determination |
| Function Module | Line ~462 | Modified | Updated method call with new parameters |
| Function Module | Line ~579 | Replaced | Dynamic scroll type determination |
| Function Module | Line ~539 | Added | Dynamic Type and SType determination |
| Function Module | Line ~617,628,638 | Modified | Updated READ TABLE to use dynamic values |

### 9.2 Configuration Summary

| Configuration Name | Purpose | Required | Fallback |
|-------------------|---------|----------|----------|
| ZSCM_SCROLLTYPE_FETCH | Scroll type determination | Yes | INRO/OBRO |
| ZSCM_APCODETYPE_FETCH | AP location Type | No | gc_ser |
| ZSCM_SERTYPE_EXIM_FETCH | AP location SType (primary) | No | ZSCM_SERTYPE_QWIK_FETCH |
| ZSCM_SERTYPE_QWIK_FETCH | AP location SType (fallback) | No | gc_logi |

### 9.3 Key Variables

| Variable | Type | Usage |
|----------|------|-------|
| `lv_scroll_type_config` | TEXTR | Scroll type from configuration |
| `lv_type_marine` | CHAR10 | Type for AP location |
| `lv_stype_marine` | CHAR10 | SType for AP location |
| `lv_type_config` | TEXTR | Type from configuration |
| `lv_stype_exim_config` | TEXTR | SType from configuration |
| `lv_config_found` | ABAP_BOOL | Flag for configuration found |

---

**Document End**

