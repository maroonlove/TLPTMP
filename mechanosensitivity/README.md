# Mechanosensitivity prediction

Predict whether a mutation increases, decreases, or approximately preserves mechanosensitive channel function relative to wild type.

## Research background

Predicting how sequence variants alter mechanosensitive channel function requires distinguishing increased function, decreased function, and function similar to wild type. This component addresses that task using pretrained protein representations as inputs to a downstream classifier.

The curated dataset provides paired wild-type and mutant sequences with relative functional labels: `more`, `less`, and `WT-like`. Its channel annotations include MscL, MSL8, PIEZO1, PIEZO2, and TRPV4. Each sample is represented by a 3840-dimensional embedding, and the released model applies principal component analysis followed by an RBF support vector classifier to predict the functional class.

These predictions can guide variant selection for experimental characterization of mechanical responses. They describe the direction of functional change relative to wild type, rather than a numerical activation force, pressure threshold, or complete stimulus-response curve. The inference package accepts precomputed embeddings; their representation model and feature-construction method must match those used to train the saved predictor.

## Input

Each sample is a floating-point NumPy `.npy` embedding with shape `(3840,)` or `(1,3840)`. Use embeddings generated with the representation model and feature-construction method used by the predictor. Sample filenames become identifiers in the output CSV.

The TRPV4 examples are `data/embeddings/test/MS0094.npy` (S542Y) and `data/embeddings/test/MS0095.npy` (R616G). Variant annotations and sample-to-feature mappings are available in `data/fianldata.csv` and `data/samples.json`.

## Installation

Run commands from the `mechanosensitivity` directory:

```bash
conda create -n tlptmp-mechanical python=3.9 -y
conda activate tlptmp-mechanical
python -m pip install -r requirements.txt
```

The model uses scikit-learn 1.6.1. Prediction runs on CPU.

## Predict

```bash
# Single sample
python predict.py --feature-file data/embeddings/test/MS0094.npy --out predictions/single.csv

# TRPV4 examples
python predict.py --features-dir data/embeddings/test --out predictions/example.csv

# Your own embeddings, including nested directories
python predict.py --features-dir /path/to/embeddings --out predictions/results.csv

# Select files by name
python predict.py --features-dir /path/to/embeddings --pattern "TRPV4_HUMAN*.npy" --out predictions/human.csv
```

The output CSV contains:

| Column | Description |
|---|---|
| `sample_id` | Input filename without the `.npy` extension |
| `relative_file` | File path relative to the input directory |
| `prediction` | `more`, `less`, or `WT-like`, relative to wild type |
| `feature_sha256` | Input feature checksum |

A `.summary.json` file records class counts and checksums. Completed outputs with matching CSV and model checksums are skipped on rerun. Use a new output path when changing the input data or model.

## Python API

```python
import numpy as np
from protein_svc import load_model, predict_array

model = load_model("models/svc_pca.joblib")
embedding = np.load("data/embeddings/test/MS0094.npy", allow_pickle=False)
prediction = predict_array(model, embedding)
print(prediction.tolist())
```

For batch inference, pass a matrix with shape `(N,3840)` to `predict_array`. Load the model once and reuse it for subsequent batches. Keep `svc_pca.joblib` and `model_info.json` in the same directory.

## Model

The predictor applies PCA with 20 components, followed by an RBF SVC with `C=3` and `gamma=auto`. It uses native SVC classification, with no input standardization. The saved pipeline performs preprocessing and classification together.
