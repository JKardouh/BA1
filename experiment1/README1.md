# Experiment 1 – Baseline Spatial-Domain Invisible Watermark

This experiment implements a minimal spatial-domain invisible watermark as a baseline.
Its purpose is to demonstrate how classical spread-spectrum watermarking behaves under
real screenshot capture and to expose its fundamental failure modes.

The focus is not on maximum robustness, but on clarity and isolation of effects.

---

## Goal of the Experiment

- Embed a fully invisible watermark into a standard RGB image
- Enable exact recovery of a 32-bit payload under blind detection
- Evaluate robustness under screenshot-induced distortions

**This experiment intentionally avoids:**
- Content-adaptive embedding
- Geometric alignment or registration

---

## Payload Construction

An arbitrary identifier string is mapped to a deterministic 32-bit checksum.
SHA-256 is used, and only the first 32 bits are kept:

```python
h = hashlib.sha256(s.encode("ascii")).digest()
v = int.from_bytes(h[:4], "big")
```

The payload is intentionally small to allow strong redundancy rather than high capacity.

---

## Error Correction and Redundancy

To support exact recovery under noise and distortion:

- The 32-bit payload is encoded using **Hamming(7,4)**
- Each encoded bit is repeated multiple times
- Majority voting is applied during detection

```python
ecc_bits, pad = hamming74_encode(bits32)
bits_rep = repeat_bits(ecc_bits, cfg.repeat)
```

This layered redundancy mirrors the design described in the thesis.

---

## Embedding Strategy

### Domain

- **Luminance (Y channel)**, derived from RGB

```python
y = 0.299*r + 0.587*g + 0.114*b
```

Only luminance is modified to minimize visible color artifacts.

---

### Spatial Structure

- Images are resized to a fixed base resolution
- The image is divided into **vertical tiles**
- Only the **central vertical band** is modified to avoid borders

```python
tile_w = W // n_embed_bits
y0 = H // 4
y1 = 3 * H // 4
```

Each repeated ECC bit is embedded into one vertical tile.

---

### Pseudorandom Noise Generation

For each tile, a deterministic pseudorandom noise pattern is generated:

```python
rng = np.random.RandomState(seed)
noise = rng.choice([-1.0, 1.0], size=(h, w))
noise = (noise - noise.mean()) / noise.std()
```

- Zero mean
- Unit variance
- Seeded per tile to allow blind detection

---

### Modulation

Each repeated bit modulates the sign of the noise pattern:

```python
m = +1 if bit == 1 else -1
y_mod[y0:y1, x0:x1] += alpha * m * noise
```

**Where:**
- **m ∈ {+1, −1}** encodes the bit value
- **noise** is the pseudorandom pattern
- **alpha** controls embedding strength

The watermark remains visually imperceptible at all tested strengths.

---

## Detection

Detection is fully blind and assumes no access to the original image.

**Steps:**
1. Resize the input image to the base resolution
2. Extract luminance
3. Re-generate the expected PN patterns
4. Compute correlation per tile
5. Recover bit values from correlation sign
6. Apply majority voting
7. Decode with Hamming(7,4)

---

### Correlation-Based Bit Recovery

```python
corr = (tile * noise).sum() / (||tile|| * ||noise||)
bit = 1 if corr > 0 else 0
```

The sign of the normalized correlation determines the recovered bit.

---

### Majority Voting and ECC Decoding

```python
groups = [bits_obs[i*r:(i+1)*r] for i in range(ecc_len)]
voted = majority_vote(groups)
decoded_bits, _ = hamming74_decode(voted, pad)
```

Exact recovery requires all 32 bits to match.

---

## Observed Behavior

- Perfect recovery on the saved watermarked image
- Frequent failures after screenshot capture

**Primary failure causes:**
- Cropping removes entire tiles
- Rescaling introduces pixel-level misalignment
- Correlation decorrelates despite ECC and repetition

This confirms that uniform spatial-domain watermarking is highly sensitive to geometric distortion.

---

## Purpose of This Baseline

This experiment serves as a reference point for:
- Saliency-guided spatial embedding (Experiment 2)
- Frequency-domain watermarking with geometric alignment (Experiment 3)

It demonstrates why additional robustness mechanisms are required for
screenshot-resilient invisible watermarking.
