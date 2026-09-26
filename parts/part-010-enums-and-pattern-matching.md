# Part 10: Enums และ Pattern Matching

> โมดูล: พื้นฐานภาษา Rust (Core Language Fundamentals) | ระดับ: พื้นฐาน | เวลาโดยประมาณ: 180 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้ว่าทำไมการจำลอง "หนึ่งในหลายตัวเลือกที่แน่นอน" ด้วย struct + boolean flags หรือ string tag ถึงเสี่ยงต่อ
  "สถานะที่ไม่ควรมีอยู่จริง" (invalid state) และ enum แก้ปัญหานี้อย่างไรโดยให้ compiler ช่วยการันตี
- นิยาม enum ได้ทั้งแบบ unit variant ธรรมดา และแบบที่แต่ละ variant มีข้อมูลติดตัวมาต่างกัน (tuple-like, struct-like)
  ผสมกันในตัวเดียว ซึ่งเป็นจุดเด่นที่ enum ของ Rust ทำได้เหนือกว่า enum ของ C/Java/C# มาก
- เขียน `impl` block บน enum และสร้าง method ที่ใช้ `match self` เพื่อดำเนินการต่างกันตาม variant ได้
- ใช้ `match` แบบเต็มรูปแบบ ทั้ง literal pattern, range pattern (`1..=5`), or-pattern (`1 | 2 | 3`), destructuring
  pattern ของ enum/tuple, และ match guard (`if condition`) พร้อมเข้าใจว่า **exhaustiveness checking** ของ `match`
  คือ feature ที่ป้องกันบั๊ก ไม่ใช่ความเข้มงวดที่น่ารำคาญ
- อธิบาย `Option<T>` ในฐานะ enum ที่ built-in ของ Rust ที่แก้ปัญหา null ได้อย่างปลอดภัยกว่าภาษาอื่น และรู้ว่า compiler
  จะบังคับให้จัดการกรณี `None` เสมอ
- เลือกใช้ `if let`, `while let`, และ `let else` แทน `match` แบบยาว ๆ ได้ถูกสถานการณ์ เมื่อสนใจ pattern เดียว
- ออกแบบระบบที่ใช้ enum + `match` ทำให้ **สถานะที่ผิดกฎ (illegal state) กลายเป็นสิ่งที่เขียนไม่ได้เลยในเชิง type**
  (illegal states unrepresentable) ผ่านตัวอย่าง state machine ของออเดอร์สั่งซื้อสินค้า

## ความรู้ที่ต้องมีมาก่อน

- **Part 4**: เคยเห็น `match` แบบเบื้องต้นมาแล้ว (match บนตัวเลข/bool ที่ต้องมี `_` ปิดท้ายเสมอ) แต่ Part 4 บอกไว้อย่าง
  ชัดเจนว่าเป็นแค่ teaser และเก็บรายละเอียดเต็มไว้ให้บทนี้ — ถ้าคุณจำ syntax `match value { pattern => expr, ... }`
  หรือเรื่อง `match` เป็น expression ได้ ก็พร้อมสำหรับบทนี้แล้ว
- **Part 6 (Ownership)**: ต้องเข้าใจเรื่อง move, ownership ของค่าที่ไม่ implement `Copy` (เช่น `String`) เพราะการ match
  บนข้อมูลที่มี field เป็น `String` เกี่ยวข้องกับ ownership โดยตรง (ใครเป็นเจ้าของค่าหลัง match)
- **Part 7 (Borrowing)**: ต้องเข้าใจ `&T`/`&mut T` เพราะการ match บน reference (`match &value` หรือ `for x in &vec`)
  จะทำงานร่วมกับ pattern matching ผ่านกลไกที่เรียกว่า **match ergonomics** ซึ่งบทนี้จะอธิบายและมีกับดักที่เกี่ยวข้องด้วย
- **Part 8 (Slices)**: ใช้ประกอบความเข้าใจเรื่อง `&str` ที่จะเห็นในตัวอย่าง field ของ enum variant
- **Part 9 (Structs)**: ต้องรู้จัก named struct, tuple struct, unit struct, `impl` block, method, associated function,
  และ `#[derive(Debug)]` มาก่อน เพราะบทนี้จะเปรียบเทียบ enum กับ struct ตลอดเวลา และใช้ `impl` block บน enum ในรูปแบบ
  เดียวกับที่คุณเรียนมาแล้วกับ struct

ถ้าคุณยังไม่คุ้นกับหัวข้อเหล่านี้ แนะนำให้กลับไปทวนก่อน เพราะบทนี้จะอ้างอิงแนวคิดเรื่อง ownership/borrowing และ struct
อยู่ตลอดทั้งบท

## เนื้อหา

### 10.1 ปัญหาที่ enum แก้: "หนึ่งในหลายตัวเลือกที่แน่นอน"

ลองนึกภาพว่าคุณกำลังเขียนระบบร้านค้าออนไลน์ และต้องเก็บข้อมูลว่าลูกค้าจะจ่ายเงินด้วยวิธีไหน — มีให้เลือก 3 ทาง คือ
**เงินสด (Cash)**, **บัตรเครดิต (CreditCard)**, หรือ **โอนเงินผ่านธนาคาร (BankTransfer)** จากความรู้ที่มีถึง Part 9
เราอาจลองออกแบบด้วย struct ธรรมดาก่อน แนวทางแรกที่มือใหม่มักคิดถึงคือใช้ `bool` เป็น flag แยกสามตัว:

```rust
struct PaymentMethod {
    is_cash: bool,
    is_credit_card: bool,
    is_bank_transfer: bool,
}

fn describe(payment: &PaymentMethod) -> &'static str {
    if payment.is_cash {
        "จ่ายด้วยเงินสด"
    } else if payment.is_credit_card {
        "จ่ายด้วยบัตรเครดิต"
    } else if payment.is_bank_transfer {
        "จ่ายด้วยการโอนเงิน"
    } else {
        "ไม่ระบุวิธีจ่าย"
    }
}

fn main() {
    let broken = PaymentMethod {
        is_cash: true,
        is_credit_card: true, // ทั้งสองเป็น true พร้อมกัน! สถานะที่ไม่ควรเกิดขึ้นได้เลย
        is_bank_transfer: false,
    };
    println!("{}", describe(&broken));
}
```

โค้ดนี้ **compile ผ่านสบาย ๆ** และรันได้ผลลัพธ์:

```
จ่ายด้วยเงินสด
```

แต่สังเกตปัญหาที่ซ่อนอยู่: เราสร้างค่า `broken` ที่ `is_cash` และ `is_credit_card` เป็น `true` พร้อมกันได้อย่างง่ายดาย
ทั้งที่ในความเป็นจริง **ไม่มีทางที่ลูกค้าจะจ่ายด้วยเงินสด "และ" บัตรเครดิตในเวลาเดียวกันในฟิลด์เดียวนี้ได้** — compiler
ไม่มีทางรู้เลยว่ากฎทางธุรกิจคือ "flag พวกนี้ต้องมีแค่ตัวเดียวที่เป็น `true`" เพราะในเชิง type แล้ว `PaymentMethod`
คือ struct ที่มี `bool` 3 ตัวซึ่งเป็นอิสระจากกันโดยสมบูรณ์ ไม่มีอะไรผูกมัดให้มันสัมพันธ์กัน การตรวจสอบว่า "ไม่มีสอง flag
เป็น true พร้อมกัน" ต้องเขียนเป็นโค้ด validation แยกและเรียกใช้ทุกครั้งที่สร้างหรือแก้ไขค่า ซึ่งลืมเรียกได้ง่ายมาก
และในตัวอย่างข้างบน ฟังก์ชัน `describe` เองก็ไม่รู้ตัวว่ากำลังทำงานกับสถานะที่ไม่สมเหตุสมผล มันเพียง "เดา" ว่า
`is_cash` มาก่อนก็ให้ตอบแบบนั้นไปเลย ทำให้บั๊กเงียบ ๆ แบบนี้หลุดไปถึง production ได้โดยไม่มีสัญญาณเตือนใด ๆ

จำนวน flag ที่เป็นไปได้จริงคือ 2^3 = 8 สถานะ แต่มีแค่ 3 สถานะ (หรือ 4 ถ้ารวม "ไม่ระบุวิธีจ่าย") ที่ **มีความหมาย**
ในทางธุรกิจ ส่วนอีก 4-5 สถานะที่เหลือ (เช่น ทั้งสามเป็น `true`, สองในสามเป็น `true`) คือ **สถานะที่ไม่ควรมีอยู่จริง**
แต่ type system ของ struct + bool ไม่มีทางป้องกันไม่ให้สร้างสถานะเหล่านั้นได้เลย

#### ลองแก้ด้วย string tag แทน — ก็ยังมีปัญหา

มือใหม่บางคนอาจคิดว่าปัญหาอยู่ที่ boolean หลายตัว จึงลองรวมเป็น field เดียวเป็น string tag แทน:

```rust
struct PaymentMethodTagged {
    kind: String,        // ตั้งใจให้เป็นได้แค่ "cash" / "credit_card" / "bank_transfer"
    card_number: String, // ใช้เฉพาะกรณี credit_card เท่านั้น ถ้าไม่ใช่ก็เก็บ "" ไปแบบไม่มีความหมาย
}

fn describe_tagged(p: &PaymentMethodTagged) -> String {
    match p.kind.as_str() {
        "cash" => "จ่ายด้วยเงินสด".to_string(),
        "credit_card" => format!("จ่ายด้วยบัตรเครดิตเลขที่ {}", p.card_number),
        "bank_transfer" => "จ่ายด้วยการโอนเงิน".to_string(),
        other => format!("ไม่รู้จักวิธีจ่ายแบบ '{other}'"), // typo ก็หลุดมาถึง runtime ได้
    }
}

fn main() {
    let typo_payment = PaymentMethodTagged {
        kind: "cassh".to_string(), // พิมพ์ผิด! compiler ไม่ช่วยจับได้เลย เพราะเป็นแค่ String ธรรมดา
        card_number: String::new(),
    };
    println!("{}", describe_tagged(&typo_payment));
}
```

ผลลัพธ์:

```
ไม่รู้จักวิธีจ่ายแบบ 'cassh'
```

ดีขึ้นเล็กน้อยตรงที่ไม่มีการ "true ซ้อนกัน" อีกแล้ว แต่ปัญหาใหม่ที่ร้ายแรงไม่แพ้กันคือ **`kind` เป็น `String` ธรรมดา
ที่รับค่าอะไรก็ได้** พิมพ์ผิดแค่ตัวอักษรเดียว (`"cassh"` แทน `"cash"`) compiler ก็ไม่มีทางรู้เลยว่านี่คือค่าที่ผิด
เพราะสำหรับ compiler แล้ว `String` คือ `String` ไม่ว่าเนื้อหาข้างในจะเป็นอะไร ข้อผิดพลาดแบบนี้จะไปโป่งแตกที่ branch
`other` ตอน **runtime** เท่านั้น (ถ้าคุณโชคดีที่มี test ครอบคลุมกรณีนี้) และยังมีปัญหาเพิ่มอีกอย่างคือ `card_number`
ที่ควรมีความหมายเฉพาะตอน `kind == "credit_card"` แต่ในเชิง type แล้วมันมีอยู่เสมอไม่ว่า `kind` จะเป็นอะไร ทำให้เกิด
"ฟิลด์ที่ไม่มีความหมายในบางสถานะ" (partial/optional field ที่ไม่ได้ประกาศว่าเป็น optional) แบบเดียวกับปัญหาที่ 1

สรุปได้ว่าทั้งสองแนวทาง (boolean flags และ string tag) มีจุดร่วมกันคือ **type ของข้อมูลไม่ได้บังคับกฎทางธุรกิจ
ที่แท้จริง** ("ต้องเป็นแค่หนึ่งในตัวเลือกที่กำหนดไว้ล่วงหน้า ไม่ใช่ค่าอะไรก็ได้") ไว้เลย มันปล่อยให้เป็นภาระของโปรแกรมเมอร์
ที่ต้องจำเอาเองว่าต้อง validate ตรงไหน เรียก validation ทุกจุดที่สร้าง/แก้ไขค่าหรือไม่ และคนอ่านโค้ดคนอื่นก็ต้องอ่าน
comment หรือ documentation เพื่อรู้ว่า `kind` รับค่าได้แค่ 3 แบบเท่านั้น — ไม่มีอะไรในตัว type เองที่บอกสิ่งนี้ตรง ๆ

#### enum: ให้ compiler บังคับว่า "ต้องเป็นหนึ่งในตัวเลือกนี้เท่านั้น"

นี่คือจุดที่ **enum** (ย่อจาก enumeration หรือ "การแจงนับ") เข้ามาแก้ปัญหาตรงจุด แนวคิดของ enum คือ: กำหนด **ชุดของ
ตัวเลือก (variant) ที่เป็นไปได้ทั้งหมดไว้ล่วงหน้าอย่างชัดเจน** แล้วให้ค่าของ type นั้น **เป็นได้แค่หนึ่งในตัวเลือก
เหล่านั้นเท่านั้น ไม่มีทางเป็นอย่างอื่น และไม่มีทางเป็นสองตัวเลือกพร้อมกัน** ซึ่งเป็นกฎที่ **compiler การันตีให้ 100%**
ไม่ใช่แค่ convention ที่ต้องจำเอง — เราจะเห็นรูปร่างของ enum ตัวจริงในหัวข้อถัดไป แต่ให้จำหลักการสำคัญที่สุดไว้ก่อน:

> **enum ทำให้ "สถานะที่ไม่มีความหมาย" (เช่น true 2 ตัวพร้อมกัน, string พิมพ์ผิด) กลายเป็นสิ่งที่เขียนไม่ได้เลย
> ในเชิง type ตั้งแต่ต้น** ไม่ใช่แค่ตรวจจับได้เร็วขึ้น — มันคือ "เขียนไม่ได้" ไปเลย

นี่คือแนวคิดที่ในวงการ type-driven design เรียกว่า **"making illegal states unrepresentable"** ซึ่งเราจะเห็นภาพเต็ม ๆ
ในตัวอย่าง state machine ท้ายบทนี้ และเป็นหนึ่งในเหตุผลสำคัญที่สุดที่ทำให้ enum ของ Rust (รวมกับ pattern matching)
ถูกยกให้เป็นหนึ่งใน feature ที่ทรงพลังที่สุดของภาษา

### 10.2 นิยาม enum พื้นฐานและการสร้าง Instance

มาดูวิธีแก้ปัญหา `PaymentMethod` ด้วย enum ตัวจริง:

```rust
enum PaymentMethod {
    Cash,
    CreditCard,
    BankTransfer,
}

fn describe(method: &PaymentMethod) -> &'static str {
    match method {
        PaymentMethod::Cash => "จ่ายด้วยเงินสด",
        PaymentMethod::CreditCard => "จ่ายด้วยบัตรเครดิต",
        PaymentMethod::BankTransfer => "จ่ายด้วยการโอนเงิน",
    }
}

fn main() {
    let method = PaymentMethod::Cash;
    println!("{}", describe(&method));
}
```

ผลลัพธ์:

```
จ่ายด้วยเงินสด
```

**อธิบายโค้ดทีละส่วน:**

- `enum PaymentMethod { Cash, CreditCard, BankTransfer }` — ประกาศ enum ชื่อ `PaymentMethod` ที่มี 3 **variant**
  (ตัวเลือก) คือ `Cash`, `CreditCard`, `BankTransfer` แต่ละ variant ในตัวอย่างนี้ไม่มีข้อมูลติดมาด้วยเลย เรียกว่า
  **unit variant** (คล้ายกับ unit struct ที่คุณเรียนใน Part 9 — ไม่มี field ใด ๆ)
- `PaymentMethod::Cash` — การสร้างค่าของ enum ต้องเขียนชื่อ type ตามด้วย `::` แล้วตามด้วยชื่อ variant เสมอ (ต่างจาก
  struct ที่สร้างด้วย `StructName { field: value }`) นี่เพราะ variant ไม่ได้ "เป็น" type แยกของตัวเอง มันเป็นเพียง
  หนึ่งในความเป็นไปได้ของ type `PaymentMethod` การเขียน `PaymentMethod::Cash` จึงระบุทั้ง type และตัวเลือกที่เลือก
  ในคำสั่งเดียว
- `let method = PaymentMethod::Cash;` — สังเกตว่า type ของ `method` คือ `PaymentMethod` ตัวเดียว **ไม่ใช่**
  `PaymentMethod::Cash` เพราะ `Cash` ไม่ใช่ type มันเป็นเพียงหนึ่งใน 3 ค่าที่เป็นไปได้ของ type `PaymentMethod` —
  ไม่ว่าคุณจะสร้าง `PaymentMethod::Cash`, `PaymentMethod::CreditCard`, หรือ `PaymentMethod::BankTransfer` ตัวแปรที่ได้
  ก็จะมี type เดียวกันคือ `PaymentMethod` เสมอ ทำให้คุณสามารถส่งค่าเหล่านี้ผ่านฟังก์ชันเดียวกัน เก็บใน `Vec<PaymentMethod>`
  เดียวกัน ฯลฯ ได้อย่างอิสระ
- `match method { PaymentMethod::Cash => ..., ... }` — วิธีเดียวที่จะ "แยกแยะ" ว่าค่าของ enum เป็น variant ไหนคือ
  `match` (เรา teaser มาแล้วใน Part 4 บทนี้จะลงรายละเอียดเต็มในหัวข้อ 10.5) สังเกตว่าเราครอบคลุมครบทั้ง 3 variant
  — ถ้าลืมข้อใดข้อหนึ่ง compiler จะฟ้อง error ทันที ซึ่งเป็นหัวใจสำคัญที่สุดของบทนี้และจะอธิบายลึกในหัวข้อ 10.5

ลองเทียบกับภาษาอื่นที่คุณอาจคุ้นเคย:

| ภาษา | enum ทำอะไรได้ | ข้อจำกัด |
|---|---|---|
| **C / C++ (แบบดั้งเดิม)** | เป็นแค่ชื่อแทนค่า integer ตัวหนึ่ง (`enum Color { RED, GREEN, BLUE };` จริง ๆ คือ `RED=0, GREEN=1, BLUE=2`) | แปลงเป็น/จาก integer ได้อย่างอิสระ (`(Color)99` ก็ compile ผ่าน แม้ 99 ไม่ตรงกับ variant ใดเลย) ไม่มีข้อมูลติดตัว |
| **Java / C#** | เป็น class พิเศษที่มีค่าคงที่จำกัด สามารถมี field/method ได้ | ทุก variant ต้องมีโครงสร้าง field เดียวกัน (ต้องใช้ inheritance/abstract method ซับซ้อนกว่าถ้าอยากให้ variant ต่างกัน) |
| **TypeScript** | ใช้ "discriminated union" (`type X = {kind:"a", ...} \| {kind:"b", ...}`) จำลองพฤติกรรมคล้าย Rust enum ได้ | ไม่ใช่ feature ภาษาโดยตรง เป็น pattern ที่ประกอบขึ้นจาก union type + literal type, ยังพลาดพิมพ์ผิด string ได้ถ้าไม่ตั้ง type ให้รัดกุม |
| **Rust** | แต่ละ variant มีข้อมูลติดตัวต่างกันได้ตามต้องการ (จะเห็นในหัวข้อถัดไป) และ compiler บังคับ exhaustiveness ตอน match | ต้องเรียนรู้ syntax ของ pattern matching เพิ่มเติม (คือเนื้อหาบทนี้) |

จะเห็นว่า enum ของ C/C++ แบบดั้งเดิมอ่อนแอกว่า Rust มาก เพราะโดยพื้นฐานมันคือ integer ที่แปลงกลับไปกลับมาได้อย่างไม่มี
การตรวจสอบ ส่วน Java/C# แม้ enum จะปลอดภัยกว่า C แต่ก็ยังไม่มีกลไกที่ทำให้แต่ละ variant มีข้อมูลติดตัว "ต่างรูปร่างกัน"
ได้อย่างเป็นธรรมชาติเท่า Rust — ซึ่งเป็นเนื้อหาของหัวข้อถัดไปที่ถือเป็นจุดขายที่แท้จริงของ enum ใน Rust

### 10.3 Enum ที่มีข้อมูลติดอยู่กับ Variant: จุดเด่นที่สุดของ Enum ใน Rust

ถ้า enum ของ Rust ทำได้แค่ที่เห็นในหัวข้อ 10.2 มันก็จะไม่ต่างจาก enum ของ Java มากนัก แต่จุดที่ทำให้ enum ของ Rust
ทรงพลังกว่าภาษาส่วนใหญ่มากคือ **แต่ละ variant สามารถมีข้อมูลติดตัวอยู่ในรูปแบบที่ต่างกันได้อย่างสมบูรณ์** กลับไปที่
ตัวอย่าง `PaymentMethod` — ในความเป็นจริง แต่ละวิธีจ่ายเงินมีข้อมูลที่ต้องเก็บต่างกัน: เงินสดไม่ต้องเก็บอะไรเพิ่ม,
บัตรเครดิตต้องเก็บเลขบัตรกับ CVV, โอนเงินต้องเก็บเลขบัญชีปลายทาง เราสามารถเขียน enum เดียวที่แต่ละ variant
"พกข้อมูลของตัวเอง" ได้ตรงตามความต้องการจริง:

```rust
enum PaymentMethod {
    Cash,
    CreditCard { number: String, cvv: String },
    BankTransfer(String), // เก็บเลขบัญชีปลายทาง
}

fn describe(method: &PaymentMethod) -> String {
    match method {
        PaymentMethod::Cash => "จ่ายด้วยเงินสด".to_string(),
        PaymentMethod::CreditCard { number, cvv } => {
            format!("จ่ายด้วยบัตรเครดิตเลขที่ {number} (cvv ยาว {} หลัก)", cvv.len())
        }
        PaymentMethod::BankTransfer(account) => format!("โอนเงินไปยังบัญชี {account}"),
    }
}

fn main() {
    let methods = vec![
        PaymentMethod::Cash,
        PaymentMethod::CreditCard {
            number: "4111111111111111".to_string(),
            cvv: "123".to_string(),
        },
        PaymentMethod::BankTransfer("123-4-56789-0".to_string()),
    ];
    for m in &methods {
        println!("{}", describe(m));
    }
}
```

ผลลัพธ์:

```
จ่ายด้วยเงินสด
จ่ายด้วยบัตรเครดิตเลขที่ 4111111111111111 (cvv ยาว 3 หลัก)
โอนเงินไปยังบัญชี 123-4-56789-0
```

**อธิบายรูปแบบ variant ทั้ง 3 แบบที่ผสมกันอยู่ในตัวอย่างนี้:**

1. **Unit variant** — `Cash` ไม่มี field เลย เหมือน unit struct ที่คุณรู้จักจาก Part 9 ใช้เมื่อ variant นั้นไม่จำเป็น
   ต้องพกข้อมูลอะไรเพิ่มเติม ตัวมันเองก็สื่อความหมายครบแล้ว
2. **Struct-like variant** — `CreditCard { number: String, cvv: String }` มีโครงสร้างเหมือน named struct ทุกประการ
   คือมี field ที่ตั้งชื่อได้ (`number`, `cvv`) พร้อม type ของแต่ละ field เวลาสร้างค่าก็ใช้ syntax คล้าย struct
   (`PaymentMethod::CreditCard { number: ..., cvv: ... }`) และเวลา match ก็ destructure ด้วยชื่อ field ได้ตรง ๆ
3. **Tuple-like variant** — `BankTransfer(String)` มีโครงสร้างเหมือน tuple struct จาก Part 9 คือมีข้อมูลติดมาแต่
   ไม่ได้ตั้งชื่อ field เข้าถึงตามลำดับตำแหน่ง เหมาะกับกรณีที่มีข้อมูลแค่ 1-2 ชิ้นและความหมายชัดเจนอยู่แล้วจากชื่อ
   variant เอง (ในที่นี้ `BankTransfer(String)` สื่อชัดว่า `String` นั้นคือเลขบัญชี ไม่ต้องตั้งชื่อ field ซ้ำอีก)

จุดสำคัญที่สุดคือ **ทั้ง 3 รูปแบบนี้อยู่ใน enum เดียวกันได้อย่างอิสระ** — Rust ไม่ได้บังคับว่าทุก variant ต้องมีรูปร่าง
ข้อมูลเหมือนกัน ต่างจาก Java enum ที่ถ้าอยากให้แต่ละค่ามี field ต่างกัน คุณต้องใช้เทคนิคที่ซับซ้อนกว่ามาก (เช่น
constructor ต่างกันต่อค่า หรือใช้ abstract method ที่ต่างกันในแต่ละ enum constant) และยังไม่ยืดหยุ่นเท่าที่นี่ทำได้

#### เปรียบเทียบกับ `union` ใน C — enum ของ Rust คือ "tagged union" ที่ปลอดภัย

ถ้าคุณมีพื้นฐาน C/C++ มาก่อน สิ่งที่ enum แบบมีข้อมูลของ Rust ทำได้จริง ๆ ก็คือสิ่งที่เรียกว่า **tagged union**
(หรือ discriminated union) — แนวคิดคือเก็บ "tag" (ตัวบอกว่าตอนนี้เป็น variant ไหน) คู่กับ memory ที่ใหญ่พอสำหรับ
variant ที่ใหญ่ที่สุด ปัญหาของ `union` ดิบ ๆ ใน C คือมันปล่อยให้คุณอ่าน field ผิด variant ได้อย่างอิสระ (undefined
behavior) เพราะ `union` ไม่มี tag ติดมาด้วย คุณต้องจัดการ tag เองแยกต่างหากและเชื่อใจตัวเองว่าจะเช็คถูกทุกครั้ง
แต่ enum ของ Rust ผนวก tag เข้ากับข้อมูลให้อัตโนมัติ และ **บังคับให้คุณเช็ค tag ผ่าน `match` ก่อนเข้าถึงข้อมูลเสมอ**
compiler จะไม่ยอมให้คุณอ่าน field `number` ของ `CreditCard` ได้เลยถ้าไม่ผ่านการ `match`/destructure ที่พิสูจน์แล้วว่า
ค่านั้นเป็น variant `CreditCard` จริง ๆ — นี่คือความปลอดภัยที่ได้แบบ "ไม่มีต้นทุนเพิ่ม" (zero-cost) เพราะ tag ที่ว่า
มักใช้พื้นที่แค่ไม่กี่ไบต์ หรือบางกรณีไม่ใช้พื้นที่เพิ่มเลยด้วยซ้ำ ลองดูตัวอย่าง:

```rust
enum PaymentMethod {
    Cash,
    CreditCard { number: String, cvv: String },
    BankTransfer(String),
}

fn main() {
    println!("ขนาดของ PaymentMethod: {} bytes", std::mem::size_of::<PaymentMethod>());
    println!("ขนาดของ String เดี่ยว ๆ: {} bytes", std::mem::size_of::<String>());
}
```

ผลลัพธ์ (บนเครื่อง 64-bit ทั่วไป):

```
ขนาดของ PaymentMethod: 48 bytes
ขนาดของ String เดี่ยว ๆ: 24 bytes
```

`String` หนึ่งตัวกิน 24 ไบต์ (pointer + length + capacity อย่างละ 8 ไบต์ ตามที่จะเรียนละเอียดใน Part 14) variant
ที่ใหญ่ที่สุดคือ `CreditCard` ที่มี `String` สองตัว = 48 ไบต์ และสังเกตว่า **ขนาดรวมของ `PaymentMethod` ก็คือ 48
ไบต์เท่ากันเป๊ะ ๆ ไม่ได้บวกเพิ่มสำหรับ "tag" เลย** — นี่เพราะ compiler ของ Rust ฉลาดพอที่จะ "ซ่อน" tag ไว้ในช่องว่าง
(padding/niche) ที่ไม่ได้ใช้งานอยู่แล้วจากการ alignment ของ `String` (เทคนิคนี้เรียกว่า **niche filling optimization**)
คุณไม่ต้องเข้าใจรายละเอียดการ optimize นี้ในระดับลึกตอนนี้ แต่ควรจำภาพใหญ่ไว้ว่า: **enum ที่มีข้อมูลใน Rust ไม่ใช่
"ของแพง" ในเชิง performance — มันแปลงเป็น machine code ที่มีประสิทธิภาพเทียบเท่าการเขียน `union` + tag ด้วยมือใน C
โดยที่คุณได้ความปลอดภัยเพิ่มมาแบบไม่มีต้นทุน** สอดคล้องกับแนวคิด zero-cost abstraction ที่เราพูดถึงตั้งแต่ Part 1

### 10.4 `impl` Blocks บน Enum: Method ที่ `match self`

จาก Part 9 คุณรู้อยู่แล้วว่า struct ผูก method เข้ากับตัวเองผ่าน `impl` block — enum ก็ทำแบบเดียวกันได้ทุกประการ
และรูปแบบที่พบบ่อยที่สุดในโค้ด Rust จริงคือ method ที่ `match self` แล้วทำสิ่งต่างกันไปตาม variant:

```rust
enum PaymentMethod {
    Cash,
    CreditCard { number: String, cvv: String },
    BankTransfer(String),
}

impl PaymentMethod {
    fn transaction_fee_percent(&self) -> f64 {
        match self {
            PaymentMethod::Cash => 0.0,
            PaymentMethod::CreditCard { .. } => 2.5,
            PaymentMethod::BankTransfer(_) => 1.0,
        }
    }

    fn describe(&self) -> String {
        match self {
            PaymentMethod::Cash => "เงินสด".to_string(),
            PaymentMethod::CreditCard { number, .. } => {
                let last4 = &number[number.len() - 4..];
                format!("บัตรเครดิตลงท้าย {last4}")
            }
            PaymentMethod::BankTransfer(account) => format!("โอนเงินเข้าบัญชี {account}"),
        }
    }
}

fn main() {
    let m = PaymentMethod::CreditCard {
        number: "4111111111111111".to_string(),
        cvv: "123".to_string(),
    };
    println!("{} (fee {}%)", m.describe(), m.transaction_fee_percent());
}
```

ผลลัพธ์:

```
บัตรเครดิตลงท้าย 1111 (fee 2.5%)
```

**อธิบายโค้ดทีละส่วน:**

- `impl PaymentMethod { ... }` — เขียน `impl` block บน enum ด้วย syntax เดียวกับที่คุณเรียนกับ struct ใน Part 9
  ทุกประการ ไม่มีอะไรพิเศษเพิ่มเติม
- `fn transaction_fee_percent(&self) -> f64` — method ที่รับ `&self` (borrow ตัวเอง ไม่ยึด ownership — ตามหลัก
  Part 7) แล้ว `match self` เพื่อตอบค่าธรรมเนียมต่างกันตามวิธีจ่าย: เงินสดไม่มีค่าธรรมเนียม บัตรเครดิตคิด 2.5%
  โอนเงินคิด 1%
- `PaymentMethod::CreditCard { .. }` — สังเกตการใช้ `{ .. }` (สองจุด) ภายใน pattern ของ struct-like variant
  เพื่อบอกว่า "ฉันรู้ว่า variant นี้มี field `number` กับ `cvv` แต่ในกรณีนี้ไม่สนใจค่าของมันเลย ข้ามไปหมด" ถ้าไม่ใส่
  `..` คุณจะต้องเขียนชื่อ field ให้ครบทุกตัว (หรือใช้ `_` แทนแต่ละตัว) ซึ่งเป็นการอธิบายซ้ำที่ไม่จำเป็นถ้าไม่ได้ใช้ค่านั้น
  เลย
- `PaymentMethod::BankTransfer(_)` — สำหรับ tuple-like variant การไม่สนใจข้อมูลใช้ `_` ตรงตำแหน่งนั้นได้เลย
  (ต่างจาก struct-like variant ที่ใช้ `..` ข้ามทั้งหมด เพราะ `_` ในบริบท tuple-like คือ "ไม่สนใจค่าตำแหน่งนี้"
  ทีละตำแหน่ง)
- ใน method `describe`, `PaymentMethod::CreditCard { number, .. }` คือการ destructure แบบ **ผสม**: ดึงค่า `number`
  มาใช้ (ตั้งชื่อตัวแปรเดียวกับชื่อ field พอดี ซึ่งเป็น shorthand ที่นิยมมากในโค้ด Rust จริง) แต่ไม่สนใจ `cvv` เลย
  (ใช้ `..` ข้ามไป) แสดงให้เห็นว่าใน pattern เดียวกัน คุณเลือกได้ว่าจะดึงค่าไหนมาใช้และปล่อยไหนทิ้งไปแบบละเอียด
- `&number[number.len() - 4..]` — การตัด string เอาแค่ 4 ตัวอักษรสุดท้าย ใช้ range indexing บน `&str` (จาก Part 8
  เรื่อง Slices) — ผลลัพธ์คือ `&str` ที่เป็น slice ของ `number` ตัวเดิม ไม่ได้สร้างข้อมูลใหม่

จุดที่ควรสังเกตเชิงลึก: `match self` แบบนี้เป็นรูปแบบที่พบบ่อยที่สุดเมื่อเขียน method บน enum เพราะพฤติกรรมของ
method มักจะ "แตกกิ่ง" ไปตาม variant อยู่แล้วโดยธรรมชาติ ซึ่งต่างจากการเขียน method บน struct ที่มักทำงานกับ field
ชุดเดียวกันตรง ๆ โดยไม่ต้อง match อะไร — นี่คือความแตกต่างเชิง idiom ระหว่าง struct และ enum ที่คุณจะเห็นซ้ำ ๆ
ตลอดทั้งหลักสูตรนี้

### 10.5 `match` แบบเต็มรูปแบบ และ Exhaustiveness Checking

Part 4 แนะนำ `match` แบบผิวเผินไปแล้ว บทนี้จะอธิบายเต็มรูปแบบ เริ่มจาก syntax ทั่วไปของ `match`:

```rust
match ค่าที่ต้องการตรวจสอบ {
    pattern1 => expression1,
    pattern2 => expression2,
    pattern3 => expression3,
    // ...
    _ => default_expression, // ถ้าจำเป็น
}
```

`match` ทำงานโดยเทียบค่าที่ต้องการตรวจสอบกับ pattern **จากบนลงล่างตามลำดับที่เขียน** เจอ pattern ไหนตรงกันก่อน
ก็รันโค้ดฝั่งขวาของ `=>` แล้ว**ออกจาก match ทันที** ไม่ไปเช็ค pattern ที่เหลือต่อ (ต่างจาก `switch` ใน C/Java/JS
ที่ default จะ "fall through" ไป case ถัดไปถ้าไม่มี `break` — Rust ไม่มีปัญหานี้เลยเพราะ match ไม่ fall through
โดยธรรมชาติ)

#### Exhaustiveness Checking: หัวใจสำคัญที่สุดของ `match`

สิ่งที่ทำให้ `match` ของ Rust ต่างจาก `switch` ของภาษาอื่นอย่างมากคือ **compiler บังคับให้ pattern ทั้งหมดครอบคลุม
ทุกความเป็นไปได้ของค่านั้น (exhaustive)** ลองดูว่าเกิดอะไรขึ้นถ้าเราลืม handle variant หนึ่งไป:

```rust
enum PaymentMethod {
    Cash,
    CreditCard,
    BankTransfer,
}

fn describe(method: PaymentMethod) -> &'static str {
    match method {
        PaymentMethod::Cash => "เงินสด",
        PaymentMethod::CreditCard => "บัตรเครดิต",
        // ลืม handle PaymentMethod::BankTransfer!
    }
}

fn main() {}
```

โค้ดนี้ **compile ไม่ผ่าน** ได้ error ตรง ๆ ทันที:

```
error[E0004]: non-exhaustive patterns: `PaymentMethod::BankTransfer` not covered
  --> src/main.rs:8:11
   |
 8 |     match method {
   |           ^^^^^^ pattern `PaymentMethod::BankTransfer` not covered
   |
note: `PaymentMethod` defined here
  --> src/main.rs:1:6
   |
 1 | enum PaymentMethod {
   |      ^^^^^^^^^^^^^
...
 4 |     BankTransfer,
   |     ------------ not covered
   = note: the matched value is of type `PaymentMethod`
help: ensure that all possible cases are being handled by adding a match arm with a wildcard pattern or an explicit pattern as shown
   |
10 ~         PaymentMethod::CreditCard => "บัตรเครดิต",
11 ~         PaymentMethod::BankTransfer => todo!(),
   |
```

สังเกตว่า error message ระบุตรง ๆ ว่า pattern ไหนที่ **ไม่ครอบคลุม** (`PaymentMethod::BankTransfer not covered`)
พร้อมชี้ไปที่จุดประกาศ variant นั้นเลย (`not covered` ตรง `BankTransfer,` ใน enum definition) และยังเสนอวิธีแก้
ให้ทันที (`help: ... adding a match arm ...`) — นี่คือ error message ที่ออกแบบมาให้เป็น "เพื่อนช่วยสอน" แบบเดียวกับ
ที่เราเห็นใน Part 3 (error E0384 เรื่อง immutable variable)

#### ทำไม Exhaustiveness Checking ถึงเป็น "feature" ไม่ใช่ "ความน่ารำคาญ"

มือใหม่บางคนอาจรู้สึกว่า "ทำไม Rust ต้องเข้มงวดจัง ทำไมไม่ปล่อยให้ match ไม่ครบก็ได้ ถ้ากรณีนั้นไม่เกิดขึ้นจริง"
เพื่อเข้าใจว่าทำไมนี่คือของขวัญไม่ใช่ภาระ ลองดูสถานการณ์นี้: สมมติคุณเขียนโค้ดวันนี้ด้วย `enum PaymentMethod` ที่มี
3 variant และเขียน `match` ครอบคลุมครบทั้ง 3 อย่างถูกต้อง โค้ด compile ผ่าน ทำงานดี ผ่านไป 6 เดือน ทีมธุรกิจแจ้งว่า
"ตอนนี้เรารองรับจ่ายด้วย QR code แล้ว" คุณเพิ่ม variant ใหม่:

```rust
enum PaymentMethod {
    Cash,
    CreditCard,
    BankTransfer,
    QrCode, // เพิ่มใหม่!
}
```

ถ้าเป็นภาษาที่ใช้ `switch`/`if-else` แบบ C, Java, หรือ JavaScript — โค้ดเก่าทุกจุดที่ `switch` บน `PaymentMethod`
จะยังคง **compile ผ่านเหมือนเดิม** เพราะภาษาพวกนี้ไม่ได้บังคับ exhaustiveness นั่นแปลว่า logic ที่คุณเขียนไว้ 6 เดือน
ก่อน (เช่นฟังก์ชันคำนวณค่าธรรมเนียม, ฟังก์ชันแสดงข้อความในหน้าสรุปคำสั่งซื้อ, ฟังก์ชันส่ง notification) จะ **เงียบ ๆ
ไม่ทำอะไรเลยหรือ fall through ไปกรณี default ที่ผิด** สำหรับลูกค้าทุกคนที่เลือกจ่ายด้วย QR code — และคุณจะไม่รู้ตัว
เลยจนกว่าจะมีใครมา report บั๊ก หรือแย่กว่านั้นคือไม่มีใคร report เลยแต่ธุรกิจเสียหายแบบเงียบ ๆ (เช่นลืมคิดค่าธรรมเนียม
QR code ไปเป็นเดือน ๆ)

แต่ใน Rust สิ่งที่เกิดขึ้นคือ: **ทุกจุดในโค้ดทั้งโปรเจกต์ที่ `match` บน `PaymentMethod` แบบไม่มี `_` catch-all จะ
compile ไม่ผ่านทันที** พร้อม error `E0004` ที่ชี้ตรงไปยังบรรทัดที่ต้อง handle `QrCode` เพิ่ม — คุณจะเห็นรายชื่อทุกจุด
ที่ต้องแก้ (ผ่าน `cargo build` ตัวเดียว) ครบถ้วนแบบไม่มีทางพลาดแม้แต่จุดเดียว ต้องเพิ่ม logic ให้ `QrCode` ในทุกจุด
ก่อนโปรแกรมจะ compile ผ่านได้อีกครั้ง — นี่คือความหมายที่แท้จริงของประโยค "Rust's compiler catches bugs at compile
time" ที่เราพูดถึงตั้งแต่ Part 1: **การเพิ่ม variant ใหม่ในระบบใหญ่ที่มี match กระจายอยู่หลายสิบ-หลายร้อยจุด
กลายเป็นงานที่ปลอดภัย 100% เพราะ compiler ไล่บอกทุกจุดที่ลืมแก้ให้เอง** ไม่ต้องจำเอง ไม่ต้อง grep หาทุกจุดที่อ้างถึง
enum นี้ในโปรเจกต์เอง

เทียบให้เห็นภาพชัด ๆ:

| สถานการณ์ | C / Java / JavaScript (`switch`) | Rust (`match`) |
|---|---|---|
| เพิ่ม enum variant ใหม่ | โค้ดเก่า compile ผ่านเหมือนเดิม แต่ทำงานผิดเงียบ ๆ (fall through/ไม่เข้า case ไหน) | compile ไม่ผ่านทุกจุดที่ยังไม่ handle variant ใหม่ — ต้องแก้ให้ครบก่อนถึงจะ build ได้ |
| ตรวจพบปัญหาเมื่อไหร่ | ตอน runtime (ถ้าโชคดีมี test/QA เจอ) หรืออาจไม่เจอเลย | ตอน compile time — ก่อนโค้ดจะรันด้วยซ้ำ |
| ความเสี่ยงของทีมใหญ่ | สูง — สมาชิกทีมอื่นอาจไม่รู้ว่าต้องไปแก้ที่ไหนบ้าง | ต่ำ — compiler ระบุตำแหน่งให้ครบถ้วนอัตโนมัติ |

นี่คือเหตุผลที่นักพัฒนา Rust ที่มีประสบการณ์มักบอกว่า `match` + `enum` คือ "refactoring superpower" — การเปลี่ยนแปลง
โครงสร้างข้อมูลหลักของระบบ (เช่นเพิ่มตัวเลือกใหม่) ที่ในภาษาอื่นเป็นงานที่น่ากังวลมาก (เพราะกลัวพลาดจุดใดจุดหนึ่ง)
กลายเป็นงานที่ทำได้อย่างมั่นใจใน Rust เพราะ compiler เป็น safety net ให้เสมอ

#### `match` เป็น Expression: คืนค่าได้เหมือน `if/else`

เช่นเดียวกับที่ Part 4 อธิบายไว้ `match` เป็น expression เขียนอยู่ฝั่งขวาของ `let` ได้ตรง ๆ และทุก arm ต้องคืนค่า
ที่มี **type เดียวกัน**:

```rust
fn describe_status_code(code: u32) -> String {
    let message = match code {
        200 => "สำเร็จ".to_string(),
        404 => "ไม่พบข้อมูล".to_string(),
        500 => "เซิร์ฟเวอร์ขัดข้อง".to_string(),
        _ => format!("ไม่รู้จักโค้ด {code}"),
    };
    message
}

fn main() {
    println!("{}", describe_status_code(200));
    println!("{}", describe_status_code(999));
}
```

ผลลัพธ์:

```
สำเร็จ
ไม่รู้จักโค้ด 999
```

ทุก arm ในตัวอย่างนี้คืนค่า `String` เหมือนกันหมด (สาม arm แรกใช้ `.to_string()`, arm สุดท้ายใช้ `format!` ที่ก็คืน
`String` เช่นกัน) เราจะเห็นในหัวข้อ "กับดักที่พบบ่อย" ว่าถ้า arm คืนค่า type ต่างกัน จะเกิด error ทันที

### 10.6 `_` Catch-all Pattern และ `other => ...` Binding Catch-all

ในหลายกรณี เราไม่จำเป็นต้อง handle ทุก variant/ค่าแยกกันหมด แต่ต้องการ "จับกรณีที่เหลือทั้งหมด" ไว้ในที่เดียว
Rust มีสอง pattern สำหรับสิ่งนี้:

```rust
enum HttpStatus {
    Ok,
    NotFound,
    ServerError,
    Teapot,
}

fn describe(status: &HttpStatus) -> &'static str {
    match status {
        HttpStatus::Ok => "OK",
        HttpStatus::NotFound => "Not Found",
        _ => "Error", // catch-all แบบไม่สนใจว่าค่าที่แท้จริงคือตัวไหน
    }
}

fn classify(code: u32) -> String {
    match code {
        200 => "OK".to_string(),
        404 => "Not Found".to_string(),
        other => format!("Unknown code: {other}"), // catch-all แบบ "จับค่าไว้ใช้"
    }
}

fn main() {
    println!("{}", describe(&HttpStatus::ServerError));
    println!("{}", describe(&HttpStatus::Teapot));
    println!("{}", classify(200));
    println!("{}", classify(999));
}
```

ผลลัพธ์:

```
Error
Error
OK
Unknown code: 999
```

ความแตกต่างระหว่าง `_` กับ `other` (หรือชื่ออื่นที่ไม่ใช่ `_`):

- **`_` (underscore)** — จับทุกกรณีที่เหลือ แต่**ไม่ผูกค่านั้นเข้ากับตัวแปรใด ๆ** ใช้เมื่อไม่สนใจค่าจริงเลยว่าคือ
  อะไร ในตัวอย่าง `describe`, ไม่ว่า `status` จะเป็น `ServerError` หรือ `Teapot` เราตอบ `"Error"` เหมือนกันหมด
  โดยไม่ต้องรู้ว่าเป็นตัวไหน
- **`other => ...`** (หรือชื่ออื่นตามที่ต้องการ เช่น `code`, `x`) — จับทุกกรณีที่เหลือ **และผูกค่านั้นเข้ากับตัวแปร
  `other` ให้ใช้งานต่อในฝั่งขวาของ `=>`** ในตัวอย่าง `classify`, เราต้องการรู้ว่า "code ที่ไม่รู้จักคือเลขอะไร"
  เพื่อเอาไปแสดงใน error message จึงต้องผูกค่าไว้แทนที่จะทิ้งไปด้วย `_`

กฎสำคัญที่ต้องจำ: **`match` เทียบ pattern จากบนลงล่าง และ `_`/catch-all ต้องอยู่ตำแหน่งสุดท้ายเสมอ (หรืออย่างน้อย
ต้องอยู่หลัง pattern เฉพาะเจาะจงทั้งหมดที่ต้องการให้ทำงานก่อน)** เพราะถ้า catch-all มาก่อน pattern เฉพาะที่ตามมา
จะไม่มีวันถูกใช้เลย ซึ่งเราจะเห็น error/warning จริงในหัวข้อ "กับดักที่พบบ่อย"

### 10.7 Pattern ที่ Destructure: Enum Variant, Tuple, Range, Literal, Or-pattern

`match` ไม่ได้ทำได้แค่เทียบว่า "เป็น variant ไหน" — มันคือระบบ **pattern matching** ที่ทรงพลังมากกว่านั้นมาก
มาดู pattern แต่ละแบบทีละประเภท

#### 10.7.1 Destructure Enum Variant (ทวนจากหัวข้อก่อน)

เราเห็นมาแล้วในหัวข้อ 10.3-10.4 ว่า struct-like variant destructure ด้วย `{ field1, field2 }` และ tuple-like
variant destructure ด้วย `(binding1, binding2)` — นี่คือ pattern ประเภทแรกที่สำคัญที่สุดเมื่อทำงานกับ enum

#### 10.7.2 Literal Pattern: จับค่าตรง ๆ

pattern ที่ง่ายที่สุดคือ literal ตรง ๆ ทั้งตัวเลข, string, bool, char:

```rust
fn day_name(day_number: u8) -> &'static str {
    match day_number {
        1 => "จันทร์",
        2 => "อังคาร",
        3 => "พุธ",
        4 => "พฤหัสบดี",
        5 => "ศุกร์",
        6 => "เสาร์",
        7 => "อาทิตย์",
        _ => "ไม่ใช่วันที่ถูกต้อง",
    }
}

fn main() {
    println!("{}", day_name(3));
    println!("{}", day_name(9));
}
```

ผลลัพธ์:

```
พุธ
ไม่ใช่วันที่ถูกต้อง
```

#### 10.7.3 Range Pattern: จับช่วงของค่า

เมื่อมีหลายค่าติดกันที่ต้องการให้ผลลัพธ์เดียวกัน เขียนทีละบรรทัดจะยาวมาก Rust ให้เขียน pattern เป็นช่วง (range)
ได้ด้วย `..=` (inclusive — รวมค่าปลายทั้งสองข้าง — เหมือน range ที่เรียนใน Part 4/8):

```rust
fn grade(score: u32) -> char {
    match score {
        90..=100 => 'A',
        80..=89 => 'B',
        70..=79 => 'C',
        60..=69 => 'D',
        _ => 'F',
    }
}

fn main() {
    println!("{}", grade(95)); // A
    println!("{}", grade(72)); // C
    println!("{}", grade(40)); // F
}
```

ผลลัพธ์:

```
A
C
F
```

Range pattern ใช้ได้กับชนิดข้อมูลที่เทียบลำดับได้ (`char`, integer ทุกชนิด) แต่ **ใช้กับ `f32`/`f64` ไม่ได้** เพราะ
floating-point (ที่เรียนไว้ใน Part 3 เรื่องความไม่เที่ยงตรงของทศนิยม) ไม่เหมาะกับการเทียบแบบ pattern ที่ต้องการ
ความแน่นอนเชิง discrete แบบนี้ — ถ้าต้องเช็คช่วงของ `f64` ให้ใช้ match guard (หัวข้อ 10.8) แทน

#### 10.7.4 Or-pattern: จับหลายค่าด้วย pattern เดียว

ใช้ `|` (pipe) คั่นระหว่างหลาย pattern ที่ต้องการให้ทำงานแบบเดียวกัน:

```rust
fn is_weekend(day: u8) -> bool {
    match day {
        6 | 7 => true,
        _ => false,
    }
}

fn describe_vowel_or_consonant(c: char) -> &'static str {
    match c {
        'a' | 'e' | 'i' | 'o' | 'u' => "สระ",
        'a'..='z' => "พยัญชนะ",
        _ => "ไม่ใช่ตัวอักษรภาษาอังกฤษพิมพ์เล็ก",
    }
}

fn main() {
    println!("{}", is_weekend(6));
    println!("{}", is_weekend(3));
    println!("{}", describe_vowel_or_consonant('a'));
    println!("{}", describe_vowel_or_consonant('b'));
    println!("{}", describe_vowel_or_consonant('9'));
}
```

ผลลัพธ์:

```
true
false
สระ
พยัญชนะ
9 -- ไม่ใช่ตัวอักษรภาษาอังกฤษพิมพ์เล็ก
```

(หมายเหตุ: บรรทัดสุดท้ายพิมพ์ค่าที่ `describe_vowel_or_consonant('9')` คืนมาคือ `"ไม่ใช่ตัวอักษรภาษาอังกฤษพิมพ์เล็ก"`
โค้ดข้างบนไม่ได้พิมพ์ตัวอักษรกำกับ แก้ไขจากตัวอย่างเพื่อความชัดเจน — ผลลัพธ์จริงจากการรันคือบรรทัดสุดท้ายเป็น
`ไม่ใช่ตัวอักษรภาษาอังกฤษพิมพ์เล็ก` เท่านั้น)

สังเกตว่า `describe_vowel_or_consonant` ผสม or-pattern (`'a' | 'e' | 'i' | 'o' | 'u'`) กับ range pattern
(`'a'..='z'`) ในฟังก์ชันเดียวกันได้ — Rust เทียบจากบนลงล่างเหมือนเดิม สระถูกจับก่อนเพราะ pattern สระเขียนไว้ก่อน
pattern range ตัวอักษรทั่วไป (ถ้าสลับลำดับ สระจะไม่ถูกแยกออกมาต่างหากเลย เพราะ range `'a'..='z'` ครอบคลุมสระอยู่แล้ว)

#### 10.7.5 Destructure Tuple

`match` บน tuple ทำได้ตรง ๆ และ **ผสมกับ match guard ได้** (จะอธิบายละเอียดในหัวข้อ 10.8) ตัวอย่างคลาสสิกคือหา
ควอดรันต์ของจุดในระบบพิกัด:

```rust
fn quadrant(point: (i32, i32)) -> &'static str {
    match point {
        (0, 0) => "จุดกำเนิด (origin)",
        (_, 0) => "อยู่บนแกน x",
        (0, _) => "อยู่บนแกน y",
        (x, y) if x > 0 && y > 0 => "ควอดรันต์ที่ 1",
        (x, y) if x < 0 && y > 0 => "ควอดรันต์ที่ 2",
        (x, y) if x < 0 && y < 0 => "ควอดรันต์ที่ 3",
        _ => "ควอดรันต์ที่ 4",
    }
}

fn main() {
    println!("{}", quadrant((3, 4)));   // ควอดรันต์ที่ 1
    println!("{}", quadrant((-3, 4)));  // ควอดรันต์ที่ 2
    println!("{}", quadrant((-3, -4))); // ควอดรันต์ที่ 3
    println!("{}", quadrant((3, -4)));  // ควอดรันต์ที่ 4
    println!("{}", quadrant((0, 0)));   // จุดกำเนิด
    println!("{}", quadrant((5, 0)));   // อยู่บนแกน x
}
```

ผลลัพธ์:

```
ควอดรันต์ที่ 1
ควอดรันต์ที่ 2
ควอดรันต์ที่ 3
ควอดรันต์ที่ 4
จุดกำเนิด (origin)
อยู่บนแกน x
```

`(0, 0)` เทียบตำแหน่งทั้งสองเป็น literal ตรง ๆ, `(_, 0)` ใช้ `_` แทนตำแหน่ง `x` เพราะไม่สนใจค่ามัน (สนใจแค่ว่า
`y` เป็น 0), `(x, y) if ...` คือการ destructure ผูกทั้งสองตำแหน่งเป็นตัวแปร `x`, `y` แล้วใช้ match guard ตรวจสอบ
เงื่อนไขเพิ่มเติม — เป็นสะพานเชื่อมไปสู่หัวขัดถัดไปพอดี

#### 10.7.6 Match Ergonomics: `match` บน Reference โดยไม่ต้องเขียน `&` ทุกจุด

ทุกตัวอย่างที่ผ่านมาในบทนี้ที่ใช้ `for m in &methods` หรือรับ `&self` แล้ว `match self`/`match m` ตรง ๆ (ไม่ต้อง
เขียน `match *self` หรือ `&PaymentMethod::Cash`) ทำงานได้เพราะฟีเจอร์ที่ชื่อ **match ergonomics** (เพิ่มเข้ามาใน
Rust 2018) ก่อนฟีเจอร์นี้จะมี นักพัฒนาต้อง dereference และใส่ `ref`/`ref mut` เองด้วยมือทุกจุดที่ match บน reference
ลองดูตัวอย่างเปรียบเทียบทั้งสองสไตล์:

```rust
enum PaymentMethod {
    BankTransfer(String),
}

fn main() {
    let m = PaymentMethod::BankTransfer("123-456".to_string());

    // สไตล์ก่อนปี 2018 (ยังใช้ได้อยู่ แต่ไม่มีใครเขียนแบบนี้แล้วในโค้ดปัจจุบัน)
    match &m {
        &PaymentMethod::BankTransfer(ref account) => println!("แบบเก่า: {account}"),
    }

    // สไตล์ match ergonomics (มาตรฐานปัจจุบัน — Rust "เดา" ให้เองว่าต้องยืม ไม่ต้องยึด ownership)
    match &m {
        PaymentMethod::BankTransfer(account) => println!("แบบใหม่: {account}"),
    }
}
```

ผลลัพธ์:

```
แบบเก่า: 123-456
แบบใหม่: 123-456
```

**อธิบายความแตกต่าง**: ใน pattern แบบเก่า เราต้องเขียน `&PaymentMethod::BankTransfer(ref account)` — เครื่องหมาย
`&` ด้านหน้าคือการ "แกะ" reference ของ `&m` ออกก่อน จึงจะเข้าถึงตัว `PaymentMethod::BankTransfer` ข้างในได้ และ
`ref account` คือการบอกว่า "ผูก `account` เป็น reference (`&String`) ไม่ใช่ยึด ownership" (ถ้าไม่มี `ref` compiler
จะพยายาม move `String` ออกมาจาก reference ที่เรายืมมา ซึ่งจะเกิด error แบบเดียวกับ E0507 ที่จะเห็นในหัวข้อ
"กับดักที่พบบ่อย") ในสไตล์ปัจจุบัน เราแค่เขียน `PaymentMethod::BankTransfer(account)` ตรง ๆ เหมือนกำลัง match
บนค่าจริง แล้ว **compiler จะวิเคราะห์เองว่า `&m` คือ reference จึงปรับให้ `account` เป็น `&String` โดยอัตโนมัติ**
โดยไม่ต้องเขียน `&`/`ref` เพิ่มเลย — นี่คือสิ่งที่ทำให้ pattern ทั้งหมดในบทนี้ (เช่น `PaymentMethod::CreditCard {
number, .. }` ตอน match บน `self: &Self`) ดูสะอาดและไม่ต้องคิดเรื่อง reference/ownership ซ้ำซ้อนตลอดเวลา

ข้อควรจำที่สำคัญ: match ergonomics **ไม่ได้ทำให้ ownership/borrowing หายไป** มันแค่ทำให้ไม่ต้อง**เขียน**สัญลักษณ์
`&`/`ref` เอง — กฎของ ownership และ borrow checker จาก Part 6-7 ยังบังคับเหมือนเดิมทุกอย่าง (field ที่ผูกมาจาก
match บน `&Enum` ก็ยังเป็น reference ที่ยืมมาอยู่ดี ยึด ownership ออกไปตรง ๆ ไม่ได้ — ตามที่จะเห็นในกับดักข้อ 5)

#### 10.7.7 Binding ค่าไว้พร้อมกับตรวจ Pattern ด้วย `@`

บางครั้งเราต้องการทั้ง "ตรวจว่าค่าตรงกับ pattern แบบไหน" (เช่น range) **และ** "เก็บค่านั้นไว้ใช้ต่อ" ในเวลาเดียวกัน
— ใช้ `@` (at-binding) ผูกชื่อตัวแปรไว้กับ pattern ทั้งก้อนได้:

```rust
fn categorize_temperature(temp: i32) -> String {
    match temp {
        t @ i32::MIN..=0 => format!("{t} องศา: เยือกแข็งหรือต่ำกว่า"),
        t @ 1..=25 => format!("{t} องศา: เย็นสบาย"),
        t @ 26..=35 => format!("{t} องศา: ร้อน"),
        t => format!("{t} องศา: ร้อนจัด"),
    }
}

fn main() {
    println!("{}", categorize_temperature(-5));
    println!("{}", categorize_temperature(20));
    println!("{}", categorize_temperature(30));
    println!("{}", categorize_temperature(40));
}
```

ผลลัพธ์:

```
-5 องศา: เยือกแข็งหรือต่ำกว่า
20 องศา: เย็นสบาย
30 องศา: ร้อน
40 องศา: ร้อนจัด
```

`t @ 1..=25` อ่านว่า "ถ้าค่าตรงกับช่วง `1..=25` ให้ผูกค่านั้นไว้กับชื่อ `t`" — ถ้าเขียนแค่ `1..=25 => ...` โดยไม่มี
`t @` เราจะรู้แค่ว่าค่าอยู่ในช่วงนี้ แต่**ไม่มีทางรู้ค่าจริงที่ match เข้ามา**เพื่อเอาไปใช้ในฝั่งขวาของ `=>` เลย
(ในตัวอย่างนี้เราต้องใช้ค่าจริงไปแสดงใน `format!` ด้วย จึงขาด `t @` ไปไม่ได้) `@` ใช้ผสมกับ enum ที่มีข้อมูลได้เช่นกัน
เช่นตรวจว่า `Option<i32>` มีค่าอยู่ในช่วงที่สนใจหรือไม่ พร้อมผูกค่านั้นไว้:

```rust
fn describe(opt: Option<i32>) -> String {
    match opt {
        Some(n @ 1..=5) => format!("ค่าน้อย: {n}"),
        Some(n) => format!("ค่า: {n}"),
        None => "ไม่มีค่า".to_string(),
    }
}

fn main() {
    println!("{}", describe(Some(3)));
    println!("{}", describe(Some(100)));
    println!("{}", describe(None));
}
```

ผลลัพธ์:

```
ค่าน้อย: 3
ค่า: 100
ไม่มีค่า
```

`Some(n @ 1..=5)` คือการ destructure `Option` (เอาค่าข้างในออกมา) **และ** ตรวจช่วงของค่านั้น **และ** ผูกชื่อ `n`
ไว้ใช้ ทั้งหมดในคำสั่งเดียว — ถ้าไม่มี `@` คุณจะต้องเขียนแยกเป็น `Some(n) if n >= 1 && n <= 5 => ...` (ใช้ match
guard แทน) ซึ่งก็ทำงานเหมือนกันได้ แต่ `@` จะกระชับกว่าเมื่อ pattern ที่ต้องการตรวจเป็น range/literal ตรง ๆ

### 10.8 Match Guards: เพิ่มเงื่อนไขให้ Pattern ด้วย `if`

**Match guard** คือการเติมเงื่อนไข `if` ต่อจาก pattern เพื่อ "กรอง" ให้ละเอียดขึ้นกว่าการจับคู่รูปร่าง (shape)
เฉย ๆ — pattern บอกว่า "รูปร่างข้อมูลตรงกับอะไร" ส่วน guard บอกว่า "แล้วต้องมีเงื่อนไขอะไรเพิ่มด้วย":

```rust
fn describe_number(n: i32) -> String {
    match n {
        x if x < 0 => format!("{x} เป็นจำนวนลบ"),
        0 => "ศูนย์".to_string(),
        x if x % 2 == 0 => format!("{x} เป็นจำนวนคู่บวก"),
        x => format!("{x} เป็นจำนวนคี่บวก"),
    }
}

fn main() {
    println!("{}", describe_number(-5)); // -5 เป็นจำนวนลบ
    println!("{}", describe_number(0));  // ศูนย์
    println!("{}", describe_number(4));  // 4 เป็นจำนวนคู่บวก
    println!("{}", describe_number(7));  // 7 เป็นจำนวนคี่บวก
}
```

สังเกตว่า `x if x < 0` คือ pattern `x` (จับค่าอะไรก็ได้ผูกกับชื่อ `x`) ตามด้วย guard `if x < 0` (แต่ต้องเป็นค่าลบ
เท่านั้นถึงจะเข้า arm นี้) ถ้าเงื่อนไข guard เป็น `false` การจับคู่ arm นั้นจะถือว่า **ไม่สำเร็จ** และ `match` จะไป
ลองเทียบ pattern ถัดไปต่อ (ไม่ใช่ error หรือ panic — เป็นพฤติกรรมปกติของ `match`)

Match guard ที่จะเจอบ่อยที่สุดในโค้ดจริงคือคู่กับ `Option`/`Result` (จะเรียนละเอียดเรื่อง `Option` ในหัวข้อถัดไป
และ `Result` เต็มรูปแบบใน Part 12):

```rust
fn categorize(opt: Option<i32>) -> &'static str {
    match opt {
        Some(x) if x > 5 => "มากกว่า 5",
        Some(_) => "0 ถึง 5",
        None => "ไม่มีค่า",
    }
}

fn main() {
    println!("{}", categorize(Some(10))); // มากกว่า 5
    println!("{}", categorize(Some(2)));  // 0 ถึง 5
    println!("{}", categorize(None));     // ไม่มีค่า
}
```

`Some(x) if x > 5` คือการ destructure `Option` ก่อน (ดึงค่าข้างในออกมาเป็น `x` ถ้ามี) แล้วค่อยเช็คเงื่อนไขเพิ่ม
ด้วย guard — นี่คือรูปแบบที่ทรงพลังมาก เพราะรวม "การพิสูจน์ว่ามีค่าอยู่จริง" (`Some(x)` เทียบกับ `None`) เข้ากับ
"การเช็คเงื่อนไขของค่านั้น" (`x > 5`) ไว้ในที่เดียว อ่านเข้าใจง่ายกว่าการเขียน `if let` ซ้อน `if` หลายชั้นมาก

**ข้อควรระวัง**: ตัวแปรที่ผูกด้วย pattern (เช่น `x` ใน `Some(x) if x > 5`) ใช้ได้แค่ในฝั่ง guard (`if x > 5`) และ
ฝั่งขวาของ `=>` ของ arm นั้นเท่านั้น ใช้ arm อื่นไม่ได้ (เพราะแต่ละ arm เป็น scope ของตัวเอง) และ guard **ไม่มีผลต่อ
exhaustiveness checking** — แม้ arm หนึ่งจะมี guard ที่ดูเหมือนครอบคลุมทุกกรณีของ pattern นั้น (เช่น `x if true`)
compiler ก็ยังมองว่า pattern `x` ธรรมดา (ไม่มี guard) คือสิ่งที่ครอบคลุม ไม่ใช่ตัว guard เพราะ guard เป็นเงื่อนไข
ที่ตรวจตอน runtime ซึ่ง compiler ไม่สามารถพิสูจน์ได้ล่วงหน้าตอน compile time ว่าจะเป็นจริงเสมอหรือไม่

### 10.9 `Option<T>`: วิธีที่ Rust แก้ "Billion Dollar Mistake" ของ Null

Tony Hoare ผู้คิดค้นแนวคิด null reference ขึ้นในภาษา ALGOL W เมื่อปี 1965 เคยกล่าวในการบรรยายปี 2009 ว่านี่คือ
**"ความผิดพลาดมูลค่าพันล้านดอลลาร์" (billion dollar mistake)** ของเขา เพราะ null ทำให้เกิดบั๊กประเภท
`NullPointerException` (Java), `undefined is not a function` (JavaScript), `AttributeError: 'NoneType' object
has no attribute` (Python), หรือ segmentation fault จาก null pointer dereference (C/C++) นับไม่ถ้วนตลอด 60 ปี
ที่ผ่านมา — ปัญหาหลักคือ **ในภาษาส่วนใหญ่ ตัวแปรที่ประกาศว่าเป็น type `T` (เช่น `String`, `User`, `int`) แต่จริง ๆ
แล้วอาจเป็น `null` ก็ได้เสมอ** โดย type system ไม่ได้บอกความแตกต่างนี้ไว้ตรง ๆ เลย ทำให้นักพัฒนาต้องคอยจำเองว่า
ค่าตัวไหน "อาจเป็น null" ต้องเช็คก่อนใช้ และเมื่อลืมเช็คสักครั้งเดียว โปรแกรมก็ล่มตอน runtime

Rust เลือกวิธีแก้ที่แตกต่างไปเลย: **ไม่มี `null` ในภาษาเลย (ในส่วนของ safe Rust)** แทนที่ Rust ใช้ enum ธรรมดา ๆ
ที่ชื่อ `Option<T>` ซึ่งอยู่ใน standard library และถูก import มาให้อัตโนมัติทุกไฟล์ (อยู่ใน "prelude" — จะเรียนคำนี้
ละเอียดใน Part 16) โครงสร้างของมันมีรูปร่างเดียวกับที่คุณเรียนมาในหัวข้อ 10.3 เป๊ะ ๆ:

```rust
mod my_option_illustration {
    // นี่คือโครงสร้างเดียวกับ Option<T> ที่ Rust standard library นิยามไว้ให้เราใช้งานอัตโนมัติ
    // (เขียนซ้ำที่นี่แค่เพื่อให้เห็นภาพ — ใช้ชื่อ MaybeValue กันชนกับ Option ตัวจริงที่มีอยู่แล้วในทุกไฟล์)
    #[derive(Debug)]
    pub enum MaybeValue<T> {
        Some(T),
        None,
    }
}

fn main() {
    use my_option_illustration::MaybeValue;
    let has_value: MaybeValue<i32> = MaybeValue::Some(42);
    let no_value: MaybeValue<i32> = MaybeValue::None;
    println!("{:?} {:?}", has_value, no_value);
}
```

ผลลัพธ์:

```
Some(42) None
```

`Option<T>` ตัวจริงในมาตรฐานของ Rust นิยามไว้แบบนี้เป๊ะ ๆ (ต่างกันแค่ชื่อ):

```
enum Option<T> {
    Some(T),
    None,
}
```

`T` ในที่นี้คือ **generic type parameter** (จะเรียนรายละเอียดเต็มใน Part 18 เรื่อง Generics — ตอนนี้ให้เข้าใจแค่ว่า
มันคือ "ตัวแทน type อะไรก็ได้ที่จะใส่เข้ามา") เช่น `Option<i32>` คือ Option ที่ถ้ามีค่า ค่านั้นจะเป็น `i32`,
`Option<String>` คือ Option ที่ถ้ามีค่า ค่านั้นจะเป็น `String` เป็นต้น

#### ทำไม `Option<T>` ปลอดภัยกว่า Null

จุดสำคัญที่สุดคือ: **ฟังก์ชันที่คืนค่า `Option<u32>` มี type ที่ต่างจากฟังก์ชันที่คืนค่า `u32` โดยตรงอย่างชัดเจนใน
เชิง type system** ลองดูตัวอย่างค้นหาผู้ใช้จากชื่อ:

```rust
fn find_user_age(name: &str) -> Option<u32> {
    match name {
        "สมชาย" => Some(30),
        "สมหญิง" => Some(25),
        _ => None,
    }
}

fn main() {
    let age = find_user_age("สมชาย");
    match age {
        Some(a) => println!("อายุ {a} ปี"),
        None => println!("ไม่พบข้อมูล"),
    }

    let missing = find_user_age("สมศรี");
    match missing {
        Some(a) => println!("อายุ {a} ปี"),
        None => println!("ไม่พบข้อมูล"),
    }
}
```

ผลลัพธ์:

```
อายุ 30 ปี
ไม่พบข้อมูล
```

`find_user_age` ประกาศ return type เป็น `Option<u32>` ไม่ใช่ `u32` ตรง ๆ — นี่คือการ**ประกาศไว้ในเชิง type ตั้งแต่
ต้นว่า "ฟังก์ชันนี้อาจไม่มีคำตอบให้"** ทุกคนที่เรียกใช้ฟังก์ชันนี้จะได้รับค่าเป็น `Option<u32>` เสมอ ไม่ใช่ `u32`
โดยตรง เมื่อ type ของค่าที่ได้กลับมาคือ `Option<u32>` ไม่ใช่ `u32` **คุณไม่มีทางใช้ค่านั้นเหมือนเป็น `u32` ตรง ๆ
ได้เลยจนกว่าจะ `match`/destructure มันก่อน** — ลองดู error ที่เกิดขึ้นถ้าพยายามข้ามขั้นตอนนี้:

```rust
fn find_user_age(name: &str) -> Option<u32> {
    if name == "สมชาย" { Some(30) } else { None }
}

fn birthday_greeting(age: u32) -> String {
    format!("สุขสันต์วันเกิดอายุ {age} ปี")
}

fn main() {
    let age = find_user_age("สมชาย");
    println!("{}", birthday_greeting(age)); // ส่ง Option<u32> เข้าฟังก์ชันที่ต้องการ u32 ตรง ๆ
}
```

จะได้ error ทันที:

```
error[E0308]: mismatched types
  --> src/main.rs:11:38
   |
11 |     println!("{}", birthday_greeting(age)); // ส่ง Option<u32> เข้าฟังก์ชันที่ต้องการ u32 ตรง ๆ
   |                    ----------------- ^^^ expected `u32`, found `Option<u32>`
   |                    |
   |                    arguments to this function are incorrect
   |
   = note: expected type `u32`
              found enum `Option<u32>`
note: function defined here
  --> src/main.rs:5:4
   |
 5 | fn birthday_greeting(age: u32) -> String {
   |    ^^^^^^^^^^^^^^^^^ --------
help: consider using `Option::expect` to unwrap the `Option<u32>` value, panicking if the value is an `Option::None`
   |
11 |     println!("{}", birthday_greeting(age.expect("REASON"))); // ...
   |                                         +++++++++++++++++
```

นี่คือความแตกต่างเชิงพื้นฐานที่สำคัญที่สุดของ Rust เทียบกับภาษาที่มี null: **ในภาษาอย่าง Java, C#, หรือ TypeScript
(แบบไม่เปิด `strictNullChecks`) ฟังก์ชันที่คืนค่า "อาจเป็น null" มักมี type หน้าตาเหมือนฟังก์ชันที่คืนค่าปกติทุกอย่าง**
เช่น Java's `Integer findUserAge(String name)` — ไม่มีอะไรในซิกเนเจอร์นี้บอกว่าอาจคืน `null` เลย คุณต้องรู้เอง
(จากอ่าน documentation หรือจากประสบการณ์) ว่าต้องเช็ค `null` ก่อนใช้ ถ้าลืมเช็คก็ compile ผ่านสบาย ๆ แล้วไปล่มตอน
runtime แทน แต่ Rust บังคับให้ `age` (type `Option<u32>`) **ต้องถูก unwrap ผ่าน pattern matching ก่อนเสมอ**
ก่อนจะเอาไปใช้เป็น `u32` ได้จริง — เป็นการเปลี่ยน "ต้องจำเช็คเอง" (ซึ่งพลาดได้ตลอด) เป็น "compiler บังคับให้เช็ค"
(ซึ่งพลาดไม่ได้เลย เพราะโปรแกรมจะไม่ compile ถ้าไม่เช็ค)

สังเกต error message ด้านบนด้วยว่า Rust แนะนำ `.expect("REASON")` ให้ ซึ่งเป็นวิธี "unwrap" ค่าจาก `Option` แบบ
เร็ว ๆ (แต่จะ panic ถ้าเป็น `None` — เหมาะกับตอน prototype เท่านั้น) วิธีที่ปลอดภัยและเหมาะสมกว่าในโค้ดจริงคือใช้
`match`, `if let` (หัวข้อถัดไป), หรือ method อื่น ๆ ของ `Option` ที่ **Part 11 จะเจาะลึกเต็มรูปแบบ** — บทนี้เพียง
แนะนำให้รู้จักรูปร่างของ `Option<T>` ในฐานะ enum และรู้ว่า `match` คือกลไกพื้นฐานที่สุดที่ใช้ดึงค่าออกมา เพื่อให้
คุณพร้อมสำหรับ Part 11 ที่จะสอน method อย่าง `.unwrap_or()`, `.map()`, `.and_then()`, และการออกแบบโค้ดที่ปลอดภัย
จาก null อย่างเต็มรูปแบบ

### 10.10 `if let`, `while let`, และ `let else`

หลายครั้งเราสนใจแค่ **pattern เดียว** จาก `match` และไม่สนใจกรณีอื่นเลย (หรือสนใจแค่ทำอย่างอื่นสั้น ๆ ในกรณีอื่น)
เขียนเป็น `match` เต็มรูปแบบก็ได้ แต่จะยาวเกินความจำเป็น Rust จึงมี syntax ย่อ (syntactic sugar) ให้ 3 แบบ

#### `if let`: เมื่อสนใจ pattern เดียว

```rust
fn main() {
    let age: Option<u32> = Some(30);

    // แบบ match เต็มรูปแบบ (ยาวเกินความจำเป็นถ้าสนใจแค่กรณี Some)
    match age {
        Some(a) => println!("อายุ {a} ปี (จาก match)"),
        None => {} // ไม่ต้องทำอะไรเลย แต่ก็ต้องเขียน arm นี้เพราะ match ต้อง exhaustive
    }

    // แบบ if let (กระชับกว่ามาก สื่อความหมายตรงประเด็นกว่า)
    if let Some(a) = age {
        println!("อายุ {a} ปี (จาก if let)");
    }
}
```

ผลลัพธ์:

```
อายุ 30 ปี (จาก match)
อายุ 30 ปี (จาก if let)
```

สังเกตความแตกต่าง: แบบ `match` เต็มรูปแบบ ต้องเขียน arm `None => {}` ที่ไม่ทำอะไรเลย เพียงเพื่อให้ผ่าน exhaustiveness
checking — ทำให้โค้ดดูรก และคนอ่านต้องเสียเวลาคิดว่า "ทำไมมี arm เปล่า ๆ นี้ด้วย" ในขณะที่ `if let Some(a) = age`
สื่อความหมายตรงตัวว่า **"ถ้า `age` match กับ pattern `Some(a)` (คือถ้ามันมีค่า) ให้ทำสิ่งนี้"** — อ่านเข้าใจง่ายกว่า
ทันที และยังใช้ `else` ต่อได้เหมือน `if` ธรรมดา สำหรับกรณีที่ pattern ไม่ตรง:

```rust
fn main() {
    let age: Option<u32> = None;

    if let Some(a) = age {
        println!("อายุ {a} ปี");
    } else {
        println!("ไม่พบข้อมูล");
    }
}
```

**ข้อควรระวัง**: `if let` ไม่มีการบังคับ exhaustiveness เหมือน `match` — เพราะจุดประสงค์ของมันคือ "สนใจแค่ pattern
นี้ กรณีอื่นไม่สนใจก็ได้" (จะเขียน `else` หรือไม่ก็ได้) นี่ทำให้ `if let` เหมาะกับกรณีที่กรณีอื่น "ไม่สำคัญพอจะต้อง
handle" เท่านั้น — ถ้า logic ของคุณจำเป็นต้องพิจารณาทุกกรณีให้ครบถ้วน (เช่นตัวอย่าง `PaymentMethod` ที่ต้องคิด
ค่าธรรมเนียมให้ถูกทุกวิธีจ่าย) ให้ใช้ `match` เต็มรูปแบบเสมอ เพื่อให้ compiler ช่วยเช็คให้

#### `while let`: วนซ้ำจนกว่า pattern จะไม่ตรงอีก

`while let` ทำงานคล้าย `if let` แต่วนซ้ำไปเรื่อย ๆ ตราบใดที่ pattern ยังตรงอยู่ ใช้บ่อยมากกับ `Vec::pop()` ที่คืนค่า
เป็น `Option<T>` (คืน `Some(ค่า)` ถ้ายังมีสมาชิกเหลือ, คืน `None` เมื่อ `Vec` ว่างแล้ว):

```rust
fn main() {
    let mut stack = vec![1, 2, 3];

    while let Some(top) = stack.pop() {
        println!("ป๊อปค่า: {top}");
    }

    println!("stack ว่างแล้ว มีสมาชิกเหลือ {} ตัว", stack.len());
}
```

ผลลัพธ์:

```
ป๊อปค่า: 3
ป๊อปค่า: 2
ป๊อปค่า: 1
stack ว่างแล้ว มีสมาชิกเหลือ 0 ตัว
```

ทุกครั้งที่วนซ้ำ `stack.pop()` จะถูกเรียกใหม่ ถ้าคืน `Some(ค่า)` loop body จะรัน (ในที่นี้ปริ้น `top` ที่ผูกไว้)
เมื่อ `stack` ว่างและ `pop()` คืน `None` แล้ว pattern `Some(top)` จะไม่ match อีกต่อไป และ `while let` จะออกจาก loop
ทันที — สังเกตว่าเราไม่ต้องเช็ค `stack.is_empty()` เองเลย ไม่ต้องมี counter หรือ index ใด ๆ ทั้งสิ้น `while let`
จัดการเงื่อนไขการวนซ้ำให้ผ่าน pattern matching ล้วน ๆ ซึ่งกระชับกว่าการเขียน `loop` + `match` + `break` แบบ manual
มาก (`Vec<T>` และ method `.pop()` จะเรียนละเอียดใน Part 13)

#### `let else`: Unwrap แบบ "early return" (Rust 1.65+)

Rust เวอร์ชัน 1.65 (ปล่อยปลายปี 2022) เพิ่ม syntax ใหม่ที่ชื่อ **`let else`** ซึ่งเหมาะกับสถานการณ์ที่พบบ่อยมาก:
"ถ้า pattern ตรง ให้ผูกค่าไว้ใช้ต่อในโค้ดปกติ (ไม่ต้อง indent เพิ่ม) — ถ้า pattern ไม่ตรง ให้ทำอะไรบางอย่างที่
**ต้องออกจากฟังก์ชัน/block ทันที** (เช่น `return`, `continue`, `break`, หรือ `panic!`)"

```rust
fn parse_positive(input: &str) -> i32 {
    let Ok(n) = input.parse::<i32>() else {
        println!("parse ไม่ได้ ใช้ค่า default");
        return 0;
    };
    // ตรงนี้ n คือ i32 ที่ parse สำเร็จแล้ว ใช้งานต่อได้เลย ไม่ต้อง indent เพิ่ม!
    n
}

fn main() {
    println!("{}", parse_positive("42"));  // 42
    println!("{}", parse_positive("abc")); // parse ไม่ได้ ใช้ค่า default \n 0
}
```

ผลลัพธ์:

```
42
parse ไม่ได้ ใช้ค่า default
0
```

เพื่อให้เห็นประโยชน์ชัดเจน ลองเทียบกับการเขียนแบบเดียวกันด้วย `match` เต็มรูปแบบ:

```rust
fn parse_positive_verbose(input: &str) -> i32 {
    let n = match input.parse::<i32>() {
        Ok(value) => value,
        Err(_) => {
            println!("parse ไม่ได้ ใช้ค่า default");
            return 0;
        }
    };
    n
}
```

ทั้งสองแบบทำงานเหมือนกันทุกประการ แต่สังเกตว่าแบบ `match` ต้องมีการ `match` แล้วสร้างตัวแปรใหม่จาก arm `Ok`
ในขณะที่ `let else` เขียนแบบเดียวกับ `let` ธรรมดาที่ผูก `n` ตรง ๆ จาก pattern `Ok(n)` เพียงแต่ต่อท้ายด้วย
`else { ... }` สำหรับกรณีที่ผูกไม่ได้ — และ **บล็อกของ `else` ต้องจบด้วยการออกจาก control flow ปัจจุบันเสมอ**
(`return`, `continue`, `break`, `panic!`, หรือ `std::process::exit`) เพราะถ้าไม่ออก จะไม่มีทางได้ค่า `n` มาให้ใช้
ในบรรทัดถัดไปเลย (compiler จะฟ้อง error ถ้า block ของ `else` ไม่จบด้วยการออกแบบนี้) รูปแบบนี้เหมาะมากกับ pattern
ของโค้ดที่เรียกว่า **"early return"** — ตรวจสอบเงื่อนไขและออกจากฟังก์ชันเร็วที่สุดถ้าไม่ผ่าน แล้วปล่อยให้ส่วนที่เหลือ
ของฟังก์ชันทำงานกับข้อมูลที่ "การันตีแล้วว่าถูกต้อง" โดยไม่ต้อง indent เข้าไปในกิ่ง `if`/`match` อีกชั้น ทำให้โค้ด
อ่านเป็นเส้นตรงมากขึ้นเมื่อมีการตรวจสอบหลายชั้นต่อกัน

**เมื่อไหร่ควรใช้อะไร** สรุปสั้น ๆ:

| ต้องการ | ใช้ |
|---|---|
| ต้อง handle **ทุกกรณี** ให้ครบ (ต้องการ exhaustiveness checking) | `match` |
| สนใจ pattern เดียว กรณีอื่นไม่สำคัญ หรือมีแค่ 2 ทาง (ตรง/ไม่ตรง) | `if let` / `if let ... else` |
| วนซ้ำจนกว่า pattern จะไม่ตรง | `while let` |
| ต้องการ unwrap pattern แล้วใช้ค่าต่อแบบไม่ indent เพิ่ม โดยกรณีไม่ตรงต้อง "ออก" จาก scope ทันที | `let else` |

### 10.11 ตัวอย่างจริง: State Machine ของออเดอร์สั่งซื้อสินค้า

กลับมาที่แนวคิด **"illegal states unrepresentable"** จากหัวข้อ 10.1 — คราวนี้เราจะออกแบบระบบสถานะออเดอร์สั่งซื้อ
สินค้าที่ต้องผ่านหลายขั้นตอน: **รอดำเนินการ (Pending) → จัดส่งแล้ว (Shipped) → ส่งถึงแล้ว (Delivered)** หรือถูก
**ยกเลิก (Cancelled)** ได้ในบางช่วง — นี่คือตัวอย่างคลาสสิกของ **state machine** (เครื่องจักรสถานะ) ที่พบในระบบ
จริงทุกประเภท ตั้งแต่ระบบ e-commerce, ระบบจองตั๋ว, ไปจนถึง workflow อนุมัติเอกสาร

ถ้าออกแบบด้วย struct + field แยก (แบบเดียวกับปัญหาใน 10.1) เราอาจเขียนแบบนี้:

```
struct OrderBadDesign {
    is_pending: bool,
    is_shipped: bool,
    is_delivered: bool,
    is_cancelled: bool,
    tracking_number: String, // มีความหมายแค่ตอน is_shipped
    delivered_date: String,  // มีความหมายแค่ตอน is_delivered
    cancel_reason: String,   // มีความหมายแค่ตอน is_cancelled
}
```

(โค้ดนี้เขียนแบบ pseudocode ไม่ต้อง compile ให้เห็นภาพเฉย ๆ) สังเกตปัญหาแบบเดียวกับหัวข้อ 10.1 ทวีคูณขึ้นไปอีก:
มี `bool` 4 ตัวที่ควรมีแค่ตัวเดียวเป็น `true`, และมี `String` อีก 3 ตัวที่มีความหมายเฉพาะบางสถานะเท่านั้น (ตอน
`is_pending == true` ทั้ง `tracking_number`, `delivered_date`, `cancel_reason` ล้วนไม่มีความหมาย แต่ในเชิง type
มันมีอยู่เสมอ ต้องใส่ค่า placeholder อย่าง `String::new()` ไปเปล่า ๆ) — นี่คือ struct ที่ "โกหก" เกี่ยวกับข้อมูล
ของตัวเองอยู่ตลอดเวลา

มาดูวิธีแก้ด้วย enum ที่แต่ละ variant พกข้อมูลเฉพาะของตัวเอง (แบบเดียวกับที่เรียนในหัวข้อ 10.3):

```rust
#[derive(Debug)]
enum Order {
    Pending,
    Shipped { tracking_number: String },
    Delivered { date: String },
    Cancelled { reason: String },
}

impl Order {
    fn status_message(&self) -> String {
        match self {
            Order::Pending => "คำสั่งซื้อกำลังรอดำเนินการ".to_string(),
            Order::Shipped { tracking_number } => {
                format!("จัดส่งแล้ว หมายเลขพัสดุ: {tracking_number}")
            }
            Order::Delivered { date } => format!("จัดส่งสำเร็จเมื่อ {date}"),
            Order::Cancelled { reason } => format!("ถูกยกเลิก เนื่องจาก: {reason}"),
        }
    }

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

    fn cancel(self, reason: String) -> Result<Order, String> {
        match self {
            Order::Delivered { .. } => {
                Err("ไม่สามารถยกเลิกออเดอร์ที่จัดส่งสำเร็จแล้วได้".to_string())
            }
            _ => Ok(Order::Cancelled { reason }),
        }
    }
}

fn main() {
    let order = Order::Pending;
    println!("{}", order.status_message());

    let order = order.ship("TH1234567890".to_string()).expect("จัดส่งไม่สำเร็จ");
    println!("{}", order.status_message());

    let order = order
        .deliver("2026-09-26".to_string())
        .expect("ยืนยันจัดส่งไม่สำเร็จ");
    println!("{}", order.status_message());

    match order.cancel("ลูกค้าเปลี่ยนใจ".to_string()) {
        Ok(cancelled) => println!("{}", cancelled.status_message()),
        Err(e) => println!("ยกเลิกไม่ได้: {e}"),
    }
}
```

ผลลัพธ์:

```
คำสั่งซื้อกำลังรอดำเนินการ
จัดส่งแล้ว หมายเลขพัสดุ: TH1234567890
จัดส่งสำเร็จเมื่อ 2026-09-26
ยกเลิกไม่ได้: ไม่สามารถยกเลิกออเดอร์ที่จัดส่งสำเร็จแล้วได้
```

**อธิบายการออกแบบทีละส่วน:**

- `#[derive(Debug)]` — เหมือนที่เรียนใน Part 9 การ derive `Debug` ให้ enum ทำให้ปริ้นด้วย `{:?}` ได้ (ใช้ใน
  `format!("... {other:?} ...")` เพื่อโชว์ว่าสถานะปัจจุบันคือ variant ไหนตอนเกิด error)
- แต่ละ variant พกแค่ข้อมูลที่**มีความหมายจริง**สำหรับสถานะนั้นเท่านั้น: `Pending` ไม่มีข้อมูลเพิ่มเลย (ยังไม่มี
  อะไรให้พก), `Shipped` มีแค่ `tracking_number`, `Delivered` มีแค่ `date`, `Cancelled` มีแค่ `reason` — ไม่มี field
  ไหน "อยู่เฉย ๆ ไม่มีความหมาย" แบบตัวอย่าง struct ที่แย่ข้างบนเลย
- `fn ship(self, tracking_number: String) -> Result<Order, String>` — สังเกตว่า method นี้รับ `self` **แบบไม่ใช่
  reference** (ยึด ownership ของ `Order` เดิมไปเลย ตามหลัก Part 6/7) แล้วคืนค่าเป็น `Order` ตัวใหม่ที่เปลี่ยนสถานะ
  แล้ว (หรือ `Err` ถ้าเปลี่ยนสถานะไม่ได้) การออกแบบแบบนี้ (เรียกว่า **consuming self** — กินค่าตัวเองไปสร้างค่า
  ใหม่) สื่อความหมายตรงตามธรรมชาติของ state machine พอดี: **"เมื่อเปลี่ยนสถานะแล้ว ออเดอร์เก่าไม่มีอยู่อีกต่อไป
  มีแต่ออเดอร์สถานะใหม่"** — คุณไม่มีทางถือ `Order::Pending` ตัวเดิมไว้ใช้ต่อพร้อมกับ `Order::Shipped` ตัวใหม่
  ได้เลย เพราะ ownership ถูกโอนไปเรียบร้อยแล้ว (ถ้าพยายามใช้ `order` ตัวเดิมหลังเรียก `.ship(...)` compiler จะฟ้อง
  error เรื่อง "use of moved value" ทันที ตามหลักที่เรียนใน Part 6) — นี่คือ ownership ที่ทำงานร่วมกับ enum
  เพื่อบังคับกฎของ state machine ให้แน่นขึ้นไปอีกชั้น ไม่ใช่แค่ enum อย่างเดียว
- `Order::Pending => Ok(Order::Shipped { tracking_number })` — การเปลี่ยนสถานะที่ถูกกฎ (จาก `Pending` ไป `Shipped`
  ได้) จะคืน `Ok` พร้อมค่าสถานะใหม่
- `other => Err(format!("ไม่สามารถจัดส่งออเดอร์ที่อยู่ในสถานะ {other:?} ได้"))` — การเปลี่ยนสถานะที่ผิดกฎ (เช่น
  จะ `ship()` ออเดอร์ที่ `Delivered` ไปแล้ว หรือที่ `Cancelled` ไปแล้ว) จะคืน `Err` พร้อมข้อความอธิบาย — สังเกตว่า
  เราไม่ต้องเขียนเงื่อนไขแยกทีละ variant ที่ผิดกฎ (`Shipped`, `Delivered`, `Cancelled` ทั้งสามล้วนผิดกฎสำหรับ
  `ship()`) เพราะ catch-all `other` จับทุกกรณีที่ไม่ใช่ `Pending` ไว้ในที่เดียว
- `Result<Order, String>` — `Result<T, E>` คล้ายกับ `Option<T>` ที่เรียนไปในหัวข้อ 10.9 มาก เพียงแต่กรณี "ไม่สำเร็จ"
  (`Err`) เก็บ**เหตุผลของความล้มเหลว**ไว้ด้วย (ในที่นี้คือ `String` อธิบายว่าทำไม transition ไม่สำเร็จ) ต่างจาก
  `Option::None` ที่บอกแค่ว่า "ไม่มีค่า" โดยไม่มีรายละเอียดเพิ่ม — เราจะเรียน `Result<T, E>` แบบเต็มรูปแบบใน
  **Part 12** รวมถึง `?` operator ที่ทำให้ chain การเรียกแบบนี้กระชับขึ้นไปอีกมาก ตอนนี้จำแค่ว่ามันเป็น enum
  ที่มี 2 variant คือ `Ok(T)` และ `Err(E)` ก็เพียงพอสำหรับอ่านโค้ดตัวอย่างนี้ให้เข้าใจ

**เชื่อมโยงกลับไปที่ปัญหาเริ่มต้นของบทนี้:** จำ struct ที่มี `bool` สามตัวใน 10.1 ได้ไหม ที่สามารถสร้างสถานะ
"is_cash: true, is_credit_card: true" พร้อมกันได้อย่างไม่มีเหตุผล — ลองถามคำถามเดียวกันกับ `Order` enum ที่เรา
ออกแบบตรงนี้: **"เป็นไปได้ไหมที่จะสร้างค่า `Order` ที่เป็นทั้ง `Shipped` และ `Cancelled` พร้อมกัน?"** คำตอบคือ
**เป็นไปไม่ได้เลยในเชิง type** เพราะ `Order` คือ enum — มันเป็นได้แค่หนึ่งใน 4 variant เท่านั้นในเวลาหนึ่ง ๆ
ไม่มีทาง "ผสม" สองสถานะเข้าด้วยกันได้ ต่างจาก struct ที่มี field แยกกันเป็นอิสระซึ่งผสมกันได้อย่างอิสระ (และผิดกฎ)
และยิ่งไปกว่านั้น ฟังก์ชัน `ship`/`deliver`/`cancel` ที่เรารับ `self` แบบยึด ownership ยัง **บังคับกฎการเปลี่ยน
สถานะ (transition rule)** เพิ่มเข้ามาอีกชั้น (เช่น ห้ามยกเลิกออเดอร์ที่ส่งถึงแล้ว) ทำให้ทั้งระบบ "เขียนโค้ดผิดกฎ
ไม่ได้เลย" ตั้งแต่ตอน compile ไปจนถึงตอน runtime ที่ตรวจสอบ business rule ผ่านค่าที่คืนมา — นี่คือภาพรวมทั้งหมด
ของแนวคิด **enum + pattern matching ทำให้ illegal states unrepresentable** ที่เราตั้งเป้าไว้ตั้งแต่ต้นบท

## กับดักที่พบบ่อย (Common Pitfalls)

**1. Non-exhaustive match — ลืม handle variant ที่เพิ่มมาใหม่ (E0004)**

นี่คือ error ที่คุณจะเจอบ่อยที่สุดเมื่อทำงานกับ enum ในระยะยาว โดยเฉพาะตอนเพิ่ม variant ใหม่ให้ enum ที่มีอยู่แล้ว
(ตามที่อธิบายละเอียดในหัวข้อ 10.5):

```rust
enum PaymentMethod {
    Cash,
    CreditCard,
    BankTransfer,
}

fn describe(method: PaymentMethod) -> &'static str {
    match method {
        PaymentMethod::Cash => "เงินสด",
        PaymentMethod::CreditCard => "บัตรเครดิต",
        // ลืม BankTransfer!
    }
}
```

```
error[E0004]: non-exhaustive patterns: `PaymentMethod::BankTransfer` not covered
```

**วิธีแก้**: เพิ่ม arm ให้ครบทุก variant หรือถ้าตั้งใจจะ "ไม่สนใจ variant ที่เหลือจริง ๆ" ให้เพิ่ม `_ => ...`
ปิดท้าย — แต่ให้ระวัง: **การใส่ `_` แบบไม่คิดให้ดีคือการปิดกั้นข้อดีที่สำคัญที่สุดของ exhaustiveness checking**
เพราะถ้าเพิ่ม variant ใหม่ในอนาคต `_` จะดักจับมันไปแบบเงียบ ๆ โดยไม่มี error เตือนให้แก้ ทำให้กลับไปมีปัญหาแบบ
`switch` ของภาษาอื่นเหมือนเดิม ควรใช้ `_` เฉพาะตอนที่แน่ใจจริง ๆ ว่า "ไม่สนใจ variant ใหม่ที่จะมาในอนาคตแน่นอน"
เช่น log ทั่วไปที่ handle เฉพาะกรณีสำคัญ ส่วนกรณีที่ต้อง handle ครบทุก variant เสมอ (เช่นคำนวณค่าธรรมเนียม) ควรเขียน
ให้ครบทุก arm โดยไม่ใช้ `_` เพื่อให้ compiler เตือนเมื่อมี variant ใหม่

**2. Match arms คืนค่า type ไม่ตรงกัน (E0308)**

ทุก arm ของ `match` ที่ใช้เป็น expression ต้องคืนค่าที่มี **type เดียวกันทั้งหมด**:

```rust
fn describe(n: i32) -> String {
    let result = match n {
        0 => "zero",   // &str
        _ => 1,        // i32 -- ผิด! ต้องเป็น &str เหมือนกัน
    };
    result.to_string()
}
```

```
error[E0308]: `match` arms have incompatible types
 --> src/main.rs:4:14
  |
2 |       let result = match n {
  |  __________________-
3 | |         0 => "zero",
  | |              ------ this is found to be of type `&str`
4 | |         _ => 1,
  | |              ^ expected `&str`, found integer
5 | |     };
  | |_____- `match` arms have incompatible types
```

**วิธีแก้**: ทำให้ทุก arm คืนค่า type เดียวกัน เช่นแปลงทุก arm ให้เป็น `String` ด้วย `.to_string()` ให้ตรงกันหมด
(`0 => "zero".to_string(), _ => 1.to_string()`) — error นี้เกิดเพราะ Rust ต้องรู้ type ของ `match` expression
ตัวเดียวที่แน่นอนตั้งแต่ compile time (ไม่ต่างจาก `if/else` ที่ทั้งสองแขนงต้องคืน type เดียวกันตามที่เรียนใน Part 4)

**3. เทียบ enum ด้วย `==` โดยไม่ derive `PartialEq` (E0369)**

Enum ที่ประกาศธรรมดาไม่มี operator `==`/`!=` ให้ใช้งานเองโดยอัตโนมัติ (เหมือนกับ struct ใน Part 9):

```rust
enum PaymentMethod {
    Cash,
    CreditCard,
    BankTransfer,
}

fn main() {
    let a = PaymentMethod::Cash;
    let b = PaymentMethod::Cash;
    if a == b {
        println!("เท่ากัน");
    }
}
```

```
error[E0369]: binary operation `==` cannot be applied to type `PaymentMethod`
  --> src/main.rs:10:10
   |
10 |     if a == b {
   |        - ^^ - PaymentMethod
   |        |
   |        PaymentMethod
   |
note: an implementation of `PartialEq` might be missing for `PaymentMethod`
help: consider annotating `PaymentMethod` with `#[derive(PartialEq)]`
   |
 1 + #[derive(PartialEq)]
 2 | enum PaymentMethod {
   |
```

**วิธีแก้**: เพิ่ม `#[derive(PartialEq)]` เหนือ enum (ต้องการเทียบด้วยตัวเลข/ลำดับด้วยก็เพิ่ม `PartialOrd` ได้ แต่
enum ที่มีข้อมูลติดตัว ทุก field ของทุก variant ต้อง implement `PartialEq` ด้วยเช่นกัน — เช่นถ้ามี field เป็น
`String` ก็ไม่มีปัญหาเพราะ `String` implement `PartialEq` อยู่แล้ว):

```rust
#[derive(PartialEq)]
enum PaymentMethod {
    Cash,
    CreditCard,
    BankTransfer,
}
```

**4. ใช้ syntax tuple-like กับ struct-like variant ผิดรูปแบบ (E0164)**

ถ้า variant ประกาศแบบ struct-like (มี field ตั้งชื่อด้วย `{ }`) แต่พยายาม match ด้วย syntax แบบ tuple-like
(`( )`) จะ compile ไม่ผ่าน:

```rust
enum PaymentMethod {
    Cash,
    CreditCard { number: String, cvv: String },
}

fn describe(m: &PaymentMethod) -> String {
    match m {
        PaymentMethod::Cash => "cash".to_string(),
        PaymentMethod::CreditCard(number, cvv) => format!("{number} {cvv}"), // ผิดรูปแบบ!
    }
}
```

```
error[E0164]: expected tuple struct or tuple variant, found struct variant `PaymentMethod::CreditCard`
 --> src/main.rs:9:9
  |
9 |         PaymentMethod::CreditCard(number, cvv) => format!("{number} {cvv}"),
  |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ not a tuple struct or tuple variant
  |
help: the struct variant's fields are being ignored
  |
9 -         PaymentMethod::CreditCard(number, cvv) => format!("{number} {cvv}"),
9 +         PaymentMethod::CreditCard { number: _, cvv: _ } => format!("{number} {cvv}"),
  |
```

**วิธีแก้**: ใช้ `{ number, cvv }` (struct-like destructure) ให้ตรงกับรูปแบบที่ประกาศ ข้อผิดพลาดนี้พบบ่อยเมื่อ
สลับไปมาระหว่าง tuple-like variant กับ struct-like variant ในโปรเจกต์เดียวกัน — ให้จำง่าย ๆ ว่า **การ match ต้อง
ใช้ syntax เดียวกับตอนประกาศ variant นั้นเสมอ**: ประกาศด้วย `( )` ก็ match ด้วย `( )`, ประกาศด้วย `{ }` ก็ match
ด้วย `{ }`

**5. พยายาม move ค่าออกจาก reference ตอน match (match ergonomics + E0507)**

เมื่อ `match` บนค่าที่เป็น reference (`&Enum`) เช่นตอนวนลูปด้วย `for x in &vec`, Rust จะใช้กลไกที่เรียกว่า
**match ergonomics**: ผูก field ภายใน pattern ให้เป็น reference โดยอัตโนมัติ (ไม่ต้องเขียน `&` ในทุก pattern เอง)
แต่นั่นแปลว่า **field ที่ผูกมาได้เป็น reference ไม่ใช่ค่าตัวจริง** — ถ้าพยายาม "ยึด ownership" ออกจาก reference
นั้นตรง ๆ จะเจอ error ที่เกี่ยวกับ ownership โดยตรง (ตามหลัก Part 6-7):

```rust
enum PaymentMethod {
    Cash,
    BankTransfer(String),
}

fn main() {
    let methods = vec![PaymentMethod::BankTransfer("123-456".to_string())];
    for m in &methods {
        match m {
            PaymentMethod::BankTransfer(account) => {
                let owned: String = *account; // account คือ &String (จาก match ergonomics)
                println!("{owned}");
            }
            _ => {}
        }
    }
}
```

```
error[E0507]: cannot move out of `*account` which is behind a shared reference
  --> src/main.rs:11:37
   |
11 |                 let owned: String = *account;
   |                                     ^^^^^^^^ move occurs because `*account` has type `String`, which does not implement the `Copy` trait
   |
help: consider cloning the value if the performance cost is acceptable
   |
11 -                 let owned: String = *account;
11 +                 let owned: String = account.clone();
   |
```

ที่เกิด error เพราะ `m` มี type `&PaymentMethod` (ไม่ใช่ `PaymentMethod`) เนื่องจากเราวนลูปด้วย `&methods` — match
ergonomics ทำให้ `account` ในแขนง `PaymentMethod::BankTransfer(account)` มี type เป็น `&String` โดยอัตโนมัติ
(ไม่ต้องเขียน `&account` ใน pattern เอง) การเขียน `*account` คือการ **dereference** เพื่อพยายามเอาค่า `String`
ข้างในออกมา ซึ่งจะเป็นการ **move ค่าออกจาก reference** ที่เราไม่ได้เป็นเจ้าของ (เรามีแค่ "สิทธิ์ยืมชั่วคราว" ผ่าน
`&methods` เท่านั้น ตามหลักการยืมใน Part 7) Rust จึงปฏิเสธ

**วิธีแก้**: ใช้ `.clone()` แทน (ถ้ายอมรับต้นทุนการ clone ได้ ตามที่ error message แนะนำ) หรือถ้าไม่ต้องการ clone
เลย ให้ทำงานกับ `&String` ตรง ๆ โดยไม่ต้อง dereference/move มันออกมา (เช่น `println!("{account}")` โดยไม่ต้องสร้าง
ตัวแปร `owned` เลย เพราะ `println!` แค่ยืมมาอ่านก็เพียงพอ) — นี่คือจุดที่แสดงให้เห็นว่าความรู้เรื่อง ownership/
borrowing จาก Part 6-7 กับ pattern matching ในบทนี้ **ทำงานประสานกันอยู่เสมอ** ไม่ใช่คนละเรื่องที่แยกจากกัน

**6. ลำดับ pattern ผิด — catch-all มาก่อน pattern เฉพาะเจาะจง**

`match` เทียบจากบนลงล่าง ถ้า `_` หรือ pattern กว้าง ๆ อยู่ก่อน pattern เฉพาะที่ควรถูกเช็คก่อน pattern เฉพาะนั้นจะ
ไม่มีวันถูกใช้เลย (unreachable):

```rust
fn classify(n: i32) -> &'static str {
    match n {
        _ => "อื่น ๆ",
        0 => "ศูนย์", // ไม่มีวันถูกเรียกใช้ เพราะ _ ข้างบนจับไปหมดแล้ว
    }
}
```

```
warning: unreachable pattern
 --> src/main.rs:4:9
  |
3 |         _ => "อื่น ๆ",
  |         - matches any value
4 |         0 => "ศูนย์",
  |         ^ no value can reach this
  |
  = note: `#[warn(unreachable_patterns)]` (part of `#[warn(unused)]`) on by default
```

นี่เป็นแค่ **warning ไม่ใช่ error** (โค้ด compile ผ่านและรันได้ แต่ arm `0 => "ศูนย์"` จะไม่ทำงานตามที่ตั้งใจไว้เลย
— ผลลัพธ์จะเป็น `"อื่น ๆ"` เสมอไม่ว่า `n` จะเป็นอะไร) **วิธีแก้**: จัดลำดับ pattern จากเฉพาะเจาะจงที่สุดไปกว้างที่สุด
เสมอ — วาง `0 => "ศูนย์"` ไว้ก่อน `_ => "อื่น ๆ"` เพราะกฎทั่วไปของ `match` คือ **ต้องเขียน pattern แคบไปกว้าง**
ไม่ใช่ตรงกันข้าม ให้สังเกต warning นี้อย่างจริงจังเสมอเมื่อ `cargo build`/`cargo check` แสดงขึ้นมา เพราะมันมักบ่งบอก
ถึง logic bug ที่ตั้งใจเขียนผิดจากที่คิดไว้จริง ๆ

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** เขียน enum ชื่อ `TrafficLight` ที่มี 3 variant คือ `Red`, `Yellow`, `Green` (ไม่มีข้อมูลติดตัว)
   จากนั้นเขียนฟังก์ชัน `fn instruction(light: &TrafficLight) -> &'static str` ที่ใช้ `match` คืนข้อความ
   "หยุดรถ" สำหรับ `Red`, "เตรียมตัว" สำหรับ `Yellow`, และ "ไปได้" สำหรับ `Green` — ลองลบ arm หนึ่งออกดูว่า
   compiler ฟ้อง error อะไร (ควรเจอ `E0004`) แล้วเพิ่มกลับเข้าไปให้ compile ผ่าน

2. **[กลาง]** ขยาย enum จากข้อ 1 ให้เป็น `TrafficLightWithTimer` ที่แต่ละสีมี field เก็บเวลาที่เหลือเป็นวินาที
   เช่น `Red(u32)`, `Yellow(u32)`, `Green(u32)` เขียนฟังก์ชันที่ใช้ `match` พร้อม pattern แบบ range หรือ match
   guard เพื่อบอกว่า "ใกล้เปลี่ยนสีแล้ว" ถ้าเวลาที่เหลือ ≤ 3 วินาที เช่น `Green(t) if t <= 3 => "สีเขียวใกล้หมดแล้ว
   เตรียมชะลอ"` (hint: อย่าลืมว่า pattern ทั่วไปของ variant เดียวกันที่ไม่มี guard ต้องอยู่หลัง pattern ที่มี guard
   ไม่เช่นนั้นจะไม่ถูกเรียกใช้)

3. **[ยาก]** สร้าง enum ชื่อ `Shape` ที่มี variant `Circle { radius: f64 }`, `Rectangle { width: f64, height: f64 }`,
   และ `Triangle { base: f64, height: f64 }` เขียน `impl` block ที่มี method `fn area(&self) -> f64` คำนวณพื้นที่
   ของแต่ละรูปทรงด้วย `match self` (สูตร: circle = π×r², rectangle = width×height, triangle = 0.5×base×height —
   ใช้ `std::f64::consts::PI` สำหรับค่า π) จากนั้นเขียนฟังก์ชัน `fn total_area(shapes: &[Shape]) -> f64` ที่รับ
   slice ของ `Shape` (ทวนความรู้จาก Part 8) แล้วรวมพื้นที่ทั้งหมดด้วย loop ที่เรียก `.area()` ทีละตัว ทดสอบด้วย
   `Vec<Shape>` ที่มีรูปทรงผสมกันหลายแบบ

4. **[ยาก/ประยุกต์ใช้งานจริง]** ออกแบบ enum ชื่อ `TicketStatus` จำลองระบบสถานะตั๋ว support ของทีมไอที ที่มี
   4 สถานะ: `Open`, `InProgress { assignee: String }`, `Resolved { solution: String }`, `Closed { closed_by: String }`
   เขียน method `transition` ที่กำหนดกฎการเปลี่ยนสถานะดังนี้ (คล้ายตัวอย่าง `Order` ในบทนี้ แต่ออกแบบกฎเอง):
   `Open` เปลี่ยนเป็น `InProgress` ได้อย่างเดียว, `InProgress` เปลี่ยนเป็น `Resolved` ได้อย่างเดียว, `Resolved`
   เปลี่ยนเป็น `Closed` ได้อย่างเดียว, ส่วน `Closed` **เปลี่ยนสถานะต่อไปอีกไม่ได้เลย** (ต้อง return `Err` เสมอ)
   ให้ method คืนค่าเป็น `Result<TicketStatus, String>` เหมือนตัวอย่าง `Order.ship()`/`Order.deliver()` ในบทนี้
   แล้วทดลองเขียนโค้ดที่พยายามเปลี่ยนสถานะผิดกฎ (เช่นพยายามเปิด ticket ที่ `Closed` แล้วกลับไป `InProgress`)
   เพื่อดูว่า business rule ทำงานถูกต้องผ่านค่า `Err` ที่ได้กลับมา (hint: การออกแบบนี้ไม่ได้ป้องกัน illegal
   transition ในระดับ type เหมือนที่ enum ป้องกัน illegal state ได้ — มันเป็นการตรวจสอบ business rule ตอน runtime
   ผ่าน `Result` ลองคิดต่อว่าทำไมกฎการเปลี่ยนสถานะ [transition rule] ถึงมักตรวจสอบตอน runtime ในขณะที่โครงสร้าง
   ของสถานะเอง [state shape] ตรวจสอบได้ตั้งแต่ compile time ด้วย type system)

## สรุป

ในบทนี้เราเริ่มจากปัญหาจริง: การจำลอง "หนึ่งในหลายตัวเลือกที่แน่นอน" ด้วย struct + boolean flags หรือ string tag
เปิดช่องให้เกิด "สถานะที่ไม่ควรมีอยู่จริง" ได้ (สอง flag เป็น `true` พร้อมกัน, string พิมพ์ผิดที่ compiler จับไม่ได้)
แล้วแนะนำ **enum** ในฐานะเครื่องมือที่ compiler ใช้บังคับว่า "ค่าต้องเป็นแค่หนึ่งใน variant ที่กำหนดไว้เท่านั้น"
เราเห็นว่าแต่ละ variant ของ enum พกข้อมูลรูปร่างต่างกันได้อย่างอิสระ (unit, tuple-like, struct-like ผสมกันในตัว
เดียวได้) ซึ่งเป็นจุดเด่นที่เหนือกว่า enum ของภาษาอื่นอย่างชัดเจน จากนั้นเจาะลึก `match` แบบเต็มรูปแบบ ตั้งแต่
**exhaustiveness checking** (ที่ทำให้การเพิ่ม variant ใหม่ในระบบใหญ่ปลอดภัย 100% เพราะ compiler ไล่บอกทุกจุดที่
ต้องแก้), `_`/catch-all binding, ไปจนถึง pattern หลากหลายรูปแบบ (destructure, range, or-pattern, literal, tuple)
และ match guard เราแนะนำ `Option<T>` ในฐานะ enum ที่แก้ปัญหา "billion dollar mistake" ของ null โดยบังคับให้ต้อง
`match`/`if let` ก่อนใช้ค่าเสมอ (รายละเอียดเต็มรออยู่ใน Part 11) รวมถึง `if let`, `while let`, และ `let else` ที่เป็น
sugar ทำให้จัดการ pattern เดียวได้กระชับขึ้น ปิดท้ายด้วยตัวอย่าง state machine ของออเดอร์สั่งซื้อสินค้าที่แสดงให้
เห็นภาพรวมทั้งหมด: **enum + pattern matching + ownership ทำให้ "สถานะที่ผิดกฎ" กลายเป็นสิ่งที่เขียนไม่ได้เลยในเชิง
type** ซึ่งเชื่อมโยงกลับไปยังปัญหาตั้งต้นของบทนี้อย่างสมบูรณ์

ใน **Part 11** เราจะเจาะลึก `Option<T>` แบบเต็มรูปแบบต่อจากที่แนะนำไว้ในหัวข้อ 10.9 — method ที่สำคัญอย่าง
`.unwrap()`, `.unwrap_or()`, `.unwrap_or_default()`, `.map()`, `.and_then()`, `.filter()`, การ chain method
เหล่านี้เพื่อเขียนโค้ดที่ปลอดภัยจาก null โดยไม่ต้อง `match` ยาว ๆ ทุกครั้ง และรูปแบบการออกแบบ API ที่ใช้
`Option<T>` อย่าง idiomatic ในโค้ด Rust จริง

---

**Part ก่อนหน้า:** [Structs](part-009-structs.md) | **Part ถัดไป:** [Option<T> และ Null Safety](part-011-option-and-null-safety.md)
