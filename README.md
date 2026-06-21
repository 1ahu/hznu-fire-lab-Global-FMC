# FMC Co-Retrieval from MODIS

> **Joint inversion of Equivalent Water Thickness (EWT) and Dry Matter Content (Cm)
> for Fuel Moisture Content (FMC) retrieval using Google Earth Engine + PyTorch MLP.**

[![License: CC BY 4.0](https://img.shields.io/badge/License-CC_BY_4.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/) 
[![Python 3.10+](https://img.shields.io/badge/Python-3.10%2B-blue)](https://www.python.org/)

---

## Overview

This repository provides two versions of a satellite-based FMC retrieval framework:

| Version | Notebook | Tree Height Input | 
|---------|----------|-------------------|
| **V1 (Baseline)** | `notebooks/01_FMC_retrieval_baseline.ipynb` | No |
| **V2 (Improved)** | `notebooks/02_FMC_retrieval_treeheight.ipynb` | Yes (GEDI V2.7) |

Both versions use MODIS reflectance (MCD43A4), LAI (MCD15A3H), and land cover (MCD12Q1)
to drive a 4-layer MLP that jointly predicts EWT and Cm. FMC is then computed as:

$$\text{FMC} = \frac{\text{EWT}}{\text{Cm}} \times 100 \quad (\text{clamped to } [0, 400])$$

### Key Features

- **End-to-end GEE processing chain** — all preprocessing, spectral index computation,
  and model inference run inside Google Earth Engine
- **PyTorch-to-GEE bridge** — MLP weights trained in PyTorch are exported as NumPy arrays
  and loaded into `ee.Image` array operations for server-side inference
- **Vegetation-type-specific models** — 4 vegetation types (Grass, Broadleaf, Conifer,
  Shrub) × 2 LAI-availability variants = 8 independent MLP models
- **Quality-controlled model selection** — a 4-level quality band determines whether
  LAI is reliable enough to be used as an input feature
- **Static GEDI canopy height** — V2 adds tree height from GEDI L4A V2.7 as a fixed input
  for woody vegetation, improving EWT/Cm retrieval accuracy


| Vegetation Type | V1 Input Dim (with / without LAI) | V2 Input Dim (with / without LAI) |
|-----------------|-----------------------------------|-----------------------------------|
| Grass           | 10 / 9 | 10 / 9 (unchanged) |
| Broadleaf       | 10 / 9 | 11 / 10 (+ TreeHeight) |
| Conifer         | 10 / 9 | 11 / 10 (+ TreeHeight) |
| Shrub           | 10 / 9 | 11 / 10 (+ TreeHeight) |

**Input features:**
- 7 MODIS NBAR reflectance bands (Band 1–7, scaled to physical units)
- LAI from MCD15A3H (included only when quality level = 0 or 2)
- SWI — Spectral Water Index (cosine angle between reflectance and water absorption at 859 & 1240 nm)
- NSAI — Normalised Spectral Angle Index (vegetation-type-specific scale factor)
- **TreeHeight** — GEDI L4A V2.7 canopy height, static 2019 mosaic (V2 only, woody types)

**Outputs:** EWT (g/cm²), Cm (g/cm²), FMC (%, integer 0–400)

---

## Data Sources

| Dataset | Product | Temporal Resolution | Spatial Resolution | Purpose |
|---------|---------|---------------------|-------------------|---------|
| MODIS MCD43A4 | NBAR Surface Reflectance | Daily | 500 m | 7 reflectance bands + quality flags |
| MODIS MCD15A3H | LAI / FPAR | 4-day composite | 500 m | LAI + quality flags |
| MODIS MCD12Q1 | Land Cover Type 1 (IGBP) | Annual | 500 m | Vegetation classification |
| GEDI L4A V2.7 | Canopy Height (rh98) | Static (2019) | ~25 m → resampled to 500 m | Tree height for woody types (V2) |

> **⚠️  Tree height data notice:** V2 uses a **pre-processed derived asset**
> (`projects/ee-ahu1293659-11/assets/forest_height_2019_global`) created from
> the original GEDI L4A V2.7 data. Users **must** access and cite the original
> data. 

---

## Quick Start

### 1. Clone the repository

```bash
git clone https://github.com/<your-username>/FMC-CoRetrieval.git
cd FMC-CoRetrieval
```

### 2. Install dependencies

```bash
pip install -r requirements.txt
```

### 3. Download model weights

Model weights (`.pkl` files) are available from **[Zenodo / Google Drive — add link]**.
Place all 8 `.pkl` files in the `models/` directory.

### 4. Configure Earth Engine

Update the configuration cell at the top of either notebook:

```python
GEE_PROJECT_ID = "your-ee-project-id"    # Replace with your GEE project
MODEL_DIR = "./models"                    # Path to .pkl files
```

### 5. Run the notebook

Open `notebooks/01_FMC_retrieval_baseline.ipynb` (V1) or
`notebooks/02_FMC_retrieval_treeheight.ipynb` (V2) in Jupyter, Colab, or VS Code.

**For Google Colab users:** The notebook auto-detects Colab and uses
`google.colab.auth`. For local environments, run `earthengine authenticate` once beforehand.

---

## Requirements

- Google Earth Engine account with an active Cloud project
- Python ≥ 3.10
- PyTorch ≥ 2.0
- See [`requirements.txt`](requirements.txt) for full dependencies

---

## Citation

If you use this code in your research, please cite:

> *[Add your paper citation here — authors, title, journal, year, DOI]*

---

## License

This project is licensed under the MIT License — see the [`LICENSE`](LICENSE) file for details.

---

## Notes

- **Model weight security:** The `.pkl` files are loaded with `torch.load(weights_only=False)`.
  Only use models from trusted sources.
- **GEDI tree height access:** V2 requires read access to
  `users/potapovpeter/GEDI_V27` in your GEE account, or an equivalent canopy height dataset.
- **Performance:** Both notebooks compute all 8 model variants for each input image.
  See the optimisation appendix in the notebooks for guidance on production-scale deployment.
