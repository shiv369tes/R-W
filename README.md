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

## Structure

```
src/
  data_ingestion/
    index.js     — ingestAndProfile(source, config), the entry point
    loader.js    — DataLoader: CSV / JSON / Excel / DB -> Table
    profiler.js  — DataProfiler: stats, missing %, outliers, cleaning
  utils/
    table.js     — dependency-free Table (DataFrame-like) + stats helpers
    logging.js   — leveled console logger
tests/
  test_ingestion.js  — smoke test (assertions, no framework needed)
trial/
  sales_trial.csv, run_trial.js, profile_output.json — worked example
config.yaml       — shared config format with the Python version
```

## Usage

```js
const { ingestAndProfile } = require('./src/data_ingestion');

const profile = await ingestAndProfile('sales.csv', { autoClean: true });

console.log(profile.originalShape);        // [rows, cols] before cleaning
console.log(profile.missingSummary);       // { col: nullFraction, ... }
console.log(profile.outlierSummary);       // { col: outlierCount, ... }
console.log(profile.recommendations);      // ["Drop column 'x' (...)", ...]
console.log(profile.cleanedTable.rows);    // cleaned rows (or raw rows if autoClean=false)
console.log(profile.toDict());             // JSON-ready summary, e.g. for an LLM prompt
```

Config keys accept either camelCase (`autoClean`) or the snake_case used in
`config.yaml` (`auto_clean`) — so you can `JSON.parse`/`yaml.load` the same
config file the Python version uses without renaming keys.

### Excel source

```js
// requires: npm install xlsx
const profile = await ingestAndProfile('sales.xlsx');
```

### Database source

```js
const { Client } = require('pg'); // npm install pg

async function pgAdapter(connectionString, { query, table }) {
  const client = new Client({ connectionString });
  await client.connect();
  const sql = query ?? `SELECT * FROM ${table}`;
  const { rows } = await client.query(sql);
  await client.end();
  return rows;
}

const profile = await ingestAndProfile(
  { connectionString: 'postgresql://user:pass@host/db', table: 'sales', adapter: pgAdapter },
  { outlierMethod: 'zscore', zscoreThreshold: 2.5 }
);
```

## Run it

```bash
npm test          # smoke test — verified passing
npm run trial     # regenerates trial/profile_output.json from trial/sales_trial.csv
```

## Differences from the Python version

- **No pandas**: the `Table` class (`src/utils/table.js`) is a minimal
  array-of-row-objects structure with just the stats (`mean`, `stddev`,
  `quantile`, `mode`, etc.) the profiler needs.
- **Date detection is stricter than Python's.** `Date.parse` in JS is
  notoriously lenient (it will happily "parse" strings like `"ORD-00000"`),
  so date-likeness is checked against an explicit format whitelist
  (`ISO 8601`, `YYYY/MM/DD`, `MM/DD/YYYY`, `DD-Mon-YYYY`, `Month DD, YYYY`)
  rather than trusting `Date.parse` alone. If your dates use a format outside
  that whitelist, add a pattern to `DATE_PATTERNS` in `table.js`.
- **DB support is adapter-based** rather than assuming SQLAlchemy, since
  Node has no single dominant universal DB layer — you inject a small
  function instead of a connection-string convention.
- **Converted "datetime" columns become native `Date` objects**, not a
  pandas dtype; `null` marks missing dates instead of `NaT`.

## Config options (unchanged from Python version)

| Key | Default | Meaning |
|---|---|---|
| `missing_threshold` | `0.8` | Drop a column if null fraction exceeds this |
| `impute_threshold` | `0.1` | Impute (median/mode) if null fraction exceeds this but is below `missing_threshold` |
| `outlier_method` | `iqr` | `iqr` or `zscore` |
| `zscore_threshold` | `3` | Z-score cutoff when `outlier_method: zscore` |
| `auto_clean` | `false` | If true, `ingestAndProfile()` also returns a cleaned table |

## Next steps

Same roadmap as the Python version: Insight Generation → Summary Generation →
Presentation Generation, each as its own module following this one's pattern
(entry-point function, config-driven class, plain-object outputs, smoke test).
