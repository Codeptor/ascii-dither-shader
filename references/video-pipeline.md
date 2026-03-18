# Video Processing Pipeline

End-to-end pipeline for rendering video to ASCII/dither output. Covers frame extraction,
temporal coherence (the hardest problem), adaptive tonemapping, vectorized rendering,
ffmpeg encoding, and real-time web playback.

Research basis:
- Wolfe et al. (2021): Spatiotemporal blue noise masks [arXiv:2112.09629]
- Georgiev & Fajardo (2016): Blue-noise dithered sampling (Pixar)
- Heitz & Belcour (2019): Distributing Monte Carlo errors as blue noise in screen space
- Ulichney (1993): Void-and-cluster for blue noise generation
- Horn & Schunck (1981): Optical flow for motion compensation

---

## 1. Frame Extraction

### Web: HTMLVideoElement + Canvas Sampling

The browser path uses a hidden `<video>` element as a pixel source, sampled onto an
offscreen canvas at the target grid resolution. `requestVideoFrameCallback` provides
frame-accurate timing — it fires once per decoded video frame rather than at display
refresh rate, avoiding duplicate processing.

```typescript
interface FrameExtractorConfig {
  src: string;
  gridWidth: number;
  gridHeight: number;
  onFrame: (pixels: Uint8ClampedArray, timestamp: number) => void;
  onComplete: () => void;
}

function createFrameExtractor(config: FrameExtractorConfig): {
  start: () => void;
  stop: () => void;
} {
  const video = document.createElement("video");
  video.src = config.src;
  video.muted = true;
  video.playsInline = true;
  video.crossOrigin = "anonymous";

  const canvas = new OffscreenCanvas(config.gridWidth, config.gridHeight);
  const ctx = canvas.getContext("2d", {
    willReadFrequently: true,
    alpha: false,
  })!;

  let callbackId: number | null = null;
  let stopped = false;

  function processFrame(_now: DOMHighResTimeStamp, metadata: VideoFrameCallbackMetadata) {
    if (stopped) return;

    ctx.drawImage(video, 0, 0, config.gridWidth, config.gridHeight);
    const imageData = ctx.getImageData(0, 0, config.gridWidth, config.gridHeight);
    config.onFrame(imageData.data, metadata.mediaTime);

    callbackId = video.requestVideoFrameCallback(processFrame);
  }

  return {
    start() {
      stopped = false;
      callbackId = video.requestVideoFrameCallback(processFrame);
      video.play();
    },
    stop() {
      stopped = true;
      video.pause();
      if (callbackId !== null) {
        video.cancelVideoFrameCallback(callbackId);
      }
    },
  };
}
```

### Web: MediaStream (Webcam / Screen Capture)

```typescript
async function captureStream(
  source: "camera" | "screen",
  gridWidth: number,
  gridHeight: number,
): Promise<{
  stream: MediaStream;
  canvas: OffscreenCanvas;
  ctx: OffscreenCanvasRenderingContext2D;
  videoTrack: MediaStreamVideoTrack;
}> {
  const stream =
    source === "camera"
      ? await navigator.mediaDevices.getUserMedia({
          video: { width: { ideal: 640 }, height: { ideal: 480 }, frameRate: { ideal: 30 } },
          audio: false,
        })
      : await navigator.mediaDevices.getDisplayMedia({ video: true });

  const videoTrack = stream.getVideoTracks()[0];
  const canvas = new OffscreenCanvas(gridWidth, gridHeight);
  const ctx = canvas.getContext("2d", { willReadFrequently: true, alpha: false })!;

  return { stream, canvas, ctx, videoTrack };
}

async function processMediaStream(
  stream: MediaStream,
  gridWidth: number,
  gridHeight: number,
  onFrame: (pixels: Uint8ClampedArray) => void,
): Promise<() => void> {
  const videoTrack = stream.getVideoTracks()[0];

  // Use VideoFrame API for zero-copy access where supported
  if ("MediaStreamTrackProcessor" in globalThis) {
    const processor = new MediaStreamTrackProcessor({ track: videoTrack });
    const reader = processor.readable.getReader();
    const canvas = new OffscreenCanvas(gridWidth, gridHeight);
    const ctx = canvas.getContext("2d", { willReadFrequently: true, alpha: false })!;

    let running = true;

    (async () => {
      while (running) {
        const { value: frame, done } = await reader.read();
        if (done || !running) break;

        ctx.drawImage(frame, 0, 0, gridWidth, gridHeight);
        frame.close();

        const imageData = ctx.getImageData(0, 0, gridWidth, gridHeight);
        onFrame(imageData.data);
      }
    })();

    return () => {
      running = false;
      reader.cancel();
      videoTrack.stop();
    };
  }

  // Fallback: video element + rAF
  const video = document.createElement("video");
  video.srcObject = stream;
  video.muted = true;
  video.playsInline = true;
  await video.play();

  const canvas = new OffscreenCanvas(gridWidth, gridHeight);
  const ctx = canvas.getContext("2d", { willReadFrequently: true, alpha: false })!;
  let running = true;

  function tick() {
    if (!running) return;
    ctx.drawImage(video, 0, 0, gridWidth, gridHeight);
    const imageData = ctx.getImageData(0, 0, gridWidth, gridHeight);
    onFrame(imageData.data);
    requestAnimationFrame(tick);
  }
  requestAnimationFrame(tick);

  return () => {
    running = false;
    videoTrack.stop();
  };
}
```

### Python: ffmpeg Subprocess

ffmpeg decodes every format and provides raw pixel data. Always use `-pix_fmt rgb24`
for predictable 3-byte-per-pixel layout. Downscale in ffmpeg (not Python) — it uses
SIMD-optimized Lanczos internally.

```python
import subprocess
import numpy as np
from pathlib import Path


def extract_frames(
    video_path: str | Path,
    width: int,
    height: int,
    fps: float | None = None,
) -> np.ndarray:
    """Extract all frames as a contiguous (N, H, W, 3) uint8 array.

    Downscales in ffmpeg for speed. For large videos, use extract_frames_iter
    to avoid loading everything into memory.
    """
    video_path = Path(video_path)
    if not video_path.exists():
        raise FileNotFoundError(f"Video not found: {video_path}")

    vf_parts = [f"scale={width}:{height}:flags=lanczos"]
    if fps is not None:
        vf_parts.insert(0, f"fps={fps}")

    cmd = [
        "ffmpeg", "-i", str(video_path),
        "-vf", ",".join(vf_parts),
        "-pix_fmt", "rgb24",
        "-f", "rawvideo",
        "-v", "error",
        "-",
    ]
    result = subprocess.run(cmd, capture_output=True, check=True)

    raw = np.frombuffer(result.stdout, dtype=np.uint8)
    frame_size = width * height * 3
    n_frames = len(raw) // frame_size
    return raw[: n_frames * frame_size].reshape(n_frames, height, width, 3)


def extract_frames_iter(
    video_path: str | Path,
    width: int,
    height: int,
    fps: float | None = None,
):
    """Yield frames one at a time. Memory-efficient for long videos."""
    video_path = Path(video_path)
    if not video_path.exists():
        raise FileNotFoundError(f"Video not found: {video_path}")

    vf_parts = [f"scale={width}:{height}:flags=lanczos"]
    if fps is not None:
        vf_parts.insert(0, f"fps={fps}")

    cmd = [
        "ffmpeg", "-i", str(video_path),
        "-vf", ",".join(vf_parts),
        "-pix_fmt", "rgb24",
        "-f", "rawvideo",
        "-v", "error",
        "-",
    ]
    # Redirect stderr to file to prevent pipe deadlock
    proc = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.DEVNULL)
    frame_bytes = width * height * 3

    try:
        while True:
            data = proc.stdout.read(frame_bytes)
            if len(data) < frame_bytes:
                break
            frame = np.frombuffer(data, dtype=np.uint8).reshape(height, width, 3)
            yield frame.copy()  # copy — buffer gets reused
    finally:
        proc.stdout.close()
        proc.wait()


def get_video_info(video_path: str | Path) -> dict:
    """Probe video metadata: resolution, fps, duration, frame count."""
    cmd = [
        "ffprobe", "-v", "error",
        "-select_streams", "v:0",
        "-show_entries", "stream=width,height,r_frame_rate,nb_frames,duration",
        "-show_entries", "format=duration",
        "-of", "json",
        str(video_path),
    ]
    result = subprocess.run(cmd, capture_output=True, text=True, check=True)
    import json
    data = json.loads(result.stdout)
    stream = data["streams"][0]
    num, den = map(int, stream["r_frame_rate"].split("/"))
    fps = num / den

    duration = float(stream.get("duration", 0) or data["format"].get("duration", 0))
    nb_frames = int(stream.get("nb_frames", 0) or int(duration * fps))

    return {
        "width": int(stream["width"]),
        "height": int(stream["height"]),
        "fps": fps,
        "duration": duration,
        "n_frames": nb_frames,
    }
```

### Python: OpenCV VideoCapture

Fallback when ffmpeg CLI is unavailable. Slower than subprocess piping for bulk work
but convenient for interactive/notebook use.

```python
import cv2
import numpy as np


def extract_frames_cv2(
    video_path: str,
    width: int,
    height: int,
    fps: float | None = None,
):
    """Yield frames via OpenCV. Converts BGR->RGB automatically."""
    cap = cv2.VideoCapture(video_path)
    if not cap.isOpened():
        raise RuntimeError(f"Cannot open video: {video_path}")

    src_fps = cap.get(cv2.CAP_PROP_FPS)
    frame_interval = 1 if fps is None else max(1, int(round(src_fps / fps)))
    frame_idx = 0

    try:
        while True:
            ret, bgr = cap.read()
            if not ret:
                break
            if frame_idx % frame_interval == 0:
                rgb = cv2.cvtColor(bgr, cv2.COLOR_BGR2RGB)
                resized = cv2.resize(rgb, (width, height), interpolation=cv2.INTER_AREA)
                yield resized
            frame_idx += 1
    finally:
        cap.release()
```

---

## 2. Temporal Coherence

The single hardest problem in video dithering. A dithering algorithm that looks perfect on
a still image can produce unwatchable shimmer on video. This section covers why and the
solutions ranked by quality.

### Why Error Diffusion Flickers

Error diffusion (Floyd-Steinberg, Atkinson, etc.) processes pixels sequentially, propagating
quantization error to neighbors. A tiny brightness change in one pixel cascades through the
entire diffusion chain, potentially flipping hundreds of output pixels between frames.

Consider a gradient where pixel P has value 0.499 in frame N and 0.501 in frame N+1:
- Frame N: P rounds to 0, error +0.499 propagates right and down
- Frame N+1: P rounds to 1, error -0.499 propagates right and down

That sign flip ripples through every downstream pixel. The result: the dither pattern
"boils" even on near-static footage. This is intrinsic to error diffusion — there is no
threshold tweak that fixes it.

**Conclusion**: error diffusion is unsuitable for video unless combined with motion
compensation (Section 2.4). Use ordered dithering for video by default.

### Bayer Dithering: Temporally Stable Baseline

Ordered (Bayer) dithering compares each pixel against a fixed threshold matrix. The threshold
at position (x, y) never changes between frames — only the input brightness does. A pixel
flips only when its brightness crosses its specific threshold, producing localized, predictable
transitions. No cascading.

```python
def bayer_matrix(n: int) -> np.ndarray:
    """Generate n×n Bayer threshold matrix, normalized to [0, 1)."""
    if n == 2:
        return np.array([[0, 2], [3, 1]], dtype=np.float32) / 4
    half = bayer_matrix(n // 2)
    return np.block([
        [4 * half + 0, 4 * half + 2],
        [4 * half + 3, 4 * half + 1],
    ]) / (n * n)


def ordered_dither_video(
    frames: np.ndarray,  # (N, H, W) float32, range [0, 1]
    bayer_n: int = 8,
) -> np.ndarray:
    """Dither a sequence of grayscale frames with stable Bayer matrix."""
    bayer = bayer_matrix(bayer_n)
    _, h, w = frames.shape
    # Tile to cover frame — this tile is identical every frame
    tiled = np.tile(bayer, (h // bayer_n + 1, w // bayer_n + 1))[:h, :w]
    # Threshold comparison is vectorized over all frames
    return (frames > tiled[np.newaxis, :, :]).astype(np.uint8)
```

For video, Bayer is the correct default. It eliminates temporal flicker at the cost of
visible grid structure — which is less objectionable in motion than shimmer.

### Spatiotemporal Blue Noise (Wolfe et al. 2021)

Bayer has a visible grid pattern. Blue noise has no grid but is spatially randomized, which
causes temporal flicker if the noise changes per frame. Wolfe et al. solve this with
**spatiotemporal blue noise masks**: a sequence of 2D masks where each individual frame has
blue noise properties AND consecutive frames are decorrelated (temporal blue noise).

The key insight is the **golden ratio temporal offset**. Given a single blue noise mask B:

```
mask_t(x, y) = fract(B(x, y) + t × φ⁻¹)
```

where `φ⁻¹ = (√5 - 1) / 2 ≈ 0.618033988...` (conjugate golden ratio) and `fract()` returns
the fractional part.

Why this works:
- Golden ratio is the most irrational number — successive multiples of φ⁻¹ mod 1 never
  cluster, producing maximally uniform coverage of [0,1) (three-distance theorem)
- Each frame's threshold values are a uniform permutation of the original blue noise
- The spatial blue noise spectral properties are preserved in every frame
- Consecutive frames have minimal correlation (unlike random permutation which can cluster)

```python
GOLDEN_RATIO_CONJUGATE = 0.6180339887498949


def load_blue_noise(path: str) -> np.ndarray:
    """Load a precomputed blue noise texture as float32 in [0, 1)."""
    from PIL import Image
    img = np.asarray(Image.open(path).convert("L"), dtype=np.float32) / 255.0
    return img


def spatiotemporal_blue_noise(
    blue_noise: np.ndarray,  # (H, W) float32 base mask
    frame_index: int,
) -> np.ndarray:
    """Generate temporally decorrelated blue noise for frame t.

    From Wolfe et al. 2021 (arXiv:2112.09629). Each frame has blue noise
    spatial distribution, consecutive frames are maximally decorrelated.
    """
    offset = (frame_index * GOLDEN_RATIO_CONJUGATE) % 1.0
    return (blue_noise + offset) % 1.0


def dither_spatiotemporal(
    frames: np.ndarray,       # (N, H, W) float32, [0, 1]
    blue_noise: np.ndarray,   # (H, W) float32 base mask
) -> np.ndarray:
    """Dither video frames with spatiotemporal blue noise."""
    n, h, w = frames.shape
    bn_h, bn_w = blue_noise.shape
    # Tile blue noise to cover frame
    tiled = np.tile(blue_noise, (h // bn_h + 1, w // bn_w + 1))[:h, :w]

    output = np.empty((n, h, w), dtype=np.uint8)
    for t in range(n):
        mask = (tiled + (t * GOLDEN_RATIO_CONJUGATE) % 1.0) % 1.0
        output[t] = (frames[t] > mask).astype(np.uint8)
    return output
```

```typescript
const GOLDEN_RATIO_CONJUGATE = 0.6180339887498949;

function spatiotemporalThreshold(
  blueNoise: Float32Array, // flattened (noiseW × noiseH)
  noiseW: number,
  x: number,
  y: number,
  frameIndex: number,
): number {
  const nx = x % noiseW;
  const ny = y % noiseW;
  const base = blueNoise[ny * noiseW + nx];
  return (base + (frameIndex * GOLDEN_RATIO_CONJUGATE) % 1.0) % 1.0;
}
```

Quality ranking for video dithering:
```
Spatiotemporal blue noise > Bayer > White noise > Error diffusion (without motion comp)
```

### Motion-Compensated Dithering

For highest quality (offline rendering), advect the dithering pattern with the motion field.
The dither mask moves with the content, so static objects keep their exact dither pattern
and only moving regions re-threshold.

Requires optical flow estimation (dense, per-pixel motion vectors).

```python
def compute_optical_flow(
    prev_gray: np.ndarray,  # (H, W) uint8
    curr_gray: np.ndarray,  # (H, W) uint8
) -> np.ndarray:
    """Farneback dense optical flow. Returns (H, W, 2) flow field."""
    import cv2
    flow = cv2.calcOpticalFlowFarneback(
        prev_gray, curr_gray,
        flow=None,
        pyr_scale=0.5, levels=3, winsize=15,
        iterations=3, poly_n=5, poly_sigma=1.2,
        flags=0,
    )
    return flow  # flow[y, x] = (dx, dy) in pixels


def warp_mask(
    mask: np.ndarray,     # (H, W) float32 threshold mask
    flow: np.ndarray,     # (H, W, 2) optical flow
) -> np.ndarray:
    """Warp dither mask by optical flow using inverse remap."""
    import cv2
    h, w = mask.shape
    # Build inverse remap grid: where each output pixel came from
    grid_x, grid_y = np.meshgrid(np.arange(w, dtype=np.float32),
                                 np.arange(h, dtype=np.float32))
    map_x = grid_x - flow[:, :, 0]
    map_y = grid_y - flow[:, :, 1]
    warped = cv2.remap(mask, map_x, map_y, cv2.INTER_LINEAR, borderMode=cv2.BORDER_WRAP)
    return warped


def motion_compensated_dither(
    frames: np.ndarray,        # (N, H, W) float32, [0, 1]
    blue_noise: np.ndarray,    # (H, W) float32 base mask
) -> np.ndarray:
    """Dither with motion-compensated blue noise mask advection.

    The dither pattern follows object motion, eliminating temporal flicker
    on moving content. Static regions maintain perfect stability.
    """
    import cv2
    n, h, w = frames.shape
    bn_h, bn_w = blue_noise.shape
    mask = np.tile(blue_noise, (h // bn_h + 1, w // bn_w + 1))[:h, :w].copy()

    # Convert to uint8 for optical flow
    frames_u8 = np.clip(frames * 255, 0, 255).astype(np.uint8)

    output = np.empty((n, h, w), dtype=np.uint8)
    output[0] = (frames[0] > mask).astype(np.uint8)

    for t in range(1, n):
        flow = compute_optical_flow(frames_u8[t - 1], frames_u8[t])
        mask = warp_mask(mask, flow)
        # Inject golden-ratio offset to prevent mask degradation over time
        mask = (mask + GOLDEN_RATIO_CONJUGATE * 0.02) % 1.0
        output[t] = (frames[t] > mask).astype(np.uint8)

    return output
```

The small golden-ratio nudge (`* 0.02`) on each frame prevents the warped mask from
degenerating into a uniform gray as bilinear interpolation accumulates. The nudge is small
enough to preserve temporal stability but prevents the mask from losing its blue noise
spectral properties over hundreds of frames.

---

## 3. Adaptive Tonemapping

Video content has wildly varying exposure — dark scenes, bright flashes, mixed indoor/outdoor.
A fixed brightness multiplier clips highlights and crushes shadows. Percentile-based
normalization + gamma correction handles everything.

### Why Linear Brightness Multipliers Are Wrong

Given a dark scene with pixel range [10, 60] and a bright scene with range [200, 255]:
- `pixel * 2.0`: dark scene → [20, 120] (still too dark), bright scene → [400, 510] → clipped to 255
- Percentile mapping: both scenes → full [0, 255] range automatically

Linear multipliers are input-dependent. They force manual tuning per scene. Percentile
mapping adapts automatically.

### The Pipeline

```
Input (uint8) → float32 → percentile normalize → gamma curve → uint8
```

The 1st/99.5th percentiles exclude outlier pixels (specular highlights, dead pixels) that
would otherwise stretch the range wastefully. Gamma < 1.0 lifts midtones, improving ASCII
character differentiation in the critical mid-brightness range where most characters live.

### Python Implementation

```python
import numpy as np


def tonemap(
    frame: np.ndarray,
    gamma: float = 0.75,
    lo_pct: float = 1.0,
    hi_pct: float = 99.5,
) -> np.ndarray:
    """Percentile-based adaptive tonemapping for ASCII rendering.

    Args:
        frame: Input image, uint8 or float32.
        gamma: Power curve. < 1.0 lifts midtones (default 0.75 optimized
               for ASCII character differentiation).
        lo_pct: Low percentile for black point (default 1.0).
        hi_pct: High percentile for white point (default 99.5).

    Returns:
        uint8 image with full dynamic range.
    """
    f = frame.astype(np.float32)
    lo = np.percentile(f, lo_pct)
    hi = np.percentile(f, hi_pct)
    f = np.clip((f - lo) / max(hi - lo, 1.0), 0.0, 1.0) ** gamma
    return (f * 255).astype(np.uint8)


def tonemap_video(
    frames: np.ndarray,
    gamma: float = 0.75,
    temporal_smooth: float = 0.85,
) -> np.ndarray:
    """Tonemap video with temporal smoothing to prevent exposure pumping.

    Uses exponential moving average of percentile bounds so the mapping
    doesn't jerk between frames.
    """
    n = frames.shape[0]
    output = np.empty_like(frames, dtype=np.uint8)

    prev_lo = np.percentile(frames[0].astype(np.float32), 1.0)
    prev_hi = np.percentile(frames[0].astype(np.float32), 99.5)

    for t in range(n):
        f = frames[t].astype(np.float32)
        lo = np.percentile(f, 1.0)
        hi = np.percentile(f, 99.5)

        # EMA smoothing of tonemap bounds
        lo = temporal_smooth * prev_lo + (1 - temporal_smooth) * lo
        hi = temporal_smooth * prev_hi + (1 - temporal_smooth) * hi
        prev_lo, prev_hi = lo, hi

        f = np.clip((f - lo) / max(hi - lo, 1.0), 0.0, 1.0) ** gamma
        output[t] = (f * 255).astype(np.uint8)

    return output
```

```typescript
function tonemap(
  pixels: Uint8ClampedArray, // RGBA pixel data
  width: number,
  height: number,
  gamma: number = 0.75,
): Uint8ClampedArray {
  const n = width * height;
  const luminances = new Float32Array(n);

  // Extract luminance values
  for (let i = 0; i < n; i++) {
    const r = pixels[i * 4];
    const g = pixels[i * 4 + 1];
    const b = pixels[i * 4 + 2];
    luminances[i] = 0.2126 * r + 0.7152 * g + 0.0722 * b;
  }

  // Sort a copy for percentile computation
  const sorted = Float32Array.from(luminances).sort();
  const lo = sorted[Math.floor(n * 0.01)];
  const hi = sorted[Math.floor(n * 0.995)];
  const range = Math.max(hi - lo, 1);

  const out = new Uint8ClampedArray(pixels.length);
  for (let i = 0; i < n; i++) {
    const mapped = Math.pow(
      Math.max(0, Math.min(1, (luminances[i] - lo) / range)),
      gamma,
    );
    const scale = luminances[i] > 0 ? (mapped * 255) / luminances[i] : 0;
    out[i * 4] = Math.min(255, pixels[i * 4] * scale);
    out[i * 4 + 1] = Math.min(255, pixels[i * 4 + 1] * scale);
    out[i * 4 + 2] = Math.min(255, pixels[i * 4 + 2] * scale);
    out[i * 4 + 3] = 255;
  }
  return out;
}

class TemporalTonemapper {
  private prevLo = 0;
  private prevHi = 255;
  private readonly smoothing: number;

  constructor(smoothing: number = 0.85) {
    this.smoothing = smoothing;
  }

  process(
    pixels: Uint8ClampedArray,
    width: number,
    height: number,
    gamma: number = 0.75,
  ): Uint8ClampedArray {
    const n = width * height;
    const luminances = new Float32Array(n);

    for (let i = 0; i < n; i++) {
      luminances[i] =
        0.2126 * pixels[i * 4] +
        0.7152 * pixels[i * 4 + 1] +
        0.0722 * pixels[i * 4 + 2];
    }

    const sorted = Float32Array.from(luminances).sort();
    let lo = sorted[Math.floor(n * 0.01)];
    let hi = sorted[Math.floor(n * 0.995)];

    lo = this.smoothing * this.prevLo + (1 - this.smoothing) * lo;
    hi = this.smoothing * this.prevHi + (1 - this.smoothing) * hi;
    this.prevLo = lo;
    this.prevHi = hi;

    const range = Math.max(hi - lo, 1);
    const out = new Uint8ClampedArray(pixels.length);

    for (let i = 0; i < n; i++) {
      const mapped = Math.pow(Math.max(0, Math.min(1, (luminances[i] - lo) / range)), gamma);
      const scale = luminances[i] > 0 ? (mapped * 255) / luminances[i] : 0;
      out[i * 4] = Math.min(255, pixels[i * 4] * scale);
      out[i * 4 + 1] = Math.min(255, pixels[i * 4 + 1] * scale);
      out[i * 4 + 2] = Math.min(255, pixels[i * 4 + 2] * scale);
      out[i * 4 + 3] = 255;
    }
    return out;
  }
}
```

---

## 4. Python/NumPy Vectorized Pipeline

Complete frame rendering pipeline, fully vectorized. No per-pixel Python loops.

### sRGB Linearization LUT

```python
import numpy as np

SRGB_TO_LINEAR = np.zeros(256, dtype=np.float32)
for _i in range(256):
    s = _i / 255.0
    SRGB_TO_LINEAR[_i] = s / 12.92 if s <= 0.04045 else ((s + 0.055) / 1.055) ** 2.4

LINEAR_TO_SRGB = np.zeros(4096, dtype=np.uint8)
for _i in range(4096):
    c = _i / 4095.0
    s = c * 12.92 if c <= 0.0031308 else 1.055 * c ** (1 / 2.4) - 0.055
    LINEAR_TO_SRGB[_i] = int(np.clip(s * 255, 0, 255))


def linearize(img: np.ndarray) -> np.ndarray:
    """sRGB uint8 → linear float32 via LUT. Fully vectorized."""
    return SRGB_TO_LINEAR[img]


def delinearize(img: np.ndarray) -> np.ndarray:
    """Linear float32 → sRGB uint8 via LUT."""
    idx = np.clip((img * 4095).astype(np.int32), 0, 4095)
    return LINEAR_TO_SRGB[idx]
```

### Vectorized Bayer Dithering

```python
def vectorized_bayer_dither(
    luminance: np.ndarray,  # (H, W) float32 in [0, 1]
    bayer_n: int = 8,
    levels: int = 2,
) -> np.ndarray:
    """Fully vectorized Bayer dither. No Python loops.

    Args:
        luminance: Grayscale image, float32 [0, 1].
        bayer_n: Bayer matrix size (must be power of 2).
        levels: Number of output levels (2 = binary, 4 = 4-tone, etc.).

    Returns:
        uint8 array with values in {0, 1, ..., levels-1}.
    """
    bayer = bayer_matrix(bayer_n)
    h, w = luminance.shape
    threshold = np.tile(bayer, (h // bayer_n + 1, w // bayer_n + 1))[:h, :w]

    if levels == 2:
        return (luminance > threshold).astype(np.uint8)

    # Multi-level: scale luminance to [0, levels-1] and add threshold bias
    scaled = luminance * (levels - 1) + threshold - 0.5
    return np.clip(np.round(scaled), 0, levels - 1).astype(np.uint8)
```

### Character Bitmap Cache

Pre-render every ASCII character to a NumPy array for fast compositing.

```python
from functools import lru_cache
from PIL import Image, ImageDraw, ImageFont


class CharBitmapCache:
    """Pre-rendered character bitmaps for fast frame compositing."""

    def __init__(
        self,
        font_path: str,
        font_size: int,
        char_width: int,
        char_height: int,
        charset: str = " .:-=+*#%@",
    ):
        self.char_w = char_width
        self.char_h = char_height
        self.charset = charset
        self.font = ImageFont.truetype(font_path, font_size)

        # Pre-render all characters to uint8 arrays
        self.bitmaps: dict[str, np.ndarray] = {}
        for ch in charset:
            img = Image.new("L", (char_width, char_height), 0)
            draw = ImageDraw.Draw(img)
            bbox = draw.textbbox((0, 0), ch, font=self.font)
            x_off = (char_width - (bbox[2] - bbox[0])) // 2
            y_off = (char_height - (bbox[3] - bbox[1])) // 2
            draw.text((x_off - bbox[0], y_off - bbox[1]), ch, fill=255, font=self.font)
            self.bitmaps[ch] = np.asarray(img, dtype=np.uint8)

        # Build lookup array indexed by quantized level
        n_chars = len(charset)
        self.level_bitmaps = np.stack(
            [self.bitmaps[charset[i]] for i in range(n_chars)]
        )  # (n_chars, char_h, char_w)

    def render_frame(
        self,
        char_grid: np.ndarray,  # (rows, cols) uint8, values = character indices
        fg_color: tuple[int, int, int] = (255, 255, 255),
        bg_color: tuple[int, int, int] = (0, 0, 0),
    ) -> np.ndarray:
        """Compose character grid into an RGB image. Fully vectorized."""
        rows, cols = char_grid.shape
        h = rows * self.char_h
        w = cols * self.char_w

        # Gather bitmaps for all grid cells at once
        # level_bitmaps[char_grid] → (rows, cols, char_h, char_w)
        cell_bitmaps = self.level_bitmaps[char_grid]

        # Reshape to image: (rows, cols, char_h, char_w) → (H, W)
        # Transpose axes to interleave rows and cols correctly
        alpha = cell_bitmaps.transpose(0, 2, 1, 3).reshape(h, w).astype(np.float32) / 255.0

        # Blend foreground and background
        frame = np.empty((h, w, 3), dtype=np.uint8)
        for c in range(3):
            frame[:, :, c] = (
                bg_color[c] + alpha * (fg_color[c] - bg_color[c])
            ).astype(np.uint8)

        return frame
```

### Parallel Rendering with ProcessPoolExecutor

```python
import os
from concurrent.futures import ProcessPoolExecutor


def render_chunk(args: tuple) -> np.ndarray:
    """Render a batch of frames. Designed for multiprocessing."""
    frames, cache_params, bayer_n, gamma = args

    cache = CharBitmapCache(**cache_params)
    bayer = bayer_matrix(bayer_n)

    results = []
    for frame in frames:
        gray = np.dot(linearize(frame), [0.2126, 0.7152, 0.0722])
        gray = delinearize(gray[:, :, np.newaxis].repeat(3, axis=2))[:, :, 0]
        tonemapped = tonemap(gray, gamma=gamma)
        lum = tonemapped.astype(np.float32) / 255.0
        char_indices = vectorized_bayer_dither(lum, bayer_n, levels=len(cache.charset))
        rendered = cache.render_frame(char_indices)
        results.append(rendered)

    return np.stack(results)


def render_video_parallel(
    frames: np.ndarray,          # (N, H, W, 3) uint8
    cache_params: dict,
    bayer_n: int = 8,
    gamma: float = 0.75,
    n_workers: int | None = None,
    chunk_size: int = 32,
) -> np.ndarray:
    """Render all frames in parallel across CPU cores.

    Args:
        frames: Input video frames.
        cache_params: kwargs for CharBitmapCache constructor.
        bayer_n: Bayer matrix order.
        gamma: Tonemap gamma.
        n_workers: Number of processes (default: CPU count - 1).
        chunk_size: Frames per chunk.

    Returns:
        (N, rendered_H, rendered_W, 3) uint8 array.
    """
    if n_workers is None:
        n_workers = max(1, os.cpu_count() - 1)

    n = len(frames)
    chunks = [
        (frames[i : i + chunk_size], cache_params, bayer_n, gamma)
        for i in range(0, n, chunk_size)
    ]

    rendered_chunks = []
    with ProcessPoolExecutor(max_workers=n_workers) as executor:
        for result in executor.map(render_chunk, chunks):
            rendered_chunks.append(result)

    return np.concatenate(rendered_chunks, axis=0)
```

---

## 5. ffmpeg Encoding

### Piping Raw RGB Frames to ffmpeg

The critical pattern: send raw RGB frames on stdin, redirect stderr to a file (not PIPE).
If you pipe both stdout and stderr, Python's `subprocess` can deadlock when ffmpeg's stderr
buffer fills and blocks while Python is blocked writing to stdin.

```python
import subprocess
import tempfile
from pathlib import Path


def encode_h264(
    frames: np.ndarray,       # (N, H, W, 3) uint8 RGB
    output_path: str | Path,
    fps: float = 30.0,
    crf: int = 18,
    preset: str = "slow",
    pixel_format: str = "yuv420p",
) -> None:
    """Encode frames to H.264 MP4 via ffmpeg stdin pipe.

    CRF guide:
      0     = lossless (huge files)
      15-17 = visually lossless
      18-22 = high quality (default 18 for ASCII art — sharp edges need it)
      23-28 = medium quality
      28+   = noticeable artifacts

    For ASCII/dither output, use CRF 18 or lower — the sharp edges and
    high-frequency patterns in dithered content are destroyed by aggressive
    quantization.
    """
    _, h, w, _ = frames.shape
    output_path = Path(output_path)
    output_path.parent.mkdir(parents=True, exist_ok=True)

    stderr_log = tempfile.NamedTemporaryFile(
        mode="w", suffix=".log", prefix="ffmpeg_", delete=False
    )

    cmd = [
        "ffmpeg", "-y",
        "-f", "rawvideo",
        "-vcodec", "rawvideo",
        "-s", f"{w}x{h}",
        "-pix_fmt", "rgb24",
        "-r", str(fps),
        "-i", "-",
        "-c:v", "libx264",
        "-preset", preset,
        "-crf", str(crf),
        "-pix_fmt", pixel_format,
        "-movflags", "+faststart",
        str(output_path),
    ]

    proc = subprocess.Popen(
        cmd,
        stdin=subprocess.PIPE,
        stdout=subprocess.DEVNULL,
        stderr=stderr_log,  # FILE, not PIPE — prevents deadlock
    )

    try:
        for frame in frames:
            proc.stdin.write(frame.tobytes())
        proc.stdin.close()
    except BrokenPipeError:
        pass

    ret = proc.wait()
    stderr_log.close()

    if ret != 0:
        log_content = Path(stderr_log.name).read_text()
        raise RuntimeError(f"ffmpeg exited with code {ret}:\n{log_content}")

    Path(stderr_log.name).unlink(missing_ok=True)


def encode_gif(
    frames: np.ndarray,
    output_path: str | Path,
    fps: float = 15.0,
    width: int | None = None,
    max_colors: int = 64,
) -> None:
    """Two-pass GIF encoding with global palette for quality.

    Uses ffmpeg's palettegen/paletteuse filters for optimal color quantization.
    For dithered content, reduce max_colors since the image is already quantized.
    """
    _, h, w, _ = frames.shape
    output_path = Path(output_path)
    palette_path = output_path.with_suffix(".palette.png")

    scale_filter = f"scale={width}:-1:flags=lanczos," if width else ""
    stderr_log = tempfile.NamedTemporaryFile(
        mode="w", suffix=".log", prefix="ffmpeg_gif_", delete=False
    )

    # Pass 1: generate palette
    cmd_palette = [
        "ffmpeg", "-y",
        "-f", "rawvideo", "-vcodec", "rawvideo",
        "-s", f"{w}x{h}", "-pix_fmt", "rgb24", "-r", str(fps),
        "-i", "-",
        "-vf", f"{scale_filter}palettegen=max_colors={max_colors}:stats_mode=diff",
        str(palette_path),
    ]

    proc = subprocess.Popen(
        cmd_palette,
        stdin=subprocess.PIPE,
        stdout=subprocess.DEVNULL,
        stderr=stderr_log,
    )
    raw_bytes = frames.tobytes()
    proc.stdin.write(raw_bytes)
    proc.stdin.close()
    proc.wait()

    # Pass 2: encode with palette
    cmd_encode = [
        "ffmpeg", "-y",
        "-f", "rawvideo", "-vcodec", "rawvideo",
        "-s", f"{w}x{h}", "-pix_fmt", "rgb24", "-r", str(fps),
        "-i", "-",
        "-i", str(palette_path),
        "-lavfi", f"{scale_filter}paletteuse=dither=bayer:bayer_scale=3",
        str(output_path),
    ]

    proc = subprocess.Popen(
        cmd_encode,
        stdin=subprocess.PIPE,
        stdout=subprocess.DEVNULL,
        stderr=stderr_log,
    )
    proc.stdin.write(raw_bytes)
    proc.stdin.close()
    proc.wait()

    stderr_log.close()
    palette_path.unlink(missing_ok=True)
    Path(stderr_log.name).unlink(missing_ok=True)


def encode_streaming(
    frame_iter,              # Iterator yielding (H, W, 3) uint8 frames
    output_path: str | Path,
    width: int,
    height: int,
    fps: float = 30.0,
    crf: int = 18,
) -> None:
    """Stream frames to ffmpeg without holding all frames in memory.

    Use with extract_frames_iter() for constant-memory video processing.
    """
    output_path = Path(output_path)
    output_path.parent.mkdir(parents=True, exist_ok=True)

    stderr_log = tempfile.NamedTemporaryFile(
        mode="w", suffix=".log", prefix="ffmpeg_stream_", delete=False
    )

    cmd = [
        "ffmpeg", "-y",
        "-f", "rawvideo", "-vcodec", "rawvideo",
        "-s", f"{width}x{height}",
        "-pix_fmt", "rgb24", "-r", str(fps),
        "-i", "-",
        "-c:v", "libx264", "-preset", "medium",
        "-crf", str(crf), "-pix_fmt", "yuv420p",
        "-movflags", "+faststart",
        str(output_path),
    ]

    proc = subprocess.Popen(
        cmd,
        stdin=subprocess.PIPE,
        stdout=subprocess.DEVNULL,
        stderr=stderr_log,
    )

    try:
        for frame in frame_iter:
            proc.stdin.write(frame.tobytes())
        proc.stdin.close()
    except BrokenPipeError:
        pass

    ret = proc.wait()
    stderr_log.close()

    if ret != 0:
        log_content = Path(stderr_log.name).read_text()
        raise RuntimeError(f"ffmpeg exited with code {ret}:\n{log_content}")

    Path(stderr_log.name).unlink(missing_ok=True)
```

### Common Pitfall: The Deadlock

```python
# WRONG — deadlocks on long videos when stderr buffer fills:
proc = subprocess.Popen(cmd, stdin=PIPE, stdout=PIPE, stderr=PIPE)
proc.stdin.write(huge_data)  # blocks because stderr buffer is full

# CORRECT — stderr goes to a file, stdin never blocks:
proc = subprocess.Popen(cmd, stdin=PIPE, stdout=DEVNULL, stderr=log_file)
```

`subprocess.communicate()` avoids the deadlock by reading stderr in a thread, but it
buffers the entire output in memory. For video encoding, redirect stderr to a file.

---

## 6. Web Video Processing

### Two-Canvas Architecture

Real-time video-to-ASCII requires two canvases:
1. **Sampler canvas** (offscreen): downscaled to grid resolution, reads pixel data
2. **Display canvas** (visible): renders ASCII characters at full resolution

Separating these is critical — reading pixels from a large canvas is slow. The sampler
canvas should be exactly grid-sized (e.g., 120x67 for a 120-column layout).

```typescript
interface AsciiVideoConfig {
  gridCols: number;
  gridRows: number;
  cellWidth: number;
  cellHeight: number;
  charset: string;
  fontFamily: string;
  fontSize: number;
  fgColor: string;
  bgColor: string;
}

class AsciiVideoRenderer {
  private sampler: OffscreenCanvas;
  private samplerCtx: OffscreenCanvasRenderingContext2D;
  private display: HTMLCanvasElement;
  private displayCtx: CanvasRenderingContext2D;
  private config: AsciiVideoConfig;
  private tonemapper: TemporalTonemapper;

  private frameCount = 0;
  private lastFrameTime = 0;
  private fps = 0;

  constructor(display: HTMLCanvasElement, config: AsciiVideoConfig) {
    this.config = config;
    this.display = display;
    this.tonemapper = new TemporalTonemapper(0.85);

    display.width = config.gridCols * config.cellWidth;
    display.height = config.gridRows * config.cellHeight;

    this.displayCtx = display.getContext("2d", { alpha: false })!;
    this.displayCtx.font = `${config.fontSize}px ${config.fontFamily}`;
    this.displayCtx.textBaseline = "top";

    this.sampler = new OffscreenCanvas(config.gridCols, config.gridRows);
    this.samplerCtx = this.sampler.getContext("2d", {
      willReadFrequently: true,
      alpha: false,
    })!;
  }

  processFrame(source: CanvasImageSource): void {
    const { gridCols, gridRows, cellWidth, cellHeight, charset, fgColor, bgColor } =
      this.config;

    // Downsample to grid resolution
    this.samplerCtx.drawImage(source, 0, 0, gridCols, gridRows);
    const imageData = this.samplerCtx.getImageData(0, 0, gridCols, gridRows);
    const pixels = imageData.data;

    // Tonemap
    const mapped = this.tonemapper.process(pixels, gridCols, gridRows, 0.75);

    // Clear and render
    this.displayCtx.fillStyle = bgColor;
    this.displayCtx.fillRect(0, 0, gridCols * cellWidth, gridRows * cellHeight);
    this.displayCtx.fillStyle = fgColor;

    const nChars = charset.length;
    for (let y = 0; y < gridRows; y++) {
      for (let x = 0; x < gridCols; x++) {
        const i = (y * gridCols + x) * 4;
        const lum =
          0.2126 * mapped[i] + 0.7152 * mapped[i + 1] + 0.0722 * mapped[i + 2];
        const charIdx = Math.min(nChars - 1, Math.floor((lum / 255) * nChars));
        const ch = charset[charIdx];
        if (ch !== " ") {
          this.displayCtx.fillText(ch, x * cellWidth, y * cellHeight);
        }
      }
    }

    // FPS tracking
    this.frameCount++;
    const now = performance.now();
    if (now - this.lastFrameTime >= 1000) {
      this.fps = this.frameCount;
      this.frameCount = 0;
      this.lastFrameTime = now;
    }
  }

  getFps(): number {
    return this.fps;
  }
}
```

### requestAnimationFrame Loop with Timing

```typescript
function startVideoLoop(
  video: HTMLVideoElement,
  renderer: AsciiVideoRenderer,
): () => void {
  let running = true;
  let lastVideoTime = -1;

  function tick() {
    if (!running) return;

    // Only process if video advanced to a new frame
    if (video.currentTime !== lastVideoTime && !video.paused) {
      lastVideoTime = video.currentTime;
      renderer.processFrame(video);
    }

    requestAnimationFrame(tick);
  }

  requestAnimationFrame(tick);

  return () => {
    running = false;
  };
}

// With requestVideoFrameCallback (preferred — fires per decoded frame):
function startVideoLoopPrecise(
  video: HTMLVideoElement,
  renderer: AsciiVideoRenderer,
): () => void {
  let running = true;

  function onFrame(_now: DOMHighResTimeStamp, _meta: VideoFrameCallbackMetadata) {
    if (!running) return;
    renderer.processFrame(video);
    video.requestVideoFrameCallback(onFrame);
  }

  video.requestVideoFrameCallback(onFrame);

  return () => {
    running = false;
  };
}
```

### Webcam Integration

```typescript
async function startWebcamAscii(
  displayCanvas: HTMLCanvasElement,
  config: AsciiVideoConfig,
): Promise<() => void> {
  const stream = await navigator.mediaDevices.getUserMedia({
    video: {
      width: { ideal: 640 },
      height: { ideal: 480 },
      frameRate: { ideal: 30 },
    },
    audio: false,
  });

  const video = document.createElement("video");
  video.srcObject = stream;
  video.muted = true;
  video.playsInline = true;
  await video.play();

  const renderer = new AsciiVideoRenderer(displayCanvas, config);
  let running = true;

  function tick() {
    if (!running) return;
    renderer.processFrame(video);
    requestAnimationFrame(tick);
  }
  requestAnimationFrame(tick);

  return () => {
    running = false;
    stream.getTracks().forEach((t) => t.stop());
    video.srcObject = null;
  };
}
```

### Performance: Grid Sizing for 30fps

The bottleneck is `getImageData` + character rendering. Approximate budget at 30fps
(33ms per frame):

```
Grid size      getImageData    Render loop     Total     Achievable?
80×45          ~0.1ms          ~2ms            ~2ms      Easy
120×67         ~0.2ms          ~5ms            ~5ms      Easy
160×90         ~0.3ms          ~9ms            ~9ms      Yes
200×112        ~0.4ms          ~14ms           ~14ms     Yes (tight)
240×135        ~0.5ms          ~20ms           ~20ms     Marginal
320×180        ~0.8ms          ~35ms           ~36ms     No — drop to 24fps
```

Guidance:
- **120x67** is the sweet spot for most content — readable characters, solid 60fps
- **160x90** for detail-heavy content, still 30fps comfortable
- Above 200 columns: consider WebGL rendering instead of Canvas 2D fillText
- `fillText` per character is the bottleneck — for larger grids, pre-render characters
  to ImageBitmap and use `drawImage` (2-3x faster than fillText)

---

## 7. Hardware Detection + Quality Profiles

### Detecting Available Resources

```typescript
interface HardwareProfile {
  cpuCores: number;
  memoryGB: number | null;
  hasGPU: boolean;
  gpuRenderer: string | null;
  isMobile: boolean;
  maxTextureSize: number;
}

function detectHardware(): HardwareProfile {
  const cpuCores = navigator.hardwareConcurrency || 4;
  const memoryGB =
    "deviceMemory" in navigator ? (navigator as any).deviceMemory : null;
  const isMobile = /Android|iPhone|iPad/.test(navigator.userAgent);

  let hasGPU = false;
  let gpuRenderer: string | null = null;
  let maxTextureSize = 4096;

  const testCanvas = document.createElement("canvas");
  const gl = testCanvas.getContext("webgl2") || testCanvas.getContext("webgl");
  if (gl) {
    hasGPU = true;
    const dbg = gl.getExtension("WEBGL_debug_renderer_info");
    if (dbg) {
      gpuRenderer = gl.getParameter(dbg.UNMASKED_RENDERER_WEBGL);
    }
    maxTextureSize = gl.getParameter(gl.MAX_TEXTURE_SIZE);
    const loseCtx = gl.getExtension("WEBGL_lose_context");
    loseCtx?.loseContext();
  }

  return { cpuCores, memoryGB, hasGPU, gpuRenderer, isMobile, maxTextureSize };
}
```

```python
import os
import shutil
import platform


def detect_hardware() -> dict:
    """Detect available hardware for quality profile selection."""
    cpu_count = os.cpu_count() or 4
    try:
        import psutil
        memory_gb = psutil.virtual_memory().total / (1024 ** 3)
    except ImportError:
        memory_gb = None

    has_gpu = False
    gpu_name = None
    try:
        import subprocess
        result = subprocess.run(
            ["nvidia-smi", "--query-gpu=name", "--format=csv,noheader"],
            capture_output=True, text=True, timeout=5,
        )
        if result.returncode == 0 and result.stdout.strip():
            has_gpu = True
            gpu_name = result.stdout.strip().split("\n")[0]
    except (FileNotFoundError, subprocess.TimeoutExpired):
        pass

    has_ffmpeg = shutil.which("ffmpeg") is not None

    return {
        "cpu_cores": cpu_count,
        "memory_gb": memory_gb,
        "has_gpu": has_gpu,
        "gpu_name": gpu_name,
        "has_ffmpeg": has_ffmpeg,
        "platform": platform.system(),
    }
```

### Quality Presets

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class QualityProfile:
    name: str
    grid_cols: int
    grid_rows: int
    bayer_n: int
    charset: str
    dither_method: str      # "bayer", "blue_noise", "motion_compensated"
    gamma: float
    temporal_smooth: float
    ffmpeg_crf: int
    ffmpeg_preset: str
    n_workers: int
    chunk_size: int


# Draft: fast preview, low quality
DRAFT = QualityProfile(
    name="draft",
    grid_cols=80,
    grid_rows=45,
    bayer_n=4,
    charset=" .:-=+*#%@",
    dither_method="bayer",
    gamma=0.8,
    temporal_smooth=0.8,
    ffmpeg_crf=28,
    ffmpeg_preset="ultrafast",
    n_workers=2,
    chunk_size=64,
)

# Preview: balanced speed and quality
PREVIEW = QualityProfile(
    name="preview",
    grid_cols=120,
    grid_rows=67,
    bayer_n=8,
    charset=" .:-=+*#%@",
    dither_method="bayer",
    gamma=0.75,
    temporal_smooth=0.85,
    ffmpeg_crf=22,
    ffmpeg_preset="medium",
    n_workers=4,
    chunk_size=32,
)

# Production: maximum quality, slow
PRODUCTION = QualityProfile(
    name="production",
    grid_cols=160,
    grid_rows=90,
    bayer_n=8,
    charset=" .'`^\",:;Il!i><~+_-?][}{1)(|\\/tfjrxnuvczXYUJCLQ0OZmwqpdbkhao*#MW&8%B@$",
    dither_method="blue_noise",
    gamma=0.72,
    temporal_smooth=0.9,
    ffmpeg_crf=18,
    ffmpeg_preset="slow",
    n_workers=-1,  # all cores
    chunk_size=16,
)


def select_profile(hardware: dict) -> QualityProfile:
    """Auto-select quality profile based on detected hardware."""
    cpu = hardware["cpu_cores"]
    mem = hardware.get("memory_gb") or 8

    if cpu <= 2 or mem < 4:
        return DRAFT
    if cpu <= 4 or mem < 8:
        return PREVIEW

    profile = PRODUCTION
    if hardware.get("has_gpu"):
        # With GPU, can afford larger grid and motion compensation
        return QualityProfile(
            name="production_gpu",
            grid_cols=200,
            grid_rows=112,
            bayer_n=8,
            charset=PRODUCTION.charset,
            dither_method="motion_compensated",
            gamma=0.72,
            temporal_smooth=0.9,
            ffmpeg_crf=16,
            ffmpeg_preset="slow",
            n_workers=cpu - 1,
            chunk_size=16,
        )

    return QualityProfile(
        name=profile.name,
        grid_cols=profile.grid_cols,
        grid_rows=profile.grid_rows,
        bayer_n=profile.bayer_n,
        charset=profile.charset,
        dither_method=profile.dither_method,
        gamma=profile.gamma,
        temporal_smooth=profile.temporal_smooth,
        ffmpeg_crf=profile.ffmpeg_crf,
        ffmpeg_preset=profile.ffmpeg_preset,
        n_workers=max(1, cpu - 1),
        chunk_size=profile.chunk_size,
    )
```

```typescript
interface QualityProfile {
  name: string;
  gridCols: number;
  gridRows: number;
  bayerN: number;
  charset: string;
  gamma: number;
  useWebGL: boolean;
}

function selectWebProfile(hw: HardwareProfile): QualityProfile {
  if (hw.isMobile) {
    return {
      name: "mobile",
      gridCols: 60,
      gridRows: 34,
      bayerN: 4,
      charset: " .:-=+*#%@",
      gamma: 0.8,
      useWebGL: false,
    };
  }

  if (hw.cpuCores <= 4 || (hw.memoryGB !== null && hw.memoryGB < 4)) {
    return {
      name: "draft",
      gridCols: 80,
      gridRows: 45,
      bayerN: 4,
      charset: " .:-=+*#%@",
      gamma: 0.8,
      useWebGL: false,
    };
  }

  if (hw.hasGPU && hw.maxTextureSize >= 4096) {
    return {
      name: "production",
      gridCols: 200,
      gridRows: 112,
      bayerN: 8,
      charset: " .:-=+*#%@",
      gamma: 0.72,
      useWebGL: true,
    };
  }

  return {
    name: "preview",
    gridCols: 120,
    gridRows: 67,
    bayerN: 8,
    charset: " .:-=+*#%@",
    gamma: 0.75,
    useWebGL: false,
  };
}
```

### Full Pipeline Integration

Putting it all together — detect hardware, select profile, process video.

```python
def process_video(
    input_path: str,
    output_path: str,
    font_path: str,
    profile: QualityProfile | None = None,
) -> None:
    """End-to-end video processing pipeline.

    Detects hardware, selects quality profile, extracts frames,
    tonemaps, dithers, renders characters, and encodes output.
    """
    hw = detect_hardware()
    if profile is None:
        profile = select_profile(hw)

    info = get_video_info(input_path)
    n_workers = profile.n_workers if profile.n_workers > 0 else max(1, hw["cpu_cores"] - 1)
    n_chars = len(profile.charset)
    cell_w, cell_h = 8, 14  # monospace cell size at typical font size

    cache_params = {
        "font_path": font_path,
        "font_size": 14,
        "char_width": cell_w,
        "char_height": cell_h,
        "charset": profile.charset,
    }

    cache = CharBitmapCache(**cache_params)

    out_w = profile.grid_cols * cell_w
    out_h = profile.grid_rows * cell_h

    stderr_log = open(f"{output_path}.ffmpeg.log", "w")
    cmd = [
        "ffmpeg", "-y",
        "-f", "rawvideo", "-vcodec", "rawvideo",
        "-s", f"{out_w}x{out_h}",
        "-pix_fmt", "rgb24", "-r", str(info["fps"]),
        "-i", "-",
        "-c:v", "libx264", "-preset", profile.ffmpeg_preset,
        "-crf", str(profile.ffmpeg_crf), "-pix_fmt", "yuv420p",
        "-movflags", "+faststart",
        output_path,
    ]
    proc = subprocess.Popen(
        cmd, stdin=subprocess.PIPE, stdout=subprocess.DEVNULL, stderr=stderr_log,
    )

    prev_lo, prev_hi = None, None

    for frame in extract_frames_iter(input_path, profile.grid_cols, profile.grid_rows):
        # Linearize, compute luminance, delinearize
        lin = linearize(frame)
        gray = np.dot(lin, [0.2126, 0.7152, 0.0722])
        gray_srgb = delinearize(gray[:, :, np.newaxis].repeat(3, axis=2))[:, :, 0]

        # Tonemap with temporal smoothing
        f = gray_srgb.astype(np.float32)
        lo = np.percentile(f, 1.0)
        hi = np.percentile(f, 99.5)
        if prev_lo is not None:
            lo = profile.temporal_smooth * prev_lo + (1 - profile.temporal_smooth) * lo
            hi = profile.temporal_smooth * prev_hi + (1 - profile.temporal_smooth) * hi
        prev_lo, prev_hi = lo, hi
        lum = np.clip((f - lo) / max(hi - lo, 1.0), 0.0, 1.0) ** profile.gamma

        # Dither to character indices
        char_indices = vectorized_bayer_dither(lum, profile.bayer_n, levels=n_chars)

        # Render to RGB
        rendered = cache.render_frame(char_indices)

        # Pipe to ffmpeg
        try:
            proc.stdin.write(rendered.tobytes())
        except BrokenPipeError:
            break

    proc.stdin.close()
    proc.wait()
    stderr_log.close()
```
