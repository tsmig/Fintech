# Portfolio Median Price Calculator

Created by claude.ai

Calculates the median PLN price of a stock for a given quarter, by combining:
- **Yahoo Finance** (via the `yfinance` Python library) — free historical daily price data, no API key
- **NBP** (Narodowy Bank Polski) — free official daily EUR/PLN, USD/PLN, etc. rates, no API key

## Setup

```bash
pip install yfinance pandas requests
```

No API keys, no accounts, no config files.

## Usage

```bash
python portfolio_median.py --ticker UCG.MI --currency EUR --quarter 25q1 --isin IT0005239360
```

or with explicit year/quarter instead of the "25q1" shorthand:

```bash
python portfolio_median.py --ticker UCG.MI --currency EUR --year 2025 --q 1
```

### Output

The script prints:
- Number of trading days found in the quarter
- Median price in the native currency
- Median price converted to PLN (using each day's matching NBP rate, then taking the median of the converted prices — not converting a single average rate)

It also saves a CSV (`median_<ticker>_<year>q<quarter>.csv`) with the full daily detail — date, native price, FX rate used, and PLN price — so you can audit or re-use the data, e.g. in Excel.

## Finding the right Yahoo Finance ticker

Yahoo Finance tickers use `<symbol>.<exchange suffix>` for non-US listings, for example:
- `UCG.MI` → UniCredit, Borsa Italiana (Milan)
- `SAP.DE` → SAP, Xetra (Germany)
- `PKN.WA` → PKN Orlen, Warsaw GPW
- `AAPL` → Apple, US exchanges (no suffix needed)

If a ticker doesn't work, search the company at **finance.yahoo.com/lookup**, open its quote page, and copy the exact symbol from the URL (e.g. `finance.yahoo.com/quote/UCG.MI` → ticker is `UCG.MI`). The quote page also shows the quoted currency (e.g. "Currency in EUR") — use that for `--currency`.

## Notes on accuracy

- **FX conversion is done per-day, not on the average rate.** Each daily price is converted to PLN using that day's NBP rate, and *then* the median is taken across all PLN-converted values. This is more accurate than taking the median price first and converting once at the end.
- **NBP doesn't publish rates on weekends/bank holidays.** The script automatically uses the most recent available rate (looking backward) for any day NBP didn't publish one — this is standard practice and matches how most brokers/accountants handle it.
- **"Median" here means the statistical median** (middle value), not the average. If you want the average instead, open the saved CSV and use `=AVERAGE()` on the `price_pln` column in Excel — or ask for a script variant that prints both.
- **Yahoo Finance prices via `yfinance` are typically delayed** (15–20 minutes during market hours) and reflect Yahoo's own adjustments (e.g. for splits). For historical quarterly medians this delay is irrelevant, but it's worth knowing if you ever check today's live price against your broker.

## A word on reliability (read this before relying on it for anything important)

`yfinance` is **not an official Yahoo API** — it's a community-maintained open-source library that uses Yahoo's publicly reachable but undocumented endpoints. This means:

- It's generally reliable for occasional, low-volume use (your stated use case)
- It **can occasionally break** when Yahoo changes its website/backend (this has happened before, e.g. in a 2025 redesign) — if a previously-working call suddenly fails, try `pip install --upgrade yfinance` first, since the maintainers usually patch quickly
- Yahoo's own terms restrict this kind of data access to personal, non-commercial use
- If you ever need guaranteed uptime or commercial-grade reliability, you'd need a paid, licensed data provider instead (e.g. Alpha Vantage, EODHD, Polygon.io)

For tracking your own portfolio occasionally, this is a normal and widely-used tradeoff — just don't be surprised if it needs an occasional `pip install --upgrade yfinance` down the line.

## Troubleshooting

| Problem | Likely cause |
|---|---|
| `Yahoo Finance returned no data for ticker '...'` | Wrong ticker symbol — verify on finance.yahoo.com directly |
| `NBP returned no exchange rate data for ...` | Wrong/unsupported currency code, or date range outside NBP's coverage |
| `Failed to get ticker... HTTP Error 403/429` | Temporary rate-limiting from Yahoo — wait a bit and retry, or upgrade yfinance |
| Network/connection errors | Check your internet connection; no proxy/VPN should block Yahoo Finance or api.nbp.pl |
