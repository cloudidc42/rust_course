# Part 23: Lifetimes ขั้นสูง

> โมดูล: ระดับกลาง (Intermediate) | ระดับ: กลาง-สูง | เวลาโดยประมาณ: 230 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- เขียนฟังก์ชันที่มี **lifetime parameter หลายตัวที่ไม่สัมพันธ์กันจริง ๆ** (`fn foo<'a, 'b>`) ได้อย่างถูกต้อง และรู้จัก
  แยกแยะได้ทันทีว่าเมื่อไหร่ input สองตัวควรใช้ `'a` ร่วมกัน (แบบ `longest` ใน Part 20) กับเมื่อไหร่ควรแยกเป็นคนละตัว
  (เพราะ output ผูกกับแค่ตัวเดียวจริง ๆ)
- อธิบาย **lifetime subtyping** ในระดับที่ใช้งานได้จริง (ไม่ต้องรู้ทฤษฎี variance เต็มรูปแบบ): เข้าใจว่า reference ที่
  มีอายุยืนกว่าใช้แทนที่ reference ที่มีอายุสั้นกว่าได้เสมอ และเขียน **outlives bound** (`'a: 'b`) เพื่อบอก compiler
  ถึงความสัมพันธ์นี้อย่างชัดเจนเมื่อ elision และการอนุมานปกติไม่พอ
- ตัดสินใจได้อย่างมีเหตุผลว่า struct ที่มี reference หลายตัวควรมี lifetime parameter กี่ตัว — ไปให้ลึกกว่าหลักการพื้นฐาน
  ที่ Part 20 หัวข้อ 20.7 ให้ไว้ จนถึงระดับที่รู้ว่าเมื่อไหร่ต้องเขียน outlives bound บน struct/ฟังก์ชัน generic เพื่อ
  แปลง lifetime หนึ่งไปเป็นอีกแบบปลอดภัย
- เขียนและอ่าน `Box<dyn Trait + 'a>` ได้อย่างถูกต้อง อธิบายได้ว่าทำไม `dyn Trait` เปล่า ๆ (ไม่เขียน lifetime) จึงมี
  ค่า default เป็น `dyn Trait + 'static` ในบริบทส่วนใหญ่ และรู้ว่าต้อง override เป็น lifetime สั้นกว่าเมื่อไหร่
- อ่านและเข้าใจ syntax **Higher-Ranked Trait Bounds** (`for<'a> Fn(&'a str) -> ...`) ที่ปรากฏใน error message และ
  เอกสารของ `std` (เช่น iterator adapter) ได้ — รู้ว่าทำไมบางครั้ง lifetime ธรรมดาที่เราคุ้นเคยจาก Part 20 ใช้ไม่ได้
  กับ closure parameter ที่ต้องรับ reference ซึ่งอายุถูกกำหนดที่จุดเรียกใช้ ไม่ใช่ที่จุดประกาศ
- เขียน trait ที่มี method คืน reference ผูกกับ `&self` (ทวนกฎ elision ข้อที่ 3 ในบริบท trait) และ trait ที่มี
  lifetime parameter ของตัวเอง (`trait Parser<'a>`) ได้อย่างถูกต้อง พร้อมอธิบายความแตกต่างระหว่างสองรูปแบบนี้
- **ล้มความเข้าใจผิดที่พบบ่อยที่สุด**ให้หมดสิ้น: lifetime annotation ไม่มีผลต่อเวลาที่ข้อมูลถูก drop จริงตอน runtime
  เลยแม้แต่นิดเดียว มันเป็นแค่เครื่องมือพิสูจน์ทางคณิตศาสตร์ตอน compile time เท่านั้น — และรู้จักปัญหาคลาส
  **self-referential struct** ที่ lifetime ธรรมดาแก้ไม่ได้ พร้อมรู้ทิศทางกว้าง ๆ ว่าโลกจริงแก้ปัญหานี้อย่างไร

## ความรู้ที่ต้องมีมาก่อน

บทนี้คือ**ภาคต่อโดยตรง**ของ **Part 20 (Lifetimes เบื้องต้น)** — ไม่ใช่บทที่เริ่มต้นใหม่ ทุกหัวข้อในบทนี้สร้างต่อจาก
รากฐานที่ Part 20 วางไว้โดยตรง โดยเฉพาะอย่างยิ่ง:

- **Part 20 หัวข้อ 20.5 (กฎ Lifetime Elision ทั้ง 3 ข้อ)**: บทนี้จะ**ไม่สอนกฎทั้ง 3 ข้อซ้ำ**อีกครั้งแบบเต็มรูปแบบ
  (แค่ทวนสั้น ๆ ในหัวข้อ 23.1) แต่จะพา**ไปให้ลึกกว่า**ในสถานการณ์ที่กฎทั้ง 3 ข้อยังไม่ครอบคลุม เช่น trait object,
  higher-ranked bound, และ struct ที่มี lifetime parameter มากกว่าหนึ่งตัวที่มีความสัมพันธ์กันแบบซับซ้อน
- **Part 20 หัวข้อ 20.6-20.8 (Struct กับ Lifetime)**: บทนี้สมมติว่าคุณเขียน `struct Excerpt<'a> { part: &'a str }`
  และ `impl<'a> Excerpt<'a>` ได้แล้วโดยไม่ต้องอธิบายซ้ำ
- **Part 20 หัวข้อ 20.7 (Struct ที่มี Lifetime หลายตัว)**: บทนี้จะไปให้ลึกกว่าตัวอย่าง `PairDiff<'a, 'b>` จนถึงระดับ
  ที่ต้องเขียน **outlives bound** (`'a: 'b`) อย่างชัดเจน ซึ่งเป็นเรื่องที่ Part 20 ยังไม่ได้แตะเลย
- **Part 20 หัวข้อ 20.9 (`'static`)**: บทนี้จะอธิบายเพิ่มว่าทำไม trait object (`dyn Trait`) เปล่า ๆ ถึงมี default
  bound เป็น `'static` โดยปริยาย ซึ่งเป็นประเด็นที่ Part 20 ทิ้งท้ายไว้ให้มาต่อในบทนี้โดยตรง (ดูหัวข้อ 20.9
  หมายเหตุสุดท้ายที่บอกว่า "รายละเอียดขั้นสูงจะเจาะลึกเต็มรูปแบบใน Part 23")
- **Part 6 (Ownership เบื้องต้น)**: กฎ move/drop และ **trait `Drop`** — บทนี้ต้องใช้ `Drop` เป็นตัวเปรียบเทียบสำคัญ
  ในหัวข้อ 23.8 เพื่อพิสูจน์ว่า lifetime annotation ไม่ใช่กลไก runtime แบบ `Drop`
- **Part 8 (Slices)**: `&str` และการ slice ข้อมูล — ตัวอย่าง lexer ในหัวข้อ 23.10 ใช้เทคนิคการ slice แบบ index range
  (`&s[start..end]`) เต็มรูปแบบ
- **Part 19 (Traits เบื้องต้น)**: syntax การประกาศ trait และ `impl Trait for Type` — บทนี้ใช้เป็นฐานสำหรับหัวข้อ 23.7
  (lifetime ใน trait definition)
- **Part 21 (Traits ขั้นสูง)**: แนวคิด `dyn Trait` และ trait object — บทนี้ (หัวข้อ 23.5) สมมติว่าคุณรู้จัก `Box<dyn
  Trait>` มาแล้วในระดับพื้นฐาน และจะพาไปลึกกว่าในมุมของ lifetime โดยเฉพาะ
- **Part 22 (Generics ขั้นสูง)**: `where` clause และ `PhantomData` — บทนี้ใช้ `where` clause เขียน outlives bound
  (`where 'a: 'b`) และใช้ `PhantomData` ในตัวอย่างกับดักหัวข้อหนึ่ง

ถ้าคุณจำได้ว่า Part 20 ปิดท้ายไว้ว่า **"ตอนนี้จำแค่ว่า `T: 'static` ตอบคำถาม 'ชนิด T นี้มีข้อจำกัดเรื่องอายุที่ยืมมาจาก
ใครหรือไม่' ... ซึ่งเป็นรายละเอียดขั้นสูงที่จะเจาะลึกเต็มรูปแบบใน Part 23 (Lifetimes ขั้นสูง)"** — บทนี้คือคำตอบเต็ม
รูปแบบของประโยคนั้น รวมกับกรณีขั้นสูงอีกหลายกรณีที่มือใหม่ (และแม้แต่ผู้เขียน Rust ที่มีประสบการณ์) มักสับสน

## เนื้อหา

### 23.1 ทวนความจำสั้น ๆ: กฎ Elision 3 ข้อ และเส้นแบ่งที่บทนี้จะก้าวข้ามไป

ก่อนไปลึกกว่านี้ ขอทวนกฎ elision ทั้ง 3 ข้อจาก Part 20 หัวข้อ 20.5 แบบสั้นที่สุด (ถ้าจำไม่ได้ ให้กลับไปอ่านหัวข้อนั้น
เต็ม ๆ ก่อน เพราะบทนี้จะไม่อธิบายซ้ำ):

1. **กฎข้อที่ 1**: reference parameter ที่ไม่ได้เขียน lifetime แต่ละตัวได้ lifetime parameter ของตัวเอง
2. **กฎข้อที่ 2**: ถ้าเหลือ input lifetime พอดีหนึ่งตัว ใช้ตัวนั้นกับ output lifetime ที่ elided ทั้งหมด
3. **กฎข้อที่ 3**: ถ้ามี `&self`/`&mut self` ใช้ lifetime ของมันกับ output lifetime ที่ elided ทั้งหมด

กฎ 3 ข้อนี้ถูกออกแบบมาให้ครอบคลุม **รูปแบบ signature ที่พบบ่อยที่สุด** ในโค้ดจริงเท่านั้น — จุดสำคัญที่ต้องเข้าใจให้
ชัดตั้งแต่ต้นบทนี้คือ: **กฎ elision ไม่ได้ครอบคลุมทุกความสัมพันธ์เรื่อง lifetime ที่เป็นไปได้ในภาษา Rust** มันเป็นแค่
"ทางลัด" สำหรับกรณีที่ชัดเจนพอ ส่วนกรณีที่ซับซ้อนกว่านั้น (ซึ่งเกิดขึ้นบ่อยมากในโค้ดระดับกลาง-สูง) ยังคง**ต้องเขียน
lifetime annotation เต็มรูปแบบด้วยมือ** และบางกรณี (หัวข้อ 23.3-23.4) ยังต้องเขียน **ความสัมพันธ์ระหว่าง lifetime
parameter สองตัว** ด้วยซ้ำ ซึ่งเป็นสิ่งที่กฎ elision ไม่มีวันทำแทนให้ได้เลย เพราะมันเกินขอบเขตของกฎง่าย ๆ 3 ข้อนั้น

บทนี้จะพาไปสำรวจ **6 พื้นที่ที่กฎ elision และความรู้พื้นฐานจาก Part 20 ยังไม่ครอบคลุม**:

1. ฟังก์ชันที่มี lifetime parameter หลายตัวที่**ไม่สัมพันธ์กันเลย** (ไม่ใช่แค่ "แยกเพราะ field มาจากคนละแหล่ง" แบบที่
   Part 20 หัวข้อ 20.7 สอนไว้แล้ว แต่รวมถึงกรณีที่ต้องบอกความสัมพันธ์ **`'a: 'b`** อย่างชัดเจน)
2. Trait object ที่ต้องผูกกับ lifetime ที่ไม่ใช่ `'static`
3. Closure/function parameter ที่ต้องรับ reference ที่อายุถูกกำหนด**ที่จุดเรียกใช้ภายในฟังก์ชัน** ไม่ใช่ที่จุดประกาศ
   (higher-ranked trait bounds)
4. Trait ที่มี lifetime parameter เป็นของตัวเอง (แยกจากการที่ method ของ trait มี `&self` เฉย ๆ)
5. ความแตกต่างเชิงพื้นฐานระหว่าง lifetime (compile-time proof) กับ `Drop` (runtime mechanism)
6. ขอบเขตที่ lifetime ธรรมดาไปไม่ถึงเลย — self-referential struct

### 23.2 หลาย Lifetime Parameter ที่ไม่สัมพันธ์กันจริง ๆ: เมื่อ Output ผูกกับตัวเดียวเท่านั้น

Part 20 หัวข้อ 20.11 (ตัวอย่าง log parser) แนะนำแนวคิดนี้ไว้แล้วสั้น ๆ ผ่านฟังก์ชัน `find_first_matching_line(log:
&'a str, pattern: &str) -> Option<&'a str>` ที่ปล่อยให้ `pattern` ได้ lifetime ของตัวเองผ่าน**กฎ elision ข้อที่ 1**
โดยไม่ต้องเขียนชื่อ `'b` ให้เห็นเลยด้วยซ้ำ — บทนี้จะเขียนสถานการณ์เดียวกัน**แบบเต็มรูปแบบ** (ไม่พึ่ง elision) เพื่อให้
เห็นภาพชัดว่า **สิ่งที่ elision "ซ่อน" ไว้ให้ตลอด Part 20 จริง ๆ แล้วคืออะไร**

```rust
// เขียนแบบเต็ม ไม่พึ่ง elision เลย — 'a ผูกกับ text และ output, 'b ผูกกับ logger_tag เท่านั้น
// (ตัวอย่างนี้คือฟังก์ชันเดียวกับที่ elision จะสร้างให้อัตโนมัติถ้าเขียนแบบย่อว่า
//  fn first_word_with_logger(text: &str, logger_tag: &str) -> &str)
fn first_word_with_logger<'a, 'b>(text: &'a str, logger_tag: &'b str) -> &'a str {
    println!("[{logger_tag}] กำลังหาคำแรก");
    text.split_whitespace().next().unwrap_or(text)
}

fn main() {
    let sentence = String::from("Rust ปลอดภัยและเร็ว");
    let result;
    {
        let tag = String::from("LOG-001"); // อายุสั้นกว่า sentence มาก
        result = first_word_with_logger(&sentence, &tag);
    } // tag หมดอายุตรงนี้ — ไม่กระทบ result เลย เพราะ result ผูกกับ 'a (sentence) เท่านั้น
    println!("คำแรก: {result}");
}
```

ผลลัพธ์:

```
[LOG-001] กำลังหาคำแรก
คำแรก: Rust
```

**สังเกตประเด็นสำคัญที่สุด**: `tag` (ผูกกับ `'b`) หมดอายุไปแล้วตั้งแต่ก่อนบรรทัด `println!("คำแรก: {result}")` แต่โค้ด
ก็ยัง compile ผ่านและรันได้ปกติ เพราะ **`result` ไม่เคยถูกผูกกับ `'b` เลยแม้แต่นิดเดียว** — ถ้าเราเขียน signature แบบ
`fn first_word_with_logger<'a>(text: &'a str, logger_tag: &'a str) -> &'a str` (ผูก `'a` เดียวกันทั้งคู่แบบผิด ๆ
เหมือนกับดักข้อ 2 ของ Part 20) โค้ดชุดนี้จะ **compile ไม่ผ่าน** ทันที ด้วย error `E0597` แบบเดียวกับที่เห็นมาแล้วหลาย
ครั้งใน Part 20 — เพราะ `'a` ที่แท้จริงจะถูกบีบให้เท่ากับอายุที่สั้นที่สุดร่วมกัน (คืออายุของ `tag`) และ `result` จะ
ถูกห้ามใช้งานหลังจาก `tag` หมดอายุไปแล้ว

#### หลักการตัดสินใจที่ชัดเจนกว่าเดิม: "ตามรอย" ว่า Output จริง ๆ มาจากไหน

วิธีที่แน่นอนที่สุดในการตัดสินใจว่าควรผูก lifetime parameter สองตัวเข้าด้วยกันหรือแยกจากกัน คือ **ไล่ตามเนื้อโค้ด
ข้างในฟังก์ชันแบบ data-flow**: ถ้าค่าที่ถูก `return` (หรือกลายเป็นส่วนหนึ่งของ output type) **มีที่มาที่เป็นไปได้จาก
parameter ตัวใดบ้าง** ให้ parameter เหล่านั้นทั้งหมดใช้ `'a` ร่วมกัน ส่วน parameter ที่ **ไม่มีทางเป็นที่มาของ output
เลยไม่ว่ากรณีไหน** (ถูกใช้แค่เพื่อเปรียบเทียบ, log, คำนวณ index แบบไม่ผูก reference เช่น `usize`, หรืออะไรก็ตามที่
ไม่ได้ "หลุด" ออกไปเป็นส่วนหนึ่งของ return value) ให้แยก lifetime parameter ของตัวเองเสมอ — วิธีนี้ใช้ได้แม่นยำกว่าการ
"เดา" จากรูปร่างของ signature อย่างเดียว

ลองดูตัวอย่างที่ซับซ้อนขึ้นอีกขั้น: ฟังก์ชันที่มี reference parameter **สามตัว** ซึ่ง output มีที่มาได้จาก**สองใน
สามตัว**เท่านั้น:

```rust
// prefix กับ body มาจากแหล่งเดียวกันเสมอ (ทั้งคู่เป็นส่วนของ document เดียวกัน) -> ใช้ 'a ร่วมกัน
// suffix_hint ใช้แค่ตรวจสอบเงื่อนไข ไม่เคยถูกคืนออกมาเลย -> แยกเป็น 'b ของตัวเอง
fn pick_section<'a, 'b>(prefix: &'a str, body: &'a str, suffix_hint: &'b str) -> &'a str {
    if body.ends_with(suffix_hint) {
        body
    } else {
        prefix
    }
}

fn main() {
    let doc_prefix = String::from("บทนำ: ");
    let doc_body = String::from("เนื้อหาหลักของเอกสาร...จบบท");
    let result;
    {
        let hint = String::from("จบบท"); // hint มีอายุสั้นกว่า doc_prefix และ doc_body
        result = pick_section(&doc_prefix, &doc_body, &hint);
    } // hint หมดอายุ — ไม่กระทบ result เลย เพราะ result ผูกกับ 'a (prefix/body) เท่านั้น
    println!("ส่วนที่เลือก: {result}");
}
```

ผลลัพธ์:

```
ส่วนที่เลือก: เนื้อหาหลักของเอกสาร...จบบท
```

สังเกตว่า `prefix` และ `body` ใช้ `'a` **ร่วมกัน** (เพราะ output อาจมาจากตัวใดตัวหนึ่งของทั้งสอง เหมือนหลักการของ
`longest` ใน Part 20) แต่ `suffix_hint` แยกเป็น `'b` ของตัวเอง (เพราะไม่เคยเป็นที่มาของ output เลย) — นี่คือการผสม
สองหลักการที่ Part 20 สอนแยกกันไว้ (การผูก `'a` ร่วมกันแบบ `longest`, และการแยก `'a`/`'b` แบบ `PairDiff`) เข้ามาอยู่
ใน**ฟังก์ชันเดียวกัน** ซึ่งเป็นสิ่งที่พบได้บ่อยมากขึ้นเมื่อจำนวน parameter เพิ่มขึ้นในโค้ดจริง

### 23.3 Lifetime Subtyping และ Variance: อายุยืนกว่าใช้แทนอายุสั้นกว่าได้เสมอ

ตอนนี้มาถึงแนวคิดที่ Part 20 ยังไม่ได้แตะเลย: **lifetime subtyping** พูดแบบไม่ต้องใช้ศัพท์ทฤษฎีประเภทให้ซับซ้อนเกิน
จำเป็น หลักการคือประโยคเดียว: **ถ้า reference ตัวหนึ่งมีอายุยืนพอสำหรับสถานการณ์ที่ต้องการอายุสั้นกว่า มันก็ใช้แทนกัน
ได้เสมอ อย่างปลอดภัย** — เหมือนธนบัตรใบละ 1000 บาทใช้จ่ายแทนธนบัตรใบละ 100 บาทได้เสมอ (มีมูลค่ามากกว่าหรือเท่ากับที่
ต้องการ) แต่ธนบัตรใบละ 100 ใช้แทนใบละ 1000 ไม่ได้ (มีน้อยกว่าที่ต้องการ)

พูดเป็นภาษาที่ตรงกับสิ่งที่เห็นมาตลอด Part 20: **`'static` ใช้แทนที่ `'a` ใด ๆ ก็ได้เสมอ** (เพราะมันคืออายุที่ยืนยาว
ที่สุด) และโดยทั่วไป **`'long` ใช้แทน `'short` ได้ ถ้า `'long` มีอายุยืนไม่น้อยกว่า `'short`** — เขียนความสัมพันธ์นี้
ด้วย syntax ที่เรียกว่า **outlives bound**: `'long: 'short` (อ่านว่า **"`'long` outlives `'short`"** หรือ **"`'long`
มีอายุยืนไม่น้อยกว่า `'short`"**)

#### ทำไม Compiler เดาความสัมพันธ์นี้ไม่ได้เองในบางกรณี

ในกรณีส่วนใหญ่ที่เห็นมาตลอด Part 20 (เช่น `longest<'a>`) compiler ไม่จำเป็นต้องรู้เรื่อง subtyping เลย เพราะเรา
ผูก `'a` ตัวเดียวให้ทุกอย่างไปแล้ว แต่ทันทีที่ฟังก์ชันมี **สอง lifetime parameter ที่ต้อง "แปลง" ไปมาระหว่างกัน**
(เช่น รับค่าที่มีอายุยืน `'a` เข้ามา แล้วต้องคืนออกไปในรูปแบบ type ที่ระบุว่าอายุแค่ `'b` ที่สั้นกว่า) compiler จำเป็น
ต้อง**รู้ล่วงหน้า**ว่า `'a` ยืนยาวกว่า `'b` จริงหรือไม่ — ถ้าไม่มีข้อมูลนี้ มันจะ**ปฏิเสธเสมอ**เพื่อความปลอดภัย (ตาม
หลักการ "conservative เสมอ" ที่ Part 20 หัวข้อ 20.4 อธิบายไว้)

ลองดูตัวอย่างที่ง่ายที่สุดที่แสดงปัญหานี้ให้เห็นตรง ๆ — ฟังก์ชันที่ **จงใจ**ต้องการคืน reference ในรูปแบบ lifetime
ที่สั้นกว่า input ของมัน (สถานการณ์ที่เกิดขึ้นบ่อยเมื่อเขียนโค้ด generic ที่ทำงานกับ lifetime ของฟังก์ชันอื่นที่รับ
lifetime สั้นกว่า):

```rust
// ❌ พยายามคืน &'b str จาก &'a str โดยไม่บอก compiler ว่า 'a และ 'b สัมพันธ์กันอย่างไร
fn shorten<'a, 'b>(input: &'a str) -> &'b str {
    input
}

fn main() {
    let s = String::from("test");
    let r: &str = shorten(&s);
    println!("{r}");
}
```

Error ที่ได้:

```
error: lifetime may not live long enough
 --> src/main.rs:2:5
  |
1 | fn shorten<'a, 'b>(input: &'a str) -> &'b str {
  |            --  -- lifetime `'b` defined here
  |            |
  |            lifetime `'a` defined here
2 |     input
  |     ^^^^^ function was supposed to return data with lifetime `'b` but it is returning data with lifetime `'a`
  |
  = help: consider adding the following bound: `'a: 'b`
```

**อ่าน error นี้ให้ตรงประเด็นที่สุด**: compiler **ไม่ได้บอกว่าโค้ดนี้ผิดหลักการ** (ในทางความจริง การคืน `&'a str`
เป็น `&'b str` ที่สั้นกว่าปลอดภัยอยู่แล้วโดยสัญชาตญาณ) มันบอกตรง ๆ ว่า **"ฉันไม่รู้ว่า `'a` ยืนยาวกว่าหรือสั้นกว่า
`'b`"** — เพราะ `'a` และ `'b` เป็นแค่ชื่อสองชื่อที่ประกาศแยกกันอิสระ ไม่มีความสัมพันธ์อะไรระหว่างกันโดย default เลย
compiler ต้อง**เผื่อกรณีที่แย่ที่สุด**ไว้เสมอ (สมมติว่า `'b` อาจยาวกว่า `'a` ก็ได้) และในกรณีนั้นการคืน `input`
(อายุ `'a`) เป็น `&'b str` (ที่อาจยาวกว่า) จะไม่ปลอดภัยเลย — compiler เดาใจไม่ได้ว่าเราหมายถึงกรณีไหน จึงต้องขอให้
เราบอกมาให้ชัดเจน

**แถมท้าย error message บอกวิธีแก้ตรง ๆ**: `consider adding the following bound: 'a: 'b` — มาลองทำตามคำแนะนำนี้:

```rust
// ✅ เขียน where 'a: 'b เพื่อบอก compiler ว่า 'a ยืนยาวไม่น้อยกว่า 'b เสมอ
// (สัญญาที่ caller ต้องรักษา: 'a ต้องยาวกว่าหรือเท่ากับ 'b ทุกครั้งที่เรียกฟังก์ชันนี้)
fn shorten<'a, 'b>(input: &'a str) -> &'b str
where
    'a: 'b,
{
    input
}

fn main() {
    let s = String::from("อายุยืน");
    let r: &str;
    {
        // ในสถานการณ์จริง 'b มักจะถูกอนุมานให้เป็น lifetime ที่สั้นกว่า 'a โดยธรรมชาติ
        // ขึ้นอยู่กับว่า caller ต้องการใช้ผลลัพธ์นานแค่ไหน — compiler ตรวจสอบ where 'a: 'b ทุกครั้งที่เรียก
        r = shorten(&s);
    }
    println!("{r}");
}
```

ผลลัพธ์:

```
อายุยืน
```

**ความหมายของ `where 'a: 'b` ที่ต้องเข้าใจให้แม่นยำ**: มันคือ**สัญญาเพิ่มเติม**ที่ผูกไว้กับฟังก์ชันนี้ บอกว่า "ทุกครั้ง
ที่ใครเรียกใช้ `shorten` lifetime `'a` ที่ compiler อนุมานให้ในการเรียกครั้งนั้นต้องมีอายุยืนไม่น้อยกว่า `'b`
ที่อนุมานให้ในการเรียกครั้งเดียวกันเสมอ" — เมื่อมีสัญญานี้แล้ว compiler จึงพิสูจน์ได้ว่าการคืน `input` (มีอายุจริง
`'a` ที่ยาวกว่าหรือเท่ากับ) ให้กลายเป็น type ที่ประกาศว่าอายุแค่ `'b` (สั้นกว่าหรือเท่ากัน) นั้น**ปลอดภัยเสมอ** —
นี่คือแก่นของ **lifetime subtyping**: `&'a T` เป็น **subtype** ของ `&'b T` เมื่อ `'a: 'b` (เขียนสัญลักษณ์ทาง
ทฤษฎีประเภทว่า `&'a T <: &'b T` ถ้า `'a: 'b`) พูดง่าย ๆ คือ **ค่าที่มี type แบบ "อายุยืนกว่า" ใช้แทนที่ในที่ที่ต้องการ
type แบบ "อายุสั้นกว่า" ได้เสมอโดยไม่ต้องแปลงอะไร (ไม่มี runtime cost)** เหมือนหลักการ subtyping ในภาษา OOP ทั่วไป
(เช่น `Dog` เป็น subtype ของ `Animal` — ที่ไหนต้องการ `Animal` ก็ส่ง `Dog` ไปแทนได้) เพียงแต่ในที่นี้ "ประเภท" ที่
เกี่ยวข้องคือ **อายุการมีชีวิตของ reference** ไม่ใช่ชนิดข้อมูลตามปรกติ

#### กรณีนี้เกิดขึ้นบ่อยแค่ไหนในโค้ดจริง

ในทางปฏิบัติ **elision และการอนุมาน (inference) ของ compiler ครอบคลุมกรณีส่วนใหญ่ไปแล้วโดยที่คุณไม่ต้องเขียน
`where 'a: 'b` เองเลย** — โดยเฉพาะเมื่อ lifetime ทั้งสองปรากฏอยู่ใน**ฟังก์ชันเดียวที่มองเห็นทั้งภาพได้** (เหมือน
ตัวอย่าง `main` ข้างบน) compiler จะอนุมาน `'a` และ `'b` ให้เข้ากันได้เองโดยอัตโนมัติจาก scope จริงตอนเรียกใช้ —
สถานการณ์ที่**ต้อง**เขียน outlives bound อย่างชัดเจนด้วยมือ มักเกิดเมื่อ:

1. **ฟังก์ชันหรือ struct เป็น generic** และต้อง "แปลง" lifetime หนึ่งเป็นอีกอันหนึ่งผ่านการเรียกใช้ที่ไม่ได้อยู่ใน
   ฟังก์ชันเดียวกันตรง ๆ (จะเห็นตัวอย่างเต็มรูปแบบในหัวข้อ 23.4)
2. **trait ต้องการ bound เรื่อง lifetime** เพื่อรับประกันว่า type parameter บางตัวไม่ได้ยืมข้อมูลที่มีอายุสั้นกว่า
   lifetime อื่นที่เกี่ยวข้อง (พบในโค้ด library ระดับสูงที่ทำงานกับ trait object และ closure ผสมกัน)
3. **struct ที่มี reference ซ้อนกัน** (reference ของ reference เช่น `&'b &'a str`) ที่ต้องส่งผ่านเข้า/ออกจาก
   ฟังก์ชันในบริบทที่ implied bounds (ที่จะพูดถึงต่อไป) ไม่ทำงานให้อัตโนมัติ

**หมายเหตุที่ควรรู้ (ไม่ต้องจำละเอียด)**: Rust สมัยใหม่มีกลไกที่เรียกว่า **implied bounds** ซึ่งบางครั้งอนุมาน
ความสัมพันธ์แบบ `'a: 'b` ให้อัตโนมัติจากรูปร่างของ type เอง (เช่น type parameter ของฟังก์ชันที่เป็น `&'b &'a str`
compiler จะรู้เองว่า `'a: 'b` ต้องเป็นจริงเสมอ ไม่งั้น type นี้จะไม่มีความหมาย) — แต่กลไกนี้ทำงานเฉพาะบางบริบทเท่านั้น
(ส่วนใหญ่คือ parameter ของฟังก์ชันโดยตรง) เมื่อความสัมพันธ์ต้องข้าม generic type ที่ซับซ้อนกว่านั้น (แบบตัวอย่างถัดไป)
คุณยังต้องเขียน `where 'a: 'b` ด้วยมือเสมอ

### 23.4 Struct ที่มี Lifetime หลายตัว: ไปให้ลึกกว่า Part 20 หัวข้อ 20.7

Part 20 หัวข้อ 20.7 สอนหลักการเลือกระหว่าง `PairSame<'a>` (lifetime เดียว) กับ `PairDiff<'a, 'b>` (lifetime แยกกัน)
ไว้แล้วอย่างละเอียด — บทนี้จะไม่สอนหลักการนั้นซ้ำ แต่จะพาไปดูสถานการณ์ที่**ลึกกว่านั้นอีกขั้น**: เมื่อคุณมี struct
generic ที่ **เก็บ struct ที่มี lifetime parameter อีกตัวไว้เป็นส่วนประกอบ** แล้วต้องเขียนฟังก์ชันที่ "แปลง" อายุของ
มันจากยืนยาวไปเป็นสั้นกว่า — สถานการณ์แบบนี้คือจุดที่ **outlives bound จากหัวข้อ 23.3 กลายเป็นสิ่งจำเป็นจริง ๆ** ไม่ใช่
แค่ตัวอย่างสมมติทางทฤษฎี

ลองนิยาม struct wrapper แบบ generic ที่เก็บ reference ไปยัง type อะไรก็ได้ (คล้ายกับที่ smart pointer หลายตัวใน
`std` ทำ ซึ่งจะเรียนเต็มรูปแบบใน Part 27-28):

```rust
// Ref<'a, T> คือ wrapper ง่าย ๆ ที่เก็บ &'a T ไว้ (แนวคิดคล้าย &T ธรรมดา แต่ครอบด้วย struct ของเราเอง
// เพื่อจะเพิ่ม method หรือ trait implementation อื่น ๆ ในอนาคตได้ — pattern นี้พบบ่อยในโค้ด library จริง)
struct Ref<'a, T: ?Sized>(&'a T);
```

สมมติว่าเรามี `Ref<'a, String>` ที่ผูกกับ `'a` ซึ่งเป็น lifetime ที่ยืนยาว (เช่นมาจากตัวแปรระดับบนของ `main`) แต่
ต้องการใช้มันในบริบทที่คาดหวัง `Ref<'b, String>` ที่ `'b` สั้นกว่า (เช่น ฟังก์ชันข้างในที่ scope แคบกว่า) — ในทาง
สัญชาตญาณ นี่ควรทำได้เสมอ (อายุยืนกว่าใช้แทนอายุสั้นกว่าได้ ตามหลัก subtyping) แต่ลองดูว่าเกิดอะไรขึ้นถ้า**ไม่บอก
compiler ถึงความสัมพันธ์นี้**:

```rust
struct Ref<'a, T: ?Sized>(&'a T);

// ❌ ลืมเขียน where 'a: 'b — compiler ไม่รู้ว่า 'a ยาวกว่า 'b จริงหรือไม่
fn extend_scope<'a, 'b, T>(r: Ref<'a, T>) -> Ref<'b, T> {
    Ref(r.0)
}

fn main() {
    let owner = String::from("ข้อมูลต้นทาง");
    let long: Ref<'_, String> = Ref(&owner);
    let short: Ref<'_, String> = extend_scope(long);
    println!("{}", short.0);
}
```

```
error: lifetime may not live long enough
 --> src/main.rs:4:5
  |
3 | fn extend_scope<'a, 'b, T>(r: Ref<'a, T>) -> Ref<'b, T> {
  |                 --  -- lifetime `'b` defined here
  |                 |
  |                 lifetime `'a` defined here
4 |     Ref(r.0)
  |     ^^^^^^^^ function was supposed to return data with lifetime `'b` but it is returning data with lifetime `'a`
  |
  = help: consider adding the following bound: `'a: 'b`
```

Error เดียวกันกับหัวข้อ 23.3 เป๊ะ ๆ — เพราะรากของปัญหาคือหลักการเดียวกัน เพียงแต่คราวนี้ `'a` และ `'b` ไม่ได้ปรากฏ
บน `&str` ตรง ๆ แต่ปรากฏอยู่**ข้างในพารามิเตอร์ generic ของ struct** (`Ref<'a, T>` และ `Ref<'b, T>`) — compiler ยัง
คงต้องการให้เราประกาศความสัมพันธ์เดียวกันนี้ชัดเจนอยู่ดี ไม่ว่า lifetime นั้นจะซ่อนอยู่ลึกแค่ไหนในโครงสร้าง type ก็ตาม

เพิ่ม `where 'a: 'b` แล้วโค้ด compile ผ่านทันที:

```rust
struct Ref<'a, T: ?Sized>(&'a T);

// ✅ เขียน where 'a: 'b ชัดเจน: "Ref<'a, T> ที่รับเข้ามาต้องมีอายุยืนไม่น้อยกว่า Ref<'b, T> ที่จะคืนออกไป"
fn extend_scope<'a, 'b, T>(r: Ref<'a, T>) -> Ref<'b, T>
where
    'a: 'b,
{
    Ref(r.0)
}

fn main() {
    let owner = String::from("ข้อมูลต้นทาง");
    let long: Ref<'_, String> = Ref(&owner);
    let short: Ref<'_, String> = extend_scope(long);
    println!("{}", short.0);
}
```

ผลลัพธ์:

```
ข้อมูลต้นทาง
```

**ทำไมฟังก์ชันแบบนี้มีประโยชน์จริงในทางปฏิบัติ**: รูปแบบ `extend_scope` (แปลง `Ref<'a, T>` เป็น `Ref<'b, T>` ที่สั้น
กว่าโดยมี `where 'a: 'b`) คือ pattern ที่พบบ่อยมากเมื่อคุณเขียนฟังก์ชัน generic ที่ต้องส่ง reference wrapper (ไม่ว่า
จะเป็น struct ที่เขียนเองแบบนี้ หรือ type จาก `std` ที่จะเรียนใน Part 27-28) เข้าไปในฟังก์ชันย่อยที่ประกาศ signature
ด้วย lifetime parameter ของตัวเอง (สั้นกว่า) — โดยไม่ต้อง clone หรือแปลงข้อมูลใหม่เลย เป็นตัวอย่างที่แสดงให้เห็นว่า
**subtyping ทำให้ generic code ที่ทำงานกับ lifetime ยืดหยุ่นได้มากกว่าที่คิด** ตราบใดที่เราบอก compiler ถึงความ
สัมพันธ์ที่ถูกต้องผ่าน outlives bound

#### ทวนหลักการตัดสินใจจาก Part 20 หัวข้อ 20.7 พร้อมเพิ่มเงื่อนไขใหม่

นำหลักการเดิมจาก Part 20 มาต่อยอด ตอนนี้เรามีเงื่อนไขที่สมบูรณ์ขึ้นสำหรับตัดสินใจเรื่อง lifetime parameter หลายตัว:

| สถานการณ์ | การตัดสินใจ |
|---|---|
| Field ทั้งหมดมาจากแหล่งเดียวกันเสมอในทางความหมาย | `'a` ตัวเดียวร่วมกัน (`Excerpt<'a>`, `KeyValue<'a>` จาก Part 20) |
| Field มาจากแหล่งที่เป็นอิสระจากกันจริง ไม่มีความสัมพันธ์เชิงความหมาย | แยก `'a`, `'b` คนละตัว ไม่ผูกกัน (`PairDiff<'a, 'b>` จาก Part 20) |
| ต้อง "แปลง" ค่าที่มี lifetime หนึ่งให้กลายเป็น type ที่ระบุ lifetime อื่นที่สั้นกว่า (เช่นส่งเข้าฟังก์ชันย่อยที่ scope แคบกว่า) | แยก `'a`, `'b` และเขียน **`where 'a: 'b`** ชัดเจน (`extend_scope` ข้างบน) |
| ไม่แน่ใจว่าความสัมพันธ์ระหว่าง lifetime สองตัวเป็นแบบไหน | ให้ compiler บอกก่อนเสมอ — เขียนโค้ดแบบง่ายที่สุดโดยไม่เดา แล้วปล่อยให้ error message (ที่มักแนะนำ `where` bound ที่ถูกต้องมาให้เลย) นำทาง |

แถวสุดท้ายคือทักษะเชิงปฏิบัติที่สำคัญที่สุด: **ไม่จำเป็นต้องท่องจำว่าเมื่อไหร่ต้องเขียน `where 'a: 'b`** — ในทาง
ปฏิบัติ คุณเขียนโค้ดแบบตรงไปตรงมาที่สุดก่อนเสมอ แล้ว **ปล่อยให้ compiler error message ("consider adding the
following bound") เป็นตัวบอกเราเองว่าต้องเพิ่ม bound อะไร** เหมือนที่เกิดขึ้นในทั้งสองตัวอย่างข้างบน — นี่คือเหตุผล
ที่ Rust compiler ได้รับการยกย่องว่า error message ของมัน "สอนภาษา" ไปในตัว ไม่ใช่แค่บอกว่าอะไรผิด

### 23.5 Trait Object กับ Lifetime: `Box<dyn Trait + 'a>` และ Default ที่เป็น `'static`

Part 21 แนะนำ `dyn Trait` และ `Box<dyn Trait>` ในฐานะเครื่องมือเก็บหลาย type ที่ implement trait เดียวกันไว้ใน
โครงสร้างเดียว ผ่าน dynamic dispatch — สิ่งที่ Part 21 อาจไม่ได้เจาะลึกคือ **`dyn Trait` ทุกตัวมี lifetime bound
แฝงอยู่เสมอ แม้จะไม่เห็นเขียนไว้ตรง ๆ ก็ตาม**

#### ทำไม Trait Object ต้องมี Lifetime Bound ด้วย

ทวนความจำสั้น ๆ จาก Part 20: reference ทุกตัวต้องมี lifetime เพื่อให้ compiler พิสูจน์ได้ว่าไม่ dangling — `dyn Trait`
ก็เป็น type ที่ **อาจมีข้อมูลที่ยืมมาจากที่อื่นซ่อนอยู่ภายใน** ได้เหมือนกัน (เช่น struct ที่ implement trait นั้นอาจ
มี field เป็น `&'a str`) ดังนั้น `dyn Trait` **ก็ต้องมีข้อจำกัดเรื่องอายุเหมือนกัน** เพื่อให้ compiler พิสูจน์ความ
ปลอดภัยได้ — คำถามคือ: ถ้าเราเขียน `Box<dyn Greet>` เฉย ๆ (ไม่เห็น lifetime ที่ไหนเลย) มันหมายถึง lifetime อะไร?

**คำตอบ**: ในบริบทส่วนใหญ่ (เช่น type ของ field ใน struct, return type ของฟังก์ชัน, หรือ type annotation ทั่วไป)
`dyn Trait` ที่ไม่เขียน lifetime ไว้ **จะถูก default เป็น `dyn Trait + 'static` โดยอัตโนมัติ** — นี่คือกฎที่แยกจาก
กฎ elision 3 ข้อของ Part 20 โดยสิ้นเชิง (elision ทำงานกับ reference `&'a T` ตรง ๆ ส่วนกฎนี้ทำงานกับ **trait object
bound**) แต่ก็เป็น "ค่า default" ในลักษณะเดียวกัน — **ความหมายคือ: ถ้าไม่ระบุอะไรเพิ่ม compiler จะสมมติว่า type
ที่ซ่อนอยู่หลัง `dyn Trait` นั้นไม่มีการยืมข้อมูลที่มีอายุจำกัดมาจากที่ไหนเลย (เป็น owned data ทั้งหมด หรือยืมแต่
ข้อมูลระดับโปรแกรมอย่าง string literal เท่านั้น)**

#### เมื่อ Default `'static` ใช้ไม่ได้จริง: ต้อง Override ด้วย Lifetime ที่สั้นกว่า

ลองดูสถานการณ์ที่ type ที่จะใส่ลงใน trait object **มี field เป็น reference ที่ไม่ใช่ `'static`**:

```rust
trait Greet {
    fn greet(&self) -> String;
}

struct Formal<'a> {
    name: &'a str,
}

impl<'a> Greet for Formal<'a> {
    fn greet(&self) -> String {
        format!("สวัสดีครับ คุณ {}", self.name)
    }
}

// ❌ ไม่เขียน + 'a ให้ dyn Greet -> default กลายเป็น dyn Greet + 'static ซึ่ง Formal<'a> ไม่เข้าเงื่อนไข
fn make_greeter<'a>(name: &'a str) -> Box<dyn Greet> {
    Box::new(Formal { name })
}

fn main() {
    let name = String::from("สมชาย");
    let greeter = make_greeter(&name);
    println!("{}", greeter.greet());
}
```

Error:

```
error: lifetime may not live long enough
  --> src/main.rs:17:5
   |
16 | fn make_greeter<'a>(name: &'a str) -> Box<dyn Greet> {
   |                 -- lifetime `'a` defined here
17 |     Box::new(Formal { name })
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^ returning this value requires that `'a` must outlive `'static`
   |
help: to declare that the trait object captures data from argument `name`, you can add an explicit `'a` lifetime bound
   |
16 | fn make_greeter<'a>(name: &'a str) -> Box<dyn Greet + 'a> {
   |                                                     ++++
```

**อ่าน error นี้ให้ตรงจุด**: `returning this value requires that 'a must outlive 'static` — เพราะ `Box<dyn Greet>`
(ไม่เขียน lifetime) ถูกตีความเป็น `Box<dyn Greet + 'static>` โดย default และ compiler พยายามพิสูจน์ว่า `Formal<'a>`
(ซึ่งมี `'a` ที่มาจาก parameter — ไม่มีทางเป็น `'static` ได้เสมอไป) เข้าเงื่อนไข `+ 'static` นั้น ซึ่งเป็นไปไม่ได้
ถ้า `'a` ไม่ใช่ `'static` จริง ๆ — **compiler ให้ help message บอกวิธีแก้มาตรง ๆ เลย**: เขียน `Box<dyn Greet + 'a>`
แทน

แก้ตามคำแนะนำ:

```rust
trait Greet {
    fn greet(&self) -> String;
}

struct Formal<'a> {
    name: &'a str,
}

impl<'a> Greet for Formal<'a> {
    fn greet(&self) -> String {
        format!("สวัสดีครับ คุณ {}", self.name)
    }
}

// ✅ เขียน + 'a อย่างชัดเจน: "trait object นี้อาจเก็บข้อมูลที่ยืมมาด้วยอายุ 'a (ไม่ใช่ 'static)"
fn make_greeter<'a>(name: &'a str) -> Box<dyn Greet + 'a> {
    Box::new(Formal { name })
}

fn main() {
    let name = String::from("สมชาย");
    let greeter = make_greeter(&name);
    println!("{}", greeter.greet());
}
```

ผลลัพธ์:

```
สวัสดีครับ คุณ สมชาย
```

**ทำไม default ต้องเป็น `'static` แทนที่จะเป็นอย่างอื่น**: เหตุผลเชิง design คือ **`'static` คือค่า default ที่
"ปลอดภัยที่สุดในทางการใช้งาน"** — เมื่อ dev ส่วนใหญ่เขียน `Box<dyn Trait>` โดยไม่คิดเรื่อง lifetime เลย (เพราะกรณี
ส่วนใหญ่ trait object ที่เก็บใน struct หรือคืนออกจากฟังก์ชันมักเป็นข้อมูล owned ทั้งหมด เช่น struct ที่มี field เป็น
`String`, `Vec<T>` ไม่มี reference เจือปนอยู่เลย) ค่า default นี้จะ**ทำงานถูกต้องได้เองโดยไม่ต้องเขียนอะไรเพิ่ม** —
กรณีที่ trait object ต้องเก็บ reference ที่มีอายุจำกัด (แบบ `Formal<'a>` ข้างบน) เป็นกรณี**พิเศษที่พบน้อยกว่า** และ
compiler เลือกที่จะ**บังคับให้ผู้เขียนโค้ดต้อง "ประกาศ" ความตั้งใจนี้อย่างชัดเจน**ผ่าน `+ 'a` แทนที่จะเดาให้ ซึ่ง
สอดคล้องกับปรัชญาของ Rust ที่ต้องการให้ signature เป็น "สัญญาที่ชัดเจนและตรวจสอบได้เสมอ" (ตามที่ Part 20 หัวข้อ 20.2
อธิบายไว้)

#### เขียนแบบเดียวกันโดยไม่ใช้ `Box`: `&dyn Trait + 'a`

หลักการเดียวกันนี้ใช้ได้กับ trait object ที่ไม่ได้ครอบด้วย `Box` ด้วย เช่น `&'a dyn Trait` (reference ธรรมดาไปยัง
trait object) — ในกรณีนี้ lifetime ของ reference ด้านนอก (`'a` หน้า `dyn`) และ lifetime bound ของ trait object
เอง (`+ 'a` ถ้าจำเป็น) เป็นสองสิ่งที่แยกกัน แต่ในทางปฏิบัติส่วนใหญ่ compiler จะอนุมาน `dyn Trait` ให้ผูกกับ lifetime
เดียวกันกับ reference ด้านนอกโดยอัตโนมัติผ่าน elision (คล้ายกฎข้อที่ 1 ของ Part 20 ที่ขยายมาใช้กับ trait object ด้วย)
จึงไม่ต้องเขียน `+ 'a` ซ้ำสองรอบในกรณีที่ trait object นั้นถูกใช้อยู่หลัง reference อยู่แล้ว

### 23.6 Higher-Ranked Trait Bounds (HRTB): เมื่อ Lifetime ถูกกำหนดที่จุดเรียกใช้ ไม่ใช่ที่จุดประกาศ

หัวข้อนี้คือส่วนที่หลายคนบอกว่า "ยากที่สุด" ของเรื่อง lifetime ทั้งหมดในภาษา Rust — แต่ในทางปฏิบัติ **คุณไม่จำเป็น
ต้องเขียน syntax นี้เองบ่อยเลย** สิ่งสำคัญที่สุดคือ**อ่านมันออกเมื่อเจอในเอกสารหรือ error message** ต่างหาก

#### สร้างปัญหาให้เห็นก่อน: ทำไม Lifetime แบบธรรมดาใช้ไม่ได้กับ Closure บางแบบ

ลองนึกภาพฟังก์ชันที่รับ **closure เป็น parameter** โดย closure นั้นต้องรับ `&str` แล้วคืน `&str` ที่ผูกกับ input ของ
มัน (เหมือน pattern `first_word` จาก Part 20) — เขียนแบบที่ "ดูเหมือน" น่าจะถูกต้องตามความรู้จาก Part 20 ก่อน:

```rust
// ❌ naive attempt: 'a ถูกประกาศเป็น generic parameter ของ apply_to_str เอง
// -> 'a ถูก "กำหนดค่าจริง" ตอนที่ caller เรียก apply_to_str (จากภายนอกฟังก์ชัน) ไม่ใช่ตอนที่เราเรียก f(&s) ข้างใน
fn apply_to_str<'a, F>(f: F) -> String
where
    F: Fn(&'a str) -> &'a str,
{
    let s = String::from("hello");
    f(&s).to_string()
}

fn identity(s: &str) -> &str {
    s
}

fn main() {
    println!("{}", apply_to_str(identity));
}
```

Error:

```
error[E0597]: `s` does not live long enough
 --> src/main.rs:7:7
  |
2 | fn apply_to_str<'a, F>(f: F) -> String
  |                 -- lifetime `'a` defined here
...
6 |     let s = String::from("hello");
  |         - binding `s` declared here
7 |     f(&s).to_string()
  |     --^^-
  |     | |
  |     | borrowed value does not live long enough
  |     argument requires that `s` is borrowed for `'a`
8 | }
  | - `s` dropped here while still borrowed
```

**นี่คือจุดที่มือใหม่ (และผู้มีประสบการณ์จำนวนมาก) งงที่สุด**: `s` ถูกสร้างและ drop อยู่**ภายใน**ฟังก์ชัน
`apply_to_str` ทั้งหมด ไม่มีทางมี "อายุสั้นเกินไป" เมื่อมองจากภายในฟังก์ชันเดียวกันเลย — แต่ error บอกว่า `s` "does
not live long enough" ทั้ง ๆ ที่มันมีชีวิตอยู่ตลอดฟังก์ชันเลยด้วยซ้ำ! **ต้นเหตุจริง ๆ ไม่ได้อยู่ที่อายุของ `s`** แต่
อยู่ที่ **`'a` ในที่นี้ถูกกำหนดให้เป็น generic parameter ของฟังก์ชัน `apply_to_str` เอง** ซึ่งหมายความว่า **`'a` ที่
แท้จริงจะถูกเลือกโดย caller ของ `apply_to_str` (จากภายนอก) ตั้งแต่ก่อนที่ `apply_to_str` จะเริ่มรันด้วยซ้ำ** —
signature `F: Fn(&'a str) -> &'a str` (เมื่อ `'a` มาจาก generic parameter ของฟังก์ชันเอง) จึงแปลว่า **"closure
`f` ต้องรับได้เฉพาะ reference ที่มีอายุเท่ากับ `'a` ตัวเดียวที่ตายตัวมาตั้งแต่ก่อนเรียกฟังก์ชันนี้"** — แต่ `&s` ที่
สร้างขึ้น**ข้างใน**ฟังก์ชันมี lifetime ที่เกิดขึ้นทีหลัง `'a` (เพราะ `s` ยังไม่มีตัวตนตอนที่ caller เลือก `'a`) จึง
**ไม่มีทางตรงกับ `'a` ที่ตายตัวมาก่อนแล้วได้เลย** นี่คือรากของปัญหาที่แท้จริง — ไม่เกี่ยวกับ "อายุยาวหรือสั้น" แต่
เกี่ยวกับ **"lifetime ไหนถูกเลือกก่อน ไหนถูกเลือกทีหลัง"**

#### ทางแก้: Higher-Ranked Trait Bound ด้วย `for<'a>`

สิ่งที่เราต้องการจริง ๆ ไม่ใช่ "closure ที่รับ `&'a str` ด้วย `'a` ตัวเดียวที่ตายตัว" แต่คือ **"closure ที่รับ
reference ด้วย lifetime **อะไรก็ได้** ที่กำหนดใหม่ทุกครั้งที่ถูกเรียก"** — นี่คือสิ่งที่ syntax **`for<'a>`** (อ่านว่า
**"for all `'a`"** หรือ **"สำหรับทุก ๆ `'a`"**) ทำได้:

```rust
// ✅ for<'a> บอกว่า: F ต้องรับได้กับ 'a "อะไรก็ได้" ที่จะถูกเลือกใหม่ทุกครั้งที่ f ถูกเรียก (ไม่ใช่ 'a ตัวเดียวตายตัว)
fn apply_to_str<F>(f: F) -> String
where
    F: for<'a> Fn(&'a str) -> &'a str,
{
    let s = String::from("hello");
    f(&s).to_string() // ตอนนี้ 'a ถูกเลือกใหม่ ณ จุดที่เรียก f(&s) นี้เอง — ตรงกับอายุของ &s พอดี
}

fn identity(s: &str) -> &str {
    s
}

fn main() {
    println!("{}", apply_to_str(identity));
}
```

ผลลัพธ์:

```
hello
```

**ความแตกต่างเชิงหลักการระหว่างสองเวอร์ชันนี้คือหัวใจของ HRTB ทั้งหมด**:

- `F: Fn(&'a str) -> &'a str` (เมื่อ `'a` เป็น generic parameter ของฟังก์ชันข้างนอก) แปลว่า **"เลือก `'a` ตัวเดียว
  ตายตัวไว้ล่วงหน้า (rank สูงกว่า — determined outside) แล้ว `f` ต้องใช้กับ `'a` ตัวนั้นเท่านั้นตลอด"**
- `F: for<'a> Fn(&'a str) -> &'a str` แปลว่า **"`f` ต้องเป็น closure ที่ใช้ได้กับ `'a` **ทุกค่าที่เป็นไปได้**
  (rank ต่ำกว่า — determined ใหม่ทุกครั้งที่ `f` ถูกเรียก ภายในฟังก์ชันของเราเอง)"**

คำว่า **"Higher-Ranked"** มาจากการที่ bound แบบ `for<'a> Fn(...)` เป็น bound ที่ **"quantify เหนือ lifetime"**
(บอกว่า "สำหรับทุกค่าที่เป็นไปได้ของ `'a`") ซึ่งเป็นแนวคิดที่ "อยู่สูงกว่า" การประกาศ `'a` เป็น generic parameter
ตัวเดียวธรรมดา (ที่ "quantify" แค่ครั้งเดียวตอนเรียกฟังก์ชันข้างนอก) — ไม่ต้องจำศัพท์นี้ก็ได้ สิ่งที่ต้องจำคือ
**`for<'a>` แปลว่า "closure/function ตัวนี้ทำงานได้กับ reference ที่มีอายุอะไรก็ได้ ไม่ผูกติดกับ `'a` ตัวเดียว"**

#### ข่าวดี: กรณีส่วนใหญ่ Compiler เขียน `for<'a>` ให้เราอัตโนมัติ

ในทางปฏิบัติ ถ้าคุณเขียน bound แบบ **ไม่เอ่ยชื่อ lifetime เลย** (ปล่อยให้ elision ทำงาน) compiler จะแปลงเป็น
`for<'a>` ให้อัตโนมัติเสมอ โดยไม่ต้องเขียน syntax นี้ให้เห็นด้วยตัวเองแม้แต่ตัวเดียว:

```rust
// เขียนแบบไม่เอ่ยชื่อ lifetime เลย — compiler แปลงเป็น for<'a> Fn(&'a str) -> &'a str ให้อัตโนมัติเบื้องหลัง
// (เทียบเท่ากับเวอร์ชัน for<'a> ข้างบนทุกประการ ไม่ต่างกันแม้แต่นิดเดียวตอน compile)
fn apply_to_str<F>(f: F) -> String
where
    F: Fn(&str) -> &str,
{
    let s = String::from("hello");
    f(&s).to_string()
}

fn identity(s: &str) -> &str {
    s
}

fn main() {
    println!("{}", apply_to_str(identity));
}
```

ผลลัพธ์:

```
hello
```

**นี่คือคำตอบว่าทำไมคุณอาจไม่เคยสังเกตว่ากำลังใช้ HRTB มาก่อนเลยตลอดหลักสูตร**: ทุกครั้งที่คุณเขียน closure bound
แบบ `Fn(&Item) -> bool` (เช่นใน `Iterator::filter` ที่จะเรียนเต็มรูปแบบใน Part 25-26) หรือ `Fn(&str) -> ...` โดย
ไม่เอ่ยชื่อ lifetime เลย **มันคือ `for<'a> Fn(&'a Item) -> bool` เสมอโดยปริยาย** — elision (ในความหมายกว้างกว่า
กฎ 3 ข้อของ Part 20) ทำงานร่วมกับ trait bound แบบนี้ด้วย และเลือก **HRTB เป็นค่า default เสมอเมื่อไม่เอ่ยชื่อ
lifetime** เพราะนี่คือกรณีที่มีประโยชน์และปลอดภัยที่สุดสำหรับ closure ที่รับ reference (closure ส่วนใหญ่ที่รับ
`&T` ควรทำงานได้กับ reference ที่มีอายุอะไรก็ได้ ไม่ควรถูกจำกัดให้ใช้ได้กับ `'a` ตัวเดียวตายตัว)

#### จุดที่จะเจอ HRTB บ่อยที่สุดในโค้ดจริง: Error Message ของ Iterator Adapter

ในความเป็นจริง คุณไม่ค่อยต้องเขียน `for<'a>` ด้วยตัวเองเลย (เพราะ elision จัดการให้อัตโนมัติแบบข้างบน) แต่คุณจะ
**เห็น syntax นี้ปรากฏใน error message** บ่อยครั้งเมื่อเขียน closure ที่ capture ตัวแปรผิดวิธีเข้ากับ iterator
adapter (`.filter()`, `.map()`, `.find()` — จะเรียนเต็มใน Part 25-26) error message ประเภทนี้มักมีคำว่า
**"implementation of `FnMut` is not general enough"** หรือ **"one type is more general than the other"** ซึ่ง
เป็นสัญญาณว่า closure ของคุณถูกจำกัดด้วย `'a` ตัวเดียวตายตัว (มักเกิดจาก closure capture reference บางตัวเข้ามา
โดยไม่ตั้งใจ) ทั้ง ๆ ที่ trait bound ของ iterator adapter ต้องการ `for<'a>` เต็มรูปแบบ — **จุดสำคัญที่ต้องจำไว้
ในระดับปฏิบัติ**: เมื่อเจอ error message ที่มีคำว่า "not general enough" ให้สงสัยเรื่อง HRTB ทันที และมักแก้ได้
ด้วยการปรับ closure ให้ไม่ capture reference ที่ทำให้มันถูกผูกกับ lifetime ใดตายตัวเกินไป

### 23.7 Lifetime ใน Trait Definition และ Implementation

หัวข้อนี้แยกออกเป็นสองกรณีที่มือใหม่มักปนกัน: **(1)** method ของ trait ที่คืน reference ผูกกับ `&self` เฉย ๆ
(เป็นแค่กฎ elision ข้อที่ 3 ของ Part 20 ที่นำมาใช้ในบริบท trait) และ **(2)** trait ที่มี **lifetime parameter เป็น
ของตัวเอง** (ซึ่งเป็นเรื่องใหม่ที่ Part 19-21 ยังไม่ได้แตะ)

#### กรณีที่ 1: Method คืน Reference ผูกกับ `&self` (ทวนกฎข้อที่ 3 ในบริบท Trait)

```rust
// rule 3 revisited: method นี้ไม่ต้องเขียน 'a เลย เพราะกฎ elision ข้อที่ 3 (Part 20) ผูก output กับ &self ให้
trait Summary {
    fn headline(&self) -> &str;
}

struct Article {
    title: String,
}

impl Summary for Article {
    fn headline(&self) -> &str {
        &self.title
    }
}

fn main() {
    let article = Article { title: String::from("Rust คืออะไร") };
    println!("{}", article.headline());
}
```

ผลลัพธ์:

```
Rust คืออะไร
```

สังเกตว่า**ไม่มีอะไรใหม่**ในตัวอย่างนี้เมื่อเทียบกับ Part 20 หัวข้อ 20.8 — trait definition ก็เป็นแค่การเขียน
function signature อีกรูปแบบหนึ่ง กฎ elision ข้อที่ 3 (ใช้ lifetime ของ `&self` กับ output ที่ elided) ทำงาน
เหมือนกันทุกประการไม่ว่า method นั้นจะประกาศอยู่ใน `trait` หรือใน `impl` ตรง ๆ — ทุก type ที่ `impl Summary`
(เช่น `Article`) ต้องเขียน `fn headline(&self) -> &str` ตาม signature ที่ trait กำหนด และ output lifetime
จะผูกกับ `&self` ของ**การเรียกครั้งนั้น** เสมอ ตามหลักการที่อธิบายละเอียดแล้วใน Part 20 หัวข้อ 20.8

#### กรณีที่ 2: Trait ที่มี Lifetime Parameter เป็นของตัวเอง

ต่างจากกรณีที่ 1 (ที่ lifetime มาจาก `&self` ของแต่ละ instance) — บางครั้ง trait ต้องการ**ผูก lifetime ของ**
**parameter ที่ method รับเข้ามา**ไว้กับ trait เอง ไม่ใช่ผูกกับ `&self` — สถานการณ์นี้ต้องเขียน **lifetime
parameter บนตัว `trait` เอง** (ตำแหน่งเดียวกับที่เขียนบน struct จาก Part 20 หัวข้อ 20.7):

```rust
// trait Parser<'a> — 'a เป็น lifetime parameter ของ "trait" เอง ไม่ใช่ของ &self
// ความหมาย: ทุก type ที่ implement Parser<'a> ต้องรับ input ที่มี lifetime 'a เดียวกันกับที่ implementation ระบุ
trait Parser<'a> {
    fn parse(&self, input: &'a str) -> &'a str;
}

struct TrimParser;

impl<'a> Parser<'a> for TrimParser {
    fn parse(&self, input: &'a str) -> &'a str {
        input.trim()
    }
}

fn main() {
    let raw = String::from("   hello world   ");
    let parser = TrimParser;
    println!("[{}]", parser.parse(&raw));
}
```

ผลลัพธ์:

```
[hello world]
```

**ความแตกต่างเชิงความหมายที่สำคัญที่สุดระหว่างสองกรณีนี้**: ใน `trait Summary { fn headline(&self) -> &str; }`
lifetime ของ output ผูกกับ **การยืม `&self` ของแต่ละครั้งที่เรียก** (ตามกฎข้อที่ 3) — เปลี่ยนไปทุกครั้งที่เรียก
แตกต่างกันได้ตามแต่ละ call site ส่วนใน `trait Parser<'a> { fn parse(&self, input: &'a str) -> &'a str; }`
`'a` เป็น**ส่วนหนึ่งของตัว trait เอง** — เมื่อคุณเขียน `impl<'a> Parser<'a> for TrimParser` คุณกำลังบอกว่า
`TrimParser` implement `Parser` สำหรับ **`'a` ค่าหนึ่งที่ compiler จะอนุมานใหม่ในแต่ละจุดที่ใช้งาน** (เพราะเขียน
`impl<'a>` แบบ generic ไว้) — แต่ในทางเทคนิค `'a` ตรงนี้คือ**พารามิเตอร์ของ trait bound เอง** ซึ่งเปิดโอกาสให้เขียน
โค้ดที่ระบุ **`'a` ที่ตายตัวชัดเจน** ได้ด้วย เช่น `impl Parser<'static> for SomeType` (บอกว่า type นี้ implement
`Parser` เฉพาะสำหรับข้อมูลที่มีอายุยืนตลอดโปรแกรมเท่านั้น) ซึ่งเป็นความสามารถที่กรณีแรก (`&self` ธรรมดา) ทำไม่ได้เลย
เพราะ lifetime ของมันผูกกับการยืม ณ ขณะนั้นเสมอ ไม่ใช่ส่วนหนึ่งของ trait

**เมื่อไหร่ควรใช้แบบไหน**: ใช้กรณีที่ 1 (`&self` ธรรมดา, ปล่อยให้ elision ทำงาน) เป็นค่าเริ่มต้นเสมอ — มันครอบคลุม
สถานการณ์ส่วนใหญ่ที่ trait แค่ต้องคืนข้อมูลบางส่วนของตัวเอง ใช้กรณีที่ 2 (`trait Parser<'a>`) เฉพาะเมื่อคุณต้องการ
ให้ **lifetime ของข้อมูลที่ method ประมวลผลเป็นส่วนหนึ่งของ trait bound เอง** เช่นเมื่อต้องเขียนโค้ด generic ที่รับ
`T: Parser<'a>` แล้วต้องพูดถึง `'a` นั้นในที่อื่นของ signature ด้วย (สถานการณ์นี้พบบ่อยในโค้ด parser/deserializer
ระดับ library เช่น `serde` ที่ Part 20 หัวข้อ 20.7 กล่าวถึงไว้สั้น ๆ ผ่าน convention การตั้งชื่อ `'de`)

### 23.8 Lifetime ไม่ใช่กลไก Runtime: พิสูจน์ให้เห็นจริงด้วยการเทียบกับ `Drop`

นี่คือความเข้าใจผิดที่พบบ่อยที่สุดข้อหนึ่งในทั้งภาษา แม้จะพูดถึงสั้น ๆ ไปแล้วใน Part 20 หัวข้อ 20.1 และ 20.3 ก็ตาม
บทนี้จะ**พิสูจน์ให้เห็นจริงด้วยโค้ดที่รันได้** ไม่ใช่แค่บอกด้วยคำพูด

ทวนหลักการจาก Part 6: **`Drop`** คือ trait ที่ทำให้คุณเขียนโค้ดที่**รันจริงตอน runtime**เมื่อค่าใดค่าหนึ่งหมด scope
— นี่คือ**กลไก runtime แท้จริง**: มันมีผลต่อสิ่งที่เกิดขึ้นจริงเมื่อโปรแกรมทำงาน (เช่น พิมพ์ข้อความ, ปิดไฟล์, คืน
memory ให้ระบบ) ในทางตรงกันข้าม **lifetime annotation (`'a`) ไม่มีผลต่อ runtime เลยแม้แต่นิดเดียว** — มันถูก
"ลบออกไปหมด" หลัง compile เสร็จ (ตามที่ Part 20 หัวข้อ 20.3 อธิบายไว้ในตาราง) ลองพิสูจน์ให้เห็นจริงด้วยการเขียน
ฟังก์ชันสองเวอร์ชันที่ **เขียน lifetime ต่างกัน** (เวอร์ชันหนึ่งพึ่ง elision เต็มที่ อีกเวอร์ชันเขียน `'a` เต็ม
รูปแบบด้วยมือ) แล้วดูว่า**ลำดับการ `drop` จริงตอน runtime เปลี่ยนไปหรือไม่**:

```rust
// lifetimes are purely static analysis: adding/removing 'a annotations never changes runtime Drop order
struct Noisy(&'static str);

impl Drop for Noisy {
    fn drop(&mut self) {
        println!("drop: {}", self.0);
    }
}

// version A: fully elided (compiler infers 'a via elision rule 2 internally, invisible to us)
fn borrow_a(x: &Noisy) -> &str {
    x.0
}

// version B: written explicitly with 'a — 100% the same signature after elision expands, byte-for-byte
fn borrow_b<'a>(x: &'a Noisy) -> &'a str {
    x.0
}

fn main() {
    let n1 = Noisy("n1");
    println!("ก่อนเรียก borrow_a: {}", borrow_a(&n1));
    let n2 = Noisy("n2");
    println!("ก่อนเรียก borrow_b: {}", borrow_b(&n2));
    println!("จบ main — ลำดับ drop จะเป็น n2 ก่อน n1 เสมอ (LIFO ตาม scope) ไม่ว่า signature จะเขียน 'a หรือไม่");
}
```

ผลลัพธ์:

```
ก่อนเรียก borrow_a: n1
ก่อนเรียก borrow_b: n2
จบ main — ลำดับ drop จะเป็น n2 ก่อน n1 เสมอ (LIFO ตาม scope) ไม่ว่า signature จะเขียน 'a หรือไม่
drop: n2
drop: n1
```

**วิเคราะห์ผลลัพธ์นี้อย่างตรงประเด็นที่สุด**: `borrow_a` (พึ่ง elision เต็มที่ ไม่เห็น `'a` เลย) และ `borrow_b`
(เขียน `<'a>` เต็มรูปแบบด้วยมือ) **มีผลต่อลำดับการ `drop` ของ `n1` และ `n2` เหมือนกันเป๊ะ ๆ ไม่ต่างกันแม้แต่นิดเดียว**
— `n1` และ `n2` ถูก `drop` ตามลำดับ **LIFO (Last In, First Out) ตาม scope ของ `main`** เท่านั้น (สร้าง `n1` ก่อน
`n2` ก็ `drop` `n2` ก่อน `n1` เสมอ ตามกฎที่เรียนมาตั้งแต่ Part 6) — **ไม่ว่าเราจะเขียนหรือไม่เขียน `'a` เลย, ไม่ว่า
จะเขียน `'a` ชื่ออะไร, ไม่ว่าจะมี lifetime parameter กี่ตัวก็ตาม ลำดับการ `drop` จริงตอน runtime จะไม่เปลี่ยนแปลง
เลยแม้แต่นิดเดียว**

**ทำไมถึงเป็นแบบนี้เสมอ**: เพราะ **ตัวกำหนดว่าค่าไหนจะถูก `drop` เมื่อไหร่คือโครงสร้าง scope ของโค้ด** (ตัวแปร
ประกาศที่ไหน หมด scope ที่ไหน ตามกฎ ownership จาก Part 6) — เรื่องนี้ถูกกำหนดไว้แน่นอนแล้วตั้งแต่ตอนเขียนโค้ด
โดยไม่เกี่ยวอะไรกับ lifetime annotation เลย **lifetime annotation ทำหน้าที่แค่อย่างเดียว**: เป็น**ข้อมูล**ที่ส่ง
ให้ borrow checker ใช้**ตรวจสอบ** (compile time เท่านั้น) ว่าโครงสร้าง scope ที่มีอยู่แล้วนั้น**จะไม่ทำให้เกิด
dangling reference** — มันเป็น**ผู้ตรวจสอบ (verifier)** ไม่ใช่**ผู้ควบคุม (controller)** ของสิ่งที่เกิดขึ้นจริง
เปรียบเทียบให้เห็นภาพชัดที่สุด: **`Drop` คือ "คนงาน" ที่ลงมือทำงานจริงตอน runtime (เก็บกวาด, ปิดไฟล์) ส่วน `'a`
คือ "ผู้ตรวจสอบเอกสาร" ที่ทำงานเสร็จสิ้นไปแล้วตั้งแต่ก่อนโปรแกรมเริ่มรันด้วยซ้ำ (ตอน compile time) — เมื่อ
`cargo build` เสร็จ "ผู้ตรวจสอบเอกสาร" คนนี้ก็ออกจากตึกไปแล้ว ไม่มีบทบาทอะไรเหลืออยู่เลยตอนที่ "คนงาน" เริ่มทำงาน
จริง**

**ทดสอบเพิ่มเพื่อความมั่นใจ**: ถ้าคุณลองสลับเปลี่ยนชื่อ `'a` เป็น `'x`, ลบ `<'a>` ออกจาก `borrow_a` (มันก็ compile
ผ่านได้เหมือนกันเพราะ elision จัดการให้), หรือเพิ่ม lifetime parameter ที่ไม่ได้ใช้งานจริงเข้าไปอีกสิบตัว —
**ผลลัพธ์ตอนรันโปรแกรม (ลำดับข้อความ `drop: ...` ที่พิมพ์ออกมา) จะเหมือนกันทุกประการเสมอ** เพราะไม่มี syntax
เรื่อง lifetime ใด ๆ ที่มีอิทธิพลต่อ runtime behavior ได้เลยไม่ว่าในกรณีใด — มันคือข้อเท็จจริงที่ตรวจสอบได้เสมอ
ไม่ใช่แค่คำกล่าวลอย ๆ

### 23.9 Self-Referential Struct: ขอบเขตที่ Lifetime ธรรมดาไปไม่ถึง

ถึงจุดนี้ เราเห็นแล้วว่า lifetime (ร่วมกับ subtyping และ HRTB) แก้ปัญหาเรื่อง "reference ข้ามขอบเขตฟังก์ชัน/struct"
ได้อย่างครอบคลุมมาก — แต่มีปัญหาคลาสหนึ่งที่ **lifetime ระบบปัจจุบันของ Rust แก้ไม่ได้เลยด้วยวิธีธรรมดา**: struct
ที่ต้องการเก็บ **reference ไปยัง field อื่นของตัวมันเอง** (self-referential struct)

#### ลองเขียนดูตรง ๆ ก่อน แล้วดูว่าทำไมมันล้มเหลว

```rust
struct SelfRef<'a> {
    value: String,
    pointer_to_value: &'a str,
}

fn make() -> SelfRef<'static> {
    let value = String::from("hello");
    let pointer_to_value = &value; // ยืม value ไว้ก่อน
    SelfRef { value, pointer_to_value } // ❌ ต้อง move `value` เข้า struct ทั้ง ๆ ที่ยังถูกยืมอยู่
}

fn main() {
    let s = make();
    println!("{} {}", s.value, s.pointer_to_value);
}
```

Error (compiler รายงานถึงสองข้อพร้อมกัน เพราะเป็นปัญหาที่ซ้อนกันสองชั้น):

```
error[E0515]: cannot return value referencing local variable `value`
 --> src/main.rs:9:5
  |
8 |     let pointer_to_value = &value; // ยืม value ไว้ก่อน
  |                            ------ `value` is borrowed here
9 |     SelfRef { value, pointer_to_value } // ❌ ต้อง move `value` เข้า struct ทั้ง ๆ ที่ยังถูกยืมอยู่
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ returns a value referencing data owned by the current function

error[E0505]: cannot move out of `value` because it is borrowed
 --> src/main.rs:9:15
  |
7 |     let value = String::from("hello");
  |         ----- binding `value` declared here
8 |     let pointer_to_value = &value; // ยืม value ไว้ก่อน
  |                            ------ borrow of `value` occurs here
9 |     SelfRef { value, pointer_to_value } // ❌ ต้อง move `value` เข้า struct ทั้ง ๆ ที่ยังถูกยืมอยู่
  |     ----------^^^^^--------------------
  |     |         |
  |     |         move out of `value` occurs here
  |     returning this value requires that `value` is borrowed for `'static`
  |
help: consider cloning the value if the performance cost is acceptable
  |
8 |     let pointer_to_value = &value.clone(); // ยืม value ไว้ก่อน
  |                                  ++++++++
```

#### ทำไมปัญหานี้ "ยากกว่า" ปัญหาที่ lifetime ธรรมดาเคยแก้ได้

**อธิบายรากของปัญหาให้ชัดที่สุด**: `SelfRef` ต้องการให้ `pointer_to_value` ชี้ไปยัง**ตำแหน่งความจำที่แน่นอนของ
`value` field ที่อยู่ในโครงสร้าง `SelfRef` เดียวกัน** — ปัญหาคือ **`SelfRef` instance ทั้งก้อนสามารถถูก "ย้าย"
(move) ไปอยู่ที่ตำแหน่งความจำอื่นได้เสมอ** (การคืนค่าออกจากฟังก์ชันแบบในตัวอย่างนี้เป็นการ move ชัด ๆ, การใส่ลง
`Vec`, การส่งผ่านเป็น parameter แบบ by-value ทั้งหมดคือการ move) — เมื่อ `SelfRef` ทั้งก้อนถูก move **field
`value` (ซึ่งอยู่ข้างในของก้อนนั้น) ก็ย้ายตำแหน่งไปด้วย** แต่ `pointer_to_value` ที่จดจำ**ตำแหน่งเดิม**ไว้จะยังคง
ชี้ไปยังตำแหน่งเก่าที่อาจไม่มีข้อมูลที่ถูกต้องอยู่แล้ว (หรือในกรณีนี้คือถูก borrow checker ปฏิเสธไปตั้งแต่ก่อนจะ
ถึงจุดนั้นด้วยซ้ำ) — **นี่คือความแตกต่างเชิงพื้นฐานจากทุกตัวอย่างที่เห็นมาตลอด Part 20 และบทนี้**: ในทุกตัวอย่าง
ก่อนหน้า reference ชี้ไปยัง**ข้อมูลที่อยู่นอก struct/ค่าที่ตัวมันเองถูกย้าย** (เช่น `Excerpt<'a>` ชี้ไปยัง `novel`
ที่เป็นตัวแปรแยกต่างหาก) ดังนั้นแม้ `Excerpt` เองจะถูก move ไปไหนก็ตาม ที่อยู่ของ `novel` (ที่มันชี้ไปยัง) ก็ไม่
เปลี่ยนแปลง — แต่ `SelfRef` ต้องการชี้ไปยัง **ส่วนหนึ่งของตัวเอง** ซึ่งเปลี่ยนตำแหน่งไปพร้อมกับการ move ของทั้งก้อน
เสมอ

**ทำไม lifetime parameter (`'a`) แก้ปัญหานี้ไม่ได้เลยไม่ว่าจะเขียนอย่างไร**: ระบบ lifetime ของ Rust ถูกออกแบบมา
เพื่อพิสูจน์ความสัมพันธ์เรื่อง **"อายุ" (เวลาที่ข้อมูลยังมีชีวิตอยู่)** — แต่ปัญหาของ self-referential struct ไม่ใช่
เรื่องอายุเลย มันเป็นเรื่อง **"ตำแหน่งที่อยู่ในความจำ (memory address) ที่อาจเปลี่ยนได้เมื่อค่าถูก move"** ซึ่งเป็น
มิติที่ต่างไปจากที่ lifetime ระบบปัจจุบันครอบคลุม ไม่มี syntax `'a` ชนิดใดที่จะบอก compiler ว่า "ห้ามย้ายค่านี้ไป
ไหนเลยตลอดชีวิตของมัน" ได้ — นี่คือ**ขอบเขตที่แท้จริง**ของระบบ lifetime แบบที่เรียนมาตลอดทั้ง Part 20 และบทนี้

#### แนวทางที่โลกจริงใช้แก้ปัญหานี้ (ภาพรวมกว้าง ๆ ไม่ใช่บทเรียนเต็มรูปแบบ)

ปัญหา self-referential struct เป็นปัญหาที่รู้จักกันดีในวงการ Rust และมีทางแก้อยู่หลายแนวทาง แต่ **ไม่มีทางแก้ไหนที่
เป็นแค่การเขียน lifetime annotation ให้ถูกต้องเพียงอย่างเดียว** ทุกทางแก้ต้อง**เปลี่ยนวิธีคิดเรื่อง design** อย่าง
น้อยหนึ่งในสามแนวทางนี้:

1. **เปลี่ยนจาก reference เป็น index/handle**: แทนที่จะเก็บ `&'a str` ที่ชี้ไปยัง `value` โดยตรง ให้เก็บ `usize`
   (ตำแหน่ง index) แล้วค่อย slice ข้อมูลจริงตอนที่ต้องใช้งาน (เช่น `&self.value[self.start..self.end]`) — วิธีนี้
   ใช้ได้ผลดีมากในโค้ด parser จริง เพราะ index ไม่มีปัญหาเรื่อง "ตำแหน่งความจำเปลี่ยน" เลย (index ยังคงถูกต้องไม่ว่า
   struct จะถูก move ไปไหนก็ตาม)
2. **ใช้ shared ownership แทน reference**: `Rc<T>` หรือ `Arc<T>` (จะเรียนเต็มรูปแบบใน Part 28 และ Part 39) ทำให้
   หลายส่วนของโปรแกรม "ใช้ข้อมูลชุดเดียวกันร่วมกัน" ได้โดยไม่ต้องมี reference ที่ผูกกับ lifetime แบบตรง ๆ เลย —
   heap allocation ที่ `Rc`/`Arc` จัดการอยู่กับที่เสมอ ไม่ขยับไปไหนแม้ตัว smart pointer ที่ชี้ไปจะถูก move
3. **ใช้ `Pin` และ unsafe code**: สำหรับกรณีที่ต้องการ self-reference แบบตรงจริง ๆ (พบมากที่สุดในโค้ดเกี่ยวกับ
   `async`/`Future` ซึ่งคอมไพเลอร์สร้าง state machine ที่เป็น self-referential โดยธรรมชาติ) Rust มี `Pin<T>` เป็น
   เครื่องมือที่**การันตีว่าค่าจะไม่ถูก move ไปไหนอีก**หลังจากถูก pin ไว้แล้ว ทำให้ self-reference ปลอดภัยได้ในบาง
   สถานการณ์ที่ควบคุมอย่างระมัดระวัง — นี่เป็นเนื้อหาขั้นสูงมากที่ต้องใช้ `unsafe` เป็นส่วนใหญ่

**บทเรียนสำคัญที่สุดของหัวข้อนี้ไม่ใช่การท่องจำวิธีแก้ทั้งสามข้อ** (จะเรียนแบบละเอียดใน Part 27-29 เมื่อถึงเวลาที่
เหมาะสม) แต่คือการ**รู้จักปัญหาคลาสนี้ทันทีที่เจอ**: ถ้าคุณพยายามออกแบบ struct ที่ "field หนึ่งชี้ไปยัง field อื่น
ของตัวมันเอง" แล้วเจอ error ประเภท `E0505` (cannot move out because borrowed) หรือ `E0515` (cannot return value
referencing local variable) ที่**ไม่มีทางแก้ด้วยการเพิ่ม/ลด lifetime annotation ได้เลย** — นั่นเป็นสัญญาณที่ชัดเจน
ว่าคุณเจอปัญหาคลาส self-referential struct แล้ว และต้อง**เปลี่ยน design ทั้งหมด** (ไม่ใช่แค่ปรับ syntax) ตามหนึ่งใน
สามแนวทางข้างบน

### 23.10 ตัวอย่างจริง: เขียน Lexer/Tokenizer ที่ยืม Slice จาก Input String โดยไม่ Copy เลย

มาปิดท้ายบทนี้ด้วยตัวอย่างที่ผสมทุกแนวคิดสำคัญเข้าด้วยกัน: Part 8 (slices), Part 20 (lifetime พื้นฐานบน struct),
และแนวคิดของบทนี้ (การเลือกจำนวน lifetime parameter อย่างมีเหตุผล) — เขียน **lexer** (ตัวแยกข้อความต้นฉบับออกเป็น
"token" ย่อย ๆ ซึ่งเป็นขั้นตอนแรกของทุก parser/compiler จริง ไม่ว่าจะเป็น parser ของภาษาโปรแกรม, ตัวแยกวิเคราะห์
สูตรคำนวณ, หรือตัวแยกวิเคราะห์คำสั่ง SQL) สำหรับนิพจน์คณิตศาสตร์ง่าย ๆ เช่น `total = (price + tax) * 2`

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
enum TokenKind {
    Number,
    Ident,
    Plus,
    Minus,
    Star,
    Slash,
    Assign,
    LParen,
    RParen,
}

// Token<'a> เก็บ "ส่วนหนึ่งของข้อความต้นฉบับ" โดยตรง ไม่สร้าง String ใหม่เลยแม้แต่ตัวเดียว
// เลือกใช้ 'a ตัวเดียว เพราะ text ทุก token มาจากแหล่งเดียวกันเสมอ (input string ต้นฉบับของ Lexer)
// ตามหลักการตัดสินใจจาก Part 20 หัวข้อ 20.7: field ที่มาจากแหล่งเดียวกันเสมอ -> ใช้ 'a ร่วมกัน
#[derive(Debug)]
struct Token<'a> {
    text: &'a str,
    kind: TokenKind,
}

// Lexer<'a> เก็บ input (ยืมมา ไม่ copy) และตำแหน่งปัจจุบันที่กำลังอ่าน
struct Lexer<'a> {
    input: &'a str,
    pos: usize,
}

impl<'a> Lexer<'a> {
    fn new(input: &'a str) -> Self {
        Lexer { input, pos: 0 }
    }

    // rule 3 (elision) ทำงานที่นี่: &mut self ได้ lifetime ของตัวเอง แต่ output ผูกกับ 'a ของ struct
    // (ไม่ใช่ผูกกับ &mut self) เพราะเราเขียน Option<Token<'a>> อย่างชัดเจน ไม่ปล่อยให้ elision เดา
    fn skip_whitespace(&mut self) {
        let bytes = self.input.as_bytes();
        while self.pos < bytes.len() && bytes[self.pos].is_ascii_whitespace() {
            self.pos += 1;
        }
    }

    fn next_token(&mut self) -> Option<Token<'a>> {
        self.skip_whitespace();
        let bytes = self.input.as_bytes();
        if self.pos >= bytes.len() {
            return None;
        }
        let start = self.pos;
        let c = bytes[self.pos];
        let kind = match c {
            b'+' => { self.pos += 1; TokenKind::Plus }
            b'-' => { self.pos += 1; TokenKind::Minus }
            b'*' => { self.pos += 1; TokenKind::Star }
            b'/' => { self.pos += 1; TokenKind::Slash }
            b'=' => { self.pos += 1; TokenKind::Assign }
            b'(' => { self.pos += 1; TokenKind::LParen }
            b')' => { self.pos += 1; TokenKind::RParen }
            b'0'..=b'9' => {
                while self.pos < bytes.len() && bytes[self.pos].is_ascii_digit() {
                    self.pos += 1;
                }
                TokenKind::Number
            }
            c if c.is_ascii_alphabetic() => {
                while self.pos < bytes.len() && bytes[self.pos].is_ascii_alphanumeric() {
                    self.pos += 1;
                }
                TokenKind::Ident
            }
            other => panic!("unexpected character: {}", other as char),
        };
        // สำคัญที่สุด: &self.input[start..self.pos] คือการ "ยืม" ส่วนหนึ่งของ input ต้นฉบับตรง ๆ
        // ไม่มีการ allocate memory ใหม่ ไม่มีการ copy ตัวอักษรแม้แต่ตัวเดียว — ทั้ง Vec<Token<'a>> ที่ได้
        // ในตอนจบเป็นแค่ "รายการของหน้าต่างที่มองเข้าไปใน input เดียวกัน" ทั้งหมด
        Some(Token { text: &self.input[start..self.pos], kind })
    }

    // consume ตัว Lexer เอง (mut self แทน &mut self) เพื่อคืน Vec<Token<'a>> ทั้งหมดในครั้งเดียว
    fn tokenize(mut self) -> Vec<Token<'a>> {
        let mut tokens = Vec::new();
        while let Some(tok) = self.next_token() {
            tokens.push(tok);
        }
        tokens
    }
}

fn main() {
    let source = String::from("total = (price + tax) * 2");
    let lexer = Lexer::new(&source);
    let tokens = lexer.tokenize();

    for tok in &tokens {
        println!("{:?} {:?}", tok.kind, tok.text);
    }

    println!("จำนวน token ทั้งหมด: {}", tokens.len());
    // source ยังใช้งานได้ปกติทุกประการ เพราะ tokenize แค่ "ยืม" มัน ไม่ได้ทำลายหรือ move มันไปไหน
    println!("source ต้นฉบับยังใช้ได้ปกติ: {source}");
}
```

ผลลัพธ์:

```
Ident "total"
Assign "="
LParen "("
Ident "price"
Plus "+"
Ident "tax"
RParen ")"
Star "*"
Number "2"
จำนวน token ทั้งหมด: 9
source ต้นฉบับยังใช้ได้ปกติ: total = (price + tax) * 2
```

#### วิเคราะห์การออกแบบนี้ให้ลึกที่สุด: ทำไมนี่คือตัวอย่างที่รวมทุกอย่างในบทนี้ไว้ครบ

1. **`Token<'a>` และ `Lexer<'a>` ใช้ `'a` ตัวเดียวร่วมกัน**: เพราะทั้ง `Token` ที่คืนออกมาและ `input` ที่ `Lexer`
   ถือไว้ **มาจากแหล่งเดียวกันเสมอ** (ตัวแปร `source` ใน `main`) — ไม่มีเหตุผลใดที่จะแยกเป็น `'a`, `'b` ตามหลักการ
   จากหัวข้อ 23.2 และ Part 20 หัวข้อ 20.7 (ถ้าแยกโดยไม่จำเป็นจะเป็นการ over-engineer ตามที่กับดักข้อ 4 ท้ายบทจะ
   อธิบาย)
2. **`tokenize(mut self)` (ไม่ใช่ `&mut self`)**: การ consume `self` แบบ by-value ทำให้ `Lexer` **ไม่มีวันถูกใช้
   ต่อหลังจาก `tokenize()` แล้ว** — เป็นการออกแบบ API ที่สื่อความหมายชัดเจนผ่านระบบ ownership (Part 6) ทำงานร่วมกับ
   lifetime อีกครั้ง: `Lexer` ทำงานได้แค่ครั้งเดียวจบ ไม่มี state ค้างให้สับสน
3. **`Vec<Token<'a>>` ที่คืนออกมาไม่มี owned `String` แม้แต่ตัวเดียว**: ทุก `text` field ใน `Token` เป็นแค่
   "หน้าต่าง" ที่มองเข้าไปยัง memory เดียวกันกับ `source` — ถ้า input มีขนาดหลายร้อย MB (เช่น source code ของ
   โปรเจกต์ขนาดใหญ่จริง ๆ) การเขียนแบบนี้ประหยัด memory และเวลาได้มหาศาลเมื่อเทียบกับการ `.to_string()` ทุก token
   (ซึ่งจะต้อง allocate heap ใหม่หลายพันครั้งสำหรับไฟล์ขนาดใหญ่)
4. **ไม่มีปัญหา self-referential struct เลย**: สังเกตว่า `Token<'a>` และ `Lexer<'a>` **ไม่ได้ชี้กลับเข้าไปยัง
   ตัวเอง** — ทั้งคู่ชี้ไปยัง `source` ซึ่งเป็นตัวแปร**แยกต่างหาก**อยู่นอกทั้งสอง struct เสมอ (แบบเดียวกับ `Excerpt`
   ในหัวข้อ 23.9 ที่ไม่มีปัญหา) — นี่คือความแตกต่างเชิงโครงสร้างที่สำคัญที่สุดระหว่างการออกแบบที่ "ยืมข้อมูลจากภายนอก
   อย่างถูกต้อง" (ปลอดภัย, ทำได้ด้วย lifetime ธรรมดา) กับ "ยืมข้อมูลจากตัวเอง" (ปัญหาคลาส self-referential ที่แก้
   ไม่ได้ด้วย lifetime ธรรมดา)

โครงสร้างแบบ `Lexer<'a>` → `Vec<Token<'a>>` นี้คือรูปแบบที่พบได้จริงในโค้ด production จำนวนมาก ตั้งแต่ parser ของ
ภาษาโปรแกรมจริง (เช่น `rustc` เองก็ใช้แนวคิดนี้กับ token stream ของมัน) ไปจนถึง library แยกวิเคราะห์ format ข้อมูล
ต่าง ๆ (JSON, TOML, CSV) ที่ใส่ใจเรื่อง performance — เป็นตัวอย่างที่ยืนยันว่า **การเข้าใจ lifetime ในระดับลึกไม่ใช่
แค่ "ทฤษฎีที่ทำให้ compiler หยุดบ่น" แต่เป็นเครื่องมือที่ทำให้ออกแบบโค้ดที่เร็วและประหยัด memory ได้จริงในสถานการณ์
ที่ performance สำคัญ**

## กับดักที่พบบ่อย (Common Pitfalls)

**1. ลืมเขียน `+ 'a` ให้ `dyn Trait` เมื่อ Trait Object เก็บ Reference ที่ไม่ใช่ `'static`**

```rust
trait Greet {
    fn greet(&self) -> String;
}

struct Formal<'a> {
    name: &'a str,
}

impl<'a> Greet for Formal<'a> {
    fn greet(&self) -> String {
        format!("สวัสดีครับ คุณ {}", self.name)
    }
}

// ❌ ไม่เขียน + 'a ให้ dyn Greet -> default กลายเป็น dyn Greet + 'static
fn make_greeter<'a>(name: &'a str) -> Box<dyn Greet> {
    Box::new(Formal { name })
}
```

```
error: lifetime may not live long enough
  --> src/main.rs:17:5
   |
16 | fn make_greeter<'a>(name: &'a str) -> Box<dyn Greet> {
   |                 -- lifetime `'a` defined here
17 |     Box::new(Formal { name })
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^ returning this value requires that `'a` must outlive `'static`
```

**วิธีแก้**: เขียน `Box<dyn Greet + 'a>` ตามที่อธิบายในหัวข้อ 23.5 — จำไว้เสมอว่า `dyn Trait` ที่ไม่เขียน lifetime
จะถูกตีความเป็น `+ 'static` โดย default ในบริบทส่วนใหญ่ ทันทีที่ type ที่ใส่ลง trait object นั้นมี field เป็น
reference ที่ไม่ใช่ `'static` ต้องเขียน `+ 'a` (หรือ lifetime ที่ตรงกับความจริง) อย่างชัดเจนเสมอ

**2. ผูก Closure Bound ด้วย Lifetime ของ Struct โดยตรง ทั้งที่ควรใช้ HRTB (`for<'s>`)**

มือใหม่ที่เขียน struct เก็บ closure ที่ต้องรับ `&str` มักผูก lifetime ของ parameter ของ closure ไว้กับ lifetime
parameter ของ struct เองแบบตรง ๆ โดยไม่รู้ตัวว่ากำลังจำกัดสิทธิ์ของ closure นั้นเกินความจำเป็น:

```rust
// ❌ ผูก lifetime ของ predicate ไว้กับ 'a ของ struct เอง — พอสร้าง Matcher เสร็จ 'a ก็ถูก "ตรึง" ไว้แล้ว
struct Matcher<'a, F: Fn(&'a str) -> bool> {
    predicate: F,
    _marker: std::marker::PhantomData<&'a ()>,
}

impl<'a, F: Fn(&'a str) -> bool> Matcher<'a, F> {
    fn new(predicate: F) -> Self {
        Matcher { predicate, _marker: std::marker::PhantomData }
    }

    fn check(&self, input: &'a str) -> bool {
        (self.predicate)(input)
    }
}

fn main() {
    let long_lived = String::from("hello world");
    let matcher = Matcher::new(|s: &str| s.contains("hello"));

    {
        let short_lived = String::from("hello there");
        // ต้องการเรียก matcher.check ด้วย reference ที่มีอายุสั้น แต่ 'a ของ matcher ถูกผูกไว้แล้วตอนสร้าง
        println!("{}", matcher.check(&short_lived));
    }

    println!("{}", matcher.check(&long_lived));
}
```

```
error[E0597]: `short_lived` does not live long enough
  --> src/main.rs:25:38
   |
23 |         let short_lived = String::from("hello there");
   |             ----------- binding `short_lived` declared here
25 |         println!("{}", matcher.check(&short_lived));
   |                                      ^^^^^^^^^^^^ borrowed value does not live long enough
26 |     }
   |     - `short_lived` dropped here while still borrowed
```

**วิธีแก้**: ใช้ **HRTB (`for<'s>`)** ตามหลักการจากหัวข้อ 23.6 เพื่อให้ `predicate` รับ reference ที่มีอายุอะไรก็ได้
ที่กำหนดใหม่ทุกครั้งที่ถูกเรียก แทนที่จะผูกกับ `'a` ตัวเดียวตายตัว:

```rust
// ✅ predicate รับ &str ที่มีอายุอะไรก็ได้ (for<'s> ถูก infer อัตโนมัติจากการไม่เอ่ยชื่อ lifetime)
// Matcher ไม่ต้องมี lifetime parameter เลยด้วยซ้ำ เพราะ predicate ไม่ได้ผูกกับข้อมูลที่ยืมมาจากที่ไหนอีกต่อไป
struct Matcher<F: for<'s> Fn(&'s str) -> bool> {
    predicate: F,
}

impl<F: for<'s> Fn(&'s str) -> bool> Matcher<F> {
    fn new(predicate: F) -> Self {
        Matcher { predicate }
    }

    fn check(&self, input: &str) -> bool {
        (self.predicate)(input)
    }
}

fn main() {
    let long_lived = String::from("hello world");
    let matcher = Matcher::new(|s: &str| s.contains("hello"));

    {
        let short_lived = String::from("hello there");
        println!("{}", matcher.check(&short_lived)); // ✅ ใช้ได้แล้ว
    }

    println!("{}", matcher.check(&long_lived));
}
```

**บทเรียนสำคัญ**: ทุกครั้งที่ออกแบบ struct/ฟังก์ชันที่เก็บ closure ซึ่งควรใช้งานได้กับ reference **ใหม่ทุกครั้งที่
เรียก** (ไม่ใช่ reference ที่ผูกกับ instance เดียวตายตัว) ให้เขียน closure bound แบบไม่เอ่ยชื่อ lifetime เลย (ปล่อย
ให้กลายเป็น HRTB โดยอัตโนมัติ) แทนการผูกมันเข้ากับ lifetime parameter ของ struct/ฟังก์ชันตรง ๆ

**3. ลืมเขียน `where 'a: 'b` เมื่อพยายาม "แปลง" Lifetime ยาวให้กลายเป็นสั้นผ่าน Generic Wrapper**

```rust
struct Ref<'a, T: ?Sized>(&'a T);

// ❌ ลืมเขียน where 'a: 'b — compiler ไม่รู้ว่า 'a ยาวกว่า 'b จริงหรือไม่
fn extend_scope<'a, 'b, T>(r: Ref<'a, T>) -> Ref<'b, T> {
    Ref(r.0)
}
```

```
error: lifetime may not live long enough
 --> src/main.rs:4:5
  |
3 | fn extend_scope<'a, 'b, T>(r: Ref<'a, T>) -> Ref<'b, T> {
  |                 --  -- lifetime `'b` defined here
  |                 |
  |                 lifetime `'a` defined here
4 |     Ref(r.0)
  |     ^^^^^^^^ function was supposed to return data with lifetime `'b` but it is returning data with lifetime `'a`
  |
  = help: consider adding the following bound: `'a: 'b`
```

**วิธีแก้**: เพิ่ม `where 'a: 'b` ตามที่อธิบายเต็มรูปแบบในหัวข้อ 23.3-23.4 — เมื่อสองก) lifetime parameter ต้องมี
ความสัมพันธ์เชิง subtyping กัน (ตัวหนึ่งต้องยืนยาวไม่น้อยกว่าอีกตัว) ให้เขียนความสัมพันธ์นั้นด้วย outlives bound
เสมอ อย่าคาดหวังว่า compiler จะเดาเองได้โดยไม่มีข้อมูลนี้

**4. แยก Lifetime Parameter มากเกินความจำเป็น (Over-Engineering) จนทำให้ API ใช้งานยากขึ้นโดยไม่ได้ประโยชน์จริง**

กับดักนี้เป็นด้านตรงข้ามของกับดักข้อ 2 ใน Part 20 (ที่ผูก `'a` เดียวกันเกินจำเป็น) — คราวนี้คือการ**แยก** lifetime
parameter ออกจากกันทั้ง ๆ ที่ field ทั้งหมดมาจากแหล่งเดียวกันเสมอในทางความหมาย:

```rust
// ❌ แยก 'a และ 'b โดยไม่มีเหตุผลเชิงความหมายเลย — key และ value มาจากการ split บรรทัดเดียวกันเสมอ
// (เทียบกับ KeyValue<'a> ตัวเดียวจาก Part 20 หัวข้อ 20.12 ที่ถูกต้องกว่ามาก)
struct KeyValueOverSplit<'a, 'b> {
    key: &'a str,
    value: &'b str,
}

fn parse_line_over_split(line: &str) -> Option<KeyValueOverSplit<'_, '_>> {
    let mut parts = line.splitn(2, '=');
    let key = parts.next()?.trim();
    let value = parts.next()?.trim();
    Some(KeyValueOverSplit { key, value })
}
```

โค้ดนี้ **compile ผ่านได้จริง** (ไม่ใช่ compiler error) แต่เป็นปัญหาเชิง **design/ergonomics**: ทุกครั้งที่ต้องส่ง
`KeyValueOverSplit` ผ่านฟังก์ชันอื่น ต้องเขียน lifetime parameter สองตัวตลอด (`KeyValueOverSplit<'_, '_>` หรือแย่กว่า
คือต้องตั้งชื่อ `'a`, `'b` ให้ตรงกันทุกจุดที่ใช้งาน) ทั้ง ๆ ที่ในทางความหมายจริง `key` และ `value` ไม่มีวันมีอายุ
ต่างกันเลย (ทั้งคู่มาจากการ `.splitn()` บรรทัดเดียวกันเสมอ) — ความซับซ้อนที่เพิ่มมาไม่ได้แลกกับความยืดหยุ่นอะไรเพิ่ม
ขึ้นจริงเลย

**วิธีแก้**: กลับไปใช้ `'a` ตัวเดียวตามที่ Part 20 หัวข้อ 20.12 สอนไว้ (`KeyValue<'a>`) — **ก่อนแยก lifetime
parameter ให้ถามตัวเองก่อนเสมอว่า "field เหล่านี้มีสถานการณ์จริงที่อายุจะต่างกันหรือไม่" ถ้าคำตอบคือไม่มีทางเป็นไป
ได้เลยในทางความหมาย ให้ใช้ `'a` ตัวเดียวเสมอ** — จำนวน lifetime parameter ที่เหมาะสมคือจำนวนที่**น้อยที่สุด**ที่ยัง
สื่อความหมายได้ถูกต้อง ไม่ใช่จำนวนที่มากที่สุดที่เป็นไปได้

**5. คิดว่า Self-Referential Struct แก้ได้ด้วยการเพิ่ม/ปรับ Lifetime Annotation ให้ "ฉลาดขึ้น"**

หลังจากเจอ error ของ self-referential struct (หัวข้อ 23.9) มือใหม่จำนวนมากจะพยายามแก้ด้วยการลองสารพัดวิธีเขียน
`'a` ใหม่ — เปลี่ยนชื่อ, เพิ่ม lifetime parameter อีกตัว, ลองใส่ `'static`, ลองสลับตำแหน่ง field — **ไม่มีวิธีไหน
ในกลุ่มนี้แก้ปัญหาได้เลย** เพราะรากของปัญหาไม่ใช่เรื่อง syntax lifetime แต่เป็นเรื่อง **"ตำแหน่งความจำที่เปลี่ยนได้
เมื่อค่าถูก move"** ตามที่อธิบายละเอียดในหัวข้อ 23.9 — ไม่ว่าจะลองปรับ `'a` แบบไหนกับตัวอย่าง `SelfRef` ในหัวข้อนั้น
ก็ตาม ผลลัพธ์จะยังเป็น `E0505`/`E0515` เดิมเสมอ เพราะปัญหาอยู่ที่**การ move ตัว `value` ในขณะที่ยังถูก borrow อยู่**
ซึ่งเป็นกฎ ownership พื้นฐานจาก Part 6 ไม่ใช่กฎเรื่อง lifetime เลย

**วิธีแก้ที่ถูกต้อง**: จำสัญญาณให้ได้ทันทีว่านี่คือปัญหาคลาส self-referential struct (field ที่ต้องการชี้กลับเข้าไป
ยังอีก field ของ struct เดียวกัน) แล้ว**เปลี่ยน design** ตามหนึ่งในสามแนวทางจากหัวข้อ 23.9 (ใช้ index แทน reference,
ใช้ `Rc`/`Arc`, หรือใช้ `Pin` กับ `unsafe` สำหรับกรณีขั้นสูงจริง ๆ) — การพยายามแก้ด้วย syntax lifetime ล้วน ๆ คือ
การเสียเวลาไปกับทิศทางที่ไม่มีวันไปถึงเป้าหมายได้เลย

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** โค้ดต่อไปนี้มี reference parameter สามตัว แต่ output ผูกกับตัวเดียวจริง ๆ (`base`) เขียน signature ที่
   ถูกต้องให้ครบถ้วน (ใช้ lifetime parameter ให้เหมาะสมกับความสัมพันธ์จริง ไม่ใช่ผูกทุกตัวเป็น `'a` เดียวกัน):
   ```rust
   fn pick_base(base: &str, unit_hint: &str, currency_symbol: &str) -> &str {
       println!("หน่วย: {unit_hint}, สัญลักษณ์เงิน: {currency_symbol}");
       base.trim()
   }

   fn main() {
       let price = String::from("  299.00  ");
       let result;
       {
           let hint = String::from("บาท");
           let symbol = String::from("฿");
           result = pick_base(&price, &hint, &symbol);
       }
       println!("ราคา: {result}");
   }
   ```
   (hint: `base` คือแหล่งเดียวที่ output มาจากได้จริง ส่วน `unit_hint` และ `currency_symbol` ถูกใช้แค่ `println!`
   เท่านั้น ไม่เคยหลุดออกไปเป็นส่วนของ output เลย — ใช้หลักการ "ตามรอย" จากหัวข้อ 23.2)

   เฉลยแบบย่อ:
   ```rust
   fn pick_base<'a, 'b, 'c>(base: &'a str, unit_hint: &'b str, currency_symbol: &'c str) -> &'a str {
       println!("หน่วย: {unit_hint}, สัญลักษณ์เงิน: {currency_symbol}");
       base.trim()
   }
   ```
   (หรือปล่อยให้ elision จัดการเองก็ได้ เพราะ `base` เป็น input lifetime ที่ output ผูกได้ แต่ในกรณีนี้ elision
   กฎข้อที่ 1/2 ใช้ไม่ได้เพราะมี input lifetime มากกว่าหนึ่งตัว จึงต้องเขียน `'a` ให้ `base` และ output อย่างชัดเจน
   ส่วน `unit_hint`, `currency_symbol` ปล่อยให้แต่ละตัวได้ lifetime ของตัวเองผ่านกฎข้อที่ 1 โดยไม่ต้องเอ่ยชื่อเลยก็ได้)

2. **[ง่าย-กลาง]** โค้ดต่อไปนี้พยายามคืน `Box<dyn Trait>` ที่เก็บ reference แต่ compile ไม่ผ่าน หาสาเหตุแล้วแก้ไข
   (ห้ามเปลี่ยน struct `Loud<'a>` หรือ logic ข้างในเลย แก้ได้แค่ signature ของ `make_loud`):
   ```rust
   trait Shout {
       fn shout(&self) -> String;
   }

   struct Loud<'a> {
       message: &'a str,
   }

   impl<'a> Shout for Loud<'a> {
       fn shout(&self) -> String {
           self.message.to_uppercase()
       }
   }

   fn make_loud<'a>(message: &'a str) -> Box<dyn Shout> {
       Box::new(Loud { message })
   }

   fn main() {
       let text = String::from("hello");
       let shouter = make_loud(&text);
       println!("{}", shouter.shout());
   }
   ```
   (hint: `dyn Shout` ที่ไม่เขียน lifetime จะถูกตีความเป็น `dyn Shout + 'static` — แต่ `Loud<'a>` เก็บ reference
   ที่ไม่ใช่ `'static` ทำตามหลักการหัวข้อ 23.5)

   เฉลยแบบย่อ:
   ```rust
   fn make_loud<'a>(message: &'a str) -> Box<dyn Shout + 'a> {
       Box::new(Loud { message })
   }
   ```

3. **[กลาง]** อธิบายเป็นข้อความ (4-6 บรรทัด) ว่าทำไมโค้ดต่อไปนี้ compile ไม่ผ่านด้วย error เกี่ยวกับ `'a` และ `'b`
   จากนั้นแก้ไขให้ compile ผ่านด้วยการเพิ่ม `where` clause เพียงบรรทัดเดียว (ห้ามเปลี่ยน signature หรือ logic
   ส่วนอื่นเลย):
   ```rust
   struct Slot<'a, T: ?Sized>(&'a T);

   fn narrow<'a, 'b, T>(slot: Slot<'a, T>) -> Slot<'b, T> {
       Slot(slot.0)
   }

   fn main() {
       let data = String::from("ข้อมูลตัวอย่าง");
       let wide: Slot<'_, String> = Slot(&data);
       let tight: Slot<'_, String> = narrow(wide);
       println!("{}", tight.0);
   }
   ```
   (hint: เหมือนตัวอย่าง `extend_scope` ในหัวข้อ 23.4 เป๊ะ ๆ — compiler ต้องรู้ว่า `'a` (ของ `slot` ที่รับเข้ามา)
   ยืนยาวไม่น้อยกว่า `'b` (ของ `Slot` ที่จะคืนออกไป) ก่อนจึงจะพิสูจน์ความปลอดภัยของการแปลงนี้ได้)

   เฉลยแบบย่อ:
   ```rust
   fn narrow<'a, 'b, T>(slot: Slot<'a, T>) -> Slot<'b, T>
   where
       'a: 'b,
   {
       Slot(slot.0)
   }
   ```

4. **[ยาก/ประยุกต์]** ขยายตัวอย่าง lexer จากหัวข้อ 23.10 ให้เพิ่ม method `fn count_kind(&self, tokens: &[Token<'a>],
   kind: TokenKind) -> usize` ที่นับจำนวน token ที่มี `kind` ตรงกับที่ระบุ จากนั้นเขียนฟังก์ชันแยกอีกตัวชื่อ
   `fn longest_ident<'t>(tokens: &'t [Token<'_>]) -> Option<&'t str>` ที่หา token ประเภท `Ident` ที่มีความยาวมาก
   ที่สุดใน slice ของ token แล้วคืน `text` ของมันออกมา (สังเกตว่าฟังก์ชันนี้รับ `&[Token<'_>]` — ใช้ anonymous
   lifetime `'_` สองครั้งในตำแหน่งต่างกัน) เขียน `main` ที่ทดสอบทั้งสองฟังก์ชันกับ token ที่ได้จากการ `tokenize()`
   ตัวอย่างในหัวข้อ 23.10 แล้วตอบคำถามสั้น ๆ (4-6 บรรทัด): **ทำไม `longest_ident` ต้องมี lifetime parameter ของ
   ตัวเอง (`'t`) แยกจาก lifetime ที่ผูกกับข้อความต้นฉบับภายใน `Token`?**
   (hint: คำตอบเกี่ยวกับ**สอง lifetime ที่ปรากฏใน `&'t [Token<'_>]`** — `'t` คือ lifetime ของการยืม **slice**
   (การยืม `&[...]` ครั้งนี้) ในขณะที่ `'_` ที่ซ่อนอยู่ใน `Token<'_>` คือ lifetime ของ**ข้อความต้นฉบับ**ที่แต่ละ
   `Token` ยืมมา — ทั้งสองเป็นอิสระจากกันโดยสิ้นเชิง: คุณยืม slice ของ token เพียงชั่วคราว (เพื่อวนอ่าน) โดยไม่ต้อง
   ยืมข้อความต้นฉบับเพิ่มอีกครั้งเลย และค่าที่คืนออกมา (`&'t str`) ต้องผูกกับ `'t` เพราะ `text` ที่อยู่ข้างใน แต่ละ
   `Token` มีอายุยืนไม่น้อยกว่า slice ที่ห่อหุ้มมันอยู่เสมอ — ใช้หลักการ subtyping จากหัวข้อ 23.3 ในการอธิบาย)

   โครงเริ่มต้น (ไม่ใช่เฉลยเต็ม แค่ช่วยตั้งต้น — ใช้ประกาศ `Token`/`Lexer`/`TokenKind` จากหัวข้อ 23.10):
   ```rust
   fn longest_ident<'t>(tokens: &'t [Token<'_>]) -> Option<&'t str> {
       tokens
           .iter()
           .filter(|t| t.kind == TokenKind::Ident)
           .map(|t| t.text)
           .max_by_key(|text| text.len())
   }

   fn count_kind(tokens: &[Token<'_>], kind: TokenKind) -> usize {
       tokens.iter().filter(|t| t.kind == kind).count()
   }
   ```

## สรุป

บทนี้พาเราไปเจาะลึกกรณีขั้นสูงของ lifetime ที่ Part 20 ยังไม่ครอบคลุม — ทวนภาพรวมทั้งหมดที่เรียนไปในบทนี้:

- **หัวข้อ 23.2**: ฟังก์ชันที่มี lifetime parameter หลายตัวไม่จำเป็นต้อง**สัมพันธ์กัน**เสมอไป — หลักการ "ตามรอย" ว่า
  output มาจาก parameter ตัวไหนได้บ้าง คือเครื่องมือที่แม่นยำที่สุดในการตัดสินใจว่าควรผูก `'a` ร่วมกันหรือแยกออกจาก
  กัน
- **หัวข้อ 23.3-23.4**: **Lifetime subtyping** (`'long: 'short`) ทำให้ reference ที่มีอายุยืนกว่าใช้แทนที่อายุสั้น
  กว่าได้เสมอ — และ **outlives bound** (`where 'a: 'b`) คือ syntax ที่ใช้บอก compiler ถึงความสัมพันธ์นี้เมื่อ
  elision และการอนุมานปกติไม่พอ โดยเฉพาะเมื่อต้อง "แปลง" lifetime ผ่าน generic wrapper
- **หัวข้อ 23.5**: `dyn Trait` มี lifetime bound แฝงอยู่เสมอ (`+ 'static` โดย default) ต้องเขียน `+ 'a` อย่างชัดเจน
  เมื่อ trait object เก็บข้อมูลที่ยืมมาแบบไม่ใช่ `'static`
- **หัวข้อ 23.6**: **Higher-Ranked Trait Bounds** (`for<'a>`) แก้ปัญหาที่ lifetime ธรรมดาแก้ไม่ได้ — เมื่อ closure
  ต้องรับ reference ที่อายุถูกกำหนดใหม่ทุกครั้งที่ถูกเรียก ไม่ใช่ตายตัวจากจุดประกาศ และในกรณีส่วนใหญ่ compiler
  สร้าง `for<'a>` ให้อัตโนมัติโดยไม่ต้องเขียนเองเลย
- **หัวข้อ 23.7**: trait สามารถมี lifetime parameter เป็นของตัวเอง (`trait Parser<'a>`) แยกจากกรณีที่ method แค่คืน
  reference ผูกกับ `&self` ธรรมดา (กฎ elision ข้อที่ 3 ที่นำมาใช้ในบริบท trait)
- **หัวข้อ 23.8**: ยืนยันด้วยโค้ดที่รันจริงว่า **lifetime annotation ไม่มีผลต่อลำดับการ `drop` ตอน runtime เลย**
  มันเป็นเครื่องมือพิสูจน์ทาง compile-time เท่านั้น ต่างจาก `Drop` (Part 6) ที่เป็นกลไก runtime แท้จริง
- **หัวข้อ 23.9**: **self-referential struct** คือขอบเขตที่ lifetime ระบบปัจจุบันของ Rust ไปไม่ถึง เพราะปัญหาอยู่ที่
  ตำแหน่งความจำที่เปลี่ยนได้เมื่อค่าถูก move ไม่ใช่เรื่องอายุ — ทางแก้ต้องเปลี่ยน design (index, `Rc`/`Arc`, หรือ
  `Pin`+`unsafe`) ไม่ใช่แค่ปรับ syntax lifetime
- **หัวข้อ 23.10**: ตัวอย่าง lexer/tokenizer แสดงให้เห็นว่าความเข้าใจ lifetime ระดับลึกนำไปสู่การออกแบบโค้ดที่เร็ว
  และประหยัด memory ได้จริงในสถานการณ์งานจริง ไม่ใช่แค่ทฤษฎีที่ทำให้ compiler หยุดบ่น

**ข้อคิดที่ควรติดตัวไปตลอด**: ทุกหัวข้อในบทนี้ — ตั้งแต่ HRTB ที่ดูซับซ้อนที่สุด ไปจนถึง self-referential struct ที่
lifetime แก้ไม่ได้เลย — ล้วนเป็น**ผลสืบเนื่อง**จากหลักการเดียวที่เรียนมาตั้งแต่ Part 6-7: **ownership กำหนดว่าใครเป็น
เจ้าของและเมื่อไหร่ถูกทำลาย, borrowing กำหนดว่าใครมีสิทธิ์เข้าถึงชั่วคราวแบบไหน, lifetime พิสูจน์ว่าการยืมทั้งหมดจะ
ไม่มีวันชี้ไปยังข้อมูลที่ถูกทำลายไปแล้ว** — บทนี้ไม่ได้เพิ่ม "กฎใหม่" ให้ borrow checker เลยแม้แต่ข้อเดียว มันแค่แสดง
ให้เห็นว่ากฎเดิมเหล่านั้นทำงานได้ครอบคลุมและลึกซึ้งกว่าที่ Part 20 มีเวลาแสดงให้เห็นแค่ไหน — และเมื่อไปถึงขอบเขตที่
มันทำงานไม่ได้จริง ๆ (self-referential struct) ก็ยังบอกเราได้อย่างตรงไปตรงมาว่าปัญหาคืออะไร ไม่ใช่การซ่อนบั๊กไว้
เงียบ ๆ แบบภาษาที่ไม่มีการตรวจสอบเรื่องนี้เลย

จาก **Part 24** เป็นต้นไป เราจะเปลี่ยนโฟกัสไปที่ **Closures** (`Fn`, `FnMut`, `FnOnce`, การ capture ตัวแปร) — ซึ่ง
คุณได้เห็นตัวอย่างเล็ก ๆ มาแล้วในหัวข้อ 23.6 ของบทนี้ (closure ที่ผ่านเข้า `apply_to_str` และ `Matcher`) แต่ยังไม่ได้
เรียนเรื่อง **การ capture ตัวแปรจาก environment** (ที่ทำให้ closure ต่างจากฟังก์ชันธรรมดาโดยพื้นฐาน) และความแตกต่าง
ระหว่าง trait ทั้งสามตัวอย่างละเอียด — ความเข้าใจเรื่อง lifetime และ HRTB จากบทนี้จะเป็นพื้นฐานสำคัญที่ทำให้เข้าใจ
ว่าทำไม `Fn`, `FnMut`, `FnOnce` ต้องถูกออกแบบมาในรูปแบบที่เป็นอยู่ และทำไม closure ที่ capture reference มักมีปัญหา
เรื่อง lifetime ที่ซับซ้อนกว่าฟังก์ชันธรรมดาที่เราคุ้นเคยมาตลอด

---

**Part ก่อนหน้า:** [Generics ขั้นสูง](part-022-generics-advanced.md) | **Part ถัดไป:** [Closures](part-024-closures.md)
