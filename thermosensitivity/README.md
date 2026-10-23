# Thermosensitivity prediction

Predict TRPV1 thermal function and half-maximal response temperature (T50) from protein embeddings. Each sample receives a functional class (`functional` or `less functional`) and a temperature estimate in degrees Celsius.

## Research background

Characterizing mutation-associated changes in TRPV1 thermal response requires both a functional annotation and a quantitative response temperature. This component transfers pretrained OmegaTherm representations to two separate downstream tasks: classifying thermal function as `functional` or `less functional`, and estimating half-maximal response temperature (T50) in degrees Celsius.

The functional task uses a support vector classifier, whereas the T50 task uses a support vector regressor trained on half-maximal response temperature annotations. T50 denotes the temperature at which the thermal response reaches half of its maximum. It is distinct from the onset temperature of activation and from the protein melting temperature (Tm). Although onset-temperature annotations are provided in `data/`, the released temperature model targets `halfmax` and does not predict onset temperature.

The two outputs describe complementary aspects of thermal response and can guide the selection of variants for experimental characterization. The current inference interface selects TRPV1 inputs; the presence of other channel annotations in the data does not establish predictive validity for those channels.

The upstream representation models and feature-extraction code are available in [OmegaTherm](https://github.com/CSUBioGroup/OmegaTherm). This component is part of [TLPTMP](https://github.com/zhuhe1107-hub/TLPTMP), the transfer learning framework for predicting protein thermosensitivity and mechanosensitivity.

## Input

Store each sample as a one-dimensional floating-point NumPy `.npy` vector generated with the OmegaTherm representation and feature-construction method used by the predictor.

| Sample type | Vector shape | Example filename |
|---|---|---|
| Wild type | `(1280,)` | `TRPV1_MOUSE_WT.npy` |
| Mutant | `(3840,)` | `TRPV1_MOUSE_G644N.npy` |

Filenames follow `TRPV1_<SPECIES>_<MUTATION>.npy`. Supported species codes are `MOUSE`, `RAT`, `HUMAN`, and `AXOLOTL`. Directory prediction recursively selects TRPV1 NPY files. Protein, species, mutation, and sample names in the results are derived from these filenames.

## Installation

Run commands from the `thermosensitivity` directory:

```bash
conda create -n tlptmp-thermal python=3.13 -y
conda activate tlptmp-thermal
python -m pip install -r requirements.txt
```

The models use scikit-learn 1.7.2. Prediction runs on CPU.

## Predict

```bash
# Single sample
python predict.py --feature-file /path/to/TRPV1_MOUSE_G644N.npy --out predictions/single.csv

# Directory of TRPV1 embeddings
python predict.py --features-dir /path/to/TRPV1_embeddings --out predictions/results.csv

# Set the inference batch size
python predict.py --features-dir /path/to/TRPV1_embeddings --out predictions/batched.csv --batch-size 256
```

Both models are loaded once per process. The predictor encodes each feature vector and evaluates samples in batches. The output path must be new; a final CSV is published after inference completes. On Linux, the output filesystem must support hard links.

The output CSV contains:

| Column | Description |
|---|---|
| `protein` | Protein name |
| `species` | Species code |
| `mutation` | Mutation identifier |
| `predicted_label` | `functional` or `less functional` |
| `predicted_temperature_celsius` | Predicted half-maximal response temperature (T50), in degrees Celsius |
| `name` | Input filename without the `.npy` extension |

## Python API

```python
import numpy as np
from thermal_predictor import load_models, encode_features, predict_array

models = load_models("models")
embedding = np.load("/path/to/TRPV1_MOUSE_G644N.npy", allow_pickle=False)
features = encode_features([embedding])
labels, temperatures = predict_array(models, features)
```

Pass a list of feature vectors to `encode_features` for batch inference. It constructs a 5122-dimensional input with separate WT and mutant blocks and two presence flags. Load the models once and reuse them for subsequent batches. Keep both `.joblib` files and `model_info.json` in the `models/` directory.

## Models

| Task | Pipeline | Decision rule |
|---|---|---|
| Functional phenotype | StandardScaler -> RBF KernelPCA(48) -> RBF SVC, `C=1` | Apply the classifier's saved class-1 probability threshold of approximately 0.76; class 1 is `less functional` |
| T50 | StandardScaler -> PCA(8) -> RBF SVR, `C=80`, `epsilon=0.5` | Return the regression output in degrees Celsius |

Use the prediction script or `predict_array` to apply the saved classification threshold. Functional annotations and temperature measurements are available in `data/`.
