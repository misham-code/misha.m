# ABAP Function Module Documentation: `/SKN/F_SO_ST_GET_DELTA`

## Overview

**Function Module Name:** `/SKN/F_SO_ST_GET_DELTA`

**Purpose:** This function module performs a delta comparison between two system run datasets. It retrieves data from source (older) and target (newer) runs, compares them using an optimized merge algorithm, and identifies records that were added or removed between the runs.

**Business Context:** Used in system orchestration scenarios to track changes between consecutive data extraction runs, enabling change detection and delta processing workflows.

---

## Function Interface

### Importing Parameters

| Parameter | Type | Optional | Description |
|-----------|------|----------|-------------|
| `SW_CLIENT` | `/SKN/E_SW_CLIENT` | Yes | Software client identifier for filtering runs |
| `IM_SI_ID` | `/SKN/E_SO_SI_ID` | Yes | System Interface ID to identify the data source |
| `IM_SI_VER` | `/SKN/E_SO_SI_ID_VER` | Yes | System Interface version number |
| `IM_RUN_NO_TARGET` | `/SKN/E_SO_RUN_NO` | Yes | Specific target (newer) run number to compare |
| `IM_RUN_NO_SOURCE` | `/SKN/E_SO_RUN_NO` | Yes | Specific source (older) run number to compare |

### Exporting Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `ST_ID` | `/SKN/E_SO_ST_ID` | System Task ID from target run |
| `ST_DESC` | `/SKN/E_SO_ST_DESC` | System Task description from target run |
| `START_DATE` | `/SKN/E_SO_LG_DATE` | Target run start date |
| `START_TIME` | `/SKN/E_SO_LG_TIME` | Target run start time |
| `ST_STATE` | `/SKN/E_SO_TASK_STATE` | Target run task state |
| `SI_ID` | `/SKN/E_SO_SI_ID` | System Interface ID used in comparison |
| `SI_VER` | `/SKN/E_SO_SI_ID_VER` | System Interface version used in comparison |
| `END_DATE` | `/SKN/E_SO_LG_DATE` | Target run end date |
| `END_TIME` | `/SKN/E_SO_LG_TIME` | Target run end time |
| `SW_SY` | `/SKN/E_SW_SYSTEM` | Software system identifier |
| `SO_PROCESS` | `/SKN/E_SO_PROCESS` | System orchestration process type |
| `EX_MESSAGES` | `/SKN/TT_MESSAGES` | Table of messages (errors, warnings, info) |

### Table Parameters

| Parameter | Structure | Optional | Description |
|-----------|-----------|----------|-------------|
| `T_SELECT` | `RSSELECT` | Yes | Selection criteria table |
| `SO_DATA_STRUCTURE` | `/SKN/S_SO_S_FCAT` | Yes | Field catalog structure definition |
| `SO_DELTA_SOURCE` | (Dynamic) | Yes | Records that exist in source but not in target (deleted) |
| `SO_DELTA_TARGET` | (Dynamic) | Yes | Records that exist in target but not in source (added) |

### Exceptions

| Exception | Description |
|-----------|-------------|
| `WRONG_PARAMETERS` | Invalid or inconsistent input parameters |
| `NO_DATA` | No data found or insufficient runs for comparison |

---

## Functional Description

### High-Level Process Flow

```
1. Input Validation & Run Selection
   ↓
2. Retrieve Field Catalog (Source)
   ↓
3. Create Dynamic Structures (Source)
   ↓
4. Retrieve Field Catalog (Target)
   ↓
5. Create Dynamic Structures (Target)
   ↓
6. Load Source Dataset
   ↓
7. Load Target Dataset
   ↓
8. Perform Delta Merge Comparison
   ↓
9. Return Results
```

### Detailed Processing Steps

#### Step 1: Retrieve and Validate Available Runs

**Functionality:**
- Validates source and target run numbers if provided
- Ensures both runs have matching `SI_ID` and `SI_VER`
- Queries database for finished runs (`st_state = 'F'`)
- Validates minimum of 2 runs exist for comparison

**Business Rules:**
- Only considers runs with status 'F' (Finished) for data consistency
- Source run must be older than target run
- Both runs must belong to the same System Interface (SI_ID) and version (SI_VER)

**Key Validations:**
```abap
- If source run not found → Error message
- If target run not found → Error message
- If SI_ID/SI_VER mismatch → Error message
- If less than 2 runs available → Error message
```

#### Step 2: Determine Source and Target Runs

**Logic Scenarios:**

1. **Both `IM_RUN_NO_SOURCE` and `IM_RUN_NO_TARGET` provided:**
   - Use the specified runs directly

2. **Only `IM_RUN_NO_TARGET` provided:**
   - Use specified run as target
   - Select immediately preceding run as source

3. **Only `IM_RUN_NO_SOURCE` provided:**
   - Use specified run as source
   - Select immediately following run as target

4. **Neither provided:**
   - Select two most recent runs
   - Most recent = target, second most recent = source

#### Step 3-4: Retrieve Field Catalogs

**Purpose:** Obtain metadata structure definitions for both runs

**Function Called:** `/SKN/F_SO_ST_GET_DATA` (with `format_only = 'X'`)

**Validates:**
- Field catalog retrieval succeeds
- Field catalog contains data

#### Step 5-6: Create Dynamic Data Structures

**Purpose:** Generate runtime data structures to handle dynamic field definitions

**Function Called:** `/SKN/F_SO_GENERATE_STRUCT`

**Returns:**
- Dynamic structure reference for single records
- Dynamic table reference for internal tables

**Field Symbols Used:**
- `<lt_source_data>` - Source dataset table
- `<ls_source_record>` - Source single record
- `<lt_target_data>` - Target dataset table
- `<ls_target_record>` - Target single record

#### Step 7-8: Load Datasets

**Source Dataset Loading:**
```abap
CALL FUNCTION '/SKN/F_SO_ST_GET_DATA'
  EXPORTING
    sw_client   = sw_client
    run_no      = ls_source_run-run_no
    format_only = ' '  " Get actual data
  TABLES
    so_data = <lt_source_data>
```

**Target Dataset Loading:**
- Loads data from target run
- Also retrieves all metadata for export parameters
- Sorts data for merge comparison

#### Step 9: Optimized Delta Merge Comparison

**Algorithm:** Merge Sort Comparison

**Complexity:**
- Time: O(n + m) where n = source records, m = target records
- Space: O(k) where k = delta records

**Algorithm Logic:**
```
Initialize: source_index = 1, target_index = 1

While (source_index <= source_lines OR target_index <= target_lines):

  If source exhausted:
    → Append target record to SO_DELTA_TARGET (new record)
    → Advance target_index
    
  If target exhausted:
    → Append source record to SO_DELTA_SOURCE (deleted record)
    → Advance source_index
    
  If target_record = source_record:
    → No change detected
    → Advance both indices
    
  If target_record > source_record:
    → Source record deleted
    → Append to SO_DELTA_SOURCE
    → Advance source_index
    
  If target_record < source_record:
    → Target record is new
    → Append to SO_DELTA_TARGET
    → Advance target_index
```

**Output Tables:**
- `SO_DELTA_SOURCE`: Records that exist in source but not in target (deleted records)
- `SO_DELTA_TARGET`: Records that exist in target but not in source (new records)

---

## Message Handling

The function uses the `EX_MESSAGES` table to communicate status, errors, and warnings. Instead of raising exceptions, messages are appended with appropriate types:

### Message Types

| Type | Constant | Description |
|------|----------|-------------|
| 'E' | `c_message_type_e` | Error message |
| 'W' | `c_message_type_w` | Warning message |
| 'I' | `c_message_type_i` | Information message |

### Key Messages

**Error Messages:**
- "No source run found for client {X}, source run_no {Y}"
- "No TARGET run found for client {X}, TARGET run_no {Y}"
- "runs has different si_id or si_ver..."
- "Specified run number {X} not found in available runs"
- "Failed to retrieve field catalog from source run {X}"
- "Failed to generate dynamic structures"

**Information Messages:**
- "Delta comparison: Source run {X} vs Target run {Y}"
- "Delta comparison completed. Source: {X} records, Target: {Y} records, Delta target: {Z} records, Delta source: {W} records"
- "No delta records found - datasets are identical"

---

## Technical Implementation Details

### Constants

| Constant | Value | Purpose |
|----------|-------|---------|
| `c_finished_state` | 'F' | Identifies completed/finished runs |
| `c_format_only` | 'X' | Flag to retrieve only metadata |
| `c_get_data` | ' ' | Flag to retrieve actual data |
| `c_end_marker` | 'END' | End of processing marker |
| `c_min_runs` | 2 | Minimum runs required for comparison |

### Data Structures

**Type Definitions:**
```abap
TYPES: BEGIN OF ty_run_info,
         run_no TYPE /skn/e_so_run_no,
       END OF ty_run_info.

TYPES: tt_run_info TYPE STANDARD TABLE OF ty_run_info 
       WITH NON-UNIQUE KEY run_no.
```

### Database Tables Referenced

- `/SKN/T_SO_ST` - System Orchestration Task table
  - Key fields: `SW_CLIENT`, `SI_ID`, `SI_VER`, `RUN_NO`, `ST_STATE`

---

## Usage Examples

### Example 1: Compare Two Most Recent Runs

```abap
DATA: lt_delta_source TYPE TABLE OF <dynamic_structure>,
      lt_delta_target TYPE TABLE OF <dynamic_structure>,
      lt_messages TYPE /skn/tt_messages.

CALL FUNCTION '/SKN/F_SO_ST_GET_DELTA'
  EXPORTING
    sw_client    = '100'
    im_si_id     = 'SALES_DATA'
    im_si_ver    = '001'
  IMPORTING
    ex_messages  = lt_messages
  TABLES
    so_delta_source = lt_delta_source
    so_delta_target = lt_delta_target
  EXCEPTIONS
    wrong_parameters = 1
    no_data         = 2
    OTHERS          = 3.

IF sy-subrc = 0.
  " Process delta records
  LOOP AT lt_delta_target ASSIGNING FIELD-SYMBOL(<delta>).
    " Handle new records
  ENDLOOP.
  
  LOOP AT lt_delta_source ASSIGNING FIELD-SYMBOL(<deleted>).
    " Handle deleted records
  ENDLOOP.
ENDIF.
```

### Example 2: Compare Specific Runs

```abap
CALL FUNCTION '/SKN/F_SO_ST_GET_DELTA'
  EXPORTING
    sw_client         = '100'
    im_run_no_source  = 1000
    im_run_no_target  = 1001
  IMPORTING
    st_id            = DATA(lv_task_id)
    si_id            = DATA(lv_si_id)
    ex_messages      = DATA(lt_messages)
  TABLES
    so_data_structure = DATA(lt_structure)
    so_delta_source   = lt_delta_source
    so_delta_target   = lt_delta_target
  EXCEPTIONS
    wrong_parameters  = 1
    no_data          = 2
    OTHERS           = 3.
```

---

## Performance Considerations

1. **Sorting:** Both source and target datasets are sorted before comparison, enabling O(n+m) merge algorithm instead of O(n*m) nested loop comparison

2. **Memory:** Uses field symbols and dynamic structures to avoid data copying overhead

3. **Database Access:** 
   - Single SELECT with ORDER BY for run retrieval
   - Efficient use of WHERE clause with indexed fields

4. **Algorithm Efficiency:** Linear time complexity makes it suitable for large datasets

---

## Error Handling Strategy

The function uses a **soft error handling** approach:
- Commented out `RAISE` statements indicate original exception-based design
- Current implementation uses message table (`EX_MESSAGES`) for error communication
- Uses `EXIT` statement to terminate processing on errors
- Caller should check `EX_MESSAGES` for error type messages

**Best Practice for Callers:**
```abap
LOOP AT lt_messages INTO DATA(ls_msg) WHERE type = 'E'.
  " Handle error messages
  EXIT.
ENDLOOP.
```

---

## Dependencies

### Function Modules Called

1. **`/SKN/F_SO_ST_GET_DATA`**
   - Purpose: Retrieve run data and metadata
   - Called twice (source and target runs)

2. **`/SKN/F_SO_GENERATE_STRUCT`**
   - Purpose: Generate dynamic structures from field catalog
   - Called twice (source and target structures)

### Database Tables

- `/SKN/T_SO_ST` - System Orchestration Task Status table

---

## Limitations and Constraints

1. **Run State:** Only processes runs with state 'F' (Finished)
2. **Minimum Runs:** Requires at least 2 completed runs
3. **Version Matching:** Source and target runs must have matching SI_ID and SI_VER
4. **Memory:** Large datasets may require sufficient memory allocation
5. **Dynamic Structures:** Relies on field catalog consistency between runs

---

## Maintenance Notes

### Code Quality Observations

1. **Commented Exception Handling:** Original `RAISE` statements are commented out; consider cleanup or documentation update

2. **Duplicate Logic:** Steps 3-4 and 5-6 have similar code for source and target; consider refactoring into a subroutine

3. **Hard-coded Values:** Consider moving message texts to message classes for internationalization

### Future Enhancement Suggestions

1. Add support for comparing non-consecutive runs
2. Implement partial delta (specific field comparison)
3. Add parallel processing for large datasets
4. Enhanced logging/tracing capabilities
5. Support for modified records (not just added/deleted)

---

## Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | - | - | Initial implementation with delta comparison |

---

## Support and Contact

For issues, questions, or enhancement requests related to this function module, contact the System Orchestration (SO) development team.

---

**Document Status:** Complete  
**Last Updated:** 2025-09-30  
**Document Version:** 1.0