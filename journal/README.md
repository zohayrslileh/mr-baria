# Journal

Out-of-sample record. `tools/snapshot` appends the screen's output here with a
timestamp; `tools/forward-test` later asks the exchange what actually happened
and scores the buckets. Because rows are written before the outcome exists,
nothing here can be fitted after the fact — which is exactly what the backtest
in `queries/` cannot promise.

- `snapshots.jsonl` — one line per contract per run (git-tracked, it is the data)
- `trades.csv` — fills you actually got, filled in by hand

`trades.csv` is where the numbers the backtest omits come from. Record the
intended price and the price you got, and after ~15 trades there is a real
slippage figure to calibrate against instead of an assumption.

Never put an API key, token or secret in this directory.
