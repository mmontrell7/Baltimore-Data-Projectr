# Baltimore Civic Insights (Go)

A concurrent Go command-line application that ingests two public datasets from the **Open Baltimore** data portal, joins them by neighborhood and City Council district, and generates reproducible municipal-stakeholder analyses and terminal visualizations.

## What it analyzes

1. **311 workload by neighborhood:** Which neighborhoods account for the most recorded service requests in the current-year 311 layer?
2. **Request categories:** Which service-request types occur most often?
3. **Cross-dataset neighborhood view:** Which neighborhoods combine high 311 request counts with many recorded vacant-building rehabilitation locations?
4. **Council district comparison:** How do recorded rehab-location counts compare with 311 demand by council district, including rehab locations per 1,000 requests?
5. **Service outcomes:** What share of 311 records are marked closed/resolved/completed, and what is the average recorded close time by district?

The third and fourth analyses are descriptive associations, **not causal claims**. “Vacant Building Rehabs” describes recorded rehab locations; it is not a complete inventory of all vacant buildings. The layer's published data last-edit timestamp is June 30, 2023, while the 311 current-year layer is updated daily. Check the source metadata before using findings for policy decisions.

## Datasets and APIs

- **311 Customer Service Requests 2026** — Open Baltimore / Baltimore City ArcGIS FeatureServer. Fields include request type, creation/closure dates, status, neighborhood, council district, agency, and ZIP code. The portal describes this layer as updated daily.
  - Layer: <https://services1.arcgis.com/UWYHeuuJISiGmgXx/ArcGIS/rest/services/311_Customer_Service_Requests_current/FeatureServer/0>
- **Vacant Building Rehabs** — Open Baltimore / Baltimore City ArcGIS FeatureServer. Fields include neighborhood, council district, date issued, address, and proposed use.
  - Layer: <https://services1.arcgis.com/UWYHeuuJISiGmgXx/arcgis/rest/services/Vacant_Building_Rehab/FeatureServer/0>
- Portal: <https://data.baltimorecity.gov/>

The app calls the ArcGIS REST query API directly and saves the downloaded feature records to `output/raw/`. It uses only the Go standard library, so no third-party dependencies or API token are needed for ordinary public queries. ArcGIS service availability, rate limits, field changes, and network access are outside the application's control.

## Requirements

- Go **1.22 or newer** (Go 1.23 was used to compile and test this source in the development environment).
- macOS with network access for live ingestion. Offline fixture mode does not require network access.

## macOS: build, run, and verify

Open Terminal and run:

```bash
# 1. Check Go is installed
go version

# 2. Change to the repository folder
cd baltimore-civic-insights

# 3. Compile and run the automated tests
go test ./...

# 4. Run the reproducible offline demo (synthetic test fixtures)
go run ./cmd/baltimore-insights --offline --out output-demo

# 5. Build a standalone executable
go build -o bin/baltimore-insights ./cmd/baltimore-insights

# 6. Run against live Open Baltimore data (requires internet)
./bin/baltimore-insights --workers 6 --limit 50000 --out output-live
```


### Command-line options

```text
--offline       Use included synthetic fixtures instead of making network calls (default false)
--limit N       Maximum records to retrieve per dataset (default 50000; 0 means no cap)
--workers N     Concurrent worker count (default: up to 8 based on available CPUs)
--out DIR       Output directory (default output)
--timeout       Per-request HTTP timeout (default 45s)
```

The live command fetches each dataset concurrently, queries object IDs, then requests object-ID batches in a bounded worker pool. By default the app caps each source at 50,000 records to avoid unexpectedly large downloads; set `--limit 0` to request all IDs, subject to service availability and memory. When capped, the app chooses the highest object IDs as a practical recent-record approximation; object IDs are not guaranteed to be chronological, so this should not be treated as a strict date filter.

### Generated outputs

After a successful run, inspect the chosen output directory:

- `raw/311_requests.json` — raw ArcGIS feature records used by the analysis.
- `raw/vacant_building_rehabs.json` — raw ArcGIS feature records used by the analysis.
- `neighborhood_summary.csv` — request/re-hab cross-dataset join by normalized neighborhood.
- `council_district_summary.csv` — 311 and rehab counts, closure rate, mean close duration, and rehabs per 1,000 requests by council district.
- `top_request_types.csv` — request-type counts.
- `summary.md` — the five research questions, generated tables, caveats, and ASCII bar charts.

The program also prints summary findings and compact ASCII charts in the terminal. CSV output uses the standard RFC-4180-compatible Go `encoding/csv` writer.

## Repository layout

```text
cmd/baltimore-insights/main.go   CLI flags and application entry point
internal/civic/sources.go        Dataset catalog and field schemas
internal/civic/loader.go         Loader strategy, ArcGIS queries, bounded fetch pool, raw snapshots
internal/civic/parse.go          Concurrent JSON-feature parsing and key normalization
internal/civic/aggregate.go      Sharded aggregates and concurrent cross-dataset joins
internal/civic/output.go         CSV, Markdown report, and ASCII chart rendering
internal/civic/run.go            End-to-end ingestion / analysis pipeline
internal/civic/civic_test.go     Unit and offline end-to-end tests
fixtures/                        Small synthetic offline fixtures
output-demo/                     Example output generated from synthetic fixtures
```

## Design and engineering notes

- **Pipeline and Strategy/Factory patterns:** `Run` orchestrates the stages, `Loader` is the strategy interface, and `NewLoader` chooses live ArcGIS or offline-fixture ingestion. Source schemas live in a central catalog.
- **Concurrency:** two source ingestion jobs run concurrently. ArcGIS object-ID batches use a bounded worker pool. Feature parsing uses workers. Request and rehab aggregates are computed concurrently in independent shards, then reduced deterministically. Neighborhood and district join rows are constructed through a worker pool.
- **Resource management:** HTTP requests have a configurable timeout; goroutine count is bounded by `--workers`; dataset records are capped by default; downloads and results are gathered through channels; each aggregation worker owns its local maps to avoid shared-map writes; cancellation and errors are propagated.
- **Memory:** records are retained in memory to support joins and report generation. For a full unbounded pull (`--limit 0`), memory usage can grow substantially. Prefer the default cap for a first run.
- **No unsupported conclusions:** the datasets have different update cadences and different meanings. The report flags the source-age mismatch and describes co-occurrence without claiming one condition causes another.

## Verification checklist

1. `go test ./...` exits successfully.
2. `go run ./cmd/baltimore-insights --offline --out output-demo` exits successfully and produces the seven files listed above (two raw JSON files plus three CSV files and one Markdown report; the app also generates a `run_metadata.json` file).
3. Open `output-demo/summary.md` and confirm it contains all five research questions and charts.
4. To verify the live pipeline, run without `--offline`; if blocked by network policy, proxy, or ArcGIS throttling, retry from a network that permits HTTPS access to `services1.arcgis.com`.

## Fixture disclaimer

The files in `fixtures/` contain **synthetic records intended only for offline compilation and demonstration**. Do not quote their counts as real Baltimore statistics. Live outputs should be used for factual reporting, with the source-date limitations above considered.
