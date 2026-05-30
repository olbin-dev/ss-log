# Macro → Industry Translation Engine (Macro Engine) MVP & Roadmap

This document summarizes the technical implementation specifications, verification results, and the long-term development roadmap for the **Macro → Industry Translation Engine (Macro Engine)**, designed to parse macroeconomic trends and score their impact on individual industry sectors.

---

## 1. Project Overview & Directory Layout

The tool ingests real-time macroeconomic indicators from external APIs (like FRED) and computes weighted contribution scores for 17 core industry sectors based on predefined impact rules.

The codebase is organized as follows inside `/Users/p/FIA/translation_engine/`:

```
/Users/p/FIA/translation_engine/
├── README.md                 # Project setup and setup guides
├── requirements.txt          # Python requirements (pandas, numpy, pytest, streamlit, plotly)
├── main.py                   # Integrated CLI demo script
├── app.py                    # Streamlit-based interactive web dashboard
└── src/
    ├── __init__.py
    ├── schema.py             # 17 core sectors, subsector mapping, and impact matrix weights
    ├── detector.py           # EventDetector (evaluates rate, inflation, oil, forex, and GDP trends)
    ├── engine.py             # RuleEngine (calculates weighted sum scores and rankings)
    └── ingestion.py          # FredDataFetcher (real-time ingestion with offline CSV fallback)
```

---

## 2. Core Implementation Logic

### A. Static Schema (`src/schema.py`)
- **17 Core Sectors**: `bank`, `insurance`, `real_estate`, `construction`, `automobile`, `machinery`, `electronics`, `semiconductor`, `trading_company`, `retail`, `shipping`, `energy`, `materials`, `pharma`, `telecom`, `defense`, `railway`.
- **Subsector Mapping (`SUBSECTOR_TO_SECTOR_MAP`)**: Maps granular factors (like `housing`, `gpu_related`, `retail_import`) into their corresponding main sectors.
- **Impact Matrix (`IMPACT_MATRIX`)**: Contains the exact impact weights (e.g., `+50`, `-30`) of macro events on specific sectors.

### B. Event Detection & Severity Mapping (`src/detector.py`)
Evaluates raw changes over 30-day windows (percentage points for interest rates, percentage change for oil and currencies) to determine event severity (levels 1 to 3):
- **10-Year Treasury Yield**: Shifts of $> +0.25\%$, $> +0.5\%$, and $> +1.0\%$ trigger `long_rate_spike` with severity `1`, `2`, and `3`. Decreases trigger `long_rate_drop`.
- **YoY CPI**: Levels of $> 2\%$, $> 3\%$, and $> 5\%$ trigger `inflation_high` with severity `1`, `2`, and `3`. Under-target rates trigger `inflation_low`.
- **WTI Crude Oil Price**: Swings $> \pm 10\%$, $> \pm 20\%$, and $> \pm 40\%$ trigger `oil_price_spike` or `oil_price_crash`.
- **USD/JPY Exchange Rate**: Fluctuations $> \pm 2\%$, $> \pm 5\%$, and $> \pm 10\%$ trigger `yen_weakening` (yen depreciation) or `yen_strengthening`.

### C. Rule Scoring Mathematical Formulation (`src/engine.py`)
For each triggered event, the engine looks up its base sector impact weights from the static matrix and calculates cumulative scores:

$$\text{Industry Score} = \sum (\text{Base Impact Weight} \times \text{Severity})$$

It then normalizes the scores relative to the absolute maximum score across all sectors:

$$\text{Normalized Score} = \frac{\text{Score}}{\max_{s \in \text{Sectors}}(|\text{Score}_s|)}$$

The resulting normalized score ranges from -1.0 (maximum headwind) to +1.0 (maximum tailwind).

---

## 3. Verification & Execution Results

### A. Automated Unit Tests (`pytest`)
A robust test suite verifies the ingestion pipelines, mock data mappings, date offsets, and math calculations:

```bash
platform darwin -- Python 3.14.5, pytest-9.0.3
collected 9 items

tests/test_ingestion.py ....                                             [ 44%]
tests/test_translation.py .....                                          [100%]

============================== 9 passed in 0.27s ===============================
```

### B. Sample CLI Run Execution Output (`python main.py`)
If no FRED API key is provided, the script fetches data from public repository mirrors. Below is an example run:

```
--- TRIGGERED MACRO EVENTS ---
🟢 [MODERATE] inflation_high            | High inflation detected. YoY CPI is 3.8%
🟢 [MODERATE] gdp_growth_accelerating   | GDP growth accelerating. YoY: 5.9%
🟢 [MINOR] ai_investment_boom        | Manual event trigger: ai_investment_boom (severity: 1)
🟢 [MINOR] semiconductor_capex_cycle | Manual event trigger: semiconductor_capex_cycle (severity: 1)
------------------------------

===============================================================================================
RANK | CORE SECTOR        | SCORE    | NORMALIZED | CONTRIBUTING FACTORS & IMPACTS
===============================================================================================
   1 | semiconductor      | +200.0 |    +1.0000 | ai_investment_boom (+50 * 1 = +50), semiconductor_capex_cycle (+50 * 1 = +50)...
-----------------------------------------------------------------------------------------------
   2 | energy             | +140.0 |    +0.7000 | inflation_high (+40 * 2 = +80), inflation_high (+30 * 2 = +60)
-----------------------------------------------------------------------------------------------
   ...
  17 | retail             | -100.0 |    -0.5000 | inflation_high (-30 * 2 = -60)...
```

---

## 4. Web Dashboard features (`app.py`)

The Streamlit web application provides a responsive graphical workspace for macro analysis:
- **KPI Metrics Panel**: Visualizes current interest rates, inflation indicators, currency pairs, oil prices, and GDP metrics along with 30-day directional trends.
- **Dynamic Overrides Panel (Sidebar)**: Interactive sliders and toggle inputs to manually adjust qualitative macro events (e.g. AI boom, semiconductor cycle, geopolitical changes).
- **Interactive Plotly Rankings Chart**: A polarity-colored horizontal bar chart (Green for tailwinds, Red for headwinds, Gray for neutral).
- **Searchable Math Audit Grid**: Allows users to inspect exact base weights and severities used to compute the sector scores.

---

## 5. Development Roadmap (Phases 1-5)

Following the completion of the Streamlit Web Mockup (Phase 1), the engine is scheduled to undergo the following evolution phases:

### Phase 2: Stock Valuation & Ticker Screening (Micro-Scoring Integration)
Translates sector scores down into individual stock picks:
- Ingest fundamental metrics (PER, PBR, Dividend Yield, ROE, Debt/Equity) using Yahoo Finance (`yfinance`) or J-Quants APIs.
- Introduce a compound scoring formula:
  $$\text{Ticker Score} = \text{Normalized Sector Score} \times w_{\text{macro}} + \text{Valuation Score} \times w_{\text{value}}$$
- Build a **BUY / WATCH / AVOID** categorization ruleset.

### Phase 3: 103-Event Library & Japanese domestic Data Expansion
- Ingest Japan cabinet/statistics office data (Domestic CPI, BOJ target rates, BOJ Tankan survey indicators).
- Add support for all 103 macroeconomic events (yield curve inversion, housing starts, container indexes, etc.).

### Phase 4: Desktop Application GUI (PySide6)
- Replace the Streamlit dashboard with a native desktop client developed in PySide6 (Qt for Python).
- Feature a sleek, optimized Dark Mode interface with real-time sortable tables and custom chart visualizations.

### Phase 5: SQLite Caching, Packaging, & Standalone Releases
- Store API requests locally in SQLite database cache to allow fast offline operation.
- Package the application into standalone, double-clickable installers: macOS DMG/APP bundles and Windows EXE executables.
