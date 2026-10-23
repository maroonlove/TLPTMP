# TLPTMP

**Transfer Learning for Predicting Protein Thermosensitivity and Mechanosensitivity**

TLPTMP predicts mutation-associated phenotypes in temperature-sensitive and mechanosensitive ion channels from pretrained protein representations. It provides models for TRPV1 thermal function and half-maximal response temperature, and for mutation-induced changes in mechanosensitive channel function.

## Research background

Understanding how sequence variants alter ion channel responses to temperature and mechanical stimuli requires predictions that distinguish functional phenotypes from quantitative response endpoints. TLPTMP addresses these complementary tasks through transfer learning from pretrained protein representations, with separate downstream models for thermosensitivity and mechanosensitivity.

For thermosensitivity, the TRPV1 component uses pretrained OmegaTherm representations to predict both thermal function and half-maximal response temperature (T50). The functional classifier assigns `functional` or `less functional` labels, whereas a separate support vector regressor estimates the temperature at which the thermal response reaches half of its maximum, in degrees Celsius. These outputs describe complementary aspects of thermal response: functional class and response temperature. T50 is distinct from the onset temperature of activation and from the protein melting temperature (Tm); the released temperature predictor targets T50, although onset-temperature annotations are also included in the data directory.

For mechanosensitivity, the curated dataset contains wild-type and mutant channel sequences with functional annotations relative to wild type. A downstream classifier uses protein embeddings to assign `more`, `less`, or `WT-like`, corresponding to increased, decreased, or approximately preserved mechanosensitive channel function. The released model applies principal component analysis followed by a support vector classifier. It predicts the direction of functional change rather than a numerical force or pressure threshold.

Together, these components extend mutation-associated phenotype prediction across thermal and mechanical responses. They provide computational estimates that can guide variant selection for subsequent experimental characterization; predictions alone do not establish a gating mechanism or experimental validation.

Pretrained models and upstream code for thermal feature extraction are available in [OmegaTherm](https://github.com/CSUBioGroup/OmegaTherm). The TLPTMP framework and its trained prediction models are available in this [TLPTMP repository](https://github.com/zhuhe1107-hub/TLPTMP).

## Prediction tasks

| Component | Predictions |
|---|---|
| [Thermosensitivity](thermosensitivity/README.md) | TRPV1 thermal phenotype: `functional` or `less functional`; half-maximal response temperature (T50) in degrees Celsius |
| [Mechanosensitivity](mechanosensitivity/README.md) | Functional change relative to wild type: `more`, `less`, or `WT-like` |

## Prepare input features

Save each protein embedding as a floating-point NumPy `.npy` file. Use the representation model and feature-construction method associated with the predictor; preprocessing is applied automatically during prediction.

| Component | NPY input shape | Filename |
|---|---|---|
| Mechanosensitivity | `(3840,)` or `(1,3840)` | Any sample identifier, such as `sample_001.npy` |
| Thermosensitivity, wild type | `(1280,)` | `TRPV1_<SPECIES>_WT.npy` |
| Thermosensitivity, mutant | `(3840,)` | `TRPV1_<SPECIES>_<MUTATION>.npy` |

Thermal filenames encode sample information, for example `TRPV1_MOUSE_G644N.npy` or `TRPV1_RAT_N628K.npy`. Supported species codes are `MOUSE`, `RAT`, `HUMAN`, and `AXOLOTL`. Directory prediction searches recursively; the thermal predictor selects filenames beginning with `TRPV1`.

## Quick start

Run the following commands from the repository root. Create a separate environment for each component to use the scikit-learn version required by its model weights. Prediction from embeddings runs on CPU.

### Mechanosensitivity

```bash
conda create -n tlptmp-mechanical python=3.9 -y
conda activate tlptmp-mechanical
python -m pip install -r mechanosensitivity/requirements.txt

# Run the TRPV4 example
python mechanosensitivity/predict.py --features-dir mechanosensitivity/data/embeddings/test --out predictions/mechanical_example.csv

# Predict your own embeddings
python mechanosensitivity/predict.py --features-dir /path/to/embeddings --out predictions/mechanical.csv
```

The `prediction` column contains:

| Label | Meaning |
|---|---|
| `more` | Increased function relative to wild type |
| `less` | Decreased function relative to wild type |
| `WT-like` | Function similar to wild type |

For single-file prediction, filename filtering, and the Python API, see [mechanosensitivity](mechanosensitivity/README.md).

### Thermosensitivity

```bash
conda create -n tlptmp-thermal python=3.13 -y
conda activate tlptmp-thermal
python -m pip install -r thermosensitivity/requirements.txt

# Predict one TRPV1 variant
python thermosensitivity/predict.py --feature-file /path/to/TRPV1_MOUSE_G644N.npy --out predictions/thermal_single.csv

# Predict a directory of TRPV1 embeddings
python thermosensitivity/predict.py --features-dir /path/to/TRPV1_embeddings --out predictions/thermal.csv
```

The output includes `predicted_label` (`functional` or `less functional`) and `predicted_temperature_celsius` (T50 in degrees Celsius), together with protein, species, mutation, and sample identifiers.

For feature encoding, model parameters, output columns, and the Python API, see [thermosensitivity](thermosensitivity/README.md).

Choose a new output path for each new prediction dataset. Both command-line tools load their model weights automatically from the corresponding `models/` directory. Add `--help` to either prediction command to list available options.
