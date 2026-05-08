# FMP Financial Dashboard 

## Overview

This is a real-time financial analytics dashboard built entirely in **Power BI**, powered by the **Financial Modeling Prep (FMP) API**. It allows users to select any supported company and instantly view its valuation signal, financial health, price history, geographic revenue, and key return metrics — all refreshed daily from live market data.

**Live Dashboard →** [View on Power BI](https://app.powerbi.com/view?r=eyJrIjoiYTdhNDg2MWYtZmE0Yy00ZTA3LWI4YTUtNWE4MWExNDcxNjQ0IiwidCI6ImQxZjE0MzQ4LWYxYjUtNGEwOS1hYzk5LTdlYmYyMTNjYmM4MSIsImMiOjEwfQ%3D%3D&pageName=7c82d96122b68c2e4a77)

**Share Link →** [Open Dashboard](https://app.powerbi.com/links/l8OHFrmRmF?ctid=d1f14348-f1b5-4a09-ac99-7ebf213cbc81&pbi_source=linkShare)

> **Note:** Due to FMP free-tier API rate limits, the dashboard currently supports the following 9 companies:
> `AAPL` · `MSFT` · `GOOGL` · `AMZN` · `NVDA` · `META` · `TSLA` · `NFLX` · `V`

---

## 1. Data Loading Architecture

### The FMP API

All data is sourced from the **Financial Modeling Prep (FMP) REST API** (`financialmodelingprep.com`). Each API endpoint returns a **JSON response** which Power Query parses, expands, and transforms into structured tables. A single `API_KEY` parameter is stored centrally and injected into every request.

The company list is maintained as a simple Power Query list:

```
{"AAPL", "MSFT", "GOOGL", "AMZN", "NVDA", "META", "TSLA", "NFLX", "V"}
```

### Functions + Loop Pattern

Each data domain has two components:

| Component | Purpose |
|---|---|
| **Function** (`fn_*`) | Takes a `symbol` as input, calls the API, parses JSON → returns a clean table |
| **`_All` query** | Loops over `companies_list`, calls the function for each symbol, combines all results with `Table.Combine()` |

The loop pattern used in every `_All` query:

```powerquery
Tables = List.Transform(companies_list, each
    try fn_getXxx(_)
    otherwise #table({col1, col2, ...}, {})
),
Combined = Table.Combine(Tables)
```

The `try...otherwise` block ensures that if any single API call fails (e.g. rate limit), the entire load does not break — it gracefully returns an empty table for that symbol.

### JSON → Table Transformation

Every API response follows the same cleaning pipeline:

1. `Web.Contents()` fetches raw JSON
2. `Json.Document()` parses it into a Power Query list or record
3. `Table.FromList()` converts the list into rows
4. `Table.ExpandRecordColumn()` flattens nested JSON fields into flat columns
5. `Table.TransformColumnTypes()` sets correct data types (date, number, text, integer)

### API Endpoints & Functions

| Function | API Endpoint | Output Table | Data |
|---|---|---|---|
| `fn_getQuote` | `/stable/quote` | `Quote_All` | Live price, volume, market cap, change % |
| `fn_getProfile` | `/stable/sec-profile` | `Profile_All` | Company name, CEO, IPO date, description, logo |
| `fn_getMetrics` | `/stable/financial-ratios` | `Metrics_All` | ROE, ROA, ROCE, ROIC, current ratio, cash conversion |
| `fn_getDCF` | `/stable/discounted-cash-flow` | `DCF_All` | Intrinsic value (DCF), stock price |
| `fn_getgeodata` | `/stable/revenue-geographic-segmentation` | `GeoSegment_All` | Revenue by region per fiscal year |
| `fn_getHistory` | `/stable/historical-price-eod/light` | `PriceHistory_All` | Daily price + volume for past 30 days |

### Special: Geographic Segmentation Cleaning

The geo API returns region names inconsistently across companies (e.g. `"North America"`, `"UNITED STATES"`, `"Americas Excluding United States"`). A `MapRegion` column is added via Power Query that normalizes all variants into standard country names that ArcGIS Maps can geocode:

```
"North America"  → "United States"
"Middle East"    → "Saudi Arabia"
"EMEA"           → "Germany"
"Greater China"  → "China"
"Asia Pacific"   → "Australia"
```

Rows mapping to `"World"` (un-geocodable segments) are filtered out.

---

## 2. Data Model & Schema

The model uses a **star schema** with `companies_list` as the central dimension table. All `_All` fact tables join to it via the `symbol` column.

```
                    companies_list  (symbol, companyName, DisplayName)
                          │
        ┌─────────────────┼──────────────────────────┐
        │                 │                          │
   Quote_All         Profile_All              Metrics_All
 (price, volume,   (CEO, description,      (ROE, ROA, ROCE,
  marketCap,        exchange, ipoDate,      ROIC, ratios)
  change%)          employees, logo)
        │                 │
   DCF_All          GeoSegment_All           PriceHistory_All
 (dcf,             (region, MapRegion,      (date, price,
  stockPrice)       revenue, fiscalYear)     volume, DaysAgo)
```

**Supporting tables (no API call):**

| Table | Purpose |
|---|---|
| `WindowSlicer` | Drives the 7 / 15 / 30 Days slicer for price history |
| `RangeSupport` | Stores 52W and Day range metadata for gauge visuals |
| `ReturnsSupport` | Maps metric names to display labels for the returns chart |
| `SymbolList` | Display names for the company filter dropdown |
| `Quote_header` | Single-row filtered view of `Quote_All` for header KPI cards |
| `Measures_All` | Dedicated DAX measure table (no data rows) |

---

<img width="1188" height="865" alt="image" src="https://github.com/user-attachments/assets/653a832c-7360-4d98-ab07-5b2f5a7e3180" />


## 3. DAX Measures

All measures live in a single `Measures_All` table for clean organisation.

### Signal & Valuation

```dax
-- Determines BUY / SELL / HOLD based on DCF vs current price
Overall Signal =
VAR dcf   = SELECTEDVALUE(DCF_All[dcf])
VAR price = SELECTEDVALUE(DCF_All[StockPrice])
VAR diff  = DIVIDE(dcf - price, price)
RETURN
    IF(diff > 0.15, "BUY", IF(diff < -0.15, "SELL", "HOLD"))

-- DCF % over/under valuation
DCF % =
VAR dcf   = SELECTEDVALUE(DCF_All[dcf])
VAR price = SELECTEDVALUE(DCF_All[StockPrice])
RETURN DIVIDE(dcf - price, price)
```

### Price History (Window-Filtered)

```dax
Selected Days = SELECTEDVALUE(WindowSlicer[Days], 30)

Avg Price =
VAR d = [Selected Days]
RETURN CALCULATE(
    AVERAGE(HistoricalPrice_All[price]),
    HistoricalPrice_All[DaysAgo] <= d
)

Total Volume =
VAR d = [Selected Days]
RETURN CALCULATE(
    SUM(HistoricalPrice_All[volume]),
    HistoricalPrice_All[DaysAgo] <= d
)
```

### 52-Week & Day Range

```dax
52W Position =
VAR price = SELECTEDVALUE(Quote_All[price])
VAR low   = SELECTEDVALUE(Measures_All[52W Low])
VAR high  = SELECTEDVALUE(Measures_All[52W High])
RETURN DIVIDE(price - low, high - low)

Day MakePointer =
-- Returns the needle position for the DAY Range gauge
VAR price   = SELECTEDVALUE(Quote_All[price])
VAR dayLow  = SELECTEDVALUE(Quote_All[dayLow])
VAR dayHigh = SELECTEDVALUE(Quote_All[dayHigh])
RETURN DIVIDE(price - dayLow, dayHigh - dayLow)
```

### Financial Health Score

```dax
Financial Health Score =
-- Weighted composite of liquidity, profitability, and efficiency ratios
-- Normalised to a 0–100 scale
VAR cr   = SELECTEDVALUE(Metrics_All[CurrentRatio])
VAR roe  = SELECTEDVALUE(Metrics_All[ReturnOnEquity])
VAR ...
RETURN ROUND(score * 100, 2)
```

---

## 4. Visuals

<img width="1378" height="901" alt="image" src="https://github.com/user-attachments/assets/5416c70e-9ecb-4271-8596-3b9aece79312" />
<img width="1350" height="783" alt="image" src="https://github.com/user-attachments/assets/5328850a-8c45-46cc-8704-dbab23c7d607" />



### Header Row
- **Company Filter** — dropdown slicer filtering all visuals by `symbol`
- **Market Index Ticker** — `^BSESN` live price card with change and % change
- **Last Updated** — card showing `NOW()` timestamp

### Signal & Valuation Section
| Visual | Description |
|---|---|
| **Overall Signal card** | BUY / SELL / HOLD with conditional color (green/red/yellow) |
| **DCF % card** | Overvalued / Undervalued % with current price vs intrinsic value |
| **Financial Health Score** | Radial gauge (0–100), red = weak, green = strong |
| **52W Range gauge** | Needle showing where current price sits in the 52-week range |
| **DAY Range gauge** | Needle showing intraday position (Day Low → Day High) |

### Right Panel KPIs
- **Current Volume** — today's traded volume
- **Avg Volume** — 30-day average volume
- **Market Cap** — formatted in T/B/M
- **Fiscal Year End** — from profile data
- **52-Week Range** — text label (Low – High)

### Company Profile Section
- **Logo** — image from Clearbit Logo API via `logoUrl` column
- **Company Name + Symbol**
- **Sector, Exchange, Country** — with icons
- **CEO, IPO Date, No. of Employees**
- **Company Description** — scrollable text card
- **Website link**

### Analytics Section
| Visual | Type | Description |
|---|---|---|
| **Returns Value by Metric** | Horizontal bar chart | ROE, ROCE, ROA, ROIC side by side |
| **Revenue by MapRegion** | ArcGIS Filled Map | Geographic revenue bubbles, filtered by fiscal year slicer |
| **Price & Volume Chart** | Line + Clustered Column | Bars = volume, line = price, filtered by 7/15/30 day slicer |

---

## 5. Refresh & Deployment

The dashboard is published to **Power BI Service** and configured with a daily scheduled refresh. On each refresh, all six API functions re-execute, fetching the latest quotes, financials, and price history for all supported symbols. The `DaysAgo` column in `PriceHistory_All` is computed dynamically at refresh time using `DateTime.LocalNow()`, so the 7/15/30 day windows are always relative to the current date.

