# Bronze Ingestion Guide

> **Status: Current implementation guide for the Bronze ingestion layer.**
>
> This document reflects the implemented PSA Bronze ingestion workflow and the shared ingestion-control framework. PSGC and DOT ingestion should reuse the same control pattern.

## Purpose

This guide explains how approved source files move from their source location into Bronze Delta tables in Databricks, how ingestion runs are tracked, and how team members can verify the results.

The goals of the Bronze ingestion layer are to:

- preserve source data as closely as practical;
- maintain clear source lineage;
- make ingestion safe to rerun;
- record run-level, file-level, and error-level operational metadata;
- keep Bronze responsibilities separate from Silver cleaning and standardization;
- provide a repeatable pattern for PSA, PSGC, and DOT ingestion.

Finalized business and analytical questions remain documented separately. This guide focuses only on ingestion and Bronze preparation.

---

## Current Architecture

The project uses the following Unity Catalog structure:

```text
tuklas_dev
├── 00-sources
├── 01_ingestion
├── 02_bronze
├── 03_silver
├── 04_gold
├── 05_analytics
├── 06_data_quality
├── default
└── information_schema
```

The responsibilities of the first three layers are:

| Schema | Responsibility |
| --- | --- |
| `00-sources` | Source registry and source definitions — what sources exist |
| `01_ingestion` | Operational ingestion metadata — what happened during ingestion |
| `02_bronze` | Raw ingested source data — what was actually loaded |

Bronze does not perform business-level cleaning, geographic harmonization, KPI calculation, or analytical transformation.

---

## Source Access

Source files are currently read directly from Google Drive using the existing Unity Catalog connection:

```text
capstone_gdrive
```

No service-account keys, tokens, or manually stored credentials are required in the ingestion notebooks.

### Current Source Folders

PSA:

```text
https://drive.google.com/drive/folders/1hBSj2dHu1ylbEjRjLzzxwMQ9CBXItiQs
```

PSGC:

```text
https://drive.google.com/drive/folders/1Xtf804IXEblOVhbxeZitGgL-UjPy8AsW
```

DOT:

```text
https://drive.google.com/drive/folders/12pSL9t9MsD4wy7Jj0L_Qadb5UsfxZJ3m
```

---

## Dataset ID and Naming Convention

Use underscores consistently in dataset IDs, filenames, metadata, and table names.

Correct:

```text
DS_01
DS_08
DS_34
```

Do not use:

```text
DS-01
DS-08
DS-34
```

### PSA File Naming Convention

General pattern:

```text
DS_XX_PSA_<DESCRIPTIVE_NAME>_<YEAR_RANGE>.<extension>
```

Examples:

```text
DS_01_PSA_PTSA_Statistical_Tables_2000_2025.xlsx
DS_02_PSA_Tourism_Statistical_Classification_2025.xlsx
DS_18_PSA_Projected_Mid_Year_Population_2020_2025.xlsx
DS_26_PSA_Table_3_63_Accommodation_Food_Service_GVA_2022_2024.csv
DS_32_PSA_Table_3_95_Accommodation_Food_Service_GVA_Constant_Prices_2019_2023.csv
```

Special case for the PSY workbook:

```text
DS_08_to_DS_17_PSA_PSY_Tables_8_1_to_8_10.xlsx
```

This is one physical workbook containing ten logical datasets.

---

## Implemented Notebooks

### `00_ingestion_setup.ipynb`

Location:

```text
notebooks/01-ingestion/00_ingestion_setup.ipynb
```

Responsibility:

- verifies the target ingestion schema;
- creates the shared ingestion-control tables;
- does not load source data;
- does not insert dummy rows;
- uses `CREATE TABLE IF NOT EXISTS`;
- does not drop, truncate, or overwrite historical control data.

It creates:

```text
`tuklas_dev`.`01_ingestion`.`ingestion_run_log`
`tuklas_dev`.`01_ingestion`.`file_manifest`
`tuklas_dev`.`01_ingestion`.`ingestion_errors`
```

### `01_ingest_psa.ipynb`

Location:

```text
notebooks/01-ingestion/01_ingest_psa.ipynb
```

Responsibility:

- discovers PSA files from Google Drive;
- reads CSV, Excel, and PDF sources;
- writes PSA Bronze Delta tables;
- validates required source-lineage metadata;
- logs each run;
- records each logical output in the manifest;
- records actual technical failures;
- finalizes the run as `SUCCESS`, `PARTIAL_SUCCESS`, or `FAILED`.

### Planned Next Notebooks

The next ingestion notebooks should reuse the same control framework:

```text
02_ingest_psgc.ipynb
03_ingest_dot.ipynb
04_bronze_validation.ipynb
```

Do not create a second ingestion-control framework for PSGC or DOT.

---

## Ingestion-Control Tables

### `ingestion_run_log`

Purpose:

> One row per ingestion notebook execution.

Columns:

```text
run_id
notebook_name
source_system
source_folder
started_at
completed_at
files_discovered
files_ingested
files_skipped
files_failed
tables_created
status
notes
```

Run statuses:

```text
STARTED
SUCCESS
PARTIAL_SUCCESS
FAILED
```

Important metric definitions:

- `files_discovered` = number of unique physical source files represented in the run;
- `files_ingested` = number of unique physical source files successfully processed;
- `files_skipped` = number of unique physical files unresolved or intentionally skipped;
- `files_failed` = number of unique physical files with at least one failed logical output;
- `tables_created` = number of distinct successfully ingested Bronze destination tables.

The file-level status precedence is:

```text
FAILED > INGESTED > SKIPPED
```

A physical file is:

- `FAILED` if any logical output from that file failed;
- `INGESTED` if no output failed and at least one output was ingested;
- `SKIPPED` if no output failed, no output was ingested, and an output is `SKIPPED` or `NEEDS_REVIEW`.

### `file_manifest`

Purpose:

> One row per logical ingestion output.

This means one physical file can create multiple manifest rows.

Examples:

- DS_01 workbook → 10 manifest rows;
- DS_02 workbook → 3 manifest rows;
- PSY workbook → 10 manifest rows;
- DS_18 Excel and DS_18 PDF → separate manifest rows.

Columns:

```text
run_id
notebook_name
source_dataset_id
source_system
source_file
source_sheet
source_path
destination_table
row_count
column_count
status
discovered_at
ingested_at
notes
```

Manifest statuses:

```text
DISCOVERED
INGESTED
SKIPPED
NEEDS_REVIEW
FAILED
```

### `ingestion_errors`

Purpose:

> Records actual technical ingestion failures and exceptions.

Typical failures include:

- Google Drive read failure;
- CSV parsing failure;
- Excel or worksheet parsing failure;
- Delta write failure;
- PDF binary ingestion failure.

Columns:

```text
run_id
notebook_name
source_dataset_id
source_system
source_file
source_sheet
source_path
destination_table
error_type
error_message
error_timestamp
notes
```

Do not create fake error records.

---

## Run Lifecycle

Each ingestion notebook follows this lifecycle.

### 1. Start the Run

At the beginning of execution:

1. Generate one unique `run_id`.
2. Insert one `STARTED` row into `ingestion_run_log`.
3. Discover the source files.

Example run ID:

```text
psa_20261002_030530_d0fa0433
```

### 2. Process Independent Sources

For each independent source or worksheet:

1. Read the source.
2. Preserve the required source values.
3. Add Bronze metadata.
4. Write the Bronze Delta table.
5. Validate the target table.
6. Record the logical output in `file_manifest`.

If processing fails:

1. record `FAILED` in `file_manifest`;
2. write the actual exception to `ingestion_errors`;
3. continue to the next independent source where safe.

### 3. Finalize the Run

At the end:

1. calculate run metrics from persisted manifest records;
2. roll logical outputs up to physical files;
3. determine the final run status;
4. update the same `ingestion_run_log` row;
5. display the current run log, manifest, and errors.

Do not append a second final run-log row.

---

## Manifest Idempotency

`file_manifest` uses `MERGE INTO` to prevent duplicate records when the same logical output is reexecuted within the same run.

The logical match key is:

```text
run_id
source_dataset_id
source_file
COALESCE(source_sheet, '')
destination_table
```

If the same cell is executed twice under the same `run_id`, the manifest row is updated instead of duplicated.

A later full notebook execution gets a new `run_id`, so historical runs remain visible.

---

## Bronze Metadata

Bronze outputs include:

```text
_source_dataset_id
_source_file
_source_sheet
_source_path
_source_system
_ingested_at
```

Purpose:

| Metadata | Purpose |
| --- | --- |
| `_source_dataset_id` | Canonical source inventory ID |
| `_source_file` | Exact source filename |
| `_source_sheet` | Original worksheet name; NULL for non-Excel sources |
| `_source_path` | Source Google Drive path/URL |
| `_source_system` | Source agency/system, such as `PSA` |
| `_ingested_at` | Ingestion timestamp |

These fields provide row-level lineage back to the source.

---

## PSA Sources Currently Implemented

The PSA notebook currently processes 12 physical source files that produce 32 logical Bronze outputs.

### Physical Files

```text
DS_01 workbook
DS_02 workbook
DS_08_to_DS_17 PSY workbook
DS_18 Excel
DS_18 PDF
DS_26 CSV
DS_27 CSV
DS_28 CSV
DS_29 CSV
DS_30 CSV
DS_31 CSV
DS_32 CSV
```

Therefore:

```text
Physical source files = 12
```

### Logical Bronze Outputs

```text
DS_01             = 10 tables
DS_02             = 3 tables
DS_08 to DS_17    = 10 tables
DS_18 Excel       = 1 table
DS_18 PDF         = 1 table
DS_26 to DS_32    = 7 tables
--------------------------------
Total             = 32 tables
```

Do not confuse physical-file counts with logical output counts.

---

## PSA Bronze Table Naming

### DS_01

```text
psa_ds_01_table_01
psa_ds_01_table_02
psa_ds_01_table_03
psa_ds_01_table_04
psa_ds_01_table_05
psa_ds_01_table_06
psa_ds_01_table_07
psa_ds_01_table_08
psa_ds_01_table_09
psa_ds_01_table_10
```

### DS_02

```text
psa_ds_02_metadata
psa_ds_02_tourism_industries
psa_ds_02_tourism_products
```

### DS_08 to DS_17

```text
T8_1  -> DS_08 -> psa_ds_08
T8_2  -> DS_09 -> psa_ds_09
T8_3  -> DS_10 -> psa_ds_10
T8_4  -> DS_11 -> psa_ds_11
T8_5  -> DS_12 -> psa_ds_12
T8_6  -> DS_13 -> psa_ds_13
T8_7  -> DS_14 -> psa_ds_14
T8_8  -> DS_15 -> psa_ds_15
T8_9  -> DS_16 -> psa_ds_16
T8_10 -> DS_17 -> psa_ds_17
```

### DS_18

```text
psa_ds_18
psa_ds_18_pdf
```

The Excel source is structured Bronze data.

The PDF is stored as a separate binary/unstructured Bronze table. No PDF-to-table analytical extraction is performed in this notebook.

### DS_26 to DS_32

```text
psa_ds_26
psa_ds_27
psa_ds_28
psa_ds_29
psa_ds_30
psa_ds_31
psa_ds_32
```

---

## Bronze Preservation Rules

Bronze should preserve the source as closely as practical.

Do not perform Silver-level business cleaning in Bronze.

For PSA, the notebook intentionally does not:

- standardize province or city names;
- join source records to PSGC;
- remove leading hierarchy markers;
- silently repair source Unicode artifacts;
- classify geographic levels;
- calculate KPIs;
- aggregate source records;
- resolve cross-source comparability.

Technical transformations required to load the data are allowed, such as:

- sanitizing invalid Delta column names;
- reading the correct Excel header row;
- converting mixed-type Excel columns to strings for preservation;
- dropping completely empty worksheet rows;
- preserving multi-level headers as source data where needed.

Any analytical cleaning belongs downstream.

---

## Safe Rerun Behavior

Bronze writes use fixed target table names and Delta overwrite behavior.

Conceptually:

```python
df.write \
    .format("delta") \
    .mode("overwrite") \
    .option("overwriteSchema", "true") \
    .saveAsTable(...)
```

This means rerunning the notebook refreshes the same Bronze target rather than blindly appending duplicate source rows.

Example:

```text
Run 1: psa_ds_32 = 133 rows
Run 2: psa_ds_32 = refreshed to 133 rows
```

It does not become:

```text
266 rows
```

Delta may create a new table version internally. That is expected and is not the same as duplicate active records.

---

## Current Verified PSA Run

The verified successful run is:

```text
run_id = psa_20261002_030530_d0fa0433
```

Final metrics:

```text
files_discovered = 12
files_ingested   = 12
files_skipped    = 0
files_failed     = 0
tables_created   = 32
status           = SUCCESS
```

Final notes:

```text
PSA ingestion completed successfully: 12 physical files processed,
32 Bronze outputs ingested, 0 failed, 0 unresolved, 0 errors.
```

An earlier stale `STARTED` run was reconciled only after its manifest showed a complete successful load.

Stale-run reconciliation must remain conservative and must not automatically convert every historical `STARTED` row to `SUCCESS`.

---

## Timestamp Convention

Control-table timestamps are stored in UTC.

Example:

```text
2026-10-02T03:08:15.101+00:00
```

For Philippine display time, use `Asia/Manila`.

Example SQL:

```sql
SELECT
    run_id,
    from_utc_timestamp(started_at, 'Asia/Manila') AS started_at_ph,
    from_utc_timestamp(completed_at, 'Asia/Manila') AS completed_at_ph,
    status
FROM `tuklas_dev`.`01_ingestion`.`ingestion_run_log`;
```

Recommended convention:

```text
Store timestamps in UTC.
Convert to Asia/Manila for reporting and display.
```

---

## Validation Queries

### Run History

```sql
SELECT *
FROM `tuklas_dev`.`01_ingestion`.`ingestion_run_log`
ORDER BY started_at DESC;
```

### Manifest for One Run

```sql
SELECT *
FROM `tuklas_dev`.`01_ingestion`.`file_manifest`
WHERE run_id = '<run_id>'
ORDER BY source_dataset_id, source_sheet, destination_table;
```

### Errors for One Run

```sql
SELECT *
FROM `tuklas_dev`.`01_ingestion`.`ingestion_errors`
WHERE run_id = '<run_id>';
```

### Physical File Count

```sql
SELECT COUNT(DISTINCT source_file) AS physical_files
FROM `tuklas_dev`.`01_ingestion`.`file_manifest`
WHERE run_id = '<run_id>';
```

### Logical Bronze Output Count

```sql
SELECT COUNT(DISTINCT destination_table) AS logical_outputs
FROM `tuklas_dev`.`01_ingestion`.`file_manifest`
WHERE run_id = '<run_id>'
  AND status = 'INGESTED';
```

For the current PSA implementation:

```text
physical_files  = 12
logical_outputs = 32
```

---

## PSGC Next Step

Known PSGC source inventory entries include:

```text
DS_33 — PSGC 2Q 2026 National and Provincial Summary
DS_34 — PSGC 2Q 2026 Publication Datafile
```

The PSGC notebook should:

- use the existing Google Drive connection;
- preserve PSGC codes and labels;
- avoid joining or normalizing geography in Bronze;
- add the same Bronze metadata columns;
- use the same `run_id`, manifest, error, and finalization pattern;
- calculate run-level metrics using physical-file rollups.

Do not redesign the ingestion framework for PSGC.

---

## DOT Next Step

DOT ingestion should follow the same control pattern, while allowing source-specific extraction logic for PDF or other non-tabular formats.

The control framework must remain the same:

```text
run_id
STARTED run record
source discovery
Bronze write
file_manifest MERGE
ingestion_errors
physical-file rollup
final run status
```

Source-specific extraction logic may differ, but operational tracking should remain consistent.

---

## Bronze and Silver Responsibilities

| Bronze | Silver |
| --- | --- |
| Preserve source records | Standardize fields and data types |
| Record source lineage | Apply approved transformation rules |
| Preserve source labels and notes | Map geography to the approved PSGC reference |
| Preserve missing-value markers | Apply missing-data rules |
| Expose source-quality issues | Resolve approved quality issues |
| Load safely and track outcomes | Produce harmonized analytical records |

Bronze should expose issues for downstream processing rather than silently fixing them.

---

## Definition of Done

A source is Bronze-ready when:

- [ ] the source is identified with a canonical `DS_XX` ID;
- [ ] the source file can be discovered/read through the approved access method;
- [ ] the reading method is documented;
- [ ] source values are preserved appropriately;
- [ ] required Bronze metadata is present;
- [ ] the Bronze destination table is created or refreshed successfully;
- [ ] the logical output is recorded in `file_manifest`;
- [ ] technical failures are recorded in `ingestion_errors`;
- [ ] run-level metrics correctly distinguish physical files from logical outputs;
- [ ] rerunning does not append duplicate active Bronze records;
- [ ] the final run status is accurately recorded;
- [ ] known limitations remain visible for downstream handling.

---

## Current Status

| Component | Status |
| --- | --- |
| `00_ingestion_setup.ipynb` | Complete |
| PSA filename standardization | Complete |
| PSA Bronze table standardization | Complete |
| PSA source-lineage metadata | Complete |
| PSA CSV ingestion | Complete |
| DS_01 workbook ingestion | Complete |
| DS_02 workbook ingestion | Complete |
| DS_08–DS_17 PSY ingestion | Complete |
| DS_18 Excel ingestion | Complete |
| DS_18 PDF binary ingestion | Complete |
| `ingestion_run_log` | Working |
| `file_manifest` | Working |
| `ingestion_errors` | Working |
| Manifest duplicate protection | Working |
| Physical-file metric rollup | Working |
| Stale-run reconciliation | Working |
| Latest PSA run | `SUCCESS` |
| PSGC ingestion | Next |
| DOT ingestion | Planned |
| Bronze validation notebook | Planned |

---

## Summary

The PSA ingestion workflow has been upgraded from a simple source-to-Bronze load into a traceable ingestion process.

The implementation now provides:

```text
consistent naming
source lineage
safe reruns
run history
logical-output manifests
technical error logging
physical-file metrics
Bronze output counts
```

The current PSA implementation processes:

```text
12 physical source files
32 logical Bronze outputs
0 failed files
0 skipped files
0 ingestion errors
```

The same pattern should now be reused for PSGC and DOT ingestion.
