# Part 87: wasm-bindgen และ JavaScript Interop

> โมดูล: Full-Stack และ WebAssembly | ระดับ: สูง | เวลาโดยประมาณ: 260 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้ว่า `wasm-bindgen` คืออะไรจริง ๆ — proc macro ที่ generate ทั้งโค้ด marshaling ฝั่ง Rust และ
  glue code ฝั่ง JavaScript พร้อมอธิบายได้ว่ามันคือ "FFI boundary เข้าสู่โลกของ JavaScript" ในความหมาย
  เดียวกับที่ Part 43 สอน `extern "C"` เป็น FFI boundary เข้าสู่โลกของ C เพียงแต่ต้องทำงานหนักกว่าเพราะ
  ปลายทางเป็น dynamic type system ไม่ใช่ static type system แบบ C
- Export ฟังก์ชัน Rust ที่รับ/คืนค่า primitive และ `String` ให้ JavaScript เรียกได้ตรง ๆ พร้อมอธิบายได้ว่า
  ทำไมการส่ง `String` ข้ามขอบเขตมีต้นทุนจริงจากการ copy ข้อมูลเป็น UTF-8 bytes
- Export `struct`/`enum` ของ Rust ให้ JavaScript ใช้งานเป็น "opaque handle" ที่มี method เรียกกลับเข้าไป
  ใน WASM ได้ พร้อมเข้าใจว่า JavaScript ถือแค่ pointer/handle ไม่ได้ถือ struct ตรง ๆ
- เรียกฟังก์ชันของ JavaScript จากฝั่ง Rust ผ่าน `extern "C"` block และใช้ `web-sys` เพื่อเรียก Web API
  มาตรฐาน (console, DOM, fetch) แบบมี type-safety เต็มรูปแบบ รวมถึงใช้ `js-sys` เมื่อต้องคุยกับ JS built-in
  ทั่วไปที่ไม่ผูกกับ browser โดยเฉพาะ
- เขียนโค้ด `async`/`await` ที่รอผลลัพธ์จาก JavaScript `Promise` (เช่น `fetch()`) ผ่าน `wasm_bindgen_futures`
  พร้อมอธิบายได้ว่าทำไม WASM ในเบราว์เซอร์ **ไม่มี Tokio runtime** และใช้ event loop ของเบราว์เซอร์เองแทน
- ส่งข้อมูลโครงสร้างซับซ้อน (struct ที่มีหลาย field) ข้ามขอบเขตเป็น JS object ทั่วไปด้วย
  `serde-wasm-bindgen` และจัดการ error ข้ามขอบเขตอย่างถูกต้องด้วย `Result<T, JsValue>` แทนการยอมให้ panic
  หลุดออกไปโดยไม่ได้ตั้งใจ
- ระวังกับดักเรื่อง memory/lifetime ที่เกิดจาก WASM linear memory เป็นพื้นที่ความจำแยกจาก JS heap
  โดยสิ้นเชิง และรู้วิธีป้องกัน use-after-free ที่เกิดขึ้นได้จริงเมื่อ JavaScript ถือ handle ของ object
  ที่ฝั่ง Rust ปล่อยคืนไปแล้ว

## ความรู้ที่ต้องมีมาก่อน

- **Part 86 (WebAssembly เบื้องต้นด้วย Rust)** — บทก่อนหน้านี้สอนพื้นฐานของ WASM: มันคือ bytecode ที่รันเร็ว
  ใกล้เคียง native ในเบราว์เซอร์ (หรือ runtime อื่น ๆ), มี **linear memory model** เป็นพื้นที่ความจำก้อนเดียว
  ต่อเนื่องที่แยกจากทุกอย่างในโลก JavaScript โดยสิ้นเชิง และที่สำคัญที่สุดคือ **WASM ในระดับ core specification
  รู้จักแค่ประเภทตัวเลขล้วน ๆ** (`i32`, `i64`, `f32`, `f64`) ไม่มีแนวคิดเรื่อง `String`, object, หรือ struct
  อยู่ในตัวมันเองเลย — Part 86 ปิดท้ายด้วยคำถามที่บทนี้จะตอบเต็ม ๆ: "แล้วถ้าฟังก์ชันของเราต้องรับ/คืนค่าที่
  ไม่ใช่ตัวเลขล่ะ จะทำยังไง" นี่คือจุดเริ่มต้นของบทนี้โดยตรง
- **Part 43 (FFI: การเชื่อมต่อกับ C)** — บทนี้สอนแนวคิด "ขอบเขตข้ามภาษา" (FFI boundary) ผ่าน `extern "C"`,
  การแมป type, และอันตรายของการข้าม boundary ที่ compiler มองไม่เห็นสิ่งที่อยู่อีกฝั่ง — `wasm-bindgen`
  คือ FFI boundary อีกแบบหนึ่งที่แก้ปัญหาเดียวกันแต่ปลายทางเป็น JavaScript ไม่ใช่ C ซึ่งจะเห็นว่าปัญหาบางอย่าง
  เหมือนกันเป๊ะ (ownership ข้าม boundary, การแมป type, ต้นทุนของการ marshal ข้อมูล) แต่บางอย่างต่างกันมาก
  เพราะ JavaScript เป็น dynamically-typed และมี garbage collector ของตัวเอง
- **Part 44-45 (Macros และ Proc Macros)** — `#[wasm_bindgen]` คือ **attribute proc macro** ตัวหนึ่ง
  ตรงกับที่ Part 44-45 สอนไว้ว่า attribute macro รับ token stream ของ item ที่มันติดอยู่ (ฟังก์ชัน, struct,
  extern block) แล้ว generate code ใหม่ขึ้นมาแทนที่/เพิ่มเข้าไป — บทนี้จะเห็นภาพจริงว่า macro ระดับ production
  ที่ใช้กันทั่วโลกทำอะไรอยู่ข้างในกับ token stream ที่มันได้รับ
- **Part 46-50 (Async/Await, Futures, Executors, Tokio)** — บทนี้จะใช้ `async fn` และ `.await` แบบเดียวกับ
  ที่เรียนมา แต่จะเห็นความแตกต่างสำคัญ: ไม่มี Tokio runtime ให้เรียก `#[tokio::main]` หรือ `tokio::spawn`
  เพราะ WASM ในเบราว์เซอร์รันอยู่ใน**event loop ของ JavaScript เอง** ซึ่งเป็นคนละ executor กับ Tokio โดย
  สิ้นเชิง
- **Part 57-58 (Serde พื้นฐานและขั้นสูง)** — `serde-wasm-bindgen` ใช้ `Serialize`/`Deserialize` trait
  ตัวเดียวกับที่เรียนมาตรง ๆ เพียงแต่ปลายทางไม่ใช่ JSON string แต่เป็น `JsValue` (โครงสร้างข้อมูล JS จริง
  ในหน่วยความจำ ไม่ใช่ text ที่ต้อง parse อีกที)

## เนื้อหา

### 87.1 ทวนปัญหาจาก Part 86: ทำไม WASM ตัวเปล่า ๆ ไม่พอ

ก่อนจะเริ่มเรียน `wasm-bindgen` ต้องเข้าใจให้ชัดก่อนว่ามันมีอยู่เพื่อแก้ปัญหาอะไร เพราะถ้าไม่เห็นปัญหาให้ชัด
ก่อน จะรู้สึกว่าโค้ดที่ `wasm-bindgen` generate ให้นั้น "ซับซ้อนเกินจำเป็น"

ทวนจาก Part 86: WebAssembly ในระดับ **core specification** (สิ่งที่ทุก WASM runtime ต้องรองรับตามมาตรฐาน)
มีแค่ 4 ประเภทข้อมูลเท่านั้น: `i32`, `i64`, `f32`, `f64` — ตัวเลขล้วน ๆ ไม่มีอะไรมากไปกว่านี้ ฟังก์ชันที่
WASM module export ออกมาให้เรียกจากภายนอกจึงมีได้แค่ signature ที่ประกอบด้วยตัวเลขเหล่านี้เท่านั้น เช่น
`fn add(a: i32, b: i32) -> i32` คือฟังก์ชันที่ export ได้ตรง ๆ แบบไม่ต้องทำอะไรเพิ่ม

แต่โลกจริงของโปรแกรมไม่ได้อยู่แค่กับตัวเลข ลองนึกภาพฟังก์ชันธรรมดาที่สุดฟังก์ชันหนึ่ง:

```rust
fn greet(name: &str) -> String {
    format!("Hello, {}!", name)
}
```

ฟังก์ชันนี้รับ `&str` (ซึ่งจริง ๆ แล้วคือ fat pointer: pointer ไปยัง byte ข้อมูล + ความยาว) และคืนค่า
`String` (owned buffer ที่มี pointer, length, capacity) — ทั้งสอง type นี้ไม่มีทางแทนด้วยตัวเลขเดี่ยว ๆ
ตัวใดตัวหนึ่งจาก 4 ตัวที่ WASM รู้จักได้เลย คำถามคือ: **แล้วจะให้ JavaScript เรียกฟังก์ชันนี้ยังไง**

ถ้าไม่มีเครื่องมือช่วยอะไรเลย คุณจะต้องทำสิ่งเหล่านี้ด้วยมือทั้งหมด:

1. **จัดการหน่วยความจำเอง** — จอง (allocate) พื้นที่ใน WASM linear memory สำหรับเก็บ byte ของ string ที่
   JavaScript จะส่งเข้ามา, เขียน byte ของ string ลงไปในตำแหน่งนั้นจากฝั่ง JavaScript (ผ่าน `TextEncoder`
   เพื่อแปลง JS string เป็น UTF-8 bytes ก่อน เพราะ JS string ภายในเก็บเป็น UTF-16 ไม่ใช่ UTF-8)
2. **ส่ง pointer + length เป็นตัวเลขสองตัว** แทน `&str` ตัวเดียว เพราะ WASM export function รับได้แค่
   ตัวเลข — สมมติ export เป็น `fn greet(ptr: i32, len: i32) -> i32` (คืนค่าเป็น pointer ไปยัง string
   ผลลัพธ์ที่ allocate ไว้ในหน่วยความจำ WASM)
3. **อ่านผลลัพธ์กลับ** — JavaScript ต้องรู้ pointer และ length ของ string ผลลัพธ์ (ซึ่งต้องส่งกลับมาเป็น
   ตัวเลขอีกสองตัวไม่ทางใดก็ทางหนึ่ง เช่น เขียน length ลง memory ที่ตำแหน่งที่รู้ล่วงหน้า) แล้วอ่าน byte
   ออกมาจาก `WebAssembly.Memory` (ที่เข้าถึงได้ผ่าน `ArrayBuffer`) แปลงกลับเป็น JS string ด้วย `TextDecoder`
4. **ปล่อยคืนหน่วยความจำ** — ต้อง export ฟังก์ชัน `dealloc` ออกมาด้วย ให้ JavaScript เรียกเพื่อบอกฝั่ง Rust
   ว่า "ใช้ string ผลลัพธ์เสร็จแล้ว ปล่อยคืนได้" ไม่งั้นจะเกิด memory leak สะสมทุกครั้งที่เรียกฟังก์ชัน

สังเกตว่านี่แค่ฟังก์ชันเดียวที่รับ/คืน `String` ตัวเดียว ก็ต้องเขียน boilerplate ระดับ manual memory
management ทั้งสองฝั่งแล้ว — ถ้าต้องทำแบบนี้กับทุกฟังก์ชัน ทุก struct ที่ต้องข้ามขอบเขต โปรเจกต์จริงจะไม่มีทาง
scale ได้เลย นี่คือปัญหาที่ **`wasm-bindgen` เกิดมาเพื่อแก้โดยเฉพาะ**

### 87.2 `wasm-bindgen` คืออะไรจริง ๆ: proc macro + JS glue code สองฝั่ง

`wasm-bindgen` ไม่ใช่ runtime และไม่ใช่ framework — มันคือ **เครื่องมือสอง component ที่ทำงานคู่กัน**:

1. **`#[wasm_bindgen]` attribute proc macro** (ฝั่ง Rust, ทำงานตอน compile) — ตรงกับที่ Part 44-45 สอนไว้
   ว่า attribute macro รับ token stream ของ item ที่มันประดับอยู่ (ในที่นี้คือฟังก์ชัน, struct, หรือ
   `extern "C"` block) แล้ว **generate code ใหม่แทนที่** โค้ดต้นฉบับ สิ่งที่มันสร้างขึ้นมาคือโค้ด **marshaling**
   — โค้ดที่แปลง type ของ Rust (`String`, `&str`, struct) ให้เป็นรูปแบบตัวเลข/`JsValue` ที่ข้าม WASM boundary
   ได้ และแปลงกลับเมื่อรับค่าเข้ามา ทั้งหมดนี้เกิดขึ้น**อัตโนมัติ**โดยที่คุณไม่ต้องเขียน pointer/length ด้วยมือ
   เหมือนหัวข้อก่อนหน้าเลย
2. **`wasm-bindgen-cli`** (เครื่องมือ command-line ที่รันหลัง compile เสร็จ) — อ่าน metadata พิเศษที่ macro
   ฝังไว้ใน `.wasm` file ที่ compile ได้ แล้ว **generate ไฟล์ JavaScript "glue code"** ขึ้นมาอีกไฟล์ต่างหาก
   (เช่น `demo.js` / `demo_bg.wasm`) ไฟล์ JS ที่ generate มานี้คือสิ่งที่ทำให้การเรียก WASM function จาก
   JavaScript **"รู้สึกเหมือนเรียกฟังก์ชัน JS ธรรมดา"** — มันซ่อนรายละเอียดเรื่อง `TextEncoder`/`TextDecoder`,
   การจัดการ pointer, การ allocate/deallocate หน่วยความจำทั้งหมดไว้ข้างใน ผู้ใช้ฝั่ง JavaScript แค่เรียก
   `wasm.greet("Alice")` แล้วได้ string กลับมาตรง ๆ

**"metadata พิเศษที่ macro ฝังไว้"** ในข้อ 2 ไม่ใช่คำอธิบายเชิงนามธรรมลอย ๆ — verify ได้จริงด้วยการเปิดดู
ไฟล์ `.wasm` ที่ compile ได้ก่อนผ่าน `wasm-bindgen-cli` ด้วยเครื่องมือ `wasm-objdump` (จาก
[WABT](https://github.com/WebAssembly/wabt)):

```bash
wasm-objdump -h target/wasm32-unknown-unknown/debug/demo.wasm
```

ผลลัพธ์จริงส่วนที่เกี่ยวข้อง (ตัดบางส่วนออก):

```
   Custom start=... end=... (size=0x0005d9d3) "__wasm_bindgen_unstable"
   Custom start=... end=... (size=0x000741e8) "name"
   Custom start=... end=... (size=0x0000004d) "producers"
```

เห็น **custom section ที่ชื่อ `"__wasm_bindgen_unstable"` อยู่จริง** — WASM binary format อนุญาตให้มี
"custom section" แถมเข้าไปได้ (section ที่ WASM runtime มาตรฐานไม่สนใจและข้ามผ่านไปเฉย ๆ ตอนโหลด แต่เครื่องมือ
อื่นอ่านได้) `#[wasm_bindgen]` macro ใช้ช่องทางนี้แหละในการ**ฝัง schema ของทุกฟังก์ชัน/struct ที่ export/
import ไว้** (ชื่อ, จำนวน parameter, type ของแต่ละ parameter ฯลฯ ในรูปแบบไบนารีของตัวเอง) — `wasm-bindgen-cli`
อ่าน custom section นี้ออกมาตอนหลัง compile เพื่อรู้ว่าต้อง generate JS wrapper หน้าตาแบบไหนให้ตรงกับ
signature ที่คุณเขียนไว้จริงในซอร์ส Rust แบบเป๊ะ ๆ (ไม่ใช่การเดา signature แบบที่ Part 43 เตือนไว้ว่าเป็น
อันตรายของ `extern "C"` — ในกรณีนี้ signature ถูก "ยืนยัน" อัตโนมัติจาก compiler เอง ไม่ใช่คนพิมพ์ซ้ำสองที่)

#### 87.2.1 เปรียบเทียบกับ Part 43: FFI เข้าสู่ C เทียบกับ FFI เข้าสู่ JavaScript

ทั้งสองกรณีคือ "ข้ามขอบเขตภาษา" แต่มีความแตกต่างเชิงลึกที่สำคัญมาก:

| | FFI → C (Part 43) | FFI → JavaScript (`wasm-bindgen`) |
|---|---|---|
| ปลายทางเป็น type system แบบไหน | Static, กำหนด layout ตายตัวตอน compile (`#[repr(C)]`) | Dynamic — JS ไม่มี "struct layout" ที่ตายตัวเลย ทุกอย่างเป็น object ที่ตรวจสอบ shape ตอน runtime |
| ใครต้องรับผิดชอบ marshaling | ผู้เขียนโค้ดต้องทำเอง (`CString`, `#[repr(C)]`) | macro/CLI generate ให้อัตโนมัติเกือบทั้งหมด |
| การส่ง string | `CString`/`CStr` แปลง null-terminated string มือ | macro generate การแปลง UTF-8 ↔ JS UTF-16 string ให้เอง |
| การส่ง struct | ต้อง `#[repr(C)]` ให้ layout ตรงกันเป๊ะ | struct กลายเป็น "opaque handle" (ดูหัวข้อ 87.4) ไม่ต้องสนใจ layout เพราะ JS ไม่เคยเห็น layout เลย |
| ความปลอดภัยของ signature | compiler เชื่อ signature ที่คุณพิมพ์แบบไม่มีข้อกังขา (อันตรายถ้าพิมพ์ผิด) | `wasm-bindgen-cli` generate signature ของฝั่ง JS ให้ตรงกับฝั่ง Rust เสมอ เพราะมันอ่านจาก metadata ที่ macro ฝังไว้ในไฟล์ `.wasm` เอง ไม่ใช่คนพิมพ์เดา |
| runtime คั่นกลาง | ไม่มี — เรียกฟังก์ชันธรรมดาระดับ CPU instruction | มี **JS glue code layer** คั่นกลางเสมอ เพราะต้องแปลง representation ระหว่างสอง type system ที่ต่างกันโดยพื้นฐาน |

จุดสุดท้ายในตารางคือหัวใจสำคัญที่สุด: **การเรียก C จาก Rust ไม่มี runtime คั่นกลางเลย (zero-cost)** เพราะทั้ง
สองฝั่งใช้ ABI เดียวกันและ static memory layout เดียวกัน แต่ **การเรียก JavaScript จาก Rust ผ่าน WASM มี
ต้นทุนจากการ marshal ข้อมูลเสมอ** เพราะ WASM linear memory กับ JS heap เป็นพื้นที่ความจำคนละก้อนกันโดย
สิ้นเชิง (ย้อนกลับไป Part 86: WASM มองไม่เห็น JS object เลย และ JS engine ก็มองเห็น WASM memory เป็นแค่
`ArrayBuffer` ก้อนหนึ่งเท่านั้น) — ทุกครั้งที่ข้อมูลข้ามขอบเขตนี้ ต้อง **copy** ข้อมูลจากพื้นที่หนึ่งไปอีก
พื้นที่หนึ่งเสมอ ไม่มีทางเลี่ยง นี่คือเหตุผลเชิงลึกที่จะเห็นผลกระทบจริงในหัวข้อถัดไป

### 87.3 ฟังก์ชันพื้นฐาน: primitives และ strings ข้ามขอบเขต

เริ่มจากตัวอย่างที่เล็กที่สุดก่อน (ต่อจากที่ Part 86 อาจจะแนะนำไว้แล้วสั้น ๆ) แล้วขยายไปที่ `String`

#### 87.3.1 ตั้ง project ให้พร้อมสำหรับ `wasm-bindgen`

```toml
# Cargo.toml
[package]
name = "wasm-bindgen-demo"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2"
```

`crate-type = ["cdylib"]` คือสิ่งเดียวกับที่ Part 43 สอนไว้ตอนพูดถึงการ expose ฟังก์ชัน Rust ให้ C เรียก —
`cdylib` คือ "C dynamic library" ที่ compile เป็น shared object แบบไม่มี metadata ของ Rust ติดมาด้วย (ต่าง
จาก `rlib` ที่มี metadata เฉพาะของ Rust สำหรับให้ crate อื่นของ Rust link ต่อได้) WASM module ที่จะโหลดจาก
JavaScript ก็ใช้หลักการเดียวกัน: ต้องเป็น `cdylib` เพื่อให้ toolchain compile ออกมาเป็น `.wasm` แบบ
self-contained ที่ไม่ต้องพึ่ง Rust runtime ใด ๆ เพิ่ม

#### 87.3.2 ฟังก์ชันตัวเลข: จุดที่ไม่ต้องทำอะไรพิเศษเลย

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

ฟังก์ชันนี้รับ/คืนค่าเป็น `i32` ตรง ๆ ซึ่งเป็นหนึ่งใน 4 type ที่ WASM รู้จักอยู่แล้วโดยธรรมชาติ —
`#[wasm_bindgen]` แทบไม่ต้อง generate โค้ด marshaling อะไรเพิ่มเลยในกรณีนี้ (มันแค่ generate metadata และ
JS wrapper ที่เรียก export function ตรง ๆ) นี่คือกรณีที่ **ต้นทุนของการข้ามขอบเขตต่ำที่สุด** เพราะไม่มีการ
copy ข้อมูลใด ๆ เกิดขึ้น ตัวเลขถูกส่งผ่าน register/stack แบบเดียวกับที่ WASM calling convention กำหนดไว้
ตรง ๆ

#### 87.3.3 ฟังก์ชันที่รับ/คืน `String`: จุดที่การ marshaling เริ่มทำงานจริง

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn greet(name: &str) -> String {
    format!("สวัสดี, {}! ยินดีต้อนรับสู่ WASM", name)
}
```

จากมุมมองของคนเขียน Rust โค้ดนี้ดูเหมือนฟังก์ชันธรรมดาที่สุด — ไม่มีอะไรพิเศษให้เห็นเลย นี่คือความตั้งใจของ
`wasm-bindgen`: **ซ่อนความซับซ้อนทั้งหมดของการข้ามขอบเขตไว้หลัง macro** แต่สิ่งที่เกิดขึ้นจริงเบื้องหลังตอน
ฟังก์ชันนี้ถูกเรียกจาก JavaScript มีหลายขั้นตอน:

1. JavaScript เรียก `wasm.greet("Alice")` — glue code (`.js` ที่ generate จาก `wasm-bindgen-cli`) รับ JS
   string `"Alice"` เข้ามา
2. glue code ใช้ `TextEncoder` แปลง JS string (เก็บภายในเป็น UTF-16) เป็น UTF-8 bytes แล้ว**เขียน byte
   เหล่านั้นลงใน WASM linear memory** (ต้อง allocate พื้นที่ในฝั่ง WASM ก่อนผ่านฟังก์ชัน allocator ที่
   wasm-bindgen เตรียมไว้ให้)
3. glue code เรียก export function ตัวจริงของ WASM โดยส่ง **pointer + length** (ตัวเลขสองตัว) แทน string
   ตัวเดียว — WASM export function ที่แท้จริงมี signature ประมาณ
   `greet(ptr: i32, len: i32, retptr: i32)` ไม่ใช่ `greet(name: &str) -> String` แบบที่เขียนในซอร์ส Rust
   (macro แปลง signature ให้ตอน compile)
4. ฝั่ง Rust ที่รันอยู่ใน WASM อ่าน byte จาก pointer/length ที่ได้รับ แปลงกลับเป็น `&str` (ตรวจสอบ UTF-8
   validity ระหว่างทางเหมือนที่ Part 14 สอนเรื่อง `&str` การันตี valid UTF-8 เสมอ) รันฟังก์ชัน `greet` จริง
   ได้ `String` ผลลัพธ์ แล้ว allocate พื้นที่ใหม่เขียน byte ผลลัพธ์ลงไปในหน่วยความจำ WASM
5. glue code อ่าน pointer/length ของผลลัพธ์กลับมา ใช้ `TextDecoder` แปลง UTF-8 bytes กลับเป็น JS string
   แล้วปล่อยคืนหน่วยความจำ WASM ที่ใช้ชั่วคราวทั้งสองก้อน (input buffer และ output buffer)

**นี่คือต้นทุนจริงที่ควรรู้**: การส่ง `String`/`&str` ข้ามขอบเขตหนึ่งครั้ง มีการ **copy ข้อมูลอย่างน้อยสองรอบ**
(encode ตอนส่งเข้า, decode ตอนส่งออก) และมีการ **allocate/deallocate หน่วยความจำ WASM อย่างน้อยสองครั้ง**
ต่อการเรียกหนึ่งครั้ง เทียบกับฟังก์ชันที่รับ/คืนค่าเป็นตัวเลขล้วน ๆ ที่ไม่มี copy หรือ allocate เลย — ถ้า
ฟังก์ชันของคุณถูกเรียกซ้ำ ๆ ในลูปที่ performance-critical (เช่น เรียกทุก frame ของ animation) ต้นทุนนี้
สะสมได้จริง และเป็นเหตุผลที่โปรเจกต์ WASM ระดับ production หลายตัวเลือก**ลดจำนวนครั้งที่ข้าม boundary**
(batch การเรียกหลายอย่างเป็นครั้งเดียว) มากกว่าจะเรียกฟังก์ชันเล็ก ๆ ถี่ ๆ

Verify ด้วยการ build จริงและเรียกจาก Node.js (ผ่าน `wasm-pack build --target nodejs`):

```javascript
// test-greet.js
const wasm = require("./pkg/demo.js");
console.log(wasm.greet("Alice"));
console.log(wasm.add(3, 4));
```

ผลลัพธ์จริงที่รันได้ (verify ไว้ท้ายบทในหัวข้อทดสอบจริง):

```
สวัสดี, Alice! ยินดีต้อนรับสู่ WASM
7
```

#### 87.3.4 `wasm-pack build --target`: เลือก glue code ให้ตรงกับที่ที่จะใช้งาน

คำสั่ง `wasm-pack build` มี flag `--target` ที่กำหนดว่า glue code ที่ generate ออกมาจะมีรูปแบบไหน — เรื่องนี้
สำคัญพอที่ต้องเข้าใจก่อนจะไปหัวข้อต่อไป เพราะโค้ด JavaScript ตัวอย่างในบทนี้ทั้งหมด (ที่ทดสอบจริงด้วย
`require(...)`) ใช้ `--target nodejs` แต่โปรเจกต์จริงที่จะรันในเบราว์เซอร์ต้องเลือก target อื่น

| target | ใช้เมื่อไหร่ | รูปแบบไฟล์ที่ได้ (verify จริงจากการ build) |
|---|---|---|
| `nodejs` | รันบน Node.js ตรง ๆ (เช่น script test, CLI tool, server-side rendering) | `demo.js` เป็น CommonJS (`require`/`module.exports`), โหลด `.wasm` แบบ synchronous ผ่าน `fs.readFileSync` |
| `web` | ใช้ `<script type="module">` ใน HTML ตรง ๆ โดยไม่มี bundler | `demo.js` เป็น ES module ที่ export ฟังก์ชัน `init()` ต้องเรียก `await init()` ก่อนใช้งานฟังก์ชันอื่น (โหลด `.wasm` ผ่าน `fetch` เอง) |
| `bundler` | ใช้กับ webpack/Vite/Rollup ที่รองรับ import ไฟล์ `.wasm` โดยตรง | แยก `demo_bg.js` (logic จริง) ออกจาก `demo.js` (public API) และ import `demo_bg.wasm` เป็น ES module ตรง ๆ ให้ bundler จัดการโหลดเอง |
| `no-modules` | หน้า HTML เก่าที่ไม่รองรับ ES module เลย | export ทุกอย่างผ่าน global variable ตัวเดียว (เช่น `wasm_bindgen`) แทนการใช้ `import`/`export` |

ตัวอย่างความต่างที่ verify จริงจากการ build ด้วย `--target web` เทียบกับ `--target bundler` (ใช้โปรเจกต์
เดียวกันกับที่เรียนมาทั้งบท):

```javascript
// --target web (demo.js ที่ generate ออกมา) — self-contained, จัดการโหลด .wasm เอง
import { format_price } from './snippets/demo-xxxx/src/util.js';

export class BookingCart {
    free() {
        const ptr = this.__destroy_into_raw();
        wasm.__wbg_bookingcart_free(ptr, 0);
    }
    // ...
}
```

```javascript
// --target bundler (demo.js ที่ generate ออกมา) — ให้ bundler จัดการโหลด .wasm แทน
import * as wasm from "./demo_bg.wasm";
import { __wbg_set_wasm } from "./demo_bg.js";

__wbg_set_wasm(wasm);
wasm.__wbindgen_start();
export {
    BookingCart, Counter, TodoList, add, /* ... */
} from "./demo_bg.js";
```

สังเกตว่า `--target bundler` **ไม่มีขั้นตอน "โหลด `.wasm`" ให้เห็นเลยในโค้ดที่ generate มา** — มันไว้ใจให้
bundler (webpack/Vite) เป็นคนจัดการแปลง `import * as wasm from "./demo_bg.wasm"` ให้กลายเป็นโค้ดโหลด wasm
จริงตอน build time แทน (bundler สมัยใหม่ส่วนใหญ่รองรับ WASM module แบบ ES module import นี้อยู่แล้ว) ในขณะ
ที่ `--target web` ต้อง `fetch`/`instantiate` เอง จึงมีฟังก์ชัน `init()` ให้เรียกก่อนเสมอ (ตามที่เห็นใน
`index.html` ของหัวข้อ 87.11.2: `await init();` ก่อน `setup_todo_app(...)`)

#### 87.3.5 ขนาดของไฟล์ `.wasm`: `--dev` เทียบกับ `--release`

ยังอยู่ในเรื่องต้นทุน — อีกมิติหนึ่งที่ควรรู้คือ**ขนาดของไฟล์ `.wasm`** ที่ต้องส่งให้ผู้ใช้ดาวน์โหลดก่อนจะ
เริ่มรันได้เลย (ต่างจากโปรแกรม native ที่ไม่มีขั้นตอน "ดาวน์โหลดก่อนรัน" นี้) เทียบขนาดจริงจาก crate เดียวกัน
กับที่ใช้สอนทั้งบทนี้ (มีฟังก์ชัน/struct ทุกตัวที่เห็นในบทนี้รวมกันอยู่แล้ว):

| Build | คำสั่ง | ขนาดไฟล์ `demo_bg.wasm` จริง |
|---|---|---|
| `--dev` | `wasm-pack build --target nodejs --dev` | 531,123 bytes (~518 KB) — มี debug info ติดมาเต็ม |
| `--release` (ไม่มี `wasm-opt`) | `wasm-pack build --target nodejs --release --no-opt` | 155,207 bytes (~152 KB) |

แค่เปลี่ยนจาก `--dev` เป็น `--release` (ปิด debug info, เปิด optimization ของ `rustc` ตาม `[profile.release]`
ใน `Cargo.toml`) ก็ลดขนาดลงไปแล้วกว่า **3 เท่า** โดยที่ยังไม่ได้ผ่านขั้นตอน optimize เพิ่มเติมด้วยเครื่องมือ
`wasm-opt` (จาก [Binaryen](https://github.com/WebAssembly/binaryen)) เลย — `wasm-pack` จะเรียก `wasm-opt`
ให้อัตโนมัติต่อจาก `--release` เพื่อลดขนาดลงไปอีก (ตัด dead code, inline function เล็ก ๆ, ปรับ instruction
ให้กระชับขึ้นในระดับ WASM bytecode) โปรเจกต์จริงที่ใส่ใจ initial load time ควร**เปิด optimize ให้สุด**
เสมอก่อน deploy จริง ไม่ใช่แค่ปล่อยให้เป็นค่า default:

```toml
[profile.release]
opt-level = "z"  # optimize เน้นขนาดไฟล์เล็กที่สุด (มีตัวเลือก "s" ที่เน้นขนาดเล็กแต่ไม่สุดเท่า "z")
lto = true       # link-time optimization: มองเห็นข้าม crate boundary ตัด dead code ได้มากกว่า
```

(หมายเหตุความซื่อสัตย์: environment ที่ใช้ทดสอบบทนี้มี `wasm-opt` เวอร์ชันที่ไม่ตรงกับที่ `wasm-pack`
คาดหวังติดตั้งอยู่ใน `PATH` ทำให้ขั้นตอน `wasm-opt` ล้มเหลวตอนทดสอบจริง (`parse exception: invalid code
after misc prefix`) จึงรายงานเฉพาะตัวเลขก่อนเข้า `wasm-opt` ข้างบนเท่านั้น ไม่ได้เดาตัวเลขหลัง optimize
เพิ่มเติมที่ไม่ได้ verify จริง — แต่หลักการที่ `wasm-opt` ช่วยลดขนาดไฟล์ได้อีกในระดับเปอร์เซ็นต์ที่มีความหมาย
เป็นที่ยืนยันกันอย่างกว้างขวางในเอกสารและการใช้งานจริงของ Rust/WASM community)

### 87.4 Export struct/enum: JavaScript ถือ "opaque handle" ไม่ใช่ struct ตรง ๆ

ปัญหาที่ใหญ่กว่า `String` คือเมื่อต้อง export `struct` ที่มี state และ method — JavaScript ไม่มีแนวคิดเรื่อง
Rust struct layout เลย (ตรงกับที่หัวข้อ 87.2.1 อธิบายไว้ว่า JS เป็น dynamic type system) วิธีที่
`wasm-bindgen` แก้ปัญหานี้คือทำให้ struct กลายเป็น **opaque handle**: JavaScript ไม่เคยเห็น field ข้างใน
struct เลย มันถือแค่ **ตัวเลข pointer ที่ชี้ไปยัง struct จริงที่อาศัยอยู่ใน WASM linear memory** แล้วห่อ
pointer ตัวนั้นไว้ในคลาส JavaScript ที่ generate มาให้ — ทุกครั้งที่เรียก method บนคลาสนั้น มันจะส่ง pointer
กลับเข้าไปใน WASM เพื่อเรียก method ตัวจริงที่ทำงานอยู่ในฝั่ง Rust

#### 87.4.1 ตัวอย่างเล็กที่สุด: `Counter`

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub struct Counter {
    value: i32,
}

#[wasm_bindgen]
impl Counter {
    // #[wasm_bindgen(constructor)] บอกว่า method นี้คือสิ่งที่ `new Counter()` ในฝั่ง JS จะเรียก
    #[wasm_bindgen(constructor)]
    pub fn new() -> Counter {
        Counter { value: 0 }
    }

    pub fn increment(&mut self) {
        self.value += 1;
    }

    pub fn decrement(&mut self) {
        self.value -= 1;
    }

    // #[wasm_bindgen(getter)] ทำให้ฝั่ง JS เข้าถึงเป็น property `counter.value` แทนต้องเรียก
    // `counter.value()` เป็น method (JavaScript ไม่มีสิ่งที่เรียกว่า "field private ที่มี getter"
    // แบบ Rust แต่มี property accessor ที่ทำหน้าที่คล้ายกัน)
    #[wasm_bindgen(getter)]
    pub fn value(&self) -> i32 {
        self.value
    }
}
```

ฝั่ง JavaScript ที่เรียกใช้ (หลัง `wasm-pack build` แล้ว):

```javascript
const { Counter } = require("./pkg/demo.js");

const c = new Counter();
c.increment();
c.increment();
c.increment();
c.decrement();
console.log("counter value =", c.value);
```

ผลลัพธ์จริง:

```
counter value = 2
```

สิ่งที่เกิดขึ้นเบื้องหลังตอน `new Counter()`: glue code เรียก export function `counter_new()` ที่
`wasm-bindgen` generate ให้ ฝั่ง Rust `Box::new(Counter { value: 0 })` แล้วคืน raw pointer (แปลงจาก
`Box::into_raw`) กลับมาเป็นตัวเลข — glue code เก็บตัวเลข pointer นี้ไว้เป็น property ภายในของ JS object
ที่มันสร้างขึ้น (มักเห็นเป็น `this.__wbg_ptr` ถ้าเปิดไฟล์ `.js` ที่ generate มาดู) ทุกครั้งที่เรียก
`c.increment()` glue code จะส่ง pointer ตัวนั้นกลับเข้าไปเป็น argument แรก (`this` ที่ถูกแปลงเป็น
pointer numeric) เรียก export function `counter_increment(ptr)` ที่ฝั่ง Rust แปลง pointer กลับเป็น
`&mut Counter` (ผ่าน unsafe บางจุดที่ macro generate ไว้) แล้วรัน method จริง

#### 87.4.2 ตัวอย่างจากโดเมนของคอร์ส: `BookingCart`

ให้เห็นภาพการใช้งานจริงมากขึ้น มาดู struct ที่ใช้โดเมน "ระบบจองตั๋ว" ที่คอร์สนี้ใช้ซ้ำหลายบท —
สมมติหน้าเว็บจองตั๋วหนังต้องการให้ JavaScript สร้าง "ตะกร้าจองที่นั่ง" แล้วให้ Rust จัดการ logic ของราคา
และรายการที่นั่งทั้งหมด (โค้ดฝั่ง Rust ที่คำนวณราคาซับซ้อนกว่านี้มากในระบบจริงมักคุ้มค่าที่จะเขียนด้วย Rust
เพื่อความเร็วและเทสต์ได้ง่ายกว่า JavaScript ล้วน ๆ):

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub struct BookingCart {
    seats: Vec<String>,
    price_per_seat: f64,
}

#[wasm_bindgen]
impl BookingCart {
    #[wasm_bindgen(constructor)]
    pub fn new(price_per_seat: f64) -> BookingCart {
        BookingCart {
            seats: Vec::new(),
            price_per_seat,
        }
    }

    pub fn add_seat(&mut self, seat_code: String) {
        self.seats.push(seat_code);
    }

    pub fn remove_seat(&mut self, seat_code: &str) -> bool {
        if let Some(pos) = self.seats.iter().position(|s| s == seat_code) {
            self.seats.remove(pos);
            true
        } else {
            false
        }
    }

    pub fn seat_count(&self) -> usize {
        self.seats.len()
    }

    pub fn total_price(&self) -> f64 {
        self.seats.len() as f64 * self.price_per_seat
    }
}
```

ฝั่ง JavaScript:

```javascript
const { BookingCart } = require("./pkg/demo.js");

const cart = new BookingCart(250.0); // ราคาที่นั่งละ 250 บาท
cart.add_seat("A1");
cart.add_seat("A2");
cart.add_seat("B5");
console.log("จำนวนที่นั่ง:", cart.seat_count());
console.log("ราคารวม:", cart.total_price());

cart.remove_seat("A2");
console.log("จำนวนที่นั่งหลังลบ:", cart.seat_count());
console.log("ราคารวมหลังลบ:", cart.total_price());
```

ผลลัพธ์จริงที่ verify ได้ (ดูหัวข้อทดสอบจริงท้ายบท):

```
จำนวนที่นั่ง: 3
ราคารวม: 750
จำนวนที่นั่งหลังลบ: 2
ราคารวมหลังลบ: 500
```

จุดสำคัญที่ต้องเข้าใจ: `Vec<String>` (`seats`) และ `f64` (`price_per_seat`) ที่เก็บอยู่ข้างใน
`BookingCart` **ไม่เคยถูกส่งออกไปให้ JavaScript เห็นตรง ๆ เลยแม้แต่ครั้งเดียว** — JavaScript รู้จักแค่
"ตัวเลข pointer" ที่ชี้ไปยัง struct นี้ใน WASM memory เท่านั้น ทุกอย่างที่ดูเหมือนว่า JS "เข้าถึง field ได้"
ในความเป็นจริงคือการเรียก method (`seat_count()`, `total_price()`) ที่วิ่งกลับเข้าไปคำนวณในฝั่ง Rust แล้ว
คืนค่าตัวเลข/string ธรรมดาออกมาให้เท่านั้น — struct ตัวจริงไม่เคยย้ายที่จาก WASM linear memory ไปไหนเลย
ตราบใดที่ handle (pointer) ของมันยังมีชีวิตอยู่ฝั่ง JS

### 87.5 เรียก JavaScript จาก Rust: `extern "C"` และ `web-sys`

หัวข้อก่อนหน้าคือทิศทาง "Rust → JS เรียก Rust" ทั้งหมด ทีนี้มาดูทิศทางกลับ: **จะเรียกฟังก์ชันของ JavaScript
จากฝั่ง Rust ได้ยังไง**

#### 87.5.1 `extern "C"` block: import ฟังก์ชัน JS เข้ามาใน Rust

รูปแบบนี้คล้ายกับ Part 43 ตรง ๆ — `extern "C"` block คือการ**ประกาศ**ว่ามีฟังก์ชันชื่อนี้อยู่ (ในที่นี้คือ
อยู่ในโลกของ JavaScript ไม่ใช่ C library) แต่ต่างจาก Part 43 ตรงที่ต้องมี `#[wasm_bindgen]` ครอบ เพื่อบอก
ว่า "ฟังก์ชันที่ import มานี้อยู่ใน JS runtime ไม่ใช่ C ABI":

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
extern "C" {
    // ฟังก์ชัน global ธรรมดา — ในเบราว์เซอร์จริงคือ window.alert
    #[wasm_bindgen(js_name = alert)]
    fn js_alert(s: &str);

    // ฟังก์ชันที่อยู่ใต้ namespace ของ object อื่น (Math.random ใน JavaScript)
    #[wasm_bindgen(js_namespace = Math)]
    fn random() -> f64;
}

#[wasm_bindgen]
pub fn greet_with_alert(name: &str) {
    js_alert(&format!("สวัสดี, {}!", name));
}

#[wasm_bindgen]
pub fn roll_dice() -> u32 {
    (random() * 6.0).floor() as u32 + 1
}
```

สังเกตความคล้ายและความต่างกับ Part 43:

- **คล้าย**: ทั้งสองกรณีคือ "ประกาศว่ามีฟังก์ชันอยู่ที่อื่น ไม่มี implementation ในไฟล์นี้" — compiler เชื่อ
  signature ที่คุณเขียนโดยไม่มีทางตรวจสอบว่าตรงกับของจริงหรือไม่ (สำหรับ `extern "C"` ของ C, ปัญหาคือ
  signature ผิดจะทำให้เกิด undefined behavior แบบเงียบ ๆ — สำหรับ `wasm-bindgen`, ปัญหาที่คล้ายกันคือถ้า
  ชื่อฟังก์ชันหรือ namespace เขียนผิด มันจะ error ตอน**runtime**เมื่อ JS หาฟังก์ชันชื่อนั้นไม่เจอ แทนที่จะ
  error ตอน compile)
- **ต่าง**: ไม่ต้องมี `unsafe` ครอบการเรียกเหมือน Part 43 — เพราะ `wasm-bindgen` generate wrapper ที่
  ตรวจสอบและแปลง type ให้อัตโนมัติทุกจุด (การเรียก JS function ที่ import มาแบบนี้ยังคง "ปลอดภัยในความหมาย
  ของ Rust memory safety" เพราะไม่มีการ dereference raw pointer ตรง ๆ ในโค้ดที่คุณเขียน — ความเสี่ยงที่
  เหลืออยู่คือ "logic ผิด" ไม่ใช่ "memory unsafety" แบบ FFI ไป C)
- **`js_name`** ใช้เมื่อชื่อฟังก์ชันในฝั่ง Rust อยากตั้งต่างจากชื่อจริงใน JS (ในตัวอย่างนี้ตั้งชื่อ Rust ว่า
  `js_alert` เพื่อไม่ให้ชนกับคำสงวนหรือสร้างความสับสน แต่บอกว่าฝั่ง JS จริง ๆ ชื่อ `alert`)
- **`js_namespace`** ใช้เมื่อฟังก์ชันที่ต้องการเรียกไม่ได้อยู่ใน global scope ตรง ๆ แต่อยู่ใต้ object อื่น
  (`Math.random`, `console.log`, `JSON.stringify` ฯลฯ — แต่สำหรับ `console` และ Web API มาตรฐานอื่น ๆ
  ปกติไม่ต้องเขียน `extern "C"` มือเองแบบนี้ เพราะมี crate `web-sys` ที่ประกาศไว้ให้ครบแล้ว จะเห็นในหัวข้อ
  ถัดไป)

#### 87.5.2 Import ฟังก์ชันจากไฟล์ JS ของเราเอง: `#[wasm_bindgen(module = "...")]`

บางครั้งฟังก์ชันที่อยากเรียกจากฝั่ง Rust ไม่ใช่ของ built-in ของ JS หรือ Web API มาตรฐาน แต่เป็นฟังก์ชันที่
คุณเขียนขึ้นมาเองในไฟล์ `.js` ต่างหาก (เช่น logic เฉพาะทางที่ทำใน JavaScript สะดวกกว่า หรือต้องเรียกใช้
library JS ตัวอื่นที่ยังไม่มี Rust binding) `wasm-bindgen` รองรับกรณีนี้ผ่าน attribute `module`:

```javascript
// src/util.js — ไฟล์ JavaScript ธรรมดาที่เราเขียนเอง วางไว้คู่กับ src/lib.rs
export function format_price(amount) {
  return `${amount.toFixed(2)} บาท`;
}
```

```rust
use wasm_bindgen::prelude::*;

// module = "/src/util.js" บอกว่าฟังก์ชันที่ import เข้ามาต่อไปนี้อยู่ในไฟล์นี้ (path เป็น relative
// จาก root ของ crate) — wasm-bindgen-cli จะจัดการ "เย็บ" import statementที่ถูกต้องเข้าไปใน glue
// code ให้เองตาม target ที่เลือก (import ธรรมดาสำหรับ --target web/bundler, require() สำหรับ
// --target nodejs)
#[wasm_bindgen(module = "/src/util.js")]
extern "C" {
    fn format_price(amount: f64) -> String;
}

#[wasm_bindgen]
pub fn show_price(amount: f64) -> String {
    format_price(amount)
}
```

ผลลัพธ์จริงจากการรัน:

```javascript
const { show_price } = require("./pkg/demo.js");
console.log(show_price(1234.5));
```

```
1234.50 บาท
```

รายละเอียดที่ verify แล้วและควรรู้: เมื่อ build ด้วย `--target web`/`--target bundler` (ต่างจาก `--target
nodejs` ที่ใช้ในตัวอย่างข้างบน) `wasm-pack` จะ**copy ไฟล์ `util.js` ไปไว้ในโฟลเดอร์ `pkg/snippets/
<ชื่อ-crate>-<hash>/src/util.js` โดยอัตโนมัติ** พร้อมแก้ path การ `import` ในไฟล์ glue code หลักให้ชี้ไปที่
ตำแหน่งใหม่นั้นให้ถูกต้อง (verify จริงจากการ build ด้วย `--target web`: เห็น
`import { format_price } from './snippets/demo-762ef964a9e5629d/src/util.js';` ที่หัวไฟล์ `demo.js`) —
นี่หมายความว่าไฟล์ JS ที่คุณเขียนเองแบบนี้จะถูก "แพ็กไปด้วย" ทุกครั้งที่ distribute WASM module ของคุณ ไม่ต้อง
จัดการแยกเอง

#### 87.5.3 `catch`: จับ exception ของฟังก์ชัน JS ที่ import มา

ฟังก์ชัน JS จำนวนมาก (โดยเฉพาะที่เกี่ยวกับการ parse ข้อมูล เช่น `JSON.parse`) มีโอกาส **throw exception ได้
ตามปกติ** ถ้า import ฟังก์ชันแบบนี้เข้ามาแบบธรรมดาโดยไม่ระบุอะไรเพิ่ม การ throw ของมันจะกลายเป็นพฤติกรรม
เดียวกับ panic ที่ข้าม boundary (หัวข้อ 87.9.1) คือทำให้ WASM instance trap ทันที — ทางแก้คือเพิ่ม attribute
`catch` เพื่อบอกว่า "ฟังก์ชันนี้อาจ throw ได้ตามปกติ ให้แปลงเป็น `Result` แทนการปล่อยให้หลุดเป็น panic":

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
extern "C" {
    // catch: บอกว่าถ้าฟังก์ชันนี้ throw ใน JS ให้แปลงเป็น Err(JsValue) แทนที่จะปล่อยให้
    // เป็น uncaught exception ที่ทำให้ WASM instance พัง — js_namespace/js_name ระบุว่า
    // ฟังก์ชันจริงคือ JSON.parse ของ JavaScript
    #[wasm_bindgen(catch, js_namespace = JSON, js_name = parse)]
    fn json_parse(s: &str) -> Result<JsValue, JsValue>;
}

#[wasm_bindgen]
pub fn parse_json_safely(s: &str) -> Result<JsValue, JsValue> {
    json_parse(s)
}
```

ผลลัพธ์จริงจากการรัน:

```javascript
console.log(wasm.parse_json_safely(`{"a": 1}`));
try {
  wasm.parse_json_safely(`{invalid json`);
} catch (e) {
  console.log("caught:", e.toString());
}
```

```
{ a: 1 }
caught: SyntaxError: Expected property name or '}' in JSON at position 1 (line 1 column 2)
```

สังเกตว่า error message ที่จับได้คือ **`SyntaxError` ตัวจริงจาก JS engine เองเป๊ะ** (`"Expected property
name or '}' in JSON at position 1..."`) ไม่ใช่ `RuntimeError: unreachable` แบบ panic — นี่คือความแตกต่างที่
สำคัญมาก: `catch` ทำให้ exception ของ JS ฝั่งที่ import เข้ามา **เดินทางกลับไปเป็น JS exception ที่มีความหมาย
เหมือนเดิมทุกประการ** โดยไม่ผ่านการ panic ของ Rust เลย ต่างจากถ้าไม่ใส่ `catch` ที่ exception นั้นจะทำให้
เกิด panic (`unreachable`) และ WASM instance ไม่น่าเชื่อถืออีกต่อไปตามหัวข้อ 87.9.1 ทันที

#### 87.5.4 `web-sys`: typed binding สำหรับ Web API มาตรฐาน

การเขียน `extern "C"` block มือเองสำหรับทุกฟังก์ชันของ DOM/Web API เป็นงานที่ไม่มีที่สิ้นสุดและเสี่ยงพิมพ์
ผิด — crate [`web-sys`](https://crates.io/crates/web-sys) คือชุด binding ที่ **generate มาจาก Web IDL**
(ข้อกำหนดมาตรฐานของ Web API ที่ W3C/WHATWG เผยแพร่) ครอบคลุม Web API เกือบทั้งหมดที่เบราว์เซอร์รองรับ
(DOM, Fetch, WebSocket, Canvas, WebGL, ฯลฯ) พร้อม type ที่ถูกต้องและตรวจสอบได้ตอน compile — ตรงกับสิ่งที่
Part 43 บอกไว้ตอนท้ายว่า "โปรเจกต์จริงไม่เขียน binding มือเปล่าสำหรับ API ขนาดใหญ่" เพียงแต่ตรงนี้ปลายทาง
เป็น Web API แทน C library

เพราะ `web-sys` ครอบคลุม API มหาศาล (มีหลายพัน type/method) มันจึงถูกออกแบบให้ **แต่ละ type ต้องเปิดผ่าน
Cargo feature แยกกัน** เพื่อไม่ให้ binary บวมจากการ generate code สำหรับ API ที่ไม่ได้ใช้:

```toml
[dependencies.web-sys]
version = "0.3"
features = [
  "console",
  "Document",
  "Window",
  "Element",
  "HtmlElement",
  "Node",
]
```

ตัวอย่างจริง: เขียนข้อความไปยัง console ของเบราว์เซอร์ (ย้อนกลับไปที่ Part 86 ที่เกริ่นไว้ว่า
`console.log` คือช่องทาง debug หลักของ WASM ในเบราว์เซอร์ เพราะไม่มี `println!`/terminal ให้เห็นเหมือน
โปรแกรม native):

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn log_message(msg: &str) {
    web_sys::console::log_1(&JsValue::from_str(msg));
}
```

`console::log_1` คือฟังก์ชันที่ `web-sys` เตรียมไว้ตรงกับ `console.log(x)` ของ JavaScript ที่รับ argument
เดียว (มี `log_2`, `log_3`, ... สำหรับจำนวน argument ต่าง ๆ เพราะ Rust เป็นภาษาที่ต้องกำหนดจำนวน parameter
ตายตัว ไม่มี variadic function แบบ JS `console.log(...args)` ตรง ๆ) — สังเกตว่าต้องแปลง `&str` เป็น
`JsValue` ก่อนด้วย `JsValue::from_str` เพราะ `console::log_1` รับ `&JsValue` (representation กลางที่คุม
ค่า JS ทุกชนิดไว้ใน type เดียว จะอธิบายเต็ม ๆ ในหัวข้อ 87.8)

ตัวอย่างที่ใหญ่ขึ้น: จัดการ DOM จริง — เปลี่ยนข้อความของ element ตาม `id`:

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn update_greeting(element_id: &str, text: &str) -> Result<(), JsValue> {
    let window = web_sys::window()
        .ok_or_else(|| JsValue::from_str("ไม่พบ global window"))?;
    let document = window
        .document()
        .ok_or_else(|| JsValue::from_str("window ไม่มี document"))?;
    let element = document
        .get_element_by_id(element_id)
        .ok_or_else(|| JsValue::from_str("ไม่พบ element ตาม id ที่ระบุ"))?;
    element.set_text_content(Some(text));
    Ok(())
}
```

ทดสอบจริงกับ DOM จริง (jsdom) — element `<p id="greeting">` ที่มีข้อความเดิมอยู่ก่อน ถูกเปลี่ยนข้อความ
จากฝั่ง Rust ได้จริง และกรณีที่ id ไม่มีอยู่จริงก็คืน `Err` ที่มีข้อความชัดเจนตามที่ตั้งใจ ไม่ panic:

```
BEFORE: original text
AFTER: อัปเดตจาก Rust ผ่าน web-sys!
caught: ไม่พบ element ตาม id ที่ระบุ
```

โครงสร้างนี้สำคัญมากและจะเห็นซ้ำตลอดบทนี้: `web_sys::window()` คืน `Option<Window>` (เพราะในบางบริบทที่รัน
WASM อาจไม่มี `window` global อยู่จริง เช่นรันอยู่ใน Web Worker) และ `document()` คืน `Option<Document>`
เช่นกัน (worker ไม่มี DOM) — Rust บังคับให้ตรวจสอบทั้งสองกรณีผ่าน `Option` ตรงกับหลักการที่ Part 12 สอนไว้
ว่า "ความล้มเหลวที่เป็นไปได้ต้องปรากฏใน type" แทนที่ JavaScript ที่มักจะปล่อยให้ `document.getElementById`
คืน `null` เฉย ๆ แล้วโปรแกรมเมอร์ลืมเช็คจนพังตอน runtime — `?` operator ที่ใช้ร่วมกับ `ok_or_else` แปลง
`Option::None` เป็น `Err(JsValue)` ทำให้ error message ที่มีความหมายเดินทางกลับไปถึงฝั่ง JavaScript ได้
(รายละเอียดเต็ม ๆ อยู่ในหัวข้อ 87.9)

### 87.6 `js-sys`: JS built-in ทั่วไปที่ไม่ผูกกับเบราว์เซอร์โดยเฉพาะ

`web-sys` ครอบคลุมเฉพาะ Web API ที่เป็นของเบราว์เซอร์ (DOM, fetch, ฯลฯ) แต่ JavaScript ในฐานะภาษายังมี
built-in object พื้นฐานที่ไม่ใช่ของเบราว์เซอร์โดยเฉพาะ เช่น `Array`, `Object`, `Map`, `Promise`,
`Date`, `Math` — สิ่งเหล่านี้มีอยู่ใน**ทุก JS engine** ไม่ว่าจะรันในเบราว์เซอร์หรือ Node.js หรือที่อื่น ๆ
crate [`js-sys`](https://crates.io/crates/js-sys) คือชุด binding สำหรับสิ่งเหล่านี้โดยเฉพาะ (แยกจาก
`web-sys` เพราะขอบเขตความรับผิดชอบต่างกันชัดเจน)

ตัวอย่าง: รับ `js_sys::Array` (แทน JS `Array` จริง ๆ ที่ JavaScript ส่งเข้ามา) แล้วรวมค่าตัวเลขทั้งหมด:

```rust
use js_sys::Array;
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn sum_js_array(arr: &Array) -> f64 {
    let mut total = 0.0;
    for i in 0..arr.length() {
        if let Some(n) = arr.get(i).as_f64() {
            total += n;
        }
    }
    total
}

#[wasm_bindgen]
pub fn make_js_array(items: Vec<i32>) -> Array {
    let arr = Array::new();
    for item in items {
        arr.push(&JsValue::from_f64(item as f64));
    }
    arr
}
```

`Vec<i32>` ใน parameter ของ `make_js_array` ก็เป็นอีกกรณีที่ `wasm-bindgen` แปลงให้อัตโนมัติ — `Vec<T>`
ของตัวเลข primitive แปลงเป็น JS `Array`/`TypedArray` ได้ตรง ๆ โดยไม่ต้องผ่าน `js-sys` มือเอง (นี่คือ
built-in support ของ `wasm-bindgen` เอง) แต่ตัวอย่างนี้ตั้งใจใช้ `js_sys::Array` เพื่อโชว์การเขียนโค้ดที่
ทำงานกับ JS `Array` แบบดิบ ๆ ทีละ index ผ่าน `.get(i)`/`.push(&value)` ซึ่งเป็นแบบที่จำเป็นเมื่อคุณต้อง
รับ array ที่มี**ค่าหลายชนิดผสมกัน** (JS array ไม่บังคับให้ทุก element เป็น type เดียวกันเหมือน `Vec<T>`
ของ Rust) — กรณีนั้น `Vec<T>` ธรรมดาใช้ไม่ได้ ต้องใช้ `js_sys::Array` หรือ `JsValue` ตรง ๆ

ผลลัพธ์จริงจากการรัน:

```
arr = [ 1, 2, 3, 4, 5 ]
sum = 15
```

`arr.get(i)` คืนค่าเป็น `JsValue` (เพราะ JS array อาจมี element ชนิดไหนก็ได้) `.as_f64()` คือ method บน
`JsValue` ที่พยายามแปลงเป็น `f64` คืนเป็น `Option<f64>` — ถ้า element ตัวนั้นไม่ใช่ตัวเลข (เช่นเป็น string
หรือ `null`) จะได้ `None` กลับมา ตรงกับปรัชญาของ `Option` อีกครั้งที่ Part 12 สอนไว้: การแปลง type ที่
"อาจล้มเหลว" ต้องคืนเป็น `Option`/`Result` เสมอ

`js-sys` ไม่ได้มีแค่ `Array` — มันครอบคลุม JS built-in อื่น ๆ ที่ใช้บ่อยด้วย เช่น `js_sys::Object` (สร้าง/
ตรวจสอบ JS object ทั่วไปผ่าน `Object::keys`, `Object::entries`), `js_sys::Map`/`js_sys::Set` (แทน `Map`/
`Set` ของ JS ที่มี key เป็น value ใดก็ได้ ต่างจาก `HashMap` ของ Rust ที่ key ต้อง implement `Hash`),
และ `js_sys::Promise` (ตัวแทนของ JS `Promise` แบบดิบที่ `wasm_bindgen_futures::JsFuture` ใช้แปลงเป็น Rust
`Future` ในหัวข้อ 87.7 นั่นเอง) — หลักการเลือกใช้เหมือนกันหมด: ใช้ `js-sys` เมื่อต้องทำงานกับ "รูปร่างของ
ข้อมูลที่ไม่คงที่แน่นอนล่วงหน้า" และใช้ type ที่เจาะจงกว่า (`Vec<T>`, struct ที่ derive serde,
`web_sys::*`) เมื่อรู้ shape ของข้อมูลชัดเจนอยู่แล้ว เพราะให้ type-safety ที่ตรวจสอบได้ตอน compile มากกว่า

### 87.7 Async interop: `wasm_bindgen_futures` และ `JsFuture`

หัวข้อนี้คือจุดที่ Part 46-50 (async/await, futures, Tokio) กับโลกของ WASM มาเจอกันโดยตรง — และมีข้อควร
ระวังสำคัญมากหนึ่งข้อที่ต้องเข้าใจก่อนเขียนโค้ดสักบรรทัด

#### 87.7.1 ข้อเท็จจริงที่สำคัญที่สุด: ไม่มี Tokio runtime ในเบราว์เซอร์

ทวนจาก Part 47 (Futures และ Executors): `async fn` ใน Rust แค่สร้างค่าที่ implement trait `Future` —
มันไม่ทำงานเองจนกว่าจะมี **executor** มา poll มัน `#[tokio::main]` ทำหน้าที่สร้าง Tokio runtime (thread
pool + reactor สำหรับ I/O) ที่ poll future ให้อัตโนมัติ — แต่ **Tokio runtime พึ่งพา OS-level API อย่าง
thread, epoll/kqueue/IOCP ที่ WASM ในเบราว์เซอร์ไม่มีให้เรียกเลย** (เบราว์เซอร์ไม่อนุญาตให้ WASM module
สร้าง OS thread ตรง ๆ หรือเรียก syscall ระดับต่ำแบบนั้น ด้วยเหตุผลด้าน security sandbox)

สิ่งที่มีอยู่แทนคือ **event loop ของ JavaScript เอง** — เบราว์เซอร์ (และ Node.js) มี event loop ที่จัดการ
`Promise`, `setTimeout`, callback ของ DOM event ฯลฯ อยู่แล้วโดยธรรมชาติ `wasm_bindgen_futures` คือสะพาน
เชื่อมระหว่างสองโลกนี้: มันทำให้ Rust `Future` **ทำงานโดยใช้ JS event loop เป็น executor** แทน Tokio
โดยสมบูรณ์ — ไม่มี thread pool, ไม่มี reactor ของ Rust ใด ๆ เกี่ยวข้องเลยในบริบทนี้

ผลกระทบเชิงปฏิบัติที่ต้องรู้:

- **ห้ามใช้ `tokio::spawn`, `tokio::time::sleep`, `tokio::net::TcpStream` ฯลฯ ในโค้ดที่ compile เป็น WASM
  สำหรับเบราว์เซอร์** — API เหล่านี้พึ่งพา Tokio runtime ที่ไม่มีอยู่จริงในบริบทนี้ จะ panic ตอน runtime
  ด้วย error ประมาณ `"there is no reactor running, must be called from the context of a Tokio 1.x
  runtime"` (ถ้าพยายามคอมไพล์ผ่านมาได้เลย เพราะบางส่วนของ Tokio ไม่รองรับ `wasm32` target ด้วยซ้ำตอน
  compile)
- **ให้ใช้ `wasm_bindgen_futures::spawn_local` แทน `tokio::spawn`** สำหรับรัน future แบบ "fire and forget"
  บน event loop ของ JS
- **`async fn` ที่ประดับด้วย `#[wasm_bindgen]` จะถูกแปลงเป็นฟังก์ชันที่คืนค่าเป็น JS `Promise` โดยอัตโนมัติ**
  — นี่คือกลไกที่ทำให้ JavaScript เขียน `await wasm.some_async_fn()` ได้ตามธรรมชาติ

#### 87.7.2 ตัวอย่างจริง: await ผลลัพธ์จาก JS `fetch()`

```rust
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use wasm_bindgen_futures::JsFuture;
use web_sys::{Request, RequestInit, RequestMode, Response};

#[wasm_bindgen]
pub async fn fetch_text(url: String) -> Result<JsValue, JsValue> {
    let mut opts = RequestInit::new();
    opts.set_method("GET");
    opts.set_mode(RequestMode::Cors);

    let request = Request::new_with_str_and_init(&url, &opts)?;

    let window = web_sys::window()
        .ok_or_else(|| JsValue::from_str("ไม่พบ global window"))?;
    // window.fetch_with_request(...) คืนค่าเป็น js_sys::Promise (ตรงกับ fetch() ของ JS ที่คืน
    // Promise<Response> เสมอ) — JsFuture::from(...) แปลง JS Promise ให้กลายเป็น Rust Future
    // ที่ .await ได้ตรง ๆ นี่คือหัวใจของ wasm_bindgen_futures ทั้งหมด
    let resp_value = JsFuture::from(window.fetch_with_request(&request)).await?;
    let resp: Response = resp_value.dyn_into()?;

    if !resp.ok() {
        return Err(JsValue::from_str(&format!(
            "HTTP error: status = {}",
            resp.status()
        )));
    }

    // resp.text() เองก็คืน Promise<String> อีกชั้น (การอ่าน body ของ response เป็น async
    // operation ใน Fetch API) — ต้อง await อีกรอบ
    let text_promise = resp.text()?;
    let text_value = JsFuture::from(text_promise).await?;
    Ok(text_value)
}
```

สังเกตว่ามีการ `.await` **สองครั้ง**: ครั้งแรกรอ `fetch()` เสร็จ (ได้ `Response` object กลับมา — แค่ header
มาถึงแล้ว ยังไม่ได้อ่าน body) ครั้งที่สองรอ `.text()` เสร็จ (อ่าน body ทั้งหมดออกมาเป็น string) — นี่ตรงกับ
พฤติกรรมจริงของ Fetch API ใน JavaScript ที่แยกสองขั้นตอนนี้ออกจากกันเสมอ (`fetch(url).then(r =>
r.text())`) เพราะการอ่าน body เป็น stream ที่อาจใช้เวลาต่างจากการได้ header กลับมา

ฝั่ง JavaScript เรียกใช้:

```javascript
const { fetch_text } = require("./pkg/demo.js");

async function main() {
  const body = await fetch_text("http://127.0.0.1:PORT/hello");
  console.log("ได้ข้อมูลจาก server:", body);
}

main();
```

ผลลัพธ์จริงจากการรัน (ทดสอบด้วย local HTTP server จริงที่เขียนด้วย Node `http` module ฟังอยู่บน
`127.0.0.1` แล้วเรียก `fetch_text` จากฝั่ง WASM ข้ามไปจริง ๆ ผ่าน `wasm_bindgen_futures::JsFuture`):

```
fetch_text result: สวัสดีจาก server จริงที่รันอยู่ใน localhost
fetch_text (404) caught: HTTP error: status = 404
```

บรรทัดที่สองยืนยันว่า `Result<JsValue, JsValue>` ทำงานถูกต้องตามที่ออกแบบ: เมื่อ server ตอบกลับด้วย
status `404` (route `/missing` ที่ไม่มีจริง) ฟังก์ชัน `fetch_text` คืน `Err(...)` ที่มีข้อความชัดเจนว่า
`"HTTP error: status = 404"` แทนที่จะพยายามอ่าน body ต่อไปแล้วได้ข้อมูลที่ไม่มีความหมาย และ error message
นี้เดินทางข้าม `.await` กลับไปถึง `catch` ฝั่ง JavaScript ได้ตรงตามที่เขียนไว้เป๊ะ

**จุดสำคัญที่ต้องย้ำอีกครั้ง**: โค้ด Rust ข้างบนไม่มี `#[tokio::main]`, ไม่มี `Runtime::new()`, ไม่มีอะไร
เกี่ยวกับ Tokio เลยแม้แต่นิดเดียว — สิ่งที่ทำให้ `.await` ทำงานได้คือ **glue code ที่ `#[wasm_bindgen]`
generate ให้กับ `async fn`** ที่แปลงมันให้กลายเป็นฟังก์ชันคืนค่า `Promise`, แล้ว JavaScript's event loop
(ที่มีอยู่แล้วในตัวเบราว์เซอร์/Node.js) เป็นคนขับเคลื่อนให้ `Promise` นั้นทำงานจนเสร็จ — `Future` ของ Rust
ในบริบทนี้แค่เป็น "ตัวกลาง" ที่คอย register ว่าจะให้ทำอะไรต่อเมื่อ `Promise` resolve เท่านั้น ไม่มี Rust
executor ของตัวเองรันอยู่เบื้องหลังเลย

### 87.8 ส่งข้อมูลโครงสร้างซับซ้อน: `serde-wasm-bindgen`

หัวข้อ 87.4 สอนวิธี export struct เป็น "opaque handle" — เหมาะกับกรณีที่ JavaScript ไม่จำเป็นต้องเห็น
ข้อมูลข้างในเลย แค่เรียก method ผ่าน handle พอ แต่บางครั้งคุณ**ต้องการส่งข้อมูลจริงข้ามไปให้ JS อ่าน/แก้ไข
ตรง ๆ** เช่น รายการ ticket ทั้งหมดที่ต้อง render เป็น HTML — กรณีนี้ opaque handle ไม่เหมาะ (JS ต้องเรียก
method ทีละ field ซึ่งช้าและอ่านยาก) สิ่งที่ต้องการคือแปลง struct เป็น **JS object ธรรมดา** ที่มี property
ตรงกับ field ของ Rust struct

`serde-wasm-bindgen` ทำหน้าที่นี้: มันใช้ `Serialize`/`Deserialize` trait ตัวเดียวกับที่ Part 57-58 สอนไว้
ทุกประการ (คือ derive `#[derive(Serialize, Deserialize)]` แบบเดิม) แต่ปลายทางไม่ใช่ JSON string เหมือน
`serde_json` — ปลายทางคือ **`JsValue`** (โครงสร้างข้อมูล JS จริงในหน่วยความจำ ไม่ใช่ text ที่ต้อง parse
อีกที) ทำให้เร็วกว่าการแปลงเป็น JSON string แล้วให้ JS `JSON.parse()` อีกที (ตัดขั้นตอน serialize เป็น
text ที่ไม่จำเป็นออกไปทั้งหมด)

```rust
use serde::{Deserialize, Serialize};
use wasm_bindgen::prelude::*;

#[derive(Serialize, Deserialize)]
pub struct Ticket {
    pub seat: String,
    pub price: f64,
}

#[wasm_bindgen]
pub fn make_ticket_js(seat: String, price: f64) -> Result<JsValue, JsValue> {
    let ticket = Ticket { seat, price };
    // to_value: Rust struct -> JsValue (JS object ธรรมดา { seat: "...", price: ... })
    serde_wasm_bindgen::to_value(&ticket).map_err(|e| JsValue::from_str(&e.to_string()))
}

#[wasm_bindgen]
pub fn total_price_from_js(tickets: JsValue) -> Result<f64, JsValue> {
    // from_value: JsValue (JS array ของ object) -> Vec<Ticket>
    let tickets: Vec<Ticket> = serde_wasm_bindgen::from_value(tickets)
        .map_err(|e| JsValue::from_str(&e.to_string()))?;
    Ok(tickets.iter().map(|t| t.price).sum())
}
```

ฝั่ง JavaScript:

```javascript
const { make_ticket_js, total_price_from_js } = require("./pkg/demo.js");

const t1 = make_ticket_js("A1", 250.0);
console.log(t1); // { seat: 'A1', price: 250 } — เป็น JS object ธรรมดาจริง ๆ ไม่ใช่ opaque handle
console.log("seat:", t1.seat, "price:", t1.price); // เข้าถึง property ได้ตรง ๆ

const total = total_price_from_js([
  { seat: "A1", price: 250.0 },
  { seat: "A2", price: 250.0 },
  { seat: "B5", price: 300.0 },
]);
console.log("total:", total);
```

ผลลัพธ์จริงจากการรัน:

```
{ seat: 'A1', price: 250 }
seat: A1 price: 250
total: 800
```

สังเกตว่า `t1` ที่ log ออกมาเป็น **JS object ธรรมดาล้วน ๆ** (`{ seat: 'A1', price: 250 }`) ไม่มีร่องรอยของ
`class`/opaque handle ใด ๆ ให้เห็นเลย ต่างจาก `Counter`/`BookingCart` ในหัวข้อ 87.4 ที่ log ออกมาจะเห็นเป็น
`Counter { ... }` พร้อม method `free()` ติดมาด้วยอย่างชัดเจน

ความแตกต่างที่สำคัญที่สุดระหว่างสองแนวทาง:

| | Opaque handle (87.4) | `serde-wasm-bindgen` (87.8) |
|---|---|---|
| JS เห็น field ข้างในไหม | ไม่เห็นเลย ต้องเรียก method | เห็นตรง ๆ เป็น property ของ object ธรรมดา |
| ข้อมูลอยู่ที่ไหนจริง ๆ หลังส่ง | ยังอยู่ใน WASM linear memory เดิม (แค่ pointer เดินทาง) | ถูก **copy** ทั้งหมดออกมาเป็น JS object ใหม่ใน JS heap |
| แก้ไขข้อมูลจาก JS ได้ไหม | ไม่ได้ตรง ๆ ต้องเรียก method ที่ Rust exposeไว้ | ได้ตรง ๆ (แต่การแก้จะไม่ส่งผลกลับไปที่ฝั่ง Rust เว้นแต่ส่งกลับมาอีกรอบ) |
| เหมาะกับ | state ที่ต้องการให้ Rust ควบคุม logic ทั้งหมด | ข้อมูลที่แค่ต้องการ "ส่งไปแสดง" หรือ "ส่งมาให้ประมวลผลครั้งเดียว" |
| ต้นทุน | ต่ำมาก (แค่ pointer, ไม่ copy struct) | มีการ copy ข้อมูลทั้งหมดเสมอ (คล้ายกับต้นทุนของ `String` ในหัวข้อ 87.3) |

### 87.9 Error handling ข้ามขอบเขต: panic เทียบกับ `Result<T, JsValue>`

นี่คือหัวข้อที่มีความสำคัญเชิง production สูงมาก เพราะพฤติกรรมของ panic ที่ข้าม WASM boundary
**ต่างจากพฤติกรรมของ panic ในโปรแกรม native โดยสิ้นเชิง** และเป็นเรื่องที่ต้อง verify ด้วยการรันจริง
ไม่ใช่เดาเอาจากความรู้สึก

#### 87.9.1 เมื่อ Rust panic ข้างใน WASM: มันไม่ได้ "unwind" แบบโปรแกรม native

ทวนจาก Part 12/41: ในโปรแกรม native ปกติ `panic!` จะ unwind stack (เรียก destructor ของทุกตัวแปรที่ยัง
ไม่ถูก drop ไล่ขึ้นไปตาม call stack) แล้วจบ thread นั้น (หรือทั้งโปรแกรมถ้าเกิดใน main thread และไม่มีการ
`catch_unwind`) — แต่ **target `wasm32-unknown-unknown` โดย default compile ด้วย panic strategy แบบ
`abort` ไม่ใช่ `unwind`** (ส่วนหนึ่งเพราะการ unwind ข้าม WASM boundary มีความซับซ้อนสูงและ toolchain
ยังไม่รองรับสมบูรณ์ในทุกกรณี) เมื่อเกิด panic จริง สิ่งที่เกิดขึ้นคือ WASM module รัน**instruction
`unreachable`** ซึ่งเป็น instruction พิเศษที่บอกกับ WASM runtime ว่า "ห้ามมาถึงจุดนี้เด็ดขาด — ถ้ามาถึง
คือมีบั๊ก" — WASM runtime (ทั้งของเบราว์เซอร์และ Node.js) จะ**throw JavaScript exception ทันที** เมื่อเจอ
instruction นี้

(หมายเหตุสำหรับผู้ที่อยากลองเชิงลึกกว่านี้: `wasm-pack build` มี flag ทดลอง `--panic-unwind` ที่ compile
`std` ใหม่ด้วย `-Z build-std` บน nightly toolchain เพื่อเปลี่ยน panic strategy เป็น `unwind` จริง ๆ ทำให้
`catch_unwind` ทำงานข้าม FFI boundary ได้ในบางกรณี — แต่ ณ วันที่เขียนบทนี้ยังเป็น feature ระดับ experimental
ที่ต้องใช้ nightly Rust และยังไม่ใช่แนวทางที่แนะนำสำหรับโปรเจกต์ production ทั่วไป แนวทางที่ยังคงเป็นมาตรฐาน
คือสิ่งที่บทนี้สอน: ใช้ `Result<T, JsValue>` สำหรับ error ที่คาดหวังได้ และปฏิบัติต่อ panic ว่าเป็นบั๊กที่ควร
ทำให้ instance ทั้งตัวไม่น่าเชื่อถืออีกต่อไป)

ทดสอบจริง (จะเห็นผลลัพธ์ verify แล้วในหัวข้อทดสอบท้ายบท):

```rust
#[wasm_bindgen]
pub fn may_panic(x: i32) -> i32 {
    if x < 0 {
        panic!("may_panic ไม่รับค่าติดลบ ได้รับ x = {}", x);
    }
    x * 2
}
```

```javascript
const { may_panic } = require("./pkg/demo.js");

console.log(may_panic(5)); // ทำงานปกติ

try {
  may_panic(-1);
} catch (e) {
  console.log("จับ exception ได้:", e);
}
```

ผลลัพธ์จริงจากการรัน (build ด้วย `wasm-pack build --target nodejs --dev` แล้วรันผ่าน Node.js — ยังไม่ได้
ติดตั้ง panic hook พิเศษใด ๆ ในขั้นนี้):

```
10
จับ exception ได้: RuntimeError: unreachable
    at demo.wasm.__rustc[...]::__rust_abort (wasm://wasm/demo.wasm-...:wasm-function[2290]:...)
    at demo.wasm.__rustc[...]::__rust_start_panic (...)
    ...
    at demo.wasm.demo::may_panic::h... (...)
typeof e: object   e instanceof Error: true   e.message: unreachable
```

**สิ่งที่ต้องเข้าใจให้ชัดที่สุดจากผลลัพธ์จริงนี้**: exception ที่ JavaScript จับได้เป็น `Error` object จริง
(`e instanceof Error === true`) แต่ **`e.message` มีค่าเป็น `"unreachable"` เสมอ ไม่ใช่ข้อความที่คุณเขียน
ไว้ใน `panic!(...)`** — ข้อความ panic ที่คุณตั้งใจเขียน (`"may_panic ไม่รับค่าติดลบ ได้รับ x = -1"`) **หายไป
โดยสมบูรณ์** ไม่ปรากฏอยู่ใน exception ที่ JS จับได้เลยแม้แต่นิดเดียว สิ่งที่เหลืออยู่ให้เห็นคือ stack trace
ระดับ WASM function name ที่พอบอกเบาะแสได้ (ในตัวอย่างนี้ยังเห็นชื่อ `demo::may_panic` เพราะ build ด้วย
`--dev` ที่มี debuginfo ติดไปด้วย — ถ้าเป็น release build แบบ optimize เต็มที่และผ่าน `wasm-opt` มา stack
trace แบบนี้จะยิ่งอ่านไม่ออกกว่านี้อีกมาก) นี่คือเหตุผลที่ต้องมี panic hook แยกต่างหาก (87.9.2) ถ้าต้องการ
เห็นข้อความ panic จริงตอน debug

อีกจุดที่ verify จริงแล้วและควรระวัง**การตีความผิด**: ในตัวอย่างทดสอบนี้ หลังจาก `may_panic(-1)` panic ไปแล้ว
การเรียก `may_panic(5)` ซ้ำอีกครั้งบน**instance เดิม**ยังคืนค่า `10` ที่ถูกต้องตามปกติ — **แต่นี่ไม่ใช่การันตี
ที่พึ่งพาได้** เหตุผลคือ `may_panic` เป็นฟังก์ชันที่ไม่มี state ใด ๆ ค้างอยู่ (ไม่ mutate ตัวแปร global หรือ
struct ที่แชร์กันอยู่) จึง "รอดมาได้" ในกรณีนี้โดยบังเอิญ — ถ้า panic เกิดขึ้น**ระหว่างที่กำลัง mutate struct
ที่มี invariant ซับซ้อนกว่านี้** (เช่น กำลังอัปเดตหลาย field ของ `BookingCart` แล้ว panic ตรงกลางระหว่าง
field แรกกับ field ที่สอง) เพราะ `wasm32-unknown-unknown` ใช้ panic strategy แบบ `abort` (ไม่ unwind)
โค้ดที่ควรจะรันเพื่อคืนค่า struct กลับสู่ state ที่สมบูรณ์**จะไม่ถูกรันเลย** ทำให้ state ที่เหลืออยู่ใน WASM
memory ค้างอยู่ในสภาพที่ไม่สมบูรณ์ — แนวทางที่ปลอดภัยที่สุดในโปรเจกต์จริงคือ**ปฏิบัติต่อทุก panic ว่า
WASM instance ทั้งตัวไม่น่าเชื่อถืออีกต่อไปแล้วเสมอ** (reload instance ใหม่) แม้ว่าในเคสง่าย ๆ แบบสถานะไม่มี
ก็ตาม เพราะไม่มีทางรู้ล่วงหน้าว่า struct ไหนกำลังถูก mutate อยู่ตอนที่ panic เกิดขึ้นจริงในโค้ด production
ที่ซับซ้อนกว่านี้

#### 87.9.2 ทำให้ panic message มีประโยชน์ขึ้น: `console_error_panic_hook`

เพื่อให้ error message จาก panic อ่านออกและ debug ได้จริง (ไม่ใช่แค่ `RuntimeError: unreachable`
เปล่า ๆ) โปรเจกต์ WASM ทุกโปรเจกต์ที่ตั้งใจ debug ได้จริงต้องติดตั้ง panic hook พิเศษ:

```toml
[dependencies]
console_error_panic_hook = "0.1"
```

```rust
#[wasm_bindgen(start)]
pub fn main() {
    // ติดตั้ง panic hook ที่พิมพ์ panic message ไปที่ console.error ของเบราว์เซอร์/Node
    // ก่อนที่ WASM จะ trap ด้วย unreachable — ถ้าไม่ตั้งค่านี้ panic message ของ Rust
    // (ที่คุณเขียนไว้ใน panic!("...")) จะหายไปเงียบ ๆ ไม่ถูกส่งไปที่ใดเลย
    console_error_panic_hook::set_once();
}
```

`#[wasm_bindgen(start)]` คือ attribute พิเศษที่บอกว่าฟังก์ชันนี้ให้รันอัตโนมัติทันทีที่ WASM module ถูกโหลด
เสร็จ (คล้ายกับ constructor ของ module) — เหมาะสำหรับงาน setup ที่ต้องทำครั้งเดียวตอนเริ่มต้นแบบนี้

หลังตั้ง panic hook แล้ว รันโค้ดเดิมอีกครั้ง — สิ่งที่ปรากฏบน `console.error` จริง (capture มาแบบคำต่อคำ)
คือ:

```
panicked at src/lib.rs:213:9:
may_panic ไม่รับค่าติดลบ ได้รับ x = -1

Stack:

Error
    at .../pkg/demo.js:607:25
    at logError (.../pkg/demo.js:981:18)
    at __wbg_new_227d7c05414eb861 (.../pkg/demo.js:606:57)
    at demo.wasm.console_error_panic_hook::Error::new::...
    at demo.wasm.console_error_panic_hook::hook_impl::...
    at demo.wasm.console_error_panic_hook::hook::...
    at demo.wasm.std::panicking::panic_with_hook::...
```

แต่ค่าที่ `.catch()`/`try-catch` ฝั่ง JavaScript จับได้จริงยังคงเป็น**แค่** `e.message === "unreachable"`
เหมือนก่อนติดตั้ง hook ทุกประการ (verify แล้วว่าไม่เปลี่ยน) — panic hook แค่เพิ่มการพิมพ์ log ที่มีบรรทัด
เลขและข้อความ panic จริงไปที่ `console.error` **ก่อนที่**การ trap ของ WASM (`unreachable`) จะเกิดขึ้น มัน
ไม่เปลี่ยนพฤติกรรมการ throw ของ exception ที่ JS จับได้เลยแม้แต่นิดเดียว (นี่คือความแตกต่างสำคัญที่มักเข้าใจ
ผิด: panic hook ช่วยเรื่อง **debugging** ในหน้าจอ console เท่านั้น ไม่ได้ช่วยเรื่อง **error recovery** ที่
โปรแกรมยังต้องพึ่ง `Result` เพื่อให้ JS จับ error message ที่มีความหมายได้จริง)

#### 87.9.3 `Result<T, JsValue>`: วิธีที่ถูกต้องสำหรับ error ที่ "คาดหวังว่าจะเกิดได้"

ตรงกับหลักการของ Part 12 ทุกประการ: **error ที่เป็นไปได้ในทางธุรกิจ (expected error) ไม่ควร panic** —
`panic!` เหมาะกับบั๊กที่ไม่ควรเกิดขึ้นเลย (invariant ที่ถูกละเมิด) ส่วน error ที่ผู้ใช้ทำผิดเงื่อนไขได้
ตามปกติ (หารด้วยศูนย์, ใส่ค่าที่ไม่ถูกต้อง, network request ล้มเหลว) ต้องคืนเป็น `Result` ให้ผู้เรียกจัดการ

```rust
#[wasm_bindgen]
pub fn safe_divide(a: f64, b: f64) -> Result<f64, JsValue> {
    if b == 0.0 {
        return Err(JsValue::from_str("หารด้วยศูนย์ไม่ได้"));
    }
    Ok(a / b)
}
```

`#[wasm_bindgen]` แปลง `Result<T, JsValue>` ให้เป็นพฤติกรรมที่ JavaScript คุ้นเคยที่สุด: **`Ok(v)` คืนค่า
`v` ตรง ๆ ส่วน `Err(e)` จะถูก throw ออกไปเป็น JS exception** (คล้ายกับว่าฟังก์ชันนี้ "โยน" ค่า `e` ออกมา
ด้วย `throw`) ฝั่ง JavaScript จึงจับ error ด้วย `try/catch` ธรรมดาได้เลยโดยไม่ต้องรู้อะไรเกี่ยวกับ `Result`
ของ Rust เลย:

```javascript
const { safe_divide } = require("./pkg/demo.js");

console.log(safe_divide(10.0, 2.0)); // 5

try {
  safe_divide(10.0, 0.0);
} catch (e) {
  console.log("จับ error ได้:", e);
}
```

ผลลัพธ์จริง:

```
5
จับ error ได้: หารด้วยศูนย์ไม่ได้
```

เปรียบเทียบผลลัพธ์นี้กับหัวข้อ 87.9.1 ให้เห็นความต่างชัด ๆ: `Err(JsValue::from_str("..."))` ให้
**exception ที่มี message ตรงกับที่คุณตั้งใจเขียนไว้เป๊ะ** และ **WASM instance ยังทำงานต่อได้ปกติสมบูรณ์
หลังจากนั้น** (ไม่มีการ trap ด้วย `unreachable` เลย) ต่างจาก panic ที่ instance ทั้งตัวเข้าสู่สถานะที่ไม่
น่าเชื่อถืออีกต่อไป — นี่คือเหตุผลที่ **API ของ WASM module ที่ออกแบบมาอย่างดีควรคืน `Result<T, JsValue>`
สำหรับทุก operation ที่มีโอกาสล้มเหลวได้ตามปกติ** แล้วเก็บ `panic!` ไว้สำหรับกรณีที่เป็นบั๊กจริง ๆ เท่านั้น
(invariant ที่ควรเป็นไปไม่ได้เลยถ้าโค้ดถูกต้อง)

### 87.10 กับดักเรื่องหน่วยความจำและ lifetime ข้ามขอบเขต

หัวข้อนี้คือการนำหลักการของ Part 42 (raw pointer, memory layout) และ Part 43 (dangling pointer ข้าม FFI)
มาปะทะกับความจริงข้อใหม่ของ WASM: **WASM linear memory เป็นพื้นที่ความจำแยกจาก JS heap โดยสิ้นเชิง** —
JavaScript garbage collector ไม่รู้จักและไม่จัดการ object ที่อาศัยอยู่ใน WASM memory เลย การจัดการ lifetime
ของ object เหล่านี้เป็นความรับผิดชอบของ Rust code (ผ่าน `Drop` และ `Box`) ที่ `wasm-bindgen` ผูกไว้กับ
`.free()` method ที่มัน generate ให้อัตโนมัติ

#### 87.10.1 Use-after-free จริง: เรียก method บน object ที่ถูก `.free()` ไปแล้ว

ทุก struct ที่ export ด้วย `#[wasm_bindgen]` (แบบใน 87.4) จะได้ method `.free()` ติดมาให้อัตโนมัติในฝั่ง
JavaScript (เพื่อบอกฝั่ง Rust ว่า "เลิกใช้ object นี้แล้ว ปล่อยคืนความจำได้") — JavaScript garbage collector
**ไม่ได้เรียก `.free()` ให้อัตโนมัติเสมอไป** ต้องเรียกมือเองถ้าต้องการให้แน่ใจว่าความจำถูกปล่อยคืนตอนที่
ตั้งใจจริง ๆ

(หมายเหตุจากการตรวจ generated code จริงของ `wasm-bindgen` เวอร์ชัน 0.2.129 ที่ใช้ใน chapter นี้: มันมี
**`FinalizationRegistry` เป็น backstop ติดมาให้อัตโนมัติ** สำหรับทุก struct ที่ export — เปิดไฟล์ `.js`
ที่ generate มาจะเห็นโค้ดประมาณ `const CounterFinalization = new FinalizationRegistry(ptr =>
wasm.__wbg_counter_free(ptr, 1));` ที่ผูกไว้กับทุก instance ที่สร้างขึ้นตอน `new Counter()` — นี่หมายความ
ว่า **ถ้าคุณลืมเรียก `.free()` เอง ในที่สุด JS garbage collector ก็จะ finalize แล้วเรียก free ให้จริง ๆ**
แต่ `FinalizationRegistry` ตาม spec ของ JavaScript **ไม่การันตีว่าจะถูกเรียกเมื่อไหร่เลยแม้แต่นิดเดียว**
(อาจจะไม่ถูกเรียกเป็นเวลานานมาก หรือไม่ถูกเรียกเลยถ้าโปรแกรมจบไปก่อน) หน่วยความจำ WASM จึงอาจคงค้างอยู่นาน
กว่าที่ควรจะเป็นถ้าพึ่งพา mechanism นี้อย่างเดียว — สรุปคือ **`FinalizationRegistry` ช่วยกัน "ลืมสนิท" แบบ
leak ตลอดไป แต่ไม่ใช่ตัวช่วยสำหรับ deterministic cleanup** ถ้าต้องการปล่อยคืนความจำ ณ จุดที่รู้แน่ชัดว่า
ไม่ใช้แล้ว (เช่น component ถูกทำลายจริง ๆ ในหน้าเว็บ) ยังต้องเรียก `.free()` มือเองเสมอ)

ถ้าเรียก `.free()` ไปแล้วแต่ยังถือ reference ของ object นั้นไว้แล้วเรียก method ต่อ:

```javascript
const { Counter } = require("./pkg/demo.js");

const c = new Counter();
c.increment();
console.log("ก่อน free:", c.value);

c.free(); // บอกฝั่ง Rust ว่าปล่อยคืนความจำของ Counter ตัวนี้ได้แล้ว

console.log(c.value); // เรียกใช้ handle ที่ถูก free ไปแล้ว — เกิดอะไรขึ้น?
```

ผลลัพธ์จริงจากการรัน (capture มาแบบคำต่อคำ ไม่มีการแต่งเติม):

```
ก่อน free: 1
EXCEPTION message: Attempt to use a moved value
```

**exception ที่เกิดขึ้นจริงคือ `Error: Attempt to use a moved value`** ไม่ใช่ error เกี่ยวกับ null pointer
อย่างที่อาจเดาไว้ — คำว่า "moved value" ในที่นี้สื่อถึงแนวคิดเดียวกับ **ownership move ของ Rust ที่ Part 6
สอนไว้**: การเรียก `.free()` คือการบอกว่า "ฉันส่งมอบ (move) การเป็นเจ้าของ object นี้ไปให้กระบวนการทำลาย
มันแล้ว" — หลังจากนั้น handle ที่ JS ยังถืออยู่คือ**การอ้างอิงถึงสิ่งที่ไม่ใช่เจ้าของอีกต่อไป** ตรงกับ error
ที่ borrow checker ของ Rust เองจะฟ้องถ้าคุณพยายามใช้ค่าที่ถูก `move` ไปแล้วในโค้ด Rust ปกติ (`error[E0382]:
use of moved value`) — `wasm-bindgen` เลียนแบบ error message รูปแบบเดียวกันมาไว้ที่ฝั่ง JavaScript
โดยเฉพาะเพื่อให้แนวคิดสอดคล้องกัน แม้ JavaScript เองไม่มี concept เรื่อง "moved value" อยู่ในภาษาเลยก็ตาม

ลองเรียก method อื่นบน handle เดิมต่อ (`c.increment()`) ก็ได้ error เดียวกันเป๊ะ (`Attempt to use a moved
value`) — และถ้าเรียก `.free()` ซ้ำสองครั้ง (double free) จะได้ error ที่ต่างออกไปอีกแบบ:

```
double free EXCEPTION message: null pointer passed to rust
```

**นี่คือ safety net ที่ `wasm-bindgen` เพิ่มให้เอง** ไม่ว่าจะเป็นกรณี use-after-free หรือ double-free —
มันไม่ปล่อยให้เกิด undefined behavior แบบเงียบ ๆ เหมือนที่ raw pointer dangling ทำใน Part 43 แต่จะ**fail
แบบชัดเจนและ throw ทันที** แทน โดยตรวจสอบผ่าน pointer ภายในของ JS wrapper object ก่อนเรียกเข้าไปใน WASM
ทุกครั้ง (เซ็ตเป็นค่าพิเศษทันทีที่ `.free()` ถูกเรียก แล้วเช็คค่านี้ก่อน generate การเรียกจริงเข้าไปในฝั่ง
Rust เสมอ)

**เหตุผลเชิงลึกที่ต้องเข้าใจ**: ต่างจาก `CString`/raw pointer dangling ใน Part 43 ที่ compiler ไม่มีทางรู้
เลยว่า pointer หมดอายุแล้ว (borrow checker ไม่ตรวจสอบ raw pointer) กรณีนี้ `wasm-bindgen` **สร้าง guard
ของตัวเองไว้ที่ระดับ JS glue code** เพื่อจับกรณีนี้โดยเฉพาะ เพราะมันรู้ดีว่า "JavaScript ไม่มี borrow
checker หรือ ownership system ใด ๆ เลย — โปรแกรมเมอร์ฝั่ง JS มีโอกาสสูงมากที่จะถือ reference ค้างไว้เกิน
อายุของ object จริง (เพราะ JS ไม่เคยต้องคิดเรื่องนี้มาก่อนในโลกของตัวเอง ทุก object ถูกดูแลโดย garbage
collector หมด)" — การ throw error ที่ชัดเจนคือทางเลือกที่ปลอดภัยที่สุดเท่าที่ทำได้ในสถานการณ์ที่ compiler
มองไม่เห็นปัญหานี้เลยทั้งสองฝั่ง

#### 87.10.2 `mem::forget` และอันตรายของการ "ลืม" ที่ตั้งใจ

ทวนจาก Part 15 (Drop trait) / Part 41: `std::mem::forget(value)` คือฟังก์ชันที่ทำให้ Rust **ไม่เรียก
destructor ของ `value` เลย** โดยเจตนา — มันไม่ได้ปล่อยคืนความจำ แต่บอก compiler ว่า "ห้ามรัน `Drop::drop`
สำหรับตัวนี้" เท่านั้น (ตัวความจำเองยังไม่ได้ปล่อยคืน ถ้าไม่มีใครปล่อยคืนอีก มันจะรั่วตลอดไปจนกว่าโปรแกรมจบ)

ในบริบทของ WASM/`wasm-bindgen` มีการใช้ pattern ที่**หน้าตาคล้าย `mem::forget` แต่จำเป็นจริง**: การเรียก
`Closure::forget()` (จาก `wasm_bindgen::closure::Closure`) ตอนสร้าง event listener ที่ต้องมีชีวิตอยู่
ตลอดไปตราบใดที่หน้าเว็บยังเปิดอยู่ (ดูตัวอย่างเต็มในหัวข้อ capstone 87.11) — `Closure::forget()` เรียก
`mem::forget` ข้างในตัวมันเองจริง ๆ เพื่อป้องกันไม่ให้ Rust ปล่อยคืน closure ตอนที่ตัวแปร Rust ที่ถือมันไว้
หลุดจาก scope ทั้งที่ยังมี JavaScript ถือ reference ไปเรียกมันอยู่ (DOM event listener ที่ผูกไว้กับปุ่ม)

**อันตรายจริงถ้าใช้ผิด**: ถ้าเผลอ `.forget()` closure ที่ควรจะถูกลบทิ้งตอน component/หน้าถูกทำลาย (เช่น
ใน single-page application ที่สร้าง/ทำลาย component ซ้ำ ๆ) closure เก่าที่ "ถูกลืมไว้" จะยังคงค้างอยู่ใน
หน่วยความจำตลอดไปแม้ปุ่มที่มันผูกอยู่ถูกลบออกจาก DOM ไปแล้ว — นี่คือ **memory leak ที่สะสมทีละนิดทุกครั้ง
ที่สร้าง component ใหม่** และเป็นบั๊กคลาสสิกของแอป WASM แบบ single-page ที่ไม่ได้จัดการ lifecycle ของ
closure ให้ถูกต้อง (แทนที่จะ `.forget()` ทุกครั้งแบบไม่คิด ควรเก็บ handle ของ `Closure` ไว้และเรียก
`remove_event_listener` คู่กับการ drop closure จริง ๆ เมื่อ component ถูกทำลาย)

#### 87.10.3 สรุปกฎที่ควรจำสำหรับหัวข้อนี้

| สถานการณ์ | อันตราย | วิธีป้องกัน |
|---|---|---|
| เรียก method บน object ที่ `.free()` ไปแล้ว | ได้ exception ทันที (ปลอดภัยกว่า native UB แต่โปรแกรมพัง) | ตรวจสอบ lifecycle ให้ชัดเจนว่าใครเป็นเจ้าของ handle และเมื่อไหร่ควร `.free()` |
| ลืมเรียก `.free()` เลย | memory leak สะสมใน WASM linear memory (ไม่มี GC มาช่วยเก็บให้เสมอไป) | เรียก `.free()` ให้ตรงกับ lifecycle ของ component เสมอ หรือพึ่ง `FinalizationRegistry` ของเวอร์ชันใหม่แต่ต้องรู้ว่าเวลาไม่แน่นอน |
| `Closure::forget()` แบบไม่จำเป็น | leak closure ทุกครั้งที่สร้างใหม่ | forget เฉพาะ closure ที่ตั้งใจให้อยู่ตลอดชีวิตของหน้าเว็บจริง ๆ เท่านั้น ที่เหลือเก็บ handle ไว้ drop เอง |
| ส่ง `&str`/`String` ปริมาณมากถี่ ๆ ข้าม boundary | ต้นทุน copy สะสม (87.3) | ลดจำนวนครั้งที่ข้าม boundary, batch การส่งข้อมูล |

### 87.11 Capstone: To-do List แบบ interactive เต็มรูปแบบ

มาประกอบทุกอย่างที่เรียนมาในบทนี้เข้าด้วยกันเป็นแอปเล็ก ๆ ที่ใช้งานได้จริงในเบราว์เซอร์: **to-do list**
ที่ state ทั้งหมดอยู่ฝั่ง Rust, DOM manipulation ผ่าน `web-sys`, และปุ่มกดเชื่อมกลับเข้าไปเรียก Rust ผ่าน
event listener

#### 87.11.1 struct `TodoList`: state และ logic ทั้งหมดอยู่ฝั่ง Rust

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub struct TodoList {
    items: Vec<String>,
}

#[wasm_bindgen]
impl TodoList {
    #[wasm_bindgen(constructor)]
    pub fn new() -> TodoList {
        TodoList { items: Vec::new() }
    }

    pub fn add(&mut self, text: String) {
        self.items.push(text);
    }

    pub fn remove(&mut self, index: usize) -> bool {
        if index < self.items.len() {
            self.items.remove(index);
            true
        } else {
            false
        }
    }

    pub fn len(&self) -> usize {
        self.items.len()
    }

    // render() คือจุดที่ web-sys เข้ามาทำงาน: ล้าง container เดิม แล้วสร้าง <li> ใหม่
    // ให้ตรงกับ state ปัจจุบันของ Vec<String> ทั้งหมด
    pub fn render(&self, container_id: &str) -> Result<(), JsValue> {
        let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
        let document = window
            .document()
            .ok_or_else(|| JsValue::from_str("no document"))?;
        let container = document
            .get_element_by_id(container_id)
            .ok_or_else(|| JsValue::from_str("container not found"))?;

        container.set_inner_html("");
        for (i, item) in self.items.iter().enumerate() {
            let li = document.create_element("li")?;
            li.set_text_content(Some(&format!("{}. {}", i + 1, item)));
            container.append_child(&li)?;
        }
        Ok(())
    }
}
```

`TodoList` ตัวนี้คือ opaque handle แบบเดียวกับ `Counter`/`BookingCart` ในหัวข้อ 87.4 — state
(`items: Vec<String>`) อาศัยอยู่ใน WASM linear memory ตลอดเวลา, `render()` คือ method ที่อ่าน state
นั้นแล้วสร้าง/อัปเดต DOM element จริงผ่าน `web-sys` (`create_element`, `set_text_content`,
`append_child` — ล้วนเป็น method ของ `web_sys::Document`/`web_sys::Element` ตรงกับหัวข้อ 87.5.4)

#### 87.11.2 ผูกปุ่มและ input เข้ากับ `TodoList` ผ่าน `Closure`

ส่วนที่ซับซ้อนที่สุดของ capstone นี้คือการทำให้**ปุ่มกดจริงในหน้าเว็บ**เรียกกลับเข้าไปแก้ไข `TodoList`
ได้ — ปัญหาคือ event listener ของ JavaScript ต้องมี callback ที่ **มีชีวิตอยู่ตราบเท่าที่ event ยังผูก
อยู่กับ element** (อาจนานเท่ากับที่หน้าเว็บเปิดอยู่) แต่ `TodoList` ที่สร้างขึ้นในฟังก์ชัน setup ปกติจะถูก
drop ทันทีที่ฟังก์ชันนั้น return — ต้องใช้ `Rc<RefCell<TodoList>>` (ทวนจาก Part 20-21: shared mutable
ownership) เพื่อให้ closure "share" การเข้าถึง `TodoList` ได้อย่างปลอดภัยตาม borrow rule ของ Rust
(ตรวจสอบตอน runtime ผ่าน `RefCell` เพราะ closure อาจถูกเรียกได้หลายครั้งไม่รู้ล่วงหน้า จึงใช้ static
borrow checking ตอน compile ไม่ได้):

```rust
use std::cell::RefCell;
use std::rc::Rc;

use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{Event, HtmlInputElement};

#[wasm_bindgen]
pub fn setup_todo_app(input_id: &str, button_id: &str, list_id: &str) -> Result<(), JsValue> {
    let window = web_sys::window().ok_or_else(|| JsValue::from_str("no window"))?;
    let document = window.document().ok_or_else(|| JsValue::from_str("no document"))?;

    let button = document
        .get_element_by_id(button_id)
        .ok_or_else(|| JsValue::from_str("button not found"))?;

    let input_id = input_id.to_string();
    let list_id = list_id.to_string();
    let document_for_closure = document.clone();

    // Rc<RefCell<TodoList>>: หลาย closure (ในแอปจริงอาจมีปุ่มลบ, ปุ่มแก้ไข ฯลฯ) share
    // การเข้าถึง TodoList ตัวเดียวกันได้ผ่าน Rc (reference counting) และแก้ไขได้ผ่าน
    // RefCell (borrow checking แบบ runtime) — ตรงกับที่ Part 20-21 สอนไว้ทุกประการ
    let state = Rc::new(RefCell::new(TodoList::new()));

    let closure = Closure::wrap(Box::new(move |_event: Event| {
        let input = document_for_closure
            .get_element_by_id(&input_id)
            .and_then(|el| el.dyn_into::<HtmlInputElement>().ok());

        if let Some(input) = input {
            let text = input.value();
            if !text.trim().is_empty() {
                state.borrow_mut().add(text);
                input.set_value("");
                let _ = state.borrow().render(&list_id);
            }
        }
    }) as Box<dyn FnMut(Event)>);

    button.add_event_listener_with_callback("click", closure.as_ref().unchecked_ref())?;

    // ปล่อย closure นี้ไว้ตลอดชีวิตของหน้าเว็บโดยตั้งใจ (ดูหัวข้อ 87.10.2) — ปุ่มนี้ต้อง
    // เรียก callback ได้ตลอดไปจนกว่าหน้าเว็บจะถูกปิดหรือ reload ทั้งหน้า ซึ่งจะปล่อยคืน
    // WASM memory ทั้งหมดให้เองโดยธรรมชาติอยู่แล้ว
    closure.forget();

    Ok(())
}
```

HTML ที่ใช้คู่กัน (ไฟล์ `index.html` อย่างง่ายสำหรับรันใน browser จริง):

```html
<!DOCTYPE html>
<html lang="th">
  <head>
    <meta charset="UTF-8" />
    <title>To-do List ด้วย Rust + wasm-bindgen</title>
  </head>
  <body>
    <input id="todo-input" type="text" placeholder="พิมพ์งานที่ต้องทำ..." />
    <button id="todo-button">เพิ่มรายการ</button>
    <ul id="todo-list"></ul>

    <script type="module">
      import init, { setup_todo_app } from "./pkg/demo.js";

      async function main() {
        await init();
        setup_todo_app("todo-input", "todo-button", "todo-list");
      }

      main();
    </script>
  </body>
</html>
```

**ผลลัพธ์จริงจากการ verify** (จำลองการกดปุ่มด้วย `dispatchEvent(new Event("click"))` ผ่าน headless DOM
แทนการคลิกด้วยเมาส์จริง แต่ event listener ที่ผูกไว้ทำงานเหมือนกันทุกประการ):

```
DOM ul innerHTML: <li>1. ซื้อนม</li><li>2. อ่านหนังสือ Rust</li><li>3. ออกกำลังกาย</li>
after remove(1), len: 2
DOM ul innerHTML: <li>1. ซื้อนม</li><li>2. ออกกำลังกาย</li>

-- หลังผูก setup_todo_app แล้วพิมพ์ข้อความลง input พร้อม dispatch เหตุการณ์ "click" ที่ปุ่มจริง --
DOM ul innerHTML after click: <li>1. งานใหม่จากปุ่มกด</li>
input value after click (should be cleared): ""
```

ยืนยันว่า flow ทั้งหมดทำงานถูกต้องครบวงจร: การพิมพ์ข้อความลง `<input>` จริง → คลิกปุ่มจริง → event listener
ที่ผูกไว้จาก Rust ทำงาน → อ่านค่าจาก `HtmlInputElement::value()` → เรียก `TodoList::add()` เข้าไปแก้ไข
state ในฝั่ง Rust → เรียก `render()` วาด `<li>` ใหม่ → เคลียร์ค่า input กลับเป็นค่าว่าง ครบทุกขั้นตอนตามที่
ออกแบบไว้

**สังเกตภาพรวมทั้งหมดของ capstone นี้**: `TodoList` (opaque handle จากหัวข้อ 87.4) ถือ state, `render()`
ใช้ `web-sys` (หัวข้อ 87.5.4) จัดการ DOM ให้ตรงกับ state, `Closure` + `Rc<RefCell<...>>` ผูก event ของ
ปุ่มจริงเข้ากับ logic ของ Rust, และทุกจุดที่อาจล้มเหลว (`window()`, `document()`, `get_element_by_id()`)
ใช้ `Result<(), JsValue>` (หัวข้อ 87.9.3) แทนการ `.unwrap()` มือ ๆ — นี่คือรูปแบบที่โปรเจกต์ WASM ระดับ
production ใช้จริง แม้จะยังไม่ใช่ framework เต็มรูปแบบแบบ Yew หรือ Leptos ที่จะเรียนใน Part 88-89 ก็ตาม
(สิ่งที่ framework เหล่านั้นทำเพิ่มคือจัดการ pattern แบบนี้ให้อัตโนมัติผ่าน virtual DOM / reactive
signal แทนที่จะต้องเขียน `web-sys` มือทุกจุดแบบในบทนี้)

### 87.12 ทดสอบโค้ดที่ export ด้วย `wasm-bindgen`: crate `wasm-bindgen-test`

โค้ดทั้งหมดที่ผ่านมาในบทนี้ verify ด้วยการเขียน JavaScript ทดสอบแยกแล้วรันผ่าน Node.js ตรง ๆ ซึ่งใช้ได้ดีสำหรับ
เรียนรู้และ debug แต่ในโปรเจกต์จริงที่มี CI/CD ต้องมีวิธีเขียน **เทสต์ที่รันเป็นส่วนหนึ่งของ `cargo test`**
ได้ — crate [`wasm-bindgen-test`](https://crates.io/crates/wasm-bindgen-test) ทำหน้าที่นี้: มัน generate
`#[test]` แบบพิเศษที่ compile เป็น WASM แล้วรันผ่าน runner ที่เหมาะกับ environment ที่เลือก

```toml
[dev-dependencies]
wasm-bindgen-test = "0.3"
```

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use wasm_bindgen_test::*;

    #[wasm_bindgen_test]
    fn test_add() {
        assert_eq!(add(2, 3), 5);
    }

    #[wasm_bindgen_test]
    fn test_greet() {
        assert_eq!(greet("Bob"), "สวัสดี, Bob! ยินดีต้อนรับสู่ WASM");
    }

    #[wasm_bindgen_test]
    fn test_safe_divide_error() {
        assert!(safe_divide(1.0, 0.0).is_err());
    }
}
```

รันด้วย `wasm-pack test --node` (สำหรับเทสต์ที่ไม่แตะ DOM/Web API เลย รันบน Node.js ได้ตรง ๆ โดยไม่ต้องมี
เบราว์เซอร์) — ผลลัพธ์จริงจากการรัน:

```
running 3 tests
test tests::test_safe_divide_error ... ok
test tests::test_greet ... ok
test tests::test_add ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 filtered out; finished in 0.01s
```

สำหรับเทสต์ที่ต้องแตะ DOM/Web API จริง (เช่นเทสต์ `update_greeting` หรือ `TodoList::render` ในหัวข้อ 87.11)
ต้องเพิ่มบรรทัด `wasm_bindgen_test_configure!(run_in_browser);` ไว้ต้นไฟล์ test แล้วรันด้วย `wasm-pack test
--headless --chrome` (หรือ `--firefox`) ซึ่งต้องมีเบราว์เซอร์ headless ติดตั้งอยู่ในเครื่อง/CI runner ที่รัน
เทสต์ — environment ที่ใช้เขียนบทนี้ไม่มีเบราว์เซอร์ headless ติดตั้งไว้ (ไม่มี Chrome/Firefox ในระบบ) จึง
verify ได้เฉพาะเส้นทาง `--node` ข้างบนเท่านั้น ส่วนตัวอย่างที่ต้องแตะ DOM จริงในบทนี้ (87.5.4, 87.11)
ถูก verify ด้วยแนวทางอื่นแทน: รันผ่าน Node.js ตรง ๆ พร้อมจำลอง DOM ด้วยไลบรารี `jsdom` (ซึ่งให้
`document`/`window`/`Element` ที่ทำงานได้จริงในระดับที่พอสำหรับทดสอบ `web-sys` โดยไม่ต้องเปิดเบราว์เซอร์
จริง) — เป็นแนวทางที่ใช้ได้จริงเมื่อไม่มี headless browser ให้ใช้ แต่ **`wasm-pack test --headless --chrome`
ยังคงเป็นแนวทางมาตรฐานที่แนะนำสำหรับโปรเจกต์จริง** เพราะทดสอบกับ JS engine ตัวจริงของเบราว์เซอร์ที่ผู้ใช้
จะเจอ ไม่ใช่ implementation จำลองแบบ `jsdom`

ข้อควรระวังเล็ก ๆ ที่เจอจริงตอนตั้งค่าเทสต์: ถ้าใช้ `#[wasm_bindgen(start)]` (หัวข้อ 87.9.2) แล้วตั้งชื่อ
ฟังก์ชันว่า `main` ตรง ๆ จะชนกับฟังก์ชัน `main` ที่ `wasm-bindgen-test` generate ขึ้นมาเองสำหรับรัน test
harness ทำให้ `wasm-pack test` ล้มเหลวด้วย error จริง:

```
Error: executing `wasm-bindgen` over the Wasm file

Caused by:
    the name `main` is exported by multiple crates in this build; rename one side with `js_name`/`js_namespace`
```

วิธีแก้คือตั้งชื่อฟังก์ชันที่ประดับด้วย `#[wasm_bindgen(start)]` เป็นชื่ออื่นที่ไม่ใช่ `main` (เช่น
`init_panic_hook`) — ชื่อฟังก์ชันนี้ไม่มีผลอะไรกับพฤติกรรมเลย เพราะ `start` attribute เป็นตัวกำหนดว่าจะถูก
เรียกอัตโนมัติตอนโหลด ไม่ใช่ชื่อฟังก์ชันที่มีความหมายพิเศษ

### 87.13 ตารางสรุป: การแมป type ระหว่าง Rust กับ JavaScript

ปิดท้ายเนื้อหาด้วยตารางสรุปการแมป type ทั้งหมดที่เจอในบทนี้ (ในความหมายเดียวกับตารางการแมป type C ↔ Rust
ของ Part 43 ข้อ 43.4 — แต่ปลายทางเป็น JavaScript) เพื่อใช้เป็น reference เร็ว ๆ ตอนเขียนโค้ดจริง:

| Rust type | JavaScript ที่เห็น | หมายเหตุ (verify จากบทนี้) |
|---|---|---|
| `i32`, `f64`, `bool` | `number` / `boolean` | ข้ามขอบเขตตรง ๆ แบบไม่มีต้นทุน copy (87.3.2) |
| `u64`, `i64` | `bigint` | JS `number` เป็น `f64` ที่แทนจำนวนเต็มได้แม่นยำแค่ถึง 2^53 — `wasm-bindgen` จึงแมป 64-bit integer เป็น `BigInt` เสมอเพื่อไม่เสีย precision (verify จริง: `big_number(9007199254740993n)` คืน `9007199254740994n` ถูกต้องเป๊ะ ถ้าแมปเป็น `number` ธรรมดาจะเพี้ยนทันที) |
| `String`, `&str` | `string` | copy เป็น UTF-8 bytes ทั้งสองทาง มีต้นทุนจริง (87.3.3) |
| `Vec<T>` (T เป็นตัวเลข primitive) | `Array`/`TypedArray` | แปลงให้อัตโนมัติโดย `wasm-bindgen` เอง ไม่ต้องพึ่ง `js-sys` |
| `struct`/`enum` ที่มี `#[wasm_bindgen]` | class ที่มี `.free()` (opaque handle) | JS ถือแค่ pointer ไม่เห็น field ข้างใน (87.4) |
| struct ที่ derive `Serialize`/`Deserialize` ผ่าน `serde-wasm-bindgen` | plain object (`{ field: value, ... }`) | ข้อมูลถูก copy ทั้งหมดออกมาเป็น JS object ใหม่ (87.8) |
| `Option<T>` | `T` หรือ `undefined`/`null` | ตรงกับความหมายของ "อาจไม่มีค่า" ทั้งสองภาษา |
| `Result<T, JsValue>` | คืนค่า `T` ปกติ หรือ `throw` (87.9.3) | `Ok` คืนตรง ๆ, `Err` ถูก throw ให้ `catch` จับ |
| `panic!` | `throw new Error()` แบบ `RuntimeError: unreachable` เสมอ (87.9.1) | ข้อความ panic จริงหายไป ไม่ปรากฏใน exception |
| `async fn` ที่มี `#[wasm_bindgen]` | ฟังก์ชันที่คืนค่าเป็น `Promise` (87.7) | ต้อง `await`/`.then()` เสมอฝั่ง JS |
| `js_sys::Array`/`js_sys::Object` | `Array`/`Object` ตัวจริงของ JS | ใช้เมื่อต้องรับ/สร้าง JS value แบบดิบที่ type ไม่คงที่ (87.6) |
| `web_sys::*` (เช่น `Window`, `Document`) | Web API object ตัวจริงของเบราว์เซอร์ | เป็น typed wrapper รอบ `JsValue` ที่ compile-time ตรวจสอบชื่อ method ได้ (87.5.4) |

### 87.14 สรุปคำสั่งที่ใช้บ่อยตลอดบทนี้ (Quick Reference)

ก่อนไปหัวข้อกับดักและแบบฝึกหัด สรุปลำดับคำสั่งทั้งหมดที่ใช้ตั้งโปรเจกต์ตั้งแต่ศูนย์จนถึงรันได้จริง (รวบรวม
จากทุกหัวข้อในบทนี้ไว้ในที่เดียว เผื่อกลับมาเปิดใช้ตอนเริ่มโปรเจกต์จริง):

```bash
# 1. สร้างโปรเจกต์และตั้งค่า crate-type ให้เป็น cdylib (หัวข้อ 87.3.1)
cargo new --lib my-wasm-project
cd my-wasm-project

# 2. เพิ่ม dependency หลัก
cargo add wasm-bindgen
cargo add js-sys wasm-bindgen-futures serde-wasm-bindgen
cargo add serde --features derive
cargo add console_error_panic_hook   # สำหรับ debug panic message (87.9.2)

# 3. web-sys ต้องเปิด feature เฉพาะที่ใช้ (87.5.4) — แก้ Cargo.toml เพิ่มด้วยตัวเอง
#    [dependencies.web-sys]
#    version = "0.3"
#    features = ["console", "Window", "Document", "Element", "HtmlElement"]

# 4. ติดตั้ง wasm-pack (ครั้งเดียว)
npm install -g wasm-pack   # หรือ cargo install wasm-pack

# 5. build ให้ตรงกับที่ที่จะใช้งาน (87.3.4)
wasm-pack build --target web       # ใช้กับ <script type="module"> ตรง ๆ
wasm-pack build --target bundler   # ใช้กับ webpack/Vite
wasm-pack build --target nodejs    # ใช้กับ Node.js/testing script

# 6. build จริงก่อน deploy ต้องใช้ --release เสมอ (87.3.5) ไม่ใช่ --dev
wasm-pack build --target web --release

# 7. เขียนเทสต์และรัน (87.12)
wasm-pack test --node                       # เทสต์ที่ไม่แตะ DOM
wasm-pack test --headless --chrome          # เทสต์ที่แตะ DOM/Web API (ต้องมี headless Chrome)
```

### 87.15 ตาราง attribute ของ `#[wasm_bindgen(...)]` ที่ใช้ในบทนี้

| Attribute | ใช้ที่ไหน | ความหมาย | หัวข้อ |
|---|---|---|---|
| (ไม่มี attribute เพิ่ม) | function, struct, impl | export ตรง ๆ ให้ JS เรียกได้ | 87.3, 87.4 |
| `constructor` | method ใน `impl` block | บอกว่า `new ClassName()` ในฝั่ง JS เรียก method นี้ | 87.4.1 |
| `getter` / `setter` | method ใน `impl` block | ทำให้ JS เข้าถึงเป็น property (`obj.value`) แทนเรียกเป็น method | 87.4.1 |
| `start` | function เดียวในทั้ง crate | รันอัตโนมัติทันทีที่ WASM module โหลดเสร็จ | 87.9.2 |
| `js_name` | ใน `extern "C"` block หรือ function/method ที่ export | ตั้งชื่อฝั่ง JS ให้ต่างจากชื่อฝั่ง Rust | 87.5.1 |
| `js_namespace` | ใน `extern "C"` block | บอกว่าฟังก์ชันอยู่ใต้ namespace ของ object อื่น (เช่น `Math`, `JSON`) | 87.5.1, 87.5.3 |
| `module` | บน `extern "C"` block | import ฟังก์ชันจากไฟล์ JS ของเราเอง แทนที่จะเป็น global/built-in | 87.5.2 |
| `catch` | บนฟังก์ชันใน `extern "C"` block | แปลง exception ของ JS ที่ import มาให้เป็น `Result::Err` แทนการปล่อยให้ panic | 87.5.3 |

## กับดักที่พบบ่อย (Common Pitfalls)

### กับดักที่ 1: ลืมเปิด Cargo feature ของ `web-sys` แล้ว compile error ว่า "no method"

โค้ดที่เรียก `web_sys::window().document()` แต่ลืมเปิด feature `"Document"` ใน `Cargo.toml`:

```toml
[dependencies.web-sys]
version = "0.3"
features = ["console"]  # ลืมเปิด "Window", "Document"
```

Error จริงตอน compile:

```
error[E0599]: no method named `document` found for struct `web_sys::Window` in the current scope
  --> src/lib.rs:10:30
   |
10 |     let document = window.document()...
   |                           ^^^^^^^^^ method not found in `Window`
   |
note: the method `document` exists but the following trait bounds were not satisfied:
      the feature `Document` is not enabled
```

(ข้อความ error จริงของ `web-sys` แต่ละเวอร์ชันอาจต่างกันเล็กน้อย แต่แกนหลักคือ **method ที่มีอยู่จริงใน
Web API จะ "หายไป" จาก Rust type จนกว่าจะเปิด feature ตรงกับชื่อ type/method นั้น** เพราะ `web-sys`
generate code เฉพาะสำหรับ feature ที่เปิดไว้เท่านั้น ไม่ generate ทุกอย่างมาให้ตั้งแต่แรกเพื่อไม่ให้ compile
time และ binary size บวมเกินจำเป็น) วิธีแก้คือเปิด feature ที่ขาดไปให้ครบ: `"Window"`, `"Document"`

### กับดักที่ 2: ลืม `wasm-pack build` แล้วพยายามรัน `.wasm` ตรง ๆ จาก JavaScript

หลาย ๆ คนเข้าใจผิดว่า `.wasm` file ที่ compile ได้จาก `cargo build --target wasm32-unknown-unknown` ใช้
`WebAssembly.instantiate` โหลดตรง ๆ จาก JavaScript ได้เหมือนตัวอย่างเปล่า ๆ จาก Part 86 — แต่ไฟล์ที่ compile
จากโปรเจกต์ที่ใช้ `#[wasm_bindgen]` **ต้องผ่าน `wasm-pack build` (หรือ `wasm-bindgen-cli` ตรง ๆ) ก่อน** เพื่อ
generate JS glue code ที่จำเป็น ถ้าลอง `WebAssembly.instantiateStreaming(fetch("demo.wasm"))` ตรง ๆ กับ
ไฟล์ `.wasm` ดิบที่ยังไม่ผ่าน `wasm-bindgen-cli`:

```
TypeError: WebAssembly.instantiate(): Import #0 module="__wbindgen_placeholder__" error: module is not an object or function
```

(error message จริงขึ้นอยู่กับว่าโค้ดใช้ feature ของ `wasm-bindgen` มากแค่ไหน แต่โดยรวมคือ **ไฟล์
`.wasm` ดิบที่ยังไม่ผ่าน `wasm-bindgen-cli` จะมี import ที่ชื่อขึ้นต้นด้วย `__wbindgen_*` ค้างอยู่**
เพราะ macro generate โค้ดที่คาดหวังว่าจะมี "environment" พิเศษที่ glue code เตรียมไว้ให้ — โหลดตรง ๆ
โดยไม่มี glue code จึงหา import เหล่านี้ไม่เจอ) วิธีแก้คือรัน `wasm-pack build --target web` (หรือ target
ที่เหมาะกับ environment ที่จะใช้) แล้วโหลดผ่านไฟล์ `.js` ที่ generate มาให้เสมอ ไม่ใช่โหลด `.wasm` ตรง ๆ

### กับดักที่ 3: ส่ง `&str`/`String` เป็น parameter ของ struct method โดยไม่รู้ต้นทุน จนแอป lag

โค้ดที่เรียก method ที่รับ `String` ซ้ำ ๆ ในลูปที่ทำงานทุก frame ของ `requestAnimationFrame` (เช่น อัปเดต
label ของ FPS counter ทุก frame ด้วย string ที่ format ใหม่ทุกครั้ง) — แต่ละครั้งที่เรียกมีต้นทุนจากการ
encode/decode UTF-8 และ allocate/deallocate หน่วยความจำ WASM ตามที่อธิบายไว้ในหัวข้อ 87.3.3 ถ้าเรียกที่
60 ครั้งต่อวินาที ต้นทุนนี้สะสมจนกลายเป็นคอขวดจริงที่วัดได้ด้วย browser profiler วิธีแก้ที่ใช้กันจริงใน
โปรเจกต์ที่ performance-critical คือ:

- ส่งค่าเป็นตัวเลข (`f64`/`i32`) แทน string ที่ format ไว้แล้ว แล้วให้ฝั่ง JavaScript format string เอง
  (JS format string เร็วกว่าการข้าม WASM boundary เสมอสำหรับ operation ง่าย ๆ แบบนี้)
- หรือลด**ความถี่**ของการเรียก (เช่น อัปเดต FPS counter ทุก 10 frame ไม่ใช่ทุก frame)

### กับดักที่ 4: เรียก `.await` บน `async fn` ที่ export ด้วย `#[wasm_bindgen]` แล้วได้ `[object Promise]` แทนค่าจริง

โค้ด JavaScript ที่ลืม `await` ตอนเรียกฟังก์ชัน async ที่ export มาจาก Rust:

```javascript
const result = fetch_text("http://127.0.0.1:PORT/hello"); // ลืม await
console.log(result);
```

ผลลัพธ์จริง:

```
Promise { <pending> }
```

นี่ไม่ใช่ error แต่เป็นพฤติกรรมที่ถูกต้องตามหัวข้อ 87.7: `async fn` ที่ export ผ่าน `#[wasm_bindgen]`
**คืนค่าเป็น `Promise` เสมอ ไม่ว่าจะเรียกจากที่ไหน** — เหมือนฟังก์ชัน `async` ธรรมดาของ JavaScript
ทุกประการ ต้อง `await` (หรือ `.then()`) เสมอเพื่อได้ค่าจริงออกมา คนที่คุ้นกับการเขียน sync function ใน
`wasm-bindgen` มาก่อนมักลืมจุดนี้พอเปลี่ยนมาเขียน `async fn` ครั้งแรก

### กับดักที่ 5: ใช้ `tokio::time::sleep` ในโค้ดที่ตั้งใจ compile เป็น WASM

โค้ดที่ copy pattern จาก backend (Part 48-50) มาใช้ตรง ๆ โดยไม่คิดว่ากำลัง compile เป็น WASM:

```rust
#[wasm_bindgen]
pub async fn wait_and_greet(name: String) -> String {
    tokio::time::sleep(std::time::Duration::from_secs(1)).await; // ผิด! ไม่มี Tokio runtime
    format!("สวัสดี, {}", name)
}
```

จุดที่น่าประหลาดใจ (verify จริงแล้วด้วยการ compile และรัน): โค้ดนี้ **`cargo build --target
wasm32-unknown-unknown` ผ่านได้ปกติ ไม่มี compile error เลย** (เพราะ module `tokio::time` เองไม่ได้ผูก
กับ `wasm32` target โดยตรง มันยัง compile ผ่านได้) ปัญหาจริงเกิด**ตอน runtime**เมื่อเรียกฟังก์ชันนี้จาก
JavaScript แล้วโค้ดพยายามอ่านเวลาปัจจุบันข้างใน `tokio::time::sleep` (ผ่าน `std::time::Instant::now()`)
— `wasm32-unknown-unknown` เป็น target ที่ **ไม่มี syscall สำหรับอ่านเวลาของระบบให้เรียกเลย** (ไม่รู้จัก
`clock_gettime` หรือเทียบเท่า เพราะไม่ได้ผูกกับ OS ใดเป็นพิเศษ) ทำให้ `Instant::now()` เข้าสู่ branch ที่
ไม่รองรับของ standard library แล้ว panic ทันที ข้อความ panic จริงที่ capture ได้ (ผ่าน
`console_error_panic_hook`):

```
panicked at /rustc/.../library/std/src/sys/pal/wasm/../unsupported/time.rs:13:9:
time not implemented on this platform
```

และ exception ที่ JavaScript จับได้จริงคือ `RuntimeError: unreachable` แบบเดียวกับหัวข้อ 87.9.1 (เพราะ
มันคือ panic ธรรมดา ไม่ต่างจาก panic อื่น ๆ ที่เกิดขึ้นข้างใน WASM) — stack trace จริงยืนยันจุดที่พังชัดเจน:
`std::time::Instant::now` เรียกมาจาก `tokio::time::instant::variant::now` ตรงกับที่อธิบายไว้ นี่คือตัวอย่าง
จริงว่า **"compile ผ่าน" ไม่ได้แปลว่า "ทำงานถูกต้องบน WASM"** — ต้อง test จริงบน target ที่ใช้งานจริงเสมอ
ไม่ใช่แค่เชื่อว่า compile ผ่านแล้วจบ วิธีแก้คือใช้ `gloo-timers` (crate ที่ wrap `setTimeout` ของ JS ให้เป็น
`Future` โดยเฉพาะสำหรับ WASM) หรือเขียน `JsFuture` ที่ wrap `setTimeout` เองผ่าน `js_sys::Promise` แทน
ไม่มีทางใช้ `tokio::time::sleep` ในบริบทนี้ได้เลยไม่ว่าจะเปิด feature ใดของ Tokio ก็ตาม

### กับดักที่ 6: ส่ง `number` ธรรมดาไปยังพารามิเตอร์ที่ฝั่ง Rust เป็น `u64`/`i64` แทน `BigInt`

ตรงกับตารางสรุปในหัวข้อ 87.12 — ฟังก์ชัน Rust ที่รับ/คืนค่าเป็น `u64`/`i64` ถูกแมปเป็น JavaScript `BigInt`
เสมอ (ไม่ใช่ `number` ธรรมดา) เพราะ `number` ของ JS คือ `f64` ที่แทนจำนวนเต็มแม่นยำได้แค่ถึง 2^53 เท่านั้น
โค้ดที่ลืมใส่ suffix `n` ตอนเรียก:

```javascript
console.log(wasm.big_number(42)); // ลืมใส่ n ต่อท้าย — ส่ง number ธรรมดา ไม่ใช่ BigInt
```

Error จริงที่ได้ (`wasm-bindgen` ตรวจสอบ type ของ argument ที่รับมาก่อนส่งเข้า WASM เสมอ ไม่ปล่อยให้ผ่านไป
แบบเงียบ ๆ แล้วได้ค่าเพี้ยน):

```
TypeError: expected a bigint argument, found number
```

วิธีแก้คือใส่ suffix `n` เสมอเมื่อเรียกฟังก์ชันที่รับ/คืน `u64`/`i64` (`wasm.big_number(42n)`) — และถ้า
ต้องนำค่านั้นไปคำนวณต่อกับ `number` ธรรมดาใน JS (เช่นค่าที่ได้จาก `Date.now()`) ต้องแปลงให้ตรงชนิดก่อนเสมอ
(`BigInt(Date.now())` หรือ `Number(bigIntValue)` — แต่การแปลงหลังนี้เสี่ยงเสีย precision ถ้าค่าใหญ่เกิน
2^53 ตามที่อธิบายไว้ในตาราง 87.12) ถ้าค่าตัวเลขของคุณไม่มีทางเกิน 2^53 ในทางปฏิบัติจริง (เช่น ID ที่นับ
ทีละ 1 ในระบบที่ไม่มีทางมีข้อมูลเกินสิบล้านล้านแถว) การเปลี่ยนไปใช้ `u32`/`i32` แทนตั้งแต่ต้นก็เป็นตัวเลือก
ที่ทำให้ฝั่ง JavaScript เขียนโค้ดง่ายขึ้นมากโดยไม่ต้องยุ่งกับ `BigInt` เลย

## แบบฝึกหัด (Exercises)

1. **โจทย์ระดับง่าย**: เขียนฟังก์ชัน `#[wasm_bindgen] pub fn to_uppercase(s: &str) -> String` ที่แปลง
   string เป็นตัวพิมพ์ใหญ่ทั้งหมด (ใช้ `str::to_uppercase()` ของ Rust ธรรมดา) แล้วเขียนไฟล์ JS ทดสอบสั้น ๆ
   ที่เรียกฟังก์ชันนี้กับ 3 string ตัวอย่าง พร้อมพิมพ์ผลลัพธ์ (hint: build ด้วย
   `wasm-pack build --target nodejs` แล้ว `require()` ไฟล์ที่ generate มาใน `pkg/`)

2. **โจทย์ระดับกลาง**: ขยาย `BookingCart` ในหัวข้อ 87.4.2 ให้มี method `seats_json(&self) -> Result<JsValue,
   JsValue>` ที่คืนรายชื่อที่นั่งทั้งหมดเป็น JS array ของ string (ใช้ `serde-wasm-bindgen` หรือ
   `js_sys::Array` ก็ได้) แล้วเขียนโค้ด JavaScript ที่เรียก method นี้แล้ว `.forEach()` พิมพ์ที่นั่งแต่ละ
   ตัวออกมา (hint: `Vec<String>` implement `Serialize` อยู่แล้วโดย derive ไม่ต้องเขียนเพิ่มถ้าใช้
   `serde-wasm-bindgen::to_value`)

3. **โจทย์ระดับยาก**: เขียนฟังก์ชัน `#[wasm_bindgen] pub async fn delayed_double(x: i32, ms: i32) ->
   i32` ที่รอเป็นเวลา `ms` มิลลิวินาที (ใช้ `js_sys::Promise` + `setTimeout` ผ่าน `Closure` เอง โดย**ห้าม
   ใช้ Tokio**) แล้วคืนค่า `x * 2` — ต้องใช้ `wasm_bindgen_futures::JsFuture` แปลง `Promise` ที่สร้างขึ้น
   เองให้ `.await` ได้ (hint: สร้าง `Promise` ด้วย `js_sys::Promise::new(&mut |resolve, _reject| { ... })`
   แล้วเรียก `resolve` ข้างใน callback ของ `setTimeout` ที่ import มาจาก JS)

4. **โจทย์ประยุกต์ใช้งานจริง**: ต่อจาก capstone ในหัวข้อ 87.11 — เพิ่มปุ่ม "ลบรายการที่เลือก" ที่เรียก
   `TodoList::remove(index)` โดยให้แต่ละ `<li>` ที่ render ออกมามีปุ่มลบของตัวเอง (ต้อง attach event
   listener ให้กับปุ่มลบแต่ละตัวที่สร้างขึ้นใหม่ทุกครั้งที่ `render()` ถูกเรียก ซึ่งหมายความว่าต้องคิดเรื่อง
   lifecycle ของ `Closure` หลายตัวพร้อมกันตามหัวข้อ 87.10.2 — จะ `.forget()` ทุกตัวไปเลยหรือจะเก็บ handle
   ไว้ลบ listener เก่าก่อนสร้างใหม่ทุกรอบ) ทดสอบด้วยการเพิ่ม 3 รายการ แล้วลบรายการกลาง ตรวจสอบว่ารายการ
   ที่เหลือ re-render ถูกต้อง (hint: ถ้าเลือกวิธี `.forget()` ทุกตัวไปเรื่อย ๆ จะมี closure ค้างอยู่ใน
   หน่วยความจำเพิ่มขึ้นทุกครั้งที่ render — ลองคิดดูว่าสำหรับ to-do list ขนาดเล็กที่ผู้ใช้ไม่ได้เพิ่ม/ลบ
   หลายพันครั้ง การ leak แบบนี้ยอมรับได้หรือไม่ เทียบกับความซับซ้อนของการจัดการ lifecycle ให้สมบูรณ์)

## สรุป

บทนี้เปิดกล่องดำที่ Part 86 ทิ้งไว้เป็นคำถาม: WASM รู้จักแต่ตัวเลข แล้วจะส่ง string, struct, error, และ
ข้อมูลซับซ้อนข้ามขอบเขตไปมากับ JavaScript ได้ยังไง — คำตอบคือ `wasm-bindgen`, proc macro ที่ generate
โค้ด marshaling ฝั่ง Rust และ JS glue code คู่กัน ทำให้การเรียกข้ามขอบเขตรู้สึกเป็นธรรมชาติทั้งสองทาง
เหมือนกับที่ Part 43 สอน `extern "C"` เป็น FFI boundary เข้าสู่ C แต่ต้องทำงานหนักกว่าเพราะปลายทางเป็น
dynamic type system ของ JavaScript

สิ่งสำคัญที่ควรติดตัวไปจากบทนี้: (1) การส่ง string/struct ข้ามขอบเขตมีต้นทุนจริงจากการ copy ข้อมูล —
struct ที่ export เป็น opaque handle หลีกเลี่ยงต้นทุนนี้ได้เพราะข้อมูลไม่เคยย้ายจาก WASM memory เลย
(2) WASM ในเบราว์เซอร์ไม่มี Tokio runtime — ใช้ event loop ของ JS ผ่าน `wasm_bindgen_futures` เสมอ
(3) panic ที่ข้าม WASM boundary ทำให้ instance ทั้งตัวไม่น่าเชื่อถืออีกต่อไป ควรสงวนไว้สำหรับบั๊กจริง ๆ
และใช้ `Result<T, JsValue>` สำหรับ error ที่คาดหวังได้ (4) WASM linear memory แยกจาก JS heap โดยสิ้นเชิง
— การจัดการ lifecycle ของ object ที่ข้าม boundary (ผ่าน `.free()`, `Closure::forget()`) ต้องทำด้วยความ
เข้าใจ ไม่ใช่การเดา

Part ถัดไปจะนำพื้นฐานทั้งหมดนี้ไปสร้างเป็น framework frontend เต็มรูปแบบด้วย **Yew** — framework ที่ใช้
แนวคิด component + virtual DOM แบบ React แต่เขียนด้วย Rust ล้วน ๆ ซ่อนการเรียก `web-sys`/`wasm-bindgen`
มือ ๆ แบบที่เขียนในบทนี้ไว้เบื้องหลัง component macro ที่สะดวกกว่ามาก — แต่ทุกอย่างที่ Yew ทำข้างใน
ก็ยังคงเป็นหลักการเดียวกับที่เรียนในบทนี้ทั้งหมด

---

**Part ก่อนหน้า:** [WebAssembly เบื้องต้นด้วย Rust](part-086-wasm-basics.md) | **Part ถัดไป:** [Yew Framework: Frontend ด้วย Rust](part-088-yew-framework.md)
