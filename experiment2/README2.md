# Experiment 2 – Saliency-Guided Spatial-Domain Invisible Watermark

This experiment extends the baseline spatial-domain watermark (Experiment 1) by
introducing **content-adaptive embedding**. The goal is to improve robustness under
cropping and partial screenshots by concentrating watermark energy in visually and
structurally important image regions.

The approach remains spatial-domain and luminance-based, but embedding strength is
modulated using a saliency (importance) map derived from image gradients.

---

## Goal of the Experiment

- Improve robustness under cropping and partial screenshots
- Preserve full visual invisibility
- Maintain exact recovery of a 32-bit payload under blind detection

**Compared to Experiment 1, this experiment adds:**
- Saliency-guided embedding strength
- Soft aggregation during detection

**It still intentionally avoids:**
- Explicit geometric alignment
- Frequency-domain transforms

---

## Payload and Redundancy

The payload and redundancy scheme are unchanged from Experiment 1:

- Payload: deterministic **32-bit checksum** derived from SHA-256
- Error correction: **Hamming(7,4)**
- Repetition coding with majority / soft voting

```python
ecc_bits, pad = hamming74_encode(payload_bits)
bits_rep = repeat_bits(ecc_bits, cfg.repeat)
```

This allows a direct comparison between uniform and content-adaptive embedding.

---

## Saliency (Importance) Map Generation

A deterministic, gradient-based saliency map is computed from the image content.

```python
gx = cv2.Sobel(gray, cv2.CV_32F, 1, 0)
gy = cv2.Sobel(gray, cv2.CV_32F, 0, 1)
magnitude = cv2.magnitude(gx, gy)
heatmap = cv2.GaussianBlur(magnitude, (51, 51), 0)
```

Steps:
1. Convert image to grayscale
2. Compute gradient magnitude (edge / texture strength)
3. Apply strong Gaussian blur to form region-level importance
4. Normalize to the range [0.0, 1.0]

High values correspond to textured or structurally important regions; flat backgrounds
receive low values.

---

## Embedding Strategy

### Domain

- **Luminance (Y channel)** in YCrCb color space

```python
ycrcb = cv2.cvtColor(img_bgr, cv2.COLOR_BGR2YCrCb)
y = cv2.split(ycrcb)[0].astype(np.float32)
```

---

### Spatial Structure

- The image width is divided into vertical tiles
- Each repeated ECC bit is embedded into one tile

```python
tile_w = W // n_bits
```

---

### Saliency-Guided Modulation

Embedding strength is modulated by the local saliency map.

```python
weight_map = np.maximum(roi_heatmap, 0.3)
delta = alpha * direction * noise * weight_map
```

Key design choices:
- High-saliency regions receive stronger watermark energy
- A **minimum-strength floor (0.3)** ensures that watermark bits are not completely
  lost in flat regions
- This balances robustness to cropping with global coverage

---

### Noise Modulation

Each bit modulates the sign of a pseudorandom noise pattern:

```python
noise = gen_noise(H, tile_w, cfg.seed + i)
direction = +1 if bit == 1 else -1
```

The watermark remains visually imperceptible despite increased robustness.

---

## Detection

Detection is fully blind and mirrors the embedding logic exactly.

### Correlation-Based Extraction

```python
raw_score = np.sum(roi * noise)
```

Instead of hard decisions per tile, **raw correlation scores** are retained.

---

### Soft Aggregation (Repetition Code)

```python
block_score = sum(chunk)
bit = 1 if block_score > 0 else 0
```

Summing correlation responses across repetitions improves robustness under noise
and partial signal loss.

---

### ECC Decoding

```python
decoded_payload = hamming74_decode(final_ecc_bits, pad)
```

Exact recovery requires all 32 payload bits to match.

---

## Observed Behavior

- Perfect recovery on the unmodified watermarked image
- Improved robustness under cropping and partial screenshots
- Failures still occur under rescaling and pixel-level misalignment

**Key insight:**
Content adaptivity improves robustness to *content loss*, but does not solve
*geometric misalignment*.

---

## Purpose of This Experiment

This experiment demonstrates that:
- Allocating watermark energy to salient regions improves robustness
- Spatial-domain watermarking remains sensitive to rescaling and translation

These limitations motivate the frequency-domain, geometry-aware approach explored
in **Experiment 3**.
