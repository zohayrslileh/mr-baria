# Binance USD-M futures testnet

Real matching engine, real API semantics, fake money. Reachable from this
environment when nothing else Binance is:

| host | result |
|---|---|
| `testnet.binancefuture.com` `/fapi/v1/*` | **200** — USD-M futures |
| `testnet.binancefuture.com` `/dapi/v1/*` | 200 — COIN-M futures |
| `testnet.binance.vision` `/api/v3/*` | 451 — spot testnet is blocked |
| `api.binance.com`, `fapi.binance.com`, and every mirror | 451 |

So the futures testnet is the one Binance surface available here, and it needs
no tunnel.

## Execution: use the official CLI

Binance publishes one, and it covers more than anything worth hand-rolling:

```bash
npm install -g @binance/binance-cli        # v1.3.0 at time of writing

export BINANCE_API_ENV=testnet
export BINANCE_FUTURES_USDS_BASE_PATH=https://testnet.binancefuture.com
export BINANCE_API_KEY=<testnet key>
export BINANCE_SECRET_KEY=<testnet secret>
```

Relevant commands:

```bash
binance-cli futures-usds futures-account-balance-v3
binance-cli futures-usds account-information-v3          # positions
binance-cli futures-usds new-order --symbol BTCUSDT --side BUY --type MARKET --quantity 0.005
binance-cli futures-usds new-algo-order ...              # TP / SL
binance-cli futures-usds cancel-all-open-orders --symbol BTCUSDT
binance-cli futures-usds change-initial-leverage --symbol BTCUSDT --leverage 2
```

The CLI reads stdin unconditionally and throws `EAGAIN: resource temporarily
unavailable` when stdin is neither a tty nor a pipe. It still prints its result
afterwards, but the error corrupts the output for anything parsing it.
Redirect stdin to fix it:

```bash
binance-cli futures-usds futures-account-balance-v3 < /dev/null
```

Public endpoints work without keys, which makes it easy to check the wiring
before authenticating:

```bash
binance-cli futures-usds kline-candlestick-data --symbol BTCUSDT --interval 4h --limit 2
```

## Getting keys

Register at <https://testnet.binancefuture.com> and generate an API key there.
Testnet accounts arrive pre-funded with play USDT. These credentials are worth
nothing and control nothing real — but they are still credentials, so do not
reuse the pair anywhere that is.

## Sizing

`tools/plan-trade --venue binance-testnet` reads live equity, computes Wilder
ATR from testnet candles, pulls `stepSize`, `minQty` and `tickSize` from
`exchangeInfo`, and prints the three `binance-cli` commands to review: the
entry, then a reduce-only `STOP_MARKET` and `TAKE_PROFIT_MARKET`.

Binance keeps protective orders separate from the entry, unlike Bybit where
`stopLoss` rides along on the order itself. Placing the entry without then
placing its stop leaves the position unprotected.

Note `MIN_NOTIONAL` is 50 USDT on BTCUSDT here, ten times Bybit's floor. At 1%
risk on a $500 account an ATR-wide stop still lands well above it, but a
tighter stop or a smaller account would not.

`tickSize` needs care. It is `0.10` on BTCUSDT, and prices must be *multiples*
of it — `Decimal.quantize(Decimal("0.10"))` gives two decimal places instead,
so 63742.27 passes local rounding and is rejected by the exchange. Snap with
`(value / tick).to_integral_value(ROUND_DOWN) * tick`.

A testnet account is funded with 5000 USDT, which does not represent the $500
being planned for. `--equity 500` sizes against the intended account while
still reading everything else live.

## Which tool reads what

`binance-cli` is for the account and for execution. bmo is for analysis. The
split is not stylistic:

| layer | tool | source |
|---|---|---|
| research and screening | bmo | the real market |
| account state | `binance-cli` | testnet |
| trade sizing | `tools/plan-trade` | live equity + one symbol's ATR |
| execution | `binance-cli` | testnet |

Twelve symbols of 500 4h candles is 843,957 bytes of raw JSON against 801 bytes
for the same question answered inside bmo — about 1000x. Beyond the size, raw
candles arrive uncomputed, so every indicator would be reimplemented against
what bmo already has in `rolling`, `accumulate`, `align` and `correlation`, and
every rerun refetches. And `binance-cli` sees only Binance, while the funding
work needed three venues at once.

The reverse is equally firm: bmo reads public market data only. It cannot see a
balance, a position or an order, which is exactly what `binance-cli` is for.

**Analyse the real market, not testnet.** Testnet klines track real prices to
about 0.04%, but the venue underneath is synthetic — running research on its
candles means studying a simulated order book. `plan-trade` does read ATR from
testnet, because a single symbol is ~10 KB and the stop should match the venue
the order lands on; if the two ever diverge, move that read to bmo.

## What testnet does and does not measure

**Does:** order and stop mechanics, whether reduce-only behaves, funding
accrual, margin and leverage behaviour, liquidation distance, and whether the
whole loop from screen to sized order to fill actually works.

**Does not:** slippage. Testnet prices track the real market closely — 64,330
against 64,358 on OKX, about 0.04% — but the book is synthetic: five levels
deep with a 19.40 spread on BTC where the real market runs a fraction of that.
Fills there say nothing about what a real fill costs.

That distinction matters, because measuring real execution cost was the whole
argument for risking money at all. Testnet validates the machinery for free;
it does not replace that measurement.
