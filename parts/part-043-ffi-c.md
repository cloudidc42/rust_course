# Part 43: FFI: การเชื่อมต่อกับ C

> โมดูล: ระดับสูง (Advanced) | ระดับ: สูง | เวลาโดยประมาณ: 260 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้ว่าทำไม **FFI (Foreign Function Interface)** ถึงเป็นความสามารถ "first-class citizen" ของ Rust
  ตั้งแต่วันแรกที่ภาษาถูกออกแบบ ไม่ใช่ feature ที่แปะเพิ่มมาทีหลังแบบภาษาอื่น ๆ
- ประกาศ signature ของฟังก์ชันจากไลบรารี C ด้วย `extern "C"` block และเรียกใช้งานได้อย่างถูกต้อง
  พร้อมอธิบายได้ว่าทำไมทุกการเรียกต้องอยู่ใน `unsafe`
- เลือกใช้ type ที่ถูกต้องจาก `std::os::raw`/`std::ffi` เวลาแมป type ระหว่าง Rust กับ C แทนการเดาว่า
  `i32` เท่ากับ `int` เสมอ
- ส่งและรับ string ข้าม FFI boundary ด้วย `CString`/`CStr` ได้อย่างปลอดภัย โดยเข้าใจว่าทำไม `String`/`&str`
  ของ Rust ใช้งานตรง ๆ กับ C ไม่ได้
- ส่ง struct ข้าม FFI ด้วย `#[repr(C)]` ได้ถูกต้อง และอธิบายได้ว่าทำไมถ้าลืมใส่ (หรือใส่ field ผิดลำดับ)
  จะเกิดหายนะแบบไม่มี error เตือนตอน compile
- Expose ฟังก์ชัน Rust ให้ภาษา C เรียกกลับได้ ด้วย `#[no_mangle]` + `extern "C"` + crate-type ที่ถูกต้อง
  (`cdylib`/`staticlib`)
- ตั้งค่า linking ให้ถูกต้องด้วย `#[link(name = "...")]` หรือ `build.rs` และรู้จักอันตรายของ panic ที่ข้าม
  FFI boundary พร้อมวิธีป้องกัน
- รู้จักเครื่องมือ `bindgen`/`cbindgen` และเข้าใจว่าทำไมโปรเจกต์จริงถึงไม่เขียน binding มือเปล่าสำหรับ C API
  ขนาดใหญ่

## ความรู้ที่ต้องมีมาก่อน

- **Part 41 (Unsafe Rust เบื้องต้น)** — บทนี้สอน 5 "superpower" ที่ `unsafe` ปลดล็อกให้ ซึ่งหนึ่งในนั้นคือ
  **"เรียกใช้ `unsafe fn` รวมถึงฟังก์ชันจากภาษาอื่นผ่าน FFI"** — Part 41 เกริ่นไว้แค่ว่ามันมีอยู่ บทนี้คือบทที่
  ขยายความเต็มรูปแบบ พร้อมกับหลักการที่ Part 41 วางไว้ว่า **"`unsafe` ไม่ได้ปิดการตรวจสอบของ Rust ทั้งหมด
  มันแค่ย้ายภาระการพิสูจน์ความปลอดภัยจาก compiler ไปให้ตัวคุณเอง"** — หลักการนี้จะกลับมาเป็นแกนกลางของทุก
  หัวข้อในบทนี้
- **Part 42 (Raw Pointers และ Memory Layout)** — สอน `*const T`/`*mut T`, การ dereference ผ่าน `unsafe`,
  และเกริ่น `#[repr(C)]` ไว้ท้ายบทว่า "เหตุผลที่มันมีอยู่จะเห็นเต็ม ๆ ตอนเรียน FFI ใน Part 43" — บทนี้คือ
  จุดที่ปิดคำเกริ่นนั้น: `#[repr(C)]` **มีอยู่เพื่อ FFI โดยเฉพาะ** ไม่มีเหตุผลอื่นที่จำเป็นต้องใช้มันเลยถ้าโค้ด
  ของคุณอยู่ในโลก Rust ล้วน ๆ
- **Part 12 (Result และ Error Handling)** — `CStr::to_str()` คืนค่าเป็น `Result<&str, Utf8Error>` เพราะ
  C string ไม่มีการันตีว่าเป็น UTF-8 ที่ถูกต้อง ต้อง handle กรณีนี้แบบเดียวกับที่ Part 12 สอนไว้
- **Part 17 (Packages, Crates, Workspaces)** — แนวคิด crate-type (`lib`, `bin`) จะถูกขยายเป็น `cdylib` และ
  `staticlib` ตอนเรียนเรื่อง expose ฟังก์ชัน Rust ให้ C เรียก
- **Part 35 (Cargo ขั้นสูง)** — เกริ่น `build.rs` ไว้สั้น ๆ ว่าเป็นสคริปต์ที่รันก่อน compile บทนี้จะใช้มันจริง
  เพื่อบอก Cargo ว่าต้อง link กับ system library ตัวไหน

## เนื้อหา

### 43.1 ทำไม FFI ถึงสำคัญ: โลกที่เต็มไปด้วยโค้ด C หลายสิบปี

ก่อนจะลงมือเขียน `extern "C"` ตัวแรก ต้องเข้าใจก่อนว่า **ทำไม** เรื่องนี้ถึงสำคัญขนาดที่ Rust ต้องออกแบบมาให้
รองรับตั้งแต่วันแรก

ลองนึกภาพโลกของซอฟต์แวร์ที่มีอยู่จริงตอนนี้: ระบบปฏิบัติการเกือบทั้งหมด (Linux kernel, Windows API, macOS
Cocoa/Core Foundation) เปิด API ของตัวเองผ่าน **C ABI** เพราะ C เป็นภาษาแรก ๆ ที่ทุกระบบปฏิบัติการเลือกใช้
เป็น "ภาษากลาง" ในการสื่อสารกับ userland โปรแกรมทุกภาษาที่ต้องการคุยกับ OS ในระดับต่ำ (เปิดไฟล์, จัดการ
memory, สร้าง thread, เข้าถึง network socket) ล้วนต้องผ่าน C ABI ไม่ทางใดก็ทางหนึ่งอยู่ดี

นอกจาก OS API แล้ว ยังมีไลบรารีระดับโลกอีกมหาศาลที่เขียนด้วย C หรือ C++ (ซึ่งมักเปิด C ABI ให้เรียกจากภาษาอื่น
ได้ง่ายกว่า C++ ABI ที่ผันแปรตามคอมไพเลอร์) และถูกพัฒนา ทดสอบ และ optimize มาแล้วหลายสิบปี:

- **BLAS/LAPACK** — ไลบรารีคำนวณพีชคณิตเชิงเส้น (linear algebra) ที่เป็นหัวใจของงาน scientific computing
  แทบทุกวงการ ตั้งแต่ปี 1970s ถูก optimize จนแทบเป็นไปไม่ได้ที่จะเขียนใหม่ให้เร็วเท่าในเวลาอันสั้น
- **OpenSSL/libsodium** — ไลบรารี cryptography ที่ผ่านการตรวจสอบความปลอดภัยจากผู้เชี่ยวชาญนับพันคนมาหลายปี
  การเขียน cryptographic primitive ขึ้นใหม่เองมีความเสี่ยงสูงมาก (แม้จะเขียนด้วย Rust ก็ตาม) เพราะบั๊กเล็ก ๆ
  ในโค้ด crypto อาจนำไปสู่ช่องโหว่ความปลอดภัยที่ตรวจจับได้ยากมาก
- **libpng/libjpeg/zlib** — ไลบรารีจัดการไฟล์รูปภาพและการบีบอัดข้อมูลที่เป็นมาตรฐานอุตสาหกรรม
- **SQLite** — ฐานข้อมูลแบบ embedded ที่เขียนด้วย C และถูกทดสอบด้วย test suite ที่ครอบคลุมที่สุดในโลก
  ซอฟต์แวร์ (มากกว่า 100% ของ code coverage เพราะทดสอบ branch ที่ผสมกันหลายแบบด้วย)

การเขียนโปรแกรม Rust ที่ต้องใช้ความสามารถเหล่านี้มีสองทางเลือก: **เขียนทุกอย่างใหม่หมดด้วย Rust** (ใช้เวลา
เป็นปี ความเสี่ยงสูง และอาจไม่มีทางเก่งเท่าของเดิมที่ optimize มาหลายสิบปี) หรือ **เรียกใช้ไลบรารี C ที่มีอยู่
แล้วตรง ๆ** — Rust เลือกออกแบบให้ทางเลือกที่สองทำได้ง่ายและมีต้นทุนต่ำที่สุดเท่าที่จะเป็นไปได้ นี่คือเหตุผลที่
ทีม Rust ตัดสินใจให้ **C ABI compatibility เป็นส่วนหนึ่งของแกนภาษา** ตั้งแต่การออกแบบเริ่มต้น ไม่ใช่ feature
ที่ผนวกเข้ามาทีหลัง

#### 43.1.1 เปรียบเทียบกับภาษาอื่น: ทำไมของ Rust ถึง "ไม่ธรรมดา"

ภาษาระดับสูงส่วนใหญ่ก็มีวิธีเรียก C ได้เหมือนกัน แต่ด้วยต้นทุนที่แตกต่างกันมาก:

- **Python**: การเขียน C extension ต้องผ่าน `Python.h`, จัดการ **reference counting ของ CPython เองด้วยมือ**
  (`Py_INCREF`/`Py_DECREF`), ต้องคอยระวังการถือ **GIL (Global Interpreter Lock)** ให้ถูกจังหวะ, และต้อง
  compile เป็น shared object ที่ผูกกับเวอร์ชัน Python ABI เฉพาะ (`cp311`, `cp312`, ...) หรือใช้เครื่องมือช่วย
  อย่าง `ctypes`/`cffi` ที่สะดวกกว่าแต่ก็มี overhead จาก marshalling ข้อมูลผ่าน Python object wrapper ทุกครั้ง
  ที่ข้าม boundary — เขียน binding ระดับ production ที่ปลอดภัยสำหรับ C library ขนาดใหญ่ ใช้ความพยายามสูงมาก
- **Java**: ใช้ **JNI (Java Native Interface)** ซึ่งขึ้นชื่อเรื่อง boilerplate ที่ต้องเขียนมาก (ต้อง generate
  header ผ่าน `javah`/`javac -h`, เขียน native method signature ให้ตรงกับ mangled name ที่ JVM คาดหวัง,
  และทุกการข้าม boundary ระหว่าง JVM heap กับ native memory มี **overhead จากการ marshal ข้อมูล** และการที่
  JVM ต้อง "pin" object ไม่ให้ garbage collector ย้ายที่ระหว่างเรียก native code) แม้จะ optimize ได้ แต่ก็ต้อง
  ระมัดระวังสูงและมีต้นทุนที่มองไม่เห็นในระดับ runtime อยู่เสมอ
- **JavaScript (Node.js)**: ต้องผ่าน N-API หรือ V8-specific binding ซึ่งมีความซับซ้อนไม่ต่างจาก JNI และมี
  ปัญหาเรื่อง version compatibility ของ V8 engine ที่เปลี่ยนบ่อย

ในทางกลับกัน Rust: `extern "C"` block, `#[repr(C)]`, และ `#[no_mangle]` **ไม่มี runtime หรือ virtual machine
คั่นกลาง** — Rust compile ตรงไปเป็น native machine code ที่ใช้ **calling convention เดียวกันกับที่ C ใช้**
เมื่อ Rust เรียกฟังก์ชัน C หรือ C เรียกฟังก์ชัน Rust มันคือการ**เรียกฟังก์ชันธรรมดาในระดับ CPU instruction**
(ผลัก argument เข้า register ตาม ABI, `call`, อ่านค่าคืนจาก register) ไม่มี garbage collector ที่ต้อง pin
object ไม่มี interpreter ที่ต้อง marshal type ไม่มี reference counting ของ runtime ที่ต้องจัดการเพิ่ม — ต้นทุน
ของการเรียกข้าม FFI boundary ใน Rust จึง**เท่ากับการเรียกฟังก์ชันธรรมดา** (แทบจะ **zero-cost**) ซึ่งเป็นเหตุผล
ตรงที่ทำให้ Rust ถูกเลือกใช้เขียน "wrapper ที่ปลอดภัยกว่า" ให้กับไลบรารี C ที่มีอยู่แล้วอย่างแพร่หลายในวงการ
เช่น `rusqlite` (wrap SQLite), `openssl` crate (wrap OpenSSL), และ driver การเชื่อมต่อฐานข้อมูลอีกนับไม่ถ้วน

ราคาที่ต้องจ่ายสำหรับความเร็วและความง่ายนี้คือ: **compiler ของ Rust ไม่มีทางมองเห็นเข้าไปในโค้ด C ได้เลย**
ซึ่งจะเป็นหัวใจของหัวข้อ 43.3 — แต่ก่อนจะไปถึงจุดนั้น มาเริ่มจากไวยากรณ์พื้นฐานที่สุดก่อน

### 43.2 `extern "C"` block: ประกาศ signature ไม่ใช่ implementation

หัวใจสำคัญที่ต้องเข้าใจให้แม่นก่อนอื่นใด: **`extern "C"` block ไม่ได้เขียนโค้ดของฟังก์ชันขึ้นมาใหม่ มันแค่
"ประกาศ" (declare) ว่ามีฟังก์ชันชื่อนี้ รับ parameter แบบนี้ คืนค่าแบบนี้ อยู่ที่ไหนสักแห่งที่จะถูก link เข้ามา
ตอน compile** เปรียบเทียบง่าย ๆ กับการเขียน `.h` header file ในภาษา C: header ไม่มี implementation อยู่ข้างใน
มันแค่บอก compiler ว่า "ฟังก์ชันนี้มีอยู่จริง มี signature แบบนี้ ไปหา implementation จากที่อื่นตอน link"

คำว่า `"C"` ใน `extern "C"` คือการระบุ **ABI (Application Binary Interface)** — ข้อตกลงระดับ binary ว่า
argument จะถูกส่งผ่าน register หรือ stack แบบไหน, ใครมีหน้าที่ clean up stack หลังเรียกฟังก์ชัน, ชื่อฟังก์ชัน
ถูกเก็บใน binary แบบไหน (จะเห็นเรื่อง name mangling ในหัวข้อ 43.7) ฯลฯ — `"C"` คือ ABI ที่ผูกกับ C compiler
มาตรฐานของแต่ละแพลตฟอร์ม (เช่น System V AMD64 ABI บน Linux/macOS x86-64, หรือ Microsoft x64 calling
convention บน Windows) ซึ่งเกือบทุกภาษาที่รองรับ FFI เลือกใช้เป็น "ภาษากลาง" เพราะมันเสถียรและมีมาตรฐานชัดเจน
ที่สุด (ต่างจาก ABI ของ C++ ที่แต่ละ compiler ทำ name mangling และ layout ของ class ต่างกันได้)

#### 43.2.1 ตัวอย่างที่เล็กที่สุด: เรียก `abs()` จาก `<stdlib.h>`

`abs()` เป็นฟังก์ชันที่มีอยู่ในทุกระบบที่มี C standard library (libc) — signature จริงในภาษา C คือ
`int abs(int x)` มาดูวิธีเรียกจาก Rust:

```rust
use std::os::raw::c_int;

// extern "C" block: "ประกาศ" ว่ามีฟังก์ชันชื่อ abs อยู่ในไลบรารีที่จะ link เข้ามา
// ไม่มี { } implementation เพราะโค้ดจริงอยู่ใน libc ไม่ใช่ในไฟล์นี้
extern "C" {
    fn abs(x: c_int) -> c_int;
}

fn main() {
    let x: c_int = -42;
    // ทุกการเรียก extern function ต้องอยู่ใน unsafe block (เหตุผลเต็ม ๆ ในหัวข้อ 43.3)
    let result = unsafe { abs(x) };
    println!("abs({}) = {}", x, result);
}
```

ผลลัพธ์จริงจากการ compile และรันด้วย `rustc --edition 2021`:

```
abs(-42) = 42
```

สังเกตว่าตัวอย่างนี้**ไม่ต้องตั้งค่า linking เพิ่มเติมเลย** — เหตุผลคือ `libc` (C standard library) เป็น
ไลบรารีที่ Rust **link เข้ามาให้อัตโนมัติเสมอ** อยู่แล้ว (เพราะ Rust runtime เองก็ต้องพึ่งฟังก์ชันพื้นฐานบางตัว
จาก libc ไม่ว่าจะเป็น `malloc`/`free` ในหลายแพลตฟอร์ม หรือฟังก์ชันเริ่มต้นโปรแกรมของระบบปฏิบัติการ) แต่ถ้า
ต้องการเรียกฟังก์ชันจากไลบรารีอื่นที่ไม่ได้ link มาให้อัตโนมัติ ต้องบอก compiler ชัดเจน ซึ่งจะเห็นในตัวอย่าง
ถัดไป

#### 43.2.2 เมื่อต้อง link เพิ่มเอง: `sqrt()` จาก libm

`sqrt()` (`double sqrt(double x)`) อยู่ในไลบรารีคณิตศาสตร์ `libm` ซึ่งบนหลายระบบ Linux **ไม่ได้ถูก link มา
อัตโนมัติเหมือน libc** ต้องบอก compiler ด้วย attribute `#[link(name = "...")]`:

```rust
use std::os::raw::c_double;

// บอก linker ว่าต้อง link กับ libm (ชื่อไฟล์จริงบน Linux คือ libm.so — ตัด "lib" และ ".so" ออก เหลือ "m")
#[link(name = "m")]
extern "C" {
    fn sqrt(x: c_double) -> c_double;
}

fn main() {
    let result = unsafe { sqrt(64.0) };
    println!("sqrt(64.0) = {}", result);
}
```

ผลลัพธ์จริง:

```
sqrt(64.0) = 8
```

(หมายเหตุ: บน macOS สมัยใหม่ ฟังก์ชันคณิตศาสตร์พื้นฐานหลายตัวถูกรวมเข้าไปใน libSystem อยู่แล้วจึงอาจ compile
ผ่านได้แม้ไม่มี `#[link]` แต่การเขียน `#[link(name = "m")]` ไว้ชัดเจนคือ practice ที่ถูกต้องและพกพาข้ามระบบ
ได้ดีกว่า เพราะบน Linux ส่วนใหญ่จำเป็นต้องมีจริง ๆ)

รูปแบบทั้งสองตัวอย่างนี้คือ**โครงสร้างพื้นฐานที่สุด**ของการเรียก C จาก Rust: `extern "C"` block ประกาศ
signature, `#[link(name = "...")]` (ถ้าจำเป็น) บอกว่าจะไปหา implementation จากไลบรารีไหน, และ `unsafe`
ครอบทุกจุดที่เรียกจริง — หัวข้อถัดไปจะอธิบายว่าทำไม `unsafe` ต้องมาคู่กันเสมอแบบไม่มีทางเลี่ยง

### 43.3 ทำไมการเรียก `extern "C"` function ต้องเป็น `unsafe` เสมอ

นี่คือคำถามที่สำคัญที่สุดของบทนี้ และคำตอบเชื่อมกลับไปที่หลักการแกนกลางของ Part 41 ตรง ๆ: **`unsafe` คือ
สัญญาณว่าคุณกำลังรับผิดชอบสิ่งที่ compiler ตรวจสอบให้ไม่ได้อีกต่อไป**

ลองไล่เหตุผลทีละชั้นว่าทำไม compiler ถึง "ตรวจสอบให้ไม่ได้" ในกรณีของ FFI โดยเฉพาะ:

1. **Rust compiler ไม่มีทางมองเห็นเข้าไปใน implementation ของฟังก์ชัน C ได้เลย** — เมื่อคุณเขียน
   `extern "C" { fn abs(x: c_int) -> c_int; }` สิ่งที่ compiler มีอยู่ในมือคือ**แค่ signature ที่คุณพิมพ์ลง
   ไปเอง** ไม่มี source code ของ `abs()` จริง ๆ ให้ตรวจสอบ (implementation อยู่ใน `libc.so`/`libc.a` ที่ถูก
   compile ไว้ล่วงหน้าเป็น binary แล้ว) compiler ไม่มีทางรู้ว่าฟังก์ชันนั้น**ทำอะไรจริง ๆ** ข้างใน มันอาจ
   dereference null pointer, เขียนทับ memory ที่ไม่ควรแตะ, หรือมี undefined behavior ของตัวเองอยู่แล้วก็ได้
   — Rust ไม่มีทางตรวจสอบ memory safety ของโค้ดที่มันไม่เคยเห็นแม้แต่ตัวอักษรเดียว
2. **ไม่มีอะไรยืนยันว่า signature ที่คุณเขียนตรงกับ signature จริงของฟังก์ชัน C 100%** — นี่คือจุดที่อันตราย
   ที่สุดและเป็นบั๊ก FFI คลาสสิกที่พบบ่อยที่สุด สมมติ `abs()` จริงในไลบรารีรับ `int` (32-bit) แต่คุณพิมพ์
   `extern "C" { fn abs(x: c_long) -> c_long; }` (สมมติเป็น 64-bit) — **compiler ไม่มีทางรู้ว่าคุณพิมพ์ผิด**
   เพราะมันไม่มี "ต้นฉบับ" ให้เทียบ มันจะเชื่อสิ่งที่คุณเขียนแบบไม่มีข้อกังขา แล้ว generate machine code ตาม
   signature ที่คุณให้ไว้ — ผลคือโปรแกรม**compile ผ่านได้ปกติ 100% ไม่มี warning ไม่มี error** แต่ตอนรันจริง
   argument จะถูกส่งไปในรูปแบบที่ผิด (จำนวน byte ผิด, การตีความ signed/unsigned ผิด) เกิด **undefined
   behavior** ที่ตรวจจับไม่ได้เลยจนกว่าจะรันแล้วเจอผลลัพธ์แปลก ๆ (ดูตัวอย่างจริงในหัวข้อกับดักที่ 43.10.1)

ทั้งสองข้อนี้คือเหตุผลเดียวกันที่ Part 41 อธิบายไว้กับ raw pointer dereference และ `unsafe fn` ทั่วไป:
**compiler ตรวจสอบเงื่อนไขความปลอดภัยที่มัน "มองเห็น" ได้เท่านั้น** สำหรับ FFI ไม่มีอะไรให้มองเห็นเลยนอกจาก
คำสัญญาที่คุณพิมพ์ขึ้นมาเอง `unsafe` ในบริบทนี้จึงแปลตรง ๆ ได้ว่า:

> **"ฉัน (ผู้เขียนโค้ด) รับรองว่า signature ที่ฉันเขียนตรงกับของจริงทุกตัวอักษร และฟังก์ชันนี้จะไม่ทำอะไรที่
> ละเมิดกฎความปลอดภัยของ Rust เมื่อมันถูกเรียก — ฉันรับผิดชอบเรื่องนี้เอง ไม่ใช่ compiler"**

สังเกตว่านี่คือหลักการเดียวกันเป๊ะกับที่ Part 41 สรุปไว้ว่า `unsafe` **ไม่ได้ปิดการตรวจสอบ borrow checker
หรือ type checker ทั้งหมด** — Rust ยังตรวจ type ของ argument ที่คุณส่งให้ `abs()` ว่าตรงกับ signature ที่คุณ
เขียนไว้เองอย่างเคร่งครัด (ถ้าคุณส่ง `&str` เข้าไปใน parameter ที่ประกาศเป็น `c_int` มันจะ error ทันที) —
สิ่งที่ `unsafe` ปลดล็อกในที่นี้แคบมาก ๆ: แค่ **"อนุญาตให้เรียกฟังก์ชันที่ compiler ไม่มีทางพิสูจน์ความถูกต้อง
ของ signature หรือความปลอดภัยของ implementation ได้"** เท่านั้น ทุกอย่างที่ compiler ตรวจสอบได้ (type
matching ตาม signature ที่คุณเขียน, การไม่ลืม `unsafe` block, ownership ของค่าที่ส่งเข้า/ออก) ยังถูกตรวจสอบ
เต็มรูปแบบเหมือนเดิม

### 43.4 การแมป type ระหว่าง Rust กับ C: `std::os::raw` / `std::ffi`

สมมติฐานที่ผิดที่พบบ่อยที่สุดในหมู่ผู้เริ่มเขียน FFI คือ **"C's `int` เท่ากับ Rust's `i32` เสมอ"** — สมมติฐาน
นี้ถูกในทางปฏิบัติบนแพลตฟอร์มส่วนใหญ่ที่ใช้กันทั่วไปในปัจจุบัน แต่ **มาตรฐานภาษา C ไม่ได้บังคับขนาดของ `int`
ไว้ตายตัว** — มันกำหนดแค่ขั้นต่ำ (`int` ต้องมีอย่างน้อย 16-bit) ขนาดจริงขึ้นอยู่กับแพลตฟอร์มและ compiler
(บนระบบ embedded บางตัว `int` อาจเป็น 16-bit; ขนาดของ `long` ก็ต่างกันชัดเจนระหว่าง Linux 64-bit ที่เป็น
64-bit กับ Windows 64-bit ที่ `long` ยังเป็น 32-bit อยู่ ทั้งที่เป็นระบบ 64-bit เหมือนกัน — ปรากฏการณ์นี้
เรียกว่า "LLP64" บน Windows เทียบกับ "LP64" บน Linux/macOS)

ถ้าคุณเขียน `extern "C" { fn some_func(x: i32); }` เพราะเดาว่า parameter เป็น `int` แล้วมันดันเป็น `long`
บนแพลตฟอร์มที่ `long` ไม่เท่ากับ 32-bit โปรแกรมของคุณจะพังทันทีที่ข้าม platform โดยไม่มี error เตือนตอน
compile (เหตุผลเดียวกับหัวข้อ 43.3) — Rust จึงมี module `std::os::raw` (และ re-export ผ่าน `std::ffi`)
ที่ให้ type ที่**ผูกกับขนาดจริงของ type C บนแพลตฟอร์มที่กำลัง compile อยู่โดยอัตโนมัติ** แทนการเดาเอง

ตารางการแมป type ที่ใช้บ่อยที่สุด:

| C type | Rust type (`std::os::raw`) | หมายเหตุ |
|---|---|---|
| `int` | `c_int` | ปกติคือ `i32` แต่ไม่ตายตัวเสมอไปในทางทฤษฎี |
| `unsigned int` | `c_uint` | ปกติคือ `u32` |
| `char` | `c_char` | **สำคัญมาก**: บน x86 ส่วนใหญ่คือ `i8` (signed) แต่บน ARM หลายระบบคือ `u8` (unsigned) — ห้ามเดาเอง |
| `signed char` | `c_schar` | บังคับให้เป็น signed เสมอไม่ว่าแพลตฟอร์มไหน |
| `unsigned char` | `c_uchar` | บังคับให้เป็น unsigned เสมอ (คือ `u8`) |
| `short` | `c_short` | ปกติคือ `i16` |
| `unsigned short` | `c_ushort` | ปกติคือ `u16` |
| `long` | `c_long` | **ต่างกันข้าม platform**: `i64` บน Linux/macOS 64-bit, `i32` บน Windows 64-bit |
| `unsigned long` | `c_ulong` | เช่นเดียวกับข้างบนแต่เป็น unsigned |
| `long long` | `c_longlong` | ปกติคือ `i64` บนแทบทุกแพลตฟอร์มสมัยใหม่ |
| `float` | `c_float` | คือ `f32` (มาตรฐาน IEEE 754 single-precision ตรงกันข้ามภาษา) |
| `double` | `c_double` | คือ `f64` (IEEE 754 double-precision) |
| `void *` | `*mut c_void` | pointer ทั่วไปที่ไม่รู้ type ข้างใน (คล้าย `void*` จริง ๆ) |
| `const void *` | `*const c_void` | pointer แบบอ่านอย่างเดียว |
| `size_t` | `usize` | ขนาดที่พอดีกับ pointer ของแพลตฟอร์ม — Rust มี `usize` ที่ตรงกับความหมายนี้อยู่แล้วโดยตรง ไม่ต้องผ่าน `std::os::raw` |
| `_Bool` / `bool` (C99+) | `bool` | ตั้งแต่ C99 ที่มี `_Bool` ทั้งสองภาษามี layout ตรงกันพอดี (1 byte) |

หลักการใช้งานที่ควรจำ: **เขียน type จาก `std::os::raw` เสมอเมื่อประกาศ `extern "C"` block แทนการเขียน
`i32`/`i64` ตรง ๆ** แม้ในทางปฏิบัติทั้งสองจะให้ผลลัพธ์เดียวกันบนแพลตฟอร์มที่คุณ compile อยู่ตอนนี้ก็ตาม —
เหตุผลคือถ้าใครเอาโค้ดของคุณไป compile ข้ามแพลตฟอร์มในอนาคต (cross-compilation เป็นเรื่องปกติมากในโลก Rust
เพราะ toolchain รองรับดีมาก) `c_long` จะปรับขนาดให้ถูกต้องอัตโนมัติ แต่ `i64` ที่เขียนตรง ๆ จะยังเป็น 64-bit
เสมอไม่ว่าแพลตฟอร์มปลายทางจะนิยาม `long` เป็นอะไรก็ตาม — นี่คือ**ความแตกต่างระหว่าง "โค้ดที่ compile ผ่าน
บนเครื่องฉันตอนนี้" กับ "โค้ดที่ถูกต้องในทุกสภาพแวดล้อม"**

ตัวอย่างที่ใช้ type mapping ถูกต้องกับหลายฟังก์ชันพร้อมกัน:

```rust
use std::os::raw::c_int;

extern "C" {
    fn getpid() -> c_int;
    fn abs(x: c_int) -> c_int;
}

#[link(name = "m")]
extern "C" {
    fn sqrt(x: f64) -> f64;
    fn pow(base: f64, exp: f64) -> f64;
}

fn main() {
    let pid = unsafe { getpid() };
    println!("PID ของโปรเซสนี้ = {}", pid);
    println!("abs(-99) = {}", unsafe { abs(-99) });
    println!("sqrt(2.0) = {}", unsafe { sqrt(2.0) });
    println!("pow(2.0, 10.0) = {}", unsafe { pow(2.0, 10.0) });
}
```

ผลลัพธ์จริง (PID จะต่างกันไปทุกครั้งที่รัน เพราะมันคือ process ID จริงของ process ที่กำลังรัน):

```
PID ของโปรเซสนี้ = 23661
abs(-99) = 99
sqrt(2.0) = 1.4142135623730951
pow(2.0, 10.0) = 1024
```

(หมายเหตุ: ในตัวอย่างนี้ใช้ `f64` ตรง ๆ แทน `c_double` เพื่อความกระชับ — เพราะ `f64` **การันตีเป็น IEEE 754
double-precision เสมอตามมาตรฐานภาษา Rust** ไม่ผันแปรตามแพลตฟอร์มแบบ `c_long`/`c_int` จึงเขียนตรง ๆ ได้อย่าง
ปลอดภัย 100% ในทุกกรณี — สิ่งที่ต้องระวังจริง ๆ คือ type จำพวก `int`/`long`/`char` ที่ขนาดผันแปรได้เท่านั้น)

`getpid()` เป็นตัวอย่างที่ดีของฟังก์ชัน libc ที่**ไม่รับ argument เลย**และ**ไม่มีทางทำให้เกิด memory unsafety
ได้จากตัวมันเอง** (มันแค่คืนตัวเลข PID กลับมา) แต่ต่อให้ปลอดภัยขนาดนี้ Rust ก็ยังบังคับให้ครอบด้วย `unsafe`
เสมอ — เพราะเหตุผลในหัวข้อ 43.3 ไม่ได้ขึ้นกับว่าฟังก์ชันนั้น "อันตรายจริงหรือไม่" แต่ขึ้นกับว่า **compiler
ไม่มีทางพิสูจน์ได้ว่ามันปลอดภัย** ไม่ว่าโดยข้อเท็จจริงมันจะปลอดภัยแค่ไหนก็ตาม — กฎนี้ใช้แบบเดียวกันหมดไม่มี
ข้อยกเว้นสำหรับ "ฟังก์ชันที่ดูไม่อันตราย"

### 43.5 ส่ง string ข้าม FFI boundary: `CString` และ `CStr`

นี่คือหัวข้อที่สำคัญที่สุดและมักเป็นจุดที่ผิดพลาดบ่อยที่สุดของ FFI ทั้งหมด เพราะ **วิธีที่ Rust กับ C เก็บ
string ในหน่วยความจำแตกต่างกันโดยพื้นฐาน**

ทวนจาก Part 14: `String`/`&str` ของ Rust เก็บ **length แยกเป็นตัวเลขต่างหาก** (fat pointer ที่มีทั้ง pointer
ไปยังข้อมูลและค่าความยาว) — มันจึงรองรับ byte `\0` (null byte) อยู่ตรงกลาง string ได้ตามปกติ และการหาความยาว
ทำได้ในเวลาคงที่ `O(1)` เพราะมี length เก็บไว้แล้ว ไม่ต้องไล่หา

ในทางกลับกัน string ของ C (`char *`) เป็น **null-terminated string** — ไม่มีที่เก็บความยาวแยกต่างหากเลย
วิธีเดียวที่จะรู้ว่า string จบตรงไหนคือ**ไล่อ่านไปทีละ byte จนกว่าจะเจอ byte ที่มีค่า `0`** (นี่คือเหตุผลที่
`strlen()` ใน C ทำงานเป็น `O(n)` เสมอ ไม่ใช่ `O(1)`) ผลที่ตามมาคือ:

- **`&str`/`String` ของ Rust ส่งให้ C ตรง ๆ ไม่ได้** เพราะ C ไม่รู้จัก fat pointer และไม่มีการันตีว่ามี
  `\0` ต่อท้าย
- **`char *` ของ C รับเข้ามาใน Rust ตรง ๆ เป็น `&str` ไม่ได้** เพราะ Rust ไม่รู้ความยาวล่วงหน้า ต้องไล่หา
  `\0` ก่อน และต้องตรวจสอบว่าข้อมูลที่ได้เป็น **valid UTF-8** หรือไม่ (C string ไม่มีการันตีเรื่อง encoding
  เลย มันอาจเป็น byte อะไรก็ได้ที่ไม่ใช่ `0`)

Rust แก้ปัญหานี้ด้วยสอง type ใน `std::ffi`: **`CString`** (owned, สำหรับส่ง string ของ Rust ออกไปให้ C) และ
**`CStr`** (borrowed, สำหรับรับ string ที่ C ส่งมาให้ Rust)

#### 43.5.1 `CString`: ส่ง string จาก Rust ไปให้ C

`CString` คือ owned buffer ที่**การันตีว่ามี `\0` ต่อท้ายเสมอ**และ**ไม่มี `\0` อยู่ตรงกลาง** (เพราะถ้ามี C
จะอ่านผิดว่า string จบตรงกลาง) `CString::new()` คืนค่าเป็น `Result` — จะ error ก็ต่อเมื่อ string ต้นฉบับมี
byte `0` แอบอยู่ข้างในเท่านั้น (ซึ่งเกิดยากมากในทางปฏิบัติสำหรับ text ทั่วไป):

```rust
use std::ffi::CString;
use std::os::raw::c_char;

extern "C" {
    fn strlen(s: *const c_char) -> usize;
}

fn main() {
    // CString::new สร้าง buffer ที่มี \0 ต่อท้ายให้อัตโนมัติ
    let s = CString::new("Hello, FFI!").unwrap();
    // as_ptr() คืน *const c_char ที่ชี้ไปยัง buffer นั้น — ใช้ส่งให้ฟังก์ชัน C ที่ต้องการ char*
    let len = unsafe { strlen(s.as_ptr()) };
    println!("strlen(\"Hello, FFI!\") = {}", len);
}
```

ผลลัพธ์จริง:

```
strlen("Hello, FFI!") = 11
```

**จุดที่ต้องระมัดระวังมากที่สุดของ `CString`** คือเรื่อง **lifetime**: `as_ptr()` คืน raw pointer ที่ชี้ไปยัง
buffer ภายใน `CString` — pointer นี้ **ใช้ได้ตราบเท่าที่ `CString` เจ้าของมันยังไม่ถูก drop เท่านั้น** ถ้าเขียน
โค้ดแบบนี้:

```rust
use std::ffi::CString;
use std::os::raw::c_char;

fn get_dangling_ptr() -> *const c_char {
    let s = CString::new("temporary").unwrap();
    s.as_ptr() // s จะถูก drop ทันทีที่ฟังก์ชันนี้ return! pointer ที่คืนไปกลายเป็น dangling pointer
} // <-- s ถูก drop ที่นี่ buffer ถูกปล่อยคืนหน่วยความจำ

fn main() {
    let ptr = get_dangling_ptr();
    // ใช้ ptr ต่อจากนี้คือ undefined behavior — ชี้ไปยัง memory ที่ถูกปล่อยคืนไปแล้ว
    println!("ptr = {:p}", ptr);
}
```

โค้ดนี้ **compile ผ่านได้ปกติ** เพราะ raw pointer ไม่ผูกกับ borrow checker เหมือน reference (`&T`) — ตรงนี้คือ
จุดที่ borrow checker "มองไม่เห็น" ปัญหา เพราะ `CString` ถูก drop แล้ว pointer ที่เหลืออยู่กลายเป็น **dangling
pointer** ทันที การใช้งานที่ถูกต้องคือต้องเก็บ `CString` ให้มีชีวิตอยู่ตลอดช่วงที่ pointer ถูกใช้งาน:

```rust
use std::ffi::CString;
use std::os::raw::c_char;

extern "C" {
    fn strlen(s: *const c_char) -> usize;
}

fn main() {
    let s = CString::new("temporary").unwrap(); // s ยังมีชีวิตอยู่ตลอด main()
    let ptr = s.as_ptr();
    let len = unsafe { strlen(ptr) }; // ใช้ ptr ตอนที่ s ยังไม่ถูก drop — ปลอดภัย
    println!("len = {}", len);
} // s ถูก drop ที่นี่ หลังจากใช้งาน ptr เสร็จแล้ว
```

#### 43.5.2 `CStr`: รับ string ที่ C ส่งมาให้ Rust

`CStr` คือ **borrowed view** ของ null-terminated C string — มันไม่ได้เป็นเจ้าของข้อมูล (ต่างจาก `CString`)
แค่ "มองผ่าน" raw pointer ที่ได้มาจาก C แล้วให้ method สำหรับแปลงเป็น `&str` ของ Rust ได้อย่างปลอดภัย

ตัวอย่างจริงที่เรียก `getenv()` จาก libc (คืนค่าเป็น `char *` ที่ชี้ไปยัง environment variable หรือ null ถ้า
ไม่พบ):

```rust
use std::ffi::{CStr, CString};
use std::os::raw::c_char;

extern "C" {
    fn getenv(name: *const c_char) -> *mut c_char;
}

fn main() {
    let key = CString::new("HOME").unwrap();
    let val_ptr = unsafe { getenv(key.as_ptr()) };

    if val_ptr.is_null() {
        println!("ไม่พบ environment variable HOME");
    } else {
        // CStr::from_ptr เป็น unsafe เพราะ Rust ไม่มีทางรู้ว่า val_ptr ชี้ไปยัง
        // memory ที่ valid จริงหรือไม่ และไม่รู้ว่ามี \0 ต่อท้ายอยู่จริงหรือเปล่า
        // (ถ้าไม่มี \0 เลย from_ptr จะไล่อ่านเรื่อยไปจนกว่าจะเจอ byte 0 ที่ไหนสักแห่ง
        //  ซึ่งอาจอ่านเลย buffer ที่ตั้งใจไว้ กลายเป็น buffer over-read)
        let val = unsafe { CStr::from_ptr(val_ptr) };

        // to_str() คืน Result<&str, Utf8Error> เพราะ C string ไม่การันตีว่าเป็น UTF-8 ที่ถูกต้อง
        // ตรงกับหลักการ error handling ที่ Part 12 สอนไว้: สิ่งที่ "อาจ" ผิดพลาดต้องคืนเป็น Result
        match val.to_str() {
            Ok(s) => println!("HOME = {}", s),
            Err(_) => println!("HOME มีข้อมูลที่ไม่ใช่ UTF-8 ที่ถูกต้อง"),
        }
    }
}
```

ผลลัพธ์จริง (ค่าจริงขึ้นอยู่กับเครื่องที่รัน):

```
HOME = /root
```

จุดสำคัญที่ต้องเข้าใจให้ลึกคือ**ทำไม `to_str()` ต้องคืน `Result` แทนที่จะคืน `&str` ตรง ๆ** — เพราะ Rust's
`&str` **การันตี 100% ว่าเป็น valid UTF-8 เสมอ** (การันตีนี้คือรากฐานของทุก method บน `&str` ที่ Rust ให้มา
ตั้งแต่ Part 14) แต่ C string เป็นแค่ลำดับของ byte ที่จบด้วย `0` — ไม่มีการันตีเรื่อง encoding เลย มันอาจเป็น
Latin-1, Shift-JIS, หรือ byte เสียหายจากบั๊กของโปรแกรม C ฝั่งตรงข้ามก็ได้ทั้งนั้น การแปลง C string เป็น `&str`
จึงเป็น operation ที่ **"อาจล้มเหลว"** โดยธรรมชาติ และ Rust บังคับให้คุณต้อง handle ทั้งสองกรณีอย่างชัดเจนผ่าน
`Result` — ตรงกับหลักการที่ Part 12 สอนไว้ว่า Rust เลือกทำให้ "ความล้มเหลวที่เป็นไปได้" ปรากฏอยู่ใน type
signature เสมอ แทนที่จะซ่อนไว้แล้วปล่อยให้ panic หรือคืนค่าผิดแบบเงียบ ๆ ตอน runtime

ตารางสรุปความแตกต่างระหว่างสอง type:

| | `CString` | `CStr` |
|---|---|---|
| ความเป็นเจ้าของ | Owned (เป็นเจ้าของ buffer เอง) | Borrowed (แค่ยืมดู ไม่เป็นเจ้าของ) |
| ใช้ตอนไหน | ส่ง string จาก Rust ไปให้ C | รับ string ที่ C ส่งมาให้ Rust |
| สร้างด้วย | `CString::new(...)` (คืน `Result`) | `unsafe { CStr::from_ptr(...) }` |
| แปลงเป็น `&str` | `.to_str()` (เพราะยัง valid UTF-8 อยู่แล้วตอนสร้าง แต่ก็ยัง Option/Result เพื่อความสอดคล้อง) | `.to_str()` (คืน `Result<&str, Utf8Error>` เพราะไม่รู้ encoding ล่วงหน้า) |
| คล้ายกับคู่ไหนใน std | `String` (owned) | `&str` (borrowed) |

### 43.6 ส่ง struct ข้าม FFI: ทำไม `#[repr(C)]` ต้องมีเสมอ

Part 42 เกริ่นไว้ว่า `#[repr(C)]` มีอยู่เพื่อ FFI โดยเฉพาะ — มาถึงตอนนี้คือจุดที่จะเห็นเหตุผลแบบเต็ม ๆ ด้วย
หลักฐานจริงจากการรันโปรแกรม

ทวนจาก Part 42: `struct` ธรรมดาใน Rust ใช้ **`#[repr(Rust)]`** โดย default (ไม่ต้องเขียนอะไรเพิ่ม มันเป็นค่า
เริ่มต้น) และ **`repr(Rust)` ไม่การันตีอะไรเกี่ยวกับ memory layout เลยแม้แต่นิดเดียว** — compiler มีสิทธิ์
**เรียง field ใหม่ตามลำดับที่คิดว่าเหมาะสมที่สุด** (เพื่อลด padding และประหยัดหน่วยความจำ) ไม่ต้องตรงกับลำดับ
ที่คุณเขียนในโค้ดเลย นี่คือ **optimization ที่ compiler อนุญาตให้ทำได้** เพราะไม่มีโค้ด Rust ตัวไหนควรจะสน
"ตำแหน่ง byte จริง" ของ field ภายใน struct — ทุกอย่างเข้าถึงผ่านชื่อ field เสมอ (`point.x`, `point.y`)
compiler จัดการ offset ให้เองโดยอัตโนมัติไม่ว่าจะเรียงยังไงก็ตาม

ปัญหาเกิดตอนที่ struct นั้นต้องข้าม FFI boundary ไปคุยกับโค้ด C — **โค้ด C ไม่รู้จัก "การเรียงใหม่แบบ Rust"
เลย** มันคาดหวัง layout ตาม**ลำดับ field ที่เขียนไว้ใน struct definition ของ C เป๊ะ ๆ** (พร้อม padding ตาม
กฎ alignment ของ C ที่เป็นมาตรฐานสากล) ถ้า Rust struct ที่ส่งไปใช้ `repr(Rust)` (default) แล้ว compiler
ตัดสินใจเรียง field ใหม่ ผลคือ **byte ที่ตำแหน่งหนึ่งใน struct ของ Rust จะไม่ตรงกับ byte ที่ตำแหน่งเดียวกันใน
struct ของ C เลย** — โค้ด C จะอ่านค่าผิด field ไปโดยสิ้นเชิงแบบเงียบ ๆ ไม่มี error ไม่มี warning

มาดูหลักฐานจริงว่า `repr(Rust)` เรียง field ใหม่จริง ๆ ด้วยการรันโปรแกรมเทียบกัน:

```rust
use std::mem::{offset_of, size_of};

// เรียง field แบบ "เล็ก-ใหญ่-เล็ก" ตั้งใจเพื่อให้ compiler มีโอกาสจัดเรียงใหม่ถ้าใช้ repr(Rust)
struct RustLayout {
    a: u8,
    b: u64,
    c: u8,
}

#[repr(C)]
struct CLayout {
    a: u8,
    b: u64,
    c: u8,
}

fn main() {
    println!("RustLayout size = {}", size_of::<RustLayout>());
    println!("CLayout   size = {}", size_of::<CLayout>());
    println!(
        "RustLayout offsets: a={} b={} c={}",
        offset_of!(RustLayout, a),
        offset_of!(RustLayout, b),
        offset_of!(RustLayout, c)
    );
    println!(
        "CLayout   offsets: a={} b={} c={}",
        offset_of!(CLayout, a),
        offset_of!(CLayout, b),
        offset_of!(CLayout, c)
    );
}
```

ผลลัพธ์จริงจากการ compile และรันด้วย `rustc --edition 2021` (ตัวอย่างนี้ใช้ `std::mem::offset_of!` ซึ่ง
stable ตั้งแต่ Rust 1.77):

```
RustLayout size = 16
CLayout   size = 24
RustLayout offsets: a=8 b=0 c=9
CLayout   offsets: a=0 b=8 c=16
```

นี่คือหลักฐานที่ชัดเจนที่สุดที่จะเห็นได้: **`RustLayout` ที่เขียน field เป็น `a, b, c` ตามลำดับ แต่ offset จริง
กลับเป็น `b` (offset 0), `a` (offset 8), `c` (offset 9)** — compiler ย้าย `b` (ซึ่งเป็น `u64` ที่ต้องการ
alignment 8 byte) ไปไว้ก่อนเพื่อลด padding ทำให้ขนาดรวมเหลือแค่ 16 byte ในขณะที่ `CLayout` ซึ่งใช้
`#[repr(C)]` **เก็บลำดับ `a, b, c` ตามที่เขียนไว้เป๊ะ** (offset 0, 8, 16 ตามลำดับ) พร้อม padding ตามกฎ C
มาตรฐาน ทำให้ขนาดรวมกลายเป็น 24 byte (ใหญ่กว่า แต่**คาดเดาได้และตรงกับที่ C compiler จะสร้าง struct เดียวกัน
ให้พอดี**)

ถ้าคุณส่ง `RustLayout` (แบบ default) ไปให้ฟังก์ชัน C ที่คาดหวัง struct `{a; b; c;}` ตามลำดับที่เขียน โค้ด C
จะอ่าน byte ที่ offset 0-7 (ซึ่งจริง ๆ คือ field `b` ของ Rust) แล้วตีความว่าเป็น field `a` — เกิดการอ่านข้อมูล
ผิดโดยสิ้นเชิงแบบไม่มี error ใด ๆ เตือนเลย

#### 43.6.1 ตัวอย่างเต็ม: ส่ง struct ผ่าน pointer ไปให้ฟังก์ชัน C

มาดูตัวอย่างที่ใช้งานได้จริง สมมติมีฟังก์ชัน C ต่อไปนี้เก็บไว้ในไฟล์ `point.c`:

```c
#include <math.h>

typedef struct {
    double x;
    double y;
} Point;

double point_distance(const Point *a, const Point *b) {
    double dx = a->x - b->x;
    double dy = a->y - b->y;
    return sqrt(dx * dx + dy * dy);
}
```

และฝั่ง Rust ที่เรียกใช้ (struct `Point` ทั้งสองฟิลด์เป็น `f64` เหมือนกันหมด กรณีนี้แม้ `repr(Rust)` อาจเรียง
ลำดับไม่ต่างจาก `repr(C)` เพราะทั้งสอง field มีขนาด/alignment เท่ากัน แต่ **ห้ามพึ่งพาโชคแบบนี้** — ต้องเขียน
`#[repr(C)]` ให้ชัดเจนเสมอเพื่อความถูกต้องที่การันตีได้ ไม่ใช่ "บังเอิญถูกในกรณีนี้"):

```rust
// ต้องใส่ #[repr(C)] เสมอเมื่อ struct นี้จะข้าม FFI boundary
// เพื่อการันตีว่า layout ตรงกับ struct Point ในไฟล์ point.c เป๊ะ ไม่ขึ้นกับว่า compiler จะ
// "บังเอิญ" ไม่เรียงลำดับใหม่หรือไม่ในกรณีที่ field มีขนาดเท่ากันแบบนี้
#[repr(C)]
struct Point {
    x: f64,
    y: f64,
}

extern "C" {
    fn point_distance(a: *const Point, b: *const Point) -> f64;
}

fn main() {
    let a = Point { x: 0.0, y: 0.0 };
    let b = Point { x: 3.0, y: 4.0 };
    // ส่ง reference ของ struct — Rust แปลง &Point ให้เป็น *const Point ให้อัตโนมัติตรงจุดนี้
    let dist = unsafe { point_distance(&a, &b) };
    println!("distance = {}", dist);
}
```

Compile ทั้งสองไฟล์เข้าด้วยกัน (คำสั่งจริงที่ใช้ทดสอบ):

```bash
gcc -c point.c -o point.o
rustc --edition 2021 main.rs -C link-arg=point.o -C link-arg=-lm -o main
./main
```

ผลลัพธ์จริง:

```
distance = 5
```

ระยะทางจากจุด `(0, 0)` ไปยัง `(3, 4)` คือ `5` ตามสูตร Pythagoras (`3-4-5` triangle) — ตรงตามที่คาดหวัง
เพราะ layout ของ `Point` ทั้งสองฝั่งตรงกันเป๊ะ

ในโปรเจกต์จริงที่ใช้ Cargo ปกติจะไม่ compile ด้วยคำสั่งมือแบบนี้ แต่ใช้ **`build.rs`** compile ไฟล์ C ให้
อัตโนมัติผ่าน crate ช่วยอย่าง [`cc`](https://crates.io/crates/cc) (จะเห็นวิธีตั้งค่าใน 43.8) — แต่หลักการ
ที่ต้องจำคือ **`#[repr(C)]` ต้องมีอยู่ในทุก struct ที่ข้าม FFI boundary ไม่มีข้อยกเว้น** แม้ว่าในบางกรณี
`repr(Rust)` อาจให้ผลลัพธ์ layout เดียวกันโดยบังเอิญก็ตาม เพราะ **`repr(Rust)` ไม่มีการันตีอะไรเลยตามสัญญา
ของภาษา** — compiler มีสิทธิ์เปลี่ยน layout ในเวอร์ชันถัดไปได้เสมอโดยไม่ถือว่าเป็น breaking change (เพราะ
`repr(Rust)` ไม่เคยสัญญาเรื่อง layout ไว้ตั้งแต่แรก) โค้ดที่ "บังเอิญถูก" ตอนนี้อาจพังตอน upgrade compiler
เวอร์ชันใหม่ในอนาคตได้แบบไม่มีการเตือนล่วงหน้าเลย

#### 43.6.2 หมายเหตุ: เมื่อ C library ใช้ `union` — ต้องใช้ `union` ของ Rust พร้อม `#[repr(C)]`

Part 41 เกริ่นไว้สั้น ๆ ว่า `union` ของ Rust (ต่างจาก `enum` ที่ไม่มี tag บอกว่า field ไหน "active" อยู่)
สำคัญมากตอนทำ FFI กับ C library ที่ประกาศ `union` ไว้ตรง ๆ — สถานการณ์นี้พบได้จริงในหลาย C API ระดับต่ำ เช่น
`union` ที่ใช้แทนค่าที่อาจเป็นได้หลาย type แต่ใช้ memory ก้อนเดียวกันเพื่อประหยัดพื้นที่ (ตรงข้ามกับ Rust's
`enum` ที่ต้องมี tag เพิ่มเพื่อบอกว่า variant ไหนถูกใช้อยู่จริง ทำให้กินพื้นที่มากกว่า `union` เสมอ)

สมมติ C library ประกาศ:

```c
typedef union {
    int i;
    float f;
} IntOrFloat;
```

ฝั่ง Rust ต้องประกาศด้วย `union` (ไม่ใช่ `enum`) พร้อม `#[repr(C)]` เพื่อการันตี layout ตรงกัน:

```rust
// #[repr(C)] union: layout ตรงกับ C union (ขนาด = ขนาดของ field ที่ใหญ่ที่สุด, ทุก field
// ใช้ตำแหน่ง memory เดียวกันร่วมกัน) — ตรงกับ C union ที่ประกาศไว้ข้างบน
#[repr(C)]
union IntOrFloat {
    i: i32,
    f: f32,
}

fn main() {
    let mut u = IntOrFloat { i: 42 };
    // การอ่าน field ของ union เป็น unsafe เสมอ เพราะ Rust ไม่มีทางรู้ว่า
    // "field ไหนถูกเขียนล่าสุด" (union ไม่มี tag บอกเหมือน enum ปกติของ Rust)
    println!("อ่านเป็น i32: {}", unsafe { u.i });
    u.f = 3.14;
    println!("อ่านเป็น f32 หลังเขียนใหม่: {}", unsafe { u.f });
    println!("size_of::<IntOrFloat>() = {}", std::mem::size_of::<IntOrFloat>());
}
```

ผลลัพธ์จริง:

```
อ่านเป็น i32: 42
อ่านเป็น f32 หลังเขียนใหม่: 3.14
size_of::<IntOrFloat>() = 4
```

สังเกตว่า `size_of::<IntOrFloat>()` ได้ `4` ไบต์เท่านั้น (ขนาดของ field ที่ใหญ่ที่สุดในบรรดา `i32`/`f32`
ซึ่งเท่ากันพอดีที่ 4 ไบต์) ไม่ใช่ผลรวมของทั้งสอง field เหมือน struct — เพราะ `union` ให้ทุก field
**ใช้ตำแหน่ง memory เดียวกันซ้อนกันเป๊ะ** ตรงกับความหมายของ C union ทุกประการ นี่คือเหตุผลที่การอ่าน field
ของ union ต้องอยู่ใน `unsafe` เสมอไม่มีข้อยกเว้น: **compiler ไม่มีทางรู้ว่า byte pattern ที่เก็บอยู่ตอนนี้
ถูกเขียนมาจาก field ไหนล่าสุด** การอ่านผิด field (เช่นเขียนเป็น `f32` แล้วอ่านกลับเป็น `i32`) จะได้ byte
pattern เดียวกันมาตีความผิดความหมายไปเลย แม้จะไม่ถึงกับ memory-unsafe (ไม่มีการอ่านนอกขอบเขตของ allocation)
แต่ก็เป็น **logic error ที่ compiler ตรวจให้ไม่ได้** จึงต้องครอบด้วย `unsafe` ตามหลักการเดียวกับทุกหัวข้อก่อน
หน้าในบทนี้

### 43.7 เรียก Rust จาก C: `#[no_mangle]` และ `extern "C"` บนฝั่ง Rust

ทุกหัวข้อที่ผ่านมาคือทิศทาง "Rust เรียก C" มาถึงตอนนี้จะกลับทิศ: **ทำอย่างไรให้โค้ด C เรียกฟังก์ชันที่เขียน
ด้วย Rust ได้**

#### 43.7.1 ปัญหาที่ต้องแก้ก่อน: Name Mangling

ทวนจาก Part 17: Rust compile function ทุกตัวโดยเปลี่ยนชื่อภายใน binary ให้เป็นรูปแบบที่เข้ารหัสข้อมูลเพิ่ม
(module path, generic parameter, hash เพื่อป้องกันชื่อชนกัน) กระบวนการนี้เรียกว่า **name mangling** — ฟังก์ชัน
ชื่อ `point_distance` ในซอร์สโค้ด อาจถูกเก็บใน binary จริงเป็นชื่อยาว ๆ แบบ
`_ZN5mylib14point_distance17h4a2f8e9c1b3d5f7aE` (ตัวอย่างรูปแบบ mangled name — รูปแบบจริงขึ้นกับเวอร์ชัน
compiler) ปัญหาคือ **linker ของ C ไม่รู้จักรูปแบบการเข้ารหัสนี้เลย** มันจะมองหาชื่อฟังก์ชันแบบ "plain text"
ตรง ๆ ตามที่เขียนในซอร์สโค้ด C (`point_distance` เฉย ๆ) — ถ้าไม่แก้ไข การ link จะล้มเหลวเพราะหาชื่อฟังก์ชัน
ที่ต้องการไม่เจอ

`#[no_mangle]` คือ attribute ที่บอก compiler ว่า **"ห้ามเข้ารหัสชื่อฟังก์ชันนี้ ให้เก็บชื่อตามที่เขียนตรง ๆ
ใน binary"** — เมื่อรวมกับ `extern "C"` (ที่บอก calling convention ให้ตรงกับที่ C คาดหวัง) ผลคือฟังก์ชัน Rust
ตัวนั้นจะมีหน้าตาเหมือนฟังก์ชัน C ปกติทุกประการจากมุมมองของ linker และโปรแกรม C ที่จะมาเรียกใช้

#### 43.7.2 ตัวอย่างเต็ม: Rust library ที่ C เรียกใช้ได้

ไฟล์ `mylib.rs`:

```rust
// ต้อง repr(C) เหมือนฝั่ง Rust-เรียก-C เป๊ะ เพื่อให้ layout ตรงกับที่โค้ด C คาดหวัง
#[repr(C)]
pub struct Point {
    pub x: f64,
    pub y: f64,
}

// #[no_mangle]: ห้ามเข้ารหัสชื่อฟังก์ชัน ให้ linker หาเจอในชื่อ "point_distance" ตรง ๆ
// extern "C": ใช้ C calling convention เพื่อให้โค้ด C เรียกได้ถูกต้อง
#[no_mangle]
pub extern "C" fn point_distance(a: *const Point, b: *const Point) -> f64 {
    // แม้จะเป็นฟังก์ชันที่ C จะเรียก แต่ภายในยังเป็นโค้ด Rust ปกติทุกอย่าง
    // การ dereference raw pointer ต้องอยู่ใน unsafe เหมือนเดิม (กฎ Part 42 ยังใช้เต็มรูปแบบ)
    unsafe {
        let dx = (*a).x - (*b).x;
        let dy = (*a).y - (*b).y;
        (dx * dx + dy * dy).sqrt()
    }
}
```

Compile เป็น **`cdylib`** (dynamic library ที่ C ABI-compatible — เทียบเท่ากับ `.so` บน Linux, `.dylib` บน
macOS, `.dll` บน Windows — ทวนจาก Part 17 ที่สอนเรื่อง crate-type `lib`/`bin` ตอนนี้ `cdylib` คือ crate-type
ตัวที่สามที่มีไว้เพื่อ FFI โดยเฉพาะ):

```bash
rustc --edition 2021 --crate-type=cdylib mylib.rs -o libmylib.so
```

ฝั่ง C ที่เรียกใช้ ไฟล์ `main_call_rust.c`:

```c
#include <stdio.h>

typedef struct {
    double x;
    double y;
} Point;

// ประกาศ signature ของฟังก์ชันที่มาจาก Rust — เหมือนการเขียน header สำหรับไลบรารีภาษาอื่นทั่วไป
extern double point_distance(const Point *a, const Point *b);

int main(void) {
    Point a = {0.0, 0.0};
    Point b = {3.0, 4.0};
    printf("distance from C calling Rust = %f\n", point_distance(&a, &b));
    return 0;
}
```

Compile และ link เข้ากับ Rust library ที่สร้างไว้:

```bash
gcc main_call_rust.c -L. -lmylib -o main_call_rust
LD_LIBRARY_PATH=. ./main_call_rust
```

ผลลัพธ์จริงจากการรัน:

```
distance from C calling Rust = 5.000000
```

นี่คือหลักฐานที่ยืนยันว่า **ทิศทางทั้งสองของ FFI ทำงานได้จริงด้วยกลไกเดียวกัน**: `#[repr(C)]` ทำให้ layout
ของ struct ตรงกันทั้งสองฝั่ง `extern "C"` ทำให้ calling convention ตรงกัน และ `#[no_mangle]` ทำให้ linker
หาชื่อฟังก์ชันเจอ — ระยะทางที่คำนวณได้ (`5.000000`) ถูกต้องตามสูตร Pythagoras เหมือนกับตัวอย่างในหัวข้อ 43.6
ทุกประการ เพียงแค่กลับทิศทางของผู้เรียกและผู้ถูกเรียก

#### 43.7.3 `cdylib` กับ `staticlib` ต่างกันอย่างไร

Part 17 สอน crate-type `lib` (rlib สำหรับใช้ใน Rust ด้วยกันเอง) และ `bin` (executable) ไปแล้ว สำหรับ FFI
มี crate-type เพิ่มอีกสองตัวที่สำคัญ:

- **`cdylib`**: สร้าง dynamic/shared library (`.so`/`.dylib`/`.dll`) ที่มี C ABI — ใช้เมื่อต้องการให้โปรแกรม
  ภาษาอื่น **load ไลบรารีตอน runtime** (dynamic linking) เหมาะกับกรณีที่ต้องการแจก library แยกจากตัวโปรแกรม
  หรือให้หลายโปรแกรมใช้ library เดียวกันร่วมกันโดยไม่ต้อง copy code ซ้ำ
- **`staticlib`**: สร้าง static library (`.a` บน Linux/macOS, `.lib` บน Windows) ที่ถูก **embed เข้าไปในตัว
  executable ปลายทางตอน compile** (static linking) เหมาะกับกรณีที่ต้องการ binary เดียวที่ไม่ต้องพึ่งไฟล์
  ไลบรารีแยกตอน deploy — ข้อดีคือ deployment ง่ายกว่า (ไฟล์เดียว ไม่ลืม copy `.so` ไปด้วย) แต่ไฟล์ผลลัพธ์จะ
  ใหญ่ขึ้นเพราะ code ของ Rust library ถูกฝังเข้าไปทั้งหมด

```bash
# ตัวอย่างการสร้าง staticlib จาก crate เดียวกัน (คำสั่งที่ทดสอบแล้วว่า compile ผ่านจริง)
rustc --edition 2021 --crate-type=staticlib mylib.rs -o libmylib.a
```

ในโปรเจกต์ Cargo จริง ไม่ต้องเรียก `rustc` มือแบบนี้ — กำหนดผ่าน `Cargo.toml`:

```toml
[lib]
name = "mylib"
crate-type = ["cdylib", "staticlib"]
```

สามารถระบุได้มากกว่าหนึ่ง crate-type พร้อมกัน (Cargo จะ build ทั้งสองรูปแบบให้ในการ compile ครั้งเดียว) —
เลือกใช้ตัวไหนขึ้นอยู่กับว่าผู้ใช้ library ปลายทางต้องการ dynamic หรือ static linking

### 43.8 Linking: `#[link(name = "...")]` และ `build.rs`

เราเห็น `#[link(name = "m")]` ไปแล้วในหัวข้อ 43.2 — นี่คือวิธี**บอก Rust compiler ตรง ๆ ในซอร์สโค้ด**ว่า
ต้อง link กับ system library ตัวไหน วิธีนี้เหมาะกับกรณีง่าย ๆ ที่ library ที่ต้อง link **มีชื่อคงที่และมีอยู่
ในระบบเสมอ** (เช่น `libm` ที่มีอยู่ในทุกระบบ Linux/macOS ที่มี C toolchain)

แต่ในสถานการณ์ที่ซับซ้อนกว่านั้น — เช่นต้อง **compile ไฟล์ C ของตัวเองก่อน** แล้วค่อย link (เหมือนตัวอย่าง
`point.c` ในหัวข้อ 43.6 ที่ตอนนี้เรา compile ด้วยคำสั่งมือ), หรือต้อง **ค้นหาตำแหน่ง library ที่อาจอยู่คนละที่
กันในแต่ละเครื่อง** (เช่น library ที่ติดตั้งผ่าน package manager ที่ path ไม่แน่นอน) — Rust ให้ใช้
**`build.rs`** ทวนจาก Part 35: `build.rs` คือสคริปต์ Rust ธรรมดาที่ Cargo รันให้อัตโนมัติ**ก่อน**ที่จะ compile
crate หลัก มันสามารถพิมพ์คำสั่งพิเศษที่ขึ้นต้นด้วย `cargo:` ออกมาทาง stdout เพื่อสั่ง Cargo ทำสิ่งต่าง ๆ ได้
รวมถึงการบอก linking flag

```rust
// build.rs — รันโดย Cargo อัตโนมัติก่อน compile src/main.rs หรือ src/lib.rs
fn main() {
    // บอก Cargo ว่าต้อง link กับ libm — เทียบเท่ากับ #[link(name = "m")] แต่กำหนดจากภายนอกซอร์สโค้ดหลักได้
    println!("cargo:rustc-link-lib=m");
}
```

ตัวอย่าง `Cargo.toml` และ `src/main.rs` ที่ทำงานคู่กับ `build.rs` ด้านบน (ทดสอบแล้วว่า build และรันได้จริง
ด้วย `cargo build`):

```toml
# Cargo.toml
[package]
name = "ffi_demo"
version = "0.1.0"
edition = "2021"
```

```rust
// src/main.rs
extern "C" {
    fn sqrt(x: f64) -> f64;
}

fn main() {
    println!("sqrt(81.0) = {}", unsafe { sqrt(81.0) });
}
```

ผลลัพธ์จริงจาก `cargo build && ./target/debug/ffi_demo`:

```
sqrt(81.0) = 9
```

สำหรับกรณีที่ซับซ้อนกว่านี้ (compile ไฟล์ `.c` ของตัวเองแล้ว link เข้ามา) โปรเจกต์จริงมักใช้ crate ช่วยชื่อ
**`cc`** เป็น **build dependency** ใน `Cargo.toml`:

```toml
[build-dependencies]
cc = "1"
```

```rust
// build.rs ที่ compile ไฟล์ point.c ให้อัตโนมัติทุกครั้งที่ build แทนการเรียก gcc มือ
fn main() {
    cc::Build::new()
        .file("src/point.c")
        .compile("point"); // จะสร้าง libpoint.a และสั่ง link ให้ Cargo อัตโนมัติ
    println!("cargo:rerun-if-changed=src/point.c");
}
```

`cc::Build` จัดการรายละเอียดของการเรียก C compiler ที่เหมาะกับแต่ละแพลตฟอร์มให้ทั้งหมด (เลือก `gcc`/`clang`
บน Unix, `cl.exe` บน Windows โดยอัตโนมัติ) ทำให้โค้ด build script พกพาข้ามแพลตฟอร์มได้โดยไม่ต้องเขียนเงื่อนไข
`if cfg!(target_os = ...)` เองทั้งหมด — บรรทัด `cargo:rerun-if-changed=src/point.c` บอก Cargo ว่าให้รัน
build script ใหม่เฉพาะเมื่อไฟล์นี้เปลี่ยนแปลง (ไม่ต้อง compile ไฟล์ C ซ้ำทุกครั้งที่ build โดยไม่จำเป็น
ซึ่งช่วยลดเวลา build อย่างมีนัยสำคัญในโปรเจกต์ขนาดใหญ่)

### 43.9 Panic ข้าม FFI boundary: อันตรายที่มองไม่เห็น

หัวข้อนี้เป็นเรื่องความปลอดภัยที่สำคัญมากแต่มักถูกมองข้าม: **ถ้า Rust code ที่ถูกเรียกจาก C เกิด panic และ
panic นั้น unwind (คลี่ stack กลับ) ข้ามผ่าน stack frame ของ C ไปด้วย นี่คือ undefined behavior เต็มรูปแบบ**

เหตุผลเชิงเทคนิค: เมื่อ Rust panic (โดย default, ไม่ใช้ `panic = "abort"`) มันจะ **unwind** — คลาย stack
กลับไปทีละ frame โดยเรียก destructor (`Drop`) ของแต่ละตัวแปรตามลำดับ กระบวนการ unwind นี้อาศัย **ข้อมูล
metadata เฉพาะของ Rust** (เช่น landing pad ที่ compiler generate ไว้สำหรับแต่ละ frame เพื่อรู้ว่าต้อง
เรียก destructor ตัวไหนบ้าง) — **โค้ด C ไม่มี metadata แบบนี้เลย** เพราะ C ไม่มีแนวคิด unwinding แบบ
exception ในตัวภาษา (C++ มี แต่ใช้ metadata รูปแบบของตัวเองที่ไม่ตรงกับของ Rust เว้นแต่จะตั้งค่าให้เข้ากันได้
โดยเฉพาะ) เมื่อ unwind process เดินทางไปถึง stack frame ของ C ที่ไม่มี metadata ที่ต้องการ **มันไม่รู้จะทำ
อย่างไรต่อ** ผลลัพธ์ไม่ได้ถูกกำหนดไว้ (undefined) — อาจ crash ทันที, อาจทำให้ stack เสียหาย, หรืออาจ "ดูเหมือน
ทำงานได้" ในบางกรณีแล้วพังในสถานการณ์อื่นแบบสุ่ม ๆ (คลาสสิกของ undefined behavior)

Rust จัดการป้องกันปัญหานี้ระดับหนึ่งเองอยู่แล้ว: ตั้งแต่ Rust 2018 เป็นต้นมา ถ้า panic unwind ไปถึงขอบของ
`extern "C" fn` (ฟังก์ชันที่ expose ให้ C เรียก) โดย **ไม่มีการดักไว้** runtime จะ**แปลง unwind นั้นเป็น
abort ทันทีที่ขอบ FFI** (แทนที่จะปล่อยให้ unwind ต่อเข้าไปใน stack ของ C จริง ๆ) ซึ่งปลอดภัยกว่า undefined
behavior เต็มรูปแบบมาก แต่ผลลัพธ์ก็คือ**โปรแกรมล่มทันที** ไม่มีโอกาส clean up หรือรายงาน error กลับไปให้ฝั่ง C
ได้อย่างสุภาพ — ยังไม่ใช่ประสบการณ์ที่ดีสำหรับ library ที่ต้องการความน่าเชื่อถือ

มีสองวิธีมาตรฐานในการจัดการเรื่องนี้อย่างถูกต้อง:

**วิธีที่ 1 — ดักจับ panic เองด้วย `std::panic::catch_unwind`:** ครอบทุก `unsafe extern "C" fn` ที่ expose
ให้ C เรียกด้วย `catch_unwind` แล้วแปลง panic เป็น error code ธรรมดาที่ C เข้าใจได้ (เช่น คืน `-1` หรือ error
code พิเศษ) แทนที่จะปล่อยให้ panic ไหลออกไปยัง boundary:

```rust
use std::panic;

fn main() {
    // catch_unwind รับ closure และคืน Result: Ok(T) ถ้าไม่ panic, Err(...) ถ้า panic
    // การดักแบบนี้ "แปลง" panic ให้เป็นค่าปกติที่จัดการต่อได้ แทนที่จะปล่อยให้ unwind
    // ไหลออกไปข้ามขอบ FFI ซึ่งเป็น undefined behavior ตามที่อธิบายไว้ข้างต้น
    let result = panic::catch_unwind(|| {
        println!("about to panic inside catch_unwind");
        panic!("oops, something went wrong");
    });

    match result {
        Ok(_) => println!("no panic occurred"),
        Err(_) => println!("caught a panic safely at the FFI boundary"),
    }
    println!("program continues normally");
}
```

ผลลัพธ์จริงจากการรัน (ข้อความ panic ยังถูกพิมพ์ลง stderr ตามปกติของกลไก panic ของ Rust แต่โปรแกรมไม่ตายและ
ทำงานต่อได้ เพราะ `catch_unwind` ดักการ unwind ไว้ที่จุดนี้แล้ว):

```
about to panic inside catch_unwind
thread 'main' panicked at ...:
oops, something went wrong
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
caught a panic safely at the FFI boundary
program continues normally
```

รูปแบบการใช้งานจริงในฟังก์ชันที่ expose ให้ C เรียก มักหน้าตาแบบนี้:

```rust
use std::panic;

#[no_mangle]
pub extern "C" fn safe_divide(a: f64, b: f64, out: *mut f64) -> i32 {
    let result = panic::catch_unwind(|| {
        if b == 0.0 {
            panic!("division by zero");
        }
        a / b
    });

    match result {
        Ok(value) => {
            unsafe { *out = value };
            0 // 0 = สำเร็จ ตามธรรมเนียม C ทั่วไป
        }
        Err(_) => {
            -1 // -1 = เกิด panic ข้างใน แปลงเป็น error code ที่ C เข้าใจได้แทนการปล่อยให้ unwind
        }
    }
}
```

**วิธีที่ 2 — คอมไพล์ทั้ง crate ด้วย `panic = "abort"`:** ทวนจาก Part 35 ที่เกริ่นการตั้งค่า profile ไว้ —
ถ้าตั้งใน `Cargo.toml`:

```toml
[profile.release]
panic = "abort"
```

การตั้งค่านี้ทำให้ panic **ไม่ unwind เลยแม้แต่ก้าวเดียว** — มันจะ**เรียก `abort()` ทันทีที่จุดที่ panic เกิด
ขึ้น** ไม่มีการคลาย stack กลับใด ๆ ทั้งสิ้น ข้อดีคือรับประกันไม่มีทางเกิด undefined behavior จากการ unwind
ข้าม FFI boundary ได้เลย (เพราะไม่มี unwind ให้ข้ามตั้งแต่แรก) แต่ข้อเสียคือ**เสีย feature `catch_unwind`
ไปด้วย**เพราะมันพึ่งพา unwinding อยู่เช่นกัน (ไม่มี unwind ให้ "catch") — จึงเหมาะกับสถานการณ์ที่ยอมรับได้ว่า
panic ใด ๆ คือจุดจบของโปรแกรมทันที ไม่ต้องการ error recovery ที่ระดับ FFI boundary

หลักการเลือกใช้ที่ควรจำ: ถ้า library ของคุณต้องการความน่าเชื่อถือระดับ production ที่ให้ผู้ใช้ (โค้ด C ฝั่ง
ตรงข้าม) รับมือกับข้อผิดพลาดได้อย่างสุภาพ ให้ใช้ **`catch_unwind` ที่ทุกจุด entry point ของ `extern "C" fn`**
— ถ้าโปรแกรมของคุณยอมรับได้ว่า panic คือความล้มเหลวร้ายแรงที่ต้องหยุดทันที (fail-fast philosophy) การใช้
**`panic = "abort"`** ทั้ง crate ก็เป็นตัวเลือกที่ปลอดภัยและเรียบง่ายกว่า

### 43.10 `bindgen` และ `cbindgen`: อัตโนมัติสิ่งที่เขียนมือได้ยาก

ทุกตัวอย่างในบทนี้เขียน `extern "C"` block ด้วยมือทั้งหมด ซึ่งทำได้ดีสำหรับ C API ขนาดเล็กแค่ไม่กี่ฟังก์ชัน
แต่ลองนึกภาพ C library ขนาดใหญ่จริง ๆ อย่าง OpenSSL หรือ SQLite ที่มี struct ซับซ้อนหลายสิบตัวและฟังก์ชัน
เป็นร้อย — การ**ไล่เขียน `extern "C"` block ด้วยมือทีละฟังก์ชันทีละ struct** มีความเสี่ยงสูงมากที่จะพิมพ์
signature ผิดสักจุดหนึ่ง (ตรงกับอันตรายที่อธิบายไว้ในหัวข้อ 43.3) และเป็นงานที่ซ้ำซ้อนน่าเบื่อโดยไม่จำเป็น
เพราะ**ข้อมูล signature ที่ถูกต้อง 100% มีอยู่แล้วใน `.h` header file ของไลบรารีนั้น**

Rust ecosystem มีเครื่องมือสองตัวที่แก้ปัญหานี้จากสองทิศทาง:

- **[`bindgen`](https://github.com/rust-lang/rust-bindgen)** — อ่านไฟล์ `.h` header ของ C แล้ว **generate
  `extern "C"` block และ `#[repr(C)]` struct ของ Rust ให้อัตโนมัติ** ตรงตาม signature ทุกตัวอักษรที่ประกาศไว้
  ใน header จริง (มันใช้ libclang parse C/C++ syntax จริง ไม่ใช่การเดาด้วย regex) เหมาะกับสถานการณ์
  "**เรียก C library ที่มีอยู่แล้วจาก Rust**" — โปรเจกต์ที่ wrap C library ขนาดใหญ่แทบทุกตัวในวงการ Rust
  (เช่น bindings ของ OpenSSL, libgit2, FFmpeg) ใช้ `bindgen` เป็นส่วนหนึ่งของ build process แทนการเขียน
  binding มือทั้งหมด
- **[`cbindgen`](https://github.com/mozilla/cbindgen)** — ทำงานตรงข้ามกัน: อ่านโค้ด Rust ที่มี `#[repr(C)]`
  struct และ `#[no_mangle] extern "C" fn` แล้ว **generate ไฟล์ `.h` header ของ C ให้อัตโนมัติ** เหมาะกับ
  สถานการณ์ "**expose Rust library ให้โปรแกรม C เรียกใช้**" (ทิศทางเดียวกับหัวข้อ 43.7) ทำให้ผู้ใช้ library
  ฝั่ง C ได้ header ที่ตรงกับ signature จริงของ Rust เสมอ ไม่ต้องเขียน `.h` มือแล้วเสี่ยงพิมพ์ผิดจาก
  signature จริง

ทั้งสองเครื่องมือ**ไม่ได้แทนที่ความเข้าใจพื้นฐานที่เรียนในบทนี้** — มันเป็นเครื่องมือที่ **generate โค้ดที่ยัง
ต้องใช้แนวคิดเดียวกันทั้งหมด** (`extern "C"`, `#[repr(C)]`, `unsafe`) เพียงแต่ generate ให้อัตโนมัติแทนการ
พิมพ์มือ เพื่อลดความเสี่ยงจากบั๊กประเภท "signature ไม่ตรงกัน" ที่อธิบายไว้ในหัวข้อ 43.3 ให้เหลือน้อยที่สุด
เท่าที่จะเป็นไปได้ — ยิ่ง C API มีขนาดใหญ่และซับซ้อนเท่าไหร่ ความคุ้มค่าของการใช้เครื่องมือเหล่านี้ก็ยิ่งสูงขึ้น
เท่านั้น (สำหรับ API เล็ก ๆ ไม่กี่ฟังก์ชันแบบตัวอย่างในบทนี้ การเขียนมือยังเป็นตัวเลือกที่สมเหตุสมผลและเข้าใจ
ง่ายกว่า)

### 43.11 ตัวอย่างสรุปแบบครบวงจร: ระบบแปลงพิกัด GPS ผ่าน FFI

มาปิดท้ายเนื้อหาด้วยตัวอย่างที่รวมทุกแนวคิดของบทนี้ไว้ในโปรเจกต์เดียว จำลองสถานการณ์จริง: บริษัทมีไลบรารี C
เดิม (`geo.c`) ที่ทีม C/C++ เขียนไว้นานแล้วสำหรับคำนวณระยะทางระหว่างพิกัด GPS สองจุด (ใช้สูตร Haversine)
และทีม Rust ต้องการเรียกใช้ไลบรารีนี้จากโปรแกรม Rust ใหม่ โดยไม่อยากเขียนสูตรคำนวณ geo ขึ้นใหม่เอง (สูตร
Haversine มีรายละเอียดปลีกย่อยเรื่องความแม่นยำของ floating point ที่ทีม C ทดสอบมาอย่างละเอียดแล้ว)

ไฟล์ `geo.c` (ไลบรารี C ที่มีอยู่แล้ว, จำลองสถานการณ์ "โค้ด C ที่ใช้งานมาหลายปี"):

```c
#include <math.h>

#define EARTH_RADIUS_KM 6371.0

typedef struct {
    double latitude;
    double longitude;
} GpsPoint;

// สูตร Haversine: คำนวณระยะทางบนพื้นผิวโลก (ทรงกลม) ระหว่างสองจุด GPS
double haversine_distance_km(const GpsPoint *a, const GpsPoint *b) {
    double lat1 = a->latitude * M_PI / 180.0;
    double lat2 = b->latitude * M_PI / 180.0;
    double dlat = (b->latitude - a->latitude) * M_PI / 180.0;
    double dlon = (b->longitude - a->longitude) * M_PI / 180.0;

    double sin_dlat = sin(dlat / 2.0);
    double sin_dlon = sin(dlon / 2.0);

    double h = sin_dlat * sin_dlat
             + cos(lat1) * cos(lat2) * sin_dlon * sin_dlon;

    double c = 2.0 * atan2(sqrt(h), sqrt(1.0 - h));
    return EARTH_RADIUS_KM * c;
}
```

ไฟล์ `main.rs` ฝั่ง Rust ที่เรียกใช้ไลบรารี C นี้:

```rust
use std::os::raw::c_double;

// ต้องตรง layout กับ GpsPoint ในไฟล์ geo.c เป๊ะ: latitude มาก่อน longitude ทั้งคู่เป็น double
#[repr(C)]
struct GpsPoint {
    latitude: c_double,
    longitude: c_double,
}

extern "C" {
    fn haversine_distance_km(a: *const GpsPoint, b: *const GpsPoint) -> c_double;
}

fn main() {
    // กรุงเทพฯ (13.7563 N, 100.5018 E) และเชียงใหม่ (18.7883 N, 98.9853 E)
    let bangkok = GpsPoint {
        latitude: 13.7563,
        longitude: 100.5018,
    };
    let chiang_mai = GpsPoint {
        latitude: 18.7883,
        longitude: 98.9853,
    };

    let distance = unsafe { haversine_distance_km(&bangkok, &chiang_mai) };
    println!("ระยะทางกรุงเทพฯ-เชียงใหม่ (ประมาณ) = {:.2} กม.", distance);
}
```

Compile และรัน:

```bash
gcc -c geo.c -o geo.o
rustc --edition 2021 main.rs -C link-arg=geo.o -C link-arg=-lm -o geo_demo
./geo_demo
```

ผลลัพธ์จริงจากการรัน:

```
ระยะทางกรุงเทพฯ-เชียงใหม่ (ประมาณ) = 582.46 กม.
```

ตัวเลขนี้ใกล้เคียงกับระยะทางตรง (great-circle distance) จริงระหว่างสองเมืองที่วัดได้จากแหล่งข้อมูลภูมิศาสตร์
ทั่วไป (ประมาณ 580–590 กม.) ยืนยันว่าสูตร Haversine ในไลบรารี C ทำงานถูกต้อง และการเชื่อมต่อ FFI ทั้งสามเสา
หลัก (`extern "C"` ประกาศ signature ถูกต้อง, `#[repr(C)]` ทำให้ layout ของ `GpsPoint` ตรงกันทั้งสองฝั่ง,
`unsafe` ครอบจุดที่เรียกจริง) ทำงานร่วมกันได้อย่างสมบูรณ์

ตอนนี้มาดูทิศทางกลับ — สมมติทีม Rust ปรับปรุงไลบรารีนี้เพิ่มฟังก์ชันคำนวณ "จุดกึ่งกลาง" (midpoint) ด้วย Rust
เอง (เพราะทีมมองว่า Rust ปลอดภัยกว่าสำหรับ logic ใหม่ ๆ ที่จะเพิ่มต่อจากนี้) แล้ว**expose ให้โค้ด C เดิมเรียก
กลับได้** โดยไม่ต้อง rewrite ทุกอย่าง:

```rust
// midpoint.rs — ฟังก์ชันใหม่เขียนด้วย Rust แต่ expose ให้เรียกจาก C ได้เหมือนฟังก์ชัน C ปกติ
#[repr(C)]
pub struct GpsPoint {
    pub latitude: f64,
    pub longitude: f64,
}

#[no_mangle]
pub extern "C" fn simple_midpoint(a: *const GpsPoint, b: *const GpsPoint) -> GpsPoint {
    unsafe {
        GpsPoint {
            latitude: ((*a).latitude + (*b).latitude) / 2.0,
            longitude: ((*a).longitude + (*b).longitude) / 2.0,
        }
    }
}
```

(หมายเหตุ: การคืนค่าเป็น struct ตรง ๆ แบบนี้ใช้ได้เพราะ `#[repr(C)]` การันตี layout ที่ C เข้าใจได้ — ฟังก์ชัน
C ฝั่งตรงข้ามจะรับค่าคืนตาม calling convention ของการคืน struct แบบ C ทุกประการ) Compile ด้วย
`rustc --edition 2021 --crate-type=cdylib midpoint.rs -o libmidpoint.so` แล้ว C ก็ประกาศและเรียกใช้ได้
เหมือนตัวอย่างในหัวข้อ 43.7.2 ทุกประการ — นี่คือภาพรวมที่แสดงให้เห็นว่าในโปรเจกต์จริง **ทั้งสองทิศทางของ FFI
มักถูกใช้ผสมกัน** ในระบบเดียวกัน: ใช้ของ C ที่มีอยู่แล้วในส่วนที่ทำงานได้ดีอยู่แล้ว และเขียน Rust ใหม่ในส่วนที่
ต้องการความปลอดภัยเพิ่มเติมหรือ logic ใหม่ ๆ โดยไม่ต้อง rewrite ทั้งระบบในครั้งเดียว

### 43.12 Function pointer และ Callback: ส่งฟังก์ชัน Rust ให้ C เรียกกลับ

หลายฟังก์ชันใน C ไม่ได้แค่รับข้อมูลธรรมดา แต่รับ **function pointer** เป็น argument เพื่อให้ตัวมันเอง
"เรียกกลับ" (callback) เข้ามาตอนทำงาน ตัวอย่างคลาสสิกที่สุดคือ `qsort()` จาก `<stdlib.h>` — ฟังก์ชัน sort
ทั่วไปที่ไม่ผูกกับ type ข้อมูลตายตัว (คล้าย generic function) โดยรับ **comparator function** เข้ามาเป็น
พารามิเตอร์ให้ผู้เรียกกำหนดกฎการเรียงเอง

Signature จริงของ `qsort` ในภาษา C คือ:

```c
void qsort(void *base, size_t nmemb, size_t size,
           int (*compar)(const void *, const void *));
```

การแมป function pointer type นี้ไปเป็น Rust ใช้ syntax `Option<unsafe extern "C" fn(...) -> ...>` —
สังเกตว่าครอบด้วย `Option` เพราะ function pointer ของ C **มีสถานะ `NULL` ได้** (สื่อความหมาย "ไม่มี callback"
ในบาง API) และ Rust แมป `NULL` function pointer เป็น `None` ได้อย่างเป็นธรรมชาติผ่าน layout optimization
พิเศษที่ garantee ว่า `Option<fn(...)>` มีขนาดเท่ากับ pointer เปล่า ๆ พอดี (ไม่มี overhead เพิ่มจากการห่อ
`Option`)

```rust
use std::os::raw::{c_int, c_void};

extern "C" {
    fn qsort(
        base: *mut c_void,
        nmemb: usize,
        size: usize,
        compar: Option<unsafe extern "C" fn(*const c_void, *const c_void) -> c_int>,
    );
}

// ฟังก์ชัน comparator ต้องเป็น extern "C" เพราะ qsort (ซึ่งเขียนด้วย C) จะเรียกกลับมาที่นี่
// ด้วย C calling convention — ถ้าไม่ประกาศ extern "C" ตัว calling convention จะไม่ตรงกัน
// และผลลัพธ์จะเป็น undefined behavior ทันทีที่ qsort พยายามเรียกฟังก์ชันนี้กลับมา
unsafe extern "C" fn compare_i32(a: *const c_void, b: *const c_void) -> c_int {
    let a = *(a as *const i32);
    let b = *(b as *const i32);
    (a - b) as c_int
}

fn main() {
    let mut numbers: Vec<i32> = vec![5, 2, 8, 1, 9, 3];
    let len = numbers.len();
    unsafe {
        qsort(
            numbers.as_mut_ptr() as *mut c_void,
            len,
            std::mem::size_of::<i32>(),
            Some(compare_i32),
        );
    }
    println!("sorted: {:?}", numbers);
}
```

ผลลัพธ์จริง:

```
sorted: [1, 2, 3, 5, 8, 9]
```

จุดที่ควรสังเกตเป็นพิเศษคือ **`compare_i32` ต้องประกาศเป็น `unsafe extern "C" fn` (function definition
ธรรมดา ไม่ใช่ closure)** — Rust closure ปกติ (`|a, b| ...`) **ไม่สามารถแปลงเป็น C function pointer ได้
โดยตรง** ถ้ามันมีการ capture ตัวแปรจาก environment ภายนอก เพราะ C function pointer เป็นแค่ที่อยู่ของ
instruction ล้วน ๆ (ไม่มีที่เก็บ "ข้อมูลที่ capture มา" เหมือน closure ของ Rust ที่จริง ๆ แล้วเป็น struct
ซ่อนที่มี field เก็บตัวแปรที่ capture ไว้) มีแต่ **ฟังก์ชันที่ไม่ capture อะไรเลย** (plain `fn`, ไม่ใช่
closure ที่มี environment) เท่านั้นที่แปลงเป็น function pointer ได้ตรง ๆ แบบ zero-cost — ถ้าต้องการส่ง "ข้อมูล
เพิ่มเติม" เข้าไปในcallback จริง ๆ API ของ C มักมี parameter พิเศษชื่อ `void *user_data` แถมมาให้เสมอ
(ไม่มีใน `qsort` แต่มีใน API สมัยใหม่จำนวนมาก) ให้ส่ง pointer ไปยังข้อมูลของคุณเองผ่านช่องนี้แทนการพึ่งพา
closure capture

### 43.13 ส่ง array/slice ข้าม FFI: pointer + length เสมอ

ปัญหาเดียวกันกับ `&str`/`String` ในหัวข้อ 43.5 เกิดกับ `&[T]`/`Vec<T>` เช่นกัน: **slice ของ Rust เป็น fat
pointer ที่มี pointer และ length รวมกัน แต่ C ไม่รู้จักแนวคิดนี้เลย** — array ในภาษา C เป็นแค่ pointer ไปยัง
ข้อมูลตัวแรก ไม่มี lengthติดมาด้วย (ต่างจาก null-terminated string ตรงที่ array เชิงตัวเลขทั่วไปไม่มี "ค่าจบ"
ที่ตายตัวแบบ `\0` — จึง**ต้องส่ง length เป็น argument แยกเสมอ**ไม่มีทางเลี่ยง)

```c
#include <stddef.h>

double sum_array(const double *arr, size_t len) {
    double total = 0.0;
    for (size_t i = 0; i < len; i++) {
        total += arr[i];
    }
    return total;
}
```

```rust
extern "C" {
    fn sum_array(arr: *const f64, len: usize) -> f64;
}

fn main() {
    let data: Vec<f64> = vec![1.0, 2.0, 3.0, 4.0, 5.0];
    // แยก pointer และ length ออกจากกันด้วยมือ — as_ptr() ให้ pointer ไปยังข้อมูลตัวแรก
    // len() ให้จำนวน element ทั้งสองต้องส่งไปพร้อมกันเสมอเมื่อข้าม FFI boundary
    let sum = unsafe { sum_array(data.as_ptr(), data.len()) };
    println!("sum = {}", sum);
}
```

ผลลัพธ์จริง:

```
sum = 15
```

หลักการนี้ใช้เหมือนกันไม่ว่าจะเป็น array ของ `f64`, `i32`, struct ที่เป็น `#[repr(C)]`, หรืออะไรก็ตาม —
**เมื่อข้าม FFI boundary ไม่มี "fat pointer" หรือ "slice" อีกต่อไป มีแต่ raw pointer เปล่า ๆ กับ length ที่
ต้องส่งควบคู่กันด้วยความรับผิดชอบของโปรแกรมเมอร์เอง 100%** สิ่งที่ต้องระวังคือ **`data` (เจ้าของ buffer)
ต้องมีชีวิตอยู่ตลอดช่วงที่ C กำลังใช้ pointer นั้น** — ตรงกับหลักการ lifetime เดียวกับ `CString` ในกับดักที่ 2

### 43.14 Safe Wrapper Pattern: ห่อ C API ที่เป็น `unsafe` ทั้งหมดให้ปลอดภัยด้วย RAII

หัวข้อสุดท้ายก่อนกับดักคือ pattern ที่โปรเจกต์ FFI ระดับ production แทบทุกตัวใช้: **ห่อฟังก์ชัน `extern "C"`
ที่เป็น `unsafe` ทั้งหมดไว้ข้างในของ struct Rust ที่ปลอดภัย** เพื่อให้ผู้ใช้ crate ของคุณไม่ต้องเห็น `unsafe`
เลยแม้แต่คำเดียว (crate อย่าง `rusqlite`, `openssl` ทำแบบนี้ทั้งหมด — ผู้ใช้เขียนโค้ด Rust ปลอดภัย 100%
ทั้งที่ภายในเรียก C library ผ่าน FFI อยู่)

ทวนจาก Part 27–28: `Drop` trait ทำให้ค่าที่ออกจาก scope ถูก "เก็บกวาด" อัตโนมัติเสมอ ไม่ว่าจะออกจาก scope
ด้วยการ `return` ปกติหรือด้วย `?` operator ที่ propagate error ออกไปกลางทาง — หลักการเดียวกันนี้ใช้ปิด
"resource" ที่มาจาก C ได้อย่างสมบูรณ์แบบ: เปิด resource ตอนสร้าง (`new`/`open`) และปิดมันอัตโนมัติใน
`impl Drop` แนวคิดนี้เรียกว่า **RAII (Resource Acquisition Is Initialization)** ซึ่งมาจากโลก C++ แต่ Rust
นำมาใช้เป็นรากฐานสำคัญของ ownership ทั้งระบบ (`Box<T>`, `Vec<T>`, `File` ทุกตัวใช้หลักการนี้อยู่แล้ว)

สมมติมี C library ที่จัดการ "resource" ผ่าน opaque handle (คือ pointer ที่ไม่ต้องรู้ว่าโครงสร้างข้างในเป็น
อย่างไร แค่ถือ pointer ไว้ส่งต่อให้ฟังก์ชันอื่นเรียกใช้):

```c
typedef struct {
    char name[64];
    int open_count;
} Resource;

Resource *resource_open(const char *name) { /* ... malloc + เก็บชื่อ ... */ }
void resource_close(Resource *r) { /* ... free ... */ }
const char *resource_name(const Resource *r) { /* ... คืน pointer ไปยัง field name ... */ }
```

ฝั่ง Rust ห่อ handle นี้ด้วย struct ที่ implement `Drop`:

```rust
use std::ffi::{CStr, CString};
use std::os::raw::c_char;

// opaque type: Rust ไม่รู้และไม่จำเป็นต้องรู้ layout ภายในจริงของ Resource ฝั่ง C เลย
// แค่ต้องมี pointer ไปยังมันเพื่อส่งต่อให้ฟังก์ชัน C อื่น ๆ ใช้งานเท่านั้น
#[repr(C)]
struct CResource {
    _private: [u8; 0],
}

extern "C" {
    fn resource_open(name: *const c_char) -> *mut CResource;
    fn resource_close(r: *mut CResource);
    fn resource_name(r: *const CResource) -> *const c_char;
}

/// Safe wrapper รอบ handle ของ C — ผู้ใช้ type นี้ไม่ต้องยุ่งกับ unsafe หรือจำเรียก close เองเลย
pub struct ManagedResource {
    ptr: *mut CResource,
}

impl ManagedResource {
    pub fn open(name: &str) -> Option<Self> {
        let c_name = CString::new(name).ok()?;
        let ptr = unsafe { resource_open(c_name.as_ptr()) };
        if ptr.is_null() {
            None
        } else {
            Some(ManagedResource { ptr })
        }
    }

    pub fn name(&self) -> String {
        let c_str = unsafe { CStr::from_ptr(resource_name(self.ptr)) };
        c_str.to_string_lossy().into_owned()
    }
}

// Drop คือกลไกที่ทำให้ resource ฝั่ง C ถูกปิดอัตโนมัติเสมอ ไม่ว่า ManagedResource
// จะออกจาก scope ทางไหน (return ปกติ, panic ที่ถูก catch, ฯลฯ) — ผู้ใช้ไม่มีทางลืมปิดเอง
// เหมือนที่ต้องคอยเรียก resource_close() เองด้วยมือทุกครั้งในโค้ด C ดั้งเดิม
impl Drop for ManagedResource {
    fn drop(&mut self) {
        // ต้องอ่านชื่อ "ก่อน" ปิด handle เสมอ — ถ้าอ่านหลัง resource_close จะกลายเป็น
        // use-after-free ทันที (หน่วยความจำถูก free ไปแล้ว การอ่านต่อคือ undefined behavior)
        let name = self.name();
        unsafe { resource_close(self.ptr) };
        println!("(ปิด resource '{}' อัตโนมัติแล้ว)", name);
    }
}

fn main() {
    {
        let res = ManagedResource::open("database-connection").unwrap();
        println!("เปิด resource ชื่อ: {}", res.name());
    } // res ออกจาก scope ที่นี่ -> Drop::drop ถูกเรียกอัตโนมัติ -> resource_close ถูกเรียกให้
    println!("จบโปรแกรม");
}
```

ผลลัพธ์จริงจากการรัน:

```
เปิด resource ชื่อ: database-connection
(ปิด resource 'database-connection' อัตโนมัติแล้ว)
จบโปรแกรม
```

สังเกตว่า `main()` **ไม่มี `unsafe` โผล่มาให้เห็นเลยแม้แต่คำเดียว** ทั้งที่ข้างใน `ManagedResource` เต็มไปด้วย
raw pointer และการเรียก `extern "C"` — นี่คือคุณค่าที่แท้จริงของการเขียน FFI binding ที่ดี: **ผู้ใช้ crate
ไม่จำเป็นต้องรับผิดชอบความปลอดภัยของ FFI เอง เพราะคนเขียน wrapper (คุณ) ได้ตรวจสอบและรับผิดชอบมันไว้ให้แล้ว
ตรงจุดเดียวที่จำกัดและตรวจสอบได้ง่ายที่สุด** — field `ptr` เป็น `private` (ไม่มี `pub` นำหน้า) จึงไม่มีทางที่
โค้ดนอก module นี้จะไปแก้ไขหรือใช้ raw pointer นั้นตรง ๆ ได้เลย ทุกการเข้าถึงต้องผ่าน method ที่ตรวจสอบความ
ปลอดภัยไว้แล้วเท่านั้น — หลักการ **"encapsulate unsafety behind a safe API"** นี้คือมาตรฐานทองของการเขียน
FFI binding ในโลก Rust ทั้งหมด

## กับดักที่พบบ่อย (Common Pitfalls)

### กับดักที่ 1: Signature mismatch — compiler ไม่มีทางจับได้ แต่ผลลัพธ์ผิดแบบเงียบ ๆ

นี่คือกับดักที่อันตรายที่สุดของ FFI ทั้งหมด เพราะ**ไม่มี error หรือ warning ตอน compile เลย** มาดูหลักฐาน
จริงจากการทดสอบ: ถ้าประกาศ `abs()` (จริง ๆ คือ `int abs(int)`) ผิดเป็น `c_long` (64-bit บน Linux):

```rust
use std::os::raw::c_long;

// ผิด! abs() จริงคือ `int abs(int)` ไม่ใช่ `long abs(long)`
extern "C" {
    fn abs(x: c_long) -> c_long;
}

fn main() {
    let x: c_long = -1;
    println!("abs({}) = {}", x, unsafe { abs(x) });

    let y: c_long = -300000000000i64;
    println!("abs({}) = {}", y, unsafe { abs(y) });
}
```

ผลลัพธ์จริง:

```
abs(-1) = 1
abs(-300000000000) = 647710720
```

สังเกตว่า `abs(-1)` ยัง "ดูถูก" อยู่ (ผลลัพธ์ `1` เพราะบังเอิญ ABI ของแพลตฟอร์มนี้ทำ sign/zero-extension แบบที่
ทำให้ค่าเล็ก ๆ ดูถูกต้อง) แต่ `abs(-300000000000)` ที่เป็นค่าเกินขอบเขตของ 32-bit `int` กลับได้ผลลัพธ์
`647710720` ซึ่ง**ผิดโดยสิ้นเชิง** เพราะ C มองเห็นแค่ 32-bit ล่างของค่าที่ส่งมา (ตัด/ตีความบิตผิดไปหมด) —
นี่คือความอันตรายที่แท้จริง: **บั๊กแบบนี้อาจ "ดูเหมือนใช้งานได้" ในการทดสอบด้วยค่าเล็ก ๆ แล้วพังในโปรดักชันเมื่อ
เจอค่าที่ต่างออกไป** ทางแก้เดียวคือ**ตรวจสอบ signature กับเอกสารหรือ header file จริงเสมอ** ไม่เดาจากความรู้
สึกว่า "น่าจะเป็น type นี้" — และใช้ `std::os::raw::c_int`/`c_long` ตามที่ระบุในหัวข้อ 43.4 แทนการเขียน
`i32`/`i64` เอง เพื่อให้ขนาดตรงกับที่แพลตฟอร์มกำหนดจริง

### กับดักที่ 2: ลืมเรื่อง lifetime ของ `CString` — dangling pointer แบบเงียบ ๆ

`CString::as_ptr()` คืน raw pointer ที่ **ใช้ได้แค่ตราบที่ `CString` ต้นฉบับยังไม่ถูก drop** — เพราะ raw
pointer ไม่ผูกกับ borrow checker แบบ reference ปกติ compiler จะไม่เตือนถ้าคุณเก็บ pointer ไว้ใช้ต่อหลังจาก
`CString` ถูก drop ไปแล้ว:

```rust
use std::ffi::CString;
use std::os::raw::c_char;

fn get_dangling_ptr() -> *const c_char {
    let s = CString::new("temporary").unwrap();
    s.as_ptr()
    // s ถูก drop ทันทีที่ฟังก์ชันนี้จบ — buffer ถูกปล่อยคืนหน่วยความจำ
    // pointer ที่ return ออกไปกลายเป็น dangling pointer ทันที
}
```

โค้ดนี้ **compile ผ่านได้ปกติโดยไม่มี warning** ทางแก้คือต้องเก็บตัวแปร `CString` ให้มีชีวิตอยู่ตลอดช่วงที่
pointer ยังถูกใช้งาน (ดูตัวอย่างที่ถูกต้องในหัวข้อ 43.5.1) — หลักการทั่วไป: **ทุกครั้งที่เห็น `.as_ptr()`
ถูกเรียกจาก `CString`, `Vec<T>`, หรือ owned type อื่น ๆ ให้ตรวจสอบเสมอว่าตัวแปรต้นฉบับ (owner) จะมีชีวิตอยู่
นานพอสำหรับช่วงที่ pointer ถูกใช้งานจริง**

### กับดักที่ 3: Panic ที่ unwind ข้าม FFI boundary — undefined behavior เต็มรูปแบบ

ถ้าเขียนฟังก์ชัน `#[no_mangle] extern "C" fn` ที่มี panic เกิดขึ้นได้ (เช่น index out of bounds, unwrap บน
`None`, หรือ integer overflow ใน debug mode) แล้ว**ไม่ได้ดักด้วย `catch_unwind`** panic นั้นจะพยายาม unwind
ออกไปยัง stack frame ของ C ที่เรียกมันมา — ตามที่อธิบายในหัวข้อ 43.9 นี่คือ undefined behavior (แม้ Rust
สมัยใหม่จะแปลงเป็น abort ทันทีที่ขอบ FFI แทนการปล่อยให้ unwind ต่อไปจริง ๆ แต่ก็ยังหมายถึงโปรแกรมล่มแบบ
ควบคุมไม่ได้ ไม่มีโอกาสรายงาน error กลับไปให้ฝั่ง C จัดการอย่างสุภาพ):

```rust
// อันตราย! ไม่มีการป้องกัน panic ที่อาจเกิดจาก .unwrap()
#[no_mangle]
pub extern "C" fn risky_divide(a: f64, b: f64) -> f64 {
    if b == 0.0 {
        panic!("division by zero"); // panic นี้จะพยายาม unwind ข้าม FFI boundary!
    }
    a / b
}
```

ทางแก้คือครอบด้วย `catch_unwind` เสมอที่ทุก entry point ที่ expose ให้ C เรียก (ดูตัวอย่างที่ถูกต้องใน
หัวข้อ 43.9) หรือตั้งค่าทั้ง crate เป็น `panic = "abort"` ถ้ายอมรับได้ว่า panic ควรทำให้โปรแกรมหยุดทันที

### กับดักที่ 4: ส่ง struct แบบ `#[repr(Rust)]` (default) ข้าม FFI — ข้อมูลผิดแบบไม่มีทางรู้ตอน compile

นี่คือกับดักที่อันตรายที่สุดในบรรดา struct-related bug เพราะแม้จะใส่ `#[repr(C)]` ถูกต้องทั้งสองฝั่งแล้ว
**การเรียงลำดับ field ผิด (ไม่ตรงกับ C struct จริง) ก็ยังทำให้พังได้เหมือนกัน** มาดูหลักฐานจริงที่ทดสอบแล้ว:
สมมติ C struct จริงคือ

```c
typedef struct {
    int id;
    double value;
} Record;

double weighted_value(const Record *r, double weight) {
    return r->value * weight;
}
```

ถ้าฝั่ง Rust เขียน field สลับลำดับผิด (ทั้งที่ใส่ `#[repr(C)]` ถูกต้องแล้ว):

```rust
#[repr(C)]
struct WrongRecord {
    value: f64, // ผิด! ของจริงใน C คือ id (int) มาก่อน value (double)
    id: i32,
}

extern "C" {
    fn weighted_value(r: *const WrongRecord, weight: f64) -> f64;
}

fn main() {
    let r = WrongRecord { value: 10.0, id: 7 };
    let result = unsafe { weighted_value(&r, 2.0) };
    println!("weighted_value (ลำดับ field ผิด) = {}", result);
}
```

ผลลัพธ์จริงจากการรัน (เทียบกับเวอร์ชันที่ถูกต้องซึ่งได้ `20`):

```
weighted_value (ลำดับ field ถูกต้อง) = 20
weighted_value (ลำดับ field ผิด) = 0.0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000138396565486762
```

ผลลัพธ์ที่ได้คือค่า floating point ที่ผิดเพี้ยนอย่างรุนแรง (เกิดจากการที่โค้ด C อ่าน byte ที่ตำแหน่งของ
field `id` (แค่ 4 byte จาก `int`) ไปตีความเป็นส่วนหนึ่งของ `double` 8 byte ผสมกับ padding และ byte อื่น ๆ
ที่ไม่เกี่ยวข้องกันเลย) **ทั้งสอง struct ใช้ `#[repr(C)]` ถูกต้องทั้งคู่** — ปัญหาไม่ได้อยู่ที่การลืมใส่
attribute แต่อยู่ที่**ลำดับการเขียน field ต้องตรงกับ C struct ต้นฉบับเป๊ะทุกตัว ไม่ใช่แค่ type ตรงกัน**
บทเรียนคือ: `#[repr(C)]` การันตีว่า **"field จะถูกจัดเรียงตามลำดับที่คุณเขียนไว้ในโค้ด Rust"** เท่านั้น —
มันไม่การันตีว่าลำดับที่คุณเขียนนั้น**ตรงกับของจริงในภาษา C** ความถูกต้องของลำดับยังเป็นความรับผิดชอบของคุณ
เอง 100% (นี่คืออีกเหตุผลที่ `bindgen` ในหัวข้อ 43.10 มีค่ามาก — มันคัดลอกลำดับ field จาก header จริงมาให้
ตรง ๆ ไม่มีโอกาสพิมพ์ผิด)

### กับดักที่ 5 (bonus): ความเป็นเจ้าของหน่วยความจำข้าม allocator — ใครสร้าง ใครต้องปล่อย

กับดักสุดท้ายนี้ไม่ได้อยู่ในโค้ดตัวอย่างของบทนี้ตรง ๆ แต่เป็นเรื่องที่พบบ่อยมากในโปรเจกต์ FFI จริง: **ถ้า
ฟังก์ชัน C จอง memory ให้คุณด้วย `malloc()` (เช่นคืน `char *` ที่ allocate มาใหม่ ไม่ใช่ string literal
คงที่) memory ก้อนนั้นถูกจองด้วย **allocator ของ C** — ต้องปล่อยคืนด้วย `free()` ของ C เท่านั้น ห้ามปล่อย
ด้วยกลไก drop ปกติของ Rust (`Box`, `Vec`, `String`) เด็ดขาด** เพราะ **allocator ของ Rust และ allocator ของ
C เป็นระบบจัดการหน่วยความจำที่แยกจากกันโดยสิ้นเชิง** (แม้บนหลายแพลตฟอร์ม Rust's default allocator จะ
เรียก `malloc`/`free` ของระบบอยู่ข้างในเหมือนกัน แต่นี่คือ**รายละเอียดการ implement ที่ไม่มีการันตีตามสัญญา
ของภาษา**เลย — Rust อาจเปลี่ยนไปใช้ allocator อื่นได้ทุกเมื่อ เช่น `jemalloc`, `mimalloc` ที่คนนิยมสับเปลี่ยน
กันในโปรเจกต์ production จำนวนมาก) ถ้าคุณเผลอสร้าง `String`/`Box<[u8]>` จาก raw pointer ที่ C `malloc`
มาให้ (ผ่าน method อย่าง `Box::from_raw` หรือ `String::from_raw_parts`) แล้วปล่อยให้ Rust `drop` มันตามปกติ
Rust จะเรียก **deallocator ของ Rust เอง** ไปพยายามปล่อยคืน memory ที่จองมาจาก **allocator คนละตัว** —
ผลลัพธ์ไม่ได้ถูกกำหนดไว้ (undefined behavior) อาจ crash ทันที หรือทำให้ heap เสียหายแบบเงียบ ๆ ที่ตรวจจับได้
ยากมาก

หลักการที่ต้องจำ: **ตรวจสอบเอกสารของ C library เสมอว่าฟังก์ชันที่คืน pointer มาให้นั้น "ใครเป็นเจ้าของ
memory หลังจากนั้น"** — ถ้า C เป็นเจ้าของ (คุณแค่ยืมดู) ห้ามพยายามปล่อยคืนจากฝั่ง Rust เด็ดขาด ให้ copy
ข้อมูลออกมาเป็น owned type ของ Rust เอง (เช่น `CStr::to_string_lossy().into_owned()` ที่เห็นในหัวข้อ
43.14 — มันสร้าง `String` ใหม่ที่ Rust เป็นเจ้าของ 100% แยกจาก buffer เดิมของ C โดยสิ้นเชิง) ถ้า C
"ยกกรรมสิทธิ์" ให้ Rust เป็นเจ้าของ (บางเอกสาร library จะเขียนชัดว่า "caller must free the returned
pointer") ให้เรียก**ฟังก์ชัน `free` ของ C library นั้นตรง ๆ** (ผ่าน `extern "C"` เหมือนฟังก์ชันอื่น ๆ) เมื่อ
ใช้งานเสร็จแล้ว ไม่ใช่ปล่อยให้ Rust's `Drop` จัดการเอง

```rust
use std::ffi::CStr;
use std::os::raw::c_char;

extern "C" {
    fn getenv(name: *const c_char) -> *mut c_char;
}

fn main() {
    let key = std::ffi::CString::new("HOME").unwrap();
    let val_ptr = unsafe { getenv(key.as_ptr()) };

    if !val_ptr.is_null() {
        // getenv คืน pointer ที่ "C เป็นเจ้าของ" (ชี้ไปยัง environment block ภายในของ libc เอง
        // ไม่ใช่ memory ที่ malloc มาใหม่ให้เรา) เอกสารของ getenv ระบุชัดว่าห้ามแก้ไขหรือ free
        // pointer นี้เด็ดขาด — วิธีที่ปลอดภัยคือ "ยืมดู" ผ่าน CStr แล้ว copy ข้อมูลออกมาเป็น
        // String ของ Rust เอง (to_string_lossy().into_owned()) จากนั้นก็ไม่ต้องแตะ val_ptr อีกเลย
        let owned: String = unsafe { CStr::from_ptr(val_ptr) }
            .to_string_lossy()
            .into_owned();
        println!("HOME (copied เป็น owned String ของ Rust แล้ว) = {}", owned);
        // ไม่มีการเรียก free(val_ptr) ที่นี่ — เพราะ Rust ไม่ได้เป็นเจ้าของ memory นั้นตั้งแต่แรก
    }
}
```

โค้ดนี้คือรูปแบบที่ปลอดภัยที่สุดสำหรับกรณี "แค่ยืมดู string จาก C": **copy ข้อมูลออกมาเป็น owned type ของ
Rust ทันทีที่เป็นไปได้ แล้วเลิกยุ่งกับ raw pointer เดิม** เพื่อไม่ให้ต้องมาคอยจำว่า pointer นั้น "ใครเป็น
เจ้าของ" ไปตลอดชีวิตของโปรแกรม — เมื่อข้อมูลกลายเป็น `String` ของ Rust แล้ว การจัดการ lifetime และ
memory ที่เหลือทั้งหมดกลับไปเป็นไปตามกฎ ownership ปกติของ Rust ที่ borrow checker ตรวจสอบให้ได้เต็มรูปแบบ
เหมือนโค้ด Rust ทั่วไปที่ไม่มี FFI เข้ามาเกี่ยวข้องเลย

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เขียนโปรแกรม Rust ที่เรียกฟังก์ชัน `labs()` จาก `<stdlib.h>` (คือ `long labs(long)` — เวอร์ชัน
   `abs()` สำหรับ `long`) ให้ประกาศ `extern "C"` block โดยใช้ type ที่ถูกต้องจาก `std::os::raw` แล้วทดสอบ
   เรียกด้วยค่าลบสองสามค่า พร้อม `println!` ผลลัพธ์
   - Hint: ต้องใช้ `c_long` ไม่ใช่ `c_int` เพราะ `labs` รับ/คืน `long` ไม่ใช่ `int` — สังเกตว่าถ้าใช้ `c_int`
     ผิด โปรแกรมอาจ compile ผ่านได้เหมือนกัน (ตามอันตรายในกับดักที่ 1) ลองทดสอบทั้งสองแบบเทียบกันดู

2. **(กลาง)** เขียนฟังก์ชัน C ชื่อ `rect_area` ที่รับ struct `Rect { double width; double height; }` ผ่าน
   pointer แล้วคืนพื้นที่ (`width * height`) จากนั้นเขียนฝั่ง Rust ที่มี `#[repr(C)] struct Rect` ตรงกัน
   ประกาศ `extern "C"` และเรียกใช้จริง พร้อม compile ทั้งสองไฟล์เข้าด้วยกันและยืนยันผลลัพธ์ถูกต้องด้วยค่าที่
   คำนวณมือได้ล่วงหน้า (เช่น width=4.0, height=5.0 ต้องได้ 20.0)
   - Hint: ใช้คำสั่ง `gcc -c rect.c -o rect.o` แล้ว `rustc main.rs -C link-arg=rect.o -o main` แบบเดียวกับ
     ตัวอย่าง `point.c`/`point_distance` ในหัวข้อ 43.6.1

3. **(ยาก)** สร้าง Rust library ที่ expose ฟังก์ชัน `#[no_mangle] extern "C" fn` ชื่อ `parse_and_double`
   ที่รับ `*const c_char` (C string ที่เก็บตัวเลข เช่น `"21"`) แปลงเป็น `&str` ด้วย `CStr`, parse เป็น
   `i32` ด้วย `.parse()`, คูณสอง แล้วคืนค่าเป็น `i32` — ถ้า parse ไม่ได้ (string ไม่ใช่ตัวเลข) ให้คืน `-1`
   แทนการ panic หรือ unwrap พร้อม compile เป็น `cdylib` และเขียนโปรแกรม C เล็ก ๆ ที่เรียกใช้ฟังก์ชันนี้จริง
   ด้วยทั้ง string ที่ถูกต้องและ string ที่ผิดรูปแบบ
   - Hint: ต้องใช้ `unsafe { CStr::from_ptr(ptr) }.to_str()` ซึ่งคืน `Result` สองชั้น (UTF-8 validity และ
     parse เป็นตัวเลข) — ใช้ pattern matching หรือ `.ok()` + `.and_then()` แปลงทั้งสองความล้มเหลวให้กลาย
     เป็น `-1` โดยไม่ panic เลยแม้แต่กรณีเดียว (เพื่อป้องกันปัญหาในกับดักที่ 3)

4. **(ยาก/ประยุกต์ใช้งานจริง)** ขยายตัวอย่างระบบ GPS ในหัวข้อ 43.11: เพิ่มฟังก์ชัน C ใหม่ชื่อ
   `is_within_radius` ที่รับสองจุด `GpsPoint` และค่ารัศมี (กิโลเมตร) แล้วคืน `int` (`1` ถ้าอยู่ในรัศมี, `0`
   ถ้าไม่) โดยเรียกใช้ `haversine_distance_km` ที่มีอยู่แล้วภายในไฟล์ C เดียวกัน จากนั้นในฝั่ง Rust เขียน
   struct `GpsRegion` (repr(C)) ที่มี field `center: GpsPoint` และ `radius_km: f64` แล้วเขียนฟังก์ชัน Rust
   (ไม่ต้อง `extern "C"` เพราะใช้ในโค้ด Rust ล้วน ๆ) ที่รับ `Vec<GpsPoint>` และ `&GpsRegion` แล้วกรองคืนเฉพาะ
   จุดที่อยู่ในรัศมีของ region นั้น โดยเรียก `is_within_radius` ผ่าน FFI ข้างในลูป — ทดสอบด้วยพิกัดเมืองไทย
   จริงหลายเมือง (เช่น กรุงเทพฯ, เชียงใหม่, ภูเก็ต, ขอนแก่น)
   - Hint: struct ที่ซ้อน struct อื่นอยู่ข้างใน (`GpsRegion` มี `GpsPoint` เป็น field) ก็ต้องมี `#[repr(C)]`
     ทั้งสอง struct ไม่ใช่แค่ struct นอกสุด — ทดสอบด้วย `size_of`/`offset_of!` แบบหัวข้อ 43.6 เพื่อยืนยัน
     layout ก่อนเรียกใช้จริงถ้าไม่มั่นใจ

## สรุป

บทนี้ปิดคำเกริ่นที่ Part 41 และ Part 42 วางไว้ทั้งสองข้อพร้อมกัน: **FFI คือหนึ่งใน 5 superpower ของ
`unsafe`** และ **`#[repr(C)]` มีอยู่เพื่อ FFI โดยเฉพาะ** — เราเห็นกลไกทั้งหมดที่ทำให้ทั้งสองสิ่งนี้ทำงาน
ร่วมกันได้จริง ตั้งแต่การประกาศ signature ด้วย `extern "C"` (พร้อมเหตุผลที่แม่นยำว่าทำไมทุกการเรียกต้อง
`unsafe` — เพราะ compiler ไม่มีทางพิสูจน์ได้ว่า signature ที่คุณเขียนตรงกับของจริง หรือฟังก์ชันฝั่ง C จะไม่
ทำอะไรที่ละเมิดกฎความปลอดภัยของ Rust), การแมป type ให้ถูกต้องด้วย `std::os::raw`, การส่ง string ข้าม
boundary ด้วย `CString`/`CStr` ที่แก้ปัญหาความแตกต่างพื้นฐานระหว่าง fat pointer ของ Rust กับ null-terminated
string ของ C, การส่ง struct ข้าม boundary ด้วย `#[repr(C)]` (พร้อมหลักฐานจริงจาก `offset_of!` ที่แสดงให้เห็น
ว่า `repr(Rust)` เรียง field ใหม่ได้จริง และหลักฐานจริงของความเสียหายเมื่อลำดับ field ผิด), การกลับทิศทาง
expose ฟังก์ชัน Rust ให้ C เรียกด้วย `#[no_mangle]` + `extern "C"` + crate-type `cdylib`/`staticlib`,
การตั้งค่า linking ผ่าน `#[link]` หรือ `build.rs`, การป้องกัน undefined behavior จาก panic ที่ unwind ข้าม
boundary ด้วย `catch_unwind` หรือ `panic = "abort"`, และการรู้จัก `bindgen`/`cbindgen` เป็นเครื่องมือที่ช่วย
ลดความเสี่ยงจากการเขียน binding มือสำหรับ C API ขนาดใหญ่

ข้อคิดที่สำคัญที่สุดที่ควรพาไปใช้ต่อจากบทนี้คือ: **FFI ไม่ใช่ "โซนอันตรายที่ไม่มีการควบคุม"** แม้จะอยู่ใน
`unsafe` ทั้งหมดก็ตาม — Rust ยังตรวจสอบทุกอย่างที่มันมองเห็นได้อย่างเคร่งครัดเหมือนเดิม (type matching ตาม
signature ที่คุณเขียน, ownership, lifetime ของ owned type อย่าง `CString`) สิ่งที่ `unsafe` ปลดล็อกในบทนี้
แคบมากและชัดเจน: **ความรับผิดชอบต่อความถูกต้องของ signature ที่ประกาศ และความปลอดภัยของโค้ดฝั่ง C/foreign
ที่ compiler ไม่มีทางเห็นได้เลย** — นี่คือปรัชญาเดียวกันกับที่ Part 41 วางรากฐานไว้ตั้งแต่ต้น เพียงแค่ประยุกต์
ใช้กับสถานการณ์ที่ซับซ้อนและมีมูลค่าทางธุรกิจสูงที่สุดอย่างหนึ่งของโลกซอฟต์แวร์จริง: **การทำให้โค้ดสองภาษา
คนละยุคคุยกันได้อย่างปลอดภัยที่สุดเท่าที่จะทำได้**

Part ถัดไปจะเปลี่ยนโฟกัสจาก "การเชื่อมกับโลกภายนอก" ไปสู่ "การขยายภาษา Rust เองจากภายใน" — **Part 44:
Procedural Macros เบื้องต้น** จะสอนวิธีเขียนโค้ด Rust ที่**สร้างโค้ด Rust อื่นขึ้นมาเองตอน compile time**
(ต่างจาก `macro_rules!` ใน Part 36 ที่เป็น declarative macro โดย procedural macro ทำงานเป็นฟังก์ชัน Rust
จริง ๆ ที่รับ token stream เข้ามาแล้วคืน token stream ใหม่ออกไป) ซึ่งเป็นกลไกเบื้องหลัง `derive` macro
ที่คุณใช้มาตลอดหลักสูตร (`#[derive(Debug)]`, `#[derive(Clone)]`) — บทนั้นจะเปิดเผยว่าเวทมนตร์เหล่านี้ทำงาน
อย่างไรจริง ๆ เบื้องหลัง

---

**Part ก่อนหน้า:** [Raw Pointers และ Memory Layout](part-042-raw-pointers-memory.md) | **Part ถัดไป:**
[Procedural Macros เบื้องต้น](part-044-proc-macros-basics.md)
