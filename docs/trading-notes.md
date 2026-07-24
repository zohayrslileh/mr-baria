# Trading notes — Bybit USDT perpetuals

What the screen in `queries/bybit-perp-scan.bmo` is, what it is not, and what
testing it actually survived. Read this before sizing anything off a row it
produces.

---

## The headline: the momentum score has no demonstrated edge

The screen ranks contracts by a volatility-normalised momentum score. That
score was tested against what actually happened next, and it did not hold up.

**Method.** `queries/factor-validation.bmo` reconstructs the price-based part
of the score bar by bar from data available *up to that bar only*, on 12 core
perps at 4h resolution, then measures the move over the following 6 bars
(24h). Outcomes are divided by ATR% so violent contracts do not dominate.
11,424 observations.

**Pooled result.**

| score bucket | n | mean forward move (ATR) | up rate |
|---|---|---|---|
| strong + | 3,072 | −0.148 | 41.2% |
| mild + | 1,336 | −0.093 | 44.8% |
| neutral | 1,669 | +0.049 | 50.6% |
| mild − | 1,526 | +0.132 | 51.6% |
| strong − | 3,821 | +0.060 | 51.8% |

Correlation −0.047, R² 0.002. Read alone this says the score is *inverted* —
that strength was followed by weakness.

**It does not survive a split-sample check.** `queries/factor-robustness.bmo`
cuts the same data in half by time:

| sample | corr 24h | corr 48h | strong + | strong − |
|---|---|---|---|---|
| first half | −0.064 | −0.067 | −0.372 | +0.430 |
| second half | **+0.026** | **+0.029** | **+0.096** | **−0.195** |

The sign flips. The first half mean-reverted, the second half trended, and the
pooled figure was driven entirely by the first. With R² around 0.002 the
magnitude is noise-level in both directions.

**So:** do not trade the ranking, and do not invert it either. Inverting would
be fitting to the first half of one sample of one universe on one horizon.

## What the screen is still good for

Everything except the ranking survives, and it is not nothing:

- **Position sizing.** `qty` and `notional` are derived so that price reaching
  `stop` costs exactly `RISK_PCT` of `ACCOUNT`. That arithmetic is correct
  regardless of whether the signal has edge, and it is the part most often got
  wrong by hand.
- **Volatility-aware stops.** Stops at 1.5 ATR and targets at 3 ATR adapt to
  each contract instead of using a fixed percentage.
- **Context per contract, in one place.** ATR%, RSI, 24h and 7d moves, open
  interest change, annualised funding, and the price/OI regime label.
- **Instrument hygiene.** See below — this one materially changes the output.

Treat it as a structured watchlist that does the arithmetic, not as a
recommendation engine.

## Instrument class matters more than any factor

Bybit lists three different things as USDT linear perpetuals, and
`market.instruments` distinguishes them through `symbol_type`:

| `symbol_type` | what it is | examples |
|---|---|---|
| null | core crypto perpetual | BTCUSDT, SOLUSDT |
| `"innovation"` | innovation-zone listing, newer and thinner | AKEUSDT, BANKUSDT |
| `"stock"` | tokenized equity | SPCXUSDT, SNDKUSDT, SKHYNIXUSDT |

An unfiltered screen fills its top ranks with the latter two. Tokenized
equities track their underlying market and gap across its closed hours, so a
crypto momentum factor reads their weekend as a signal. The first version of
this screen ranked SpaceX, SanDisk and SK Hynix above every crypto contract.

`CLASS: "core"` is the default for that reason. `launch_time` from the same
source gives a real listing age, which is not the same as having 120 candles —
the API happily returns a full window for a contract listed last month.

## Funding needs a sanity bound

Annualising an instantaneous funding rate produces four-digit percentages on
squeezed contracts. In one scan AKEUSDT showed −2015% APR, and the naive
scoring read that as maximum carry in favour of a long. It is the opposite: a
funding rate that extreme is a squeeze already underway.

`FUND_CAP` neutralises the carry component beyond ±100% APR and flags the row
`funding dislocation` instead of rewarding it. Two of 44 liquid contracts were
past that bound at the time of writing, and both had ranked in the top ten.

## Costs the backtest does not model

`queries/strategy-engine.bmo` reports gross returns. Missing:

- **Taker fees**, charged on entry and exit. Check your own tier — the
  standard perpetual taker rate is around 0.055% per side, so roughly 0.11%
  round trip.
- **Funding**, paid or received every 8 hours while the position is open. At
  10% APR that is about 0.11% per day. On a multi-day hold this can exceed the
  fees.
- **Slippage**, which grows with size and with how thin the contract is.
- **Liquidation mechanics.** Nothing here models leverage, margin mode, or the
  distance to a liquidation price.

Rough scale: on a contract with 1.5% ATR, a 3-ATR target is a 4.5% move and a
0.11% round trip is about 2.4% of it. On BTC at 1% ATR the target is 3% and
the round trip is 3.7% of it. Not fatal, not ignorable, and enough to erase a
0.002-R² edge several times over.

## Before trusting any change to the scoring

The validation harness is the durable part of this work. Any new factor should
clear all four bars:

1. Reconstructed from **past data only** at each bar — no lookahead.
2. Tested against a **normalised forward outcome**, not raw percent.
3. **Split by time** and required to keep its sign in both halves.
4. Compared against costs — an edge under roughly 0.15% per round trip does
   not survive fees.

Note that overlapping windows inflate the apparent sample: with a 6-bar
forward window, consecutive observations share five bars, so 11,424 rows carry
closer to 1,900 independent observations. Significance is weaker than the row
count suggests.

## What would be worth testing next

- Funding and open-interest factors on their **history** rather than the
  current snapshot — both are available per-bar and neither was validated
  above.
- Cross-sectional ranking (long the top decile, short the bottom, market
  neutral) rather than directional calls, which removes the market beta that
  dominates the current result.
- Longer horizons. Everything above is 24h and 48h; a 4h series supports a
  week or more.
- A funding-carry basis trade, which does not depend on predicting price at
  all — `queries/funding-spread.bmo` already shows 4–8% annualised dispersion
  between venues.

---

Nothing in this repository is financial advice, and none of it can place an
order — bmo is read-only. Every number is a starting point for your own
judgment.
