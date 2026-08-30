# Pipeline architecture

Conceptual description of the deployed application. Source code is not included in this repository (see the README).

## Stages

### 1. Series auto-discovery

Input is a folder, not a file. Clinical DICOM exports arrive as deeply nested directories with unpredictable naming, often containing several MR series plus scouts, screenshots and sometimes an existing structure set.

The stage walks the tree, identifies DICOM by magic bytes rather than extension (many exports have no `.dcm` suffix), groups files by directory, discards directories with fewer than ten images and anything with modality `RTSTRUCT`, then scores the remaining candidates:

- modality `MR` dominates the score,
- keywords associated with post-contrast T1 in the series description or protocol name (`t1`, `gado`, `contrast`, `fspgr`, `mprage`, `bravo`, `tfe`, `post`, `ce-`, `+c`, `enhancing`) each add weight,
- slice count breaks remaining ties in favour of the thin-slice volumetric acquisition.

The highest-scoring series is used. The full ranked list is surfaced in the UI so the operator can see what was chosen and what was rejected.

### 2. Structure detection — one patient or a cohort

Before processing, the input folder is classified:

- images sitting directly in the folder → single study;
- sub-folders whose series all share one `PatientID` → single study;
- sub-folders carrying distinct `PatientID`s → cohort, processed in batch.

Each study in a cohort is assigned a sequential anonymous label and written to its own output directory. No patient identifier from the source data is ever used to name an output.

### 3. DICOM → NIfTI

The selected series is read as an ordered volume through SimpleITK's GDCM series reader and written to NIfTI, preserving spacing, origin and direction. Geometry is carried through the whole pipeline so the final contour can be placed back on the original grid exactly.

### 4. Skull stripping

HD-BET is invoked on the volume, on GPU when available. It is optional by design: if the binary is absent or fails, the pipeline falls back to the unstripped volume rather than aborting. This matters for deployment — the tool has to keep working on a machine where an optional dependency did not install.

### 5. Inference

nnU-Net v2 `3d_fullres` predictor, initialised once and cached for the lifetime of the process so a batch of studies pays the model-load cost only on the first case.

- 5-fold ensemble, `checkpoint_best.pth` from each fold
- sliding window, tile step 0.5
- test-time mirroring disabled
- all computation kept on device
- CUDA when present, CPU fallback otherwise

Output is a binary mask on the preprocessed grid.

### 6. Post-processing

The mask is resampled to the exact geometry of the source DICOM series with nearest-neighbour interpolation, then labelled into connected components. Each component's volume is computed from the voxel spacing, and components below 0.1 cc are dropped — below that size, detections are dominated by false positives and the lesions are not treatable targets in any case. If nothing survives, the study is reported as negative and no structure set is written.

### 7. RT Structure Set export

Surviving components become one ROI each, individually named and colour-coded, marked with an automatic generation algorithm, and built against the source MR series so the structure set carries a correct frame-of-reference reference. This is what allows a TPS to fuse the MR onto the planning CT and display the AI contours in the planning workspace.

### 8. De-identification

The generated structure set is rewritten with patient, institution, physician and study identifiers cleared, private tags removed, `PatientIdentityRemoved` set to `YES` and a de-identification method recorded. Verified across every file produced in the 12-study run.

### 9. Reporting

Per study: a machine-readable summary with lesion count, per-ROI volumes, device used and whether skull stripping succeeded; plus a multi-slice QC preview rendering the contours over the source slices for a quick visual check before importing anything into the planner. A cohort-level summary aggregates counts and volumes across studies.

## Failure handling

Each study is processed inside its own error boundary — a failure on one case is recorded and the batch continues. UI progress callbacks are isolated from the pipeline so a rendering error can never kill a running job. Temporary working directories are cleaned per study, including on the no-lesion path.

## Distribution

- **Installable app**: dependency installer plus launcher, opening a local web UI on the operator's machine. The custom trainer class required by folds 2–4 is installed into the `nnunetv2` package at startup if it is not already importable.
- **Standalone executable**: PyInstaller one-directory build — a 77 MB launcher plus a bundled runtime (Python 3.11, PyTorch, nnU-Net, SimpleITK, pydicom and their dependencies). Runs on a machine with no Python installed. Model checkpoints ship alongside and are byte-identical to those in the source tree.
