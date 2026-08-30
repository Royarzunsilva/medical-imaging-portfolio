# Applied AI — Medical Imaging & Clinical ML

Seven projects in applied machine learning for medicine, built and evaluated by a
**radiation oncologist working as an ML engineer**. The combination is the point: the
same person defines the clinical target, trains the model, and then attacks the result
until it either survives or is reported as failing.

Every project ships the metric as computed — not rounded up, not cherry-picked — and
several of them exist mainly because a number that looked good turned out not to be.

---

## Deployed systems

### [Brain Metastasis Auto-Contouring](./brain_mets_autocontouring/)
Packaged desktop application: segments contrast-enhancing brain metastases on
post-contrast T1 MRI and exports a **DICOM RT Structure Set that imports directly into
a treatment planning system** (Eclipse, Monaco, RayStation).

nnU-Net 3d_fullres, 5-fold ensemble, custom early-stopping trainer. Internal EMA
foreground Dice **0.8389** (fold 0, epoch 87) — nnU-Net's own held-out-fold metric, not
independent clinical validation, and the README says so. **Measured GPU latency: 0.84 s
median per full volume** (RTX 6000 Ada, 2.07 GB peak).

Segmentation overlays included. Application source code withheld; model weights
publishable pending a licence check on the training dataset.

---

## Published research

### [Airway Coach — Videolaryngoscopy Strategy Prediction](./airway_coach_videolaryngoscopy/)
**Published in BMC Anesthesiology (2026)** ·
[10.1186/s12871-026-03943-4](https://doi.org/10.1186/s12871-026-03943-4) · second
author of eight.

Gradient boosting on clinical + point-of-care ultrasound variables to predict which
videolaryngoscopy strategy a patient will need. Single-centre prospective study, 250
adults: **AUC 0.95, accuracy 92%** on an independent test set — with per-class F1 of
0.75 and 0.67 on the two minority classes, which are precisely the two that would
change clinical behaviour.

What makes it worth reading is how the target was defined. Airway scores are usually
validated against the glottic view — what the operator can see. This one predicts what
the procedure actually *required*: whether an adjunct was needed, whether the device had
to be swapped. And the per-class numbers are reported next to the aggregate, which is
where a 0.95 stops looking conclusive.

---

## Evaluation and benchmarking

### [Breast BI-RADS — Multiple-Instance Learning](./breast_birads_mil_radiomics/)
Ordinal MIL over per-patient bags for BI-RADS assessment. Deployed head reaches
**AUC 0.880 / weighted κ 0.730**, and **external validation on 2,447 cases** (different
country) holds the ranking at **AUC 0.807** — while showing that the *thresholds* do not
transfer at all: imported cut-offs send 94% of cases to the urgent lane; recalibrated
locally, PPV 0.70 at 3.2% miss.

Two pre-registered decision rules, both honoured and both reported including the arm
that lost. And an ablation worth knowing about before trusting any ROI-based number,
my own included: with only the 23 **shape** features, the human-ROI arm reaches 0.923
of its 0.940 — so much of what the classifier reads is the outline the radiologist
drew. Whoever draws a spiculated margin already knew it was one. That 0.94 says more
about the annotation than about what is reachable in production.

*Results and methodology only — implementation withheld.*

### [Breast DCE-MRI Segmentation + Choosing a Fair Baseline](./breast_dcemri_segmentation/)
An nnU-Net segmenter (Dice **0.7391**, n=306 closed test set) and, more usefully, the
check that came before using the public baseline: the released checkpoint and the
official test split share cases, so scoring one on the other measures something other
than generalisation. Seen 0.7995 vs unseen 0.7092 (p = 0.018), with a negative control
ruling out case difficulty.

Recomputed out-of-fold, the comparison changes direction — and my own model goes from
apparently losing to **tying** (+0.004, p = 0.17). The contribution is establishing what
the right comparison point was, including when that works against me.

### [Head & Neck GTV — Custom Architecture vs. nnU-Net](./hecktor_head_neck_segmentation/)
**MiniUNet3D** (19 M parameters, custom) benchmarked against **nnU-Net v2** (88 M) on
the identical split, 102 paired cases, with Wilcoxon tests, bootstrap CIs and
Bonferroni + FDR correction across seven metrics.

Primary tumour: **0.800 vs 0.799** — a tie (p_FDR = 0.174). Nodal disease: nnU-Net wins
(0.738 vs 0.774, p_FDR = 0.022). Effect sizes small throughout.

This project also contains a correction of our *own* earlier analysis: a headline
"custom model doubles nnU-Net on small tumours" claim turned out to be an artifact of
binning by each model's predicted volume plus three tumour-absent cases scoring a free
Dice of 1.0. Re-binned correctly, it reverses. The README reports the corrected result.

---

## Systems engineering

### [Clinical RAG Platform](./clinical_rag_platform/)
Local pipeline turning free-text clinical documents into a structured database:
de-identification → ingestion → LLM extraction → curated records → embedding/index →
retrieval. Runtime is **100% local** by design (Ollama/Gemma) — privacy is an
architectural decision, not an add-on.

The differentiator is not the model: the schema and the curated records were built
**by hand, by a clinician**, and the extraction outputs were corrected against clinical
judgement rather than accepted. Honest status: **TRL 3**, retrieval metric not
defensible as evidence, not currently reproducible, domain logic still coupled to the
core. All stated in the README rather than omitted.

### [MRI→sCT Synthesis — VoxelMorph + pix2pix](./voxelmorph_pix2pix_sct/) · [Efficient variant](./voxelmorph_pix2pix_efficient/)
Deformable registration feeding a pix2pix generator to synthesise CT from MRI.
Validation MAE **0.0374** (epoch 59). The lighter variant is **7.3× smaller** (68 MB vs
515 MB) and validates *better*: MAE **0.0171**, PSNR **27.0 dB**, SSIM **0.798**.

---

## How these numbers were produced

- **Frozen splits.** Test and lockbox sets are fixed by seed before any model selection,
  and verified by hash at evaluation time.
- **Pre-registered decisions.** Where a choice between approaches was made, the rule was
  written down before the result was known — and honoured when the preferred option lost.
- **External validation wherever possible.** Several projects here exist because a model
  that looked strong internally degraded across centres or countries. That degradation is
  reported, not buried.
- **Failure cases shown.** Segmentation galleries deliberately include cases where the
  model scored zero.
- **Self-correction.** Two entries report errors found in this portfolio's own earlier
  analyses. A portfolio that only shows wins is not evidence of anything.
- **No patient data.** No images, records or identifiers from clinical sources appear in
  this repository. Public research datasets are named and cited.

## Author

Physician (radiation oncology) and ML engineer. Medical image segmentation, model
evaluation, and clinical deployment — DICOM/NIfTI, nnU-Net, PyTorch, CUDA.
