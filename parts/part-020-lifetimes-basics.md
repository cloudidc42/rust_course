# Part 20: Lifetimes เบื้องต้น

> โมดูล: พื้นฐานภาษา Rust (Core Language Fundamentals) | ระดับ: พื้นฐาน | เวลาโดยประมาณ: 200 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้อย่างลึกซึ้งว่า **lifetime คืออะไรกันแน่** และทำไม borrow checker ถึงต้องมีแนวคิดนี้อยู่ในภาษา — ไม่ใช่แค่จำ
  syntax `'a` ไปใช้แบบท่องจำ แต่เข้าใจว่ามันคือ "คำอธิบายความสัมพันธ์ที่มีอยู่จริงในโค้ดอยู่แล้ว" ไม่ใช่เครื่องมือที่ไป
  "เปลี่ยน" อายุของข้อมูลใด ๆ เลย
- อ่านและแก้ error `E0106: missing lifetime specifier` ได้ในทุกบริบท ไม่ใช่แค่กรณี dangling reference ล้วน ๆ ที่เห็นใน
  Part 7 แต่รวมถึงกรณีที่ฟังก์ชันมี reference parameter มากกว่าหนึ่งตัวและ compiler ไม่สามารถเดาได้ว่า reference ที่คืน
  ออกมาควรผูกกับตัวไหน
- เขียน generic lifetime parameter (`'a`) บน function signature, struct definition, และ `impl` block ได้อย่างถูกต้อง
  และอธิบายความหมายที่แท้จริงของมันได้แม่นยำ (ไม่ใช่แค่ "ทำให้ compiler หยุดบ่น")
- ท่องจำและอธิบาย **กฎ lifetime elision ทั้ง 3 ข้อ** ได้ พร้อมอธิบายได้ว่าทำไมฟังก์ชันส่วนใหญ่ที่เขียนมาตลอด 19 บทที่ผ่านมา
  ไม่เคยต้องเขียน `'a` เองเลยแม้แต่ครั้งเดียว
- นิยาม struct ที่เก็บ reference ไว้เป็น field ได้อย่างถูกต้องสมบูรณ์ (ไม่ใช่แค่ทำตาม help ของ compiler แบบไม่เข้าใจ)
  และอธิบายได้ว่าทำไม struct instance หนึ่งตัว "ห้ามมีอายุยืนกว่า" ข้อมูลที่ field ของมันอ้างอิงถึง
- แยกแยะได้ว่า **`'static` lifetime** เหมาะสมกับสถานการณ์ไหนจริง ๆ และรู้จักหลีกเลี่ยง anti-pattern ที่มือใหม่มักทำ
  (ใช้ `'static` เป็นทางลัดแก้ compiler error แบบไม่เข้าใจต้นเหตุ)
- ผสมผสาน generics (จาก Part 18), trait bounds (จาก Part 19), และ lifetimes (บทนี้) ไว้ใน signature เดียวกันได้อย่าง
  เป็นธรรมชาติ และเข้าใจว่าทั้งสามระบบนี้เป็น "generic ประเภทต่างกัน" ที่ทำงานเสริมกัน ไม่ใช่แนวคิดที่แยกจากกันคนละโลก

## ความรู้ที่ต้องมีมาก่อน

บทนี้คือ**จุดบรรจบของทุกแนวคิดในโมดูล 1** — โดยเฉพาะอย่างยิ่ง:

- **Part 6 (Ownership เบื้องต้น)**: กฎ 3 ข้อของ ownership, move semantics, การ drop เมื่อหมด scope — lifetime คือ
  ระบบที่ทำให้ compiler พิสูจน์ได้ว่ากฎเหล่านี้จะไม่ถูกละเมิดแม้ผ่าน reference
- **Part 7 (Borrowing และ References)**: กฎ borrowing ทั้งสองข้อ, dangling reference, และที่สำคัญที่สุดคือ **error
  `E0106: missing lifetime specifier`** ที่เกิดกับฟังก์ชัน `dangle() -> &String` — บทนี้จะอธิบาย error ตัวนี้แบบเต็ม
  รูปแบบที่ Part 7 บอกไว้ว่า "จะเรียนเต็ม ๆ ใน Part 20"
- **Part 8 (Slices)**: `&str` และ `&[T]` เป็น type ที่ใช้เป็นตัวอย่างหลักตลอดบทนี้ เพราะ slice คือ reference ประเภทที่
  เจอปัญหาเรื่อง lifetime บ่อยที่สุดในโค้ดจริง
- **Part 9 (Structs)**: หัวข้อ 9.5 ที่เกริ่นไว้ว่า struct ที่เก็บ `&str` เป็น field ทำให้เกิด **error `E0106`** อีกครั้ง
  ในบริบทที่ต่างจาก Part 7 (struct definition ไม่ใช่ function signature) และบอกไว้ตรง ๆ ว่า "รายละเอียดเชิงลึกของ
  syntax นี้ ... จะเรียนเต็ม ๆ ใน Part 20" — บทนี้คือคำตอบเต็มรูปแบบของคำถามที่ค้างไว้นั้น
- **Part 18 (Generics เบื้องต้น)**: syntax `<T>` สำหรับนิยาม generic type parameter บนฟังก์ชันและ struct — บทนี้จะ
  แสดงให้เห็นว่า lifetime parameter (`'a`) มีไวยากรณ์และหลักการทำงานที่ **คล้ายกันมาก** กับ generic type parameter
  เพียงแต่ "generic เหนือระยะเวลา" ไม่ใช่ "generic เหนือชนิดข้อมูล"
- **Part 19 (Traits เบื้องต้น)**: trait bound (เช่น `T: Display`) ที่จะถูกนำมาผสมกับ lifetime parameter ในหัวข้อ 20.10
  ของบทนี้ เพื่อแสดงให้เห็นว่าทั้งสามระบบ (generics, traits, lifetimes) ทำงานร่วมกันได้อย่างเป็นธรรมชาติใน signature
  เดียวกัน

ถ้าคุณจำได้ว่า Part 7 ทิ้งท้ายไว้ว่า **"reference ที่ฟังก์ชันคืนออกมาต้องมีที่มาจาก reference parameter ที่รับเข้ามา
เสมอ ไม่สามารถสร้างขึ้นมาลอย ๆ จากตัวแปร local ได้"** และ Part 9 ทิ้งท้ายไว้ว่า **"struct ที่เก็บ reference ต้องมี
lifetime parameter เพื่อบอกว่า reference นั้นต้องมีอายุยืนไม่น้อยกว่า struct instance เอง"** — บทนี้จะอธิบายกลไก
เบื้องหลังคำพูดทั้งสองประโยคนี้อย่างละเอียดที่สุด จนคุณเข้าใจว่ามันไม่ใช่กฎที่แยกจากกันสองข้อ แต่เป็น**หลักการเดียวกัน
เพียงข้อเดียว**ที่ปรากฏในสองบริบทที่ต่างกัน

## เนื้อหา

### 20.1 ทวนความจำ: ทำไม Rust ถึงต้องมีแนวคิด "Lifetime" อยู่ในภาษา

จาก Part 7 เราเรียนรู้กฎเหล็กข้อที่ 2 ของ borrowing: **reference ทุกตัวต้องอ้างอิงถึงข้อมูลที่ยังมีชีวิตอยู่จริงเสมอ**
ห้ามเกิด dangling reference (reference ที่ชี้ไปยังข้อมูลที่ถูกทำลายไปแล้ว) เด็ดขาด — และเราเห็นว่า borrow checker
ตรวจสอบกฎนี้ได้อย่างสมบูรณ์ตอน **compile time** โดยไม่ต้องรอให้โปรแกรมรันจริงเลย นี่คือสิ่งที่ทำให้ Rust ปลอดภัยกว่า
C/C++ ในเรื่อง memory safety โดยไม่ต้องเสีย runtime cost แบบภาษาที่มี garbage collector

คำถามที่ยังไม่มีคำตอบเต็มรูปแบบจนถึงตอนนี้คือ: **borrow checker "รู้" ได้อย่างไรว่า reference ตัวหนึ่งจะยังไม่ dangling?**
สำหรับตัวแปร local ธรรมดาในฟังก์ชันเดียว (เช่น `let r = &x;` ที่เราเห็นมาตลอด Part 7) คำตอบตรงไปตรงมา: compiler มองเห็น
โค้ดทั้งฟังก์ชันได้ทั้งหมด มันวิเคราะห์ scope ของ `x` และ `r` ได้เองโดยไม่ต้องมีใครบอกอะไรเพิ่ม — นี่คือเหตุผลที่ตลอด
19 Part ที่ผ่านมา เราไม่เคยต้องเขียนอะไรพิเศษเพื่อบอก compiler เรื่อง "อายุ" ของ reference เลยแม้แต่ครั้งเดียว

แต่ปัญหาเกิดขึ้นทันทีที่ reference ต้อง **ข้ามขอบเขตของฟังก์ชัน** — เมื่อฟังก์ชันหนึ่งรับ reference เข้ามาเป็น parameter
แล้วต้อง**คืน reference ออกไป**เป็น return value หรือเมื่อ struct หนึ่งต้อง**เก็บ reference ไว้เป็น field** เพื่อใช้
ในอนาคต ในสถานการณ์แบบนี้ compiler ไม่มีทาง "มองเห็นทั้งภาพ" ได้อีกต่อไป เพราะฟังก์ชัน (หรือ struct) นั้นอาจถูกเรียกใช้
(หรือสร้าง instance) จากที่ไหนก็ได้ในโปรแกรม ด้วยตัวแปรต้นทางที่มีอายุยาวสั้นแตกต่างกันไปในแต่ละครั้งที่เรียก — compiler
จำเป็นต้องมี**สัญญา (contract)** ที่ระบุไว้ล่วงหน้าอย่างชัดเจนว่า "ความสัมพันธ์เรื่องอายุระหว่าง reference ที่รับเข้ามา
กับ reference ที่ส่งออกไป (หรือระหว่าง reference ที่เก็บไว้กับ struct ที่เก็บมัน) เป็นอย่างไร" — สัญญานี้แหละคือสิ่งที่
เรียกว่า **lifetime annotation**

**ข้อควรเข้าใจที่สำคัญที่สุดตั้งแต่ต้นบท (ป้องกันความเข้าใจผิดที่พบบ่อยที่สุด)**: lifetime annotation **ไม่ได้เปลี่ยน
อายุการมีชีวิตของข้อมูลใด ๆ เลยแม้แต่นิดเดียว** มันไม่ใช่คำสั่งที่บอกว่า "จงมีอายุยืนขึ้น" หรือ "จงตายเร็วลง" — อายุจริง
ของแต่ละตัวแปรถูกกำหนดโดยโครงสร้าง scope ของโค้ด (เหมือนที่เรียนมาตลอดใน Part 6-7) อยู่แล้วเหมือนเดิมทุกประการ ไม่ว่า
คุณจะเขียน `'a` หรือไม่เขียนเลยก็ตาม **สิ่งที่ lifetime annotation ทำคือการ "อธิบาย" ความสัมพันธ์ที่มีอยู่จริงอยู่แล้ว
ให้ compiler เข้าใจ** เพื่อให้มันพิสูจน์ได้ว่ากฎข้อที่ 2 ของ borrowing จะไม่ถูกละเมิดในทุกกรณีที่เป็นไปได้ — เปรียบเทียบ
ง่าย ๆ คือ lifetime annotation เหมือนกับการเขียน **type annotation** (`x: i32`) ทั่วไป: การเขียน `x: i32` ไม่ได้
"เปลี่ยน" ค่าของ `x` ให้กลายเป็น `i32` แต่เป็นการ**บอก**สิ่งที่ compiler ต้องรู้เพื่อตรวจสอบความถูกต้องเท่านั้น
lifetime annotation ก็ทำงานในลักษณะเดียวกันเป๊ะ ๆ เพียงแค่สิ่งที่มันบอกคือ "อายุสัมพัทธ์" แทนที่จะเป็น "ชนิดข้อมูล"

ข่าวดีคือ **กรณีส่วนใหญ่ที่พบในโค้ดจริง compiler เดาความสัมพันธ์นี้ได้เองโดยไม่ต้องให้เราเขียน `'a` เลย** ผ่านกลไกที่
เรียกว่า **lifetime elision** (การละ/งดเขียน lifetime เพราะมันชัดเจนพอที่ compiler จะเดาได้ถูกต้องเสมอ) ซึ่งเราจะเจาะลึก
กฎทั้ง 3 ข้อของมันในหัวข้อ 20.5 — แต่ก่อนจะไปถึงจุดนั้น เราต้องเห็นก่อนว่า **กรณีไหนที่ elision เดาไม่ได้** เพื่อเข้าใจ
ว่าทำไมเราต้องเขียน `'a` เองในบางสถานการณ์

### 20.2 ตัวอย่างคลาสสิกที่สุด: ฟังก์ชัน `longest` ที่ Compiler เดาไม่ได้

ลองเขียนฟังก์ชันที่ดูเรียบง่ายมาก: รับ string slice สองตัว (`&str` จาก Part 8) แล้วคืนตัวที่ยาวกว่า:

```rust
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}

fn main() {
    let s1 = String::from("Hello");
    let s2 = String::from("Rust");
    println!("{}", longest(&s1, &s2));
}
```

โค้ดนี้ดูเหมือนไม่มีอะไรผิดเลย — ทั้ง `x` และ `y` เป็น `&str` และเราคืนหนึ่งในสองตัวนั้นออกไปตรง ๆ ไม่มี local variable
สร้างขึ้นมาแบบ `dangle()` ใน Part 7 เลยด้วยซ้ำ แต่โค้ดนี้ **compile ไม่ผ่าน**:

```
error[E0106]: missing lifetime specifier
 --> src/main.rs:1:33
  |
1 | fn longest(x: &str, y: &str) -> &str {
  |               ----     ----     ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but the signature does not say whether it is borrowed from `x` or `y`
help: consider introducing a named lifetime parameter
  |
1 | fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
  |           ++++     ++          ++          ++
```

สังเกตว่า error code เดียวกัน (`E0106: missing lifetime specifier`) กับที่เจอในฟังก์ชัน `dangle()` ของ Part 7 —
แต่ **ข้อความ help บอกเหตุผลคนละแบบกันเลย** และนี่คือจุดสำคัญที่สุดที่ต้องแยกแยะให้ได้:

- ใน Part 7, `dangle()` ไม่มี reference parameter แม้แต่ตัวเดียว ข้อความ help บอกว่า **"there is no value for it to
  be borrowed from"** (ไม่มีค่าอะไรเลยให้ reference นี้ยืมมาจาก) — ปัญหาคือ **ไม่มีตัวเลือกให้ผูกอายุด้วยเลย**
- ในตัวอย่าง `longest()` นี้ มี reference parameter **สองตัว** (`x` และ `y`) ข้อความ help บอกตรง ๆ ว่า **"the
  signature does not say whether it is borrowed from `x` or `y`"** (signature ไม่ได้บอกว่า reference ที่คืนออกมา
  ยืมมาจาก `x` หรือ `y`) — ปัญหาคือ **มีตัวเลือกสองตัว แต่ compiler ไม่รู้ว่าต้องเลือกตัวไหน**

ทั้งสองกรณีคือการละเมิด **หลักการเดียวกัน**: "ฟังก์ชันที่คืน reference ต้องบอกให้ compiler รู้ว่า reference นั้นมี
ที่มาจากไหน" — เพียงแต่สาเหตุที่ compiler เดาไม่ได้แตกต่างกัน (ไม่มีตัวเลือกเลย vs มีตัวเลือกแต่กำกวม)

#### ทำไม compiler เดาไม่ได้จริง ๆ: วิเคราะห์เชิงลึก

ลองนึกภาพว่าถ้า Rust "ยอม" ให้โค้ดนี้ compile ผ่านโดยไม่มี lifetime annotation ใด ๆ เลย มันจะเกิดปัญหาอะไรตามมา? ลอง
พิจารณาสถานการณ์การใช้งานจริงที่ `x` และ `y` มีอายุแตกต่างกันมาก:

```rust
fn longest(x: &str, y: &str) -> &str { // (โค้ดสมมติที่ compile ไม่ผ่านจริง — ใช้เพื่ออธิบายเหตุผลเท่านั้น)
    if x.len() > y.len() { x } else { y }
}

fn main() {
    let s1 = String::from("ข้อความที่มีอายุยืนตลอดทั้งฟังก์ชัน main");
    let result;
    {
        let s2 = String::from("สั้น"); // s2 มีอายุสั้นกว่า s1 มาก — จะถูกทำลายทิ้งตอนจบ block นี้
        result = longest(&s1, &s2);    // ถ้า s2 ยาวกว่า s1 result จะกลายเป็น reference ไปยัง s2!
    } // s2 หมดอายุตรงนี้ — ถูก drop
    println!("{result}"); // ถ้า result ชี้ไปยัง s2 ที่ตายไปแล้ว นี่คือ dangling reference โดยสมบูรณ์!
}
```

ปัญหาคือ **compiler ไม่มีทางรู้ล่วงหน้าได้เลยว่า `if x.len() > y.len()` จะประเมินผลเป็น `true` หรือ `false` ตอน compile
time** (มันขึ้นกับข้อมูลจริงตอน runtime) ดังนั้น **มันต้องเผื่อไว้เสมอว่า `result` อาจกลายเป็น reference ไปยัง `y`
(คือ `s2`) ได้** และถ้า `y` มีอายุสั้นกว่าที่ `result` จะถูกใช้งานต่อ (เหมือนในตัวอย่างข้างบน) ก็จะเกิด dangling
reference ทันที **นี่คือเหตุผลที่แท้จริงที่ compiler ต้องปฏิเสธฟังก์ชันนี้ตั้งแต่ตอนดู signature อย่างเดียว** — มันไม่
สามารถพิสูจน์ความปลอดภัยได้ ถ้าไม่รู้ก่อนว่า "อายุของ input ทั้งสองตัวสัมพันธ์กับอายุของ output อย่างไร"

พูดให้ชัดที่สุด: **ปัญหาไม่ได้อยู่ที่ตัวโค้ดข้างในฟังก์ชัน (`if x.len() > y.len() { x } else { y }` ไม่มีอะไรผิดเลย)
แต่อยู่ที่ signature ของฟังก์ชันไม่ได้ "สัญญา" อะไรไว้เกี่ยวกับความสัมพันธ์ของอายุระหว่าง input กับ output** — และ
compiler ต้องตรวจสอบ signature นี้แยกจากการวิเคราะห์เนื้อโค้ดข้างในเสมอ เพราะฟังก์ชันอาจถูกเรียกจากที่อื่นที่ compiler
ยังไม่เห็นตอนนี้ก็ได้ (โดยเฉพาะถ้าฟังก์ชันนี้ถูก export ออกไปเป็น public API ให้ crate อื่นเรียกใช้)

### 20.3 ทางแก้: Generic Lifetime Parameter `'a`

วิธีแก้ตามที่ compiler แนะนำคือเพิ่ม **lifetime parameter** ชื่อ `'a` (อ่านว่า "tick-a" หรือ "lifetime a") เข้าไปทั้งที่
parameter และ return type:

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}

fn main() {
    let s1 = String::from("Hello world");
    let s2 = String::from("Rust");
    let result = longest(&s1, &s2);
    println!("ประโยคที่ยาวกว่าคือ: {result}");
}
```

ผลลัพธ์:

```
ประโยคที่ยาวกว่าคือ: Hello world
```

โค้ดนี้ compile ผ่านทันที มาแยกวิเคราะห์ syntax ทีละส่วน:

1. **`<'a>` หลังชื่อฟังก์ชัน**: นี่คือการ**ประกาศ** lifetime parameter ชื่อ `'a` ให้ฟังก์ชันนี้ — สังเกตตำแหน่งที่เขียน
   เทียบกับ generic type parameter จาก Part 18 (`fn largest<T>(list: &[T]) -> T`) **มันอยู่ในตำแหน่งเดียวกันเป๊ะ ๆ**
   คือหลังชื่อฟังก์ชัน ก่อนวงเล็บ parameter — เพราะในทางเทคนิค **lifetime parameter คือ generic parameter ประเภทหนึ่ง**
   เพียงแต่เป็น "generic เหนือระยะเวลาการมีชีวิต (duration of validity)" ไม่ใช่ "generic เหนือชนิดข้อมูล (type)"
2. **`x: &'a str` และ `y: &'a str`**: บอกว่าทั้ง `x` และ `y` เป็น reference ที่มี lifetime **เดียวกัน** ชื่อ `'a`
3. **`-> &'a str`**: บอกว่า reference ที่คืนออกมาก็มี lifetime **เดียวกัน** `'a` นั้นด้วย

**สิ่งที่ signature นี้ "สัญญา" ไว้กับ compiler คือ**: "ไม่ว่า caller จะส่ง `x` และ `y` ที่มีอายุจริงต่างกันแค่ไหนก็ตาม
reference ที่ฟังก์ชันนี้คืนออกมาจะมีอายุใช้งานได้ปลอดภัยแค่**ช่วงเวลาที่สั้นที่สุดร่วมกัน**ระหว่างอายุของ `x` กับอายุ
ของ `y` เท่านั้น — ไม่มากกว่านั้น" นี่คือสัญญาที่ compiler ใช้ตรวจสอบทุกจุดที่เรียกใช้ `longest` เพื่อพิสูจน์ว่ากฎ
ห้าม dangling reference จะไม่ถูกละเมิดเลยไม่ว่า caller จะเรียกด้วยอะไรก็ตาม เราจะพิสูจน์ประโยคนี้ให้เห็นจริงในหัวข้อ
ถัดไปด้วยตัวอย่างที่ compile **ไม่ผ่าน**อย่างจงใจ

#### เปรียบเทียบคู่กันชัด ๆ: Generic Type Parameter (Part 18) vs Generic Lifetime Parameter (บทนี้)

เพื่อให้เห็นภาพว่าทั้งสองระบบนี้เป็น "ญาติกัน" ทางไวยากรณ์และหลักการมากแค่ไหน ลองเทียบฟังก์ชัน generic ทั่วไปจาก
Part 18 กับฟังก์ชัน `longest` ข้างบนแบบเคียงข้างกัน:

```rust
// Generic เหนือ "ชนิดข้อมูล" — T คือ placeholder แทน type ที่จะรู้ตอน compile จากจุดที่เรียกใช้
fn largest<T: PartialOrd + Copy>(list: &[T]) -> T {
    let mut result = list[0];
    for &item in list {
        if item > result {
            result = item;
        }
    }
    result
}

// Generic เหนือ "อายุการมีชีวิต" — 'a คือ placeholder แทน lifetime ที่จะรู้ตอน compile จากจุดที่เรียกใช้
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

fn main() {
    let numbers = vec![3, 7, 2, 9, 4];
    println!("ค่ามากที่สุด: {}", largest(&numbers));

    let s1 = String::from("Hello world");
    let s2 = String::from("Rust");
    println!("ยาวกว่าคือ: {}", longest(&s1, &s2));
}
```

ผลลัพธ์:

```
ค่ามากที่สุด: 9
ยาวกว่าคือ: Hello world
```

ตารางเปรียบเทียบเชิงหลักการ (ไม่ใช่แค่ syntax) ระหว่างสองระบบนี้:

| แง่มุม | Generic Type Parameter (`T`) | Generic Lifetime Parameter (`'a`) |
|---|---|---|
| ตอบคำถามอะไร | "ชนิดข้อมูลตรงนี้คืออะไร" | "reference ตรงนี้มีอายุใช้งานได้นานแค่ไหน" |
| ประกาศที่ไหน | `<T>` หลังชื่อฟังก์ชัน/struct | `<'a>` หลังชื่อฟังก์ชัน/struct (ตำแหน่งเดียวกัน) |
| ใครกำหนดค่าจริง | compiler อนุมาน (infer) จาก argument ที่ caller ส่งมาจริง | compiler อนุมานจาก scope จริงของ reference ที่ caller ส่งมาจริง |
| มี "ค่าจริง" หลุดไปตอน runtime หรือไม่ | ไม่มี — ถูกลบออกหมดตอน compile (monomorphization, จะเรียนเต็มใน Part 22) | ไม่มี — ถูกลบออกหมดตอน compile เช่นกัน ไม่มี "ตัวแปรอายุ" หลงเหลืออยู่ใน binary เลย |
| ป้องกันอะไร | type mismatch (ผสม type ที่เข้ากันไม่ได้) | dangling reference (ใช้ข้อมูลที่ถูกทำลายไปแล้ว) |
| cost ตอน runtime | ศูนย์ (zero-cost abstraction) | ศูนย์ (zero-cost abstraction) — สำคัญมาก อธิบายเพิ่มด้านล่างนี้ทันที |

ข้อสังเกตสุดท้ายในตารางคือหัวใจสำคัญที่มือใหม่มักเข้าใจผิด: **lifetime annotation ไม่มี "ตัวตน" อะไรเหลืออยู่ในโปรแกรม
ที่ compile เสร็จแล้วเลยแม้แต่ byte เดียว** มันเป็นแค่ข้อมูลที่ borrow checker ใช้ตรวจสอบตอน compile time เท่านั้น
เมื่อ compile ผ่านแล้ว เครื่องหมาย `'a` ทั้งหมดจะถูก "ลบ" ออกไปโดยสิ้นเชิง เหลือแค่เครื่อง machine code ที่ทำงานกับ
pointer ธรรมดา ๆ ไม่ต่างจากโค้ด C ที่ไม่มี garbage collector หรือ runtime check ใด ๆ เลย — นี่คือความหมายที่แท้จริง
ของคำว่า **zero-cost abstraction** ที่เราพูดถึงมาตั้งแต่ Part 1: ความปลอดภัยที่ได้มา**ไม่มีราคาที่ต้องจ่ายตอนโปรแกรม
รันจริงเลยแม้แต่นิดเดียว** ทุกอย่างถูกพิสูจน์เสร็จสิ้นไปแล้วตั้งแต่ตอน compile

### 20.4 ความหมายที่แท้จริงของ `'a`: อายุที่สั้นที่สุดร่วมกัน (The Shorter of the Two)

ตอนนี้เรารู้ syntax แล้ว แต่ต้องเข้าใจ **ความหมายที่แม่นยำ** ของมันด้วย เพราะนี่คือจุดที่มือใหม่จำนวนมากเข้าใจผิดบ่อย
ที่สุด: **`fn longest<'a>(x: &'a str, y: &'a str) -> &'a str` ไม่ได้แปลว่า "x และ y ต้องมีอายุยาวเท่ากันเป๊ะ ๆ"**
แต่แปลว่า **"lifetime `'a` ที่ compiler จะอนุมานให้จริง ๆ ในแต่ละครั้งที่เรียกใช้ฟังก์ชันนี้ คือ lifetime ที่สั้นที่สุด
ร่วมกัน (the smaller/shorter of the two) ระหว่างอายุจริงของ `x` กับอายุจริงของ `y`" — และ reference ที่คืนออกมา
จะมีสัญญาว่าใช้งานได้ปลอดภัยแค่ในช่วงเวลานั้นเท่านั้น** ไม่มากไปกว่านั้น แม้ว่าในความเป็นจริงมันอาจมาจากตัวที่มีอายุยืน
กว่าก็ตาม (เพราะ compiler ไม่รู้ล่วงหน้าว่าจะคืนตัวไหนจนกว่าจะรันจริง จึงต้อง "การันตีแบบระมัดระวังที่สุด" คือใช้อายุ
ที่สั้นกว่าเป็นขอบเขตความปลอดภัยเสมอ)

ลองพิสูจน์ประโยคนี้ให้เห็นจริงด้วยตัวอย่างที่ scope ของ `x` และ `y` ไม่เท่ากัน:

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}

fn main() {
    let s1 = String::from("Hello world"); // s1 มีอายุยืนตลอดทั้ง main
    let result;
    {
        let s2 = String::from("Rust"); // s2 มีอายุสั้นกว่า — จะถูก drop เมื่อจบ block นี้
        result = longest(&s1, &s2);    // ❌ ลองดูว่า compiler ยอมให้ทำแบบนี้หรือไม่
    }
    println!("ประโยคที่ยาวกว่าคือ: {result}"); // ใช้ result หลังจาก s2 ถูก drop ไปแล้ว
}
```

โค้ดนี้ **compile ไม่ผ่าน**:

```
error[E0597]: `s2` does not live long enough
  --> src/main.rs:14:31
   |
13 |         let s2 = String::from("Rust"); // s2 มีอายุสั้นกว่า — จะถูก drop เมื่อจบ block นี้
   |             -- binding `s2` declared here
14 |         result = longest(&s1, &s2);    // ❌ ลองดูว่า compiler ยอมให้ทำแบบนี้หรือไม่
   |                               ^^^ borrowed value does not live long enough
15 |     }
   |     - `s2` dropped here while still borrowed
16 |     println!("ประโยคที่ยาวกว่าคือ: {result}"); // ใช้ result หลังจาก s2 ถูก drop ไปแล้ว
   |                                 ------ borrow later used here
```

#### วิเคราะห์ error นี้อย่างละเอียด: ทำไม compiler ปฏิเสธ ทั้ง ๆ ที่จริง ๆ แล้ว `s1` ยาวกว่า `s2` เสมอ

จุดที่มือใหม่งงมากที่สุดในตัวอย่างนี้คือ: **"ในโค้ดจริง `s1.len()` (11) มากกว่า `s2.len()` (4) เสมอ ดังนั้น `result`
จะเท่ากับ `x` (คือ `&s1`) เสมอ ไม่มีทางเป็น `&s2` เลย แล้วทำไม compiler ยังปฏิเสธ?"** — คำตอบคือ **borrow checker
ไม่ได้รันโปรแกรมจริงเพื่อดูว่า `if x.len() > y.len()` จะเป็น `true` หรือ `false`** มันวิเคราะห์แค่ **signature** ของ
ฟังก์ชัน `longest` เท่านั้น และ signature บอกไว้ชัดว่า "output มี lifetime `'a` เดียวกันกับทั้ง input สองตัว" —
ในทางตรรกะแปลว่า **output ต้องมีอายุสั้นเท่ากับตัวที่สั้นที่สุดในสองตัวเสมอ ไม่ว่าเนื้อโค้ดข้างในจะเลือกคืนตัวไหน
จริง ๆ ก็ตาม** compiler เลือกที่จะ**ไม่วิเคราะห์เนื้อโค้ดข้างในฟังก์ชัน (control flow analysis) เพื่อพิสูจน์ความ
ปลอดภัยของ lifetime** เพราะสิ่งนี้จะทำให้การตรวจสอบ **ซับซ้อนเกินไปและเปราะบางเกินไป** (แค่แก้ logic เล็ก ๆ ข้างใน
ฟังก์ชัน โดยไม่แก้ signature เลย ก็อาจทำให้ผลการวิเคราะห์เปลี่ยนไปได้ ซึ่งอันตรายมากสำหรับ public API ที่ผู้อื่นเรียก
ใช้โดยไม่เห็นเนื้อโค้ดข้างในด้วยซ้ำ) — Rust เลือกใช้หลักการที่**ปลอดภัยที่สุดเสมอ (conservative/pessimistic)**: ตรวจ
สอบจาก **signature เพียงอย่างเดียว** โดยไม่สนใจว่าเนื้อโค้ดข้างในจะ "บังเอิญ" ปลอดภัยกว่าที่ signature สัญญาไว้หรือไม่

พูดอีกแบบ: **signature `fn longest<'a>(x: &'a str, y: &'a str) -> &'a str` คือสัญญาที่ใช้ตรวจสอบ "ทุกความเป็นไปได้"
ของการเรียกใช้ฟังก์ชันนี้ ไม่ใช่แค่กรณีเฉพาะที่คุณเขียนอยู่ตรงหน้า** ถ้าวันหนึ่งมีคนแก้ไข logic ข้างในฟังก์ชัน (เช่น
เปลี่ยนเงื่อนไขจาก `>` เป็น `<` โดยไม่ได้ตั้งใจ) โค้ดที่เรียกใช้ `longest` จากภายนอกก็ยังต้อง **ปลอดภัยเหมือนเดิมทุก
ประการโดยไม่ต้องแก้ไขอะไรเลย** เพราะ compiler รับประกันความปลอดภัยจาก signature ล้วน ๆ ไม่ได้พึ่งพา behavior ที่แท้จริง
ของเนื้อโค้ดข้างในแม้แต่นิดเดียว — นี่คือข้อดีเชิง **software engineering** ที่สำคัญมาก: **signature ของฟังก์ชันคือ
"สัญญาที่เสถียร" ที่ผู้เรียกใช้พึ่งพาได้เสมอ ไม่ต้องกังวลว่าการแก้ไข implementation ภายในจะทำให้โค้ดที่เรียกใช้อยู่
พังกลางทาง**

#### วิธีแก้: จัดโครงสร้าง scope ให้สอดคล้องกับความหมายของสัญญา

ถ้าต้องการให้โค้ดนี้ compile ผ่าน มีสองทางเลือกหลัก ขึ้นอยู่กับว่าคุณต้องการอะไรจริง ๆ:

**ทางเลือกที่ 1**: ถ้า `result` แค่ต้องใช้งานในช่วงที่ทั้ง `s1` และ `s2` ยังมีชีวิตอยู่พร้อมกัน ให้ย้ายการใช้งานเข้าไป
อยู่ใน scope เดียวกัน:

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

fn main() {
    let s1 = String::from("Hello world");
    {
        let s2 = String::from("Rust");
        let result = longest(&s1, &s2);
        println!("ประโยคที่ยาวกว่าคือ: {result}"); // ✅ ใช้ result ก่อนที่ s2 จะหมดอายุ
    }
}
```

**ทางเลือกที่ 2**: ถ้าต้องการเก็บผลลัพธ์ไว้ใช้นานกว่าอายุของ `s2` ให้เปลี่ยนเป็นคืน **owned value** (`String`) แทน
reference — ย้อนกลับไปใช้หลักการเดียวกันกับ "วิธีแก้ dangling reference" จาก Part 7 หัวข้อ 7.8:

```rust
fn longest_owned(x: &str, y: &str) -> String {
    // คืน String ที่ .clone() หรือสร้างใหม่ แทนการคืน reference — ไม่ผูกอายุกับ x, y อีกต่อไป
    if x.len() > y.len() {
        x.to_string()
    } else {
        y.to_string()
    }
}

fn main() {
    let s1 = String::from("Hello world");
    let result;
    {
        let s2 = String::from("Rust");
        result = longest_owned(&s1, &s2); // ไม่มี lifetime ผูกกับ s2 อีกแล้ว เพราะคืน owned String
    } // s2 หมดอายุตรงนี้ได้อย่างปลอดภัย เพราะ result ไม่ได้ยืมมันอยู่
    println!("ประโยคที่ยาวกว่าคือ: {result}"); // ✅ compile ผ่าน — result เป็นเจ้าของข้อมูลของตัวเองแล้ว
}
```

ผลลัพธ์:

```
ประโยคที่ยาวกว่าคือ: Hello world
```

สังเกตว่า **ทางเลือกที่ 2 แลกความยืดหยุ่นเรื่อง scope กับ cost การ allocate memory ใหม่ทุกครั้งที่เรียก** (จาก
`.to_string()`) — นี่คือ trade-off แบบเดียวกันที่เจอมาตลอดหลักสูตร: **owned value ยืดหยุ่นกว่าแต่มี cost, reference
ไม่มี cost แต่ถูกจำกัดด้วยกฎเรื่องอายุ** การเลือกใช้แบบไหนขึ้นอยู่กับว่าสถานการณ์จริงต้องการอะไร ไม่มีคำตอบที่ถูกเสมอ
ไปแบบเดียว

#### หมายเหตุสำคัญ: `'a` (Lifetime Parameter) กับ NLL (จาก Part 7) เป็นคนละเรื่องกัน แต่ทำงานร่วมกัน

Part 7 หัวข้อ 7.7 สอนเรื่อง **Non-Lexical Lifetimes (NLL)** — กลไกที่ทำให้ borrow checker วิเคราะห์ "อายุจริง" ของ
reference แบบ local ตัวหนึ่ง ๆ ได้อย่างละเอียด (จบตรงจุดที่ใช้งานครั้งสุดท้าย ไม่ใช่รอถึงปีกกาปิดของ scope) มือใหม่
บางคนพอเรียนคำว่า "lifetime" ในบทนี้แล้วอาจสับสนคิดว่า **`'a` ที่เราเขียนเองคือสิ่งเดียวกับ "lifetime" ที่ NLL คำนวณ
ให้อัตโนมัติใน Part 7** — ทั้งสองเกี่ยวข้องกันแต่**ไม่ใช่สิ่งเดียวกัน**:

- **"Lifetime" ที่ NLL วิเคราะห์ (Part 7)** คือ **ช่วงเวลาที่แท้จริง** (a concrete region of code) ที่ reference
  ตัวหนึ่ง ๆ ยังถูกใช้งานอยู่จริงในโปรแกรม — เป็นสิ่งที่ compiler **คำนวณเอง** จากการไล่วิเคราะห์โค้ดทั้งหมด ไม่มีชื่อ
  ให้เราเรียก ไม่ต้องเขียนเองเลย
- **`'a` (Lifetime Parameter, บทนี้)** คือ **ชื่อของ placeholder** ที่ใช้ในการเขียน signature ของฟังก์ชัน/struct
  เพื่ออธิบาย**ความสัมพันธ์**ระหว่างช่วงเวลาที่แท้จริงหลาย ๆ ช่วง (ที่ NLL จะไปคำนวณอีกทีตอนตรวจสอบจุดที่เรียกใช้จริง)
  — `'a` เองไม่มี "ความยาว" ตายตัว มันแค่บอกว่า "ช่วงเวลาที่แท้จริงของ x และช่วงเวลาที่แท้จริงของ output ต้องสัมพันธ์
  กันแบบนี้" เท่านั้น

พูดให้เห็นภาพว่าทั้งสองทำงานร่วมกันอย่างไร: เมื่อคุณเรียก `longest(&s1, &s2)` compiler จะ (1) ใช้ **NLL** คำนวณช่วง
เวลาจริงที่ `&s1` และ `&s2` แต่ละตัวถูกใช้งานอยู่ในจุดที่เรียกนั้น แล้ว (2) ใช้ **สัญญาที่ `'a` ประกาศไว้ใน signature**
ตรวจสอบว่าช่วงเวลาจริงเหล่านั้นสอดคล้องกับความสัมพันธ์ที่ `'a` กำหนดไว้หรือไม่ (เช่น output ต้องไม่ถูกใช้เกินกว่าช่วง
เวลาที่สั้นที่สุดร่วมกัน) — **NLL ให้ "ข้อมูลจริง" ส่วน lifetime parameter ให้ "กฎที่ต้องตรวจสอบกับข้อมูลจริงนั้น"**
ทั้งสองเป็นคนละชั้นของระบบเดียวกัน (borrow checker) ที่ทำงานประสานกันเสมอ ไม่ใช่คนละระบบที่แยกจากกัน

### 20.5 กฎ Lifetime Elision: ทำไมเราไม่เคยต้องเขียน `'a` มาก่อนเลยตลอด 19 Part

ถึงจุดนี้คุณอาจสงสัยว่า **"ถ้าฟังก์ชันที่คืน reference ต้องมี lifetime annotation เสมอ แล้วทำไมตลอด 19 Part ที่ผ่านมา
เราเขียนฟังก์ชันที่รับ/คืน `&str`, `&[T]` เต็มไปหมด แต่ไม่เคยต้องเขียน `'a` เลยแม้แต่ครั้งเดียว?"** คำตอบคือ **Rust มี
กฎที่เรียกว่า lifetime elision rules (กฎการละ/งดเขียน lifetime)** ซึ่งเป็นชุดกฎที่ compiler ใช้ **เดา (infer)**
lifetime annotation ให้อัตโนมัติในกรณีที่ **รูปแบบของ signature ชัดเจนพอจนไม่มีทางกำกวม** — กฎเหล่านี้ไม่ได้ทำให้
กฎเรื่อง lifetime หายไป แต่เป็นการที่ compiler "เขียน `'a` ให้เราโดยไม่ต้องพิมพ์เอง" ในกรณีที่เดาได้ถูกต้อง 100%
เสมอ

**สิ่งสำคัญที่ต้องเข้าใจ**: กฎ elision ไม่ใช่ "การอนุมานแบบ AI ที่ฉลาดขึ้นเรื่อย ๆ" แต่เป็น **3 กฎที่ตายตัว ตรวจสอบ
ตามลำดับ** ถ้าทำตามกฎครบ 3 ข้อแล้วยังมี output lifetime ที่ไม่รู้ว่าจะผูกกับอะไร (เหมือนกรณี `longest` ที่มี input
สองตัว) compiler จะปฏิเสธและขอให้เราเขียน `'a` เอง — มาดูกฎแต่ละข้อพร้อมตัวอย่างที่แสดงให้เห็นว่ามันทำงานอย่างไรจริง ๆ

#### กฎข้อที่ 1: Input Lifetime แต่ละตัวได้ Lifetime Parameter ของตัวเองเสมอ

**กฎ**: reference parameter ทุกตัวที่ไม่ได้เขียน lifetime ไว้ จะได้รับ lifetime parameter ของตัวเอง (ที่ไม่ซ้ำกับ
ตัวอื่น) โดยอัตโนมัติ — ไม่ว่าจะมี reference parameter กี่ตัวก็ตาม

พูดง่าย ๆ: `fn foo(x: &i32, y: &i32)` ถูก compiler มองเป็น `fn foo<'a, 'b>(x: &'a i32, y: &'b i32)` ภายใน (คุณไม่เห็น
`'a`, `'b` นี้ในโค้ดที่เขียน แต่ compiler ใช้มันในการวิเคราะห์เบื้องหลัง) — สังเกตว่ากฎนี้ **ให้ lifetime คนละตัวกัน**
แก่ `x` และ `y` โดย default (ไม่ใช่ตัวเดียวกันแบบที่เราเขียนเองใน `longest<'a>`) นี่คือเหตุผลว่าทำไมฟังก์ชันที่มีแค่
reference parameter แต่**ไม่คืน reference ออกมาเลย** ไม่มีปัญหาอะไร แม้จะมี reference หลายตัว:

```rust
// มี reference parameter สองตัว (rule 1: แต่ละตัวได้ lifetime ของตัวเอง)
// แต่ไม่คืน reference ออกมาเลย — จึงไม่มี output lifetime ที่ต้องเดาว่าผูกกับตัวไหน ไม่มีปัญหาเลย
fn compare_lengths(x: &str, y: &str) -> bool {
    x.len() > y.len()
}

fn main() {
    let s1 = String::from("Hello world");
    let s2 = String::from("Rust");
    println!("s1 ยาวกว่า s2 หรือไม่: {}", compare_lengths(&s1, &s2));
}
```

ผลลัพธ์:

```
s1 ยาวกว่า s2 หรือไม่: true
```

กฎข้อที่ 1 เพียงอย่างเดียวไม่เพียงพอที่จะแก้ปัญหาของ `longest` เพราะ `longest` **คืน reference ออกมาด้วย** — กฎข้อที่ 1
แค่บอกว่า input แต่ละตัวได้ lifetime ของตัวเอง แต่ไม่ได้บอกว่า **output lifetime** ควรผูกกับตัวไหน นั่นคือหน้าที่ของ
กฎข้อที่ 2 และ 3

#### กฎข้อที่ 2: ถ้ามี Input Lifetime เพียงตัวเดียว มันถูกใช้กับ Output Lifetime ที่ elided ทั้งหมด

**กฎ**: ถ้าฟังก์ชันมี input lifetime parameter (หลังใช้กฎข้อที่ 1 แล้ว) **เหลืออยู่แค่ตัวเดียว** lifetime ตัวนั้นจะถูก
ใช้กับ output lifetime ที่ elided (ไม่ได้เขียนไว้) ทุกตัวโดยอัตโนมัติ

**ข้อสังเกตสำคัญ**: กฎนี้สนใจว่า "มี reference parameter ที่ไม่ซ้ำกันกี่ตัว" ไม่ใช่ "มี parameter กี่ตัว" — ฟังก์ชันที่
มี parameter หลายตัวแต่มี**reference แค่ตัวเดียว**ก็ยังเข้าเงื่อนไขกฎข้อที่ 2 ได้:

```rust
// rule 1: s ได้ lifetime ของตัวเอง (สมมติเรียกว่า 'a) — เป็น reference parameter ตัวเดียวในฟังก์ชันนี้
// (n: usize ไม่ใช่ reference จึงไม่มี lifetime เกี่ยวข้องเลย)
// rule 2: มี input lifetime เหลือแค่ตัวเดียว ('a) -> ใช้ 'a นั้นกับ output ที่ elided ทั้งหมด
// เทียบเท่ากับเขียนเต็ม ๆ ว่า: fn first_n_chars<'a>(s: &'a str, n: usize) -> &'a str
fn first_n_chars(s: &str, n: usize) -> &str {
    let end = s.char_indices().nth(n).map(|(i, _)| i).unwrap_or(s.len());
    &s[..end]
}

// อีกตัวอย่างคลาสสิกที่ใช้กฎเดียวกันนี้: หาคำแรกในประโยค (ทบทวนจากแนวคิดของ Part 8)
// rule 1: s ได้ 'a เป็นของตัวเอง (parameter เดียว ไม่มีตัวอื่นแข่ง)
// rule 2: input lifetime เหลือตัวเดียว -> output ที่ elided ก็ได้ 'a ไปด้วย
fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();
    for (i, &b) in bytes.iter().enumerate() {
        if b == b' ' {
            return &s[..i];
        }
    }
    s
}

fn main() {
    let sentence = String::from("สวัสดี ครับ ผม เขียน Rust");
    println!("6 ตัวอักษรแรก: {}", first_n_chars(&sentence, 6));
    println!("คำแรก: {}", first_word(&sentence));
}
```

ผลลัพธ์:

```
6 ตัวอักษรแรก: สวัสดี
คำแรก: สวัสดี
```

**นี่คือคำตอบที่แท้จริงว่าทำไมฟังก์ชันแบบ `first_word` (ซึ่งเป็นรูปแบบที่เจอบ่อยที่สุดในโค้ด Rust จริง — ฟังก์ชันที่
รับ slice ตัวเดียวแล้วคืน slice ย่อยของมัน) ไม่เคยต้องเขียน `'a` เลย** เพราะมันตรงกับกฎข้อที่ 2 เป๊ะ ๆ: input
reference มีตัวเดียว output ก็ต้องผูกกับตัวเดียวนั้นอย่างไม่มีทางกำกวม compiler จึงเดาได้ถูกต้อง 100% เสมอโดยไม่ต้อง
ถามเราเลย

ลองเขียนแบบเต็ม (ไม่ใช้ elision) เพื่อยืนยันว่าทั้งสองแบบเทียบเท่ากันจริง — compile ได้ผลลัพธ์เดียวกันทุกประการ:

```rust
// เขียนแบบเต็ม (explicit) — เทียบเท่ากับ first_word แบบ elided ทุกประการ ไม่ต่างกันแม้แต่นิดเดียวตอน compile
fn first_word_explicit<'a>(s: &'a str) -> &'a str {
    let bytes = s.as_bytes();
    for (i, &b) in bytes.iter().enumerate() {
        if b == b' ' {
            return &s[..i];
        }
    }
    s
}

fn main() {
    let sentence = String::from("สวัสดี ครับ ผม เขียน Rust");
    println!("คำแรก: {}", first_word_explicit(&sentence));
}
```

ผลลัพธ์:

```
คำแรก: สวัสดี
```

**ทำไม `longest` ไม่เข้าเงื่อนไขกฎข้อที่ 2**: เพราะ `longest` มี reference parameter **สองตัว** (`x` และ `y`) หลังใช้
กฎข้อที่ 1 จะได้ input lifetime สองตัวที่ไม่ซ้ำกัน (`'a` สำหรับ `x`, `'b` สำหรับ `y`) — กฎข้อที่ 2 ใช้ได้เฉพาะเมื่อ
**เหลือ input lifetime พอดีหนึ่งตัว** เท่านั้น เมื่อมีสองตัว compiler ไม่มีทางเลือกได้ว่าจะใช้ `'a` หรือ `'b` กับ output
นี่คือจุดที่กฎข้อที่ 2 "หมดหน้าที่" และต้องไปดูกฎข้อที่ 3 ต่อ

#### กฎข้อที่ 3: ถ้ามี `&self` หรือ `&mut self` Lifetime ของมันถูกใช้กับ Output Lifetime ที่ elided ทั้งหมด

**กฎ**: ถ้าฟังก์ชันเป็น method (มี `&self` หรือ `&mut self` เป็น parameter ตัวแรก) lifetime ของ `self` จะถูกใช้กับ
output lifetime ที่ elided ทุกตัวโดยอัตโนมัติ **ไม่ว่าจะมี reference parameter อื่นอยู่ด้วยกี่ตัวก็ตาม** — กฎนี้มี
ความสำคัญพิเศษเพราะมันคือกฎที่ทำให้ **method ส่วนใหญ่ที่คืน reference จาก struct ของตัวเอง ไม่ต้องเขียน `'a` เลย**
เราจะเจาะลึกกฎนี้เต็มรูปแบบพร้อมตัวอย่าง struct จริงในหัวข้อ 20.8 (เพราะมันผูกกับเรื่อง lifetime บน struct โดยตรง)
แต่ให้ดูตัวอย่างสั้น ๆ ก่อนเพื่อให้เห็นภาพรวมของกฎทั้ง 3 ข้อครบถ้วน:

```rust
struct TextHolder {
    content: String,
}

impl TextHolder {
    // rule 1: &self ได้ lifetime ของตัวเอง (สมมติ 's) — เป็น reference parameter ตัวเดียวที่นี่
    // (ไม่มี parameter อื่นที่เป็น reference)
    // rule 2: ไม่เข้าเงื่อนไข เพราะกฎข้อ 2 ใช้กับ "input lifetime" ทั่วไป แต่ในเมื่อมี &self อยู่ กฎข้อ 3 จะถูกใช้แทน
    // rule 3: มี &self -> lifetime ของ self ('s) ถูก assign ให้ output ที่ elided
    // เทียบเท่ากับ: fn get_content<'s>(&'s self) -> &'s str
    fn get_content(&self) -> &str {
        &self.content
    }
}

fn main() {
    let holder = TextHolder { content: String::from("ข้อมูลใน struct") };
    println!("{}", holder.get_content());
}
```

ผลลัพธ์:

```
ข้อมูลใน struct
```

#### สรุปกฎ 3 ข้อ เป็นตารางอ้างอิงด่วน

| กฎ | เงื่อนไข | ผลลัพธ์ที่ compiler ทำให้อัตโนมัติ |
|---|---|---|
| กฎข้อที่ 1 | มี reference parameter ที่ไม่ได้เขียน lifetime | แต่ละตัวได้ lifetime parameter ของตัวเอง (ไม่ซ้ำกัน) |
| กฎข้อที่ 2 | หลังใช้กฎข้อ 1 แล้ว เหลือ input lifetime พอดี**หนึ่งตัว** | ใช้ lifetime ตัวนั้นกับ output lifetime ที่ elided ทั้งหมด |
| กฎข้อที่ 3 | ฟังก์ชันเป็น method ที่มี `&self`/`&mut self` | ใช้ lifetime ของ `self` กับ output lifetime ที่ elided ทั้งหมด (ไม่สนใจว่ามี reference parameter อื่นกี่ตัว) |

**ถ้าตรวจกฎทั้ง 3 ข้อครบแล้ว ยังมี output lifetime ที่ elided เหลืออยู่โดยไม่รู้ว่าจะผูกกับอะไร — compiler จะปฏิเสธและ
ให้เราเขียน `'a` เอง** นี่คือสิ่งที่เกิดกับ `longest` (มี input lifetime สองตัว ไม่เข้ากฎข้อ 2, ไม่ใช่ method จึงไม่เข้า
กฎข้อ 3) และเป็นสิ่งที่เกิดกับ `dangle()` จาก Part 7 (ไม่มี input lifetime ให้ใช้เลย ไม่เข้ากฎข้อ 1 ด้วยซ้ำเพราะไม่มี
reference parameter ให้กฎข้อ 1 ทำงาน)

**ข้อคิดสำคัญที่ควรจำไว้ตลอดไป**: กฎ elision ทั้ง 3 ข้อนี้**ไม่ใช่กฎที่ทำให้โปรแกรม "ปลอดภัยน้อยลง"** เมื่อเทียบกับ
การเขียน `'a` เอง — มันเป็นแค่ **shortcut ทางไวยากรณ์ (syntax sugar)** ที่ compiler แปลงกลับเป็นรูปแบบเต็มก่อนตรวจสอบ
เสมอ (เหมือน field init shorthand ที่เราเห็นใน Part 9) การตรวจสอบความปลอดภัยเข้มงวดเท่ากันทุกกรณี ไม่ว่าจะเขียน `'a`
เองหรือปล่อยให้ elision ทำงานให้ก็ตาม

### 20.6 Lifetime บน Struct Definition: เฉลยเต็มรูปแบบของปริศนา Part 9

ตอนนี้เรามีความรู้พอที่จะกลับไปตอบคำถามที่ Part 9 หัวข้อ 9.5 ทิ้งไว้แบบเต็มรูปแบบ ทวนสถานการณ์เดิม: ถ้าเราพยายามให้
struct เก็บ `&str` เป็น field โดยไม่มี lifetime annotation:

```rust
struct Product {
    name: &str, // พยายามเก็บ reference แทนการเป็นเจ้าของ String
    price: f64,
    quantity: u32,
}

fn main() {
    let product = Product {
        name: "เมาส์ไร้สาย",
        price: 299.0,
        quantity: 3,
    };
    println!("{}", product.name);
}
```

โค้ดนี้ compile ไม่ผ่านด้วย error เดียวกับที่เห็นใน Part 9 เป๊ะ ๆ:

```
error[E0106]: missing lifetime specifier
 --> src/main.rs:2:11
  |
2 |     name: &str, // พยายามเก็บ reference แทนการเป็นเจ้าของ String
  |           ^ expected named lifetime parameter
  |
help: consider introducing a named lifetime parameter
  |
1 ~ struct Product<'a> {
2 ~     name: &'a str, // พยายามเก็บ reference แทนการเป็นเจ้าของ String
  |
```

#### ทำไม struct field ต้องมี lifetime: ใช้หลักการเดียวกับฟังก์ชันเป๊ะ ๆ

ตอนนี้เราสามารถอธิบาย error นี้ได้แบบเต็มรูปแบบแล้ว: **struct instance หนึ่งตัวอาจถูกส่งผ่านไปมา เก็บไว้ในตัวแปรนาน ๆ
หรือ return ออกจากฟังก์ชันได้ เหมือนกับที่ reference ที่คืนออกจากฟังก์ชันสามารถถูกใช้งานต่อนานแค่ไหนก็ได้** compiler
จำเป็นต้องมี**สัญญา**ที่บอกไว้ล่วงหน้าว่า "reference ที่ field นี้เก็บอยู่ มีความสัมพันธ์เรื่องอายุกับตัว struct
instance เองอย่างไร" — เหมือนกับที่ signature ของฟังก์ชันต้องสัญญาความสัมพันธ์ระหว่าง input lifetime กับ output
lifetime **struct definition ก็ต้องสัญญาความสัมพันธ์ระหว่าง lifetime ของ field ที่เป็น reference กับอายุของ struct
instance เองในลักษณะเดียวกันทุกประการ**

วิธีแก้คือเพิ่ม lifetime parameter `<'a>` ต่อท้ายชื่อ struct (สังเกตตำแหน่ง — เหมือนกับที่เขียนบนฟังก์ชันและตรงกับ
ตำแหน่งของ generic type parameter `<T>` จาก Part 18 เป๊ะ ๆ):

```rust
// struct Product<'a> คือ "struct ที่ generic เหนือ lifetime 'a"
// ความหมาย: Product instance หนึ่งตัวไม่สามารถมีอายุยืนกว่า reference ที่ field `name` เก็บอยู่ได้เลย
struct Product<'a> {
    name: &'a str,
    price: f64,
    quantity: u32,
}

impl<'a> Product<'a> {
    fn describe(&self) -> String {
        format!("{} ({:.2} บาท x {})", self.name, self.price, self.quantity)
    }
}

fn main() {
    let owned_name = String::from("เมาส์ไร้สาย");

    let product = Product {
        name: &owned_name, // ยืม owned_name มา ไม่ copy ข้อมูลจริง
        price: 299.0,
        quantity: 3,
    };

    println!("{}", product.describe());
    // product ใช้งานได้ตลอดไป ตราบใดที่ owned_name ยังไม่ถูก drop
}
```

ผลลัพธ์:

```
เมาส์ไร้สาย (299.00 บาท x 3)
```

**อ่าน `Product<'a>` ให้ถูกต้อง**: `'a` ในที่นี้ไม่ใช่ "อายุคงที่ตัวเดียวที่ตายตัว" แต่เป็น **placeholder** ที่จะถูก
แทนที่ด้วยอายุจริงของ reference ที่ส่งเข้ามาในแต่ละครั้งที่สร้าง instance — เหมือนกับที่ `T` ใน `Vec<T>` ถูกแทนที่ด้วย
`i32` ในบางจุดและ `String` ในอีกจุดหนึ่ง `'a` ใน `Product<'a>` ก็ถูกแทนที่ด้วยอายุที่ต่างกันไปในแต่ละครั้งที่สร้าง
`Product` instance เช่นกัน (ในตัวอย่างข้างบน มันถูกแทนที่ด้วยอายุของ `owned_name` ภายใน `main`)

#### พิสูจน์สัญญานี้ให้เห็นจริง: E0597 เมื่อ Struct มีอายุยืนกว่าข้อมูลต้นทาง

Part 9 บอกไว้ว่า "reference ที่ field เก็บอยู่ ต้องมีอายุยืนไม่น้อยกว่า struct instance เองเสมอ" — มาพิสูจน์ประโยคนี้
ให้เห็นจริงด้วยตัวอย่างที่จงใจละเมิดสัญญานี้ (คล้ายกับตัวอย่าง E0597 ของ `longest` ในหัวข้อ 20.4 แต่คราวนี้เป็น
struct):

```rust
struct Excerpt<'a> {
    part: &'a str,
}

impl<'a> Excerpt<'a> {
    fn new(full_text: &'a str) -> Excerpt<'a> {
        let first_sentence = full_text
            .split('.')
            .next()
            .expect("ไม่มีจุด (.) ในข้อความนี้เลย");
        Excerpt { part: first_sentence }
    }
}

fn main() {
    let excerpt;
    {
        let novel = String::from("Call me Ishmael. Some years ago never mind.");
        excerpt = Excerpt::new(&novel); // excerpt.part ยืมมาจาก novel
    } // novel หมดอายุตรงนี้ — ถูก drop
    println!("{}", excerpt.part); // ❌ excerpt ยังพยายามใช้ reference ที่ชี้ไปยัง novel ที่ตายไปแล้ว
}
```

```
error[E0597]: `novel` does not live long enough
  --> src/main.rs:19:32
   |
18 |         let novel = String::from("Call me Ishmael. Some years ago never mind.");
   |             ----- binding `novel` declared here
19 |         excerpt = Excerpt::new(&novel); // excerpt.part ยืมมาจาก novel
   |                                ^^^^^^ borrowed value does not live long enough
20 |     } // novel หมดอายุตรงนี้ — ถูก drop
   |     - `novel` dropped here while still borrowed
21 |     println!("{}", excerpt.part); // ❌ excerpt ยังพยายามใช้ reference ที่ชี้ไปยัง novel ที่ตายไปแล้ว
   |                    ------------ borrow later used here
```

**สังเกตว่า error นี้มีรูปแบบเดียวกันเป๊ะ ๆ กับ E0597 ของ `longest` ในหัวข้อ 20.4** (`does not live long enough`,
`dropped here while still borrowed`, `borrow later used here`) — เพราะมันคือการละเมิด**หลักการเดียวกัน**: `excerpt`
(struct instance) พยายามมีอายุยืนกว่า `novel` (ข้อมูลที่ field `part` ของมันอ้างอิงถึง) ซึ่งเป็นสิ่งที่ `Excerpt<'a>`
ประกาศไว้ตั้งแต่ต้นว่าห้ามเกิดขึ้น — `<'a>` บน struct คือสัญญาเดียวกันกับ `<'a>` บนฟังก์ชัน เพียงแต่ตรวจสอบกับ "อายุของ
instance" แทน "อายุของ return value ที่ใช้งานต่อ" เท่านั้น

**วิธีแก้**: จัดโครงสร้าง scope ให้ `novel` มีอายุยืนไม่น้อยกว่า `excerpt` เสมอ (ย้าย `let novel = ...;` ออกมาไว้นอก
block เดียวกับ `excerpt`) หรือถ้าไม่มีความจำเป็นต้อง borrow จริง ๆ ให้ `Excerpt` เก็บ `String` (owned) แทน `&str`
ตามคำแนะนำที่ Part 9 ให้ไว้ตอนที่ยังไม่ได้เรียน lifetime เต็มรูปแบบ

### 20.7 Struct ที่มี Lifetime Parameter หลายตัว: เมื่อไหร่ควรแยก `'a` กับ `'b`

หัวข้อ 20.6 แนะนำ struct ที่มี lifetime parameter **ตัวเดียว** (`Product<'a>`, `Excerpt<'a>`) แต่ในความเป็นจริง struct
สามารถมี lifetime parameter **ได้มากกว่าหนึ่งตัว** เช่นเดียวกับที่มันมี generic type parameter ได้มากกว่าหนึ่งตัว
(`Pair<A, B>` จาก Part 18) — คำถามสำคัญคือ **เมื่อไหร่ควรใช้ `'a` ตัวเดียวให้ทุก field ที่เป็น reference ใช้ร่วมกัน
และเมื่อไหร่ควรแยกเป็น `'a`, `'b` คนละตัว?** คำตอบคือหลักการเดียวกันกับกับดักข้อ 2 ที่จะเห็นท้ายบทนี้: **ใช้ `'a`
ร่วมกันเมื่อ field ทั้งสองมักมีที่มาและอายุที่สัมพันธ์กันจริง ๆ (เช่นยืมมาจากข้อมูลชุดเดียวกัน) และแยกเป็นคนละตัวเมื่อ
field แต่ละตัวมีที่มาที่เป็นอิสระจากกันจริง**

ลองดูตัวอย่างที่แสดงผลกระทบของการเลือกแบบผิดให้เห็นภาพชัดที่สุด — สมมติต้องการเก็บคู่ข้อความสองชิ้นไว้ด้วยกันใน struct
เดียว โดยแต่ละชิ้นอาจยืมมาจากตัวแปรต้นทางที่**คนละตัวกัน** และมีอายุไม่เท่ากัน:

```rust
// เวอร์ชันที่ผูก lifetime เดียวกันทั้งสอง field (เหมาะเมื่อ first/second มาจากแหล่งเดียวกันจริง ๆ)
struct PairSame<'a> {
    first: &'a str,
    second: &'a str,
}

fn main() {
    let s1 = String::from("อายุยืน");
    let pair;
    {
        let s2 = String::from("อายุสั้น");
        pair = PairSame { first: &s1, second: &s2 };
        let _ = &pair;
    } // s2 หมดอายุตรงนี้
    println!("{}", pair.first); // ❌ ล้มเหลว แม้เราจะใช้แค่ pair.first ที่มาจาก s1 (อายุยืน) เท่านั้นก็ตาม!
}
```

```
error[E0597]: `s2` does not live long enough
  --> src/main.rs:12:47
   |
11 |         let s2 = String::from("อายุสั้น");
   |             -- binding `s2` declared here
12 |         pair = PairSame { first: &s1, second: &s2 };
   |                                               ^^^ borrowed value does not live long enough
13 |         let _ = &pair;
14 |     } // s2 หมดอายุตรงนี้
   |     - `s2` dropped here while still borrowed
15 |     println!("{}", pair.first); // ❌ ล้มเหลว แม้เราจะใช้แค่ pair.first ที่มาจาก s1 (อายุยืน) เท่านั้นก็ตาม!
   |                    ---------- borrow later used here
```

**สังเกตจุดสำคัญที่สุดของตัวอย่างนี้**: เราพยายามใช้แค่ `pair.first` ที่มาจาก `s1` (อายุยืนตลอด `main`) เท่านั้น
ไม่ได้แตะ `pair.second` เลยหลังจากออกจาก block — แต่ก็ยัง compile ไม่ผ่าน! เหตุผลคือ `PairSame<'a>` ประกาศไว้ว่า
`first` และ `second` มี lifetime **`'a` เดียวกัน** ซึ่งหมายความว่า **`'a` ที่แท้จริงต้องเป็นอายุที่สั้นที่สุดร่วมกัน
ระหว่างอายุของ `s1` และ `s2`** (หลักการเดียวกับ `longest` ในหัวข้อ 20.4) — เมื่อ `s2` หมดอายุ `'a` ทั้งก้อนของ
`pair` (รวมถึง `pair.first` ที่ไม่เกี่ยวอะไรกับ `s2` เลย) ก็ถือว่าหมดอายุไปด้วย เพราะ compiler มองว่า `pair` ทั้ง
instance ผูกกับ `'a` เดียวเท่านั้น ไม่ได้แยกวิเคราะห์ทีละ field

#### วิธีแก้: แยก Lifetime Parameter เป็นสองตัว (`'a`, `'b`)

ถ้า `first` และ `second` มีที่มาที่เป็นอิสระจากกันจริง ๆ (ไม่มีเหตุผลทางความหมายที่ต้องผูกกันเลย) ให้ประกาศ lifetime
parameter **สองตัวแยกกัน**:

```rust
struct PairDiff<'a, 'b> {
    first: &'a str,
    second: &'b str,
}

impl<'a, 'b> PairDiff<'a, 'b> {
    fn new(first: &'a str, second: &'b str) -> PairDiff<'a, 'b> {
        PairDiff { first, second }
    }
}

fn main() {
    let s1 = String::from("อายุยืน");
    let first_ref;
    {
        let s2 = String::from("อายุสั้น");
        let pair = PairDiff::new(&s1, &s2);
        println!("ใช้ครบทั้งคู่ตอนที่ s2 ยังอยู่: {} / {}", pair.first, pair.second);
        first_ref = pair.first; // เก็บแค่ field first ไว้ใช้ต่อ — field นี้มี lifetime 'a ผูกกับ s1 เท่านั้น
    } // s2 หมดอายุตรงนี้ — ไม่กระทบ first_ref เลย เพราะมันไม่เคยผูกกับ 'b (อายุของ s2) แม้แต่นิดเดียว
    println!("ใช้ได้แม้ s2 หมดอายุแล้ว เพราะ first_ref ผูกกับ 'a เท่านั้น: {first_ref}");
}
```

ผลลัพธ์:

```
ใช้ครบทั้งคู่ตอนที่ s2 ยังอยู่: อายุยืน / อายุสั้น
ใช้ได้แม้ s2 หมดอายุแล้ว เพราะ first_ref ผูกกับ 'a เท่านั้น: อายุยืน
```

ตอนนี้ `pair.first` (มี type `&'a str` ผูกกับ `s1` เท่านั้น) และ `pair.second` (มี type `&'b str` ผูกกับ `s2` เท่านั้น)
เป็น**อิสระจากกันอย่างสมบูรณ์**ในสายตาของ borrow checker — การที่ `s2` หมดอายุไม่มีผลกระทบต่อ `first_ref` เลยแม้แต่
นิดเดียว เพราะมันไม่เคยถูกผูกไว้กับ `'b` ตั้งแต่ต้น

**หลักการเลือกที่ควรจำ**: การประกาศ struct ด้วย lifetime parameter ตัวเดียว (`Pair<'a>`) เมื่อ field ทั้งหมดควรจะ
เป็นอิสระจากกันจริง ๆ คือการ **over-constrain** โดยไม่จำเป็น (เหมือนกับดักข้อ 2 ของฟังก์ชันที่จะเห็นท้ายบท) —
ในทางกลับกัน ถ้า field ทั้งสองมาจากแหล่งเดียวกันเสมอในทางความหมาย (เช่น struct `Excerpt<'a>` ที่ `part` เป็นส่วนหนึ่ง
ของข้อความต้นฉบับเดียวกันเสมอ) การใช้ `'a` ตัวเดียวก็เป็นทางเลือกที่ถูกต้องและเรียบง่ายกว่า ไม่จำเป็นต้องแยกให้ซับซ้อน
เกินความจำเป็น — **จำนวน lifetime parameter ที่ struct ควรมีสะท้อนความสัมพันธ์เชิงความหมายที่มีอยู่จริงของข้อมูล
ไม่ใช่กฎตายตัวว่ายิ่งแยกมากยิ่งดี**

#### หมายเหตุเรื่องการตั้งชื่อ Lifetime Parameter

จนถึงตอนนี้เราใช้ชื่อ `'a`, `'b` เป็นหลัก ซึ่งเป็น **convention ที่นิยมที่สุด** ในโค้ด Rust (เทียบได้กับการใช้ชื่อ
`T`, `U` สำหรับ generic type parameter จาก Part 18) แต่ในความเป็นจริง **คุณตั้งชื่อ lifetime parameter เป็นคำอะไรก็ได้
ที่ขึ้นต้นด้วย apostrophe** ตราบใดที่ยังไม่ชนกับ `'static` ที่ถูกจองไว้แล้ว — ในโค้ด production จริงบางแห่งจะเห็นชื่อ
ที่สื่อความหมายมากกว่า `'a` เปล่า ๆ เมื่อมี lifetime parameter หลายตัวที่อาจสับสนกันได้ง่าย เช่น library ที่ชื่อ
**`serde`** (สำหรับแปลงข้อมูลเป็น/จาก JSON ที่จะเรียนใน Part 57-58) ใช้ชื่อ **`'de`** (ย่อจาก "deserialize") เป็น
convention เฉพาะของมันเองสำหรับ lifetime ที่ผูกกับข้อมูลที่กำลังถูก deserialize — ตัวอย่างสั้น ๆ ที่แสดงว่าชื่ออื่น
ก็ใช้ได้ปกติ ไม่ต่างจาก `'a` เลยในทางไวยากรณ์:

```rust
// ใช้ 'input แทน 'a — ไวยากรณ์และความหมายเหมือนกันทุกประการ แค่ชื่อสื่อความหมายชัดเจนขึ้นสำหรับผู้อ่านโค้ด
struct Excerpt<'input> {
    part: &'input str,
}

fn main() {
    let novel = String::from("Call me Ishmael.");
    let excerpt = Excerpt { part: &novel };
    println!("{}", excerpt.part);
}
```

ผลลัพธ์:

```
Call me Ishmael.
```

**คำแนะนำเชิงปฏิบัติ**: ใช้ `'a`, `'b`, `'c` ตามลำดับสำหรับกรณีทั่วไปที่มี lifetime parameter ไม่เกิน 2-3 ตัว (เป็น
convention ที่คนอ่านโค้ด Rust ทุกคนคุ้นเคยอยู่แล้ว) แต่ถ้า struct/ฟังก์ชันของคุณมี lifetime parameter หลายตัวมากจน
สับสนได้ง่าย หรือกำลังเขียน library ที่คนอื่นจะใช้งาน parameter เหล่านี้ต้องปรากฏใน error message บ่อย ๆ (แบบเดียวกับ
`'de` ของ serde) การตั้งชื่อที่สื่อความหมายอาจช่วยให้อ่านง่ายขึ้นได้จริง

### 20.8 Lifetime ใน `impl` Block: ผสมกับกฎ Elision ข้อที่ 3

เมื่อ struct มี lifetime parameter (`Product<'a>`) การเขียน `impl` block ให้มันต้องประกาศ `'a` ที่ตัว `impl` เองด้วย
เสมอ — ไวยากรณ์คือ `impl<'a> Product<'a> { ... }` สังเกตว่า `'a` ปรากฏสองครั้ง: ครั้งแรกหลัง `impl` (ประกาศว่า block
นี้ทำงานกับ generic lifetime parameter ชื่อ `'a`) และครั้งที่สองต่อจากชื่อ struct (บอกว่ากำลัง implement ให้กับ
`Product` เวอร์ชันที่ใช้ `'a` นั้น) — รูปแบบนี้เหมือนกับ `impl<T> Container<T>` ที่จะเห็นเต็มรูปแบบเมื่อ generic
ผสมกับ struct ใน Part 22 เป๊ะ ๆ

ทีนี้มาดูว่ากฎ elision ข้อที่ 3 (เรื่อง `&self`) ทำงานร่วมกับ struct ที่มี lifetime อย่างไรในทางปฏิบัติจริง:

```rust
struct Excerpt<'a> {
    part: &'a str,
}

impl<'a> Excerpt<'a> {
    fn new(full_text: &'a str) -> Excerpt<'a> {
        let first_sentence = full_text
            .split('.')
            .next()
            .expect("ไม่มีจุด (.) ในข้อความนี้เลย");
        Excerpt { part: first_sentence }
    }

    // rule 1: &self ได้ lifetime ของตัวเอง (เรียกว่า 'b ในการวิเคราะห์ภายใน เพื่อไม่ให้สับสนกับ 'a ของ struct)
    // rule 3: มี &self -> lifetime ของ self ('b) ถูก assign ให้ output ที่ elided
    // เทียบเท่ากับเขียนเต็ม ๆ ว่า: fn part<'b>(&'b self) -> &'b str
    // สังเกตว่า output ผูกกับอายุของ "การยืม &self ครั้งนี้" ไม่ใช่ผูกกับ 'a ของ struct ตรง ๆ
    fn part(&self) -> &str {
        self.part
    }

    // ผสม rule 3 กับพารามิเตอร์อื่นที่ไม่ใช่ &self — rule 3 ยังใช้ lifetime ของ self เสมอ ไม่สนใจพารามิเตอร์อื่น
    fn announce_and_return(&self, announcement: &str) -> &str {
        println!("ประกาศ: {announcement}");
        self.part
    }
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago never mind.");
    let excerpt = Excerpt::new(&novel);

    println!("ส่วนที่ตัดมา: {}", excerpt.part());
    println!("{}", excerpt.announce_and_return("มีข้อความใหม่"));
}
```

ผลลัพธ์:

```
ส่วนที่ตัดมา: Call me Ishmael
ประกาศ: มีข้อความใหม่
Call me Ishmael
```

**จุดที่ต้องเข้าใจให้ลึกที่สุดในตัวอย่างนี้**: method `part(&self) -> &str` และ `announce_and_return(&self,
announcement: &str) -> &str` **ทั้งสองไม่ต้องเขียน `'a` เองเลยแม้แต่ตัวเดียว** ทั้ง ๆ ที่คืน `&str` ที่มาจาก
`self.part` (ซึ่งภายใต้ผิวคือ `&'a str` ตาม struct definition) — เหตุผลคือกฎข้อที่ 3 ทำงานโดยผูก output กับ
**"อายุของการยืม `&self` ในการเรียกครั้งนั้น ๆ"** ซึ่งในทางปฏิบัติจะสั้นกว่าหรือเท่ากับ `'a` ของ struct เสมอ (เพราะ
คุณไม่สามารถยืม `&self` ได้นานกว่าอายุของ struct instance เองอยู่แล้ว) และ `self.part` ที่มี lifetime `'a` (ยืนยาว
กว่าหรือเท่ากับอายุของ `&self` เสมอ) ก็ยัง "ยืมพอ" ที่จะคืนออกมาด้วยอายุที่สั้นกว่านั้นได้อย่างปลอดภัย — นี่คือเหตุผล
ที่ elision ยังทำงานได้ถูกต้องแม้ในสถานการณ์ที่มี lifetime สองตัวเกี่ยวข้อง (`'a` ของ struct และ lifetime ของ `&self`
ในแต่ละการเรียก) เพราะกฎข้อที่ 3 เลือกใช้ตัวที่ **"เข้มงวดกว่าหรือเท่ากันเสมอ"** ซึ่งพิสูจน์แล้วว่าปลอดภัยในทุกกรณี

### 20.9 `'static` Lifetime: ความหมายจริง, การใช้ที่ถูกต้อง, และ Anti-Pattern ที่ต้องเลี่ยง

มี lifetime พิเศษหนึ่งตัวที่มีชื่อจองไว้แล้วในภาษา Rust เขียนว่า **`'static`** — มันแปลว่า **"reference นี้ใช้งานได้
ปลอดภัยตลอดระยะเวลาที่โปรแกรมทำงานอยู่ (the entire duration of the program)"** เป็น lifetime ที่ "ยืนยาวที่สุดที่
เป็นไปได้" ในภาษา Rust ทั้งหมด

#### ทำไม String Literal ถึงมี `'static` โดยธรรมชาติ

ตัวอย่างที่พบบ่อยที่สุดของ `'static` คือ **string literal** (ข้อความในเครื่องหมายคำพูดที่เขียนตรง ๆ ในโค้ด เช่น
`"hello"`) — string literal ทุกตัวมี type เป็น `&'static str` โดยอัตโนมัติ:

```rust
fn main() {
    // string literal มี type &'static str โดยธรรมชาติ เพราะข้อมูลของมันถูก "ฝัง" (baked)
    // เข้าไปในตัวไฟล์ binary ที่ compile ออกมาแล้วโดยตรง ไม่ได้ถูกจัดสรรบน heap ตอน runtime เลย
    let s: &'static str = "I have a static lifetime.";
    println!("{s}");

    println!("{}", app_name());
}

// ค่าคงที่ระดับโปรแกรม (const) ที่เป็น &str ก็เป็น &'static str เสมอเช่นกัน ด้วยเหตุผลเดียวกัน
const APP_NAME: &'static str = "InventoryPro";

fn app_name() -> &'static str {
    APP_NAME
}
```

ผลลัพธ์:

```
I have a static lifetime.
InventoryPro
```

**เหตุผลเชิงลึกที่ string literal มี `'static`**: เมื่อ compiler แปลงโค้ด Rust เป็น machine code (binary) ข้อความ
`"I have a static lifetime."` จะถูก**เขียนลงไปในส่วนของไฟล์ binary ที่เรียกว่า read-only data section** (ส่วนหนึ่ง
ของไฟล์โปรแกรมที่ถูกโหลดเข้า memory พร้อมกับตัวโปรแกรมตอนเริ่มรัน และจะอยู่ใน memory ตลอดไปจนกว่าโปรแกรมจะปิดตัวลง)
มันไม่เคยถูก allocate บน heap ตอน runtime และไม่เคยถูก drop เลยตราบใดที่โปรแกรมยังรันอยู่ — ด้วยเหตุนี้ reference ไปยัง
มันจึงปลอดภัยที่จะใช้งานได้ **นานเท่าที่โปรแกรมยังทำงานอยู่** ซึ่งตรงกับความหมายของ `'static` เป๊ะ ๆ

#### `'static` เมื่อไหร่ที่เหมาะสมจริง ๆ

`'static` เหมาะกับข้อมูลที่เป็น **"ค่าคงที่ระดับโปรแกรม" อย่างแท้จริง** — ข้อมูลที่ไม่เคยเปลี่ยนแปลง ไม่ได้ถูกยืมมา
จากที่อื่นที่มีอายุจำกัด และควรมีชีวิตอยู่ตลอดโปรแกรม เช่น string literal ที่ใช้เป็นข้อความ error คงที่, ชื่อโปรแกรม,
เวอร์ชันที่ฝังมาจาก `env!("CARGO_PKG_VERSION")`, หรือค่าคงที่ทาง config ที่กำหนดไว้ตายตัวตั้งแต่ compile time

#### Anti-Pattern: ใช้ `'static` เป็นทางลัดแก้ Compiler Error โดยไม่เข้าใจต้นเหตุ

มือใหม่ที่เจอ error เรื่อง lifetime บ่อย ๆ (โดยเฉพาะจากหัวข้อ 20.6 ที่ struct เก็บ reference แล้วเจอ E0597) มักคว้า
`'static` มาใช้เป็น "ทางลัด" เพราะเห็นว่า **มันแก้ error ได้จริง** โดยไม่เข้าใจว่ากำลังทำอะไรอยู่ ลองดูตัวอย่าง
สถานการณ์นี้ให้เห็นภาพชัด:

```rust
struct Cache {
    label: &'static str, // ❌ เลือก &'static str เพราะเห็นว่ามันแก้ error เรื่อง lifetime ได้ (ผิดที่ต้นเหตุ)
}

fn make_cache(name: String) -> Cache {
    // เพื่อให้ String ที่ได้มาจาก parameter (อายุแค่ในฟังก์ชันนี้) กลายเป็น &'static str ได้
    // ต้องใช้ Box::leak — ฟังก์ชันนี้ "จงใจรั่ว" memory เพื่อยืดอายุของมันให้เท่ากับทั้งโปรแกรม
    let leaked: &'static str = Box::leak(name.into_boxed_str());
    Cache { label: leaked }
}

fn main() {
    let cache = make_cache(String::from("product-42"));
    println!("{}", cache.label);
}
```

ผลลัพธ์:

```
product-42
```

**โค้ดนี้ compile ผ่านและทำงานถูกต้องตามที่เห็น** — แต่นี่คือ **anti-pattern ที่ร้ายแรง** ด้วยเหตุผลหลายข้อ:

1. **`Box::leak`** ทำสิ่งที่ชื่อของมันบอกตรง ๆ: มันจงใจ "ปล่อยรั่ว" (leak) memory — เอา `Box<str>` (ที่ปกติจะถูก
   `drop` และคืน memory เมื่อหมด scope ตามกฎ ownership จาก Part 6) มา**บังคับให้ไม่มีวันถูก drop เลยตลอดการรันโปรแกรม**
   เพื่อให้มันมี lifetime เทียบเท่า `'static` ได้จริง ๆ (ไม่ใช่การ "โกหก" compiler แต่เป็นการยืดอายุข้อมูลจริง ๆ ด้วย
   วิธีที่ผลข้างเคียงรุนแรง)
2. ถ้าฟังก์ชัน `make_cache` ถูกเรียกซ้ำ ๆ หลายครั้ง (เช่นในระบบที่สร้าง cache หลายตัวตลอดการทำงานของโปรแกรม)
   **memory จะรั่วเพิ่มขึ้นเรื่อย ๆ ทุกครั้งที่เรียก และไม่มีวันถูกคืนกลับให้ระบบเลยจนกว่าโปรแกรมจะปิดตัวลงทั้งหมด**
   — นี่คือ memory leak ในความหมายที่แท้จริงตามที่เข้าใจกันในภาษาที่ไม่มี garbage collector
3. ปัญหาที่แท้จริงไม่ได้ถูก "แก้" เลย มันแค่ถูก **"ย้าย" จาก compile-time error ไปเป็น runtime memory leak** ซึ่ง
   แย่กว่าเดิมมาก เพราะ compile error ที่เห็นตั้งแต่แรกคือของขวัญที่บอกเราว่า "design ของ `Cache` ผิดตั้งแต่ต้น" —
   การไปหาทางทำให้ error หายไปโดยไม่แก้ design คือการเสีย "ของขวัญ" นั้นไปโดยเปล่าประโยชน์

**วิธีแก้ที่ถูกต้อง**: ถ้า `Cache` ไม่มีเหตุผลที่แท้จริงที่ต้องผูกกับข้อมูลระดับโปรแกรมจริง ๆ (เช่นในกรณีนี้ `name`
มาจาก parameter ที่รับเข้ามาแต่ละครั้ง ไม่ใช่ค่าคงที่ตายตัว) ให้เปลี่ยนไปเก็บ **owned type** (`String`) แทน:

```rust
struct Cache {
    label: String, // ✅ เก็บ owned String แทน — ไม่มีข้อจำกัดเรื่อง lifetime เลย ไม่ต้อง leak memory
}

fn make_cache(name: String) -> Cache {
    Cache { label: name } // ย้าย ownership ตรง ๆ ไม่ต้องยืมหรือ leak อะไรเลย
}

fn main() {
    let cache = make_cache(String::from("product-42"));
    println!("{}", cache.label);
} // cache หมด scope ที่นี่ — memory ถูกคืนกลับให้ระบบตามปกติ ไม่มี memory leak เลย
```

ผลลัพธ์:

```
product-42
```

โค้ดเวอร์ชันนี้ **ไม่มี memory leak, ไม่มี lifetime parameter ที่ซับซ้อน, และเก็บ ownership ของข้อมูลไว้อย่างถูกต้อง**
ตามหลักการที่เรียนมาตั้งแต่ Part 6-9 **บทเรียนสำคัญที่สุดจากหัวข้อนี้**: เมื่อเจอ compiler error เรื่อง lifetime
ให้ถามตัวเองก่อนเสมอว่า **"ข้อมูลนี้จำเป็นต้องเป็นระดับโปรแกรมจริง ๆ ไหม หรือแค่ต้องการเป็นเจ้าของมันเอง (owned)
มากกว่าที่จะยืม (borrowed)?"** — ในกรณีส่วนใหญ่ (ยิ่งตอนที่ยังไม่ชำนาญเรื่อง lifetime) คำตอบคือ **owned type แก้
ปัญหาได้ดีกว่า `'static` เสมอ** สงวน `'static` ไว้ใช้เฉพาะกับข้อมูลที่**เป็นค่าคงที่ระดับโปรแกรมจริง ๆ ตั้งแต่ต้น**
เท่านั้น (เช่น string literal, `const`) ไม่ใช่ใช้เป็นเครื่องมือ "บีบ" ข้อมูลที่มีอายุจำกัดให้กลายเป็นอายุยืนตลอดกาล

#### หมายเหตุแยกให้ชัด: `&'static str` (Reference) vs `T: 'static` (Lifetime Bound บน Generic)

มีอีกไวยากรณ์หนึ่งที่มือใหม่มักสับสนกับ `&'static str` คือ **`T: 'static`** ซึ่งเป็น **lifetime bound** ที่เขียนบน
generic type parameter (คล้ายกับ trait bound `T: Display` จาก Part 19 แต่ใช้ `'static` แทนชื่อ trait) — ความหมาย
ของทั้งสองต่างกันโดยสิ้นเชิงแม้จะมีคำว่า `'static` ร่วมกัน:

- **`&'static str`**: หมายถึง **reference ตัวนี้** ชี้ไปยังข้อมูลที่มีอายุยืนตลอดโปรแกรม (เช่น string literal)
- **`T: 'static`**: หมายถึง **type `T` เอง** (ไม่ว่าจะเป็น reference หรือไม่ก็ตาม) **ไม่มีการยืมข้อมูลที่มีอายุจำกัด
  มาจากที่ไหนเลย** — พูดง่าย ๆ คือ `T` เป็นได้ทั้ง owned value ธรรมดา (`String`, `i32`, ซึ่งไม่มี reference แฝงอยู่
  เลยจึงผ่านเงื่อนไขนี้โดยอัตโนมัติ) **หรือ** เป็น `&'static str` ก็ได้ (เพราะ `'static` ก็ถือว่า "ไม่มีข้อจำกัดเรื่อง
  อายุที่สั้นกว่าโปรแกรม" เช่นกัน) แต่ **ห้ามเป็น `&'a str` ที่ `'a` สั้นกว่าอายุทั้งโปรแกรม**

```rust
use std::fmt::Display;

// T: Display + 'static — ผสม trait bound (Part 19) กับ lifetime bound (บทนี้) เข้าด้วยกัน
fn print_it<T: Display + 'static>(input: T) {
    println!("{input}");
}

fn main() {
    // owned String ผ่านเงื่อนไข T: 'static ได้ เพราะไม่ได้ยืมข้อมูลมาจากที่ไหนเลย ไม่มี reference แฝงอยู่ในตัวมัน
    print_it(String::from("owned value ก็ผ่านเงื่อนไข T: 'static ได้"));

    // i32 (Copy type ธรรมดา) ก็ผ่านด้วยเหตุผลเดียวกัน
    print_it(42);

    // &'static str ก็ผ่าน เพราะ 'static คือ lifetime ที่ยืนยาวที่สุดอยู่แล้ว ไม่ขัดกับ bound เลย
    print_it("string literal ก็ผ่านเพราะเป็น &'static str อยู่แล้ว");
}
```

ผลลัพธ์:

```
owned value ก็ผ่านเงื่อนไข T: 'static ได้
42
string literal ก็ผ่านเพราะเป็น &'static str อยู่แล้ว
```

ถ้าลองส่ง `&i32` ที่ยืมมาจากตัวแปร local อายุสั้นเข้าไปใน `print_it` แทน (เช่น `print_it(&some_local_variable)`)
โค้ดจะ compile ไม่ผ่าน เพราะ `&'a i32` ที่มี `'a` อายุสั้นกว่าโปรแกรมไม่ผ่านเงื่อนไข `T: 'static` — นี่คือรายละเอียด
ที่ลึกกว่านี้อีกขั้นซึ่งเกี่ยวข้องกับการเก็บค่า generic ไว้ใช้งานข้าม thread หรือเก็บไว้ใน struct ที่ไม่รู้จักอายุของ
`T` ล่วงหน้า (เช่น `Box<dyn Trait + 'static>`) ซึ่งเป็นรายละเอียดขั้นสูงที่จะเจาะลึกเต็มรูปแบบใน **Part 23 (Lifetimes
ขั้นสูง)** — ตอนนี้จำแค่ว่า **`T: 'static` ตอบคำถาม "ชนิด T นี้มีข้อจำกัดเรื่องอายุที่ยืมมาจากใครหรือไม่" ไม่ใช่คำถาม
เดียวกับ `&'static str`** แม้จะเขียนคำว่า `'static` เหมือนกันก็ตาม

### 20.10 รวมทุกอย่างไว้ในที่เดียว: Generics + Trait Bounds + Lifetimes

หนึ่งในจุดที่ทำให้ Rust ทรงพลังมากคือ **generic type parameter (Part 18), trait bound (Part 19), และ lifetime
parameter (บทนี้) ทั้งสามระบบนี้ใช้ syntax เดียวกันในการประกาศ (ภายใน `<...>` หลังชื่อฟังก์ชัน) และทำงานประสานกันได้
อย่างเป็นธรรมชาติในลายเซ็นเดียวกัน** — มาดูตัวอย่างที่ผสมทั้งสามระบบเข้าด้วยกัน ต่อยอดจากฟังก์ชัน `longest`:

```rust
use std::fmt::Display;

// 'a คือ lifetime parameter (บทนี้)
// T คือ generic type parameter (Part 18) พร้อม trait bound T: Display (Part 19)
fn longest_with_label<'a, T: Display>(x: &'a str, y: &'a str, label: T) -> &'a str {
    println!("บันทึก: {label}");
    if x.len() > y.len() {
        x
    } else {
        y
    }
}

fn main() {
    let s1 = String::from("Hello world");
    let s2 = String::from("Rust");

    // เรียกด้วย label เป็น &str
    let result = longest_with_label(&s1, &s2, "เปรียบเทียบข้อความ");
    println!("ยาวกว่าคือ: {result}");

    // เรียกด้วย label เป็น i32 — เพราะ T ถูก bound ด้วย Display เท่านั้น ไม่ใช่ type ตายตัว
    let result2 = longest_with_label(&s1, &s2, 42);
    println!("ยาวกว่าคือ: {result2}");
}
```

ผลลัพธ์:

```
บันทึก: เปรียบเทียบข้อความ
ยาวกว่าคือ: Hello world
บันทึก: 42
ยาวกว่าคือ: Hello world
```

**สังเกตความแตกต่างเชิงหลักการที่สำคัญมาก**: `'a` และ `T` แม้จะเขียนอยู่ใน `<...>` เดียวกัน แต่ตอบคำถามคนละเรื่องกัน
โดยสิ้นเชิงและ**ไม่มีความสัมพันธ์กันเลย**:

- `'a` ตอบคำถามว่า **"reference ตัวนี้ (`x`, `y`, และ output) มีอายุใช้งานได้สัมพันธ์กันอย่างไร"** — ล้วนเป็นเรื่อง
  ของ memory safety ที่ borrow checker ตรวจสอบ
- `T: Display` ตอบคำถามว่า **"parameter `label` เป็น type อะไรก็ได้ ตราบใดที่ type นั้น implement trait `Display`
  (พิมพ์ออกมาด้วย `{}` ได้)"** — เป็นเรื่องของ type safety และ behavior ที่ trait bound ตรวจสอบ

ทั้งสองระบบทำงาน**คนละหน้าที่แต่เสริมกันได้อย่างสมบูรณ์**: `label` ไม่ได้มีข้อจำกัดเรื่อง lifetime เลยแม้แต่นิดเดียว
(เพราะมันไม่ใช่ reference — สามารถรับ owned value อย่าง `i32` ที่ implement `Copy` มาได้ตรง ๆ) ในขณะที่ `x`, `y` มี
ข้อจำกัดเรื่อง lifetime อย่างเข้มงวด แต่ไม่มีข้อจำกัดเรื่อง trait ใด ๆ เลย (เพราะเป็น `&str` ที่กำหนด type ตายตัวไว้
แล้ว ไม่ใช่ generic) — นี่คือตัวอย่างที่แสดงให้เห็นว่า **Rust ออกแบบระบบ type, trait, และ lifetime ให้เป็นแนวคิดที่
"orthogonal" (เป็นอิสระจากกัน แต่ผสมกันได้) อย่างสมบูรณ์** ทำให้คุณสามารถผสมทั้งสามอย่างในระดับความซับซ้อนเท่าไหร่ก็ได้
ตามที่ปัญหาจริงต้องการ โดยไม่ต้องกังวลว่าระบบหนึ่งจะ "ชน" กับอีกระบบหนึ่ง

### 20.11 ตัวอย่างจริงที่หนึ่ง: Log Parser ที่คืน Slice จากข้อความต้นฉบับโดยไม่ Copy ข้อมูล

มาดูตัวอย่างที่ใกล้เคียงงานจริงมากที่สุด ที่รวมทุกแนวคิดจาก Part 7 (borrowing), Part 8 (slice), และบทนี้ (lifetime)
เข้าด้วยกัน: ฟังก์ชัน**หาบรรทัดแรกในไฟล์ log ที่มีคำที่ต้องการ** แล้วคืน**ส่วนหนึ่งของข้อความต้นฉบับโดยตรง**
(ไม่สร้าง `String` ใหม่ ไม่ copy ข้อมูลเลย) — สถานการณ์นี้พบได้บ่อยมากในงานประมวลผลข้อความจริง เช่นระบบตรวจสอบ log
ของ server ที่มีข้อมูลขนาดใหญ่มาก การ copy ข้อมูลทุกครั้งที่ค้นหาจะเสีย performance โดยไม่จำเป็นเลย ทั้ง ๆ ที่แค่ต้อง
"ชี้ไปยังตำแหน่งที่พบ" ก็เพียงพอแล้ว:

```rust
// หาบรรทัดแรกใน log ที่มี pattern ที่ต้องการ แล้วคืน &str ที่ "ยืม" มาจาก log ต้นฉบับโดยตรง
//
// วิเคราะห์ lifetime: มี reference parameter สองตัว (log และ pattern) แต่ output ผูกกับ log เท่านั้น
// (เพราะบรรทัดที่คืนออกมามาจากการ slice ของ log ล้วน ๆ ไม่เคยมาจาก pattern เลย)
// ดังนั้นเราต้อง "แยก" lifetime ของ log ('a) ออกจาก lifetime ของ pattern (ปล่อยให้ elision จัดการเอง)
// เพื่อไม่ให้ caller ถูกบีบให้ pattern ต้องมีอายุยืนเท่า log โดยไม่จำเป็น (เหมือนกับดักข้อ 2 ท้ายบทนี้)
fn find_first_matching_line<'a>(log: &'a str, pattern: &str) -> Option<&'a str> {
    for line in log.lines() {
        if line.contains(pattern) {
            return Some(line);
        }
    }
    None
}

fn main() {
    let log_text = String::from(
        "2024-01-01 10:00:00 INFO server started\n\
         2024-01-01 10:00:05 WARN disk usage high\n\
         2024-01-01 10:00:10 ERROR connection refused\n\
         2024-01-01 10:00:15 ERROR database timeout\n",
    );

    // pattern เป็น string literal ('static) ที่มีอายุสั้นกว่า log_text ก็ยังใช้ได้ตามปกติ
    // เพราะ pattern ไม่ได้ถูกผูกเข้ากับ lifetime ของ output เลย
    match find_first_matching_line(&log_text, "ERROR") {
        Some(line) => println!("พบบรรทัด ERROR แรก: {line}"),
        None => println!("ไม่พบ ERROR เลย"),
    }

    match find_first_matching_line(&log_text, "FATAL") {
        Some(line) => println!("พบ: {line}"),
        None => println!("ไม่พบบรรทัด FATAL เลย"),
    }
}
```

ผลลัพธ์:

```
พบบรรทัด ERROR แรก: 2024-01-01 10:00:10 ERROR connection refused
ไม่พบบรรทัด FATAL เลย
```

**วิเคราะห์การออกแบบ signature นี้อย่างละเอียด** — นี่คือจุดที่แสดงให้เห็นว่าความเข้าใจเรื่อง lifetime ในระดับลึก
ช่วยให้ออกแบบ API ที่ **ยืดหยุ่นกว่า** สิ่งที่มือใหม่ (ที่แค่ทำตาม compiler error แบบผิวเผิน) มักจะเขียน:

- **ทำไมไม่ใช้ `fn find_first_matching_line<'a>(log: &'a str, pattern: &'a str) -> Option<&'a str>`** (ผูก
  lifetime เดียวกันทั้งคู่แบบ `longest`)? เพราะถ้าทำแบบนั้น **`pattern` จะถูกบีบให้ต้องมีอายุยืนเท่ากับ `log` และ
  output เสมอ** ทั้ง ๆ ที่ในความเป็นจริง `pattern` ถูกใช้แค่ตอน**เปรียบเทียบ**ภายในฟังก์ชันเท่านั้น (ผ่าน
  `.contains(pattern)`) ไม่เคยถูกนำไปประกอบเป็นส่วนหนึ่งของ output เลย — การผูก lifetime ที่ไม่จำเป็นแบบนี้จะสร้าง
  ข้อจำกัดปลอมให้กับผู้เรียกใช้ฟังก์ชัน (เหมือนกับดักข้อ 2 ในหัวข้อ "กับดักที่พบบ่อย" ท้ายบทนี้) เช่น ถ้า `pattern` มาจาก
  ตัวแปรที่มี scope สั้นกว่า `log` ผลลัพธ์ที่คืนออกมาก็จะถูกบีบให้มี scope สั้นตามไปด้วยอย่างไม่มีเหตุผล
- การเขียน `log: &'a str, pattern: &str` (ปล่อยให้ `pattern` ได้ lifetime ของตัวเองผ่านกฎ elision ข้อที่ 1 โดยไม่
  เกี่ยวกับ `'a`) **สื่อความหมายเชิง design ได้ตรงกว่ามาก**: มันบอกชัดเจนว่า **"ผลลัพธ์ที่คืนออกมาสัมพันธ์กับอายุของ
  `log` เท่านั้น ไม่เกี่ยวกับอายุของ `pattern` เลย"** — นี่คือทักษะสำคัญที่บทนี้ต้องการปลูกฝัง: **การเลือก lifetime
  parameter ไม่ใช่แค่ "ทำให้ compiler หยุดบ่น" แต่คือการออกแบบสัญญาของ API ให้ตรงกับความสัมพันธ์ที่มีอยู่จริงมากที่สุด**

### 20.12 ตัวอย่างจริงที่สอง: ตัวแยกวิเคราะห์ไฟล์ Config (`key = value`) ด้วย Struct ที่มี Lifetime

ตัวอย่างที่สองนี้แสดงให้เห็นการผสมทุกแนวคิดจากหัวข้อ 20.6-20.8 (struct with lifetime, elision) เข้ากับสถานการณ์ที่พบ
บ่อยมากในงานจริง: การ**แยกวิเคราะห์ (parse) ไฟล์ config รูปแบบ `key = value`** โดยให้ทั้ง key และ value เป็น slice
ที่ยืมมาจากบรรทัดต้นฉบับตรง ๆ ไม่สร้าง `String` ใหม่เลยแม้แต่ตัวเดียว — ต่างจากตัวอย่าง log parser ในหัวข้อ 20.11
ที่คืนแค่ `&str` ตัวเดียว ตัวอย่างนี้จะคืน **struct ที่มี 2 field เป็น reference พร้อมกัน** ซึ่งทั้งคู่มาจากแหล่งเดียวกัน
เสมอ (บรรทัดเดียวกัน) จึงเหมาะกับการใช้ `'a` **ตัวเดียว** ตามหลักการที่สรุปไว้ท้ายหัวข้อ 20.7:

```rust
struct KeyValue<'a> {
    key: &'a str,
    value: &'a str,
}

// input lifetime มีตัวเดียว (line) -> กฎ elision ข้อที่ 2 ผูก output ที่ elided ทั้งหมดเข้ากับมันโดยอัตโนมัติ
// เขียน '_' อย่างชัดเจนใน Option<KeyValue<'_>> เพื่อ "สื่อสาร" ว่ามี lifetime ที่ elide ไว้ตรงจุดนี้
// (สไตล์ที่ clippy และ Rust สมัยใหม่แนะนำ ยิ่งช่วยให้คนอ่านโค้ดรู้ทันทีว่า type นี้พ่วง lifetime มาด้วย)
fn parse_line(line: &str) -> Option<KeyValue<'_>> {
    let mut parts = line.splitn(2, '=');
    let key = parts.next()?.trim();
    let value = parts.next()?.trim();
    Some(KeyValue { key, value })
}

fn main() {
    let config_text = String::from(
        "host = 127.0.0.1\nport=8080\n# comment line without equals\ntimeout = 30\n",
    );

    for line in config_text.lines() {
        if line.trim().is_empty() || line.trim_start().starts_with('#') {
            continue; // ข้ามบรรทัดว่างและบรรทัด comment
        }
        match parse_line(line) {
            Some(kv) => println!("{} -> {}", kv.key, kv.value),
            None => println!("บรรทัดนี้ parse ไม่ได้: {line}"),
        }
    }
}
```

ผลลัพธ์:

```
host -> 127.0.0.1
port -> 8080
timeout -> 30
```

**วิเคราะห์การทำงานของ elision ในตัวอย่างนี้อย่างละเอียด**: ฟังก์ชัน `parse_line` มี reference parameter ตัวเดียว
(`line: &str`) — ตามกฎข้อที่ 1 มันได้ lifetime ของตัวเอง (เรียกว่า `'a`) ตามกฎข้อที่ 2 เพราะเหลือ input lifetime
พอดีหนึ่งตัว `'a` นั้นถูกใช้กับ output lifetime ที่ elided ทั้งหมด — แต่ output ของเราคือ `Option<KeyValue<'_>>` ซึ่ง
`'_` (อ่านว่า "anonymous lifetime" หรือ "lifetime นิรนาม") คือสัญลักษณ์พิเศษที่บอกว่า **"ให้ elision เดา lifetime
ตรงนี้ให้อัตโนมัติ (ตามกฎข้อ 2) แต่เขียนไว้ให้ชัดเจนว่ามี lifetime parameter อยู่ตรงนี้จริง ๆ นะ"** — ต่างจากการเขียน
`Option<KeyValue>` เฉย ๆ (ไม่มี `'_` เลย) ซึ่งในโค้ดสมัยใหม่ (edition 2018 ขึ้นไป) มักจะถูก clippy เตือนว่าให้เขียน
`'_` แทน เพื่อความชัดเจนในการอ่าน แม้ทั้งสองแบบจะ compile ผ่านและมีความหมายเดียวกันทุกประการก็ตาม

**เหตุผลที่ `KeyValue<'a>` เลือกใช้ lifetime parameter ตัวเดียวสำหรับทั้งสอง field**: เพราะ `key` และ `value`
**มาจากการ `split` บรรทัดเดียวกันเสมอ** — ทั้งสองมีที่มาและอายุที่สัมพันธ์กันอย่างแท้จริง (ทั้งคู่เป็น sub-slice ของ
`line` ตัวเดียวกัน) การใช้ `'a` ตัวเดียวจึงเป็นการออกแบบที่ **สื่อความหมายตรงกับความเป็นจริงมากที่สุด** และเรียบง่าย
กว่าการแยกเป็น `'a`, `'b` ที่ไม่จำเป็น (ต่างจากตัวอย่าง `PairDiff<'a, 'b>` ในหัวข้อ 20.7 ที่ field ทั้งสองมีที่มาที่
เป็นอิสระจากกันจริง) — นี่คือทักษะสำคัญที่สุดของทั้งบทนี้: **การเลือกจำนวนและการจัดกลุ่ม lifetime parameter ต้อง
สะท้อนความสัมพันธ์เชิงความหมายที่มีอยู่จริงของข้อมูล ไม่ใช่การท่องจำ pattern แบบตายตัว**

### 20.13 แนวทางตัดสินใจ: เมื่อไหร่ควรใช้ Owned Type และเมื่อไหร่ควรใช้ Lifetime Parameter

หลังจากเห็นตัวอย่างมามากพอ ควรมี **decision framework** สั้น ๆ ที่ใช้ได้จริงเวลาต้องออกแบบ struct หรือฟังก์ชันใหม่
ในโค้ดของตัวเอง คล้ายกับตารางตัดสินใจเรื่อง `T`/`&T`/`&mut T` ที่ Part 7 หัวข้อ 7.11 ให้ไว้ ต่อยอดมาเป็นเวอร์ชันที่
รวมเรื่อง lifetime เข้าไปด้วย:

| คำถามที่ต้องถามตัวเอง | ถ้าคำตอบคือ "ใช่" ให้เลือก |
|---|---|
| struct/ฟังก์ชันนี้ต้องส่งต่อ, เก็บไว้ใน collection, หรือคืนออกจากฟังก์ชันไปให้ context ที่ไม่รู้จักอายุของข้อมูลต้นทางแน่ชัด | **Owned type** (`String`, `Vec<T>`) — ปลอดภัยและยืดหยุ่นที่สุด ไม่มีข้อจำกัดเรื่อง lifetime เลย |
| มีการวัดผลจริง (benchmark) แล้วว่าการ copy ข้อมูล (`.clone()`/`.to_string()`) ส่งผลเสียต่อ performance อย่างมีนัยสำคัญ **และ** ข้อมูลต้นทางมีอายุยืนพอที่จะพิสูจน์ได้จริงด้วย borrow checker | **Reference พร้อม lifetime parameter** (`&'a str`, `struct Foo<'a>`) |
| field/reference สองตัวขึ้นไปมีที่มาจากแหล่งเดียวกันเสมอในทางความหมาย (เช่น slice จากบรรทัดเดียวกัน) | ใช้ **`'a` ตัวเดียวร่วมกัน** (เหมือน `KeyValue<'a>` และ `Excerpt<'a>`) |
| field/reference แต่ละตัวมีที่มาที่เป็นอิสระจากกันจริง ไม่มีความสัมพันธ์เชิงความหมายเลย | **แยก lifetime parameter คนละตัว** (เหมือน `PairDiff<'a, 'b>`) เพื่อไม่จำกัดสิทธิ์ผู้ใช้เกินจำเป็น |
| ข้อมูลเป็นค่าคงที่ระดับโปรแกรมจริง ๆ (string literal, `const`, ค่า config ที่ฝังตายตัวตั้งแต่ compile time) | **`&'static str`** — เหมาะสมและถูกต้องตามความหมายที่แท้จริง |
| ไม่แน่ใจ / เพิ่งเริ่มออกแบบ struct นี้ | **เริ่มจาก owned type ก่อนเสมอ** แล้วค่อยเปลี่ยนไปใช้ lifetime parameter ทีหลังเมื่อมีเหตุผลชัดเจนจริง ๆ เท่านั้น (ตามคำแนะนำเดิมจาก Part 9 หัวข้อ 9.5) |

แถวสุดท้ายคือหลักการที่สำคัญที่สุดในทางปฏิบัติ ไม่ต่างจากคำแนะนำเรื่อง `&T` vs `T` ใน Part 7: **lifetime parameter
บน struct คือเครื่องมือที่ทรงพลังแต่มี cost เชิง ergonomics (struct ที่มี `'a` ใช้งานยากขึ้น ผูกติดกับข้อมูลต้นทาง
มากขึ้น) — ควรใช้เมื่อมีเหตุผลชัดเจนจริง ๆ เท่านั้น ไม่ใช่ใช้เพราะ "เพิ่งเรียนมาแล้วอยากใช้ให้คุ้ม"** (ย้อนกลับไปดู
กับดักข้อ 6 ท้ายบทนี้ ที่เจาะจงพูดถึงความเข้าใจผิดนี้โดยตรง)

### 20.14 ทบทวนภาพรวม: Lifetime คือส่วนเสริมของ Borrow Checker ไม่ใช่ระบบที่แยกจากกัน

ก่อนจะไปหัวข้อกับดักและแบบฝึกหัด ขอสรุปภาพรวมทั้งหมดของบทนี้ให้เห็นเป็นภาพเดียวกัน: ตลอดบทนี้เราไม่ได้เรียนรู้ "กฎใหม่"
ของ borrow checker เลยแม้แต่ข้อเดียว — **กฎเหล็ก 2 ข้อจาก Part 7 ยังคงเหมือนเดิมทุกประการ** (mutable ได้ตัวเดียวหรือ
immutable ได้หลายตัว, ห้าม dangling reference) สิ่งที่เราเรียนรู้เพิ่มคือ **syntax และกลไก (`'a`, elision rules)
ที่ทำให้ compiler สามารถพิสูจน์ว่ากฎข้อที่ 2 (ห้าม dangling) จะไม่ถูกละเมิด แม้ในสถานการณ์ที่ reference ต้องข้าม
ขอบเขตของฟังก์ชันเดียว (ผ่าน parameter/return value) หรือถูกเก็บไว้ใน struct ที่มีอายุการใช้งานยาวกว่าหนึ่ง statement**

เปรียบเทียบง่าย ๆ: ถ้า Part 7 สอนว่า "reference ต้องไม่ dangling" เหมือนสอนกฎจราจรว่า "ห้ามรถชนกัน" บทนี้ก็เหมือนสอน
"ป้ายบอกทาง" ที่ทำให้รถแต่ละคัน (แต่ละ reference) รู้ล่วงหน้าว่าตัวเองควรวิ่งไปทางไหน นานแค่ไหน เพื่อไม่ให้ไปชนกับ
รถคันอื่นที่หยุดไปแล้ว — ป้ายบอกทางไม่ได้เปลี่ยนกฎจราจร มันแค่ทำให้ปฏิบัติตามกฎได้แม่นยำขึ้นในสถานการณ์ที่ซับซ้อนขึ้น

#### เปรียบเทียบกับภาษาอื่น: วิธีแก้ปัญหา "อายุของ Reference" ที่แตกต่างกันโดยสิ้นเชิง

เพื่อให้เห็นภาพว่าปัญหาที่ lifetime แก้ไม่ใช่ปัญหาที่ Rust "คิดขึ้นมาเอง" แต่เป็นปัญหาที่**ทุกภาษาที่มีแนวคิด
reference/pointer ต้องเจอ** เพียงแต่แต่ละภาษาเลือกแก้ด้วยวิธีที่ต่างกันโดยสิ้นเชิง:

| ภาษา | วิธีจัดการปัญหา "reference ชี้ไปยังข้อมูลที่ถูกทำลายไปแล้ว" | ตรวจพบเมื่อไหร่ | cost ตอน runtime |
|---|---|---|---|
| **C / C++** | ไม่มีการตรวจสอบอัตโนมัติเลย ผู้เขียนโค้ดต้องระวังเอง (คืน pointer ไปยัง local variable แบบที่เห็นใน Part 7 หัวข้อ 7.8 คือตัวอย่างคลาสสิก) | ไม่ตรวจเลย (จนกว่าจะ crash หรือได้ผลลัพธ์ผิดตอน runtime) | ไม่มี cost แต่แลกมาด้วยความเสี่ยง undefined behavior เต็ม ๆ |
| **Java / Python / JavaScript / Go** | ใช้ **garbage collector**: ข้อมูลจะไม่ถูกทำลายจริงตราบใดที่ยังมี reference เหลืออยู่ที่ไหนสักที่ในโปรแกรม (นับจำนวน reference หรือไล่ scan หา "ของที่เข้าถึงไม่ได้อีกแล้ว" เป็นระยะ) | ตรวจ/จัดการตอน **runtime** ตลอดเวลาที่โปรแกรมทำงาน | มี cost แน่นอน — ใช้ CPU และ memory เพิ่มสำหรับตัว garbage collector เอง และมักหยุดโปรแกรมชั่วครู่เป็นระยะ ๆ (GC pause) |
| **Swift / Objective-C** | ใช้ **reference counting** (ARC — Automatic Reference Counting): นับจำนวน reference ที่ชี้มา ถ้าเหลือ 0 ค่อยทำลาย | ตรวจ/จัดการตอน **runtime** เช่นกัน (แต่กระจายงานเป็นจุดเล็ก ๆ ไม่ต้อง pause ยาวเหมือน GC แบบ mark-and-sweep) | มี cost จากการเพิ่ม/ลดตัวนับทุกครั้งที่ reference ถูกสร้าง/ทำลาย |
| **Rust** | ใช้ **lifetime + borrow checker**: พิสูจน์ทางคณิตศาสตร์ตั้งแต่ตอน compile ว่า reference ทุกตัวจะไม่มีวัน dangling ได้เลย | ตรวจทั้งหมดตอน **compile time** เท่านั้น | **ศูนย์** — ไม่มีตัวนับ ไม่มี GC ไม่มี pause ใด ๆ เหลืออยู่ตอนโปรแกรมรันจริงเลยแม้แต่นิดเดียว |

จุดที่ทำให้ Rust อยู่ในตำแหน่งพิเศษมากคือ **มันได้ความปลอดภัยระดับเดียวกับ (หรือมากกว่า) ภาษาที่มี garbage
collector/reference counting โดยไม่ต้องเสีย cost ตอน runtime แบบภาษาเหล่านั้นเลย** และในเวลาเดียวกันก็ปลอดภัยกว่า
C/C++ อย่างมหาศาล (ที่ไม่ตรวจสอบอะไรเลย) — เหตุผลที่ทำได้แบบนี้ทั้งสองอย่างพร้อมกันคือ **การย้ายการตรวจสอบทั้งหมดไปไว้
ที่ compile time ผ่านระบบ ownership + borrowing + lifetime ที่เราเรียนมาตลอดทั้งโมดูล** นี่คือคำตอบแบบสมบูรณ์ที่สุด
ของคำถามที่ Part 1 ทิ้งไว้ตั้งแต่บทแรกของทั้งหลักสูตร

## กับดักที่พบบ่อย (Common Pitfalls)

กับดักเรื่อง lifetime ส่วนใหญ่ที่มือใหม่เจอไม่ได้มาจากความเข้าใจ concept ผิด แต่มาจาก**การไม่รู้ว่ากฎ 3 ข้อของ
elision (หัวข้อ 20.5) ทำงานอย่างไรในทุกรายละเอียด** — เมื่อ elision เดาผิดจากที่ตั้งใจไว้ (เช่นกับดักข้อ 3) หรือเมื่อ
เราเขียน annotation ที่ถูกต้องทางไวยากรณ์แต่ผิดทางความหมาย (เช่นกับดักข้อ 2) โค้ดก็จะ compile ไม่ผ่านหรือทำงานไม่ตรง
กับที่ตั้งใจ ต่อไปนี้คือ 6 กับดักที่พบบ่อยที่สุด เรียงจากกรณีพื้นฐานไปจนถึงกรณีที่ต้องเข้าใจกฎ elision อย่างละเอียด
ที่สุด:

**1. คิดว่า `E0106` แปลว่า "โค้ดผิดแน่ ๆ" ทั้ง ๆ ที่จริง ๆ แค่ต้องเพิ่ม annotation**

มือใหม่ที่เจอ `E0106: missing lifetime specifier` ครั้งแรกมักคิดว่ามี logic ผิดพลาดร้ายแรงบางอย่างในโค้ด ทั้ง ๆ ที่
ส่วนใหญ่แค่ต้องเพิ่ม lifetime annotation ให้ signature เท่านั้น (เหมือนที่เห็นในหัวข้อ 20.2):

```rust
fn shorter(x: &str, y: &str) -> &str {
    if x.len() < y.len() { x } else { y }
}

fn main() {
    let a = String::from("Rust");
    let b = String::from("Go");
    println!("{}", shorter(&a, &b));
}
```

```
error[E0106]: missing lifetime specifier
 --> src/main.rs:1:33
  |
1 | fn shorter(x: &str, y: &str) -> &str {
  |               ----     ----     ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but the signature does not say whether it is borrowed from `x` or `y`
help: consider introducing a named lifetime parameter
  |
1 | fn shorter<'a>(x: &'a str, y: &'a str) -> &'a str {
  |           ++++     ++          ++          ++
```

**วิธีแก้**: เติม `<'a>` และผูก `'a` เข้ากับ parameter ทั้งสองตัวและ return type ตามที่ compiler แนะนำ (`fn
shorter<'a>(x: &'a str, y: &'a str) -> &'a str`) — logic ข้างในฟังก์ชันไม่ต้องแก้อะไรเลยแม้แต่บรรทัดเดียว ปัญหาอยู่
ที่ signature ไม่ได้ประกาศความสัมพันธ์เรื่องอายุ ไม่ใช่ที่การคำนวณข้างใน

**2. ผูก lifetime เดียวกันให้ทุก parameter แบบเผื่อไว้ ทั้งที่ output ผูกกับตัวเดียวจริง ๆ (Over-constraining)**

กับดักนี้ตรงข้ามกับข้อ 1 — มือใหม่ที่เพิ่งเรียนเรื่อง lifetime มักแก้ปัญหาด้วยการใส่ `'a` เดียวกันให้กับ**ทุก**
reference parameter แบบ "เผื่อไว้ให้ปลอดภัยที่สุด" โดยไม่คิดว่า output จริง ๆ ผูกกับตัวไหนแค่ตัวเดียว:

```rust
// ผูก 'a เดียวกันให้ทั้ง x และ y ทั้ง ๆ ที่ output คืนแค่ x เท่านั้น ไม่เคยคืน y เลย
fn always_first<'a>(x: &'a str, y: &'a str) -> &'a str {
    let _ = y;
    x
}

fn main() {
    let s1 = String::from("คงอยู่ยาว");
    let result;
    {
        let s2 = String::from("อายุสั้น");
        // signature บีบให้ y (คือ s2) ต้องมีอายุยืนเท่า 'a เดียวกับ x และ output
        // ทั้ง ๆ ที่ค่าที่คืนมาไม่เคยมาจาก y เลยแม้แต่นิดเดียว
        result = always_first(&s1, &s2);
    }
    println!("{result}");
}
```

```
error[E0597]: `s2` does not live long enough
  --> src/main.rs:14:36
   |
11 |         let s2 = String::from("อายุสั้น");
   |             -- binding `s2` declared here
...
14 |         result = always_first(&s1, &s2);
   |                                    ^^^ borrowed value does not live long enough
15 |     }
   |     - `s2` dropped here while still borrowed
16 |     println!("{result}");
   |                ------ borrow later used here
```

**วิธีแก้**: ใช้ lifetime parameter **แยกกันคนละตัว** สำหรับ parameter ที่ไม่เกี่ยวข้องกับ output จริง ๆ — บอก
compiler ให้ตรงกับความสัมพันธ์ที่มีอยู่จริง ไม่ใช่ "เผื่อ" แบบตายตัว:

```rust
// แยก lifetime เป็น 'a (ผูกกับ x และ output) และ 'b (ผูกกับ y เท่านั้น) — สื่อความหมายตรงกับความจริง
fn always_first<'a, 'b>(x: &'a str, y: &'b str) -> &'a str {
    let _ = y;
    x
}

fn main() {
    let s1 = String::from("คงอยู่ยาว");
    let result;
    {
        let s2 = String::from("อายุสั้น");
        result = always_first(&s1, &s2); // ✅ compile ผ่าน เพราะ y ('b) ไม่ถูกผูกกับ output เลย
        println!("ใช้ result ได้ตอนที่ s2 ยังอยู่: {result}");
    }
    // ใช้ result ต่อได้แม้ s2 หมดอายุไปแล้ว เพราะ result ผูกกับ 'a (อายุของ s1) เท่านั้น
    println!("ใช้ result ได้แม้ s2 หมดอายุแล้ว: {result}");
}
```

**บทเรียนสำคัญ**: **การผูก lifetime สองตัวเข้าด้วยกันเป็น `'a` เดียวกันคือการ "จำกัดสิทธิ์" ของ caller** เหมือนกับ
ที่การรับ `T` ตรง ๆ (owned) จำกัดสิทธิ์มากกว่ารับ `&T` (Part 7 หัวข้อ 7.11) — **ให้ผูก lifetime ร่วมกันเฉพาะเมื่อ
output จริง ๆ อาจมาจากทั้งสองตัว (เหมือน `longest`) เท่านั้น** ถ้า output ผูกกับแค่ตัวเดียวจริง ๆ ให้แยก lifetime
parameter ออกจากกันเสมอ เพื่อให้ผู้เรียกใช้มีอิสระมากที่สุดเท่าที่ความสัมพันธ์จริงจะอนุญาต

**3. ลืมว่า Elision Rule 3 ผูก Output กับ `&self` เสมอ แม้มี Reference Parameter อื่นที่ตั้งใจจะคืนจริง ๆ**

กฎข้อที่ 3 (หัวข้อ 20.5) ทำงาน "แบบไม่มีข้อยกเว้น": ถ้า method มี `&self` lifetime ของ `self` จะถูกผูกกับ output
ที่ elided **เสมอ ไม่สนใจว่ามี parameter อื่นที่เป็น reference ด้วยกี่ตัว** มือใหม่มักเข้าใจผิดว่า compiler จะ "เดา
เอง" ว่าตั้งใจคืนจาก parameter ตัวไหน:

```rust
struct Greeter {
    greeting: String,
}

impl Greeter {
    // ตั้งใจจะคืน `name` กลับไปตรง ๆ แต่ elision rule 3 ผูก output กับ lifetime ของ &self ไปแล้ว (ไม่ใช่ของ name)
    fn greet(&self, name: &str) -> &str {
        println!("{}, {}!", self.greeting, name);
        name
    }
}

fn main() {
    let g = Greeter { greeting: String::from("สวัสดี") };
    let result;
    {
        let visitor_name = String::from("สมชาย");
        result = g.greet(&visitor_name);
    }
    println!("{result}");
}
```

```
error: lifetime may not live long enough
 --> src/main.rs:9:9
  |
7 |     fn greet(&self, name: &str) -> &str {
  |              -            - let's call the lifetime of this reference `'1`
  |              |
  |              let's call the lifetime of this reference `'2`
8 |         println!("{}, {}!", self.greeting, name);
9 |         name
  |         ^^^^ method was supposed to return data with lifetime `'2` but it is returning data with lifetime `'1`
  |
help: consider introducing a named lifetime parameter and update trait if needed
  |
7 |     fn greet<'a>(&self, name: &'a str) -> &'a str {
  |             ++++               ++          ++
```

สังเกตข้อความ error ที่บอกตรงจุดมาก: **"method was supposed to return data with lifetime `'2`"** (lifetime ของ
`name`) **"but it is returning data with lifetime `'1`"** (lifetime ของ `&self` ที่ elision ผูกไว้ให้ output แทน)
— compiler สับสนแทนเราไม่ได้ มันแค่ทำตามกฎข้อที่ 3 อย่างเคร่งครัด ผลคือ signature ที่ elision สร้างให้ **ไม่ตรงกับ
เจตนาจริงของโค้ด** เลย

**วิธีแก้**: เขียน lifetime parameter ของ `name` เองอย่างชัดเจน แยกจาก `&self` ตามที่ compiler แนะนำ:

```rust
struct Greeter {
    greeting: String,
}

impl Greeter {
    // ระบุ 'a ของ name อย่างชัดเจน แยกจาก lifetime ของ &self (ที่ elide ไว้แบบไม่มีชื่อ)
    fn greet<'a>(&self, name: &'a str) -> &'a str {
        println!("{}, {}!", self.greeting, name);
        name
    }
}

fn main() {
    let g = Greeter { greeting: String::from("สวัสดี") };
    let result;
    {
        let visitor_name = String::from("สมชาย");
        result = g.greet(&visitor_name);
        println!("{result}");
    }
}
```

**บทเรียนสำคัญ**: กฎ elision ทำงาน "ตามกฎตายตัว" ไม่ได้ "เดาเจตนา" ของผู้เขียนโค้ด ทุกครั้งที่ method มี `&self`
**และ**มี reference parameter อื่นที่ตั้งใจจะเป็นที่มาของ output ให้เขียน lifetime parameter ของ parameter นั้น
อย่างชัดเจนเสมอ อย่าปล่อยให้ elision ทำงานแบบเดา เพราะกฎข้อที่ 3 จะเลือก `&self` ให้ก่อนเสมอโดยไม่มีข้อยกเว้น

**4. ลืมประกาศ `<'a>` ที่ `impl` block เมื่อ Struct มี Lifetime Parameter**

```rust
struct Excerpt<'a> {
    part: &'a str,
}

// ❌ ลืมประกาศ <'a> ที่ตัว impl block เอง
impl Excerpt {
    fn part(&self) -> &str {
        self.part
    }
}

fn main() {
    let text = String::from("ตัวอย่างข้อความ");
    let e = Excerpt { part: &text };
    println!("{}", e.part());
}
```

```
error[E0726]: implicit elided lifetime not allowed here
 --> src/main.rs:6:6
  |
6 | impl Excerpt {
  |      ^^^^^^^ expected lifetime parameter
  |
help: indicate the anonymous lifetime
  |
6 | impl Excerpt<'_> {
  |             ++++
```

**วิธีแก้ที่ถูกต้องที่สุด**: เขียน `impl<'a> Excerpt<'a> { ... }` ให้ครบทั้งสองตำแหน่ง (ไม่ใช่แค่ `impl
Excerpt<'_>` ตามที่ help แนะนำ ซึ่งเป็นทางออกที่ใช้ได้เฉพาะกรณีที่ method ข้างในไม่ได้อ้างถึง `'a` แบบเจาะจงอะไรเป็น
พิเศษเลย) — ต้องจำไว้ว่า **struct ที่มี generic parameter (ไม่ว่าจะเป็น type parameter `<T>` จาก Part 18 หรือ
lifetime parameter `<'a>` จากบทนี้) ต้องประกาศ generic parameter นั้นซ้ำอีกครั้งที่ `impl` block เสมอ** ไวยากรณ์
`impl<T> Container<T>` และ `impl<'a> Excerpt<'a>` ใช้หลักการเดียวกันเป๊ะ ๆ ตามที่เปรียบเทียบไว้ในหัวข้อ 20.8

**5. เข้าใจผิดว่า `'static` "แก้ทุกปัญหา" เรื่อง Lifetime ได้เสมอ**

ดังที่อธิบายละเอียดในหัวข้อ 20.9 — มือใหม่จำนวนมากพอเจอ error เรื่อง lifetime ที่ซับซ้อนจะลองใส่ `'static` แบบสุ่ม
ดูว่า error หายไปหรือไม่ โดยไม่เข้าใจว่ากำลังบังคับให้ compiler ต้องการข้อมูลที่มีอายุยืนตลอดโปรแกรมจริง ๆ ซึ่งใน
กรณีส่วนใหญ่**เป็นไปไม่ได้เลย**สำหรับข้อมูลที่มาจาก parameter หรือตัวแปร local:

```rust
fn wrap_greeting(name: &str) -> &'static str { // ❌ พยายามบอกว่า output มีอายุยืนตลอดโปรแกรม
    name
}

fn main() {
    let name = String::from("สมชาย");
    println!("{}", wrap_greeting(&name));
}
```

```
error: lifetime may not live long enough
 --> src/main.rs:2:5
  |
1 | fn wrap_greeting(name: &str) -> &'static str { // ❌ พยายามบอกว่า output มีอายุยืนตลอดโปรแกรม
  |                         - let's call the lifetime of this reference `'1`
2 |     name
  |     ^^^^ returning this value requires that `'1` must outlive `'static`
```

**วิธีแก้**: ในกรณีนี้ **ไม่มีทางแก้ด้วย `'static` ได้เลย** เพราะ `name` มาจาก parameter ที่มีอายุจำกัดเสมอ (ไม่มี
ทางรู้ล่วงหน้าว่า caller จะส่งอะไรมา อาจเป็นตัวแปร local อายุสั้นก็ได้) ทางแก้ที่ถูกต้องคือกลับไปใช้ elision ธรรมดา
(`fn wrap_greeting(name: &str) -> &str` ตามกฎข้อที่ 2) หรือคืน owned `String` ถ้าจำเป็นต้องมีอายุยืนกว่า `name`
จริง ๆ — จำหลักจากหัวข้อ 20.9 ไว้เสมอ: **`'static` ใช้ได้กับข้อมูลที่เป็นค่าคงที่ระดับโปรแกรมจริง ๆ เท่านั้น
ไม่ใช่ "ปุ่มแก้ปัญหาทุกอย่าง" ที่ใส่แล้วต้องหวังว่าจะผ่าน**

**6. คิดว่า Struct ที่มี Lifetime Parameter "แก้ปัญหาถาวร" ในสถานการณ์ที่ควรใช้ Owned Type**

กับดักเชิง design (ไม่ใช่ compile error โดยตรง แต่เป็นแนวคิดที่ผิด): มือใหม่ที่เพิ่งเรียนจบบทนี้บางคนจะรู้สึกว่า
"เรียน lifetime มาแล้ว ต้องใช้ให้คุ้ม" แล้วพยายามใช้ `struct Foo<'a> { data: &'a str }` ในทุกสถานการณ์ ทั้ง ๆ ที่
Part 9 แนะนำไว้ชัดเจนว่า **owned type (`String`) ควรเป็นตัวเลือกแรกเสมอถ้าไม่แน่ใจ**:

```rust
// struct ที่ผูกกับ lifetime โดยไม่มีเหตุผลด้าน performance ที่ชัดเจนจริง ๆ
struct UserProfile<'a> {
    display_name: &'a str,
    bio: &'a str,
}

// ปัญหาเชิง ergonomics: struct นี้ "ผูกติด" อยู่กับข้อมูลต้นทางตลอดไป ทำให้เก็บใส่ Vec<UserProfile>
// แล้วส่งกลับข้ามฟังก์ชัน หรือใส่ใน struct อื่นที่ไม่มี lifetime parameter เลย ทำได้ยากขึ้นมาก
// (ตัวอย่างสถานการณ์เชิง design เท่านั้น ไม่ compile เพราะไม่มีข้อมูลจริงมาสร้าง)
```

**วิธีแก้**: ก่อนใช้ lifetime parameter บน struct ให้ถามตัวเองเสมอว่า **"struct นี้จำเป็นต้อง `borrow` ข้อมูลจริง ๆ
เพื่อประหยัด cost การ allocate ที่วัดผลได้จริงหรือไม่ หรือแค่รู้สึกว่าน่าจะ 'เท่กว่า' ถ้าใช้ lifetime?"** ในโค้ด
ระดับ production จริงจำนวนมาก struct ที่เก็บ `String`/`Vec<T>` (owned) ยังคงเป็นตัวเลือกที่ถูกต้องและใช้งานได้จริง
มากกว่า struct ที่มี lifetime parameter ในสถานการณ์ทั่วไป — สงวนการใช้ lifetime parameter บน struct ไว้สำหรับกรณีที่
มีเหตุผลชัดเจนจริง ๆ เช่นฟังก์ชัน parser ที่ต้อง slice ข้อความขนาดใหญ่มากซ้ำ ๆ (แบบตัวอย่าง log parser ในหัวข้อ
20.10) ที่ cost ของการ copy ข้อมูลมีผลจริงต่อ performance ที่วัดได้

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** โค้ดต่อไปนี้ compile ไม่ผ่านด้วย `E0106` หาสาเหตุแล้วแก้ไข signature ของฟังก์ชัน `shortest` ให้ compile
   ผ่าน (ห้ามเปลี่ยน logic ข้างในฟังก์ชันเลย):
   ```rust
   fn shortest(x: &str, y: &str) -> &str {
       if x.len() < y.len() {
           x
       } else {
           y
       }
   }

   fn main() {
       let a = String::from("Rust");
       let b = String::from("C");
       println!("{}", shortest(&a, &b));
   }
   ```
   (hint: `x`, `y`, และ output ล้วนต้องผูก lifetime เดียวกัน เพราะ output อาจมาจากตัวใดตัวหนึ่งก็ได้ ขึ้นกับผลของ
   `if x.len() < y.len()` ตอน runtime — ใช้ syntax แบบเดียวกับ `longest<'a>` จากหัวข้อ 20.3 เป๊ะ ๆ)

   เฉลยแบบย่อ:
   ```rust
   fn shortest<'a>(x: &'a str, y: &'a str) -> &'a str {
       if x.len() < y.len() { x } else { y }
   }
   ```

2. **[ง่าย-กลาง]** นิยาม struct `Highlight<'a>` ที่มี field เดียวคือ `text: &'a str` จากนั้นเขียน `impl<'a>
   Highlight<'a>` ที่มี associated function `fn new(source: &'a str, keyword: &str) -> Option<Highlight<'a>>`
   ซึ่งค้นหา `keyword` ใน `source` ด้วย `.find(keyword)` — ถ้าเจอ ให้คืน `Some(Highlight { text: &source[start..] })`
   (ส่วนของ `source` ตั้งแต่ตำแหน่งที่เจอ `keyword` ไปจนสุดข้อความ) ถ้าไม่เจอให้คืน `None` แล้วเขียน `main` ที่ทดสอบ
   ทั้งกรณีเจอและไม่เจอ
   (hint: สังเกตว่า `keyword` **ไม่ต้อง** มี lifetime เดียวกับ `'a` เพราะมันไม่เคยถูกใส่เข้าไปใน output เลย — ใช้
   หลักการเดียวกับกับดักข้อ 2 และตัวอย่าง log parser ในหัวข้อ 20.11 — `source.find(keyword)` คืน `Option<usize>`
   ซึ่งเป็น index ตำแหน่งที่พบ ไม่ใช่ reference จึงไม่มีปัญหาเรื่อง lifetime เลย)

   เฉลยแบบย่อ:
   ```rust
   struct Highlight<'a> {
       text: &'a str,
   }

   impl<'a> Highlight<'a> {
       fn new(source: &'a str, keyword: &str) -> Option<Highlight<'a>> {
           source.find(keyword).map(|start| Highlight { text: &source[start..] })
       }
   }

   fn main() {
       let source = String::from("กรุณาตรวจสอบ ERROR ในระบบด้วย");

       match Highlight::new(&source, "ERROR") {
           Some(h) => println!("เจอ: {}", h.text),
           None => println!("ไม่เจอคำที่ค้นหา"),
       }

       match Highlight::new(&source, "FATAL") {
           Some(h) => println!("เจอ: {}", h.text),
           None => println!("ไม่เจอคำที่ค้นหา"),
       }
   }
   ```

3. **[กลาง]** โค้ดต่อไปนี้มี error `E0597` ให้อธิบายว่าทำไมเกิด error นี้ (อ้างอิงถึง lifetime parameter `'a` ของ
   struct `Wrapper<'a>` ในคำอธิบาย) แล้วแก้ไขให้ compile ผ่านด้วย**การจัดโครงสร้าง scope ใหม่** (ไม่ใช่เปลี่ยน field
   ของ struct ไปเป็น owned type):
   ```rust
   struct Wrapper<'a> {
       value: &'a str,
   }

   fn main() {
       let wrapper;
       {
           let data = String::from("ข้อมูลชั่วคราว");
           wrapper = Wrapper { value: &data };
       }
       println!("{}", wrapper.value);
   }
   ```
   (hint: `wrapper` ถูกประกาศไว้นอก block แต่ `data` (ข้อมูลที่ `wrapper.value` ยืมมา) ถูกประกาศ**ใน**block —
   `wrapper` จึงมีโอกาสมีอายุยืนกว่า `data` ซึ่งเป็นการละเมิดสัญญาที่ `Wrapper<'a>` ประกาศไว้โดยตรง ย้าย `data` และ
   `println!` เข้ามาอยู่ใน scope เดียวกับที่ใช้งาน `wrapper` จริง)

   เฉลยแบบย่อ:
   ```rust
   struct Wrapper<'a> {
       value: &'a str,
   }

   fn main() {
       let data = String::from("ข้อมูลชั่วคราว");
       let wrapper = Wrapper { value: &data };
       println!("{}", wrapper.value); // ✅ ใช้ wrapper ก่อนที่ data จะหมดอายุ ไม่มี block ซ้อนที่ทำให้ scope ไม่ตรงกัน
   }
   ```

4. **[ยาก/ประยุกต์]** ขยายตัวอย่าง log parser จากหัวข้อ 20.11 ให้ครบวงจรมากขึ้น: เขียนฟังก์ชัน `fn
   count_matching_lines(log: &str, pattern: &str) -> usize` ที่นับจำนวนบรรทัดทั้งหมดที่มี `pattern` (ไม่ใช่แค่
   บรรทัดแรก) และฟังก์ชัน `fn all_matching_lines<'a>(log: &'a str, pattern: &str) -> Vec<&'a str>` ที่คืน **ทุก
   บรรทัด**ที่ตรงกับ `pattern` เป็น `Vec<&str>` (ยืมมาจาก `log` ทั้งหมด ไม่ copy เลยแม้แต่บรรทัดเดียว) จากนั้นเขียน
   `main` ที่ใช้ log ตัวอย่างจากหัวข้อ 20.11 ทดสอบทั้งสองฟังก์ชัน แล้วตอบคำถามสั้น ๆ (4-6 บรรทัด): **ทำไม
   `count_matching_lines` ไม่ต้องมี lifetime parameter เลยแม้แต่ตัวเดียว ทั้ง ๆ ที่รับ `&str` สองตัวเข้ามา
   ในขณะที่ `all_matching_lines` ต้องมี `'a` อย่างชัดเจน?**
   (hint: คำตอบเกี่ยวข้องกับ **output type** ของทั้งสองฟังก์ชันโดยตรง — `usize` ไม่ใช่ reference เลยแม้แต่นิดเดียว
   จึงไม่มี output lifetime ให้ผูกกับอะไรตั้งแต่ต้น ในขณะที่ `Vec<&str>` มี reference เป็นสมาชิกภายใน ต้องมี
   lifetime parameter บอกว่า reference เหล่านั้นยืมมาจากไหน — ทวนหลักการจากหัวข้อ 20.1 ว่า lifetime annotation
   จำเป็นเฉพาะเมื่อ reference ต้อง "ข้ามขอบเขต" ของฟังก์ชันเท่านั้น)

   โครงเริ่มต้น (ไม่ใช่เฉลยเต็ม แค่ช่วยตั้งต้น):
   ```rust
   fn count_matching_lines(log: &str, pattern: &str) -> usize {
       log.lines().filter(|line| line.contains(pattern)).count()
   }

   fn all_matching_lines<'a>(log: &'a str, pattern: &str) -> Vec<&'a str> {
       log.lines().filter(|line| line.contains(pattern)).collect()
   }

   fn main() {
       let log_text = String::from(
           "2024-01-01 10:00:00 INFO server started\n\
            2024-01-01 10:00:05 WARN disk usage high\n\
            2024-01-01 10:00:10 ERROR connection refused\n\
            2024-01-01 10:00:15 ERROR database timeout\n",
       );

       println!("จำนวนบรรทัด ERROR: {}", count_matching_lines(&log_text, "ERROR"));
       for line in all_matching_lines(&log_text, "ERROR") {
           println!("  - {line}");
       }
   }
   ```

## สรุป

บทนี้คือ**บทสุดท้ายของโมดูล 1: พื้นฐานภาษา Rust** และเป็นจุดที่ทุกแนวคิดเรื่อง memory safety ที่เรียนมาตลอด 14 บท
(Part 6-20) มาบรรจบกันเป็นภาพเดียวที่สมบูรณ์ เรามาทวนเส้นทางทั้งหมดที่เดินทางมาด้วยกัน:

- **Part 6 (Ownership)** สอนว่าทุกค่ามีเจ้าของเพียงหนึ่งเดียว และการ move/drop เกิดขึ้นเมื่อไหร่ — นี่คือ**รากฐาน**
  ที่ทุกอย่างในโมดูลนี้ยืนอยู่บน
- **Part 7 (Borrowing)** แก้ปัญหาที่ ownership เพียงอย่างเดียวทำไม่ได้ (การ "ยืม" ใช้โดยไม่ต้องโอนกรรมสิทธิ์) ผ่าน
  กฎเหล็ก 2 ข้อ และทิ้งปริศนา `E0106` ของฟังก์ชัน `dangle()` ไว้
- **Part 8 (Slices)** ให้เครื่องมือ `&str`/`&[T]` ที่ทำให้ reference ชี้ไปยัง "ส่วนหนึ่ง" ของข้อมูลได้ ซึ่งกลายเป็น
  ตัวละครหลักของตัวอย่าง lifetime เกือบทั้งหมดในบทนี้
- **Part 9 (Structs)** ขยายแนวคิด ownership/borrowing เข้าสู่ชนิดข้อมูลที่ซับซ้อนขึ้น และทิ้งปริศนา `E0106` อีกครั้ง
  ในบริบทของ struct ที่เก็บ `&str`
- **Part 10-17** (enums, pattern matching, `Option`/`Result`, collections, modules, packages) สร้างเครื่องมือ
  และคำศัพท์เพิ่มเติมที่ทำให้เราเขียนโปรแกรมที่ซับซ้อนขึ้นได้ โดยยังใช้กฎ ownership/borrowing เดียวกันตลอด
- **Part 18-19** (Generics, Traits) แนะนำแนวคิด "generic" ที่ทำให้ฟังก์ชันและ struct ทำงานกับหลาย type ได้ผ่าน
  `<T>` และ trait bound
- **Part 20 (บทนี้)** ปิดวงจรทั้งหมด: แสดงให้เห็นว่า **lifetime parameter (`'a`) คือ generic parameter อีกประเภท
  หนึ่ง** ที่ทำงานคู่กับ borrow checker เพื่อพิสูจน์ว่ากฎห้าม dangling reference จาก Part 7 จะไม่ถูกละเมิด แม้ใน
  สถานการณ์ที่ซับซ้อนที่สุด (reference ข้ามขอบเขตฟังก์ชัน, struct ที่เก็บ reference ไว้นาน ๆ) — และเราได้เฉลย
  ปริศนา `E0106` ทั้งสองข้อที่ Part 7 และ Part 9 ทิ้งไว้อย่างครบถ้วนสมบูรณ์

**นี่คือคำตอบที่แท้จริงของคำถามใหญ่ที่สุดของทั้งโมดูล**: Rust ทำให้โปรแกรมปลอดภัยจาก use-after-free, double-free,
dangling pointer, และ data race ได้ทั้งหมด **โดยไม่ต้องมี garbage collector และไม่มี runtime cost เลยแม้แต่นิดเดียว**
เพราะทุกอย่างถูกพิสูจน์เสร็จสิ้นไปแล้วตั้งแต่ตอน compile time ผ่านระบบสามชั้นที่ทำงานร่วมกัน: **ownership** (ใครเป็น
เจ้าของและเมื่อไหร่จะถูกทำลาย), **borrowing** (ใครมีสิทธิ์เข้าถึงชั่วคราวแบบไหน), และ **lifetime** (พิสูจน์ว่าการยืม
ทั้งหมดจะไม่มีวันชี้ไปยังข้อมูลที่ถูกทำลายไปแล้ว ไม่ว่า reference จะถูกส่งผ่านไปกี่ฟังก์ชัน หรือถูกเก็บไว้ใน struct
นานแค่ไหนก็ตาม) — เมื่อ `'a` ทั้งหมดถูกลบออกไปหลัง compile เสร็จ สิ่งที่เหลืออยู่ใน binary คือ pointer ธรรมดา ๆ ที่
ทำงานเร็วเท่า C ทุกประการ นี่คือความหมายที่สมบูรณ์ที่สุดของคำว่า **zero-cost abstraction** ที่หลักสูตรนี้พูดถึงมา
ตั้งแต่ Part 1

จาก **Part 21** เป็นต้นไป เราจะเข้าสู่ **โมดูล 2: ระดับกลาง (Intermediate)** ซึ่งจะกลับไปเจาะลึกทั้งสามระบบที่เพิ่ง
เรียนจบไป (generics, traits, lifetimes) ในระดับที่ซับซ้อนขึ้นอีกขั้น โดยเริ่มจาก **Part 21: Traits ขั้นสูง** ที่จะ
สอน default method implementation, **trait object** (`dyn Trait`) สำหรับเก็บหลาย type ที่ implement trait เดียวกัน
ไว้ในโครงสร้างข้อมูลเดียว, และหลักการ **dynamic dispatch vs static dispatch** ที่เกี่ยวข้องกับ performance โดยตรง
— ตามด้วย **Part 22: Generics ขั้นสูง** (trait bounds ที่ซับซ้อนขึ้น, `where` clause, `PhantomData`) และ **Part 23:
Lifetimes ขั้นสูง** ที่จะกลับมาเจาะลึกกรณีที่ struct มี lifetime parameter หลายตัวพร้อมกัน, **higher-ranked trait
bounds**, และกรณีขั้นสูงอื่น ๆ ที่บทนี้ยังไม่ได้ครอบคลุม ทุกอย่างที่เรียนในบทนี้จะเป็น**พื้นฐานที่ขาดไม่ได้**สำหรับ
สามบทถัดไป เพราะทั้งสามระบบจะถูกผสมเข้าด้วยกันในความซับซ้อนที่มากขึ้นเรื่อย ๆ ตลอดโมดูล 2

---

**Part ก่อนหน้า:** [Traits เบื้องต้น](part-019-traits-basics.md) | **Part ถัดไป:** [Traits ขั้นสูง](part-021-traits-advanced.md)
