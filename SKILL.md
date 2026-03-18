---
name: ascii-dither-shader
license: MIT
description: >
  Complete engine for ASCII art, dithering, halftone, and character-based shaders over images
  and videos. 14 algorithms (Floyd-Steinberg, Bayer, Atkinson, blue noise, Riemersma, Stucki,
  Sierra, JJN), 40+ charsets, 9 art styles, WebGL/WGSL/GLSL shaders, CRT/bloom/glitch effects,
  color science (OKLAB), video pipeline, React components. Use when converting images or video
  to ASCII, applying dithering or halftone effects, building retro/CRT/terminal visuals, writing
  GPU dithering shaders, rendering braille or block characters, or any character-grid visual.
  Triggers on: ASCII, dither, halftone, error diffusion, character art, text art, retro rendering,
  stippling, blue noise, Bayer, braille, CRT, scanline, image-to-text, video-to-ASCII, mosaic.
---

# ASCII / Dither / Shader Engine

Complete reference for building ASCII art renderers, dithering effects, and character-based
visual shaders. Covers mathematical foundations, 14 dithering algorithms, 9 art style renderers,
color science, WebGL acceleration, video pipelines, and post-processing effects.

Grounded in peer-reviewed research:
- Bayer (1973) — recursive threshold matrix construction
- Floyd & Steinberg (1976) — error diffusion with serpentine scanning
- Ulichney (1987) — blue noise dithering and spectral analysis
- Krahmer & Veselovska (2024) — mathematical foundations of halftoning (arXiv:2406.12760)
- Wolfe et al. (2021) — spatiotemporal blue noise for temporal coherence (arXiv:2112.09629)
- Jiang et al. (2023) — RL-optimized halftoning quality bounds (arXiv:2304.12152)
- Coumar & Kingston (2025) — ML-based ASCII art evaluation (arXiv:2503.14375)

## Architecture

Two rendering targets, same algorithmic core:

| Target | Stack | Output | Reference |
|--------|-------|--------|-----------|
| **Web (React/TSX)** | Canvas 2D / WebGL, single `.tsx` file, zero deps beyond React | Live interactive component | `component-patterns.md` |
| **Video (Python)** | NumPy + Pillow + ffmpeg, single `.py` file | MP4 / GIF / PNG sequence | `video-pipeline.md` |

Both share the same rendering pipeline:

```
INPUT → SAMPLE → LUMINANCE → DITHER → MAP CHARS → COLOR → POST-FX → OUTPUT
  │        │          │          │          │          │        │
  │     downsample  BT.709    algorithm  charset    palette  CRT/bloom
  │     to grid     sRGB→lin  selection  lookup     apply    scanlines
  │     resolution  + gamma                                  grain
```

## Quick Decision Tree

```
User wants...
├── Animated background (no input image) → Generative mode
│   └── Read: algorithms.md + shader-effects.md + component-patterns.md
├── Image → ASCII conversion → Source mode (static)
│   └── Read: algorithms.md + ascii-rendering.md + color-science.md
├── Video/webcam → ASCII → Source mode (real-time)
│   └── Read: algorithms.md + performance.md + video-pipeline.md
├── Halftone/dot pattern → Halftone renderer
│   └── Read: algorithms.md § Halftone + ascii-rendering.md § Halftone
├── Dithered image (not ASCII) → Pure dithering
│   └── Read: algorithms.md (full)
├── WebGL shader effect → GPU pipeline
│   └── Read: algorithms.md § WebGL + performance.md § WebGL
├── Retro/CRT/terminal look → Post-processing
│   └── Read: shader-effects.md + color-science.md § Retro Palettes
└── Braille/block char art → Specialized renderers
    └── Read: ascii-rendering.md § Braille + algorithms.md
```

## Props System (Web Components)

Every component exposes this configurable surface:

```typescript
interface AsciiShaderProps {
  // --- Source ---
  src?: string | HTMLImageElement | HTMLVideoElement | MediaStream
  mode?: 'generative' | 'source' | 'hybrid'

  // --- Grid ---
  fontSize?: number          // 8-32, default 14
  cellSpacing?: number       // 0.5-1.0, default 0.6 (char width / height ratio)
  maxCols?: number           // grid cap, default 400

  // --- Characters ---
  charset?: string           // dark-to-bright ramp, e.g. ' .:-=+*#%@'
  charsetPreset?: CharsetPreset // 'standard' | 'blocks' | 'detailed' | 'braille' | ...

  // --- Dithering ---
  ditherAlgorithm?: DitherAlgorithm  // 14 options, default 'none'
  ditherStrength?: number             // 0-2, default 0.5
  ditherLevels?: number               // 2-256, for multi-level quantization

  // --- Color ---
  colorMode?: ColorMode      // 'grayscale' | 'fullcolor' | 'matrix' | 'amber' | ...
  customColor?: string       // hex for custom mode
  palette?: string           // named retro palette
  colorSpace?: 'srgb' | 'oklab' | 'oklch'  // perceptual color mode

  // --- Style ---
  artStyle?: ArtStyle        // 'classic' | 'halftone' | 'braille' | 'line' | ...
  halftoneShape?: HalftoneShape  // 'circle' | 'square' | 'diamond' | ...
  halftoneSize?: number
  lineLength?: number
  lineThickness?: number

  // --- Post-FX ---
  crt?: boolean | CRTConfig
  scanlines?: boolean | number
  bloom?: boolean | BloomConfig
  chromaticAberration?: number
  vignette?: number
  grain?: number
  glitch?: boolean | GlitchConfig

  // --- Adjustments ---
  brightness?: number        // -0.5 to 0.5
  contrast?: number          // 0.5 to 2.0
  gamma?: number             // 0.3 to 3.0
  invert?: boolean
  edgeBoost?: number         // 0-1, Sobel edge enhancement

  // --- Animation ---
  speed?: number             // time multiplier
  fps?: number               // target framerate cap

  // --- Layout ---
  className?: string
  style?: React.CSSProperties
}
```

## Algorithm Selection Guide

Read `references/algorithms.md` for complete implementations. Summary:

### Dithering Algorithms (14 total)

| Algorithm | Type | Speed | Quality | Visual Character | Best For |
|-----------|------|-------|---------|-----------------|----------|
| **None** | — | — | — | Continuous tone mapped to chars | Clean gradient rendering |
| **Threshold** | Quantize | Fastest | Low | Hard binary cutoff | High-contrast graphics |
| **Random** | Noise | Fastest | Low | White noise stipple | Artistic noise texture |
| **Bayer 2×2** | Ordered | Fastest | Low | Coarse cross-hatch | Pixel art, 1-bit games |
| **Bayer 4×4** | Ordered | Fastest | Medium | Fine cross-hatch | Background textures |
| **Bayer 8×8** | Ordered | Fastest | Good | Patterned, retro | Animation (flicker-free) |
| **Blue Noise** | Stochastic | Fast | Excellent | Film grain feel | Photographic, cinematic |
| **Floyd-Steinberg** | Diffusion | Fast | Good | Classic, balanced | General purpose |
| **Atkinson** | Diffusion | Fast | Stylized | High-contrast, Mac | Retro, graphic design |
| **Sierra** | Diffusion | Medium | Very Good | Smooth gradients | Portraits, photography |
| **Sierra-Lite** | Diffusion | Very Fast | Decent | Slight noise | Real-time video |
| **Stucki** | Diffusion | Slow | Excellent | Sharp edges | Technical images |
| **Jarvis-Judice-Ninke** | Diffusion | Slow | Excellent | Smoothest gradients | Print-quality output |
| **Riemersma** | Space-fill | Medium | Good | No directional bias | Isotropic rendering |

### When to Use What

- **Real-time video ≥30fps**: Sierra-Lite or Bayer 8×8. Error diffusion with >6 neighbors will drop frames.
- **Static image, max quality**: JJN or Stucki → large kernels minimize banding.
- **Animation / procedural**: Bayer → spatially fixed thresholds prevent temporal shimmer.
- **Cinematic / film look**: Blue noise → perceptually optimal, no visible structure.
- **Retro computing**: Atkinson (Mac), Bayer 4×4 (Game Boy), Threshold (ZX Spectrum).
- **Print halftone**: Halftone renderer with CMYK angle offsets (see `algorithms.md` § Halftone).

## Art Style Renderers (9 total)

| Style | Description | Read |
|-------|-------------|------|
| **Classic** | Char-mapped luminance, the default | `ascii-rendering.md` |
| **Halftone** | Geometric dots/shapes sized by brightness | `ascii-rendering.md` § Halftone |
| **Braille** | 2×4 sub-cell dot patterns for ultra-fine detail | `ascii-rendering.md` § Braille |
| **DotCross** | Edge-aware dual ramps (dots for smooth, crosses for edges) | `ascii-rendering.md` |
| **Line** | Oriented line segments, flow-field hatchwork | `ascii-rendering.md` |
| **Particles** | Threshold-based scattered bright points | `ascii-rendering.md` |
| **Retro** | Duotone with grain, posterization, gamma | `ascii-rendering.md` |
| **Terminal** | Green phosphor, scanline interleave | `ascii-rendering.md` |
| **Custom** | User-defined render function | `component-patterns.md` |

## Post-Processing Effects

| Effect | What It Does | Read |
|--------|-------------|------|
| **CRT** | Barrel distortion + scanlines + phosphor RGB sub-pixels + bloom | `shader-effects.md` § CRT |
| **Scanlines** | Alternating row darkening with configurable gap/opacity | `shader-effects.md` § Scanlines |
| **Bloom/Glow** | Gaussian blur on bright regions, screen-blended | `shader-effects.md` § Bloom |
| **Chromatic Aberration** | RGB channel offset for lens fringe | `shader-effects.md` § Chromatic |
| **Vignette** | Radial edge darkening | `shader-effects.md` § Vignette |
| **Film Grain** | Temporal noise with per-frame seed | `shader-effects.md` § Grain |
| **Glitch** | Block displacement, color channel shift, scan tear | `shader-effects.md` § Glitch |
| **Interlace** | Even/odd field alternation per frame | `shader-effects.md` § Interlace |

## References

| File | What's Inside |
|------|---------------|
| `references/algorithms.md` | All 14 dithering algorithms with mathematical derivations, error diffusion framework, Bayer matrix recursive construction, blue noise generation, halftone CMYK patterns, multi-level quantization, WebGL GLSL implementations |
| `references/ascii-rendering.md` | Character mapping, BT.709 luminance, sRGB LUT, 30+ charset presets, braille sub-cell rendering, edge-directed char selection, halftone shape renderers, art style implementations |
| `references/color-science.md` | sRGB/linear conversion, OKLAB/OKLCH perceptual spaces, BT.709/BT.601 luminance, duotone palettes, color harmony generation, phosphor colors, HSV/HSL utilities |
| `references/shader-effects.md` | CRT barrel distortion, scanlines, bloom (Gaussian + kawase), chromatic aberration, vignette, film grain, glitch, feedback buffers, noise FX (simplex 3D + fBm), direction system |
| `references/performance.md` | Canvas 2D optimization, WebGL shader pipeline, OffscreenCanvas + Workers, Float32Array patterns, sRGB LUT caching, grid cap tuning, frame budget breakdown, memory management |
| `references/video-pipeline.md` | Frame extraction, temporal coherence (blue noise masks + motion estimation), ffmpeg encoding, Python/NumPy vectorized pipeline, parallel rendering, adaptive quality profiles |
| `references/component-patterns.md` | React component template, useCanvas hook, ResizeObserver + HiDPI, props interface, source mode (image/video/webcam), multi-layer compositing, mouse interaction, a11y |

## Critical Rules

1. **sRGB linearization is non-negotiable.** Never compute luminance from raw sRGB values — the gamma curve makes mid-tones appear 2× brighter than they are. Always use the sRGB-to-linear LUT (see `color-science.md`).

2. **Bayer dithering for animation, error diffusion for stills.** Error diffusion causes temporal shimmer because nearby pixel quantization decisions cascade differently each frame. Bayer thresholds are spatially fixed → stable animation.

3. **Blue noise is perceptually optimal.** Research (Ulichney 1987, Wolfe et al. 2021) proves that blue noise error distribution minimizes visible artifacts because human vision is most sensitive to low-frequency patterns. Use blue noise textures for photographic/cinematic quality.

4. **Character density determines visual quality.** A 68-char ramp (`detailed`) wastes tonal range at 40-column grids because adjacent brightness levels map to the same char. Match charset length to grid resolution (see `ascii-rendering.md` § Choosing a Palette).

5. **Aspect ratio correction is mandatory for ASCII.** Monospace characters are taller than wide (~1.67:1 ratio). Without correction, circles become ovals. Apply `cellWidth = fontSize * 0.6` (not 1.0) for the horizontal cell dimension.

6. **Adaptive tonemap, not linear brightness.** Especially for video: use percentile-based normalization with gamma (see `video-pipeline.md` § Tonemap). Linear `* N` multipliers clip highlights and crush shadows.

## Workflow

### Step 1: Determine What the User Wants
- Read the Quick Decision Tree above
- Identify: source mode (generative/source/hybrid), target (web/video), primary effect

### Step 2: Read the Minimum Set of References
- Always read `algorithms.md` for any dithering/quantization work
- Read additional references per the decision tree

### Step 3: Build the Component/Script
- **Web**: single `.tsx` file, all utilities inlined, zero external deps beyond React
- **Video**: single `.py` file, NumPy + Pillow + ffmpeg

### Step 4: Verify Quality
- sRGB linearization used for luminance
- Aspect ratio correction applied
- Dithering algorithm appropriate for use case (animation vs static)
- Post-FX applied after character rendering
- Performance within frame budget (see `performance.md`)
