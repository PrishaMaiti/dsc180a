# Poverty Targeting in Malawi: PMT Calibration and the Relative Value of Prediction

This repository applies a poverty-targeting analysis, first done for Ethiopia, to Malawi's 2019/20 Fifth Integrated Household Survey (IHS5). All of the code is in one notebook, `malawi_analysis.ipynb`, which:

1. **Cleans the IHS5 survey modules** into one household-level table. This covers consumption, assets, housing, household head and demographics, and education. The result is written to `survey2019.csv`.
2. **Calibrates a proxy means test (PMT).** A LASSO model predicts household consumption from survey features over 100 random 75/25 train/test splits. Predictions are written to `pmt_predictions/`, and the notebook reports R² and a calibration plot.
3. **Runs two allocation case studies** with the [`rvp`](https://github.com/unai-fa/relative-value-of-prediction) package: a step-function utility (CS1) and a CRRA utility (CS2). Each one includes welfare surfaces, the relative value of prediction, and budget frontiers.

---

## 1. Getting and storing the data

The raw survey data is **not** in this repository. It is too large for GitHub, and the World Bank terms of use don't allow redistributing it. Each user downloads their own copy.

### Download

1. Go to the World Bank Microdata Library entry for **Malawi – Fifth Integrated Household Survey 2019-2020** (study ID `MWI_2019_IHS-V_v06_M`): <https://microdata.worldbank.org/index.php/catalog/3818>.
2. Sign in or create a free account, open **Get Microdata**, and accept the terms of use.
3. Download the **CSV** version of the data. The file is named `MWI_2019_IHS-V_v06_M_CSV.zip`.
4. Unzip it. You should get a folder named `MWI_2019_IHS-V_v06_M_CSV/` containing files such as `HH_MOD_B.csv`, `HH_MOD_F.csv` and `ihs5_consumption_aggregate.csv`.

The notebook reads these files from that folder:

| File | Used for |
|---|---|
| `ihs5_consumption_aggregate.csv` | Household consumption (outcome), region, urban/rural, household size, poverty lines |
| `HH_MOD_B.csv` | Household roster: head's sex, age and marital status; children under 5 |
| `HH_MOD_C.csv` | Education: head's literacy and highest school level |
| `HH_MOD_F.csv` | Housing: walls, roof, floor, fuel, lighting, water, toilet, mobile phone, etc. |
| `HH_MOD_L.csv` | Durable assets owned |

### Where to store it

Keep the data folder **outside** the repository, or at least never commit it. By default the notebook looks for:

```
~/Downloads/MWI_2019_IHS-V_v06_M_CSV/
```

If you put the folder somewhere else, set the `MWI_DATA_DIR` environment variable to its path before you launch Jupyter:

```bash
export MWI_DATA_DIR="/path/to/MWI_2019_IHS-V_v06_M_CSV"
```

If the folder can't be found, the notebook stops at the data-loading cell with `Data folder not found: ...`.

As a safeguard, `.gitignore` excludes `MWI_2019_IHS-V_v06_M_CSV/`, plus the files the notebook generates (`survey2019.csv` and `pmt_predictions/`), in case any of them ends up inside the repo folder.

---

## 2. Software dependencies

The code was tested with **Python 3.12** and these package versions:

| Package | Version tested | Purpose |
|---|---|---|
| `numpy` | 2.1.1 | Numerical work |
| `pandas` | 2.2.3 | Data cleaning and merging |
| `scikit-learn` | 1.5.2 | `LassoCV`, `train_test_split`, `r2_score` |
| `matplotlib` | 3.9.2 | Figures |
| `scipy` | 1.15.2 | Dependency of `rvp` |
| `rvp` | 0.1.0 | Allocation problems, welfare surfaces, budget frontiers |
| `jupyter` / `nbconvert` | any recent | Running the notebook |

`rvp` is not on PyPI. It is installed from GitHub:

```
git+https://github.com/unai-fa/relative-value-of-prediction.git
```

### Environment setup

With conda:

```bash
conda create -n malawi python=3.12
conda activate malawi
pip install numpy pandas scikit-learn matplotlib scipy jupyter
pip install git+https://github.com/unai-fa/relative-value-of-prediction.git
```

Or with a plain virtual environment:

```bash
python3.12 -m venv venv
source venv/bin/activate
pip install numpy pandas scikit-learn matplotlib scipy jupyter
pip install git+https://github.com/unai-fa/relative-value-of-prediction.git
```

The notebook also contains a `%pip install` cell for `rvp`, so it will install itself into the active kernel if you skip the last command above. If you run the notebook in an IDE such as VS Code, select this environment as the kernel.

---

## 3. Reproducing the results

### Option A: run interactively

```bash
git clone https://github.com/PrishaMaiti/dsc180a.git
cd dsc180a
conda activate malawi                         # or: source venv/bin/activate
export MWI_DATA_DIR="/path/to/MWI_2019_IHS-V_v06_M_CSV"   # only if not in ~/Downloads
jupyter notebook malawi_analysis.ipynb
```

Then choose **Kernel → Restart & Run All**. Run the cells in order: later sections read the files that earlier sections write.

### Option B: run headless from the command line

```bash
cd dsc180a
export MWI_DATA_DIR="/path/to/MWI_2019_IHS-V_v06_M_CSV"   # only if not in ~/Downloads
jupyter nbconvert --to notebook --execute --inplace \
    --ExecutePreprocessor.timeout=-1 malawi_analysis.ipynb
```

This runs every cell and saves the outputs and figures back into the notebook.

### What gets produced

| Output | Created by | Description |
|---|---|---|
| `survey2019.csv` | Data-cleaning section | Cleaned household table: 11,434 households × 94 columns (consumption, `hhid`, encoded features) |
| `pmt_predictions/seed_000.csv` … `seed_099.csv` | PMT calibration cell | Test-set `ground_truth` and `predictions` (consumption, USD PPP) for each of the 100 splits |
| Figures and tables in the notebook | Visualization and analysis cells | Descriptive figures, PMT R² and calibration plot, CS1/CS2 welfare surfaces, RVP ratio table, budget frontiers |

Expected key results:

- **PMT R²:** 0.711 (std 0.021 over 100 splits), computed on consumption in levels with no log transform.
- **Relative value of prediction (dW/$ data ÷ dW/$ capacity):** 0.57 for the step utility and 2.64 for the CRRA utility.

The splits use fixed seeds (0–99), so reruns should match these numbers up to small floating-point differences across library versions.

### Notes on rerunning

- Rerunning the calibration cell **overwrites** the files in `pmt_predictions/`. If you lower `N_SEEDS`, delete `pmt_predictions/` first. Otherwise leftover files from the earlier run will be loaded by the later cells.
- The 100-seed calibration is the slowest step. The full notebook runs in a few minutes on a laptop.
- Main parameters, set to match the Ethiopia analysis: targeting threshold = 33rd percentile of consumption, benefit b = $100 PPP, CRRA ρ = 3, data collection cost = $13 PPP per household. Consumption is converted from Malawi kwacha to USD PPP with a factor of 306.03 (the 2020 World Bank PPP conversion factor).
