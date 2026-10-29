# ScanLake Format

The Parquet format ScanLake accepts. [`format.json`](format.json) lists, for each of the three
files an upload consists of (`scans`, `compositions`, `confidences`), the columns the file must
have and their types.

ScanLake Store checks every upload against the `main` branch of this file and rejects files that
don't match. Edit and push here to change the format; the Store picks it up within about ten
minutes (GitHub's raw CDN and the Store each cache it for five).

## Where to get it

| Source | URL |
| --- | --- |
| This repository | `https://raw.githubusercontent.com/mort13/ScanLake-Format/main/format.json` |
| ScanLake Gateway (no API key needed) | `GET /api/format` |

## Structure

```jsonc
{
  "version": 1,                      // bump when the format changes; shown in Store errors
  "files": {
    "scans": {                       // one of: scans, compositions, confidences
      "description": "…",
      "allowExtraColumns": false,    // default false: columns not listed below are rejected
      "columns": [
        { "name": "mass", "type": "INT32", "description": "…" },
        { "name": "note", "type": "STRING", "required": false },   // may be absent
        { "name": "count", "type": ["INT32", "INT64"] }            // either type is accepted
      ]
    }
  }
}
```

A column is `required` unless it says `"required": false`. A file kind missing from `files` is
not checked. Column order does not matter.

### Types

Columns are named after their Parquet logical type if they have one, otherwise after the
physical type:

| Type | Parquet column |
| --- | --- |
| `STRING` | `BYTE_ARRAY` annotated `UTF8` / `STRING` (what pyarrow, polars, DuckDB and hyparquet write for strings) |
| `BINARY` | `BYTE_ARRAY` without annotation |
| `BOOLEAN`, `FLOAT`, `DOUBLE`, `INT96`, `FIXED_LEN_BYTE_ARRAY` | that physical type without annotation |
| `INT8` … `INT64`, `UINT8` … `UINT64` | integers; plain `INT32` / `INT64` count as `INT32` / `INT64` |
| `DATE`, `TIME`, `TIMESTAMP`, `DECIMAL`, `FLOAT16`, `INTERVAL`, `JSON`, `BSON`, `UUID`, `ENUM` | the matching logical type, any unit or precision |
| `NESTED` | struct, list and map columns |

Nullability is not checked: a column may be `OPTIONAL` or `REQUIRED`.

Note that pandas writes integers as `INT64` by default; cast to `int32` (or list both types) to
match an `INT32` column.

## Errors

A rejected upload returns `422` with the file kind and every mismatch:

```json
{
  "error": "scans: does not match parquet format v1",
  "violations": [
    "column \"mass\" is INT64, expected INT32",
    "missing column \"volume\"",
    "unexpected column \"notes\""
  ]
}
```

If `format.json` cannot be fetched or is malformed, the Store logs an error and keeps using the
last good copy it has; without one it accepts uploads unchecked rather than dropping data. Check
the Store logs (`wrangler tail scanlake-store`) after changing the format.
