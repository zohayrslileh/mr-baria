# bmo — working notes

Distilled from `/docs` and `/docs/language`, then checked against a live
instance. Every claim marked **[verified]** was run and its output observed.
The instance's own documentation stays authoritative; this file records the
parts that are easy to get wrong.

---

## 1. The shape of a program

A program is `;`-separated statements. Its value is the last statement's
value; an empty program is `null`. `#` comments run to end of line.

```
x: 10;
y: 20;
x + y
```

Two worlds sit side by side:

- **expressions** compute values,
- **pipelines** move tables through stages.

```
@range 100                                  # source: starts a pipeline
| where (row) => row.value >= 10            # stage
| extend squared: (row) => row.value * row.value    # stage
| take 5
```

Three extension kinds, in **separate syntactic namespaces**:

| kind | signature | call | first-class? |
|---|---|---|---|
| source | args → one finite value | `@name args` | no |
| stage | table → table | `table \| name args` | no |
| function | args → value | `f(x)` · `f x` · `x::f()` | **yes** |

Because the namespaces are separate, one name can be all three at once.
Sources and stages never enter lexical lookup, so registering one can never
change what a bare identifier means.

### Discovery is just data

```
@sources · @stages · @functions          # tables of [name, description]
@docs "range", kind: "source"            # full contract, as markdown
@binance "operations"                    # per-source operation catalog
@binance "describe", operation: "spot.klines"
```

`kind:` is only needed when a name appears in more than one catalog.

---

## 2. Gotchas worth knowing before you write anything

### `??` binds **tighter** than arithmetic — the opposite of most languages

Precedence runs loosest → tightest: `&` · `|` · `then/else` · `or` · `and` ·
`not` · `= !=` · `< <= > >= in` · `+ -` · `* / %` · `??` · unary `-` ·
postfix.

```
null ?? 0 + 1     # 1     — parses as (null ?? 0) + 1        [verified]
2 * 3 ?? 9        # 6     — parses as 2 * (3 ?? 9)           [verified]
```

The first line is why `(s ?? 0) + r.value` can safely be written without
parentheses in `accumulate`. The second is why you should add them anyway.

### Only five string escapes exist — regexes need doubled backslashes

`\"` `\\` `\n` `\t` `\r`. Anything else is a **lex error**, so a regex
metacharacter escape must be written `\\.`:

```
matches("a.b", "a\.b")    # lex error: unknown escape `\.`      [verified]
matches("a.b", "a\\.b")   # true                                [verified]
```

Slashes are ordinary characters in patterns, not delimiters.

### Decimal recurrences overflow unless you round

Exact Decimal keeps 128 significant digits. A recurrence adds roughly one
digit per row, so an unrounded EMA dies partway through the table:

```
| accumulate ema: (s, r) => s = null then r.close else 0.2 * r.close + 0.8 * s
# eval error: decimal multiplication exceeds the decimal range     [verified]
```

Round every recurrence at the precision you actually need:

```
| accumulate ema: (s, r) =>
    round(s = null then r.close else 0.2 * r.close + 0.8 * s, 8)
```

This applies to EMA, Wilder smoothing, running products — anything whose
state feeds back into itself.

It is not only recurrences. **Any chain of exact arithmetic can reach the
bound**, and aggregates that square their inputs are a common trigger:

```
| extend d: (x) => x.value - group_mean      # 60+ digit differences
... ::correlation("d", "e")
# eval error: decimal multiplication exceeds the decimal range   [verified]
```

Round wherever precision stops being meaningful — `round(x.value - mean, 8)`
here — rather than only inside folds. `rolling` is unaffected because its
declared statistics manage their own accumulation.

### There is no truthiness

`null` means absence only. Conditions must be Boolean:

```
null then 1 else 2      # eval error: conditional requires a boolean   [verified]
```

Aggregates never skip null — they fail on it. Filter explicitly:

```
[1, null, 3]::avg()                          # contract error         [verified]
[1, null, 3]::filter(v => v != null)::avg()  # 2                      [verified]
rows | where "amount"                        # keeps non-null amount
```

### Stage names only work after `|`

```
@range 3 | take 1 | first()      # parse error                        [verified]
(@range 3 | take 1)::first()     # correct — first is a function
```

Leave the table world outside the pipe. The same applies inside callbacks:
`(g.rows | sort "px" | take 1)::first()`.

### `select` keys are the **existing** column

```
| select "id", old_name: "new_name"     # rename during projection
```

Positional names project in order and always precede keyed outputs.

### `group` always names its key column `key`

Schema after `group` is `[key, rows]`, whatever you grouped by. Add
`| rename key: "t"` when you want the original name back.

### Shapes never collapse implicitly

A one-row table stays a table; a one-item list stays a list. Cross shapes
explicitly with `first`, `last`, `list(table)`, `table(records)`. Tables
cannot be indexed at all.

---

## 3. Numbers

Two worlds, and you cross between them only on purpose.

| world | types | entry |
|---|---|---|
| exact | `integer`, `decimal` | literals, arithmetic |
| approximate | `float` | `3.14f`, `float(...)` |

```
0.1 + 0.2      # 0.3 exactly                                        [verified]
10 / 3         # 3.333…333 — 34 significant digits, rounded once    [verified]
sqrt(2)        # decimal, same context                              [verified]
pow(2, 10)     # 1024, integer — exact for integral exponents       [verified]
1 = float("1.0")    # true  — comparison is exact across types      [verified]
0.1 = float("0.1")  # false — the binary value genuinely differs    [verified]
```

`ln`, `log`, `exp` **require** Float input, so approximation is always
explicit. `%` is mathematical modulo: `-10 % 3` is `2`, `10 % -3` is `-2`.
[verified]

Durations: `ms s m h d`, a day is exactly 24h, no calendar units.
`t2 - t1` → duration · `t + 2h` → timestamp · `2h / 5m` → decimal ratio.
Display picks the largest exact unit (`1h + 30m` → `90m`). [verified]

---

## 4. Scope, closures, `out`

Reads search outward; **writes never do** — they bind locally and shadow.

A `=>` function captures a *value snapshot* of its defining scope:

```
x: 1; read: () => x; x: 2; read()      # 1                          [verified]
```

`out` gives the parent scope priority. Counting the levels matters, because
a block body adds a scope that a bare body does not:

```
x: 10;
f: (x) => out x           ; f(20)   # 10  — bare body: out reaches the snapshot
g: (x) => { out x }       ; g(20)   # 20  — block: out reaches the parameters
h: (x) => { out out x }   ; h(20)   # 10                            [verified]
```

At statement position `out` owns the whole binding: `out x: 0` writes to the
parent. Inside an expression it is greedy up to the pipeline boundary, so
`out x + y` means `out (x + y)`.

No implicit self-binding — a function's snapshot predates the binding that
receives it. Recursion passes the function to itself:

```
fact: (self, n) => n <= 1 then 1 else n * self(self, n - 1);
fact(fact, 20)      # 2432902008176640000                           [verified]
```

### Calls have two independent channels

Positional and keyed, evaluated left to right. Bare parameters take only
positional arguments; keyed parameters take same-named keyed arguments and
may carry defaults. Unfilled bare parameters are `null`.

```
greet: (name, title: null) => [name: name, title: title];
greet("Ada")          # name: "Ada"
greet(name: "Ada")    # bare name stays null — different channel
```

One positional parameter may drop its parentheses, and so may one positional
argument — but the bare call is **greedy**:

```
double 7 + 1      # 16 — double(7 + 1)                              [verified]
double(7) + 1     # 15                                              [verified]
```

---

## 5. Parallelism

Two forms, both request-response. No futures, no `await`.

```
[a, b, c]: first() & second() & third();       # static, flat list
results: parallel(jobs);                        # dynamic, list of () => …
limited: parallel(jobs, 10);                    # with a concurrency ceiling
```

Each operand runs in a private copy-on-write world: reads start from the same
snapshot, and nothing written inside crosses back or sideways. If one operand
fails, its peers are cancelled and the whole expression errors — there is no
partial list. Parenthesised groups nest their results: `a & (b & c)` gives
`[a, [b, c]]`.

Sources take bare argument lists, so wrap them when combining:

```
[bn, ok]: (@binance "spot.klines", symbol: "BTCUSDT", interval: "1h")
        & (@okx "market.candles", instrument_id: "BTC-USDT", interval: "1H");
```

### Stage parallelism

An attached `&` after `|` supplies a per-invocation ceiling:

```
rows | 8& extend snapshot: (row) => load(row.key)
```

The attachment is what distinguishes it — `workers& stage` is the modifier,
`workers & expr` is ordinary parallel composition. It may change execution
strategy but never results, ordering, or errors, and output always returns to
input order.

Only these accept it, and only for their **function** forms: `expand`,
`extend`, `where`, `sort`, `rank`, `group`, `rolling`, `window`, `relate`,
`align`. Column forms reject it — their direct path is already cheaper. A
value of `1`, empty input, or a single row all fall back to sequential.

Use it only when callbacks are independent and heavy enough to repay
scheduling — typically per-row network calls, not arithmetic.

---

## 6. Choosing a stage

| need | stage | cost |
|---|---|---|
| filter rows | `where` | `O(n×k)` keyed, `O(n)` calls opaque |
| add columns | `extend` | per-row callbacks |
| trailing statistics | `rolling` | `O(n×k)`, **independent of width** |
| arbitrary frame, lag/lead, centered | `window` | `O(n×w)` |
| state that feeds back (EMA, running totals) | `accumulate` | `O(n×k)` |
| equality join, may match many | `relate` | expected `O(n+m+z)` |
| ordered lookup, at most one match | `align` | `O(m log m + n log m)` |
| partition then aggregate | `group` | expected `O(n+g)` |
| row → table, general relationship | `expand` | `O(n+z×k)`, can be Cartesian |

The three order-aware stages (`accumulate`, `window`, `rolling`) consume the
table's **current** order — `sort` first, always.

`rolling` beats `window` whenever its declared metrics answer the question,
because its cost does not grow with the window width. Reach for `window` only
for lag/lead, centered frames, or non-statistical frame work.

Prefer `relate` over a Cartesian `expand` + predicate whenever the
relationship is equality; prefer `align` when it is ordered and picks one row.

### `align` is the tool for real-world timestamps

Cross-venue timestamps drift by milliseconds. `align` absorbs that; `relate`
(exact equality) would silently drop the rows.

```
| align other, "t", direction: "nearest", within: 30m, kind: "inner"
```

`direction:` is `"backward"` (default, greatest right key ≤ left),
`"forward"`, or `"nearest"`. `within:` caps the distance. `by:` partitions
(or `left_by:`/`right_by:` across differing schemas). `kind:` is `"left"` or
`"inner"` only — it is a left-driven lookup, so there is no right or full.

Both `align` and `relate` output schema `[left, right]`, each cell holding a
whole record, so flatten with `extend t: (x) => x.left.t, …` and use `x.right?.f`
wherever the kind can leave it null.

---

## 7. What the two halves buy you

The pipeline half and the expression half share one grammar, and the leverage
comes from where they meet. Four capabilities follow from that, none of which
a relational query language can express.

### The query decides what to fetch next

`expand` runs a row callback that may call a **source**, so a later fetch can
depend on what an earlier one returned:

```
@binance "spot.ticker_24h", symbols: [...]
| sort quote_volume: "desc"
| take 2
| 2& expand (row) => (
      (@binance "spot.klines", symbol: row.symbol, interval: "1h", limit: 3)
      | extend src: row.symbol
  )
```

[verified] SQL joins tables that already exist; it has no way to say "given
these rows, now go and retrieve those." Here the universe is discovered and
the fan-out follows from it, in one query.

### State that reads its own previous output

`accumulate` state may be any value, including a record — which makes it a
path-dependent state machine over rows:

```
| accumulate st: (s, r) => {
      p: s ?? [pos: 0, entry: 0, eq: 1, trades: 0];
      (p.pos = 0) then (enter(r) then [pos: 1, entry: r.close, …] else p)
                  else (exit(r)  then [pos: 0, eq: round(p.eq * (1 + ret), 12), …] else p)
  }
```

[verified] A SQL window function cannot reference its own prior result, which
is why EMA, trailing stops and position machines all fall outside it.

### Behaviour as a value

Functions are ordinary values, so a strategy can *be* data — a record holding
its own prep pipeline and its own predicates — and one engine runs any of
them without inspecting them:

```
strategies: [
    breakout: [ name: "…",
                prep:  (t) => (t | window 20, 0, ph: …),
                enter: (r) => r.close > r.ph,
                exit:  (r) => r.close < r.pl ] ];

backtest: (candles, strat) => { … strat.prep(candles) … strat.enter(r) … };
```

[verified — `queries/strategy-engine.bmo`] Adding a strategy adds a value;
the engine never changes. SQL has no lambdas, so the equivalent is generating
query text as strings.

### Tables nest and keep flowing

`group` puts a whole table in a cell, and that cell pipes like any table:

```
| group "category"
| extend top: (g) => (g.rows | sort "px" | take 1)::first().venue
```

Relational algebra is flat; nesting needs JSON or array escape hatches.

### Where it is genuinely smaller than SQL

No cost-based optimizer or index selection — stages declare their own
complexity and you choose among them yourself. No persistence, schemas or
transactions spanning requests. No ecosystem of tooling. The trade is
deliberate: less machinery underneath, more expressive power on top.

---

## 8. Result representations

`POST /api/run` takes **raw query text as the body**, not JSON. Use
`curl --data-raw` — plain `--data` treats a leading `@` as a filename, which
matters a lot in a language where every source starts with `@`.

| `Accept` | result |
|---|---|
| omitted, `application/json` | standard JSON |
| `application/json; value-format=typed` | every value tagged |
| `text/event-stream` | SSE |
| `text/event-stream; value-format=typed` | SSE, typed final result |

Standard JSON renders Decimal as a JSON number, which a consumer like Python
will silently round to a float. When exactness matters downstream, ask for
typed — Integer and Decimal arrive as **strings**:

```
10 / 3  standard → 3.3333333333333335        (already damaged by the reader)
10 / 3  typed    → {"type":"decimal","value":"3.333…333"}          [verified]
```

SSE emits `begin` / `log` / `done` / `result` | `error`. Lifecycle events
appear only for operations that stay pending two seconds or that log — fast
calls emit nothing at all. [verified] A failed query still ends with HTTP
`200` because the headers precede the outcome. Browser `EventSource` cannot
be used (it is GET-only); use `curl -N` or streaming `fetch`.

Limits: 30 min and 1 GiB per query; 1 MiB request body; 32 KiB headers;
10 s to deliver a request. One request per connection.

---

## 9. Registered on this instance

13 sources — `binance` (109 ops), `okx` (96), `bybit` (60), `kraken` (27),
`deribit` (26), plus `defillama`, `geckoterminal`, `mempool_space`, `range`,
and the four catalog sources.

16 stages — `where extend select rename drop take sort rank expand union
relate align group accumulate window rolling`.

48 functions — heavily statistical: `statistics correlation covariance
linear_regression percentile median mode stddev variance weighted_avg mae mse
rmse`, plus collection, text, and conversion helpers.

Public market data only: no credentials, accounts, orders, or wallets.
