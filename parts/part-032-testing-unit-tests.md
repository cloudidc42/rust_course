# Part 32: Testing: Unit Tests (#[test], assert!, mocking เบื้องต้น)

> โมดูล: ระดับกลาง (Intermediate) | ระดับ: กลาง | เวลาโดยประมาณ: 220 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้อย่างลึกซึ้งว่าทำไม Rust ยังต้องการ automated test ทั้งที่ compiler ตรวจสอบ type/ownership/borrow
  เข้มงวดขนาดนั้นแล้ว และรู้ว่าเทสจับ "บั๊กประเภทไหน" ที่ compiler จับไม่ได้เลย
- เขียน unit test ด้วย `#[test]`, จัดเก็บไว้ใน `#[cfg(test)] mod tests { use super::*; ... }` ในไฟล์เดียวกับ
  โค้ดที่ทดสอบ และอธิบายได้ว่าทำไม convention นี้จึงมีประโยชน์ในเชิง module privacy
- ใช้คำสั่ง `cargo test` ได้อย่างคล่องแคล่วในทุกรูปแบบที่ใช้งานจริง: รันทั้งหมด, filter ด้วยชื่อ, ส่ง flag ผ่าน `--`,
  ควบคุมจำนวน thread, และเข้าใจผลกระทบของการรัน test แบบ parallel-by-default
- เลือกใช้ `assert!`, `assert_eq!`, `assert_ne!` ได้ถูกสถานการณ์ อ่าน failure message จาก `cargo test` ออกและรู้
  ว่าข้อความแต่ละส่วนหมายถึงอะไร พร้อมเขียน custom failure message ที่ช่วย debug ได้จริง
- ใช้ `#[should_panic]` (พร้อม `expected = "..."`) เพื่อพิสูจน์ว่าโค้ดปฏิเสธ input ที่ไม่ถูกต้องอย่างถูกวิธี และเขียนเทส
  ที่คืนค่า `Result<(), E>` เพื่อใช้ `?` แทน `.unwrap()` เกลื่อนเทส
- จัดระเบียบ test suite ที่โตขึ้นด้วย submodule หลายชุด, helper/fixture function ที่ใช้ร่วมกัน, และรู้จัก `#[ignore]`
  สำหรับเทสที่ช้า/แพง
- เข้าใจแนวคิดพื้นฐานของ "mocking" ผ่านเทคนิค trait-based dependency injection ที่ Rust ทำได้เองโดยไม่ต้องพึ่ง
  framework พิเศษใด ๆ และรู้จัก trade-off ระหว่างการทดสอบ private function กับการทดสอบผ่าน public API เท่านั้น

## ความรู้ที่ต้องมีมาก่อน

- **Part 2**: เรารู้จัก `cargo test` มาแล้วแบบผิวเผินตั้งแต่ Part 2 (หัวข้อ 2.4) และเคยเห็นโค้ดตัวอย่างที่ `cargo new --lib`
  สร้างให้มี `#[cfg(test)] mod tests { ... }` ติดมาด้วย ตอนนั้นเราบอกไว้ว่า "รายละเอียดเต็มจะอยู่ใน Part 32" — นี่คือ
  บทนั้น เราจะรื้อทุกอย่างที่เคยเห็นแบบผ่าน ๆ ออกมาอธิบายให้ลึกที่สุด
- **Part 9**: Struct, `impl` block, method ที่ใช้ `&self`/`&mut self`/`self` — ตัวอย่างหลักของบทนี้ (ระบบ `ShoppingCart`)
  ใช้ struct และ method จาก Part 9 เต็มรูปแบบ
- **Part 12**: `Result<T, E>` และ operator `?` — จำเป็นสำหรับหัวข้อ "เทสที่คืนค่า `Result<(), E>`" ซึ่งใช้หลักการเดียวกับ
  `fn main() -> Result<(), Box<dyn Error>>` ที่เคยเรียนใน Part 12 (หัวข้อ 12.8) มาประยุกต์กับฟังก์ชันเทส
- **Part 16**: Module system และกฎ privacy (`pub`, private-by-default, การมองเห็น item ภายใน module เดียวกัน) —
  จำเป็นมากสำหรับเข้าใจว่าทำไม `mod tests` ที่อยู่ในไฟล์เดียวกันถึงมองเห็นฟังก์ชัน/field ที่เป็น private ได้ ทั้งที่โค้ด
  จากไฟล์อื่นมองไม่เห็น
- **Part 19 และ Part 21**: Trait พื้นฐานและขั้นสูง (การนิยาม trait, การ implement ให้หลาย type, generic ที่มี trait
  bound, trait object) — จำเป็นสำหรับหัวข้อ mocking เบื้องต้น ซึ่งใช้เทคนิค "นิยาม trait แทน dependency แล้ว implement
  สองแบบ (ของจริง/ของปลอม)" ที่เป็นแนวคิดต่อเนื่องจากสองบทนี้โดยตรง

ถ้ายังไม่มั่นใจเรื่อง privacy rules จาก Part 16 หรือ trait จาก Part 19/21 แนะนำให้ย้อนไปทวนก่อนเริ่มหัวข้อ 32.10 และ
32.12 เพราะสองหัวข้อนั้นสร้างอยู่บนพื้นฐานเหล่านั้นตรง ๆ

## เนื้อหา

### 32.1 ทำไมต้องมี automated test ทั้งที่ Rust compiler เข้มงวดขนาดนี้แล้ว

ถ้าคุณเรียนมาถึง Part 32 ของหลักสูตรนี้ คุณคงสัมผัสมาแล้วว่า Rust compiler นั้น "จู้จี้" กว่าภาษาส่วนใหญ่มาก — มัน
บังคับให้จัดการ ownership ให้ถูกต้อง (Part 6), บังคับให้ borrow ตาม rule ที่ชัดเจน (Part 7), บังคับให้จัดการทุก
กรณีของ `enum`/`Option`/`Result` ให้ครบผ่าน pattern matching (Part 10-12), และปฏิเสธโค้ดที่มีโอกาสเกิด null pointer
dereference, data race, หรือ use-after-free ตั้งแต่ตอน compile เลยไม่ต้องรอไปเจอตอนรัน คำถามที่มือใหม่หลายคนสงสัยคือ
**"ถ้า compiler เข้มงวดขนาดนี้แล้ว ยังต้องเขียนเทสอีกทำไม?"**

คำตอบคือ **compiler และ automated test จับบั๊กคนละประเภทกันโดยสิ้นเชิง** สิ่งที่ compiler ของ Rust ตรวจสอบได้คือ
"ความถูกต้องเชิงโครงสร้าง" (structural correctness) — โค้ดของคุณ type ตรงกันทุกจุดหรือไม่ ownership ชัดเจนหรือไม่
ไม่มี reference ที่ dangling หรือไม่ ไม่มี data race ข้าม thread หรือไม่ สิ่งเหล่านี้คือ **"บั๊กเชิงกลไก"
(mechanical bugs)** ที่ตรวจสอบได้จากการวิเคราะห์โครงสร้างโค้ดเพียงอย่างเดียว โดยไม่ต้องรู้เลยว่าโปรแกรม "ควรทำอะไร"

แต่มีบั๊กอีกประเภทหนึ่งที่ compiler **ไม่มีทางรู้ได้เลย** เพราะมันคือ **บั๊กเชิงตรรกะ (logic bugs)** — โค้ดที่ compile
ผ่านสมบูรณ์แบบ ไม่มี type error, ไม่มี borrow checker error, รันได้โดยไม่ panic เลยแม้แต่ครั้งเดียว แต่**ให้ผลลัพธ์ที่
ผิดจากความต้องการทางธุรกิจ** ลองดูตัวอย่างนี้:

```rust
/// คำนวณราคาสินค้าหลังหักส่วนลด
fn apply_discount(price_cents: u32, discount_percent: u32) -> u32 {
    // บั๊ก: ลืมหาร 100 หลังคูณ percent — สูตรที่ถูกต้องคือ price * (100 - discount) / 100
    price_cents * (100 - discount_percent)
}

fn main() {
    let price = apply_discount(10000, 10); // ต้องการ: ลด 10% จาก 100 บาท ควรได้ 90 บาท (9000 สตางค์)
    println!("ราคาหลังหักส่วนลด: {price} สตางค์");
}
```

ลองดูผลลัพธ์:

```
ราคาหลังหักส่วนลด: 900000 สตางค์
```

โค้ดนี้ **compile ผ่านสมบูรณ์แบบ ไม่มี warning ใด ๆ เลย** และรันได้โดยไม่ panic — แต่ผลลัพธ์ผิดพลาดไปมหาศาล (900,000
สตางค์ = 9,000 บาท ทั้งที่ควรได้ 90 บาท) เพราะสูตรคำนวณลืมหารด้วย 100 ตรงนี้คือประเด็นสำคัญที่สุดของบทนี้:

> **compiler รับประกันว่าโค้ดของคุณ "ทำงานได้อย่างสม่ำเสมอตามที่เขียน" แต่ไม่มีทางรับประกันได้เลยว่าสิ่งที่คุณเขียน
> "คือสิ่งที่คุณตั้งใจจะให้มันทำ"** ช่องว่างระหว่างสองสิ่งนี้คือพื้นที่ที่ automated test เข้ามาเติมเต็ม

ถ้าเราเขียนเทสสำหรับฟังก์ชันนี้ไว้ตั้งแต่แรก:

```rust
fn apply_discount(price_cents: u32, discount_percent: u32) -> u32 {
    price_cents * (100 - discount_percent)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn ten_percent_discount_on_100_baht() {
        // ลด 10% จาก 10000 สตางค์ (100 บาท) ต้องได้ 9000 สตางค์ (90 บาท)
        assert_eq!(apply_discount(10000, 10), 9000);
    }
}
```

การรัน `cargo test` จะจับบั๊กนี้ได้ทันทีตั้งแต่ก่อนขึ้น production เพราะเทสจะ **FAILED** ทันที (ได้ 900000 แทนที่จะ
เป็น 9000 ตามที่คาด) — นี่คือคุณค่าของเทส: มันไม่ได้มาแทนที่ compiler แต่มาทำงาน **ต่อจาก** compiler ในจุดที่
compiler มองไม่เห็น

**ประเด็นสำคัญที่ต้องเข้าใจให้ชัด: เทสไม่ได้มาแทนที่สิ่งที่ compiler ทำอยู่แล้ว** ในภาษาอย่าง Python หรือ JavaScript
ที่ไม่มี static type checking เข้มงวด นักพัฒนาต้องเขียนเทสจำนวนมากเพื่อจับบั๊กประเภท "ส่ง argument ผิด type",
"เรียก method ที่ไม่มีอยู่จริง (typo)", "ค่าที่ควรมีอยู่กลับเป็น `None`/`null` แล้วพยายามเข้าถึง attribute ของมัน"
— บั๊กเหล่านี้ใน Rust **compiler จับให้ตั้งแต่ compile time หมดแล้ว** ทำให้ test suite ของโปรแกรม Rust ไม่จำเป็นต้อง
เสียเวลาเขียนเทสสำหรับกรณีเหล่านี้เลย (compiler ปฏิเสธโค้ดที่ผิดแบบนั้นไปตั้งแต่ต้น ไม่มีทางรันถึงจุดที่เทสจะจับได้
ด้วยซ้ำ) ทำให้ test suite ของ Rust สามารถ**โฟกัสเต็มที่กับบั๊กเชิงตรรกะ** (สูตรคำนวณผิด, edge case ที่ลืมจัดการ,
เงื่อนไขทางธุรกิจที่ implement ผิด) ซึ่งเป็นเนื้อหาที่ "มีคุณค่าสูงสุด" ของการเขียนเทสจริง ๆ

สรุปความสัมพันธ์ระหว่าง compiler กับเทสเป็นตารางนี้:

| สิ่งที่ตรวจสอบ | ใครจับได้ | ตัวอย่างบั๊ก |
|---|---|---|
| Type ไม่ตรงกัน | Compiler (compile time) | ส่ง `&str` ให้ฟังก์ชันที่รับ `u32` |
| Null/uninitialized value | Compiler (ผ่าน `Option<T>` + exhaustive match) | ลืมจัดการกรณี `None` |
| Data race ข้าม thread | Compiler (ผ่าน `Send`/`Sync` + borrow checker) | สอง thread เขียนตัวแปรเดียวกันพร้อมกันโดยไม่ sync |
| Use-after-free / dangling reference | Compiler (ผ่าน lifetime + borrow checker) | คืน reference ไปยังตัวแปร local ที่ตายไปแล้ว |
| สูตรคำนวณผิด | **เทส (runtime)** | ลืมหาร 100 ในการคำนวณเปอร์เซ็นต์ |
| เงื่อนไขทางธุรกิจ implement ผิด | **เทส (runtime)** | ลืมกรณี "ลูกค้า VIP ไม่ต้องเสียค่าส่ง" |
| Edge case ที่ไม่ได้คิดถึง | **เทส (runtime)** | ลืมจัดการตะกร้าสินค้าว่างตอน checkout |
| Regression (แก้ที่หนึ่งแล้วพังอีกที่) | **เทส (runtime)** | แก้บั๊ก A แต่ทำให้ feature B ที่เคยถูกต้องกลับพัง |

แถวบนสามแถวคือสิ่งที่ Rust compiler จัดการให้แบบ "ฟรี" (คุณได้มันมาโดยไม่ต้องเขียนเทสเพิ่มเลย) ส่วนแถวล่างสี่แถวคือ
พื้นที่ที่เทสเข้ามาทำงาน — และนี่คือเหตุผลที่แม้ Rust จะปลอดภัยกว่าภาษาอื่นมากในเชิง memory safety แต่โปรเจกต์ Rust
จริงจังทุกโปรเจกต์ก็ยังมี test suite ขนาดใหญ่เสมอ เพราะ "ความปลอดภัยเชิง memory" กับ "ความถูกต้องเชิงตรรกะทาง
ธุรกิจ" เป็นคนละเรื่องกันโดยสิ้นเชิง

อีกมุมมองหนึ่งที่ควรเข้าใจคือเทสทำหน้าที่เป็น **"สัญญา (contract) ที่ยืนยันได้อัตโนมัติ"** ระหว่างคนที่เขียนโค้ด
กับความตั้งใจดั้งเดิมของโค้ดนั้น เมื่อเวลาผ่านไปและมีคนอื่น (หรือตัวคุณเองในอีกหกเดือนข้างหน้า) มา refactor โค้ด
เทสที่เขียนไว้จะทำหน้าที่เป็น "ผู้เฝ้าประตู" ที่ตรวจจับได้ทันทีถ้าการ refactor นั้นเผลอเปลี่ยน behavior ที่ตั้งใจไว้แต่
แรกโดยไม่รู้ตัว (สิ่งที่เรียกว่า **regression** ในแถวสุดท้ายของตาราง) — คุณค่าของเทสจึงไม่ใช่แค่ "จับบั๊กตอนเขียนครั้ง
แรก" แต่รวมถึง "ป้องกันบั๊กใหม่ตอนแก้ไขโค้ดเก่าในอนาคต" ด้วย ซึ่งความสำคัญของแง่มุมนี้จะยิ่งชัดขึ้นเรื่อย ๆ เมื่อ
โปรเจกต์โตขึ้นและมีคนทำงานร่วมกันหลายคน

### 32.2 กายวิภาคของ `#[test]`: ฟังก์ชันทดสอบที่เรียบง่ายที่สุด

หัวใจของ testing framework ที่ผูกมากับ Rust (ไม่ต้องติดตั้ง library เพิ่มเติมใด ๆ — มันเป็นส่วนหนึ่งของ `rustc`/
`cargo` เองโดยตรง) คือ attribute `#[test]` วางไว้เหนือฟังก์ชันที่**ไม่รับ argument และไม่คืนค่า** (หรือคืนค่าที่
implement trait พิเศษ ซึ่งเราจะพูดถึงในหัวข้อ 32.8):

```rust
fn add_two(a: i32, b: i32) -> i32 {
    a + b
}

#[test]
fn it_adds_two_numbers() {
    let result = add_two(2, 2);
    assert_eq!(result, 4);
}
```

กลไกการทำงานเบื้องหลัง `#[test]` คือ: เมื่อคุณ compile โค้ดด้วย `cargo build`/`cargo run` ปกติ **ฟังก์ชันที่มี
`#[test]` จะไม่ถูก compile เข้าไปในโปรแกรมเลย** มันจะถูก compile เข้าไปก็ต่อเมื่อสั่งด้วย `cargo test` เท่านั้น
เบื้องหลังคือ `cargo test` จะ compile โค้ดของคุณด้วย flag พิเศษที่เปลี่ยนทุกฟังก์ชันที่มี `#[test]` ให้กลายเป็น
"test case" หนึ่งตัว แล้วสร้าง binary พิเศษ (เรียกว่า **test harness** หรือ **test runner**) ที่หน้าที่หลักคือ:

1. รันทุกฟังก์ชันที่ทำ mark ด้วย `#[test]`
2. ถ้าฟังก์ชันรันจบโดยไม่ `panic!` เลย → ถือว่า test นั้น **ผ่าน (passed)**
3. ถ้าฟังก์ชัน `panic!` ระหว่างรัน (ไม่ว่าจะมาจาก `assert!`/`assert_eq!` ที่ fail, index out of bounds,
   `.unwrap()` บน `None`, หรือ `panic!()` ตรง ๆ) → ถือว่า test นั้น **ล้มเหลว (failed)**
4. สรุปผลรวมทั้งหมดเป็นรายงานท้ายสุด

จุดที่สำคัญมากคือ **กลไกการตัดสิน pass/fail ของ Rust test framework ใช้ mechanism เดียวกับ panic ทั่วไปทั้งหมด**
ไม่มี API พิเศษแยกต่างหากสำหรับ "บอกว่า test นี้ fail" — `assert!`/`assert_eq!`/`assert_ne!` ที่เราจะเรียนในหัวข้อ
32.5 ก็เป็นเพียง macro ที่ท้ายที่สุดเรียก `panic!()` เมื่อเงื่อนไขไม่เป็นจริง test harness จะ "จับ" panic นั้นด้วย
`std::panic::catch_unwind` (กลไกระดับ standard library ที่ทำให้ thread หนึ่งจับ panic ของอีก thread ได้โดยไม่ทำให้
โปรแกรมทั้งตัว crash) แล้วรายงานว่า test นั้น fail พร้อมข้อความ panic ที่ได้

ลองรัน `cargo test` กับตัวอย่างข้างบน (สมมติอยู่ใน `src/lib.rs` ของโปรเจกต์ library):

```
   Compiling calc_demo v0.1.0 (/home/user/calc_demo)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.28s
     Running unittests src/lib.rs (target/debug/deps/calc_demo-04c9d39c6861919b)

running 1 test
test it_adds_two_numbers ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests calc_demo

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

มาแยกอ่านผลลัพธ์ทีละส่วน:

- **`Running unittests src/lib.rs (target/debug/deps/calc_demo-...)`** — cargo compile test harness เป็น binary
  แยกต่างหากไว้ที่ `target/debug/deps/` แล้วรันมัน (สังเกตชื่อไฟล์มี hash ต่อท้ายเพื่อไม่ให้ชนกับ binary อื่น)
- **`running 1 test`** — บอกจำนวนเทสทั้งหมดที่พบใน binary นี้
- **`test it_adds_two_numbers ... ok`** — สถานะของแต่ละเทสทีละตัว (`ok` = ผ่าน)
- **`test result: ok. 1 passed; 0 failed; ...`** — สรุปผลรวม เราจะกลับมาอธิบายตัวเลขแต่ละตัว (`measured`,
  `filtered out`) ในหัวข้อถัดไปที่เกี่ยวข้องโดยตรง (`#[ignore]` ทำให้เห็น `ignored` ไม่เป็น 0, การ filter ด้วยชื่อ
  ทำให้เห็น `filtered out` ไม่เป็น 0)
- **`Doc-tests calc_demo`** — ส่วนนี้มาจาก doc comment (`///`) ที่มี code block ข้างในซึ่ง cargo จะรันเป็นเทสด้วย
  โดยอัตโนมัติ (เรื่องนี้เป็นเนื้อหาของ Part 34 เรื่อง Documentation — ตอนนี้แค่รู้ไว้ว่าส่วนนี้มีอยู่จริงและจะโผล่มา
  ทุกครั้งที่รัน `cargo test` แม้ว่าคุณยังไม่ได้เขียน doc comment เลยก็ตาม เพราะ cargo ค้นหา doc test เสมอไม่ว่าจะ
  เจอหรือไม่)

**Exit code ของ `cargo test`**: ถ้าทุกเทสผ่าน `cargo test` จะคืน exit code `0` (สำเร็จ) แต่ถ้ามีเทสล้มเหลวแม้แต่ตัว
เดียว จะคืน exit code `101` — นี่คือสิ่งที่ทำให้ `cargo test` ใช้เป็นส่วนหนึ่งของ CI/CD pipeline ได้ตรงไปตรงมา
(pipeline เช็คแค่ exit code ว่าเป็น 0 หรือไม่ ก็รู้แล้วว่าควรปล่อยโค้ดต่อไปหรือหยุดไว้ก่อน)

### 32.3 `#[cfg(test)]` และ `mod tests { use super::*; }`: ทำไมเทสอยู่ไฟล์เดียวกับโค้ด

Convention มาตรฐานของ Rust (ที่เราเห็นมาแล้วตั้งแต่ตอน `cargo new --lib` สร้างให้อัตโนมัติใน Part 2) คือการวาง
unit test ไว้ **ในไฟล์เดียวกับโค้ดที่ทดสอบ** ภายใน module พิเศษที่ชื่อ `tests` (ชื่อนี้เป็น convention ล้วน ๆ ไม่ใช่
keyword พิเศษ — คุณตั้งชื่ออื่นก็ได้ แต่ทุกคนในระบบนิเวศ Rust ใช้ชื่อ `tests` เหมือนกันหมด เพื่อความคุ้นเคยข้าม
โปรเจกต์):

```rust
// src/discount.rs

pub fn apply_discount(price_cents: u32, discount_percent: u32) -> u32 {
    price_cents * (100 - discount_percent) / 100
}

fn validate_discount_percent(percent: u32) -> bool {
    percent <= 100
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn ten_percent_off_100_baht() {
        assert_eq!(apply_discount(10000, 10), 9000);
    }

    #[test]
    fn valid_percent_is_accepted() {
        // เทสนี้เรียก validate_discount_percent ซึ่งเป็นฟังก์ชัน private ของ module นี้ — ทำได้ปกติ
        assert!(validate_discount_percent(50));
        assert!(!validate_discount_percent(150));
    }
}
```

มีสาม element ที่ต้องเข้าใจแยกกันชัด ๆ:

#### `#[cfg(test)]`: บอก compiler ว่า "compile module นี้เฉพาะตอนเทสเท่านั้น"

`cfg` คือ **conditional compilation attribute** — มันสั่ง compiler ว่า "ให้ compile item ข้างใต้นี้เข้าไปในผลลัพธ์
สุดท้าย **เฉพาะเมื่อเงื่อนไขในวงเล็บเป็นจริง** เท่านั้น" `cfg(test)` หมายถึง "เงื่อนไข `test` เป็นจริง" ซึ่ง cargo จะ
เปิดเงื่อนไขนี้ให้เองโดยอัตโนมัติเฉพาะตอนที่คุณสั่ง `cargo test` เท่านั้น — เมื่อคุณสั่ง `cargo build` หรือ
`cargo run` ปกติ เงื่อนไข `test` จะเป็นเท็จ ทำให้ module `tests` ทั้งก้อน **ไม่ถูก compile เข้าไปในโปรแกรมจริงเลย
แม้แต่ไบต์เดียว**

ผลลัพธ์เชิงปฏิบัติของสิ่งนี้คือ:

- โค้ดเทสไม่เพิ่มขนาด binary ที่จะแจกจ่ายให้ผู้ใช้จริงเลยแม้แต่นิดเดียว (ตรงกับสิ่งที่เรียนใน Part 2 เรื่อง
  `[dev-dependencies]` — dependency ที่ใช้เฉพาะตอนเทส เช่น library ช่วยสร้าง mock ก็จะไม่ถูก compile ไปด้วยด้วย
  เหตุผลเดียวกัน)
- โค้ดเทสสามารถใช้ syntax หรือเรียก dependency ที่ไม่มีอยู่ใน production build ได้อย่างปลอดภัย (เช่น เรียก
  `println!` เพื่อ debug ระหว่างเทส โดยไม่ต้องห่วงว่าจะไปโผล่ใน production log)
- ถ้าคุณเผลอเขียนโค้ดเทสผิด syntax หรือมี type error `cargo build`/`cargo run` แบบปกติ**จะไม่รู้เลยว่ามันผิด**
  เพราะ compiler ไม่ได้แม้แต่จะมองเห็นโค้ดชิ้นนั้น (`cfg(test)` เป็นเท็จ) — ต้องรัน `cargo test` หรือ
  `cargo check --tests` เท่านั้นถึงจะเห็น error พวกนี้ นี่คือเหตุผลที่แนะนำให้รัน `cargo test` (หรืออย่างน้อย
  `cargo check --tests`) เป็นประจำระหว่างพัฒนา ไม่ใช่รอจนอยากรันเทสจริง ๆ ค่อยมาเจอ compile error สะสมทีเดียว

#### `mod tests { ... }`: module ธรรมดาตาม Part 16 ทุกกฎ

`mod tests` ไม่ใช่ syntax พิเศษอะไรเลย — มันคือการนิยาม module แบบเดียวกับที่เรียนใน Part 16 เป๊ะ ๆ เพียงแต่ตั้งชื่อ
ว่า `tests` ตาม convention และ mark ด้วย `#[cfg(test)]` เท่านั้น กฎ **privacy ทั้งหมดจาก Part 16 ยังใช้กับมันเหมือน
module อื่น ๆ ทุกอย่าง** ซึ่งพาเราไปยังจุดสำคัญที่สุดของหัวข้อนี้

#### `use super::*;`: ทำไมเทสถึงมองเห็นฟังก์ชัน private ได้

จาก Part 16 (หัวข้อ 16.5) เราเรียนกฎนี้ไปแล้ว: **item ที่เป็น private (ไม่มี `pub`) จะมองไม่เห็นจาก "ภายนอก module"
ที่นิยามมัน แต่ "ภายนอก module" ในที่นี้ไม่รวมโมดูลลูกของมันเอง** — module ลูกมองเห็น item private ของ module พ่อ
(และของทุก ancestor ขึ้นไป) ได้เสมอ

`mod tests` ที่นิยามอยู่**ข้างใน**ไฟล์เดียวกับฟังก์ชันที่จะทดสอบ จึงเป็น **module ลูก** ของ module ที่ครอบไฟล์นั้นอยู่
โดยอัตโนมัติ (ไม่ว่าไฟล์นั้นจะเป็น crate root อย่าง `lib.rs` หรือเป็น module ย่อยอย่าง `discount.rs`) ดังนั้นตาม
กฎ privacy จาก Part 16 `mod tests` จึง**มองเห็น item private ทุกตัวของไฟล์นั้นได้ทันที** — นี่คือเหตุผลที่บรรทัด
`use validate_discount_percent(...)` (ฟังก์ชันที่ไม่มี `pub` เลย) ในตัวอย่างข้างบน compile ผ่านได้สบาย ๆ ทั้งที่ถ้า
เราลองเรียกฟังก์ชันเดียวกันนี้จากไฟล์อื่นข้างนอก module `discount` จะได้ error `E0603: function ... is private`
ทันที (ตามที่ Part 16 แสดงไว้)

ส่วน `use super::*;` คือบรรทัดที่ทำให้ทุกอย่างข้างในทำงานได้จริง — `super` หมายถึง "module พ่อของ module ปัจจุบัน"
(ตาม Part 16 หัวข้อ 16.5) และ `*` คือการ `use` ทุก item ที่ module พ่อมองเห็นได้ทั้งหมด (ทั้ง public และ private
เพราะ `mod tests` เป็นลูกของมัน) เข้ามาอยู่ใน scope ของ `mod tests` โดยไม่ต้องเขียน path เต็มซ้ำ ๆ ถ้าไม่มีบรรทัดนี้
คุณจะต้องเขียน `super::apply_discount(...)` แบบ path เต็มทุกครั้งที่เรียก ซึ่งเวิร์กได้เหมือนกันแต่ยาวกว่ามาก
(convention จึงเลือกใช้ `use super::*;` เพื่อความสะดวก — เป็นหนึ่งในไม่กี่ที่ที่ community ยอมรับ glob import
`use ...::*` แม้ว่าปกติ Rust จะไม่ค่อยแนะนำ glob import ใน production code เพราะทำให้มองไม่เห็นชัดว่า item มาจากไหน
แต่ในบริบทของ `mod tests` มันตรงไปตรงมาพอ เพราะทุกอย่างมาจาก parent module เดียวกันเท่านั้น ไม่มี ambiguity)

**นี่คือเหตุผลหลักที่ Rust เลือก convention "เทสอยู่ไฟล์เดียวกับโค้ด" แทนที่จะแยกไปไว้ไฟล์อื่นแบบภาษาอื่นบางภาษา**
(เช่น Python ที่มักแยกไฟล์ `test_*.py` ไว้คนละโฟลเดอร์) เหตุผลเชิงเทคนิคคือ **การเข้าถึง private item ได้โดยตรง**
— คุณสามารถทดสอบ implementation detail ภายใน (ฟังก์ชัน private, helper function เล็ก ๆ ที่ไม่ควร expose ออกไป
ให้ผู้ใช้ crate เรียกใช้เอง) ได้ง่ายพอ ๆ กับทดสอบ public API เลย โดยไม่ต้องเปลี่ยน visibility ของอะไรเลยแม้แต่นิดเดียว
เพียงเพื่อ "เปิดช่องให้เทสเข้าถึงได้" (ซึ่งเป็น anti-pattern ที่พบบ่อยในภาษาที่ทดสอบ private ได้ยาก — นักพัฒนาจำนวน
มากยอมทำให้ method เป็น public ทั้งที่ไม่ควร เพียงเพราะอยากให้เทสเรียกได้) เราจะพูดถึงข้อดี-ข้อเสียของการทดสอบ
private function อย่างละเอียดในหัวข้อ 32.10

**ข้อสังเกตเสริม**: การจัดระเบียบแบบนี้ยังหมายความว่า test suite ของ unit test **ไม่มีสิทธิ์เข้าถึงอะไรที่อยู่นอก
crate เลย** (มันยังคงเป็นแค่ module ลูกภายใน crate เดียวกัน ไม่ใช่ "สิทธิพิเศษข้าม crate") — ถ้าต้องการทดสอบ crate
ของคุณ "จากมุมมองของผู้ใช้ภายนอก" (เห็นแค่ public API เหมือนคนอื่นที่มา `use` crate ของคุณ) นั่นคือหน้าที่ของ
**integration test** ที่อยู่ในโฟลเดอร์ `tests/` แยกต่างหาก ซึ่งเป็นเนื้อหาของ **Part 33** ที่จะเรียนต่อจากบทนี้
บทนี้โฟกัสเฉพาะ unit test ที่อยู่ในไฟล์เดียวกับ implementation เท่านั้น

### 32.4 `cargo test` เจาะลึก: filter, flag, และการรันแบบ parallel

`cargo test` มี option มากกว่าที่เห็นตอนรันเปล่า ๆ มาก มาดูรูปแบบที่ใช้งานจริงบ่อยที่สุด

#### รันทั้งหมด

```bash
cargo test
```

รันทุกเทสที่ compiler หาเจอในทั้ง unit test (`src/`) และ integration test (`tests/`, ถ้ามี — เรียนใน Part 33)
รวมถึง doc test ด้วย

#### รันเฉพาะเทสที่ชื่อมี substring ตรงกับที่ระบุ

```bash
cargo test independent
```

Argument ตัวแรกของ `cargo test` (ถ้ามี) ไม่ใช่ชื่อเทสแบบเป๊ะ ๆ แต่เป็น **substring filter** — มันจะรัน**ทุกเทสที่
ชื่อ (รวม path ของ module ที่ครอบอยู่) มีคำนี้เป็นส่วนหนึ่ง** ตัวอย่างจริงจากการรัน (สมมติมีเทสชื่อ
`tests::independent_test_a` และ `tests::independent_test_b` อยู่ในไฟล์เดียวกัน):

```
running 2 tests
test tests::independent_test_b ... ok
test tests::independent_test_a ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 11 filtered out; finished in 0.00s
```

สังเกตตัวเลข **`11 filtered out`** — นี่คือจำนวนเทสที่**มีอยู่จริง**ในโปรเจกต์แต่ไม่ตรงกับ filter `independent`
เลยไม่ถูกรัน (มันไม่ใช่ error หรือปัญหาใด ๆ เป็นแค่การรายงานให้รู้ว่ามีเทสอื่นอยู่ด้วยที่รอบนี้ไม่ได้ถูกเลือก) นี่คือ
feature ที่มีประโยชน์มากตอนกำลังไล่แก้บั๊กของฟีเจอร์เดียว — ไม่ต้องรอเทสทั้ง suite ที่อาจมีหลายร้อยตัวรันจนครบทุกครั้ง
ที่แก้โค้ดนิดเดียว

ระบุชื่อ module เพื่อรันทั้ง module ก็ได้ เช่น `cargo test tests::` จะรันทุกเทสที่อยู่ใน module ที่ชื่อมีคำว่า
`tests::` เป็นส่วนหนึ่งของ path (ซึ่งในกรณีปกติที่ทุกไฟล์ตั้งชื่อ module เทสว่า `tests` เหมือนกัน จะหมายถึง "รันทุก
unit test ทั้งหมด" นั่นเอง)

#### `--` : เส้นแบ่งระหว่าง flag ของ `cargo` กับ flag ของ test binary

จุดที่มือใหม่สับสนบ่อยที่สุดคือ `cargo test` มี flag สองชุดที่แยกกันเด็ดขาด:

- **flag ก่อน `--`** เป็นของ `cargo` เอง (เช่น `--release` เพื่อ compile แบบ optimize, `--lib` เพื่อรันแค่ unit
  test ไม่รัน integration test, `-p <package>` เพื่อเลือก package ใน workspace)
- **flag หลัง `--`** เป็นของ **test binary** ที่ cargo compile ออกมา (test harness ที่พูดถึงในหัวข้อ 32.2) — flag
  พวกนี้ cargo เองไม่รู้จักเลย มันแค่ pass ต่อให้ binary นั้นไปจัดการเอง

รูปแบบเต็ม:

```bash
cargo test [FILTER] [CARGO_FLAGS] -- [TEST_BINARY_FLAGS]
```

#### `-- --nocapture`: ดู `println!` ระหว่างเทส

โดย default, `cargo test` จะ **ดัก (capture) ทุก output ที่มาจาก `println!`/`print!`/`eprintln!` ระหว่างรัน
เทส และไม่แสดงมันเลยถ้าเทสนั้นผ่าน** (แสดงเฉพาะตอนเทส fail เท่านั้น อย่างที่เห็นในหัวข้อ 32.5) เหตุผลของการ
ออกแบบแบบนี้คือ **ความสะอาดของ output** — ถ้าคุณมีเทสหลายร้อยตัวที่ทุกตัวมี `println!` debug อยู่ การเห็น log
ท่วมจอทุกครั้งที่รัน `cargo test` (ทั้งที่ทุกอย่างผ่านหมด ไม่มีอะไรต้อง debug) จะรบกวนมากกว่าช่วย

ถ้าต้องการเห็น `println!` แม้เทสจะผ่าน (เวลากำลัง debug อยู่จริง ๆ) ใช้:

```bash
cargo test prints_something_useful -- --nocapture
```

ผลลัพธ์จริงจากการรัน:

```
running 1 test
กำลังทดสอบ add_two ด้วยค่า 10 และ 20
ผลลัพธ์ที่ได้คือ 30
test tests::prints_something_useful ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 12 filtered out; finished in 0.00s
```

สังเกตว่าบรรทัด `println!` ทั้งสองปรากฏขึ้น**ก่อน**บรรทัด `test ... ok` — เพราะ output ทั้งหมดของเทสหนึ่งตัวจะถูก
buffer ไว้แล้วพิมพ์ออกมาเป็นก้อนเดียวหลังเทสนั้นรันจบ (ไม่ใช่ interleave กับ output ของเทสตัวอื่นที่รันคู่กันแบบ
สุ่ม ๆ ซึ่งจะทำให้อ่านไม่รู้เรื่องเลยถ้ารันแบบ parallel)

#### `-- --test-threads=1`: รันแบบ sequential (ทีละตัว ไม่ parallel)

```bash
cargo test -- --test-threads=1
```

บังคับให้ test harness รันเทสทีละตัว**เรียงตามลำดับ** ไม่กระจายไปรันพร้อมกันหลาย thread เราจะอธิบายว่าทำไม flag
นี้ถึงจำเป็นในบางสถานการณ์ในหัวข้อย่อยถัดไปทันที

#### Default: parallel by default — และทำไม test ต้อง independent จากกัน

นี่คือประเด็นเชิงออกแบบที่สำคัญที่สุดของหัวข้อนี้: **`cargo test` รันทุกเทสแบบ parallel (หลาย thread พร้อมกัน) โดย
default** จำนวน thread ที่ใช้ขึ้นกับจำนวน CPU core ที่เครื่องมี (ปรับได้ด้วย `--test-threads=N` อย่างที่เห็นข้างบน)

**เหตุผลของการออกแบบแบบนี้คือความเร็ว** — โปรเจกต์จริงอาจมีเทสนับพันตัว ถ้ารันทีละตัวเรียงกันบน CPU ตัวเดียวจะ
เสียเวลามาก แต่ถ้าเทสแต่ละตัวเป็นอิสระจากกันจริง ๆ (ไม่แก้ไข state ร่วมกัน) การรันหลายตัวพร้อมกันบนหลาย core
ก็ปลอดภัยและได้ผลลัพธ์เหมือนรันทีละตัวทุกประการ แต่**เร็วกว่ามาก** (ในเครื่องที่มี 8 core อาจเร็วขึ้นได้ถึง
ประมาณ 8 เท่าในทางทฤษฎี)

แต่การออกแบบนี้มาพร้อม**ข้อผูกมัดสำคัญ**: **เทสของคุณต้องเป็นอิสระจากกันโดยสมบูรณ์ (independent)** ห้ามมีเทสสอง
ตัวที่แข่งกันแก้ไข shared mutable state ตัวเดียวกัน (เช่น ตัวแปร `static mut`, ไฟล์ที่ path เดียวกันบน disk,
environment variable เดียวกัน, แถวเดียวกันในฐานข้อมูลทดสอบ) เพราะถ้าเทสสองตัวรันพร้อมกันแล้วแตะ resource ร่วมกัน
ผลลัพธ์จะกลายเป็น **race condition** — บางครั้งเทสผ่าน บางครั้งเทสล้มเหลว โดยที่โค้ดจริงไม่ได้เปลี่ยนแปลงอะไรเลย
(เทสที่มีพฤติกรรมแบบนี้เรียกว่า **flaky test** — เป็นสิ่งที่ทีมพัฒนาทุกทีมเกลียดมากที่สุด เพราะทำให้ไม่มีใครเชื่อ
ผลลัพธ์ของ CI อีกต่อไป "fail รอบนี้เพราะบั๊กจริง หรือเพราะ flaky กันแน่?")

ตัวอย่างโค้ดที่**ผิด** (สาธิตปัญหา ไม่ควรทำตาม):

```rust
use std::fs;

fn save_report_to_disk(content: &str) -> std::io::Result<()> {
    // บั๊กเชิงออกแบบ: ใช้ path คงที่เดียวกันทุกครั้ง ไม่สนใจว่าใครเรียกจากที่ไหน
    fs::write("/tmp/report.txt", content)
}

fn read_report_from_disk() -> std::io::Result<String> {
    fs::read_to_string("/tmp/report.txt")
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn writes_report_a() {
        save_report_to_disk("รายงาน A").unwrap();
        let content = read_report_from_disk().unwrap();
        assert_eq!(content, "รายงาน A"); // อาจ fail แบบสุ่ม! ถ้า writes_report_b ทับไฟล์นี้พอดีระหว่างที่เทสนี้กำลังอ่าน
    }

    #[test]
    fn writes_report_b() {
        save_report_to_disk("รายงาน B").unwrap();
        let content = read_report_from_disk().unwrap();
        assert_eq!(content, "รายงาน B"); // เจอปัญหาแบบเดียวกันในทิศตรงข้าม
    }
}
```

เพราะ `cargo test` รันทั้งสองเทสนี้แบบ parallel โดย default ทั้งสองอาจแข่งกันเขียนไฟล์ `/tmp/report.txt` ไฟล์
เดียวกันพร้อมกัน — ผลลัพธ์คือใครเขียนทีหลังจะ "ชนะ" แต่ลำดับการรันของแต่ละ thread ไม่แน่นอน ทำให้เทสตัวใดตัวหนึ่ง
(หรือทั้งสองตัว) อ่านค่าที่**อีกเทสหนึ่งเขียนทับไป**แล้ว fail แบบสุ่ม ๆ ไม่ fail ทุกครั้งแต่ fail เป็นบางครั้ง —
นี่คือลักษณะเฉพาะของ flaky test ที่ทำให้ debug ยากมาก (รันซ้ำอีกทีอาจผ่านเฉย ๆ)

**วิธีแก้ที่ถูกต้อง** มีสองระดับ:

1. **แก้ที่ต้นเหตุ (แนะนำที่สุด)**: ออกแบบให้เทสแต่ละตัวใช้ resource ของตัวเองแยกกันเด็ดขาด เช่น ใช้ path ไฟล์ที่
   unique ต่อเทส (เช่นใช้ crate `tempfile` สร้างไฟล์ชั่วคราวที่ guarantee ไม่ชนกัน หรือใส่ชื่อเทสเข้าไปเป็นส่วน
   หนึ่งของ path) หรือถ้าเป็นไปได้ ออกแบบฟังก์ชันให้รับ path เป็น parameter แทนการ hardcode ไว้ข้างในเลย (ซึ่งยัง
   ทำให้โค้ด production เองก็ flexible ขึ้นด้วย เป็น win-win)
2. **แก้แบบเลี่ยงปัญหาชั่วคราว**: บังคับรันแบบ sequential ด้วย `cargo test -- --test-threads=1` — วิธีนี้ทำให้
   เทสที่แย่งกันแก้ shared state ไม่ชนกันอีก เพราะรันทีละตัวจริง ๆ แต่**เป็นการแก้ปัญหาที่ปลายเหตุ ไม่ใช่ต้นเหตุ**
   (เทสยังพึ่งพาลำดับการรันอยู่ดี ถ้าใครมาเพิ่มเทสใหม่ที่ไม่รู้ข้อจำกัดนี้ ปัญหาก็จะกลับมาอีก) ควรใช้เป็นทางออก
   เฉพาะกิจตอน debug เท่านั้น ไม่ใช่วิธีแก้ปัญหาระยะยาวของโค้ดจริง

**กฎทองที่ควรจำ**: เขียนเทสทุกตัวให้ทำงานได้ถูกต้องไม่ว่าจะถูกรันคนเดียว รันคู่กับเทสอื่นแบบ parallel หรือรันตาม
ลำดับไหนก็ตาม — ถ้าเทสของคุณ "ต้องรันก่อนเทสอื่น" หรือ "ต้องไม่รันพร้อมกับเทสอื่น" นั่นคือสัญญาณว่าการออกแบบเทส
(หรือบางทีการออกแบบโค้ดที่ทดสอบเอง) มีจุดที่ควรปรับปรุง

### 32.5 `assert!`, `assert_eq!`, `assert_ne!`: เลือกใช้ให้ถูกและอ่าน failure message ให้เป็น

Rust มี assertion macro หลักสามตัวที่ใช้ในเทส (และใช้นอกเทสก็ได้เหมือนกัน แต่ในทางปฏิบัติเจอเกือบทั้งหมดในเทส)

#### `assert!(condition)`: ตรวจสอบว่าเงื่อนไขเป็น `true`

```rust
fn is_even(n: i32) -> bool {
    n % 2 == 0
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn four_is_even() {
        assert!(is_even(4));
    }

    #[test]
    fn three_is_not_even() {
        assert!(!is_even(3));
    }
}
```

ถ้าเงื่อนไขเป็น `false`, `assert!` จะ `panic!` ด้วยข้อความบอก **expression ที่เขียนไว้ตรง ๆ** (ไม่ได้บอกค่าจริงที่
คำนวณได้) ตัวอย่างจริงจากการรันเทสที่ตั้งใจให้ fail (`result.contains("Hello")` เมื่อ `result` จริง ๆ คือ
`"สวัสดี สมชาย"` ไม่มีคำว่า `"Hello"` เลย):

```
thread 'bare_assert_demo::bare_assert_without_message_fails' panicked at src/lib.rs:125:9:
assertion failed: result.contains("Hello")
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

สังเกตว่าข้อความ `assertion failed: result.contains("Hello")` เป็นแค่ **การ echo กลับมาว่า expression ไหนที่ผล
เป็น false** — มันไม่ได้บอกเราเลยว่า `result` ที่ได้จริง ๆ **มีค่าเป็นอะไร** ทำให้ debug ยากกว่าที่ควร (ต้องเดา
หรือเปิดโค้ด/เพิ่ม `println!` เองเพื่อดูค่าจริง) นี่คือข้อจำกัดสำคัญของ `assert!` แบบเปล่า ๆ ที่ทำให้ `assert_eq!`
เหนือกว่ามากในกรณีที่กำลังเทียบค่าสองค่า

#### `assert_eq!(left, right)` และ `assert_ne!(left, right)`: จุดแข็งเรื่อง error message

```rust
#[test]
fn ten_percent_off_100_baht() {
    assert_eq!(apply_discount(10000, 10), 9000);
}
```

ถ้าค่าไม่ตรงกัน `assert_eq!` จะ `panic!` พร้อมแสดง**ทั้งค่าซ้ายและค่าขวาให้ดูเทียบกันตรง ๆ ทันที** โดยอัตโนมัติ
ไม่ต้องเขียนอะไรเพิ่มเลย ตัวอย่างจริงจากการรันเทสที่ตั้งใจให้ fail (`add_two(2, 2)` ให้ผล `4` แต่เทสคาดหวัง `5`):

```
thread 'tests::add_two_fails_on_purpose' panicked at src/lib.rs:37:9:
assertion `left == right` failed
  left: 4
 right: 5
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

เทียบกับ `assert!` เปล่า ๆ ที่ต้องเขียนแบบ `assert!(apply_discount(10000, 10) == 9000)` แล้วจะได้ข้อความแค่
`assertion failed: apply_discount(10000, 10) == 9000` (ไม่บอกว่าฝั่งซ้ายคำนวณได้ค่าอะไรจริง ๆ) — `assert_eq!`
**คำนวณค่าทั้งสองฝั่งแล้วเก็บไว้ใน internal variable ก่อน** จากนั้นค่อยเทียบและพิมพ์ค่าที่เก็บไว้นั้นออกมาให้ดู
โดยตรงในข้อความ error นี่คือเหตุผลที่ **ถ้าคุณกำลังเทียบสองค่าให้เท่ากัน ควรใช้ `assert_eq!` เสมอ แทน
`assert!(a == b)`** เพราะได้ debug information ที่ดีกว่ามากโดยไม่มีต้นทุนเพิ่มเลย (ทั้งสองใช้ syntax สั้นพอ ๆ กัน)

ตัวอย่างที่แสดงจุดแข็งของ `assert_eq!` ชัดเจนอีกกรณีคือ floating point:

```rust
#[test]
fn floating_point_fails() {
    let x = 0.1 + 0.2;
    assert_eq!(x, 0.3);
}
```

ผลลัพธ์จริงจากการรัน:

```
thread 'tests::floating_point_fails' panicked at src/lib.rs:53:9:
assertion `left == right` failed
  left: 0.30000000000000004
 right: 0.3
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

สังเกตว่าค่า `left` ที่พิมพ์ออกมาคือ `0.30000000000000004` (ค่าจริงของ `0.1 + 0.2` ใน `f64` binary floating point
representation ที่มีความคลาดเคลื่อนเล็กน้อยตามธรรมชาติของ IEEE 754) ทั้งที่เราคาดว่าจะได้ `0.3` เป๊ะ — เห็นค่าจริง
ตรงนี้ช่วยให้เข้าใจปัญหาได้ทันทีว่า "อ๋อ นี่คือปัญหา floating point precision ไม่ใช่บั๊กในสูตรคำนวณ" ซึ่งจะเป็น
ปัญหาที่วินิจฉัยยากกว่ามากถ้าใช้ `assert!` เปล่า ๆ ที่ไม่โชว์ค่าให้เห็น (เราจะพูดถึงวิธีเทียบ floating point อย่าง
ถูกต้องในหัวข้อกับดักที่พบบ่อยท้ายบท)

`assert_ne!` ทำงานตรงข้ามกับ `assert_eq!` — ใช้ตรวจสอบว่า **สองค่าต้องไม่เท่ากัน**:

```rust
#[test]
fn different_products_have_different_names() {
    let a = Product::new("เมาส์ไร้สาย", 29900);
    let b = Product::new("คีย์บอร์ด", 89000);
    assert_ne!(a.name, b.name);
}
```

ใช้บ่อยเวลาต้องการยืนยันว่าการดำเนินการบางอย่าง**เปลี่ยนแปลงค่าจริง** (เช่นหลังเรียก function ที่ควรสร้าง ID ใหม่
ทุกครั้ง ต้องมั่นใจว่า ID สองครั้งที่สร้างไม่ซ้ำกัน) หรือยืนยันว่า object สองตัวที่ควรจะต่างกันจริง ๆ ไม่ได้ถูก
สร้างมาเหมือนกันโดยไม่ตั้งใจ

**ทั้ง `assert_eq!` และ `assert_ne!` บังคับให้ type ของค่าที่เทียบต้อง implement ทั้ง `PartialEq` (สำหรับเทียบ
ความเท่ากัน) และ `Debug` (สำหรับพิมพ์ค่าออกมาตอน fail)** — นี่คือเหตุผลที่ struct ที่คุณสร้างเองต้องมี
`#[derive(Debug, PartialEq)]` ถ้าต้องการนำไปใช้กับ `assert_eq!`/`assert_ne!` โดยตรง (เราเห็น pattern นี้มาแล้วใน
Part 9 ตอนพูดถึง `#[derive(Debug)]`/`#[derive(PartialEq)]` — ตอนนั้นยังไม่ได้อธิบายว่าทำไมถึงจำเป็นสำหรับเทส
บทนี้คือคำตอบ) ถ้าลืม derive ตัวใดตัวหนึ่งไป compiler จะ error ทันทีตอน compile เทส เช่น
`error[E0277]: Product doesn't implement Debug` — เป็น compile error ที่ป้องกันบั๊กเชิงเครื่องมือ (คุณลืม derive)
ไม่ให้หลุดไปถึงตอนรันเทสจริง

### 32.6 ข้อความ error แบบกำหนดเอง (Custom Failure Messages)

ทั้ง `assert!`, `assert_eq!`, และ `assert_ne!` รับ argument เพิ่มเติมหลังเงื่อนไข/ค่าที่เทียบ ในรูปแบบเดียวกับ
`format!` macro (string literal พร้อม `{}`/`{name}` placeholder) เพื่อแนบบริบทเพิ่มเติมเข้าไปในข้อความ panic:

```rust
fn greeting(name: &str) -> String {
    format!("สวัสดี {name}")
}

#[test]
fn assert_with_custom_message_fails() {
    let name = "สมชาย";
    let result = greeting(name);
    assert!(
        result.contains("Hello"),
        "greeting({name}) ควรมีคำว่า 'Hello' แต่ได้ผลลัพธ์เป็น '{result}'"
    );
}
```

ผลลัพธ์จริงตอน fail:

```
thread 'tests::assert_with_custom_message_fails' panicked at src/lib.rs:44:9:
greeting(สมชาย) ควรมีคำว่า 'Hello' แต่ได้ผลลัพธ์เป็น 'สวัสดี สมชาย'
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

สังเกตว่าข้อความ **`assertion failed: ...`** แบบ default หายไปเลย ถูกแทนที่ด้วยข้อความที่เราเขียนเองทั้งหมด —
นี่คือพฤติกรรมที่ตั้งใจ: เมื่อคุณให้ custom message มา, `assert!` จะแสดง**เฉพาะ**ข้อความนั้น ไม่แสดง expression
ดั้งเดิมซ้ำอีก (ถ้าอยากให้เห็นทั้งสองอย่าง ต้องเขียน expression ลงในข้อความเองด้วยตรง ๆ)

Custom message มีประโยชน์มากที่สุดในสองสถานการณ์:

1. **เมื่อ `assert!` เปล่า ๆ ให้ข้อมูลไม่พอ** (อย่างที่เห็นในหัวข้อ 32.5 ว่า `assert!` ไม่โชว์ค่าจริงที่คำนวณได้) —
   การเติม custom message ที่ interpolate ค่าจริงเข้าไปด้วย (เหมือนตัวอย่างข้างบนที่ใส่ `{result}` เข้าไป) ทำให้
   ได้ debug information ที่เทียบเท่า `assert_eq!` แม้ว่าเงื่อนไขจะไม่ใช่การเทียบค่าเท่ากันตรง ๆ ก็ตาม (ในตัวอย่าง
   นี้คือการเช็ค `.contains(...)` ซึ่งไม่มี macro เฉพาะแบบ `assert_eq!` ให้ใช้)
2. **เมื่อต้องการอธิบาย "ทำไม" เงื่อนไขนี้ถึงสำคัญในเชิงธุรกิจ** — แม้ `assert_eq!` จะโชว์ค่าซ้าย/ขวาให้ดูแล้ว
   บางครั้งตัวเลขเปล่า ๆ ไม่พอที่จะเข้าใจว่า "ทำไมค่านี้ต้องตรงกับค่านั้น" การเติม custom message ก็ช่วยให้คนที่
   มาอ่าน failure message ในอนาคต (อาจเป็นตัวคุณเองอีกหกเดือนข้างหน้า หรือเพื่อนร่วมทีมที่ไม่คุ้นกับโค้ดส่วนนี้)
   เข้าใจบริบททันทีโดยไม่ต้องเปิดโค้ดไปอ่านทีละบรรทัด:

```rust
#[test]
fn discount_never_exceeds_hundred_percent_of_price() {
    let price = 10000;
    let result = apply_discount(price, 150); // ส่วนลดเกิน 100% ไม่สมเหตุสมผล
    assert!(
        result <= price,
        "ราคาหลังหักส่วนลด ({result}) ต้องไม่มากกว่าราคาตั้งต้น ({price}) เด็ดขาด \
         แต่สูตรคำนวณ apply_discount ให้ค่าที่มากกว่าราคาตั้งต้น ซึ่งไม่สมเหตุสมผลทางธุรกิจ"
    );
}
```

สำหรับ `assert_eq!`/`assert_ne!` วิธีการเหมือนกันทุกอย่าง แค่เติม argument ต่อจากค่าที่เทียบสองตัว:

```rust
#[test]
fn cart_total_matches_manual_calculation() {
    let mut cart = ShoppingCart::new();
    cart.add_item(Product::new("เมาส์ไร้สาย", 29900), 2).unwrap();

    let expected = 29900 * 2;
    assert_eq!(
        cart.total_cents(),
        expected,
        "ยอดรวมตะกร้าไม่ตรงกับที่คำนวณมือ (สินค้า 2 ชิ้น ราคาชิ้นละ 29900 สตางค์)"
    );
}
```

หมายเหตุ: **แม้จะใส่ custom message, `assert_eq!` ก็ยังคงแสดงค่า `left`/`right` ต่อจากข้อความที่เราเขียนเองเสมอ**
(ไม่ได้ถูกแทนที่แบบ `assert!`) เพราะการแสดงค่าเทียบเป็น "จุดขาย" หลักของ `assert_eq!` ที่ macro ไม่ยอมตัดออกแม้จะมี
custom message มาเสริมก็ตาม

### 32.7 `#[should_panic]`: พิสูจน์ว่าโค้ด "ปฏิเสธ" input ที่ไม่ถูกต้องอย่างถูกวิธี

จนถึงตอนนี้เราเขียนเทสที่ตรวจสอบว่า **"happy path" ให้ผลลัพธ์ถูกต้อง** แต่มีอีกครึ่งหนึ่งของการทดสอบที่สำคัญไม่แพ้
กันคือ **การพิสูจน์ว่าโค้ดปฏิเสธ input ที่ไม่ควรยอมรับอย่างถูกต้องด้วย** ในโค้ด Rust จำนวนมาก การปฏิเสธ input ที่
ผิดพลาดร้ายแรงมักทำผ่าน `panic!` (แทนที่จะคืน `Result::Err` — เมื่อไหร่ควรเลือกแบบไหนเป็นเนื้อหาที่ Part 30/31
พูดถึงไปแล้ว) และ Rust มี attribute พิเศษสำหรับเทสกรณีนี้โดยเฉพาะคือ `#[should_panic]`

```rust
pub fn withdraw(balance: u32, amount: u32) -> u32 {
    if amount > balance {
        panic!("ยอดเงินไม่พอ: มี {balance} แต่ขอถอน {amount}");
    }
    balance - amount
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn withdraw_normal_amount_works() {
        assert_eq!(withdraw(1000, 300), 700);
    }

    #[test]
    #[should_panic]
    fn withdraw_too_much_panics() {
        withdraw(100, 500); // ต้อง panic เพราะขอถอนมากกว่าที่มี
    }
}
```

`#[should_panic]` **กลับตรรกะ pass/fail ของเทสนั้นตัวเดียว**: เทสนี้จะถูกรายงานว่า **`ok`** ก็ต่อเมื่อ**เกิด
`panic!` ขึ้นจริงระหว่างรัน** — ถ้าฟังก์ชันรันจบโดย**ไม่** panic เลย เทสนี้จะถูกรายงานว่า **`FAILED`** (สลับด้าน
กับเทสปกติทุกตัวที่ pass เมื่อไม่ panic) ผลลัพธ์จริงตอนรัน:

```
test tests::withdraw_too_much_panics - should panic ... ok
```

สังเกตข้อความ `- should panic` ต่อท้ายชื่อเทส — เป็นการบอกให้รู้ในรายงานว่าเทสนี้มีเงื่อนไข pass/fail แบบพิเศษ ไม่
เหมือนเทสทั่วไป

**ทำไมการเทสแบบนี้จึงสำคัญ?** เพราะการ "ปฏิเสธ input ที่ผิด" เป็นส่วนหนึ่งของ contract ของฟังก์ชันไม่น้อยไปกว่า
"ให้ผลลัพธ์ถูกต้องกับ input ที่ถูก" ลองนึกภาพว่ามีคนมา refactor ฟังก์ชัน `withdraw` แล้วเผลอลบเงื่อนไขตรวจสอบออกไป
โดยไม่ตั้งใจ:

```rust
// หลัง refactor แบบผิดพลาด — ลืมเช็คเงื่อนไข amount > balance ไปเลย
pub fn withdraw(balance: u32, amount: u32) -> u32 {
    balance - amount // ถ้า amount > balance จะเกิด integer underflow (panic ใน debug build, wrap around ใน release)
}
```

ถ้าไม่มีเทส `withdraw_too_much_panics` ไว้เตือน โค้ดที่ผิดนี้จะไม่ถูกจับได้จนกว่าจะมีคนไปเจอ behavior แปลก ๆ ใน
production (หรือแย่กว่านั้นคือไม่มีใครเจอเลย เพราะ release build ที่ปิด overflow check ไว้จะทำให้ `balance -
amount` แค่ wrap around กลับไปเป็นตัวเลขบวกมหาศาลแบบเงียบ ๆ ไม่ panic ให้เห็นด้วยซ้ำ) แต่ถ้ามีเทสนี้ไว้
`cargo test` จะ **FAILED ทันที** เพราะฟังก์ชันไม่ panic แบบที่คาดไว้อีกต่อไป (มันแค่คืนค่าผิด ๆ เงียบ ๆ) — นี่คือ
เหตุผลที่ `#[should_panic]` ควรถูกใช้เป็น**คู่กัน**กับเทส happy path เสมอสำหรับฟังก์ชันที่ปฏิเสธ input ผิดด้วย panic
เพื่อยืนยันว่า guard เงื่อนไขยังทำงานอยู่จริงในทุกครั้งที่มีการแก้ไขโค้ด

#### `expected = "..."`: ตรวจสอบ**ข้อความ**ของ panic ด้วย ไม่ใช่แค่ "panic เกิดขึ้นหรือไม่"

`#[should_panic]` แบบเปล่า ๆ มีข้อจำกัดสำคัญ: มันจะ pass **ถ้า panic เกิดขึ้นจากสาเหตุใดก็ตาม** แม้จะเป็นสาเหตุที่
ไม่เกี่ยวกับที่เราตั้งใจทดสอบเลยก็ตาม ลองดูตัวอย่างที่มีปัญหาแบบนี้:

```rust
pub fn withdraw(balance: u32, amount: u32) -> u32 {
    if amount > balance {
        panic!("ยอดเงินไม่พอ: มี {balance} แต่ขอถอน {amount}");
    }
    balance - amount
}

#[test]
#[should_panic] // เทสนี้ "ผ่าน" ได้แม้ panic จะมาจากบั๊กคนละตัวโดยสิ้นเชิง!
fn withdraw_too_much_panics_but_test_is_weak() {
    // สมมติมีคนพิมพ์ผิดตรงนี้ (เขียน index array ที่ไม่มีอยู่จริงแทนการเรียก withdraw)
    let accounts = vec![100, 200, 300];
    let _ = accounts[99]; // panic จาก index out of bounds — คนละสาเหตุกับที่ตั้งใจทดสอบเลย!
}
```

เทสนี้จะรายงานว่า **`ok`** เพราะเกิด panic ขึ้นจริง (จาก index out of bounds) แต่มัน**ไม่ได้พิสูจน์อะไรเกี่ยวกับ
`withdraw` เลยแม้แต่นิดเดียว** — ถ้าฟังก์ชัน `withdraw` มีบั๊กจริง ๆ (เช่นไม่ panic ตอนควร panic) เทสนี้ก็จะยังคง
"ผ่าน" อยู่ดี เพราะมันไปเจอ panic จากที่อื่นแทน นี่คือ **false positive** ที่อันตรายมาก เพราะทำให้ทีมเข้าใจผิดว่า
"มีเทสคุ้มครองอยู่แล้ว" ทั้งที่จริง ๆ ไม่มีการคุ้มครองอะไรเลย

วิธีแก้คือใส่ parameter `expected` เพื่อระบุว่า **panic message ต้องมี substring นี้อยู่ด้วย** ถึงจะถือว่าเทส
ผ่าน:

```rust
#[test]
#[should_panic(expected = "ยอดเงินไม่พอ")]
fn withdraw_too_much_panics_with_expected_message() {
    withdraw(100, 500);
}
```

ตอนนี้ถ้า panic เกิดจากสาเหตุอื่นที่ไม่มีคำว่า `"ยอดเงินไม่พอ"` ในข้อความเลย เทสจะ **FAILED** แม้ panic จะเกิดขึ้น
จริงก็ตาม ลองดูตัวอย่างที่ตั้งใจให้ข้อความคาดหวังไม่ตรง เพื่อดู failure message จริง:

```rust
#[test]
#[should_panic(expected = "ข้อความที่ไม่ตรงกับ panic จริง")]
fn should_panic_wrong_expected_message_fails() {
    withdraw(100, 500);
}
```

ผลลัพธ์จริงจากการรัน:

```
thread 'tests::should_panic_wrong_expected_message_fails' panicked at src/lib.rs:15:9:
ยอดเงินไม่พอ: มี 100 แต่ขอถอน 500
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
note: panic did not contain expected string
      panic message: "ยอดเงินไม่พอ: มี 100 แต่ขอถอน 500"
 expected substring: "ข้อความที่ไม่ตรงกับ panic จริง"
```

สังเกตสามบรรทัดสุดท้าย: **`note: panic did not contain expected string`** พร้อมโชว์ทั้ง `panic message` ที่เกิด
ขึ้นจริง และ `expected substring` ที่เราระบุไว้ให้ดูเทียบกันตรง ๆ — ทำให้เห็นชัดเจนทันทีว่าปัญหาคือข้อความไม่ตรงกัน
ไม่ใช่ปัญหาว่า "panic เกิดขึ้นหรือไม่" (เพราะ panic เกิดขึ้นจริง เห็นได้จากบรรทัดบนสุด)

> **หมายเหตุเรื่อง encoding**: ถ้าคุณลองรันโค้ดตัวอย่างนี้เองแล้วสังเกตเห็นว่าบางครั้งข้อความ Thai ที่มีวรรณยุกต์/
> สระลอย (เช่น ่, ้, ึ) แสดงเป็น `\u{e48}` แทนตัวอักษรจริงในผลลัพธ์ของ `cargo test` — นั่นไม่ใช่บั๊ก เป็นเพราะ trait
> `Debug` ของ Rust สำหรับ `str`/`String` จะ escape ตัวอักษรที่ถูกจัดว่า "ไม่ใช่ตัวอักษรที่พิมพ์ได้ด้วยตัวเอง"
> (non-printable ตามเกณฑ์ภายในของ `core`) ซึ่งรวม combining diacritical mark ของบางภาษาด้วย (สระ/วรรณยุกต์ไทย
> บางตัวถูกจัดอยู่ในกลุ่มนี้เพราะมันเป็นสัญลักษณ์ที่ "ประกอบ" กับตัวอักษรอื่น ไม่ใช่ตัวอักษรที่แสดงผลได้ตัวเดียว) —
> สิ่งนี้จะเกิดเฉพาะตอนพิมพ์ผ่าน `{:?}` (Debug format) เท่านั้น ถ้าพิมพ์ผ่าน `{}` (Display format) ปกติ ข้อความจะ
> แสดงถูกต้องสมบูรณ์เสมอ ข้อความ panic ที่แสดงในหัวข้อนี้ (ผ่าน `panic!` และ `println!`) ใช้ Display format จึงไม่มี
> ปัญหานี้ แต่บางที่ในบทนี้ (หัวข้อ 32.8) จะเจอกรณีที่ escape เกิดขึ้นจริง เพราะ libtest เลือกใช้ Debug format ใน
> จุดนั้น

### 32.8 เทสที่คืนค่า `Result<(), E>`: ใช้ `?` แทน `.unwrap()` เกลื่อนเทส

จาก Part 12 (หัวข้อ 12.8) เราเรียนรู้จัก pattern `fn main() -> Result<(), Box<dyn Error>>` ที่ทำให้เขียน `?` ใน
`main()` ได้โดยตรง แทนที่จะต้อง `.unwrap()`/`.expect()` ทุกจุดที่อาจ error — **หลักการเดียวกันนี้ใช้กับฟังก์ชัน
เทสได้เป๊ะ ๆ** เพราะ Rust test framework รองรับให้ฟังก์ชัน `#[test]` **คืนค่าชนิดใดก็ได้ที่ implement trait
`std::process::Termination`** ซึ่ง `()` และ `Result<T, E>` (โดยที่ `E: Debug`) ทั้งสองแบบ implement trait นี้ไว้
ให้แล้วโดย standard library

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn result_based_test_ok() -> Result<(), String> {
        if add_two(2, 2) == 4 {
            Ok(())
        } else {
            Err(String::from("add_two(2,2) ไม่เท่ากับ 4"))
        }
    }
}
```

ตรรกะการตัดสิน pass/fail สำหรับเทสแบบนี้คือ: **คืน `Ok(())` → pass, คืน `Err(_)` → fail** (โดยไม่ต้อง `panic!`
เลยแม้แต่ครั้งเดียว) — นี่เปิดโอกาสให้เขียนเทสที่เรียกฟังก์ชันหลายตัวที่คืน `Result` ต่อ ๆ กันด้วย `?` ได้อย่าง
กระชับ แทนที่จะต้อง `.unwrap()` ทุกจุด:

```rust
use std::collections::HashMap;

fn parse_price_from_catalog(catalog: &HashMap<&str, &str>, sku: &str) -> Result<u32, String> {
    let raw = catalog
        .get(sku)
        .ok_or_else(|| format!("ไม่พบ SKU '{sku}' ในแคตตาล็อก"))?;
    raw.parse::<u32>()
        .map_err(|e| format!("แปลงราคาของ '{sku}' เป็นตัวเลขไม่ได้: {e}"))
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_valid_catalog_entries() -> Result<(), String> {
        let mut catalog = HashMap::new();
        catalog.insert("SKU-1", "29900");
        catalog.insert("SKU-2", "89000");

        // ใช้ ? แทน .unwrap() ทุกจุด — ถ้าจุดใดจุดหนึ่ง Err ขึ้นมา เทสจะ fail ทันที
        // พร้อมข้อความ Err ที่มีบริบทชัดเจน แทนที่จะเห็นแค่ "called Result::unwrap() on an Err value"
        let price1 = parse_price_from_catalog(&catalog, "SKU-1")?;
        let price2 = parse_price_from_catalog(&catalog, "SKU-2")?;

        assert_eq!(price1, 29900);
        assert_eq!(price2, 89000);
        Ok(())
    }
}
```

**ข้อดีเทียบกับการใช้ `.unwrap()` ทุกจุด**: ถ้าเราเขียนแบบเดิมด้วย `.unwrap()`:

```rust
#[test]
fn parses_valid_catalog_entries_with_unwrap() {
    let mut catalog = HashMap::new();
    catalog.insert("SKU-1", "29900");

    let price1 = parse_price_from_catalog(&catalog, "SKU-1").unwrap();
    assert_eq!(price1, 29900);
}
```

ถ้า `parse_price_from_catalog` คืน `Err` ขึ้นมาโดยไม่คาดคิด (เช่นเพราะเราพิมพ์ SKU ผิด) `.unwrap()` จะ panic ด้วย
ข้อความทั่วไปแบบ `called \`Result::unwrap()\` on an \`Err\` value: "..."` ซึ่งก็ยังพอใช้ได้ แต่ **`?` ให้ผลลัพธ์
คล้ายกันโดยไม่ต้องเขียน `.unwrap()` ซ้ำ ๆ ทุกบรรทัดที่มีการเรียก function ที่คืน `Result`** — ในเทสที่ต้องเรียก
หลาย operation ต่อกันเป็นสิบบรรทัด ความกระชับนี้ต่างกันมาก และยิ่งสำคัญขึ้นเมื่อ error type ของแต่ละจุดต้องแปลง
กันไปมา (ซึ่ง `?` จัดการให้ผ่าน `From`/`Into` แบบเดียวกับที่เรียนใน Part 30 เรื่อง custom error type)

ลองดู failure message จริงของเทสแบบ `Result<(), E>` เมื่อ fail (สังเกตความแตกต่างจาก panic-based failure ในหัวข้อ
ก่อน ๆ):

```rust
#[test]
fn result_based_test_fails() -> Result<(), String> {
    if add_two(2, 2) == 5 {
        Ok(())
    } else {
        Err(format!("คาดว่า 5 แต่ add_two(2,2) ให้ {}", add_two(2, 2)))
    }
}
```

ผลลัพธ์จริง:

```
running 1 test
test tests::result_based_test_fails ... FAILED

failures:

---- tests::result_based_test_fails stdout ----
Error: "คาดว่\u{e48}า 5 แต\u{e48} add_two(2,2) ให\u{e49} 4"

failures:
    tests::result_based_test_fails

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 12 filtered out; finished in 0.00s
```

สังเกตความแตกต่างสำคัญสองจุดจาก failure ที่มาจาก `panic!`/`assert_eq!` ที่เห็นในหัวข้อก่อน ๆ:

1. **ไม่มี `thread '...' panicked at ...`** เลย — เพราะเทสนี้ไม่ได้ panic แต่จบด้วยการคืน `Err(...)` ตามปกติ
   libtest จับ `Err` variant แล้วรายงานว่า `FAILED` โดยไม่ต้องพึ่งกลไก panic-catching เลย
2. **ข้อความขึ้นต้นด้วย `Error:`** ตามด้วยค่าของ `Err` ที่พิมพ์ด้วย **Debug format** (`{:?}`) — นี่คือที่มาของ
   `\u{e48}`/`\u{e49}` ที่เห็นในข้อความ (ตามที่อธิบายไว้ในกล่องหมายเหตุท้ายหัวข้อ 32.7): เพราะ libtest เลือกพิมพ์
   ค่า `E` ของ `Err(E)` ด้วย `{:?}` เสมอ (เนื่องจาก `E` ต้อง implement เพียง `Debug` เท่านั้นตาม bound ของ
   `Termination`, ไม่ได้บังคับ `Display`) ทำให้ string ที่มีสระ/วรรณยุกต์ไทยบางตัวถูก escape เป็น unicode
   escape sequence แบบที่เห็น — ค่าที่ escape ไปนั้นยังคงถูกต้อง 100% เพียงแค่ "หน้าตา" ตอนแสดงผลผ่าน Debug ไม่
   สวยเท่าตอนแสดงผลผ่าน Display เท่านั้นเอง (ถ้าอ่านดี ๆ จะเห็นว่าข้อความยังอ่านออกครบถ้วน แค่มีอักขระบางตัวถูก
   สลับเป็นรหัส unicode)

**ข้อจำกัดที่ต้องรู้**: เทสแบบ `Result<(), E>` **ใช้กับ `#[should_panic]` ร่วมกันไม่ได้** (เพราะตรรกะขัดกันเอง —
`#[should_panic]` ต้องการให้ฟังก์ชัน panic ถึงจะ pass แต่เทสแบบ `Result` ไม่ panic เลยด้วยตัวมันเอง มันจบด้วยการ
`return` ค่าปกติ) ถ้าต้องการทดสอบว่าโค้ด panic ให้ใช้ `#[should_panic]` กับฟังก์ชันที่คืน `()` ตามหัวข้อ 32.7
แทน — ทั้งสองรูปแบบแก้ปัญหาคนละแบบ: `Result<(), E>` เหมาะกับ "เทสที่มีหลาย operation ที่อาจ error ระหว่างทาง
และอยากใช้ `?`", ส่วน `#[should_panic]` เหมาะกับ "เทสที่ยืนยันว่าโค้ด panic ในกรณีที่ควร panic"

### 32.9 `#[ignore]`: เทสที่ช้า/แพง และการรันเฉพาะตอนต้องการจริง ๆ

บางเทสใช้เวลานานผิดปกติ (เช่นทดสอบ performance กับ dataset ขนาดใหญ่, ทดสอบที่ต้อง sleep รอ timeout จริง) หรือ
ต้องพึ่งพา resource ที่ไม่มีเสมอ (เช่น network connection, ฐานข้อมูลจริงที่ไม่ได้ตั้งอยู่ในเครื่อง dev ทุกเครื่อง)
— เทสประเภทนี้ถ้าให้รันทุกครั้งพร้อมกับเทสอื่นทั้งหมดจะทำให้ loop เขียน-เทสช้าลงมาก ทั้งที่ระหว่างพัฒนาฟีเจอร์
ทั่วไปไม่ได้ต้องการผลจากเทสเหล่านี้บ่อยขนาดนั้น

`#[ignore]` ทำให้เทสนั้น**ไม่ถูกรัน**ใน `cargo test` แบบปกติ แต่ยังคง compile และรอเรียกใช้เมื่อต้องการจริง ๆ:

```rust
fn slow_calculation(n: u64) -> u64 {
    std::thread::sleep(std::time::Duration::from_millis(50));
    n * 2
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    #[ignore]
    fn expensive_slow_test() {
        let result = slow_calculation(21);
        assert_eq!(result, 42);
    }
}
```

รัน `cargo test` แบบปกติ:

```
running 13 tests
test tests::expensive_slow_test ... ignored
...
test result: ok. 7 passed; 0 failed; 1 ignored; 0 measured; 5 filtered out; finished in 0.10s
```

สังเกตตัวเลข **`1 ignored`** ในบรรทัดสรุป — บอกให้รู้ว่ามีเทสถูก mark `#[ignore]` อยู่ 1 ตัว แต่**ไม่ได้นับเป็น
failed หรือถูก skip เพราะปัญหาอะไร** เป็นแค่การรายงานให้รู้ตัวเลขตรงไปตรงมา

เมื่อต้องการรันเฉพาะเทสที่ถูก ignore ไว้ (เช่นก่อน merge เข้า main branch ที่อยากรันเทสทุกตัวรวมตัวที่ช้าด้วย):

```bash
cargo test -- --ignored
```

ผลลัพธ์จริง:

```
running 1 test
test tests::expensive_slow_test ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 12 filtered out; finished in 0.05s
```

สังเกตว่าตอนนี้ `expensive_slow_test` ถูกรันจริง (ใช้เวลา 0.05s ตามที่ตั้ง `sleep(50ms)` ไว้) และเทสอื่นที่**ไม่ได้**
ถูก mark `#[ignore]` กลับกลายเป็น "filtered out" แทน — เพราะ `--ignored` หมายถึง **"รันเฉพาะเทสที่ถูก ignore
เท่านั้น"** ไม่ใช่ "รันทุกอย่างรวมเทสที่ถูก ignore ด้วย" ถ้าต้องการรัน**ทุกเทสรวมทั้งที่ ignore ไว้ในครั้งเดียว**
ใช้ flag `--include-ignored` แทน:

```bash
cargo test -- --include-ignored
```

**เมื่อไหร่ควรใช้ `#[ignore]`**: เทสที่ช้ากว่าปกติมาก (มากกว่าเสี้ยววินาที ในระดับที่รบกวน feedback loop ระหว่าง
พัฒนา), เทสที่ต้องพึ่งพา external resource ที่ไม่ใช่ทุกเครื่องจะมี (ควรพิจารณาย้ายไปเป็น integration test ที่มี
setup แยกต่างหากด้วยใน Part 33 ก่อนด้วยซ้ำ), หรือเทสที่รู้อยู่แล้วว่า fail อยู่ชั่วคราวเพราะรอ dependency ตัวอื่น
แก้ไข (ใช้ `#[ignore = "เหตุผล"]` เพื่อบันทึกเหตุผลไว้ในโค้ดเลย เช่น `#[ignore = "รอ API endpoint ใหม่จากทีม backend"]`)
— ไม่ควรใช้ `#[ignore]` เพื่อ "ซ่อน" เทสที่ fail เพราะขี้เกียจแก้ เพราะนั่นทำให้ test suite เสียความน่าเชื่อถือไป
เรื่อย ๆ (เทสที่ไม่มีใครรันคือเทสที่ไม่มีค่าอะไรเลย)

### 32.10 ทดสอบฟังก์ชัน private ได้: ข้อดีที่ Rust มีเหนือหลายภาษา และ trade-off ที่ต้องรู้

เราพูดถึงประเด็นนี้ไปบ้างแล้วในหัวข้อ 32.3 — เพราะ `mod tests` เป็น module ลูกของไฟล์ที่มันอยู่ มันจึงมองเห็นและ
เรียกฟังก์ชัน private ได้โดยตรงเหมือนเป็น item ธรรมดาในไฟล์เดียวกัน มาดูตัวอย่างที่ชัดเจนขึ้น:

```rust
// src/pricing.rs

pub struct Order {
    pub subtotal_cents: u32,
    pub is_member: bool,
}

/// คำนวณค่าจัดส่ง — เป็น implementation detail ภายในที่ผู้ใช้ crate ไม่ควรเรียกตรง ๆ
/// (ผู้ใช้ควรเรียกผ่าน total_with_shipping() ที่เป็น public API เท่านั้น)
fn calculate_shipping_fee(order: &Order) -> u32 {
    if order.is_member {
        0 // สมาชิกส่งฟรีเสมอ
    } else if order.subtotal_cents >= 100000 {
        0 // ยอดซื้อเกิน 1000 บาท ส่งฟรี
    } else {
        4000 // ค่าส่งปกติ 40 บาท
    }
}

pub fn total_with_shipping(order: &Order) -> u32 {
    order.subtotal_cents + calculate_shipping_fee(order)
}

#[cfg(test)]
mod tests {
    use super::*;

    // เทส public API ตามปกติ — มุมมองเดียวกับที่ผู้ใช้ crate จะเห็น
    #[test]
    fn member_gets_free_shipping_via_public_api() {
        let order = Order { subtotal_cents: 5000, is_member: true };
        assert_eq!(total_with_shipping(&order), 5000);
    }

    // เทสฟังก์ชัน private ตรง ๆ — ทำได้เพราะอยู่ module เดียวกัน (module ลูก)
    #[test]
    fn calculate_shipping_fee_directly_for_non_member_below_threshold() {
        let order = Order { subtotal_cents: 5000, is_member: false };
        assert_eq!(calculate_shipping_fee(&order), 4000);
    }

    #[test]
    fn calculate_shipping_fee_directly_for_non_member_above_threshold() {
        let order = Order { subtotal_cents: 150000, is_member: false };
        assert_eq!(calculate_shipping_fee(&order), 0);
    }
}
```

ความสามารถนี้เป็นข้อได้เปรียบจริงเมื่อเทียบกับหลายภาษาที่แยกไฟล์เทสออกจากไฟล์ implementation อย่างเด็ดขาด (เช่น
Java ที่เทสอยู่คนละไฟล์ในโฟลเดอร์ `src/test/java/` และเข้าไม่ถึง `private` method ได้เลยนอกจากใช้ reflection ซึ่ง
เป็นเทคนิคซับซ้อนและเปราะบาง หรือ Python ที่แม้ technically เข้าถึงได้เพราะไม่มี true privacy แต่ convention
`_leading_underscore` ก็บอกใบ้ว่า "ไม่ควรแตะจากข้างนอก") ใน Rust การทดสอบ implementation detail ทำได้ **ง่ายพอ
กับการทดสอบ public API เป๊ะ ๆ** โดยไม่ต้องเปลี่ยน visibility เพื่อ "เปิดช่อง" ให้เทสเข้าถึงเลยแม้แต่นิดเดียว

#### แต่มี trade-off ที่ต้องเข้าใจ: เทสที่ผูกกับ implementation detail เปราะบางกว่า

การเทส private function ตรง ๆ มีข้อดีชัดเจนคือ **จับบั๊กได้ใกล้ต้นเหตุที่สุด** — ถ้า `calculate_shipping_fee` มี
บั๊ก เทสที่เรียกมันตรง ๆ จะบอกได้ทันทีว่าปัญหาอยู่ที่ฟังก์ชันนี้เอง ไม่ต้องมานั่งวิเคราะห์ผ่าน `total_with_shipping`
ที่อาจมีตัวแปรอื่นปนเข้ามาด้วย (เช่นถ้า `total_with_shipping` เองก็มีบั๊กเรื่องการบวกเลข การเทสผ่าน public API
เพียงอย่างเดียวอาจทำให้บั๊กสองตัวหักลบกันเองจนดูเหมือนผ่าน ทั้งที่จริง ๆ มีบั๊กซ่อนอยู่สองที่)

แต่ข้อเสียคือ **เทสที่ผูกกับ implementation detail จะ "เปราะบางต่อการ refactor" มากกว่า** — ถ้าวันหนึ่งคุณ
ตัดสินใจ refactor `calculate_shipping_fee` ให้เปลี่ยนชื่อ, เปลี่ยน signature, หรือรวมมันเข้ากับฟังก์ชันอื่น (เพราะ
มันเป็นแค่ implementation detail ที่ "ควร" เปลี่ยนได้เสรีตราบใดที่ public API ยังทำงานถูกต้องเหมือนเดิม) เทสที่
เรียกมันตรง ๆ จะพังทันที (compile error เพราะฟังก์ชันหายไป หรือชื่อเปลี่ยน) **ทั้งที่ public API ยังทำงานถูกต้อง
สมบูรณ์แบบทุกอย่าง** คุณจะต้องมานั่งแก้เทสเหล่านั้นทุกครั้งที่ refactor internal structure แม้ว่า behavior ที่
ผู้ใช้เห็นจะไม่เปลี่ยนแปลงเลยก็ตาม — นี่คือสิ่งที่ชุมชน testing เรียกว่า **"testing implementation, not behavior"**
ซึ่งถือเป็นกลิ่นของการออกแบบเทสที่ไม่ค่อยดี ในระดับที่รุนแรงมันจะทำให้ทีมกลัวการ refactor (เพราะรู้ว่าต้องแก้เทส
เพียบ) ซึ่งขัดกับเจตนาดั้งเดิมของเทส (ที่ควรจะ "ทำให้กล้า refactor มากขึ้น" ไม่ใช่ "ทำให้กลัว refactor")

**แนวทางที่หลักสูตรนี้แนะนำเป็นหลัก (rule of thumb)**:

- **เทสผ่าน public API เป็นค่าเริ่มต้นเสมอ** เพราะมันเทส "behavior ที่ผู้ใช้จริงพึ่งพา" ซึ่งคือสิ่งที่สำคัญที่สุด
  และทนต่อการ refactor internal ได้ดีที่สุด (ตราบใดที่ public API contract ไม่เปลี่ยน เทสก็ไม่ต้องแก้)
- **เทสฟังก์ชัน private ตรง ๆ เฉพาะเมื่อมีเหตุผลเจาะจงจริง ๆ** เช่น: ฟังก์ชันนั้นมี logic ซับซ้อนมาก (มีหลาย edge
  case ที่อยากเทสละเอียด) แต่การ trigger ทุก edge case ผ่าน public API ทำได้ยากหรือต้อง setup ซับซ้อนเกินจำเป็น,
  หรือฟังก์ชันนั้นเป็น pure function ที่แยกออกมาเพื่อให้ทดสอบได้ง่ายโดยเฉพาะ (pattern ที่พบบ่อย: แยก "การคำนวณ"
  ออกจาก "การเรียกใช้จริง" เพื่อให้เทสส่วนคำนวณได้ตรง ๆ โดยไม่ต้อง mock อะไรเลย)
- เมื่อเทส private function ตรง ๆ ให้คิดไว้เสมอว่านี่คือ **การลงทุนที่มีต้นทุนการดูแลรักษาเพิ่มขึ้น** — ไม่ผิดที่จะ
  ทำ แต่ควรทำอย่างมีสติว่ากำลังแลก "ความละเอียดของการทดสอบ" กับ "ความคล่องตัวในการ refactor ในอนาคต"

### 32.11 จัดระเบียบ Test Suite ที่โตขึ้น: submodule, helper function, และ fixture

เมื่อโค้ดโตขึ้น จำนวนเทสก็โตตามไปด้วยเป็นธรรมดา — module `tests` เดียวที่มีเทสยี่สิบสามสิบตัวเรียงกันจะอ่านยาก
มาดูสองเทคนิคหลักที่ช่วยให้ test suite ขนาดใหญ่ยังอ่านง่ายและดูแลง่าย

#### แบ่ง submodule ตามหมวดหมู่

`mod tests` เป็น module ธรรมดา คุณจึงนิยาม `mod` ซ้อนลงไปข้างในมันได้อีกตามปกติ (module tree ซ้อนกันได้หลายชั้น
ตามที่เรียนใน Part 16) เพื่อแบ่งเทสเป็นหมวดหมู่ที่ชัดเจน:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    mod happy_path {
        use super::*;

        #[test]
        fn adding_one_item_updates_totals() {
            let mut cart = ShoppingCart::new();
            cart.add_item(Product::new("เมาส์ไร้สาย", 29900), 2).unwrap();
            assert_eq!(cart.total_cents(), 59800);
        }
    }

    mod edge_cases {
        use super::*;

        #[test]
        fn adding_zero_quantity_returns_err() {
            let mut cart = ShoppingCart::new();
            let result = cart.add_item(Product::new("เมาส์ไร้สาย", 29900), 0);
            assert!(result.is_err());
        }
    }

    mod panics {
        use super::*;

        #[test]
        #[should_panic(expected = "empty cart")]
        fn checkout_on_empty_cart_panics() {
            let cart = ShoppingCart::new();
            cart.checkout();
        }
    }
}
```

สังเกตว่าแต่ละ submodule ต้องมี `use super::*;` เป็นของตัวเอง (เพื่อดึง item จาก module พ่อคือ `tests` — ซึ่งตัวมัน
เองก็ดึงมาจาก module พ่อของมันอีกทีจาก `use super::*;` ที่บรรทัดบนสุด) การแบ่งแบบนี้มีประโยชน์สองทาง: **(1)** ชื่อ
เทสในรายงานของ `cargo test` จะมี path ที่สื่อความหมายมากขึ้น (เช่น `tests::edge_cases::adding_zero_quantity_...`
อ่านแล้วรู้ทันทีว่าเป็นเทส edge case) และ **(2)** สามารถใช้ path นั้นเป็น filter ตอนรัน `cargo test` ได้ตรงเป้า
มากขึ้น เช่น `cargo test edge_cases` จะรันเฉพาะเทสในกลุ่ม edge case ทั้งหมดโดยไม่ต้องพิมพ์ชื่อเทสแต่ละตัว

#### Helper / Fixture function: ลด code duplication ใน setup

เมื่อหลายเทสต้องการ "จุดเริ่มต้น" แบบเดียวกัน (เช่น ตะกร้าที่มีสินค้าอยู่แล้วสองสามรายการ) การ copy-paste โค้ด
setup ซ้ำ ๆ ในทุกเทสทำให้ดูแลยากขึ้นเรื่อย ๆ (ถ้าวันหนึ่ง `Product::new` เปลี่ยน signature ต้องไปแก้ทุกที่ที่
copy-paste ไว้) แนวทางที่ดีกว่าคือแยกเป็น **helper function** (บางครั้งเรียกว่า **test fixture**) ที่เทสหลายตัว
เรียกใช้ร่วมกัน:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    /// สร้างตะกร้าที่มีสินค้ามาตรฐาน 2 รายการไว้ล่วงหน้า ใช้ซ้ำได้หลายเทสที่ต้องการ "ตะกร้าที่ไม่ว่าง" เป็นจุดเริ่มต้น
    fn setup_cart_with_two_products() -> ShoppingCart {
        let mut cart = ShoppingCart::new();
        cart.add_item(Product::new("เมาส์ไร้สาย", 29900), 2).unwrap();
        cart.add_item(Product::new("คีย์บอร์ด", 89000), 1).unwrap();
        cart
    }

    #[test]
    fn setup_fixture_has_expected_total() {
        let cart = setup_cart_with_two_products();
        assert_eq!(cart.total_cents(), 29900 * 2 + 89000);
    }

    #[test]
    fn setup_fixture_removing_one_leaves_the_other() {
        let mut cart = setup_cart_with_two_products();
        cart.remove_item("คีย์บอร์ด");
        assert_eq!(cart.distinct_product_count(), 1);
        assert_eq!(cart.total_cents(), 29900 * 2);
    }
}
```

สังเกตว่า `setup_cart_with_two_products` **ไม่มี `#[test]` attribute** — มันเป็นฟังก์ชันธรรมดาที่อยู่ใน module
`tests` เฉย ๆ ไม่ได้ถูกนับเป็น test case เอง (ถ้าลองรัน `cargo test` จะไม่เห็นชื่อนี้ในรายงานเลย) มันมีหน้าที่
เดียวคือ **ถูกเรียกจากภายในฟังก์ชันเทสตัวอื่น** เพื่อลดความซ้ำซ้อน — pattern นี้พบได้บ่อยมากในโปรเจกต์ Rust จริง
และในเทสที่ซับซ้อนกว่านี้ ฟังก์ชัน setup อาจคืนค่าเป็น struct พิเศษที่รวม "ข้อมูลทดสอบ" หลายอย่างไว้ด้วยกัน (บางครั้ง
เรียกว่า **test context** หรือ **fixture struct**):

```rust
struct TestData {
    cart: ShoppingCart,
    expected_total_cents: u32,
}

fn setup() -> TestData {
    let mut cart = ShoppingCart::new();
    cart.add_item(Product::new("เมาส์ไร้สาย", 29900), 2).unwrap();
    cart.add_item(Product::new("คีย์บอร์ด", 89000), 1).unwrap();

    TestData {
        expected_total_cents: 29900 * 2 + 89000,
        cart,
    }
}

#[test]
fn setup_struct_pattern_example() {
    let data = setup();
    assert_eq!(data.cart.total_cents(), data.expected_total_cents);
}
```

รูปแบบ `fn setup() -> TestData { ... }` แบบนี้มีประโยชน์เพิ่มขึ้นเมื่อ "สิ่งที่ต้อง setup" มีมากกว่าหนึ่งอย่างที่
สัมพันธ์กัน (ในตัวอย่างนี้คือทั้งตะกร้าและค่าที่คาดหวังไว้คู่กัน) — การรวมไว้ใน struct เดียวทำให้แก้ไข setup logic
ในที่เดียว แล้วทุกเทสที่เรียก `setup()` ได้ผลลัพธ์ที่สอดคล้องกันโดยอัตโนมัติเสมอ

**ข้อควรระวัง**: helper/fixture function ควรอยู่ **ภายใน `mod tests`** เท่านั้น (ไม่ใช่ประกาศเป็น item ระดับเดียว
กับโค้ด production) เพื่อยืนยันว่ามันจะถูก compile เข้าไปเฉพาะตอนเทส (ผ่าน `#[cfg(test)]` ที่ครอบ `mod tests`
ทั้งก้อนอยู่แล้ว) ไม่หลุดไปปนกับ production code โดยไม่ตั้งใจ

### 32.12 Mocking เบื้องต้น: สับเปลี่ยน dependency จริงด้วยของปลอมผ่าน Trait

หัวข้อสุดท้ายก่อนตัวอย่างจริงจังท้ายบทคือแนวคิดที่หลายคนได้ยินคำว่า **"mock"**/**"mocking"** มาจากภาษาอื่นแล้วสงสัย
ว่า Rust ทำยังไง — บทนี้จะแนะนำแนวคิดพื้นฐานที่สุดเท่านั้น (framework เฉพาะทางสำหรับ mocking แบบเต็มรูปแบบ เช่น
crate `mockall` ที่เคยเกริ่นไว้ใน Part 2 อยู่นอกสโคปของบทนี้ — ที่นี่เราจะทำ "mocking แบบมือ" ด้วย trait ธรรมดา
ซึ่งเพียงพอสำหรับสถานการณ์ส่วนใหญ่ และช่วยให้เข้าใจว่า framework พวกนั้น "ทำงานเบื้องหลังยังไง" ก่อนจะไปใช้งานจริง)

#### ปัญหาที่ mocking แก้: dependency ที่ทำให้เทสยาก/ช้า/ไม่ deterministic

ลองนึกภาพฟังก์ชันที่ต้องคำนวณราคารวมของสินค้าหลายตัว โดยดึงราคาแต่ละตัวจาก**ฐานข้อมูลหรือบริการภายนอก**:

```rust
// สมมติ struct นี้ผูกกับ database connection จริง (โค้ดจำลอง ไม่ compile จริงเพราะไม่มี database driver อยู่)
// struct RealDatabasePriceProvider {
//     connection: DatabaseConnection,
// }
//
// impl RealDatabasePriceProvider {
//     fn price_for(&self, sku: &str) -> Option<u32> {
//         self.connection.query_price(sku) // ยิง SQL query จริงไปหาฐานข้อมูล
//     }
// }
```

ถ้าฟังก์ชันธุรกิจของเราผูกติดกับ `RealDatabasePriceProvider` ตรง ๆ การเขียนเทสจะมีปัญหาใหญ่หลายข้อ:

- **ช้า**: การเทสต้องรอ network round-trip ไปฐานข้อมูลจริงทุกครั้ง ทำให้ `cargo test` ที่ควรรันในเสี้ยววินาที
  กลายเป็นรันหลักวินาทีหรือช้ากว่านั้น
- **ไม่ deterministic**: ราคาในฐานข้อมูลจริงอาจเปลี่ยนแปลงได้ตลอดเวลา ทำให้เทสที่ควรได้ผลเหมือนกันทุกครั้ง กลับ
  ได้ผลต่างกันไปตามข้อมูลที่มีอยู่จริงตอนนั้น (ขัดกับหลักการ "เทสต้องให้ผลลัพธ์เดิมทุกครั้งที่รัน" ที่จำเป็นมากสำหรับ
  ความน่าเชื่อถือของ test suite)
- **ต้องมี infrastructure พร้อม**: ทุกคนที่ต้องการรันเทสต้องมีฐานข้อมูลทดสอบตั้งอยู่และเข้าถึงได้ก่อน ทำให้การ
  รันเทสไม่ทำได้ "ทันทีที่ clone repo มา" อีกต่อไป (ขัดกับ workflow ที่สะดวกของ `cargo test`)

**หลักการของ mocking (แบบที่บทนี้แนะนำ) คือ**: แทนที่จะให้ฟังก์ชันธุรกิจผูกติดกับ**ชนิดข้อมูลที่เป็นของจริงตรง ๆ**
ให้มันผูกติดกับ **trait** ที่นิยาม "สิ่งที่ dependency นี้ต้องทำได้" เท่านั้น แล้ว implement trait นั้นสองแบบ:
**(1)** แบบจริง (real implementation) ที่ใช้งานจริงใน production และ **(2)** แบบปลอม (fake/stub implementation)
ที่คืนค่าคงที่ที่กำหนดไว้ล่วงหน้า (canned data) ใช้เฉพาะตอนเทสเท่านั้น — นี่คือการต่อยอดตรงจาก Part 19/21 ที่เรียน
เรื่อง trait และ generic ที่มี trait bound มาแล้ว

#### ขั้นที่ 1: นิยาม trait แทน dependency

```rust
/// dependency ภายนอกสมมติว่าเป็นการยิง query ไปฐานข้อมูล/บริการราคาส่วนกลางที่อาจช้าหรือมีต้นทุน
pub trait PriceProvider {
    fn price_for(&self, sku: &str) -> Option<u32>;
}
```

Trait นี้นิยามเพียง **"สัญญา" (contract)** ว่า "อะไรก็ตามที่เป็น `PriceProvider` ต้องมี method `price_for` ที่รับ
SKU แล้วคืน `Option<u32>` (ราคา หรือ `None` ถ้าไม่พบ)" — มันไม่สนใจเลยว่าเบื้องหลัง `price_for` จะไปคำนวณยังไง
จะเป็นฐานข้อมูล, HTTP call, หรือ HashMap ในหน่วยความจำก็ได้ทั้งนั้น

#### ขั้นที่ 2: implement แบบจริง

```rust
use std::collections::HashMap;

pub struct CatalogPriceProvider {
    catalog: HashMap<String, u32>,
}

impl CatalogPriceProvider {
    pub fn new(catalog: HashMap<String, u32>) -> Self {
        CatalogPriceProvider { catalog }
    }
}

impl PriceProvider for CatalogPriceProvider {
    fn price_for(&self, sku: &str) -> Option<u32> {
        self.catalog.get(sku).copied()
    }
}
```

(ในตัวอย่างนี้เราจำลองด้วย `HashMap` ในหน่วยความจำเพื่อให้โค้ดทั้งบทนี้ compile และรันได้จริงโดยไม่ต้องพึ่งฐานข้อมูล
จริง — ในโปรเจกต์จริง struct นี้อาจถือ database connection หรือ HTTP client แทน `HashMap` แต่หลักการของ trait
ยังเหมือนกันทุกอย่าง)

#### ขั้นที่ 3: ฟังก์ชันธุรกิจรับ dependency ผ่าน trait bound แทนชนิดที่ผูกตายตัว

```rust
/// ฟังก์ชันธุรกิจที่รับ dependency ผ่าน trait bound แบบ generic (static dispatch — ตามที่เรียนใน Part 21)
/// เพราะรับผ่าน trait ไม่ใช่ struct ที่ผูกตายตัว โค้ดนี้จึง "testable" — เราสับเปลี่ยน provider
/// เป็นตัวปลอมตอนเทสได้โดยไม่ต้องแก้โค้ดฟังก์ชันนี้เลยแม้แต่บรรทัดเดียว
pub fn total_price_for_skus<P: PriceProvider>(provider: &P, skus: &[&str]) -> Option<u32> {
    let mut total = 0u32;
    for sku in skus {
        total += provider.price_for(sku)?;
    }
    Some(total)
}
```

จุดสำคัญที่สุดของโค้ดชิ้นนี้คือ signature `fn total_price_for_skus<P: PriceProvider>(provider: &P, ...)` —
ฟังก์ชันนี้ **ไม่รู้จักและไม่สนใจ** `CatalogPriceProvider` เลยแม้แต่นิดเดียว มันรู้จักแค่ trait `PriceProvider`
เท่านั้น (concept นี้ตรงกับที่ Part 21 เรียกว่า **dependency inversion ผ่าน trait bound** — โค้ด "ระดับสูง"
อย่าง `total_price_for_skus` ไม่ผูกติดกับรายละเอียด "ระดับต่ำ" อย่าง `CatalogPriceProvider` ตรง ๆ แต่ผูกติดกับ
abstraction คือ trait แทน) นี่คือสิ่งที่ทำให้เราสับเปลี่ยน implementation ที่ส่งเข้ามาได้อย่างอิสระ ตราบใดที่มัน
implement `PriceProvider` ก็ใช้ได้เสมอ

> **หมายเหตุเรื่อง static vs dynamic dispatch**: ตัวอย่างนี้ใช้ generic (`<P: PriceProvider>`) ซึ่งเป็น **static
> dispatch** (compiler สร้างโค้ดแยกเฉพาะสำหรับแต่ละชนิด `P` ที่ถูกเรียกใช้จริง ตามที่เรียนใน Part 21) ถ้าต้องการ
> **dynamic dispatch** แทน (เช่นต้องการเก็บ provider หลายชนิดปนกันใน collection เดียว หรือไม่อยากให้ signature มี
> generic parameter) เปลี่ยนเป็น `&dyn PriceProvider` ได้เช่นกัน: `fn total_price_for_skus(provider: &dyn
> PriceProvider, skus: &[&str]) -> Option<u32>` — ทั้งสองแบบใช้เทคนิค mocking แบบเดียวกันได้เหมือนกันทุกประการ
> เพียงแค่เลือก dispatch mechanism ต่างกันตาม trade-off ที่เรียนไว้ใน Part 21

#### ขั้นที่ 4: implement แบบปลอมสำหรับเทสเท่านั้น

```rust
#[cfg(test)]
mod tests {
    use super::*;

    /// ตัวปลอมที่ใช้เฉพาะตอนเทส คืนค่าที่กำหนดไว้ล่วงหน้าแบบคงที่ (canned data)
    /// ไม่แตะเครือข่าย/ฐานข้อมูลจริงเลย ทำให้เทสเร็วและ deterministic
    struct FakePriceProvider {
        canned: HashMap<String, u32>,
    }

    impl FakePriceProvider {
        fn new() -> Self {
            let mut canned = HashMap::new();
            canned.insert("SKU-MOUSE".to_string(), 29900);
            canned.insert("SKU-KEYBOARD".to_string(), 89000);
            FakePriceProvider { canned }
        }
    }

    impl PriceProvider for FakePriceProvider {
        fn price_for(&self, sku: &str) -> Option<u32> {
            self.canned.get(sku).copied()
        }
    }

    #[test]
    fn total_price_for_skus_uses_fake_provider() {
        let fake = FakePriceProvider::new();

        let total = total_price_for_skus(&fake, &["SKU-MOUSE", "SKU-KEYBOARD"]);

        assert_eq!(total, Some(29900 + 89000));
    }

    #[test]
    fn total_price_for_skus_returns_none_when_sku_unknown() {
        let fake = FakePriceProvider::new();

        let total = total_price_for_skus(&fake, &["SKU-MOUSE", "SKU-DOES-NOT-EXIST"]);

        assert_eq!(total, None);
    }
}
```

สังเกตว่า `FakePriceProvider` นิยามอยู่**ภายใน `mod tests`** (ไม่มี `pub`, ไม่ compile เข้า production build เลย
ตาม `#[cfg(test)]` ที่เรียนในหัวข้อ 32.3) — มันเป็น "ของปลอม" ที่มีอยู่เฉพาะในโลกของเทสเท่านั้น ไม่มีใครนอก crate
หรือแม้แต่โค้ด production ใน crate เดียวกันเห็นมันเลยด้วยซ้ำ การทดสอบ `total_price_for_skus_returns_none_when_
sku_unknown` แสดงให้เห็นข้อดีอีกข้อของ mocking: เราทดสอบ **edge case ที่ยากจะจำลองด้วยฐานข้อมูลจริง** (เช่น
"SKU ที่ไม่มีอยู่จริงเลย") ได้อย่างง่ายดายและแน่นอน 100% เพราะเราควบคุม `FakePriceProvider` ได้เต็มที่ — ไม่ต้อง
ไปนั่งลบ/เพิ่มข้อมูลในฐานข้อมูลจริงเพื่อจำลอง scenario นี้เลย

**ทดสอบว่าของจริงก็ยัง implement contract เดียวกันได้ถูกต้อง**: อย่าลืมว่าเรายังต้องมีเทสสำหรับ
`CatalogPriceProvider` (ของจริง) ด้วยเช่นกัน เพื่อยืนยันว่า mapping ระหว่าง `HashMap` กับ trait `PriceProvider`
ทำงานถูกต้องจริง (การ mock ไม่ได้แปลว่าไม่ต้องเทสของจริงเลย มันแค่แยกความรับผิดชอบ: เทสของจริงยืนยันว่า
"implementation นี้ทำงานถูกต้องตาม contract", เทสด้วย fake ยืนยันว่า "โค้ดที่เรียกใช้ trait ทำงานถูกต้องตาม
contract โดยไม่สนใจว่า implementation จริงเป็นยังไง"):

```rust
#[test]
fn catalog_price_provider_still_works_as_real_impl() -> Result<(), String> {
    let mut catalog = HashMap::new();
    catalog.insert("SKU-MOUSE".to_string(), 29900);
    let real_provider = CatalogPriceProvider::new(catalog);

    match total_price_for_skus(&real_provider, &["SKU-MOUSE"]) {
        Some(29900) => Ok(()),
        other => Err(format!("คาดว่าได้ Some(29900) แต่ได้ {other:?}")),
    }
}
```

#### สรุปแนวคิด mocking แบบง่ายที่สุด

| แนวคิด | ในตัวอย่างนี้คือ |
|---|---|
| Dependency ที่อยากสับเปลี่ยนได้ | การดึงราคาสินค้า (อาจเป็นฐานข้อมูล/API จริง) |
| สัญญา (contract) ที่นิยามไว้ | trait `PriceProvider` |
| Implementation จริงที่ใช้ตอน production | `CatalogPriceProvider` |
| Implementation ปลอมที่ใช้เฉพาะตอนเทส | `FakePriceProvider` (นิยามใน `mod tests`) |
| โค้ดธุรกิจที่ไม่รู้เลยว่ากำลังคุยกับของจริงหรือของปลอม | `total_price_for_skus<P: PriceProvider>` |

นี่คือรูปแบบที่สุดของ **dependency injection** ในภาษาที่ไม่มี framework DI แบบภาษาอื่น (เช่น Spring ของ Java หรือ
Angular's DI ของ TypeScript) — Rust ไม่ต้องมี framework พิเศษเลย เพราะ trait + generic (หรือ `dyn Trait`)
ทำหน้าที่นี้ได้ครบถ้วนอยู่แล้วโดยธรรมชาติของภาษา ข้อควรรู้คือเทคนิคนี้ใช้ได้ดีที่สุดเมื่อ dependency ถูกออกแบบผ่าน
trait ไว้ตั้งแต่แรก — ถ้าโค้ดเดิมผูกกับ concrete type ตรง ๆ (เช่นเรียก `RealDatabasePriceProvider` ตรง ๆ ไม่ผ่าน
trait) การจะ "แปลงมาให้ mock ได้" ต้อง refactor ให้ดึง trait ออกมาก่อน ซึ่งเป็นงานที่ทำได้เสมอแต่ต้องลงมือทำอย่าง
ตั้งใจ ไม่ได้เกิดขึ้นโดยอัตโนมัติ (framework อย่าง `mockall` ที่กล่าวถึงข้างต้นช่วยลดงาน boilerplate ของการเขียน
fake implementation ด้วยมือแบบนี้ลงไปอีก โดยใช้ macro generate โค้ดที่คล้ายกันให้อัตโนมัติ — แต่หลักการพื้นฐาน
เบื้องหลังยังเป็น trait-based dependency injection แบบเดียวกันนี้เป๊ะ ๆ)

### 32.13 ตัวอย่างจริงจัง: ระบบ `ShoppingCart` พร้อม Test Suite ครบวงจร

มาประกอบทุกเทคนิคที่เรียนมาทั้งบทเข้าด้วยกันเป็นตัวอย่างเดียวที่สมบูรณ์ — ระบบตะกร้าสินค้า (`ShoppingCart`) ที่มี
ทั้ง happy path, edge case, `#[should_panic]`, การทดสอบ private function, fixture function, และ mocking ครบใน
ไฟล์เดียว (โค้ดชุดนี้ compile และรันผ่านทุกเทสจริง ได้ตรวจสอบด้วย `cargo test` แล้ว — ดูรายละเอียดวิธีตรวจสอบท้ายบท)

```rust
use std::collections::HashMap;

/// สินค้าหนึ่งตัวในแคตตาล็อก (ราคาเก็บเป็นหน่วย "สตางค์" (cents) เพื่อเลี่ยงปัญหาความคลาดเคลื่อนของ floating point
/// ที่เห็นตัวอย่างไปแล้วในหัวข้อ 32.5 — นี่คือเหตุผลเชิงปฏิบัติที่ระบบการเงินจริงแทบทุกระบบเลี่ยงใช้ f64 เก็บเงิน)
#[derive(Debug, Clone, PartialEq)]
pub struct Product {
    pub name: String,
    pub unit_price_cents: u32,
}

impl Product {
    pub fn new(name: &str, unit_price_cents: u32) -> Self {
        Product {
            name: name.to_string(),
            unit_price_cents,
        }
    }
}

#[derive(Debug, Clone, PartialEq)]
struct CartItem {
    product: Product,
    quantity: u32,
}

/// ตะกร้าสินค้า — เก็บ field เป็น private เพื่อบังคับให้ทุกการแก้ไขต้องผ่าน method ที่ตรวจสอบความถูกต้อง
/// (ตาม pattern ที่เรียนใน Part 16 หัวข้อ 16.9 เรื่อง struct field privacy)
#[derive(Debug, Default)]
pub struct ShoppingCart {
    items: Vec<CartItem>,
}

impl ShoppingCart {
    pub fn new() -> Self {
        ShoppingCart { items: Vec::new() }
    }

    /// เพิ่มสินค้าเข้าตะกร้า ปฏิเสธด้วย Err ถ้า quantity เป็น 0 (edge case ที่ไม่มีความหมายทางธุรกิจ)
    pub fn add_item(&mut self, product: Product, quantity: u32) -> Result<(), String> {
        if quantity == 0 {
            return Err(format!(
                "ไม่สามารถเพิ่ม '{}' ด้วยจำนวน 0 ชิ้นได้",
                product.name
            ));
        }

        // ถ้าสินค้าชื่อเดียวกันมีอยู่แล้วในตะกร้า ให้รวมจำนวนเข้าด้วยกันแทนการเพิ่ม item ใหม่
        if let Some(existing) = self.items.iter_mut().find(|i| i.product.name == product.name) {
            existing.quantity += quantity;
        } else {
            self.items.push(CartItem { product, quantity });
        }
        Ok(())
    }

    pub fn is_empty(&self) -> bool {
        self.items.is_empty()
    }

    pub fn item_count(&self) -> u32 {
        self.items.iter().map(|i| i.quantity).sum()
    }

    pub fn distinct_product_count(&self) -> usize {
        self.items.len()
    }

    /// ราคารวมทั้งตะกร้า (หน่วยสตางค์)
    pub fn total_cents(&self) -> u32 {
        self.items
            .iter()
            .map(|i| i.product.unit_price_cents * i.quantity)
            .sum()
    }

    /// ลบสินค้าออกจากตะกร้าตามชื่อ คืนค่า true ถ้าลบสำเร็จ (เจอสินค้านั้นจริง)
    pub fn remove_item(&mut self, product_name: &str) -> bool {
        let original_len = self.items.len();
        self.items.retain(|i| i.product.name != product_name);
        self.items.len() != original_len
    }

    /// สรุปยอด checkout — ตั้งใจ panic เมื่อตะกร้าว่าง เพราะการ checkout ตะกร้าว่างถือเป็น "bug ของผู้เรียก"
    /// ไม่ใช่กรณีที่ควรจัดการแบบ error ทั่วไป (ทีมออกแบบ API นี้ตัดสินใจว่าผู้เรียกต้องเช็ค is_empty() ก่อนเสมอ —
    /// การตัดสินใจแบบนี้เป็นการออกแบบที่ถูกต้องอย่างหนึ่งตามหลักที่เรียนใน Part 30/31 เรื่องเมื่อไหร่ควร panic
    /// เมื่อไหร่ควรคืน Result)
    pub fn checkout(&self) -> u32 {
        if self.is_empty() {
            panic!("ไม่สามารถ checkout ตะกร้าสินค้าที่ว่างได้ (empty cart)");
        }
        self.total_cents()
    }
}

/// แปลงหน่วยสตางค์เป็น string แบบบาท เช่น 129900 -> "1299.00 บาท"
/// ฟังก์ชันนี้เป็น private (ไม่มี pub) เพราะเป็นรายละเอียดการ format ภายใน ไม่ใช่ contract สาธารณะของโมดูล
fn format_cents_as_baht(cents: u32) -> String {
    let baht = cents / 100;
    let sub = cents % 100;
    format!("{baht}.{sub:02} บาท")
}

// ---------------------------------------------------------------------
// Mocking เบื้องต้น: trait-based dependency injection (จากหัวข้อ 32.12)
// ---------------------------------------------------------------------

/// dependency ภายนอกสมมติว่าเป็นการยิง query ไปฐานข้อมูล/บริการราคาส่วนกลางที่อาจช้าหรือมีต้นทุน
pub trait PriceProvider {
    fn price_for(&self, sku: &str) -> Option<u32>;
}

/// ของจริงที่ใช้งานตอน production — ในโค้ดจริงตรงนี้อาจเป็น HTTP client หรือ SQL query
/// แต่ในตัวอย่างนี้จำลองด้วย HashMap ในหน่วยความจำเพื่อให้ compile ได้เองโดยไม่ต้องพึ่งเครือข่าย/ฐานข้อมูลจริง
pub struct CatalogPriceProvider {
    catalog: HashMap<String, u32>,
}

impl CatalogPriceProvider {
    pub fn new(catalog: HashMap<String, u32>) -> Self {
        CatalogPriceProvider { catalog }
    }
}

impl PriceProvider for CatalogPriceProvider {
    fn price_for(&self, sku: &str) -> Option<u32> {
        self.catalog.get(sku).copied()
    }
}

/// ฟังก์ชันธุรกิจที่รับ dependency ผ่าน trait bound แบบ generic (static dispatch)
pub fn total_price_for_skus<P: PriceProvider>(provider: &P, skus: &[&str]) -> Option<u32> {
    let mut total = 0u32;
    for sku in skus {
        total += provider.price_for(sku)?;
    }
    Some(total)
}

#[cfg(test)]
mod tests {
    use super::*;

    // -------------------- ShoppingCart: happy path --------------------

    #[test]
    fn new_cart_is_empty() {
        let cart = ShoppingCart::new();
        assert!(cart.is_empty());
        assert_eq!(cart.item_count(), 0);
        assert_eq!(cart.total_cents(), 0);
    }

    #[test]
    fn adding_one_item_updates_totals() {
        let mut cart = ShoppingCart::new();
        let mouse = Product::new("เมาส์ไร้สาย", 29900);

        cart.add_item(mouse, 2).expect("เพิ่มสินค้าปกติต้องไม่ error");

        assert!(!cart.is_empty());
        assert_eq!(cart.item_count(), 2);
        assert_eq!(cart.total_cents(), 59800);
        assert_eq!(cart.distinct_product_count(), 1);
    }

    #[test]
    fn adding_same_product_twice_merges_quantity() {
        let mut cart = ShoppingCart::new();
        cart.add_item(Product::new("คีย์บอร์ด", 89000), 1).unwrap();
        cart.add_item(Product::new("คีย์บอร์ด", 89000), 2).unwrap();

        // ต้องรวมกันเป็น item เดียวที่ quantity = 3 ไม่ใช่สอง item แยกกัน
        assert_eq!(cart.distinct_product_count(), 1);
        assert_eq!(cart.item_count(), 3);
        assert_eq!(cart.total_cents(), 89000 * 3);
    }

    #[test]
    fn multiple_distinct_products_sum_correctly() {
        let mut cart = ShoppingCart::new();
        cart.add_item(Product::new("เมาส์ไร้สาย", 29900), 1).unwrap();
        cart.add_item(Product::new("คีย์บอร์ด", 89000), 1).unwrap();

        assert_eq!(cart.distinct_product_count(), 2);
        assert_eq!(cart.total_cents(), 29900 + 89000);
    }

    #[test]
    fn remove_item_that_exists_returns_true_and_removes_it() {
        let mut cart = ShoppingCart::new();
        cart.add_item(Product::new("เมาส์ไร้สาย", 29900), 1).unwrap();

        let removed = cart.remove_item("เมาส์ไร้สาย");

        assert!(removed);
        assert!(cart.is_empty());
    }

    // -------------------- edge cases --------------------

    #[test]
    fn remove_item_that_does_not_exist_returns_false() {
        let mut cart = ShoppingCart::new();
        cart.add_item(Product::new("เมาส์ไร้สาย", 29900), 1).unwrap();

        let removed = cart.remove_item("จอมอนิเตอร์"); // ไม่มีสินค้านี้ในตะกร้า

        assert!(!removed);
        assert_eq!(cart.item_count(), 1); // ตะกร้าต้องไม่เปลี่ยนแปลง
    }

    #[test]
    fn adding_zero_quantity_returns_err_and_does_not_modify_cart() {
        let mut cart = ShoppingCart::new();

        let result = cart.add_item(Product::new("เมาส์ไร้สาย", 29900), 0);

        assert!(result.is_err());
        assert!(cart.is_empty()); // ต้องยังว่างอยู่ เพราะการเพิ่มไม่ควรมีผลใด ๆ
    }

    #[test]
    fn checkout_on_cart_with_single_item_returns_total() {
        let mut cart = ShoppingCart::new();
        cart.add_item(Product::new("เมาส์ไร้สาย", 29900), 1).unwrap();

        assert_eq!(cart.checkout(), 29900);
    }

    // -------------------- should_panic case --------------------

    #[test]
    #[should_panic(expected = "empty cart")]
    fn checkout_on_empty_cart_panics() {
        let cart = ShoppingCart::new();
        cart.checkout(); // ต้อง panic เพราะตะกร้าว่าง — พิสูจน์ว่า guard เงื่อนไขนี้ยังทำงานอยู่จริง
    }

    // -------------------- ทดสอบ private function ได้เพราะอยู่ไฟล์เดียวกัน --------------------

    #[test]
    fn format_cents_as_baht_formats_correctly() {
        assert_eq!(format_cents_as_baht(129900), "1299.00 บาท");
        assert_eq!(format_cents_as_baht(50), "0.50 บาท");
        assert_eq!(format_cents_as_baht(0), "0.00 บาท");
    }

    // -------------------- fixture / helper function ที่ใช้ร่วมกันหลายเทส --------------------

    /// helper สร้างตะกร้าที่มีสินค้ามาตรฐาน 2 รายการไว้ล่วงหน้า ใช้ซ้ำได้หลายเทสที่ต้องการ "ตะกร้าที่ไม่ว่าง" เป็นจุดเริ่มต้น
    fn setup_cart_with_two_products() -> ShoppingCart {
        let mut cart = ShoppingCart::new();
        cart.add_item(Product::new("เมาส์ไร้สาย", 29900), 2).unwrap();
        cart.add_item(Product::new("คีย์บอร์ด", 89000), 1).unwrap();
        cart
    }

    #[test]
    fn setup_fixture_has_expected_total() {
        let cart = setup_cart_with_two_products();
        assert_eq!(cart.total_cents(), 29900 * 2 + 89000);
    }

    #[test]
    fn setup_fixture_removing_one_leaves_the_other() {
        let mut cart = setup_cart_with_two_products();
        cart.remove_item("คีย์บอร์ด");
        assert_eq!(cart.distinct_product_count(), 1);
        assert_eq!(cart.total_cents(), 29900 * 2);
    }

    // -------------------- mocking ด้วย trait-based fake --------------------

    /// ตัวปลอมที่ใช้เฉพาะตอนเทส คืนค่าที่กำหนดไว้ล่วงหน้าแบบคงที่ (canned data)
    /// ไม่แตะเครือข่าย/ฐานข้อมูลจริงเลย ทำให้เทสเร็วและ deterministic
    struct FakePriceProvider {
        canned: HashMap<String, u32>,
    }

    impl FakePriceProvider {
        fn new() -> Self {
            let mut canned = HashMap::new();
            canned.insert("SKU-MOUSE".to_string(), 29900);
            canned.insert("SKU-KEYBOARD".to_string(), 89000);
            FakePriceProvider { canned }
        }
    }

    impl PriceProvider for FakePriceProvider {
        fn price_for(&self, sku: &str) -> Option<u32> {
            self.canned.get(sku).copied()
        }
    }

    #[test]
    fn total_price_for_skus_uses_fake_provider() {
        let fake = FakePriceProvider::new();

        let total = total_price_for_skus(&fake, &["SKU-MOUSE", "SKU-KEYBOARD"]);

        assert_eq!(total, Some(29900 + 89000));
    }

    #[test]
    fn total_price_for_skus_returns_none_when_sku_unknown() {
        let fake = FakePriceProvider::new();

        let total = total_price_for_skus(&fake, &["SKU-MOUSE", "SKU-DOES-NOT-EXIST"]);

        assert_eq!(total, None);
    }

    #[test]
    fn catalog_price_provider_still_works_as_real_impl() -> Result<(), String> {
        let mut catalog = HashMap::new();
        catalog.insert("SKU-MOUSE".to_string(), 29900);
        let real_provider = CatalogPriceProvider::new(catalog);

        match total_price_for_skus(&real_provider, &["SKU-MOUSE"]) {
            Some(29900) => Ok(()),
            other => Err(format!("คาดว่าได้ Some(29900) แต่ได้ {other:?}")),
        }
    }
}
```

รัน `cargo test` กับไฟล์นี้ทั้งไฟล์ ผลลัพธ์จริงที่ได้ (ทุกเทสผ่านหมด):

```
running 15 tests
test tests::adding_one_item_updates_totals ... ok
test tests::adding_same_product_twice_merges_quantity ... ok
test tests::adding_zero_quantity_returns_err_and_does_not_modify_cart ... ok
test tests::catalog_price_provider_still_works_as_real_impl ... ok
test tests::checkout_on_cart_with_single_item_returns_total ... ok
test tests::format_cents_as_baht_formats_correctly ... ok
test tests::multiple_distinct_products_sum_correctly ... ok
test tests::new_cart_is_empty ... ok
test tests::remove_item_that_does_not_exist_returns_false ... ok
test tests::remove_item_that_exists_returns_true_and_removes_it ... ok
test tests::setup_fixture_has_expected_total ... ok
test tests::setup_fixture_removing_one_leaves_the_other ... ok
test tests::total_price_for_skus_returns_none_when_sku_unknown ... ok
test tests::total_price_for_skus_uses_fake_provider ... ok
test tests::checkout_on_empty_cart_panics - should panic ... ok

test result: ok. 15 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.08s
```

15 เทสครอบคลุม: happy path 5 เทส (การเพิ่มสินค้า, การรวม quantity, สินค้าหลายชนิด, การลบสินค้า, การ checkout ปกติ),
edge case 2 เทส (ลบสินค้าที่ไม่มีอยู่, เพิ่ม quantity เป็น 0), `#[should_panic]` 1 เทส (checkout ตะกร้าว่าง), การ
ทดสอบ private function ตรง ๆ 1 เทส (`format_cents_as_baht`), fixture-based เทส 2 เทส (ใช้ `setup_cart_with_two_
products`), และ mocking-based เทส 3 เทส (ทดสอบทั้ง fake และของจริงผ่าน trait เดียวกัน) — นี่คือภาพรวมของ test
suite ที่ "ครบวงจร" ในความหมายที่หลักสูตรนี้ต้องการสื่อ: ไม่ใช่แค่ทดสอบว่า "โค้ดทำงาน" แต่ทดสอบครอบคลุมทุกมุมที่
เทคนิคต่าง ๆ ในบทนี้ช่วยให้ทำได้

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. ลืม `use super::*;` ใน `mod tests` แล้วงงว่าทำไม compiler บอกว่าหาฟังก์ชันไม่เจอ

```rust
fn add_two(a: i32, b: i32) -> i32 {
    a + b
}

#[cfg(test)]
mod tests {
    // ลืม use super::*; ไปเลย

    #[test]
    fn calls_add_two_without_import() {
        assert_eq!(add_two(2, 2), 4);
    }
}
```

Error จริงที่ได้:

```
error[E0425]: cannot find function `add_two` in this scope
   --> src/lib.rs:135:20
    |
135 |         assert_eq!(add_two(2, 2), 4);
    |                    ^^^^^^^ not found in this scope
    |
help: consider importing this function
    |
133 +     use crate::add_two;
    |
```

แม้ `mod tests` จะเป็น module ลูกที่**มีสิทธิ์**มองเห็น `add_two` ได้ตามกฎ privacy จาก Part 16 (เพราะมันเป็น private
function ของ module พ่อ) แต่ "มีสิทธิ์มองเห็น" กับ "เขียนชื่อสั้น ๆ เรียกได้ตรง ๆ โดยไม่ต้องใส่ path" เป็นคนละเรื่อง
กัน — ต้อง `use` เข้ามาใน scope ก่อนเสมอ (หรือเขียน path เต็ม `super::add_two(...)` ก็ได้ แต่ไม่มีใครทำแบบนั้นใน
ทางปฏิบัติ) วิธีแก้คือเติม `use super::*;` เป็นบรรทัดแรกใน `mod tests` เสมอ ตามที่ `cargo new --lib` generate ให้
เป็นตัวอย่างมาตั้งแต่ Part 2 นั่นเอง — สังเกตข้อความ `help:` ที่ compiler แนะนำมาด้วย (`use crate::add_two;`) ก็ใช้
ได้เหมือนกัน เพียงแต่ community convention เลือกใช้ `use super::*;` เพราะสะดวกกว่าเมื่อต้องเรียกหลายฟังก์ชัน

### 2. `#[should_panic]` แบบไม่มี `expected` ทำให้เทส "ผ่านผิดที่" (false positive)

อย่างที่อธิบายในหัวข้อ 32.7 — `#[should_panic]` เปล่า ๆ จะ pass ทันทีที่เกิด panic ขึ้น**จากสาเหตุใดก็ตาม** แม้จะ
ไม่เกี่ยวกับสิ่งที่ตั้งใจทดสอบเลย ทำให้เทสดู "ผ่าน" แต่ไม่ได้พิสูจน์อะไรจริง ๆ วิธีแก้คือใส่ `expected = "..."`
เสมอเมื่อทำได้ เพื่อยืนยันว่า panic ที่เกิดขึ้นเป็น panic ที่ต้องการจริง ๆ ไม่ใช่ panic จากบั๊กอื่นที่ปนเข้ามาโดยบังเอิญ
ลองดู error ที่เกิดเมื่อข้อความไม่ตรงกับที่คาด (แสดงว่า `expected` ทำงานถูกต้องในการจับกรณีนี้):

```
note: panic did not contain expected string
      panic message: "ยอดเงินไม่พอ: มี 100 แต่ขอถอน 500"
 expected substring: "ข้อความที่ไม่ตรงกับ panic จริง"
```

ถ้าเห็น error แบบนี้ ให้ตรวจสอบว่า `expected` ที่เขียนไว้ตรงกับข้อความ panic จริงหรือไม่ (ควรใช้ substring สั้น ๆ
ที่มั่นใจว่าจะไม่เปลี่ยนบ่อยจากการ refactor ข้อความ panic เล็กน้อย เช่นแค่คำสำคัญที่สุดของข้อความ ไม่ใช่ทั้งประโยค
แบบเป๊ะ ๆ ที่อาจพังง่ายถ้ามีคนแก้ wording นิดเดียว)

### 3. ใช้ `assert_eq!` เทียบ floating point ตรง ๆ โดยไม่คิดเรื่อง precision

```rust
#[test]
fn floating_point_fails() {
    let x = 0.1 + 0.2;
    assert_eq!(x, 0.3); // FAILED! เพราะ 0.1 + 0.2 != 0.3 เป๊ะ ๆ ใน f64
}
```

Error จริงที่ได้:

```
thread 'tests::floating_point_fails' panicked at src/lib.rs:53:9:
assertion `left == right` failed
  left: 0.30000000000000004
 right: 0.3
```

นี่ไม่ใช่บั๊กของ Rust แต่เป็นธรรมชาติของ **IEEE 754 floating point representation** ที่ทุกภาษาโปรแกรมมิ่งเจอปัญหา
เดียวกันหมด (ทศนิยมฐานสองไม่สามารถแทนค่า 0.1 หรือ 0.2 ได้เป๊ะ ๆ เหมือนที่เราคุ้นเคยในฐานสิบ) วิธีแก้ที่ถูกต้องคือ
**ห้ามใช้ `assert_eq!` เทียบ `f32`/`f64` แบบตรง ๆ เด็ดขาด** ให้เทียบ "ความต่างต้องน้อยกว่าค่า epsilon ที่ยอมรับได้"
แทน:

```rust
#[test]
fn floating_point_compared_with_tolerance() {
    let x: f64 = 0.1 + 0.2;
    let expected: f64 = 0.3;
    let epsilon = 1e-10;
    assert!(
        (x - expected).abs() < epsilon,
        "คาดว่า {expected} แต่ได้ {x} (ต่างกันเกิน epsilon ที่ยอมรับได้)"
    );
}
```

หรือถ้าเป็นไปได้ ให้พิจารณาออกแบบให้ใช้จำนวนเต็ม (เช่นหน่วยสตางค์อย่างที่ตัวอย่าง `ShoppingCart` ในบทนี้ทำ) แทน
floating point เมื่อค่าที่เกี่ยวข้องเป็นเงินหรือค่าที่ต้องการความแน่นอนสัมบูรณ์ — วิธีนี้ตัดปัญหา precision ออกไป
ทั้งหมดตั้งแต่ต้น ไม่ต้องมาคอยเทียบด้วย epsilon ทุกจุดที่เกี่ยวกับเงินเลย

### 4. เทสที่แก้ไข shared mutable state (ไฟล์, static variable, environment variable) แล้ว flaky เพราะรันแบบ parallel

ตามที่อธิบายละเอียดในหัวข้อ 32.4 — ถ้าเทสหลายตัวแก้ไข resource ร่วมกัน (ไฟล์ path เดียวกัน, `static mut`
เดียวกัน, environment variable เดียวกัน) การรันแบบ parallel by default จะทำให้เกิด race condition แบบสุ่ม ๆ:

```rust
// ตัวอย่างที่มีปัญหา: สองเทสแก้ environment variable ตัวเดียวกัน
#[test]
fn test_reads_config_a() {
    std::env::set_var("APP_MODE", "production");
    // ... อ่านค่า APP_MODE แล้วเทส
}

#[test]
fn test_reads_config_b() {
    std::env::set_var("APP_MODE", "development");
    // ... อ่านค่า APP_MODE แล้วเทส — อาจอ่านได้ "production" ถ้า test_reads_config_a
    //     รันแทรกเข้ามาระหว่างกลางพอดี เพราะ environment variable เป็น process-wide!
}
```

อาการของปัญหานี้คือ **เทส fail แบบไม่สม่ำเสมอ** — รันสิบครั้งอาจผ่านเก้าครั้ง fail หนึ่งครั้งแบบสุ่ม ๆ ไม่เกี่ยวกับ
การแก้โค้ดเลย วิธีตรวจสอบคร่าว ๆ ว่าใช่ปัญหานี้หรือไม่คือลองรันด้วย `cargo test -- --test-threads=1` แล้วดูว่า
เทสที่เคย flaky กลับผ่านสม่ำเสมอทุกครั้งหรือไม่ — ถ้าใช่ แสดงว่ามีเทสบางตัวพึ่งพา shared state ที่ชนกันจริง วิธีแก้
ที่ถูกต้องคือออกแบบให้แต่ละเทสใช้ resource ของตัวเองแยกกัน (เช่นใช้ crate `tempfile` สำหรับไฟล์ชั่วคราว หรือฉีด
ค่า config เป็น parameter ของฟังก์ชันแทนการอ่านจาก environment variable ตรง ๆ ข้างในฟังก์ชันเลย) ไม่ใช่แค่บังคับ
`--test-threads=1` ไว้ตลอดกาล (อย่างที่อธิบายไว้ว่าเป็นการแก้ปลายเหตุ ไม่ใช่ต้นเหตุ)

### 5. ลืมว่าฟังก์ชันที่เป็น private และถูกเรียกจาก `#[cfg(test)]` เท่านั้น จะโดน warning `dead_code` ตอน build ปกติ

```rust
fn format_cents_as_baht(cents: u32) -> String {
    let baht = cents / 100;
    let sub = cents % 100;
    format!("{baht}.{sub:02} บาท")
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn format_cents_as_baht_formats_correctly() {
        assert_eq!(format_cents_as_baht(129900), "1299.00 บาท");
    }
}
```

ถ้าฟังก์ชัน private ตัวนี้**ไม่ถูกเรียกจากที่ไหนเลยนอกจากใน `mod tests`** การรัน `cargo build`/`cargo check`/
`cargo clippy` แบบปกติ (ไม่ใช่ `cargo test`) จะแสดง warning จริง:

```
warning: function `format_cents_as_baht` is never used
  --> src/lib.rs:93:4
   |
93 | fn format_cents_as_baht(cents: u32) -> String {
   |    ^^^^^^^^^^^^^^^^^^^^
   |
   = note: `#[warn(dead_code)]` (part of `#[warn(unused)]`) on by default
```

เหตุผลคือ **การ build แบบปกติไม่เปิด `cfg(test)` เลย** ทำให้ compiler มองไม่เห็นว่า `mod tests` เรียกใช้ฟังก์ชันนี้
อยู่ (มันถูกตัดออกไปตั้งแต่ต้นตามที่อธิบายในหัวข้อ 32.3) เมื่อไม่มีใครเรียกมันเลยใน build ที่กำลังพิจารณาอยู่
`dead_code` lint จึงเตือนตามปกติ — **นี่ไม่ใช่ error และไม่ใช่ปัญหาที่ต้องแก้เสมอไป** ถ้าฟังก์ชันนั้นมีแผนจะถูกเรียก
ใช้จริงในโค้ด production ในอนาคตอันใกล้ (กำลังพัฒนาฟีเจอร์ที่ยังไม่เสร็จ) หรือถ้ามันเป็น helper ที่ตั้งใจให้เป็น
private detail ที่ยังไม่มีจุดเรียกใช้จริงในโค้ดตอนนี้แต่จำเป็นสำหรับเทสที่ครอบคลุม logic สำคัญ ก็สามารถปล่อย warning
นี้ไว้ได้ (หรือกด `#[allow(dead_code)]` ไว้ชั่วคราวถ้า warning รบกวนมากเกินไป) แต่ถ้า warning นี้ปรากฏขึ้นสำหรับ
ฟังก์ชันที่ **ไม่ได้ตั้งใจ**ให้เป็นแบบนี้ (เช่นคุณคิดว่ามันถูกเรียกจากที่อื่นด้วย แต่จริง ๆ ไม่ได้ถูกเรียก) นี่คือ
สัญญาณเตือนที่ดีว่าอาจมีโค้ดที่ตายแล้วจริง ๆ (dead code) หลุดเข้ามาในโปรเจกต์ ควรตรวจสอบให้แน่ใจ

### 6. เทสที่คืน `Result<(), E>` ใช้ร่วมกับ `#[should_panic]` ไม่ได้

```rust
#[test]
#[should_panic] // compile error! ใช้ร่วมกับฟังก์ชันที่คืน Result ไม่ได้
fn wrong_combination() -> Result<(), String> {
    withdraw(100, 500);
    Ok(())
}
```

การผสมสองแบบนี้จะทำให้ compile error หรือ behavior ไม่ตรงกับที่คาดหวัง เพราะตรรกะของทั้งสอง attribute ขัดแย้งกัน
โดยพื้นฐาน (`#[should_panic]` ต้องการ panic ถึง pass, ส่วนฟังก์ชันที่คืน `Result` ควรจบด้วยการ `return` ค่าปกติ
ไม่ใช่ panic) วิธีแก้คือเลือกอย่างใดอย่างหนึ่งให้ตรงกับสิ่งที่ต้องการทดสอบจริง ๆ: ถ้าต้องการทดสอบว่าโค้ด panic ให้
เขียนฟังก์ชันเทสแบบคืน `()` ธรรมดาและใช้ `#[should_panic(expected = "...")]` ตามหัวข้อ 32.7 ถ้าต้องการใช้ `?`
สำหรับ operation ที่คืน `Result` หลายตัวต่อกัน ให้เขียนฟังก์ชันคืน `Result<(), E>` ตามหัวข้อ 32.8 แต่**ไม่ผสมทั้ง
สองแบบเข้าด้วยกันในเทสเดียว**

## แบบฝึกหัด (Exercises)

1. **โจทย์ระดับง่าย**: เขียนฟังก์ชัน `fn is_valid_email_length(email: &str) -> bool` ที่คืน `true` ถ้าความยาวของ
   `email` อยู่ระหว่าง 5 ถึง 254 ตัวอักษร (ตามขีดจำกัดจริงของ email address ตามมาตรฐาน RFC) แล้วเขียนเทสอย่างน้อย
   4 ตัวครอบคลุม: ความยาวปกติที่ควรผ่าน, ความยาวสั้นเกินไป (เช่น `"a@b"`), ความยาวยาวเกินไป (สร้าง string ยาว 300
   ตัวอักษรด้วย `"a".repeat(300)`), และค่าขอบเขต (boundary) พอดี 5 และ 254 ตัวอักษร (hint: ขอบเขตแบบนี้มักเป็น
   จุดที่บั๊ก off-by-one หลุดบ่อยที่สุด ลองเขียนเทสสำหรับความยาว 4 และ 255 ด้วยเพื่อยืนยันว่า "อยู่นอกขอบเขตพอดี"
   ก็ถูกปฏิเสธเช่นกัน)

2. **โจทย์ระดับกลาง**: ขยายฟังก์ชัน `apply_discount` จากหัวข้อ 32.1 ให้กลายเป็นฟังก์ชันที่ **panic** เมื่อ
   `discount_percent` มีค่ามากกว่า 100 (เพราะส่วนลดเกิน 100% ไม่มีความหมายทางธุรกิจ) เขียนเทสให้ครบ: หนึ่งเทส
   สำหรับ happy path (ส่วนลดปกติ เช่น 10%, 50%), หนึ่งเทสสำหรับ edge case ที่ discount = 0 (ไม่มีส่วนลดเลย ควรได้
   ราคาเดิม), หนึ่งเทสสำหรับ edge case ที่ discount = 100 (ควรได้ราคา 0 พอดี ไม่ panic เพราะ 100 ยังอยู่ในขอบเขต
   ที่รับได้), และหนึ่งเทสแบบ `#[should_panic(expected = "...")]` สำหรับกรณี discount = 150 (hint: ระวังว่าขอบเขต
   ระหว่าง "ยอมรับได้" กับ "ต้อง panic" อยู่ที่ค่าไหนกันแน่ — เขียนเทสให้ตรงกับเงื่อนไข `>` หรือ `>=` ที่คุณเลือกใช้
   ในโค้ดจริงให้สอดคล้องกัน)

3. **โจทย์ระดับยาก**: นำระบบ `ShoppingCart` จากหัวข้อ 32.13 มาต่อยอด เพิ่ม method ใหม่ชื่อ
   `apply_member_discount(&mut self, percent: u32)` ที่ลดราคาสินค้า**ทุกตัว**ในตะกร้าลงตามเปอร์เซ็นต์ที่กำหนด
   (ปรับ `unit_price_cents` ของสินค้าแต่ละตัวในตะกร้าโดยตรง) แล้วเขียน test suite ให้ครบทุกมุม: happy path (เช่น
   ลด 10% แล้วยอดรวมลดลงตามสัดส่วนที่ถูกต้อง), edge case ตะกร้าว่าง (เรียก method นี้กับตะกร้าว่างไม่ควร panic
   หรือ error — แค่ไม่มีอะไรให้ลดราคา), edge case percent = 0 (ไม่ควรมีอะไรเปลี่ยนแปลง), และเขียนฟังก์ชัน
   `setup()` แบบ fixture (ตามที่เรียนในหัวข้อ 32.11) ที่คืนค่า struct รวม "ตะกร้าที่มีสินค้าอยู่แล้ว" กับ "ยอดรวม
   ที่คาดหวังไว้ก่อนหักส่วนลด" มาใช้ร่วมกันในหลายเทส (hint: คิดให้ดีว่าถ้า `percent` มากกว่า 100 ควร panic หรือคืน
   `Result::Err` — ทั้งสองแบบมีเหตุผลรองรับได้ ให้เลือกแบบหนึ่งแล้วเขียนเทสให้สอดคล้องกับตัวเลือกนั้นอย่างชัดเจน
   ที่สุด)

4. **โจทย์ประยุกต์ใช้งานจริง**: ออกแบบ trait ชื่อ `ShippingCostCalculator` ที่มี method
   `fn cost_for(&self, order_weight_grams: u32, destination_zone: &str) -> u32` (คืนค่าเป็นสตางค์) แล้ว
   implement สองแบบ: **(1)** `RealShippingCostCalculator` ที่คำนวณจากสูตรง่าย ๆ เอง (เช่น น้ำหนักคูณอัตราต่อกรัม
   บวกค่าธรรมเนียมพื้นฐานตาม zone) และ **(2)** `FakeShippingCostCalculator` (ใช้เฉพาะในเทส) ที่คืนค่าคงที่ตาม
   `HashMap` ที่กำหนดไว้ล่วงหน้า จากนั้นเขียนฟังก์ชันธุรกิจ
   `fn total_order_cost<C: ShippingCostCalculator>(subtotal_cents: u32, calculator: &C, weight_grams: u32, zone: &str) -> u32`
   ที่รวมยอดสินค้ากับค่าส่งเข้าด้วยกัน แล้วเขียนเทสสำหรับ `total_order_cost` โดยใช้ **เฉพาะ** `FakeShippingCostCalculator`
   (ไม่ต้องยุ่งกับสูตรคำนวณจริงของ `RealShippingCostCalculator` เลยในเทสกลุ่มนี้) พร้อมเขียนเทสแยกอีกกลุ่มที่ทดสอบ
   `RealShippingCostCalculator` เพียงลำพัง (ไม่ผ่าน `total_order_cost`) เพื่อยืนยันว่าสูตรคำนวณค่าส่งจริงถูกต้อง
   เอง (hint: นี่คือแบบฝึกหัดที่จำลองสถานการณ์จริงที่สุดของบทนี้ — สังเกตว่าคุณกำลังแยกความรับผิดชอบของเทสออกเป็น
   สองชั้นอย่างชัดเจน: ชั้นหนึ่งทดสอบ "ตรรกะทางธุรกิจที่ไม่สนใจว่าค่าส่งคำนวณยังไง" อีกชั้นทดสอบ "สูตรคำนวณค่าส่งเอง
   ถูกต้องหรือไม่" — นี่คือวิธีคิดแบบเดียวกับที่ mocking framework ระดับใหญ่ในโลกจริงใช้กันทุกที่)

## สรุป

บทนี้พาเราไปทำความเข้าใจ testing framework ที่ผูกมากับ Rust เองแบบครบวงจร เริ่มจากคำถามพื้นฐานที่สุดว่า "ทำไมต้อง
มีเทสทั้งที่ compiler เข้มงวดขนาดนี้แล้ว" — คำตอบคือ compiler จับบั๊กเชิงโครงสร้าง (type, ownership, memory safety)
ได้หมด แต่เทสเท่านั้นที่จับบั๊กเชิงตรรกะทางธุรกิจได้ ทั้งสองทำงานเสริมกัน ไม่ได้แทนที่กัน

จากนั้นเราเรียนรู้กลไกทั้งหมดที่ประกอบกันเป็น unit test ใน Rust: `#[test]` attribute ที่ทำให้ฟังก์ชันกลายเป็น
test case, `#[cfg(test)]` ที่ทำให้เทสไม่ถูก compile เข้า production build เลย, `mod tests { use super::*; }`
ที่ใช้ประโยชน์จากกฎ module privacy จาก Part 16 ในการทดสอบ private function ได้โดยตรง, การใช้ `cargo test` ใน
รูปแบบต่าง ๆ (filter ด้วยชื่อ, `--nocapture`, `--test-threads=1`, `--ignored`) และความเข้าใจว่าทำไมเทสต้อง
independent จากกันเพราะการรันแบบ parallel by default, การเลือกใช้ `assert!`/`assert_eq!`/`assert_ne!` ให้ถูก
สถานการณ์พร้อมอ่าน failure message ให้เป็น, การเขียน custom failure message ที่ช่วย debug ได้จริง, `#[should_panic]`
พร้อม `expected` สำหรับพิสูจน์ว่าโค้ดปฏิเสธ input ผิดอย่างถูกวิธี, เทสที่คืน `Result<(), E>` เพื่อใช้ `?` แทน
`.unwrap()` เกลื่อนเทส, `#[ignore]` สำหรับเทสที่ช้า/แพง, การจัดระเบียบ test suite ที่โตขึ้นด้วย submodule และ
fixture function, และปิดท้ายด้วยแนวคิด mocking แบบพื้นฐานที่สุดผ่านเทคนิค trait-based dependency injection ที่
ต่อยอดจาก trait ใน Part 19/21 ตรง ๆ

ตัวอย่าง `ShoppingCart` ท้ายบทรวมทุกเทคนิคเข้าด้วยกันเป็นภาพเดียว และควรเป็นแบบอ้างอิงเมื่อคุณต้องออกแบบ test suite
ของโมดูลใหม่ในโปรเจกต์จริงของตัวเองต่อไป — สิ่งสำคัญที่สุดที่ควรพกติดตัวไปจากบทนี้คือ **เทสที่ดีไม่ใช่เทสที่มี
จำนวนมากที่สุด แต่เป็นเทสที่ครอบคลุมทั้ง happy path, edge case, และเงื่อนไขที่ต้องถูกปฏิเสธอย่างถูกวิธี** พร้อม
เป็นอิสระจากกันและอ่านง่ายพอที่จะเป็นเอกสารอธิบาย behavior ของโค้ดได้ในตัวมันเอง

สิ่งที่บทนี้**ยังไม่ได้พูดถึง**คือการทดสอบ crate จากมุมมองภายนอก (เหมือนผู้ใช้ที่มา `use` crate ของคุณ เห็นแค่
public API เท่านั้น) และการจัดระเบียบ test suite ขนาดใหญ่ข้ามหลายไฟล์ผ่านโฟลเดอร์ `tests/` — นั่นคือเนื้อหาของ
**Part 33: Testing: Integration Tests และ Test Organization** ที่จะเรียนต่อจากบทนี้ทันที

---

**Part ก่อนหน้า:** [thiserror และ anyhow ในโปรเจกต์จริง](part-031-thiserror-anyhow.md) | **Part ถัดไป:** [Testing: Integration Tests และ Test Organization](part-033-testing-integration.md)
