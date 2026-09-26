# Part 24: Closures (Fn, FnMut, FnOnce, capturing)

> โมดูล: ระดับกลาง (Intermediate) | ระดับ: กลาง | เวลาโดยประมาณ: 200 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้ว่า closure คืออะไรในเชิง type system จริง ๆ (ไม่ใช่แค่ "ฟังก์ชันสั้น ๆ") และเขียน closure ได้ทั้งสามรูปแบบ
  ไวยากรณ์ (`|x| x + 1`, `|x: i32| -> i32 { x + 1 }`, และ closure หลายบรรทัดที่มี block body) พร้อมเข้าใจว่า
  compiler infer type ของ parameter/return ให้เมื่อไหร่ และทำไมมันถึง "ล็อก" type นั้นไว้ทันทีหลังใช้งานครั้งแรก
- อธิบายความแตกต่างพื้นฐานที่สุดระหว่าง closure กับ function ธรรมดา (`fn`) ได้อย่างแม่นยำ: closure **capture**
  ตัวแปรจาก environment รอบตัวมันได้ ในขณะที่ `fn` ทำไม่ได้เลยไม่ว่ากรณีใด
- แยกแยะ trait `Fn`, `FnMut`, และ `FnOnce` ได้ทั้งในเชิงความหมาย (capture by reference / by mutable reference /
  by value) และรู้ว่า **compiler เป็นคนตัดสินใจให้เองจากการใช้งานจริงในตัว closure** ไม่ใช่สิ่งที่เราต้องประกาศ
- ใช้ `move` keyword ได้อย่างถูกจุด เข้าใจว่าทำไมมันจำเป็นเมื่อ closure ต้อง "มีชีวิตอยู่ต่อ" นานกว่า scope ที่มัน
  ถูกสร้างขึ้นมา (เช่น return closure ออกจากฟังก์ชัน หรือส่งเข้า thread ใหม่ที่จะเรียนเต็มรูปแบบใน Part 37)
- เลือกวิธีรับ closure เป็น parameter ได้ถูกต้องตามสถานการณ์ (generic `<F: Fn(...)>`, `impl Fn(...)`, หรือ
  `&dyn Fn(...)`) และเลือกวิธี return closure ออกจากฟังก์ชันได้ถูกต้อง (`impl Fn(...)` เมื่อ return type เดียว
  เสมอ กับ `Box<dyn Fn(...)>` เมื่อ return หลาย concrete type ได้)
- เข้าใจ mental model ว่า closure ที่ capture ตัวแปรจริง ๆ แล้วถูก compiler แปลงเป็น struct ที่ไม่มีชื่อ (anonymous
  struct) เก็บตัวแปรที่ capture ไว้เป็น field แล้ว implement trait ที่เหมาะสมให้ — ทำให้ closure ไม่ใช่เรื่องมายากล
  อีกต่อไป แต่เป็นน้ำตาลไวยากรณ์ (syntax sugar) บน struct + trait ที่เรารู้จักมาแล้วตั้งแต่ Part 9 และ 19
- นำความรู้ทั้งหมดมาประยุกต์เขียนระบบ "discount rule engine" ที่เก็บ closure หลายตัวไว้ใน `Vec<Box<dyn Fn(&Order)
  -> f64>>` แต่ละตัว capture ค่า config ของตัวเอง (เปอร์เซ็นต์ส่วนลด, เกณฑ์ขั้นต่ำ) แล้วนำมาคำนวณราคาสุทธิร่วมกัน

## ความรู้ที่ต้องมีมาก่อน

- **Part 6 (Ownership เบื้องต้น)** และ **Part 7 (Borrowing และ References)**: หัวใจของบทนี้คือการเข้าใจว่า closure
  "ยืม" (`&T`), "ยืมแบบแก้ไขได้" (`&mut T`), หรือ "ยึด ownership" (`T` ตรง ๆ ผ่านการ move) ตัวแปรที่ capture มาแบบ
  ไหน — ถ้ายังไม่แน่นเรื่อง move/borrow/borrow checker จาก Part 6-7 เนื้อหาเรื่อง `Fn`/`FnMut`/`FnOnce` และ `move`
  keyword ในบทนี้จะเข้าใจยากมาก เพราะมันคือการนำกฎ ownership แบบเดียวกันมาใช้กับตัวแปรที่ closure "ขอยืม" จาก
  environment รอบตัวมันเอง
- **Part 11 (Option<T> และ Null Safety)**: บทนั้นแนะนำ closure ไว้แบบไม่เป็นทางการแล้วผ่าน `.map(|x| ...)`,
  `.unwrap_or_else(|| ...)`, `.and_then(|x| ...)`, `.filter(|&x| ...)` และบอกตรง ๆ ว่าจะเก็บรายละเอียดเรื่อง
  capture/`Fn`/`FnMut`/`FnOnce` ไว้ให้บทนี้ — ถ้าคุณยังไม่คุ้นกับการอ่าน syntax `|x| expression` แนะนำให้กลับไป
  ทวนหัวข้อ 11.6 ก่อน เพราะบทนี้จะไม่สอนไวยากรณ์พื้นฐานที่สุดซ้ำอีก (แต่จะเจาะลึกกลไกที่อยู่ข้างใต้มันเต็มรูปแบบ)
  Part 13 (`Vec::sort_by`, `sort_by_key`) และ Part 15 (`HashMap::entry().or_insert_with()`) ก็ใช้ closure แบบ
  ไม่เป็นทางการในทำนองเดียวกัน
- **Part 18 (Generics เบื้องต้น)**: การรับ closure เป็น parameter ด้วย generic `<F: Fn(i32) -> i32>` ใช้กลไก
  monomorphization แบบเดียวกับ generic function ทั่วไปที่เรียนมาแล้ว — compiler สร้างโค้ดเฉพาะสำหรับ concrete
  type ของ closure แต่ละตัวที่เรียกจริง (static dispatch)
- **Part 19 (Traits เบื้องต้น)**: บทนั้นสอน `impl Trait` เป็น parameter/return type และ `Box<dyn Trait>` เป็น
  ทางออกเมื่อต้อง return หลาย concrete type แล้ว (หัวข้อ 19.6-19.9) พร้อมเกริ่นเรื่อง static dispatch เทียบกับ
  dynamic dispatch ไว้คร่าว ๆ — บทนี้จะนำแนวคิดเดียวกันมาใช้กับ `Fn`/`FnMut`/`FnOnce` โดยตรง (มันคือ trait ธรรมดา
  ที่ std library กำหนดไว้ ไม่ใช่กลไกพิเศษแยกต่างหาก) และ **Part 21 (Traits ขั้นสูง)** ที่เจาะลึก trait object,
  vtable, และ object safety แบบเต็มรูปแบบ — บทนี้จะอ้างอิงกลไก dynamic dispatch ของ `dyn Trait` จาก Part 21 บ่อย
  เมื่อพูดถึง `Box<dyn Fn(...)>` และ `&dyn Fn(...)`

## เนื้อหา

### 24.1 ทวนความจำ: เราใช้ Closure มาแล้วโดยไม่รู้ตัวเลยตั้งแต่ Part 11

ก่อนเข้าเนื้อหาใหม่ ให้ย้อนกลับไปดูโค้ดที่เราเคยเขียนมาแล้วในบทก่อน ๆ ของหลักสูตรนี้อีกครั้ง — คุณเห็นรูปแบบนี้มาแล้ว
หลายสิบครั้งโดยไม่รู้ตัวว่ามันมีชื่อทางการว่าอะไร:

```rust
fn main() {
    // จาก Part 11 (Option<T> combinators)
    let price: Option<f64> = Some(500.0);
    let price_with_vat = price.map(|p| p * 1.07);
    println!("{price_with_vat:?}");

    let discount: Option<u32> = None;
    let final_discount = discount.unwrap_or_else(|| 5);
    println!("{final_discount}");

    // จาก Part 13 (Vec::sort_by)
    let mut numbers = vec![5, 2, 8, 1, 9];
    numbers.sort_by(|a, b| b.cmp(a)); // เรียงจากมากไปน้อย
    println!("{numbers:?}");
}
```

```
Some(535.0)
5
[9, 8, 5, 2, 1]
```

ทุกอย่างที่อยู่ระหว่าง `|` สองตัวนั้น (`|p| p * 1.07`, `|| 5`, `|a, b| b.cmp(a)`) คือสิ่งที่เรียกว่า **closure**
มาตั้งแต่แรก — Part 11 บอกไว้สั้น ๆ ตอนหัวข้อ 11.6 ว่า "เราจะเรียน closure แบบเต็มรูปแบบใน Part 24" และตอนนี้เรามา
ถึงจุดนั้นแล้ว คำถามที่บทนี้จะตอบให้ครบคือ:

- closure คืออะไรกันแน่ในเชิง **type system** ของ Rust ไม่ใช่แค่ "ไวยากรณ์สั้น ๆ สำหรับเขียนฟังก์ชัน"
- ทำไม closure บางตัวเรียกได้หลายครั้ง (`sort_by` เรียก closure เปรียบเทียบซ้ำ ๆ หลายรอบ) แต่บางตัวเรียกได้แค่
  ครั้งเดียว
- closure "จำ" ตัวแปรจาก scope ที่มันถูกสร้างขึ้นมาได้อย่างไร (เช่นสมมติว่าเราเขียน `|a, b| b.cmp(a)` แบบที่มี
  ตัวแปร `reverse_order: bool` จากภายนอกมาตัดสินใจว่าจะกลับด้านหรือไม่ — closure จะ "เห็น" `reverse_order` ได้
  อย่างไร ทั้งที่มันไม่ได้เป็น parameter ของ closure เลย)
- ทำไมบางครั้งเราต้องเขียน `move ||` แทน `||` เฉย ๆ — และถ้าลืมจะเกิด error แบบไหน

ทั้งหมดนี้คือสิ่งที่ Part 11-15 "แกล้งไม่พูดถึง" เพราะยังไม่ถึงเวลา ตอนนี้เราพร้อมแล้วเพราะมีพื้นฐาน ownership
(Part 6-7), generics (Part 18), และ trait (Part 19, 21) ครบถ้วน — closure ที่จริงแล้วเป็นแค่การนำสามเรื่องนี้มา
รวมกันในรูปแบบที่สวยงามเป็นพิเศษ

### 24.2 Syntax พื้นฐานของ Closure: สามรูปแบบเดียวกัน

closure ใน Rust เขียนได้สามรูปแบบหลัก ที่จริงแล้วมันคือรูปแบบเดียวกันที่ค่อย ๆ เพิ่มรายละเอียดเข้าไป:

```rust
fn main() {
    // รูปแบบที่ 1: parameter และ return type ให้ compiler infer เองทั้งหมด (กระชับที่สุด)
    let add_one = |x| x + 1;

    // รูปแบบที่ 2: ระบุ type ของ parameter และ return type ตรง ๆ (เหมือน fn เต็มรูปแบบ แต่ไม่มีชื่อ)
    let add_one_typed = |x: i32| -> i32 { x + 1 };

    // รูปแบบที่ 3: closure ที่มี body หลายบรรทัด (block expression) ต้องใช้ { } ครอบเสมอ
    let subtract_one_block = |x: i32| -> i32 {
        let result = x - 1;
        result
    };

    println!("add_one(5) = {}", add_one(5));
    println!("add_one_typed(5) = {}", add_one_typed(5));
    println!("subtract_one_block(5) = {}", subtract_one_block(5));
}
```

```
add_one(5) = 6
add_one_typed(5) = 6
subtract_one_block(5) = 4
```

**เทียบกับการประกาศ function ธรรมดาที่คุณคุ้นเคยมาแล้วตั้งแต่ Part 4:**

```rust
fn add_one_fn(x: i32) -> i32 {
    x + 1
}
```

สังเกตความคล้ายกัน — closure คือ "ฟังก์ชันที่ไม่มีชื่อ" (anonymous function) ที่ตัด `fn` และชื่อฟังก์ชันออก แล้ว
เปลี่ยนวงเล็บ parameter `(x: i32)` เป็น pipe `|x: i32|` แทน สังเกตความต่างที่สำคัญอีกจุดคือ closure ที่มี body
เป็น expression เดียว (ไม่ใช่ block) **ไม่ต้องใส่ `{ }` เลย** เช่น `|x| x + 1` ในขณะที่ `fn` เต็มรูปแบบต้องมี `{ }`
เสมอไม่ว่า body จะสั้นแค่ไหนก็ตาม — นี่คือความกระชับที่ทำให้ closure เหมาะกับการเขียน logic สั้น ๆ ตรงจุดที่ต้องใช้
เลย โดยไม่ต้องประกาศฟังก์ชันแยกไว้ที่อื่นแล้วอ้างชื่อกลับมา

รูปแบบที่กระชับที่สุด (`|x| x + 1`) ทำงานได้เพราะ **type inference** ของ Rust ที่เรียนมาตั้งแต่ Part 3 — compiler
พยายามอนุมาน type ของ `x` และ return type จากบริบทที่ closure ถูกใช้งาน หัวข้อถัดไปจะเจาะลึกกฎการ infer type ของ
closure ให้ครบ เพราะมันมีความแตกต่างสำคัญจาก type inference ของตัวแปรธรรมดาที่ต้องระวัง

### 24.3 Type Inference ของ Closure: ทำไม Rust ต้อง "ล็อก" Type หลังเรียกครั้งแรก

นี่คือจุดที่มือใหม่มักแปลกใจที่สุดเมื่อเจอครั้งแรก — closure ที่ไม่ได้ระบุ type ของ parameter ไว้ตรง ๆ (`|x| x * x`)
**ไม่ใช่ generic function** แบบที่เรียนใน Part 18 (ที่รับ type `T` อะไรก็ได้ที่ตรงตาม bound ในการเรียกแต่ละครั้ง)
แต่มันคือ closure ที่ compiler จะ **อนุมาน concrete type เดียวให้ตายตัว จากการเรียกใช้งานครั้งแรกเท่านั้น** แล้วล็อก
type นั้นไว้สำหรับ closure ตัวนั้นตลอดไป

```rust
fn main() {
    let square = |x| x * x;

    let a = square(5); // เรียกครั้งแรกด้วย integer literal -> compiler infer x: i32 (ค่า default ของ integer)
    println!("a = {a}");
}
```

```
a = 25
```

โค้ดข้างบนทำงานได้ปกติ — `square` ถูก infer เป็น closure ที่รับ `i32` คืน `i32` (เพราะ `5` เป็น integer literal
ที่ไม่มี suffix ระบุ type ชัดเจน Rust จะใช้ type default คือ `i32` ตามที่เรียนใน Part 3) แต่ลองดูว่าเกิดอะไรขึ้นถ้า
เราพยายามเรียก `square` ตัวเดิมนี้อีกครั้งด้วย argument ที่เป็น type อื่น:

```rust
fn main() {
    let square = |x| x * x;

    let a = square(5);   // เรียกครั้งแรก -> x ถูกล็อกเป็น i32 จากจุดนี้ไปตลอดชีวิตของ closure ตัวนี้
    let b = square(5.0); // เรียกครั้งที่สอง ด้วย f64 -> ขัดกับ type ที่ล็อกไว้แล้ว!
    println!("{a} {b}");
}
```

```
error[E0308]: mismatched types
 --> src/main.rs:4:20
  |
4 |     let b = square(5.0);
  |             ------ ^^^ expected integer, found floating-point number
  |             |
  |             arguments to this function are incorrect
  |
note: expected because the closure was earlier called with an argument of type `{integer}`
 --> src/main.rs:3:20
  |
3 |     let a = square(5);
  |             ------ ^ expected because this argument is of type `{integer}`
  |             |
  |             in this closure call
note: closure parameter defined here
 --> src/main.rs:2:19
  |
2 |     let square = |x| x * x;
  |                   ^
```

อ่าน error message นี้อย่างละเอียด — compiler บอกตรง ๆ ว่า `expected because the closure was earlier called with
an argument of type` **integer** — นี่คือหลักฐานชัดเจนว่า **การเรียก `square(5)` ครั้งแรกคือตัวกำหนด concrete type
ของ closure ตัวนี้ทั้งหมด** ไม่ใช่แค่ของการเรียกครั้งนั้นครั้งเดียว เมื่อเรียกครั้งที่สองด้วย `5.0` (ซึ่งเป็น
`f64`) ที่ไม่ตรงกับ type ที่ล็อกไว้ compiler จึงปฏิเสธทันทีด้วย E0308 (mismatched types) เหมือนกับที่เราเคยเห็น
มาแล้วหลายครั้งตั้งแต่ Part 11

**เหตุผลเชิงกลไกที่แท้จริงว่าทำไมต้องเป็นแบบนี้**: closure แต่ละตัวใน Rust ไม่ใช่แค่ "โค้ดที่ generic ใช้ type
ไหนก็ได้" — เมื่อ compiler เห็น `|x| x * x` มันจะสร้าง **concrete type ที่ไม่มีชื่อ (anonymous, unique type)
ขึ้นมาหนึ่งตัวโดยเฉพาะสำหรับ closure ตัวนี้เท่านั้น** (เราจะเห็น mental model ที่อธิบายเรื่องนี้แบบละเอียดใน
หัวข้อ 24.9) type ที่ไม่มีชื่อนี้ต้องมี signature ที่ตายตัว — เหมือนกับ `fn` ธรรมดาที่ signature ต้องเป็น
`fn(i32) -> i32` แน่นอนหนึ่งแบบ ไม่ใช่ `fn(T) -> T` ที่ยัง generic อยู่ ต่างจาก generic function ที่คุณเรียนใน
Part 18 ซึ่งแต่ละครั้งที่เรียกด้วย type ต่างกัน compiler จะ **monomorphize** สร้างฟังก์ชันคนละตัวขึ้นมาใหม่
(เช่น `largest::<i32>` กับ `largest::<f64>` เป็นสองฟังก์ชันคนละตัวกันจริง ๆ ใน binary) แต่ closure ตัวเดียวกัน
(`square` ในที่นี้) มันคือ**ค่าตัวเดียว ที่ต้องมี type เดียวเท่านั้น** — มันจึงไม่มีทาง "generic ข้ามการเรียก"
ได้แบบ generic function เพราะ `square` เป็นตัวแปรตัวหนึ่งที่ถูกสร้างขึ้นมาครั้งเดียว ไม่ใช่ template ที่จะถูก
สร้างใหม่ทุกครั้งที่เรียก

ถ้าต้องการให้ทำงานได้กับหลาย type จริง ๆ ต้องเขียนเป็น generic function ที่รับ closure เป็น parameter (ตามที่
จะเรียนในหัวข้อ 24.7) หรือเขียน closure สองตัวแยกกันคนละ type ไปเลย — closure เดียวไม่สามารถ "เปลี่ยน type ตาม
บริบทของการเรียกแต่ละครั้ง" ได้เหมือน generic function

### 24.4 Closure ทำสิ่งที่ Function ทำไม่ได้: การ Capture ตัวแปรจาก Environment

ตอนนี้เรารู้ syntax ของ closure แล้ว มาถึงคำถามที่สำคัญที่สุดของบทนี้: **closure ต่างจาก `fn` ธรรมดาอย่างไรกันแน่
ในเชิงความสามารถ ไม่ใช่แค่ไวยากรณ์?**

คำตอบคือ **capturing**: closure สามารถ "มองเห็นและใช้งานตัวแปรที่ประกาศอยู่นอกตัวมันเอง" (ตัวแปรจาก scope ที่
closure ถูกสร้างขึ้นมา เรียกว่า **environment** ของ closure) ได้โดยตรง โดยไม่ต้องรับมันเป็น parameter เลย ในขณะที่
`fn` ธรรมดา **ไม่มีทางทำแบบนี้ได้เลยไม่ว่ากรณีใด** — ทุกอย่างที่ `fn` ต้องใช้ต้องผ่านมาทาง parameter ที่ประกาศไว้
ชัดเจนเท่านั้น

```rust
fn main() {
    let discount_rate = 0.1; // ตัวแปรใน scope ของ main

    // closure นี้ "เห็น" discount_rate ได้เลย โดยไม่ต้องรับมันเป็น parameter
    let apply_discount = |price: f64| price * (1.0 - discount_rate);

    println!("{}", apply_discount(200.0));
}
```

```
180
```

สังเกต signature ของ `apply_discount` — มันรับ parameter แค่ `price: f64` ตัวเดียว แต่ตัว body กลับใช้
`discount_rate` ได้ด้วย ทั้งที่ `discount_rate` ไม่ใช่ parameter ของ closure เลย — closure **"ยืม" ตัวแปรนั้นมา
จาก environment รอบตัวมันโดยอัตโนมัติ** เพราะมันถูกใช้อยู่ใน body

ทีนี้ลองทำสิ่งเดียวกันด้วย `fn` ธรรมดาดูว่าจะเกิดอะไรขึ้น:

```rust
fn main() {
    let discount_rate = 0.1;

    fn apply_discount_bare(price: f64) -> f64 {
        price * (1.0 - discount_rate) // พยายามใช้ discount_rate จาก main ตรง ๆ
    }

    println!("{}", apply_discount_bare(200.0));
}
```

```
error[E0434]: can't capture dynamic environment in a fn item
 --> src/main.rs:5:24
  |
5 |         price * (1.0 - discount_rate)
  |                        ^^^^^^^^^^^^^
  |
  = help: use the `|| { ... }` closure form instead
```

นี่คือ error ที่ตรงประเด็นที่สุดในเรื่องนี้ — **`can't capture dynamic environment in a fn item`** และ compiler
ยังแนะนำวิธีแก้ตรง ๆ ด้วยว่า `use the || { ... } closure form instead` เหตุผลเชิงกลไกคือ: `fn` (แม้จะประกาศซ้อน
อยู่ข้างในฟังก์ชันอื่นแบบในตัวอย่างนี้ก็ตาม) **ไม่มี "environment" ผูกติดตัวมันเลย** มันคือ code path ที่ตายตัว
คงที่ระดับ binary เหมือนกับฟังก์ชัน `main` เอง ไม่ได้มี "instance" ของตัวเองที่เก็บสถานะอะไรไว้ — เมื่อ Rust
compile `fn apply_discount_bare` มันไม่รู้ (และไม่สามารถรู้ได้) ว่าตอนถูกเรียกจะมี `discount_rate` ตัวไหนอยู่ใน
scope บ้าง เพราะ `fn` ตัวนี้อาจถูกเรียกจากที่อื่นที่ไม่มี `discount_rate` เลยก็ได้ (ในทางเทคนิค `fn` แต่ละตัวคือ
function pointer ที่ชี้ไปยัง code ตายตัวใน binary — ไม่มีที่เก็บข้อมูลเพิ่มเติมสำหรับ "จำ" ค่าตัวแปรจากที่มัน
ถูกประกาศไว้เลย)

closure ในทางกลับกัน **ไม่ใช่แค่ code path — มันคือ "ค่า" (value) ที่มีข้อมูลเก็บอยู่ในตัวมันเองด้วย** (ตัวแปรที่
capture มา) นี่คือความแตกต่างพื้นฐานที่สุดที่ทำให้ closure ทำสิ่งที่ `fn` ทำไม่ได้: **closure = code + data ที่
capture มาจาก environment ผูกติดกัน ในขณะที่ `fn` = code เปล่า ๆ ไม่มี data ผูกติดเลย** หัวข้อถัดไปจะเจาะลึกว่า
"data ที่ capture มา" นั้นถูกเก็บแบบไหนกันแน่ (ยืมมาดู, ยืมมาแก้ไข, หรือยึดมาเป็นเจ้าของ) ซึ่งนำไปสู่ trait สาม
ตัวที่เป็นหัวใจของบทนี้

### 24.5 กลไกเบื้องหลัง: Capture Mode สามแบบ และ Trait `Fn` / `FnMut` / `FnOnce`

เมื่อ closure capture ตัวแปรจาก environment มันมีวิธี "ยึด" ตัวแปรนั้นได้สามแบบ ตรงกับสามระดับของ ownership ที่
เรียนมาแล้วตั้งแต่ Part 6-7 พอดี:

1. **ยืมมาอ่านเฉย ๆ** (`&T`) — closure ตัวนี้ implement trait **`Fn`**
2. **ยืมมาแบบแก้ไขได้** (`&mut T`) — closure ตัวนี้ implement trait **`FnMut`**
3. **ยึด ownership มาเป็นของตัวเอง** (`T` ตรง ๆ ผ่านการ move) — closure ตัวนี้ implement trait **`FnOnce`**

จุดที่สำคัญที่สุดที่ต้องเข้าใจตั้งแต่ต้น (จะย้ำอีกครั้งในหัวข้อ 24.5.4): **คุณไม่ได้เป็นคนเลือกหรือประกาศว่า
closure ของคุณจะ implement trait ไหน** — **compiler วิเคราะห์ body ของ closure ว่าตัวแปรที่ capture มาถูกใช้
งานแบบไหน แล้วเลือก capture mode ที่ "restrictive น้อยที่สุด" ที่ยังทำงานถูกต้องให้เองโดยอัตโนมัติ** คุณเขียน
`|x| price * (1.0 - discount_rate)` เฉย ๆ compiler เป็นคนตัดสินใจว่า `discount_rate` ควรถูกยืม (`&f64`) หรือ
ยึดมา (`f64`) จากการดูว่า body ใช้มันอย่างไร

#### 24.5.1 `Fn`: Capture by Reference — เรียกได้หลายครั้ง ไม่แก้ไขค่าที่ capture มา

```rust
fn main() {
    let threshold = 100;

    // closure นี้แค่ "อ่าน" threshold เพื่อเปรียบเทียบ ไม่เคยแก้ไขค่าของมันเลย
    // -> compiler เลือก capture mode ที่ผ่อนคลายที่สุด: ยืมมาอ่านเฉย ๆ (&i32)
    // -> closure ตัวนี้ implement trait Fn
    let is_expensive = |price: i32| price > threshold;

    println!("{}", is_expensive(50));
    println!("{}", is_expensive(150));

    // threshold ยังใช้งานได้ตามปกติหลังเรียก closure ไปแล้วหลายครั้ง เพราะ is_expensive แค่ "ยืมดู" มันเท่านั้น
    println!("threshold ยังใช้ได้อยู่ที่นี่: {threshold}");
}
```

```
false
true
threshold ยังใช้ได้อยู่ที่นี่: 100
```

สังเกตว่าเราเรียก `is_expensive` ได้ถึงสองครั้ง (`is_expensive(50)` และ `is_expensive(150)`) และหลังจากนั้น
`threshold` ก็ยังใช้งานได้ตามปกติ — นี่คือคุณสมบัติของ `Fn`: **capture โดยการยืมด้วย `&T` (immutable borrow)
เท่านั้น** เหมือนกับที่ `&T` ธรรมดาให้ยืมพร้อมกันได้หลายที่ตามกฎ borrow checker จาก Part 7 closure ที่ implement
`Fn` จึงเรียกซ้ำได้ไม่จำกัดจำนวนครั้ง (ตราบใดที่ยังอยู่ใน scope ที่ตัวแปรต้นทางยังไม่ตาย) และตัวแปรต้นทาง
(`threshold`) ก็ยังเป็นเจ้าของค่าเดิมอยู่ครบถ้วนหลังจากนั้น เหมือนกับที่ `.as_ref()` ใน Part 11 ไม่ทำลาย
ownership ของเจ้าของเดิม

#### 24.5.2 `FnMut`: Capture by Mutable Reference — เรียกได้หลายครั้ง และแก้ไขค่าที่ capture มาได้

```rust
fn main() {
    let mut count = 0;

    // closure นี้ "แก้ไข" count ทุกครั้งที่ถูกเรียก (count += 1)
    // -> compiler ต้อง capture แบบยืมมาแก้ไขได้ (&mut i32)
    // -> closure ตัวนี้ implement trait FnMut (ไม่ใช่ Fn อีกต่อไป เพราะมันแก้ไขค่าที่ capture มา)
    let mut increment = || {
        count += 1;
        println!("count = {count}");
    };

    increment();
    increment();
    increment();
}
```

```
count = 1
count = 2
count = 3
```

สังเกตสองจุดสำคัญ: (1) เราต้องประกาศตัวแปร `increment` ด้วย `let mut increment = ...` **ไม่ใช่แค่ `let
increment = ...`** เฉย ๆ — เพราะการเรียก `FnMut` closure แต่ละครั้งต้องยืม `increment` แบบ `&mut` (เพื่อให้
closure แก้ไข `count` ที่มันเก็บ reference ไว้ได้) และการยืม `&mut` ได้ก็ต่อเมื่อตัวแปรต้นทางเป็น `mut` เท่านั้น
ตามกฎ Part 7 (ถ้าลืม `mut` ตรงนี้จะเจอ error ที่เราจะเห็นเต็ม ๆ ในหัวข้อ "กับดักที่พบบ่อย") (2) `count` เพิ่มขึ้น
ทีละ 1 ทุกครั้งที่เรียก `increment()` และค่านั้น**คงอยู่ข้ามการเรียกแต่ละครั้ง** (call ที่ 3 เห็นผลจาก call ที่ 1
และ 2 สะสมมา) — นี่พิสูจน์ว่า closure ไม่ได้แค่ "ยืมดู" `count` เฉย ๆ แบบ `Fn` แต่มันยึด **mutable reference**
ไปเก็บไว้ในตัวเอง แก้ไขค่าจริงของ `count` ทุกครั้งที่ถูกเรียก เหมือนกับที่ `.as_mut()` ใน Part 11 ให้สิทธิ์แก้ไข
ค่าข้างในโดยไม่ต้อง unwrap/reassign ทั้งตัว

**`FnMut` เรียกได้หลายครั้งเหมือน `Fn`** (ต่างจาก `FnOnce` ที่จะเห็นต่อไป) เพราะ mutable reference ที่ closure
ยึดไว้ไม่ได้ถูก "ใช้แล้วทิ้ง" — มันแค่ถูกยืมชั่วคราวในแต่ละการเรียก แล้วคืนกลับให้ต้นทางหลังเรียกจบ (ตามกฎ borrow
checker ปกติ) พร้อมให้ยืมใหม่ในการเรียกครั้งถัดไปได้อีก

#### 24.5.3 `FnOnce`: Capture by Value — ยึด Ownership มาเป็นของตัวเอง เรียกได้ครั้งเดียวเท่านั้น

```rust
fn main() {
    let name = String::from("สมชาย");

    // move บอก compiler ให้ยึด ownership ของ name เข้ามาใน closure ตรง ๆ (จะเจาะลึก move ในหัวข้อ 24.6)
    // ข้างใน body เราย้าย name ออกไปเป็น owned ตัวใหม่ (let owned = name;) -> "ย้าย" ค่าที่ capture มาออกไปจริง ๆ
    // -> closure ตัวนี้ implement ได้แค่ FnOnce เท่านั้น (เรียกซ้ำไม่ได้)
    let consume = move || {
        let owned = name;
        println!("สวัสดี {owned}");
    };

    consume();
}
```

```
สวัสดี สมชาย
```

โค้ดนี้เรียก `consume()` แค่ครั้งเดียว ทำงานได้ปกติ แต่ลองเรียกซ้ำเป็นครั้งที่สองดูว่าจะเกิดอะไรขึ้น:

```rust
fn main() {
    let name = String::from("สมชาย");
    let consume = move || {
        let owned = name;
        println!("สวัสดี {owned}");
    };

    consume();
    consume(); // เรียกซ้ำครั้งที่สอง
}
```

```
error[E0382]: use of moved value: `consume`
 --> src/main.rs:9:5
  |
8 |     consume();
  |     --------- `consume` moved due to this call
9 |     consume();
  |     ^^^^^^^ value used here after move
  |
note: closure cannot be invoked more than once because it moves the variable `name` out of its environment
 --> src/main.rs:4:21
  |
4 |         let owned = name;
  |                     ^^^^
note: this value implements `FnOnce`, which causes it to be moved when called
 --> src/main.rs:8:5
  |
8 |     consume();
  |     ^^^^^^^
```

อ่าน error message นี้ทีละบรรทัด — compiler บอกตรงจุดที่สำคัญที่สุดไว้ชัดเจนมาก:

- **`use of moved value: consume`** — ตัวแปร `consume` **เอง** (ไม่ใช่แค่ `name` ข้างในมัน) ถูก move ไปแล้วตอน
  เรียก `consume()` ครั้งแรก
- **`closure cannot be invoked more than once because it moves the variable name out of its environment`**
  — นี่คือคำอธิบายที่ตรงประเด็นที่สุด: เพราะ body ของ closure มีบรรทัด `let owned = name;` ซึ่งคือการ **move**
  `name` ออกจาก field ที่ closure เก็บไว้ (ตามกฎ move semantics จาก Part 6 — เมื่อ move ค่าออกจากที่หนึ่งแล้ว
  ที่เดิมจะใช้ต่อไม่ได้อีก) การ move แบบนี้ทำได้แค่ **ครั้งเดียว** เพราะหลังจากย้าย `name` ออกไปแล้ว ไม่มี `name`
  เหลือให้ย้ายอีกในการเรียกครั้งที่สอง
- **`this value implements FnOnce, which causes it to be moved when called`** — บรรทัดนี้เผยกลไกที่ลึกที่สุด:
  **การเรียก `FnOnce` closure หนึ่งครั้ง คือการ move closure ตัวนั้นทั้งตัวเข้าไปในการเรียก** (ไม่ใช่แค่ยืม
  `&self`/`&mut self` แบบ `Fn`/`FnMut`) เพราะ trait `FnOnce` นิยาม method ที่ใช้เรียกไว้ว่า `call_once(self,
  args)` — รับ `self` แบบ **by value** ตรง ๆ ไม่ใช่ `&self` หรือ `&mut self` — นี่คือเหตุผลที่ครั้งที่สองที่เรา
  พยายามเรียก `consume()` จึงเจอ "use of moved value" กับตัว `consume` เอง ไม่ใช่แค่กับ `name` ข้างในมัน:
  **ตัว closure ทั้งก้อนถูกยึดไปใช้ (consumed) ตั้งแต่การเรียกครั้งแรกแล้ว**

นี่คือเหตุผลที่ชื่อ trait นี้คือ **"FnOnce"** ตรงตัว — มันคือสัญญาว่า **"เรียกได้อย่างมากหนึ่งครั้ง"** เพราะการ
เรียกแต่ละครั้งอาจ move ข้อมูลที่ capture มาออกไปใช้จนหมด ไม่มีอะไรเหลือให้เรียกซ้ำ

#### 24.5.4 Compiler เลือก Trait ให้เอง — คุณไม่ต้องประกาศ แต่ควรอ่านมันออกจาก Body

นี่คือประเด็นที่ลึกและสำคัญที่สุดของบทนี้ ให้ย้ำอีกครั้งให้ชัด: **คุณไม่มีทางเขียนบอก compiler ตรง ๆ ว่า "closure
ตัวนี้จงเป็น `Fn`" หรือ "จงเป็น `FnMut`"** — ไม่มี syntax แบบนั้นอยู่จริงเลย สิ่งที่คุณทำได้คือเขียน **body** ของ
closure ให้ทำสิ่งที่ต้องการ แล้ว **compiler จะวิเคราะห์ทุกตัวแปรที่ capture มา ดูว่าแต่ละตัวถูกใช้อย่างไรใน body
แล้วอนุมาน capture mode ที่ "ผ่อนคลายที่สุดเท่าที่ยังทำงานถูกต้อง" ให้เองทั้งหมด**

กฎการอนุมานสรุปสั้น ๆ ได้ดังนี้ (เรียงจากผ่อนคลายไปเข้มงวด — compiler จะเลือกตัวที่ผ่อนคลายที่สุดที่ยังพอ):

| ลักษณะการใช้ตัวแปรที่ capture มาใน body | Capture mode ที่ compiler เลือก | Trait ที่ implement |
|---|---|---|
| แค่**อ่าน**ค่า (ไม่แก้ไข ไม่ย้ายออก) | ยืมแบบ `&T` | `Fn` (และ `FnMut`, `FnOnce` ด้วย ตามหัวข้อ 24.5.5) |
| **แก้ไข**ค่า (เช่น `+=`, เรียก method ที่รับ `&mut self`) | ยืมแบบ `&mut T` | `FnMut` (และ `FnOnce`) |
| **ย้าย/ยึด ownership** ออกไป (เช่น move ค่าออกไปที่อื่น, drop มันไปตรง ๆ, ส่งเข้าฟังก์ชันที่รับ `T` by value) | ยึดมาเป็นของตัวเอง | `FnOnce` เท่านั้น |

ตัวอย่างที่แสดงให้เห็นว่า compiler "อ่าน body" จริง ๆ ไม่ใช่แค่ดูว่ามี `move` keyword หรือไม่:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    // closure นี้ไม่มี move keyword เลย แต่ body แค่อ่าน numbers (ผ่าน .len() ที่รับ &self)
    // -> compiler capture numbers แบบ &Vec<i32> โดยอัตโนมัติ -> ได้ Fn
    let describe = || println!("numbers มี {} ตัว: {:?}", numbers.len(), numbers);

    describe();
    describe();

    // numbers ยังใช้งานได้ตามปกติ เพราะ describe แค่ยืมมันไปดูเฉย ๆ
    println!("ใช้ numbers ต่อได้ตามปกติ: {numbers:?}");
}
```

```
numbers มี 5 ตัว: [1, 2, 3, 4, 5]
numbers มี 5 ตัว: [1, 2, 3, 4, 5]
ใช้ numbers ต่อได้ตามปกติ: [1, 2, 3, 4, 5]
```

สังเกตว่าเราไม่ได้เขียน `move` เลยสักคำ แต่ closure ก็ capture `numbers` มาแบบยืม (`&Vec<i32>`) ได้เองโดยอัตโนมัติ
เพราะ body ของมันแค่ "อ่าน" (`numbers.len()`, print มันออกมา) ไม่มีจุดไหนที่ move `numbers` ออกไปเลย — **นี่คือ
สิ่งที่บอกไว้ว่า Rust ค่า default (ไม่ใส่ `move`) จะพยายามยืม (borrow) ก่อนเสมอถ้าเป็นไปได้ และยึด ownership
(move) ก็ต่อเมื่อ body ต้องการมันจริง ๆ เท่านั้น** (หรือเราสั่งบังคับด้วย `move` keyword ตามหัวข้อ 24.6)

**ข้อสังเกตสำคัญเชิง design ที่ควรจำไว้**: การที่ compiler เลือก trait ให้เองจาก **การใช้งานจริง** (usage-based
inference) แทนที่จะให้เราประกาศเอง มีข้อดีที่สำคัญมาก — คุณไม่มีทาง "ประกาศผิด" ได้เลย closure ที่เขียนแบบไม่
ต้องแก้ไขอะไรเลยจะได้ `Fn` ที่ผ่อนคลายที่สุดโดยอัตโนมัติเสมอ ไม่ต้องกลัวว่าจะลืมเปลี่ยนจาก `FnMut` เป็น `Fn` ตอน
refactor code แล้วลืมอัปเดต signature (ปัญหาที่พบได้จริงในภาษาอื่นที่ต้องประกาศ type ของ callback ไว้ตรง ๆ)
สิ่งที่คุณควบคุมได้มีแค่ **สิ่งที่ body ทำกับตัวแปรที่ capture มา** — เปลี่ยนพฤติกรรมของ body เมื่อไหร่ trait ที่
closure implement ก็เปลี่ยนไปตามโดยอัตโนมัติทันที

#### 24.5.5 ลำดับชั้นของ Trait: `Fn` ⊆ `FnMut` ⊆ `FnOnce`

จากตารางข้างบน ให้สังเกตว่าคอลัมน์ "Trait ที่ implement" ของ `Fn` เขียนไว้ว่า **"`Fn` (และ `FnMut`, `FnOnce`
ด้วย)"** — นี่ไม่ใช่การเขียนพลาด แต่คือกฎสำคัญที่ต้องเข้าใจ: **`Fn` เป็น subtrait ของ `FnMut`, และ `FnMut` เป็น
subtrait ของ `FnOnce`** เขียนเป็นสัญลักษณ์ได้ว่า:

```
Fn : FnMut : FnOnce
```

(อ่านว่า "`Fn` ต้อง implement `FnMut` ด้วย" และ "`FnMut` ต้อง implement `FnOnce` ด้วย" — เหมือนกับ trait bound
`trait Fn: FnMut` และ `trait FnMut: FnOnce` ที่ std library ประกาศไว้จริง ๆ) ความหมายในทางปฏิบัติคือ:

- **closure ที่ implement `Fn` ก็ implement `FnMut` และ `FnOnce` โดยอัตโนมัติด้วย** — เพราะสิ่งที่แค่ "อ่าน"
  ได้อยู่แล้ว ก็ย่อม "แก้ไขได้ (ไม่แก้ก็ได้)" และ "เรียกครั้งเดียวได้ (เรียกหลายครั้งก็ยิ่งได้)" อย่างไม่มีปัญหา
- **closure ที่ implement `FnMut` (แต่ไม่ implement `Fn`) ก็ implement `FnOnce` ด้วย** — เพราะสิ่งที่เรียกได้
  หลายครั้งโดยแก้ไขสถานะ ก็ย่อมเรียกได้อย่างน้อยหนึ่งครั้งเช่นกัน
- **closure ที่ implement ได้แค่ `FnOnce` เท่านั้น** (แบบ `consume` ในหัวข้อ 24.5.3) **ไม่ implement `Fn` หรือ
  `FnMut` เลย** เพราะมันเรียกซ้ำไม่ได้ตั้งแต่ต้น

นี่คือเหตุผลว่าทำไมฟังก์ชันที่รับ closure ด้วย bound `FnOnce` (ผ่อนคลายที่สุด) จึงรับ closure **ได้ทุกแบบ** ไม่ว่า
closure นั้นจะ capture แบบไหนก็ตาม (แม้แต่ closure ที่ implement `Fn` เต็ม ๆ ก็ยังส่งเข้าไปในพารามิเตอร์ที่ขอ
`FnOnce` ได้ เพราะ `Fn` implies `FnOnce`) ในขณะที่ฟังก์ชันที่ขอ bound `Fn` (เข้มงวดที่สุด) จะรับได้แค่ closure
ที่ capture แบบอ่านอย่างเดียวเท่านั้น — หัวข้อ 24.10 จะกลับมาขยายความเรื่องนี้พร้อมตัวอย่าง signature เปรียบเทียบ
ให้เห็นภาพชัดเจนกว่านี้

### 24.6 คำสั่ง `move`: บังคับให้ Closure ยึด Ownership

หัวข้อที่แล้วบอกว่า Rust จะพยายาม **ยืม** ตัวแปรที่ capture มาก่อนเสมอถ้าเป็นไปได้ (ผ่อนคลายที่สุด) — แต่บางครั้ง
เราต้องการ **บังคับ** ให้ closure ยึด ownership ของตัวแปรที่ capture มาทั้งหมด แม้ว่า body ของมันจะแค่ "อ่าน"
เฉย ๆ และไม่จำเป็นต้องยึดมาก็ได้ — นี่คือหน้าที่ของ keyword **`move`** ที่วางไว้หน้า closure: `move || ...`

#### ทำไมต้องมี `move` — ปัญหาคลาสสิก: Return Closure ที่ Capture ตัวแปร Local

สถานการณ์ที่พบบ่อยที่สุดที่ **ต้อง** ใช้ `move` คือเมื่อ closure ต้อง **"มีชีวิตอยู่ต่อ" นานกว่า scope ที่มันถูก
สร้างขึ้นมา** ตัวอย่างที่ชัดที่สุดคือการ return closure ออกจากฟังก์ชัน — ลองดูว่าเกิดอะไรขึ้นถ้าเราลืมใส่ `move`:

```rust
fn make_printer() -> impl Fn() {
    let message = String::from("สวัสดีจาก closure");
    || println!("{message}") // ไม่ได้ใส่ move -> closure พยายามแค่ "ยืม" message
}

fn main() {
    let printer = make_printer();
    printer();
}
```

```
error[E0373]: closure may outlive the current function, but it borrows `message`, which is owned by the current function
 --> src/main.rs:3:5
  |
3 |     || println!("{message}")
  |     ^^            ------- `message` is borrowed here
  |     |
  |     may outlive borrowed value `message`
  |
note: closure is returned here
 --> src/main.rs:3:5
  |
3 |     || println!("{message}")
  |     ^^^^^^^^^^^^^^^^^^^^^^^^
help: to force the closure to take ownership of `message` (and any other referenced variables), use the `move` keyword
  |
3 |     move || println!("{message}")
  |     ++++
```

อ่าน error นี้ให้เข้าใจกลไกจริง ๆ ที่อยู่ข้างใต้ — **`closure may outlive the current function, but it borrows
message, which is owned by the current function`** พูดตรง ๆ ว่า: closure ตัวนี้ **มีทีท่าว่าจะถูกใช้งานนานกว่า
อายุของฟังก์ชัน `make_printer` เอง** (เพราะมันถูก return ออกไปให้ `main` เก็บไว้ในตัวแปร `printer` แล้วเรียกใช้
ทีหลัง) แต่ `message` เป็นตัวแปร **local** ที่เป็นเจ้าของอยู่ใน scope ของ `make_printer` เท่านั้น — เมื่อ
`make_printer` return (จบการทำงาน) `message` จะถูก `drop` ไปตามกฎ ownership จาก Part 6 (ตัวแปร local ที่ไม่ได้
ถูก move ออกไปไหน จะถูกทำลายทันทีที่ scope จบ) ถ้า closure แค่ "ยืม" reference ไปยัง `message` (ไม่ได้ยึด
ownership มา) reference นั้นจะกลายเป็น **dangling reference** ที่ชี้ไปยังหน่วยความจำที่ถูกคืนไปแล้ว ทันทีที่
`main` เรียก `printer()` — ซึ่งเป็นสิ่งที่ borrow checker **ห้ามเด็ดขาด** (นี่คือ use-after-free ที่ Rust ป้องกัน
ไว้ตั้งแต่ compile time ตามหลักการที่เรียนมาตั้งแต่ Part 6-7)

compiler แนะนำวิธีแก้ไว้ตรง ๆ ท้าย error: **`use the move keyword`** — เพิ่ม `move` เข้าไปหน้า closure:

```rust
fn make_printer() -> impl Fn() {
    let message = String::from("สวัสดีจาก closure");
    move || println!("{message}") // ยึด ownership ของ message เข้ามาในตัว closure ทั้งตัว
}

fn main() {
    let printer = make_printer();
    printer();
    printer(); // เรียกได้หลายครั้ง เพราะ closure ยึด message มาเก็บเอง ไม่ใช่ยืม -> ยังเป็น Fn ปกติ (แค่อ่าน)
}
```

```
สวัสดีจาก closure
สวัสดีจาก closure
```

เมื่อเพิ่ม `move` เข้าไป `message` จะถูก **move เข้าไปเก็บเป็นส่วนหนึ่งของ closure เองโดยตรง** (ตามกลไกที่จะ
เห็นเป็นภาพชัดเจนขึ้นในหัวข้อ 24.9 — closure ที่มี `move` จะเก็บ `message: String` เป็น field ของมันเองตรง ๆ ไม่
ใช่ `&String` ที่ชี้กลับไปยัง `make_printer`) เมื่อ `make_printer` จบการทำงาน `message` ตัวแปร local เดิมไม่มีอยู่
แล้วก็จริง แต่ **ข้อมูล `String` จริง ๆ ถูกย้ายติดไปกับ closure ที่ return ออกไปด้วย** — ไม่มี dangling reference
เกิดขึ้นเลย closure จึง "พกพา" ข้อมูลที่ต้องใช้ติดตัวไปได้ ไม่ขึ้นกับ scope เดิมที่มันถูกสร้างขึ้นมาอีกต่อไป

ข้อสังเกตที่สำคัญ: closure ตัวนี้ที่มี `move` **ยังคง implement `Fn` อยู่** (ไม่ได้กลายเป็น `FnOnce` ไปเพราะมี
`move`) สังเกตจากที่เราเรียก `printer()` ได้ถึงสองครั้งในตัวอย่างข้างบน — **`move` ควบคุมแค่ "วิธี capture"
(ยึดมาเป็นเจ้าของ แทนการยืม) แต่ไม่ได้ควบคุม "จำนวนครั้งที่เรียกได้" โดยตรง** เพราะ trait ที่ closure implement
ยังขึ้นอยู่กับว่า **body ทำอะไรกับข้อมูลที่ยึดมานั้นต่อ** (ในที่นี้ body แค่ `println!` อ่านค่า `message` เฉย ๆ
ไม่ได้ move มันออกไปไหนต่อ จึงยัง implement `Fn` ได้ตามปกติ ต่างจากตัวอย่าง `consume` ในหัวข้อ 24.5.3 ที่ body
move `name` ออกไปต่อ (`let owned = name;`) จึงเหลือแค่ `FnOnce`)

#### เมื่อไหร่ต้องใช้ `move` — สรุปกฎการตัดสินใจ

ใช้ `move` เมื่อสถานการณ์เข้าเงื่อนไขข้อใดข้อหนึ่งต่อไปนี้:

1. **Return closure ออกจากฟังก์ชัน** ที่ closure นั้น capture ตัวแปร local ของฟังก์ชันนั้นเอง (ตามตัวอย่างข้างบน)
   — ตัวแปร local จะตายไปตอนฟังก์ชันจบ closure ที่จะอยู่ต่อจึงต้องยึดข้อมูลมาเป็นของตัวเอง
2. **ส่ง closure ข้าม thread boundary** เช่นผ่าน `std::thread::spawn(closure)` ที่จะเรียนเต็มรูปแบบใน **Part 37**
   — thread ใหม่อาจทำงานนานกว่า scope เดิมที่สร้าง closure ขึ้นมา (หรือแม้กระทั่ง thread หลักอาจจบไปแล้วก่อน
   thread ใหม่ด้วยซ้ำ) การยืม reference ข้าม thread แบบนี้จึงเสี่ยงต่อ dangling reference เหมือนกันในหลักการ
   `thread::spawn` จึง**บังคับ**ให้ closure ที่ส่งเข้าไปต้อง implement `'static` lifetime เสมอ ซึ่งในทางปฏิบัติ
   มักหมายถึงต้องใส่ `move` เพื่อยึดข้อมูลมาเป็นของตัวเอง ไม่ผูกกับ lifetime ของ scope เดิมอีกต่อไป (บทนี้ขอ
   แค่เกริ่นไว้เป็นแรงจูงใจ รายละเอียดเต็มรูปแบบเรื่อง thread และทำไม `move` จำเป็นในบริบทนั้นจะอยู่ใน Part 37)
3. **ต้องการยึด ownership ของค่าที่ capture มาไปเลย โดยไม่สนว่า body จะแค่อ่านหรือไม่** เช่นเมื่อต้องการให้
   closure เป็นเจ้าของข้อมูลอย่างชัดเจน (explicit) เพื่อความง่ายในการอ่านโค้ด แม้ในทางเทคนิคการยืมก็เพียงพอ
   (บางครั้งใช้ `move` เพื่อ "จบปัญหา lifetime" ให้เรียบง่ายขึ้น แม้ compiler จะยืมให้ได้ก็ตาม)

ถ้าไม่เข้าเงื่อนไขข้อใดข้อหนึ่งเลย (closure ถูกใช้แค่ในขอบเขต scope เดียวกับที่มันถูกสร้างขึ้นมา ไม่ได้ return
ออกไปไหน ไม่ได้ส่งข้าม thread) **ไม่ต้องใส่ `move`** — ให้ compiler เลือกยืมให้เองตามค่า default (ผ่อนคลาย
ที่สุด) จะดีกว่า เพราะการยึด ownership โดยไม่จำเป็นอาจทำให้ตัวแปรต้นทางใช้งานต่อไม่ได้ทั้งที่ควรจะยังใช้ได้อยู่

### 24.7 Closure เป็น Parameter: Generic, `impl Trait`, และ `dyn Trait`

เมื่อเราต้องการเขียนฟังก์ชันที่ **รับ closure เป็น parameter** (higher-order function) มีสามวิธีให้เลือก ซึ่งตรง
กับสามวิธีที่ Part 19 สอนไว้แล้วสำหรับรับ trait ทั่วไปเป็น parameter — เพราะ `Fn`/`FnMut`/`FnOnce` ก็คือ trait
ธรรมดา ๆ ไม่มีอะไรพิเศษไปกว่านั้น:

```rust
// วิธีที่ 1: Generic parameter ที่มี trait bound เป็น Fn — static dispatch เต็มรูปแบบ
fn apply<F: Fn(i32) -> i32>(f: F, x: i32) -> i32 {
    f(x)
}

// วิธีที่ 2: impl Trait syntax — น้ำตาลไวยากรณ์ของวิธีที่ 1 เป๊ะ ๆ (ตามที่เรียนใน Part 19 หัวข้อ 19.6)
fn apply_impl(f: impl Fn(i32) -> i32, x: i32) -> i32 {
    f(x)
}

// วิธีที่ 3: Trait object ผ่าน &dyn Trait — dynamic dispatch (ตามที่เรียนใน Part 19 หัวข้อ 19.9 และ Part 21)
fn apply_dyn(f: &dyn Fn(i32) -> i32, x: i32) -> i32 {
    f(x)
}

fn main() {
    let double = |x| x * 2;

    println!("apply (generic):      {}", apply(double, 5));
    println!("apply_impl:           {}", apply_impl(double, 5));
    println!("apply_dyn (trait obj): {}", apply_dyn(&double, 5));
}
```

```
apply (generic):      10
apply_impl:           10
apply_dyn (trait obj): 10
```

สังเกตว่าเราใช้ `double` closure ตัวเดียวกันเรียกได้ทั้งสามฟังก์ชันติดกัน — เพราะ `double` (`|x| x * 2`) เป็น
closure ที่ **ไม่ capture อะไรเลย** (ไม่มีตัวแปรจาก environment ให้ใช้) closure แบบนี้ implement `Copy` โดย
อัตโนมัติด้วย (เหมือน `i32`/`bool` ที่เรียนใน Part 6) ทำให้การส่ง `double` เข้า `apply(double, 5)` ที่รับ `F`
by value ไม่ได้ "ทำลาย" `double` ตัวเดิม (มันแค่ copy ค่า — ซึ่งในกรณีนี้คือ copy "ความว่างเปล่า" เพราะไม่มี
data ให้เก็บเลย) `double` จึงยังใช้ต่อกับ `apply_impl` และ `apply_dyn` ได้อีก

**เมื่อไหร่ควรเลือกวิธีไหน — เกณฑ์เดียวกับที่เรียนใน Part 19 นำมาประยุกต์กับ closure ตรง ๆ:**

- **`<F: Fn(...)>` (generic) หรือ `impl Fn(...)` (น้ำตาลไวยากรณ์ตัวเดียวกัน)** — เลือกใช้เป็นค่า default เสมอ
  เมื่อไม่มีเหตุผลพิเศษให้เลือกอย่างอื่น เพราะมันคือ **static dispatch**: compiler ทำ **monomorphization**
  (ตามที่เรียนใน Part 18) สร้างเวอร์ชันของฟังก์ชันขึ้นมาเฉพาะสำหรับ concrete type ของ closure ที่ถูกเรียกจริง
  แต่ละแบบ ทำให้ compiler สามารถ **inline** การเรียก `f(x)` เข้าไปในตัวฟังก์ชันได้ตรง ๆ ไม่มีการ "ค้นหา method
  ที่ถูกต้องตอน runtime" เลยแม้แต่นิดเดียว — นี่คือ **zero-cost abstraction** (คำที่หลักสูตรนี้พูดถึงมาตั้งแต่
  Part 1 และเห็นตัวอย่างจริงมาแล้วใน Part 11 กับ Part 19) ข้อจำกัดเดียวคือ ถ้าคุณต้องการเก็บ closure หลายตัวที่
  concrete type ต่างกันไว้ใน collection เดียว (เช่น `Vec<F>`) generic แบบนี้ทำไม่ได้ เพราะ `Vec<F>` ต้องมี `F`
  เป็น concrete type เดียวตายตัว
- **`&dyn Fn(...)` หรือ `Box<dyn Fn(...)>` (trait object)** — เลือกใช้เมื่อต้องการ **dynamic dispatch** จริง ๆ
  เช่นเมื่อต้องเก็บ closure หลาย concrete type ที่ไม่เหมือนกันไว้ใน collection เดียวกัน (`Vec<Box<dyn Fn(...)>>`
  ที่จะเห็นเต็มรูปแบบในหัวข้อ 24.11) หรือเมื่อต้องส่ง closure ผ่าน function pointer/callback ที่ signature ต้อง
  ตายตัวไม่ผันตาม type (เช่น FFI หรือ plugin system ที่ compile แยกกัน) ต้นทุนที่ต้องแลกคือการค้นหา method ที่
  ถูกต้องผ่าน **vtable** ตอน runtime (รายละเอียดเต็มรูปแบบเรื่อง vtable และ object safety อยู่ใน Part 21) ซึ่ง
  ช้ากว่า static dispatch เล็กน้อยแต่ในโปรแกรมส่วนใหญ่ไม่มีผลกระทบจนต้องกังวล

**กฎง่าย ๆ สำหรับตอนนี้**: ถ้าฟังก์ชันรับ closure แค่ตัวเดียวและเรียกมันตรง ๆ ไม่ต้องเก็บไว้ใน collection —
ใช้ `impl Fn(...)` (หรือ generic ก็ได้ ผลลัพธ์เหมือนกัน) เป็นค่า default เสมอ เพราะเร็วกว่าและอ่านง่ายกว่า
ใช้ `dyn Fn(...)` ก็ต่อเมื่อจำเป็นจริง ๆ เท่านั้น (เก็บใน collection ที่มีหลาย concrete type, หรือ signature ต้อง
ตายตัวไม่ generic)

### 24.8 Return Closure จากฟังก์ชัน: ทำไม `-> Fn(...)` เขียนตรง ๆ ไม่ได้

หัวข้อที่แล้วพูดถึงการรับ closure เป็น parameter — ทีนี้มาดูฝั่ง **return closure ออกจากฟังก์ชัน** ซึ่งมีข้อจำกัด
ที่ต่างออกไปเล็กน้อย ลองเขียนแบบตรงไปตรงมาที่สุด (เขียนชื่อ trait `Fn` เป็น return type ตรง ๆ เหมือนที่เขียน
`i32` หรือ `String`) ดูก่อนว่าเกิดอะไรขึ้น:

```rust
fn make_adder(x: i32) -> Fn(i32) -> i32 {
    move |y| x + y
}

fn main() {
    let add5 = make_adder(5);
    println!("{}", add5(10));
}
```

```
error[E0782]: expected a type, found a trait
 --> src/main.rs:1:26
  |
1 | fn make_adder(x: i32) -> Fn(i32) -> i32 {
  |                          ^^^^^^^^^^^^^^
  |
help: use `impl Fn(i32) -> i32` to return an opaque type, as long as you return a single underlying type
  |
1 | fn make_adder(x: i32) -> impl Fn(i32) -> i32 {
  |                          ++++
help: alternatively, you can return an owned trait object
  |
1 | fn make_adder(x: i32) -> Box<dyn Fn(i32) -> i32> {
  |                          +++++++               +
```

error **E0782 (expected a type, found a trait)** นี้คุ้นเคยกับสิ่งที่เรียนมาแล้วใน Part 19 หัวข้อ 19.8-19.9
พอดี — **`Fn(i32) -> i32` เป็น trait ไม่ใช่ type ที่มีขนาดตายตัว** (concrete, sized type) การเขียน `-> Fn(i32)
-> i32` ตรง ๆ เท่ากับบอก compiler ว่า "return ค่าที่เป็น trait ตัวนี้" ซึ่งไม่สมเหตุสมผล เพราะ trait บอกแค่ว่า
"มีพฤติกรรมอะไร" (เรียกได้แบบ `Fn(i32) -> i32`) แต่ไม่ได้บอกว่า **ข้อมูลข้างในมีขนาดเท่าไหร่** — closure ที่
capture ตัวแปรมาก ๆ ก็มีขนาดใหญ่กว่า closure ที่ไม่ capture อะไรเลย ทั้งที่ทั้งคู่อาจ implement `Fn(i32) ->
i32` เหมือนกัน compiler จึงไม่รู้ว่าจะจอง stack space เท่าไหร่ให้กับ return value นี้ — นี่คือปัญหาเรื่อง
**sized-ness** แบบเดียวกับที่เจอตอนพยายาม return `dyn Summary` ตรง ๆ ใน Part 19

compiler แนะนำวิธีแก้ไว้สองแบบ ตรงกับสองวิธีที่ Part 19 สอนไว้แล้วเป๊ะ ๆ:

#### ทางแก้ที่ 1: `impl Fn(...)` — Static Dispatch (ใช้ได้เมื่อ Return Concrete Type เดียวเสมอ)

```rust
fn make_adder(x: i32) -> impl Fn(i32) -> i32 {
    move |y| x + y // x ถูก capture มาแบบ move เพราะต้องอยู่ต่อหลังฟังก์ชันจบ (ตามหัวข้อ 24.6)
}

fn main() {
    let add5 = make_adder(5);
    println!("{}", add5(10)); // 15
    println!("{}", add5(20)); // 25 -- เรียกซ้ำได้เพราะ add5 implement Fn (body แค่บวก ไม่ move x ออกไปไหน)
}
```

```
15
25
```

`impl Fn(i32) -> i32` บอกผู้เรียกว่า "ฟังก์ชันนี้ return ค่าที่ implement `Fn(i32) -> i32`" โดยไม่เปิดเผยว่า
concrete type จริง ๆ ข้างในคืออะไร (compiler รู้ทั้งหมดตอน compile time แต่ผู้เรียกไม่จำเป็นต้องรู้) — นี่คือ
**static dispatch เต็มรูปแบบ ไม่มี heap allocation เลย ไม่มี dynamic dispatch เลย** เร็วที่สุดเท่าที่จะเป็นไปได้
แต่มีข้อจำกัดเดียวกับที่ Part 19 สอนไว้: **ฟังก์ชันตัวนี้ต้อง return concrete type เดียวเสมอไม่ว่าจะเรียกด้วย
argument อะไรก็ตาม** (ในที่นี้คือ closure ที่มี body `move |y| x + y` เสมอ — ไม่มี branch ที่ return closure
คนละแบบกัน)

#### ทางแก้ที่ 2: `Box<dyn Fn(...)>` — Dynamic Dispatch (ใช้ได้เมื่อต้อง Return หลาย Concrete Type)

```rust
fn make_calculator(op: char, value: i32) -> Box<dyn Fn(i32) -> i32> {
    match op {
        '+' => Box::new(move |x| x + value),
        '-' => Box::new(move |x| x - value),
        '*' => Box::new(move |x| x * value),
        _ => Box::new(move |x| x), // ไม่รู้จัก op -> คืน identity function เฉย ๆ
    }
}

fn main() {
    let add5 = make_calculator('+', 5);
    let times3 = make_calculator('*', 3);

    println!("{}", add5(10));    // 15
    println!("{}", times3(10));  // 30
}
```

```
15
30
```

สังเกตว่า `make_calculator` มี **สี่ branch ที่ return closure คนละตัวกัน** (แต่ละ `move |x| ...` เป็นคนละ
concrete type กันทั้งหมดในเชิง compiler แม้จะ "หน้าตาคล้ายกัน" ในสายตาคนอ่าน) `impl Fn(i32) -> i32` **ทำไม่ได้
ในกรณีนี้** เพราะ Rust ต้องเลือก concrete type เดียวตายตัวให้กับ `impl Trait` ตั้งแต่ compile time ซึ่งเป็นไป
ไม่ได้เมื่อแต่ branch เป็นคนละ type กัน — `Box<dyn Fn(i32) -> i32>` แก้ปัญหานี้ได้เพราะทุก branch ถูก **box**
(จองพื้นที่บน heap ผ่าน `Box::new` ตามที่เรียนใน Part 19 หัวข้อ 19.9) แล้วมองผ่าน `dyn Fn(i32) -> i32` (trait
object) ที่ทำให้ทุก branch **"หน้าตาเหมือนกันจากมุมมองภายนอก"** คือ `Box<dyn Fn(i32) -> i32>` ตัวเดียวกันเป๊ะ
ไม่ว่า concrete closure ข้างในจะต่างกันแค่ไหน — ต้นทุนที่ต้องแลกคือ heap allocation ตอนสร้าง (`Box::new`) และ
dynamic dispatch ตอนเรียก (ค้นหา method ผ่าน vtable ตาม Part 21) แลกกับความยืดหยุ่นที่ทำสิ่งที่ `impl Trait`
ทำไม่ได้เลย

**กฎตัดสินใจสรุป** (เหมือนกับ Part 19 หัวข้อ 19.9 เป๊ะ ๆ แค่นำมาใช้กับ closure): **ถ้าฟังก์ชัน return closure
concrete type เดียวเสมอ (นับรวมทุก branch ของ `match`/`if` ในฟังก์ชันนั้น) ใช้ `impl Fn(...)` เพราะเร็วกว่าและ
ไม่มี heap allocation แต่ถ้าต้อง return closure ที่มีหน้าตา/พฤติกรรมต่างกันจริง ๆ ขึ้นอยู่กับเงื่อนไขตอน runtime
(แบบ `make_calculator` ข้างบน) ใช้ `Box<dyn Fn(...)>` เป็นทางออกที่ตรงไปตรงมาที่สุด**

### 24.9 เบื้องหลัง Closure คือ Struct ที่ Compiler สร้างให้ (Desugaring Mental Model)

ตอนนี้เรารู้จัก `Fn`/`FnMut`/`FnOnce`, `move`, และวิธีส่ง/return closure ครบแล้ว มาถึงหัวข้อที่ทำให้ทุกอย่าง
"ไม่ใช่เรื่องมายากล" อีกต่อไป — **closure ที่ capture ตัวแปรจริง ๆ ถูก compiler แปลง (desugar) เป็นสิ่งที่
เทียบเท่ากับ struct ที่ไม่มีชื่อ (anonymous struct) เก็บตัวแปรที่ capture มาไว้เป็น field แล้ว implement trait
ที่เหมาะสม (`Fn`/`FnMut`/`FnOnce`) ให้มัน** เหมือนกับที่เราสร้าง struct + `impl` เองตั้งแต่ Part 9 และ 19

ลองเทียบ closure ที่ capture ตัวแปรกับ struct ที่เราเขียนขึ้นมาเองที่ทำหน้าที่เดียวกันเป๊ะ ๆ (หมายเหตุ: การ
`impl` trait `Fn`/`FnMut`/`FnOnce` ให้ type ของเราเองนั้น **เป็น unstable feature** ในภาษา Rust ปัจจุบัน — ต้อง
ใช้ nightly compiler เท่านั้น เพราะฉะนั้นตัวอย่างข้างล่างจะใช้ method ธรรมดาชื่อ `call()` แทน เพื่อจำลองพฤติกรรม
ให้เห็นภาพ โดยไม่ได้ implement trait ตัวจริงของ std):

```rust
fn main() {
    let offset = 10;

    // closure ตัวจริงที่เราคุ้นเคย — capture `offset` มาแบบยืม (&i32) เพราะ body แค่อ่านค่า
    let add_offset = |x: i32| x + offset;
    println!("closure จริง: {}", add_offset(5));

    // struct ที่เขียนขึ้นมาเอง จำลอง "สิ่งที่ compiler สร้างให้" กับ closure ตัวข้างบน
    struct AddOffset<'a> {
        offset: &'a i32, // เก็บตัวแปรที่ capture มา เป็น field ของ struct ตรง ๆ (ในที่นี้ยืมมาด้วย reference)
    }

    impl<'a> AddOffset<'a> {
        fn call(&self, x: i32) -> i32 {
            x + *self.offset
        }
    }

    let add_offset_struct = AddOffset { offset: &offset };
    println!("struct ที่จำลองไว้: {}", add_offset_struct.call(5));
}
```

```
closure จริง: 15
struct ที่จำลองไว้: 15
```

ทั้งสองวิธีให้ผลลัพธ์เดียวกันเป๊ะ (`15`) เพราะ **นี่คือสิ่งที่ compiler ทำให้กับ closure `add_offset` จริง ๆ
อยู่ข้างใต้อยู่แล้ว** เพียงแต่เราไม่เห็นมันตรง ๆ เท่านั้น มาดูภาพรวมของ "การแปล" (mapping) ระหว่างแนวคิด closure
กับ struct + trait ที่เรารู้จักมาแล้ว:

```text
closure ที่เราเขียน:                struct ที่ compiler สร้างให้ (แนวคิด):

let offset = 10;                    struct ClosureAtLine12<'a> {
let add_offset =                        offset: &'a i32,   // ← field สำหรับตัวแปรที่ capture มาแต่ละตัว
    |x: i32| x + offset;            }

                                     impl<'a> Fn<(i32,)> for ClosureAtLine12<'a> {
                                         type Output = i32;
                                         fn call(&self, (x,): (i32,)) -> i32 {
                                             x + *self.offset       // ← เนื้อ body ของ closure ยกมาตรง ๆ
                                         }
                                     }
                                     // เพราะ Fn: FnMut: FnOnce จึงได้ impl FnMut และ FnOnce มาด้วยโดยอัตโนมัติ

add_offset(5)                       add_offset.call((5,))
```

**อ่าน diagram นี้อย่างละเอียด**: สิ่งที่อยู่ระหว่าง `|...|` (parameter ของ closure) กลายเป็น parameter ของ
`call` method ตัวแปรที่ capture มาแต่ละตัว (`offset` ในที่นี้) กลายเป็น **field หนึ่ง field ของ struct** ที่
compiler สร้างขึ้นมา (ตั้งชื่อแบบไม่ให้ใครอ้างถึงได้ตรง ๆ จากภายนอก — นี่คือเหตุผลที่ type ของ closure แต่ละตัว
เรียกว่า **"anonymous type" หรือ "unique unnameable type"**) ส่วน body ของ closure ก็คือเนื้อ implementation
ของ `call` method (ในชื่อจริงคือ `call`, `call_mut`, หรือ `call_once` ขึ้นกับ trait) นั่นเอง

**นี่คือคำตอบว่าทำไม closure สองตัวที่ "หน้าตาเหมือนกัน" (เขียนโค้ดเหมือนกันเป๊ะ) แต่ประกาศแยกกันสองที่ ถือเป็น
คนละ type กัน**: เพราะแต่ละ closure literal ที่คุณเขียนในโค้ด compiler จะสร้าง struct ที่ไม่มีชื่อ **คนละตัว**
ให้เสมอ (คิดง่าย ๆ ว่าคือ struct ที่ตั้งชื่อจากตำแหน่งบรรทัด/คอลัมน์ในซอร์สโค้ด) แม้ field และ implementation
จะเหมือนกันเป๊ะก็ตาม — นี่คือเหตุผลที่ generic parameter `<F: Fn(i32) -> i32>` ใน `apply` (หัวข้อ 24.7) ต้อง
monomorphize สร้าง `apply` แยกกันสำหรับ closure แต่ละตัวที่ถูกส่งเข้ามา เหมือนกับที่ generic function ทั่วไป
monomorphize ให้กับ concrete type ต่างกันตามที่เรียนใน Part 18

**นี่ยังอธิบายเหตุผลของ capture mode สามแบบ (`Fn`/`FnMut`/`FnOnce`) ในเชิงกลไกได้ตรงที่สุด**:

- ถ้า field ของ struct ที่ compiler สร้างให้เก็บเป็น **`&T`** (reference ธรรมดา) → `call` เรียกผ่าน `&self`
  ได้พอ ไม่ต้องแก้ไขอะไร → **`Fn`**
- ถ้า field เก็บเป็น **`&mut T`** (mutable reference) → `call_mut` ต้องเรียกผ่าน `&mut self` เพื่อให้แก้ไข
  ค่าที่ field ชี้ไปได้ → **`FnMut`**
- ถ้า field เก็บเป็น **`T`** ตรง ๆ (owned value) และ body ต้อง move มันออกไปใช้ → `call_once` ต้องรับ `self`
  by value (ไม่ใช่ reference) เพื่อยึด struct ทั้งตัว (พร้อม field ข้างในมัน) มาทำลาย/ย้ายออกได้ → **`FnOnce`**

mental model นี้คือคำตอบที่แท้จริงว่าทำไมหัวข้อ 24.5.3 จึงเจอ error "**`consume` moved due to this call**"
เวลาเรียก `FnOnce` closure ครั้งที่สอง — เพราะการเรียก `call_once(self, ...)` ต้อง **move struct ทั้งตัว** เข้า
ไปในการเรียกจริง ๆ (เหมือนกับที่ฟังก์ชันไหนก็ตามที่รับ parameter แบบ `self` by value จะ move ค่านั้นเข้ามาแล้ว
ใช้ต่อไม่ได้ตามกฎ Part 6) closure จึงไม่เหลือให้เรียกซ้ำได้อีกหลังจากนั้น

### 24.10 เลือก Bound ให้ถูก: `FnOnce` vs `FnMut` vs `Fn` ตามจำนวนครั้งที่เรียก

จาก mental model ในหัวข้อ 24.9 เราสามารถสรุปกฎการเลือก trait bound เมื่อ**เขียนฟังก์ชันที่รับ closure**ได้อย่าง
ชัดเจน — คำถามหลักที่ต้องถามตัวเองคือ **"ฟังก์ชันของฉันจะเรียก closure ที่รับมากี่ครั้ง?"**

```rust
// เรียก closure แค่ครั้งเดียวเท่านั้น -> ใช้ FnOnce (ผ่อนคลายที่สุด รับ closure ได้ทุกแบบ)
fn call_with_value<F: FnOnce(i32) -> String>(f: F, value: i32) -> String {
    f(value) // เรียกครั้งเดียว แล้วจบ — ไม่มีการเรียก f ซ้ำที่ไหนในฟังก์ชันนี้เลย
}

// เรียก closure หลายครั้ง และ closure นั้นต้อง "จำ" สถานะข้ามการเรียกได้ (แก้ไขค่าที่ capture มา) -> ใช้ FnMut
fn call_repeatedly<F: FnMut() -> i32>(mut f: F, times: u32) -> Vec<i32> {
    let mut results = Vec::new();
    for _ in 0..times {
        results.push(f()); // เรียกซ้ำหลายครั้ง — ต้องใช้ FnMut ไม่ใช่ FnOnce
    }
    results
}

fn main() {
    let label = String::from("รายการ");
    // describe capture label มาแบบ move (เพราะ format! ต้องใช้ label เป็นเจ้าของ ผ่าน string interpolation)
    let describe = move |n: i32| format!("{label} หมายเลข {n}");
    println!("{}", call_with_value(describe, 7));

    let mut total = 0;
    // accumulate แก้ไข total ทุกครั้งที่ถูกเรียก -> ได้ FnMut พอดีกับที่ call_repeatedly ต้องการ
    let accumulate = || {
        total += 10;
        total
    };
    let history = call_repeatedly(accumulate, 4);
    println!("{history:?}");
}
```

```
รายการ หมายเลข 7
[10, 20, 30, 40]
```

สังเกตสิ่งสำคัญสองจุด: (1) `call_with_value` รับ bound `FnOnce` เพราะ body ของมันเรียก `f(value)` แค่ครั้งเดียว
เท่านั้น — ถ้าเราลองใส่ bound `Fn` แทนกับ `describe` (ที่ capture `label: String` มาแบบ move และใช้มันในการ
สร้าง `String` ใหม่ทุกครั้งผ่าน `format!`) ก็ยังทำงานได้อยู่ดี เพราะ `format!("{label} ...")` แค่ **อ่าน**
`label` (ไม่ได้ move มันออกไปจริง ๆ เพราะ `format!` ยืม `label` มา format เท่านั้น) แต่การใช้ `FnOnce` เป็น bound
คือ **การเลือกที่ผ่อนคลายที่สุดเท่าที่ฟังก์ชันต้องการจริง ๆ** ซึ่งคือ practice ที่ดี — เพราะฟังก์ชันที่ขอ
`FnOnce` รับ closure ได้ **กว้างที่สุด** (รับได้ทั้ง `Fn`, `FnMut`, และ `FnOnce` ตามลำดับชั้นในหัวข้อ 24.5.5)
ในขณะที่ฟังก์ชันที่ขอ `Fn` เข้มงวดที่สุด รับได้แค่ closure ที่ capture แบบอ่านอย่างเดียว

(2) `call_repeatedly` **ต้อง** ใช้ bound `FnMut` (ใช้ `FnOnce` ไม่ได้เลย) เพราะ body ของมันเรียก `f()` **ใน
loop หลายรอบ** — ถ้าลองเปลี่ยน bound เป็น `FnOnce` จะเจอ error ทันที เพราะ `FnOnce::call_once` ต้อง move
`self` เข้าไปทุกครั้งที่เรียก การเรียกในลูปหลายรอบจึงเป็นไปไม่ได้เลยในเชิง ownership (เหมือนที่เห็นจริงในหัวข้อ
24.5.3) สังเกตว่าเราต้องเขียน `mut f: F` ใน parameter ของ `call_repeatedly` ด้วย — เพราะ `FnMut::call_mut`
ต้องยืม `&mut self` ในการเรียกทุกครั้ง และการยืม `&mut` ได้ต้องมาจากตัวแปรที่เป็น `mut` เท่านั้น (เหมือนกับที่
ต้องเขียน `let mut increment` ในหัวข้อ 24.5.2) หลักการเดียวกันนี้ใช้ได้กับ parameter ของฟังก์ชันเช่นกัน

**กฎจำง่าย ๆ สรุปได้ว่า**: **เลือก `FnOnce` เมื่อฟังก์ชันเรียก closure แค่ครั้งเดียวแน่นอน (ผ่อนคลายที่สุด รับ
closure ได้กว้างที่สุด), เลือก `FnMut` เมื่อต้องเรียกหลายครั้งและ closure อาจต้องแก้ไขสถานะข้ามการเรียก, และ
เลือก `Fn` เมื่อต้องเรียกหลายครั้งพร้อมกัน (เช่นเก็บไว้ใน `Vec` แล้วเรียกวนหลายตัวพร้อมกันหรือเรียกจาก thread
หลายตัว) หรือเมื่อต้องการยืนยันกับผู้เรียกอย่างชัดเจนว่า closure จะไม่แก้ไขสถานะภายในเลย** ยิ่ง bound ผ่อนคลาย
เท่าไหร่ ฟังก์ชันของคุณก็ยิ่งรับ closure ได้หลากหลายมากขึ้นเท่านั้น — นี่คือหลักการทั่วไปเดียวกับการเลือก trait
bound ให้ "แคบที่สุดเท่าที่จำเป็น" ที่เรียนมาแล้วใน Part 18-19 (ขอ bound แค่พอที่ต้องใช้จริง อย่าขอเกินความ
จำเป็น เพราะจะจำกัดสิ่งที่ผู้เรียกส่งเข้ามาได้โดยไม่มีเหตุผล)

### 24.11 ตัวอย่างโลกจริง: Discount Rule Engine ด้วย `Vec<Box<dyn Fn(&Order) -> f64>>`

มาถึงตัวอย่างสุดท้ายที่รวมทุกอย่างที่เรียนมาในบทนี้เข้าด้วยกัน — ระบบ **"discount rule engine"** ที่พบได้จริงใน
ระบบ e-commerce: ธุรกิจต้องการกฎส่วนลดหลายแบบที่ **เปลี่ยนแปลงได้ตามการตั้งค่า** (configurable) เช่น "ลด 5%
ทุกออเดอร์", "ลดเพิ่ม 50 บาทถ้าซื้อครบ 1,000 บาท", "สมาชิกลดเพิ่มอีก 3%" — และต้องนำกฎเหล่านี้มา**คำนวณรวมกัน**
เพื่อได้ราคาสุทธิ

closure คือเครื่องมือที่เหมาะกับปัญหานี้พอดี เพราะแต่ละกฎมี **ค่า config เป็นของตัวเอง** (เปอร์เซ็นต์ส่วนลด,
เกณฑ์ขั้นต่ำ) ที่ต้อง "จำ" ไว้ใช้ตอนคำนวณจริง — นี่คือสิ่งที่ capturing (หัวข้อ 24.4) ทำได้ดีกว่าฟังก์ชันธรรมดา
ที่ต้องรับ config เป็น parameter เพิ่มทุกครั้งที่เรียก:

```rust
// โครงสร้างข้อมูลออเดอร์ที่กฎส่วนลดแต่ละตัวจะต้องอ่านไปตัดสินใจ
struct Order {
    subtotal: f64,
    is_member: bool,
}

// ฟังก์ชันโรงงาน (factory function) แต่ละตัว "สร้าง" closure ที่ capture ค่า config ของตัวเองไว้
// สังเกตว่าทุกฟังก์ชันคืน Box<dyn Fn(&Order) -> f64> — เพราะเราต้องเก็บกฎที่หน้าตาต่างกันไว้ใน Vec เดียวกัน
// (ตามที่เรียนไว้ในหัวข้อ 24.8: ต้อง return หลาย concrete type -> ใช้ Box<dyn Trait> ไม่ใช่ impl Trait)

fn percentage_discount(rate: f64) -> Box<dyn Fn(&Order) -> f64> {
    // rate ถูก capture มาแบบ move (ผ่าน move ||) เพื่อให้ closure นี้พกพา config ของตัวเองไปได้ตลอด
    Box::new(move |order: &Order| order.subtotal * rate)
}

fn threshold_discount(min_subtotal: f64, flat_amount: f64) -> Box<dyn Fn(&Order) -> f64> {
    Box::new(move |order: &Order| {
        if order.subtotal >= min_subtotal {
            flat_amount
        } else {
            0.0
        }
    })
}

fn member_discount(rate: f64) -> Box<dyn Fn(&Order) -> f64> {
    Box::new(move |order: &Order| {
        if order.is_member {
            order.subtotal * rate
        } else {
            0.0
        }
    })
}

// รวมส่วนลดจากทุกกฎเข้าด้วยกัน แล้วคำนวณราคาสุทธิ (ไม่ให้ต่ำกว่า 0)
fn final_price(order: &Order, rules: &[Box<dyn Fn(&Order) -> f64>]) -> f64 {
    let mut total_discount = 0.0;
    for rule in rules {
        total_discount += rule(order); // เรียก closure ผ่าน dyn Fn -- dynamic dispatch ตาม Part 19/21
    }
    (order.subtotal - total_discount).max(0.0)
}

fn main() {
    // ตั้งค่ากฎส่วนลดทั้งหมดของร้าน ณ ตอนนี้ — แต่ละตัวจำค่า config ของตัวเองไว้แล้วในตัวมันเอง
    let rules: Vec<Box<dyn Fn(&Order) -> f64>> = vec![
        percentage_discount(0.05),        // ลด 5% ทุกออเดอร์
        threshold_discount(1000.0, 50.0), // ซื้อครบ 1,000 บาท ลดเพิ่ม 50 บาท
        member_discount(0.03),            // สมาชิกลดเพิ่มอีก 3%
    ];

    let order = Order {
        subtotal: 1200.0,
        is_member: true,
    };

    let price = final_price(&order, &rules);
    println!("ยอดก่อนหักส่วนลด: {:.2} บาท", order.subtotal);
    println!("ราคาสุทธิหลังหักส่วนลดทุกกฎ: {price:.2} บาท");
}
```

```
ยอดก่อนหักส่วนลด: 1200.00 บาท
ราคาสุทธิหลังหักส่วนลดทุกกฎ: 1054.00 บาท
```

**อธิบายกลไกทีละส่วน โยงกลับไปยังทุกหัวข้อในบทนี้:**

- **`percentage_discount(0.05)` ฯลฯ คือฟังก์ชันโรงงาน (factory function)** — แต่ละครั้งที่เรียก มันจะสร้าง
  closure ตัวใหม่ที่ capture ค่า `rate`/`min_subtotal`/`flat_amount` ที่ส่งเข้ามาแบบ `move` (ตามหัวข้อ 24.6)
  เข้าไปเก็บไว้เป็นส่วนหนึ่งของ closure โดยตรง — นี่คือเหตุผลที่ **ต้องใช้ `move`** ในทุกฟังก์ชันโรงงานนี้:
  ค่า config เป็นตัวแปร parameter ในฟังก์ชันโรงงาน ซึ่งจะตายไปตอนฟังก์ชันจบ ถ้าไม่ `move` closure ที่ return
  ออกไปจะเจอ error E0373 แบบเดียวกับที่เห็นในหัวข้อ 24.6 ทันที
- **return type เป็น `Box<dyn Fn(&Order) -> f64>` ทุกฟังก์ชัน** — แม้ closure ข้างในแต่ละฟังก์ชันจะมี concrete
  type ต่างกัน (เพราะ capture ค่า config คนละชุดกัน ตามที่อธิบายไว้ใน desugaring model หัวข้อ 24.9) แต่ทุกตัว
  "หน้าตาเหมือนกันจากมุมมองภายนอก" คือ `Box<dyn Fn(&Order) -> f64>` เดียวกัน — ทำให้เก็บมันไว้ใน `Vec` เดียวกัน
  ได้ (`impl Fn(&Order) -> f64` ทำแบบนี้ไม่ได้เลย ตามกฎจากหัวข้อ 24.8 เพราะ `Vec<F>` ต้องมี `F` เป็น concrete
  type เดียวตายตัว)
- **closure แต่ละตัวรับ `&Order` (ยืม ไม่ยึด ownership)** — ออกแบบให้ closure แค่ "อ่าน" ข้อมูลของ order เพื่อ
  คำนวณส่วนลด ไม่จำเป็นต้องยึด ownership ของ `order` เลย ทำให้ `final_price` เรียก `rule(order)` ได้หลายครั้ง
  ในลูป (วนเรียกทุกกฎ) โดย `order` ยังใช้งานต่อได้ปกติทุกครั้ง — เหมือนกับหลักการ "รับข้อมูลมาแบบยืม ควรคืนหรือ
  ทำงานกับ `&T` ไม่ใช่ `T` ตรง ๆ" ที่ Part 11 สอนไว้ตั้งแต่หัวข้อ 11.2
- **`for rule in rules { total_discount += rule(order); }`** — แต่ละ `rule` มี type เป็น `&Box<dyn Fn(&Order)
  -> f64>` (ยืมมาจาก `rules` ที่รับเข้ามาแบบ `&[...]`) การเรียก `rule(order)` ทำงานผ่าน **dynamic dispatch**:
  runtime ต้องค้นหาผ่าน vtable ว่า closure ตัวนี้ (concrete type จริง ๆ ที่ซ่อนอยู่หลัง `dyn Fn`) implement
  `call` แบบไหน แล้วเรียกมัน — ต้นทุนนี้ยอมรับได้เพราะแลกกับความสามารถในการ**เพิ่ม/ลบ/เปลี่ยนกฎส่วนลดได้อย่าง
  ยืดหยุ่นตอน runtime** (เช่นโหลดกฎจาก config file หรือ database แล้วสร้าง closure ขึ้นมาตามนั้น) ซึ่งเป็น
  ความสามารถที่ generic + static dispatch ให้ไม่ได้เลย (เพราะ generic ต้องรู้ concrete type ทั้งหมดตั้งแต่
  compile time)
- **การคำนวณ**: `percentage_discount(0.05)` ให้ `1200.0 * 0.05 = 60.0`, `threshold_discount(1000.0, 50.0)`
  เช็คว่า `1200.0 >= 1000.0` (จริง) จึงให้ `50.0` ตรง ๆ, `member_discount(0.03)` เช็คว่า `order.is_member`
  เป็น `true` จึงให้ `1200.0 * 0.03 = 36.0` — รวมส่วนลดทั้งหมด `60.0 + 50.0 + 36.0 = 146.0` ราคาสุทธิจึงเป็น
  `1200.0 - 146.0 = 1054.0` ตรงกับผลลัพธ์ที่ได้

ตัวอย่างนี้แสดงให้เห็นว่าความรู้ทั้งหมดในบทนี้ทำงานร่วมกันได้อย่างไรในโค้ดจริง: **capturing** (แต่ละกฎจำ config
ของตัวเอง), **`move`** (ยึดค่า config ออกจากฟังก์ชันโรงงานที่กำลังจะจบ), **`Fn` trait** (กฎแต่ละตัวแค่อ่าน
`Order` ไม่แก้ไข จึงเรียกซ้ำได้หลายครั้งพร้อมกันในลูปได้อย่างปลอดภัย), และ **`Box<dyn Fn>`** (เก็บ closure ที่
concrete type ต่างกันไว้ใน collection เดียว)

### 24.12 ขนาดของ Closure ในหน่วยความจำ: Capture ยิ่งมาก ยิ่งใหญ่

หัวข้อ 24.9 บอกไว้ว่า closure ที่ capture ตัวแปรคือ struct ที่ไม่มีชื่อ เก็บตัวแปรที่ capture มาไว้เป็น field —
ถ้าเป็น struct จริง มันก็ต้อง **มีขนาด (size) ที่วัดได้จริง** เหมือน struct ทั่วไปที่เรียนมาตั้งแต่ Part 9 และ
ขนาดนั้นต้องขึ้นอยู่กับว่า field ข้างในเก็บอะไรบ้าง — หัวข้อนี้จะพิสูจน์ mental model นั้นด้วยตัวเลขจริงจาก
`std::mem::size_of_val` (เทคนิคเดียวกับที่ Part 11 หัวข้อ 11.3 ใช้พิสูจน์เรื่อง null pointer optimization ของ
`Option<&T>`)

```rust
fn main() {
    // closure ที่ไม่ capture อะไรจาก environment เลยแม้แต่ตัวเดียว
    let non_capturing = |x: i32| x + 1;

    // closure ที่ capture ตัวแปรเดียว เป็น i32 ผ่านการยืม (&i32 -- อ่านอย่างเดียว จึงได้ Fn)
    let a = 10;
    let capture_one_i32 = |x: i32| x + a;

    // closure ที่ capture String มาแบบ move -- เก็บ String ทั้งตัว (ไม่ใช่ &String) เป็น field ของตัวเอง
    let name = String::from("สมชาย");
    let capture_owned_string = move |greeting: &str| format!("{greeting} {name}");

    // closure ที่ capture ตัวแปรเล็ก ๆ สามตัว (u8 แต่ละตัว 1 ไบต์) มาแบบ move
    let b: u8 = 1;
    let c: u8 = 2;
    let d: u8 = 3;
    let capture_three = move |x: i32| x + (b as i32) + (c as i32) + (d as i32);

    println!("non_capturing          : {} ไบต์", std::mem::size_of_val(&non_capturing));
    println!("capture_one_i32 (&i32)  : {} ไบต์", std::mem::size_of_val(&capture_one_i32));
    println!("capture_owned_string    : {} ไบต์", std::mem::size_of_val(&capture_owned_string));
    println!("capture_three (3 x u8)  : {} ไบต์", std::mem::size_of_val(&capture_three));

    println!("เทียบ: size_of &i32     = {}", std::mem::size_of::<&i32>());
    println!("เทียบ: size_of String   = {}", std::mem::size_of::<String>());

    // เรียกใช้ทุกตัวจริง เพื่อไม่ให้ compiler เตือนว่าตัวแปรไม่ได้ถูกใช้
    println!("{} {} {} {}", non_capturing(1), capture_one_i32(1), capture_owned_string("hi"), capture_three(1));
}
```

ผลลัพธ์ (บนเครื่อง 64-bit ทั่วไป — ตัวเลขจริงอาจต่างกันเล็กน้อยตามรุ่น compiler แต่ความสัมพันธ์ระหว่างค่าจะเหมือนกัน
เสมอ):

```
non_capturing          : 0 ไบต์
capture_one_i32 (&i32)  : 8 ไบต์
capture_owned_string    : 24 ไบต์
capture_three (3 x u8)  : 3 ไบต์
เทียบ: size_of &i32     = 8
เทียบ: size_of String   = 24
1 20 hi สมชาย 7
```

**อ่านตัวเลขทีละบรรทัดโยงกลับไปยัง mental model ของหัวข้อ 24.9:**

- **`non_capturing` มีขนาด 0 ไบต์** — เพราะมันไม่ capture ตัวแปรใด ๆ จาก environment เลย struct ที่ compiler สร้าง
  ให้จึง **ไม่มี field เลยแม้แต่ field เดียว** ในภาษา Rust struct ที่ไม่มี field (หรือมี field ที่ล้วนไม่มีขนาด)
  เรียกว่า **zero-sized type (ZST)** — มันมีตัวตนในเชิง type system (compiler แยกแยะมันจาก closure ตัวอื่นได้)
  แต่ไม่ต้องใช้พื้นที่ memory จริงเลยสักไบต์ตอน runtime นี่คือเหตุผลเชิงวิศวกรรมที่ทำให้การ "ห่อ" logic สั้น ๆ
  ไว้ในรูป closure ที่ไม่ capture อะไร **ไม่มีต้นทุนด้าน memory เพิ่มขึ้นมาเลยแม้แต่นิดเดียว** เมื่อเทียบกับการ
  เรียกฟังก์ชันธรรมดาตรง ๆ
- **`capture_one_i32` มีขนาด 8 ไบต์ เท่ากับ `size_of::<&i32>()` เป๊ะ** — เพราะ field เดียวของมันคือ `&i32`
  (reference ไปยัง `a`) ขนาดของ closure จึงเท่ากับขนาดของ pointer หนึ่งตัวบนเครื่อง 64-bit พอดี ไม่มีอะไรซับซ้อน
  กว่านั้น
- **`capture_owned_string` มีขนาด 24 ไบต์ เท่ากับ `size_of::<String>()` เป๊ะ** — เพราะ `move` ทำให้ closure เก็บ
  `name: String` **ทั้งตัว** เป็น field ของมันตรง ๆ (ไม่ใช่ `&String` ที่จะมีขนาดแค่ 8 ไบต์) และ `String` เองมี
  โครงสร้างภายในสามส่วน (pointer ไปยังข้อมูลบน heap, ความยาวปัจจุบัน, capacity — รายละเอียดเต็มรูปแบบอยู่ใน
  Part 14) รวมกันเป็น 24 ไบต์บนเครื่อง 64-bit — นี่คือหลักฐานที่จับต้องได้ว่า `move` **เปลี่ยนสิ่งที่ closure
  เก็บไว้จริง ๆ** จาก "ที่อยู่ที่ชี้กลับไปยังเจ้าของเดิม" (เล็ก คงที่เสมอ) เป็น "ข้อมูลตัวจริงทั้งก้อน" (ขนาดแปรผัน
  ตามชนิดข้อมูลที่ยึดมา)
- **`capture_three` มีขนาดแค่ 3 ไบต์** — เพราะ `move` ทำให้มันเก็บ `b`, `c`, `d` (ตัวละ `u8` คือ 1 ไบต์) เป็น field
  ทั้งสามตัวตรง ๆ ไม่ใช่ reference (ซึ่งจะกิน 8 ไบต์ต่อตัวรวมเป็น 24 ไบต์ ใหญ่กว่าข้อมูลจริงที่ต้องการเก็บเสียอีก)
  — สังเกตว่ากรณีนี้ `u8` ไม่มีข้อกำหนดเรื่อง alignment ที่บีบให้ต้องเติม padding เพิ่ม (ต่างจากตัวเลขขนาดใหญ่กว่า
  เช่น `i32`/`i64` ที่อาจมี padding เข้ามาเกี่ยวข้อง) จึงได้ขนาดที่กระชับที่สุดเท่าที่เป็นไปได้จริง ๆ

**ข้อสรุปเชิงวิศวกรรมที่สำคัญที่สุดจากตัวเลขทั้งหมดนี้**: **"ขนาด" ของ closure ไม่ได้ขึ้นอยู่กับว่า body ของมันยาว
หรือซับซ้อนแค่ไหนเลย แต่ขึ้นอยู่กับ "จำนวนและชนิดของตัวแปรที่มัน capture มาเก็บเป็น field" เท่านั้น** closure ที่มี
body ยาวสิบบรรทัดแต่ไม่ capture อะไรเลยก็ยังมีขนาด 0 ไบต์เท่ากับ `non_capturing` เพราะ body ไม่ใช่ "data" ที่ต้อง
เก็บไว้ตอน runtime — มันคือ code ที่ถูก compile ไปเป็นคำสั่งของ CPU ตั้งแต่ตอน compile time ต่างหาก (เหมือนกับ
ฟังก์ชันธรรมดาทุกฟังก์ชัน) สิ่งที่กิน memory จริง ๆ มีแค่ field ที่ struct ต้องพกไปด้วยเท่านั้น — หลักการเดียวกัน
กับที่ Part 9 สอนไว้เรื่องขนาดของ struct ทั่วไป (ขึ้นอยู่กับ field ไม่ใช่จำนวนบรรทัดของ `impl` ที่ผูกกับมัน)

**ผลข้างเคียงที่น่าสนใจอีกอย่างของการที่ closure ไม่ capture อะไรเลยมีขนาด 0 ไบต์**: closure แบบนี้สามารถ
**coerce (แปลงชนิดโดยอัตโนมัติ) เป็น function pointer ธรรมดา (`fn(...) -> ...`) ได้โดยตรง** เพราะไม่มี "environment"
อะไรให้ต้องพกไปด้วยเลย — มันจึงมีหน้าตาเหมือน `fn` เปล่า ๆ ทุกประการในเชิง representation ตอน runtime:

```rust
fn main() {
    // closure ที่ไม่ capture อะไร -- assign ให้ตัวแปรที่ประกาศ type เป็น fn pointer ตรง ๆ ได้เลย
    let non_capturing: fn(i32) -> i32 = |x| x + 1;

    println!("{}", non_capturing(4));
    println!("size_of fn pointer = {}", std::mem::size_of_val(&non_capturing));
}
```

```
5
size_of fn pointer = 8
```

แต่ถ้า closure ตัวนั้น **capture** ตัวแปรอะไรมาแม้แต่ตัวเดียว การ coerce เป็น `fn` pointer แบบนี้จะทำไม่ได้ทันที
เพราะ `fn` pointer ตามนิยามของภาษา (function pointer ดิบ ๆ) **ไม่มีที่เก็บ "environment" ใด ๆ เลย** มันคือแค่
ที่อยู่ของ code ในหน่วยความจำเท่านั้น ไม่มีทางแปะข้อมูลเพิ่มเข้าไปได้:

```rust
fn main() {
    let a = 10;
    let capturing: fn(i32) -> i32 = |x| x + a; // ผิด -- closure ตัวนี้ capture `a` มาด้วย
    println!("{}", capturing(1));
}
```

```
error[E0308]: mismatched types
 --> src/main.rs:3:37
  |
3 |     let capturing: fn(i32) -> i32 = |x| x + a;
  |                    --------------   ^^^^^^^^^ expected fn pointer, found closure
  |                    |
  |                    expected due to this
  |
  = note: expected fn pointer `fn(i32) -> i32`
                found closure `{closure@src/main.rs:3:37: 3:40}`
note: closures can only be coerced to `fn` types if they do not capture any variables
 --> src/main.rs:3:45
  |
3 |     let capturing: fn(i32) -> i32 = |x| x + a;
  |                                             ^ `a` captured here
```

หมายเหตุ `note` บรรทัดสุดท้ายของ error นี้พูดตรงประเด็นที่สุด: **`closures can only be coerced to fn types if
they do not capture any variables`** — ยืนยัน mental model ของหัวข้อนี้ทั้งหมดในประโยคเดียว: closure ที่ไม่
capture อะไรเลยไม่มีความต่างจาก `fn` ธรรมดาแม้แต่นิดเดียวในเชิง representation ตอน runtime (ทั้งคู่คือแค่ "ที่อยู่
ของ code") แต่ทันทีที่ต้อง capture อะไรสักอย่าง มันต้องกลายเป็น "code + data" ที่ไม่มีทาง represent ด้วย function
pointer ดิบ ๆ ได้อีกต่อไป — ต้องใช้ type ของ closure เอง (ผ่าน generic, `impl Trait`, หรือ `dyn Trait` ตามที่
เรียนมาในหัวข้อ 24.7-24.8) เท่านั้น

### 24.13 เปรียบเทียบกับภาษาอื่น: Closure ใน C++, JavaScript, และ Python

ผู้เรียนที่มีพื้นฐานภาษาอื่นมาก่อนมักมี "ความคุ้นเคยผิด ๆ" เกี่ยวกับ closure ติดตัวมาโดยไม่รู้ตัว เพราะแต่ละภาษา
เลือกออกแบบเรื่อง capture ต่างกันมาก หัวข้อนี้จะเทียบให้เห็นชัดว่า Rust เลือกวิธีที่ **ปลอดภัยที่สุดในเชิง compile
time** แลกกับการที่ต้องเรียนรู้กฎ `Fn`/`FnMut`/`FnOnce`/`move` ที่ภาษาอื่นไม่มีให้เรียนเลย เพราะพวกเขาผลักปัญหา
เดียวกันนี้ไปเป็นพฤติกรรม runtime แทน

#### เทียบกับ C++: ต้องเขียน Capture List เอง — ผิดแล้วไม่ compile error แต่เป็น Undefined Behavior

C++ (ตั้งแต่ C++11) มี lambda expression ที่ต้อง **ระบุ capture mode เองตรง ๆ ทุกครั้งผ่าน capture list**
(สัญลักษณ์ `[...]` หน้า parameter list) ไม่มีการ "compiler อนุมานให้อัตโนมัติจาก body" แบบ Rust เลย:

```cpp
#include <iostream>
#include <string>

int main() {
    int threshold = 100;

    auto by_value  = [threshold](int price) { return price > threshold; };  // capture โดย copy ค่า
    auto by_ref     = [&threshold](int price) { return price > threshold; }; // capture โดย reference
    auto copy_all   = [=](int price) { return price > threshold; };          // capture ทุกตัวที่ใช้ โดย copy
    auto ref_all    = [&](int price) { return price > threshold; };          // capture ทุกตัวที่ใช้ โดย reference

    std::cout << by_value(150) << std::endl;
    return 0;
}
```

สังเกตว่าโปรแกรมเมอร์ C++ ต้อง **เลือกเองตรง ๆ** ว่าจะ capture แบบ copy (`[threshold]`, `[=]`) หรือแบบ reference
(`[&threshold]`, `[&]`) — ต่างจาก Rust ที่ compiler เลือก capture mode ที่ผ่อนคลายที่สุดให้เองจากการวิเคราะห์ body
(ตามหัวข้อ 24.5.4) ปัญหาที่ใหญ่กว่านั้นคือ **ถ้าโปรแกรมเมอร์เลือกผิด — capture โดย reference (`[&]`) ไปยังตัวแปร
local แล้ว return lambda ตัวนั้นออกจากฟังก์ชัน (สถานการณ์เดียวกับที่เราเห็นใน `make_printer` ในหัวข้อ 24.6) —
**C++ compiler ไม่มีทางจับได้ตอน compile time เลย** โปรแกรมจะ compile ผ่านอย่างราบรื่น แล้วไปพังตอน runtime แบบ
**undefined behavior** (อาจ crash, อาจได้ค่าขยะ, หรือที่แย่ที่สุดคือ "ดูเหมือนทำงานถูกต้อง" ในการทดสอบแต่พังตอน
production เพราะ timing/memory layout ต่างกันเล็กน้อย) นี่คือความต่างเชิงปรัชญาที่สำคัญที่สุดระหว่างสองภาษา: **สิ่ง
ที่ Rust บังคับให้แก้ตอน compile time ด้วย error E0373 (ตามหัวข้อ 24.6) คือบั๊กเดียวกันเป๊ะที่ C++ ปล่อยให้กลาย
เป็น dangling reference ที่ตรวจจับไม่ได้จนกว่าจะพังตอน runtime** — นี่คือเหตุผลที่กฎเรื่อง `move`/`Fn`/`FnMut`/
`FnOnce` ที่ดูยุ่งยากตอนเรียนใหม่ ๆ กลับกลายเป็นตาข่ายนิรภัยที่ C++ ไม่มีให้เลย

#### เทียบกับ JavaScript: Capture ตัวแปร (Binding) เสมอ ไม่ใช่ Capture ค่า — กับดัก `var` ใน Loop คลาสสิก

JavaScript closure **capture ตัวแปร (variable binding) ไม่ใช่ capture ค่า ณ ขณะนั้น** เสมอ ไม่มีแนวคิดเรื่อง
"ยืมแบบอ่านอย่างเดียว" กับ "ยึด ownership" แบบ Rust เลย — ทุก closure ใน JS มองเห็น "ช่องตัวแปรตัวเดียวกัน" ที่
มันถูกประกาศไว้ในสโคปเดิมเสมอ ถ้าตัวแปรนั้นถูกแก้ไขทีหลัง closure ทุกตัวที่ capture มันไว้จะเห็นค่าที่แก้ไขแล้ว
**ทั้งหมดพร้อมกัน** นี่คือต้นเหตุของบั๊กคลาสสิกที่โปรแกรมเมอร์ JS รุ่นเก่าเกือบทุกคนเคยเจอ:

```javascript
// ปัญหาคลาสสิกของ JavaScript รุ่นก่อน ES6 -- ใช้ var (function-scoped, ไม่ใช่ block-scoped)
var callbacks = [];
for (var i = 0; i < 3; i++) {
  callbacks.push(function () {
    console.log(i); // capture ตัวแปร i ตัวเดียวกันทุก closure ไม่ใช่ capture ค่า ณ ตอนสร้าง
  });
}
callbacks.forEach(function (cb) { cb(); });
// ผลลัพธ์จริง: 3, 3, 3  (ไม่ใช่ 0, 1, 2 ตามที่คนเขียนใหม่ ๆ คาดหวัง!)
// เพราะ i ตัวเดียวกันถูก capture ไว้ในทุก closure และ loop เพิ่มค่ามันไปจนครบ (=3) ก่อน callback จะถูกเรียก
```

ทุก closure ในลูปนี้ capture **ตัวแปร `i` ตัวเดียวกัน** (ไม่ใช่คนละสำเนา) เพราะ `var` ใน JS เป็น function-scoped
ไม่ใช่ block-scoped พอ loop จบ `i` มีค่าเป็น `3` และ closure ทั้งสามตัวก็เห็นค่า `3` เหมือนกันหมดตอนถูกเรียก
ทีหลัง — วิธีแก้ในยุค ES6 คือเปลี่ยนจาก `var` เป็น `let` (block-scoped ทำให้แต่ละรอบของลูปได้ตัวแปรคนละตัวกันจริง ๆ)

เทียบกับ Rust ที่ปัญหานี้ **ไม่มีทางเกิดขึ้นได้เลยตั้งแต่แรก** เพราะ closure ที่ capture ตัวแปรด้วยการ **ยืม**
(`Fn`/`FnMut`) จะยึด reference ไปยังตัวแปรต้นทางตัวเดียวกันก็จริง แต่ borrow checker (Part 7) จะไม่ยอมให้คุณ
เก็บ closure ที่ยืม `&mut` ตัวแปรเดียวกันไว้หลายตัวพร้อมกันเลยตั้งแต่ compile time (จะเจอ error ทันทีถ้าลองทำ) และ
ถ้าใช้ `move` (ตามหัวข้อ 24.6) แต่ละ closure ที่สร้างขึ้นในแต่ละรอบของ `for`/`.map()` จะได้ **สำเนาข้อมูลของตัวเอง**
คนละชุดจริง ๆ (ไม่ใช่ตัวแปรตัวเดียวกันที่ใครมาแก้ทีหลังก็เห็นหมด) มาดูโค้ด Rust ที่ทำงานแบบเดียวกันแต่ไม่มีกับดักนี้:

```rust
fn main() {
    let mut callbacks: Vec<Box<dyn Fn() -> i32>> = Vec::new();

    for i in 0..3 {
        // i แต่ละรอบของ for loop คือตัวแปรคนละตัวกันจริง ๆ ใน Rust (ไม่ใช่ตัวแปรเดียวกันแบบ var ใน JS)
        // move ยึดค่า i ของรอบนั้นเข้าไปเก็บในตัว closure โดยตรง -- แต่ละ closure จึงมีสำเนาของตัวเอง
        callbacks.push(Box::new(move || i));
    }

    for cb in &callbacks {
        println!("{}", cb());
    }
}
```

```
0
1
2
```

ผลลัพธ์ออกมาตามสัญชาตญาณเป๊ะ (`0, 1, 2`) เพราะทุกรอบของ `for` ใน Rust สร้างตัวแปร `i` คนละตัวกันจริง (ไม่ใช่แค่
คนละค่าของตัวแปรตัวเดียวกันแบบ `var` เก่าของ JS) และ `move` ทำให้ closure แต่ละตัว **ยึดสำเนาค่าของ `i` ณ รอบนั้น
ไปเก็บเป็นของตัวเองจริง ๆ** ไม่มีการ "แชร์ช่องตัวแปรเดียวกัน" ให้เกิดกับดักแบบ JS ได้เลยตั้งแต่ต้น

(ถ้าต้องการพฤติกรรมแบบ JS ที่ตั้งใจให้ closure หลายตัว **แชร์สถานะเดียวกันจริง ๆ** และแก้ไขมันร่วมกันได้ Rust ก็
ทำได้ แต่ต้องขอความช่วยเหลือจาก `Rc<RefCell<T>>` อย่างชัดเจน — ซึ่งเป็นการ "บอก type system ตรง ๆ ว่าฉันต้องการ
แชร์ ownership และ mutability ข้ามหลาย closure โดยตั้งใจ" ไม่ใช่ผลข้างเคียงที่เกิดขึ้นเองแบบ JS เราจะเจาะลึก
`Rc<RefCell<T>>` เต็มรูปแบบใน Part หลัง ๆ ของหลักสูตรที่พูดถึง smart pointer)

#### เทียบกับ Python: Late Binding เหมือน JavaScript และต้องมี `nonlocal` เพื่อแก้ไขตัวแปรนอก

Python closure มีพฤติกรรมคล้าย JavaScript รุ่นก่อน ES6 มาก คือ **capture ตัวแปรแบบ late binding** (อ้างถึงชื่อ
ตัวแปรในสโคปที่ล้อมรอบ ไม่ใช่ค่า ณ ตอนสร้าง closure) กับดักคลาสสิกที่คู่กันคือการสร้าง lambda หลายตัวในลูปเดียวกัน:

```python
callbacks = []
for i in range(3):
    callbacks.append(lambda: i)  # capture ชื่อ i ไม่ใช่ค่า -- ทุก lambda อ้างถึงตัวแปร i ตัวเดียวกัน

for cb in callbacks:
    print(cb())
# ผลลัพธ์จริง: 3, 3, 3  (เหมือนกับปัญหา var ใน JavaScript เป๊ะ เพราะ Python ก็ late-bind ชื่อตัวแปรเหมือนกัน)

# วิธีแก้แบบดั้งเดิมใน Python: บังคับให้ capture ค่า ณ ขณะนั้นผ่าน default argument
fixed_callbacks = []
for i in range(3):
    fixed_callbacks.append(lambda i=i: i)  # default argument ถูกประเมินค่าตอนสร้าง lambda เท่านั้น

for cb in fixed_callbacks:
    print(cb())
# ผลลัพธ์: 0, 1, 2
```

สังเกตว่า Python ไม่มีวิธี "บอก compiler ให้ capture by value" ตรง ๆ แบบ `move` ของ Rust เลย โปรแกรมเมอร์ Python
ต้องใช้ **workaround** ผ่าน default argument (`lambda i=i: i`) ซึ่งจริง ๆ แล้วไม่ใช่กลไก "closure capture"
เลยด้วยซ้ำ แต่อาศัยพฤติกรรมของ default argument ที่ถูกประเมินค่าแค่ครั้งเดียวตอนนิยามฟังก์ชัน (evaluation ตอน
def-time) มาแก้ปัญหาแทน — เป็นวิธีที่ต้อง "รู้เคล็ดลับ" ถึงจะนึกออก ไม่ได้เป็นส่วนหนึ่งของไวยากรณ์ closure ที่
ออกแบบมาให้ตรงประเด็นตั้งแต่แรกแบบ `move` ของ Rust

อีกจุดที่ต่างชัดเจนคือการ **แก้ไข** ตัวแปรจาก enclosing scope: Python ต้องประกาศ `nonlocal` ก่อนเสมอถ้าต้องการ
assign ค่าใหม่ให้ตัวแปรนอกจากภายใน closure (ไม่ใช่แค่อ่านเฉย ๆ):

```python
def make_counter():
    count = 0
    def increment():
        nonlocal count  # ถ้าไม่มีบรรทัดนี้ -- Python จะถือว่า count เป็นตัวแปร local ใหม่ในฟังก์ชัน increment แทน
        count += 1
        return count
    return increment

counter = make_counter()
print(counter())  # 1
print(counter())  # 2
print(counter())  # 3
```

ถ้าลืม `nonlocal count` ไป Python จะไม่ error ตอน define แต่จะ error ตอน**เรียกจริง**ด้วย
`UnboundLocalError: local variable 'count' referenced before assignment` เพราะ Python ตีความ `count += 1`
ว่ากำลังสร้างตัวแปร local ชื่อ `count` ใหม่ในฟังก์ชัน `increment` (ซึ่งยังไม่มีค่าเลยตอนอ่าน `count +=` ทำให้อ่าน
ก่อน assign ไม่ได้) — นี่คือ error ที่เจอตอน**รันจริง**เท่านั้น ต่างจาก Rust ที่ปัญหาแบบเดียวกัน (ลืมทำให้ closure
แก้ไขตัวแปรที่ capture มาได้) จะถูกจับตอน **compile time** เสมอ (ตามหัวข้อ 24.5.2 และกับดักข้อ 4 ท้ายบทนี้ — ลืม
`mut` บนตัวแปรที่เก็บ `FnMut` closure จะเจอ E0596 ตั้งแต่ยัง compile ไม่ผ่าน ไม่ต้องรอไปพังตอนรัน)

#### สรุปภาพรวมของการเปรียบเทียบ

| ภาษา | วิธี capture | ใครเป็นคนเลือก capture mode | ถ้าเลือกผิด/ลืม จะพังตอนไหน |
|---|---|---|---|
| **Rust** | ยืม (`&T`), ยืมแก้ไขได้ (`&mut T`), หรือยึด ownership (`T`) — บังคับด้วย `move` ได้ | **compiler** วิเคราะห์จาก body ให้อัตโนมัติ | **compile time** เสมอ (E0373/E0382/E0596 ฯลฯ) |
| **C++** | copy (`[x]`, `[=]`) หรือ reference (`[&x]`, `[&]`) | **โปรแกรมเมอร์** ต้องเขียน capture list เอง | **runtime** (undefined behavior ถ้าเลือกผิด — ไม่มี compile error เตือน) |
| **JavaScript** | capture ตัวแปร (binding) เสมอ ไม่มีให้เลือก | ไม่มีให้เลือก — เป็น late binding เสมอ | **runtime** (ค่าที่ได้ผิดจากที่คาด แต่โปรแกรมไม่ crash) |
| **Python** | capture ชื่อตัวแปร (late binding) เสมอ ต้อง `nonlocal` เพื่อแก้ไข | ไม่มีให้เลือก — ต้องใช้ workaround (default argument) เพื่อบังคับ capture by value | **runtime** (`UnboundLocalError` หรือค่าที่ได้ผิดจากที่คาด) |

ตารางนี้สรุปสิ่งที่บทนี้พยายามสอนมาตั้งแต่ต้นให้เห็นภาพเดียว: **สิ่งที่ทำให้ Rust closure "เรียนรู้ยากกว่า" ภาษา
อื่นตอนแรก (ต้องเข้าใจ `Fn`/`FnMut`/`FnOnce`, ต้องรู้ว่าเมื่อไหร่ต้องใส่ `move`) ที่จริงคือสิ่งเดียวกันกับสิ่งที่ทำให้
มันปลอดภัยกว่าอย่างมหาศาล** — ทุกกับดักที่ภาษาอื่นปล่อยให้เกิดตอน runtime (dangling reference ใน C++, ค่าตัวแปร
ที่ผิดจากที่คาดใน JS/Python) ถูก Rust ผลักให้กลายเป็น compile error ที่ต้องแก้ให้เรียบร้อยก่อนโปรแกรมจะรันได้เลย
แม้แต่ครั้งเดียว — สอดคล้องกับปรัชญาของทั้งหลักสูตรนี้ที่เห็นมาตลอดตั้งแต่ Part 6 (ownership), Part 7 (borrowing),
Part 11 (`Option<T>` แทน null), และ Part 20/23 (lifetimes): **ผลักปัญหาให้ compiler จับได้ก่อน ดีกว่าปล่อยให้มัน
กลายเป็นบั๊กที่ผู้ใช้จริงเจอ**

## กับดักที่พบบ่อย (Common Pitfalls)

**1. เรียก closure ตัวเดิมด้วย argument type คนละแบบ หลังจาก type ถูก "ล็อก" ไปแล้ว (E0308)**

ตามที่อธิบายไว้เต็มรูปแบบในหัวข้อ 24.3 — closure ที่ไม่ระบุ type ของ parameter ตรง ๆ จะถูก infer type ให้ตายตัว
จาก**การเรียกครั้งแรก** ไม่ใช่ generic ที่เปลี่ยน type ได้ตามการเรียกแต่ละครั้งแบบ generic function:

```rust
fn main() {
    let square = |x| x * x;
    let a = square(5);
    let b = square(5.0); // ล็อกเป็น i32 ไปแล้วจากบรรทัดก่อนหน้า -- ส่ง f64 เข้ามาไม่ได้
    println!("{a} {b}");
}
```

```
error[E0308]: mismatched types
  |
  |     let b = square(5.0);
  |             ------ ^^^ expected integer, found floating-point number
  |
note: expected because the closure was earlier called with an argument of type `{integer}`
```

**วิธีแก้**: ถ้าต้องการให้ทำงานได้กับหลาย type จริง ๆ ต้องเขียนเป็น generic function ที่รับ closure เป็น
parameter (ตามหัวข้อ 24.7) หรือระบุ type ให้ closure ชัดเจนตั้งแต่แรกด้วย type annotation (`|x: f64| x * x`)
เพื่อไม่ให้ compiler infer จากการเรียกครั้งแรกแบบไม่ตั้งใจ

**2. ลืม `move` เมื่อ Return Closure ที่ Capture ตัวแปร Local (E0373)**

ตามที่อธิบายไว้เต็มรูปแบบในหัวข้อ 24.6 — เมื่อ closure ต้องมีชีวิตอยู่นานกว่า scope ที่มันถูกสร้างขึ้น (เช่น
ถูก return ออกจากฟังก์ชัน) แต่ยัง capture ตัวแปร local แบบยืม (ไม่ใส่ `move`) จะเจอ:

```rust
fn make_printer() -> impl Fn() {
    let message = String::from("สวัสดี");
    || println!("{message}")
}
```

```
error[E0373]: closure may outlive the current function, but it borrows `message`,
which is owned by the current function
help: to force the closure to take ownership of `message` ..., use the `move` keyword
```

(หมายเหตุ: กรณีนี้ compiler รายงานเป็น **E0373** เพราะจับปัญหาได้ตั้งแต่ตอน type-check closure ก่อนถึงขั้น
borrow-check เต็มรูปแบบ — ในสถานการณ์ที่ใกล้เคียงกันแต่ซับซ้อนกว่า เช่นเมื่อ reference ถูกส่งผ่านหลายชั้นก่อน
ถึงจุดที่ตรวจพบ อาจเจอ **E0597 "borrowed value does not live long enough"** แทน ซึ่งมีสาเหตุร่วมเดียวกันคือ
ตัวแปรต้นทางตายก่อนสิ่งที่ยืมมันไว้ — ทั้งสอง error code นี้แก้ด้วยวิธีเดียวกันคือใส่ `move` หรือปรับให้ตัวแปร
ต้นทางมีอายุยาวพอ) **วิธีแก้**: เพิ่ม `move` หน้า closure (`move || println!("{message}")`) เพื่อให้ closure
ยึด ownership ของ `message` มาเก็บเป็นของตัวเอง ไม่ผูกกับ scope เดิมที่กำลังจะจบอีกต่อไป

**3. เรียก `FnOnce` Closure ซ้ำเป็นครั้งที่สอง (E0382)**

ตามที่อธิบายไว้เต็มรูปแบบในหัวข้อ 24.5.3 — closure ที่ body ของมัน move ตัวแปรที่ capture มาออกไปใช้ (เช่น
`let owned = name;`) จะ implement ได้แค่ `FnOnce` เท่านั้น เรียกได้ครั้งเดียว:

```rust
fn main() {
    let name = String::from("สมชาย");
    let consume = move || {
        let owned = name;
        println!("สวัสดี {owned}");
    };
    consume();
    consume(); // เรียกซ้ำ -- closure ทั้งตัวถูก move ไปแล้วตั้งแต่การเรียกครั้งแรก
}
```

```
error[E0382]: use of moved value: `consume`
note: closure cannot be invoked more than once because it moves the variable `name`
out of its environment
note: this value implements `FnOnce`, which causes it to be moved when called
```

**วิธีแก้**: ถ้าต้องการเรียก closure ซ้ำได้หลายครั้ง ต้องเปลี่ยนวิธีเขียน body ไม่ให้ move ค่าที่ capture มาออก
ไป — เช่นใช้ `.clone()` ก่อนใช้ (`let owned = name.clone();`) เพื่อให้ `name` ต้นฉบับยังอยู่ในความครอบครองของ
closure ต่อไปได้ (closure จะกลายเป็น `Fn` แทน เพราะแค่ยืมมา clone ไม่ได้ move ค่าจริงออกไป) หรือถ้าตั้งใจให้
เรียกได้ครั้งเดียวอยู่แล้วโดยธรรมชาติของ logic (เช่น callback ตอนปิดโปรแกรม, callback รับผลลัพธ์ครั้งเดียวจาก
การคำนวณแบบ async) การเหลือแค่ `FnOnce` ก็คือพฤติกรรมที่ถูกต้องแล้ว ไม่ต้องแก้ไขอะไร — ให้ตรวจสอบแค่ว่าโค้ด
ส่วนอื่นไม่ได้พยายามเรียกมันซ้ำโดยไม่ตั้งใจ

**4. ลืม `mut` บนตัวแปรที่เก็บ `FnMut` Closure (E0596)**

ตามที่อธิบายไว้ในหัวข้อ 24.5.2 — การเรียก `FnMut` closure ทุกครั้งต้องยืม `&mut self` ซึ่งต้องมาจากตัวแปรที่
ประกาศเป็น `mut` เท่านั้น ถ้าลืมใส่ `mut` ตอนประกาศ:

```rust
fn main() {
    let mut total = 0;
    let accumulate = || { // ลืม mut ตรงนี้ -- ควรเป็น let mut accumulate = ...
        total += 1;
        total
    };
    println!("{}", accumulate());
}
```

```
error[E0596]: cannot borrow `accumulate` as mutable, as it is not declared as mutable
  |
  |     total += 1;
  |     ----- calling `accumulate` requires mutable binding due to mutable borrow of `total`
...
  |     println!("{}", accumulate());
  |                    ^^^^^^^^^^ cannot borrow as mutable
help: consider changing this to be mutable
  |
  |     let mut accumulate = || {
```

**วิธีแก้**: เพิ่ม `mut` ตอนประกาศตัวแปรที่เก็บ closure (`let mut accumulate = || { ... }`) — สัญญาณเตือนที่
จำง่าย: ถ้า body ของ closure มีการ**แก้ไข**ตัวแปรที่ capture มา (`+=`, `.push()`, `*x = ...` ฯลฯ) แทบจะแน่นอน
เสมอว่าต้องประกาศตัวแปรที่เก็บ closure ตัวนั้นด้วย `mut` เสมอ (เหมือนกับกฎการประกาศตัวแปร `mut` ทั่วไปจาก
Part 3 แค่นำมาใช้กับตัวแปรที่เก็บ closure)

**5. ใช้ `dyn Fn(...)` เป็น Parameter Type โดยไม่มี `&` หรือ `Box` (E0277)**

ตามที่เชื่อมโยงกับความรู้เรื่อง dynamically sized type จาก Part 21 — `dyn Trait` เพียว ๆ (ไม่มี `&` หรือ `Box`
ครอบ) **ไม่มีขนาดตายตัวที่รู้ตอน compile time** จึงใช้เป็น parameter type ตรง ๆ ไม่ได้:

```rust
fn apply_dyn(f: dyn Fn(i32) -> i32, x: i32) -> i32 {
    f(x)
}
```

```
error[E0277]: the size for values of type `(dyn Fn(i32) -> i32 + 'static)` cannot be
known at compilation time
  |
  | fn apply_dyn(f: dyn Fn(i32) -> i32, x: i32) -> i32 {
  |                 ^^^^^^^^^^^^^^^^^^ doesn't have a size known at compile-time
  = help: the trait `Sized` is not implemented for `(dyn Fn(i32) -> i32 + 'static)`
help: function arguments must have a statically known size, borrowed types always
have a known size
  |
  | fn apply_dyn(f: &dyn Fn(i32) -> i32, x: i32) -> i32 {
```

**วิธีแก้**: ต้องครอบด้วย `&` (`&dyn Fn(i32) -> i32` — ยืมมา ขนาดตายตัวคือขนาดของ pointer + ข้อมูล vtable เสมอ
ไม่ว่า concrete type ข้างในจะใหญ่แค่ไหน) หรือครอบด้วย `Box` (`Box<dyn Fn(i32) -> i32>` — ยึด ownership บน heap
ขนาดของ `Box` เองก็ตายตัวเสมอด้วยเหตุผลเดียวกัน) ตามที่เรียนมาแล้วใน Part 19/21 — กฎง่าย ๆ ที่จำได้ทันที: **เห็น
`dyn Trait` โดด ๆ ไม่มี `&`/`Box`/`Rc` ครอบอยู่ ให้สงสัยว่าลืมใส่ pointer-like wrapper ไปข้างหน้ามันเสมอ**

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** เขียนฟังก์ชัน `fn make_multiplier(factor: i32) -> impl Fn(i32) -> i32` ที่ return closure ซึ่ง
   capture `factor` มาไว้ (ต้องใช้ `move` หรือไม่ ลองคิดดูว่าทำไม แล้วลองลบ `move` ออกดูว่า compiler บ่นอย่างไร
   ถ้ามี) แล้วทดสอบเรียก `make_multiplier(3)` เก็บไว้ในตัวแปร `triple` เรียก `triple(7)` และ `triple(10)` ดูว่า
   ได้ผลลัพธ์ตามที่คาดหรือไม่ (hint: closure ตัวนี้ implement `Fn` เพราะ body แค่คูณเลข ไม่ได้แก้ไขหรือ move
   `factor` ออกไปไหนเลย)

2. **[กลาง]** เขียนฟังก์ชัน `fn count_matching<F: Fn(&i32) -> bool>(numbers: &[i32], predicate: F) -> usize`
   ที่วนผ่าน `numbers` ทั้งหมดด้วย loop ธรรมดา (`for n in numbers`) นับจำนวนตัวที่ทำให้ `predicate(n)` เป็น
   `true` แล้วคืนจำนวนนั้น ทดสอบเรียกด้วย closure ที่ capture ตัวแปรจาก environment เช่น `let threshold = 10;
   count_matching(&data, |n| *n > threshold)` จากนั้นลองตอบคำถาม: ทำไม `predicate` ในตัวอย่างนี้ต้องเป็น
   `Fn(&i32) -> bool` ไม่ใช่ `FnMut`/`FnOnce` ทั้งที่ในทางเทคนิคใช้ `FnMut`/`FnOnce` เป็น bound ก็ compile ผ่าน
   เหมือนกัน (hint: คิดเรื่อง "bound ที่แคบที่สุดเท่าที่จำเป็น" จากหัวข้อ 24.10 — `Fn` สื่อความหมายกับผู้เรียก
   ได้ชัดเจนกว่าว่า predicate นี้ไม่มีทางมี side effect ที่แก้ไขสถานะภายในเลย)

3. **[ยาก]** เขียนฟังก์ชัน generic `fn retry<F, T, E>(mut action: F, attempts: u32) -> Result<T, E> where F:
   FnMut() -> Result<T, E>` ที่เรียก `action()` วนซ้ำสูงสุด `attempts` ครั้ง คืนค่า `Ok(value)` ทันทีที่เรียก
   สำเร็จครั้งแรก หรือคืน `Err` ของครั้งล่าสุดถ้าครบจำนวนครั้งแล้วยังไม่สำเร็จ (ใช้ตัวแปร `Option<E>` เก็บ error
   ล่าสุดไว้ระหว่างวน แบบเดียวกับที่ Part 11 สอนไว้) ทดสอบด้วย closure ที่ capture ตัวแปร `mut` เพื่อจำลอง
   "พยายามครั้งที่เท่าไหร่แล้ว" (เช่นนับถอยหลังจนถึง 0 ค่อยสำเร็จ) จากนั้นลองอธิบายว่าทำไม `retry` **ต้อง**
   ใช้ bound `FnMut` เท่านั้น ใช้ `FnOnce` แทนไม่ได้เลยในหลักการ (hint: ลองนึกภาพ error message ที่จะเจอถ้า
   ใช้ `FnOnce` แล้วพยายามเรียก `action()` ในลูปมากกว่าหนึ่งรอบ — มันจะหน้าตาคล้ายกับ error ในกับดักข้อ 3
   ของบทนี้)

4. **[ยาก/ประยุกต์ใช้งานจริง]** ขยายตัวอย่าง discount rule engine ท้ายบทนี้ (หัวข้อ 24.11): เขียนฟังก์ชัน
   `fn capped(rule: Box<dyn Fn(&Order) -> f64>, max_ratio: f64) -> Box<dyn Fn(&Order) -> f64>` ที่รับกฎ
   ส่วนลดหนึ่งตัวเข้ามา แล้ว **คืนกฎใหม่ที่ห่อกฎเดิมไว้** — กฎใหม่นี้เรียกกฎเดิมเพื่อคำนวณส่วนลดปกติก่อน แล้ว
   "จำกัดเพดาน" ไว้ไม่ให้เกิน `order.subtotal * max_ratio` (ใช้ `.min()` ของ `f64` เปรียบเทียบ) ทดสอบด้วยการ
   ครอบ `percentage_discount(0.5)` (ลด 50% ซึ่งสูงมาก) ด้วย `capped(..., 0.2)` (จำกัดไม่ให้ลดเกิน 20% ของ
   ยอดรวม) แล้วดูว่าผลลัพธ์ตรงกับเพดานที่ตั้งไว้หรือไม่ — ลองคิดต่อว่า `capped` เป็นตัวอย่างของ closure ที่
   **capture closure อีกตัวหนึ่ง** เข้ามาไว้ข้างใน (ไม่ใช่แค่ capture ค่าตัวเลข/string ธรรมดา) ทำไมสิ่งนี้ถึง
   ทำได้ในเชิง type system (hint: `Box<dyn Fn(&Order) -> f64>` ก็เป็นแค่ค่าตัวหนึ่งที่มี type แน่นอน ไม่มีอะไร
   ห้ามให้ closure ตัวใหม่ capture มันเหมือนตัวแปรทั่วไปเลย)

## สรุป

บทนี้เจาะลึก closure ที่เราใช้แบบไม่เป็นทางการมาตั้งแต่ Part 11 ให้เข้าใจกลไกจริงที่อยู่ข้างใต้อย่างครบถ้วน
เริ่มจาก syntax สามรูปแบบ (`|x| x + 1`, `|x: i32| -> i32 { x + 1 }`, และ block body) พร้อมกฎ type inference
ที่สำคัญที่สุด: closure ที่ไม่ระบุ type parameter จะถูก**ล็อก concrete type** ไว้ตายตัวจากการเรียกครั้งแรก
ต่างจาก generic function ที่ monomorphize ใหม่ได้ทุกครั้ง จากนั้นเจาะลึกความแตกต่างพื้นฐานที่สุดระหว่าง closure
กับ `fn`: **closure capture ตัวแปรจาก environment ได้ ในขณะที่ `fn` ทำไม่ได้เลย** (`E0434: can't capture
dynamic environment in a fn item`) นำไปสู่หัวใจของบทคือ trait สามตัว — `Fn` (capture by reference, เรียกซ้ำได้
ไม่จำกัด), `FnMut` (capture by mutable reference, เรียกซ้ำได้และแก้ไขสถานะได้), `FnOnce` (capture by value,
เรียกได้ครั้งเดียวเพราะการเรียกคือการ move closure ทั้งตัว) พร้อมย้ำว่า **compiler เป็นคนเลือก trait ให้เอง
จากการวิเคราะห์ body จริง ไม่ใช่สิ่งที่เราประกาศ** และลำดับชั้น `Fn: FnMut: FnOnce` ที่ทำให้ bound ที่ผ่อนคลาย
กว่ารับ closure ได้กว้างกว่า

เรียนรู้ `move` keyword สำหรับบังคับให้ closure ยึด ownership แทนการยืม โดยเฉพาะเมื่อ closure ต้องมีชีวิตอยู่
นานกว่า scope เดิม (return จากฟังก์ชัน, ส่งข้าม thread ที่จะเจาะลึกใน Part 37) พร้อม error จริง E0373 ตอนลืม
ใส่ จากนั้นดูสามวิธีรับ closure เป็น parameter (`<F: Fn(...)>`, `impl Fn(...)`, `&dyn Fn(...)`) และสองวิธี
return closure (`impl Fn(...)` สำหรับ concrete type เดียว, `Box<dyn Fn(...)>` สำหรับหลาย concrete type) ที่
เชื่อมโยงกับ static/dynamic dispatch จาก Part 19 และ 21 ตรง ๆ ปิดท้ายด้วย mental model สำคัญที่สุด: **closure
ที่ capture ตัวแปรคือน้ำตาลไวยากรณ์ของ struct ที่ไม่มีชื่อ เก็บตัวแปรที่ capture มาเป็น field แล้ว implement
trait ที่เหมาะสมให้** ทำให้ closure ไม่ใช่เรื่องมายากลอีกต่อไป และตัวอย่าง discount rule engine ท้ายบทแสดงให้
เห็นว่าทุกแนวคิดนี้ทำงานร่วมกันได้อย่างไรในระบบจริง

ใน **Part 25** เราจะเจาะลึก `Iterator` trait อย่างเป็นทางการ — ทำไม `for` loop ที่เราใช้มาตั้งแต่ Part 4 จริง ๆ
แล้ว desugar เป็นการเรียก `.next()` ซ้ำ ๆ, `IntoIterator`/`Iterator` ต่างกันอย่างไร, และวิธีเขียน custom
iterator ของเราเอง — ความรู้เรื่อง closure จากบทนี้จะสำคัญมากในบทหน้า เพราะ iterator adapter ส่วนใหญ่ (ที่จะ
เจาะลึกเต็มรูปแบบใน Part 26) อย่าง `.map()`, `.filter()`, `.fold()` รับ closure เป็น parameter ทั้งหมด และใช้
`FnMut`/`Fn` bound ตามหลักการเดียวกับที่เรียนไปแล้วในบทนี้ทุกจุด

---

**Part ก่อนหน้า:** [Lifetimes ขั้นสูง](part-023-lifetimes-advanced.md) | **Part ถัดไป:** [Iterators เบื้องต้น](part-025-iterators-basics.md)
