# Experiment 3 – Frequency-Domain Watermarking with Geometric Alignment

This experiment explores **frequency-domain watermarking** as a strategy to improve
robustness against screenshot-induced geometric distortions such as cropping,
translation, and rescaling.

Unlike Experiments 1 and 2, watermark information is embedded in the **DCT domain**
and detection explicitly compensates for unknown pixel-level misalignment.

---

## Goal of the Experiment

- Achieve robust watermark recovery under screenshot capture
- Tolerate pixel-level misalignment caused by cropping and rescaling
- Maintain full visual invisibility
- Enable exact recovery of a 32-bit payload under blind detection

This experiment introduces:
- Frequency-domain embedding
- Structural redundancy via payload tiling
- Geometry-aware detection using grid-offset search

---

## Payload Construction

The payload is a deterministic 32-bit checksum derived from a string identifier:

```python
h = hashlib.sha256(s.encode("ascii")).digest()
v = int.from_bytes(h[:4], "big")
bits = [(v >> (31 - i)) & 1 for i in range(32)]
```

The payload is intentionally small to allow strong redundancy across frequency blocks.

---

## Embedding Strategy

### Domain

- **Luminance (Y channel)** in YCrCb color space

```python
ycrcb = cv2.cvtColor(img, cv2.COLOR_BGR2YCrCb)
y = cv2.split(ycrcb)[0].astype(float)
```

---

### Block-Based DCT Embedding

- The image is divided into **8×8 blocks**
- Each block is transformed using the Discrete Cosine Transform (DCT)

```python
dct_block = cv2.dct(block)
```

---

### Mid-Frequency Coefficient Selection

Watermark bits are embedded into selected **mid-frequency coefficients**:

```python
dct_block[4, 4] += alpha * sign
dct_block[3, 5] += alpha * sign
```

Rationale:
- Low frequencies → visible artifacts
- High frequencies → unstable under compression/resampling
- Mid frequencies provide the best invisibility–robustness trade-off

---

### Payload Tiling (Structural Redundancy)

The 32-bit payload is embedded **cyclically** across all DCT blocks:

```python
bit = payload_bits[idx % payload_len]
idx += 1
```

This ensures that each payload bit is repeated many times across the image,
providing strong redundancy under cropping and partial screenshots.

---

## Detection

Detection is fully blind and explicitly addresses screenshot-induced misalignment.

---

### Whitening (Content Suppression)

Before detection, low-frequency image content is suppressed:

```python
y_blur = cv2.GaussianBlur(y, (5,5), 0)
y_whitened = y - y_blur
```

This improves the signal-to-noise ratio of the embedded watermark.

---

### Accumulated Energy Detection

For a given grid alignment, watermark energy is accumulated across all blocks:

```python
val = dct_block[4, 4] + dct_block[3, 5]
accumulator[slot] += val
```

Energy is folded into a buffer of size 32 (one slot per payload bit).

---

### Grid-Offset Search (Geometric Alignment)

Screenshot capture may introduce unknown pixel offsets.
To compensate, detection performs a grid search over possible alignments:

```python
for dy in range(0, 8, 2):
    for dx in range(0, 8, 2):
        signal = extract_accumulated_signal(...)
```

The alignment that maximizes mean absolute signal energy is selected.

---

### Bit Recovery

Recovered bits are determined from the sign of the accumulated energy:

```python
bit = 1 if signal > 0 else 0
```

Exact recovery requires all 32 bits to match.

---

## Observed Behavior

- Perfect recovery on unmodified images
- Reliable recovery under moderate cropping and rescaling
- Successful compensation for pixel-level misalignment
- Failures occur under extreme downscaling or very large borders

This represents a significant robustness improvement over spatial-domain methods.

---

## Purpose of This Experiment

This experiment demonstrates that:
- Frequency-domain embedding provides structural robustness
- Payload tiling mitigates cropping and partial capture
- Explicit geometric alignment is critical for screenshot resilience

Together, these mechanisms achieve the highest robustness among all experiments
and validate the frequency-domain approach described in the thesis.
