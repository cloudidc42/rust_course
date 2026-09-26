# Part 27: Smart Pointers: Box<T>

> โมดูล: ระดับกลาง (Intermediate) | ระดับ: กลาง | เวลาโดยประมาณ: 150 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้อย่างถูกต้องว่า **smart pointer** คืออะไรในทางเทคนิค (struct ที่ implement `Deref` และมักจะ implement
  `Drop` ด้วย) และแยกความแตกต่างจาก **reference ธรรมดา** (`&T` จาก Part 7) ได้ชัดเจนในมุม **ownership**: reference
  แค่ "ยืม" ข้อมูล ส่วน smart pointer อย่าง `Box<T>` **เป็นเจ้าของ** ข้อมูลที่มันชี้ไป
- อธิบายปัญหาต้นตอที่ `Box<T>` เกิดมาแก้ได้อย่างถ่องแท้: **recursive type ที่มีขนาดไม่จำกัด (infinite size)** อ่าน
  error จริง `E0072` ได้ และรู้ว่าทำไม `Box<T>` (ซึ่งมีขนาดคงที่เสมอ) คือทางแก้ที่ตรงจุดที่สุด
- ใช้ `Box::new()` เพื่อจัดสรรข้อมูลบน heap, ใช้ `*box_value` เพื่อ dereference ผ่าน trait `Deref`, และเข้าใจกลไก
  **auto-deref** ที่ทำให้เรียก method บน `Box<T>` ได้โดยไม่ต้องเขียน `*` เอง (เชื่อมโยงกับ auto-deref ของ `&String`
  ไปเป็น `&str` ที่เคยเห็นใน Part 8)
- อธิบายกลไกจริงของ trait `Deref`/`DerefMut` ได้ (ไม่ใช่แค่ "มันแปลงให้อัตโนมัติแบบมายากล") พร้อมอ่าน signature จริง
  ของ `fn deref(&self) -> &Self::Target` ได้
- ใช้กรอบการตัดสินใจสามข้อเพื่อเลือกได้อย่างมีเหตุผลว่า **เมื่อไหร่ควรใช้ `Box<T>`** จริง ๆ ในโค้ดของตัวเอง ไม่ใช่ใช้
  แบบสุ่มหรือตามความเคยชิน
- อธิบายว่า `Drop` ทำงานกับ `Box<T>` อย่างไร (ทำลายทั้ง pointer และข้อมูลบน heap โดยอัตโนมัติ) และเปรียบเทียบ
  `Box<T>` กับ raw pointer และ C++ `std::unique_ptr<T>` ได้อย่างถูกต้องสำหรับผู้มีพื้นฐาน C/C++
- สร้างโครงสร้างข้อมูลแบบ recursive จริง (cons list / expression tree) ด้วย `Box<T>` ได้ และรู้จักใช้ `Box<T>` เพื่อ
  ย้ายข้อมูลขนาดใหญ่อย่างประหยัดโดยไม่ต้อง copy ข้อมูลทั้งก้อนบน stack

## ความรู้ที่ต้องมีมาก่อน

- **Part 6 (Ownership เบื้องต้น) — จำเป็นที่สุดสำหรับบทนี้**: บทนี้ยืนอยู่บนแนวคิด **ownership** และ **move
  semantics** จาก Part 6 โดยตรง `Box<T>` คือค่าที่ **เป็นเจ้าของ** ข้อมูลบน heap เหมือนที่ `String`/`Vec<T>` เป็น
  เจ้าของข้อมูลบน heap ของตัวเอง — กฎ "ค่าหนึ่งค่ามีเจ้าของได้แค่หนึ่งตัวในเวลาเดียว" และ "เมื่อเจ้าของหลุด scope ค่า
  จะถูก `drop`" ใช้กับ `Box<T>` ทุกประการ ไม่มีข้อยกเว้น ถ้าจำหัวข้อ **`Drop` trait** ใน Part 6 ไม่ได้ ควรย้อนไปทวน
  ก่อน เพราะหัวข้อ 27.5 ของบทนี้จะพึ่งพาความเข้าใจนั้นเต็มที่
- **Part 7 (Borrowing และ References) — จำเป็นที่สุดสำหรับบทนี้**: บทนี้เปรียบเทียบ `Box<T>` กับ `&T` ตลอดทั้งบท
  คุณต้องแยกให้ออกว่า `&T` (reference) เป็นแค่ "ที่อยู่ชั่วคราวที่ยืมมา ไม่มีสิทธิ์ทำลายข้อมูล" ในขณะที่ `Box<T>`
  เป็น "เจ้าของข้อมูลจริง มีสิทธิ์และหน้าที่ทำลายข้อมูลเมื่อหมดอายุ" — ถ้าความแตกต่างนี้ยังไม่แน่น บทนี้จะสับสนได้ง่าย
- **Part 21 (Traits ขั้นสูง)**: คุณใช้ `Box<dyn Trait>` มาแล้วตั้งแต่ Part 21 (และ `Box<dyn Error>` ตั้งแต่ Part 12)
  ในบริบทเฉพาะของ **dynamic dispatch / trait object** เท่านั้น — บทนี้จะ**ถอยกลับมาสอน `Box<T>` เองแบบเป็นทางการ**
  ในฐานะแนวคิดที่กว้างกว่า: `Box<dyn Trait>` เป็นแค่ **หนึ่งกรณีการใช้งาน** ของ `Box<T>` (กรณีที่ `T = dyn Trait`)
  เท่านั้น ไม่ใช่ตัว `Box<T>` เอง เหตุผลข้อที่สามของกรอบการตัดสินใจในหัวข้อ 27.6 จะย้อนกลับไปใช้ความรู้จาก Part 21
  โดยตรง
- **Part 8 (Slices)**: จำเป็นสำหรับเข้าใจ **auto-deref** — Part 8 เคยโชว์ว่า `&String` แปลงเป็น `&str` ให้อัตโนมัติ
  ตอนเรียก method หรือส่งเข้าฟังก์ชัน บทนี้หัวข้อ 27.3-27.4 จะอธิบายว่านี่คือกลไกเดียวกันเป๊ะกับที่ทำให้เรียก method
  บน `Box<T>` ได้โดยไม่ต้องเขียน `*box_value` เอง เพียงแค่ตัวขับเคลื่อนคือ trait `Deref` ที่ `Box<T>` implement ไว้
- **Part 23 (Lifetimes ขั้นสูง)**: Part 23 หัวข้อ 23.9 เกริ่นปัญหา **self-referential struct** ไว้ และบอกว่า
  `Rc<T>`/`Arc<T>` (Part 28) เป็นทางแก้หนึ่ง — บทนี้ไม่ได้แก้ปัญหานั้นตรง ๆ (Part 28 จะแก้) แต่จะปูพื้นฐานที่จำเป็น
  ก่อน: `Rc<T>` และ `RefCell<T>` ใน Part 28 ต่อยอดจาก `Deref`/`Drop` ที่เรียนในบทนี้ทั้งคู่ ถ้าไม่เข้าใจ `Box<T>`
  แน่นก่อน Part 28 จะเข้าใจยากขึ้นมาก เพราะ `Rc<T>` ก็ implement `Deref` แบบเดียวกัน เพียงมีความหมายเรื่อง ownership
  ต่างออกไป
- **Part 9 (Structs)**: ใช้ทวนความคุ้นเคยกับการนิยาม struct และ `impl` method เพราะตัวอย่างท้ายบท (cons list,
  expression tree, filesystem tree) ทั้งหมดสร้างจาก struct/enum ที่คุณคุ้นเคยมาแล้ว

ถ้าให้สรุปภาพรวมสั้น ๆ ก่อนเริ่ม: **บทนี้ไม่ได้สอนของใหม่ที่คุณไม่เคยเห็นหน้าค่าตามาก่อน** — คุณเห็น `Box::new(...)`
และ `Box<dyn Trait>` มาแล้วหลายครั้งตั้งแต่ Part 12 และ Part 19/21 แต่ทุกครั้งที่เห็น มันถูกใช้แบบ **pragmatic**
("ใช้แก้ปัญหาตรงหน้า ไม่ต้องรู้กลไกลึก") บทนี้คือจุดที่เราหยุดแล้วอธิบาย `Box<T>` เองแบบเป็นทางการทุกซอกทุกมุม — และ
เพราะ `Box<T>` เป็น smart pointer ที่**เรียบง่ายที่สุด**ในตระกูล smart pointer ของ Rust บทนี้จึงเป็นจุดเริ่มต้นที่ดี
ที่สุดของ mini-arc สามบทที่กำลังจะมาถึง: **Part 27 (`Box<T>`) → Part 28 (`Rc<T>`/`RefCell<T>`) → Part 29
(`Weak<T>`/`Cow<T>`)** ทุกอย่างที่เรียนในบทนี้จะกลับมาใช้ซ้ำในสองบทถัดไปแทบทั้งหมด

## เนื้อหา

### 27.1 Smart Pointer คืออะไรกันแน่ — และต่างจาก Reference (`&T`) อย่างไร

คำว่า **smart pointer** (ตัวชี้อัจฉริยะ) ฟังดูเหมือนศัพท์เทคนิคขั้นสูง แต่ความหมายจริงตรงไปตรงมากว่าที่คิด:

> **Smart pointer คือ struct ที่ "ทำตัวเหมือน pointer" (สามารถ dereference เพื่อเข้าถึงข้อมูลที่มันชี้ไปได้ ผ่าน
> trait `Deref`) แต่มีความสามารถหรือข้อมูล metadata เพิ่มเติมนอกเหนือจาก pointer ธรรมดา**

"pointer ธรรมดา" ในที่นี้หมายถึงแค่ **ที่อยู่หน่วยความจำ (memory address)** ล้วน ๆ ไม่มีอะไรมากกว่านั้น — ใน Rust
reference ธรรมดา (`&T`) ที่เรียนมาตั้งแต่ **Part 7** ก็คือ pointer แบบนี้เป๊ะ ๆ: มันเก็บที่อยู่ของค่า `T` ไว้ตัวเดียว
ไม่มีอะไรมากไปกว่านั้น (ยกเว้นกรณี fat pointer อย่าง `&str`/`&dyn Trait` ที่ Part 8 และ Part 21 อธิบายไว้ ซึ่งก็ยัง
เป็น "การยืม" อยู่ดี ไม่ใช่ "การเป็นเจ้าของ")

`Box<T>` คือ smart pointer ที่ **เรียบง่ายที่สุด** ในตระกูล smart pointer ของ Rust ความสามารถเพิ่มเติมของมันเมื่อ
เทียบกับ pointer เปล่า ๆ มีอยู่สองข้อหลัก:

1. **มันจัดสรร (allocate) หน่วยความจำบน heap ให้เองตอนสร้าง และคืนหน่วยความจำนั้นให้ระบบเองตอนถูกทำลาย** —
   pointer เปล่า ๆ ไม่ทำอะไรแบบนี้ให้ มันแค่ "ชี้ไปที่ที่อยู่หนึ่ง" เฉย ๆ ใครจัดสรร/คืนความจำเป็นเรื่องนอกตัว pointer
   เองทั้งหมด
2. **มันเป็นเจ้าของ (owns) ข้อมูลที่มันชี้ไป** — เมื่อ `Box<T>` หลุด scope ข้อมูลที่มันชี้ไปจะถูกทำลายไปด้วยเสมอ
   (รายละเอียดเต็มอยู่ในหัวข้อ 27.5) ต่าง pointer เปล่า ๆ ที่ "ไม่รับผิดชอบ" อะไรกับข้อมูลที่มันชี้ไปเลย

นี่คือความแตกต่างที่สำคัญที่สุดระหว่าง `Box<T>` กับ `&T` ที่คุณเรียนมาตั้งแต่ Part 7 — มาดูเทียบกันตรง ๆ ในโค้ด:

```rust
// 27.1 - เปรียบเทียบ &T (ยืม ไม่เป็นเจ้าของ) กับ Box<T> (เป็นเจ้าของข้อมูลบน heap)
struct Config {
    name: String,
    max_retry: u32,
}

fn main() {
    let original = Config {
        name: String::from("production"),
        max_retry: 3,
    };

    // reference ธรรมดา: "ยืม" ข้อมูลของ original ไปใช้ชั่วคราว
    // original ยังคงเป็นเจ้าของอยู่เหมือนเดิม ไม่มีอะไรเปลี่ยนเรื่อง ownership เลย
    let borrowed: &Config = &original;
    println!("ยืมดู: {} (max_retry={})", borrowed.name, borrowed.max_retry);

    // ณ จุดนี้ original ยังใช้งานได้ตามปกติ เพราะ borrowed แค่ "ยืมดู" ไม่ได้ "ยึด" อะไรไปเลย
    println!("original ยังอยู่: {}", original.name);

    // Box<T>: ย้าย (move) ความเป็นเจ้าของของ Config ตัวใหม่ไปไว้บน heap
    // ค่าที่ห่อด้วย Box::new ถูก "ยึดครอง" อย่างเต็มตัว ไม่ใช่แค่ยืมดู
    let owned: Box<Config> = Box::new(Config {
        name: String::from("staging"),
        max_retry: 5,
    });
    println!("เป็นเจ้าของ: {} (max_retry={})", owned.name, owned.max_retry);
} // ที่นี่ original ถูก drop ตามปกติ (เจ้าของคือ original เอง)
  // owned ก็ถูก drop เช่นกัน แต่ owned เป็นเจ้าของข้อมูลบน heap ของมันเอง — ทั้ง Box กับข้อมูลข้างในถูกทำลายพร้อมกัน
```

ผลลัพธ์:

```
ยืมดู: production (max_retry=3)
original ยังอยู่: production
เป็นเจ้าของ: staging (max_retry=5)
```

สังเกตความแตกต่างเชิงความหมายให้ชัด: `borrowed` ไม่มีสิทธิ์อะไรกับ `original` เลยนอกจาก "อ่านดู" — ถ้า `original`
หลุด scope ไปก่อน `borrowed` จะกลายเป็น **dangling reference** ทันที (ซึ่ง borrow checker จาก Part 7 จะไม่ยอมให้
เกิดสถานการณ์แบบนี้ตั้งแต่ compile time อยู่แล้ว) ในทางกลับกัน `owned` **คือเจ้าของ** ข้อมูล `Config { staging }`
โดยสมบูรณ์ ไม่มี "เจ้าของเดิม" ที่อื่นให้ dangling กับใครเลย เพราะข้อมูลถูกสร้างขึ้นมาบน heap ตรง ๆ ผ่าน `Box::new`
และ `owned` เป็นเจ้าของแต่ผู้เดียวตั้งแต่วินาทีที่สร้างขึ้น — นี่คือความหมายจริงของคำว่า **"`Box<T>` เป็นเจ้าของ
ข้อมูลที่มันชี้ไป"** ตามที่กล่าวไว้ข้างบน

อีกจุดที่ควรสังเกต (ทวนจาก Part 6): เพราะ `owned` เป็นเจ้าของข้อมูล การ **move** `owned` ไปที่อื่นก็คือการย้าย
ความเป็นเจ้าของไปทั้งก้อน เหมือน `String`/`Vec<T>` ทุกประการ — ประเด็นนี้สำคัญมากและจะเป็นหัวใจของเหตุผลข้อที่สอง
ในกรอบการตัดสินใจ (หัวข้อ 27.6)

### 27.2 ปัญหาที่ `Box<T>` เกิดมาแก้: Recursive Type ที่มีขนาดไม่จำกัด (`E0072`)

ก่อนจะรู้ "วิธีใช้" `Box<T>` เราควรรู้ "ปัญหาต้นกำเนิด" ของมันก่อน เพราะมันช่วยให้เข้าใจว่าทำไม Rust ต้องมี smart
pointer แบบนี้อยู่เลย ไม่ใช่แค่ "มีไว้เผื่อสะดวก"

ทวนกฎพื้นฐานที่สุดข้อหนึ่งของ Rust ที่เราใช้มาตลอดโดยไม่ทันสังเกต: **compiler ต้องรู้ขนาด (size) ที่แน่นอนของทุก
type ตั้งแต่ compile time** เพื่อคำนวณว่าจะจองพื้นที่บน stack เท่าไหร่ หรือ struct หนึ่งตัวกว้างกี่ byte — กฎนี้
ใช้ได้ดีกับ type ทั่วไปทุกตัวที่เราเคยเจอมา (`i32` มี 4 byte เสมอ, `struct Point { x: f64, y: f64 }` มี 16 byte
เสมอ) แต่จะเกิดอะไรขึ้นถ้าเราลองนิยาม type ที่ **"มีตัวเอง" อยู่ข้างในตัวเอง** — สถานการณ์คลาสสิกที่สุดคือการนิยาม
**linked list** แบบ enum ตรง ๆ โดยไม่คิดอะไรมาก:

```rust
// 27.2 - โค้ดที่ "ไม่" compile: enum ที่ชี้กลับไปยังตัวเองโดยตรง (ไม่มี indirection)
enum List {
    Cons(i32, List), // แต่ละ node เก็บเลข i32 หนึ่งตัว บวก List อีกตัวหนึ่ง (ตัวต่อไปในลิสต์)
    Nil,             // จุดสิ้นสุดของลิสต์
}

fn main() {}
```

Error จริงจาก compiler:

```
error[E0072]: recursive type `List` has infinite size
 --> src/main.rs:1:1
  |
1 | enum List {
  | ^^^^^^^^^
2 |     Cons(i32, List),
  |               ---- recursive without indirection
  |
help: insert some indirection (e.g., a `Box`, `Rc`, or `&`) to break the cycle
  |
2 |     Cons(i32, Box<List>),
  |               ++++    +

error: aborting due to 1 previous error

For more information about this error, try `rustc --explain E0072`.
```

**อ่าน error นี้ให้ทะลุ ทีละขั้น**: ลองจินตนาการว่า compiler พยายามคำนวณขนาดของ `List` ทีละขั้นตอนจริง ๆ ดู —

1. `List` มีสอง variant: `Cons(i32, List)` และ `Nil`
2. enum ต้องมีขนาดใหญ่พอที่จะเก็บ variant ที่ใหญ่ที่สุดได้ (บวก tag เล็ก ๆ ไว้บอกว่าตอนนี้เป็น variant ไหน — concept
   นี้ Part 10 อธิบายไว้แล้วตอนสอน enum กับ memory layout คร่าว ๆ)
3. เพื่อจะรู้ขนาดของ `Cons(i32, List)` ต้องรู้ขนาดของ `i32` (4 byte, รู้แน่นอน) **บวก** ขนาดของ `List` เอง (field
   ที่สอง)
4. แต่ขนาดของ `List` เอง... ก็คือสิ่งที่เรากำลังพยายามคำนวณอยู่นี่แหละ! เพื่อจะรู้ขนาดของ `List` ต้องรู้ขนาดของ
   `List` ก่อน — เป็น**วงวนไม่มีที่สิ้นสุด (infinite loop)** ในระดับตรรกะ ไม่มีทางคำนวณจบได้เลย

ลองแทนค่าดูเป็นตัวเลขจะเห็นภาพชัดขึ้นไปอีก: สมมติ `List` มีขนาด `N` byte → `Cons` ต้องมีขนาดอย่างน้อย `4 + N` byte
(เพราะเก็บ `i32` บวก `List` อีกตัว) → แต่ `Cons` เป็นส่วนหนึ่งของ `List` เอง ดังนั้น `N` ต้อง `>= 4 + N` ซึ่งเป็นไป
ไม่ได้เลยไม่ว่า `N` จะเป็นตัวเลขอะไรก็ตาม (ยกเว้น "อนันต์" ซึ่งไม่มีจริงในหน่วยความจำคอมพิวเตอร์) — **นี่คือความหมาย
ตรงตัวของคำว่า "recursive type has infinite size"** ไม่ใช่คำเปรียบเปรย แต่เป็นการคำนวณทางคณิตศาสตร์ที่ไม่มีคำตอบจริง

สังเกตว่า compiler ใจดีมากในเวอร์ชันปัจจุบัน — มัน**แนะนำทางแก้มาให้เองตรง ๆ** ในบรรทัด `help:` พร้อมบอกด้วยว่ามี
ตัวเลือกอยู่สามแบบ: `Box`, `Rc`, หรือ `&` (แนวคิด **indirection** — "การอ้อม" — ทั้งสามตัวนี้ล้วนเป็น pointer ที่มี
**ขนาดคงที่ตายตัว** ไม่ว่าจะชี้ไปยังข้อมูลขนาดเท่าไหร่ก็ตาม) `&List` ใช้ไม่ได้จริงในกรณีนี้เพราะเราต้องการให้ node
**เป็นเจ้าของ** node ต่อไปเสมอ (ไม่มีใครยืม lifetime มาจากที่ไหน) ส่วน `Rc<List>` เดี๋ยวจะเรียนใน Part 28 (ใช้เมื่อ
ต้องการ **shared ownership** — มีหลายเจ้าของพร้อมกัน) แต่สำหรับ linked list ธรรมดาที่มีเจ้าของเดียวต่อ node
**`Box<List>` คือคำตอบที่ตรงที่สุด**:

```rust
// 27.2 - แก้ด้วย Box<T>: ตอนนี้ List มีขนาดคงที่แล้ว
enum List {
    Cons(i32, Box<List>), // Box<List> มีขนาดคงที่เสมอ (เท่ากับ pointer หนึ่งตัว) ไม่ว่า List ข้างในจะซ้อนกี่ชั้น
    Nil,
}

fn main() {
    // สร้างลิสต์ 1 -> 2 -> 3 -> Nil ด้วยมือ (ยังไม่ต้องมี helper method ก็ทำงานได้แล้ว)
    let list = List::Cons(1, Box::new(List::Cons(2, Box::new(List::Cons(3, Box::new(List::Nil))))));

    // ยืนยันว่า List ตอนนี้มีขนาดคงที่จริง ๆ แล้ว
    println!("size_of::<List>() = {} bytes", std::mem::size_of::<List>());

    match list {
        List::Cons(value, _) => println!("ตัวแรกของลิสต์คือ {}", value),
        List::Nil => println!("ลิสต์เปล่า"),
    }
}
```

ผลลัพธ์ (ขนาดจริงบนเครื่อง 64-bit อาจมี padding เพิ่มเล็กน้อยตาม alignment ของ `i32` แต่หลักการคือคงที่แน่นอน):

```
size_of::<List>() = 16 bytes
ตัวแรกของลิสต์คือ 1
```

**ทำไมการเปลี่ยนแค่ `List` เป็น `Box<List>` ทำให้ทุกอย่างคำนวณจบได้**: เพราะตอนนี้ field ที่สองของ `Cons` ไม่ใช่
"`List` เต็มตัว" อีกต่อไป แต่เป็น **"pointer ไปยัง `List` ที่อยู่บน heap"** — ไม่ว่า `List` ที่ปลาย pointer นั้นจะ
ซ้อนกันลึกแค่ไหน ลึก 3 ชั้นหรือ 3 ล้านชั้น **ตัว `Box<List>` เองก็ยังมีขนาดเท่ากันเป๊ะเสมอ (หนึ่ง pointer, `usize`
หนึ่งตัว บนเครื่อง 64-bit คือ 8 byte)** เพราะมันแค่ "บอกที่อยู่" ไม่ได้ "เก็บข้อมูลทั้งก้อน" ไว้ในตัวมันเอง —
ข้อมูลจริงของ node ถัดไปอยู่ที่อื่นบน heap ต่างหาก compiler จึงคำนวณขนาดของ `List` จบได้ทันที: `4 byte (i32) + 8
byte (Box<List>) + tag เล็ก ๆ` — ไม่มีการวนซ้ำไม่มีที่สิ้นสุดอีกต่อไปแล้ว **นี่คือหัวใจที่แท้จริงของ `Box<T>`: มันคือ
เครื่องมือ "ตัดวงจรอนันต์" ของ recursive type ด้วยการแทรก indirection ที่มีขนาดคงที่เข้าไปตรงจุดที่เกิดการวนซ้ำ**

หัวข้อ 27.9 จะกลับมาสร้าง cons list แบบสมบูรณ์ที่มี `push`/`sum`/`print` ให้ใช้งานได้จริงต่อจากพื้นฐานนี้

### 27.3 `Box::new()`, `*box_value`, และ Auto-Deref สำหรับ Method Call

การใช้ `Box<T>` พื้นฐานมีอยู่สองคำสั่งหลักที่ต้องคุ้นเคย: **`Box::new(value)`** สำหรับจัดสรรค่าบน heap และคืน
`Box<T>` ที่เป็นเจ้าของค่านั้น และ **`*box_value`** สำหรับ dereference กลับไปเข้าถึงค่าจริงข้างใน (เหมือนกับที่
Part 7 สอน dereference ของ `&T` ด้วย `*reference`)

```rust
// 27.3 - Box::new และ dereference ด้วย *
fn main() {
    // Box::new(5) จัดสรรพื้นที่บน heap ขนาดเท่ากับ i32 หนึ่งตัว แล้วเก็บค่า 5 ไว้ตรงนั้น
    // b คือ Box<i32> ซึ่งเป็นเจ้าของค่า 5 บน heap ตัวนั้น
    let b = Box::new(5);
    println!("b = {}", b);

    // *b คือการ dereference: "เอาค่าที่ b ชี้ไป" ออกมา ได้ i32 จริง ๆ (ไม่ใช่ Box อีกต่อไป)
    let doubled = *b * 2;
    println!("*b * 2 = {}", doubled);

    // ทดสอบด้วยค่าที่ซับซ้อนกว่า: Box<Vec<i32>>
    let boxed_vec = Box::new(vec![10, 20, 30]);

    // เรียก .len() บน boxed_vec ตรง ๆ โดยไม่ต้องเขียน (*boxed_vec).len() เลย — นี่คือ "auto-deref"
    println!("boxed_vec.len() = {}", boxed_vec.len());

    // แต่ถ้าต้องการค่า Vec<i32> จริง ๆ ออกมาทั้งก้อน (ไม่ใช่แค่เรียก method) ต้องเขียน * ตรง ๆ
    let inner: Vec<i32> = *boxed_vec;
    println!("inner = {:?}", inner);
}
```

ผลลัพธ์:

```
b = 5
*b * 2 = 10
boxed_vec.len() = 3
inner = [10, 20, 30]
```

จุดที่น่าสังเกตที่สุดในตัวอย่างนี้คือบรรทัด `boxed_vec.len()` — เราไม่ได้เขียน `(*boxed_vec).len()` เหมือนที่ควรจะ
ต้องทำถ้าคิดตามหลักการ dereference แบบตรงไปตรงมา แต่ Rust ยอมให้เรียก `.len()` บน `Box<Vec<i32>>` ได้ตรง ๆ เลย
เหมือนกับว่า `boxed_vec` เป็น `Vec<i32>` เองอยู่แล้ว — พฤติกรรมนี้เรียกว่า **auto-deref (automatic dereferencing)**
สำหรับ method call ซึ่ง**เคยเห็นมาแล้ว**ตั้งแต่ **Part 8**: ตอนที่คุณเรียก `.len()` หรือ `.chars()` บน `&String`
ได้ตรง ๆ ทั้งที่ method เหล่านั้นถูกนิยามบน `str` ไม่ใช่ `String` — เบื้องหลังคือ compiler แปลง `&String` เป็น
`&str` ให้อัตโนมัติก่อนเรียก method (deref coercion) นั่นเอง สิ่งที่เกิดขึ้นกับ `boxed_vec.len()` คือกลไก**เดียวกัน
เป๊ะ** เพียงแค่คนละ type: compiler เห็นว่า `Box<Vec<i32>>` ไม่มี method ชื่อ `len` ของตัวเอง จึงลองตาม chain ของ
`Deref` ไปเรื่อย ๆ (`Box<Vec<i32>>` → `Vec<i32>` ผ่าน `Deref::deref`) จนเจอ method `len` ที่นิยามบน `Vec<i32>`
แล้วเรียกมันแทน — ผู้เขียนโค้ดจึงไม่ต้องมานั่งเขียน `*` ซ้อนกันหลายชั้นเองเลย ต่อให้ type ซ้อนกันลึกกว่านี้ (เช่น
`Box<Box<Vec<i32>>>`) compiler ก็ตาม chain ของ `Deref` ไปเรื่อย ๆ จนสุดให้อัตโนมัติเหมือนกัน

กลไกที่ทำให้ทั้ง `*b` และ auto-deref ของ method call ทำงานได้ คือ trait ตัวหนึ่งที่ `Box<T>` implement ไว้ให้ ชื่อว่า
**`Deref`** — หัวข้อถัดไปจะเจาะกลไกจริงของมันแบบเต็มรูปแบบ

### 27.4 `Deref`/`DerefMut` เจาะกลไกจริง: ทำไม `*box_value` ทำงานได้

จนถึงตอนนี้เราพูดถึง `*box_value` และ auto-deref แบบ "ใช้งานได้" มาตลอด แต่ยังไม่ได้อธิบายว่า **ทำไม** มันทำงานได้
— คำตอบอยู่ในสอง trait จาก standard library: `std::ops::Deref` และ `std::ops::DerefMut`

```rust
// สัญญาจริงของ trait Deref (มาจาก std::ops — แสดงไว้ให้ดูเฉย ๆ ไม่ต้องพิมพ์ตามเพราะมีอยู่แล้วใน standard library)
pub trait Deref {
    type Target: ?Sized; // associated type: "ชนิดของสิ่งที่อยู่ปลาย pointer"

    fn deref(&self) -> &Self::Target; // รับ &self คืนค่าเป็น reference ไปยัง Target
}

// DerefMut ทำงานคล้ายกัน แต่คืนค่าเป็น &mut Self::Target (สำหรับกรณีที่ต้องการแก้ไขค่าข้างในผ่าน mutable
// reference) และ DerefMut ต้อง require Deref ก่อนเสมอ (เขียนเป็น "trait DerefMut: Deref")
pub trait DerefMut: Deref {
    fn deref_mut(&mut self) -> &mut Self::Target;
}
```

`Box<T>` implement `Deref` ให้เองโดยที่คุณไม่ต้องเขียนเองเลย (เขียนแบบแนวคิดคร่าว ๆ ให้เห็นภาพ — โค้ดจริงใน
standard library ซับซ้อนกว่านี้เพราะต้องรองรับ `T: ?Sized` และเทคนิคภายในหลายอย่าง แต่แนวคิดหลักตรงตามนี้):

```
impl<T> Deref for Box<T> {
    type Target = T;

    fn deref(&self) -> &T {
        // คืน reference ไปยังข้อมูลที่ Box ชี้ไปบน heap
    }
}
```

ประโยคสำคัญที่ควรจำไว้คือ: **เครื่องหมาย `*` เมื่อใช้กับค่าที่ implement `Deref` จะถูกแปลงโดย compiler ให้เป็นการ
เรียก `*(value.deref())` เสมอ** — เขียนให้เห็นภาพเทียบกัน:

```rust
// 27.4 - พิสูจน์ว่า *b เทียบเท่ากับ *(b.deref()) จริง ๆ
use std::ops::Deref;

fn main() {
    let b = Box::new(42);

    // สองบรรทัดนี้ให้ผลลัพธ์เหมือนกันทุกประการ เพราะ *b ถูก compiler แปลงเป็นแบบที่สองให้โดยอัตโนมัติ
    let a = *b;
    let c = *(b.deref());

    println!("a = {}, c = {}", a, c);
    assert_eq!(a, c);
}
```

ผลลัพธ์:

```
a = 42, c = 42
```

ส่วน**auto-deref สำหรับ method call** (เช่น `boxed_vec.len()` จากหัวข้อก่อน) ก็ใช้กลไกเดียวกันนี้ แต่ compiler
ทำงานแบบ **"ตามหา method วนไปเรื่อย ๆ"**: เมื่อเรียก `value.method()` แล้ว `method` ไม่มีอยู่บน type ของ `value`
ตรง ๆ compiler จะเรียก `value.deref()` แล้วลองหา `method` บน type ใหม่ที่ได้ ทำซ้ำแบบนี้ไปเรื่อย ๆ จนกว่าจะเจอ
method ที่ตรงกัน (หรือจนกว่าจะไม่มี `Deref` ให้ตามต่อแล้ว ซึ่งกรณีนั้นจะเป็น compile error ว่าไม่มี method นี้) —
นี่คือเหตุผลที่ `boxed_vec.len()` ทำงานได้แม้ `Box<Vec<i32>>` ไม่มี `len` ของตัวเอง เพราะ compiler ตามไปเจอ `len`
บน `Vec<i32>` ผ่าน `Deref` หนึ่งชั้น

**`Deref`/`DerefMut` ไม่ใช่ trait ที่สงวนไว้เฉพาะ `Box<T>` เท่านั้น — คุณสามารถ implement `Deref` ให้ type ของ
ตัวเองได้เช่นกัน** เพื่อทำให้ wrapper type ของคุณ "ทำตัวเหมือน" type ที่มันห่ออยู่ (เช่น newtype pattern ที่ Part 21
สอนไว้ตอนแก้ orphan rule มักจะ implement `Deref` ควบคู่ไปด้วย เพื่อให้เรียก method ของ type ข้างในได้สะดวกโดยไม่ต้อง
เขียน `.0` ทุกครั้ง) มาดูตัวอย่างสั้น ๆ ให้เห็นภาพว่า custom `Deref` เขียนอย่างไร — newtype ที่ห่อ `Vec<T>` ไว้
เพื่อเพิ่มชื่อเรียกที่สื่อความหมายกว่า (`Stack<T>`) แต่ยังอยากเรียก method อ่านอย่างเดียวของ `Vec<T>` (เช่น `len`,
`iter`) ผ่านมันได้ตรง ๆ โดยไม่ต้องเขียน method เหล่านั้นซ้ำเองทุกตัว:

```rust
// 27.4 (เสริม) - implement Deref เองให้ newtype: Stack<T> ทำตัวเหมือน &Vec<T> ได้
use std::ops::Deref;

struct Stack<T> {
    items: Vec<T>,
}

impl<T> Stack<T> {
    fn new() -> Self {
        Stack { items: Vec::new() }
    }

    fn push(&mut self, item: T) {
        self.items.push(item);
    }
}

// implement Deref ให้ Stack<T>: Target คือ Vec<T> — บอก compiler ว่า "ถ้าเรียก method ที่ไม่มีบน Stack<T>
// ให้ตามไปหาบน Vec<T> ต่อ" ผ่าน deref(&self) -> &Vec<T>
impl<T> Deref for Stack<T> {
    type Target = Vec<T>;

    fn deref(&self) -> &Vec<T> {
        &self.items
    }
}

fn main() {
    let mut s = Stack::new();
    s.push(1);
    s.push(2);
    s.push(3);

    // len ไม่ได้นิยามบน Stack<T> เองเลย — เรียกได้ตรง ๆ ผ่าน auto-deref ไปหา Vec<T>::len()
    println!("s.len() = {}", s.len());
    // iter ก็มาจาก Vec<T> ทั้งหมด ไม่ใช่ method ของ Stack<T> เอง
    println!("s.iter().sum::<i32>() = {}", s.iter().sum::<i32>());
}
```

ผลลัพธ์:

```
s.len() = 3
s.iter().sum::<i32>() = 6
```

สังเกตว่า `Stack<T>` **ไม่ได้นิยาม** `len()` หรือ `iter()` ของตัวเองเลยแม้แต่ตัวเดียว — ทั้งสอง method นี้ถูกเรียก
ผ่าน auto-deref ไปหา `Vec<T>` ที่ `Stack<T>` ห่ออยู่ทั้งหมด นี่คือประโยชน์หลักของการ implement `Deref` เอง: **คุณ
ได้ "ยืม" method ทั้งหมดของ type ที่ห่ออยู่มาใช้แบบอัตโนมัติ โดยไม่ต้องเขียน wrapper method ซ้ำทุกตัว** รายละเอียด
การเขียน custom `Deref` แบบเต็มรูปแบบ (รวมถึงข้อควรระวังเรื่องการใช้มากเกินไปจนซ่อน logic ของ type ไว้ไม่ชัดเจน) ขอ
ยกไปพูดสั้น ๆ พอเป็นแนวทางในบทนี้ เพราะ Part 28-29 (`Rc<T>`, `RefCell<T>`, `Cow<T>`) จะใช้ pattern แบบ
"struct ที่ทำตัวเหมือน pointer ผ่าน `Deref`" นี้ซ้ำอีกหลายรอบ และจะได้เห็นรายละเอียดเชิงลึกเพิ่มเติมตอนนั้น — สิ่งที่
ต้องจำจากบทนี้ให้แน่นคือ **`*` ไม่ใช่ syntax แบบตายตัวที่ใช้ได้กับ pointer เท่านั้น มันคือ operator ที่ compiler
แปลว่า "เรียก `deref()` แล้ว dereference ผลลัพธ์" และ type ไหนก็ตามที่ implement `Deref` ก็ใช้ `*` กับมันได้ทั้งนั้น**

### 27.5 `Drop` สำหรับ `Box<T>`: ทำลายทั้ง Pointer และข้อมูลบน Heap

ทวนจาก **Part 6**: trait `Drop` ให้คุณนิยาม method `fn drop(&mut self)` ที่จะถูกเรียกโดยอัตโนมัติทันทีที่ค่าหลุด
scope — คำถามที่บทนี้ต้องตอบคือ: เมื่อ `Box<T>` หลุด scope เกิดอะไรขึ้นกับ **ทั้งตัว `Box` เอง** และ **ข้อมูลบน heap
ที่มันชี้ไป**?

คำตอบคือ **ทั้งสองอย่างถูกทำลายไปพร้อมกันเสมอ ไม่มีข้อยกเว้น** — `Box<T>` implement `Drop` ให้เองโดยอัตโนมัติ
(compiler generate ให้ ไม่ต้องเขียนเอง) ซึ่งภายใน `drop` ของ `Box<T>` จะทำสองอย่างเรียงกัน: (1) เรียก `drop` ของ
`T` ที่อยู่บน heap ก่อน (ถ้า `T` implement `Drop` เอง หรือมี field ที่ต้อง drop) และ (2) คืนหน่วยความจำบน heap
กลับให้ตัวจัดสรรความจำ (allocator) — มายืนยันด้วยตัวอย่างจริงที่ print เพื่อพิสูจน์ว่า destructor ของข้อมูลบน heap
ถูกเรียกจริง:

```rust
// 27.5 - พิสูจน์ว่า Drop ของข้อมูลข้างใน Box ถูกเรียกจริงตอน Box หลุด scope
struct Connection {
    name: String,
}

impl Drop for Connection {
    fn drop(&mut self) {
        // บรรทัดนี้จะถูก print ทันทีที่ Connection ตัวนี้ (ไม่ว่าจะอยู่บน stack หรือถูก Box ไว้บน heap) ถูกทำลาย
        println!("ปิดการเชื่อมต่อ: {}", self.name);
    }
}

fn main() {
    println!("เริ่มโปรแกรม");

    {
        // Connection ถูกจัดสรรบน heap ผ่าน Box::new และ boxed_conn เป็นเจ้าของมัน
        let boxed_conn = Box::new(Connection {
            name: String::from("database-primary"),
        });
        println!("เปิดการเชื่อมต่อ: {}", boxed_conn.name);
        // ใช้งาน boxed_conn ต่อไปตามปกติ...
    } // <-- ที่นี่ boxed_conn หลุด scope: Box<Connection> ถูก drop
      //     ซึ่งภายในจะเรียก Connection::drop ก่อน (เห็น println! ของ Connection)
      //     แล้วค่อยคืนหน่วยความจำบน heap กลับให้ allocator (ขั้นตอนนี้ไม่มี output ให้เห็น เพราะเป็นแค่การคืน
      //     หน่วยความจำ ไม่ใช่ logic ของโปรแกรมเรา)

    println!("จบโปรแกรม");
}
```

ผลลัพธ์:

```
เริ่มโปรแกรม
เปิดการเชื่อมต่อ: database-primary
ปิดการเชื่อมต่อ: database-primary
จบโปรแกรม
```

สังเกตลำดับผลลัพธ์ให้ดี: `"ปิดการเชื่อมต่อ: database-primary"` ถูก print **ทันทีที่ block ในบรรทัด `{ ... }` ปิดลง**
ไม่ใช่ตอนจบ `main()` — นี่คือพฤติกรรมเดียวกันเป๊ะกับที่ **Part 6** สอนไว้ตอน `struct` ธรรมดา (ไม่ใช่ `Box`) หลุด
scope: `Drop::drop` ถูกเรียกตอนจบ scope ของเจ้าของ ไม่ใช่ตอนจบโปรแกรม — สิ่งที่บทนี้เพิ่มเติมให้เห็นคือ **การใส่ค่า
ไว้ใน `Box` ไม่ได้เปลี่ยนกฎเรื่องเวลาที่ `drop` ถูกเรียกเลยแม้แต่นิดเดียว** เพียงแค่ข้อมูลย้ายไปอยู่บน heap เท่านั้น
เอง — `boxed_conn` (ตัวแปรบน stack ที่เก็บ pointer) หลุด scope เมื่อไหร่ ข้อมูลบน heap ที่มันเป็นเจ้าของก็ถูกทำลาย
ไปพร้อมกันในจังหวะเดียวกันเสมอ ไม่มีทาง "ลืม" คืนความจำได้เลยตราบใดที่ไม่ได้ทำอะไรผิดปกติ (เช่นสร้าง cycle ด้วย
`Rc<RefCell<T>>` ซึ่งเป็นเรื่องของ Part 28-29) — **นี่คือสิ่งที่ทำให้ Rust ไม่ต้องมี garbage collector**: ทุกอย่าง
ถูกกำหนดตายตัวตอน compile time ผ่านกฎ scope ธรรมดา ไม่ต้องมี runtime มาคอยไล่เก็บขยะทีหลัง

เทียบกับภาษาที่มี garbage collector (เช่น Java, Python, JavaScript, Go) ซึ่งการคืนหน่วยความจำเกิดขึ้น **"เมื่อไหร่
ก็ได้"** ตามจังหวะที่ GC ตัดสินใจ (ไม่แน่นอน คาดเดาเวลาแม่นยำไม่ได้) `Box<T>` ของ Rust รับประกัน **deterministic
destruction** (การทำลายที่แน่นอนตายตัว) — คุณรู้เป๊ะ ๆ ว่า resource จะถูกคืนตรงไหนในโค้ด นี่คือเหตุผลที่ `Box<T>`
(และ smart pointer อื่นของ Rust) เหมาะกับการจัดการ resource ที่ต้องปิดให้ถูกจังหวะ เช่น file handle, network
connection, lock — ประเด็นนี้จะกลับมาสำคัญมากขึ้นไปอีกใน Part 28 เมื่อพูดถึง `Rc<T>` ที่นับ reference แล้วจึงตัดสินใจ
ว่าจะ drop เมื่อไหร่

### 27.6 เมื่อไหร่ควรใช้ `Box<T>` จริง ๆ: กรอบการตัดสินใจ 3 ข้อ

มือใหม่ Rust จำนวนมากเจอปัญหาคล้ายกัน: เห็น `Box<T>` แล้วไม่แน่ใจว่า **"ควรใช้ตอนไหน"** เพราะดูเผิน ๆ แล้วมันแค่
"ห่อค่าไว้บน heap" ซึ่งฟังดูเหมือนใช้ได้ทุกที่ (และก็จริงในทางเทคนิค — คุณ `Box::new` อะไรก็ได้เสมอ) แต่การใช้แบบไม่
มีเหตุผลจะเพิ่มต้นทุนโดยไม่จำเป็น (การจัดสรร heap มีต้นทุนสูงกว่าการใช้ค่าบน stack ตรง ๆ เสมอ) — กรอบการตัดสินใจ
สามข้อนี้ช่วยตอบคำถาม "เมื่อไหร่ควรใช้" ได้อย่างมีเหตุผล ไม่ใช่ใช้ตามความเคยชิน:

#### เหตุผลข้อที่ 1: type ที่ขนาดไม่รู้แน่นอนตอน compile time แต่ context ต้องการรู้ขนาดแน่นอน

นี่คือกรณีที่หัวข้อ 27.2 อธิบายไว้แล้วอย่างละเอียด — **recursive type** (เช่น cons list, expression tree, ต้นไม้
โครงสร้างข้อมูล) เป็นกรณีคลาสสิกที่สุด แต่ยังมีอีกกรณีที่คุณ**คุ้นเคยมาแล้วตั้งแต่ Part 21**: **`Box<dyn Trait>`**
เพราะ `dyn Trait` เป็น **unsized type** (ไม่รู้ขนาดตอน compile time เพราะอาจเป็น concrete type ไหนก็ได้ที่
implement trait นั้น) — ทั้งสองกรณีมีสาเหตุเดียวกันเป๊ะ: "ขนาดที่แท้จริงไม่รู้ล่วงหน้า แต่ต้องมี handle ที่มีขนาด
คงที่มาแทน" `Box<T>` แก้ปัญหานี้ได้ทั้งคู่ เพราะ `Box<T>` เองมีขนาดเท่ากับ pointer หนึ่งตัวเสมอ ไม่ว่า `T` จะเป็น
type ไหนหรือมีขนาดเท่าไหร่ก็ตาม

#### เหตุผลข้อที่ 2: ต้องการย้ายความเป็นเจ้าของของข้อมูลขนาดใหญ่โดยไม่ copy ข้อมูลทั้งก้อน

ทวนจาก **Part 6**: การ **move** ค่าใน Rust (เช่น `let b = a;` เมื่อ `a` ไม่ implement `Copy`) ไม่ใช่การ copy บิต
ทั้งหมดของข้อมูลไปยังตำแหน่งใหม่เสมอไป — ขึ้นอยู่กับว่าข้อมูลอยู่ที่ไหน ถ้าเป็น struct ธรรมดาบน stack (เช่น `struct
BigData { values: [f64; 1_000_000] }` ที่เก็บ array ขนาดใหญ่ตรง ๆ ในตัว struct) การ move struct นี้ต้อง **copy
ข้อมูลทั้งก้อน (8 ล้าน byte) ไปยังตำแหน่งใหม่บน stack/heap ที่รับค่า** เพราะข้อมูลทั้งหมดอยู่ในตัว struct เอง ไม่มี
indirection คั่นกลาง

ในทางกลับกัน ถ้าห่อ struct นั้นด้วย `Box<T>` ก่อน (`Box<BigData>`) การ move `Box<BigData>` จะ **ย้ายแค่ pointer
ตัวเดียว (8 byte) เท่านั้น** ข้อมูลจริง 8 ล้าน byte บน heap **ไม่ต้องขยับที่เลยแม้แต่ byte เดียว** — นี่คือความ
ต่างของต้นทุนที่จับต้องได้จริง หัวข้อ 27.10 จะสร้างตัวอย่างที่วัดผลชัดเจนขึ้นด้วย `size_of`

#### เหตุผลข้อที่ 3: ต้องการเป็นเจ้าของค่า แต่สนใจแค่ว่ามัน implement trait อะไร ไม่สนใจ concrete type

นี่คือกรณีที่คุณคุ้นเคยที่สุดจาก **Part 21**: **trait object** (`Box<dyn Trait>`) — เมื่อคุณต้องการเก็บค่าหลาย
concrete type ปนกันไว้ใน collection เดียว (เช่น `Vec<Box<dyn Shape>>`) หรือ return หลาย concrete type ตามเงื่อนไข
runtime จากฟังก์ชันเดียว บทนี้ไม่สอนซ้ำในส่วนนี้เพราะ Part 21 อธิบายกลไก vtable/fat pointer/object safety ไว้ครบ
ถ้วนแล้ว สิ่งที่บทนี้อยากให้เห็นคือ **`Box<dyn Trait>` เป็นแค่กรณีพิเศษหนึ่งของเหตุผลข้อที่ 1 กับข้อที่ 3 รวมกัน**
เท่านั้นเอง ไม่ใช่ฟีเจอร์แยกต่างหากของ `Box<T>`

ตารางสรุปกรอบการตัดสินใจทั้งสามข้อ:

```
เหตุผลข้อที่ 1: ขนาดไม่รู้แน่นอนตอน compile time
  → ตัวอย่าง: recursive type (List, Expr tree), Box<dyn Trait>
  → คำถามเช็คตัวเอง: "ถ้าไม่ใส่ Box ตรงนี้ compiler จะฟ้อง E0072/E0277 (Sized) ไหม?"

เหตุผลข้อที่ 2: ต้องการย้ายข้อมูลขนาดใหญ่อย่างประหยัด
  → ตัวอย่าง: struct ที่มี array/buffer ขนาดใหญ่ที่ต้อง move ไปมาบ่อย ๆ
  → คำถามเช็คตัวเอง: "ข้อมูลนี้ใหญ่กว่า 2-3 คำ machine word ไหม และถูก move บ่อยแค่ไหน?"

เหตุผลข้อที่ 3: ต้องการเป็นเจ้าของค่าที่รู้แค่ว่า implement trait อะไร ไม่สนใจ concrete type
  → ตัวอย่าง: Box<dyn Trait> สำหรับ heterogeneous collection หรือ return หลาย concrete type
  → คำถามเช็คตัวเอง: "ฉันต้องเก็บ/return หลาย concrete type ที่ implement trait เดียวกันไหม?"
```

**สิ่งที่ควรหลีกเลี่ยง**: การใช้ `Box<T>` กับค่าเล็ก ๆ ที่ `Copy` ได้อยู่แล้ว (เช่น `Box<i32>`, `Box<bool>`) โดยไม่มี
เหตุผลข้อใดข้อหนึ่งข้างบนรองรับ — ตัวอย่างในหัวข้อ 27.3 ที่ใช้ `Box::new(5)` เป็นแค่ตัวอย่างสอน syntax เท่านั้น ใน
โค้ดจริงแทบไม่มีเหตุผลต้อง `Box` ค่า `i32` เดี่ยว ๆ เลย เพราะการจัดสรร heap มีต้นทุนสูงกว่าการเก็บค่าบน stack เสมอ
(รายละเอียดเต็มอยู่ในกับดักที่ 4 ท้ายบท)

### 27.7 `Box<T>` เทียบกับ Raw Pointer และ C++ `std::unique_ptr<T>`

สำหรับผู้มีพื้นฐาน C/C++ การเทียบ `Box<T>` กับสิ่งที่คุ้นเคยอยู่แล้วจะช่วยให้เข้าใจเร็วขึ้นมาก:

```
ภาษา C:        int* p = malloc(sizeof(int));  // จัดสรรเอง, ต้อง free() เอง, ไม่มีการันตีความปลอดภัยอะไรเลย
                                                // ลืม free() = memory leak, free() ซ้ำ = undefined behavior

C++ raw ptr:    int* p = new int(5);            // จัดสรรเอง, ต้อง delete เอง, ปัญหาเดียวกับ C
                                                // ไม่มีความหมายเรื่อง ownership ในตัว type เลย (แค่ที่อยู่)

C++ unique_ptr: std::unique_ptr<int> p(new int(5));
                // เป็นเจ้าของแต่ผู้เดียว (single ownership) เหมือน Box<T> เป๊ะ
                // ทำลายอัตโนมัติตอนหลุด scope (RAII) เหมือน Box<T> เป๊ะ
                // ย้าย ownership ได้ด้วย std::move เหมือน Box<T> move
                // Copy ไม่ได้ (deleted copy constructor) เหมือน Box<T> ที่ไม่ implement Copy

Rust:           let p: Box<i32> = Box::new(5);
                // เป็นเจ้าของแต่ผู้เดียว, ทำลายอัตโนมัติตอนหลุด scope, ย้าย ownership ได้ด้วย move ธรรมดา
                // สิ่งที่ต่างจาก unique_ptr: borrow checker ป้องกัน use-after-move และ dangling reference
                // ให้ตั้งแต่ compile time (C++ compiler ไม่เช็คเรื่องนี้ให้ ต้องพึ่งความระมัดระวังของผู้เขียนเอง
                // หรือ tool เสริมอย่าง sanitizer/valgrind ตรวจตอน runtime เท่านั้น)
```

**`Box<T>` คือสิ่งที่เทียบเคียงได้ตรงที่สุดกับ C++ `std::unique_ptr<T>` — ไม่ใช่ `std::shared_ptr<T>`** ความสับสน
ที่พบบ่อยที่สุดของผู้มีพื้นฐาน C++ คือคิดว่า smart pointer ของ Rust ทุกตัวเทียบเท่า `shared_ptr` (เพราะคำว่า "smart
pointer" ในโลก C++ มักถูกใช้แทน `shared_ptr` บ่อยที่สุดในบทสนทนาทั่วไป) แต่ในความเป็นจริง:

- `Box<T>` ↔ `std::unique_ptr<T>` — **single ownership** เจ้าของเดียว copy ไม่ได้ (ต้อง `.clone()` explicit ถ้า
  ต้องการก็อปปี้ข้อมูลจริง ไม่ใช่แชร์)
- `Rc<T>`/`Arc<T>` (Part 28/39) ↔ `std::shared_ptr<T>` — **shared ownership** นับ reference count หลายเจ้าของ
  พร้อมกันได้ clone ได้ราคาถูก (แค่เพิ่ม counter ไม่ copy ข้อมูล)
- `Weak<T>` (Part 29) ↔ `std::weak_ptr<T>` — reference ที่ไม่นับเข้า ownership count เพื่อป้องกัน reference cycle

จุดที่ทำให้ `Box<T>` **ปลอดภัยกว่า** ทั้ง raw pointer ของ C และ `unique_ptr` ของ C++ (แม้ `unique_ptr` จะปลอดภัยกว่า
raw pointer มากแล้วก็ตาม) คือ **borrow checker ของ Rust ป้องกัน use-after-move ให้ตั้งแต่ compile time** — ใน C++
ถ้าคุณ `std::move` ค่าออกจาก `unique_ptr` ไปแล้วแต่ดันไปใช้ `unique_ptr` ตัวเดิมอีกครั้งโดยไม่ตั้งใจ (มันจะกลายเป็น
`nullptr` แล้ว) โปรแกรมจะ **crash ตอน runtime** หรือแย่กว่านั้นคือ undefined behavior หากลืมเช็ค `nullptr` ก่อน
dereference แต่ใน Rust สถานการณ์เทียบเท่ากัน **compiler จะปฏิเสธไม่ให้ compile เลยตั้งแต่แรก** ด้วย error แบบ
"use of moved value" (ทวนจาก Part 6) — ความปลอดภัยนี้ได้มาโดย**ไม่มีต้นทุนด้าน performance เพิ่มเติมเลย**
(zero-cost) เพราะการเช็คทั้งหมดเกิดขึ้นตอน compile time ล้วน ๆ ไม่มี runtime check เพิ่มขึ้นมาแม้แต่นิดเดียว
เทียบกับ `unique_ptr` ที่ปลอดภัยกว่า raw pointer ก็จริง แต่ยังต้องพึ่งความระมัดระวังของผู้เขียนโค้ดเรื่อง
use-after-move อยู่ดี เพราะ C++ compiler ไม่มีกลไกตรวจสอบเรื่องนี้ให้แบบเข้มงวดเท่า Rust

มายืนยันคำกล่าวข้างบนด้วยโค้ดจริง — ลองใช้ `Box<i32>` ที่ถูก move ไปแล้ว (คล้ายสถานการณ์ `unique_ptr` ที่เพิ่ง
`std::move` ออกไป) ดูว่า Rust ตอบสนองอย่างไร:

```rust
// 27.7 - พิสูจน์ว่า Rust ปฏิเสธการใช้ Box<T> หลัง move ตั้งแต่ compile time (ไม่ใช่ runtime crash แบบ C++)
fn main() {
    let a = Box::new(10);
    let b = a; // ย้าย ownership ของ a ไปยัง b โดยสมบูรณ์ — a ใช้ต่อไม่ได้แล้วนับจากบรรทัดนี้
    println!("{} {}", a, b); // พยายามใช้ a อีกครั้งหลังจาก move ไปแล้ว
}
```

Error จริง:

```
error[E0382]: borrow of moved value: `a`
 --> src/main.rs:4:23
  |
2 |     let a = Box::new(10);
  |         - move occurs because `a` has type `Box<i32>`, which does not implement the `Copy` trait
3 |     let b = a;
  |             - value moved here
4 |     println!("{} {}", a, b);
  |                       ^ value borrowed here after move
  |
  = note: this error originates in the macro `$crate::format_args_nl` which comes from the expansion of the macro `println` (in Nightly builds, run with -Z macro-backtrace for more info)
help: consider cloning the value if the performance cost is acceptable
  |
3 |     let b = a.clone();
  |              ++++++++
```

**นี่คือหลักฐานที่จับต้องได้**: ไม่มีการรันโปรแกรมแม้แต่วินาทีเดียว — compiler ปฏิเสธตั้งแต่ก่อน compile จะเสร็จ
สมบูรณ์ด้วยซ้ำ เทียบกับ C++ ที่โค้ดสถานการณ์เดียวกัน (`auto b = std::move(a); std::cout << *a;`) จะ **compile ผ่าน
ได้สบาย ๆ** และ crash หรือ undefined behavior เอาตอน**รันจริง**เท่านั้น (เพราะ `a` กลายเป็น `nullptr` ไปแล้วหลัง
`std::move` แต่ compiler ไม่ได้ห้ามการ dereference `nullptr` ให้แบบ Rust) — ต้นทุนของความปลอดภัยนี้คือ **ศูนย์**
ในแง่ performance เพราะการตรวจสอบทั้งหมดเกิดที่ compile time ไม่มีการเช็ค `nullptr` หรือ flag อะไรเพิ่มเข้ามาใน
runtime เลยแม้แต่นิดเดียว

### 27.8 Memory Layout: ขนาดของ `Box<T>` ไม่ขึ้นกับขนาดของ `T`

หัวข้อนี้จะพิสูจน์ด้วยตัวเลขจริงในหน่วยความจำ (แนวทางเดียวกับที่ **Part 21 หัวข้อ 21.3** พิสูจน์เรื่อง fat pointer
ของ `&dyn Trait`) ว่า **ไม่ว่า `T` จะมีขนาดใหญ่แค่ไหน `Box<T>` จะมีขนาดเท่ากับ pointer หนึ่งตัวเสมอ** (ยกเว้นกรณี
`T` เป็น unsized type อย่าง `dyn Trait` ที่จะเป็น fat pointer สองเท่า ตามที่ Part 21 อธิบายไว้แล้ว):

```rust
// 27.8 - พิสูจน์ว่า Box<T> มีขนาดคงที่ (หนึ่ง pointer) ไม่ว่า T จะใหญ่แค่ไหน
use std::mem::size_of;

fn main() {
    println!("size_of::<i32>()             = {} bytes", size_of::<i32>());
    println!("size_of::<Box<i32>>()        = {} bytes", size_of::<Box<i32>>());
    println!();

    println!("size_of::<[i32; 100]>()      = {} bytes", size_of::<[i32; 100]>());
    println!("size_of::<Box<[i32; 100]>>() = {} bytes", size_of::<Box<[i32; 100]>>());
    println!();

    println!("size_of::<[u8; 1_000_000]>()      = {} bytes", size_of::<[u8; 1_000_000]>());
    println!("size_of::<Box<[u8; 1_000_000]>>() = {} bytes", size_of::<Box<[u8; 1_000_000]>>());
    println!();

    println!("size_of::<usize>()           = {} bytes (ขนาด pointer หนึ่งตัวบนเครื่องนี้)", size_of::<usize>());
}
```

ผลลัพธ์บนเครื่อง 64-bit (ตัวเลขจะเปลี่ยนไปตามสถาปัตยกรรมเครื่อง แต่แนวคิดยังเหมือนกัน):

```
size_of::<i32>()             = 4 bytes
size_of::<Box<i32>>()        = 8 bytes

size_of::<[i32; 100]>()      = 400 bytes
size_of::<Box<[i32; 100]>>() = 8 bytes

size_of::<[u8; 1_000_000]>()      = 1000000 bytes
size_of::<Box<[u8; 1_000_000]>>() = 8 bytes

size_of::<usize>()           = 8 bytes (ขนาด pointer หนึ่งตัวบนเครื่องนี้)
```

ตัวเลขนี้พูดชัดกว่าคำอธิบายใด ๆ: `[i32; 100]` (array 100 ตัวเลขแบบเก็บตรง ๆ) มีขนาด 400 byte แต่ `Box<[i32;
100]>` มีขนาดแค่ 8 byte เท่ากับ `Box<i32>` ตัวเดียว! และเมื่อขยายไปถึง array ขนาด 1 ล้าน byte (`[u8;
1_000_000]`) — `Box<[u8; 1_000_000]>` ก็**ยังคง 8 byte เท่าเดิมไม่เปลี่ยนแปลง** ไม่ว่าข้อมูลข้างในจะใหญ่ขึ้นกี่เท่า
ก็ตาม เพราะ `Box<T>` **ไม่ได้เก็บข้อมูลของ `T` ไว้ในตัวมันเอง** มันเก็บแค่ **ที่อยู่บน heap ที่ข้อมูลจริงถูกจัดเก็บ
ไว้** — ตัว `Box` ที่อยู่บน stack (หรือใน struct อื่น) จึงมีขนาดคงที่ตายตัวเสมอเท่ากับ `usize` หนึ่งตัว

ข้อเท็จจริงนี้มีความหมายในทางปฏิบัติที่สำคัญมาก: **ถ้าคุณมี struct ที่มี field เป็น array/buffer ขนาดใหญ่ และ
struct นั้นถูกเก็บไว้ในโครงสร้างอื่นที่จำกัดขนาดไว้ (เช่นเป็นสมาชิกของ `Vec<T>`, หรือถูก move ไปมาบ่อย ๆ, หรือถูก
เก็บบน stack ของ thread ที่มีขนาด stack จำกัด) การห่อด้วย `Box<T>` ช่วยลดขนาด "footprint" ของค่านั้นบน stack/ใน
struct แม่ ได้อย่างมาก** — ประเด็นนี้จะขยายเป็นตัวอย่างเต็มรูปแบบในหัวข้อ 27.10

**โบนัสที่เชื่อมโยงกับ Part 8**: หัวข้อนี้ยังอธิบายได้ว่าทำไม `Box<[T]>` (boxed slice) และ `Box<str>` มีขนาดเล็ก
กว่า `Vec<T>` และ `String` ที่มันแปลงมา — `Vec<T>`/`String` ต้องเก็บสาม field (`pointer`, `length`, `capacity`)
เพราะรองรับการ grow/shrink ได้ แต่ `Box<[T]>`/`Box<str>` (ที่ได้จาก `.into_boxed_slice()`/`.into_boxed_str()`)
มีขนาด**คงที่ตายตัวแล้ว** จึงไม่ต้องเก็บ `capacity` อีกต่อไป เหลือแค่สอง field (`pointer` + `length` — คือ fat
pointer แบบเดียวกับ `&str`/`&[T]` ที่ Part 8 สอนไว้ เพียงเปลี่ยนจาก "ยืม" เป็น "เป็นเจ้าของบน heap"):

```rust
// 27.8 (เสริม) - Box<[T]> และ Box<str>: fat pointer ที่เป็นเจ้าของ ขนาดเล็กกว่า Vec<T>/String เพราะไม่มี capacity
use std::mem::size_of;

fn main() {
    let v: Vec<i32> = vec![1, 2, 3, 4, 5];
    // Vec<T> เก็บ pointer + length + capacity (3 field) — Box<[T]> เก็บแค่ pointer + length (2 field)
    // เพราะ Box<[T]> ไม่ต้องรองรับการ push/grow อีกต่อไปหลังแปลงแล้ว (ขนาดตายตัว) จึงไม่ต้องมี capacity
    let boxed_slice: Box<[i32]> = v.into_boxed_slice();

    println!("size_of::<Vec<i32>>()   = {} bytes", size_of::<Vec<i32>>());
    println!("size_of::<Box<[i32]>>() = {} bytes", size_of::<Box<[i32]>>());
    println!("boxed_slice = {:?}", boxed_slice);

    let s = String::from("สวัสดี Rust");
    let boxed_str: Box<str> = s.into_boxed_str();
    println!("size_of::<String>()   = {} bytes", size_of::<String>());
    println!("size_of::<Box<str>>() = {} bytes", size_of::<Box<str>>());
    println!("boxed_str = {}", boxed_str);
}
```

ผลลัพธ์บนเครื่อง 64-bit:

```
size_of::<Vec<i32>>()   = 24 bytes
size_of::<Box<[i32]>>() = 16 bytes
boxed_slice = [1, 2, 3, 4, 5]
size_of::<String>()   = 24 bytes
size_of::<Box<str>>() = 16 bytes
boxed_str = สวัสดี Rust
```

`Box<[i32]>`/`Box<str>` เล็กกว่า `Vec<i32>`/`String` ต้นฉบับพอดี 8 byte (หนึ่ง `usize`) เพราะตัด field `capacity`
ออกไปได้ — เหมาะใช้ตอนที่รู้แน่ชัดแล้วว่าข้อมูลจะไม่ถูก push/grow อีกต่อไป (เช่น อ่านค่ามาจากไฟล์ config ครั้งเดียว
แล้วเก็บไว้ใช้ตลอดอายุโปรแกรม) และต้องการประหยัดหน่วยความจำอีก 8 byte ต่อค่า — ในโปรแกรมที่มีค่าแบบนี้เก็บอยู่
จำนวนมาก (นับล้านค่า) 8 byte ต่อค่าที่ประหยัดได้อาจรวมกันเป็นหน่วยความจำที่มีความหมายจริง

### 27.9 ตัวอย่างจริงที่ 1: Cons List (Linked List) ด้วย `Box<T>`

ตอนนี้มาสร้าง **cons list** (ชื่อมาจากภาษา Lisp — "cons" ย่อจาก "construct" หมายถึง node หนึ่งตัวที่เก็บค่า +
ตัวชี้ไปยัง node ถัดไป) แบบสมบูรณ์ที่ใช้งานได้จริง ต่อยอดจากพื้นฐานในหัวข้อ 27.2 — คราวนี้เพิ่ม method `push`
(เพิ่มสมาชิกใหม่ไว้ด้านหน้า), `sum` (รวมค่าทั้งหมด), และ `print_all` (แสดงสมาชิกทั้งหมดตามลำดับ):

```rust
// 27.9 - Cons list แบบสมบูรณ์: push, sum, print_all
use List::{Cons, Nil};

enum List {
    Cons(i32, Box<List>),
    Nil,
}

impl List {
    // สร้างลิสต์เปล่า
    fn new() -> Self {
        Nil
    }

    // เพิ่มสมาชิกใหม่ไว้ด้านหน้าสุด — รับ self แบบ by value (ยึด ownership ของลิสต์เดิมมาทั้งก้อน)
    // แล้วคืนลิสต์ใหม่ที่มี value อยู่หน้าสุด ตามด้วยลิสต์เดิมทั้งก้อนที่ห่อด้วย Box
    fn push(self, value: i32) -> Self {
        Cons(value, Box::new(self))
    }

    // รวมค่าทั้งหมดในลิสต์แบบ recursive
    fn sum(&self) -> i32 {
        match self {
            Cons(value, next) => value + next.sum(), // next: &Box<List> เรียก .sum() ผ่าน auto-deref ได้ตรง ๆ
            Nil => 0,
        }
    }

    // แสดงสมาชิกทั้งหมดตามลำดับ โดยไม่ recursive (ใช้ loop เพื่อไม่เปลืองพื้นที่ call stack เมื่อลิสต์ยาวมาก
    // ทวนจากกับดักท้ายบทเรื่อง stack overflow ตอน drop ลิสต์ยาว ๆ — การเขียน method แบบไม่ recursive
    // ช่วยเลี่ยงปัญหาคล้ายกันได้ในบาง scenario)
    fn print_all(&self) {
        let mut current: &List = self;
        loop {
            match current {
                Cons(value, next) => {
                    print!("{} -> ", value);
                    // next: &Box<List>, ต้องแปลงให้เป็น &List ก่อนกำหนดให้ current
                    // deref coercion เกิดขึ้นอัตโนมัติในนิพจน์การกำหนดค่า (assignment) เช่นนี้
                    current = next;
                }
                Nil => {
                    println!("Nil");
                    break;
                }
            }
        }
    }

    // นับจำนวนสมาชิกในลิสต์
    fn len(&self) -> usize {
        match self {
            Cons(_, next) => 1 + next.len(),
            Nil => 0,
        }
    }
}

fn main() {
    // สร้างลิสต์ผ่านการเรียก push ต่อกันเป็น chain: เริ่มจากเปล่า แล้วเพิ่ม 3, 2, 1 ตามลำดับ
    // เพราะ push เพิ่มไว้ด้านหน้าสุด ผลลัพธ์สุดท้ายคือ 1 -> 2 -> 3 -> Nil
    let list = List::new().push(3).push(2).push(1);

    print!("รายการทั้งหมด: ");
    list.print_all();

    println!("จำนวนสมาชิก: {}", list.len());
    println!("รวมค่าทั้งหมด: {}", list.sum());
}
```

ผลลัพธ์:

```
รายการทั้งหมด: 1 -> 2 -> 3 -> Nil
จำนวนสมาชิก: 3
รวมค่าทั้งหมด: 6
```

**เจาะ method `sum` เพื่อดูว่าทำไม `next.sum()` เรียกได้ตรง ๆ**: ใน `match self { Cons(value, next) => ... }`
เมื่อ `self` เป็น `&List` (จาก signature `fn sum(&self)`) การ pattern match ด้วย `Cons(value, next)` จะใช้
**match ergonomics** (Part 10 อธิบายไว้ตอน pattern matching เบื้องต้น) ทำให้ `next` มี type เป็น `&Box<List>`
โดยอัตโนมัติ (ไม่ต้องเขียน `&Box<List>` ตรง ๆ เอง) — เมื่อเรียก `next.sum()` compiler จะพยายามหา method `sum` บน
`&Box<List>` ก่อน ไม่พบ จึง auto-deref ไปหาบน `Box<List>` ก็ยังไม่พบ (เพราะ `sum` นิยามบน `List` ไม่ใช่บน
`Box<List>`) จึง auto-deref อีกชั้นไปหาบน `List` ตรงนั้นเจอ `sum` ที่ถูกต้อง แล้วเรียกด้วย `&next` (ที่ได้จากการ
deref สองชั้น) — นี่คือ auto-deref แบบ **หลายชั้น (multi-hop)** ที่หัวข้อ 27.4 อธิบายไว้แบบสั้น ๆ ตอนนี้เห็นการใช้
งานจริงแล้ว

**เจาะ method `print_all` ที่ทำไม `current = next` compile ผ่าน**: `current` ประกาศชนิดเป็น `&List` ชัดเจน
(`let mut current: &List = self;`) แต่ `next` ที่ได้จาก pattern match มี type เป็น `&Box<List>` — การกำหนดค่า
`current = next` ต้องให้ type ทั้งสองฝั่งตรงกัน แต่ตรงนี้ compile ผ่านได้เพราะ **deref coercion เป็นหนึ่งใน
"coercion site" มาตรฐานของภาษา** (จุดที่ compiler ยอมแปลง type ให้อัตโนมัติผ่าน `Deref`) — ฝั่งขวา (`&Box<List>`)
ถูกแปลงเป็น `&List` โดยอัตโนมัติผ่าน `Box<List>: Deref<Target = List>` ก่อนถูกกำหนดให้ `current` พฤติกรรมนี้เป็น
ธรรมชาติเดียวกับ deref coercion ของ argument ฟังก์ชันที่ Part 8 สอนไว้ตอน `&String` แปลงเป็น `&str`

### 27.10 ตัวอย่างจริงที่ 2: ย้าย Struct ขนาดใหญ่อย่างประหยัดด้วย `Box<T>`

ตัวอย่างนี้ตรงกับเหตุผลข้อที่ 2 ของกรอบการตัดสินใจในหัวข้อ 27.6 — สมมติระบบประมวลผลภาพที่แต่ละ "เฟรม" เก็บข้อมูล
พิกเซลไว้ใน array ขนาดคงที่ค่อนข้างใหญ่ (จำลองด้วย `[u8; 1_000_000]` แทนภาพขนาด ~1000x1000 พิกเซล ระดับสีเทา):

```rust
// 27.10 - Box<T> สำหรับย้ายข้อมูลขนาดใหญ่อย่างประหยัด (ไม่ copy ข้อมูลทั้งก้อนตอน move)
use std::mem::size_of;

// จำลองข้อมูลภาพขนาดใหญ่: buffer 1MB เก็บอยู่ตรงในตัว struct เอง (ไม่ผ่าน Vec/heap แยก)
struct FrameBuffer {
    pixels: [u8; 1_000_000],
    width: u32,
    height: u32,
}

impl FrameBuffer {
    fn new(width: u32, height: u32) -> Self {
        FrameBuffer {
            pixels: [0; 1_000_000],
            width,
            height,
        }
    }

    fn brightness_sum(&self) -> u64 {
        // sum ค่าพิกเซลทั้งหมด (แค่ตัวอย่างการประมวลผลง่าย ๆ เพื่อพิสูจน์ว่าข้อมูลยังถูกต้องหลัง move)
        self.pixels.iter().map(|&p| p as u64).sum()
    }
}

// ฟังก์ชันที่รับ FrameBuffer แบบ boxed เข้ามา "ประมวลผล" แล้วส่งกลับ (จำลอง pipeline การประมวลผลภาพ
// ที่ frame ต้องถูกส่งผ่านหลายขั้นตอน/หลายฟังก์ชันต่อกัน)
fn apply_brightness_filter(mut frame: Box<FrameBuffer>, delta: u8) -> Box<FrameBuffer> {
    for pixel in frame.pixels.iter_mut() {
        *pixel = pixel.saturating_add(delta);
    }
    frame
}

fn main() {
    // เปรียบเทียบขนาด: FrameBuffer ตัวเต็ม (1MB+) เทียบกับ Box<FrameBuffer> (แค่ pointer เดียว)
    println!("size_of::<FrameBuffer>()      = {} bytes", size_of::<FrameBuffer>());
    println!("size_of::<Box<FrameBuffer>>() = {} bytes", size_of::<Box<FrameBuffer>>());

    // จัดสรร FrameBuffer บน heap ตั้งแต่แรก ผ่าน Box::new
    let frame = Box::new(FrameBuffer::new(1000, 1000));
    println!("brightness ก่อนปรับ: {}", frame.brightness_sum());

    // ส่ง frame (Box<FrameBuffer>) เข้าฟังก์ชัน — สิ่งที่ถูก "ย้าย" เข้าไปในฟังก์ชันคือ pointer 8 byte
    // เดียวเท่านั้น ไม่ใช่ข้อมูลพิกเซล 1 ล้าน byte เลย ต่อให้ frame ถูกส่งผ่านฟังก์ชันแบบนี้อีกสิบขั้นตอนใน
    // pipeline การประมวลผลภาพ ต้นทุนของการ "ย้าย" ในแต่ละขั้นก็ยังคงเป็นแค่ 8 byte เท่ากันทุกครั้ง
    let brightened = apply_brightness_filter(frame, 50);
    println!("brightness หลังปรับ: {}", brightened.brightness_sum());
}
```

ผลลัพธ์บนเครื่อง 64-bit:

```
size_of::<FrameBuffer>()      = 1000008 bytes
size_of::<Box<FrameBuffer>>() = 8 bytes
brightness ก่อนปรับ: 0
brightness หลังปรับ: 50000000
```

(ตัวเลข `1000008` มาจาก `1_000_000 byte` ของ `pixels` บวก `4 byte` ของ `width` บวก `4 byte` ของ `height` พอดี
ไม่มี padding เพิ่มในกรณีนี้เพราะ `1_000_000` หารด้วย `4` (alignment ของ `u32`) ลงตัว — รายละเอียด padding ไม่ใช่
จุดสำคัญของตัวอย่างนี้ จุดสำคัญคือความต่างระดับ**หลักแสนเท่า**ระหว่างสอง `size_of`)

**เจาะประเด็นสำคัญที่สุดของตัวอย่างนี้**: ถ้า `apply_brightness_filter` รับ `FrameBuffer` แบบ by-value ตรง ๆ (ไม่
ผ่าน `Box`) การเรียกฟังก์ชันนี้ (และการ `return frame` กลับออกมา) จะเกี่ยวข้องกับการย้ายข้อมูลระดับ **~1 ล้าน byte**
ไปมาระหว่าง stack frame ของฟังก์ชันที่เรียกกับฟังก์ชันที่ถูกเรียก ทุกครั้งที่มีการเรียกฟังก์ชัน (ในทางเทคนิค
compiler อาจ optimize บางกรณีด้วยเทคนิค move elision แต่ไม่ใช่การันตีในทุกสถานการณ์ และยิ่งซับซ้อนขึ้นเมื่อ pipeline
มีหลายขั้นตอน) เมื่อเปลี่ยนมาใช้ `Box<FrameBuffer>` **การย้ายทุกครั้งจะเหลือแค่การย้าย pointer ตัวเดียว (8 byte)**
เท่านั้น ไม่ว่าจะส่งผ่านฟังก์ชันกี่ชั้นก็ตาม เพราะข้อมูลจริงอยู่บน heap นิ่ง ๆ ที่เดียวตลอดเวลา มีแค่ "ใครเป็น
เจ้าของ pointer ที่ชี้ไปยังมัน" เท่านั้นที่เปลี่ยนมือไปตามลำดับการเรียกฟังก์ชัน — **นี่คือประโยชน์ที่จับต้องได้จริง
ของเหตุผลข้อที่ 2** และเป็นเทคนิคที่ใช้กันจริงในโค้ดประมวลผลภาพ/เสียง/ข้อมูลขนาดใหญ่ที่ struct มี buffer คงที่ขนาด
ใหญ่เป็น field

## กับดักที่พบบ่อย (Common Pitfalls)

### กับดักที่ 1: ลืมว่าต้องใส่ `Box` ในทุก recursive branch ไม่ใช่แค่บางจุด (`E0072` ที่ยังไม่หายสนิท)

มือใหม่บางคนเจอ `E0072` แล้วรีบใส่ `Box` ตรงจุดแรกที่เจอ error โดยไม่ทันสังเกตว่า field อื่นที่ recursive เหมือนกัน
ก็ต้องใส่ด้วย ลองดูตัวอย่างที่มี recursive field สองจุด:

```rust
// พยายามแก้ E0072 แต่ใส่ Box ไม่ครบทุกจุดที่ recursive
enum Tree {
    Leaf(i32),
    Node(Box<Tree>, Tree), // ใส่ Box ให้ field แรก แต่ลืม field ที่สอง — ยังคง recursive without indirection อยู่
}

fn main() {}
```

Error ที่ได้ยังคงเป็น `E0072` เหมือนเดิม เพราะ `Node` variant ยังมี field ที่สอง (`Tree` เปล่า ๆ) ที่ไม่มี
indirection คั่นอยู่ดี — **วิธีแก้**: ต้องใส่ `Box` ให้ทุก field ที่อ้างถึง type ตัวเองโดยตรงในทุก variant ไม่ใช่
แค่จุดแรกที่ compiler รายงาน (compiler รายงานแค่จุดเดียวก่อน แต่ถ้ามีหลายจุดต้องไล่แก้ให้ครบทุกจุด):

```rust
enum Tree {
    Leaf(i32),
    Node(Box<Tree>, Box<Tree>), // ต้องใส่ Box ทั้งสอง field ที่เป็น Tree ซ้อนตัวเอง
}

fn main() {
    let tree = Tree::Node(
        Box::new(Tree::Leaf(1)),
        Box::new(Tree::Node(Box::new(Tree::Leaf(2)), Box::new(Tree::Leaf(3)))),
    );
    let _ = tree;
}
```

### กับดักที่ 2: พยายาม move ค่าออกจาก `Box` ที่อยู่หลัง reference (`E0507`)

```rust
// พยายาม dereference และ move ค่าออกจาก Box ที่เข้าถึงผ่าน reference — ไม่ผ่าน
fn main() {
    let b = Box::new(String::from("hello"));
    let r = &b;
    let inner = *r; // พยายามย้าย String ออกจาก Box ที่ r แค่ "ยืม" มา
    println!("{}", inner);
}
```

Error จริง:

```
error[E0507]: cannot move out of `*r` which is behind a shared reference
 --> src/main.rs:4:17
  |
4 |     let inner = *r;
  |                 ^^ move occurs because `*r` has type `Box<String>`, which does not implement the `Copy` trait
  |
help: consider removing the dereference here
  |
4 -     let inner = *r;
4 +     let inner = r;
  |
help: consider cloning the value if the performance cost is acceptable
  |
4 -     let inner = *r;
4 +     let inner = r.clone();
  |
```

**อ่าน error นี้ให้ทะลุ**: `r` เป็น `&Box<String>` — มัน "ยืม" `Box<String>` มาเฉย ๆ ไม่ได้เป็นเจ้าของ การเขียน
`*r` พยายามดึงค่า `Box<String>` ที่อยู่ปลาย `r` ออกมาเป็นเจ้าของใหม่ (`inner`) ซึ่งขัดกับกฎ **Part 7** โดยตรง:
**คุณย้ายค่าออกจากสิ่งที่แค่ยืมมาไม่ได้** เพราะจะทำให้เจ้าของเดิม (`b`) เหลือแค่ "โพรง" ที่ไม่มีข้อมูลอยู่ ทั้งที่
ยังมีคนอื่น (`r`) คิดว่ากำลังยืมข้อมูลที่สมบูรณ์อยู่ — **วิธีแก้ที่ compiler แนะนำ**: ถ้าแค่ต้องการอ่านค่า ให้ตัด
`*` ออก (`let inner = r;` จะได้ `inner: &Box<String>` แทน) หรือถ้าต้องการเป็นเจ้าของข้อมูลจริง ๆ (ไม่ใช่แค่ยืม)
ให้ `.clone()` (ต้อง `T: Clone` — `String` implement `Clone` อยู่แล้ว) ซึ่งจะสร้างสำเนาข้อมูลใหม่แยกออกมา ไม่แย่ง
ความเป็นเจ้าของจากใคร

### กับดักที่ 3: `.clone()` บน `Box<T>` คือการก็อปปี้ข้อมูลทั้งก้อน ไม่ใช่การแชร์ ownership

จุดที่สับสนบ่อยที่สุดสำหรับคนที่เพิ่งได้ยินชื่อ `Rc<T>` มาบ้างแล้ว (Part 28) คือคิดว่า `Box<T>::clone()` ทำงาน
เหมือน `Rc<T>::clone()` (ที่แค่เพิ่ม reference count ราคาถูก ไม่ copy ข้อมูล) — แต่ `Box<T>` **ไม่มีแนวคิดเรื่อง
shared ownership เลย** มันเป็น single-ownership เท่านั้น ถ้า `T: Clone` การเรียก `boxed.clone()` จะ **สร้าง heap
allocation ใหม่ทั้งก้อน แล้ว copy ข้อมูลของ `T` ไปไว้ในนั้น** — เป็นการก็อปปี้แบบเต็มรูปแบบ (deep copy) ไม่ใช่การ
แชร์ pointer เดียวกันแบบที่ `Rc<T>` ทำ:

```rust
fn main() {
    let a = Box::new(vec![1, 2, 3]);
    let b = a.clone(); // สร้าง Vec ใหม่ทั้งก้อนบน heap คนละตำแหน่งกับ a โดยสมบูรณ์

    // พิสูจน์ว่า a และ b เป็นข้อมูลแยกกันจริง ไม่ใช่ตัวเดียวกัน โดยดูที่ address ของ heap data
    let addr_a: *const Vec<i32> = &*a;
    let addr_b: *const Vec<i32> = &*b;
    println!("a และ b ชี้ไปคนละตำแหน่งบน heap: {}", addr_a != addr_b);
}
```

ผลลัพธ์:

```
a และ b ชี้ไปคนละตำแหน่งบน heap: true
```

ถ้าเป้าหมายจริง ๆ คือ **"อยากให้หลายตัวแปรแชร์ข้อมูลเดียวกัน โดยไม่ต้อง copy"** นั่นคือสัญญาณว่าคุณต้องการ `Rc<T>`
(สำหรับ single-thread) ไม่ใช่ `Box<T>` — Part 28 จะอธิบายเรื่องนี้แบบเต็มรูปแบบ **จำไว้ให้แน่น**: `Box<T>` คือ
single ownership เสมอ ไม่มีข้อยกเว้น การ `.clone()` มันจึงหมายถึง "สร้างเจ้าของใหม่ที่เป็นอิสระจากกัน" เท่านั้น

### กับดักที่ 4: Drop ลิสต์ที่ยาวมากด้วย recursive `Drop` อาจทำให้ stack overflow

นี่คือกับดักที่มีชื่อเสียงในวงการ Rust และเป็นตัวอย่างที่ดีว่าทำไม "ปลอดภัยจาก memory leak/dangling pointer" ไม่ได้
แปลว่า "ไม่มีปัญหาอะไรเลย" — โครงสร้าง cons list แบบหัวข้อ 27.9 ถ้าถูกสร้างให้ยาวมาก (เช่นหลักแสนหรือหลักล้าน node)
การ `drop` ลิสต์ทั้งก้อนอาจทำให้ **stack overflow** ได้จริง เพราะ `Drop` ที่ compiler generate ให้กับ enum
recursive แบบนี้ (ที่ไม่ได้เขียน custom `Drop` เอง) จะ **ทำงานแบบ recursive**: drop node แรกต้อง drop node ที่
สองก่อน ซึ่งต้อง drop node ที่สามก่อน ... วนไปเรื่อย ๆ ตามความยาวของลิสต์ — แต่ละชั้นของการ recursive ใช้พื้นที่
บน call stack เพิ่มขึ้น ถ้าลิสต์ยาวพอ (มากกว่าที่ stack ของ thread จะรับได้ ปกติค่า default อยู่ที่ประมาณ 8MB
สำหรับ main thread บนหลาย platform) โปรแกรมจะ crash ด้วย stack overflow

มายืนยันด้วยโค้ดจริง — สร้างลิสต์ยาว 1 ล้าน node แบบตรงไปตรงมา (ไม่มี custom `Drop`) แล้วปล่อยให้มันหลุด scope:

```rust
// สร้างลิสต์ยาว 1 ล้าน node ด้วยวิธีตรงไปตรงมา แล้วปล่อยให้ Drop เริ่มทำงานแบบ recursive ตามค่า default
enum List {
    Cons(i32, Box<List>),
    Nil,
}

fn main() {
    let mut list = List::Nil;
    for i in 1..=1_000_000 {
        list = List::Cons(i, Box::new(list));
    }
    println!("สร้างลิสต์ยาว 1,000,000 node สำเร็จ กำลังจะ drop...");
    drop(list); // <-- จุดนี้คือจุดที่ค้าง/crash จริงบนเครื่องส่วนใหญ่
    println!("drop สำเร็จ"); // บรรทัดนี้มักไม่ถูก print เลย เพราะโปรแกรม crash ไปก่อน
}
```

ผลลัพธ์จริงที่ได้ (อาจต่างกันไปเล็กน้อยตามขนาด stack ของแต่ละเครื่อง แต่ผลลัพธ์โดยรวมคือ crash เหมือนกัน):

```
สร้างลิสต์ยาว 1,000,000 node สำเร็จ กำลังจะ drop...

thread 'main' has overflowed its stack
fatal runtime error: stack overflow, aborting
```

โปรแกรม crash จริงตอน `drop(list)` — `println!("drop สำเร็จ")` ไม่ถูก print เลยแม้แต่บรรทัดเดียว ยืนยันว่าปัญหานี้
ไม่ใช่เรื่องสมมติ แต่เกิดขึ้นจริงกับ default `Drop` ที่ compiler generate ให้กับ enum recursive เมื่อลิสต์ยาวพอ

**วิธีแก้ในทางปฏิบัติ**: เทคนิคมาตรฐานที่ใช้แก้ปัญหานี้ (ใช้จริงใน source code ของ `std::collections::LinkedList`
และหนังสือ Rust หลายเล่มที่สอนเขียน linked list) คือ**แยก type ที่ implement `Drop` ออกจาก type ที่ recursive**
— ให้ type ชั้นนอก (ที่มีอยู่แค่ตัวเดียวเสมอ ไม่ซ้อนตัวเอง) เป็นตัว implement `Drop` แบบ **ไม่ recursive** (ใช้ loop
ไล่ทำลาย node ทีละตัว) ส่วน type ที่ซ้อนกันอยู่ข้างในไม่ต้อง implement `Drop` เอง (ให้ compiler generate default
drop ธรรมดา ซึ่งจะไม่ recursive ต่อ เพราะเราตัดข้อมูลข้างในให้เหลือ "ว่าง" ก่อนที่แต่ละ node จะถูกทำลายจริง):

```rust
// เทคนิคมาตรฐาน: แยก node/link (ไม่ implement Drop) ออกจาก wrapper ชั้นนอก (implement Drop แบบ loop)
enum Link {
    Empty,
    More(Box<Node>),
}

struct Node {
    value: i32,
    next: Link, // Node และ Link ไม่ implement Drop เอง — ใช้ default drop ของ compiler เท่านั้น
}

struct SafeList {
    head: Link,
}

impl SafeList {
    fn new() -> Self {
        SafeList { head: Link::Empty }
    }

    fn push(&mut self, value: i32) {
        let new_node = Box::new(Node {
            value,
            // ดึง head เดิมออกมาเป็น next ของ node ใหม่ แล้วแทนที่ head เดิมด้วย Empty ชั่วคราว
            next: std::mem::replace(&mut self.head, Link::Empty),
        });
        self.head = Link::More(new_node);
    }
}

// implement Drop เฉพาะที่ SafeList (type ชั้นนอกที่มีอยู่แค่ตัวเดียวในระบบเสมอ) เท่านั้น — Node/Link ไม่มี Drop
impl Drop for SafeList {
    fn drop(&mut self) {
        // ดึง node ออกมาทีละตัวด้วย loop (ไม่ใช่ recursive function call) จึงไม่กิน call stack เพิ่มตามความยาวลิสต์
        let mut cur_link = std::mem::replace(&mut self.head, Link::Empty);
        while let Link::More(mut boxed_node) = cur_link {
            // ตัด next ของ node ปัจจุบันให้เป็น Empty ก่อน แล้วเลื่อนไป node ถัดไปในรอบ loop ถัดไป
            cur_link = std::mem::replace(&mut boxed_node.next, Link::Empty);
            // boxed_node หลุด scope ตรงนี้ (จบ while body) — เพราะ next ของมันถูกตั้งเป็น Empty ไปแล้ว การ drop
            // Node ตัวนี้จึงจบแค่ชั้นเดียว ไม่ recursive ต่อไปยัง node ถัดไปเลย (ต่างจาก List เดิมที่พังเพราะ
            // ตัว List เองถูก implement Drop ซ้อนอยู่ทุกชั้นของการซ้อนกัน)
        }
    }
}

fn main() {
    let mut list = SafeList::new();
    for i in 1..=1_000_000 {
        list.push(i);
    }
    println!("สร้างลิสต์ยาว 1,000,000 node สำเร็จ กำลังจะ drop...");
    drop(list);
    println!("drop สำเร็จโดยไม่ stack overflow");
}
```

ผลลัพธ์:

```
สร้างลิสต์ยาว 1,000,000 node สำเร็จ กำลังจะ drop...
drop สำเร็จโดยไม่ stack overflow
```

คราวนี้โปรแกรม drop ลิสต์ 1 ล้าน node ได้สำเร็จโดยไม่ crash เลย **สังเกตความต่างที่สำคัญที่สุด**: `SafeList` มีอยู่
แค่ **หนึ่งตัว** ในทั้งระบบเสมอ (มันคือ "กล่องห่อ" ชั้นนอกสุด) ดังนั้น `Drop::drop` ของ `SafeList` **ถูกเรียกแค่ครั้ง
เดียว** ไม่ว่าลิสต์จะยาวแค่ไหนก็ตาม — ส่วน `Node`/`Link` ที่ซ้อนกันอยู่ข้างในนับล้านชั้นนั้น **ไม่ได้ implement
`Drop` เอง** จึงใช้ default drop ธรรมดาของ compiler ซึ่งไม่ทำอะไรซับซ้อนไปกว่า "drop field ของตัวเองตามลำดับ" —
เพราะเราตั้งใจตัด `next` ของแต่ละ node ให้เป็น `Link::Empty` ก่อนที่ node นั้นจะหลุด scope จริงในแต่ละรอบ loop
การ drop node แต่ละตัวจึงจบในตัวเอง **ไม่ไล่ recursive ต่อไปยัง node ถัดไปเลย** ตรงกันข้ามกับโค้ดเดิมที่พังเพราะ
`List` (type เดียวกันที่ซ้อนตัวเองอยู่ทุกชั้น) implement `Drop` ตรง ๆ ทำให้ทุกชั้นของการซ้อนกันกลายเป็นการเรียก
`drop` แบบ recursive เต็มรูปแบบ

**ข้อคิดสำคัญที่ควรติดตัวไป**: กับดักนี้ไม่ได้เกิดจาก `Box<T>` "มีบั๊ก" แต่เกิดจาก **การออกแบบโครงสร้างข้อมูลแบบ
recursive เอง** มีข้อจำกัดตามธรรมชาติของมัน — ในโค้ดจริงที่ต้องรองรับ list/tree ที่อาจยาว/ลึกมาก ๆ (เช่น parser
ที่ parse ไฟล์ input ขนาดใหญ่จนได้ AST ที่ลึกมาก) นี่เป็นเหตุผลหนึ่งที่วิศวกร Rust จำนวนมากเลือกใช้ `Vec<T>` แทน
linked list แบบ `Box`-based ตั้งแต่แรกเมื่อทำได้ (เพราะ `Vec<T>` เก็บข้อมูลต่อกันเป็นแถวเดียว ไม่มีปัญหา recursive
drop เลย) — cons list ใน Part นี้เป็นตัวอย่างที่ดีมากสำหรับ**สอนแนวคิด** `Box<T>` แต่ไม่ใช่โครงสร้างข้อมูลที่แนะนำ
ให้ใช้ในงานจริงที่ต้องการ list ทั่วไป (`Vec<T>`, `VecDeque<T>` จาก Part 13 เหมาะกว่าในสถานการณ์ส่วนใหญ่)

### กับดักที่ 5: ใช้ `Box<T>` กับค่าเล็ก ๆ ที่ `Copy` ได้อยู่แล้วโดยไม่มีเหตุผล

```rust
// ใช้ Box โดยไม่มีเหตุผลจากกรอบการตัดสินใจในหัวข้อ 27.6 ข้อใดเลย
fn add_boxed(a: Box<i32>, b: Box<i32>) -> Box<i32> {
    Box::new(*a + *b)
}

fn main() {
    let result = add_boxed(Box::new(3), Box::new(4));
    println!("{}", result);
}
```

โค้ดนี้ **compile ผ่านและรันได้ถูกต้อง** ไม่มี error ใด ๆ เลย — แต่นี่คือกับดักเชิง **design/performance** ไม่ใช่
กับดักเชิง compile error: `i32` มีขนาดแค่ 4 byte และ implement `Copy` อยู่แล้ว การส่งผ่านค่าตรง ๆ (`a: i32, b:
i32`) มีต้นทุนแค่การ copy 4 byte บน stack (เร็วระดับเดียวกับการอ่านค่าจาก register) แต่การห่อด้วย `Box` ทำให้
ต้อง **จัดสรร heap memory** (ผ่าน allocator เรียก system call หรือจัดการ heap metadata) ทั้งขาจัดสรรและขาคืน
ซึ่งช้ากว่าการใช้ค่าบน stack ตรง ๆ หลายเท่าตัว (ระดับหลายสิบถึงหลักร้อย nanosecond ต่อครั้งของการจัดสรร เทียบกับ
การ copy 4 byte ที่เร็วกว่ามาก) โดยไม่ได้ผลประโยชน์อะไรตอบแทนเลยจากเหตุผลข้อ 1-3 ในหัวข้อ 27.6 — **วิธีแก้**: ใช้
ค่าตรง ๆ โดยไม่ต้อง `Box` เมื่อ type นั้นเล็กและ `Copy` ได้อยู่แล้ว (`fn add(a: i32, b: i32) -> i32 { a + b }`)
เก็บ `Box<T>` ไว้สำหรับสถานการณ์ที่มีเหตุผลรองรับจริง ๆ ตามกรอบการตัดสินใจในหัวข้อ 27.6 เท่านั้น

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** เขียนฟังก์ชัน `fn double_boxed(b: Box<i32>) -> Box<i32>` ที่รับ `Box<i32>` เข้ามา คำนวณค่าข้างในคูณ
   สอง แล้วคืนค่าเป็น `Box<i32>` ตัวใหม่ ทดสอบด้วย `main()` ที่เรียก `double_boxed(Box::new(21))` แล้ว print ผลลัพธ์
   ควรได้ `42`
   (Hint: dereference ด้วย `*b` เพื่อเอาค่า `i32` ออกมาคำนวณ แล้วห่อผลลัพธ์กลับด้วย `Box::new` อีกครั้ง)

2. **[ง่าย-กลาง]** ต่อจากตัวอย่าง cons list ในหัวข้อ 27.9 เขียน method ใหม่ชื่อ `fn contains(&self, target: i32)
   -> bool` ที่ตรวจว่าลิสต์มีค่า `target` อยู่หรือไม่ (แบบ recursive หรือ loop ก็ได้) ทดสอบกับลิสต์ `1 -> 2 -> 3 ->
   Nil` ว่า `contains(2)` คืน `true` และ `contains(9)` คืน `false`
   (Hint: โครงสร้างคล้าย method `sum`/`len` ที่มีอยู่แล้ว เพียงเปลี่ยนเงื่อนไขการเช็คและค่าที่ return)

3. **[กลาง-ยาก]** สร้าง recursive enum ชื่อ `Expr` แทนต้นไม้นิพจน์ทางคณิตศาสตร์ที่รองรับ: ตัวเลข (`Num(f64)`),
   การบวก (`Add(Box<Expr>, Box<Expr>)`), และการคูณ (`Mul(Box<Expr>, Box<Expr>)`) เขียนฟังก์ชัน `fn eval(expr:
   &Expr) -> f64` ที่คำนวณผลลัพธ์ของนิพจน์แบบ recursive ทดสอบด้วยนิพจน์ที่แทน `(2 + 3) * 4` (ควรได้ผลลัพธ์ `20.0`)
   (Hint: โครงสร้างคล้าย `List` ในหัวข้อ 27.2/27.9 แต่มีสอง recursive field ใน `Add`/`Mul` แทนที่จะมีแค่หนึ่ง —
   ระวังกับดักที่ 1 ท้ายบท ต้องใส่ `Box` ให้ทั้งสอง field)

4. **[ยาก/ประยุกต์]** จำลองระบบไฟล์แบบง่าย (คล้ายโครงสร้างโฟลเดอร์) ด้วย struct `FileSystemNode` ที่มี field
   `name: String`, `size: u64` (สำหรับไฟล์ คือขนาดไฟล์จริง สำหรับโฟลเดอร์ให้เป็น `0` แล้วคำนวณจาก children รวมกัน),
   และ `children: Vec<Box<FileSystemNode>>` (ถ้าเป็นไฟล์ธรรมดา `children` จะเป็น `Vec` เปล่า) เขียนฟังก์ชัน
   `fn total_size(node: &FileSystemNode) -> u64` ที่คำนวณขนาดรวมของ node นั้นบวกกับ children ทั้งหมดแบบ
   recursive และฟังก์ชัน `fn print_tree(node: &FileSystemNode, depth: usize)` ที่แสดงโครงสร้างต้นไม้แบบมี
   indentation ตาม `depth` (ใช้ `"  ".repeat(depth)` นำหน้าแต่ละบรรทัด) ทดสอบด้วยโครงสร้างจำลอง: โฟลเดอร์ `"root"`
   มีไฟล์ `"readme.txt"` (100 byte) และโฟลเดอร์ย่อย `"src"` ที่มีไฟล์ `"main.rs"` (2000 byte) กับ `"lib.rs"` (1500
   byte) — `total_size` ของ `"root"` ควรได้ `3600`
   (Hint: นี่คือการรวมเหตุผลข้อที่ 1 ในหัวข้อ 27.6 — `Vec<Box<FileSystemNode>>` ไม่จำเป็นต้อง `Box` เพื่อแก้ปัญหา
   ขนาดไม่จำกัดเหมือน enum recursive ตรง ๆ เพราะ `Vec<T>` เก็บข้อมูลบน heap อยู่แล้วโดยธรรมชาติของมันเอง — ลองคิด
   ต่อว่าทำไมโจทย์นี้ยังแนะนำให้ใช้ `Box` อยู่ดี คำตอบเกี่ยวกับเหตุผลข้อที่ 2 เรื่องการย้ายข้อมูลอย่างประหยัดเมื่อ
   `FileSystemNode` มี field อื่นเพิ่มเข้ามาในอนาคต)

## สรุป

บทนี้อธิบาย `Box<T>` แบบเป็นทางการเต็มรูปแบบ หลังจากที่คุณใช้มันแบบ pragmatic มาตั้งแต่ Part 12 และ Part 19/21 ใน
บริบทของ `Box<dyn Trait>`/`Box<dyn Error>` เท่านั้น — ประเด็นสำคัญที่สุดที่ควรติดตัวไปคือ:

- **Smart pointer** คือ struct ที่ implement `Deref` (และมักจะ `Drop`) ให้ "ทำตัวเหมือน pointer" แต่มีความสามารถ
  เพิ่มเติม — `Box<T>` เป็นตัวที่เรียบง่ายที่สุด: จัดสรร heap + เป็นเจ้าของข้อมูล ต่างจาก `&T` ที่แค่ยืมดูไม่เป็น
  เจ้าของ
- `Box<T>` เกิดมาแก้ปัญหา **recursive type ที่มีขนาดไม่จำกัด** (`E0072`) เพราะตัวมันเองมีขนาดคงที่เท่ากับ pointer
  หนึ่งตัวเสมอ ไม่ว่าข้อมูลข้างในจะซ้อนลึกแค่ไหนก็ตาม
- `*box_value` และ auto-deref ของ method call ทั้งคู่ขับเคลื่อนด้วย trait `Deref` (`fn deref(&self) ->
  &Self::Target`) — กลไกเดียวกับที่ทำให้ `&String` แปลงเป็น `&str` อัตโนมัติตั้งแต่ Part 8
- `Drop` ของ `Box<T>` ทำลายทั้ง pointer และข้อมูลบน heap พร้อมกันเสมอ ตามจังหวะ scope ธรรมดา ไม่ต้องพึ่ง garbage
  collector — ให้ deterministic destruction ที่ C++ `std::unique_ptr<T>` (ไม่ใช่ `shared_ptr`) เทียบเคียงได้ตรงที่สุด
- กรอบการตัดสินใจสามข้อ (ขนาดไม่รู้แน่นอน, ย้ายข้อมูลใหญ่อย่างประหยัด, เป็นเจ้าของค่าที่รู้แค่ trait) ช่วยเลือกได้
  อย่างมีเหตุผลว่าเมื่อไหร่ควรใช้ `Box<T>` จริง ๆ ไม่ใช่ใช้แบบสุ่มหรือเผื่อไว้

บทถัดไป (**Part 28: Smart Pointers `Rc<T>` และ `RefCell<T>`**) จะต่อยอดจากทุกอย่างที่เรียนในบทนี้โดยตรง: `Rc<T>`
implement `Deref` แบบเดียวกับ `Box<T>` เพียงเปลี่ยนความหมายเรื่อง ownership จาก "เจ้าของเดียว" เป็น "หลายเจ้าของ
ร่วมกัน (shared ownership)" ผ่านการนับ reference count ส่วน `RefCell<T>` จะแก้ปัญหาอีกมุมหนึ่งที่ borrow checker
ตอน compile time ยังไม่ยืดหยุ่นพอ ด้วยการย้ายการเช็ค borrow rule ไปทำตอน runtime แทน (**interior mutability**) —
ทั้งสองอย่างนี้มักถูกใช้ร่วมกันเป็น `Rc<RefCell<T>>` เพื่อแก้ปัญหา self-referential struct ที่ Part 23 เกริ่นไว้
ตั้งแต่หัวข้อ 23.9 ว่า "lifetime ธรรมดาไปไม่ถึง" — เตรียมความเข้าใจเรื่อง `Deref`/`Drop`/ownership จากบทนี้ให้แน่น
เพราะทุกอย่างจะถูกนำกลับมาใช้ทันทีในบทถัดไป

---

**Part ก่อนหน้า:** [Iterators ขั้นสูง](part-026-iterators-advanced.md) | **Part ถัดไป:** [Smart Pointers:
Rc<T> และ RefCell<T>](part-028-smart-pointers-rc-refcell.md)
