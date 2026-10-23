# SRT — Mouse Brain Slice 3D Reconstruction

Research repository (lyn14/SRT) for reconstructing a 3D volume of mouse brain
histology slices (WT10, 0.5mm, 72h; 59 serial-section PNG images, ~3000×5000 px,
RGB, some sections missing) and interpolating missing slices between known ones.
All code lives in Jupyter notebooks; there is no build system, package manifest,
or test suite. The project write-up is `SRT Commission.pdf` (Chinese).

## Repository layout

- `Programming/插值算法/work1.ipynb` — slice-stacking 3D reconstruction with
  linear interpolation between slices (PyVista rendering).
- `Programming/插值算法/work2.ipynb` — cubic polynomial interpolation
  (per-pixel Lagrange fit across 4 neighboring slices), vectorized NumPy.
- `Programming/UNET/U-NET.ipynb` — U-Net model that predicts the middle slice
  from two neighbors (2-channel input → 1-channel output), trained with MSE +
  SSIM evaluation on GPU; most recent approach.
- `Working__stack/` — rendered output images (stack renders, SSIM curve, test
  renders). Note: notebook `OUTPUT_PNG` paths reference the older folder name
  `Working_stack` (no double underscore).
- `Literature/` — reference papers (NeRF, GAN, GANsformer, Style-NeRF) and
  3D-reconstruction survey; background reading, not code.
- `SRT Commission.pdf` — project commission/description (Chinese).
- `WT10 0.5mm_72h_1_s01c2.zip` — a subset of the slice data (10 PNGs, ~80 MB).

## Environment setup

No requirements.txt/pyproject exists. Notebooks were developed against the
local conda env `cellpose` (Python 3.12, torch 2.14+cu130, pyvista, torchmetrics,
matplotlib, Pillow). Run:

    conda activate cellpose
    jupyter lab   # or: jupyter notebook

Kernel name in the notebooks is plain `python3`; select the `cellpose` kernel
when opening them. Dependencies used across notebooks: numpy, Pillow, torch,
torchmetrics, pyvista, matplotlib.

## Data

- The notebooks read slices from `E:\working_stack\WT10 0.5mm_72h` (outside the
  repo, 59 PNGs, e.g. `WT10 0.5mm_72h_1_s01c2.png`). Sorted by filename this
  gives top-to-bottom slice order; the filename pattern is
  `<sample>_<series>_<slice>c2.png` (some slice numbers are skipped, e.g.
  `s07` missing in series 1).
- The zip at the repo root contains the first 10 slices if the working folder
  is unavailable.

## Running / validating

- Everything is notebook-driven; run cells top-to-bottom (later cells depend on
  variables like `image_set` and `rgb_arrays` defined in earlier cells).
- Rendering cells use PyVista off-screen plotting (`off_screen=True`) and write
  PNGs to `Working__stack/`; do not call `plotter.enable_depth_peeling()` — it
  produces all-black output in this off-screen environment (verified).
- Memory is a first-class constraint (images ~0.9 GB as uint8 for 59 slices;
  float32 would be ~3.5 GB):
  - Keep slice arrays as `uint8`; convert to float32 only for small crops.
  - Use lazy/generator-based loading (see `lazy_load_slices` in work1) rather
    than materializing all slices.
  - In the U-Net notebook, `num_workers` must be 0 (Windows + Jupyter
    multiprocessing hangs; each worker copies the ~0.9 GB dataset).
- GPU: training uses CUDA if available; batch size 4 ≈ 2.5 GB VRAM, batch 8
  fits on an 8.5 GB GPU. `torch.manual_seed(0)` is used for reproducibility.
- To evaluate a single change quickly, use small `N_SLICES` (e.g. 10) — the
  parameter appears in the parameter block of both interpolation notebooks.

## Conventions

- Comments, print statements, and docstrings are largely in Chinese; commit
  messages are in English. Keep this convention when editing notebook code.
- Cells are numbered and sequential: each notebook begins with imports, then
  data loading from `E:\working_stack\WT10 0.5mm_72h`, then processing, then
  rendering/export. Don't reorder cells without following the data flow.
- Rendering parameter blocks (RENDER_SCALE, Z_DISPLAY_SCALE, SLICE_OPACITY,
  OUTPUT_PNG) are grouped at the top of rendering cells with inline comments
  explaining each knob.
- The slice coordinate convention: z axis decreases downward, e.g.
  `z(I_a) = -4a` and inserted slice `z(I_in_a) = -4a - 2` after 4× calibration
  scaling; interpolation fitting coordinates (z_calc = 0.5 etc.) stay
  unscaled.
- Interpolation semantics: an inserted slice between I_a and I_{a+1} is fitted
  from four source slices (I_a..I_{a+3}, clamped at the stack ends), computed
  as a per-pixel weighted sum via the Lagrange/Vandermonde shortcut in
  `interp_image_fast` (work2) — mathematically equivalent to per-pixel
  cubic fit, but vectorized over the whole image.
- U-Net training details (U-NET.ipynb): train/test split is by triplet groups
  (57 triplets → 40 train / 17 test), not per-sample, to avoid data leakage;
  SSIM is computed per-image on GPU then averaged (SSIM is nonlinear — never
  average images before scoring); dataset uses index-mapped random crops
  (crop 1024, 20 crops per triplet) instead of pre-generated slices.
- Markdown links and image outputs use relative paths from repo root;
  notebooks hard-code absolute paths (`E:\working_stack\...`, `E:\SRT\...`) —
  update these when running on another machine.

## Git conventions

- Branch `main`; remote `origin` is `git@github.com:lyn14/SRT.git` (SSH).
- Commit messages are short English one-liners describing the algorithmic
  change (e.g. "cubic interpolating", "Remove the downsample procedure").
- Git status currently shows old root-level copies of the notebooks (`work1.ipynb`,
  `work2.ipynb`, `U-NET.ipynb` and output PNGs) as deleted pending commit — the
  live files are under `Programming/` and `Working__stack/`; the reorganization
  commit has not yet been made. (State as of Oct 2026.)
