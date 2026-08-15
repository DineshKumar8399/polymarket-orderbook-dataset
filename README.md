# Polymarket Order Book Dataset

Order-book snapshots from a prediction market, collected continuously between
**2026-07-10** and **2026-08-14**: `237,607,140` quote observations across
`164,709` markets, plus settlement outcomes and a separate high-frequency feed
that records actual traded prices.

It is published so other people can build and train on it without first spending
a month running collectors. Everything here is an independent observational
recording of publicly displayed market data.

**Read [Known issues](#known-issues) before you train anything on this.** The
collection has a four-day outage, one column that dies partway through, and a
label set with time censoring. All three are documented, none are hidden, and
each one will quietly wreck a model if you miss it.

---

## Get the data

The parquet files are **not in this git repo** — they are rebuilt daily, and
committing them would grow the history by roughly 12 MB a day forever. They
live in two places instead, both refreshed every night:

**Hugging Face** (browsable, has a dataset viewer, resumable):

```bash
pip install huggingface_hub
hf download DineshKumar8399/polymarket-orderbook-dataset --repo-type dataset --local-dir polymarket-data
```

```python
import duckdb
duckdb.sql("SELECT * FROM 'polymarket-data/quotes/**/*.parquet' LIMIT 5").show()
```

**GitHub Releases** (a single dated tarball, ~131 MB):

```bash
gh release download data-2026-08-15 --repo DineshKumar8399/polymarket-orderbook-dataset
tar --zstd -xf polymarket-orderbook-*.tar.zst
```

Each release is a frozen snapshot, so `data-2026-08-15` is reproducible: cite the
tag and anyone can reconstruct the exact data you trained on. Latest build:
**2026-08-15**.

---

## Contents

| File | Rows | What it is |
|---|---|---|
| `quotes/dt=YYYY-MM-DD/*.parquet` | `237,607,140` | Book quotes for every tracked market, partitioned by date |
| `markets.parquet` | `164,709` | One row per market: question text, category, coverage |
| `labels.parquet` | `139,167` | Binary settlement outcomes, with a `source` column |
| `watch_quotes.parquet` | `1,441,448` | High-frequency feed — **the only table with traded prices** |
| `data_quality.parquet` | 7 | The known issues below, as queryable rows |

Total: about `131 MB` of ZSTD-compressed Parquet.

---

## Quick start

```python
import duckdb

con = duckdb.connect()

# the whole quote history — the partition layout means you can slice by date
# without reading the rest
con.sql("""
    SELECT * FROM read_parquet('quotes/**/*.parquet', hive_partitioning=1)
    WHERE dt = '2026-08-01' AND slug = 'some-market-slug'
""").show()

# join quotes to outcomes, using ONLY the authoritative labels
con.sql("""
    SELECT q.slug, q.ts, q.bid, q.ask, q.mid, l.y
    FROM read_parquet('quotes/**/*.parquet', hive_partitioning=1) q
    JOIN read_parquet('labels.parquet') l USING (slug)
    WHERE l.source = 'api'
""").show()
```

With pandas or polars:

```python
import pandas as pd, polars as pl

markets = pd.read_parquet("markets.parquet")
one_day = pl.read_parquet("quotes/dt=2026-08-01/*.parquet")
```

---

## Schemas

### `quotes/`

One row per market per poll of the order book.

| Column | Type | Notes |
|---|---|---|
| `ts` | timestamp | When the snapshot was taken |
| `slug` | string | Market identifier, joins to `markets` and `labels` |
| `category` | string | `sports`, `politics`, `climate`, `culture`, … |
| `bid` | double | Best bid. NULL when no bid was resting |
| `ask` | double | Best ask. NULL when no ask was resting |
| `mid` | double | Midpoint; NULL unless the book was two-sided |
| `spread` | double | `ask - bid` as reported upstream |
| `volume24hr` | double | **Mostly unusable — see issue 2** |
| `segment` | int | `1` = before the outage, `2` = after. **See issue 1** |
| `dt` | date | Partition key |

`dt` is stored in the directory name, not inside the parquet files, so it only
materialises as a column when the reader is told to parse the partitions —
`hive_partitioning=1` in DuckDB, automatic in `pandas.read_parquet` /
`pyarrow.dataset` when you point them at the `quotes/` directory rather than at
individual files. The Hugging Face viewer does not parse it, so `dt` is absent
there; slice on `ts` instead when browsing.

Prices are probabilities in `[0, 1]`: a market at `0.35` implies a 35% chance.
A `YES` share pays 1.00 if the event happens and 0.00 otherwise, so the price is
also the cost per unit of payoff.

### `markets.parquet`

| Column | Type | Notes |
|---|---|---|
| `slug` | string | Primary key |
| `question` | string | Human-readable question |
| `category` | string | |
| `n_snaps` | bigint | Quote rows present for this market |
| `first_ts` / `last_ts` | timestamp | Coverage window |

`question` is stored here rather than on every quote row — repeating it
`237,607,140` times is most of why the raw CSV was 32 GB.

### `labels.parquet`

| Column | Type | Notes |
|---|---|---|
| `slug` | string | Joins to `quotes` / `markets` |
| `y` | int | `1` = resolved YES, `0` = resolved NO |
| `source` | string | `api` = authoritative. `convergence` = inferred, **biased** |

**Filter on `source`.** See issues 5 and 6 — this is the single easiest way to
get a wrong answer out of this dataset.

### `watch_quotes.parquet`

| Column | Type | Notes |
|---|---|---|
| `ts` | timestamp | |
| `slug` | string | |
| `bid` / `ask` / `mid` | double | |
| `last_traded` | double | **Last traded price — the only print data here** |

---

## How it was collected

Two independent collectors ran continuously on a dedicated machine:

**Broad sweep** — polled the full market list roughly every two minutes and
recorded the top of book for every market it could see. This produced `quotes/`.
It is wide (every market) but shallow: it records what was *quoted*, never what
*traded*.

**Watchlist** — polled about twelve actively-traded markets every twenty
seconds, rotating the selection every thirty minutes, and recorded the last
traded price alongside the book. This produced `watch_quotes.parquet`. It is
narrow but deep, and it is the only place in this dataset where you can ask
whether a trade actually happened.

That split matters more than it sounds. Displayed quotes are not the same thing
as executable prices, and nothing in `quotes/` can tell you whether a given
quote could have been filled.

---

## Known issues

Also shipped as `data_quality.parquet` so you can assert on them in a pipeline.

### 1. A four-day collector outage

No rows exist between **2026-07-17 17:26:09** and **2026-07-22 10:32:48**. The
machine lost its storage enclosure and stopped writing.

The dataset is therefore **two disjoint series**, not one 35-day window. Any
per-market price path that spans the gap is broken, and any price *change*
computed across it is meaningless — you would be measuring a 4-day-17-hour jump
as if it were a normal interval.

The `segment` column marks which side each row falls on. Restrict to a single
segment, or handle the discontinuity explicitly.

### 2. `volume24hr` dies partway through

94.4% NULL overall, and **0.0% populated after 2026-07-22** — the upstream API
stopped returning the field. It is 26–40% populated before the outage.

Any feature built on volume or liquidity silently becomes all-NULL for the
larger part of the dataset. Use `spread`, or per-market snapshot frequency
(`markets.n_snaps`) as a rough activity proxy, and label them as proxies.

### 3. No traded prices in `quotes/`

The broad sweep records book quotes only. Its `last` column was 100% NULL, so it
has been **dropped rather than shipped as an empty column named `last`**.

If your question is "did this actually transact" — fill realism, execution
modelling, print-versus-quote — it is only answerable on `watch_quotes`, which
covers 4,372 markets rather than 164,709. Note `last_traded` is itself 85.6%
populated, not 100%.

### 4. Crossed books

A small number of rows have `ask < bid`, which is not physically meaningful and
reflects the two sides being read a moment apart. Filter with `ask >= bid` if
your method is sensitive to it.

### 5. Labels are time-censored

Most `source = 'api'` labels come from a one-off backfill run on 2026-07-23/24.
So "has a label" correlates strongly with "settled before Jul 24" — 37,815
markets (23%) carry an authoritative label, and they are **not a random 23%**.

This bites hardest on walk-forward validation: naively splitting train/test on a
late date can leave you with an empty test set and a script that reports success
anyway. Check your split sizes.

### 6. Convergence labels are biased — prefer `source = 'api'`

Rows with `source = 'convergence'` were inferred by watching the price settle
toward 0 or 1. That method systematically mislabels markets whose books died
before converging, and it selects for markets that converged at all.

On this data that bias was large enough to manufacture an edge that did not
exist — a backtest showed a substantial per-share profit that vanished entirely
when the same cell was recomputed on authoritative labels. They are included
because throwing away data is worse than labelling it, but treat them as a
weak-supervision signal, never as ground truth, and never blend the two sources
without checking how much the choice moves your result.

### 7. `watch_quotes` is not a random sample

The watchlist deliberately tracked the *most active* markets, rotating every
thirty minutes. Anything you measure there describes liquid, high-attention
markets — typically live in-play sports — and will not generalise to the long
tail in `quotes/`.

---

## Coverage

Markets by category:

```
   category  slugs
     sports 161511
   politics   1132
    climate    917
    culture    755
      macro    142
 technology     93
    finance     82
     crypto     51
    science     14
geopolitics     12
```

Sports dominates by design: it is the bulk of what the venue lists and the bulk
of what trades.

---

## License

**CC BY 4.0** — use it, remix it, build commercial things on it; just give
credit. Full legal code in [LICENSE](LICENSE); attribution and disclaimer in
[NOTICE](NOTICE).

```
Polymarket Order Book Dataset (2026), Dinesh Gopalakrishnan.
Licensed under CC BY 4.0.
https://github.com/DineshKumar8399/polymarket-orderbook-dataset
```

## Disclaimer

Research and educational use. This is an independent observational recording of
publicly displayed data and is not affiliated with, endorsed by, or supplied
under agreement with any exchange or venue. Nothing here is financial advice.
Past market behaviour does not predict future market behaviour, and a backtest
on this data is not a trading strategy.
