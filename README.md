# Medical MNIST — Image Classification, and Why 99% Accuracy Means Nothing Here

Classification of 58,954 medical images (64x64) across 6 imaging modalities:
`AbdomenCT`, `BreastMRI`, `CXR`, `ChestCT`, `Hand`, `HeadCT`.

**The point of this notebook is not the score.** A small CNN reaches ~99.8% on this
dataset, and that number is close to meaningless on its own. The interesting work is
figuring out *why* it is so high, and whether it can be trusted. So this notebook is
organised as an investigation:

1. Look at the data before modelling anything.
2. Fit a **linear baseline** first, so the CNN has something to beat.
3. Train the CNN and compare.
4. Evaluate properly — held-out test set, confusion matrix, and a look at every error.
5. **Audit the dataset for leakage**, because CT and MRI images come from volumes,
   and neighbouring slices of the same patient are nearly identical.

### Summary of results

| Model | Input | Test accuracy | Errors (out of 8,844) |
|---|---|---|---|
| Logistic regression | 16x16 raw pixels (256 features) | 99.38% | 55 |
| Small CNN (2 conv blocks) | 64x64 RGB | 99.97% | 3 |

A linear model on 256 downsampled pixels is within six tenths of a point of the CNN.
That is the real finding: **the six modalities are visually disjoint, so the task is
close to trivial**, and a high accuracy here is not evidence that the model is good.
(The CNN is not useless — it removes 94% of the errors the baseline still makes — but
that is a very different claim from "99.97%".)

### Results

**Near-duplicates are real and concentrated**. Almost a third of the test images have a training image at cosine similarity above 0.95.
We also see that the effect is very class-dependent. `AbdomenCT` and `BreastMRI` have their median above 0.95. 
The others classes have less than that. This could be explained because abdominal CT and breast MRI series produce
many similar consecutive slices. And chest radiograph is one image per patient. 

**But the leakage does not explain the accuracy.** Because, we have a very high accuracy for each  classes. Even the ones with less cosin similarity.
Futhermore, when we remove the "contaminate" pictures we keep an accuracy above 99.95%.

This is worth being explicit about, because the audit was launched on a hypothesis that
turned out to be **wrong in its consequence**. 

Our first guess was that the accuracy was false because of leakage. We have sown that the leakage has an impact, but it is not significant.
The explantion is just that the task is not very difficult.

But we need to keep in mind that this **method has limitations**. 
Pixel correlation is a proxy for "same patient", not a perfect 
measurement of it.  Two different patients imaged at the same anatomical level can
have a high cosine similarity, and two distant slices from the same patient can have a little one.

## 10. Conclusions

**On the dataset.** This was not a difficult dataset. A simple logistic regression on 256
raw pixels has an accuracy of 99.38%. A small CNN reaches 99.97% ! (this means 3 errors on 8,844 images).
When we inspected the dataset we already found that it was not difficult.
We already see it during the visual inspectation of the pictures, the logistic regression and the very fast 
convergence (after one epoch). We can conclude that the errors are mostly noise. 

**On the model.** The CNN only adds ~0.6%. It is not really important. Something quite fun
is that it remove 94% of the errors (55 to 3).

**On methodology.** We try to go futher than the simple accuracy score. With a leakage analysis 
for example. And that two of our predictions were false. The section
2's confusion prediction and the leakage hypothesis were both refuted by measurement.

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
