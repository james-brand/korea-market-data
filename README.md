# Korean Market Data — daily foreign investor flows and KRX sector indices, in English

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.21833374.svg)](https://doi.org/10.5281/zenodo.21833374)
[![Licence: CC BY 4.0](https://img.shields.io/badge/Licence-CC%20BY%204.0-blue.svg)](LICENSE)

Two datasets covering the Korean stock market (KOSPI and KOSDAQ), rebuilt every trading day
from primary Korean sources and published in English, machine-readable, under CC BY 4.0.

**Why this exists.** Korea is the world's 13th-largest equity market, and the two series below
are published daily by Korean institutions — in Korean, in formats built for domestic use.
Mainstream English-language finance sites do not carry them. Yahoo Finance and TradingView
carry no KRX sector indices at all; the wires publish a single aggregate foreign-flow number
and nothing per-stock. This repository closes that gap.

| | |
|---|---|
| **Coverage** | 2,555 KOSPI/KOSDAQ common stocks · 44 KRX sector indices + 2 benchmarks |
| **Updated** | every trading day, ~02:00 KST |
| **Latest** | 2026-08-14 close |
| **Licence** | [CC BY 4.0](LICENSE) — cite *K-Export Stars (kexportstars.com)* |
| **History** | daily snapshots in [`archive/`](archive/) from 2026-07-31 onward |
| **DOI** | [10.5281/zenodo.21833374](https://doi.org/10.5281/zenodo.21833374) — archived on Zenodo, always resolves to the newest release |

---

## 1. Foreign and institutional investor flows

Daily net buying by investor class for **every KOSPI and KOSDAQ common stock** — not a
top-50 sample. SPACs, ETFs and preferred shares are excluded.

- [`data/foreign-flows-aggregates.csv`](data/foreign-flows-aggregates.csv) — all 2,555 stocks,
  1-day / 5-day / 20-day net flows in shares and value
- [`data/foreign-flows-latest.json`](data/foreign-flows-latest.json) — the latest session's
  top-20 buys and sells by value, in KRW and USD, with metadata

```
ticker,name_en,market,last_close,f_1d_sh,f_5d_sh,f_5d_val,f_20d_sh,f_20d_val,i_1d_sh,i_5d_sh,i_20d_sh
```
`f_` = foreign, `i_` = institution · `_sh` = net shares · `_val` = net value in KRW

*(preceded by a `#` banner line — see [Using it](#using-it))*

**Accuracy check.** On 24 July 2026 the settled market-wide foreign net was reported as
−₩3.2828tn. Summing this dataset over the same session gives **−₩3.2796tn — a 0.1%
difference.** We publish that check because per-stock flow data valued at daily closes is an
approximation of traded value, and you should know how close the approximation lands.

## 2. KRX sector index performance

All 44 Korea Exchange sector indices plus the KOSPI and KOSDAQ benchmarks, with returns over
four windows and each sector's excess return against its own market.

- [`data/korea-sectors-latest.csv`](data/korea-sectors-latest.csv) — one row per index
- [`data/korea-sectors-latest.json`](data/korea-sectors-latest.json) — same, plus a daily
  year-to-date time series for every index, rebased to 100 at the first session of the year

```
market,sector_en,sector_ko,close,return_1d_pct,return_1w_pct,return_1m_pct,return_ytd_pct,excess_1m_vs_market_pp
```

Sector names are given in both English and Korean, because the Korean name is the join key
back to any KRX source.

*(also preceded by a `#` banner line.)*

**What a single index level hides.** In the month to 31 July 2026 the KOSPI fell 20.6%. Over
the same month KOSPI Food, Beverage & Tobacco rose 3.7% and Pharmaceuticals rose 3.1%, while
Electronics & Semiconductors fell 25.6% — and that same Electronics index was still **+105.0%
year to date**. None of that is visible in the headline number.

---

## Using it

Every file is served raw from GitHub, so nothing needs to be downloaded first and no API key
is involved.

> ⚠️ **Both CSVs open with a `#` banner line** carrying the as-of date, licence and source, so
> that attribution travels with the file. **Pass `comment="#"`** — without it `read_csv` does
> not raise, it silently returns a single column.

```python
import pandas as pd

BASE = "https://raw.githubusercontent.com/james-brand/korea-market-data/main/data"

# Who did foreigners buy hardest over the last 20 sessions?
flows = pd.read_csv(f"{BASE}/foreign-flows-aggregates.csv", comment="#")
print(flows.nlargest(10, "f_20d_val")[["ticker", "name_en", "f_20d_val"]])

# Which sectors beat their own market over the last month?
sectors = pd.read_csv(f"{BASE}/korea-sectors-latest.csv", comment="#")
print(sectors.nlargest(10, "excess_1m_vs_market_pp")[
    ["market", "sector_en", "return_1m_pct", "excess_1m_vs_market_pp"]])
```

The JSON files carry the same numbers plus metadata. `korea-sectors-latest.json` additionally
holds a daily year-to-date series for all 46 lines (44 sectors + 2 benchmarks), rebased to 100
at the first session of the year:

```python
import pandas as pd, requests

doc = requests.get(f"{BASE}/korea-sectors-latest.json").json()
print(doc["meta"]["as_of_trading_day"])     # e.g. "2026-08-06"

dates = pd.to_datetime(doc["series"]["dates"])
frame = pd.DataFrame(
    {line["sector_en"]: line["values"] for line in doc["series"]["lines"]},
    index=dates,
)
print(frame.iloc[-1].nlargest(5))           # best year-to-date lines
```

Each entry in `series["lines"]` carries `market`, `sector_ko`, `sector_en`, `is_benchmark` and
`values`. Filter on `is_benchmark` to separate the KOSPI and KOSDAQ lines from the sectors.

For a fixed snapshot rather than the moving latest, read from `archive/<YYYY-MM-DD>/`, or cite
the version DOI of the matching release (see **Licence and citation** below).

---

## Sources and method

| Field | Source |
|---|---|
| Investor net-buy, per stock | Korea Investment & Securities OpenAPI |
| Closes, share counts, market caps, sector indices | [Korea Exchange Open API](http://data.krx.co.kr/) (authenticated) |
| Corporate filings referenced in our writing | [DART](https://opendart.fss.or.kr/), Korea's electronic disclosure system |

Full method, definitions and known limitations: **[docs/METHODOLOGY.md](docs/METHODOLOGY.md)**.

Three things worth knowing before you use the numbers:

1. **Value figures are derived.** Flows arrive as share counts. We value each day's net shares
   at *that day's* settled close and sum. Valuing a whole week at one price instead can shift
   the answer by ~20% in a volatile week — we have made that mistake and corrected it publicly.
2. **Settled data only.** Every figure comes from a completed session. Korean intraday
   snapshots are widely republished as if they were closes; a 9:53 a.m. figure we once quoted
   was **four times smaller** than the settled one.
3. **We do not publish raw redistributions.** These files are derived aggregates and returns,
   not a mirror of the underlying vendor feeds.

## Licence and citation

Released under [CC BY 4.0](LICENSE). Copyright © 2026 K-Export Stars (kexportstars.com).
Use it commercially, redistribute it, build on it — attribution is the only condition:

> Ju, J. (2026). *Korean Market Data: daily foreign investor flows and KRX sector indices*
> [Data set]. K-Export Stars. https://doi.org/10.5281/zenodo.21833374

That DOI is the **concept DOI** — it always resolves to the newest release. To cite the exact
snapshot you used, take the version DOI from that record instead (v1.0.0 is
[10.5281/zenodo.21833375](https://doi.org/10.5281/zenodo.21833375)). Because the data is
rebuilt every trading day, pinning the version matters for anything reproducible.

The licence covers the derived data and documentation published here. It does not extend to
the underlying exchange and vendor feeds these aggregates are computed from; those remain
governed by their own terms.

Live charts, weekly written analysis of these flows, and the download endpoints:
**[kexportstars.com/tools](https://kexportstars.com/tools/)** ·
**[Foreign Flow Watch](https://kexportstars.com/foreign-flow-watch/)**

## Corrections

We log corrections rather than editing silently. Anything we get wrong in the data or in the
writing built on it is disclosed at [kexportstars.com/about](https://kexportstars.com/about/).
If you find an error here, open an issue — we would rather hear it than not.
