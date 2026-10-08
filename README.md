# Medical MNIST — Image Classification, and Why 0.99 Accuracy Means Nothing Here

Classification of 58,954 medical images (64×64) across six classes, 
`AbdomenCT`, `BreastMRI`, `CXR`, `ChestCT`, `Hand` and `HeadCT` by using pytorch.

**This project is not about the score.** A small CNN reaches 0.9997 on this dataset in a
single epoch. So, the interesting question will be about

is what that number actually measures, so the
notebook is structured as an investigation rather than a training script: fit a trivial
baseline first, evaluate properly on a held-out test set, and audit the dataset for the
leakage that volumetric medical imaging makes likely.

## Results

| Model | Input | Test accuracy | Errors (of 8,844) |
|---|---|---|---|
| Logistic regression | 16×16 raw pixels (256 features) | **99.38%** | 55 |
| Small CNN (2 conv blocks, 1.05M params) | 64×64 RGB | **99.97%** | 3 |

A linear model on 256 downsampled pixels lands within six tenths of a point of the CNN.
Stated the other way, the CNN removes 94% of the errors the baseline still makes. Both
framings are true, and quoting only one of them is how benchmark numbers get oversold.

## Three findings

**1. The benchmark is saturated.** Three independent lines of evidence agree: the six
modalities are visually disjoint on inspection, a linear classifier on downsampled pixels
already solves 99.4% of it, and the CNN converges in one epoch. With three test errors
remaining, the test set can no longer discriminate between two reasonable models.

**2. Near-duplicate leakage is real but does not explain the accuracy.** CT and MRI scans
are volumes, so consecutive slices of one patient are near-identical images. Measuring
cosine similarity between test and training images shows ~32% of test images have a
training near-duplicate above 0.95 — concentrated in `AbdomenCT` (72%) and `BreastMRI`
(92%), and nearly absent in `CXR` and `ChestCT`, exactly as the volumetric explanation
predicts. But splitting the test set by contamination shows accuracy of 100% on the
leaked half against 99.96% on the clean half. The leakage is worth ~0.04 points. The task
is simply easy.

**3. The errors follow acquisition physics, not anatomy.** The prediction that the three
CT classes would be the main confusion was wrong — they are classified perfectly. The
baseline's errors concentrate on `Hand`→`HeadCT` (20) and `CXR`→`Hand` (11): plain
radiographs, which are projections through the body rather than reconstructed
cross-sections, and which are genuinely closer in pixel space.

Two hypotheses in this notebook were refuted by measurement. Both are kept, with the
reasoning that produced them, because that is where most of the value is.

## Dataset

[Medical MNIST on Kaggle](https://www.kaggle.com/datasets/andrewmvd/medical-mnist) —
58,954 greyscale JPEGs at 64×64, roughly balanced (10,000 per class; 8,954 for
`BreastMRI`).

The images are **not** tracked in this repository. Download the archive and extract it so
the layout is:

```
archive/
├── AbdomenCT/
├── BreastMRI/
├── ChestCT/
├── CXR/
├── Hand/
└── HeadCT/
```

## Running it

```bash
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
jupyter notebook projet.ipynb
```

Then run all cells. The notebook detects CUDA or Apple MPS automatically and falls back
to CPU. End-to-end runtime is about 5 minutes on an M-series Mac; the training loop is
roughly 20 seconds per epoch and the leakage audit about 1 minute.

## Notebook structure

| Section | Content |
|---|---|
| 1–2 | Setup, dataset exploration, and the hypotheses it suggests |
| 3–4 | Preprocessing, augmentation rationale, 70/15/15 split |
| 5 | **Linear baseline** — logistic regression on raw pixels |
| 6–7 | CNN architecture and training, with learning curves |
| 8 | **Evaluation** — confusion matrix, per-class report, every misclassified image |
| 9 | **Leakage audit** — near-duplicate detection and its effect on the score |
| 10 | Conclusions and next steps |

## Methodological choices

- **Three-way split (70/15/15).** Any decision informed by the validation score
  contaminates it as a final metric, so the test set is used once, at the end.
- **A single `random_split` draw** over indices, guaranteeing disjoint subsets, with
  assertions verifying it.
- **Two transform pipelines.** Augmentation is applied to training data only; validation
  and test are measured on clean images.
- **Horizontal flips are safe here but would not be for a diagnostic task** — mirroring a
  chest X-ray produces a heart on the wrong side. It is acceptable because the label is
  the modality, not a finding.
- **Seeded throughout** (`SEED = 67`) for reproducibility.

## Limitations

Pixel correlation is a proxy for "same patient", not a measurement of it: two different
patients imaged at the same anatomical level can correlate above 0.98, and two distant
slices from one patient can fall below 0.95. The audit therefore bounds the effect of
*near-duplicates*, not of true patient-level leakage, which cannot be settled without
patient identifiers this dataset does not provide.

## Next steps

- **Data-efficiency curve** — retrain with 10/50/100/500/1000 images per class. On a
  saturated task, how much data the problem *needs* is more informative than the score.
- **Robustness to corruption** — Gaussian noise, blur and contrast shift applied to the
  test set only. Scanner and protocol variability between hospitals is the real
  distribution shift in medical imaging.
- **Grad-CAM** — verify the model attends to anatomy rather than a border artefact. On a
  task this easy, that is a live possibility.
