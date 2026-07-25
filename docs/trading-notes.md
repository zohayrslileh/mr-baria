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

## Cross-sectional momentum looked promising, then failed replication

Every *absolute* price factor flips sign between halves — trend, momentum and
the weekly measure alike, at both horizons. The flipping component is the
market move itself, shared by all of them.

Subtracting the cross-sectional mean at each timestamp removes that shared
component, and on the Bybit sample what remained held its sign:

| factor | all | half A | half B |
|---|---|---|---|
| trend, absolute | −0.021 | **−0.203** | **+0.014** |
| momentum, absolute | −0.008 | **−0.122** | **+0.069** |
| trend, relative | +0.112 | +0.029 | +0.158 |
| momentum, relative | +0.113 | +0.030 | +0.155 |

Tercile spread, top minus bottom, in ATR units: +0.154 pooled, +0.021 in half
A, +0.287 in half B. Same direction in both halves — which is more than
anything else here had managed.

**It does not replicate.** `queries/cross-sectional-okx.bmo` runs the identical
test on OKX perpetuals over roughly 200 days, 13,752 observations against the
original 11,352:

| | all | half A | half B |
|---|---|---|---|
| correlation | +0.094 | **−0.036** | +0.154 |
| tercile spread | +0.107 | **−0.061** | +0.275 |

Half A is negative. The +0.029 that made the Bybit sample look stable was
sitting on top of zero, and a longer independent sample puts it on the other
side.

Read the two together and the pattern is plain: **half B is positive in both
samples, half A is not.** What the factor tracks is the regime that happened to
prevail recently, not a relationship that holds. That is the same failure the
absolute factors had, one level down — and it is exactly why the demeaning
looked like a fix rather than being one.

So: nothing here has an edge that survives out-of-sample. The correct posture
remains the one at the top of this file.

**A methodological note, since it cost a round of false confidence.** The
demeaning was chosen *after* watching the absolute factors fail, on the same
data that showed the failure. A result found that way needs an independent
sample before it means anything — and when it got one, it did not survive.
Replicate before believing, not after.

## The scoreboard below was computed on the wrong universe

Every result in this file used a hand-written list of eight symbols chosen from
general familiarity — BTC, ETH, SOL, XRP, DOGE, ADA, AVAX, LINK. Checked
against the exchange, that list does not survive:

| symbol | turnover rank of 411 | 24h turnover |
|---|---|---|
| BTC | 1 | $181bn |
| ETH | 3 | $20.7bn |
| SOL | 14 | $265m |
| AVAX | 32 | $41m |
| LINK | 48 | $10.8m |
| XRP | **135** | **$774k** |
| DOGE | **183** | **$283k** |
| ADA | **196** | **$213k** |

Three of the eight trade under a million dollars a day on this venue. The names
are famous; the liquidity was assumed, and the assumption was wrong.

This matters more than any single factor, because a cross-sectional factor is
measured *within* its universe. Demeaning against a set of eight arbitrary
instruments measures something different from what the results claim to
measure. **The scoreboard is not valid as it stands and needs recomputing.**

`queries/universe.bmo` now derives the universe from the exchange. Sorting by
turnover alone is not enough — OKX lists commodities and tokenized equities as
USDT swaps, and gold is the second most traded instrument on the venue, so a
naive top-20 pulls in XAU, XAG, CL, BZ and SPCX. `fee_group_id` separates them:
4 is the venue's own tier-1 crypto, 5 the rest of crypto, 6 commodities, 7
tokenized equities. That classification is observed rather than recalled.

The derived top-24 above $5m turnover keeps four of the original eight and adds
names no amount of familiarity would have produced — ZEC third at $13bn, plus
YFI, TAO, OKB, ICP, TRUMP, GIGGLE, VVV. The venue's own majors are BTC, ETH,
SOL, DOGE, HYPE, XRP, SUI, ADA, PEPE, which is again not the remembered list.

## Factor scoreboard

Everything tested so far, on 4h bars against the next 24h, outcomes divided by
ATR%, sample split in half by time. A factor passes only if it keeps its sign
in **both** halves and survives on a **second venue**.

| factor | half A | half B | replication | verdict |
|---|---|---|---|---|
| trend, absolute | −0.203 | +0.014 | — | flips |
| momentum, absolute | −0.122 | +0.069 | — | flips |
| weekly, absolute | −0.103 | +0.029 | — | flips |
| momentum, cross-sectional | +0.030 | +0.155 | OKX: −0.036 / +0.154 | flips on replication |
| funding rate | −0.089 | +0.027 | — | flips |
| open-interest change | −0.068 | −0.014 | OKX: −0.044 / +0.008 | flips on replication |

**Nothing has passed.** Two of them looked like they had until a second venue
was tried, which is the whole reason the replication column exists.

Two honest qualifications. The OKX open-interest test carries only 492
observations, because that venue's open-interest history overlaps its candle
range poorly — it is underpowered, and a fair reading is "not confirmed"
rather than "refuted". And every one of these lives on the same twelve liquid
majors at one horizon; a different universe or timeframe is untested ground,
not proven barren.

The shape repeats across all six: half B leans positive, half A does not. That
is a description of which regime happened to fall in which half of the window,
which is what a factor with no edge looks like when it is measured this way.

## What would be worth testing next

- Funding and open-interest factors on their **history** rather than the
  current snapshot — both are available per-bar and neither was validated
  above.
- A longer window. Everything above spans months, and the halves are regimes
  rather than independent draws. Bybit funding reaches back to August 2025 and
  OKX candles paginate further; that is the cheapest way to get a sample where
  the split means something.
- A different horizon. 24h was chosen once and never revisited. Carry-shaped
  ideas plausibly need days, not hours.
- The funding basis between venues, which does not require predicting price at
  all — `queries/funding-spread.bmo` shows 4–8% annualised dispersion on the
  same contract.
- Longer horizons. Everything above is 24h and 48h; a 4h series supports a
  week or more.
- A funding-carry basis trade, which does not depend on predicting price at
  all — `queries/funding-spread.bmo` already shows 4–8% annualised dispersion
  between venues.

---

Nothing in this repository is financial advice, and none of it can place an
order — bmo is read-only. Every number is a starting point for your own
judgment.
