# dbt without the warehouse (or the bill)

A conference talk and its live demo: a dbt project **developed, tested, deployed, and
served entirely on DuckDB**, no server, no account, no warehouse. It runs from a fresh
checkout on one laptop, over ~3M rows of NYC yellow-taxi data that DuckDB reads straight
from a public Parquet URL.

The talk is [dbt without the warehouse (or the bill): DuckDB end to end][abstract] at
the dbt Summit, see the abstract for the context of this repo and presentation.

| Folder | What it is |
|---|---|
| [`demo`](demo) | The dbt project. Four commands take it from `dbt run` to a DuckDB-WASM page in the browser. Runs on dbt-core 1.10 with the `dbt-duckdb` adapter. |
| [`presentation`](presentation) | The deck ([`slides/`](presentation/slides): `deck.html`, plus `.pptx`/`.pdf`) and the full [speaker notes](presentation/notes/speaker-notes.md), including the pre-flight checklist. (The dbt-core vs Fusion engine note is in the "On the engine" section below.) |

## The whole demo, in four commands

```bash
cd demo
export DBT_PROFILES_DIR=.

dbt run                 # DEVELOP, reads the Parquet, builds 3 models + a Python model
dbt test                # TEST, schema tests + "card riders tip, cash riders don't"
                        # DEPLOY, dbt run already wrote web/*.parquet (open format)
cd web && python3 -m http.server 8000     # SERVE, open http://localhost:8000 (DuckDB-WASM)
```

See [`demo/README.md`](demo/README.md) for what each model does, and
[`presentation/notes/speaker-notes.md`](presentation/notes/speaker-notes.md) for the talk track.

## Timing and pacing

The talk runs anywhere from 20 minutes to an hour, as needed. The working assumption is
that although it's live coding (as promised in the [abstract][abstract]), we want to
actually type as little as possible while talking about and showing the code as much as
possible.

## On the engine: dbt-core 1.10 vs the Fusion engine

Two dbt engines exist. This demo runs on one of them.

- **dbt-core 1.10**, the Python-based dbt (v1) with the
  [`dbt-duckdb`](https://github.com/duckdb/dbt-duckdb) adapter. This is what the demo
  runs on. Every stage works, including the in-process Python model.
- **The dbt Fusion engine**, dbt's rewrite (v2, in Rust). DuckDB support on Fusion is
  fast and real, and it's the "public beta on the dbt Fusion engine" the [abstract][abstract] refers
  to. It's CLI-only and still in beta.

**Why the demo doesn't run on Fusion yet.** The project was tested on a Fusion 2.0.0
preview build (2.0.0-preview.218 at the time, though the previews update every few days) with
the DuckDB driver installed via `dbc install duckdb`.

- **Most of it works**, reading Parquet from the URL, `ref()` SQL models, the schema
  tests, the custom business assertion, and the `external` Parquet materialization, with
  the same numbers throughout.
- **The exception is Python models**, on the Fusion DuckDB beta, `trips_by_hour.py`
  fails with `Python models are not supported for duckdb adapter`. The in-process Python
  model is specifically mentioned in the [abstract][abstract], so the demo should probably stay on dbt-core 1.10, where
  the whole pipeline (develop, test, deploy, serve) runs end to end. Once Python models
  land on the Fusion DuckDB beta, the same project runs on Fusion unchanged.
- **Extensions need a system driver**, `httpfs`/`parquet` aren't in Fusion's bundled
  driver, so you install one with `dbc install duckdb`. `profiles.yml` is otherwise
  identical.

In short: DuckDB on Fusion is in public beta and works well. The demo runs on dbt-core
today because it uses a Python model, which the Fusion DuckDB beta doesn't support yet.

[abstract]: https://www.getdbt.com/dbt-summit/agenda/dbt-without-the-warehouse-or-the-bill-duckdb-end-to-end
