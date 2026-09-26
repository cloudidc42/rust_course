# Part 33: Testing: Integration Tests และการจัดระเบียบ Test Suite

> โมดูล: ระดับกลาง (Intermediate) | ระดับ: กลาง | เวลาโดยประมาณ: 120 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายความแตกต่างที่แท้จริงระหว่าง **unit test** (Part 32) กับ **integration test** ได้อย่างแม่นยำในระดับ
  "compilation unit" ไม่ใช่แค่ระดับ "อยู่คนละไฟล์" — และอธิบายได้ว่าทำไม integration test จะมีความหมายก็ต่อเมื่อ
  package นั้นมี **library crate** (`src/lib.rs`) เท่านั้น
- สร้างไดเรกทอรี `tests/` ที่ระดับ root ของโปรเจกต์อย่างถูกต้องตามธรรมเนียมที่ Cargo กำหนด และเข้าใจว่าทำไมไฟล์
  แต่ละไฟล์ใน `tests/` จึงถูกคอมไพล์เป็น **crate อิสระของตัวเอง** ที่มองเห็นได้เฉพาะ public API ของ library
- ใช้ `cargo test` เพื่อรันทั้ง unit test, integration test, และ doc-test พร้อมกันในคำสั่งเดียว และใช้
  `cargo test --test <ชื่อไฟล์>` เมื่อต้องการรันเฉพาะไฟล์ integration test ไฟล์เดียว
- จัดระเบียบโค้ดช่วยเทสที่ต้องใช้ร่วมกันในหลายไฟล์ผ่านธรรมเนียม `tests/common/mod.rs` ได้อย่างถูกต้อง และอธิบาย
  ได้ว่าทำไมชื่อไฟล์นี้ต้องเป็น `mod.rs` อยู่ใต้โฟลเดอร์ `common/` เท่านั้น ไม่ใช่ไฟล์แบน ๆ ชื่อ `common.rs`
- ระบุได้ว่า binary crate ล้วน ๆ (ไม่มี `src/lib.rs`) ไม่สามารถถูก integration test ด้วยวิธีนี้ได้ และรู้วิธีแก้
  ด้วย pattern "lib บาง ๆ + main บาง ๆ" ที่ต่อยอดจาก Part 17 มาใช้แก้ปัญหานี้โดยตรง
- วางกลยุทธ์การทดสอบทั้งโปรเจกต์แบบ "testing pyramid" ได้อย่างสมเหตุสมผล — รู้ว่าอะไรควรเป็น unit test อะไรควร
  เป็น integration test และรู้จักการมีอยู่ของ doc-test (Part 34) และเครื่องมือวัด coverage เป็นข้อมูลพื้นฐาน

## ความรู้ที่ต้องมีมาก่อน

- **Part 32 (Testing: Unit Tests)**: ต้องเข้าใจ `#[test]`, มาโคร `assert!`/`assert_eq!`/`assert_ne!`, ทำไมต้องมี
  `#[cfg(test)] mod tests { ... }` ครอบ test module ไว้ (เพื่อไม่ให้โค้ดทดสอบถูก compile เข้าไปใน binary จริงตอน
  release), และแนวคิดพื้นฐานของการ mock ผ่าน trait — บทนี้จะพูดถึง **สิ่งที่ unit test ทำไม่ได้** เป็นจุดเริ่มต้น
  จึงต้องเข้าใจขอบเขตของ unit test มาก่อนอย่างแม่นยำ
- **Part 16 (Modules และการจัดระเบียบโค้ด)**: ต้องเข้าใจกฎ visibility (`pub`, private-by-default) ให้แม่น เพราะ
  หัวใจของบทนี้คือการอธิบายว่า integration test มองเห็น "แค่สิ่งที่เป็น `pub`" เหมือนกับผู้ใช้ crate จากภายนอกทุก
  ประการ — เป็นกฎเดียวกับที่ Part 16 สอนเรื่อง private-by-default ทุกตัวอักษร ไม่มีข้อยกเว้นพิเศษให้กับ test
- **Part 17 (Packages, Crates, Workspaces)**: ต้องเข้าใจความต่างระหว่าง **package**, **crate**, และ **module**
  อย่างแม่นยำ (หัวข้อ 17.1-17.2) โดยเฉพาะกฎที่ว่า `src/lib.rs` และ `src/main.rs` คือคนละ crate กัน แต่ละ crate มี
  module tree ของตัวเองที่แยกจากกันสมบูรณ์ — บทนี้จะแสดงให้เห็นว่าไฟล์ใน `tests/` ก็เป็น **crate อิสระอีกต้นหนึ่ง**
  ตามกฎเดียวกันนี้เป๊ะ ๆ และ pattern "lib บาง ๆ + main บาง ๆ" จาก Part 17 หัวข้อ 17.3 จะถูกนำมาใช้แก้ปัญหาการ
  ทดสอบ binary crate โดยตรงในหัวข้อ 33.7
- **Part 2 (Cargo และโครงสร้างโปรเจกต์)**: ต้องคุ้นเคยกับโครงสร้างโฟลเดอร์มาตรฐานของ Cargo package (`Cargo.toml`,
  `src/`, `target/`) เพราะ `tests/` คือโฟลเดอร์ที่สี่ที่ Cargo รู้จักเป็นพิเศษ อยู่ระดับเดียวกับ `src/` ที่ root
  ของ package

## เนื้อหา

### 33.1 ทวนขอบเขตของ Unit Test: มันทดสอบอะไรไม่ได้?

ก่อนเริ่มเรื่องใหม่ ต้องทวนให้ชัดว่า unit test จาก Part 32 ทำอะไรได้บ้าง เพื่อเห็นว่ามันมี "ขอบเขต" ที่ไปไม่ถึง
บางจุด ซึ่งเป็นเหตุผลที่ Rust ต้องมี test อีกชนิดหนึ่งเพิ่ม

Unit test ใน Part 32 มีลักษณะเฉพาะ 3 อย่าง:

1. **อยู่ในไฟล์เดียวกันกับโค้ดที่ทดสอบ** — โดยทั่วไปเขียนไว้ที่ก้นไฟล์ `src/lib.rs` หรือ `src/xxx.rs` ที่มี logic
   นั้นอยู่ ครอบด้วย `#[cfg(test)] mod tests { ... }`
2. **compile เป็นส่วนหนึ่งของ crate เดียวกันกับโค้ดจริง** — เพราะ `mod tests` เป็นแค่ module หนึ่งในต้นไม้ module
   เดียวกัน (ตาม Part 16) ไม่ใช่ crate แยก
3. **มองเห็นทุกอย่างในไฟล์นั้น ไม่ว่าจะเป็น `pub` หรือ private** — เพราะ unit test module อยู่ "ภายใน" crate
   เดียวกัน กฎ visibility ของ Rust (Part 16) จึงยอมให้ sibling module มองเห็นกันได้ผ่าน `use super::*;`

คุณสมบัติข้อที่ 3 นี้คือทั้งจุดแข็งและข้อจำกัดของ unit test ในเวลาเดียวกัน จุดแข็งคือมันทำให้ unit test ทดสอบ
**รายละเอียดภายใน** ของ implementation ได้ตรง ๆ (เช่นฟังก์ชัน private ที่ทำหน้าที่ช่วยคำนวณ) แต่ข้อจำกัดที่ตามมา
คือ unit test **ไม่ได้พิสูจน์ว่า "คนนอก" ที่ import crate ของคุณไปใช้จริง จะเรียกใช้งานมันได้ถูกต้องหรือไม่** —
เพราะ unit test ไม่เคยผ่านขั้นตอนแบบเดียวกับผู้ใช้จริงเลย มันเข้าถึงทุกอย่างแบบ "ลัด" ผ่าน `super::*` โดยไม่ต้อง
เขียน `use my_crate::...;` แบบที่คนนอกต้องเขียนจริง ๆ

ลองนึกภาพสถานการณ์นี้: คุณเขียน library ที่มีฟังก์ชัน `pub fn calculate_price(...)` แต่ดันลืมเติม `pub` (เขียนแค่
`fn calculate_price(...)`) unit test ที่อยู่ใน `mod tests` ของไฟล์เดียวกันจะ**ยังคงเรียกมันได้ตามปกติ**และ test
ผ่านหมด เพราะ unit test ไม่สนใจ visibility ข้าม crate เลย — ผลคือ CI ของคุณเขียวหมดทุกอย่าง ทั้งที่ในความเป็นจริง
**ไม่มีใครสามารถเรียก `calculate_price` จากนอก crate ได้เลย** เพราะมันไม่ได้เป็น `pub` บั๊กแบบนี้ unit test ตรวจ
ไม่พบ เพราะ unit test ไม่ได้จำลองการเป็น "ผู้ใช้ภายนอก" คำถามคือ: เราจะเขียน test ที่บังคับให้ตัวมันเองมองเห็น
**เฉพาะ public API** เหมือนผู้ใช้จริงได้อย่างไร? นี่คือคำถามที่ integration test ถูกออกแบบมาตอบโดยตรง

### 33.2 นิยาม Integration Test: ทดสอบจากภายนอกเหมือนผู้ใช้จริง

**Integration test** คือ test ที่เขียนขึ้นเพื่อทดสอบ crate ของคุณ **"จากมุมมองของคนนอก"** อย่างแท้จริง — ไม่ใช่แค่
เปรียบเปรยแบบ "ทำตัวเหมือนคนนอก" แต่หมายถึง **มันคือคนละ compilation unit กันจริง ๆ ตามที่ `rustc` มองเห็น**

Cargo กำหนดธรรมเนียมนี้ไว้ชัดเจน:

> ไฟล์ `.rs` แต่ละไฟล์ที่วางอยู่**โดยตรง**ในโฟลเดอร์ `tests/` ที่ root ของ package จะถูก Cargo คอมไพล์เป็น
> **crate อิสระของตัวเอง** หนึ่งต้น โดยจะถูก **link เข้ากับ library crate ของ package โดยอัตโนมัติ** เหมือนกับ
> ที่ crate ภายนอกตัวหนึ่ง `use` library ของคุณผ่าน `Cargo.toml`

ประโยคนี้มีนัยสำคัญที่ต้องแยกออกมาอธิบายทีละส่วน เพราะมันเชื่อมกับสิ่งที่ Part 16 และ Part 17 สอนไปแล้วโดยตรง

#### "crate อิสระของตัวเอง" หมายความว่าอย่างไรจริง ๆ

จาก Part 17 หัวข้อ 17.2 คุณได้เห็นแล้วว่า `src/lib.rs` คือ crate root ของ module tree หนึ่งต้น และ `src/main.rs`
คือ crate root ของอีกต้นหนึ่งที่แยกจากกันสมบูรณ์ — ไฟล์ใน `tests/` ก็ทำงานตามกฎเดียวกันนี้เป๊ะ: **ไฟล์
`tests/api_tests.rs` หนึ่งไฟล์ คือ crate root ของ module tree อีกต้นหนึ่ง** ที่ไม่ได้แชร์ module tree เดียวกันกับ
`src/lib.rs` เลย มันเป็นแค่ crate ที่ **มี dependency ชี้กลับไปที่ library crate ของ package เดียวกัน**
โดยอัตโนมัติ (Cargo เซ็ตอัพ dependency นี้ให้เองโดยไม่ต้องเขียนอะไรใน `Cargo.toml`)

ผลที่ตามมาโดยตรงจากข้อเท็จจริงนี้คือ: **กฎ visibility ของ Rust (Part 16) จะบังคับใช้กับไฟล์ใน `tests/` แบบเดียวกัน
เป๊ะกับที่บังคับใช้กับ crate ภายนอกใด ๆ ที่มาใช้ library ของคุณ** — item ที่ไม่มี `pub` จะมองไม่เห็นเลยจากไฟล์ใน
`tests/` ไม่มีทางลัดผ่าน `super::` แบบที่ unit test ทำได้ เพราะไฟล์ใน `tests/` ไม่ได้เป็น module ลูกของ
`src/lib.rs` เลย มันอยู่คนละ crate กันโดยสิ้นเชิง

#### ทำไม integration test ต้องมี library crate เท่านั้น

นี่คือจุดที่เชื่อมกับ Part 17 หัวข้อ 17.2-17.3 อย่างตรงที่สุด: **"public API" ของ package หนึ่ง มีอยู่ได้ก็เพราะมี
library crate (`src/lib.rs`)** — ถ้า package ของคุณมีแค่ `src/main.rs` (binary crate ล้วน ๆ ไม่มี `src/lib.rs`
เลย) จะไม่มี "สิ่งที่ให้ crate อื่นมา `use`" อยู่จริง เพราะ binary crate ไม่มีแนวคิดของการถูก import โดย crate
อื่นเลยตั้งแต่ต้น (มันถูกออกแบบมาให้เป็น "โปรแกรมที่รันเอง" ไม่ใช่ "ไลบรารีให้คนอื่นเรียก")

ดังนั้นเมื่อไฟล์ใน `tests/` เขียน `use my_package::something;` มันกำลังพยายาม `use` **library crate** ของ package
(ชื่อ crate จะตรงกับชื่อ package เสมอ ตามที่ Part 17 อธิบาย) — ถ้า package นั้นไม่มี `src/lib.rs` เลย ก็ไม่มี
library crate ให้ `use` ตั้งแต่แรก การเขียน integration test แบบนี้จึงเป็นไปไม่ได้เลยในทางเทคนิค (จะพิสูจน์ด้วย
error จริงและวิธีแก้ในหัวข้อ 33.7)

นี่คือเหตุผลที่หัวข้อเป้าหมายของบทนี้เขียนไว้ตรง ๆ ว่า **"integration test มีความหมายก็ต่อเมื่อ package มี
library crate"** — มันไม่ใช่ "ข้อแนะนำ" แต่เป็นข้อจำกัดทางเทคนิคที่มาจากนิยามของ crate เอง

#### เปรียบเทียบ unit test กับ integration test แบบเห็นภาพชัด

| ประเด็น | Unit Test (Part 32) | Integration Test (บทนี้) |
|---|---|---|
| อยู่ที่ไหน | ในไฟล์เดียวกับ source code ใต้ `#[cfg(test)] mod tests` | ไฟล์แยกใต้โฟลเดอร์ `tests/` ที่ root ของ package |
| Compile เป็น | module ของ crate เดียวกันกับโค้ดจริง | crate **อิสระ** แยกจาก library crate โดยสิ้นเชิง |
| มองเห็นอะไร | ทุกอย่าง ทั้ง `pub` และ private (เหมือนอยู่ในบ้านเดียวกัน) | เฉพาะ `pub` item เท่านั้น (เหมือนแขกที่มาเยี่ยม) |
| ต้อง `#[cfg(test)]` ไหม | ต้อง (ไม่งั้นถูก compile เข้า release build ด้วย) | ไม่ต้อง (ทั้งไฟล์มีไว้เพื่อ test อย่างเดียว Cargo ไม่ compile `tests/` เข้า release build อยู่แล้ว) |
| ทดสอบอะไร | รายละเอียด implementation, ฟังก์ชัน private, edge case เล็ก ๆ | "สัญญา" (contract) ของ public API ว่าทำงานถูกต้องตามที่ผู้ใช้คาดหวัง |
| ต้องมี library crate ไหม | ไม่ต้อง (ทดสอบได้แม้ใน binary crate ล้วน ๆ) | **ต้องมี** เพราะมันคือการ `use` library crate จากภายนอก |

แถวสุดท้ายของตารางนี้คือจุดที่มือใหม่มักพลาด — unit test ใช้ได้กับทั้ง binary crate และ library crate เพราะมัน
เป็นแค่ module ภายในไฟล์เดียวกัน แต่ integration test ผูกติดกับการมี library crate โดยธรรมชาติของมัน

### 33.3 กฎของโฟลเดอร์ `tests/`: แต่ละไฟล์คือ Crate อิสระ

มาดูกฎที่ Cargo ใช้ตรวจจับไฟล์ integration test แบบละเอียดที่สุด:

> ไฟล์ `.rs` **ทุกไฟล์ที่วางอยู่ตรง ๆ** ใต้โฟลเดอร์ `tests/` (ไม่ใช่ไฟล์ที่อยู่ในโฟลเดอร์ย่อยของ `tests/`) จะถูก
> Cargo มองเป็น "integration test crate" หนึ่งตัวโดยอัตโนมัติ ชื่อ crate จะตรงกับชื่อไฟล์ (ไม่รวม `.rs`)

จุดที่ต้องเน้นคือคำว่า **"วางอยู่ตรง ๆ"** — ไฟล์ที่อยู่ **ใต้โฟลเดอร์ย่อย** ของ `tests/` (เช่น `tests/common/`)
จะ**ไม่ถูก**มองเป็น integration test crate แยกอัตโนมัติ นี่คือกฎที่หัวข้อ 33.6 จะใช้แก้ปัญหาเรื่อง helper code
ที่ใช้ร่วมกัน

โครงสร้างไดเรกทอรีมาตรฐานของ package ที่มีทั้ง unit test และ integration test หน้าตาแบบนี้:

```
my_package/
├── Cargo.toml
├── src/
│   └── lib.rs              <- library crate: มี #[cfg(test)] mod tests อยู่ข้างในสำหรับ unit test (Part 32)
└── tests/                   <- โฟลเดอร์ใหม่ที่บทนี้แนะนำ: อยู่ "ข้าง" src/ ที่ root ไม่ใช่ข้างใน src/
    ├── api_tests.rs         <- crate อิสระ #1: integration test ไฟล์ที่ 1
    └── edge_cases.rs        <- crate อิสระ #2: integration test ไฟล์ที่ 2
```

สังเกตว่า `tests/` อยู่**คนละระดับกับ `src/`** ทั้งคู่เป็น sibling folder ใต้ root ของ package (จุดเดียวกับที่มี
`Cargo.toml`) นี่เป็นจุดที่คนสับสนบ่อย — `tests/` **ไม่ได้**อยู่ใต้ `src/tests/` แบบที่บางภาษาโปรแกรมมิ่งอื่นทำ
(เช่น Java ที่มักมี `src/test/java/` คู่กับ `src/main/java/`) Rust/Cargo แยก `tests/` ออกมาเป็นโฟลเดอร์ระดับบน
สุดของ package โดยตรง

#### ทำไมไฟล์ใน `tests/` ไม่ต้องเขียน `#[cfg(test)]`

ใน Part 32 คุณเรียนว่า unit test module ต้องครอบด้วย `#[cfg(test)]` เพื่อไม่ให้ compiler รวมโค้ด test เข้าไปใน
release build จริง (ไม่งั้น binary จะพองไปด้วยโค้ด test ที่ผู้ใช้ปลายทางไม่ต้องการ) แต่ไฟล์ใน `tests/` **ไม่ต้อง**
เขียน `#[cfg(test)]` เลยแม้แต่บรรทัดเดียว เหตุผลคือ:

**Cargo รู้อยู่แล้วโดยธรรมชาติของโฟลเดอร์ `tests/` ว่าทั้งไฟล์มีไว้เพื่อการทดสอบเท่านั้น** — มันไม่ได้เป็นส่วนหนึ่ง
ของ library crate หรือ binary crate ที่จะถูก build ไปแจกจริงเลยตั้งแต่ต้น มันเป็น crate ที่ Cargo สร้างขึ้น**เฉพาะ
ตอนรัน `cargo test`** เท่านั้น (`cargo build` หรือ `cargo build --release` จะไม่แตะไฟล์ใน `tests/` เลยแม้แต่นิด
เดียว) ดังนั้นคำว่า `#[cfg(test)]` ที่ใช้ "บอก compiler ว่าให้ compile ชิ้นนี้เฉพาะตอน test" จึงไม่จำเป็น เพราะ
**ทั้ง crate (คือทั้งไฟล์) มีไว้เพื่อ test อยู่แล้วโดยตำแหน่งของมัน** ไม่ต้องมี attribute มากำกับซ้ำอีกชั้น

### 33.4 ตัวอย่างเต็มรูปแบบ: ต่อยอด `shop_pricing` จาก Part 17 ด้วย Integration Test

Part 17 หัวข้อ 17.3 สร้าง package ชื่อ `shop_pricing` ที่มี `src/lib.rs` เก็บฟังก์ชัน `calculate_discount` และมี
unit test อยู่แล้ว บทนี้จะ**ต่อยอด**ของเดิม โดยเพิ่มฟังก์ชัน public ใหม่ที่ซับซ้อนขึ้นอีกนิด แล้วค่อยเพิ่ม
integration test เข้าไป เพื่อให้เห็นภาพที่ต่อเนื่องกันจริง

#### ปัญหาที่ทำให้ต้องมีฟังก์ชัน private ช่วย: floating point ไม่แม่นยำ

ก่อนไปถึง integration test เราต้องเพิ่มฟังก์ชันใหม่ก่อน: `final_price_with_tax` ที่คำนวณราคาหลังหักส่วนลดแล้ว
บวกภาษีกลับเข้าไป ปัญหาคือถ้าคำนวณด้วย `f64` ตรง ๆ (ตามที่ Part 3 เตือนไว้แล้วเรื่องความไม่แม่นยำของ floating
point) ผลลัพธ์อาจไม่ใช่เลขกลม ๆ อย่างที่คาด:

```rust
fn main() {
    let noisy = 100.0_f64 * 0.07;
    println!("{noisy}"); // พิมพ์ 7.000000000000001 ไม่ใช่ 7 เป๊ะ ๆ
}
```

เพื่อแก้ปัญหานี้ เราจึงต้องมีฟังก์ชัน **private** ชื่อ `round_to_cents` ที่ปัดเศษให้เหลือ 2 ตำแหน่งทศนิยมเสมอ
ก่อนคืนค่าออกจาก public API — นี่คือตัวอย่างจริงของ "implementation detail" ที่ผู้ใช้ library ไม่จำเป็นต้องรู้
ว่ามันมีอยู่ด้วยซ้ำ (ผู้ใช้แค่ต้องรู้ว่าเรียก `final_price_with_tax` แล้วได้เลขที่ปัดเรียบร้อยกลับมา ไม่ต้องรู้ว่า
ข้างในมันจัดการความไม่แม่นยำของ float อย่างไร)

```rust
// src/lib.rs (ต่อยอดจาก Part 17: shop_pricing)

/// คำนวณราคาสุทธิหลังหักส่วนลด คืนค่าเป็น tuple (ส่วนลดที่หักไป, ราคาสุทธิ)
/// (ฟังก์ชันเดิมจาก Part 17 — ยังเป็น public API เหมือนเดิมทุกประการ ไม่ได้แก้อะไรเลย)
pub fn calculate_discount(price: f64, discount_percent: f64) -> (f64, f64) {
    let discount_amount = price * (discount_percent / 100.0);
    let final_price = price - discount_amount;
    (discount_amount, final_price)
}

/// ฟังก์ชัน public ตัวใหม่: คำนวณราคาสุทธิหลังหักส่วนลด แล้วบวกภาษีกลับเข้าไป
/// ปัดเศษให้เหลือ 2 ตำแหน่งทศนิยมเสมอ (หน่วยเป็นบาท) ก่อนคืนค่า
pub fn final_price_with_tax(price: f64, discount_percent: f64, tax_percent: f64) -> f64 {
    let (_discount, discounted_price) = calculate_discount(price, discount_percent);
    let taxed_price = discounted_price + discounted_price * (tax_percent / 100.0);
    round_to_cents(taxed_price)
}

// ฟังก์ชัน private: implementation detail ล้วน ๆ — ผู้ใช้ crate จากภายนอกไม่จำเป็นต้องรู้ว่า
// มันมีอยู่ด้วยซ้ำ มีหน้าที่แก้ปัญหาความไม่แม่นยำของ floating point ก่อนคืนค่าออกไป
fn round_to_cents(amount: f64) -> f64 {
    (amount * 100.0).round() / 100.0
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

    // unit test ของฟังก์ชัน private โดยตรง — เป็นไปได้เฉพาะเพราะ unit test module
    // นี้อยู่ "ภายใน" crate เดียวกัน (Part 32) ถ้าเป็น integration test ใน tests/
    // จะเรียก round_to_cents ตรง ๆ แบบนี้ไม่ได้เลย เพราะไม่มี pub (พิสูจน์ในหัวข้อกับดัก)
    #[test]
    fn round_to_cents_fixes_floating_point_noise() {
        let noisy = 100.0_f64 * 0.07;
        assert_ne!(noisy, 7.0); // ยืนยันว่าปัญหา floating point นี้มีอยู่จริง
        assert_eq!(round_to_cents(noisy), 7.0); // helper แก้ปัญหาให้
    }

    #[test]
    fn final_price_with_tax_rounds_correctly() {
        assert_eq!(final_price_with_tax(100.0, 0.0, 7.0), 107.0);
    }
}
```

สังเกตว่า unit test ทั้ง 4 ตัวนี้เขียนไว้ตาม pattern เดียวกับ Part 32 ทุกประการ — สิ่งที่ต่างจากเดิมคือตอนนี้
เรามีทั้งฟังก์ชัน `pub` (`calculate_discount`, `final_price_with_tax`) และฟังก์ชัน private (`round_to_cents`)
ที่ unit test มองเห็นได้ทั้งคู่แบบไม่มีข้อจำกัด

#### เพิ่ม `tests/api_tests.rs`: integration test ตัวแรก

ทีนี้มาเพิ่มไดเรกทอรี `tests/` เข้าไปในโปรเจกต์ (ยังไม่มีมาก่อนเลยจาก Part 17):

```
shop_pricing/
├── Cargo.toml
├── src/
│   └── lib.rs
└── tests/
    └── api_tests.rs      <- ไฟล์ใหม่ที่จะเพิ่มในหัวข้อนี้
```

```rust
// tests/api_tests.rs — integration test: มองเห็นเฉพาะ public API ของ shop_pricing เท่านั้น
// (เขียนเหมือนกับที่ผู้ใช้จริงที่เพิ่ม shop_pricing = "0.1" ใน Cargo.toml ของตัวเองจะเขียน)
use shop_pricing::{calculate_discount, final_price_with_tax};

#[test]
fn integration_calculate_discount() {
    let (discount, final_price) = calculate_discount(250.0, 15.0);
    assert_eq!(discount, 37.5);
    assert_eq!(final_price, 212.5);
}

#[test]
fn integration_final_price_with_tax() {
    let total = final_price_with_tax(100.0, 0.0, 7.0);
    assert_eq!(total, 107.0);
}

#[test]
fn integration_full_pipeline_with_discount_and_tax() {
    // ราคา 200 บาท ลด 10% (เหลือ 180) แล้วบวกภาษี 7% (180 * 1.07 = 192.6)
    let total = final_price_with_tax(200.0, 10.0, 7.0);
    assert_eq!(total, 192.6);
}
```

จุดที่ต้องสังเกตให้ชัด: บรรทัด `use shop_pricing::{calculate_discount, final_price_with_tax};` เขียนแบบเดียวกับ
ที่ Part 17 อธิบายไว้ว่าเวลา `use` library crate ของ package ตัวเอง (จาก main.rs หรือจาก `src/bin/`) ต้อง `use`
ด้วย**ชื่อ package** เหมือนเป็น crate ภายนอก — จากไฟล์ใน `tests/` ก็เขียนแบบเดียวกันเป๊ะ เพราะมันคือ crate ที่
แยกออกมาจริง ๆ ไม่ใช่แค่เปรียบเปรย

ลองรันทั้งชุดด้วย `cargo test` (ยังไม่ใส่ flag เพิ่มเติม):

```bash
cargo test
```

```
   Compiling shop_pricing v0.1.0 (/home/user/shop_pricing)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.33s
     Running unittests src/lib.rs (target/debug/deps/shop_pricing-b35da4fe9a819ef3)

running 4 tests
test tests::final_price_with_tax_rounds_correctly ... ok
test tests::fifteen_percent_discount ... ok
test tests::no_discount_keeps_price_same ... ok
test tests::round_to_cents_fixes_floating_point_noise ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running tests/api_tests.rs (target/debug/deps/api_tests-0bf89723bcbfc2a9)

running 3 tests
test integration_calculate_discount ... ok
test integration_final_price_with_tax ... ok
test integration_full_pipeline_with_discount_and_tax ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests shop_pricing

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

(ผลลัพธ์นี้ได้จากการรัน `cargo test` จริงกับโปรเจกต์ตัวอย่างนี้ ด้วย `cargo 1.94.1`)

มาแยกอ่านผลลัพธ์นี้ทีละส่วน เพราะมันคือกุญแจสำคัญของทั้งบท:

1. **`Compiling shop_pricing v0.1.0 ...`** — Cargo compile library crate ก่อนอันดับแรก (compile ครั้งเดียว)
2. **`Running unittests src/lib.rs (target/debug/deps/shop_pricing-...)`** — นี่คือ**crate ตัวที่ 1** ที่ถูกรัน:
   คือ library crate เดิม (`shop_pricing`) ที่ compile พร้อม `#[cfg(test)]` เปิดอยู่ (ตาม Part 32) ทำให้
   `mod tests` ที่ฝังอยู่ใน `src/lib.rs` ปรากฏออกมาเป็น test binary ตัวหนึ่ง สังเกตชื่อ hash แปลก ๆ ต่อท้าย
   (`-b35da4fe9a819ef3`) — เป็น hash ที่ Cargo ใช้แยกแยะ build artifact แต่ละตัวไม่ให้ชนกัน
3. **`Running tests/api_tests.rs (target/debug/deps/api_tests-...)`** — นี่คือ**crate ตัวที่ 2** แยกจากตัวแรก
   โดยสิ้นเชิง: มันคือไฟล์ `tests/api_tests.rs` ที่ถูก compile เป็น binary ของตัวเองชื่อ `api_tests` (ตรงกับชื่อ
   ไฟล์ ไม่รวม `.rs` ตามกฎหัวข้อ 33.3) แล้วรัน 3 test ที่อยู่ในไฟล์นั้น
4. **`Doc-tests shop_pricing`** — นี่คือ**การทดสอบชนิดที่สาม** (doc-test) ที่ `cargo test` รันให้อัตโนมัติด้วย
   แต่ตอนนี้แสดง `running 0 tests` เพราะเรายังไม่ได้เขียนตัวอย่างโค้ดใน doc comment เลย (จะพูดถึงในหัวข้อ 33.9)

นี่คือหลักฐานที่ชัดที่สุดว่า **`cargo test` เพียงคำสั่งเดียว รันทั้งสามชนิดของ test ให้ครบในรอบเดียว** และแต่ละ
บล็อก "Running ..." คือ**การรัน binary แยกกันคนละตัว** ไม่ใช่การรันภายใน process เดียวกันทั้งหมด — เพราะแต่ละ
บล็อกคือ crate ที่ compile แยกจากกันตามกฎที่อธิบายในหัวข้อ 33.2-33.3

### 33.5 รันเจาะจงไฟล์เดียว: `cargo test --test` และต้นทุนของการมีหลาย Crate

#### `cargo test --test <ชื่อไฟล์>`: รันแค่ไฟล์ integration test ไฟล์เดียว

เมื่อมี integration test หลายไฟล์ (หรือแค่อยากข้าม unit test/doc-test ไปเลยตอน debug) ใช้ flag `--test` ตามด้วย
ชื่อไฟล์ (ไม่รวม `.rs`):

```bash
cargo test --test api_tests
```

```
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.01s
     Running tests/api_tests.rs (target/debug/deps/api_tests-0bf89723bcbfc2a9)

running 3 tests
test integration_calculate_discount ... ok
test integration_final_price_with_tax ... ok
test integration_full_pipeline_with_discount_and_tax ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

สังเกตว่าคราวนี้**ไม่มี**ส่วน `Running unittests` และ `Doc-tests` เลย — `--test api_tests` บอก Cargo ว่า "รันแค่
crate ที่มาจากไฟล์ `tests/api_tests.rs` เท่านั้น ไม่ต้องรัน crate อื่น" ซึ่งสมเหตุสมผลตามที่อธิบายไว้: เพราะแต่ละ
บล็อกคือ binary แยกกัน คุณจึงเลือกรันแค่บางตัวได้อย่างอิสระ (เทียบกับ `cargo run --bin <ชื่อ>` ใน Part 17 หัวข้อ
17.4 ที่ใช้หลักการเดียวกันสำหรับเลือกรัน binary ตัวเดียวจากหลายตัวใน `src/bin/`)

ถ้าตั้งชื่อไฟล์ผิดหรือไฟล์นั้นไม่มีอยู่ Cargo จะบอกรายชื่อไฟล์ที่มีจริงให้ทันที:

```bash
cargo test --test does_not_exist
```

```
error: no test target named `does_not_exist` in default-run packages
help: available test targets:
    api_tests
```

**วิธีอ่าน error นี้**: ข้อความ `help:` ท้าย error บอกรายชื่อ integration test file ทั้งหมดที่ Cargo มองเห็นจริง
ใน `tests/` ของ package นี้ — มีประโยชน์มากเวลาโปรเจกต์มีไฟล์ integration test หลายไฟล์แล้วจำชื่อไม่แม่น

#### ต้นทุนที่มองไม่เห็น: การมีหลายไฟล์ใน `tests/` ทำให้ compile ช้าขึ้นจริง

นี่คือประเด็นเชิงปฏิบัติที่สำคัญมากแต่มือใหม่มักไม่รู้จนกว่าโปรเจกต์จะโตขึ้น: **ไฟล์ใน `tests/` แต่ละไฟล์คือ
crate อิสระ (ตามหัวข้อ 33.2-33.3) และแต่ละ crate ต้อง link เข้ากับ library crate ของคุณ "ใหม่ทุกครั้ง"**

ลองนึกภาพ package ที่มี library crate ขนาดใหญ่ (สมมติใช้เวลา compile 5 วินาที) แล้วมีไฟล์ integration test 20
ไฟล์ — แม้แต่ละไฟล์จะมีแค่ 1-2 test เล็ก ๆ **Cargo ก็ต้อง compile และ link library crate เข้ากับแต่ละไฟล์แยกกัน
20 รอบ** (แม้ตัว library เองจะถูก compile ครั้งเดียวและ cache ไว้ในรูป `.rlib` ที่ compile ซ้ำไม่ได้ก็ตาม แต่
**ขั้นตอน link** ยังต้องทำใหม่สำหรับทุก binary ที่เกิดจากไฟล์ `tests/` แต่ละไฟล์อยู่ดี) เมื่อรวมกับ overhead ของ
การเริ่ม process ใหม่สำหรับแต่ละ test binary (แต่ละบล็อก "Running tests/xxx.rs" คือการ spawn process ใหม่หนึ่ง
ตัว) จำนวนไฟล์ integration test ที่มากเกินไปจะทำให้เวลารวมของ `cargo test` ช้าขึ้นอย่างชัดเจน แม้จำนวน `#[test]`
function โดยรวมจะเท่าเดิมก็ตาม

**คำแนะนำเชิงปฏิบัติที่ใช้กันจริงในโปรเจกต์ขนาดใหญ่**: จัดกลุ่ม integration test ตาม **พื้นที่ของ public API ที่
ทดสอบ** (เช่น `tests/pricing_tests.rs`, `tests/inventory_tests.rs`) แทนที่จะสร้างไฟล์ใหม่ทุกครั้งที่เพิ่ม test
case ใหม่หนึ่งตัว — ยิ่งไฟล์น้อยและแต่ละไฟล์มี `#[test]` function หลายตัวรวมกัน ยิ่งลด overhead การ link/spawn
process ที่ไม่จำเป็นได้มาก จำนวน `#[test]` function ที่ได้จะเท่าเดิม แต่จำนวน "crate ที่ต้อง compile+link" จะ
น้อยลงมาก ซึ่งคือต้นทุนตัวจริงที่ทำให้ compile time ต่างกัน ไม่ใช่จำนวน test function

### 33.6 Helper Code ที่ใช้ร่วมกันหลายไฟล์: ธรรมเนียม `tests/common/mod.rs`

เมื่อมี integration test หลายไฟล์ มักมีโค้ดช่วยเทส (setup, fixture, ค่าตัวอย่าง) ที่อยากใช้ร่วมกันหลายไฟล์ —
คำถามคือจะเก็บโค้ดช่วยนี้ไว้ที่ไหนใน `tests/` โดยไม่ให้มันถูกมองเป็น "ไฟล์ test" อีกไฟล์หนึ่ง (เพราะมันไม่มี
`#[test]` function เลย เก็บไว้แค่ฟังก์ชันช่วย)

#### ลองแบบที่ดูเหมือนจะใช้ได้ก่อน: `tests/common.rs` แบบไฟล์แบน ๆ

```rust
// tests/common.rs — วิธีที่ "ดูเหมือนจะโอเค" แต่มีปัญหาแอบแฝง
#[allow(dead_code)]
pub fn setup_value() -> f64 {
    100.0
}
```

ปัญหาคือไฟล์นี้อยู่ **"ตรง ๆ" ใต้ `tests/`** ตามกฎหัวข้อ 33.3 — Cargo จะมองมันเป็น **integration test crate อีก
ตัวหนึ่งโดยอัตโนมัติ** ทั้งที่ในไฟล์นี้ไม่มี `#[test]` function เลยแม้แต่ตัวเดียว! ลองรัน `cargo test` ดู:

```
   Compiling shop_pricing v0.1.0 (/home/user/shop_pricing)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.08s
     Running unittests src/lib.rs (target/debug/deps/shop_pricing-33da8c4867b6f18d)

running 2 tests
test tests::add_with_tax_is_correct ... ok
test tests::calculate_tax_is_correct ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running tests/api_tests.rs (target/debug/deps/api_tests-36385261ec9e057d)

running 2 tests
test integration_add_with_tax ... ok
test integration_zero_tax ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running tests/common.rs (target/debug/deps/common-a43960e040436c2d)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests temp_lib
   ...
```

เห็นบล็อก **`Running tests/common.rs ... running 0 tests`** ไหม? นี่คือปัญหาจริง: Cargo compile
`tests/common.rs` เป็น **crate ที่ 3** แยกออกมาโดยไม่มีใครขอ แล้วรันมันในฐานะ test binary ที่ไม่มี test อยู่ข้าง
ในเลย — เสียเวลา compile+link ไปเปล่า ๆ (ตามที่หัวข้อ 33.5 อธิบาย) แถมยังทำให้ output ของ `cargo test` รกไป
ด้วยบล็อกที่ไม่มีความหมายอะไรเลย ยิ่งทีมงานเห็น "running 0 tests" บ่อย ๆ อาจชินและมองข้าม test ที่ควรมีจริงแต่
เขียนพลาดจนไม่ถูกรันไปด้วยก็เป็นไปได้

#### วิธีที่ถูกต้อง: ย้ายเข้าไปเป็น `tests/common/mod.rs`

Cargo มีข้อยกเว้นสำคัญข้อหนึ่งกับกฎในหัวข้อ 33.3: **ไฟล์ที่อยู่ใต้โฟลเดอร์ย่อยของ `tests/` จะไม่ถูกมองเป็น
integration test crate โดยอัตโนมัติ** — Cargo จะสแกนหา test crate เฉพาะไฟล์ที่อยู่ "ชั้นบนสุด" ของ `tests/`
เท่านั้น ไฟล์ที่ซ้อนอยู่ในโฟลเดอร์ย่อยจะถูก "มองข้าม" จากการสแกนหา test target โดยสิ้นเชิง มันจะถูก compile ก็
ต่อเมื่อมีไฟล์ top-level ไฟล์ใดไฟล์หนึ่งประกาศ `mod common;` เพื่อดึงมันเข้าไปเป็นส่วนหนึ่งของ crate นั้นเอง
(เหมือนกับที่ Part 16 สอนเรื่อง `mod` ประกาศดึง submodule เข้ามาในต้นไม้)

โครงสร้างไฟล์ที่ถูกต้อง:

```
shop_pricing/
├── Cargo.toml
├── src/
│   └── lib.rs
└── tests/
    ├── api_tests.rs         <- top-level: ถูกมองเป็น test crate (มี #[test] จริง)
    └── common/
        └── mod.rs            <- อยู่ใต้โฟลเดอร์ย่อย: ไม่ถูกมองเป็น test crate เอง
```

```rust
// tests/common/mod.rs — helper ที่ใช้ร่วมกันได้หลายไฟล์ ไม่ถูกมองเป็น test crate ของตัวเอง
#[allow(dead_code)]
pub fn setup_value() -> f64 {
    100.0
}
```

```rust
// tests/api_tests.rs — ประกาศ mod common; เพื่อดึงไฟล์ tests/common/mod.rs เข้ามาใช้
mod common;

use shop_pricing::final_price_with_tax;

#[test]
fn integration_add_with_tax() {
    let base = common::setup_value();
    let total = final_price_with_tax(base, 0.0, 50.0);
    assert_eq!(total, 150.0);
}
```

รัน `cargo test` อีกครั้งหลังย้ายไฟล์:

```
     Running unittests src/lib.rs (target/debug/deps/shop_pricing-33da8c4867b6f18d)

running 2 tests
test tests::add_with_tax_is_correct ... ok
test tests::calculate_tax_is_correct ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running tests/api_tests.rs (target/debug/deps/api_tests-36385261ec9e057d)

running 2 tests
test integration_zero_tax ... ok
test integration_add_with_tax ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests temp_lib
   ...
```

บล็อก `Running tests/common.rs` **หายไปโดยสมบูรณ์** — ยืนยันว่า `tests/common/mod.rs` ไม่ถูกมองเป็น test crate
อีกต่อไป มันถูก compile เป็นแค่ module ภายในของ crate `api_tests` เท่านั้น (ถูกดึงเข้ามาผ่าน `mod common;`)
เหมือนกับที่ module ธรรมดาถูกดึงเข้า crate เดียวกันตามที่ Part 16 สอน

#### ทำไมต้องเจาะจงว่า "`mod.rs`" ไม่ใช่ชื่ออื่น

คำถามที่ตามมาคือ: ทำไมต้องตั้งชื่อไฟล์ว่า `mod.rs` เท่านั้น ทำไมตั้งชื่อ `tests/common/helpers.rs` ไม่ได้?

คำตอบคือ **ตั้งชื่ออื่นก็ได้** ตราบใดที่มันอยู่ใต้โฟลเดอร์ย่อยของ `tests/` (ไม่ได้อยู่ตรง ๆ ใต้ `tests/`) มันจะ
ไม่ถูกมองเป็น test crate อัตโนมัติเสมอ — สิ่งที่สำคัญคือ**ตำแหน่ง** (อยู่ในโฟลเดอร์ย่อย) ไม่ใช่**ชื่อไฟล์**
ธรรมเนียมที่ใช้ชื่อ `mod.rs` มาจากรูปแบบการตั้งชื่อไฟล์โมดูลแบบเดิม (pre-2018 module file convention ที่ Part 16
อาจกล่าวถึงผ่าน ๆ) ที่ยังคงเป็นที่นิยมเฉพาะสำหรับกรณีนี้ เพราะมันสื่อความหมายชัดเจนว่า **"นี่คือไฟล์ที่ทำหน้าที่
เป็น root ของโมดูล `common` ทั้งโฟลเดอร์"** — คุณจะเขียน `tests/common/helpers.rs` แล้วประกาศ
`#[path = "common/helpers.rs"] mod common;` ก็ได้เหมือนกัน แต่จะยุ่งยากขึ้นโดยไม่จำเป็น ในทางปฏิบัติเกือบทุก
โปรเจกต์ Rust ใช้ชื่อ `tests/common/mod.rs` เป็นค่ามาตรฐาน เพราะมันคือชื่อที่สั้นที่สุดและสื่อความหมายได้ตรงที่สุด
สำหรับ "helper module ที่ไม่ใช่ test file"

### 33.7 Integration Test สำหรับ Binary Crate: ข้อจำกัดและวิธีแก้

หัวข้อ 33.2 บอกไว้แล้วว่า integration test ต้องพึ่งการมี library crate — มาดูว่าเกิดอะไรขึ้นจริงถ้าลองทำกับ
package ที่มีแค่ `src/main.rs` ล้วน ๆ

#### จำลองสถานการณ์: package binary-only

```
temp_bin/
├── Cargo.toml
├── src/
│   └── main.rs        <- มีแค่นี้ ไม่มี src/lib.rs เลย
└── tests/
    └── some_test.rs    <- พยายามเขียน integration test
```

```rust
// tests/some_test.rs
use temp_bin::add_with_tax;

#[test]
fn integration_add_with_tax() {
    assert_eq!(add_with_tax(100.0, 0.5), 150.0);
}
```

ลองรัน `cargo test`:

```bash
cargo test
```

```
   Compiling temp_bin v0.1.0 (/home/user/temp_bin)
error[E0432]: unresolved import `temp_bin`
 --> tests/some_test.rs:1:5
  |
1 | use temp_bin::add_with_tax;
  |     ^^^^^^^^ use of unresolved module or unlinked crate `temp_bin`
  |
  = help: if you wanted to use a crate named `temp_bin`, use `cargo add temp_bin` to add it to your Cargo.toml

For more information about this error, try `rustc --explain E0432`.
error: could not compile `temp_bin` (test "some_test") due to 1 previous error
```

ข้อความ error ตรงกับที่อธิบายไว้ทุกประการ: **`use of unresolved module or unlinked crate` temp_bin** — เพราะ
package `temp_bin` **ไม่มี library crate เลย** จึงไม่มีอะไรให้ `tests/some_test.rs` (ซึ่งเป็นคนละ crate)
`use` ได้ ข้อความ `help:` ที่แนะนำให้ `cargo add temp_bin` ยิ่งตอกย้ำว่า compiler มองว่านี่คือการพยายาม `use`
crate ภายนอกตัวหนึ่งที่ชื่อ `temp_bin` (ซึ่งไม่มีจริง ไม่ได้เกี่ยวกับ binary crate ที่ชื่อเดียวกันเลย เพราะ
binary crate ไม่ใช่สิ่งที่ crate อื่น `use` ได้ตามที่อธิบายในหัวข้อ 33.2)

#### วิธีแก้: ย้อนกลับไปใช้ Pattern จาก Part 17 หัวข้อ 17.3

นี่คือจุดที่ pattern "lib บาง ๆ + main บาง ๆ" จาก Part 17 หัวข้อ 17.3 มีประโยชน์อย่างชัดเจนที่สุด — ตอนนั้น
เหตุผลที่ให้ไว้คือ **testability และ reusability** บทนี้แสดงให้เห็น**รูปธรรม**ของคำว่า testability นั้น: ถ้า
logic ทั้งหมดถูกฝังอยู่ใน `main()` ตรง ๆ (ไม่มี `src/lib.rs`) การเขียน integration test เพื่อทดสอบ logic นั้นจาก
"ภายนอก" จะเป็นไปไม่ได้เลยในทางเทคนิค ไม่ใช่แค่ "ยากกว่า" แต่ "ทำไม่ได้จริง ๆ"

ขั้นตอนแก้ไข:

```rust
// src/lib.rs — ย้าย logic ทั้งหมดมาไว้ที่นี่ ทำให้มี library crate เกิดขึ้น
pub fn add_with_tax(price: f64, tax_percent: f64) -> f64 {
    price + price * (tax_percent / 100.0)
}
```

```rust
// src/main.rs — เหลือแค่เปลือกบาง ๆ เรียก library ของตัวเอง
use temp_bin::add_with_tax;

fn main() {
    let total = add_with_tax(100.0, 50.0);
    println!("ราคารวม: {total:.2} บาท");
}
```

`Cargo.toml` ไม่ต้องแก้อะไรเลย เพราะ Cargo ตรวจจับทั้ง `src/lib.rs` และ `src/main.rs` ให้อัตโนมัติตามที่ Part 17
อธิบาย (package หนึ่งมีทั้ง library crate 1 ตัวและ binary crate 1 ตัวพร้อมกันได้เสมอ) เมื่อมี `src/lib.rs` เกิด
ขึ้นแล้ว ไฟล์ `tests/some_test.rs` เดิมที่เขียน `use temp_bin::add_with_tax;` จะ **compile ผ่านทันที** โดยไม่
ต้องแก้โค้ด test แม้แต่บรรทัดเดียว เพราะตอนนี้ `temp_bin` (ชื่อ package = ชื่อ library crate) มีจริงแล้ว

```bash
cargo test
```

```
   Compiling temp_bin v0.1.0 (/home/user/temp_bin)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.15s
     Running unittests src/lib.rs (target/debug/deps/temp_bin-...)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running unittests src/main.rs (target/debug/deps/temp_bin-...)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running tests/some_test.rs (target/debug/deps/some_test-...)

running 1 test
test integration_add_with_tax ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

สังเกตว่าตอนนี้มี **สี่บล็อก** ไม่ใช่สาม: `Running unittests src/lib.rs`, `Running unittests src/main.rs` (เพราะ
`main.rs` ก็เป็น crate ของตัวเองที่ `cargo test` รันหาด้วย แม้จะยังไม่มี `#[test]` อยู่ข้างในเลยก็ตาม จึงแสดง
"running 0 tests"), และ `Running tests/some_test.rs` ที่ตอนนี้ compile ผ่านและรันได้จริงแล้ว — บทเรียนสำคัญของ
หัวข้อนี้คือ **การเพิ่ม `src/lib.rs` เข้าไปในโปรเจกต์ที่มีแต่ binary มาก่อน คือ "กุญแจ" เดียวที่ปลดล็อกความ
สามารถในการเขียน integration test ได้** — ไม่มีทางลัดอื่นเลย

### 33.8 Testing Pyramid แบบ Rust: อะไรควรเทสระดับไหน

เมื่อโปรเจกต์เริ่มมี test ทั้งสองชนิดปนกัน คำถามที่ตามมาคือ: **ควรมี test แต่ละชนิดในสัดส่วนเท่าไร และแบ่งงาน
กันอย่างไร** แนวคิดที่ใช้กันแพร่หลายในวงการซอฟต์แวร์ (ไม่ใช่เฉพาะ Rust) เรียกว่า **testing pyramid** — ภาพ
สามเหลี่ยมที่แบ่งเป็น 3 ชั้น จากฐานกว้างไปยอดแคบ:

```
        /\
       /  \      ยอด:   End-to-End Tests (นอกขอบเขตบทนี้ — ดู Part 95)
      /----\             จำนวนน้อยที่สุด, ช้าที่สุด, ครอบคลุมระบบทั้งหมดจริง ๆ
     /      \
    /--------\   กลาง:  Integration Tests (บทนี้)
   /          \          จำนวนปานกลาง, ทดสอบ "สัญญา" ของ public API
  /------------\
 /              \ ฐาน:   Unit Tests (Part 32)
/----------------\        จำนวนมากที่สุด, เร็วที่สุด, ทดสอบฟังก์ชัน/logic ย่อยแต่ละชิ้น
```

หลักการของรูปสามเหลี่ยมนี้คือ: **ยิ่งลงไปที่ฐาน test ควรมีจำนวนมากและรันเร็ว ยิ่งขึ้นไปที่ยอด test ควรมีจำนวน
น้อยลงแต่ครอบคลุมภาพกว้างมากขึ้น** เหตุผลที่เป็นเช่นนี้เกี่ยวข้องกับ trade-off สามอย่าง: **ความเร็ว**,
**ความเปราะบางต่อการเปลี่ยนแปลง (fragility)**, และ **ความมั่นใจที่ได้ (confidence)**

#### Unit Test (ฐาน): จำนวนมาก เร็ว ตรงจุด

Unit test ทดสอบฟังก์ชันเดี่ยว ๆ แบบแยกส่วน ไม่ต้องพึ่ง I/O หรือ setup ที่ซับซ้อน จึงรันได้เร็วมาก (ดูจาก
`finished in 0.00s` ในทุก transcript ของบทนี้) — ข้อดีคือเขียนง่าย รันเร็ว และถ้า fail จะรู้ทันทีว่า "ฟังก์ชันไหน
พังตรงไหน" (เพราะทดสอบเจาะจงมาก) ข้อเสียคือ unit test **ไม่รู้ว่าฟังก์ชันหลายตัวทำงานร่วมกันถูกต้องหรือไม่** —
สมมติ `round_to_cents` ทำงานถูกและ `calculate_discount` ทำงานถูกแยกกัน แต่ถ้ามีบั๊กที่จุดต่อระหว่างสองฟังก์ชันนี้
ใน `final_price_with_tax` (เช่นลำดับการเรียกผิด หรือส่ง parameter สลับตำแหน่งกัน) unit test ของแต่ละฟังก์ชันจะ
ยังผ่านหมด เพราะมันทดสอบแต่ละฟังก์ชันแยกกัน ไม่ได้ทดสอบ "การประกอบกัน"

**ควร unit test อะไร**: ฟังก์ชัน private ที่มี logic ซับซ้อน (เช่น `round_to_cents`, `apply_percent_discount`),
edge case ของแต่ละฟังก์ชันเดี่ยว ๆ (ค่า 0, ค่าติดลบ, ค่าขอบเขต), และฟังก์ชัน public ที่ตรงไปตรงมาไม่มีการเรียก
ฟังก์ชันอื่นซับซ้อน

#### Integration Test (กลาง): จำนวนปานกลาง ทดสอบ "สัญญา" ของ API

Integration test ทดสอบว่า public API ทั้งชุดทำงานร่วมกันถูกต้องจากมุมมองผู้ใช้จริง — มันจับบั๊กที่เกิดจาก "การ
ประกอบกัน" ของฟังก์ชันหลายตัวได้ (แบบตัวอย่าง `final_price_with_tax` ข้างต้น) และยังจับบั๊กเรื่อง **visibility**
ได้ด้วย (ลืมเติม `pub` ตามที่อธิบายในหัวข้อ 33.1) ซึ่ง unit test ทำไม่ได้เลย

**ควร integration test อะไร**: workflow ทั้งชุดที่ผู้ใช้จริงจะทำ (เช่น "สร้างตะกร้า → เพิ่มสินค้า → ใส่ discount
→ ดูยอดรวม" ในตัวอย่างหัวข้อ 33.11), การรวมกันของฟังก์ชัน public หลายตัว, และกรณีที่อยากยืนยันว่า public API
เสถียร (ไม่เปลี่ยนพฤติกรรมโดยไม่ตั้งใจตอน refactor ภายใน) — integration test ไม่ควรทดสอบ **รายละเอียดวิธีคำนวณ
ภายใน** ซ้ำกับที่ unit test ทดสอบไปแล้ว เพราะจะกลายเป็น test ที่ซ้ำซ้อนกัน (redundant) ทำให้ต้นทุนการดูแล
(maintenance cost) สูงขึ้นโดยไม่ได้ความมั่นใจเพิ่มขึ้นจริง

#### End-to-End Test (ยอด): นอกขอบเขตของบทนี้

ยอดสามเหลี่ยมคือ end-to-end test (E2E) — ทดสอบระบบทั้งระบบที่ทำงานจริง (เช่น รัน web server จริงแล้วยิง HTTP
request จริงเข้าไป, หรือควบคุม browser จริงให้คลิกปุ่มบนหน้าเว็บ) ซึ่งช้าที่สุดและซับซ้อนที่สุดในการ setup แต่ให้
ความมั่นใจสูงสุดว่า "ระบบทำงานได้จริงในสภาพแวดล้อมที่ใกล้เคียงการใช้งานจริงที่สุด" หลักสูตรนี้จะกลับมาพูดถึงเรื่อง
นี้อย่างเจาะจงใน **Part 95 (Testing Web Applications แบบครบวงจร)** เมื่อคุณมีพื้นฐานเรื่อง web server (`axum`)
และ concurrency มาเพียงพอแล้ว ตอนนี้ให้รู้จักไว้แค่ว่า "มันมีอยู่ และอยู่คนละระดับกับ unit/integration test"
ก็เพียงพอ

#### กฎคร่าว ๆ สำหรับโปรเจกต์ขนาดเล็กถึงกลาง

ในทางปฏิบัติสำหรับ library หรือแอปพลิเคชันขนาดเล็กถึงกลางที่หลักสูตรนี้เน้น แนวทางที่ใช้ได้ดีคือ:

1. เขียน unit test ให้ครอบคลุม **ฟังก์ชัน private ทุกตัวที่มี logic การคำนวณ/ตัดสินใจ** และ edge case ที่คิดได้
2. เขียน integration test สัก **1-3 ไฟล์ต่อ "กลุ่มความสามารถ" ของ public API** (ไม่ใช่ 1 ไฟล์ต่อ 1 ฟังก์ชัน)
   ครอบคลุม happy path หลักและ error case สำคัญ ๆ ของการใช้งานจริง
3. ถ้า integration test เริ่มต้องเซ็ตอัพข้อมูลซับซ้อนคล้ายกันซ้ำหลายที่ ให้แยกเป็น helper ใน
   `tests/common/mod.rs` (หัวข้อ 33.6) ก่อนที่จะปล่อยให้โค้ด setup ซ้ำซ้อนกันหลายสิบครั้ง

### 33.9 Doc-Tests: การทดสอบชนิดที่สามที่ซ่อนอยู่ใน Documentation

จาก transcript ในหัวข้อ 33.4 คุณเห็นบล็อก `Doc-tests shop_pricing` โผล่มาทุกครั้งที่รัน `cargo test` แล้ว — นี่
คือการทดสอบชนิดที่ **สาม** ที่ยังไม่ได้อธิบายเลย เกริ่นไว้ในบทนี้สั้น ๆ ก่อน เพราะรายละเอียดเต็มรูปแบบจะสอนใน
**Part 34: Documentation: rustdoc**

แนวคิดของมันคือ: ถ้าคุณเขียน doc comment (`///`) ที่มีบล็อกโค้ดตัวอย่างอยู่ข้างใน (คร่อมด้วย ` ``` `)
`cargo test` จะ**ดึงโค้ดตัวอย่างนั้นออกมา compile และรันจริง**เหมือนเป็น test อีกตัวหนึ่ง

```rust
/// คำนวณราคาสุทธิหลังหักส่วนลด แล้วบวกภาษีกลับเข้าไป ปัดเศษเป็น 2 ตำแหน่งทศนิยม
///
/// ```
/// let total = shop_pricing::final_price_with_tax(100.0, 0.0, 7.0);
/// assert_eq!(total, 107.0);
/// ```
pub fn final_price_with_tax(price: f64, discount_percent: f64, tax_percent: f64) -> f64 {
    // ...
    # 0.0
}
```

เมื่อเพิ่มตัวอย่างนี้เข้าไปแล้วรัน `cargo test` บล็อก doc-test จะเปลี่ยนจาก `running 0 tests` เป็น:

```
   Doc-tests shop_pricing

running 1 test
test src/lib.rs - final_price_with_tax (line 3) ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

ประโยชน์ที่ทรงพลังของมันคือแนวคิดที่เรียกกันว่า **"เอกสารโกหกไม่ได้เพราะมันถูกทดสอบจริง"** — ถ้าคุณแก้ signature
ของฟังก์ชันแล้วลืมอัปเดตตัวอย่างในเอกสาร (หรือถ้าพฤติกรรมของฟังก์ชันเปลี่ยนไปจนตัวอย่างเดิมให้ผลลัพธ์ผิด)
`cargo test` จะ**fail ทันที**ที่ doc-test ไม่ใช่แค่ปล่อยให้เอกสารที่ผิดหลุดไปถึงผู้ใช้จริงแบบเงียบ ๆ — เอกสาร
กลายเป็นสิ่งที่ต้อง "ผ่าน CI" เหมือน test อื่น ๆ ทุกประการ

**Part 34 จะสอนรายละเอียดเต็มรูปแบบ**: ไวยากรณ์การซ่อนบางบรรทัดจาก reader ด้วย `#` (ที่เห็นในตัวอย่างข้างบนที่
`# 0.0` ใช้ปิด placeholder ของ body จริงเพื่อให้ตัวอย่าง compile ได้แต่ไม่ต้องโชว์ใน docs.rs), การใช้
` ```ignore ` หรือ ` ```no_run ` เมื่อไม่อยากให้ตัวอย่างถูก compile/run จริง, และวิธีจัดการ doc-test ที่ควร
`panic` โดยตั้งใจ — ตอนนี้รู้จักไว้แค่ว่า **doc-test คือ test ชนิดที่สามที่ `cargo test` รันให้อัตโนมัติ** ก็
เพียงพอสำหรับบทนี้

### 33.10 เครื่องมือวัด Test Coverage: รู้จักไว้ก่อน (Awareness เท่านั้น)

คำถามที่มักตามมาหลังมี test หลายสิบตัวคือ: **"โค้ดของเราถูก test ครอบคลุมกี่เปอร์เซ็นต์?"** — คำตอบไม่สามารถ
ได้จากการอ่านโค้ด test ด้วยตาเปล่า ต้องใช้เครื่องมือวัด **test coverage** ที่รันโปรแกรมภายใต้การทดสอบ (instrumented
binary) แล้วบันทึกว่าบรรทัดไหน/branch ไหนถูกรันจริงบ้างระหว่าง test ทั้งหมด

เครื่องมือที่นิยมที่สุดในระบบนิเวศ Rust คือ **`cargo-tarpaulin`** (ติดตั้งผ่าน `cargo install cargo-tarpaulin`
แล้วรันด้วย `cargo tarpaulin`) ซึ่งจะรายงานเปอร์เซ็นต์ของบรรทัดโค้ดที่ถูกรันผ่านระหว่าง test suite ทั้งหมด
(ทั้ง unit และ integration test) พร้อมระบุบรรทัดที่ **ไม่เคย** ถูก test แตะเลย เป็นข้อมูลตั้งต้นที่ดีในการหาว่า
ควรเพิ่ม test ที่จุดไหนก่อน (นอกจาก `cargo-tarpaulin` ยังมี `cargo-llvm-cov` ที่ใช้ LLVM's source-based
coverage instrumentation โดยตรง ซึ่งบางทีมมองว่าให้ผลลัพธ์แม่นยำกว่าในบางสถานการณ์)

ข้อควรระวังสั้น ๆ ที่ควรรู้ไว้: **ตัวเลข coverage สูงไม่ได้แปลว่า test ดี** — โค้ดอาจถูก "รันผ่าน" (บรรทัดถูก
execute) แต่ผลลัพธ์ไม่ได้ถูกตรวจสอบด้วย `assert!` ที่มีความหมายจริงเลยก็ได้ (เช่นเรียกฟังก์ชันแต่ไม่เช็คค่าที่
คืนกลับมา) coverage เป็นแค่ **สัญญาณเตือนว่า "บรรทัดนี้ไม่มี test แตะเลยแน่ ๆ "** ไม่ใช่การันตีว่า "บรรทัดที่ถูก
แตะแล้วถูกทดสอบอย่างถูกต้อง" เครื่องมือเหล่านี้เป็น external tool ที่อยู่นอกขอบเขตเนื้อหาเชิงลึกของหลักสูตรนี้
บทนี้เพียงต้องการให้คุณรู้จักชื่อและแนวคิดของมันไว้เป็นข้อมูลพื้นฐาน

### 33.11 ตัวอย่างประยุกต์ครบวงจร: `shopping_cart` — สอง Layer ทำงานร่วมกัน

มาปิดท้ายบทนี้ด้วยตัวอย่างที่สมบูรณ์ที่สุด: library หนึ่งตัวที่มี**ทั้ง**ชั้น unit test (ทดสอบ implementation
detail ภายใน ตาม Part 32) **และ**ชั้น integration test (ทดสอบ public API จากภายนอก ตามที่บทนี้สอน) ทำงานเสริม
กันอย่างสมบูรณ์ ตามหลัก testing pyramid ในหัวข้อ 33.8

โครงสร้างโปรเจกต์:

```
shopping_cart/
├── Cargo.toml
├── src/
│   └── lib.rs                <- public API + unit tests ของ implementation detail
└── tests/
    └── integration_test.rs    <- integration tests ของ public API เท่านั้น
```

```toml
# Cargo.toml
[package]
name = "shopping_cart"
version = "0.1.0"
edition = "2021"

[dependencies]
```

#### `src/lib.rs`: Public API + Unit Tests

สังเกตว่าตะกร้าสินค้าเก็บราคาเป็นหน่วย **"สตางค์" (`u32`)** แทน `f64` โดยตั้งใจ — เพื่อหลีกเลี่ยงปัญหาความไม่
แม่นยำของ floating point ที่เห็นไปแล้วในหัวข้อ 33.4 (การคำนวณเงินด้วย integer หน่วยเล็กที่สุดเป็นวิธีที่นิยมใช้
ในระบบจริงที่เกี่ยวกับการเงิน เพื่อไม่ต้องพึ่ง `round_to_cents` เลยตั้งแต่ต้น)

```rust
//! shopping_cart: library ตัวอย่างสำหรับ Part 33 — สองชั้นของการทดสอบทำงานร่วมกัน
//!
//! เก็บราคาเป็นหน่วย "สตางค์" (cents, `u32`) เพื่อหลีกเลี่ยงปัญหาความแม่นยำของ floating point
//! ที่จะพบถ้าใช้ `f64` แทนเงินตราโดยตรง (ดูปัญหานี้แบบเต็มรูปแบบได้ในหัวข้อ 33.4)

/// สินค้าหนึ่งรายการในตะกร้า
#[derive(Debug, Clone)]
pub struct Item {
    pub name: String,
    pub unit_price_cents: u32,
    pub quantity: u32,
}

/// ตะกร้าสินค้า — public API ทั้งหมดของ crate นี้อยู่ที่ struct นี้
#[derive(Debug, Default)]
pub struct ShoppingCart {
    items: Vec<Item>,
}

impl ShoppingCart {
    /// สร้างตะกร้าเปล่า
    pub fn new() -> Self {
        ShoppingCart { items: Vec::new() }
    }

    /// เพิ่มสินค้าเข้าตะกร้า
    pub fn add_item(&mut self, name: impl Into<String>, unit_price_cents: u32, quantity: u32) {
        self.items.push(Item {
            name: name.into(),
            unit_price_cents,
            quantity,
        });
    }

    /// ลบสินค้าตามชื่อ (ลบตัวแรกที่ชื่อตรงกัน) คืนค่า true ถ้าลบสำเร็จ
    pub fn remove_item(&mut self, name: &str) -> bool {
        if let Some(pos) = self.items.iter().position(|item| item.name == name) {
            self.items.remove(pos);
            true
        } else {
            false
        }
    }

    /// จำนวนรายการ (ไม่ใช่จำนวนสินค้ารวม) ที่อยู่ในตะกร้า
    pub fn item_count(&self) -> usize {
        self.items.len()
    }

    /// ยอดรวมก่อนหักส่วนลด (หน่วยสตางค์)
    pub fn subtotal_cents(&self) -> u32 {
        self.items.iter().map(|item| line_total(item)).sum()
    }

    /// ยอดรวมหลังหักส่วนลดเป็นเปอร์เซ็นต์ (หน่วยสตางค์)
    pub fn total_after_discount_cents(&self, discount_percent: u32) -> u32 {
        apply_percent_discount(self.subtotal_cents(), discount_percent)
    }
}

// ฟังก์ชัน private: implementation detail ของการคำนวณ — ไม่มี pub
// จึงเป็น "ของภายใน" ที่ integration test (crate แยก) มองไม่เห็นเลย
// มีแต่ unit test (อยู่ใน crate เดียวกัน) เท่านั้นที่เรียกมันตรง ๆ ได้

fn line_total(item: &Item) -> u32 {
    item.unit_price_cents * item.quantity
}

fn apply_percent_discount(amount_cents: u32, percent: u32) -> u32 {
    amount_cents - (amount_cents * percent / 100)
}

#[cfg(test)]
mod tests {
    use super::*;

    // --- unit tests ของฟังก์ชัน private: ทดสอบ "รายละเอียดการคำนวณ" ---

    #[test]
    fn line_total_multiplies_price_by_quantity() {
        let item = Item {
            name: "ปากกา".to_string(),
            unit_price_cents: 1500,
            quantity: 3,
        };
        assert_eq!(line_total(&item), 4500);
    }

    #[test]
    fn apply_percent_discount_ten_percent() {
        assert_eq!(apply_percent_discount(10_000, 10), 9_000);
    }

    #[test]
    fn apply_percent_discount_zero_percent_unchanged() {
        assert_eq!(apply_percent_discount(10_000, 0), 10_000);
    }

    // --- unit tests ของ public API ก็เขียนที่นี่ได้เหมือนกัน (Part 32) ---

    #[test]
    fn empty_cart_has_zero_subtotal() {
        let cart = ShoppingCart::new();
        assert_eq!(cart.subtotal_cents(), 0);
        assert_eq!(cart.item_count(), 0);
    }
}
```

สังเกตการแบ่งหน้าที่ของ unit test ทั้ง 4 ตัวนี้: สามตัวแรก (`line_total_multiplies_price_by_quantity`,
`apply_percent_discount_ten_percent`, `apply_percent_discount_zero_percent_unchanged`) ทดสอบ **ฟังก์ชัน
private** ตรง ๆ — สิ่งที่ integration test ทำไม่ได้เลยเพราะไม่มี `pub` ส่วนตัวที่สี่
(`empty_cart_has_zero_subtotal`) ทดสอบ public API แต่เป็น edge case เล็ก ๆ (ตะกร้าเปล่า) ที่เหมาะเป็น unit test
มากกว่า integration test เพราะไม่ต้องพึ่ง workflow ที่ซับซ้อน

#### `tests/integration_test.rs`: ทดสอบเฉพาะ Public API

```rust
// tests/integration_test.rs
// Integration test: มองเห็นเฉพาะ public API ของ shopping_cart เท่านั้น
// (ทดสอบเหมือนผู้ใช้ crate นี้จริง ๆ ที่เขียน shopping_cart = "0.1" ใน Cargo.toml ของตัวเอง)

use shopping_cart::ShoppingCart;

#[test]
fn cart_calculates_subtotal_across_multiple_items() {
    let mut cart = ShoppingCart::new();
    cart.add_item("ปากกา", 1500, 3); // 45.00 บาท
    cart.add_item("สมุด", 4000, 2); // 80.00 บาท

    assert_eq!(cart.item_count(), 2);
    assert_eq!(cart.subtotal_cents(), 12_500); // 125.00 บาท
}

#[test]
fn removing_item_reduces_subtotal() {
    let mut cart = ShoppingCart::new();
    cart.add_item("ปากกา", 1500, 3);
    cart.add_item("สมุด", 4000, 2);

    let removed = cart.remove_item("ปากกา");

    assert!(removed);
    assert_eq!(cart.item_count(), 1);
    assert_eq!(cart.subtotal_cents(), 8_000);
}

#[test]
fn removing_missing_item_returns_false() {
    let mut cart = ShoppingCart::new();
    cart.add_item("ปากกา", 1500, 1);

    let removed = cart.remove_item("ยางลบ");

    assert!(!removed);
    assert_eq!(cart.item_count(), 1);
}

#[test]
fn discount_is_applied_to_whole_cart() {
    let mut cart = ShoppingCart::new();
    cart.add_item("หนังสือ", 20_000, 1); // 200.00 บาท

    let total = cart.total_after_discount_cents(25); // ลด 25%

    assert_eq!(total, 15_000); // 150.00 บาท
}
```

สังเกตว่า integration test ทั้ง 4 ตัวนี้ **ไม่มีตัวไหนแตะ `line_total` หรือ `apply_percent_discount` เลยแม้แต่
บรรทัดเดียว** — มันเรียกผ่าน `ShoppingCart` (public API) เท่านั้น และตรวจสอบ **ผลลัพธ์ที่ผู้ใช้จริงจะเห็น** เช่น
"เพิ่มสินค้า 2 รายการแล้วยอดรวมถูกไหม", "ลบสินค้าแล้วยอดรวมลดลงถูกไหม", "ใส่ discount แล้วยอดสุทธิถูกไหม" — นี่
คือความต่างเชิงปฏิบัติที่ชัดที่สุดระหว่างสองชั้นของการทดสอบ: unit test ตอบคำถาม **"ฟังก์ชันเดี่ยว ๆ คำนวณถูกไหม"**
ส่วน integration test ตอบคำถาม **"ผู้ใช้จะได้ผลลัพธ์ที่ถูกต้องไหมเมื่อใช้งานจริงผ่าน API สาธารณะ"**

รัน `cargo test` กับโปรเจกต์เต็มรูปแบบนี้:

```bash
cargo test
```

```
   Compiling shopping_cart v0.1.0 (/home/user/shopping_cart)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.27s
     Running unittests src/lib.rs (target/debug/deps/shopping_cart-197128f3528586e4)

running 4 tests
test tests::apply_percent_discount_ten_percent ... ok
test tests::apply_percent_discount_zero_percent_unchanged ... ok
test tests::empty_cart_has_zero_subtotal ... ok
test tests::line_total_multiplies_price_by_quantity ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running tests/integration_test.rs (target/debug/deps/integration_test-e1b0ea0c92cd6840)

running 4 tests
test cart_calculates_subtotal_across_multiple_items ... ok
test discount_is_applied_to_whole_cart ... ok
test removing_missing_item_returns_false ... ok
test removing_item_reduces_subtotal ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests shopping_cart

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

(ผลลัพธ์นี้ได้จากการรัน `cargo test` จริงกับโปรเจกต์ตัวอย่างนี้ — ทั้ง 4 unit test และ 4 integration test ผ่าน
หมด รวม 8 test ทั้งโปรเจกต์)

ผลลัพธ์นี้คือภาพสรุปของทั้งบท: **crate เดียวกัน (`shopping_cart`) ถูกทดสอบจากสองมุมมองที่ต่างกันโดยสิ้นเชิง** —
มุมมองจาก "ภายใน" (unit test เห็นฟังก์ชัน private ทุกตัว) และมุมมองจาก "ภายนอก" (integration test เห็นแค่
`ShoppingCart` และ `Item` ที่เป็น `pub`) ทั้งสองชั้นไม่ได้ซ้ำซ้อนกัน แต่ **เสริมกัน**: ถ้าวันหนึ่งมีคนแก้สูตร
คำนวณใน `apply_percent_discount` ผิด unit test จะจับได้ทันทีอย่างเจาะจง (บอกชัดว่าฟังก์ชันไหนพัง) และถ้ามีคน
เปลี่ยน `total_after_discount_cents` ให้เรียกฟังก์ชัน private ผิดลำดับ (แม้แต่ละฟังก์ชันย่อยจะยังถูกต้องแยกกัน)
integration test จะจับได้ (เพราะผลลัพธ์ปลายทางที่ผู้ใช้เห็นผิดไปจากที่คาด) — นี่คือเหตุผลที่ตำราการทดสอบซอฟต์แวร์
ทุกเล่มแนะนำให้มีทั้งสองชั้นควบคู่กัน ไม่ใช่เลือกอย่างใดอย่างหนึ่ง

### 33.12 การใช้ Mocking (จาก Part 32) ข้าม Crate Boundary ใน Integration Test

Part 32 สอนแนวคิด mocking เบื้องต้นด้วย trait: ออกแบบ dependency ที่อยากแทนที่ได้ตอน test (เช่น แหล่งข้อมูลราคา
สินค้าที่ปกติต้องเรียก API ภายนอก) ให้อยู่หลัง trait แล้วสร้าง mock struct ที่ implement trait นั้นสำหรับใช้ตอน
test แทนของจริง คำถามที่น่าสนใจคือ: **เทคนิคนี้ยังใช้ได้ไหมถ้าเราย้ายมาเขียนเป็น integration test ที่อยู่คนละ
crate กับ trait นั้น?**

คำตอบคือ **ใช้ได้เต็มรูปแบบ แต่มีเงื่อนไขหนึ่งข้อที่สำคัญมาก: trait ที่จะให้ mock มา implement ต้องเป็น `pub`**
เพราะการ `impl SomeTrait for MyMock` จากไฟล์ใน `tests/` ก็คือการเรียกใช้ชื่อ `SomeTrait` ข้าม crate เหมือนกับ
การเรียกฟังก์ชันใด ๆ — กฎ visibility เดียวกันจากหัวข้อ 33.2 บังคับใช้เป๊ะ ๆ

มาต่อยอด `shopping_cart` อีกครั้ง: เพิ่ม trait `PriceSource` สำหรับดึงราคาสินค้าจากแหล่งข้อมูลภายนอก (จำลอง
สถานการณ์ที่ราคาจริงต้องมาจาก API หรือฐานข้อมูล ไม่ใช่ค่าที่ผู้ใช้กำหนดตรง ๆ):

```rust
// src/lib.rs (เพิ่มเติมจากหัวข้อ 33.11)

/// trait สำหรับแหล่งข้อมูลราคาสินค้า — ออกแบบให้ฉีด dependency ได้ตาม Part 32
/// ต้องเป็น pub เสมอ ไม่งั้น integration test (คนละ crate) จะ implement มันเองไม่ได้เลย
pub trait PriceSource {
    fn price_for(&self, name: &str) -> Option<u32>;
}

impl ShoppingCart {
    // ... (method เดิมทั้งหมดจากหัวข้อ 33.11 ยังอยู่เหมือนเดิม)

    /// เพิ่มสินค้าโดยดึงราคาจาก PriceSource ภายนอก (dependency injection ตาม Part 32)
    /// คืนค่า true ถ้าหาราคาเจอและเพิ่มสำเร็จ, false ถ้า source ไม่มีราคาสินค้านี้
    pub fn add_item_from_source(
        &mut self,
        name: &str,
        quantity: u32,
        source: &dyn PriceSource,
    ) -> bool {
        match source.price_for(name) {
            Some(price_cents) => {
                self.add_item(name, price_cents, quantity);
                true
            }
            None => false,
        }
    }
}
```

จากนั้นใน integration test สร้าง mock ของ `PriceSource` ขึ้นมาเอง:

```rust
// tests/integration_test.rs (เพิ่มเติมจากหัวข้อ 33.11)
use shopping_cart::{PriceSource, ShoppingCart};

// mock ที่ implement trait PriceSource (public) ของ shopping_cart — struct นี้เองไม่ต้องเป็น pub
// เพราะถูกประกาศและใช้อยู่ภายใน crate ของไฟล์นี้เท่านั้น ไม่มีใครนอก crate นี้ต้องมองเห็นมัน
struct FixedPriceSource;

impl PriceSource for FixedPriceSource {
    fn price_for(&self, name: &str) -> Option<u32> {
        if name == "ของพิเศษ" {
            Some(5000)
        } else {
            None
        }
    }
}

#[test]
fn add_item_from_source_uses_mock_price() {
    let mut cart = ShoppingCart::new();
    let source = FixedPriceSource;

    let added = cart.add_item_from_source("ของพิเศษ", 2, &source);

    assert!(added);
    assert_eq!(cart.subtotal_cents(), 10_000);
}

#[test]
fn add_item_from_source_fails_when_price_unknown() {
    let mut cart = ShoppingCart::new();
    let source = FixedPriceSource;

    let added = cart.add_item_from_source("ของที่ไม่รู้จัก", 1, &source);

    assert!(!added);
    assert_eq!(cart.item_count(), 0);
}
```

จุดที่ควรสังเกตให้ชัดคือ **`FixedPriceSource` เองไม่จำเป็นต้องเป็น `pub` เลย** — มันถูกประกาศและใช้อยู่ภายใน
ไฟล์ `tests/integration_test.rs` เท่านั้น (ซึ่งคือ crate ของตัวมันเอง) ไม่มีโค้ดที่อื่นนอก crate นี้ต้องอ้างถึง
มัน สิ่งที่ต้องเป็น `pub` มีแค่ **`PriceSource` (trait) และ `price_for` (method ของ trait)** เพราะสองอย่างนี้คือ
สิ่งที่ crate `integration_test` (คนละ crate กับ `shopping_cart`) ต้องอ้างถึงเพื่อเขียน `impl PriceSource for
FixedPriceSource` — นี่คือตัวอย่างที่ชัดเจนของการนำกฎ visibility จากหัวข้อ 33.2 มาปฏิบัติจริงร่วมกับเทคนิค
mocking จาก Part 32: **ทุกอย่างที่ mock ต้อง "แตะ" ข้าม crate ต้องเป็น `pub` แต่ตัว mock เองไม่ต้อง**

รัน `cargo test` แล้วจะเห็น integration test เพิ่มขึ้นเป็น 6 ตัว (จากเดิม 4 ตัวในหัวข้อ 33.11):

```
     Running tests/integration_test.rs (target/debug/deps/integration_test-e1b0ea0c92cd6840)

running 6 tests
test add_item_from_source_fails_when_price_unknown ... ok
test cart_calculates_subtotal_across_multiple_items ... ok
test discount_is_applied_to_whole_cart ... ok
test add_item_from_source_uses_mock_price ... ok
test removing_item_reduces_subtotal ... ok
test removing_missing_item_returns_false ... ok

test result: ok. 6 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## กับดักที่พบบ่อย (Common Pitfalls)

**1. พยายามเรียกฟังก์ชัน private จากไฟล์ใน `tests/` — compile error E0603**

มือใหม่ที่คุ้นกับการเรียกอะไรก็ได้แบบไม่มีข้อจำกัดจาก unit test (Part 32) มักเผลอเขียน integration test ที่
พยายาม `use` ฟังก์ชัน private ตรง ๆ:

```rust
// tests/broken_test.rs
use shop_pricing::round_to_cents; // round_to_cents ไม่มี pub!

#[test]
fn tries_to_reach_private_helper() {
    assert_eq!(round_to_cents(7.004), 7.0);
}
```

```
error[E0603]: function `round_to_cents` is private
  --> tests/broken_test.rs:1:19
   |
 1 | use shop_pricing::round_to_cents;
   |                   ^^^^^^^^^^^^^^ private function
   |
note: the function `round_to_cents` is defined here
  --> src/lib.rs:19:1
   |
19 | fn round_to_cents(amount: f64) -> f64 {
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

For more information about this error, try `rustc --explain E0603`.
```

**วิธีแก้**: นี่ไม่ใช่บั๊กของ Cargo หรือ compiler แต่เป็น**การทำงานที่ถูกต้องตามออกแบบ** — ถ้าฟังก์ชันนั้นเป็น
implementation detail จริง ๆ ให้ทดสอบมันด้วย **unit test** (อยู่ใน `#[cfg(test)] mod tests` ข้างในไฟล์เดียวกับ
ฟังก์ชันนั้น ตาม Part 32) ไม่ใช่ integration test ถ้าคุณคิดว่าจำเป็นต้องทดสอบมันจาก "ภายนอก" จริง ๆ ให้ถามตัวเอง
ว่าควรทำให้มันเป็น `pub` หรือไม่ — ถ้าใช่ก็เติม `pub` เข้าไป (แล้วมันจะกลายเป็นส่วนหนึ่งของ public API ที่ต้อง
รักษาความเสถียรต่อไป) ถ้าไม่ใช่ก็ยอมรับว่ามันคือรายละเอียดภายในที่ควรอยู่หลัง unit test เท่านั้น

**2. สร้าง `tests/common.rs` แบบไฟล์แบน ๆ แทน `tests/common/mod.rs` — เกิด test crate เปล่าที่ไม่มีใครขอ**

```
     Running tests/common.rs (target/debug/deps/common-a43960e040436c2d)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**วิธีแก้**: ตามที่อธิบายในหัวข้อ 33.6 — ย้ายไฟล์ helper ไปไว้ใต้โฟลเดอร์ย่อย เช่น `tests/common/mod.rs` แล้ว
ประกาศ `mod common;` ในไฟล์ integration test ที่ต้องใช้มัน กฎจำง่าย ๆ คือ: **ถ้าไฟล์ไม่มี `#[test]` function
อยู่ข้างในเลย ไฟล์นั้นไม่ควรอยู่ "ตรง ๆ" ใต้ `tests/`** ให้ย้ายมันเข้าโฟลเดอร์ย่อยเสมอ

**3. เขียน integration test ให้กับ package ที่มีแต่ `src/main.rs` (binary-only) — E0432**

```
error[E0432]: unresolved import `temp_bin`
 --> tests/some_test.rs:1:5
  |
1 | use temp_bin::add_with_tax;
  |     ^^^^^^^^ use of unresolved module or unlinked crate `temp_bin`
  |
  = help: if you wanted to use a crate named `temp_bin`, use `cargo add temp_bin` to add it to your Cargo.toml
```

**วิธีแก้**: ตามที่อธิบายในหัวข้อ 33.7 — สร้าง `src/lib.rs` แล้วย้าย logic ทั้งหมดที่อยากทดสอบไปไว้ที่นั่น ให้
`src/main.rs` เหลือแค่หน้าที่เรียก library crate ของตัวเองแบบเปลือกบาง ๆ (ตาม Part 17 หัวข้อ 17.3) ข้อความ
`help:` ที่แนะนำ `cargo add` เป็นสัญญาณที่ทำให้เข้าใจผิดได้ง่าย เพราะ compiler มองว่านี่คือการหา crate ภายนอกที่
ไม่มีจริง ไม่ได้เกี่ยวกับการที่ binary crate ชื่อเดียวกันมีอยู่แล้ว — สิ่งที่ต้องทำไม่ใช่ `cargo add` แต่คือสร้าง
library crate ขึ้นมาในโปรเจกต์เดียวกัน

**4. เจาะจงไฟล์ผิดชื่อด้วย `cargo test --test <ชื่อ>`**

```bash
cargo test --test does_not_exist
```

```
error: no test target named `does_not_exist` in default-run packages
help: available test targets:
    api_tests
```

**วิธีแก้**: ชื่อที่ตามหลัง `--test` ต้องตรงกับชื่อไฟล์ (ไม่รวม `.rs`) ที่วางอยู่ตรง ๆ ใต้ `tests/` เป๊ะ ๆ
(case-sensitive) ถ้าจำชื่อไม่แม่น ให้ดูจากข้อความ `help:` ที่ error แสดงรายชื่อไฟล์ทั้งหมดที่มีจริงให้เลย หรือใช้
`cargo test` เปล่า ๆ (ไม่ใส่ `--test`) แล้วดูบรรทัด `Running tests/...` จาก output เพื่อยืนยันชื่อไฟล์ที่ถูกต้อง

**5. เผลอใส่ dependency ที่ใช้เฉพาะใน `tests/` ไว้ใต้ `[dependencies]` แทน `[dev-dependencies]`**

สมมติมี crate ช่วยสร้างข้อมูลจำลอง (fake/mock data) ชื่อ `fake_data` ที่ใช้แค่ในไฟล์ integration test เท่านั้น
ไม่มีที่ไหนใน `src/` เรียกใช้มันเลย แต่ถ้าใส่ผิดที่ใต้ `[dependencies]`:

```toml
[dependencies]
fake_data = { path = "../fake_data" }
```

```bash
cargo tree
```

```
shop_pricing v0.1.0 (/home/user/shop_pricing)
└── fake_data v0.1.0 (/home/user/fake_data)
```

`fake_data` จะถูกดึงเข้ามาเป็น dependency ของ **library/binary crate จริง** ด้วย ทำให้มันถูก compile เข้าไปใน
`cargo build --release` (release build ที่จะแจกให้ผู้ใช้จริง) โดยไม่จำเป็น ทั้งที่มันมีไว้ช่วย test เท่านั้น

**วิธีแก้**: ย้ายไปไว้ใต้ `[dev-dependencies]` แทน (ธรรมเนียมเดียวกับที่ Part 17 หัวข้อ 17.13 อธิบายไว้ว่า
`[dev-dependencies]` ใช้กับ `cargo test`/`cargo bench`/`examples/` เท่านั้น ไม่ถูกรวมเข้า `cargo build` ปกติ):

```toml
[dependencies]

[dev-dependencies]
fake_data = { path = "../fake_data" }
```

ตรวจสอบด้วย `cargo tree -e normal` (เฉพาะ dependency ปกติ ไม่รวม dev-dependencies):

```bash
cargo tree -e normal
```

```
shop_pricing v0.1.0 (/home/user/shop_pricing)
```

ไม่มี `fake_data` ปรากฏเลย ยืนยันว่ามันจะไม่ถูกดึงเข้า release build อีกต่อไป — แต่ยังใช้ได้ปกติเวลารัน
`cargo test` (เพราะ `cargo test` จะรวม `[dev-dependencies]` เข้ามาเสมอ) ตรวจสอบด้วย
`cargo tree -e normal,dev` เพื่อดูว่ามันยังอยู่ในฐานะ dev-dependency:

```bash
cargo tree -e normal,dev
```

```
shop_pricing v0.1.0 (/home/user/shop_pricing)
[dev-dependencies]
└── fake_data v0.1.0 (/home/user/fake_data)
```

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** สร้าง package ใหม่ชื่อ `calc_tool` ด้วย `cargo new calc_tool --lib` เขียนฟังก์ชัน public 4 ตัวใน
   `src/lib.rs`: `add`, `subtract`, `multiply`, `divide` (รับ `f64` สองตัว คืนค่า `f64`) จากนั้นสร้างไดเรกทอรี
   `tests/` ที่ root ของ package (ไม่ใช่ใต้ `src/`) และเพิ่มไฟล์ `tests/calc_tests.rs` ที่มี `#[test]` function
   อย่างน้อย 4 ตัว ทดสอบทั้ง 4 ฟังก์ชันผ่าน `use calc_tool::{add, subtract, multiply, divide};` รัน `cargo test`
   แล้วสังเกตว่ามีบล็อก `Running unittests` และ `Running tests/calc_tests.rs` แยกกันสองบล็อกจริงหรือไม่ (hint:
   ถ้ายังไม่มี unit test ใน `src/lib.rs` เลย บล็อก `Running unittests` จะแสดง "running 0 tests" ก็ยังนับว่าถูก
   ต้อง เพราะ Cargo รัน crate นั้นอยู่ดีแม้จะไม่มี test อยู่ข้างในเลย)

2. **[กลาง]** ต่อยอดจาก `shop_pricing` ในหัวข้อ 33.4 ของบทนี้: เพิ่มฟังก์ชัน private ใหม่ชื่อ `clamp_percent`
   (รับ `f64` แทนค่าเปอร์เซ็นต์ ถ้าน้อยกว่า 0 ให้คืน 0.0 ถ้ามากกว่า 100 ให้คืน 100.0 ไม่งั้นคืนค่าเดิม) แล้วเรียก
   ใช้ใน `calculate_discount` เพื่อป้องกัน discount percent ที่ผิดปกติ (เช่น -10% หรือ 150%) เขียน unit test
   ทดสอบ `clamp_percent` ตรง ๆ ทั้ง 3 กรณี (ต่ำกว่าขอบ, สูงกว่าขอบ, อยู่ในขอบ) จากนั้นสร้าง `tests/common/mod.rs`
   ที่มีฟังก์ชัน `pub fn assert_close_enough(a: f64, b: f64)` (เช็คว่าค่าห่างกันไม่เกิน `0.001`) แล้วใช้มันในไฟล์
   `tests/api_tests.rs` แทน `assert_eq!` ตรง ๆ ทดสอบว่า `cargo test` ยังไม่มีบล็อก `Running tests/common.rs`
   โผล่มาแยก (hint: ทวนหัวข้อ 33.6 เรื่องตำแหน่งของไฟล์ที่ Cargo จะมองเป็น test crate หรือไม่)

3. **[กลาง-ยาก]** สร้าง package binary-only ชื่อ `todo_cli` ด้วย `cargo new todo_cli` (ไม่ใส่ `--lib`) เขียน
   โปรแกรมจำลองการจัดการ to-do list ทั้งหมดไว้ใน `main()` ตรง ๆ ก่อน (เก็บ `Vec<String>` ของงาน, ฟังก์ชันเพิ่ม/
   ลบ/แสดงงาน ที่เขียนเป็น closure หรือฟังก์ชันภายใน `main`) จากนั้นลองสร้าง `tests/todo_tests.rs` ที่พยายาม
   `use todo_cli::...;` แล้วสังเกต error E0432 ที่เกิดขึ้นจริง ให้แก้ปัญหาตาม pattern หัวข้อ 33.7: แยก logic
   ทั้งหมดออกมาเป็น `src/lib.rs` (เช่น struct `TodoList` ที่มี method `add_task`, `remove_task`, `list_tasks`)
   ให้ `src/main.rs` เหลือแค่เรียก `TodoList` แล้วพิมพ์ผล จากนั้นเขียน integration test ให้ผ่านจริงใน
   `tests/todo_tests.rs` (hint: อ้างอิง Part 17 หัวข้อ 17.3 ทวนรูปแบบ "lib บาง ๆ + main บาง ๆ" ให้ครบ)

4. **[ยาก/ประยุกต์]** ต่อยอดจาก `shopping_cart` ในหัวข้อ 33.11: เพิ่มฟีเจอร์ใหม่ `apply_coupon_code` — เขียน
   ฟังก์ชัน private ชื่อ `coupon_discount_percent(code: &str) -> Option<u32>` ที่ใช้ `match` จับคู่ coupon code
   ที่รู้จัก (เช่น `"SAVE10"` -> `Some(10)`, `"SAVE25"` -> `Some(25)`, code อื่นที่ไม่รู้จัก -> `None`) แล้วเพิ่ม
   method public `pub fn total_with_coupon_cents(&self, code: &str) -> u32` ใน `ShoppingCart` ที่เรียกใช้มัน
   (ถ้า `coupon_discount_percent` คืน `None` ให้ใช้ `subtotal_cents()` แบบไม่มีส่วนลดแทน) เขียน unit test
   ทดสอบ `coupon_discount_percent` ตรง ๆ ครบทั้งกรณี code ที่รู้จัก, code ที่ไม่รู้จัก, และ code ที่เป็น empty
   string `""` แยกต่างหาก จากนั้นเพิ่ม `tests/common/mod.rs` ที่มีฟังก์ชัน
   `pub fn cart_with_two_books() -> shopping_cart::ShoppingCart` คืนตะกร้าที่มีสินค้าตั้งต้นสำหรับ integration
   test หลายไฟล์ใช้ร่วมกัน แล้วเขียน integration test ทดสอบ `total_with_coupon_cents` ทั้งกรณี coupon ถูกต้อง
   และ coupon ผิด/ไม่มีอยู่จริง สุดท้ายเขียนสรุปสั้น ๆ (3-5 บรรทัด) อธิบายว่าทำไมการทดสอบ `coupon_discount_percent`
   (private, มี logic การจับคู่หลายกรณี) ด้วย unit test และทดสอบ `total_with_coupon_cents` (public, เป็น
   workflow ที่ผู้ใช้เรียกจริง) ด้วย integration test จึงตรงกับหลัก testing pyramid ในหัวข้อ 33.8 มากกว่าการเลือก
   ทดสอบแบบใดแบบหนึ่งเพียงอย่างเดียว

## สรุป

บทนี้ต่อยอดจาก Part 32 โดยตอบคำถามที่ unit test ทำไม่ได้: **จะทดสอบ crate ของเราจากมุมมองของ "คนนอก" ที่มองเห็น
แค่ public API ได้อย่างไร** คำตอบคือ **integration test** — test ที่เขียนไว้ในไฟล์ `.rs` แต่ละไฟล์ใต้โฟลเดอร์
`tests/` ที่ root ของ package โดยแต่ละไฟล์ถูก Cargo compile เป็น **crate อิสระของตัวเอง** ที่ link เข้ากับ
library crate โดยอัตโนมัติ และมองเห็นได้เฉพาะ item ที่เป็น `pub` เท่านั้น — กฎเดียวกันเป๊ะกับที่ Part 16 สอนเรื่อง
visibility และ Part 17 สอนเรื่างการที่ `src/lib.rs`/`src/main.rs`/ไฟล์ใน `tests/` ต่างเป็นคนละ crate กันโดย
สิ้นเชิง ข้อสรุปที่สำคัญที่สุดคือ **integration test มีความหมายก็ต่อเมื่อ package มี library crate** — package
ที่มีแต่ binary crate จะเจอ error `E0432` ทันทีเมื่อพยายาม `use` มัน และวิธีแก้คือดึง logic ออกมาเป็น
`src/lib.rs` ตาม pattern "lib บาง ๆ + main บาง ๆ" จาก Part 17 หัวข้อ 17.3

เราเห็นวิธีรัน `cargo test` ให้ครอบคลุมทั้งสามชนิดของ test (unit, integration, และ doc-test ที่เกริ่นไว้ก่อน
เนื้อหาเต็มใน Part 34) ในคำสั่งเดียว, วิธีเจาะจงรันไฟล์เดียวด้วย `cargo test --test <ชื่อ>`, และข้อพิจารณาเรื่อง
ต้นทุนการ compile ที่เพิ่มขึ้นตามจำนวนไฟล์ใน `tests/` เราเรียนธรรมเนียม `tests/common/mod.rs` สำหรับแชร์ helper
code โดยไม่ให้ Cargo มองมันเป็น test crate ที่ไม่มี test อยู่ข้างใน และปิดท้ายด้วยแนวคิด **testing pyramid** ที่
ช่วยตัดสินใจว่าควรเขียน test แบบไหนตรงจุดไหน พร้อมตัวอย่างประยุกต์เต็มรูปแบบ `shopping_cart` ที่แสดงให้เห็นทั้ง
สองชั้นทำงานเสริมกันจริงในโปรเจกต์เดียว

ตอนนี้คุณมีกลยุทธ์การทดสอบที่ครบทั้งสองชั้นหลักแล้ว สิ่งที่ยังเหลืออยู่คือ doc-test ที่เกริ่นไว้สั้น ๆ ในหัวข้อ
33.9 — ใน **Part 34: Documentation: rustdoc** เราจะเจาะลึกเรื่องการเขียนเอกสารด้วย `///`/`//!`, ไวยากรณ์เต็ม
รูปแบบของ doc-test (การซ่อนบรรทัดด้วย `#`, ` ```ignore `, ` ```no_run `), การจัดหมวดหมู่เอกสารด้วย section
พิเศษ, และวิธีที่ docs.rs สร้างเอกสารจาก crate ที่ publish ขึ้น crates.io โดยอัตโนมัติ

---

**Part ก่อนหน้า:** [Testing: Unit Tests](part-032-testing-unit-tests.md) | **Part ถัดไป:** [Documentation: rustdoc](part-034-documentation-rustdoc.md)
