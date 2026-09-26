# Part 21: Traits ขั้นสูง (default methods, trait objects, dyn Trait)

> โมดูล: ระดับกลาง (Intermediate) | ระดับ: กลาง | เวลาโดยประมาณ: 200 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้ว่าทำไม generic (`Vec<T>`) ไม่สามารถเก็บ struct หลายชนิดที่ implement trait เดียวกันไว้ใน collection
  เดียวกันได้ และอ่าน compile error จริงที่เกิดขึ้นเมื่อพยายามทำแบบนั้นได้อย่างถูกต้อง
- ใช้ **trait object** (`&dyn Trait`, `Box<dyn Trait>`) เพื่อแก้ปัญหา heterogeneous collection ได้จริง และอธิบาย
  ความหมายของ keyword `dyn` ได้อย่างถูกต้อง
- อธิบายความแตกต่างเชิงลึกระหว่าง **static dispatch** (monomorphization จาก Part 18) กับ **dynamic dispatch**
  (`dyn Trait`) ได้ทั้งในมุมกลไกภายใน (**vtable**, **fat pointer**) และมุมต้นทุนด้าน performance ที่แท้จริง
- อธิบายกฎของ **object safety** (หรือชื่อใหม่ **dyn compatibility** ที่ compiler รุ่นใหม่ใช้) ได้ครบทั้งสามข้อหลัก
  พร้อมยกตัวอย่าง trait ที่ผิดกฎ อ่าน error E0038 จริง และรู้วิธีแก้ไขบางกรณีด้วย `where Self: Sized`
- แก้ปัญหาการ return หลาย concrete type ตามเงื่อนไข runtime ด้วย `Box<dyn Trait>` ได้อย่างมั่นใจ (ต่อยอดจากที่
  Part 19 เกริ่นไว้) และรู้ว่าเมื่อไหร่ `impl Trait` ยังใช้ได้อยู่ เมื่อไหร่ต้องเปลี่ยนไปใช้ trait object
- อธิบาย **supertrait** (`trait B: A`) ได้ถูกต้องว่าไม่ใช่ inheritance แบบ OOP แต่เป็นแค่ "ข้อบังคับเพิ่มเติม"
  ระดับสัญญา และเขียน default method ที่พึ่งพา method จาก supertrait ได้
- อธิบาย **orphan rule** พร้อม error E0117 จริง และใช้ **newtype pattern** แก้ปัญหานั้นได้อย่างถูกต้อง
- อ่านและเขียนโค้ดที่ใช้ `Box<dyn Error>` (ที่เคยใช้แบบ pragmatic มาตั้งแต่ Part 12) และ `Box<dyn Fn(...)>` ได้อย่าง
  เข้าใจกลไกเบื้องหลังอย่างถ่องแท้ ไม่ใช่แค่ "ใช้ตามที่เคยเห็น"
- ออกแบบระบบแบบ "plugin" ที่รับ type ใหม่เข้ามาได้เรื่อย ๆ โดยไม่ต้องแก้โค้ดเดิม ผ่าน `Vec<Box<dyn Trait>>` และรู้ว่า
  เมื่อไหร่ควรเลือก static dispatch เมื่อไหร่ควรเลือก dynamic dispatch ในสถานการณ์จริง

## ความรู้ที่ต้องมีมาก่อน

- **Part 19 (Traits เบื้องต้น) — จำเป็นที่สุดสำหรับบทนี้**: บทนี้ต่อยอดจาก Part 19 โดยตรงแบบไม่มีการสอนซ้ำ คุณต้อง
  แน่ใจว่าคุ้นเคยกับ: การนิยาม `trait` และ `impl Trait for Type`, **default method** (รวมถึง pattern ที่ default
  method เรียกใช้ method อื่นซึ่งยังไม่มี body — หัวข้อ 19.5), `&impl Trait` เทียบกับ `<T: Trait>`, multiple trait
  bounds และ `where` clause, และที่สำคัญที่สุดคือ **หัวข้อ 19.8-19.9** ที่ Part 19 เกริ่นไว้ว่า `impl Trait` ใน
  return position ต้อง return concrete type เดียวเสมอ พร้อม**เกริ่น** `Box<dyn Trait>` ไว้เป็นทางออกเบื้องต้น และ
  **หัวข้อ 19.14** ที่ใช้ระบบรูปทรงเรขาคณิต (`Shape`, `Circle`, `Rectangle`, `Triangle`) เป็นตัวอย่างหลัก — บทนี้
  จะใช้ระบบเดียวกันนี้ต่อเนื่อง ไม่แนะนำใหม่ ถ้าจำรายละเอียดพวกนี้ไม่ได้ ควรย้อนไปทวน Part 19 ก่อนอ่านบทนี้
- **Part 18 (Generics เบื้องต้น)**: จำเป็นสำหรับเข้าใจ **monomorphization** (หัวข้อ 18.7-18.8) ซึ่งเป็นกลไกของ
  static dispatch ที่บทนี้จะเทียบกับ dynamic dispatch ตลอดทั้งบท — Part 18 หัวข้อ 18.8 ยังเกริ่น `dyn Trait` ไว้สั้น ๆ
  ในฐานะ "ทางเลือกอื่นเมื่อขนาด binary สำคัญกว่าความเร็วดิบ" พร้อมบอกว่าจะเรียนเต็มรูปแบบในบทนี้ — **นี่คือบทนั้น**
- **Part 9 (Structs)**: newtype pattern (struct ที่ห่อ type เดียวรอบตัว) จะกลับมาใช้แก้ปัญหา orphan rule
- **Part 8 (Slices)**: จำเป็นสำหรับเข้าใจแนวคิด **fat pointer** (`&str` ที่เก็บ pointer + length) ซึ่งบทนี้จะเทียบ
  กับ `&dyn Trait` (`&dyn Trait` เก็บ pointer + vtable pointer) เพราะเป็นแนวคิด "reference ที่อ้วนกว่าปกติ" แบบ
  เดียวกันเป๊ะ เพียงข้อมูลเสริมที่พกไปต่างกัน
- **Part 12 (Result และ Error Handling)**: คุณใช้ `Box<dyn Error>` มาแล้วตั้งแต่หัวข้อ 12.7-12.8 แบบ pragmatic
  (รู้แค่ว่า "ใช้แก้ปัญหา error หลายชนิดได้") บทนี้จะอธิบายกลไกเบื้องหลังแบบเต็มรูปแบบว่ามันคือ trait object ธรรมดา
  ตัวหนึ่งเท่านั้นเอง ไม่มีอะไรพิเศษไปกว่า `Box<dyn Shape>` ที่กำลังจะเรียน
- **Part 15 (HashMap, HashSet)**: ใช้ทวนความเข้าใจเรื่อง iteration order ที่ไม่แน่นอนของ `HashMap` ในตัวอย่าง
  newtype pattern ท้ายบท

ถ้าให้สรุปภาพรวมสั้น ๆ ก่อนเริ่ม: **Part 19 สอนให้คุณ "เขียน trait ได้" และ "ใช้ trait เป็น bound ได้" ส่วน Part 21
นี้สอนกลไก "ทางเลือกที่สอง" ในการใช้ trait — ไม่ใช่ให้ compiler generate โค้ดแยกสำหรับทุก type (static dispatch)
แต่ให้ทุก type "พูดภาษาเดียวกัน" ผ่าน pointer ชนิดพิเศษที่ตัดสินใจตอน runtime (dynamic dispatch)** ทั้งสองทางเลือก
มีที่ใช้งานของตัวเอง และวิศวกร Rust ที่ดีต้องเลือกได้อย่างมีเหตุผล ไม่ใช่ใช้อย่างหนึ่งอย่างเดียวตลอดโดยไม่คิด

## เนื้อหา

### 21.1 ทวนสั้น ๆ จาก Part 19 และปัญหาที่ยังค้างอยู่

Part 19 สอนให้เราเห็นว่า struct ที่ไม่มีความสัมพันธ์กันเลย (เช่น `Circle`, `Rectangle`, `Triangle`) สามารถมี
"พฤติกรรมร่วม" ผ่าน `trait Shape` ได้ โดยไม่ต้องมี class inheritance แบบภาษา OOP ดั้งเดิม — เราเขียน `impl Shape for
Circle`, `impl Shape for Rectangle`, `impl Shape for Triangle` แยกกันสามครั้ง แล้วเขียนฟังก์ชันเดียว
(`fn print_shape_report(shape: &impl Shape)`) ที่ใช้ได้กับทุกรูปทรง — นี่คือ **polymorphism** ที่ไม่ต้องมี
inheritance เลย และ Part 19 ยังโชว์ให้เห็นแบบไว ๆ ว่า `Box<dyn Shape>` ทำให้เก็บรูปทรงหลายชนิดปนกันไว้ใน `Vec`
เดียวได้ พร้อมบอกไว้ตรง ๆ ว่า **"รายละเอียดเชิงลึกทั้งหมด — vtable, object safety, ต้นทุนด้าน performance ที่แท้จริง
— จะเป็นเนื้อหาหลักของ Part 21"** — บทนี้คือคำตอบเต็มรูปแบบของสิ่งที่ Part 19 ทิ้งไว้

ก่อนไปถึงคำตอบ มาดูปัญหาให้ชัดเจนอีกครั้งด้วยมุมมองที่ตรงไปตรงมาที่สุด: **ทำไม generic (`Vec<T>`) ถึงแก้ปัญหา
heterogeneous collection (collection ที่มีสมาชิกหลาย concrete type) ไม่ได้เลย**

ทวนจาก Part 18: `Vec<T>` คือ generic struct — เมื่อคุณเขียน `Vec<Circle>` compiler จะทำ **monomorphization**
(สร้างสำเนาโค้ดของ `Vec` เวอร์ชันที่ผูกกับ `Circle` แน่นอนแล้วตั้งแต่ compile time) พูดให้ตรงคือ **`T` ใน `Vec<T>`
ต้องเป็น "หนึ่งชนิดข้อมูลที่แน่นอน" เสมอ ไม่ใช่ "ชนิดข้อมูลใดก็ได้ที่ implement Shape สลับกันไปมาในแต่ละตำแหน่ง"**
มาดู error จริงเมื่อพยายามฝ่ากฎนี้:

```rust
// 21.1 - โค้ดที่ "ไม่" compile: พยายามเก็บ Circle และ Rectangle ไว้ใน Vec เดียวกัน
trait Shape {
    fn area(&self) -> f64;
    fn name(&self) -> &str;

    fn describe(&self) -> String {
        format!("{}: พื้นที่ {:.2} ตารางหน่วย", self.name(), self.area())
    }
}

struct Circle {
    radius: f64,
}

impl Shape for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.radius * self.radius
    }
    fn name(&self) -> &str {
        "วงกลม"
    }
}

struct Rectangle {
    width: f64,
    height: f64,
}

impl Shape for Rectangle {
    fn area(&self) -> f64 {
        self.width * self.height
    }
    fn name(&self) -> &str {
        "สี่เหลี่ยมผืนผ้า"
    }
}

fn main() {
    // Vec::new() ยังไม่รู้ว่าเก็บ type ไหน — แต่พอ push ตัวแรกเข้าไป compiler จะ "ล็อก" T ให้เป็น Circle ทันที
    let mut shapes = Vec::new();
    shapes.push(Circle { radius: 3.0 });
    // พยายาม push Rectangle เข้า Vec ที่ถูกล็อกเป็น Vec<Circle> ไปแล้ว
    shapes.push(Rectangle { width: 4.0, height: 5.0 });

    for s in &shapes {
        println!("{}", s.describe());
    }
}
```

Error จริงจาก compiler:

```
error[E0308]: mismatched types
  --> src/main.rs:43:17
   |
41 |     shapes.push(Circle { radius: 3.0 });
   |     ------      ---------------------- this argument has type `Circle`...
   |     |
   |     ... which causes `shapes` to have type `Vec<Circle>`
42 |     // พยายาม push Rectangle เข้า Vec ที่ถูกล็อกเป็น Vec<Circle> ไปแล้ว
43 |     shapes.push(Rectangle { width: 4.0, height: 5.0 });
   |            ---- ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected `Circle`, found `Rectangle`
   |            |
   |            arguments to this method are incorrect
```

**อ่าน error นี้ให้ทะลุ**: บรรทัด `shapes.push(Circle { radius: 3.0 })` ทำให้ compiler อนุมาน (infer) type ของ
`shapes` เป็น `Vec<Circle>` ไปเรียบร้อยแล้ว — ไม่ใช่ `Vec<อะไรก็ได้ที่ implement Shape>` แบบที่มือใหม่มักคาดหวัง
เมื่อบรรทัดต่อมาพยายาม push `Rectangle` เข้าไป compiler จึงปฏิเสธทันที เพราะ `Rectangle` ไม่ใช่ `Circle` — นี่คือ
**ข้อเท็จจริงพื้นฐานที่สุดของ generics**: `T` ใน `Vec<T>` ถูกตัดสินใจ (resolve) เป็น **concrete type เดียว**
ตั้งแต่จุดแรกที่คอมไพเลอร์อนุมานได้ และจะไม่เปลี่ยนแปลงอีกเลยตลอดอายุของตัวแปรนั้น ไม่ว่า `Circle` และ `Rectangle`
จะ implement `Shape` เหมือนกันแค่ไหนก็ตาม — **`T: Shape` บอกแค่ว่า "T ต้องทำอะไรได้บ้าง" ไม่ได้บอกว่า "T เปลี่ยนไป
เปลี่ยนมาได้ในแต่ละสมาชิกของ collection เดียวกัน"** ทั้งสองเรื่องนี้เป็นคนละเรื่องกันโดยสิ้นเชิง และนี่คือรากของ
ปัญหาที่เราเจอมาตั้งแต่หัวข้อ 19.1 (พยายามยัด `Product` กับ `Article` ไว้ใน array เดียวกัน) จนถึงตอนนี้

คำถามที่บทนี้จะตอบให้ครบคือ: **ในเมื่อ generic ทำไม่ได้ เรามีทางเลือกอื่นไหม ที่ยอมให้ collection เดียวเก็บหลาย
concrete type ได้จริง โดยยังเรียก method ร่วมผ่าน trait เดียวกันได้ตามปกติ?** คำตอบคือ **trait object** — เนื้อหา
หลักของหัวข้อถัดไป

### 21.2 Trait Object: `&dyn Trait` และ `Box<dyn Trait>`

**Trait object** คือค่าที่เก็บผ่าน pointer ชนิดพิเศษ ที่บอกแค่ว่า "ค่าที่ชี้ไปนี้ implement trait ตัวนี้แน่ ๆ"
โดย**ไม่ต้องรู้ concrete type ที่แน่นอนตั้งแต่ compile time** — keyword `dyn` (ย่อจาก **dynamic**) ที่นำหน้าชื่อ
trait คือสิ่งที่บอก compiler ว่า "ตำแหน่งนี้คือ trait object ไม่ใช่ generic type parameter ธรรมดา" ไวยากรณ์หลักมี
สองรูปแบบที่ใช้บ่อยที่สุด:

- **`&dyn Trait`** — reference (แบบยืม ไม่ยึดความเป็นเจ้าของ) ไปยังค่าที่ implement trait ใดก็ได้
- **`Box<dyn Trait>`** — กล่องบน heap (ยึดความเป็นเจ้าของ) ที่เก็บค่าที่ implement trait ใดก็ได้

```rust
// 21.2 - trait object: &dyn Shape และ Box<dyn Shape>
trait Shape {
    fn area(&self) -> f64;
    fn name(&self) -> &str;

    fn describe(&self) -> String {
        format!("{}: พื้นที่ {:.2} ตารางหน่วย", self.name(), self.area())
    }
}

struct Circle {
    radius: f64,
}

impl Shape for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.radius * self.radius
    }
    fn name(&self) -> &str {
        "วงกลม"
    }
}

struct Rectangle {
    width: f64,
    height: f64,
}

impl Shape for Rectangle {
    fn area(&self) -> f64 {
        self.width * self.height
    }
    fn name(&self) -> &str {
        "สี่เหลี่ยมผืนผ้า"
    }
}

struct Triangle {
    base: f64,
    height: f64,
}

impl Shape for Triangle {
    fn area(&self) -> f64 {
        0.5 * self.base * self.height
    }
    fn name(&self) -> &str {
        "สามเหลี่ยม"
    }
}

// รับ trait object ผ่าน reference: &dyn Shape — "อ้างอิงไปยังค่าอะไรก็ได้ที่ implement Shape"
// โดย concrete type จริงจะถูกตัดสินใจตอน runtime ไม่ใช่ compile time
fn print_report(shape: &dyn Shape) {
    println!("{}", shape.describe());
}

fn main() {
    let circle = Circle { radius: 3.0 };
    let rectangle = Rectangle { width: 4.0, height: 5.0 };

    // &circle ถูก "แปลง" เป็น &dyn Shape โดยอัตโนมัติ (unsized coercion) ตอนส่งเข้าฟังก์ชัน
    print_report(&circle);
    print_report(&rectangle);

    // ตอนนี้เก็บ Circle, Rectangle, Triangle ไว้ใน Vec เดียวกันได้แล้ว ผ่าน Box<dyn Shape>
    let shapes: Vec<Box<dyn Shape>> = vec![
        Box::new(Circle { radius: 2.0 }),
        Box::new(Rectangle { width: 3.0, height: 3.0 }),
        Box::new(Triangle { base: 4.0, height: 6.0 }),
    ];

    for shape in &shapes {
        println!("{}", shape.describe());
    }

    println!(
        "จำนวนรูปทรงทั้งหมดใน Vec เดียว (ต่าง concrete type กัน): {}",
        shapes.len()
    );
}
```

ผลลัพธ์:

```
วงกลม: พื้นที่ 28.27 ตารางหน่วย
สี่เหลี่ยมผืนผ้า: พื้นที่ 20.00 ตารางหน่วย
วงกลม: พื้นที่ 12.57 ตารางหน่วย
สี่เหลี่ยมผืนผ้า: พื้นที่ 9.00 ตารางหน่วย
สามเหลี่ยม: พื้นที่ 12.00 ตารางหน่วย
จำนวนรูปทรงทั้งหมดใน Vec เดียว (ต่าง concrete type กัน): 3
```

**ทำไม `Vec<Box<dyn Shape>>` ทำได้ในขณะที่ `Vec<T: Shape>` ทำไม่ได้**: คำตอบคือ **`Box<dyn Shape>` เป็น "หนึ่ง
concrete type" จริง ๆ ในสายตาของ compiler** — ไม่ว่าข้างในกล่องจะเป็น `Circle`, `Rectangle`, หรือ `Triangle` ก็ตาม
`Box<dyn Shape>` เองมี**ขนาดคงที่เท่ากันเสมอ** (รายละเอียดว่าทำไมขนาดคงที่ได้ อธิบายในหัวข้อ 21.3 เรื่อง fat pointer)
— `Vec<Box<dyn Shape>>` จึงเป็น `Vec` ของ type เดียว (`Box<dyn Shape>`) ตามกฎเดิมของ generics ทุกประการ **ไม่ใช่
ข้อยกเว้นของกฎ generics เลยแม้แต่นิดเดียว** เพียงแค่ `Box<dyn Shape>` เป็น "กล่องที่ซ่อนความแตกต่างของ concrete type
ไว้ข้างใน" เท่านั้นเอง — เปรียบเทียบง่าย ๆ ได้กับกล่องพัสดุที่หน้าตาข้างนอกเหมือนกันหมดทุกกล่อง (ขนาดเท่ากัน หยิบจับ
เหมือนกัน) แต่ข้างในบรรจุสินค้าต่างชนิดกันได้อย่างอิสระ

สังเกตรายละเอียดสำคัญอีกจุด: `print_report(&circle)` ส่ง `&Circle` เข้าไปในฟังก์ชันที่รับ `&dyn Shape` ได้โดยตรง
โดยไม่ต้องแปลงอะไรเอง — นี่คือ **unsized coercion** ที่ compiler ทำให้อัตโนมัติ (แปลง "reference ไปยัง concrete
type ที่ implement trait" ให้กลายเป็น "reference ไปยัง trait object ของ trait นั้น") จะเกิดขึ้นทุกครั้งที่ context
คาดหวัง `&dyn Trait`/`Box<dyn Trait>` และคุณส่งค่าที่ implement trait นั้นเข้าไป (ไม่ว่าจะเป็น reference ธรรมดา
หรือค่าที่ห่อด้วย `Box::new` ก็ตาม)

### 21.3 Static Dispatch เทียบกับ Dynamic Dispatch: กลไกภายในแบบเจาะลึก

นี่คือหัวใจสำคัญที่สุดของบทนี้ — ทวนจาก Part 18 และ Part 19 หัวข้อ 19.6 ว่า `&impl Trait`/`<T: Trait>` ใช้กลไกที่
เรียกว่า **static dispatch** ส่วน `dyn Trait` ใช้กลไกที่เรียกว่า **dynamic dispatch** ทั้งสองคำนี้บอกตรง ๆ ว่า
**"การตัดสินใจว่าจะเรียก method เวอร์ชันไหนของ type ไหน เกิดขึ้นตอนไหน"** — static dispatch ตัดสินใจ **ตอน compile
time (แบบคงที่)** ส่วน dynamic dispatch ตัดสินใจ **ตอน runtime (แบบเปลี่ยนแปลงได้)**

#### Static Dispatch: Monomorphization ทวนจาก Part 18

Part 18 หัวข้อ 18.7 สอนไว้ว่า เมื่อคุณเขียนฟังก์ชัน generic เช่น `fn total_area_static<T: Shape>(shapes: &[T]) ->
f64` แล้วเรียกใช้กับ `T = Circle` ที่หนึ่ง และ `T = Rectangle` ที่อีกที่หนึ่ง **compiler จะสร้างสำเนาโค้ดแยกกัน
สองเวอร์ชัน** เสมือนคุณเขียนฟังก์ชันสองตัวแยกกันด้วยมือ:

```
// สิ่งที่ compiler "เห็น" จริง ๆ หลัง monomorphization (ชื่อ mangled จริงจะซับซ้อนกว่านี้ แต่แนวคิดเดียวกัน)
fn total_area_static_Circle(shapes: &[Circle]) -> f64 { /* โค้ดที่ผูกกับ Circle แน่นอนแล้ว */ }
fn total_area_static_Rectangle(shapes: &[Rectangle]) -> f64 { /* โค้ดที่ผูกกับ Rectangle แน่นอนแล้ว */ }
```

เพราะโค้ดแต่ละเวอร์ชัน "รู้" concrete type ที่แน่นอนตั้งแต่ compile time การเรียก `s.area()` ข้างในจึงเป็นการเรียก
ฟังก์ชันตรง ๆ (**direct call**) ไปยัง `Circle::area` หรือ `Rectangle::area` ที่ compiler ทราบตำแหน่งแน่ชัดอยู่แล้ว
— ไม่มีการ "ค้นหา" หรือ "ตัดสินใจ" อะไรเกิดขึ้นตอนโปรแกรมรันจริงเลยแม้แต่นิดเดียว ซึ่งเปิดโอกาสให้ compiler ทำ
**inlining** ได้ด้วย (เอา body ของ `area()` มาแทนที่ตรงจุดที่เรียกเลย ไม่ต้องกระโดดไปที่อื่น) — นี่คือเหตุผลที่
static dispatch ถูกเรียกว่า **zero-cost abstraction**: โค้ดที่ compile ออกมาเร็วเท่า (หรือเร็วกว่า) การเขียนแยก
ฟังก์ชันด้วยมือทุกประการ ไม่มีต้นทุนซ่อนจากการใช้ trait/generic เลย

ข้อแลกเปลี่ยนที่ Part 18 หัวข้อ 18.8 อธิบายไว้แล้วคือ **code bloat**: ถ้าเรียก `total_area_static` กับ 10 concrete
type ที่ต่างกัน จะได้โค้ด 10 สำเนาแยกกันในตัว binary สุดท้าย — เร็วแต่ใหญ่

#### Dynamic Dispatch: Vtable คืออะไรกันแน่

ในทางกลับกัน เมื่อคุณเขียน `fn total_area_dynamic(shapes: &[Box<dyn Shape>]) -> f64` compiler **ไม่สามารถ**
generate โค้ดแยกสำหรับทุก concrete type ที่เป็นไปได้ เพราะ ณ จุดที่ compile ฟังก์ชันนี้ ยังไม่รู้เลยว่าจะมี type
ไหนบ้างมาเรียกใช้ (อาจมี type ใหม่ถูกเพิ่มเข้ามาในอนาคตด้วยซ้ำ ใน crate อื่นที่ยังไม่มีอยู่ตอนที่คุณเขียนฟังก์ชัน
นี้!) ทางออกของ Rust (และภาษาอื่นที่มี dynamic dispatch เช่น C++ virtual function, Java interface) คือ
**vtable** (**virtual method table** — "ตารางรายชื่อ method แบบ virtual")

แนวคิดของ vtable อธิบายง่าย ๆ ได้ว่า: **มันคือ struct ที่เก็บ "function pointer" (ตำแหน่งที่อยู่ในโค้ดของแต่ละ
method) ไว้เรียงกันเป็นตาราง** — สำหรับทุก `impl Shape for ConcreteType` หนึ่งชุด compiler จะสร้าง vtable
หนึ่งตัวขึ้นมาแยกกัน (ตอน compile time) ที่หน้าตาโดยแนวคิดประมาณนี้:

```
สำหรับ impl Shape for Circle จะได้ vtable หน้าตาแนวคิดประมาณนี้ (ไม่ใช่ syntax จริงที่เขียนได้ตรง ๆ):

ShapeVTableForCircle {
    area_fn:     ที่อยู่ของ Circle::area,
    name_fn:     ที่อยู่ของ Circle::name,
    describe_fn: ที่อยู่ของ Circle::describe (default หรือ override ก็ได้ ขึ้นกับว่า Circle เขียนทับไว้หรือไม่),
    drop_fn:     ที่อยู่ของฟังก์ชันที่ทำลาย Circle อย่างถูกต้อง (สำหรับ Box<dyn Shape> ที่ต้อง drop ค่าบน heap ให้ถูก),
    size, align: ขนาดและ alignment ของ Circle จริง ๆ (จำเป็นสำหรับ Box::new/drop จัดการ heap memory ถูกต้อง)
}

สำหรับ impl Shape for Rectangle ก็มี ShapeVTableForRectangle แยกต่างหากอีกชุดหนึ่ง โดยมี field ชุดเดียวกัน
(area_fn, name_fn, describe_fn, drop_fn, size, align) แต่ค่าข้างในชี้ไปยัง Rectangle::area, Rectangle::name ฯลฯ
แทน
```

เมื่อคุณเขียน `let s: Box<dyn Shape> = Box::new(Circle { radius: 2.0 });` สิ่งที่เกิดขึ้นจริงคือ Rust สร้าง**สอง
ส่วนประกอบ** ขึ้นมาพร้อมกัน:

1. **ข้อมูลจริงของ `Circle`** ถูกจัดเก็บบน heap (เหมือน `Box<Circle>` ธรรมดา)
2. **pointer ไปยัง vtable ของ `impl Shape for Circle`** ที่ compiler สร้างไว้ล่วงหน้าแล้วตั้งแต่ compile time
   (vtable ไม่ได้ถูกสร้างใหม่ตอน runtime — มันมีอยู่แล้วในตัว binary เป็น static data เพียงแค่ "ตัวเลือกว่าจะใช้
   vtable ตัวไหน" ถูกผูกไว้ตอนสร้างค่าเท่านั้นเอง)

แล้วเมื่อคุณเรียก `s.area()` สิ่งที่เกิดขึ้นตอน runtime คือ: **ไปดูใน vtable ที่ `s` ชี้อยู่ ว่า field `area_fn`
ชี้ไปยังฟังก์ชันไหน แล้วกระโดดไปเรียกฟังก์ชันนั้น (ผ่าน pointer โดยอ้อม — indirect call)** ต่างจาก static dispatch
ที่รู้ตำแหน่งฟังก์ชันตรง ๆ ตั้งแต่ compile time — นี่คือที่มาของคำว่า **dynamic**: การตัดสินใจ "จะเรียกฟังก์ชัน
ไหน" เกิดขึ้นจริง ๆ ตอนโปรแกรมรัน (ขึ้นกับว่า vtable ที่ pointer ชี้อยู่ ณ ขณะนั้นเป็นของ type ไหน) ไม่ใช่ตัดสินใจ
ไว้ล่วงหน้าตอน compile

#### Fat Pointer: `&dyn Trait` มีสอง Field ไม่ใช่หนึ่ง

ทวนจาก **Part 8 หัวข้อเรื่อง `&str`**: reference ธรรมดา (`&T`) คือ **"thin pointer"** — pointer เปล่า ๆ ตัวเดียว
ที่ชี้ไปยังตำแหน่งของค่า `T` ส่วน `&str` เป็น **"fat pointer"** เพราะ `str` เป็น unsized type การอ้างอิงไปยังมัน
ต้องพก **length** ไปด้วยเสมอ (pointer + length = 2 field) — `&dyn Trait` เดินตามหลักการเดียวกันเป๊ะ: `dyn Shape`
(ไม่มี `&` นำหน้า) ก็เป็น **unsized type** เช่นกัน (เพราะ `dyn Shape` อาจหมายถึง `Circle` หรือ `Rectangle` ที่มี
ขนาดต่างกัน compiler ไม่รู้ขนาดที่แน่นอนของ "ค่าที่อยู่ข้างหลัง trait object" ตั้งแต่ compile time) reference ที่
ชี้ไปยังมันจึงต้องพก**ข้อมูลเสริม**ไปด้วยเสมอ — เพียงแต่สิ่งที่พกไปไม่ใช่ length (เหมือน `&str`) แต่เป็น **pointer
ไปยัง vtable** ต่างหาก:

```
&Circle (thin pointer):      [data pointer]                          -> 1 field, 8 bytes บนเครื่อง 64-bit
&str    (fat pointer):       [data pointer][length]                   -> 2 field, 16 bytes
&dyn Shape (fat pointer):    [data pointer][vtable pointer]           -> 2 field, 16 bytes
```

มายืนยันด้วยขนาดจริงในหน่วยความจำ เหมือนที่ Part 8 ทำกับ `&str`:

```rust
// 21.3 - พิสูจน์ว่า &dyn Shape เป็น "fat pointer" (data pointer + vtable pointer)
trait Shape {
    fn area(&self) -> f64;
}

struct Circle {
    radius: f64,
}

impl Shape for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.radius * self.radius
    }
}

fn main() {
    // reference ธรรมดา (&Circle) คือ "thin pointer" — pointer เปล่า ๆ ตัวเดียว ชี้ไปยัง Circle บน stack/heap
    println!("size_of &Circle       = {} bytes", std::mem::size_of::<&Circle>());

    // &dyn Shape คือ "fat pointer" — สอง field: data pointer + vtable pointer
    println!("size_of &dyn Shape    = {} bytes", std::mem::size_of::<&dyn Shape>());

    // usize บนเครื่อง 64-bit คือ 8 bytes — ยืนยันว่า &dyn Shape มีขนาดเป็นสองเท่าของ pointer เปล่า ๆ พอดี
    println!("size_of usize         = {} bytes", std::mem::size_of::<usize>());

    // Box<Circle> เก็บแค่ pointer เดียวไปยัง heap (เพราะรู้ concrete type แน่นอนตั้งแต่ compile time)
    println!("size_of Box<Circle>   = {} bytes", std::mem::size_of::<Box<Circle>>());

    // Box<dyn Shape> ก็เป็น fat pointer เช่นกัน (data pointer ไปยัง heap + vtable pointer)
    // ขนาดเท่ากับ &dyn Shape เป๊ะ ๆ เพราะโครงสร้างข้อมูลเบื้องหลังเหมือนกัน (สองคำ machine word)
    println!("size_of Box<dyn Shape> = {} bytes", std::mem::size_of::<Box<dyn Shape>>());
}
```

ผลลัพธ์บนเครื่อง 64-bit:

```
size_of &Circle       = 8 bytes
size_of &dyn Shape    = 16 bytes
size_of usize         = 8 bytes
size_of Box<Circle>   = 8 bytes
size_of Box<dyn Shape> = 16 bytes
```

ตัวเลขนี้ยืนยันทุกอย่างที่อธิบายไปข้างบน: `&Circle`/`Box<Circle>` มีขนาด 8 bytes (thin pointer หนึ่งตัวบนเครื่อง
64-bit) ในขณะที่ `&dyn Shape`/`Box<dyn Shape>` มีขนาด 16 bytes เป๊ะ (สอง `usize` — data pointer + vtable pointer)
**นี่คือคำตอบที่สมบูรณ์ว่าทำไม `Box<dyn Shape>` มีขนาดคงที่ไม่ว่าข้างในจะเป็น `Circle` หรือ `Rectangle`**: เพราะ
`Box<dyn Shape>` เองไม่ได้เก็บข้อมูลของ `Circle`/`Rectangle` ไว้ตรง ๆ ในตัวมัน มันเก็บแค่ **สอง pointer** เท่านั้น
— data pointer ที่ชี้ไปยัง heap (ที่เก็บข้อมูลจริงของ `Circle` หรือ `Rectangle` ซึ่งจะมีขนาดต่างกันไปก็ไม่เป็นไร
เพราะอยู่คนละที่กับตัว `Box` เอง) และ vtable pointer ที่ชี้ไปยังตารางฟังก์ชันที่ถูกต้องสำหรับ concrete type นั้น

#### ต้นทุนที่แท้จริงของ Dynamic Dispatch

เทียบกับ static dispatch ที่ compiler รู้ทุกอย่างล่วงหน้าและ inline โค้ดได้เต็มที่ dynamic dispatch มีต้นทุนที่
จับต้องได้จริงสองส่วน:

1. **Indirect call**: การเรียก method ทุกครั้งต้อง "อ่าน vtable pointer ก่อน แล้วอ่าน function pointer จาก field
   ที่ถูกต้องใน vtable แล้วค่อยกระโดดไปเรียก" — เพิ่มการอ่าน memory หนึ่งถึงสองครั้งเทียบกับ direct call ที่กระโดด
   ไปตำแหน่งที่รู้แน่นอนอยู่แล้วทันที
2. **ไม่มี inlining**: เพราะ compiler ไม่รู้ ณ compile time ว่า vtable ที่จะถูกใช้จริงตอน runtime เป็นของ type
   ไหน มันจึง**เอา body ของ method มาแทนที่ตรงจุดเรียกไม่ได้เลย** (inline ได้ก็ต่อเมื่อรู้ concrete function
   ล่วงหน้าเท่านั้น) การที่ไม่มี inlining อาจทำให้พลาดโอกาสด้าน optimization อื่น ๆ ที่ compiler มักทำต่อจาก
   inlining ด้วย (เช่น loop unrolling, constant folding ข้าม method boundary)

ในทางปฏิบัติ **ต้นทุนนี้เล็กน้อยมากสำหรับโปรแกรมส่วนใหญ่** (การกระโดดผ่าน pointer เพิ่มหนึ่งครั้งใช้เวลาระดับ
nanosecond) และมักไม่ใช่ bottleneck จริงของระบบ ยกเว้นกรณีที่เรียก method ผ่าน `dyn Trait` ใน **hot loop** ที่รัน
นับล้านครั้งต่อวินาที (เช่น game engine ที่ประมวลผล entity นับพันตัวทุก frame) ซึ่งตอนนั้นถึงจะคุ้มค่าที่จะพิจารณา
เปลี่ยนกลับไปใช้ static dispatch แลกกับ code bloat ที่มากขึ้น — **กฎที่ใช้ได้จริงในงานส่วนใหญ่**: เริ่มจากเขียนโค้ด
ให้ถูกต้องและอ่านง่ายก่อน (เลือก `dyn Trait` เมื่อต้องการความยืดหยุ่นของ heterogeneous collection หรือ return
หลาย type) แล้ววัดผลจริงด้วยเครื่องมือ profiling ก่อนตัดสินใจเปลี่ยนไปใช้ static dispatch เพื่อความเร็ว — อย่า
ปรับให้เร็วขึ้นก่อนที่จะรู้ว่ามันช้าจริง (premature optimization)

### 21.4 Object Safety (Dyn Compatibility): ไม่ใช่ทุก Trait ที่ใช้เป็น `dyn Trait` ได้

Part 19 เกริ่นคำว่า **object safety** ไว้สองครั้งโดยยังไม่อธิบายรายละเอียด — ถึงเวลาอธิบายให้ครบ **ข้อเท็จจริงที่
สำคัญที่สุดในหัวข้อนี้คือ: ไม่ใช่ทุก trait ที่จะใช้เขียน `&dyn Trait` หรือ `Box<dyn Trait>` ได้** trait ต้องผ่าน
เงื่อนไขบางอย่างก่อน ที่เรียกว่า **object safety** — และ error message ของ compiler ในเวอร์ชันปัจจุบัน (rustc
1.94+) เปลี่ยนชื่อแนวคิดนี้เป็น **dyn compatibility** (**dyn-compatible**) แล้ว เป็นชื่อใหม่ของแนวคิดเดียวกันเป๊ะ
ที่สื่อความหมายชัดขึ้น (บทความ/หนังสือ/โค้ดเก่าจำนวนมากยังใช้คำว่า "object safety" อยู่ ทั้งสองคำหมายถึงสิ่งเดียวกัน)

**เหตุผลที่มีกฎนี้อยู่เลยคือเรื่องเดียวกับที่อธิบายไปในหัวข้อ 21.3**: vtable ต้องเป็น "ตารางที่มีจำนวน field
คงที่ รู้ล่วงหน้าตอน compile time" และทุก field ต้องเป็น **function pointer ที่เรียกได้โดยไม่ต้องรู้ concrete
type ที่แท้จริง** — ถ้า method ใน trait เขียนในรูปแบบที่ทำให้ประกอบ vtable แบบนั้นไม่ได้ trait นั้นก็ใช้เป็น
`dyn Trait` ไม่ได้เลย ไม่ว่าคุณจะพยายามแค่ไหนก็ตาม

กฎหลักสามข้อที่ทำให้ trait ไม่ dyn-compatible:

1. **มี method ที่ return `Self` โดยตรง (by value)** — เพราะ `Self` ในบริบทของ `dyn Trait` หมายถึง "concrete
   type จริงที่ไม่รู้ล่วงหน้า" ซึ่งมีขนาดไม่แน่นอน (อาจเป็น `Circle` หรือ `Rectangle` ก็ได้) แต่ vtable ต้องรู้ว่า
   จะจองพื้นที่คืนค่าขนาดเท่าไหร่แน่นอนตั้งแต่ compile time — return `Self` แบบ by value จึงขัดกับข้อกำหนดนี้
   โดยตรง
2. **มี method ที่เป็น generic** (`fn convert<T>(&self, ...)`) — เพราะ vtable ต้องมี field คงที่จำนวนหนึ่งต่อ
   method หนึ่งตัว แต่ method generic อาจถูกเรียกด้วย `T` ได้ไม่จำกัดจำนวนแบบ (ทุกครั้งที่เรียกด้วย `T` ต่างกัน
   จะได้ monomorphized version ใหม่) ทำให้ไม่มีทาง "จบ" รายการ field ใน vtable ได้เลย
3. **มี associated function ที่ไม่มี `self`/`&self`/`&mut self`** (เหมือน associated function ปกติที่เรียนมา
   ตั้งแต่ Part 9 — `fn make_default() -> Self`) — เพราะ vtable ผูกกับ **ค่าหนึ่งค่า** เสมอ (มันคือตารางของ
   "เมธอดที่เรียกกับค่านี้ได้") การเรียก associated function ที่ไม่มี `self` ไม่มี "ค่า" ให้ค้นหา vtable ที่ถูกต้อง
   จากที่ไหนเลย (จะรู้ได้อย่างไรว่าจะใช้ vtable ของ `Circle` หรือ `Rectangle` ในเมื่อไม่มี instance ให้ดูเลย)

มาดู error จริงของกฎข้อที่ 1 (พบบ่อยที่สุดในทางปฏิบัติ เพราะ `Clone`-like pattern เป็นที่นิยม):

```rust
// 21.4 - trait ที่ "ไม่ dyn-compatible": method คืนค่า Self แบบ by-value
trait Cloneable {
    // clone_it คืนค่าเป็น Self (concrete type ของตัวเอง) โดยตรง — นี่คือสิ่งที่ทำให้ trait นี้ใช้เป็น dyn ไม่ได้
    fn clone_it(&self) -> Self;
}

struct Widget {
    id: u32,
}

impl Cloneable for Widget {
    fn clone_it(&self) -> Self {
        Widget { id: self.id }
    }
}

// พยายามใช้ Cloneable เป็น trait object — ต้องล้มเหลวตอน compile
fn describe(item: &dyn Cloneable) {
    let _ = item;
}

fn main() {
    let w = Widget { id: 1 };
    describe(&w);
}
```

Error จริง:

```
error[E0038]: the trait `Cloneable` is not dyn compatible
  --> src/main.rs:18:20
   |
18 | fn describe(item: &dyn Cloneable) {
   |                    ^^^^^^^^^^^^^ `Cloneable` is not dyn compatible
   |
note: for a trait to be dyn compatible it needs to allow building a vtable
      for more information, visit <https://doc.rust-lang.org/reference/items/traits.html#dyn-compatibility>
  --> src/main.rs:4:27
   |
 2 | trait Cloneable {
   |       --------- this trait is not dyn compatible...
 3 |     // clone_it คืนค่าเป็น Self (concrete type ของตัวเอง) โดยตรง — นี่คือสิ่งที่ทำให้ trait นี้ใช้เป็น dyn ไม่ได้
 4 |     fn clone_it(&self) -> Self;
   |                           ^^^^ ...because method `clone_it` references the `Self` type in its return type
   = help: consider moving `clone_it` to another trait
   = help: only type `Widget` implements `Cloneable`; consider using it directly instead.
```

**อ่าน error นี้ให้ทะลุ**: บรรทัด `...because method 'clone_it' references the 'Self' type in its return type`
บอกตรงเป๊ะตามกฎข้อที่ 1 ข้างบน — สังเกตด้วยว่า compiler ยัง**แนะนำทางแก้มาให้เอง**สองแบบ (`consider moving
clone_it to another trait` และ `only type Widget implements Cloneable; consider using it directly instead`)
ซึ่งชี้ให้เห็นว่าในทางปฏิบัติจริง ถ้า method แบบนี้จำเป็นต้องมี มักจะแยกมันออกไปเป็น trait อีกตัวที่ไม่ต้องการ
dyn-compatible เลย (เพราะ `std::clone::Clone` ในความเป็นจริงก็เป็นแบบนี้เหมือนกัน — `Clone` ไม่ dyn-compatible
เพราะ `clone(&self) -> Self` ก็ return `Self` แบบเดียวกันนี้เอง นี่คือเหตุผลที่คุณเขียน `Box<dyn Shape + Clone>`
ตรง ๆ ไม่ได้)

กฎข้อที่ 2 และ 3 ก็ให้ error รูปแบบเดียวกัน เพียงเปลี่ยนเหตุผลตรงกลาง — ลองดูสั้น ๆ:

```rust
// กฎข้อ 2: generic method ทำให้ trait ไม่ dyn-compatible
trait Converter {
    fn convert<T: From<u32>>(&self, val: u32) -> T;
}
```

จะได้ error บอกว่า `...because method 'convert' has generic type parameters` แทน และ:

```rust
// กฎข้อ 3: associated function ที่ไม่มี self ทำให้ trait ไม่ dyn-compatible
trait Factory {
    fn make_default() -> Self;
}
```

จะได้ error บอกว่า `...because associated function 'make_default' has no 'self' parameter` แทน — สังเกตว่า
รูปแบบ error เหมือนกันทุกครั้ง (`the trait 'X' is not dyn compatible ... because method 'Y' ...`) เปลี่ยนแค่
เหตุผลท้ายประโยคตามกฎที่ถูกละเมิด ทำให้จำแนกปัญหาได้ง่ายมากเมื่อเจอ error นี้ในโค้ดจริง

#### เทคนิคขั้นสูง: `where Self: Sized` กันไม่ให้ Method หนึ่งตัวทำให้ Trait ทั้งก้อนใช้ `dyn` ไม่ได้

ข้อสังเกตที่น่าสนใจจาก error ของกฎข้อ 3 ข้างบนคือ compiler แนะนำทางแก้ที่สองมาด้วย: **`fn make_default() -> Self
where Self: Sized;`** — การเพิ่ม `where Self: Sized` บอก compiler ว่า **"method นี้เรียกได้เฉพาะตอนที่ Self เป็น
concrete type ที่รู้ขนาดแน่นอนแล้วเท่านั้น (ไม่ใช่ dyn Trait ที่เป็น unsized type)"** เมื่อบอกแบบนี้แล้ว compiler
จะ**ไม่รวม method นั้นเข้าไปใน vtable เลย** (เพราะมันรู้ว่าจะไม่มีทางถูกเรียกผ่าน `dyn Trait` อยู่แล้ว) ส่วน
method อื่นในสัญญาเดียวกันที่ dyn-compatible ก็ยังใช้งานผ่าน `dyn Trait` ได้ตามปกติ — พูดง่าย ๆ คือ **"ยกเว้น
method ที่มีปัญหาออกจาก vtable ไปเลย โดยไม่ต้องแยก trait ทั้งก้อน"**:

```rust
// เทคนิคขั้นสูง: ใช้ "where Self: Sized" กันไม่ให้ method ที่ไม่ dyn-compatible ทำให้ trait ทั้งก้อนใช้เป็น
// dyn ไม่ได้ — บอก compiler ว่า "method นี้เรียกได้เฉพาะตอนที่รู้ concrete type แน่นอนแล้วเท่านั้น (Self: Sized)"
// จึงไม่ต้องรวมมันเข้าไปใน vtable เลย ส่วน method อื่นที่ dyn-compatible ก็ยังเรียกผ่าน dyn Trait ได้ปกติ
trait Factory {
    fn describe(&self) -> String;

    // ต้องมี where Self: Sized เพราะ dyn Factory (unsized) ไม่มีทางสร้างค่า Self คืนกลับมาได้
    fn make_default() -> Self
    where
        Self: Sized;
}

struct Simple {
    id: u32,
}

impl Factory for Simple {
    fn describe(&self) -> String {
        format!("Simple #{}", self.id)
    }

    fn make_default() -> Self {
        Simple { id: 0 }
    }
}

fn describe_any(f: &dyn Factory) {
    println!("{}", f.describe());
}

fn main() {
    // make_default() เรียกผ่าน concrete type ตรง ๆ (Simple::make_default()) ได้ปกติ เพราะ Simple: Sized เสมอ
    let s = Simple::make_default();
    // describe() เรียกผ่าน &dyn Factory ได้ เพราะ describe ไม่มีเงื่อนไข Self: Sized และ dyn-compatible
    describe_any(&s);
}
```

ผลลัพธ์:

```
Simple #0
```

สังเกตว่าโปรแกรมนี้ compile ผ่านสมบูรณ์ ทั้งที่ `Factory` มี `make_default() -> Self` อยู่จริง เพราะ `where Self:
Sized` บอก compiler ชัดเจนแล้วว่า "อย่าพยายามเรียก `make_default` ผ่าน `dyn Factory` เลย เพราะมันไม่ได้ถูก
ออกแบบมาให้ทำแบบนั้น" — นี่คือรูปแบบที่ standard library ใช้จริงกับหลาย trait เช่น `Iterator::size_hint` เทียบกับ
บาง associated function อื่นที่ต้องการ `Self: Sized` โดยเฉพาะ

### 21.5 `impl Trait` เทียบกับ `Box<dyn Trait>` ใน Return Position: แก้ปัญหาที่ Part 19 ทิ้งไว้ให้ครบ

Part 19 หัวข้อ 19.8-19.9 โชว์ให้เห็นแล้วว่า `impl Trait` ใน return position ต้อง return concrete type เดียวเสมอ
และ `Box<dyn Trait>` คือทางแก้เมื่อต้อง return หลาย concrete type ตามเงื่อนไข runtime — มาดูสถานการณ์ที่สมจริงกว่า
เดิม เพื่อตอกย้ำหลักการนี้ด้วยตัวอย่างใหม่: **โรงงาน (factory) ที่สร้างรูปทรงจาก config string** ที่อ่านมาจาก
ภายนอกตอน runtime (เช่นไฟล์ตั้งค่า หรือ argument ของโปรแกรม — ไม่รู้ล่วงหน้าตอน compile time ว่าจะเป็นชนิดไหน):

```rust
// 21.5 - impl Trait ใน return position ล้มเหลวเมื่อต้อง return หลาย concrete type ตามเงื่อนไข runtime
trait Shape {
    fn area(&self) -> f64;
    fn name(&self) -> &str;
}

struct Circle {
    radius: f64,
}

impl Shape for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.radius * self.radius
    }
    fn name(&self) -> &str {
        "วงกลม"
    }
}

struct Rectangle {
    width: f64,
    height: f64,
}

impl Shape for Rectangle {
    fn area(&self) -> f64 {
        self.width * self.height
    }
    fn name(&self) -> &str {
        "สี่เหลี่ยมผืนผ้า"
    }
}

// โรงงาน (factory) สร้างรูปทรงจากชื่อ config ที่อ่านมาตอน runtime (เช่นจากไฟล์ config หรือ argument ผู้ใช้)
// พยายามใช้ impl Shape เป็น return type — ล้มเหลวเพราะสอง branch คืนค่าคนละ concrete type
fn shape_from_config(kind: &str, a: f64, b: f64) -> impl Shape {
    if kind == "circle" {
        Circle { radius: a }
    } else {
        Rectangle { width: a, height: b }
    }
}

fn main() {
    let s = shape_from_config("circle", 3.0, 0.0);
    println!("{} พื้นที่ {:.2}", s.name(), s.area());
}
```

Error จริง (สังเกตว่าเหมือนกับที่ Part 19 เจอเป๊ะ ๆ เพราะเป็นกฎเดียวกัน):

```
error[E0308]: `if` and `else` have incompatible types
  --> src/main.rs:40:9
   |
37 | /     if kind == "circle" {
38 | |         Circle { radius: a }
   | |         -------------------- expected because of this
39 | |     } else {
40 | |         Rectangle { width: a, height: b }
   | |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected `Circle`, found `Rectangle`
41 | |     }
   | |_____- `if` and `else` have incompatible types
   |
help: you could change the return type to be a boxed trait object
   |
36 - fn shape_from_config(kind: &str, a: f64, b: f64) -> impl Shape {
36 + fn shape_from_config(kind: &str, a: f64, b: f64) -> Box<dyn Shape> {
   |
help: if you change the return type to expect trait objects, box the returned expressions
   |
38 ~         Box::new(Circle { radius: a })
39 |     } else {
40 ~         Box::new(Rectangle { width: a, height: b })
   |
```

แก้ตามที่ compiler แนะนำ เปลี่ยน `-> impl Shape` เป็น `-> Box<dyn Shape>` แล้วห่อทุก branch ด้วย `Box::new(...)`
— คราวนี้ขยายให้เป็นโรงงานที่รองรับสามชนิดรูปทรง (คล้ายระบบ plugin ที่รับ "ชนิดใหม่" เข้ามาได้เรื่อย ๆ):

```rust
// 21.5 - แก้ด้วย Box<dyn Shape>: โรงงานสร้างรูปทรงจาก config string ได้จริงตามเงื่อนไข runtime
trait Shape {
    fn area(&self) -> f64;
    fn name(&self) -> &str;
}

struct Circle {
    radius: f64,
}

impl Shape for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.radius * self.radius
    }
    fn name(&self) -> &str {
        "วงกลม"
    }
}

struct Rectangle {
    width: f64,
    height: f64,
}

impl Shape for Rectangle {
    fn area(&self) -> f64 {
        self.width * self.height
    }
    fn name(&self) -> &str {
        "สี่เหลี่ยมผืนผ้า"
    }
}

struct Triangle {
    base: f64,
    height: f64,
}

impl Shape for Triangle {
    fn area(&self) -> f64 {
        0.5 * self.base * self.height
    }
    fn name(&self) -> &str {
        "สามเหลี่ยม"
    }
}

// เปลี่ยน return type เป็น Box<dyn Shape> — ตอนนี้ branch ไหนจะ return concrete type ไหนก็ได้
// ตราบใดที่ implement Shape ทั้งหมด เพราะ Box<dyn Shape> มีขนาดคงที่เสมอไม่ว่าข้างในจะเป็น type ไหน
fn shape_from_config(kind: &str, a: f64, b: f64) -> Box<dyn Shape> {
    match kind {
        "circle" => Box::new(Circle { radius: a }),
        "rectangle" => Box::new(Rectangle { width: a, height: b }),
        _ => Box::new(Triangle { base: a, height: b }),
    }
}

fn main() {
    // จำลองการอ่าน config มาจากภายนอกตอน runtime (เช่นไฟล์ตั้งค่า หรือ argument ของโปรแกรม)
    let configs = [("circle", 3.0, 0.0), ("rectangle", 4.0, 5.0), ("triangle", 6.0, 8.0)];

    for (kind, a, b) in configs {
        // แต่ละรอบของ loop สร้างรูปทรงคนละ concrete type กัน แต่ทุกตัวมี type เดียวกันจากมุมมองภายนอก
        // (Box<dyn Shape>) — เก็บไว้ในตัวแปรเดียว วนซ้ำ เรียก method เดียวกันได้หมด
        let shape = shape_from_config(kind, a, b);
        println!("{} พื้นที่ {:.2}", shape.name(), shape.area());
    }
}
```

ผลลัพธ์:

```
วงกลม พื้นที่ 28.27
สี่เหลี่ยมผืนผ้า พื้นที่ 20.00
สามเหลี่ยม พื้นที่ 24.00
```

**กฎการเลือกที่สรุปได้ชัดเจนแล้วในตอนนี้**: ใช้ `impl Trait` เมื่อฟังก์ชัน return concrete type **เดียวเสมอ**
ไม่ว่าจะซ่อนชื่อไว้แค่ไหนก็ตาม (เร็วที่สุด ไม่มี heap allocation, static dispatch เต็มรูปแบบ) — เปลี่ยนไปใช้
`Box<dyn Trait>` ทันทีที่ตรรกะของฟังก์ชันต้อง return **มากกว่าหนึ่ง concrete type** ขึ้นกับเงื่อนไขที่รู้แค่ตอน
runtime (ไม่ว่าจะเป็น `if`/`else`, `match`, loop ที่สร้างค่าคนละรอบกัน ฯลฯ) — ไม่มีทางเลี่ยงกฎนี้ได้เลยไม่ว่าจะพยายาม
เขียนอย่างไรก็ตาม เพราะเป็นข้อจำกัดพื้นฐานของการที่ compiler ต้องรู้ขนาด memory ของ return value ที่แน่นอนตั้งแต่
compile time เสมอ

### 21.6 Default Method ผ่าน Trait Object: พิสูจน์ว่า Dynamic Dispatch เรียก Method ของ "Type จริง" เสมอ

Part 19 หัวข้อ 19.5 สอน pattern สำคัญไว้แล้ว: default method เรียกใช้ method อื่นที่ยังไม่มี body (abstract
method) ได้ เพราะ compiler ค้ำประกันว่าไม่ว่า `Self` จะเป็น type ไหน มันต้องมี method นั้นแน่นอน — คำถามที่น่าสนใจ
คือ **เมื่อเรียก default method ผ่าน `&dyn Trait` แล้ว type ที่ implement จริง override บาง method ไว้ การเรียก
ผ่าน `dyn Trait` จะได้พฤติกรรมของ override นั้นหรือไม่ หรือจะได้ default ของ trait แทนเพราะ "รู้แค่ว่าเป็น dyn
Trait ไม่รู้ว่า override อะไรไว้"?**

คำตอบคือ **ได้พฤติกรรมของ override เสมอ 100%** — และนี่คือข้อพิสูจน์ที่เป็นรูปธรรมที่สุดว่า **dynamic dispatch
เรียก method ของ concrete type จริงที่อยู่หลัง trait object เสมอ ไม่ใช่เรียกจาก "trait ที่ประกาศไว้" แบบตายตัว**
เพราะ vtable ของแต่ละ type ถูกสร้างขึ้นจาก `impl Trait for ConcreteType` ของ**type นั้นโดยเฉพาะ** ไม่ว่า
ConcreteType จะเลือกใช้ default หรือ override ก็ตาม — field ใน vtable จะชี้ไปยังฟังก์ชันที่ถูกต้องสำหรับ type
นั้นเสมอ:

```rust
// 21.6 - default method ผ่าน trait object: พิสูจน์ว่า dynamic dispatch เรียก method ของ "concrete type จริง"
// เสมอ ไม่ใช่ default ของ trait แบบมึน ๆ ไม่สนใจว่า type ไหน override ไว้
trait Greeter {
    fn name(&self) -> String;

    // default method ที่เรียกใช้ self.name() (abstract) — เหมือน pattern จาก Part 19 หัวข้อ 19.5 ทุกประการ
    fn greet(&self) -> String {
        format!("สวัสดี {} ครับ/ค่ะ", self.name())
    }
}

struct Formal {
    name: String,
}

impl Greeter for Formal {
    fn name(&self) -> String {
        self.name.clone()
    }
    // ไม่ override greet() — ใช้ default ของ trait ตรง ๆ
}

struct Casual {
    name: String,
}

impl Greeter for Casual {
    fn name(&self) -> String {
        self.name.clone()
    }

    // Casual override greet() ทับ default ของ trait โดยสิ้นเชิง
    fn greet(&self) -> String {
        format!("เฮ้ {}!", self.name())
    }
}

// ฟังก์ชันนี้รู้จักแค่ "สัญญา" ของ Greeter เท่านั้น ไม่รู้เลยว่าข้างในเป็น Formal หรือ Casual
fn print_greeting(g: &dyn Greeter) {
    println!("{}", g.greet());
}

fn main() {
    // เก็บ Formal และ Casual ปนกันไว้ใน Vec<Box<dyn Greeter>> เดียว
    let people: Vec<Box<dyn Greeter>> = vec![
        Box::new(Formal { name: String::from("คุณสมชาย") }),
        Box::new(Casual { name: String::from("เอ็ม") }),
    ];

    for person in &people {
        // person มี type เป็น &Box<dyn Greeter> — ต่างจาก &Box<ConcreteType> ตรงที่ deref coercion จาก
        // &Box<dyn Greeter> ไปเป็น &dyn Greeter ตรง ๆ ในตำแหน่ง argument "ไม่" เกิดขึ้นอัตโนมัติ (รายละเอียด
        // เหตุผลอยู่ในกับดักที่ 5 ท้ายบท) ต้องเรียก .as_ref() เพื่อ "แปลง" &Box<dyn Greeter> เป็น &dyn Greeter
        // อย่างชัดเจนเสมอ
        print_greeting(person.as_ref());
    }
}
```

ผลลัพธ์:

```
สวัสดี คุณสมชาย ครับ/ค่ะ
เฮ้ เอ็ม!
```

**อธิบายผลลัพธ์**: `print_greeting` ทั้งสองครั้งเรียก `g.greet()` ผ่าน `&dyn Greeter` เหมือนกันทุกประการ (โค้ด
ข้างใน `print_greeting` เขียนครั้งเดียว ไม่รู้เลยว่ากำลังทำงานกับ `Formal` หรือ `Casual`) แต่ผลลัพธ์ต่างกันสิ้นเชิง
— `Formal` ได้ข้อความจาก **default** ของ trait (`"สวัสดี ... ครับ/ค่ะ"`) ในขณะที่ `Casual` ได้ข้อความจาก
**override** ของตัวเอง (`"เฮ้ ...!"`) — นี่คือเพราะตอนสร้าง `Box::new(Formal {...})` compiler ผูก vtable ของ
`impl Greeter for Formal` (ที่ field `greet_fn` ชี้ไปยัง default body ของ trait เพราะ `Formal` ไม่ override) ส่วน
`Box::new(Casual {...})` ผูก vtable ของ `impl Greeter for Casual` (ที่ field `greet_fn` ชี้ไปยัง override ของ
`Casual` เอง) ไว้ตั้งแต่ตอนสร้างค่า — เมื่อ `print_greeting` เรียก `g.greet()` มันแค่ **"ทำตามที่ vtable ของ
`g` บอก"** โดยไม่รู้และไม่จำเป็นต้องรู้ด้วยซ้ำว่ากำลังเรียก default หรือ override — vtable แต่ละอันถูกประกอบไว้
ถูกต้องสมบูรณ์แล้วตั้งแต่ compile time ของแต่ละ `impl` block

พูดให้กระชับที่สุด: **trait object ไม่ได้ "ลืม" ว่า type จริงคืออะไร มันแค่ "ซ่อน" ชื่อ concrete type จากผู้เรียก
เท่านั้น** แต่พฤติกรรมการทำงานจริงยังคงถูกต้อง 100% ตาม type จริงที่ซ่อนอยู่เสมอ — นี่คือหลักการเดียวกันกับ virtual
function ใน C++ หรือ interface method ใน Java/Go ทุกประการ (แนวคิด **runtime polymorphism** ที่พบในภาษา OOP
ส่วนใหญ่ เพียงแต่ Rust implement ผ่าน trait + explicit `dyn` ไม่ใช่ class hierarchy)

### 21.7 Supertraits: Trait ที่ต้องการ Trait อื่นเป็นเงื่อนไขก่อน

**Supertrait** คือ trait ที่ประกาศว่า "ชนิดข้อมูลใดก็ตามที่จะ implement เรา ต้อง implement trait อื่นตัวหนึ่ง
มาก่อนแล้วด้วย" ไวยากรณ์คือ `trait B: A { ... }` อ่านว่า **"B ต้องการ A เป็นเงื่อนไขก่อน"** — เรื่องสำคัญที่ต้อง
เข้าใจให้ถูกคือ **นี่ไม่ใช่ inheritance แบบ OOP** แม้ syntax จะคล้าย `class B extends A` ในภาษาอื่นก็ตาม ความ
แตกต่างที่สำคัญที่สุดคือ:

- **ไม่มีการแบ่งปัน field หรือข้อมูลใด ๆ ระหว่าง `A` และ `B`** — trait ไม่มี field ตั้งแต่แรกอยู่แล้ว (ทวนจาก
  Part 19: trait เป็นแค่สัญญาเรื่อง method ไม่ใช่โครงสร้างข้อมูล) supertrait จึงเป็นแค่ **"ข้อบังคับเพิ่มเติมระดับ
  สัญญา"** ไม่ใช่การสืบทอดโครงสร้างข้อมูลแบบ class
- **type หนึ่งต้อง `impl A for Type` และ `impl B for Type` แยกกันสองครั้งอย่างชัดเจน** — ไม่มีการ "ได้ A มาโดย
  อัตโนมัติ" เพียงเพราะ implement B (ต่างจาก class ที่ extend แล้วได้ method ของ parent class มาทันทีโดยไม่ต้อง
  ทำอะไรเพิ่ม)
- สิ่งที่ supertrait ให้ประโยชน์จริง ๆ คือ: **default method ของ `B` เรียกใช้ method ของ `A` ผ่าน `self` ได้อย่าง
  ปลอดภัย** เพราะ compiler ค้ำประกันแล้วว่า type ใดก็ตามที่ผ่านมาถึงจุดที่ implement `B` สำเร็จ ต้อง implement
  `A` ด้วยเสมอ — เป็น pattern เดียวกันกับ default method ที่เรียก abstract method ใน Part 19 หัวข้อ 19.5 เพียงแค่
  ขยายจาก "method อื่นในสัญญาเดียวกัน" ไปเป็น "method จากสัญญาอื่นทั้งก้อน"

```rust
// 21.7 - Supertrait: trait Greetable: Named บอกว่า "type ใดก็ตามที่ implement Greetable ต้อง implement Named
// มาก่อนแล้วด้วย" — ไม่ใช่ inheritance แบบ OOP (ไม่มีการแบ่งปัน field หรือ data ใด ๆ ระหว่างสอง trait)
// เป็นแค่ "ข้อบังคับเพิ่มเติม" ระดับสัญญาเท่านั้น
trait Named {
    fn name(&self) -> String;
}

// Greetable: Named คือ syntax ของ supertrait — อ่านว่า "Greetable ต้องการ Named เป็นเงื่อนไขก่อน"
trait Greetable: Named {
    // เพราะ trait นี้รู้แน่ ๆ ว่า Self ต้อง implement Named ด้วย (compiler ค้ำประกันไว้แล้ว)
    // default method จึงเรียก self.name() ได้เลย แม้ name() ไม่ได้ประกาศอยู่ใน Greetable เองเลยก็ตาม
    fn greet(&self) -> String {
        format!("สวัสดี {} ครับ/ค่ะ", self.name())
    }
}

struct Employee {
    full_name: String,
}

// ต้อง implement Named ให้ Employee ก่อน ไม่งั้น impl Greetable for Employee จะ compile ไม่ผ่าน
impl Named for Employee {
    fn name(&self) -> String {
        self.full_name.clone()
    }
}

impl Greetable for Employee {
    // ไม่ override greet() — ใช้ default ที่พึ่งพา Named::name() ตรง ๆ
}

fn main() {
    let emp = Employee { full_name: String::from("สมหญิง") };
    println!("{}", emp.greet());

    // เพราะ Employee implement Greetable ซึ่งมี Named เป็น supertrait อยู่แล้ว
    // trait object ที่ต้องการทั้งสอง trait พร้อมกันก็เขียนได้ด้วย + เหมือน multiple bound ปกติจาก Part 19
    let g: &dyn Greetable = &emp;
    println!("ผ่าน &dyn Greetable: {}", g.greet());
}
```

ผลลัพธ์:

```
สวัสดี สมหญิง ครับ/ค่ะ
ผ่าน &dyn Greetable: สวัสดี สมหญิง ครับ/ค่ะ
```

สังเกตว่า `&dyn Greetable` ใช้งานได้ปกติ (dyn-compatible) แม้ `Greetable` มี supertrait อยู่ก็ตาม — **supertrait
ไม่ทำให้ trait เสีย dyn compatibility เลย** ตราบใดที่ method ของทั้ง `Greetable` เองและ `Named` (supertrait) ยัง
ผ่านกฎสามข้อจากหัวข้อ 21.4 ปกติ — ในความเป็นจริง vtable ของ `dyn Greetable` จะรวม entry ของ method จาก **ทั้ง
`Greetable` และ `Named`** ไว้ในตารางเดียวกัน (คุณจึงเรียก `g.name()` ผ่าน `&dyn Greetable` ได้โดยตรงด้วย แม้
`name()` เป็น method ของ `Named` ไม่ใช่ของ `Greetable` เอง)

ลองดู error ที่เกิดขึ้นถ้า**ลืม** implement supertrait ก่อน:

```rust
trait Named {
    fn name(&self) -> String;
}

trait Greetable: Named {
    fn greet(&self) -> String {
        format!("สวัสดี {}", self.name())
    }
}

struct Employee {
    full_name: String,
}

// ลืม impl Named for Employee ก่อน
impl Greetable for Employee {}

fn main() {
    let emp = Employee { full_name: String::from("สมหญิง") };
    println!("{}", emp.greet());
}
```

```
error[E0277]: the trait bound `Employee: Named` is not satisfied
   |
16 | impl Greetable for Employee {}
   |                    ^^^^^^^^ unsatisfied trait bound
   |
help: the trait `Named` is not implemented for `Employee`
...
note: required by a bound in `Greetable`
   |
 5 | trait Greetable: Named {
   |                  ^^^^^ required by this bound in `Greetable`
```

**อ่าน error นี้ให้ทะลุ**: compiler ปฏิเสธตั้งแต่บรรทัด `impl Greetable for Employee {}` เลย ไม่ต้องรอให้เรียก
`.greet()` ก่อนด้วยซ้ำ — เพราะ **supertrait bound ถูกตรวจสอบทันทีที่จุด `impl` ไม่ใช่ตอนเรียกใช้** สื่อความหมาย
เดียวกันกับ trait bound ของ generic function ที่ Part 19 เรียนมา (`T: Summary + Display`) เพียงแค่ตอนนี้เงื่อนไข
ถูกเขียนไว้ที่**ตัว trait เอง** แทนที่จะเป็น**ฟังก์ชันที่รับ trait นั้นเป็น bound**

### 21.8 Newtype Pattern สำหรับ Orphan Rule: ทวนให้ครบพร้อมตัวอย่างใหม่

Part 19 หัวข้อ 19.13 อธิบาย **orphan rule** ไว้แล้วพร้อม error E0117 จริง — บทนี้จะทวนสั้น ๆ ด้วยตัวอย่างใหม่ที่
เกี่ยวกับระบบสินค้าคงคลัง (inventory) ซึ่งเป็นสถานการณ์จริงที่พบบ่อยกว่า: **อยากพิมพ์รายงานสต็อกสินค้าที่เก็บใน
`HashMap<String, u32>` ให้สวยงามผ่าน `{}` (trait `Display`) แต่ทั้ง `Display` และ `HashMap` เป็นของ std ทั้งคู่**:

```rust
use std::collections::HashMap;
use std::fmt;

// พยายาม implement std::fmt::Display (trait จาก std) ให้กับ HashMap<String, u32> (type จาก std เช่นกัน)
// ทั้งสองอย่างไม่ได้เป็นของ crate นี้เลยแม้แต่อย่างเดียว — ละเมิด orphan rule ทันที
impl fmt::Display for HashMap<String, u32> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        for (name, qty) in self {
            writeln!(f, "{name}: {qty} หน่วย")?;
        }
        Ok(())
    }
}

fn main() {
    let mut stock: HashMap<String, u32> = HashMap::new();
    stock.insert(String::from("เมาส์"), 10);
    println!("{stock}");
}
```

Error จริง:

```
error[E0117]: only traits defined in the current crate can be implemented for types defined outside of the crate
 --> src/main.rs:6:1
  |
6 | impl fmt::Display for HashMap<String, u32> {
  | ^^^^^^^^^^^^^^^^^^^^^^--------------------
  |                       |
  |                       `HashMap` is not defined in the current crate
  |
  = note: impl doesn't have any local type before any uncovered type parameters
  = note: for more information see https://doc.rust-lang.org/reference/items/implementations.html#orphan-rules
  = note: define and implement a trait or new type instead
```

**ทวนกฎสั้น ๆ**: orphan rule อนุญาตให้ `impl TraitX for TypeY` ได้ก็ต่อเมื่อ **`TraitX` หรือ `TypeY` (อย่างน้อย
หนึ่งในสอง) เป็นของ crate ปัจจุบัน** — ในตัวอย่างนี้ทั้ง `Display` และ `HashMap` เป็นของ std ทั้งคู่ ไม่มีอันไหน
เป็นของเราเลย จึงถูกปฏิเสธ (เหตุผลเชิงลึกเรื่อง coherence อธิบายไว้ครบแล้วใน Part 19) — วิธีแก้ที่ standard และ
compiler แนะนำมาให้เองคือ **newtype pattern**: ห่อ `HashMap<String, u32>` ด้วย struct ของเราเอง เพื่อให้มี "type
ที่เป็นของเรา" อยู่ในสมการ:

```rust
use std::collections::HashMap;
use std::fmt;

// newtype pattern: ห่อ HashMap<String, u32> ด้วย struct ของเราเอง — InventoryReport เป็น local type
// (เป็นของ crate นี้) แม้ข้างในจะเก็บ HashMap ที่เป็นของ std ก็ตาม orphan rule ผ่านได้เพราะ
// "อย่างน้อยหนึ่งใน trait/type ต้องเป็นของ crate ปัจจุบัน" — ตอนนี้ type (InventoryReport) เป็นของเราแล้ว
struct InventoryReport(HashMap<String, u32>);

impl fmt::Display for InventoryReport {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        for (name, qty) in &self.0 {
            writeln!(f, "{name}: {qty} หน่วย")?;
        }
        Ok(())
    }
}

fn main() {
    let mut stock: HashMap<String, u32> = HashMap::new();
    stock.insert(String::from("เมาส์"), 10);
    stock.insert(String::from("คีย์บอร์ด"), 5);

    let report = InventoryReport(stock);
    print!("{report}");
}
```

ผลลัพธ์ (หมายเหตุ: ลำดับที่แสดงเป็นเพียงตัวอย่างหนึ่งที่เป็นไปได้เท่านั้น เพราะ `HashMap` ไม่การันตีลำดับการ
iterate เลย ทวนจาก Part 15):

```
คีย์บอร์ด: 5 หน่วย
เมาส์: 10 หน่วย
```

**สิ่งที่เกิดขึ้นเบื้องหลัง**: `InventoryReport` เป็น struct ที่เราประกาศเองในไฟล์นี้ (เข้าเงื่อนไข "type ที่เป็น
ของ crate ปัจจุบัน" ครบถ้วน) แม้ field เดียวของมัน (`self.0`) จะเป็น `HashMap<String, u32>` ของ std ก็ตาม —
orphan rule มองที่ **type ที่อยู่ตรงกลาง `impl TraitX for TypeY`** เท่านั้น ไม่ได้มองลึกเข้าไปในโครงสร้างข้อมูล
ภายใน จึงผ่านได้ทันที นี่คือเหตุผลที่ newtype pattern เป็นทางแก้ orphan rule ที่นิยมที่สุดในโค้ด Rust จริง — ต้นทุน
ที่แลกมาคือต้องเขียน `self.0` (หรือ destructure) เพื่อเข้าถึงข้อมูลข้างในเสมอ และถ้าต้องการ method อื่นของ
`HashMap` (เช่น `.insert()`, `.get()`) ต้องเขียน wrapper method เพิ่มเอง หรือ implement `Deref`/`DerefMut` ให้
`InventoryReport` (ซึ่งเป็นอีกเทคนิคหนึ่งที่ทำให้ `.insert()` ถูกเรียกผ่าน `report` ตรง ๆ ได้ ผ่าน auto-deref)

**newtype pattern ที่เห็นในหัวข้อนี้เป็นแค่การใช้งานเบื้องต้น** (แก้ orphan rule เฉพาะหน้า) — ในความเป็นจริง
newtype ยังใช้แก้ปัญหาอื่นได้อีกมาก เช่นสร้าง type ที่ปลอดภัยกว่าเดิม (`struct Meters(f64)` แยกจาก `struct
Seconds(f64)` แม้ทั้งคู่เก็บ `f64` เพื่อป้องกันการบวก/สับสนหน่วยผิดตอน compile time), ซ่อน implementation
detail ของ type ภายนอกจาก public API ของเรา, หรือแม้แต่ implement **typestate pattern** (ใช้ type system บังคับ
ลำดับการเรียกใช้ method ให้ถูกต้อง) — รูปแบบเหล่านี้จะเป็นเนื้อหาหลักแบบเต็มรูปแบบใน **Part 53: Design Patterns
ใน Rust (Newtype, Typestate, RAII)**

### 21.9 Trait Object ที่ใช้บ่อยที่สุดใน Standard Library: `Box<dyn Error>` และ `Box<dyn Fn(...)>`

ตอนนี้เรามีความรู้พอที่จะเข้าใจ trait object สองตัวที่คุณ**ใช้มาแล้วแบบ pragmatic** ในบทก่อน ๆ ได้อย่างครบถ้วน

#### `Box<dyn Error>`: ที่คุณใช้มาตั้งแต่ Part 12

ทวนจาก **Part 12 หัวข้อ 12.7-12.8**: เราเขียน `fn parse_port(raw: &str) -> Result<u16, Box<dyn Error>>` และ
`fn main() -> Result<(), Box<dyn Error>>` มาแล้ว พร้อมคำอธิบายแบบ pragmatic ว่า "`dyn Error` คือ trait object
ที่หมายถึงอะไรก็ได้ที่ implement `std::error::Error`" — **ตอนนี้เรารู้กลไกเบื้องหลังครบแล้ว**: `Box<dyn Error>`
ไม่มีอะไรพิเศษไปกว่า `Box<dyn Shape>` ที่เรียนมาทั้งบทนี้เลยแม้แต่นิดเดียว มันคือ fat pointer (data pointer +
vtable pointer ของ `impl Error for ...`) ที่ทำให้ **error หลายชนิดที่ต่างกัน** (เช่น `ParseIntError` จาก parse
ตัวเลข, `std::io::Error` จากการอ่านไฟล์, error ที่คุณสร้างเอง) **ถูก return จากฟังก์ชันเดียวกันได้** โดยไม่ต้อง
สร้าง enum รวม error ทุกชนิดเอง — เหตุผลเดียวกันเป๊ะกับที่ `shape_from_config` ในหัวข้อ 21.5 ต้องใช้ `Box<dyn
Shape>` เพราะ error แต่ละชนิดที่อาจเกิดขึ้นเป็นคนละ concrete type กัน (คนละขนาด คนละ field) และฟังก์ชันหนึ่งตัว
ต้อง return ได้แค่ type เดียวที่แน่นอนเสมอ (ตามกฎจาก 21.5) — `Box<dyn Error>` คือคำตอบที่ "ลบ" ความแตกต่างของ
concrete type เหล่านั้นออกไป เหลือแค่ "อะไรก็ได้ที่ implement Error" (เรียก `.to_string()` หรือ `{}` แสดงข้อความ
error ได้แน่นอน เพราะ `Error: Display` เป็น supertrait อยู่แล้ว — ใช่แล้ว, `trait Error: Debug + Display` ใน std
ก็ใช้ supertrait ที่เพิ่งเรียนในหัวข้อ 21.7 นี่เอง)

ข้อแลกเปลี่ยนที่ Part 12 อธิบายไว้แล้ว (เสียข้อมูลชนิดที่แน่นอนของ error ไป ผู้เรียกแยกกรณีตามชนิดไม่ได้ตรง ๆ)
ก็คือข้อแลกเปลี่ยนเดียวกันกับ dynamic dispatch ที่อธิบายในหัวข้อ 21.3 ทุกประการ — ได้ความยืดหยุ่นในการรวม
หลาย concrete type ไว้ในที่เดียว แลกกับการไม่รู้ concrete type ที่แน่นอนอีกต่อไป

#### `Box<dyn Fn(...)>`: Trait Object ของ Closure

closure (ฟังก์ชันนิรนามแบบ `|x| expr` ที่เห็นมาแบบผิวเผินตั้งแต่ Part 11 — จะเรียนเต็มรูปแบบเรื่อง `Fn`/`FnMut`/
`FnOnce` และการ capture ตัวแปรใน **Part 24**) ก็เป็น concrete type หนึ่งชนิดในสายตา compiler เช่นกัน — และ
ที่น่าแปลกใจคือ **closure สองตัวที่หน้าตาเหมือนกันแทบทุกอย่าง (รับ parameter ชนิดเดียวกัน คืนค่าชนิดเดียวกัน)
ก็ยังเป็นคนละ concrete type กันอยู่ดี** ถ้า body หรือสิ่งที่ capture ไว้ต่างกัน (compiler generate struct
ที่ไม่ซ้ำกันให้ทุก closure หนึ่งตัวเสมอ) ปัญหานี้คือปัญหาเดียวกันกับ `Circle`/`Rectangle` ทุกประการ — และคำตอบก็
คือ trait object เหมือนกัน เพียงแค่ trait ที่ใช้เป็น `Fn(Args) -> ReturnType` (trait พิเศษที่ std กำหนดไว้ให้
closure ทุกตัว implement โดยอัตโนมัติ) แทน trait ที่เราเขียนเอง:

```rust
// 21.9 - Box<dyn Fn(...)>: trait object ของ closure — เก็บ closure หลายตัวที่ "หน้าตา" ต่างกัน
// (capture ตัวแปรคนละชุด) ไว้ใน collection เดียวกันได้ ด้วยกลไกเดียวกันกับ Box<dyn Shape> ทุกประการ
// (รายละเอียดเรื่อง closure, Fn/FnMut/FnOnce แบบเต็มรูปแบบจะเรียนใน Part 24)
fn make_handlers(threshold: u32) -> Vec<Box<dyn Fn(&str)>> {
    vec![
        // closure ตัวแรก: ไม่ capture อะไรจากภายนอกเลย
        Box::new(|msg: &str| println!("[LOG] {msg}")),
        // closure ตัวที่สอง: capture ตัวแปร threshold จากภายนอกฟังก์ชัน
        Box::new(move |msg: &str| println!("[ALERT ระดับ {threshold}] {}", msg.to_uppercase())),
    ]
}

fn main() {
    // แต่ละ closure ใน Vec นี้เป็นคนละ "concrete type" กันจริง ๆ ในสายตา compiler (compiler generate
    // struct ที่ไม่ซ้ำกันให้ทุก closure หนึ่งตัว) — สิ่งที่ทำให้เก็บไว้ใน Vec เดียวกันได้คือ dyn Fn(&str)
    // ที่ลบข้อมูลเรื่อง concrete closure type ทิ้งไปเหลือแค่ "เรียกได้ด้วย &str แล้วไม่คืนค่า" เท่านั้น
    let handlers = make_handlers(3);

    for handler in &handlers {
        handler("ระบบเริ่มทำงาน");
    }
}
```

ผลลัพธ์:

```
[LOG] ระบบเริ่มทำงาน
[ALERT ระดับ 3] ระบบเริ่มทำงาน
```

**สังเกตว่านี่คือปัญหาและคำตอบแบบเดียวกันเป๊ะกับทั้งบทนี้**: closure หลายตัวที่ไม่มีความสัมพันธ์กันเลยในแง่
concrete type (เหมือน `Circle`/`Rectangle`/`Triangle`) ต้องการเก็บไว้ใน `Vec` เดียวกัน แก้ด้วย trait object
(`Box<dyn Fn(&str)>` แทน `Box<dyn Shape>`) — เพียงแค่ trait ที่ใช้คราวนี้ std เตรียมไว้ให้แล้ว (`Fn`) ไม่ต้อง
ประกาศเอง เราจะกลับมาเจาะลึกเรื่อง closure, `Fn`/`FnMut`/`FnOnce` ต่างกันอย่างไร และ capture ตัวแปรแบบไหนบ้าง
อย่างเต็มรูปแบบใน Part 24 — ตอนนี้จำแค่หลักการที่ว่า **`Box<dyn Fn(...)>` คือการประยุกต์ trait object แบบเดียวกัน
กับที่เรียนมาทั้งบท เข้ากับ closure** ก็เพียงพอแล้ว

### 21.10 ตัวอย่างจริงจัง: ระบบ "Plugin" รูปทรงเรขาคณิต — Static Dispatch เทียบ Dynamic Dispatch แบบเห็นภาพ

มาถึงตัวอย่างสรุปที่รวมทุกแนวคิดหลักของบทนี้เข้าด้วยกัน — สร้างระบบที่จำลองสถานการณ์ "plugin": โมดูลต่าง ๆ ใน
ระบบ (เช่นตัวอ่าน config, ตัวโหลด extension) สามารถ "เพิ่มรูปทรงชนิดใหม่" เข้ามาที่ runtime ได้เรื่อย ๆ โดยไม่
ต้องรู้จักกันมาก่อนตอน compile time — แล้วเทียบตรง ๆ กับแนวทาง static dispatch ที่ทำสิ่งเดียวกันไม่ได้:

```rust
// 21.10 - ตัวอย่างจริงจัง: ระบบ "plugin" รูปทรงเรขาคณิต — เทียบ static dispatch (generic) กับ
// dynamic dispatch (Box<dyn Shape>) แบบเห็นภาพชัดเจนที่สุดในบทนี้
use std::f64::consts::PI;

trait Shape {
    fn area(&self) -> f64;
    fn name(&self) -> &str;

    fn describe(&self) -> String {
        format!("{}: พื้นที่ {:.2} ตารางหน่วย", self.name(), self.area())
    }
}

struct Circle {
    radius: f64,
}
impl Shape for Circle {
    fn area(&self) -> f64 {
        PI * self.radius * self.radius
    }
    fn name(&self) -> &str {
        "วงกลม"
    }
}

struct Rectangle {
    width: f64,
    height: f64,
}
impl Shape for Rectangle {
    fn area(&self) -> f64 {
        self.width * self.height
    }
    fn name(&self) -> &str {
        "สี่เหลี่ยมผืนผ้า"
    }
}

struct Triangle {
    base: f64,
    height: f64,
}
impl Shape for Triangle {
    fn area(&self) -> f64 {
        0.5 * self.base * self.height
    }
    fn name(&self) -> &str {
        "สามเหลี่ยม"
    }
}

// === เวอร์ชัน static dispatch (generic) ===
// total_area_static ใช้ monomorphization (ทวนจาก Part 18): compiler จะ generate โค้ดแยกกัน
// หนึ่งเวอร์ชันสำหรับทุก T ที่ถูกเรียกใช้จริง — เร็วที่สุด ไม่มี runtime overhead เลย
// แต่ "T" ต้องเป็น concrete type เดียวเท่านั้นตลอดทั้ง slice — รับ &[Circle] ได้ หรือ &[Rectangle] ได้
// แต่รับ slice ที่ผสมกันไม่ได้เด็ดขาด
fn total_area_static<T: Shape>(shapes: &[T]) -> f64 {
    shapes.iter().map(|s| s.area()).sum()
}

// === เวอร์ชัน dynamic dispatch (trait object) ===
// total_area_dynamic รับ slice ของ Box<dyn Shape> — ยอมรับรูปทรงต่าง concrete type กันปนกันได้ในครั้งเดียว
// แลกกับ indirect call ผ่าน vtable ทุกครั้งที่เรียก .area() (อธิบายละเอียดในหัวข้อ 21.3)
fn total_area_dynamic(shapes: &[Box<dyn Shape>]) -> f64 {
    shapes.iter().map(|s| s.area()).sum()
}

// ระบบ "plugin registry" — จุดที่ dynamic dispatch เหนือกว่าชัดเจนที่สุด: โมดูลอื่นในระบบ "เพิ่มรูปทรงชนิดใหม่"
// เข้ามาที่ runtime ได้เรื่อย ๆ โดยไม่ต้องรู้จักกันมาก่อนตอน compile time เลย (คล้าย plugin ที่โหลดเข้าระบบทีหลัง)
struct ShapeRegistry {
    shapes: Vec<Box<dyn Shape>>,
}

impl ShapeRegistry {
    fn new() -> Self {
        Self { shapes: Vec::new() }
    }

    // register รับ "รูปทรงอะไรก็ได้ที่ implement Shape" ผ่าน generic + Box::new ตอนเรียก
    // (เขียนแบบนี้เพื่อให้ผู้เรียกไม่ต้องเขียน Box::new เองทุกครั้ง)
    fn register<S: Shape + 'static>(&mut self, shape: S) {
        self.shapes.push(Box::new(shape));
    }

    fn total_area(&self) -> f64 {
        total_area_dynamic(&self.shapes)
    }

    fn report_all(&self) {
        for shape in &self.shapes {
            println!("  - {}", shape.describe());
        }
    }
}

fn main() {
    println!("=== static dispatch: จัดการรูปทรงทีละชนิดแยกกัน ===");
    let circles = [Circle { radius: 2.0 }, Circle { radius: 3.0 }];
    let rectangles = [Rectangle { width: 4.0, height: 5.0 }];

    // ต้องเรียก total_area_static แยกกันคนละครั้งสำหรับแต่ละ concrete type
    // ไม่มีทางเรียกครั้งเดียวรวมทั้ง circles และ rectangles พร้อมกันได้เลย เพราะเป็นคนละ T กัน
    let circle_area = total_area_static(&circles);
    let rectangle_area = total_area_static(&rectangles);
    println!("พื้นที่วงกลมรวม: {circle_area:.2}");
    println!("พื้นที่สี่เหลี่ยมรวม: {rectangle_area:.2}");
    println!("ต้องบวกเอง (ไม่มี type เดียวที่เก็บทั้งสองกลุ่มได้): {:.2}", circle_area + rectangle_area);

    println!("\n=== dynamic dispatch: ระบบ plugin registry เดียวรับได้ทุกรูปทรง ===");
    let mut registry = ShapeRegistry::new();
    registry.register(Circle { radius: 2.0 });
    registry.register(Rectangle { width: 4.0, height: 5.0 });
    registry.register(Triangle { base: 6.0, height: 8.0 });
    // สมมติว่ามี "plugin" ใหม่ถูกเพิ่มเข้ามาตอน runtime (เช่นอ่านจาก config) — Circle อีกตัวที่ค่าต่างกัน
    registry.register(Circle { radius: 1.0 });

    registry.report_all();
    println!("พื้นที่รวมทั้งหมดในระบบเดียว: {:.2}", registry.total_area());
}
```

ผลลัพธ์:

```
=== static dispatch: จัดการรูปทรงทีละชนิดแยกกัน ===
พื้นที่วงกลมรวม: 40.84
พื้นที่สี่เหลี่ยมรวม: 20.00
ต้องบวกเอง (ไม่มี type เดียวที่เก็บทั้งสองกลุ่มได้): 60.84

=== dynamic dispatch: ระบบ plugin registry เดียวรับได้ทุกรูปทรง ===
  - วงกลม: พื้นที่ 12.57 ตารางหน่วย
  - สี่เหลี่ยมผืนผ้า: พื้นที่ 20.00 ตารางหน่วย
  - สามเหลี่ยม: พื้นที่ 24.00 ตารางหน่วย
  - วงกลม: พื้นที่ 3.14 ตารางหน่วย
พื้นที่รวมทั้งหมดในระบบเดียว: 59.71
```

**สังเกตความแตกต่างที่เป็นรูปธรรมที่สุดของทั้งบทนี้**:

1. **`total_area_static`** ต้องถูกเรียก**แยกกันคนละครั้ง**สำหรับ `circles` (`Vec<Circle>` — เป็น array แต่หลักการ
   เดียวกัน) และ `rectangles` เพราะทั้งสองเป็นคนละ `T` กัน — ไม่มีทางเขียน `total_area_static` ให้รับทั้งสองกลุ่ม
   พร้อมกันในการเรียกครั้งเดียวได้เลย ต้อง**บวกผลลัพธ์เอาเองด้วยมือ**ทีหลัง (`circle_area + rectangle_area`)
   ซึ่งยิ่งมีรูปทรงมากชนิดขึ้นเรื่อย ๆ ก็ต้องเขียนโค้ดบวกเพิ่มไปเรื่อย ๆ ไม่มีที่สิ้นสุด
2. **`ShapeRegistry`** ใช้ `Box<dyn Shape>` ข้างใน ทำให้ `register()` รับรูปทรง**ชนิดไหนก็ได้ที่ implement Shape**
   สลับกันไปมาได้อย่างอิสระในลำดับใดก็ได้ (`Circle`, `Rectangle`, `Triangle`, `Circle` อีกตัว) เก็บไว้ใน field
   เดียว (`shapes: Vec<Box<dyn Shape>>`) แล้ว `total_area()` คำนวณผลรวมของ**ทุกรูปทรงในระบบพร้อมกันในครั้งเดียว**
   โดยไม่ต้องรู้ล่วงหน้าเลยว่าจะมีรูปทรงกี่ชนิด — นี่คือสิ่งที่ static dispatch ทำไม่ได้เลยไม่ว่าจะเขียนอย่างไร
   ก็ตาม เพราะเป็นข้อจำกัดพื้นฐานของ generics (Vec ต้องมี `T` เดียว) ไม่ใช่ข้อจำกัดที่แก้ด้วยเทคนิคการเขียนโค้ด
3. **ต้นทุนที่แลกมา**: การเรียก `shape.area()` ข้างใน `total_area_dynamic`/`report_all` ทุกครั้งคือ indirect
   call ผ่าน vtable (ตามที่อธิบายในหัวข้อ 21.3) — ช้ากว่า `total_area_static` เล็กน้อยในทางทฤษฎี แต่ในทางปฏิบัติ
   ระบบ plugin แบบนี้มักไม่ได้ถูกเรียกในจำนวนที่มากถึงขั้นความต่างนี้มีผลกระทบจริง (การคำนวณพื้นที่รูปทรงไม่กี่สิบ
   หรือไม่กี่ร้อยตัวต่อครั้ง ไม่ใช่ hot loop ระดับล้านครั้งต่อวินาที) — **ความยืดหยุ่นที่ได้มาคุ้มค่ากับต้นทุนที่
   จ่ายไปมาก** สำหรับสถานการณ์แบบนี้

**บทสรุปของการเลือก**: ถ้ารู้ล่วงหน้าตอน compile time ว่าจะทำงานกับ type ไหน (และมักจะเป็น type เดียวตลอด หรือ
จำนวน type จำกัดไม่กี่ตัว) ใช้ static dispatch (`impl Trait`/`<T: Trait>`) — ถ้าต้องรองรับ type ที่**ไม่รู้จำนวน
หรือชนิดล่วงหน้า** (โดยเฉพาะระบบที่ออกแบบให้ขยายได้ในอนาคตแบบ plugin, event handler, GUI widget ที่ต่าง component
กันมีพฤติกรรมต่างกัน) ใช้ dynamic dispatch (`dyn Trait`) — ทั้งสองไม่ใช่คู่แข่งที่ต้องเลือกอย่างใดอย่างหนึ่งเสมอ
ในโปรเจกต์เดียว โค้ด Rust ระดับ production จริงส่วนใหญ่ใช้**ทั้งสองแบบผสมกัน** ตามความเหมาะสมของแต่ละจุดในระบบ

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. Trait มี method ที่ return `Self` แล้วพยายามใช้เป็น `dyn Trait` — E0038

```rust
trait Cloneable {
    fn clone_it(&self) -> Self;
}

struct Widget {
    id: u32,
}

impl Cloneable for Widget {
    fn clone_it(&self) -> Self {
        Widget { id: self.id }
    }
}

fn describe(item: &dyn Cloneable) {
    let _ = item;
}

fn main() {
    let w = Widget { id: 1 };
    describe(&w);
}
```

```
error[E0038]: the trait `Cloneable` is not dyn compatible
   |
18 | fn describe(item: &dyn Cloneable) {
   |                    ^^^^^^^^^^^^^ `Cloneable` is not dyn compatible
   |
 4 |     fn clone_it(&self) -> Self;
   |                           ^^^^ ...because method `clone_it` references the `Self` type in its return type
```

**วิธีแก้**: ถ้า method ที่ return `Self` ไม่จำเป็นต้องเรียกผ่าน `dyn Trait` เลย ให้เพิ่ม `where Self: Sized`
เข้าไป (อธิบายละเอียดในหัวข้อ 21.4) — ถ้าจำเป็นต้องเรียกผ่าน `dyn Trait` จริง ๆ ให้เปลี่ยน signature เป็น return
`Box<Self>` หรือรูปแบบอื่นที่ไม่อ้างอิง `Self` แบบ by-value ตรง ๆ (เช่น `fn clone_boxed(&self) -> Box<dyn
Cloneable>` — สังเกตว่านี่ **dyn-compatible** เพราะ return type เป็น `Box<dyn Cloneable>` ที่มีขนาดคงที่เสมอ
ไม่ใช่ `Self` ที่ขนาดไม่แน่นอน)

### 2. Orphan Rule: implement foreign trait ให้ foreign type ไม่ได้ — E0117

```rust
use std::collections::HashMap;
use std::fmt;

impl fmt::Display for HashMap<String, u32> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        for (name, qty) in self {
            writeln!(f, "{name}: {qty} หน่วย")?;
        }
        Ok(())
    }
}

fn main() {
    let stock: HashMap<String, u32> = HashMap::new();
    println!("{stock}");
}
```

```
error[E0117]: only traits defined in the current crate can be implemented for types defined outside of the crate
  |
  |                       `HashMap` is not defined in the current crate
  |
  = note: define and implement a trait or new type instead
```

**วิธีแก้**: ใช้ newtype pattern ห่อ type ภายนอกด้วย struct ของเราเอง (`struct InventoryReport(HashMap<String,
u32>)`) แล้ว implement trait ให้ struct ห่อนั้นแทน ตามที่อธิบายละเอียดในหัวข้อ 21.8 — จำกฎง่าย ๆ ไว้: **orphan
rule อนุญาตก็ต่อเมื่อ trait หรือ type อย่างน้อยหนึ่งอย่างเป็นของ crate ปัจจุบัน**

### 3. ลืม `Box::new` ตอน push ค่าเข้า `Vec<Box<dyn Trait>>`

```rust
trait Shape {
    fn area(&self) -> f64;
}

struct Circle {
    radius: f64,
}

impl Shape for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.radius * self.radius
    }
}

struct Rectangle {
    width: f64,
    height: f64,
}

impl Shape for Rectangle {
    fn area(&self) -> f64 {
        self.width * self.height
    }
}

fn main() {
    let mut shapes: Vec<Box<dyn Shape>> = Vec::new();
    shapes.push(Box::new(Circle { radius: 2.0 }));
    // ลืม Box::new ครอบ Rectangle — พยายาม push ค่า Rectangle ตรง ๆ เข้า Vec<Box<dyn Shape>>
    shapes.push(Rectangle { width: 3.0, height: 4.0 });

    let total: f64 = shapes.iter().map(|s| s.area()).sum();
    println!("{total}");
}
```

```
error[E0308]: mismatched types
   |
30 |     shapes.push(Rectangle { width: 3.0, height: 4.0 });
   |            ---- ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected `Box<dyn Shape>`, found `Rectangle`
   |
   = note: expected struct `Box<dyn Shape>`
              found struct `Rectangle`
help: store this in the heap by calling `Box::new`
   |
30 |     shapes.push(Box::new(Rectangle { width: 3.0, height: 4.0 }));
   |                 +++++++++                                     +
```

**วิธีแก้**: ครอบทุกค่าที่ push เข้า `Vec<Box<dyn Trait>>` ด้วย `Box::new(...)` เสมอ — สังเกตว่า error message
ประเภทนี้ (`expected Box<dyn Shape>, found Rectangle`) หน้าตาคล้ายกับ error ตอนพยายามยัดสอง concrete type ต่าง
กันเข้า `Vec` เดียวโดยไม่มี `dyn` เลย (หัวข้อ 21.1) เพราะโดยรากเหง้าเป็นกฎเดียวกัน: **`Vec<T>` ต้องการค่าที่ตรงกับ
`T` เป๊ะเสมอ** เพียงแต่คราวนี้ `T` คือ `Box<dyn Shape>` ไม่ใช่ `Circle`/`Rectangle` ตรง ๆ — `Rectangle` เพียว ๆ
(ไม่ใส่ `Box`) จึงไม่ตรงกับ `T` ที่ต้องการ ต้องแปลงเป็น `Box<dyn Shape>` ก่อนเสมอ

### 4. ลืม `dyn` keyword — E0782 (edition 2021 บังคับให้เขียน `dyn` เสมอ)

```rust
trait Shape {
    fn area(&self) -> f64;
}

struct Circle {
    radius: f64,
}
impl Shape for Circle {
    fn area(&self) -> f64 {
        self.radius * self.radius * 3.14
    }
}

// ลืมเขียน dyn หน้า Shape — ใน edition รุ่นเก่า (2015) เขียนแบบนี้ได้ (แต่จะได้ warning) ส่วน edition 2018
// เป็นต้นมาบังคับให้ต้องเขียน dyn ชัดเจนเสมอ
fn print_area(shape: &Shape) {
    println!("{}", shape.area());
}

fn main() {
    let c = Circle { radius: 2.0 };
    print_area(&c);
}
```

```
error[E0782]: expected a type, found a trait
  --> src/main.rs:14:23
   |
14 | fn print_area(shape: &Shape) {
   |                       ^^^^^
   |
help: you can also use an opaque type, but users won't be able to specify the type parameter when calling the `fn`, having to rely exclusively on type inference
   |
14 | fn print_area(shape: &impl Shape) {
   |                       ++++
help: alternatively, use a trait object to accept any type that implements `Shape`, accessing its methods at runtime using dynamic dispatch
   |
14 | fn print_area(shape: &dyn Shape) {
   |                       +++
```

**วิธีแก้**: เขียน `&dyn Shape` เสมอเมื่อต้องการ trait object — เหตุผลที่ Rust บังคับให้เขียน `dyn` ชัดเจน (ตั้งแต่
edition 2018 เป็นต้นมา) คือเพื่อ**ความชัดเจนในการอ่านโค้ด**: แค่มองเห็น `dyn` ก็รู้ทันทีว่าตำแหน่งนี้ใช้ dynamic
dispatch (มี vtable, fat pointer, indirect call ตามที่เรียนในหัวข้อ 21.3) ต่างจาก `&impl Shape`/`<T: Shape>` ที่
ใช้ static dispatch — สังเกตว่า compiler แนะนำทั้งสองทางเลือกมาให้เอง (`&impl Shape` หรือ `&dyn Shape`) เพราะ
มันไม่รู้ว่าคุณตั้งใจจะใช้ทางไหน คุณต้องเลือกเองตามหลักการในหัวข้อ 21.3 และ 19.6

### 5. ส่ง `&Box<dyn Trait>` เข้าฟังก์ชันที่รับ `&dyn Trait` ตรง ๆ โดยไม่ deref

```rust
trait Greeter {
    fn greet(&self) -> String {
        "hi".to_string()
    }
}

struct A;
impl Greeter for A {}

fn print_greeting(g: &dyn Greeter) {
    println!("{}", g.greet());
}

fn main() {
    let b: Box<dyn Greeter> = Box::new(A);
    // คาดว่า &Box<dyn Greeter> จะถูก deref coercion ให้เป็น &dyn Greeter อัตโนมัติเหมือน &String -> &str
    print_greeting(&b);
}
```

```
error[E0277]: the trait bound `Box<dyn Greeter>: Greeter` is not satisfied
   |
   |     print_greeting(&b);
   |                     ^^ the trait `Greeter` is not implemented for `Box<dyn Greeter>`
   |
   = note: required for the cast from `&Box<dyn Greeter>` to `&dyn Greeter`
```

**ทำไมถึง error**: มือใหม่มักคาดหวังว่า `&Box<dyn Greeter>` จะถูก **deref coercion** (ทวนจาก Part 7-8) แปลงเป็น
`&dyn Greeter` ให้อัตโนมัติในตำแหน่ง function argument เหมือนที่ `&String` แปลงเป็น `&str` ได้เอง — แต่ในทางปฏิบัติ
การ coercion แบบนี้ **ไม่เกิดขึ้นอัตโนมัติในตำแหน่ง argument ของฟังก์ชัน** เมื่อปลายทางเป็น `dyn Trait` (ต่างจาก
การเรียก **method** ผ่าน `.` ที่ autoderef เต็มรูปแบบเกิดขึ้นเสมอ อย่างที่เห็นใน `for shape in &shapes {
shape.describe() }` ในหัวข้อ 21.2 ที่ compile ผ่านได้ปกติ) **วิธีแก้**: เรียก `.as_ref()` (เปลี่ยน `&Box<dyn
Greeter>` เป็น `&dyn Greeter` อย่างชัดเจน) หรือ deref สองชั้นด้วยมือ (`&**b`) เสมอเมื่อต้องส่ง `Box<dyn Trait>`
เข้าฟังก์ชันที่รับ `&dyn Trait` ตรง ๆ — จำไว้ว่า **"เรียก method ผ่าน `.`" กับ "ส่งเป็น argument ของฟังก์ชัน"
เป็นกฎ coercion คนละชุดกัน** แม้จะดูคล้ายกันก็ตาม

### 6. เก็บ `dyn Trait` แบบ "by value" ตรง ๆ โดยไม่มี indirection

```rust
trait Shape {
    fn area(&self) -> f64;
}

struct Circle {
    radius: f64,
}
impl Shape for Circle {
    fn area(&self) -> f64 {
        self.radius * self.radius * 3.14
    }
}

// ลืมใส่ & หรือ Box หน้า dyn Shape — พยายามรับ trait object "แบบ by value ตรง ๆ" โดยไม่มี indirection ใด ๆ
fn print_shape(s: dyn Shape) {
    println!("{:.2}", s.area());
}

fn main() {
    let c = Circle { radius: 2.0 };
    print_shape(c);
}
```

```
error[E0277]: the size for values of type `(dyn Shape + 'static)` cannot be known at compilation time
   |
11 | fn print_shape(s: dyn Shape) {
   |                   ^^^^^^^^^ doesn't have a size known at compile-time
   |
   = help: the trait `Sized` is not implemented for `(dyn Shape + 'static)`
help: function arguments must have a statically known size, borrowed types always have a known size
   |
11 | fn print_shape(s: &dyn Shape) {
   |                   +
```

**ทำไมถึง error**: `dyn Shape` (ไม่มี `&` หรือ `Box` นำหน้า) คือ **unsized type** (ทวนจากหัวข้อ 21.3 — เพราะ
compiler ไม่รู้ขนาดของ "ค่าที่อยู่หลัง trait object" ตั้งแต่ compile time ได้เลย) และ Rust **ไม่อนุญาตให้ตัวแปร
หรือ parameter ของฟังก์ชันเป็น unsized type ตรง ๆ โดยไม่มี indirection** (หลักการเดียวกับที่ Part 8 อธิบายว่า
ทำไมประกาศ `let x: str = ...;` ตรง ๆ ไม่ได้ ต้องผ่าน `&str` เท่านั้น) — **วิธีแก้**: ต้องเข้าถึง `dyn Trait` ผ่าน
pointer ชนิดใดชนิดหนึ่งเสมอ ที่พบบ่อยที่สุดคือ `&dyn Shape` (ยืม ไม่ยึดความเป็นเจ้าของ) หรือ `Box<dyn Shape>`
(ยึดความเป็นเจ้าของบน heap) — จะเลือกแบบไหนขึ้นกับว่าฟังก์ชันต้องการความเป็นเจ้าของค่านั้นหรือแค่ยืมใช้ชั่วคราว
(หลักการเดียวกับการเลือก `&T` เทียบกับ `T`/`Box<T>` ที่เรียนมาตั้งแต่ Part 6-7)

## แบบฝึกหัด (Exercises)

1. **โจทย์ระดับง่าย**: เพิ่ม `struct Square { side: f64 }` เข้าไปในระบบจากหัวข้อ 21.10 (`impl Shape for Square`
   โดย `area()` คือ `side * side` และ `name()` คือ `"สี่เหลี่ยมจัตุรัส"`) แล้วเรียก
   `registry.register(Square { side: 5.0 })` เพิ่มเข้าไปใน `main()` ทดสอบว่า `registry.total_area()` เปลี่ยนไป
   ถูกต้องตามที่คำนวณได้ (Hint: ไม่ต้องแก้ signature ของ `register()`, `total_area()`, หรือ `ShapeRegistry` เลย
   แม้แต่บรรทัดเดียว — นี่คือจุดสำคัญที่สุดที่ต้องสังเกต: ระบบที่ออกแบบด้วย `dyn Trait` รองรับ type ใหม่ได้โดย
   **ไม่ต้องแก้โค้ดเดิมแม้แต่นิดเดียว** ตรงข้ามกับระบบที่ใช้ `enum` ปิด (closed set) ที่ทวนจาก Part 10 ซึ่งต้องแก้
   `match` ทุกที่ที่ใช้เมื่อเพิ่ม variant ใหม่)

2. **โจทย์ระดับกลาง**: เขียน trait ของคุณเองชื่อ `trait Resizable { fn scale(&self, factor: f64) -> Self; }`
   แล้วลองเขียนฟังก์ชัน `fn use_resizable(item: &dyn Resizable)` ดูว่าเกิด error อะไร อ่าน error message ให้เข้าใจ
   ว่าทำไม (Hint: ต้องเป็น E0038 เพราะ `scale` return `Self` แบบ by-value — ตามด้วยการแก้ปัญหาสองแบบ: (ก) เพิ่ม
   `where Self: Sized` ให้ `scale` แล้วดูว่า `use_resizable` ใช้ได้ปกติไหมถ้าไม่เรียก `.scale()` ข้างในเลย และ
   (ข) เปลี่ยน signature เป็น `fn scale_boxed(&self, factor: f64) -> Box<dyn Resizable>` แทน แล้วดูว่า trait
   กลายเป็น dyn-compatible ทันทีหรือไม่)

3. **โจทย์ระดับยาก**: ออกแบบ supertrait `trait Priced: Shape { fn price_per_unit_area(&self) -> f64; fn
   total_price(&self) -> f64 { self.area() * self.price_per_unit_area() } }` แล้ว implement `Priced` ให้กับ
   `Circle` และ `Rectangle` จากหัวข้อ 21.10 (ราคาต่อตารางหน่วยกำหนดเองได้ตามใจ) จากนั้นสร้าง
   `Vec<Box<dyn Priced>>` ที่เก็บทั้งสองชนิด แล้วเขียนฟังก์ชัน `fn total_inventory_value(items: &[Box<dyn
   Priced>]) -> f64` ที่รวม `total_price()` ของทุกตัว (Hint: ต้อง `impl Shape for X` ให้ครบก่อน `impl Priced for
   X` เสมอ เพราะ `Priced: Shape` — ถ้าลืมจะได้ E0277 แบบเดียวกับหัวข้อ 21.7 ที่แสดง error ของ `Employee: Named`)

4. **โจทย์ระดับยาก/ประยุกต์ใช้งานจริง**: สร้างระบบ "plugin command" ง่าย ๆ — นิยาม `trait Command { fn run(&self)
   -> String; fn describe(&self) -> String { format!("รันคำสั่งแล้วได้ผลลัพธ์: {}", self.run()) } }` implement
   `Command` ให้ struct อย่างน้อยสองชนิดที่ไม่เกี่ยวข้องกันเลย (เช่น `struct GreetCommand { name: String }` ที่
   `run()` คืนคำทักทาย, และ `struct SumCommand { numbers: Vec<i32> }` ที่ `run()` คืนผลรวมเป็น string) แล้วเก็บไว้
   ใน `Vec<Box<dyn Command>>` วนเรียก `.describe()` ทุกตัว (ทวนหัวข้อ 21.6 ว่าทำไม `describe()` ที่เป็น default
   ยังทำงานถูกต้องผ่าน `dyn Command` แม้แต่ละ `Command` จะมี `run()` ที่ทำงานต่างกันสิ้นเชิง) ขั้นสุดท้าย ลองเพิ่ม
   method `fn clone_command(&self) -> Box<dyn Command>` เข้าไปใน trait (**ไม่ใช่** `fn clone(&self) -> Self`)
   แล้วสังเกตว่า trait ยังคง dyn-compatible อยู่ (Hint: เพราะ return type เป็น `Box<dyn Command>` ที่มีขนาดคงที่
   เสมอ ไม่ใช่ `Self` ที่ขนาดไม่แน่นอน — ทวนความแตกต่างนี้จากกับดักที่ 1 ท้ายบท)

## สรุป

บทนี้เจาะลึก **trait object** และ **dynamic dispatch** ซึ่งเป็นสิ่งที่ Part 19 เกริ่นไว้เป็นเบื้องต้นเท่านั้น —
เราเริ่มจากยืนยันปัญหาที่ generic แก้ไม่ได้ (heterogeneous collection ที่ต้องเก็บหลาย concrete type ใน `Vec`
เดียว พร้อม compile error จริง) แล้วแก้ด้วย `&dyn Trait`/`Box<dyn Trait>` จากนั้นเจาะกลไกภายในแบบเต็มรูปแบบ:
**monomorphization** (static dispatch, ทวนจาก Part 18) เทียบกับ **vtable** (dynamic dispatch) พร้อมพิสูจน์ด้วย
`std::mem::size_of` ว่า `&dyn Trait` เป็น **fat pointer** (data pointer + vtable pointer) เหมือนหลักการเดียวกับ
`&str` ที่เรียนมาตั้งแต่ Part 8 เราเรียนกฎ **object safety** (หรือชื่อใหม่ **dyn compatibility**) ครบทั้งสามข้อ
พร้อม error E0038 จริงและเทคนิค `where Self: Sized` เจาะลึกการ return `impl Trait` เทียบกับ `Box<dyn Trait>` ด้วย
ตัวอย่างโรงงานสร้างรูปทรงจาก config พิสูจน์ว่า default method ผ่าน trait object ยัง dispatch ไปยัง override ของ
concrete type จริงเสมอ (ไม่ใช่ default แบบตายตัว) เรียน **supertrait** (`trait B: A`) ในฐานะข้อบังคับเพิ่มเติม
ระดับสัญญา (ไม่ใช่ inheritance) ทวน **orphan rule** และ **newtype pattern** ด้วยตัวอย่างใหม่ (พร้อมทีเซอร์ไปยัง
Part 53 สำหรับรูปแบบเต็ม) อธิบาย `Box<dyn Error>` (ที่ใช้มาแบบ pragmatic ตั้งแต่ Part 12) และ `Box<dyn
Fn(...)>` (ทีเซอร์ closure เต็มรูปแบบใน Part 24) ในฐานะ trait object ธรรมดา และปิดท้ายด้วยระบบ plugin ที่เทียบ
static dispatch กับ dynamic dispatch แบบเห็นภาพชัดเจนที่สุด

ตอนนี้คุณมีความเข้าใจ trait system ของ Rust ครบทั้งสองเสาหลัก: **static dispatch** (เร็วที่สุด ตรวจสอบเข้มงวด
ที่สุด แต่ต้องรู้ concrete type ตายตัว) และ **dynamic dispatch** (ยืดหยุ่นที่สุด รองรับ type ใหม่ในอนาคตได้โดยไม่
ต้องแก้โค้ดเดิม แลกกับต้นทุน runtime เล็กน้อย) — ความรู้นี้จะติดตัวคุณไปใช้ตลอดทั้งหลักสูตรที่เหลือ โดยเฉพาะตอน
เรียน **generics ขั้นสูง** ใน Part ถัดไป ที่จะกลับไปสำรวจฝั่ง static dispatch ให้ลึกยิ่งขึ้นไปอีก — trait bound
ที่ซับซ้อนกว่าเดิม, `PhantomData`, และเทคนิค generic ระดับ production ที่ผสมทั้งสองแนวคิดเข้าด้วยกันอย่างแนบเนียน

---

**Part ก่อนหน้า:** [Lifetimes เบื้องต้น](part-020-lifetimes-basics.md) | **Part ถัดไป:** [Generics ขั้นสูง](part-022-generics-advanced.md)
