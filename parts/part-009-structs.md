# Part 9: Structs (การสร้างชนิดข้อมูลของตัวเอง)

> โมดูล: พื้นฐานภาษา Rust (Core Language Fundamentals) | ระดับ: พื้นฐาน | เวลาโดยประมาณ: 150 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้ว่าทำไม tuple ถึงไม่เพียงพอสำหรับข้อมูลที่มีโครงสร้างซับซ้อนขึ้น และ struct แก้ปัญหานั้นอย่างไร
- นิยาม named-field struct, สร้าง instance, เข้าถึง field ด้วย `.`, และใช้ field init shorthand ได้อย่างถูกต้อง
- อธิบายได้ว่า struct "เป็นเจ้าของ" field ของมันเองอย่างไร และเข้าใจว่าทำไมการเก็บ reference ไว้ใน struct
  ต้องมี lifetime annotation (โดยไม่ต้องเจาะลึกทุกรายละเอียด — จะเรียนเต็มใน Part 20/23)
- อธิบายและหลีกเลี่ยงปัญหา **partial move** ที่เกิดจากการย้าย field ออกจาก struct โดยไม่ตั้งใจ
- ใช้ **struct update syntax** (`..`) เพื่อสร้าง instance ใหม่จาก instance เดิมได้อย่างกระชับ
- นิยามและใช้งาน **tuple struct** และ **unit-like struct** ได้ พร้อมรู้ว่าแต่ละแบบเหมาะกับสถานการณ์ไหน
- เขียน `impl` block พร้อม method ที่ใช้ `&self`, `&mut self`, และ `self` ได้อย่างถูกต้อง และอธิบายความหมาย
  เชิง ownership ของ receiver ทั้ง 3 แบบได้อย่างลึกซึ้ง
- เขียน **associated function** เป็น constructor ตาม convention `new(...)` ของ Rust
- ใช้ `#[derive(Debug)]` เพื่อ debug print struct ด้วย `{:?}`/`{:#?}` และรู้จัก `#[derive(Clone)]`/`#[derive(PartialEq)]` เบื้องต้น
- ออกแบบระบบข้อมูลจริงที่ประกอบด้วยหลาย struct ทำงานร่วมกัน (domain model) ในสไตล์ idiomatic Rust

## ความรู้ที่ต้องมีมาก่อน

- **Part 3**: ตัวแปร, `let`/`mut`, tuple, array, และการ destructure — บทนี้จะเริ่มจากปัญหาของ tuple โดยตรง
- **Part 6**: Ownership เบื้องต้น (move, copy, clone, drop) — struct ผูกกับแนวคิด ownership อย่างแน่นแฟ้น
  ทุก field ของ struct ก็มีเจ้าของเหมือนตัวแปรทั่วไป และกฎ move/copy ที่เรียนมาจะกลับมาใช้ซ้ำเต็ม ๆ ในบทนี้
- **Part 7**: Borrowing และ References (`&T`, `&mut T`, borrow checker rules) — จำเป็นมากสำหรับทำความเข้าใจ
  receiver แบบ `&self`/`&mut self` ของ method และการหลีกเลี่ยงปัญหา partial move ด้วยการยืม
- **Part 8**: Slices (`&str`, `&[T]`) — จำเป็นสำหรับเข้าใจว่าทำไมการเก็บ `&str` ไว้ใน struct ต้องมี lifetime annotation

ถ้ายังไม่มั่นใจเรื่อง move/borrow จาก Part 6-7 แนะนำให้ย้อนไปทวนก่อน เพราะบทนี้คือจุดที่แนวคิดเหล่านั้น "ประกอบกัน" เป็น
ชนิดข้อมูลที่ซับซ้อนขึ้นเป็นครั้งแรก — struct ไม่ได้มีกฎ ownership ของตัวเองแยกต่างหาก แต่ใช้กฎเดียวกับที่เรียนมาแล้วทั้งหมด
เพียงแค่ตอนนี้ "ค่าหนึ่งค่า" อาจประกอบด้วยหลาย field ที่แต่ละ field ก็มีเจ้าของและกฎ move/borrow เป็นของตัวเองด้วย

## เนื้อหา

### 9.1 ปัญหาของการใช้ Tuple แทนข้อมูลที่มีความหมาย

จาก Part 3 เราได้เรียนรู้จักการใช้ tuple เก็บค่าหลาย ๆ ค่าที่มีชนิดต่างกันไว้ในตัวแปรเดียว เช่น การเก็บข้อมูลสินค้าหนึ่งตัว
เป็น `(String, f64, u32)` — ชื่อสินค้า, ราคา, จำนวน ลองมาดูว่าเมื่อโปรแกรมเริ่มโตขึ้น การใช้ tuple แบบนี้ต่อไปจะเริ่มมีปัญหา
อะไรบ้าง:

```rust
fn print_receipt_line(product: (String, f64, u32)) {
    // product.0 = ชื่อ, product.1 = ราคา, product.2 = จำนวน — ใช่ไหม?
    // ต้องเปิดไปดูจุดที่สร้าง tuple ทุกครั้งเพื่อยืนยันลำดับ ไม่มีอะไรบอกตรง ๆ ในซิกเนเจอร์นี้เลย
    println!(
        "{} x{} = {:.2} บาท",
        product.0,
        product.2,
        product.1 * product.2 as f64
    );
}

fn main() {
    let item_a: (String, f64, u32) = (String::from("เมาส์ไร้สาย"), 299.0, 3);
    print_receipt_line(item_a);

    // จุดอ่อนที่แท้จริง: ไม่มีอะไรห้ามสลับลำดับตอนสร้าง tuple แบบนี้
    // สมมติมีคนเขียนโค้ดใหม่ตรงนี้แล้วจำลำดับผิด (เอาจำนวนไปไว้ก่อนราคา)
    let item_b: (String, f64, u32) = (String::from("คีย์บอร์ด"), 2.0, 890); // ตั้งใจ "สลับผิด"
    print_receipt_line(item_b); // compile ผ่านสบาย ๆ เพราะ type ตรงกันทุกตำแหน่ง (String, f64, u32)
                                 // แต่ความหมายทางธุรกิจพังไปแล้ว: ราคากลายเป็น 2.0 บาท จำนวนกลายเป็น 890 ชิ้น
}
```

ผลลัพธ์:

```
เมาส์ไร้สาย x3 = 897.00 บาท
คีย์บอร์ด x890 = 1780.00 บาท
```

สังเกตว่า **โค้ดนี้ compile ผ่านโดยไม่มี warning หรือ error ใด ๆ เลย** ทั้ง ๆ ที่ `item_b` ใส่ค่าผิดตำแหน่งไปโดยสิ้นเชิง
(ตั้งใจใส่ราคา 890 บาท จำนวน 2 ชิ้น แต่เขียนสลับเป็นราคา 2.0 บาท จำนวน 890 ชิ้น) — เพราะในสายตาของ type checker
`(String, f64, u32)` ก็คือ `(String, f64, u32)` ไม่ว่าค่าไหนจะอยู่ตำแหน่งไหน type ยังตรงกันทุกตำแหน่งเป๊ะ ๆ **compiler
ไม่มีทางรู้เลยว่าตำแหน่งที่ 1 "ต้องแปลว่า" ราคา และตำแหน่งที่ 2 "ต้องแปลว่า" จำนวน** เพราะ tuple ไม่มีแนวคิดเรื่อง
"ความหมาย" ของแต่ละตำแหน่งอยู่ในระดับ type เลย มันมีแค่ "ลำดับ" กับ "ชนิดข้อมูล" เท่านั้น

ปัญหานี้จะยิ่งรุนแรงขึ้นเมื่อ:

1. **จำนวน field เพิ่มขึ้น** — ลองนึกภาพ tuple ที่มี 6-7 ตำแหน่ง `(String, f64, u32, bool, String, u32)` ใครจะจำได้ว่า
   ตำแหน่งที่ 5 คืออะไร โดยไม่ต้องเปิดโค้ดจุดที่สร้าง tuple ไปดู
2. **ฟังก์ชันหลายตัวรับ tuple รูปแบบเดียวกัน** — ถ้าแก้ไขความหมายของตำแหน่งใดตำแหน่งหนึ่ง (เช่นเปลี่ยนจาก "ราคาต่อหน่วย"
   เป็น "ราคารวม") ต้องไล่แก้ทุกจุดที่ใช้ tuple นี้ด้วยตัวเอง ไม่มี compiler ช่วยเตือนว่าจุดไหนลืมแก้ความหมาย
3. **ไม่มีชื่อผูกกับ field** — เวลาอ่านโค้ดคนอื่น (หรือโค้ดตัวเองในอีก 6 เดือน) `product.1` ไม่สื่อความหมายอะไรเลยในตัวมันเอง
   ต่างจาก `product.price` ที่อ่านแล้วเข้าใจทันที

สิ่งที่เราต้องการจริง ๆ คือชนิดข้อมูลที่ **ผูกชื่อความหมายเข้ากับแต่ละค่าโดยตรงในระดับ type** เพื่อให้ compiler และคนอ่านโค้ด
รู้ทันทีว่าค่านี้ "คือ" อะไร ไม่ใช่แค่ "อยู่ตำแหน่งที่เท่าไหร่" — นี่คือสิ่งที่ **struct** มีไว้ให้โดยเฉพาะ

### 9.2 นิยาม Struct แบบมีชื่อ Field (Named-Field / Classic Struct)

เราใช้ keyword `struct` เพื่อนิยามชนิดข้อมูลใหม่ที่มี field พร้อมชื่อกำกับความหมายชัดเจน:

```rust
struct Product {
    name: String,
    price: f64,
    quantity: u32,
}

fn main() {} // ยังไม่สร้าง instance ในตัวอย่างนี้ แค่แสดงไวยากรณ์การนิยาม struct เฉย ๆ
```

รูปแบบไวยากรณ์: `struct ชื่อ_Type { ชื่อ_field: ชนิดข้อมูล, ... }` — สังเกตว่าชื่อ struct (`Product`) เขียนแบบ
**PascalCase** (ตัวใหญ่ตัวแรกของทุกคำ) ตาม convention ของ Rust สำหรับชื่อ type ทุกชนิด (ต่างจากตัวแปรและ field ที่ใช้
snake_case) — นี่ไม่ใช่แค่ความชอบส่วนตัวแต่เป็น convention ที่ clippy บังคับเช่นเดียวกับ `SCREAMING_SNAKE_CASE` ของ
`const` ที่เราเห็นมาแล้วใน Part 3

การสร้าง instance (คำที่ใช้เรียก "ตัวแทนค่าจริง" ของ struct หนึ่งตัว) ทำได้ด้วย **struct literal syntax** — ระบุชื่อ
field ตามด้วย `:` และค่า คั่นด้วย comma:

```rust
struct Product {
    name: String,
    price: f64,
    quantity: u32,
}

fn print_receipt_line(product: &Product) {
    // เข้าถึงแต่ละ field ด้วยชื่อที่สื่อความหมาย ไม่ต้องจำลำดับเลย
    println!(
        "{} x{} = {:.2} บาท",
        product.name,
        product.quantity,
        product.price * product.quantity as f64
    );
}

fn main() {
    let item_a = Product {
        name: String::from("เมาส์ไร้สาย"),
        price: 299.0,
        quantity: 3,
    };
    print_receipt_line(&item_a);

    // ลองสร้างแบบ "สลับลำดับ field ตอนเขียน" ดูบ้าง — เพราะ struct literal ระบุชื่อ field เสมอ
    // ลำดับที่เขียนในโค้ดจะไม่มีผลต่อความหมายอีกต่อไป
    let item_b = Product {
        quantity: 8,
        name: String::from("คีย์บอร์ดเมคานิคอล"),
        price: 890.0,
    };
    print_receipt_line(&item_b);
}
```

ผลลัพธ์:

```
เมาส์ไร้สาย x3 = 897.00 บาท
คีย์บอร์ดเมคานิคอล x8 = 7120.00 บาท
```

สังเกตจุดสำคัญ 2 อย่างที่ต่างจาก tuple อย่างสิ้นเชิง:

1. **`product.name` สื่อความหมายในตัวเองทันที** ไม่ต้องเดาว่าตำแหน่งไหนคือชื่อ ตำแหน่งไหนคือราคา
2. **ลำดับการเขียน field ใน struct literal ไม่มีผลต่อความหมายเลย** — `item_b` เขียน `quantity` มาก่อน `name` และ `price`
   แต่ compiler ยัง map ค่าตามชื่อ field ให้ถูกต้องเสมอ ไม่ใช่ตามลำดับที่เขียน สิ่งนี้แก้ปัญหาคลาสสิกของ tuple ในหัวข้อ
   ก่อนหน้าไปโดยสิ้นเชิง — ต่อให้เขียนสลับตำแหน่งยังไงก็ตาม ตราบใดที่ระบุชื่อ field ให้ตรง ความหมายก็ยังถูกต้องเสมอ

**ข้อจำกัดสำคัญที่ต้องรู้**: struct literal ต้อง **ระบุค่าให้ครบทุก field เสมอ** ไม่มี field ที่เป็น "optional" หรือมีค่า
default ในตัวมันเอง (ถ้าต้องการค่า default จะมี pattern แยกที่เกี่ยวข้องกับ trait `Default` ซึ่งจะพูดถึงใน Part หลัง ๆ)
ถ้าลืม field ใดไป compiler จะปฏิเสธทันที ซึ่งเราจะเห็น error จริงในหัวข้อกับดักท้ายบท

### 9.3 Field Init Shorthand: เมื่อชื่อตัวแปรตรงกับชื่อ Field

เวลาสร้างฟังก์ชันที่รับ parameter ชื่อตรงกับ field ของ struct เป๊ะ ๆ (สถานการณ์นี้เกิดขึ้นบ่อยมาก โดยเฉพาะใน constructor)
Rust มี syntax ย่อที่เรียกว่า **field init shorthand** ให้ไม่ต้องเขียนชื่อ field ซ้ำสองครั้ง:

```rust
struct Product {
    name: String,
    price: f64,
    quantity: u32,
}

// ฟังก์ชันที่รับ parameter ชื่อเดียวกับ field ของ Product เป๊ะ ๆ
fn new_product(name: String, price: f64, quantity: u32) -> Product {
    // field init shorthand: เพราะชื่อตัวแปร name/price/quantity ตรงกับชื่อ field เป๊ะ ๆ
    // ไม่ต้องเขียน "name: name, price: price, quantity: quantity" ให้ซ้ำซ้อน
    Product {
        name,
        price,
        quantity,
    }
}

fn main() {
    let item = new_product(String::from("แผ่นรองเมาส์"), 150.0, 40);
    println!(
        "{} ราคา {:.2} บาท จำนวน {} ชิ้น",
        item.name, item.price, item.quantity
    );
}
```

ผลลัพธ์:

```
แผ่นรองเมาส์ ราคา 150.00 บาท จำนวน 40 ชิ้น
```

`Product { name, price, quantity }` เทียบเท่ากับ `Product { name: name, price: price, quantity: quantity }` ทุกประการ
— compiler มองว่าเมื่อไม่มี `:` ตามหลังชื่อ field ให้เข้าใจว่า "หยิบค่าจากตัวแปรที่ชื่อตรงกันในขอบเขตปัจจุบันมาใส่" นี่คือ
syntax sugar (ไวยากรณ์ที่ย่อให้เขียนสั้นลง แต่ compiler แปลงกลับเป็นรูปแบบเต็มก่อนตรวจสอบ) เล็ก ๆ ที่ช่วยลดความซ้ำซ้อน
(DRY — Don't Repeat Yourself) ได้มาก โดยเฉพาะเมื่อ struct มี field จำนวนมาก — เราจะใช้ pattern นี้อย่างเป็นระบบใน
associated function `new(...)` ที่หัวข้อ 9.11

**ข้อสังเกต**: shorthand ใช้ได้เฉพาะเมื่อชื่อตัวแปร**ตรงกับชื่อ field เป๊ะ ๆ** (case-sensitive) ถ้าตัวแปรชื่อ `product_name`
แต่ field ชื่อ `name` จะใช้ shorthand ไม่ได้ ต้องเขียนแบบเต็ม `Product { name: product_name, ... }`

### 9.4 Struct เป็นเจ้าของ Field ของมันเอง (Ownership และ Struct)

จาก Part 6 เราเรียนรู้ว่าทุกค่าใน Rust มี "เจ้าของ" เพียงหนึ่งเดียว คำถามคือ: เมื่อ struct มี field เป็น `String`
(ชนิดข้อมูลที่เป็นเจ้าของ heap memory ของตัวเอง) ใครเป็นเจ้าของ `String` นั้นกันแน่?

คำตอบคือ **struct instance เป็นเจ้าของ field ของมันเองโดยตรง** — เมื่อคุณเขียน `Product { name: String::from("..."), ... }`
ความเป็นเจ้าของของ `String` ที่สร้างขึ้นจะถูกย้าย (moved) เข้าไปเป็นของ field `name` ของ `Product` instance นั้นทันที
ไม่ใช่การยืมหรือ reference ใด ๆ ทั้งสิ้น พูดอีกแบบคือ **struct คือ "กล่อง" ที่รวบรวมความเป็นเจ้าของของหลาย ๆ ค่าไว้ด้วยกัน
เป็นหนึ่งเดียว**

ผลที่ตามมาที่สำคัญที่สุดคือเรื่อง **การ drop**: เมื่อ `Product` instance หมดอายุ (ออกจาก scope) ตาม concept จาก Part 6
**ทุก field ของมันจะถูก drop ตามไปด้วยโดยอัตโนมัติ** ไม่ต้องเขียนโค้ดจัดการเองแม้แต่บรรทัดเดียว:

```rust
struct Product {
    name: String,
    price: f64,
    quantity: u32,
}

fn main() {
    {
        let item = Product {
            name: String::from("เมาส์ไร้สาย"),
            price: 299.0,
            quantity: 3,
        };
        println!("สร้าง item แล้ว: {}", item.name);
    } // item ออกจาก scope ที่นี่ — Rust จะ drop item ทั้งก้อน
      // ซึ่งหมายถึง drop field name (String) ด้วย เพราะ item เป็นเจ้าของ name โดยตรง
      // memory บน heap ที่ String::from("เมาส์ไร้สาย") จองไว้จะถูกคืนกลับให้ระบบทันที ไม่มี memory leak
    println!("จบ scope ของ item แล้ว");
}
```

นี่คือข้อดีสำคัญของการให้ struct "เป็นเจ้าของ" field ของมันเอง (แทนที่จะเก็บแค่ reference ไปยังข้อมูลที่อยู่ที่อื่น):
**อายุการใช้งาน (lifetime) ของทุก field ผูกติดกับอายุของ struct โดยอัตโนมัติ** ไม่ต้องคอยเช็คว่า field ไหนยังมีชีวิตอยู่
หรือถูกทำลายไปแล้วหรือยัง — ถ้า `Product` instance ยังอยู่ คุณมั่นใจได้ 100% ว่า `name`, `price`, `quantity` ของมันก็ยังอยู่
ครบถ้วนเสมอ นี่คือเหตุผลที่ struct ส่วนใหญ่ในโค้ด Rust มือใหม่ (และแม้แต่โค้ดโปรดักชันจริงจำนวนมาก) เลือกเก็บ `String`
แทน `&str` และเก็บ `Vec<T>` แทน `&[T]` — เพื่อให้ struct เป็นเจ้าของข้อมูลของมันเองเต็มที่ ไม่ต้องยุ่งกับเรื่อง lifetime
ที่ซับซ้อนขึ้น (ซึ่งเราจะพูดถึงในหัวข้อถัดไป)

### 9.5 ถ้าอยากเก็บ Reference ไว้ใน Struct ล่ะ? (เกริ่น Lifetime)

ในเมื่อ `&str` (string slice จาก Part 8) ก็เป็นชนิดข้อมูลที่ใช้แทนข้อความได้ คุณอาจสงสัยว่าทำไมไม่ใช้ `&str` เป็น type
ของ field `name` ไปเลย จะได้ไม่ต้องเสีย cost ในการจอง heap memory ใหม่ทุกครั้งด้วย `String::from(...)` ลองดูว่าเกิดอะไรขึ้น
ถ้าเขียนแบบนั้นตรง ๆ:

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

โค้ดนี้ **compile ไม่ผ่าน** ทันที:

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

**ทำไมถึง error?** ทวนจาก Part 7: reference (`&T`) ทุกตัวมี "อายุ" (lifetime) ที่ borrow checker ต้องตรวจสอบให้แน่ใจว่า
มันไม่ได้ชี้ไปยังข้อมูลที่ตายไปแล้ว (dangling reference) เมื่อ reference เป็นตัวแปร local ธรรมดา (`let r = &s;`) compiler
สามารถวิเคราะห์ scope ของฟังก์ชันได้เองว่า `r` มีอายุสั้นแค่ไหน แต่เมื่อ reference กลายเป็น **field ของ struct** เรื่องราว
ซับซ้อนขึ้นมาก — struct instance หนึ่งตัวอาจถูกส่งผ่านไปมาระหว่างหลายฟังก์ชัน เก็บไว้ในตัวแปรนาน ๆ หรือ return ออกจาก
ฟังก์ชัน **compiler จำเป็นต้องรู้ล่วงหน้าว่า "reference ที่ field นี้เก็บอยู่ ต้องมีอายุยืนยาวไม่น้อยกว่า struct instance
เองเสมอ"** ไม่อย่างนั้นก็อาจเกิด `Product` ที่ยังมีชีวิตอยู่ แต่ `name` ของมันชี้ไปยัง `&str` ที่ถูก drop ไปแล้วได้

วิธีแก้คือเพิ่ม **lifetime parameter** (เขียนแทนด้วยชื่อที่ขึ้นต้นด้วย apostrophe เช่น `'a`) ต่อท้ายชื่อ struct
ตาม help ที่ compiler เสนอมา:

```rust
// เพิ่ม lifetime parameter 'a เพื่อบอก compiler ว่า "reference ที่ field name เก็บอยู่
// ต้องมีชีวิตอยู่ไม่สั้นกว่า struct Product<'a> ตัวนี้เอง" — รายละเอียดเชิงลึกของ syntax นี้
// (ทำไมต้องมี 'a, compiler infer ให้ได้แค่ไหน, ใช้กับหลาย field/หลาย lifetime พร้อมกันยังไง)
// จะเรียนเต็ม ๆ ใน Part 20 และ Part 23 — ตอนนี้แค่รู้ว่าไวยากรณ์หน้าตาเป็นแบบนี้ก็เพียงพอ
struct Product<'a> {
    name: &'a str,
    price: f64,
    quantity: u32,
}

fn main() {
    let owned_name = String::from("เมาส์ไร้สาย");

    let product = Product {
        name: &owned_name, // ยืม owned_name มา ไม่ได้ copy ข้อมูลจริง
        price: 299.0,
        quantity: 3,
    };

    println!(
        "{} ราคา {:.2} บาท จำนวน {} ชิ้น",
        product.name, product.price, product.quantity
    );
    // product ใช้ได้ตลอดตราบใดที่ owned_name ยังไม่ถูก drop
    // ถ้า owned_name หมดอายุก่อน product จะยังพยายามใช้งาน compiler จะปฏิเสธทันที (คล้ายที่เห็นใน Part 7)
}
```

ผลลัพธ์:

```
เมาส์ไร้สาย ราคา 299.00 บาท จำนวน 3 ชิ้น
```

โค้ดนี้ compile ผ่านและทำงานถูกต้อง แต่สังเกตว่า `Product<'a>` ตอนนี้ "ผูกติด" อยู่กับอายุของ `owned_name` เสมอ —
ตัวแปร `product` จะใช้งานได้แค่ในช่วงที่ `owned_name` ยังไม่ถูก drop เท่านั้น ซึ่งเพิ่มความซับซ้อนในการใช้งานพอสมควร
(ต้องคอยดูแลเรื่อง lifetime ของทั้งสองตัวแปรไปพร้อมกันตลอดเวลา)

**คำแนะนำสำหรับตอนนี้**: จนกว่าจะเรียน Part 20 (Lifetimes เบื้องต้น) และ Part 23 (Lifetimes ขั้นสูง) อย่างเป็นทางการ
ให้ **เลือกเก็บ owned type อย่าง `String` ในสถานการณ์ที่ไม่แน่ใจเสมอ** แทนการเก็บ `&str`/reference ไว้ใน struct
เพราะ owned type ไม่มีข้อจำกัดเรื่อง lifetime ผูกติดกับ struct อื่นเลย ใช้งานได้อย่างอิสระมากกว่ามาก ต้นทุนที่แลกมาคือ
การจอง heap memory เพิ่มขึ้นเล็กน้อย ซึ่งในโปรแกรมส่วนใหญ่ไม่ใช่ปัญหาจนกว่าจะถึงจุดที่ต้อง optimize performance จริงจัง
(และตอนนั้นคุณจะมีความรู้เรื่อง lifetime พอที่จะตัดสินใจได้อย่างมั่นใจแล้ว)

### 9.6 Partial Move: ย้าย Field ออกจาก Struct ทำให้ทั้งก้อนใช้ไม่ได้

ทวนจาก Part 6: การ assign ค่าที่ไม่ implement `Copy` (เช่น `String`) ให้ตัวแปรใหม่ คือการ "ย้าย" (move) ความเป็นเจ้าของ
ไม่ใช่การ copy สิ่งที่มือใหม่มักไม่คาดคิดคือ **กฎนี้ใช้กับ field ของ struct ด้วยเช่นกัน** และผลที่ตามมาน่าสนใจกว่าที่คิด:

```rust
struct Product {
    name: String,
    price: f64,
    quantity: u32,
}

fn print_summary(product: &Product) {
    println!(
        "{} ราคา {:.2} จำนวน {} ชิ้น",
        product.name, product.price, product.quantity
    );
}

fn main() {
    let item = Product {
        name: String::from("เมาส์ไร้สาย"),
        price: 299.0,
        quantity: 3,
    };

    // item.name เป็น String (ไม่ implement Copy) — เขียนแบบนี้คือ "ย้าย" ความเป็นเจ้าของ
    // ออกจาก item.name ไปยัง moved_name ตัวใหม่ ไม่ใช่การ copy ค่า
    let moved_name = item.name;

    // แม้เราไม่ได้แตะ item.price หรือ item.quantity เลย แต่ทั้ง struct item
    // ก็ถือว่า "ถูก partial move" ไปแล้ว — จะยืม (borrow) item ทั้งตัวอีกไม่ได้อีกต่อไป
    print_summary(&item);

    println!("ชื่อที่ย้ายออกไปแล้ว: {moved_name}");
}
```

โค้ดนี้ **compile ไม่ผ่าน** ที่บรรทัด `print_summary(&item)`:

```
error[E0382]: borrow of partially moved value: `item`
  --> src/main.rs:27:19
   |
23 |     let moved_name = item.name;
   |                      --------- value partially moved here
...
27 |     print_summary(&item);
   |                   ^^^^^ value borrowed here after partial move
   |
   = note: partial move occurs because `item.name` has type `String`, which does not implement the `Copy` trait
```

นี่คือสิ่งที่เรียกว่า **partial move** (การย้ายออกเพียงบางส่วน) — คำว่า "partial" สำคัญมาก: เราย้ายออกไปแค่ **field เดียว**
(`item.name`) เท่านั้น field อื่น (`price`, `quantity`) ยังอยู่ใน `item` ครบถ้วนดี แต่ **Rust ไม่อนุญาตให้ยืม (borrow)
struct ทั้งก้อนอีก ถ้ามี field ใดก็ตามที่ถูกย้ายออกไปแล้ว** เพราะการยืม `&item` หมายถึง "ขอดูข้อมูลทุก field ของ item"
แต่ในความเป็นจริง `item.name` ไม่มีข้อมูลอยู่แล้ว (มันถูกย้ายไปเป็นของ `moved_name` แล้ว) — ถ้า Rust ยอมให้ยืมได้
`print_summary` ก็จะพยายามอ่าน `product.name` ที่ไม่มีข้อมูลจริงรองรับอยู่เลย ซึ่งเป็นสถานการณ์อันตรายพอดีกับที่ borrow
checker ถูกออกแบบมาป้องกันตั้งแต่ต้น

ข้อสังเกตที่สำคัญ: ถ้าเราลบบรรทัด `print_summary(&item);` ออก แล้วเข้าถึงแค่ `item.price` หรือ `item.quantity` ตรง ๆ
(ไม่ใช่ยืม `item` ทั้งก้อน) โค้ดจะยัง compile ผ่าน เพราะ field ที่เป็น `Copy` (เช่น `f64`, `u32`) ยังอ่านได้ตามปกติแม้
field อื่นในก้อนเดียวกันถูกย้ายออกไปแล้ว — Rust ตรวจสอบสถานะ "ถูกย้ายหรือยัง" เป็นรายตัวต่อ field ไม่ใช่ทั้งก้อนเสมอไป
แต่ทันทีที่คุณต้องการ **อ้างถึง struct ทั้งก้อนพร้อมกัน** (เช่นยืมทั้งก้อน, ส่งทั้งก้อนไปฟังก์ชัน, หรือ debug print
ทั้งก้อน) การมี field ที่ถูกย้ายไปแล้วแม้แต่ตัวเดียวก็ทำให้ทำแบบนั้นไม่ได้อีก

#### วิธีแก้: ยืม (`&`) แทนการย้าย

ถ้าคุณต้องการแค่ "ดู" ค่าของ field โดยไม่ต้องการเป็นเจ้าของมันจริง ๆ (ซึ่งเป็นกรณีส่วนใหญ่) ให้ **ยืม** field นั้นด้วย `&`
แทนการ assign ตรง ๆ — วิธีนี้ตรงกับหลักการ "ยืมเมื่อแค่ต้องการดู เป็นเจ้าของเมื่อต้องการเก็บไว้ใช้ต่อ" จาก Part 7:

```rust
struct Product {
    name: String,
    price: f64,
    quantity: u32,
}

fn print_summary(product: &Product) {
    println!(
        "{} ราคา {:.2} จำนวน {} ชิ้น",
        product.name, product.price, product.quantity
    );
}

fn main() {
    let item = Product {
        name: String::from("เมาส์ไร้สาย"),
        price: 299.0,
        quantity: 3,
    };

    // แทนที่จะ "ย้าย" item.name ออกมาด้วย let moved_name = item.name;
    // ให้ "ยืม" ด้วย & แทน — ผลคือ borrowed_name เป็น &String ที่ชี้กลับไปยัง item.name เดิม
    // item ยังคงเป็นเจ้าของ field ทุกตัวครบถ้วน ไม่มี field ไหนถูกย้ายออกไปเลย
    let borrowed_name: &String = &item.name;
    println!("ชื่อที่ยืมมาดู: {borrowed_name}");

    // เพราะไม่มี field ไหนถูก move ออก item ทั้งตัวจึงยังยืมซ้ำได้ตามปกติ
    print_summary(&item);

    // และยังใช้ item ต่อได้อีกเรื่อย ๆ ตราบใดที่ borrowed_name หมดอายุไปแล้ว (ตาม borrow checker rule จาก Part 7)
    println!("เข้าถึง item.name ตรง ๆ ได้เหมือนเดิม: {}", item.name);
}
```

ผลลัพธ์:

```
ชื่อที่ยืมมาดู: เมาส์ไร้สาย
เมาส์ไร้สาย ราคา 299.00 จำนวน 3 ชิ้น
เข้าถึง item.name ตรง ๆ ได้เหมือนเดิม: เมาส์ไร้สาย
```

**บทเรียนสำคัญของหัวข้อนี้**: เมื่อทำงานกับ struct ที่มี field เป็น owned type (เช่น `String`) กฎง่าย ๆ ที่ควรจำไว้คือ
**อย่า `let x = struct_instance.field;` เว้นแต่คุณตั้งใจจริง ๆ ที่จะย้ายความเป็นเจ้าของออกมาและไม่ใช้ struct instance
เดิมทั้งก้อนอีก** ถ้าแค่ต้องการอ่านค่า ให้ยืมด้วย `&struct_instance.field` เสมอ ซึ่งเป็นสิ่งที่ method แบบ `&self`
(ที่เราจะเรียนในหัวข้อ 9.10) ช่วยบังคับให้เกิดขึ้นโดยธรรมชาติอยู่แล้ว เพราะ `&self` คือการยืม struct ทั้งก้อนมาแบบอ่านอย่าง
เดียวตั้งแต่ต้น

### 9.7 Struct Update Syntax: สร้าง Instance ใหม่จากของเดิม

สถานการณ์ที่พบบ่อยมากคือ: มี struct instance หนึ่งตัวอยู่แล้ว และต้องการสร้าง instance ใหม่ที่ **เหมือนเดิมทุกอย่าง
ยกเว้นบาง field** — เช่น สินค้าตัวเดิมแต่ปรับราคาลดพิเศษ Rust มี syntax เฉพาะสำหรับกรณีนี้เรียกว่า **struct update
syntax** ใช้ `..instance_เดิม` เพื่อบอกว่า "field ที่ไม่ได้เขียนไว้ข้างบน ให้เอามาจาก instance เดิมทั้งหมด":

```rust
#[derive(Debug)]
struct Product {
    name: String,
    price: f64,
    quantity: u32,
}

fn main() {
    let original = Product {
        name: String::from("เมาส์ไร้สาย"),
        price: 299.0,
        quantity: 15,
    };

    // สร้าง Product ตัวใหม่ที่เหมือน original เป๊ะ ทุก field ยกเว้น price ที่ปรับลดราคาลงเหลือ 249.0
    // ..original บอกว่า "field ที่ไม่ได้เขียนไว้ข้างบน ให้เอามาจาก original ทั้งหมด"
    let discounted = Product {
        price: 249.0,
        ..original
    };

    println!("{discounted:?}");

    // ข้อควรระวัง: original.name เป็น String (ไม่ implement Copy) และ ..original
    // ต้อง "ย้าย" field ที่ไม่ใช่ Copy ออกจาก original มาสร้าง discounted
    // ดังนั้น original ทั้งตัวจะถูก partial move ไปแล้ว (เหมือนหัวข้อ partial move ก่อนหน้า)
    // บรรทัดนี้จึงใช้ original ทั้งตัวไม่ได้อีก (ลอง uncomment ดูจะเจอ E0382):
    // println!("{original:?}");

    // แต่ field ที่เป็น Copy (เช่น quantity: u32) ยังอ่านผ่าน original ได้ตามปกติแม้ name ถูกย้ายไปแล้ว
    println!(
        "จำนวนสินค้าต้นฉบับ (field ที่เป็น Copy ยังอ่านได้): {}",
        original.quantity
    );
}
```

ผลลัพธ์:

```
Product { name: "เมาส\u{e4c}ไร\u{e49}สาย", price: 249.0, quantity: 15 }
จำนวนสินค้าต้นฉบับ (field ที่เป็น Copy ยังอ่านได้): 15
```

(หมายเหตุเรื่อง `\u{e4c}` และ `\u{e49}` ที่ปรากฏใน output — นี่ไม่ใช่บั๊ก เราจะอธิบายเหตุผลจริงของมันในหัวข้อ 9.12
เรื่อง `#[derive(Debug)]`)

สังเกตว่า struct update syntax คือ **น้ำตาลไวยากรณ์ (syntax sugar)** ที่ compiler แปลงเป็นการเขียนแบบเต็มให้เอง — บรรทัด
`Product { price: 249.0, ..original }` เทียบเท่ากับการเขียน
`Product { price: 249.0, name: original.name, quantity: original.quantity }` ทุกประการ ซึ่งอธิบายได้ชัดเจนว่าทำไม
`original` ถึงถูก partial move ไปแล้ว: เพราะภายใต้ผิว มันคือการ assign `original.name` (ที่เป็น `String`, ไม่ใช่ `Copy`)
ให้กับ field ใหม่นั่นเอง กฎ ownership ที่เรียนมาทั้งหมดใช้ได้เหมือนเดิมทุกประการ ไม่มีข้อยกเว้นพิเศษสำหรับ struct update
syntax แต่อย่างใด

**ข้อสังเกตเชิงปฏิบัติ**: struct update syntax เหมาะมากกับสถานการณ์ที่ต้องการ "variant" ของค่าเดิมแบบเปลี่ยนเพียงไม่กี่
field (เช่น การใช้ default config แล้ว override บางค่า, หรือสร้าง "รายการสั่งซื้อพิเศษ" จากสินค้าตัวเดิมแต่ปรับราคา)
แต่ถ้าคุณยังต้องใช้ instance เดิมทั้งก้อนต่อไปหลังจากนั้น (ไม่ใช่แค่ field ที่เป็น `Copy`) ให้ `.clone()` instance เดิม
ก่อนใช้ `..` เพื่อหลีกเลี่ยง partial move เช่น `..original.clone()` — แต่ก็ต้องแลกกับ cost การ clone ข้อมูลทั้งก้อน
เพิ่มขึ้นมา ควรพิจารณาตามความเหมาะสมของแต่ละสถานการณ์

### 9.8 Tuple Struct: เมื่อชื่อ Field ไม่จำเป็น

บางครั้งเราต้องการ "ห่อ" ค่าง่าย ๆ ไว้เป็น type ใหม่ โดยที่ชื่อ field ไม่ได้ช่วยอะไรเพิ่มขึ้น (เพราะตำแหน่งของค่าก็สื่อ
ความหมายชัดเจนอยู่แล้วในตัวมันเอง หรือมีแค่ field เดียว) Rust มี **tuple struct** สำหรับกรณีนี้ — นิยามคล้าย struct ปกติ
แต่ใช้วงเล็บแทนปีกกา และไม่ต้องตั้งชื่อ field เลย:

```rust
// tuple struct: มีชื่อ type แต่ field ไม่มีชื่อ อ้างอิงด้วยตำแหน่งเหมือน tuple ธรรมดา
struct Point(f64, f64);

// อีกตัวอย่าง: ใช้ tuple struct เป็น "wrapper" บาง ๆ รอบชนิดข้อมูลพื้นฐาน
// เพื่อให้ compiler แยกแยะความหมายที่ต่างกันได้ แม้ข้างในเป็น f64 เหมือนกัน
struct Meters(f64);
struct Feet(f64);

fn distance_from_origin(p: &Point) -> f64 {
    (p.0 * p.0 + p.1 * p.1).sqrt()
}

// รับเฉพาะ Meters เท่านั้น ส่ง Feet เข้ามาตรง ๆ จะ compile ไม่ผ่าน แม้ทั้งคู่เก็บ f64 ข้างในเหมือนกัน
fn print_height_in_meters(height: &Meters) {
    println!("ความสูง: {:.2} เมตร", height.0);
}

fn main() {
    let origin_offset = Point(3.0, 4.0);
    println!("ระยะจากจุดกำเนิด: {}", distance_from_origin(&origin_offset));

    // เข้าถึงสมาชิกด้วย .0 / .1 เหมือน tuple ปกติ
    println!("x = {}, y = {}", origin_offset.0, origin_offset.1);

    let height = Meters(1.75);
    print_height_in_meters(&height);

    // Point(3.0, 4.0) และ Meters(1.75) ทั้งคู่เก็บ f64 อยู่ข้างใน แต่เป็นคนละ type กันโดยสิ้นเชิง
    // ในสาย compiler มองว่า Meters ≠ Feet ≠ f64 เปล่า ๆ แม้ shape ข้างในจะเหมือนกันก็ตาม
    // แนวคิดนี้เรียกว่า "newtype pattern" ซึ่งเราจะเจาะลึกประโยชน์เต็มรูปแบบใน Part 53
    let feet = Feet(5.74);
    println!("ความสูงอีกหน่วย: {:.2} ฟุต", feet.0);
}
```

ผลลัพธ์:

```
ระยะจากจุดกำเนิด: 5
x = 3, y = 4
ความสูง: 1.75 เมตร
ความสูงอีกหน่วย: 5.74 ฟุต
```

**เมื่อไหร่ควรใช้ tuple struct**:

1. **Wrapper แบบเบา (lightweight wrapper)** — เมื่อต้องการห่อค่าพื้นฐานเพียงตัวเดียวเป็น type ใหม่ เพื่อให้ compiler
   ช่วยแยกแยะความหมาย เช่น `Meters(f64)` กับ `Feet(f64)` ในตัวอย่างข้างบน — แม้ข้างในเก็บ `f64` เหมือนกัน แต่ประกาศเป็น
   type คนละตัว ทำให้ **compiler ปฏิเสธการส่ง `Feet` ไปยังฟังก์ชันที่ต้องการ `Meters` ทันที** ลองดู error จริงถ้าพยายาม
   ทำแบบนั้น:

   ```rust
   struct Meters(f64);
   struct Feet(f64);

   fn print_height_in_meters(height: &Meters) {
       println!("ความสูง: {:.2} เมตร", height.0);
   }

   fn main() {
       let feet = Feet(5.74);
       print_height_in_meters(&feet); // ส่ง Feet เข้าไปในฟังก์ชันที่รับ &Meters ตรง ๆ
   }
   ```

   ```
   error[E0308]: mismatched types
     --> src/main.rs:10:28
      |
   10 |     print_height_in_meters(&feet); // ส่ง Feet เข้าไปในฟังก์ชันที่รับ &Meters ตรง ๆ
      |     ---------------------- ^^^^^ expected `&Meters`, found `&Feet`
      |     |
      |     arguments to this function are incorrect
   ```

   ลองนึกภาพว่าถ้าทั้งสองฟังก์ชันรับ `f64` ตรง ๆ (ไม่ห่อเป็น type ใหม่) บั๊กแบบนี้ (ส่งความสูงหน่วยฟุตเข้าไปในฟังก์ชัน
   ที่คาดหวังหน่วยเมตร) จะ **compile ผ่านโดยไม่มี error เลย** เพราะ `f64` ก็คือ `f64` แต่พอห่อเป็น `Meters`/`Feet`
   compiler จะจับบั๊กเชิง "ความหมาย" แบบนี้ให้เราตั้งแต่ก่อนรันโปรแกรมเลย — นี่คือแก่นของแนวคิดที่เรียกว่า **newtype
   pattern** ซึ่งเราจะเห็นประโยชน์เพิ่มเติมอีกมาก (เช่นการ implement trait ให้ type ที่ยืมมาจาก crate อื่น) ใน Part 53

2. **จำนวน field น้อยและตำแหน่งสื่อความหมายชัดเจนอยู่แล้ว** — เช่น `Point(f64, f64)` สำหรับพิกัด x, y ที่รู้กันเป็น
   convention ทั่วไปอยู่แล้วว่าตัวแรกคือ x ตัวที่สองคือ y ไม่จำเป็นต้องตั้งชื่อ field ให้ยาวขึ้น

**เมื่อไหร่ไม่ควรใช้**: ถ้า struct มีหลาย field ที่ความหมายไม่ชัดเจนจากตำแหน่งเพียงอย่างเดียว (เช่น `Product` ที่มี name/
price/quantity) ควรใช้ named-field struct เสมอ เพราะปัญหาเดียวกับที่เราเห็นในหัวข้อ 9.1 (สลับตำแหน่งผิดโดยไม่มีอะไรเตือน)
ก็ยังเกิดขึ้นได้กับ tuple struct เช่นกัน ถ้ามันมีหลาย field ที่ type เหมือนกัน — tuple struct แก้ปัญหาเรื่อง "แยกแยะ type"
ระหว่าง type ต่างกัน แต่ไม่ได้แก้ปัญหาเรื่อง "จำลำดับ field ภายใน type เดียวกัน"

### 9.9 Unit-Like Struct: Struct ที่ไม่มี Field เลย

Rust ยังมี struct อีกแบบที่ **ไม่มี field เลยแม้แต่ตัวเดียว** เรียกว่า **unit-like struct** เขียนโดยไม่มีวงเล็บหรือปีกกา
ต่อท้ายชื่อเลย:

```rust
// unit-like struct: ไม่มี field เลยแม้แต่ตัวเดียว ไม่มีวงเล็บ ไม่มีปีกกา
// ใช้เป็น "marker" — ตัวแทนของแนวคิดหนึ่ง ๆ ที่ไม่ต้องเก็บข้อมูลอะไรเลย มีไว้เพื่อให้ type system
// แยกแยะความหมาย หรือเพื่อ implement trait ลงไป (เราจะเห็นประโยชน์เต็มรูปแบบใน Part 19 เรื่อง Traits)
struct AlwaysEqual;

// ตัวอย่างที่เป็นรูปธรรมกว่า: ใช้ unit struct แทน "สถานะ" ที่ไม่มีข้อมูลประกอบ
struct GuestUser;

fn greet_guest(_guest: &GuestUser) {
    println!("สวัสดีผู้ใช้ทั่วไป (ไม่ระบุตัวตน)");
}

fn main() {
    // สร้าง instance ได้โดยไม่ต้องใส่ field ใด ๆ เลย (ไม่มีวงเล็บ ไม่มีปีกกา)
    let _subject = AlwaysEqual;

    let guest = GuestUser;
    greet_guest(&guest);
}
```

ผลลัพธ์:

```
สวัสดีผู้ใช้ทั่วไป (ไม่ระบุตัวตน)
```

ชื่อ "unit-like" มาจากความคล้ายกับ **unit type** `()` ที่เราเรียนใน Part 3 — เป็น type ที่มี "ค่าที่เป็นไปได้" อยู่แค่
ค่าเดียว ไม่มีข้อมูลอะไรให้เลือกหรือแยกแยะเลย คำถามที่มือใหม่มักถามคือ: **แล้วมันมีประโยชน์อะไร ในเมื่อไม่เก็บข้อมูลเลย?**

คำตอบคือ unit-like struct มีประโยชน์ 2 อย่างหลัก ๆ ที่จะเห็นชัดเจนขึ้นมากในบทหลัง:

1. **เป็น "marker type"** — ตัวแทนของแนวคิดในระดับ type system โดยไม่ต้องมีข้อมูลจริง เช่นในตัวอย่างข้างบน `GuestUser`
   ใช้แยกแยะ "ผู้ใช้แบบไม่ระบุตัวตน" จาก "ผู้ใช้ที่ล็อกอินแล้ว" (ซึ่งอาจเป็น struct ที่มี field เก็บ username/email จริง)
   compiler จะบังคับให้ฟังก์ชันที่รับ `&GuestUser` รับได้แค่ guest user เท่านั้น แยกออกจาก user ประเภทอื่นอย่างชัดเจน
   ในระดับ type แม้ในความเป็นจริงมันไม่มีข้อมูลอะไรเลยก็ตาม
2. **implement trait โดยไม่ต้องมีข้อมูล** — บางครั้งเราต้องการสร้าง type ที่มีพฤติกรรมบางอย่าง (ผ่านการ implement trait)
   แต่ตัว type เองไม่จำเป็นต้องเก็บข้อมูลอะไรเลย เช่นตัวแทนของ "กลยุทธ์การคำนวณ" อย่างหนึ่งที่ไม่มี state ภายใน — เรื่องนี้
   จะเห็นภาพชัดเจนมากขึ้นเมื่อเราเรียน trait อย่างเป็นทางการใน **Part 19**

ตอนนี้จำแค่ว่า unit-like struct คือ struct ที่เบาที่สุดที่เป็นไปได้ในภาษา Rust — ไม่มีข้อมูลใด ๆ เก็บอยู่ใน memory เลย
(ขนาด 0 byte) แต่ยังมีตัวตนเป็น type ที่ชัดเจนในระดับ compile time

### 9.10 `impl` Block และ Method: `&self`, `&mut self`, `self`

จนถึงตรงนี้ struct ของเรามีแต่ข้อมูล (field) แต่ไม่มี "พฤติกรรม" ผูกอยู่กับมันเลย — เราต้องเขียนฟังก์ชันแยกข้างนอกที่รับ
struct เป็น parameter ทุกครั้ง (เช่น `print_receipt_line(&item)`) ซึ่งอ่านไม่เป็นธรรมชาติเท่ากับการเขียน `item.something()`
Rust ให้เรานิยาม **method** (ฟังก์ชันที่ผูกกับ struct) ผ่าน **`impl` block**:

```rust
struct Product {
    name: String,
    price: f64,
    quantity: u32,
}

impl Product {
    // (1) &self — ยืมแบบอ่านอย่างเดียว (immutable borrow)
    // ใช้เมื่อ method แค่ "อ่าน" ข้อมูลของ struct แล้วคำนวณ/คืนค่าอะไรบางอย่าง โดยไม่แก้ไขตัวมันเอง
    // เป็น receiver ที่ควรเลือกเป็นอันดับแรกเสมอ เว้นแต่มีเหตุผลชัดเจนว่าต้องแก้ไขหรือย้ายเจ้าของ
    fn total_value(&self) -> f64 {
        self.price * self.quantity as f64
    }

    fn describe(&self) -> String {
        format!("{} ({:.2} บาท x {})", self.name, self.price, self.quantity)
    }

    // (2) &mut self — ยืมแบบแก้ไขได้ (mutable borrow)
    // ใช้เมื่อ method ต้องเปลี่ยนแปลงข้อมูลภายใน struct แต่ผู้เรียกยังต้องการเป็นเจ้าของ struct ต่อไป
    // หลังเรียก method เสร็จ (ไม่ได้ต้องการ "กิน" struct นั้นทิ้งไปเลย)
    fn restock(&mut self, additional: u32) {
        self.quantity += additional;
    }

    fn apply_discount_percent(&mut self, percent: f64) {
        self.price *= 1.0 - percent / 100.0;
    }

    // (3) self — รับความเป็นเจ้าของทั้งหมด (take ownership)
    // ใช้เมื่อ method ต้องการ "ใช้ครั้งเดียวแล้วจบ" เช่นแปลง struct นี้ไปเป็นอะไรอีกอย่างหนึ่ง
    // (consume แล้วคืน type ใหม่) หลังเรียกแล้ว struct ตัวเดิมจะถูกย้ายเข้าไปใน method
    // และผู้เรียกจะใช้ตัวแปรเดิมต่อไม่ได้อีก (เหมือนกฎ move ปกติจาก Part 6)
    fn into_receipt_line(self) -> String {
        // self ถูกย้าย (moved) เข้ามาทั้งตัวแล้ว จึงดึง field ออกมาใช้ได้โดยไม่ต้อง clone
        format!(
            "{} x{} = {:.2} บาท",
            self.name,
            self.quantity,
            self.price * self.quantity as f64
        )
    }
}

fn main() {
    let mut item = Product {
        name: String::from("เมาส์ไร้สาย"),
        price: 299.0,
        quantity: 10,
    };

    // เรียก &self method: ยืมอ่าน item แล้วยังใช้ item ต่อได้
    println!("มูลค่ารวม: {:.2} บาท", item.total_value());
    println!("รายละเอียด: {}", item.describe());

    // เรียก &mut self method: ต้องมี item เป็น mut binding เท่านั้น (ตาม borrow rule จาก Part 7)
    item.restock(5);
    item.apply_discount_percent(10.0);
    println!("หลังเติมสต็อกและลดราคา: {}", item.describe());

    // เรียก self method (consuming): item ถูกย้ายเข้าไปทั้งตัว
    let receipt_line = item.into_receipt_line();
    println!("บรรทัดใบเสร็จ: {receipt_line}");

    // item ใช้ต่อไม่ได้อีกแล้ว ณ จุดนี้ — ลอง uncomment บรรทัดล่างจะเจอ E0382 (use of moved value)
    // println!("{}", item.name);
}
```

ผลลัพธ์:

```
มูลค่ารวม: 2990.00 บาท
รายละเอียด: เมาส์ไร้สาย (299.00 บาท x 10)
หลังเติมสต็อกและลดราคา: เมาส์ไร้สาย (269.10 บาท x 15)
บรรทัดใบเสร็จ: เมาส์ไร้สาย x15 = 4036.50 บาท
```

มาดูรายละเอียดของ syntax และความหมายเชิงลึกของ receiver ทั้ง 3 แบบทีละตัว:

#### `impl Product { ... }` คืออะไร

`impl` (ย่อจาก **implementation**) คือ block ที่ใช้ **นิยามพฤติกรรม (method และ associated function) ให้กับ type
ที่มีอยู่แล้ว** — สังเกตว่า `impl` **แยกจากการนิยาม struct โดยสิ้นเชิง** (ต่างจากภาษาอย่าง Java/C++ ที่ field และ method
เขียนรวมกันอยู่ใน class เดียว) เหตุผลที่ Rust แยกออกจากกันมีทั้งเชิงปฏิบัติ (คุณสามารถมี `impl` block ได้หลายบล็อกสำหรับ
struct เดียวกัน จัดกลุ่ม method ตามหมวดหมู่ได้อย่างอิสระ) และเชิงสถาปัตยกรรม (ทำให้ trait — ซึ่งเป็นการ "เพิ่มพฤติกรรม"
ให้ type ทีหลัง — เป็น mechanism เดียวกันกับ `impl` ธรรมดา เพียงแค่ระบุว่า implement trait ไหน ซึ่งเราจะเห็นใน Part 19)

#### `&self`: ยืมแบบอ่านอย่างเดียว

`self` ในตำแหน่ง parameter แรกของ method หมายถึง "ตัว instance ที่กำลังเรียก method นี้" — เมื่อเขียน `&self` แปลว่า
method นี้ **ยืม** instance มาแบบอ่านอย่างเดียว (เทียบเท่ากับ `self: &Self` แบบเต็ม โดย `Self` หมายถึง type ที่กำลัง impl
อยู่ ในที่นี้คือ `Product`) นี่คือ receiver ที่ **ควรเลือกเป็นตัวเลือกแรกเสมอ** เมื่อเขียน method ใหม่ เพราะมันมีข้อจำกัด
น้อยที่สุดสำหรับผู้เรียก — ผู้เรียกยังเป็นเจ้าของ instance เดิม ยังใช้งานมันต่อได้อีกหลังเรียก method (และยังเรียก `&self`
method อื่น ๆ พร้อมกันได้หลายตัว ตามกฎ "ยืมอ่านพร้อมกันได้หลายที่" จาก Part 7) `total_value(&self)` และ `describe(&self)`
ในตัวอย่างข้างบนแค่ต้องการอ่านค่า `price`, `quantity`, `name` มาคำนวณ/จัดรูปแบบ ไม่มีความจำเป็นต้องแก้ไขอะไรเลย จึงใช้
`&self` ได้อย่างเหมาะสมที่สุด

#### `&mut self`: ยืมแบบแก้ไขได้

`&mut self` (เทียบเท่า `self: &mut Self`) คือการยืม instance มาแบบ **แก้ไขได้** — ใช้เมื่อ method ต้องเปลี่ยนแปลง field
ของ instance จริง ๆ เช่น `restock(&mut self, additional: u32)` ที่เพิ่มค่า `quantity` เข้าไป ข้อจำกัดสำคัญตามกฎจาก
Part 7 คือ **ตัวแปรที่จะเรียก `&mut self` method ได้ ต้องประกาศด้วย `let mut` เท่านั้น** (เราจะเห็น error จริงถ้าลืม
`mut` ในหัวข้อกับดักท้ายบท) และในเวลาที่ borrow แบบ mutable นี้ยังมีชีวิตอยู่ จะไม่สามารถมี borrow อื่น (ทั้งแบบอ่านหรือ
แบบเขียน) พร้อมกันได้เลย ตามกฎ "ยืมเขียนได้แค่ที่เดียว ห้ามมีการยืมอ่านพร้อมกัน" ที่เรียนมาแล้ว

ข้อดีของ `&mut self` เทียบกับการ return instance ใหม่ทุกครั้ง (immutable style) คือ **ประสิทธิภาพ**: ไม่ต้อง copy/clone
ข้อมูลทั้งก้อนเพื่อสร้าง instance ใหม่แค่เพราะต้องการเปลี่ยนค่าเพียง field เดียว — สำหรับ struct ขนาดใหญ่หรือถูกเรียกใช้
บ่อยมาก การ mutate ในที่เดิมแบบนี้มีประสิทธิภาพดีกว่ามาก

#### `self`: รับความเป็นเจ้าของทั้งหมด (consuming)

`self` เพียว ๆ (ไม่มี `&` นำหน้า) หมายถึง method นี้ **รับความเป็นเจ้าของ instance ทั้งตัวเข้ามาโดยตรง** — instance เดิม
จะถูกย้าย (moved) เข้าไปใน method ตามกฎ move ปกติจาก Part 6 ทุกประการ ผลคือ **ตัวแปรเดิมที่เรียก method นี้จะใช้งานต่อ
ไม่ได้อีกเลยหลังจากเรียกเสร็จ** (เว้นแต่ type นั้น implement `Copy` ซึ่ง struct ที่มี field เป็น `String` ไม่สามารถ
implement `Copy` ได้อยู่แล้ว)

ทำไมถึงต้องมี receiver แบบนี้ด้วย ในเมื่อดูเหมือนจะจำกัดกว่า `&mut self`? เหตุผลคือบางสถานการณ์ **ต้องการ "แปลง" ค่าเดิม
ไปเป็นอะไรอีกอย่างหนึ่งแบบถาวร** ไม่ใช่แค่แก้ไขค่าในที่เดิม — `into_receipt_line(self) -> String` ในตัวอย่างข้างบนคือ
ตัวอย่างที่ดี: มันไม่ได้ "แก้ไข" `Product` แต่ "แปลง" `Product` ทั้งก้อนให้กลายเป็น `String` แล้วก้อนเดิมก็ไม่มีประโยชน์
ที่จะเก็บไว้ใช้ต่ออีก (concept นี้จะพบอีกบ่อยมากตอนเรียน iterator ใน Part 25-26 ที่ method อย่าง `.into_iter()` ก็ใช้
`self` receiver แบบนี้เหมือนกัน) การใช้ `self` แทน `&self`/`&mut self` ยังมีข้อดีด้าน**ประสิทธิภาพเชิง ownership**:
เพราะ method รับความเป็นเจ้าของมาแล้ว มันสามารถ **ย้าย field ออกไปใช้ตรง ๆ** (เช่น `self.name` ในตัวอย่าง) โดยไม่ต้อง
clone ข้อมูล — ถ้าใช้ `&self` แทน จะต้อง `.clone()` field ที่เป็น `String` ก่อนเสมอ เพราะไม่มีความเป็นเจ้าของให้ย้ายออก

ลองดู error จริงถ้าพยายามใช้ `item` ต่อหลังเรียก `into_receipt_line()`:

```
error[E0382]: borrow of moved value: `item`
  --> src/main.rs:66:20
   |
46 |     let mut item = Product {
   |         -------- move occurs because `item` has type `Product`, which does not implement the `Copy` trait
...
62 |     let receipt_line = item.into_receipt_line();
   |                             ------------------- `item` moved due to this method call
...
66 |     println!("{}", item.name);
   |                    ^^^^^^^^^ value borrowed here after move
   |
note: `Product::into_receipt_line` takes ownership of the receiver `self`, which moves `item`
  --> src/main.rs:34:26
   |
34 |     fn into_receipt_line(self) -> String {
   |                          ^^^^
```

#### ตารางสรุปการเลือก Receiver

| Receiver | ยืม/เป็นเจ้าของ | ผู้เรียกใช้ instance ต่อได้ไหม | ใช้เมื่อ |
|---|---|---|---|
| `&self` | ยืมแบบอ่าน | ได้ (ยืมอ่านซ้ำได้หลายที่) | อ่านข้อมูล คำนวณ คืนค่า โดยไม่แก้ไข instance — ควรเป็นตัวเลือกแรกเสมอ |
| `&mut self` | ยืมแบบแก้ไข | ได้ (แต่ระหว่าง borrow นี้ ใช้ borrow อื่นพร้อมกันไม่ได้) | แก้ไขค่า field ในที่เดิม โดยยังต้องการเป็นเจ้าของ instance ต่อไปหลังเรียก |
| `self` | รับเป็นเจ้าของทั้งหมด | ไม่ได้ (instance เดิมถูก move เข้าไปแล้ว) | แปลง instance ไปเป็นอย่างอื่นแบบถาวร ("ใช้ครั้งเดียวจบ"), ต้องการย้าย field ออกไปโดยไม่ clone |

**หลักการเลือกที่ควรจำ**: เริ่มต้นด้วย `&self` เสมอ แล้วขยับไปใช้ `&mut self` เมื่อจำเป็นต้องแก้ไขค่าจริง ๆ และสงวน `self`
(consuming) ไว้สำหรับกรณีที่ต้องการ "แปลงร่าง" instance ไปเป็นสิ่งอื่นแบบเด็ดขาดเท่านั้น การเลือก receiver ที่จำกัดสิทธิ์
ผู้เรียกน้อยที่สุดที่ยังทำงานได้ (least privilege) เป็นหลักการออกแบบที่ดีในทุกภาษา และ Rust ทำให้หลักการนี้ถูกบังคับใช้
จริงโดย compiler ไม่ใช่แค่ convention ที่พึ่งความมีวินัยของนักพัฒนาอย่างเดียว

### 9.11 Associated Function เป็น Constructor

สังเกตว่าในทุกตัวอย่างก่อนหน้า เราต้องสร้าง `Product` instance ด้วย struct literal เต็มรูปแบบทุกครั้ง
(`Product { name: ..., price: ..., quantity: ... }`) ซึ่งค่อนข้างยาวและซ้ำซ้อนเมื่อต้องสร้างหลาย instance Rust มี
convention มาตรฐานสำหรับแก้ปัญหานี้ด้วย **associated function** — ฟังก์ชันที่ประกาศอยู่ใน `impl` block เหมือน method
แต่ **ไม่มี `self` เป็น parameter เลย**:

```rust
struct Product {
    name: String,
    price: f64,
    quantity: u32,
}

impl Product {
    // associated function: ไม่มี self เป็น parameter เลย — ไม่ใช่ method ที่เรียกผ่าน instance (.)
    // แต่เรียกผ่านชื่อ type ตรง ๆ ด้วย :: เหมือน type-level function
    // convention มาตรฐานของ Rust คือตั้งชื่อ "new" สำหรับ constructor หลักของ type
    fn new(name: &str, price: f64, quantity: u32) -> Self {
        // Self (ตัว S ใหญ่) คือชื่อย่อของ type ที่กำลัง impl อยู่ (ในที่นี้คือ Product)
        // ใช้ Self แทนการเขียน Product ตรง ๆ ซ้ำ ๆ — ถ้าเปลี่ยนชื่อ struct ทีหลัง ไม่ต้องแก้จุดนี้เลย
        Self {
            name: name.to_string(),
            price,
            quantity,
        }
    }

    // associated function อีกตัว: constructor พิเศษสำหรับกรณี "สินค้าหมดสต็อก" (quantity เริ่มที่ 0 เสมอ)
    fn new_out_of_stock(name: &str, price: f64) -> Self {
        Self::new(name, price, 0) // เรียก associated function อื่นซ้ำได้ตามปกติ
    }

    fn total_value(&self) -> f64 {
        self.price * self.quantity as f64
    }
}

fn main() {
    // เรียกผ่านชื่อ type ตรง ๆ ด้วย :: ไม่ใช่ผ่าน instance ด้วย . แบบ method ทั่วไป
    let item = Product::new("เมาส์ไร้สาย", 299.0, 10);
    println!("{} มูลค่ารวม {:.2} บาท", item.name, item.total_value());

    let empty_item = Product::new_out_of_stock("คีย์บอร์ดรุ่นลิมิเต็ด", 3500.0);
    println!(
        "{} จำนวนคงเหลือ {} ชิ้น",
        empty_item.name, empty_item.quantity
    );
}
```

ผลลัพธ์:

```
เมาส์ไร้สาย มูลค่ารวม 2990.00 บาท
คีย์บอร์ดรุ่นลิมิเต็ด จำนวนคงเหลือ 0 ชิ้น
```

#### Method เทียบกับ Associated Function

ความแตกต่างที่สำคัญที่สุดคือ **วิธีเรียก**:

- **Method** (มี `self`/`&self`/`&mut self`) เรียกผ่าน instance ด้วย `.` เช่น `item.total_value()` — ต้องมี instance
  อยู่แล้วก่อนถึงจะเรียกได้
- **Associated function** (ไม่มี `self`) เรียกผ่านชื่อ type ตรง ๆ ด้วย `::` เช่น `Product::new(...)` — ยังไม่มี instance
  อยู่เลย เพราะจุดประสงค์หลักคือ**สร้าง** instance ตัวแรกขึ้นมา

Rust ไม่มี keyword พิเศษอย่าง `constructor` หรือ `static` (เหมือน static method ใน Java/C#) แยกต่างหาก — associated
function ก็คือฟังก์ชันธรรมดาที่ประกาศอยู่ใน `impl` block โดยไม่มี `self` เท่านั้นเอง แต่ Rust community มี **convention
ที่ยึดถือกันอย่างกว้างขวาง** ว่า associated function ที่ทำหน้าที่เป็น constructor หลักควรตั้งชื่อว่า **`new`** เสมอ —
นี่ไม่ใช่กฎที่ compiler บังคับ (คุณตั้งชื่ออื่นก็ compile ผ่าน) แต่เป็น convention ที่ผู้ใช้ crate ของคุณคาดหวังไว้แล้ว
เมื่อเห็น type ใหม่ พวกเขาจะลองพิมพ์ `TypeName::new(` แล้วดู auto-complete ก่อนเป็นอันดับแรกเสมอ

#### `Self` (S ใหญ่) คืออะไร

สังเกตว่าใน `fn new(...) -> Self` เราใช้ `Self` (ตัว S ใหญ่) แทนการเขียน `Product` ตรง ๆ — `Self` ในบริบทของ `impl`
block หมายถึง **"type ที่กำลัง impl อยู่ ณ ตอนนี้"** เสมอ การใช้ `Self` แทนชื่อ type ตรง ๆ มีข้อดีเชิงการดูแลโค้ด: ถ้าคุณ
เปลี่ยนชื่อ `struct Product` เป็นชื่ออื่นในอนาคต (เช่น `struct InventoryItem`) คุณต้องแก้แค่บรรทัดประกาศ `struct` และ
`impl` เท่านั้น ทุกจุดที่ใช้ `Self` ข้างในจะอัปเดตตามให้อัตโนมัติโดยไม่ต้องแก้อะไรเพิ่ม — เป็นตัวช่วยลด maintenance cost
เล็ก ๆ ที่ใช้กันเป็น idiomatic style ทั่วทั้ง ecosystem Rust

### 9.12 `#[derive(Debug)]` และเพื่อนบ้าน: `Clone`, `PartialEq`

ทวนจาก Part 3: การใช้ `{:?}` (debug formatting) กับ tuple ทำงานได้ทันทีโดยไม่ต้องเขียนอะไรเพิ่ม เพราะ tuple implement
trait `Debug` ให้เองเป็นส่วนหนึ่งของภาษา แต่ **struct ที่เราสร้างขึ้นเองไม่ implement `Debug` ให้อัตโนมัติ** — ถ้าต้องการ
ใช้ `{:?}` กับ struct ของเราเอง ต้องขอให้ compiler generate ให้ ด้วย **attribute** (คำสั่งพิเศษที่ไม่ใช่ syntax ปกติ
ของภาษา แต่บอก compiler ให้ทำอะไรเป็นพิเศษกับโค้ดส่วนที่ระบุ) `#[derive(Debug)]`:

```rust
// #[derive(Debug)] บอก compiler ให้ "เขียน" การ implement trait Debug ให้กับ struct นี้อัตโนมัติ
// ทำให้ใช้ {:?} (และ {:#?} แบบ pretty-print) กับ instance ของ struct นี้ได้
// โดยไม่ต้องเขียน impl Debug for Product ด้วยมือเอง (รายละเอียดเรื่อง trait และ derive macro
// จะเจาะลึกใน Part 19)
#[derive(Debug)]
struct Product {
    name: String,
    price: f64,
    quantity: u32,
}

fn main() {
    let item = Product {
        name: String::from("cable USB-C"),
        price: 199.0,
        quantity: 25,
    };

    // {:?} — debug print แบบบรรทัดเดียว เหมาะกับ log สั้น ๆ หรือ debug ระหว่างพัฒนา
    println!("{item:?}");

    // {:#?} — pretty-print แบบหลายบรรทัด จัด indent ให้อ่านง่าย เหมาะกับข้อมูลที่มีหลาย field/ซับซ้อน
    println!("{item:#?}");
}
```

ผลลัพธ์:

```
Product { name: "cable USB-C", price: 199.0, quantity: 25 }
Product {
    name: "cable USB-C",
    price: 199.0,
    quantity: 25,
}
```

**`derive` คืออะไรกันแน่**: มันคือ **procedural macro** ประเภทหนึ่ง (รายละเอียดเต็มเรื่อง macro จะเรียนใน Part 36)
ที่ทำงานตอน compile time — เมื่อเห็น `#[derive(Debug)]` เหนือ `struct Product`, compiler จะ **generate โค้ด
`impl Debug for Product { ... }` ให้เองโดยอัตโนมัติ** ตามโครงสร้าง field ที่มีอยู่จริง คุณไม่เห็นโค้ดที่ generate นี้
ในไฟล์ source ของคุณเลย (มันถูกสร้างขึ้นเบื้องหลังตอน compile) แต่ผลลัพธ์คือ struct ของคุณมีพฤติกรรม `{:?}` ใช้งานได้
เหมือนกับเขียน `impl Debug` ด้วยมือเองทุกประการ เพียงแค่ประหยัดเวลาเขียนโค้ดซ้ำ ๆ ไปได้มาก

**ข้อจำกัดสำคัญ**: `#[derive(Debug)]` ใช้ได้ก็ต่อเมื่อ **ทุก field ของ struct implement `Debug` ด้วยเช่นกัน** — ถ้า field
ใด field หนึ่งเป็น type ที่ไม่ implement `Debug` (พบได้เมื่อใช้ type จาก crate ภายนอกบางตัว หรือ type ที่ตั้งใจไม่ให้
debug print ได้) การ derive จะ compile ไม่ผ่าน พร้อม error ที่ชี้ไปยัง field ที่เป็นปัญหาโดยตรง

#### หมายเหตุสำคัญ: ข้อความไทยกับ `{:?}`

ลองสังเกตพฤติกรรมที่น่าสนใจเมื่อ debug print ข้อความไทยที่มีวรรณยุกต์หรือสระลอย (เช่น ่ ้ ๊ ๋ ั ึ ื ิ ี ุ ู ซึ่งในทาง
Unicode คือ **combining mark** — อักขระที่ประกอบร่วมกับตัวอักษรฐานเพื่อแสดงผลรวมเป็นหนึ่งรูปตัวอักษร ไม่ใช่ตัวอักษร
ที่สมบูรณ์ในตัวเอง):

```rust
#[derive(Debug)]
struct Product {
    name: String,
}

fn main() {
    let item = Product {
        name: String::from("เมาส์ไร้สาย"),
    };

    println!("{}", item.name); // Display ผ่าน field ตรง ๆ (String implement Display) — ได้ข้อความไทยปกติ
    println!("{item:?}"); // Debug ของทั้ง struct
}
```

ผลลัพธ์:

```
เมาส์ไร้สาย
Product { name: "เมาส\u{e4c}ไร\u{e49}สาย" }
```

สังเกตว่า `{}` (Display) พิมพ์ข้อความไทยออกมาปกติทุกตัวอักษร แต่ `{:?}` (Debug) กลับแสดงตัวอักษร **ทัณฑฆาต ์ (U+0E4C)**
และ **สระ/วรรณยุกต์ ้ (U+0E49)** เป็น escape sequence `\u{e4c}` และ `\u{e49}` แทน — **นี่ไม่ใช่บั๊กของ Rust หรือของบทเรียน
แต่เป็นพฤติกรรมจริงและตั้งใจของ standard library**: การ implement `Debug` สำหรับ `str`/`String` จะ escape อักขระที่เป็น
Unicode combining mark เสมอ (ไม่ว่าจะอยู่ตำแหน่งไหนในสตริง) เพื่อป้องกันความกำกวมในการอ่านค่าที่ debug print ออกมา —
เพราะ combining mark เป็นอักขระที่ "ต้องพึ่งพา" ตัวอักษรข้างหน้าเพื่อแสดงผลรวมกันเป็นรูปที่อ่านได้ การแสดงมันแบบดิบ ๆ
ในบริบทของการ debug ค่าอาจทำให้สับสนว่าอักขระที่แสดงคืออะไรกันแน่ ในขณะที่ `Display` (ที่มีไว้แสดงผลให้มนุษย์อ่านจริง ๆ)
ไม่มีความจำเป็นต้อง escape เพราะเป้าหมายคือให้อ่านออกมาเป็นภาษาธรรมชาติให้ถูกต้องเสมอ

**ข้อคิดเชิงปฏิบัติ**: ปรากฏการณ์นี้ไม่ใช่ปัญหาที่ต้อง "แก้" อะไร — ถ้าต้องการแสดงข้อความไทยให้มนุษย์อ่าน ให้ใช้ `{}`
(Display) เสมอ ส่วน `{:?}`/`{:#?}` (Debug) มีไว้สำหรับ debug ระหว่างพัฒนาเป็นหลัก ซึ่งบางครั้งก็แสดง escape sequence
แบบนี้ได้เป็นเรื่องปกติ — ไม่ได้แปลว่าข้อมูลเสียหายหรือ encoding ผิดพลาดแต่อย่างใด

#### `#[derive(Clone)]` และ `#[derive(PartialEq)]`

นอกจาก `Debug` แล้ว derive macro ที่ใช้บ่อยรองลงมาคือ `Clone` และ `PartialEq` — ทั้งสองตัวคือ **trait** ที่เราจะเจาะลึก
ความหมายเต็มรูปแบบใน **Part 19** ตอนนี้ให้รู้จักผลลัพธ์ที่ derive ให้ก่อนเป็นเบื้องต้น:

```rust
// Debug ไม่ใช่ derive macro ตัวเดียวที่มี — ที่ใช้บ่อยรองลงมาคือ Clone และ PartialEq
// (ทั้งสองตัวคือ "trait" ที่เราจะเจาะลึกความหมายเต็มรูปแบบใน Part 19 ตอนนี้รู้จักแค่ผลลัพธ์ที่ derive ให้ก่อน)
#[derive(Debug, Clone, PartialEq)]
struct Product {
    name: String,
    price: f64,
    quantity: u32,
}

fn main() {
    let item = Product {
        name: String::from("USB-C cable"),
        price: 199.0,
        quantity: 25,
    };

    // derive(Clone) ทำให้เรียก .clone() เพื่อสร้าง instance ใหม่ที่เป็น "สำเนาลึก" ได้
    // (ทวน concept clone จาก Part 6 — struct ทั้งก้อนถูก clone ไปทุก field รวมถึง String ข้างในด้วย)
    let copy = item.clone();

    // derive(PartialEq) ทำให้เทียบ struct สองตัวด้วย == ได้ (เทียบทุก field ว่าตรงกันหมดหรือไม่)
    println!("item และ copy มีค่าเท่ากันหรือไม่: {}", item == copy);

    let different = Product {
        name: String::from("USB-C cable"),
        price: 249.0, // ราคาต่างกัน
        quantity: 25,
    };
    println!("item และ different เท่ากันหรือไม่: {}", item == different);
}
```

ผลลัพธ์:

```
item และ copy มีค่าเท่ากันหรือไม่: true
item และ different เท่ากันหรือไม่: false
```

- **`#[derive(Clone)]`** — เพิ่ม method `.clone()` ให้ struct โดยอัตโนมัติ ซึ่งจะเรียก `.clone()` ของทุก field
  ภายในตามลำดับ (deep clone) ทำได้ก็ต่อเมื่อทุก field implement `Clone` เช่นเดียวกัน (`String` implement `Clone` อยู่แล้ว
  จึงไม่มีปัญหาในตัวอย่างนี้) — นี่คือการนำ concept `.clone()` ที่เรียนกับ `String` เดี่ยว ๆ ใน Part 6 มาขยายผลกับ struct
  ทั้งก้อน
- **`#[derive(PartialEq)]`** — เพิ่มความสามารถเทียบ `==`/`!=` ให้ struct โดยอัตโนมัติ นิยาม "เท่ากัน" ที่ derive ให้คือ
  **ทุก field ต้องเท่ากันหมดทุกตัว** (compare แบบ field-by-field) ถ้า field ใดต่างกันแม้แต่ตัวเดียว ทั้ง struct จะถือว่า
  ไม่เท่ากันทันที เหมือนที่เห็นในตัวอย่าง `different` ที่ราคาต่างกันเพียงอย่างเดียวก็ทำให้ผลลัพธ์เป็น `false` แล้ว

ทั้ง `Clone` และ `PartialEq` (รวมถึง `Debug`) คือตัวอย่างของสิ่งที่เรียกว่า **trait** ที่ Rust standard library เตรียม
"สูตรสำเร็จ" ในการ implement ให้อัตโนมัติผ่าน `derive` เมื่อโครงสร้างข้อมูลเรียบง่ายพอ (คือแค่ทำแบบเดียวกันกับทุก field
ตามลำดับ) แต่เมื่อต้องการ logic ที่ซับซ้อนกว่านั้น (เช่น เทียบ `Product` สองตัวว่า "เหมือนกัน" แค่ตอนที่ `name` ตรงกัน
โดยไม่สนใจราคา/จำนวน) คุณสามารถเขียน `impl PartialEq for Product { ... }` ด้วยมือเองแทนได้ — รายละเอียดวิธีเขียน trait
implementation แบบเต็มจะเรียนใน Part 19

### 9.13 Method Chaining: กลิ่นอายของ Builder Pattern

เมื่อ struct มี method ที่รับ `&mut self` และ **คืนค่า `&mut Self` กลับออกไปด้วย** เราจะสามารถ **เรียก method ต่อ ๆ กัน
เป็นสาย (chain)** ในบรรทัดเดียวได้ — เทคนิคนี้เรียกว่า **method chaining**:

```rust
struct Product {
    name: String,
    price: f64,
    quantity: u32,
    is_featured: bool,
}

impl Product {
    fn new(name: &str, price: f64) -> Self {
        Self {
            name: name.to_string(),
            price,
            quantity: 0,
            is_featured: false,
        }
    }

    // เทคนิคสำคัญของ method chaining: รับ &mut self แต่ "คืนค่า &mut Self กลับออกไปด้วย"
    // ทำให้เรียก method ต่อ ๆ กันเป็นสายได้ในบรรทัดเดียว โดยไม่ต้องเก็บตัวแปรกลางทาง
    fn with_quantity(&mut self, quantity: u32) -> &mut Self {
        self.quantity = quantity;
        self // คืน reference กลับไปยังตัวเอง เพื่อให้ chain ต่อ .with_xxx() อีกได้
    }

    fn mark_featured(&mut self) -> &mut Self {
        self.is_featured = true;
        self
    }

    fn describe(&self) -> String {
        format!(
            "{}{} — {:.2} บาท x {}",
            if self.is_featured { "⭐ " } else { "" },
            self.name,
            self.price,
            self.quantity
        )
    }
}

fn main() {
    let mut item = Product::new("หูฟังไร้สาย", 1290.0);

    // chain เรียก .with_quantity(...) ต่อด้วย .mark_featured() ในสายเดียว
    // เพราะแต่ละ method คืน &mut Self ทำให้ . ต่อไปเรื่อย ๆ ได้เหมือนอ่านเป็นประโยคเดียว
    item.with_quantity(20).mark_featured();

    println!("{}", item.describe());

    // นี่เป็นแค่ "รสชาติ" ของแนวคิด method chaining เท่านั้น รูปแบบที่สมบูรณ์และปลอดภัยกว่านี้
    // (เช่น การสร้าง instance ใหม่ทุกขั้นตอนแทนการ mutate ของเดิม เพื่อรองรับการ validate
    // ก่อนคืนค่าจริงตอนสุดท้าย) คือ "Builder pattern" ซึ่งเราจะเจาะลึกแบบเต็มรูปแบบใน Part 52
}
```

ผลลัพธ์:

```
⭐ หูฟังไร้สาย — 1290.00 บาท x 20
```

หลักการเบื้องหลัง method chaining ง่ายมาก: `item.with_quantity(20)` เรียก method ที่คืนค่าเป็น `&mut Product` (ชี้กลับไป
ยัง `item` ตัวเดิม) ทำให้เขียน `.mark_featured()` ต่อท้ายได้ทันที เพราะสิ่งที่อยู่ก่อน `.mark_featured()` ก็คือ
`&mut Product` เหมือนกับที่ `item` เองก็เป็น (ผ่านการ auto-deref ของ Rust ที่ยอมให้เรียก method ผ่าน reference ได้
โดยตรงเหมือนเรียกผ่าน instance เอง) การอ่านโค้ดแบบ chain นี้มีข้อดีเชิง **ความอ่านง่ายแบบเรียงลำดับขั้นตอน** เหมือนอ่าน
เป็นประโยคภาษาธรรมชาติ ("สร้างสินค้า → กำหนดจำนวน 20 → ทำเครื่องหมายว่าเป็นสินค้าแนะนำ") โดยไม่ต้องสร้างตัวแปรกลางทาง
ที่ไม่มีความหมายอะไรเลยหลายตัว

อย่างไรก็ตาม ตัวอย่างข้างบนนี้เป็นแค่ **"รสชาติ" เริ่มต้น** ของแนวคิดนี้เท่านั้น รูปแบบที่สมบูรณ์และนิยมใช้ในโค้ด Rust
production จริงที่เรียกว่า **Builder pattern** จะต่างออกไปเล็กน้อยในรายละเอียดสำคัญ เช่น:

- มักสร้าง instance ใหม่ทีละขั้น (แทนการ mutate instance เดิมในที่เดิม) เพื่อให้แต่ละ method คืนค่าเป็น `Self` (ไม่ใช่
  `&mut Self`) ทำให้ chain แบบ "consuming" ได้ ซึ่งปลอดภัยกว่าในหลายสถานการณ์
- มี method สุดท้าย (มักชื่อ `.build()`) ที่ทำการ validate ค่าทั้งหมดก่อนคืน instance จริงออกมา (อาจคืนเป็น `Result`
  เพื่อรองรับกรณี validate ไม่ผ่าน)
- ใช้แยก struct "builder" ออกจาก struct จริงที่ใช้งาน เพื่อให้ field ระหว่างสร้างเป็น `Option<T>` ได้ (ยังไม่ต้องกำหนด
  ค่าครบทุก field ตอนเริ่มสร้าง) แล้วค่อย validate ให้ครบก่อน `.build()` เสร็จสมบูรณ์

รายละเอียดทั้งหมดนี้ต้องใช้ความรู้เรื่อง `Option<T>` (Part 11) และ `Result<T, E>` (Part 12) ประกอบด้วย เราจะกลับมาเจาะลึก
Builder pattern แบบเต็มรูปแบบพร้อมตัวอย่างจริงจังใน **Part 52** ตอนนี้แค่รู้จัก "กลไกพื้นฐาน" ที่ทำให้ method chaining
เป็นไปได้ก็เพียงพอแล้ว

### 9.14 ตัวอย่างจริง: ระบบ Inventory และ Order

มาถึงจุดที่รวมทุกแนวคิดของบทนี้เข้าด้วยกัน ผ่านตัวอย่างที่ใกล้เคียงระบบจริง — ระบบจัดการคลังสินค้า (`Product`) และ
คำสั่งซื้อ (`Order`/`OrderLine`) ที่ทำงานร่วมกัน 2 struct โดยตั้งใจออกแบบให้ใช้เฉพาะแนวคิดที่เรียนมาแล้วเท่านั้น (struct,
impl, method ทั้ง 3 แบบ, associated function, array และ for loop จาก Part 3-4, ownership/borrowing จาก Part 6-7)
โดยยังไม่ใช้ `Vec<T>` หรือ `Option<T>` ที่ยังไม่ได้เรียนอย่างเป็นทางการ (จะเรียนใน Part 11 และ Part 13):

```rust
// ===== struct ที่ 1: Product — สินค้าหนึ่งตัวในคลัง =====
#[derive(Debug, Clone)]
struct Product {
    id: u32,
    name: String,
    price_cents: u64, // เก็บเป็นสตางค์แบบ integer เพื่อเลี่ยงปัญหาความไม่เที่ยงตรงของ float (ทวนจาก Part 3)
    quantity: u32,
}

impl Product {
    // associated function เป็น constructor หลัก — รับราคาเป็น "บาท" (f64) ที่มนุษย์อ่านง่าย
    // แล้วแปลงเป็นสตางค์ (u64) เก็บไว้ภายในให้ทันที
    fn new(id: u32, name: &str, price_baht: f64, quantity: u32) -> Self {
        Self {
            id,
            name: name.to_string(),
            price_cents: (price_baht * 100.0).round() as u64,
            quantity,
        }
    }

    fn price_baht(&self) -> f64 {
        self.price_cents as f64 / 100.0
    }

    fn total_value_cents(&self) -> u64 {
        self.price_cents * self.quantity as u64
    }

    fn is_in_stock(&self) -> bool {
        self.quantity > 0
    }

    // &mut self: แก้ไข quantity ของ instance เดิมโดยตรง ไม่ต้องคืนค่าอะไร
    fn restock(&mut self, additional: u32) {
        self.quantity += additional;
    }

    // &mut self ที่คืนค่า bool บอกผลลัพธ์: ลดสต็อกสำเร็จหรือไม่ (ป้องกัน quantity ติดลบ
    // เพราะ quantity เป็น u32 — ถ้าลบเกินจะ panic ทันทีตอน debug build ตามกฎ overflow จาก Part 3)
    fn try_reduce_stock(&mut self, amount: u32) -> bool {
        if self.quantity >= amount {
            self.quantity -= amount;
            true
        } else {
            false
        }
    }

    fn describe(&self) -> String {
        format!(
            "[#{:03}] {:<20} {:>8.2} บาท  คงเหลือ {:>4} ชิ้น",
            self.id,
            self.name,
            self.price_baht(),
            self.quantity
        )
    }
}

// ===== struct ที่ 2: OrderLine — หนึ่งบรรทัดในคำสั่งซื้อ (อ้างอิงสินค้าด้วย "ตำแหน่งใน catalog") =====
struct OrderLine {
    product_index: usize,
    quantity_ordered: u32,
}

impl OrderLine {
    fn new(product_index: usize, quantity_ordered: u32) -> Self {
        Self {
            product_index,
            quantity_ordered,
        }
    }
}

// ===== struct ที่ 3: Order — คำสั่งซื้อที่ประกอบด้วยหลาย OrderLine =====
struct Order {
    order_id: u32,
    lines: [OrderLine; 2],
}

impl Order {
    fn new(order_id: u32, lines: [OrderLine; 2]) -> Self {
        Self { order_id, lines }
    }

    // ตรวจสอบก่อนว่าทุกบรรทัดมีสต็อกเพียงพอทั้งหมด (all-or-nothing) ก่อนจะเริ่มตัดสต็อกจริง
    // รับ catalog เป็น &mut [Product; 3] — ยืมแบบแก้ไขได้ เพราะต้องเปลี่ยนแปลง quantity ข้างใน
    fn fulfill(&self, catalog: &mut [Product; 3]) -> bool {
        for line in &self.lines {
            if catalog[line.product_index].quantity < line.quantity_ordered {
                return false;
            }
        }

        for line in &self.lines {
            catalog[line.product_index].try_reduce_stock(line.quantity_ordered);
        }

        true
    }

    fn total_cents(&self, catalog: &[Product; 3]) -> u64 {
        let mut total: u64 = 0;
        for line in &self.lines {
            let product = &catalog[line.product_index];
            total += product.price_cents * line.quantity_ordered as u64;
        }
        total
    }
}

fn print_catalog(catalog: &[Product; 3]) {
    println!("=== คลังสินค้าปัจจุบัน ===");
    for product in catalog {
        println!("{}", product.describe());
    }
}

fn main() {
    // สร้าง catalog เริ่มต้นด้วย associated function Product::new
    let mut catalog: [Product; 3] = [
        Product::new(1, "เมาส์ไร้สาย", 299.0, 15),
        Product::new(2, "คีย์บอร์ดเมคานิคอล", 890.0, 8),
        Product::new(3, "แผ่นรองเมาส์เกมมิ่ง", 150.0, 40),
    ];

    print_catalog(&catalog);

    // ตัวอย่างการใช้ &self method ธรรมดา: อ่านมูลค่ารวมและสถานะสต็อกของสินค้าแต่ละตัว
    for product in &catalog {
        let status = if product.is_in_stock() { "มีสต็อก" } else { "หมดสต็อก" };
        println!(
            "  -> {} มูลค่ารวมในคลัง {}.{:02} บาท ({status})",
            product.name,
            product.total_value_cents() / 100,
            product.total_value_cents() % 100
        );
    }

    // เติมสต็อกคีย์บอร์ด (index 1 ใน catalog) อีก 5 ชิ้น
    catalog[1].restock(5);
    println!("\nหลังเติมสต็อกคีย์บอร์ด +5 ชิ้น:");
    println!("{}", catalog[1].describe());

    // สร้างคำสั่งซื้อ: ซื้อเมาส์ไร้สาย (index 0) 3 ชิ้น และแผ่นรองเมาส์ (index 2) 10 ชิ้น
    let order = Order::new(
        1001,
        [OrderLine::new(0, 3), OrderLine::new(2, 10)],
    );

    let order_total = order.total_cents(&catalog);
    println!(
        "\nคำสั่งซื้อ #{}: ยอดรวม {}.{:02} บาท",
        order.order_id,
        order_total / 100,
        order_total % 100
    );

    let fulfilled = order.fulfill(&mut catalog);
    println!("ตัดสต็อกสำเร็จหรือไม่: {fulfilled}");

    println!();
    print_catalog(&catalog);

    // ลองสั่งซื้อเกินสต็อกที่มี (คีย์บอร์ด index 1 เหลือ 13 ชิ้น แต่สั่ง 999 ชิ้น) ควรถูกปฏิเสธทั้งคำสั่งซื้อ
    let impossible_order = Order::new(1002, [OrderLine::new(1, 999), OrderLine::new(0, 1)]);
    let fulfilled_2 = impossible_order.fulfill(&mut catalog);
    println!(
        "\nคำสั่งซื้อ #{} (สั่งเกินสต็อก) ตัดสต็อกสำเร็จหรือไม่: {fulfilled_2}",
        impossible_order.order_id
    );
    println!("สต็อกยังคงเดิมเพราะ all-or-nothing:");
    print_catalog(&catalog);
}
```

ผลลัพธ์เมื่อรัน:

```
=== คลังสินค้าปัจจุบัน ===
[#001] เมาส์ไร้สาย            299.00 บาท  คงเหลือ   15 ชิ้น
[#002] คีย์บอร์ดเมคานิคอล     890.00 บาท  คงเหลือ    8 ชิ้น
[#003] แผ่นรองเมาส์เกมมิ่ง    150.00 บาท  คงเหลือ   40 ชิ้น
  -> เมาส์ไร้สาย มูลค่ารวมในคลัง 4485.00 บาท (มีสต็อก)
  -> คีย์บอร์ดเมคานิคอล มูลค่ารวมในคลัง 7120.00 บาท (มีสต็อก)
  -> แผ่นรองเมาส์เกมมิ่ง มูลค่ารวมในคลัง 6000.00 บาท (มีสต็อก)

หลังเติมสต็อกคีย์บอร์ด +5 ชิ้น:
[#002] คีย์บอร์ดเมคานิคอล     890.00 บาท  คงเหลือ   13 ชิ้น

คำสั่งซื้อ #1001: ยอดรวม 2397.00 บาท
ตัดสต็อกสำเร็จหรือไม่: true

=== คลังสินค้าปัจจุบัน ===
[#001] เมาส์ไร้สาย            299.00 บาท  คงเหลือ   12 ชิ้น
[#002] คีย์บอร์ดเมคานิคอล     890.00 บาท  คงเหลือ   13 ชิ้น
[#003] แผ่นรองเมาส์เกมมิ่ง    150.00 บาท  คงเหลือ   30 ชิ้น

คำสั่งซื้อ #1002 (สั่งเกินสต็อก) ตัดสต็อกสำเร็จหรือไม่: false
สต็อกยังคงเดิมเพราะ all-or-nothing:
=== คลังสินค้าปัจจุบัน ===
[#001] เมาส์ไร้สาย            299.00 บาท  คงเหลือ   12 ชิ้น
[#002] คีย์บอร์ดเมคานิคอล     890.00 บาท  คงเหลือ   13 ชิ้น
[#003] แผ่นรองเมาส์เกมมิ่ง    150.00 บาท  คงเหลือ   30 ชิ้น
```

**สิ่งที่ตัวอย่างนี้รวมไว้จากทั้งบท**:

- **`Product`** เป็น struct หลักที่เก็บข้อมูลสินค้า (owned field ทั้งหมด, ไม่มี reference เลยตามคำแนะนำในหัวข้อ 9.5)
  พร้อม `#[derive(Debug, Clone)]` เพื่อให้ debug print และ clone ได้ในอนาคตถ้าจำเป็น
- **Associated function `Product::new(...)`** เป็น constructor ที่แปลง "บาท" (หน่วยที่มนุษย์อ่านง่าย) เป็น "สตางค์"
  (หน่วยที่คำนวณแม่นยำ) ให้ตั้งแต่จุดสร้าง — ผู้ใช้ struct นี้ไม่ต้องกังวลเรื่องการแปลงหน่วยเองเลย
- **Method ทั้ง 3 แบบของ `impl Product`**: `&self` (`price_baht`, `total_value_cents`, `is_in_stock`, `describe`)
  สำหรับอ่านข้อมูล, `&mut self` (`restock`, `try_reduce_stock`) สำหรับแก้ไขสต็อก — สังเกตว่าเราไม่ได้ใช้ `self`
  (consuming) เลยใน `Product` เพราะไม่มีสถานการณ์ที่ต้อง "แปลงร่าง" สินค้าไปเป็นอย่างอื่นแบบถาวรในตัวอย่างนี้
  (แต่ก็ยังใช้ในหัวข้อ 9.10 ให้เห็นภาพไปแล้ว)
- **`OrderLine`** เป็น struct ที่สองซึ่ง**ไม่ได้เก็บ `Product` ไว้ตรง ๆ** แต่เก็บแค่ `product_index: usize` (ตำแหน่งใน
  array ของ catalog) — การออกแบบแบบนี้หลีกเลี่ยงปัญหาการเก็บ reference ข้าม struct (ซึ่งจะโยงไปเรื่อง lifetime ที่ยังไม่
  ได้เรียนอย่างเป็นทางการ) และยังหลีกเลี่ยงการต้อง clone `Product` ทั้งก้อนมาเก็บซ้ำในทุกคำสั่งซื้อด้วย
- **`Order`** เป็น struct ที่สามที่ **ทำงานร่วมกับทั้งสอง struct ก่อนหน้า** ผ่าน method `fulfill(&self, catalog: &mut
  [Product; 3])` และ `total_cents(&self, catalog: &[Product; 3])` — สังเกตว่า method เหล่านี้รับ catalog เป็น
  parameter แยก (ไม่ได้เก็บไว้เป็น field ของ `Order`) ซึ่งตรงกับกฎ ownership ที่เรียนมา: `Order` ไม่ได้ "เป็นเจ้าของ"
  catalog เลย มันแค่ "ยืม" มาใช้ชั่วคราวตอนประมวลผลเท่านั้น
- **การตรวจสอบแบบ all-or-nothing** ใน `fulfill()` — วน loop ตรวจสอบสต็อกให้ครบทุกบรรทัดก่อน แล้วค่อยวน loop ตัดสต็อก
  จริงอีกรอบ เพื่อไม่ให้เกิดสถานการณ์ที่ตัดสต็อกไปแล้วครึ่งทางแล้วค่อยพบว่าอีกบรรทัดสต็อกไม่พอ (ซึ่งจะทำให้ระบบอยู่ใน
  สถานะครึ่ง ๆ กลาง ๆ ที่ไม่ตรงกับความเป็นจริงทางธุรกิจ) — นี่คือตัวอย่างของการออกแบบ logic ให้ **ปลอดภัยเชิงธุรกิจ**
  ไม่ใช่แค่ปลอดภัยเชิง memory เพียงอย่างเดียว

ตัวอย่างนี้แสดงให้เห็นว่า struct และ `impl` ไม่ใช่แค่ "ที่เก็บข้อมูล" แต่เป็นหน่วยพื้นฐานที่ทำให้เราออกแบบ **domain model**
(แบบจำลองของโลกจริงในโปรแกรม) ที่มีทั้งข้อมูลและพฤติกรรมประกอบกันอย่างเป็นระบบ — เป็นก้าวสำคัญจากการเขียนโค้ดที่มีแต่
ตัวแปรเดี่ยว ๆ และฟังก์ชันลอย ๆ ไปสู่การออกแบบซอฟต์แวร์ที่จัดระเบียบและขยายต่อได้ในระยะยาว ซึ่งเราจะนำ struct ไปผสมกับ
enum (Part 10), `Option`/`Result` (Part 11-12), และ collection อย่าง `Vec`/`HashMap` (Part 13-15) เพื่อสร้างระบบที่
สมบูรณ์ขึ้นไปอีกในบทถัด ๆ ไป

## กับดักที่พบบ่อย (Common Pitfalls)

**1. ลืม `#[derive(Debug)]` แล้วพยายามใช้ `{:?}`**

```rust
struct Product {
    name: String,
    price: f64,
}

fn main() {
    let item = Product {
        name: String::from("เมาส์ไร้สาย"),
        price: 299.0,
    };

    println!("{item:?}"); // ลืมใส่ #[derive(Debug)] ไว้บน struct
}
```

```
error[E0277]: `Product` doesn't implement `Debug`
  --> src/main.rs:12:15
   |
12 |     println!("{item:?}"); // ลืมใส่ #[derive(Debug)] ไว้บน struct
   |               ^^^^^^^^ `Product` cannot be formatted using `{:?}` because it doesn't implement `Debug`
   |
   = help: the trait `Debug` is not implemented for `Product`
   = note: add `#[derive(Debug)]` to `Product` or manually `impl Debug for Product`
help: consider annotating `Product` with `#[derive(Debug)]`
   |
 1 + #[derive(Debug)]
 2 | struct Product {
   |
```

**วิธีแก้**: เติม `#[derive(Debug)]` ไว้เหนือบรรทัด `struct` ตามที่ compiler แนะนำ — สังเกตว่านี่คือตัวอย่างที่ดีของ
ปรัชญาการออกแบบ error message ของ Rust: compiler ไม่เพียงบอกว่าอะไรผิด แต่เสนอ patch ที่แก้ได้ตรงจุดให้เลย มือใหม่มักลืม
derive นี้บ่อยเพราะภาษาอื่นจำนวนมาก (เช่น Python ที่ `print(obj)` ใช้ได้กับทุก object โดยไม่ต้องเตรียมอะไร หรือ Java ที่
`Object.toString()` มี default implementation ให้เสมอ) ไม่บังคับให้ "ขอสิทธิ์" การแสดงผลแบบ debug ก่อนใช้งานแบบ Rust

**2. Partial Move: ย้าย field ออกมาแล้วใช้ struct ทั้งก้อนต่อไม่ได้**

```rust
struct Product {
    name: String,
    price: f64,
    quantity: u32,
}

fn print_summary(product: &Product) {
    println!(
        "{} ราคา {:.2} จำนวน {} ชิ้น",
        product.name, product.price, product.quantity
    );
}

fn main() {
    let item = Product {
        name: String::from("เมาส์ไร้สาย"),
        price: 299.0,
        quantity: 3,
    };

    let moved_name = item.name; // ย้าย field name ออกจาก item

    print_summary(&item); // พยายามยืม item ทั้งก้อนอีก — แต่ item ถูก partial move ไปแล้ว

    println!("ชื่อที่ย้ายออกไปแล้ว: {moved_name}");
}
```

```
error[E0382]: borrow of partially moved value: `item`
  --> src/main.rs:22:19
   |
20 |     let moved_name = item.name; // ย้าย field name ออกจาก item
   |                      ---------- value partially moved here
21 |
22 |     print_summary(&item); // พยายามยืม item ทั้งก้อนอีก — แต่ item ถูก partial move ไปแล้ว
   |                   ^^^^^ value borrowed here after partial move
   |
   = note: partial move occurs because `item.name` has type `String`, which does not implement the `Copy` trait
```

**วิธีแก้**: ถ้าแค่ต้องการอ่านค่า field ให้ยืมด้วย `&item.name` แทนการ assign ตรง ๆ (`let moved_name = item.name;`)
เสมอ ยกเว้นในกรณีที่ตั้งใจจริง ๆ ว่าจะไม่ใช้ `item` ทั้งก้อนอีกต่อไป (ดูรายละเอียดเต็มในหัวข้อ 9.6)

**3. ลืมใส่ field ให้ครบตอนสร้าง struct literal**

```rust
struct Product {
    name: String,
    price: f64,
    quantity: u32,
}

fn main() {
    let item = Product {
        name: String::from("เมาส์ไร้สาย"),
        price: 299.0,
        // ลืมใส่ quantity ไปเลย
    };

    println!("{}", item.name);
}
```

```
error[E0063]: missing field `quantity` in initializer of `Product`
 --> src/main.rs:8:16
  |
8 |     let item = Product {
  |                ^^^^^^^ missing `quantity`
```

**วิธีแก้**: ระบุค่าให้ครบทุก field เสมอ — Rust ไม่มีแนวคิด "ค่า default โดยปริยาย" สำหรับ field ที่ไม่ได้ระบุแบบภาษา
อื่น (เช่น Python ที่ attribute ที่ไม่ได้กำหนดจะเป็น `None` โดยปริยาย) การบังคับให้ระบุครบทุก field ตั้งแต่ compile time
ป้องกันบั๊กจากการลืมกำหนดค่าที่สำคัญไปโดยไม่ตั้งใจได้ตั้งแต่ต้น ถ้าต้องการ field ที่มีค่า default จริง ๆ ให้พิจารณาใช้
trait `Default` (จะพูดถึงในบทหลัง) หรือเขียน associated function constructor ที่กำหนดค่า default ให้ชัดเจนในโค้ด
เหมือนที่ทำกับ `Product::new_out_of_stock` ในหัวข้อ 9.11

**4. เก็บ `&str` เป็น field โดยไม่รู้ว่าต้องมี lifetime annotation**

```rust
struct Product {
    name: &str,
    price: f64,
}

fn main() {
    let product = Product {
        name: "เมาส์ไร้สาย",
        price: 299.0,
    };
    println!("{}", product.name);
}
```

```
error[E0106]: missing lifetime specifier
 --> src/main.rs:2:11
  |
2 |     name: &str,
  |           ^ expected named lifetime parameter
  |
help: consider introducing a named lifetime parameter
  |
1 ~ struct Product<'a> {
2 ~     name: &'a str,
  |
```

**วิธีแก้ที่แนะนำสำหรับตอนนี้**: เปลี่ยน field ให้เป็น `String` (owned type) แทน `&str` ไปก่อน เพื่อหลีกเลี่ยงความซับซ้อน
เรื่อง lifetime annotation ที่ยังไม่ได้เรียนอย่างเป็นทางการ (จะเรียนเต็มใน Part 20/23) ถ้าจำเป็นต้องใช้ `&str` จริง ๆ
(เช่นเพื่อประสิทธิภาพในโค้ดที่ optimize แล้ว) ให้เพิ่ม lifetime parameter ตามที่ compiler แนะนำ (ดูรายละเอียดในหัวข้อ 9.5)

**5. เรียก method ที่รับ `&mut self` บน binding ที่ไม่ได้ประกาศด้วย `mut`**

```rust
struct Product {
    name: String,
    quantity: u32,
}

impl Product {
    fn restock(&mut self, additional: u32) {
        self.quantity += additional;
    }
}

fn main() {
    let item = Product {
        // ไม่ได้เขียน mut ไว้ตอนประกาศ item
        name: String::from("เมาส์ไร้สาย"),
        quantity: 10,
    };

    item.restock(5); // พยายามเรียก method ที่รับ &mut self บน binding ที่ไม่ mut
    println!("{}", item.quantity);
}
```

```
error[E0596]: cannot borrow `item` as mutable, as it is not declared as mutable
  --> src/main.rs:19:5
   |
19 |     item.restock(5); // พยายามเรียก method ที่รับ &mut self บน binding ที่ไม่ mut
   |     ^^^^ cannot borrow as mutable
   |
help: consider changing this to be mutable
   |
13 |     let mut item = Product {
   |         +++
```

**วิธีแก้**: เติม `mut` ตอนประกาศตัวแปร (`let mut item = ...`) ตามที่ compiler แนะนำ — error นี้เป็นการนำกฎ `mut`
จาก Part 3 มาใช้กับ struct โดยตรง: การเรียก `&mut self` method เทียบเท่ากับการยืม instance แบบแก้ไขได้ ซึ่งตาม
กฎจาก Part 7 ตัวแปรต้นทางต้องเป็น `mut` binding เสมอ ไม่ว่าจะยืมแบบ `&mut` ตรง ๆ หรือผ่าน method ก็ตาม

**6. ใช้ struct update syntax (`..`) แล้วลืมว่า instance เดิมถูก partial move ไปแล้ว**

```rust
#[derive(Debug)]
struct Product {
    name: String,
    price: f64,
    quantity: u32,
}

fn main() {
    let original = Product {
        name: String::from("เมาส์ไร้สาย"),
        price: 299.0,
        quantity: 15,
    };

    let discounted = Product {
        price: 249.0,
        ..original // ย้าย field name (String) ออกจาก original มาสร้าง discounted
    };

    println!("{original:?}"); // พยายามใช้ original ทั้งตัวอีกครั้ง
    println!("{discounted:?}");
}
```

```
error[E0382]: borrow of partially moved value: `original`
  --> src/main.rs:21:16
   |
15 |       let discounted = Product {
   |  ______________________-
16 | |         price: 249.0,
17 | |         ..original // ย้าย field name (String) ออกจาก original มาสร้าง discounted
18 | |     };
   | |_____- value partially moved here
...
21 |       println!("{original:?}"); // พยายามใช้ original ทั้งตัวอีกครั้ง
   |                  ^^^^^^^^ value borrowed here after partial move
   |
   = note: partial move occurs because `original.name` has type `String`, which does not implement the `Copy` trait
```

**วิธีแก้**: ถ้ายังต้องใช้ `original` ทั้งก้อนต่อหลังสร้าง `discounted` ให้ clone ก่อนใช้ `..` เช่น `..original.clone()`
(ต้อง derive หรือ implement `Clone` ให้ `Product` ก่อน) หรือถ้าไม่จำเป็นต้องใช้ `original` ต่อจริง ๆ ก็ไม่ต้องแก้อะไร
เพราะนี่คือพฤติกรรมที่ถูกต้องตามกฎ ownership อยู่แล้ว — struct update syntax ไม่ใช่ "การ copy แบบพิเศษ" แต่เป็นน้ำตาล
ไวยากรณ์ของการ assign field แต่ละตัว ซึ่งยังต้องเดินตามกฎ move/copy ปกติทุกประการ (ดูรายละเอียดเต็มในหัวข้อ 9.7)

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** แปลงโค้ดที่ใช้ tuple ต่อไปนี้ให้เป็น struct ชื่อ `BankAccount` ที่มี field `owner_name: String`,
   `balance_cents: i64` (ทวนเหตุผลจาก Part 3 ว่าทำไมยอดเงินควรเป็น signed integer ไม่ใช่ float หรือ unsigned)
   แล้วเขียน method `&self` ชื่อ `balance_baht(&self) -> f64` ที่แปลงสตางค์กลับเป็นบาท:
   ```rust
   fn print_account(account: (String, i64)) {
       println!("{}: {} สตางค์", account.0, account.1);
   }

   fn main() {
       let acc = (String::from("สมชาย"), 150_000);
       print_account(acc);
   }
   ```
   (hint: นิยาม `struct BankAccount { owner_name: String, balance_cents: i64 }` แล้วเขียน `impl BankAccount` ที่มี
   `balance_baht(&self) -> f64 { self.balance_cents as f64 / 100.0 }` — ลองเพิ่ม field init shorthand ใน
   associated function `new` ด้วย)

2. **[กลาง]** เขียน associated function `BankAccount::new(owner_name: &str, initial_balance_cents: i64) -> Self`
   และ method 2 ตัว: `deposit(&mut self, amount_cents: i64)` (เพิ่มยอด ไม่มีทางล้มเหลว) และ
   `try_withdraw(&mut self, amount_cents: i64) -> bool` (ลดยอด แต่คืน `false` และไม่แก้ไขยอดถ้าเงินไม่พอ คล้ายกับ
   `try_reduce_stock` ในหัวข้อ 9.14) จากนั้นทดลองสร้างโค้ดที่มี partial move error โดยตั้งใจ (เช่น
   `let name = account.owner_name;` แล้วพยายามเรียก `account.deposit(...)` ต่อ) บันทึก error message ที่ได้
   แล้วแก้ไขด้วยการยืม `&account.owner_name` แทน
   (hint: `try_withdraw` ควรมีโครงคล้าย `if self.balance_cents >= amount_cents { self.balance_cents -= amount_cents;
   true } else { false }`)

3. **[กลาง-ยาก]** ขยายตัวอย่างหัวข้อ 9.14 โดยเพิ่ม struct ที่สามชื่อ `Receipt` ที่มี field `order_id: u32` และ
   `total_baht: f64` เขียน method **แบบ consuming** (`self` ไม่ใช่ `&self`) ชื่อ `into_summary_line(self) -> String`
   บน `Order` ที่รับ `catalog: &[Product; 3]` เป็น parameter เพิ่ม คำนวณยอดรวม แล้ว **ย้าย** `self.order_id` ออกมาใช้
   สร้าง `Receipt` ก่อนคืนค่าเป็น `String` ที่จัดรูปแบบสวยงาม จากนั้นลองอธิบายว่าทำไมในกรณีนี้การใช้ `self` (consuming)
   สมเหตุสมผลกว่า `&self` (hint: เพราะ `Order` ที่ "ออกใบเสร็จไปแล้ว" ไม่ควรถูกนำมาประมวลผลซ้ำอีก — การ consume ทิ้งไป
   เป็นการบังคับด้วย type system ว่า order เดิมใช้ครั้งเดียวแล้วจบ ตรงกับความหมายทางธุรกิจจริง)

4. **[ยาก/ประยุกต์]** ออกแบบระบบ "สมาชิก VIP" เล็ก ๆ ที่รวมแนวคิดต่อไปนี้ให้ครบ: (ก) tuple struct newtype ชื่อ
   `Baht(u64)` สำหรับห่อจำนวนเงินหน่วยสตางค์ ป้องกันการส่งตัวเลขเงินสลับกับตัวเลขอื่นโดยไม่ตั้งใจ (คล้าย `Meters`/`Feet`
   ในหัวข้อ 9.8), (ข) unit-like struct ชื่อ `VipTier;` ที่ใช้เป็น marker แยกแยะ "ลูกค้าทั่วไป" จาก "ลูกค้า VIP" (คล้าย
   `GuestUser` ในหัวข้อ 9.9), (ค) struct `Customer` ที่มี method chaining แบบหัวข้อ 9.13 อย่างน้อย 2 method ที่คืน
   `&mut Self` (เช่น `.add_points(...)` และ `.upgrade_to_vip(...)`) จากนั้นเขียน `main()` ที่สร้างลูกค้าหลายคน คำนวณ
   ยอดซื้อสะสมรวมเป็น `Baht`, และพิมพ์สรุปด้วย `#[derive(Debug)]`
   (hint: ฟังก์ชันที่รับเงินควรรับ parameter เป็น `Baht` เสมอ ไม่ใช่ `u64` ตรง ๆ เพื่อให้ compiler ช่วยจับบั๊กเรื่องส่ง
   ค่าผิดความหมายให้ ลองเปรียบเทียบว่าถ้าใช้ `u64` ตรง ๆ ทั้งหมด จะมีจุดไหนที่ compiler ไม่ช่วยจับบั๊กให้บ้าง)

## สรุป

ในบทนี้เราก้าวข้ามจากการใช้ชนิดข้อมูลสำเร็จรูป (scalar, tuple, array, `String`, `&str`, slice) ไปสู่การ **สร้างชนิด
ข้อมูลของตัวเอง** เป็นครั้งแรก เราเริ่มจากปัญหาจริงของ tuple ที่ไม่มีชื่อผูกกับความหมายของแต่ละค่า ทำให้ **struct**
แบบมีชื่อ field เข้ามาแก้ปัญหานี้อย่างตรงจุด พร้อม field init shorthand ที่ช่วยลดความซ้ำซ้อนของโค้ด

เราเจาะลึกว่า struct ผูกกับแนวคิด **ownership** จาก Part 6 อย่างแน่นแฟ้น — struct เป็นเจ้าของ field ของมันเองโดยตรง
ทำให้อายุของทุก field ผูกติดกับอายุของ struct โดยอัตโนมัติ, การเก็บ reference (`&str`) ไว้ใน struct ต้องมี **lifetime
annotation** เพื่อบอก compiler ว่า reference นั้นต้องมีชีวิตอยู่ไม่สั้นกว่า struct เอง (รายละเอียดเต็มรอไว้ Part 20/23),
และการ **partial move** field ใดก็ตามที่ไม่ implement `Copy` จะทำให้ struct ทั้งก้อนยืม (borrow) ต่อไม่ได้อีก ซึ่งแก้ได้
ด้วยการยืม `&field` แทนการย้ายค่าตรง ๆ เสมอเมื่อแค่ต้องการอ่าน

เรารู้จัก **struct update syntax** (`..instance`) สำหรับสร้าง variant ใหม่จาก instance เดิมอย่างกระชับ, **tuple struct**
สำหรับ wrapper แบบเบาและ newtype pattern (ที่ compiler ช่วยแยกแยะความหมายของ type ที่ shape เหมือนกันแต่ความหมายต่างกัน
เต็มรูปแบบใน Part 53), และ **unit-like struct** สำหรับ marker type ที่ไม่ต้องเก็บข้อมูลเลย (นำไปใช้เต็มรูปแบบกับ trait
ใน Part 19)

หัวใจสำคัญที่สุดของบทนี้คือ **`impl` block** และการเลือก receiver ของ method ให้ถูกต้อง: `&self` สำหรับอ่านข้อมูล
(ตัวเลือกแรกเสมอ), `&mut self` สำหรับแก้ไขข้อมูลในที่เดิม, และ `self` สำหรับแปลงร่าง instance ไปเป็นสิ่งอื่นแบบถาวร
(consuming) พร้อม **associated function** ตาม convention `new(...)` สำหรับเป็น constructor, `#[derive(Debug)]` สำหรับ
debug print (พร้อมข้อสังเกตเรื่องการ escape combining mark ในข้อความไทย), และรสชาติเริ่มต้นของ **method chaining**
ที่จะขยายเป็น Builder pattern เต็มรูปแบบใน Part 52 ทั้งหมดนี้ถูกรวบรวมไว้ในตัวอย่างระบบ Inventory/Order ท้ายบท ซึ่งแสดง
ให้เห็นว่า struct หลายตัวทำงานร่วมกันเป็น domain model ที่ใช้งานได้จริงได้อย่างไร

ใน **Part 10** เราจะเรียนรู้จัก **Enum และ Pattern Matching** — ชนิดข้อมูลอีกแบบที่ต่างจาก struct ตรงที่แทน "หนึ่งใน
หลายความเป็นไปได้" (แทนที่จะเป็น "ทุก field รวมกัน" แบบ struct) ผ่าน `match`, `if let`, `while let` ซึ่งเมื่อรวมกับ
struct ที่เรียนในบทนี้แล้ว จะทำให้เราออกแบบชนิดข้อมูลที่ซับซ้อนและตรงกับโลกจริงได้ครบทุกรูปแบบที่ภาษาโปรแกรมมิ่งทั่วไป
รองรับ และเป็นพื้นฐานสำคัญก่อนจะเรียน `Option<T>` ใน Part 11 (ซึ่งที่จริงแล้วก็คือ enum ชนิดหนึ่งที่ Rust ใช้แทนค่า
"อาจมีหรือไม่มี" โดยไม่ต้องมี null เลย)

---

**Part ก่อนหน้า:** [Slices](part-008-slices.md) | **Part ถัดไป:** [Enums และ Pattern Matching](part-010-enums-and-pattern-matching.md)
