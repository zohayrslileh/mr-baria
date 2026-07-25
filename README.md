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
queries/bybit-perp-scan.bmo      Bybit USDT perp screen, ATR-sized
queries/factor-validation.bmo    do the screen factors predict anything?
queries/factor-robustness.bmo    same factors, split by time
queries/cross-sectional-factor.bmo  the one factor that survived
docs/trading-notes.md            what the screen survived, and what it did not
docs/bybit-connect.md            OAuth AI sub-account setup, and the geo-block
tools/bybit-api                  signed Bybit V5 client, dry-run by default
tools/snapshot                   append the screen to the out-of-sample journal
tools/forward-test               score that journal against what happened
tools/bmo-serve                  run the engine locally instead of over a tunnel
```

## Running the engine locally

bmo ships as a single 19 MB static-ish ELF binary (only libc, libm, libgcc).
Running it here removes the dependency on a tunnel staying up:

```bash
cp /path/to/bmo tools/bin/bmo && chmod +x tools/bin/bmo
sha256sum tools/bin/bmo
# c7d7e16e698c80fc5ff9475e77faf29ea8a5eec8e26a039ed36a9f3f7fa0c059   v0.1.6

tools/bmo-serve start
export BMO_URL="http://127.0.0.1:8080"
```

The binary is not tracked — `tools/bin/.gitignore` keeps it out.

**One source is unreachable from a CloudFront-blocked region.** `bybit` returns
HTTP 403; `binance`, `okx`, `kraken`, `deribit`, `defillama`, `geckoterminal`
and `mempool_space` all answer. So:

| runs locally | needs a Bybit-capable instance |
|---|---|
| `indicators` `bollinger` `scanner` | `bybit-perp-scan` |
| `beta-vs-btc` `strategy-engine` | `bybit-executable-size` |
| | `funding-spread` `cross-venue-spread` |

bmo honours `HTTPS_PROXY`, so Bybit can be routed through a proxy that reaches
it. For price-level analysis the gap matters less than it looks —
`cross-venue-spread.bmo` measures Binance against Bybit at under 1.2 bps.

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

### Sessions

Newer instances keep bindings between requests, so expensive data is fetched
once and every later query reuses it:

```bash
tools/bmo --session-new
tools/bmo -S 'panel: <expensive fetch>; panel::count()'   # 2202 ms
tools/bmo -S 'panel | <a hypothesis>'                     #    2 ms
tools/bmo -S '$ | take 5'        # $ is the previous successful output
tools/bmo --session-info         # state and remaining idle time
tools/bmo --session-end
```

Idle sessions expire after 30 minutes and only one query runs in a session at
a time. Older builds have no `/api/sessions`; the script says so rather than
failing obscurely. This is what made testing six factor hypotheses take 635 ms
instead of six refetches.

Every query in `queries/` runs against a live instance and returns rows; they
are meant to be read as worked examples as much as run.

## Before trading off any of this

`docs/trading-notes.md` is not optional reading. The short version: the
screen's momentum ranking was tested against forward returns and does not
hold up — its sign flips between the two halves of the sample, at an R² of
0.002. Use the screen for sizing, volatility-aware stops and context; do not
trade its ranking. bmo is read-only and cannot place an order.

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
