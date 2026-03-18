# Performance Optimization Reference

Production techniques for high-performance ASCII/dither rendering across Canvas 2D, WebGL2,
WebGPU, and Web Workers. Every code sample is production-ready TypeScript, GLSL, or WGSL.

---

## 1. Canvas 2D Optimization

### Float32Array Grid Storage

Never use `number[][]` for luminance/dither grids. A flat `Float32Array` avoids pointer
indirection, is GC-invisible, and enables direct transfer to workers.

```typescript
interface AsciiGrid {
  data: Float32Array
  cols: number
  rows: number
}

function createGrid(cols: number, rows: number): AsciiGrid {
  return { data: new Float32Array(cols * rows), cols, rows }
}

function gridGet(g: AsciiGrid, col: number, row: number): number {
  return g.data[row * g.cols + col]
}

function gridSet(g: AsciiGrid, col: number, row: number, val: number): void {
  g.data[row * g.cols + col] = val
}
```

### Module-Scope LUT Caching

Compute lookup tables once at module load. Never inside a render loop.

```typescript
// sRGB linearization LUT — computed once, reused every frame
const SRGB_TO_LINEAR = new Float32Array(256)
for (let i = 0; i < 256; i++) {
  const s = i / 255
  SRGB_TO_LINEAR[i] = s <= 0.04045 ? s / 12.92 : ((s + 0.055) / 1.055) ** 2.4
}

// BT.709 luminance from raw sRGB byte triplet
function luminance(r: number, g: number, b: number): number {
  return 0.2126 * SRGB_TO_LINEAR[r] + 0.7152 * SRGB_TO_LINEAR[g] + 0.0722 * SRGB_TO_LINEAR[b]
}

// Bayer threshold LUT (8x8, normalized to 0..1)
const BAYER_8X8 = new Float32Array(64)
;(function buildBayer() {
  for (let y = 0; y < 8; y++) {
    for (let x = 0; x < 8; x++) {
      let value = 0
      let xc = x ^ y, yc = y
      for (let bit = 0; bit < 3; bit++) {
        value = (value << 2) | ((((xc >> (2 - bit)) & 1) << 1) | ((yc >> (2 - bit)) & 1))
      }
      BAYER_8X8[y * 8 + x] = (value + 0.5) / 64
    }
  }
})()

// ASCII ramp — index directly by quantized luminance bucket
const ASCII_RAMP = ' .:-=+*#%@'
const RAMP_LEN = ASCII_RAMP.length
const RAMP_LUT = new Uint8Array(256)
for (let i = 0; i < 256; i++) {
  RAMP_LUT[i] = Math.min(Math.floor((i / 255) * RAMP_LEN), RAMP_LEN - 1)
}
```

### Zero String Allocation in Hot Loops

String concatenation in a per-cell loop creates hundreds of intermediate strings per frame.
Pre-allocate a typed array of character codes and batch-convert once.

```typescript
function renderGridToString(grid: AsciiGrid): string {
  const { data, cols, rows } = grid
  // +1 per row for newline
  const buf = new Uint16Array(cols * rows + rows)
  let idx = 0

  for (let r = 0; r < rows; r++) {
    for (let c = 0; c < cols; c++) {
      const lum = data[r * cols + c]
      const charIdx = Math.min(Math.floor(lum * RAMP_LEN), RAMP_LEN - 1)
      buf[idx++] = ASCII_RAMP.charCodeAt(charIdx)
    }
    buf[idx++] = 10 // newline
  }

  return String.fromCharCode.apply(null, buf.subarray(0, idx) as unknown as number[])
}

// For Canvas 2D rendering (no string needed — draw chars directly):
function renderGridToCanvas(
  ctx: CanvasRenderingContext2D,
  grid: AsciiGrid,
  cellW: number,
  cellH: number
): void {
  const { data, cols, rows } = grid

  ctx.clearRect(0, 0, cols * cellW, rows * cellH)

  for (let r = 0; r < rows; r++) {
    const y = r * cellH + cellH
    for (let c = 0; c < cols; c++) {
      const lum = data[r * cols + c]
      const charIdx = Math.min(Math.floor(lum * RAMP_LEN), RAMP_LEN - 1)
      ctx.fillText(ASCII_RAMP[charIdx], c * cellW, y)
    }
  }
}
```

### Grid Cell Cap

Cap the maximum grid resolution to prevent runaway frame times on large viewports.

```typescript
const MAX_GRID_CELLS = 20_000 // ~200×100 or ~160×125

function clampGridSize(
  canvasW: number,
  canvasH: number,
  cellW: number,
  cellH: number
): { cols: number; rows: number; cellW: number; cellH: number } {
  let cols = Math.floor(canvasW / cellW)
  let rows = Math.floor(canvasH / cellH)
  const total = cols * rows

  if (total > MAX_GRID_CELLS) {
    const scale = Math.sqrt(MAX_GRID_CELLS / total)
    cols = Math.max(1, Math.floor(cols * scale))
    rows = Math.max(1, Math.floor(rows * scale))
    cellW = canvasW / cols
    cellH = canvasH / rows
  }

  return { cols, rows, cellW, cellH }
}
```

### requestAnimationFrame Pattern

Use a single rAF loop with frame time tracking. Never nest rAF calls or create multiple loops.

```typescript
interface RenderState {
  running: boolean
  rafId: number
  lastFrameTime: number
  frameTimeSmoothed: number
  targetFps: number
  frameInterval: number
}

function createRenderLoop(
  render: (dt: number, state: RenderState) => void,
  targetFps: number = 60
): RenderState {
  const state: RenderState = {
    running: false,
    rafId: 0,
    lastFrameTime: 0,
    frameTimeSmoothed: 16.67,
    targetFps,
    frameInterval: 1000 / targetFps,
  }

  function tick(now: number): void {
    if (!state.running) return

    const dt = now - state.lastFrameTime

    // Skip frame if we're ahead of target (throttle to targetFps)
    if (dt < state.frameInterval * 0.9) {
      state.rafId = requestAnimationFrame(tick)
      return
    }

    state.lastFrameTime = now

    // Exponential moving average of frame time (α=0.1)
    state.frameTimeSmoothed = state.frameTimeSmoothed * 0.9 + dt * 0.1

    render(dt, state)

    state.rafId = requestAnimationFrame(tick)
  }

  state.running = true
  state.lastFrameTime = performance.now()
  state.rafId = requestAnimationFrame(tick)

  return state
}

function stopRenderLoop(state: RenderState): void {
  state.running = false
  cancelAnimationFrame(state.rafId)
}
```

### devicePixelRatio Handling

Scale the canvas backing store to match device DPR, but keep the ASCII grid resolution
independent of DPR to avoid wasted computation.

```typescript
function setupHiDPICanvas(
  canvas: HTMLCanvasElement,
  width: number,
  height: number
): CanvasRenderingContext2D {
  const dpr = window.devicePixelRatio || 1

  canvas.width = width * dpr
  canvas.height = height * dpr
  canvas.style.width = `${width}px`
  canvas.style.height = `${height}px`

  const ctx = canvas.getContext('2d')!
  ctx.scale(dpr, dpr)

  // Crisp text rendering at any DPR
  ctx.imageSmoothingEnabled = false
  ctx.textBaseline = 'top'

  return ctx
}

// Resize observer for responsive canvases
function observeResize(
  canvas: HTMLCanvasElement,
  onResize: (width: number, height: number) => void
): ResizeObserver {
  const observer = new ResizeObserver((entries) => {
    const entry = entries[0]
    if (!entry) return
    const { width, height } = entry.contentRect
    onResize(Math.floor(width), Math.floor(height))
  })
  observer.observe(canvas)
  return observer
}
```

---

## 2. WebGL2 Shader Pipeline

Use WebGL2 as a post-processing pass: render ASCII to a Canvas 2D, upload as a texture,
apply shader effects (CRT, bloom, color grading), display the result.

### Complete WebGL2 Pipeline

```typescript
// ─── Types ───

interface ShaderProgram {
  program: WebGLProgram
  uniforms: Record<string, WebGLUniformLocation>
  attributes: Record<string, number>
}

interface GLPipeline {
  gl: WebGL2RenderingContext
  program: ShaderProgram
  vao: WebGLVertexArrayObject
  texture: WebGLTexture
  sourceCanvas: HTMLCanvasElement
  displayCanvas: HTMLCanvasElement
}

// ─── Shader Compilation ───

function compileShader(
  gl: WebGL2RenderingContext,
  type: number,
  source: string
): WebGLShader {
  const shader = gl.createShader(type)
  if (!shader) throw new Error('Failed to create shader')

  gl.shaderSource(shader, source)
  gl.compileShader(shader)

  if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS)) {
    const log = gl.getShaderInfoLog(shader)
    gl.deleteShader(shader)
    throw new Error(`Shader compile error: ${log}`)
  }

  return shader
}

function linkProgram(
  gl: WebGL2RenderingContext,
  vertSrc: string,
  fragSrc: string,
  uniformNames: string[],
  attributeNames: string[]
): ShaderProgram {
  const vert = compileShader(gl, gl.VERTEX_SHADER, vertSrc)
  const frag = compileShader(gl, gl.FRAGMENT_SHADER, fragSrc)

  const program = gl.createProgram()!
  gl.attachShader(program, vert)
  gl.attachShader(program, frag)
  gl.linkProgram(program)

  if (!gl.getProgramParameter(program, gl.LINK_STATUS)) {
    const log = gl.getProgramInfoLog(program)
    gl.deleteProgram(program)
    throw new Error(`Program link error: ${log}`)
  }

  gl.deleteShader(vert)
  gl.deleteShader(frag)

  const uniforms: Record<string, WebGLUniformLocation> = {}
  for (const name of uniformNames) {
    const loc = gl.getUniformLocation(program, name)
    if (loc !== null) uniforms[name] = loc
  }

  const attributes: Record<string, number> = {}
  for (const name of attributeNames) {
    attributes[name] = gl.getAttribLocation(program, name)
  }

  return { program, uniforms, attributes }
}

// ─── Fullscreen Quad ───

function createFullscreenQuad(gl: WebGL2RenderingContext, program: ShaderProgram): WebGLVertexArrayObject {
  // Two triangles covering clip space
  const vertices = new Float32Array([
    -1, -1,  0, 0,
     1, -1,  1, 0,
    -1,  1,  0, 1,
     1,  1,  1, 1,
  ])

  const vao = gl.createVertexArray()!
  gl.bindVertexArray(vao)

  const vbo = gl.createBuffer()!
  gl.bindBuffer(gl.ARRAY_BUFFER, vbo)
  gl.bufferData(gl.ARRAY_BUFFER, vertices, gl.STATIC_DRAW)

  const posLoc = program.attributes['a_position']
  const uvLoc = program.attributes['a_uv']

  gl.enableVertexAttribArray(posLoc)
  gl.vertexAttribPointer(posLoc, 2, gl.FLOAT, false, 16, 0)

  gl.enableVertexAttribArray(uvLoc)
  gl.vertexAttribPointer(uvLoc, 2, gl.FLOAT, false, 16, 8)

  gl.bindVertexArray(null)

  return vao
}

// ─── Texture from Canvas ───

function createTextureFromCanvas(
  gl: WebGL2RenderingContext,
  source: HTMLCanvasElement
): WebGLTexture {
  const tex = gl.createTexture()!
  gl.bindTexture(gl.TEXTURE_2D, tex)

  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE)
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE)
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR)
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.NEAREST) // sharp pixels for ASCII

  gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE, source)

  return tex
}

function updateTexture(gl: WebGL2RenderingContext, tex: WebGLTexture, source: HTMLCanvasElement): void {
  gl.bindTexture(gl.TEXTURE_2D, tex)
  gl.texSubImage2D(gl.TEXTURE_2D, 0, 0, 0, gl.RGBA, gl.UNSIGNED_BYTE, source)
}

// ─── Vertex Shader (shared by all post-processing) ───

const FULLSCREEN_VERT = `#version 300 es
precision highp float;

in vec2 a_position;
in vec2 a_uv;
out vec2 v_uv;

void main() {
  v_uv = a_uv;
  gl_Position = vec4(a_position, 0.0, 1.0);
}
`

// ─── CRT Post-Process Fragment Shader ───

const CRT_FRAG = `#version 300 es
precision highp float;

uniform sampler2D u_texture;
uniform vec2 u_resolution;
uniform float u_time;
uniform float u_curvature;
uniform float u_scanlineIntensity;
uniform float u_bloomRadius;
uniform float u_vignetteStrength;

in vec2 v_uv;
out vec4 fragColor;

vec2 barrelDistort(vec2 uv, float k) {
  vec2 c = uv * 2.0 - 1.0;
  float r2 = dot(c, c);
  c *= 1.0 + k * r2;
  return c * 0.5 + 0.5;
}

vec3 sampleBloom(vec2 uv, float radius) {
  vec3 sum = vec3(0.0);
  vec2 texel = 1.0 / u_resolution;
  float total = 0.0;

  for (int x = -3; x <= 3; x++) {
    for (int y = -3; y <= 3; y++) {
      float w = exp(-float(x*x + y*y) / (2.0 * radius * radius));
      sum += texture(u_texture, uv + vec2(float(x), float(y)) * texel).rgb * w;
      total += w;
    }
  }
  return sum / total;
}

void main() {
  vec2 uv = barrelDistort(v_uv, u_curvature);

  // Discard outside curved screen area
  if (uv.x < 0.0 || uv.x > 1.0 || uv.y < 0.0 || uv.y > 1.0) {
    fragColor = vec4(0.0, 0.0, 0.0, 1.0);
    return;
  }

  vec3 color = texture(u_texture, uv).rgb;

  // Bloom
  vec3 bloom = sampleBloom(uv, u_bloomRadius);
  color += bloom * 0.15;

  // Scanlines
  float scanline = sin(uv.y * u_resolution.y * 3.14159) * 0.5 + 0.5;
  color *= 1.0 - pow(scanline, 2.0) * u_scanlineIntensity;

  // Phosphor mask (RGB sub-pixel pattern)
  float sub = mod(gl_FragCoord.x, 3.0);
  vec3 mask = vec3(0.8);
  if (sub < 1.0) mask.r = 1.0;
  else if (sub < 2.0) mask.g = 1.0;
  else mask.b = 1.0;
  color *= mix(vec3(1.0), mask, 0.25);

  // Vignette
  vec2 vig = uv * (1.0 - uv);
  float vigFactor = pow(vig.x * vig.y * 15.0, u_vignetteStrength);
  color *= vigFactor;

  // Flicker
  color *= 0.98 + 0.02 * sin(u_time * 8.0);

  fragColor = vec4(color, 1.0);
}
`

// ─── Pipeline Setup ───

function createGLPipeline(
  sourceCanvas: HTMLCanvasElement,
  displayCanvas: HTMLCanvasElement
): GLPipeline {
  const gl = displayCanvas.getContext('webgl2', {
    alpha: false,
    antialias: false,
    premultipliedAlpha: false,
    preserveDrawingBuffer: false,
  })
  if (!gl) throw new Error('WebGL2 not supported')

  const program = linkProgram(
    gl,
    FULLSCREEN_VERT,
    CRT_FRAG,
    ['u_texture', 'u_resolution', 'u_time', 'u_curvature', 'u_scanlineIntensity', 'u_bloomRadius', 'u_vignetteStrength'],
    ['a_position', 'a_uv']
  )

  const vao = createFullscreenQuad(gl, program)
  const texture = createTextureFromCanvas(gl, sourceCanvas)

  return { gl, program, vao, texture, sourceCanvas, displayCanvas }
}

// ─── Per-Frame Render ───

function renderGLFrame(pipeline: GLPipeline, time: number): void {
  const { gl, program, vao, texture, sourceCanvas, displayCanvas } = pipeline

  // Upload latest ASCII canvas content as texture
  updateTexture(gl, texture, sourceCanvas)

  gl.viewport(0, 0, displayCanvas.width, displayCanvas.height)

  gl.useProgram(program.program)

  gl.activeTexture(gl.TEXTURE0)
  gl.bindTexture(gl.TEXTURE_2D, texture)
  gl.uniform1i(program.uniforms['u_texture'], 0)

  gl.uniform2f(program.uniforms['u_resolution'], displayCanvas.width, displayCanvas.height)
  gl.uniform1f(program.uniforms['u_time'], time * 0.001)
  gl.uniform1f(program.uniforms['u_curvature'], 0.12)
  gl.uniform1f(program.uniforms['u_scanlineIntensity'], 0.25)
  gl.uniform1f(program.uniforms['u_bloomRadius'], 1.5)
  gl.uniform1f(program.uniforms['u_vignetteStrength'], 0.4)

  gl.bindVertexArray(vao)
  gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4)
  gl.bindVertexArray(null)
}

// ─── Readback (when you need pixel data from GL output) ───

function readbackPixels(gl: WebGL2RenderingContext, width: number, height: number): Uint8Array {
  const pixels = new Uint8Array(width * height * 4)

  // Use PBO for async readback (non-blocking)
  const pbo = gl.createBuffer()!
  gl.bindBuffer(gl.PIXEL_PACK_BUFFER, pbo)
  gl.bufferData(gl.PIXEL_PACK_BUFFER, pixels.byteLength, gl.STREAM_READ)

  gl.readPixels(0, 0, width, height, gl.RGBA, gl.UNSIGNED_BYTE, 0)

  // Fence sync — wait for GPU to finish
  const sync = gl.fenceSync(gl.SYNC_GPU_COMMANDS_COMPLETE, 0)!
  gl.flush()

  function pollFence(resolve: (data: Uint8Array) => void): void {
    const status = gl.clientWaitSync(sync, 0, 0)
    if (status === gl.WAIT_FAILED) {
      throw new Error('GL fence sync failed')
    }
    if (status === gl.TIMEOUT_EXPIRED) {
      requestAnimationFrame(() => pollFence(resolve))
      return
    }
    // Ready — copy data from PBO
    gl.bindBuffer(gl.PIXEL_PACK_BUFFER, pbo)
    gl.getBufferSubData(gl.PIXEL_PACK_BUFFER, 0, pixels)
    gl.deleteSync(sync)
    gl.deleteBuffer(pbo)
    resolve(pixels)
  }

  return pixels // For sync path; use the async version below in production
}

// Async readback (preferred — doesn't stall the pipeline)
function readbackPixelsAsync(
  gl: WebGL2RenderingContext,
  width: number,
  height: number
): Promise<Uint8Array> {
  return new Promise((resolve, reject) => {
    const byteLen = width * height * 4
    const pbo = gl.createBuffer()!
    gl.bindBuffer(gl.PIXEL_PACK_BUFFER, pbo)
    gl.bufferData(gl.PIXEL_PACK_BUFFER, byteLen, gl.STREAM_READ)

    gl.readPixels(0, 0, width, height, gl.RGBA, gl.UNSIGNED_BYTE, 0)

    const sync = gl.fenceSync(gl.SYNC_GPU_COMMANDS_COMPLETE, 0)!
    gl.flush()

    function poll(): void {
      const status = gl.clientWaitSync(sync, 0, 0)
      if (status === gl.WAIT_FAILED) {
        gl.deleteSync(sync)
        gl.deleteBuffer(pbo)
        reject(new Error('GL fence sync failed'))
        return
      }
      if (status === gl.TIMEOUT_EXPIRED) {
        requestAnimationFrame(poll)
        return
      }
      const result = new Uint8Array(byteLen)
      gl.bindBuffer(gl.PIXEL_PACK_BUFFER, pbo)
      gl.getBufferSubData(gl.PIXEL_PACK_BUFFER, 0, result)
      gl.bindBuffer(gl.PIXEL_PACK_BUFFER, null)
      gl.deleteSync(sync)
      gl.deleteBuffer(pbo)
      resolve(result)
    }

    requestAnimationFrame(poll)
  })
}

// ─── Integration: Canvas2D ASCII → WebGL Post-FX → Display ───

function createAsciiWithPostFX(
  container: HTMLElement,
  width: number,
  height: number
): { renderFrame: (drawAscii: (ctx: CanvasRenderingContext2D) => void) => void; destroy: () => void } {
  // Off-screen canvas for ASCII rendering
  const asciiCanvas = document.createElement('canvas')
  asciiCanvas.width = width
  asciiCanvas.height = height
  const asciiCtx = asciiCanvas.getContext('2d')!
  asciiCtx.font = '12px monospace'
  asciiCtx.fillStyle = '#33ff33'
  asciiCtx.textBaseline = 'top'

  // Visible canvas for WebGL output
  const displayCanvas = document.createElement('canvas')
  displayCanvas.width = width
  displayCanvas.height = height
  displayCanvas.style.width = `${width}px`
  displayCanvas.style.height = `${height}px`
  container.appendChild(displayCanvas)

  const pipeline = createGLPipeline(asciiCanvas, displayCanvas)

  let running = true
  let rafId = 0
  let drawFn: ((ctx: CanvasRenderingContext2D) => void) | null = null

  function loop(time: number): void {
    if (!running) return
    if (drawFn) {
      asciiCtx.clearRect(0, 0, width, height)
      drawFn(asciiCtx)
    }
    renderGLFrame(pipeline, time)
    rafId = requestAnimationFrame(loop)
  }

  rafId = requestAnimationFrame(loop)

  return {
    renderFrame(draw: (ctx: CanvasRenderingContext2D) => void): void {
      drawFn = draw
    },
    destroy(): void {
      running = false
      cancelAnimationFrame(rafId)
      pipeline.gl.deleteTexture(pipeline.texture)
      pipeline.gl.deleteProgram(pipeline.program.program)
      displayCanvas.remove()
    },
  }
}
```

---

## 3. WebGPU / WGSL Pipeline

WebGPU compute shaders for GPU-side dithering. The compute shader processes a source image
texture and writes dithered output to a storage texture or buffer.

**Browser support**: Chrome 113+, Edge 113+, Firefox behind flag (Nightly). Safari 18+
(macOS Sequoia / iOS 18). No IE/legacy support. Always feature-detect.

```typescript
// ─── Device Setup ───

async function initWebGPU(canvas: HTMLCanvasElement): Promise<{
  device: GPUDevice
  context: GPUCanvasContext
  format: GPUTextureFormat
}> {
  if (!navigator.gpu) {
    throw new Error('WebGPU not supported in this browser')
  }

  const adapter = await navigator.gpu.requestAdapter({
    powerPreference: 'high-performance',
  })
  if (!adapter) throw new Error('No GPU adapter found')

  const device = await adapter.requestDevice({
    requiredFeatures: [],
    requiredLimits: {
      maxStorageBufferBindingSize: adapter.limits.maxStorageBufferBindingSize,
      maxComputeWorkgroupSizeX: 16,
      maxComputeWorkgroupSizeY: 16,
    },
  })

  device.lost.then((info) => {
    console.error(`WebGPU device lost: ${info.message}`)
    if (info.reason !== 'destroyed') {
      // Attempt re-initialization
      initWebGPU(canvas)
    }
  })

  const context = canvas.getContext('webgpu')!
  const format = navigator.gpu.getPreferredCanvasFormat()
  context.configure({ device, format, alphaMode: 'opaque' })

  return { device, context, format }
}

// ─── Bayer Dither Compute Shader (WGSL) ───

const BAYER_DITHER_WGSL = `
struct Params {
  width: u32,
  height: u32,
  levels: u32,       // number of output quantization levels (2 = binary)
  bayerSize: u32,    // 2, 4, or 8
}

@group(0) @binding(0) var<uniform> params: Params;
@group(0) @binding(1) var inputTex: texture_2d<f32>;
@group(0) @binding(2) var<storage, read_write> output: array<u32>;

// Bayer 8x8 matrix (pre-normalized to 0..63)
const bayer8x8 = array<u32, 64>(
   0u, 48u, 12u, 60u,  3u, 51u, 15u, 63u,
  32u, 16u, 44u, 28u, 35u, 19u, 47u, 31u,
   8u, 56u,  4u, 52u, 11u, 59u,  7u, 55u,
  40u, 24u, 36u, 20u, 43u, 27u, 39u, 23u,
   2u, 50u, 14u, 62u,  1u, 49u, 13u, 61u,
  34u, 18u, 46u, 30u, 33u, 17u, 45u, 29u,
  10u, 58u,  6u, 54u,  9u, 57u,  5u, 53u,
  42u, 26u, 38u, 22u, 41u, 25u, 37u, 21u
);

// sRGB to linear (approximate — accurate enough for dithering)
fn srgbToLinear(s: f32) -> f32 {
  if (s <= 0.04045) {
    return s / 12.92;
  }
  return pow((s + 0.055) / 1.055, 2.4);
}

// BT.709 luminance from linear RGB
fn luminance(r: f32, g: f32, b: f32) -> f32 {
  return 0.2126 * srgbToLinear(r) + 0.7152 * srgbToLinear(g) + 0.0722 * srgbToLinear(b);
}

@compute @workgroup_size(16, 16)
fn main(@builtin(global_invocation_id) gid: vec3<u32>) {
  let x = gid.x;
  let y = gid.y;

  if (x >= params.width || y >= params.height) {
    return;
  }

  let pixel = textureLoad(inputTex, vec2<i32>(i32(x), i32(y)), 0);
  let lum = luminance(pixel.r, pixel.g, pixel.b);

  // Bayer threshold lookup
  let bx = x % params.bayerSize;
  let by = y % params.bayerSize;

  // Map to 8x8 regardless of requested size (subsample for 2x2, 4x4)
  let scale = 8u / params.bayerSize;
  let mx = bx * scale;
  let my = by * scale;
  let threshold = f32(bayer8x8[my * 8u + mx]) / 64.0;

  // Multi-level quantization with dither offset
  let spread = 1.0 / f32(params.levels - 1u);
  let dithered = lum + (threshold - 0.5) * spread;
  let quantized = clamp(round(dithered * f32(params.levels - 1u)) / f32(params.levels - 1u), 0.0, 1.0);

  // Pack as 8-bit value
  let idx = y * params.width + x;
  output[idx] = u32(quantized * 255.0);
}
`

// ─── Pipeline Creation ───

interface DitherPipeline {
  device: GPUDevice
  pipeline: GPUComputePipeline
  bindGroupLayout: GPUBindGroupLayout
  paramsBuffer: GPUBuffer
  outputBuffer: GPUBuffer
  readbackBuffer: GPUBuffer
  width: number
  height: number
}

async function createDitherPipeline(
  device: GPUDevice,
  width: number,
  height: number
): Promise<DitherPipeline> {
  const shaderModule = device.createShaderModule({ code: BAYER_DITHER_WGSL })

  const bindGroupLayout = device.createBindGroupLayout({
    entries: [
      { binding: 0, visibility: GPUShaderStage.COMPUTE, buffer: { type: 'uniform' } },
      { binding: 1, visibility: GPUShaderStage.COMPUTE, texture: { sampleType: 'float' } },
      { binding: 2, visibility: GPUShaderStage.COMPUTE, buffer: { type: 'storage' } },
    ],
  })

  const pipeline = device.createComputePipeline({
    layout: device.createPipelineLayout({ bindGroupLayouts: [bindGroupLayout] }),
    compute: { module: shaderModule, entryPoint: 'main' },
  })

  const paramsBuffer = device.createBuffer({
    size: 16, // 4 x u32
    usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST,
  })

  const outputSize = width * height * 4 // u32 per pixel
  const outputBuffer = device.createBuffer({
    size: outputSize,
    usage: GPUBufferUsage.STORAGE | GPUBufferUsage.COPY_SRC,
  })

  const readbackBuffer = device.createBuffer({
    size: outputSize,
    usage: GPUBufferUsage.MAP_READ | GPUBufferUsage.COPY_DST,
  })

  return { device, pipeline, bindGroupLayout, paramsBuffer, outputBuffer, readbackBuffer, width, height }
}

// ─── Execute Dithering ───

async function executeDither(
  dp: DitherPipeline,
  sourceImage: ImageBitmap,
  levels: number = 2,
  bayerSize: number = 8
): Promise<Uint32Array> {
  const { device, pipeline, bindGroupLayout, paramsBuffer, outputBuffer, readbackBuffer, width, height } = dp

  // Upload params
  device.queue.writeBuffer(paramsBuffer, 0, new Uint32Array([width, height, levels, bayerSize]))

  // Create texture from source image
  const texture = device.createTexture({
    size: [width, height],
    format: 'rgba8unorm',
    usage: GPUTextureUsage.TEXTURE_BINDING | GPUTextureUsage.COPY_DST | GPUTextureUsage.RENDER_ATTACHMENT,
  })
  device.queue.copyExternalImageToTexture(
    { source: sourceImage },
    { texture },
    [width, height]
  )

  const bindGroup = device.createBindGroup({
    layout: bindGroupLayout,
    entries: [
      { binding: 0, resource: { buffer: paramsBuffer } },
      { binding: 1, resource: texture.createView() },
      { binding: 2, resource: { buffer: outputBuffer } },
    ],
  })

  const encoder = device.createCommandEncoder()
  const pass = encoder.beginComputePass()
  pass.setPipeline(pipeline)
  pass.setBindGroup(0, bindGroup)
  pass.dispatchWorkgroups(Math.ceil(width / 16), Math.ceil(height / 16))
  pass.end()

  encoder.copyBufferToBuffer(outputBuffer, 0, readbackBuffer, 0, readbackBuffer.size)
  device.queue.submit([encoder.finish()])

  await readbackBuffer.mapAsync(GPUMapMode.READ)
  const result = new Uint32Array(readbackBuffer.getMappedRange().slice(0))
  readbackBuffer.unmap()

  texture.destroy()

  return result
}
```

---

## 4. OffscreenCanvas + Web Workers

Move the entire render loop off the main thread. The main thread only handles input events
and DOM updates; the worker owns the canvas and renders at full speed.

### Worker Message Protocol

```typescript
// ─── shared types (e.g., worker-protocol.ts) ───

type WorkerMessage =
  | { type: 'init'; canvas: OffscreenCanvas; width: number; height: number }
  | { type: 'resize'; width: number; height: number }
  | { type: 'updateSource'; bitmap: ImageBitmap }
  | { type: 'setConfig'; config: Partial<RenderConfig> }
  | { type: 'stop' }

type MainMessage =
  | { type: 'ready' }
  | { type: 'fps'; fps: number }
  | { type: 'error'; message: string }

interface RenderConfig {
  cellWidth: number
  cellHeight: number
  ditherMethod: 'bayer' | 'floyd-steinberg' | 'atkinson' | 'none'
  rampChars: string
  fgColor: string
  bgColor: string
  postFx: 'none' | 'crt' | 'bloom'
}
```

### Main Thread Setup

```typescript
function startWorkerRenderer(
  container: HTMLElement,
  width: number,
  height: number
): {
  updateSource: (img: ImageBitmap) => void
  setConfig: (config: Partial<RenderConfig>) => void
  destroy: () => void
} {
  const canvas = document.createElement('canvas')
  canvas.width = width
  canvas.height = height
  canvas.style.width = `${width}px`
  canvas.style.height = `${height}px`
  container.appendChild(canvas)

  const offscreen = canvas.transferControlToOffscreen()
  const worker = new Worker(new URL('./ascii-worker.ts', import.meta.url), { type: 'module' })

  // Transfer the OffscreenCanvas — it becomes unusable on the main thread
  worker.postMessage(
    { type: 'init', canvas: offscreen, width, height } satisfies WorkerMessage,
    [offscreen]
  )

  worker.addEventListener('message', (e: MessageEvent<MainMessage>) => {
    switch (e.data.type) {
      case 'ready':
        break
      case 'fps':
        // Surface FPS to UI if desired
        break
      case 'error':
        console.error(`Worker error: ${e.data.message}`)
        break
    }
  })

  // Resize observer
  const observer = new ResizeObserver((entries) => {
    const { width: w, height: h } = entries[0].contentRect
    worker.postMessage({ type: 'resize', width: Math.floor(w), height: Math.floor(h) } satisfies WorkerMessage)
  })
  observer.observe(canvas)

  return {
    updateSource(bitmap: ImageBitmap): void {
      // ImageBitmap is transferable — zero-copy handoff to worker
      worker.postMessage({ type: 'updateSource', bitmap } satisfies WorkerMessage, [bitmap])
    },
    setConfig(config: Partial<RenderConfig>): void {
      worker.postMessage({ type: 'setConfig', config } satisfies WorkerMessage)
    },
    destroy(): void {
      observer.disconnect()
      worker.postMessage({ type: 'stop' } satisfies WorkerMessage)
      worker.terminate()
      canvas.remove()
    },
  }
}

// Creating an ImageBitmap from video/image for transfer:
async function captureFrame(video: HTMLVideoElement): Promise<ImageBitmap> {
  return createImageBitmap(video, { resizeQuality: 'low' })
}
```

### Worker (ascii-worker.ts)

```typescript
/// <reference lib="webworker" />

// Re-use the module-scope LUTs from Section 1 here — they work identically in workers
const SRGB_TO_LINEAR = new Float32Array(256)
for (let i = 0; i < 256; i++) {
  const s = i / 255
  SRGB_TO_LINEAR[i] = s <= 0.04045 ? s / 12.92 : ((s + 0.055) / 1.055) ** 2.4
}

const ASCII_RAMP = ' .:-=+*#%@'
const RAMP_LEN = ASCII_RAMP.length

let canvas: OffscreenCanvas | null = null
let ctx: OffscreenCanvasRenderingContext2D | null = null
let running = false
let sourceBitmap: ImageBitmap | null = null
let config: RenderConfig = {
  cellWidth: 8,
  cellHeight: 14,
  ditherMethod: 'bayer',
  rampChars: ASCII_RAMP,
  fgColor: '#33ff33',
  bgColor: '#0a0a0a',
  postFx: 'none',
}

// Pre-allocated buffers — resized on dimension change
let gridData: Float32Array = new Float32Array(0)
let pixelBuf: Uint8ClampedArray = new Uint8ClampedArray(0)
let samplerCanvas: OffscreenCanvas | null = null
let samplerCtx: OffscreenCanvasRenderingContext2D | null = null

let frameCount = 0
let fpsTimestamp = 0

function resize(width: number, height: number): void {
  if (!canvas) return
  canvas.width = width
  canvas.height = height
  ctx = canvas.getContext('2d')!

  // Sampler canvas for downscaling source to grid resolution
  const cols = Math.floor(width / config.cellWidth)
  const rows = Math.floor(height / config.cellHeight)
  samplerCanvas = new OffscreenCanvas(cols, rows)
  samplerCtx = samplerCanvas.getContext('2d', { willReadFrequently: true })!
  gridData = new Float32Array(cols * rows)
}

function renderFrame(): void {
  if (!ctx || !canvas || !samplerCtx || !samplerCanvas) return

  const width = canvas.width
  const height = canvas.height
  const { cellWidth, cellHeight } = config
  const cols = Math.floor(width / cellWidth)
  const rows = Math.floor(height / cellHeight)

  if (cols <= 0 || rows <= 0) return

  // Downsample source to grid resolution
  if (sourceBitmap) {
    samplerCtx.drawImage(sourceBitmap, 0, 0, cols, rows)
  }
  const imageData = samplerCtx.getImageData(0, 0, cols, rows)
  const px = imageData.data

  // Compute luminance grid
  for (let i = 0; i < cols * rows; i++) {
    const off = i * 4
    gridData[i] =
      0.2126 * SRGB_TO_LINEAR[px[off]] +
      0.7152 * SRGB_TO_LINEAR[px[off + 1]] +
      0.0722 * SRGB_TO_LINEAR[px[off + 2]]
  }

  // Render
  ctx.fillStyle = config.bgColor
  ctx.fillRect(0, 0, width, height)
  ctx.fillStyle = config.fgColor
  ctx.font = `${cellHeight}px monospace`
  ctx.textBaseline = 'top'

  const ramp = config.rampChars
  const rampLen = ramp.length

  for (let r = 0; r < rows; r++) {
    for (let c = 0; c < cols; c++) {
      const lum = gridData[r * cols + c]
      const ci = Math.min(Math.floor(lum * rampLen), rampLen - 1)
      ctx.fillText(ramp[ci], c * cellWidth, r * cellHeight)
    }
  }

  // FPS reporting
  frameCount++
  const now = performance.now()
  if (now - fpsTimestamp >= 1000) {
    self.postMessage({ type: 'fps', fps: frameCount } satisfies MainMessage)
    frameCount = 0
    fpsTimestamp = now
  }
}

function loop(): void {
  if (!running) return
  renderFrame()
  requestAnimationFrame(loop)
}

self.addEventListener('message', (e: MessageEvent<WorkerMessage>) => {
  switch (e.data.type) {
    case 'init':
      canvas = e.data.canvas
      resize(e.data.width, e.data.height)
      running = true
      fpsTimestamp = performance.now()
      self.postMessage({ type: 'ready' } satisfies MainMessage)
      requestAnimationFrame(loop)
      break

    case 'resize':
      resize(e.data.width, e.data.height)
      break

    case 'updateSource':
      sourceBitmap?.close()
      sourceBitmap = e.data.bitmap
      break

    case 'setConfig':
      Object.assign(config, e.data.config)
      break

    case 'stop':
      running = false
      sourceBitmap?.close()
      sourceBitmap = null
      break
  }
})
```

---

## 5. Frame Budget Breakdown

Time budgets per pipeline stage. Total must fit within the frame deadline.

### 30 FPS (33.3ms budget)

| Stage              | Budget  | Notes                                          |
| ------------------ | ------- | ---------------------------------------------- |
| Pixel sampling     | 1-3ms   | `drawImage` downsample + `getImageData`        |
| Luminance compute  | 0.5-2ms | LUT-based sRGB→linear→BT.709, flat array      |
| Dithering          | 1-4ms   | Bayer: 1ms, Floyd-Steinberg: 2-4ms (scanline)  |
| Character render   | 8-15ms  | `fillText` per cell — the bottleneck           |
| Post-FX (Canvas)   | 2-5ms   | getImageData + pixel manipulation              |
| Post-FX (WebGL)    | 0.5-2ms | GPU shader pass — nearly free                  |
| Compositing/flip   | 0.5-1ms | Browser compositing                            |
| **Headroom**       | **5-20ms** | GC, browser tasks, input handling           |

### 60 FPS (16.67ms budget)

| Stage              | Budget   | Notes                                         |
| ------------------ | -------- | --------------------------------------------- |
| Pixel sampling     | 0.5-1ms  | Must use pre-downscaled source                |
| Luminance compute  | 0.3-1ms  | LUT mandatory, no per-pixel sqrt/pow          |
| Dithering          | 0.5-2ms  | Ordered only — error diffusion too slow       |
| Character render   | 4-8ms    | Cap at ~10k cells, consider bitmap font atlas |
| Post-FX (WebGL)    | 0.3-1ms  | Single draw call                              |
| Compositing/flip   | 0.3-0.5ms| Browser compositing                           |
| **Headroom**       | **3-8ms** | Tight — consider 30fps or worker offload     |

### Bitmap Font Atlas (Eliminating fillText)

`fillText` is the single largest bottleneck. Pre-render all ASCII ramp characters into a
texture atlas, then use `drawImage` to blit character tiles. 3-5x faster.

```typescript
interface FontAtlas {
  canvas: OffscreenCanvas | HTMLCanvasElement
  charWidth: number
  charHeight: number
  chars: string
}

function buildFontAtlas(
  chars: string,
  font: string,
  cellWidth: number,
  cellHeight: number,
  color: string
): FontAtlas {
  const atlasCanvas = new OffscreenCanvas(chars.length * cellWidth, cellHeight)
  const ctx = atlasCanvas.getContext('2d')!

  ctx.font = font
  ctx.fillStyle = color
  ctx.textBaseline = 'top'

  for (let i = 0; i < chars.length; i++) {
    ctx.fillText(chars[i], i * cellWidth, 0)
  }

  return { canvas: atlasCanvas, charWidth: cellWidth, charHeight: cellHeight, chars }
}

function renderWithAtlas(
  ctx: CanvasRenderingContext2D | OffscreenCanvasRenderingContext2D,
  atlas: FontAtlas,
  grid: Float32Array,
  cols: number,
  rows: number
): void {
  const { canvas: atlasCanvas, charWidth, charHeight, chars } = atlas
  const rampLen = chars.length

  for (let r = 0; r < rows; r++) {
    for (let c = 0; c < cols; c++) {
      const lum = grid[r * cols + c]
      const ci = Math.min(Math.floor(lum * rampLen), rampLen - 1)
      ctx.drawImage(
        atlasCanvas,
        ci * charWidth, 0, charWidth, charHeight,
        c * charWidth, r * charHeight, charWidth, charHeight
      )
    }
  }
}
```

---

## 6. Memory Management

### Object Pooling for Float32Arrays

Avoid allocating new typed arrays every frame. Pool and reuse them.

```typescript
class TypedArrayPool<T extends Float32Array | Uint8Array | Uint32Array> {
  private pool: Map<number, T[]> = new Map()
  private ctor: new (length: number) => T

  constructor(ctor: new (length: number) => T) {
    this.ctor = ctor
  }

  acquire(length: number): T {
    const bucket = this.pool.get(length)
    if (bucket && bucket.length > 0) {
      return bucket.pop()!
    }
    return new this.ctor(length)
  }

  release(arr: T): void {
    const len = arr.length
    let bucket = this.pool.get(len)
    if (!bucket) {
      bucket = []
      this.pool.set(len, bucket)
    }
    // Cap pool size to avoid unbounded growth
    if (bucket.length < 8) {
      arr.fill(0 as never)
      bucket.push(arr)
    }
  }

  clear(): void {
    this.pool.clear()
  }
}

// Module-scope pools
const float32Pool = new TypedArrayPool(Float32Array)
const uint8Pool = new TypedArrayPool(Uint8Array)
const uint32Pool = new TypedArrayPool(Uint32Array)
```

### Pre-Allocated Grid Registry

For a renderer with a known maximum size, pre-allocate all grids at init time.

```typescript
interface PreallocatedBuffers {
  luminance: Float32Array
  dithered: Float32Array
  charIndices: Uint8Array
  maxCols: number
  maxRows: number
}

function preallocateBuffers(maxCols: number, maxRows: number): PreallocatedBuffers {
  const size = maxCols * maxRows
  return {
    luminance: new Float32Array(size),
    dithered: new Float32Array(size),
    charIndices: new Uint8Array(size),
    maxCols,
    maxRows,
  }
}

// Usage: pass actual cols/rows each frame, use subarray views
function getSubgrid(buf: Float32Array, actualCols: number, actualRows: number): Float32Array {
  return buf.subarray(0, actualCols * actualRows)
}
```

### GC Pressure Indicators

Watch for these patterns that create garbage every frame:

| Pattern | Problem | Fix |
| ------- | ------- | --- |
| `grid[r][c]` nested arrays | Array-of-arrays, many GC objects | Flat `Float32Array` |
| `str += char` in loop | Intermediate string per iteration | `Uint16Array` + batch `fromCharCode` |
| `{ r, g, b }` per pixel | Object allocation per pixel | Pass components directly or use typed arrays |
| `array.map(...)` | New array allocation | For-loop with pre-allocated output |
| `new ImageData(...)` | 4-byte-per-pixel array each call | Reuse via `ctx.createImageData()` once |
| Closures in hot path | Captures create new function objects | Extract to module scope |

---

## 7. Resolution Scaling

Adaptive quality: if frame time exceeds the target, reduce grid resolution. When headroom
returns, increase resolution back up. Prevents dropped frames on weaker hardware.

```typescript
interface AdaptiveQuality {
  currentScale: number
  minScale: number
  maxScale: number
  targetFrameTime: number
  hysteresisUp: number
  hysteresisDown: number
  consecutiveSlowFrames: number
  consecutiveFastFrames: number
  scaleStep: number
}

function createAdaptiveQuality(targetFps: number = 30): AdaptiveQuality {
  return {
    currentScale: 1.0,
    minScale: 0.25,
    maxScale: 1.0,
    targetFrameTime: 1000 / targetFps,
    hysteresisUp: 10,    // consecutive fast frames before scaling up
    hysteresisDown: 3,   // consecutive slow frames before scaling down
    consecutiveSlowFrames: 0,
    consecutiveFastFrames: 0,
    scaleStep: 0.1,
  }
}

function updateAdaptiveQuality(aq: AdaptiveQuality, frameTimeMs: number): number {
  const overBudget = frameTimeMs > aq.targetFrameTime * 1.1
  const underBudget = frameTimeMs < aq.targetFrameTime * 0.7

  if (overBudget) {
    aq.consecutiveSlowFrames++
    aq.consecutiveFastFrames = 0

    if (aq.consecutiveSlowFrames >= aq.hysteresisDown) {
      aq.currentScale = Math.max(aq.minScale, aq.currentScale - aq.scaleStep)
      aq.consecutiveSlowFrames = 0
    }
  } else if (underBudget) {
    aq.consecutiveFastFrames++
    aq.consecutiveSlowFrames = 0

    if (aq.consecutiveFastFrames >= aq.hysteresisUp) {
      aq.currentScale = Math.min(aq.maxScale, aq.currentScale + aq.scaleStep)
      aq.consecutiveFastFrames = 0
    }
  } else {
    aq.consecutiveSlowFrames = 0
    aq.consecutiveFastFrames = 0
  }

  return aq.currentScale
}

// Apply to grid dimensions
function scaledGridSize(
  baseCols: number,
  baseRows: number,
  scale: number
): { cols: number; rows: number } {
  return {
    cols: Math.max(4, Math.round(baseCols * scale)),
    rows: Math.max(2, Math.round(baseRows * scale)),
  }
}

// Integration in render loop:
function adaptiveRenderLoop(
  render: (cols: number, rows: number, dt: number) => void,
  baseCols: number,
  baseRows: number,
  targetFps: number = 30
): () => void {
  const aq = createAdaptiveQuality(targetFps)
  let running = true
  let lastTime = performance.now()

  function tick(now: number): void {
    if (!running) return

    const dt = now - lastTime
    lastTime = now

    const scale = updateAdaptiveQuality(aq, dt)
    const { cols, rows } = scaledGridSize(baseCols, baseRows, scale)

    render(cols, rows, dt)

    requestAnimationFrame(tick)
  }

  requestAnimationFrame(tick)
  return () => { running = false }
}
```

---

## 8. Shader Language Comparison

### GLSL vs WGSL vs HLSL for Image Processing

| Feature | GLSL (WebGL2) | WGSL (WebGPU) | HLSL (DirectX / ref) |
| ------- | ------------- | ------------- | -------------------- |
| **Platform** | All browsers | Chrome 113+, Safari 18+, Firefox Nightly | Windows/Xbox native; not web |
| **Shader types** | Vertex, Fragment | Vertex, Fragment, Compute | All (Vertex, Pixel, Compute, Hull, Domain, Geometry, Mesh, Amplification) |
| **Compute shaders** | No (WebGL2 lacks compute) | Yes — primary advantage | Yes |
| **Texture sampling** | `texture(sampler2D, uv)` | `textureSample(tex, samp, uv)` | `tex.Sample(samp, uv)` |
| **Texture load (no filter)** | `texelFetch(sampler2D, ivec2, lod)` | `textureLoad(tex, vec2<i32>, lod)` | `tex.Load(int3(x,y,0))` |
| **Storage buffers** | Not available | `var<storage, read_write>` | `RWStructuredBuffer<T>` |
| **Entry point** | `void main()` | `@compute @workgroup_size(X,Y) fn main(...)` | `[numthreads(X,Y,1)] void CSMain(...)` |
| **Type syntax** | `vec3`, `mat4`, `float` | `vec3<f32>`, `mat4x4<f32>`, `f32` | `float3`, `float4x4`, `float` |
| **Swizzle** | `v.xyz`, `v.rgb` | `v.xyz` (no `.rgb`) | `v.xyz`, `v.rgb` |
| **Integer types** | `int`, `uint` (32-bit) | `i32`, `u32` | `int`, `uint` |
| **Branching cost** | Moderate (fragment divergence) | Low in compute (workgroup uniform) | Low in compute |
| **Max texture size** | 4096-16384 (device) | 8192-16384 | 16384+ |

### When to Use Each

| Scenario | Recommendation | Why |
| -------- | -------------- | --- |
| Post-processing only (CRT, bloom, color grading) | **GLSL / WebGL2** | Maximum browser support; fragment shaders sufficient |
| GPU-side dithering / luminance computation | **WGSL / WebGPU** | Compute shaders avoid the render pipeline overhead; direct buffer readback |
| Real-time video to ASCII (full pipeline on GPU) | **WGSL / WebGPU** | Upload video frame as texture, compute luminance + dither + char lookup in one dispatch |
| Maximum compatibility needed | **GLSL / WebGL2** | Works everywhere including mobile Safari, older Android |
| Offline / batch processing (server-side) | **Neither** — use CPU (SIMD) or CUDA | WebGL/WebGPU require a browser context |
| Hybrid: wide compat + perf where available | **GLSL with WebGPU fallforward** | Start with WebGL2, progressively enhance to WebGPU compute |

### Syntax Quick Reference

Dithering threshold check in all three languages:

**GLSL (Fragment)**
```glsl
float bayerThreshold = texelFetch(u_bayerTex, ivec2(gl_FragCoord.xy) % 8, 0).r;
float lum = dot(color.rgb, vec3(0.2126, 0.7152, 0.0722));
float dithered = step(bayerThreshold, lum);
```

**WGSL (Compute)**
```wgsl
let bayer_val = textureLoad(bayer_tex, vec2<i32>(gid.xy % vec2<u32>(8u)), 0).r;
let lum = dot(color.rgb, vec3<f32>(0.2126, 0.7152, 0.0722));
let dithered = select(0.0, 1.0, lum > bayer_val);
```

**HLSL (Compute, reference only)**
```hlsl
float bayerVal = BayerTex.Load(int3(gid.xy % 8, 0)).r;
float lum = dot(color.rgb, float3(0.2126, 0.7152, 0.0722));
float dithered = step(bayerVal, lum);
```
