# RTA Data Pipeline

> Python pipeline that cleans, validates and analyzes contact-center operational data, and raises real-time alerts for Real-Time Analysts (RTA).

## Overview

A Real-Time Analyst spends a lot of time preparing raw exports before any monitoring or reporting can start:
agent states arrive with inconsistent names, mixed timestamp formats, duplicates and overlaps, and queue statistics
contain impossible values. This project automates that whole preparation step and turns the clean data into the KPIs
and alerts an RTA tracks every day.

**Inputs** (CSV exports)

| File | Content |
|------|---------|
| `agent_states.csv` | Agent state intervals (On Call, Available, After Call Work, Break, Lunch, Not Ready) |
| `schedule.csv` | Planned activities per agent (Work, Break, Lunch) |
| `call_intervals.csv` | 30-minute queue statistics (offered, answered, abandoned, answered within SLA, AHT) |

**Outputs** (`output/`)

| File | Content |
|------|---------|
| `rta_report.xlsx` | Cleaning log, validation results, team summary, agent daily KPIs, queue daily KPIs, queue intervals, alerts |
| `agent_states_clean.csv`, `call_intervals_clean.csv` | Cleaned datasets |
| `agent_states_rejected.csv`, `call_intervals_rejected.csv` | Rejected rows, each with a reject reason |
| `alerts.csv` | All alerts raised |

## Features

- **Standardization**: unifies agent IDs, state names (`ON_CALL`, `Talking` become `On Call`), queue names and mixed timestamp formats (`YYYY-MM-DD` and `DD/MM/YYYY`).
- **Cleaning**: removes duplicates, fills missing end times from the next state, trims overlapping intervals, rejects impossible rows (end before start, negative counts, answered greater than offered, and so on).
- **Validation**: 10 automated integrity checks run on the cleaned data, and the pipeline exits with an error if any check fails.
- **KPIs**: schedule adherence (minute-level comparison of planned vs actual), occupancy, shrinkage, service level, abandon rate and weighted AHT.
- **Alerts**: long break, lunch or Not Ready, low adherence, service-level breach, high abandon rate.
- **Automation**: watch mode re-runs the pipeline whenever the input files change.

## Project structure

```
.
├── generate_sample_data.py   # creates messy sample data in data/raw/
├── rta_pipeline.py           # clean -> validate -> KPIs -> alerts -> report
├── requirements.txt
├── data/raw/                 # input CSVs
└── output/                   # generated results
```

## Getting started

```bash
git clone https://github.com/<your-username>/rta-data-pipeline.git
cd rta-data-pipeline
pip install -r requirements.txt

python generate_sample_data.py      # generate sample data (24 agents, 5 days, 3 queues)
python rta_pipeline.py              # run the pipeline
python rta_pipeline.py --watch 60   # optional: re-run every time the input files change
```

To use your own data, put your exports in `data/raw/` with the same column names, or use `--indir` and `--outdir`:

```bash
python rta_pipeline.py --indir path/to/exports --outdir path/to/results
```

## Sample run

Running the pipeline on the generated sample data (synthetic, with deliberately injected errors):

| Cleaning step | Rows affected |
|---------------|--------------:|
| States: duplicate rows removed | 182 |
| States: missing end_time filled from next state | 178 |
| States: invalid rows rejected | 94 |
| States: overlapping intervals trimmed | 90 |
| Calls: duplicate rows removed | 8 |
| Calls: invalid rows rejected | 15 |

All 10 validation checks pass after cleaning.

## KPI definitions

| KPI | Formula |
|-----|---------|
| Adherence % | Minutes where the actual state matches the schedule / scheduled minutes |
| Occupancy % | (On Call + After Call Work) / (On Call + After Call Work + Available) |
| Shrinkage % | (Break + Lunch + Not Ready) / logged-in time |
| Service Level % | Answered within SLA / offered |
| Abandon Rate % | Abandoned / offered |
| AHT | Average handle time, weighted by answered calls |

## Alert thresholds

Defined at the top of `rta_pipeline.py` and easy to change:

| Alert | Default |
|-------|---------|
| Low adherence | below 85% |
| Service-level breach | below 80% (intervals with at least 10 offered calls) |
| High abandon rate | above 5% (intervals with at least 10 offered calls) |
| Long break / lunch / Not Ready | above 20 / 50 / 10 minutes |

## Tech stack

Python, pandas, NumPy, openpyxl

## License

MIT
