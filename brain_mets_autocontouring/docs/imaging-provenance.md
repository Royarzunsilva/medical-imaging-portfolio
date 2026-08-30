# Imaging provenance and privacy handling

How the figures in `assets/` were produced, and what was done to make them safe to publish.

## Source

Every panel comes from the QC previews written by the packaged application during a real batch run over 12 studies. The imaging is post-contrast T1 MRI; the red regions are the model's own output for that slice, from the run that produced the exported RT Structure Sets.

The raw imaging volumes are not redistributed. The pipeline deletes its NIfTI working files after each study, so the figures were rebuilt from the rendered previews rather than re-rendered from voxel data.

## Reconstruction method

The application draws its overlay as a pure-red fill at 25% opacity over the grayscale slice, plus an opaque contour band on the lesion boundary. That composite is analytically invertible where the fill is semi-transparent:

- **Fill region** — a pixel rendered as `0.25 · red + 0.75 · gray` recovers its original grayscale value exactly from the green channel divided by 0.75.
- **Contour band** — fully opaque, so the underlying pixels are unrecoverable. That thin band, dilated to absorb antialiased edge pixels, was inpainted by iterative neighbourhood averaging. This is the only part of the "input" panel that is reconstructed rather than recovered, and it is confined to a few pixels around each lesion boundary.
- **Segmentation mask** — recovered from the fill region (which covers the mask minus its one-voxel eroded boundary) and re-dilated, then re-rendered at a consistent style across all figures.

Consequence to be aware of when reading the figures: the left-hand "input" panels show a slight smoothing halo at the lesion rim where the contour band was inpainted. The lesion itself, the surrounding parenchyma and all other anatomy are the original pixel values.

## Privacy handling

### Header-level

The exported RT Structure Sets were confirmed de-identified before anything was used: `PatientName = ANONYMOUS`, `PatientID = ANON_0xx`, `PatientIdentityRemoved = YES`, private tags removed. Checked on all ten exported files.

### Pixel-level — the part that actually matters here

De-identifying the DICOM header does not de-identify a head MRI. Axial slices that include the orbits and facial soft tissue permit face reconstruction, which is a documented re-identification route for neuroimaging.

Handling:

- Only axial slices at the level of the **lateral ventricles / centrum semiovale** were selected. At those levels the field of view contains cranial vault, scalp and brain, and nothing below the orbital roof.
- Every candidate slice containing orbits, globes, nose, paranasal sinuses or facial soft tissue was discarded rather than cropped. Several studies in the run were rejected outright for the figures because their lesions were infratentorial and every preview slice sat at skull-base level.
- Each final image was opened and inspected visually after generation. All four were confirmed to show no orbits, no globes, no nasal or sinus anatomy, no facial soft tissue, no ears, and no burned-in text, laboratory marks or identifiers.

### Other identifiers

- Slice indices, the application's own header text and its byline were removed by cropping to the image panels; all captions in the published figures were written fresh.
- No absolute filesystem paths, usernames, hostnames or machine identifiers appear in any file in this folder, including inside the PNG binaries. PNG text chunks were stripped on save.
- Study labels used in the README (`1–12`, lesion counts, volumes) are sequential internal labels assigned by the batch runner. They carry no PHI and are not traceable to a patient. No individual clinical data beyond lesion count and contoured volume is reported.

## Laterality

Image left/right is not asserted anywhere in the captions, because the display convention of the rendered previews was not verified against the DICOM orientation tags. Lesion locations are described by lobe only.
