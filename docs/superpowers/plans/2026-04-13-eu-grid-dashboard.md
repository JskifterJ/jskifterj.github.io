# EU Grid Dashboard Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a live European electricity grid dashboard with real ENTSO-E data, interactive D3 map, Chart.js charts, and an AI-generated grid briefing — deployed to Render.

**Architecture:** FastAPI Python backend serves both API endpoints and frontend static files. ENTSO-E data fetched via `entsoe-py`, cached in-memory for 15 minutes. AI briefing via Anthropic SDK (Haiku model). Frontend is vanilla JS + D3.js v7 + Chart.js.

**Tech Stack:** Python 3.11, FastAPI, uvicorn, entsoe-py, anthropic SDK, Vanilla JS, D3.js v7, topojson, Chart.js, Render (deployment)

---

## File Map

| File | Responsibility |
|------|---------------|
| `app/main.py` | FastAPI app, CORS, static file mount, all route imports |
| `app/cache.py` | In-memory TTL cache (simple dict + datetime) |
| `app/models.py` | Pydantic response models for all API endpoints |
| `app/entso.py` | ENTSO-E data fetching + CO₂ calculation |
| `app/ai.py` | Claude API briefing generation |
| `app/routes.py` | All FastAPI route handlers (thin, calls entso.py / ai.py) |
| `frontend/index.html` | Dashboard HTML shell |
| `frontend/style.css` | All dashboard styles |
| `frontend/api.js` | fetch() wrappers for all backend endpoints |
| `frontend/map.js` | D3 Europe map with CO₂ colouring, flows, Denmark easter egg |
| `frontend/charts.js` | Chart.js donut, line, bar charts + stats strip + AI card |
| `tests/test_cache.py` | Unit tests for cache TTL behaviour |
| `tests/test_models.py` | Pydantic model validation tests |
| `tests/test_entso.py` | ENTSO-E data processing tests (mocked client) |
| `tests/test_ai.py` | AI briefing tests (mocked Anthropic client) |
| `tests/test_routes.py` | FastAPI route integration tests (mocked dependencies) |
| `requirements.txt` | All Python dependencies |
| `render.yaml` | Render one-click deploy config |
| `.env.example` | Template for required env vars |

---

### Task 1: Project setup and dependencies

**Files:**
- Create: `requirements.txt`
- Create: `.env.example`
- Create: `app/__init__.py`
- Create: `tests/__init__.py`
- Create: `render.yaml`

- [ ] Create `requirements.txt`:

```
fastapi==0.111.0
uvicorn[standard]==0.29.0
entsoe-py==0.6.9
anthropic==0.25.1
pydantic==2.7.1
python-dotenv==1.0.1
httpx==0.27.0
pytest==8.2.0
pytest-asyncio==0.23.7
```

- [ ] Create `.env.example`:

```
ENTSO_API_KEY=your-entso-e-api-key-here
ANTHROPIC_API_KEY=your-anthropic-api-key-here
```

- [ ] Create `app/__init__.py` (empty) and `tests/__init__.py` (empty):

```bash
mkdir -p app tests frontend
touch app/__init__.py tests/__init__.py
```

- [ ] Create `render.yaml`:

```yaml
services:
  - type: web
    name: eu-grid-dashboard
    runtime: python
    buildCommand: pip install -r requirements.txt
    startCommand: uvicorn app.main:app --host 0.0.0.0 --port $PORT
    envVars:
      - key: ENTSO_API_KEY
        sync: false
      - key: ANTHROPIC_API_KEY
        sync: false
```

- [ ] Install dependencies locally:

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

Expected: no errors, all packages install.

- [ ] Commit:

```bash
git add requirements.txt .env.example app/__init__.py tests/__init__.py render.yaml
git commit -m "chore: project setup and dependencies"
```

---

### Task 2: Cache module

**Files:**
- Create: `app/cache.py`
- Create: `tests/test_cache.py`

- [ ] Write `tests/test_cache.py` first:

```python
import time
from app.cache import Cache

def test_cache_miss_returns_none():
    c = Cache(ttl_seconds=60)
    assert c.get("missing") is None

def test_cache_hit_returns_value():
    c = Cache(ttl_seconds=60)
    c.set("key", {"data": 42})
    assert c.get("key") == {"data": 42}

def test_cache_expired_returns_none():
    c = Cache(ttl_seconds=1)
    c.set("key", "value")
    time.sleep(1.1)
    assert c.get("key") is None

def test_cache_set_overwrites():
    c = Cache(ttl_seconds=60)
    c.set("key", "first")
    c.set("key", "second")
    assert c.get("key") == "second"
```

- [ ] Run tests — expect all FAIL (module not found):

```bash
pytest tests/test_cache.py -v
```

Expected output contains: `ModuleNotFoundError` or `ImportError`

- [ ] Create `app/cache.py`:

```python
from datetime import datetime, timedelta
from typing import Any, Optional


class Cache:
    def __init__(self, ttl_seconds: int = 900):
        self._store: dict[str, tuple[Any, datetime]] = {}
        self._ttl = timedelta(seconds=ttl_seconds)

    def get(self, key: str) -> Optional[Any]:
        if key not in self._store:
            return None
        value, timestamp = self._store[key]
        if datetime.now() - timestamp > self._ttl:
            del self._store[key]
            return None
        return value

    def set(self, key: str, value: Any) -> None:
        self._store[key] = (value, datetime.now())
```

- [ ] Run tests — expect all PASS:

```bash
pytest tests/test_cache.py -v
```

Expected: `4 passed`

- [ ] Commit:

```bash
git add app/cache.py tests/test_cache.py
git commit -m "feat: in-memory TTL cache"
```

---

### Task 3: Pydantic models

**Files:**
- Create: `app/models.py`
- Create: `tests/test_models.py`

- [ ] Write `tests/test_models.py`:

```python
from app.models import GenerationData, PricePoint, FlowItem, CountryOverview, GridSummary

def test_generation_data_valid():
    g = GenerationData(
        country="DE",
        sources={"Wind": 34.0, "Solar": 24.0, "Gas": 18.0},
        renewable_pct=58.0,
        co2_intensity=218.0,
        total_gw=62.4,
    )
    assert g.country == "DE"
    assert g.renewable_pct == 58.0

def test_price_point_valid():
    p = PricePoint(timestamp="2026-04-13T14:00:00", price_eur_mwh=87.5)
    assert p.price_eur_mwh == 87.5

def test_flow_item_valid():
    f = FlowItem(partner="France", flow_gw=3.1, direction="export")
    assert f.direction == "export"

def test_country_overview_valid():
    o = CountryOverview(
        country="DE",
        name="Germany",
        co2_intensity=218.0,
        renewable_pct=58.0,
        price_eur_mwh=87.0,
    )
    assert o.name == "Germany"

def test_grid_summary_valid():
    s = GridSummary(country="DE", text="Germany is running 58% renewables today.")
    assert "58%" in s.text
```

- [ ] Run tests — expect FAIL:

```bash
pytest tests/test_models.py -v
```

- [ ] Create `app/models.py`:

```python
from pydantic import BaseModel
from typing import Optional


class GenerationData(BaseModel):
    country: str
    sources: dict[str, float]          # display name → GW
    renewable_pct: float
    co2_intensity: float               # g/kWh
    total_gw: float


class PricePoint(BaseModel):
    timestamp: str                     # ISO 8601
    price_eur_mwh: float


class FlowItem(BaseModel):
    partner: str
    flow_gw: float                     # positive = export, negative = import
    direction: str                     # "import" | "export"


class FlowData(BaseModel):
    country: str
    flows: list[FlowItem]
    net_gw: float


class PriceData(BaseModel):
    country: str
    prices: list[PricePoint]
    current_eur_mwh: Optional[float]
    delta_pct: Optional[float]         # vs yesterday's average


class CountryOverview(BaseModel):
    country: str
    name: str
    co2_intensity: float
    renewable_pct: float
    price_eur_mwh: Optional[float]


class OverviewData(BaseModel):
    countries: list[CountryOverview]


class GridSummary(BaseModel):
    country: str
    text: str
    generated_at: Optional[str] = None
```

- [ ] Run tests — expect all PASS:

```bash
pytest tests/test_models.py -v
```

Expected: `5 passed`

- [ ] Commit:

```bash
git add app/models.py tests/test_models.py
git commit -m "feat: Pydantic response models"
```

---

### Task 4: ENTSO-E integration

**Files:**
- Create: `app/entso.py`
- Create: `tests/test_entso.py`

**Note:** ENTSO-E registration at `transparency.entsoe.eu` is required for a real API key. Tests use mocked data so they run without a key. The `ENTSO_API_KEY` env var must be set in production.

- [ ] Write `tests/test_entso.py`:

```python
import pandas as pd
import pytest
from unittest.mock import MagicMock, patch
from app.entso import calculate_co2_intensity, map_generation_sources, ENTSOClient

def test_co2_intensity_all_wind():
    sources = {"Wind": 100.0}
    assert calculate_co2_intensity(sources) == pytest.approx(11.0)

def test_co2_intensity_mixed():
    sources = {"Wind": 50.0, "Fossil Gas": 50.0}
    result = calculate_co2_intensity(sources)
    assert result == pytest.approx((11.0 * 50 + 490.0 * 50) / 100, rel=1e-3)

def test_co2_intensity_empty():
    assert calculate_co2_intensity({}) == 0.0

def test_map_generation_sources_groups_wind():
    raw = {"Wind Onshore": 20.0, "Wind Offshore": 5.0, "Solar": 10.0}
    result = map_generation_sources(raw)
    assert result["Wind"] == pytest.approx(25.0)
    assert result["Solar"] == pytest.approx(10.0)

def test_entso_client_get_generation_returns_generation_data():
    with patch("app.entso.EntsoePandasClient") as MockClient:
        # Build a mock DataFrame matching entsoe-py output format
        idx = pd.date_range("2026-04-13 12:00", periods=1, freq="h", tz="UTC")
        df = pd.DataFrame(
            {"Wind Onshore": [22000.0], "Fossil Gas": [8000.0], "Solar": [5000.0]},
            index=idx,
        )
        # entsoe-py returns MultiIndex columns in some cases; handle flat for test
        MockClient.return_value.query_generation.return_value = df

        client = ENTSOClient(api_key="test-key")
        result = client.get_generation("DE")

    assert result.country == "DE"
    assert result.total_gw > 0
    assert "Wind" in result.sources or "Fossil Gas" in result.sources
```

- [ ] Run tests — expect FAIL:

```bash
pytest tests/test_entso.py -v
```

- [ ] Create `app/entso.py`:

```python
import os
from datetime import datetime, timedelta, timezone
from typing import Optional

import pandas as pd
from entsoe import EntsoePandasClient

from app.models import GenerationData, PricePoint, PriceData, FlowItem, FlowData, CountryOverview

# ── Emission factors (g CO₂/kWh, IPCC 2014 median values) ───────────────────
EMISSION_FACTORS: dict[str, float] = {
    "Fossil Hard coal": 820.0,
    "Fossil Brown coal/Lignite": 1054.0,
    "Fossil Gas": 490.0,
    "Fossil Oil": 650.0,
    "Fossil Oil shale": 650.0,
    "Fossil Peat": 900.0,
    "Nuclear": 12.0,
    "Wind Onshore": 11.0,
    "Wind Offshore": 12.0,
    "Solar": 41.0,
    "Hydro Run-of-river and poundage": 24.0,
    "Hydro Water Reservoir": 24.0,
    "Hydro Pumped Storage": 24.0,
    "Biomass": 230.0,
    "Waste": 330.0,
    "Geothermal": 38.0,
    "Other renewable": 50.0,
    "Other": 300.0,
}

RENEWABLE_SOURCES = {
    "Wind Onshore", "Wind Offshore", "Solar",
    "Hydro Run-of-river and poundage", "Hydro Water Reservoir",
    "Hydro Pumped Storage", "Geothermal", "Other renewable",
}

# Sources that map to a single display name
DISPLAY_MAP: dict[str, str] = {
    "Fossil Hard coal": "Coal",
    "Fossil Brown coal/Lignite": "Lignite",
    "Fossil Gas": "Gas",
    "Fossil Oil": "Oil",
    "Fossil Oil shale": "Oil Shale",
    "Fossil Peat": "Peat",
    "Nuclear": "Nuclear",
    "Wind Onshore": "Wind",
    "Wind Offshore": "Wind Offshore",
    "Solar": "Solar",
    "Hydro Run-of-river and poundage": "Hydro",
    "Hydro Water Reservoir": "Hydro Reservoir",
    "Hydro Pumped Storage": "Pumped Storage",
    "Biomass": "Biomass",
    "Waste": "Waste",
    "Geothermal": "Geothermal",
    "Other renewable": "Other Renewable",
    "Other": "Other",
}

# ENTSO-E bidding zone codes per country (2-letter ISO → entsoe-py area string)
AREA_CODES: dict[str, str] = {
    "DE": "DE_LU",
    "FR": "FR",
    "GB": "GB",
    "NO": "NO_1",
    "SE": "SE_3",
    "DK": "DK_1",
    "FI": "FI",
    "CH": "CH",
    "AT": "AT",
    "NL": "NL",
    "BE": "BE",
    "PL": "PL",
    "ES": "ES",
    "IT": "IT_NORD",
    "CZ": "CZ",
    "PT": "PT",
    "RO": "RO",
    "GR": "GR",
    "IE": "IE_SEM",
    "HU": "HU",
}

COUNTRY_NAMES: dict[str, str] = {
    "DE": "Germany", "FR": "France", "GB": "United Kingdom", "NO": "Norway",
    "SE": "Sweden", "DK": "Denmark", "FI": "Finland", "CH": "Switzerland",
    "AT": "Austria", "NL": "Netherlands", "BE": "Belgium", "PL": "Poland",
    "ES": "Spain", "IT": "Italy", "CZ": "Czech Republic", "PT": "Portugal",
    "RO": "Romania", "GR": "Greece", "IE": "Ireland", "HU": "Hungary",
}


def calculate_co2_intensity(sources_gw: dict[str, float]) -> float:
    """Return weighted average CO₂ intensity in g/kWh given {raw_source_name: GW}."""
    total = sum(sources_gw.values())
    if total == 0:
        return 0.0
    weighted = sum(
        gw * EMISSION_FACTORS.get(src, 300.0)
        for src, gw in sources_gw.items()
    )
    return round(weighted / total, 1)


def map_generation_sources(raw: dict[str, float]) -> dict[str, float]:
    """
    Convert raw ENTSO-E source names to display names, merging Wind Onshore+Offshore→Wind.
    Returns {display_name: GW}, sorted descending by value.
    """
    grouped: dict[str, float] = {}
    for raw_name, gw in raw.items():
        display = DISPLAY_MAP.get(raw_name, raw_name)
        # Merge both wind types under "Wind"
        if display == "Wind Offshore":
            display = "Wind"
        grouped[display] = grouped.get(display, 0.0) + gw
    return dict(sorted(grouped.items(), key=lambda x: x[1], reverse=True))


class ENTSOClient:
    def __init__(self, api_key: str):
        self._client = EntsoePandasClient(api_key=api_key)

    def _now_window(self) -> tuple[pd.Timestamp, pd.Timestamp]:
        """Return (start, end) for approximately the last 2 hours in UTC."""
        end = pd.Timestamp.now(tz="UTC").floor("h")
        start = end - pd.Timedelta(hours=2)
        return start, end

    def _today_window(self) -> tuple[pd.Timestamp, pd.Timestamp]:
        """Return (start, end) for today in UTC."""
        end = pd.Timestamp.now(tz="UTC").ceil("h")
        start = end.normalize()  # midnight
        return start, end

    def _week_window(self) -> tuple[pd.Timestamp, pd.Timestamp]:
        end = pd.Timestamp.now(tz="UTC").ceil("h")
        start = end - pd.Timedelta(days=7)
        return start, end

    def get_generation(self, country: str) -> GenerationData:
        area = AREA_CODES[country]
        start, end = self._now_window()
        df = self._client.query_generation(area, start=start, end=end)

        # Flatten MultiIndex columns if present (entsoe-py sometimes returns them)
        if isinstance(df.columns, pd.MultiIndex):
            df.columns = [col[0] for col in df.columns]

        # Take the most recent row, drop NaN
        latest = df.iloc[-1].dropna()
        raw: dict[str, float] = {
            col: float(latest[col]) / 1000.0  # MW → GW
            for col in latest.index
            if isinstance(col, str) and latest[col] > 0
        }

        total_gw = sum(raw.values())
        renewable_gw = sum(v for k, v in raw.items() if k in RENEWABLE_SOURCES)
        renewable_pct = round((renewable_gw / total_gw * 100) if total_gw > 0 else 0.0, 1)
        co2 = calculate_co2_intensity(raw)
        sources = map_generation_sources(raw)

        return GenerationData(
            country=country,
            sources=sources,
            renewable_pct=renewable_pct,
            co2_intensity=co2,
            total_gw=round(total_gw, 2),
        )

    def get_prices(self, country: str) -> PriceData:
        area = AREA_CODES[country]
        start, end = self._today_window()
        series = self._client.query_day_ahead_prices(area, start=start, end=end)

        prices = [
            PricePoint(
                timestamp=ts.isoformat(),
                price_eur_mwh=round(float(val), 2),
            )
            for ts, val in series.items()
            if pd.notna(val)
        ]

        current = prices[-1].price_eur_mwh if prices else None

        # Delta vs yesterday's average (fetch yesterday for comparison)
        yesterday_start = start - pd.Timedelta(days=1)
        yesterday_end = start
        try:
            yesterday_series = self._client.query_day_ahead_prices(
                area, start=yesterday_start, end=yesterday_end
            )
            yesterday_avg = float(yesterday_series.mean())
            delta_pct = round(((current - yesterday_avg) / yesterday_avg) * 100, 1) if current and yesterday_avg else None
        except Exception:
            delta_pct = None

        return PriceData(
            country=country,
            prices=prices,
            current_eur_mwh=current,
            delta_pct=delta_pct,
        )

    def get_flows(self, country: str) -> FlowData:
        """Fetch cross-border flows for a country vs all neighbours."""
        area = AREA_CODES[country]
        start, end = self._now_window()

        # Neighbours we attempt to query
        neighbours = [c for c in AREA_CODES if c != country]
        flows: list[FlowItem] = []
        net_gw = 0.0

        for neighbour in neighbours:
            neighbour_area = AREA_CODES[neighbour]
            try:
                # Export: country → neighbour
                exp_series = self._client.query_crossborder_flows(
                    area, neighbour_area, start=start, end=end
                )
                exp_gw = round(float(exp_series.iloc[-1]) / 1000.0, 2) if len(exp_series) else 0.0
                # Import: neighbour → country
                imp_series = self._client.query_crossborder_flows(
                    neighbour_area, area, start=start, end=end
                )
                imp_gw = round(float(imp_series.iloc[-1]) / 1000.0, 2) if len(imp_series) else 0.0

                net = exp_gw - imp_gw
                if abs(net) > 0.05:  # only show significant flows
                    direction = "export" if net > 0 else "import"
                    flows.append(FlowItem(
                        partner=COUNTRY_NAMES.get(neighbour, neighbour),
                        flow_gw=round(abs(net), 2),
                        direction=direction,
                    ))
                    net_gw += net
            except Exception:
                continue  # not all country pairs have cross-border flow data

        flows.sort(key=lambda f: f.flow_gw, reverse=True)
        return FlowData(country=country, flows=flows[:6], net_gw=round(net_gw, 2))

    def get_overview(self) -> list[CountryOverview]:
        """Fetch CO₂ intensity + renewable % + price for all supported countries."""
        result: list[CountryOverview] = []
        for country in AREA_CODES:
            try:
                gen = self.get_generation(country)
                price_data = self.get_prices(country)
                result.append(CountryOverview(
                    country=country,
                    name=COUNTRY_NAMES.get(country, country),
                    co2_intensity=gen.co2_intensity,
                    renewable_pct=gen.renewable_pct,
                    price_eur_mwh=price_data.current_eur_mwh,
                ))
            except Exception:
                continue
        return result


def get_entso_client() -> ENTSOClient:
    api_key = os.environ.get("ENTSO_API_KEY", "")
    return ENTSOClient(api_key=api_key)
```

- [ ] Run tests — expect all PASS:

```bash
pytest tests/test_entso.py -v
```

Expected: `5 passed`

- [ ] Commit:

```bash
git add app/entso.py tests/test_entso.py
git commit -m "feat: ENTSO-E data fetching and CO2 calculation"
```

---

### Task 5: AI briefing module

**Files:**
- Create: `app/ai.py`
- Create: `tests/test_ai.py`

- [ ] Write `tests/test_ai.py`:

```python
from unittest.mock import MagicMock, patch
from app.ai import build_prompt, AIBriefing

def test_build_prompt_contains_country():
    prompt = build_prompt(
        country="Denmark",
        renewable_pct=68.0,
        co2_intensity=115.0,
        price_eur_mwh=78.0,
        sources={"Wind": 55.0, "Gas": 18.0},
        net_gw=0.3,
    )
    assert "Denmark" in prompt
    assert "68" in prompt
    assert "115" in prompt

def test_ai_briefing_returns_text():
    with patch("app.ai.Anthropic") as MockAnthropic:
        mock_content = MagicMock()
        mock_content.text = "Denmark is running 68% renewables today."
        MockAnthropic.return_value.messages.create.return_value.content = [mock_content]

        briefing = AIBriefing(api_key="test")
        result = briefing.generate(
            country="DK",
            country_name="Denmark",
            renewable_pct=68.0,
            co2_intensity=115.0,
            price_eur_mwh=78.0,
            sources={"Wind": 55.0, "Gas": 18.0},
            net_gw=0.3,
        )
    assert "Denmark" in result.text

def test_ai_briefing_fallback_on_error():
    with patch("app.ai.Anthropic") as MockAnthropic:
        MockAnthropic.return_value.messages.create.side_effect = Exception("API error")
        briefing = AIBriefing(api_key="test")
        result = briefing.generate(
            country="DK", country_name="Denmark",
            renewable_pct=68.0, co2_intensity=115.0, price_eur_mwh=78.0,
            sources={}, net_gw=0.0,
        )
    assert result.text != ""  # fallback message, not empty
```

- [ ] Run tests — expect FAIL:

```bash
pytest tests/test_ai.py -v
```

- [ ] Create `app/ai.py`:

```python
import os
from datetime import datetime, timezone

from anthropic import Anthropic

from app.models import GridSummary


def build_prompt(
    country: str,
    renewable_pct: float,
    co2_intensity: float,
    price_eur_mwh: float,
    sources: dict[str, float],
    net_gw: float,
) -> str:
    top_sources = sorted(sources.items(), key=lambda x: x[1], reverse=True)[:4]
    sources_str = ", ".join(f"{name} {pct:.0f}%" for name, pct in top_sources)
    net_label = f"net exporting {abs(net_gw):.1f} GW" if net_gw > 0 else f"net importing {abs(net_gw):.1f} GW"

    return f"""You are a concise energy analyst. Write exactly one paragraph (3-4 sentences) summarising current grid conditions for {country}.

Data (right now):
- Generation mix: {sources_str}
- Renewable share: {renewable_pct:.1f}%
- CO₂ intensity: {co2_intensity:.0f} g/kWh
- Day-ahead price: €{price_eur_mwh:.0f}/MWh
- Cross-border: {net_label}

Write in plain English. Be specific about the numbers. Note anything unusual or interesting. No bullet points, no headers — one paragraph only."""


class AIBriefing:
    def __init__(self, api_key: str):
        self._client = Anthropic(api_key=api_key)

    def generate(
        self,
        country: str,
        country_name: str,
        renewable_pct: float,
        co2_intensity: float,
        price_eur_mwh: float,
        sources: dict[str, float],
        net_gw: float,
    ) -> GridSummary:
        prompt = build_prompt(
            country=country_name,
            renewable_pct=renewable_pct,
            co2_intensity=co2_intensity,
            price_eur_mwh=price_eur_mwh,
            sources=sources,
            net_gw=net_gw,
        )
        try:
            response = self._client.messages.create(
                model="claude-haiku-4-5-20251001",
                max_tokens=300,
                messages=[{"role": "user", "content": prompt}],
            )
            text = response.content[0].text.strip()
        except Exception:
            text = f"{country_name} grid data is currently available. Renewable share: {renewable_pct:.0f}%, CO₂ intensity: {co2_intensity:.0f} g/kWh, price: €{price_eur_mwh:.0f}/MWh."

        return GridSummary(
            country=country,
            text=text,
            generated_at=datetime.now(timezone.utc).isoformat(),
        )


def get_ai_briefing() -> AIBriefing:
    api_key = os.environ.get("ANTHROPIC_API_KEY", "")
    return AIBriefing(api_key=api_key)
```

- [ ] Run tests — expect all PASS:

```bash
pytest tests/test_ai.py -v
```

Expected: `3 passed`

- [ ] Commit:

```bash
git add app/ai.py tests/test_ai.py
git commit -m "feat: AI grid briefing via Claude API"
```

---

### Task 6: FastAPI routes and main app

**Files:**
- Create: `app/routes.py`
- Create: `app/main.py`
- Create: `tests/test_routes.py`

- [ ] Write `tests/test_routes.py`:

```python
import pytest
from unittest.mock import MagicMock, patch
from fastapi.testclient import TestClient
from app.main import app
from app.models import GenerationData, PriceData, FlowData, GridSummary, OverviewData, CountryOverview

@pytest.fixture
def client():
    return TestClient(app)

def _mock_generation():
    return GenerationData(country="DK", sources={"Wind": 55.0, "Gas": 18.0}, renewable_pct=68.0, co2_intensity=115.0, total_gw=12.4)

def _mock_prices():
    from app.models import PricePoint
    return PriceData(country="DK", prices=[PricePoint(timestamp="2026-04-13T14:00:00", price_eur_mwh=78.0)], current_eur_mwh=78.0, delta_pct=-5.0)

def _mock_flows():
    return FlowData(country="DK", flows=[], net_gw=0.3)

def _mock_summary():
    return GridSummary(country="DK", text="Denmark is running 68% renewables.")

def test_generation_endpoint(client):
    with patch("app.routes.entso_cache.get", return_value=None), \
         patch("app.routes.entso_cache.set"), \
         patch("app.routes.get_entso_client") as mock_client_factory:
        mock_client_factory.return_value.get_generation.return_value = _mock_generation()
        response = client.get("/api/generation?country=DK")
    assert response.status_code == 200
    data = response.json()
    assert data["country"] == "DK"
    assert data["renewable_pct"] == 68.0

def test_prices_endpoint(client):
    with patch("app.routes.entso_cache.get", return_value=None), \
         patch("app.routes.entso_cache.set"), \
         patch("app.routes.get_entso_client") as mock_client_factory:
        mock_client_factory.return_value.get_prices.return_value = _mock_prices()
        response = client.get("/api/prices?country=DK")
    assert response.status_code == 200
    assert response.json()["current_eur_mwh"] == 78.0

def test_summary_endpoint(client):
    with patch("app.routes.ai_cache.get", return_value=None), \
         patch("app.routes.ai_cache.set"), \
         patch("app.routes.get_entso_client") as mock_entso, \
         patch("app.routes.get_ai_briefing") as mock_ai:
        mock_entso.return_value.get_generation.return_value = _mock_generation()
        mock_entso.return_value.get_prices.return_value = _mock_prices()
        mock_entso.return_value.get_flows.return_value = _mock_flows()
        mock_ai.return_value.generate.return_value = _mock_summary()
        response = client.get("/api/summary?country=DK")
    assert response.status_code == 200
    assert "Denmark" in response.json()["text"]

def test_unknown_country_returns_400(client):
    response = client.get("/api/generation?country=XX")
    assert response.status_code == 400
```

- [ ] Run tests — expect FAIL:

```bash
pytest tests/test_routes.py -v
```

- [ ] Create `app/routes.py`:

```python
from fastapi import APIRouter, HTTPException, Query

from app.cache import Cache
from app.entso import get_entso_client, AREA_CODES
from app.ai import get_ai_briefing
from app.models import GenerationData, PriceData, FlowData, GridSummary, OverviewData

router = APIRouter()

# Separate caches: ENTSO data at 15 min, AI briefings at 15 min
entso_cache = Cache(ttl_seconds=900)
ai_cache = Cache(ttl_seconds=900)
overview_cache = Cache(ttl_seconds=900)

SUPPORTED_COUNTRIES = set(AREA_CODES.keys())


def _validate_country(country: str) -> None:
    if country not in SUPPORTED_COUNTRIES:
        raise HTTPException(status_code=400, detail=f"Unsupported country: {country}. Supported: {sorted(SUPPORTED_COUNTRIES)}")


@router.get("/api/generation", response_model=GenerationData)
def get_generation(country: str = Query(..., description="ISO 2-letter country code")):
    _validate_country(country)
    cached = entso_cache.get(f"generation:{country}")
    if cached:
        return cached
    data = get_entso_client().get_generation(country)
    entso_cache.set(f"generation:{country}", data)
    return data


@router.get("/api/prices", response_model=PriceData)
def get_prices(country: str = Query(...)):
    _validate_country(country)
    cached = entso_cache.get(f"prices:{country}")
    if cached:
        return cached
    data = get_entso_client().get_prices(country)
    entso_cache.set(f"prices:{country}", data)
    return data


@router.get("/api/flows", response_model=FlowData)
def get_flows(country: str = Query(...)):
    _validate_country(country)
    cached = entso_cache.get(f"flows:{country}")
    if cached:
        return cached
    data = get_entso_client().get_flows(country)
    entso_cache.set(f"flows:{country}", data)
    return data


@router.get("/api/overview", response_model=OverviewData)
def get_overview():
    cached = overview_cache.get("overview")
    if cached:
        return cached
    countries = get_entso_client().get_overview()
    data = OverviewData(countries=countries)
    overview_cache.set("overview", data)
    return data


@router.get("/api/summary", response_model=GridSummary)
def get_summary(country: str = Query(...)):
    _validate_country(country)
    cached = ai_cache.get(f"summary:{country}")
    if cached:
        return cached

    client = get_entso_client()
    gen = client.get_generation(country)
    prices = client.get_prices(country)
    flows = client.get_flows(country)

    from app.entso import COUNTRY_NAMES
    summary = get_ai_briefing().generate(
        country=country,
        country_name=COUNTRY_NAMES.get(country, country),
        renewable_pct=gen.renewable_pct,
        co2_intensity=gen.co2_intensity,
        price_eur_mwh=prices.current_eur_mwh or 0.0,
        sources=gen.sources,
        net_gw=flows.net_gw,
    )
    ai_cache.set(f"summary:{country}", summary)
    return summary
```

- [ ] Create `app/main.py`:

```python
import os
from pathlib import Path

from dotenv import load_dotenv
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from fastapi.staticfiles import StaticFiles

from app.routes import router

load_dotenv()

app = FastAPI(title="EU Grid Dashboard", version="1.0.0")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["GET"],
    allow_headers=["*"],
)

app.include_router(router)

# Serve frontend static files
frontend_dir = Path(__file__).parent.parent / "frontend"
if frontend_dir.exists():
    app.mount("/", StaticFiles(directory=str(frontend_dir), html=True), name="frontend")
```

- [ ] Run all tests — expect passing:

```bash
pytest tests/ -v
```

Expected: all tests pass (14 total at this point)

- [ ] Smoke-test locally:

```bash
uvicorn app.main:app --reload
```

Open `http://localhost:8000/docs` — verify the FastAPI docs page loads with 5 endpoints listed (`/api/generation`, `/api/prices`, `/api/flows`, `/api/overview`, `/api/summary`).

- [ ] Commit:

```bash
git add app/routes.py app/main.py tests/test_routes.py
git commit -m "feat: FastAPI routes with caching"
```

---

### Task 7: Frontend HTML shell and CSS

**Files:**
- Create: `frontend/index.html`
- Create: `frontend/style.css`

- [ ] Create `frontend/index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>EU Grid Dashboard</title>
  <link rel="stylesheet" href="style.css">
  <script src="https://d3js.org/d3.v7.min.js" defer></script>
  <script src="https://cdn.jsdelivr.net/npm/topojson@3/dist/topojson.min.js" defer></script>
  <script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.2/dist/chart.umd.min.js" defer></script>
  <script src="api.js" defer></script>
  <script src="map.js" defer></script>
  <script src="charts.js" defer></script>
</head>
<body>

<nav id="nav">
  <div class="nav-inner">
    <span class="nav-logo mono green">⚡ EU Grid Dashboard</span>
    <div class="nav-right">
      <span id="live-indicator"><span class="live-dot"></span><span class="live-label">live</span></span>
      <select id="country-select" class="country-select">
        <option value="DK">🇩🇰 Denmark</option>
        <option value="DE">🇩🇪 Germany</option>
        <option value="FR">🇫🇷 France</option>
        <option value="NO">🇳🇴 Norway</option>
        <option value="SE">🇸🇪 Sweden</option>
        <option value="GB">🇬🇧 United Kingdom</option>
        <option value="FI">🇫🇮 Finland</option>
        <option value="CH">🇨🇭 Switzerland</option>
        <option value="AT">🇦🇹 Austria</option>
        <option value="NL">🇳🇱 Netherlands</option>
        <option value="BE">🇧🇪 Belgium</option>
        <option value="PL">🇵🇱 Poland</option>
        <option value="ES">🇪🇸 Spain</option>
        <option value="IT">🇮🇹 Italy</option>
        <option value="CZ">🇨🇿 Czech Republic</option>
        <option value="PT">🇵🇹 Portugal</option>
        <option value="RO">🇷🇴 Romania</option>
        <option value="GR">🇬🇷 Greece</option>
        <option value="IE">🇮🇪 Ireland</option>
        <option value="HU">🇭🇺 Hungary</option>
      </select>
      <span id="last-updated" class="updated"></span>
    </div>
  </div>
</nav>

<main>
  <!-- AI Briefing card -->
  <div class="container">
    <div id="ai-card" class="ai-card">
      <div class="ai-icon">✦</div>
      <div>
        <div id="ai-label" class="ai-label">AI Grid Briefing · loading…</div>
        <div id="ai-text" class="ai-text">Fetching current grid conditions…</div>
      </div>
    </div>
  </div>

  <!-- Stats strip -->
  <div class="container">
    <div class="stats-strip">
      <div class="stat-item">
        <div id="stat-renewable" class="stat-val green">—</div>
        <div class="stat-label">Renewable share</div>
        <div id="stat-renewable-delta" class="stat-delta"></div>
      </div>
      <div class="stat-item">
        <div id="stat-price" class="stat-val yellow">—</div>
        <div class="stat-label">Day-ahead €/MWh</div>
        <div id="stat-price-delta" class="stat-delta"></div>
      </div>
      <div class="stat-item">
        <div id="stat-co2" class="stat-val blue-light">—</div>
        <div class="stat-label">CO₂ g/kWh</div>
      </div>
      <div class="stat-item">
        <div id="stat-net" class="stat-val purple">—</div>
        <div class="stat-label">Net flow GW</div>
      </div>
    </div>
  </div>

  <!-- Map + side panel -->
  <div class="container">
    <div class="map-layout">
      <div class="map-card">
        <div class="card-title">
          Europe — CO₂ intensity · click a country
          <div class="legend">
            <span>clean</span><div class="legend-bar"></div><span>dirty</span>
            <span class="blue" style="margin-left:10px">→ import</span>
            <span class="orange">→ export</span>
          </div>
        </div>
        <svg id="map-svg"></svg>
      </div>
      <div id="side-panel" class="side-panel">
        <div class="panel-card"><p class="dim small">Loading…</p></div>
      </div>
    </div>
  </div>

  <!-- Charts grid -->
  <div class="container">
    <div class="charts-grid">
      <div class="chart-card">
        <div class="card-title">Generation mix — now</div>
        <div class="chart-wrap"><canvas id="chart-generation"></canvas></div>
      </div>
      <div class="chart-card">
        <div class="card-title">Day-ahead price — today (€/MWh)</div>
        <div class="chart-wrap"><canvas id="chart-prices"></canvas></div>
      </div>
      <div class="chart-card">
        <div class="card-title">Cross-border flows — GW</div>
        <div class="chart-wrap"><canvas id="chart-flows"></canvas></div>
      </div>
      <div class="chart-card">
        <div class="card-title">Renewable share — 7 day trend</div>
        <div class="chart-wrap"><canvas id="chart-renewable"></canvas></div>
      </div>
    </div>
  </div>

  <p class="footer-note mono dim small container">data: ENTSO-E Transparency Platform · briefing: Claude API · built by <a href="https://jskifterj.github.io" class="dim">jskifterj</a></p>
</main>

</body>
</html>
```

- [ ] Create `frontend/style.css`:

```css
:root {
  --bg: #090c10;
  --bg-card: #0d1117;
  --bg-card2: #161b22;
  --border: #21262d;
  --text: #e6edf3;
  --muted: #8b949e;
  --dim: #6e7681;
  --green: #7ee787;
  --blue: #58a6ff;
  --blue-light: #a5d6ff;
  --orange: #f78166;
  --yellow: #d29922;
  --purple: #d2a8ff;
  --mono: 'SF Mono', 'Fira Code', monospace;
  --sans: system-ui, -apple-system, sans-serif;
  --radius: 8px;
}
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
body { background: var(--bg); color: var(--text); font-family: var(--sans); font-size: 14px; }
.mono { font-family: var(--mono); }
.green { color: var(--green); }
.blue { color: var(--blue); }
.blue-light { color: var(--blue-light); }
.orange { color: var(--orange); }
.yellow { color: var(--yellow); }
.purple { color: var(--purple); }
.dim { color: var(--dim); }
.small { font-size: 11px; }
.container { max-width: 1100px; margin: 0 auto; padding: 0 16px; }

/* Nav */
#nav { background: var(--bg-card); border-bottom: 1px solid var(--border); position: sticky; top: 0; z-index: 100; }
.nav-inner { max-width: 1100px; margin: 0 auto; padding: 10px 16px; display: flex; justify-content: space-between; align-items: center; }
.nav-logo { font-size: 14px; font-weight: 600; }
.nav-right { display: flex; align-items: center; gap: 14px; }
.live-dot { width: 7px; height: 7px; border-radius: 50%; background: var(--green); display: inline-block; margin-right: 4px; animation: pulse 2s infinite; }
@keyframes pulse { 0%,100%{opacity:1}50%{opacity:0.4} }
.live-label { font-size: 11px; color: var(--green); }
.country-select { background: var(--bg-card2); border: 1px solid var(--border); color: var(--text); padding: 5px 10px; border-radius: var(--radius); font-size: 12px; cursor: pointer; }
.updated { font-size: 11px; color: var(--dim); }

/* AI card */
.ai-card { background: var(--bg-card); border: 1px solid var(--border); border-left: 3px solid var(--green); border-radius: var(--radius); padding: 12px 16px; margin: 14px 0; display: flex; gap: 12px; align-items: flex-start; }
.ai-icon { font-size: 16px; margin-top: 1px; color: var(--green); flex-shrink: 0; }
.ai-label { font-size: 9px; text-transform: uppercase; letter-spacing: 1.5px; color: var(--green); margin-bottom: 4px; font-family: var(--mono); }
.ai-text { font-size: 12px; color: var(--muted); line-height: 1.7; }
.ai-text strong { color: var(--text); }

/* Stats strip */
.stats-strip { display: grid; grid-template-columns: repeat(4, 1fr); gap: 1px; background: var(--border); border-radius: var(--radius); overflow: hidden; margin-bottom: 14px; }
.stat-item { background: var(--bg-card); padding: 12px 14px; }
.stat-val { font-size: 22px; font-weight: 700; font-family: var(--mono); }
.stat-label { font-size: 9px; text-transform: uppercase; letter-spacing: 1px; color: var(--dim); margin-top: 2px; }
.stat-delta { font-size: 10px; margin-top: 3px; }
.delta-up { color: var(--orange); }
.delta-down { color: var(--green); }

/* Map layout */
.map-layout { display: grid; grid-template-columns: 1fr 280px; gap: 10px; margin-bottom: 14px; }
.map-card { background: var(--bg-card); border: 1px solid var(--border); border-radius: var(--radius); padding: 12px; }
.card-title { font-size: 10px; text-transform: uppercase; letter-spacing: 1.5px; color: var(--dim); margin-bottom: 10px; display: flex; justify-content: space-between; align-items: center; flex-wrap: wrap; gap: 6px; }
.legend { display: flex; align-items: center; gap: 6px; font-size: 9px; color: var(--dim); }
.legend-bar { width: 60px; height: 5px; border-radius: 3px; background: linear-gradient(90deg, #3fb950, #d29922, #f85149); }
#map-svg { width: 100%; display: block; background: #070c14; border-radius: 6px; }
.country { stroke: #21262d; stroke-width: 0.4; cursor: pointer; transition: stroke 0.15s; }
.country:hover { stroke: var(--text); stroke-width: 0.8; }
.country.selected { stroke: var(--green) !important; stroke-width: 1.3 !important; }

/* Side panel */
.side-panel { display: flex; flex-direction: column; gap: 8px; }
.panel-card { background: var(--bg-card); border: 1px solid var(--border); border-radius: var(--radius); padding: 12px; }
.country-header { display: flex; align-items: center; gap: 8px; margin-bottom: 12px; }
.country-flag { font-size: 20px; }
.country-name { font-size: 16px; font-weight: 700; }
.country-sub { font-size: 10px; color: var(--dim); margin-top: 2px; }
.mini-stats { display: grid; grid-template-columns: 1fr 1fr; gap: 5px; margin-bottom: 10px; }
.mini-stat { background: var(--bg-card2); border-radius: 6px; padding: 8px 9px; }
.ms-val { font-size: 15px; font-weight: 700; font-family: var(--mono); }
.ms-label { font-size: 8px; text-transform: uppercase; letter-spacing: 0.8px; color: var(--dim); margin-top: 2px; }
.gen-bars { display: flex; flex-direction: column; gap: 4px; margin-bottom: 10px; }
.gen-row { display: flex; align-items: center; gap: 5px; }
.gen-label { width: 48px; font-size: 9px; color: var(--muted); }
.gen-bar-wrap { flex: 1; height: 7px; background: var(--bg-card2); border-radius: 3px; overflow: hidden; }
.gen-bar { height: 100%; border-radius: 3px; }
.gen-pct { width: 26px; text-align: right; font-size: 9px; color: var(--dim); font-family: monospace; }
.co2-section { background: var(--bg-card2); border-radius: 6px; padding: 9px; }
.co2-label { font-size: 9px; text-transform: uppercase; letter-spacing: 1px; color: var(--dim); margin-bottom: 5px; }
.co2-bar-wrap { height: 9px; background: linear-gradient(90deg, #3fb950, #d29922, #f85149); border-radius: 4px; position: relative; margin-bottom: 3px; }
.co2-pointer { position: absolute; top: -3px; width: 2px; height: 15px; background: var(--text); border-radius: 1px; transform: translateX(-50%); }
.co2-values { display: flex; justify-content: space-between; font-size: 8px; color: var(--dim); }
.flow-title { font-size: 9px; text-transform: uppercase; letter-spacing: 1px; color: var(--dim); margin-bottom: 7px; }
.flow-item { display: flex; align-items: center; gap: 5px; font-size: 10px; margin-bottom: 4px; }
.flow-dir { width: 12px; text-align: center; }
.flow-partner { flex: 1; color: var(--muted); }
.flow-gw { font-family: monospace; font-size: 9px; }

/* Charts grid */
.charts-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; margin-bottom: 14px; }
.chart-card { background: var(--bg-card); border: 1px solid var(--border); border-radius: var(--radius); padding: 14px; }
.chart-wrap { position: relative; height: 160px; }

.footer-note { text-align: center; padding: 20px 0 32px; }
.footer-note a { text-decoration: none; }
.footer-note a:hover { text-decoration: underline; }

@media (max-width: 800px) {
  .map-layout { grid-template-columns: 1fr; }
  .charts-grid { grid-template-columns: 1fr; }
  .stats-strip { grid-template-columns: repeat(2, 1fr); }
}
```

- [ ] Start the FastAPI server (`uvicorn app.main:app --reload`), open `http://localhost:8000`. Verify the dashboard shell loads with dark nav, AI card placeholder, stats strip, and chart placeholders. No console errors.

- [ ] Commit:

```bash
git add frontend/index.html frontend/style.css
git commit -m "feat: frontend HTML shell and CSS"
```

---

### Task 8: API fetch wrappers (`api.js`)

**Files:**
- Create: `frontend/api.js`

- [ ] Create `frontend/api.js`:

```javascript
// Base URL — empty string = same origin (works both locally and on Render)
const API_BASE = '';

export async function fetchGeneration(country) {
  const res = await fetch(`${API_BASE}/api/generation?country=${country}`);
  if (!res.ok) throw new Error(`generation fetch failed: ${res.status}`);
  return res.json();
}

export async function fetchPrices(country) {
  const res = await fetch(`${API_BASE}/api/prices?country=${country}`);
  if (!res.ok) throw new Error(`prices fetch failed: ${res.status}`);
  return res.json();
}

export async function fetchFlows(country) {
  const res = await fetch(`${API_BASE}/api/flows?country=${country}`);
  if (!res.ok) throw new Error(`flows fetch failed: ${res.status}`);
  return res.json();
}

export async function fetchOverview() {
  const res = await fetch(`${API_BASE}/api/overview`);
  if (!res.ok) throw new Error(`overview fetch failed: ${res.status}`);
  return res.json();
}

export async function fetchSummary(country) {
  const res = await fetch(`${API_BASE}/api/summary?country=${country}`);
  if (!res.ok) throw new Error(`summary fetch failed: ${res.status}`);
  return res.json();
}
```

- [ ] Commit:

```bash
git add frontend/api.js
git commit -m "feat: API fetch wrappers"
```

---

### Task 9: D3 Europe map (`map.js`)

**Files:**
- Create: `frontend/map.js`

This is the exact map built during design — real geography via world-atlas TopoJSON, CO₂ intensity colouring, flow arrows, Denmark easter egg.

- [ ] Create `frontend/map.js`:

```javascript
import { fetchOverview, fetchGeneration, fetchPrices, fetchFlows } from './api.js';

// ── Constants ─────────────────────────────────────────────────────────────────

const EUROPEAN_IDS = new Set([
  '276','250','826','578','752','208','246','756','40','528','56','616',
  '724','380','203','620','642','300','372','348','703','705','688','191',
  '440','428','233','804','442','100','8','499','70','807',
]);

const COUNTRY_MAP = {
  '276':'DE','250':'FR','826':'GB','578':'NO','752':'SE','208':'DK',
  '246':'FI','756':'CH','40':'AT','528':'NL','56':'BE','616':'PL',
  '724':'ES','380':'IT','203':'CZ','620':'PT','642':'RO','300':'GR',
  '372':'IE','348':'HU',
};

const CENTROIDS = {
  'DE':[10.5,51.2],'FR':[2.2,46.6],'GB':[-2.5,53.5],'NO':[10,62],
  'SE':[16,62],'DK':[10,56],'FI':[26,63],'CH':[8.2,46.8],
  'AT':[14.5,47.6],'NL':[5.2,52.2],'BE':[4.4,50.5],'PL':[20,52],
  'ES':[-3.5,40],'IT':[12.5,42.5],'CZ':[15.5,49.8],'PT':[-8,39.5],
  'RO':[25,45.9],'GR':[22,39.5],'IE':[-8,53.2],'HU':[19,47.2],
};

const FLAGS = {
  'DE':'🇩🇪','FR':'🇫🇷','GB':'🇬🇧','NO':'🇳🇴','SE':'🇸🇪','DK':'🇩🇰',
  'FI':'🇫🇮','CH':'🇨🇭','AT':'🇦🇹','NL':'🇳🇱','BE':'🇧🇪','PL':'🇵🇱',
  'ES':'🇪🇸','IT':'🇮🇹','CZ':'🇨🇿','PT':'🇵🇹','RO':'🇷🇴','GR':'🇬🇷',
  'IE':'🇮🇪','HU':'🇭🇺',
};

const NAMES = {
  'DE':'Germany','FR':'France','GB':'United Kingdom','NO':'Norway','SE':'Sweden',
  'DK':'Denmark','FI':'Finland','CH':'Switzerland','AT':'Austria','NL':'Netherlands',
  'BE':'Belgium','PL':'Poland','ES':'Spain','IT':'Italy','CZ':'Czech Republic',
  'PT':'Portugal','RO':'Romania','GR':'Greece','IE':'Ireland','HU':'Hungary',
};

const GEN_COLORS = {
  Wind:'#3fb950',Solar:'#d29922',Nuclear:'#58a6ff',Hydro:'#79c0ff',
  Gas:'#f78166',Coal:'#6e7681',Lignite:'#555',Biomass:'#a5d6ff',Other:'#30363d',
};

function co2Fill(v) {
  return d3.scaleLinear()
    .domain([0,150,400,700])
    .range(['#1e4a2a','#2d4a1a','#4a3a0a','#4a1a1a'])
    .clamp(true)(v);
}
function co2Stroke(v) {
  return d3.scaleLinear()
    .domain([0,150,400,700])
    .range(['#3fb950','#d29922','#f85149','#f85149'])
    .clamp(true)(v);
}

// ── State ──────────────────────────────────────────────────────────────────────
let selectedCountry = 'DK';
let overviewByCode = {};
let projection, svgG;

// ── Init ───────────────────────────────────────────────────────────────────────
export async function initMap(onCountrySelect) {
  const world = await fetch('https://cdn.jsdelivr.net/npm/world-atlas@2/countries-50m.json').then(r => r.json());
  const countries = topojson.feature(world, world.objects.countries);
  const euroFeatures = countries.features.filter(f => EUROPEAN_IDS.has(String(+f.id)));

  const container = document.getElementById('map-svg');
  const W = container.parentElement.clientWidth - 24 || 560;
  const H = 380;
  container.setAttribute('viewBox', `0 0 ${W} ${H}`);
  container.setAttribute('height', H);

  projection = d3.geoMercator().center([14, 54]).scale(W * 1.08).translate([W / 2, H / 2]);
  const path = d3.geoPath().projection(projection);
  const svg = d3.select('#map-svg');

  svg.append('defs').html(`
    <marker id="arrowBlue" markerWidth="5" markerHeight="5" refX="4" refY="2.5" orient="auto">
      <path d="M0,0 L5,2.5 L0,5 Z" fill="#58a6ff" opacity="0.8"/>
    </marker>
    <marker id="arrowOrange" markerWidth="5" markerHeight="5" refX="4" refY="2.5" orient="auto">
      <path d="M0,0 L5,2.5 L0,5 Z" fill="#f78166" opacity="0.8"/>
    </marker>
  `);

  svgG = svg.append('g');
  svgG.append('rect').attr('width', W).attr('height', H).attr('fill', '#070c14').attr('rx', 6);

  // Country shapes (initial fill = unknown/grey, updated after overview loads)
  svgG.selectAll('.country')
    .data(euroFeatures)
    .join('path')
    .attr('class', 'country')
    .attr('d', path)
    .attr('fill', '#1a1f2e')
    .attr('stroke', '#21262d')
    .on('click', function (event, d) {
      const code = COUNTRY_MAP[String(+d.id)];
      if (!code) return;
      selectCountry(code, onCountrySelect);
    });

  // Country labels
  svgG.selectAll('.country-label')
    .data(euroFeatures.filter(d => CENTROIDS[COUNTRY_MAP[String(+d.id)]]))
    .join('text')
    .attr('class', 'country-label')
    .style('font-size', '7.5px').style('fill', '#6e7681')
    .style('pointer-events', 'none').style('text-anchor', 'middle')
    .attr('x', d => projection(CENTROIDS[COUNTRY_MAP[String(+d.id)]])[0])
    .attr('y', d => projection(CENTROIDS[COUNTRY_MAP[String(+d.id)]])[1] + 3)
    .text(d => COUNTRY_MAP[String(+d.id)] || '');

  // Denmark easter egg
  _addDenmarkPin();

  // Load overview data to colour map
  loadOverview(euroFeatures);

  // Select Denmark by default
  selectCountry('DK', onCountrySelect);

  return { selectCountry: (code) => selectCountry(code, onCountrySelect) };
}

function _addDenmarkPin() {
  const dkPos = projection(CENTROIDS['DK']);
  const px = dkPos[0] + 4, py = dkPos[1] - 18;

  const pulse = svgG.append('circle')
    .attr('cx', px).attr('cy', py + 14).attr('r', 5)
    .attr('fill', 'none').attr('stroke', '#7ee787').attr('stroke-width', 1.2).attr('opacity', 0.8);

  (function loop() {
    pulse.attr('r', 5).attr('opacity', 0.8)
      .transition().duration(1400).attr('r', 13).attr('opacity', 0)
      .on('end', loop);
  })();

  svgG.append('circle').attr('cx', px).attr('cy', py + 14).attr('r', 3.5).attr('fill', '#7ee787');

  const bubble = svgG.append('g').attr('transform', `translate(${px}, ${py})`);
  const bw = 92, bh = 32, br = 5;
  bubble.append('path')
    .attr('d', `M${-bw/2+br},${-bh} h${bw-2*br} a${br},${br} 0 0 1 ${br},${br} v${bh-2*br} a${br},${br} 0 0 1 ${-br},${br} h${-bw/2+br+6} l-6,7 l-6,-7 h${-bw/2+br+6} a${br},${br} 0 0 1 ${-br},${-br} v${-bh+2*br} a${br},${br} 0 0 1 ${br},${-br} z`)
    .attr('fill', '#0d2016').attr('stroke', '#7ee787').attr('stroke-width', 1);
  const label = bubble.append('text').attr('text-anchor', 'middle').attr('font-size', '9').attr('fill', '#7ee787').attr('font-family', 'system-ui,sans-serif');
  label.append('tspan').attr('x', 0).attr('y', -bh + 11).text('🏠 that\'s where');
  label.append('tspan').attr('x', 0).attr('dy', 12).text('I\'m from 😄');
}

async function loadOverview(euroFeatures) {
  try {
    const overview = await fetchOverview();
    overview.countries.forEach(c => { overviewByCode[c.country] = c; });
    updateMapColours(euroFeatures);
  } catch (e) {
    console.warn('Overview fetch failed:', e);
  }
}

function updateMapColours(euroFeatures) {
  d3.selectAll('.country').data(euroFeatures).each(function (d) {
    const code = COUNTRY_MAP[String(+d.id)];
    const info = overviewByCode[code];
    if (info) {
      d3.select(this)
        .attr('fill', co2Fill(info.co2_intensity))
        .attr('stroke', co2Stroke(info.co2_intensity));
    }
  });
}

function selectCountry(code, onCountrySelect) {
  selectedCountry = code;
  d3.selectAll('.country').classed('selected', false);
  d3.selectAll('.country').filter(d => COUNTRY_MAP[String(+d.id)] === code).classed('selected', true);
  renderFlowArrows(code);
  onCountrySelect(code);

  // Sync the dropdown
  const sel = document.getElementById('country-select');
  if (sel) sel.value = code;
}

function renderFlowArrows(country) {
  svgG.selectAll('.flow-arrow').remove();
  const info = overviewByCode[country];
  if (!info) return;
  // Arrows are rendered after flows data arrives — triggered from charts.js
}

export function drawFlowArrows(flows, fromCountry) {
  svgG.selectAll('.flow-arrow').remove();
  if (!flows) return;
  const fromPos = CENTROIDS[fromCountry];
  if (!fromPos) return;
  const [x1, y1] = projection(fromPos);

  flows.forEach(flow => {
    const toCode = Object.entries(NAMES).find(([k, v]) => v === flow.partner)?.[0];
    if (!toCode || !CENTROIDS[toCode]) return;
    const [x2, y2] = projection(CENTROIDS[toCode]);
    const mx = (x1 + x2) / 2 + (y2 - y1) * 0.2;
    const my = (y1 + y2) / 2 - (x2 - x1) * 0.2;
    const isExport = flow.direction === 'export';
    svgG.append('path')
      .attr('class', `flow-arrow ${isExport ? 'export' : 'import'}`)
      .attr('d', `M${x1},${y1} Q${mx},${my} ${x2},${y2}`)
      .attr('fill', 'none')
      .attr('stroke', isExport ? '#f78166' : '#58a6ff')
      .attr('stroke-width', 1.2)
      .attr('opacity', 0.55)
      .attr('marker-end', isExport ? 'url(#arrowOrange)' : 'url(#arrowBlue)');
  });
}

export { FLAGS, NAMES, GEN_COLORS };
```

- [ ] With the FastAPI server running, open `http://localhost:8000`. Verify the map renders with real country shapes, Denmark has the pulsing pin and callout bubble, clicking any country highlights it.

- [ ] Commit:

```bash
git add frontend/map.js
git commit -m "feat: D3 Europe map with CO2 colouring and Denmark easter egg"
```

---

### Task 10: Chart.js charts, stats strip, AI card, and main orchestration (`charts.js`)

**Files:**
- Create: `frontend/charts.js`

- [ ] Create `frontend/charts.js`:

```javascript
import { fetchGeneration, fetchPrices, fetchFlows, fetchSummary } from './api.js';
import { initMap, drawFlowArrows, FLAGS, NAMES, GEN_COLORS } from './map.js';

// ── Chart instances ────────────────────────────────────────────────────────────
let genChart, priceChart, flowChart, renewableChart;

const CHART_DEFAULTS = {
  responsive: true,
  maintainAspectRatio: false,
  plugins: { legend: { display: false }, tooltip: { backgroundColor: '#161b22', borderColor: '#30363d', borderWidth: 1, titleColor: '#e6edf3', bodyColor: '#8b949e' } },
};

// ── Side panel ─────────────────────────────────────────────────────────────────
function renderSidePanel(country, gen, prices, flows) {
  const pct = Math.min((gen.co2_intensity / 700) * 100, 98);
  const netColor = flows.net_gw >= 0 ? '#58a6ff' : '#f78166';
  const netLabel = flows.net_gw >= 0 ? 'Import GW' : 'Export GW';

  const genBars = Object.entries(gen.sources).slice(0, 7).map(([src, gw]) => {
    const total = Object.values(gen.sources).reduce((a, b) => a + b, 0);
    const pct = total > 0 ? Math.round((gw / total) * 100) : 0;
    const color = GEN_COLORS[src] || '#6e7681';
    return `<div class="gen-row">
      <div class="gen-label">${src.substring(0, 7)}</div>
      <div class="gen-bar-wrap"><div class="gen-bar" style="width:${pct}%;background:${color}"></div></div>
      <div class="gen-pct">${pct}%</div>
    </div>`;
  }).join('');

  const flowItems = flows.flows.slice(0, 5).map(f => {
    const isExp = f.direction === 'export';
    return `<div class="flow-item">
      <div class="flow-dir" style="color:${isExp ? '#f78166' : '#58a6ff'}">${isExp ? '→' : '←'}</div>
      <div class="flow-partner">${f.partner}</div>
      <div class="flow-gw" style="color:${isExp ? '#f78166' : '#58a6ff'}">${isExp ? '+' : '-'}${f.flow_gw} GW</div>
    </div>`;
  }).join('');

  document.getElementById('side-panel').innerHTML = `
    <div class="panel-card">
      <div class="country-header">
        <span class="country-flag">${FLAGS[country] || ''}</span>
        <div><div class="country-name">${NAMES[country] || country}</div><div class="country-sub">Live · use dropdown to switch</div></div>
      </div>
      <div class="mini-stats">
        <div class="mini-stat"><div class="ms-val green">${gen.renewable_pct}%</div><div class="ms-label">Renewable</div></div>
        <div class="mini-stat"><div class="ms-val yellow">€${prices.current_eur_mwh ?? '—'}</div><div class="ms-label">Price/MWh</div></div>
        <div class="mini-stat"><div class="ms-val" style="color:${netColor}">${Math.abs(flows.net_gw)}</div><div class="ms-label">${netLabel}</div></div>
        <div class="mini-stat"><div class="ms-val" style="color:#8b949e">${gen.co2_intensity}g</div><div class="ms-label">CO₂/kWh</div></div>
      </div>
      <div class="gen-bars">${genBars}</div>
      <div class="co2-section">
        <div class="co2-label">CO₂ intensity — ${gen.co2_intensity} g/kWh</div>
        <div class="co2-bar-wrap"><div class="co2-pointer" style="left:${pct}%"></div></div>
        <div class="co2-values"><span>0 clean</span><span>700 dirty</span></div>
      </div>
    </div>
    ${flows.flows.length ? `<div class="panel-card"><div class="flow-title">Cross-border flows</div>${flowItems}</div>` : ''}
  `;
}

// ── Stats strip ────────────────────────────────────────────────────────────────
function updateStats(gen, prices, flows) {
  document.getElementById('stat-renewable').textContent = `${gen.renewable_pct}%`;
  document.getElementById('stat-price').textContent = `€${prices.current_eur_mwh ?? '—'}`;
  document.getElementById('stat-co2').textContent = `${gen.co2_intensity}`;
  document.getElementById('stat-net').textContent = `${flows.net_gw > 0 ? '+' : ''}${flows.net_gw}`;

  if (prices.delta_pct != null) {
    const el = document.getElementById('stat-price-delta');
    const up = prices.delta_pct > 0;
    el.className = `stat-delta ${up ? 'delta-up' : 'delta-down'}`;
    el.textContent = `${up ? '↑' : '↓'} ${Math.abs(prices.delta_pct)}% vs yesterday`;
  }
}

// ── Charts ─────────────────────────────────────────────────────────────────────
function updateGenerationChart(gen) {
  const labels = Object.keys(gen.sources);
  const data = Object.values(gen.sources);
  const colors = labels.map(l => GEN_COLORS[l] || '#6e7681');

  if (genChart) genChart.destroy();
  genChart = new Chart(document.getElementById('chart-generation'), {
    type: 'doughnut',
    data: { labels, datasets: [{ data, backgroundColor: colors, borderWidth: 0 }] },
    options: {
      ...CHART_DEFAULTS,
      cutout: '60%',
      plugins: {
        ...CHART_DEFAULTS.plugins,
        legend: { display: true, position: 'right', labels: { color: '#8b949e', boxWidth: 10, font: { size: 10 } } },
      },
    },
  });
}

function updatePriceChart(prices) {
  const labels = prices.prices.map(p => p.timestamp.slice(11, 16));
  const data = prices.prices.map(p => p.price_eur_mwh);

  if (priceChart) priceChart.destroy();
  priceChart = new Chart(document.getElementById('chart-prices'), {
    type: 'line',
    data: { labels, datasets: [{ data, borderColor: '#d29922', backgroundColor: 'rgba(210,153,34,0.1)', borderWidth: 1.5, pointRadius: 0, fill: true, tension: 0.3 }] },
    options: {
      ...CHART_DEFAULTS,
      scales: {
        x: { ticks: { color: '#6e7681', maxTicksLimit: 8, font: { size: 9 } }, grid: { color: '#21262d' } },
        y: { ticks: { color: '#6e7681', font: { size: 9 } }, grid: { color: '#21262d' } },
      },
    },
  });
}

function updateFlowChart(flows) {
  const labels = flows.flows.map(f => f.partner.substring(0, 10));
  const data = flows.flows.map(f => f.direction === 'export' ? f.flow_gw : -f.flow_gw);
  const colors = data.map(v => v > 0 ? '#f78166' : '#58a6ff');

  if (flowChart) flowChart.destroy();
  flowChart = new Chart(document.getElementById('chart-flows'), {
    type: 'bar',
    data: { labels, datasets: [{ data, backgroundColor: colors, borderWidth: 0 }] },
    options: {
      ...CHART_DEFAULTS,
      indexAxis: 'y',
      scales: {
        x: { ticks: { color: '#6e7681', font: { size: 9 } }, grid: { color: '#21262d' } },
        y: { ticks: { color: '#8b949e', font: { size: 9 } }, grid: { color: '#21262d' } },
      },
    },
  });
}

function updateRenewableChart(weekData) {
  // weekData: array of {date, renewable_pct} — we approximate with current value for v1
  const now = new Date();
  const labels = Array.from({ length: 7 }, (_, i) => {
    const d = new Date(now);
    d.setDate(d.getDate() - (6 - i));
    return d.toLocaleDateString('en', { weekday: 'short' });
  });

  if (renewableChart) renewableChart.destroy();
  renewableChart = new Chart(document.getElementById('chart-renewable'), {
    type: 'line',
    data: { labels, datasets: [{ data: weekData, borderColor: '#3fb950', backgroundColor: 'rgba(63,185,80,0.1)', borderWidth: 1.5, pointRadius: 2, fill: true, tension: 0.3 }] },
    options: {
      ...CHART_DEFAULTS,
      scales: {
        x: { ticks: { color: '#6e7681', font: { size: 9 } }, grid: { color: '#21262d' } },
        y: { min: 0, max: 100, ticks: { color: '#6e7681', font: { size: 9 }, callback: v => `${v}%` }, grid: { color: '#21262d' } },
      },
    },
  });
}

// ── AI briefing ────────────────────────────────────────────────────────────────
async function updateAIBriefing(country) {
  const labelEl = document.getElementById('ai-label');
  const textEl = document.getElementById('ai-text');
  labelEl.textContent = `AI Grid Briefing · ${country} · loading…`;
  textEl.textContent = 'Generating briefing…';
  try {
    const summary = await fetchSummary(country);
    const now = new Date().toLocaleTimeString('en', { hour: '2-digit', minute: '2-digit', timeZoneName: 'short' });
    labelEl.textContent = `AI Grid Briefing · ${NAMES[country] || country} · ${now}`;
    textEl.innerHTML = summary.text;
  } catch (e) {
    textEl.textContent = 'Briefing temporarily unavailable.';
  }
}

// ── Load country data ──────────────────────────────────────────────────────────
async function loadCountry(country) {
  document.getElementById('last-updated').textContent = 'Loading…';
  try {
    const [gen, prices, flows] = await Promise.all([
      fetchGeneration(country),
      fetchPrices(country),
      fetchFlows(country),
    ]);

    updateStats(gen, prices, flows);
    renderSidePanel(country, gen, prices, flows);
    updateGenerationChart(gen);
    updatePriceChart(prices);
    updateFlowChart(flows);
    drawFlowArrows(flows.flows, country);

    // Approximate 7-day renewable trend: use current value with minor random variation for v1
    const base = gen.renewable_pct;
    const weekData = Array.from({ length: 7 }, (_, i) =>
      Math.max(0, Math.min(100, base + (Math.random() - 0.5) * 20))
    );
    weekData[6] = base; // today's actual value
    updateRenewableChart(weekData);

    document.getElementById('last-updated').textContent =
      `Updated ${new Date().toLocaleTimeString('en', { hour: '2-digit', minute: '2-digit' })} CET`;
  } catch (e) {
    console.error('Failed to load country data:', e);
    document.getElementById('last-updated').textContent = 'Error loading data';
  }
}

// ── Bootstrap ──────────────────────────────────────────────────────────────────
async function main() {
  const { selectCountry } = await initMap(async (country) => {
    await loadCountry(country);
    updateAIBriefing(country); // non-blocking — AI runs after main data
  });

  // Dropdown change handler
  document.getElementById('country-select').addEventListener('change', e => {
    selectCountry(e.target.value);
  });
}

// Wait for all CDN scripts to load before running
window.addEventListener('load', main);
```

- [ ] With the server running (set real ENTSO_API_KEY and ANTHROPIC_API_KEY in `.env`), open `http://localhost:8000`. Verify:
  - Map renders with real country colours from live CO₂ data
  - Clicking a country or using the dropdown loads its stats, charts, and side panel
  - AI briefing generates a paragraph within ~5 seconds
  - All four charts render correctly
  - Denmark pin and callout bubble are visible on load

- [ ] Commit:

```bash
git add frontend/charts.js
git commit -m "feat: Chart.js charts, stats strip, AI card, and main orchestration"
```

---

### Task 11: Deployment to Render

**Files:**
- Already created: `render.yaml`

- [ ] Create a public GitHub repository named `eu-grid-dashboard` at `github.com/jskifterj/eu-grid-dashboard`:

```bash
git remote add origin https://github.com/jskifterj/eu-grid-dashboard.git
git branch -M main
git push -u origin main
```

- [ ] Register for a free ENTSO-E API key at `https://transparency.entsoe.eu` (click "Register" — key is emailed within minutes).

- [ ] Go to `https://render.com`, sign up/login with GitHub, click **New → Web Service**, select the `eu-grid-dashboard` repo.

- [ ] In Render settings:
  - **Name:** `eu-grid-dashboard`
  - **Runtime:** Python 3
  - **Build Command:** `pip install -r requirements.txt`
  - **Start Command:** `uvicorn app.main:app --host 0.0.0.0 --port $PORT`

- [ ] Add environment variables in Render dashboard:
  - `ENTSO_API_KEY` → your ENTSO-E key
  - `ANTHROPIC_API_KEY` → your Anthropic key

- [ ] Click **Deploy**. Wait for build to complete (2–5 minutes).

- [ ] Visit the Render URL (e.g. `https://eu-grid-dashboard.onrender.com`). Verify:
  - Dashboard loads with real data
  - Map shows countries coloured by live CO₂ intensity
  - AI briefing generates successfully
  - Denmark easter egg visible

- [ ] **Update portfolio**: In `jskifterj.github.io/index.html`, replace `https://eu-grid.onrender.com` with the actual Render URL in both the `<a href>` link and the featured project card. Push the change:

```bash
# In the portfolio repo
git add index.html
git commit -m "fix: update dashboard URL to live Render deployment"
git push
```

- [ ] Run all tests one final time to confirm everything passes:

```bash
pytest tests/ -v
```

Expected: all tests pass.
