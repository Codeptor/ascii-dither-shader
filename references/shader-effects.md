# Shader Effects Reference

Post-processing effects for ASCII/dither visuals. Apply after character rendering for authentic
retro, cinematic, or artistic aesthetics. Includes both Canvas 2D and GLSL/WGSL implementations.

---

## 1. CRT Screen Effect

Full CRT simulation: barrel distortion, scanlines, RGB phosphor sub-pixels, and bloom glow.

### Barrel Distortion

```typescript
function barrelDistort(
  x: number, y: number, w: number, h: number, strength: number = 0.15
): [number, number] {
  // Normalize to -1..1
  const nx = (x / w) * 2 - 1
  const ny = (y / h) * 2 - 1
  const r2 = nx * nx + ny * ny

  // Barrel distortion formula: r' = r × (1 + k × r²)
  const distortion = 1 + strength * r2
  const dx = nx * distortion
  const dy = ny * distortion

  return [(dx + 1) * 0.5 * w, (dy + 1) * 0.5 * h]
}
```

### GLSL CRT Shader (Complete)

```glsl
uniform sampler2D u_texture;
uniform vec2 u_resolution;
uniform float u_time;
uniform float u_curvature;       // 0.0 - 0.3, default 0.15
uniform float u_scanlineIntensity; // 0.0 - 1.0, default 0.3
uniform float u_bloomStrength;    // 0.0 - 1.0, default 0.15
uniform float u_vignetteStrength; // 0.0 - 1.0, default 0.4

vec2 crtDistort(vec2 uv) {
  vec2 centered = uv * 2.0 - 1.0;
  float r2 = dot(centered, centered);
  centered *= 1.0 + u_curvature * r2;
  return centered * 0.5 + 0.5;
}

vec3 scanlines(vec2 uv, vec3 color) {
  float scanline = sin(uv.y * u_resolution.y * 3.14159) * 0.5 + 0.5;
  scanline = pow(scanline, 1.5) * u_scanlineIntensity;
  return color * (1.0 - scanline);
}

vec3 phosphorMask(vec2 pos, vec3 color) {
  // RGB sub-pixel pattern (3-pixel repeat)
  int subpixel = int(mod(pos.x, 3.0));
  vec3 mask = vec3(0.7);
  if (subpixel == 0) mask.r = 1.0;
  else if (subpixel == 1) mask.g = 1.0;
  else mask.b = 1.0;
  return color * mix(vec3(1.0), mask, 0.3);
}

vec3 bloom(sampler2D tex, vec2 uv, float strength) {
  vec3 sum = vec3(0.0);
  float weights[5] = float[](0.227, 0.194, 0.122, 0.054, 0.016);
  vec2 texel = 1.0 / u_resolution;

  for (int i = -4; i <= 4; i++) {
    for (int j = -4; j <= 4; j++) {
      vec2 offset = vec2(float(i), float(j)) * texel * 2.0;
      vec3 sample_color = texture(tex, uv + offset).rgb;
      // Only bloom bright pixels
      float bright = max(sample_color.r, max(sample_color.g, sample_color.b));
      if (bright > 0.5) {
        float w = weights[abs(i)] * weights[abs(j)];
        sum += sample_color * w * (bright - 0.5) * 2.0;
      }
    }
  }
  return sum * strength;
}

float vignette(vec2 uv, float strength) {
  vec2 centered = uv * 2.0 - 1.0;
  float dist = length(centered);
  return 1.0 - smoothstep(0.5, 1.5, dist) * strength;
}

void main() {
  vec2 uv = gl_FragCoord.xy / u_resolution;
  vec2 distortedUV = crtDistort(uv);

  // Reject pixels outside the screen
  if (distortedUV.x < 0.0 || distortedUV.x > 1.0 ||
      distortedUV.y < 0.0 || distortedUV.y > 1.0) {
    gl_FragColor = vec4(0.0, 0.0, 0.0, 1.0);
    return;
  }

  vec3 color = texture(u_texture, distortedUV).rgb;

  // Apply effects in order
  color = phosphorMask(gl_FragCoord.xy, color);
  color = scanlines(distortedUV, color);
  color += bloom(u_texture, distortedUV, u_bloomStrength);
  color *= vignette(distortedUV, u_vignetteStrength);

  // Subtle flicker
  color *= 0.98 + 0.02 * sin(u_time * 8.0);

  gl_FragColor = vec4(color, 1.0);
}
```

### WGSL CRT Shader

```wgsl
struct Uniforms {
  resolution: vec2<f32>,
  time: f32,
  curvature: f32,
  scanline_intensity: f32,
  bloom_strength: f32,
}

@group(0) @binding(0) var<uniform> u: Uniforms;
@group(0) @binding(1) var tex: texture_2d<f32>;
@group(0) @binding(2) var tex_sampler: sampler;

fn crt_distort(uv: vec2<f32>) -> vec2<f32> {
  let centered = uv * 2.0 - 1.0;
  let r2 = dot(centered, centered);
  let distorted = centered * (1.0 + u.curvature * r2);
  return distorted * 0.5 + 0.5;
}

fn scanlines(uv: vec2<f32>, color: vec3<f32>) -> vec3<f32> {
  let scan = sin(uv.y * u.resolution.y * 3.14159) * 0.5 + 0.5;
  let intensity = pow(scan, 1.5) * u.scanline_intensity;
  return color * (1.0 - intensity);
}

@fragment
fn fs_main(@builtin(position) pos: vec4<f32>) -> @location(0) vec4<f32> {
  let uv = pos.xy / u.resolution;
  let duv = crt_distort(uv);

  if (duv.x < 0.0 || duv.x > 1.0 || duv.y < 0.0 || duv.y > 1.0) {
    return vec4<f32>(0.0, 0.0, 0.0, 1.0);
  }

  var color = textureSample(tex, tex_sampler, duv).rgb;
  color = scanlines(duv, color);
  color = color * (0.98 + 0.02 * sin(u.time * 8.0));

  return vec4<f32>(color, 1.0);
}
```

### Canvas 2D CRT (Post-Process)

```typescript
function applyCRT(
  ctx: CanvasRenderingContext2D, w: number, h: number, time: number,
  config: { scanlineOpacity?: number; bloomRadius?: number; vignetteStrength?: number }
): void {
  const { scanlineOpacity = 0.25, bloomRadius = 3, vignetteStrength = 0.35 } = config

  // Scanlines
  ctx.fillStyle = `rgba(0,0,0,${scanlineOpacity})`
  for (let y = 0; y < h; y += 3) {
    ctx.fillRect(0, y, w, 1)
  }

  // Phosphor RGB mask (very subtle)
  const imageData = ctx.getImageData(0, 0, w, h)
  const data = imageData.data
  for (let i = 0; i < data.length; i += 4) {
    const px = (i / 4) % w
    const sub = px % 3
    if (sub === 0) { data[i + 1] *= 0.85; data[i + 2] *= 0.85 }
    else if (sub === 1) { data[i] *= 0.85; data[i + 2] *= 0.85 }
    else { data[i] *= 0.85; data[i + 1] *= 0.85 }
  }
  ctx.putImageData(imageData, 0, 0)

  // Vignette overlay
  const gradient = ctx.createRadialGradient(w / 2, h / 2, w * 0.25, w / 2, h / 2, w * 0.7)
  gradient.addColorStop(0, 'rgba(0,0,0,0)')
  gradient.addColorStop(1, `rgba(0,0,0,${vignetteStrength})`)
  ctx.fillStyle = gradient
  ctx.fillRect(0, 0, w, h)

  // Subtle flicker
  ctx.fillStyle = `rgba(0,0,0,${0.01 * Math.sin(time * 8)})`
  ctx.fillRect(0, 0, w, h)
}
```

---

## 2. Scanlines

```typescript
function applyScanlines(
  ctx: CanvasRenderingContext2D, w: number, h: number,
  gap: number = 3, opacity: number = 0.25, offset: number = 0
): void {
  ctx.fillStyle = `rgba(0,0,0,${opacity})`
  for (let y = offset; y < h; y += gap) {
    ctx.fillRect(0, y, w, 1)
  }
}
```

### GLSL

```glsl
vec3 applyScanlines(vec2 uv, vec3 color, float gap, float opacity) {
  float scanline = step(0.5, fract(uv.y * gap));
  return mix(color, color * (1.0 - opacity), scanline);
}
```

---

## 3. Bloom / Glow

Two-pass Gaussian blur on bright pixels, then screen-blend onto the original.

### Kawase Bloom (Fast, Multi-Pass)

```typescript
function kawaseBloom(
  ctx: CanvasRenderingContext2D, w: number, h: number,
  threshold: number = 0.5, passes: number = 4, strength: number = 0.3
): void {
  // Create a temporary canvas for the bloom
  const bloomCanvas = document.createElement('canvas')
  bloomCanvas.width = w / 2; bloomCanvas.height = h / 2
  const bctx = bloomCanvas.getContext('2d')!

  // Downsample
  bctx.drawImage(ctx.canvas, 0, 0, w, h, 0, 0, w / 2, h / 2)

  // Multi-pass blur
  for (let i = 0; i < passes; i++) {
    bctx.filter = `blur(${2 + i * 2}px)`
    bctx.drawImage(bloomCanvas, 0, 0)
    bctx.filter = 'none'
  }

  // Screen blend onto original
  ctx.globalCompositeOperation = 'screen'
  ctx.globalAlpha = strength
  ctx.drawImage(bloomCanvas, 0, 0, w / 2, h / 2, 0, 0, w, h)
  ctx.globalCompositeOperation = 'source-over'
  ctx.globalAlpha = 1
}
```

### GLSL Bloom

```glsl
// Bright pass: extract pixels above threshold
vec3 brightPass(vec3 color, float threshold) {
  float brightness = max(color.r, max(color.g, color.b));
  return color * smoothstep(threshold, threshold + 0.1, brightness);
}

// Gaussian blur (single direction — call twice for full 2D blur)
vec3 gaussianBlur(sampler2D tex, vec2 uv, vec2 direction) {
  vec3 sum = vec3(0.0);
  float weights[5] = float[](0.227027, 0.194595, 0.121622, 0.054054, 0.016216);
  vec2 texel = 1.0 / vec2(textureSize(tex, 0));
  sum += texture(tex, uv).rgb * weights[0];
  for (int i = 1; i < 5; i++) {
    vec2 off = direction * texel * float(i) * 2.0;
    sum += texture(tex, uv + off).rgb * weights[i];
    sum += texture(tex, uv - off).rgb * weights[i];
  }
  return sum;
}
```

---

## 4. Chromatic Aberration

RGB channel offset simulating lens fringe.

```typescript
function chromaticAberration(
  ctx: CanvasRenderingContext2D, w: number, h: number, amount: number = 2
): void {
  const imageData = ctx.getImageData(0, 0, w, h)
  const data = imageData.data
  const copy = new Uint8ClampedArray(data)

  for (let y = 0; y < h; y++) {
    for (let x = 0; x < w; x++) {
      const i = (y * w + x) * 4
      // Shift red channel left
      const rx = Math.max(0, x - amount)
      const ri = (y * w + rx) * 4
      data[i] = copy[ri]  // R from shifted position

      // Shift blue channel right
      const bx = Math.min(w - 1, x + amount)
      const bi = (y * w + bx) * 4
      data[i + 2] = copy[bi + 2]  // B from shifted position

      // Green stays in place
    }
  }
  ctx.putImageData(imageData, 0, 0)
}
```

### GLSL

```glsl
vec3 chromaticAberration(sampler2D tex, vec2 uv, float amount) {
  vec2 offset = vec2(amount / u_resolution.x, 0.0);
  float r = texture(tex, uv - offset).r;
  float g = texture(tex, uv).g;
  float b = texture(tex, uv + offset).b;
  return vec3(r, g, b);
}
```

---

## 5. Vignette

```typescript
function applyVignette(
  ctx: CanvasRenderingContext2D, w: number, h: number, strength: number = 0.4
): void {
  const gradient = ctx.createRadialGradient(
    w / 2, h / 2, Math.min(w, h) * 0.3,
    w / 2, h / 2, Math.max(w, h) * 0.7
  )
  gradient.addColorStop(0, 'rgba(0,0,0,0)')
  gradient.addColorStop(1, `rgba(0,0,0,${strength})`)
  ctx.fillStyle = gradient
  ctx.fillRect(0, 0, w, h)
}

function vignetteFactor(x: number, y: number, cols: number, rows: number, strength: number): number {
  const nx = (x / cols) * 2 - 1
  const ny = (y / rows) * 2 - 1
  const dist = Math.sqrt(nx * nx + ny * ny) / Math.SQRT2
  return Math.max(0, Math.min(1, 1 - dist * dist * strength))
}
```

### GLSL

```glsl
float vignette(vec2 uv, float strength) {
  vec2 centered = uv * 2.0 - 1.0;
  return 1.0 - smoothstep(0.4, 1.4, length(centered)) * strength;
}
```

---

## 6. Film Grain

```typescript
function applyGrain(
  ctx: CanvasRenderingContext2D, w: number, h: number,
  amount: number = 0.05, time: number = 0
): void {
  const imageData = ctx.getImageData(0, 0, w, h)
  const data = imageData.data
  const seed = (time * 60) | 0

  for (let i = 0; i < data.length; i += 4) {
    const px = (i / 4)
    const noise = ((px * 2654435761 + seed * 340573321) & 0xffffff) / 0xffffff
    const grain = ((noise - 0.5) * 2 * amount * 255) | 0
    data[i] = Math.max(0, Math.min(255, data[i] + grain))
    data[i + 1] = Math.max(0, Math.min(255, data[i + 1] + grain))
    data[i + 2] = Math.max(0, Math.min(255, data[i + 2] + grain))
  }
  ctx.putImageData(imageData, 0, 0)
}
```

### GLSL

```glsl
float filmGrain(vec2 uv, float time, float amount) {
  float noise = fract(sin(dot(uv + fract(time), vec2(12.9898, 78.233))) * 43758.5453);
  return (noise - 0.5) * amount;
}
```

---

## 7. Glitch Effects

### Block Displacement

```typescript
function applyGlitch(
  ctx: CanvasRenderingContext2D, w: number, h: number,
  intensity: number = 0.3, time: number = 0
): void {
  // Random glitch trigger (not every frame)
  const seed = (time * 60) | 0
  const trigger = ((seed * 2654435761) & 0xffff) / 0xffff
  if (trigger > intensity) return  // skip most frames

  const imageData = ctx.getImageData(0, 0, w, h)
  const data = new Uint8ClampedArray(imageData.data)

  // Random horizontal block shift
  const numBlocks = 3 + ((seed * 1103515245) & 7)
  for (let b = 0; b < numBlocks; b++) {
    const blockY = (((seed + b) * 340573321) & 0x7fff) % h
    const blockH = 2 + (((seed + b * 7) * 2654435761) & 0x1f)
    const shift = (((seed + b * 13) * 1103515245) & 0xff) - 128

    for (let y = blockY; y < Math.min(h, blockY + blockH); y++) {
      for (let x = 0; x < w; x++) {
        const srcX = Math.max(0, Math.min(w - 1, x + shift))
        const si = (y * w + srcX) * 4
        const di = (y * w + x) * 4
        imageData.data[di] = data[si]
        imageData.data[di + 1] = data[si + 1]
        imageData.data[di + 2] = data[si + 2]
      }
    }
  }

  // Color channel split on glitch blocks
  for (let y = 0; y < h; y++) {
    if (((y * 2654435761 + seed) & 0xff) > 240) {
      for (let x = 0; x < w; x++) {
        const i = (y * w + x) * 4
        const cx = Math.min(w - 1, x + 3)
        const ci = (y * w + cx) * 4
        imageData.data[i] = data[ci]  // shift red channel
      }
    }
  }

  ctx.putImageData(imageData, 0, 0)
}
```

### GLSL Glitch

```glsl
vec3 glitch(sampler2D tex, vec2 uv, float time, float intensity) {
  // Random block displacement
  float block = floor(uv.y * 20.0 + floor(time * 10.0));
  float noise = fract(sin(block * 43758.5453) * 2.0);

  vec2 offset = vec2(0.0);
  if (noise > 1.0 - intensity * 0.3) {
    offset.x = (noise - 0.5) * 0.1 * intensity;
  }

  vec3 color;
  color.r = texture(tex, uv + offset + vec2(0.003 * intensity, 0.0)).r;
  color.g = texture(tex, uv + offset).g;
  color.b = texture(tex, uv + offset - vec2(0.003 * intensity, 0.0)).b;

  // Scan tear
  float tear = step(0.99 - intensity * 0.05, fract(sin(time * 100.0 + uv.y * 10.0)));
  color = mix(color, vec3(1.0), tear * 0.5);

  return color;
}
```

---

## 8. Interlace

```typescript
function applyInterlace(
  ctx: CanvasRenderingContext2D, w: number, h: number,
  frame: number, opacity: number = 0.15
): void {
  const field = frame & 1  // even/odd field
  ctx.fillStyle = `rgba(0,0,0,${opacity})`
  for (let y = field; y < h; y += 2) {
    ctx.fillRect(0, y, w, 1)
  }
}
```

---

## 9. Noise FX (Simplex 3D)

Animated noise for pre-render brightness modification.

```typescript
// Simplex 3D noise (for animated procedural effects)
// Full implementation — self-contained, no external deps

const GRAD3 = [
  [1,1,0],[-1,1,0],[1,-1,0],[-1,-1,0],
  [1,0,1],[-1,0,1],[1,0,-1],[-1,0,-1],
  [0,1,1],[0,-1,1],[0,1,-1],[0,-1,-1],
]

const PERM = new Uint8Array(512)
const P = [151,160,137,91,90,15,131,13,201,95,96,53,194,233,7,225,140,36,103,30,
  69,142,8,99,37,240,21,10,23,190,6,148,247,120,234,75,0,26,197,62,94,252,219,203,
  117,35,11,32,57,177,33,88,237,149,56,87,174,20,125,136,171,168,68,175,74,165,71,
  134,139,48,27,166,77,146,158,231,83,111,229,122,60,211,133,230,220,105,92,41,55,
  46,245,40,244,102,143,54,65,25,63,161,1,216,80,73,209,76,132,187,208,89,18,169,
  200,196,135,130,116,188,159,86,164,100,109,198,173,186,3,64,52,217,226,250,124,
  123,5,202,38,147,118,126,255,82,85,212,207,206,59,227,47,16,58,17,182,189,28,42,
  223,183,170,213,119,248,152,2,44,154,163,70,221,153,101,155,167,43,172,9,129,22,
  39,253,19,98,108,110,79,113,224,232,178,185,112,104,218,246,97,228,251,34,242,
  193,238,210,144,12,191,179,162,241,81,51,145,235,249,14,239,107,49,192,214,31,
  181,199,106,157,184,84,204,176,115,121,50,45,127,4,150,254,138,236,205,93,222,
  114,67,29,24,72,243,141,128,195,78,66,215,61,156,180]

for (let i = 0; i < 512; i++) PERM[i] = P[i & 255]

function simplex3(x: number, y: number, z: number): number {
  const F3 = 1 / 3, G3 = 1 / 6
  const s = (x + y + z) * F3
  const i = Math.floor(x + s), j = Math.floor(y + s), k = Math.floor(z + s)
  const t = (i + j + k) * G3
  const X0 = i - t, Y0 = j - t, Z0 = k - t
  const x0 = x - X0, y0 = y - Y0, z0 = z - Z0

  let i1: number, j1: number, k1: number, i2: number, j2: number, k2: number
  if (x0 >= y0) {
    if (y0 >= z0) { i1=1;j1=0;k1=0;i2=1;j2=1;k2=0 }
    else if (x0 >= z0) { i1=1;j1=0;k1=0;i2=1;j2=0;k2=1 }
    else { i1=0;j1=0;k1=1;i2=1;j2=0;k2=1 }
  } else {
    if (y0 < z0) { i1=0;j1=0;k1=1;i2=0;j2=1;k2=1 }
    else if (x0 < z0) { i1=0;j1=1;k1=0;i2=0;j2=1;k2=1 }
    else { i1=0;j1=1;k1=0;i2=1;j2=1;k2=0 }
  }

  const x1=x0-i1+G3,y1=y0-j1+G3,z1=z0-k1+G3
  const x2=x0-i2+2*G3,y2=y0-j2+2*G3,z2=z0-k2+2*G3
  const x3=x0-1+3*G3,y3=y0-1+3*G3,z3=z0-1+3*G3

  const ii=i&255,jj=j&255,kk=k&255
  const gi0=PERM[ii+PERM[jj+PERM[kk]]]%12
  const gi1=PERM[ii+i1+PERM[jj+j1+PERM[kk+k1]]]%12
  const gi2=PERM[ii+i2+PERM[jj+j2+PERM[kk+k2]]]%12
  const gi3=PERM[ii+1+PERM[jj+1+PERM[kk+1]]]%12

  let n0=0,n1=0,n2=0,n3=0
  let t0=0.6-x0*x0-y0*y0-z0*z0
  if(t0>0){t0*=t0;n0=t0*t0*(GRAD3[gi0][0]*x0+GRAD3[gi0][1]*y0+GRAD3[gi0][2]*z0)}
  let t1=0.6-x1*x1-y1*y1-z1*z1
  if(t1>0){t1*=t1;n1=t1*t1*(GRAD3[gi1][0]*x1+GRAD3[gi1][1]*y1+GRAD3[gi1][2]*z1)}
  let t2=0.6-x2*x2-y2*y2-z2*z2
  if(t2>0){t2*=t2;n2=t2*t2*(GRAD3[gi2][0]*x2+GRAD3[gi2][1]*y2+GRAD3[gi2][2]*z2)}
  let t3=0.6-x3*x3-y3*y3-z3*z3
  if(t3>0){t3*=t3;n3=t3*t3*(GRAD3[gi3][0]*x3+GRAD3[gi3][1]*y3+GRAD3[gi3][2]*z3)}

  return 32*(n0+n1+n2+n3)  // returns -1..1
}

// Fractional Brownian Motion (layered noise for natural textures)
function fbm(x: number, y: number, z: number, octaves: number = 4): number {
  let value = 0, amplitude = 0.5, frequency = 1
  for (let i = 0; i < octaves; i++) {
    value += amplitude * simplex3(x * frequency, y * frequency, z * frequency)
    amplitude *= 0.5
    frequency *= 2
  }
  return value
}
```

### GLSL Noise (From McEwan et al. 2012, arXiv:1204.1461)

```glsl
// Ashima Arts simplex noise (optimized, no texture lookups)
vec3 mod289(vec3 x) { return x - floor(x * (1.0 / 289.0)) * 289.0; }
vec4 mod289(vec4 x) { return x - floor(x * (1.0 / 289.0)) * 289.0; }
vec4 permute(vec4 x) { return mod289(((x * 34.0) + 1.0) * x); }
vec4 taylorInvSqrt(vec4 r) { return 1.79284291400159 - 0.85373472095314 * r; }

float snoise(vec3 v) {
  const vec2 C = vec2(1.0/6.0, 1.0/3.0);
  const vec4 D = vec4(0.0, 0.5, 1.0, 2.0);

  vec3 i = floor(v + dot(v, C.yyy));
  vec3 x0 = v - i + dot(i, C.xxx);

  vec3 g = step(x0.yzx, x0.xyz);
  vec3 l = 1.0 - g;
  vec3 i1 = min(g.xyz, l.zxy);
  vec3 i2 = max(g.xyz, l.zxy);

  vec3 x1 = x0 - i1 + C.xxx;
  vec3 x2 = x0 - i2 + C.yyy;
  vec3 x3 = x0 - D.yyy;

  i = mod289(i);
  vec4 p = permute(permute(permute(
    i.z + vec4(0.0, i1.z, i2.z, 1.0))
    + i.y + vec4(0.0, i1.y, i2.y, 1.0))
    + i.x + vec4(0.0, i1.x, i2.x, 1.0));

  float n_ = 0.142857142857;
  vec3 ns = n_ * D.wyz - D.xzx;

  vec4 j = p - 49.0 * floor(p * ns.z * ns.z);
  vec4 x_ = floor(j * ns.z);
  vec4 y_ = floor(j - 7.0 * x_);
  vec4 x = x_ * ns.x + ns.yyyy;
  vec4 y = y_ * ns.x + ns.yyyy;
  vec4 h = 1.0 - abs(x) - abs(y);

  vec4 b0 = vec4(x.xy, y.xy);
  vec4 b1 = vec4(x.zw, y.zw);
  vec4 s0 = floor(b0) * 2.0 + 1.0;
  vec4 s1 = floor(b1) * 2.0 + 1.0;
  vec4 sh = -step(h, vec4(0.0));
  vec4 a0 = b0.xzyw + s0.xzyw * sh.xxyy;
  vec4 a1 = b1.xzyw + s1.xzyw * sh.zzww;

  vec3 p0 = vec3(a0.xy, h.x);
  vec3 p1 = vec3(a0.zw, h.y);
  vec3 p2 = vec3(a1.xy, h.z);
  vec3 p3 = vec3(a1.zw, h.w);

  vec4 norm = taylorInvSqrt(vec4(dot(p0,p0), dot(p1,p1), dot(p2,p2), dot(p3,p3)));
  p0 *= norm.x; p1 *= norm.y; p2 *= norm.z; p3 *= norm.w;

  vec4 m = max(0.6 - vec4(dot(x0,x0), dot(x1,x1), dot(x2,x2), dot(x3,x3)), 0.0);
  m = m * m;
  return 42.0 * dot(m * m, vec4(dot(p0,x0), dot(p1,x1), dot(p2,x2), dot(p3,x3)));
}
```

---

## 10. Feedback Buffer (Echo/Trail)

Creates motion trails and echo effects by blending previous frames.

```typescript
class FeedbackBuffer {
  private buffer: ImageData | null = null
  private decay: number

  constructor(decay: number = 0.85) {
    this.decay = decay
  }

  apply(ctx: CanvasRenderingContext2D, w: number, h: number): void {
    const current = ctx.getImageData(0, 0, w, h)

    if (this.buffer && this.buffer.width === w && this.buffer.height === h) {
      const cur = current.data
      const prev = this.buffer.data
      for (let i = 0; i < cur.length; i += 4) {
        // Screen blend with decay
        cur[i]     = Math.max(cur[i],     prev[i] * this.decay | 0)
        cur[i + 1] = Math.max(cur[i + 1], prev[i + 1] * this.decay | 0)
        cur[i + 2] = Math.max(cur[i + 2], prev[i + 2] * this.decay | 0)
      }
      ctx.putImageData(current, 0, 0)
    }

    this.buffer = ctx.getImageData(0, 0, w, h)
  }
}
```

---

## 11. Edge Detection (Sobel)

For edge-aware character selection and outline effects.

```typescript
function sobelEdge(
  grid: Float32Array, x: number, y: number, cols: number, rows: number
): number {
  const idx = y * cols + x
  const left  = x > 0 ? grid[idx - 1] : grid[idx]
  const right = x < cols - 1 ? grid[idx + 1] : grid[idx]
  const up    = y > 0 ? grid[idx - cols] : grid[idx]
  const down  = y < rows - 1 ? grid[idx + cols] : grid[idx]
  const gx = Math.abs(right - left)
  const gy = Math.abs(down - up)
  return Math.min(1, Math.sqrt(gx * gx + gy * gy))
}

// Full 3x3 Sobel with diagonal neighbors
function sobelFull(
  grid: Float32Array, x: number, y: number, cols: number, rows: number
): [number, number] {
  const g = (dx: number, dy: number) => {
    const nx = Math.max(0, Math.min(cols - 1, x + dx))
    const ny = Math.max(0, Math.min(rows - 1, y + dy))
    return grid[ny * cols + nx]
  }
  const gx = -g(-1,-1) + g(1,-1) - 2*g(-1,0) + 2*g(1,0) - g(-1,1) + g(1,1)
  const gy = -g(-1,-1) - 2*g(0,-1) - g(1,-1) + g(-1,1) + 2*g(0,1) + g(1,1)
  return [gx, gy]  // gradient direction for edge-oriented character selection
}
```

### GLSL

```glsl
float sobelEdge(sampler2D tex, vec2 uv) {
  vec2 texel = 1.0 / vec2(textureSize(tex, 0));
  float tl = luminance(texture(tex, uv + vec2(-texel.x, -texel.y)).rgb);
  float t  = luminance(texture(tex, uv + vec2(0, -texel.y)).rgb);
  float tr = luminance(texture(tex, uv + vec2(texel.x, -texel.y)).rgb);
  float l  = luminance(texture(tex, uv + vec2(-texel.x, 0)).rgb);
  float r  = luminance(texture(tex, uv + vec2(texel.x, 0)).rgb);
  float bl = luminance(texture(tex, uv + vec2(-texel.x, texel.y)).rgb);
  float b  = luminance(texture(tex, uv + vec2(0, texel.y)).rgb);
  float br = luminance(texture(tex, uv + vec2(texel.x, texel.y)).rgb);

  float gx = -tl + tr - 2.0*l + 2.0*r - bl + br;
  float gy = -tl - 2.0*t - tr + bl + 2.0*b + br;
  return sqrt(gx*gx + gy*gy);
}
```
