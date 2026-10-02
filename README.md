# Hydropower Optimisation

A reproducible forecasting workflow developed for a Zindi hydropower optimisation challenge. The repository contains two notebook implementations that predict hourly electricity consumption (`kwh`) for individual sources and aggregate those predictions into a daily submission. It documents the original modelling ideas and fills in the training, validation, inference, and export steps.

## What is included

| Notebook | Model | Forecast method |
| --- | --- | --- |
| `hydro_optimization.ipynb` | One-dimensional CNN, LSTM, and source embedding | One-hour predictions, fed back recursively into a 168-hour history |
| `hydro_optimization_TFT.ipynb` | Temporal Fusion Transformer (TFT) | 24-hour prediction blocks, fed back for longer horizons |

Both notebooks share an equivalent data-loading and submission contract but are **self-contained**: run either notebook from its first cell. The CNN–LSTM predicts `power_factor`, derived `power`, and `kwh`; the TFT predicts `kwh`. Both use climate observations and cyclic hour-of-day features. The notebooks report validation `kwh` mean absolute error (MAE) alongside a previous-day seasonal baseline.

## Repository layout

```text
.
├── README.md
├── hydro_optimization.ipynb
├── hydro_optimization_TFT.ipynb
├── data/                      # supply locally; not included
│   ├── Data.csv
│   ├── Kalam Climate Data.xlsx
│   └── SampleSubmission.csv   # optional
└── outputs/                   # created by the notebooks
```

The input data is not bundled with this repository. Obtain it through the competition's permitted access route and review its usage terms before sharing it.

## Requirements

Use Python 3.10+ and a Jupyter notebook environment. Install the packages for the model you want to run:

```bash
python -m pip install jupyter pandas numpy openpyxl scikit-learn
python -m pip install tensorflow                            # CNN–LSTM only
python -m pip install torch lightning pytorch-forecasting  # TFT only
```

Use a compatible TensorFlow or PyTorch installation for your hardware. The TFT notebook uses automatic device selection; the CNN notebook also runs without a GPU, although training may take longer. Package versions are not locked, so check compatibility in your environment before a full training run.

## Input data

Set `HYDRO_DATA_DIR` to the directory containing the following files, or place them in `data/` under the notebook working directory:

- `Data.csv`: `date_time`, `Source`, `current`, `power_factor`, `kwh`, `v_red`, `v_yellow`, `v_blue`.
- `Kalam Climate Data.xlsx`: `Date Time`, `Temperature (°C)`, `Dewpoint Temperature (°C)`, `U Wind Component (m/s)`, `V Wind Component (m/s)`, `Total Precipitation (mm)`, `Snowfall (mm)`, and `Snow Cover (%)`.
- `SampleSubmission.csv` (optional): unique `ID` values in `YYYY-MM-DD_Source` form. If present, it determines the requested dates, sources, and output row order.

Set `HYDRO_OUTPUT_DIR` to change the output location; otherwise files go to `outputs/`. Both paths can also be edited in the configuration cell of each notebook. There are no embedded credentials or drive mounts.

Measurements are read in chunks, aggregated to hourly observations per source, and joined to hourly climate data. `kwh` is summed by hour. Current, power factor, and voltage are averaged by hour; voltage is first derived from the sum of the three phase readings for each raw record. Climate values are interpolated within the available climate range, and hour-of-day sine and cosine terms are added. Missing electrical hours are retained as gaps rather than silently assigned zero consumption.

## Run a notebook

1. Install the packages for the chosen model and put the input files in `data/` or set `HYDRO_DATA_DIR`.
2. Open `hydro_optimization.ipynb` or `hydro_optimization_TFT.ipynb` and run the cells **from top to bottom**.
3. Review the printed count of usable sources, the validation MAE, and the seasonal baseline MAE.
4. Check the generated submission in `outputs/` against the current competition template before using it.

The CNN notebook saves `outputs/cnn_lstm.keras` and `outputs/cnn_submission.csv`. The TFT notebook saves the best Lightning checkpoint under `outputs/tft_checkpoints/` and writes `outputs/tft_submission.csv`. Each submission has `ID` and `kwh` columns. The code checks for unique template IDs, missing source coverage, complete 24-hour forecast days, and finite predictions.

When a sample submission is absent, the notebooks forecast `HORIZON_HOURS = 744` hours immediately after the latest observed hour. In that mode, the next hour must be midnight to yield complete daily outputs. Adjust the configuration and confirm the intended dates against the actual challenge specification. The climate spreadsheet must contain weather through the final requested forecast hour.

## Modelling and evaluation notes

- **Time-aware validation:** The last 24 observed hours are held out. In the CNN workflow, per-source MinMax scalers are fitted on training history only. The TFT workflow uses a common cutoff across sources and lets `TimeSeriesDataSet` handle target normalisation.
- **Incomplete histories:** The CNN skips training windows containing missing features or labels; both notebooks reject sources that cannot supply the needed forecast history. The TFT notebook additionally requires complete hourly series for training and validation.
- **Long forecasts:** The CNN feeds each one-hour prediction into its next input. The TFT repeats 24-hour predictions; future non-target electrical covariates are carried forward from the last available value between blocks. This approximation can accumulate error over long horizons.
- **Validation versus final training:** The saved models retain the validation holdout. If model selection is complete, a separate final training pass on all available observations may improve the final submission; that pass is not automatically performed here.

These notebooks were structurally checked and the shared preparation/submission logic was checked with synthetic data. They were not trained or scored on the competition dataset in this repository. No leaderboard score or competition ranking is claimed.
