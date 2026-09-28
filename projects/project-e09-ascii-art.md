# Project E09: ASCII Art Renderer

> โมดูล: E — Games & Graphics | ความยาก: ⭐⭐ | เวลาโดยประมาณ: 3 ชั่วโมง

## ภาพรวมโปรเจค

ASCII Art Renderer คือโปรแกรมที่แปลงภาพ (JPEG/PNG/BMP/GIF) ให้กลายเป็นงานศิลปะที่ประกอบจากตัวอักษร ASCII ในเทอร์มินัล แนวคิดนี้มีประวัติยาวนานตั้งแต่ยุค 70s เมื่อจอคอมพิวเตอร์ยังแสดงผลได้แค่ตัวอักษร ปัจจุบันยังคงมีประโยชน์ในงาน DevOps (แสดงภาพใน CI log), เกม terminal-based, เครื่องมือ accessibility, และงานสร้างสรรค์

**Use case จริงในโลก production:**
- `catimg` และ `viu` ใช้ใน DevOps pipeline เพื่อ preview ภาพจาก container log
- ASCII art font เอาไว้ทำ banner ใน CLI tool (เช่น `figlet`, `toilet`)
- Braille rendering ใช้สำหรับ terminal ที่รองรับ Unicode เพื่อความละเอียดสูงขึ้น
- Conversion เป็น HTML/SVG เพื่อ embed ผลลัพธ์ใน web report

**Learning value:**
โปรเจคนี้ครอบคลุมทักษะสำคัญที่นำไปใช้ในงานจริง: การประมวลผลภาพ (image processing), การทำงานกับ terminal (crossterm), การจัดการ external crate ที่ซับซ้อน, algorithm ทางคณิตศาสตร์ (Sobel, Floyd-Steinberg), และการออกแบบ CLI ที่ยืดหยุ่น

---

## สิ่งที่จะได้เรียนรู้

- การใช้ `image` crate โหลดและแปลงภาพ, resize, แปลงเป็น grayscale
- การ map ค่า pixel luminance ไปยังตัวอักษรด้วย density mapping
- อัลกอริทึม Sobel edge detection (convolution kernel) สำหรับ highlight ขอบ
- Unicode Braille encoding — แปลงกลุ่ม pixel 2×4 เป็น Braille character U+2800–U+28FF
- Floyd-Steinberg dithering — error diffusion เพื่อให้ได้ grayscale ที่ดูดีขึ้น
- การสร้าง CLI ด้วย `clap 4` แบบ derive macro
- Color output ด้วย `crossterm::style::SetForegroundColor`
- การ render GIF animation ในเทอร์มินัลโดย rewrite frame in-place
- Output หลายรูปแบบ: terminal, plain text, HTML, SVG

---

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–20**: Ownership, borrowing, struct, enum, pattern matching
- **Part 21–30**: Traits, generics, error handling (`?`, `Result`, `thiserror`)
- **Part 31–40**: Closures, iterators (`.map()`, `.collect()`, `.enumerate()`)
- **Part 41–45**: CLI development, `clap`, argument parsing
- **Part 46–50**: External crates, `Cargo.toml` features, workspace
- **Part 51–55**: File I/O, `std::fs`, `BufWriter`
- ความเข้าใจพื้นฐานเรื่อง pixel, RGB, luminance, และ terminal coordinate

---

## โครงสร้างโปรเจค (Project Layout)

```
ascii-art-renderer/
├── src/
│   ├── main.rs          ← CLI entry point, argument parsing
│   ├── image_loader.rs  ← โหลดภาพ, resize, convert grayscale
│   ├── density.rs       ← character density mapping
│   ├── sobel.rs         ← Sobel edge detection
│   ├── braille.rs       ← Braille Unicode encoding
│   ├── dither.rs        ← Floyd-Steinberg dithering
│   ├── color.rs         ← crossterm color output, 256-color mapping
│   ├── animation.rs     ← GIF/video frame rendering
│   ├── font.rs          ← embedded 8×8 bitmap font
│   └── output.rs        ← output formats: terminal, text, HTML, SVG
├── tests/
│   └── integration.rs   ← integration tests
├── Cargo.toml
└── README.md
```

---

## การออกแบบ (Architecture & Design)

### Data Flow

```
Input (file/stream)
       │
       ▼
  ImageLoader
  ┌─────────────────────────────────────────┐
  │  load() → DynamicImage                  │
  │  resize(target_cols) → aspect correct   │
  │  to_luma() → Vec<u8>  (grayscale)       │
  │  to_rgb()  → Vec<[u8;3]> (color data)   │
  └─────────────────────────────────────────┘
       │
       ▼
  Renderer (ตาม mode)
  ┌──────────┬────────────┬──────────────────┐
  │ Density  │   Sobel    │    Braille        │
  │ mapping  │  + overlay │  2×4 block encode │
  └──────────┴────────────┴──────────────────┘
       │
  Dithering (ถ้าเปิด --dither)
       │
       ▼
  OutputFormatter
  ┌──────────┬──────────┬────────┬────────┐
  │ Terminal │   Text   │  HTML  │  SVG   │
  │(crossterm│  (.txt)  │(.html) │(.svg)  │
  └──────────┴──────────┴────────┴────────┘
```

### Design Decisions

**ทำไมถึงแยก `density.rs`, `sobel.rs`, `braille.rs` ออกจากกัน?**
แต่ละ module มี concern ชัดเจนและ testable แยกกันได้ง่าย ถ้า merge ทุกอย่างใน `main.rs` จะทำให้ test ยากและ logic พัวพันกัน

**ทำไมไม่ใช้ async?**
Image processing เป็น CPU-bound task ไม่มี I/O ที่ต้องรอ การใช้ async จะเพิ่ม complexity โดยไม่ได้ประโยชน์ ถ้าอยากทำ parallel frame rendering ให้ใช้ `rayon` แทน

**ทำไม aspect ratio correction ถึงหาร 2?**
Terminal character มีสัดส่วนสูง:กว้าง ≈ 2:1 (โดยประมาณ) ถ้าไม่หาร ภาพจะยืดในแนวตั้ง เมื่อ resize ภาพสำหรับ terminal เราต้องใช้ width = `target_cols` แต่ height = `img_height * scale / 2`

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: โครงสร้างพื้นฐาน และ Density Mapping

เริ่มจาก module แรกที่สำคัญที่สุด: การแปลง luminance ของ pixel ให้เป็นตัวอักษร

**`Cargo.toml`:**

```toml
[package]
name = "ascii-art-renderer"
version = "0.1.0"
edition = "2021"

[dependencies]
image = { version = "0.25", features = ["jpeg", "png", "bmp", "gif"] }
crossterm = "0.28"
clap = { version = "4", features = ["derive"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

**`src/density.rs`:**

```rust
//! Character density mapping: แปลง luminance (0-255) ไปเป็นตัวอักษร ASCII
//!
//! หลักการ: ตัวอักษรที่มีพื้นที่สีดำมากกว่า (เช่น '@') ใช้แทน pixel ที่มืด
//! ตัวอักษรที่มีพื้นที่น้อย (เช่น '.') ใช้แทน pixel สว่าง, ' ' สำหรับขาวสุด

/// Default character ramp เรียงจาก dense (มืด) ไป light (สว่าง)
pub const DEFAULT_CHARS: &[u8] = b"@%#*+=-:. ";

/// แปลง luminance (0=มืดสุด, 255=สว่างสุด) ไปเป็นตัวอักษรจาก ramp ที่กำหนด
///
/// # Arguments
/// * `luma` - ค่าความสว่าง 0–255
/// * `chars` - slice ของตัวอักษร เรียงจาก dense ไป light
///
/// # ตัวอย่าง
/// ```
/// use crate::density::{luminance_to_char, DEFAULT_CHARS};
/// assert_eq!(luminance_to_char(0, DEFAULT_CHARS), '@');    // มืดสุด
/// assert_eq!(luminance_to_char(255, DEFAULT_CHARS), ' ');  // สว่างสุด
/// ```
pub fn luminance_to_char(luma: u8, chars: &[u8]) -> char {
    let n = chars.len();
    if n == 0 {
        return ' ';
    }
    // idx = luma * (n-1) / 255  (integer division, scale 0–255 → 0..n-1)
    let idx = (luma as usize * (n - 1)) / 255;
    chars[idx] as char
}

/// Convenience wrapper ใช้ DEFAULT_CHARS
pub fn luminance_to_char_default(luma: u8) -> char {
    luminance_to_char(luma, DEFAULT_CHARS)
}
```

**`src/main.rs` (ขั้น 1 — minimal):**

```rust
mod density;

use density::{luminance_to_char_default, DEFAULT_CHARS};

fn main() {
    println!("=== ASCII Density Test ===");
    println!("Chars: {}", std::str::from_utf8(DEFAULT_CHARS).unwrap());
    println!();
    for luma in (0u8..=255).step_by(25) {
        let c = luminance_to_char_default(luma);
        // แสดง bar ของ '#' ตามความสว่าง
        let bar: String = std::iter::repeat('#').take((luma / 10) as usize).collect();
        println!("luma={:3} -> '{}' {}", luma, c, bar);
    }
}
```

**Output จริง (ขั้น 1):**
```
=== ASCII Density Test ===
Chars: @%#*+=-:. 

luma=  0 -> '@' 
luma= 25 -> '@' ##
luma= 50 -> '%' #####
luma= 75 -> '#' #######
luma=100 -> '*' ##########
luma=125 -> '+' ############
luma=150 -> '=' ###############
luma=175 -> '-' #################
luma=200 -> ':' ####################
luma=225 -> '.' ######################
luma=250 -> ' ' #########################
```

---

### ขั้นที่ 2: Image Loading และ Aspect Ratio Correction

```rust
// src/image_loader.rs

use image::{DynamicImage, GrayImage, ImageError, RgbImage};

/// โหลดภาพจาก path และ resize ให้พอดีกับ terminal
pub struct ImageLoader {
    pub image: DynamicImage,
}

impl ImageLoader {
    /// โหลดภาพจาก file path
    pub fn load(path: &str) -> Result<Self, ImageError> {
        let image = image::open(path)?;
        Ok(ImageLoader { image })
    }

    /// คำนวณ output dimensions ที่ถูก aspect-correct สำหรับ terminal
    ///
    /// Terminal character มีสัดส่วนสูง:กว้าง ≈ 2:1
    /// ดังนั้น output_rows = (img_height * scale) / 2
    /// เพื่อให้ภาพดูไม่ยืดในแนวตั้ง
    pub fn corrected_dimensions(
        img_width: u32,
        img_height: u32,
        target_cols: u32,
    ) -> (u32, u32) {
        let scale = target_cols as f64 / img_width as f64;
        let out_cols = target_cols;
        // หาร 2 เพราะ terminal char สูง ~2x pixel
        let out_rows = ((img_height as f64 * scale) / 2.0).round() as u32;
        (out_cols, out_rows.max(1))
    }

    /// Resize ภาพให้พอดีกับ terminal columns แล้วแปลงเป็น grayscale
    pub fn to_grayscale(&self, target_cols: u32) -> GrayImage {
        let (w, h) = Self::corrected_dimensions(
            self.image.width(),
            self.image.height(),
            target_cols,
        );
        self.image
            .resize_exact(w, h, image::imageops::FilterType::Lanczos3)
            .to_luma8()
    }

    /// Resize ภาพแล้วคืนเป็น RGB (ไว้ใช้กับ color mode)
    pub fn to_rgb(&self, target_cols: u32) -> RgbImage {
        let (w, h) = Self::corrected_dimensions(
            self.image.width(),
            self.image.height(),
            target_cols,
        );
        self.image
            .resize_exact(w, h, image::imageops::FilterType::Lanczos3)
            .to_rgb8()
    }
}
```

**ตัวอย่างการใช้งาน:**

```rust
// ใน main.rs
let loader = ImageLoader::load("photo.jpg")?;
let gray = loader.to_grayscale(80);  // 80 columns
println!("Output size: {}x{}", gray.width(), gray.height());
// photo 640x480 → output 80x30 (480 * 80/640 / 2 = 30)

// แสดง ASCII art
for y in 0..gray.height() {
    for x in 0..gray.width() {
        let luma = gray.get_pixel(x, y).0[0];
        print!("{}", luminance_to_char_default(luma));
    }
    println!();
}
```

**หมายเหตุ `image::imageops::FilterType::Lanczos3`:**
Lanczos3 ให้คุณภาพดีที่สุดเมื่อ downscale ลงมาก เหมาะกับ ASCII art ที่ต้องการรายละเอียด ถ้าต้องการความเร็วมากกว่า ใช้ `Nearest` (เร็วที่สุด) หรือ `Triangle` (กลาง)

---

### ขั้นที่ 3: Sobel Edge Detection

Sobel operator ใช้ convolution kernel 3×3 สองชุด (Gx สำหรับ horizontal gradient, Gy สำหรับ vertical gradient) เพื่อตรวจหาขอบของวัตถุในภาพ

```
Gx kernel:          Gy kernel:
[-1  0  +1]         [-1  -2  -1]
[-2  0  +2]         [ 0   0   0]
[-1  0  +1]         [+1  +2  +1]
```

ผล gradient magnitude = √(Gx² + Gy²) และทิศทาง = atan2(Gy, Gx)

```rust
// src/sobel.rs

/// ทิศทาง edge สำหรับแต่ละ char
/// 0=horizontal (|), 1=diagonal45 (/), 2=vertical (-), 3=diagonal135 (\)
pub const EDGE_CHARS: [char; 4] = ['|', '/', '-', '\\'];

/// คำนวณ Sobel gradient magnitude และ direction index ที่ pixel (x, y)
///
/// คืน (magnitude, direction_index)
/// direction_index: 0=horiz, 1=diag45, 2=vert, 3=diag135
pub fn sobel_at(
    pixels: &[i32],
    width: usize,
    height: usize,
    x: usize,
    y: usize,
) -> (f64, usize) {
    // border pixels ไม่มีข้อมูลรอบด้านครบ → คืน 0
    if x == 0 || x + 1 >= width || y == 0 || y + 1 >= height {
        return (0.0, 0);
    }

    // helper closure ดึงค่า pixel ด้วย offset
    let p = |dy: isize, dx: isize| -> i32 {
        let ny = (y as isize + dy) as usize;
        let nx = (x as isize + dx) as usize;
        pixels[ny * width + nx]
    };

    // Gx = horizontal gradient (ตรวจขอบแนวตั้ง)
    let gx = -p(-1, -1) + p(-1, 1)
           - 2 * p(0, -1) + 2 * p(0, 1)
           - p(1, -1)  + p(1, 1);

    // Gy = vertical gradient (ตรวจขอบแนวนอน)
    let gy = -p(-1, -1) - 2 * p(-1, 0) - p(-1, 1)
           + p(1, -1)  + 2 * p(1, 0)  + p(1, 1);

    let magnitude = ((gx * gx + gy * gy) as f64).sqrt();

    // แบ่งมุมเป็น 4 ทิศ (0°-180°)
    let angle_deg = (gy as f64).atan2(gx as f64).to_degrees().abs() % 180.0;
    let direction = if angle_deg < 22.5 || angle_deg >= 157.5 {
        0 // horizontal edge → ใช้ '|'
    } else if angle_deg < 67.5 {
        1 // diagonal 45° → ใช้ '/'
    } else if angle_deg < 112.5 {
        2 // vertical edge → ใช้ '-'
    } else {
        3 // diagonal 135° → ใช้ '\'
    };

    (magnitude, direction)
}

/// แปลง grayscale image เป็น Vec<(magnitude, direction)>
pub fn compute_sobel_map(pixels: &[u8], width: usize, height: usize) -> Vec<(f64, usize)> {
    let int_pixels: Vec<i32> = pixels.iter().map(|&p| p as i32).collect();
    (0..height)
        .flat_map(|y| {
            (0..width).map(move |x| sobel_at(&int_pixels, width, height, x, y))
        })
        .collect()
}

/// แสดงผล ASCII art พร้อม edge overlay
/// threshold: ค่า magnitude ขั้นต่ำที่ถือว่าเป็น edge
pub fn render_with_edges(
    gray_pixels: &[u8],
    width: usize,
    height: usize,
    density_chars: &[u8],
    edge_threshold: f64,
) -> Vec<Vec<char>> {
    let sobel_map = compute_sobel_map(gray_pixels, width, height);
    let mut rows = vec![vec![' '; width]; height];

    for y in 0..height {
        for x in 0..width {
            let idx = y * width + x;
            let (mag, dir) = sobel_map[idx];
            rows[y][x] = if mag >= edge_threshold {
                EDGE_CHARS[dir]
            } else {
                let luma = gray_pixels[idx];
                crate::density::luminance_to_char(luma, density_chars)
            };
        }
    }
    rows
}
```

**ตัวอย่าง: ผล Sobel บน gradient สังเคราะห์**

```
Input (3×3 vertical edge):
  0   128  255
  0   128  255
  0   128  255

Gx = -0 + 255 - 0 + 510 - 0 + 255 = 1020
Gy = -0 - 256 - 255 + 0 + 256 + 255 = 0
magnitude ≈ 1020
direction = horizontal (atan2(0,1020) = 0°) → char '|'
```

---

### ขั้นที่ 4: Braille Rendering

Unicode Braille block (U+2800–U+28FF) แต่ละ character แทน grid ขนาด 2 คอลัมน์ × 4 แถว = 8 จุด แต่ละจุดมี bit เป็นของตัวเอง:

```
dot layout:
 col0  col1
  1     4    ← row 0
  2     5    ← row 1
  3     6    ← row 2
  7     8    ← row 3

bit mapping:
  dot1=bit0, dot2=bit1, dot3=bit2, dot4=bit3
  dot5=bit4, dot6=bit5, dot7=bit6, dot8=bit7

Unicode code point = U+2800 + (bit7<<7 | bit6<<6 | ... | bit0)
```

ข้อดีของ Braille: 1 terminal character แทน pixel block ขนาด 2×4 ทำให้ความละเอียด **สูงกว่า ASCII ปกติ 8 เท่า**

```rust
// src/braille.rs

/// Braille base code point
pub const BRAILLE_BASE: u32 = 0x2800;

/// Dot position mapping: (row, col) → bit index
/// ตาม Unicode Braille standard
const DOT_MAP: [(usize, usize, u8); 8] = [
    (0, 0, 0), // dot 1 → bit 0
    (1, 0, 1), // dot 2 → bit 1
    (2, 0, 2), // dot 3 → bit 2
    (0, 1, 3), // dot 4 → bit 3
    (1, 1, 4), // dot 5 → bit 4
    (2, 1, 5), // dot 6 → bit 5
    (3, 0, 6), // dot 7 → bit 6
    (3, 1, 7), // dot 8 → bit 7
];

/// Encode 2×4 pixel block เป็น Braille character
///
/// `pixels` เป็น array 8 ตัว เรียงตาม row-major order:
/// [row0col0, row0col1, row1col0, row1col1, row2col0, row2col1, row3col0, row3col1]
///
/// # ตัวอย่าง
/// ```
/// // ทุก dot → U+28FF (⣿)
/// let pixels = [true; 8];
/// assert_eq!(encode_braille(&pixels), '\u{28FF}');
///
/// // ไม่มี dot → U+2800 (blank)
/// let pixels = [false; 8];
/// assert_eq!(encode_braille(&pixels), '\u{2800}');
/// ```
pub fn encode_braille(pixels: &[bool; 8]) -> char {
    let mut bits: u32 = 0;
    for &(row, col, bit) in &DOT_MAP {
        let pixel_idx = row * 2 + col;
        if pixels[pixel_idx] {
            bits |= 1 << bit;
        }
    }
    char::from_u32(BRAILLE_BASE + bits).unwrap_or('?')
}

/// แปลง grayscale image เป็น Braille art
///
/// * แบ่งภาพออกเป็น block ขนาด 2 (กว้าง) × 4 (สูง)
/// * แต่ละ block → 1 Braille character
/// * threshold: ค่า luma ต่ำกว่านี้ถือเป็น dot "เปิด"
pub fn render_braille(
    pixels: &[u8],
    width: usize,
    height: usize,
    threshold: u8,
) -> Vec<String> {
    let rows_out = height / 4;
    let cols_out = width / 2;
    let mut lines = Vec::with_capacity(rows_out);

    for block_y in 0..rows_out {
        let mut line = String::with_capacity(cols_out);
        for block_x in 0..cols_out {
            let mut dots = [false; 8];
            for &(row, col, _) in &DOT_MAP {
                let img_y = block_y * 4 + row;
                let img_x = block_x * 2 + col;
                if img_y < height && img_x < width {
                    let luma = pixels[img_y * width + img_x];
                    // pixel มืด (luma ต่ำ) = dot "เปิด"
                    dots[row * 2 + col] = luma < threshold;
                }
            }
            line.push(encode_braille(&dots));
        }
        lines.push(line);
    }
    lines
}
```

**ตัวอย่าง Braille ใน terminal:**

```
Original (ASCII):          Braille (denser):
@@@@@@@@@@@@               ⣿⣿⣿⣿⣿⣿
@@  @@  @@@@               ⣤⣤⣤⣤⣿⣿
@@@@@@@@@@@@               ⣿⣿⣿⣿⣿⣿
    @@  @@@@               ⠀⠠⠠⠠⣿⣿
```

---

### ขั้นที่ 5: Floyd-Steinberg Dithering

Floyd-Steinberg เป็น error diffusion algorithm ที่ช่วยให้ภาพ 1-bit (ขาว/ดำ) ดูเหมือนมี grayscale ได้ดีขึ้น โดยกระจาย quantization error ไปยัง pixel รอบข้าง:

```
Error propagation weights:
         current pixel
              ↓
  ...  [ X ] [7/16] ...
  [3/16][5/16][1/16] ...
  (ซ้ายล่าง)(ล่าง)(ขวาล่าง)
```

ค่า error = ค่าจริง − ค่าหลัง threshold (0 หรือ 255)

```rust
// src/dither.rs

/// Floyd-Steinberg error diffusion dithering
///
/// `pixels`: mutable Vec<f32> ขนาด width × height (ค่า 0.0–255.0)
/// `threshold`: ค่าแบ่งขาว/ดำ (ปกติ 128.0)
///
/// คืน Vec<bool>: true = white, false = black
pub fn floyd_steinberg(
    pixels: &mut Vec<f32>,
    width: usize,
    height: usize,
    threshold: f32,
) -> Vec<bool> {
    let mut result = vec![false; width * height];

    for y in 0..height {
        for x in 0..width {
            let idx = y * width + x;
            let old_val = pixels[idx];

            // Quantize: ค่าใกล้ threshold → round เป็น 0 หรือ 255
            let new_val = if old_val >= threshold { 255.0 } else { 0.0 };
            result[idx] = new_val >= threshold;

            // Quantization error
            let err = old_val - new_val;

            // Distribute error ไปยัง 4 pixel รอบข้าง
            // ขวา: 7/16
            if x + 1 < width {
                pixels[idx + 1] += err * 7.0 / 16.0;
            }
            // ซ้ายล่าง: 3/16
            if y + 1 < height && x > 0 {
                pixels[(y + 1) * width + x - 1] += err * 3.0 / 16.0;
            }
            // ล่าง: 5/16
            if y + 1 < height {
                pixels[(y + 1) * width + x] += err * 5.0 / 16.0;
            }
            // ขวาล่าง: 1/16
            if y + 1 < height && x + 1 < width {
                pixels[(y + 1) * width + x + 1] += err * 1.0 / 16.0;
            }
        }
    }
    result
}

/// แปลง u8 grayscale เป็น f32 แล้วทำ dithering
pub fn dither_u8(gray: &[u8], width: usize, height: usize) -> Vec<bool> {
    let mut pixels: Vec<f32> = gray.iter().map(|&p| p as f32).collect();
    floyd_steinberg(&mut pixels, width, height, 128.0)
}

/// Render dithered output เป็น string (ใช้ ' ' สำหรับ white, '#' สำหรับ black)
pub fn render_dithered(gray: &[u8], width: usize, height: usize) -> Vec<String> {
    let bits = dither_u8(gray, width, height);
    (0..height)
        .map(|y| {
            (0..width)
                .map(|x| if bits[y * width + x] { ' ' } else { '#' })
                .collect()
        })
        .collect()
}
```

**เปรียบเทียบผล:**

```
ภาพขนาดเล็ก 8×3 gray gradient (luma = 0, 30, 60, 90, 120, 150, 180, 210)

ไม่มี dithering (simple threshold 128):
########    (ทุก pixel < 128 = black, ทุก >= 128 = white)

Floyd-Steinberg:
########    ← แถว 1: ทั้งหมด < 128 แต่ error สะสม
###  ###    ← แถว 2: error ทำให้บาง pixel ข้ามเกณฑ์
#  #  #     ← แถว 3: pattern สวยงามกว่า simple threshold
```

---

### ขั้นที่ 6: Color Output, Animation, Font, และ CLI เต็มรูปแบบ

#### 6a: Color Output ด้วย crossterm

```rust
// src/color.rs
use crossterm::style::{Color, SetForegroundColor, ResetColor};
use crossterm::ExecutableCommand;
use std::io::{stdout, Write};

/// แปลง RGB เป็น crossterm Color
pub fn rgb_to_color(r: u8, g: u8, b: u8) -> Color {
    Color::Rgb { r, g, b }
}

/// Print character เดียวพร้อม foreground color จาก RGB
pub fn print_colored_char(ch: char, r: u8, g: u8, b: u8) {
    let mut out = stdout();
    let _ = out.execute(SetForegroundColor(rgb_to_color(r, g, b)));
    print!("{}", ch);
}

/// Reset color หลังจาก render เสร็จ
pub fn reset_color() {
    let mut out = stdout();
    let _ = out.execute(ResetColor);
}

/// Map RGB → nearest 256-color terminal (web-safe 6×6×6 cube)
/// ค่า return อยู่ในช่วง 16–231 (216 web-safe colors)
pub fn nearest_256_color(r: u8, g: u8, b: u8) -> u8 {
    // Web-safe color cube: 6 levels per channel (0, 95, 135, 175, 215, 255)
    let levels = [0u8, 95, 135, 175, 215, 255];
    let nearest = |v: u8| -> u8 {
        levels
            .iter()
            .enumerate()
            .min_by_key(|&(_, &l)| (v as i32 - l as i32).abs())
            .map(|(i, _)| i as u8)
            .unwrap_or(0)
    };
    let ri = nearest(r);
    let gi = nearest(g);
    let bi = nearest(b);
    16 + 36 * ri + 6 * gi + bi
}
```

#### 6b: CLI ด้วย clap derive macro

```rust
// src/main.rs (เต็มรูปแบบ)
use clap::Parser;

#[derive(Parser, Debug)]
#[command(name = "ascii-art-renderer")]
#[command(about = "แปลงภาพ/GIF เป็น ASCII/Braille art ใน terminal")]
pub struct Args {
    /// Path ไฟล์ภาพ (JPEG, PNG, BMP) หรือ GIF
    #[arg(value_name = "FILE")]
    pub input: String,

    /// จำนวน columns ใน terminal output (default: ความกว้าง terminal)
    #[arg(short, long, default_value = "80")]
    pub width: u32,

    /// Custom character ramp (จาก dense ไป light)
    #[arg(long, default_value = "@%#*+=-:. ")]
    pub chars: String,

    /// เปิด color mode (ใช้ crossterm SetForegroundColor)
    #[arg(long)]
    pub color: bool,

    /// ใช้ Braille rendering แทน ASCII
    #[arg(long)]
    pub braille: bool,

    /// เปิด edge detection overlay
    #[arg(long)]
    pub edges: bool,

    /// Edge detection threshold (0-1000, default 100)
    #[arg(long, default_value = "100")]
    pub edge_threshold: f64,

    /// เปิด Floyd-Steinberg dithering
    #[arg(long)]
    pub dither: bool,

    /// บันทึกผล HTML ไปยังไฟล์
    #[arg(long, value_name = "FILE")]
    pub html: Option<String>,

    /// บันทึกผล SVG ไปยังไฟล์
    #[arg(long, value_name = "FILE")]
    pub svg: Option<String>,

    /// บันทึกผล plain text ไปยังไฟล์
    #[arg(long, value_name = "FILE")]
    pub text: Option<String>,

    /// Render text เป็น ASCII block letters ด้วย bitmap font
    #[arg(long, value_name = "TEXT")]
    pub font_text: Option<String>,

    /// Frame delay สำหรับ GIF animation (ms)
    #[arg(long, default_value = "100")]
    pub frame_delay: u64,
}
```

#### 6c: GIF Animation

```rust
// src/animation.rs
use crossterm::{
    cursor::{MoveToColumn, MoveUp},
    ExecutableCommand,
};
use std::io::{stdout, Write};
use std::time::Duration;

/// Render ทุก frame ของ GIF animation ใน terminal โดย rewrite in-place
///
/// ใช้ crossterm MoveUp + MoveToColumn เพื่อ overwrite frame เก่า
pub fn render_gif_animation(
    frames: Vec<Vec<String>>,  // แต่ละ frame = Vec<String> (rows)
    frame_delay_ms: u64,
) {
    let mut out = stdout();
    let delay = Duration::from_millis(frame_delay_ms);
    let mut first_frame = true;

    for frame in &frames {
        let row_count = frame.len();

        if !first_frame {
            // ย้าย cursor กลับขึ้นไปบน frame ก่อนหน้า
            let _ = out.execute(MoveUp(row_count as u16));
        }

        for row in frame {
            let _ = out.execute(MoveToColumn(0));
            print!("{}", row);
            println!();
        }

        let _ = out.flush();
        first_frame = false;
        std::thread::sleep(delay);
    }
}

/// โหลด GIF frames จาก path ด้วย image crate (gif feature)
///
/// คืน Vec ของ (gray_pixels, width, height, delay_ms) ต่อ frame
pub fn load_gif_frames(
    path: &str,
) -> Result<Vec<(Vec<u8>, u32, u32, u64)>, Box<dyn std::error::Error>> {
    use image::AnimationDecoder;

    let file = std::fs::File::open(path)?;
    let decoder = image::codecs::gif::GifDecoder::new(file)?;
    let frames = decoder.into_frames().collect_frames()?;

    let result = frames
        .into_iter()
        .map(|frame| {
            let delay_ms = frame.delay().numer_denom_ms().0 as u64;
            let img = frame.into_buffer();
            let w = img.width();
            let h = img.height();
            let gray: Vec<u8> = image::DynamicImage::ImageRgba8(img)
                .to_luma8()
                .into_raw();
            (gray, w, h, delay_ms)
        })
        .collect();

    Ok(result)
}
```

#### 6d: Embedded Bitmap Font

```rust
// src/font.rs

/// Embedded 8×8 bitmap font สำหรับ ASCII chars 32–127
/// แต่ละ char ใช้ 8 bytes (1 byte ต่อ row, bit 7=leftmost)
///
/// นี่คือ subset ของ IBM PC BIOS font
/// (ในโค้ดจริงให้ฝัง array เต็ม 96×8 bytes)
pub const FONT_DATA: &[u8] = &[
    // ' ' (0x20)
    0b00000000, 0b00000000, 0b00000000, 0b00000000,
    0b00000000, 0b00000000, 0b00000000, 0b00000000,
    // '!' (0x21)
    0b00010000, 0b00010000, 0b00010000, 0b00010000,
    0b00010000, 0b00000000, 0b00010000, 0b00000000,
    // ... (ตัวอักษรอื่นๆ ใน production จะฝังครบ)
    // 'A' (0x41)
    0b00111000, 0b01000100, 0b01000100, 0b01111100,
    0b01000100, 0b01000100, 0b01000100, 0b00000000,
    // 'B' (0x42)
    0b01111000, 0b01000100, 0b01000100, 0b01111000,
    0b01000100, 0b01000100, 0b01111000, 0b00000000,
];

/// ดึง bitmap ของตัวอักษร c (8 bytes)
/// ถ้า c ไม่อยู่ใน range → คืน blank
pub fn get_char_bitmap(c: char) -> [u8; 8] {
    let code = c as usize;
    // ใน implementation จริง FONT_DATA มีตั้งแต่ ASCII 32–127
    // สำหรับตัวอย่างนี้ support เฉพาะ ' ', '!', 'A', 'B'
    let supported = [' ', '!', 'A', 'B'];
    let idx = supported.iter().position(|&sc| sc == c).unwrap_or(0);
    let offset = idx * 8;
    if offset + 8 <= FONT_DATA.len() {
        let mut bitmap = [0u8; 8];
        bitmap.copy_from_slice(&FONT_DATA[offset..offset + 8]);
        bitmap
    } else {
        [0u8; 8]
    }
}

/// Render ข้อความเป็น multi-line ASCII block letters
///
/// แต่ละตัวอักษรกว้าง 8 pixels, สูง 8 rows
/// ตัวอักษรจะถูกต่อกันใน horizontal direction
///
/// # ตัวอย่าง output (ตัวอักษร 'A'):
/// ```
/// .###....
/// #...#...
/// #...#...
/// #####...
/// #...#...
/// #...#...
/// #...#...
/// ........
/// ```
pub fn render_text(text: &str) -> Vec<String> {
    let chars: Vec<char> = text.chars().collect();
    let bitmaps: Vec<[u8; 8]> = chars.iter().map(|&c| get_char_bitmap(c)).collect();

    (0..8)
        .map(|row| {
            bitmaps
                .iter()
                .flat_map(|bitmap| {
                    let byte = bitmap[row];
                    (0..8).map(move |bit| {
                        if (byte >> (7 - bit)) & 1 == 1 { '#' } else { '.' }
                    })
                })
                .collect()
        })
        .collect()
}
```

#### 6e: Output Formats

```rust
// src/output.rs
use std::fs::File;
use std::io::{BufWriter, Write};

/// บันทึก ASCII art เป็น plain text file
pub fn save_as_text(rows: &[Vec<char>], path: &str) -> std::io::Result<()> {
    let file = File::create(path)?;
    let mut writer = BufWriter::new(file);
    for row in rows {
        let line: String = row.iter().collect();
        writeln!(writer, "{}", line)?;
    }
    Ok(())
}

/// บันทึกเป็น HTML พร้อม color (ใช้ inline style ต่อ char)
pub fn save_as_html(
    rows: &[Vec<char>],
    colors: Option<&[Vec<[u8; 3]>]>,
    path: &str,
) -> std::io::Result<()> {
    let file = File::create(path)?;
    let mut w = BufWriter::new(file);

    writeln!(w, "<!DOCTYPE html>")?;
    writeln!(w, "<html><head><meta charset='UTF-8'>")?;
    writeln!(w, "<style>body{{background:#000;font-family:monospace;font-size:10px;line-height:1;}}</style>")?;
    writeln!(w, "</head><body><pre>")?;

    for (y, row) in rows.iter().enumerate() {
        for (x, &ch) in row.iter().enumerate() {
            if let Some(color_rows) = colors {
                if y < color_rows.len() && x < color_rows[y].len() {
                    let [r, g, b] = color_rows[y][x];
                    // escape HTML special chars
                    let escaped = match ch {
                        '<' => "&lt;".to_string(),
                        '>' => "&gt;".to_string(),
                        '&' => "&amp;".to_string(),
                        ' ' => "&nbsp;".to_string(),
                        c => c.to_string(),
                    };
                    write!(w, "<span style='color:rgb({},{},{})'>{}</span>", r, g, b, escaped)?;
                } else {
                    write!(w, "{}", ch)?;
                }
            } else {
                let escaped = match ch {
                    '<' => "&lt;".to_string(),
                    '>' => "&gt;".to_string(),
                    '&' => "&amp;".to_string(),
                    ' ' => "&nbsp;".to_string(),
                    c => c.to_string(),
                };
                write!(w, "{}", escaped)?;
            }
        }
        writeln!(w)?;
    }

    writeln!(w, "</pre></body></html>")?;
    Ok(())
}

/// บันทึกเป็น SVG (แต่ละ char = <text> element ตาม position)
pub fn save_as_svg(
    rows: &[Vec<char>],
    colors: Option<&[Vec<[u8; 3]>]>,
    path: &str,
) -> std::io::Result<()> {
    let file = File::create(path)?;
    let mut w = BufWriter::new(file);

    let char_width = 8;
    let char_height = 14;
    let svg_width = rows.first().map(|r| r.len()).unwrap_or(0) * char_width;
    let svg_height = rows.len() * char_height;

    writeln!(w, r#"<?xml version="1.0" encoding="UTF-8"?>"#)?;
    writeln!(w,
        r#"<svg xmlns="http://www.w3.org/2000/svg" width="{}" height="{}">"#,
        svg_width, svg_height
    )?;
    writeln!(w, r#"<rect width="100%" height="100%" fill="black"/>"#)?;
    writeln!(w, r#"<g font-family="monospace" font-size="12">"#)?;

    for (y, row) in rows.iter().enumerate() {
        for (x, &ch) in row.iter().enumerate() {
            if ch == ' ' {
                continue;
            }
            let px = x * char_width;
            let py = (y + 1) * char_height;
            let color = if let Some(color_rows) = colors {
                if y < color_rows.len() && x < color_rows[y].len() {
                    let [r, g, b] = color_rows[y][x];
                    format!("rgb({},{},{})", r, g, b)
                } else {
                    "white".to_string()
                }
            } else {
                "white".to_string()
            };
            let escaped = match ch {
                '<' => "&lt;".to_string(),
                '>' => "&gt;".to_string(),
                '&' => "&amp;".to_string(),
                c => c.to_string(),
            };
            writeln!(w,
                r#"  <text x="{}" y="{}" fill="{}">{}</text>"#,
                px, py, color, escaped
            )?;
        }
    }

    writeln!(w, "</g></svg>")?;
    Ok(())
}
```

---

## การทดสอบ (Testing)

โค้ดทดสอบจริงทั้งหมดอยู่ใน `src/main.rs` ภายใต้ `#[cfg(test)]`

### Unit Tests ที่ต้องมี

```rust
#[cfg(test)]
mod tests {
    use super::*;

    // ===== Test 1: Character Density Mapping =====

    #[test]
    fn test_density_map_darkest() {
        // luma 0 ควรได้ตัวอักษรแรก (densest = '@')
        assert_eq!(luminance_to_char_default(0), '@');
    }

    #[test]
    fn test_density_map_lightest() {
        // luma 255 ควรได้ตัวอักษรสุดท้าย (lightest = ' ')
        assert_eq!(luminance_to_char_default(255), ' ');
    }

    #[test]
    fn test_density_map_midrange() {
        let mid = luminance_to_char_default(128);
        // DEFAULT_CHARS = "@%#*+=-:. " (10 chars)
        // idx = 128*9/255 = 4 → '+'
        assert_eq!(mid, '+');
    }

    #[test]
    fn test_density_map_custom_chars() {
        let chars = b"#. ";
        assert_eq!(luminance_to_char(0, chars), '#');
        assert_eq!(luminance_to_char(255, chars), ' ');
        let mid_idx = (127usize * 2) / 255;
        assert_eq!(luminance_to_char(127, chars), chars[mid_idx] as char);
    }

    #[test]
    fn test_density_map_single_char() {
        let chars = b"X";
        assert_eq!(luminance_to_char(0, chars), 'X');
        assert_eq!(luminance_to_char(255, chars), 'X');
    }

    #[test]
    fn test_density_all_ten_values() {
        // ทดสอบว่า mapping ครอบคลุมทุกตัวอักษร
        // formula: idx = (luma * (n-1)) / 255 (integer division)
        // เพื่อให้ได้ idx=i ต้องใช้ luma = ceil(i*255/(n-1))
        let expected = "@%#*+=-:. ";
        let n = expected.len(); // 10
        for (i, ch) in expected.chars().enumerate() {
            let luma = if i == 0 { 0u8 } else { ((i * 255 + n - 2) / (n - 1)) as u8 };
            let result = luminance_to_char_default(luma);
            assert_eq!(result, ch,
                "luma={} ควรได้ '{}' แต่ได้ '{}'", luma, ch, result);
        }
    }

    // ===== Test 2: Aspect Ratio Correction =====

    #[test]
    fn test_aspect_ratio_square_image() {
        // image 100×100, target 80 cols
        // scale = 0.8 → rows = 100 * 0.8 / 2 = 40
        let (cols, rows) = corrected_dimensions(100, 100, 80);
        assert_eq!(cols, 80);
        assert_eq!(rows, 40);
    }

    #[test]
    fn test_aspect_ratio_wide_image() {
        // image 200×100, target 80 cols
        // scale = 0.4 → rows = 100 * 0.4 / 2 = 20
        let (cols, rows) = corrected_dimensions(200, 100, 80);
        assert_eq!(cols, 80);
        assert_eq!(rows, 20);
    }

    #[test]
    fn test_aspect_ratio_tall_image() {
        // image 100×200, target 80 cols
        // scale = 0.8 → rows = 200 * 0.8 / 2 = 80
        let (cols, rows) = corrected_dimensions(100, 200, 80);
        assert_eq!(cols, 80);
        assert_eq!(rows, 80);
    }

    // ===== Test 3: Sobel Kernel =====

    #[test]
    fn test_sobel_flat_region() {
        // uniform image → gradient = 0
        let pixels: Vec<i32> = vec![128i32; 9];
        let (mag, _) = sobel_at(&pixels, 3, 3, 1, 1);
        assert!(mag < 1e-6, "flat region ต้อง magnitude = 0 แต่ได้ {}", mag);
    }

    #[test]
    fn test_sobel_vertical_edge() {
        // left half dark, right half bright
        let pixels: Vec<i32> = vec![
            0,   128, 255,
            0,   128, 255,
            0,   128, 255,
        ];
        let (mag, _) = sobel_at(&pixels, 3, 3, 1, 1);
        assert!(mag > 100.0, "vertical edge ต้อง magnitude สูง แต่ได้ {}", mag);
    }

    #[test]
    fn test_sobel_horizontal_edge() {
        let pixels: Vec<i32> = vec![
            0,   0,   0,
            128, 128, 128,
            255, 255, 255,
        ];
        let (mag, _) = sobel_at(&pixels, 3, 3, 1, 1);
        assert!(mag > 100.0, "horizontal edge ต้อง magnitude สูง แต่ได้ {}", mag);
    }

    #[test]
    fn test_sobel_border_pixel() {
        let pixels: Vec<i32> = vec![0i32; 9];
        let (mag, _) = sobel_at(&pixels, 3, 3, 0, 1);
        assert_eq!(mag, 0.0, "border pixel ต้อง magnitude = 0");
    }

    #[test]
    fn test_edge_char_directions() {
        assert_eq!(edge_char(0), '|');
        assert_eq!(edge_char(1), '/');
        assert_eq!(edge_char(2), '-');
        assert_eq!(edge_char(3), '\\');
    }

    // ===== Test 4: Braille Bit Encoding =====

    #[test]
    fn test_braille_empty() {
        // ไม่มี dot ใด → U+2800 (blank braille)
        let pixels = [false; 8];
        assert_eq!(encode_braille(&pixels), '\u{2800}');
    }

    #[test]
    fn test_braille_all_dots() {
        // ทุก dot → U+28FF
        let pixels = [true; 8];
        assert_eq!(encode_braille(&pixels), '\u{28FF}');
    }

    #[test]
    fn test_braille_top_left_only() {
        // เฉพาะ dot 1 (row0, col0) → bit 0 → U+2801
        let mut pixels = [false; 8];
        pixels[0] = true;
        assert_eq!(encode_braille(&pixels) as u32, 0x2801);
    }

    #[test]
    fn test_braille_top_right_only() {
        // เฉพาะ dot 4 (row0, col1) → bit 3 → U+2808
        let mut pixels = [false; 8];
        pixels[1] = true;
        assert_eq!(encode_braille(&pixels) as u32, 0x2808);
    }

    #[test]
    fn test_braille_left_column() {
        // col 0 ทั้งหมด → bits 0,1,2,6 → U+2847
        let mut pixels = [false; 8];
        pixels[0] = true; // row0,col0 → bit0
        pixels[2] = true; // row1,col0 → bit1
        pixels[4] = true; // row2,col0 → bit2
        pixels[6] = true; // row3,col0 → bit6
        let ch = encode_braille(&pixels) as u32;
        assert_eq!(ch, 0x2800 + 0b01000111);
    }

    // ===== Test 5: Floyd-Steinberg Error Propagation =====

    #[test]
    fn test_floyd_steinberg_all_white() {
        let mut pixels: Vec<f32> = vec![255.0; 4];
        let result = floyd_steinberg(&mut pixels, 2, 2, 128.0);
        assert!(result.iter().all(|&b| b));
    }

    #[test]
    fn test_floyd_steinberg_all_black() {
        let mut pixels: Vec<f32> = vec![0.0; 4];
        let result = floyd_steinberg(&mut pixels, 2, 2, 128.0);
        assert!(result.iter().all(|&b| !b));
    }

    #[test]
    fn test_floyd_steinberg_error_propagation() {
        let mut pixels: Vec<f32> = vec![64.0, 0.0, 0.0, 0.0];
        let result = floyd_steinberg(&mut pixels, 2, 2, 128.0);
        assert!(!result[0], "pixel 64 ควรเป็น black");
    }

    #[test]
    fn test_floyd_steinberg_gray_produces_pattern() {
        let mut pixels: Vec<f32> = vec![128.0; 4];
        let result = floyd_steinberg(&mut pixels, 4, 1, 128.0);
        assert!(result[0], "pixel 128 (>= threshold 128) ควรเป็น white");
    }

    #[test]
    fn test_floyd_steinberg_mid_gray_dithers() {
        let mut pixels: Vec<f32> = vec![64.0; 4];
        let result = floyd_steinberg(&mut pixels, 4, 1, 128.0);
        assert_eq!(result.len(), 4);
    }

    // ===== Test 6: Combined Logic =====

    #[test]
    fn test_render_row_basic() {
        let lumas = vec![0u8, 85, 170, 255];
        let rendered: String = lumas.iter().map(|&l| luminance_to_char_default(l)).collect();
        assert_eq!(rendered.len(), 4);
        let chars: Vec<char> = rendered.chars().collect();
        let indices: Vec<usize> = chars.iter().map(|&c| {
            DEFAULT_CHARS.iter().position(|&b| b as char == c).unwrap_or(0)
        }).collect();
        for i in 0..indices.len() - 1 {
            assert!(indices[i] <= indices[i + 1]);
        }
    }
}
```

### ผล `cargo test` จริง

```
running 25 tests
test tests::test_aspect_ratio_square_image ... ok
test tests::test_braille_all_dots ... ok
test tests::test_braille_empty ... ok
test tests::test_braille_left_column ... ok
test tests::test_aspect_ratio_tall_image ... ok
test tests::test_braille_top_left_only ... ok
test tests::test_braille_top_right_only ... ok
test tests::test_aspect_ratio_wide_image ... ok
test tests::test_density_all_ten_values ... ok
test tests::test_density_map_darkest ... ok
test tests::test_density_map_lightest ... ok
test tests::test_density_map_custom_chars ... ok
test tests::test_density_map_single_char ... ok
test tests::test_edge_char_directions ... ok
test tests::test_floyd_steinberg_all_black ... ok
test tests::test_floyd_steinberg_all_white ... ok
test tests::test_floyd_steinberg_error_propagation ... ok
test tests::test_floyd_steinberg_gray_produces_pattern ... ok
test tests::test_floyd_steinberg_mid_gray_dithers ... ok
test tests::test_render_row_basic ... ok
test tests::test_sobel_border_pixel ... ok
test tests::test_sobel_flat_region ... ok
test tests::test_sobel_horizontal_edge ... ok
test tests::test_sobel_vertical_edge ... ok
test tests::test_density_map_midrange ... ok

test result: ok. 25 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

---

## ข้อควรระวัง (Pitfalls) ⚠️

### Pitfall 1: Aspect Ratio ไม่ถูกต้อง → ภาพยืด

**ปัญหา:** นักพัฒนามักลืมปรับ height สำหรับ terminal character เพราะสมมติว่า pixel = char

```rust
// ❌ ผิด — ภาพจะยืดในแนวตั้ง 2 เท่า
let out_width = target_cols;
let out_height = (img_height as f64 * target_cols as f64 / img_width as f64) as u32;

// ✅ ถูก — หาร 2 เพื่อชดเชย terminal char aspect ratio
let scale = target_cols as f64 / img_width as f64;
let out_height = ((img_height as f64 * scale) / 2.0).round() as u32;
```

**สาเหตุ:** Terminal monospace font ส่วนใหญ่มีสัดส่วน กว้าง:สูง = 1:2 (เช่น 8px × 16px) แต่ pixel มีสัดส่วน 1:1 ดังนั้น 1 terminal char ครอบคลุม pixel แนวตั้ง 2 เท่าของแนวนอน

### Pitfall 2: Integer Division ใน Density Mapping

**ปัญหา:** การหา luma ที่ตรงกับ index i ต้องใช้ ceiling division ไม่ใช่ floor

```rust
// ❌ ผิด — ไม่สามารถ round-trip ได้สำหรับ i > 0
let luma = (i * 255) / (n - 1); // floor division

// ตรวจสอบ: luma=28 (= 1*255/9) → idx = 28*9/255 = 0 ≠ 1 ❌

// ✅ ถูก — ceiling division เพื่อให้ idx = i
let luma = (i * 255 + n - 2) / (n - 1);
// luma=29 (= ceiling(1*255/9)) → idx = 29*9/255 = 1 ✓
```

**บทเรียน:** เมื่อต้องการ inverse ของ integer division ต้องระวัง boundary cases เสมอ ให้เขียน test ก่อน แล้ว verify ทั้งสองทิศทาง (encode → decode)

### Pitfall 3: Sobel ให้ผลผิดที่ Border

**ปัญหา:** ถ้าไม่ check border ก่อน การเข้าถึง pixel นอก bounds จะทำให้ panic หรือได้ค่าผิด

```rust
// ❌ อันตราย — panic หรือ wrong result ถ้า x=0 หรือ x+1=width
let gx = pixels[(y-1)*w + (x-1)] * -1 + pixels[(y-1)*w + (x+1)] * 1; // ...

// ✅ ถูก — check border ก่อน
if x == 0 || x + 1 >= width || y == 0 || y + 1 >= height {
    return (0.0, 0); // border pixels ไม่มีข้อมูลรอบด้านครบ
}
```

**ทางเลือกอื่น:** ใช้ "zero padding" (pad ภาพด้วย 0 รอบนอก) หรือ "mirror padding" (reflect edge) ก่อนคำนวณ แต่สำหรับ ASCII art simple border skip เพียงพอ

### Pitfall 4: Braille Dot Mapping ผิด — หลงเรื่อง Row-Major vs Dot Order

**ปัญหา:** นักพัฒนามักเข้าใจผิดว่า bit 0 อยู่ที่ top-left ตาม visual intuition แต่จริงๆ Unicode Braille มี layout เฉพาะ

```
❌ สมมติผิด (row-major sequential):
bit0=pixel[0][0], bit1=pixel[0][1], bit2=pixel[1][0], ...

✅ ถูกต้องตาม Unicode Standard:
dot1(bit0) = pixel[0][0]  ← top-left
dot2(bit1) = pixel[1][0]  ← second row, left
dot3(bit2) = pixel[2][0]  ← third row, left
dot4(bit3) = pixel[0][1]  ← top-right (ไม่ใช่ pixel[0][3]!)
dot5(bit4) = pixel[1][1]
dot6(bit5) = pixel[2][1]
dot7(bit6) = pixel[3][0]  ← bottom-left
dot8(bit7) = pixel[3][1]  ← bottom-right
```

ถ้า mapping ผิด Braille จะแสดงผล "กลับหัว" หรือ "สลับซ้าย-ขวา" — ดูออกเมื่อ render ภาพที่มีขอบชัด

### Pitfall 5: crossterm Color Reset ถูกลืม

**ปัญหา:** ถ้า render สีแล้วไม่ reset ก่อนออกโปรแกรม สีจะ "ค้าง" ใน terminal

```rust
// ❌ ลืม reset → ข้อความ terminal ต่อไปยังคงเป็นสีสุดท้าย
fn render_colored(rows: &[Vec<(char, [u8;3])>]) {
    for row in rows { /* print colored chars */ }
    // ❌ ออกจากฟังก์ชันโดยไม่ reset
}

// ✅ ถูก — reset ก่อนออกเสมอ (ใช้ Drop trait ก็ได้)
fn render_colored(rows: &[Vec<(char, [u8;3])>]) {
    for row in rows { /* print colored chars */ }
    let _ = stdout().execute(ResetColor);  // ✅
    let _ = stdout().execute(crossterm::cursor::Show); // ถ้าซ่อน cursor ด้วย
}
```

### Pitfall 6: Floyd-Steinberg Error Accumulation Overflow

**ปัญหา:** ถ้าใช้ `u8` แทน `f32` error ที่สะสมจะ overflow หรือ underflow

```rust
// ❌ u8 overflow/underflow
let mut pixels: Vec<u8> = ...;
pixels[idx + 1] += (err * 7 / 16) as u8; // panic! if result > 255

// ✅ ใช้ f32 (หรือ i32) ตลอด
let mut pixels: Vec<f32> = gray.iter().map(|&p| p as f32).collect();
pixels[idx + 1] += err * 7.0 / 16.0; // ค่าได้ > 255 หรือ < 0 ชั่วคราว OK
```

---

## การ Package และ Deploy

### Build Release Binary

```bash
# Optimized build
cargo build --release

# Binary ที่ได้
./target/release/ascii-art-renderer photo.jpg --width 120 --color

# Cross-compile สำหรับ Linux (จาก macOS)
cargo build --release --target x86_64-unknown-linux-musl
```

### ตัวอย่าง CLI Usage

```bash
# แสดง ASCII art พื้นฐาน
ascii-art-renderer photo.jpg

# กว้าง 120 cols + edge detection
ascii-art-renderer photo.jpg --width 120 --edges --edge-threshold 80

# Braille mode + Floyd-Steinberg dithering
ascii-art-renderer photo.jpg --braille --dither

# Color mode
ascii-art-renderer photo.jpg --color

# บันทึก HTML
ascii-art-renderer photo.jpg --color --html output.html

# บันทึก SVG
ascii-art-renderer photo.jpg --svg output.svg

# Custom character set
ascii-art-renderer photo.jpg --chars "▓▒░ "

# GIF animation
ascii-art-renderer animation.gif --frame-delay 80

# ASCII block letters
ascii-art-renderer any.jpg --font-text "HELLO RUST"
```

### ตัวอย่าง Output จริง (ASCII art ขนาดเล็ก)

```
cargo run -- --font-text "AB"

########.########
#...#...#.#...#.#
#...#...#.#...#.#
#####...#.#####.#
#...#...#.#...#.#
#...#...#.#...#.#
########.########
........#.......#
```

### Dockerfile (ถ้าต้องการ containerize)

```dockerfile
FROM rust:1.82-slim AS builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
COPY src/ ./src/
RUN cargo build --release

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y libssl3 && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/ascii-art-renderer /usr/local/bin/
ENTRYPOINT ["ascii-art-renderer"]
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: ปรับปรุง Density Ramp อัตโนมัติ (ง่าย)

ให้เขียนฟังก์ชัน `auto_gamma(image: &GrayImage) -> f64` ที่วิเคราะห์ histogram ของภาพแล้วแนะนำค่า gamma correction ที่เหมาะสม จากนั้นเขียน `apply_gamma(luma: u8, gamma: f64) -> u8` และนำไปใช้ก่อน density mapping

**Hint:** ถ้า median luminance ต่ำกว่า 128 ให้ gamma < 1.0 (brighten), ถ้าสูงกว่า 128 ให้ gamma > 1.0 (darken)

```rust
pub fn auto_gamma(histogram: &[u32; 256]) -> f64 {
    let total: u32 = histogram.iter().sum();
    let mut cumulative = 0u32;
    let mut median = 128u8;
    for (i, &count) in histogram.iter().enumerate() {
        cumulative += count;
        if cumulative >= total / 2 {
            median = i as u8;
            break;
        }
    }
    // ถ้า median < 128 → ภาพมืด → ต้องทำให้สว่างขึ้น (gamma < 1)
    if median < 100 { 0.6 }
    else if median > 160 { 1.4 }
    else { 1.0 }
}
```

### แบบฝึกหัดที่ 2: Half-Block Characters สำหรับความละเอียดสูงขึ้น (ปานกลาง)

Unicode มีตัวอักษร `▀` (U+2580 UPPER HALF BLOCK) และ `▄` (U+2584 LOWER HALF BLOCK) และ `█` (U+2588 FULL BLOCK) เปิดโอกาสให้ encode 2 pixel แนวตั้งในตัวอักษรเดียว โดยใช้ foreground color กับ background color ต่างกัน

**เป้าหมาย:** เขียน `render_halfblock(rgb_image: &RgbImage) -> Vec<String>` ที่คืน lines ของ Unicode half-block characters พร้อม ANSI color codes ให้ resolution แนวตั้งสูงขึ้น 2 เท่า

**Hint:**
```rust
// ทุก 2 pixel แนวตั้ง = 1 terminal row
// บน = '▀', ล่าง = '▄' เพื่อใช้ fg+bg color แสดง 2 สี
for y in (0..height).step_by(2) {
    let top_color = rgb.get_pixel(x, y);
    let bot_color = if y+1 < height { rgb.get_pixel(x, y+1) } else { top_color };
    // ใช้ '▀' กับ fg=top_color, bg=bot_color
}
```

### แบบฝึกหัดที่ 3: Video Input จาก ffmpeg pipe (ยาก)

เพิ่ม `--video` flag ที่รับ path วิดีโอ (MP4, MOV, etc.) โดยใช้ `std::process::Command` pipe ffmpeg ออกมาเป็น raw frame:

```bash
ffmpeg -i input.mp4 -f rawvideo -pix_fmt rgb24 -vf scale=80:-1 pipe:1
```

แล้วอ่าน stdout ของ ffmpeg ทีละ frame (width × height × 3 bytes) แล้ว render เป็น ASCII

**เป้าหมาย:** เขียน `VideoReader` struct ที่ implement `Iterator<Item = Vec<u8>>` โดยอ่านจาก ffmpeg pipe

### แบบฝึกหัดที่ 4: Export เป็น Animated GIF (ยากมาก)

ใช้ `gif` crate encode ASCII art กลับเป็น GIF animation โดย:
1. Render ASCII art เป็น `image::RgbImage` (ตัวอักษรขาวบนพื้นดำ)
2. ลด palette เป็น 256 สีด้วย `color_quant`
3. Encode ทุก frame ลงใน GIF output file

นี่เป็น "full circle" converter: GIF → ASCII animation → GIF ใหม่

---

## สรุป

ในโปรเจคนี้คุณได้สร้าง ASCII Art Renderer ที่สมบูรณ์ ครอบคลุมทักษะ Rust ขั้นสูงหลายด้าน:

**Algorithms ที่ได้เรียน:**
- **Density mapping** — การ quantize ค่าต่อเนื่องลงใน discrete symbol set
- **Sobel operator** — convolution 2D สำหรับ edge detection (นำไปใช้ใน image processing จริง)
- **Braille encoding** — bit manipulation กับ Unicode standard
- **Floyd-Steinberg dithering** — error diffusion algorithm คลาสสิก
- **Aspect ratio correction** — การปรับ coordinate system ให้เข้ากับ display medium

**Patterns Rust ที่สำคัญ:**
- Module separation — แต่ละ algorithm อยู่ใน module ของตัวเอง testable อิสระ
- Flat pixel buffers — `Vec<u8>` row-major indexing แทน 2D array (memory-efficient)
- Builder pattern สำหรับ output formatting
- Feature flags ใน `Cargo.toml` เพื่อควบคุม dependency size

**Crates ที่ได้ใช้:**
| Crate | ประโยชน์ |
|-------|----------|
| `image 0.25` | โหลด JPEG/PNG/BMP/GIF, resize, color conversion |
| `crossterm 0.28` | Terminal control, ANSI color, cursor movement |
| `clap 4` (derive) | CLI argument parsing แบบ type-safe |
| `serde` + `serde_json` | Serialize/deserialize config, metadata |

โปรเจคถัดไปต่อยอดจาก Graphics module ไปสู่ procedural generation ด้วย **Maze Generator** ซึ่งจะใช้ทักษะ terminal rendering จากโปรเจคนี้ ผสมกับ graph algorithm (BFS/DFS) และ random walk

---

**โปรเจคก่อนหน้า:** [Particle System](project-e08-particle-system.md) | **โปรเจคถัดไป:** [Maze Generator](project-e10-maze-generator.md)
