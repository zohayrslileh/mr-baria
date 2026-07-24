# mr-baria

Trading analysis built on [bmo](#the-engine) — a linear query language whose
engine exposes public market data from Binance, Bybit, OKX, Kraken, Deribit,
DefiLlama, GeckoTerminal and mempool.space as first-class query sources.

This repository currently holds the groundwork: a verified language reference
and a library of working queries to build the trading system on.

## The engine

bmo runs as an HTTP service. Query text is the **raw request body** of
`POST /api/run` — not JSON:

```bash
export BMO_URL="https://<instance>.ngrok-free.app"
curl -X POST "$BMO_URL/api/run" --data-raw '1 + 2'
# {"result":3,"metadata":{"elapsed_ms":0,"peak_memory_bytes":4096,"format":"json"}}
```

The instance serves its own documentation at `/docs` (HTTP interface) and
`/docs/language` (the language), both as Markdown. Extension contracts are
queryable from inside the language:

```
@sources · @stages · @functions
@docs "align", kind: "stage"
@binance "operations"
@binance "describe", operation: "spot.klines"
```

Only public data is reachable — no credentials, accounts, orders or wallets.

## Layout

```
tools/bmo                        CLI wrapper: JSON, typed, SSE, markdown modes
docs/bmo-cheatsheet.md           language reference, with the traps verified
queries/indicators.bmo           EMA · MACD · Wilder RSI(14) · ATR(14)
queries/bollinger.bmo            20-period bands, %B, hourly returns
queries/scanner.bmo              parallel multi-symbol momentum scan
queries/cross-venue-spread.bmo   one instrument across three venues
queries/funding-spread.bmo       annualized perp funding, three venues
queries/beta-vs-btc.bmo          correlation, beta and R² against BTC
queries/strategy-engine.bmo      parameterized backtest, strategies as values
```

## Running a query

`BMO_URL` is read from the environment because ngrok tunnels change address on
every restart.

```bash
export BMO_URL="https://<instance>.ngrok-free.app"

tools/bmo -f queries/scanner.bmo     # run a file
tools/bmo '1h + 30m'                 # inline
tools/bmo -t '10 / 3'                # typed — exact numbers as strings
tools/bmo -s 'wait(5s)'              # stream lifecycle events
tools/bmo -r '@docs "rolling", kind: "stage"'   # markdown contract
```

Every query in `queries/` runs against a live instance and returns rows; they
are meant to be read as worked examples as much as run.

## Notes before writing queries

Four things in the language reliably catch people out. The
[cheatsheet](docs/bmo-cheatsheet.md) covers them in full, with the checks that
confirm each one.

- **`??` binds tighter than arithmetic.** `null ?? 0 + 1` is `1`.
- **Recurrences must be rounded.** Exact Decimal grows about a digit per row,
  so an unrounded EMA fails partway through the table.
- **Regex escapes double up.** The language knows five string escapes, so a
  pattern needs `"\\."`, never `"\."`.
- **There is no truthiness.** `null` means absence; conditions must be
  Boolean, and aggregates fail on null rather than skipping it.
