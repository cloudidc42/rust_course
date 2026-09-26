# Part 2: Cargo และโครงสร้างโปรเจกต์

> โมดูล: เริ่มต้นใช้งาน Rust (Getting Started) | ระดับ: พื้นฐาน | เวลาโดยประมาณ: 90 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อ่านและเข้าใจทุก section ของไฟล์ `Cargo.toml` ได้ (`[package]`, `[dependencies]`, `[dev-dependencies]`, `[build-dependencies]`)
- อธิบายความแตกต่างระหว่าง `cargo new` กับ `cargo init`, และระหว่างโปรเจกต์แบบ binary กับแบบ library ได้
- ใช้คำสั่ง cargo หลัก ๆ ในชีวิตประจำวันได้อย่างคล่องแคล่ว: `cargo build`, `cargo run`, `cargo check`, `cargo doc --open`, `cargo clean`
- เข้าใจว่าโฟลเดอร์ `target/` คืออะไร และความแตกต่างระหว่าง debug build กับ release build ว่าทำไม release ถึง
  compile ช้ากว่าแต่รันเร็วกว่า
- เข้าใจว่า `Cargo.lock` คืออะไร ทำไมต้องมี และควร commit เข้า git หรือไม่ในสถานการณ์ต่าง ๆ
- เพิ่ม dependency จาก crates.io เข้าโปรเจกต์ได้ทั้งแบบแก้ไฟล์ตรง ๆ และแบบใช้คำสั่ง `cargo add` พร้อมเข้าใจ
  ไวยากรณ์ semantic versioning ที่ใช้ระบุเวอร์ชัน
- รู้จักภาพรวมของ workspace และ build profile ว่ามีไว้ทำอะไร (รายละเอียดเต็มจะอยู่ใน Part 17 และ Part 35)

## ความรู้ที่ต้องมีมาก่อน

- **Part 1**: ต้องติดตั้ง Rust toolchain (`rustup`, `cargo`, `rustc`) เรียบร้อยแล้ว และเคยรัน `cargo new hello_cargo`
  กับ `cargo run` มาแล้วอย่างน้อยหนึ่งครั้ง บทนี้จะใช้โปรเจกต์ `hello_cargo` ที่สร้างไว้จาก Part 1 เป็นตัวอย่างต่อเนื่อง
  ถ้ายังไม่มีโปรเจกต์นี้อยู่ ให้กลับไปรัน `cargo new hello_cargo` ก่อนเริ่มบทนี้

## เนื้อหา

### 2.1 ทบทวน: cargo new สร้างอะไรให้เราบ้าง

ใน Part 1 เราสร้างโปรเจกต์แรกด้วยคำสั่ง:

```bash
cargo new hello_cargo
```

และได้โครงสร้างไฟล์แบบนี้:

```
hello_cargo/
├── Cargo.toml
├── .gitignore
└── src/
    └── main.rs
```

ก่อนหน้านี้เราแค่ "ใช้" มันเพื่อรัน Hello World แต่ยังไม่ได้อธิบายว่าแต่ละไฟล์คืออะไร ทำไมต้องมีไฟล์เหล่านี้
บทนี้เราจะแยกส่วนทุกไฟล์ ทุกคำสั่งของ cargo ออกมาดูอย่างละเอียด เพราะ **cargo คือเครื่องมือที่คุณจะเปิดใช้งาน
เกือบทุกครั้งที่เขียน Rust ตลอดหลักสูตรนี้** — เข้าใจมันให้แน่นตั้งแต่ต้นจะช่วยประหยัดเวลาในบทต่อ ๆ ไปมาก

สังเกตว่านอกจาก `Cargo.toml` และ `src/main.rs` ที่เราเห็นใน Part 1 แล้ว cargo ยังสร้างไฟล์ `.gitignore` ให้ด้วย
โดยอัตโนมัติ และ (ถ้าโฟลเดอร์ปัจจุบันยังไม่ได้อยู่ใน git repository) cargo จะรัน `git init` ให้เองด้วย เนื้อหาของ
`.gitignore` ที่ cargo สร้างให้จะมีเพียงบรรทัดเดียว:

```
/target
```

เหตุผลที่ต้อง ignore โฟลเดอร์ `target/` เราจะอธิบายละเอียดในหัวข้อ 2.6 — สรุปสั้น ๆ ก่อนคือ มันเป็นโฟลเดอร์ที่เก็บไฟล์
ที่ compile ออกมา ซึ่งสร้างขึ้นใหม่ได้เสมอจาก source code จึงไม่มีประโยชน์ที่จะเก็บไว้ใน version control (แถมมีขนาด
ใหญ่มาก อาจหลายร้อย MB ถึงหลาย GB สำหรับโปรเจกต์ใหญ่)

### 2.2 กายวิภาคของ Cargo.toml

`Cargo.toml` คือ **manifest file** ของโปรเจกต์ Rust เขียนด้วยภาษา [TOML](https://toml.io/) (Tom's Obvious,
Minimal Language) ซึ่งเป็นรูปแบบไฟล์ config ที่อ่านง่าย คล้าย INI แต่มีโครงสร้างข้อมูลที่ชัดเจนกว่า (รองรับ array,
table, nested table) ลองเปิดไฟล์ `hello_cargo/Cargo.toml` ที่ cargo สร้างให้ดู:

```toml
[package]
name = "hello_cargo"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
```

มาแยกดูทีละ section:

#### `[package]`

นี่คือ table ที่เก็บ **metadata ของแพ็กเกจ (package)** เอง — ยังไม่เกี่ยวกับ dependency ใด ๆ ทั้งสิ้น

- **`name`** — ชื่อของแพ็กเกจ ต้องเป็น string ที่ประกอบด้วยตัวอักษร ตัวเลข, `-`, และ `_` เท่านั้น (ห้ามมีเว้นวรรค
  หรืออักขระพิเศษอื่น) ชื่อนี้จะถูกใช้เป็นชื่อ crate เวลามีคนอื่น depend on แพ็กเกจของคุณ และถ้าจะ publish ขึ้น
  crates.io ชื่อนี้ต้อง**ไม่ซ้ำ**กับแพ็กเกจอื่นที่มีอยู่แล้วในทั้งระบบ (คล้าย npm package name หรือ PyPI package name)
- **`version`** — เวอร์ชันของแพ็กเกจ ต้องเป็นไปตามรูปแบบ [Semantic Versioning (SemVer)](https://semver.org/)
  คือ `MAJOR.MINOR.PATCH` เช่น `0.1.0` ที่ cargo ตั้งให้เป็นค่าเริ่มต้นหมายถึง "เวอร์ชันพัฒนาเริ่มต้น ยังไม่ stable"
  (ตาม convention ของ SemVer, major version `0` แปลว่า API อาจเปลี่ยนแปลงได้ตลอดโดยไม่ถือว่า breaking change
  อย่างเป็นทางการ) เราจะพูดถึง SemVer แบบละเอียดอีกครั้งในหัวข้อ 2.8 เมื่อพูดถึงการระบุเวอร์ชันของ dependency
- **`edition`** — นี่คือคีย์ที่สำคัญมากและมักสร้างความสับสนให้มือใหม่ **"edition" ไม่ใช่เวอร์ชันของ compiler**
  แต่เป็นการกำหนด "ชุดกฎไวยากรณ์และ default behavior" ของภาษา Rust ที่ crate นี้จะใช้ Rust มี edition ออกมาแล้ว
  4 รุ่น: **2015** (edition แรกตอนเปิดตัว 1.0), **2018** (เพิ่ม module system ใหม่, `async`/`await` keyword
  reserved, NLL borrow checker), **2021** (เพิ่ม `IntoIterator` สำหรับ array, `TryInto`/`TryFrom` เข้า prelude,
  ปรับปรุง closure capture), และ **2024** (ปรับปรุง lifetime capture ใน `impl Trait`, เปลี่ยน default บางจุดของ
  `unsafe` block) — เวอร์ชัน `rustc`/`cargo` ของคุณที่ติดตั้งจาก Part 1 อาจ default ให้ edition ใหม่กว่า 2021
  เมื่อรัน `cargo new` ก็ได้ (ขึ้นกับเวอร์ชันของ cargo ที่คุณติดตั้ง) ไม่ต้องกังวล — ตัวอย่างโค้ดทั้งหมดในหลักสูตรนี้
  เขียนให้ทำงานถูกต้องทั้งบน edition 2021 และ 2024 เราจะยึด **2021** เป็น baseline อ้างอิงเพราะเป็น edition ที่ได้รับ
  การรองรับกว้างที่สุดในระบบนิเวศปัจจุบัน (library ส่วนใหญ่ยังตั้ง edition 2021 ไว้)

  จุดที่สำคัญที่สุดที่ต้องเข้าใจคือ **edition ไม่ทำให้ crate ที่เขียนด้วย edition เก่ากว่าใช้งานไม่ได้กับ crate ที่ใช้
  edition ใหม่กว่า** — คุณสามารถมี dependency graph ที่ผสมกันได้ (บาง crate ใช้ 2018, บาง crate ใช้ 2021) เพราะ
  `rustc` ตัวเดียวรองรับคอมไพล์ทุก edition พร้อมกันได้ในโปรเจกต์เดียว นี่คือการออกแบบที่ตั้งใจให้ **backward
  compatible ตลอดกาล** — ต่างจากภาษาอื่นบางภาษาที่การอัปเดตเวอร์ชันภาษาอาจทำให้โค้ดเก่า compile ไม่ผ่านอีกต่อไป
  Rust แก้ปัญหานี้ด้วยการแยก "เวอร์ชัน compiler" ออกจาก "เวอร์ชันไวยากรณ์ภาษา" อย่างชัดเจน การอัปเดต `rustc`
  เป็นเวอร์ชันใหม่จะได้ bug fix, performance improvement, และ standard library ใหม่ ๆ เสมอ แต่ไวยากรณ์ของโค้ด
  คุณจะไม่เปลี่ยนจนกว่าคุณจะแก้ `edition` ใน `Cargo.toml` ด้วยตัวเองอย่างตั้งใจ (ปกติทำผ่านคำสั่ง `cargo fix --edition`)

  ถ้าไม่มีคีย์ `edition` เลยใน `Cargo.toml` เก่ามาก ๆ (โปรเจกต์ที่สร้างสมัย Rust 1.0 แรก ๆ) cargo จะ fallback เป็น
  edition 2015 ให้อัตโนมัติ ซึ่งขาด feature หลายอย่างที่หลักสูตรนี้จะใช้ (เช่น `dyn Trait` แบบ explicit, `?` operator
  ในบาง context) ดังนั้น**ทุกโปรเจกต์ใหม่ควรมี `edition` ระบุไว้ชัดเจนเสมอ** — ซึ่ง `cargo new`/`cargo init` ทำให้
  อัตโนมัติอยู่แล้ว ไม่ต้องกังวลถ้าคุณไม่ได้ลบมันออกเอง

#### `[dependencies]`

Table นี้ระบุ **library ภายนอก (crate) ที่โค้ดของเราต้องใช้ในการ compile และรันโปรแกรมจริง** (production
dependency) ในไฟล์ที่ cargo สร้างให้ตอนแรก table นี้จะว่างเปล่า เพราะ Hello World ไม่ต้องพึ่ง library ภายนอก
เลย เราจะเติมเนื้อหาลงในนี้แบบละเอียดในหัวข้อ 2.8-2.9

#### `[dev-dependencies]`

Table นี้ระบุ dependency ที่ใช้**เฉพาะตอนพัฒนา** เช่น ตอนรัน `cargo test` หรือ `cargo bench` — **จะไม่ถูก
compile เข้าไปใน binary จริงที่ปล่อยให้ผู้ใช้** ตัวอย่าง crate ที่มักอยู่ใน `[dev-dependencies]` เช่น
`criterion` (สำหรับ benchmark, Part 54), `mockall` (สำหรับสร้าง mock object ในการทดสอบ), หรือ crate ที่ช่วย
เขียน assertion แบบละเอียดขึ้นอย่าง `pretty_assertions`

```toml
[dev-dependencies]
criterion = "0.5"
```

เหตุผลที่ต้องแยก table นี้ออกจาก `[dependencies]` คือเรื่อง **ขนาดและความปลอดภัยของ binary ที่ปล่อยจริง**
ถ้าคุณใส่ testing library ปนเข้าไปใน `[dependencies]` มันจะถูก compile ติดไปกับโปรแกรมที่ผู้ใช้ปลายทางรัน
ทำให้ binary ใหญ่ขึ้นโดยไม่จำเป็น และเพิ่ม attack surface (โค้ดที่ไม่ได้ตั้งใจให้รันใน production) แบบไม่มีประโยชน์
เราจะเรียนรู้เรื่อง testing และการใช้ `[dev-dependencies]` แบบเจาะลึกใน **Part 32-33**

#### `[build-dependencies]`

Table นี้ (พบไม่บ่อยเท่าสองตัวบน) ใช้ระบุ dependency ที่ต้องใช้ตอนรัน **build script** — ไฟล์พิเศษชื่อ
`build.rs` ที่วางไว้ที่ root ของแพ็กเกจ ซึ่งจะถูก compile และรัน**ก่อน**โค้ดหลักของแพ็กเกจจะถูก compile
ใช้สำหรับงานเช่น generate โค้ดจาก schema (เช่น compile ไฟล์ `.proto` ของ gRPC ด้วย crate `tonic-build`
ใน Part 80), หรือ link กับ library ภาษา C ผ่าน FFI (Part 43)

```toml
[build-dependencies]
tonic-build = "0.12"
```

crate ใน `[build-dependencies]` จะถูก compile แยกจาก crate ใน `[dependencies]` โดยสิ้นเชิง (คนละ target,
อาจคนละ platform ด้วยถ้าเป็นการ cross-compile) เพราะ build script รันบนเครื่องที่ใช้ compile (host machine)
ไม่ใช่รันบนเครื่องปลายทางที่โปรแกรมสุดท้ายจะถูก deploy ไป — รายละเอียดเชิงลึกของ build script จะไม่อยู่ในสโคปของ
หลักสูตรพื้นฐานนี้ แต่จะกลับมาแตะอีกครั้งเมื่อพูดถึง FFI ใน Part 43

### 2.3 cargo new กับ cargo init: ต่างกันตรงไหน

ทั้งสองคำสั่งทำสิ่งเดียวกันเป๊ะ ๆ คือสร้าง `Cargo.toml`, `.gitignore`, และไฟล์ source เริ่มต้น — **ความต่างมีจุดเดียว
คือเรื่องโฟลเดอร์**

`cargo new` **สร้างโฟลเดอร์ใหม่ให้เอง** ตามชื่อที่คุณระบุ:

```bash
cargo new hello_cargo
# สร้างโฟลเดอร์ ./hello_cargo/ ขึ้นมาใหม่ พร้อมไฟล์ข้างในครบ
```

`cargo init` ใช้ **ในโฟลเดอร์ที่มีอยู่แล้ว** (ไม่สร้างโฟลเดอร์ใหม่) — เหมาะกับสถานการณ์ที่คุณมีโฟลเดอร์โปรเจกต์อยู่
แล้ว เช่น `git clone` มา หรือสร้างโฟลเดอร์เปล่าไว้ล่วงหน้าด้วยเหตุผลอื่น:

```bash
mkdir my_existing_folder
cd my_existing_folder
cargo init
# สร้าง Cargo.toml, .gitignore, src/main.rs ในโฟลเดอร์ปัจจุบันนี้เลย ไม่สร้างโฟลเดอร์ย่อยเพิ่ม
```

`cargo init` จะตั้งชื่อแพ็กเกจใน `Cargo.toml` (คีย์ `name`) ตามชื่อโฟลเดอร์ปัจจุบันโดยอัตโนมัติ ถ้าต้องการชื่ออื่น
แก้ในไฟล์ `Cargo.toml` เองได้ทันทีหลังสร้าง (การเปลี่ยนชื่อแพ็กเกจไม่กระทบการทำงานของโปรแกรมเลย เป็นแค่ metadata)

#### `--lib` vs default (binary)

โดย default ทั้ง `cargo new` และ `cargo init` จะสร้างโปรเจกต์แบบ **binary crate** (โปรแกรมที่รันได้ มีจุดเริ่มต้น
คือฟังก์ชัน `main`) ไฟล์ source หลักคือ `src/main.rs` แต่ถ้าคุณกำลังเขียน **library** (โค้ดที่ตั้งใจให้โปรเจกต์อื่นมา
`use` หรือ depend on ไม่ใช่โปรแกรมที่รันเองได้) ให้เพิ่ม flag `--lib`:

```bash
cargo new my_math_lib --lib
```

ได้โครงสร้าง:

```
my_math_lib/
├── Cargo.toml
├── .gitignore
└── src/
    └── lib.rs
```

และเนื้อหาเริ่มต้นของ `src/lib.rs` (cargo เวอร์ชันปัจจุบันจะสร้างตัวอย่างพร้อม test มาให้เลย):

```rust
pub fn add(left: u64, right: u64) -> u64 {
    left + right
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_works() {
        let result = add(2, 2);
        assert_eq!(result, 4);
    }
}
```

สังเกตความแตกต่างสำคัญจาก `main.rs`:

- **ไม่มีฟังก์ชัน `main`** — library crate ไม่มีจุดเริ่มต้นให้รันเอง มันเป็นแค่ "คลังฟังก์ชัน/type" ที่รอให้โปรแกรมอื่น
  มาเรียกใช้
- ฟังก์ชัน `add` มี keyword **`pub`** นำหน้า — หมายถึงฟังก์ชันนี้ **public** สามารถถูกเรียกใช้จากนอก crate ได้
  (ถ้าไม่มี `pub` ฟังก์ชันจะเป็น private ใช้ได้แค่ภายใน crate เดียวกันเท่านั้น) เราจะเรียนเรื่อง `pub`/module
  visibility แบบละเอียดใน **Part 16**
- มี `mod tests { ... }` พร้อม `#[test]` attribute ติดมาให้เป็นตัวอย่าง — นี่คือ **unit test** ซึ่งเราจะเจาะลึกใน
  **Part 32** ตอนนี้แค่รู้ไว้ว่าสั่งรันได้ด้วย `cargo test`

**สรุปกฎง่าย ๆ**: `src/main.rs` = binary crate root (ต้องมี `fn main`, ให้ผลเป็นไฟล์ executable ที่รันตรง ๆ ได้)
ส่วน `src/lib.rs` = library crate root (ห้ามมี `fn main`, ให้ผลเป็น library ที่ crate อื่นมา depend on ได้)
**แพ็กเกจเดียวสามารถมีทั้งสองไฟล์พร้อมกันได้** — เป็น pattern ที่พบบ่อยมากในโปรเจกต์จริง คือเขียน logic หลักไว้
ใน `src/lib.rs` แล้วให้ `src/main.rs` สั้น ๆ ทำหน้าที่แค่เรียก logic จาก lib มาประกอบเป็นโปรแกรม CLI — ข้อดีคือ
ทำให้เขียน integration test เรียก logic ผ่าน library API ได้ง่ายกว่าการยัดทุกอย่างไว้ใน `main.rs` อย่างเดียว
เราจะเห็น pattern นี้ชัดเจนขึ้นใน Part 16-17 และตอนทำโปรเจกต์ CLI จริงใน Part 59

### 2.4 คำสั่ง cargo ที่ใช้บ่อยที่สุด

ต่อไปนี้คือคำสั่งที่คุณจะพิมพ์ซ้ำ ๆ นับพันครั้งตลอดหลักสูตรและอาชีพนักพัฒนา Rust ของคุณ ลองเปิด terminal เข้าไปใน
โฟลเดอร์ `hello_cargo` จาก Part 1 แล้วทดลองตามไปด้วย

#### `cargo build`

Compile โปรเจกต์ (และ dependency ทั้งหมด) ให้เป็น binary โดย**ไม่รัน**ทันที:

```bash
cd hello_cargo
cargo build
```

ผลลัพธ์:

```
   Compiling hello_cargo v0.1.0 (/home/user/hello_cargo)
    Finished dev [unoptimized + debuginfo] target(s) in 0.42s
```

Binary ที่ได้จะอยู่ที่ `target/debug/hello_cargo` (หรือ `target\debug\hello_cargo.exe` บน Windows) รันตรง ๆ ได้
ด้วย `./target/debug/hello_cargo`

#### `cargo run`

Compile แล้วรันต่อทันที ในคำสั่งเดียว (ที่เราใช้ไปแล้วใน Part 1):

```bash
cargo run
```

```
   Compiling hello_cargo v0.1.0 (/home/user/hello_cargo)
    Finished dev [unoptimized + debuginfo] target(s) in 0.38s
     Running `target/debug/hello_cargo`
Hello, world!
```

จุดที่ควรสังเกตคือ **ถ้าไม่มีอะไรเปลี่ยนแปลงในโค้ด** cargo จะฉลาดพอที่จะข้ามขั้นตอน compile และรัน binary เดิม
ที่ compile ไว้แล้วทันที (incremental build) ลองรัน `cargo run` ซ้ำอีกครั้งโดยไม่แก้ไขไฟล์อะไรเลย จะเห็นว่าไม่มี
บรรทัด `Compiling` ปรากฏขึ้นมาอีก — ประหยัดเวลาได้มากเมื่อโปรเจกต์ใหญ่ขึ้น

#### `cargo check`

นี่คือคำสั่งที่คุณจะใช้**บ่อยที่สุด**ระหว่างเขียนโค้ด (แม้จะใช้บ่อยกว่า `cargo run` เสียอีกในตอนที่กำลังไล่แก้ error):

```bash
cargo check
```

```
    Checking hello_cargo v0.1.0 (/home/user/hello_cargo)
    Finished dev [unoptimized + debuginfo] target(s) in 0.15s
```

**ทำไม `cargo check` เร็วกว่า `cargo build` มาก?** เพราะ `cargo check` ทำแค่ขั้นตอน **parsing + type checking +
borrow checking** เท่านั้น — มันตรวจสอบว่าโค้ดของคุณถูกไวยากรณ์ ถูก type ทุกจุด และผ่าน borrow checker หรือไม่
แต่**หยุดอยู่แค่นั้น ไม่เดินหน้าไปสร้าง machine code จริง** (ขั้นตอน code generation หรือ "codegen" ซึ่งเป็นขั้นตอน
ที่ใช้เวลานานที่สุดในกระบวนการ compile ทั้งหมด โดยเฉพาะกับ optimization) ส่วน `cargo build` ต้องทำ codegen เต็ม
รูปแบบเพื่อให้ได้ executable ไฟล์จริงที่รันได้ ดังนั้นในระหว่างที่คุณกำลังเขียนโค้ดและอยากรู้แค่ว่า **"โค้ดที่พิมพ์ไปยัง
compile ผ่านหรือยัง"** โดยยังไม่จำเป็นต้องรันจริง `cargo check` จะให้ feedback เร็วกว่าอย่างเห็นได้ชัด — ในโปรเจกต์
ขนาดกลางถึงใหญ่ ความต่างอาจเป็นวินาทีต่อวินาทีนับสิบเท่า

นี่คือเหตุผลที่ Part 1 แนะนำให้ตั้งค่า `rust-analyzer.check.command` เป็น `clippy` (ซึ่งภายในก็รัน check แบบขยาย
ผลเพิ่ม lint) เพราะ rust-analyzer จะเรียก `cargo check`/`clippy check` ให้อัตโนมัติแบบเงียบ ๆ ทุกครั้งที่คุณ save
ไฟล์ ทำให้เห็น error/warning สีแดง/เหลืองใน editor แบบ real-time โดยไม่ต้องเสียเวลา codegen เต็มรูปแบบทุกครั้ง

**workflow ที่แนะนำ**: ระหว่างเขียนโค้ดฟีเจอร์ใหม่ ให้รัน `cargo check` ซ้ำ ๆ บ่อย ๆ (หรือให้ editor ทำให้อัตโนมัติ)
แล้วรัน `cargo run` เฉพาะตอนที่อยากเห็นผลลัพธ์จริงของโปรแกรม

#### `cargo test`

รัน automated test ทั้งหมดในโปรเจกต์ (ทั้ง unit test ใน `src/` และ integration test ใน `tests/`):

```bash
cargo test
```

บทนี้จะไม่ลงรายละเอียดเรื่อง test เพราะมีบทเฉพาะทางรออยู่ — **Part 32: Testing: Unit Tests** และ
**Part 33: Testing: Integration Tests** จะอธิบายการเขียน `#[test]`, `assert!`, `assert_eq!`, และการจัดระเบียบ
test suite แบบละเอียดครบถ้วน ตอนนี้แค่รู้ไว้ว่าคำสั่งนี้มีอยู่และเป็นส่วนหนึ่งของ workflow ปกติของ cargo

#### `cargo doc --open`

สร้างเอกสาร (documentation) แบบ HTML จาก doc comment ในโค้ดของคุณ **รวมถึง doc ของทุก dependency ที่ใช้ด้วย**
แล้วเปิดในเบราว์เซอร์ทันที:

```bash
cargo doc --open
```

คำสั่งนี้มีประโยชน์มากกว่าที่คิด — ไม่ใช่แค่ตอนที่คุณเขียน doc comment (`///`) ให้โค้ดตัวเอง (จะเรียนใน Part 34)
แต่ยังใช้เป็น**เครื่องมือค้นหา API ของ dependency ที่ติดตั้งในโปรเจกต์แบบ offline** ได้ทันที โดยไม่ต้องเปิด
docs.rs ผ่านอินเทอร์เน็ต — สะดวกมากเวลาทำงานแบบ offline หรืออยากดู API เวอร์ชันที่ตรงกับที่ล็อกไว้ใน
`Cargo.lock` เป๊ะ ๆ (ไม่ใช่เวอร์ชันล่าสุดบน docs.rs ซึ่งอาจต่างจากที่คุณใช้จริง)

#### `cargo clean`

ลบโฟลเดอร์ `target/` ทั้งหมดทิ้ง:

```bash
cargo clean
```

ใช้เมื่อ: ต้องการเคลียร์พื้นที่ดิสก์ (โฟลเดอร์ `target/` โตได้เร็วมาก), หรือสงสัยว่า build cache เสียหาย/ค้าง
(บางครั้ง incremental compilation cache อาจมีปัญหาหลังจากสลับ branch git บ่อย ๆ ทำให้ error แปลก ๆ ที่แก้ไม่ได้
ด้วยวิธีปกติ — `cargo clean` แล้ว build ใหม่คือวิธีแก้ไขแบบ "brute force" ที่ได้ผลเสมอ) ข้อเสียเพียงอย่างเดียวคือ
การ build ครั้งถัดไปหลัง clean จะช้าเท่ากับ build ครั้งแรกสุด เพราะต้อง compile ทุกอย่างใหม่หมด (รวม dependency
ทั้งหมดด้วย ไม่ใช่แค่โค้ดของคุณ)

### 2.5 โฟลเดอร์ target/ และความแตกต่างระหว่าง debug กับ release build

หลังรัน `cargo build` ไปสักพัก โฟลเดอร์ `target/` จะมีโครงสร้างประมาณนี้:

```
target/
├── debug/
│   ├── hello_cargo          <- binary ที่รันได้ (executable)
│   ├── hello_cargo.d
│   ├── deps/                <- compiled dependency + object file ระดับกลาง
│   ├── build/                <- output ของ build script (ถ้ามี)
│   ├── incremental/          <- cache สำหรับ incremental compilation
│   └── .fingerprint/         <- ใช้เช็คว่าไฟล์ไหนเปลี่ยนไปบ้าง จะได้ไม่ compile ซ้ำโดยไม่จำเป็น
└── CACHEDIR.TAG
```

ถ้ารัน `cargo build --release` จะได้โฟลเดอร์ `target/release/` เพิ่มขึ้นมาอีกชุดคู่กัน (debug กับ release ไม่ทับกัน
เพราะเก็บ artifact คนละแบบ):

```bash
cargo build --release
```

```
   Compiling hello_cargo v0.1.0 (/home/user/hello_cargo)
    Finished release [optimized] target(s) in 1.87s
```

สังเกตว่าข้อความเปลี่ยนจาก `Finished dev [unoptimized + debuginfo]` เป็น `Finished release [optimized]` —
นี่คือความต่างสำคัญที่สุดระหว่างสอง profile นี้:

| ด้าน | `cargo build` (dev/debug) | `cargo build --release` |
|---|---|---|
| Optimization level | `opt-level = 0` (ไม่ optimize) | `opt-level = 3` (optimize เต็มที่) |
| ความเร็วตอน compile | เร็ว | ช้ากว่ามาก (บางโปรเจกต์ใหญ่ช้ากว่าหลายสิบเท่า) |
| ความเร็วตอนรัน binary | ช้ากว่า (อาจช้ากว่า 10-100 เท่าในโค้ดที่ loop หนัก) | เร็วที่สุด |
| Debug symbol / debug info | มี (`debug = true`) ทำให้ debugger อ่าน stack trace ได้ละเอียด | ปกติไม่มี (`debug = false`) |
| Integer overflow check | เปิด (จะ panic ทันทีถ้า overflow) | ปิด (overflow จะ wrap around แบบเงียบ ๆ) |
| ขนาดไฟล์ binary | ใหญ่กว่า | เล็กกว่า (ปกติ) |

**ทำไม release build ถึง compile ช้ากว่า?** เพราะขั้นตอน optimization ของ `rustc` (ซึ่งใช้ LLVM เป็น backend
ในการสร้าง machine code) ต้องทำงานหนักขึ้นมาก เช่น inline ฟังก์ชันเข้าหากัน, ลบโค้ดที่ไม่มีผลลัพธ์ (dead code
elimination), จัดเรียง register ใหม่ให้มีประสิทธิภาพสูงสุด, vectorize loop ให้ใช้ SIMD instruction — งานเหล่านี้
ใช้เวลาวิเคราะห์มากกว่าการแปลงโค้ดแบบตรงไปตรงมาแบบใน debug build หลายเท่า แต่ผลลัพธ์ที่ได้คือ binary ที่รันเร็ว
กว่าอย่างมาก เพราะ CPU instruction ที่ได้มีประสิทธิภาพสูงกว่ามาก

**กฎการใช้งานจริง**: ใช้ `cargo build`/`cargo run` (debug) ระหว่างพัฒนาโปรแกรมเสมอ เพราะ compile เร็วกว่ามาก
ทำให้ loop การเขียน-ทดสอบเร็ว และมี debug info/overflow check ช่วยจับบั๊กได้ง่ายกว่า ส่วน `--release` ใช้ตอน
จะ deploy จริง, ตอนวัด performance จริง (benchmark), หรือตอนแจกจ่ายโปรแกรมให้ผู้ใช้ปลายทาง — **ห้ามวัด
performance ของโปรแกรม Rust จาก debug build เด็ดขาด** เพราะตัวเลขที่ได้จะช้ากว่าความเป็นจริงมาก จนทำให้เข้าใจผิด
ว่า Rust ช้า (มือใหม่หลายคนเข้าใจผิดจุดนี้บ่อยมาก)

ทั้งสอง profile นี้ปรับแต่งค่าย่อยได้ผ่าน section `[profile.dev]` และ `[profile.release]` ใน `Cargo.toml`
เช่นเปิด **Link-Time Optimization (LTO)** เพื่อบีบให้ optimizer มองเห็นทั้งโปรแกรม (รวม dependency) พร้อมกัน
แทนที่จะ optimize แยกทีละ crate ทำให้ optimize ได้ลึกขึ้นอีก (แต่ compile ช้าขึ้นไปอีกมาก) — เราจะพูดถึง
`[profile.*]` และ LTO แบบละเอียดจริงจังใน **Part 35** (Cargo ขั้นสูง) และ **Part 54** (Performance Optimization)
ตอนนี้แค่รู้ไว้ว่า section เหล่านี้มีอยู่และปรับได้ ตัวอย่าง (ยังไม่ต้องเข้าใจทุกคีย์ตอนนี้):

```toml
[profile.release]
opt-level = 3
lto = true
codegen-units = 1
```

### 2.6 Cargo.lock: ล็อกเวอร์ชันเพื่อ build ที่ทำซ้ำได้

ลองเปิดดูไฟล์ `Cargo.lock` ในโฟลเดอร์ `hello_cargo` — ถ้ายังไม่มี dependency เลย ไฟล์นี้จะสั้นมาก:

```toml
# This file is automatically @generated by Cargo.
# It is not intended for manual editing.
version = 4

[[package]]
name = "hello_cargo"
version = "0.1.0"
```

สังเกตคอมเมนต์ที่ cargo generate ให้เอง: **"It is not intended for manual editing"** — ห้ามแก้ไฟล์นี้ด้วยมือ
เด็ดขาด ให้ cargo จัดการมันเองเสมอ

**`Cargo.lock` คืออะไร และทำไมต้องมี?** ต้องเข้าใจก่อนว่า `Cargo.toml` ระบุ**ช่วง**ของเวอร์ชันที่ยอมรับได้ ไม่ใช่
เวอร์ชันที่แน่นอนตัวเดียว (รายละเอียดของ syntax นี้อยู่ในหัวข้อ 2.8) เช่นถ้า `Cargo.toml` เขียนว่า
`rand = "0.8"` มันแปลว่า "ยอมรับ rand เวอร์ชันไหนก็ได้ที่ `>=0.8.0, <0.9.0`" — ซึ่งอาจมีเวอร์ชันย่อยที่ตรงเงื่อนไข
นี้หลายสิบเวอร์ชัน (0.8.0, 0.8.1, ..., 0.8.5, ...) ถ้าปล่อยให้ cargo เลือกเวอร์ชันที่ match เองทุกครั้งที่ build
โดยไม่มีการล็อกไว้ ปัญหาที่จะเกิดคือ:

- เพื่อนร่วมทีมคนละคน clone repo เดียวกัน อาจได้ dependency เวอร์ชันย่อยต่างกัน (คนหนึ่งได้ 0.8.3 อีกคนได้ 0.8.5
  เพราะ crates.io ปล่อยเวอร์ชันใหม่ระหว่างนั้น) ทำให้เกิดบั๊กที่ **reproduce ไม่ได้** ("ทำงานบนเครื่องฉันนะ")
- CI/CD pipeline ที่ build วันนี้กับ build อีกหกเดือนข้างหน้า อาจได้ dependency คนละเวอร์ชันย่อย ถ้า dependency
  ตัวนั้นมี bug ใหม่หลุดออกมาในเวอร์ชันย่อยที่ใหม่กว่า โปรแกรมของคุณอาจพังโดยที่คุณไม่ได้แก้โค้ดตัวเองเลยแม้แต่บรรทัด
  เดียว

`Cargo.lock` แก้ปัญหานี้โดยการ**บันทึกเวอร์ชันที่ resolve ได้แบบเจาะจงเป๊ะ ๆ** (รวมถึง checksum ของแต่ละแพ็กเกจ
เพื่อยืนยันว่าไฟล์ที่ดาวน์โหลดมาไม่ถูกแก้ไข/สับเปลี่ยน) ของ**ทุก dependency ในทุกระดับของ dependency tree**
(รวม transitive dependency คือ dependency ของ dependency ด้วย) ทำให้ตราบใดที่ `Cargo.lock` ยังอยู่ ทุกครั้งที่
เรียก `cargo build` บนเครื่องไหนก็ตาม จะได้ dependency เวอร์ชันเดียวกันเป๊ะ ๆ เสมอ — นี่คือแนวคิดเดียวกับ
`package-lock.json` ของ npm หรือ `poetry.lock`/`Pipfile.lock` ของ Python ถ้าคุณคุ้นเคยกับระบบนิเวศเหล่านั้นมาก่อน

**กฎ rule of thumb เรื่อง commit `Cargo.lock` เข้า git**:

- **โปรเจกต์ที่เป็น binary/application** (มี `src/main.rs`, ตั้งใจให้รันเป็นโปรแกรมจบในตัว เช่น CLI tool, web
  server) → **ควร commit `Cargo.lock` เข้า git เสมอ** เหตุผลคือแอปพลิเคชันปลายทางต้องการความแน่นอน (reproducible
  build) สูงสุด — คุณอยากให้ deploy บน production วันนี้กับ deploy อีกครั้งเดือนหน้าด้วยโค้ด commit เดียวกัน
  ได้ dependency เวอร์ชันเดียวกันเป๊ะ ไม่อยากให้มีตัวแปรที่ควบคุมไม่ได้จาก crates.io มาทำให้ behavior เปลี่ยนโดย
  ไม่รู้ตัว
- **โปรเจกต์ที่เป็น library** (มี `src/lib.rs` ตั้งใจให้โปรเจกต์อื่นมา depend on) → **โดยธรรมเนียมทั่วไปมักไม่
  commit `Cargo.lock`** เหตุผลเชิงเทคนิคคือ **cargo จะไม่สนใจ `Cargo.lock` ของ library ที่เป็น dependency เลย**
  — เมื่อโปรเจกต์ A ของคุณ depend on library B, สิ่งที่ถูกใช้ตัดสินเวอร์ชันจริง ๆ คือ `Cargo.lock` ที่ root ของ
  A (binary ปลายทาง) เท่านั้น `Cargo.lock` ที่อยู่ใน repo ของ library B จะถูกใช้แค่ตอนที่คุณ build/test library
  B **แบบเดี่ยว ๆ ในตัวมันเอง** เท่านั้น ดังนั้นการ commit มันเข้า repo ของ library แทบไม่มีผลต่อผู้ใช้ปลายทางที่
  มาเรียก dependency ของคุณเลย แต่กลับเพิ่ม noise ให้ diff ใน pull request (เพราะ `Cargo.lock` ของ library จะ
  เปลี่ยนบ่อยตาม dependency ของ dependency ที่อัปเดต) และอาจทำให้ contributor เข้าใจผิดว่าเวอร์ชันถูกล็อกตายตัว
  ทั้งที่จริง ๆ library ควรออกแบบให้ทำงานได้กับ**ช่วง**เวอร์ชันที่กว้างพอสมควรตามที่ระบุใน `Cargo.toml`

ข้อยกเว้นที่พบได้: บางทีมเลือก commit `Cargo.lock` ของ library ด้วยเหมือนกัน เพื่อให้ CI ของ library เอง
build ด้วยเวอร์ชันที่แน่นอน ทดสอบซ้ำได้เสมอ (ลดโอกาสที่ CI จะ fail แบบไม่เกี่ยวกับโค้ดที่เพิ่งแก้) — นี่ไม่ผิด
และเป็น trade-off ที่สมเหตุสมผล เพียงแต่กฎ **rule of thumb แบบดั้งเดิม** ที่หลักสูตรนี้อยากให้คุณจำไว้เป็นหลักคือ
**"binary → commit, library → ปกติไม่ต้อง"** เพราะมันตอบโจทย์ "จุดประสงค์การใช้งานของ dependency นั้น" ได้
ตรงที่สุด (แอปต้องการความแน่นอน, library ต้องการความยืดหยุ่นให้ผู้ใช้เลือกเวอร์ชันเอง) เราจะพูดถึงประเด็นนี้อีกครั้ง
พร้อมตัวอย่างจริงจังกว่าใน **Part 17** (Packages, Crates, Workspaces) และ **Part 35** (Cargo ขั้นสูง)

### 2.7 การเพิ่ม dependency: แก้ไฟล์ตรง ๆ vs cargo add

มาลองเพิ่ม dependency จริง ๆ กันในโปรเจกต์ `hello_cargo` — เราจะใช้ crate ชื่อ **`rand`** ซึ่งเป็น crate มาตรฐาน
ที่นักพัฒนา Rust แทบทุกคนต้องเคยใช้สำหรับสุ่มตัวเลข (standard library ของ Rust **ไม่มี** โมดูลสุ่มเลขในตัว
เพราะทีม Rust core ตั้งใจแยกฟีเจอร์ที่ไม่ใช่ทุกโปรแกรมต้องใช้ออกจาก std ให้เป็น crate แยก — ปรัชญานี้ทำให้ std
library เล็กและ maintain ง่าย)

#### วิธีที่ 1: แก้ `Cargo.toml` ด้วยมือ

เปิดไฟล์ `Cargo.toml` แล้วเพิ่มบรรทัดใน `[dependencies]`:

```toml
[package]
name = "hello_cargo"
version = "0.1.0"
edition = "2021"

[dependencies]
rand = "0.8"
```

แค่แก้ไฟล์แล้ว save เฉย ๆ — dependency จะยังไม่ถูกดาวน์โหลดจนกว่าคุณจะรัน `cargo build`, `cargo run`, หรือ
`cargo check` ครั้งต่อไป ลองรัน:

```bash
cargo build
```

```
    Updating crates.io index
  Downloaded rand v0.8.5
  Downloaded rand_core v0.6.4
  Downloaded getrandom v0.2.15
  Downloaded ppv-lite86 v0.2.20
  Downloaded libc v0.2.161
   Compiling libc v0.2.161
   Compiling getrandom v0.2.15
   Compiling rand_core v0.6.4
   Compiling ppv-lite86 v0.2.20
   Compiling rand v0.8.5
   Compiling hello_cargo v0.1.0 (/home/user/hello_cargo)
    Finished dev [unoptimized + debuginfo] target(s) in 3.21s
```

สังเกตว่า cargo ดาวน์โหลด**มากกว่า 1 crate** ทั้งที่เราสั่งเพิ่มแค่ `rand` ตัวเดียว — นี่คือ **transitive
dependency** (dependency ของ dependency) `rand` เองก็ใช้ `rand_core`, `getrandom`, `ppv-lite86`, `libc`
ภายใน cargo จะไล่ resolve dependency tree ทั้งหมดให้อัตโนมัติ คุณไม่ต้องมาคอยเพิ่มเองทีละตัว และไฟล์
`Cargo.lock` ก็จะถูกอัปเดตให้บันทึกทุกเวอร์ชันที่ resolve ได้เอาไว้ทั้งหมด (ตามที่อธิบายในหัวข้อ 2.6)

#### วิธีที่ 2: ใช้คำสั่ง `cargo add` (สะดวกกว่า, แนะนำ)

`cargo add` เป็นความสามารถ built-in ของ cargo มาตั้งแต่เวอร์ชัน 1.62 (กลางปี 2022) — ก่อนหน้านั้นต้องติดตั้ง
tool เสริมชื่อ `cargo-edit` แยกต่างหาก ปัจจุบันไม่ต้องแล้วเพราะรวมเข้า cargo core ไปเรียบร้อย:

```bash
cargo add rand
```

```
    Updating crates.io index
      Adding rand v0.8.5 to dependencies
             Features:
             + std
             + std_rng
             ...
```

คำสั่งนี้จะไปค้นหาเวอร์ชันล่าสุดของ `rand` บน crates.io ให้อัตโนมัติ แล้วเขียนบรรทัดลงใน `[dependencies]`
ของ `Cargo.toml` ให้เสร็จเลย (พร้อม comment แสดง feature ที่เปิดอยู่ด้วย ถ้า crate นั้นรองรับ feature flag —
เรื่อง feature flag จะเรียนใน Part 35) ไม่ต้องพิมพ์ TOML เอง ลดโอกาสพิมพ์ผิด

ตัวเลือกที่ใช้บ่อยของ `cargo add`:

```bash
cargo add rand@0.8          # ระบุเวอร์ชันที่ต้องการชัดเจน (แทนล่าสุดอัตโนมัติ)
cargo add criterion --dev   # เพิ่มเข้า [dev-dependencies] แทน [dependencies]
cargo add tonic-build --build   # เพิ่มเข้า [build-dependencies]
cargo add rand --dry-run    # ดูตัวอย่างว่าจะเพิ่มอะไร โดยไม่แก้ไฟล์จริง (ทดสอบก่อนตัดสินใจ)
```

หลักสูตรนี้จะใช้ทั้งสองวิธีปนกันไปตามความเหมาะสม — บทที่เน้นความเข้าใจโครงสร้างไฟล์ TOML แบบละเอียดจะแก้ด้วยมือ
ให้เห็นภาพชัด ส่วนบทที่เน้นความเร็วในการทำงานจะใช้ `cargo add`

### 2.8 Semantic Versioning: ไวยากรณ์การระบุเวอร์ชันใน Cargo.toml

เวลาเขียน `rand = "0.8"` ใน `Cargo.toml` เครื่องหมาย/รูปแบบตัวเลขที่ใส่ไปมีความหมายเจาะจงมาก ไม่ใช่แค่
"เวอร์ชันนี้เท่านั้น" cargo ใช้ไวยากรณ์ตาม [SemVer](https://semver.org/) ที่มีกฎดังนี้:

| ไวยากรณ์ | ชื่อเรียก | ความหมาย | ตัวอย่างเวอร์ชันที่ยอมรับ |
|---|---|---|---|
| `"1.2.3"` หรือ `"^1.2.3"` | Caret (default) | `>=1.2.3, <2.0.0` | 1.2.3, 1.5.0, 1.99.0 (ไม่ใช่ 2.0.0) |
| `"1.2"` | Caret (ละ patch) | `>=1.2.0, <2.0.0` | 1.2.0, 1.9.9 |
| `"1"` | Caret (ละ minor) | `>=1.0.0, <2.0.0` | 1.0.0, 1.99.99 |
| `"=1.2.3"` | Exact | เวอร์ชัน `1.2.3` เท่านั้น ห้ามขยับเลย | 1.2.3 (เท่านั้น) |
| `"~1.2.3"` | Tilde | `>=1.2.3, <1.3.0` (อนุญาตแค่ patch update) | 1.2.3, 1.2.9 (ไม่ใช่ 1.3.0) |
| `"~1.2"` | Tilde (ละ patch) | `>=1.2.0, <1.3.0` | 1.2.0, 1.2.9 |
| `">=1.2, <1.5"` | Comparison range | ระบุช่วงตรง ๆ เอง | 1.2.0 ถึง 1.4.x |

**ตรรกะเบื้องหลัง caret requirement (ค่า default)**: SemVer กำหนด convention ไว้ว่า **การเปลี่ยน MAJOR version
(ตัวแรก) หมายถึง breaking change** (API เปลี่ยนจนโค้ดเดิมอาจ compile ไม่ผ่านหรือ behavior เปลี่ยน) ส่วนการเปลี่ยน
MINOR (ตัวกลาง) ต้อง**ไม่ทำให้โค้ดเดิมพังได้ (backward compatible)** — เป็นแค่การ "เพิ่มฟีเจอร์ใหม่" และ PATCH
(ตัวท้าย) คือ bug fix ล้วน ๆ ไม่มี API เปลี่ยนเลย ด้วย convention นี้ caret requirement `"1.2"` จึงปลอดภัยที่จะ
บอกว่า "รับเวอร์ชันไหนก็ได้ที่ >= 1.2.0 แต่ < 2.0.0" เพราะตาม convention แล้วทุกเวอร์ชันในช่วงนี้**ควร**ใช้แทนกัน
ได้โดยไม่พังโค้ดเรา — นี่คือเหตุผลที่ cargo ใช้ caret เป็นค่า default (ไม่ต้องเขียน `^` ก็ได้ผลเหมือนกัน) เพราะ
มันให้ความยืดหยุ่นที่สมดุลที่สุดระหว่าง "ได้ bug fix/security patch ใหม่ ๆ อัตโนมัติ" กับ "ไม่เสี่ยงโดน breaking
change แบบไม่รู้ตัว"

**กรณีที่ควรใช้ `=` (exact pin)**: เมื่อคุณรู้ว่า dependency ตัวนั้นมีบั๊กในบางเวอร์ชัน หรือโปรเจกต์คุณอ่อนไหวต่อ
การเปลี่ยนแปลงพฤติกรรมเล็กน้อยมาก (เช่น library ที่ผลลัพธ์ floating-point ต้องเหมือนกันเป๊ะข้ามเวอร์ชันสำหรับงาน
วิทยาศาสตร์/การเงิน) การ pin แบบ exact จะป้องกันไม่ให้ `cargo update` เปลี่ยนเวอร์ชันโดยไม่ได้ตั้งใจ แต่ข้อเสีย
คือคุณจะไม่ได้รับ security patch อัตโนมัติ ต้องอัปเดต manual เอง — โดยทั่วไปจึงไม่แนะนำให้ pin exact พร่ำเพรื่อ
ควรใช้เฉพาะกรณีที่มีเหตุผลเจาะจงจริง ๆ

**กรณีที่ควรใช้ `~` (tilde)**: เมื่อต้องการความระมัดระวังกว่า caret เล็กน้อย คือยอมรับแค่ patch update
(bug fix) แต่ไม่ยอมรับ minor update (feature ใหม่) แม้ว่า SemVer บอกว่า minor update ไม่ควรพังอะไร แต่ในทางปฏิบัติ
บาง library อาจไม่ได้ยึด SemVer เคร่งครัดขนาดนั้นจริง ๆ — tilde requirement จึงเป็นตัวเลือกกลาง ๆ ที่บางทีมเลือกใช้
เพื่อความปลอดภัยเพิ่มขึ้นอีกระดับ

### 2.9 crates.io: ศูนย์กลาง package ของ Rust

[crates.io](https://crates.io) คือ **package registry กลาง** ของ Rust ecosystem — เทียบเท่ากับ npm สำหรับ
JavaScript, PyPI สำหรับ Python, หรือ Maven Central สำหรับ Java เมื่อคุณเขียน `rand = "0.8"` ใน `Cargo.toml`
โดยไม่ระบุ source อื่น cargo จะไปค้นหาและดาวน์โหลด crate ตัวนั้นจาก crates.io โดยอัตโนมัติ

ทุก crate ที่ publish ขึ้น crates.io จะมีหน้าเว็บของตัวเอง (เช่น https://crates.io/crates/rand) ที่แสดง:

- README และคำอธิบายการใช้งาน
- จำนวนการดาวน์โหลดทั้งหมด (บอกความนิยม/ความน่าเชื่อถือในระดับหนึ่ง)
- รายการเวอร์ชันทั้งหมดที่เคย publish
- ลิงก์ไปหน้า documentation ที่ generate อัตโนมัติบน [docs.rs](https://docs.rs) (เว็บไซต์ที่ compile
  documentation ของทุก crate บน crates.io ให้อัตโนมัติ ไม่ต้องเชื่อใจ README เจ้าของ crate เขียนเองอย่างเดียว)
- source repository (มักลิงก์ไป GitHub)

มาลองเขียนโปรแกรมเล็ก ๆ ที่ใช้ `rand` จริงกัน — แก้ไฟล์ `src/main.rs` ในโปรเจกต์ `hello_cargo` (ที่มี dependency
`rand = "0.8"` แล้วจากหัวข้อ 2.7):

```rust
use rand::Rng;

fn main() {
    // สร้างตัวสุ่มที่ผูกกับ thread ปัจจุบัน (เร็ว เพราะไม่ต้อง synchronize ข้าม thread)
    let mut rng = rand::thread_rng();

    // สุ่มเลขจำนวนเต็มในช่วง 1 ถึง 100 (inclusive ทั้งสองด้าน เพราะใช้ ..= )
    let secret_number = rng.gen_range(1..=100);
    println!("เลขลับที่สุ่มได้: {secret_number}");

    // สุ่มทอยลูกเต๋า 6 หน้า 5 ครั้ง แล้วเก็บผลไว้ใน Vec
    let mut dice_rolls: Vec<u8> = Vec::new();
    for _ in 0..5 {
        let roll: u8 = rng.gen_range(1..=6);
        dice_rolls.push(roll);
    }
    println!("ผลทอยลูกเต๋า 5 ครั้ง: {dice_rolls:?}");
}
```

รันด้วย `cargo run`:

```
เลขลับที่สุ่มได้: 74
ผลทอยลูกเต๋า 5 ครั้ง: [3, 6, 1, 4, 2]
```

(ตัวเลขจะสุ่มต่างกันไปทุกครั้งที่รัน — นั่นคือจุดประสงค์ของมัน)

**อธิบายโค้ดทีละส่วน**:

- `use rand::Rng;` — `Rng` คือ **trait** (จะเรียนละเอียดใน Part 19) ที่นิยาม method อย่าง `gen_range` เอาไว้
  ต้อง `use` trait เข้ามาก่อนถึงจะเรียก method ของมันได้ ถ้าลืมบรรทัดนี้ compiler จะฟ้อง error ทันที (ลองลบบรรทัด
  นี้ออกดูจะได้ error `no method named 'gen_range' found` — เป็นตัวอย่าง error ที่มือใหม่เจอบ่อยเวลาลืม import
  trait ที่จำเป็น)
- `rand::thread_rng()` — เรียกฟังก์ชันจาก module `thread_rng` ของ crate `rand` เพื่อสร้าง random number
  generator ตัวหนึ่ง ผูกกับ thread ปัจจุบันของโปรแกรม (ยังไม่ต้องเข้าใจเรื่อง thread ลึก ๆ ตอนนี้ — จะเรียนใน
  Part 37 — แค่รู้ว่ามันคือแหล่งสุ่มเลขที่ efficient สำหรับโปรแกรมทั่วไป)
- `rng.gen_range(1..=100)` — `1..=100` คือ Range แบบ inclusive ทั้งสองด้าน (มีเลข 1 และ 100 รวมอยู่ในช่วงด้วย)
  ต่างจาก `1..100` ที่จะไม่รวม 100 (exclusive ด้านขวา) — Rust มีทั้งสองรูปแบบ Range ให้เลือกใช้ตามความเหมาะสม
- `Vec<u8>` — คือ dynamic array ที่เก็บตัวเลขชนิด `u8` (จำนวนเต็มไม่ติดลบ 8-bit, ค่า 0-255) เราจะเรียน `Vec<T>`
  แบบละเอียดใน Part 13 ตอนนี้แค่รู้ว่ามันคือ list ที่ขยายขนาดได้ตอน runtime
- `{dice_rolls:?}` — เครื่องหมาย `:?` ใน format string เรียกว่า **Debug format** ใช้พิมพ์ค่าที่ไม่มี `Display`
  implementation แบบสวยงามอ่านง่าย เหมาะสำหรับ debug (พิมพ์ `Vec` ตรง ๆ ด้วย `{}` ธรรมดาจะ compile ไม่ผ่าน เพราะ
  `Vec<u8>` ไม่ implement trait `Display` แต่ implement `Debug` ให้อัตโนมัติ) — เราจะเข้าใจความต่างของ
  `Display` กับ `Debug` แบบละเอียดใน Part 19

นี่คือตัวอย่างที่แสดงพลังของ crates.io ชัดเจน: แทนที่จะต้องเขียน algorithm สุ่มเลขคุณภาพดีเองจากศูนย์ (ซึ่งทำได้
ยากกว่าที่คิด ถ้าอยากได้ความสุ่มที่มีคุณภาพทางสถิติดีจริง ๆ) เราแค่เพิ่ม dependency บรรทัดเดียวก็ได้ library
ที่ผ่านการทดสอบและใช้งานจริงจากคนนับล้านมาแล้ว

### 2.10 Workspaces: ภาพรวมสั้น ๆ (รายละเอียดเต็มใน Part 17)

เมื่อโปรเจกต์ใหญ่ขึ้นจนอยากแยกเป็นหลาย crate ที่ทำงานร่วมกัน (เช่น มี crate `core` เก็บ business logic หลัก,
crate `cli` เป็นโปรแกรม command-line ที่เรียกใช้ `core`, และ crate `web_server` ที่เรียกใช้ `core` เหมือนกันแต่
ให้บริการผ่าน HTTP) Cargo มีฟีเจอร์ชื่อ **workspace** สำหรับจัดการหลาย package ภายใต้ `Cargo.lock` และ
`target/` ชุดเดียวกัน

โครงสร้างคร่าว ๆ ของ workspace หน้าตาแบบนี้:

```
my_workspace/
├── Cargo.toml          <- root manifest ระบุ [workspace] และ members
├── Cargo.lock          <- ล็อกเวอร์ชันร่วมกันทุก member
├── target/              <- โฟลเดอร์ build ผลลัพธ์ร่วมกันทุก member
├── core/
│   ├── Cargo.toml
│   └── src/lib.rs
├── cli/
│   ├── Cargo.toml
│   └── src/main.rs
└── web_server/
    ├── Cargo.toml
    └── src/main.rs
```

ไฟล์ `Cargo.toml` ที่ root จะมีหน้าตาประมาณนี้ (แทนที่จะมี `[package]` แบบปกติ):

```toml
[workspace]
members = ["core", "cli", "web_server"]
resolver = "2"
```

ข้อดีหลักของ workspace: (1) ทุก member ใช้ dependency เวอร์ชันเดียวกันได้ผ่าน `Cargo.lock` ร่วมกัน ลดปัญหา
เวอร์ชันชนกัน (2) compile ครั้งเดียวได้ทุก crate โดยแชร์ intermediate build artifact ใน `target/` เดียวกัน
ทำให้ compile เร็วกว่าแยกเป็นโปรเจกต์เดี่ยว ๆ หลายอันมาก (3) crate ภายใน workspace อ้างอิงกันเองได้ง่ายผ่าน
**path dependency** เช่น `core = { path = "../core" }` แทนที่จะต้อง publish ขึ้น crates.io ก่อนถึงจะใช้ได้

เราจะเจาะลึกเรื่อง workspace, path dependency, และการออกแบบโครงสร้างโปรเจกต์แบบ multi-crate อย่างเต็มรูปแบบ
ใน **Part 17: Packages, Crates, Workspaces** ตอนนี้แค่รู้ไว้ว่ามันมีอยู่และมีไว้แก้ปัญหาอะไร

### 2.11 Build Profiles: ภาพรวมสั้น ๆ (รายละเอียดเต็มใน Part 35)

จากหัวข้อ 2.5 เราเห็นแล้วว่า cargo มี profile `dev` และ `release` เป็นค่า default ที่ปรับแต่งค่าต่าง ๆ ได้
ผ่าน `[profile.dev]` และ `[profile.release]` ใน `Cargo.toml` — คีย์ที่ปรับได้มีมากกว่าที่เห็นในตัวอย่างก่อนหน้า
มาก เช่น:

```toml
[profile.dev]
opt-level = 0      # ไม่ optimize เลย เน้น compile เร็วที่สุด
debug = true       # เก็บ debug symbol ไว้เต็มรูปแบบ
overflow-checks = true   # panic ทันทีถ้าเลข integer overflow (ช่วยจับบั๊กตั้งแต่ dev)

[profile.release]
opt-level = 3      # optimize สูงสุดเพื่อความเร็วตอนรัน
debug = false      # ไม่เก็บ debug symbol เพื่อลดขนาดไฟล์
overflow-checks = false  # ปิดการเช็ค overflow เพื่อความเร็ว (ยอมรับ wrap-around แบบเงียบ)
```

นอกจากนี้ยังสร้าง **custom profile** ของตัวเองได้ (เช่น profile `staging` ที่ optimize ปานกลางแต่ยังเก็บ debug
info ไว้บางส่วน สำหรับทดสอบก่อน deploy จริง) และปรับ dependency แต่ละตัวแยกจาก profile หลักได้ด้วย — เนื้อหา
เชิงลึกเหล่านี้ (รวมถึง `codegen-units`, `panic = "abort"` vs `"unwind"`, `strip`) จะอยู่ใน **Part 35: Cargo
ขั้นสูง** ซึ่งเป็นจุดที่เหมาะสมกว่า เพราะต้องอาศัยความเข้าใจเรื่อง panic handling (Part 12) และการ deploy จริง
มาก่อน

## กับดักที่พบบ่อย (Common Pitfalls)

**1. พิมพ์ชื่อ crate ผิด**

ถ้าพิมพ์ชื่อ crate ผิดใน `Cargo.toml` เช่น พิมพ์ `rnd` แทน `rand`:

```toml
[dependencies]
rnd = "0.8"
```

รัน `cargo build` จะได้ error แบบนี้:

```
error: no matching package named `rnd` found
location searched: crates.io index
required by package `hello_cargo v0.1.0 (/home/user/hello_cargo)`
```

**วิธีแก้**: ตรวจสอบชื่อ crate ให้ตรงกับที่แสดงบนหน้า crates.io ทุกตัวอักษร (ชื่อ crate case-sensitive และ
บางครั้งใช้ `-` ปนกับ `_` ต่างกันในแต่ละ crate — ที่หน้า crates.io ของ crate นั้นจะมีบรรทัด "Add this to your
Cargo.toml" ที่ copy วางได้ตรง ๆ เสมอ ปลอดภัยกว่าพิมพ์เอง)

**2. เขียน version requirement ผิดไวยากรณ์**

ถ้าพิมพ์ผิดแบบใส่ `^` ซ้ำ (เข้าใจผิดคิดว่ายิ่งใส่มากยิ่งการันตีเวอร์ชันแม่นขึ้น):

```toml
[dependencies]
rand = "^^0.8"
```

จะได้ error ตอน build:

```
error: failed to parse the version requirement `^^0.8` for dependency `rand`

Caused by:
  unexpected character '^' while parsing major version number
```

**วิธีแก้**: caret requirement เขียนแค่ `^0.8` (หรือละ `^` ไปเลยเพราะเป็น default) เครื่องหมายพิเศษ (`^`, `~`,
`=`) ใส่ได้แค่ตัวเดียวนำหน้าตัวเลขเท่านั้น อ้างอิงตารางในหัวข้อ 2.8 ทุกครั้งที่ไม่แน่ใจ

**3. แก้ `Cargo.lock` ด้วยมือ**

บางคนเห็น `Cargo.lock` เป็นไฟล์ text ธรรมดาแล้วลองแก้เวอร์ชันในนั้นตรง ๆ เพื่อบังคับเปลี่ยนเวอร์ชัน dependency
— **อย่าทำ** เพราะไฟล์นี้มี checksum และโครงสร้าง dependency graph ที่ซับซ้อน การแก้มือมีโอกาสสูงที่จะทำให้ไฟล์
ไม่ตรงกับความเป็นจริง แล้ว cargo จะเขียนทับกลับเป็นค่าที่ถูกต้องเองอยู่ดีในการ build ครั้งถัดไป (ทำให้การแก้มือ
ไม่มีผลจริง หรือแย่กว่านั้นคือทำให้ error แปลก ๆ) วิธีที่ถูกต้องในการเปลี่ยนเวอร์ชันคือแก้ `Cargo.toml` แล้วรัน
`cargo update -p rand` (อัปเดตแค่ `rand` ตัวเดียวให้ตรง constraint ใหม่) หรือ `cargo update` (อัปเดตทุก
dependency ที่ทำได้ภายใต้ constraint เดิม)

**4. ลืมว่า binary อยู่ใน `target/debug/` ไม่ใช่ root โปรเจกต์**

มือใหม่บางคนหลังรัน `cargo build` แล้วพิมพ์ `./hello_cargo` ตรง ๆ ที่ root โปรเจกต์ จะได้:

```
bash: ./hello_cargo: No such file or directory
```

**วิธีแก้**: ต้องเรียกที่ path เต็ม `./target/debug/hello_cargo` (หรือใช้ `cargo run` ที่จัดการเรื่องนี้ให้
อัตโนมัติ ซึ่งเป็นวิธีที่แนะนำเสมอในระหว่างพัฒนา)

**5. commit โฟลเดอร์ `target/` เข้า git โดยไม่ได้ตั้งใจ**

ถ้าเผลอลบไฟล์ `.gitignore` ที่ cargo สร้างให้ทิ้ง หรือทำงานในโฟลเดอร์ที่ `.gitignore` ไม่ครอบคลุม `target/`
คุณอาจ commit ไฟล์ binary/object file หลายพันไฟล์ ขนาดหลายร้อย MB เข้า repository โดยไม่รู้ตัว — ตรวจสอบด้วย
`git status` ก่อน commit เสมอ ถ้าเห็น `target/` โผล่มาในรายการไฟล์ที่ยังไม่ track ให้ตรวจสอบไฟล์
`.gitignore` ว่ามีบรรทัด `/target` อยู่จริงหรือไม่

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** ในโปรเจกต์ `hello_cargo` (ที่ยังไม่มี dependency ใด ๆ) ให้รัน `cargo check`, `cargo build`, และ
   `cargo run` ตามลำดับ สังเกตความแตกต่างของเวลาที่ใช้และข้อความที่แสดงในแต่ละคำสั่ง จากนั้นเข้าไปดูภายในโฟลเดอร์
   `target/debug/` ว่ามีไฟล์อะไรถูกสร้างขึ้นมาบ้าง (hint: ใช้ `ls -la target/debug/` บน macOS/Linux หรือ
   `dir target\debug\` บน Windows)

2. **[กลาง]** เพิ่ม dependency `rand` เข้าโปรเจกต์ `hello_cargo` ด้วยคำสั่ง `cargo add rand` แล้วเขียนโปรแกรม
   ที่จำลองการสุ่มไพ่ 1 ใบจาก 52 ใบ (สุ่มเลข 1-52 แล้วแปลงเป็นชื่อไพ่ เช่น เลข 1 = "A ของ Spade") ทำซ้ำ 5 รอบ
   แล้วพิมพ์ผลลัพธ์ทั้งหมด (hint: ใช้ `rng.gen_range(1..=52)` แล้วใช้ integer division กับ modulo (`/` และ `%`)
   เพื่อแยกว่าเป็นไพ่หมายเลขอะไรของดอกอะไร — ยังไม่ต้องสมบูรณ์แบบ 100% เป้าหมายคือฝึกใช้ dependency จริง)

3. **[กลาง-ยาก]** สร้าง library crate ใหม่ด้วย `cargo new my_utils --lib` เขียนฟังก์ชัน public ชื่อ
   `is_prime(n: u64) -> bool` ที่ตรวจสอบว่าตัวเลขเป็นจำนวนเฉพาะหรือไม่ พร้อมเขียน unit test อย่างน้อย 3 เคส
   ในบล็อก `#[cfg(test)] mod tests { ... }` (ดูตัวอย่างโครงจาก `cargo new --lib` เป็นแนวทาง) แล้วรัน
   `cargo test` เพื่อดูว่า test ผ่านหมดหรือไม่ (ยังไม่ต้องเข้าใจ syntax ของ `#[test]` ลึกซึ้ง แค่ลองทำตามโครงที่
   cargo generate ให้ — จะเรียนรายละเอียดเต็มใน Part 32)

4. **[ยาก/ประยุกต์]** ในโปรเจกต์จากข้อ 2 ให้เปลี่ยน version requirement ของ `rand` ใน `Cargo.toml` จาก `"0.8"`
   เป็น `"=0.8.3"` (เจาะจงเวอร์ชันแน่นอน) แล้วรัน `cargo update -p rand --precise 0.8.3` จากนั้นเปิดไฟล์
   `Cargo.lock` ดูว่าเวอร์ชันของ `rand` ที่บันทึกไว้เปลี่ยนไปตามที่คาดหรือไม่ เขียนสรุปสั้น ๆ (3-5 บรรทัด) อธิบาย
   ว่าทำไมการ pin เวอร์ชันแบบ exact (`=`) ถึงมีทั้งข้อดีและข้อเสีย โดยเชื่อมโยงกับสถานการณ์การ deploy โปรแกรม
   production จริง (เช่น ถ้า `rand` ปล่อยเวอร์ชัน 0.8.4 ที่มีบั๊กร้ายแรงออกมา โปรเจกต์ที่ pin แบบ `=0.8.3` จะได้
   รับผลกระทบหรือไม่ เทียบกับโปรเจกต์ที่ใช้ `"0.8"` แบบ default)

## สรุป

ในบทนี้เราได้เจาะลึกเข้าไปในหัวใจของ Rust ecosystem นั่นคือ **Cargo** เราเข้าใจโครงสร้างของ `Cargo.toml`
ทุก section (`[package]`, `[dependencies]`, `[dev-dependencies]`, `[build-dependencies]`) รู้ความแตกต่างของ
`cargo new`/`cargo init` และระหว่าง binary crate (`src/main.rs`) กับ library crate (`src/lib.rs`) คล่องแคล่ว
กับคำสั่งหลักที่จะใช้ทุกวัน (`build`, `run`, `check`, `doc --open`, `clean`) เข้าใจว่า debug build กับ release
build ต่างกันอย่างไรและทำไม เข้าใจบทบาทของ `Cargo.lock` ในการทำให้ build ทำซ้ำได้ และรู้วิธีเพิ่ม dependency
จริงจาก crates.io ทั้งแบบแก้ไฟล์เองและแบบใช้ `cargo add` พร้อมไวยากรณ์ semantic versioning ที่ควบคุมว่าเวอร์ชัน
ไหนอัปเดตได้บ้าง สุดท้ายเรายังได้แอบดูภาพรวมของ workspace และ build profile ไว้เป็นแนวทางสำหรับบทที่ลึกกว่าต่อไป

ตอนนี้คุณมีเครื่องมือครบมือแล้วสำหรับการเริ่มเขียนโค้ด Rust จริงจัง ใน **Part 3** เราจะเริ่มลงลึกในตัวภาษา Rust
เอง โดยเริ่มจากพื้นฐานที่สุด: **ตัวแปร ชนิดข้อมูลพื้นฐาน (scalar และ compound type) และแนวคิด mutability** —
ทำไม Rust ถึงกำหนดให้ตัวแปรเป็น immutable โดย default ต่างจากภาษาส่วนใหญ่ที่คุณอาจคุ้นเคยมาก่อน

---

**Part ก่อนหน้า:** [Part 1: แนะนำ Rust และการติดตั้งเครื่องมือ](part-001-intro-and-setup.md) | **Part ถัดไป:** [Part 3: ตัวแปร ชนิดข้อมูลพื้นฐาน และ Mutability](part-003-variables-and-data-types.md)
