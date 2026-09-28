# Project J09: WebAssembly กับ wasm-bindgen และ npm Interop

> โมดูล: J — Full-Stack / WASM | ความยาก: ⭐⭐⭐⭐ | เวลาโดยประมาณ: 10 ชั่วโมง

## ภาพรวมโปรเจค

ในโปรเจคนี้เราจะสร้าง **library ที่ compile ด้วย Rust แล้ว publish เป็น npm package** สำหรับใช้ใน JavaScript และ TypeScript โดยตรง — ไม่ใช่แค่ demo ง่าย ๆ แต่เป็น workflow แบบ production จริง ตั้งแต่ติดตั้ง toolchain จนถึง bundle ด้วย Vite และ publish ไปที่ npm registry

**WebAssembly (WASM)** คือ binary instruction format ที่รันได้ใน browser และ Node.js โดยมี performance ใกล้เคียง native code Rust เป็นภาษาที่เหมาะที่สุดสำหรับการ compile ไป WASM เพราะไม่มี garbage collector และ memory footprint ต่ำมาก

**wasm-bindgen** คือ glue code generator ที่ช่วยให้ Rust functions สามารถรับ/ส่งค่า JavaScript types ได้อัตโนมัติ เช่น `String`, `Array`, objects และ callbacks โดยไม่ต้องเขียน FFI (Foreign Function Interface) ด้วยมือ

**Use cases จริงในโลก production:**
- Figma ใช้ Rust/WASM สำหรับ rendering engine หลัก ทำให้ canvas operations เร็วกว่า JavaScript หลายเท่า
- Google Earth Web ใช้ C++/WASM สำหรับ 3D geometry และ terrain processing
- AutoCAD Web ใช้ WASM สำหรับ CAD kernel ที่เดิม compile มาจาก C++
- Cloudflare Workers รัน WASM modules สำหรับ edge computing logic
- NPM packages อย่าง `@automerge/automerge` (CRDT library) ใช้ Rust/WASM ทั้งหมด

**Learning value:**
- เข้าใจ boundary ระหว่าง JavaScript heap และ WASM linear memory
- รู้วิธี design API ที่ minimize overhead จาก JS↔Wasm crossing
- สามารถ publish Rust logic เป็น npm package ให้ทีม frontend ใช้ได้ทันที
- เข้าใจวิธี generate TypeScript type definitions จาก Rust types

---

## สิ่งที่จะได้เรียนรู้

- **wasm-bindgen macro system** — ใช้ `#[wasm_bindgen]` บน `pub fn`, `pub struct`, `impl` blocks เพื่อ expose Rust API ให้ JavaScript
- **Type mapping** — แปลง Rust types (`String`, `&str`, `f64`, `bool`, `Vec`, `JsValue`) ไป/กลับ JavaScript types
- **Calling JS from Rust** — ใช้ `web_sys`, `js_sys` และ `#[wasm_bindgen(module = "...")]` extern blocks
- **wasm-pack build targets** — เข้าใจความแตกต่างของ `--target web`, `--target bundler`, `--target nodejs`
- **TypeScript types** — wasm-bindgen generate `.d.ts` อัตโนมัติ และเพิ่ม custom annotations
- **Error handling** — ใช้ `Result<T, JsValue>`, `JsError`, และ panic hook สำหรับ debug
- **Complex data** — ใช้ `serde-wasm-bindgen` และ JSON bridge สำหรับ structs/enums
- **npm packaging** — สร้าง `package.json`, bundle ด้วย Vite, import ใน TypeScript
- **Performance optimization** — ลด boundary crossings, batch operations, ใช้ shared memory

---

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–20**: Rust basics — ownership, borrowing, lifetimes, structs, enums
- **Part 21–35**: Collections, iterators, closures, error handling (`Result`, `?`)
- **Part 36–50**: Traits, generics, trait objects, `impl Trait`
- **Part 51–60**: `cfg` attributes, conditional compilation, crate features, `Cargo.toml` workspace
- **Part 61–75**: `async/await` พื้นฐาน, Promises (สำหรับ async WASM API)
- **Part 76–90**: `unsafe` Rust พื้นฐาน, raw pointers, FFI concepts
- ความรู้ JavaScript/TypeScript พื้นฐาน: modules, `async/await`, npm
- ความรู้ HTML/CSS พื้นฐาน: DOM manipulation

---

## โครงสร้างโปรเจค (Project Layout)

```
wasm-geometry/
├── src/
│   └── lib.rs               ← Rust core: API ทั้งหมดที่ expose ให้ JS
├── tests/
│   └── native_tests.rs      ← Rust native tests (cargo test)
├── pkg/                     ← output ของ wasm-pack build (generated, git-ignored)
│   ├── wasm_geometry_bg.wasm
│   ├── wasm_geometry.js     ← JS glue code (generated)
│   ├── wasm_geometry.d.ts   ← TypeScript types (generated)
│   └── package.json
├── demo-app/                ← Vite + TypeScript demo
│   ├── index.html
│   ├── main.ts
│   ├── package.json
│   └── vite.config.ts
├── Cargo.toml
└── README.md
```

---

## การออกแบบ (Architecture & Design)

### Data Flow ระหว่าง JavaScript และ WASM

```
TypeScript / JavaScript              WASM Linear Memory
────────────────────────             ──────────────────────────────
const lib = await init()             Rust heap (allocated by wasm-bindgen)
                                           │
lib.compute(input)                         │ #[wasm_bindgen] fn
       │                                   │
       │  wasm-bindgen glue ──────────────►│ pub fn compute(...)
       │  (serialize args)                 │   → Result<T, JsValue>
       │                                   │
       │◄─────────────────────────────────│ return value
       │  (deserialize result)             │
       ▼                                   │
TypeScript value                     GC'd by wasm-bindgen
```

### Boundary Crossing Costs

ทุกครั้งที่ JavaScript เรียก WASM function มี overhead ดังนี้:
1. **Argument marshaling** — แปลง JS values ไป Wasm linear memory
2. **JIT deoptimization** — JavaScript JIT ไม่สามารถ inline WASM calls ได้
3. **Context switch** — CPU เปลี่ยน execution context จาก JS engine ไป WASM runtime

ดังนั้น design principle สำคัญคือ **ลดจำนวน crossings ให้น้อยที่สุด** และ **ส่งข้อมูลเป็น batch** แทนที่จะเรียก WASM function หลายครั้ง

### ทำไมต้องใช้ `cdylib` และ `rlib` ร่วมกัน

```toml
[lib]
crate-type = ["cdylib", "rlib"]
```

- **`cdylib`** — Dynamic library สำหรับ compile เป็น `.wasm` file
- **`rlib`** — Rust library สำหรับ `cargo test` บน native target (ไม่ต้องใช้ WASM runtime ในการ test)

ถ้าไม่มี `rlib` จะไม่สามารถรัน `cargo test` ได้บน host machine

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: ติดตั้ง Toolchain และสร้างโปรเจค

#### ติดตั้ง wasm-pack

`wasm-pack` คือ all-in-one tool สำหรับ build, test, และ publish Rust→WASM packages:

```bash
# ติดตั้ง wasm-pack
curl https://rustwasm.github.io/wasm-pack/installer/init.sh -sSf | sh

# หรือผ่าน cargo
cargo install wasm-pack

# ตรวจสอบ version
wasm-pack --version
# wasm-pack 0.13.1
```

#### ติดตั้ง wasm32-unknown-unknown target

```bash
# เพิ่ม WASM compile target
rustup target add wasm32-unknown-unknown

# ตรวจสอบ
rustup target list --installed | grep wasm
# wasm32-unknown-unknown
```

#### สร้างโปรเจค

```bash
# สร้างด้วย wasm-pack template
wasm-pack new wasm-geometry
cd wasm-geometry

# หรือสร้าง library ด้วย cargo ปกติ
cargo new --lib wasm-geometry
cd wasm-geometry
```

#### Cargo.toml เริ่มต้น

```toml
[package]
name = "wasm-geometry"
version = "0.1.0"
edition = "2021"
description = "Geometry computation library for WebAssembly"
license = "MIT"

[lib]
# cdylib: สำหรับ compile เป็น .wasm
# rlib: สำหรับ cargo test บน native target
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2"

[dev-dependencies]
wasm-bindgen-test = "0.3"
```

#### ไฟล์ `.gitignore` ที่ควรมี

```gitignore
/target
/pkg
/node_modules
/demo-app/node_modules
/demo-app/dist
```

---

### ขั้นที่ 2: `#[wasm_bindgen]` พื้นฐาน — Functions และ Type Mapping

#### src/lib.rs (เวอร์ชันแรก)

`#[wasm_bindgen]` attribute ทำงานอย่างไร:
- ใส่บน `pub fn` → generate JavaScript wrapper function
- ใส่บน `pub struct` → generate JavaScript class
- ใส่บน `impl` block → generate methods บน JavaScript class

```rust
use wasm_bindgen::prelude::*;

// ─── Simple value types ───────────────────────────────────────────────────────

/// บวกเลข 2 ตัว — f64 แปลงเป็น JavaScript number อัตโนมัติ
#[wasm_bindgen]
pub fn add(a: f64, b: f64) -> f64 {
    a + b
}

/// ตรวจสอบว่าจำนวนเป็นจำนวนเฉพาะหรือไม่ — bool แปลงเป็น JS boolean
#[wasm_bindgen]
pub fn is_prime(n: u32) -> bool {
    if n < 2 { return false; }
    if n == 2 { return true; }
    if n % 2 == 0 { return false; }
    let mut i = 3u32;
    while i * i <= n {
        if n % i == 0 { return false; }
        i += 2;
    }
    true
}

// ─── String types ─────────────────────────────────────────────────────────────

/// รับ &str (JavaScript string) และคืน String
/// wasm-bindgen จัดการ UTF-8 encoding/decoding ให้อัตโนมัติ
#[wasm_bindgen]
pub fn greet(name: &str) -> String {
    format!("สวัสดี, {}! จาก Rust WASM", name)
}

/// ROT13 cipher — ทดสอบ string round-trip
#[wasm_bindgen]
pub fn rot13(input: &str) -> String {
    input
        .chars()
        .map(|c| match c {
            'a'..='z' => (((c as u8 - b'a' + 13) % 26) + b'a') as char,
            'A'..='Z' => (((c as u8 - b'A' + 13) % 26) + b'A') as char,
            _ => c,
        })
        .collect()
}

// ─── Vec<f64> ↔ js_sys::Float64Array ─────────────────────────────────────────

/// คำนวณ factorial — คืน u64 (ระวัง: JS number มี precision จำกัด)
#[wasm_bindgen]
pub fn factorial(n: u32) -> f64 {
    // ใช้ f64 เพราะ JavaScript ไม่มี u64 native type
    // สำหรับค่าใหญ่ต้องใช้ BigInt แทน
    let mut result = 1.0f64;
    for i in 2..=(n as u64) {
        result *= i as f64;
    }
    result
}

/// Fibonacci sequence คืน Vec<f64> (JS Array)
#[wasm_bindgen]
pub fn fibonacci_vec(count: usize) -> Vec<f64> {
    if count == 0 { return vec![]; }
    let mut seq = Vec::with_capacity(count);
    let (mut a, mut b) = (0.0f64, 1.0f64);
    for _ in 0..count {
        seq.push(a);
        let next = a + b;
        a = b;
        b = next;
    }
    seq
}
```

#### ทดสอบว่า Type Mapping ทำงานถูกต้อง

| Rust Type | JavaScript Type | หมายเหตุ |
|-----------|----------------|----------|
| `f64` | `number` | ตรงกันพอดี |
| `i32`, `u32` | `number` | ต้องระวัง overflow |
| `bool` | `boolean` | ตรงกันพอดี |
| `&str` | `string` | copy เข้า WASM memory |
| `String` | `string` | ownership ย้ายมาจาก WASM |
| `Vec<f64>` | `Float64Array` | หรือ `Array` ตาม config |
| `Option<T>` | `T \| undefined` | None → undefined |
| `Result<T, E>` | `T` หรือ throw | Err → JS exception |
| `JsValue` | `any` | raw JS value |

---

### ขั้นที่ 3: Struct และ Impl Blocks — JavaScript Classes

#### เพิ่ม Struct Point และ Polygon

```rust
use wasm_bindgen::prelude::*;

/// #[wasm_bindgen] บน struct สร้าง JavaScript class
/// JavaScript จะมี `new Point(x, y)` ใช้ได้
#[wasm_bindgen]
pub struct Point {
    x: f64,
    y: f64,
}

#[wasm_bindgen]
impl Point {
    /// constructor — ใช้ชื่อ `new` ได้เหมือน JS convention
    #[wasm_bindgen(constructor)]
    pub fn new(x: f64, y: f64) -> Point {
        Point { x, y }
    }

    /// getter — ต้องประกาศ #[wasm_bindgen(getter)] เพื่อให้ JS เรียก point.x ได้
    #[wasm_bindgen(getter)]
    pub fn x(&self) -> f64 {
        self.x
    }

    #[wasm_bindgen(getter)]
    pub fn y(&self) -> f64 {
        self.y
    }

    /// ระยะห่างระหว่างสองจุด
    pub fn distance_to(&self, other: &Point) -> f64 {
        let dx = self.x - other.x;
        let dy = self.y - other.y;
        (dx * dx + dy * dy).sqrt()
    }

    /// Translate (เลื่อน) จุด — คืน Point ใหม่ (immutable pattern)
    pub fn translate(&self, dx: f64, dy: f64) -> Point {
        Point {
            x: self.x + dx,
            y: self.y + dy,
        }
    }

    /// แปลงเป็น string สำหรับ debug
    pub fn to_string(&self) -> String {
        format!("Point({}, {})", self.x, self.y)
    }
}

/// Polygon — สาธิตการรับ Vec<Point> ไม่ได้โดยตรง ต้องผ่าน JsValue หรือ JSON
#[wasm_bindgen]
pub struct Polygon {
    vertices: Vec<(f64, f64)>,  // เก็บเป็น tuple ภายใน (ไม่ expose ตรง ๆ)
}

#[wasm_bindgen]
impl Polygon {
    #[wasm_bindgen(constructor)]
    pub fn new() -> Polygon {
        Polygon { vertices: vec![] }
    }

    /// เพิ่มจุด vertex ทีละจุด
    pub fn add_vertex(&mut self, x: f64, y: f64) {
        self.vertices.push((x, y));
    }

    /// จำนวน vertices
    pub fn vertex_count(&self) -> usize {
        self.vertices.len()
    }

    /// คำนวณ perimeter
    pub fn perimeter(&self) -> f64 {
        let n = self.vertices.len();
        if n < 2 { return 0.0; }
        let mut total = 0.0;
        for i in 0..n {
            let (x1, y1) = self.vertices[i];
            let (x2, y2) = self.vertices[(i + 1) % n];
            let dx = x2 - x1;
            let dy = y2 - y1;
            total += (dx * dx + dy * dy).sqrt();
        }
        total
    }

    /// คำนวณ area ด้วย Shoelace formula
    pub fn area(&self) -> f64 {
        let n = self.vertices.len();
        if n < 3 { return 0.0; }
        let mut sum = 0.0;
        for i in 0..n {
            let (x1, y1) = self.vertices[i];
            let (x2, y2) = self.vertices[(i + 1) % n];
            sum += x1 * y2 - x2 * y1;
        }
        (sum / 2.0).abs()
    }

    /// ตรวจสอบว่าจุดอยู่ภายใน polygon หรือไม่ (Ray casting algorithm)
    pub fn contains_point(&self, px: f64, py: f64) -> bool {
        let n = self.vertices.len();
        if n < 3 { return false; }
        let mut inside = false;
        let mut j = n - 1;
        for i in 0..n {
            let (xi, yi) = self.vertices[i];
            let (xj, yj) = self.vertices[j];
            if ((yi > py) != (yj > py))
                && (px < (xj - xi) * (py - yi) / (yj - yi) + xi)
            {
                inside = !inside;
            }
            j = i;
        }
        inside
    }
}
```

#### ตัวอย่างการใช้งานใน TypeScript

```typescript
import init, { Point, Polygon } from './pkg/wasm_geometry';

async function main() {
    // ต้อง await init() ก่อนเสมอเพื่อโหลด .wasm file
    await init();

    // สร้าง Point objects
    const p1 = new Point(0, 0);
    const p2 = new Point(3, 4);

    console.log(`distance: ${p1.distance_to(p2)}`);  // 5
    console.log(`translated: ${p1.translate(1, 1).to_string()}`);  // Point(1, 1)

    // สร้าง Polygon (square)
    const poly = new Polygon();
    poly.add_vertex(0, 0);
    poly.add_vertex(4, 0);
    poly.add_vertex(4, 4);
    poly.add_vertex(0, 4);

    console.log(`area: ${poly.area()}`);       // 16
    console.log(`perimeter: ${poly.perimeter()}`);  // 16
    console.log(`contains (2,2): ${poly.contains_point(2, 2)}`);  // true
    console.log(`contains (5,5): ${poly.contains_point(5, 5)}`);  // false
}

main();
```

> **สำคัญ**: WASM objects ใน JavaScript ถือ ownership ของ Rust heap memory
> เมื่อ JavaScript GC collect object เหล่านี้ Rust memory จะถูก free ด้วย
> แต่ถ้าต้องการ free memory ก่อน GC ให้เรียก `.free()` method ที่ wasm-bindgen generate ให้

---

### ขั้นที่ 4: Calling JS from Rust — web_sys, js_sys, extern Blocks

#### ใช้ web_sys สำหรับ DOM API

`web_sys` crate มี bindings สำหรับ Web APIs ทั้งหมด — ต้องเปิด features ที่ต้องการ:

```toml
[dependencies]
wasm-bindgen = "0.2"
web-sys = { version = "0.3", features = [
    "Window",
    "Document",
    "HtmlElement",
    "console",
    "Performance",
]}
js-sys = "0.3"
```

```rust
use wasm_bindgen::prelude::*;
use web_sys::{window, console};

/// Log ข้อความไปที่ browser console
#[wasm_bindgen]
pub fn log_to_console(msg: &str) {
    console::log_1(&JsValue::from_str(msg));
}

/// วัดเวลาการรัน (ใช้ performance.now())
#[wasm_bindgen]
pub fn benchmark_computation(n: u32) -> f64 {
    let window = window().expect("no global window");
    let performance = window
        .performance()
        .expect("performance not available");

    let start = performance.now();

    // ทำ computation ที่ต้องการ benchmark
    let mut sum = 0.0f64;
    for i in 0..n {
        sum += (i as f64).sqrt();
    }

    let elapsed = performance.now() - start;

    // log ผลไปที่ console
    console::log_2(
        &JsValue::from_str(&format!("Sum: {:.2}, Time:", sum)),
        &JsValue::from_f64(elapsed),
    );

    elapsed
}
```

#### ใช้ js_sys สำหรับ JavaScript built-in objects

```rust
use js_sys::{Array, Date, Math, Object, Reflect};

/// สร้าง JavaScript Array จาก Rust Vec
#[wasm_bindgen]
pub fn create_js_array(values: Vec<f64>) -> Array {
    let arr = Array::new();
    for v in values {
        arr.push(&JsValue::from_f64(v));
    }
    arr
}

/// ใช้ Math.random() ของ JavaScript
#[wasm_bindgen]
pub fn random_in_range(min: f64, max: f64) -> f64 {
    let r = Math::random();
    min + r * (max - min)
}

/// ดึง timestamp ปัจจุบัน (milliseconds since epoch)
#[wasm_bindgen]
pub fn current_timestamp_ms() -> f64 {
    Date::now()
}
```

#### Custom extern bindings — เรียก JavaScript functions โดยตรง

บางครั้งต้องการเรียก JavaScript functions ที่ไม่มีใน `web_sys` หรือ `js_sys`:

```rust
// นำเข้า JavaScript functions จาก module ของเรา
#[wasm_bindgen(module = "/src/js_bridge.js")]
extern "C" {
    /// เรียก JavaScript function `fetchData` จาก js_bridge.js
    #[wasm_bindgen(catch)]
    async fn fetch_data(url: &str) -> Result<JsValue, JsValue>;

    /// เรียก callback function ที่ JavaScript ส่งมา
    fn notify_progress(percent: f64);
}

// นำเข้า JavaScript global functions
#[wasm_bindgen]
extern "C" {
    /// เรียก setTimeout ของ browser
    fn setTimeout(closure: &Closure<dyn FnMut()>, millis: f64) -> f64;

    /// alert dialog
    fn alert(s: &str);
}
```

#### ไฟล์ js_bridge.js

```javascript
// src/js_bridge.js
export async function fetch_data(url) {
    const response = await fetch(url);
    if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }
    return response.json();
}

export function notify_progress(percent) {
    // อัปเดต UI progress bar
    const bar = document.getElementById('progress');
    if (bar) {
        bar.style.width = `${percent}%`;
    }
}
```

---

### ขั้นที่ 5: TypeScript Type Generation และ Custom Annotations

#### TypeScript .d.ts ที่ wasm-bindgen generate

เมื่อรัน `wasm-pack build` จะได้ไฟล์ `pkg/wasm_geometry.d.ts` อัตโนมัติ:

```typescript
/* tslint:disable */
/* eslint-disable */
/**
 * บวกเลข 2 ตัว
 */
export function add(a: number, b: number): number;
/**
 * ตรวจสอบว่าจำนวนเป็นจำนวนเฉพาะหรือไม่
 */
export function is_prime(n: number): boolean;
/**
 * ROT13 cipher
 */
export function rot13(input: string): string;
/**
 * Point class
 */
export class Point {
    free(): void;
    constructor(x: number, y: number);
    get x(): number;
    get y(): number;
    distance_to(other: Point): number;
    translate(dx: number, dy: number): Point;
    to_string(): string;
}
/**
 * Polygon class
 */
export class Polygon {
    free(): void;
    constructor();
    add_vertex(x: number, y: number): void;
    vertex_count(): number;
    perimeter(): number;
    area(): number;
    contains_point(px: number, py: number): boolean;
}
```

#### เพิ่ม JSDoc comments ใน Rust เพื่อ document TypeScript types

```rust
/// คำนวณ Bezier curve point ที่ parameter t
///
/// # Arguments
/// * `p0` - จุดเริ่มต้น (x, y)
/// * `p1` - control point (x, y)
/// * `p2` - จุดสิ้นสุด (x, y)
/// * `t` - parameter ระหว่าง 0.0 และ 1.0
///
/// # Returns
/// จุดบน curve เป็น `[x, y]` array
///
/// # Example
/// ```javascript
/// const point = bezier_point(0, 0, 50, 100, 100, 0, 0.5);
/// console.log(point); // [50, 50]
/// ```
#[wasm_bindgen]
pub fn bezier_point(
    p0x: f64, p0y: f64,
    p1x: f64, p1y: f64,
    p2x: f64, p2y: f64,
    t: f64,
) -> Vec<f64> {
    let mt = 1.0 - t;
    let x = mt * mt * p0x + 2.0 * mt * t * p1x + t * t * p2x;
    let y = mt * mt * p0y + 2.0 * mt * t * p1y + t * t * p2y;
    vec![x, y]
}
```

JSDoc comments ใน Rust จะปรากฏใน generated `.d.ts` file ทำให้ IDE ของ TypeScript แสดง documentation ได้

#### Custom type-safe wrappers ใน TypeScript

บางครั้ง generated types ยังไม่ type-safe พอ ให้สร้าง wrapper:

```typescript
// geometry-types.ts — Type-safe wrappers
import type * as WasmGeometry from '../pkg/wasm_geometry';

// Branded type สำหรับ angle (ป้องกัน confuse degrees กับ radians)
type Radians = number & { readonly _brand: 'radians' };
type Degrees = number & { readonly _brand: 'degrees' };

export const toRadians = (deg: Degrees): Radians =>
    (deg * (Math.PI / 180)) as Radians;

// Typed wrapper สำหรับ Polygon.contains_point
export function polygonContains(
    polygon: WasmGeometry.Polygon,
    point: { x: number; y: number }
): boolean {
    return polygon.contains_point(point.x, point.y);
}

// Factory function ที่ type-safe กว่า constructor ตรง ๆ
export interface PointLike { x: number; y: number }
export function createPoint(
    lib: typeof WasmGeometry,
    p: PointLike
): WasmGeometry.Point {
    return new lib.Point(p.x, p.y);
}
```

---

### ขั้นที่ 6: Error Handling — JsError, Result, และ Panic Hook

#### Panic Hook สำหรับ Debug

ตาม default เมื่อ Rust panic ใน WASM จะได้ error message ว่า "RuntimeError: unreachable" ซึ่งบอกอะไรไม่ได้เลย `console_error_panic_hook` ช่วยแปลง panic message ให้อ่านได้:

```toml
[dependencies]
wasm-bindgen = "0.2"
console_error_panic_hook = { version = "0.1", optional = true }

[features]
default = ["console_error_panic_hook"]
```

```rust
use wasm_bindgen::prelude::*;

/// เรียก function นี้ครั้งเดียวตอน init เพื่อตั้ง panic hook
#[wasm_bindgen(start)]
pub fn init_panic_hook() {
    // ตั้งค่า panic hook เมื่อ feature เปิดอยู่
    #[cfg(feature = "console_error_panic_hook")]
    console_error_panic_hook::set_once();
}
```

หลังจากนี้เมื่อ Rust panic ใน WASM จะได้ error message ที่อ่านได้ใน browser console:

```
panicked at 'index out of bounds: the len is 3 but the index is 5', src/lib.rs:42:5
```

#### Result<T, JsValue> — ส่ง Error ไป JavaScript

```rust
use wasm_bindgen::prelude::*;

/// ตัวอย่าง: parse string เป็น JSON และ validate
#[wasm_bindgen]
pub fn parse_and_validate_json(json_str: &str) -> Result<String, JsValue> {
    // parse JSON
    let value: serde_json::Value = serde_json::from_str(json_str)
        .map_err(|e| JsValue::from_str(&format!("JSON parse error: {}", e)))?;

    // validate — ต้องมี field "name" เป็น string
    let name = value.get("name")
        .and_then(|v| v.as_str())
        .ok_or_else(|| JsValue::from_str("Missing required field: 'name'"))?;

    Ok(format!("Valid! Name: {}", name))
}
```

ใน TypeScript ใช้ try/catch:

```typescript
try {
    const result = parse_and_validate_json('{"name": "Alice"}');
    console.log(result); // "Valid! Name: Alice"
} catch (e) {
    console.error("Rust error:", e); // ข้อความ error จาก Rust
}
```

#### JsError — Error object ที่ดีกว่า JsValue::from_str

```rust
use wasm_bindgen::prelude::*;

#[derive(Debug)]
pub enum GeometryError {
    InsufficientVertices { needed: usize, got: usize },
    InvalidIndex { index: usize, max: usize },
    DivisionByZero,
}

impl std::fmt::Display for GeometryError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            Self::InsufficientVertices { needed, got } =>
                write!(f, "Need at least {} vertices, got {}", needed, got),
            Self::InvalidIndex { index, max } =>
                write!(f, "Index {} out of bounds (max: {})", index, max),
            Self::DivisionByZero =>
                write!(f, "Division by zero"),
        }
    }
}

impl From<GeometryError> for JsValue {
    fn from(err: GeometryError) -> JsValue {
        // สร้าง JavaScript Error object (ไม่ใช่แค่ string)
        // ทำให้ stack trace ใน browser ดีขึ้น
        JsValue::from(js_sys::Error::new(&err.to_string()))
    }
}

#[wasm_bindgen]
pub fn compute_centroid_x(xs: &[f64]) -> Result<f64, JsValue> {
    if xs.is_empty() {
        return Err(GeometryError::InsufficientVertices {
            needed: 1,
            got: 0,
        }.into());
    }
    Ok(xs.iter().sum::<f64>() / xs.len() as f64)
}
```

ใน browser DevTools จะเห็น Error object ที่มี stack trace แทนที่จะเป็นแค่ string

---

### ขั้นที่ 7: Complex Data — serde-wasm-bindgen และ JSON Bridge

#### เพิ่ม Dependencies

```toml
[dependencies]
wasm-bindgen = "0.2"
serde = { version = "1.0", features = ["derive"] }
serde-wasm-bindgen = "0.6"
# หรือใช้ serde_json สำหรับ JSON string bridge:
serde_json = "1.0"
```

#### serde-wasm-bindgen — ส่ง Struct โดยตรงเป็น JS Object

```rust
use wasm_bindgen::prelude::*;
use serde::{Serialize, Deserialize};

/// Struct นี้จะ serialize เป็น JavaScript object อัตโนมัติ
#[derive(Serialize, Deserialize, Debug, Clone)]
pub struct ComputationResult {
    pub input_count: usize,
    pub mean: f64,
    pub variance: f64,
    pub min: f64,
    pub max: f64,
    pub histogram: Vec<u32>,
}

/// คำนวณ statistics และคืนเป็น JS object ผ่าน serde-wasm-bindgen
#[wasm_bindgen]
pub fn compute_statistics(data: &[f64]) -> Result<JsValue, JsValue> {
    if data.is_empty() {
        return Err(JsValue::from_str("data cannot be empty"));
    }

    let n = data.len() as f64;
    let mean = data.iter().sum::<f64>() / n;
    let variance = data.iter()
        .map(|&x| (x - mean).powi(2))
        .sum::<f64>() / n;
    let min = data.iter().cloned().fold(f64::INFINITY, f64::min);
    let max = data.iter().cloned().fold(f64::NEG_INFINITY, f64::max);

    // สร้าง histogram แบบง่าย (10 buckets)
    let bucket_count = 10;
    let mut histogram = vec![0u32; bucket_count];
    let range = max - min;
    if range > 0.0 {
        for &v in data {
            let idx = ((v - min) / range * (bucket_count - 1) as f64) as usize;
            histogram[idx.min(bucket_count - 1)] += 1;
        }
    }

    let result = ComputationResult {
        input_count: data.len(),
        mean,
        variance,
        min,
        max,
        histogram,
    };

    // serialize Rust struct → JavaScript object
    serde_wasm_bindgen::to_value(&result)
        .map_err(|e| JsValue::from_str(&e.to_string()))
}

/// รับ JavaScript object → Rust struct (deserialize)
#[wasm_bindgen]
pub fn process_config(config_js: JsValue) -> Result<String, JsValue> {
    #[derive(Deserialize)]
    struct Config {
        precision: u32,
        #[serde(default)]
        verbose: bool,
        label: String,
    }

    let config: Config = serde_wasm_bindgen::from_value(config_js)
        .map_err(|e| JsValue::from_str(&format!("Config parse error: {}", e)))?;

    Ok(format!(
        "Config OK: label='{}', precision={}, verbose={}",
        config.label, config.precision, config.verbose
    ))
}
```

#### JSON Bridge สำหรับ Complex Nested Types

บางครั้ง `serde-wasm-bindgen` มีปัญหากับ circular references หรือ non-serializable types ให้ใช้ JSON string เป็น intermediary:

```rust
use wasm_bindgen::prelude::*;
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize, Debug)]
pub struct TreeNode {
    pub value: i32,
    pub children: Vec<TreeNode>,
}

/// แปลง JSON string → Tree → compute sum → JSON string
/// ใช้ JSON bridge เมื่อ serde-wasm-bindgen ไม่รองรับ type โดยตรง
#[wasm_bindgen]
pub fn tree_sum_json(json: &str) -> Result<String, JsValue> {
    let tree: TreeNode = serde_json::from_str(json)
        .map_err(|e| JsValue::from_str(&e.to_string()))?;

    fn sum(node: &TreeNode) -> i32 {
        node.value + node.children.iter().map(sum).sum::<i32>()
    }

    let total = sum(&tree);
    Ok(serde_json::json!({ "sum": total }).to_string())
}
```

ใน TypeScript:
```typescript
const treeJson = JSON.stringify({
    value: 1,
    children: [
        { value: 2, children: [] },
        { value: 3, children: [{ value: 4, children: [] }] }
    ]
});

const resultJson = tree_sum_json(treeJson);
const { sum } = JSON.parse(resultJson);
console.log(sum); // 10
```

---

### ขั้นที่ 8: wasm-pack Build Targets และ npm Package

#### wasm-pack Build Targets

```bash
# Target: web — สำหรับ load โดยตรงใน browser ด้วย <script type="module">
wasm-pack build --target web

# Target: bundler — สำหรับ webpack, vite, rollup (ใช้บ่อยที่สุด)
wasm-pack build --target bundler

# Target: nodejs — สำหรับ Node.js CommonJS/ESM
wasm-pack build --target nodejs

# Target: no-modules — สำหรับ browser ที่ไม่ support ES modules (legacy)
wasm-pack build --target no-modules
```

**ความแตกต่างหลัก:**

| Target | โหลด .wasm | JavaScript glue | ใช้กับ |
|--------|-----------|----------------|-------|
| `web` | `fetch()` async | ES module | script type="module" |
| `bundler` | import inline | ES module | webpack/vite |
| `nodejs` | `fs.readFile` | CommonJS | Node.js |
| `no-modules` | `fetch()` async | global var | legacy browser |

#### โครงสร้าง Output ใน pkg/

```
pkg/
├── wasm_geometry_bg.wasm       ← WASM binary (หลัก)
├── wasm_geometry_bg.js         ← TypeScript: re-export ทุกอย่าง
├── wasm_geometry.js            ← JS glue (load + init)
├── wasm_geometry.d.ts          ← TypeScript type definitions
├── wasm_geometry_bg.wasm.d.ts  ← Type def สำหรับ .wasm file
└── package.json                ← npm package metadata
```

#### package.json ที่ wasm-pack generate

```json
{
  "name": "wasm-geometry",
  "version": "0.1.0",
  "files": [
    "wasm_geometry_bg.wasm",
    "wasm_geometry.js",
    "wasm_geometry_bg.js",
    "wasm_geometry.d.ts",
    "wasm_geometry_bg.wasm.d.ts"
  ],
  "module": "wasm_geometry.js",
  "types": "wasm_geometry.d.ts",
  "sideEffects": ["./snippets/*"]
}
```

สามารถแก้ `name` ใน `Cargo.toml` ภายใต้ `[package.metadata.wasm-pack.profile.release]` หรือแก้ package.json โดยตรง

#### ตั้งค่า Vite สำหรับ WASM

```bash
cd demo-app
npm create vite@latest . -- --template vanilla-ts
npm install
npm install ../pkg  # ติดตั้ง local package
```

```typescript
// demo-app/vite.config.ts
import { defineConfig } from 'vite';

export default defineConfig({
    // ต้องตั้งค่า optimizeDeps เพื่อให้ vite ไม่พยายาม pre-bundle .wasm
    optimizeDeps: {
        exclude: ['wasm-geometry'],
    },
    build: {
        target: 'esnext',  // ต้องการ top-level await
    },
});
```

```typescript
// demo-app/main.ts
import init, {
    Point,
    Polygon,
    compute_statistics,
    rot13,
} from 'wasm-geometry';

async function main() {
    // ขั้นตอนสำคัญ: ต้อง await init() ก่อนเสมอ
    // init() โหลด .wasm file ผ่าน fetch() และ instantiate WebAssembly module
    await init();

    console.log('WASM loaded!');

    // ใช้ functions
    console.log(rot13('Hello, World!'));  // Uryyb, Jbeyq!

    // ใช้ classes
    const p1 = new Point(0, 0);
    const p2 = new Point(3, 4);
    console.log(`Distance: ${p1.distance_to(p2)}`);  // 5

    // Statistics
    const data = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
    const stats = compute_statistics(new Float64Array(data));
    console.log('Stats:', stats);

    // สำคัญ: free WASM objects เมื่อไม่ใช้แล้ว
    p1.free();
    p2.free();
}

main().catch(console.error);
```

```html
<!-- demo-app/index.html -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>WASM Geometry Demo</title>
</head>
<body>
    <div id="app">
        <h1>Wasm Geometry</h1>
        <button id="run">Run Demo</button>
        <pre id="output"></pre>
    </div>
    <script type="module" src="/main.ts"></script>
</body>
</html>
```

---

### ขั้นที่ 9: Performance Optimization — ลด Boundary Crossings

#### ปัญหา: เรียก WASM function วนซ้ำ

```typescript
// ❌ วิธีผิด: เรียก WASM function 10,000 ครั้ง
// แต่ละ call มี overhead ของ boundary crossing
const points: Point[] = [];
for (let i = 0; i < 10_000; i++) {
    const p = new Point(Math.random(), Math.random());
    points.push(p);
    // ไม่ดี: ทุก new Point() คือ 1 boundary crossing
}

// ยิ่งแย่: เรียก method ใน loop
let totalDist = 0;
for (let i = 0; i < points.length - 1; i++) {
    totalDist += points[i].distance_to(points[i + 1]);  // crossing ทุก iteration!
}
```

#### ทางออก: Batch Operations บน Rust Side

```rust
use wasm_bindgen::prelude::*;

/// ─── Batch API: ประมวลผล array of points ทั้งหมดใน Rust เดียว ───────────────
/// ลด boundary crossings จาก O(n) เป็น O(1)

/// รับ flat array [x0, y0, x1, y1, ...] และคำนวณ total path length
#[wasm_bindgen]
pub fn path_length(coords: &[f64]) -> f64 {
    if coords.len() < 4 || coords.len() % 2 != 0 {
        return 0.0;
    }
    let mut total = 0.0;
    for i in (0..coords.len() - 2).step_by(2) {
        let dx = coords[i + 2] - coords[i];
        let dy = coords[i + 3] - coords[i + 1];
        total += (dx * dx + dy * dy).sqrt();
    }
    total
}

/// รับ flat array และหา bounding box [min_x, min_y, max_x, max_y]
#[wasm_bindgen]
pub fn bounding_box(coords: &[f64]) -> Vec<f64> {
    if coords.is_empty() || coords.len() % 2 != 0 {
        return vec![];
    }
    let mut min_x = f64::INFINITY;
    let mut min_y = f64::INFINITY;
    let mut max_x = f64::NEG_INFINITY;
    let mut max_y = f64::NEG_INFINITY;

    for i in (0..coords.len()).step_by(2) {
        let (x, y) = (coords[i], coords[i + 1]);
        if x < min_x { min_x = x; }
        if y < min_y { min_y = y; }
        if x > max_x { max_x = x; }
        if y > max_y { max_y = y; }
    }
    vec![min_x, min_y, max_x, max_y]
}

/// Normalize ค่าใน array ให้อยู่ในช่วง [0, 1] — batch in-place
#[wasm_bindgen]
pub fn normalize_batch(data: &[f64]) -> Vec<f64> {
    if data.is_empty() {
        return vec![];
    }
    let min = data.iter().cloned().fold(f64::INFINITY, f64::min);
    let max = data.iter().cloned().fold(f64::NEG_INFINITY, f64::max);
    let range = max - min;
    if range == 0.0 {
        return vec![0.0; data.len()];
    }
    data.iter().map(|&v| (v - min) / range).collect()
}

/// Moving average — ประมวลผลทั้ง array ในครั้งเดียว
#[wasm_bindgen]
pub fn moving_average(data: &[f64], window: usize) -> Vec<f64> {
    if window == 0 || data.len() < window {
        return vec![];
    }
    let mut result = Vec::with_capacity(data.len() - window + 1);
    let mut sum: f64 = data[..window].iter().sum();
    result.push(sum / window as f64);
    for i in window..data.len() {
        sum += data[i] - data[i - window];
        result.push(sum / window as f64);
    }
    result
}
```

ใน TypeScript:

```typescript
// ✅ วิธีที่ดี: ส่ง data ทั้งหมดเป็น typed array เดียว
// เพียง 1 boundary crossing!
const coords = new Float64Array(10_000 * 2);
for (let i = 0; i < 10_000; i++) {
    coords[i * 2] = Math.random() * 100;
    coords[i * 2 + 1] = Math.random() * 100;
}

// boundary crossing เพียงครั้งเดียว
const totalLength = path_length(coords);
const bbox = bounding_box(coords);
```

#### TextDecoder vs Rust String — String Performance

wasm-bindgen ใช้ JavaScript `TextDecoder` API ในการแปลง UTF-8 bytes จาก WASM memory เป็น JavaScript string สำหรับ string ที่ยาวมาก ๆ มี overhead พอสมควร

**เปรียบเทียบวิธีส่ง string data:**

```rust
// ─── วิธีที่ 1: คืน String (copy เสมอ) ──────────────────────────────────────
#[wasm_bindgen]
pub fn process_text_copy(input: &str) -> String {
    // wasm-bindgen copy input เข้า WASM memory ก่อน
    // แล้ว copy result กลับออกไป JS
    // สำหรับ string ยาว: ช้ากว่าวิธีที่ 2
    input.to_uppercase()
}

// ─── วิธีที่ 2: คืน Vec<u8> แล้วให้ JS decode เอง (zero-copy ไม่ได้จริง ๆ ────
// แต่ skip TextDecoder overhead สำหรับ binary data)
#[wasm_bindgen]
pub fn process_to_bytes(input: &str) -> Vec<u8> {
    input.to_uppercase().into_bytes()
}

// ─── วิธีที่ 3: ใช้ shared memory สำหรับ hot path ─────────────────────────────
// ส่ง pointer + length แทนที่จะ copy string
// (advanced pattern ต้องใช้ unsafe + js_sys::Uint8Array)
#[wasm_bindgen]
pub fn get_text_ptr_len(input: &str) -> Vec<u32> {
    // คืน [pointer, length] เพื่อให้ JS สร้าง view โดยตรง
    let ptr = input.as_ptr() as u32;
    let len = input.len() as u32;
    vec![ptr, len]
}
```

**กฎ Performance:**
1. ส่งข้อมูลเป็น `Float64Array` / `Uint8Array` แทน JavaScript `Array` เสมอ
2. Batch operations ให้มากที่สุด — ลด crossing ต่อ item
3. สำหรับ output ขนาดใหญ่ คืน pointer+length แล้วให้ JS อ่านจาก WASM memory โดยตรง
4. ใช้ `#[wasm_bindgen(skip)]` บน struct fields ที่ไม่ต้องการ expose เพื่อลด glue code

---

## การทดสอบ (Testing)

### Native Tests ด้วย cargo test

สำหรับ logic ที่จะรันใน WASM เราสามารถ test ได้บน native target ก่อน ซึ่งเร็วกว่ามาก เราสร้าง scratch project ที่มี logic เดียวกันกับ WASM library แล้วรัน `cargo test`:

```
running 21 tests
test tests::test_factorial_base_cases ... ok
test tests::test_matrix_multiply_dimension_mismatch ... ok
test tests::test_fibonacci_empty ... ok
test tests::test_matrix_out_of_bounds ... ok
test tests::test_matrix_multiply_identity ... ok
test tests::test_matrix_set_get ... ok
test tests::test_moving_average ... ok
test tests::test_normalize_constant ... ok
test tests::test_normalize_range ... ok
test tests::test_moving_average_window_too_large ... ok
test tests::test_point_distance ... ok
test tests::test_factorial_small ... ok
test tests::test_point_serde_roundtrip ... ok
test tests::test_polygon_area_triangle ... ok
test tests::test_fibonacci_sequence ... ok
test tests::test_polygon_area_square ... ok
test tests::test_polygon_perimeter_square ... ok
test tests::test_polygon_serde_roundtrip ... ok
test tests::test_rot13_basic ... ok
test tests::test_word_frequency ... ok
test tests::test_rot13_double_application ... ok

test result: ok. 21 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s

   Doc-tests wasm_bindgen_demo

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

### ตัวอย่าง Test Cases ที่ครอบคลุม Logic หลัก

```rust
#[cfg(test)]
mod tests {
    use super::*;

    // ─── factorial ───────────────────────────────────────────────────────────
    #[test]
    fn test_factorial_base_cases() {
        // factorial(0) และ factorial(1) ต้องเป็น 1 (base case)
        assert_eq!(factorial(0), 1);
        assert_eq!(factorial(1), 1);
    }

    #[test]
    fn test_factorial_small() {
        // 5! = 120, 10! = 3,628,800
        assert_eq!(factorial(5), 120);
        assert_eq!(factorial(10), 3_628_800);
    }

    // ─── fibonacci ────────────────────────────────────────────────────────────
    #[test]
    fn test_fibonacci_empty() {
        // count=0 ต้องคืน empty Vec
        assert_eq!(fibonacci_sequence(0), Vec::<u64>::new());
    }

    #[test]
    fn test_fibonacci_sequence() {
        // ตรวจสอบ 8 items แรก: 0 1 1 2 3 5 8 13
        assert_eq!(
            fibonacci_sequence(8),
            vec![0, 1, 1, 2, 3, 5, 8, 13]
        );
    }

    // ─── Point / Polygon ─────────────────────────────────────────────────────
    #[test]
    fn test_point_distance() {
        let p1 = Point::new(0.0, 0.0);
        let p2 = Point::new(3.0, 4.0);
        // 3-4-5 right triangle: distance = 5
        assert!((p1.distance_to(&p2) - 5.0).abs() < 1e-10);
    }

    #[test]
    fn test_polygon_area_square() {
        // unit square area = 1.0
        let poly = Polygon::new(vec![
            Point::new(0.0, 0.0), Point::new(1.0, 0.0),
            Point::new(1.0, 1.0), Point::new(0.0, 1.0),
        ]);
        assert!((poly.area() - 1.0).abs() < 1e-10);
    }

    #[test]
    fn test_polygon_area_triangle() {
        // right triangle legs 3,4 → area = 6
        let poly = Polygon::new(vec![
            Point::new(0.0, 0.0),
            Point::new(3.0, 0.0),
            Point::new(0.0, 4.0),
        ]);
        assert!((poly.area() - 6.0).abs() < 1e-10);
    }

    // ─── Matrix ───────────────────────────────────────────────────────────────
    #[test]
    fn test_matrix_multiply_identity() {
        let mut a = Matrix::new(2, 2);
        a.set(0, 0, 1.0).unwrap(); a.set(0, 1, 2.0).unwrap();
        a.set(1, 0, 3.0).unwrap(); a.set(1, 1, 4.0).unwrap();

        let mut id = Matrix::new(2, 2);
        id.set(0, 0, 1.0).unwrap(); id.set(1, 1, 1.0).unwrap();

        let result = a.multiply(&id).unwrap();
        // A * I = A
        assert_eq!(result.get(0, 0).unwrap(), 1.0);
        assert_eq!(result.get(1, 1).unwrap(), 4.0);
    }

    #[test]
    fn test_matrix_multiply_dimension_mismatch() {
        let a = Matrix::new(2, 3);
        let b = Matrix::new(2, 2);  // cols(a)=3 ≠ rows(b)=2
        assert!(a.multiply(&b).is_err());
    }

    // ─── String processing ────────────────────────────────────────────────────
    #[test]
    fn test_rot13_basic() {
        assert_eq!(rot13("Hello, World!"), "Uryyb, Jbeyq!");
    }

    #[test]
    fn test_rot13_double_application() {
        // ROT13 applied twice = identity
        let original = "The quick brown fox";
        assert_eq!(rot13(&rot13(original)), original);
    }

    // ─── serde round-trip ─────────────────────────────────────────────────────
    #[test]
    fn test_point_serde_roundtrip() {
        let p = Point::new(1.5, 2.5);
        let json = serde_json::to_string(&p).unwrap();
        let p2: Point = serde_json::from_str(&json).unwrap();
        assert_eq!(p, p2);
    }
}
```

### WASM Tests ด้วย wasm-pack test

สำหรับ tests ที่ต้องการ browser APIs หรือ WASM environment จริง:

```bash
# รัน tests ใน headless Chrome
wasm-pack test --headless --chrome

# รัน tests ใน headless Firefox
wasm-pack test --headless --firefox

# รัน tests ใน Node.js
wasm-pack test --node
```

```rust
// tests/web.rs — ต้องการ wasm-bindgen-test
use wasm_bindgen_test::*;

// ตั้งค่าให้รันใน browser
wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
fn test_greet() {
    let result = wasm_geometry::greet("World");
    assert!(result.contains("World"));
}

#[wasm_bindgen_test]
fn test_compute_statistics_browser() {
    use wasm_bindgen::JsValue;
    // สามารถใช้ browser APIs ได้ใน test นี้
    let data = vec![1.0, 2.0, 3.0, 4.0, 5.0];
    let result = wasm_geometry::compute_statistics(&data);
    assert!(result.is_ok());
}
```

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

### ข้อผิดพลาดที่ 1: ลืม `await init()` ก่อนใช้งาน

**อาการ:** `TypeError: Cannot read properties of undefined` หรือ `WebAssembly.instantiate is not a function`

```typescript
// ❌ ผิด: เรียก function ก่อน init()
import { rot13 } from 'wasm-geometry';
console.log(rot13("hello"));  // Error!

// ✅ ถูก: ต้อง await init() ก่อนเสมอ
import init, { rot13 } from 'wasm-geometry';
async function run() {
    await init();
    console.log(rot13("hello"));  // ✓ works
}
```

**สาเหตุ:** `.wasm` file ต้องถูก fetch และ instantiate แบบ async ก่อนจึงจะใช้ functions ได้ wasm-bindgen generate `init()` function ที่ทำขั้นตอนนี้

**วิธีแก้:** ใช้ top-level await (ต้องตั้ง `target: 'esnext'` ใน vite config) หรือ wrap ทุกอย่างใน async function

---

### ข้อผิดพลาดที่ 2: Memory Leak — ไม่เรียก `.free()` บน WASM Objects

**อาการ:** Memory ของ browser เพิ่มขึ้นเรื่อย ๆ เมื่อสร้าง WASM objects ในลูป

```typescript
// ❌ ผิด: สร้าง objects ในลูปโดยไม่ free
for (let i = 0; i < 1_000_000; i++) {
    const p = new Point(i, i);
    // p จะไม่ถูก GC เพราะ Rust holds ownership ของ heap memory
    // JavaScript GC รู้แค่ว่า JS wrapper object ถูก collect แต่ไม่ free Rust memory
}
// Rust heap memory รั่ว!

// ✅ ถูก: เรียก .free() เมื่อไม่ใช้แล้ว
for (let i = 0; i < 1_000_000; i++) {
    const p = new Point(i, i);
    const dist = p.distance_to(new Point(0, 0));
    p.free();  // free Rust heap memory
}
```

**สาเหตุ:** wasm-bindgen ใช้ reference counting บน Rust side JavaScript GC ไม่รู้ถึง Rust heap memory ดังนั้น JavaScript GC collect wrapper object แต่ไม่ได้ free Rust memory โดยอัตโนมัติ (จนกว่า finalizer จะทำงาน ซึ่งอาจช้า)

**วิธีแก้:** เรียก `.free()` เมื่อไม่ต้องการ object แล้ว หรือใช้ `try/finally` pattern:
```typescript
const p = new Point(1, 2);
try {
    const result = p.distance_to(origin);
    return result;
} finally {
    p.free();
}
```

---

### ข้อผิดพลาดที่ 3: ส่ง `Vec<Point>` ตรง ๆ ไม่ได้ — ต้องผ่าน JsValue

**อาการ:** `the trait bound Vec<Point>: FromWasmAbi is not satisfied`

```rust
// ❌ ผิด: Vec<Point> ไม่ implement FromWasmAbi โดยตรง
#[wasm_bindgen]
pub fn total_distance(points: Vec<Point>) -> f64 {
    // compile error!
    todo!()
}

// ✅ วิธีที่ 1: ใช้ flat array (ดีที่สุดสำหรับ performance)
#[wasm_bindgen]
pub fn total_distance_flat(coords: &[f64]) -> f64 {
    // coords = [x0, y0, x1, y1, ...]
    let mut total = 0.0;
    for i in (0..coords.len().saturating_sub(2)).step_by(2) {
        let dx = coords[i + 2] - coords[i];
        let dy = coords[i + 3] - coords[i + 1];
        total += (dx * dx + dy * dy).sqrt();
    }
    total
}

// ✅ วิธีที่ 2: ผ่าน JSON string
#[wasm_bindgen]
pub fn total_distance_json(json: &str) -> Result<f64, JsValue> {
    #[derive(serde::Deserialize)]
    struct P { x: f64, y: f64 }
    let pts: Vec<P> = serde_json::from_str(json)
        .map_err(|e| JsValue::from_str(&e.to_string()))?;
    let mut total = 0.0;
    for i in 0..pts.len().saturating_sub(1) {
        let dx = pts[i + 1].x - pts[i].x;
        let dy = pts[i + 1].y - pts[i].y;
        total += (dx * dx + dy * dy).sqrt();
    }
    Ok(total)
}
```

**สาเหตุ:** wasm-bindgen รองรับ `Vec<T>` เฉพาะเมื่อ T เป็น primitive types เช่น `f64`, `u8`, `i32` สำหรับ `Vec<CustomStruct>` ต้องใช้ flat arrays, JSON, หรือ `serde-wasm-bindgen`

---

### ข้อผิดพลาดที่ 4: `wasm-pack build` ล้มเหลวเพราะ `crate-type` ผิด

**อาการ:** `error: only one crate type allowed` หรือ binary ไม่มี `main.rs`

```toml
# ❌ ผิด: ไม่ได้ตั้ง crate-type
[package]
name = "my-wasm"

# ❌ ผิด: มีแต่ "bin" crate
[[bin]]
name = "my-app"

# ✅ ถูก: ต้องมี cdylib สำหรับ wasm-pack
[lib]
crate-type = ["cdylib", "rlib"]
```

**สาเหตุ:** wasm-pack ต้องการ `cdylib` เพื่อสร้าง dynamic library ที่ compile เป็น `.wasm` ถ้าไม่มีจะ build ไม่ผ่าน ส่วน `rlib` ต้องการสำหรับ `cargo test` บน native target

**วิธีแก้:** ตรวจสอบ `Cargo.toml` ว่ามี `[lib]` section ที่ถูกต้อง ถ้าเป็น binary project ให้แยก logic ออกมาเป็น library

---

### ข้อผิดพลาดที่ 5: `serde-wasm-bindgen` version ไม่ match กับ `wasm-bindgen`

**อาการ:** `error[E0277]: the trait bound JsValue: From<SerdeError> is not satisfied` หรือ type mismatch errors

```toml
# ❌ อาจเกิด mismatch
[dependencies]
wasm-bindgen = "0.2.90"
serde-wasm-bindgen = "0.4"  # ต้องการ wasm-bindgen < 0.2.84

# ✅ ถูก: ใช้ version ที่ compatible กัน
[dependencies]
wasm-bindgen = "0.2"
serde-wasm-bindgen = "0.6"  # compatible กับ wasm-bindgen 0.2.84+
```

**สาเหตุ:** `serde-wasm-bindgen` 0.4.x ใช้ internal API ของ `wasm-bindgen` ที่เปลี่ยนไปใน version 0.2.84 ต้องใช้ `serde-wasm-bindgen` 0.6+ สำหรับ `wasm-bindgen` เวอร์ชันใหม่

**วิธีแก้:** รัน `cargo update` แล้วดู error message เพื่อหา compatible versions

---

### ข้อผิดพลาดที่ 6: Panic ใน WASM ได้ Error ที่อ่านไม่รู้เรื่อง

**อาการ:** `RuntimeError: unreachable executed` ใน browser console โดยไม่มี stack trace

```rust
// ❌ ลืมตั้ง panic hook
#[wasm_bindgen]
pub fn dangerous_operation(index: usize) -> f64 {
    let data = vec![1.0, 2.0, 3.0];
    data[index]  // panic ถ้า index >= 3 แต่ error ไม่อ่านออก!
}

// ✅ ตั้ง panic hook ใน #[wasm_bindgen(start)]
use wasm_bindgen::prelude::*;

#[wasm_bindgen(start)]
pub fn init() {
    #[cfg(feature = "console_error_panic_hook")]
    console_error_panic_hook::set_once();
}
```

```toml
# ✅ เพิ่ม dependency
[dependencies]
console_error_panic_hook = { version = "0.1", optional = true }

[features]
default = ["console_error_panic_hook"]
```

หลังจากตั้ง panic hook แล้ว browser console จะแสดง:
```
panicked at 'index out of bounds: the len is 3 but the index is 5',
src/lib.rs:15:5
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

---

## การ Package และ Deploy

### Build สำหรับ Production

```bash
# Build สำหรับ bundler (Vite, webpack)
wasm-pack build --release --target bundler

# ผลลัพธ์ใน pkg/ มี .wasm ที่ optimize แล้ว
ls -la pkg/
# wasm_geometry_bg.wasm  (≈ 50-200KB ขึ้นอยู่กับ code)
# wasm_geometry.js
# wasm_geometry.d.ts
# package.json
```

### Optimize .wasm ขนาด

```bash
# ติดตั้ง wasm-opt (ส่วนหนึ่งของ binaryen toolchain)
cargo install wasm-opt  # หรือ brew install binaryen

# Optimize สำหรับขนาด
wasm-opt -Oz pkg/wasm_geometry_bg.wasm -o pkg/wasm_geometry_bg.wasm

# Optimize สำหรับ speed
wasm-opt -O3 pkg/wasm_geometry_bg.wasm -o pkg/wasm_geometry_bg.wasm
```

### Cargo.toml สำหรับ Production

```toml
[profile.release]
# optimize สำหรับขนาด (สำคัญมากสำหรับ WASM)
opt-level = "s"        # หรือ "z" สำหรับขนาดเล็กสุด
lto = true             # Link-time optimization
codegen-units = 1      # เพิ่ม optimization แต่ build ช้าลง
panic = "abort"        # ลบ panic machinery (ประหยัดขนาด ~10%)
```

### Publish ไปที่ npm

```bash
# 1. เพิ่ม scope และ registry ใน package.json (ถ้าต้องการ)
# 2. เพิ่มรายละเอียดใน Cargo.toml
# 3. Build
wasm-pack build --release --target bundler

# 4. Login npm
npm login --scope=@yourusername

# 5. Publish
wasm-pack publish

# หรือ publish โดยตรง
cd pkg && npm publish --access public
```

### ใช้ใน Project อื่น

```bash
# ติดตั้งจาก npm
npm install @yourusername/wasm-geometry

# หรือจาก GitHub
npm install github:yourusername/wasm-geometry
```

```typescript
// ใช้ใน project
import init, { compute_statistics } from '@yourusername/wasm-geometry';

await init();
const stats = compute_statistics(myData);
```

### GitHub Actions CI/CD

```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Rust
        uses: dtolnay/rust-toolchain@stable
        with:
          targets: wasm32-unknown-unknown

      - name: Install wasm-pack
        run: curl https://rustwasm.github.io/wasm-pack/installer/init.sh -sSf | sh

      - name: Run native tests
        run: cargo test

      - name: Build WASM
        run: wasm-pack build --release --target bundler

      - name: Publish to npm (on tag)
        if: startsWith(github.ref, 'refs/tags/')
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
        run: |
          cd pkg
          npm publish --access public
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: Image Processing Library (ระดับกลาง)

สร้าง WASM library สำหรับ process images ใน browser:

**ภารกิจ:**
- รับ `Uint8ClampedArray` จาก `ImageData` (pixel data RGBA)
- Implement image filters: grayscale, blur (box blur), brightness/contrast
- คืน processed `Uint8ClampedArray` กลับไป JavaScript
- Measure performance เทียบกับ pure JavaScript implementation

**Hint:**
```rust
#[wasm_bindgen]
pub fn grayscale(pixels: &mut [u8]) {
    // pixels = [R, G, B, A, R, G, B, A, ...]
    for chunk in pixels.chunks_exact_mut(4) {
        let gray = (0.299 * chunk[0] as f64
                  + 0.587 * chunk[1] as f64
                  + 0.114 * chunk[2] as f64) as u8;
        chunk[0] = gray;
        chunk[1] = gray;
        chunk[2] = gray;
        // chunk[3] = alpha ไม่เปลี่ยน
    }
}
```

**สิ่งที่จะได้เรียนรู้:** การทำงานกับ Canvas API, `ImageData`, zero-copy pixel processing

---

### แบบฝึกหัดที่ 2: CSV Parser และ Data Aggregator (ระดับกลาง-สูง)

สร้าง high-performance CSV parser ที่รันใน browser:

**ภารกิจ:**
- รับ CSV string ขนาดใหญ่ (หลาย MB) จาก JavaScript
- Parse ใน Rust (เร็วกว่า JavaScript parser 3-5x)
- Implement aggregations: sum, mean, group_by, filter
- คืนผลลัพธ์เป็น `serde-wasm-bindgen` objects
- เปรียบ performance กับ JavaScript CSV library อย่าง `papaparse`

**Hint:** ใช้ `csv` crate ใน Rust แต่ต้องระวัง feature flags:
```toml
[dependencies]
csv = { version = "1.3", default-features = false }  # ไม่ include serde ถ้าไม่ต้องการ
```

---

### แบบฝึกหัดที่ 3: Markdown Parser และ Syntax Highlighter (ระดับสูง)

**ภารกิจ:**
- Implement Markdown parser ง่าย ๆ ใน Rust (headers, bold, italic, code blocks)
- Expose `parse_markdown(input: &str) -> String` ที่คืน HTML
- เพิ่ม syntax highlighting สำหรับ code blocks
- ส่ง streaming output ผ่าน JavaScript callbacks

**Hint:**
```rust
#[wasm_bindgen]
pub fn parse_markdown(input: &str) -> String {
    // Implement basic CommonMark subset
    // หรือใช้ pulldown-cmark crate
    todo!()
}
```

เพิ่ม `pulldown-cmark` ใน dependencies:
```toml
[dependencies]
pulldown-cmark = "0.12"
```

---

### แบบฝึกหัดที่ 4: WebGL Particle System (ระดับสูงมาก)

**ภารกิจ:**
- ใช้ `web_sys::WebGlRenderingContext` ใน Rust
- Simulate 100,000 particles ใน WASM (physics update loop)
- Render ผ่าน WebGL โดยตรงจาก Rust โดยไม่ผ่าน JavaScript
- Implement gravitational attraction ระหว่าง particles

**Hint:**
```toml
[dependencies]
web-sys = { version = "0.3", features = [
    "WebGlRenderingContext",
    "WebGlBuffer",
    "WebGlProgram",
    "WebGlShader",
    "HtmlCanvasElement",
]}
```

```rust
#[wasm_bindgen]
pub struct ParticleSystem {
    positions: Vec<f32>,  // flat [x0, y0, x1, y1, ...]
    velocities: Vec<f32>,
    gl: WebGlRenderingContext,
    vertex_buffer: WebGlBuffer,
}
```

---

### แบบฝึกหัดที่ 5: Async WASM — Fetch และ Process Data (ระดับกลาง)

**ภารกิจ:**
- สร้าง `async fn fetch_and_process(url: &str)` ที่คืน `Promise`
- ใช้ `wasm-bindgen-futures` สำหรับ async/await ใน WASM
- Fetch JSON data, parse, aggregate, คืนผล
- Handle network errors อย่าง graceful

**Hint:**
```toml
[dependencies]
wasm-bindgen = "0.2"
wasm-bindgen-futures = "0.4"
web-sys = { version = "0.3", features = ["Request", "Response", "Window"] }
```

```rust
use wasm_bindgen_futures::JsFuture;
use web_sys::{Request, RequestInit, Response};

#[wasm_bindgen]
pub async fn fetch_and_sum(url: &str) -> Result<f64, JsValue> {
    let window = web_sys::window().unwrap();
    let resp_value = JsFuture::from(window.fetch_with_str(url)).await?;
    let resp: Response = resp_value.dyn_into()?;
    let json = JsFuture::from(resp.json()?).await?;
    // process json...
    Ok(0.0)
}
```

---

## สรุป

ในโปรเจคนี้เราได้เรียนรู้ workflow ทั้งหมดของการสร้าง Rust → WASM → npm package:

**สิ่งที่ได้สร้าง:**
- Library ที่มี `#[wasm_bindgen]` functions, structs, และ impl blocks
- Type-safe API ที่ generate TypeScript `.d.ts` อัตโนมัติ
- Error handling ผ่าน `Result<T, JsValue>` และ panic hook
- Complex data exchange ผ่าน `serde-wasm-bindgen` และ JSON bridge
- Performance-optimized batch API ที่ minimize boundary crossings
- npm package ที่ใช้งานได้ใน Vite + TypeScript project

**Pattern สำคัญที่ได้เรียน:**

| Pattern | ใช้เมื่อ |
|---------|---------|
| `#[wasm_bindgen]` fn | Expose simple computations |
| `#[wasm_bindgen]` struct | Stateful objects กับ methods |
| `Result<T, JsValue>` | Functions ที่อาจ fail |
| `serde-wasm-bindgen` | แลกเปลี่ยน complex objects |
| Flat `&[f64]` arrays | High-performance batch operations |
| JSON bridge | Nested/recursive data structures |
| Panic hook | Debug mode error messages |
| `wasm-pack build --release` | Production deployment |

**เชื่อมโยงไปโปรเจคถัดไป:**

โปรเจค J10 (Full-Stack Auth) จะนำความรู้ WASM ที่ได้มาผสมกับ full-stack Rust framework อย่าง `Axum` + `Leptos` สร้าง Single Page Application ที่ใช้ Rust ทั้ง frontend (WASM) และ backend (native) — Shared types, authentication flow, และ real-time updates ผ่าน WebSocket ล้วนเขียนด้วย Rust เพียงภาษาเดียว

---

**โปรเจคก่อนหน้า:** [Project J08: SSE Server](project-j08-sse-server.md) | **โปรเจคถัดไป:** [Project J10: Full-Stack Auth](project-j10-fullstack-auth.md)
