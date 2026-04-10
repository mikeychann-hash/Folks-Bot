# WeatherBet — Polymarket Weather Trading Bot

Automated weather market trading bot for Polymarket. Finds mispriced temperature outcomes using real forecast data from multiple sources across 20 cities worldwide.

No SDK. No black box. Pure Python.

---

## The two bots

This repo ships two self-contained bots. They both live at the top level and share the same `config.json`, but they are separate programs with different goals.

### `bot_v1.py` — Base Bot (learning / reference)

The smallest thing that works. Scans six US cities, fetches forecasts from NWS, finds the matching temperature bucket on Polymarket, and enters when the market price is below a flat entry threshold. Exits when price crosses a flat exit threshold.

- Six US cities (NYC, Chicago, Miami, Dallas, Seattle, Atlanta)
- One forecast source: NWS hourly + NWS station observations
- Flat 5%-of-balance position sizing
- Flat entry / exit thresholds
- Local `simulation.json` as the paper-trading ledger
- CLI flags: `--live`, `--positions`, `--reset`

**Read this first if you're learning how the bot works.** It's short enough to hold in your head — no EV math, no Kelly, no stops, no calibration.

### `bot_v2.py` — Full Bot (the one you actually run)

Everything in v1, plus a complete trading pipeline designed to run 24/7.

- **20 cities across 4 continents** — US, Europe, Asia, South America, Oceania
- **3 forecast sources** — ECMWF via Open-Meteo (global), HRRR/GFS via Open-Meteo (US hourly), METAR (real-time station observations), combined into a "best" forecast per market
- **Expected Value filter** — skips any trade where `EV < MIN_EV`, computed from the calibrated normal-CDF probability of the forecast landing in the bucket
- **Fractional Kelly sizing** — positions scaled by edge strength, capped at `MAX_BET`
- **Slippage filter** — refetches real bestBid / bestAsk before entry, skips if spread > `MAX_SLIPPAGE`
- **Stop-loss + trailing stop** — 20% stop, moves to breakeven on +20% gain
- **Time-based take-profit** — sell at $0.85 with 24–48h left, $0.75 with >48h left, hold to resolution with <24h left
- **Forecast-changed exit** — close the position if the forecast moves ≥2°F (1°C) beyond the bucket edge
- **Self-calibration** — learns each city's forecast MAE per source (ECMWF / HRRR / METAR) from closed markets and uses it as the σ in the EV calculation
- **Atomic writes** — `state.json`, per-market files, and `calibration.json` are written via `os.replace` so a Ctrl+C mid-write cannot corrupt them
- **Balance reconciliation** — the per-market JSON files are the source of truth; every scan and every monitor pass recomputes `balance`, `wins`, `losses`, and `peak_balance` from the ledger and logs any drift
- **CLI commands**: `run` (default), `status`, `report`, `reconcile`

### Side-by-side

|                              | `bot_v1.py`                  | `bot_v2.py`                                                |
| ---------------------------- | ---------------------------- | ---------------------------------------------------------- |
| Cities                       | 6 (US only)                  | 20 (US, EU, Asia, SA, Oceania)                             |
| Forecast sources             | NWS (US only)                | ECMWF + HRRR/GFS (US) + METAR, blended                     |
| Sizing                       | Flat 5% of balance           | Fractional Kelly, capped at `MAX_BET`                      |
| Entry rule                   | Price < `ENTRY_THRESHOLD`    | EV ≥ `MIN_EV` and spread ≤ `MAX_SLIPPAGE` and volume ≥ `MIN_VOLUME` |
| Exit rule                    | Price ≥ `EXIT_THRESHOLD`     | Stop-loss, trailing stop, time-based take-profit, forecast-changed, auto-resolve |
| Probability model            | —                            | `bucket_prob` — normal CDF integrated across the bucket    |
| Calibration                  | —                            | Per-city, per-source MAE from closed markets               |
| Ledger                       | `simulation.json`            | `data/state.json` + one `data/markets/{city}_{date}.json` per market |
| Crash-safe writes            | No                           | Yes (atomic `os.replace`)                                  |
| Drift detection              | No                           | Reconciles from ledger every scan                          |
| Runs as                      | One-shot script              | Continuous loop (scan every `SCAN_INTERVAL`, monitor every 10 min) |
| Loop cadence                 | `python bot_v1.py` per run   | `python bot_v2.py` keeps running                           |
| Purpose                      | Understand the idea          | Trade the strategy                                         |

### Which should I run?

- **`bot_v2.py`**, for any real use. All the risk controls and all the bug fixes live here.
- **`bot_v1.py`** is kept as a reference implementation so you can read 450 lines and understand the core scan → find bucket → check price → enter pattern without being buried in Kelly, calibration, stops, and reconciliation.

---

## How it works

Polymarket runs markets like _"Will the highest temperature in Chicago be between 46–47°F on March 7?"_ These markets are frequently mispriced — the forecast says 37% likely but the market is trading at 8 cents. The bot finds those gaps.

Each scan cycle (hourly by default):

1. For each city, generate the next 4 dates in that city's **local** timezone (important — Polymarket slugs and Open-Meteo output are both keyed by local date)
2. Fetch ECMWF and HRRR forecasts from Open-Meteo and a METAR station observation
3. Pick the "best" forecast (HRRR for US D+0/D+1, ECMWF otherwise)
4. Query Polymarket Gamma for the matching event and its bucket markets
5. Find the bucket that contains the forecast temperature
6. Compute `p = bucket_prob(forecast, t_low, t_high, sigma)` — the probability that the actual will land in the bucket, integrated across the bucket using each city's calibrated σ
7. Compute `EV` and `Kelly`, size the position, refetch real bestAsk, and enter if everything passes
8. Save a snapshot of forecast + market price to the per-market JSON file

Every 10 minutes between full scans, a lightweight `monitor_positions` pass checks stop-loss, trailing stop, and time-based take-profit on every open position.

After the city loop, an auto-resolution pass queries Polymarket for each open position's market ID, detects closed markets, and — if `VC_KEY` is set — pulls the actual temperature from Visual Crossing to both populate `actual_temp` (which the calibration loop needs) and serve as the authoritative win/loss source.

---

## Why airport coordinates matter

Most bots use city center coordinates. That's wrong.

Every Polymarket weather market resolves on a specific airport station. NYC resolves on LaGuardia (KLGA), Dallas on Love Field (KDAL) — not DFW. The difference between city center and airport can be 3–8°F. On markets with 1–2°F buckets, that's the difference between the right trade and a guaranteed loss.

| City         | Station | Airport             |
|--------------|---------|---------------------|
| NYC          | KLGA    | LaGuardia           |
| Chicago      | KORD    | O'Hare              |
| Miami        | KMIA    | Miami Intl          |
| Dallas       | KDAL    | Love Field          |
| Seattle      | KSEA    | Sea-Tac             |
| Atlanta      | KATL    | Hartsfield          |
| London       | EGLC    | London City         |
| Paris        | LFPG    | Charles de Gaulle   |
| Munich       | EDDM    | Munich              |
| Tokyo        | RJTT    | Haneda              |
| Seoul        | RKSI    | Incheon             |
| Shanghai     | ZSPD    | Pudong              |
| Singapore    | WSSS    | Changi              |
| Toronto      | CYYZ    | Pearson             |
| Sao Paulo    | SBGR    | Guarulhos           |
| Buenos Aires | SAEZ    | Ezeiza              |
| Wellington   | NZWN    | Wellington          |
| …            | …       | …                   |

See the `LOCATIONS` dict at the top of `bot_v2.py` for the full list and exact coordinates.

---

## Installation

```bash
git clone <this-repo>
cd Folks-Bot
pip install requests
```

Python 3.9+ is required (for `zoneinfo`).

Edit `config.json` in the project folder:

```json
{
  "balance": 10000.0,
  "max_bet": 20.0,
  "min_ev": 0.10,
  "max_price": 0.45,
  "min_volume": 500,
  "min_hours": 2.0,
  "max_hours": 72.0,
  "kelly_fraction": 0.25,
  "max_slippage": 0.03,
  "scan_interval": 3600,
  "calibration_min": 30,
  "vc_key": "YOUR_KEY_HERE"
}
```

Optional: get a free Visual Crossing API key at visualcrossing.com. It's used to fetch actual temperatures after market resolution, which populates `actual_temp` on each closed market file and drives the self-calibration loop. If omitted, the bot falls back to Polymarket's outcome prices for win/loss and calibration won't run.

`config.json` is checked into the repo with a placeholder VC key. If you put a real key in it, the `.gitignore` covers `data/` and runtime state — but **do not commit a real VC key**. The typical workflow is to edit `config.json` locally and never push it.

---

## Usage

### Full bot (`bot_v2.py`)

```bash
python bot_v2.py              # run the main loop (Ctrl+C to stop)
python bot_v2.py status       # balance and currently-open positions
python bot_v2.py report       # full breakdown of every closed position
python bot_v2.py reconcile    # rewrite state.json from the market ledger
```

`reconcile` is a safety net. If you ever suspect `state.json` has drifted from the per-market files (after an unclean shutdown, a crash mid-write from an older build, or if you're migrating from a buggy version), run it once before restarting the main loop. It reads every `data/markets/*.json`, recomputes `balance`, `wins`, `losses`, `total_trades`, and `peak_balance` from scratch, and writes them back. The main loop also runs a lighter reconcile at the end of every scan and every monitor pass and logs any drift loudly.

### Base bot (`bot_v1.py`)

```bash
python bot_v1.py              # paper-mode scan (no trades executed)
python bot_v1.py --live       # execute trades against the simulation ledger
python bot_v1.py --positions  # show open positions
python bot_v1.py --reset      # wipe the simulation ledger and start fresh at $1,000
```

v1 is one-shot. It doesn't loop — run it by hand or put it in cron. v1 uses `simulation.json` as its ledger; it does not share state with v2.

---

## Data storage (v2)

All runtime state lives under `data/` (gitignored):

```
data/
├── state.json                 # balance, wins, losses, starting_balance, peak
├── calibration.json           # per-city per-source sigma, updated from ledger
└── markets/
    ├── nyc_2024-03-07.json    # one file per (city, date)
    ├── chicago_2024-03-07.json
    └── …
```

Each market file contains:

- Hourly forecast snapshots from ECMWF, HRRR, and METAR — with the "best" source picked each hour
- Market price snapshots (top bucket and price)
- All observed buckets + bids/asks at scan time
- Position details (entry, stop, PnL, close reason)
- `actual_temp` (when VC_KEY is configured) and `resolved_outcome`

The market files are the single source of truth. `state.json` is a cache that gets reconciled against them.

### What counts as a "close"?

A position can exit via any of these paths, and **every one of them counts as a win or loss in the reports**:

| Close reason       | Triggered by                                                         |
| ------------------ | -------------------------------------------------------------------- |
| `stop_loss`        | Current bid ≤ 80% of entry                                           |
| `trailing_stop`    | Trailing stop was moved to breakeven and price came back to entry    |
| `take_profit`      | `monitor_positions` saw price ≥ take-profit threshold for that horizon |
| `forecast_changed` | Forecast moved ≥ 2°F (1°C) beyond the bucket edge                    |
| `resolved`         | Market closed on Polymarket (auto-resolution pass)                   |

Positive/zero PnL is a win; negative PnL is a loss.

---

## APIs used

| API                   | Auth     | Purpose                                   |
| --------------------- | -------- | ----------------------------------------- |
| Open-Meteo            | None     | ECMWF + HRRR/GFS forecasts                |
| Aviation Weather      | None     | METAR station observations (real-time)    |
| Polymarket Gamma      | None     | Event discovery, market prices, bestBid/bestAsk, resolution status |
| Visual Crossing       | Free key | Historical actual temperatures for resolution and calibration |
| NWS (v1 only)         | None     | Hourly forecast + station observations    |

---

## Disclaimer

This is not financial advice. Prediction markets carry real risk. Run the paper-trading simulation thoroughly — and pay attention to the calibration output — before committing real capital.
