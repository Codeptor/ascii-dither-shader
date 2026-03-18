# Color Science Reference

Perceptual color models, luminance calculation, palette systems, and color space conversions
for ASCII/dither rendering. Every color operation should be perceptually accurate — raw sRGB
math produces visually wrong results.

---

## 1. sRGB Linearization

sRGB is NOT linear — it has a gamma curve (~2.2) that compresses dark values. All luminance
and color math must operate on linear values, then convert back to sRGB for display.

### sRGB-to-Linear LUT (Pre-compute Once)

```typescript
const SRGB_LUT = new Float32Array(256)
for (let i = 0; i < 256; i++) {
  const s = i / 255
  SRGB_LUT[i] = s <= 0.04045 ? s / 12.92 : Math.pow((s + 0.055) / 1.055, 2.4)
}
```

### Linear-to-sRGB (Inverse)

```typescript
function linearToSrgb(c: number): number {
  return c <= 0.0031308 ? c * 12.92 : 1.055 * Math.pow(c, 1 / 2.4) - 0.055
}
```

### Python

```python
import numpy as np

SRGB_LUT = np.zeros(256, dtype=np.float32)
for i in range(256):
    s = i / 255.0
    SRGB_LUT[i] = s / 12.92 if s <= 0.04045 else ((s + 0.055) / 1.055) ** 2.4

def srgb_to_linear(img: np.ndarray) -> np.ndarray:
    """Convert uint8 sRGB image to float32 linear. Vectorized via LUT."""
    return SRGB_LUT[img]

def linear_to_srgb(img: np.ndarray) -> np.ndarray:
    """Convert float32 linear to uint8 sRGB."""
    out = np.where(img <= 0.0031308, img * 12.92, 1.055 * np.power(np.clip(img, 0, None), 1/2.4) - 0.055)
    return np.clip(out * 255, 0, 255).astype(np.uint8)
```

### GLSL

```glsl
vec3 srgbToLinear(vec3 srgb) {
  vec3 low = srgb / 12.92;
  vec3 high = pow((srgb + 0.055) / 1.055, vec3(2.4));
  return mix(low, high, step(vec3(0.04045), srgb));
}

vec3 linearToSrgb(vec3 lin) {
  vec3 low = lin * 12.92;
  vec3 high = 1.055 * pow(lin, vec3(1.0/2.4)) - 0.055;
  return mix(low, high, step(vec3(0.0031308), lin));
}
```

---

## 2. Luminance Models

### BT.709 (HD, Web Standard)

```typescript
function luminanceBT709(r: number, g: number, b: number): number {
  return 0.2126 * SRGB_LUT[r] + 0.7152 * SRGB_LUT[g] + 0.0722 * SRGB_LUT[b]
}
```

Weights: R=0.2126, G=0.7152, B=0.0722. This is the standard for all web/HD content.
Green dominates because human vision is most sensitive to green wavelengths.

### BT.601 (SD, Legacy)

```typescript
function luminanceBT601(r: number, g: number, b: number): number {
  return 0.299 * SRGB_LUT[r] + 0.587 * SRGB_LUT[g] + 0.114 * SRGB_LUT[b]
}
```

Use only for SD video content. BT.709 is preferred for all new work.

### Perceived Brightness (Gamma-Aware Display)

For display-referred brightness (how bright a pixel actually appears on screen):

```typescript
function perceivedBrightness(linearLum: number): number {
  // Apply display gamma to get perceptual brightness
  return Math.pow(linearLum, 1 / 2.2)
}
```

### Brightness Adjustment

```typescript
function adjustBrightness(value: number, brightness: number, contrast: number): number {
  let v = value + brightness           // additive offset (-0.5 to 0.5)
  v = (v - 0.5) * contrast + 0.5       // contrast around midpoint (0.5 to 2.0)
  return Math.max(0, Math.min(1, v))
}
```

### GLSL

```glsl
float luminance(vec3 color) {
  vec3 lin = srgbToLinear(color);
  return dot(lin, vec3(0.2126, 0.7152, 0.0722));
}
```

---

## 3. OKLAB / OKLCH (Perceptually Uniform)

OKLAB is the best color space for perceptual operations — interpolation, distance, and
palette generation. Unlike HSL, distances in OKLAB correspond to perceived color differences.

### sRGB → OKLAB

```typescript
function srgbToOklab(r: number, g: number, b: number): [number, number, number] {
  // sRGB to linear
  const lr = SRGB_LUT[r], lg = SRGB_LUT[g], lb = SRGB_LUT[b]

  // Linear RGB to LMS (cone response)
  const l = 0.4122214708 * lr + 0.5363325363 * lg + 0.0514459929 * lb
  const m = 0.2119034982 * lr + 0.6806995451 * lg + 0.1073969566 * lb
  const s = 0.0883024619 * lr + 0.2817188376 * lg + 0.6299787005 * lb

  // Cube root (perceptual compression)
  const l_ = Math.cbrt(l), m_ = Math.cbrt(m), s_ = Math.cbrt(s)

  // LMS to OKLAB
  const L = 0.2104542553 * l_ + 0.7936177850 * m_ - 0.0040720468 * s_
  const A = 1.9779984951 * l_ - 2.4285922050 * m_ + 0.4505937099 * s_
  const B = 0.0259040371 * l_ + 0.7827717662 * m_ - 0.8086757660 * s_

  return [L, A, B]  // L: 0-1 (lightness), a/b: ~-0.4 to 0.4 (color axes)
}
```

### OKLAB → sRGB

```typescript
function oklabToSrgb(L: number, A: number, B: number): [number, number, number] {
  const l_ = L + 0.3963377774 * A + 0.2158037573 * B
  const m_ = L - 0.1055613458 * A - 0.0638541728 * B
  const s_ = L - 0.0894841775 * A - 1.2914855480 * B

  const l = l_ * l_ * l_, m = m_ * m_ * m_, s = s_ * s_ * s_

  const lr = +4.0767416621 * l - 3.3077115913 * m + 0.2309699292 * s
  const lg = -1.2684380046 * l + 2.6097574011 * m - 0.3413193965 * s
  const lb = -0.0041960863 * l - 0.7034186147 * m + 1.7076147010 * s

  return [
    Math.round(Math.max(0, Math.min(1, linearToSrgb(lr))) * 255),
    Math.round(Math.max(0, Math.min(1, linearToSrgb(lg))) * 255),
    Math.round(Math.max(0, Math.min(1, linearToSrgb(lb))) * 255),
  ]
}
```

### OKLCH (Polar Form)

OKLCH is OKLAB in polar coordinates — easier for palette generation and hue operations.

```typescript
function oklabToOklch(L: number, A: number, B: number): [number, number, number] {
  const C = Math.sqrt(A * A + B * B)        // chroma (saturation)
  const H = Math.atan2(B, A) * 180 / Math.PI  // hue in degrees
  return [L, C, H < 0 ? H + 360 : H]
}

function oklchToOklab(L: number, C: number, H: number): [number, number, number] {
  const rad = H * Math.PI / 180
  return [L, C * Math.cos(rad), C * Math.sin(rad)]
}
```

### Perceptual Color Interpolation

```typescript
function interpolateOklab(
  c1: [number, number, number],  // OKLAB
  c2: [number, number, number],
  t: number
): [number, number, number] {
  return [
    c1[0] + (c2[0] - c1[0]) * t,
    c1[1] + (c2[1] - c1[1]) * t,
    c1[2] + (c2[2] - c1[2]) * t,
  ]
}
```

---

## 4. Color Modes (Character Tinting)

### Mode Implementations

```typescript
type ColorMode = 'grayscale' | 'fullcolor' | 'matrix' | 'amber' | 'sepia'
               | 'cool-blue' | 'neon' | 'custom' | 'phosphor-green'
               | 'phosphor-amber' | 'ice' | 'blood' | 'void' | 'sunset'

function getColor(mode: ColorMode, brightness: number,
  r: number, g: number, b: number, customHex?: string): string {

  const gray = Math.round(brightness * 255)

  switch (mode) {
    case 'grayscale':
      return `rgb(${gray},${gray},${gray})`

    case 'fullcolor':
      return `rgb(${r},${g},${b})`

    case 'matrix': {
      const f = Math.min(255, gray + 25)
      return `rgb(${f * 0.2 | 0},${f},${f * 0.3 | 0})`
    }

    case 'amber': {
      const f = Math.min(255, gray + 20)
      return `rgb(${f},${f * 0.7 | 0},${f * 0.15 | 0})`
    }

    case 'sepia': {
      const f = Math.min(255, gray + 15)
      return `rgb(${Math.min(255, f * 1.1) | 0},${f * 0.85 | 0},${f * 0.65 | 0})`
    }

    case 'cool-blue': {
      const f = Math.min(255, gray + 15)
      return `rgb(${f * 0.55 | 0},${f * 0.75 | 0},${f})`
    }

    case 'neon': {
      const hue = (brightness * 300) | 0
      const lightness = (20 + brightness * 50) | 0
      return `hsl(${hue},100%,${lightness}%)`
    }

    case 'phosphor-green': {
      const p = Math.min(255, gray + 32)
      return `rgb(${Math.min(72, p * 0.12) | 0},${p},${Math.min(88, p * 0.2) | 0})`
    }

    case 'phosphor-amber': {
      const p = Math.min(255, gray + 24)
      return `rgb(${Math.min(255, p * 1.1) | 0},${p * 0.65 | 0},${p * 0.08 | 0})`
    }

    case 'ice': {
      const f = Math.min(255, gray + 10)
      return `rgb(${f * 0.85 | 0},${f * 0.95 | 0},${f})`
    }

    case 'blood': {
      const f = Math.min(255, gray + 10)
      return `rgb(${f},${f * 0.15 | 0},${f * 0.1 | 0})`
    }

    case 'void': {
      const f = Math.min(255, gray + 5)
      return `rgb(${f * 0.6 | 0},${f * 0.3 | 0},${f})`
    }

    case 'sunset': {
      // Warm-to-cool gradient based on brightness
      const hue = 20 + brightness * 40  // orange to red-pink
      const sat = 80 + brightness * 20
      const light = 15 + brightness * 55
      return `hsl(${hue | 0},${sat | 0}%,${light | 0}%)`
    }

    case 'custom': {
      if (!customHex) return `rgb(${gray},${gray},${gray})`
      const hex = customHex.replace('#', '')
      const cr = parseInt(hex.slice(0, 2), 16)
      const cg = parseInt(hex.slice(2, 4), 16)
      const cb = parseInt(hex.slice(4, 6), 16)
      return `rgb(${cr * brightness | 0},${cg * brightness | 0},${cb * brightness | 0})`
    }

    default: return `rgb(${gray},${gray},${gray})`
  }
}
```

---

## 5. Retro Duotone Palettes

Two-tone palettes that interpolate between dark and bright endpoints. Combined with vignette,
grain, and shimmer for authentic retro CRT/film aesthetics.

```typescript
interface DuotonePalette {
  low: [number, number, number]   // RGB for darkest
  high: [number, number, number]  // RGB for brightest
}

const RETRO_PALETTES: Record<string, DuotonePalette> = {
  'amber-classic':   { low: [20, 12, 6],   high: [255, 223, 178] },
  'cyan-night':      { low: [6, 16, 22],    high: [166, 240, 255] },
  'violet-haze':     { low: [17, 10, 26],   high: [242, 198, 255] },
  'lime-pulse':      { low: [10, 18, 8],    high: [226, 255, 162] },
  'mono-ice':        { low: [12, 12, 12],   high: [245, 248, 255] },
  'rose-gold':       { low: [18, 8, 10],    high: [255, 200, 180] },
  'electric-blue':   { low: [4, 8, 20],     high: [100, 180, 255] },
  'toxic-green':     { low: [5, 12, 5],     high: [130, 255, 80]  },
  'warm-sepia':      { low: [22, 15, 8],    high: [240, 210, 170] },
  'cold-steel':      { low: [10, 12, 14],   high: [190, 200, 210] },
  'synthwave-pink':  { low: [15, 5, 20],    high: [255, 100, 200] },
  'forest-twilight': { low: [8, 14, 10],    high: [140, 220, 160] },
}

function duotoneColor(
  brightness: number, palette: DuotonePalette,
  x: number, y: number, cols: number, rows: number, time: number,
  vignetteStrength: number = 0.3, grainAmount: number = 0.05
): string {
  let b = Math.max(0, Math.min(1, brightness))

  // Vignette
  const nx = (x / cols) * 2 - 1, ny = (y / rows) * 2 - 1
  const dist = Math.sqrt(nx * nx + ny * ny) / Math.SQRT2
  b *= Math.max(0, 1 - dist * dist * vignetteStrength)

  // Film grain
  const seed = (x * 2654435761 + y * 340573321 + (time * 60 | 0)) & 0xffffff
  b = Math.max(0, Math.min(1, b + ((seed / 0xffffff) - 0.5) * 2 * grainAmount))

  // Shimmer (subtle scanline brightness)
  b = Math.max(0, Math.min(1, b * (1 + 0.02 * Math.sin(y * 0.5 + time * 3))))

  // Interpolate duotone
  const [lr, lg, lb] = palette.low
  const [hr, hg, hb] = palette.high
  return `rgb(${lr + (hr - lr) * b | 0},${lg + (hg - lg) * b | 0},${lb + (hb - lb) * b | 0})`
}
```

---

## 6. Color Harmony Generation

Generate palettes using color theory relationships in OKLCH space.

```typescript
type Harmony = 'complementary' | 'triadic' | 'analogous' | 'split-complementary' | 'tetradic'

function generateHarmony(
  baseHue: number, harmony: Harmony, lightness: number = 0.7, chroma: number = 0.15
): Array<[number, number, number]> {
  const hues: number[] = [baseHue]

  switch (harmony) {
    case 'complementary':
      hues.push((baseHue + 180) % 360)
      break
    case 'triadic':
      hues.push((baseHue + 120) % 360, (baseHue + 240) % 360)
      break
    case 'analogous':
      hues.push((baseHue + 30) % 360, (baseHue - 30 + 360) % 360)
      break
    case 'split-complementary':
      hues.push((baseHue + 150) % 360, (baseHue + 210) % 360)
      break
    case 'tetradic':
      hues.push((baseHue + 90) % 360, (baseHue + 180) % 360, (baseHue + 270) % 360)
      break
  }

  return hues.map(h => {
    const [L, A, B] = oklchToOklab(lightness, chroma, h)
    return oklabToSrgb(L, A, B)
  })
}
```

---

## 7. HSV / HSL Utilities

### HSV to RGB

```typescript
function hsv2rgb(h: number, s: number, v: number): [number, number, number] {
  const c = v * s
  const hp = h / 60
  const x = c * (1 - Math.abs((hp % 2) - 1))
  const m = v - c
  let r1: number, g1: number, b1: number

  if      (hp < 1) { r1 = c; g1 = x; b1 = 0 }
  else if (hp < 2) { r1 = x; g1 = c; b1 = 0 }
  else if (hp < 3) { r1 = 0; g1 = c; b1 = x }
  else if (hp < 4) { r1 = 0; g1 = x; b1 = c }
  else if (hp < 5) { r1 = x; g1 = 0; b1 = c }
  else             { r1 = c; g1 = 0; b1 = x }

  return [(r1 + m) * 255 | 0, (g1 + m) * 255 | 0, (b1 + m) * 255 | 0]
}
```

### RGB to HSV

```typescript
function rgb2hsv(r: number, g: number, b: number): [number, number, number] {
  r /= 255; g /= 255; b /= 255
  const max = Math.max(r, g, b), min = Math.min(r, g, b)
  const d = max - min
  let h = 0
  const s = max === 0 ? 0 : d / max
  const v = max

  if (d !== 0) {
    if (max === r) h = ((g - b) / d + (g < b ? 6 : 0)) * 60
    else if (max === g) h = ((b - r) / d + 2) * 60
    else h = ((r - g) / d + 4) * 60
  }
  return [h, s, v]
}
```

---

## 8. CRT Phosphor Colors

Authentic CRT phosphor emission spectra for terminal emulation.

```typescript
const PHOSPHOR_COLORS = {
  'P1':  { peak: [0.12, 1.0, 0.20], decay: 0.92 },  // Green (classic VT100)
  'P3':  { peak: [1.0, 0.70, 0.15], decay: 0.88 },  // Amber (IBM 5151)
  'P4':  { peak: [1.0, 1.0, 1.0],   decay: 0.95 },  // White (TV)
  'P7':  { peak: [0.1, 0.8, 0.95],  decay: 0.90 },  // Blue-white (radar)
  'P22': { peak: [1.0, 1.0, 1.0],   decay: 0.93 },  // Tri-color (color TV)
  'P31': { peak: [0.15, 1.0, 0.30], decay: 0.85 },  // Green (fast, oscilloscope)
  'P39': { peak: [0.08, 1.0, 0.15], decay: 0.98 },  // Green (slow, long persist)
}

function phosphorColor(brightness: number, phosphor: keyof typeof PHOSPHOR_COLORS): string {
  const { peak } = PHOSPHOR_COLORS[phosphor]
  const b = Math.min(1, brightness + 0.04)  // minimum glow
  const r = Math.min(255, b * peak[0] * 255 | 0)
  const g = Math.min(255, b * peak[1] * 255 | 0)
  const bl = Math.min(255, b * peak[2] * 255 | 0)
  return `rgb(${r},${g},${bl})`
}
```

---

## 9. Color Quantization

Reduce a full-color image to N colors for palette-limited effects (Game Boy, NES, CGA, etc.).

### Median Cut (Fast, Good Quality)

```typescript
function medianCut(
  pixels: Uint8Array,  // RGBA flat array
  numColors: number
): Array<[number, number, number]> {
  // Collect all unique-ish colors (sample every 4th pixel for speed)
  const colors: Array<[number, number, number]> = []
  for (let i = 0; i < pixels.length; i += 16) {
    colors.push([pixels[i], pixels[i + 1], pixels[i + 2]])
  }

  let buckets: Array<Array<[number, number, number]>> = [colors]

  while (buckets.length < numColors) {
    // Find bucket with largest range
    let maxRange = -1, splitIdx = 0, splitChannel = 0
    for (let i = 0; i < buckets.length; i++) {
      for (let ch = 0; ch < 3; ch++) {
        const vals = buckets[i].map(c => c[ch])
        const range = Math.max(...vals) - Math.min(...vals)
        if (range > maxRange) { maxRange = range; splitIdx = i; splitChannel = ch }
      }
    }

    // Sort by widest channel and split at median
    const bucket = buckets.splice(splitIdx, 1)[0]
    bucket.sort((a, b) => a[splitChannel] - b[splitChannel])
    const mid = bucket.length >> 1
    buckets.push(bucket.slice(0, mid), bucket.slice(mid))
  }

  // Average each bucket to get palette color
  return buckets.map(bucket => {
    const avg: [number, number, number] = [0, 0, 0]
    for (const c of bucket) { avg[0] += c[0]; avg[1] += c[1]; avg[2] += c[2] }
    const n = bucket.length || 1
    return [avg[0] / n | 0, avg[1] / n | 0, avg[2] / n | 0]
  })
}
```

### Classic Gaming Palettes

```typescript
const CLASSIC_PALETTES: Record<string, Array<[number, number, number]>> = {
  'gameboy': [[15, 56, 15], [48, 98, 48], [139, 172, 15], [155, 188, 15]],
  'nes': [
    [0,0,0],[252,252,252],[188,188,188],[124,124,124],
    [168,0,32],[228,0,88],[248,56,0],[228,92,16],
    [172,124,0],[0,184,0],[0,168,0],[0,168,68],
    [0,136,136],[0,120,248],[104,68,252],[216,0,204],
  ],
  'cga': [[0,0,0],[85,255,255],[255,85,255],[255,255,255]],
  'amber-mono': [[0,0,0],[255,176,0]],
  'green-mono': [[0,0,0],[0,255,65]],
}

function nearestPaletteColor(
  r: number, g: number, b: number,
  palette: Array<[number, number, number]>
): [number, number, number] {
  let minDist = Infinity, best = palette[0]
  for (const [pr, pg, pb] of palette) {
    // Weighted Euclidean (perceptual approximation)
    const dr = r - pr, dg = g - pg, db = b - pb
    const dist = 2 * dr * dr + 4 * dg * dg + 3 * db * db
    if (dist < minDist) { minDist = dist; best = [pr, pg, pb] }
  }
  return best
}
```
