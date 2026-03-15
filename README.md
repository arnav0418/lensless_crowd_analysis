# Lensless Crowd Analysis

## Overview
This repository contains a prototype pipeline for **privacy‑preserving crowd monitoring** using lensless **Fresnel Zone Aperture (FZA)** imaging, combined with digital reconstruction and post‑processing for downstream analytics (e.g., object tracking, anomaly detection).

**Workflow:** start with a scene video → extract evenly spaced frames → reconstruct each lensless observation to a human‑viewable format → (optionally) reassemble processed frames back into a video for crowd‑analysis models.

The pipeline showcases two complementary reconstruction approaches:
- **Back‑Propagation (BP)** — conventional Fresnel propagation from the sensor to the object plane.
- **Compressed Sensing (CS) via TwIST** — Two‑step Iterative Shrinkage/Thresholding with total‑variation (TV) regularization.

Both algorithms operate on simulated or real lensless captures generated through an **FZA mask**. This reconstruction front‑end can feed downstream models (e.g., **YOLOv4 + DeepSORT** tracking, **CNN–LSTM** anomaly detection) outside this repository.

---

## 📄 Research Paper

**Lensless CCTV Surveillance: Single-Shot Imaging for Privacy-Aware Crowd Monitoring**

📄 [Read the Paper](https://ieeexplore.ieee.org/document/11411024)

---

## Directory Structure
```text
.
├── frame_000000.png               # Example raw frame
├── lensless.m                     # Batch processor for lensless reconstruction
├── lensless_reconstruct.m         # Grayscale reconstruction (BP + CS)
├── lensless_reconstruct_bp_cs.m   # RGB-aware reconstruction wrapper
├── center_crop.m                  # Utility to crop reconstructed frames
├── video_frames_pipeline.py       # Python script to extract/assemble video frames
├── original_script.m              # Reference script demonstrating end-to-end steps
└── functions/                     # MATLAB helper functions
    ├── FZA.m                      # Generates Fresnel Zone Aperture mask
    ├── MyForwardOperatorPropagation.m
    ├── MyAdjointOperatorPropagation.m
    ├── TwIST.m                    # TwIST implementation
    ├── tvdenoise.m, TVnorm.m      # TV regularization helpers
    ├── pinhole.m                  # Pinhole imaging model
    ├── conv2c.m, diffh.m, diffv.m # Convolution & finite differences
    └── (additional utilities)
```

---

## Prerequisites

### MATLAB
- R2019b or later recommended
- Image Processing Toolbox (for `imread`, `imwrite`, etc.)
- Ensure all `.m` files under `functions/` are on MATLAB’s path (scripts call `addpath('./functions')`).

### Python
- Python **3.8+**
- Packages: `opencv-python` (cv2), `argparse`, `pathlib`

```bash
pip install opencv-python
```

---

## Workflow

### 1) Extract Frames from Video
```bash
python video_frames_pipeline.py extract \
    --video input.mp4 \
    --out_dir frames_raw \
    --num 120
```
Saves **120** evenly spaced frames as `frame_XXXXXX.png` in `frames_raw/`.

### 2) Reconstruct Frames (MATLAB)
- Place extracted frames in `../frames_raw/` relative to the MATLAB scripts.
- In MATLAB, run:

```matlab
lensless
```
For each input frame, the script:
1. Loads the image.
2. Executes `lensless_reconstruct` (BP + CS) or the RGB‑aware `lensless_reconstruct_bp_cs`.
3. Crops the results to a **215×300** central patch.
4. Writes two outputs with the same filename to:
   - `../frames_processed_cs/` — **compressed‑sensing** reconstruction.
   - `../frames_processed_bp/` — **back‑propagation** reconstruction.

### 3) Assemble Processed Frames (optional)

```bash
python video_frames_pipeline.py assemble \
    --frames_dir frames_processed_cs \
    --out output.mp4 \
    --fps 30
```

To reuse the original video’s FPS automatically:
```bash
python video_frames_pipeline.py assemble \
    --frames_dir frames_processed_cs \
    --out output.mp4 \
    --ref_video input.mp4
```

The resulting `output.mp4` can serve as input to downstream crowd‑analysis modules (tracking, anomaly detection, etc.).

---

## Reconstruction Details

### FZA Mask Generation
`FZA.m` produces a circular Fresnel Zone Aperture mask. Example snippet:
```matlab
[x, y] = meshgrid(linspace(-S/2, S/2 - S/N, N));
r2   = x.^2 + y.^2;
mask = 0.5 * (1 + cos(pi * r2 / r1^2));
```

### Forward & Adjoint Propagation
- **`MyForwardOperatorPropagation.m`** — applies the Fresnel transfer function in the frequency domain.
- **`MyAdjointOperatorPropagation.m`** — inverse operation for reconstruction.

### TwIST‑based Compressed Sensing
We solve the TV‑regularized problem:

$$
\min_{x}\; \tfrac{1}{2}\,\lVert A x - I \rVert_2^2 + \tau\,\mathrm{TV}(x)
$$

where `A` / `A^T` are propagation operators and `tvdenoise.m`, `TVnorm.m` implement TV denoising and norm. Key parameters include `tau`, `Psi`, `Phi`.

---

## Scripts

### `lensless_reconstruct.m`
Standalone function to reconstruct a **grayscale** observation:
```matlab
[bp, cs] = lensless_reconstruct(I);
```
Returns back‑propagated (`bp`) and TwIST‑based (`cs`) images.

### `lensless_reconstruct_bp_cs.m`
Wrapper handling both **grayscale and RGB** inputs; applies the same pipeline channel‑wise for RGB images.

### `center_crop.m`
Extracts a crop centered at `(H/2, W/2)` with user‑specified height and width.

### `video_frames_pipeline.py`
Utility for frame extraction/assembly. Supports **natural sorting** and **on‑the‑fly resizing** to prevent codec errors.

---

## Example End‑to‑End (MATLAB)
```matlab
img = im2double(imread('frame_000000.png'));
[bp, cs] = lensless_reconstruct(img);
imshow(cs, []); title('Compressed-Sensing Reconstruction');
```

---

## Notes for Research Use
- Reconstruction parameters (`dp`, `di`, `z1`, `tau`, etc.) are tuned for the prototype; adjust for different sensor geometries or noise levels.
- TwIST iterations are fixed at **200** for reproducibility; adjust for desired **quality/speed** trade‑offs.
- For color inputs, `lensless_reconstruct_bp_cs` processes channels independently, preserving per‑channel fidelity.

---

## Acknowledgments
- The TwIST algorithm and utilities are derived from the original implementation by **José Bioucas‑Dias** and **Mário Figueiredo** (GNU **GPLv2**).
- TV denoising routines adapted from **Pascal Getreuer’s** work.

---

## Citation
If this repository or the associated methodology contributes to your research, please cite your corresponding paper and acknowledge the use of this lensless imaging framework.

```bibtex
@article{YourCitation,
  author  = {Arnav Gupta},
  title   = {Lensless Crowd Analysis with FZA Imaging and TwIST Reconstruction},
  journal = {To appear},
  year    = {2025}
}
```

