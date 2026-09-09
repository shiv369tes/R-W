# AI-Powered Reporting Workflow — Data Ingestion & Profiling (JavaScript)

A JavaScript/Node.js port of the Python `data_ingestion` module. Same behavior,
same config keys, same `to_dict()`-style JSON output — so it can drop into a
Node backend or be swapped 1:1 with the Python version.

## Zero required dependencies

CSV and JSON parsing are implemented from scratch (no `papaparse`, no `pandas`
equivalent needed). Excel and databases are **optional, lazy-loaded**:

- Excel (`.xlsx`/`.xls`) needs `npm install xlsx` (SheetJS) — only required if
  you actually load an Excel file.
- Databases need you to supply an `adapter` function (see below) built on
  whatever driver you use (`pg`, `mysql2`, etc.) — this module doesn't assume
  a specific database.

