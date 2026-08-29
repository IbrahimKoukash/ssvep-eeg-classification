# SSVEP BCI — Spring School Hackathon 2025

EEG pipeline for a four-class SSVEP (steady-state visual evoked potential) speller,
built for the BCI Spring School Hackathon 2025. The notebook goes from raw MATLAB
recordings to a trained classifier: artifact rejection, bandpass filtering, trial
segmentation, wavelet + PSD feature extraction, and a comparison of an ANN against
five classical classifiers.

## Data

Recordings are LED-driven SSVEP sessions sampled at **256 Hz** with 8 occipital /
parieto-occipital channels:

```
PO7  PO3  POz  PO4  PO8  O1  Oz  O2
```

Each `.mat` file also carries a `Time`, `Trigger`, and `LDA_output` column. Files
follow the naming pattern `subject_{N}_fvep_led_training_{M}.mat`, and the class
order comes from `classInfo_4_5.m`.

Stimulation frequencies (the four classes): **9, 10, 12, 15 Hz**, presented as
20 trials per run in the repeating order `[15, 12, 10, 9] × 5`.

The `.mat` files are HDF5 (MATLAB v7.3), so they are read with `h5py` rather than
`scipy.io.loadmat`.

Data is **not** tracked in this repository. Drop the recordings in `data/` (see
`data/README.md`) or point the notebook at wherever you keep them.

## Pipeline

| Stage | What happens |
| --- | --- |
| Load | HDF5 read of the `.mat` file, channel labelling, class matrix parsed from `classInfo_4_5.m` |
| Inspect | Per-channel time series plots and Welch PSD (0.5–35 Hz) |
| Slice | Crop to the region between the first and last trigger |
| Artifact rejection | `FastICA` over the 8 channels, artifact component zeroed, signal reconstructed from the mixing matrix |
| Filtering | 2nd-order Butterworth bandpass 0.5–35 Hz, zero-phase (`filtfilt`), with SNR compared before and after |
| Segmentation | Trials cut on rising/falling trigger edges, padded/truncated to 20 |
| Features | Wavelet (`db4`, level 2) coefficient statistics + Welch band power in 1 Hz bins from 8.5 to 30.5 Hz |
| Classification | Keras ANN (1180-944-708-4, softmax) plus Logistic Regression, Random Forest, Gradient Boosting, SVM (RBF), and KNN |

Feature tables are written to CSV and reloaded by the classification section, so
feature extraction and model training can be run independently.

## Setup

```bash
python -m venv .venv
source .venv/bin/activate # Windows: .venv\Scripts\activate
pip install -r requirements.txt
jupyter notebook notebooks/bciHackathon2025Final.ipynb
```

Developed on Python 3.9.

## Before you run

The notebook still contains absolute Windows paths from the original machine.
Update these before running:

- **Cell 4** — reference list of data file paths
- **Cell 5** — `eegDataPath` and `classDataPath`
- **Cell 31** — STFT figure output directory (currently commented out)
- **Cell 40** — `features_DF_all.to_csv(...)` output path
- **Cell 42** — `pd.read_csv(...)` for the feature table used in classification

Suggested replacements, relative to the repository root: `data/` for recordings
and `results/` for generated CSVs and figures.

The pipeline is per-recording: set `subjectNum` and `trainingNum` at the top of
cell 5, run through feature extraction, and each pass appends to `features_DF_all`.
Repeat for every subject/session, then export once.

## Repository layout

```
├── notebooks/
│   └── bciHackathon2025Final.ipynb   # full pipeline
├── data/                             # recordings (git-ignored)
├── results/                          # feature CSVs, figures (git-ignored)
├── requirements.txt
└── README.md
```

## Notes

- The notebook is committed with its outputs so the figures and scores stay
  readable on GitHub. If you would rather keep diffs clean, install
  [`nbstripout`](https://github.com/kynan/nbstripout) and run `nbstripout --install`
  in the repo.
- The ICA artifact component is selected manually (`artifact_indices = [2]`) and
  is specific to one recording — check the component plots before reusing it.
