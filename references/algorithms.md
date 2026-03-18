# Dithering & Quantization Algorithms

Complete mathematical foundations and implementations for all 14 dithering algorithms,
halftone patterns, and multi-level quantization. Implementations provided in TypeScript
(web), Python/NumPy (video), and GLSL/WGSL (GPU shaders).

Research basis:
- Bayer (1973): Recursive threshold matrix via bit-interleaving
- Floyd & Steinberg (1976): 4-neighbor error diffusion
- Jarvis, Judice & Ninke (1976): 12-neighbor diffusion
- Stucki (1981): Modified JJN with sharper edges
- Atkinson (1984): 75% error propagation for Mac aesthetics
- Sierra (1989): 3 variants (full, two-row, lite)
- Ulichney (1987, 1993): Blue noise and void-and-cluster
- Riemersma (2000): Hilbert-curve space-filling diffusion
- Krahmer & Veselovska (2024): Mathematical foundations of halftoning [arXiv:2406.12760]
- Jiang et al. (2023): RL-based halftoning quality bounds [arXiv:2304.12152]

---

## 1. Mathematical Foundations

### Quantization

Dithering is controlled distortion added before quantization to break up banding artifacts.
For an input value `v ∈ [0,1]` and `N` output levels:

```
quantize(v, N) = round(v × (N-1)) / (N-1)
```

For binary (1-bit) quantization: `quantize(v) = v < 0.5 ? 0 : 1`

The **quantization error** `e = v - quantize(v)` is what dithering algorithms redistribute.

### Why Dithering Works (Spectral Analysis)

Human vision acts as a low-pass filter — we're sensitive to low-frequency patterns (banding,
contours) but relatively insensitive to high-frequency noise (grain, stipple). Dithering
pushes quantization error into high frequencies where it becomes invisible.

The **power spectral density** (PSD) of the error pattern characterizes dithering quality:
- **White noise**: flat PSD → visible grain at all frequencies
- **Blue noise**: PSD ∝ f² → energy concentrated at high frequencies → least visible
- **Ordered dithering**: PSD has peaks at matrix frequency → visible grid pattern
- **Error diffusion**: PSD approximates blue noise → good quality

### Perceptual Quality Metrics

From Jiang et al. (2023), quality ranking by SSIM against Direct Binary Search (optimal):

```
DBS > Stucki ≈ JJN > Sierra > Floyd-Steinberg > Bayer 8×8 > Atkinson > Threshold
```

DBS is optimal but O(n²) — impractical for real-time. Error diffusion approximates DBS
at O(n) cost.

---

## 2. Ordered Dithering (Bayer Matrices)

### Recursive Construction

The Bayer matrix of order `n` is constructed recursively from the 2×2 base:

```
M₁ = | 0  2 |
     | 3  1 |

Mₙ = | 4·Mₙ₋₁ + 0    4·Mₙ₋₁ + 2 |
     | 4·Mₙ₋₁ + 3    4·Mₙ₋₁ + 1 |
```

This generates threshold matrices of size `2ⁿ × 2ⁿ`. The normalized threshold at
position `(x, y)` is: `T(x,y) = M[y % size][x % size] / (size × size)`

Equivalently, via bit-interleaving (Bayer 1973): for an `n×n` matrix where `n = 2^k`,
the value at `(x, y)` can be computed by interleaving the bits of `x ⊕ y` and `y`:

```
bayer(x, y, bits) = reverse(interleave(x ^ y, y, bits))
```

### 2×2 Matrix

```typescript
const BAYER_2X2 = [
  [0, 2],
  [3, 1],
]
```

4 threshold levels. Creates coarse cross-hatch. Use for pixel-art / 1-bit game aesthetic.

### 4×4 Matrix

```typescript
const BAYER_4X4 = [
  [ 0,  8,  2, 10],
  [12,  4, 14,  6],
  [ 3, 11,  1,  9],
  [15,  7, 13,  5],
]
```

16 threshold levels. Good for background textures and subtle dither overlays.

### 8×8 Matrix

```typescript
const BAYER_8X8 = [
  [ 0, 32,  8, 40,  2, 34, 10, 42],
  [48, 16, 56, 24, 50, 18, 58, 26],
  [12, 44,  4, 36, 14, 46,  6, 38],
  [60, 28, 52, 20, 62, 30, 54, 22],
  [ 3, 35, 11, 43,  1, 33,  9, 41],
  [51, 19, 59, 27, 49, 17, 57, 25],
  [15, 47,  7, 39, 13, 45,  5, 37],
  [63, 31, 55, 23, 61, 29, 53, 21],
]
```

64 threshold levels. Best general-purpose ordered dithering. The standard for retro effects
and animation (spatially fixed thresholds → no temporal flicker).

### TypeScript Implementation

```typescript
function bayerDither(
  pixels: Float32Array,
  w: number, h: number,
  matrix: number[][],
  strength: number
): Float32Array {
  const size = matrix.length
  const levels = size * size
  const output = new Float32Array(pixels.length)

  for (let y = 0; y < h; y++) {
    for (let x = 0; x < w; x++) {
      const idx = y * w + x
      const threshold = (matrix[y % size][x % size] / levels - 0.5) * strength
      output[idx] = Math.max(0, Math.min(1, pixels[idx] + threshold))
    }
  }
  return output
}
```

The `% size` wraps coordinates into the tile, creating seamless repetition. Unlike error
diffusion which produces binary 0/1, Bayer preserves continuous values — the threshold
offsets brightness, and the char mapper handles final quantization.

### GLSL Implementation

```glsl
// Bayer 8x8 as a function (no texture lookup needed)
float bayer8x8(ivec2 pos) {
  int x = pos.x & 7;
  int y = pos.y & 7;
  // Bit-interleave computation
  int v = 0;
  for (int i = 0; i < 3; i++) {
    int xb = (x >> i) & 1;
    int yb = (y >> i) & 1;
    v |= ((xb ^ yb) << (2 * i + 1)) | (yb << (2 * i));
  }
  return float(v) / 64.0;
}

// Apply ordered dithering in fragment shader
vec4 orderedDither(vec2 uv, sampler2D tex, float strength) {
  vec4 color = texture(tex, uv);
  float lum = dot(color.rgb, vec3(0.2126, 0.7152, 0.0722));
  ivec2 pos = ivec2(gl_FragCoord.xy);
  float threshold = bayer8x8(pos);
  float dithered = lum + (threshold - 0.5) * strength;
  return vec4(vec3(dithered), color.a);
}
```

### WGSL Implementation

```wgsl
fn bayer8x8(pos: vec2<u32>) -> f32 {
  let x = pos.x & 7u;
  let y = pos.y & 7u;
  var v: u32 = 0u;
  for (var i: u32 = 0u; i < 3u; i = i + 1u) {
    let xb = (x >> i) & 1u;
    let yb = (y >> i) & 1u;
    v = v | ((xb ^ yb) << (2u * i + 1u)) | (yb << (2u * i));
  }
  return f32(v) / 64.0;
}

@fragment
fn dither_fragment(@builtin(position) pos: vec4<f32>) -> @location(0) vec4<f32> {
  let uv = pos.xy / uniforms.resolution;
  let color = textureSample(input_texture, tex_sampler, uv);
  let lum = dot(color.rgb, vec3<f32>(0.2126, 0.7152, 0.0722));
  let threshold = bayer8x8(vec2<u32>(pos.xy));
  let dithered = lum + (threshold - 0.5) * uniforms.strength;
  return vec4<f32>(vec3<f32>(clamp(dithered, 0.0, 1.0)), color.a);
}
```

### Python/NumPy Implementation

```python
import numpy as np

def make_bayer(n: int) -> np.ndarray:
    """Generate Bayer matrix of size 2^n × 2^n recursively."""
    if n == 0:
        return np.array([[0]])
    smaller = make_bayer(n - 1)
    size = smaller.shape[0]
    result = np.zeros((size * 2, size * 2), dtype=np.int32)
    result[:size, :size] = 4 * smaller + 0
    result[:size, size:] = 4 * smaller + 2
    result[size:, :size] = 4 * smaller + 3
    result[size:, size:] = 4 * smaller + 1
    return result

BAYER_8 = make_bayer(3)  # 8×8

def bayer_dither(image: np.ndarray, matrix: np.ndarray, strength: float = 1.0) -> np.ndarray:
    """Apply ordered dithering. image: float32 H×W [0,1]. Returns float32 H×W [0,1]."""
    h, w = image.shape
    size = matrix.shape[0]
    levels = size * size
    # Tile the threshold matrix across the image
    ty = np.arange(h) % size
    tx = np.arange(w) % size
    threshold = matrix[np.ix_(ty, tx)].astype(np.float32) / levels - 0.5
    return np.clip(image + threshold * strength, 0, 1)
```

---

## 3. Error Diffusion Framework

Error diffusion algorithms share a common structure: scan pixels left-to-right, top-to-bottom,
quantize each pixel, then distribute the quantization error to neighboring unprocessed pixels
according to a kernel of weights.

### General Framework

```typescript
interface DiffusionKernel {
  offsets: Array<[number, number, number]>  // [dx, dy, weight]
  divisor: number
}

function errorDiffusion(
  pixels: Float32Array, w: number, h: number,
  strength: number, kernel: DiffusionKernel, levels: number = 2
): Float32Array {
  const output = new Float32Array(pixels.length)
  output.set(pixels)
  const step = 1 / (levels - 1)

  for (let y = 0; y < h; y++) {
    // Serpentine scanning reduces directional artifacts
    const leftToRight = (y & 1) === 0
    const xStart = leftToRight ? 0 : w - 1
    const xEnd = leftToRight ? w : -1
    const xStep = leftToRight ? 1 : -1

    for (let x = xStart; x !== xEnd; x += xStep) {
      const idx = y * w + x
      const oldVal = output[idx]

      // Multi-level quantization
      const newVal = Math.round(oldVal / step) * step
      output[idx] = Math.max(0, Math.min(1, newVal))

      const error = (oldVal - newVal) * strength

      for (const [dx, dy, weight] of kernel.offsets) {
        const nx = x + dx * xStep  // flip kernel on odd rows
        const ny = y + dy
        if (nx >= 0 && nx < w && ny >= 0 && ny < h) {
          output[ny * w + nx] += (error * weight) / kernel.divisor
        }
      }
    }
  }
  return output
}
```

Key details:
- **Serpentine scanning** (alternating L→R and R→L per row) reduces the directional drift
  artifacts that plague left-to-right-only scanning. Mirror the kernel offsets on odd rows.
- **Multi-level quantization**: `levels=2` gives binary, `levels=4` gives 4-tone, etc.
  More levels = less error = subtler dithering.
- **Strength** controls error propagation: 0 = no dithering, 1 = full, >1 = exaggerated.

### GLSL Error Diffusion (Approximation)

True error diffusion is inherently serial (each pixel depends on prior results), making it
GPU-unfriendly. Two practical approaches:

**1. Multi-pass with shared memory** (compute shader):
```glsl
// Compute shader — process one row at a time, sync between rows
layout(local_size_x = 256) in;

shared float row_errors[MAX_WIDTH];

void main() {
  uint x = gl_LocalInvocationID.x;
  uint y = gl_WorkGroupID.x;
  // Each workgroup processes one row
  float val = imageLoad(input_img, ivec2(x, y)).r + row_errors[x];
  float quantized = step(0.5, val);
  float error = val - quantized;

  barrier();
  // Distribute error to neighbors
  if (x + 1 < width) atomicAdd(row_errors[x + 1], error * 7.0 / 16.0);
  // Cross-row errors written to a buffer for the next workgroup
  // ...
}
```

**2. Ordered dithering as GPU-friendly substitute**: for real-time GPU rendering, use
Bayer or blue noise instead — they're embarrassingly parallel and produce comparable quality.

---

## 4. Error Diffusion Kernels

### Floyd-Steinberg (1976)

The classic. 4 neighbors, divisor 16. Good balance of quality and speed.

```
        * 7
    3 5 1
    ÷ 16
```

```typescript
const FLOYD_STEINBERG: DiffusionKernel = {
  offsets: [[1,0,7], [-1,1,3], [0,1,5], [1,1,1]],
  divisor: 16,
}
```

### Jarvis-Judice-Ninke (1976)

12 neighbors across 2 rows. Smoothest gradients of any diffusion algorithm. ~3× slower
than Floyd-Steinberg due to wider kernel.

```
            * 7 5
    3 5 7 5 3
    1 3 5 3 1
        ÷ 48
```

```typescript
const JJN: DiffusionKernel = {
  offsets: [
    [1,0,7], [2,0,5],
    [-2,1,3], [-1,1,5], [0,1,7], [1,1,5], [2,1,3],
    [-2,2,1], [-1,2,3], [0,2,5], [1,2,3], [2,2,1],
  ],
  divisor: 48,
}
```

### Stucki (1981)

Same spread as JJN but with different weights. Sharper edges, crisper midtones.
The choice between Stucki and JJN is aesthetic — Stucki for technical images, JJN for photos.

```
            * 8 4
    2 4 8 4 2
    1 2 4 2 1
        ÷ 42
```

```typescript
const STUCKI: DiffusionKernel = {
  offsets: [
    [1,0,8], [2,0,4],
    [-2,1,2], [-1,1,4], [0,1,8], [1,1,4], [2,1,2],
    [-2,2,1], [-1,2,2], [0,2,4], [1,2,2], [2,2,1],
  ],
  divisor: 42,
}
```

### Burkes (1988)

Modified Stucki with only 1 row of diffusion (not 2). Faster than JJN/Stucki while
maintaining most of their quality.

```
        * 8 4
    2 4 8 4 2
        ÷ 32
```

```typescript
const BURKES: DiffusionKernel = {
  offsets: [
    [1,0,8], [2,0,4],
    [-2,1,2], [-1,1,4], [0,1,8], [1,1,4], [2,1,2],
  ],
  divisor: 32,
}
```

### Atkinson (1984)

Propagates only 6/8 (75%) of error. The "lost" 25% creates high-contrast, graphic aesthetic
associated with classic Macintosh. Faster than Floyd-Steinberg due to uniform weights.

```
        * 1 1
    1 1 1
        1
    ÷ 8 (only 6/8 propagated)
```

```typescript
const ATKINSON: DiffusionKernel = {
  offsets: [[1,0,1], [2,0,1], [-1,1,1], [0,1,1], [1,1,1], [0,2,1]],
  divisor: 8,  // sum of weights is 6, but divisor is 8 → 25% error is absorbed
}
```

### Sierra (Full, 1989)

10 neighbors. Very smooth gradients, especially for photographic content. Wide kernel
preserves tonal accuracy across smooth regions.

```
            * 5 3
    2 4 5 4 2
        2 3 2
        ÷ 32
```

```typescript
const SIERRA: DiffusionKernel = {
  offsets: [
    [1,0,5], [2,0,3],
    [-2,1,2], [-1,1,4], [0,1,5], [1,1,4], [2,1,2],
    [-1,2,2], [0,2,3], [1,2,2],
  ],
  divisor: 32,
}
```

### Sierra Two-Row

Lighter version of Sierra — drops the third row for speed while keeping most of the quality.

```
        * 4 3
    1 2 3 2 1
        ÷ 16
```

```typescript
const SIERRA_TWO_ROW: DiffusionKernel = {
  offsets: [
    [1,0,4], [2,0,3],
    [-2,1,1], [-1,1,2], [0,1,3], [1,1,2], [2,1,1],
  ],
  divisor: 16,
}
```

### Sierra-Lite

Minimal 3-neighbor kernel. Fastest diffusion algorithm. Slightly noisier than Floyd-Steinberg
but nearly 2× faster for large grids. Use for real-time video.

```
    * 2
    1 1
    ÷ 4
```

```typescript
const SIERRA_LITE: DiffusionKernel = {
  offsets: [[1,0,2], [0,1,1], [-1,1,1]],
  divisor: 4,
}
```

### Kernel Registry

```typescript
const DIFFUSION_KERNELS: Record<string, DiffusionKernel> = {
  'floyd-steinberg': FLOYD_STEINBERG,
  'jarvis-judice-ninke': JJN,
  'stucki': STUCKI,
  'burkes': BURKES,
  'atkinson': ATKINSON,
  'sierra': SIERRA,
  'sierra-two-row': SIERRA_TWO_ROW,
  'sierra-lite': SIERRA_LITE,
}
```

---

## 5. Blue Noise Dithering

Blue noise is the gold standard for perceptual quality. The error pattern has energy
concentrated at high spatial frequencies (blue = high frequency), matching human visual
system insensitivity. No visible structure, no grid artifacts, no directional bias.

### Theory (Ulichney 1987, Wolfe et al. 2021)

A blue noise pattern has:
- **Radially averaged PSD** that increases with frequency: `P(f) ∝ f^α` where `α ≈ 1-2`
- **No low-frequency energy** → no visible clustering or voids
- **Isotropic** → no directional preference

### Pre-computed Blue Noise Texture

The void-and-cluster algorithm generates blue noise textures. For real-time use, pre-compute
a 64×64 or 128×128 tile and tile it across the image:

```typescript
// Pre-computed 64×64 blue noise texture (store as Float32Array or Uint8Array)
// Generate offline using void-and-cluster, then embed as a constant.
// Many free blue noise textures available from Christoph Peters' research.

function blueNoiseDither(
  pixels: Float32Array, w: number, h: number,
  blueNoise: Float32Array, noiseSize: number,
  strength: number
): Float32Array {
  const output = new Float32Array(pixels.length)
  for (let y = 0; y < h; y++) {
    for (let x = 0; x < w; x++) {
      const idx = y * w + x
      const noiseIdx = (y % noiseSize) * noiseSize + (x % noiseSize)
      const threshold = (blueNoise[noiseIdx] - 0.5) * strength
      output[idx] = Math.max(0, Math.min(1, pixels[idx] + threshold))
    }
  }
  return output
}
```

### Generating Blue Noise (Void-and-Cluster)

```python
import numpy as np
from scipy.ndimage import gaussian_filter

def generate_blue_noise(size: int = 64) -> np.ndarray:
    """Void-and-cluster algorithm for blue noise generation."""
    # Initialize with ~10% random white pixels
    density = 0.1
    pattern = (np.random.random((size, size)) < density).astype(np.float32)

    # Iteratively move the tightest cluster to the largest void
    for _ in range(size * size * 2):
        # Gaussian-filtered version reveals clusters and voids
        filtered = gaussian_filter(pattern, sigma=1.5, mode='wrap')

        # Find tightest cluster (brightest in filtered among set pixels)
        set_mask = pattern > 0.5
        if not set_mask.any():
            break
        cluster_vals = np.where(set_mask, filtered, -np.inf)
        cluster_pos = np.unravel_index(np.argmax(cluster_vals), pattern.shape)

        # Find largest void (darkest in filtered among unset pixels)
        void_mask = pattern < 0.5
        if not void_mask.any():
            break
        void_vals = np.where(void_mask, filtered, np.inf)
        void_pos = np.unravel_index(np.argmin(void_vals), pattern.shape)

        # Move pixel from cluster to void
        pattern[cluster_pos] = 0
        pattern[void_pos] = 1

    # Convert binary pattern to threshold values by ranking
    result = np.zeros_like(pattern)
    flat = pattern.flatten()
    # Assign ranks based on the order pixels were placed
    # (simplified — full algorithm tracks insertion order)
    filtered_final = gaussian_filter(pattern, sigma=1.0, mode='wrap')
    ranks = np.argsort(filtered_final.flatten())
    for rank, pos in enumerate(ranks):
        result.flat[pos] = rank / (size * size)

    return result
```

### Spatiotemporal Blue Noise (Video)

From Wolfe et al. (2021): for temporally coherent dithering across video frames, use
3D blue noise masks — noise values that are blue-noise-distributed in both space AND time.

```typescript
// Temporal offset per frame using golden ratio for low-discrepancy sequence
function temporalBlueNoise(
  blueNoise: Float32Array, noiseSize: number,
  x: number, y: number, frame: number
): number {
  const noiseIdx = (y % noiseSize) * noiseSize + (x % noiseSize)
  // Golden ratio temporal offset (R₂ sequence) — minimizes frame-to-frame correlation
  const phi = 0.6180339887498949  // (√5 - 1) / 2
  return (blueNoise[noiseIdx] + frame * phi) % 1.0
}
```

### GLSL Blue Noise Dithering

```glsl
uniform sampler2D u_blueNoise;  // pre-computed blue noise texture
uniform float u_strength;
uniform float u_frame;

vec4 blueNoiseDither(vec2 uv, sampler2D tex) {
  vec4 color = texture(tex, uv);
  float lum = dot(color.rgb, vec3(0.2126, 0.7152, 0.0722));

  // Sample blue noise with golden ratio temporal offset
  vec2 noiseUV = gl_FragCoord.xy / vec2(textureSize(u_blueNoise, 0));
  float noise = texture(u_blueNoise, noiseUV).r;
  float phi = 0.6180339887;
  noise = fract(noise + u_frame * phi);  // temporal decorrelation

  float dithered = lum + (noise - 0.5) * u_strength;
  return vec4(vec3(clamp(dithered, 0.0, 1.0)), color.a);
}
```

---

## 6. Simple Dithering Methods

### Threshold (1-bit quantization)

Binary cutoff with no error redistribution. Harsh but fast.

```typescript
function thresholdDither(
  pixels: Float32Array, w: number, h: number,
  threshold: number = 0.5
): Float32Array {
  const output = new Float32Array(pixels.length)
  for (let i = 0; i < pixels.length; i++) {
    output[i] = pixels[i] < threshold ? 0 : 1
  }
  return output
}
```

### Random (White Noise)

Each pixel gets independent random threshold. Produces visible grain but no structural patterns.

```typescript
function randomDither(
  pixels: Float32Array, w: number, h: number,
  strength: number
): Float32Array {
  const output = new Float32Array(pixels.length)
  for (let i = 0; i < pixels.length; i++) {
    const noise = (Math.random() - 0.5) * strength
    output[i] = Math.max(0, Math.min(1, pixels[i] + noise))
  }
  return output
}
```

For deterministic results (required for animation), use a seeded PRNG:

```typescript
function seededRandom(seed: number): number {
  seed = (seed * 1103515245 + 12345) & 0x7fffffff
  return seed / 0x7fffffff
}

function randomDitherSeeded(
  pixels: Float32Array, w: number, h: number,
  strength: number, frameSeed: number
): Float32Array {
  const output = new Float32Array(pixels.length)
  for (let y = 0; y < h; y++) {
    for (let x = 0; x < w; x++) {
      const idx = y * w + x
      const seed = (x * 2654435761 + y * 340573321 + frameSeed) & 0x7fffffff
      const noise = (seededRandom(seed) - 0.5) * strength
      output[idx] = Math.max(0, Math.min(1, pixels[idx] + noise))
    }
  }
  return output
}
```

---

## 7. Riemersma Dithering

Space-filling curve dithering (Hilbert or Riemersma curve). Distributes error along a
fractal path rather than scanline order, eliminating directional bias artifacts.

The key insight: error diffusion along a space-filling curve produces isotropic results
because the curve visits neighbors in all directions equally.

```typescript
// Hilbert curve coordinate generator
function* hilbertCurve(
  x: number, y: number, ax: number, ay: number,
  bx: number, by: number
): Generator<[number, number]> {
  const w = Math.abs(ax + ay)
  const h = Math.abs(bx + by)
  const dax = Math.sign(ax), day = Math.sign(ay)
  const dbx = Math.sign(bx), dby = Math.sign(by)

  if (h === 1) {
    for (let i = 0; i < w; i++) {
      yield [x, y]
      x += dax; y += day
    }
    return
  }
  if (w === 1) {
    for (let i = 0; i < h; i++) {
      yield [x, y]
      x += dbx; y += dby
    }
    return
  }

  const ax2 = Math.floor(ax / 2), ay2 = Math.floor(ay / 2)
  const bx2 = Math.floor(bx / 2), by2 = Math.floor(by / 2)

  const w2 = Math.abs(ax2 + ay2)
  const h2 = Math.abs(bx2 + by2)

  if (2 * w > 3 * h) {
    if ((w2 & 1) && w > 2) {
      // Prefer even steps
      yield* hilbertCurve(x, y, ax2 + dax, ay2 + day, bx, by)
      yield* hilbertCurve(x + ax2 + dax, y + ay2 + day,
                          ax - ax2 - dax, ay - ay2 - day, bx, by)
    } else {
      yield* hilbertCurve(x, y, ax2, ay2, bx, by)
      yield* hilbertCurve(x + ax2, y + ay2, ax - ax2, ay - ay2, bx, by)
    }
  } else {
    if ((h2 & 1) && h > 2) {
      yield* hilbertCurve(x, y, bx2 + dbx, by2 + dby, ax2, ay2)
      yield* hilbertCurve(x + bx2 + dbx, y + by2 + dby,
                          ax, ay, bx - bx2 - dbx, by - by2 - dby)
      yield* hilbertCurve(x + (ax - dax) + (bx2 + dbx),
                          y + (ay - day) + (by2 + dby),
                          -(bx2 + dbx), -(by2 + dby), ax2, ay2)
    } else {
      yield* hilbertCurve(x, y, bx2, by2, ax2, ay2)
      yield* hilbertCurve(x + bx2, y + by2, ax, ay, bx - bx2, by - by2)
      yield* hilbertCurve(x + (ax - dax) + bx2, y + (ay - day) + by2,
                          -bx2, -by2, ax2, ay2)
    }
  }
}

function riemersmaDither(
  pixels: Float32Array, w: number, h: number,
  strength: number, historyLength: number = 16
): Float32Array {
  const output = new Float32Array(pixels.length)
  output.set(pixels)

  // Error history with exponential decay
  const history: number[] = new Array(historyLength).fill(0)
  let histIdx = 0

  for (const [x, y] of hilbertCurve(0, 0, w, 0, 0, h)) {
    if (x < 0 || x >= w || y < 0 || y >= h) continue
    const idx = y * w + x

    // Add decayed historical error
    let errorSum = 0
    for (let i = 0; i < historyLength; i++) {
      const age = (histIdx - i + historyLength) % historyLength
      errorSum += history[i] * Math.exp(-age * 0.2) * strength
    }

    const val = output[idx] + errorSum / historyLength
    const quantized = val < 0.5 ? 0 : 1
    output[idx] = quantized

    history[histIdx % historyLength] = val - quantized
    histIdx++
  }

  return output
}
```

---

## 8. Multi-Level Quantization

For color or multi-tone dithering (not just binary). Quantize to N levels instead of 2.

```typescript
function multiLevelDither(
  pixels: Float32Array, w: number, h: number,
  algorithm: string, strength: number, levels: number
): Float32Array {
  if (algorithm === 'bayer' || algorithm === 'blue-noise') {
    // Ordered methods: offset then quantize
    const dithered = algorithm === 'bayer'
      ? bayerDither(pixels, w, h, BAYER_8X8, strength)
      : blueNoiseDither(pixels, w, h, blueNoiseTexture, 64, strength)

    const output = new Float32Array(dithered.length)
    const step = 1 / (levels - 1)
    for (let i = 0; i < dithered.length; i++) {
      output[i] = Math.round(dithered[i] / step) * step
    }
    return output
  }

  // Error diffusion with multi-level quantization
  const kernel = DIFFUSION_KERNELS[algorithm] ?? FLOYD_STEINBERG
  return errorDiffusion(pixels, w, h, strength, kernel, levels)
}
```

---

## 9. Halftone Patterns

Classical print halftoning uses angled grids of variable-size dots. CMYK printing uses
4 screens at different angles to avoid moiré.

### CMYK Screen Angles

```typescript
const CMYK_ANGLES = {
  cyan:    15,   // degrees
  magenta: 75,
  yellow:  0,
  black:   45,
}
```

### Halftone Dot Generation

```typescript
function halftonePattern(
  x: number, y: number,
  angle: number, frequency: number,
  brightness: number
): boolean {
  const rad = angle * Math.PI / 180
  const cos = Math.cos(rad), sin = Math.sin(rad)

  // Rotate coordinates into screen space
  const rx = x * cos + y * sin
  const ry = -x * sin + y * cos

  // Compute position within halftone cell
  const cellX = (rx * frequency) % 1
  const cellY = (ry * frequency) % 1

  // Distance from cell center determines dot coverage
  const cx = cellX - 0.5
  const cy = cellY - 0.5
  const dist = Math.sqrt(cx * cx + cy * cy)

  // Dot radius proportional to brightness (area coverage)
  const radius = Math.sqrt(brightness) * 0.5  // √ for perceptual linearity

  return dist < radius
}
```

### GLSL Halftone Shader

```glsl
uniform float u_frequency;
uniform float u_angle;

float halftone(vec2 pos, float brightness) {
  float rad = u_angle * 3.14159 / 180.0;
  mat2 rot = mat2(cos(rad), sin(rad), -sin(rad), cos(rad));
  vec2 rotated = rot * pos * u_frequency;
  vec2 cell = fract(rotated) - 0.5;
  float dist = length(cell);
  float radius = sqrt(brightness) * 0.5;
  return smoothstep(radius + 0.02, radius - 0.02, dist);
}
```

### Halftone Shape Variations

```glsl
// Circle (default)
float halftoneCircle(vec2 cell, float radius) {
  return smoothstep(radius + 0.02, radius - 0.02, length(cell));
}

// Square
float halftoneSquare(vec2 cell, float radius) {
  vec2 d = abs(cell);
  float sq = max(d.x, d.y);
  return smoothstep(radius + 0.02, radius - 0.02, sq);
}

// Diamond
float halftoneDiamond(vec2 cell, float radius) {
  float d = abs(cell.x) + abs(cell.y);
  return smoothstep(radius * 1.4 + 0.02, radius * 1.4 - 0.02, d);
}

// Line
float halftoneLine(vec2 cell, float radius) {
  return smoothstep(radius + 0.02, radius - 0.02, abs(cell.y));
}

// Ellipse
float halftoneEllipse(vec2 cell, float radius, float aspect) {
  vec2 scaled = vec2(cell.x / aspect, cell.y);
  return smoothstep(radius + 0.02, radius - 0.02, length(scaled));
}
```

---

## 10. Unified Dither Dispatcher

```typescript
type DitherAlgorithm =
  | 'none' | 'threshold' | 'random'
  | 'bayer-2x2' | 'bayer-4x4' | 'bayer-8x8'
  | 'blue-noise'
  | 'floyd-steinberg' | 'jarvis-judice-ninke' | 'stucki' | 'burkes'
  | 'atkinson' | 'sierra' | 'sierra-two-row' | 'sierra-lite'
  | 'riemersma'

function applyDithering(
  pixels: Float32Array, w: number, h: number,
  algorithm: DitherAlgorithm, strength: number, levels: number = 2
): Float32Array {
  if (algorithm === 'none' || strength <= 0) return new Float32Array(pixels)

  switch (algorithm) {
    case 'threshold': return thresholdDither(pixels, w, h)
    case 'random': return randomDitherSeeded(pixels, w, h, strength, 0)
    case 'bayer-2x2': return bayerDither(pixels, w, h, BAYER_2X2, strength)
    case 'bayer-4x4': return bayerDither(pixels, w, h, BAYER_4X4, strength)
    case 'bayer-8x8': return bayerDither(pixels, w, h, BAYER_8X8, strength)
    case 'blue-noise': return blueNoiseDither(pixels, w, h, blueNoiseTexture, 64, strength)
    case 'riemersma': return riemersmaDither(pixels, w, h, strength)
    default: {
      const kernel = DIFFUSION_KERNELS[algorithm]
      if (kernel) return errorDiffusion(pixels, w, h, strength, kernel, levels)
      return new Float32Array(pixels)
    }
  }
}
```

---

## 11. Dithering Strength Guide

| Strength | Visual Effect | Use Case |
|----------|-------------|----------|
| `0.0` | No dithering — continuous tone maps directly to chars | Clean gradient rendering |
| `0.05–0.15` | Barely visible texture | Subtle background dither |
| `0.2–0.4` | Moderate stipple, preserves tonal accuracy | Photographic content |
| `0.5` | Standard dithering — visible pattern, good tonal range | General purpose default |
| `0.7–0.8` | Strong stipple, clear dither pattern | Retro/artistic effect |
| `1.0` | Full error propagation / maximum threshold offset | Maximum effect |
| `>1.0` | Overdrive — exaggerated noise/texture | Artistic distortion |
