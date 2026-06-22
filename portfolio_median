"""

Created by claude.ai

portfolio_median.py

Calculates the median PLN price for a stock in a given quarter by:
1. Fetching daily price history from Yahoo Finance (via the yfinance library)
2. Fetching daily NBP exchange rates (table A) for the price's currency
3. Joining by date (nearest NBP rate <= price date) and converting to PLN
4. Computing the median

Usage examples:
    python portfolio_median.py --ticker UCG.MI --currency EUR --quarter 25q1
    python portfolio_median.py --ticker UCG.MI --currency EUR --year 2025 --q 1
    python portfolio_median.py --ticker AAPL --currency USD --quarter 24q4

Notes on ticker format (Yahoo Finance convention):
    US tickers: plain symbol, e.g. AAPL, MSFT
    Non-US tickers: <symbol>.<exchange suffix>, e.g.:
        UCG.MI    -> UniCredit, Borsa Italiana (Milan)
        SAP.DE    -> SAP, Xetra (Germany)
        PKN.WA    -> PKN Orlen, Warsaw GPW
    If unsure, search the company on https://finance.yahoo.com/lookup first
    and copy the exact symbol shown (e.g. the URL finance.yahoo.com/quote/UCG.MI
    means the ticker is "UCG.MI").

Notes on currency:
    Use the currency the price is actually quoted in (shown on the Yahoo
    Finance quote page, e.g. "Currency in EUR"). If currency == PLN, no FX
    conversion is performed.
    NBP only publishes rates for trading days; weekends/holidays are skipped
    by NBP, so we backfill using the most recent available rate.

Important caveat:
    yfinance is an unofficial, community-maintained library that scrapes
    Yahoo Finance's public endpoints. It is not a guaranteed/official API,
    and Yahoo's terms restrict this kind of access to personal use. It can
    occasionally break or rate-limit if used heavily. For occasional manual
    runs (the intended use here) this is normally not a problem, but if you
    start seeing repeated failures, check for a `pip install --upgrade
    yfinance` first, since fixes are released frequently.
"""

import argparse
import sys
import re
from datetime import date, timedelta

import pandas as pd
import yfinance as yf
import requests


NBP_URL = "http://api.nbp.pl/api/exchangerates/rates/A/{code}/{d1}/{d2}/?format=json"
HEADERS = {"User-Agent": "Mozilla/5.0 (compatible; portfolio-median-script/1.0)"}


def parse_quarter_string(qstr: str) -> tuple[int, int]:
    """Parse formats like '25q1', '2025q1', '25Q1' -> (2025, 1)."""
    m = re.match(r"^(\d{2}|\d{4})[qQ]([1-4])$", qstr.strip())
    if not m:
        raise ValueError(
            f"Could not parse quarter string '{qstr}'. Expected format like '25q1' or '2025q1'."
        )
    yr_raw, q = m.group(1), int(m.group(2))
    if len(yr_raw) == 4:
        year = int(yr_raw)
    else:
        # 2-digit year: assume 2000s (correct for any realistic brokerage
        # history). Flip this manually if you ever need pre-2000 data.
        year = 2000 + int(yr_raw)
    return year, q


def quarter_date_range(year: int, quarter: int) -> tuple[date, date]:
    start_month = (quarter - 1) * 3 + 1
    start = date(year, start_month, 1)
    if quarter == 4:
        end = date(year, 12, 31)
    else:
        end_month = start_month + 3
        end = date(year, end_month, 1) - timedelta(days=1)
    return start, end


def fetch_yahoo_prices(ticker: str, d1: date, d2: date) -> pd.DataFrame:
    # yfinance's 'end' date is exclusive, so add one day to include d2
    t = yf.Ticker(ticker)
    hist = t.history(start=d1.isoformat(), end=(d2 + timedelta(days=1)).isoformat())

    if hist is None or hist.empty:
        raise ValueError(
            f"Yahoo Finance returned no data for ticker '{ticker}' in range {d1}..{d2}.\n"
            f"  - Check the ticker symbol is correct (search it at https://finance.yahoo.com/lookup).\n"
            f"  - Check the date range actually has trading data for this listing."
        )

    df = hist.reset_index()[["Date", "Close"]].rename(columns={"Close": "price"})
    # Normalize to timezone-naive dates for clean merging later
    df["Date"] = pd.to_datetime(df["Date"]).dt.tz_localize(None)
    return df


def fetch_nbp_rates(currency_code: str, d1: date, d2: date) -> pd.DataFrame:
    currency_code = currency_code.upper()
    if currency_code == "PLN":
        return pd.DataFrame(columns=["Date", "rate"])

    # NBP table A has a max range per request; chunk to be safe.
    all_rows = []
    chunk_start = d1
    while chunk_start <= d2:
        chunk_end = min(d2, chunk_start + timedelta(days=360))
        url = NBP_URL.format(code=currency_code, d1=chunk_start.isoformat(), d2=chunk_end.isoformat())
        resp = requests.get(url, headers=HEADERS, timeout=20)
        if resp.status_code == 404:
            chunk_start = chunk_end + timedelta(days=1)
            continue
        resp.raise_for_status()
        data = resp.json()
        for entry in data.get("rates", []):
            all_rows.append({"Date": entry["effectiveDate"], "rate": entry["mid"]})
        chunk_start = chunk_end + timedelta(days=1)

    if not all_rows:
        raise ValueError(
            f"NBP returned no exchange rate data for {currency_code} in range {d1}..{d2}.\n"
            f"  - Check the currency code is valid (e.g. EUR, USD, GBP, CHF).\n"
            f"  - NBP only covers a limited historical window for some currencies."
        )

    df = pd.DataFrame(all_rows)
    df["Date"] = pd.to_datetime(df["Date"])
    return df.sort_values("Date").reset_index(drop=True)


def merge_nearest_rate(prices: pd.DataFrame, rates: pd.DataFrame) -> pd.DataFrame:
    """For each price date, attach the most recent available NBP rate
    (rates aren't published on weekends/holidays, so we look backwards)."""
    if rates.empty:
        prices = prices.copy()
        prices["rate"] = 1.0
        return prices

    prices = prices.sort_values("Date")
    rates = rates.sort_values("Date")
    merged = pd.merge_asof(prices, rates, on="Date", direction="backward")

    missing = merged["rate"].isna().sum()
    if missing:
        print(
            f"Warning: {missing} price date(s) had no prior NBP rate available "
            f"and were dropped from the calculation.",
            file=sys.stderr,
        )
        merged = merged.dropna(subset=["rate"])

    return merged


def main():
    parser = argparse.ArgumentParser(description="Median PLN price of a stock in a given quarter.")
    parser.add_argument("--ticker", required=True, help="Yahoo Finance ticker, e.g. UCG.MI, AAPL, PKN.WA")
    parser.add_argument("--currency", required=True, help="Currency the price is quoted in, e.g. EUR, USD, PLN")
    parser.add_argument("--quarter", help="Quarter string, e.g. 25q1")
    parser.add_argument("--year", type=int, help="Year, e.g. 2025 (use with --q instead of --quarter)")
    parser.add_argument("--q", type=int, choices=[1, 2, 3, 4], help="Quarter number 1-4")
    parser.add_argument("--isin", help="Optional ISIN, for display/logging only", default=None)
    args = parser.parse_args()

    if args.quarter:
        year, quarter = parse_quarter_string(args.quarter)
    elif args.year and args.q:
        year, quarter = args.year, args.q
    else:
        parser.error("Provide either --quarter (e.g. 25q1) or both --year and --q.")

    d1, d2 = quarter_date_range(year, quarter)
    label = args.isin or args.ticker

    print(f"Fetching prices for {label} ({args.ticker}) from Yahoo Finance, {d1} to {d2}...")
    prices = fetch_yahoo_prices(args.ticker, d1, d2)
    print(f"  -> {len(prices)} trading day(s) found.")

    print(f"Fetching {args.currency.upper()}/PLN rates from NBP, {d1} to {d2}...")
    rates = fetch_nbp_rates(args.currency, d1, d2)
    print(f"  -> {len(rates)} rate day(s) found." if not rates.empty else "  -> currency is PLN, no conversion needed.")

    merged = merge_nearest_rate(prices, rates)
    merged["price_pln"] = merged["price"] * merged["rate"]

    median_pln = merged["price_pln"].median()
    median_native = merged["price"].median()

    print()
    print(f"=== Q{quarter} {year} — {label} ===")
    print(f"Trading days used:        {len(merged)}")
    print(f"Median price ({args.currency.upper()}):     {median_native:.4f}")
    if args.currency.upper() != "PLN":
        print(f"Median price (PLN):        {median_pln:.4f}")
    print()

    # Save detail for inspection
    out_path = f"median_{args.ticker.replace('.', '_')}_{year}q{quarter}.csv"
    merged.to_csv(out_path, index=False)
    print(f"Full daily detail saved to: {out_path}")


if __name__ == "__main__":
    main()

