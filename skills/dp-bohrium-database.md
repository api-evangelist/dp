---
name: bohrium-database
description: "Operate Bohrium private databases with bohrium-open-sdk. Use when: the user asks to list private/authorized Bohrium databases, inspect tables or schemas, query/filter/export rows, insert/update/delete records, create tables, or alter table columns using db_ak/table_ak IDs. NOT for: public materials databases such as crystal/electrolyte/positive-electrode databases, internal shrimp PostgreSQL operations, general file storage, datasets, or compute jobs."
metadata:
  openclaw:
    primaryEnv: BOHR_ACCESS_KEY
---

# SKILL: Bohrium Private Database

## Overview

Use `bohrium-open-sdk` to operate Bohrium private databases that the user can access. "Private databases" here includes databases created by the user, team-authorized databases, and any database visible to the current AK. It does not include the public materials database catalog such as crystal, electrolyte, or positive-electrode databases. Common identifiers:

- `db_ak`: database access key, for example `669ng`
- `table_ak`: table access key, for example `669ng00`

Prefer the bundled `database_manager.py` for routine operations. Use direct SDK calls only for complex filters, data cleaning, or custom exports.

## Authentication and Install

Read `BOHR_ACCESS_KEY` from the environment. Never print, log, or hardcode it.

```bash
test -n "$BOHR_ACCESS_KEY" || echo "BOHR_ACCESS_KEY is missing"
python3 -m pip install "bohrium-open-sdk>=1.0.5" -q
```

If `BOHR_ACCESS_KEY` is missing, ask the user to configure their AccessKey before calling the API.

## Quick CLI

Run these commands from the skill directory, or replace `database_manager.py` with the actual script path:

```bash
python3 database_manager.py list-dbs
python3 database_manager.py list-tables 669ng          # shows up to 30 fields per table by default
python3 database_manager.py detail 669ng00
```

Query rows:

```bash
python3 database_manager.py query 669ng00 \
  --where '[{"field":"name","op":"LIKE","value":"sample"}]' \
  --order-by createTime --desc --page 1 --page-size 20
```

Insert rows:

```bash
python3 database_manager.py insert 669ng00 \
  --rows '[{"name":"new sample","value_a":1.5,"value_b":2.3}]'
```

Update and delete preview the target rows by default. Add `--yes` after confirming:

```bash
python3 database_manager.py update 669ng00 \
  --where '[{"field":"name","op":"EQ","value":"old value"}]' \
  --set '{"name":"new value","value_a":99}' \
  --yes

python3 database_manager.py delete 669ng00 \
  --where '[{"field":"name","op":"EQ","value":"delete me"}]' \
  --yes
```

Create or alter tables:

```bash
python3 database_manager.py create-table 669ng "new_table" \
  --schema '[{"title":"sample_name","dataType":"str"},{"title":"mass","dataType":"num"}]'

python3 database_manager.py alter-table 669ng00 \
  --schema '[{"title":"sample_name","dataType":"str"},{"title":"mass","dataType":"num"},{"title":"new_column","dataType":"str"}]' \
  --yes
```

`--rows`, `--where`, `--set`, and `--schema` accept either JSON strings or `@/path/to/file.json`.

## Direct SDK Template

```python
import os
from bohrium_open_sdk.db import SQLClient

client = SQLClient("", os.environ["BOHR_ACCESS_KEY"], "https://openapi.dp.tech")
```

### List Databases

```python
data = client.base().Dbs()
for db in data.get("list", []):
    print(db["name"], db["ak"], db.get("createTime", ""))
```

### List Tables and Schema

```python
result = client.db_with_ak("669ng").Tables()
for table in result.get("tables", []):
    fields = ", ".join(f"{f['name']}({f['type']})" for f in table.get("fields", []))
    print(table["name"], table["tableAk"], fields)

detail = client.table_with_ak("669ng00").Detail()
for field in detail.get("fields", []):
    print(field["name"], field["type"])
```

### Query

```python
from bohrium_open_sdk.db import Op, Where

count, rows = (
    client.table_with_ak("669ng00")
    .Where("price", Op.GTE, 10)
    .And(Where("name", Op.LIKE, "sample"))
    .order(key="price", is_asc=False)
    .page(1)
    .page_size(20)
    .Find()
)
print(count, rows[:20])
```

### Full Export

```python
from bohrium_open_sdk.db.func import db_query_full

table = client.db_with_ak("669ng")
table.table_ak = "669ng00"
count, df = db_query_full(table, batch_size=1000)
df.to_csv("bohrium_table_export.csv", index=False)
print(f"exported {count} rows")
```

## Filter Format

`database_manager.py --where` accepts an array of conditions:

| Field | Required | Description |
|-------|----------|-------------|
| `field` | Yes | Column name, case-sensitive |
| `op` | Yes | Operator, see below |
| `value` | Yes | Filter value |
| `join` | No | From the second condition onward: `AND` or `OR`; default `AND` |

Operators:

| Operator | Meaning | Value type |
|----------|---------|------------|
| `EQ` | Equal | string or number |
| `NEQ` | Not equal | string or number |
| `GT` / `GTE` | Greater than / greater or equal | number |
| `LT` / `LTE` | Less than / less or equal | number |
| `BETWEEN` | Closed range | `"1,100"` or `[1, 100]` |
| `LIKE` | String contains | string |
| `IN` / `NIN` | In / not in | array |
| `ELEMENTMATCHOFNUM` | Numeric array element range | `{"min":0,"max":10}` |

## Column Types

When creating or altering tables, schema items use `{"title": "column_name", "dataType": "type"}`.

| Type | Description |
|------|-------------|
| `str` | String |
| `num` | Number |
| `smiles` | SMILES string |
| `img` | Image object, usually uploaded to Tiefblue first |
| `file` | File object, usually uploaded to Tiefblue first |
| `file_arr` | Array |

Altering a table requires the complete new schema. Omitting an existing column usually means deleting it and may lose data.

## Safety Rules

- Preview target rows or target schema before deleting, updating, or altering tables.
- For batch inserts, updates, or deletes affecting more than 100 rows, explain the impact and ask for confirmation first.
- If a query returns more than 20 rows, show only the first 20 rows and the total count unless the user asks for export.
- For wide tables, use `list-tables --max-fields 30` for an overview, then use `detail` for the full schema of one table.
- Prefer filters for wide or large tables; unfiltered scans may time out. Use `export` when the user needs the full table.
- For image and file columns, show short fields such as `name`; do not show full signed URLs.
- Hide metadata fields such as `_id`, `audit_id`, `createTime/create_time`, `updateTime/update_time`, `authors`, `owner_id`, `status`, and internal audit/permission fields unless requested.
- Do not expose complete API responses; they may contain internal IDs, temporary paths, or signed links.

## Error Handling

| Signal | Handling |
|--------|----------|
| `BOHR_ACCESS_KEY` missing | Ask the user to configure AccessKey |
| 401 / 403 / `code: 2000` | Treat as invalid key or missing permission |
| Column `KeyError`, or unexpectedly empty query | Run `detail` to verify exact column names and types |
| Table or database not found | Use `list-dbs` and `list-tables` to confirm `db_ak` / `table_ak` |
| Network timeout | The helper retries once; if it still fails, narrow the filter/page or use `export` |
