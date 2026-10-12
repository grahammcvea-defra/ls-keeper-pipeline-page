# LS Keeper Data Bridge — ETL pipeline explainer

A single-page, self-contained animated explainer of the **ETL import pipeline** in
`ls-keeper-data-bridge-backend`: how encrypted supplier files become a consumable
SQLite read model.

Everything (markup, styles, animation logic) lives in `index.html` — no build step,
no dependencies, works offline over `file://` or any static host.

## View it locally

Open `index.html` in any modern browser. That's it.

## Publish on GitHub Pages

1. Push this directory to a GitHub repository (it already contains a `.nojekyll`).
2. In the repo: **Settings → Pages → Build and deployment**.
3. Source: **Deploy from a branch** → branch `main`, folder `/ (root)` → Save.
4. The page appears at `https://<org>.github.io/<repo>/` once the deploy finishes.

## What it shows

`POST /api/etl/imports` (or the daily job / `etl run` CLI) →
`source` → `discover` → `decrypt` → `normalise` → `optimise` → `snapshot` →
`load-duckdb` → `export-sqlite` → `views/*.sqlite` read model.

Each stage materialises artefacts in the internal S3 bucket
(`raw/`, `normalised/`, `optimised/`, `snapshots/`, `staging/`, `views/`),
shown as chips appearing under the stage cards.

A **Stakeholder / Engineer** toggle swaps plain-English descriptions for stage names,
payload types, storage folders and code references.

## Accuracy

Content was hand-built from `origin/develop` of `ls-keeper-data-bridge-backend`
(@ `4c6761e`, October 2026 — `main` is behind and lacks the `optimise` stage).
The animation is illustrative: stage order and names match the code, timings do not.
It deliberately covers only the import pipeline — the separate CPH export, storage
report and cleanse flows are not depicted. If the pipeline changes, edit the `NODES`,
`EDGES` and `TIMELINE` constants in `index.html` — the source files it was derived
from are listed in a comment at the top of the file.
