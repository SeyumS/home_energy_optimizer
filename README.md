# home_energy_optimizer

Schedules a home battery against day-ahead electricity prices and a PV forecast, and
measures how much that saves compared with no battery and with the "store solar surplus"
logic most home batteries ship with.

```
day-ahead prices ─┐
PV forecast      ─┼─► linear program (PuLP) ─► 15-min battery schedule ─► API ─► dashboard
load forecast    ─┘
```

[![CI](../../actions/workflows/ci.yml/badge.svg)](../../actions/workflows/ci.yml)

## Quickstart

Requires [uv](https://docs.astral.sh/uv/).

```bash
cd backend
uv sync                              # add --extra entsoe for the ENTSO-E client
uv run pytest                        # 16 tests: physics, economics, DST, parsers, API

uv run heo run --date 2026-07-15
uv run heo backtest --start 2026-01-01 --end 2026-12-31
uv run uvicorn heo.api:app --reload  # then open http://127.0.0.1:8000/docs
```

Everything runs offline with synthetic data by default. Switch to real data with
`--prices energy-charts` (no key) or `--prices entsoe` (needs `ENTSOE_API_KEY`, see
`backend/.env.example`), and `--weather open-meteo`.

Full machine setup (terminal, tools, editor, GitHub) is in
[docs/dev-setup.md](docs/dev-setup.md); project conventions are in [CLAUDE.md](CLAUDE.md).

## Repository layout

```
backend/     Python package `heo`, tests, uv.lock
frontend/    Next.js dashboard (planned)
docs/        developer setup
.github/     CI, Dependabot, issue and PR templates
```

Every pull request runs Ruff (lint and format), Pyright (types) and pytest in CI.

## How it works

| Module | What it does |
|---|---|
| `data/prices.py` | Day-ahead spot prices from ENTSO-E, Energy-Charts or a synthetic duck curve. Rejects data with gaps. |
| `data/weather.py` | Irradiance on the panel plane from Open-Meteo (`global_tilted_irradiance`). |
| `forecast/pv.py` | Irradiance → PV power, with performance ratio and inverter clipping. |
| `forecast/load.py` | Household demand shaped like the BDEW H0 profile, scaled to annual kWh. |
| `optimize/battery.py` | The linear program. The docstring explains every constraint. |
| `baselines.py` | No-battery and rule-based scenarios, plus cost accounting. |
| `pipeline.py` | Runs all scenarios per day; the backtest carries battery charge across days. |
| `api.py` | FastAPI endpoints `/schedule` and `/backtest` for the frontend. |

All parameters (battery size, kWp, tariff) are in `config.py`.

### Modelling decisions

- **15-minute resolution.** The European day-ahead market moved to 15-minute products in
  October 2025.
- **UTC internally.** Local days are converted to UTC ranges, so daylight-saving days
  correctly have 92 or 100 steps. There's a test for this.
- **Household price, not spot price.** `buy = (spot + fees and levies) × (1 + VAT)`. Fees
  dominate, which flattens the price spread the battery can exploit.
- **Battery wear has a cost** (`degradation_eur_per_kwh`), so the optimizer doesn't cycle
  for a few cents.
- **No grid arbitrage.** Export is limited to PV plus battery discharge; otherwise the
  solver would buy and sell simultaneously during negative prices.

## Results (synthetic data, one year)

| Scenario | Total cost | Saving vs. no battery |
|---|---|---|
| No battery | €120 | – |
| Rule-based battery | −€240 | €360 |
| Optimized battery | −€261 | €381 |

Most of the value comes from having a battery at all; price-aware scheduling adds about
6 % on top. That's a realistic finding: grid fees and taxes make up most of a German
household price, so spot-price swings shrink. **Re-run this with real 2025/26 prices and
put your own numbers here** — that's the headline of the project.

## Roadmap

- [ ] Real data backtest with Energy-Charts prices and Open-Meteo archive weather
- [ ] Next.js dashboard (prices, PV, load, battery charge, savings per scenario)
- [ ] Official BDEW H0 profile instead of the approximation
- [ ] pvlib model with temperature losses; measure forecast error
- [ ] EV charging with a "full by 7:00" constraint
- [ ] Forecast uncertainty: optimize on forecasts, evaluate on actuals
- [ ] Docker Compose with Postgres/TimescaleDB and a daily scheduled job
