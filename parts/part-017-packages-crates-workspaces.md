# Part 17: Packages, Crates, Workspaces

> โมดูล: พื้นฐานภาษา Rust (Core Language Fundamentals) | ระดับ: กลาง | เวลาโดยประมาณ: 130 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายความแตกต่างระหว่างคำสามคำที่มือใหม่ Rust สับสนกันบ่อยที่สุด — **package**, **crate**, และ **module** —
  ได้อย่างแม่นยำ พร้อมบอกได้ว่าคำแต่ละคำ "คือไฟล์/แนวคิดไหน" ใน Cargo.toml และ source tree จริง
- อธิบายได้ว่าทำไม crate คือ "compilation unit" ที่ `rustc` มองเห็น และเชื่อมโยงกับแนวคิด module tree ที่เรียนจาก
  Part 16 ได้ว่าจริง ๆ แล้วมันคือต้นไม้เดียวกัน มองจากคนละมุม
- ออกแบบแพ็กเกจที่มีทั้ง library crate (`src/lib.rs`) และ binary crate (`src/main.rs`) อยู่ร่วมกัน และอธิบายได้ว่า
  ทำไม pattern "logic อยู่ใน lib, main.rs เป็นแค่เปลือกบาง ๆ" ถึงเป็น pattern มาตรฐานในโปรเจกต์ Rust จริง
- สร้างแพ็กเกจที่มีโปรแกรม executable หลายตัวพร้อมกันผ่าน `src/bin/*.rs` และรันแต่ละตัวแยกกันด้วย
  `cargo run --bin <name>`
- อธิบายได้ว่าเมื่อไรควรแตกโปรเจกต์เดียวออกเป็นหลายแพ็กเกจภายใต้ **Cargo workspace** เดียว และประกอบ workspace
  ที่มี library crate กับ binary crate หลายตัวอ้างอิงกันผ่าน **path dependency** ได้จริง
- ใช้คำสั่ง `cargo build`/`cargo test`/`cargo run` ในบริบทของ workspace ได้อย่างถูกต้อง ทั้งแบบรันทุก member
  (`--workspace`) และแบบเจาะจง member เดียว (`-p <package>`)
- เขียน `[dependencies]` แบบ expanded table syntax, ตั้งค่า `optional = true`, และออกแบบ `[features]` section
  พร้อมใช้ `#[cfg(feature = "...")]` ควบคุม conditional compilation ได้จริงในโค้ด

## ความรู้ที่ต้องมีมาก่อน

- **Part 2 (Cargo และโครงสร้างโปรเจกต์)**: ต้องเข้าใจกายวิภาคของ `Cargo.toml` แล้ว (`[package]`,
  `[dependencies]`, `[dev-dependencies]`, `[build-dependencies]`), รู้ความต่างระหว่าง `cargo new`/`cargo init`,
  รู้ว่า `src/main.rs` คือ binary crate root และ `src/lib.rs` คือ library crate root, เข้าใจไวยากรณ์ SemVer ของ
  version requirement (`^`, `~`, `=`), และเคยเห็นภาพรวมสั้น ๆ ของ workspace ในหัวข้อ 2.10 มาแล้ว (บทนี้จะเจาะลึก
  ต่อจากจุดนั้นแบบเต็มรูปแบบ ไม่ใช่ทวนของเดิม)
- **Part 16 (Modules และการจัดระเบียบโค้ด)**: ต้องเข้าใจ `mod`, `pub`, `use`, `super`, และคีย์เวิร์ด `crate`
  มาก่อน โดยเฉพาะแนวคิด **module tree** ภายในไฟล์เดียว/crate เดียว — บทนี้จะซูมออกมาอีกระดับหนึ่ง อธิบายว่า
  "crate ทั้งก้อน" ที่ Part 16 พูดถึงนั้น สัมพันธ์กับคำว่า "package" อย่างไร และหนึ่ง package จะมีได้กี่ crate

## เนื้อหา

### 17.1 ทบทวนปัญหาคำศัพท์: Package, Crate, Module สามคำที่มือใหม่สับสนที่สุด

ก่อนจะไปต่อ เราต้องหยุดพูดคำสามคำนี้ให้ชัดเจนที่สุดก่อน เพราะเอกสารภาษาอังกฤษ (รวมถึง error message ของ
compiler เอง) ใช้คำเหล่านี้สลับกันไปมาในบริบทต่างกัน ทำให้คนเรียนใหม่จำนวนมากงงว่า "มันคืออันเดียวกันหรือเปล่า"
คำตอบคือ **ไม่ใช่อันเดียวกัน** ทั้งสามคำอยู่กันคนละ "ระดับ" ของโครงสร้างโปรเจกต์ Rust:

```
Package (สิ่งที่ Cargo.toml อธิบาย)
  └── Crate (compilation unit ที่ rustc คอมไพล์)
        └── Module (การจัดกลุ่มโค้ดภายใน crate เดียว — เรียนไปแล้วใน Part 16)
```

มาดูนิยามของแต่ละคำแบบเจาะจงที่สุดเท่าที่จะทำได้:

**Package (แพ็กเกจ)** คือหน่วยที่ **Cargo** (ตัวจัดการโปรเจกต์/dependency) มองเห็นและจัดการ — พูดให้ตรงที่สุดคือ
**package คือสิ่งที่ไฟล์ `Cargo.toml` หนึ่งไฟล์อธิบาย** package มี `name` และ `version` ของตัวเอง (คีย์ใน
`[package]` ที่เราเรียนใน Part 2) และเป็นหน่วยที่ถูก publish ขึ้น crates.io ทีละหนึ่ง package เสมอ ทุกครั้งที่คุณ
รัน `cargo new` หรือ `cargo init` คุณกำลังสร้าง package หนึ่งใบขึ้นมา

**Crate (เครท)** คือหน่วยที่ **rustc** (ตัว compiler จริง ๆ) มองเห็นและคอมไพล์ — เรียกอย่างเป็นทางการว่า
**compilation unit** crate คือ "ต้นไม้ของ module" หนึ่งต้นที่มี **root** เดียว (root ก็คือไฟล์ `src/lib.rs` หรือ
`src/main.rs` หรือไฟล์ใน `src/bin/*.rs` แต่ละไฟล์) เวลา Part 16 อธิบายเรื่อง module tree ว่า `mod` แต่ละตัวคือ
กิ่งก้านของต้นไม้ นั่นคือ**ภาพของ crate เดียว** ที่คุณกำลังมองอยู่ — เราจะอธิบายจุดนี้ละเอียดในหัวข้อ 17.2

**Module (โมดูล)** คือหน่วยที่ใช้จัดกลุ่มโค้ด**ภายในหนึ่ง crate** — คือสิ่งที่ Part 16 สอนไปแล้วทั้งหมด (`mod foo
{ ... }`, `pub`, `use`, `super::`, คีย์เวิร์ด `crate::`) module ไม่มี version ของตัวเอง ไม่ถูก publish แยกจาก
crate ที่มันอยู่ และไม่มี `Cargo.toml` เป็นของตัวเอง — มันเป็นแค่การแบ่งโค้ด "ภายใน" crate หนึ่งก้อนเท่านั้น

ทำไมความสัมพันธ์ระหว่างสามคำนี้จึงสำคัญกับการทำงานจริง? เพราะคำถามที่มือใหม่ถามกันบ่อยที่สุดคือ **"หนึ่ง package
มีได้กี่ crate?"** และคำตอบที่ถูกต้องคือกฎที่หัวข้อถัดไปจะอธิบายอย่างเจาะจง — มันไม่ใช่ "1 ต่อ 1" เสมอไปอย่างที่
คนจำนวนมากเข้าใจผิด

### 17.2 กฎของ Cargo: หนึ่ง Package มีได้กี่ Crate?

นี่คือกฎที่ต้องจำให้แม่นเพราะเป็นแกนของทั้งบทนี้:

> **หนึ่ง package มีได้ library crate ได้ "อย่างมากที่สุด 1 ตัว" แต่มี binary crate ได้ "หลายตัวไม่จำกัด"**

มาดูรายละเอียดของกฎนี้ทีละส่วน:

#### Library crate: มีได้สูงสุด 1 ตัวต่อ package

Cargo ตัดสินว่า package หนึ่งมี library crate หรือไม่จากการมีไฟล์ `src/lib.rs` อยู่ (หรือระบุ path อื่นผ่านคีย์
`[lib]` ใน `Cargo.toml` ก็ได้ แต่ปกติแทบไม่มีใครทำแบบนั้น) เพราะ library คือ "หนึ่งหน่วย API" ที่ crate อื่นจะมา
`use` — การมี library มากกว่าหนึ่งตัวใน package เดียวจะทำให้ไม่รู้ว่าชื่อ package ที่คนอื่นเขียน `foo = "1.0"`
ใน `Cargo.toml` ของเขานั้นหมายถึง library ตัวไหน จึงเป็นข้อจำกัดที่ตั้งใจออกแบบไว้ ไม่ใช่บั๊ก

#### Binary crate: มีได้หลายตัวไม่จำกัดต่อ package

Cargo หา binary crate จากสองแหล่ง:

1. ไฟล์ `src/main.rs` เพียงไฟล์เดียว (ถ้ามี) → กลายเป็น binary crate ชื่อเดียวกับ package
2. ไฟล์ทุกไฟล์ที่วางอยู่ใน `src/bin/*.rs` → แต่ละไฟล์กลายเป็น binary crate แยกกัน ชื่อ crate จะตรงกับชื่อไฟล์
   (ไม่รวม `.rs`)

เพราะ binary แต่ละตัวคือ "โปรแกรมที่รันเองได้" หนึ่งโปรแกรม การมีโปรแกรม CLI หลายตัวที่เกี่ยวข้องกันอยู่ใน
package เดียวจึงเป็นเรื่องปกติมาก (เราจะเห็นตัวอย่างจริงในหัวข้อ 17.4)

#### เชื่อมกับ Part 16: crate คือ module tree หนึ่งต้น

จาก Part 16 คุณได้เรียนไปแล้วว่าไฟล์ `src/main.rs` (หรือ `src/lib.rs`) คือ **crate root** — จุดเริ่มต้นของ module
tree ที่ `mod` ประกาศกิ่งก้านต่อออกไป จุดที่บทนี้อยากให้คุณเห็นภาพชัดขึ้นคือ: **เมื่อ package มี library crate 1
ตัวและ binary crate 2 ตัว นั่นแปลว่า package นั้นมี module tree อยู่ "3 ต้น" ที่แยกจากกันโดยสมบูรณ์** — แต่ละ
crate มี root ของตัวเอง มี `mod` ของตัวเอง และ item หนึ่งใน crate หนึ่งจะมองไม่เห็น private item ของอีก crate
เลย แม้จะอยู่ใน package เดียวกันก็ตาม (จะมองเห็นได้เฉพาะสิ่งที่เป็น `pub` และถูก `use` ข้ามมาเท่านั้น เหมือนกับ
`use` ข้าม crate จากภายนอกทุกประการ) นี่คือเหตุผลที่คำว่า "crate" ใน error message ของ compiler (เช่น
`error[E0433]: failed to resolve: use of undeclared crate or module`) จึงหมายถึง **compilation unit** จริง ๆ
ไม่ใช่แค่คำเรียกลอย ๆ

ลองจินตนาการภาพรวมของ package ที่มีทั้ง lib และ 2 binary:

```
my_package/                  <- 1 package (มี Cargo.toml ใบเดียว)
├── Cargo.toml
└── src/
    ├── lib.rs                <- crate #1 (library crate, root ของ module tree ต้นที่ 1)
    ├── main.rs                <- crate #2 (binary crate ชื่อ "my_package", root ของ module tree ต้นที่ 2)
    └── bin/
        └── extra_tool.rs      <- crate #3 (binary crate ชื่อ "extra_tool", root ของ module tree ต้นที่ 3)
```

package นี้มี "ชื่อเดียว" (`name` ใน `[package]`) และ "version เดียว" แต่ compile ออกมาได้ **3 crate** ที่แยก
กันอย่างสิ้นเชิง — นี่คือคำตอบที่แม่นยำที่สุดของคำถาม "package ต่างจาก crate อย่างไร"

### 17.3 Pattern มาตรฐาน: Binary + Library ในแพ็กเกจเดียว

Part 2 หัวข้อ 2.3 เกริ่นไว้สั้น ๆ ว่า "แพ็กเกจเดียวมีทั้ง `main.rs` และ `lib.rs` พร้อมกันได้ เป็น pattern ที่พบบ่อย
มาก" บทนี้จะแสดงให้เห็นแบบเต็มรูปแบบว่าทำไม และทำอย่างไร

#### ทำไมต้องแยก logic ออกจาก main.rs?

ลองดูโปรแกรมที่เขียนแบบ "ยัดทุกอย่างไว้ใน main.rs" ก่อน เพื่อเห็นปัญหา — สมมติโปรแกรมคำนวณราคาสินค้าหลังหักส่วนลด
ในระบบร้านค้า:

```rust
// src/main.rs — เขียนทุกอย่างไว้ในที่เดียว (แบบที่ไม่แนะนำสำหรับโปรเจกต์จริง)
fn main() {
    let price = 250.0;
    let discount_percent = 15.0;

    let discount_amount = price * (discount_percent / 100.0);
    let final_price = price - discount_amount;

    println!("ราคาเดิม: {price:.2} บาท");
    println!("ส่วนลด: {discount_amount:.2} บาท");
    println!("ราคาสุทธิ: {final_price:.2} บาท");
}
```

ปัญหาของโค้ดแบบนี้คือ **logic การคำนวณส่วนลดถูกฝังอยู่ใน `main` โดยตรง ไม่มีทางเรียกใช้ logic นี้จากที่อื่นได้เลย
นอกจากรันโปรแกรมทั้งตัว** ถ้าอยากเขียน automated test เพื่อยืนยันว่าการคำนวณถูกต้อง (เช่น ทดสอบกรณีส่วนลด 0%,
100%, หรือราคาติดลบ) คุณจะเขียน test เรียกฟังก์ชันนี้ตรง ๆ ไม่ได้ เพราะ `main` ไม่ return ค่าอะไรที่ตรวจสอบได้
และการรัน `main` แต่ละครั้งก็ทำได้แค่ "รันทั้งโปรแกรม" ไม่ใช่ "เรียกเฉพาะส่วนคำนวณ"

#### วิธีแก้: ย้าย logic ไป lib.rs แล้วให้ main.rs เป็นเปลือกบาง ๆ

```rust
// src/lib.rs — เก็บ logic ทั้งหมดไว้ที่นี่ เป็น public API ที่ทดสอบได้และเรียกจากที่อื่นได้
/// คำนวณราคาสุทธิหลังหักส่วนลด คืนค่าเป็น tuple (ส่วนลดที่หักไป, ราคาสุทธิ)
pub fn calculate_discount(price: f64, discount_percent: f64) -> (f64, f64) {
    let discount_amount = price * (discount_percent / 100.0);
    let final_price = price - discount_amount;
    (discount_amount, final_price)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn no_discount_keeps_price_same() {
        let (discount, final_price) = calculate_discount(250.0, 0.0);
        assert_eq!(discount, 0.0);
        assert_eq!(final_price, 250.0);
    }

    #[test]
    fn fifteen_percent_discount() {
        let (discount, final_price) = calculate_discount(250.0, 15.0);
        assert_eq!(discount, 37.5);
        assert_eq!(final_price, 212.5);
    }

    #[test]
    fn full_discount_makes_price_zero() {
        let (_discount, final_price) = calculate_discount(250.0, 100.0);
        assert_eq!(final_price, 0.0);
    }
}
```

```rust
// src/main.rs — เหลือแค่หน้าที่ "รับ input, เรียก lib, แสดงผล" เท่านั้น ไม่มี business logic เลย
use shop_pricing::calculate_discount;

fn main() {
    let price = 250.0;
    let discount_percent = 15.0;

    let (discount_amount, final_price) = calculate_discount(price, discount_percent);

    println!("ราคาเดิม: {price:.2} บาท");
    println!("ส่วนลด: {discount_amount:.2} บาท");
    println!("ราคาสุทธิ: {final_price:.2} บาท");
}
```

สังเกตว่า `main.rs` เรียก `use shop_pricing::calculate_discount;` โดย `shop_pricing` คือชื่อ package (ตามที่
ตั้งใน `Cargo.toml`) — นี่คือจุดสำคัญที่ต้องเข้าใจ: **จาก "ภายนอก" library crate ของตัวเอง (คือจาก main.rs หรือ
จากไฟล์ใน src/bin/) คุณ `use` มันเหมือนกับที่คุณ `use` crate ภายนอกจาก crates.io ทุกประการ** เพราะ main.rs และ
lib.rs คือคนละ crate กันตามที่อธิบายในหัวข้อ 17.2 — ชื่อที่ใช้ `use` คือชื่อ package (แปลง `-` เป็น `_`
โดยอัตโนมัติถ้าชื่อ package มี hyphen เช่น package ชื่อ `shop-pricing` จะ `use` ด้วย `shop_pricing`)

`Cargo.toml` ของ package นี้ไม่ต้องเขียนอะไรพิเศษเลย เพราะ Cargo ตรวจจับ `src/lib.rs` และ `src/main.rs` ให้
อัตโนมัติทั้งคู่:

```toml
[package]
name = "shop_pricing"
version = "0.1.0"
edition = "2021"

[dependencies]
```

#### เหตุผลเชิงลึกที่ pattern นี้ถึงเป็นมาตรฐานในโลกจริง

1. **Testability (ทดสอบได้ง่ายกว่ามาก)** — อย่างที่เห็นในตัวอย่าง `#[cfg(test)] mod tests` เขียนอยู่ข้าง
   ฟังก์ชันใน `lib.rs` ได้ตรง ๆ และ `cargo test` จะรัน test เหล่านี้ได้ทันทีโดยไม่ต้องรันโปรแกรมทั้งตัว ไม่ต้อง
   จำลอง input ผ่าน stdin หรือ command-line argument เลย — ยิ่ง logic ซับซ้อนขึ้น ความต่างของความง่ายในการ
   ทดสอบจะยิ่งชัดมาก (เราจะเจาะลึกเรื่อง testing ใน Part 32-33)
2. **Reusability (เอาไป reuse ได้)** — ถ้าวันหนึ่งคุณอยากสร้างโปรแกรมที่สองที่ใช้ logic การคำนวณส่วนลดเดียวกัน
   (เช่น เว็บ API ที่ต้องคำนวณราคาแบบเดียวกัน) คุณสามารถเพิ่ม binary ตัวใหม่ (หัวข้อ 17.4) หรือสร้าง package ใหม่
   ที่มี path dependency ไปที่ library เดิม (หัวข้อ 17.7) ได้เลยโดยไม่ต้อง copy-paste โค้ดซ้ำ — ถ้า logic ถูกฝัง
   อยู่ใน `main.rs` การ reuse แบบนี้ทำไม่ได้เลย
3. **แยกความรับผิดชอบชัดเจน (separation of concerns)** — `main.rs` มีหน้าที่แค่ "ประกอบ" (wiring): รับ argument,
   เรียก library, จัดการ I/O (อ่านไฟล์, พิมพ์ผลลัพธ์, จัดการ error สำหรับผู้ใช้) ส่วน `lib.rs` มีหน้าที่แค่
   "คำนวณ/ตัดสินใจ" (business logic) ล้วน ๆ ไม่ยุ่งกับ I/O เลย — การแยกแบบนี้ทำให้อ่านโค้ดง่ายขึ้นมาก เพราะรู้ว่า
   ถ้าจะหาบั๊กเรื่องการคำนวณผิด ต้องไปดูที่ `lib.rs` แต่ถ้าโปรแกรม crash ตอนอ่านไฟล์ ต้องไปดูที่ `main.rs`
4. **อนุญาตให้คนอื่น depend on logic ของคุณโดยไม่ต้องพึ่ง CLI ของคุณ** — ถ้า `shop_pricing` เป็น library ที่ดี
   มากพอจนอยากแบ่งปันให้ทีมอื่นใช้ พวกเขาสามารถเพิ่ม `shop_pricing = "0.1"` ใน `Cargo.toml` ของโปรเจกต์ตัวเอง
   แล้วเรียก `calculate_discount` ได้ตรง ๆ โดยไม่ต้องสนใจว่า CLI ของคุณทำงานยังไง ไม่ต้องรันโปรแกรมของคุณเป็น
   subprocess เลย — นี่คือความต่างสำคัญระหว่าง "แจก logic เป็น library" กับ "แจก logic เป็นแค่ executable"

### 17.4 หลาย Binary ในแพ็กเกจเดียว: src/bin/

บางครั้งโปรเจกต์หนึ่งมี logic หลักชุดเดียว แต่ต้องการโปรแกรม CLI หลายตัวที่ใช้ logic นั้นในบริบทต่างกัน — เช่น
ระบบจัดการไฟล์ log ที่มี logic การ parse log format กลางไว้ที่ `lib.rs` แล้วมีทั้ง `analyze` (โปรแกรมสรุปสถิติ)
และ `tail_watch` (โปรแกรม watch ไฟล์แบบ real-time) เป็นสอง entry point ที่แยกกัน

Cargo จัดการเรื่องนี้ให้ง่ายมาก: **ไฟล์ `.rs` ทุกไฟล์ที่วางอยู่ตรง ๆ ใน `src/bin/` จะถูกมองเป็น binary crate
แยกกันโดยอัตโนมัติ** ไม่ต้องประกาศอะไรเพิ่มใน `Cargo.toml` เลย (ต่างจาก `[[bin]]` แบบระบุ path เอง ซึ่งจะพูดถึง
ด้านล่าง)

มาดูโครงสร้างไฟล์:

```
log_tools/
├── Cargo.toml
└── src/
    ├── lib.rs              <- logic กลาง: parse log format
    └── bin/
        ├── analyze.rs      <- binary crate ชื่อ "analyze"
        └── tail_watch.rs   <- binary crate ชื่อ "tail_watch"
```

```toml
[package]
name = "log_tools"
version = "0.1.0"
edition = "2021"

[dependencies]
```

```rust
// src/lib.rs
/// หนึ่งบรรทัด log ที่ parse แล้ว
#[derive(Debug)]
pub struct LogEntry {
    pub level: String,
    pub message: String,
}

/// parse บรรทัด log แบบง่าย ๆ รูปแบบ "LEVEL: message" เช่น "ERROR: disk full"
/// คืนค่า None ถ้าบรรทัดไม่ตรงรูปแบบ (ไม่มี ": " คั่นอยู่)
pub fn parse_line(line: &str) -> Option<LogEntry> {
    let (level, message) = line.split_once(": ")?;
    Some(LogEntry {
        level: level.to_string(),
        message: message.to_string(),
    })
}
```

```rust
// src/bin/analyze.rs — โปรแกรมที่ 1: อ่าน log ทั้งหมด แล้วสรุปจำนวนแต่ละ level
use log_tools::parse_line;

fn main() {
    let sample_logs = [
        "INFO: server started",
        "ERROR: disk full",
        "ERROR: connection timeout",
        "INFO: request handled",
    ];

    let mut error_count = 0;
    let mut info_count = 0;

    for line in sample_logs {
        if let Some(entry) = parse_line(line) {
            match entry.level.as_str() {
                "ERROR" => error_count += 1,
                "INFO" => info_count += 1,
                _ => {}
            }
        }
    }

    println!("สรุป log: INFO = {info_count}, ERROR = {error_count}");
}
```

```rust
// src/bin/tail_watch.rs — โปรแกรมที่ 2: จำลองการดู log ทีละบรรทัดแบบ real-time
use log_tools::parse_line;

fn main() {
    let sample_logs = ["INFO: server started", "ERROR: disk full"];

    for line in sample_logs {
        if let Some(entry) = parse_line(line) {
            println!("[{}] {}", entry.level, entry.message);
        }
    }
}
```

รันแต่ละโปรแกรมแยกกันด้วย `--bin` ตามด้วยชื่อไฟล์ (ไม่ต้องมี `.rs`):

```bash
cargo run --bin analyze
```

```
   Compiling log_tools v0.1.0 (/home/user/log_tools)
    Finished dev [unoptimized + debuginfo] target(s) in 0.31s
     Running `target/debug/analyze`
สรุป log: INFO = 2, ERROR = 2
```

```bash
cargo run --bin tail_watch
```

```
     Running `target/debug/tail_watch`
[INFO] server started
[ERROR] disk full
```

ถ้าลองรัน `cargo build` เฉย ๆ (ไม่ระบุ `--bin`) Cargo จะ compile **ทุก binary crate ที่เจอทั้งหมด** ให้ในครั้ง
เดียว (ทั้ง `analyze` และ `tail_watch`) แต่ถ้ารัน `cargo run` โดยไม่ระบุ `--bin` และแพ็กเกจมี binary มากกว่าหนึ่ง
ตัว จะเจอ error ทันทีเพราะ Cargo ไม่รู้ว่าคุณต้องการรันตัวไหน:

```
error: `cargo run` could not determine which binary to run. Use the `--bin` option to specify a binary,
or the `default-run` manifest key.
available binaries: analyze, tail_watch
```

**วิธีแก้ตามที่ error message บอก**: ระบุ `--bin` เสมอเมื่อมีหลาย binary หรือถ้ามีตัวหนึ่งที่อยากให้เป็นค่า
default เวลารัน `cargo run` เปล่า ๆ สามารถตั้งค่าผ่านคีย์ `default-run` ใน `[package]` ได้:

```toml
[package]
name = "log_tools"
version = "0.1.0"
edition = "2021"
default-run = "analyze"
```

#### `[[bin]]`: ระบุ binary เองแบบไม่ต้องอยู่ใน src/bin/

ปกติวาง `.rs` ไว้ใน `src/bin/` ก็เพียงพอสำหรับ 99% ของกรณีใช้งาน แต่ถ้าอยากตั้งชื่อ binary ให้ต่างจากชื่อไฟล์
หรืออยากเก็บไฟล์ entry point ไว้ที่ path อื่นที่ไม่ใช่ `src/bin/` สามารถประกาศ table `[[bin]]` (สังเกตวงเล็บสอง
ชั้น — เป็น TOML array-of-tables ความหมายคือ "ประกาศ binary เพิ่มอีกหนึ่งตัว" ประกาศซ้ำได้หลายรอบสำหรับหลาย
binary) ตรง ๆ ใน `Cargo.toml`:

```toml
[[bin]]
name = "analyze"
path = "src/tools/analyze_main.rs"

[[bin]]
name = "tail_watch"
path = "src/tools/tail_watch_main.rs"
```

เมื่อคุณประกาศ `[[bin]]` ด้วยมือแบบนี้ (path อยู่นอก `src/bin/`) Cargo จะไม่ auto-detect ไฟล์ใน `src/bin/`
ให้อีกต่อไปสำหรับกรณีที่ทับซ้อนกัน — แต่ในทางปฏิบัติ การปล่อยให้ Cargo auto-detect จาก `src/bin/*.rs`
(ไม่ต้องเขียน `[[bin]]` เลย) คือวิธีที่ง่ายและอ่านง่ายที่สุด แนะนำให้ใช้เป็นทางเลือกแรกเสมอ ใช้ `[[bin]]` เฉพาะ
กรณีที่มีเหตุผลเจาะจงจริง ๆ (เช่นต้องรวมกลุ่มไฟล์ tool ไว้ในโฟลเดอร์ตามโดเมนแทนโฟลเดอร์ตามชนิดไฟล์)

### 17.5 เมื่อไรที่ควรแตกเป็นหลายแพ็กเกจ: จุดกำเนิดของ Workspace

Pattern ใน 17.3-17.4 (lib + หลาย binary) ยังอยู่ใน "package เดียว" — เหมาะเมื่อทุกโปรแกรมยังพอจะแชร์
`Cargo.toml` เดียวกันได้ (dependency ชุดเดียวกัน, version เดียวกัน, publish เป็นก้อนเดียวกันได้) แต่เมื่อ
โปรเจกต์เติบโตขึ้น จะเริ่มมีสถานการณ์ที่ "การอยู่ package เดียวกัน" กลายเป็นข้อจำกัดมากกว่าข้อดี ตัวอย่าง
สถานการณ์จริงที่พบบ่อย:

- คุณมี **core business logic** ที่อยากแชร์ระหว่างโปรแกรม CLI กับเว็บ server แต่เว็บ server ต้องพึ่ง dependency
  หนัก ๆ อย่าง `axum`/`tokio` ในขณะที่ CLI ไม่ควรต้องแบก dependency พวกนี้เลยเพื่อให้ compile เร็วและ binary
  เล็ก — ถ้าทุกอย่างอยู่ package เดียว `[dependencies]` จะรวมกันหมด แม้ตัว CLI ไม่ได้ใช้ `axum` เลยก็ต้อง
  compile ผ่านมันอยู่ดี
- คุณอยาก publish แค่บางส่วนของโค้ดขึ้น crates.io ให้คนอื่นใช้ (เช่น core logic ที่เป็น library ทั่วไป) แต่ไม่
  อยาก publish โปรแกรม CLI ภายในที่ใช้เฉพาะทีมตัวเอง — package เดียวไม่แยกสิทธิ์การ publish แบบนี้ได้ เพราะ
  publish ทีคือ publish "ทั้ง package"
- ทีมมีคนหลายกลุ่มดูแลส่วนต่างกัน (ทีม backend ดูแล core, ทีม frontend/CLI ดูแล UI) การแยกเป็นหลาย package
  ทำให้กำหนด ownership และ `version` ของแต่ละส่วนแยกกันได้ชัดเจน (core อาจอยู่ที่ v2.3.0 ในขณะที่ CLI อยู่ที่
  v0.9.0 — versioning ไม่ต้องผูกกัน)
- อยากให้ compile time เร็วขึ้นด้วยการ parallelize — ถ้าแยก crate ชัดเจน `rustc`/`cargo` สามารถ compile
  หลาย crate ที่ไม่ได้ depend on กันแบบ parallel ได้ (แม้จะยังอยู่ใน dependency graph เดียวกัน)

Cargo แก้ปัญหานี้ด้วยฟีเจอร์ที่เรียกว่า **workspace** — วิธีจัดกลุ่ม **หลาย package** ให้ใช้ `Cargo.lock` และ
โฟลเดอร์ build (`target/`) ร่วมกันได้ โดยที่แต่ละ package ยังคงมี `Cargo.toml`, `name`, `version`, และ
`[dependencies]` เป็นของตัวเองอย่างสมบูรณ์ Part 2 หัวข้อ 2.10 ได้เกริ่นภาพรวมนี้ไว้แล้วสั้น ๆ — ต่อจากนี้เราจะ
ลงรายละเอียดทั้งหมดของมัน

### 17.6 โครงสร้าง Workspace แบบสมบูรณ์: core + cli + web

ให้พิจารณาโปรเจกต์ตัวอย่างที่สมจริงที่สุด: ระบบจัดการงาน (task manager) ที่มี business logic กลางเรื่อง
"งาน/task" ใช้ร่วมกันระหว่างโปรแกรม CLI กับเว็บ server — สถานการณ์ตรงกับที่หัวข้อ 17.5 บอกไว้ทุกประการ

โครงสร้าง workspace เต็มรูปแบบ:

```
task_workspace/
├── Cargo.toml              <- ROOT manifest: มี [workspace] อย่างเดียว ไม่มี [package]
├── Cargo.lock              <- ล็อกเวอร์ชันของ "ทุก" member รวมกันเป็นไฟล์เดียว
├── target/                  <- โฟลเดอร์ build ผลลัพธ์ที่ทุก member ใช้ร่วมกัน
├── core/
│   ├── Cargo.toml           <- [package] name = "task_core"
│   └── src/
│       └── lib.rs
├── cli/
│   ├── Cargo.toml           <- [package] name = "task_cli", depends on task_core
│   └── src/
│       └── main.rs
└── web/
    ├── Cargo.toml           <- [package] name = "task_web", depends on task_core
    └── src/
        └── main.rs
```

จุดที่ต้องสังเกตให้ชัดคือ **root `Cargo.toml` ของ workspace ไม่มี section `[package]` เลย** (ในกรณีที่ root
เองไม่ได้เป็น package ด้วย — จะพูดถึงกรณี "root เป็น member ด้วย" แยกในหัวข้อถัดไป) มันมีแค่ section
`[workspace]` เดียว ทำหน้าที่เป็น "ตัวประสาน" ให้กับทุก member เท่านั้น:

```toml
# task_workspace/Cargo.toml (root)
[workspace]
resolver = "2"
members = [
    "core",
    "cli",
    "web",
]
```

**อธิบายแต่ละคีย์**:

- **`members`** — array ของ path (สัมพัทธ์กับตำแหน่งไฟล์ root `Cargo.toml`) ไปยังแต่ละ package ที่เป็นสมาชิก
  ของ workspace นี้ แต่ละ path ต้องมีไฟล์ `Cargo.toml` ของตัวเองอยู่ (เป็น package จริง ๆ ตามนิยามหัวข้อ 17.1)
- **`resolver = "2"`** — กำหนดว่าจะใช้ dependency resolver แบบไหนตอน Cargo คำนวณเวอร์ชันที่ต้องใช้ (feature
  unification ระหว่าง member ต่างกัน) resolver รุ่น 2 (เปิดตัวคู่กับ Rust edition 2021) แก้ปัญหาสำคัญของ
  resolver รุ่น 1 คือ: รุ่น 1 จะ "รวม feature flag ของทุก target เข้าด้วยกันแบบไม่แยกแยะ" เช่นถ้า dependency
  ตัวหนึ่งถูกใช้ทั้งใน `[dependencies]` ปกติและใน `[dev-dependencies]`/`[build-dependencies]` ด้วย feature ต่างกัน
  รุ่น 1 จะเปิดทุก feature รวมกันให้กับทุก target แม้จะไม่จำเป็น ทำให้ production binary อาจพ่วง feature ที่ควร
  มีแค่ตอน test เท่านั้นมาด้วย ส่วน resolver รุ่น 2 จะฉลาดพอที่จะแยก feature ของแต่ละ target (`[dependencies]` vs
  `[dev-dependencies]` vs `[build-dependencies]`, และแยก host platform กับ target platform ตอน cross-compile)
  ออกจากกัน ให้ผลลัพธ์ที่ตรงกับที่คนเขียน `Cargo.toml` ตั้งใจมากกว่า **สำหรับโปรเจกต์ใหม่ทุกโปรเจกต์ ควรใส่
  `resolver = "2"` เสมอ** (ถ้า `edition` ของ package เป็น 2021 ขึ้นไปแล้ว Cargo เวอร์ชันใหม่จะเลือก resolver 2
  ให้เป็นค่า default โดยอัตโนมัติอยู่แล้วสำหรับ package เดี่ยว แต่ **workspace ต้องระบุ `resolver` ที่ root เอง
  อย่างชัดเจน** เพราะค่า default ของ workspace root จะพิจารณาจาก edition ของ "root package" ถ้า root ไม่มี
  `[package]` เลยแบบตัวอย่างนี้ ค่า default จะย้อนกลับไปเป็น resolver รุ่น 1 เพื่อความ backward compatible — จึง
  ต้องเขียน `resolver = "2"` เองเสมอเพื่อความชัดเจน ไม่ปล่อยให้เดา)

ทีนี้มาดูไฟล์ของแต่ละ member ทีละตัว:

```toml
# task_workspace/core/Cargo.toml
[package]
name = "task_core"
version = "0.1.0"
edition = "2021"
description = "Core domain logic สำหรับระบบจัดการงาน (task manager)"

[dependencies]
```

```rust
// task_workspace/core/src/lib.rs
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum TaskStatus {
    Todo,
    InProgress,
    Done,
}

#[derive(Debug, Clone)]
pub struct Task {
    pub id: u32,
    pub title: String,
    pub status: TaskStatus,
}

#[derive(Debug, Default)]
pub struct TaskManager {
    tasks: Vec<Task>,
    next_id: u32,
}

impl TaskManager {
    pub fn new() -> Self {
        TaskManager { tasks: Vec::new(), next_id: 1 }
    }

    pub fn add_task(&mut self, title: impl Into<String>) -> u32 {
        let id = self.next_id;
        self.tasks.push(Task { id, title: title.into(), status: TaskStatus::Todo });
        self.next_id += 1;
        id
    }

    pub fn mark_done(&mut self, id: u32) -> bool {
        if let Some(task) = self.tasks.iter_mut().find(|t| t.id == id) {
            task.status = TaskStatus::Done;
            true
        } else {
            false
        }
    }

    pub fn all_tasks(&self) -> &[Task] {
        &self.tasks
    }
}
```

```toml
# task_workspace/cli/Cargo.toml
[package]
name = "task_cli"
version = "0.1.0"
edition = "2021"

[dependencies]
task_core = { path = "../core" }
```

```rust
// task_workspace/cli/src/main.rs
use task_core::TaskManager;

fn main() {
    let mut manager = TaskManager::new();
    let id1 = manager.add_task("เขียนเอกสาร Part 17");
    manager.add_task("รีวิว pull request");
    manager.mark_done(id1);

    for task in manager.all_tasks() {
        println!("[{:?}] #{} {}", task.status, task.id, task.title);
    }
}
```

```toml
# task_workspace/web/Cargo.toml
[package]
name = "task_web"
version = "0.1.0"
edition = "2021"

[dependencies]
task_core = { path = "../core" }
```

```rust
// task_workspace/web/src/main.rs
// (ตัวอย่างนี้จำลองว่ามี "web server" เรียก core logic — ในหลักสูตรจริงเรื่อง HTTP server
// ด้วย axum จะเรียนเจาะลึกใน Part 62 เป็นต้นไป ตอนนี้เขียนแบบง่าย ไม่มี dependency เว็บจริง
// เพื่อโฟกัสที่แนวคิด workspace ล้วน ๆ)
use task_core::TaskManager;

fn handle_request(manager: &TaskManager) -> String {
    let lines: Vec<String> = manager
        .all_tasks()
        .iter()
        .map(|t| format!("{}: {}", t.id, t.title))
        .collect();
    lines.join("\n")
}

fn main() {
    let mut manager = TaskManager::new();
    manager.add_task("Deploy service เวอร์ชันใหม่");

    let response_body = handle_request(&manager);
    println!("จำลอง HTTP response body:\n{response_body}");
}
```

สังเกตจุดสำคัญ: ทั้ง `task_cli` และ `task_web` เขียน dependency เดียวกันเป๊ะ:

```toml
task_core = { path = "../core" }
```

นี่คือ **path dependency** ซึ่งจะอธิบายเจาะลึกในหัวข้อถัดไป

### 17.7 Path Dependency: การอ้างอิงกันเองภายใน Workspace

Path dependency คือรูปแบบการเขียน `[dependencies]` ที่ชี้ไปยัง **โฟลเดอร์บนดิสก์** โดยตรง แทนที่จะชี้ไปยัง
crates.io (ซึ่งเป็นค่า default เมื่อเขียนแค่ `rand = "0.8"` แบบที่เรียนใน Part 2):

```toml
[dependencies]
task_core = { path = "../core" }
```

ไวยากรณ์นี้คือ **expanded table syntax** (inline table `{ ... }`) ที่จะอธิบายเต็มรูปแบบในหัวข้อ 17.11 — ตอนนี้
โฟกัสที่คีย์ `path` ก่อน: path เป็น relative path นับจากตำแหน่งของไฟล์ `Cargo.toml` ที่เขียนบรรทัดนี้ (ไม่ใช่
นับจาก root workspace) เช่นในตัวอย่าง `task_cli/Cargo.toml` เขียน `path = "../core"` เพราะ `core/` อยู่ในระดับ
เดียวกับ `cli/` (เป็น sibling folder กัน ทั้งคู่อยู่ใต้ root `task_workspace/`)

#### ทำไมต้องมี path dependency: แก้ปัญหา "chicken-and-egg" ของการ publish

ถ้าไม่มี path dependency วิธีเดียวที่จะให้ `task_cli` ใช้โค้ดจาก `task_core` ได้คือต้อง **publish `task_core`
ขึ้น crates.io ก่อน** แล้วค่อยเขียน `task_core = "0.1"` แบบปกติ — ปัญหาคือระหว่างพัฒนา คุณอาจแก้ `task_core`
วันละหลายสิบครั้ง ถ้าต้อง publish ขึ้น crates.io ทุกครั้งที่แก้ (ซึ่ง publish แต่ละเวอร์ชันแก้ไม่ได้ ต้องขึ้น
version ใหม่เสมอ — จะพูดถึงในหัวข้อ 17.10) งานพัฒนาจะช้าลงมหาศาลและสร้าง "เวอร์ชันขยะ" นับพันเวอร์ชันบน
crates.io ที่ไม่มีใครใช้จริง

Path dependency แก้ปัญหานี้โดยบอก Cargo ว่า **"ไม่ต้องไปหาที่ crates.io เลย ใช้โค้ดจากโฟลเดอร์นี้ตรง ๆ"** ทำให้:

- ทุกครั้งที่แก้ `core/src/lib.rs` แล้วรัน `cargo build` ที่ `cli/` หรือที่ root workspace, `task_cli` จะเห็นการ
  เปลี่ยนแปลงนั้นทันที **ไม่ต้อง publish อะไรเลย** ไม่ต้อง bump version ไม่ต้องรอ network round-trip ไปหา
  crates.io
- เหมาะสมบูรณ์แบบกับช่วงพัฒนา (development) ที่โค้ดยังเปลี่ยนบ่อย ก่อนที่ core logic จะ "อิ่มตัว" พร้อม publish
  จริง

#### path dependency ไม่จำเป็นต้องอยู่ใน workspace เดียวกันเสมอไป

ข้อควรรู้เพิ่ม: path dependency ใช้ได้แม้ทั้งสอง package ไม่ได้อยู่ใน workspace เดียวกันเลย (แค่ path ชี้ถูกที่
ก็พอ) แต่ **การรวมกันเป็น workspace ให้ประโยชน์เพิ่มเติม** ที่ path dependency เดี่ยว ๆ ไม่มี — คือการแชร์
`Cargo.lock` และ `target/` ร่วมกัน (หัวข้อ 17.8) ซึ่งเป็นเหตุผลว่าทำไมในทางปฏิบัติ package ที่อ้างอิงกันด้วย
path dependency บ่อยครั้งจึงถูกจัดให้อยู่ใน workspace เดียวกันเสมอ ทั้งสองแนวคิด (path dependency และ workspace)
เสริมกันแต่เป็นฟีเจอร์คนละตัวที่แยกจากกันได้ทางเทคนิค

#### สิ่งที่เกิดขึ้นเมื่อจะ publish จริง: ต้องเปลี่ยนเป็น version dependency

จุดที่ต้องระวังมาก: **ถ้าวันหนึ่งอยาก publish `task_cli` ขึ้น crates.io จริง จะต้องเปลี่ยน path dependency เป็น
version dependency ก่อน** เพราะ crates.io ไม่ยอมให้ package ที่ publish มี dependency ชี้ไปยัง path บนดิสก์ของ
คนอื่น (path ของคุณไม่มีความหมายอะไรกับเครื่องคนอื่นที่ดาวน์โหลด package ไป) วิธีที่ถูกต้องคือเขียนทั้ง `path`
และ `version` คู่กัน:

```toml
[dependencies]
task_core = { path = "../core", version = "0.1" }
```

เมื่อเขียนแบบนี้ **ระหว่างพัฒนาในเครื่องเดียวกัน/workspace เดียวกัน Cargo จะยังใช้ path เสมอ** (path มี priority
สูงกว่า) แต่ **เมื่อ publish จริง Cargo จะเปลี่ยนไปเขียนแค่ `version = "0.1"` ลงใน package ที่ publish ให้
อัตโนมัติ** (ตัด `path` ออกเพราะไม่มีความหมายกับผู้ใช้ปลายทาง) — นี่คือ pattern ที่ library ทั่วไปในระบบนิเวศ
Rust ที่ประกอบด้วยหลาย crate (เช่น `tokio` ที่แตกเป็น `tokio`, `tokio-util`, `tokio-macros` หลาย crate ที่
depend on กันเอง) ใช้กันเป็นมาตรฐาน

### 17.8 Cargo.lock และ target/ ที่แชร์กันทั้ง Workspace

นี่คือประโยชน์ที่จับต้องได้ที่สุดของ workspace เมื่อเทียบกับการแยก package ให้อยู่คนละโฟลเดอร์แบบไม่มี
`[workspace]` ผูกกัน

#### หนึ่ง Cargo.lock ต่อทั้ง workspace

สังเกตในโครงสร้างไฟล์หัวข้อ 17.6 ว่า **`Cargo.lock` มีอยู่ที่ root ของ workspace เพียงไฟล์เดียว** — สมาชิกแต่ละ
ตัว (`core/`, `cli/`, `web/`) **ไม่มี** `Cargo.lock` ของตัวเองเลย แม้แต่ละ member จะมี `[dependencies]` ของ
ตัวเองที่อาจไม่เหมือนกัน (เช่นสมมติ `task_web` เพิ่ม `serde = "1"` เข้ามาเพื่อ serialize response แต่ `task_cli`
ไม่ได้ใช้ `serde` เลย) Cargo จะ resolve dependency ของ**ทุก member รวมกัน**เป็น dependency graph เดียว แล้ว
บันทึกผลลัพธ์ลง `Cargo.lock` ไฟล์เดียวที่ root

**ประโยชน์เชิงปฏิบัติที่สำคัญที่สุด**: สมมติทั้ง `task_core` และ `task_web` ต่างใช้ crate `serde` (แม้จะระบุ
version requirement ต่างกันเล็กน้อย เช่น `task_core` เขียน `serde = "1.0"` และ `task_web` เขียน `serde =
"1.0.100"`) การมี `Cargo.lock` ร่วมกันจะบังคับให้ Cargo resolve `serde` **เวอร์ชันเดียวที่ตรงกับทุกข้อจำกัดพร้อม
กัน** (เช่นได้ `1.0.210` ที่ทั้งสอง constraint ยอมรับ) แทนที่จะมี `serde` สองเวอร์ชันแยกกันซ้อนอยู่ใน dependency
tree (ซึ่งเป็นไปได้ในบางสถานการณ์ถ้า version ไม่ compatible กันเลย แต่ทำให้ binary ใหญ่ขึ้นและอาจเกิดปัญหา type
ไม่ตรงกันข้าม crate — เช่นถ้า `task_core` return `serde_json::Value` เวอร์ชันหนึ่ง แต่ `task_web` คาดหวัง
`serde_json::Value` อีกเวอร์ชัน จะเป็นคนละ type กันทั้งที่ชื่อเหมือนกัน compile ไม่ผ่าน) การแชร์ `Cargo.lock`
เดียวกันจึงช่วย**หลีกเลี่ยงปัญหาเวอร์ชันชนกัน (version conflict)** ได้อย่างเป็นระบบตั้งแต่ต้น

#### หนึ่ง target/ ต่อทั้ง workspace

เช่นเดียวกัน โฟลเดอร์ `target/` ที่เก็บผลลัพธ์การ compile ก็มีอยู่ที่ root ของ workspace เพียงโฟลเดอร์เดียว
ไม่ใช่แต่ละ member มี `target/` ของตัวเอง — นี่คือ **shared build cache** ประโยชน์คือ:

- ถ้า `task_cli` และ `task_web` ทั้งคู่ depend on `task_core` (และ `serde` เวอร์ชันเดียวกันตามที่ resolve ได้
  จากข้อด้านบน) Cargo จะ **compile `task_core` และ `serde` แค่ครั้งเดียว** แล้วให้ทั้งสอง binary มา link ใช้
  ผลลัพธ์ที่ compile ไว้แล้วร่วมกัน — ถ้าแยกกันเป็นสอง package คนละโฟลเดอร์ที่ไม่ได้อยู่ workspace เดียวกัน ทั้ง
  สองจะมี `target/` ของตัวเอง และต้อง compile `task_core`/`serde` ซ้ำสองรอบ (คนละ build cache กันโดยสิ้นเชิง)
  เสียเวลา compile และพื้นที่ดิสก์ซ้ำซ้อนโดยไม่จำเป็น
- รัน `cargo build` จาก root ครั้งเดียวจะได้ artifact ของทุก member วางอยู่ใต้ `target/debug/` เดียวกันหมด (เช่น
  `target/debug/task_cli`, `target/debug/task_web`)

โครงสร้าง `target/` ของ workspace หลัง build ครบทุก member หน้าตาประมาณนี้ (ต่อยอดจากที่ Part 2 หัวข้อ 2.5
อธิบายไว้แล้วสำหรับ package เดี่ยว):

```
task_workspace/target/debug/
├── task_cli              <- executable ของ member "cli"
├── task_web               <- executable ของ member "web"
├── libtask_core.rlib      <- compiled library ของ member "core" (รูปแบบ intermediate ที่ rustc ใช้ link)
└── deps/                   <- compiled dependency ที่ทุก member ใช้ร่วมกัน (serde, ฯลฯ)
```

#### ข้อควรระวังตรงกันข้าม: การเปลี่ยน dependency ของ member หนึ่งอาจกระทบ build time ของ member อื่น

เพราะทุก member ใช้ `Cargo.lock` ร่วมกัน การเพิ่ม/แก้ dependency ใน member ใดก็ตาม อาจทำให้ต้อง resolve
dependency graph ใหม่ทั้งชุด (แม้ member อื่นจะไม่ได้แก้โค้ดเลย) ซึ่งในบางกรณีอาจทำให้เวอร์ชันของ dependency
ที่ member อื่นใช้ร่วมด้วยขยับตามไปด้วย (เช่นถ้า `task_web` เพิ่ม `serde = "1.0.200"` ที่บังคับให้ต้องอัปเกรด
`serde` เป็นเวอร์ชันที่ใหม่กว่าที่ `task_core` เคย pin ไว้ก่อนหน้า) นี่ไม่ใช่ปัญหาร้ายแรง (เพราะตามหลัก SemVer
ที่ Part 2 อธิบาย minor/patch update ไม่ควร breaking) แต่เป็นจุดที่ทีมงานขนาดใหญ่ที่ใช้ workspace ร่วมกันหลาย
สิบ member ต้องรู้ตัว — การรัน `cargo build`/`cargo test` ที่ root workspace เป็นประจำ (ไม่ใช่แค่ build member
ของตัวเองแบบแยกโดด ๆ) จะช่วยจับปัญหานี้ได้เร็วก่อนที่จะกลายเป็นปัญหาใหญ่ตอน merge code

### 17.9 คำสั่ง Cargo ในบริบท Workspace: -p และ --workspace

เมื่อทำงานอยู่ใน workspace คำสั่ง cargo พื้นฐานที่เรียนใน Part 2 (`cargo build`, `cargo run`, `cargo test`,
`cargo check`) ยังใช้ได้เหมือนเดิมทุกคำสั่ง แต่ความหมายของ **"scope"** (มันจะทำงานกับ member ไหน) เปลี่ยนไป
ตามตำแหน่งที่คุณรันคำสั่งและ flag ที่ใส่เพิ่ม

#### รันจาก root ของ workspace

```bash
cd task_workspace
cargo build
```

```
   Compiling task_core v0.1.0 (/home/user/task_workspace/core)
   Compiling task_cli v0.1.0 (/home/user/task_workspace/cli)
   Compiling task_web v0.1.0 (/home/user/task_workspace/web)
    Finished dev [unoptimized + debuginfo] target(s) in 0.94s
```

เมื่อรัน `cargo build` (หรือ `cargo check`, `cargo test`) จาก **root ของ workspace โดยไม่ระบุ flag เพิ่ม** —
พฤติกรรม default คือ **build ทุก member ที่เป็น "default member"** ซึ่งโดย default แล้วคือทุก member ใน
`members` list (ยกเว้นถ้ามีการตั้งค่า `default-members` แยกไว้เฉพาะบางตัว ซึ่งเป็นฟีเจอร์ขั้นสูงที่ไม่ได้ใช้บ่อย
ในระดับพื้นฐาน)

#### `-p <package>`: เจาะจง member เดียว

ถ้าอยาก build/run/test แค่ member เดียว ใช้ flag `-p` (ย่อของ `--package`) ตามด้วยชื่อ package (ชื่อใน
`[package]` ของ member นั้น ไม่ใช่ชื่อโฟลเดอร์ — ปกติสองอย่างนี้เหมือนกันแต่ไม่จำเป็นต้องเหมือนกันเสมอไป):

```bash
cargo build -p task_cli
```

```
   Compiling task_core v0.1.0 (/home/user/task_workspace/core)
   Compiling task_cli v0.1.0 (/home/user/task_workspace/cli)
    Finished dev [unoptimized + debuginfo] target(s) in 0.51s
```

สังเกตว่า `task_web` ไม่ถูก compile เลย เพราะเราสั่งเจาะจงแค่ `task_cli` — แต่ `task_core` ยังถูก compile ด้วย
เพราะเป็น dependency ของ `task_cli` (Cargo compile ทุก dependency ที่จำเป็นเสมอ ไม่ว่าจะสั่ง `-p` เจาะจงแค่ตัว
เดียวหรือไม่)

`cargo run -p <package>` ก็ใช้หลักเดียวกัน — สำคัญมากเมื่อ workspace มี binary หลายตัวในหลาย member (ไม่ใช่แค่
หลาย binary ใน member เดียวแบบหัวข้อ 17.4):

```bash
cargo run -p task_web
```

```
    Finished dev [unoptimized + debuginfo] target(s) in 0.02s
     Running `target/debug/task_web`
จำลอง HTTP response body:
1: Deploy service เวอร์ชันใหม่
```

`cargo test -p task_core` จะรันแค่ test ของ `task_core` เท่านั้น ไม่รัน test ของ `task_cli`/`task_web` เลย —
มีประโยชน์มากตอนแก้บั๊กที่รู้แน่ชัดว่ากระทบแค่ member เดียว ไม่ต้องรอ test ของทุก member ทั้ง workspace

#### `--workspace`: สั่งชัดเจนว่าเอาทุก member (เทียบเท่ากับ `--all` ในเวอร์ชันเก่า)

```bash
cargo test --workspace
```

```
     Running unittests src/lib.rs (target/debug/deps/task_core-...)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running unittests src/main.rs (target/debug/deps/task_cli-...)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running unittests src/main.rs (target/debug/deps/task_web-...)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

แม้ `--workspace` จะให้ผลเหมือนกับการรันแบบไม่ใส่ flag อะไรเลยจาก root ในกรณีปกติ (เพราะ default คือทุก member
อยู่แล้ว) แต่การเขียน `--workspace` ชัดเจนมีประโยชน์สองอย่าง: (1) **สื่อความตั้งใจให้คนอ่านคำสั่งเข้าใจง่ายขึ้น**
ว่าตั้งใจรันทุกตัว ไม่ใช่ลืมใส่ `-p` (2) **ยังทำงานถูกต้องแม้ในอนาคตมีคนมาตั้งค่า `default-members` ให้ไม่ครอบ
คลุมทุกตัว** — `--workspace` จะบังคับ "ทุก member เสมอ" ไม่ว่า `default-members` จะตั้งไว้อย่างไร ในขณะที่การไม่
ใส่ flag อะไรเลยจะเคารพ `default-members` ถ้ามันถูกตั้งไว้

**กฎการใช้งานจริงที่แนะนำ**: ระหว่างพัฒนา member ใดตัวหนึ่งอยู่ ใช้ `-p <package>` เพื่อความเร็ว (ไม่ต้องรอ
compile/test member อื่นที่ไม่เกี่ยวข้อง) ส่วนก่อน commit/push หรือใน CI pipeline ให้ใช้ `cargo build
--workspace` และ `cargo test --workspace` เสมอ เพื่อยืนยันว่าการเปลี่ยนแปลงของคุณไม่ได้ทำให้ member อื่นพังโดย
ไม่ตั้งใจ (เช่นแก้ signature ของฟังก์ชันใน `task_core` แล้วลืมว่า `task_web` เรียกใช้ฟังก์ชันนั้นอยู่ด้วย)

### 17.10 การ Publish Package ใน Workspace ขึ้น crates.io: ภาพรวมที่ต้องรู้

บทนี้จะไม่ลงรายละเอียดเต็มรูปแบบของขั้นตอน publish (เช่น `cargo login`, `cargo publish`, การตั้งค่า API token)
เพราะเป็นเนื้อหาที่เหมาะกับบทที่ลึกกว่านี้ — แต่มีความเข้าใจพื้นฐานสามข้อที่ต้องรู้ตั้งแต่ตอนนี้เพื่อไม่ให้ออกแบบ
workspace ผิดทาง:

**1. Publish เป็นหน่วย "package" ไม่ใช่หน่วย "workspace"** — `cargo publish` รันจากโฟลเดอร์ของ package ที่
ต้องการ publish (หรือใช้ `-p <package>` จาก root workspace) และ publish package นั้น**เดียว**ขึ้นไป แต่ละ
member ที่อยากให้คนอื่น `cargo add` ได้ต้อง publish แยกกันทีละตัว ไม่มี "publish ทั้ง workspace ในคำสั่งเดียว"

**2. Member ที่จะ publish ต้องมี metadata ที่ครบสำหรับ crates.io** — นอกจาก `name` และ `version` ที่มีอยู่แล้ว
(บังคับเสมอ) crates.io ยังบังคับให้มีคีย์อื่นเพิ่มก่อนจะ publish ผ่านได้จริง โดยเฉพาะ **`license`** (หรือ
`license-file`) และแนะนำอย่างยิ่งให้มี **`description`** ด้วย (ไม่บังคับทางเทคนิคแต่ `cargo publish` จะเตือนถ้า
ไม่มี และหน้า crates.io ของ crate ที่ไม่มี description จะดูไม่น่าเชื่อถือ):

```toml
[package]
name = "task_core"
version = "0.1.0"
edition = "2021"
description = "Core domain logic สำหรับระบบจัดการงาน (task manager)"
license = "MIT"
```

**3. แต่ละ member เลือก publish หรือไม่ publish ได้อย่างอิสระ** — สถานการณ์จริงที่พบบ่อยมากคือ `task_core`
(library ที่มีประโยชน์ทั่วไป ไม่ผูกกับธุรกิจเฉพาะ) อาจถูก publish ขึ้น crates.io ให้คนอื่นใช้ได้ ในขณะที่
`task_cli`/`task_web` (โปรแกรมภายในที่ผูกกับ business เฉพาะขององค์กร ไม่มีประโยชน์กับคนนอก) **ไม่ต้อง publish
เลย** — ใช้แค่ path dependency ภายใน workspace ต่อไปตลอดไปก็ได้ ไม่มีข้อบังคับว่าทุก member ต้อง publish ให้
ครบ ถ้า member ไหนไม่ต้องการ publish สามารถเพิ่มคีย์ `publish = false` เพื่อป้องกันการ publish โดยไม่ตั้งใจ
(เผลอรัน `cargo publish` ผิด package):

```toml
[package]
name = "task_cli"
version = "0.1.0"
edition = "2021"
publish = false
```

ถ้าพยายาม `cargo publish` package ที่มี `publish = false` จะได้ error ทันที ป้องกันความผิดพลาดที่แก้คืนยากมาก
(เพราะ crates.io **ไม่อนุญาตให้ลบ version ที่ publish ไปแล้ว** ทำได้แค่ "yank" คือประกาศว่าไม่ให้ project ใหม่
เลือก version นั้น แต่ project ที่ pin version นั้นไว้แล้วจะยังใช้งานได้ต่อ — การ publish ผิดพลาดจึงเป็นความ
ผิดพลาดที่ "ลบล้างไม่ได้สมบูรณ์" ต้องระวังให้ดีตั้งแต่ต้น)

### 17.11 Dependency แบบละเอียด: Expanded Table Syntax เทียบกับ Inline

Part 2 หัวข้อ 2.7-2.8 แนะนำการเขียน dependency แบบสั้นที่สุด (`rand = "0.8"`) ซึ่งเพียงพอเมื่อต้องการแค่ระบุ
version requirement เฉย ๆ แต่ในหลายกรณี (path dependency ที่เห็นแล้วในหัวข้อ 17.7, หรือการเปิด/ปิด feature
ของ dependency) จำเป็นต้องระบุ**คุณสมบัติมากกว่าหนึ่งอย่าง** พร้อมกัน ซึ่ง TOML รองรับสองวิธีเขียนที่ให้ผลลัพธ์
เหมือนกันทุกประการ:

#### วิธีที่ 1: Inline table (สั้น กระชับ อยู่ในบรรทัดเดียว)

```toml
[dependencies]
serde = { version = "1.0", features = ["derive"] }
task_core = { path = "../core" }
```

#### วิธีที่ 2: Expanded table (ใช้ dotted key เป็น section ของตัวเอง)

```toml
[dependencies.serde]
version = "1.0"
features = ["derive"]

[dependencies.task_core]
path = "../core"
```

ทั้งสองแบบ**สร้างข้อมูลเดียวกันเป๊ะใน parser ของ TOML** — เลือกใช้ตามความชอบและความยาวของค่าที่ต้องระบุ:
inline table เหมาะกับกรณีที่มีไม่กี่คีย์และค่าสั้น (เขียนบรรทัดเดียวอ่านง่ายกว่า) ส่วน expanded table เหมาะกับ
กรณีที่ต้องระบุหลายคีย์ที่มีค่ายาว ๆ หรือ array ยาว ๆ (เขียนแยกบรรทัดอ่านง่ายกว่า ไม่ต้องนับวงเล็บปิดให้ครบ) —
โปรเจกต์จริงจำนวนมากใช้ทั้งสองแบบปนกันในไฟล์เดียวตามความเหมาะสมของแต่ละ dependency ไม่มีกฎตายตัวว่าต้องเลือก
แบบเดียวทั้งไฟล์

คีย์ที่ใช้ได้ใน table ของ dependency (ทั้งสองรูปแบบ) มีมากกว่าที่เห็นมาแล้ว ตัวที่สำคัญที่ควรรู้จักคือ:

| คีย์ | ความหมาย |
|---|---|
| `version` | version requirement แบบ SemVer (ตามหัวข้อ 2.8) |
| `path` | ใช้โค้ดจาก local path แทน crates.io (หัวข้อ 17.7) |
| `git` | ใช้โค้ดจาก git repository ตรง ๆ (ระบุ `branch`/`tag`/`rev` เสริมได้) |
| `features` | array ของชื่อ feature flag ที่ต้องการเปิดเพิ่มจาก dependency ตัวนั้น |
| `default-features` | ตั้งเป็น `false` เพื่อปิด default feature ของ dependency (แล้วเลือกเปิดเฉพาะที่ต้องการผ่าน `features`) |
| `optional` | ตั้งเป็น `true` เพื่อทำให้ dependency ตัวนี้ "ไม่บังคับต้องดึงมา" — ต้องเปิดผ่าน feature flag เท่านั้น (หัวข้อ 17.12) |

ตัวอย่างที่ใช้หลายคีย์พร้อมกัน — สมมติต้องการใช้ `serde` แบบปิด default feature แล้วเปิดแค่ `derive`:

```toml
[dependencies]
serde = { version = "1.0", default-features = false, features = ["derive"] }
```

**ทำไมบางทีมถึงปิด `default-features`?** เพราะ dependency บางตัวเปิด feature จำนวนมากเป็นค่าเริ่มต้น ซึ่งบาง
feature อาจดึง dependency ย่อยเพิ่มเข้ามาโดยที่โปรเจกต์ของคุณไม่ได้ใช้เลย การปิด default แล้วเปิดเฉพาะ feature
ที่ใช้จริงช่วยลดขนาด dependency tree, ลดเวลา compile, และลด attack surface (โค้ดที่ compile เข้ามาโดยไม่ได้ใช้
งานจริงแต่ยังเป็นช่องโหว่ความปลอดภัยที่เป็นไปได้) — เป็น trade-off เดียวกับที่ Part 2 อธิบายเรื่อง
`[dev-dependencies]` ไม่ปนกับ `[dependencies]` แค่ในระดับ feature ย่อยลงไปอีกขั้น

### 17.12 Optional Dependencies และ Feature Flags: การเปิด/ปิดความสามารถของ Crate ตัวเอง

หัวข้อที่แล้วพูดถึงการเปิด/ปิด feature ของ **dependency ที่คนอื่นเขียน** ตอนนี้มาดูสิ่งที่ทรงพลังกว่านั้นคือการ
ให้ **crate ของตัวเอง** มี feature ที่ผู้ใช้เลือกเปิด/ปิดได้ — นี่คือกลไกเดียวกับที่ crate ใหญ่ ๆ ในระบบนิเวศ
Rust อย่าง `tokio` หรือ `serde` ใช้ เพื่อให้ผู้ใช้ compile เข้าเฉพาะส่วนที่ต้องการจริง ไม่ต้องแบกทั้ง crate เต็ม
รูปแบบเสมอไป

#### แนวคิด: Optional Dependency

Dependency ที่ตั้ง `optional = true` จะ**ไม่ถูกดึงเข้ามาคอมไพล์โดย default** แม้จะเขียนอยู่ใน
`[dependencies]` ก็ตาม — มันจะถูกดึงมาก็ต่อเมื่อมี **feature** ที่เปิดมันไว้อย่างชัดเจนเท่านั้น:

```toml
[package]
name = "report_gen"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { version = "1.0", optional = true, features = ["derive"] }

[features]
default = []
json_export = ["serde"]
```

**อธิบายทีละส่วน**:

- `serde = { version = "1.0", optional = true, ... }` — บอก Cargo ว่า `serde` เป็น dependency ที่ "มีตัวเลือก"
  จะไม่ compile เข้ามาถ้าไม่มีใครขอ
- `[features]` — section ใหม่ที่ยังไม่เคยเจอมาก่อนในหลักสูตรนี้ ใช้นิยาม **ชื่อ feature ที่ผู้ใช้ crate นี้
  เลือกเปิดได้** โดยแต่ละ feature คือ key ที่ map ไปยัง array ของสิ่งที่ feature นั้น "เปิดตามไปด้วย" (เรียกว่า
  feature's dependencies — ทั้ง optional dependency ตัวอื่น หรือ feature อื่นในตัวมันเองก็ได้)
- `default = []` — feature พิเศษชื่อ `default` (ชื่อนี้ถูกสงวนไว้เฉพาะความหมายนี้) กำหนดว่า **ถ้าผู้ใช้ไม่ระบุ
  feature อะไรเลย** จะได้ feature อะไรเปิดอยู่บ้างโดยอัตโนมัติ ในตัวอย่างนี้ตั้งเป็น array เปล่า หมายความว่า
  **ไม่มี feature ไหนเปิดเป็นค่าเริ่มต้นเลย** (ต้องเปิด `json_export` เองอย่างตั้งใจ)
- `json_export = ["serde"]` — ประกาศ feature ชื่อ `json_export` ที่เมื่อเปิดแล้วจะ **เปิด optional dependency
  ชื่อ `serde` ตามไปด้วยโดยอัตโนมัติ** (ชื่อ `"serde"` ในนี้อ้างถึงชื่อ dependency ที่ประกาศไว้ใน
  `[dependencies]` ด้านบนพอดี — Cargo เข้าใจว่าชื่อที่ตรงกับ dependency ที่มีอยู่ หมายถึง "เปิด dependency ตัวนั้น
  ให้ compile เข้ามา" ไม่ใช่นิยาม feature ใหม่ซ้อนกัน)

#### ใช้ `#[cfg(feature = "...")]` ควบคุมโค้ดตามฝั่ง Rust

ฝั่ง TOML บอก Cargo ว่าจะ "compile dependency เข้ามาหรือไม่" แต่ฝั่งโค้ด Rust ต้องบอก compiler ว่า "โค้ดส่วนไหน
ให้มีอยู่ก็ต่อเมื่อ feature นั้นเปิด" ผ่าน attribute `#[cfg(feature = "...")]` — นี่คือกลไก **conditional
compilation** (คอมไพล์แบบมีเงื่อนไข) แนวคิดเดียวกับ `#[cfg(test)]` ที่เจอมาแล้วตั้งแต่ Part 2 (ซึ่งจริง ๆ
`test` ก็คือ "feature พิเศษ" ที่ Cargo เปิดให้อัตโนมัติตอนรัน `cargo test` เท่านั้น — เป็น cfg predicate ชนิด
เดียวกัน แค่ Cargo ควบคุมให้เอง ไม่ต้องประกาศใน `[features]`)

```rust
// src/lib.rs
pub struct Report {
    pub title: String,
    pub score: u32,
}

impl Report {
    pub fn new(title: impl Into<String>, score: u32) -> Self {
        Report { title: title.into(), score }
    }

    /// method นี้อยู่เสมอ ไม่ว่าเปิด feature ไหน
    pub fn summary(&self) -> String {
        format!("{}: {}", self.title, self.score)
    }

    /// method นี้จะ "มีอยู่จริง" ในผลลัพธ์ compile ก็ต่อเมื่อเปิด feature "json_export" เท่านั้น
    /// ถ้าไม่เปิด feature นี้ method นี้จะไม่ถูก compile เข้ามาเลย
    /// (ไม่ใช่แค่ error ตอนเรียก — มันไม่มีอยู่ในโค้ดที่ผ่าน parser ไปยังขั้นถัดไปด้วยซ้ำ)
    #[cfg(feature = "json_export")]
    pub fn to_json(&self) -> String {
        // ในตัวอย่างจริงควรใช้ serde_json::to_string แต่ที่นี่เขียนแบบ manual
        // เพื่อโฟกัสที่แนวคิด feature flag ไม่ใช่รายละเอียดของ serde
        format!("{{\"title\":\"{}\",\"score\":{}}}", self.title, self.score)
    }
}
```

และในโปรแกรมที่เรียกใช้ (`src/bin/demo.rs` หรือ `src/main.rs`) ก็ใช้ `#[cfg(feature = "...")]` ควบคุมโค้ดฝั่ง
เรียกใช้ได้เหมือนกัน:

```rust
// src/bin/demo.rs
use report_gen::Report;

fn main() {
    let report = Report::new("รายงานไตรมาส 3", 95);
    println!("{}", report.summary());

    #[cfg(feature = "json_export")]
    println!("{}", report.to_json());

    #[cfg(not(feature = "json_export"))]
    println!("(json_export feature ปิดอยู่ ข้ามการแสดงผล JSON)");
}
```

รันโดยไม่เปิด feature อะไรเลย (ใช้ `default` ซึ่งเราตั้งเป็น `[]` ไว้):

```bash
cargo run --bin demo
```

```
   Compiling report_gen v0.1.0 (/home/user/report_gen)
    Finished dev [unoptimized + debuginfo] target(s) in 0.31s
     Running `target/debug/demo`
รายงานไตรมาส 3: 95
(json_export feature ปิดอยู่ ข้ามการแสดงผล JSON)
```

สังเกตว่า `serde` **ไม่ถูกดาวน์โหลด/compile เลย** ในการ build นี้ (ลองสังเกตบรรทัด `Compiling` — จะไม่มี
`serde`, `serde_derive` ปรากฏขึ้นมาให้เห็น) เพราะไม่มีใครเปิด feature `json_export` ที่จะไปเปิด optional
dependency `serde` ตามมา

ทีนี้เปิด feature ด้วย flag `--features`:

```bash
cargo run --bin demo --features json_export
```

```
    Downloading crates ...
   Compiling serde v1.0.210
   Compiling serde_derive v1.0.210
   Compiling report_gen v0.1.0 (/home/user/report_gen)
    Finished dev [unoptimized + debuginfo] target(s) in 1.42s
     Running `target/debug/demo`
รายงานไตรมาส 3: 95
{"title":"รายงานไตรมาส 3","score":95}
```

ครั้งนี้ `serde`/`serde_derive` ถูก compile เข้ามาจริง และ method `to_json()` ก็มีอยู่จริงแล้วให้เรียกได้ —
นี่คือพลังของ feature flag: **ผู้ใช้ crate ของคุณที่ไม่ต้องการความสามารถ export JSON เลย จะไม่ต้องแบก
dependency `serde` เข้าไปใน binary ของเขาโดยไม่จำเป็น** ทำให้ binary เล็กลง compile เร็วขึ้น และ attack surface
เล็กลง ในขณะที่คนที่ต้องการความสามารถนี้จริง ๆ ก็เปิดใช้ได้ง่าย ๆ ด้วย flag เดียว

**ข้อควรรู้เพิ่ม**: ถ้าลองเรียก `report.to_json()` ในโค้ดที่**ไม่ได้อยู่ใต้** `#[cfg(feature = "json_export")]`
เอง (เช่นเขียน `println!("{}", report.to_json());` แบบไม่มี attribute คลุม) ในขณะที่ build โดยไม่เปิด feature
`json_export` เลย จะได้ compiler error ทันที เพราะ method `to_json` ไม่มีอยู่จริงในผลลัพธ์การ parse (มันถูก
"ตัดออก" ไปตั้งแต่ก่อนขั้นตอน type checking ด้วยซ้ำ):

```
error[E0599]: no method named `to_json` found for struct `Report` in the current scope
```

นี่ตอกย้ำว่า `#[cfg(...)]` ไม่ใช่ "if-condition ที่ประเมินตอน runtime" แบบ `if` ปกติ — มันคือคำสั่งบอก compiler
ว่า **"โค้ดตรงนี้อยู่จริงหรือไม่อยู่จริง ตัดสินใจตั้งแต่ก่อนคอมไพล์"** ต่างจาก `if some_runtime_condition { ... }`
ที่โค้ดทั้งสองฝั่งจะถูก compile เข้ามาเสมอ แค่เลือก execute ฝั่งไหนตอน runtime — `#[cfg(feature = "...")]` ตัด
โค้ดออกไปเลยทั้งกิ่ง ไม่มีอยู่ใน binary สุดท้ายด้วยซ้ำถ้า feature ไม่เปิด (คนละกลไกกันโดยสิ้นเชิงกับ `if`)

#### `cargo add` กับ optional dependency

ถ้าใช้ `cargo add` แทนการแก้ไฟล์เอง สามารถเพิ่ม optional dependency ได้ตรง ๆ ด้วย flag `--optional`:

```bash
cargo add serde --optional --features derive
```

คำสั่งนี้จะเขียน `serde = { version = "...", optional = true, features = ["derive"] }` ลงใน `[dependencies]`
ให้อัตโนมัติ แต่ **จะไม่สร้าง section `[features]` หรือ feature ที่เปิด `serde` ให้เอง** — ต้องเข้าไปเพิ่ม
`[features]` เองด้วยมือเสมอ เพราะ Cargo ไม่รู้ว่าคุณต้องการตั้งชื่อ feature ว่าอะไร (`json_export` หรือชื่ออื่น
ตามที่ออกแบบ API ของ crate ตัวเอง)

### 17.13 Dev-Dependencies ทบทวนอีกครั้ง: ตัวอย่างที่จับต้องได้

Part 2 หัวข้อ (`[dev-dependencies]`) อธิบายไว้แล้วว่า table นี้เก็บ dependency ที่ใช้แค่ตอนพัฒนา/ทดสอบ ไม่ถูก
compile เข้า binary จริง — ตอนนี้มาดูตัวอย่างที่จับต้องได้จริงว่าทำไมการแยก table นี้ถึงสำคัญในทางปฏิบัติ

สมมติเรากลับไปที่ crate `task_core` จากหัวข้อ 17.6 แล้วอยากเขียน test ที่เทียบ struct ที่ซับซ้อนแบบละเอียด
มากขึ้น — crate ชื่อ `pretty_assertions` (นิยมใช้กันมากในระบบนิเวศ Rust) ช่วยแสดงผล diff ของค่าที่ assert ไม่
ตรงกันแบบสวยงามอ่านง่ายกว่า `assert_eq!` มาตรฐานมาก (highlight ส่วนที่ต่างด้วยสี) แต่ **ไม่มีประโยชน์อะไรเลยกับ
โปรแกรมที่ compile ไปใช้งานจริง** เพราะมันมีหน้าที่แค่ "แสดงผล diff ให้อ่านง่ายตอน test fail" เท่านั้น:

```toml
[package]
name = "task_core"
version = "0.1.0"
edition = "2021"

[dependencies]

[dev-dependencies]
pretty_assertions = "1"
```

```rust
// task_core/src/lib.rs (ส่วน test เพิ่มเข้ามา)
#[cfg(test)]
mod tests {
    use super::*;
    use pretty_assertions::assert_eq; // shadow assert_eq! มาตรฐานด้วยตัวที่แสดง diff สวยกว่า

    #[test]
    fn adding_task_starts_as_todo() {
        let mut manager = TaskManager::new();
        let id = manager.add_task("ทดสอบ");
        let task = manager.all_tasks().iter().find(|t| t.id == id).unwrap();
        assert_eq!(task.status, TaskStatus::Todo);
    }
}
```

**ทำไมต้องแยก `pretty_assertions` ไว้ใน `[dev-dependencies]` แทน `[dependencies]`?** ลองจินตนาการว่าถ้าใส่
มันปนกับ `[dependencies]` ปกติ — ทุกครั้งที่ `task_cli` หรือ `task_web` compile ตัวเองแบบ release เพื่อแจกจ่าย
จริง (`cargo build --release`) แม้จะ**ไม่ได้เรียกใช้ `pretty_assertions` เลยแม้แต่บรรทัดเดียวในโค้ดที่ไม่ใช่
test** dependency ตัวนี้ก็จะยังถูกดึงมา compile เป็นส่วนหนึ่งของ dependency graph ของ `task_core` อยู่ดี (เพราะ
`task_cli`/`task_web` depend on `task_core` และ `task_core` ก็ list มันไว้ใน `[dependencies]`) ทำให้:

- **เวลา compile นานขึ้นโดยไม่จำเป็น** — ต้อง compile crate ที่ไม่มีทางถูกเรียกใช้จริงตอน production เลย
- **Binary มีขนาดใหญ่ขึ้น** (แม้ dead code elimination ของ compiler มักจะตัดโค้ดที่ไม่ถูกเรียกออกไปได้เยอะ แต่
  ไม่ใช่ทุกกรณีจะตัดได้ 100% โดยเฉพาะถ้า crate นั้นมี static initialization หรือ macro ที่ generate โค้ดจำนวนมาก)
- **เพิ่ม attack surface โดยไม่มีประโยชน์** — โค้ดของ dependency ที่ไม่จำเป็นต่อ production แต่ถูก compile เข้า
  binary จริง คือความเสี่ยงด้านความปลอดภัยที่ไม่มีเหตุผลอะไรมารองรับ (ถ้า `pretty_assertions` มี security
  vulnerability ในเวอร์ชันหนึ่ง โปรแกรม production ของคุณจะเสี่ยงตามไปด้วยทั้งที่ไม่ได้ใช้ฟีเจอร์นั้นเลย)

การใส่ไว้ใน `[dev-dependencies]` แก้ปัญหาทั้งหมดนี้ เพราะ **Cargo รับประกันว่า `[dev-dependencies]` จะถูก
compile เข้ามาเฉพาะตอนสั่ง `cargo test`, `cargo bench`, หรือ compile โค้ดตัวอย่างใน `examples/` เท่านั้น** —
เมื่อ `task_cli` สั่ง `cargo build --release` เพื่อแจกจ่ายจริง `pretty_assertions` จะไม่ถูกแตะเลยแม้แต่นิดเดียว
เพราะ dev-dependencies ของ `task_core` (ซึ่งเป็น dependency ของ `task_cli`) ไม่ถูกนำมาพิจารณาตอน build
`task_cli` เอง — **นี่คือกฎสำคัญที่ควรจำ: `[dev-dependencies]` ของ dependency ของคุณ ไม่มีวันกระทบ dependency
graph ของโปรแกรมคุณเลย ไม่ว่ากรณีใด** ต่างจาก `[dependencies]` ปกติที่ transitive dependency (dependency ของ
dependency) จะถูกดึงมาด้วยเสมอ

### 17.14 ตัวอย่างจริงแบบเต็มรูปแบบ: library_core + library_cli

มาประกอบทุกความรู้ในบทนี้เข้าด้วยกันเป็นตัวอย่างที่สมบูรณ์ที่สุด — ออกแบบ workspace 2 member สำหรับระบบห้องสมุด
(ยืม-คืนหนังสือ) ในสไตล์เดียวกับการโมเดลโดเมนที่เรียนไปแล้วใน Part 9 (structs, methods, impl) แต่ครั้งนี้แยก
เป็นสอง package จริง ๆ ไม่ใช่แค่ไฟล์เดียว

โครงสร้างไฟล์:

```
library_workspace/
├── Cargo.toml
├── library_core/
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs
└── library_cli/
    ├── Cargo.toml
    └── src/
        └── main.rs
```

#### Root workspace manifest

```toml
# library_workspace/Cargo.toml
[workspace]
resolver = "2"
members = [
    "library_core",
    "library_cli",
]
```

#### Member 1: library_core (library crate)

```toml
# library_workspace/library_core/Cargo.toml
[package]
name = "library_core"
version = "0.1.0"
edition = "2021"
description = "Core domain logic for a simple library (book lending) system"
license = "MIT"

[dependencies]
```

```rust
// library_workspace/library_core/src/lib.rs
//! library_core: โดเมนหลักของระบบห้องสมุดง่าย ๆ
//! เก็บ struct `Book`, `Library` และ logic การยืม-คืนหนังสือ
//! ไม่มีส่วนติดต่อผู้ใช้ใด ๆ อยู่ในนี้เลย (ไม่มี println!, ไม่มีการอ่าน stdin)
//! เพื่อให้ crate อื่น (CLI, web, GUI) เอาไปประกอบ UI ของตัวเองได้อย่างอิสระ

/// หนังสือหนึ่งเล่มในระบบห้องสมุด
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Book {
    pub isbn: String,
    pub title: String,
    pub author: String,
    pub is_borrowed: bool,
}

impl Book {
    /// สร้างหนังสือเล่มใหม่ สถานะเริ่มต้นคือ "ยังไม่ถูกยืม"
    /// รับ `impl Into<String>` เพื่อให้เรียกได้ทั้งด้วย &str และ String ตรง ๆ โดยไม่ต้อง .to_string() เอง
    pub fn new(isbn: impl Into<String>, title: impl Into<String>, author: impl Into<String>) -> Self {
        Book {
            isbn: isbn.into(),
            title: title.into(),
            author: author.into(),
            is_borrowed: false,
        }
    }
}

/// ข้อผิดพลาดที่อาจเกิดขึ้นระหว่างการดำเนินการกับห้องสมุด
#[derive(Debug, PartialEq, Eq)]
pub enum LibraryError {
    BookNotFound(String),
    AlreadyBorrowed(String),
    NotBorrowed(String),
}

/// ห้องสมุด: เก็บรายการหนังสือทั้งหมด และให้ method จัดการยืม-คืน
#[derive(Debug, Default)]
pub struct Library {
    books: Vec<Book>,
}

impl Library {
    pub fn new() -> Self {
        Library { books: Vec::new() }
    }

    pub fn add_book(&mut self, book: Book) {
        self.books.push(book);
    }

    pub fn find_by_isbn(&self, isbn: &str) -> Option<&Book> {
        self.books.iter().find(|b| b.isbn == isbn)
    }

    pub fn borrow_book(&mut self, isbn: &str) -> Result<(), LibraryError> {
        let book = self
            .books
            .iter_mut()
            .find(|b| b.isbn == isbn)
            .ok_or_else(|| LibraryError::BookNotFound(isbn.to_string()))?;

        if book.is_borrowed {
            return Err(LibraryError::AlreadyBorrowed(isbn.to_string()));
        }
        book.is_borrowed = true;
        Ok(())
    }

    pub fn return_book(&mut self, isbn: &str) -> Result<(), LibraryError> {
        let book = self
            .books
            .iter_mut()
            .find(|b| b.isbn == isbn)
            .ok_or_else(|| LibraryError::BookNotFound(isbn.to_string()))?;

        if !book.is_borrowed {
            return Err(LibraryError::NotBorrowed(isbn.to_string()));
        }
        book.is_borrowed = false;
        Ok(())
    }

    /// คืน iterator ของหนังสือที่ "ยังว่างให้ยืม" เท่านั้น (ยังไม่ถูกยืม)
    pub fn available_books(&self) -> impl Iterator<Item = &Book> {
        self.books.iter().filter(|b| !b.is_borrowed)
    }

    pub fn all_books(&self) -> &[Book] {
        &self.books
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn borrow_and_return_flow() {
        let mut lib = Library::new();
        lib.add_book(Book::new("111", "The Rust Book", "Steve & Carol"));

        assert!(lib.borrow_book("111").is_ok());
        assert_eq!(
            lib.borrow_book("111"),
            Err(LibraryError::AlreadyBorrowed("111".to_string()))
        );
        assert!(lib.return_book("111").is_ok());
    }

    #[test]
    fn borrow_unknown_book_fails() {
        let mut lib = Library::new();
        assert_eq!(
            lib.borrow_book("999"),
            Err(LibraryError::BookNotFound("999".to_string()))
        );
    }
}
```

#### Member 2: library_cli (binary crate ที่ depend on library_core ผ่าน path dependency)

```toml
# library_workspace/library_cli/Cargo.toml
[package]
name = "library_cli"
version = "0.1.0"
edition = "2021"
publish = false

[dependencies]
library_core = { path = "../library_core" }
```

สังเกตว่า `library_cli` ตั้ง `publish = false` ไว้ (ตามหลักที่อธิบายในหัวข้อ 17.10) เพราะเป็นโปรแกรม CLI ตัวอย่าง
ที่ไม่มีเหตุผลจะ publish ขึ้น crates.io เลย ในขณะที่ `library_core` **ไม่ได้ตั้ง** `publish = false` (ปล่อยให้
publish ได้ตามปกติ) เพราะเป็น library ทั่วไปที่อาจมีประโยชน์กับคนอื่น

```rust
// library_workspace/library_cli/src/main.rs
use library_core::{Book, Library};

fn main() {
    let mut library = Library::new();
    library.add_book(Book::new(
        "978-1",
        "The Rust Programming Language",
        "Steve Klabnik",
    ));
    library.add_book(Book::new("978-2", "Programming Rust", "Jim Blandy"));

    println!("หนังสือทั้งหมดในห้องสมุด:");
    for book in library.all_books() {
        println!("  - {} โดย {}", book.title, book.author);
    }

    match library.borrow_book("978-1") {
        Ok(()) => println!("ยืม 978-1 สำเร็จ"),
        Err(e) => println!("ยืมไม่สำเร็จ: {e:?}"),
    }

    println!("หนังสือที่ยังว่างให้ยืม:");
    for book in library.available_books() {
        println!("  - {}", book.title);
    }
}
```

#### รันและทดสอบทั้ง workspace

```bash
cd library_workspace
cargo build
```

```
   Compiling library_core v0.1.0 (/home/user/library_workspace/library_core)
   Compiling library_cli v0.1.0 (/home/user/library_workspace/library_cli)
    Finished dev [unoptimized + debuginfo] target(s) in 0.42s
```

```bash
cargo run -p library_cli
```

```
    Finished dev [unoptimized + debuginfo] target(s) in 0.02s
     Running `target/debug/library_cli`
หนังสือทั้งหมดในห้องสมุด:
  - The Rust Programming Language โดย Steve Klabnik
  - Programming Rust โดย Jim Blandy
ยืม 978-1 สำเร็จ
หนังสือที่ยังว่างให้ยืม:
  - Programming Rust
```

```bash
cargo test --workspace
```

```
     Running unittests src/lib.rs (target/debug/deps/library_core-...)

running 2 tests
test tests::borrow_and_return_flow ... ok
test tests::borrow_unknown_book_fails ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

ตัวอย่างนี้แสดงให้เห็นทุกแนวคิดหลักของบทนี้ประกอบกันจริง ๆ: **package สองใบ (`library_core`,
`library_cli`) ที่รวมกันเป็น workspace เดียว, แต่ละ package คือ crate คนละต้น (module tree คนละต้นตามหัวข้อ
17.2), `library_cli` อ้างอิง `library_core` ผ่าน path dependency (หัวข้อ 17.7), ทั้งคู่แชร์ `Cargo.lock`/
`target/` ร่วมกัน (หัวข้อ 17.8), ควบคุมด้วย `-p`/`--workspace` (หัวข้อ 17.9), และตั้งค่า publish ให้เหมาะสมกับ
แต่ละ member (หัวข้อ 17.10)** — นี่คือรูปแบบที่คุณจะเจอซ้ำ ๆ ในโปรเจกต์ Rust ระดับ production จริงตลอดหลักสูตร
ที่เหลือ โดยเฉพาะตอนทำโปรเจกต์ CLI จริงใน Part 59 และ full-stack project ใน Part 92-94

## กับดักที่พบบ่อย (Common Pitfalls)

**1. ลืมว่า path dependency คือ relative path จากตำแหน่งไฟล์ Cargo.toml ไม่ใช่จาก root workspace**

มือใหม่มักเข้าใจผิดว่า `path` ใน path dependency นับจาก root ของ workspace เสมอ แล้วเขียนผิดแบบนี้ในไฟล์
`library_cli/Cargo.toml`:

```toml
[dependencies]
library_core = { path = "library_core" }   # ผิด! คิดว่านับจาก root
```

จะได้ error ทันทีตอน build:

```
error: failed to load source for dependency `library_core`

Caused by:
  Unable to update /home/user/library_workspace/library_cli/library_core

Caused by:
  failed to read `/home/user/library_workspace/library_cli/library_core/Cargo.toml`

Caused by:
  No such file or directory (os error 2)
```

**วิธีแก้**: `path` นับจากตำแหน่งของไฟล์ `Cargo.toml` ที่กำลังเขียนอยู่เสมอ — เนื่องจาก `library_cli/` และ
`library_core/` เป็น sibling กัน (อยู่คนละโฟลเดอร์ในระดับเดียวกันใต้ root) ต้องเขียน `path = "../library_core"`
(ขึ้นไปหนึ่งระดับก่อน แล้วค่อยลงไปหา `library_core`) ไม่ใช่ `path = "library_core"` ตรง ๆ

**2. สร้าง workspace แต่ยังเผลอมี `[package]` ปนกับ `[workspace]` ในไฟล์เดียวกันโดยไม่ตั้งใจ (Root package ปนกับ Virtual workspace)**

Cargo รองรับสองรูปแบบของ workspace root: **virtual workspace** (root มีแค่ `[workspace]` ไม่มี `[package]`
เลย — แบบที่ใช้ตลอดบทนี้) กับ **root ที่เป็น package ด้วย** (root มีทั้ง `[package]` และ `[workspace]` พร้อมกัน
ทำให้ root โฟลเดอร์เองก็เป็น member หนึ่งของ workspace ไปในตัว) ทั้งสองแบบใช้ได้ถูกต้องตามกฎ Cargo แต่ปัญหาที่
มือใหม่มักเจอคือ **สร้าง root package ด้วย `cargo new` ธรรมดาก่อน (ได้ `[package]` มาแล้ว) แล้วค่อยเพิ่ม
`[workspace]` กับ `members` เข้าไปทีหลังโดยไม่ได้ตั้งใจให้ root เป็น member จริง ๆ** ทำให้ path ของ dependency
สับสน เพราะ root เองก็มี `src/main.rs`/`src/lib.rs` ปนอยู่ด้วย

ตัวอย่างสถานการณ์ที่งง: root `Cargo.toml` หน้าตาแบบนี้ (มีทั้ง `[package]` และ `[workspace]`):

```toml
[package]
name = "my_root_app"
version = "0.1.0"
edition = "2021"

[workspace]
members = ["core"]

[dependencies]
```

ที่นี้ `my_root_app` (จาก `src/main.rs` ที่ root) จะเป็น member หนึ่งของ workspace โดยอัตโนมัติ (ไม่ต้องเขียนชื่อ
ตัวเองใน `members` ก็ได้ Cargo เข้าใจเอง) **ไม่ผิด** แต่ต้องตั้งใจออกแบบแบบนี้จริง ๆ ไม่ใช่เผลอทำ — ถ้าตั้งใจจะให้
เป็น "root ที่เป็นแค่ตัวประสาน ไม่มี source code ของตัวเอง" (virtual workspace แบบที่บทนี้แนะนำเป็นหลัก เพราะ
เข้าใจง่ายกว่าสำหรับมือใหม่) ต้อง**ลบ** `[package]`, `[dependencies]`, และไฟล์ `src/` ที่ root ออกทั้งหมด เหลือ
แค่ `[workspace]` เพียว ๆ

**วิธีแก้/ป้องกัน**: เวลาจะสร้าง workspace ใหม่ ให้เริ่มจากสร้างโฟลเดอร์เปล่า เขียน root `Cargo.toml` ที่มีแค่
`[workspace]` ก่อนอย่างตั้งใจ แล้วค่อยรัน `cargo new` แยกสำหรับแต่ละ member ทีละตัว (เช่น `cargo new
core --lib`, `cargo new cli`) จะชัดเจนกว่าการพยายามดัด `Cargo.toml` ที่ `cargo new` สร้างให้ตอนแรกให้กลายเป็น
workspace root

**3. เพิ่ม feature ผิด ทำให้ `serde` (หรือ optional dependency ตัวอื่น) ไม่ถูก compile เข้ามาทั้งที่คิดว่าเปิดแล้ว**

เขียน `[features]` ผิดแบบนี้ (สลับชื่อ feature กับชื่อ dependency กัน หรือพิมพ์ชื่อ dependency ผิด):

```toml
[dependencies]
serde = { version = "1.0", optional = true }

[features]
default = []
json_export = ["serd"]   # พิมพ์ผิด! ต้องเป็น "serde"
```

จะได้ error ทันทีตอน build:

```
error: feature `json_export` includes `serd` which is neither a dependency nor another feature
```

**วิธีแก้**: ชื่อในลิสต์ทางขวาของ feature ต้องตรงกับชื่อ dependency ที่ประกาศใน `[dependencies]` เป๊ะ ๆ ทุกตัวอักษร
(case-sensitive) หรือตรงกับชื่อ feature อื่นที่ประกาศไว้ใน `[features]` ก็ได้เหมือนกัน (feature เปิด feature
อื่นต่อกันเป็นทอด ๆ ได้) ตรวจสอบการพิมพ์ให้ตรงเสมอ และถ้าไม่แน่ใจว่า feature ไหนเปิดอะไรอยู่บ้าง ใช้คำสั่ง
`cargo tree --features json_export -e features` เพื่อดู feature graph จริงได้

**4. รัน `cargo run` เปล่า ๆ ที่ root workspace ที่มี binary หลายตัวจากหลาย member โดยไม่ระบุ `-p`/`--bin`**

```bash
cd task_workspace   # workspace ที่มีทั้ง task_cli และ task_web เป็น binary
cargo run
```

```
error: `cargo run` could not determine which package to run. Use the `-p` option to specify a package,
or the `default-run` manifest key.
available packages: task_core, task_cli, task_web
```

**วิธีแก้**: ระบุ `-p <package>` เสมอเมื่อ workspace มี binary มากกว่าหนึ่งตัวกระจายอยู่คนละ member (ต่างจาก
กรณีหัวข้อ 17.4 ที่ error message จะพูดถึง "binaries" ภายใน package เดียว — สังเกตความต่างของคำใน error message
ว่าพูดถึง "package" หรือ "binary" เพื่อรู้ว่าต้องใส่ `-p` หรือ `--bin`) หรือกำหนด `default-run` ไว้ในแต่ละ
package ที่มีมากกว่าหนึ่ง binary และตั้งใจให้ `cargo run` ที่ root workspace ไม่ระบุ flag รันตัวไหนเป็นค่าเริ่มต้น
ก็ได้เช่นกัน (แต่ Cargo ยังต้องรู้ว่าจะรัน "package" ไหนก่อนอยู่ดีถ้ามีมากกว่าหนึ่ง package ที่มี binary)

**5. เข้าใจผิดว่า `[dev-dependencies]` ของ dependency (ของคนอื่น) จะกระทบ dependency graph ของตัวเอง**

บางคนเห็นว่า crate ที่ตนใช้ (เช่น `library_core`) มี `[dev-dependencies]` ขนาดใหญ่ (เช่นใช้ `criterion` สำหรับ
benchmark) แล้วกังวลว่าโปรแกรมของตัวเอง (`library_cli`) จะต้อง compile `criterion` เข้ามาด้วยตอน
`cargo build --release` — **ความกังวลนี้ไม่ถูกต้อง** ตามที่อธิบายไว้ในหัวข้อ 17.13 `[dev-dependencies]` ของ
dependency จะไม่ถูกนำมาพิจารณาเลยเมื่อ compile package อื่นที่มาใช้มันเป็น dependency ปกติ ยืนยันได้ด้วยคำสั่ง
`cargo tree` (ไม่แสดง dev-dependencies ของ dependency โดย default):

```bash
cargo tree -p library_cli
```

```
library_cli v0.1.0 (/home/user/library_workspace/library_cli)
└── library_core v0.1.0 (/home/user/library_workspace/library_core)
```

ถ้าอยากดู dev-dependencies ด้วย (ของ package ตัวเองเท่านั้น ไม่ใช่ของ dependency ที่ลึกลงไป) ต้องใส่ flag
`-e normal,dev` แต่ผลลัพธ์จะยังยืนยันหลักการเดิมว่า dev-dependencies ของ `library_core` (dependency ของเรา)
ไม่โผล่มาให้เห็นเลย

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** สร้าง package ใหม่ชื่อ `calc_tool` ด้วย `cargo new calc_tool --lib` เขียนฟังก์ชัน public 4 ตัวใน
   `src/lib.rs`: `add`, `subtract`, `multiply`, `divide` (รับ `f64` สองตัว คืนค่า `f64`) จากนั้นสร้างไฟล์
   `src/main.rs` เพิ่มเข้ามาในแพ็กเกจเดียวกัน (ทำให้ `calc_tool` มีทั้ง lib crate และ binary crate) ให้
   `main.rs` เรียกทั้ง 4 ฟังก์ชันผ่าน `use calc_tool::{add, subtract, multiply, divide};` แล้วพิมพ์ผลลัพธ์
   (hint: อย่าลืม `pub` หน้าทุกฟังก์ชันใน `lib.rs` ไม่อย่างนั้น `main.rs` จะมองไม่เห็นเพราะกฎ visibility จาก
   Part 16)

2. **[กลาง]** ต่อยอดจากข้อ 1 ให้เพิ่มไฟล์ `src/bin/interactive.rs` เป็น binary ตัวที่สองในแพ็กเกจเดียวกัน ที่
   เรียกใช้ฟังก์ชันจาก `calc_tool` เหมือนกัน แต่แสดงผลในรูปแบบต่างจาก `main.rs` (เช่น `main.rs` แสดงผลลัพธ์ของ
   เลขคงที่ที่กำหนดไว้ในโค้ด ส่วน `interactive.rs` ให้จำลองการรับค่าจาก array ของคู่เลขหลาย ๆ คู่แล้ววนคำนวณทีละ
   คู่) ทดสอบรันทั้งสอง binary แยกกันด้วย `cargo run --bin calc_tool` และ `cargo run --bin interactive` (hint:
   ชื่อ binary จาก `main.rs` จะเท่ากับชื่อ package เสมอ ไม่ใช่ `main`)

3. **[กลาง-ยาก]** สร้าง Cargo workspace ใหม่ชื่อ `store_workspace` ที่มี 2 member: `store_core` (library crate
   เก็บ struct `Product` ที่มี field `name: String`, `price: f64`, `stock: u32` พร้อม method `total_value(&self)
   -> f64` ที่คำนวณ `price * stock as f64`) และ `store_report` (binary crate ที่ depend on `store_core` ผ่าน
   path dependency, สร้างสินค้าหลายตัวเก็บใน `Vec<Product>`, แล้วพิมพ์รายงานมูลค่าสินค้ารวมทั้งหมดในคลัง)
   ทดสอบว่า `cargo build --workspace` และ `cargo run -p store_report` ทำงานถูกต้อง (hint: อย่าลืมเขียน
   `resolver = "2"` ที่ root `Cargo.toml`)

4. **[ยาก/ประยุกต์]** ต่อยอดจากข้อ 3 ให้เพิ่ม optional dependency `serde` (พร้อม `features = ["derive"]`) เข้า
   `store_core` และเพิ่ม `[features]` section ที่มี `csv_export` เป็นชื่อ feature ที่เปิด `serde` ตามไป เขียน
   method ใหม่ใน `Product` ชื่อ `to_csv_row(&self) -> String` ที่คืนค่าเป็น string รูปแบบ
   `"name,price,stock"` (เช่น `"Rust Book,350.00,20"`) โดยครอบ method นี้ด้วย `#[cfg(feature = "csv_export")]`
   ทดสอบว่า `cargo build -p store_core` (ไม่เปิด feature) compile ผ่านโดยไม่ดึง `serde` มาเลย และ `cargo build
   -p store_core --features csv_export` compile ผ่านพร้อมดึง `serde` มาด้วย เขียนสรุปสั้น ๆ (3-5 บรรทัด)
   อธิบายว่าทำไม design แบบนี้ (แยก CSV export เป็น optional feature) ถึงมีประโยชน์กับผู้ใช้ library ที่ไม่
   ต้องการความสามารถนี้ เทียบกับการใส่ `serde` เป็น `[dependencies]` ปกติแบบไม่มีเงื่อนไขเลย

## สรุป

บทนี้เราซูมออกมาหนึ่งระดับจาก Part 16 (ที่พูดถึงการจัดระเบียบโค้ด**ภายใน**หนึ่ง crate ด้วย `mod`/`pub`/`use`)
มาดูภาพที่ใหญ่กว่า: **package** คือสิ่งที่ `Cargo.toml` หนึ่งไฟล์อธิบาย (มี name/version เป็นของตัวเอง),
**crate** คือ compilation unit ที่ `rustc` มองเห็นจริง (คือ module tree หนึ่งต้นที่มี root เป็น `src/lib.rs`
หรือไฟล์ binary หนึ่งไฟล์) — หนึ่ง package มี library crate ได้สูงสุด 1 ตัว แต่มี binary crate ได้หลายตัวผ่าน
`src/bin/*.rs` เราได้เห็น pattern มาตรฐาน "logic อยู่ที่ lib.rs, main.rs เป็นแค่เปลือกบาง ๆ" ที่ให้ประโยชน์ด้าน
testability และ reusability อย่างชัดเจน

จากนั้นเราขยายไปสู่ **workspace** — การรวมหลาย package ให้ใช้ `Cargo.lock` และ `target/` ร่วมกัน แก้ปัญหาเวอร์ชัน
ชนกันและลดเวลา compile ซ้ำซ้อน พร้อมกับ **path dependency** (`{ path = "../core" }`) ที่ทำให้ package ในเครือ
เดียวกันอ้างอิงกันได้โดยไม่ต้อง publish ขึ้น crates.io ก่อน เราเรียนคำสั่ง `-p`/`--workspace` สำหรับควบคุม scope
ของ `cargo build`/`test`/`run` ในบริบท workspace และภาพรวมของการ publish แต่ละ member อย่างเป็นอิสระ

ปิดท้ายด้วยการเจาะลึก dependency ที่ Part 2 เกริ่นไว้แค่ผิวเผิน: expanded table syntax (`[dependencies.foo]`),
optional dependency (`optional = true`), และ **feature flags** (`[features]`, `default = [...]`,
`#[cfg(feature = "...")]`) ที่ให้ผู้ใช้ crate ของคุณเลือกเปิด/ปิดความสามารถได้เอง พร้อมทวนความเข้าใจเรื่อง
`[dev-dependencies]` ด้วยตัวอย่างที่จับต้องได้กว่าเดิม

ตอนนี้คุณมีความเข้าใจโครงสร้างระดับ "หลายไฟล์ หลาย crate หลาย package" ครบถ้วนแล้ว — พร้อมสำหรับการกลับไปเจาะลึก
เนื้อหาของภาษา Rust เองต่อ ใน **Part 18: Generics เบื้องต้น** เราจะเรียนรู้วิธีเขียนฟังก์ชันและ struct ที่ทำงาน
ได้กับหลายชนิดข้อมูลโดยไม่ต้อง copy-paste โค้ดซ้ำสำหรับแต่ละ type — รากฐานสำคัญก่อนจะไปเรียน trait bounds และ
lifetime ในบทต่อ ๆ ไป

---

**Part ก่อนหน้า:** [Part 16: Modules และการจัดระเบียบโค้ด](part-016-modules.md) | **Part ถัดไป:** [Part 18: Generics เบื้องต้น](part-018-generics-basics.md)
