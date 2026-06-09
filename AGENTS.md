# AGENTS.md

This file provides guidance to AI agents when working with code in this repository.

## What This Repository Is

This is the **MOSIP Master Data Repository** — a data-only repo containing sample/sandbox master data for the [MOSIP identity platform](https://docs.mosip.io/1.2.0/id-lifecycle-management). It has no application code, no build system, and no test suite. Changes here feed directly into MOSIP database initialization and upgrades.

## Data Layout

```
mosip_master/xlsx/       ← Canonical source: 37 Excel files (edit these)
mosip_master/csv/        ← Primary CSV source used by the masterdata-loader
mosip_master_csv/csv/    ← Auto-generated from xlsx via CI (do not edit manually)
mosip_master/data_upgrade/ ← Version-to-version delta migration scripts
```

**Key rule:** Only edit files in `mosip_master/xlsx/` or `mosip_master/csv/`. The `mosip_master_csv/csv/` directory is regenerated automatically by the `xlsx-to-csv` GitHub Actions workflow on every PR — manual edits there will be overwritten.

## Workflows

### xlsx-to-csv (`.github/workflows/xlsx-to-csv.yml`)
Triggers on PR open/sync or manual `workflow_dispatch`. Converts every `.xlsx` in `mosip_master/xlsx/` to CSV using `xlsx2csv` and commits the result into `mosip_master_csv/csv/` on the PR branch. Requires the `ACTION_PAT` secret.

To trigger manually: GitHub → Actions → "xlsx-to-csv" → Run workflow.

### push-trigger (`.github/workflows/push-trigger.yml`)
Validates the full data load pipeline end-to-end:
1. Spins up PostgreSQL 16 in Docker
2. Runs `mosipdev/postgres-init:develop` to create the `mosip_master` schema
3. Runs `mosipdev/masterdata-loader:develop` to load CSV data from this repo
4. Checks for loader errors

This is the closest thing to a "test" for this repo. A passing run confirms the data is loadable.

## Data Entities

The master data covers:

| Domain | Key files |
|---|---|
| Identity | `identity_schema`, `ui_spec`, `dynamic_field` |
| Templates | `template`, `template_type` |
| Devices | `device_master`, `device_spec`, `device_type` |
| Machines | `machine_master`, `machine_spec`, `machine_type` |
| Locations | `location`, `loc_hierarchy_list`, `zone`, `zone_user` |
| Registration | `registration_center`, `reg_center_type`, `reg_working_nonworking` |
| Documents | `valid_document`, `doc_category`, `doc_type`, `applicant_valid_document` |
| Lookup | `language`, `title`, `reason_list`, `blocklisted_words` |

**`identity_schema.csv`** is the most complex file — it embeds full JSON Schema Draft-07 definitions inside CSV cells. Each row is one schema version per language (e.g. `eng, 1001, 0.1, ...`). Schema ID 1001 = standard identity; 1002 = Mosip Identity for handle.

## Data Upgrade Scripts

`mosip_master/data_upgrade/` contains **delta** migrations (not full datasets) between MOSIP versions:

- `1.1.5.5_to_1.2.0.1/` — Dynamic field format changed (array → object per language); UI spec split into separate table; new machine/template types
- `1.2.0.1_to_1.3.0/` — Template additions and updates only

**To run an upgrade:**
```bash
cd mosip_master/data_upgrade/<version>/
cp upgrade.properties upgrade.local.properties
# Fill in DB_SERVERIP, SU_USER_PWD, UPGRADE_DOMAIN_NAME, admin creds, language codes
bash upgrade.sh upgrade.local.properties
```

The Python helper scripts (`data-uploader.py`, `migration-ui_spec.py`, `migration-dynamicfield.py`) authenticate against the MOSIP Admin API and/or connect directly to PostgreSQL. They are invoked by `upgrade.sh` via `upgrade_commands.txt`.

## Masterdata Reference

Full field definitions and valid values: https://docs.mosip.io/1.2.0/id-lifecycle-management/support-systems/administration/masterdata-guide
