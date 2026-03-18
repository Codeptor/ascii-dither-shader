# ascii-dither-shader

A comprehensive AI agent skill for creating ASCII art renderers, dithering effects, halftone patterns, and character-based shader components over images and videos. Works with any AI agent or coding assistant — not limited to any specific platform.

**8,554 lines | 284KB** of self-contained, production-ready reference material — every algorithm, mathematical derivation, and implementation an AI agent needs to produce stunning ASCII/dither visuals.

## What's Inside

```
ascii-dither-shader/
├── SKILL.md                          # Orchestration, decision tree, props API
└── references/
    ├── algorithms.md                 # 14 dithering algorithms with math
    ├── ascii-rendering.md            # 40+ charsets, 9 art styles, braille sub-cell
    ├── color-science.md              # sRGB, OKLAB, luminance, palettes, phosphors
    ├── component-patterns.md         # React hooks, templates, compositing, a11y
    ├── performance.md                # Canvas 2D, WebGL2, WebGPU, Workers
    ├── shader-effects.md             # CRT, bloom, glitch, grain, noise
    └── video-pipeline.md             # Frame extraction, temporal coherence, ffmpeg
```

## Algorithms

14 dithering algorithms with implementations in **TypeScript**, **Python/NumPy**, **GLSL**, and **WGSL**:

| Algorithm | Type | Visual Character |
|-----------|------|-----------------|
| Threshold | Quantize | Hard binary cutoff |
| Random | Noise | White noise stipple |
| Bayer 2x2 / 4x4 / 8x8 | Ordered | Cross-hatch patterns |
| Blue Noise | Stochastic | Film grain, perceptually optimal |
| Floyd-Steinberg | Diffusion | Classic balanced dithering |
| Atkinson | Diffusion | High-contrast Mac aesthetic |
| Sierra / Sierra-Lite | Diffusion | Smooth gradients |
| Stucki | Diffusion | Sharp edges |
| Jarvis-Judice-Ninke | Diffusion | Smoothest gradients |
| Burkes | Diffusion | Fast wide-kernel |
| Riemersma | Space-filling | Isotropic, no directional bias |

## Rendering

- **40+ character set presets** — standard, blocks, braille (3 variants), katakana, rune, greek, cyrillic, math, circuit, arrows, dots, stars, and more
- **Braille sub-cell rendering** — each braille character (U+2800-U+28FF) as a 2x4 dot grid for ultra-high detail
- **Edge-directed character selection** — Sobel gradient → directional chars (`/`, `\`, `|`, `-`)
- **9 art style renderers** — Classic, Halftone (5 shapes), Braille, DotCross, Line, Particles, Retro, Terminal, Custom

## Color Science

- sRGB linearization with pre-computed LUT
- BT.709 / BT.601 perceptual luminance
- OKLAB / OKLCH perceptual color spaces
- 14 color modes (grayscale, matrix, amber, phosphor-green, neon, ice, blood, void, sunset...)
- 12 retro duotone palettes with grain + shimmer
- CRT phosphor emission spectra (P1, P3, P4, P7, P22, P31, P39)
- Color harmony generation (complementary, triadic, analogous, split-complementary, tetradic)
- Median-cut color quantization + classic gaming palettes (Game Boy, NES, CGA)

## Shader Effects

All effects implemented in **Canvas 2D**, **GLSL**, and **WGSL**:

- CRT barrel distortion + phosphor RGB sub-pixels + scanlines + bloom
- Kawase bloom / Gaussian glow
- Chromatic aberration
- Vignette
- Film grain with temporal seeding
- Glitch (block displacement + channel split + scan tear)
- Interlace field alternation
- Feedback buffer (motion trails)
- Simplex 3D noise + fractional Brownian motion
- Sobel edge detection

## Performance

- Canvas 2D optimization patterns (Float32Array, LUT caching, bitmap font atlas)
- Full WebGL2 post-processing pipeline (compile, link, fullscreen quad, texture upload)
- WebGPU compute shader pipeline (Bayer dither in WGSL)
- OffscreenCanvas + Web Worker architecture
- Adaptive quality controller with hysteresis
- Shader language comparison table (GLSL vs WGSL vs HLSL)

## Video Pipeline

- Frame extraction (web: `requestVideoFrameCallback`, Python: ffmpeg subprocess)
- Temporal coherence via spatiotemporal blue noise (golden ratio offset)
- Adaptive tonemap (percentile normalization + gamma)
- NumPy vectorized rendering with character bitmap cache
- ffmpeg H.264/GIF encoding with deadlock prevention
- Hardware detection + adaptive quality profiles (draft/preview/production)

## Research Basis

Grounded in peer-reviewed papers:

- Bayer (1973) — Recursive threshold matrix construction
- Floyd & Steinberg (1976) — Error diffusion
- Ulichney (1987) — Blue noise dithering
- Krahmer & Veselovska (2024) — [Mathematical foundations of halftoning](https://arxiv.org/abs/2406.12760)
- Wolfe et al. (2021) — [Spatiotemporal blue noise masks](https://arxiv.org/abs/2112.09629)
- Jiang et al. (2023) — [RL-optimized halftoning](https://arxiv.org/abs/2304.12152)
- Coumar & Kingston (2025) — [ML-based ASCII art evaluation](https://arxiv.org/abs/2503.14375)
- McEwan et al. (2012) — [Efficient GLSL noise](https://arxiv.org/abs/1204.1461)

## Usage

This is a self-contained knowledge base that any AI agent or coding assistant can use as context for generating ASCII/dither visuals. It works with Claude Code, Cursor, Windsurf, Copilot, or any agent that supports skill/reference files.

### Claude Code
```bash
git clone https://github.com/codeptor/ascii-dither-shader.git ~/.claude/skills/ascii-dither-shader
```
Auto-triggers when you ask for ASCII art, dithering, halftone, or character-based rendering.

### Any AI Agent
Feed the relevant reference files as context when prompting for ASCII/dither work. Start with `SKILL.md` for the decision tree, then load the specific reference file you need.

### Direct Reference
The markdown files are human-readable too — use them as a technical reference for implementing dithering algorithms, shader effects, or ASCII rendering in any language.

## License

MIT
