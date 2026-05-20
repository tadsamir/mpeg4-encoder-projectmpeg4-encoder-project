# Multimedia Mini Project — Report

## Topic
Simplified MPEG-4 video encoder pipeline.

## a) Pipeline description

A short clip of `.png` frames is encoded into one `.bin` file and decoded
back into images.  Each frame travels through:

1. **BGR → YCbCr (BT.601)** and 4:2:0 chroma sub-sampling.  The two
   chroma planes are box-filtered by 2× horizontally and vertically.
2. **Intra coding (I-frames).**  Every `gop`-th frame is coded on its
   own: centre on zero, split each plane into 8×8 tiles, run `cv2.dct`,
   quantise by a JPEG luma/chroma table scaled by the quality factor.
3. **Inter coding (P-frames).**  Every other frame is predicted from
   the previously *reconstructed* frame.  The luma plane is divided
   into 16×16 macroblocks and matched against the reference using a
   **three-step search** (TSS) within a ±S window.  The residual is
   DCT/quantised exactly like an I-frame.  Chroma residuals re-use the
   luma motion field at half resolution.
4. **Entropy coding.**  Each frame's quantised arrays + motion field
   are packed inline with `struct` and the whole blob is `bz2`-compressed
   at level 9, then written as `.bin`.
5. **Evaluation.**  Compression ratio, frame-type counts, per-frame
   PSNR (when an original is provided), and a single matplotlib figure
   are produced.

## b) Design choices

* **YCbCr + 4:2:0** — perceptual sub-sampling that's effectively free
  in quality; standard pre-processing in every modern codec.
* **`cv2.dct`** — extremely fast (BLAS-vectorised under the hood) and
  matches the DCT-II that JPEG/MPEG use.
* **Three-step search** instead of full search.  Loses a tiny amount
  of compression efficiency vs full search but is roughly an order of
  magnitude faster and is the textbook fast-ME algorithm.
* **JPEG quant tables, libjpeg-style scaling.**  Lower frequencies are
  preserved, higher frequencies are aggressively coarsened — matches
  human vision sensitivity.
* **`bz2` instead of `zlib`.**  Block-sorting compression (BWT) gives
  slightly better ratios on highly repetitive quantised-coefficient
  streams than DEFLATE.
* **Inline struct packing.**  No `pickle`: the bitstream is portable
  and language-agnostic — a separate decoder in C/C++ could read it
  with no Python at all.

## c) Experimental analysis

Generated with `python run.py sweep sample_frames -o experiments.png`.

### Compression ratio vs quality

Lower quality scales the quant table up: more coefficients collapse
to zero, `bz2` compresses better, ratio rises.  Quality ≈ 50 gives
the canonical knee of the curve.

### Compression ratio vs GOP

`GOP=1` (all-I) is the worst — no inter-frame redundancy is exploited.
Ratios climb rapidly until ~GOP=8 and then flatten: longer GOPs save
the cost of an extra I-frame but quantisation error drifts further
between resets.

## Reproduce everything

```bash
python gen_test_frames.py -o sample_frames -n 12
python run.py encode sample_frames -o video.bin --gop 8 --q 50
python run.py decode video.bin -o decoded --ref sample_frames
python run.py viz sample_frames video.bin -o pipeline.png
python run.py sweep sample_frames -o experiments.png
```
