# Part 35: Cargo ขั้นสูง (features, profiles, workspaces จริง, Cargo.lock)

> โมดูล: ระดับกลาง (Intermediate) | ระดับ: กลาง | เวลาโดยประมาณ: 160 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายทุกคีย์สำคัญของ `[profile.dev]`/`[profile.release]` ได้อย่างละเอียด (`opt-level`, `debug`,
  `debug-assertions`, `overflow-checks`, `lto`, `codegen-units`, `panic`, `strip`) พร้อมบอก trade-off เชิง
  performance/compile-time/binary-size ของแต่ละคีย์ได้จริง ไม่ใช่แค่จำชื่อ
- สร้าง **custom profile** ของตัวเอง (เช่น profile สำหรับ profiling) ด้วย `inherits` เพื่อผสมข้อดีของหลาย
  profile เข้าด้วยกัน
- อธิบายพฤติกรรม **feature unification** ข้าม dependency graph/workspace ได้อย่างแม่นยำ — รู้ว่าเมื่อไร Cargo
  จะ "รวม" feature ของ dependency ตัวเดียวกันที่ถูกเรียกจากหลายจุดเข้าด้วยกัน และผลกระทบที่ตามมา
- ป้องกัน feature ที่ขัดแย้งกัน (mutually exclusive features) ด้วย `compile_error!()` และเขียน
  feature-gated module ทั้งโมดูล ไม่ใช่แค่ระดับฟังก์ชัน
- ออกแบบ production workspace หลาย crate ด้วย `[workspace.dependencies]` และ `[workspace.package]` เพื่อรวม
  ศูนย์เวอร์ชัน dependency และ metadata ที่ใช้ร่วมกัน ลดปัญหาการพิมพ์เวอร์ชันไม่ตรงกันระหว่าง member
- อ่านโครงสร้างไฟล์ `Cargo.lock` ได้เข้าใจ, เลือกใช้ `cargo update`/`cargo update -p`/`cargo update --precise`
  ได้ถูกสถานการณ์ และอธิบายความต่างของ resolver v1/v2/v3 ได้
- ตั้งค่า `.cargo/config.toml` (alias คำสั่ง, environment variable, build target) และรู้จักหน้าที่พื้นฐานของ
  `build.rs`, `cargo tree`, `cargo install`, และ `cargo audit`

## ความรู้ที่ต้องมีมาก่อน

- **Part 2 (Cargo และโครงสร้างโปรเจกต์)**: ต้องเข้าใจกายวิภาคของ `Cargo.toml`, ความต่างระหว่าง debug/release
  build, และเคยเห็นภาพรวมสั้น ๆ ของ `[profile.dev]`/`[profile.release]` มาแล้ว (หัวข้อ 2.5) — บทนี้จะเจาะลึกทุก
  คีย์ของ profile ที่ Part 2 บอกไว้ว่า "จะพูดถึงแบบละเอียดจริงจังใน Part 35" คือบทนี้เอง
- **Part 17 (Packages, Crates, Workspaces)**: ต้องเข้าใจ package/crate/module, path dependency, workspace
  พื้นฐาน (`[workspace]`, `members`, `-p`/`--workspace`), optional dependency, และ `[features]` พื้นฐานพร้อม
  `#[cfg(feature = "...")]` มาแล้ว — บทนี้จะไม่ทวนพื้นฐานเหล่านั้นซ้ำ แต่จะไปลึกกว่านั้นในเรื่อง feature
  unification, `[workspace.dependencies]`, และ `[workspace.package]` ที่ Part 17 ยังไม่ได้พูดถึง
- **Part 3 (ตัวแปร ชนิดข้อมูลพื้นฐาน)**: ต้องจำได้ว่า integer overflow ใน debug build จะ panic แต่ใน release
  build จะ wrap around — บทนี้จะโยงกลับไปหาคีย์ `overflow-checks` ที่เป็นตัวควบคุมพฤติกรรมนี้จริง ๆ
- **Part 12 (Result และ Error Handling เบื้องต้น)**: ต้องเข้าใจว่า panic คืออะไร และ `unwind` กับ stack
  unwinding เกี่ยวข้องกับการ cleanup ตอน panic อย่างไร — บทนี้จะโยงกลับไปหาคีย์ `panic = "unwind" | "abort"`
- **Part 32-33 (Testing)** และ **Part 34 (Documentation: rustdoc)**: ควรเคยรัน `cargo test`, `cargo doc` มา
  บ้างแล้ว เพราะบางหัวข้อ (feature unification, resolver v2/v3) จะอ้างถึงพฤติกรรมของ dev-dependencies/
  build-dependencies ที่ใช้ตอน test/doc

## เนื้อหา

### 35.1 ทวนสิ่งที่ Part 2 เกริ่นไว้: `[profile.*]` คืออะไรกันแน่

Part 2 หัวข้อ 2.5 แสดงให้เห็นว่า `cargo build` (ไม่มี flag) กับ `cargo build --release` ให้ผลลัพธ์ต่างกันมาก —
debug build compile เร็วแต่รันช้า ส่วน release build compile ช้าแต่รันเร็ว และปิดท้ายด้วยการโชว์ตัวอย่างสั้น ๆ
ว่าปรับแต่งค่าเหล่านี้ได้ผ่าน section `[profile.dev]`/`[profile.release]` โดยยังไม่อธิบายรายละเอียด ตอนนี้ถึงเวลา
เจาะลึกจริง ๆ แล้ว

**Profile คือชุดค่า config ที่ควบคุมว่า `rustc` จะ compile โค้ดของคุณ (และทุก dependency) อย่างไร** — ไม่ใช่
ควบคุม "โค้ดทำอะไร" (behavior ทาง logic ไม่เปลี่ยน) แต่ควบคุม "compile ออกมาแบบไหน" (เร็ว/ช้าตอน compile, เร็ว/
ช้าตอนรัน, ใหญ่/เล็กแค่ไหน, มี debug info หรือไม่) Cargo มี profile ติดตั้งมาให้ 4 ตัวโดย default:

| Profile | ใช้เมื่อ | ค่า optimize เริ่มต้น |
|---|---|---|
| `dev` | `cargo build`, `cargo run` (ไม่ใส่ `--release`) | ไม่ optimize เลย เน้น compile เร็ว |
| `release` | `cargo build --release`, `cargo run --release` | optimize เต็มที่ เน้นรันเร็ว |
| `test` | `cargo test` (ไม่ใส่ `--release`) | เหมือน `dev` โดย inherit ค่ามาแทบทั้งหมด |
| `bench` | `cargo bench` | เหมือน `release` โดย inherit ค่ามาแทบทั้งหมด |

ทั้ง 4 profile นี้เขียนทับ (override) ค่า default บางส่วนได้ผ่าน `Cargo.toml` — และ (ที่จะเห็นในหัวข้อ 35.9) คุณ
ยังสร้าง **profile ของตัวเองเพิ่มได้** นอกเหนือจาก 4 ตัวนี้ด้วย

ลองดูค่า default เต็มรูปแบบของ `dev` และ `release` ที่ Cargo ใช้จริงถ้าคุณไม่เขียนอะไรเพิ่มเลย (นี่คือค่าที่ถูก
"ฝัง" อยู่ในตัว Cargo เอง ไม่ได้อยู่ในไฟล์ `Cargo.toml` ที่ `cargo new` สร้างให้ — คุณเห็นมันได้ก็ต่อเมื่อเขียนมัน
ลงไปเองอย่างชัดเจนเพื่อ override):

```toml
[profile.dev]
opt-level = 0
debug = true
debug-assertions = true
overflow-checks = true
lto = false
panic = "unwind"
incremental = true
codegen-units = 256
strip = false

[profile.release]
opt-level = 3
debug = false
debug-assertions = false
overflow-checks = false
lto = false
panic = "unwind"
incremental = false
codegen-units = 16
strip = false
```

สังเกตว่าทั้งสอง profile มีคีย์ชื่อเดียวกันหมด แค่ **ค่าเริ่มต้นตรงกันข้ามกันในเกือบทุกคีย์** — นี่คือปรัชญาการ
ออกแบบของ Cargo: `dev` ปรับทุกอย่างให้ compile เร็วที่สุดและช่วย debug ได้ง่ายที่สุด (แลกกับความเร็วตอนรัน) ส่วน
`release` ปรับทุกอย่างให้รันเร็วที่สุด/binary เล็กที่สุด (แลกกับเวลา compile) หัวข้อถัดไปเราจะไล่ดูคีย์เหล่านี้
ทีละตัวว่าทำอะไรจริง ๆ และทำไมค่า default ของแต่ละ profile ถึงตั้งไว้แบบนั้น

### 35.2 `opt-level`: ระดับการ optimize ตั้งแต่ 0 ถึง 3 และ s/z

`opt-level` คือคีย์ที่มีผลต่อความเร็วรันมากที่สุดในบรรดาคีย์ทั้งหมดของ profile — มันส่งตรงไปเป็น flag
`-C opt-level` ของ `rustc` ซึ่งควบคุมว่า **LLVM** (compiler backend ที่ `rustc` ใช้สร้าง machine code จริง)
จะพยายาม optimize โค้ดมากแค่ไหนก่อนสร้างเป็น binary

| ค่า | ความหมาย | เหมาะกับ |
|---|---|---|
| `0` | ไม่ optimize เลย compile เร็วที่สุด | development (ค่า default ของ `dev`) |
| `1` | optimize พื้นฐานเล็กน้อย | ไม่ค่อยมีใครใช้ตรง ๆ (อยู่ระหว่าง 0 กับ 2) |
| `2` | optimize ปานกลาง ไม่ inline ฟังก์ชันใหญ่มาก | บาง production ที่อยากบาลานซ์ compile time |
| `3` | optimize เต็มที่ (aggressive inlining, vectorization, loop unrolling) | ค่า default ของ `release`, ส่วนใหญ่ใช้ตัวนี้ |
| `"s"` | optimize เพื่อ**ขนาดไฟล์เล็ก** โดยยอมเสียความเร็วบางส่วน | embedded, WASM, เมื่อขนาด binary สำคัญกว่าความเร็ว |
| `"z"` | เหมือน `"s"` แต่บีบขนาดให้เล็กที่สุด (ไม่สนความเร็วเลย) | สถานการณ์ที่ขนาดสำคัญที่สุด (เช่น firmware เล็ก ๆ) |

ข้อสังเกตสำคัญ: `"s"`/`"z"` ต้องเขียนเป็น **string** (มีเครื่องหมายคำพูดคร่อม) ส่วน `0`-`3` เป็น**เลขจำนวนเต็ม**
ล้วน ๆ (ไม่มีเครื่องหมายคำพูด) — สับสนกันบ่อยเพราะทั้งคู่อยู่ในคีย์เดียวกัน แต่ TOML ต้องแยก type ให้ถูกต้อง

**ทดลองจริง**: เขียนโปรแกรมที่คำนวณหนักพอสมควร (นับจำนวนขั้นตอนของลำดับ Collatz สำหรับตัวเลข 1 ถึง 3 ล้าน) แล้ว
เทียบเวลารันระหว่าง `opt-level = 0` (ค่า default ของ dev) กับ `opt-level = 3` (ค่า default ของ release):

```rust
// src/main.rs
fn collatz_len(mut n: u64) -> u32 {
    let mut steps = 0;
    while n != 1 {
        n = if n % 2 == 0 { n / 2 } else { 3 * n + 1 };
        steps += 1;
    }
    steps
}

fn main() {
    let mut total: u64 = 0;
    for n in 1..3_000_000u64 {
        total += collatz_len(n) as u64;
    }
    println!("total steps = {total}");
}
```

ผลลัพธ์จริงที่วัดได้บนเครื่องทดสอบ (คำสั่ง `time` ของ Linux, รันหลังจาก `cargo build`/`cargo build --release`
เสร็จแล้ว วัดแค่เวลารัน binary ไม่รวมเวลา compile):

```
# opt-level = 0 (debug build ปกติ)
total steps = 428343355
real    0m1.750s

# opt-level = 3 (release build ปกติ)
total steps = 428343355
real    0m0.542s
```

ตัวเลขที่คำนวณได้ (`total steps`) **เหมือนกันเป๊ะ** ทั้งสอง build — ตอกย้ำสิ่งที่ย้ำไว้ในหัวข้อ 35.1 ว่า profile
ไม่เปลี่ยน behavior ทาง logic เปลี่ยนแค่ความเร็ว/ขนาดของ binary ที่ได้ ในกรณีนี้ release build (opt-level 3) เร็ว
กว่า debug build ประมาณ **3.2 เท่า** สำหรับ loop คำนวณหนัก ๆ แบบนี้ — ตัวเลขจริงจะต่างกันไปตามลักษณะโค้ด (โค้ดที่
มี branch/loop เยอะจะได้ประโยชน์จาก optimize มากกว่าโค้ดที่ทำ I/O เป็นหลัก เพราะ I/O ถูก bottleneck ด้วยความเร็ว
ดิสก์/network ไม่ใช่ CPU)

**ทำไม LLVM ทำให้เร็วขึ้นได้มากขนาดนี้?** เหตุผลหลัก ๆ คือ **inlining** (เอาโค้ดของฟังก์ชันเล็ก ๆ ไปแทรกตรงจุดที่
เรียกใช้ ตัด overhead ของการ call function ออก และเปิดโอกาสให้ optimize ต่อข้ามขอบฟังก์ชันได้), **loop
unrolling** (คลี่ loop ออกเป็นโค้ดตรง ๆ หลายชุดเพื่อลด overhead ของการเช็คเงื่อนไข loop ซ้ำ ๆ), **dead code
elimination** (ลบโค้ดที่คำนวณแล้วไม่มีใครใช้ผลลัพธ์ทิ้งไปเลย), และ **vectorization** (แปลง loop ธรรมดาให้ใช้
SIMD instruction ประมวลผลข้อมูลหลายตัวพร้อมกันในคำสั่งเดียว) งานเหล่านี้ต้องวิเคราะห์โค้ดอย่างละเอียดก่อนสร้าง
machine code จริง จึงใช้เวลา compile นานขึ้นตามระดับ `opt-level` ที่สูงขึ้น

### 35.3 `debug`, `debug-assertions`, `overflow-checks`: สามคีย์ที่มือใหม่สับสนบ่อยที่สุด

คีย์สามตัวนี้มีชื่อคล้ายกันจนสับสนได้ง่าย แต่ทำหน้าที่**คนละอย่างกันโดยสิ้นเชิง**:

#### `debug`: ควบคุม debug symbol/debug info ในไฟล์ binary

```toml
[profile.dev]
debug = true       # ค่า default ของ dev

[profile.release]
debug = false       # ค่า default ของ release
```

`debug` ควบคุมว่า compiler จะฝัง**ข้อมูล debug** (ชื่อตัวแปร, mapping ระหว่าง machine code กับเลขบรรทัด source
code, ชนิดข้อมูลของแต่ละค่า) ลงในไฟล์ binary หรือไม่ — ข้อมูลนี้เป็นสิ่งที่ debugger อย่าง `gdb`/`lldb` (และ
breakpoint ใน VS Code) ใช้เพื่อแสดงชื่อตัวแปรที่อ่านรู้เรื่อง แทนที่จะเห็นแค่ตำแหน่ง memory address ดิบ ๆ
นอกจาก `true`/`false` ยังมีค่า **`"line-tables-only"`** ที่เก็บแค่ mapping เลขบรรทัดอย่างเดียว (พอสำหรับอ่าน
stack trace ตอน panic ว่า error เกิดที่บรรทัดไหน) โดยไม่เก็บข้อมูลตัวแปร/ชนิดข้อมูลแบบเต็ม ทำให้ไฟล์เล็กกว่า
`true` มากแต่ยัง debug บางระดับได้:

```toml
[profile.release]
debug = "line-tables-only"   # ได้ stack trace ที่มีเลขบรรทัด แต่ไม่ได้ debug แบบเต็มใน debugger
```

**ทำไม `debug` ไม่กระทบความเร็วตอนรันเลย?** เพราะข้อมูล debug เป็นแค่ **metadata ที่แนบมาข้าง ๆ machine code**
ไม่ใช่ instruction ที่ CPU ต้องประมวลผล ดังนั้นการเปิด `debug = true` ใน release build (ซึ่งบางทีมทำเพื่อให้
debug production ได้ง่ายขึ้นตอนเกิดปัญหา) จะทำให้**ไฟล์ binary ใหญ่ขึ้นและ compile ช้าลงเล็กน้อย** แต่**ไม่ทำให้
โปรแกรมรันช้าลงแม้แต่นิดเดียว** — นี่คือกลยุทธ์ที่บริษัทจริงหลายแห่งใช้: build release แบบ optimize เต็มที่
(`opt-level = 3`) แต่**ยังเก็บ debug info ไว้** เพื่อให้ตอน production พัง สามารถอ่าน stack trace แบบมีเลขบรรทัด
ได้ทันที ไม่ต้อง reproduce บัคใน debug build ใหม่

#### `debug-assertions`: เปิด/ปิด macro ตรวจสอบเงื่อนไขที่ผังไว้ในโค้ด

```toml
[profile.dev]
debug-assertions = true    # ค่า default ของ dev

[profile.release]
debug-assertions = false    # ค่า default ของ release
```

`debug-assertions` ควบคุมว่า macro `debug_assert!`, `debug_assert_eq!`, `debug_assert_ne!` จะ**ถูก compile
เข้ามาจริง**หรือไม่ (คล้ายกับ `assert()` ของ C ที่ปิดได้ด้วย `NDEBUG`) macro พวกนี้เขียนเหมือน `assert!`,
`assert_eq!` ทุกประการ แต่ **จะไม่ทำอะไรเลยถ้า `debug-assertions = false`** (ถูกลบออกไปตั้งแต่ก่อนคอมไพล์ ไม่ใช่
แค่ข้าม runtime check) เหมาะกับการเช็ค **invariant ภายใน** ที่อยากตรวจสอบตอนพัฒนา แต่ไม่อยากให้เสียเวลา CPU
ตรวจสอบซ้ำในโค้ด production ที่ผ่านการทดสอบมาแล้ว:

```rust
pub struct BankAccount {
    balance_cents: i64,
}

impl BankAccount {
    pub fn withdraw(&mut self, amount_cents: i64) {
        self.balance_cents -= amount_cents;
        // เช็คว่า balance ไม่ติดลบ — เช็คนี้ "ควร" เป็นจริงเสมอถ้า logic ที่เหลือถูกต้อง
        // (สมมติมีการเช็คเงื่อนไข insufficient funds ไปแล้วก่อนเรียกฟังก์ชันนี้)
        // ใช้ debug_assert! เพื่อจับบั๊กตอนพัฒนา/test โดยไม่ทำให้ production ช้าลง
        debug_assert!(self.balance_cents >= 0, "balance ติดลบ แสดงว่า logic การเช็คเงินก่อนหน้ามีบั๊ก");
    }
}
```

ในโค้ดจริงขนาดใหญ่ที่มี invariant แบบนี้กระจายอยู่หลายร้อยจุด การเปิด `debug-assertions` ในทุก build (รวม
release) จะทำให้ CPU เสียเวลาตรวจสอบเงื่อนไขที่ (ถ้าโค้ดถูกต้อง) ไม่มีทางเป็นจริงอยู่แล้วซ้ำ ๆ นับล้านครั้งต่อ
วินาที — นี่คือเหตุผลที่ `release` ปิดมันเป็น default

#### `overflow-checks`: คีย์ที่ควบคุม integer overflow behavior ที่ Part 3 เคยพูดถึง

```toml
[profile.dev]
overflow-checks = true    # ค่า default ของ dev

[profile.release]
overflow-checks = false    # ค่า default ของ release
```

นี่คือคีย์ที่ Part 3 อธิบายพฤติกรรมไว้แล้ว (debug panic ตอน overflow, release wrap around แบบเงียบ ๆ) แต่ยังไม่
เคยชี้ตรง ๆ ว่า **คีย์ไหนใน Cargo.toml เป็นตัวควบคุม** — คำตอบคือ `overflow-checks` นี่เอง เมื่อเปิดไว้ (`true`)
ทุกครั้งที่ integer บวก/ลบ/คูณกันแล้วผลลัพธ์เกินขนาดที่ type นั้นเก็บได้ โปรแกรมจะ **panic ทันที** แทนที่จะปล่อยให้
ค่า wrap around กลับไปเริ่มจากน้อยที่สุด (หรือมากที่สุดถ้าเป็นการลบ) แบบเงียบ ๆ โดยไม่มีการเตือนใด ๆ

ทดลองจริง — โค้ดที่ทำให้ `u8` (เก็บได้ 0-255) overflow ตอน runtime (ตั้งใจให้ compiler ไม่รู้ค่าตอน compile-time
ด้วยการอ่านจำนวน command-line argument เข้ามาผสม ไม่อย่างนั้น compiler จะจับ error นี้ได้ตั้งแต่ตอน compile ด้วย
lint `arithmetic_overflow` ไปเลย ไม่ต้องรอถึง runtime):

```rust
fn main() {
    // args().count() ได้อย่างน้อย 1 เสมอ (ตัวโปรแกรมเองนับเป็น arg แรก) แต่ compiler ไม่รู้ค่านี้ตอน compile-time
    let x: u8 = std::env::args().count() as u8 + 250;
    let y: u8 = 10;
    let z = x + y; // 251 + 10 = 261 ซึ่งเกิน 255 ที่ u8 เก็บได้ -> overflow แน่นอนตอน runtime
    println!("z = {z}");
}
```

รันแบบ debug build (`overflow-checks = true`):

```
thread 'main' (24812) panicked at src/main.rs:4:13:
attempt to add with overflow
stack backtrace:
   0: __rustc::rust_begin_unwind
   ...
```

โปรแกรมจบด้วย exit code `101` (exit code มาตรฐานของ Rust เมื่อ panic) — ไม่มีทางที่ `z` จะได้ค่าอะไรเลย เพราะ
โปรแกรม crash ก่อนถึงบรรทัด `println!`

รันแบบ release build (`overflow-checks = false`):

```
z = 5
```

ไม่มี panic เลย — `261 mod 256 = 5` (wrap around แบบวนกลับไปเริ่มจาก 0 ใหม่แบบเงียบ ๆ) โปรแกรมรันจบตามปกติ
โดยที่ `z` ได้ค่าที่ (ในทางตรรกะทางธุรกิจ) น่าจะผิดโดยสิ้นเชิง แต่ compiler ไม่บอกอะไรเลย

**ทำไมค่า default ของแต่ละ profile ถึงตั้งไว้แบบนี้?** ในระหว่างพัฒนา คุณ**อยาก**ให้โปรแกรม panic ทันทีที่เกิด
overflow เพราะมันมักหมายถึงบั๊ก (เช่นคำนวณราคาสินค้าผิด, index อาเรย์ผิด) ยิ่ง panic ให้เห็นเร็วเท่าไหร่ ยิ่งแก้
บั๊กได้เร็วเท่านั้น แต่ในโค้ด production ที่ผ่านการทดสอบมาอย่างละเอียดแล้ว การเช็ค overflow ทุกครั้งที่มีการบวก/
ลบ/คูณเลขมีต้นทุนด้าน performance (แม้จะเล็กน้อยต่อครั้ง แต่สะสมมากในโค้ดที่วน loop คำนวณหนัก) — Rust จึงเลือก
ปิดมันใน release เพื่อความเร็ว โดยตั้งสมมติฐานว่าโค้ดที่ผ่าน test ใน debug build มาแล้วไม่ควรมี overflow ที่ไม่
ตั้งใจหลุดเข้า production

**ถ้าอยากเปิด overflow-checks ใน release ด้วย (สำหรับโค้ดที่อ่อนไหวเรื่องความถูกต้องของตัวเลขมาก เช่นระบบการเงิน)**
เขียน override ตรง ๆ:

```toml
[profile.release]
overflow-checks = true
```

ข้อเสียคือเสีย performance เล็กน้อยเพื่อความปลอดภัยที่มากขึ้น — trade-off นี้สมเหตุสมผลมากสำหรับโค้ดที่ความ
ถูกต้องสำคัญกว่าความเร็วเล็กน้อย (เช่นระบบคำนวณดอกเบี้ย, ระบบตัดเงินในบัตร) แต่ไม่คุ้มสำหรับโค้ดที่ต้อง
ประมวลผลข้อมูลจำนวนมากแบบ real-time (เช่น game engine, video codec)

### 35.4 `lto`: Link-Time Optimization

`lto` (Link-Time Optimization) คือคีย์ที่ทรงพลังที่สุดในบรรดา profile key ทั้งหมด แต่ก็เป็นตัวที่ทำให้ compile
ช้าที่สุดด้วยเช่นกัน — เข้าใจมันให้ดีจะช่วยตัดสินใจได้ว่าคุ้มค่าจะเปิดหรือไม่สำหรับโปรเจกต์ของคุณ

**ปัญหาที่ LTO แก้**: โดย default (`lto = false`) `rustc` จะ optimize **แต่ละ crate แยกกัน** — ตอน compile
crate A มันมองไม่เห็นเนื้อในของ crate B แม้ A จะเรียกฟังก์ชันจาก B ก็ตาม (มันรู้แค่ signature ของฟังก์ชันนั้น
ผ่านสิ่งที่เรียกว่า "metadata" ที่ crate B export ไว้) ทำให้ optimizer **inline ข้าม crate ไม่ได้** — ถ้าฟังก์ชัน
เล็ก ๆ จาก crate B ถูกเรียกจาก crate A บ่อยมาก การ inline มันเข้าไปในจุดที่เรียกใช้จะช่วยได้มาก แต่ทำไม่ได้ถ้า
compile แยก crate กัน

**LTO แก้ปัญหานี้โดยดึงทุก crate เข้ามา optimize รวมกันเป็นก้อนเดียวตอนขั้นตอน link** (ขั้นตอนสุดท้ายที่รวม
object file ของทุก crate เป็น executable ไฟล์เดียว) ทำให้ optimizer มองเห็น**ทั้งโปรแกรม**พร้อมกัน เปิดโอกาสให้
inline ข้าม crate, ลบ dead code ข้าม crate, และ optimize แบบที่มองเห็นภาพรวมกว้างกว่าเดิมได้

ค่าที่ตั้งได้:

| ค่า | ความหมาย |
|---|---|
| `false` (หรือ `"off"`) | ไม่ทำ LTO เลย (ค่า default ของทั้ง dev และ release) — แต่ยังมี "thin-local LTO" ระดับ codegen unit ภายในแต่ละ crate ทำงานอยู่เบื้องหลังอัตโนมัติเมื่อ `codegen-units > 1` |
| `true` หรือ `"fat"` | LTO แบบเต็มรูปแบบ ("fat"/"full") — รวมทุก crate เป็น IR (intermediate representation) เดียวก่อน optimize ให้ optimize ได้ลึกที่สุด แต่ compile **ช้าที่สุด** และกิน RAM มากที่สุดตอน link |
| `"thin"` | LTO แบบเบากว่า "fat" — แต่ละ crate ยัง optimize แยกกันบางส่วนก่อน แล้วค่อยทำ cross-crate optimization แบบจำกัดกว่า "fat" compile เร็วกว่า "fat" มาก แต่ optimize ได้ลึกน้อยกว่าเล็กน้อย เป็นจุดกึ่งกลางที่ทีมส่วนใหญ่เลือกใช้ |

**ทดลองจริง**: ใช้โปรแกรม Collatz จากหัวข้อ 35.2 เดิม เทียบเวลา build และเวลารันระหว่าง release ธรรมดา
(`lto = false`, ค่า default) กับ release ที่เปิด `lto = "fat"` พร้อม `codegen-units = 1`:

```
# release ธรรมดา (lto = false, codegen-units = 16 ค่า default)
runtime: 0.542s

# release + lto = "fat" + codegen-units = 1
build time: 3.231s   <- ช้ากว่าปกติอย่างเห็นได้ชัด แม้โปรแกรมมีแค่ ~15 บรรทัด
runtime:    0.569s   <- แทบไม่ต่างจากเดิมเลย (ในบางรันอาจช้ากว่าเล็กน้อยด้วยซ้ำ เพราะ noise ของการวัดเวลา)
```

**ทำไมรันไม่เร็วขึ้นเลยในตัวอย่างนี้?** เพราะโปรแกรมทดสอบนี้มี**แค่ crate เดียว** (ไม่มี dependency ภายนอกที่ถูก
เรียกข้าม crate) — LTO ให้ประโยชน์จากการ inline **ข้าม crate boundary** เป็นหลัก ถ้าโปรแกรมของคุณไม่มีการเรียก
ข้าม crate บ่อย ๆ (หรือ dependency ที่คุณใช้ถูก optimize มาอย่างดีอยู่แล้วภายในตัวมันเอง) LTO จะแทบไม่ช่วยอะไร
เลย แต่ยัง**เสียเวลา compile เพิ่มขึ้นมาก**อยู่ดี — นี่คือบทเรียนสำคัญ: **LTO ไม่ใช่ "ปุ่มวิเศษที่กดแล้วเร็วขึ้น
เสมอ"** มันช่วยได้จริงเฉพาะโปรแกรมที่มี dependency graph ซับซ้อน มีการเรียกข้าม crate บ่อย (เช่นโปรแกรมที่ใช้
`serde`/`tokio`/library ขนาดใหญ่จำนวนมาก เรียก method เล็ก ๆ ข้าม crate ตลอดเวลาใน hot path) ซึ่งเป็นสถานการณ์ที่
พบได้บ่อยในโปรแกรม production จริง (ต่างจากตัวอย่างการทดลองนี้ที่เขียนแค่ crate เดียวโดด ๆ)

**กฎการใช้งานจริง**: เปิด `lto = "thin"` เป็นค่าเริ่มต้นที่สมเหตุสมผลสำหรับ release build ของโปรแกรมที่จะแจกจ่าย
จริง (compile ช้าขึ้นไม่มากแต่ได้ประโยชน์บางส่วน) แล้วลองวัด benchmark จริงของโปรแกรมคุณเปรียบเทียบกับ `lto =
"fat"` ก่อนตัดสินใจใช้ตัวที่ compile ช้าที่สุด — ถ้าความต่างของ performance ไม่คุ้มกับเวลา compile ที่เสียไป
(โดยเฉพาะใน CI/CD ที่ build ทุก commit) `"thin"` มักเป็นจุดสมดุลที่ดีที่สุด เราจะกลับมาวัด LTO อย่างเป็นระบบมาก
ขึ้นด้วยเครื่องมือ benchmark จริงจังใน **Part 54 (Performance Optimization)**

**ข้อจำกัดที่ควรรู้**: LTO ไม่มีผลกับ crate ที่ compile เป็น `dylib` (dynamic library) — ถ้า `[lib]` ของคุณตั้ง
`crate-type = ["dylib"]` และเปิด `lto` ไว้ Cargo จะเตือนว่า setting นี้ถูกละเว้นสำหรับ crate type นั้น เพราะ
dynamic library ต้องคง symbol แยกไว้ให้ program อื่น link ตอน runtime ได้ ไม่สามารถ inline ทุกอย่างเข้าด้วยกัน
จนไม่เหลือ boundary ของ crate แบบที่ LTO ต้องการ

### 35.5 `codegen-units`: จำนวนหน่วยที่ compile แบบ parallel

```toml
[profile.dev]
codegen-units = 256    # ค่า default ของ dev — มาก เพื่อ parallelize compile ให้เร็วที่สุด

[profile.release]
codegen-units = 16    # ค่า default ของ release — น้อยกว่า เพื่อเปิดโอกาส optimize ได้มากขึ้น
```

เมื่อ `rustc` compile crate หนึ่งตัว มันจะแบกโค้ดออกเป็น **codegen unit** หลายชิ้น แล้ว compile แต่ละชิ้นแบบ
**parallel** (ใช้หลาย CPU core ทำงานพร้อมกัน) เพื่อความเร็ว — นี่คือ trade-off ตรงข้ามกับ LTO โดยตรง:

- **`codegen-units` มาก** (256 ใน dev) → แบ่งงานเป็นชิ้นเล็ก ๆ จำนวนมาก compile แบบ parallel ได้เต็มที่ **เร็ว
  ตอน compile** แต่ optimizer มองเห็นโค้ดแค่ทีละชิ้นเล็ก ๆ ไม่เห็นภาพรวมทั้ง crate ทำให้ optimize ได้จำกัดกว่า
  (inline ข้ามชิ้นไม่ได้ เพราะแต่ละชิ้น optimize แยกกันคนละ thread)
- **`codegen-units` น้อย** (16 ใน release, หรือ `1` สำหรับบีบให้เหลือชิ้นเดียว) → parallelize ได้น้อยกว่า
  ("1" คือปิด parallel ไปเลย compile ทีละ core เดียวสำหรับ crate นั้น) **compile ช้าลง** แต่ optimizer เห็นโค้ด
  เป็นก้อนใหญ่ขึ้น มีโอกาส inline/optimize ข้ามฟังก์ชันภายใน crate เดียวกันได้ลึกกว่า

**ความสัมพันธ์กับ `lto`**: `codegen-units = 1` มักถูกตั้งควบคู่กับ `lto = "fat"` (ตามที่เห็นในตัวอย่างหัวข้อ
35.4) เพราะทั้งสองคีย์มีเป้าหมายเดียวกันคือ "ยอมเสียเวลา compile เพื่อให้ optimizer เห็นโค้ดเป็นก้อนใหญ่ที่สุด
เท่าที่จะทำได้" — `codegen-units = 1` บีบให้เห็นทั้ง crate เป็นก้อนเดียว ส่วน `lto` บีบให้เห็นทั้งโปรแกรม (ข้าม
crate) เป็นก้อนเดียว ใช้คู่กันจะได้ optimize ที่ลึกที่สุดเท่าที่ Cargo ทำได้ แต่ก็แลกมาด้วยเวลา compile ที่นานสุด
เช่นกัน (ไม่มี parallelism เหลือให้ใช้เลยทั้งภายใน crate และข้าม crate)

**กฎการใช้งานจริง**: ปล่อยค่า default (`16` สำหรับ release) ไว้ก่อนสำหรับการพัฒนาปกติ แล้วค่อยลองปรับเป็น `1`
เฉพาะตอน build binary ที่จะแจกจ่ายจริง (production release, publish บน crates.io เป็นต้น) ที่ยอมเสียเวลา CI
build นานขึ้นเพื่อ performance สูงสุด — ไม่ควรตั้ง `codegen-units = 1` ใน `[profile.dev]` เด็ดขาด เพราะจะทำให้
loop เขียนโค้ด-ทดสอบช้าลงมากโดยไม่ได้ประโยชน์อะไร (dev build ไม่ได้เน้นความเร็วตอนรันอยู่แล้ว)

### 35.6 `panic`: `unwind` กับ `abort`

```toml
[profile.dev]
panic = "unwind"    # ค่า default ของทั้ง dev และ release

[profile.release]
panic = "unwind"
```

Part 12 อธิบายไว้แล้วว่าเมื่อโปรแกรม panic ค่า default ของ Rust คือ **unwind the stack** — คลาย stack frame
ออกทีละชั้นจากจุดที่ panic กลับขึ้นไปเรื่อย ๆ เรียก destructor (`Drop::drop`) ของทุกค่าที่ยัง alive อยู่ตามทาง
เพื่อ cleanup resource (ปิดไฟล์, คืน memory, ปลด lock) ให้ครบถ้วนก่อนโปรแกรมจะจบ — นี่คือคีย์ `panic` ที่ควบคุม
พฤติกรรมนี้จริง ๆ และมันมีค่าอื่นที่เลือกได้คือ `"abort"`:

- **`"unwind"`** (default) — panic แล้ว unwind stack, เรียก `Drop` ครบทุกจุด, เปิดโอกาสให้ใช้
  `std::panic::catch_unwind` "จับ" panic ไว้แล้วให้โปรแกรมทำงานต่อได้ (ไม่ crash ทั้งโปรเซส) — เหมาะกับโปรแกรม
  ที่ต้องทนทานต่อบัคบางจุด เช่น web server ที่อยากให้ 1 request ที่ panic ไม่ทำให้ทั้ง server ล่ม
- **`"abort"`** — panic แล้ว**เรียก `abort()` ของระบบปฏิบัติการทันที** ไม่มีการ unwind stack เลย ไม่มีการเรียก
  `Drop` ใด ๆ โปรแกรมจบทันทีแบบดิบ ๆ (เหมือน C `abort()`)

**ทำไมถึงมีตัวเลือก `"abort"`?** เหตุผลหลักคือ**ขนาด binary และความเร็วเล็กน้อย** — กลไก unwind ต้องมี "unwind
table" (ข้อมูลบอกว่า stack frame แต่ละจุดต้อง cleanup อะไรบ้างถ้า unwind ผ่าน) ฝังอยู่ในไฟล์ binary ซึ่งกินพื้นที่
พอสมควร การตั้ง `panic = "abort"` ตัดข้อมูลนี้ทิ้งไปเลย ทำให้ **binary เล็กลง** และ code path บางจุดที่ compiler
ไม่ต้องเผื่อ "จะ unwind ผ่านจุดนี้ไหม" ก็ optimize ได้ง่ายขึ้นเล็กน้อยด้วย

**ข้อควรระวังสำคัญที่สุด**: `panic = "abort"` ทำให้ `std::panic::catch_unwind` **ใช้ไม่ได้ผลตามที่คาดหวัง** —
ทดลองจริง:

```rust
fn main() {
    let result = std::panic::catch_unwind(|| {
        panic!("boom");
    });
    println!("{:?}", result.is_err());
}
```

Compile ผ่านได้ปกติแม้ตั้ง `panic = "abort"` ไว้ (compiler ไม่เตือนอะไรเลยตอน compile) แต่พฤติกรรมตอน**รัน**ต่าง
จากที่คาดไว้มาก — ถ้าตั้ง `panic = "unwind"` (ค่า default) โปรแกรมนี้จะพิมพ์ `true` ออกมาปกติ (จับ panic ไว้ได้
โปรแกรมทำงานต่อ) แต่ถ้าตั้ง `panic = "abort"`:

```
thread 'main' (24964) panicked at src/main.rs:3:9:
boom
stack backtrace:
   0: __rustc::rust_begin_unwind
   ...
Aborted (exit code: 134)
```

โปรแกรม**ไม่ได้จับ panic ไว้ได้เลย** — ทั้งโปรเซส abort ไปทันที (`Aborted`, exit code `134` ซึ่งคือ
`128 + SIGABRT`) `catch_unwind` ทำงานไม่ได้ผลเลยเมื่อไม่มีการ unwind ให้จับ นี่คือจุดที่ทำให้หลายทีมเลือกไม่ตั้ง
`panic = "abort"` ในโปรแกรมที่พึ่ง `catch_unwind` เป็นกลไก resilience (เช่น web framework บางตัวใช้
`catch_unwind` ครอบแต่ละ request handler ไว้ เพื่อไม่ให้ 1 request ที่ panic ทำให้ server ทั้งตัวล่ม) —
ถ้าต้องการทั้งขนาด binary เล็ก**และ** resilience แบบนี้ ต้องออกแบบให้ระมัดระวังไม่ให้เกิด panic เลยตั้งแต่ต้น
(ใช้ `Result` แทนตลอด) มากกว่าจะพึ่ง `catch_unwind` เป็นตาข่ายรองรับ

**ข้อควรรู้อีกจุด**: ถ้าตั้ง `panic = "abort"` ไว้ที่ `[profile.release]` แล้วรัน `cargo test --release`, Cargo
จะยังคง**บังคับใช้ `unwind` สำหรับ binary ของ test harness เสมอ** โดยไม่สนใจค่าที่ตั้งไว้ (เพราะ test framework
ของ Rust ต้องพึ่ง unwind เพื่อรายงานผล "test ไหน fail" แยกจากกันโดยไม่ทำให้ test ตัวอื่นหยุดตามไปด้วย — ถ้าใช้
`abort` การ panic ใน 1 test จะ crash ทั้ง process ทำให้รายงานผล test อื่นที่เหลือไม่ได้เลย) จึงมั่นใจได้ว่า
`cargo test`/`cargo bench` ใช้งานได้ปกติเสมอไม่ว่าจะตั้ง `panic` เป็นอะไรใน profile ก็ตาม

**กฎการใช้งานจริง**: สำหรับ CLI tool หรือโปรแกรมที่ crash ทั้งตัวเมื่อเกิด bug ร้ายแรงเป็นพฤติกรรมที่ยอมรับได้
(ไม่มี "1 request" ให้แยก isolate จากกัน) การตั้ง `panic = "abort"` ใน release คุ้มค่าเพื่อลดขนาด binary ส่วน
web server/library ที่ต้องทนทานต่อ panic บางจุดโดยไม่ล่มทั้งระบบ ควรคงค่า default `"unwind"` ไว้

### 35.7 `strip`: ตัด debug symbol ออกจาก binary จริง

```toml
[profile.release]
strip = false    # ค่า default (ไม่ strip)
```

`strip` ควบคุมว่าจะลบ **symbol table** (ชื่อฟังก์ชัน, ชื่อตัวแปร global, debug info ถ้ามี) ออกจากไฟล์ binary
ที่ compile เสร็จแล้วหรือไม่ ค่าที่ตั้งได้:

| ค่า | ความหมาย |
|---|---|
| `false` | ไม่ strip อะไร (ค่า default) |
| `true` หรือ `"symbols"` | ลบ symbol table ทั้งหมด รวม debug info (เทียบเท่ากับรันคำสั่ง `strip` ของ Unix หลัง link เสร็จ) |
| `"debuginfo"` | ลบเฉพาะ debug info แต่**เก็บ** symbol table ไว้ (ยังเห็นชื่อฟังก์ชันใน stack trace/profiler ได้ แต่ debug ด้วย debugger แบบเต็มไม่ได้) |

**ทดลองจริง**: build โปรแกรมเดียวกัน (ที่ตั้ง `strip = true` ไว้ใน release) เทียบกับ profile `profiling` ที่จะ
สร้างในหัวข้อถัดไป (ตั้ง `strip = false` ไว้ตั้งใจ):

```
target/release/store_cli     — 343,960 bytes  — stripped
target/profiling/store_cli   — 2,099,944 bytes — with debug_info, not stripped
```

ความต่างของขนาดในตัวอย่างนี้คือประมาณ **6 เท่า** (343 KB เทียบกับ 2.1 MB) สำหรับโปรแกรมขนาดเล็ก — สำหรับโปรแกรม
ใหญ่ที่มี dependency จำนวนมาก ความต่างสัมบูรณ์ (จำนวน MB ที่ประหยัดได้) จะยิ่งมากขึ้นตามสัดส่วน

**ทำไมถึงอยากลบ debug info ออกจาก binary ที่แจกจ่ายจริง?** สามเหตุผลหลัก:

1. **ขนาดไฟล์เล็กลงมาก** — สำคัญมากถ้าต้อง distribute binary ผ่าน network (Docker image, download ตรง) หรือ
   deploy ไปยังอุปกรณ์ที่มีพื้นที่จำกัด (embedded, container ขนาดเล็ก)
2. **ลดข้อมูลที่รั่วไหลได้** — symbol table เผยชื่อฟังก์ชัน/module path ภายในของคุณ ซึ่งบางองค์กรมองว่าเป็น
   ข้อมูลที่ไม่ควรเปิดเผยให้คนนอกเห็นง่าย ๆ (แม้จะ reverse-engineer กลับมาได้อยู่ดีถ้าพยายามมากพอ แต่การ strip
   symbol ก็เพิ่มความยากในการวิเคราะห์ binary ได้ระดับหนึ่ง)
3. **compile/link เร็วขึ้นเล็กน้อย** — ขั้นตอน strip ไม่ได้ช้า และการไม่ต้องเขียน debug info จำนวนมากลงดิสก์
   ช่วยประหยัดเวลา I/O ตอน link ได้บ้าง

**ข้อเสีย**: ถ้า production binary panic หรือ crash คุณจะได้ stack trace ที่ไม่มีชื่อฟังก์ชัน/เลขบรรทัดให้อ่าน
เลย (เห็นแค่ memory address ดิบ ๆ) ทำให้ debug ปัญหาที่เกิดใน production ยากขึ้นมาก — นี่คือเหตุผลที่บางทีมเลือก
ใช้ `strip = "debuginfo"` (เก็บ symbol ไว้ แต่ตัด debug info เต็มรูปแบบออก) เป็นจุดกึ่งกลาง หรือเก็บไฟล์ debug
info แยกไว้นอก binary (ผ่านเครื่องมือเสริมอย่าง `splitdebuginfo`) เพื่อให้ debug ได้เมื่อจำเป็นโดยไม่ต้องแจก
debug info ไปกับ binary ทุกตัวที่ deploy จริง — รายละเอียดของการตั้งค่าระดับนี้เกี่ยวโยงกับเรื่อง profiling ที่
จะพูดถึงเต็มรูปแบบใน **Part 55 (Profiling Rust Applications)**

### 35.8 ประกอบทุกคีย์เข้าด้วยกัน: `[profile.release]` ที่ tune สำหรับ production CLI tool

มาดูตัวอย่าง `[profile.release]` ที่ปรับแต่งอย่างมีเหตุผลสำหรับ CLI tool ที่จะแจกจ่ายให้ผู้ใช้จริง (สถานการณ์ที่
เหมาะกับตัวอย่าง — ไม่มี server ที่ต้องพึ่ง `catch_unwind`, ต้องการ binary เล็กและเร็วที่สุด):

```toml
[profile.release]
opt-level = 3        # optimize เต็มที่ — CLI tool ต้องการความเร็วตอนรันสูงสุด ไม่ใช่ compile time
lto = "thin"          # LTO แบบเบา ได้ประโยชน์บางส่วนจากการ inline ข้าม crate โดยไม่เสียเวลา compile มากเกินไป
codegen-units = 1     # บีบให้ compile เป็นก้อนเดียวต่อ crate เพื่อให้ optimizer เห็นภาพรวมกว้างสุด
panic = "abort"       # CLI ไม่มี "1 request" ที่ต้อง isolate จากกัน — crash ทั้งโปรแกรมยอมรับได้ แลกกับ binary เล็กลง
strip = true          # ตัด debug symbol ออก ลดขนาดไฟล์ที่แจกจ่ายจริงให้เล็กที่สุด
debug = false         # ไม่ต้องมี debug info เลย (สอดคล้องกับ strip = true อยู่แล้ว)
overflow-checks = false  # ยอมรับความเสี่ยงมาตรฐานของ release — ถ้าโปรแกรมผ่าน test ใน dev build มาอย่างดีแล้ว
```

**อธิบายเหตุผลการเลือกทีละคีย์**:

- `opt-level = 3` + `codegen-units = 1` — คู่นี้ให้ประโยชน์สูงสุดสำหรับความเร็วรัน ใช้ได้เต็มที่เพราะ CLI tool
  มักไม่ได้ build บ่อยเท่า library ที่ทีมอื่นแก้ทุกวัน (build ตอน release มักเป็นกิจกรรมที่เกิดไม่บ่อย เช่น
  ตอน tag version ใหม่) จึงยอม trade เวลา compile ที่นานขึ้นเพื่อความเร็วตอนรันสูงสุด
- `lto = "thin"` — จุดกึ่งกลางที่สมเหตุสมผล ถ้าทีมวัดแล้วว่า `"fat"` ให้ประโยชน์เพิ่มอย่างมีนัยสำคัญสำหรับ
  โปรแกรมนี้โดยเฉพาะ (ผ่านการ benchmark จริงตาม Part 54) ค่อยเปลี่ยนเป็น `"fat"` ทีหลังได้
- `panic = "abort"` — เลือกได้เพราะ CLI tool เป็นโปรแกรมที่รันครั้งเดียวจบ ไม่มีแนวคิด "1 คำสั่งพัง แต่โปรแกรม
  ต้องรันคำสั่งอื่นต่อ" แบบ server ที่รับหลาย request พร้อมกัน — ถ้า logic ผิดจนเกิด panic การ abort ทั้งโปรแกรม
  ทันทีเป็นพฤติกรรมที่ผู้ใช้เข้าใจง่าย (เห็น error แล้วโปรแกรมจบ ไม่ใช่ทำงานต่อในสภาพที่ไม่แน่ใจว่าถูกต้องหรือไม่)
- `strip = true` — CLI tool มักแจกจ่ายเป็นไฟล์ binary เดี่ยวให้ผู้ใช้ดาวน์โหลดตรง ๆ (ผ่าน GitHub Release,
  package manager) ขนาดไฟล์ที่เล็กลงช่วยลดเวลาดาวน์โหลดและพื้นที่เก็บ

ข้อควรระวัง: ชุดค่านี้**ไม่ใช่สูตรตายตัว**ที่ใช้ได้กับทุกโปรเจกต์ — เว็บ server ที่ต้องพึ่ง `catch_unwind` ควรคง
`panic = "unwind"`, โปรเจกต์ embedded ที่ขนาด binary สำคัญที่สุดอาจต้องการ `opt-level = "z"` แทน `3`, และทีมที่
ต้อง debug production บ่อยอาจอยากคง `strip = "debuginfo"` แทน `true` เต็มรูปแบบ — **หลักการที่สำคัญกว่าค่าเฉพาะ
เจาะจงคือ เข้าใจ trade-off ของแต่ละคีย์ แล้วเลือกค่าที่ตรงกับความต้องการจริงของโปรเจกต์คุณ**

### 35.9 Custom Profile: สร้าง profile ของตัวเองด้วย `inherits`

Cargo ไม่ได้จำกัดให้มีแค่ `dev`/`release`/`test`/`bench` เท่านั้น — คุณสามารถประกาศ **profile ใหม่ที่ชื่ออะไรก็ได้**
เพิ่มเข้ามาได้ โดยใช้คีย์ `inherits` เพื่อบอกว่าให้ **สืบทอดค่า default มาจาก profile ที่มีอยู่แล้ว** ก่อน แล้ว
ค่อย override บางคีย์เพิ่มเฉพาะจุดที่ต้องการ

**สถานการณ์จริงที่พบบ่อย**: คุณอยากได้ binary ที่ **optimize เต็มที่แบบ release** (เพื่อวัด performance ที่
ใกล้เคียงของจริง) แต่**ยังเก็บ debug symbol ไว้** (เพื่อให้ profiler อย่าง `perf`/`flamegraph` — ซึ่งจะเรียนใน
**Part 55** — อ่าน stack trace ได้ว่าเวลาไปอยู่ที่ฟังก์ชันไหน) `[profile.release]` ธรรมดาตัด debug info ทิ้งไป
แล้ว (`debug = false` เป็นค่า default) ทำให้ profiler อ่านชื่อฟังก์ชันไม่ได้ — นี่คือจุดที่ custom profile ชื่อ
`profiling` เข้ามาแก้ปัญหา:

```toml
[profile.profiling]
inherits = "release"    # สืบทอดทุกค่าจาก release มาก่อน (opt-level = 3, lto, codegen-units, ฯลฯ)
debug = true              # override เฉพาะคีย์นี้: เปิด debug info กลับมา
strip = false              # override เฉพาะคีย์นี้: ไม่ strip symbol ออก
```

สั่ง build ด้วย profile นี้ผ่าน flag `--profile`:

```bash
cargo build --profile profiling
```

```
    Finished `profiling` profile [optimized + debuginfo] target(s) in 7.83s
```

สังเกตข้อความ `[optimized + debuginfo]` — ต่างจาก `dev` (`[unoptimized + debuginfo]`) และ `release`
(`[optimized]` เฉย ๆ ไม่มี debuginfo) เพราะ profile นี้ผสมข้อดีของทั้งสองแบบเข้าด้วยกันจริง ๆ: **optimize ระดับ
เดียวกับ release แต่เก็บ debug info เหมือน dev**

ทดสอบตรวจสอบด้วยคำสั่ง `file` ของ Unix ว่า binary ที่ได้มี debug info จริงหรือไม่:

```bash
file target/profiling/store_cli
```

```
target/profiling/store_cli: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked,
interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, BuildID[sha1]=3cb44a31...,
with debug_info, not stripped
```

เทียบกับ `target/release/store_cli` ที่ตั้ง `strip = true` ไว้:

```
target/release/store_cli: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked,
interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, BuildID[sha1]=dc4dba0f...,
stripped
```

ผลลัพธ์ตรงตามที่ตั้งใจ — `profiling` profile ได้ `with debug_info, not stripped` ในขณะที่ `release` ได้
`stripped` ล้วน ๆ ตามที่ config บอกไว้ทุกประการ

**ข้อควรรู้เกี่ยวกับ `target/` directory**: แต่ละ custom profile จะมี **โฟลเดอร์ของตัวเองใต้ `target/`**
(เช่น `target/profiling/`) แยกจาก `target/debug/`/`target/release/` โดยสิ้นเชิง ไม่ทับกันและไม่ต้องแก้ config
เพิ่ม — Cargo จัดการให้อัตโนมัติ

**ตัวอย่างการใช้งานอื่นที่พบบ่อย**: ทีมจำนวนมากสร้าง profile ชื่อ `ci` ที่ inherit จาก `dev` แต่ปิด `debug`
เพื่อให้ CI build เร็วขึ้น (ไม่ต้องเขียน debug info ที่ CI ไม่ได้ใช้เลยเพราะ CI แค่รัน test แล้วทิ้ง ไม่มีใครมา
attach debugger):

```toml
[profile.ci]
inherits = "dev"
debug = false    # CI ไม่ต้องใช้ debug info เลย ตัดออกช่วยประหยัดเวลา I/O ตอนเขียนไฟล์และพื้นที่ดิสก์ของ CI runner
```

### 35.10 Feature Unification: พฤติกรรมที่แปลกใจได้บ่อยที่สุดของ Cargo

Part 17 หัวข้อ 17.12 สอนวิธีสร้าง `[features]` ของตัวเอง แต่ยังไม่ได้อธิบายพฤติกรรมสำคัญที่เกิดขึ้นเมื่อ
**dependency ตัวเดียวกันถูกเรียกใช้จากหลายจุดในเดียวกัน dependency graph ด้วย feature ที่ต่างกัน** — นี่คือแนวคิด
ที่เรียกว่า **feature unification**

**กฎของ feature unification**: ถ้า dependency ตัวหนึ่ง (เช่น `serde`) ถูกเรียกจากหลาย crate ในเดียวกัน build
(เดียวกัน `Cargo.lock`) โดยแต่ละจุดขอ feature ต่างกัน **Cargo จะรวม (union) feature ทั้งหมดที่ถูกขอเข้าด้วยกัน
แล้วเปิดให้ `serde` ตัวเดียวที่ compile จริงในทุก feature ที่ถูกขอรวมกัน** — ไม่ใช่ "แต่ละจุดได้ serde คนละ
เวอร์ชัน/คนละ feature set แยกกัน" แต่เป็น "serde ถูก compile แค่ครั้งเดียว ด้วย feature ที่กว้างที่สุดที่ทุกคนขอ
รวมกัน" เหตุผลเชิงเทคนิคคือ **crate หนึ่ง crate จะถูก compile ได้แค่ครั้งเดียวต่อ build** (ไม่ compile `serde`
สองรอบด้วยสอง feature set ที่ต่างกัน) เพราะจะทำให้เกิด type ที่ชื่อเหมือนกันแต่เป็นคนละ type กันโดยสิ้นเชิง
(compile ไม่ผ่านถ้ามีการส่งค่าข้าม)

**ทดลองจริง**: สร้าง workspace 3 member ที่จำลองสถานการณ์นี้ให้เห็นชัด —

```toml
# workspace root Cargo.toml (ส่วน [workspace.dependencies])
[workspace.dependencies]
serde = { version = "1", default-features = false }
```

```toml
# core/Cargo.toml — core ขอ serde แบบ optional เปิดเฉพาะ "derive" ผ่าน feature "json" ของตัวเอง
[dependencies]
serde = { workspace = true, features = ["derive"], optional = true }

[features]
default = []
json = ["serde"]
```

```toml
# api/Cargo.toml — api ขอ serde แบบไม่ optional เลย เปิด "derive" และ "std" ตรง ๆ
[dependencies]
store_core = { path = "../core" }
serde = { workspace = true, features = ["derive", "std"] }
```

```toml
# cli/Cargo.toml — cli depend on ทั้ง core และ api
[dependencies]
store_core = { path = "../core" }
store_api = { path = "../api" }
```

รันคำสั่ง `cargo tree -e features` (flag `-e features` แสดง**feature graph**แทนที่จะแสดงแค่ dependency tree
เฉย ๆ) เพื่อดูว่า `serde` ที่ compile จริงมี feature อะไรเปิดอยู่บ้าง — ครั้งนี้เปิด feature `json` ของ
`store_core` เพิ่มเข้ามาด้วย (`--features store_core/json`) เพื่อให้ `serde` ถูกเรียกจากทั้งสองจุด:

```bash
cargo tree -e features -p store_cli --features store_core/json
```

```
store_cli v0.3.0 (...)
├── store_api feature "default"
│   └── store_api v0.3.0 (...)
│       ├── serde feature "derive"
│       │   └── serde v1.0.229
│       │       ├── serde_core feature "result"
│       │       │   └── serde_core v1.0.229
│       │       └── serde_derive feature "default"
│       │           └── serde_derive v1.0.229 (proc-macro)
│       │               └── ...
│       ├── serde feature "std"
│       │   ├── serde v1.0.229 (*)
│       │   └── serde_core feature "std"
│       │       └── serde_core v1.0.229
│       ├── serde_json feature "default"
...
```

สังเกตว่า `serde v1.0.229` (บรรทัดเดียวที่ compile จริง) มีทั้ง feature **`"derive"`** (ที่ `core` ขอผ่าน
`json` feature) และ **`"std"`** (ที่ `api` ขอตรง ๆ) ปรากฏขึ้นมา**พร้อมกัน** — แม้ `core` เองไม่ได้ขอ `"std"`
เลยก็ตาม เพราะ `serde` ตัวที่ compile จริงในกราฟนี้ถูก**รวม feature จากทุกจุดที่เรียกมันในเดียวกัน build**เข้า
ด้วยกัน

**ผลกระทบเชิงปฏิบัติที่ต้องรู้**: สมมติ `store_core` ตั้งใจออกแบบให้ `json` feature เป็น "ทางเลือก" สำหรับคนที่
อยากได้ความสามารถ JSON export เท่านั้น (คนที่ไม่เปิด `json` ไม่ควรต้องแบก `serde` compile เข้ามาเลย) — แต่ถ้า
มี **crate อื่นในเดียวกัน workspace/dependency graph** (เช่น `store_api` ในตัวอย่างนี้) ที่ดึง `serde` เข้ามา
แบบไม่มีเงื่อนไข (ไม่ใช่ optional) `serde` ก็จะถูก compile เข้ามาอยู่ดีเมื่อ build `store_cli` (เพราะ `store_cli`
depend on ทั้ง `store_core` และ `store_api`) แม้ `store_cli` จะไม่เปิด feature `json` ของ `store_core` เลยก็ตาม
— **นี่คือพฤติกรรมที่ optional dependency ป้องกันไม่ได้ 100%**: มันป้องกันได้แค่ "ไม่มีใครเปิด feature ที่ทำให้
`store_core` เองไปเปิด `serde`" แต่ป้องกันไม่ได้ว่า "crate อื่นในกราฟเดียวกันไม่ได้ดึง `serde` เข้ามาด้วยเหตุผล
ของตัวเอง"

ผลที่ตามมาคือ ถ้า `serde` (หรือ dependency ตัวไหนก็ตาม) ถูกเรียกจากหลายจุดในกราฟด้วย feature ที่กว้างกว่าที่คุณ
คาดไว้ (เช่น feature ที่มีผลข้างเคียงด้าน compile time หรือดึง dependency ย่อยเพิ่มเข้ามาอีก) **การ optimize
ขนาด binary/compile time ด้วยการปิด feature ในจุดหนึ่งอาจไม่ได้ผลจริงถ้ามีอีกจุดในกราฟที่เปิด feature นั้นกว้าง
กว่า** — นี่คือเหตุผลที่ก่อนจะเชื่อว่า "ปิด feature แล้ว dependency จะเล็กลง" ควรตรวจสอบด้วย `cargo tree -e
features` เสมอว่าจริง ๆ แล้ว feature ไหนถูกเปิดอยู่บ้างใน**ทุก**จุดของกราฟ ไม่ใช่แค่มองที่ `Cargo.toml` ของ
crate ตัวเดียวที่คุณกำลังแก้อยู่

**ขอบเขตของ unification**: feature unification เกิดขึ้น**ภายใน dependency graph เดียวกันที่ resolve พร้อมกัน
เป็น `Cargo.lock` เดียว**เท่านั้น — ถ้า `store_core` และ `store_api` ไม่ได้อยู่ใน workspace เดียวกันเลย (คนละ
`Cargo.lock` คนละ build โดยสิ้นเชิง) ก็จะไม่มี unification ข้ามกันแบบนี้ นี่คือเหตุผลเพิ่มเติมที่ workspace
เดียวกันต้องระมัดระวังเรื่อง feature ของ dependency ที่ใช้ร่วมกันให้มากกว่าโปรเจกต์เดี่ยว ๆ ทั่วไป

### 35.11 Mutually Exclusive Features: ป้องกันการเปิด feature ที่ขัดแย้งกัน

บางครั้ง feature สองตัวไม่ควรถูกเปิดพร้อมกันเลย เพราะออกแบบมาให้เป็น "ทางเลือก" ที่แยกกันโดยเจตนา (เช่น
backend สองแบบที่ implement logic เดียวกันคนละวิธี — เปิดทั้งคู่พร้อมกันจะทำให้ compile สับสนว่าจะใช้ตัวไหน)
Cargo **ไม่มีกลไกบังคับ "mutually exclusive feature" ในระดับ `[features]` โดยตรง** (ต่างจากบางระบบ build ที่มี
`conflicts_with` ให้ประกาศตรง ๆ) — วิธีที่ระบบนิเวศ Rust ใช้กันทั่วไปคือ**เช็คเงื่อนไขในโค้ด Rust เองด้วย
`compile_error!()`**

```toml
# cli/Cargo.toml
[features]
default = ["backend_memory"]
backend_memory = []
backend_sqlite = []
```

```rust
// src/main.rs — บรรทัดแรกสุดของไฟล์ (ก่อนโค้ดอื่นใด) เช็คว่าไม่มีการเปิดสอง backend พร้อมกัน
#[cfg(all(feature = "backend_memory", feature = "backend_sqlite"))]
compile_error!("เปิดได้แค่ backend เดียว: เลือก \"backend_memory\" หรือ \"backend_sqlite\" อย่างใดอย่างหนึ่งเท่านั้น");

fn main() {
    // ...
}
```

`compile_error!()` เป็น macro พิเศษที่ **บังคับให้ compile fail ทันทีพร้อมข้อความที่คุณกำหนด** ถ้าโค้ดตรงจุดนั้น
ถูก compile เข้ามาจริง (ผ่าน `#[cfg(...)]` คลุมไว้) — มันคือ item ชนิดหนึ่งเหมือนฟังก์ชัน/struct แต่ผลลัพธ์ของ
มันคือ "การ compile ล้มเหลวพร้อมข้อความนี้" ไม่ต่างจากการเขียน syntax ผิดตรงจุดนั้นเลย เพียงแต่ข้อความ error
เป็นสิ่งที่**คุณเลือกเองได้**และอ่านเข้าใจง่ายกว่า error message ทั่วไปของ compiler มาก

**ทดสอบจริง**: เมื่อเปิดทั้งสอง feature พร้อมกัน:

```bash
cargo check -p store_cli --features backend_memory,backend_sqlite
```

```
error: เปิดได้แค่ backend เดียว: เลือก "backend_memory" หรือ "backend_sqlite" อย่างใดอย่างหนึ่งเท่านั้น
 --> cli/src/main.rs:2:1
  |
2 | compile_error!("เปิดได้แค่ backend เดียว: เลือก \"backend_memory\" หรือ \"backend_sqlite\" ...");
  | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

error: could not compile `store_cli` (bin "store_cli") due to 1 previous error
```

ข้อความ error ชี้ตรงไปที่บรรทัดของ `compile_error!()` พร้อมข้อความภาษาไทยที่เราเขียนไว้เอง — ผู้ใช้ crate ของ
คุณที่พลาดเปิด feature ผิดจะเห็นคำแนะนำที่เข้าใจง่ายทันที **ไม่ต้องไปนั่งงงว่า error ประหลาด ๆ ที่เกิดตามมา
(เช่น "duplicate definition of struct X" เพราะทั้งสอง backend ประกาศ struct ชื่อเดียวกัน) มาจากไหน** — นี่คือ
เหตุผลที่ `compile_error!()` ควรอยู่**บรรทัดต้น ๆ ของไฟล์** เสมอ (ก่อนโค้ดอื่นที่อาจ error ตามมาจากสาเหตุเดียวกัน)
เพื่อให้ error ที่เกี่ยวข้องกันจริง ๆ ปรากฏก่อนเสมอ ไม่ปนกับ error อื่นที่เป็นผลพลอยได้จากการเปิด feature ผิด

**Pattern ที่ใช้บ่อยอีกแบบ**: บาง crate เลือกเช็คแบบ "ต้องมีอย่างน้อยหนึ่ง backend เปิดอยู่" (ไม่ใช่แค่ "ห้ามเปิด
พร้อมกัน") ด้วยเงื่อนไข `not(any(...))`:

```rust
#[cfg(not(any(feature = "backend_memory", feature = "backend_sqlite")))]
compile_error!("ต้องเปิด backend อย่างน้อยหนึ่งตัว: \"backend_memory\" หรือ \"backend_sqlite\"");
```

รวมสองเงื่อนไขเข้าด้วยกัน (ห้ามเปิดพร้อมกัน + ต้องเปิดอย่างน้อยหนึ่งตัว) ก็ทำได้โดยเขียน `compile_error!()`
สองบรรทัดคู่กัน ครอบด้วย `#[cfg(...)]` คนละแบบ — วิธีนี้ทำให้ feature flag ของ crate ทำงานเหมือนกับ "enum ที่
เลือกได้ทาง Cargo.toml" ซึ่งเป็น pattern ที่ crate จริงในระบบนิเวศ (เช่น crate ที่เลือก TLS backend ระหว่าง
`rustls` กับ `native-tls`) ใช้กันทั่วไป

### 35.12 Feature-Gated Modules: ครอบทั้งโมดูล ไม่ใช่แค่ฟังก์ชันเดียว

Part 17 หัวข้อ 17.12 แสดงตัวอย่าง `#[cfg(feature = "...")]` ครอบ**ฟังก์ชันเดียว** — แนวทางที่ scale ได้ดีกว่า
เมื่อความสามารถที่เกี่ยวกับ feature หนึ่งมีขนาดใหญ่ขึ้น (หลายฟังก์ชัน, หลาย struct) คือครอบ**ทั้งโมดูล**เข้าไป
เลย ทำให้จัดกลุ่มโค้ดที่เกี่ยวกับ feature นั้นไว้ที่เดียวชัดเจน อ่านง่ายกว่าไปแปะ `#[cfg(...)]` กระจัดกระจายทั่ว
ไฟล์:

```rust
// src/lib.rs
pub struct Product {
    pub sku: String,
    pub price_cents: u64,
    pub stock: u32,
}

impl Product {
    pub fn new(sku: impl Into<String>, price_cents: u64, stock: u32) -> Self {
        Product { sku: sku.into(), price_cents, stock }
    }

    pub fn total_value_cents(&self) -> u64 {
        self.price_cents * self.stock as u64
    }
}

// ทั้งโมดูล json_export (รวมทุกฟังก์ชัน/struct ภายใน) จะ "มีอยู่จริง" ในผลลัพธ์ compile
// ก็ต่อเมื่อเปิด feature "json" เท่านั้น — ไม่ต้องเขียน #[cfg(...)] แยกในแต่ละ item ภายในโมดูลนี้เลย
// เพราะการครอบที่ `mod` เดียวมีผลกับทุกอย่างที่อยู่ข้างในโดยอัตโนมัติ
#[cfg(feature = "json")]
pub mod json_export {
    use super::Product;
    use serde::Serialize;

    #[derive(Serialize)]
    struct ProductJson<'a> {
        sku: &'a str,
        price_cents: u64,
        stock: u32,
    }

    pub fn to_json(p: &Product) -> String {
        let wire = ProductJson { sku: &p.sku, price_cents: p.price_cents, stock: p.stock };
        serde_json::to_string(&wire).expect("Product ควร serialize ได้เสมอ")
    }

    pub fn to_json_pretty(p: &Product) -> String {
        let wire = ProductJson { sku: &p.sku, price_cents: p.price_cents, stock: p.stock };
        serde_json::to_string_pretty(&wire).expect("Product ควร serialize ได้เสมอ")
    }
}
```

**ข้อดีของการครอบทั้งโมดูลเทียบกับครอบทีละฟังก์ชัน**:

1. **อ่านง่ายกว่ามาก** — เห็น `#[cfg(feature = "json")] pub mod json_export { ... }` บรรทัดเดียว ก็รู้ทันทีว่า
   ทุกอย่างข้างในผูกกับ feature `json` ทั้งหมด ไม่ต้องไล่เช็คทุกฟังก์ชันว่ามี `#[cfg(...)]` ติดอยู่ครบหรือเปล่า
   (ถ้าครอบทีละฟังก์ชัน มีโอกาสลืมแปะ `#[cfg(...)]` ที่ฟังก์ชันใหม่ที่เพิ่มเข้ามาทีหลัง ทำให้ฟังก์ชันนั้นถูก
   compile เข้ามาเสมอทั้งที่ควรผูกกับ feature)
2. **`use` ภายในโมดูลก็ถูกครอบไปด้วยโดยอัตโนมัติ** — สังเกตว่า `use serde::Serialize;` อยู่ *ภายใน* `mod
   json_export` ไม่ใช่ที่ระดับบนสุดของไฟล์ — ถ้าเขียน `use serde::Serialize;` ไว้ที่ระดับบนสุดของ
   `lib.rs` โดยไม่ครอบ `#[cfg(...)]` จะได้ compile error ทันทีเมื่อไม่เปิด feature `json` (เพราะไม่มี `serde`
   ให้ `use` เลยตอนนั้น) — การจัดกลุ่ม `use` ไว้ในโมดูลที่ถูก cfg ไปด้วยกันจึงถูกต้องกว่าและปลอดภัยกว่า
3. **เพิ่ม item ใหม่ในอนาคตไม่ต้องกลัวลืม** — ทีมที่มาแก้โมดูลนี้ทีหลัง เพิ่มฟังก์ชันใหม่ในนี้ได้เลยโดยไม่ต้อง
   จำว่าต้องแปะ `#[cfg(feature = "json")]` ซ้ำทุกครั้ง เพราะการครอบที่ `mod` ทำหน้าที่นั้นให้ครบอัตโนมัติแล้ว

**รูปแบบที่ใช้บ่อยอีกแบบ**: แยกไฟล์ทั้งไฟล์ให้เป็นโมดูลที่ผูกกับ feature โดยใช้ `#[cfg(...)]` ที่จุดประกาศ `mod`
ในไฟล์ `lib.rs` (ไม่ต้องเปิดไฟล์นั้นด้วยซ้ำถ้าไม่เปิด feature):

```rust
// src/lib.rs
#[cfg(feature = "json")]
mod json_export; // ไฟล์ src/json_export.rs — cfg ตรงนี้ครอบทั้งไฟล์โดยไม่ต้องเปิดไฟล์นั้นมาแก้เลย

pub use json_export::to_json; // การ re-export ก็ต้อง cfg คู่กันด้วย ไม่อย่างนั้นจะ error ตอนไม่เปิด feature

#[cfg(feature = "json")]
pub use json_export::to_json;
```

วิธีนี้เหมาะกับกรณีที่ความสามารถของ feature มีขนาดใหญ่มากจนสมควรแยกเป็นไฟล์ของตัวเองต่างหาก (มากกว่า inline
`mod { ... }` แบบตัวอย่างก่อนหน้า)

### 35.13 Production Workspace Layout: core / api / cli / web

Part 17 หัวข้อ 17.6 แสดง workspace 3 member (`core`/`cli`/`web`) เป็นตัวอย่างพื้นฐาน — ในระดับ production จริง
โปรเจกต์ขนาดกลางถึงใหญ่มักขยายเป็น**4 ชั้นที่แยกความรับผิดชอบชัดเจนกว่านั้น**:

```
inventory_platform/
├── Cargo.toml                 <- workspace root: [workspace], [workspace.package], [workspace.dependencies]
├── Cargo.lock
├── .cargo/
│   └── config.toml            <- alias, env เฉพาะโปรเจกต์นี้
├── core/                        <- domain logic ล้วน ๆ ไม่มี I/O, ไม่มี framework ใด ๆ ผูกอยู่เลย
│   ├── Cargo.toml
│   └── src/lib.rs
├── api/                         <- HTTP layer: รับ request, แปลงเป็น core call, ตอบ response
│   ├── Cargo.toml               (depend on core + axum/tokio ที่ core ไม่ต้องแบก)
│   └── src/lib.rs
├── cli/                         <- โปรแกรม command-line สำหรับ operator/admin ใช้จัดการข้อมูลผ่าน terminal
│   ├── Cargo.toml               (depend on core + clap ที่ core/api ไม่ต้องแบก)
│   └── src/main.rs
└── web/                         <- เว็บ server จริง (binary ที่ประกอบ api layer เข้ากับ HTTP server เพื่อรัน)
    ├── Cargo.toml               (depend on core + api)
    └── src/main.rs
```

**เหตุผลของการแยกเป็น 4 ชั้นนี้ (ต่อจากที่ Part 17 หัวข้อ 17.5 อธิบายเหตุผลทั่วไปของการแยก workspace ไว้แล้ว)**:

- **`core`** คือ "แหล่งความจริงเดียว" (single source of truth) ของ business logic — ไม่ผูกกับวิธีที่ผลลัพธ์จะถูก
  ใช้ (HTTP, CLI, หรืออนาคตอาจมี GUI/gRPC เพิ่มมา) ทำให้ทดสอบได้ง่ายที่สุด (unit test ล้วน ๆ ไม่ต้อง mock HTTP
  request) และไม่แบก dependency หนัก ๆ ที่ layer อื่นต้องการ
- **`api`** คือ layer ที่แปลง HTTP request/response เป็นการเรียก `core` — แยกจาก `web` เพราะบางทีมอยากทดสอบ
  logic ของ handler โดยไม่ต้องรัน HTTP server จริง (เรียก handler function ตรง ๆ ใน test) และเผื่อในอนาคตอยาก
  ห่อ `api` เดียวกันด้วย framework คนละตัว (สลับจาก `axum` เป็น `actix-web`) โดยแก้แค่ `web` ไม่ต้องแก้ logic ของ
  handler ที่อยู่ใน `api`
- **`cli`** ให้ operator/admin จัดการข้อมูลผ่าน terminal โดยไม่ต้องเปิด HTTP server เลย (เช่น script migrate
  ข้อมูล, คำสั่ง seed ข้อมูลทดสอบ) — ไม่ต้องแบก `axum`/`tokio` ของ `api` เลยถ้า `cli` ไม่จำเป็นต้องมี async
  runtime
- **`web`** เป็น binary ที่ประกอบทุกอย่างเข้าด้วยกันจริง (เลือก HTTP framework, ตั้งค่า routing, เริ่ม server)
  — มันมักเป็น crate ที่ "บางที่สุด" ในทั้งหมด เพราะ logic จริง ๆ กระจายอยู่ใน `core`/`api` หมดแล้ว

การแยกแบบนี้ทำให้ **compile time ของแต่ละส่วนกระทบกันน้อยที่สุด** — ถ้าทีม backend แก้ `core` บ่อย ๆ แต่ไม่แตะ
`web`, การรัน `cargo build -p core` หรือ `cargo test -p core` ระหว่างพัฒนาจะไม่ต้อง compile `axum`/`tokio` ของ
`web`/`api` เลย เร็วกว่ามาก และเมื่อ publish ขึ้น crates.io ก็ทำได้เฉพาะ `core` ที่เป็น library ทั่วไป (ตามหลัก
`publish = false` ที่ Part 17 หัวข้อ 17.10 อธิบายไว้) โดยไม่ publish `api`/`cli`/`web` ที่เป็นโปรแกรมภายในของ
องค์กร

### 35.14 `[workspace.dependencies]`: รวมศูนย์เวอร์ชัน dependency ทั้ง workspace

นี่คือฟีเจอร์ที่สำคัญที่สุดของบทนี้สำหรับ workspace ขนาดจริง — Part 17 ยังไม่ได้พูดถึงเลย เพราะ workspace 2-3
member ในตัวอย่างของ Part 17 มี dependency น้อยมากจนยังไม่เห็นปัญหาที่ `[workspace.dependencies]` แก้

#### ปัญหา "ก่อน" ใช้ `[workspace.dependencies]`

ลองจินตนาการ workspace 4 member (`core`/`api`/`cli`/`web`) ที่ทุกตัวต้องใช้ `serde` (สำหรับ serialize ข้อมูล)
และ `anyhow` (สำหรับ error handling — จะเรียนเจาะลึกใน Part 31) โดยไม่มีการรวมศูนย์อะไรเลย แต่ละ member เขียน
`Cargo.toml` ของตัวเองแบบนี้:

```toml
# core/Cargo.toml
[dependencies]
serde = { version = "1.0.210", features = ["derive"] }
anyhow = "1.0.89"

# api/Cargo.toml
[dependencies]
serde = { version = "1.0.208", features = ["derive"] }   # เวอร์ชันเขียนไม่ตรงกับ core!
anyhow = "1.0.86"                                          # เวอร์ชันเขียนไม่ตรงกับ core!

# cli/Cargo.toml
[dependencies]
serde = { version = "1.0", features = ["derive"] }         # กว้างกว่าที่อื่น
anyhow = "1"

# web/Cargo.toml
[dependencies]
serde = { version = "1.0.210", features = ["derive"] }
anyhow = "1.0.89"
```

**ปัญหาที่เกิดขึ้นจริงเมื่อทีมใหญ่ทำแบบนี้**:

1. **พิมพ์เวอร์ชันไม่ตรงกันโดยไม่ตั้งใจ** (`1.0.210` vs `1.0.208` vs `1.0`) — แม้ทั้งหมดจะ resolve ไปที่เวอร์ชัน
   จริงเดียวกันได้ในที่สุด (เพราะ SemVer caret requirement ยอมรับช่วงที่ overlap กัน ตามที่ Part 2 หัวข้อ 2.8
   อธิบายไว้) แต่ **การอ่านโค้ดแล้วเห็นเวอร์ชันไม่ตรงกันทำให้เข้าใจผิดได้ง่าย** ว่าจงใจ pin คนละเวอร์ชันด้วย
   เหตุผลอะไรหรือเปล่า ทั้งที่จริง ๆ แค่พิมพ์ไม่ตรงกันเฉย ๆ
2. **อัปเกรดเวอร์ชันต้องแก้หลายไฟล์** — เมื่อ `serde` ออกเวอร์ชันใหม่ที่อยากอัปเกรดไป (เช่นเพื่อ security patch)
   ต้องไปแก้ `Cargo.toml` ของทุก member ที่ใช้ `serde` ทีละไฟล์ ถ้าลืมแก้ตัวใดตัวหนึ่ง อาจเกิดสถานการณ์ที่
   member นั้นยัง pin เวอร์ชันเก่าไว้แบบไม่ตั้งใจ (ยิ่งอันตรายถ้าเป็น exact pin `=1.0.86` ที่ไม่ได้อัปเกรดตาม)
3. **feature ไม่ตรงกันโดยไม่รู้ตัว** — สมมติ `core` ลืมเปิด feature `derive` ในขณะที่ตัวอื่นเปิดหมด (จุดบกพร่อง
   เล็ก ๆ ที่มองข้ามได้ง่ายเวลามีไฟล์เยอะ) — แม้ feature unification (หัวข้อ 35.10) จะทำให้สุดท้าย `serde` ที่
   compile จริงมี `derive` เปิดอยู่ดี (เพราะ member อื่นขอไว้) แต่ `core` เองจะดู "เหมือน" ไม่ได้ใช้ `derive`
   ทำให้ตอนอ่านโค้ดของ `core` เดี่ยว ๆ (โดยไม่รู้ context ของทั้ง workspace) เข้าใจผิดว่า `derive` ไม่จำเป็น

#### วิธีแก้: `[workspace.dependencies]`

ประกาศเวอร์ชัน (และ feature ร่วม) ไว้**ครั้งเดียวที่ root workspace** แล้วให้ทุก member "สืบทอด" มาด้วยคีย์
`workspace = true`:

```toml
# inventory_platform/Cargo.toml (root)
[workspace]
resolver = "2"
members = ["core", "api", "cli", "web"]

[workspace.dependencies]
serde = { version = "1.0.210", features = ["derive"] }
anyhow = "1.0.89"
axum = "0.7"
tokio = { version = "1", features = ["full"] }
clap = { version = "4", features = ["derive"] }
```

ทุก member เขียนแค่:

```toml
# core/Cargo.toml
[dependencies]
serde.workspace = true
anyhow.workspace = true
```

```toml
# api/Cargo.toml
[dependencies]
store_core = { path = "../core" }
serde.workspace = true
anyhow.workspace = true
axum.workspace = true
tokio.workspace = true
```

สังเกตไวยากรณ์ `serde.workspace = true` — นี่คือ **dotted key syntax** เทียบเท่ากับเขียนแบบ inline table
`serde = { workspace = true }` ทุกประการ (เลือกใช้แบบไหนก็ได้ตามความชอบ เหมือนที่ Part 17 หัวข้อ 17.11 อธิบาย
เรื่อง inline table vs expanded table ไว้) คีย์ `workspace = true` บอก Cargo ว่า **"ไปเอา version/features ของ
dependency ชื่อนี้จาก `[workspace.dependencies]` ที่ root มา ห้ามระบุ version ซ้ำที่นี่เองอีก"**

**ผลลัพธ์ที่ได้**:

1. **อัปเกรดเวอร์ชันแก้ที่เดียว** — เปลี่ยน `serde = { version = "1.0.210", ... }` เป็น `"1.0.215"` ที่บรรทัด
   เดียวใน root `Cargo.toml` มีผลกับทุก member ที่เขียน `serde.workspace = true` ทันที ไม่มีทางลืมแก้ member
   ใดม์ตัวหนึ่งเพราะไม่มีให้แก้แยกอยู่แล้ว
2. **การันตีว่าทุก member เห็นเวอร์ชัน/feature เดียวกันเป๊ะ** — ไม่มีการพิมพ์คลาดเคลื่อนระหว่างไฟล์ได้อีก เพราะ
   ไม่มีการพิมพ์เวอร์ชันซ้ำเลยนอกจากที่ root
3. **member ยังเพิ่ม feature เฉพาะตัวทับได้** ถ้าต้องการ feature เพิ่มจากที่ root ประกาศไว้ (ใช้คู่กับ `features`
   ปกติ ซึ่งจะถูก**รวม**เข้ากับ feature ที่ root ประกาศไว้ ไม่ใช่แทนที่):

```toml
# api/Cargo.toml — ต้องการ feature "rc" เพิ่มจาก serde นอกจากที่ root ประกาศไว้ (derive)
[dependencies]
serde = { workspace = true, features = ["rc"] }
```

`serde` ที่ `api` ได้จริงจะมีทั้ง `derive` (จาก root) และ `rc` (ที่ `api` เพิ่มเอง) รวมกัน — เป็นการ union แบบ
เดียวกับ feature unification ในหัวข้อ 35.10 แต่คราวนี้เกิดที่ระดับ `workspace.dependencies` เอง

**ข้อจำกัดที่ต้องรู้**: ถ้า member เขียน `serde.workspace = true` แต่ **`serde` ไม่ได้ถูกประกาศไว้ใน
`[workspace.dependencies]` ที่ root เลย** (เช่น พิมพ์ชื่อผิด หรือลืมเพิ่มที่ root) จะได้ error ทันทีตอน
`cargo check`:

```
error: failed to parse manifest at `/path/to/api/Cargo.toml`

Caused by:
  error inheriting `anyhow` from workspace root manifest's `workspace.dependencies.anyhow`

Caused by:
  `dependency.anyhow` was not found in `workspace.dependencies`
```

**ทดสอบจริง**: เพิ่มบรรทัด `anyhow = { workspace = true }` ใน `api/Cargo.toml` ของ workspace ตัวอย่าง (ที่ยัง
ไม่ได้ประกาศ `anyhow` ไว้ใน `[workspace.dependencies]` ที่ root) แล้วรัน `cargo check -p store_api` — ได้
error ข้อความข้างต้นทุกตัวอักษร (เอามาจากการทดสอบจริงบนเครื่อง) ข้อความนี้ชัดเจนมากว่าปัญหาคือ **"หา
`anyhow` ใน `workspace.dependencies` ไม่เจอ"** — วิธีแก้คือเพิ่ม `anyhow = "..."` เข้า
`[workspace.dependencies]` ที่ root ให้ครบก่อนที่ member จะ inherit มันได้

### 35.15 `[workspace.package]`: แชร์ package metadata ร่วมกัน

คู่กับ `[workspace.dependencies]` คือ `[workspace.package]` — ใช้รวมศูนย์ **metadata ของ package** ที่ member
ส่วนใหญ่มักมีค่าเหมือนกัน (`version`, `edition`, `authors`, `license`, `repository`, `description` บางส่วน)
เพื่อไม่ต้องพิมพ์ซ้ำทุก member:

```toml
# inventory_platform/Cargo.toml (root)
[workspace.package]
version = "0.3.0"
edition = "2021"
authors = ["Inventory Platform Team <platform@example.com>"]
license = "MIT OR Apache-2.0"
repository = "https://github.com/example/inventory-platform"
```

แต่ละ member สืบทอดคีย์ที่ต้องการผ่าน dotted key `.workspace = true` เหมือนกับ dependency:

```toml
# core/Cargo.toml
[package]
name = "inventory_core"
version.workspace = true
edition.workspace = true
authors.workspace = true
license.workspace = true
repository.workspace = true
description = "Core domain logic สำหรับระบบจัดการคลังสินค้า"   # description ไม่ inherit — เขียนเฉพาะของตัวเอง
```

**สังเกตว่า `name` ไม่มีให้ inherit** — เพราะ `name` เป็นสิ่งที่**ต้องต่างกันในแต่ละ package เสมอ** (คือตัวระบุ
package แต่ละใบ) จึงไม่มีเหตุผลที่จะรวมศูนย์มัน ส่วน `description` ในตัวอย่างนี้เลือกไม่ inherit เพราะแต่ละ
member มักมีคำอธิบายเฉพาะตัวที่ต่างกัน (`core` อธิบายว่าเป็น domain logic, `cli` อธิบายว่าเป็นเครื่องมือ
command-line) — คุณเลือกได้ว่าคีย์ไหนควรรวมศูนย์ (ค่าที่เหมือนกันทุก member จริง ๆ) และคีย์ไหนควรเขียนแยก (ค่า
ที่ควรต่างกันตามธรรมชาติของแต่ละ member)

**ทำไม `version.workspace = true` มีประโยชน์มากในทางปฏิบัติ?** เพราะหลายทีมเลือก**ผูก version ของทุก member
ในเครือไว้ด้วยกัน** (bump version ของทั้งชุดพร้อมกันเสมอตอน release แม้บาง member จะไม่มีอะไรเปลี่ยนเลยก็ตาม —
กลยุทธ์นี้เรียกว่า "lockstep versioning" ตรงข้ามกับ "independent versioning" ที่แต่ละ member มี version ของ
ตัวเองแยกอิสระตามที่ Part 17 หัวข้อ 17.10 อธิบายไว้) ถ้าเลือกกลยุทธ์ lockstep การอัปเดต version ทำได้ที่บรรทัด
เดียวใน root แทนที่จะไล่แก้ทุก `Cargo.toml` ของทุก member — เหมาะกับ workspace ที่ member ทั้งหมด release
พร้อมกันเสมอ (เช่น mono-repo ขององค์กรเดียว ไม่ใช่ library แยกที่ publish อิสระให้คนนอกใช้)

**ข้อควรระวัง**: ถ้าโปรเจกต์คุณเลือกกลยุทธ์ "independent versioning" (แต่ละ member มี release cycle ของตัวเอง
เช่น `core` เป็น library ทั่วไปที่ publish บ่อยกว่า `cli` ภายในที่แทบไม่ bump version เลย) การ inherit `version`
จาก workspace แบบนี้จะ**ไม่เหมาะ** — ควรให้แต่ละ member เขียน `version` ของตัวเองตรง ๆ แทน แล้ว inherit แค่คีย์
ที่เหมือนกันจริง ๆ อย่าง `edition`/`license`/`authors` เท่านั้น

### 35.16 Cargo.lock ทบทวนอีกครั้ง: โครงสร้างไฟล์และ Resolver

Part 2 หัวข้อ 2.6 อธิบายแนวคิดพื้นฐานของ `Cargo.lock` ไว้แล้ว (ล็อกเวอร์ชันที่ resolve ได้เพื่อ build ซ้ำได้)
ตอนนี้มาดูรายละเอียดเชิงลึกกว่านั้น

#### อ่านโครงสร้างไฟล์ `Cargo.lock` จริง

```toml
# This file is automatically @generated by Cargo.
# It is not intended for manual editing.
version = 4

[[package]]
name = "itoa"
version = "1.0.18"
source = "registry+https://github.com/rust-lang/crates.io-index"
checksum = "8f42a60cbdf9a97f5d2305f08a87dc4e09308d1276d28c869c684d7777685682"

[[package]]
name = "proc-macro2"
version = "1.0.107"
source = "registry+https://github.com/rust-lang/crates.io-index"
checksum = "985e7ec9bb745e6ce6535b544d84d6cd6f7ad8bd711c398938ae983b91a766d9"
dependencies = [
 "unicode-ident",
]

[[package]]
name = "serde"
version = "1.0.229"
source = "registry+https://github.com/rust-lang/crates.io-index"
checksum = "4148590afebada386688f18773da617792bf2ef03ffc1e4cbd2b1d45b023e0ba"
dependencies = [
 "serde_core",
 "serde_derive",
]
```

**อธิบายทีละส่วน**:

- **`version = 4`** ที่บรรทัดบนสุด — นี่คือ**เวอร์ชันของ*ฟอร์แมตไฟล์*** `Cargo.lock` เอง (ไม่เกี่ยวกับ resolver
  version ที่จะพูดถึงต่อไป) Cargo ใหม่กว่าอ่านฟอร์แมตเก่ากว่าได้เสมอ แต่ Cargo เก่ามากอาจอ่านฟอร์แมตใหม่ไม่ได้
  — นี่เป็นอีกเหตุผลที่ทีมควรใช้ Cargo เวอร์ชันใกล้เคียงกันในทีมเดียวกัน (ผ่าน `rust-toolchain.toml` ที่จะพูดถึง
  ในบทหลัง ๆ ของหลักสูตร)
- **`[[package]]`** — array-of-tables (สังเกตวงเล็บสองชั้น เหมือน `[[bin]]` ที่ Part 17 เคยเจอ) หนึ่ง block
  ต่อหนึ่ง package ที่ resolve ได้ในทั้ง dependency graph รวมทั้ง package ของคุณเองและทุก transitive
  dependency
- **`source`** — บอกว่า package นี้มาจากไหน (`registry+https://...` คือมาจาก crates.io ปกติ ส่วน path
  dependency ภายใน workspace จะ**ไม่มี** `source` เลย เพราะไม่ได้มาจาก registry ใด ๆ)
- **`checksum`** — hash SHA-256 ของไฟล์ `.crate` ที่ดาวน์โหลดมา ใช้ยืนยันว่าไฟล์ที่ดาวน์โหลดมาไม่ถูกแก้ไข/
  สับเปลี่ยนระหว่างทาง (คล้าย `integrity` field ใน `package-lock.json` ของ npm) — ถ้า checksum ไม่ตรงกับที่
  บันทึกไว้ (เช่นถูก man-in-the-middle attack ระหว่างดาวน์โหลด) Cargo จะปฏิเสธและแจ้ง error ทันที
- **`dependencies`** — array ของชื่อ package ที่ package นี้ต้องพึ่งพา (transitive dependency ของ package นี้)
  — สังเกตว่าไม่มี version ระบุตรงนี้ (แค่ชื่อ) เพราะแต่ละชื่อ package หนึ่งชื่อจะ resolve ไปที่ **version เดียว**
  ที่ประกาศไว้ใน `[[package]]` block ของชื่อนั้นในไฟล์เดียวกันนี้เอง (ยกเว้นกรณีพิเศษที่มี major version ต่างกัน
  ของ crate เดียวกันอยู่ในกราฟพร้อมกันจริง ๆ ซึ่งจะมี block `[[package]]` แยกกันคนละ version)

#### Resolver v1, v2, v3: อะไรเปลี่ยนไปบ้าง

Part 17 หัวข้อ 17.6 อธิบายไว้แล้วว่า `resolver = "2"` แก้ปัญหาของ resolver v1 ที่ "รวม feature ของทุก target
เข้าด้วยกันแบบไม่แยกแยะ" — มาดูรายละเอียดครบทั้ง 3 เวอร์ชัน:

| Resolver | เริ่มใช้เมื่อ | พฤติกรรม feature unification |
|---|---|---|
| **v1** (default เดิม) | Cargo รุ่นแรก ๆ | รวม feature ของ `[dependencies]`, `[dev-dependencies]`, `[build-dependencies]` เข้าด้วยกันหมดสำหรับ dependency ตัวเดียวกัน แม้ target จะต่างกัน (production build vs test build) |
| **v2** | คู่กับ edition 2021 | แยก feature ของ `[dev-dependencies]`/`[build-dependencies]` ออกจาก `[dependencies]` ปกติ — dependency ที่ถูกใช้ทั้งใน production และใน dev-dependency ด้วย feature ต่างกัน จะได้ feature ที่ตรงกับ target จริง ๆ ไม่ใช่รวมกันแบบเหมาโหมด |
| **v3** | คู่กับ edition 2024 | ปรับปรุงเพิ่มเติมจาก v2 ในกรณี cross-compile (แยก feature ของ dependency ฝั่ง host platform กับ target platform ให้ชัดเจนขึ้นกว่า v2) และปรับพฤติกรรมบางจุดของ `[patch]`/`[replace]` ให้สอดคล้องกันมากขึ้น |

**ตัวอย่างที่ resolver v1 มีปัญหาแบบเจาะจง**: สมมติ package หนึ่งมี `[dependencies]` ใช้ `regex` แบบปิด
default feature (เพื่อ binary เล็ก ไม่ต้องมี Unicode table เต็มรูปแบบ) แต่ `[dev-dependencies]` ใช้ `regex` ตัว
เดียวกันแบบเปิด feature เต็ม (เพราะ test ต้องการความสามารถ Unicode เต็มที่) — ด้วย **resolver v1** ผลลัพธ์คือ
`regex` ที่ compile เข้า **production binary จริง** (ตอน `cargo build --release` แบบไม่รวม test) จะดันมี
feature Unicode เต็มติดมาด้วย (เพราะ v1 รวม feature ของทุก target ปนกันหมด แม้ target production ไม่ได้ขอ
feature นั้นเลย) ทำให้ binary ใหญ่กว่าที่ตั้งใจไว้โดยไม่รู้ตัว — resolver v2/v3 แก้ปัญหานี้โดยตรง: production
build จะได้ `regex` แบบปิด feature ตามที่ `[dependencies]` ขอจริง ๆ ไม่ปนกับที่ `[dev-dependencies]` ขอเพิ่ม

**กฎการใช้งานจริง**: **ทุก package/workspace ใหม่ควรตั้ง `resolver = "2"` เสมอ** (หรือปล่อยให้ Cargo เลือกให้
อัตโนมัติตาม edition ก็ได้ ถ้าเป็น package เดี่ยวที่ edition 2021+ — แต่สำหรับ workspace ต้องเขียนชัดเจนที่ root
เสมอตามที่ Part 17 อธิบายไว้) เพราะพฤติกรรมของ v2/v3 ตรงกับสัญชาตญาณของโปรแกรมเมอร์มากกว่า v1 ในเกือบทุกกรณี —
เหตุผลเดียวที่ยังต้องรู้จัก v1 คือถ้าไปเจอโปรเจกต์เก่ามาก ๆ ที่ไม่ได้ตั้ง `resolver` ไว้เลยและยังใช้ edition
2018 หรือเก่ากว่า จะได้เข้าใจว่าทำไม production binary ของโปรเจกต์นั้นอาจมี dependency ที่ดูเหมือนไม่จำเป็นติด
มาด้วย

### 35.17 `cargo update`: อัปเดตเวอร์ชันใน Cargo.lock อย่างถูกวิธี

Part 2 บอกไว้แล้วว่า `Cargo.lock` ล็อกเวอร์ชันไว้เพื่อ reproducible build — แต่เมื่อถึงเวลาที่**ต้องการ**อัปเดต
เวอร์ชัน (ได้ security patch ใหม่, bug fix ใหม่) มีคำสั่งย่อยหลายแบบให้เลือกตามความละเอียดที่ต้องการควบคุม

#### `cargo update`: อัปเดตทุก dependency ภายในขอบเขต SemVer ที่ `Cargo.toml` อนุญาต

```bash
cargo update
```

อัปเดต**ทุก dependency ในกราฟ**ไปยังเวอร์ชันล่าสุดที่ยังตรงกับ version requirement ที่เขียนไว้ใน `Cargo.toml`
(เช่นถ้าเขียน `serde = "1.0"` ไว้ และมี `1.0.230` ออกใหม่ล่าสุดบน crates.io จะได้ `1.0.230` มา แต่จะไม่ข้ามไป
`2.0.0` แม้จะออกมาแล้วก็ตาม เพราะ caret requirement ไม่ยอมรับ major version ที่เปลี่ยน — ตรงตามหลัก SemVer ที่
Part 2 หัวข้อ 2.8 อธิบายไว้) เหมาะกับการอัปเดตแบบ "รับ patch/minor update ล่าสุดของทุกอย่างพร้อมกัน" เป็นประจำ
(เช่น รันทุกสัปดาห์/เดือนเพื่อรับ security patch)

#### `cargo update -p <crate>`: อัปเดตแค่ crate เดียว (และ dependency ของมัน)

```bash
cargo update -p serde
```

จำกัดการอัปเดตให้แค่ `serde` (และ dependency ที่ `serde` เองต้องพึ่งพาที่จำเป็นต้องขยับตาม) โดย**ไม่แตะ**
dependency อื่นในกราฟที่ไม่เกี่ยวข้องเลย มีประโยชน์มากเมื่อคุณรู้ว่ามี security advisory เจาะจงกับ crate ตัวใด
ตัวหนึ่ง แล้วอยากอัปเดตเฉพาะตัวนั้นโดยไม่เสี่ยงให้ dependency อื่นทั้งหมดขยับเวอร์ชันไปด้วย (ซึ่งอาจนำบั๊กใหม่ที่
ไม่เกี่ยวข้องเข้ามาโดยไม่ตั้งใจ) — ทดสอบด้วย `--dry-run` ก่อนอัปเดตจริงเสมอเพื่อดูว่าจะมีอะไรเปลี่ยนบ้าง:

```bash
cargo update -p serde --dry-run
```

```
    Updating crates.io index
     Locking 0 packages to latest compatible versions
warning: not updating lockfile due to dry run
```

(ในตัวอย่างนี้ไม่มีอะไรให้อัปเดตเพราะ `serde` เป็นเวอร์ชันล่าสุดที่ตรงกับ requirement อยู่แล้ว)

#### `cargo update --precise <version>`: ปักหมุดเวอร์ชันเจาะจงเป๊ะ ๆ

```bash
cargo update -p serde_json --precise 1.0.140
```

บังคับให้ `serde_json` ใน `Cargo.lock` เป็นเวอร์ชัน `1.0.140` **เป๊ะ** (ต้องยังอยู่ในขอบเขตที่ version
requirement ของ `Cargo.toml` ยอมรับด้วย ไม่อย่างนั้นจะ error) มีประโยชน์เมื่อคุณรู้ว่าเวอร์ชันล่าสุดมีบั๊ก
เจาะจงที่กระทบโปรเจกต์คุณ แล้วอยาก "ถอยกลับ" (downgrade) ไปเวอร์ชันก่อนหน้าที่รู้ว่าใช้งานได้ดี โดยไม่ต้องรอ
เวอร์ชันใหม่ที่แก้บั๊กออกมา

**ทดสอบจริง** (`--dry-run` เพื่อดูผลกระทบก่อนอัปเดตจริง): ถอย `serde_json` จาก `1.0.151` กลับไปที่ `1.0.140`
ใน workspace ตัวอย่าง:

```
    Updating crates.io index
      Adding ryu v1.0.23
 Downgrading serde_json v1.0.151 -> v1.0.140
    Removing zmij v1.0.23
warning: not updating lockfile due to dry run
```

สังเกตว่าการเปลี่ยนแค่เวอร์ชันของ `serde_json` ตัวเดียว**กระทบ transitive dependency ของมันด้วย** — เวอร์ชัน
`1.0.140` ใช้ crate ชื่อ `ryu` เป็น dependency ภายใน (สำหรับแปลง floating-point เป็น string) ในขณะที่เวอร์ชัน
`1.0.151` เปลี่ยนไปใช้ crate ชื่ออื่นแทนสำหรับงานเดียวกัน — นี่คือตัวอย่างจริงที่ตอกย้ำว่า **การ pin เวอร์ชัน
เจาะจงของ crate ตัวเดียวอาจทำให้ dependency graph ทั้งชุดขยับตามได้ ไม่ใช่แค่เปลี่ยนตัวเลขเวอร์ชันของ crate นั้น
ตัวเดียวลอย ๆ** — ควรตรวจสอบผลลัพธ์ด้วย `--dry-run` และรัน `cargo test --workspace` ให้ครบหลังอัปเดตจริงเสมอ
ก่อน commit `Cargo.lock` ใหม่

**สรุปเมื่อไรใช้อะไร**:

| สถานการณ์ | คำสั่งที่ใช้ |
|---|---|
| อัปเดตทุกอย่างตามรอบปกติ (weekly/monthly) | `cargo update` |
| รู้ว่ามี security patch เจาะจงกับ crate ตัวเดียว | `cargo update -p <crate>` |
| ต้องการเวอร์ชันเจาะจงเป๊ะ ๆ (downgrade เพื่อเลี่ยงบั๊ก, หรือ pin ตาม audit requirement) | `cargo update -p <crate> --precise <version>` |
| อยากดูผลกระทบก่อนอัปเดตจริง | เพิ่ม `--dry-run` เข้ากับคำสั่งข้างบนแบบไหนก็ได้ |

### 35.18 `.cargo/config.toml`: ตั้งค่า Cargo เฉพาะโปรเจกต์/เฉพาะเครื่อง

`.cargo/config.toml` คือไฟล์ config ที่ควบคุมพฤติกรรมของ**ตัว Cargo เอง** (ต่างจาก `Cargo.toml` ที่อธิบาย
package/workspace) — Cargo จะมองหาไฟล์นี้ไล่จากโฟลเดอร์ปัจจุบันขึ้นไปเรื่อย ๆ จนถึง root ของ filesystem รวมทั้ง
ที่ `$CARGO_HOME/config.toml` (ปกติคือ `~/.cargo/config.toml`) ด้วย — ถ้ามีหลายไฟล์พบตามทาง **ค่าจะถูก merge
กัน** โดยไฟล์ที่อยู่ใกล้โปรเจกต์กว่ามีสิทธิ์ override ค่าของไฟล์ที่อยู่ไกลกว่า (เช่น config ระดับ project เฉพาะ
override config ระดับ user ที่ home directory ได้)

#### `[alias]`: ตั้งชื่อย่อให้คำสั่งที่พิมพ์บ่อย

```toml
# .cargo/config.toml
[alias]
ci = "check --workspace --all-targets"
covtest = "test --workspace --all-features"
b = "build"
```

หลังจากนี้พิมพ์ `cargo ci` จะเทียบเท่ากับพิมพ์ `cargo check --workspace --all-targets` เต็ม ๆ — ทดสอบจริง:

```bash
cargo ci
```

```
    Checking store_core v0.3.0 (...)
    Checking store_api v0.3.0 (...)
    Checking store_cli v0.3.0 (...)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.06s
```

ทำงานเหมือนพิมพ์คำสั่งเต็มทุกประการ — alias มีประโยชน์มากสำหรับคำสั่งที่ทีมพิมพ์ซ้ำ ๆ ทุกวันแต่ยาวเกินจะพิมพ์
เต็มทุกครั้ง (`--all-targets` ตรวจสอบทั้ง binary, test, example, bench ให้ครบในคำสั่งเดียว ไม่ใช่แค่ library
เฉย ๆ) — หลายทีมตั้ง `ci` เป็น alias มาตรฐานให้ตรงกับคำสั่งที่ CI pipeline รันจริง เพื่อให้ developer รันคำสั่ง
เดียวกันกับที่ CI จะรันได้ง่าย ๆ บนเครื่องตัวเองก่อน push

#### `[env]`: ตั้ง environment variable ให้ทุกคำสั่ง cargo

```toml
[env]
STORE_PLATFORM_ENV = "development"
DATABASE_URL = { value = "postgres://localhost/store_dev", force = false }
```

ตัวแปรใน `[env]` จะถูกตั้งให้กับ**ทุก process ที่ cargo สั่งรัน** (ตัว `rustc` เอง, `cargo run`, `cargo test`,
build script) — key ธรรมดา (`STORE_PLATFORM_ENV = "development"`) ตั้งค่าตรง ๆ ส่วน expanded table syntax
(`{ value = "...", force = false }`) ให้ควบคุมเพิ่มว่า **`force = false`** (ค่า default) หมายถึง "ตั้งค่านี้
ก็ต่อเมื่อ environment ของเครื่องยังไม่มีตัวแปรชื่อนี้อยู่ก่อน" (ให้ environment variable จริงของเครื่อง override
ได้เสมอถ้ามี) ในขณะที่ `force = true` จะบังคับทับค่าที่มีอยู่แล้วในเครื่องเสมอ

มีประโยชน์มากสำหรับตั้งค่าเริ่มต้นที่นักพัฒนาใหม่ในทีม clone repo มาแล้วรัน `cargo run` ได้ทันทีโดยไม่ต้องไปตั้ง
environment variable เองก่อน (ลด friction ตอน onboarding)

#### `[build]`: ตั้ง target หรือ jobs (จำนวน thread compile) เริ่มต้น

```toml
[build]
target = "x86_64-unknown-linux-musl"   # ตั้ง compile target เริ่มต้นให้ทุกคำสั่ง cargo build/run/test
jobs = 4                                 # จำกัดจำนวน parallel job ตอน compile (ค่า default คือจำนวน CPU core)
```

`target` มีประโยชน์มากเมื่อโปรเจกต์ต้อง cross-compile ไปยัง target เฉพาะเป็นประจำ (เช่น compile เป็น static
binary ด้วย `musl` libc สำหรับ deploy ใน container ที่ไม่มี `glibc`) — ตั้งไว้ที่นี่ครั้งเดียว ไม่ต้องพิมพ์
`--target x86_64-unknown-linux-musl` ต่อท้ายทุกคำสั่งเอง

#### `[source]`: ตั้ง registry mirror/alias

```toml
[source.crates-io]
replace-with = "my-mirror"

[source.my-mirror]
registry = "https://my-internal-mirror.example.com/index"
```

ใช้ในองค์กรที่มี **internal mirror ของ crates.io** (เพื่อความเร็ว, ควบคุม supply chain, หรือทำงานในเครือข่าย
ที่ปิดกั้นการเข้าถึง crates.io จริงจากอินเทอร์เน็ตภายนอก) — ทุกครั้งที่ Cargo จะไปดึง crate จาก crates.io มัน
จะถูก "redirect" ไปที่ mirror ที่ตั้งไว้แทนโดยอัตโนมัติ โดยไม่ต้องแก้ `Cargo.toml` ของทุกโปรเจกต์เลย (เพราะ
`Cargo.toml` อ้างถึง "crates-io" เป็นชื่อ source เชิงตรรกะ ไม่ใช่ URL ตรง ๆ — `.cargo/config.toml` เป็นตัว
กำหนดว่าชื่อเชิงตรรกะนั้นชี้ไปที่ URL จริงอันไหน)

**ระดับของไฟล์ config**: `.cargo/config.toml` วางที่**root ของ workspace/repository** จะมีผลกับทุกคนที่ clone
repo แล้วรันคำสั่งในนั้น (ควร commit เข้า git ถ้าเป็นการตั้งค่าที่ทุกคนในทีมควรได้เหมือนกัน เช่น `[alias]`) ส่วน
`~/.cargo/config.toml` (ระดับ user, ไม่อยู่ใน repo ไหนเลย) เหมาะกับการตั้งค่าเฉพาะตัว เช่น registry mirror ที่
คุณใช้ส่วนตัว หรือ `target` ที่เกี่ยวกับเครื่องของคุณคนเดียว

### 35.19 `build.rs`: รู้จักหน้าที่พื้นฐาน (ไม่ลงรายละเอียดเต็มรูปแบบ)

Part 2 หัวข้อ `[build-dependencies]` เกริ่นไว้แล้วว่า `build.rs` คือไฟล์พิเศษที่ compile และรัน**ก่อน**โค้ดหลัก
ของแพ็กเกจ — บทนี้จะแค่ให้เห็นภาพว่ามันหน้าตาเป็นอย่างไรและใช้ทำอะไรได้บ้าง (รายละเอียดเต็มรูปแบบเกี่ยวกับ FFI/
code generation ขั้นสูงเกินสโคปของบทนี้ อยู่นอกเหนือหลักสูตรระดับกลาง)

วาง `build.rs` ไว้ที่ **root ของ package** (ระดับเดียวกับ `Cargo.toml`, ไม่ใช่ใน `src/`) Cargo จะ detect และรัน
มันให้อัตโนมัติโดยไม่ต้องประกาศอะไรเพิ่มเติม:

```rust
// build.rs — ตัวอย่างง่าย ๆ: ฝัง git commit hash ปัจจุบันเข้าไปเป็น environment variable
// ที่โค้ดหลักอ่านได้ตอน compile-time ผ่าน env!() macro
use std::process::Command;

fn main() {
    let git_hash = Command::new("git")
        .args(["rev-parse", "--short", "HEAD"])
        .output()
        .map(|o| String::from_utf8_lossy(&o.stdout).trim().to_string())
        .unwrap_or_else(|_| "unknown".to_string());

    // directive พิเศษที่ cargo อ่านจาก stdout ของ build.rs เพื่อตั้ง environment variable
    // ให้กับขั้นตอน compile โค้ดหลักของ package นี้เท่านั้น (ไม่กระทบ dependency อื่น)
    println!("cargo::rustc-env=GIT_HASH={git_hash}");

    // บอก cargo ว่า build script นี้ต้อง "รันใหม่" ถ้าไฟล์ .git/HEAD เปลี่ยน
    // (ไม่อย่างนั้น cargo จะ cache ผลลัพธ์ของ build script ไว้ ไม่รันซ้ำทุกครั้งที่ build)
    println!("cargo::rerun-if-changed=.git/HEAD");
}
```

```rust
// src/main.rs — อ่านค่าที่ build.rs ตั้งไว้ผ่าน env!() macro (ทำงานตอน compile-time ไม่ใช่ runtime)
fn main() {
    println!("build จาก git commit: {}", env!("GIT_HASH"));
}
```

**หน้าที่หลักที่ `build.rs` มักถูกใช้ทำในระบบนิเวศ Rust จริง**:

1. **Code generation** — เช่น compile ไฟล์ `.proto` (protocol buffer schema) เป็นโค้ด Rust ด้วย
   `tonic-build` ก่อนจะ compile โค้ดหลัก (ที่ Part 2 พูดถึงไว้แล้ว จะเจาะลึกเมื่อเรียน gRPC ใน Part 80)
2. **Link กับ library ภาษา C** — ผ่าน directive `cargo::rustc-link-lib`/`cargo::rustc-link-search` เพื่อบอก
   linker ว่าต้อง link กับ `.so`/`.a` ไฟล์ไหนบ้าง (จะเจาะลึกเมื่อเรียน FFI ใน Part 43)
3. **ตั้งค่า compile-time environment variable** — แบบตัวอย่างข้างบน (git hash, build timestamp, ฟีเจอร์ของ
   ระบบปฏิบัติการที่ตรวจพบตอน compile)
4. **ตรวจสอบ system dependency ก่อน compile** — เช่นเช็คว่าเครื่องมี library ระบบที่จำเป็นติดตั้งอยู่หรือไม่
   ถ้าไม่มีให้ fail ตั้งแต่ตอน build ด้วยข้อความที่ชัดเจน แทนที่จะไป fail ตอน link ด้วย error ที่งงกว่ามาก

**ข้อควรรู้เชิงเทคนิคสั้น ๆ**: `build.rs` ถูก compile และรันบน**host machine** (เครื่องที่กำลัง compile อยู่)
เสมอ ไม่ใช่ target machine (เครื่องปลายทางที่ binary สุดท้ายจะไปรัน) — สำคัญมากตอน cross-compile เพราะ
dependency ที่ `build.rs` ใช้ (`[build-dependencies]`) ต้อง compile ได้บน host แม้ตัวโปรแกรมหลักจะ target
เป็นแพลตฟอร์มอื่น (เช่น ARM embedded) ก็ตาม — นี่คือเหตุผลที่ Part 2 อธิบายไว้ว่า `[build-dependencies]` compile
แยกจาก `[dependencies]` โดยสิ้นเชิง

### 35.20 `cargo install`, `cargo tree`, และ `cargo audit`

#### `cargo install`: ติดตั้ง binary crate เป็นเครื่องมือ command-line

```bash
cargo install ripgrep
```

ดาวน์โหลด, compile (แบบ release เสมอ โดย default), แล้วติดตั้ง binary ที่ได้ไปที่ `$CARGO_HOME/bin`
(ปกติคือ `~/.cargo/bin`) ให้เรียกใช้จาก command line ได้เหมือนโปรแกรมทั่วไป (ต้องมี `~/.cargo/bin` อยู่ใน
`PATH` — `rustup` มักตั้งให้อัตโนมัติตอนติดตั้งครั้งแรกตาม Part 1) ต่างจาก `cargo add` ที่เพิ่ม dependency
เข้า**โปรเจกต์**ของคุณ `cargo install` คือการติดตั้ง**เครื่องมือ**ที่ใช้จาก terminal โดยตรง ไม่เกี่ยวกับ
`Cargo.toml` ของโปรเจกต์ไหนเลย

Flag ที่ใช้บ่อย:

```bash
cargo install ripgrep --version 14.1.0   # ระบุเวอร์ชันเจาะจง
cargo install --path .                     # compile และติดตั้ง package ในโฟลเดอร์ปัจจุบัน (มีประโยชน์ตอน dev CLI ของตัวเอง)
cargo install cargo-audit                  # ติดตั้ง cargo subcommand เพิ่ม (ดูหัวข้อถัดไป)
cargo install --list                       # ดูรายการที่ติดตั้งไว้แล้วทั้งหมด
```

**จุดที่น่าสนใจ**: `cargo install cargo-audit` คือวิธีที่ Cargo ใช้เพิ่ม **custom subcommand** — ถ้า binary
ที่ติดตั้งชื่อขึ้นต้นด้วย `cargo-` (เช่น `cargo-audit`, `cargo-edit` สมัยก่อนที่ยังไม่รวมเข้า core, `cargo-watch`)
Cargo จะรู้จักมันโดยอัตโนมัติว่าเรียกผ่าน `cargo audit`/`cargo watch` ได้เลย (ตัด prefix `cargo-` ออกกลายเป็นชื่อ
subcommand) — นี่คือกลไกเดียวกับที่ทำให้ระบบนิเวศ subcommand ของ Cargo ขยายได้ไม่จำกัดโดยไม่ต้องแก้ตัว Cargo
เอง

#### `cargo tree`: มองภาพ dependency graph ทั้งหมด

Part 17 หัวข้อ 17.12/กับดักข้อ 5 ใช้ `cargo tree` ไปบ้างแล้ว — สรุปการใช้งานที่สำคัญเพิ่มเติมสำหรับ debug
ปัญหาเวอร์ชันซ้ำซ้อน:

```bash
cargo tree -d
```

Flag `-d` (ย่อของ `--duplicates`) แสดง**เฉพาะ package ที่มีมากกว่าหนึ่งเวอร์ชันอยู่ในกราฟเดียวกัน** — มีประโยชน์
มากตอนสงสัยว่าทำไม binary ใหญ่กว่าที่คาดไว้ (สองเวอร์ชันของ crate เดียวกันถูก compile เข้ามาทั้งคู่ เพราะ
version requirement ไม่ overlap กันพอที่จะ resolve เป็นเวอร์ชันเดียว) เช่นถ้าเจอผลลัพธ์แบบนี้:

```
regex v1.10.4
regex v0.4.6
```

แปลว่ามี dependency บางตัวในกราฟ pin `regex` เวอร์ชัน 0.x เก่ามาก ๆ ที่ไม่ compatible กับ 1.x เลย ทำให้ Cargo
ต้อง compile ทั้งสองเวอร์ชันแยกกันจริง ๆ (กิน binary size และ compile time เพิ่มขึ้นโดยไม่จำเป็น) — วิธีแก้คือ
หา dependency ตัวที่ pin เวอร์ชันเก่า แล้วดูว่ามีเวอร์ชันใหม่กว่าที่อัปเดต `regex` ให้ตรงกันแล้วหรือยัง

```bash
cargo tree -i serde   # -i (--invert): ดูว่า "ใครบ้าง" ที่ depend on serde (มองย้อนทาง)
```

มีประโยชน์เมื่อรู้ว่า `serde` มีปัญหา (เช่น security advisory) แล้วอยากรู้ว่า dependency ตัวไหนในโปรเจกต์ที่ดึง
`serde` เข้ามา (บางทีคุณอาจไม่ได้เขียน `serde` ใน `Cargo.toml` ของคุณเองเลย แต่ dependency ของ dependency
ดึงมันเข้ามาโดยที่คุณไม่รู้ตัว)

#### `cargo audit`: ตรวจสอบช่องโหว่ความปลอดภัยของ dependency

```bash
cargo install cargo-audit
cargo audit
```

`cargo audit` **ไม่ได้ติดตั้งมากับ Cargo core** ต้อง `cargo install cargo-audit` ก่อนเสมอ (ตามกลไก custom
subcommand ที่อธิบายไว้ข้างบน) เมื่อรันแล้ว มันจะเทียบทุก dependency ใน `Cargo.lock` กับฐานข้อมูลช่องโหว่ความ
ปลอดภัยที่รู้จักแล้ว (RustSec Advisory Database) แล้วรายงานถ้าเจอ dependency ตัวไหนมี CVE (Common
Vulnerabilities and Exposures) ที่ประกาศไว้แล้ว พร้อมบอกเวอร์ชันที่แก้ปัญหานั้นแล้วให้อัปเดตไป

นี่คือเครื่องมือสำคัญมากสำหรับโปรเจกต์ production จริง — dependency ของคุณอาจมี security patch ใหม่ออกมาโดยที่
คุณไม่รู้เลยถ้าไม่มีเครื่องมือแบบนี้คอยเช็ค หลายทีมตั้งให้ `cargo audit` รันอัตโนมัติใน CI pipeline ทุกครั้งที่
มี pull request เพื่อจับปัญหาก่อนที่ dependency ที่มีช่องโหว่จะถูก merge เข้า production — เราจะพูดถึง security
practice แบบเจาะจงและ workflow การจัดการช่องโหว่แบบเต็มรูปแบบใน **Part 100 (Security Best Practices ใน Rust)**

### 35.21 ตัวอย่างจริงแบบเต็มรูปแบบ: ประกอบทุกอย่างในบทนี้เข้าด้วยกัน

มาประกอบทุกแนวคิดของบทนี้เข้าเป็น workspace 3 crate ที่ตั้งค่าแบบ production จริง — ระบบจัดการสินค้าคงคลัง
(`store_core`/`store_api`/`store_cli`) ที่มี: `[workspace.dependencies]`, `[workspace.package]`, custom
profile, feature ที่มีผลต่อ unification, `.cargo/config.toml` พร้อม alias

```
store_platform/
├── Cargo.toml
├── .cargo/
│   └── config.toml
├── core/
│   ├── Cargo.toml
│   └── src/lib.rs
├── api/
│   ├── Cargo.toml
│   └── src/lib.rs
└── cli/
    ├── Cargo.toml
    └── src/main.rs
```

**Root manifest — รวม workspace, profile, dependency, package metadata ทั้งหมด**:

```toml
# store_platform/Cargo.toml
[workspace]
resolver = "2"
members = ["core", "api", "cli"]

[workspace.package]
version = "0.3.0"
edition = "2021"
authors = ["Store Platform Team <platform@example.com>"]
license = "MIT OR Apache-2.0"
repository = "https://github.com/example/store-platform"

[workspace.dependencies]
serde = { version = "1", default-features = false }
serde_json = "1"

[profile.release]
opt-level = 3
lto = "thin"
codegen-units = 1
panic = "abort"
strip = true
debug = false

[profile.profiling]
inherits = "release"
debug = true
strip = false
```

**`.cargo/config.toml`**:

```toml
# store_platform/.cargo/config.toml
[alias]
ci = "check --workspace --all-targets"
covtest = "test --workspace --all-features"

[env]
STORE_PLATFORM_ENV = "development"
```

**`core/Cargo.toml`** — library หลัก, `serde` เป็น optional dependency เปิดผ่าน feature `json`:

```toml
[package]
name = "store_core"
version.workspace = true
edition.workspace = true
authors.workspace = true
license.workspace = true

[dependencies]
serde = { workspace = true, features = ["derive"], optional = true }
serde_json = { workspace = true, optional = true }

[features]
default = []
json = ["serde", "serde_json"]
```

**`core/src/lib.rs`**:

```rust
//! store_core: โดเมนหลักของระบบคลังสินค้า ไม่มี I/O ใด ๆ

#[cfg_attr(feature = "serde", derive(serde::Serialize, serde::Deserialize))]
#[derive(Debug, Clone, PartialEq)]
pub struct Product {
    pub sku: String,
    pub price_cents: u64,
    pub stock: u32,
}

impl Product {
    pub fn new(sku: impl Into<String>, price_cents: u64, stock: u32) -> Self {
        Product { sku: sku.into(), price_cents, stock }
    }

    pub fn total_value_cents(&self) -> u64 {
        self.price_cents * self.stock as u64
    }
}

// feature-gated module ทั้งโมดูล (หัวข้อ 35.12) — มีอยู่จริงก็ต่อเมื่อเปิด feature "json" เท่านั้น
#[cfg(feature = "json")]
pub mod json_export {
    use super::Product;

    pub fn to_json(p: &Product) -> String {
        serde_json::to_string(p).expect("Product ควร serialize ได้เสมอ")
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn total_value_is_price_times_stock() {
        let p = Product::new("SKU-1", 1000, 5);
        assert_eq!(p.total_value_cents(), 5000);
    }
}
```

**`api/Cargo.toml`** — ใช้ `serde` แบบเปิด feature กว้างกว่า `core` โดยตรง (ตัวอย่าง feature unification จริง):

```toml
[package]
name = "store_api"
version.workspace = true
edition.workspace = true
authors.workspace = true
license.workspace = true

[dependencies]
store_core = { path = "../core" }
serde = { workspace = true, features = ["derive", "std"] }
serde_json.workspace = true
```

**`api/src/lib.rs`**:

```rust
use store_core::Product;

#[derive(serde::Serialize)]
pub struct ProductResponse {
    pub sku: String,
    pub price_cents: u64,
}

pub fn to_response(p: &Product) -> ProductResponse {
    ProductResponse { sku: p.sku.clone(), price_cents: p.price_cents }
}

pub fn response_json(p: &Product) -> String {
    serde_json::to_string(&to_response(p)).unwrap()
}
```

**`cli/Cargo.toml`** — binary ปลายทาง, มี mutually exclusive feature สำหรับเลือก backend:

```toml
[package]
name = "store_cli"
version.workspace = true
edition.workspace = true
authors.workspace = true
license.workspace = true
publish = false

[dependencies]
store_core = { path = "../core" }
store_api = { path = "../api" }

[features]
default = ["backend_memory"]
backend_memory = []
backend_sqlite = []
```

**`cli/src/main.rs`**:

```rust
#[cfg(all(feature = "backend_memory", feature = "backend_sqlite"))]
compile_error!("เปิดได้แค่ backend เดียว: เลือก \"backend_memory\" หรือ \"backend_sqlite\" อย่างใดอย่างหนึ่งเท่านั้น");

use store_core::Product;

fn main() {
    let p = Product::new("SKU-100", 25000, 3);
    println!("มูลค่ารวม: {} สตางค์", p.total_value_cents());
    println!("api json: {}", store_api::response_json(&p));
}
```

**ทดสอบ end-to-end ทั้งหมด**:

```bash
cargo build --workspace
```

```
   Compiling serde v1.0.229
   Compiling serde_json v1.0.151
   Compiling store_core v0.3.0 (.../core)
   Compiling serde_derive v1.0.229
   Compiling store_api v0.3.0 (.../api)
   Compiling store_cli v0.3.0 (.../cli)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 6.36s
```

```bash
cargo run -p store_cli
```

```
มูลค่ารวม: 75000 สตางค์
api json: {"sku":"SKU-100","price_cents":25000}
```

```bash
cargo tree -e features -p store_cli --features store_core/json
```

แสดง `serde` ที่ compile จริงมีทั้ง feature `derive` (จาก `core` ผ่าน `json` feature) และ `std` (จาก `api`)
รวมกันตามหลัก feature unification ในหัวข้อ 35.10

```bash
cargo build -p store_cli --profile profiling --release
cargo build -p store_cli --release
```

ได้ binary สองตัวที่ opt-level เดียวกันทุกประการ แต่ `target/profiling/store_cli` เก็บ debug info ไว้ (ใหญ่กว่า
มาก, ใช้กับ profiler ได้) ในขณะที่ `target/release/store_cli` ถูก strip เต็มรูปแบบ (เล็กสุด, พร้อมแจกจ่ายจริง)
— ตัวอย่างนี้แสดงให้เห็นทุกฟีเจอร์หลักของบทนี้ทำงานประกอบกันจริงในโปรเจกต์เดียว: **`[workspace.dependencies]`
รวมศูนย์เวอร์ชัน `serde`/`serde_json`, `[workspace.package]` รวมศูนย์ metadata, custom profile `profiling`
สำหรับ debug production build, feature unification ที่ทำให้ `serde` ถูก compile ครั้งเดียวด้วย feature ที่รวม
จากทุกจุด, `compile_error!()` ป้องกัน feature ขัดแย้งกัน, และ `.cargo/config.toml` alias สำหรับ workflow ที่
ใช้ซ้ำบ่อย**

## กับดักที่พบบ่อย (Common Pitfalls)

**1. ตั้ง `panic = "abort"` แล้วคาดหวังว่า `catch_unwind` จะยังจับ panic ได้เหมือนเดิม**

```toml
[profile.release]
panic = "abort"
```

```rust
fn main() {
    let result = std::panic::catch_unwind(|| {
        panic!("boom");
    });
    println!("{:?}", result.is_err());
}
```

Compile ผ่านปกติโดยไม่มี warning เตือนเลย แต่พอรัน release build จะได้:

```
thread 'main' (24964) panicked at src/main.rs:3:9:
boom
Aborted (exit code: 134)
```

โปรแกรม**ไม่ได้พิมพ์ `true` ออกมาแบบที่คาดไว้เลย** — มัน abort ทั้งโปรเซสไปตรง ๆ เพราะไม่มีการ unwind ให้จับ
**วิธีแก้**: ถ้าโค้ดของคุณพึ่ง `catch_unwind` เป็นกลไก resilience จริง ๆ (เช่น web server ที่ isolate 1 request
ที่ panic ไม่ให้ทั้ง server ล่ม) ต้องคง `panic = "unwind"` ไว้เสมอ ห้ามตั้ง `"abort"` แม้จะอยากได้ binary เล็ก
ก็ตาม — ทั้งสองอย่างขัดแย้งกันโดยธรรมชาติ เลือกได้แค่อย่างเดียว

**2. เขียน `dependency.workspace = true` โดยไม่ได้ประกาศ dependency นั้นไว้ใน `[workspace.dependencies]` ที่ root**

```toml
# api/Cargo.toml
[dependencies]
anyhow = { workspace = true }
```

แต่ root `Cargo.toml` ไม่มี `anyhow` อยู่ใน `[workspace.dependencies]` เลย จะได้ error ทันทีตอน `cargo check`:

```
error: failed to parse manifest at `/path/to/api/Cargo.toml`

Caused by:
  error inheriting `anyhow` from workspace root manifest's `workspace.dependencies.anyhow`

Caused by:
  `dependency.anyhow` was not found in `workspace.dependencies`
```

**วิธีแก้**: เพิ่ม `anyhow = "..."` เข้า `[workspace.dependencies]` ที่ root ก่อนเสมอ ก่อนที่ member จะ
`.workspace = true` มันได้ — ข้อความ error บอกชื่อ dependency ที่หาไม่เจอไว้ตรง ๆ แล้ว ตรวจสอบการพิมพ์ชื่อให้
ตรงกันทั้งสองที่เสมอ (case-sensitive)

**3. เข้าใจผิดว่าปิด feature ที่ member ของตัวเองแล้ว dependency จะ "เล็กลง" แน่นอน โดยไม่เช็คว่า member อื่นในกราฟเดียวกันเปิดกว้างกว่าอยู่หรือไม่**

สมมติปิด feature `json` ของ `store_core` (ไม่เปิดใน `cargo build -p store_cli` เลย) แล้วคาดหวังว่า `serde`
จะไม่ถูก compile เข้ามาในผลลัพธ์สุดท้ายเลย — แต่ถ้า `store_api` (ที่ `store_cli` depend on อยู่ด้วย) ดึง `serde`
เข้ามาแบบไม่มีเงื่อนไข (ไม่ใช่ optional) `serde` ก็จะยังถูก compile เข้ามาอยู่ดี เพราะ feature unification
(หัวข้อ 35.10) รวมทุกจุดที่เรียก `serde` ในกราฟเดียวกันเข้าด้วยกัน ไม่ใช่แค่ดูที่ `store_core` ตัวเดียว
**วิธีแก้**: ก่อนจะเชื่อว่า "ปิด feature แล้ว dependency หาย" ให้ตรวจสอบด้วย `cargo tree -e features -p
<binary>` เสมอ เพื่อดูว่า dependency ตัวนั้นถูกเรียกจากจุดอื่นในกราฟเดียวกันด้วยหรือไม่

**4. ตั้ง `codegen-units = 1` และ `lto = "fat"` ใน `[profile.dev]` เพราะคิดว่า "ยิ่ง optimize มากยิ่งดี"**

```toml
[profile.dev]
opt-level = 0
lto = "fat"        # ผิด! dev profile ไม่ควรมี LTO เลย
codegen-units = 1   # ผิด! ตัด parallelism ของ dev build ที่ตั้งใจให้เร็วที่สุดออกไปหมด
```

ผลลัพธ์คือ dev build ที่**ช้าลงมหาศาล** (compile ทีละ core เดียว, ทำ whole-program LTO ทุกครั้งที่แก้โค้ดแม้แค่
บรรทัดเดียว) โดยไม่ได้ประโยชน์ด้าน runtime speed เลยเพราะ `opt-level = 0` ยังปิดการ optimize อยู่ดี (LTO/
codegen-units ไม่มีความหมายอะไรถ้าไม่มีการ optimize ให้ทำงานร่วมด้วยตั้งแต่ต้น) **วิธีแก้**: คีย์เหล่านี้ (`lto`,
`codegen-units = 1`) เหมาะกับ **release profile เท่านั้น** ที่ต้องการความเร็วตอนรันสูงสุดและยอมเสียเวลา compile
— dev profile ควรเน้น compile เร็วที่สุดเสมอ ปล่อยค่า default (`opt-level = 0`, `codegen-units = 256`,
`lto = false`) ไว้ตามเดิม

**5. ลืมว่า `cargo update --precise` อาจทำให้ transitive dependency เปลี่ยนไปด้วย ไม่ใช่แค่ crate ที่ระบุตรง ๆ**

```bash
cargo update -p serde_json --precise 1.0.140
```

```
      Adding ryu v1.0.23
 Downgrading serde_json v1.0.151 -> v1.0.140
    Removing zmij v1.0.23
```

การถอย `serde_json` เวอร์ชันเดียวทำให้ transitive dependency ของมัน (crate ที่ใช้ทำงานภายใน) เปลี่ยนตามไปด้วย
ทั้งเพิ่มและลบ **วิธีแก้**: รัน `--dry-run` ก่อนเสมอเพื่อดูผลกระทบทั้งหมด และรัน `cargo test --workspace` ให้
ครบหลังอัปเดตจริงก่อน commit `Cargo.lock` ที่เปลี่ยนแปลง — ห้าม `cargo update --precise` แล้ว commit ทันทีโดย
ไม่ตรวจสอบผลกระทบก่อน

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** สร้าง package ใหม่ (`cargo new profile_lab`) แล้วเขียนโปรแกรมที่มี loop คำนวณหนักพอสมควร (เช่น
   หาผลรวมของจำนวนเฉพาะทั้งหมดที่น้อยกว่า 1,000,000 ด้วยวิธี brute-force ตรง ๆ ไม่ต้องใช้ algorithm ที่ซับซ้อน)
   วัดเวลารันด้วยคำสั่ง `time` เทียบระหว่าง `cargo run` (debug) กับ `cargo run --release` แล้วบันทึกตัวเลขที่ได้
   ลองเพิ่ม `[profile.release]` ที่ตั้ง `opt-level = "s"` แทน `3` แล้ววัดเวลาใหม่ พร้อมเทียบขนาดไฟล์ binary ทั้ง
   สามแบบด้วย `ls -la target/*/profile_lab` (hint: ผลลัพธ์ควรแสดงให้เห็นว่า `opt-level = "s"` รันช้ากว่า `3`
   เล็กน้อยแต่ binary เล็กกว่า)

2. **[กลาง]** สร้าง custom profile ชื่อ `ci` ที่ `inherits = "dev"` แต่ตั้ง `debug = false` และ
   `incremental = false` เพิ่มเข้ามา ทดสอบ build ด้วย `cargo build --profile ci` แล้วเทียบขนาดโฟลเดอร์
   `target/ci/` กับ `target/debug/` (ใช้ `du -sh target/ci target/debug`) เขียนสรุปสั้น ๆ (3-5 บรรทัด) ว่า
   ทำไม CI pipeline มักไม่ต้องการ debug info เต็มรูปแบบแบบที่ dev build ปกติมี (hint: CI แค่รัน `cargo test`
   แล้วจบ ไม่มีใคร attach debugger เข้าไปดู)

3. **[กลาง-ยาก]** สร้าง workspace 3 member ชื่อ `notify_platform` ที่มี `notify_core` (library, มี struct
   `Notification { title: String, body: String }` กับ optional dependency `serde` เปิดผ่าน feature `json`),
   `notify_email` (library, depend on `notify_core`, ใช้ `serde` แบบไม่มีเงื่อนไขพร้อม feature `["derive"]`),
   และ `notify_cli` (binary, depend on ทั้งสอง) ใช้ `[workspace.dependencies]` รวมศูนย์เวอร์ชัน `serde` ไว้ที่
   root ทดสอบด้วย `cargo tree -e features -p notify_cli --features notify_core/json` ว่า `serde` ที่ compile
   จริงมี feature อะไรรวมกันบ้าง แล้วอธิบายผลลัพธ์ตามหลัก feature unification ในหัวข้อ 35.10 (hint: ต้องเห็น
   feature จากทั้ง `notify_core` และ `notify_email` รวมกันในบรรทัดเดียวของ `serde`)

4. **[ยาก/ประยุกต์]** ต่อยอดจากข้อ 3 เพิ่ม feature `transport_smtp` และ `transport_local` ให้ `notify_cli`
   (mutually exclusive, `default = ["transport_local"]`) พร้อม `compile_error!()` ป้องกันการเปิดสองตัวพร้อมกัน
   จากนั้นสร้าง custom profile ชื่อ `release_slim` ที่ `inherits = "release"` แต่เพิ่ม `panic = "abort"` และ
   `strip = true` ทดสอบว่า `cargo check --features transport_smtp,transport_local` ล้มเหลวด้วยข้อความที่คุณ
   กำหนดเอง และ `cargo build --profile release_slim` ให้ binary ที่เล็กกว่า `cargo build --release` ธรรมดา
   (สมมติ `[profile.release]` เดิมไม่มี `strip`/`panic` เหล่านี้) เขียนสรุปสั้น ๆ อธิบายว่าทำไมการแยก
   `release_slim` เป็น profile ต่างหาก (แทนที่จะแก้ `[profile.release]` ตรง ๆ) ถึงมีประโยชน์ในบางสถานการณ์
   (hint: บางทีมยังต้องการ release build ปกติที่ debug ได้ง่ายกว่า ไว้ใช้ตอนสืบปัญหา ควบคู่กับ build ที่บีบ
   ขนาดสุด ๆ สำหรับแจกจ่ายจริง)

## สรุป

บทนี้ปิดท้าย**โมดูล 2: ระดับกลาง (Intermediate)** ด้วยการพา Cargo กลับมาอีกครั้งในความลึกที่ Part 2 และ Part 17
ยังไม่ได้พูดถึง เราไล่ทำความเข้าใจทุกคีย์สำคัญของ `[profile.dev]`/`[profile.release]` — `opt-level` (0-3, s, z),
`debug`, `debug-assertions`, `overflow-checks` (ที่โยงกลับไปหาพฤติกรรม integer overflow จาก Part 3), `lto`
(false/thin/fat กับ trade-off compile time เทียบ runtime speed), `codegen-units` (parallel compile เทียบ
optimization opportunity), `panic` (`unwind`/`abort` ที่โยงกลับไปหา Part 12 และผลกระทบต่อ `catch_unwind`), และ
`strip` — พร้อมประกอบเป็น `[profile.release]` ที่ tune สำหรับ production CLI จริง และสร้าง **custom profile**
ด้วย `inherits` สำหรับ use case เฉพาะทาง เช่น profiling

เราเจาะลึก **feature unification** ที่เป็นพฤติกรรมสำคัญของ Cargo ที่มือใหม่ (และมือเก่าบางคน) มักไม่รู้ตัว —
dependency ตัวเดียวกันที่ถูกเรียกจากหลายจุดในกราฟเดียวกันจะถูกรวม feature เข้าด้วยกันเสมอ พร้อมวิธีป้องกัน
feature ที่ขัดแย้งกันด้วย `compile_error!()` และการจัดกลุ่มโค้ดด้วย feature-gated module ทั้งโมดูล จากนั้นขยาย
ความเข้าใจ workspace จาก Part 17 ไปสู่ระดับ production จริงด้วย `[workspace.dependencies]` (รวมศูนย์เวอร์ชัน
dependency) และ `[workspace.package]` (รวมศูนย์ package metadata) ที่แก้ปัญหาการพิมพ์เวอร์ชันไม่ตรงกันระหว่าง
member ได้อย่างเป็นระบบ

ปิดท้ายด้วยการอ่านโครงสร้าง `Cargo.lock` อย่างละเอียด, ความต่างของ resolver v1/v2/v3, คำสั่ง `cargo update`
ทั้งสามระดับความละเอียด (`update`, `-p`, `--precise`), การตั้งค่า `.cargo/config.toml` (`[alias]`, `[env]`,
`[build]`, `[source]`), ความรู้พื้นฐานเกี่ยวกับ `build.rs`, และเครื่องมือเสริม `cargo install`/`cargo tree`/
`cargo audit` ที่ทุกทีม production จริงควรรู้จัก

ตอนนี้คุณมีความเข้าใจ Cargo ในระดับที่ครอบคลุมเกือบทุกสถานการณ์ที่จะเจอในงานจริงแล้ว — พร้อมสำหรับ **โมดูล 3:
ระดับสูง (Advanced)** ที่จะเริ่มต้นด้วย **Part 41: Unsafe Rust เบื้องต้น** ซึ่งจะพาคุณไปสัมผัสส่วนของภาษาที่
ปลด "การันตีความปลอดภัยของ borrow checker" ออกชั่วคราวเพื่อทำสิ่งที่ safe Rust ทำไม่ได้ (raw pointer, FFI,
low-level memory manipulation) — แต่ก่อนจะไปถึงจุดนั้น เรายังมีบทเรื่อง Macros (Part 36) และ Concurrency
พื้นฐาน (Part 37-40) รออยู่ก่อน ซึ่งเป็นเนื้อหาที่ยังอยู่ในระดับกลางแต่จำเป็นมากสำหรับเขียนโปรแกรม Rust ที่ทำงาน
แบบขนานได้อย่างปลอดภัย

---

**Part ก่อนหน้า:** [Documentation: rustdoc](part-034-documentation-rustdoc.md) | **Part ถัดไป:** [Macros: Declarative Macros](part-036-macros-declarative.md)
