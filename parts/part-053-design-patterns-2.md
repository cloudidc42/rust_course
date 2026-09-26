# Part 53: Design Patterns ใน Rust (Newtype, Typestate, RAII)

> โมดูล: Design Patterns และสถาปัตยกรรมโค้ด | ระดับ: สูง | เวลาโดยประมาณ: 230 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายและใช้ **newtype pattern** ได้ครบทั้งสามเหตุผลหลัก ไม่ใช่แค่แก้ orphan rule (ที่ Part 21 สอนไว้เบื้องต้น
  แล้ว) แต่รวมถึง **type safety** (ป้องกันการสลับค่าที่มี underlying type เดียวกันโดยไม่ตั้งใจ พร้อมอ่าน error
  จริง), **encapsulation/invariant enforcement** (สร้าง type ที่ "การมีอยู่ของค่า" เท่ากับ "การันตีว่าค่านั้นถูก
  ต้องแล้วเสมอ" ผ่าน private field + validating constructor) และรู้ต้นทุนด้าน ergonomics ที่ต้องแลกมา (การเข้าถึง
  ผ่าน `.0` หรือการ implement `Deref`) พร้อมรู้ว่าเมื่อไหร่ควรจ่ายต้นทุนนั้น
- ออกแบบและเขียน **typestate pattern** เต็มรูปแบบ: ใช้ marker type (zero-sized type ผสาน `PhantomData` จาก
  Part 22) เป็น generic parameter แทนสถานะของ object แล้วทำให้ compiler **ปฏิเสธการเรียก method ผิดสถานะตั้งแต่
  ก่อนโปรแกรมจะรันเลย** อ่านและอธิบาย compile error จริง `E0599` ที่เกิดขึ้นเมื่อพยายามฝ่ากฎนี้ได้อย่างแม่นยำ
- อธิบายกรอบการตัดสินใจ (decision framework) ว่าเมื่อไหร่ typestate คุ้มค่า (จำนวนสถานะน้อย ลำดับการเปลี่ยนสถานะ
  ชัดเจนตายตัว) และเมื่อไหร่ควรกลับไปใช้ runtime state machine แบบ enum ที่เรียนมาจาก Part 10 แทน (สถานะจำนวนมาก
  หรือลำดับการเปลี่ยนสถานะขึ้นกับข้อมูล runtime ที่รู้ไม่ได้ตอน compile)
- เรียกชื่อและอธิบายกลไกที่คุณ**ใช้มาโดยไม่รู้ตัวชื่อเต็มมาตั้งแต่ Part 6**อย่างเป็นทางการ: **RAII (Resource
  Acquisition Is Initialization)** — ผูกอายุของ resource เข้ากับ scope ของตัวแปรผ่าน `Drop` เพื่อให้ cleanup
  เกิดขึ้นอัตโนมัติและแน่นอน (deterministic) เทียบกับภาษาที่มี garbage collector (เวลา cleanup ไม่แน่นอน) และ
  ภาษา C (ต้องจำเรียก free/close เอง)
- เขียน RAII guard ของตัวเองสองแบบที่ใช้งานได้จริง: **`TempFile`** (ลบไฟล์ชั่วคราวอัตโนมัติ แม้ระหว่าง panic
  unwinding) และ **`Timer`/`ScopeGuard`** (วัดเวลาและรัน cleanup closure อัตโนมัติตอนหลุด scope) พร้อมอธิบายว่า
  ทำไม `JoinHandle<T>` **ไม่ใช่** ตัวอย่างของ RAII ทั้งที่ดูคล้ายกันมาก
- ผสานสามรูปแบบ (newtype + typestate + RAII) เข้าด้วยกันในตัวอย่างเดียว พร้อมเจอและแก้ปัญหาจริงที่เกิดขึ้นเมื่อ
  ผสานสองแนวคิดนี้ (`E0366`, `E0509`) เพื่อเข้าใจว่าทั้งสามรูปแบบเป็น "เครื่องมือที่ทำงานร่วมกันได้" ไม่ใช่คู่แข่ง
  กันเอง

## ความรู้ที่ต้องมีมาก่อน

- **Part 52 (Design Patterns ใน Rust: Builder, Strategy, Observer) — จำเป็นในฐานะครึ่งแรกของ mini-arc นี้**:
  Part 52 แนะนำแนวคิด "design pattern" ในบริบทของ Rust ผ่านสามรูปแบบที่ **ดัดแปลงมาจากหนังสือ Gang of Four
  (OOP ดั้งเดิม)** — Builder, Strategy, Observer — โดยเฉพาะหัวข้อ **52.6 "Typestate-adjacent Builder"** ที่
  เกริ่นกลไกของ typestate ไว้แบบง่าย ๆ ก่อนแล้ว (ใช้ type parameter บอกสถานะ "ยังไม่ครบ"/"ครบแล้ว" ของ builder)
  พร้อมบอกตรง ๆ ว่า **Part 53 จะสอนเทคนิคนี้แบบเต็มรูปแบบ** — บทนี้คือคำตอบเต็มรูปแบบของทีเซอร์นั้น ถ้ายังไม่ได้
  อ่าน Part 52 ควรอ่านก่อน เพราะบทนี้จะไม่ทวนความแตกต่างระหว่าง pattern แบบ "ดัดแปลงจาก OOP" กับแบบ "Rust แท้ ๆ"
  ซ้ำอีกในรายละเอียด
- **Part 21 (Traits ขั้นสูง) — จำเป็นที่สุดสำหรับหัวข้อ newtype**: หัวข้อ 21.8 ของ Part 21 สอน **newtype pattern**
  ไปแล้วในฐานะทางแก้ **orphan rule** (พร้อม error `E0117` จริง) และ**ทีเซอร์ไว้ตรง ๆ**ว่า "newtype ยังใช้แก้ปัญหา
  อื่นได้อีกมาก... รูปแบบเหล่านี้จะเป็นเนื้อหาหลักแบบเต็มรูปแบบใน Part 53" — **บทนี้คือคำตอบเต็มรูปแบบของทีเซอร์
  นั้น** เราจะทวนกรณี orphan rule แบบสั้น ๆ เท่านั้น (ไม่สอนซ้ำเต็มรูปแบบ) แล้วขยายไปสู่เหตุผลอีกสองข้อที่ Part 21
  ยังไม่ได้ลงรายละเอียด
- **Part 22 (Generics ขั้นสูง) — จำเป็นที่สุดสำหรับหัวข้อ typestate**: `PhantomData<T>` ที่ Part 22 หัวข้อ 22.7
  สอนไว้ (zero-sized type ที่ใช้ "หลอก" compiler ว่า type parameter ถูกใช้งานอยู่จริง โดยไม่กิน memory เพิ่มเลย)
  คือกลไกสำคัญที่ทำให้ typestate pattern เขียนได้จริงในทางปฏิบัติ ถ้าจำหัวข้อ 22.7 ไม่ได้ ควรย้อนทวนก่อน
- **Part 9 (Structs)**: newtype ทุกรูปแบบในบทนี้คือ tuple struct ที่มี field เดียว (`struct Meters(f64)`) และ
  หัวข้อ encapsulation ต้องใช้ความเข้าใจเรื่อง **field privacy** (private field เข้าถึงได้แค่ในโมดูลที่นิยาม
  struct นั้น) จาก Part 9 โดยตรง
- **Part 3 (ตัวแปร ชนิดข้อมูลพื้นฐาน)**: หัวข้อ 3.12 สอน `TryFrom`/`TryInto` ไว้แล้วในฐานะการแปลง type ตัวเลข
  แบบปลอดภัยที่คืนค่าเป็น `Result` — บทนี้จะขยายการใช้ `TryFrom` ไปสู่การ**สร้าง type ที่การันตี invariant** ผ่าน
  validating constructor ซึ่งเป็นการใช้งาน `TryFrom` ที่ทรงพลังกว่าการแปลงตัวเลขมาก
- **Part 12 (Result และ Error Handling)**: การเขียน `TryFrom` ที่คืน `Result<Self, Error>` พร้อม custom error
  type ใช้ทุกอย่างที่ Part 12 สอนไว้เรื่อง `Result<T, E>`, การออกแบบ error type ของตัวเอง, และ `impl
  std::error::Error`
- **Part 10 (Enums และ Pattern Matching)**: ตัวอย่าง state machine ของออเดอร์สั่งซื้อ (`enum Order { Pending,
  Shipped, Delivered, Cancelled }`) ที่ Part 10 ใช้สอนแนวคิด "illegal states unrepresentable" จะกลับมาเป็น
  จุดเทียบเคียงสำคัญที่สุดของหัวข้อ typestate — เพื่อให้เห็นความแตกต่างระหว่าง "ตรวจสถานะตอนรันจริงผ่าน `match`"
  (Part 10) กับ "ห้ามเรียกผิดสถานะตั้งแต่ compile time" (typestate ในบทนี้)
- **Part 6 (Ownership เบื้องต้น)**: หัวข้อ 6.5 และ 6.12 ของ Part 6 สอน `Drop` และเรียกชื่อแนวคิด **RAII** ไว้แล้ว
  ตรง ๆ ("Ownership ของ Rust คือ RAII ที่ compiler บังคับใช้จริง") — บทนี้จะไม่สอน `Drop` ใหม่ตั้งแต่ต้น แต่จะ
  รวบรวมทุกจุดที่คุณใช้ RAII มาแล้วแบบ pragmatic (Part 6, 27-29, 39) ให้เห็นเป็นภาพเดียวกัน แล้วขยายไปสร้าง RAII
  guard ของตัวเอง
- **Part 27-29 (Smart Pointers: Box, Rc/RefCell, Weak/Cow)**: `Box<T>` (Part 27) ปลดปล่อย heap memory อัตโนมัติ
  ตอน Drop, `Rc<T>`/`RefCell<T>` (Part 28) ลดตัวนับอัตโนมัติ, ทั้งหมดนี้คือตัวอย่างของ RAII ที่คุณเห็นมาแล้ว
- **Part 39 (Mutex, Arc และ Shared-State Concurrency)**: หัวข้อ 39.4 ตั้งชื่อ `MutexGuard<T>` ตรง ๆ ว่าเป็น
  "smart pointer แบบ RAII" — ปลดล็อกอัตโนมัติตอนหลุด scope ไม่ว่าจะจบแบบปกติหรือระหว่าง panic unwinding — บทนี้
  จะอธิบายว่า "ทำไม" กลไกนี้ทำงานได้แม้ระหว่าง panic ด้วยตัวอย่างของเราเอง
- **Part 37-38 (Threads, Channels)**: `JoinHandle<T>` จาก Part 37 จะกลับมาเป็นตัวอย่าง **ตรงข้าม** ของ RAII —
  การ `drop()` ทิ้ง `JoinHandle` ไม่ได้ทำให้ thread ลูกถูก join หรือถูกฆ่าเลย ต่างจาก `Box`/`MutexGuard` อย่าง
  สิ้นเชิง

## เนื้อหา

### 53.1 ภาพรวม: สามรูปแบบที่ "เป็น Rust" ไม่ใช่แค่แปลจากหนังสือ OOP

Part 52 แนะนำ Builder, Strategy, Observer ในฐานะ pattern ที่ **ดัดแปลง**มาจากหนังสือ "Design Patterns: Elements
of Reusable Object-Oriented Software" (Gang of Four, 1994) ให้เข้ากับไวยากรณ์ของ Rust — pattern เหล่านั้นมีต้นกำเนิด
มาจากโลกของ class inheritance และ virtual method ของภาษา OOP ดั้งเดิม เพียงแค่ Rust เอามาปรับใช้ผ่าน trait และ
ownership แทน

สามรูปแบบในบทนี้ **ต่างออกไปโดยพื้นฐาน**: **Newtype**, **Typestate**, และ **RAII** ไม่ได้ถูกคิดค้นขึ้นในหนังสือ
Gang of Four เลยแม้แต่รูปแบบเดียว — มันเป็นรูปแบบที่**เกิดขึ้นเองโดยธรรมชาติจากกลไกหลักของ Rust** ที่คุณเรียนมา
ตลอดทั้งหลักสูตร:

- **Newtype** เกิดจากการผสมกันของ **tuple struct** (Part 9) + **trait system** (Part 19/21) + **zero-cost
  abstraction** (Part 18) — มันไม่มีความหมายในภาษาที่ไม่มี trait หรือไม่สนใจ zero-cost wrapping เลย
- **Typestate** เกิดจากการผสมกันของ **ownership แบบ consuming self** (Part 6/9) + **generics** (Part 18/22) +
  **monomorphization** (Part 18) — มันใช้ประโยชน์จากข้อเท็จจริงที่ว่า Rust ตรวจสอบ type อย่างเข้มงวดที่สุด
  ตอน compile time และ generic แต่ละตัวจะถูกสร้างเป็นโค้ดแยกกันจริง (ไม่ได้ "แค่" ใส่ label ไว้เฉย ๆ)
- **RAII** เกิดจาก **ownership กฎข้อ 3 และ `Drop`** (Part 6) โดยตรง — มันคือชื่อที่ยืมมาจาก C++ (ที่คิดค้นแนวคิดนี้
  ขึ้นก่อน) แต่ Rust เป็นภาษาที่**บังคับใช้แนวคิดนี้ผ่าน compiler อย่างเข้มงวดที่สุด** (ทวนจาก Part 6.12) ไม่ใช่
  แค่ "ธรรมเนียมที่โปรแกรมเมอร์ควรทำตาม" แบบใน C++

สังเกตด้วยว่าทั้งสามรูปแบบนี้ **ทำงานร่วมกันได้อย่างเป็นธรรมชาติ** — ปลายบทเราจะเห็นตัวอย่างที่ทั้งสามอย่างปรากฏ
อยู่ในโค้ดชุดเดียวกันพร้อมกัน (พร้อมปัญหาจริงที่ต้องแก้เมื่อผสานมันเข้าด้วยกัน) นี่คือสิ่งที่ทำให้สามรูปแบบนี้
ได้รับการยกให้เป็น **"Rust idioms"** มากกว่าจะเรียกว่า "design pattern" แบบดั้งเดิม — มันไม่ใช่เทคนิคที่เลือกใช้
เฉพาะบางสถานการณ์ แต่เป็นวิธีคิดที่แทรกซึมอยู่ในโค้ด Rust ระดับ production เกือบทุกที่ที่คุณจะเจอ

### 53.2 ทวนสั้น ๆ จาก Part 21: Newtype แก้ Orphan Rule

Part 21 หัวข้อ 21.8 สอนไว้แล้วว่า **orphan rule** ห้าม `impl TraitX for TypeY` ถ้าทั้ง `TraitX` และ `TypeY` ไม่ได้
เป็นของ crate ปัจจุบันเลยแม้แต่อย่างเดียว — และทางแก้คือ**ห่อ type ภายนอกด้วย tuple struct ของเราเอง** เพื่อให้ได้
"type ที่เป็นของ crate นี้จริง ๆ" มา `impl` ให้ ทวนโค้ดสั้น ๆ:

```rust
// 53.2 - ทวนจาก Part 21: newtype แก้ orphan rule (E0117)
use std::collections::HashMap;
use std::fmt;

// InventoryReport เป็น local type (ของ crate นี้) แม้ข้างในจะห่อ HashMap<String, u32> ที่เป็นของ std ก็ตาม
// orphan rule มองที่ "type ตรงกลาง impl TraitX for TypeY" เท่านั้น ไม่มองลึกเข้าไปในโครงสร้างข้อมูลภายใน
struct InventoryReport(HashMap<String, u32>);

impl fmt::Display for InventoryReport {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        for (name, qty) in &self.0 {
            writeln!(f, "{name}: {qty} ชิ้น")?;
        }
        Ok(())
    }
}

fn main() {
    let mut map = HashMap::new();
    map.insert("apple".to_string(), 10);
    let report = InventoryReport(map);
    print!("{report}");
}
```

ผลลัพธ์:

```
apple: 10 ชิ้น
```

นี่คือ**การใช้งาน newtype แบบที่ 1 จากสามแบบที่บทนี้จะสอน**: **แก้ปัญหา orphan rule** — ถ้ายังไม่มั่นใจในกลไก
ของ orphan rule เอง ควรย้อนไปอ่าน Part 21 หัวข้อ 21.8 ก่อน เพราะบทนี้จะไม่อธิบายกลไก `E0117` ซ้ำอีก แต่จะโฟกัสไป
ที่**อีกสองเหตุผล**ที่ Part 21 ทีเซอร์ไว้แต่ยังไม่ได้สอนเต็มรูปแบบ

### 53.3 Newtype เหตุผลที่ 1: Type Safety — ป้องกันการสลับค่าที่มี Underlying Type เดียวกัน

ลองนึกภาพระบบ e-commerce ที่มี `UserId` และ `ProductId` ทั้งสองแทนด้วยตัวเลข `u64` เหมือนกัน (เพราะมาจาก
auto-increment ID ของฐานข้อมูล) ถ้าเขียนฟังก์ชันทุกตัวให้รับพารามิเตอร์เป็น `u64` เปล่า ๆ **compiler ไม่มีทางรู้
เลยว่า `u64` ตัวนี้ "ควรจะเป็น" user id หรือ product id** — มันเห็นแค่ว่าเป็น `u64` เหมือนกันทั้งคู่ ลองดูโค้ดที่
มีบั๊กซ่อนอยู่แต่ compile ผ่านสมบูรณ์:

```rust
// 53.3 - ปัญหา: u64 เปล่า ๆ ทำให้สลับ id ผิดชนิดกันได้โดย compiler ไม่รู้ตัวเลย
fn charge_user_plain(user_id: u64, amount_cents: u64) {
    println!("เก็บเงินจาก user #{} จำนวน {} เซนต์", user_id, amount_cents);
}

fn main() {
    let user_id: u64 = 42;
    let product_id: u64 = 42;

    // สลับ argument ผิดที่ — ส่ง product_id เข้าไปในตำแหน่งของ user_id
    // compiler ไม่มีทางรู้เลยว่านี่คือบั๊ก เพราะทั้งสองตัวเป็น u64 เหมือนกันเป๊ะในสายตาของ type system
    charge_user_plain(product_id, 500);
    charge_user_plain(user_id, 500);
}
```

ผลลัพธ์ (compile ผ่านและรันได้ปกติ — **นี่คือปัญหา** ไม่ใช่ข้อดี):

```
เก็บเงินจาก user #42 จำนวน 500 เซนต์
เก็บเงินจาก user #42 จำนวน 500 เซนต์
```

โปรแกรมนี้**ดูเหมือนทำงานถูกต้อง**เพราะ `user_id` และ `product_id` บังเอิญมีค่าเท่ากัน (`42`) ในตัวอย่างนี้ — แต่
ในระบบจริงที่ทั้งสองมีค่าต่างกัน การสลับ argument แบบนี้จะทำให้**เก็บเงินจาก user คนละคนกับที่ตั้งใจ** ซึ่งเป็นบั๊ก
ที่ร้ายแรงมากในระบบธุรกิจจริง และที่แย่ที่สุดคือ **compiler ไม่มีทางเตือนคุณได้เลย** เพราะในเชิง type แล้ว
`charge_user_plain(product_id, 500)` ถูกต้อง 100% — `product_id` ก็เป็น `u64` เหมือน `user_id` ทุกประการ

**ทางแก้ด้วย newtype**: ห่อ `u64` แต่ละความหมายด้วย tuple struct คนละตัวกัน ทำให้มันเป็น**คนละ type กัน**ในสายตา
ของ compiler ทั้งที่ข้างในเก็บข้อมูลชนิดเดียวกันเป๊ะ:

```rust
// 53.3 - แก้ด้วย newtype: UserId และ ProductId เป็นคนละ type แม้ทั้งคู่ห่อ u64 เหมือนกัน
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
struct UserId(u64);

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
struct ProductId(u64);

fn charge_user(user: UserId, amount_cents: u64) {
    println!("เก็บเงินจาก user #{} จำนวน {} เซนต์", user.0, amount_cents);
}

fn main() {
    let user = UserId(42);
    let product = ProductId(42);

    charge_user(user, 500);
    // ผิดพลาด: ส่ง ProductId เข้าไปในตำแหน่งที่ต้องการ UserId — แม้ทั้งคู่ห่อ u64 เหมือนกัน
    charge_user(product, 500);
}
```

Error จริงจาก compiler:

```
error[E0308]: mismatched types
  --> src/main.rs:17:17
   |
17 |     charge_user(product, 500);
   |     ----------- ^^^^^^^ expected `UserId`, found `ProductId`
   |     |
   |     arguments to this function are incorrect
   |
note: function defined here
  --> src/main.rs:7:4
   |
 7 | fn charge_user(user: UserId, amount_cents: u64) {
   |    ^^^^^^^^^^^ ------------
```

**อ่าน error นี้ให้ทะลุ**: `expected 'UserId', found 'ProductId'` คือสิ่งที่ `u64` เปล่า ๆ ไม่มีทางให้เราได้เลย
— บั๊กที่เคยเป็น **logic bug ที่ซ่อนอยู่จนกว่าจะรันจริงและมีคนสังเกตเห็นความผิดปกติของข้อมูล** (หรือแย่กว่านั้น
คือไม่มีใครสังเกตเห็นเลย) ถูกเปลี่ยนเป็น **compile error ที่เห็นทันทีตั้งแต่ก่อน commit โค้ดด้วยซ้ำ** ต้นทุนที่
ต้องจ่ายมีแค่การเขียน `struct UserId(u64);` เพิ่มหนึ่งบรรทัด และเข้าถึงค่าจริงผ่าน `.0` — ต้นทุนที่ต่ำมากเมื่อ
เทียบกับความเสี่ยงที่ป้องกันได้ นี่คือเหตุผลที่ในโค้ด Rust ระดับ production จริง คุณจะเห็น `UserId`, `OrderId`,
`ProductId`, `SessionToken` ห่อด้วย newtype อยู่เกือบทุกที่ แทนที่จะใช้ `u64`/`String` เปล่า ๆ ตรง ๆ

สังเกตด้วยว่าตัวอย่างนี้เป็นการประยุกต์ใช้แนวคิดเดียวกับที่ Part 22 หัวข้อ 22.7 สอนผ่าน `Distance<Unit>` (ที่ใช้
`PhantomData<Unit>` แยก `Meters` กับ `Feet` ผ่าน generic parameter ตัวเดียวกัน) — แต่ในกรณีนี้เราเลือกสร้าง
**สอง struct แยกกันไปเลย** (`UserId`/`ProductId`) แทนที่จะใช้ generic parameter ตัวเดียว เพราะ `UserId` และ
`ProductId` **ไม่มีตรรกะร่วมกันที่ควร generic ข้ามกันได้** (ผิดกับ `Distance<Meters>`/`Distance<Feet>` ที่มี
ตรรกะการบวกเหมือนกันทุกหน่วย เพียงแค่ต้องแยกไม่ให้บวกข้ามหน่วยกัน) — **นี่คือหลักการเลือกที่สำคัญ**: ถ้า type
ต่าง ๆ มีตรรกะร่วมกันแต่ต้องแยก "ป้ายกำกับ" ออกจากกัน ใช้ generic + `PhantomData` แบบ Part 22 แต่ถ้า type ต่าง ๆ
เป็นแนวคิดที่**ไม่เกี่ยวข้องกันเลยในเชิงธุรกิจ** (user กับ product เป็นสองสิ่งที่แยกจากกันโดยสิ้นเชิง) ใช้
newtype แยกกันไปเลยตรง ๆ ชัดเจนกว่า

### 53.4 Newtype เหตุผลที่ 2: Encapsulation และ Invariant Enforcement

เหตุผลข้อที่สองของ newtype ทรงพลังกว่าข้อแรกไปอีกขั้น: **ทำให้ "การมีอยู่ของค่า" เท่ากับ "การันตีว่าค่านั้นถูก
ต้องแล้วเสมอ"** แนวคิดนี้เรียกในวงการ Rust ว่า **"parse, don't validate"** — แทนที่จะตรวจสอบความถูกต้องของข้อมูล
ซ้ำ ๆ ทุกจุดที่ใช้งาน (validate) เราตรวจสอบ**ครั้งเดียวตอนแปลงข้อมูลดิบเป็น type ที่ปลอดภัย** (parse) แล้วให้
type system การันตีความถูกต้องนั้นไปตลอดที่เหลือของโปรแกรม

ลองสร้าง `EmailAddress` ที่รับประกันว่า **ทุกค่าที่มีอยู่จริงในระบบต้องผ่านการตรวจสอบรูปแบบมาแล้วเสมอ** — กุญแจ
สำคัญคือ field ข้างในต้องเป็น **private** (จาก Part 9) และวิธีสร้างค่าต้องมีทางเดียวคือผ่าน constructor ที่
ตรวจสอบ (ในที่นี้ใช้ `TryFrom<&str>` ที่ทวนจาก Part 3/12):

```rust
// 53.4 - EmailAddress: newtype ที่การันตี invariant ผ่าน private field + TryFrom ที่ตรวจสอบ
mod email {
    use std::convert::TryFrom;
    use std::fmt;

    // newtype ห่อ String — field เป็น private (ไม่มี `pub` นำหน้า) เพราะเราต้องการันตี invariant:
    // "ทุกค่า EmailAddress ที่มีอยู่ในระบบต้องผ่านการตรวจสอบรูปแบบมาแล้วเสมอ"
    #[derive(Debug, Clone, PartialEq, Eq)]
    pub struct EmailAddress(String);

    #[derive(Debug, Clone, PartialEq, Eq)]
    pub struct EmailParseError(pub String);

    impl fmt::Display for EmailParseError {
        fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
            write!(f, "รูปแบบอีเมลไม่ถูกต้อง: {}", self.0)
        }
    }

    impl std::error::Error for EmailParseError {}

    impl EmailAddress {
        pub fn as_str(&self) -> &str {
            &self.0
        }
    }

    // TryFrom<&str> คือ "ประตูเดียว" ที่สร้าง EmailAddress ได้ — และประตูนี้ตรวจสอบก่อนเสมอ
    impl TryFrom<&str> for EmailAddress {
        type Error = EmailParseError;

        fn try_from(value: &str) -> Result<Self, Self::Error> {
            let trimmed = value.trim();

            if trimmed.matches('@').count() != 1 {
                return Err(EmailParseError(value.to_string()));
            }

            let mut parts = trimmed.split('@');
            let local = parts.next().unwrap_or("");
            let domain = parts.next().unwrap_or("");

            if local.is_empty() || domain.is_empty() || !domain.contains('.') {
                return Err(EmailParseError(value.to_string()));
            }

            Ok(EmailAddress(trimmed.to_string()))
        }
    }
}

use email::EmailAddress;

fn send_welcome_email(to: &EmailAddress) {
    // ฟังก์ชันนี้ "มั่นใจ" ได้ 100% ว่า to.as_str() เป็นอีเมลที่ถูกรูปแบบแล้วเสมอ
    // เพราะไม่มีทางสร้าง EmailAddress ที่ไม่ผ่านการตรวจสอบได้เลยในระบบนี้
    println!("ส่งอีเมลต้อนรับไปยัง: {}", to.as_str());
}

fn main() {
    match EmailAddress::try_from("someone@example.com") {
        Ok(addr) => send_welcome_email(&addr),
        Err(e) => println!("สร้างไม่สำเร็จ: {e}"),
    }

    match EmailAddress::try_from("not-an-email") {
        Ok(addr) => send_welcome_email(&addr),
        Err(e) => println!("สร้างไม่สำเร็จ: {e}"),
    }

    match EmailAddress::try_from("missing-domain@") {
        Ok(addr) => send_welcome_email(&addr),
        Err(e) => println!("สร้างไม่สำเร็จ: {e}"),
    }
}
```

ผลลัพธ์:

```
ส่งอีเมลต้อนรับไปยัง: someone@example.com
สร้างไม่สำเร็จ: รูปแบบอีเมลไม่ถูกต้อง: not-an-email
สร้างไม่สำเร็จ: รูปแบบอีเมลไม่ถูกต้อง: missing-domain@
```

**อธิบายโค้ดทีละส่วนอย่างละเอียด:**

- **`mod email { ... }`** — เราจงใจใส่ `EmailAddress` ไว้ใน module ย่อย เพื่อให้กฎ **private field** (จาก
  Part 9) มีผลจริง หลักการของ Rust คือ private หมายถึง "เข้าถึงได้แค่ในโมดูลที่นิยาม (และโมดูลลูกของมัน)" — ถ้า
  เราเขียน `EmailAddress` ไว้ที่ระดับบนสุดของไฟล์เดียวกันกับ `main()` โดยไม่มี `mod` ครอบ `main()` ก็จะยังเข้าถึง
  field private ได้ตรง ๆ อยู่ดี (เพราะ `main()` อยู่ในโมดูลเดียวกัน) — การมี `mod email` แยกออกมาคือสิ่งที่ทำให้
  invariant "บังคับใช้ได้จริง" ข้ามขอบเขตโมดูล ไม่ใช่แค่ทฤษฎี
- **`pub struct EmailAddress(String)`** — สังเกตว่า `pub` อยู่หน้า `struct` (struct เองเข้าถึงได้จากนอกโมดูล)
  แต่**ไม่มี** `pub` หน้า `String` ข้างใน — field `.0` จึงเป็น **private** เข้าถึงได้แค่ในโมดูล `email` เท่านั้น
  นี่คือกุญแจของทั้งหมด: โค้ดภายนอกโมดูล `email` **ไม่มีทางสร้าง `EmailAddress` ด้วยตัวเองตรง ๆ ได้เลย** (จะเห็น
  error จริงในหัวข้อกับดักถัดไป)
- **`impl TryFrom<&str> for EmailAddress`** — นี่คือ "ประตูเดียว" ที่สร้างค่าได้ ทวนจาก Part 3 (`TryFrom`/
  `TryInto` สำหรับแปลง type แบบปลอดภัย คืนค่าเป็น `Result`) และ Part 12 (การออกแบบ error type ของตัวเอง พร้อม
  `impl std::error::Error`) — ทุก path ที่นำไปสู่การสร้าง `EmailAddress(trimmed.to_string())` (บรรทัด `Ok(...)`)
  ต้องผ่านการตรวจสอบ `@` ตัวเดียว, local part ไม่ว่าง, domain ไม่ว่างและมี `.` มาก่อนเสมอ **ไม่มีทางลัดใด ๆ เลย**
- **`send_welcome_email(to: &EmailAddress)`** — ฟังก์ชันนี้**ไม่ต้องตรวจสอบรูปแบบอีเมลอีกเลย** เพราะรับประกันได้
  100% แล้วว่าถ้ามีค่า `&EmailAddress` อยู่ในมือ ค่านั้นต้องผ่านการตรวจสอบมาแล้วจาก `TryFrom` เท่านั้น — **นี่คือ
  พลังของ "parse, don't validate"**: ตรวจสอบครั้งเดียวตรงจุดที่ข้อมูลดิบเข้าสู่ระบบ (parse) แล้วให้ type
  `EmailAddress` เป็น "ใบรับรอง" ที่พกไปได้ทุกที่โดยไม่ต้องตรวจซ้ำอีก ต่างจากการใช้ `String` เปล่า ๆ ทุกที่ที่ทุก
  ฟังก์ชันต้อง `validate_email(&str) -> bool` ซ้ำเอง (และมีโอกาสลืม validate บางจุดได้ง่าย ๆ)

เทียบกับ Part 21 หัวข้อ 21.8 ที่ใช้ newtype แก้ orphan rule เท่านั้น จะเห็นว่า**ตัวอย่างนี้ไม่มีปัญหา orphan rule
เกี่ยวข้องด้วยเลย** — `EmailAddress` ไม่ได้ implement trait ภายนอกให้ type ภายนอกแต่อย่างใด มันคือการใช้ newtype
เพื่อ**เหตุผลที่ต่างไปจากเดิมโดยสิ้นเชิง**: ควบคุมว่า "ใครสร้างค่าประเภทนี้ได้บ้าง และต้องผ่านการตรวจสอบอะไรก่อน"

### 53.5 Newtype เหตุผลที่ 3: Extra Trait/Method บน Foreign Type — ต้นทุนด้าน Ergonomics ที่ต้องจ่าย

เหตุผลข้อที่สาม (ที่ Part 21 สอนไปแล้วในกรณี orphan rule) มีอีกมุมที่ควรเจาะลึก: **เมื่อห่อ type ภายนอกด้วย
newtype แล้ว เราจะ "เสีย" method ทั้งหมดของ type ภายในไปทันที** เพราะ `SortedVec<T>` ไม่ใช่ `Vec<T>` อีกต่อไปใน
สายตาของ compiler แม้จะห่อ `Vec<T>` อยู่ข้างในก็ตาม ลองดูตัวอย่าง: newtype ที่รักษา invariant "เรียงลำดับอยู่
เสมอ" รอบ `Vec<T>`:

```rust
// 53.5 - SortedVec<T>: newtype รอบ Vec<T> ที่รักษา invariant "เรียงลำดับเสมอ" ผ่าน insert() ที่ควบคุมเอง
use std::ops::Deref;

pub struct SortedVec<T: Ord>(Vec<T>);

impl<T: Ord> SortedVec<T> {
    pub fn new() -> Self {
        SortedVec(Vec::new())
    }

    // ทางเดียวที่เพิ่มค่าเข้าไปได้ — insert() หาตำแหน่งที่ถูกต้องให้เสมอ เพื่อรักษา invariant "เรียงลำดับแล้วเสมอ"
    pub fn insert(&mut self, value: T) {
        let pos = self.0.partition_point(|x| x < &value);
        self.0.insert(pos, value);
    }
}

// implement Deref (แต่ "ไม่" implement DerefMut) — เปิดให้เรียก method อ่านอย่างเดียวของ Vec<T>
// (เช่น .len(), .iter(), .first(), indexing แบบอ่าน) ได้ตรง ๆ ผ่าน auto-deref โดยไม่ต้องเขียน wrapper
// method ซ้ำเองสักตัว แต่ยังปิดกั้นการแก้ไขข้อมูลข้างในแบบไม่ผ่าน insert() อยู่ (เพราะไม่มี DerefMut)
impl<T: Ord> Deref for SortedVec<T> {
    type Target = Vec<T>;

    fn deref(&self) -> &Vec<T> {
        &self.0
    }
}

fn main() {
    let mut v: SortedVec<i32> = SortedVec::new();
    v.insert(5);
    v.insert(1);
    v.insert(3);

    // เมธอดเหล่านี้ไม่ได้ประกาศไว้ใน SortedVec เลยแม้แต่ตัวเดียว — ทั้งหมดมาจาก Vec<i32> ผ่าน auto-deref
    println!("len = {}", v.len());
    println!("first = {:?}", v.first());
    print!("สมาชิกทั้งหมดตามลำดับ: ");
    for x in v.iter() {
        print!("{x} ");
    }
    println!();
}
```

ผลลัพธ์:

```
len = 3
first = Some(1)
สมาชิกทั้งหมดตามลำดับ: 1 3 5 
```

**อ่านโค้ดนี้ให้ทะลุ**: ทวนจาก Part 27 หัวข้อ 27.4 ที่อธิบายกลไก `Deref` เจาะลึกไว้แล้ว (auto-deref สำหรับ method
call) — `impl Deref for SortedVec<T>` ทำให้ `v.len()`, `v.first()`, `v.iter()` **ทำงานได้ทั้งหมดโดยไม่ต้องเขียน
wrapper method สักตัวเดียว** compiler จะพยายามหา method `len()` บน `SortedVec<i32>` ก่อน ไม่พบ จึง auto-deref ผ่าน
`Deref::deref()` ไปหา `&Vec<i32>` แล้วพบ `len()` ที่นั่น — นี่คือ**การจ่ายต้นทุน ergonomics เพียงครั้งเดียว**
(เขียน `impl Deref` หนึ่งครั้ง) เพื่อได้ method ทั้งหมดของ `Vec<T>` กลับมาแบบ "ฟรี" โดยไม่ต้องเขียน `pub fn
len(&self) -> usize { self.0.len() }` ซ้ำทุก method

**แต่สังเกตให้ดี — เราจงใจ implement `Deref` เท่านั้น ไม่ implement `DerefMut`** นี่คือจุดที่อันตรายที่สุดของ
เทคนิคนี้ ถ้าเผลอเพิ่ม `DerefMut` เข้าไปด้วย จะเปิดช่องให้ผู้เรียกได้ `&mut Vec<T>` ตรง ๆ แล้วเรียกเมธอดที่**ไม่
ผ่านการตรวจสอบของเรา**เลย เช่น `.push()` ต่อท้ายดื้อ ๆ:

```rust
// 53.5 - อันตราย: DerefMut เปิดช่องให้ทำลาย invariant ของ newtype ได้โดย compiler ไม่ฟ้องอะไรเลย
use std::ops::{Deref, DerefMut};

pub struct SortedVec<T: Ord>(Vec<T>);

impl<T: Ord> SortedVec<T> {
    pub fn new() -> Self {
        SortedVec(Vec::new())
    }

    pub fn insert(&mut self, value: T) {
        let pos = self.0.partition_point(|x| x < &value);
        self.0.insert(pos, value);
    }
}

impl<T: Ord> Deref for SortedVec<T> {
    type Target = Vec<T>;
    fn deref(&self) -> &Vec<T> {
        &self.0
    }
}

// อันตราย: ถ้าเผลอ implement DerefMut ด้วย จะเปิดช่องให้ผู้เรียกได้ &mut Vec<T> ตรง ๆ
// แล้วเรียกเมธอดของ Vec<T> ที่ไม่ผ่านการตรวจสอบตำแหน่งของเราเลย (เช่น .push() ต่อท้ายดื้อ ๆ)
// compiler ไม่ฟ้อง error อะไรเลยเพราะทุกอย่าง "ถูกต้อง" ในเชิง type — แต่ invariant ของ SortedVec พังทันที
impl<T: Ord> DerefMut for SortedVec<T> {
    fn deref_mut(&mut self) -> &mut Vec<T> {
        &mut self.0
    }
}

fn main() {
    let mut v: SortedVec<i32> = SortedVec::new();
    v.insert(3);
    v.insert(1);
    v.insert(2);
    println!("ก่อนแก้ตรง ๆ ผ่าน DerefMut: {:?}", *v);

    // .push() ไม่ใช่ method ของ SortedVec — แต่เรียกผ่าน auto-deref ไปหา &mut Vec<i32> ได้เลย เพราะมี DerefMut
    v.push(0);

    println!(
        "หลังแก้ตรง ๆ ผ่าน DerefMut: {:?}  <- ไม่เรียงลำดับแล้ว! invariant พังโดย compiler ไม่ฟ้องอะไรเลย",
        *v
    );
}
```

ผลลัพธ์ (compile ผ่านสมบูรณ์ — **นี่คือปัญหา**):

```
ก่อนแก้ตรง ๆ ผ่าน DerefMut: [1, 2, 3]
หลังแก้ตรง ๆ ผ่าน DerefMut: [1, 2, 3, 0]  <- ไม่เรียงลำดับแล้ว! invariant พังโดย compiler ไม่ฟ้องอะไรเลย
```

**บทเรียนสำคัญที่สุดของหัวข้อนี้**: `Deref`/`DerefMut` ไม่ใช่แค่ "ความสะดวก" — มันคือ**การเลือกเปิดหรือปิด
ประตูสู่ข้อมูลภายใน** ถ้า newtype ของคุณมี invariant ที่ต้องรักษา (แบบ `SortedVec`, `EmailAddress`, หรือ
`InventoryReport` จาก 53.2) การ implement `DerefMut` (หรือ `pub` field ตรง ๆ) คือการ**ยกเลิก invariant ทั้งหมด
ที่ตั้งใจสร้างไว้โดยสิ้นเชิง** — กฎที่ใช้ได้จริง:

| สถานการณ์ | ควรทำอะไร |
|---|---|
| Newtype ไม่มี invariant ที่ต้องรักษา (ห่อไว้แค่แก้ orphan rule หรือความชัดเจนของ type) | `Deref`+`DerefMut` ได้ทั้งคู่ ปลอดภัย สะดวกสุด |
| Newtype มี invariant ที่ต้องรักษา แต่ยังต้องการให้อ่านข้อมูลข้างในได้สะดวก | `Deref` เท่านั้น (อ่านได้ แก้ไม่ได้ผ่าน auto-deref) |
| Newtype มี invariant ที่ต้องรักษาอย่างเข้มงวดที่สุด (เช่น `EmailAddress`) | ไม่ implement `Deref`/`DerefMut` เลย เขียน accessor method ที่จำเป็นเอง (เช่น `.as_str()`) |

**เมื่อไหร่ต้นทุนของการเขียน `Deref`/wrapper method คุ้มค่าที่จะจ่าย**: เมื่อ type ภายในมี method จำนวนมาก
(เช่น `Vec<T>`, `String`, `HashMap<K, V>`) และคุณต้องการใช้ method ส่วนใหญ่ของมันซ้ำ ๆ โดยไม่อยากเขียน wrapper
ทุกตัว — `Deref` คุ้มค่ามาก ในทางกลับกัน ถ้า type ภายในมี method น้อย (เช่น `String` ที่ใช้แค่ `.len()`/
`.is_empty()` สองตัว) การเขียน wrapper method สองตัวตรง ๆ (`pub fn len(&self) -> usize { self.0.len() }`)
อาจชัดเจนและปลอดภัยกว่าการเปิด `Deref` ทั้งชุดที่อาจเปิดช่องให้เรียก method ที่ไม่อยากให้เรียกโดยไม่ตั้งใจ

### 53.6 สรุปสามเหตุผลของ Newtype Pattern

ก่อนไปต่อที่ typestate มาสรุปภาพรวมของ newtype pattern ที่เรียนไปทั้งหมด (รวมของ Part 21 ด้วย):

| เหตุผล | ปัญหาที่แก้ | ตัวอย่างในบทนี้ | ต้นทุนที่ต้องจ่าย |
|---|---|---|---|
| 1. Orphan rule (Part 21) | ไม่สามารถ `impl` trait ภายนอกให้ type ภายนอกได้ (`E0117`) | `InventoryReport(HashMap<...>)` | ต้อง implement trait ที่ต้องการเอง (ไม่ได้ "ฟรี" มาจาก type ภายใน) |
| 2. Type safety | ค่าที่มี underlying type เดียวกันสลับกันได้โดย compiler ไม่รู้ | `UserId(u64)` vs `ProductId(u64)` | เข้าถึงค่าจริงผ่าน `.0` แทนตัวแปรตรง ๆ |
| 3. Encapsulation/invariant | ไม่มีอะไรการันตีว่าข้อมูลถูกต้องตลอดอายุของค่า | `EmailAddress(String)` + `TryFrom` | ต้องผ่าน constructor ที่ตรวจสอบเสมอ เขียน accessor เอง |
| 4. เพิ่ม method/trait บน foreign type | ต้องการ method เพิ่มที่ type เดิมไม่มี โดยไม่กระทบ type เดิม | `SortedVec<T>(Vec<T>)` | ต้องเลือก `Deref`/wrapper method ตามความเหมาะสมกับ invariant |

สังเกตว่าเหตุผลข้อ 2, 3, 4 ล้วนใช้โครงสร้างเดียวกันเป๊ะ (`struct Name(InnerType)`) เพียงแค่**เจตนา**ในการใช้งาน
ต่างกัน — นี่คือเหตุผลที่ newtype ถูกเรียกว่าเป็น pattern ที่ **"เรียบง่ายที่สุดแต่ทรงพลังที่สุด"** ในโค้ด Rust
จริง แทบไม่มีต้นทุนด้าน runtime เลย (ไม่มี indirection พิเศษ ไม่มี vtable เหมือน `dyn Trait` จาก Part 21 — มัน
เป็นแค่ type ใหม่ที่ compiler ตรวจสอบเข้มงวดกว่า) แต่ให้ผลตอบแทนด้าน correctness สูงมาก

### 53.7 Typestate Pattern: จากทีเซอร์ของ Part 52 สู่เนื้อหาเต็มรูปแบบ

ทวนจาก Part 52: **typestate pattern** คือการใช้ **type system เป็นตัวบังคับลำดับการเรียก method** ของ object
ที่มีสถานะเปลี่ยนแปลงได้ (stateful object) — แนวคิดหลักคือ: **ถ้า object มีสถานะที่ต่างกัน (เช่น "ยังไม่เชื่อม
ต่อ" กับ "เชื่อมต่อแล้ว") ให้ทำให้แต่ละสถานะเป็นคนละ concrete type กันไปเลยในสายตาของ compiler** ไม่ใช่แค่ field
ภายในตัวแปรเดียวกัน ผลลัพธ์คือ: **method ที่เรียกได้เฉพาะบางสถานะจะ "ไม่มีอยู่จริง" บน type ของสถานะอื่นเลย** —
เรียกผิดสถานะ = compile error ทันที ไม่ใช่ runtime panic หรือ `Result::Err`

ก่อนเข้าโค้ดจริง มาดูว่าทำไม pattern นี้ถึงจำเป็น ด้วยการทวนตัวอย่างที่คุณคุ้นเคยที่สุด: **state machine แบบ enum
จาก Part 10**

### 53.8 ทวนจาก Part 10: Enum State Machine ยังเป็น Runtime Check อยู่ดี

Part 10 สอนเรื่อง **"illegal states unrepresentable"** ผ่านตัวอย่าง `enum Order` ที่แต่ละ variant พกข้อมูลเฉพาะ
ของสถานะนั้นเท่านั้น (`Pending`, `Shipped { tracking_number }`, `Delivered { date }`) แก้ปัญหา "struct ที่มี
`bool` หลายตัวสร้างสถานะที่ไม่มีความหมายได้" ไปแล้วอย่างสมบูรณ์ในเชิง**ข้อมูล** — แต่ทวนโค้ดเดิมอีกครั้งพร้อม
สังเกตจุดที่ Part 10 ยังไม่ได้แก้:

```rust
// 53.8 - ทวนจาก Part 10: enum ทำให้ "สถานะที่ไม่มีความหมาย" เขียนไม่ได้ในเชิงข้อมูล
// แต่สังเกตให้ดี: ship() และ deliver() ทั้งสองเมธอดยังคง "มีอยู่จริง" บน Order ทุกสถานะเหมือนกันหมด
// การห้ามเรียกผิดลำดับ (เช่น deliver() ก่อน ship()) ยังต้องตรวจสอบตอน "รันจริง" ผ่าน match แล้วคืน Err เท่านั้น
// — ไม่มีอะไรห้าม caller ไม่ให้ "เรียก" deliver() บน Order::Pending ตั้งแต่ตอน compile เลย
#[allow(dead_code)]
#[derive(Debug)]
enum Order {
    Pending,
    Shipped { tracking_number: String },
    Delivered { date: String },
}

impl Order {
    fn ship(self, tracking_number: String) -> Result<Order, String> {
        match self {
            Order::Pending => Ok(Order::Shipped { tracking_number }),
            other => Err(format!("ไม่สามารถจัดส่งออเดอร์ที่อยู่ในสถานะ {other:?} ได้")),
        }
    }

    fn deliver(self, date: String) -> Result<Order, String> {
        match self {
            Order::Shipped { .. } => Ok(Order::Delivered { date }),
            other => Err(format!("ไม่สามารถยืนยันว่าจัดส่งสำเร็จจากสถานะ {other:?} ได้")),
        }
    }
}

fn main() {
    // เรียกผิดลำดับโดยตั้งใจ: deliver() ก่อน ship() — โค้ดนี้ "compile ผ่าน" สมบูรณ์แบบ ไม่มี error ใด ๆ เลย
    let order = Order::Pending;
    match order.deliver("2026-01-01".to_string()) {
        Ok(o) => println!("สำเร็จ: {o:?}"),
        Err(e) => println!("ผิดพลาดตอนรันจริง (runtime check): {e}"),
    }

    // ในทางกลับกัน ถ้าเรียกตามลำดับที่ถูกต้อง (ship() ก่อน deliver()) ก็ผ่านได้ตามปกติ
    let order2 = Order::Pending;
    let order2 = order2.ship("TRACK-123".to_string()).unwrap();
    let order2 = order2.deliver("2026-01-02".to_string()).unwrap();
    println!("สำเร็จ: {order2:?}");
}
```

ผลลัพธ์:

```
ผิดพลาดตอนรันจริง (runtime check): ไม่สามารถยืนยันว่าจัดส่งสำเร็จจากสถานะ Pending ได้
สำเร็จ: Delivered { date: "2026-01-02" }
```

**จุดสำคัญที่สุดของหัวข้อนี้**: บรรทัด `order.deliver(...)` ที่เรียกก่อน `ship()` **compile ผ่านสมบูรณ์** ไม่มี
warning ไม่มี error อะไรเลยจาก compiler — เพราะในสายตาของ compiler `deliver()` เป็น method ที่มีอยู่จริงบน
`Order` **ทุกสถานะเหมือนกันหมด** (ไม่ว่าจะเป็น `Pending`, `Shipped`, หรือ `Delivered` ก็ตาม เพราะทั้งหมดเป็น
`Order` ตัวเดียวกันในเชิง type) การตรวจสอบว่า "เรียกถูกลำดับไหม" เกิดขึ้น**ข้างใน** method ผ่าน `match self { ...
}` — ซึ่งเป็นการตรวจสอบที่เกิดขึ้น**ตอนโปรแกรมรันจริงเท่านั้น** ถ้าโปรแกรมเมอร์คนหนึ่งลืมเช็ค `Result` ที่คืนมา
(หรือใช้ `.unwrap()` อย่างไม่ระมัดระวัง) บั๊กนี้จะไปโป่งตอน production แทนที่จะถูกจับตั้งแต่ตอน compile

คำถามที่บทนี้จะตอบคือ: **เราจะทำให้ `deliver()` "ไม่มีอยู่จริง" บน `Order::Pending` ได้ไหม ในระดับ type เลย ไม่ใช่
แค่ตรวจสอบข้างในด้วย `match`?** คำตอบคือ **typestate pattern** — หัวข้อถัดไป

### 53.9 เปลี่ยนจาก Runtime Check เป็น Compile-Time Check: กลไกหลักของ Typestate

แนวคิดหลักของ typestate คือ: **ใช้ marker type (zero-sized type ไม่มี field เลย จาก Part 22) เป็น generic
parameter ที่แทน "สถานะ" ของ object** แล้วเขียน `impl Type<StateX> { ... }` แยกกันสำหรับแต่ละสถานะ — method
ไหนอยู่ใน `impl` block ของสถานะไหน ก็จะ **"มีอยู่จริง"** เฉพาะบน type ของสถานะนั้นเท่านั้น

มาสร้างตัวอย่างคลาสสิกที่สุดของ pattern นี้: **`Connection`** ที่มีสามสถานะ — `Disconnected` (ยังไม่เชื่อมต่อ),
`Connected` (เชื่อมต่อแล้วแต่ยังไม่ authenticate), และ `Authenticated` (authenticate สำเร็จแล้ว พร้อมส่งข้อมูล)

```rust
// 53.9 - Typestate pattern เต็มรูปแบบ: Connection<State> ที่บังคับลำดับ connect -> authenticate -> send_data
// ผ่าน type system ล้วน ๆ ไม่ใช่ runtime check
use std::marker::PhantomData;

// Marker type สามตัวแทนสามสถานะที่เป็นไปได้ของ Connection — เป็น "unit struct" ไม่มี field เลย (จาก Part 22)
// ไม่มีใครสร้าง instance ของ Disconnected/Connected/Authenticated ขึ้นมาเก็บไว้จริง ๆ เลย
// มันมีหน้าที่เดียวคือเป็น "ป้ายกำกับ" ที่ส่งเข้าไปเป็น type parameter ของ Connection<State> เท่านั้น
pub struct Disconnected;
pub struct Connected;
pub struct Authenticated;

pub struct Connection<State> {
    address: String,
    _state: PhantomData<State>,
}

#[derive(Debug)]
pub struct AuthError(pub String);

// เมธอดที่มีอยู่ "เฉพาะ" ตอนสถานะเป็น Disconnected เท่านั้น — new() และ connect()
impl Connection<Disconnected> {
    pub fn new(address: &str) -> Self {
        Connection {
            address: address.to_string(),
            _state: PhantomData,
        }
    }

    // connect() กิน (consume) ค่า self ที่เป็น Connection<Disconnected> ไปเลย
    // แล้วคืนค่าใหม่เป็น Connection<Connected> — Connection<Disconnected> ตัวเดิมไม่มีอยู่อีกต่อไป
    pub fn connect(self) -> Connection<Connected> {
        println!("[{}] เชื่อมต่อสำเร็จ (ยังไม่ authenticate)", self.address);
        Connection {
            address: self.address,
            _state: PhantomData,
        }
    }
}

// เมธอดที่มีอยู่ "เฉพาะ" ตอนสถานะเป็น Connected เท่านั้น — authenticate()
impl Connection<Connected> {
    pub fn authenticate(self, token: &str) -> Result<Connection<Authenticated>, AuthError> {
        if token == "secret-token" {
            println!("[{}] authenticate สำเร็จ", self.address);
            Ok(Connection {
                address: self.address,
                _state: PhantomData,
            })
        } else {
            Err(AuthError(format!("token ไม่ถูกต้องสำหรับ {}", self.address)))
        }
    }
}

// เมธอดที่มีอยู่ "เฉพาะ" ตอนสถานะเป็น Authenticated เท่านั้น — send_data()
// นี่คือหัวใจของ typestate: send_data ไม่มีอยู่จริงเลยบน Connection<Disconnected> หรือ Connection<Connected>
impl Connection<Authenticated> {
    pub fn send_data(&self, data: &[u8]) {
        println!("[{}] ส่งข้อมูล {} bytes", self.address, data.len());
    }
}

fn main() {
    let conn = Connection::<Disconnected>::new("db.example.com:5432");
    let conn = conn.connect();
    let conn = conn.authenticate("secret-token").expect("authenticate ล้มเหลว");
    conn.send_data(b"SELECT 1");
}
```

ผลลัพธ์:

```
[db.example.com:5432] เชื่อมต่อสำเร็จ (ยังไม่ authenticate)
[db.example.com:5432] authenticate สำเร็จ
[db.example.com:5432] ส่งข้อมูล 8 bytes
```

**อธิบายโค้ดทีละส่วนอย่างละเอียด:**

- **`pub struct Disconnected; pub struct Connected; pub struct Authenticated;`** — สาม **unit struct** ที่ไม่มี
  field เลย ทวนจาก Part 22 หัวข้อ 22.7 ว่านี่คือรูปแบบเดียวกับ `struct Meters;`/`struct Feet;` ที่ใช้เป็น "ป้าย
  กำกับ" — ต่างจาก `Distance<Unit>` ของ Part 22 ที่ใช้ marker type แค่แยก**หน่วยข้อมูล**ที่มีตรรกะเหมือนกัน ในที่
  นี้เราใช้ marker type แยก **"สถานะ"** ที่มี method ต่างกันไปเลยในแต่ละสถานะ — เจตนาต่างกัน แต่กลไกเบื้องหลัง
  (zero-sized marker + `PhantomData`) เหมือนกันเป๊ะ
- **`_state: PhantomData<State>`** — field เดียวที่ทำให้ `State` "ถูกใช้งาน" ใน struct (ผ่านการตรวจสอบ `E0392`
  ที่ Part 22 อธิบายไว้) โดยไม่กิน memory เพิ่มเลยแม้แต่ไบต์เดียว (พิสูจน์ได้ด้วย `std::mem::size_of` แบบเดียวกับ
  ที่ Part 22 ทำกับ `Distance<Unit>`)
- **`impl Connection<Disconnected> { ... }`** — บล็อกนี้มี **แค่** `new()` และ `connect()` — ไม่มี `authenticate()`
  หรือ `send_data()` อยู่ในบล็อกนี้เลย เพราะฉะนั้นสอง method นี้**ไม่มีอยู่จริง**บน `Connection<Disconnected>`
- **`pub fn connect(self) -> Connection<Connected>`** — สังเกตว่า `connect` รับ `self` แบบ **consuming (by
  value ไม่ใช่ `&self`)** — นี่คือรูปแบบเดียวกับ `Order::ship(self, ...)` ของ Part 10 ที่ "กิน" ค่าตัวเองไปสร้าง
  ค่าใหม่ (consuming self, ทวนจาก Part 6/7) เพียงแต่ในที่นี้ค่าใหม่ที่คืนมาเป็น**คนละ concrete type กันเลย**
  (`Connection<Connected>` ไม่ใช่ `Connection<Disconnected>`) — หลังเรียก `connect()` แล้ว `Connection<Disconnected>`
  ตัวเดิม**ไม่มีอยู่อีกต่อไปเลย** (ownership ถูกกินไปสร้างค่าใหม่) ทำให้**เป็นไปไม่ได้เลย**ที่จะเรียก `connect()`
  ซ้ำสองครั้งบนตัวแปรเดิม หรือถือ `Connection<Disconnected>` ไว้ใช้ต่อพร้อมกับ `Connection<Connected>` ที่สร้าง
  จากมัน — คนละเรื่องกับ `Order` ของ Part 10 ที่ `ship()`/`deliver()` ยังคง "มีอยู่จริง" บน `Order` ทุก variant
  เหมือนกันหมด (ต่างกันแค่ผลลัพธ์ตอนรันจริงผ่าน `match`)
- **`authenticate()` คืนค่าเป็น `Result<Connection<Authenticated>, AuthError>`** — นี่คือจุดที่ typestate และ
  `Result` (Part 12) ทำงานร่วมกัน: **การเปลี่ยนสถานะเองอาจล้มเหลวได้** (token ผิด) ซึ่งเป็นเรื่องที่ตรวจสอบได้
  แค่ตอนรันจริงเท่านั้น (เพราะขึ้นกับข้อมูลจาก input ภายนอก) — typestate ควบคุม **"ลำดับการเรียก method"** แต่
  ไม่ได้ทำให้ **"เนื้อหาของการเรียกแต่ละครั้งสำเร็จเสมอ"** สองเรื่องนี้เป็นคนละมิติกัน ใช้ `Result` คู่กับ
  typestate ได้อย่างเป็นธรรมชาติ

มาพิสูจน์ข้อกล่าวอ้างที่ว่า `PhantomData<State>` ไม่กิน memory เพิ่มเลย ด้วยวิธีเดียวกับที่ Part 22 หัวข้อ 22.7
ใช้พิสูจน์กับ `Distance<Unit>` — เปรียบเทียบขนาดของ `Connection<State>` ในทุกสถานะที่เป็นไปได้:

```rust
// 53.9 - พิสูจน์ด้วย std::mem::size_of ว่า marker type ของ typestate ไม่กิน memory เพิ่มเลยแม้แต่ไบต์เดียว
use std::marker::PhantomData;

#[allow(dead_code)]
pub struct Disconnected;
#[allow(dead_code)]
pub struct Connected;
#[allow(dead_code)]
pub struct Authenticated;

#[allow(dead_code)]
pub struct Connection<State> {
    address: String,
    _state: PhantomData<State>,
}

fn main() {
    // พิสูจน์ว่า marker type ของ typestate ไม่กิน memory เพิ่มเลยแม้แต่ไบต์เดียว — เหมือนกับที่ Part 22
    // พิสูจน์กับ Distance<Unit> ทุกประการ ไม่ว่า State จะเป็น Disconnected, Connected หรือ Authenticated
    // ขนาดของ Connection<State> ยังเท่ากับขนาดของ String เปล่า ๆ เป๊ะ (PhantomData<State> = 0 ไบต์เสมอ)
    println!(
        "size_of Connection<Disconnected>  = {} bytes",
        std::mem::size_of::<Connection<Disconnected>>()
    );
    println!(
        "size_of Connection<Connected>     = {} bytes",
        std::mem::size_of::<Connection<Connected>>()
    );
    println!(
        "size_of Connection<Authenticated> = {} bytes",
        std::mem::size_of::<Connection<Authenticated>>()
    );
    println!("size_of String alone              = {} bytes", std::mem::size_of::<String>());
}
```

ผลลัพธ์บนเครื่อง 64-bit:

```
size_of Connection<Disconnected>  = 24 bytes
size_of Connection<Connected>     = 24 bytes
size_of Connection<Authenticated> = 24 bytes
size_of String alone              = 24 bytes
```

ตัวเลขนี้ยืนยันสิ่งที่ Part 22 สอนไว้แล้ว: `Connection<State>` มีขนาด **24 ไบต์เท่ากันเป๊ะทุกสถานะ** (เท่ากับ
`String` เปล่า ๆ พอดี เพราะ `String` เก็บ pointer + length + capacity รวม 3 `usize` = 24 ไบต์บนเครื่อง 64-bit) —
**typestate ไม่มีต้นทุนด้าน runtime memory เลยแม้แต่ไบต์เดียว** ทุกอย่างที่ typestate ทำคือการ**ตรวจสอบตอน
compile time เท่านั้น** พอ compile เสร็จแล้ว `PhantomData<State>` จะหายไปจากโค้ดที่ generate ออกมาโดยสิ้นเชิง
เหลือแค่ `Connection` ธรรมดาที่มี field `address: String` หนึ่งตัว — สอดคล้องกับหลักการ **zero-cost
abstraction** ที่ทวนมาตั้งแต่ Part 18 และ Part 21: ความปลอดภัยที่เพิ่มขึ้นจาก type system ไม่ได้แลกมาด้วยต้นทุน
ด้าน performance เลย

### 53.10 พิสูจน์: เรียก `.send_data()` บนสถานะที่ยังไม่ Authenticate — `E0599` จริง

มาดูว่าเกิดอะไรขึ้นถ้าพยายามเรียก `send_data()` บน `Connection<Connected>` (เชื่อมต่อแล้ว แต่ยังไม่ authenticate)
โดยตรง — โค้ดนี้ต้อง**ไม่ compile** เลย:

```rust
// 53.10 - พยายามเรียก send_data() ก่อน authenticate — ต้องไม่ compile
use std::marker::PhantomData;

pub struct Disconnected;
pub struct Connected;
pub struct Authenticated;

pub struct Connection<State> {
    address: String,
    _state: PhantomData<State>,
}

#[derive(Debug)]
pub struct AuthError(pub String);

impl Connection<Disconnected> {
    pub fn new(address: &str) -> Self {
        Connection { address: address.to_string(), _state: PhantomData }
    }

    pub fn connect(self) -> Connection<Connected> {
        Connection { address: self.address, _state: PhantomData }
    }
}

impl Connection<Connected> {
    pub fn authenticate(self, token: &str) -> Result<Connection<Authenticated>, AuthError> {
        if token == "secret-token" {
            Ok(Connection { address: self.address, _state: PhantomData })
        } else {
            Err(AuthError(format!("token ไม่ถูกต้องสำหรับ {}", self.address)))
        }
    }
}

impl Connection<Authenticated> {
    pub fn send_data(&self, data: &[u8]) {
        println!("[{}] ส่งข้อมูล {} bytes", self.address, data.len());
    }
}

fn main() {
    let conn = Connection::<Disconnected>::new("db.example.com:5432");
    let conn = conn.connect(); // ตอนนี้ conn คือ Connection<Connected> — ยังไม่ authenticate

    // ผิดพลาด: send_data ไม่มีอยู่บน Connection<Connected> เลย มีแค่บน Connection<Authenticated> เท่านั้น
    conn.send_data(b"SELECT 1");
}
```

Error จริงจาก compiler:

```
error[E0599]: no method named `send_data` found for struct `Connection<Connected>` in the current scope
  --> src/main.rs:56:10
   |
 7 | pub struct Connection<State> {
   | ---------------------------- method `send_data` not found for this struct
...
56 |     conn.send_data(b"SELECT 1");
   |          ^^^^^^^^^ method not found in `Connection<Connected>`
   |
   = note: the method was found for
           - `Connection<Authenticated>`
```

**อ่าน error นี้ให้ทะลุ — นี่คือหัวใจสำคัญที่สุดของทั้งบท**: สังเกตบรรทัดสุดท้าย `the method was found for -
'Connection<Authenticated>'` — compiler **รู้เอง** ว่า `send_data` มีอยู่จริง เพียงแต่มีอยู่บน type ที่ต่างจาก
`Connection<Connected>` ที่ `conn` เป็นอยู่ในขณะนี้ นี่คือสิ่งที่ Part 21 หัวข้อ 21.4 (object safety) และ error
`E0599` ที่คุณอาจเคยเจอมาก่อนจากการเรียก method ผิด trait bound (Part 22 หัวข้อ 22.2) ทำงานเหมือนกันในเชิง
กลไก — **แต่ในบริบทนี้ มันทำหน้าที่เป็น "state machine validator" ที่ทำงานตอน compile time แทน runtime** ผู้เขียน
โค้ดไม่มีทางส่ง binary ที่มีบั๊กนี้ออกไปให้ production ได้เลย เพราะ**มันไม่ผ่านขั้นตอน `cargo build` ตั้งแต่แรก**

เทียบกับ Part 10 หัวข้อ 53.8 ที่บั๊กแบบเดียวกัน (เรียกผิดลำดับ) compile ผ่านและไปโป่งตอนรันจริงแทน — นี่คือความ
แตกต่างที่สำคัญที่สุดระหว่างสองแนวทาง: **runtime state machine (enum + match) ตรวจจับบั๊กตอนรัน ส่วน typestate
ตรวจจับบั๊กตอน compile** ทั้งสองแนวทาง "ถูกต้อง" ในเชิงการออกแบบ เพียงแค่เหมาะกับสถานการณ์ต่างกัน (รายละเอียด
ในหัวข้อ 53.12)

### 53.11 เชื่อมกับปรัชญาหลักของหลักสูตร: ผลักบั๊กจาก Runtime ไปสู่ Compile Time

ทวนธีมที่ปรากฏซ้ำ ๆ ตลอดทั้งหลักสูตรตั้งแต่ Part 1: Rust เลือกที่จะ**เข้มงวดตอน compile time เพื่อความปลอดภัย
ตอน runtime** — คุณเห็นธีมนี้มาแล้วในหลายรูปแบบ:

- Part 6-7: **borrow checker** ปฏิเสธโค้ดที่อาจมี use-after-free ตั้งแต่ compile time (ต่างจาก C ที่รอให้ crash
  หรือมี undefined behavior ตอนรันจริง)
- Part 10: **enum + exhaustive `match`** ทำให้ "ลืม handle บาง case" กลายเป็น compile error (`E0004`) ไม่ใช่
  บั๊กที่หลุดไปถึง production
- Part 11-12: **`Option<T>`/`Result<T, E>`** ทำให้ "ลืม handle กรณี null/error" ต้องผ่านการตรวจสอบของ compiler
  ก่อนใช้ค่าได้ (ต่างจาก null pointer ของภาษาอื่นที่ crash ตอนรันจริง)
- Part 21: **object safety** ปฏิเสธการเขียนโค้ดที่ compiler ไม่สามารถสร้าง vtable ที่ถูกต้องได้ตั้งแต่ compile
  time

**Typestate คือจุดสูงสุดของธีมนี้เมื่อพูดถึง stateful object**: มันไม่ได้แค่ "เตือน" หรือ "บังคับให้ handle
กรณีที่อาจผิดพลาด" (แบบ `Result`) แต่มัน**ทำให้การเรียก method ผิดสถานะกลายเป็นสิ่งที่เขียนไม่ได้เลยในเชิง
ไวยากรณ์** — เปรียบเทียบง่าย ๆ ได้แบบนี้:

| แนวทาง | บั๊ก "เรียกผิดลำดับ" ถูกจับที่ไหน | ต้นทุนถ้าพลาด |
|---|---|---|
| ไม่มีการตรวจสอบเลย (เช่น struct ธรรมดา + method ทุกตัวเปิดหมด) | ไม่ถูกจับเลย — undefined behavior หรือบั๊กทาง logic เงียบ ๆ | สูงที่สุด — อาจไม่มีใครรู้ตัวจนกว่าจะเกิดความเสียหายจริง |
| runtime check คืน `panic!`/`Result::Err` (Part 10) | ตอนโปรแกรมรันจริง ถ้ามี test/QA/monitoring ที่ดีพอ | ปานกลาง — ต้องมีคนหรือระบบตรวจจับได้ทันเวลา |
| Typestate (บทนี้) | ตอน `cargo build`/`cargo check` — ก่อนแม้แต่จะรันโปรแกรมครั้งแรก | ต่ำที่สุด — ไม่มีทางส่ง binary ที่มีบั๊กนี้ออกไปได้เลย |

นี่คือเหตุผลที่ typestate ถูกใช้อย่างจริงจังในไลบรารีระดับ production ของ Rust จำนวนมาก เช่น `embedded-hal`
(ควบคุม GPIO pin ที่ต้อง configure เป็น input/output ก่อนใช้), HTTP client builder หลายตัว (บังคับต้องตั้ง URL
ก่อน `.send()`), และ parser combinator บางตัว (บังคับลำดับขั้นตอนการ parse)

### 53.12 เมื่อไหร่ Typestate คุ้มค่า เมื่อไหร่ไม่คุ้ม: กรอบการตัดสินใจ

Typestate ไม่ใช่ยาวิเศษที่ควรใช้กับทุก stateful object — มันมีข้อจำกัดที่สำคัญมากสองข้อที่ต้องเข้าใจก่อนเลือกใช้:

**ข้อจำกัดที่ 1: จำนวนสถานะและการเปลี่ยนสถานะขยายแบบ combinatorial** ถ้า object มี N สถานะที่แต่ละสถานะมี
method ที่ overlap กันบางส่วน (ไม่ใช่แยกกันเด็ดขาดเหมือนตัวอย่าง `Connection`) คุณอาจต้องเขียน `impl` block ซ้ำ
กันหลายจุด หรือต้องพึ่งพา trait เพิ่มเติมเพื่อลด duplication — ยิ่งสถานะมากและกฎการเปลี่ยนสถานะซับซ้อน (ไม่ใช่
เส้นตรงเดียวแบบ `Connection`) โค้ดจะยิ่งเทอะทะขึ้นเรื่อย ๆ

**ข้อจำกัดที่ 2 (สำคัญที่สุด): heterogeneous collection ทำไม่ได้ตรง ๆ** เพราะ `Connection<Connected>` และ
`Connection<Authenticated>` เป็น**คนละ concrete type กันเลย** (ทวนจาก Part 21 หัวข้อ 21.1 เรื่อง generic กับ
heterogeneous collection) จึงเก็บไว้ใน `Vec` เดียวกันตรง ๆ ไม่ได้:

```rust
// 53.12 - ข้อจำกัดของ typestate: Connection คนละสถานะเป็นคนละ type กันเลย เก็บใน Vec เดียวกันไม่ได้
use std::marker::PhantomData;

pub struct Disconnected;
pub struct Connected;
pub struct Authenticated;

pub struct Connection<State> {
    address: String,
    _state: PhantomData<State>,
}

#[derive(Debug)]
pub struct AuthError(pub String);

impl Connection<Disconnected> {
    pub fn new(address: &str) -> Self {
        Connection { address: address.to_string(), _state: PhantomData }
    }

    pub fn connect(self) -> Connection<Connected> {
        Connection { address: self.address, _state: PhantomData }
    }
}

impl Connection<Connected> {
    pub fn authenticate(self, token: &str) -> Result<Connection<Authenticated>, AuthError> {
        if token == "secret-token" {
            Ok(Connection { address: self.address, _state: PhantomData })
        } else {
            Err(AuthError(format!("token ไม่ถูกต้องสำหรับ {}", self.address)))
        }
    }
}

fn main() {
    let c1 = Connection::<Disconnected>::new("a.example.com").connect();
    let c2 = Connection::<Disconnected>::new("b.example.com")
        .connect()
        .authenticate("secret-token")
        .expect("authenticate ล้มเหลว");

    // ผิดพลาด: c1 คือ Connection<Connected> ส่วน c2 คือ Connection<Authenticated> — เป็นคนละ concrete type กัน
    // ต่างจาก enum state machine (Part 10) ที่ทุกสถานะยังเป็น "Order" type เดียวกัน เก็บใน Vec<Order> เดียวกันได้
    let connections = vec![c1, c2];
    println!("จำนวน connection: {}", connections.len());
}
```

Error จริง:

```
error[E0308]: mismatched types
  --> src/main.rs:53:32
   |
53 |     let connections = vec![c1, c2];
   |                                ^^ expected `Connection<Connected>`, found `Connection<Authenticated>`
   |
   = note: expected struct `Connection<Connected>`
              found struct `Connection<Authenticated>`
```

ถ้าจำเป็นต้องเก็บ connection ที่อยู่คนละสถานะกันไว้ใน collection เดียว ทางแก้คือใช้ `enum` ครอบอีกชั้น (เช่น
`enum AnyConnection { Disconnected(Connection<Disconnected>), Connected(Connection<Connected>), ... }`) หรือใช้
`Box<dyn Trait>` (Part 21) แต่ทั้งสองทางนี้**เสียจุดแข็งของ typestate ไปบางส่วน** (ต้อง `match`/`match` ผ่าน
`dyn Trait` เพื่อรู้ว่าสถานะไหนอีกครั้ง กลับไปเป็น runtime check อยู่ดี) — นี่คือสัญญาณสำคัญที่บอกว่า **ถ้าคุณ
ต้องเก็บ object หลายสถานะไว้ใน collection เดียวกันบ่อย ๆ typestate อาจไม่ใช่ทางเลือกที่เหมาะสมที่สุด**

**กรอบการตัดสินใจที่ใช้ได้จริง:**

| ใช้ Typestate เมื่อ... | ใช้ Runtime State Machine แบบ Enum (Part 10) เมื่อ... |
|---|---|
| จำนวนสถานะน้อย (2-5 สถานะ) และรู้ล่วงหน้าตายตัวตั้งแต่ตอนออกแบบ | จำนวนสถานะมาก หรือเพิ่ม/ลดสถานะบ่อยตามการเปลี่ยนแปลงของธุรกิจ |
| ลำดับการเปลี่ยนสถานะเป็นเส้นตรงหรือกิ่งก้านที่ชัดเจน รู้ตอน compile time | ลำดับการเปลี่ยนสถานะขึ้นกับข้อมูล runtime ที่รู้ไม่ได้ล่วงหน้า (เช่น อ่านจาก config/database/user input) |
| ไม่ต้องเก็บ object ที่อยู่คนละสถานะไว้ใน collection เดียวกัน | ต้องเก็บ object หลายสถานะปนกันใน `Vec`/`HashMap` เดียวกันบ่อย ๆ (เช่น queue ของออเดอร์ที่มีหลายสถานะปนกัน) |
| ต้องการให้ผู้ใช้ library (API consumer) ไม่มีทางเรียกผิดลำดับได้เลย (compile-time guarantee) | แค่ต้องการตรวจสอบภายในระบบเดียวกัน ที่ทีมเดียวกันดูแลทั้งหมดอยู่แล้ว |
| ตัวอย่างจริง: network protocol (handshake states), parser (token states), builder ที่ต้องตั้งค่าตามลำดับ | ตัวอย่างจริง: order/ticket/workflow ที่มีสถานะจำนวนมาก เปลี่ยนตามการกระทำของผู้ใช้หรือระบบภายนอก |

กฎง่าย ๆ ที่จำได้เสมอ: **typestate เหมาะกับ "โปรโตคอลที่ตายตัว" (protocol) ส่วน enum state machine เหมาะกับ
"เอกสารทางธุรกิจที่มีสถานะเปลี่ยนตามเหตุการณ์" (business object)** — ทั้งสองแนวทางไม่ได้แข่งกัน แต่แก้ปัญหาคน
ละประเภทกัน และในระบบใหญ่จริง คุณอาจใช้ทั้งสองแนวทางในโปรเจกต์เดียวกัน คนละส่วนของระบบ

**ตัวอย่างที่เห็นภาพชัดที่สุดของ "โปรโตคอลที่ตายตัว" คือ Builder pattern เอง** — ทวนจาก Part 52: Builder ทั่วไป
มักตรวจสอบตอน `.build()`/`.send()` ว่า field ที่จำเป็นถูกตั้งค่าแล้วหรือยัง (คืน `Result::Err` หรือ `panic!` ถ้า
ลืม) ซึ่งเป็น **runtime check** ทั้งหมด — เราสามารถผสาน typestate เข้ากับ Builder เพื่อเปลี่ยนการตรวจสอบนี้ให้
เป็น compile-time check ได้ ลองดูตัวอย่าง `RequestBuilder` ที่บังคับว่าต้องตั้ง `.url()` ก่อนจะเรียก `.send()`
ได้:

```rust
// 53.12 - Typestate ผสาน Builder pattern (Part 52): บังคับว่าต้องตั้ง url ก่อน .send() ตั้งแต่ compile time
use std::marker::PhantomData;

pub struct NoUrl;
pub struct HasUrl;

pub struct RequestBuilder<UrlState> {
    url: Option<String>,
    method: String,
    _marker: PhantomData<UrlState>,
}

impl RequestBuilder<NoUrl> {
    pub fn new() -> Self {
        RequestBuilder {
            url: None,
            method: "GET".to_string(),
            _marker: PhantomData,
        }
    }

    // .url() คือ "จุดเปลี่ยนสถานะ" เดียวที่ทำให้ builder ไปจาก NoUrl -> HasUrl ได้
    pub fn url(self, url: &str) -> RequestBuilder<HasUrl> {
        RequestBuilder {
            url: Some(url.to_string()),
            method: self.method,
            _marker: PhantomData,
        }
    }
}

// method() ใช้ได้ทั้งสองสถานะ (ไม่กระทบว่าตั้ง url แล้วหรือยัง) จึงเขียนแบบ generic ครอบทุก UrlState ได้
impl<UrlState> RequestBuilder<UrlState> {
    pub fn method(mut self, m: &str) -> Self {
        self.method = m.to_string();
        self
    }
}

// send() มีอยู่ "เฉพาะ" ตอนสถานะเป็น HasUrl เท่านั้น — เรียกตอนยังไม่ตั้ง url เลยจะได้ E0599 ทันที
impl RequestBuilder<HasUrl> {
    pub fn send(self) -> String {
        format!("{} {}", self.method, self.url.expect("url ต้องมีค่าเสมอในสถานะ HasUrl"))
    }
}

fn main() {
    let request = RequestBuilder::new()
        .method("POST")
        .url("https://api.example.com/orders")
        .send();

    println!("ส่ง request: {request}");
}
```

ผลลัพธ์:

```
ส่ง request: POST https://api.example.com/orders
```

สังเกตว่า `impl<UrlState> RequestBuilder<UrlState>` (ครอบทุกสถานะ) กับ `impl RequestBuilder<NoUrl>`/`impl
RequestBuilder<HasUrl>` (เจาะจงเฉพาะสถานะ) **อยู่ร่วมกันได้ในไฟล์เดียวกันตามปกติ** — method ที่ไม่เกี่ยวกับ
`UrlState` เลย (`.method()`) เขียนแบบ generic ได้ ส่วน method ที่ผูกกับสถานะเฉพาะ (`.url()` เปลี่ยนสถานะ,
`.send()` เรียกได้เฉพาะ `HasUrl`) เขียนแยก `impl` block ตามสถานะนั้น ๆ — ถ้าลองลบ `.url(...)` ออกจาก chain นี้
จะได้ `E0599: no method named 'send' found for struct 'RequestBuilder<NoUrl>'` ทันที ไม่ต่างจากตัวอย่าง
`Connection` เลย นี่คือเหตุผลที่หลาย HTTP client library ระดับ production ในโลก Rust จริง (เช่น builder ของ
`reqwest` บางเวอร์ชัน หรือ builder ที่ทีมภายในองค์กรออกแบบเอง) เลือกใช้แนวทางนี้เพื่อการันตีความถูกต้องของ
request ตั้งแต่ compile time แทนการพึ่งพา runtime panic เพียงอย่างเดียว

### 53.13 RAII: สิ่งที่คุณใช้มาตั้งแต่ Part 6 แต่ยังไม่เคยเรียกชื่อเต็มรูปแบบ

ทวนจาก Part 6 หัวข้อ 6.12 ที่ตั้งชื่อแนวคิดนี้ไว้ตรง ๆ แล้ว: **"Ownership ของ Rust คือ RAII ที่ compiler บังคับ
ใช้จริง"** — **RAII (Resource Acquisition Is Initialization)** คือแนวคิดที่ว่า **"การได้มาซึ่ง resource
(memory, file handle, lock, socket) ต้องเกิดขึ้นพร้อมกับการสร้าง object ที่เป็นเจ้าของ resource นั้น และการ
คืน resource ต้องเกิดขึ้นอัตโนมัติเมื่อ object นั้นหลุด scope"** — ผ่าน `Drop` trait (Part 6.5)

คุณใช้แนวคิดนี้มาแล้วในหลายจุดของหลักสูตรโดยไม่รู้ตัวว่ามีชื่อทางการ:

- **Part 27**: `Box<T>` ปลดปล่อย heap memory อัตโนมัติตอนหลุด scope (`impl Drop for Box<T>` ภายใน)
- **Part 28**: `Rc<T>` ลดตัวนับ (`strong_count`) อัตโนมัติตอนหลุด scope, `RefCell<T>` ปลดล็อก borrow flag
  อัตโนมัติเมื่อ `Ref<T>`/`RefMut<T>` หลุด scope
- **Part 39**: `MutexGuard<T>` **ตั้งชื่อ RAII ไว้ตรง ๆ แล้ว** ในหัวข้อ 39.4 — ปลดล็อก mutex อัตโนมัติเมื่อ
  `MutexGuard<T>` หลุด scope ไม่ว่าจะจบแบบปกติ, จบ block, หรือแม้แต่ตอน panic (unwinding)

**เทียบกับภาษาอื่นให้เห็นภาพชัดเจน**:

| ภาษา/แนวทาง | Cleanup เกิดขึ้นเมื่อไหร่ | ปัญหาที่อาจเกิด |
|---|---|---|
| C (manual memory management) | ต้องเรียก `free()`/`fclose()` เองทุกครั้ง | ลืมเรียก = memory leak, เรียกซ้ำ = double-free, ใช้หลัง free = use-after-free |
| ภาษาที่มี Garbage Collector (Java, Python, Go, JavaScript) | ไม่แน่นอน (nondeterministic) — GC ตัดสินใจเองว่าจะเก็บกวาดเมื่อไหร่ | ไม่รับประกันว่า resource (เช่น file handle) จะถูกปิดตรงเวลา แม้จะมี `finally`/`try-with-resources` ช่วยได้บางส่วนก็ต้องเขียนเอง ไม่ใช่ default |
| Rust (RAII ผ่าน ownership + `Drop`) | **แน่นอน (deterministic)** — ตอน owner หลุด scope เท่านั้น ไม่ว่าจะจบแบบปกติหรือ panic | ไม่มี — compiler บังคับใช้ผ่าน borrow checker (ทวนจาก Part 6.12) |
| C++ (RAII ดั้งเดิม ผ่าน destructor) | แน่นอนเหมือน Rust | แนวคิดเดียวกัน แต่ C++ ไม่มี ownership checker ที่เข้มงวดเท่า Rust — ยังเผลอ double-free/use-after-free ได้ถ้าจัดการ pointer ผิด |

**สิ่งที่ต้องเข้าใจให้แม่นยำที่สุด**: RAII ไม่ใช่แค่ "cleanup อัตโนมัติ" — จุดสำคัญคือ **deterministic** (รู้
ล่วงหน้าแน่นอนว่าเกิดขึ้นตอนไหน คือตอน scope จบ ไม่ใช่ "สักวันหนึ่งที่ GC ว่าง") และ**ทำงานแม้ระหว่าง panic**
(unwinding ทวนจาก Part 12 หัวข้อ unwind vs abort) — เราจะพิสูจน์ข้อนี้ด้วยตัวอย่างจริงในหัวข้อ 53.15

**ตัวอย่างตรงข้ามที่สำคัญ — `JoinHandle<T>` ไม่ใช่ RAII**: ทวนจาก Part 37 — `thread::spawn()` คืนค่าเป็น
`JoinHandle<T>` ซึ่ง**ดูคล้าย** smart pointer แบบ RAII มาก (เป็น handle ที่ "ถือ" thread ลูกไว้) แต่มันทำงาน**ไม่
เหมือน** `Box`/`MutexGuard` เลย:

```rust
// 53.13 - JoinHandle<T> ไม่ใช่ RAII: drop มันไม่ทำให้ thread ลูกถูก join หรือถูกฆ่าเลย
use std::thread;
use std::time::Duration;

fn main() {
    {
        let handle = thread::spawn(|| {
            thread::sleep(Duration::from_millis(60));
            println!("[thread ลูก] ทำงานเสร็จแล้ว");
        });

        // ตั้งใจ drop handle ตรงนี้เลย โดยไม่เรียก .join() — ถ้า JoinHandle เป็น RAII แบบ MutexGuard/Box จริง ๆ
        // เราน่าจะคาดหวังว่ามันจะ "รอ" ให้ thread จบก่อนแล้วค่อยหลุด scope แต่ความจริงไม่ใช่แบบนั้นเลย
        drop(handle);
        println!("[main] scope ของ handle จบแล้ว (drop(handle) ทำงานเสร็จ) แต่ thread ลูกอาจยังไม่ทำงานจบ");
    }

    println!("[main] ออกจาก scope ของ handle ไปแล้ว โปรแกรมยังทำงานต่อได้ตามปกติ ไม่ได้ถูกบล็อกรอ thread ลูกเลย");

    // ให้เวลา thread ลูกทำงานจบเพื่อความเรียบร้อยของ output การสาธิตนี้เท่านั้น
    // (ในโปรแกรมจริง นี่คือความเสี่ยง: thread ลูกกลายเป็น "detached" ไม่มีใครรอหรือดูแลมันอีกเลย)
    thread::sleep(Duration::from_millis(150));
    println!("[main] จบโปรแกรม");
}
```

ผลลัพธ์ (สังเกตลำดับข้อความ):

```
[main] scope ของ handle จบแล้ว (drop(handle) ทำงานเสร็จ) แต่ thread ลูกอาจยังไม่ทำงานจบ
[main] ออกจาก scope ของ handle ไปแล้ว โปรแกรมยังทำงานต่อได้ตามปกติ ไม่ได้ถูกบล็อกรอ thread ลูกเลย
[thread ลูก] ทำงานเสร็จแล้ว
[main] จบโปรแกรม
```

**อ่านผลลัพธ์นี้ให้ทะลุ**: ข้อความ `"[thread ลูก] ทำงานเสร็จแล้ว"` มาปรากฏ**หลังจาก** `drop(handle)` และแม้แต่
หลังข้อความที่บอกว่า "ออกจาก scope ไปแล้ว" ด้วยซ้ำ — ถ้า `JoinHandle` เป็น RAII แบบ `MutexGuard` จริง ๆ (ที่ปลด
ล็อกและรอให้ resource คืนกลับมาสมบูรณ์ก่อนหลุด scope) เราน่าจะคาดหวังให้โปรแกรมรอ thread ลูกจบก่อน — แต่ความจริง
คือ **`Drop` ของ `JoinHandle<T>` ทำแค่ "ตัดการเชื่อมโยง" (detach) ไม่ได้ทำให้ thread ถูก join หรือถูกฆ่าเลย**
thread ลูกยังทำงานต่อไปอย่างอิสระโดยไม่มีใครดูแลมันอีก (documentation ของ `std::thread` ใช้คำว่า **"detached"**
ตรง ๆ) — นี่คือเหตุผลที่ Part 37 สอนไว้ว่าต้องเรียก `.join()` เองเสมอถ้าต้องการรอผลลัพธ์หรือรอให้ thread ทำงาน
จบแน่นอน `Drop` ของ `JoinHandle<T>` **ไม่ได้ทำสิ่งนั้นให้อัตโนมัติ** ต่างจาก `MutexGuard<T>` ที่ `Drop` ของมัน
**ทำหน้าที่หลัก** ของ pattern นี้จริง ๆ (ปลดล็อกให้)

### 53.14 สร้าง RAII Guard ของตัวเอง 1: `TempFile` — ลบไฟล์อัตโนมัติแม้ระหว่าง Panic

มาสร้าง RAII guard ของเราเองตัวแรก: **`TempFile`** — สร้างไฟล์ชั่วคราวตอน construct แล้วลบไฟล์นั้นให้อัตโนมัติ
ตอน `Drop` **ไม่ว่า scope ของมันจะจบแบบปกติ หรือจบเพราะ panic ระหว่างทางก็ตาม** — นี่คือจุดที่เชื่อมกับ Part 12
เรื่อง **unwinding**: เมื่อ `panic!` เกิดขึ้น Rust จะไล่คลาย stack ขึ้นไป (unwind) และเรียก `Drop::drop` ของทุก
ค่าที่ยังเป็นเจ้าของอยู่ในแต่ละ stack frame ที่ไล่ผ่าน — **นี่คือประโยชน์ที่จับต้องได้จริงของ RAII**: แม้โปรแกรม
กำลังจะพัง resource ก็ยังถูกคืนอย่างถูกต้อง ไม่รั่วไหล

```rust
// 53.14 - TempFile: RAII guard ที่ลบไฟล์ชั่วคราวอัตโนมัติ แม้ระหว่าง panic unwinding
use std::fs;
use std::io::Write;
use std::path::{Path, PathBuf};

// RAII guard: สร้างไฟล์ชั่วคราวตอน construct แล้วลบไฟล์นั้นอัตโนมัติตอน Drop
// ไม่ว่า scope ของมันจะจบแบบปกติ หรือจบเพราะ panic (ระหว่าง unwinding) ก็ตาม
struct TempFile {
    path: PathBuf,
}

impl TempFile {
    fn create(name_hint: &str, contents: &str) -> std::io::Result<Self> {
        let filename = format!("rust_course_part53_{}_{name_hint}.tmp", std::process::id());
        let path = std::env::temp_dir().join(filename);
        let mut file = fs::File::create(&path)?;
        file.write_all(contents.as_bytes())?;
        Ok(TempFile { path })
    }

    fn path(&self) -> &Path {
        &self.path
    }
}

impl Drop for TempFile {
    fn drop(&mut self) {
        match fs::remove_file(&self.path) {
            Ok(()) => println!(
                "[Drop] ลบไฟล์ชั่วคราว {:?} สำเร็จ (ไม่ว่าจะจบแบบปกติหรือระหว่าง panic unwinding)",
                self.path
            ),
            Err(e) => println!("[Drop] ลบไฟล์ {:?} ไม่สำเร็จ: {e}", self.path),
        }
    }
}

fn do_work_that_panics() {
    // temp เป็นเจ้าของอยู่ใน stack frame ของฟังก์ชันนี้
    let temp = TempFile::create("demo1", "ข้อมูลชั่วคราว").expect("สร้างไฟล์ไม่สำเร็จ");
    println!("สร้างไฟล์ชั่วคราวที่ {:?} แล้ว", temp.path());
    assert!(temp.path().exists(), "ไฟล์ต้องมีอยู่จริงตอนนี้");

    panic!("เกิดข้อผิดพลาดระหว่างทำงาน! (แต่ temp ต้องยังถูก Drop ระหว่าง unwinding อยู่ดี)");
    // ตั้งแต่บรรทัด panic! ด้านบน โค้ดหลังจากนี้ในฟังก์ชันจะไม่ถูกรันเลย — Rust เริ่มกระบวนการ unwinding
    // ไล่คลาย stack กลับขึ้นไป และจะเรียก Drop::drop ของทุกค่าที่ยังเป็นเจ้าของอยู่ในแต่ละ stack frame
    // ที่ไล่ผ่าน รวมถึง temp ตัวนี้ด้วย — นี่คือสิ่งที่เราจะพิสูจน์ในฟังก์ชัน main()
}

fn main() {
    let expected_path =
        std::env::temp_dir().join(format!("rust_course_part53_{}_demo1.tmp", std::process::id()));

    // catch_unwind ดัก panic ไว้ไม่ให้ทั้งโปรแกรมล่ม (ปกติใช้ในขอบเขตพิเศษ เช่น web server ที่ไม่อยากให้
    // request เดียวที่ panic ทำให้ทั้งเซิร์ฟเวอร์ล่มไปด้วย — ไม่ใช่วิธีจัดการ error ปกติในโค้ดทั่วไป)
    let result = std::panic::catch_unwind(do_work_that_panics);

    assert!(result.is_err(), "คาดว่า do_work_that_panics ต้อง panic");
    println!("[main] จับ panic ได้ด้วย catch_unwind แล้ว โปรแกรมหลักยังทำงานต่อได้ตามปกติ");

    println!(
        "[main] ไฟล์ {:?} ยังอยู่หรือไม่หลัง panic+unwind: {}",
        expected_path,
        expected_path.exists()
    );
    assert!(
        !expected_path.exists(),
        "ไฟล์ควรถูกลบไปแล้วโดย Drop ระหว่าง unwinding"
    );
    println!("[main] ยืนยันแล้ว: RAII (Drop) ทำงานถูกต้องแม้ระหว่าง panic unwinding");
}
```

ผลลัพธ์จริง (รวม panic message ที่ Rust พิมพ์ไปที่ stderr ตามปกติเมื่อ panic เกิดขึ้น แม้จะถูก `catch_unwind`
จับไว้ก็ตาม — `catch_unwind` ดักไม่ให้โปรแกรมจบ แต่ไม่ได้ปิดการพิมพ์ panic message เริ่มต้น):

```
สร้างไฟล์ชั่วคราวที่ "/tmp/rust_course_part53_26945_demo1.tmp" แล้ว

thread 'main' (26945) panicked at src/main.rs:43:5:
เกิดข้อผิดพลาดระหว่างทำงาน! (แต่ temp ต้องยังถูก Drop ระหว่าง unwinding อยู่ดี)
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
[Drop] ลบไฟล์ชั่วคราว "/tmp/rust_course_part53_26945_demo1.tmp" สำเร็จ (ไม่ว่าจะจบแบบปกติหรือระหว่าง panic unwinding)
[main] จับ panic ได้ด้วย catch_unwind แล้ว โปรแกรมหลักยังทำงานต่อได้ตามปกติ
[main] ไฟล์ "/tmp/rust_course_part53_26945_demo1.tmp" ยังอยู่หรือไม่หลัง panic+unwind: false
[main] ยืนยันแล้ว: RAII (Drop) ทำงานถูกต้องแม้ระหว่าง panic unwinding
```

**อ่านลำดับเหตุการณ์นี้ให้ทะลุ**: สังเกตว่าข้อความ `[Drop] ลบไฟล์ชั่วคราว ... สำเร็จ` ปรากฏ**ก่อน**ข้อความ
`[main] จับ panic ได้ด้วย catch_unwind แล้ว` — นั่นเพราะลำดับเหตุการณ์จริงคือ: `panic!` ถูกเรียกข้างใน
`do_work_that_panics()` → Rust เริ่ม unwinding → **ระหว่างที่ไล่คลาย stack กลับขึ้นไปนี่แหละที่ `temp` (ซึ่งเป็น
เจ้าของอยู่ใน stack frame ของ `do_work_that_panics`) ถูก `Drop::drop` เรียกโดยอัตโนมัติ** (พิมพ์ข้อความลบไฟล์) →
unwinding ไล่ไปถึง `catch_unwind` แล้วหยุด (กลายเป็น `Err(...)` ที่ `catch_unwind` คืนกลับมา) → `main()` ทำงาน
ต่อตามปกติ — assertion `!expected_path.exists()` ผ่านสำเร็จ พิสูจน์ว่าไฟล์ถูกลบไปแล้ว**จริง** ก่อนที่
`catch_unwind` จะคืนค่ากลับมาด้วยซ้ำ — นี่คือหลักฐานที่จับต้องได้ว่า **RAII ทำงานถูกต้องแม้ระหว่าง panic
unwinding** ซึ่งเป็นสิ่งที่ `MutexGuard<T>` (Part 39.3) พึ่งพาอยู่เหมือนกันทุกประการ (thread ที่ panic ขณะถือ
lock ยังคงปลดล็อกให้ก่อนที่ mutex จะถูก "poison" — ทวนจาก Part 39 หัวข้อ mutex poisoning)

### 53.15 สร้าง RAII Guard ของตัวเอง 2: `Timer` และ `ScopeGuard`

RAII guard ไม่จำเป็นต้องจัดการ resource ที่ "อันตราย" อย่างไฟล์หรือ lock เสมอไป — มันเป็น pattern ที่มีประโยชน์
มากสำหรับ **"รันโค้ดบางอย่างให้แน่ใจว่าเกิดขึ้นเมื่อหลุด scope"** แม้จะเป็นแค่การพิมพ์ log ก็ตาม มาดูสอง utility
ที่ใช้บ่อยจริงในโค้ด production:

```rust
// 53.15 - Timer (จับเวลาเฉพาะทาง) และ ScopeGuard (defer-style cleanup ทั่วไป)
use std::time::Instant;

// RAII utility ตัวที่ 1: Timer — จับเวลาตอน construct พิมพ์เวลาที่ใช้ตอน Drop
struct Timer {
    label: String,
    start: Instant,
}

impl Timer {
    fn new(label: &str) -> Self {
        println!("[Timer] เริ่มจับเวลา: {label}");
        Timer {
            label: label.to_string(),
            start: Instant::now(),
        }
    }
}

impl Drop for Timer {
    fn drop(&mut self) {
        let elapsed = self.start.elapsed();
        println!("[Timer] '{}' ใช้เวลาไป {elapsed:?}", self.label);
    }
}

// RAII utility ตัวที่ 2: ScopeGuard — รูปแบบทั่วไปกว่า Timer เก็บ closure ไว้เรียกตอน Drop
// (แนวคิดเดียวกับ `defer` ใน Go หรือ crate `scopeguard` ที่ใช้กันจริงในโค้ด production)
struct ScopeGuard<F: FnOnce()> {
    cleanup: Option<F>,
}

impl<F: FnOnce()> ScopeGuard<F> {
    fn new(cleanup: F) -> Self {
        ScopeGuard {
            cleanup: Some(cleanup),
        }
    }
}

impl<F: FnOnce()> Drop for ScopeGuard<F> {
    fn drop(&mut self) {
        // .take() ดึง closure ออกมาเพื่อเรียกแบบ by-value (FnOnce ต้องกินตัวเองไปเรียกครั้งเดียว)
        // เหลือ None ไว้ข้างใน ป้องกันไม่ให้เรียกซ้ำสองครั้งโดยไม่ตั้งใจ
        if let Some(cleanup) = self.cleanup.take() {
            cleanup();
        }
    }
}

fn expensive_computation() -> u64 {
    let _timer = Timer::new("expensive_computation");
    let mut total = 0u64;
    for i in 0..1_000_000u64 {
        total = total.wrapping_add(i);
    }
    total
} // <- _timer หลุด scope ที่นี่ พิมพ์เวลาที่ใช้ไปโดยอัตโนมัติ ก่อนที่ expensive_computation จะ return ค่ากลับไปด้วยซ้ำ

fn main() {
    let result = expensive_computation();
    println!("ผลลัพธ์ = {result}");

    {
        println!("เริ่มทำงานในบล็อกที่มี cleanup แบบ defer");
        let _guard = ScopeGuard::new(|| {
            println!("[ScopeGuard] cleanup ทำงานตอนหลุด scope โดยไม่ต้องเรียกเองเลย");
        });
        println!("ทำงานอื่น ๆ อยู่ในบล็อกนี้ต่อไป...");
    }
    println!("ออกจากบล็อกไปแล้ว");
}
```

ผลลัพธ์ (เวลาที่แสดงจริงจะต่างกันไปในแต่ละเครื่อง/แต่ละครั้งที่รัน):

```
[Timer] เริ่มจับเวลา: expensive_computation
[Timer] 'expensive_computation' ใช้เวลาไป 186ns
ผลลัพธ์ = 499999500000
เริ่มทำงานในบล็อกที่มี cleanup แบบ defer
ทำงานอื่น ๆ อยู่ในบล็อกนี้ต่อไป...
[ScopeGuard] cleanup ทำงานตอนหลุด scope โดยไม่ต้องเรียกเองเลย
ออกจากบล็อกไปแล้ว
```

**อธิบายจุดสำคัญ**: สังเกตว่า `[Timer] 'expensive_computation' ใช้เวลาไป ...` พิมพ์ออกมา**ก่อน**บรรทัด `ผลลัพธ์ =
...` ทั้งที่ `_timer` ถูกสร้างเป็นตัวแปรตัวแรกใน `expensive_computation()` — นี่เพราะ `_timer` หลุด scope ตอน
ฟังก์ชัน `return` (ค่าที่ประกาศทีหลังในฟังก์ชันจะถูก drop ก่อนค่าที่ประกาศก่อนหน้า ตามลำดับย้อนกลับ — แต่ในที่นี้
`_timer` เป็นตัวแปรเดียวที่ต้อง drop) `Drop::drop` ของมันถูกเรียก**ก่อน**ที่ค่า return (`total`) จะถูกส่งกลับไป
ให้ `main()` พิมพ์ต่อ — ลำดับนี้คือสิ่งที่ทำให้ `Timer` วัดเวลาได้ถูกต้องเสมอ ไม่ว่าฟังก์ชันจะ return ผ่านทางไหน
(ปกติ, early return, หรือ `?`) เพราะ `Drop` รับประกันว่าจะถูกเรียกทุกทางที่ scope จะจบ — ต่างจากการเขียน
`let start = Instant::now(); /* ... */ println!("ใช้เวลา {:?}", start.elapsed());` ธรรมดาที่ถ้ามี early return
ระหว่างทาง จะ**ไม่มีทางพิมพ์เวลาที่ใช้ไปเลย** เพราะโค้ดวัดเวลาไม่ได้ถูกวางไว้ที่ "จุดจบของทุก path" แต่ RAII
แก้ปัญหานี้ได้เพราะ `Drop` ทำงานที่ "จุดจบของ scope" เสมอ ไม่ว่า path การ return จะเป็นแบบไหน

`ScopeGuard` คือรูปแบบทั่วไปกว่า `Timer` — มันรับ closure อะไรก็ได้มาเก็บไว้ แล้วเรียกให้ตอนหลุด scope ใช้แนวคิด
เดียวกับ keyword `defer` ของภาษา Go เลย เพียงแค่ Rust ไม่มี keyword พิเศษสำหรับสิ่งนี้ (เพราะ `Drop` + RAII ทำ
หน้าที่นี้ได้อยู่แล้วโดยไม่ต้องมี syntax ใหม่เพิ่ม) — สังเกตการใช้ `Option<F>` + `.take()` ข้างใน `Drop::drop`:
เพราะ `FnOnce` ต้องถูกเรียกแบบ **by value** (กินตัวเองไปเลย เรียกได้ครั้งเดียว) แต่ `Drop::drop(&mut self)` รับ
แค่ `&mut self` ไม่ใช่ `self` — การห่อด้วย `Option<F>` แล้วใช้ `.take()` (ที่สลับ `Some(f)` เป็น `None` แล้วคืน
`Some(f)` ออกมา) คือเทคนิคมาตรฐานที่ใช้ "ย้ายค่าออกจาก field ผ่าน `&mut self`" ได้โดยไม่ผิดกฎ ownership — เทคนิค
นี้จะกลับมาเป็นกุญแจสำคัญอีกครั้งในหัวข้อถัดไปที่เราจะผสาน RAII เข้ากับ typestate

### 53.16 ผสานสามรูปแบบเข้าด้วยกัน: `DatabaseConnection<State>` ที่เป็นทั้ง Newtype + Typestate + RAII

ถึงเวลาพิสูจน์ประโยคที่บอกไว้ในหัวข้อ 53.1: **ทั้งสามรูปแบบทำงานร่วมกันได้** — เราจะสร้าง
`DatabaseConnection<State>` ที่:

1. ใช้ **newtype** (`ConnectionId(u64)`) ป้องกันการสลับ id ของ connection กับตัวเลขอื่นในระบบ
2. ใช้ **typestate** (`Disconnected`/`Connected`) บังคับว่าต้อง `.open()` ก่อนจะ `.query()` ได้
3. ใช้ **RAII** ปิดการเชื่อมต่ออัตโนมัติเมื่อหลุด scope

มาลองเขียนแบบตรงไปตรงมาที่สุดก่อน (ตามสัญชาตญาณจากหัวข้อ 53.9): implement `Drop` ให้เฉพาะสถานะ `Connected`
เพราะมีแค่สถานะนี้ที่ "มีอะไรให้ปิดจริง" (`Disconnected` ยังไม่เคยเปิดอะไรเลย):

```rust
// 53.16 - ความพยายามแรก (ยัง "ไม่" compile): implement Drop ให้เฉพาะ DatabaseConnection<Connected>
use std::marker::PhantomData;

struct Disconnected;
struct Connected;

struct Conn<State> {
    dsn: String,
    _state: PhantomData<State>,
}

// ผิดพลาด: พยายาม implement Drop ให้เฉพาะ Conn<Connected> ตัวเดียว โดยไม่ครอบคลุมทุก State ที่เป็นไปได้
// ของ struct แบบ generic เดียวกัน — Rust ไม่ยอมให้ "เจาะจง" Drop แบบนี้เด็ดขาด
impl Drop for Conn<Connected> {
    fn drop(&mut self) {
        println!("ปิดการเชื่อมต่อ {}", self.dsn);
    }
}

fn main() {
    let _c = Conn::<Connected> {
        dsn: "postgres://localhost".to_string(),
        _state: PhantomData,
    };
}
```

Error จริง:

```
error[E0366]: `Drop` impls cannot be specialized
  --> src/main.rs:13:1
   |
13 | impl Drop for Conn<Connected> {
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: `Connected` is not a generic parameter
note: use the same sequence of generic lifetime, type and const parameters as the struct definition
  --> src/main.rs:6:1
   |
 6 | struct Conn<State> {
   | ^^^^^^^^^^^^^^^^^^
```

**อ่าน error นี้ให้ทะลุ**: Rust มีกฎเข้มงวดว่า **`impl Drop for SomeType<...>` ต้องครอบคลุม generic parameter
เดียวกันกับที่ struct นิยามไว้ทุกตัว** — คุณเขียน `impl<State> Drop for Conn<State>` (ครอบทุกสถานะ) ได้ แต่เขียน
`impl Drop for Conn<Connected>` (เจาะจงแค่สถานะเดียว) ไม่ได้เด็ดขาด เหตุผลเชิงลึกคือ **Rust รับประกันว่าทุกค่า
ของ type หนึ่ง ๆ ต้องมีพฤติกรรม `Drop` เดียวกันเสมอ ไม่ใช่ขึ้นกับ generic parameter ที่ต่างกัน** (ถ้าอนุญาตแบบนี้
ได้ ระบบ trait coherence และการตรวจสอบว่า type ไหน "implement Drop" จะซับซ้อนขึ้นมาก และขัดกับหลักการที่ว่า
"drop glue" ของ generic type ต้องรู้แน่นอนตอน compile time สำหรับทุก instantiation)

ลองแก้แบบเขียน `impl<State> Drop for Conn<State>` ครอบทุกสถานะแทน (ให้ตรวจสอบด้วย field ธรรมดาว่ามีอะไรต้องปิด
จริงไหม แทนการพึ่งพา `State`):

```rust
// 53.16 - ความพยายามที่สอง (ยัง "ไม่" compile): Drop ครอบทุก State แล้ว แต่ methods ที่ต้อง move field ออกจาก self พังหมด
struct Session {
    dsn: String,
}

impl Drop for Session {
    fn drop(&mut self) {
        println!("ปิดการเชื่อมต่อ {}", self.dsn);
    }
}

struct DatabaseConnection {
    session: Session,
    extra: String,
}

// ผิดพลาด: DatabaseConnection ไม่ได้ implement Drop เองตรง ๆ แต่ถ้าเปลี่ยนมา implement Drop ให้
// DatabaseConnection ตรง ๆ (ไม่ใช่แค่ให้ Session ข้างใน) จะย้าย field ออกจาก self แบบนี้ไม่ได้อีกต่อไป
impl Drop for DatabaseConnection {
    fn drop(&mut self) {
        println!("DatabaseConnection กำลังจะถูกทำลาย");
    }
}

impl DatabaseConnection {
    fn into_session(self) -> Session {
        // พยายามย้าย field session ออกจาก self ทั้งก้อน — แต่ตอนนี้ DatabaseConnection implement Drop เอง
        // ทำให้ compiler ห้ามย้าย field ออกแบบนี้ เพราะจะทำให้ค่าที่เหลือของ self ไม่สมบูรณ์ตอนต้องเรียก
        // Drop::drop(&mut self) ถ้าเกิด panic ระหว่างฟังก์ชันนี้ทำงานอยู่
        self.session
    }
}

fn main() {
    let conn = DatabaseConnection {
        session: Session {
            dsn: "postgres://localhost".to_string(),
        },
        extra: "metadata".to_string(),
    };
    let _session = conn.into_session();
}
```

Error จริง:

```
error[E0509]: cannot move out of type `DatabaseConnection`, which implements the `Drop` trait
  --> src/main.rs:29:9
   |
29 |         self.session
   |         ^^^^^^^^^^^^
   |         |
   |         cannot move out of here
   |         move occurs because `self.session` has type `Session`, which does not implement the `Copy` trait
```

**อ่าน error นี้ให้ทะลุ**: เหตุผลที่ compiler ให้มาตรงเผงกับที่ comment อธิบายไว้ — **type ที่ implement `Drop`
เองตรง ๆ จะถูกห้ามไม่ให้ "ย้าย field บางส่วนออกจาก `self`"** เหตุผลเชิงลึกคือ: ถ้าอนุญาตให้ย้าย `self.session`
ออกไปได้ แล้วเกิด panic ขึ้นระหว่างบรรทัดถัดไปในฟังก์ชันเดียวกัน (ก่อน `self` ทั้งก้อนจะหลุด scope) Rust จะต้อง
เรียก `Drop::drop(&mut self)` ของ `DatabaseConnection` ตอน unwinding — แต่ `self.session` ถูกย้ายออกไปแล้ว ทำให้
`self` อยู่ในสถานะที่ "ไม่สมบูรณ์" (partially moved) ซึ่ง `drop()` ไม่สามารถทำงานอย่างปลอดภัยกับค่าที่ไม่สมบูรณ์
แบบนี้ได้เลย — **นี่คือเหตุผลเดียวกับกับดักที่ 6 ในหัวข้อถัดไป** (การเรียก `.drop()` ตรง ๆ ก็ผิดกฎด้วยเหตุผล
คล้ายกัน คือป้องกันสถานะที่ไม่สมบูรณ์/double-drop)

**ทางแก้ที่ถูกต้อง**: แยก **resource ที่ต้อง `Drop` จริง** (`Session`) ออกจาก **wrapper ที่ทำ state transition**
(`DatabaseConnection<State>`) — ให้ wrapper **ไม่** implement `Drop` เองเลย แค่ "ถือ" `Session` ไว้ข้างใน แล้ว
ปล่อยให้ **drop glue อัตโนมัติ** ของ compiler (ไม่ใช่ `impl Drop` ที่เราเขียนเอง) ดูแล `Session` ตอน
`DatabaseConnection<State>` หลุด scope ตามปกติ — เพราะ `DatabaseConnection<State>` เองไม่ได้ implement `Drop`
ตรง ๆ กฎ `E0509` จึง**ไม่มีผล**กับมัน ย้าย field ออกได้อย่างอิสระในทุกเมธอด transition:

```rust
// 53.16 - เวอร์ชันที่ถูกต้อง: แยก Session (ที่ Drop จริง) ออกจาก DatabaseConnection<State> (wrapper ที่ไม่ Drop เอง)
use std::marker::PhantomData;

// Newtype: ConnectionId ห่อ u64 — ป้องกันการสลับ id ของ connection กับตัวเลขอื่น ๆ ในระบบโดยไม่ตั้งใจ
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct ConnectionId(u64);

impl ConnectionId {
    fn generate(counter: &mut u64) -> Self {
        *counter += 1;
        ConnectionId(*counter)
    }
}

// จำลอง handle จริงของ connection (ในโปรแกรมจริงอาจเป็น socket file descriptor หรือ handle จาก driver)
struct RawHandle;

// Session คือส่วนที่ "เป็นเจ้าของ resource จริง" และเป็นที่เดียวที่ implement Drop (RAII)
// แยกออกมาจาก wrapper ที่ทำ state transition โดยเจตนา (ตามที่อธิบายไว้ด้านบน หลังจากลองแบบไม่แยกแล้วพังก่อน)
struct Session {
    id: ConnectionId,
    dsn: String,
    // handle เป็น Some(...) เฉพาะตอนเปิดเชื่อมต่อจริงแล้วเท่านั้น — None เสมอตอนยังไม่เปิด
    handle: Option<RawHandle>,
}

impl Drop for Session {
    fn drop(&mut self) {
        if self.handle.take().is_some() {
            println!(
                "[conn #{}] ปิดการเชื่อมต่อไปยัง {} โดยอัตโนมัติ (RAII ผ่าน Drop)",
                self.id.0, self.dsn
            );
        }
    }
}

// Typestate: สองสถานะของการเชื่อมต่อฐานข้อมูล
pub struct Disconnected;
pub struct Connected;

// DatabaseConnection<State> เอง "ไม่" implement Drop — มันแค่ "ถือ" Session ไว้ข้างใน แล้วปล่อยให้
// drop glue อัตโนมัติของ compiler ดูแล Session ตอน DatabaseConnection หลุด scope ตามปกติ เพราะ
// DatabaseConnection<State> เองไม่ได้ implement Drop ตรง ๆ เราจึง "ย้าย" field session ออกจาก self
// ได้อย่างอิสระในทุกเมธอด transition (เช่น open()) — นี่คือกุญแจที่ทำให้ typestate (ซึ่งต้องกิน self
// แล้วสร้างค่าใหม่) และ RAII (ซึ่งต้องมี Drop) อยู่ร่วมกันได้จริงโดยไม่ชนกัน
pub struct DatabaseConnection<State> {
    session: Session,
    _state: PhantomData<State>,
}

impl DatabaseConnection<Disconnected> {
    pub fn new(id: ConnectionId, dsn: &str) -> Self {
        DatabaseConnection {
            session: Session {
                id,
                dsn: dsn.to_string(),
                handle: None,
            },
            _state: PhantomData,
        }
    }

    pub fn open(self) -> DatabaseConnection<Connected> {
        // ย้าย session ออกจาก self ทั้งก้อนได้เลย เพราะ DatabaseConnection<Disconnected> เอง "ไม่" implement Drop
        // (มีแค่ Session ข้างในที่ implement Drop) — ถ้า DatabaseConnection implement Drop ตรง ๆ เอง
        // บรรทัดนี้จะ compile ไม่ผ่าน (เหมือน error E0509 ด้านบน)
        let mut session = self.session;
        println!("[conn #{}] เปิดการเชื่อมต่อไปยัง {}", session.id.0, session.dsn);
        session.handle = Some(RawHandle);

        DatabaseConnection {
            session,
            _state: PhantomData,
        }
    }
}

impl DatabaseConnection<Connected> {
    pub fn query(&self, sql: &str) -> Vec<String> {
        println!("[conn #{}] รันคำสั่ง: {sql}", self.session.id.0);
        vec![format!("แถวผลลัพธ์จาก: {sql}")]
    }
}

fn main() {
    let mut id_counter = 0u64;
    let id = ConnectionId::generate(&mut id_counter);

    {
        let conn = DatabaseConnection::new(id, "postgres://localhost/app_db");
        let conn = conn.open();
        let rows = conn.query("SELECT * FROM users");
        println!("ได้ผลลัพธ์ {} แถว", rows.len());
    } // <- conn (DatabaseConnection<Connected>) หลุด scope ที่นี่ -> Session ข้างในถูก Drop -> ปิด connection ให้อัตโนมัติ

    // ตัวอย่างเปรียบเทียบ: สร้าง connection ที่ไม่เคยเปิดเลย (Disconnected) แล้วปล่อยให้หลุด scope ทันที
    // เพื่อพิสูจน์ว่า Drop ไม่ได้พิมพ์อะไรออกมา (ไม่มี handle ให้ปิดจริง เพราะยังไม่เคยเปิด)
    {
        let id2 = ConnectionId::generate(&mut id_counter);
        let _never_opened = DatabaseConnection::new(id2, "postgres://localhost/unused_db");
        println!("สร้าง connection #{} ไว้เฉย ๆ โดยไม่เปิดเลย", id2.0);
    }

    println!("[main] จบการทำงาน");
}
```

ผลลัพธ์:

```
[conn #1] เปิดการเชื่อมต่อไปยัง postgres://localhost/app_db
[conn #1] รันคำสั่ง: SELECT * FROM users
ได้ผลลัพธ์ 1 แถว
[conn #1] ปิดการเชื่อมต่อไปยัง postgres://localhost/app_db โดยอัตโนมัติ (RAII ผ่าน Drop)
สร้าง connection #2 ไว้เฉย ๆ โดยไม่เปิดเลย
[main] จบการทำงาน
```

**นี่คือ "payoff moment" ของบททั้งบท**: สังเกตว่าตัวอย่างสุดท้ายที่สร้าง connection #2 ไว้เฉย ๆ โดยไม่เคย
`.open()` เลย **ไม่มีข้อความ "ปิดการเชื่อมต่อ" ปรากฏออกมาเลย** — เพราะ `Session` ของมันมี `handle: None` ตลอด
เวลา (ทวนเช็คใน `Drop::drop`: `if self.handle.take().is_some()`) การไม่พิมพ์อะไรออกมาคือพฤติกรรมที่ถูกต้อง 100%
ไม่ใช่บั๊ก — และตรวจสอบให้แน่ใจว่าทั้งสามรูปแบบทำงานร่วมกันจริงในตัวอย่างนี้:

- **Newtype** (`ConnectionId(u64)`): ป้องกันไม่ให้ id ของ connection ถูกสลับกับตัวเลขอื่นในระบบ (ทวนหัวข้อ 53.3)
- **Typestate** (`DatabaseConnection<Disconnected>`/`<Connected>`): `query()` เรียกได้เฉพาะหลัง `.open()` แล้ว
  เท่านั้น — เรียกก่อนหน้านั้นจะได้ `E0599` แบบเดียวกับหัวข้อ 53.10 ทุกประการ
- **RAII** (`Session` + `Drop`): connection ที่เปิดจริงจะถูกปิดอัตโนมัติเสมอเมื่อหลุด scope ไม่ว่าจะลืมปิดเอง
  หรือไม่ก็ตาม — และถ้าเกิด panic ระหว่างใช้งาน connection (แบบเดียวกับหัวข้อ 53.14) `Session` ก็จะยังถูก Drop
  ระหว่าง unwinding เหมือนกัน

ทั้งสามรูปแบบไม่ได้ "แข่งกัน" หรือ "เลือกอย่างใดอย่างหนึ่ง" — มันเป็นเครื่องมือคนละชั้นที่ประกอบกันได้: newtype
ดูแลเรื่อง**ความถูกต้องของข้อมูล** typestate ดูแลเรื่อง**ลำดับการเรียกใช้งาน** และ RAII ดูแลเรื่อง**การคืน
resource** — สามมิติที่แตกต่างกันโดยสิ้นเชิงของการออกแบบ API ที่ปลอดภัย

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. Newtype: ลืมว่า Method ของ Type ภายในไม่ได้ "ติดมาด้วย" อัตโนมัติ

มือใหม่ที่เพิ่งห่อ newtype มักคาดหวังว่า method ของ type ภายในจะเรียกได้ตรง ๆ เหมือนเดิม แต่ถ้าไม่ได้ implement
`Deref` หรือเขียน wrapper method ไว้ จะไม่มี method นั้นอยู่จริงบน newtype เลย:

```rust
struct Score(u32);

fn main() {
    let s = Score(42);
    // คาดหวังว่าจะเรียก method ของ u32 ได้ตรง ๆ เหมือนที่ทำกับตัวเลขปกติ — แต่ Score ไม่ใช่ u32
    // และไม่ได้ implement Deref<Target = u32> ไว้ จึงไม่มี method นี้อยู่จริงบน Score เลย
    println!("{}", s.pow(2));
}
```

Error จริง:

```
error[E0599]: no method named `pow` found for struct `Score` in the current scope
 --> src/main.rs:7:22
  |
1 | struct Score(u32);
  | ------------ method `pow` not found for this struct
...
7 |     println!("{}", s.pow(2));
  |                      ^^^ method not found in `Score`
  |
help: one of the expressions' fields has a method of the same name
  |
7 |     println!("{}", s.0.pow(2));
  |                      ++
```

**วิธีแก้**: เข้าถึงผ่าน `.0` ตรง ๆ (`s.0.pow(2)` ตามที่ compiler แนะนำ), เขียน wrapper method เอง
(`impl Score { fn pow(&self, exp: u32) -> u32 { self.0.pow(exp) } }`), หรือ implement `Deref` ถ้าต้องการ method
จำนวนมากจาก type ภายใน (ทวนหัวข้อ 53.5 — ต้องพิจารณา invariant ก่อนเสมอว่าควร implement `DerefMut` ด้วยหรือไม่)

### 2. Newtype: พยายามสร้างค่าตรง ๆ โดยไม่ผ่าน Validating Constructor

ถ้า field ของ newtype เป็น private (เพื่อการันตี invariant ตามหัวข้อ 53.4) การพยายามสร้างค่าด้วย tuple struct
literal ตรง ๆ จากนอกโมดูลจะไม่ผ่าน:

```rust
mod email {
    #[derive(Debug, Clone, PartialEq, Eq)]
    pub struct EmailAddress(String);

    impl EmailAddress {
        pub fn as_str(&self) -> &str {
            &self.0
        }
    }
}

use email::EmailAddress;

fn main() {
    // พยายามสร้าง EmailAddress ตรง ๆ ผ่าน tuple struct literal โดยไม่ผ่านการตรวจสอบใด ๆ เลย
    let fake = EmailAddress("ไม่ใช่อีเมลแน่ ๆ".to_string());
    println!("{}", fake.as_str());
}
```

Error จริง:

```
error[E0423]: cannot initialize a tuple struct which contains private fields
  --> src/main.rs:16:16
   |
16 |     let fake = EmailAddress("ไม่ใช่อีเมลแน่ ๆ".to_string());
   |                ^^^^^^^^^^^^
   |
note: constructor is not visible here due to private fields
  --> src/main.rs:3:29
   |
 3 |     pub struct EmailAddress(String);
   |                             ^^^^^^ private field
help: consider making the field publicly accessible
   |
 3 |     pub struct EmailAddress(pub String);
   |                             +++
```

**สำคัญมาก**: ห้ามทำตามคำแนะนำสุดท้ายของ compiler (`pub String`) ถ้าเจตนาของ newtype คือการันตี invariant — การ
ทำให้ field เป็น `pub` จะ**ทำลาย invariant ทั้งหมด**ที่ตั้งใจสร้างไว้ทันที (ย้อนกลับไปสู่ปัญหาแบบเดียวกับหัวข้อ
53.5 ที่ `DerefMut` ทำลาย invariant ของ `SortedVec`) วิธีแก้ที่ถูกต้องคือสร้างค่าผ่าน `EmailAddress::try_from(...)`
(constructor ที่ตรวจสอบ) เท่านั้น

### 3. Typestate: เรียก Method ในสถานะที่ไม่ถูกต้อง — `E0599`

นี่คือกับดักที่สำคัญที่สุดของบทนี้ (และเป็นพฤติกรรมที่ **ถูกต้อง** ไม่ใช่บั๊ก — มันคือสิ่งที่ typestate ถูกออกแบบ
มาให้ทำ) — รายละเอียดเต็มอยู่ในหัวข้อ 53.10 ทวนสั้น ๆ: เรียก method ของสถานะหนึ่งบน object ที่ยังอยู่คนละสถานะ
จะได้ `E0599: no method named '...' found for struct '...' in the current scope` เสมอ — สังเกต hint สุดท้ายของ
compiler ที่บอกว่า "the method was found for" ตามด้วย type ของสถานะที่ถูกต้อง — hint นี้คือเบาะแสสำคัญที่สุดใน
การไล่บั๊กประเภทนี้: **แปลว่าคุณต้องเรียก method เปลี่ยนสถานะ (เช่น `.connect()`/`.authenticate()`) ก่อน ไม่ใช่
เพิ่ม method ที่ขาดไปเข้า `impl` block ปัจจุบัน**

### 4. Typestate: พยายามเก็บ Object คนละสถานะไว้ใน Collection เดียวกัน

ทวนจากหัวข้อ 53.12 — `Connection<Connected>` และ `Connection<Authenticated>` เป็นคนละ concrete type กันเลย
เก็บใน `Vec` เดียวกันตรง ๆ ไม่ได้ จะได้ `E0308: mismatched types` พร้อมข้อความ `expected 'Connection<Connected>',
found 'Connection<Authenticated>'` — ถ้าเจอ error แบบนี้บ่อย ๆ ในระบบของคุณ นั่นคือสัญญาณว่า **typestate อาจไม่
เหมาะกับสถานการณ์นี้แล้ว** ควรกลับไปพิจารณา enum state machine (Part 10) หรือ `enum` ครอบอีกชั้นตามที่แนะนำใน
หัวข้อ 53.12

### 5. RAII: เรียก `.drop()` ตรง ๆ ด้วยตัวเอง — `E0040`

มือใหม่ที่เข้าใจว่า `drop` เป็น "method ธรรมดา" มักลองเรียกมันตรง ๆ เพื่อ "บังคับ" ให้ resource ถูกคืนทันที:

```rust
struct Resource {
    name: String,
}

impl Drop for Resource {
    fn drop(&mut self) {
        println!("ปล่อย resource: {}", self.name);
    }
}

fn main() {
    let r = Resource {
        name: "การเชื่อมต่อฐานข้อมูล".to_string(),
    };
    // เรียก destructor ตรง ๆ ด้วยตัวเอง — Rust ห้ามเด็ดขาด เพราะจะทำให้ drop() ถูกเรียกซ้ำสองครั้ง
    // (อีกครั้งตอน r หลุด scope ตามปกติ) ซึ่งอาจนำไปสู่ double-free หรือ undefined behavior ได้ถ้าเกิดใน C/C++
    r.drop();
}
```

Error จริง:

```
error[E0040]: explicit use of destructor method
  --> src/main.rs:17:7
   |
17 |     r.drop();
   |       ^^^^ explicit destructor calls not allowed
   |
help: consider using `drop` function
   |
17 -     r.drop();
17 +     drop(r);
   |
```

**วิธีแก้**: ใช้ฟังก์ชัน `drop(r)` (ตัวพิมพ์เล็ก ไม่มี `::`) ที่มาจาก prelude แทน — มันคือฟังก์ชันธรรมดาที่รับ
`r` เข้าไปแบบ **take ownership** แล้วปล่อยให้มันหลุด scope ทันทีข้างในตัวมันเอง (compiler จะ generate การเรียก
`Drop::drop` ให้ตามปกติตอนจบฟังก์ชัน `drop()` นั้น) — วิธีนี้ปลอดภัยเพราะ ownership ของ `r` ถูกโอนไปให้ฟังก์ชัน
`drop()` เรียบร้อยแล้ว ทำให้ `r` ที่ scope เดิมไม่มีอยู่อีกต่อไป (compiler รู้และจะไม่เรียก `Drop::drop` ซ้ำอีก
ตอนจบ scope เดิมแน่นอน) ต่างจากการเรียก `r.drop()` ตรง ๆ ที่ `r` ยังคง "มีอยู่" ในสายตาของ borrow checker ต่อไป
หลังเรียก (เพราะ `drop(&mut self)` รับแค่ `&mut self` ไม่ได้กิน ownership) ซึ่งจะนำไปสู่การเรียก `Drop::drop`
ซ้ำสองครั้งถ้า Rust ยอมให้ทำแบบนี้ได้

### 6. RAII: Return Reference ที่ชี้เข้าไปใน Guard ชั่วคราว — `E0515`

กับดักคลาสสิกที่สุดของการทำงานกับ `MutexGuard<T>` (หรือ guard ประเภทอื่น) คือพยายามเขียนฟังก์ชัน helper ที่
"unlock แล้ว return ค่าข้างในออกไปเป็น reference":

```rust
use std::sync::Mutex;

// พยายาม return reference ที่ชี้เข้าไปในข้อมูลที่ MutexGuard คุ้มครองอยู่ ออกจากฟังก์ชัน
fn borrow_locked(m: &Mutex<i32>) -> &i32 {
    &*m.lock().unwrap()
}

fn main() {
    let m = Mutex::new(5);
    let r = borrow_locked(&m);
    println!("{r}");
}
```

Error จริง:

```
error[E0515]: cannot return value referencing temporary value
 --> src/main.rs:5:5
  |
5 |     &*m.lock().unwrap()
  |     ^^-----------------
  |     | |
  |     | temporary value created here
  |     returns a value referencing data owned by the current function
```

**อ่าน error นี้ให้ทะลุ**: `m.lock().unwrap()` สร้าง `MutexGuard<i32>` ที่เป็นค่า**ชั่วคราว** (temporary) ซึ่งมี
อายุอยู่แค่ภายในฟังก์ชัน `borrow_locked` เท่านั้น — พอฟังก์ชัน return, guard ตัวนี้จะถูก `Drop` (ปลดล็อก) ทันที
ถ้า Rust ยอมให้ return `&i32` ที่ชี้เข้าไปในข้อมูลที่ guard คุ้มครองอยู่ได้ reference นั้นจะกลายเป็น **dangling
reference** (ชี้ไปยังข้อมูลที่ไม่มี lock คุ้มครองแล้ว) ทันทีที่ออกจากฟังก์ชัน — borrow checker จับปัญหานี้ได้
ตั้งแต่ compile time พอดี ตรงกับธีมที่ย้ำมาตลอดทั้งหลักสูตร (หัวข้อ 53.11): ผลักบั๊กจาก runtime (dangling
pointer ที่เป็นหายนะที่สุดของ C/C++) มาเป็น compile error ที่แก้ได้ก่อนโปรแกรมรันจริงด้วยซ้ำ

**วิธีแก้**: return ค่าที่ **clone ออกมาแทน** (`*m.lock().unwrap()` ถ้า `T: Copy`, หรือ `.clone()` ถ้า `T:
Clone`) หรือปรับ design ให้ฟังก์ชันรับ closure ที่ทำงานกับข้อมูลข้างในทันทีตอนยังถือ lock อยู่ (แบบเดียวกับ
`with_lock(m, |value| { ... })`) แทนที่จะพยายามส่ง reference ออกมาให้ผู้เรียกใช้ทีหลัง

## แบบฝึกหัด (Exercises)

1. **(ง่าย) Newtype สำหรับหน่วยวัด**: สร้าง `struct Kilometers(f64)` และ `struct Miles(f64)` พร้อม method
   `to_miles(&self) -> Miles` และ `to_kilometers(&self) -> Kilometers` (ใช้ตัวคูณ 1 กม. = 0.621371 ไมล์) เขียน
   ฟังก์ชัน `fn fuel_cost(distance: Kilometers, price_per_km: f64) -> f64` แล้วลองเรียกมันด้วย `Miles` ตรง ๆ ดูว่า
   compiler ฟ้อง error อะไร (ควรเป็น `E0308` แบบเดียวกับ `UserId`/`ProductId` ในหัวข้อ 53.3) — Hint: ไม่ต้อง derive
   `Copy` ก็ได้ถ้าไม่อยากให้ค่าถูก copy โดยไม่ตั้งใจ แต่การ derive `Debug`/`Clone`/`Copy`/`PartialEq` ไว้ล่วงหน้า
   จะช่วยให้เขียน test ง่ายขึ้นมาก

2. **(กลาง) Validated Newtype**: สร้าง `struct ProductSku(String)` ที่การันตีว่า SKU ต้องมีความยาวตรง 8 ตัวอักษร
   และเป็นตัวอักษร A-Z หรือเลข 0-9 เท่านั้น (ห้ามมีตัวพิมพ์เล็กหรือสัญลักษณ์อื่น) ผ่าน `impl TryFrom<&str> for
   ProductSku` ที่ตรวจสอบก่อนสร้างค่าเสมอ (แบบเดียวกับ `EmailAddress` ในหัวข้อ 53.4) — Hint: ใช้ `.len()`,
   `.chars().all(|c| c.is_ascii_uppercase() || c.is_ascii_digit())` ในการตรวจสอบ และอย่าลืมทำ field เป็น private
   พร้อมเขียน accessor `.as_str()` เอง ลองเขียน error type ของตัวเองที่ implement `std::error::Error` ตามที่
   Part 12 สอนไว้ด้วย

3. **(ยาก) Typestate สำหรับ Document Workflow**: ออกแบบ `Document<State>` ที่มีสามสถานะ: `Draft` (แก้ไขได้ผ่าน
   `.edit(&mut self, content: &str)`), `Published` (อ่านได้ผ่าน `.content(&self) -> &str` แต่แก้ไม่ได้แล้ว), และ
   `Archived` (อ่านได้อย่างเดียวเหมือน `Published` แต่ไม่สามารถกลับไป `Published` ได้อีก) เขียน method
   `.publish(self) -> Document<Published>` (จาก `Draft`) และ `.archive(self) -> Document<Archived>` (จาก
   `Published`) จากนั้นลองเขียนโค้ดที่เรียก `.edit()` บน `Document<Published>` ดูว่าได้ error `E0599` ตรงตาม
   ที่คาดไว้หรือไม่ — Hint: ทบทวนกรอบการตัดสินใจในหัวข้อ 53.12 แล้วลองตอบคำถามนี้ด้วย: ถ้าระบบจริงต้องรองรับ
   "เวิร์กโฟลว์การอนุมัติเอกสาร" ที่มีสถานะเพิ่มขึ้นเรื่อย ๆ ตามการตั้งค่าของแต่ละองค์กร (เช่น บางองค์กรมี 3 ขั้น
   บางองค์กรมี 7 ขั้น กำหนดจาก config ไม่ใช่ compile time) ยังควรใช้ typestate อยู่ไหม เพราะอะไร

4. **(ยากมาก/ประยุกต์ใช้งานจริง) ผสานสามรูปแบบ**: ออกแบบ `FileHandle<State>` ที่มีสองสถานะ `Closed`/`Open` โดยมี
   newtype `struct FileDescriptorId(u32)` แทน id ของ handle ในระบบของคุณ (จำลอง ไม่ต้องเป็น fd จริงของ OS ก็ได้)
   `.open(path: &str) -> std::io::Result<FileHandle<Open>>` เปิดไฟล์จริง (`std::fs::File`) เก็บไว้ข้างใน และ
   method `.read_line(&mut self) -> std::io::Result<String>` ที่เรียกได้เฉพาะสถานะ `Open` เท่านั้น พร้อม RAII:
   ปิดไฟล์อัตโนมัติเมื่อ `FileHandle<Open>` หลุด scope — Hint: ใช้เทคนิคเดียวกับหัวข้อ 53.16 เป๊ะ ๆ (แยก struct
   ที่เป็นเจ้าของ `std::fs::File` จริง ๆ ออกมาต่างหาก อย่าให้ `FileHandle<State>` เอง implement `Drop` ตรง ๆ
   ไม่เช่นนั้นจะเจอ `E0509` เหมือนความพยายามแรกในหัวข้อ 53.16 ทุกประการ) ลองพิสูจน์ด้วย `assert!` ว่าไฟล์ถูกปิด
   จริงหลังหลุด scope (ใช้เทคนิคตรวจสอบว่าเปิดไฟล์ซ้ำได้อีกครั้งโดยไม่มี error "already in use" ถ้าระบบปฏิบัติการ
   ของคุณล็อกไฟล์ที่เปิดอยู่ — หรือใช้วิธีที่ตรงไปตรงมากว่าคือพิมพ์ log ยืนยันจาก `Drop::drop` แบบเดียวกับ
   `TempFile` ในหัวข้อ 53.14)

## สรุป

บทนี้ปิด mini-arc "Design Patterns ใน Rust" ที่ Part 52 เริ่มไว้ (Builder, Strategy, Observer — pattern ที่
ดัดแปลงจาก OOP ดั้งเดิม) ด้วยสามรูปแบบที่**เกิดขึ้นเองจากกลไกหลักของ Rust** ไม่ได้แปลมาจากที่ไหน:

**Newtype** ที่เคยรู้จักแค่ในฐานะทางแก้ orphan rule จาก Part 21 ตอนนี้ขยายเต็มรูปแบบเป็นสามเหตุผลหลัก: **type
safety** (ป้องกันการสลับค่าที่มี underlying type เดียวกัน พิสูจน์ด้วย `UserId`/`ProductId` และ error `E0308`
จริง), **encapsulation/invariant enforcement** (`EmailAddress` ที่การันตีความถูกต้องผ่าน private field +
`TryFrom` ที่ตรวจสอบ ผสาน Part 3/12 เข้าด้วยกัน) และ **การเพิ่ม method บน foreign type** (`SortedVec<T>` ผ่าน
`Deref` พร้อมข้อเตือนสำคัญเรื่องอันตรายของ `DerefMut` ที่ทำลาย invariant)

**Typestate** ที่ Part 52 ทีเซอร์ไว้ ตอนนี้เราสร้างเต็มรูปแบบผ่าน `Connection<State>` ที่ใช้ marker type ผสาน
`PhantomData` จาก Part 22 บังคับลำดับ `connect → authenticate → send_data` จนถึงจุดที่พิสูจน์ error `E0599`
จริงเมื่อเรียกผิดสถานะ — เชื่อมกลับไปที่ธีมของทั้งหลักสูตรตั้งแต่ Part 1: ผลักบั๊กจาก runtime ไปสู่ compile time
พร้อมกรอบการตัดสินใจที่ชัดเจนว่าเมื่อไหร่ควรใช้ typestate (โปรโตคอลที่ตายตัว สถานะน้อย) เมื่อไหร่ควรกลับไปใช้
enum state machine แบบ Part 10 แทน (สถานะมาก เปลี่ยนตามข้อมูล runtime)

**RAII** ที่คุณใช้มาโดยไม่รู้ตัวชื่อเต็มตั้งแต่ Part 6 (และ Part 6.12 ตั้งชื่อไว้ตรง ๆ แล้ว) ผ่าน `Box` (Part 27),
`Rc`/`RefCell` (Part 28), และ `MutexGuard` (Part 39) — ตอนนี้เราสร้าง RAII guard ของตัวเองสองแบบที่ใช้งานได้จริง
(`TempFile` ที่ลบไฟล์แม้ระหว่าง panic unwinding, `Timer`/`ScopeGuard` แบบ defer) พร้อมพิสูจน์ว่า `JoinHandle<T>`
**ไม่ใช่** RAII (drop แค่ detach ไม่ join) เพื่อให้เห็นขอบเขตของแนวคิดนี้ชัดเจน

ปิดท้ายด้วยตัวอย่างที่พิสูจน์ว่าสามรูปแบบทำงานร่วมกันได้จริง (`DatabaseConnection<State>` ที่เป็นทั้ง newtype +
typestate + RAII) พร้อมเจอและแก้ปัญหาจริงที่เกิดจากการผสานสองแนวคิด (`E0366` ที่ห้าม specialize `Drop`, และ
`E0509` ที่ห้ามย้าย field ออกจาก type ที่ implement `Drop` เอง) — บทเรียนสำคัญคือการ**แยก resource ที่ต้อง Drop
ออกจาก wrapper ที่ทำ state transition** ซึ่งเป็นเทคนิคที่ใช้ได้จริงทุกครั้งที่ต้องผสาน typestate กับ RAII

หลักสูตรได้พาคุณผ่านทั้ง "pattern ที่ดัดแปลงจาก OOP" (Part 52) และ "pattern ที่เป็น Rust แท้ ๆ" (บทนี้) มาครบ
แล้ว — ทั้งหมดนี้คือเครื่องมือออกแบบ API ระดับ production ที่คุณจะเห็นซ้ำ ๆ ในโค้ด Rust จริงจากนี้ไป ใน **Part
54: Performance Optimization และ Benchmarking** เราจะเปลี่ยนโฟกัสจาก "การออกแบบให้ถูกต้องและปลอดภัย" ไปสู่ "การ
วัดผลและปรับให้เร็วขึ้นอย่างมีหลักการ" — เรียนรู้การใช้ `criterion` เขียน benchmark ที่เชื่อถือได้ทางสถิติ อ่าน
ผลลัพธ์เพื่อตัดสินใจว่าจุดไหนของโค้ดคือ bottleneck จริง ก่อนจะลงมือ optimize (ทวนหลักการ "อย่า optimize ก่อนรู้
ว่าช้าจริง" ที่ Part 21 หัวข้อ 21.3 เกริ่นไว้ตอนพูดถึงต้นทุนของ dynamic dispatch)

---

**Part ก่อนหน้า:** [Design Patterns ใน Rust (Builder, Strategy, Observer)](part-052-design-patterns-1.md) | **Part ถัดไป:** [Performance Optimization และ Benchmarking](part-054-performance-benchmarking.md)
