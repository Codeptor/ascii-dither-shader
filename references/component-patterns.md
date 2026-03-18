# React Component Architecture Patterns

Production-ready patterns for building ASCII/dither rendering components in Next.js. Every
component is a single `.tsx` file with zero imports beyond React. All utilities are inlined.

---

## 1. useCanvas Hook

The core hook every ASCII/dither component needs. Handles canvas setup, HiDPI scaling,
responsive resizing, animation loop with time tracking, and accessibility.

```tsx
'use client'

import { useRef, useEffect, useCallback } from 'react'

interface CanvasState {
  ctx: CanvasRenderingContext2D
  width: number
  height: number
  dpr: number
  time: number
  deltaTime: number
  frame: number
}

type RenderFn = (state: CanvasState) => void

interface UseCanvasOptions {
  /** Target frames per second. 0 = uncapped. Default: 0 */
  fps?: number
  /** Skip animation entirely for prefers-reduced-motion users. Default: true */
  respectReducedMotion?: boolean
}

function useCanvas(
  renderFn: RenderFn,
  deps: React.DependencyList = [],
  options: UseCanvasOptions = {}
) {
  const canvasRef = useRef<HTMLCanvasElement>(null)
  const rafRef = useRef<number>(0)
  const renderRef = useRef<RenderFn>(renderFn)
  const prevTimeRef = useRef<number>(0)
  const frameRef = useRef<number>(0)

  // Keep render function ref current without restarting the loop
  useEffect(() => {
    renderRef.current = renderFn
  }, [renderFn])

  useEffect(() => {
    const canvas = canvasRef.current
    if (!canvas) return

    const ctx = canvas.getContext('2d')
    if (!ctx) return

    const { fps = 0, respectReducedMotion = true } = options

    // Accessibility: check prefers-reduced-motion
    const prefersReduced =
      respectReducedMotion &&
      typeof window !== 'undefined' &&
      window.matchMedia('(prefers-reduced-motion: reduce)').matches

    const dpr = typeof window !== 'undefined' ? window.devicePixelRatio || 1 : 1
    const minFrameInterval = fps > 0 ? 1000 / fps : 0

    // Size the canvas to fill its container at native resolution
    function resize() {
      const rect = canvas!.getBoundingClientRect()
      const w = rect.width
      const h = rect.height
      canvas!.width = Math.round(w * dpr)
      canvas!.height = Math.round(h * dpr)
      ctx!.setTransform(dpr, 0, 0, dpr, 0, 0)
    }

    resize()

    // ResizeObserver for responsive sizing
    const ro = new ResizeObserver(() => resize())
    ro.observe(canvas)

    // If reduced motion, render a single frame and stop
    if (prefersReduced) {
      renderRef.current({
        ctx,
        width: canvas.getBoundingClientRect().width,
        height: canvas.getBoundingClientRect().height,
        dpr,
        time: 0,
        deltaTime: 0,
        frame: 0,
      })
      return () => ro.disconnect()
    }

    // Animation loop
    function loop(timestamp: number) {
      const dt = timestamp - prevTimeRef.current

      // Frame rate limiting
      if (minFrameInterval > 0 && dt < minFrameInterval) {
        rafRef.current = requestAnimationFrame(loop)
        return
      }

      prevTimeRef.current = timestamp
      frameRef.current++

      const rect = canvas!.getBoundingClientRect()
      renderRef.current({
        ctx: ctx!,
        width: rect.width,
        height: rect.height,
        dpr,
        time: timestamp / 1000,
        deltaTime: dt / 1000,
        frame: frameRef.current,
      })

      rafRef.current = requestAnimationFrame(loop)
    }

    rafRef.current = requestAnimationFrame(loop)

    return () => {
      cancelAnimationFrame(rafRef.current)
      ro.disconnect()
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, deps)

  return canvasRef
}
```

### What this handles

| Concern | Mechanism |
|---------|-----------|
| **HiDPI** | `devicePixelRatio` scales the backing store; `setTransform` normalizes draw coordinates to CSS pixels |
| **Responsive** | `ResizeObserver` fires on any container size change (flex, grid, viewport resize) |
| **Frame rate** | `minFrameInterval` skips frames to hit target FPS without `setInterval` jitter |
| **Time tracking** | `time` (seconds since start), `deltaTime` (seconds since last frame), `frame` (count) |
| **Accessibility** | `prefers-reduced-motion: reduce` renders one static frame, no animation |
| **Cleanup** | Cancels `requestAnimationFrame` and disconnects `ResizeObserver` on unmount |
| **Render ref stability** | `renderRef` avoids restarting the loop when the render function captures new closure values |

---

## 2. Component Template

Full boilerplate for a self-contained ASCII effect component. Copy this, implement the
`render` function, and the component is ready to drop into any Next.js page.

```tsx
'use client'

import { useRef, useEffect, useCallback } from 'react'

// ---------------------------------------------------------------------------
// Types
// ---------------------------------------------------------------------------

interface AsciiEffectProps {
  /** Character ramp from dark to light. Default: ' .:-=+*#%@' */
  charset?: string
  /** Font size in CSS pixels. Controls grid density. Default: 14 */
  fontSize?: number
  /** Character cell width as fraction of height (monospace ratio). Default: 0.6 */
  cellSpacing?: number
  /** Foreground color. Default: '#00ff41' */
  color?: string
  /** Background color. Default: '#0a0a0a' */
  bgColor?: string
  /** Animation speed multiplier. Default: 1.0 */
  speed?: number
  /** Target FPS cap. 0 = uncapped. Default: 0 */
  fps?: number
  /** Additional CSS class for the wrapper. */
  className?: string
  /** Inline styles for the wrapper. */
  style?: React.CSSProperties
}

// ---------------------------------------------------------------------------
// Inlined utilities
// ---------------------------------------------------------------------------

// sRGB-to-linear LUT for perceptually correct luminance
const SRGB_LUT = new Float32Array(256)
for (let i = 0; i < 256; i++) {
  const s = i / 255
  SRGB_LUT[i] = s <= 0.04045 ? s / 12.92 : Math.pow((s + 0.055) / 1.055, 2.4)
}

function luminance(r: number, g: number, b: number): number {
  return 0.2126 * SRGB_LUT[r] + 0.7152 * SRGB_LUT[g] + 0.0722 * SRGB_LUT[b]
}

// ---------------------------------------------------------------------------
// Hook (inlined for zero-dep constraint)
// ---------------------------------------------------------------------------

interface CanvasState {
  ctx: CanvasRenderingContext2D
  width: number
  height: number
  dpr: number
  time: number
  deltaTime: number
  frame: number
}

function useCanvas(
  renderFn: (state: CanvasState) => void,
  deps: React.DependencyList = [],
  fps = 0
) {
  const canvasRef = useRef<HTMLCanvasElement>(null)
  const rafRef = useRef<number>(0)
  const renderRef = useRef(renderFn)
  const prevTimeRef = useRef<number>(0)
  const frameRef = useRef<number>(0)

  useEffect(() => { renderRef.current = renderFn }, [renderFn])

  useEffect(() => {
    const canvas = canvasRef.current
    if (!canvas) return
    const ctx = canvas.getContext('2d')
    if (!ctx) return

    const dpr = window.devicePixelRatio || 1
    const minInterval = fps > 0 ? 1000 / fps : 0

    const prefersReduced =
      window.matchMedia('(prefers-reduced-motion: reduce)').matches

    function resize() {
      const rect = canvas!.getBoundingClientRect()
      canvas!.width = Math.round(rect.width * dpr)
      canvas!.height = Math.round(rect.height * dpr)
      ctx!.setTransform(dpr, 0, 0, dpr, 0, 0)
    }
    resize()

    const ro = new ResizeObserver(() => resize())
    ro.observe(canvas)

    if (prefersReduced) {
      const rect = canvas!.getBoundingClientRect()
      renderRef.current({ ctx, width: rect.width, height: rect.height, dpr, time: 0, deltaTime: 0, frame: 0 })
      return () => ro.disconnect()
    }

    function loop(ts: number) {
      const dt = ts - prevTimeRef.current
      if (minInterval > 0 && dt < minInterval) {
        rafRef.current = requestAnimationFrame(loop)
        return
      }
      prevTimeRef.current = ts
      frameRef.current++
      const rect = canvas!.getBoundingClientRect()
      renderRef.current({ ctx: ctx!, width: rect.width, height: rect.height, dpr, time: ts / 1000, deltaTime: dt / 1000, frame: frameRef.current })
      rafRef.current = requestAnimationFrame(loop)
    }

    rafRef.current = requestAnimationFrame(loop)
    return () => { cancelAnimationFrame(rafRef.current); ro.disconnect() }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, deps)

  return canvasRef
}

// ---------------------------------------------------------------------------
// Component
// ---------------------------------------------------------------------------

export function AsciiEffect({
  charset = ' .:-=+*#%@',
  fontSize = 14,
  cellSpacing = 0.6,
  color = '#00ff41',
  bgColor = '#0a0a0a',
  speed = 1,
  fps = 0,
  className,
  style,
}: AsciiEffectProps) {
  const cellW = fontSize * cellSpacing
  const cellH = fontSize

  const render = useCallback(
    ({ ctx, width, height, time }: CanvasState) => {
      const cols = Math.ceil(width / cellW)
      const rows = Math.ceil(height / cellH)
      const t = time * speed

      // Clear
      ctx.fillStyle = bgColor
      ctx.fillRect(0, 0, width, height)

      // Render ASCII grid
      ctx.font = `${fontSize}px monospace`
      ctx.textBaseline = 'top'
      ctx.fillStyle = color

      for (let row = 0; row < rows; row++) {
        for (let col = 0; col < cols; col++) {
          // Replace this with your brightness/effect computation
          const nx = col / cols
          const ny = row / rows
          const brightness = (Math.sin(nx * 6 + t) * Math.cos(ny * 4 + t * 0.7) + 1) * 0.5

          const charIdx = Math.floor(brightness * (charset.length - 1))
          const char = charset[charIdx]

          ctx.fillText(char, col * cellW, row * cellH)
        }
      }
    },
    [charset, fontSize, cellW, cellH, color, bgColor, speed]
  )

  const canvasRef = useCanvas(render, [render], fps)

  return (
    <canvas
      ref={canvasRef}
      aria-hidden="true"
      className={className}
      style={{
        width: '100%',
        height: '100%',
        display: 'block',
        ...style,
      }}
    />
  )
}

// Usage:
// <AsciiEffect charset=" .oO@" color="#e0e0e0" fontSize={12} speed={0.5} />
```

### Key decisions

- **`aria-hidden="true"`** on the canvas because it is purely decorative. Screen readers
  skip it. If the ASCII conveys meaningful content, use a visually hidden text alternative instead.
- **`display: block`** eliminates the 4px inline gap that canvas gets by default.
- **Width/height 100%** means the component fills its container. Size it with the parent's CSS.
- **`useCallback` on render** prevents unnecessary effect restarts when parent re-renders.
- **All utilities inlined** so the file stands alone with zero imports beyond React.

---

## 3. Source Mode Architecture

Two-canvas pattern for rendering ASCII from an image, video, or webcam source. The hidden
"sampler" canvas downsamples the source to grid resolution, then brightness + color values
are extracted from the pixel data.

```tsx
// Inside the component body, before the render callback:

const samplerRef = useRef<HTMLCanvasElement | null>(null)
const samplerCtxRef = useRef<CanvasRenderingContext2D | null>(null)

// Create the sampler canvas once (never attached to the DOM)
useEffect(() => {
  const sampler = document.createElement('canvas')
  // willReadFrequently: true tells the browser to keep the pixel data in CPU memory
  // instead of GPU textures, making getImageData ~10x faster for repeated reads
  const sCtx = sampler.getContext('2d', { willReadFrequently: true })
  samplerRef.current = sampler
  samplerCtxRef.current = sCtx
}, [])
```

### Sampling the source

```tsx
interface GridCell {
  brightness: number  // 0-1, linearized luminance
  r: number           // sRGB 0-255
  g: number           // sRGB 0-255
  b: number           // sRGB 0-255
}

function sampleSource(
  source: CanvasImageSource,
  cols: number,
  rows: number,
  sampler: HTMLCanvasElement,
  sCtx: CanvasRenderingContext2D
): GridCell[] {
  // Resize sampler to match grid dimensions exactly
  sampler.width = cols
  sampler.height = rows

  // drawImage downsamples the source to grid resolution in one GPU call.
  // The browser's bilinear filter handles anti-aliasing.
  sCtx.drawImage(source, 0, 0, cols, rows)

  const imageData = sCtx.getImageData(0, 0, cols, rows)
  const data = imageData.data  // Uint8ClampedArray: [R, G, B, A, R, G, B, A, ...]
  const grid: GridCell[] = new Array(cols * rows)

  for (let i = 0; i < cols * rows; i++) {
    const offset = i * 4
    const r = data[offset]
    const g = data[offset + 1]
    const b = data[offset + 2]

    // BT.709 luminance from linearized sRGB
    const lum = 0.2126 * SRGB_LUT[r] + 0.7152 * SRGB_LUT[g] + 0.0722 * SRGB_LUT[b]

    grid[i] = { brightness: lum, r, g, b }
  }

  return grid
}
```

### Render loop with source sampling

```tsx
const render = useCallback(
  ({ ctx, width, height }: CanvasState) => {
    if (!sourceElement || !samplerRef.current || !samplerCtxRef.current) return

    const cols = Math.ceil(width / cellW)
    const rows = Math.ceil(height / cellH)
    const grid = sampleSource(
      sourceElement,
      cols, rows,
      samplerRef.current,
      samplerCtxRef.current
    )

    ctx.fillStyle = bgColor
    ctx.fillRect(0, 0, width, height)
    ctx.font = `${fontSize}px monospace`
    ctx.textBaseline = 'top'

    for (let row = 0; row < rows; row++) {
      for (let col = 0; col < cols; col++) {
        const cell = grid[row * cols + col]
        const charIdx = Math.floor(cell.brightness * (charset.length - 1))

        // Full-color mode: use the source pixel's color
        ctx.fillStyle = `rgb(${cell.r},${cell.g},${cell.b})`
        ctx.fillText(charset[charIdx], col * cellW, row * cellH)
      }
    }
  },
  [sourceElement, charset, fontSize, cellW, cellH, bgColor]
)
```

### Why two canvases

| Canvas | Purpose | Size | Visible |
|--------|---------|------|---------|
| **Display** | Final rendered ASCII characters | Full container size (CSS pixels x DPR) | Yes |
| **Sampler** | Downsample source to grid resolution | `cols x rows` (e.g. 120x40) | No (offscreen) |

Separating them avoids reading back from the display canvas (which would include already-rendered
characters) and lets the browser optimize the sampler for repeated `getImageData` calls via the
`willReadFrequently` hint.

---

## 4. Image Loading

Loading images from URL, `File` object, or existing `HTMLImageElement`.

### From URL (with CORS)

```tsx
interface ImageSourceProps {
  /** Image URL, File, or HTMLImageElement */
  src: string | File | HTMLImageElement
}

function useImageSource(src: string | File | HTMLImageElement) {
  const [image, setImage] = useState<HTMLImageElement | null>(null)

  useEffect(() => {
    if (src instanceof HTMLImageElement) {
      if (src.complete && src.naturalWidth > 0) {
        setImage(src)
      } else {
        src.onload = () => setImage(src)
      }
      return
    }

    const img = new Image()

    // CORS: required for getImageData on cross-origin images.
    // Without this, drawImage works but getImageData throws a SecurityError.
    // The server must send Access-Control-Allow-Origin for this to work.
    if (typeof src === 'string' && !src.startsWith('data:')) {
      img.crossOrigin = 'anonymous'
    }

    img.onload = () => setImage(img)
    img.onerror = () => console.error(`Failed to load image: ${typeof src === 'string' ? src : 'File'}`)

    if (src instanceof File) {
      const url = URL.createObjectURL(src)
      img.src = url
      return () => URL.revokeObjectURL(url)
    }

    img.src = src
  }, [src])

  return image
}
```

### Responding to src changes

The `useEffect` dependency on `src` handles prop changes. When `src` changes:
1. A new `Image` element is created
2. The old one is garbage collected (no manual cleanup needed for Image elements)
3. For `File` sources, the previous `objectURL` is revoked in the cleanup function

### Integration with the render loop

```tsx
export function AsciiImage({ src, charset = ' .:-=+*#%@', fontSize = 14, ...rest }: Props) {
  const image = useImageSource(src)

  const render = useCallback(
    ({ ctx, width, height }: CanvasState) => {
      if (!image) return  // Not loaded yet — skip frame
      // ... sample and render
    },
    [image, charset, fontSize]
  )

  // Re-run effect when image changes so the first frame renders immediately
  const canvasRef = useCanvas(render, [image])

  return <canvas ref={canvasRef} aria-hidden="true" style={{ width: '100%', height: '100%', display: 'block' }} />
}
```

---

## 5. Video / Webcam Source

Processing video elements and webcam feeds. The render loop samples the `<video>` element
on every animation frame via `drawImage`.

### Video from URL

```tsx
function useVideoSource(src: string | MediaStream | undefined) {
  const videoRef = useRef<HTMLVideoElement | null>(null)
  const [ready, setReady] = useState(false)

  useEffect(() => {
    if (!src) return

    const video = document.createElement('video')
    video.playsInline = true
    video.muted = true
    video.loop = true

    // Prevent the video from appearing in the DOM
    video.style.position = 'absolute'
    video.style.width = '0'
    video.style.height = '0'
    video.style.opacity = '0'
    video.style.pointerEvents = 'none'

    if (src instanceof MediaStream) {
      video.srcObject = src
    } else {
      video.crossOrigin = 'anonymous'
      video.src = src
    }

    video.onloadeddata = () => {
      setReady(true)
      video.play()
    }

    // Append to body so the browser will decode frames
    // (some browsers pause decode on detached elements)
    document.body.appendChild(video)
    videoRef.current = video

    return () => {
      video.pause()

      // Stop all MediaStream tracks (webcam)
      if (video.srcObject instanceof MediaStream) {
        video.srcObject.getTracks().forEach((track) => track.stop())
        video.srcObject = null
      }

      // Revoke blob URLs
      if (typeof src === 'string' && src.startsWith('blob:')) {
        URL.revokeObjectURL(src)
      }

      document.body.removeChild(video)
      videoRef.current = null
      setReady(false)
    }
  }, [src])

  return { video: ready ? videoRef.current : null, ready }
}
```

### Webcam hook

```tsx
function useWebcam(enabled: boolean, constraints?: MediaStreamConstraints) {
  const [stream, setStream] = useState<MediaStream | null>(null)

  useEffect(() => {
    if (!enabled) return

    let cancelled = false

    const defaultConstraints: MediaStreamConstraints = {
      video: { width: { ideal: 640 }, height: { ideal: 480 }, facingMode: 'user' },
      audio: false,
      ...constraints,
    }

    navigator.mediaDevices
      .getUserMedia(defaultConstraints)
      .then((mediaStream) => {
        if (cancelled) {
          mediaStream.getTracks().forEach((t) => t.stop())
          return
        }
        setStream(mediaStream)
      })
      .catch((err) => {
        console.error('Webcam access denied:', err)
      })

    return () => {
      cancelled = true
      setStream((prev) => {
        prev?.getTracks().forEach((t) => t.stop())
        return null
      })
    }
  }, [enabled])

  return stream
}
```

### Cleanup checklist for video/webcam

| Resource | Cleanup |
|----------|---------|
| `MediaStream` tracks | `track.stop()` on every track — prevents the webcam LED staying on |
| `video.srcObject` | Set to `null` after stopping tracks |
| Blob URLs from `URL.createObjectURL` | `URL.revokeObjectURL(url)` to free memory |
| DOM-attached `<video>` | `document.body.removeChild(video)` |
| `requestAnimationFrame` | `cancelAnimationFrame(rafId)` via the `useCanvas` hook |

---

## 6. Multi-Layer Compositing

Canvas 2D blend modes enable layering multiple effects without WebGL. Each layer renders
to the same canvas with a specific `globalCompositeOperation` and `globalAlpha`.

### Layer interface

```tsx
interface CompositeLayer {
  /** Unique identifier for the layer */
  id: string
  /** Canvas blend mode. Default: 'source-over' */
  blendMode: GlobalCompositeOperation
  /** Layer opacity 0-1. Default: 1 */
  opacity: number
  /** Render function for this layer */
  render: (ctx: CanvasRenderingContext2D, width: number, height: number, time: number) => void
  /** Whether this layer is active. Default: true */
  enabled: boolean
}
```

### Layer compositor

```tsx
function compositeLayers(
  ctx: CanvasRenderingContext2D,
  width: number,
  height: number,
  time: number,
  layers: CompositeLayer[]
) {
  for (const layer of layers) {
    if (!layer.enabled) continue

    ctx.save()
    ctx.globalCompositeOperation = layer.blendMode
    ctx.globalAlpha = layer.opacity
    layer.render(ctx, width, height, time)
    ctx.restore()
  }
}
```

### Example: ASCII + glow + vignette

```tsx
const layers: CompositeLayer[] = [
  {
    id: 'base-ascii',
    blendMode: 'source-over',
    opacity: 1,
    enabled: true,
    render: (ctx, w, h, t) => {
      // Standard ASCII character rendering
      ctx.fillStyle = '#0a0a0a'
      ctx.fillRect(0, 0, w, h)
      ctx.font = '14px monospace'
      ctx.textBaseline = 'top'
      ctx.fillStyle = '#00ff41'
      // ... grid loop with char mapping
    },
  },
  {
    id: 'glow',
    blendMode: 'screen',  // Additive blending — brightens without darkening
    opacity: 0.3,
    enabled: true,
    render: (ctx, w, h, _t) => {
      // Blur the existing content for a glow effect.
      // screen blend + blur = bloom approximation without WebGL.
      ctx.filter = 'blur(8px)'
      ctx.drawImage(ctx.canvas, 0, 0)
      ctx.filter = 'none'
    },
  },
  {
    id: 'vignette',
    blendMode: 'multiply',  // Darkens — black edges fade in
    opacity: 1,
    enabled: true,
    render: (ctx, w, h, _t) => {
      const gradient = ctx.createRadialGradient(
        w / 2, h / 2, Math.min(w, h) * 0.3,
        w / 2, h / 2, Math.max(w, h) * 0.7
      )
      gradient.addColorStop(0, 'rgba(255,255,255,1)')
      gradient.addColorStop(1, 'rgba(0,0,0,1)')
      ctx.fillStyle = gradient
      ctx.fillRect(0, 0, w, h)
    },
  },
]

// In the render callback:
const render = useCallback(
  ({ ctx, width, height, time }: CanvasState) => {
    compositeLayers(ctx, width, height, time, layers)
  },
  [layers]
)
```

### Blend mode reference for common effects

| Effect | Blend Mode | Why |
|--------|-----------|-----|
| Glow / bloom | `screen` | Adds light: `result = 1 - (1-a)(1-b)`. Black is transparent. |
| Vignette | `multiply` | Darkens: `result = a * b`. White is transparent. |
| Color overlay | `overlay` | Combines multiply + screen. Preserves contrast. |
| Noise/grain | `overlay` at low opacity | Mid-gray noise shifts brightness up/down. |
| Additive light | `lighter` | Pure addition: `result = a + b`. Clips at white. |
| Silhouette/mask | `destination-in` | Only keeps pixels where the new layer is opaque. |
| Erase/cutout | `destination-out` | Removes pixels where the new layer is opaque. |

---

## 7. Mouse Interaction

Character offset and brightness modulation near the cursor. Tracks mouse position normalized
to 0-1 coordinates and applies distance-based falloff to the ASCII grid.

### Mouse position tracking

```tsx
function useMousePosition(canvasRef: React.RefObject<HTMLCanvasElement | null>) {
  const mouseRef = useRef({ x: 0.5, y: 0.5, active: false })

  useEffect(() => {
    const canvas = canvasRef.current
    if (!canvas) return

    function onMove(e: MouseEvent) {
      const rect = canvas!.getBoundingClientRect()
      mouseRef.current = {
        x: (e.clientX - rect.left) / rect.width,
        y: (e.clientY - rect.top) / rect.height,
        active: true,
      }
    }

    function onLeave() {
      mouseRef.current = { ...mouseRef.current, active: false }
    }

    canvas.addEventListener('pointermove', onMove)
    canvas.addEventListener('pointerleave', onLeave)

    return () => {
      canvas.removeEventListener('pointermove', onMove)
      canvas.removeEventListener('pointerleave', onLeave)
    }
  }, [canvasRef])

  return mouseRef
}
```

### Distance falloff

```tsx
interface MouseInteraction {
  /** Effect radius in normalized coordinates (0-1 range). Default: 0.15 */
  radius: number
  /** Effect strength at the cursor center. Default: 1.0 */
  strength: number
  /** 'push' offsets chars outward, 'attract' pulls toward cursor, 'brighten' adds light. */
  mode: 'push' | 'attract' | 'brighten'
}

function mouseInfluence(
  col: number,
  row: number,
  cols: number,
  rows: number,
  mouse: { x: number; y: number; active: boolean },
  interaction: MouseInteraction
): number {
  if (!mouse.active) return 0

  // Normalize grid position to 0-1
  const nx = col / cols
  const ny = row / rows

  // Euclidean distance (aspect-corrected: treat the grid as square)
  const dx = nx - mouse.x
  const dy = ny - mouse.y
  const dist = Math.sqrt(dx * dx + dy * dy)

  if (dist > interaction.radius) return 0

  // Smooth falloff: 1 at center, 0 at radius edge
  // smoothstep produces a perceptually pleasing curve
  const t = dist / interaction.radius
  const falloff = 1 - t * t * (3 - 2 * t)

  return falloff * interaction.strength
}
```

### Applying to the render loop

```tsx
// Inside the grid render loop:
for (let row = 0; row < rows; row++) {
  for (let col = 0; col < cols; col++) {
    let brightness = baseBrightness[row * cols + col]
    const influence = mouseInfluence(col, row, cols, rows, mouseRef.current, interaction)

    switch (interaction.mode) {
      case 'brighten':
        brightness = Math.min(1, brightness + influence * 0.5)
        break
      case 'push': {
        // Offset character selection toward brighter (more "dense") chars
        const pushOffset = Math.floor(influence * charset.length * 0.3)
        const charIdx = Math.min(charset.length - 1, Math.floor(brightness * (charset.length - 1)) + pushOffset)
        ctx.fillText(charset[charIdx], col * cellW, row * cellH)
        continue
      }
      case 'attract': {
        // Offset toward darker (less "dense") chars
        const attractOffset = Math.floor(influence * charset.length * 0.3)
        const charIdx = Math.max(0, Math.floor(brightness * (charset.length - 1)) - attractOffset)
        ctx.fillText(charset[charIdx], col * cellW, row * cellH)
        continue
      }
    }

    const charIdx = Math.floor(brightness * (charset.length - 1))
    ctx.fillText(charset[charIdx], col * cellW, row * cellH)
  }
}
```

### Touch support

`pointermove` and `pointerleave` handle both mouse and touch events via the Pointer Events
API. No separate touch event listeners needed. For multi-touch, track `e.pointerId` and
store multiple positions.

---

## 8. Full Props Interface

The complete configurable surface for an ASCII shader component. Every prop is typed and
documented with default values and valid ranges.

```tsx
// ---------------------------------------------------------------------------
// Enums / union types
// ---------------------------------------------------------------------------

type DitherAlgorithm =
  | 'none'
  | 'threshold'
  | 'random'
  | 'bayer2x2'
  | 'bayer4x4'
  | 'bayer8x8'
  | 'blueNoise'
  | 'floydSteinberg'
  | 'atkinson'
  | 'sierra'
  | 'sierraLite'
  | 'stucki'
  | 'jarvisJudiceNinke'
  | 'riemersma'

type ColorMode =
  | 'grayscale'
  | 'fullcolor'
  | 'matrix'       // green-on-black (#00ff41)
  | 'amber'        // amber-on-black (#ffb000)
  | 'duotone'      // two-color with palette
  | 'custom'       // single customColor

type ArtStyle =
  | 'classic'      // standard char-mapped luminance
  | 'halftone'     // geometric dots/shapes
  | 'braille'      // 2x4 sub-cell braille patterns
  | 'dotcross'     // edge-aware dual ramps
  | 'line'         // oriented line segments
  | 'particles'    // scattered bright points
  | 'retro'        // duotone + grain + posterize
  | 'terminal'     // green phosphor + scanlines
  | 'custom'       // user render function

type CharsetPreset =
  | 'standard'     // ' .:-=+*#%@'               (10 chars)
  | 'blocks'       // ' ░▒▓█'                    (5 chars)
  | 'minimal'      // ' .oO@'                    (5 chars)
  | 'detailed'     // full 68-char ramp
  | 'binary'       // ' █'                       (2 chars)
  | 'braille'      // Unicode braille patterns    (256 chars)
  | 'box'          // ' ╌│┐┘─┤┬┼└├┴┌'            (14 chars)
  | 'katakana'     // Katakana half-width set

type HalftoneShape = 'circle' | 'square' | 'diamond' | 'line' | 'cross' | 'star'

type SourceMode = 'generative' | 'source' | 'hybrid'

// ---------------------------------------------------------------------------
// Sub-config interfaces
// ---------------------------------------------------------------------------

interface CRTConfig {
  /** Barrel distortion strength. Range: 0-0.3. Default: 0.15 */
  curvature?: number
  /** Scanline darkness. Range: 0-1. Default: 0.3 */
  scanlineIntensity?: number
  /** Bloom glow strength. Range: 0-1. Default: 0.15 */
  bloomStrength?: number
  /** Edge darkening. Range: 0-1. Default: 0.4 */
  vignetteStrength?: number
  /** Phosphor RGB sub-pixel mask opacity. Range: 0-1. Default: 0.3 */
  phosphorMask?: number
}

interface BloomConfig {
  /** Blur radius in CSS pixels. Range: 1-20. Default: 8 */
  radius?: number
  /** Blend strength. Range: 0-1. Default: 0.3 */
  strength?: number
  /** Brightness threshold for bloom activation. Range: 0-1. Default: 0.5 */
  threshold?: number
}

interface GlitchConfig {
  /** How often glitches occur (probability per frame). Range: 0-1. Default: 0.05 */
  frequency?: number
  /** Max pixel displacement. Range: 0-50. Default: 20 */
  displacement?: number
  /** Enable RGB channel splitting. Default: true */
  colorSplit?: boolean
  /** Enable horizontal scan tears. Default: true */
  scanTear?: boolean
}

interface MouseConfig {
  /** Effect radius in normalized coordinates. Range: 0-0.5. Default: 0.15 */
  radius?: number
  /** Effect strength at cursor center. Range: 0-2. Default: 1.0 */
  strength?: number
  /** Interaction behavior. Default: 'brighten' */
  mode?: 'push' | 'attract' | 'brighten'
}

// ---------------------------------------------------------------------------
// Main props
// ---------------------------------------------------------------------------

interface AsciiShaderProps {
  // --- Source ---
  /** Image/video source. URL string, File, HTMLImageElement, HTMLVideoElement, or MediaStream (webcam). */
  src?: string | File | HTMLImageElement | HTMLVideoElement | MediaStream
  /** Rendering mode. 'generative' = no input, 'source' = input required, 'hybrid' = input + procedural overlay. Default: 'generative' */
  mode?: SourceMode

  // --- Grid ---
  /** Font size in CSS pixels. Controls grid density — smaller = more cells = more detail. Range: 4-48. Default: 14 */
  fontSize?: number
  /** Character cell width as a fraction of cell height. Corrects monospace aspect ratio. Range: 0.4-1.0. Default: 0.6 */
  cellSpacing?: number
  /** Maximum grid columns. Prevents excessive CPU usage on wide displays. Range: 40-800. Default: 400 */
  maxCols?: number
  /** Maximum grid rows. Range: 20-400. Default: 200 */
  maxRows?: number

  // --- Characters ---
  /** Character ramp string from dark (first) to bright (last). Overrides charsetPreset. */
  charset?: string
  /** Named character set preset. Ignored if charset is provided. Default: 'standard' */
  charsetPreset?: CharsetPreset
  /** Invert the character ramp (bright-to-dark instead of dark-to-bright). Default: false */
  invertCharset?: boolean

  // --- Dithering ---
  /** Dithering algorithm applied to the brightness grid before character mapping. Default: 'none' */
  ditherAlgorithm?: DitherAlgorithm
  /** Dithering intensity multiplier. 0 = no dither, 1 = standard, 2 = exaggerated. Range: 0-2. Default: 0.5 */
  ditherStrength?: number
  /** Number of quantization levels for multi-level dithering. Range: 2-256. Default: 2 (binary) */
  ditherLevels?: number

  // --- Color ---
  /** Color rendering mode. Default: 'matrix' */
  colorMode?: ColorMode
  /** Hex color for 'custom' colorMode. Default: '#ffffff' */
  customColor?: string
  /** Named retro palette for 'duotone' mode (e.g. 'gameboy', 'cga', 'commodore'). */
  palette?: string
  /** Perceptual color space for interpolation and matching. Default: 'srgb' */
  colorSpace?: 'srgb' | 'oklab' | 'oklch'
  /** Background color. Default: '#0a0a0a' */
  bgColor?: string

  // --- Art Style ---
  /** Visual rendering style. Default: 'classic' */
  artStyle?: ArtStyle
  /** Halftone dot shape (only used when artStyle is 'halftone'). Default: 'circle' */
  halftoneShape?: HalftoneShape
  /** Halftone maximum dot size in pixels. Range: 2-20. Default: 8 */
  halftoneSize?: number
  /** Line segment length for 'line' art style. Range: 2-20. Default: 8 */
  lineLength?: number
  /** Line stroke width for 'line' art style. Range: 0.5-4. Default: 1.5 */
  lineThickness?: number

  // --- Post-FX ---
  /** CRT screen simulation. true = default config, or provide CRTConfig. Default: false */
  crt?: boolean | CRTConfig
  /** Scanline overlay. true = default intensity (0.3), or provide number 0-1. Default: false */
  scanlines?: boolean | number
  /** Bloom/glow effect. true = default config, or provide BloomConfig. Default: false */
  bloom?: boolean | BloomConfig
  /** Chromatic aberration pixel offset. Range: 0-10. Default: 0 */
  chromaticAberration?: number
  /** Vignette edge darkening strength. Range: 0-1. Default: 0 */
  vignette?: number
  /** Film grain noise intensity. Range: 0-1. Default: 0 */
  grain?: number
  /** Glitch effect. true = default config, or provide GlitchConfig. Default: false */
  glitch?: boolean | GlitchConfig

  // --- Adjustments ---
  /** Brightness offset applied after sampling. Range: -0.5 to 0.5. Default: 0 */
  brightness?: number
  /** Contrast multiplier. Range: 0.5-2.0. Default: 1.0 */
  contrast?: number
  /** Gamma correction exponent. Range: 0.3-3.0. Default: 1.0 */
  gamma?: number
  /** Invert brightness values (negative image). Default: false */
  invert?: boolean
  /** Sobel edge detection strength blended into brightness. Range: 0-1. Default: 0 */
  edgeBoost?: number

  // --- Animation ---
  /** Time multiplier for animated effects. 0 = frozen. Default: 1.0 */
  speed?: number
  /** Target frames per second. 0 = uncapped (synced to display refresh). Range: 0-144. Default: 0 */
  fps?: number

  // --- Mouse ---
  /** Mouse interaction config. Disabled if undefined. */
  mouse?: MouseConfig

  // --- Layout ---
  /** CSS class name for the canvas element. */
  className?: string
  /** Inline styles for the canvas element. */
  style?: React.CSSProperties

  // --- Callbacks ---
  /** Called once after the first frame renders. Receives grid dimensions. */
  onReady?: (cols: number, rows: number) => void
  /** Called when the source media (image/video) finishes loading. */
  onSourceLoad?: () => void
}
```

### Resolving shorthand props

Several props accept `boolean | Config` for ergonomics. Resolve them at the top of the
component before the render loop:

```tsx
function resolveConfig<T>(value: boolean | T | undefined, defaults: T): T | null {
  if (value === false || value === undefined) return null
  if (value === true) return defaults
  return { ...defaults, ...value }
}

// Usage:
const crtConfig = resolveConfig<CRTConfig>(crt, {
  curvature: 0.15,
  scanlineIntensity: 0.3,
  bloomStrength: 0.15,
  vignetteStrength: 0.4,
  phosphorMask: 0.3,
})

const bloomConfig = resolveConfig<BloomConfig>(bloom, {
  radius: 8,
  strength: 0.3,
  threshold: 0.5,
})

const glitchConfig = resolveConfig<GlitchConfig>(glitch, {
  frequency: 0.05,
  displacement: 20,
  colorSplit: true,
  scanTear: true,
})
```

---

## 9. Output Checklist

Verification before generating any ASCII/dither component.

### File structure
- [ ] Single `.tsx` file
- [ ] `'use client'` directive at the top
- [ ] Zero imports beyond `react` (`useRef`, `useEffect`, `useCallback`, `useState`)
- [ ] All utilities (sRGB LUT, luminance, dither algorithms, noise functions) inlined
- [ ] Named export for the component
- [ ] Props interface with TypeScript types

### Canvas setup
- [ ] `useCanvas` hook (or equivalent inline) handles the animation loop
- [ ] `ResizeObserver` for responsive container-fill sizing
- [ ] `devicePixelRatio` scaling: backing store = CSS size * DPR
- [ ] `ctx.setTransform(dpr, 0, 0, dpr, 0, 0)` so draw calls use CSS pixel coordinates
- [ ] `requestAnimationFrame` loop with `cancelAnimationFrame` cleanup
- [ ] `ResizeObserver.disconnect()` in the cleanup function

### Accessibility
- [ ] `aria-hidden="true"` on the canvas element (decorative content)
- [ ] `prefers-reduced-motion: reduce` check — renders a single static frame instead of animating
- [ ] No flashing effects faster than 3Hz for photosensitivity compliance

### Props & defaults
- [ ] All props have sensible defaults (component works with zero props)
- [ ] Numeric ranges validated or clamped where misuse could break rendering
- [ ] `className` and `style` props forwarded to the canvas for layout flexibility
- [ ] `useCallback` on the render function with correct dependency array

### Rendering
- [ ] sRGB linearization via LUT for luminance calculation (never raw `(r + g + b) / 3`)
- [ ] BT.709 luminance coefficients: `0.2126 * R + 0.7152 * G + 0.0722 * B`
- [ ] Aspect ratio correction: `cellWidth = fontSize * cellSpacing` (not 1:1)
- [ ] Character mapping: `charset[Math.floor(brightness * (charset.length - 1))]`
- [ ] `ctx.textBaseline = 'top'` for consistent character positioning

### Source mode (if applicable)
- [ ] Hidden sampler canvas with `willReadFrequently: true`
- [ ] `drawImage` downsamples source to grid resolution
- [ ] `getImageData` reads pixel data from sampler, not display canvas
- [ ] Image CORS: `img.crossOrigin = 'anonymous'` for cross-origin URLs
- [ ] Video/webcam: tracks stopped and `srcObject` nulled on cleanup
- [ ] Blob URLs revoked via `URL.revokeObjectURL` on cleanup

### Performance
- [ ] Grid capped by `maxCols` / `maxRows` to prevent frame drops on large displays
- [ ] No per-cell allocations in the render loop (pre-allocate arrays outside the loop)
- [ ] `Float32Array` for brightness grids instead of regular arrays
- [ ] FPS limiting via elapsed time check, not `setInterval`
