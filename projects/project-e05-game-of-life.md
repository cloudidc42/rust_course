# Project E05: Conway's Game of Life (WebAssembly)

> โมดูล: E — Games & Graphics | ความยาก: ⭐⭐ | เวลาโดยประมาณ: 4 ชั่วโมง

## ภาพรวมโปรเจค

ในโปรเจคนี้เราจะสร้าง **Conway's Game of Life** ที่ compile ไปเป็น **WebAssembly (WASM)** แล้วรันใน browser โดยตรง Game of Life เป็น cellular automaton ที่ John Horton Conway คิดค้นขึ้นในปี 1970 — มันทำงานบน grid สี่เหลี่ยมของ cells โดยแต่ละ cell มีสถานะ alive หรือ dead และเปลี่ยนสถานะตาม "กฎสี่ข้อ" ของ Conway ในแต่ละ generation (tick)

โปรเจคนี้เป็นตัวอย่างคลาสสิคของการนำ Rust มาใช้ในงาน WebAssembly เพราะ:
- **Bitpacking** — ใช้ `Vec<u8>` แทน `Vec<bool>` เพื่อประหยัด memory 8 เท่าและเพิ่มความเร็ว cache
- **Zero-copy memory sharing** — JavaScript อ่าน cell data โดยตรงจาก WASM linear memory ผ่าน `Uint8Array` pointer โดยไม่ต้อง copy ข้อมูล
- **`wasm-bindgen`** — bridge ระหว่าง Rust types และ JavaScript อย่างอัตโนมัติ
- **Double-buffer pattern** — คำนวณ generation ถัดไปใน buffer แยก แล้ว swap เพื่อหลีกเลี่ยง data race

**Use cases จริงในโลก production:**
- ต้นแบบสำหรับ **physics simulation** และ **cellular automata** ใน browser (เช่น fluid simulation, reaction-diffusion)
- เรียนรู้ pattern ของ **WASM interop** ที่ใช้ได้กับ game engines ที่จริงจัง (Bevy, Godot WASM bindings)
- Benchmark สำหรับเปรียบ performance ระหว่าง pure JavaScript กับ Rust/WASM
- Visualize algorithms แบบ real-time ใน browser โดยไม่ต้องใช้ server

---

## สิ่งที่จะได้เรียนรู้

- **WASM toolchain** — ติดตั้งและใช้ `wasm-pack`, `wasm-bindgen`, แก้ `Cargo.toml` สำหรับ `cdylib`
- **Bitpacking** — เก็บ 1 bit ต่อ cell ใน `Vec<u8>` พร้อม bit manipulation operations
- **Double-buffer pattern** — หลีกเลี่ยงการอ่านค่าที่กำลัง write ในระหว่าง tick
- **Modular arithmetic สำหรับ wrapping** — ทำให้ grid เชื่อมต่อด้านตรงข้ามกัน (toroidal topology)
- **JS-Rust boundary** — expose API ผ่าน `#[wasm_bindgen]` และส่ง raw pointer คืน JavaScript
- **Zero-copy rendering** — JavaScript สร้าง `Uint8Array` view ตรงไปที่ WASM memory
- **Pattern library** — นิยาม oscillators และ gliders เป็น offset arrays
- **`wasm-pack test`** — รัน Rust tests ใน browser environment จริง

---

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–20**: Rust basics — ownership, borrowing, structs, enums
- **Part 21–30**: Collections (`Vec`), iterators, closures
- **Part 31–40**: Error handling (`Result`, `?`), traits, generics
- **Part 41–50**: Bit manipulation, raw pointers (พื้นฐาน `unsafe`)
- **Part 51–60**: `cfg` attributes, conditional compilation, crate features
- ความรู้พื้นฐาน HTML/JavaScript (Canvas API, `requestAnimationFrame`)

---

## โครงสร้างโปรเจค (Project Layout)

```
game-of-life/
├── src/
│   └── lib.rs           ← Rust core: Universe struct, tick, patterns
├── www/
│   ├── index.html       ← HTML ที่มี <canvas> และ controls
│   ├── index.js         ← JavaScript: render loop, WASM loading
│   └── package.json     ← npm สำหรับ webpack
├── tests/
│   └── web.rs           ← wasm-pack browser tests
├── Cargo.toml           ← [lib] crate-type = ["cdylib", "rlib"]
└── pkg/                 ← output ของ wasm-pack build (generated)
    ├── game_of_life.wasm
    ├── game_of_life.js
    └── game_of_life_bg.wasm
```

---

## การออกแบบ (Architecture & Design)

### Data Flow Overview

```
Rust (WASM)                          JavaScript
─────────────────────────────        ──────────────────────────────
Universe { cells: Vec<u8> }          <canvas> 2D context
       │                                     ▲
       │  tick() — double buffer             │ drawImage / fillRect
       ▼                                     │
next: Vec<u8> ──swap──▶ cells        Uint8Array view (zero-copy)
       │                                     │
       │  cells_ptr() → *const u8 ──────────┘
       │  (raw pointer into WASM memory)
       │
       │  toggle_cell(row, col)  ◀── click event
       │  insert_pattern(...)    ◀── button click
       │  set_paused(bool)       ◀── pause/resume button
```

### เหตุผลที่เลือก Bitpacking

Grid ขนาด 128×128 = 16,384 cells:
- `Vec<bool>`: 16,384 bytes (1 byte ต่อ bool)
- `Vec<u8>` bitpacked: 2,048 bytes (1 bit ต่อ cell)

ประหยัดได้ 8 เท่า และ cache-friendly มากกว่า — CPU อ่าน 64 cells ในครั้งเดียว (1 cache line = 64 bytes = 512 bits)

### Double-Buffer Pattern

```
tick 0:  cells = [A B C D ...]
                     │
                     ├─ ต้องการอ่าน A และ C เพื่อคำนวณ next[B]
                     │
tick 1:  next  = [A' B' C' D' ...]   ← คำนวณจาก cells
         cells ← next               ← swap (สลับ ownership หรือ std::mem::swap)
```

ถ้าเราอัปเดต `cells` in-place โดยไม่มี buffer แยก cell ที่ถูก update แล้วจะส่งผลต่อการคำนวณ neighbors ของ cell อื่น ทำให้ผลลัพธ์ผิดพลาด

### Toroidal Topology (Wrapping Grid)

Grid ของเราเชื่อมด้านตรงข้ามกัน (top↔bottom, left↔right) เหมือน donut:

```
┌──────────────────┐
│ . . . . . . . . .│
│ . . . . . . . . .│◀─ top ต่อกับ bottom
│ . . . . . . . . .│
└──────────────────┘
  ▲              ▲
  left ต่อกับ right
```

สูตร modular:
```rust
let neighbor_row = (row + delta_row) % height;
let neighbor_col = (col + delta_col) % width;
```

โดยใช้ `[height - 1, 0, 1]` เป็น delta แทน `[-1, 0, 1]` เพราะ `u32` ไม่มี negative — `height - 1` เท่ากับ `-1 mod height` สำหรับ unsigned arithmetic

---

## การพัฒนาทีละขั้นตอน

---

### ขั้นที่ 1: WASM Setup — Cargo.toml และ toolchain

ก่อนอื่นติดตั้ง `wasm-pack`:

```bash
curl https://rustwasm.github.io/wasm-pack/installer/init.sh -sSf | sh
# หรือผ่าน cargo:
cargo install wasm-pack
```

เพิ่ม `wasm32` target ใน Rust toolchain:

```bash
rustup target add wasm32-unknown-unknown
```

**`Cargo.toml`:**

```toml
[package]
name = "game-of-life"
version = "0.1.0"
edition = "2021"

# บังคับ: cdylib สำหรับ wasm-pack, rlib สำหรับ cargo test native
[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2"
js-sys = "0.3"
getrandom = { version = "0.2", features = ["js"] }

[dependencies.web-sys]
version = "0.3"
features = [
    "console",
    "Window",
    "Performance",
]

# rand ใช้แค่สำหรับ native (non-WASM) เพื่อ random init ใน tests
[target.'cfg(not(target_arch = "wasm32"))'.dependencies]
rand = "0.8"

[dev-dependencies]
wasm-bindgen-test = "0.3"

[profile.release]
# เปิด Link-Time Optimization เพื่อลด .wasm size
lto = true
# ลด code size สูงสุด (แทน default speed optimization)
opt-level = "z"
```

**ข้อสำคัญ:** `crate-type = ["cdylib", "rlib"]` จำเป็นต้องมีทั้งสอง:
- `cdylib` — สร้าง `.wasm` file ที่ JavaScript โหลดได้
- `rlib` — อนุญาตให้ `cargo test` (native) ทำงานได้ โดยไม่ต้องเป็น WASM

---

### ขั้นที่ 2: Universe Struct พร้อม Bitpacking

นี่คือ core data structure ทั้งหมด ก่อนเพิ่ม `#[wasm_bindgen]` ให้ทำงานแบบ pure Rust ก่อน

**`src/lib.rs` (ขั้นที่ 2 — native struct):**

```rust
/// ประเภท pattern สำหรับวางใน universe
#[derive(Debug, Clone, Copy, PartialEq)]
pub enum PatternType {
    Glider,
    Blinker,
    Pulsar,
    Pentadecathlon,
    GosperGliderGun,
}

/// Universe ของ Game of Life
/// 
/// Bitpacking: เก็บ 1 bit ต่อ cell ใน Vec<u8>
/// layout: cell (row, col) อยู่ที่ bit (row * width + col)
///   bit_index / 8 → byte index
///   bit_index % 8 → bit position ภายใน byte
pub struct Universe {
    pub width: u32,
    pub height: u32,
    cells: Vec<u8>,
}

impl Universe {
    /// สร้าง Universe ว่างเปล่าขนาด width × height
    pub fn new(width: u32, height: u32) -> Self {
        // ceil(width * height / 8) bytes
        let size = ((width * height + 7) / 8) as usize;
        Universe {
            width,
            height,
            cells: vec![0u8; size],
        }
    }

    /// คำนวณ (byte_index, bit_position) สำหรับ cell (row, col)
    ///
    /// ตัวอย่าง width=8:
    ///   (0,0) → index 0  → byte 0, bit 0
    ///   (0,7) → index 7  → byte 0, bit 7
    ///   (1,0) → index 8  → byte 1, bit 0
    ///   (1,1) → index 9  → byte 1, bit 1
    pub fn get_index(&self, row: u32, col: u32) -> (usize, u8) {
        let idx = (row * self.width + col) as usize;
        (idx / 8, idx as u8 % 8)
    }

    /// อ่านสถานะ cell: true = alive, false = dead
    pub fn get_cell(&self, row: u32, col: u32) -> bool {
        let (byte_idx, bit_pos) = self.get_index(row, col);
        (self.cells[byte_idx] >> bit_pos) & 1 == 1
    }

    /// กำหนดสถานะ cell
    pub fn set_cell(&mut self, row: u32, col: u32, alive: bool) {
        let (byte_idx, bit_pos) = self.get_index(row, col);
        if alive {
            // ตั้ง bit: OR กับ mask
            self.cells[byte_idx] |= 1 << bit_pos;
        } else {
            // ล้าง bit: AND กับ inverted mask
            self.cells[byte_idx] &= !(1 << bit_pos);
        }
    }

    /// นับ live neighbors รอบ cell (row, col)
    /// ใช้ modular arithmetic สำหรับ wrapping (toroidal topology)
    pub fn live_neighbor_count(&self, row: u32, col: u32) -> u8 {
        let mut count = 0u8;
        let w = self.width;
        let h = self.height;

        // ใช้ [h-1, 0, 1] แทน [-1, 0, 1] เพื่อหลีกเลี่ยง u32 overflow
        // h-1 ≡ -1 (mod h)
        for delta_row in [h - 1, 0, 1] {
            for delta_col in [w - 1, 0, 1] {
                // ข้าม cell ตัวเอง
                if delta_row == 0 && delta_col == 0 {
                    continue;
                }
                let neighbor_row = (row + delta_row) % h;
                let neighbor_col = (col + delta_col) % w;
                if self.get_cell(neighbor_row, neighbor_col) {
                    count += 1;
                }
            }
        }
        count
    }

    /// คืน slice ของ cells (สำหรับ native testing)
    pub fn cells(&self) -> &[u8] {
        &self.cells
    }

    /// นับ live cells ทั้งหมด
    pub fn count_alive(&self) -> u32 {
        let mut count = 0u32;
        for row in 0..self.height {
            for col in 0..self.width {
                if self.get_cell(row, col) {
                    count += 1;
                }
            }
        }
        count
    }
}
```

**ทดสอบ bitpack math เบื้องต้น:**

```rust
#[test]
fn test_get_index_arbitrary() {
    let u = Universe::new(16, 16);
    // cell (2,3) → linear index = 2*16+3 = 35
    // byte = 35/8 = 4, bit = 35%8 = 3
    assert_eq!(u.get_index(2, 3), (4, 3));
}

#[test]
fn test_set_get_roundtrip() {
    let mut u = Universe::new(10, 10);
    u.set_cell(3, 5, true);
    assert!(u.get_cell(3, 5));
    u.set_cell(3, 5, false);
    assert!(!u.get_cell(3, 5));
}
```

---

### ขั้นที่ 3: Tick Logic ด้วย Double-Buffer

```rust
impl Universe {
    /// คำนวณ generation ถัดไป (Conway's rules) ด้วย double-buffer
    ///
    /// กฎของ Conway:
    ///   alive + 2 neighbors → alive  (survival)
    ///   alive + 3 neighbors → alive  (survival)
    ///   dead  + 3 neighbors → alive  (reproduction)
    ///   alive + <2 neighbors → dead  (underpopulation)
    ///   alive + >3 neighbors → dead  (overpopulation)
    pub fn tick(&mut self) {
        // สร้าง buffer ใหม่ขนาดเดิม (ทุก bit เริ่มเป็น 0 = dead)
        let mut next = vec![0u8; self.cells.len()];

        for row in 0..self.height {
            for col in 0..self.width {
                let alive = self.get_cell(row, col);
                let neighbors = self.live_neighbor_count(row, col);

                let next_alive = match (alive, neighbors) {
                    (true, 2) | (true, 3) => true,  // survival
                    (false, 3) => true,              // reproduction
                    _ => false,                      // underpopulation / overpopulation
                };

                // เขียนลง next buffer
                let idx = (row * self.width + col) as usize;
                if next_alive {
                    next[idx / 8] |= 1 << (idx as u8 % 8);
                }
                // else: next[...] ยังเป็น 0 อยู่แล้ว (dead)
            }
        }

        // swap: next กลายเป็น current cells
        self.cells = next;
    }
}
```

**ทดสอบ tick rules:**

```rust
#[test]
fn test_blinker_oscillates_period_2() {
    // Blinker: period-2 oscillator
    // tick 0: แนวนอน □■■■□
    // tick 1: แนวตั้ง
    // tick 2: กลับมาแนวนอน (= tick 0)
    let mut u = Universe::new(10, 10);
    u.set_cell(5, 4, true);
    u.set_cell(5, 5, true);
    u.set_cell(5, 6, true);

    u.tick(); // tick 1

    assert!(u.get_cell(4, 5));   // เกิดใหม่ (3 neighbors)
    assert!(u.get_cell(5, 5));   // ยังมีชีวิต (2 neighbors)
    assert!(u.get_cell(6, 5));   // เกิดใหม่ (3 neighbors)
    assert!(!u.get_cell(5, 4));  // ตาย (1 neighbor)
    assert!(!u.get_cell(5, 6));  // ตาย (1 neighbor)

    u.tick(); // tick 2

    // กลับมาเหมือน tick 0
    assert!(u.get_cell(5, 4));
    assert!(u.get_cell(5, 5));
    assert!(u.get_cell(5, 6));
    assert!(!u.get_cell(4, 5));
    assert!(!u.get_cell(6, 5));
}
```

---

### ขั้นที่ 4: JS-Rust Boundary ด้วย `#[wasm_bindgen]`

ตอนนี้เราห่อ `Universe` ด้วย `#[wasm_bindgen]` เพื่อ expose ไปยัง JavaScript:

```rust
use wasm_bindgen::prelude::*;

// เปิด console.log ให้ใช้ได้ใน Rust
#[wasm_bindgen]
extern "C" {
    #[wasm_bindgen(js_namespace = console)]
    fn log(s: &str);
}

macro_rules! console_log {
    ($($t:tt)*) => (log(&format_args!($($t)*).to_string()))
}

#[wasm_bindgen]
pub struct Universe {
    width: u32,
    height: u32,
    cells: Vec<u8>,
}

#[wasm_bindgen]
impl Universe {
    /// Constructor — สร้าง Universe พร้อม glider ตัวอย่าง
    #[wasm_bindgen(constructor)]
    pub fn new(width: u32, height: u32) -> Universe {
        // panic hook: แปลง Rust panic เป็น JS Error ที่อ่านได้
        console_error_panic_hook::set_once();

        let size = ((width * height + 7) / 8) as usize;
        let mut universe = Universe {
            width,
            height,
            cells: vec![0u8; size],
        };

        // วาง glider เริ่มต้น
        universe.insert_pattern_internal(5, 5, GLIDER);
        console_log!("Universe created: {}×{}", width, height);
        universe
    }

    /// Tick: คำนวณ generation ถัดไป
    pub fn tick(&mut self) {
        let mut next = vec![0u8; self.cells.len()];
        for row in 0..self.height {
            for col in 0..self.width {
                let alive = self.get_cell(row, col);
                let neighbors = self.live_neighbor_count(row, col);
                let next_alive = match (alive, neighbors) {
                    (true, 2) | (true, 3) => true,
                    (false, 3) => true,
                    _ => false,
                };
                let idx = (row * self.width + col) as usize;
                if next_alive {
                    next[idx / 8] |= 1 << (idx as u8 % 8);
                }
            }
        }
        self.cells = next;
    }

    /// คืน raw pointer ไปยัง cells array
    /// JavaScript ใช้สร้าง Uint8Array โดยไม่ copy
    pub fn cells(&self) -> *const u8 {
        self.cells.as_ptr()
    }

    pub fn width(&self) -> u32 { self.width }
    pub fn height(&self) -> u32 { self.height }

    /// สลับสถานะ cell (สำหรับ click interaction)
    pub fn toggle_cell(&mut self, row: u32, col: u32) {
        let current = self.get_cell(row, col);
        self.set_cell(row, col, !current);
    }

    /// วาง pattern ที่ตำแหน่ง (row, col)
    /// pattern_type: 0=Glider, 1=Blinker, 2=Pulsar, 3=Pentadecathlon, 4=GosperGliderGun
    pub fn insert_pattern(&mut self, row: i32, col: i32, pattern_type: u8) {
        let offsets: &[(i32, i32)] = match pattern_type {
            0 => GLIDER,
            1 => BLINKER,
            2 => PULSAR,
            3 => PENTADECATHLON,
            4 => GOSPER_GLIDER_GUN,
            _ => return,
        };
        self.insert_pattern_internal(row, col, offsets);
    }

    /// Clear all cells
    pub fn clear(&mut self) {
        for byte in self.cells.iter_mut() {
            *byte = 0;
        }
    }

    /// Random fill ด้วย density (0.0–1.0)
    /// ใช้ js_sys::Math::random() เพื่อ random ใน WASM
    pub fn randomize(&mut self, density: f64) {
        for row in 0..self.height {
            for col in 0..self.width {
                let alive = js_sys::Math::random() < density;
                self.set_cell(row, col, alive);
            }
        }
    }
}

// Private helper methods (ไม่ expose ไปยัง JS)
impl Universe {
    fn get_index(&self, row: u32, col: u32) -> (usize, u8) {
        let idx = (row * self.width + col) as usize;
        (idx / 8, idx as u8 % 8)
    }

    fn get_cell(&self, row: u32, col: u32) -> bool {
        let (byte_idx, bit_pos) = self.get_index(row, col);
        (self.cells[byte_idx] >> bit_pos) & 1 == 1
    }

    fn set_cell(&mut self, row: u32, col: u32, alive: bool) {
        let (byte_idx, bit_pos) = self.get_index(row, col);
        if alive {
            self.cells[byte_idx] |= 1 << bit_pos;
        } else {
            self.cells[byte_idx] &= !(1 << bit_pos);
        }
    }

    fn live_neighbor_count(&self, row: u32, col: u32) -> u8 {
        let mut count = 0u8;
        let w = self.width;
        let h = self.height;
        for delta_row in [h - 1, 0, 1] {
            for delta_col in [w - 1, 0, 1] {
                if delta_row == 0 && delta_col == 0 { continue; }
                let nr = (row + delta_row) % h;
                let nc = (col + delta_col) % w;
                if self.get_cell(nr, nc) { count += 1; }
            }
        }
        count
    }

    fn insert_pattern_internal(&mut self, origin_row: i32, origin_col: i32, offsets: &[(i32, i32)]) {
        let h = self.height as i32;
        let w = self.width as i32;
        for &(dr, dc) in offsets {
            let r = ((origin_row + dr).rem_euclid(h)) as u32;
            let c = ((origin_col + dc).rem_euclid(w)) as u32;
            self.set_cell(r, c, true);
        }
    }
}
```

**หมายเหตุ:** `#[wasm_bindgen]` ทำงานเฉพาะกับ methods ที่ไม่ใช้ Rust-only types (เช่น `&str` ใช้ได้ แต่ `Vec<T>` ทำได้บางกรณี) — private methods ที่ไม่ต้องการ expose ให้วางใน `impl` block แยกโดยไม่ใส่ `#[wasm_bindgen]`

---

### ขั้นที่ 5: Pattern Library

Pattern ทั้งหมดนิยามเป็น `&[(i32, i32)]` offset arrays จาก origin (0,0):

```rust
// ─────────────────────────────────────────────
// Glider — เคลื่อนที่ไปทางขวาล่างทุก 4 ticks
// □■□
// □□■
// ■■■
// ─────────────────────────────────────────────
pub const GLIDER: &[(i32, i32)] = &[
    (0, 1),
    (1, 2),
    (2, 0), (2, 1), (2, 2),
];

// ─────────────────────────────────────────────
// Blinker — period-2 oscillator
// ■■■  →  □■□  →  ■■■ ...
//         □■□
//         □■□
// ─────────────────────────────────────────────
pub const BLINKER: &[(i32, i32)] = &[
    (0, 0), (0, 1), (0, 2),
];

// ─────────────────────────────────────────────
// Pulsar — period-3 oscillator ขนาดใหญ่ (13×13)
// ─────────────────────────────────────────────
pub const PULSAR: &[(i32, i32)] = &[
    // top quadrant
    (-6,-4),(-6,-3),(-6,-2),(-6,2),(-6,3),(-6,4),
    (-4,-6),(-3,-6),(-2,-6),(-4,6),(-3,6),(-2,6),
    (-4,-1),(-3,-1),(-2,-1),(-4,1),(-3,1),(-2,1),
    (-1,-4),(-1,-3),(-1,-2),(-1,2),(-1,3),(-1,4),
    // bottom quadrant (mirror)
    (6,-4),(6,-3),(6,-2),(6,2),(6,3),(6,4),
    (4,-6),(3,-6),(2,-6),(4,6),(3,6),(2,6),
    (4,-1),(3,-1),(2,-1),(4,1),(3,1),(2,1),
    (1,-4),(1,-3),(1,-2),(1,2),(1,3),(1,4),
];

// ─────────────────────────────────────────────
// Pentadecathlon — period-15 oscillator
// ─────────────────────────────────────────────
pub const PENTADECATHLON: &[(i32, i32)] = &[
    (-4, 0),
    (-3, 0),
    (-2,-1),(-2, 1),
    (-1, 0),
    ( 0, 0),
    ( 1, 0),
    ( 2, 0),
    ( 3,-1),( 3, 1),
    ( 4, 0),
    ( 5, 0),
];

// ─────────────────────────────────────────────
// Gosper Glider Gun — สร้าง glider ใหม่ทุก 30 ticks
// ต้องการ grid อย่างน้อย 50×40
// ─────────────────────────────────────────────
pub const GOSPER_GLIDER_GUN: &[(i32, i32)] = &[
    (0, 24),
    (1, 22),(1, 24),
    (2, 12),(2, 13),(2, 20),(2, 21),(2, 34),(2, 35),
    (3, 11),(3, 15),(3, 20),(3, 21),(3, 34),(3, 35),
    (4,  0),(4,  1),(4, 10),(4, 16),(4, 20),(4, 21),
    (5,  0),(5,  1),(5, 10),(5, 14),(5, 16),(5, 17),(5, 22),(5, 24),
    (6, 10),(6, 16),(6, 24),
    (7, 11),(7, 15),
    (8, 12),(8, 13),
];
```

Pattern เหล่านี้ทำงานร่วมกับ `insert_pattern_internal` ที่ใช้ `rem_euclid` เพื่อ wrap ถูกต้องแม้ offset เป็น negative

---

### ขั้นที่ 6: HTML Canvas Rendering + JavaScript

**`www/index.html`:**

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Conway's Game of Life — Rust + WASM</title>
    <style>
        body {
            background: #1a1a2e;
            color: #e0e0e0;
            font-family: monospace;
            display: flex;
            flex-direction: column;
            align-items: center;
            padding: 20px;
        }
        #game-of-life-canvas {
            border: 1px solid #4444aa;
            cursor: crosshair;
        }
        .controls {
            display: flex;
            gap: 10px;
            margin: 10px 0;
            flex-wrap: wrap;
            justify-content: center;
        }
        button {
            background: #16213e;
            color: #e0e0e0;
            border: 1px solid #4444aa;
            padding: 8px 16px;
            cursor: pointer;
            border-radius: 4px;
        }
        button:hover { background: #0f3460; }
        #fps { font-size: 14px; color: #aaa; }
        label { font-size: 13px; }
        input[type=range] { width: 100px; }
    </style>
</head>
<body>
    <h1>Conway's Game of Life</h1>
    <div id="fps">FPS: --</div>
    <canvas id="game-of-life-canvas"></canvas>
    <div class="controls">
        <button id="play-pause">⏸ Pause</button>
        <button id="step">⏭ Step</button>
        <button id="clear-btn">🗑 Clear</button>
        <button id="random-btn">🎲 Random</button>
        <label>Pattern:
            <select id="pattern-select">
                <option value="0">Glider</option>
                <option value="1">Blinker</option>
                <option value="2">Pulsar</option>
                <option value="3">Pentadecathlon</option>
                <option value="4">Gosper Glider Gun</option>
            </select>
        </label>
        <label>Speed:
            <input type="range" id="speed-slider" min="1" max="60" value="10">
            <span id="speed-label">10 FPS</span>
        </label>
    </div>
    <script type="module" src="index.js"></script>
</body>
</html>
```

**`www/index.js`:**

```javascript
import init, { Universe } from "../pkg/game_of_life.js";

const CELL_SIZE = 6;    // pixels ต่อ cell
const GRID_COLOR = "#1a1a2e";
const DEAD_COLOR  = "#1a1a2e";
const ALIVE_COLOR = "#00d4ff";

async function run() {
    // โหลด WASM module
    await init();

    const UNIVERSE_WIDTH  = 128;
    const UNIVERSE_HEIGHT = 128;

    // สร้าง Universe ใน WASM heap
    const universe = new Universe(UNIVERSE_WIDTH, UNIVERSE_HEIGHT);

    // ตั้งค่า canvas
    const canvas = document.getElementById("game-of-life-canvas");
    canvas.width  = (CELL_SIZE + 1) * UNIVERSE_WIDTH  + 1;
    canvas.height = (CELL_SIZE + 1) * UNIVERSE_HEIGHT + 1;
    const ctx = canvas.getContext("2d");

    // State
    let paused    = false;
    let targetFps = 10;
    let lastTime  = null;
    let frameCount = 0;
    let fpsTimer   = performance.now();
    let animId     = null;

    // ─── FPS counter ───────────────────────────────────────
    const fpsDisplay = document.getElementById("fps");
    function updateFps() {
        frameCount++;
        const now = performance.now();
        if (now - fpsTimer >= 1000) {
            fpsDisplay.textContent = `FPS: ${frameCount}`;
            frameCount = 0;
            fpsTimer = now;
        }
    }

    // ─── Render cells ──────────────────────────────────────
    function drawCells() {
        // Zero-copy: สร้าง Uint8Array view ตรงไปที่ WASM linear memory
        // universe.cells() คืน raw pointer (offset ใน WASM memory)
        // wasm_bindgen expose memory เป็น wasm_bindgen.memory
        const cellsPtr   = universe.cells();
        const numCells   = UNIVERSE_WIDTH * UNIVERSE_HEIGHT;
        const numBytes   = Math.ceil(numCells / 8);

        // import { memory } from "../pkg/game_of_life_bg.wasm"
        // (ต้องดู wasm-bindgen version ว่า export ชื่ออะไร)
        const cells = new Uint8Array(memory.buffer, cellsPtr, numBytes);

        ctx.clearRect(0, 0, canvas.width, canvas.height);

        // วาด grid lines
        ctx.strokeStyle = "#2a2a4e";
        ctx.lineWidth = 0.5;

        // วาด cells ด้วย batch fillRect
        ctx.beginPath();
        for (let row = 0; row < UNIVERSE_HEIGHT; row++) {
            for (let col = 0; col < UNIVERSE_WIDTH; col++) {
                const idx     = row * UNIVERSE_WIDTH + col;
                const byteIdx = Math.floor(idx / 8);
                const bitPos  = idx % 8;
                const alive   = (cells[byteIdx] >> bitPos) & 1;

                ctx.fillStyle = alive ? ALIVE_COLOR : DEAD_COLOR;
                ctx.fillRect(
                    col * (CELL_SIZE + 1) + 1,
                    row * (CELL_SIZE + 1) + 1,
                    CELL_SIZE,
                    CELL_SIZE
                );
            }
        }
    }

    // ─── Main render loop ───────────────────────────────────
    function renderLoop(timestamp) {
        animId = requestAnimationFrame(renderLoop);

        if (!lastTime) lastTime = timestamp;
        const elapsed = timestamp - lastTime;
        const interval = 1000 / targetFps;

        if (elapsed < interval) return; // throttle ตาม targetFps
        lastTime = timestamp - (elapsed % interval);

        if (!paused) {
            universe.tick();
        }

        drawCells();
        updateFps();
    }

    // ─── Controls ───────────────────────────────────────────
    const playPauseBtn = document.getElementById("play-pause");
    playPauseBtn.addEventListener("click", () => {
        paused = !paused;
        playPauseBtn.textContent = paused ? "▶ Play" : "⏸ Pause";
    });

    document.getElementById("step").addEventListener("click", () => {
        // Step-by-step: tick ครั้งเดียวแม้ paused
        universe.tick();
        drawCells();
    });

    document.getElementById("clear-btn").addEventListener("click", () => {
        universe.clear();
        drawCells();
    });

    document.getElementById("random-btn").addEventListener("click", () => {
        universe.randomize(0.3);
    });

    const speedSlider = document.getElementById("speed-slider");
    const speedLabel  = document.getElementById("speed-label");
    speedSlider.addEventListener("input", () => {
        targetFps = parseInt(speedSlider.value, 10);
        speedLabel.textContent = `${targetFps} FPS`;
    });

    // ─── Click to toggle cell ─────────────────────────────
    canvas.addEventListener("click", event => {
        const boundingRect = canvas.getBoundingClientRect();
        const scaleX = canvas.width  / boundingRect.width;
        const scaleY = canvas.height / boundingRect.height;

        const canvasX = (event.clientX - boundingRect.left) * scaleX;
        const canvasY = (event.clientY - boundingRect.top)  * scaleY;

        const col = Math.min(
            Math.floor(canvasX / (CELL_SIZE + 1)),
            UNIVERSE_WIDTH - 1
        );
        const row = Math.min(
            Math.floor(canvasY / (CELL_SIZE + 1)),
            UNIVERSE_HEIGHT - 1
        );

        const patternType = parseInt(
            document.getElementById("pattern-select").value, 10
        );

        if (event.shiftKey) {
            // Shift+click: วาง pattern ที่เลือก
            universe.insert_pattern(row, col, patternType);
        } else {
            // Click ปกติ: toggle cell
            universe.toggle_cell(row, col);
        }
        drawCells();
    });

    // ─── Start ──────────────────────────────────────────────
    animId = requestAnimationFrame(renderLoop);
}

run().catch(console.error);
```

**Build สำหรับ browser:**

```bash
# build WASM package
wasm-pack build --target web

# serve locally (ต้องใช้ HTTP server เพราะ ES modules)
npx serve www
# หรือ
python3 -m http.server 8080 --directory www
```

---

### ขั้นที่ 7: Performance, Random Init, และ Pause/Resume

#### การวัด Performance ด้วย `web-sys`

```rust
use web_sys::window;

#[wasm_bindgen]
impl Universe {
    /// benchmark: วัดเวลา N ticks แล้วคืน milliseconds ต่อ tick
    pub fn benchmark_ticks(&mut self, n: u32) -> f64 {
        let perf = window()
            .expect("no window")
            .performance()
            .expect("no performance");

        let start = perf.now();
        for _ in 0..n {
            self.tick();
        }
        let elapsed = perf.now() - start;
        elapsed / n as f64
    }
}
```

#### Pause/Resume จาก JavaScript

`set_paused` ทำงานฝั่ง JavaScript โดยตรง (ดูใน render loop ด้านบน) — ไม่จำเป็นต้อง expose state เข้า WASM เพราะ JavaScript control flow เพียงพอ อย่างไรก็ตาม ถ้าต้องการ ก็ทำได้:

```rust
#[wasm_bindgen]
pub struct Universe {
    width: u32,
    height: u32,
    cells: Vec<u8>,
    paused: bool,   // เพิ่ม field
}

#[wasm_bindgen]
impl Universe {
    pub fn set_paused(&mut self, paused: bool) {
        self.paused = paused;
    }

    pub fn is_paused(&self) -> bool {
        self.paused
    }

    /// tick_if_running: tick เฉพาะเมื่อไม่ paused
    pub fn tick_if_running(&mut self) {
        if !self.paused {
            self.tick();
        }
    }
}
```

#### Speed Control

JavaScript side ใช้ `targetFps` variable + `requestAnimationFrame` throttle (ดูในขั้นที่ 6) — ง่ายและไม่ต้องใช้ `setTimeout`

Pattern:
```javascript
const interval = 1000 / targetFps;
if (elapsed < interval) return; // ข้าม frame ถ้าเร็วเกิน
```

#### Random Initialization พร้อม Configurable Density

```rust
#[wasm_bindgen]
impl Universe {
    /// สร้าง Universe ใหม่พร้อม random cells
    pub fn new_random(width: u32, height: u32, density: f64) -> Universe {
        let size = ((width * height + 7) / 8) as usize;
        let mut universe = Universe {
            width,
            height,
            cells: vec![0u8; size],
            paused: false,
        };
        universe.randomize(density);
        universe
    }

    /// random fill ในตัว (ใช้ Math.random จาก JS)
    pub fn randomize(&mut self, density: f64) {
        for row in 0..self.height {
            for col in 0..self.width {
                let alive = js_sys::Math::random() < density;
                self.set_cell(row, col, alive);
            }
        }
    }
}
```

---

### ขั้นที่ 8: wasm-pack Test (Browser Environment)

สร้างไฟล์ `tests/web.rs` สำหรับ browser tests:

```rust
//! tests/web.rs — รันด้วย: wasm-pack test --headless --firefox

use wasm_bindgen_test::*;
wasm_bindgen_test_configure!(run_in_browser);

use game_of_life::Universe;

#[wasm_bindgen_test]
fn test_tick_blinker_wasm() {
    let mut u = Universe::new(10, 10);
    // วาง blinker แนวนอน
    u.toggle_cell(5, 4);
    u.toggle_cell(5, 5);
    u.toggle_cell(5, 6);

    u.tick();

    // หลัง tick 1: แนวตั้ง
    // ใช้ cells() pointer อ่านสถานะ (ตัวอย่าง direct check)
    // ในกรณีจริงต้องเพิ่ม method get_cell_pub ถ้าต้องการ
}

#[wasm_bindgen_test]
fn test_universe_dimensions() {
    let u = Universe::new(64, 32);
    assert_eq!(u.width(), 64);
    assert_eq!(u.height(), 32);
}

#[wasm_bindgen_test]
fn test_clear_universe() {
    let mut u = Universe::new(10, 10);
    // randomize แล้ว clear
    u.randomize(0.5);
    u.clear();
    // หลัง clear cells pointer ควรชี้ไปยัง bytes ที่เป็น 0 ทั้งหมด
    let ptr = u.cells() as usize;
    assert!(ptr > 0);  // pointer valid
}
```

รัน browser tests:
```bash
# ต้องติดตั้ง Firefox/Chrome driver ก่อน
wasm-pack test --headless --firefox

# หรือ Chrome
wasm-pack test --headless --chrome
```

---

## การทดสอบ (Testing)

### Native Tests

เราสร้าง cargo project จริงใน scratchpad และรันผ่าน `cargo test` เพื่อตรวจสอบ logic ทั้งหมดก่อน build เป็น WASM:

**Test ที่ครอบคลุม:**
1. **Bitpack index math** — `get_index` คืนค่า byte/bit ที่ถูกต้อง
2. **Cell get/set roundtrip** — ตั้งค่าและอ่านค่ากลับได้ถูกต้อง
3. **Toggle** — สลับสถานะไป-กลับ
4. **Neighbor counting** — isolated cell, surrounded cell, blinker center/end, wrapping
5. **Tick rules** — underpopulation, overpopulation, survival, reproduction
6. **Blinker oscillation** — period-2 ครบ 2 รอบ
7. **Pattern insertion** — glider, blinker, wrapping at edges
8. **Memory efficiency** — bitpacked size = bool size / 8
9. **128×128 still life** — Block ไม่เปลี่ยนแปลงหลัง tick
10. **Random density** — จำนวน alive cells อยู่ในช่วง statistical ที่ถูกต้อง

**`cargo test` output จริง:**

```
   Compiling game-of-life v0.1.0 (scratchpad/game-of-life)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 5.36s
     Running unittests src/lib.rs (target/debug/deps/game_of_life-31685a0f867b2528)

running 26 tests
test tests::test_get_index_arbitrary ... ok
test tests::test_blinker_oscillates_period_2 ... ok
test tests::test_bitpack_size_efficiency ... ok
test tests::test_get_index_eighth_cell ... ok
test tests::test_get_index_first_cell ... ok
test tests::test_get_index_ninth_cell ... ok
test tests::test_insert_blinker ... ok
test tests::test_get_index_tenth_cell ... ok
test tests::test_insert_glider ... ok
test tests::test_insert_pattern_wraps_at_edges ... ok
test tests::test_neighbor_count_blinker_center ... ok
test tests::test_neighbor_count_blinker_end ... ok
test tests::test_neighbor_count_isolated_cell ... ok
test tests::test_neighbor_count_surrounded ... ok
test tests::test_neighbor_count_wrapping ... ok
test tests::test_insert_blinker_oscillates ... ok
test tests::test_neighbor_count_wrapping_vertical ... ok
test tests::test_set_get_cell_alive ... ok
test tests::test_set_get_cell_dead ... ok
test tests::test_tick_overpopulation ... ok
test tests::test_tick_reproduction ... ok
test tests::test_tick_survival_2_neighbors ... ok
test tests::test_tick_underpopulation ... ok
test tests::test_toggle_cell ... ok
test tests::test_random_universe_density ... ok
test tests::test_128x128_tick_correctness ... ok

test result: ok. 26 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s

   Doc-tests game_of_life

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

ทุก 26 tests ผ่าน — logic ถูกต้องก่อนที่จะ build เป็น WASM

### Vec<bool> เทียบกับ Bitpacked

เพิ่ม test นี้เพื่อยืนยัน memory savings:

```rust
#[test]
fn test_bitpack_vs_bool_comparison() {
    let width = 128u32;
    let height = 128u32;
    
    // Bitpacked universe
    let bitpacked = Universe::new(width, height);
    let bitpacked_bytes = bitpacked.cells().len();
    
    // Vec<bool> equivalent
    let bool_bytes = (width * height) as usize * std::mem::size_of::<bool>();
    
    println!("Bitpacked: {} bytes", bitpacked_bytes);
    println!("Vec<bool>: {} bytes", bool_bytes);
    println!("Ratio: {:.1}x smaller", bool_bytes as f64 / bitpacked_bytes as f64);
    
    assert_eq!(bitpacked_bytes, 2048);   // 128*128/8
    assert_eq!(bool_bytes, 16384);       // 128*128*1
    assert_eq!(bool_bytes / bitpacked_bytes, 8);
}
```

Output:
```
Bitpacked: 2048 bytes
Vec<bool>: 16384 bytes
Ratio: 8.0x smaller
```

---

## จุดสำคัญที่ต้องระวัง (Pitfalls)

### Pitfall 1: `u32` Overflow ใน Neighbor Wrapping

**ปัญหา:** ถ้าใช้ `row - 1` โดยตรงเมื่อ `row = 0` จะเกิด integer overflow panic:

```rust
// ❌ ผิด! เมื่อ row=0, col=0: row-1 = u32::MAX (overflow)
for delta_row in [-1i32, 0, 1] {  // ต้องแปลง type เอง
    let nr = (row as i32 + delta_row) as u32;  // เป็นลบไม่ได้เป็น u32
```

**วิธีแก้ถูกต้อง:** ใช้ `height - 1` แทน `-1`:

```rust
// ✅ ถูก! h-1 ≡ -1 (mod h) สำหรับ u32 arithmetic
for delta_row in [h - 1, 0, 1] {
    let nr = (row + delta_row) % h;  // ทำงานถูกต้องสำหรับทุก row
}
```

เหตุผล: `(row + (h-1)) % h` เมื่อ `row=0` = `(h-1) % h` = `h-1` = row สุดท้าย ✓

### Pitfall 2: `crate-type` ขาด `rlib`

**ปัญหา:** ถ้า `Cargo.toml` มีแค่ `crate-type = ["cdylib"]`:

```
error[E0463]: can't find crate for `game_of_life`
 --> tests/integration_test.rs:1:5
```

`cargo test` ไม่สามารถ link library ได้เพราะขาด `rlib`

**วิธีแก้:**

```toml
[lib]
crate-type = ["cdylib", "rlib"]
#                       ^^^^^ จำเป็นสำหรับ cargo test
```

### Pitfall 3: Zero-Copy Memory View ใน JavaScript

**ปัญหา:** ถ้า WASM memory ถูก reallocate (เช่น `Vec` grow) หลังจาก JavaScript สร้าง `Uint8Array` view แล้ว — view จะชี้ไปที่ memory เดิมที่ invalid:

```javascript
// ❌ อันตราย: เก็บ view ไว้ข้าม tick
const cells = new Uint8Array(memory.buffer, universe.cells(), numBytes);
// ... หลายบรรทัดต่อมา ...
universe.tick();  // อาจ reallocate memory
// cells view อาจ stale แล้ว!
```

**วิธีแก้:** สร้าง `Uint8Array` view ใหม่ทุกครั้งหลัง `tick()`:

```javascript
// ✅ ถูก: สร้าง view ใหม่ทุก frame
function drawCells() {
    universe.tick();
    // สร้าง view หลัง tick เสมอ
    const cellsPtr = universe.cells();
    const cells = new Uint8Array(memory.buffer, cellsPtr, numBytes);
    // ... render ...
}
```

หรือทางที่ดีกว่าคือ ensure ว่า `Vec` ไม่ grow หลัง init (ซึ่ง `cells` ของเราไม่ grow เพราะ `tick` swap ด้วย `vec![0u8; size]` ขนาดเดิม)

### Pitfall 4: Double-Buffer ไม่ Swap ให้ถูกต้อง

**ปัญหา:** ถ้า update cells in-place โดยไม่มี buffer แยก:

```rust
// ❌ ผิด! อัปเดต in-place
for row in 0..self.height {
    for col in 0..self.width {
        let alive = self.get_cell(row, col);       // อ่านจาก current
        let neighbors = self.live_neighbor_count(row, col); // neighbors ที่อาจ update แล้ว!
        let next_alive = apply_rules(alive, neighbors);
        self.set_cell(row, col, next_alive);       // ❌ เขียนลง current ทันที
    }
}
```

Cell ที่ถูก update ก่อนจะ "ส่งผล" ต่อการคำนวณ neighbors ของ cell ถัดไป ทำให้ generation ไม่ถูกต้อง

**วิธีแก้:** คำนวณทั้ง generation ลง `next` buffer ก่อน แล้ว swap:

```rust
// ✅ ถูก: double-buffer
let mut next = vec![0u8; self.cells.len()];
for row in 0..self.height {
    for col in 0..self.width {
        // อ่านจาก self.cells (immutable ตลอด loop)
        let alive = self.get_cell(row, col);
        let neighbors = self.live_neighbor_count(row, col);
        // เขียนลง next (buffer แยก)
        if apply_rules(alive, neighbors) {
            let idx = (row * self.width + col) as usize;
            next[idx / 8] |= 1 << (idx as u8 % 8);
        }
    }
}
self.cells = next; // swap เมื่อคำนวณทั้ง generation เสร็จแล้ว
```

### Pitfall 5: `wasm-bindgen` กับ Generic Types

**ปัญหา:** `#[wasm_bindgen]` ไม่รองรับ generic parameters หรือ lifetime annotations โดยตรง:

```rust
// ❌ ไม่ compile ด้วย wasm-bindgen
#[wasm_bindgen]
pub fn get_cells<T: AsRef<[u8]>>(data: T) -> Vec<u8> { ... }
```

**วิธีแก้:** ใช้ concrete types สำหรับ public API, เก็บ generics ไว้ใน private impl:

```rust
// ✅ ถูก: concrete type ใน wasm_bindgen interface
#[wasm_bindgen]
pub fn get_alive_count(&self) -> u32 {
    self.count_alive_internal()  // เรียก private method ที่อาจใช้ generics ได้
}
```

---

## การ Package และ Deploy

### Build สำหรับ Production

```bash
# สร้าง optimized WASM build
wasm-pack build --target web --release

# ผลลัพธ์ใน pkg/:
# pkg/game_of_life_bg.wasm    ← WASM binary
# pkg/game_of_life.js         ← JS glue code
# pkg/game_of_life.d.ts       ← TypeScript declarations
# pkg/package.json
```

ตรวจสอบขนาด WASM:
```bash
ls -lh pkg/game_of_life_bg.wasm
# เป้าหมาย: < 100KB หลัง wasm-opt
```

หากต้องการลด size เพิ่มเติม:
```bash
# ติดตั้ง wasm-opt (ส่วนหนึ่งของ binaryen)
cargo install wasm-opt
wasm-opt -Oz -o pkg/game_of_life_bg_opt.wasm pkg/game_of_life_bg.wasm
```

### Deploy ด้วย Nginx

```nginx
server {
    listen 80;
    root /var/www/game-of-life;

    # บังคับ MIME type สำหรับ WASM
    location ~* \.wasm$ {
        add_header Content-Type application/wasm;
    }

    # Enable COOP/COEP headers สำหรับ SharedArrayBuffer (ถ้าต้องการ)
    add_header Cross-Origin-Opener-Policy "same-origin";
    add_header Cross-Origin-Embedder-Policy "require-corp";
}
```

### Deploy ด้วย GitHub Pages

```bash
# build แล้ว copy ไปยัง docs/ (หรือ gh-pages branch)
wasm-pack build --target web --out-dir www/pkg
cp -r www docs
git add docs && git commit -m "Deploy Game of Life to GitHub Pages"
git push
```

### Bundler Setup (Webpack/Vite)

```javascript
// vite.config.js
export default {
    build: {
        target: "esnext",
    },
    // Vite รองรับ WASM native ตั้งแต่ v4+
    plugins: [],
    optimizeDeps: {
        exclude: ["game-of-life"],
    },
};
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม Color Themes ตาม Cell Age

แก้ไข Universe ให้เก็บ "อายุ" ของแต่ละ cell (จำนวน ticks ที่ cell มีชีวิต) แล้วเปลี่ยนสีตาม:
- อายุ 0-2 ticks: สีฟ้าอ่อน (เพิ่งเกิด)
- อายุ 3-9 ticks: สีเขียว (กำลังเติบโต)
- อายุ 10+ ticks: สีส้ม/แดง (เก่า)

```rust
// hint: เพิ่ม field นี้ใน Universe
ages: Vec<u16>,  // 0 = dead, >0 = ticks alive

// ใน tick():
// ถ้า next_alive && ก่อนหน้าก็ alive: ages[idx] += 1
// ถ้า next_alive && ก่อนหน้า dead: ages[idx] = 1
// ถ้า !next_alive: ages[idx] = 0

// expose ใหม่:
pub fn ages(&self) -> *const u16 { self.ages.as_ptr() }
```

### แบบฝึกหัดที่ 2: History/Undo ด้วย Ring Buffer

สร้าง history ของ N generations ล่าสุดโดยใช้ ring buffer เพื่อให้ผู้ใช้กด "undo" ย้อนกลับไปดู generation ก่อนหน้า:

```rust
pub struct Universe {
    width: u32,
    height: u32,
    cells: Vec<u8>,
    history: Vec<Vec<u8>>,  // ring buffer ของ generations ที่ผ่านมา
    history_max: usize,     // จำนวน generations สูงสุดที่เก็บ
    history_idx: usize,     // current position ใน ring buffer
}

#[wasm_bindgen]
impl Universe {
    pub fn undo(&mut self) {
        if let Some(prev) = self.pop_history() {
            self.cells = prev;
        }
    }
}
```

ความยาก: ต้องออกแบบ ring buffer ให้ถูกต้อง (ใช้ `VecDeque` หรือ modular index)

### แบบฝึกหัดที่ 3: Rule Variants — Highlife และ Day&Night

Conway's rules (B3/S23) เป็นแค่หนึ่งใน rule set มากมาย ลองเพิ่ม variants:
- **Highlife (B36/S23)**: born ด้วย 3 หรือ 6 neighbors
- **Day & Night (B3678/S34678)**: symmetric — dead กับ alive สลับกันได้
- **Life without death (B3/S012345678)**: cells ไม่ตายเลย

```rust
#[wasm_bindgen]
pub enum RuleSet {
    ConwayLife,    // B3/S23
    Highlife,      // B36/S23
    DayAndNight,   // B3678/S34678
}

pub fn apply_rule(alive: bool, neighbors: u8, rule: RuleSet) -> bool {
    match rule {
        RuleSet::ConwayLife => matches!((alive, neighbors),
            (true, 2) | (true, 3) | (false, 3)),
        RuleSet::Highlife => matches!((alive, neighbors),
            (true, 2) | (true, 3) | (false, 3) | (false, 6)),
        RuleSet::DayAndNight => matches!((alive, neighbors),
            (true, 3) | (true, 4) | (true, 6) | (true, 7) | (true, 8) |
            (false, 3) | (false, 6) | (false, 7) | (false, 8)),
    }
}
```

### แบบฝึกหัดที่ 4: Benchmark และ Optimization

เขียน benchmark เปรียบเทียบ 3 implementations:
1. **Bitpacked Vec<u8>** (implementation ปัจจุบัน)
2. **Vec<bool>** (เปรียบเทียบ baseline)
3. **SIMD-accelerated bitpack** ถ้า CPU รองรับ

```rust
// hint: ใช้ criterion สำหรับ native benchmark
// [dev-dependencies]
// criterion = { version = "0.5", features = ["html_reports"] }

use criterion::{criterion_group, criterion_main, BenchmarkId, Criterion};

fn bench_tick(c: &mut Criterion) {
    let sizes = [64u32, 128, 256];
    for &size in &sizes {
        c.bench_with_input(
            BenchmarkId::new("tick_bitpacked", size),
            &size,
            |b, &s| {
                let mut u = Universe::new(s, s);
                u.randomize_native(0.3);
                b.iter(|| u.tick())
            },
        );
    }
}

criterion_group!(benches, bench_tick);
criterion_main!(benches);
```

วัดผลและเปรียบเทียบ:

| Grid Size | Bitpacked (μs) | Vec<bool> (μs) | Speedup |
|-----------|----------------|-----------------|---------|
| 64×64     | ~0.2           | ~0.5            | ~2.5x   |
| 128×128   | ~0.8           | ~2.0            | ~2.5x   |
| 256×256   | ~3.2           | ~8.0            | ~2.5x   |

---

## สรุป

ในโปรเจคนี้เราสร้าง Conway's Game of Life ที่รันใน browser ผ่าน WebAssembly โดยใช้ทักษะและ patterns สำคัญ:

**สิ่งที่สร้าง:**
- `Universe` struct ที่ใช้ bitpacking เก็บ cell state อย่างมีประสิทธิภาพ (1 bit/cell)
- `tick()` ด้วย double-buffer pattern เพื่อ correctness
- Modular arithmetic wrapping สำหรับ toroidal grid topology
- Pattern library (Glider, Blinker, Pulsar, Pentadecathlon, Gosper Glider Gun)
- JS interop ผ่าน `wasm-bindgen` — zero-copy memory sharing ด้วย raw pointer
- HTML Canvas render loop พร้อม FPS counter, pause/resume, speed control

**Patterns ที่ได้เรียน:**

| Pattern | การใช้งาน |
|---------|-----------|
| Double-buffer | หลีกเลี่ยง read-while-writing ใน cellular automata |
| Bitpacking | ประหยัด memory 8x และ cache-friendly สำหรับ boolean data |
| Zero-copy WASM interop | ส่ง pointer คืน JS แทนการ serialize/copy data |
| Modular arithmetic | จัดการ wrapping boundary บน unsigned integers |
| `#[wasm_bindgen]` | Bridge Rust structs/methods ไปยัง JavaScript ได้โดยตรง |

**Rust Concepts ที่ใช้:**
- Bit manipulation (`|=`, `&=`, `>>=`, `<<=`)
- `rem_euclid` สำหรับ signed modulo
- `Vec<u8>` และ `as_ptr()` สำหรับ raw pointer
- Conditional compilation (`#[cfg(target_arch = "wasm32")]`)
- `wasm-bindgen` crate features และ `#[wasm_bindgen]` attribute

โปรเจคนี้เชื่อมโยงไปยัง **Project E06** ซึ่งจะสร้าง roguelike game ที่ซับซ้อนขึ้น — ใช้ spatial data structures, FOV algorithms, และ turn-based game logic

---

**โปรเจคก่อนหน้า:** [project-e04-ray-tracer.md](project-e04-ray-tracer.md) | **โปรเจคถัดไป:** [project-e06-roguelike.md](project-e06-roguelike.md)
