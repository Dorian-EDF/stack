# `edfg-bloomany-api` — LLM Reference

> This document provides everything needed to write Python code using the `edfg_bloomany_api` package
> **without access to the source code**. It is a Python client for querying the Bloomberg Desktop API
> (`blpapi`) covering BDP, BDH, BDS, and BQL request types, returning `polars.DataFrame` results.

## Installation

```bash
# From Git with uv
uv add git+https://github.com/EDF-Gestion/edfg-bloomany-api.git

# From local path
uv add edfg-bloomany-api --path ../edfg-bloomany-api
```

Requires Python >= 3.14. Dependencies: `blpapi`, `polars`, `loguru`, `python-dotenv`, `diskcache`.

**Bloomberg SDK prerequisite:** the C++ Bloomberg `blpapi` shared libraries must be installed
and available in the system PATH. Download them from [developer.bloomberg.com](https://developer.bloomberg.com).
The `blpapi` Python package is pulled from a custom PyPI registry
(`https://blpapi.bloomberg.com/repository/releases/python/simple`).

---

## Public API

All public symbols are importable directly from `edfg_bloomany_api`:

```python
from edfg_bloomany_api import (
    configure,
    clear_cache,
    get_bql,
    get_bulk,
    get_cache_info,
    get_historical,
    get_reference,
)
```

| Function | Bloomberg equivalent | Service | Purpose |
|---|---|---|---|
| `get_reference()` | `=BDP(...)` | `//blp/refdata` | Static / point-in-time data |
| `get_historical()` | `=BDH(...)` | `//blp/refdata` | Time series data |
| `get_bulk()` | `=BDS(...)` | `//blp/refdata` | Tabular bulk data |
| `get_bql()` | `=BQL.Query(...)` | `//blp/bqlsvc` | Bloomberg Query Language |
| `configure()` | — | — | Set global defaults (cache settings) |
| `clear_cache()` | — | — | Wipe all cached responses |
| `get_cache_info()` | — | — | Inspect cache state |

---

## 1. `get_reference()` — Static data (BDP)

```python
def get_reference(
    securities: list[str],
    fields: list[str],
    *,
    overrides: dict[str, str] | None = None,
    host: str = "localhost",
    port: int = 8194,
) -> pl.DataFrame
```

Returns a `polars.DataFrame` with one row per security and one column per field,
plus a `security` column.

### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `securities` | `list[str]` | **required** | Bloomberg tickers (e.g. `["SX5E Index", "FP FP Equity"]`) |
| `fields` | `list[str]` | **required** | Field mnemonics (e.g. `["NAME", "CRNCY", "PX_LAST"]`) |
| `overrides` | `dict[str, str] \| None` | `None` | Optional Bloomberg overrides as key-value string pairs |
| `host` | `str` | `"localhost"` | Bloomberg terminal host |
| `port` | `int` | `8194` | Bloomberg terminal port |

### Behavior

- Invalid securities are **silently skipped** (no error raised).
- If a field returns bulk (array) data, it appears as the string `"<BULK:N rows>"` — use `get_bulk()` instead.
- Returns an **empty typed DataFrame** if no data is found.

### Examples

```python
from edfg_bloomany_api import get_reference

# Basic reference data
df = get_reference(
    securities=["SX5E Index", "CAC Index"],
    fields=["NAME", "CRNCY", "PX_LAST"],
)
# ┌──────────────┬──────────────────────┬───────┬─────────┐
# │ security     │ NAME                 │ CRNCY │ PX_LAST │
# │ ---          │ ---                  │ ---   │ ---     │
# │ str          │ str                  │ str   │ f64     │
# ├──────────────┼──────────────────────┼───────┼─────────┤
# │ SX5E Index   │ Euro Stoxx 50 Pr     │ EUR   │ 5865.46 │
# │ CAC Index    │ CAC 40               │ EUR   │ 8203.98 │
# └──────────────┴──────────────────────┴───────┴─────────┘

# With overrides (e.g. currency conversion)
df = get_reference(
    securities=["FP FP Equity"],
    fields=["PX_LAST", "CRNCY_ADJ_PX_LAST"],
    overrides={"EQY_FUND_CRNCY": "USD"},
)
```

### Common fields

| Field | Description |
|---|---|
| `NAME` | Full security name |
| `CRNCY` | Currency |
| `PX_LAST` | Last price |
| `COUNTRY` | Country |
| `GICS_SECTOR_NAME` | GICS sector |
| `CUR_MKT_CAP` | Market capitalization |
| `BEST_PE_RATIO` | Consensus P/E ratio |
| `COUNT_INDEX_MEMBERS` | Number of index constituents |
| `ID_ISIN` | ISIN code |

---

## 2. `get_historical()` — Time series (BDH)

```python
from datetime import date
from typing import Literal

Frequency = Literal["DAILY", "WEEKLY", "MONTHLY", "QUARTERLY", "SEMI_ANNUALLY", "YEARLY"]

def get_historical(
    securities: list[str],
    fields: list[str],
    start: date | str,
    end: date | str | None = None,
    frequency: Frequency = "DAILY",
    *,
    host: str = "localhost",
    port: int = 8194,
) -> pl.DataFrame
```

Returns a `polars.DataFrame` with columns `security` (str), `date` (Date), and one column
per field (Float64). Results are **sorted by `security`, then `date`**.

### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `securities` | `list[str]` | **required** | Bloomberg tickers |
| `fields` | `list[str]` | **required** | Field mnemonics |
| `start` | `date \| str` | **required** | Start date (`datetime.date` or ISO string `"YYYY-MM-DD"`) |
| `end` | `date \| str \| None` | `None` (today) | End date (same format as `start`) |
| `frequency` | `Frequency` | `"DAILY"` | Data periodicity |
| `host` | `str` | `"localhost"` | Bloomberg terminal host |
| `port` | `int` | `8194` | Bloomberg terminal port |

### Behavior

- If `end` is `None`, defaults to today.
- Accepts both `datetime.date` objects and ISO format strings.
- Invalid securities are silently skipped.
- Returns an empty typed DataFrame if no data.

### Examples

```python
from edfg_bloomany_api import get_historical
import datetime as dt

# Monthly prices for multiple indices
df = get_historical(
    securities=["SX5E Index", "CAC Index", "SPX Index"],
    fields=["PX_LAST"],
    start="2024-06-01",
    frequency="MONTHLY",
)
# ┌──────────────┬────────────┬─────────┐
# │ security     │ date       │ PX_LAST │
# │ ---          │ ---        │ ---     │
# │ str          │ date       │ f64     │
# ├──────────────┼────────────┼─────────┤
# │ CAC Index    │ 2024-06-28 │ 7724.32 │
# │ CAC Index    │ 2024-07-31 │ 7431.04 │
# │ SX5E Index   │ 2024-06-28 │ 4894.02 │
# │ …            │ …          │ …       │
# └──────────────┴────────────┴─────────┘

# Daily data with date objects and multiple fields
df = get_historical(
    securities=["SX5E Index"],
    fields=["PX_LAST", "CHG_PCT_1D", "VOLUME"],
    start=dt.date(2025, 1, 1),
    end=dt.date(2025, 3, 31),
    frequency="DAILY",
)
```

### Common fields

| Field | Description |
|---|---|
| `PX_LAST` | Close price |
| `PX_OPEN` | Open price |
| `PX_HIGH` | Daily high |
| `PX_LOW` | Daily low |
| `CHG_PCT_1D` | 1-day % change |
| `VOLUME` | Trading volume |
| `TOT_RETURN_INDEX_GROSS_DVDS` | Total return index |
| `EQY_DPS` | Dividend per share |

---

## 3. `get_bulk()` — Tabular data (BDS)

```python
def get_bulk(
    securities: list[str],
    field: str,
    *,
    overrides: dict[str, str] | None = None,
    host: str = "localhost",
    port: int = 8194,
) -> pl.DataFrame
```

Returns a `polars.DataFrame` with a `security` column plus the columns defined by the bulk
field's structure (variable names and types from Bloomberg).

### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `securities` | `list[str]` | **required** | Bloomberg tickers |
| `field` | `str` | **required** | **Single** bulk field mnemonic |
| `overrides` | `dict[str, str] \| None` | `None` | Optional Bloomberg overrides |
| `host` | `str` | `"localhost"` | Bloomberg terminal host |
| `port` | `int` | `8194` | Bloomberg terminal port |

### Behavior

- **Only one field per call** — bulk fields return arrays with differing structures, so each
  call handles a single field.
- If the field is not a bulk field (doesn't return a list), returns an empty DataFrame.
- Invalid securities are silently skipped.
- Each row of bulk data becomes one row in the DataFrame.

### Examples

```python
from edfg_bloomany_api import get_bulk

# Index composition with weights
df = get_bulk(
    securities=["SX5E Index"],
    field="INDX_MWEIGHT",
)
# ┌──────────────┬──────────────────────────────────────────┬───────────────────┐
# │ security     │ Member Ticker and Exchange Code          │ Percentage Weight │
# │ ---          │ ---                                      │ ---               │
# │ str          │ str                                      │ f64               │
# ├──────────────┼──────────────────────────────────────────┼───────────────────┤
# │ SX5E Index   │ SAP GY                                   │ 8.5               │
# │ SX5E Index   │ ASML NA                                  │ 7.2               │
# │ …            │ …                                        │ …                 │
# └──────────────┴──────────────────────────────────────────┴───────────────────┘

# Dividend history with overrides
df = get_bulk(
    securities=["FP FP Equity"],
    field="DVD_HIST_ALL",
    overrides={"DVD_START_DT": "20200101", "DVD_CRNCY": "EUR"},
)

# Yield curve tenors
df = get_bulk(
    securities=["YCGT0025 Index"],
    field="CURVE_TENOR_RATES",
)
```

### Common bulk fields

| Field | Description | Useful overrides |
|---|---|---|
| `INDX_MWEIGHT` | Index composition & weights | — |
| `INDX_MEMBERS` | Index member list | — |
| `DVD_HIST_ALL` | Dividend history | `DVD_START_DT`, `DVD_END_DT`, `DVD_CRNCY` |
| `CURVE_TENOR_RATES` | Yield curve by tenor | `CURVE_DATE` |
| `ERN_ANN_DT_AND_PER` | Earnings announcement dates | — |
| `CORP_RATING` | Credit ratings | — |

---

## 4. `get_bql()` — Bloomberg Query Language

```python
def get_bql(
    expression: str,
    *,
    host: str = "localhost",
    port: int = 8194,
) -> BqlResult
```

Executes a BQL expression and returns a `BqlResult` container.

### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `expression` | `str` | **required** | Complete BQL expression |
| `host` | `str` | `"localhost"` | Bloomberg terminal host |
| `port` | `int` | `8194` | Bloomberg terminal port |

### `BqlResult` container

`BqlResult` holds one `polars.DataFrame` per data-item in the `get(...)` clause.

```python
results = get_bql("get(name, px_last) for(['SX5E Index'])")

results[0]          # DataFrame for 'name'
results[1]          # DataFrame for 'px_last'
len(results)        # 2
results.names       # ['name', 'px_last']

for df in results:
    print(df)       # iterate over DataFrames

# Join all DataFrames on common columns (typically 'ID')
df = results.combine()
```

Each DataFrame contains:
- `ID` (String) — security identifier
- A column named after the data-item (typed automatically)
- Optional secondary columns: `DATE` (Date), `CURRENCY` (String), `REVISION_DATE`, `PERIOD_END_DATE`, `ORIG_IDS`, etc.

### BQL expression structure

```
[let(#variable = expression;)]    -- optional variable definitions
get(field1, field2, #variable)    -- required: data items to fetch
for(universe)                     -- required: securities / universe
[with(fill=PREV)]                 -- optional: fill strategy
```

### Examples

```python
from edfg_bloomany_api import get_bql

# --- Single field, multiple securities ---
results = get_bql("get(px_last) for(['SX5E Index', 'CAC Index', 'SPX Index'])")
df = results[0]
# ┌──────────────┬─────────┬────────────┬──────────┐
# │ ID           │ px_last │ DATE       │ CURRENCY │
# │ ---          │ ---     │ ---        │ ---      │
# │ str          │ f64     │ date       │ str      │
# ├──────────────┼─────────┼────────────┼──────────┤
# │ SX5E Index   │ 5865.46 │ 2026-04-11 │ EUR      │
# │ CAC Index    │ 8203.98 │ 2026-04-11 │ EUR      │
# │ SPX Index    │ 5363.36 │ 2026-04-11 │ USD      │
# └──────────────┴─────────┴────────────┴──────────┘

# --- Multiple fields, combine into a single DataFrame ---
results = get_bql("get(name, px_last, crncy) for(['SX5E Index'])")
df = results.combine()

# --- Screening: index members with price > 200 ---
results = get_bql("""
    get(name(), px_last)
    for(filter(members('SX5E Index'), px_last() > 200))
""")
df = results.combine()

# --- Variables with technical indicators ---
results = get_bql("""
    let(#ema20 = emavg(period=20);
        #ema200 = emavg(period=200);
        #rsi = rsi(close=px_last());)
    get(name(), #ema20, #ema200, #rsi)
    for(filter(members('CAC Index'),
               and(#ema20 > #ema200, #rsi > 53)))
    with(fill=PREV)
""")
df = results.combine()

# --- Bonds of an issuer ---
results = get_bql("""
    let(#oas = spread(st=oas);
        #rank = normalized_payment_rank();)
    get(name(), #rank, #oas)
    for(bonds('FP FP Equity'))
""")
df = results.combine()

# --- Historical data via BQL ---
results = get_bql("""
    get(px_last)
    for(['SX5E Index'])
    with(dates=range(-1Y, 0D), fill=PREV)
""")
df = results[0]  # DataFrame with ID, px_last, DATE columns

# --- Screen results ---
results = get_bql("""
    get(name(), px_last, cur_mkt_cap)
    for(screenresults(type=SRCH, screen_name='@MyScreen'))
""")
df = results.combine()
```

### Common BQL functions

| Function | Description |
|---|---|
| `px_last()` | Last price |
| `name()` | Security name |
| `members('XXX Index')` | Index constituents |
| `filter(universe, condition)` | Filter a universe |
| `bonds('XXX Equity')` | Bonds of an issuer |
| `avg(group(field, groupby))` | Group aggregate |
| `emavg(period=N)` | Exponential moving average |
| `rsi(close=px_last())` | Relative Strength Index |
| `spread(st=oas)` | OAS spread |
| `range(-1Y, 0D)` | Relative date range |
| `screenresults(type=SRCH, screen_name='@XXX')` | Saved screen results |
| `and(cond1, cond2)` | Logical AND |
| `or(cond1, cond2)` | Logical OR |

---

## Connection & Authentication

Authentication is handled automatically by the Bloomberg terminal:

- The library connects to a **locally running Bloomberg terminal** via `blpapi`.
- Default connection: `localhost:8194`.
- All four functions accept `host` and `port` parameters to override the default.
- There is **no token, no login, no env variable** — the Bloomberg terminal session is the authentication.

### Connection example

```python
# Default: local terminal
df = get_reference(securities=["SX5E Index"], fields=["PX_LAST"])

# Remote terminal
df = get_reference(
    securities=["SX5E Index"],
    fields=["PX_LAST"],
    host="bloomberg-server.internal",
    port=8194,
)
```

---

## Error Handling

| Situation | Exception | Description |
|---|---|---|
| Terminal not open or unreachable | `BloombergConnectionError` | Launch Bloomberg, check network |
| Request timeout (>32s) | `TimeoutError` | Simplify the query or retry |
| Bloomberg API error (invalid field, etc.) | `RuntimeError` | Message prefixed with `"Bloomberg responseError:"` |
| BQL syntax error | `RuntimeError` | Message prefixed with `"Erreur BQL:"` |
| No common columns in `BqlResult.combine()` | `ValueError` | Ensure data-items share a join key |
| No DataFrames in `BqlResult.combine()` | `ValueError` | Check that the BQL query returned results |

### Silent failures

- **Invalid securities** are silently skipped — only valid securities appear in results.
- **Empty results** return a typed empty DataFrame (never `None` or an exception).

### `BloombergConnectionError`

```python
from edfg_bloomany_api import get_reference
from edfg_bloomany_api.session import BloombergConnectionError

try:
    df = get_reference(securities=["SX5E Index"], fields=["PX_LAST"])
except BloombergConnectionError:
    print("Bloomberg terminal is not accessible")
```

---

## Type Mapping (Bloomberg → Polars)

When data is returned from Bloomberg, types are automatically mapped:

| Bloomberg type | Polars type |
|---|---|
| `STRING` | `Utf8` |
| `DOUBLE`, `FLOAT` | `Float64` |
| `INT`, `INT32`, `INT64` | `Int64` |
| `DATE` | `Date` |
| `DATETIME` | `Utf8` |
| `BOOLEAN` | `Boolean` |
| Unknown | `Utf8` (fallback) |

**BQL-specific conversions:**
- ISO date strings (`"2026-04-09T00:00:00Z"`) → `datetime.date(2026, 4, 9)`
- `"NaN"` → `None`
- `"Infinity"` / `"-Infinity"` → `float('inf')` / `float('-inf')`

---

## Return Type Summary

| Function | Return type | Columns |
|---|---|---|
| `get_reference()` | `pl.DataFrame` | `security` + one per field |
| `get_historical()` | `pl.DataFrame` | `security`, `date` + one per field (Float64) |
| `get_bulk()` | `pl.DataFrame` | `security` + columns from bulk structure |
| `get_bql()` | `BqlResult` | Each DataFrame: `ID` + data-item + secondary columns |

---

## Overrides

Overrides are optional key-value **string** pairs that modify Bloomberg field behavior.
They are supported by `get_reference()` and `get_bulk()`.

```python
# Currency override
df = get_reference(
    securities=["FP FP Equity"],
    fields=["PX_LAST"],
    overrides={"EQY_FUND_CRNCY": "USD"},
)

# Date filter on bulk data
df = get_bulk(
    securities=["FP FP Equity"],
    field="DVD_HIST_ALL",
    overrides={"DVD_START_DT": "20200101", "DVD_END_DT": "20251231"},
)
```

**Important:** override values are always strings, and dates must be in `YYYYMMDD` format
(Bloomberg convention).

---

## Caching

All query functions (`get_reference`, `get_historical`, `get_bulk`, `get_bql`)
can be cached on disk. The cache is **opt-in** (default off) and is backed by
`diskcache` (SQLite under the hood) with LRU eviction and TTL.

### Enable

```python
from edfg_bloomany_api import configure
configure(cache_enabled=True)   # use defaults (7-day TTL, 500 MB, ~/.edfg-bloomany-api/query_cache/)
```

Or via env vars:

```env
EDFG_BLOOMANY_CACHE_ENABLED=true
EDFG_BLOOMANY_CACHE_TTL=604800
EDFG_BLOOMANY_CACHE_DIR=/path/to/cache
EDFG_BLOOMANY_CACHE_SIZE_LIMIT=524288000
```

### Per-call override

Every query function accepts `cache: bool | None = None`:

| Value   | Behavior                                                    |
|---------|-------------------------------------------------------------|
| `None`  | Use the global setting (`configure(cache_enabled=...)`)     |
| `True`  | Use the cache even if globally disabled                     |
| `False` | Skip the cache, always hit the network, don't store result  |

```python
# Force a fresh call even if a cached entry exists
df = get_reference(["SX5E Index"], ["PX_LAST"], cache=False)

# Force cache usage even if globally disabled
df = get_reference(["SX5E Index"], ["PX_LAST"], cache=True)
```

### Cache key

`SHA-256(namespace | json.dumps(args, sort_keys=True))`. The namespace is the
function's name (`"reference"`, `"historical"`, `"bulk"`, `"bql"`), and the
payload includes **all call arguments** (securities, fields, overrides, dates,
frequency, expression, host, port). Changing any of them produces a new key.

### Helpers

```python
from edfg_bloomany_api import clear_cache, get_cache_info

clear_cache()        # wipe all entries (e.g. at the start of a trading day)
get_cache_info()
# {
#   "enabled": True,
#   "directory": "/home/user/.edfg-bloomany-api/query_cache",
#   "ttl_seconds": 604800,
#   "size_limit_bytes": 524288000,
#   "size_bytes": 12345,
#   "count": 3,
# }
```

### TTL

Default **7 days**. Override with `cache_ttl_seconds` via `configure()` or
`EDFG_BLOOMANY_CACHE_TTL` if your refresh cadence differs (e.g. `86400` for
daily).

---

## Complete Working Example

```python
import polars as pl
from edfg_bloomany_api import get_reference, get_historical, get_bulk, get_bql
from edfg_bloomany_api.session import BloombergConnectionError

try:
    # 1. Static data
    ref = get_reference(
        securities=["SX5E Index", "CAC Index"],
        fields=["NAME", "CRNCY", "PX_LAST", "COUNT_INDEX_MEMBERS"],
    )
    print(ref)

    # 2. Historical prices
    hist = get_historical(
        securities=["SX5E Index", "CAC Index"],
        fields=["PX_LAST"],
        start="2025-01-01",
        end="2025-03-31",
        frequency="MONTHLY",
    )
    print(hist)

    # 3. Index composition
    comp = get_bulk(securities=["SX5E Index"], field="INDX_MWEIGHT")
    print(comp)

    # 4. BQL screening
    results = get_bql("""
        get(name(), px_last, cur_mkt_cap)
        for(filter(members('SX5E Index'), px_last() > 200))
    """)
    screened = results.combine()
    print(screened)

except BloombergConnectionError:
    print("Bloomberg terminal is not accessible")
```

---

## Known Limitations

- Requires a **locally running Bloomberg terminal** (Desktop API).
- Request timeout is **32 seconds** (hardcoded in `send_and_collect`).
- `get_bulk()` accepts only **one field per call**.
- All return DataFrames are **`polars.DataFrame`**, not pandas.
- BQL `combine()` joins on common columns — if data-items share no common column, it raises `ValueError`.
- Invalid securities are silently dropped (no warning or error).
