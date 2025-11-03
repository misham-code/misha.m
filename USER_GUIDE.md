# /SKN/F_SO_ST_GET_DELTA ? Quick Guide

- **Purpose**: compare two completed Solution Order runs and list records that were added or removed between them.
- **When to use**: you need the delta between the latest run and its predecessor, or between two specific run numbers.

## Inputs
- `SW_CLIENT` (optional): SAP client to read from; required if run numbers are provided.
- `IM_SI_ID`, `IM_SI_VER` (optional): solution interface identifier/version when runs are not supplied.
- `IM_RUN_NO_SOURCE`, `IM_RUN_NO_TARGET` (optional): older/newer run numbers. Provide both for an explicit comparison, or just the target to compare with its previous run.

## Outputs
- Metadata of the target run (`ST_ID`, `ST_DESC`, timing, status, interface IDs, `SO_PROCESS`).
- `SO_DELTA_SOURCE`: records that exist only in the source (deleted in target).
- `SO_DELTA_TARGET`: records that exist only in the target (new or changed).
- `EX_MESSAGES`: success info, warnings, or errors encountered during processing.

## Processing Summary
1. Validates supplied run numbers and finds the appropriate source/target pair (falls back to the two most recent finished runs).
2. Builds the field catalog once and generates matching dynamic structures for both runs.
3. Loads and sorts source and target datasets, then performs a single-pass merge comparison to collect delta records.
4. Reports completion details and adds informative messages when no differences are found.

## Typical Errors
- **Wrong parameters**: run numbers do not exist, do not match the same interface, or lack a comparable partner run.
- **No data**: the client/interface has fewer than two finished runs or the data retrieval call fails.
