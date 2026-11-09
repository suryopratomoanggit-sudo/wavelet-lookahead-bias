# Wavelet denoising and look-ahead bias in stock forecasting

Code and data for the article *"Wavelet denoising and look-ahead bias in stock forecasting"* (Anggit Suryopratomo, Universitas Ma'soem), submitted to *PeerJ Computer Science* (submission ID 152416).

## Description

Hybrid stock-forecasting models often denoise prices with the discrete wavelet transform (DWT) before training a forecaster. Standard DWT filters are two-sided, so denoising the whole series before splitting it into training and test sets lets future prices leak into the model inputs.

This repository reproduces every experiment in the article. It compares three input schemes on identical test days:

| Scheme | Input signal at day *t* | Uses future data? |
| --- | --- | --- |
| A | Raw log price | No |
| B | Full-sample wavelet-denoised log price (common practice) | **Yes** (look-ahead bias) |
| C | Causal denoising: last value of a denoised rolling window of the past *W* = 256 days | No |

Next-day return direction is forecast under an expanding-window walk-forward protocol and the schemes are compared with exact McNemar, Diebold–Mariano and Wilcoxon signed-rank tests.

## Repository contents

| File / folder | Content |
| --- | --- |
| `Code_analysis_notebook.ipynb` | Python notebook (Google Colab) that downloads the data, screens the sample, applies wavelet denoising and runs all experiments, tests and robustness checks |
| `Raw_data_daily_prices.zip` | Raw daily prices from Yahoo Finance for all 47 candidate stocks, one CSV per stock |
| `Result_files.zip` | Result files (CSV, in English) produced by the notebook, with a codebook (see *Output files* below) |
| `requirements.txt` | Python dependencies |
| `README.md` | This file |
| `LICENSE` | License for the code |

## Dataset information

- **Source:** Yahoo Finance, retrieved with the `yfinance` Python library.
- **Market:** Indonesia Stock Exchange (IDX); ticker suffix `.JK`.
- **Period:** 4 January 2021 to 30 October 2026, daily (the notebook requests `start="2021-01-01"`, `end="2026-10-31"`; the end date is exclusive).
- **Stocks:** the 45 LQ45 constituents for 2 November 2026 – 29 January 2027, plus ARTO and BUKA (47 candidates). After screening, 41 stocks remain.
- **Fields per CSV:** `Date`, `Open`, `High`, `Low`, `Close`, `Adj Close`, `Volume`. Downloaded with `auto_adjust=False`; the analysis uses `Close` (split-adjusted, not dividend-adjusted).
- **Case stocks** (robustness and LSTM experiments): EMTK, ARTO, BUKA, BBCA.

**Screening rules** (a stock is excluded if any rule fails):

1. fewer than 750 trading days;
2. closing price unchanged on more than 15% of days;
3. any closing price at or below the IDX floor of IDR 50;
4. any absolute daily change above 35% (split or data error).

**Reproducibility note:** Yahoo Finance data can be revised after download. To reproduce the published numbers exactly, unzip `Raw_data_daily_prices.zip` into the working folder before running the notebook and skip the download cells (sections 1 and 7, download loop), or place the CSV files under the file names the notebook expects (see *Usage*).

## Code information

The notebook is organised in eight sections:

| Section | Purpose | Article output |
| --- | --- | --- |
| 1. Setup and data for case stocks | Install packages, download case-stock prices | – |
| 2. Data-quality screening of case stocks | Screening statistics and price plots | – |
| 3. Wavelet decomposition and denoising | DWT decomposition check and the `denoise()` function | Eq. (1) |
| 4. Schemes A/B/C and walk-forward evaluation | Causal denoising, feature construction, walk-forward loop, McNemar and Diebold–Mariano tests | Tables 2–3 |
| 5. Robustness experiment | 8 configurations × 4 case stocks = 32 combinations | Figures 1–2 |
| 6. Exact test statistics | Exact binomial, McNemar and Diebold–Mariano statistics | Tables 2–3 |
| 7. LQ45 cross-section | Downloads and screens 47 candidates, runs schemes A/B/C on 41 stocks, Wilcoxon tests | Table 1, Figure 3, Tables S1–S2 |
| 8. LSTM experiment | LSTM forecaster on the case stocks (about 15–20 minutes on CPU) | Table 4 |

**Key functions**

- `denoise(signal, wavelet="db4", level=4)` – DWT denoising with a universal threshold, σ̂ = median(|d1|)/0.6745, T = σ̂·√(2 ln N), and soft thresholding of all detail levels.
- `denoise_kausal(x, jendela=256)` – causal (rolling-window) denoising; the value at day *t* uses only data up to *t*.
- `bangun(x, signal, t_awal, lag=5)` – builds features (the last 5 changes of the input signal) and the target (next-day log return).
- `walk_forward(...)` – expanding-window walk-forward: initial training on 250 observations, refit every 21 trading days.
- `mcnemar(...)`, `diebold_mariano(...)` – paired tests between schemes.

**Main settings**

| Setting | Value |
| --- | --- |
| Wavelet / level / boundary | db4 / 4 / symmetric (PyWavelets default) |
| Causal window *W* | 256 days |
| Forecasting model | Ordinary least squares (scikit-learn `LinearRegression`) |
| Robustness variants | Haar, sym8; levels 3 and 5; windows 128 and 512; Random forest (200 trees, max depth 5, `random_state=0`) |
| LSTM | 1 layer, 16 units, 20 lags, Adam (lr = 0.001), MSE loss, 20 epochs, batch 32, seed 0, refit every 63 days |
| Significance level | 0.05 |

**Glossary of Indonesian identifiers.** Some variable and column names in the notebook are in Indonesian: *saham* = stock, *sinyal* = signal, *jendela* = window, *akurasi* = accuracy, *benar* = correct, *galat* = error, *latih* = training, *uji* = test, *langkah* = step, *mulai/selesai* = start/end, *naif tren* = naive trend baseline, *naif kemarin* = naive persistence baseline, *ketahanan* = robustness, *seleksi* = screening, *hasil* = results, *Masuk/Keluar* = included/excluded.

## Usage instructions

**Option 1 – Google Colab (recommended)**

1. Open `Code_analysis_notebook.ipynb` in Google Colab.
2. (Optional, for exact reproduction) upload and unzip `Raw_data_daily_prices.zip` in the Colab working folder.
3. Select *Runtime → Run all*. The notebook installs `yfinance` and `PyWavelets` automatically.

**Option 2 – Local Python**

```bash
git clone <repository-URL>
cd <repository-folder>
pip install -r requirements.txt
jupyter notebook Code_analysis_notebook.ipynb
```

Run the cells in order. Sections 4–6 and 8 read the case-stock CSV files written in section 1; section 7 writes and reads files in `data_lq45/`.

**Output files** (in `Result_files.zip`, in English; see `CODEBOOK.md` inside the zip for every column)

| File | Notebook writes it as | Content |
| --- | --- | --- |
| `walkforward_accuracy.csv`, `walkforward_tests.csv` | `wf_akurasi.csv`, `wf_uji.csv` | Walk-forward accuracy per scheme and paired tests, case stocks |
| `exact_test_statistics.csv` | `statistik_lengkap.csv` | Exact test statistics and p-values, case stocks |
| `robustness_results.csv` | `uji_ketahanan.csv` | Robustness results, 32 configuration–stock combinations |
| `sample_screening.csv` | `seleksi_sampel.csv` | Screening results for all 47 candidates (Table S1) |
| `lq45_results.csv` | `hasil_lq45.csv` | Per-stock results for the 41-stock sample (Table S2) |
| `lstm_results.csv` | `hasil_lstm.csv` | LSTM results, case stocks (Table 4) |
| `CODEBOOK.md` | – | English description of every file, column and label |

The notebook saves its outputs with Indonesian file names and labels. The files in `Result_files.zip` are the same results with file names, column names and labels translated into English; no numbers were changed.

## Requirements

- Python 3.10 or later
- numpy, pandas, matplotlib
- PyWavelets
- scikit-learn
- SciPy
- TensorFlow (Keras), section 8 only
- yfinance (data download only)

See `requirements.txt` for the list. Runtime: sections 1–7 take a few minutes on Google Colab; the LSTM section takes about 15–20 minutes on CPU.

## Methodology

1. Download daily prices and screen candidates with the four rules above.
2. Take log prices, *x<sub>t</sub>* = ln *P<sub>t</sub>*.
3. Build three input signals: raw (A), full-sample denoised (B) and causally denoised (C).
4. Use the last five changes of the input signal as features to predict the sign of the next-day log return.
5. Evaluate on identical test days with an expanding-window walk-forward protocol; exclude days with zero actual return.
6. Compare schemes with exact McNemar tests (direction), Diebold–Mariano tests (squared errors), exact binomial tests against 50%, two naive baselines, and Wilcoxon signed-rank tests across stocks with Holm correction.

## Citation

If you use this code or data, please cite:

> Suryopratomo A. Wavelet denoising and look-ahead bias in stock forecasting. *PeerJ Computer Science* (under review).

Software used: PyWavelets (Lee et al., 2019, *JOSS* 4(36):1237), scikit-learn (Pedregosa et al., 2011, *JMLR* 12:2825–2830).

## License

Code: MIT License (see `LICENSE`). Price data originate from Yahoo Finance and are provided for research reproducibility only; they remain subject to Yahoo's terms of use.

## Contact

Anggit Suryopratomo – Department of Digital Business, Universitas Ma'soem, Sumedang, West Java, Indonesia – suryopratomo.anggit@gmail.com
