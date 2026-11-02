# Reproducible workflow

This document walks through the full pipeline end to end, from installing the
package to producing the final maps. The pipeline has two stages:

1. **Data extraction (Python, this package):** collect bus routes and hotel
   coordinates into standardized spreadsheets.
2. **Spatial analysis (R, in `post-processing/`):** turn those datasets into an
   origin-destination (OD) flow map and hotel kernel-density maps.

Stage 2 is not part of the installable package; it is included to document the
complete research workflow, and it ships with ready-to-run example data so the
R scripts can be executed as-is.

---

## Stage 1 - Data extraction

### 1. Install

```bash
python -m pip install "git+https://github.com/ivanierruk/movilidad-foranea.git@v1.0.0"
```

Google Chrome must be installed on the system (see the README requirements).

### 2. Configure

Copy the example configuration and edit it:

```bash
cp examples/config.example.yaml config.yaml
```

For a **reproducible** run, set explicit dates (`start_date`, `checkin_date`)
rather than the special value `"today"`. An explicit date lets anyone repeat the
exact query later; `"today"` is a convenience for routine use, but makes the
query non-deterministic because it depends on the day it is run.

City names must be spelled as each platform expects them (see the README note on
platform-specific naming).

### 3. Run

```bash
# Everything (the four bus platforms and Booking), sequentially
movilidad-foranea all --config config.yaml

# Or one group / one platform at a time
movilidad-foranea mobility --config config.yaml
movilidad-foranea mobility --platform ado --config config.yaml
movilidad-foranea hotels --config config.yaml
```

Runs are sequential by design, so a large run (many city pairs x dates) takes
time. Each query waits for results to render; if none appear, a warning is
written to the platform's log rather than silently reporting "no trips".

### 4. Outputs

Results are written under `output/<platform>/`:

- per route/date files (`*.xlsx`),
- a consolidated, de-duplicated file in **both** `*_combined.xlsx` and
  `*_combined.csv` (identical content, two formats),
- a `scraping_errors_<PLATFORM>.log` with any warnings/errors.

For example, ADO produces `output/ado/ado_trips_combined.xlsx`,
`output/ado/ado_trips_combined.csv` and `output/ado/scraping_errors_ADO.log`.

The R scripts in Stage 2 read the **`.xlsx`** version of these consolidated
files; the `.csv` is provided for convenience in other tools.

---

## Stage 2 - Spatial analysis (R)

The R scripts live in `post-processing/`, split into two self-contained folders.
Each folder holds its R script, the input spreadsheets, the city coordinates
(`ciudades.csv`), the national boundary shapefile (`contorno_pais/`), and the
example map outputs, so each script runs on its own with no shared setup:

```
post-processing/
├── mobility/
│   ├── interurban_mobility_flows.R
│   ├── ado.xlsx  budBus.xlsx  clickBus.xlsx  omnibus.xlsx   # inputs
│   ├── ciudades.csv
│   ├── contorno_pais/   # Contorno_Mexico.shp + .shx + .dbf + .prj
│   └── flow_cities.jpg  # example output
└── hotels/
    ├── hotel_kernel_analysis.R
    ├── Booking.xlsx                                         # input
    ├── ciudades.csv
    ├── contorno_pais/
    └── local_kernel.jpg  regional_kernel.jpg               # example outputs
```

### Required R packages

- **Mobility:** `readxl`, `tidyverse`, `sf`
- **Hotels:** `readxl`, `spatstat.geom`, `spatstat.explore`, `sf`, `terra`,
  `tidyverse`

Each script begins with the `install.packages(...)` calls for its dependencies;
these only need to be run once per machine.

### Preparing fresh data (optional)

The two folders already ship example spreadsheets, so the scripts run without any
preparation. To analyze a **new** extraction instead, replace those spreadsheets
with the consolidated outputs from Stage 1. The scripts read the following file
names from their own folder (set R's working directory to that folder, e.g. with
`setwd()`):

| R script                      | Expected input files                                   |
|-------------------------------|--------------------------------------------------------|
| `interurban_mobility_flows.R` | `ado.xlsx`, `budBus.xlsx`, `clickBus.xlsx`, `omnibus.xlsx` |
| `hotel_kernel_analysis.R`     | `Booking.xlsx`                                          |

So each platform's `*_combined.xlsx` is copied in and renamed to the name above
(for example `output/omnibus/omnibus_trips_combined.xlsx` becomes
`omnibus.xlsx`; the BusBud and ClickBus outputs become `budBus.xlsx` and
`clickBus.xlsx`).

### Column preparation - mobility only

The **mobility** script expects a cleaned, renamed set of columns. Before running
it, the extracted columns are renamed to the names used in R, and the unused
`Day Offset` and `Price` columns are removed. The mapping is:

| Extracted column | Column used in R |
|------------------|------------------|
| Origin General   | ori_gral         |
| Origin           | origen           |
| Destination      | destino          |
| Day Offset       | (removed)        |
| Departure Time   | salida           |
| Arrival Time     | llegada          |
| Price            | (removed)        |
| Travel Date      | fecha_viaje      |
| Scraping Date    | scraping         |
| id               | id               |

> This renaming is currently a manual step; it is documented here so the full
> workflow is reproducible. Automating it is a candidate for future work.

The **hotels** script needs no renaming: it reads the Booking output columns
directly (`id`, `pageName`, `destination`, `longitude`, `latitude`,
`query_date`).

### Scripts and outputs

- `interurban_mobility_flows.R` - integrates the four bus datasets, standardizes
  terminals into city keys, builds the OD matrix, and draws the OD flow map (line
  thickness = number of available trips). The example output is `flow_cities.jpg`.
- `hotel_kernel_analysis.R` - runs kernel density estimation (KDE) on the Booking
  coordinates and draws two maps: local scale (sigma = 30 km) and regional scale
  (sigma = 60 km). The example outputs are `local_kernel.jpg` and
  `regional_kernel.jpg`.

Each script renders its maps to R's active graphics device; saving them to an
image file is a manual export step (for example via RStudio's *Export* menu, or
by wrapping the drawing in `jpeg(...)` / `dev.off()`). The committed `.jpg`
files are the example results of that export.

---

## Reproducibility notes

- Every extracted row records its own `Travel Date` and `Scraping Date`, so a
  dataset documents exactly when it was collected.
- Because the data comes from live commercial websites, re-running on a different
  day will not reproduce the same rows (availability and prices change). Use the
  archived dataset (and its DOI) to reproduce a specific analysis; use the code
  to collect a fresh dataset.
- Web sources are fragile: platforms change their markup, add bot protection, or
  vary their layout (see the README limitations). A scraper that works today may
  need updating later.
