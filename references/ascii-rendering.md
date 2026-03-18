# ASCII Rendering Reference

Complete reference for implementing ASCII art renderers and dithering effects. All code is production TypeScript targeting Canvas 2D. Every function is self-contained -- copy what you need.

---

## 1. Character-to-Brightness Mapping

The fundamental primitive: map a normalized brightness value [0,1] to a character from a sorted palette string where characters are ordered darkest (index 0, usually space) to brightest (last index).

```typescript
function getCharForBrightness(brightness: number, chars: string): string {
  if (chars.length === 0) return ' '
  const clamped = Math.max(0, Math.min(1, brightness))
  return chars[Math.floor(clamped * (chars.length - 1))]
}
```

The mapping is linear: brightness 0.0 returns `chars[0]`, brightness 1.0 returns `chars[chars.length - 1]`. Intermediate values are evenly distributed. A palette with N characters provides N discrete brightness levels.

### Brightness Computation

Always use perceptual luminance (BT.709) with gamma-correct sRGB linearization. Raw pixel values produce incorrect brightness mapping.

```typescript
const SRGB_LUT = new Float32Array(256)
for (let i = 0; i < 256; i++) {
  const s = i / 255
  SRGB_LUT[i] = s <= 0.04045 ? s / 12.92 : Math.pow((s + 0.055) / 1.055, 2.4)
}

function pixelBrightness(r: number, g: number, b: number): number {
  return 0.2126 * SRGB_LUT[r] + 0.7152 * SRGB_LUT[g] + 0.0722 * SRGB_LUT[b]
}

function adjustBrightness(value: number, brightness: number, contrast: number): number {
  let v = value + brightness
  v = (v - 0.5) * contrast + 0.5
  return Math.max(0, Math.min(1, v))
}
```

- `brightness`: additive offset, typically -0.5 to 0.5
- `contrast`: multiplicative factor around 0.5 midpoint, typically 0.5 to 2.0

---

## 2. Charset Presets

All palettes ordered dark-to-bright (space = darkest, last char = brightest). The brightness mapper indexes into these strings linearly.

### Core Charsets

```typescript
const CHARSET_STANDARD  = ' .:-=+*#%@'
const CHARSET_BLOCKS    = ' ░▒▓█'
const CHARSET_DETAILED  = " .'`^\",:;Il!i><~+_-?][}{1)(|\\/tfjrxnuvczXYUJCLQ0OZmwqpdbkhao*#MW&8%B@$"
const CHARSET_MINIMAL   = ' ·░█'
const CHARSET_BINARY    = ' 01'
const CHARSET_DENSE     = ' .:;+=xX$#@█'
const CHARSET_DEFAULT   = " .`'-:;!><=+*^~?/|(){}[]#&$@%"
```

### Letter Sets

```typescript
const CHARSET_LETTERS_UPPER = ' ABCDEFGHIJKLMNOPQRSTUVWXYZ'
const CHARSET_LETTERS_LOWER = ' abcdefghijklmnopqrstuvwxyz'
const CHARSET_LETTERS_MIXED = ' aAbBcCdDeEfFgGhHiIjJkKlLmMnNoOpPqQrRsStTuUvVwWxXyYzZ'
const CHARSET_SYMBOLS       = ' !@#$%^&*()+-=[]{}|;:,.<>?/'
```

### Braille Patterns

```typescript
// Full 64-char Braille block (U+2800-U+283F), maximum tonal range
const CHARSET_BRAILLE_STANDARD =
  ' ⠁⠂⠃⠄⠅⠆⠇⠈⠉⠊⠋⠌⠍⠎⠏⠐⠑⠒⠓⠔⠕⠖⠗⠘⠙⠚⠛⠜⠝⠞⠟⠠⠡⠢⠣⠤⠥⠦⠧⠨⠩⠪⠫⠬⠭⠮⠯⠰⠱⠲⠳⠴⠵⠶⠷⠸⠹⠺⠻⠼⠽⠾⠿'

// Sparse (10 chars) -- clear brightness steps
const CHARSET_BRAILLE_SPARSE = ' ⠁⠉⠋⠛⠟⠿⡿⣿⣿'

// Dense (5 chars) -- blocky rendering
const CHARSET_BRAILLE_DENSE = ' ⠐⠶⣿⣿'

// Full 256-char Braille block (U+2800-U+28FF) for sub-cell rendering
const CHARSET_BRAILLE_FULL =
  '⠀⠁⠂⠃⠄⠅⠆⠇⡀⡁⡂⡃⡄⡅⡆⡇⠈⠉⠊⠋⠌⠍⠎⠏⡈⡉⡊⡋⡌⡍⡎⡏' +
  '⠐⠑⠒⠓⠔⠕⠖⠗⡐⡑⡒⡓⡔⡕⡖⡗⠘⠙⠚⠛⠜⠝⠞⠟⡘⡙⡚⡛⡜⡝⡞⡟' +
  '⠠⠡⠢⠣⠤⠥⠦⠧⡠⡡⡢⡣⡤⡥⡦⡧⠨⠩⠪⠫⠬⠭⠮⠯⡨⡩⡪⡫⡬⡭⡮⡯' +
  '⠰⠱⠲⠳⠴⠵⠶⠷⡰⡱⡲⡳⡴⡵⡶⡷⠸⠹⠺⠻⠼⠽⠾⠿⡸⡹⡺⡻⡼⡽⡾⡿' +
  '⢀⢁⢂⢃⢄⢅⢆⢇⣀⣁⣂⣃⣄⣅⣆⣇⢈⢉⢊⢋⢌⢍⢎⢏⣈⣉⣊⣋⣌⣍⣎⣏' +
  '⢐⢑⢒⢓⢔⢕⢖⢗⣐⣑⣒⣓⣔⣕⣖⣗⢘⢙⢚⢛⢜⢝⢞⢟⣘⣙⣚⣛⣜⣝⣞⣟' +
  '⢠⢡⢢⢣⢤⢥⢦⢧⣠⣡⣢⣣⣤⣥⣦⣧⢨⢩⢪⢫⢬⢭⢮⢯⣨⣩⣪⣫⣬⣭⣮⣯' +
  '⢰⢱⢲⢳⢴⢵⢶⢷⣰⣱⣲⣳⣴⣵⣶⣷⢸⢹⢺⢻⢼⢽⢾⢿⣸⣹⣺⣻⣼⣽⣾⣿'
```

### Script / Language Charsets

```typescript
const CHARSET_KATAKANA = ' ·ｦｧｨｩｪｫｬｭｮｯｰｱｲｳｴｵｶｷ'
const CHARSET_RUNE     = ' .ᚠᚢᚦᛒᛗᛡᛇᛞᛟᛚᛞᛟ'
const CHARSET_GREEK    = ' αβγδεζηθικλμνξπρστφψω'
const CHARSET_HIRAGANA = ' ぁあぃいぅうぇえぉおかきくけこさしすせそ'
const CHARSET_CYRILLIC = ' ·абвгдежзийклмнопрстуфхцчшщъыьэюя'
const CHARSET_ARABIC   = ' ·ءابتثجحخدذرزسشصضطظعغفقكلمنهوي'
const CHARSET_HEBREW   = ' ·אבגדהוזחטיכלמנסעפצקרשת'
const CHARSET_THAI     = ' ·กขคงจฉชซญดตถทนบปพฟมยรลวศสหอ'
const CHARSET_DEVANAGARI = ' ·अआइईउऊएऐओऔकखगघचछजझटठडढणतथदधनपफबभम'
```

### Mathematical / Technical Charsets

```typescript
const CHARSET_MATH    = ' ·∘∙•°±∓×÷≈≠≡≤≥∞∫∑∏√∇∂∆Ω'
const CHARSET_BOX     = ' ─│┌┐└┘├┤┬┴┼═║╔╗╚╝╠╣╦╩╬'
const CHARSET_CIRCUIT = ' .·─│┌┐└┘┼○●□■△▽≡'
const CHARSET_HEX     = ' 0123456789abcdef'
const CHARSET_LOGIC   = ' ·∧∨¬⊕⊗⊞⊟∀∃∄∈∉⊂⊃⊆⊇∪∩'
const CHARSET_MUSIC   = ' ·♩♪♫♬♭♮♯𝄞𝄢'
```

### Symbolic / Decorative Charsets

```typescript
const CHARSET_ARROWS     = ' ←↑→↓↔↕↖↗↘↙↩↪↻➡'
const CHARSET_DOTS       = ' ⋅∘∙●◉◎◆✦★'
const CHARSET_STARS      = ' ·✧✦✩✨★✶✳✸'
const CHARSET_GEOMETRIC  = ' ·∘○◎●◉⬤'
const CHARSET_CARDS      = ' ·♠♣♥♦♤♧♡♢'
const CHARSET_CHESS      = ' ·♙♘♗♖♕♔♟♞♝♜♛♚'
const CHARSET_ZODIAC     = ' ·♈♉♊♋♌♍♎♏♐♑♒♓'
const CHARSET_WEATHER    = ' ·☁☂☃☀☼⚡❄'
const CHARSET_EMOJI_MOON = ' 🌑🌒🌓🌔🌕'
const CHARSET_CROSSES    = ' ·✙✚✛✜✝✞✟†‡'
const CHARSET_SNOWFLAKE  = ' ·❅❆❄✻✼❉✾'
```

### Terminal Charsets

```typescript
const CHARSET_TERMINAL_101010   = ' 01'
const CHARSET_TERMINAL_BRACKETS = ' ()[]{}><'
const CHARSET_TERMINAL_DOLLAR   = ' $€£¥₿'
const CHARSET_TERMINAL_MIXED    = ' .·:;|/\\-=+*#@$%&'
const CHARSET_TERMINAL_PIPES    = ' ─│┌┐└┘├┤┬┴┼'
```

### Block / Structural Charsets

```typescript
const CHARSET_BLOCKS_EXTENDED = ' ░▒▓█▄▀▐▌'
const CHARSET_SHADE_FINE      = ' ░░▒▒▓▓██'
const CHARSET_VERTICAL_BARS   = ' ▁▂▃▄▅▆▇█'
const CHARSET_HORIZONTAL_BARS = ' ▏▎▍▌▋▊▉█'
```

### DotCross Ramps

```typescript
const DOT_RAMP   = '  .·:oO'
const CROSS_RAMP = '  ·+xX#'
```

### Complete Palette Registry

```typescript
const CHARSETS: Record<string, string> = {
  // Core
  'standard':          CHARSET_STANDARD,
  'blocks':            CHARSET_BLOCKS,
  'detailed':          CHARSET_DETAILED,
  'minimal':           CHARSET_MINIMAL,
  'binary':            CHARSET_BINARY,
  'dense':             CHARSET_DENSE,
  'default':           CHARSET_DEFAULT,

  // Letters
  'letters-upper':     CHARSET_LETTERS_UPPER,
  'letters-lower':     CHARSET_LETTERS_LOWER,
  'letters-mixed':     CHARSET_LETTERS_MIXED,
  'symbols':           CHARSET_SYMBOLS,

  // Braille
  'braille':           CHARSET_BRAILLE_STANDARD,
  'braille-sparse':    CHARSET_BRAILLE_SPARSE,
  'braille-dense':     CHARSET_BRAILLE_DENSE,
  'braille-full':      CHARSET_BRAILLE_FULL,

  // Script / Language
  'katakana':          CHARSET_KATAKANA,
  'rune':              CHARSET_RUNE,
  'greek':             CHARSET_GREEK,
  'hiragana':          CHARSET_HIRAGANA,
  'cyrillic':          CHARSET_CYRILLIC,
  'arabic':            CHARSET_ARABIC,
  'hebrew':            CHARSET_HEBREW,
  'thai':              CHARSET_THAI,
  'devanagari':        CHARSET_DEVANAGARI,

  // Mathematical / Technical
  'math':              CHARSET_MATH,
  'box':               CHARSET_BOX,
  'circuit':           CHARSET_CIRCUIT,
  'hex':               CHARSET_HEX,
  'logic':             CHARSET_LOGIC,
  'music':             CHARSET_MUSIC,

  // Symbolic / Decorative
  'arrows':            CHARSET_ARROWS,
  'dots':              CHARSET_DOTS,
  'stars':             CHARSET_STARS,
  'geometric':         CHARSET_GEOMETRIC,
  'cards':             CHARSET_CARDS,
  'chess':             CHARSET_CHESS,
  'zodiac':            CHARSET_ZODIAC,
  'weather':           CHARSET_WEATHER,
  'crosses':           CHARSET_CROSSES,
  'snowflake':         CHARSET_SNOWFLAKE,

  // Terminal
  'terminal-binary':   CHARSET_TERMINAL_101010,
  'terminal-brackets': CHARSET_TERMINAL_BRACKETS,
  'terminal-dollar':   CHARSET_TERMINAL_DOLLAR,
  'terminal-mixed':    CHARSET_TERMINAL_MIXED,
  'terminal-pipes':    CHARSET_TERMINAL_PIPES,

  // Block / Structural
  'blocks-extended':   CHARSET_BLOCKS_EXTENDED,
  'shade-fine':        CHARSET_SHADE_FINE,
  'vertical-bars':     CHARSET_VERTICAL_BARS,
  'horizontal-bars':   CHARSET_HORIZONTAL_BARS,

  // DotCross
  'dot-ramp':          DOT_RAMP,
  'cross-ramp':        CROSS_RAMP,
}
```

---

## 3. Braille Sub-Cell Rendering

Each Unicode Braille character (U+2800 to U+28FF) encodes 8 dots in a 2-column x 4-row grid. The 256 possible patterns map one-to-one to the 256 codepoints. This means a single Braille character can represent a 2x4 pixel block, giving 4x the vertical resolution and 2x the horizontal resolution of normal character-based rendering.

### Braille Dot Layout

The 8 dots in each Braille character cell are numbered and mapped to bit positions:

```
Column:  0   1
Row 0:  d1  d4    bit 0  bit 3
Row 1:  d2  d5    bit 1  bit 4
Row 2:  d3  d6    bit 2  bit 5
Row 3:  d7  d8    bit 6  bit 7
```

The Unicode codepoint is `0x2800 + (bit pattern)`. Dot 1 raised = bit 0 set = U+2801 (⠁). Dot 4 raised = bit 3 set = U+2808 (⠈). All dots raised = bits 0-7 all set = U+28FF (⣿).

### Dot-to-Bit Mapping

```typescript
// BRAILLE_DOT_MAP[row][col] gives the bit position for that dot
const BRAILLE_DOT_MAP: number[][] = [
  [0, 3],  // row 0: dot 1 (bit 0), dot 4 (bit 3)
  [1, 4],  // row 1: dot 2 (bit 1), dot 5 (bit 4)
  [2, 5],  // row 2: dot 3 (bit 2), dot 6 (bit 5)
  [6, 7],  // row 3: dot 7 (bit 6), dot 8 (bit 7)
]
```

### Brightness Sub-Grid to Braille Character

Given a 2x4 block of brightness values (each 0.0 to 1.0), compute the exact Braille character by thresholding each sub-pixel and setting the corresponding bit.

```typescript
/**
 * Convert a 2x4 brightness sub-grid into a single Braille character.
 * Each cell in the sub-grid is compared against `threshold` to determine
 * whether that dot is raised.
 *
 * @param subGrid - 4 rows x 2 cols of brightness values [0,1].
 *                  subGrid[row][col] where row 0..3, col 0..1
 * @param threshold - brightness cutoff for raising a dot (default 0.5)
 * @returns single Braille character U+2800..U+28FF
 */
function brightnessToBraille(
  subGrid: number[][],
  threshold: number = 0.5,
): string {
  let bits = 0

  for (let row = 0; row < 4; row++) {
    for (let col = 0; col < 2; col++) {
      if (subGrid[row][col] >= threshold) {
        bits |= 1 << BRAILLE_DOT_MAP[row][col]
      }
    }
  }

  return String.fromCharCode(0x2800 + bits)
}
```

### Full Braille Sub-Cell Renderer

Renders an entire brightness grid at 2x4 sub-cell resolution. The source brightness grid is at the sub-pixel level (2x wider, 4x taller than the output character grid). Each Braille character in the output represents an 8-pixel block from the source.

```typescript
interface BrailleSubCellConfig {
  threshold: number   // dot activation threshold, default 0.5
  adaptive: boolean   // use local mean as threshold instead of fixed
}

/**
 * Render a high-resolution brightness grid as Braille sub-cell characters.
 *
 * @param pixels - brightness grid at sub-pixel resolution (width x height)
 * @param pixelW - width of brightness grid (must be even)
 * @param pixelH - height of brightness grid (must be divisible by 4)
 * @param config - rendering options
 * @returns 2D array of Braille characters [rows][cols]
 *          where cols = pixelW / 2, rows = pixelH / 4
 */
function renderBrailleSubCell(
  pixels: Float32Array,
  pixelW: number,
  pixelH: number,
  config: BrailleSubCellConfig = { threshold: 0.5, adaptive: false },
): string[][] {
  const charCols = Math.floor(pixelW / 2)
  const charRows = Math.floor(pixelH / 4)
  const output: string[][] = []

  for (let cy = 0; cy < charRows; cy++) {
    const row: string[] = []
    for (let cx = 0; cx < charCols; cx++) {
      const px = cx * 2
      const py = cy * 4

      // Extract the 2x4 sub-grid
      const subGrid: number[][] = []
      let sum = 0
      for (let r = 0; r < 4; r++) {
        const subRow: number[] = []
        for (let c = 0; c < 2; c++) {
          const val = pixels[(py + r) * pixelW + (px + c)]
          subRow.push(val)
          sum += val
        }
        subGrid.push(subRow)
      }

      // Adaptive threshold: use local mean of the 8 sub-pixels
      const thresh = config.adaptive ? sum / 8 : config.threshold

      row.push(brightnessToBraille(subGrid, thresh))
    }
    output.push(row)
  }

  return output
}
```

### Braille Sub-Cell Canvas Renderer

Complete canvas renderer that samples source image data at sub-cell resolution and outputs Braille characters.

```typescript
function renderBrailleSubCellToCanvas(
  ctx: CanvasRenderingContext2D,
  brightnessGrid: Float32Array,
  gridW: number,
  gridH: number,
  cellWidth: number,
  cellHeight: number,
  fontSize: number,
  config: BrailleSubCellConfig & { color: string },
): void {
  // The input brightness grid is at sub-pixel resolution
  // Output character grid is gridW/2 cols x gridH/4 rows
  const charCols = Math.floor(gridW / 2)
  const charRows = Math.floor(gridH / 4)

  ctx.font = `${fontSize}px "Courier New", "Consolas", monospace`
  ctx.textBaseline = 'top'
  ctx.textAlign = 'left'
  ctx.fillStyle = config.color

  // Each output character covers 2 sub-pixels wide, 4 sub-pixels tall
  const outCellW = cellWidth * 2
  const outCellH = cellHeight * 4

  for (let cy = 0; cy < charRows; cy++) {
    for (let cx = 0; cx < charCols; cx++) {
      const px = cx * 2
      const py = cy * 4

      let bits = 0
      let sum = 0

      for (let r = 0; r < 4; r++) {
        for (let c = 0; c < 2; c++) {
          const val = brightnessGrid[(py + r) * gridW + (px + c)]
          sum += val
        }
      }

      const thresh = config.adaptive ? sum / 8 : config.threshold

      for (let r = 0; r < 4; r++) {
        for (let c = 0; c < 2; c++) {
          if (brightnessGrid[(py + r) * gridW + (px + c)] >= thresh) {
            bits |= 1 << BRAILLE_DOT_MAP[r][c]
          }
        }
      }

      if (bits === 0) continue

      const char = String.fromCharCode(0x2800 + bits)
      ctx.fillText(char, cx * outCellW, cy * outCellH)
    }
  }
}
```

### Dithered Braille Sub-Cell Rendering

Combine error-diffusion dithering with sub-cell rendering for the highest quality output. Apply dithering at the sub-pixel level before converting to Braille dots.

```typescript
function renderDitheredBrailleSubCell(
  pixels: Float32Array,
  pixelW: number,
  pixelH: number,
  ditherAlgorithm: 'floyd-steinberg' | 'atkinson' | 'bayer' | 'none',
  ditherStrength: number,
): string[][] {
  // Apply dithering at sub-pixel resolution
  let dithered: Float32Array

  if (ditherAlgorithm === 'none' || ditherStrength <= 0) {
    dithered = pixels
  } else if (ditherAlgorithm === 'bayer') {
    dithered = bayerDither8x8(pixels, pixelW, pixelH, ditherStrength)
  } else {
    const kernel = DIFFUSION_KERNELS[ditherAlgorithm]
    dithered = kernel
      ? errorDiffusion(pixels, pixelW, pixelH, ditherStrength, kernel)
      : pixels
  }

  // Convert dithered result to Braille with fixed 0.5 threshold
  // (dithering already handled the tonal mapping)
  return renderBrailleSubCell(dithered, pixelW, pixelH, {
    threshold: 0.5,
    adaptive: false,
  })
}
```

### Braille Resolution Comparison

| Technique | Effective Resolution per Character | Use Case |
|-----------|-----------------------------------|----------|
| Standard char mapping | 1x1 | General ASCII art |
| Braille brightness ramp | 1x1 (64 or 256 tone levels) | Dense dot-matrix aesthetic |
| Braille sub-cell | 2x4 (8 sub-pixels) | Ultra-detail, image conversion |

Sub-cell rendering at font size 10 in a 1920x1080 canvas yields approximately 160 columns x 108 rows of Braille characters, but with an effective pixel resolution of 320x432 -- nearly matching the source image detail.

---

## 4. Edge-Directed Character Selection

Use Sobel gradient direction to pick directional characters for edge regions. This produces strokes that follow the contour direction of edges in the source image.

### Sobel Gradient Computation

```typescript
interface SobelResult {
  magnitude: number  // edge strength 0-1
  angle: number      // gradient direction in radians (-PI to PI)
}

function sobelAt(
  grid: Float32Array,
  x: number,
  y: number,
  cols: number,
  rows: number,
): SobelResult {
  // Sample 3x3 neighborhood with clamped boundary
  const get = (gx: number, gy: number): number => {
    const cx = Math.max(0, Math.min(cols - 1, gx))
    const cy = Math.max(0, Math.min(rows - 1, gy))
    return grid[cy * cols + cx]
  }

  const tl = get(x - 1, y - 1), tc = get(x, y - 1), tr = get(x + 1, y - 1)
  const ml = get(x - 1, y),                           mr = get(x + 1, y)
  const bl = get(x - 1, y + 1), bc = get(x, y + 1), br = get(x + 1, y + 1)

  // Sobel kernels
  const gx = (-tl + tr) + (-2 * ml + 2 * mr) + (-bl + br)
  const gy = (-tl - 2 * tc - tr) + (bl + 2 * bc + br)

  const magnitude = Math.min(1, Math.sqrt(gx * gx + gy * gy))
  const angle = Math.atan2(gy, gx)

  return { magnitude, angle }
}
```

### Directional Character Map

Map the gradient angle to directional characters. The gradient points perpendicular to the edge, so rotate 90 degrees to get the edge direction.

```typescript
const EDGE_CHARS: Record<string, string> = {
  horizontal:     '-',
  vertical:       '|',
  diag_forward:   '/',
  diag_backward:  '\\',
  corner_tl:      '/',
  corner_tr:      '\\',
  corner_bl:      '\\',
  corner_br:      '/',
  cross:          '+',
  dot:            '.',
}

/**
 * Pick a directional character based on Sobel gradient angle.
 * The gradient angle points perpendicular to the edge -- we rotate
 * 90 degrees to align the character with the edge direction.
 *
 * 8 angular sectors, 45 degrees each:
 *   0: horizontal edge    (gradient points up/down)
 *  45: forward diagonal   (gradient points upper-right/lower-left)
 *  90: vertical edge      (gradient points left/right)
 * 135: backward diagonal  (gradient points upper-left/lower-right)
 */
function getEdgeChar(angle: number): string {
  // Normalize angle to 0-PI (edges are symmetric)
  let a = ((angle % Math.PI) + Math.PI) % Math.PI

  // 8 sectors of 22.5 degrees each, but we group into 4 edge directions
  if (a < Math.PI * 0.125 || a >= Math.PI * 0.875) {
    return '-'     // nearly horizontal edge
  } else if (a < Math.PI * 0.375) {
    return '\\'    // backward diagonal edge
  } else if (a < Math.PI * 0.625) {
    return '|'     // nearly vertical edge
  } else {
    return '/'     // forward diagonal edge
  }
}
```

### Extended Directional Character Set

For richer edge rendering with Unicode box-drawing characters:

```typescript
function getEdgeCharExtended(angle: number, magnitude: number): string {
  let a = ((angle % Math.PI) + Math.PI) % Math.PI

  // Thin edges (low magnitude) get lighter characters
  if (magnitude < 0.3) {
    if (a < Math.PI * 0.125 || a >= Math.PI * 0.875) return '─'
    if (a < Math.PI * 0.375) return '╲'
    if (a < Math.PI * 0.625) return '│'
    return '╱'
  }

  // Strong edges get heavier characters
  if (a < Math.PI * 0.125 || a >= Math.PI * 0.875) return '━'
  if (a < Math.PI * 0.375) return '╲'
  if (a < Math.PI * 0.625) return '┃'
  return '╱'
}
```

### Edge-Directed Renderer

Complete renderer that blends standard brightness-mapped characters in smooth areas with directional characters at edges.

```typescript
interface EdgeDirectedConfig {
  edgeThreshold: number    // minimum Sobel magnitude to use edge chars, default 0.15
  edgeBlend: number        // 0 = only brightness chars, 1 = only edge chars at edges, default 0.7
  edgeCharsExtended: boolean // use box-drawing characters, default false
}

function renderEdgeDirected(
  ctx: CanvasRenderingContext2D,
  brightnessGrid: Float32Array,
  colorGrid: Array<[number, number, number]>,
  cols: number,
  rows: number,
  cellWidth: number,
  cellHeight: number,
  fontSize: number,
  chars: string,
  config: EdgeDirectedConfig,
): void {
  ctx.font = `${fontSize}px "Courier New", "Consolas", monospace`
  ctx.textBaseline = 'top'
  ctx.textAlign = 'left'

  for (let y = 0; y < rows; y++) {
    for (let x = 0; x < cols; x++) {
      const idx = y * cols + x
      const b = brightnessGrid[idx]

      if (b < 0.01) continue

      const { magnitude, angle } = sobelAt(brightnessGrid, x, y, cols, rows)

      let char: string
      if (magnitude >= config.edgeThreshold) {
        // Edge region: use directional character
        char = config.edgeCharsExtended
          ? getEdgeCharExtended(angle, magnitude)
          : getEdgeChar(angle)
      } else {
        // Smooth region: use brightness-mapped character
        char = getCharForBrightness(b, chars)
      }

      if (char === ' ') continue

      const [r, g, bl] = colorGrid[idx]
      const gray = Math.round(b * 255)
      ctx.fillStyle = `rgb(${gray},${gray},${gray})`
      ctx.fillText(char, x * cellWidth, y * cellHeight)
    }
  }
}
```

---

## 5. All 9 Art Style Renderers

### Shared Types and Utilities

```typescript
interface RenderContext {
  ctx: CanvasRenderingContext2D
  brightnessGrid: Float32Array
  colorGrid: Array<[number, number, number]>
  cols: number
  rows: number
  cellWidth: number
  cellHeight: number
  time: number
  mouseX: number  // normalized 0-1, -1 if no mouse
  mouseY: number  // normalized 0-1, -1 if no mouse
  chars: string
  colorMode: ColorMode
  customColor?: string
  fontSize: number
  invert: boolean
  vignetteStrength: number
  edgeBoost: number
  brightness: number
  contrast: number
}

type ColorMode =
  | 'grayscale' | 'fullcolor' | 'matrix' | 'amber'
  | 'sepia' | 'cool-blue' | 'neon' | 'custom'

// Screen interference pattern -- used by Halftone and Braille
function getScreenLevel(x: number, y: number): number {
  return (
    Math.sin(x * 0.82 + y * 0.33) * 1.55 +
    Math.cos(x * 0.27 - y * 0.94) * 1.25 +
    2
  ) * 0.25
}

// Local edge contrast (simplified Sobel magnitude)
function getLocalEdgeContrast(
  grid: Float32Array, x: number, y: number, cols: number, rows: number,
): number {
  const idx = y * cols + x
  const left  = x > 0        ? grid[idx - 1]    : grid[idx]
  const right = x < cols - 1  ? grid[idx + 1]    : grid[idx]
  const up    = y > 0        ? grid[idx - cols]  : grid[idx]
  const down  = y < rows - 1  ? grid[idx + cols]  : grid[idx]
  const gx = Math.abs(right - left)
  const gy = Math.abs(down - up)
  return Math.min(1, Math.sqrt(gx * gx + gy * gy))
}

// Vignette factor: 1.0 at center, decreasing toward edges
function getVignetteFactor(
  x: number, y: number, cols: number, rows: number, strength: number,
): number {
  if (strength <= 0) return 1.0
  const nx = (x / cols) * 2 - 1
  const ny = (y / rows) * 2 - 1
  const dist = Math.sqrt(nx * nx + ny * ny) / Math.SQRT2
  return Math.max(0, Math.min(1, 1 - dist * dist * strength))
}

// Regular polygon helper (for Halftone shapes)
function drawRegularPolygon(
  ctx: CanvasRenderingContext2D,
  cx: number, cy: number, radius: number, sides: number, rotation: number,
): void {
  if (sides < 3 || radius <= 0) return
  ctx.beginPath()
  for (let i = 0; i <= sides; i++) {
    const angle = (i / sides) * Math.PI * 2 + rotation
    const px = cx + Math.cos(angle) * radius
    const py = cy + Math.sin(angle) * radius
    if (i === 0) ctx.moveTo(px, py)
    else ctx.lineTo(px, py)
  }
  ctx.closePath()
  ctx.fill()
}
```

### Style 1: Classic

Brightness-to-character mapping with mouse-interactive offset.

```typescript
function renderClassic(rc: RenderContext): void {
  const {
    ctx, brightnessGrid, colorGrid, cols, rows,
    cellWidth, cellHeight, time, mouseX, mouseY,
    chars, colorMode, customColor, fontSize,
    invert, vignetteStrength, brightness, contrast, edgeBoost,
  } = rc

  ctx.font = `${fontSize}px "Courier New", "Consolas", monospace`
  ctx.textBaseline = 'top'
  ctx.textAlign = 'left'

  for (let y = 0; y < rows; y++) {
    for (let x = 0; x < cols; x++) {
      const idx = y * cols + x
      let b = brightnessGrid[idx]
      b = adjustBrightness(b, brightness, contrast)

      if (vignetteStrength > 0) {
        b *= getVignetteFactor(x, y, cols, rows, vignetteStrength)
      }
      if (edgeBoost > 0) {
        const edge = getLocalEdgeContrast(brightnessGrid, x, y, cols, rows)
        b = Math.min(1, b + edge * edgeBoost)
      }
      if (invert) b = 1 - b

      const char = getCharForBrightness(b, chars)
      if (char === ' ') continue

      const [r, g, bl] = colorGrid[idx]
      const gray = Math.round(b * 255)
      const color = getColorForMode(colorMode, {
        r, g, b: bl, gray, brightness: b, customColor,
      })

      let ox = 0, oy = 0
      if (mouseX >= 0 && mouseY >= 0) {
        const cx = (x + 0.5) / cols
        const cy = (y + 0.5) / rows
        const dx = mouseX - cx
        const dy = mouseY - cy
        const dist = Math.sqrt(dx * dx + dy * dy)
        const falloff = Math.max(0, 1 - dist * 4)
        const strength = falloff * cellWidth * 0.3
        ox = dx * strength
        oy = dy * strength
      }

      ctx.fillStyle = color
      ctx.fillText(char, x * cellWidth + ox, y * cellHeight + oy)
    }
  }
}
```

### Style 2: Halftone

Geometric shapes sized by brightness, with screen interference pattern.

```typescript
interface HalftoneConfig {
  halftoneShape: 'circle' | 'square' | 'diamond' | 'pentagon' | 'hexagon'
  halftoneSize: number      // 0.4 to 2.2, default 1.0
  halftoneRotation: number  // degrees, default 0
}

function renderHalftone(rc: RenderContext, config: HalftoneConfig): void {
  const {
    ctx, brightnessGrid, colorGrid, cols, rows,
    cellWidth, cellHeight, colorMode, customColor,
    invert, vignetteStrength, brightness, contrast,
  } = rc
  const { halftoneShape, halftoneSize, halftoneRotation } = config
  const rotRad = (halftoneRotation * Math.PI) / 180

  for (let y = 0; y < rows; y++) {
    for (let x = 0; x < cols; x++) {
      const idx = y * cols + x
      let b = brightnessGrid[idx]
      b = adjustBrightness(b, brightness, contrast)

      if (vignetteStrength > 0) {
        b *= getVignetteFactor(x, y, cols, rows, vignetteStrength)
      }
      if (invert) b = 1 - b

      const screen = getScreenLevel(x, y)
      const dotLevel = b * screen
      if (dotLevel < 0.02) continue

      const cx = x * cellWidth + cellWidth * 0.5
      const cy = y * cellHeight + cellHeight * 0.5
      const maxRadius = Math.min(cellWidth, cellHeight) * 0.5 * halftoneSize
      const radius = maxRadius * Math.sqrt(dotLevel)
      if (radius < 0.5) continue

      const [r, g, bl] = colorGrid[idx]
      const gray = Math.round(b * 255)
      const color = getColorForMode(colorMode, {
        r, g, b: bl, gray, brightness: b, customColor,
      })
      ctx.fillStyle = color

      switch (halftoneShape) {
        case 'circle':
          ctx.beginPath()
          ctx.arc(cx, cy, radius, 0, Math.PI * 2)
          ctx.fill()
          break
        case 'square':
          ctx.save()
          ctx.translate(cx, cy)
          ctx.rotate(rotRad)
          const side = radius * 2
          ctx.fillRect(-side * 0.5, -side * 0.5, side, side)
          ctx.restore()
          break
        case 'diamond':
          drawRegularPolygon(ctx, cx, cy, radius, 4, Math.PI / 4 + rotRad)
          break
        case 'pentagon':
          drawRegularPolygon(ctx, cx, cy, radius, 5, -Math.PI / 2 + rotRad)
          break
        case 'hexagon':
          drawRegularPolygon(ctx, cx, cy, radius, 6, rotRad)
          break
      }
    }
  }
}
```

### Style 3: Braille

Braille patterns with edge-contrast boost and screen pattern modulation.

```typescript
const BRAILLE_CHARS =
  '⠀⠁⠂⠃⠄⠅⠆⠇⠈⠉⠊⠋⠌⠍⠎⠏' +
  '⠐⠑⠒⠓⠔⠕⠖⠗⠘⠙⠚⠛⠜⠝⠞⠟' +
  '⠠⠡⠢⠣⠤⠥⠦⠧⠨⠩⠪⠫⠬⠭⠮⠯' +
  '⠰⠱⠲⠳⠴⠵⠶⠷⠸⠹⠺⠻⠼⠽⠾⠿'

type BrailleVariant = 'standard' | 'dense' | 'sparse'

function renderBraille(rc: RenderContext, variant: BrailleVariant = 'standard'): void {
  const {
    ctx, brightnessGrid, colorGrid, cols, rows,
    cellWidth, cellHeight, colorMode, customColor,
    fontSize, invert, vignetteStrength, brightness, contrast,
  } = rc

  ctx.font = `${fontSize}px "Courier New", "Consolas", monospace`
  ctx.textBaseline = 'top'
  ctx.textAlign = 'left'

  const variantBias = variant === 'dense' ? 0.11 : variant === 'sparse' ? -0.08 : 0

  for (let y = 0; y < rows; y++) {
    for (let x = 0; x < cols; x++) {
      const idx = y * cols + x
      let b = brightnessGrid[idx]
      b = adjustBrightness(b, brightness, contrast)

      if (vignetteStrength > 0) {
        b *= getVignetteFactor(x, y, cols, rows, vignetteStrength)
      }

      const edge = getLocalEdgeContrast(brightnessGrid, x, y, cols, rows)
      b = Math.min(1, b + edge * 0.15)
      b = Math.max(0, Math.min(1, b + variantBias))

      if (invert) b = 1 - b

      const screen = getScreenLevel(x, y)
      const concentration = 0.5 + Math.sin(x * 0.4 + y * 0.6) * 0.15
      const adjusted = b * screen * (0.6 + concentration * 0.8)
      const charIdx = Math.max(0, Math.min(63, Math.floor(adjusted * 63)))
      const char = BRAILLE_CHARS[charIdx]

      if (char === '⠀') continue

      const [r, g, bl] = colorGrid[idx]
      const gray = Math.round(b * 255)
      const color = getColorForMode(colorMode, {
        r, g, b: bl, gray, brightness: b, customColor,
      })

      ctx.fillStyle = color
      ctx.fillText(char, x * cellWidth, y * cellHeight)
    }
  }
}
```

### Style 4: DotCross

Dual character ramps -- dots for smooth areas, crosses for edges.

```typescript
function renderDotCross(rc: RenderContext): void {
  const {
    ctx, brightnessGrid, colorGrid, cols, rows,
    cellWidth, cellHeight, colorMode, customColor,
    fontSize, invert, vignetteStrength, brightness, contrast,
  } = rc

  ctx.font = `${fontSize}px "Courier New", "Consolas", monospace`
  ctx.textBaseline = 'top'
  ctx.textAlign = 'left'

  for (let y = 0; y < rows; y++) {
    for (let x = 0; x < cols; x++) {
      const idx = y * cols + x
      let b = brightnessGrid[idx]
      b = adjustBrightness(b, brightness, contrast)

      if (vignetteStrength > 0) {
        b *= getVignetteFactor(x, y, cols, rows, vignetteStrength)
      }
      if (invert) b = 1 - b

      const edge = getLocalEdgeContrast(brightnessGrid, x, y, cols, rows)
      const weave = (Math.sin(x * 1.2 + y * 0.8) * 0.5 + 0.5) * 0.3
      const crossWeight = Math.min(1, edge * 2.5 + weave)
      const charIndex = Math.max(0, Math.min(6, Math.floor(b * 6)))

      const char = crossWeight > 0.5 ? CROSS_RAMP[charIndex] : DOT_RAMP[charIndex]
      if (char === ' ') continue

      const [r, g, bl] = colorGrid[idx]
      const gray = Math.round(b * 255)
      const color = getColorForMode(colorMode, {
        r, g, b: bl, gray, brightness: b, customColor,
      })

      ctx.fillStyle = color
      ctx.fillText(char, x * cellWidth, y * cellHeight)
    }
  }
}
```

### Style 5: Line

Vectorized line segments with screen-pattern rotation and brightness-proportional length.

```typescript
interface LineConfig {
  lineLength: number     // 0.1 to 2.5, default 1.0
  lineWidth: number      // 0.2 to 2.5, default 1.0
  lineThickness: number  // 0.2 to 8, default 1.5
  lineRotation: number   // degrees, default 0
}

function renderLine(rc: RenderContext, config: LineConfig): void {
  const {
    ctx, brightnessGrid, colorGrid, cols, rows,
    cellWidth, cellHeight, colorMode, customColor,
    invert, vignetteStrength, brightness, contrast,
  } = rc
  const { lineLength, lineWidth, lineThickness, lineRotation } = config
  const baseRotRad = (lineRotation * Math.PI) / 180

  ctx.lineCap = 'round'

  for (let y = 0; y < rows; y++) {
    for (let x = 0; x < cols; x++) {
      const idx = y * cols + x
      let b = brightnessGrid[idx]
      b = adjustBrightness(b, brightness, contrast)

      if (vignetteStrength > 0) {
        b *= getVignetteFactor(x, y, cols, rows, vignetteStrength)
      }
      if (invert) b = 1 - b
      if (b < 0.02) continue

      const cx = x * cellWidth + cellWidth * 0.5
      const cy = y * cellHeight + cellHeight * 0.5

      const screenAngle = Math.sin(x * 0.7 + y * 0.5) * 0.8 +
                          Math.cos(x * 0.3 - y * 0.9) * 0.6
      const angle = baseRotRad + screenAngle
      const halfLen = b * Math.min(cellWidth, cellHeight) * 0.5 * lineLength

      const dx = Math.cos(angle) * halfLen
      const dy = Math.sin(angle) * halfLen

      const [r, g, bl] = colorGrid[idx]
      const gray = Math.round(b * 255)
      const color = getColorForMode(colorMode, {
        r, g, b: bl, gray, brightness: b, customColor,
      })

      ctx.strokeStyle = color
      ctx.lineWidth = lineThickness * b

      ctx.beginPath()
      ctx.moveTo(cx - dx, cy - dy)
      ctx.lineTo(cx + dx, cy + dy)
      ctx.stroke()
    }
  }
}
```

### Style 6: Particles

Threshold-based radiant particle spray with center bias, edge glow, and pseudo-random grain.

```typescript
interface ParticleConfig {
  particleDensity: number  // 0.05 to 1, default 0.5
  particleChar: string     // default '*'
}

function renderParticles(rc: RenderContext, config: ParticleConfig): void {
  const {
    ctx, brightnessGrid, colorGrid, cols, rows,
    cellWidth, cellHeight, time, colorMode, customColor,
    fontSize, invert, vignetteStrength, brightness, contrast,
  } = rc
  const { particleDensity, particleChar } = config

  ctx.font = `${fontSize}px "Courier New", "Consolas", monospace`
  ctx.textBaseline = 'top'
  ctx.textAlign = 'left'

  for (let y = 0; y < rows; y++) {
    for (let x = 0; x < cols; x++) {
      const idx = y * cols + x
      let b = brightnessGrid[idx]
      b = adjustBrightness(b, brightness, contrast)

      if (vignetteStrength > 0) {
        b *= getVignetteFactor(x, y, cols, rows, vignetteStrength)
      }
      if (invert) b = 1 - b

      const nx = (x / cols) * 2 - 1
      const ny = (y / rows) * 2 - 1
      const centerBias = 1 - Math.sqrt(nx * nx + ny * ny) * 0.5

      const edge = getLocalEdgeContrast(brightnessGrid, x, y, cols, rows)
      const grain = ((x * 2654435761 + y * 340573321 + Math.floor(time * 30)) & 0xffff) / 0xffff

      const threshold = centerBias * 0.3 + edge * 0.25 + b * 0.35 + grain * 0.1
      if (threshold < (1 - particleDensity)) continue

      const [r, g, bl] = colorGrid[idx]
      const glowBoost = centerBias * 0.15 + edge * 0.2
      const boostedB = Math.min(1, b + glowBoost)
      const boostedGray = Math.round(boostedB * 255)

      const color = getColorForMode(colorMode, {
        r: Math.min(255, r + Math.round(glowBoost * 60)),
        g: Math.min(255, g + Math.round(glowBoost * 60)),
        b: Math.min(255, bl + Math.round(glowBoost * 60)),
        gray: boostedGray,
        brightness: boostedB,
        customColor,
      })

      ctx.fillStyle = color
      ctx.fillText(particleChar, x * cellWidth, y * cellHeight)
    }
  }
}
```

### Style 7: Retro

Duotone palettes with grain, posterization, and gamma correction.

```typescript
interface DuotonePalette {
  low: [number, number, number]
  high: [number, number, number]
}

const RETRO_PALETTES: Record<string, DuotonePalette> = {
  'amber-classic': { low: [20, 12, 6],   high: [255, 223, 178] },
  'cyan-night':    { low: [6, 16, 22],   high: [166, 240, 255] },
  'violet-haze':   { low: [17, 10, 26],  high: [242, 198, 255] },
  'lime-pulse':    { low: [10, 18, 8],   high: [226, 255, 162] },
  'mono-ice':      { low: [12, 12, 12],  high: [245, 248, 255] },
}

interface RetroConfig {
  retroNoise: number     // 0 to 1, default 0.15
  retroDuotone: string   // palette name, default 'amber-classic'
}

function getRetroColor(
  brightness: number,
  paletteName: string,
  x: number, y: number,
  cols: number, rows: number,
  time: number,
  options?: {
    vignetteStrength?: number
    grainAmount?: number
    shimmerSpeed?: number
  },
): string {
  const palette = RETRO_PALETTES[paletteName]
  if (!palette) return `rgb(${Math.round(brightness * 255)},${Math.round(brightness * 255)},${Math.round(brightness * 255)})`

  const vigStr = options?.vignetteStrength ?? 0.3
  const grainAmt = options?.grainAmount ?? 0.05
  const shimSpd = options?.shimmerSpeed ?? 1.0

  let b = Math.max(0, Math.min(1, brightness))
  b *= getVignetteFactor(x, y, cols, rows, vigStr)

  const grainSeed = (x * 2654435761 + y * 340573321 + Math.floor(time * 60)) & 0xffffff
  const grain = ((grainSeed / 0xffffff) - 0.5) * 2 * grainAmt
  b = Math.max(0, Math.min(1, b + grain))

  const shimmer = 1.0 + 0.02 * Math.sin(y * 0.5 + time * shimSpd * 3.0)
  b = Math.max(0, Math.min(1, b * shimmer))

  const [lr, lg, lb] = palette.low
  const [hr, hg, hb] = palette.high
  const r = Math.round(lr + (hr - lr) * b)
  const g = Math.round(lg + (hg - lg) * b)
  const bl = Math.round(lb + (hb - lb) * b)

  return `rgb(${r},${g},${bl})`
}

function renderRetro(rc: RenderContext, config: RetroConfig): void {
  const {
    ctx, brightnessGrid, cols, rows,
    cellWidth, cellHeight, time, chars, fontSize,
    invert, vignetteStrength, brightness, contrast,
  } = rc
  const { retroNoise, retroDuotone } = config

  ctx.font = `${fontSize}px "Courier New", "Consolas", monospace`
  ctx.textBaseline = 'top'
  ctx.textAlign = 'left'

  const numBands = Math.max(2, chars.length)

  for (let y = 0; y < rows; y++) {
    for (let x = 0; x < cols; x++) {
      const idx = y * cols + x
      let b = brightnessGrid[idx]
      b = adjustBrightness(b, brightness, contrast)
      if (invert) b = 1 - b

      let charB = Math.pow(Math.max(0, b), 0.78)

      const grainSeed = (x * 2654435761 + y * 340573321 + Math.floor(time * 60)) & 0xffffff
      const grain = ((grainSeed / 0xffffff) - 0.5) * 2 * retroNoise
      charB = Math.max(0, Math.min(1, charB + grain * 0.15))

      const band = Math.floor(charB * numBands)
      const quantized = Math.min(1, band / (numBands - 1))

      const char = getCharForBrightness(quantized, chars)
      if (char === ' ') continue

      const color = getRetroColor(b, retroDuotone, x, y, cols, rows, time, {
        vignetteStrength,
        grainAmount: retroNoise * 0.3,
        shimmerSpeed: 1.0,
      })

      ctx.fillStyle = color
      ctx.fillText(char, x * cellWidth, y * cellHeight)
    }
  }
}
```

### Style 8: Terminal

Green phosphor monospace with scanline alternation.

```typescript
const TERMINAL_CHARSETS: Record<string, string> = {
  '101010':   ' 01',
  'brackets': ' []{}()<>',
  'dollar':   ' $#@!%&',
  'mixed':    ' .:;+=*#@',
  'pipes':    ' |/-\\=+#',
}

function getTerminalColor(brightness: number, x: number, y: number): string {
  const gray = Math.round(Math.max(0, Math.min(1, brightness)) * 255)
  const phosphor = Math.min(255, gray + 32)
  const scanShade = ((x + y) & 1) === 0 ? 1 : 0.84
  const green = Math.min(255, Math.floor(phosphor * scanShade))
  return `rgb(${Math.min(72, Math.floor(green * 0.12))},${green},${Math.min(88, Math.floor(green * 0.2))})`
}

function renderTerminal(
  rc: RenderContext,
  terminalCharset: string = 'mixed',
): void {
  const {
    ctx, brightnessGrid, cols, rows,
    cellWidth, cellHeight, fontSize,
    invert, vignetteStrength, brightness, contrast,
  } = rc

  const termChars = TERMINAL_CHARSETS[terminalCharset] ?? TERMINAL_CHARSETS['mixed']

  ctx.font = `${fontSize}px "VT323", monospace`
  ctx.textBaseline = 'top'
  ctx.textAlign = 'left'

  for (let y = 0; y < rows; y++) {
    for (let x = 0; x < cols; x++) {
      const idx = y * cols + x
      let b = brightnessGrid[idx]
      b = adjustBrightness(b, brightness, contrast)

      if (vignetteStrength > 0) {
        b *= getVignetteFactor(x, y, cols, rows, vignetteStrength)
      }
      if (invert) b = 1 - b

      const charB = Math.pow(Math.max(0, b), 0.82)
      const char = getCharForBrightness(charB, termChars)
      if (char === ' ') continue

      ctx.fillStyle = getTerminalColor(b, x, y)
      ctx.fillText(char, x * cellWidth, y * cellHeight)
    }
  }
}
```

### Style 9: Custom (Brand/Themed)

Fixed color with block charset. This example shows the warm orange-amber brand variant (Claude style). Replace the color formula with any brand color.

```typescript
function renderCustomBrand(
  rc: RenderContext,
  brandColor: { rMul: number; gMul: number; bMul: number; boost: number },
): void {
  const {
    ctx, brightnessGrid, cols, rows,
    cellWidth, cellHeight, fontSize,
    invert, vignetteStrength, brightness, contrast,
  } = rc

  const BLOCKS_CHARSET = ' ░▒▓█'

  ctx.font = `${fontSize}px "Courier New", "Consolas", monospace`
  ctx.textBaseline = 'top'
  ctx.textAlign = 'left'

  for (let y = 0; y < rows; y++) {
    for (let x = 0; x < cols; x++) {
      const idx = y * cols + x
      let b = brightnessGrid[idx]
      b = adjustBrightness(b, brightness, contrast)

      if (vignetteStrength > 0) {
        b *= getVignetteFactor(x, y, cols, rows, vignetteStrength)
      }
      if (invert) b = 1 - b

      const char = getCharForBrightness(b, BLOCKS_CHARSET)
      if (char === ' ') continue

      const gray = Math.round(b * 255)
      const intensity = gray + brandColor.boost
      const r = Math.max(0, Math.min(255, Math.round(intensity * brandColor.rMul)))
      const g = Math.max(0, Math.min(255, Math.round(intensity * brandColor.gMul)))
      const bl = Math.max(0, Math.min(255, Math.round(intensity * brandColor.bMul)))

      ctx.fillStyle = `rgb(${r},${g},${bl})`
      ctx.fillText(char, x * cellWidth, y * cellHeight)
    }
  }
}

// Usage: warm orange-amber (Claude)
// renderCustomBrand(rc, { rMul: 1.03, gMul: 0.5, bMul: 0.08, boost: 36 })
//
// Usage: ice blue
// renderCustomBrand(rc, { rMul: 0.3, gMul: 0.6, bMul: 1.05, boost: 20 })
```

---

## 6. Halftone Shape Renderers

Detailed implementations for each halftone shape. All shapes use brightness-proportional sizing with screen interference modulation.

### Circle

```typescript
function drawHalftoneCircle(
  ctx: CanvasRenderingContext2D,
  cx: number, cy: number,
  radius: number,
): void {
  ctx.beginPath()
  ctx.arc(cx, cy, radius, 0, Math.PI * 2)
  ctx.fill()
}
```

### Square

Rotatable filled rectangle.

```typescript
function drawHalftoneSquare(
  ctx: CanvasRenderingContext2D,
  cx: number, cy: number,
  radius: number,
  rotRad: number,
): void {
  ctx.save()
  ctx.translate(cx, cy)
  ctx.rotate(rotRad)
  const side = radius * 2
  ctx.fillRect(-side * 0.5, -side * 0.5, side, side)
  ctx.restore()
}
```

### Diamond

4-sided polygon rotated 45 degrees.

```typescript
function drawHalftoneDiamond(
  ctx: CanvasRenderingContext2D,
  cx: number, cy: number,
  radius: number,
  rotRad: number,
): void {
  drawRegularPolygon(ctx, cx, cy, radius, 4, Math.PI / 4 + rotRad)
}
```

### Pentagon

5-sided polygon with point-up orientation.

```typescript
function drawHalftonePentagon(
  ctx: CanvasRenderingContext2D,
  cx: number, cy: number,
  radius: number,
  rotRad: number,
): void {
  drawRegularPolygon(ctx, cx, cy, radius, 5, -Math.PI / 2 + rotRad)
}
```

### Hexagon

6-sided polygon (flat-top by default).

```typescript
function drawHalftoneHexagon(
  ctx: CanvasRenderingContext2D,
  cx: number, cy: number,
  radius: number,
  rotRad: number,
): void {
  drawRegularPolygon(ctx, cx, cy, radius, 6, rotRad)
}
```

### Screen Interference Pattern

The interference pattern creates a moire-like screen texture that modulates dot sizes across the grid, preventing uniform flat-looking output.

```typescript
function getScreenLevel(x: number, y: number): number {
  return (
    Math.sin(x * 0.82 + y * 0.33) * 1.55 +
    Math.cos(x * 0.27 - y * 0.94) * 1.25 +
    2
  ) * 0.25
}
```

The function combines two sinusoidal waves at different frequencies and phases. The output range is approximately [0, 1.2], biased toward 0.5. Multiply cell brightness by this value before computing dot radius to get the interference effect.

### Shape Selection Guide

| Shape | Visual Character | Best For |
|-------|-----------------|----------|
| Circle | Smooth, natural dots | Photographic content, portraits |
| Square | Crisp, geometric grid | Technical diagrams, pixel art |
| Diamond | Angled energy, dynamic feel | Action shots, geometric patterns |
| Pentagon | Organic irregularity | Abstract art, nature imagery |
| Hexagon | Honeycomb tiling, dense packing | Textures, backgrounds, scientific |

---

## 7. Choosing a Palette

### By Content Type

| Content | Recommended Palettes |
|---------|---------------------|
| Photographic images | `detailed`, `standard`, `default`, `dense` |
| Webcam / live video | `standard`, `blocks`, `dense` (fewer chars = faster) |
| High-contrast graphics | `blocks`, `minimal`, `binary` |
| Portraits / faces | `detailed`, `braille` (fine gradation preserves features) |
| Text / documents | `standard`, `detailed` |
| Dark scenes | `braille-sparse`, `dots`, `stars` (bright chars stand out) |
| Data visualization | `hex`, `circuit`, `math` |
| Anime / manga | `katakana`, `hiragana` |
| Fantasy / RPG | `rune`, `greek`, `zodiac` |
| Financial dashboards | `hex`, `terminal-dollar` |
| Music visualizer | `music`, `dots`, `stars` |
| Sci-fi interfaces | `circuit`, `math`, `logic`, `box` |
| Cultural theming | `cyrillic`, `arabic`, `hebrew`, `thai`, `devanagari` |

### By Aesthetic

| Aesthetic | Recommended Palettes |
|-----------|---------------------|
| Classic terminal | `standard`, `terminal-mixed`, `terminal-pipes` |
| Matrix / hacker | `binary`, `katakana`, `terminal-binary` |
| Retro computing | `blocks`, `minimal`, `braille-dense` |
| Sci-fi / futuristic | `rune`, `greek`, `math`, `circuit` |
| Elegant / artistic | `braille`, `dots`, `stars` |
| Chunky / bold | `blocks`, `blocks-extended`, `minimal` |
| Financial / data | `hex`, `terminal-dollar` |
| Playful / decorative | `arrows`, `stars`, `symbols`, `cards` |
| Gothic / mystical | `rune`, `crosses`, `chess` |
| Winter / cold | `snowflake`, `geometric`, `minimal` |
| Progress / bar charts | `vertical-bars`, `horizontal-bars`, `shade-fine` |

### By Grid Resolution

| Grid Size | Recommended |
|-----------|-------------|
| Small (< 40 cols) | Short palettes: `blocks` (5), `minimal` (4), `binary` (3) |
| Medium (40-120 cols) | Mid palettes: `standard` (10), `dense` (12), `default` (29) |
| Large (120+ cols) | Long palettes: `detailed` (68), `braille` (64), `default` (29) |

Longer palettes waste tonal range at low resolutions because adjacent brightness levels map to the same character. Shorter palettes at high resolution produce banding.

### Palette Length and Tonal Resolution

The number of distinct brightness levels equals the charset length. To determine if your palette is adequate:

```
Required tonal levels = visible brightness range / minimum perceivable brightness step
```

For typical viewing conditions, humans can distinguish roughly 20-30 brightness levels in ASCII art. Palettes with fewer than 8 characters produce visible posterization. Palettes beyond 64 characters provide diminishing returns except at very large grid sizes.

### Performance Note

All palettes have O(1) character lookup (direct index into string). Unicode characters render at the same speed as ASCII in Canvas 2D. The main performance factor is grid size (`cols * rows`), not palette length.

---

## 8. Aspect Ratio Correction

Monospace characters are not square. In standard monospace fonts, a character cell is approximately 60% as wide as it is tall. If you assume square cells, the output will be horizontally stretched.

### The Problem

A monospace font at 14px has a character cell roughly 8.4px wide and 16.8px tall (ratio ~0.5). If you naively compute grid dimensions assuming square cells, you get columns that are too narrow and rows that are too short, distorting the aspect ratio of the source content.

### The Fix

```typescript
const fontSize = 14
const characterSpacing = 0.6  // monospace width-to-height ratio

const cellWidth  = fontSize * characterSpacing  // 14 * 0.6 = 8.4px
const cellHeight = fontSize * 1.2               // 14 * 1.2 = 16.8px (line height)

const cols = Math.max(1, Math.floor(canvasWidth / cellWidth))
const rows = Math.max(1, Math.floor(canvasHeight / cellHeight))
```

### Why 0.6?

The exact ratio varies by font:

| Font | Width/Height Ratio |
|------|--------------------|
| Courier New | ~0.60 |
| Consolas | ~0.55 |
| SF Mono | ~0.60 |
| Fira Code | ~0.60 |
| JetBrains Mono | ~0.60 |
| VT323 | ~0.60 |
| Source Code Pro | ~0.60 |

0.6 is a safe default that works across essentially all monospace fonts. You can measure precisely:

```typescript
function measureCharRatio(ctx: CanvasRenderingContext2D, fontSize: number): number {
  ctx.font = `${fontSize}px monospace`
  const metrics = ctx.measureText('M')
  return metrics.width / fontSize
}
```

### Grid Dimensions at Common Resolutions

With `fontSize=14`, `characterSpacing=0.6`:

| Canvas Size | Cell Size | Grid (cols x rows) | Total Cells |
|-------------|-----------|---------------------|-------------|
| 1920 x 1080 | 8.4 x 16.8 | 228 x 64 | 14,592 |
| 1280 x 720  | 8.4 x 16.8 | 152 x 42 | 6,384 |
| 960 x 540   | 8.4 x 16.8 | 114 x 32 | 3,648 |
| 3840 x 2160 | 8.4 x 16.8 | 457 x 128 | 58,496 |

For 60fps rendering, keep total cells under ~40,000. At 4K, increase font size to reduce cell count.

### Source Image Sampling

When sampling a source image/video onto the character grid, draw the source at the grid resolution on a hidden sampler canvas:

```typescript
const samplerCanvas = document.createElement('canvas')
samplerCanvas.width = cols
samplerCanvas.height = rows
const samplerCtx = samplerCanvas.getContext('2d', { willReadFrequently: true })!

// Correct aspect: the source is resampled to grid dimensions
// which already account for non-square cells
samplerCtx.drawImage(source, 0, 0, sourceW, sourceH, 0, 0, cols, rows)
const imageData = samplerCtx.getImageData(0, 0, cols, rows)
```

The sampled pixel at `(col, row)` directly maps to the character at that grid position. The aspect correction is inherent because `cols` and `rows` were computed from non-square cell dimensions.

### Font Setup

```typescript
ctx.font = `${fontSize}px "Courier New", "Consolas", monospace`
ctx.textBaseline = 'top'
ctx.textAlign = 'left'
```

Always use `textBaseline: 'top'` so that `fillText(char, x * cellWidth, y * cellHeight)` positions the character at the top-left corner of its cell. Using `'alphabetic'` or `'middle'` causes vertical misalignment that breaks the grid.
