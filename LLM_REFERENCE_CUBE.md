# `edfg-cube-api` — LLM Reference

> This document provides everything needed to write Python code using the `edfg_cube_api` package
> **without access to the source code**. It is a Python client for querying a Power BI cube
> (Analysis Services) via the REST API `executeQueries`, returning `polars.DataFrame` results.

## Installation

```bash
uv add edfg-cube-api --path ../edfg-cube-api
# or from Git:
uv add git+https://github.com/EDF-Gestion/edfg-cube-api.git
```

Requires Python >= 3.14. Dependencies: `msal`, `requests`, `polars`, `python-dotenv`, `diskcache`, `loguru`.

---

## Public API

All public symbols are importable directly from `edfg_cube_api`:

```python
from edfg_cube_api import (
    configure,
    execute,
    cube_value,
    cube_table,
    list_parameters,
    list_values,
    list_tables,
    clear_cache,
    get_cache_info,
    PowerBIError,
)
```

---

## 1. `configure()` — Set default workspace/dataset

```python
def configure(
    workspace_id: str | None = None,
    dataset_id: str | None = None,
    *,
    timeout: int | None = None,          # default 60s
    cache_enabled: bool | None = None,   # default False (opt-in)
    cache_dir: str | Path | None = None, # default ~/.edfg-cube-api/query_cache/
    cache_ttl_seconds: int | None = None, # default 604800 (7 days)
    cache_size_limit: int | None = None, # default 500 MB (LRU eviction)
) -> None
```

Call once at the start. After this, all query functions work without passing `workspace_id`/`dataset_id`.

**Resolution order** for workspace/dataset IDs:
1. Explicit arguments passed to each function
2. Global config set via `configure()`
3. Environment variables `POWERBI_WORKSPACE_ID` / `POWERBI_DATASET_ID`

If none found, raises `ValueError`.

**Cache settings** follow the same priority, with env vars `EDFG_CUBE_CACHE_ENABLED`,
`EDFG_CUBE_CACHE_TTL`, `EDFG_CUBE_CACHE_DIR`, `EDFG_CUBE_CACHE_SIZE_LIMIT`.

### Example

```python
from edfg_cube_api import configure

# Option A: explicit
configure(workspace_id="xxxxxxxx-...", dataset_id="yyyyyyyy-...")

# Option B: env vars (via .env file)
from dotenv import load_dotenv
load_dotenv()  # reads POWERBI_WORKSPACE_ID and POWERBI_DATASET_ID
```

---

## 2. `cube_value()` — Get a single scalar (like Excel's CUBEVALUE)

```python
def cube_value(
    measure: str,
    filters: dict[str, FilterValue] | None = None,
    *,
    workspace_id: str | None = None,
    dataset_id: str | None = None,
    timeout: int | None = None,
    cache: bool | None = None,
    **kwargs: FilterValue,
) -> float | str | None
```

Returns a single aggregated value for the given measure with optional filters. Returns `None` if the result is empty.

### Examples

```python
# Simple scalar at a date
vb = cube_value("VB", date="2025-01-15")

# With a portfolio filter
vb = cube_value("VB", date="2025-01-15", niv3="Trésorerie")

# Date range (tuple = inclusive range)
vb = cube_value("VB", date=("2025-01-01", "2025-03-31"))

# Multiple values (list = IN)
vb = cube_value("VB", niv3=["Actions", "Obligations"], date="2025-01-15")

# Combine multiple filter types
vb = cube_value("VB", date=("2025-01-01", "2025-03-31"), niv3="Trésorerie", devise="EUR")

# Raw DAX column filter (for columns not in the named parameter registry)
vb = cube_value("VB", date="2025-01-15", filters={"F_VB[Poche]": "TAUX"})
```

**DAX generated** (under the hood):
- No filters: `EVALUATE ROW("VB", [VB])`
- With filters: `EVALUATE ROW("VB", CALCULATE([VB], <filter_clauses>))`

---

## 3. `cube_table()` — Breakdown by axes (like a pivot table)

```python
def cube_table(
    measure: str | list[str],
    *,
    by: list[str],
    filters: dict[str, FilterValue] | None = None,
    workspace_id: str | None = None,
    dataset_id: str | None = None,
    timeout: int | None = None,
    cache: bool | None = None,
    **kwargs: FilterValue,
) -> pl.DataFrame
```

Returns a `polars.DataFrame` with one row per combination of the `by` axes, and one column per measure. Results are **sorted by the `by` columns**.

- `measure`: one measure name (`str`) or several (`list[str]`)
- `by`: list of named parameters (e.g. `"date"`, `"niv3"`) or raw DAX columns (e.g. `"D_Instrument[ISIN]"`)

### Examples

```python
# VB over a date range
df = cube_table("VB", by=["date"], date=("2025-01-01", "2025-01-15"))
# Result columns: ["Date", "VB"]

# Breakdown by two axes
df = cube_table("VB", by=["niv3", "devise"], date="2025-01-15")
# Result columns: ["Niv3", "DEVISE", "VB"]

# Multiple measures
df = cube_table(["VB", "Performance YTD %"], by=["niv5"], date="2025-01-15", niv3="Actions")
# Result columns: ["Niv5", "VB", "Performance YTD %"]

# Using raw DAX column in `by`
df = cube_table("VB", by=["D_Instrument[ISIN]"], date="2025-01-15")

# Combining kwargs and raw filters
df = cube_table(
    "VB",
    by=["date"],
    date=("2025-01-01", "2025-03-31"),
    filters={"F_VB[Poche]": ["TAUX", "ACTIFS SANS RISQUE"]},
)
```

**DAX generated**: `EVALUATE SUMMARIZECOLUMNS(<by_cols>, <FILTER(ALL(...))>, "Measure", [Measure])`

---

## 4. `execute()` — Raw DAX query

```python
def execute(
    dax: str,
    *,
    workspace_id: str | None = None,
    dataset_id: str | None = None,
    timeout: int | None = None,
    cache: bool | None = None,
) -> pl.DataFrame
```

Sends an arbitrary DAX query. Column names are cleaned: `D_Calendrier[Date]` becomes `Date`, `[VB]` becomes `VB`.

### Examples

```python
# Simple scalar
df = execute('EVALUATE ROW("x", [VB])')

# Top 10 by VB
df = execute("""
    EVALUATE
    TOPN(
        10,
        SUMMARIZECOLUMNS(
            D_Portefeuille[Niv5],
            FILTER(ALL(D_Calendrier[Date]), D_Calendrier[Date] = DATEVALUE("2025-01-15")),
            "VB", [VB]
        ),
        [VB], DESC
    )
""")
```

---

## 5. `list_values()` — Distinct values of a parameter (like Excel's CUBESET)

```python
def list_values(
    parameter: str,
    filters: dict[str, FilterValue] | None = None,
    *,
    workspace_id: str | None = None,
    dataset_id: str | None = None,
    timeout: int | None = None,
    cache: bool | None = None,
    **kwargs: FilterValue,
) -> list[Any]
```

Returns a Python list of distinct values. The `parameter` can be a named parameter or a raw DAX column.

### Examples

```python
list_values("devise")
# → ['AUD', 'CHF', 'EUR', 'GBP', 'JPY', 'NOK', 'SEK', 'USD']

list_values("niv3")
# → ['Actions', 'Actifs de rendement', 'Obligations', 'Trésorerie', ...]

# Filtered: sub-categories within a category
list_values("niv5", niv3="Actions")
# → ['Actions Amérique du Nord', 'Actions Europe', ...]

# Also accepts raw DAX column
list_values("D_Instrument[Type]")
```

---

## 6. `list_parameters()` — Available filter parameters (offline)

```python
def list_parameters() -> pl.DataFrame
```

Returns a DataFrame with columns: `name`, `dax_column`, `description`, `table`. **No connection required.**

---

## 7. `list_tables()` — Model introspection

```python
def list_tables(
    *,
    workspace_id: str | None = None,
    dataset_id: str | None = None,
    timeout: int | None = None,
    cache: bool | None = None,
) -> pl.DataFrame
```

Runs `EVALUATE COLUMNSTATISTICS()` to list all tables and columns in the Power BI model.

---

## Filter System

### Named Parameters

These are the kwargs accepted by `cube_value()`, `cube_table()`, and `list_values()`. They are automatically mapped to qualified DAX columns.

| Python kwarg | DAX Column | Description | Table |
|---|---|---|---|
| `date` | `D_Calendrier[Date]` | Valuation date (YYYY-MM-DD) | D_Calendrier |
| `annee` | `D_Calendrier[Année]` | Year (integer) | D_Calendrier |
| `trimestre` | `D_Calendrier[Trimestre]` | Quarter (1-4) | D_Calendrier |
| `mois` | `D_Calendrier[Mois_nb]` | Month number (1-12) | D_Calendrier |
| `periode` | `D_Calendrier[Periode]` | Period as YYYY_MM | D_Calendrier |
| `niv1` | `D_Portefeuille[Niv1]` | Portfolio level 1 | D_Portefeuille |
| `niv2` | `D_Portefeuille[Niv2]` | Portfolio level 2 | D_Portefeuille |
| `niv3` | `D_Portefeuille[Niv3]` | Portfolio level 3 (e.g. Actifs de rendement, Trésorerie) | D_Portefeuille |
| `niv4` | `D_Portefeuille[Niv4]` | Portfolio level 4 | D_Portefeuille |
| `niv5` | `D_Portefeuille[Niv5]` | Portfolio level 5 (finest granularity) | D_Portefeuille |
| `code_ped` | `D_Portefeuille[Code PED]` | PED code | D_Portefeuille |
| `instrument` | `D_Instrument[Instrument]` | Instrument name | D_Instrument |
| `isin` | `D_Instrument[ISIN]` | ISIN code | D_Instrument |
| `code_bloomberg` | `D_Instrument[Code Bloomberg]` | Bloomberg ticker | D_Instrument |
| `type_instrument` | `D_Instrument[Type]` | Instrument type (Coupon, ZCoupon...) | D_Instrument |
| `gestion` | `D_Instrument[Gestion]` | Management style (Active, Indicielle) | D_Instrument |
| `classifie` | `D_Instrument[Classifié]` | Classification (Absolute Return, TAUX...) | D_Instrument |
| `societe_gestion` | `D_Instrument[Société de gestion]` | Fund management company | D_Instrument |
| `devise` | `D_Devise[DEVISE]` | Currency code (EUR, USD, GBP...) | D_Devise |
| `poche` | `F_VB[Poche]` | Valuation pocket (ACTIFS SANS RISQUE, TAUX...) | F_VB |

### Filter Value Types

| Python type | Meaning | DAX generated | Example |
|---|---|---|---|
| `str` | Exact equality | `Col = "val"` | `date="2025-01-15"` |
| `list[str]` | IN (membership) | `Col IN {"a", "b"}` | `niv3=["Actions", "Obligations"]` |
| `tuple[str, str]` | Inclusive range | `Col >= min AND Col <= max` | `date=("2025-01-01", "2025-03-31")` |

**Automatic behaviors:**
- Dates (YYYY-MM-DD format) are wrapped in `DATEVALUE("...")`
- Numeric strings are injected without quotes
- The `filters` dict parameter accepts **raw DAX column names** (e.g. `{"F_VB[Poche]": "TAUX"}`) and takes priority over kwargs in case of conflict

---

## Error Handling

| Situation | Exception | Key attributes |
|---|---|---|
| HTTP error from Power BI (400, 401, 403...) | `PowerBIError` | `.status_code`, `.pbi_code`, `.pbi_message`, `.dax` |
| DAX error in a 200 response | `ValueError` | Error code + message |
| Empty result | No exception | `cube_value()` → `None`; `cube_table()` → empty DataFrame |
| Missing workspace/dataset ID | `ValueError` | Lists resolution options |
| Unknown named parameter | `KeyError` | Lists available parameters |
| Invalid filter value type | `TypeError` | Expected str, list, or tuple |
| Empty list filter | `ValueError` | — |
| Tuple with != 2 elements | `TypeError` | Expected (min, max) |

### PowerBIError example

```python
from edfg_cube_api import PowerBIError, cube_value

try:
    cube_value("NonExistentMeasure", date="2025-01-15")
except PowerBIError as e:
    print(e.status_code)   # 400
    print(e.pbi_code)      # "DatasetExecuteQueriesError"
    print(e.pbi_message)   # "The value for 'NonExistentMeasure' cannot be determined..."
    print(e.dax)           # the full DAX query for debugging
```

---

## Caching

All query functions (`execute`, `cube_value`, `cube_table`, `list_values`,
`list_tables`) can be cached on disk. The cache is **opt-in** (default
off) and is backed by `diskcache` (SQLite under the hood) with LRU
eviction and TTL.

### Enable

```python
from edfg_cube_api import configure
configure(cache_enabled=True)   # use defaults (7-day TTL, 500 MB, ~/.edfg-cube-api/query_cache/)
```

Or via env vars:

```env
EDFG_CUBE_CACHE_ENABLED=true
EDFG_CUBE_CACHE_TTL=604800
EDFG_CUBE_CACHE_DIR=/path/to/cache
EDFG_CUBE_CACHE_SIZE_LIMIT=524288000
```

### Per-call override

Every query function accepts `cache: bool | None = None`:

| Value   | Behavior                                                    |
|---------|-------------------------------------------------------------|
| `None`  | Use the global setting (`configure(cache_enabled=...)`)     |
| `True`  | Use the cache even if globally disabled                     |
| `False` | Skip the cache, always hit the network, don't store result  |

### Cache key

`SHA-256(workspace_id | dataset_id | dax)`. Two calls producing the
exact same DAX query on the same dataset share one entry. Changing a
filter, measure, or axis yields a new key.

### Helpers

```python
from edfg_cube_api import clear_cache, get_cache_info

clear_cache()        # wipe all entries (e.g. after a cube refresh)
get_cache_info()
# {
#   "enabled": True,
#   "directory": "/home/user/.edfg-cube-api/query_cache",
#   "ttl_seconds": 604800,
#   "size_limit_bytes": 524288000,
#   "size_bytes": 12345,
#   "count": 3,
# }
```

### TTL

Default **7 days**, matching the cube's weekly refresh. Override with
`cache_ttl_seconds` if your cube refresh cadence differs.

---

## Authentication

Authentication is **automatic and transparent**:
- Uses Azure AD device code flow via `msal`
- On first use, prints a URL + code to the terminal for the user to authenticate in a browser
- Token cache is persisted at `~/.edfg-cube-api/token_cache.json`
- Refresh token (~90 days) is reused silently on subsequent calls

No code is needed to handle auth — it happens on the first query call.

---

## Complete Working Example

```python
import os
from dotenv import load_dotenv
from edfg_cube_api import configure, cube_value, cube_table, list_values, PowerBIError

load_dotenv()
configure(
    workspace_id=os.environ["POWERBI_WORKSPACE_ID"],
    dataset_id=os.environ["POWERBI_DATASET_ID"],
)

# 1. Explore available values
devises = list_values("devise")
poches = list_values("niv3")
sub_poches = list_values("niv5", niv3="Actions")

# 2. Get a single value
vb_total = cube_value("VB", date="2025-01-15")
vb_tresorerie = cube_value("VB", date="2025-01-15", niv3="Trésorerie")

# 3. Get a breakdown table
df_by_date = cube_table("VB", by=["date"], date=("2025-01-01", "2025-03-31"))

df_by_poche = cube_table(
    ["VB", "Performance YTD %"],
    by=["niv3", "devise"],
    date="2025-01-15",
)

# 4. Handle errors
try:
    cube_value("BadMeasure", date="2025-01-15")
except PowerBIError as e:
    print(f"Error: {e.pbi_message}")
```

---

## Known Limitations

- **100,000 rows max** per API result (Power BI limit)
- **One DAX query per API call**
- **Only DAX** is supported (no MDX, no DMV)
- **Measures must exist** in the Power BI model (not raw fact table columns)
- All return DataFrames are **`polars.DataFrame`**, not pandas
