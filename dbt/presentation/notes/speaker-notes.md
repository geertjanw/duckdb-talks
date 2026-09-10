# Speaker notes

Full talk track for **dbt without the warehouse (or the bill): DuckDB end to end**.
The same per-slide notes are in [`../slides/deck.html`](../slides/deck.html) (press
**s**). This file adds the pre-flight checklist, timings, and the fallback plan.
Everything references the [`demo`](../../demo) project.

## Before you walk on stage

Do all of this once, off-camera, then reset so the demo is genuinely from scratch:

```bash
cd dbt/demo
export DBT_PROFILES_DIR=.              # read profiles.yml from here, not ~/.dbt
pip install dbt-duckdb pandas          # if not already

# Pre-download the data so the demo never depends on conference Wi-Fi:
mkdir -p data
duckdb -c "COPY (SELECT * FROM read_parquet('https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2024-01.parquet')) TO 'data/2024-01.parquet'"

# Warm the toolchain once, then delete the outputs so the stage run is clean:
dbt run  --vars '{trips: "data/*.parquet"}'
dbt test --vars '{trips: "data/*.parquet"}'
rm -rf target web/*.parquet warehouse.duckdb
```

`export DBT_PROFILES_DIR=.` points dbt at the [`profiles.yml`](../../demo/profiles.yml)
in this folder instead of the default `~/.dbt/profiles.yml`. The project ships its own profile, so the demo
runs straight from a fresh checkout without depending on anything in your home
directory, handy if someone asks how it's configured.

Checklist:
- [ ] Terminal font large. A second tab already `cd`'d in with `DBT_PROFILES_DIR` set.
- [ ] Browser open but tab NOT yet pointed at `localhost:8000`.
- [ ] `data/*.parquet` present, decide whether you're demoing one month (~3M rows)
      or a whole year (~40M), use `--vars '{trips: "data/*.parquet"}'` either way.
- [ ] Rehearse once: the only things you type live are three commands described below.

## Slide-by-slide

**1. dbt without the warehouse (or the bill).** The idea: somewhere along the way we
all accepted that transforming a few gigabytes of data means renting a distributed
system by the second. This talk questions that assumption with real code, dbt end to
end on DuckDB.

**2. We rented a distributed system to transform a few gigabytes.** Set up the
problem. This is the everyday experience most people in the room will recognise.

**3. Every dbt run is a round-trip you didn't need.** Walk through the three costs:
it's slow, you pay by the second, and you wait behind your colleagues' CI jobs. The
last one is the one everyone has felt: queuing on a shared warehouse.

**4. A few hundred GB, a strange way to live. You can stop.** State the main point: 
a few hundred gigabytes fits on the laptop in front of you, so renting a
cluster for it doesn't make sense and you don't have to.

**5. DuckDB.** Free, MIT-licensed, and in-process, which means it runs inside dbt
rather than over a network. It's now in public beta on the dbt Fusion engine, which
is what makes this even more practical today. (This live demo runs on dbt-core 1.10,
not Fusion, see the "On the engine" note in [`../../README.md`](../../README.md) for why.)

**6. What that unlocks.** Cover the three things this gives you: fast model
iteration on your laptop, CI that runs in seconds on a plain GitHub runner, and
Python models that run in-process so your data never leaves. 

**7. The warehouse becomes optional, at every stage.** DuckDB's native readers
pull Parquet, Iceberg, and Postgres sources in place, DuckLake publishes your output
tables in open formats, and DuckDB-WASM serves them in the browser. So it's optional
at every stage, not just development. This leads into the demo.

**8. Let's build one. Live.** Switch to the terminal and increase the font size.
The warehouse is already gone, so you're starting from scratch and building
the whole thing: developed, tested, deployed, and served, on DuckDB, on
[more data than you'd think reasonable](https://www.getdbt.com/dbt-summit/agenda/dbt-without-the-warehouse-or-the-bill-duckdb-end-to-end)
(the phrase is from the abstract).

**9. 1 · Develop → type `dbt run`.** Say what the data is first: about three million
NYC yellow-taxi trips for one month, and the question we're after is simple, do riders
actually tip, and does it depend on how they pay (credit card vs cash)? While it builds,
explain what's happening: it reads those trips straight from a Parquet URL, no load step,
builds a [staging model](../../demo/models/stg_trips.sql) and a
[payment-summary mart](../../demo/models/payment_summary.sql) through `ref()`, and runs a
[Python model](../../demo/models/trips_by_hour.py), all in the same process. It finishes
in a few seconds.

To make the "no load step" claim concrete, open
[`models/stg_trips.sql`](../../demo/models/stg_trips.sql) and point at
the one line that does it: `from read_parquet('{{ var("trips") }}')`. That
`read_parquet(...)` is plain DuckDB: it reads the file in place, no `COPY` and no
external table. The URL it defaults to is set once as the `trips` var in
[`dbt_project.yml`](../../demo/dbt_project.yml), the `--vars '{trips: ...}'` flag just
points it somewhere else.

This is an opportunity to also show other DuckDB SQL syntax.

When it finishes you should see `Done. PASS=5 ... ERROR=0 ... TOTAL=5`, five models
built in dependency order in a few seconds: the three tables
([`stg_trips`](../../demo/models/stg_trips.sql),
[`payment_summary`](../../demo/models/payment_summary.sql), and the Python
[`trips_by_hour`](../../demo/models/trips_by_hour.py)) and the two `publish_*` external
models ([`publish_payment_summary`](../../demo/models/publish_payment_summary.sql),
[`publish_trips_by_hour`](../../demo/models/publish_trips_by_hour.sql)). Three things should be pointed out:
- **`ls -lh warehouse.duckdb`**, the whole "warehouse" is one file on your laptop.
  No server, no account. This is the core image of the talk.
- **The Python model built in the same run**, listed alongside the SQL models: same
  process, same command, and the data never left.
- **`ls web/`**, the two Parquet files are already written. Deploy happened as part
  of `dbt run`, so there's no separate deploy command to show. That's the connection
  to slide 11.

**10. 2 · Test → type `dbt test`.** This is the "test" stage: dbt's data-quality
gate, running entirely on DuckDB. Everything passes in under a second, about six
assertions over ~3M rows:
- **Schema tests** from [`schema.yml`](../../demo/models/schema.yml): `not_null`,
  `unique`, and `accepted_values` (payment_type must be 0–6).
- **One custom business rule** in
  [`tests/assert_card_tips_beat_cash.sql`](../../demo/tests/assert_card_tips_beat_cash.sql): it asserts
  that credit-card riders tip more than cash riders (cash tips mostly go unrecorded).
  It's a real business expectation encoded as a test that fails the build if it breaks.

The test only prints PASS/FAIL, it doesn't show the tip figures. Those come from the
[`payment_summary`](../../demo/models/payment_summary.sql) model and appear on screen at slide 12. The point: dbt testing works
fully and instantly on DuckDB, on your laptop.

**11. 3 · Deploy.** No new command. `ls web/`, the two `publish_*` models
([`publish_payment_summary`](../../demo/models/publish_payment_summary.sql),
[`publish_trips_by_hour`](../../demo/models/publish_trips_by_hour.sql)) already
wrote `payment_summary.parquet` and `trips_by_hour.parquet`. Deploying here didn't
mean loading a warehouse: it meant writing open Parquet files. DuckLake is the
managed, production version of the same idea.

**12. 4 · Serve → `cd web && python3 -m http.server 8000`.** Open the browser tab.
The [page](../../demo/web/index.html) runs DuckDB compiled to WebAssembly and queries the
Parquet locally. This is the first time the tip numbers are actually on screen,
credit-card riders about **$4.17 (28.7%)**, cash riders **$0.00**. `dbt run` computed
them into the [`payment_summary`](../../demo/models/payment_summary.sql) model earlier. `dbt test` checked that card riders tip more than cash
riders. Now the same DuckDB engine, compiled to WebAssembly, reads the published Parquet
and shows them, in the browser, with no backend.

**13. The receipts.** About three million rows, built and tested in seconds, the
scan itself in milliseconds, and nothing on the cloud bill. The few seconds are
mostly dbt starting up, so adding more data doesn't change the time spent.

**14. Your data is not that big. Your bill doesn't have to be either.** Close on
these two lines, thank the audience, and point them to the source code and/or DuckDB site.

## If the demo misbehaves

- **Wi-Fi / URL flaky:** you already pre-downloaded, run everything with
  `--vars '{trips: "data/*.parquet"}'`. Nothing touches the network.
- **`dbt run` errors on stage:** you have the warmed run from pre-flight. Re-run it,
  or fall back to showing the pre-generated `web/*.parquet` and jump to Serve.
- **WASM page won't load (CDN):** the page fetches the DuckDB-WASM bundle from the
  jsDelivr CDN, so it needs network, if it's down, show the same numbers from the
  terminal with `duckdb -c "select * from read_parquet('web/payment_summary.parquet')"`.

## More data than you'd think reasonable

"More data than you'd think reasonable" is from the
[talk abstract](https://www.getdbt.com/dbt-summit/agenda/dbt-without-the-warehouse-or-the-bill-duckdb-end-to-end).

Point the `trips` var at a folder holding a full year:
`dbt run --vars '{trips: "data/*.parquet"}'` over ~40M rows still finishes in seconds.
Your data is not that big.
