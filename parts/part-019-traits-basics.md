# Part 19: Traits เบื้องต้น

> โมดูล: พื้นฐานภาษา Rust (Core Language Fundamentals) | ระดับ: พื้นฐาน | เวลาโดยประมาณ: 180 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้ว่าทำไม struct หลายตัวที่ไม่มีความสัมพันธ์กันเลย (ไม่มี inheritance แบบ OOP ดั้งเดิม) ยังสามารถมี
  "พฤติกรรมร่วม" ที่ใช้งานสลับกันได้ผ่าน **trait** — และทำไมปัญหานี้แก้ด้วยฟังก์ชันแยกกันคนละตัวไม่ได้ดีพอ
- นิยาม trait ด้วย `trait ชื่อ { fn ...; }` และเข้าใจว่า method signature ที่ไม่มี body คือ "สัญญา" (contract)
  ไม่ใช่การ implement จริง แล้ว `impl TraitName for TypeName { ... }` ให้กับหลาย type ที่ไม่เกี่ยวข้องกันได้อย่างถูกต้อง
- เขียน **default method implementation** ที่ type เลือกใช้ตามเดิมได้หรือ override เองก็ได้ และเข้าใจ pattern
  คลาสสิกที่ default method หนึ่งเรียกใช้ method อื่นซึ่งยัง "ไม่มี body" (abstract) ในเวลาที่ trait ถูกประกาศ
- เขียนฟังก์ชันที่รับ "อะไรก็ได้ที่ implement trait X" ได้ทั้งสองรูปแบบ — `&impl Trait` และ `<T: Trait>` — อธิบายได้ว่า
  ทั้งสองรูปแบบเทียบเท่ากันอย่างไร และรู้ว่าเมื่อไหร่ควรเลือกแบบไหน
- ใช้ **multiple trait bounds** (`T: A + B`) และ **`where` clause** เพื่อเขียนเงื่อนไข generic ที่อ่านง่ายขึ้น
  เมื่อ bound ซับซ้อนหรือมีหลาย type parameter
- อธิบายข้อจำกัดสำคัญของการ return `impl Trait` (ต้อง return concrete type เดียวเสมอ) พร้อมอ่าน compile error จริง
  และรู้จัก `Box<dyn Trait>` ในฐานะทางออกเบื้องต้น (ก่อนเจาะลึกเต็มรูปแบบใน Part 21)
- เข้าใจ **conditional method implementation** (`impl<T: Bound> Type<T> { ... }`) และ **blanket implementation**
  (`impl<T: Display> ToString for T`) ในฐานะรูปแบบขั้นสูงของการใช้ trait bound กับ `impl` block
- อธิบายได้อย่างถูกต้องว่า `#[derive(...)]` แต่ละตัว (`Debug`, `Clone`, `Copy`, `PartialEq`, `Eq`, `PartialOrd`, `Ord`,
  `Hash`, `Default`) generate trait implementation อะไรให้บ้าง และเชื่อมโยงกลับไปยัง ownership (Part 6) และ
  `HashMap` key requirement (Part 15) ได้อย่างลึกซึ้ง
- Overload ตัวดำเนินการ (เช่น `+`) ให้ type ของตัวเองผ่าน `std::ops::Add` และเข้าใจว่ามันคือ trait mechanism เดียวกัน
  กับที่ `impl Add<&str> for String` ใช้มาตั้งแต่ Part 14
- ออกแบบระบบเล็ก ๆ ที่มีหลาย type ทำงานผ่าน trait ร่วมกัน (ระบบรูปทรงเรขาคณิต) ในสไตล์ idiomatic Rust

## ความรู้ที่ต้องมีมาก่อน

- **Part 6 (Ownership เบื้องต้น)**: แนวคิด `Copy`/`Clone`/move ที่เคยเห็นในฐานะ "ป้ายกำกับพฤติกรรม" — บทนี้จะอธิบาย
  อย่างเป็นทางการว่าป้ายกำกับเหล่านั้นคือ **trait** และ `#[derive(Copy)]`/`#[derive(Clone)]` ทำอะไรกันแน่
- **Part 7 (Borrowing และ References)**: จำเป็นสำหรับเข้าใจ `&self` ใน method ของ trait และ `&impl Trait` เป็น
  parameter type
- **Part 9 (Structs)**: struct, `impl` block, method, associated function, และที่สำคัญที่สุดคือ `#[derive(Debug)]`/
  `#[derive(Clone)]`/`#[derive(PartialEq)]` ที่ Part 9 แนะนำไว้แบบผิวเผินและบอกไว้ตรง ๆ ว่า **"จะเจาะลึกความหมายเต็ม
  รูปแบบใน Part 19"** — นี่คือ Part นั้น
- **Part 10 (Enums และ Pattern Matching)**: ใช้ทวนความเข้าใจเรื่อง type ที่ "ปิด" (closed set, กำหนดสมาชิกทั้งหมด
  ไว้ล่วงหน้าใน enum) เทียบกับ trait ที่เป็นระบบ "เปิด" (type ใหม่เพิ่มเข้ามา implement ทีหลังได้เรื่อย ๆ)
- **Part 14 (String และการจัดการข้อความ)**: บทนั้นแนะนำ `impl Add<&str> for String` และ **orphan rule** ไว้แบบ
  เกริ่น ๆ พร้อมบอกว่า "จะเจาะลึก orphan rule ใน Part 19 ตอนเรียน trait อย่างเป็นทางการ" — บทนี้จะอธิบายให้ครบ
- **Part 15 (HashMap, HashSet)**: บทนั้นอธิบายว่าคีย์ของ `HashMap` ต้อง implement `Hash + Eq` และบอกไว้ว่า
  "รายละเอียดทาง type system จะเข้าใจลึกขึ้นตอนเรียน generics และ trait bounds ใน Part 18-19" — บทนี้จะอธิบายว่า
  `Hash`/`Eq` คือ trait อะไรกันแน่ และทำไมต้องมาคู่กันเสมอ
- **Part 18 (Generics เบื้องต้น)**: **สำคัญที่สุดสำหรับบทนี้** — Part 18 สอนไวยากรณ์ generic (`fn largest<T>(...)`)
  และเริ่มเขียน trait bound แบบ `T: PartialOrd + Copy` เพื่อจำกัดว่า `T` ต้อง "ทำอะไรได้บ้าง" โดยยังไม่อธิบายว่า
  `PartialOrd` หรือ `Copy` คือ **trait** ที่แท้จริงแล้วคืออะไร มีโครงสร้างยังไง หรือเขียนของตัวเองได้อย่างไร —
  Part 18 บอกไว้ตรง ๆ ว่าเรื่องนี้จะมาอธิบายเต็ม ๆ ใน Part 19 นี่แหละ ถ้ายังไม่คุ้นกับไวยากรณ์ generic พื้นฐาน
  (`<T>`, การเรียกใช้ generic function, ความหมายของ "generic over type") แนะนำให้ย้อนไปทวน Part 18 ก่อน เพราะบทนี้
  จะใช้ไวยากรณ์ generic ผสมกับ trait ตลอดทั้งบท

ถ้าให้สรุปภาพรวมสั้น ๆ: **generics (Part 18) คือ "กลไก" สำหรับเขียนโค้ดที่ทำงานกับหลาย type — ส่วน trait (Part 19)
คือ "ภาษา" ที่ใช้บอกว่า type เหล่านั้นต้อง "ทำอะไรได้บ้าง"** ทั้งสองแนวคิดนี้แยกกันไม่ได้ในทางปฏิบัติ — คุณจะเห็นว่า
เกือบทุกตัวอย่างในบทนี้ใช้ generic syntax จาก Part 18 ควบคู่กับ trait bound ที่กำลังจะเรียนใหม่ในบทนี้เสมอ

## เนื้อหา

### 19.1 ปัญหาเริ่มต้น: หลาย Struct ที่ไม่เกี่ยวข้องกันเลย แต่ต้องมี "พฤติกรรมร่วม"

ลองนึกภาพระบบเว็บไซต์สื่อผสม (media/content platform) หนึ่งระบบ ที่ต้องแสดง "feed" รวมของเนื้อหาหลากหลายประเภท
บนหน้าเดียวกัน — สินค้าที่ขาย บทความข่าว และโฆษณา สามอย่างนี้เป็นแนวคิดที่ **ไม่มีความสัมพันธ์กันเลยในทางธุรกิจ**
สินค้าไม่ใช่ประเภทย่อยของบทความ บทความก็ไม่ใช่ประเภทย่อยของโฆษณา แต่ทั้งสามอย่างมีความต้องการร่วมกันอย่างหนึ่ง:
**ต้องสรุปตัวเองเป็นข้อความสั้น ๆ ได้** เพื่อแสดงในหน้า feed

มาดูกันว่าถ้าไม่มีแนวคิดอะไรมาช่วยเชื่อมโยงสาม struct นี้เข้าด้วยกัน เราจะเขียนโค้ดแบบไหน:

```rust
// 19.1 - โค้ดที่ compile ผ่าน: สามฟังก์ชันแยกกัน ไม่มี trait ใช้ร่วม
struct Product {
    name: String,
    price: f64,
}

struct Article {
    headline: String,
    author: String,
}

struct Advertisement {
    sponsor: String,
    slogan: String,
}

impl Product {
    fn summarize(&self) -> String {
        format!("สินค้า: {} ราคา {:.2} บาท", self.name, self.price)
    }
}

impl Article {
    fn summarize(&self) -> String {
        format!("บทความ: \"{}\" โดย {}", self.headline, self.author)
    }
}

impl Advertisement {
    fn summarize(&self) -> String {
        format!("โฆษณาจาก {}: {}", self.sponsor, self.slogan)
    }
}

// ต้องเขียนฟังก์ชันแยกกัน 3 ตัว เพราะ Rust ไม่มี function overloading
// (ฟังก์ชันชื่อเดียวกันรับ parameter type ต่างกันไม่ได้ ต่างจาก C++/Java)
fn print_product_brief(p: &Product) {
    println!("{}", p.summarize());
}

fn print_article_brief(a: &Article) {
    println!("{}", a.summarize());
}

fn print_advertisement_brief(ad: &Advertisement) {
    println!("{}", ad.summarize());
}

fn main() {
    let product = Product {
        name: String::from("เมาส์ไร้สาย"),
        price: 299.0,
    };
    let article = Article {
        headline: String::from("Rust แซงหน้า C++ ในโพลนักพัฒนาปี 2026"),
        author: String::from("กองบรรณาธิการ"),
    };
    let ad = Advertisement {
        sponsor: String::from("RustCorp"),
        slogan: String::from("เขียนโค้ดปลอดภัย ไม่ต้องกลัว null pointer"),
    };

    print_product_brief(&product);
    print_article_brief(&article);
    print_advertisement_brief(&ad);
}
```

ผลลัพธ์:

```
สินค้า: เมาส์ไร้สาย ราคา 299.00 บาท
บทความ: "Rust แซงหน้า C++ ในโพลนักพัฒนาปี 2026" โดย กองบรรณาธิการ
โฆษณาจาก RustCorp: เขียนโค้ดปลอดภัย ไม่ต้องกลัว null pointer
```

โค้ดนี้ **ทำงานถูกต้อง 100%** และถ้าคุณมีแค่สาม type นี้ตลอดไป มันก็อาจจะพอไหว แต่ลองมาดูจุดอ่อนที่ซ่อนอยู่:

1. **`Product::summarize`, `Article::summarize`, `Advertisement::summarize` เป็น method คนละตัวกันโดยสิ้นเชิงในสาย
   ตาของ compiler** — แม้ชื่อจะเหมือนกันเป๊ะทั้งสามตัว แต่ compiler ไม่มองว่ามัน "เป็นสิ่งเดียวกัน" เลยแม้แต่นิดเดียว
   มันคือ inherent method (method ที่ผูกกับ type โดยตรง ไม่ผ่าน trait ใด ๆ) สาม method ที่บังเอิญตั้งชื่อเหมือนกัน
   เท่านั้นเอง
2. **ไม่มีทางเขียนฟังก์ชันเดียวที่รับ "อะไรก็ได้ที่มี summarize()"** — ต้องเขียน `print_product_brief`,
   `print_article_brief`, `print_advertisement_brief` แยกกันสามตัว ทั้งที่ logic ข้างในเหมือนกันทุกตัวเป๊ะ ๆ
   (`println!("{}", x.summarize())`) นี่คือการละเมิดหลัก DRY (Don't Repeat Yourself) ที่เจอมาตั้งแต่ Part 9 อีกครั้ง
   แต่รุนแรงกว่าเดิม เพราะทุกครั้งที่เพิ่ม content type ใหม่ (เช่น `Video`, `Announcement`) ต้องเขียนฟังก์ชัน "ตัวห่อ"
   เพิ่มอีกหนึ่งตัวเสมอ ไม่มีทางย่อให้เหลือฟังก์ชันเดียว
3. **ไม่มีทางเก็บสาม type นี้ไว้ใน collection เดียวกันเพื่อวนแสดงผลรวมได้เลย** — ลองดูว่าเกิดอะไรขึ้นถ้าพยายามทำแบบนั้น
   ตรง ๆ:

```rust
struct Product {
    name: String,
    price: f64,
}

struct Article {
    headline: String,
    author: String,
}

impl Product {
    fn summarize(&self) -> String {
        format!("สินค้า: {} ราคา {:.2} บาท", self.name, self.price)
    }
}

impl Article {
    fn summarize(&self) -> String {
        format!("บทความ: \"{}\" โดย {}", self.headline, self.author)
    }
}

fn main() {
    let product = Product {
        name: String::from("เมาส์ไร้สาย"),
        price: 299.0,
    };
    let article = Article {
        headline: String::from("Rust แซงหน้า C++ ในโพลนักพัฒนาปี 2026"),
        author: String::from("กองบรรณาธิการ"),
    };

    // พยายามรวม product และ article ไว้ใน array เดียวกัน เพื่อวน loop เรียก summarize() ทีเดียว
    let feed = [product, article];

    for item in &feed {
        println!("{}", item.summarize());
    }
}
```

โค้ดนี้ **compile ไม่ผ่าน**:

```
error[E0308]: mismatched types
  --> src/main.rs:29:26
   |
29 |     let feed = [product, article];
   |                          ^^^^^^^ expected `Product`, found `Article`
```

**ทำไมถึง error?** ทวนจาก Part 3-4: array ในภาษา Rust (`[T; N]`) ต้องมีสมาชิกเป็น **type เดียวกันเป๊ะทุกตำแหน่ง**
ไม่มีข้อยกเว้น เพราะ compiler ต้องรู้ขนาด (byte) ของสมาชิกแต่ละตัวล่วงหน้าตั้งแต่ compile time เพื่อจองพื้นที่ใน memory
ให้พอดี — `Product` กับ `Article` มี field ต่างกัน ขนาดต่างกัน จึงเป็นคนละ type อย่างสิ้นเชิงในสายตา compiler
ไม่มีทางอยู่ใน array เดียวกันได้เลยไม่ว่าจะเขียนยังไงก็ตาม (เว้นแต่จะห่อด้วยกลไกพิเศษบางอย่าง ซึ่งเราจะเห็นทางออกจริง
ในหัวข้อ 19.9 และ 19.14 ท้ายบท)

**ปัญหาแท้จริงที่เรากำลังเจอคือ**: `Product`, `Article`, `Advertisement` ไม่มี **ความสัมพันธ์ในระดับ type system**
ที่บอกได้ว่า "ทั้งสามตัวนี้ทำอะไรร่วมกันได้" เลย — ภาษาที่มี class inheritance แบบดั้งเดิม (Java, C++, Python) แก้ปัญหา
นี้ด้วยการให้ทั้งสาม class extend/implement class หรือ interface ร่วมกัน (เช่น `abstract class Summarizable` หรือ
`interface Summarizable`) แต่ Rust **ไม่มี class inheritance เลย** (ไม่มี `class`, ไม่มี `extends` แบบนั้น) struct
ทุกตัวเป็นอิสระจากกันโดยสมบูรณ์ตามที่เรียนมาตั้งแต่ Part 9

สิ่งที่ Rust มีให้แทนคือ **trait** — กลไกที่ประกาศ "สัญญาเรื่องพฤติกรรม" แยกออกมาต่างหากจากตัว type แล้วให้ type ไหนก็ตาม
(ไม่ว่าจะเกี่ยวข้องกันในทางธุรกิจหรือไม่ก็ตาม) เข้ามา "ลงชื่อรับสัญญา" นั้นทีหลังได้อย่างอิสระ นี่คือหัวข้อที่เราจะเจาะลึก
ตลอดทั้งบทนี้

### 19.2 นิยาม Trait: สัญญา (Contract) ไม่ใช่การ Implement

**Trait** คือการประกาศว่า "ชนิดข้อมูลใดก็ตามที่ประกาศว่า implement trait นี้ ต้องมี method (หรือ associated
function/type/const) ตามที่ระบุไว้" — คำสำคัญคือ **trait เองไม่ได้บอกว่า method นั้นทำงานยังไง** มันบอกแค่
**ชื่อ method, parameter, และ return type** เท่านั้น พูดอีกแบบคือ trait คือ **สัญญา (contract)** ไม่ใช่การ implement
จริง — เปรียบเทียบง่าย ๆ ได้กับ **interface** ในภาษาอื่น (Java `interface`, TypeScript `interface`, Go `interface`)
แม้จะไม่เหมือนกันเป๊ะทุกรายละเอียด (trait มี default method ได้ ซึ่งจะเรียนในหัวข้อ 19.4-19.5)

ไวยากรณ์การนิยาม trait:

```rust
// trait Summary: สัญญาที่บอกว่า "type ใดก็ตามที่ implement Summary ต้องมี method summarize(&self) -> String"
// สังเกตว่า method signature จบด้วย ; ตรง ๆ ไม่มีวงเล็บปีกกา { } และไม่มี body เลยแม้แต่บรรทัดเดียว
// นี่คือความแตกต่างสำคัญที่สุดจาก fn ปกติ: fn ธรรมดาต้องมี body เสมอ แต่ method signature ใน trait
// (ที่ไม่ใช่ default method) ห้ามมี body — เพราะ trait บอกแค่ "ต้องมีอะไร" ไม่ได้บอกว่า "ทำยังไง"
trait Summary {
    fn summarize(&self) -> String;
}

fn main() {} // แค่โชว์ไวยากรณ์การนิยาม trait เฉย ๆ ยังไม่ implement ให้ type ไหน
```

ลองสังเกตรายละเอียดของ syntax นี้ให้ครบถ้วน:

- คำ keyword คือ `trait` ตามด้วยชื่อ (PascalCase ตาม convention เดียวกับชื่อ type ทุกชนิดที่เรียนมาตั้งแต่ Part 9)
- ภายในปีกกาของ trait คือรายการ method signature ที่ type ที่ implement ต้องมีให้ครบ
- `fn summarize(&self) -> String;` — สังเกตว่ามี `&self` เหมือน method ปกติทุกประการ (ทวนจาก Part 9: `&self` คือ
  "ยืมแบบอ่านอย่างเดียว") เพราะ trait method ก็คือ method จริง ๆ เพียงแต่ยังไม่ระบุว่า "ทำอะไรกับ `self`" ตรงนี้
- จบด้วย `;` ไม่ใช่ `{ }` — นี่คือสิ่งที่ทำให้ trait "เป็นแค่สัญญา" ไม่ใช่การ implement (เราจะเห็น "default method"
  ที่มี body จริงในหัวข้อ 19.4 ซึ่งเปลี่ยนความหมายไปเล็กน้อย)

ณ จุดนี้ trait `Summary` **ยังไม่ผูกกับ type ไหนเลย** ไม่มี `Product`, `Article` หรือ struct อื่นใดที่ "ตอบสนอง"
สัญญานี้ได้ ถ้าคุณลองเรียก `product.summarize()` ตอนนี้จะได้ error ทันทีว่าไม่มี method นี้ — trait ต้อง **ถูก
implement ให้กับ type จริง** ก่อนเสมอ ซึ่งเป็นเนื้อหาของหัวข้อถัดไป

### 19.3 Implement Trait ให้กับหลาย Type

การ "เติมเนื้อหาจริง" ให้สัญญาที่ trait ประกาศไว้ ใช้ไวยากรณ์ `impl TraitName for TypeName { ... }` — สังเกตว่า
ไวยากรณ์นี้คล้ายกับ `impl TypeName { ... }` ธรรมดาที่เรียนมาตั้งแต่ Part 9 มาก เพียงแค่เพิ่ม `TraitName for` เข้ามา
ตรงกลาง เพื่อบอกว่า "impl block นี้คือการ implement trait ตัวไหนให้กับ type ตัวไหน" (ไม่ใช่ inherent method ธรรมดา
ที่ไม่ผูกกับ trait ใดเลย):

```rust
// 19.2-19.3: นิยาม trait และ implement ให้หลาย type
trait Summary {
    // method signature ที่ไม่มี body เลย จบด้วย ; ตรง ๆ — นี่คือ "สัญญา" (contract)
    // บอกว่า "ชนิดข้อมูลใดก็ตามที่ implement Summary ต้องมี method summarize(&self) -> String"
    // แต่ trait เองไม่รู้และไม่สนใจว่าแต่ละ type จะคำนวณ String นั้นออกมาอย่างไร
    fn summarize(&self) -> String;
}

struct Product {
    name: String,
    price: f64,
}

struct Article {
    headline: String,
    author: String,
}

struct Advertisement {
    sponsor: String,
    slogan: String,
}

// impl Summary for Product: เติม "เนื้อหาจริง" ให้กับสัญญาที่ trait กำหนดไว้
impl Summary for Product {
    fn summarize(&self) -> String {
        format!("สินค้า: {} ราคา {:.2} บาท", self.name, self.price)
    }
}

impl Summary for Article {
    fn summarize(&self) -> String {
        format!("บทความ: \"{}\" โดย {}", self.headline, self.author)
    }
}

impl Summary for Advertisement {
    fn summarize(&self) -> String {
        format!("โฆษณาจาก {}: {}", self.sponsor, self.slogan)
    }
}

// ตอนนี้เขียนฟังก์ชันเดียวที่รับ "อะไรก็ได้ที่ implement Summary" ได้แล้ว
// (รายละเอียดไวยากรณ์ &impl Summary จะอธิบายเต็ม ๆ ในหัวข้อ 19.6 — ตอนนี้แค่โชว์ว่ามันใช้งานได้จริง)
fn print_brief(item: &impl Summary) {
    println!("{}", item.summarize());
}

fn main() {
    let product = Product {
        name: String::from("เมาส์ไร้สาย"),
        price: 299.0,
    };
    let article = Article {
        headline: String::from("Rust แซงหน้า C++ ในโพลนักพัฒนาปี 2026"),
        author: String::from("กองบรรณาธิการ"),
    };
    let ad = Advertisement {
        sponsor: String::from("RustCorp"),
        slogan: String::from("เขียนโค้ดปลอดภัย ไม่ต้องกลัว null pointer"),
    };

    // ฟังก์ชันเดียวกัน (print_brief) เรียกได้กับทั้ง 3 type ที่ไม่มีความสัมพันธ์กันเลยในแง่ struct
    print_brief(&product);
    print_brief(&article);
    print_brief(&ad);
}
```

ผลลัพธ์:

```
สินค้า: เมาส์ไร้สาย ราคา 299.00 บาท
บทความ: "Rust แซงหน้า C++ ในโพลนักพัฒนาปี 2026" โดย กองบรรณาธิการ
โฆษณาจาก RustCorp: เขียนโค้ดปลอดภัย ไม่ต้องกลัว null pointer
```

นี่คือจุดเปลี่ยนสำคัญเทียบกับ 19.1: **`print_brief` เป็นฟังก์ชันเดียว เขียนครั้งเดียว แต่ใช้ได้กับทุก type ที่
implement `Summary`** ไม่ว่าจะเป็น `Product`, `Article`, `Advertisement` หรือ type ใหม่ที่จะเพิ่มเข้ามาในอนาคต
(เช่น `Video`) ตราบใดที่ type นั้น `impl Summary for` ตัวเอง มันก็ใช้กับ `print_brief` ได้ทันทีโดย**ไม่ต้องแก้โค้ด
`print_brief` แม้แต่บรรทัดเดียว** — นี่คือสิ่งที่เรียกว่า **polymorphism** (พหุสัณฐาน — ความสามารถให้ code เดียวกัน
ทำงานกับหลาย type ที่มีพฤติกรรมต่างกัน) แบบที่ Rust ทำได้โดยไม่ต้องมี class inheritance เลย

ข้อสังเกตสำคัญอีกจุด: **`Product`, `Article`, `Advertisement` ไม่รู้จักกันเลย ไม่ได้ extend อะไรร่วมกัน** สิ่งเดียว
ที่เชื่อมโยงทั้งสามเข้าด้วยกันคือ **การที่ทั้งสามตัวเลือก `impl Summary for` ตัวเอง** — trait ไม่ได้ "ฝัง" อยู่ใน
struct ตอนประกาศเหมือนการ extends class ในภาษาอื่น แต่เป็นการ "เพิ่มพฤติกรรม" ให้ type ทีหลัง แยกจากกันโดยสิ้นเชิง
นี่คือดีไซน์ที่เรียกว่า **composition over inheritance** ในเชิงปฏิบัติ — คุณออกแบบ struct ให้เก็บข้อมูลตามธรรมชาติ
ของมันก่อน แล้วค่อยมา "แปะ" trait ที่ต้องการทีหลังได้อย่างอิสระ ไม่ต้องวางแผนลำดับชั้นการสืบทอดล่วงหน้าแบบภาษา OOP
ดั้งเดิม

**กฎสำคัญที่ต้องรู้**: เมื่อ `impl Summary for Product` แล้ว **ต้อง implement ทุก method ที่ trait ประกาศไว้ให้ครบ**
(ในกรณีนี้มีแค่ `summarize` ตัวเดียว) ถ้าลืม method ใดไป (และ method นั้นไม่มี default — จะเรียนเรื่อง default ใน
หัวข้อถัดไป) compiler จะปฏิเสธทันที พร้อม error ที่บอกตรง ๆ ว่า "not all trait items implemented" — เราจะเห็น error
จริงแบบนี้ในหัวข้อกับดักท้ายบท

### 19.4 Default Method: ใช้ตามที่มีให้ หรือ Override เอง

จนถึงตอนนี้ trait `Summary` มีแค่ method ที่ **ไม่มี body** (บังคับให้ทุก type ต้องเขียนเอง 100%) แต่ trait ยังมี
ความสามารถอีกอย่างที่ทรงพลังมาก: การให้ method มี **body จริง** ไว้เป็น "ค่าเริ่มต้น" (default) ที่ type ที่
implement trait นั้นจะ **ใช้ตามที่มีให้เลยก็ได้ หรือเขียนทับ (override) เองก็ได้**

```rust
// 19.4: default method อย่างง่าย — type ส่วนใหญ่ใช้ตามที่มีให้ มีแค่บางตัวที่ override
trait Summary {
    fn summarize(&self) -> String;

    // default method: มี body ให้เลย — type ที่ implement Summary "ไม่บังคับ" ต้องเขียน is_promotional เอง
    // ถ้าไม่เขียนทับ จะได้ค่าเริ่มต้นนี้ไปใช้โดยอัตโนมัติ
    fn is_promotional(&self) -> bool {
        false
    }
}

struct Product {
    name: String,
    price: f64,
}

struct Article {
    headline: String,
    author: String,
}

struct Advertisement {
    sponsor: String,
    slogan: String,
}

impl Summary for Product {
    fn summarize(&self) -> String {
        format!("สินค้า: {} ราคา {:.2} บาท", self.name, self.price)
    }
    // ไม่เขียน is_promotional เลย — ใช้ default (false) ไปเลย เพราะสินค้าไม่ใช่เนื้อหาโฆษณา
}

impl Summary for Article {
    fn summarize(&self) -> String {
        format!("บทความ: \"{}\" โดย {}", self.headline, self.author)
    }
    // เช่นเดียวกัน: บทความข่าวไม่ใช่โฆษณา ใช้ default is_promotional() = false ไปเลย
}

impl Summary for Advertisement {
    fn summarize(&self) -> String {
        format!("โฆษณาจาก {}: {}", self.sponsor, self.slogan)
    }

    // Advertisement คือกรณีที่ "ต้องการพฤติกรรมต่างจาก default" จึง override ทับ
    // เขียน method ชื่อเดียวกัน signature เดียวกันซ้ำใน impl block นี้ — compiler จะใช้ตัวนี้แทน default ของ trait
    fn is_promotional(&self) -> bool {
        true
    }
}

fn describe_flag(item: &impl Summary) -> String {
    if item.is_promotional() {
        format!("[โฆษณา] {}", item.summarize())
    } else {
        item.summarize()
    }
}

fn main() {
    let product = Product {
        name: String::from("เมาส์ไร้สาย"),
        price: 299.0,
    };
    let article = Article {
        headline: String::from("Rust แซงหน้า C++ ในโพลนักพัฒนาปี 2026"),
        author: String::from("กองบรรณาธิการ"),
    };
    let ad = Advertisement {
        sponsor: String::from("RustCorp"),
        slogan: String::from("เขียนโค้ดปลอดภัย ไม่ต้องกลัว null pointer"),
    };

    println!("{}", describe_flag(&product));
    println!("{}", describe_flag(&article));
    println!("{}", describe_flag(&ad));
}
```

ผลลัพธ์:

```
สินค้า: เมาส์ไร้สาย ราคา 299.00 บาท
บทความ: "Rust แซงหน้า C++ ในโพลนักพัฒนาปี 2026" โดย กองบรรณาธิการ
[โฆษณา] โฆษณาจาก RustCorp: เขียนโค้ดปลอดภัย ไม่ต้องกลัว null pointer
```

สังเกตความแตกต่างของไวยากรณ์ระหว่าง method ที่มี default กับที่ไม่มี:

```rust
trait Summary {
    fn summarize(&self) -> String;       // ไม่มี default — จบด้วย ; บังคับให้ implement ทุก type
    fn is_promotional(&self) -> bool {   // มี default — มี { } พร้อม body จริง เขียนทับได้แต่ไม่บังคับ
        false
    }
}

fn main() {} // แค่โชว์ไวยากรณ์การนิยาม trait เฉย ๆ ยังไม่ implement ให้ type ไหน
```

**สิ่งที่เกิดขึ้นเบื้องหลัง**: เมื่อ `Product` ไม่เขียน `is_promotional` เลยใน `impl Summary for Product`
compiler จะ "หยิบ" body ของ `is_promotional` จากตัวนิยาม trait มาใช้แทนโดยอัตโนมัติ เสมือนกับว่า `Product` เขียน
`impl Summary for Product` แบบเต็มแบบนี้เอง (แม้ในซอร์สโค้ดจริงที่คุณพิมพ์จะไม่มีบรรทัด `is_promotional` ปรากฏให้เห็น
เลยก็ตาม):

```rust
trait Summary {
    fn summarize(&self) -> String;
    fn is_promotional(&self) -> bool {
        false
    }
}

struct Product {
    name: String,
    price: f64,
}

// นี่คือสิ่งที่ compiler "มองเห็น" จริง ๆ หลังเติม default method ที่ Product ไม่ได้เขียนเองเข้าไปให้ครบ
impl Summary for Product {
    fn summarize(&self) -> String {
        format!("สินค้า: {} ราคา {:.2} บาท", self.name, self.price)
    }

    // บรรทัดนี้ compiler เติมให้เองเบื้องหลัง แม้ในซอร์สโค้ดที่คุณพิมพ์จริงจะไม่มีเลยก็ตาม
    fn is_promotional(&self) -> bool {
        false
    }
}

fn main() {
    let product = Product { name: String::from("เมาส์ไร้สาย"), price: 299.0 };
    println!("{} (โฆษณา: {})", product.summarize(), product.is_promotional());
}
```

ในทางกลับกัน
เมื่อ `Advertisement` เขียน `fn is_promotional(&self) -> bool { true }` ใน `impl` block ของตัวเอง compiler จะ
**ใช้ตัวที่ type เขียนเอง แทนที่ default ของ trait ไปเลยโดยสมบูรณ์** ไม่มีการ "รวม" logic ทั้งสองเข้าด้วยกันแต่
อย่างใด — ถ้า override แล้ว default ของ trait จะไม่ถูกเรียกใช้อีกต่อไปสำหรับ type นั้นโดยเด็ดขาด

**ประโยชน์เชิงปฏิบัติของ default method**: ลองนึกภาพว่า trait `Summary` มี type ที่ implement มันอยู่ 50 ชนิด และ
45 ชนิดในนั้นต้องการ `is_promotional() -> false` เหมือนกันหมด — ถ้าไม่มี default method คุณจะต้องเขียน
`fn is_promotional(&self) -> bool { false }` ซ้ำ ๆ ถึง 45 ครั้งในแต่ละ `impl` block ซึ่งขัดกับหลัก DRY อย่างรุนแรง
default method ทำให้เขียน "พฤติกรรมปกติที่ใช้ได้กับ type ส่วนใหญ่" ไว้ที่จุดเดียว แล้วให้แค่ type ส่วนน้อยที่ต้องการ
พฤติกรรมต่างออกไป (`Advertisement` ในตัวอย่างนี้) เขียนโค้ดเพิ่มแค่ตรงจุดที่ต่างจริง ๆ

### 19.5 Default Method ที่พึ่งพา Method อื่นซึ่งยัง "ไม่มี Body" (Abstract Method)

รูปแบบของ default method ที่ทรงพลังที่สุด (และพบได้บ่อยที่สุดในโค้ด Rust ระดับ production จริง) คือ default method
ที่ **เรียกใช้ method อื่นในสัญญาเดียวกันซึ่งยังไม่มี body ณ ตอนที่ trait ถูกประกาศ** — พูดให้ชัดคือ: default
method รู้แค่ว่า "ต้องมี method X ให้เรียกได้แน่ ๆ เพราะ trait บังคับไว้" แต่ไม่รู้เลยว่า method X ของ type ไหนจะ
คำนวณอะไรออกมา ลองปรับ trait `Summary` ให้ใช้ pattern นี้:

```rust
// 19.5: default method ที่เรียกใช้ method อื่นซึ่งยังไม่ implement (abstract method)
trait Summary {
    // summarize_author เป็น method แบบไม่มี body — ทุก type ที่ implement Summary
    // "ต้อง" เขียนมันเอง ไม่มีทางเลี่ยง
    fn summarize_author(&self) -> String;

    // summarize มี default body ที่เรียกใช้ self.summarize_author() ข้างใน
    // แม้ว่า ณ จุดที่ประกาศ trait นี้ compiler ยังไม่รู้เลยว่า summarize_author()
    // ของ type ไหนจะ return string อะไร — มันรู้แค่ว่า "ไม่ว่า type ไหนก็ตามที่ผ่านมาถึงจุดนี้
    // ต้องมี summarize_author(&self) -> String ให้เรียกใช้ได้แน่ ๆ" เพราะ trait บังคับไว้แล้วข้างบน
    fn summarize(&self) -> String {
        format!("(อ่านเพิ่มเติมจาก {}...)", self.summarize_author())
    }
}

struct Product {
    name: String,
    price: f64,
    brand: String,
}

struct Article {
    headline: String,
    author: String,
}

struct Advertisement {
    sponsor: String,
    slogan: String,
}

impl Summary for Product {
    fn summarize_author(&self) -> String {
        self.brand.clone()
    }

    // Product เลือก override summarize() ทั้งหมด เพราะ default message ("อ่านเพิ่มเติมจาก...")
    // ไม่เหมาะกับสินค้า (สินค้าไม่มี "เพิ่มเติมให้อ่าน" แต่มีราคาที่ต้องโชว์ต่างหาก)
    fn summarize(&self) -> String {
        format!("สินค้า: {} ราคา {:.2} บาท (แบรนด์ {})", self.name, self.price, self.brand)
    }
}

impl Summary for Article {
    fn summarize_author(&self) -> String {
        self.author.clone()
    }
    // Article ไม่เขียน summarize() เลย — ใช้ default ที่ trait เตรียมไว้ให้ตรง ๆ
    // ซึ่งเหมาะกับบทความพอดี เพราะ "อ่านเพิ่มเติมจาก {ผู้เขียน}..." คือรูปแบบทีเซอร์ที่สมเหตุสมผล
}

impl Summary for Advertisement {
    fn summarize_author(&self) -> String {
        self.sponsor.clone()
    }

    fn summarize(&self) -> String {
        format!("โฆษณาจาก {}: {}", self.sponsor, self.slogan)
    }
}

fn main() {
    let product = Product {
        name: String::from("เมาส์ไร้สาย"),
        price: 299.0,
        brand: String::from("RustGear"),
    };
    let article = Article {
        headline: String::from("Rust แซงหน้า C++ ในโพลนักพัฒนาปี 2026"),
        author: String::from("กองบรรณาธิการ"),
    };
    let ad = Advertisement {
        sponsor: String::from("RustCorp"),
        slogan: String::from("เขียนโค้ดปลอดภัย ไม่ต้องกลัว null pointer"),
    };

    println!("{}", product.summarize());
    println!("{}", article.summarize());
    println!("{}", ad.summarize());

    // headline ไม่ได้ใช้ใน summarize ของ Article เลย (เพราะ default message ใช้แค่ author)
    // แต่ field นี้ยังมีประโยชน์ในส่วนอื่นของระบบ (เช่นแสดงหัวข้อเต็มในหน้าอ่านบทความจริง)
    println!("(หัวข้อเต็มของบทความ: {})", article.headline);
}
```

ผลลัพธ์:

```
สินค้า: เมาส์ไร้สาย ราคา 299.00 บาท (แบรนด์ RustGear)
(อ่านเพิ่มเติมจาก กองบรรณาธิการ...)
โฆษณาจาก RustCorp: เขียนโค้ดปลอดภัย ไม่ต้องกลัว null pointer
(หัวข้อเต็มของบทความ: Rust แซงหน้า C++ ในโพลนักพัฒนาปี 2026)
```

**ทำไม pattern นี้ compile ผ่านได้ ในเมื่อ default method เรียกใช้ method ที่ "ยังไม่มี body"?** คำตอบคือ compiler
ไม่ได้ตรวจสอบ trait ทีละ method แบบแยกส่วนกัน — มันตรวจสอบ **trait ทั้งก้อนพร้อมกัน** และเห็นว่า `summarize_author`
ถูกประกาศไว้ในสัญญาเดียวกัน (มี signature `fn summarize_author(&self) -> String` ชัดเจน) ดังนั้นภายใน default body
ของ `summarize` การเรียก `self.summarize_author()` จึงถูกมองว่า **ปลอดภัยเสมอ ไม่ว่า `Self` จะเป็น type ไหนก็ตาม**
เพราะไม่ว่า type ใดจะมาถึงจุดที่ implement `Summary` ได้สำเร็จ (compiler ยอมให้ผ่าน) นั่นแปลว่า type นั้น**ต้อง**มี
`summarize_author` อยู่แน่นอนแล้ว — เป็นการรับประกันที่ trait system ค้ำประกันไว้ ไม่ใช่การเดาหรือหวังว่าจะมี

นี่คือ pattern การออกแบบ trait ที่พบบ่อยมากในโค้ด Rust จริง: **แยก method เป็นสอง "ระดับ"**

- **method ที่เป็น "รายละเอียดเล็กที่สุดที่แต่ละ type ต้องบอกเอง"** (ในที่นี้คือ `summarize_author` — ใครคือผู้เขียน/
  เจ้าของเนื้อหานี้ ซึ่งแต่ละ type รู้คำตอบต่างกันแน่นอน ไม่มีค่า default ที่สมเหตุสมผล)
- **method ระดับสูงกว่าที่ "ประกอบ" รายละเอียดเล็กเหล่านั้นเข้าด้วยกันเป็นพฤติกรรมที่ซับซ้อนขึ้น** (ในที่นี้คือ
  `summarize` — ใช้ `summarize_author` มาประกอบเป็นข้อความทีเซอร์มาตรฐาน)

ข้อดีของการแยกแบบนี้คือ: type ส่วนใหญ่ (เช่น `Article`) แค่ตอบคำถามเล็ก ๆ ข้อเดียว (`summarize_author`) ก็ได้
พฤติกรรมระดับสูง (`summarize`) มาแบบมาตรฐานฟรี ๆ โดยไม่ต้องคิดเรื่อง formatting เองเลย ในขณะที่ type ที่ต้องการ
พฤติกรรมต่างออกไปจริง ๆ (เช่น `Product`, `Advertisement`) ก็ยัง override ได้เต็มที่เมื่อจำเป็น — ได้ทั้งความสะดวก
(สำหรับกรณีทั่วไป) และความยืดหยุ่น (สำหรับกรณีพิเศษ) พร้อมกันในกลไกเดียว

**หมายเหตุสำคัญ**: จากนี้ไปในบทที่เหลือ เราจะใช้ trait `Summary` เวอร์ชันนี้ (ที่มี `summarize_author` เป็น abstract
method และ `summarize` เป็น default method) เป็นตัวอย่างหลักต่อเนื่องไปตลอดทั้งบท

### 19.6 Trait เป็น Parameter: `impl Trait` เทียบกับ Generic `<T: Trait>`

ในหัวข้อ 19.3 เราเขียน `fn print_brief(item: &impl Summary)` ไปแล้วโดยยังไม่อธิบายไวยากรณ์นี้อย่างละเอียด ถึงเวลา
เจาะลึกมันอย่างเป็นทางการ พร้อมเทียบกับอีกรูปแบบหนึ่งที่**ความหมายเหมือนกันทุกประการ**:

```rust
trait Summary {
    fn summarize_author(&self) -> String;
    fn summarize(&self) -> String {
        format!("(อ่านเพิ่มเติมจาก {}...)", self.summarize_author())
    }
}

struct Product { name: String, price: f64, brand: String }
struct Article { headline: String, author: String }

impl Summary for Product {
    fn summarize_author(&self) -> String { self.brand.clone() }
    fn summarize(&self) -> String {
        format!("สินค้า: {} ราคา {:.2} บาท (แบรนด์ {})", self.name, self.price, self.brand)
    }
}

impl Summary for Article {
    fn summarize_author(&self) -> String { self.author.clone() }
}

// รูปแบบที่ 1: impl Trait syntax — อ่านง่าย กระชับ เหมาะกับ parameter เดียวที่ไม่ต้องอ้างชื่อ type ซ้ำที่อื่น
fn notify_impl_trait(item: &impl Summary) {
    println!("ประกาศ (impl Trait): {}", item.summarize());
}

// รูปแบบที่ 2: generic แบบเต็ม — ความหมายเหมือนกันกับรูปแบบที่ 1 เป๊ะ ๆ
// เพียงแค่ "ตั้งชื่อ" type parameter ว่า T ไว้อย่างชัดเจน ทำให้อ้างอิงชื่อ T ที่อื่นในซิกเนเจอร์ได้ด้วย
fn notify_generic<T: Summary>(item: &T) {
    println!("ประกาศ (generic <T>)  : {}", item.summarize());
}

// เหตุผลที่ต้องมีรูปแบบ generic: เมื่อต้องบังคับว่า "สอง parameter ต้องเป็น concrete type เดียวกัน"
// impl Trait ทำแบบนี้ไม่ได้ เพราะ &impl Summary แต่ละตำแหน่งคือ "ชนิดที่ implement Summary ชนิดใดก็ได้"
// อย่างเป็นอิสระจากกัน (อาจเป็นคนละ type กันเลยก็ได้) — แต่ <T: Summary> ผูก "ชื่อ T เดียว" ให้กับทั้งสองตำแหน่ง
fn compare_same_type<T: Summary>(a: &T, b: &T) -> String {
    format!("เทียบ:\n- {}\n- {}", a.summarize(), b.summarize())
}

// อีกเหตุผลที่ต้องมีรูปแบบ generic: ต้อง "อ้างชื่อ T" ในส่วนอื่นของ signature ด้วย เช่น return type
// impl Trait ที่ตำแหน่ง parameter ไม่ได้ตั้งชื่อให้ type นั้นเลย จึงเอาไปประกอบ Vec<T> ใน return type ไม่ได้
fn wrap_in_vec<T: Summary>(item: T) -> Vec<T> {
    vec![item]
}

fn main() {
    let product = Product { name: String::from("เมาส์ไร้สาย"), price: 299.0, brand: String::from("RustGear") };
    let article = Article { headline: String::from("ข่าว Rust"), author: String::from("กองบรรณาธิการ") };

    // ทั้งสองรูปแบบเรียกได้เหมือนกันทุกประการ ผลลัพธ์เหมือนกัน 100%
    notify_impl_trait(&product);
    notify_generic(&product);

    // impl Trait รับ argument คนละ type กันได้ในสองตำแหน่งที่ต่างฟังก์ชันกัน (คนละการเรียก)
    notify_impl_trait(&article);

    // compare_same_type ต้องการสอง argument ที่เป็น concrete type เดียวกันเท่านั้น
    let product2 = Product { name: String::from("แผ่นรองเมาส์"), price: 150.0, brand: String::from("RustGear") };
    println!("{}", compare_same_type(&product, &product2));

    println!("หัวข้อบทความ: {}", article.headline);
    let wrapped = wrap_in_vec(article);
    println!("จำนวนใน Vec หลัง wrap: {}", wrapped.len());
}
```

ผลลัพธ์:

```
ประกาศ (impl Trait): สินค้า: เมาส์ไร้สาย ราคา 299.00 บาท (แบรนด์ RustGear)
ประกาศ (generic <T>)  : สินค้า: เมาส์ไร้สาย ราคา 299.00 บาท (แบรนด์ RustGear)
ประกาศ (impl Trait): (อ่านเพิ่มเติมจาก กองบรรณาธิการ...)
เทียบ:
- สินค้า: เมาส์ไร้สาย ราคา 299.00 บาท (แบรนด์ RustGear)
- สินค้า: แผ่นรองเมาส์ ราคา 150.00 บาท (แบรนด์ RustGear)
หัวข้อบทความ: ข่าว Rust
จำนวนใน Vec หลัง wrap: 1
```

#### `&impl Summary` คือ "น้ำตาลไวยากรณ์" ของ `<T: Summary>`

ข้อเท็จจริงสำคัญที่ต้องจำ: `fn notify_impl_trait(item: &impl Summary)` **คือ syntax sugar ที่ compiler แปลงเป็น**
`fn notify_impl_trait<T: Summary>(item: &T)` **โดยอัตโนมัติเบื้องหลัง** ทั้งสองแบบ compile ออกมาเป็นโค้ดเดียวกัน
เป๊ะ ๆ ไม่มีความต่างด้าน performance เลยแม้แต่นิดเดียว (ทั้งคู่ใช้กลไกที่เรียกว่า **static dispatch** — compiler
จะ generate โค้ดเวอร์ชันเฉพาะเจาะจงสำหรับแต่ละ concrete type ที่ถูกเรียกใช้จริงตอน compile time ทั้งหมด ไม่มีการ
"เลือกพฤติกรรมตอน runtime" เกิดขึ้นเลย เรื่องนี้จะเจาะลึกกลไกภายในเต็มรูปแบบใน Part 21 เทียบกับ **dynamic dispatch**
ของ `dyn Trait`) ความแตกต่างเดียวระหว่างสองรูปแบบนี้คือ **ความสามารถในการอ้างอิงชื่อ type** ไม่ใช่ความหมายทาง
semantics หรือ performance

#### เมื่อไหร่ควรเลือก `impl Trait`

ใช้ `impl Trait` เมื่อสถานการณ์เข้าเงื่อนไขทั้งหมดนี้:

- ฟังก์ชันมี parameter ที่รับ trait นี้แค่**ตำแหน่งเดียว** (หรือหลายตำแหน่งที่ **ไม่จำเป็นต้องเป็น type เดียวกัน**)
- ไม่ต้องอ้างชื่อ type parameter ที่อื่นในซิกเนเจอร์เลย (ไม่มีใน return type, ไม่มีใน parameter อื่นที่ต้องตรงกัน)
- ต้องการความกระชับและอ่านง่ายที่สุด — `&impl Summary` สื่อความหมาย "รับอะไรก็ได้ที่ summarize ได้" ได้ทันทีตั้งแต่
  อ่านซิกเนเจอร์ครั้งแรก โดยไม่ต้องเห็น `<T: Summary>` ที่ต้อง "แปลกลับ" ในหัวก่อนว่า `T` คืออะไร

#### เมื่อไหร่ต้องเลือก generic `<T: Trait>`

ใช้รูปแบบ generic เต็มเมื่อเข้าเงื่อนไขข้อใดข้อหนึ่งต่อไปนี้ (`impl Trait` **ทำไม่ได้เลย** ในทุกกรณีนี้):

1. **ต้องการบังคับว่าหลาย parameter เป็น concrete type เดียวกัน** — เหมือน `compare_same_type<T: Summary>(a: &T, b: &T)`
   ข้างบน ถ้าเขียนเป็น `fn compare_same_type(a: &impl Summary, b: &impl Summary)` แทน ความหมายจะ**เปลี่ยนไปทันที**
   — กลายเป็น "รับสอง argument ที่ implement Summary ก็พอ จะเป็น type เดียวกันหรือคนละ type ก็ได้" ซึ่งอาจไม่ใช่สิ่ง
   ที่ต้องการ (เช่นถ้า logic ข้างในต้องเทียบ `a == b` แบบ `PartialEq` ที่ปกติต้องการ type เดียวกันทั้งสองข้าง)
2. **ต้องการอ้างชื่อ type parameter ในส่วนอื่นของ signature** — เหมือน `wrap_in_vec<T: Summary>(item: T) -> Vec<T>`
   ที่ต้อง "ผูก" ชื่อ `T` ไว้ตั้งแต่ parameter แล้วเอาไปใช้ต่อใน return type `Vec<T>` — `impl Trait` ที่ตำแหน่ง
   parameter ไม่ได้ให้ชื่อที่อ้างอิงได้แบบนี้เลย
3. **ต้องมี bound หลายตัวที่ซับซ้อน หรือมีหลาย type parameter ที่สัมพันธ์กัน** — เมื่อ bound ยาวขึ้นมาก การเขียน
   ทุกอย่างเป็น `impl Trait1 + Trait2 + Trait3` ซ้ำ ๆ หลายตำแหน่งจะอ่านยากกว่าการตั้งชื่อ `T` ครั้งเดียวแล้วเขียน
   bound ที่ตำแหน่งประกาศ `<T: ...>` เพียงจุดเดียว (จะเห็นตัวอย่างเต็มในหัวข้อ 19.7)

ลองดู error จริงถ้าพยายามส่ง `Product` กับ `Article` (คนละ concrete type) เข้า `compare_same_type<T: Summary>`
ที่บังคับว่าทั้งสอง argument ต้องเป็น `T` เดียวกัน:

```
error[E0308]: mismatched types
  --> src/main.rs:31:48
   |
31 |     println!("{}", compare_same_type(&product, &article));
   |                    -----------------           ^^^^^^^^ expected `&Product`, found `&Article`
   |                    |
   |                    arguments to this function are incorrect
   |
   = note: expected reference `&Product`
              found reference `&Article`
note: function defined here
  --> src/main.rs:22:4
   |
22 | fn compare_same_type<T: Summary>(a: &T, b: &T) -> String {
   |    ^^^^^^^^^^^^^^^^^                    -----
```

สังเกตว่า error นี้เกิดขึ้น**เพราะ** compiler เห็นว่า argument แรก (`&product`) กำหนดค่า `T = Product` ไปแล้ว
เมื่อเจอ argument ที่สอง (`&article`) ที่เป็น `&Article` ซึ่งไม่ตรงกับ `T = Product` ที่ตกลงไปแล้ว มันจึงปฏิเสธทันที
— นี่คือหลักฐานที่ยืนยันชัดเจนว่า `<T: Summary>` "ผูก" ทั้งสอง parameter ให้เป็น type เดียวกันจริง ๆ ไม่ใช่แค่
"implement Summary ก็พอ" แบบที่ `impl Trait` ให้ความหมายไว้

### 19.7 Multiple Trait Bounds และ `where` Clause

บ่อยครั้งที่ type parameter หนึ่งตัวต้อง implement **มากกว่าหนึ่ง trait พร้อมกัน** เพื่อให้ฟังก์ชันเรียกใช้ method
จากทุก trait ที่ต้องการได้ครบ — ใช้เครื่องหมาย `+` เพื่อรวม bound หลายตัวเข้าด้วยกัน:

```rust
use std::fmt;

trait Summary {
    fn summarize_author(&self) -> String;
    fn summarize(&self) -> String {
        format!("(อ่านเพิ่มเติมจาก {}...)", self.summarize_author())
    }
}

#[derive(Debug, Clone)]
struct Product {
    name: String,
    price: f64,
    brand: String,
}

impl Summary for Product {
    fn summarize_author(&self) -> String {
        self.brand.clone()
    }
    fn summarize(&self) -> String {
        format!("สินค้า: {} ราคา {:.2} บาท (แบรนด์ {})", self.name, self.price, self.brand)
    }
}

// implement Display ให้ Product ด้วย เพื่อสร้างสถานการณ์ที่ต้องการ "หลาย trait bound พร้อมกัน"
impl fmt::Display for Product {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{} ({:.2} บาท)", self.name, self.price)
    }
}

// multiple trait bounds ด้วย + : T ต้อง implement ทั้ง Summary และ Display พร้อมกัน
// ถึงจะเรียก method ของทั้งสอง trait ได้ในฟังก์ชันเดียว (item.summarize() จาก Summary, "{item}" จาก Display)
fn print_full<T: Summary + fmt::Display>(item: &T) {
    println!("แบบย่อ (Summary): {}", item.summarize());
    println!("แบบเต็ม (Display): {item}");
}

// เมื่อ trait bound ยาวขึ้นมาก (หลาย trait, หรือหลาย type parameter) การเขียนในวงเล็บ <> จะอ่านยากขึ้นเรื่อย ๆ
// where clause ช่วยแยกส่วน "ชื่อ type parameter" ออกจาก "เงื่อนไขที่ต้องผ่าน" ให้อ่านง่ายขึ้นมาก
fn log_summary_details<T>(item: &T)
where
    T: Summary + Clone + fmt::Debug,
{
    let cloned = item.clone();
    println!("log: {}", item.summarize());
    println!("log (debug ของสำเนา): {cloned:?}");
}

fn main() {
    let product = Product {
        name: String::from("เมาส์ไร้สาย"),
        price: 299.0,
        brand: String::from("RustGear"),
    };

    print_full(&product);
    log_summary_details(&product);
}
```

ผลลัพธ์:

```
แบบย่อ (Summary): สินค้า: เมาส์ไร้สาย ราคา 299.00 บาท (แบรนด์ RustGear)
แบบเต็ม (Display): เมาส์ไร้สาย (299.00 บาท)
log: สินค้า: เมาส์ไร้สาย ราคา 299.00 บาท (แบรนด์ RustGear)
log (debug ของสำเนา): Product { name: "เมาส์ไร้สาย", price: 299.0, brand: "RustGear" }
```

**ทำไมต้องมี `Display` เป็น bound เพิ่ม**: ถ้าเขียน `fn print_full<T: Summary>(item: &T)` เฉย ๆ (ไม่มี `+ Display`)
แล้วพยายามเขียน `println!("{item}")` ข้างใน จะ compile ไม่ผ่านทันที เพราะ `{}` (Display formatting) ต้องการให้
`T` implement `Display` แต่ bound ของฟังก์ชันบอกไว้แค่ `T: Summary` เท่านั้น — compiler **ไม่ยอมสมมติเอาเองว่า
T น่าจะ implement Display ด้วย** แม้ในความเป็นจริง `Product` (ที่เราจะเรียกใช้จริง) จะ implement ทั้งสองก็ตาม เพราะ
generic function ต้อง compile ผ่านสำหรับ **T ตัวใดก็ได้ที่ผ่าน bound ที่ประกาศไว้เท่านั้น** ไม่ใช่แค่ type ที่คุณ
บังเอิญตั้งใจจะเรียกใช้จริง — นี่คือหลักการ "ความปลอดภัยที่ตรวจสอบได้แบบคงที่" (static guarantee) ที่ทำให้ generic
ใน Rust ปลอดภัยกว่า generic/template ในหลายภาษา: **bound ที่ประกาศไว้คือทุกสิ่งที่ compiler รู้เกี่ยวกับ T ได้**
ไม่มากไปกว่านั้นแม้แต่นิดเดียว

**`where` clause คือไวยากรณ์เดียวกัน เพียงแค่จัดวางต่างที่**: `fn log_summary_details<T>(item: &T) where T: Summary + Clone + fmt::Debug`
มีความหมายเหมือนกับ `fn log_summary_details<T: Summary + Clone + fmt::Debug>(item: &T)` ทุกประการ 100% — ไม่มี
ความต่างด้านความหมายหรือ performance เลย ต่างกันแค่**การจัดรูปแบบให้อ่านง่าย** เมื่อ:

- มี type parameter หลายตัว ที่แต่ละตัวมี bound ต่างกัน (เช่น `<T: A + B, U: C + D + E>` จะยาวและอ่านยากมากถ้าอยู่ใน
  วงเล็บเดียวติดกับชื่อฟังก์ชัน)
- bound ของ type parameter ตัวเดียวมีหลาย trait รวมกัน (เหมือนตัวอย่างข้างบนที่มีถึง 3 trait: `Summary + Clone + Debug`)

`where` clause แยก "รายการ type parameter" (`<T>`) ออกจาก "เงื่อนไขที่ type parameter ต้องผ่าน" (`where T: ...`)
ทำให้ชื่อฟังก์ชันกับ parameter list อ่านง่ายในบรรทัดแรก แล้วไปดูรายละเอียดเงื่อนไขในบรรทัดถัดไปแยกต่างหาก — สำหรับ
bound สั้น ๆ (1 trait, 1 type parameter) ส่วนใหญ่คนยังเขียนแบบ inline (`<T: Summary>`) เพราะกระชับกว่า แต่ทันทีที่
bound เริ่มซับซ้อนขึ้น `where` clause คือรูปแบบที่ community ยอมรับว่าอ่านง่ายกว่าอย่างชัดเจน

### 19.8 Return Types ที่ Implement Trait: ข้อจำกัดสำคัญของ `impl Trait`

เช่นเดียวกับที่ `impl Trait` ใช้เป็น parameter type ได้ มันก็ใช้เป็น **return type** ได้เช่นกัน — บอกผู้เรียกว่า
"ฟังก์ชันนี้คืนค่าบางอย่างที่ implement trait นี้แน่ ๆ" โดยไม่ต้องเปิดเผย concrete type จริงเลย:

```rust
trait Summary {
    fn summarize_author(&self) -> String;
    fn summarize(&self) -> String {
        format!("(อ่านเพิ่มเติมจาก {}...)", self.summarize_author())
    }
}

struct Product { name: String, price: f64, brand: String }

impl Summary for Product {
    fn summarize_author(&self) -> String { self.brand.clone() }
    fn summarize(&self) -> String {
        format!("สินค้า: {} ราคา {:.2} บาท (แบรนด์ {})", self.name, self.price, self.brand)
    }
}

// return type เขียนเป็น "impl Summary" — บอกผู้เรียกว่า "ฉันคืนอะไรบางอย่างที่ implement Summary"
// โดยไม่ต้องเปิดเผยว่า concrete type จริง ๆ คือ Product — ผู้เรียกรู้แค่ว่าเรียก .summarize() ได้แน่นอน
fn make_featured_product_summary() -> impl Summary {
    Product {
        name: String::from("หูฟังไร้สาย รุ่นพรีเมียม"),
        price: 3990.0,
        brand: String::from("RustGear"),
    }
}

fn main() {
    let item = make_featured_product_summary();
    println!("{}", item.summarize());
}
```

ผลลัพธ์:

```
สินค้า: หูฟังไร้สาย รุ่นพรีเมียม ราคา 3990.00 บาท (แบรนด์ RustGear)
```

โค้ดนี้ทำงานได้อย่างสมบูรณ์แบบ — แต่นี่คือจุดที่มือใหม่มักเข้าใจผิดบ่อยที่สุดเกี่ยวกับ `impl Trait` ในตำแหน่ง return
type: **มันไม่ได้แปลว่า "คืนอะไรก็ได้ที่ implement Summary ในแต่ละครั้งที่เรียก"** — ความจริงคือ **compiler ต้อง
รู้ concrete type ที่แน่นอนเพียงหนึ่งเดียวของ return value ตั้งแต่ compile time เสมอ** เพียงแต่ **ไม่บอกชื่อ type
นั้นให้ผู้เรียกเห็นตรง ๆ** เท่านั้นเอง (compiler ยังต้องรู้ภายในว่า return type จริง ๆ คือ `Product` เพื่อคำนวณขนาด
memory ที่ต้องใช้ตอน return ค่า — นี่คือความต่างสำคัญจาก `dyn Trait` ที่จะเห็นในหัวข้อถัดไป)

ลองดูว่าเกิดอะไรขึ้นถ้าพยายาม return **สอง concrete type ที่ต่างกัน** ขึ้นกับเงื่อนไข โดยยังใช้ `impl Summary`
เป็น return type:

```rust
trait Summary {
    fn summarize_author(&self) -> String;
    fn summarize(&self) -> String {
        format!("(อ่านเพิ่มเติมจาก {}...)", self.summarize_author())
    }
}

struct Product { name: String, price: f64, brand: String }
struct Article { headline: String, author: String }

impl Summary for Product {
    fn summarize_author(&self) -> String { self.brand.clone() }
    fn summarize(&self) -> String {
        format!("สินค้า: {} ราคา {:.2} บาท (แบรนด์ {})", self.name, self.price, self.brand)
    }
}

impl Summary for Article {
    fn summarize_author(&self) -> String { self.author.clone() }
}

// พยายาม return "Product หรือ Article" ขึ้นกับเงื่อนไข โดยยังใช้ impl Summary เป็น return type
fn make_summary(is_featured: bool) -> impl Summary {
    if is_featured {
        Product {
            name: String::from("หูฟังไร้สาย รุ่นพรีเมียม"),
            price: 3990.0,
            brand: String::from("RustGear"),
        }
    } else {
        Article {
            headline: String::from("ข่าว Rust"),
            author: String::from("กองบรรณาธิการ"),
        }
    }
}

fn main() {
    let item = make_summary(true);
    println!("{}", item.summarize());
}
```

โค้ดนี้ **compile ไม่ผ่าน**:

```
error[E0308]: `if` and `else` have incompatible types
  --> src/main.rs:24:9
   |
17 | /       if is_featured {
18 | | /         Product {
19 | | |             name: String::from("หูฟังไร้สาย รุ่นพรีเมียม"),
20 | | |             price: 3990.0,
21 | | |             brand: String::from("RustGear"),
22 | | |         }
   | | |_________- expected because of this
23 | |       } else {
24 | | /         Article {
25 | | |             headline: String::from("ข่าว Rust"),
26 | | |             author: String::from("กองบรรณาธิการ"),
27 | | |         }
   | | |_________^ expected `Product`, found `Article`
28 | |       }
   | |_______- `if` and `else` have incompatible types
   |
help: you could change the return type to be a boxed trait object
   |
16 - fn make_summary(is_featured: bool) -> impl Summary {
16 + fn make_summary(is_featured: bool) -> Box<dyn Summary> {
   |
help: if you change the return type to expect trait objects, box the returned expressions
   |
18 ~         Box::new(Product {
19 |             name: String::from("หูฟังไร้สาย รุ่นพรีเมียม"),
20 |             price: 3990.0,
21 |             brand: String::from("RustGear"),
22 ~         })
23 |     } else {
24 ~         Box::new(Article {
25 |             headline: String::from("ข่าว Rust"),
26 |             author: String::from("กองบรรณาธิการ"),
27 ~         })
   |
```

**นี่คือข้อจำกัดที่สำคัญที่สุดของ `impl Trait` ในตำแหน่ง return type**: **ฟังก์ชันหนึ่งตัวต้อง return concrete
type เดียวกันเสมอในทุกเส้นทางการทำงาน (ทุก branch)** ไม่ว่า branch นั้นจะถูกเลือกจริงตอน runtime หรือไม่ก็ตาม —
สังเกตว่า error message บอกตรง ๆ ว่า **"if and else have incompatible types" — `expected Product, found Article`**
เหมือนกับ error ที่เจอตอนพยายามสร้าง array `[product, article]` ในหัวข้อ 19.1 เป๊ะ ๆ นี่ไม่ใช่เรื่องบังเอิญ:
**เหตุผลเบื้องหลังเหมือนกัน** — compiler ต้องรู้ **ขนาด memory ที่แน่นอนของ return value** ตั้งแต่ compile time
เสมอ (`Product` กับ `Article` มีขนาดต่างกัน เพราะมี field ต่างกัน) `impl Summary` เป็นแค่วิธี "ซ่อนชื่อ type" จาก
ผู้เรียก แต่ **ไม่ได้ทำให้ type นั้นมีขนาดไม่แน่นอนได้** — มันยังต้องเป็น**หนึ่ง concrete type ที่แน่นอน**เสมอ
เพียงแค่ไม่เปิดเผยชื่อให้เห็นเท่านั้นเอง

### 19.9 `Box<dyn Trait>`: ทางออกเมื่อต้อง Return หลาย Concrete Type

สังเกตว่า error message ในหัวข้อก่อนหน้า **compiler เสนอทางแก้มาให้เองเลย**: เปลี่ยน return type จาก `impl Summary`
เป็น `Box<dyn Summary>` มาดูกันว่าทำไมวิธีนี้ถึงแก้ปัญหาได้:

```rust
trait Summary {
    fn summarize_author(&self) -> String;
    fn summarize(&self) -> String {
        format!("(อ่านเพิ่มเติมจาก {}...)", self.summarize_author())
    }
}

struct Product { name: String, price: f64, brand: String }
struct Article { headline: String, author: String }

impl Summary for Product {
    fn summarize_author(&self) -> String { self.brand.clone() }
    fn summarize(&self) -> String {
        format!("สินค้า: {} ราคา {:.2} บาท (แบรนด์ {})", self.name, self.price, self.brand)
    }
}

impl Summary for Article {
    fn summarize_author(&self) -> String { self.author.clone() }
}

// เปลี่ยน return type จาก "impl Summary" เป็น "Box<dyn Summary>"
// Box<dyn Summary> คือ "กล่องบน heap ที่เก็บค่าอะไรก็ได้ที่ implement Summary โดยไม่ต้องรู้ concrete type
// ที่แน่นอนตั้งแต่ compile time" — ต่างจาก impl Summary ที่ compiler ต้อง "รู้" concrete type ที่แน่นอน
// เพียงหนึ่งเดียวตั้งแต่ compile time (แค่ไม่บอกผู้เรียกตรง ๆ เท่านั้นเอง)
// รายละเอียดเชิงลึกเรื่อง dynamic dispatch, vtable, และ dyn Trait แบบเต็มรูปแบบจะเรียนใน Part 21
fn make_summary(is_featured: bool) -> Box<dyn Summary> {
    if is_featured {
        Box::new(Product {
            name: String::from("หูฟังไร้สาย รุ่นพรีเมียม"),
            price: 3990.0,
            brand: String::from("RustGear"),
        })
    } else {
        Box::new(Article {
            headline: String::from("ข่าว Rust"),
            author: String::from("กองบรรณาธิการ"),
        })
    }
}

fn main() {
    let featured = make_summary(true);
    let regular = make_summary(false);

    println!("{}", featured.summarize());
    println!("{}", regular.summarize());
}
```

ผลลัพธ์:

```
สินค้า: หูฟังไร้สาย รุ่นพรีเมียม ราคา 3990.00 บาท (แบรนด์ RustGear)
(อ่านเพิ่มเติมจาก กองบรรณาธิการ...)
```

**ทำไมวิธีนี้แก้ปัญหาได้**: ทวนจาก Part 9 (`Box<T>` จะเรียนเต็มรูปแบบใน Part 27 แต่ตอนนี้รู้แค่หลักการพอ) — `Box<T>`
คือค่าที่ถูกจัดเก็บบน **heap** แทน stack โดย `Box<T>` เองมีขนาดคงที่เสมอไม่ว่า `T` จะเป็นอะไร (มันคือ pointer
ตัวเดียวที่ชี้ไปยังข้อมูลจริงบน heap ขนาด 8 bytes บนเครื่อง 64-bit) — เมื่อรวมกับ `dyn Summary` (อ่านว่า "dynamic
Summary" — บอกว่า "ค่าที่กล่องนี้เก็บอยู่ implement trait Summary แต่ concrete type จริงจะถูกตัดสินใจตอน runtime")
ทำให้ `Box<dyn Summary>` มีขนาดคงที่เสมอ (แค่ pointer + ข้อมูลเสริมเล็กน้อยสำหรับหา method ที่ถูกต้องตอน runtime)
**ไม่ว่าข้างในจะเป็น `Product` หรือ `Article` ก็ตาม** — นี่คือเหตุผลที่ `if`/`else` สอง branch คืนค่า concrete type
ต่างกันได้แล้วในกรณีนี้: เพราะทั้งสอง branch คืนค่าที่ **หน้าตาเหมือนกันจากมุมมองภายนอก** (คือ `Box<dyn Summary>`
ที่มีขนาดเท่ากันเป๊ะ) แม้ข้างในจะเป็นคนละ type กันจริง ๆ

ข้อแลกเปลี่ยน (trade-off) ที่ต้องรู้ไว้ก่อน (จะเจาะลึกเต็มรูปแบบใน **Part 21**):

- **ต้นทุนที่เพิ่มขึ้น**: ต้องจอง heap memory (ผ่าน `Box::new`) และการเรียก method ผ่าน `dyn Trait` ใช้กลไก
  **dynamic dispatch** (ค้นหา method ที่ถูกต้องผ่าน vtable ตอน runtime) ซึ่งช้ากว่า **static dispatch** ของ
  `impl Trait`/generic เล็กน้อย (แต่ในโปรแกรมส่วนใหญ่ความต่างนี้ไม่มีผลกระทบจนต้องกังวล)
- **สิ่งที่ได้กลับมา**: ความยืดหยุ่นในการเก็บ/return หลาย concrete type ที่ implement trait เดียวกันไว้ด้วยกันได้
  จริง ทั้งใน collection (`Vec<Box<dyn Summary>>`) และใน return position ที่มีหลาย branch ต่าง concrete type

**กฎง่าย ๆ สำหรับตอนนี้**: ถ้าฟังก์ชัน return concrete type เดียวเสมอ (แม้จะซ่อนชื่อไว้) ใช้ `impl Trait` เพราะเร็ว
กว่าและไม่ต้องจอง heap memory เพิ่ม — ถ้าต้อง return หลาย concrete type ที่ implement trait เดียวกันขึ้นกับเงื่อนไข
รันไทม์ (เหมือนตัวอย่างนี้) ใช้ `Box<dyn Trait>` เป็นทางออกที่ตรงไปตรงมาที่สุด รายละเอียดเรื่อง trait object, vtable,
object safety, และเมื่อไหร่ควรใช้ `dyn Trait` แทน generic ในสถานการณ์อื่น ๆ (ไม่ใช่แค่ return type) จะเป็นเนื้อหาหลัก
ของ **Part 21: Traits ขั้นสูง**

### 19.10 Conditional Method Implementation: `impl<T: Bound> Type<T>`

จนถึงตอนนี้เราใช้ trait bound กับ **ฟังก์ชัน** เท่านั้น แต่ trait bound ยังใช้กับ **`impl` block ของ generic struct**
ได้ด้วย — เพื่อบอกว่า "method บางตัวจะมีอยู่เฉพาะเมื่อ type parameter ผ่านเงื่อนไขที่กำหนด" ลองดูตัวอย่างคลาสสิก
ที่ทวนแนวคิด generic struct จาก Part 18:

```rust
use std::fmt::Display;

// Pair<T> ทั่วไป: เก็บค่าสองตัวชนิดเดียวกัน — ทวนแนวคิด generic struct จาก Part 18
struct Pair<T> {
    first: T,
    second: T,
}

// impl block แรก: ไม่มี trait bound เพิ่มเติมเลย (นอกจากที่ Pair<T> ประกาศไว้)
// ทำให้ new() ใช้ได้กับ Pair<T> ทุกชนิด T โดยไม่มีข้อจำกัดอะไรเพิ่ม
impl<T> Pair<T> {
    fn new(first: T, second: T) -> Self {
        Self { first, second }
    }
}

// impl block ที่สอง: เพิ่มเงื่อนไข T: Display + PartialOrd เข้ามา
// นี่คือ "conditional implementation" — method cmp_display() จะมีอยู่ "ก็ต่อเมื่อ" T ที่ใช้จริง
// implement ทั้ง Display (พิมพ์ค่าออกมาได้) และ PartialOrd (เทียบมากกว่า/น้อยกว่าได้) พร้อมกันเท่านั้น
// ถ้า T ไม่ผ่านเงื่อนไขนี้ Pair<T> ตัวนั้นจะยังใช้ .new() ได้ปกติ แต่จะไม่มี .cmp_display() ให้เรียกเลย
impl<T: Display + PartialOrd> Pair<T> {
    fn cmp_display(&self) {
        if self.first >= self.second {
            println!("ค่าที่มากกว่าคือ first = {}", self.first);
        } else {
            println!("ค่าที่มากกว่าคือ second = {}", self.second);
        }
    }
}

// type ตัวอย่างที่ "ไม่" implement Display หรือ PartialOrd เลย — ใช้ Pair<T>::new() ได้ปกติ
// แต่จะเรียก .cmp_display() ไม่ได้เด็ดขาด (compiler จะปฏิเสธ ไม่ใช่ runtime panic)
struct RawBlob {
    #[allow(dead_code)]
    bytes: Vec<u8>,
}

fn main() {
    let numbers = Pair::new(5, 10);
    numbers.cmp_display(); // i32 implement ทั้ง Display และ PartialOrd อยู่แล้ว — เรียกได้ทันที

    let words = Pair::new(String::from("แอปเปิล"), String::from("กล้วย"));
    words.cmp_display(); // String ก็ implement ทั้งสอง trait เช่นกัน

    // Pair<RawBlob> ยังสร้างได้ปกติด้วย ::new() เพราะ impl<T> Pair<T> ไม่มีเงื่อนไขอะไรเลย
    let blobs = Pair::new(
        RawBlob { bytes: vec![1, 2, 3] },
        RawBlob { bytes: vec![4, 5, 6] },
    );
    // แต่ blobs.cmp_display() จะเรียกไม่ได้ — RawBlob ไม่ implement Display/PartialOrd
    // ลองเปิดคอมเมนต์บรรทัดล่างดูจะเจอ error E0599 (method not found due to unsatisfied trait bounds)
    // blobs.cmp_display();
    println!("สร้าง Pair<RawBlob> ได้ปกติ แต่เรียก cmp_display ไม่ได้");
    let _ = blobs; // กันไม่ให้ warning เตือนว่า blobs ไม่ได้ใช้
}
```

ผลลัพธ์:

```
ค่าที่มากกว่าคือ second = 10
ค่าที่มากกว่าคือ first = แอปเปิล
สร้าง Pair<RawBlob> ได้ปกติ แต่เรียก cmp_display ไม่ได้
```

ถ้าลองเปิดคอมเมนต์บรรทัด `blobs.cmp_display();` จริง จะได้ error:

```
error[E0599]: the method `cmp_display` exists for struct `Pair<RawBlob>`, but its trait bounds were not satisfied
   |
   = note: the following trait bounds were not satisfied:
           `RawBlob: PartialOrd`
           `RawBlob: std::fmt::Display`
help: consider annotating `RawBlob` with `#[derive(PartialEq, PartialOrd)]`
```

**สิ่งที่เกิดขึ้นเบื้องหลัง**: `Pair<T>` เป็น struct เดียว แต่มี `impl` block ได้หลายบล็อก (ทวนจาก Part 9 ว่า
`impl` แยกจากการนิยาม struct ได้อย่างอิสระ) แต่ละ `impl` block สามารถมี **เงื่อนไข trait bound ของตัวเอง** ที่ไม่
เหมือนกันได้ — `impl<T> Pair<T>` (ไม่มี bound) ให้ method ที่ใช้ได้กับ `T` ทุกชนิดแบบไม่มีข้อยกเว้น ในขณะที่
`impl<T: Display + PartialOrd> Pair<T>` ให้ method เพิ่มเติมที่ **มีอยู่ก็ต่อเมื่อ** `T` ที่ใช้จริงผ่านเงื่อนไข
นั้น — ทั้งหมดนี้ตรวจสอบตอน **compile time** ล้วน ๆ ไม่มีการเช็คตอน runtime เลยแม้แต่นิดเดียว (`Pair<RawBlob>`
"มีตัวตนอยู่" ในโปรแกรมได้ปกติผ่าน `::new()` เพราะ `impl<T> Pair<T>` ครอบคลุมทุก `T` แต่ compiler รู้ตั้งแต่
compile time แล้วว่า `Pair<RawBlob>` ไม่มี method `cmp_display` ให้เรียก จึงปฏิเสธทันทีที่เห็นการเรียกใช้ ไม่ต้อง
รอให้โปรแกรมรันจริงแล้วค่อย panic)

นี่คือรูปแบบที่ standard library ของ Rust ใช้อย่างกว้างขวางมาก เช่น `Vec<T>` มี method `.sort()` ที่ต้องการ
`T: Ord` เท่านั้น (ถ้า `T` ไม่ implement `Ord` จะสร้าง `Vec<T>` ได้ปกติ แต่เรียก `.sort()` ไม่ได้) เป็นดีไซน์แบบ
เดียวกันเป๊ะกับ `Pair<T>` ในตัวอย่างนี้

### 19.11 Blanket Implementation: Implement Trait ให้ "ทุก Type ที่ผ่านเงื่อนไข" ในครั้งเดียว

ขยายแนวคิดจากหัวข้อก่อนหน้าไปอีกขั้น: จนถึงตอนนี้เราเขียน `impl SomeTrait for ConcreteType` ทีละ type เสมอ (เช่น
`impl Summary for Product`, `impl Summary for Article`) แต่ Rust ยังให้เขียน **`impl SomeTrait for T` โดย `T`
เป็น generic type parameter ที่มี bound** ได้ด้วย — เรียกว่า **blanket implementation** (การ implement trait
ให้กับ **type ใดก็ตามที่ผ่านเงื่อนไขที่กำหนด** ในครั้งเดียว ครอบคลุมทุก type ทั้งที่มีอยู่แล้วและที่จะเพิ่มเข้ามา
ในอนาคต)

ตัวอย่างที่โด่งดังที่สุดของ pattern นี้คือใน standard library เอง: **`impl<T: Display> ToString for T`** — ทุก
type ที่ implement `Display` (พิมพ์ค่าออกมาเป็นข้อความให้มนุษย์อ่านได้) จะได้ method `.to_string()` มา**อัตโนมัติ
ทันที** โดยที่ผู้เขียน type นั้นไม่ต้องเขียน `impl ToString for MyType` เองเลยแม้แต่บรรทัดเดียว — นี่คือเหตุผลที่
คุณเรียก `.to_string()` กับ `i32`, `f64`, หรือ struct ที่ implement `Display` เองได้ตั้งแต่ Part 3-9 โดยไม่ต้อง
สงสัยว่ามันมาจากไหน

เราไม่สามารถ implement `ToString` ซ้ำเองได้ (เพราะ std implement blanket ไว้ให้ครบทุก `Display` type แล้ว การ
implement ซ้ำจะชนกันทันที) แต่ลองสร้าง trait ของเราเองที่ทำงานคล้ายกัน เพื่อดูกลไกนี้แบบลงมือเขียนจริง:

```rust
use std::fmt::Display;

// trait ของเราเอง สำหรับสาธิต blanket implementation (เราจะไม่ implement ToString ตรง ๆ
// เพราะ standard library implement blanket ให้ทุก Display type ไว้แล้ว การพยายาม implement ซ้ำ
// จะชนกับของ std ทันที — เราจึงสร้าง trait ใหม่ชื่อ Loud ที่ทำงานคล้ายกันเพื่อสาธิตแนวคิดแทน)
trait Loud {
    fn shout(&self) -> String;
}

// blanket implementation: "implement Loud ให้กับทุก type T ที่ implement Display"
// พูดอีกแบบคือ: ไม่ต้องเขียน impl Loud for Product, impl Loud for Article ฯลฯ ทีละตัวเลย
// ตราบใดที่ type นั้น implement Display อยู่แล้ว มันจะได้ Loud มาโดยอัตโนมัติทันที "ฟรี ๆ"
impl<T: Display> Loud for T {
    fn shout(&self) -> String {
        format!("{}!!!", self.to_string().to_uppercase())
    }
}

struct Money {
    baht: f64,
}

impl Display for Money {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{:.2} baht", self.baht)
    }
}

fn main() {
    // i32 implement Display อยู่แล้วใน std จึงได้ Loud มาโดยไม่ต้องเขียน impl อะไรเพิ่มเลย
    println!("{}", 42.shout());

    // &str ก็ implement Display เช่นกัน
    println!("{}", "hello".shout());

    // Money ที่เราเพิ่ง implement Display เองข้างบน ก็ได้ Loud มาโดยอัตโนมัติทันทีเช่นกัน
    // นี่คือพลังของ blanket implementation: เขียน impl ครั้งเดียวในระดับ "T: Display ใด ๆ"
    // แต่ครอบคลุมทุก type ที่ implement Display ทั้งที่มีอยู่แล้วและที่จะเพิ่มในอนาคต
    let price = Money { baht: 199.5 };
    println!("{}", price.shout());
}
```

ผลลัพธ์:

```
42!!!
HELLO!!!
199.50 BAHT!!!
```

**ข้อควรระวัง**: blanket implementation เป็นเครื่องมือที่ **ทรงพลังมากแต่ต้องใช้อย่างระมัดระวัง** เพราะมันส่งผลกับ
type จำนวนมหาศาลในครั้งเดียว (ทุก type ที่ผ่าน bound ทั้งที่มีอยู่แล้วในทุก crate และที่จะถูกสร้างในอนาคต) ถ้า
bound กว้างเกินไปหรือ logic ข้างในไม่เหมาะกับทุก type ที่เข้าเงื่อนไข อาจสร้างพฤติกรรมที่ไม่คาดคิดในวงกว้างได้ง่าย
กว่าการ implement ทีละ type ปกติ นอกจากนี้ยังมีข้อจำกัดทาง type system ที่ซับซ้อนกว่านี้อีก (เช่น "coherence" —
กฎที่ป้องกันไม่ให้มี blanket implementation สองตัวที่ทับซ้อนกันจนกำกวม) ซึ่งเกินขอบเขตของบทนี้ — ตอนนี้จำแค่ว่า
**blanket implementation คือกลไกเบื้องหลัง `ToString`, และอีกหลาย trait ใน std ที่คุณ "ได้มาฟรี ๆ"** โดยไม่ต้อง
implement เองทีละ type ก็เพียงพอแล้ว เพราะการใช้งานจริงส่วนใหญ่ในโค้ดระดับกลาง-สูงคือการ**ใช้ประโยชน์**จาก blanket
implementation ที่ std มีให้ มากกว่าการ**เขียน blanket implementation ของตัวเอง**

### 19.12 `#[derive(...)]` อย่างเป็นทางการ: Derive Macro คือการ Generate Trait Implementation

ตอนนี้เรามีความรู้พอที่จะเข้าใจ `#[derive(Debug)]` (และเพื่อนบ้านที่ Part 9 แนะนำไว้แบบผิวเผิน) ได้อย่างครบถ้วนแล้ว:
**`derive` คือ procedural macro ที่ generate `impl TraitName for YourType { ... }` ให้อัตโนมัติตอน compile time**
มันไม่ใช่เวทมนตร์ — มันคือการเขียน `impl` block แบบเดียวกับที่เราเขียนมือมาทั้งบทนี้ เพียงแค่ compiler เขียนให้เรา
ตามสูตรมาตรฐาน (field-by-field) แทนที่เราต้องเขียนซ้ำ ๆ ด้วยมือเอง

มาดู derive macro ที่ใช้บ่อยที่สุด 8 ตัวพร้อมกันในตัวอย่างเดียว:

```rust
use std::collections::HashMap;

// รวม derive macro ที่ใช้บ่อยที่สุดไว้ในตัวอย่างเดียว — ProductId เป็น newtype wrapper รอบ u32
// (ทวนแนวคิด newtype จาก Part 9) ที่เบามาก เหมาะจะสาธิตทุก derive พร้อมกันโดยไม่มี field ที่เป็นปัญหา
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash, Default)]
struct ProductId(u32);

fn main() {
    // --- Debug ---
    let id = ProductId(42);
    println!("Debug: {id:?}");

    // --- Clone + Copy ---
    // เพราะ ProductId derive ทั้ง Clone และ Copy การ assign ให้ตัวแปรใหม่คือการ "copy" (ทวนจาก Part 6)
    // ไม่ใช่การ move — id ตัวเดิมยังใช้ต่อได้ปกติหลังบรรทัดนี้ ต่างจาก String ที่ไม่ implement Copy
    let id_copy = id;
    println!("ต้นฉบับยังใช้ได้: {id:?}, สำเนา: {id_copy:?}");

    // --- PartialEq / Eq ---
    let other_id = ProductId(42);
    println!("id == other_id: {}", id == other_id);
    println!("id == id_copy: {}", id == id_copy);

    // --- PartialOrd / Ord ---
    let mut ids = vec![ProductId(30), ProductId(10), ProductId(20)];
    ids.sort(); // ต้องการ Ord ถึงจะเรียก .sort() ได้ตรง ๆ แบบนี้
    println!("เรียงลำดับแล้ว: {ids:?}");
    println!("30 > 10: {}", ProductId(30) > ProductId(10));

    // --- Hash + Eq: ใช้เป็น key ของ HashMap ได้ (ทวนความรู้จาก Part 15) ---
    let mut stock: HashMap<ProductId, u32> = HashMap::new();
    stock.insert(ProductId(1), 50);
    stock.insert(ProductId(2), 30);
    println!("สต็อกของ ProductId(1): {:?}", stock.get(&ProductId(1)));

    // --- Default ---
    let blank_id: ProductId = Default::default();
    println!("ค่า default ของ ProductId: {blank_id:?}");
    // เทียบเท่ากับ ProductId::default() — derive(Default) สร้าง associated function default() ให้เอง
    let blank_id2 = ProductId::default();
    println!("เท่ากับ default อีกตัวไหม: {}", blank_id == blank_id2);
}
```

ผลลัพธ์:

```
Debug: ProductId(42)
ต้นฉบับยังใช้ได้: ProductId(42), สำเนา: ProductId(42)
id == other_id: true
id == id_copy: true
เรียงลำดับแล้ว: [ProductId(10), ProductId(20), ProductId(30)]
30 > 10: true
สต็อกของ ProductId(1): Some(50)
ค่า default ของ ProductId: ProductId(0)
เท่ากับ default อีกตัวไหม: true
```

มาดูรายละเอียดว่าแต่ละ derive macro generate อะไรให้บ้าง และผลลัพธ์เชิง "พฤติกรรม" ที่ได้คืออะไร:

| Derive | Trait ที่ Generate | ให้ความสามารถอะไร |
|---|---|---|
| `Debug` | `std::fmt::Debug` | ใช้ `{:?}`/`{:#?}` debug print ได้ (Part 9) |
| `Clone` | `std::clone::Clone` | เรียก `.clone()` เพื่อสร้าง deep copy ได้ (field-by-field) |
| `Copy` | `std::marker::Copy` | assign/ส่งผ่านฟังก์ชันเป็นการ **copy** ไม่ใช่ **move** (Part 6) |
| `PartialEq` | `std::cmp::PartialEq` | เทียบ `==`/`!=` ได้ (เทียบทุก field, ไม่บังคับ reflexive เต็มรูปแบบ) |
| `Eq` | `std::cmp::Eq` | บอกว่าการเทียบ `==` มี reflexive เต็มรูปแบบ (`a == a` เป็น `true` เสมอไม่มีข้อยกเว้น) — ต้องมี `PartialEq` อยู่แล้วก่อน |
| `PartialOrd` | `std::cmp::PartialOrd` | เทียบ `<`, `>`, `<=`, `>=` ได้ (ไม่บังคับว่าทุกคู่ค่าต้องเทียบกันได้เสมอ) |
| `Ord` | `std::cmp::Ord` | เรียงลำดับได้แบบสมบูรณ์ (ใช้กับ `.sort()`, `BTreeMap`/`BTreeSet` ได้) — ต้องมี `PartialOrd` + `Eq` อยู่แล้วก่อน |
| `Hash` | `std::hash::Hash` | แปลงค่าเป็น hash number ได้ ใช้เป็น key ของ `HashMap`/`HashSet` ได้เมื่อมี `Eq` ควบคู่ |
| `Default` | `std::default::Default` | มี associated function `::default()` ที่คืนค่า "เริ่มต้น" (field ทุกตัวต้อง implement `Default` ด้วย) |

#### เชื่อมโยงกลับไปยัง Part 6: `Copy` และ `Clone` คือ Trait

ทวนจาก Part 6: เราเรียนว่า `i32`, `bool`, `char`, `f64` และ tuple ของชนิดเหล่านี้ **implement trait `Copy`** ทำให้
assign ให้ตัวแปรใหม่เป็นการ copy ไม่ใช่ move ในขณะที่ `String` **ไม่ implement `Copy`** (แต่ implement `Clone`)
ทำให้ต้องเรียก `.clone()` แบบชัดเจนถ้าต้องการสำเนา — Part 6 ตอนนั้นบอกไว้ว่า "`Copy` คือป้ายกำกับที่บอก compiler
ว่าชนิดข้อมูลนี้ปฏิบัติแบบไหน" **ตอนนี้เรารู้แล้วว่า "ป้ายกำกับ" นั้นคือ trait จริง ๆ** และ `#[derive(Copy)]`
คือการขอให้ compiler generate `impl Copy for YourType {}` ให้ (สังเกตว่า `Copy` เป็น **marker trait** — trait ที่
ไม่มี method ให้ implement เลยแม้แต่ตัวเดียว มันแค่ "ติดป้าย" บอก compiler ว่า type นี้ปฏิบัติตามกฎการ copy ได้
ปลอดภัย — เพราะทุก field ของมัน copy ได้อย่างปลอดภัยเช่นกัน โดยไม่มี resource บน heap ที่ต้องดูแลเป็นพิเศษ)

**ข้อจำกัดสำคัญที่เชื่อมกับ Part 6**: struct ที่มี field เป็น `String` (หรือ type อื่นที่ไม่ implement `Copy`)
**ไม่สามารถ derive `Copy` ได้เลย** เพราะ derive macro จะปฏิเสธทันที ด้วยเหตุผลเดียวกับที่ `String` เองไม่ implement
`Copy` ตั้งแต่ต้น (การ copy หมายถึงมีสอง owner ที่ต้อง drop heap memory เดียวกันพร้อมกัน ซึ่งอันตราย) — นี่คือ
เหตุผลที่ `ProductId(u32)` ในตัวอย่างข้างบน derive `Copy` ได้ (เพราะ `u32` implement `Copy`) แต่ `struct Product { name: String, .. }`
ใน Part 9 จะ derive `Copy` ไม่ได้เด็ดขาด

#### เชื่อมโยงกลับไปยัง Part 15: `Hash` + `Eq` คือเงื่อนไขคีย์ของ `HashMap`

ทวนจาก Part 15: `HashMap<K, V>` ต้องการให้ `K` implement **`Hash` และ `Eq` พร้อมกันเสมอ** และ Part 15 บอกไว้ว่า
รายละเอียดทาง type system จะเข้าใจลึกขึ้นตอนเรียน trait อย่างเป็นทางการ — **ตอนนี้เราเข้าใจแล้วว่า `Hash` และ `Eq`
คือ trait จริง ๆ ที่มี method ของตัวเอง** (`Hash` มี `hash()` ที่แปลงค่าเป็นเลข hash, `Eq` เป็น trait ย่อยที่
"ประกาศเพิ่ม" ว่าการเทียบ `==` ที่ `PartialEq` ให้มามี reflexive เต็มรูปแบบ) และการ `#[derive(Hash, Eq, PartialEq)]`
ก็คือการขอให้ compiler generate `impl` ทั้งสาม trait ให้ตามสูตร field-by-field มาตรฐาน — นี่คือเหตุผลที่ `ProductId`
ในตัวอย่างข้างบนใช้เป็น key ของ `HashMap` ได้ทันทีหลังจาก derive ทั้งสาม trait นี้ (และเหตุผลเดียวกันที่ `f64`
เป็น key ของ `HashMap` ไม่ได้ตามที่ Part 15 อธิบายไว้ — เพราะ `f64` implement แค่ `PartialEq` ไม่ implement `Eq`
เนื่องจาก `NaN == NaN` ให้ผลเป็น `false` ซึ่งละเมิดกฎ reflexive ของ `Eq` โดยตรง)

#### ข้อสังเกตสำคัญ: derive ต้องการให้ทุก Field ผ่านเงื่อนไขเดียวกันด้วย

กฎที่ใช้ร่วมกันทุก derive macro ในตารางข้างบน: **derive จะสำเร็จได้ก็ต่อเมื่อทุก field ของ struct implement trait
เดียวกันนั้นด้วย** — `#[derive(Hash)]` ใช้ไม่ได้ถ้ามี field ใด field หนึ่งไม่ implement `Hash` เลย, `#[derive(Copy)]`
ใช้ไม่ได้ถ้ามี field ที่ไม่ implement `Copy` เลย เป็นต้น นี่คือเหตุผลเดียวกันกับที่ Part 9 อธิบายไว้สำหรับ
`#[derive(Debug)]` — เพียงแค่ตอนนี้เรารู้แล้วว่ากฎนี้ใช้กับ **derive macro ทุกตัวโดยไม่มีข้อยกเว้น** เพราะ derive
macro ทำงานโดย "เรียก method เดียวกันของทุก field ตามลำดับ แล้วประกอบผลลัพธ์เข้าด้วยกัน" เสมอ — ถ้า field ใดขาด
method นั้นไปเลย ก็ไม่มีทางประกอบ implementation ที่สมบูรณ์ได้

### 19.13 Operator Overloading: `std::ops::Add` และปริศนา `String + &str` จาก Part 14

ทวนจาก Part 14: เราเคยเห็น `impl Add<&str> for String` เป็นคำอธิบายว่าทำไม `+` ถึงต่อสตริงได้ พร้อมบอกไว้ตรง ๆ ว่า
"เจาะลึก orphan rule ใน Part 19 ตอนเรียน trait อย่างเป็นทางการ" — ตอนนี้เรารู้แล้วว่า `Add` คือ **trait ธรรมดา**
ตัวหนึ่งจาก `std::ops` ไม่มีอะไรพิเศษกว่า trait อื่นที่เรียนมาทั้งบทนี้เลย และ**คุณก็ implement มันให้ type ของ
ตัวเองได้เหมือนกัน** นี่คือกลไกที่เรียกว่า **operator overloading** (การกำหนดความหมายใหม่ให้ตัวดำเนินการ เช่น `+`,
`-`, `*`, `==` สำหรับ type ของตัวเอง) — Rust ไม่มี syntax พิเศษสำหรับ operator overloading แยกต่างหาก มันคือการ
implement trait ที่กำหนดไว้ล่วงหน้าใน `std::ops` เท่านั้นเอง:

```rust
use std::ops::Add;

// Point: จุดในพิกัด 2 มิติ — ตัวอย่างคลาสสิกของการ overload ตัวดำเนินการ +
#[derive(Debug, Clone, Copy, PartialEq)]
struct Point {
    x: f64,
    y: f64,
}

// impl Add for Point คือการบอก compiler ว่า "เมื่อเจอ point_a + point_b ให้ทำตามนี้"
// std::ops::Add คือ trait เดียวกันเป๊ะกับที่ String ใช้ (Part 14: impl Add<&str> for String)
// เพียงแค่ตอนนั้นเรายังไม่รู้จัก trait อย่างเป็นทางการ ตอนนี้เรารู้แล้วว่า "คุณก็ทำแบบนี้ได้เหมือนกัน"
impl Add for Point {
    // associated type: บอกว่าผลลัพธ์ของการบวกคือ type อะไร (ในที่นี้บวกกันแล้วได้ Point กลับมา)
    type Output = Point;

    fn add(self, other: Point) -> Point {
        Point {
            x: self.x + other.x,
            y: self.y + other.y,
        }
    }
}

// Money: เก็บเงินเป็นสตางค์ (u64) เพื่อเลี่ยงปัญหา float ทวนจาก Part 3/9
#[derive(Debug, Clone, Copy, PartialEq)]
struct Money {
    cents: u64,
}

impl Money {
    fn from_baht(baht: f64) -> Self {
        Self { cents: (baht * 100.0).round() as u64 }
    }

    fn baht(&self) -> f64 {
        self.cents as f64 / 100.0
    }
}

impl Add for Money {
    type Output = Money;

    fn add(self, other: Money) -> Money {
        Money { cents: self.cents + other.cents }
    }
}

fn main() {
    let origin = Point { x: 0.0, y: 0.0 };
    let offset = Point { x: 3.0, y: 4.0 };

    // ตัวดำเนินการ + ทำงานได้เพราะ Point implement std::ops::Add แล้ว
    // เบื้องหลังบรรทัดนี้ compiler แปลงเป็น origin.add(offset) ให้เอง
    let moved = origin + offset;
    println!("จุดใหม่: {moved:?}");

    let price_a = Money::from_baht(299.0);
    let price_b = Money::from_baht(150.5);
    let total = price_a + price_b;
    println!("รวมราคา: {:.2} บาท", total.baht());
}
```

ผลลัพธ์:

```
จุดใหม่: Point { x: 3.0, y: 4.0 }
รวมราคา: 449.50 บาท
```

สังเกตประเด็นสำคัญของ `trait Add`:

- **`type Output = Point;`** — นี่คือ **associated type** (type ที่ประกาศไว้ข้างใน trait implementation แทน
  parameter ธรรมดา) บอกว่า "ผลลัพธ์ของการบวกจะเป็น type อะไร" — trait `Add` ที่แท้จริงในนิยามจาก std หน้าตาประมาณ
  `trait Add<Rhs = Self> { type Output; fn add(self, rhs: Rhs) -> Self::Output; }` (สังเกต `Rhs = Self` — ค่า
  default ของ "ฝั่งขวาของ +" คือ type เดียวกันกับตัวเอง ซึ่งเป็นกรณีปกติที่สุด แต่ Part 14 แสดงให้เห็นว่ากำหนดให้
  ต่างกันได้เช่น `Add<&str> for String` ที่ฝั่งขวาเป็น `&str` ไม่ใช่ `String`)
- **`fn add(self, other: Point) -> Point`** — สังเกตว่า receiver เป็น `self` (consuming, ทวนจาก Part 9) ไม่ใช่
  `&self` — นี่คือดีไซน์เดียวกับที่ Part 14 อธิบายไว้สำหรับ `String + &str`: **ฝั่งซ้ายของ `+` ถูก "บริโภค"
  (consumed) เข้าไปในการดำเนินการเสมอ** เพราะ `add` รับ `self` ไม่ใช่ `&self` — เหตุผลเชิง performance คือทำให้
  บาง type (เช่น `String`) สามารถ "ใช้ buffer เดิม" ต่อได้โดยไม่ต้องจองหน่วยความจำใหม่ทั้งหมด (แต่สำหรับ `Point`/
  `Money` ที่ implement `Copy` ในตัวอย่างนี้ การ "บริโภค" ไม่มีผลกระทบอะไรเป็นพิเศษ เพราะ copy ค่าไปแทนได้ตลอดอยู่แล้ว)
- **`origin + offset` ถูกแปลงเป็น `origin.add(offset)`** โดย compiler โดยอัตโนมัติ — ตัวดำเนินการทุกตัวใน Rust
  (`+`, `-`, `*`, `/`, `==`, `<`, `[]`, และอีกมาก) ทำงานผ่านกลไก trait แบบนี้ทั้งหมดไม่มีข้อยกเว้น ซึ่งหมายความว่า
  **คุณสามารถ overload ตัวดำเนินการอะไรก็ได้ที่ std กำหนด trait ไว้** โดยแค่ `impl` trait ที่ตรงกันให้ type ของตัวเอง
  (`std::ops::Sub` สำหรับ `-`, `std::ops::Mul` สำหรับ `*`, `std::ops::Index` สำหรับ `[]` ฯลฯ)

**ข้อคิดสำคัญที่เชื่อมกับ Part 14**: ปริศนาที่ Part 14 ทิ้งไว้ ("ทำไม `String + &str` ทำงานได้ แต่ `String + String`
ทำไม่ได้ตรง ๆ") ตอนนี้ตอบได้อย่างสมบูรณ์แล้ว: เพราะ std เลือก implement **`impl Add<&str> for String`** เท่านั้น
(ไม่ implement `Add<String> for String`) — เป็นการตัดสินใจทางดีไซน์ที่จงใจของทีม std เพื่อบังคับให้ผู้เขียนโค้ด
คิดเรื่อง ownership ตอนต่อสตริงเสมอ (ฝั่งขวาแค่ "ยืม" ก็พอ ไม่จำเป็นต้องยกความเป็นเจ้าของมาทั้งก้อน) — นี่คือ
การตัดสินใจระดับ **library design** ที่ทำผ่านกลไก trait เดียวกันทุกประการกับที่คุณเพิ่งเขียน `Add for Point` เอง
ไม่มีอะไรพิเศษกว่ากันเลย

### 19.14 ตัวอย่างจริงจัง: ระบบรูปทรงเรขาคณิต (Shape System)

มาถึงจุดที่รวมทุกแนวคิดของบทนี้เข้าด้วยกันในตัวอย่างเดียว — ระบบจัดการรูปทรงเรขาคณิต ที่มีหลายรูปทรง (circle,
rectangle, triangle) ต่างกันโดยสิ้นเชิงในแง่โครงสร้างข้อมูล แต่ทั้งหมดต้องคำนวณ "พื้นที่" ได้ และรายงานตัวเองในรูป
แบบมาตรฐานเดียวกันได้ — สถานการณ์นี้คือภาพสะท้อนของปัญหาที่เราเริ่มบทนี้มา (หัวข้อ 19.1) แบบตรงตัวที่สุด:

```rust
// ตัวอย่างจริงจัง: ระบบรูปทรงเรขาคณิต — รวมทุกแนวคิดของบทนี้เข้าด้วยกัน
trait Shape {
    // abstract method — ทุกรูปทรงต้องรู้วิธีคำนวณพื้นที่ของตัวเอง ไม่มี default ให้ เพราะแต่ละรูปทรง
    // มีสูตรคำนวณต่างกันโดยสิ้นเชิง ไม่มี "ค่าเริ่มต้น" ที่สมเหตุสมผลสำหรับพื้นที่เลย
    fn area(&self) -> f64;

    // ชื่อของรูปทรง — ก็เป็น abstract เช่นกัน เพราะแต่ละรูปทรงมีชื่อของตัวเอง
    fn name(&self) -> &str;

    // default method: สร้างคำอธิบายมาตรฐานจาก area() และ name() ที่ trait รู้แน่ ๆ ว่าทุก type ต้องมี
    // รูปทรงส่วนใหญ่ใช้ default นี้ได้เลยโดยไม่ต้อง override เอง
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
    // ไม่ override describe() — ใช้ default ของ trait ตรง ๆ
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
    // ไม่ override describe() เช่นกัน
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

    // Triangle เลือก override describe() เพื่อโชว์ base/height เพิ่มเติมจาก default
    fn describe(&self) -> String {
        format!(
            "{}: ฐาน {:.2} สูง {:.2} พื้นที่ {:.2} ตารางหน่วย",
            self.name(),
            self.base,
            self.height,
            self.area()
        )
    }
}

// ฟังก์ชันที่รับ "รูปทรงใดก็ได้ที่ implement Shape" หนึ่งตัว — ใช้ impl Trait เพราะมีแค่ parameter เดียว
// และไม่ต้องอ้างชื่อ type ที่อื่นในซิกเนเจอร์เลย (ตรงตามหลักการเลือกใน 19.6)
fn print_shape_report(shape: &impl Shape) {
    println!("{}", shape.describe());
}

// เกริ่นเบา ๆ ก่อนจบ: ถ้าต้องการเก็บรูปทรงหลายชนิดปนกันไว้ใน collection เดียว (เช่น Vec)
// เราไม่สามารถใช้ Vec<impl Shape> หรือ Vec<T: Shape> ได้ เพราะ Vec ต้องมี "หนึ่ง concrete type"
// สำหรับสมาชิกทุกตัว (เหมือนปัญหาที่เจอตอนต้นบทกับ array [product, article] เลย)
// ทางออกคือ Vec<Box<dyn Shape>> — เก็บ "กล่องที่หุ้มรูปทรงอะไรก็ได้ที่ implement Shape" ไว้แทน
// รายละเอียดเชิงลึกเรื่อง dynamic dispatch และ dyn Trait แบบเต็มรูปแบบจะเรียนใน Part 21 — ตอนนี้แค่โชว์ว่า
// "ทำได้" และมันคือทางออกของปัญหาที่เราเจอมาตั้งแต่ 19.1 จริง ๆ
fn total_area(shapes: &[Box<dyn Shape>]) -> f64 {
    let mut sum = 0.0;
    for shape in shapes {
        sum += shape.area();
    }
    sum
}

fn main() {
    let circle = Circle { radius: 3.0 };
    let rectangle = Rectangle { width: 4.0, height: 5.0 };
    let triangle = Triangle { base: 6.0, height: 8.0 };

    print_shape_report(&circle);
    print_shape_report(&rectangle);
    print_shape_report(&triangle);

    // ตอนนี้เราแก้ปัญหา "heterogeneous collection" จาก 19.1 ได้แล้วจริง ๆ ด้วย Box<dyn Shape>
    let shapes: Vec<Box<dyn Shape>> = vec![
        Box::new(Circle { radius: 2.0 }),
        Box::new(Rectangle { width: 3.0, height: 3.0 }),
        Box::new(Triangle { base: 4.0, height: 6.0 }),
    ];

    for shape in &shapes {
        println!("{}", shape.describe());
    }

    println!("พื้นที่รวมทั้งหมด: {:.2} ตารางหน่วย", total_area(&shapes));
}
```

ผลลัพธ์:

```
วงกลม: พื้นที่ 28.27 ตารางหน่วย
สี่เหลี่ยมผืนผ้า: พื้นที่ 20.00 ตารางหน่วย
สามเหลี่ยม: ฐาน 6.00 สูง 8.00 พื้นที่ 24.00 ตารางหน่วย
วงกลม: พื้นที่ 12.57 ตารางหน่วย
สี่เหลี่ยมผืนผ้า: พื้นที่ 9.00 ตารางหน่วย
สามเหลี่ยม: ฐาน 4.00 สูง 6.00 พื้นที่ 12.00 ตารางหน่วย
พื้นที่รวมทั้งหมด: 33.57 ตารางหน่วย
```

ตัวอย่างนี้แสดงให้เห็นทุกแนวคิดหลักของบทนี้ทำงานร่วมกันในโปรแกรมเดียว:

1. **`trait Shape`** ประกาศสัญญาที่ `Circle`, `Rectangle`, `Triangle` (สาม struct ที่ไม่เกี่ยวข้องกันเลยในแง่
   โครงสร้างข้อมูล — คนละ field, คนละสูตรคำนวณ) ต้องตอบสนอง
2. **`area()` และ `name()` เป็น abstract method** — บังคับให้ทุกรูปทรงต้องบอกสูตรคำนวณและชื่อของตัวเอง เพราะไม่มี
   ค่า default ที่สมเหตุสมผลสำหรับสิ่งเหล่านี้
3. **`describe()` เป็น default method ที่พึ่งพา `area()`/`name()`** — `Circle` และ `Rectangle` ใช้ default ตรง ๆ
   ในขณะที่ `Triangle` เลือก override เพื่อแสดงข้อมูลเพิ่มเติม
4. **`&impl Shape` เป็น parameter** — `print_shape_report` เขียนครั้งเดียว ใช้ได้กับทุกรูปทรงที่จะเพิ่มเข้ามาในอนาคต
5. **`Box<dyn Shape>` แก้ปัญหา heterogeneous collection** — `Vec<Box<dyn Shape>>` เก็บสามรูปทรงที่ต่าง concrete
   type กันไว้ในตัวแปรเดียว วน `for` loop เรียก `.describe()`/`.area()` ได้ตามปกติ โดยไม่ต้องรู้ล่วงหน้าว่าสมาชิก
   แต่ละตัวเป็นรูปทรงอะไรกันแน่ — นี่คือคำตอบที่สมบูรณ์ของปัญหาที่ 19.1 เริ่มต้นไว้

**ย้ำอีกครั้งก่อนไปหัวข้อถัดไป**: `Box<dyn Shape>` ในตัวอย่างนี้เป็นแค่ **"รสชาติ" เริ่มต้น** ของแนวคิด **trait
object** และ **dynamic dispatch** เท่านั้น รายละเอียดเชิงลึกทั้งหมด — วิธีทำงานภายในผ่าน **vtable**, กฎ **object
safety** (trait แบบไหนถึงใช้เป็น `dyn Trait` ได้บ้าง ไม่ใช่ทุก trait ทำได้), ต้นทุนด้าน performance ที่แท้จริง,
และรูปแบบการใช้งานขั้นสูงอื่น ๆ ของ trait object — จะเป็นเนื้อหาหลักทั้งหมดของ **Part 21: Traits ขั้นสูง**

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. เรียก method ของ trait ไม่ได้ เพราะ trait ไม่ได้อยู่ใน scope

มือใหม่มักงงว่าทำไม struct ที่ "implement trait แล้วแน่ ๆ" กลับเรียก method ของ trait นั้นไม่ได้ — สาเหตุที่พบบ่อย
ที่สุดคือ **import แค่ struct แต่ลืม import trait**:

```rust
mod shapes {
    pub trait Shape {
        fn area(&self) -> f64;
    }

    pub struct Circle {
        pub radius: f64,
    }

    impl Shape for Circle {
        fn area(&self) -> f64 {
            std::f64::consts::PI * self.radius * self.radius
        }
    }
}

use shapes::Circle; // import struct แต่ "ลืม" import trait Shape มาด้วย

fn main() {
    let circle = Circle { radius: 2.0 };
    println!("{}", circle.area());
}
```

```
error[E0599]: no method named `area` found for struct `Circle` in the current scope
   |
   = help: items from traits can only be used if the trait is in scope
help: trait `Shape` which provides `area` is implemented but not in scope; perhaps you want to import it
   |
 1 + use crate::shapes::Shape;
   |
```

**วิธีแก้**: import trait มาด้วยเสมอ (`use shapes::{Circle, Shape};`) — **นี่คือกฎที่สำคัญมากของ Rust ที่ต่างจาก
หลายภาษา**: การที่ type หนึ่ง implement trait ไว้แล้วไม่ได้แปลว่า method ของ trait นั้นจะ "โผล่มาให้เรียกอัตโนมัติ"
ทุกที่ — **trait ต้องอยู่ใน scope ปัจจุบันก่อน** จึงจะเรียก method ของมันผ่าน `.` ได้ (ไม่ว่า type นั้นจะ implement
trait จริงหรือไม่ก็ตาม) เหตุผลเชิงลึกของกฎนี้คือ**ป้องกันความกำกวมเมื่อมีหลาย trait ที่มี method ชื่อเดียวกัน** —
ถ้า import ทุก trait ที่ implement ไว้ให้อัตโนมัติเสมอ อาจเกิดกรณีที่ type หนึ่งมี method ชื่อ `area()` จากสอง
trait ต่างกันพร้อมกัน แล้ว compiler ไม่รู้ว่าคุณหมายถึงตัวไหน การบังคับให้ต้อง `use` trait อย่างชัดเจนทำให้ผู้เขียน
โค้ดเป็นคนตัดสินใจเองว่า "ต้องการ method จาก trait ไหน" อย่างเจาะจง ไม่ปล่อยให้ compiler ต้องเดา

### 2. ลืม derive trait ที่จำเป็น แล้วตัวดำเนินการหรือ collection ที่ต้องพึ่งมันใช้ไม่ได้

```rust
struct Money {
    cents: u64,
}

fn main() {
    let a = Money { cents: 100 };
    let b = Money { cents: 100 };

    // ลืม derive(PartialEq) ให้ Money — เทียบด้วย == ตรง ๆ ไม่ได้
    if a == b {
        println!("เท่ากัน");
    }
}
```

```
error[E0369]: binary operation `==` cannot be applied to type `Money`
   |
note: an implementation of `PartialEq` might be missing for `Money`
help: consider annotating `Money` with `#[derive(PartialEq)]`
   |
 1 + #[derive(PartialEq)]
   |
```

**วิธีแก้**: เพิ่ม `#[derive(PartialEq)]` เหนือ struct — error นี้มักเกิดกับ derive อื่นด้วยรูปแบบเดียวกัน เช่น
ใช้ struct ที่ไม่ derive `Eq`/`Hash` เป็น key ของ `HashMap` (ทวนจาก Part 15) จะได้ error คล้ายกันที่พูดถึง
`the trait bound Money: Eq is not satisfied` แทน หรือใช้ struct ที่ไม่ derive `Debug` กับ `{:?}` จะได้ error ที่
พูดถึง `Money doesn't implement Debug` แทน — **compiler บอก trait ที่ขาดไปตรง ๆ พร้อมแนะนำ derive ที่ต้องเพิ่ม
เสมอ** ทำให้แก้ปัญหาประเภทนี้ตรงไปตรงมามาก เพียงแค่ต้องอ่าน error message ให้ครบ (โดยเฉพาะบรรทัด `help:`)

### 3. Return `impl Trait` แต่พยายาม return หลาย concrete type ตามเงื่อนไข

```rust
trait Summary {
    fn summarize_author(&self) -> String;
    fn summarize(&self) -> String {
        format!("(อ่านเพิ่มเติมจาก {}...)", self.summarize_author())
    }
}

struct Product { name: String, price: f64, brand: String }
struct Article { headline: String, author: String }

impl Summary for Product {
    fn summarize_author(&self) -> String { self.brand.clone() }
}

impl Summary for Article {
    fn summarize_author(&self) -> String { self.author.clone() }
}

fn make_summary(is_featured: bool) -> impl Summary {
    if is_featured {
        Product { name: String::from("A"), price: 1.0, brand: String::from("B") }
    } else {
        Article { headline: String::from("C"), author: String::from("D") }
    }
}

fn main() {
    println!("{}", make_summary(true).summarize());
}
```

```
error[E0308]: `if` and `else` have incompatible types
   |
   |         expected `Product`, found `Article`
   |
help: you could change the return type to be a boxed trait object
   |
   - fn make_summary(is_featured: bool) -> impl Summary {
   + fn make_summary(is_featured: bool) -> Box<dyn Summary> {
```

**วิธีแก้**: เปลี่ยน return type เป็น `Box<dyn Summary>` แล้วห่อทุก branch ด้วย `Box::new(...)` ตามที่อธิบายละเอียด
ในหัวข้อ 19.8-19.9 — จำไว้ว่า `impl Trait` ใน return position **ไม่ใช่** "trait อะไรก็ได้แบบผสมกันได้" มันคือ
"concrete type หนึ่งตัวที่แน่นอน เพียงแค่ไม่บอกชื่อ" เท่านั้น

### 4. Orphan Rule: implement foreign trait ให้ foreign type ไม่ได้

ทวนจาก Part 14 ที่เกริ่นไว้ว่าจะเจาะลึกใน Part 19 — นี่คือ error จริงเมื่อพยายามฝ่า orphan rule:

```rust
use std::fmt;

// พยายาม implement std::fmt::Display (trait จาก std) ให้กับ Vec<i32> (type จาก std เช่นกัน)
// ทั้ง trait และ type ไม่ได้เป็นของ crate นี้เลยแม้แต่ตัวเดียว — orphan rule ห้ามแบบนี้
impl fmt::Display for Vec<i32> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{:?}", self)
    }
}

fn main() {
    let v: Vec<i32> = vec![1, 2, 3];
    println!("{v}");
}
```

```
error[E0117]: only traits defined in the current crate can be implemented for types defined outside of the crate
  |
  |                       `Vec` is not defined in the current crate
  |
  = note: define and implement a trait or new type instead
```

**ทำไมมีกฎนี้**: **orphan rule** บอกว่า `impl TraitX for TypeY` ทำได้ก็ต่อเมื่อ **`TraitX` หรือ `TypeY` (อย่างน้อย
หนึ่งใน สองอย่าง) ต้องเป็นของ crate ปัจจุบัน** — ถ้าทั้งสองมาจาก crate อื่นพร้อมกัน (เหมือนตัวอย่างนี้ที่ทั้ง
`Display` และ `Vec` มาจาก std) จะถูกปฏิเสธทันที เหตุผลคือ**ป้องกันความกำกวมระดับ ecosystem ทั้งหมด**: ลองนึกภาพว่า
สอง crate ที่ไม่รู้จักกัน (crate A และ crate B) ทั้งคู่ implement `Display for Vec<i32>` ของตัวเอง — ถ้าโปรแกรมหนึ่ง
ใช้ทั้งสอง crate พร้อมกัน compiler จะไม่รู้เลยว่าต้องใช้ implementation ของใคร (coherence ล้มเหลว) orphan rule
ป้องกันสถานการณ์นี้โดยการันตีว่า **มีแค่ crate เดียวเท่านั้นที่มีสิทธิ์ implement คู่ trait-type นั้นได้** เสมอ

**วิธีแก้ที่นิยมที่สุด**: ใช้ **newtype pattern** (ทวนจาก Part 9) — ห่อ type ที่เป็นของ crate อื่นด้วย struct ของ
ตัวเอง แล้ว implement trait ให้ struct ห่อนั้นแทน:

```rust
use std::fmt;

// newtype pattern: ห่อ Vec<i32> ด้วย struct ของเราเอง เพื่อให้ "type" เป็นของ crate เรา
// แม้ข้างในจะเก็บ Vec<i32> ที่เป็นของ std ก็ตาม orphan rule ผ่านได้เพราะ IntList เป็น local type
struct IntList(Vec<i32>);

impl fmt::Display for IntList {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{:?}", self.0)
    }
}

fn main() {
    let v = IntList(vec![1, 2, 3]);
    println!("{v}");
}
```

### 5. Override method ผิด signature — กลายเป็นสร้าง method ใหม่ ไม่ใช่ override

```rust
trait Summary {
    fn summarize_author(&self) -> String;
    fn summarize(&self) -> String {
        format!("(อ่านเพิ่มเติมจาก {}...)", self.summarize_author())
    }
}

struct Article {
    author: String,
}

impl Summary for Article {
    fn summarize_author(&self) -> String {
        self.author.clone()
    }

    // ตั้งใจ "override" summarize() แต่พิมพ์ signature ผิด — เผลอเพิ่ม parameter emoji: bool เข้าไป
    fn summarize(&self, emoji: bool) -> String {
        if emoji {
            format!("📰 โดย {}", self.author)
        } else {
            format!("โดย {}", self.author)
        }
    }
}

fn main() {
    let article = Article { author: String::from("กองบรรณาธิการ") };
    println!("{}", article.summarize(true));
}
```

```
error[E0050]: method `summarize` has 2 parameters but the declaration in trait `Summary::summarize` has 1
   |
   |     fn summarize(&self) -> String {
   |                  ----- trait requires 1 parameter
...
   |     fn summarize(&self, emoji: bool) -> String {
   |                  ^^^^^^^^^^^^^^^^^^ expected 1 parameter, found 2
```

**วิธีแก้**: ทำ signature ให้ตรงกับที่ trait ประกาศไว้เป๊ะ ๆ (`fn summarize(&self) -> String`) — ถ้าต้องการ
parameter เพิ่มจริง ๆ ต้องแก้ที่ตัวนิยาม trait ให้ทุก type ที่ implement ต้องรับ parameter นั้นเหมือนกันหมด
**ข้อสังเกตที่สำคัญกว่า error นี้อีก**: ถ้า Rust ไม่เข้มงวดเรื่อง signature ให้ตรงกันเป๊ะแบบนี้ การพิมพ์ผิดเล็ก ๆ
(เช่นลืม `&` หน้า `self`, ใส่ parameter เกิน, หรือ return type ไม่ตรง) จะทำให้เกิด method ใหม่ที่ **ดูเหมือน**
override แต่จริง ๆ ไม่ใช่เลย — โปรแกรมจะยัง compile ผ่านได้ (ถ้า trait ไม่บังคับ) แต่ default method ของ trait
จะยังถูกเรียกใช้อยู่เสมอโดยไม่มีใครรู้ตัว เป็นบั๊กเชิง logic ที่ตรวจจับยากมากเพราะไม่มี error ใด ๆ เลย — Rust
เลือกป้องกันความเสี่ยงนี้โดยบังคับให้ signature ต้องตรงกันแบบเข้มงวด (strict) จนถึงขั้นปฏิเสธไม่ให้ compile ผ่าน
ถ้าไม่ตรงกันแม้แต่นิดเดียว ดีกว่าปล่อยให้เกิดบั๊กเชิง logic ที่ตรวจจับไม่ได้ตอน compile time

## แบบฝึกหัด (Exercises)

1. **โจทย์ระดับง่าย**: เพิ่ม struct ใหม่ชื่อ `Video` (มี field เช่น `title: String`, `channel: String`,
   `duration_seconds: u32`) แล้ว `impl Summary for Video` โดยใช้ trait `Summary` เวอร์ชันจากหัวข้อ 19.5
   (`summarize_author` เป็น abstract, `summarize` เป็น default) ลองทำสองเวอร์ชัน: เวอร์ชันแรกให้ `Video` implement
   แค่ `summarize_author` แล้วใช้ default `summarize()` ไปเลย ดูว่าข้อความที่ได้เหมาะสมไหม จากนั้นเวอร์ชันที่สอง
   ให้ override `summarize()` เองเพื่อโชว์ `duration_seconds` ด้วย (Hint: `summarize_author` ควร return
   `self.channel.clone()`)

2. **โจทย์ระดับกลาง**: เขียนฟังก์ชัน generic ชื่อ `feature_of_the_day<T>(item: &T) -> String` ที่รับ item ใดก็ได้
   ที่ implement ทั้ง `Summary` และ `std::fmt::Debug` พร้อมกัน (ใช้ `where` clause เพื่อความอ่านง่าย) แล้วคืนค่า
   string ที่รวมทั้ง `.summarize()` และ `{:?}` ของ item นั้นเข้าด้วยกัน จากนั้นทดลองว่าถ้าเปลี่ยนจาก `where` clause
   เป็นการเขียน bound แบบ inline (`<T: Summary + std::fmt::Debug>`) ผลลัพธ์การ compile จะเหมือนกันหรือไม่
   (Hint: ควรเหมือนกันทุกประการ เพราะทั้งสองรูปแบบมีความหมายเดียวกัน)

3. **โจทย์ระดับยาก**: เขียน `enum Currency { Baht, UsDollar }` แล้ว implement `std::ops::Add` ให้กับ struct
   `Money { amount_cents: u64, currency: Currency }` โดยที่ `add` ต้อง **panic** (ใช้ `panic!` ทวนจาก Part 3-4)
   ถ้าพยายามบวก `Money` ที่ currency ต่างกัน (เพราะบวกเงินคนละสกุลตรง ๆ ไม่มีความหมายทางธุรกิจ) ทดลองว่าจะต้อง
   derive อะไรเพิ่มให้ `Currency` บ้างเพื่อเทียบ `self.currency == other.currency` ได้ในฟังก์ชัน `add` (Hint:
   `Currency` ต้อง derive อย่างน้อย `PartialEq`, และควร derive `Debug`/`Clone`/`Copy` ไปด้วยเพื่อความสะดวกในการ
   ใช้งานทั่วไป)

4. **โจทย์ระดับยาก/ประยุกต์ใช้งานจริง**: ขยายระบบ `Shape` จากหัวข้อ 19.14 โดยเพิ่ม `struct Square { side: f64 }`
   ที่ implement `Shape` เอง จากนั้นเขียนฟังก์ชัน `fn largest_shape(shapes: &[Box<dyn Shape>]) -> &Box<dyn Shape>`
   ที่คืนค่า reference ไปยังรูปทรงที่มีพื้นที่มากที่สุดใน slice (ใช้ `.iter()` และ `.max_by()`
   หรือวน loop ธรรมดาเทียบ `.area()` ก็ได้ ทวนแนวคิด iterator เบื้องต้นที่จะเรียนเต็มใน Part 25) แล้วทดสอบกับ
   `Vec<Box<dyn Shape>>` ที่มีทั้ง `Circle`, `Rectangle`, `Triangle`, และ `Square` ปนกัน (Hint: เพราะ `Box<dyn Shape>`
   ไม่ implement `PartialOrd`/`Ord` ให้อัตโนมัติ — ต้องเทียบด้วย `.area()` ของแต่ละตัวตรง ๆ ไม่ใช่เทียบ `Box`
   ทั้งก้อนด้วย `<`/`>` โดยตรง)

## สรุป

บทนี้เจาะลึก **trait** ในฐานะกลไกหลักของ Rust สำหรับกำหนด "พฤติกรรมร่วม" ให้กับ type ที่ไม่มีความสัมพันธ์กันเลยใน
เชิงโครงสร้างข้อมูล — เราเริ่มจากปัญหาจริง (สาม struct ที่ต้อง `summarize()` แต่เขียนฟังก์ชันรวมกันไม่ได้) แล้วแก้ด้วย
`trait Summary { fn summarize(&self) -> String; }` ที่เป็น **สัญญา** ไม่ใช่การ implement จริง จากนั้นขยายไปสู่
**default method** (ใช้ตามเดิมได้หรือ override เองก็ได้ รวมถึง pattern คลาสสิกที่ default method เรียกใช้ abstract
method อื่นที่ยังไม่มี body) เรียนรู้สองรูปแบบของ trait bound ที่ความหมายเหมือนกัน (`&impl Trait` และ `<T: Trait>`)
พร้อมรู้ว่าเมื่อไหร่ต้องเลือกแบบไหน ผสมกับ **multiple bounds** (`T: A + B`) และ **`where` clause** สำหรับ bound
ที่ซับซ้อน เราเห็นข้อจำกัดสำคัญของการ return `impl Trait` (ต้อง return concrete type เดียวเสมอ) พร้อม compile
error จริง และรู้จัก `Box<dyn Trait>` เป็นทางออกเบื้องต้น (รอเจาะลึกเต็มใน Part 21) นอกจากนี้ยังเรียน **conditional
method implementation** และ **blanket implementation** ในฐานะรูปแบบขั้นสูงของ `impl` block ปิดท้ายด้วยการอธิบาย
`#[derive(...)]` อย่างเป็นทางการว่าคือการ generate trait implementation ให้อัตโนมัติ (เชื่อมกลับไปยัง `Copy`/`Clone`
จาก Part 6 และ `Hash`/`Eq` จาก Part 15) และการ overload ตัวดำเนินการผ่าน `std::ops::Add` (เชื่อมกลับไปยังปริศนา
`String + &str` จาก Part 14)

ตอนนี้คุณมีความเข้าใจ trait ที่ลึกพอจะอ่านโค้ด Rust ระดับกลาง-สูงส่วนใหญ่ได้อย่างมั่นใจแล้ว เพราะ trait คือกลไกที่
แทรกซึมอยู่ทุกที่ในภาษา Rust — ตั้งแต่ตัวดำเนินการพื้นฐาน, `derive` macro, ไปจนถึง generic bound ที่ควบคุมว่า type
ไหน "ทำอะไรได้บ้าง" อย่างไรก็ตาม เรายังทิ้งประเด็นสำคัญไว้ 2 เรื่องที่แค่ "แง้มประตู" ให้เห็นในบทนี้: **trait object
(`dyn Trait`) และ dynamic dispatch** ที่ยังไม่ได้อธิบายกลไกภายในอย่างละเอียด (vtable, object safety) และก่อนจะไปถึง
จุดนั้น เรายังมีอีกหนึ่งเรื่องพื้นฐานที่ trait ทุกตัวใน struct ที่เก็บ reference ต้องเจอ — **lifetime** ซึ่ง Part 9
เกริ่นไว้สั้น ๆ ตอนพูดถึง `struct Product<'a> { name: &'a str, ... }` และบอกว่าจะเรียนเต็มใน Part 20 — **Part
ถัดไปนี้แหละคือจุดนั้น**

---

**Part ก่อนหน้า:** [Generics เบื้องต้น](part-018-generics-basics.md) | **Part ถัดไป:** [Lifetimes เบื้องต้น](part-020-lifetimes-basics.md)
