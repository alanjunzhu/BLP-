# Airline merger simulation with BLP demand: Alaska–Hawaiian (validation) and JetBlue–Spirit (application)

An empirical IO project on U.S. domestic airline markets, 2022Q1–2025Q2. It builds a product-level dataset from the
DB1B ticket sample and T-100 segment data and estimates logit, nested logit and random-coefficients (BLP) demand
with PyBLP. It then validates merger simulation on the consummated Alaska–Hawaiian merger and applies it to the
proposed JetBlue–Spirit merger. Extensions cover joint demand–supply estimation, optimal instruments, consumer
heterogeneity, conduct and efficiencies, structure–conduct–performance (SCP) regressions, and entry.

The summary paper is [`paper/summary.pdf`](paper/summary.pdf). The project is notebooks only: there are no `.py`
scripts in the repository.

## Repository layout

```
notebooks/
  00_setup.ipynb                       paths, sample periods, carrier lists, plotting/table helpers (loaded by all)
  00_model_functions.ipynb             instruments, IV-GMM logit/nested logit, PyBLP problems, merger simulation
  01_download_data.ipynb               downloads all raw data (BTS, Census, FRED)
  02_build_dataset.ipynb               cleaning (logged step by step) and product construction, 3 market definitions
  02b_cost_shifters_demographics.ipynb Form 41 / jet fuel cost shifters; ACS income distributions
  03_demand_estimation.ipynb           instrument diagnostics, first stage, logit, nested logit, RC (+ supply, optimal IV)
  03b_heterogeneity.ipynb              income random coefficients; business/leisure two-type model
  04_validation_alaska_hawaiian.ipynb  pre-merger estimation, simulation, difference in differences, prediction errors
  04b_conduct_efficiencies.ipynb       conduct test; conduct vs efficiency decomposition for Alaska–Hawaiian
  05_jetblue_spirit.ipynb              JetBlue–Spirit simulation, consumer surplus, harmed routes, efficiencies
  06_robustness.ipynb                  instruments, fixed effects, market definition, nesting, market size, RC draws/starts
  07_scp.ipynb                         SCP regressions, merger-induced concentration as an instrument
  08_entry.ipynb                       fixed-cost bounds for nonstop entry; JetBlue–Spirit with entry responses
data/
  raw/        downloaded files (not committed; recreated by notebook 01)
  interim/    large intermediate files (not committed; recreated by notebooks 02, 03)
  clean/      cleaned product-level datasets and auxiliary tables (Parquet, committed)
output/
  tables/     LaTeX (booktabs) and CSV tables
  figures/    PDF and PNG figures
  results/    JSON/CSV results used by later notebooks and the paper
paper/
  summary.tex, summary.pdf
```

## Reproducing everything

Requirements: Python 3.11 or later, about 15 GB of free disk space, 16 GB of RAM. A full run takes roughly 8–12
hours on 4 cores, most of it random-coefficients estimation. Install the packages:

```bash
pip install -r requirements.txt
```

The cleaned data in `data/clean/` are committed, so you can skip notebooks 01, 02 and 02b and start at 03; those
three are only needed to rebuild the data from the raw downloads (about 3 GB). PyBLP uses up to 6 CPU cores by default;
set `N_CORES` to change that.

Run the notebooks in order from the `notebooks/` directory (each one loads `00_setup` and `00_model_functions` itself):

```bash
cd notebooks
for nb in 01_download_data 02_build_dataset 02b_cost_shifters_demographics 03_demand_estimation 03b_heterogeneity 04_validation_alaska_hawaiian 04b_conduct_efficiencies 05_jetblue_spirit 06_robustness 07_scp 08_entry; do
  jupyter nbconvert --to notebook --execute --inplace --ExecutePreprocessor.timeout=-1 $nb.ipynb
done
```

For a quick end-to-end check on small samples (minutes rather than hours), set `SMOKE=1`. Smoke runs write their
result files with a `_smoke` suffix and overwrite tables and figures, so re-run without `SMOKE` afterwards.

```bash
SMOKE=1 jupyter nbconvert --to notebook --execute --output /tmp/check.ipynb 03_demand_estimation.ipynb
```

Compile the paper (requires a LaTeX distribution with `booktabs`, `graphicx`, `caption`, `hyperref`):

```bash
cd paper && latexmk -pdf summary.tex
```

## Data sources

| Data | Source | Use |
|---|---|---|
| DB1B Market and Ticket, 2022Q1–2025Q2 | BTS TranStats `PREZIP` | fares, passengers, itineraries |
| T-100 Domestic Segment (U.S. carriers), 2022–2025 | BTS TranStats (table 259) | nonstop confirmation, airport presence, airborne time |
| Master Coordinate | BTS TranStats (table 288) | airport locations |
| Form 41 Schedule P-5.2 | BTS TranStats (table 297) | cost per airborne hour (pilot, fuel, other) |
| Metro/micro population estimates, Vintage 2025 | Census Bureau | market size |
| CBSA boundaries 2024 | Census Bureau TIGER/cartographic | airport → metro assignment |
| ACS 2020–2024 5-year, table B19001 | Census Bureau summary file | income distributions |
| CPI-U (CPIAUCSL), jet fuel (MJFUELUSGULF) | FRED | deflator, fuel cost shifter |

DB1B was available through 2025Q2 when the data were downloaded (October 2026). Change `LAST_Q` in `00_setup` to
extend the sample.

## Main modelling choices

* **Markets** are non-directional airport pairs per quarter (see the reasoning at the top of `02_build_dataset`).
  Directional and city-pair markets are robustness checks.
* **Products** are ticketing carrier × market × service type (nonstop or one-stop). Nonstop products must be
  confirmed by T-100 departures.
* **Market size** is the geometric mean of the endpoint metro populations (Census Vintage 2025).
* **Instruments.** The baseline uses BLP rival-characteristic and rival-count instruments. Hausman fares and
  engineering cost shifters give upward-sloping demand in this data (see `03`), so they are reported but not used
  for identification. The cost shifters enter the marginal-cost equation instead.
* **Nesting.** Nested logit is estimated with the specified nonstop vs connecting nests, which yield ρ ≈ 0, and with a
  single air-travel nest, which yields ρ ≈ 0.4.
* **Random coefficients** use a lognormal price coefficient and normal random coefficients on the constant and on
  nonstop service. They are estimated on a random sample of 600 routes and evaluated on all markets.
