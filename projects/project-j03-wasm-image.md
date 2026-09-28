# Project J03: WASM Image Processor

> โมดูล: J — Full-Stack & WebAssembly | ความยาก: ⭐⭐⭐⭐⭐ | เวลาโดยประมาณ: 10 ชั่วโมง

## ภาพรวมโปรเจค

โปรเจคนี้สร้าง **Image Processing Library ที่ compile เป็น WebAssembly** แล้วเรียกใช้จาก JavaScript โดยตรง ผลลัพธ์คือ library ที่ให้ browser ประมวลผลภาพบน client-side ได้อย่างรวดเร็วโดยไม่ต้องส่งข้อมูลไปยัง server — ทั้ง grayscale, blur, sharpen, edge detection, color rotation, และ resize ล้วนทำงานใน WASM runtime ใน browser

ในโลก production การประมวลผลภาพบน client มีประโยชน์หลายกรณี: ตัวอย่างเช่น Squoosh (เครื่องมือ compress ภาพของ Google) ใช้ WASM สำหรับ codec หนัก ๆ อย่าง WebP และ AVIF, Figma ใช้ WASM render canvas ที่ซับซ้อน, และ Photopea (Photoshop บนเว็บ) ใช้ WASM ประมวลผลภาพทั้งหมด แนวทางนี้ทำให้ performance ใกล้เคียง native code โดยไม่ต้องติดตั้งโปรแกรมเพิ่มเติม

**Learning value** ของโปรเจคนี้ครอบคลุมหลายทักษะที่หายาก: การ compile Rust เป็น WASM target, การออกแบบ FFI boundary ระหว่าง Rust และ JavaScript, การจัดการ memory ที่ shared ระหว่าง WASM และ JS heap, รวมถึงการใช้ API ของ browser เช่น `ImageData` และ `canvas` ผ่าน `web-sys`

## สิ่งที่จะได้เรียนรู้

- ตั้งค่า Rust WASM project ด้วย `crate-type = ["cdylib"]` และ `wasm-bindgen`
- ประมวลผล RGBA pixel buffer ด้วย grayscale, brightness, contrast, invert
- สร้าง convolution kernel สำหรับ blur (Gaussian 3×3/5×5), sharpen, emboss, Sobel edge detection
- แปลงสี HSL ↔ RGB เพื่อทำ hue rotation และ saturation adjustment
- Resize ภาพด้วย nearest-neighbor และ bilinear interpolation
- ใช้ `js_sys::Uint8ClampedArray` และ `web_sys::ImageData` เชื่อมต่อกับ HTML canvas
- จัดการ memory อย่างมีประสิทธิภาพด้วย `wasm_bindgen::memory()` เพื่อหลีกเลี่ยง copy ที่ไม่จำเป็น
- เข้าใจข้อจำกัดของ SIMD และ threading ใน WASM environment
- สร้าง demo HTML page แบบ drag-drop พร้อม filter buttons และ canvas display

## ความรู้ที่ต้องมีมาก่อน

- จาก Part 1–30 เรื่อง Rust fundamentals: ownership, borrowing, slices, Vec, closures
- จาก Part 31–40 เรื่อง generics, traits, iterators, และ error handling
- จาก Part 41–50 เรื่อง unsafe Rust, raw pointers, FFI basics
- จาก Part 95–100 เรื่อง WebAssembly และ JavaScript interop
- ความเข้าใจพื้นฐานด้าน HTML5 Canvas API และ JavaScript ES modules

## โครงสร้างโปรเจค (Project Layout)

```
wasm-image/
├── src/
│   ├── lib.rs               ← entry point: #[wasm_bindgen] exports
│   ├── pixel.rs             ← pixel operations: grayscale, brightness, contrast
│   ├── convolution.rs       ← kernels: blur, sharpen, emboss, Sobel
│   ├── color.rs             ← HSL ↔ RGB, hue rotation, saturation
│   ├── resize.rs            ← nearest-neighbor, bilinear interpolation
│   └── memory.rs            ← shared memory buffer management
├── tests/
│   └── image_tests.rs       ← integration tests (native target)
├── web/
│   ├── index.html           ← demo page: drag-drop + filter buttons
│   ├── bootstrap.js         ← WASM loader (async import)
│   └── app.js               ← UI logic, canvas management
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Data Flow ระหว่าง JavaScript และ WASM

```
[Browser]                    [WASM Module]
    │                              │
    │  ImageData.data              │
    │  (Uint8ClampedArray RGBA)    │
    │                              │
    ├──── wasm_bindgen call ──────►│
    │     ptr: *mut u8             │  ประมวลผล
    │     len: usize               │  pixel in-place
    │     width, height            │
    │                              │
    │◄─── return (modified) ───────┤
    │                              │
    │  putImageData(canvas)        │
    │                              │
```

### สาเหตุที่ต้อง compile เป็น `cdylib`

Rust WASM library ต้องการ `crate-type = ["cdylib"]` เพราะ:
- `cdylib` สร้าง C-compatible dynamic library — WASM runtime ของ browser คาดหวัง exported symbols แบบ C ABI
- `rlib` (Rust library ปกติ) ไม่ export symbols ออกมาให้ JavaScript เห็น
- `wasm-bindgen` จะ generate JavaScript glue code จาก `cdylib` output โดยอัตโนมัติ

### การตัดสินใจด้าน Memory

**แบบที่ 1: Copy ข้อมูลเข้า/ออก WASM** — ง่ายที่สุด แต่ copy ภาพ 4K (33MB) สองครั้ง
**แบบที่ 2: Shared memory buffer** — JavaScript เขียนลง `wasm_bindgen::memory()` โดยตรงผ่าน pointer แล้ว call Rust function ที่ทำงาน in-place — ไม่ต้อง copy

โปรเจคนี้ใช้ทั้งสองแบบและอธิบายว่าเมื่อไหรควรใช้แบบไหน

### ข้อจำกัดของ WASM Threading

WASM Threads ต้องการ `SharedArrayBuffer` ซึ่งต้องมี COOP/COEP headers บน server จึงจะใช้งานได้ `rayon` ทำงานใน native ได้ แต่ใน WASM ต้องใช้ `wasm-bindgen-rayon` crate เพิ่มเติมและ configure web worker ด้วย — โปรเจคนี้จะอธิบายทั้งแบบ single-threaded และแนวทาง parallel

---

## การพัฒนาทีละขั้นตอน

---

### ขั้นที่ 1: ตั้งค่า WASM Project และ wasm-bindgen พื้นฐาน

ขั้นแรกคือการสร้างโปรเจคและตรวจสอบว่า toolchain ครบถ้วน จากนั้น export ฟังก์ชันง่าย ๆ ออกไปให้ JavaScript เรียกใช้ได้

#### ติดตั้ง WASM toolchain

```bash
# เพิ่ม WASM target
rustup target add wasm32-unknown-unknown

# ติดตั้ง wasm-bindgen CLI (จำเป็นสำหรับ generate JS glue)
cargo install wasm-bindgen-cli

# ติดตั้ง wasm-pack (ทางเลือกที่ครบกว่า)
cargo install wasm-pack
```

#### Cargo.toml

```toml
[package]
name = "wasm-image"
version = "0.1.0"
edition = "2021"

# crate-type บังคับสำหรับ WASM library
# cdylib = C-compatible dynamic library สำหรับ WASM target
# rlib = Rust library สำหรับ cargo test บน native target
[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
# wasm-bindgen: สร้าง JS ↔ Rust FFI bridge
wasm-bindgen = "0.2"

# web-sys: binding ไปยัง Web APIs (DOM, Canvas, ImageData)
web-sys = { version = "0.3", features = [
    "ImageData",
    "CanvasRenderingContext2d",
    "HtmlCanvasElement",
    "console",
] }

# js-sys: binding ไปยัง JavaScript built-in types
js-sys = "0.3"

# panic hook สำหรับ WASM — แปลง Rust panic เป็น JS Error
console_error_panic_hook = "0.1"

[dev-dependencies]
# ไม่ต้องมี WASM dependencies สำหรับ test

[profile.release]
# optimize size สำหรับ WASM deployment
opt-level = "s"    # optimize for size
lto = true         # link-time optimization
```

#### ทำไมต้องมีทั้ง `cdylib` และ `rlib`?

`rlib` ทำให้ `cargo test` ทำงานบน native target ได้ — เราสามารถ test pure algorithm logic โดยไม่ต้องมี WASM runtime และไม่ต้องมี browser

`cdylib` คือ output ที่ WASM target ต้องการ — ถ้ามีแค่ `cdylib` อย่างเดียว `cargo test` จะ fail เพราะ test runner ต้องการ `rlib`

#### src/lib.rs — entry point พื้นฐาน

```rust
// src/lib.rs
use wasm_bindgen::prelude::*;

// โหลด sub-modules
pub mod color;
pub mod convolution;
pub mod memory;
pub mod pixel;
pub mod resize;

/// เรียกก่อนใช้งาน library — ตั้งค่า panic hook
/// ทำให้ Rust panic แสดงเป็น JavaScript Error ที่อ่านได้
#[wasm_bindgen(start)]
pub fn init() {
    console_error_panic_hook::set_once();
}

/// ตัวอย่างฟังก์ชัน hello world สำหรับ verify WASM works
#[wasm_bindgen]
pub fn greet(name: &str) -> String {
    format!("สวัสดี {}! จาก Rust WASM", name)
}

/// คืนค่า version string ของ library
#[wasm_bindgen]
pub fn version() -> String {
    env!("CARGO_PKG_VERSION").to_string()
}
```

#### การ Build ครั้งแรก

```bash
# Build WASM โดยใช้ wasm-pack (แนะนำ)
wasm-pack build --target web --out-dir pkg

# หรือ build manual
cargo build --target wasm32-unknown-unknown --release
wasm-bindgen target/wasm32-unknown-unknown/release/wasm_image.wasm \
    --out-dir pkg --target web
```

หลัง build สำเร็จ ใน `pkg/` จะมี:
- `wasm_image_bg.wasm` — WASM binary
- `wasm_image.js` — JavaScript glue code ที่ generate อัตโนมัติ
- `wasm_image.d.ts` — TypeScript definitions

#### ทดสอบ greet จาก HTML

```html
<!-- web/index.html (ขั้นตอนทดสอบ) -->
<script type="module">
  import init, { greet, version } from './pkg/wasm_image.js';

  async function run() {
    await init(); // โหลด WASM module
    console.log(greet("Rust learner")); // "สวัสดี Rust learner! จาก Rust WASM"
    console.log("Version:", version()); // "0.1.0"
  }

  run();
</script>
```

---

### ขั้นที่ 2: Pixel Operations — RGBA Buffer, Grayscale, Brightness, Contrast

RGBA buffer คือ array ของ bytes โดยแต่ละ pixel ใช้ 4 bytes: [R, G, B, A] สลับกันไปเรื่อย ๆ ภาพ 100×100 pixel จะมี buffer ขนาด 100 × 100 × 4 = 40,000 bytes

นี่คือ format ที่ `ImageData.data` ของ HTML canvas ใช้ — ทำให้เราส่ง buffer นั้นตรงเข้า Rust ได้โดยไม่ต้องแปลง format

#### src/pixel.rs — การดำเนินการระดับ pixel

```rust
// src/pixel.rs

/// แปลงค่า RGB เป็น Grayscale ด้วยสูตร luminance มาตรฐาน ITU-R BT.601
/// สูตรนี้ถ่วงน้ำหนักตามความไวของตามนุษย์ต่อแต่ละสี:
/// เขียว (G) มีผลมากที่สุด (0.587) รองลงมาคือแดง (0.299) และน้ำเงิน (0.114)
pub fn rgb_to_gray(r: u8, g: u8, b: u8) -> u8 {
    let r = r as f32;
    let g = g as f32;
    let b = b as f32;
    (0.299 * r + 0.587 * g + 0.114 * b).round() as u8
}

/// ปรับ brightness ของ channel เดียว
/// delta บวก = สว่างขึ้น, delta ลบ = มืดลง
/// clamp ไว้ที่ 0-255 เสมอ
pub fn adjust_brightness(value: u8, delta: i32) -> u8 {
    (value as i32 + delta).clamp(0, 255) as u8
}

/// ปรับ contrast ด้วย factor
/// factor = 1.0 → ค่าเดิม
/// factor > 1.0 → contrast เพิ่มขึ้น (ส่วนสว่างยิ่งสว่าง ส่วนมืดยิ่งมืด)
/// factor < 1.0 → contrast ลดลง (ภาพดูหม่น)
/// สูตร: output = factor × (input - 128) + 128
pub fn adjust_contrast(value: u8, factor: f32) -> u8 {
    let v = value as f32;
    let adjusted = factor * (v - 128.0) + 128.0;
    adjusted.clamp(0.0, 255.0).round() as u8
}

/// แปลงภาพทั้งหมดเป็น grayscale
/// buf: RGBA buffer — แก้ไข in-place
pub fn apply_grayscale(buf: &mut [u8]) {
    debug_assert_eq!(buf.len() % 4, 0, "buffer must be multiple of 4");
    for chunk in buf.chunks_exact_mut(4) {
        let gray = rgb_to_gray(chunk[0], chunk[1], chunk[2]);
        chunk[0] = gray;
        chunk[1] = gray;
        chunk[2] = gray;
        // chunk[3] = alpha channel — ไม่แตะ
    }
}

/// ปรับ brightness ของภาพทั้งหมด — in-place
pub fn apply_brightness(buf: &mut [u8], delta: i32) {
    debug_assert_eq!(buf.len() % 4, 0);
    for chunk in buf.chunks_exact_mut(4) {
        chunk[0] = adjust_brightness(chunk[0], delta);
        chunk[1] = adjust_brightness(chunk[1], delta);
        chunk[2] = adjust_brightness(chunk[2], delta);
        // alpha ไม่เปลี่ยน
    }
}

/// ปรับ contrast ของภาพทั้งหมด — in-place
pub fn apply_contrast(buf: &mut [u8], factor: f32) {
    debug_assert_eq!(buf.len() % 4, 0);
    for chunk in buf.chunks_exact_mut(4) {
        chunk[0] = adjust_contrast(chunk[0], factor);
        chunk[1] = adjust_contrast(chunk[1], factor);
        chunk[2] = adjust_contrast(chunk[2], factor);
    }
}

/// invert สีทั้งหมด (negative effect)
pub fn apply_invert(buf: &mut [u8]) {
    debug_assert_eq!(buf.len() % 4, 0);
    for chunk in buf.chunks_exact_mut(4) {
        chunk[0] = 255 - chunk[0];
        chunk[1] = 255 - chunk[1];
        chunk[2] = 255 - chunk[2];
    }
}
```

#### export ออก WASM — เพิ่มใน src/lib.rs

```rust
// เพิ่มใน src/lib.rs

/// แปลงภาพเป็น grayscale (in-place บน buffer)
/// pixels: RGBA Uint8ClampedArray จาก JavaScript
#[wasm_bindgen]
pub fn grayscale(pixels: &mut [u8]) {
    pixel::apply_grayscale(pixels);
}

/// ปรับ brightness — delta: -255 ถึง +255
#[wasm_bindgen]
pub fn brightness(pixels: &mut [u8], delta: i32) {
    pixel::apply_brightness(pixels, delta);
}

/// ปรับ contrast — factor: 0.0 ถึง 3.0 (1.0 = ไม่เปลี่ยน)
#[wasm_bindgen]
pub fn contrast(pixels: &mut [u8], factor: f32) {
    pixel::apply_contrast(pixels, factor);
}

/// invert สีภาพ
#[wasm_bindgen]
pub fn invert(pixels: &mut [u8]) {
    pixel::apply_invert(pixels);
}
```

#### วิธีใช้จาก JavaScript

```javascript
// app.js
import init, { grayscale, brightness, contrast } from './pkg/wasm_image.js';

await init();

// รับ ImageData จาก canvas
const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d');
const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);

// ส่ง RGBA buffer ตรงเข้า WASM — ไม่มี copy!
// imageData.data เป็น Uint8ClampedArray ที่ share memory กับ JS heap
grayscale(imageData.data);

// นำผลลัพธ์กลับไปแสดงบน canvas
ctx.putImageData(imageData, 0, 0);
```

**หมายเหตุสำคัญ**: `grayscale(imageData.data)` ส่ง reference ของ buffer ตรงเข้า WASM โดยไม่ copy — `wasm-bindgen` จัดการ pointer translation ให้อัตโนมัติ ทำให้ทำงานได้เร็ว

---

### ขั้นที่ 3: Convolution Kernels — Blur, Sharpen, Emboss, Edge Detection

Convolution คือการ apply matrix (kernel) ขนาด 3×3 หรือ 5×5 กับ neighborhood ของแต่ละ pixel การเลือก kernel ต่างกันให้ผลลัพธ์ที่ต่างกันมาก

#### ทฤษฎี Convolution สั้น ๆ

สำหรับ pixel ที่ตำแหน่ง (x, y) ค่าใหม่คือ:

```
output(x, y) = Σ kernel(i,j) × input(x+i-1, y+j-1)
                i=0..2, j=0..2
```

ที่ edge ของภาพ pixel บางตัวจะอยู่นอก boundary — มีหลายวิธีจัดการ: padding ด้วย 0, clamp ที่ edge, หรือ wrap around โปรเจคนี้ใช้ **replicate padding** (ซ้ำค่า pixel ที่ขอบ)

#### src/convolution.rs

```rust
// src/convolution.rs

/// Gaussian blur 3×3 kernel (normalized)
/// ถ่วงน้ำหนักตาม Gaussian distribution — pixel ตรงกลางมีน้ำหนักมากสุด
pub const GAUSSIAN_3X3: [f32; 9] = [
    1.0/16.0, 2.0/16.0, 1.0/16.0,
    2.0/16.0, 4.0/16.0, 2.0/16.0,
    1.0/16.0, 2.0/16.0, 1.0/16.0,
];

/// Gaussian blur 5×5 kernel (normalized) — blur มากกว่า 3×3
pub const GAUSSIAN_5X5: [f32; 25] = [
    1.0/256.0,  4.0/256.0,  6.0/256.0,  4.0/256.0, 1.0/256.0,
    4.0/256.0, 16.0/256.0, 24.0/256.0, 16.0/256.0, 4.0/256.0,
    6.0/256.0, 24.0/256.0, 36.0/256.0, 24.0/256.0, 6.0/256.0,
    4.0/256.0, 16.0/256.0, 24.0/256.0, 16.0/256.0, 4.0/256.0,
    1.0/256.0,  4.0/256.0,  6.0/256.0,  4.0/256.0, 1.0/256.0,
];

/// Sharpen kernel — เพิ่มความคมชัดโดยเน้น pixel ตรงกลาง
/// sum ของ kernel = 1 ทำให้ brightness เฉลี่ยไม่เปลี่ยน
pub const SHARPEN_3X3: [f32; 9] = [
     0.0, -1.0,  0.0,
    -1.0,  5.0, -1.0,
     0.0, -1.0,  0.0,
];

/// Emboss kernel — สร้าง effect นูน 3D
/// pixel สว่างด้านหนึ่ง มืดอีกด้านหนึ่ง ขึ้นอยู่กับทิศทาง gradient
pub const EMBOSS_3X3: [f32; 9] = [
    -2.0, -1.0, 0.0,
    -1.0,  1.0, 1.0,
     0.0,  1.0, 2.0,
];

/// Sobel X — ตรวจจับ edge แนวตั้ง (gradient horizontal)
pub const SOBEL_X: [f32; 9] = [
    -1.0, 0.0, 1.0,
    -2.0, 0.0, 2.0,
    -1.0, 0.0, 1.0,
];

/// Sobel Y — ตรวจจับ edge แนวนอน (gradient vertical)
pub const SOBEL_Y: [f32; 9] = [
    -1.0, -2.0, -1.0,
     0.0,  0.0,  0.0,
     1.0,  2.0,  1.0,
];

/// ใช้ convolution kernel 3×3 กับ pixel เดียว (channel เดียว)
/// ใช้ replicate padding ที่ขอบภาพ
fn convolve_pixel_3x3(
    buf: &[u8],
    width: usize,
    height: usize,
    cx: usize,
    cy: usize,
    channel: usize,
    kernel: &[f32; 9],
) -> u8 {
    let mut sum = 0.0f32;
    for ky in 0..3usize {
        for kx in 0..3usize {
            let px = (cx as isize + kx as isize - 1).clamp(0, width as isize - 1) as usize;
            let py = (cy as isize + ky as isize - 1).clamp(0, height as isize - 1) as usize;
            let idx = (py * width + px) * 4 + channel;
            sum += buf[idx] as f32 * kernel[ky * 3 + kx];
        }
    }
    sum.clamp(0.0, 255.0).round() as u8
}

/// ใช้ convolution kernel 5×5 กับ pixel เดียว
fn convolve_pixel_5x5(
    buf: &[u8],
    width: usize,
    height: usize,
    cx: usize,
    cy: usize,
    channel: usize,
    kernel: &[f32; 25],
) -> u8 {
    let mut sum = 0.0f32;
    for ky in 0..5usize {
        for kx in 0..5usize {
            let px = (cx as isize + kx as isize - 2).clamp(0, width as isize - 1) as usize;
            let py = (cy as isize + ky as isize - 2).clamp(0, height as isize - 1) as usize;
            let idx = (py * width + px) * 4 + channel;
            sum += buf[idx] as f32 * kernel[ky * 5 + kx];
        }
    }
    sum.clamp(0.0, 255.0).round() as u8
}

/// Apply 3×3 kernel ทั้งภาพ — สร้าง output buffer ใหม่
pub fn apply_3x3(buf: &[u8], width: usize, height: usize, kernel: &[f32; 9]) -> Vec<u8> {
    let mut out = buf.to_vec();
    for y in 0..height {
        for x in 0..width {
            let idx = (y * width + x) * 4;
            out[idx]     = convolve_pixel_3x3(buf, width, height, x, y, 0, kernel);
            out[idx + 1] = convolve_pixel_3x3(buf, width, height, x, y, 1, kernel);
            out[idx + 2] = convolve_pixel_3x3(buf, width, height, x, y, 2, kernel);
            // alpha channel ไม่เปลี่ยน
        }
    }
    out
}

/// Apply 5×5 kernel ทั้งภาพ
pub fn apply_5x5(buf: &[u8], width: usize, height: usize, kernel: &[f32; 25]) -> Vec<u8> {
    let mut out = buf.to_vec();
    for y in 0..height {
        for x in 0..width {
            let idx = (y * width + x) * 4;
            out[idx]     = convolve_pixel_5x5(buf, width, height, x, y, 0, kernel);
            out[idx + 1] = convolve_pixel_5x5(buf, width, height, x, y, 1, kernel);
            out[idx + 2] = convolve_pixel_5x5(buf, width, height, x, y, 2, kernel);
        }
    }
    out
}

/// Gaussian blur 3×3
pub fn apply_blur_3x3(buf: &[u8], width: usize, height: usize) -> Vec<u8> {
    apply_3x3(buf, width, height, &GAUSSIAN_3X3)
}

/// Gaussian blur 5×5 — smooth กว่าแต่ช้ากว่า
pub fn apply_blur_5x5(buf: &[u8], width: usize, height: usize) -> Vec<u8> {
    apply_5x5(buf, width, height, &GAUSSIAN_5X5)
}

/// Sharpen filter
pub fn apply_sharpen(buf: &[u8], width: usize, height: usize) -> Vec<u8> {
    apply_3x3(buf, width, height, &SHARPEN_3X3)
}

/// Emboss filter
pub fn apply_emboss(buf: &[u8], width: usize, height: usize) -> Vec<u8> {
    apply_3x3(buf, width, height, &EMBOSS_3X3)
}

/// Sobel edge detection
/// แปลงเป็น grayscale ก่อน แล้วรวม Gx และ Gy
pub fn apply_edge_sobel(buf: &[u8], width: usize, height: usize) -> Vec<u8> {
    // แปลงเป็น grayscale ก่อน (ใช้เฉพาะ channel R ที่แปลงแล้ว)
    let mut gray = buf.to_vec();
    crate::pixel::apply_grayscale(&mut gray);

    let gx = apply_3x3(&gray, width, height, &SOBEL_X);
    let gy = apply_3x3(&gray, width, height, &SOBEL_Y);

    let mut out = buf.to_vec();
    for i in 0..(width * height) {
        let idx = i * 4;
        // magnitude = sqrt(Gx² + Gy²) — approximation ที่เร็วกว่า: |Gx| + |Gy|
        let magnitude = ((gx[idx] as f32).powi(2) + (gy[idx] as f32).powi(2))
            .sqrt()
            .clamp(0.0, 255.0) as u8;
        out[idx]     = magnitude;
        out[idx + 1] = magnitude;
        out[idx + 2] = magnitude;
    }
    out
}
```

#### Export convolution functions ออก WASM

```rust
// เพิ่มใน src/lib.rs

/// Gaussian blur 3×3
/// คืน Vec<u8> ใหม่เนื่องจาก convolution ต้องการ read จาก original buffer ขณะ write
#[wasm_bindgen]
pub fn blur(pixels: &[u8], width: u32, height: u32) -> Vec<u8> {
    convolution::apply_blur_3x3(pixels, width as usize, height as usize)
}

/// Gaussian blur 5×5 — smooth มากกว่า
#[wasm_bindgen]
pub fn blur_strong(pixels: &[u8], width: u32, height: u32) -> Vec<u8> {
    convolution::apply_blur_5x5(pixels, width as usize, height as usize)
}

/// Sharpen filter
#[wasm_bindgen]
pub fn sharpen(pixels: &[u8], width: u32, height: u32) -> Vec<u8> {
    convolution::apply_sharpen(pixels, width as usize, height as usize)
}

/// Emboss effect
#[wasm_bindgen]
pub fn emboss(pixels: &[u8], width: u32, height: u32) -> Vec<u8> {
    convolution::apply_emboss(pixels, width as usize, height as usize)
}

/// Sobel edge detection
#[wasm_bindgen]
pub fn edge_detect(pixels: &[u8], width: u32, height: u32) -> Vec<u8> {
    convolution::apply_edge_sobel(pixels, width as usize, height as usize)
}
```

#### ใช้ Filter จาก JavaScript

```javascript
// app.js — apply blur filter
const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);

// convolution ต้องการ buffer ใหม่ (ไม่ใช่ in-place)
// blur() คืน Uint8Array ใหม่จาก WASM heap
const blurred = blur(imageData.data, canvas.width, canvas.height);

// สร้าง ImageData ใหม่จาก buffer
const newImageData = new ImageData(
    new Uint8ClampedArray(blurred.buffer),
    canvas.width,
    canvas.height
);
ctx.putImageData(newImageData, 0, 0);
```

---

### ขั้นที่ 4: Color Transformations — HSL ↔ RGB, Hue Rotation, Saturation

RGB เป็น color model ที่ hardware ใช้ แต่ไม่ intuitive สำหรับการปรับสีตามที่ตามนุษย์รับรู้ **HSL** (Hue, Saturation, Lightness) อธิบายสีด้วย:
- **H (Hue)** — เฉดสี 0-360°: 0° แดง, 120° เขียว, 240° น้ำเงิน
- **S (Saturation)** — ความเข้มสี 0-1: 0 = เทา, 1 = สีเต็ม
- **L (Lightness)** — ความสว่าง 0-1: 0 = ดำ, 0.5 = ปกติ, 1 = ขาว

#### src/color.rs

```rust
// src/color.rs

/// แปลง RGB (0-255) เป็น HSL (H: 0-360°, S: 0-1, L: 0-1)
pub fn rgb_to_hsl(r: u8, g: u8, b: u8) -> (f32, f32, f32) {
    let r = r as f32 / 255.0;
    let g = g as f32 / 255.0;
    let b = b as f32 / 255.0;

    let max = r.max(g).max(b);
    let min = r.min(g).min(b);
    let delta = max - min;

    // Lightness = average ของ max และ min
    let l = (max + min) / 2.0;

    // Saturation = 0 ถ้าสีเป็น achromatic (gray)
    let s = if delta < 1e-6 {
        0.0
    } else {
        delta / (1.0 - (2.0 * l - 1.0).abs())
    };

    // Hue คำนวณจาก channel ที่เป็น max
    let h = if delta < 1e-6 {
        0.0
    } else if (max - r).abs() < 1e-6 {
        // max = R: hue อยู่ระหว่าง yellow และ magenta
        60.0 * (((g - b) / delta) % 6.0)
    } else if (max - g).abs() < 1e-6 {
        // max = G: hue อยู่ระหว่าง cyan และ yellow
        60.0 * ((b - r) / delta + 2.0)
    } else {
        // max = B: hue อยู่ระหว่าง magenta และ cyan
        60.0 * ((r - g) / delta + 4.0)
    };

    // normalize hue ให้อยู่ใน [0, 360)
    let h = if h < 0.0 { h + 360.0 } else { h };

    (h, s, l)
}

/// Helper: แปลง hue component กลับเป็น RGB channel value
fn hue_to_rgb(p: f32, q: f32, mut t: f32) -> f32 {
    if t < 0.0 { t += 1.0; }
    if t > 1.0 { t -= 1.0; }
    if t < 1.0 / 6.0 { return p + (q - p) * 6.0 * t; }
    if t < 1.0 / 2.0 { return q; }
    if t < 2.0 / 3.0 { return p + (q - p) * (2.0 / 3.0 - t) * 6.0; }
    p
}

/// แปลง HSL กลับเป็น RGB (0-255)
pub fn hsl_to_rgb(h: f32, s: f32, l: f32) -> (u8, u8, u8) {
    // สีเป็น achromatic (no saturation)
    if s < 1e-6 {
        let v = (l * 255.0).round() as u8;
        return (v, v, v);
    }

    let q = if l < 0.5 { l * (1.0 + s) } else { l + s - l * s };
    let p = 2.0 * l - q;
    let h_norm = h / 360.0; // normalize ให้อยู่ใน [0, 1]

    let r = hue_to_rgb(p, q, h_norm + 1.0 / 3.0);
    let g = hue_to_rgb(p, q, h_norm);
    let b = hue_to_rgb(p, q, h_norm - 1.0 / 3.0);

    (
        (r * 255.0).round() as u8,
        (g * 255.0).round() as u8,
        (b * 255.0).round() as u8,
    )
}

/// หมุน hue ของทุก pixel ตาม degrees
/// degrees = 180 → สีตรงข้าม (complementary colors)
/// degrees = 120 → rotate color wheel 1/3
pub fn apply_hue_rotation(buf: &mut [u8], degrees: f32) {
    debug_assert_eq!(buf.len() % 4, 0);
    for chunk in buf.chunks_exact_mut(4) {
        let (h, s, l) = rgb_to_hsl(chunk[0], chunk[1], chunk[2]);
        let new_h = (h + degrees).rem_euclid(360.0);
        let (r, g, b) = hsl_to_rgb(new_h, s, l);
        chunk[0] = r;
        chunk[1] = g;
        chunk[2] = b;
    }
}

/// ปรับ saturation ของภาพ
/// factor = 0.0 → grayscale
/// factor = 1.0 → ไม่เปลี่ยน
/// factor = 2.0 → สีเข้มขึ้นสองเท่า
pub fn apply_saturation(buf: &mut [u8], factor: f32) {
    debug_assert_eq!(buf.len() % 4, 0);
    for chunk in buf.chunks_exact_mut(4) {
        let (h, s, l) = rgb_to_hsl(chunk[0], chunk[1], chunk[2]);
        let new_s = (s * factor).clamp(0.0, 1.0);
        let (r, g, b) = hsl_to_rgb(h, new_s, l);
        chunk[0] = r;
        chunk[1] = g;
        chunk[2] = b;
    }
}

/// sepia tone effect — ให้ภาพดูเหมือนภาพเก่า
pub fn apply_sepia(buf: &mut [u8]) {
    debug_assert_eq!(buf.len() % 4, 0);
    for chunk in buf.chunks_exact_mut(4) {
        let r = chunk[0] as f32;
        let g = chunk[1] as f32;
        let b = chunk[2] as f32;
        chunk[0] = (r * 0.393 + g * 0.769 + b * 0.189).clamp(0.0, 255.0) as u8;
        chunk[1] = (r * 0.349 + g * 0.686 + b * 0.168).clamp(0.0, 255.0) as u8;
        chunk[2] = (r * 0.272 + g * 0.534 + b * 0.131).clamp(0.0, 255.0) as u8;
    }
}
```

#### Export สำหรับ WASM

```rust
// เพิ่มใน src/lib.rs

/// หมุน hue — degrees: 0-360
#[wasm_bindgen]
pub fn hue_rotate(pixels: &mut [u8], degrees: f32) {
    color::apply_hue_rotation(pixels, degrees);
}

/// ปรับ saturation — factor: 0.0 (grayscale) ถึง 3.0
#[wasm_bindgen]
pub fn saturate(pixels: &mut [u8], factor: f32) {
    color::apply_saturation(pixels, factor);
}

/// เพิ่ม sepia effect
#[wasm_bindgen]
pub fn sepia(pixels: &mut [u8]) {
    color::apply_sepia(pixels);
}
```

---

### ขั้นที่ 5: Image Resizing — Nearest-Neighbor และ Bilinear Interpolation

Resize ภาพมีสองแนวทางหลัก:

**Nearest-Neighbor**: เร็วที่สุด — ใช้ค่า pixel ที่ใกล้ที่สุดโดยตรง ผลลัพธ์ดูเป็นแบบ "pixelated" เหมาะกับ pixel art

**Bilinear Interpolation**: คุณภาพดีกว่า — interpolate ระหว่าง pixel 4 ตัวโดยรอบ ผลลัพธ์ smooth กว่า เหมาะกับภาพทั่วไป

#### src/resize.rs

```rust
// src/resize.rs

/// Resize ภาพด้วย Nearest-Neighbor interpolation
/// เร็วที่สุด แต่ผลลัพธ์อาจดู blocky ถ้า upscale มาก
pub fn resize_nearest(
    src: &[u8],
    src_w: usize,
    src_h: usize,
    dst_w: usize,
    dst_h: usize,
) -> Vec<u8> {
    debug_assert_eq!(src.len(), src_w * src_h * 4);
    let mut dst = vec![0u8; dst_w * dst_h * 4];

    let x_ratio = src_w as f32 / dst_w as f32;
    let y_ratio = src_h as f32 / dst_h as f32;

    for dy in 0..dst_h {
        for dx in 0..dst_w {
            // หา pixel ต้นฉบับที่ตรงกับ pixel ปลายทาง
            let sx = ((dx as f32 + 0.5) * x_ratio) as usize;
            let sy = ((dy as f32 + 0.5) * y_ratio) as usize;
            // clamp กันเกิน boundary
            let sx = sx.min(src_w - 1);
            let sy = sy.min(src_h - 1);

            let src_idx = (sy * src_w + sx) * 4;
            let dst_idx = (dy * dst_w + dx) * 4;
            dst[dst_idx..dst_idx + 4].copy_from_slice(&src[src_idx..src_idx + 4]);
        }
    }
    dst
}

/// Resize ภาพด้วย Bilinear interpolation
/// คุณภาพดีกว่า nearest-neighbor โดยเฉพาะเมื่อ downscale
pub fn resize_bilinear(
    src: &[u8],
    src_w: usize,
    src_h: usize,
    dst_w: usize,
    dst_h: usize,
) -> Vec<u8> {
    debug_assert_eq!(src.len(), src_w * src_h * 4);
    let mut dst = vec![0u8; dst_w * dst_h * 4];

    let x_ratio = src_w as f32 / dst_w as f32;
    let y_ratio = src_h as f32 / dst_h as f32;

    for dy in 0..dst_h {
        for dx in 0..dst_w {
            // จุดพิกัดใน source image (เป็น float)
            let gx = (dx as f32 + 0.5) * x_ratio - 0.5;
            let gy = (dy as f32 + 0.5) * y_ratio - 0.5;

            // pixel 4 ตัวรอบ ๆ จุด (gx, gy)
            let x0 = (gx.floor() as isize).max(0) as usize;
            let y0 = (gy.floor() as isize).max(0) as usize;
            let x1 = (x0 + 1).min(src_w - 1);
            let y1 = (y0 + 1).min(src_h - 1);

            // fractional part สำหรับ interpolation weights
            let wx = (gx - x0 as f32).max(0.0);
            let wy = (gy - y0 as f32).max(0.0);

            let dst_idx = (dy * dst_w + dx) * 4;

            for c in 0..4 {
                // bilinear interpolation สำหรับแต่ละ channel
                let p00 = src[(y0 * src_w + x0) * 4 + c] as f32;
                let p10 = src[(y0 * src_w + x1) * 4 + c] as f32;
                let p01 = src[(y1 * src_w + x0) * 4 + c] as f32;
                let p11 = src[(y1 * src_w + x1) * 4 + c] as f32;

                // interpolate แนวนอนก่อน แล้ว interpolate แนวตั้ง
                let top    = p00 * (1.0 - wx) + p10 * wx;
                let bottom = p01 * (1.0 - wx) + p11 * wx;
                let value  = top * (1.0 - wy) + bottom * wy;

                dst[dst_idx + c] = value.round() as u8;
            }
        }
    }
    dst
}

/// Crop ภาพตาม region ที่กำหนด
/// คืน Err ถ้า region เกิน boundary
pub fn crop(
    src: &[u8],
    src_w: usize,
    src_h: usize,
    x: usize,
    y: usize,
    w: usize,
    h: usize,
) -> Result<Vec<u8>, String> {
    if x + w > src_w || y + h > src_h {
        return Err(format!(
            "crop region ({},{},{},{}) เกิน image boundary ({}x{})",
            x, y, w, h, src_w, src_h
        ));
    }
    let mut dst = vec![0u8; w * h * 4];
    for row in 0..h {
        let src_start = ((y + row) * src_w + x) * 4;
        let dst_start = row * w * 4;
        dst[dst_start..dst_start + w * 4]
            .copy_from_slice(&src[src_start..src_start + w * 4]);
    }
    Ok(dst)
}
```

#### Export สำหรับ WASM

```rust
// เพิ่มใน src/lib.rs

/// Resize ด้วย nearest-neighbor
#[wasm_bindgen]
pub fn resize_nn(
    pixels: &[u8],
    src_width: u32,
    src_height: u32,
    dst_width: u32,
    dst_height: u32,
) -> Vec<u8> {
    resize::resize_nearest(
        pixels,
        src_width as usize,
        src_height as usize,
        dst_width as usize,
        dst_height as usize,
    )
}

/// Resize ด้วย bilinear interpolation (คุณภาพดีกว่า)
#[wasm_bindgen]
pub fn resize_bl(
    pixels: &[u8],
    src_width: u32,
    src_height: u32,
    dst_width: u32,
    dst_height: u32,
) -> Vec<u8> {
    resize::resize_bilinear(
        pixels,
        src_width as usize,
        src_height as usize,
        dst_width as usize,
        dst_height as usize,
    )
}

/// Crop ภาพ — คืน JsValue::NULL ถ้า region เกิน boundary
#[wasm_bindgen]
pub fn crop_image(
    pixels: &[u8],
    src_width: u32,
    src_height: u32,
    x: u32,
    y: u32,
    w: u32,
    h: u32,
) -> Result<Vec<u8>, JsValue> {
    resize::crop(
        pixels,
        src_width as usize,
        src_height as usize,
        x as usize,
        y as usize,
        w as usize,
        h as usize,
    )
    .map_err(|e| JsValue::from_str(&e))
}
```

---

### ขั้นที่ 6: JavaScript Interop — Uint8ClampedArray, ImageData, Canvas

ขั้นนี้เชื่อม WASM library กับ browser API อย่างสมบูรณ์ โดยใช้ `web_sys` และ `js_sys` สำหรับ typed API

#### การทำงานกับ ImageData

`ImageData` ของ Canvas มี `.data` ที่เป็น `Uint8ClampedArray` — array ของ bytes RGBA ที่ clamp ค่าไว้ที่ 0-255 อัตโนมัติ

```rust
// src/lib.rs — helper ที่ทำงานกับ web-sys types

use wasm_bindgen::prelude::*;
use web_sys::{CanvasRenderingContext2d, HtmlCanvasElement, ImageData};
use js_sys::Uint8ClampedArray;

/// Apply filter บน ImageData โดยตรง (สำหรับใช้กับ canvas)
/// คืน ImageData ใหม่พร้อมผลลัพธ์
#[wasm_bindgen]
pub fn apply_filter_to_image_data(
    image_data: &ImageData,
    filter_name: &str,
) -> Result<ImageData, JsValue> {
    // ดึง bytes จาก ImageData
    let data = image_data.data();
    let width = image_data.width();
    let height = image_data.height();
    let len = (width * height * 4) as usize;

    // copy ลง Vec<u8> เพื่อประมวลผล
    let mut pixels = vec![0u8; len];
    for i in 0..len {
        pixels[i] = data.get_index(i as u32);
    }

    // apply filter ตามชื่อ
    let result = match filter_name {
        "grayscale" => {
            pixel::apply_grayscale(&mut pixels);
            pixels
        }
        "invert" => {
            pixel::apply_invert(&mut pixels);
            pixels
        }
        "sepia" => {
            color::apply_sepia(&mut pixels);
            pixels
        }
        "blur" => convolution::apply_blur_3x3(&pixels, width as usize, height as usize),
        "sharpen" => convolution::apply_sharpen(&pixels, width as usize, height as usize),
        "emboss" => convolution::apply_emboss(&pixels, width as usize, height as usize),
        "edges" => convolution::apply_edge_sobel(&pixels, width as usize, height as usize),
        _ => return Err(JsValue::from_str(&format!("unknown filter: {}", filter_name))),
    };

    // สร้าง ImageData ใหม่จาก result buffer
    let clamped = Uint8ClampedArray::new_with_length(len as u32);
    for (i, &byte) in result.iter().enumerate() {
        clamped.set_index(i as u32, byte);
    }

    ImageData::new_with_u8_clamped_array_and_sh(&clamped, width, height)
}
```

#### web/bootstrap.js — WASM Loader

```javascript
// web/bootstrap.js
// โหลด WASM module แบบ async และส่ง API ออกไป

let wasmModule = null;

export async function loadWasm() {
    if (wasmModule) return wasmModule;

    // dynamic import ของ WASM JS glue
    const wasm = await import('./pkg/wasm_image.js');
    await wasm.default(); // รัน init() ซึ่ง set panic hook

    wasmModule = wasm;
    console.log('[WASM] loaded, version:', wasm.version());
    return wasm;
}

export function getWasm() {
    if (!wasmModule) throw new Error('WASM not loaded yet — call loadWasm() first');
    return wasmModule;
}
```

#### web/app.js — UI Logic

```javascript
// web/app.js
import { loadWasm } from './bootstrap.js';

let wasm = null;
let originalImageData = null;

// โหลด WASM เมื่อหน้าเว็บเปิด
window.addEventListener('DOMContentLoaded', async () => {
    wasm = await loadWasm();
    setupDragDrop();
    setupFilterButtons();
    document.getElementById('status').textContent = 'WASM โหลดแล้ว — ลากภาพมาวางเพื่อเริ่ม';
});

function setupDragDrop() {
    const canvas = document.getElementById('canvas');
    const dropZone = document.getElementById('drop-zone');

    // prevent default browser behavior
    ['dragenter', 'dragover', 'dragleave', 'drop'].forEach(event => {
        dropZone.addEventListener(event, e => e.preventDefault());
        document.body.addEventListener(event, e => e.preventDefault());
    });

    dropZone.addEventListener('drop', async (e) => {
        const file = e.dataTransfer.files[0];
        if (!file || !file.type.startsWith('image/')) {
            alert('กรุณาลากไฟล์ภาพ (PNG, JPEG, WebP)');
            return;
        }
        await loadImageFile(file);
    });

    // รองรับ click เพื่อ browse ด้วย
    document.getElementById('file-input').addEventListener('change', async (e) => {
        const file = e.target.files[0];
        if (file) await loadImageFile(file);
    });
}

async function loadImageFile(file) {
    const url = URL.createObjectURL(file);
    const img = new Image();

    await new Promise((resolve, reject) => {
        img.onload = resolve;
        img.onerror = reject;
        img.src = url;
    });

    const canvas = document.getElementById('canvas');
    canvas.width = img.width;
    canvas.height = img.height;

    const ctx = canvas.getContext('2d');
    ctx.drawImage(img, 0, 0);

    // เก็บ original ไว้สำหรับ reset
    originalImageData = ctx.getImageData(0, 0, canvas.width, canvas.height);

    URL.revokeObjectURL(url);
    document.getElementById('status').textContent =
        `โหลดภาพ ${img.width}×${img.height} px`;
    document.getElementById('drop-zone').style.display = 'none';
}

function setupFilterButtons() {
    const filters = [
        { id: 'btn-grayscale', filter: 'grayscale', label: 'Grayscale' },
        { id: 'btn-blur',      filter: 'blur',      label: 'Blur 3×3' },
        { id: 'btn-sharpen',   filter: 'sharpen',   label: 'Sharpen' },
        { id: 'btn-emboss',    filter: 'emboss',    label: 'Emboss' },
        { id: 'btn-edges',     filter: 'edges',     label: 'Edge Detect' },
        { id: 'btn-invert',    filter: 'invert',    label: 'Invert' },
        { id: 'btn-sepia',     filter: 'sepia',     label: 'Sepia' },
    ];

    filters.forEach(({ id, filter }) => {
        document.getElementById(id)?.addEventListener('click', () => applyFilter(filter));
    });

    document.getElementById('btn-reset')?.addEventListener('click', resetImage);

    // slider สำหรับ brightness/contrast
    document.getElementById('brightness-slider')?.addEventListener('input', (e) => {
        applyBrightness(parseInt(e.target.value));
    });

    document.getElementById('hue-slider')?.addEventListener('input', (e) => {
        applyHueRotate(parseFloat(e.target.value));
    });
}

function applyFilter(filterName) {
    if (!originalImageData || !wasm) return;

    const canvas = document.getElementById('canvas');
    const ctx = canvas.getContext('2d');

    // เริ่มจาก original เสมอ (ไม่ stack filters)
    ctx.putImageData(originalImageData, 0, 0);
    const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);

    const t0 = performance.now();

    // เรียก WASM function
    const newData = wasm.apply_filter_to_image_data(imageData, filterName);
    ctx.putImageData(newData, 0, 0);

    const ms = (performance.now() - t0).toFixed(1);
    document.getElementById('status').textContent =
        `Filter: ${filterName} — ${ms} ms`;
}

function applyBrightness(delta) {
    if (!originalImageData || !wasm) return;
    const canvas = document.getElementById('canvas');
    const ctx = canvas.getContext('2d');
    ctx.putImageData(originalImageData, 0, 0);
    const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
    // in-place modification
    wasm.brightness(imageData.data, delta);
    ctx.putImageData(imageData, 0, 0);
}

function applyHueRotate(degrees) {
    if (!originalImageData || !wasm) return;
    const canvas = document.getElementById('canvas');
    const ctx = canvas.getContext('2d');
    ctx.putImageData(originalImageData, 0, 0);
    const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
    wasm.hue_rotate(imageData.data, degrees);
    ctx.putImageData(imageData, 0, 0);
}

function resetImage() {
    if (!originalImageData) return;
    const canvas = document.getElementById('canvas');
    const ctx = canvas.getContext('2d');
    ctx.putImageData(originalImageData, 0, 0);
    document.getElementById('status').textContent = 'Reset แล้ว';

    // reset sliders
    document.getElementById('brightness-slider').value = 0;
    document.getElementById('hue-slider').value = 0;
}
```

---

### ขั้นที่ 7: Memory Management — Shared Buffer และหลีกเลี่ยง Copy

การส่งข้อมูลระหว่าง JavaScript และ WASM มีสองโหมด:

**โหมด Copy** (ปลอดภัย ง่าย): `wasm-bindgen` copy ข้อมูลจาก JS heap เข้า WASM memory แล้ว copy กลับ — ภาพ 1080p (8MB) จะ copy 2 ครั้ง

**โหมด Zero-Copy** (เร็วกว่า ซับซ้อนกว่า): ใช้ `wasm_bindgen::memory()` เข้าถึง WASM memory โดยตรงจาก JS แล้วส่ง pointer เข้า WASM — ไม่ copy เลย

#### src/memory.rs — Shared Buffer Management

```rust
// src/memory.rs
use wasm_bindgen::prelude::*;

/// Allocate buffer ใน WASM heap และคืน pointer + length
/// JavaScript จะใช้ pointer นี้เขียนข้อมูลลงตรง ๆ
#[wasm_bindgen]
pub struct ImageBuffer {
    data: Vec<u8>,
    width: u32,
    height: u32,
}

#[wasm_bindgen]
impl ImageBuffer {
    /// สร้าง buffer ขนาด width * height * 4 bytes
    #[wasm_bindgen(constructor)]
    pub fn new(width: u32, height: u32) -> ImageBuffer {
        let len = (width * height * 4) as usize;
        ImageBuffer {
            data: vec![0u8; len],
            width,
            height,
        }
    }

    /// คืน pointer ไปยัง data array (JavaScript ใช้เพื่อ zero-copy write)
    pub fn data_ptr(&self) -> *const u8 {
        self.data.as_ptr()
    }

    /// คืน mutable pointer สำหรับ JavaScript write
    pub fn data_ptr_mut(&mut self) -> *mut u8 {
        self.data.as_mut_ptr()
    }

    /// คืนความยาวของ buffer เป็น bytes
    pub fn data_len(&self) -> u32 {
        self.data.len() as u32
    }

    pub fn width(&self) -> u32 { self.width }
    pub fn height(&self) -> u32 { self.height }

    /// Apply grayscale in-place
    pub fn apply_grayscale(&mut self) {
        crate::pixel::apply_grayscale(&mut self.data);
    }

    /// Apply brightness in-place
    pub fn apply_brightness(&mut self, delta: i32) {
        crate::pixel::apply_brightness(&mut self.data, delta);
    }

    /// Apply blur — สร้าง buffer ใหม่ (convolution ต้องการ read original)
    pub fn apply_blur(&self) -> ImageBuffer {
        let result = crate::convolution::apply_blur_3x3(
            &self.data,
            self.width as usize,
            self.height as usize,
        );
        ImageBuffer {
            data: result,
            width: self.width,
            height: self.height,
        }
    }

    /// Copy ข้อมูลออกเป็น Vec สำหรับ JavaScript อ่าน
    pub fn get_data(&self) -> Vec<u8> {
        self.data.clone()
    }
}
```

#### Zero-Copy จาก JavaScript

```javascript
// zero-copy pattern — ไม่มีการ copy ข้อมูลเลย
import init, * as wasm from './pkg/wasm_image.js';

await init();

// สร้าง buffer ใน WASM heap
const buf = new wasm.ImageBuffer(canvas.width, canvas.height);

// อ่าน pointer ของ buffer
const ptr = buf.data_ptr_mut();
const len = buf.data_len();

// สร้าง Uint8ClampedArray ที่ point ตรงไปยัง WASM memory
// ไม่มีการ copy ข้อมูล!
const wasmMemory = wasm.memory; // WebAssembly.Memory object
const wasmView = new Uint8ClampedArray(wasmMemory.buffer, ptr, len);

// copy ImageData จาก canvas เข้า WASM memory โดยตรง
const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
wasmView.set(imageData.data); // copy เข้า WASM memory ครั้งเดียว

// Apply filter — ทำงานใน WASM memory โดยตรง ไม่ copy
buf.apply_grayscale();

// อ่านผลลัพธ์กลับ
// ณ จุดนี้ wasmView แสดงข้อมูลที่แก้ไขแล้วแล้ว
const resultData = new ImageData(
    new Uint8ClampedArray(wasmView), // copy กลับครั้งเดียว
    canvas.width,
    canvas.height
);
ctx.putImageData(resultData, 0, 0);

// อย่าลืม free buffer เมื่อใช้เสร็จ
buf.free();
```

**ข้อควรระวัง**: เมื่อ WASM memory grow (เพราะ allocate ใหม่ใน Rust) `wasmMemory.buffer` จะ detach — `wasmView` จะชี้ไปที่ ArrayBuffer เก่าที่ detached แล้ว ต้องสร้าง `wasmView` ใหม่ทุกครั้งหลัง WASM allocation หรือใช้ `try/catch` detect และ rebuild

---

### ขั้นที่ 8: Performance — SIMD Hints และ Parallel Processing

#### SIMD ใน Rust WASM

WebAssembly SIMD (128-bit) รองรับใน browser สมัยใหม่ทุกตัว Rust สามารถ target WASM SIMD ด้วย `#[target_feature(enable = "simd128")]`

```rust
// src/pixel.rs — SIMD-optimized grayscale (สำหรับ WASM SIMD target)

// compile ด้วย: RUSTFLAGS="-C target-feature=+simd128"
// หรือ .cargo/config.toml:
// [build]
// target = "wasm32-unknown-unknown"
// rustflags = ["-C", "target-feature=+simd128"]

/// Grayscale แบบ manual SIMD hint — ให้ compiler vectorize
/// ใช้ `chunks_exact` + simple arithmetic ที่ compiler อาจ auto-vectorize ได้
pub fn apply_grayscale_fast(buf: &mut [u8]) {
    // เขียนแบบ "SIMD-friendly": ไม่มี branching, access pattern ง่าย
    let n = buf.len() / 4;
    for i in 0..n {
        let base = i * 4;
        // integer arithmetic แทน float — เร็วกว่าและ SIMD-friendly
        // สูตร: (77*R + 150*G + 29*B) / 256  ≈  ITU BT.601
        let r = buf[base] as u32;
        let g = buf[base + 1] as u32;
        let b = buf[base + 2] as u32;
        let gray = ((77 * r + 150 * g + 29 * b) >> 8) as u8;
        buf[base]     = gray;
        buf[base + 1] = gray;
        buf[base + 2] = gray;
    }
}
```

#### Rayon — Parallel Processing

Rayon ทำงานได้ดีบน **native target** (desktop/server) แต่ใน WASM ต้องการ setup พิเศษ:

```toml
# Cargo.toml — เพิ่ม dependency ตามเงื่อนไข
[target.'cfg(not(target_arch = "wasm32"))'.dependencies]
rayon = "1.10"

[target.'cfg(target_arch = "wasm32")'.dependencies]
wasm-bindgen-rayon = { version = "1.2", optional = true }
```

```rust
// src/pixel.rs — parallel grayscale สำหรับ native target

#[cfg(not(target_arch = "wasm32"))]
pub fn apply_grayscale_parallel(buf: &mut [u8]) {
    use rayon::prelude::*;
    buf.par_chunks_exact_mut(4).for_each(|chunk| {
        let gray = rgb_to_gray(chunk[0], chunk[1], chunk[2]);
        chunk[0] = gray;
        chunk[1] = gray;
        chunk[2] = gray;
    });
}

// WASM target ใช้ single-threaded version เดิม
#[cfg(target_arch = "wasm32")]
pub fn apply_grayscale_parallel(buf: &mut [u8]) {
    apply_grayscale(buf); // fallback to single-threaded
}
```

#### Benchmark ง่าย ๆ ใน Browser

```javascript
// เพิ่มใน app.js สำหรับ performance measurement
async function benchmarkFilter(filterName, iterations = 10) {
    if (!originalImageData || !wasm) return;

    const canvas = document.getElementById('canvas');
    const ctx = canvas.getContext('2d');
    const times = [];

    for (let i = 0; i < iterations; i++) {
        ctx.putImageData(originalImageData, 0, 0);
        const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);

        const t0 = performance.now();
        const result = wasm.apply_filter_to_image_data(imageData, filterName);
        times.push(performance.now() - t0);

        ctx.putImageData(result, 0, 0);
    }

    const avg = times.reduce((a, b) => a + b) / times.length;
    const min = Math.min(...times);
    console.log(`${filterName}: avg=${avg.toFixed(1)}ms, min=${min.toFixed(1)}ms`);
}
```

#### ผลลัพธ์ที่คาดหวังสำหรับภาพ 1920×1080

| Filter | WASM (SIMD off) | WASM (SIMD on) | Native Rust (rayon) |
|--------|-----------------|----------------|---------------------|
| Grayscale | ~15ms | ~5ms | ~2ms |
| Blur 3×3 | ~90ms | ~30ms | ~20ms |
| Blur 5×5 | ~250ms | ~80ms | ~50ms |
| Edge Detect | ~200ms | ~65ms | ~40ms |

---

### ขั้นที่ 9: Demo HTML Page — Drag-Drop, Filters, Canvas Display

#### web/index.html — หน้า Demo ครบถ้วน

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>WASM Image Processor — Rust + WebAssembly</title>
    <style>
        * { box-sizing: border-box; margin: 0; padding: 0; }

        body {
            font-family: 'Segoe UI', sans-serif;
            background: #1a1a2e;
            color: #e0e0e0;
            min-height: 100vh;
            padding: 20px;
        }

        h1 {
            text-align: center;
            color: #00d4ff;
            margin-bottom: 10px;
            font-size: 1.8rem;
        }

        #status {
            text-align: center;
            color: #888;
            margin-bottom: 20px;
            font-size: 0.9rem;
        }

        #drop-zone {
            border: 2px dashed #00d4ff;
            border-radius: 12px;
            padding: 40px;
            text-align: center;
            cursor: pointer;
            color: #00d4ff;
            margin-bottom: 20px;
            transition: background 0.2s;
        }

        #drop-zone:hover { background: rgba(0, 212, 255, 0.05); }

        input[type="file"] { display: none; }

        .controls {
            display: flex;
            flex-wrap: wrap;
            gap: 8px;
            justify-content: center;
            margin-bottom: 20px;
        }

        button {
            background: #16213e;
            color: #00d4ff;
            border: 1px solid #00d4ff;
            border-radius: 6px;
            padding: 8px 16px;
            cursor: pointer;
            font-size: 0.85rem;
            transition: all 0.2s;
        }

        button:hover { background: #00d4ff; color: #1a1a2e; }
        button:disabled { opacity: 0.4; cursor: not-allowed; }

        #btn-reset {
            border-color: #ff6b6b;
            color: #ff6b6b;
        }
        #btn-reset:hover { background: #ff6b6b; color: white; }

        .slider-group {
            display: flex;
            align-items: center;
            gap: 10px;
            margin-bottom: 10px;
        }

        .slider-group label {
            min-width: 100px;
            font-size: 0.85rem;
        }

        input[type="range"] {
            flex: 1;
            accent-color: #00d4ff;
        }

        #canvas-container {
            display: flex;
            justify-content: center;
            overflow: auto;
        }

        canvas {
            max-width: 100%;
            border-radius: 8px;
            border: 1px solid #333;
        }

        .section-title {
            color: #888;
            font-size: 0.75rem;
            text-transform: uppercase;
            letter-spacing: 1px;
            text-align: center;
            margin-bottom: 8px;
        }
    </style>
</head>
<body>
    <h1>🦀 WASM Image Processor</h1>
    <p id="status">กำลังโหลด WASM...</p>

    <!-- Drag & Drop Zone -->
    <div id="drop-zone" onclick="document.getElementById('file-input').click()">
        <p>ลากภาพมาวางที่นี่ หรือคลิกเพื่อเลือกไฟล์</p>
        <p style="font-size:0.8rem; margin-top:8px; color:#666">รองรับ PNG, JPEG, WebP, GIF</p>
    </div>
    <input type="file" id="file-input" accept="image/*">

    <!-- Filter Buttons -->
    <p class="section-title">Filters</p>
    <div class="controls">
        <button id="btn-grayscale">Grayscale</button>
        <button id="btn-blur">Blur 3×3</button>
        <button id="btn-blur-strong">Blur 5×5</button>
        <button id="btn-sharpen">Sharpen</button>
        <button id="btn-emboss">Emboss</button>
        <button id="btn-edges">Edge Detect</button>
        <button id="btn-invert">Invert</button>
        <button id="btn-sepia">Sepia</button>
        <button id="btn-reset">Reset</button>
    </div>

    <!-- Adjustments Sliders -->
    <p class="section-title">Adjustments</p>
    <div style="max-width:500px; margin: 0 auto 20px;">
        <div class="slider-group">
            <label>Brightness</label>
            <input type="range" id="brightness-slider" min="-100" max="100" value="0">
            <span id="brightness-value">0</span>
        </div>
        <div class="slider-group">
            <label>Contrast</label>
            <input type="range" id="contrast-slider" min="0" max="300" value="100" step="10">
            <span id="contrast-value">1.0</span>
        </div>
        <div class="slider-group">
            <label>Hue Rotation</label>
            <input type="range" id="hue-slider" min="0" max="360" value="0">
            <span id="hue-value">0°</span>
        </div>
        <div class="slider-group">
            <label>Saturation</label>
            <input type="range" id="sat-slider" min="0" max="300" value="100" step="10">
            <span id="sat-value">1.0</span>
        </div>
    </div>

    <!-- Canvas -->
    <div id="canvas-container">
        <canvas id="canvas"></canvas>
    </div>

    <script type="module" src="app.js"></script>
</body>
</html>
```

#### การ Serve Demo ด้วย Local Server

เนื่องจาก WASM ต้องมี `Content-Type: application/wasm` และ modules ต้องการ HTTP (ไม่ใช่ `file://`) จึงต้องใช้ local server:

```bash
# ติดตั้ง simple-http-server
cargo install simple-http-server

# หรือใช้ Python
python3 -m http.server 8080 --directory web/

# หรือใช้ Node.js serve
npx serve web/

# build WASM ก่อน
wasm-pack build --target web --out-dir web/pkg

# แล้ว open browser
open http://localhost:8080
```

---

## การทดสอบ (Testing)

เราทดสอบ pure algorithm logic บน native target โดยไม่ต้องการ WASM runtime หรือ browser

### การรัน Tests

```bash
# รัน tests บน native target (เร็วกว่า WASM)
cargo test

# หรือระบุ target ชัดเจน
cargo test --lib
```

### Real `cargo test` Output

```
   Compiling wasm_image_test v0.1.0 (/tmp/claude-0/-home-user-rust-course/.../scratchpad/wasm_image_test)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.49s
     Running unittests src/lib.rs (target/debug/deps/wasm_image_test-ca3e3c721aa2651b)

running 28 tests
test tests::test_adjust_brightness_clamp_min ... ok
test tests::test_adjust_brightness_negative ... ok
test tests::test_adjust_brightness_positive ... ok
test tests::test_adjust_contrast_clamp ... ok
test tests::test_adjust_contrast_increase ... ok
test tests::test_adjust_brightness_clamp_max ... ok
test tests::test_apply_grayscale_preserves_alpha ... ok
test tests::test_adjust_contrast_neutral ... ok
test tests::test_apply_invert ... ok
test tests::test_apply_grayscale_single_pixel ... ok
test tests::test_gaussian_blur_uniform_image ... ok
test tests::test_resize_bilinear_downsample ... ok
test tests::test_resize_bilinear_uniform_color ... ok
test tests::test_resize_nearest_2x2_to_4x4 ... ok
test tests::test_resize_nearest_preserves_uniform_color ... ok
test tests::test_rgb_to_gray_black ... ok
test tests::test_rgb_to_gray_blue ... ok
test tests::test_rgb_to_gray_green ... ok
test tests::test_rgb_to_gray_red ... ok
test tests::test_hsl_to_rgb_roundtrip ... ok
test tests::test_rgb_to_gray_white ... ok
test tests::test_rgb_to_hsl_black ... ok
test tests::test_rgb_to_hsl_red ... ok
test tests::test_rgb_to_hsl_white ... ok
test tests::test_saturation_zero_is_grayscale ... ok
test tests::test_sharpen_kernel_identity_for_uniform ... ok
test tests::test_sobel_edge_on_uniform_image ... ok
test tests::test_hue_rotation_360_is_identity ... ok

test result: ok. 28 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s

   Doc-tests wasm_image_test

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

### Unit Tests ที่สำคัญ

Tests ครอบคลุมทุก subsystem:

#### Grayscale Tests
```rust
#[test]
fn test_rgb_to_gray_red() {
    // 0.299 × 255 = 76.245 → round → 76
    assert_eq!(rgb_to_gray(255, 0, 0), 76);
}

#[test]
fn test_apply_grayscale_preserves_alpha() {
    let mut buf = vec![100u8, 150, 200, 128];
    apply_grayscale(&mut buf);
    assert_eq!(buf[3], 128); // alpha channel ต้องไม่เปลี่ยน
}
```

#### Brightness Clamp Tests
```rust
#[test]
fn test_adjust_brightness_clamp_max() {
    // ไม่ควร overflow เกิน 255
    assert_eq!(adjust_brightness(200, 100), 255);
}

#[test]
fn test_adjust_brightness_clamp_min() {
    // ไม่ควร underflow ต่ำกว่า 0
    assert_eq!(adjust_brightness(20, -100), 0);
}
```

#### HSL Roundtrip Tests
```rust
#[test]
fn test_hsl_to_rgb_roundtrip() {
    // แปลง RGB → HSL → RGB ต้องได้ค่าใกล้เคียง (±2 สำหรับ floating point rounding)
    let (r0, g0, b0) = (120u8, 80u8, 200u8);
    let (h, s, l) = rgb_to_hsl(r0, g0, b0);
    let (r1, g1, b1) = hsl_to_rgb(h, s, l);
    assert!((r0 as i32 - r1 as i32).abs() <= 2);
    assert!((g0 as i32 - g1 as i32).abs() <= 2);
    assert!((b0 as i32 - b1 as i32).abs() <= 2);
}
```

#### Resize Tests
```rust
#[test]
fn test_resize_nearest_2x2_to_4x4() {
    // ภาพ 2×2 สี่ quadrant → resize เป็น 4×4
    // top-left quadrant ควรเป็นสีเดียวกับ pixel ต้นฉบับ
    let src = vec![
        255u8, 0, 0, 255,   // TL = red
        0, 255, 0, 255,     // TR = green
        0, 0, 255, 255,     // BL = blue
        255, 255, 0, 255,   // BR = yellow
    ];
    let dst = resize_nearest(&src, 2, 2, 4, 4);
    assert_eq!(dst.len(), 4 * 4 * 4);
    assert_eq!(&dst[0..4], &[255u8, 0, 0, 255]); // TL ยังเป็นแดง
}
```

### Integration Tests — tests/image_tests.rs

```rust
// tests/image_tests.rs
use wasm_image::{pixel, convolution, color, resize};

#[test]
fn integration_full_pipeline() {
    // สร้างภาพ test 4×4 pixel ที่มีสีหลากหลาย
    let mut image = create_test_image(4, 4);

    // grayscale ต้องทำให้ R=G=B ทุก pixel
    pixel::apply_grayscale(&mut image);
    for i in 0..(4 * 4) {
        assert_eq!(image[i * 4], image[i * 4 + 1],
            "pixel {} should have R=G", i);
        assert_eq!(image[i * 4 + 1], image[i * 4 + 2],
            "pixel {} should have G=B", i);
    }
}

#[test]
fn integration_blur_then_sharpen_approximately_identity() {
    // blur แล้ว sharpen ควรได้ค่าใกล้เคียงต้นฉบับ (ไม่เท่าเพราะ rounding)
    let original = create_test_image(8, 8);
    let blurred = convolution::apply_blur_3x3(&original, 8, 8);
    let sharpened = convolution::apply_sharpen(&blurred, 8, 8);

    // ตรวจสอบว่า pixel ที่อยู่ตรงกลาง (ไม่ใช่ edge) ใกล้เคียงต้นฉบับ
    let center_idx = (3 * 8 + 3) * 4;
    let diff_r = (original[center_idx] as i32 - sharpened[center_idx] as i32).abs();
    assert!(diff_r <= 20, "center pixel should be close to original");
}

fn create_test_image(width: usize, height: usize) -> Vec<u8> {
    let mut buf = vec![0u8; width * height * 4];
    for i in 0..(width * height) {
        buf[i * 4]     = ((i * 13) % 256) as u8; // R
        buf[i * 4 + 1] = ((i * 17 + 50) % 256) as u8; // G
        buf[i * 4 + 2] = ((i * 23 + 100) % 256) as u8; // B
        buf[i * 4 + 3] = 255; // A = opaque
    }
    buf
}
```

---

## การ Package และ Deploy

### Build สำหรับ Production

```bash
# build WASM + generate JS glue ด้วย wasm-pack
wasm-pack build --target web --out-dir web/pkg --release

# ตรวจสอบขนาด WASM
ls -lh web/pkg/wasm_image_bg.wasm
# ควรน้อยกว่า 200KB สำหรับ library ขนาดนี้

# optimize ขนาดเพิ่มเติมด้วย wasm-opt (ต้องติดตั้ง binaryen)
wasm-opt -Oz web/pkg/wasm_image_bg.wasm -o web/pkg/wasm_image_bg.wasm
```

### โครงสร้าง Output หลัง Build

```
web/pkg/
├── wasm_image.js          ← JavaScript entry point (ES module)
├── wasm_image_bg.wasm     ← WASM binary (optimized)
├── wasm_image.d.ts        ← TypeScript definitions
├── wasm_image_bg.js       ← Internal glue (ไม่ต้อง import โดยตรง)
└── package.json           ← npm package descriptor
```

### Serve Requirements

WASM ต้องการ HTTP headers เฉพาะ:

```nginx
# nginx configuration
location /pkg/ {
    add_header Content-Type application/wasm;
    # สำหรับ WASM Threads (SharedArrayBuffer)
    add_header Cross-Origin-Opener-Policy same-origin;
    add_header Cross-Origin-Embedder-Policy require-corp;
}
```

### Deploy บน GitHub Pages

```yaml
# .github/workflows/deploy.yml
name: Deploy WASM Demo

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Rust + wasm-pack
        run: |
          rustup target add wasm32-unknown-unknown
          cargo install wasm-pack

      - name: Build WASM
        run: wasm-pack build --target web --out-dir web/pkg --release

      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v4
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: web/
```

### Publish เป็น npm Package

```bash
# wasm-pack สร้าง package.json ให้แล้ว
cd pkg/
npm publish --access public
```

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

### 1. ลืมใส่ `crate-type = ["cdylib"]`

**อาการ**: `cargo build --target wasm32-unknown-unknown` ผ่าน แต่ไม่มี `.wasm` file ใน output หรือ `wasm-bindgen` fail

**สาเหตุ**: ค่า default คือ `rlib` ซึ่ง build เป็น Rust library — ไม่ใช่ WASM module ที่ JavaScript เรียกได้

**วิธีแก้**: ต้องมีทั้งสอง type ใน Cargo.toml:
```toml
[lib]
crate-type = ["cdylib", "rlib"]
```
- `cdylib` สำหรับ WASM target
- `rlib` สำหรับ `cargo test` บน native

---

### 2. WASM Memory Detach หลัง Allocation

**อาการ**: `wasmView.set(data)` throw `TypeError: Cannot perform %TypedArray%.prototype.set on a detached ArrayBuffer`

**สาเหตุ**: เมื่อ Rust allocate memory ใหม่ (เช่น Vec grow) WASM linear memory อาจ grow ตาม — `WebAssembly.Memory` ต้อง reallocate buffer ใหม่ ทำให้ `ArrayBufferView` เก่าทั้งหมด detach

**วิธีแก้**: สร้าง view ใหม่จาก `wasm.memory.buffer` ทุกครั้งหลัง WASM call ที่อาจ allocate:
```javascript
function getWasmView(ptr, len) {
    // อย่า cache wasmView — สร้างใหม่ทุกครั้ง
    return new Uint8ClampedArray(wasm.memory.buffer, ptr, len);
}
```

---

### 3. Off-by-One ใน Convolution Boundary

**อาการ**: ภาพมี artifact แปลก ๆ ที่ขอบด้านขวาและล่าง — เส้นสีเข้มหรือสีจาง

**สาเหตุ**: การคำนวณ index ผิดตอนทำ replicate padding โดยเฉพาะเมื่อ `cx = width - 1` แล้วพยายาม access `cx + 1`

**วิธีแก้**: ใช้ `clamp` อย่างถูกต้อง:
```rust
// ผิด — อาจ panic หรือ access pixel ผิดตัว
let px = (cx as isize + kx as isize - 1) as usize;

// ถูก — clamp ก่อนแปลง type
let px = (cx as isize + kx as isize - 1)
    .clamp(0, (width as isize) - 1) as usize;
```

---

### 4. Buffer Length ไม่ใช่ Multiple of 4

**อาการ**: `panic: buffer must be multiple of 4` หรือ `debug_assert` fail

**สาเหตุ**: JavaScript ส่ง buffer ที่ตัดมาจากส่วนหนึ่งของ ImageData หรือ Uint8ClampedArray ที่ slice ผิด

**วิธีแก้**: ตรวจสอบทั้ง Rust และ JavaScript:
```rust
// Rust — assert เสมอ
pub fn apply_grayscale(buf: &mut [u8]) {
    assert_eq!(buf.len() % 4, 0,
        "RGBA buffer ต้องมี length เป็น multiple of 4, got {}", buf.len());
    // ...
}
```
```javascript
// JavaScript — ตรวจก่อนส่ง
if (imageData.data.length % 4 !== 0) {
    throw new Error('ImageData corrupted');
}
```

---

### 5. `console_error_panic_hook` ไม่ถูก Initialize

**อาการ**: Rust panic ใน WASM แสดงเป็น `RuntimeError: unreachable executed` ใน browser console — อ่านไม่รู้เรื่อง

**สาเหตุ**: ไม่ได้เรียก `console_error_panic_hook::set_once()` ก่อนใช้งาน

**วิธีแก้**: เรียกผ่าน `#[wasm_bindgen(start)]`:
```rust
#[wasm_bindgen(start)]
pub fn init() {
    console_error_panic_hook::set_once();
}
```
หรือใน JavaScript หลัง `init()`:
```javascript
await init(); // WASM init() runs automatically including panic hook
```

---

### 6. wasm-bindgen Version Mismatch

**อาการ**: `Error: wasm-bindgen version mismatch` หรือ imported symbols ไม่พบ

**สาเหตุ**: version ของ `wasm-bindgen` crate ใน Cargo.toml ไม่ตรงกับ `wasm-bindgen-cli` ที่ติดตั้ง

**วิธีแก้**: ตรวจสอบและ match version:
```bash
# ดู version ใน Cargo.lock
grep 'wasm-bindgen' Cargo.lock | head -5

# ติดตั้ง CLI ให้ตรง version
cargo install wasm-bindgen-cli --version 0.2.99
# หรือใช้ wasm-pack ซึ่งจัดการ version ให้อัตโนมัติ
```

---

### 7. `Vec<u8>` Return ที่ JavaScript ต้องจัดการ Memory

**อาการ**: Memory leak ใน WASM heap เมื่อเรียก filter บ่อย ๆ

**สาเหตุ**: เมื่อ WASM function คืน `Vec<u8>` ไปยัง JavaScript `wasm-bindgen` copy ข้อมูลลง JS heap แล้ว Vec ใน WASM ถูก drop — แต่ JavaScript ต้องไม่ hold reference ไว้นานเกินจำเป็น

**วิธีแก้**: สำหรับ filter ที่เรียกซ้ำ ๆ ใช้ `ImageBuffer` struct ที่ reuse ได้แทนการสร้าง buffer ใหม่ทุกครั้ง:
```javascript
// ไม่ดี — สร้าง buffer ใหม่ทุกครั้ง
function applyFilter() {
    const result = wasm.blur(pixels, w, h); // allocate ใน WASM ทุกครั้ง
    // result ถูก GC ใน JS แต่ WASM memory อาจไม่ลดทันที
}

// ดีกว่า — reuse buffer
const buf = new wasm.ImageBuffer(canvas.width, canvas.height);
function applyFilter() {
    buf.apply_blur(); // in-place หรือ reuse memory
}
// เมื่อเลิกใช้
buf.free();
```

---

## แบบฝึกหัด (Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม Filter Sepia และ Cool/Warm Tone ⭐⭐

สร้าง color grading filters 3 ตัวใหม่:

1. **Warm tone** — เพิ่ม R ลด B (ให้ภาพดูอบอุ่น)
2. **Cool tone** — เพิ่ม B ลด R (ให้ภาพดูเย็น)
3. **Vintage** — ลด saturation 30% + เพิ่ม brightness ที่ shadows

**Starter code:**
```rust
pub fn apply_warm_tone(buf: &mut [u8], strength: f32) {
    // strength: 0.0 - 1.0
    // ใช้ strength × 30 เพิ่ม R, strength × 20 ลด B
    // TODO: implement
}
```

**เป้าหมาย**: เพิ่ม export ใน `lib.rs` และทดสอบว่า warm(0.0) ไม่เปลี่ยนภาพ และ warm(1.0) เพิ่ม R channel

---

### แบบฝึกหัดที่ 2: Histogram Equalization ⭐⭐⭐

Histogram equalization คือเทคนิคปรับ contrast โดยกระจาย distribution ของ brightness ให้สม่ำเสมอ — ทำให้ภาพที่มืดเกินหรือสว่างเกินดูดีขึ้น

**ขั้นตอน:**
1. คำนวณ histogram ของ luminance (256 bucket)
2. คำนวณ cumulative distribution function (CDF)
3. Map ค่า luminance ใหม่ตาม CDF
4. Apply mapping ไปยังทุก pixel

```rust
pub fn apply_histogram_equalization(buf: &mut [u8], width: usize, height: usize) {
    // 1. คำนวณ histogram ของ grayscale value
    let mut histogram = [0u32; 256];
    for chunk in buf.chunks_exact(4) {
        let gray = rgb_to_gray(chunk[0], chunk[1], chunk[2]) as usize;
        histogram[gray] += 1;
    }

    // 2. คำนวณ CDF
    // TODO: implement CDF calculation

    // 3. สร้าง mapping table
    // TODO: implement mapping

    // 4. Apply mapping
    // TODO: apply to each pixel
}
```

**เป้าหมาย**: ทดสอบกับภาพที่ luminance distribution ไม่สม่ำเสมอ — histogram หลังทำควรกระจายสม่ำเสมอกว่า

---

### แบบฝึกหัดที่ 3: Dithering สำหรับลด Color Depth ⭐⭐⭐⭐

Floyd-Steinberg dithering แปลงภาพเป็น 1-bit (ขาวดำ) หรือ limited palette โดยกระจาย quantization error ไปยัง pixel โดยรอบ ทำให้ภาพดูมี detail มากกว่าการ threshold ธรรมดา

```rust
/// Floyd-Steinberg dithering — แปลงเป็น 1-bit B&W
pub fn apply_dithering_1bit(buf: &mut [u8], width: usize, height: usize) {
    // แปลงเป็น grayscale ก่อน
    apply_grayscale(buf);

    // สร้าง error buffer (f32 สำหรับ accumulate error)
    let mut errors = vec![0.0f32; width * height];

    for y in 0..height {
        for x in 0..width {
            let idx = (y * width + x) * 4;
            let old_val = buf[idx] as f32 + errors[y * width + x];

            // quantize เป็น 0 หรือ 255
            let new_val = if old_val > 127.0 { 255.0 } else { 0.0 };
            let error = old_val - new_val;

            buf[idx]     = new_val as u8;
            buf[idx + 1] = new_val as u8;
            buf[idx + 2] = new_val as u8;

            // กระจาย error ตาม Floyd-Steinberg pattern:
            // . * 7/16
            // 3/16 5/16 1/16
            // TODO: กระจาย error ไปยัง neighbors
        }
    }
}
```

**เป้าหมาย**: implement การกระจาย error ตาม Floyd-Steinberg pattern และทดสอบว่าภาพที่ได้ดูมี detail กว่า simple threshold

---

### แบบฝึกหัดที่ 4: Multi-pass Pipeline API ⭐⭐⭐

ออกแบบ API ที่ให้ chain filter ได้หลายตัวใน single call เพื่อลดจำนวน copy และ round-trip ระหว่าง JS และ WASM:

```rust
// ไม่ดี — 3 round-trip + 6 copy
let result1 = blur(pixels, w, h);
let result2 = contrast(&result1, 1.5);
let result3 = hue_rotate(&result2, 90.0);

// ดี — 1 round-trip, pipeline ทำงานใน WASM ทั้งหมด
let result = apply_pipeline(pixels, w, h, &[
    Filter::Blur,
    Filter::Contrast(1.5),
    Filter::HueRotate(90.0),
]);
```

**สร้าง:**

```rust
#[wasm_bindgen]
#[derive(Debug)]
pub struct FilterPipeline {
    steps: Vec<PipelineStep>,
}

#[derive(Debug)]
enum PipelineStep {
    Grayscale,
    Brightness(i32),
    Contrast(f32),
    Blur3x3,
    Blur5x5,
    Sharpen,
    HueRotate(f32),
    Saturate(f32),
}

#[wasm_bindgen]
impl FilterPipeline {
    #[wasm_bindgen(constructor)]
    pub fn new() -> FilterPipeline {
        FilterPipeline { steps: Vec::new() }
    }

    pub fn add_grayscale(&mut self) -> &mut FilterPipeline {
        self.steps.push(PipelineStep::Grayscale);
        self
    }

    // TODO: เพิ่ม methods สำหรับแต่ละ step

    pub fn execute(&self, pixels: &mut [u8], width: u32, height: u32) {
        // TODO: execute แต่ละ step ตามลำดับ
    }
}
```

**เป้าหมาย**: วัดว่า pipeline API ลด execution time เท่าไหรเทียบกับ call แยก (เพราะลด copy)

---

### แบบฝึกหัดที่ 5: Perlin Noise Texture Generation ⭐⭐⭐⭐⭐

สร้าง Perlin noise ใน Rust แล้ว export เป็น WASM สำหรับ generate texture แบบ procedural:

```rust
/// Generate Perlin noise เป็น RGBA buffer
/// octaves: จำนวน noise layer (มากขึ้น = detail มากขึ้น)
/// persistence: amplitude ลดลงแต่ละ octave (0.0-1.0)
#[wasm_bindgen]
pub fn generate_perlin_noise(
    width: u32,
    height: u32,
    scale: f32,
    octaves: u32,
    persistence: f32,
    seed: u32,
) -> Vec<u8> {
    // TODO: implement Perlin noise
    // 1. สร้าง permutation table จาก seed
    // 2. สำหรับแต่ละ pixel คำนวณ noise value รวม octaves ทั้งหมด
    // 3. normalize ให้อยู่ใน [0, 255]
    // 4. คืนเป็น RGBA grayscale buffer
    todo!()
}
```

**เป้าหมาย**: render noise texture ใน canvas ด้วย sliders สำหรับ scale, octaves, persistence แล้วเห็น terrain-like pattern

---

## สรุป

โปรเจคนี้สร้าง **WASM Image Processing Library** ที่ครบถ้วน — ตั้งแต่ pixel operations พื้นฐาน ไปจนถึง convolution kernels, color spaces, resizing และ JavaScript interop

### Pattern สำคัญที่ได้เรียน

**1. WASM Project Structure**
- `crate-type = ["cdylib", "rlib"]` — cdylib สำหรับ WASM, rlib สำหรับ cargo test
- `#[wasm_bindgen]` macro ทำงาน FFI boundary ระหว่าง Rust และ JavaScript
- `#[wasm_bindgen(start)]` สำหรับ initialization code ที่รันอัตโนมัติ

**2. Memory Management ใน Cross-Language Boundary**
- `wasm-bindgen` copy data ผ่าน `Uint8Array` โดยอัตโนมัติ — ง่ายแต่ overhead สำหรับภาพใหญ่
- Zero-copy ผ่าน `wasm_bindgen::memory()` — เร็วกว่าแต่ต้องระวัง memory detach
- WASM linear memory ไม่ shrink อัตโนมัติ — ใช้ struct แทน Vec กรณีที่ reuse บ่อย

**3. Algorithm Testing Strategy**
- Test pure algorithms บน native target — ไม่ต้องรอ WASM compile
- WASM target เหมาะสำหรับ browser integration test ที่ต้องการ DOM API

**4. Performance ใน WASM**
- SIMD 128-bit ทำงานบน modern browsers ทุกตัว — enable ด้วย rustflags
- Threading ต้องการ COOP/COEP headers และ SharedArrayBuffer — setup ซับซ้อนกว่า
- สำหรับ CPU-bound workload ขนาดใหญ่ WASM SIMD เร็วกว่า pure JavaScript 2-5x

**5. Deployment**
- `wasm-pack` จัดการ toolchain, packaging, และ npm publish ครบ
- GitHub Pages ใช้ได้ แต่ต้องเพิ่ม custom headers สำหรับ WASM Threads

### เชื่อมโยงไปโปรเจคถัดไป

โปรเจคถัดไป **Project J04: WASM Plugin System** จะขยายแนวทางนี้ไปสู่ **plugin architecture** — ที่ผู้ใช้สามารถ load WASM module ใหม่ตอน runtime เพื่อเพิ่ม filter หรือ algorithm โดยไม่ต้อง rebuild application หลัก ซึ่งจะใช้ `WebAssembly.instantiateStreaming` สำหรับ lazy load และ WASM Component Model สำหรับ standardized interface

---

**โปรเจคก่อนหน้า:** [Project J02: Full-Stack Axum](project-j02-fullstack-axum.md) | **โปรเจคถัดไป:** [Project J04: WASM Plugin System](project-j04-wasm-plugin.md)
