# WRDS characteristics panel (data preparation for Han et al., 2023)

[![Python](https://img.shields.io/badge/python-3.12%2B-3776AB?logo=python&logoColor=white)](#get-started)
[![pandas](https://img.shields.io/badge/pandas-2.2.1-150458?logo=pandas&logoColor=white)](requirements.txt)
[![Data](https://img.shields.io/badge/data-WRDS%20%2F%20Compustat%20%2B%20CRSP-1f6feb)](#2-data)
[![Characteristics](https://img.shields.io/badge/characteristics-78-success)](input/_characteristics.csv)
[![Paper](https://img.shields.io/badge/paper-Han%20et%20al.%20(EJOR%202023)-b31b1b)](input/han_et_al.pdf)

This repository prepares the firm-characteristic panel used by *Pairs trading via unsupervised learning*
(Chulwoo Han, Zhaodong He, Alenson Jun Wei Toh, **European Journal of Operational Research** 307, 2023, 929–947;
a copy is kept at `input/han_et_al.pdf`). That strategy clusters stocks on firm characteristics rather than on return
series alone, so it needs, for every month, a clean cross section of characteristics and past returns per firm.

The input is Jeremiah Green's firm-characteristic file (the `temp2` table of his SAS code, built from WRDS
Compustat and CRSP). `main.py` keeps the 78 characteristics that Han et al. use, splits the stacked file into one
CSV per month end, appends momentum columns computed from the price column, and optionally produces three
missing-value-free versions of each monthly cross section. No model is trained and no strategy is backtested here;
this is the data stage only.

## Pipeline

```
input/green.csv                       Green's stacked file: one row per (gvkey, month end)
  │   columns kept: the 78 names in input/_characteristics.csv + gvkey, DATE, prc
  │   rows without gvkey/DATE dropped; row 163354 dropped (duplicated in the source)
  ▼
pivot on price                        prc_df: index DATE (month ends), columns gvkey
  │
  ├─► monthly returns ───────────────► output_direct/_momentum.csv
  │     r(t) = prc(t)/prc(t-1) - 1      index DATE, columns gvkey
  │
  ▼
per month end t  (green2output)
  │   characteristics of month t, indexed by gvkey, columns sorted
  │   + momentum from the trailing 48 month ends:
  │       mom_1 = prc(t)/prc(t-1) - 1
  │       mom_n = prc(t-1)/prc(t-n) - 1          for n > 1
  ▼
output_direct/YYYY-MM-DD.csv          one cross section per month end, NaNs left in place
  │
  ▼
per file  (direct2dropnas)            three missing-value treatments
  ├─► output_dropna/simple/           drop every column that has any NaN
  ├─► output_dropna/smart/            smartdropna: search column/row keep-thresholds
  │                                   from 100% down to 20% in 10-point steps, keep the
  │                                   complete block with the most cells
  └─► output_dropna/smartexmom/       smartdropna with mom_1..mom_48 excluded from the
                                      search and re-attached afterwards
```

`mom_1` is the return of the month just ended; `mom_n` for `n > 1` is the return from `n` months ago to one month
ago, so the momentum columns never overlap the return being predicted. The worked examples embedded as docstrings
in `main.py` were produced on a panel of 18,430 gvkeys whose month ends run from 1980 to 2023-12-29.

As committed, `main.py` runs `checkdir()` and `green2output()`; the `direct2dropnas()` call at the bottom is
commented out, so the `output_dropna/` trees are produced only after uncommenting it.

## Repository layout

| Path | Content |
|---|---|
| `main.py` | the pipeline: `checkdir()` creates the folders, `green2output()` builds `output_direct/`, `direct2dropnas()` builds `output_dropna/` |
| `common.py` | helpers shared by the stages: `listdir()` (sorted files by extension) and `smartdropna()` (threshold search for the largest complete block) |
| `input/_characteristics.csv` | the 78 characteristics used: `name`, `green`, `han` flags, `description` |
| `input/han_et_al.pdf` | the paper this panel is built for |
| `input/green.csv` | Green's stacked file — **not** version controlled, supply it yourself (see below) |
| `output_direct/` | generated: `YYYY-MM-DD.csv` per month end plus `_momentum.csv`; not version controlled |
| `output_dropna/simple\|smart\|smartexmom/` | generated: the three missing-value treatments, same shape and names as `output_direct/`; not version controlled |
| `attic/` | two figures (a used-car price box plot, a loan-status pie chart) committed with the initial import; no code here produces them, kept only so nothing is lost |
| `requirements.txt` | pinned environment of the original run |

## Get Started

**1. Install.** `main.py` uses an f-string with nested same-type quotes (PEP 701), so it needs **Python 3.12 or
newer**; it fails to parse on 3.11. Only pandas is imported.

```bash
pip install -r requirements.txt
```

**2. Data.** Place Green's characteristic file at `input/green.csv`. It is the `temp2` table of Jeremiah Green's
published SAS code, run on WRDS Compustat and CRSP; it is licensed data and is therefore not distributed here.
The file must contain `gvkey`, `DATE` (as `YYYYMMDD`), `prc`, and the characteristic columns — column names are
matched case-insensitively against `input/_characteristics.csv`.

**3. Run.**

```bash
python main.py            # writes output_direct/
```

Uncomment the `direct2dropnas()` call at the bottom of `main.py` to also write the three `output_dropna/` trees.
Both stages print a per-month progress bar; `green2output()` also prints the overall NaN ratio of the input.

## Related

`common.py` here is byte-identical to the one in the `WRDS/` folder of the
[Study_DL_and_Coding](https://github.com/JUHWANKIM-ai/Study_DL_and_Coding) repository, and that folder's `main.py`
is the same program with the paths prefixed by `WRDS/`, reading `green.parquet` instead of `green.csv` and calling
`direct2dropnas()`. This repository is the standalone version.
