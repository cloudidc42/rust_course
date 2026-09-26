# Part 36: Macros: Declarative Macros (macro_rules!)

> โมดูล: ระดับกลาง (Intermediate) | ระดับ: กลาง | เวลาโดยประมาณ: 200 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้อย่างถูกต้องว่า macro ใน Rust คืออะไร ทำงานตอนไหน (compile time) และแตกต่างจากฟังก์ชัน (runtime) อย่างไร — พร้อมตอบคำถามที่ค้างมาตั้งแต่ Part 1 ว่าทำไม `println!` ต้องมี `!`
- แยกความแตกต่างระหว่าง declarative macro (`macro_rules!`) กับ procedural macro (`#[derive(...)]`, attribute macro, function-like proc macro) ได้
- เขียน `macro_rules!` macro ตั้งแต่แบบง่ายที่สุด (ไม่มี argument) ไปจนถึงแบบที่มี pattern หลาย arm และมี repetition
- ใช้ fragment specifier ที่พบบ่อยที่สุดได้อย่างถูกต้อง: `expr`, `ident`, `ty`, `pat`, `block`, `stmt`, `tt`
- เขียน macro ที่รับจำนวน argument ไม่จำกัดด้วย repetition syntax (`$(...),*` และ `$(...),+`) และสร้าง `my_vec!` และ `hashmap!` ของตัวเองได้
- อธิบายและพิสูจน์คุณสมบัติ "hygiene" ของ macro ใน Rust ได้ พร้อมเข้าใจว่าทำไมมันสำคัญเมื่อเทียบกับ macro ของภาษา C
- ใช้ `#[macro_export]` และ `$crate` เพื่อทำให้ macro ใช้งานข้าม module/crate ได้อย่างถูกต้อง
- ตัดสินใจได้ว่าเมื่อไหร่ควรใช้ macro และเมื่อไหร่ควรใช้ generic/trait/function แทน (และรู้ว่าทำไม macro ควรเป็นตัวเลือกสุดท้าย ไม่ใช่ตัวเลือกแรก)

## ความรู้ที่ต้องมีมาก่อน

- **Part 1** — จุดเริ่มต้นของบทนี้จริง ๆ คือประโยคที่เราทิ้งไว้ตั้งแต่บทแรก: `println!("Hello, World!")` มีเครื่องหมาย `!` ต่อท้ายเพราะมันไม่ใช่ฟังก์ชันธรรมดา แต่เป็น **macro** และบทนั้นบอกไว้ว่า "เราจะเรียนเรื่อง macro ละเอียดใน Part 36" — นี่คือ Part 36 ที่ว่า เราจะเฉลยคำถามนี้อย่างละเอียดในหัวข้อแรก
- **Part 4** — เรื่อง function signature ตายตัว (fixed arity), การไม่มี default parameter และ function overloading ใน Rust ซึ่งเป็นเหตุผลสำคัญว่าทำไมฟังก์ชันธรรมดาทำสิ่งที่ `println!`/`vec!` ทำไม่ได้
- **Part 10** — เรื่อง pattern matching และ `match` ที่มีหลาย arm ซึ่งเป็น mental model เดียวกับที่ `macro_rules!` ใช้จับคู่ pattern ของ token ที่ส่งเข้ามา
- **Part 15** — เรื่อง `HashMap` ซึ่งเราจะใช้เป็นตัวอย่างหลักตอนสร้าง macro `hashmap!` ของเราเอง
- **Part 16** — เรื่อง module และ privacy system (`mod`, `pub`, `use`) ซึ่งเกี่ยวข้องโดยตรงกับการทำความเข้าใจว่าทำไม macro ถึงมีกฎ export ที่ต่างจาก item ปกติ
- **Part 18** — เรื่อง generics เบื้องต้น ซึ่งเราจะใช้เปรียบเทียบว่า generics แก้ปัญหา code ซ้ำซ้อนแบบไหนได้ และ macro ต้องเข้ามาแก้ปัญหาแบบไหนที่ generics ทำไม่ได้
- **Part 19/21** — เรื่อง trait พื้นฐานและขั้นสูง ซึ่งเราจะอ้างถึงตอนพูดถึงการใช้ macro ช่วย generate trait implementation ซ้ำ ๆ

## เนื้อหา

### 36.1 เฉลยคำถามที่ค้างมาตั้งแต่ Part 1: ทำไม `println!` ต้องมี `!`

จำได้ไหมว่าใน Part 1 เราเขียนโปรแกรมแรกไว้แบบนี้:

```rust
fn main() {
    println!("Hello, World!");
}
```

และเราบอกไว้สั้น ๆ ว่า `println!` ไม่ใช่ฟังก์ชันธรรมดา แต่เป็น **macro** — ตอนนั้นเราขอให้คุณ "จำไว้ก่อน" เพราะยังไม่มีพื้นฐานพอจะอธิบายลึกกว่านั้น ตอนนี้คุณผ่านมาแล้วถึง generics, traits, closures, iterators — พร้อมแล้วที่จะเข้าใจคำตอบเต็ม ๆ

**คำตอบสั้น ๆ ก่อน แล้วค่อยขยาย:** `println!`, `vec!`, `format!`, `assert_eq!` ที่คุณใช้มาตลอดทั้งหลักสูตรไม่ใช่ฟังก์ชัน (function) แต่เป็น **macro** เครื่องหมาย `!` คือสัญลักษณ์ที่ Rust ใช้บอกทั้ง compiler และมนุษย์ที่อ่านโค้ดว่า "นี่คือการเรียก macro ไม่ใช่การเรียกฟังก์ชัน" ความแตกต่างนี้ไม่ใช่แค่เรื่อง syntax แต่เป็นความแตกต่างเชิง**เวลาทำงาน**ที่ลึกมาก:

| | ฟังก์ชัน (function) | Macro |
|---|---|---|
| ทำงานตอนไหน | **Runtime** — ตอนโปรแกรมกำลังรันจริง | **Compile time** — ตอน compiler กำลังแปลงโค้ดของคุณเป็นโค้ดอื่น ก่อนจะทำการ type-check และ compile จริง |
| รับ "อะไร" เป็น input | รับ **ค่า (value)** ที่มี type แน่นอนตาม signature | รับ **token** (ชิ้นส่วนของ source code เอง) แล้วสร้าง (expand) เป็น source code ชิ้นใหม่ |
| จำนวน argument | ตายตัว (fixed arity) ตาม signature — Part 4 บอกไว้ว่า Rust ไม่มี default parameter และไม่มี overloading | ไม่ตายตัว — รับ token ได้หลากหลายรูปแบบตาม pattern ที่ผู้เขียน macro กำหนด รวมถึงจำนวนไม่จำกัดได้ |
| ตรวจสอบ type ตอนไหน | ตรวจตอน compile แต่ signature คงที่เสมอ | โค้ดที่ macro **expand ออกมา** ถูกตรวจ type ตามปกติ แต่ตัว macro เองจับคู่กับ "รูปแบบของ token" ไม่ใช่ type |

ทีนี้มาดูว่าความแตกต่างนี้อธิบายอะไรได้บ้าง

**ประเด็นที่ 1: ทำไม `println!` รับ argument ได้หลายจำนวน หลาย type พร้อมกัน**

ลองดูตัวอย่างการเรียก `println!` แบบต่าง ๆ:

```rust
fn main() {
    println!("สวัสดี");
    println!("ตัวเลข: {}", 42);
    println!("{} และ {} และ {}", "หนึ่ง", 2, 3.0);
    println!("{name} อายุ {age} ปี", name = "สมชาย", age = 30);
}
```

สังเกตว่าเราเรียก `println!` ด้วย 0, 1, 3, และ 2 argument (ไม่นับ format string) และ argument เหล่านั้นมี type ต่างกันโดยสิ้นเชิง: `&str`, `i32`, `f64` ผสมกัน ถ้า `println!` เป็น**ฟังก์ชันธรรมดา** สิ่งนี้จะเป็นไปไม่ได้เลย เพราะ Part 4 สอนเราไว้ว่า:

1. **ฟังก์ชันใน Rust มี arity ตายตัว** — ถ้าคุณประกาศ `fn foo(a: i32, b: i32)` คุณต้องเรียกด้วย argument 2 ตัวเสมอ จะเรียกด้วย 1 หรือ 3 ตัวไม่ได้
2. **Rust ไม่มี function overloading** — คุณประกาศ `fn foo(a: i32)` และ `fn foo(a: i32, b: i32)` ชื่อเดียวกันพร้อมกันไม่ได้ (`error[E0428]: the name foo is defined multiple times`)
3. **Rust ไม่มี variadic function** (ฟังก์ชันที่รับจำนวน argument ไม่จำกัดแบบ `printf` ใน C หรือ `*args` ใน Python) สำหรับ safe Rust ทั่วไป — สาเหตุคือ Rust ต้องรู้ type และขนาดของทุกอย่างบน stack ตอน compile time อย่างแน่นอน การมี variadic function ที่ type ปนกันแบบ C's `printf(const char*, ...)` จะทำให้ตรวจสอบ type ไม่ได้เต็มร้อย (นี่คือสาเหตุที่ `printf` ใน C ถ้าใส่ `%d` แต่ส่ง `string` เข้าไปจะ crash ตอน runtime หรือมี undefined behavior — Rust ไม่ยอมให้เกิดสถานการณ์แบบนี้)

ดังนั้นถ้า `println!` ต้องเป็นฟังก์ชัน มันจะต้องมี signature ตายตัวแบบใดแบบหนึ่ง เช่น `fn println(s: &str)` ซึ่งจะรับได้แค่ string เดียว ไม่มี argument แทรก ไม่มี formatting เลย — ใช้งานไม่ได้จริง

**แต่ macro ไม่ใช่ฟังก์ชัน — macro ทำงานตอน compile time โดยรับ "token" (ชิ้นส่วนของโค้ดที่คุณเขียน) แล้ว "ขยาย" (expand) กลายเป็นโค้ดชิ้นใหม่ ก่อนที่ตัว type checker จะเริ่มทำงานเสียอีก** เมื่อคุณเขียน:

```rust
println!("{} และ {}", "หนึ่ง", 2);
```

สิ่งที่เกิดขึ้นจริงคือ **ก่อน** compiler จะตรวจสอบ type ใด ๆ, macro `println!` จะถูก**ขยาย**เป็นโค้ดที่ซับซ้อนกว่านั้นมาก (แบบง่ายมาก ๆ เพื่อความเข้าใจ ประมาณว่า):

```rust
// นี่คือรูปแบบคร่าว ๆ (simplified) ของสิ่งที่ println! ขยายออกมา
// ไม่ใช่ token-for-token จริง แต่แนวคิดถูกต้อง
{
    use std::io::Write;
    let mut stdout = std::io::stdout();
    let _ = write!(stdout, "{} และ {}\n", "หนึ่ง", 2);
}
```

หลังจาก**ขยายเสร็จแล้ว** compiler จึงเริ่มตรวจสอบ type ของโค้ดที่ขยายออกมา — และเพราะ macro รับ**ชุดของ token** (ที่มีจำนวนและรูปแบบต่างกันได้) ไม่ใช่รับ "ค่าที่มี type ตายตัว" แบบฟังก์ชัน มันจึงสร้างโค้ดที่แตกต่างกันได้ตามจำนวน argument ที่คุณส่งเข้ามา นี่คือกลไกที่ทำให้ macro รับ argument จำนวนไม่จำกัด type ไม่จำกัดได้ — ไม่ใช่ "เวทมนตร์" อะไร แต่เป็นเพราะมันสร้างโค้ดคนละชุดสำหรับแต่ละการเรียกที่มีรูปแบบต่างกัน

**ประเด็นที่ 2: ทำไม format string ที่ผิดถึงเป็น compile error ไม่ใช่ runtime crash**

ลองดูตัวอย่างนี้:

```rust
fn main() {
    println!("{} และ {}", "มีตัวแปรเดียว");
}
```

ถ้าคุณลองคอมไพล์โค้ดนี้ จะได้ error ทันที**ตอน compile time**:

```
error: 2 positional arguments in format string, but there is 1 argument
 --> src/main.rs:2:14
  |
2 |     println!("{} และ {}", "มีตัวแปรเดียว");
  |               ^^   ^^                     -- ตรงนี้มีแค่ 1 argument
```

เปรียบเทียบกับ C ที่ `printf("%s and %s\n", "only one arg")` จะไม่ error ตอน compile (บาง compiler สมัยใหม่มี warning แต่ไม่ error) แต่จะไป**อ่าน memory ที่ไม่ใช่ของตัวเองตอน runtime** (undefined behavior) เพราะ `printf` เป็นฟังก์ชัน C ธรรมดาที่ไม่รู้ล่วงหน้าว่า format string หน้าตาเป็นอย่างไร มันแค่ไล่อ่าน argument ตาม `%s`/`%d` ที่เจอใน string ไปเรื่อย ๆ ตอน**รันจริง** ถ้า argument ไม่ครบ มันจะอ่านขยะจาก stack ต่อไปอย่างมั่นใจ

ใน Rust เพราะ `println!` เป็น macro ที่ทำงาน**ตอน compile time**: compiler เห็น**ทั้ง** format string (`"{} และ {}"`) **และ** รายการ argument ที่ส่งมาพร้อมกันในตอนเดียวกัน ณ ตอนที่ macro กำลังขยายตัว มันจึง**นับจำนวน `{}` ใน string เทียบกับจำนวน argument ได้ทันที** ก่อนที่โปรแกรมจะถูกคอมไพล์เป็น binary เสียอีก — นี่คือเหตุผลที่ typo เรื่อง placeholder เป็น**compile error** ไม่ใช่ runtime crash ซึ่งเป็นตัวอย่างที่ชัดเจนมากของหลักการ "catch bugs at compile time" ที่ Rust ยึดถือมาตั้งแต่ Part 1 (เรื่อง ownership/borrow checker ก็เป็นแนวคิดเดียวกัน คือย้าย runtime bug ไปเป็น compile-time error)

**สรุปสั้น ๆ ของหัวข้อนี้:** macro ทำงานก่อน type-checking โดยแปลง (expand) token ที่คุณเขียนเป็นโค้ด Rust ชิ้นใหม่ตาม "รูปแบบ" ของ token ที่ได้รับ — ไม่ใช่ตาม "ค่า" แบบฟังก์ชัน นี่คือสาเหตุที่มันหลบเลี่ยงข้อจำกัดเรื่อง fixed arity และ no-overloading ของฟังก์ชันได้ ในหัวข้อต่อไปเราจะเรียนวิธีเขียน macro ของเราเองตั้งแต่พื้นฐาน

### 36.2 สองประเภทหลักของ Macro ใน Rust

Rust มี macro system อยู่ 2 ประเภทใหญ่ ๆ ที่ทำงานคนละกลไก:

**1. Declarative macros (macro_rules!)** — คือหัวข้อหลักของบทนี้ เขียนด้วย keyword `macro_rules!` ทำงานโดย **จับคู่ pattern ของ token** ที่ส่งเข้ามา (คล้าย `match` ใน Part 10 แต่จับคู่ token แทนค่า) แล้วขยายเป็นโค้ดตาม template ที่กำหนดไว้ macro ที่คุณใช้มาตลอดหลักสูตรอย่าง `println!`, `vec!`, `format!`, `write!`, `matches!`, `assert_eq!`, `todo!`, `unreachable!` เกือบทั้งหมดเป็น declarative macro (`vec!` แม้จะดูซับซ้อน แต่ implementation จริงในเบื้องหลังก็คือ `macro_rules!` macro)

**2. Procedural macros (proc macro)** — คือ macro ที่เขียนเป็น**ฟังก์ชัน Rust จริง** ที่รับ token stream เข้ามาเป็น input และคืน token stream เป็น output (ทำงานคล้าย compiler plugin) แบ่งเป็น 3 รูปแบบย่อย:

- **Derive macro** — `#[derive(Debug)]`, `#[derive(Clone)]`, `#[derive(Serialize)]` (จาก crate `serde`) ที่คุณอาจเคยเห็นผ่าน ๆ — สร้าง trait implementation อัตโนมัติจาก struct/enum definition
- **Attribute macro** — เช่น `#[tokio::main]`, `#[get("/users")]` (จาก web framework บางตัว) ที่แปลง item ทั้งก้อนตาม attribute ที่ติดไว้
- **Function-like proc macro** — หน้าตาคล้าย declarative macro (เรียกด้วย `!`) เช่น `sqlx::query!("SELECT ...")` แต่เบื้องหลังเป็นฟังก์ชัน Rust ที่ประมวลผล token เอง ไม่ใช่แค่จับคู่ pattern

ความแตกต่างสำคัญ: **declarative macro จับคู่ pattern แล้ว substitute เข้า template** (เหมือน "find and replace" ที่ฉลาดมาก) ในขณะที่ **procedural macro คือโค้ด Rust ที่รันตอน compile time เพื่อสร้าง/แปลง token stream ได้อย่างอิสระเต็มที่** (สามารถ parse, วิเคราะห์, generate โค้ดอะไรก็ได้ตาม logic ที่เขียน) proc macro ทรงพลังกว่ามากแต่ก็เขียนยากกว่ามาก (ต้องพึ่ง crate ภายนอกอย่าง `syn`, `quote`, `proc-macro2` และต้องอยู่ใน crate แยกที่ประกาศ `proc-macro = true`)

ตัวอย่าง derive macro ที่คุณใช้มาตั้งแต่ Part 19/21 โดยอาจไม่ทันสังเกตว่ามันคือ macro ประเภทหนึ่ง:

```rust
#[derive(Debug, Clone, PartialEq)]  // นี่คือ derive macro 3 ตัว ทำงานพร้อมกัน
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p1 = Point { x: 1, y: 2 };
    let p2 = p1.clone();          // มาจาก impl Clone ที่ #[derive(Clone)] generate ให้
    println!("{:?}", p1);          // มาจาก impl Debug ที่ #[derive(Debug)] generate ให้
    println!("{}", p1 == p2);      // มาจาก impl PartialEq ที่ #[derive(PartialEq)] generate ให้
}
```

ผลลัพธ์:

```
Point { x: 1, y: 2 }
true
```

เมื่อ compiler เห็น `#[derive(Debug, Clone, PartialEq)]` มันจะเรียก**ฟังก์ชัน proc macro ของแต่ละ trait** (ซึ่งเป็นโค้ด Rust ที่คอมไพล์และรันแยกต่างหากตอน build เพื่อ**generate** โค้ด) ให้แต่ละตัวสร้าง `impl Debug for Point { ... }`, `impl Clone for Point { ... }`, `impl PartialEq for Point { ... }` ให้อัตโนมัติ ตามโครงสร้าง field ของ `Point` — สังเกตว่านี่คือสิ่งที่คุณใช้มาตลอดหลักสูตรตั้งแต่ต้น (แทบทุกครั้งที่ประกาศ `struct` ใหม่คุณมักติด `#[derive(Debug)]` ไว้ด้วยความเคยชิน) โดยไม่รู้ตัวว่ากำลังใช้ procedural macro อยู่ ต่างจาก `macro_rules!` ที่บทนี้สอน ตรงที่ derive macro **ไม่ได้จับคู่ pattern ของ token** แต่ **parse โครงสร้างทั้ง struct/enum เป็น syntax tree แล้วเขียนโค้ดใหม่ด้วย logic ของโปรแกรมจริง ๆ** — นี่คือเหตุผลที่มันทำสิ่งที่ซับซ้อนกว่า `macro_rules!` ได้มาก (เช่นรู้จำนวนและชื่อของทุก field ใน struct โดยอัตโนมัติ ซึ่ง `macro_rules!` ทำไม่ได้ตรง ๆ เพราะมันเห็นแค่ "token" ไม่ได้ "เข้าใจ" โครงสร้าง type อย่างสมบูรณ์)

บทนี้จะโฟกัสที่ **declarative macro (`macro_rules!`)** เท่านั้น เพราะเป็นพื้นฐานที่ต้องเข้าใจก่อน — เราจะเรียน procedural macro แบบเต็ม ๆ (รวมถึงวิธีเขียน derive macro ของตัวเองด้วย `syn`/`quote`) ใน **Part 44-45** ของโมดูล Advanced

### 36.3 macro_rules! พื้นฐานที่สุด: Macro ไม่มี Argument

เริ่มจาก macro ที่เรียบง่ายที่สุดเท่าที่จะเป็นไปได้: macro ที่ไม่รับ argument เลย และขยายเป็นโค้ดชุดเดียวที่ตายตัวเสมอ

```rust
// ประกาศ macro ชื่อ say_hello
macro_rules! say_hello {
    // arm นี้จับคู่กับ "ไม่มี token เลย" (วงเล็บว่าง)
    () => {
        println!("สวัสดี จาก macro!");
    };
}

fn main() {
    say_hello!();  // เรียกใช้ macro ด้วย !
    say_hello!();  // เรียกได้หลายครั้ง แต่ละครั้ง expand เป็น println! ใหม่
}
```

ผลลัพธ์:

```
สวัสดี จาก macro!
สวัสดี จาก macro!
```

**อธิบายโครงสร้างทีละส่วน:**

- `macro_rules!` คือ keyword พิเศษที่ใช้ประกาศ declarative macro (สังเกตว่า `macro_rules` เองก็มี `!` ต่อท้าย — เพราะมันเองก็เป็น macro built-in ของ compiler ที่ใช้สำหรับนิยาม macro ตัวอื่น)
- `say_hello` คือชื่อของ macro ที่เราตั้ง (ไม่มี `!` ตอนประกาศ แต่ตอน**เรียกใช้**ต้องมี `!` เสมอ)
- ภายใน `{ ... }` คือรายการ **arm** (คล้าย `match` ใน Part 10) แต่ละ arm มีรูปแบบ `(pattern) => { template };` — สังเกต semicolon `;` ปิดท้ายแต่ละ arm (ยกเว้น arm สุดท้ายจะไม่มีก็ได้ แต่การใส่ไว้เสมอเป็นธรรมเนียมที่ดี)
- `()` คือ **pattern ที่ macro นี้จะจับคู่กับ token ที่ผู้เรียกส่งมา** — ในที่นี้คือ "pattern ว่าง" หมายความว่า macro นี้ต้องถูกเรียกโดยไม่มี token อะไรอยู่ภายในวงเล็บเลย ถ้าคุณเรียก `say_hello!(1, 2, 3)` จะ**ไม่ compile** เพราะไม่มี arm ไหนจับคู่กับ pattern แบบนั้น
- `=> { println!(...); }` คือ**สิ่งที่ macro จะขยายออกมาแทนที่การเรียก** เมื่อเราเขียน `say_hello!();` ใน `main()` ตัว compiler จะแทนที่ token `say_hello!()` ด้วย `{ println!("สวัสดี จาก macro!"); }` แบบเป๊ะ ๆ **ก่อน**ที่จะเริ่ม type-check ฟังก์ชัน `main`

ลองดูข้อผิดพลาดที่จะเกิดถ้าเรียกผิด pattern:

```rust
macro_rules! say_hello {
    () => {
        println!("สวัสดี จาก macro!");
    };
}

fn main() {
    say_hello!(42);  // ผิด! pattern () ไม่รับ token ใด ๆ
}
```

Compiler จะฟ้อง:

```
error: no rules expected `42`
 --> src/main.rs:8:16
  |
1 | macro_rules! say_hello {
  | ---------------------- when calling this macro
...
8 |     say_hello!(42);  // ผิด! pattern () ไม่รับ token ใด ๆ
  |                ^^ no rules expected this token in macro call
  |
note: while trying to match end of macro
```

นี่คือหลักฐานว่า macro **จับคู่ token** ไม่ใช่ค่า — compiler กำลังพยายาม "match" token `42` เข้ากับ arm ที่มีอยู่ (มีแค่ arm เดียวคือ `()`) แล้วไม่มีตัวไหนรองรับ token ตัวนี้เลย จึง error ก่อนที่จะไปถึงเรื่อง type อะไรเลย

### 36.4 macro_rules! ที่มี Argument เดียว

ทีนี้มาลองสร้าง macro ที่รับ 1 argument โดยใช้ **metavariable** — ตัวแปรพิเศษของ macro ที่เขียนด้วย `$` นำหน้า

```rust
// macro รับ 1 argument เป็น expression แล้วพิมพ์ค่าพร้อม label
macro_rules! print_labeled {
    ($value:expr) => {
        println!("ค่า: {:?}", $value);
    };
}

fn main() {
    print_labeled!(42);
    print_labeled!("สวัสดี");
    print_labeled!(3.14 + 1.0);       // expression ที่ซับซ้อนกว่า ก็ยังจับคู่ได้
    print_labeled!(vec![1, 2, 3]);    // แม้แต่ macro call อีกตัวก็เป็น expression ได้
}
```

ผลลัพธ์:

```
ค่า: 42
ค่า: "สวัสดี"
ค่า: 4.14
ค่า: [1, 2, 3]
```

**อธิบายส่วนที่เพิ่มมา:**

- `$value:expr` คือ **metavariable ชื่อ `value`** ที่ประกาศว่าต้องจับคู่กับ**expression** เท่านั้น (`:expr` คือ **fragment specifier** ซึ่งจะอธิบายละเอียดในหัวข้อถัดไป) เครื่องหมาย `$` ข้างหน้าคือสิ่งที่บอกว่า "นี่คือ placeholder ของ macro ไม่ใช่ตัวแปรจริงในโค้ด"
- ในส่วน template (`=> { ... }`) เราเรียกใช้ค่าที่จับคู่ได้ด้วย `$value` (มี `$` แต่ไม่มี `:expr` ต่อท้ายแล้ว เพราะตรงนี้คือการ**ใช้**ค่า ไม่ใช่การ**ประกาศ pattern**)
- เมื่อเรียก `print_labeled!(42)` compiler จะจับคู่ `42` เข้ากับ pattern `$value:expr` สำเร็จ (เพราะ `42` เป็น expression ที่ valid) แล้วขยายเป็น `println!("ค่า: {:?}", 42);` แบบตรงตัว

สิ่งสำคัญที่ต้องเข้าใจ: **`$value:expr` ไม่ได้ "รับค่า" แบบพารามิเตอร์ฟังก์ชัน มันจับคู่กับ "โครงสร้างของโค้ดที่เป็น expression" แล้ว copy โครงสร้างนั้นไปแทนที่ `$value` ในทุกที่ที่ปรากฏใน template** เพราะฉะนั้นถ้าคุณใช้ `$value` ซ้ำหลายครั้งใน template แต่ argument ที่ส่งมาเป็น expression ที่มี side effect (เช่น ฟังก์ชันที่ print อะไรบางอย่าง) โค้ดนั้นจะถูก**ประเมินซ้ำหลายครั้ง** — เป็นกับดักที่พบบ่อยซึ่งเราจะพูดถึงในหัวข้อกับดัก

### 36.5 Fragment Specifiers: ตัวจับคู่ประเภทของ Token

fragment specifier คือสิ่งที่บอก `macro_rules!` ว่า metavariable ตัวนั้นควรจับคู่กับ**โครงสร้างไวยากรณ์แบบไหน** ของ Rust ไม่ใช่แค่ "ตัวอักษรอะไรก็ได้" — นี่คือสิ่งที่ทำให้ macro pattern มีความหมายเชิงไวยากรณ์ ไม่ใช่แค่ text replacement แบบดิบ ๆ (ต่างจาก C preprocessor `#define` ที่เป็น text substitution ล้วน ๆ ไม่รู้จักไวยากรณ์เลย)

ตารางสรุป fragment specifier ที่พบบ่อยที่สุด:

| Specifier | จับคู่กับ | ตัวอย่าง token ที่ match ได้ |
|---|---|---|
| `expr` | Expression (นิพจน์ที่ประเมินค่าได้) | `42`, `x + 1`, `foo()`, `vec![1,2]`, `if cond { 1 } else { 2 }` |
| `ident` | Identifier (ชื่อตัวแปร/ฟังก์ชัน/type) | `x`, `my_function`, `Foo`, `_temp` |
| `ty` | Type | `i32`, `String`, `Vec<u8>`, `&str`, `Option<T>` |
| `pat` | Pattern (แบบที่ใช้ใน `match`/`let`) | `Some(x)`, `1..=5`, `(a, b)`, `_` |
| `block` | Block (โค้ดในปีกกา `{ ... }`) | `{ let x = 1; x + 1 }` |
| `stmt` | Statement (คำสั่งเดียว) | `let x = 5;`, `foo();` |
| `tt` | Token tree — **general ที่สุด** จับคู่กับ **token เดี่ยว หรือกลุ่ม token ในวงเล็บคู่ใดคู่หนึ่ง** | อะไรก็ได้แทบทั้งหมด — ใช้เมื่อไม่แน่ใจว่าจะได้ token แบบไหน |
| `literal` | ค่า literal ตรง ๆ | `42`, `"text"`, `3.14`, `true` |
| `path` | Path (เส้นทางไปยัง item) | `std::collections::HashMap`, `crate::foo::Bar` |
| `lifetime` | Lifetime | `'a`, `'static` |
| `vis` | Visibility modifier | `pub`, `pub(crate)`, (หรือค่าว่าง) |

มาดูตัวอย่างที่ชัดเจนของ 3 ตัวที่ใช้บ่อยที่สุด: `expr`, `ident`, `ty`

**ตัวอย่าง `ident` — จับคู่กับชื่อ:**

```rust
// macro ที่สร้างตัวแปรชื่อตามที่ระบุ พร้อมค่าเริ่มต้นเป็น 0
macro_rules! declare_counter {
    ($name:ident) => {
        let mut $name = 0;
    };
}

fn main() {
    declare_counter!(score);   // ขยายเป็น: let mut score = 0;
    score += 10;
    println!("score = {score}");

    declare_counter!(lives);   // ขยายเป็น: let mut lives = 0;
    println!("lives = {lives}");
}
```

ผลลัพธ์:

```
score = 10
lives = 0
```

สังเกตว่า `$name:ident` จับคู่ได้เฉพาะ**ชื่อ** (identifier) เท่านั้น ถ้าคุณเรียก `declare_counter!(1 + 2)` จะ**ไม่ compile** เพราะ `1 + 2` เป็น expression ไม่ใช่ identifier:

```
error: no rules expected `1`
 --> src/main.rs:7:22
  |
7 |     declare_counter!(1 + 2);
  |                      ^ no rules expected this token in macro call
  |
note: while trying to match meta-variable `$name:ident`
```

(compiler ฟ้องตั้งแต่ token แรกคือ `1` เลย เพราะเลข `1` ไม่ใช่ identifier ตั้งแต่ต้น ไม่ต้องรอไปถึง `+`)

**ตัวอย่าง `ty` — จับคู่กับ type:**

```rust
// macro สร้างฟังก์ชันที่คืนค่า default ของ type ที่ระบุ (ใช้ trait Default)
macro_rules! make_default_fn {
    ($fn_name:ident, $type_name:ty) => {
        fn $fn_name() -> $type_name {
            <$type_name>::default()
        }
    };
}

make_default_fn!(default_number, i32);
make_default_fn!(default_text, String);

fn main() {
    println!("default_number() = {}", default_number());
    println!("default_text() = {:?}", default_text());
}
```

ผลลัพธ์:

```
default_number() = 0
default_text() = ""
```

ตรงนี้เราใช้ metavariable 2 ตัวในหนึ่ง pattern (`$fn_name:ident, $type_name:ty` — คั่นด้วย comma literal `,` ซึ่งต้องพิมพ์ตรงตามที่กำหนดตอนเรียก) macro นี้แสดงให้เห็นว่า `ty` metavariable สามารถถูกใช้เป็น **type annotation ในการประกาศฟังก์ชัน** (`-> $type_name`) และใช้เป็นส่วนหนึ่งของ **path expression** (`<$type_name>::default()`) ได้พร้อมกัน — สิ่งนี้เป็นไปไม่ได้เลยถ้าเป็นแค่ text replacement ธรรมดา เพราะ macro system ของ Rust รู้ว่า `$type_name` ต้องเป็น**ไวยากรณ์ของ type** เท่านั้น ทำให้ compile-time error message ในกรณีที่ผิดจะชัดเจนกว่า preprocessor ทั่วไปมาก

**ตัวอย่าง `pat` — จับคู่กับ pattern (เชื่อมกับ Part 10):**

```rust
// macro ช่วยลดโค้ดซ้ำในการเช็คว่า Option ตรงกับ pattern ที่ระบุหรือไม่
macro_rules! is_match {
    // Arm 1: pattern ธรรมดา ไม่มี guard
    ($value:expr, $pattern:pat) => {
        match $value {
            $pattern => true,
            _ => false,
        }
    };
    // Arm 2: pattern พร้อม guard (if condition) — ต้องแยก arm เพราะ
    // fragment specifier `pat` เก็บได้แค่ตัว pattern เท่านั้น ไม่รวม `if ...`
    // ที่ตามมา (guard เป็น token คนละกลุ่ม ต้องจับด้วย $guard:expr แยกต่างหาก)
    ($value:expr, $pattern:pat if $guard:expr) => {
        match $value {
            $pattern if $guard => true,
            _ => false,
        }
    };
}

fn main() {
    let x: Option<i32> = Some(5);
    println!("{}", is_match!(x, Some(_)));            // true
    println!("{}", is_match!(x, None));                // false
    println!("{}", is_match!(x, Some(n) if n > 10));   // false (guard ทำให้ n=5 ไม่ผ่าน)
    println!("{}", is_match!(x, Some(n) if n > 2));     // true (n=5 ผ่าน guard)
}
```

ผลลัพธ์:

```
true
false
false
true
```

macro นี้จริง ๆ แล้วคือแนวคิดเดียวกับ macro `matches!` ที่มีอยู่แล้วใน standard library — ตอนนี้คุณเข้าใจแล้วว่ามันสร้างจาก `match` expression ธรรมดา (Part 10) ที่ macro ช่วยห่อให้กระชับขึ้นเท่านั้นเอง สังเกตว่าเราต้องแยก arm สำหรับกรณีมี guard (`if $guard:expr`) ออกจากกรณีไม่มี guard เพราะ `$pattern:pat` จับคู่ได้แค่ตัว pattern เท่านั้น ไม่ได้รวมส่วน `if ...` ที่ตามมาด้วย — ถ้าคุณลองรวมทั้งสอง arm เป็น arm เดียว compiler จะฟ้อง `no rules expected keyword \`if\`` เพราะมันมองว่า `if` เป็น token ที่เกินมาหลังจากจับคู่ `$pattern:pat` จบไปแล้ว

**ตัวอย่าง `block` และ `stmt` — จับคู่กับ block/statement:**

```rust
// macro จับเวลาการรันของ block โค้ด (คล้าย timing utility)
macro_rules! time_it {
    ($label:expr, $body:block) => {{
        let start = std::time::Instant::now();
        let result = $body;
        let elapsed = start.elapsed();
        println!("[{}] ใช้เวลา {:?}", $label, elapsed);
        result
    }};
}

fn main() {
    let sum = time_it!("คำนวณผลรวม", {
        let mut total = 0u64;
        for i in 1..=1000 {
            total += i;
        }
        total
    });
    println!("ผลรวม = {sum}");
}
```

ผลลัพธ์ (เวลาจะแตกต่างกันไปในแต่ละการรัน):

```
[คำนวณผลรวม] ใช้เวลา 2.3µs
ผลรวม = 500500
```

สังเกต syntax `=> {{ ... }}` — ปีกกาคู่นอกเป็นส่วนที่บอกว่า template คือ block เดียว (จำเป็นเมื่อ template มีหลาย statement และต้อง evaluate เป็น expression เดียวได้ — ปีกกาชั้นในคือ block ธรรมดาของ Rust) `$body:block` บังคับว่า argument ตัวที่สองต้องเป็น block (`{ ... }`) เท่านั้น จะส่ง expression เปล่า ๆ แบบ `1 + 2` โดยไม่มีปีกกาไม่ได้

**ตัวอย่าง `tt` — general ที่สุด:**

`tt` (token tree) คือ specifier ที่ครอบคลุมที่สุด มันจับคู่ได้กับ**token เดี่ยว** (เช่น `+`, `foo`, `42`) หรือ**กลุ่มของ token ที่อยู่ในวงเล็บคู่ใดคู่หนึ่งทั้งกลุ่ม** (เช่น `(a, b, c)`, `[1, 2, 3]`, `{ ... }` ทั้งก้อนนับเป็น 1 tree) เราจะเห็นการใช้ `tt` เยอะมากในหัวข้อ repetition และ recursive macro ถัดไป เพราะมันมี "ความหยาบ" (granularity) ที่ยืดหยุ่นที่สุด เหมาะกับตอนที่เราต้องการ "รับมาเก็บไว้เฉย ๆ แล้วส่งต่อ" โดยไม่สนใจว่าเนื้อในคืออะไร

### 36.6 Fragment Specifiers ที่เหลือ: `literal`, `path`, `lifetime`, `vis`

นอกจาก 7 ตัวหลักที่เราเห็นตัวอย่างไปแล้ว ยังมี fragment specifier เฉพาะทางอีก 4 ตัวที่พบเจอบ่อยพอสมควรในโค้ดจริง มาดูตัวอย่างรวมกันในที่เดียว:

```rust
// literal: จับคู่กับค่า literal ตรง ๆ เท่านั้น (แคบกว่า expr — ไม่รับ 1 + 1 หรือ foo())
macro_rules! show_literal {
    ($lit:literal) => {
        println!("literal: {}", $lit);
    };
}

// path: จับคู่กับ path ที่ชี้ไปยัง item (ใช้ตอนอ้างอิงถึง type/module แบบเต็ม)
macro_rules! full_path_name {
    ($p:path) => {
        println!("path: {}", stringify!($p));
    };
}

// lifetime: จับคู่กับ lifetime annotation (เชื่อมกับ Part 20/23 เรื่อง lifetimes)
macro_rules! with_lifetime {
    ($lt:lifetime) => {
        println!("lifetime: {}", stringify!($lt));
    };
}

// vis: จับคู่กับ visibility modifier (pub, pub(crate), หรือ "ไม่มีอะไรเลย" ก็ match ได้)
// มักใช้ตอน macro ต้อง generate struct/field ที่ผู้เรียกกำหนด visibility เองได้
macro_rules! declare_wrapper {
    ($v:vis $name:ident : $ty:ty) => {
        $v struct Wrapper {
            $v $name: $ty,
        }
    };
}

declare_wrapper!(pub value: i32);

fn main() {
    show_literal!(42);
    show_literal!("สวัสดี");
    full_path_name!(std::collections::HashMap);
    with_lifetime!('a);

    let w = Wrapper { value: 10 };
    println!("wrapper.value = {}", w.value);
}
```

ผลลัพธ์:

```
literal: 42
literal: สวัสดี
path: std::collections::HashMap
lifetime: 'a
wrapper.value = 10
```

ข้อควรสังเกตเรื่อง `literal` เทียบกับ `expr`: `$lit:literal` **แคบกว่า** `$x:expr` มาก — มันรับได้แค่ค่า literal ตรง ๆ (`42`, `"text"`, `3.14`, `true`, `'a'`) ไม่รับ expression ที่ต้องคำนวณ เช่น `1 + 1` หรือ `foo()` ถ้าคุณเรียก `show_literal!(1 + 1)` จะ error ทันทีเพราะ `1 + 1` เป็น expression ไม่ใช่ literal เดี่ยว ๆ — นี่คือประโยชน์ของการมี specifier แคบ: มันทำให้ macro ของคุณ**ปฏิเสธ input ที่ไม่เหมาะสมได้ตั้งแต่ตอน pattern matching** โดยไม่ต้องเขียน logic ตรวจสอบเพิ่มเอง (หลักการเดียวกับการเลือก type ที่ตรงที่สุดใน function signature แทนที่จะรับ `&str` แบบกว้าง ๆ แล้วค่อย validate เอง)

`vis` เป็น specifier ที่น่าสนใจเป็นพิเศษเพราะมันคือ specifier เดียวที่ **match ได้แม้ไม่มี token เลย** (ถ้าผู้เรียกไม่ระบุ `pub` อะไรเลย `$v:vis` จะจับคู่กับ "ความว่างเปล่า" ได้พอดี ซึ่งหมายถึง private/default visibility ตาม Part 16) พฤติกรรมนี้คล้ายกับ `?` repetition (ศูนย์หรือหนึ่งครั้ง) ที่เราจะเรียนในหัวขัดถัดไป แต่ built-in มาเป็นคุณสมบัติของ specifier นี้เองโดยไม่ต้องเขียน `$($v:vis)?`

### 36.7 Repetition: สร้าง `vec!` เวอร์ชันของตัวเอง

ปัญหาของทุก macro ที่เราเขียนมาจนถึงตอนนี้คือ: มันรับจำนวน argument ที่ตายตัว (0 หรือ 1 หรือ 2) แต่ macro อย่าง `vec![1, 2, 3, 4, 5]` ต้องรับจำนวน argument **ไม่จำกัด** นี่คือจุดที่ **repetition syntax** เข้ามา

syntax ของ repetition คือ `$( ... ) sep rep` โดย:

- `$( ... )` คือส่วนที่จะถูก "ทำซ้ำ"
- `sep` (separator) คือ token ที่คั่นระหว่างแต่ละรอบ (มักเป็น `,` หรือ `;` หรือไม่มีเลย)
- `rep` คือตัวบอกจำนวนรอบ: `*` (ศูนย์ครั้งขึ้นไป — zero or more), `+` (หนึ่งครั้งขึ้นไป — one or more), หรือ `?` (ศูนย์หรือหนึ่งครั้ง — zero or one, ใช้ได้กับ pattern เท่านั้นไม่มี separator)

มาสร้าง `my_vec!` ทีละขั้นตอน เริ่มจากเวอร์ชันที่รับ argument เดียวก่อน แล้วค่อยขยายเป็น repetition:

**ขั้นที่ 1: เวอร์ชันรับ argument เดียว (ทวนความเข้าใจ)**

```rust
macro_rules! my_vec_v1 {
    ($x:expr) => {
        {
            let mut v = Vec::new();
            v.push($x);
            v
        }
    };
}

fn main() {
    let v = my_vec_v1!(42);
    println!("{v:?}");  // [42]
}
```

ใช้ได้กับ argument เดียวเท่านั้น เรียก `my_vec_v1!(1, 2, 3)` จะ error ทันที (ไม่มี arm ที่ match)

**ขั้นที่ 2: ใส่ repetition เข้าไป — เวอร์ชันเต็มของ `my_vec!`**

```rust
macro_rules! my_vec {
    // Arm 1: เรียกโดยไม่มี argument เลย เช่น my_vec!() -> Vec ว่าง
    () => {
        Vec::new()
    };
    // Arm 2: เรียกด้วย argument หนึ่งตัวขึ้นไป คั่นด้วย comma
    // $( $x:expr ),+ หมายถึง "expression หนึ่งตัวขึ้นไป คั่นด้วย comma"
    // comma ท้ายสุดจะมีหรือไม่มีก็ได้ (trailing comma) เพราะเราใส่ $(,)? ต่อท้าย
    ( $( $x:expr ),+ $(,)? ) => {
        {
            let mut v = Vec::new();
            $(
                v.push($x);
            )+
            v
        }
    };
}

fn main() {
    let empty: Vec<i32> = my_vec!();
    println!("{empty:?}");                       // []

    let one = my_vec!(10);
    println!("{one:?}");                          // [10]

    let many = my_vec!(1, 2, 3, 4, 5);
    println!("{many:?}");                          // [1, 2, 3, 4, 5]

    let trailing = my_vec!(1, 2, 3,);              // มี comma ท้ายสุดก็ยังใช้ได้
    println!("{trailing:?}");                      // [1, 2, 3]

    let strings = my_vec!("a".to_string(), "b".to_string());
    println!("{strings:?}");                        // ["a", "b"]
}
```

ผลลัพธ์:

```
[]
[10]
[1, 2, 3, 4, 5]
[1, 2, 3]
["a", "b"]
```

**อธิบาย repetition ทีละส่วน อย่างละเอียด:**

- `$( $x:expr ),+` — ส่วนนี้อ่านว่า "จับคู่กับ expression หนึ่งตัวหรือมากกว่า โดยแต่ละตัวคั่นด้วย comma" กลไกภายในคือ: compiler จะพยายามจับคู่ `$x:expr` ซ้ำไปเรื่อย ๆ ทีละตัว โดยระหว่างแต่ละรอบต้องเจอ token `,` (separator ที่เราระบุไว้นอก `+`) จนกว่าจะไม่มี expression เหลือให้จับคู่แล้ว
- `+` หมายถึง**ต้องมีอย่างน้อย 1 ตัว** — ถ้าเราเขียนแค่ arm นี้อย่างเดียวแล้วเรียก `my_vec!()` (ไม่มี argument เลย) จะ error เพราะ `+` ไม่ยอมรับศูนย์ตัว (นี่คือสาเหตุที่เราต้องมี **arm แยก** สำหรับกรณีว่างเปล่า — เหมือนกับที่ `match` ใน Part 10 ต้องมี arm ให้ครบทุกกรณี ไม่งั้น non-exhaustive)
- `$(,)?` ต่อท้าย — คือการรองรับ **trailing comma** (comma ตัวสุดท้ายที่ห้อยอยู่หลัง argument ตัวสุดท้าย เช่น `my_vec!(1, 2, 3,)`) `?` หมายถึง "ศูนย์หรือหนึ่งครั้ง" — ถ้าไม่มี syntax นี้ การเรียก `my_vec!(1, 2, 3,)` จะ error เพราะ compiler จะงงว่า comma ตัวสุดท้ายคั่นอะไรกับอะไร (นี่คือ pattern มาตรฐานที่ macro ระดับ production ทุกตัว รวมถึง `vec!` จริงใน standard library ใช้)
- ในส่วน template: `$( v.push($x); )+` — ส่วนนี้คือ**การขยาย repetition ในทิศทางกลับ** เมื่อ compiler จับคู่ pattern ได้ว่ามี expression 5 ตัว (`1, 2, 3, 4, 5`) มันจะ**ทำซ้ำ**ส่วน `v.push($x);` **5 ครั้ง** โดยแทนที่ `$x` ด้วยตัวที่ตรงกันในแต่ละรอบ ผลคือขยายเป็น:

```rust
{
    let mut v = Vec::new();
    v.push(1);
    v.push(2);
    v.push(3);
    v.push(4);
    v.push(5);
    v
}
```

สิ่งสำคัญมากที่ต้องสังเกต: **จำนวนครั้งที่ repeat ใน pattern (ตอนจับคู่) กับจำนวนครั้งที่ repeat ใน template (ตอนขยาย) จะเท่ากันเสมอโดยอัตโนมัติ** — คุณไม่ต้องนับจำนวนเอง ไม่ต้องมี index หรือ loop counter ใด ๆ macro system จะจับคู่ `$x` แต่ละตัวที่ match ได้ในรอบเดียวกันโดยอัตโนมัติ นี่คือแก่นของ metaprogramming แบบ pattern-based

เปรียบเทียบกับสิ่งที่เป็นไปไม่ได้ด้วย generics (Part 18): ต่อให้คุณเขียน generic function ที่เก่งขนาดไหน คุณก็เขียน `fn my_vec<T>(items: ???) -> Vec<T>` ที่รับจำนวน argument ไม่จำกัดไม่ได้ เพราะฟังก์ชันต้องมี arity ตายตัว (Part 4) วิธีเดียวที่ฟังก์ชันจะรับจำนวนไม่จำกัดได้คือรับเป็น slice หรือ array `fn my_vec_alt<T>(items: &[T]) -> Vec<T>` แต่นั่นก็บังคับให้ผู้เรียก**ต้องสร้าง array/slice เองก่อน** ซึ่งย้อนกลับไปที่ปัญหาเดิม (จะสร้าง array แบบไม่จำกัดจำนวนตอน compile time ได้อย่างไร) — นี่คือเหตุผลแท้จริงที่ทีม Rust ต้องมี macro system: เพื่อแก้ปัญหาที่ type system ปกติ (generics + functions) แก้ไม่ได้โดยธรรมชาติ

**ทดสอบว่า macro ทำงานถูกต้องจริง** เราจะเขียน `#[test]` (จาก Part 32) เพื่อยืนยัน:

```rust
macro_rules! my_vec {
    () => {
        Vec::new()
    };
    ( $( $x:expr ),+ $(,)? ) => {
        {
            let mut v = Vec::new();
            $(
                v.push($x);
            )+
            v
        }
    };
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_empty() {
        let v: Vec<i32> = my_vec!();
        assert_eq!(v, Vec::<i32>::new());
    }

    #[test]
    fn test_single() {
        let v = my_vec!(99);
        assert_eq!(v, vec![99]);
    }

    #[test]
    fn test_multiple() {
        let v = my_vec!(1, 2, 3);
        assert_eq!(v, vec![1, 2, 3]);
    }

    #[test]
    fn test_trailing_comma() {
        let v = my_vec!(1, 2, 3,);
        assert_eq!(v, vec![1, 2, 3]);
    }
}

fn main() {}
```

### 36.8 Multiple Match Arms: Macro ที่ทำงานต่างกันตามรูปแบบ Argument

เหมือนที่ `match` (Part 10) มีหลาย arm ให้ครอบคลุมหลายกรณี `macro_rules!` ก็รองรับหลาย arm เช่นกัน โดย compiler จะ**ลองจับคู่ทีละ arm ตามลำดับที่เขียนไว้ จนกว่าจะเจอ arm แรกที่ match ได้** (ไม่ต่างจากลำดับความสำคัญของ `match` arm)

ตัวอย่าง: macro `greet!` ที่พฤติกรรมต่างกันตามจำนวน/ชนิดของ argument ที่ได้รับ

```rust
macro_rules! greet {
    // Arm 1: ไม่มี argument -> ทายทักทั่วไป
    () => {
        println!("สวัสดีครับ/ค่ะ!");
    };
    // Arm 2: มี argument เดียวเป็น expression (ชื่อคนที่จะทัก)
    ($name:expr) => {
        println!("สวัสดี {}!", $name);
    };
    // Arm 3: มี 2 argument คือ ชื่อ และ ช่วงเวลาของวัน (identifier พิเศษ)
    ($name:expr, morning) => {
        println!("อรุณสวัสดิ์ {}! ทานอาหารเช้าหรือยัง", $name);
    };
    ($name:expr, night) => {
        println!("ราตรีสวัสดิ์ {}! หลับฝันดีนะ", $name);
    };
}

fn main() {
    greet!();                    // Arm 1
    greet!("สมชาย");              // Arm 2
    greet!("สมหญิง", morning);    // Arm 3
    greet!("วิชัย", night);       // Arm 4
}
```

ผลลัพธ์:

```
สวัสดีครับ/ค่ะ!
สวัสดี สมชาย!
อรุณสวัสดิ์ สมหญิง! ทานอาหารเช้าหรือยัง
ราตรีสวัสดิ์ วิชัย! หลับฝันดีนะ
```

**ข้อสังเกตสำคัญ:** `morning` และ `night` ใน arm ที่ 3 และ 4 **ไม่ใช่ metavariable** (ไม่มี `$` นำหน้า) มันคือ**token ตายตัว**ที่ pattern คาดหวังว่าจะเจอเป๊ะ ๆ — เหมือนกับตอนที่ `match` จับคู่กับ literal ตายตัว เช่น `match x { 0 => ..., _ => ... }` ที่ arm แรกจับคู่กับเลข 0 เป๊ะ ๆ ไม่ใช่ตัวแปร ในที่นี้ `morning`/`night` ทำหน้าที่คล้าย ๆ "keyword argument" ของ macro นี้เอง

ลองดูว่าถ้าเรียกด้วย argument ที่ไม่มี arm ไหนรองรับ จะเกิดอะไรขึ้น:

```rust
fn main() {
    greet!("สมชาย", afternoon);  // ไม่มี arm ไหนรองรับ "afternoon"
}
```

Error:

```
error: no rules expected `afternoon`
  --> src/main.rs:20:21
   |
1  | macro_rules! greet {
   | ------------------ when calling this macro
...
20 |     greet!("สมชาย", afternoon);
   |                     ^^^^^^^^^ no rules expected this token in macro call
   |
note: while trying to match `morning`
```

Compiler จะลองจับคู่กับทุก arm ตามลำดับ (arm 1 ไม่ match เพราะมี argument, arm 2 ไม่ match เพราะมี 2 argument ไม่ใช่ 1, arm 3 ไม่ match เพราะ token ที่สองไม่ใช่ `morning`, arm 4 ไม่ match เพราะไม่ใช่ `night`) แล้วสรุปว่าไม่มี arm ไหนรองรับเลย จึง error — concept นี้เหมือนกับ non-exhaustive `match` แต่ macro ไม่มี "warning เตือนล่วงหน้า" แบบ `match` มีสำหรับ enum เพราะ macro ไม่ได้ผูกกับ type ปิด (closed set) แบบ enum แต่ผูกกับ "รูปแบบของ token" ซึ่งมีได้ไม่จำกัด — คุณในฐานะผู้เขียน macro ต้องรับผิดชอบเองว่า caller อาจส่ง token ที่ไม่ match arm ไหนเลยได้เสมอ

### 36.9 Hygiene: ทำไม Macro ของ Rust ไม่ทำให้ตัวแปรชนกัน

หนึ่งในปัญหาคลาสสิกของระบบ macro/preprocessor ในภาษาอื่น (โดยเฉพาะ C's `#define`) คือ **variable capture** — ตัวแปรที่ macro สร้างขึ้นภายในไปชนกับตัวแปรของโค้ดที่เรียกใช้ macro โดยไม่ตั้งใจ Rust แก้ปัญหานี้ด้วยแนวคิดที่เรียกว่า **macro hygiene (ความสะอาดของ macro)**

**ตัวอย่างปัญหาใน C ก่อน (เพื่อเห็นภาพว่าทำไมมันน่ากลัว):**

```c
/* C preprocessor macro - เป็นแค่ text substitution ล้วน ๆ ไม่รู้จักไวยากรณ์ */
#define SWAP(a, b) { int tmp = a; a = b; b = tmp; }

int main() {
    int tmp = 5;
    int y = 10;
    SWAP(tmp, y);  /* ขยายเป็น: { int tmp = tmp; tmp = y; y = tmp; } */
    /* บั๊ก! ตัวแปร tmp ของเราชนกับ tmp ที่ macro สร้างขึ้นเอง
       ผลลัพธ์ที่ควรได้คือ tmp=10, y=5 แต่จะได้ผลลัพธ์ผิดเพราะ
       "int tmp = tmp;" ใน scope ใหม่บังตัวแปร tmp เดิม แล้วค่อยเอาค่า
       ที่ยังไม่ initialize มา assign - พฤติกรรมนี้เป็น bug ที่หาสาเหตุยากมาก
       เพราะโค้ดเรียกดูเหมือนถูกทุกอย่าง จนกว่าจะไป "ขยาย" macro ดูจริง ๆ */
    printf("%d %d\n", tmp, y);
    return 0;
}
```

ปัญหานี้เกิดเพราะ C's `#define` เป็นแค่ **text substitution แบบดิบ** — preprocessor ไม่รู้จักไวยากรณ์เลย มันแค่ copy-paste ตัวอักษรแทนที่กันตรง ๆ ตัวแปร `tmp` ที่ macro ประกาศขึ้นภายในกับ `tmp` ที่ผู้ใช้ส่งเข้ามาเป็น argument จึงเป็น "ชื่อเดียวกัน" ในสายตาของ compiler โดยสมบูรณ์ ทำให้ชนกัน

**ทีนี้มาดูว่า Rust ป้องกันปัญหานี้ได้อย่างไร:**

```rust
// macro ที่สร้างตัวแปรชื่อ x ไว้ใช้ภายในตัวเอง โดยตั้งใจใช้ชื่อ "x" ซึ่งเดายากว่าจะชนกับอะไร
macro_rules! double_it {
    ($val:expr) => {
        {
            let x = $val;   // ตัวแปร x นี้เป็นของ macro เอง
            x * 2
        }
    };
}

fn main() {
    let x = 100;               // ตัวแปร x ของโค้ดผู้ใช้ (คนละตัวกับ x ใน macro)
    let result = double_it!(5);
    println!("x ของเรายังเป็น {x}");        // x ของเรา (100) ไม่ถูกแก้ไข!
    println!("result = {result}");           // result = 10 (5 * 2) ถูกต้อง
}
```

ผลลัพธ์:

```
x ของเรายังเป็น 100
result = 10
```

**สิ่งที่เกิดขึ้นจริง (ทำไมไม่ชนกัน):** เมื่อ `macro_rules!` macro ขยายตัว ตัวแปรที่**macro เองเป็นคนสร้างขึ้นภายใน template** (ในที่นี้คือ `x` ใน `let x = $val;`) จะถูก compiler ผูกไว้กับ**"hygiene context" ของการขยาย macro ครั้งนั้นโดยเฉพาะ** ซึ่งแยกจาก scope ของโค้ดที่เรียกใช้ macro อย่างสิ้นเชิง แม้ว่าทั้งคู่จะสะกดว่า `x` เหมือนกันเป๊ะ แต่ในมุมมองของ compiler มันคือ**ตัวแปรคนละตัว** (คล้ายกับว่า compiler แอบเปลี่ยนชื่อ `x` ภายใน macro ให้เป็นชื่อพิเศษที่ไม่มีใครสะกดชนได้ แม้ในซอร์สโค้ดที่แสดงให้เราเห็นจะยังเขียนว่า `x` เหมือนเดิม) นี่คือความหมายของคำว่า **"hygienic macro"** — สะอาดในแง่ที่ไม่เปื้อน (ไม่รั่วไหลไปชน) scope ของผู้เรียกใช้

ข้อควรรู้เพิ่มเติม: hygiene ของ `macro_rules!` **ไม่สมบูรณ์แบบร้อยเปอร์เซ็นต์เหมือน proc macro** ในบางกรณี (เช่นตัวแปรที่ macro รับผ่าน metavariable ไม่ใช่ตัวแปรที่ macro สร้างเอง) hygiene จะไม่ครอบคลุม แต่สำหรับ 95% ของการใช้งานทั่วไป (ตัวแปรชั่วคราวที่ macro สร้างขึ้นเองภายใน) hygiene ทำงานได้ดีเยี่ยมและป้องกันบั๊กประเภทนี้ได้อย่างมีประสิทธิภาพ — ต่างจาก C ที่ **ไม่มี hygiene เลย** ต้องพึ่งพาธรรมเนียมของโปรแกรมเมอร์ (เช่นตั้งชื่อตัวแปรภายใน macro ให้แปลก ๆ จนไม่มีใครใช้ชื่อชนโดยไม่ตั้งใจ) ซึ่งเป็นวิธีที่ error-prone มาก

**ข้อสำคัญที่ต้องแยกให้ออก: hygiene ไม่ได้แปลว่า macro "เข้าถึงตัวแปรของผู้เรียกไม่ได้เลย"** — ถ้า macro รับชื่อตัวแปรผ่าน **metavariable** (เช่น `$v:ident`) โดยตั้งใจ มันจะอ้างอิงถึงตัวแปรตัวจริงของผู้เรียกได้ตามปกติ (เพราะ metavariable ไม่ได้ "ถูกสร้างขึ้นภายใน" macro — มันเป็นแค่ placeholder ที่ถูกแทนที่ด้วย token ที่ผู้เรียก**ส่งมาเอง**) hygiene ป้องกันแค่กรณีที่ macro สร้างตัวแปรขึ้นมาใหม่**เองภายใน** template เท่านั้น:

```rust
// macro รับชื่อตัวแปรของผู้เรียกผ่าน $v:ident แล้วแก้ไขตัวแปรนั้นตรง ๆ
macro_rules! increment {
    ($v:ident) => {
        $v += 1;   // $v ถูกแทนที่ด้วยตัวแปรจริงของผู้เรียก ไม่ใช่ตัวแปรใหม่ของ macro
    };
}

fn main() {
    let mut counter = 10;
    increment!(counter);              // ขยายเป็น: counter += 1;
    increment!(counter);
    println!("counter = {counter}");  // 12 — แก้ไขตัวแปรของผู้เรียกได้จริง ตามที่ตั้งใจ
}
```

ผลลัพธ์:

```
counter = 12
```

เปรียบเทียบให้ชัด: ใน `double_it!` ก่อนหน้า ตัวแปร `x` **ถูกสร้างขึ้นเองโดย macro** (`let x = $val;`) จึงถูก hygiene ป้องกันไม่ให้ชนกับ `x` ของผู้เรียก แต่ใน `increment!` ตัวแปรที่ถูกแก้ไข (`$v`) **มาจากผู้เรียกส่งเข้ามาเอง** ผ่าน metavariable จึงไม่ใช่ตัวแปรใหม่ที่ macro สร้าง และ hygiene จึงไม่เข้ามาเกี่ยวข้อง (และไม่ควรเข้ามาเกี่ยวข้องด้วย เพราะจุดประสงค์ของ `increment!` คือต้องการแก้ไขตัวแปรของผู้เรียกโดยตรง) กฎง่าย ๆ ที่จำได้คือ: **hygiene คุ้มครองแค่สิ่งที่ macro สร้างขึ้นเอง ไม่คุ้มครอง (และไม่ควรคุ้มครอง) สิ่งที่ macro รับเข้ามาจากผู้เรียกผ่าน metavariable**

### 36.10 `$crate`: อ้างอิงถึง crate ที่นิยาม Macro

ปัญหาหนึ่งที่เกิดขึ้นเมื่อ macro ของคุณถูกใช้งานจาก**crate อื่น** (ไม่ใช่ crate ที่นิยาม macro) คือ: ถ้า template ของ macro อ้างอิงถึง item อื่นในเดียวกัน (เช่นฟังก์ชัน helper, struct, หรือ item อื่นใน crate ที่นิยาม macro) การอ้างอิงนั้นต้องรู้ว่า "crate ไหน" คือ crate ต้นทาง — ซึ่งเปลี่ยนไปตามมุมมองของ caller `$crate` คือ metavariable พิเศษที่ compiler เติมให้อัตโนมัติ ไม่ต้องประกาศเอง ใช้แทน**path ที่ชี้กลับไปยัง crate ที่นิยาม macro นี้เสมอ ไม่ว่า macro จะถูกเรียกจากที่ไหน**

ตัวอย่าง:

```rust
// สมมติว่านี่คือ lib.rs ของ crate ชื่อ "my_utils"
pub fn helper_function(x: i32) -> i32 {
    x * 10
}

#[macro_export]
macro_rules! process {
    ($x:expr) => {
        // ใช้ $crate เพื่ออ้างอิง helper_function ให้ถูกต้อง
        // ไม่ว่า macro นี้จะถูกเรียกจาก crate ไหนก็ตาม
        $crate::helper_function($x)
    };
}
```

ถ้าเราเขียนแบบไม่ใช้ `$crate`:

```rust
// ผิด! ถ้าเขียนแบบนี้แล้ว export ออกไปให้ crate อื่นใช้
macro_rules! process_bad {
    ($x:expr) => {
        helper_function($x)   // สมมติว่า caller ไม่มีฟังก์ชันชื่อนี้ หรือมีคนละตัว จะ error/ผิดพลาด
    };
}
```

ถ้า crate อื่นเรียก `my_utils::process_bad!(5)` โดยไม่มี `helper_function` อยู่ใน scope ของตัวเอง จะเกิด error ทำนอง `cannot find function helper_function in this scope` เพราะ token `helper_function` ที่ขยายออกมาจะถูก resolve ในบริบทของ**crate ผู้เรียก** ไม่ใช่บริบทของ `my_utils` — การใช้ `$crate::helper_function` แก้ปัญหานี้เพราะ `$crate` ถูก compiler แทนที่ด้วย path ที่ถูกต้องเสมอ ไม่ว่าจะเรียกจากภายใน crate เดียวกัน (กรณีนั้น `$crate` จะกลายเป็น `crate`) หรือจาก crate อื่น (กรณีนั้นจะกลายเป็นชื่อ crate จริง เช่น `my_utils`)

นี่คือเหตุผลที่คุณจะเห็น `$crate` ปรากฏอยู่ทั่วไปใน macro ของ standard library และ crate ที่เป็น library จริงจัง — เป็นแนวปฏิบัติที่ดี (best practice) ที่ควรทำเป็นธรรมเนียมทุกครั้งที่ macro ของคุณอ้างอิงถึง item อื่นในเดียวกัน โดยเฉพาะถ้าตั้งใจจะ `#[macro_export]` ออกไปให้ที่อื่นใช้

### 36.11 การ Export Macro: `#[macro_export]` และความสัมพันธ์กับ Module System (Part 16)

จาก Part 16 เราเรียนรู้ว่า item ปกติ (function, struct, enum, ฯลฯ) ควบคุมการมองเห็น (visibility) ด้วย `pub`, `pub(crate)`, `pub(super)` และการ import ด้วย `use` — ตาม module tree ที่เป็นลำดับชั้น **แต่ `macro_rules!` macro มีกฎที่ต่างออกไปในเชิงประวัติศาสตร์** ซึ่งเป็นเรื่องที่ควรรู้ตรง ๆ เพื่อไม่ให้สับสน

**กฎพื้นฐาน (แบบไม่มี attribute):** โดย default `macro_rules!` macro จะมองเห็นได้เฉพาะ**ที่ประกาศและหลังจากจุดประกาศลงมาใน source code เดียวกัน** (textual scope) ตาม**ลำดับที่เขียนในไฟล์** ไม่ใช่ตาม module tree แบบ item อื่น ๆ

```rust
mod math_utils {
    // ประกาศ macro ในนี้
    macro_rules! double {
        ($x:expr) => { $x * 2 };
    }

    pub fn use_it() -> i32 {
        double!(21)  // ใช้ได้ เพราะอยู่ใน module เดียวกัน หลังจุดประกาศ
    }
}

fn main() {
    // double!(5) ตรงนี้จะ error เพราะ macro ไม่ได้ถูก pub ออกมา
    // และไม่ได้อยู่ใน textual scope เดียวกัน
    println!("{}", math_utils::use_it());
}
```

**การทำให้ macro ใช้งานได้ข้าม module/crate มี 2 วิธีหลัก:**

**วิธีที่ 1: `#[macro_export]` (แบบเก่า แต่ยังใช้กันแพร่หลาย)**

```rust
// ประกาศที่ root ของ crate (หรือที่ไหนก็ได้ แต่ #[macro_export] จะดึงมันไปที่ crate root เสมอ)
#[macro_export]
macro_rules! triple {
    ($x:expr) => {
        $x * 3
    };
}

fn main() {
    let result = triple!(7);
    println!("{result}");  // 21
}
```

**ข้อควรระวังเชิงประวัติศาสตร์ (quirk ที่ควรรู้ตรง ๆ):** `#[macro_export]` มีพฤติกรรมพิเศษที่**ต่างจาก `pub` ของ item ปกติโดยสิ้นเชิง** — มันจะทำให้ macro นั้นถูกยกไปอยู่ที่ **crate root เสมอ** ไม่ว่าคุณจะประกาศ macro นั้นไว้ลึกแค่ไหนในโครงสร้าง module ก็ตาม พูดอีกแบบคือ `#[macro_export]` **ไม่สนใจ module hierarchy เลย** ต่างจาก `pub fn`/`pub struct` ที่ยังคงอยู่ตาม path ของ module ที่มันถูกประกาศ (`crate::math_utils::my_function`) ผลคือถ้าคุณมี macro ประกาศอยู่ใน `mod deep { mod nested { #[macro_export] macro_rules! foo {...} } }` คนที่จะใช้ macro นี้จากนอก crate จะเขียน `use my_crate::foo;` (เหมือนอยู่ที่ root) **ไม่ใช่** `use my_crate::deep::nested::foo;` แม้ว่ามันถูกประกาศอยู่ลึกขนาดนั้นก็ตาม — นี่คือความแปลกที่หลายคนงงตอนเจอครั้งแรก เพราะขัดกับสัญชาตญาณที่ได้จาก Part 16 ว่า path ของ item จะตาม module tree เสมอ

**วิธีที่ 2: `pub use` แบบ 2018+ edition (แนวทางใหม่กว่า สอดคล้องกับ module system มากขึ้น)**

ตั้งแต่ Rust 2018 edition เป็นต้นมา คุณสามารถใช้ `pub(crate) use` หรือ `pub use` เพื่อ re-export macro ให้ตาม path ของ module จริง ๆ ได้ (ยังต้องมี `#[macro_export]` หรือประกาศแบบ `pub macro_rules!`-like บางกรณี แต่โดยทั่วไปการผสม `use crate::some_macro;` ในไฟล์ที่ต้องการใช้ก็ทำงานได้ดีขึ้นกว่าเดิมมาก เพราะ macro (ตั้งแต่ 2018 edition) สามารถ `use` เข้ามาใน scope ได้เหมือน item ทั่วไปในหลาย ๆ กรณี):

```rust
mod formatting {
    macro_rules! bracketed {
        ($x:expr) => {
            format!("[{}]", $x)
        };
    }
    // re-export macro ให้ path เป็นไปตาม module จริง (2018+ edition behavior)
    pub(crate) use bracketed;
}

fn main() {
    use formatting::bracketed;
    let s = bracketed!(42);
    println!("{s}");  // [42]
}
```

**สรุปเชิงปฏิบัติ:** ถ้าคุณเขียน macro สำหรับใช้เองภายใน crate เดียวและอยากให้ path สอดคล้องกับ module tree แบบที่ Part 16 สอนไว้ ให้ใช้แนวทาง `pub(crate) use macro_name;` แต่ถ้าคุณกำลังเขียน library ที่จะ publish ให้คนอื่นใช้และต้องการความเข้ากันได้แบบดั้งเดิม (backward compatible กับวิธีที่ macro หลาย ๆ ตัวใน ecosystem ใช้กันมานาน) `#[macro_export]` ยังคงเป็นวิธีมาตรฐานที่พบมากที่สุด แต่ต้องจำ quirk เรื่อง "ไปโผล่ที่ crate root เสมอ" ไว้เสมอ เพื่อไม่ให้งงตอน debug เรื่อง path ไม่เจอ macro

### 36.12 Recursive Macro: สร้าง `max!` และ `min!` แบบ Variadic

อีกเทคนิคสำคัญของ declarative macro คือ **recursion** — macro ที่เรียกตัวเองซ้ำ ๆ โดยแต่ละครั้งลด argument ลง จนกว่าจะถึง base case ก็คล้ายกับ recursive function ทั่วไป แต่เกิดขึ้นตอน compile time ทั้งหมด

มาสร้าง `my_max!` ที่รับ argument 2 ตัวขึ้นไป แล้วคืนค่ามากที่สุด:

```rust
macro_rules! my_max {
    // Base case: มี argument เดียว -> คืนค่านั้นตรง ๆ
    ($x:expr) => {
        $x
    };
    // Recursive case: มี argument ตัวแรก ตามด้วยตัวที่เหลืออีกอย่างน้อย 1 ตัว
    // $($rest:expr),+ ใช้ tt เก็บส่วนที่เหลือไว้ส่งต่อแบบไม่แตะ
    ($x:expr, $($rest:expr),+) => {
        {
            let a = $x;
            let b = my_max!($($rest),+);  // เรียกตัวเองซ้ำ ด้วย argument ที่เหลือ (ตัดตัวแรกออกไปแล้ว)
            if a > b { a } else { b }
        }
    };
}

macro_rules! my_min {
    ($x:expr) => {
        $x
    };
    ($x:expr, $($rest:expr),+) => {
        {
            let a = $x;
            let b = my_min!($($rest),+);
            if a < b { a } else { b }
        }
    };
}

fn main() {
    println!("max(3, 7) = {}", my_max!(3, 7));
    println!("max(3, 7, 1, 9, 2) = {}", my_max!(3, 7, 1, 9, 2));
    println!("min(3, 7, 1, 9, 2) = {}", my_min!(3, 7, 1, 9, 2));
    println!("max(single) = {}", my_max!(42));
}
```

ผลลัพธ์:

```
max(3, 7) = 7
max(3, 7, 1, 9, 2) = 9
min(3, 7, 1, 9, 2) = 1
max(single) = 42
```

**อธิบายกลไก recursion ทีละ step ด้วย `my_max!(3, 7, 1, 9, 2)` เป็นตัวอย่าง:**

1. Compiler ลองจับคู่กับ arm แรก `($x:expr)` — ไม่ match เพราะมี comma อยู่ (มากกว่า 1 argument)
2. จับคู่กับ arm ที่สอง `($x:expr, $($rest:expr),+)` — match! `$x` = `3`, `$rest` (repetition) = `[7, 1, 9, 2]`
3. ขยายเป็น: `{ let a = 3; let b = my_max!(7, 1, 9, 2); if a > b { a } else { b } }`
4. `my_max!(7, 1, 9, 2)` ถูกเรียกซ้ำ — compiler จับคู่กับ arm 2 อีกครั้ง: `$x` = `7`, `$rest` = `[1, 9, 2]`
5. ขยายเป็น: `{ let a = 7; let b = my_max!(1, 9, 2); if a > b { a } else { b } }`
6. ทำซ้ำแบบนี้ไปเรื่อย ๆ จนเหลือ `my_max!(2)` ซึ่งจับคู่กับ **arm แรก** (base case) แล้วคืน `2` ตรง ๆ
7. จากนั้นค่าจะถูกเทียบกลับขึ้นมาเป็นชั้น ๆ (คล้าย recursive function ทั่วไปที่ unwind กลับ) จนได้ผลลัพธ์สุดท้าย `9`

สิ่งที่สำคัญมากคือ **การ recursion ทั้งหมดนี้เกิดขึ้นตอน compile time** — เมื่อ compile เสร็จ โค้ดที่ได้จริงคือ chain ของ `if...else` ที่ขยายเต็ม ๆ ไม่มี function call หรือ recursion เหลืออยู่ใน binary ที่รันจริงเลย (ตรงข้ามกับ recursive function ที่ recursion เกิดขึ้นตอน**runtime** และมี stack frame จริงในแต่ละ call) นี่คือความแตกต่างเชิงลึกอีกจุดหนึ่งระหว่าง macro กับฟังก์ชันที่ตอกย้ำหัวข้อ 36.1: macro ทำงาน (และ "จบงาน") ก่อนโปรแกรมจะรันจริงเสียอีก

**ข้อจำกัดสำคัญของ recursive macro:** Rust compiler มี **recursion limit** เริ่มต้นอยู่ที่ 128 ระดับ ถ้า macro ของคุณ recurse ลึกเกินไป (เช่นเรียก `my_max!` ด้วย argument หลายร้อยตัว) จะเจอ error:

```
error: recursion limit reached while expanding `my_max!`
  --> src/main.rs:6:21
   |
 6 | ...b = my_max!($($rest),+);
   |        ^^^^^^^^^^^^^^^^^^^
   |
   = help: consider increasing the recursion limit by adding a `#![recursion_limit = "256"]` attribute to your crate
   = note: this error originates in the macro `my_max`
```

วิธีแก้คือเพิ่ม `#![recursion_limit = "256"]` (หรือค่าที่มากกว่า) ไว้ที่ต้นไฟล์ `main.rs`/`lib.rs` — แต่ในทางปฏิบัติ ถ้า macro ของคุณต้อง recurse เกินหลักสิบ มักเป็นสัญญาณว่าควรพิจารณาออกแบบใหม่ (เช่นใช้ iterator method อย่าง `.iter().max()` ซึ่งเป็นวิธี idiomatic ที่ Part 25-26 สอนไว้ สำหรับกรณีที่ argument มาจาก collection อยู่แล้ว ไม่ใช่ literal ที่พิมพ์ตรง ๆ ในโค้ด)

### 36.13 ตัวอย่างประยุกต์ใช้งานจริง: `hashmap!` Macro

ทีนี้มารวมทุกเทคนิคที่เรียนมา (repetition, multiple metavariable ต่อรอบ, trailing comma) มาสร้าง macro ที่ใช้งานจริงบ่อยมาก: `hashmap!` ที่สร้าง `HashMap` จาก key-value pair ในรูปแบบ `key => value` (เชื่อมกับ Part 15)

```rust
use std::collections::HashMap;

macro_rules! hashmap {
    // กรณีว่างเปล่า
    () => {
        HashMap::new()
    };
    // กรณีมี key => value อย่างน้อย 1 คู่ คั่นด้วย comma รองรับ trailing comma
    ( $( $key:expr => $value:expr ),+ $(,)? ) => {
        {
            let mut map = HashMap::new();
            $(
                map.insert($key, $value);
            )+
            map
        }
    };
}

fn main() {
    // สมมติสถานการณ์จริง: ระบบเก็บราคาสินค้าในร้านค้าออนไลน์ (เชื่อมกับ Part 15)
    let prices: HashMap<&str, f64> = hashmap! {
        "กาแฟ" => 45.0,
        "ชาไทย" => 40.0,
        "โกโก้" => 50.0,
    };

    let mut names: Vec<&&str> = prices.keys().collect();
    names.sort();
    for name in names {
        println!("{name}: {} บาท", prices[name]);
    }

    let empty: HashMap<&str, i32> = hashmap!();
    println!("จำนวนสินค้าใน empty map: {}", empty.len());
}
```

ผลลัพธ์:

```
กาแฟ: 45 บาท
ชาไทย: 40 บาท
โกโก้: 50 บาท
จำนวนสินค้าใน empty map: 0
```

**อธิบายส่วนที่ต่างจาก `my_vec!`:**

- pattern `$key:expr => $value:expr` มี metavariable **2 ตัวต่อ 1 รอบ repetition** คั่นด้วย token literal `=>` (ต้องพิมพ์ `=>` ตรงตามที่กำหนดตอนเรียก macro เสมอ) และแต่ละคู่ (`key => value`) คั่นกันด้วย `,`
- เมื่อ compiler จับคู่ `"กาแฟ" => 45.0, "ชาไทย" => 40.0, "โกโก้" => 50.0` มันจะสร้าง**รายการคู่**ของ `$key`/`$value` ไว้ 3 คู่พร้อมกัน (นึกภาพเหมือน `zip` ของสอง list) แล้วตอนขยาย `$( map.insert($key, $value); )+` จะทำซ้ำ 3 รอบ โดยแต่ละรอบ `$key`/`$value` ถูกแทนที่จากคู่เดียวกันเสมอ (ไม่มีทางเกิดการ "ผสมคู่ผิด" เพราะระบบ repetition ผูก metavariable ที่มาจาก pattern เดียวกันไว้ด้วยกันเป็น "กลุ่ม" เสมอ)
- เราเรียก macro ด้วย `hashmap! { ... }` (ใช้ปีกกาแทนวงเล็บ) — สังเกตว่า `macro_rules!` macro เรียกได้ทั้ง 3 แบบของวงเล็บ: `foo!(...)`, `foo![...]`, `foo!{...}` โดย**ความหมายเหมือนกันทุกแบบ** (ตัวเลือกวงเล็บเป็นเรื่องของ "convention" ล้วน ๆ ไม่ใช่ syntax rule ที่ตายตัว — ธรรมเนียมทั่วไปคือใช้ `()` เมื่อ macro ทำตัวเหมือน expression/statement, ใช้ `[]` เมื่อทำตัวเหมือน collection literal อย่าง `vec![]`, และใช้ `{}` เมื่อ template มีหลายบรรทัดจัด layout สวย ๆ อย่าง struct/map literal — เป็นเหตุผลว่าทำไม `vec!` นิยมเขียนด้วย `[]` แต่ `hashmap!` สไตล์นี้นิยมเขียนด้วย `{}`)

**ทดสอบด้วย unit test (Part 32):**

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_hashmap_macro_basic() {
        let m: HashMap<&str, i32> = hashmap! { "a" => 1, "b" => 2 };
        assert_eq!(m.get("a"), Some(&1));
        assert_eq!(m.get("b"), Some(&2));
        assert_eq!(m.len(), 2);
    }

    #[test]
    fn test_hashmap_macro_empty() {
        let m: HashMap<i32, i32> = hashmap!();
        assert!(m.is_empty());
    }

    #[test]
    fn test_hashmap_macro_trailing_comma() {
        let m: HashMap<&str, i32> = hashmap! { "x" => 10, };
        assert_eq!(m.get("x"), Some(&10));
    }
}
```

### 36.14 เมื่อไหร่ควรใช้ Macro และเมื่อไหร่ไม่ควร

หัวข้อนี้สำคัญไม่แพ้เรื่อง syntax เลย — เพราะการรู้ "วิธีเขียน" macro โดยไม่รู้ "เมื่อไหร่ควรเขียน" อาจนำไปสู่โค้ดที่อ่านยากและ debug ยากโดยไม่จำเป็น

**Macro เหมาะกับปัญหาแบบไหน (ที่ generics/function แก้ไม่ได้):**

1. **API ที่ต้องรับจำนวน argument ไม่จำกัด โดยที่ argument แต่ละตัวมี type ต่างกันได้** — อย่าง `println!`, `vec!`, `hashmap!` ที่เราสร้างมา เป็นสิ่งที่ Part 4 บอกไว้แล้วว่าฟังก์ชันธรรมดาทำไม่ได้ (fixed arity, no overloading, no variadics)
2. **การ generate โค้ดที่ซ้ำซ้อนในรูปแบบที่ generics ทำไม่ถึง** — ตัวอย่างคลาสสิกคือการ implement trait เดียวกันให้กับหลาย ๆ type ที่ไม่มีความสัมพันธ์กันทาง type system (generics ทำงานผ่าน "type parameter เดียวที่ครอบคลุมทุก type ที่ตรง bound" แต่บางครั้งคุณต้องการ implementation ที่**ต่างกันเล็กน้อย**สำหรับแต่ละ concrete type ซึ่ง generics เดียวไม่พอ) — เราจะเห็นตัวอย่างด้านล่าง
3. **DSL (Domain-Specific Language) ขนาดเล็กภายในโค้ด Rust** — เช่น macro สำหรับสร้าง test case แบบ table-driven, macro สำหรับ define state machine, หรือ macro สำหรับสร้าง builder pattern อัตโนมัติ

**ตัวอย่างข้อ 2 — generate trait implementation ซ้ำๆ ให้หลาย type:**

```rust
trait Describe {
    fn describe(&self) -> String;
}

// ไม่มี macro: ต้องเขียนซ้ำแบบนี้สำหรับทุก primitive type ที่ต้องการ
// impl Describe for i32 {
//     fn describe(&self) -> String { format!("i32 ค่า {}", self) }
// }
// impl Describe for f64 {
//     fn describe(&self) -> String { format!("f64 ค่า {}", self) }
// }
// impl Describe for bool { ... } // ซ้ำไปเรื่อย ๆ

// ใช้ macro ลดความซ้ำซ้อน: ระบุแค่ชื่อ type แล้ว macro generate impl ให้เอง
macro_rules! impl_describe {
    ($($t:ty),+ $(,)?) => {
        $(
            impl Describe for $t {
                fn describe(&self) -> String {
                    format!("{} ค่า {}", stringify!($t), self)
                }
            }
        )+
    };
}

impl_describe!(i32, f64, bool, char);

fn main() {
    println!("{}", 42i32.describe());
    println!("{}", 3.14f64.describe());
    println!("{}", true.describe());
    println!("{}", 'x'.describe());
}
```

ผลลัพธ์:

```
i32 ค่า 42
f64 ค่า 3.14
bool ค่า true
char ค่า x
```

(หมายเหตุ: `stringify!` คือ macro built-in อีกตัวที่แปลง token เป็น string literal ตอน compile time — ใช้ `stringify!($t)` เพื่อได้ชื่อ type เป็น string โดยไม่ต้อง implement `Debug`/`Display` อะไรเพิ่ม)

macro `impl_describe!` นี้**ไม่สามารถแทนที่ด้วย generic เดียวได้** เพราะเราต้องการให้ทุก type มี implementation ที่**เขียนแยกกันจริง ๆ** (`impl Describe for i32`, `impl Describe for f64`, ...) ไม่ใช่ blanket implementation เดียว (`impl<T: Display> Describe for T { ... }` ซึ่งจะชนกับ coherence rule ถ้ามีคนอื่นพยายาม implement `Describe` ให้ type ของตัวเองด้วย — เป็นรายละเอียดเรื่อง trait coherence ที่ Part 21 พูดถึงบางส่วน) macro ในที่นี้ทำหน้าที่แค่ "พิมพ์โค้ดซ้ำ ๆ ให้เรา" โดยที่โค้ดที่พิมพ์ออกมายังคงเป็น concrete implementation แยกกันตามปกติทุกอย่าง

**แต่ macro ก็มีข้อเสียที่ชัดเจน ต้องชั่งน้ำหนักเสมอ:**

1. **Error message แย่กว่ามาก** — เมื่อโค้ดใน macro ผิด (เช่น type ไม่ตรงกันหลัง expand) error message ของ compiler จะชี้ไปที่**โค้ดที่ expand ออกมา** ซึ่งคุณไม่เห็นตรง ๆ ในซอร์สโค้ด ทำให้ต้องเดาหรือใช้ `cargo expand` (เครื่องมือเสริมที่ไม่ได้ติดมากับ cargo default) เพื่อดูโค้ดที่ขยายจริงถึงจะเข้าใจ error
2. **IDE/rust-analyzer ช่วยได้น้อยกว่า** — Auto-completion, "go to definition", type hint ที่คุณคุ้นเคยจาก Part 1 (rust-analyzer) มักทำงานได้ไม่ดีเท่าโค้ดปกติ**ภายใน**ตัว macro definition เอง เพราะ macro body ยังไม่ถูก expand ให้ตรวจสอบจนกว่าจะถูกเรียกใช้จริง
3. **อ่านยากกว่าฟังก์ชัน** — คนอ่านโค้ดต้องเข้าใจทั้ง "macro syntax" และ "เนื้อหาที่ macro ทำ" พร้อมกัน ในขณะที่ฟังก์ชันธรรมดาอ่านตรงไปตรงมา (เห็น signature ก็รู้ input/output ทันที)
4. **Debug ยากกว่า** — คุณ step-through macro ด้วย debugger แบบ breakpoint ทีละบรรทัดใน macro definition ไม่ได้ตรง ๆ เหมือนฟังก์ชันปกติ (ต้อง step เข้าไปในโค้ดที่ expand แล้ว)

**เครื่องมือช่วยเวลา error message ชี้ไปที่โค้ดที่ expand แล้วจนงง:** ติดตั้ง `cargo-expand` (subcommand เสริมของ cargo ที่ต้องติดตั้งเพิ่มเองด้วย `cargo install cargo-expand`) แล้วรัน:

```bash
cargo expand --bin my_project   # แสดงโค้ดทั้งหมดหลัง macro ทุกตัวถูก expand แล้ว
```

คำสั่งนี้จะพิมพ์**โค้ด Rust ฉบับเต็มหลังจาก macro ทุกตัวถูกขยายเสร็จแล้ว** ออกมาให้ดู — เช่นถ้าคุณเรียก `my_vec!(1, 2, 3)` ในโค้ด `cargo expand` จะแสดงให้เห็นว่ามันกลายเป็น `{ let mut v = Vec::new(); v.push(1); v.push(2); v.push(3); v ; }` จริง ๆ ตามที่เราอธิบายไว้ในหัวข้อ 36.7 ทำให้เวลา error message ชี้ไปที่โค้ดที่ขยายแล้วซึ่งไม่ตรงกับซอร์สโค้ดที่คุณเห็น คุณสามารถเปิดผลลัพธ์จาก `cargo expand` เทียบดูได้ทันทีว่า macro ขยายผิดจากที่ตั้งใจไว้ตรงไหน — เป็นเครื่องมือที่นักพัฒนา Rust ที่เขียน macro บ่อย ๆ ติดตั้งไว้แทบทุกเครื่อง

**หลักการตัดสินใจ (เชื่อมกับ Part 1 เรื่อง "zero-cost abstraction"):** Rust ออกแบบ generics และ trait ให้เป็น **zero-cost abstraction** — คุณใช้ generic function หรือ trait ได้โดยไม่เสีย performance เลยเมื่อเทียบกับเขียน concrete code ด้วยมือ (เพราะ monomorphization ตอน compile time — Part 18/22) เพราะฉะนั้น **ในแง่ performance generics กับ macro ให้ผลลัพธ์เท่ากัน** (ทั้งคู่ resolve ทุกอย่างตอน compile time ไม่มี runtime overhead) แต่ **generics อ่านง่ายกว่า debug ง่ายกว่า และมี tooling support ดีกว่ามาก** ดังนั้นกฎที่ควรยึดถือคือ:

> **ถามตัวเองก่อนเสมอว่า "ปัญหานี้ generic function/trait แก้ได้ไหม" ถ้าคำตอบคือได้ ให้ใช้ generics/trait ก่อน macro ควรเป็นตัวเลือกสุดท้าย (last resort) ที่ใช้เฉพาะตอนที่ปัญหาเป็นเรื่อง "จำนวน/รูปแบบ argument ที่ type system ปกติแก้ไม่ได้จริง ๆ" เท่านั้น**

ตารางสรุปเปรียบเทียบ 3 เครื่องมือหลักที่ Rust มีให้สำหรับ "ลดความซ้ำซ้อนของโค้ด" ซึ่งเราเรียนมาครบทั้งหมดแล้วตอนนี้ (generics จาก Part 18/22, trait จาก Part 19/21, และ declarative macro จากบทนี้):

| คุณสมบัติ | Generic function/trait | Declarative macro (`macro_rules!`) | Procedural macro |
|---|---|---|---|
| ทำงานตอนไหน | Compile time (monomorphization) | Compile time (token expansion) | Compile time (รันโค้ด Rust จริงเพื่อ generate) |
| Performance runtime | Zero-cost (เท่ากับเขียนมือ) | Zero-cost (เท่ากับเขียนมือ) | Zero-cost (เท่ากับเขียนมือ) |
| รับจำนวน argument ไม่จำกัดได้ไหม | ไม่ได้ (fixed arity ตาม Part 4) | ได้ (ผ่าน repetition) | ได้ (ควบคุมเองผ่านโค้ด) |
| IDE support (completion, go-to-def) | ดีมาก | ปานกลาง-น้อย | น้อยที่สุด (ต้อง expand ก่อนตรวจ) |
| อ่าน/debug ง่ายแค่ไหน | ง่ายที่สุด | ปานกลาง (ต้องรู้ macro syntax) | ยากที่สุด (ต้องเข้าใจทั้ง proc macro crate) |
| เขียนยากแค่ไหน | ง่าย-ปานกลาง | ปานกลาง-ยาก | ยากที่สุด (ต้องพึ่ง `syn`/`quote`) |
| ตัวอย่างที่คุ้นเคย | `Vec<T>`, `Option<T>`, `impl Display for T` | `vec!`, `println!`, `hashmap!` ของเรา | `#[derive(Debug)]`, `#[tokio::main]` |

ลำดับความสำคัญที่ควรใช้ (จากบนลงล่าง คือ**ควรลองก่อน**ไปเรื่อย ๆ จนกว่าจะพบตัวที่ตอบโจทย์): **(1) generic function/trait ธรรมดา → (2) declarative macro ถ้ามีปัญหาเรื่อง argument ไม่จำกัด/รูปแบบหลากหลาย → (3) procedural macro ถ้าต้องวิเคราะห์โครงสร้าง type อย่างละเอียด (เช่นรู้จำนวน field ของ struct)** อย่ากระโดดไปใช้เครื่องมือที่อยู่ล่างกว่าถ้าตัวที่อยู่บนกว่ายังทำงานได้ดีอยู่

ตัวอย่างสถานการณ์ที่ควรใช้ generic แทน macro (ตัวอย่าง "ผิด" ที่ overkill):

```rust
// ไม่ควรทำ: ใช้ macro สำหรับสิ่งที่ generic function ทำได้อยู่แล้วอย่างง่าย ๆ
macro_rules! add_one {
    ($x:expr) => {
        $x + 1
    };
}

// ควรทำแบบนี้แทน: generic function ธรรมดา อ่านง่ายกว่า debug ง่ายกว่า
// performance เท่ากันเป๊ะ (Rust compiler inline ฟังก์ชันเล็ก ๆ แบบนี้ให้อัตโนมัติอยู่แล้ว)
fn add_one<T: std::ops::Add<i32, Output = T>>(x: T) -> T {
    x + 1
}

fn main() {
    println!("{}", add_one!(5));  // ใช้ macro (ทำงานได้ แต่ overkill)
    println!("{}", add_one(5));   // ใช้ function (ดีกว่า — อ่านง่ายกว่า, IDE ช่วยเต็มที่)
}
```

ทั้งสองแบบให้ผลลัพธ์เหมือนกันและ performance เท่ากัน แต่ `add_one` แบบฟังก์ชันดีกว่าในทุกมิติอื่น: มี type signature ชัดเจนให้ rust-analyzer ช่วย, error message ตรงไปตรงมา, และไม่ต้องเรียนรู้ macro syntax เพื่อเข้าใจว่ามันทำอะไร — นี่คือตัวอย่างที่ตอกย้ำว่า macro ไม่ใช่ "เครื่องมือที่เจ๋งกว่าเสมอ" แต่เป็นเครื่องมือที่มีที่ใช้เฉพาะทางจริง ๆ

## กับดักที่พบบ่อย (Common Pitfalls)

**1. ลืม fragment specifier แล้วใส่ `$x` เปล่า ๆ ใน pattern**

```rust
macro_rules! broken {
    ($x) => {   // ผิด! ลืมระบุ fragment specifier
        println!("{}", $x);
    };
}
```

Error ที่ได้:

```
error: missing fragment specifier
 --> src/main.rs:2:6
  |
2 |     ($x) => {   // ผิด! ลืมระบุ fragment specifier
  |      ^^
  |
  = note: fragment specifiers must be provided
  = help: valid fragment specifiers are `ident`, `block`, `stmt`, `expr`, `pat`,
          `ty`, `lifetime`, `literal`, `path`, `meta`, `tt`, `item` and `vis`
help: try adding a specifier here
  |
2 |     ($x:spec) => {   // ผิด! ลืมระบุ fragment specifier
  |        +++++
```

**วิธีแก้:** ต้องระบุ fragment specifier เสมอตอนประกาศ pattern เช่น `$x:expr` ระบบ `macro_rules!` **ไม่มี** การ infer type ให้ metavariable เอง เพราะมันไม่ได้รู้จัก "type" แบบ Rust ทั่วไป มันรู้จักแค่ "ประเภทของไวยากรณ์ (syntax category)" เท่านั้น การไม่ระบุ specifier ทำให้ compiler ไม่รู้ว่าจะพยายาม parse token ที่ตามมาแบบไหน

---

**2. ใช้ expression ที่มี side effect ซ้ำ เมื่อ metavariable ถูกอ้างถึงหลายครั้งใน template**

```rust
macro_rules! square {
    ($x:expr) => {
        $x * $x   // อันตราย! ถ้า $x มี side effect จะรันซ้ำ 2 ครั้ง
    };
}

fn next_number() -> i32 {
    println!("กำลังเรียก next_number()...");  // side effect: การ print
    5
}

fn main() {
    let result = square!(next_number());
    // ขยายเป็น: next_number() * next_number()
    // ผลคือ "กำลังเรียก next_number()..." จะถูก print 2 ครั้ง!
    // และถ้าฟังก์ชันนี้คืนค่าไม่เหมือนกันทุกครั้ง (เช่นอ่านจาก counter ที่เพิ่มขึ้น)
    // ผลลัพธ์จะผิดพลาดโดยไม่รู้ตัว
    println!("result = {result}");
}
```

ผลลัพธ์ (จะเห็นข้อความ print ซ้ำ 2 ครั้งซึ่งไม่ตรงกับสิ่งที่คนอ่านโค้ดคาดหวัง):

```
กำลังเรียก next_number()...
กำลังเรียก next_number()...
result = 25
```

**วิธีแก้:** เก็บค่าไว้ในตัวแปรก่อนใช้ซ้ำ (เทคนิคเดียวกับที่เราใช้ใน `time_it!` และ `my_max!` ก่อนหน้านี้):

```rust
macro_rules! square_fixed {
    ($x:expr) => {
        {
            let val = $x;   // ประเมิน $x แค่ครั้งเดียว เก็บไว้ในตัวแปร
            val * val       // ใช้ตัวแปรซ้ำ ไม่ใช่ $x ซ้ำ
        }
    };
}
```

นี่คือกับดักที่พบบ่อยที่สุดของมือใหม่ที่เขียน `macro_rules!` เพราะภาษาไม่มี warning เตือนอัตโนมัติ — ต้องระวังด้วยตัวเองทุกครั้งที่ metavariable แบบ `expr` ถูกใช้มากกว่า 1 ครั้งใน template

---

**3. ลืม arm สำหรับกรณีว่างเปล่า (empty case) ตอนใช้ `+` แทน `*`**

```rust
macro_rules! sum_all {
    ( $( $x:expr ),+ ) => {   // ใช้ + (หนึ่งตัวขึ้นไป) แต่ไม่มี arm รองรับ 0 ตัว
        {
            let mut total = 0;
            $( total += $x; )+
            total
        }
    };
}

fn main() {
    let s = sum_all!();  // เรียกแบบไม่มี argument
    println!("{s}");
}
```

Error:

```
error: unexpected end of macro invocation
 --> src/main.rs:10:15
   |
10 |     let s = sum_all!();
   |               ^^^^^^^ missing tokens in macro arguments
```

**วิธีแก้:** ถ้าต้องการรองรับกรณีว่างเปล่าด้วย ให้ใช้ `*` แทน `+` หรือเพิ่ม arm แยกสำหรับ `()` แบบที่เราทำใน `my_vec!`:

```rust
macro_rules! sum_all_fixed {
    () => { 0 };  // arm แยกสำหรับกรณีว่างเปล่า
    ( $( $x:expr ),+ ) => {
        {
            let mut total = 0;
            $( total += $x; )+
            total
        }
    };
}
```

หรือใช้ `*` ตัวเดียวถ้าตัว initial value (`0`) เหมือนกันในทุกกรณีอยู่แล้ว (ไม่ต้องมี 2 arm):

```rust
macro_rules! sum_all_star {
    ( $( $x:expr ),* ) => {   // * รองรับ 0 ตัวได้ในตัวเอง ไม่ต้องมี arm แยก
        {
            let mut total = 0;
            $( total += $x; )*
            total
        }
    };
}
```

---

**4. ลืมรองรับ trailing comma ทำให้ macro ใช้งานไม่สะดวก**

```rust
macro_rules! my_vec_no_trailing {
    ( $( $x:expr ),+ ) => {   // ไม่มี $(,)? ต่อท้าย
        {
            let mut v = Vec::new();
            $( v.push($x); )+
            v
        }
    };
}

fn main() {
    let v = my_vec_no_trailing!(1, 2, 3,);  // มี comma ท้ายสุด
    println!("{v:?}");
}
```

Error:

```
error: unexpected end of macro invocation
  --> src/main.rs:9:13
   |
1  | macro_rules! my_vec_no_trailing {
   | -------------------------------- when calling this macro
...
9  |     let v = my_vec_no_trailing!(1, 2, 3,);  // มี comma ท้ายสุด
   |             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ missing tokens in macro arguments
   |
note: while trying to match meta-variable `$x:expr`
```

**วิธีแก้:** เพิ่ม `$(,)?` ต่อท้าย pattern เสมอเมื่อใช้ comma-separated repetition (ตามที่ทำใน `my_vec!` และ `hashmap!` ก่อนหน้า) — เป็นธรรมเนียมมาตรฐานของ macro ระดับ production เพราะผู้ใช้ที่คุ้นเคยกับการเขียน struct literal/array literal ใน Rust (ที่รองรับ trailing comma เป็นปกติ) จะคาดหวังพฤติกรรมเดียวกันจาก macro ของคุณด้วย ถ้าไม่รองรับจะทำให้ผู้ใช้หงุดหงิดโดยไม่จำเป็น

---

**5. สับสนระหว่างลำดับ arm ที่ทำให้ arm ทั่วไปบัง arm เฉพาะ**

```rust
macro_rules! check {
    ($x:expr) => {          // arm ทั่วไป (จับคู่ expression อะไรก็ได้) มาก่อน
        println!("เป็น expression ทั่วไป: {}", stringify!($x));
    };
    ($x:literal) => {       // arm เฉพาะเจาะจงกว่า (จับคู่แค่ literal) แต่ไม่มีทางถูกเรียกใช้!
        println!("เป็น literal โดยเฉพาะ: {}", $x);
    };
}

fn main() {
    check!(42);  // คาดว่าจะได้ arm ที่ 2 (literal) แต่จริง ๆ ได้ arm ที่ 1 เสมอ
}
```

ผลลัพธ์ (ไม่ error แต่ผลลัพธ์ผิดจากที่คาดหวัง — เป็นกับดักที่อันตรายกว่าเพราะไม่มี error เตือน):

```
เป็น expression ทั่วไป: 42
```

**สาเหตุ:** `macro_rules!` จับคู่ arm ตาม**ลำดับที่เขียนจากบนลงล่าง** และหยุดที่ arm แรกที่ match ได้ทันที เพราะ `42` เป็นทั้ง `literal` และ `expr` ได้ (literal เป็น subset ของ expression) arm แรก (`$x:expr`) จะ match ก่อนเสมอ ทำให้ arm ที่สอง (`$x:literal`) ไม่มีโอกาสถูกใช้เลยไม่ว่าจะเรียกแบบไหน **วิธีแก้:** เรียงลำดับ arm จาก**เฉพาะเจาะจงที่สุดไปทั่วไปที่สุด** เสมอ (หลักการเดียวกับการเรียง arm ของ `match` ใน Part 10 ที่ arm เฉพาะเจาะจงต้องมาก่อน arm ทั่วไปอย่าง `_`):

```rust
macro_rules! check_fixed {
    ($x:literal) => {        // ย้าย arm เฉพาะเจาะจงมาก่อน
        println!("เป็น literal โดยเฉพาะ: {}", $x);
    };
    ($x:expr) => {           // arm ทั่วไปอยู่ท้ายสุด
        println!("เป็น expression ทั่วไป: {}", stringify!($x));
    };
}
```

---

**6. ใช้ metavariable สองกลุ่มที่ repeat ไม่เท่ากันในการขยาย (mismatched repetition)**

```rust
// macro ที่ตั้งใจจะบวกค่าจากสองรายการ (list a และ list b) เข้าคู่กันทีละตัว
macro_rules! pair_sum {
    ( $( $a:expr ),* ; $( $b:expr ),* ) => {
        {
            let mut total = 0;
            $( total += $a + $b; )*   // ผิด! ใช้ $a กับ $b ในการ repeat เดียวกัน
                                       // ทั้งที่จับคู่มาจาก pattern คนละกลุ่ม
            total
        }
    };
}

fn main() {
    let s = pair_sum!(1, 2, 3 ; 10, 20);  // list a มี 3 ตัว, list b มี 2 ตัว
    println!("{s}");
}
```

Error:

```
error: meta-variable `a` repeats 3 times, but `b` repeats 2 times
 --> src/main.rs:5:14
  |
5 |             total += $a + $b;
  |             ^^^^^^^^^^^^^^^^^^^^^
```

**สาเหตุ:** เมื่อคุณเขียน `$( ... )*` ที่มี metavariable มากกว่า 1 ตัวอยู่ในกลุ่มเดียวกัน (เช่น `$( total += $a + $b; )*`) compiler จะพยายาม**ขยายทั้งสองตัวไปด้วยกันแบบคู่ต่อคู่ (paired iteration)** เหมือนกับ `zip` ของสอง iterator (Part 25-26) — แต่ `$a` มาจาก repetition กลุ่มแรก (`$($a:expr),*` ที่จับคู่ได้ 3 รอบ) และ `$b` มาจาก repetition กลุ่มที่สอง (`$($b:expr),*` ที่จับคู่ได้ 2 รอบ) ซึ่งเป็น**คนละกลุ่ม repetition กัน** และมีจำนวนรอบไม่เท่ากัน compiler จึงไม่รู้ว่าจะขยายกี่รอบและจะจับคู่ตัวที่เกินมาของ `$a` กับอะไร จึง error ทันที (นี่คือการตรวจสอบที่ชาญฉลาดมาก เพราะถ้าไม่มี safety check นี้ โปรแกรมจะรัน silent bug ที่ตรวจจับยากมาก — คล้ายกับการ `zip` สอง `Vec` ที่ความยาวไม่เท่ากันใน Rust ปกติซึ่ง `.zip()` จะตัดตามตัวที่สั้นกว่าเงียบ ๆ แต่ macro system เข้มงวดกว่านั้นด้วยการ error ให้เห็นตั้งแต่ compile time)

**วิธีแก้:** ถ้าตั้งใจให้ทั้งสอง list ยาวเท่ากันเสมอ ต้องรับผิดชอบเรื่องนี้เอง (เช่น เขียน comment เตือนผู้ใช้ macro หรือเพิ่ม `assert_eq!` ตรวจสอบความยาวถ้าเปลี่ยนมารับเป็น `Vec` ตอน runtime แทน) หรือถ้าต้องการให้ macro ทำงานได้แม้ length ไม่เท่ากัน ให้ออกแบบใหม่ให้ไม่ต้อง zip สอง repetition กลุ่มต่างกันในรอบเดียว เช่นเก็บเป็น `Vec` ก่อนแล้วใช้ `.iter().zip()` ทำงานตอน runtime แทนการพยายาม zip ตอน macro expansion

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** เขียน macro ชื่อ `shout!` ที่รับ argument เดียวเป็น `expr` (สมมติว่าเป็น `&str`) แล้ว print ข้อความนั้นตามด้วยเครื่องหมาย `!!!` สามตัว เช่น `shout!("ระวัง")` ควร print `ระวัง!!!` จากนั้นเพิ่ม arm ที่สองที่รับ 2 argument (`expr` และตัวเลขจำนวนครั้งที่จะ print ซ้ำเป็น `expr`) ให้ `shout!("ระวัง", 3)` print `ระวัง!!!` สามบรรทัด
   (hint: arm ที่สองต้องมาก่อนหรือหลัง arm แรกก็ได้ในกรณีนี้ เพราะจำนวน argument ต่างกันชัดเจน ไม่ชนกัน ลองใช้ `for _ in 0..$n` ใน template)

2. **[กลาง]** เขียน macro ชื่อ `min_max!` ที่รับ argument ตั้งแต่ 1 ตัวขึ้นไป (ใช้ repetition `$(...),+`) แล้วคืนค่าเป็น tuple `(ค่าน้อยที่สุด, ค่ามากที่สุด)` โดยใช้แนวทาง "เก็บทุก argument ไว้ใน `Vec` ก่อน แล้วเรียก `.iter().min()`/`.iter().max()`" (เชื่อมกับ Part 25-26 เรื่อง iterator) ไม่ต้องใช้ recursive macro แบบ `my_max!`/`my_min!` ในบทเรียน — ลองวิธีที่ต่างออกไปดู เขียน unit test ยืนยันด้วยว่า `min_max!(5, 2, 8, 1, 9)` ให้ผลลัพธ์ `(1, 9)`

3. **[ยาก]** เขียน macro ชื่อ `enum_with_names!` ที่รับรายชื่อ identifier (ใช้ `$($name:ident),+`) แล้ว generate 2 อย่างพร้อมกัน: (ก) `enum` ที่มี variant ตามชื่อที่ระบุ และ (ข) ฟังก์ชัน `fn name(&self) -> &'static str` ที่คืนชื่อของ variant นั้นเป็น string (ใช้ `stringify!` และ `match` ภายใน macro — เชื่อมกับ Part 10) เช่น เรียก `enum_with_names!(Red, Green, Blue)` ควรสร้าง enum `enum Color { Red, Green, Blue }` (คุณต้องตั้งชื่อ enum ให้ macro รับเป็น argument แรกด้วย เช่น `enum_with_names!(Color; Red, Green, Blue)`) พร้อมให้ `Color::Red.name()` คืน `"Red"`
   (hint: ต้องใช้ metavariable 2 กลุ่ม คือชื่อ enum (`ident` เดี่ยว) และรายชื่อ variant (repetition ของ `ident`) คั่นด้วย `;`)

4. **[ยาก/ประยุกต์ใช้งานจริง]** สร้างระบบ "banking transaction log" ง่าย ๆ: เขียน macro ชื่อ `transaction!` ที่รับ pattern แบบ `deposit amount` หรือ `withdraw amount` (โดย `deposit`/`withdraw` เป็น token ตายตัว ไม่ใช่ metavariable, `amount` เป็น `expr`) แล้ว expand เป็นการเรียกฟังก์ชัน (หรือ method ของ struct `Account` ที่คุณออกแบบเอง ซึ่งมี field `balance: f64`) ที่ตรวจสอบว่า `withdraw` ไม่ทำให้ balance ติดลบ (ถ้าติดลบให้ print คำเตือนแทนการหักเงิน) เขียนโปรแกรมตัวอย่างที่สร้าง `Account` แล้วเรียก `transaction!(acc, deposit 1000.0)` และ `transaction!(acc, withdraw 300.0)` และ `transaction!(acc, withdraw 999999.0)` (กรณีเงินไม่พอ) แล้วแสดง balance สุดท้าย — โจทย์นี้ฝึกทั้ง multiple arm, token literal, และการเชื่อม macro เข้ากับ struct/method จริงจากบทก่อน ๆ

## สรุป

ในบทนี้เราได้เฉลยคำถามที่ค้างมาตั้งแต่ Part 1 อย่างละเอียด: `println!` และ macro อื่น ๆ ที่คุณใช้มาตลอดหลักสูตรไม่ใช่ฟังก์ชัน แต่เป็น**declarative macro** ที่ทำงาน**ตอน compile time** โดยจับคู่ (pattern-match) **token** ที่ส่งเข้ามา แล้วขยาย (expand) เป็นโค้ด Rust ชิ้นใหม่ **ก่อน**ที่ type checker จะเริ่มทำงาน — นี่คือกลไกที่อธิบายว่าทำไม macro รับจำนวน argument ไม่จำกัดและหลาย type พร้อมกันได้ ในขณะที่ฟังก์ชันธรรมดาทำไม่ได้เนื่องจาก fixed arity และการไม่มี function overloading (Part 4) และเป็นเหตุผลว่าทำไม format string ที่ผิดจึงเป็น compile error ไม่ใช่ runtime crash

เราได้เรียนรู้ syntax ของ `macro_rules!` ตั้งแต่พื้นฐาน (macro ไม่มี argument, macro รับ argument เดียว) ไปจนถึง fragment specifier ที่พบบ่อย (`expr`, `ident`, `ty`, `pat`, `block`, `stmt`, `tt`) และเทคนิคสำคัญที่สุดคือ **repetition** (`$(...),*` และ `$(...),+`) ซึ่งเราใช้สร้าง `my_vec!` และ `hashmap!` ของตัวเองจนใช้งานได้จริงและผ่านการทดสอบ เราได้เห็น **multiple match arm** ที่ทำให้ macro ตัวเดียวมีพฤติกรรมต่างกันตามรูปแบบ argument (เหมือน `match` ใน Part 10) และเทคนิค **recursive macro** ที่ใช้สร้าง `max!`/`min!` แบบ variadic

เราได้พิสูจน์ด้วยตัวอย่างจริงว่า Rust macro เป็น **hygienic** — ตัวแปรภายใน macro ไม่ชนกับตัวแปรของผู้เรียกใช้ ต่างจากปัญหาคลาสสิกของ C's `#define` และได้เรียนเรื่อง `$crate` สำหรับอ้างอิง crate ต้นทางอย่างถูกต้อง และ `#[macro_export]` พร้อม quirk เชิงประวัติศาสตร์ที่ทำให้ macro ที่ export ออกมาจะไปโผล่ที่ crate root เสมอ ต่างจาก `pub` ของ item ปกติที่ Part 16 สอนไว้

ที่สำคัญไม่แพ้กัน คือหลักการตัดสินใจว่า**เมื่อไหร่ควรใช้ macro**: macro เหมาะกับปัญหาที่ generics/function แก้ไม่ได้จริง ๆ (variable-argument API, การ generate โค้ดซ้ำข้าม concrete type) แต่ควรเป็น**ตัวเลือกสุดท้าย** เพราะ error message แย่กว่า, IDE ช่วยได้น้อยกว่า, debug ยากกว่า — ในขณะที่ generics/trait ให้ performance เท่ากันเป๊ะ (zero-cost abstraction ตามที่ Part 1 บอกไว้) แต่อ่านและ debug ง่ายกว่ามาก

Part 36 นี้ปิดท้ายเนื้อหาเชิง "เขียนโค้ดให้กระชับและยืดหยุ่นขึ้น" ของโมดูล 2 (ระดับกลาง) ใน **Part 37** เราจะเปลี่ยนทิศทางไปสู่โลกของ **concurrency** — เริ่มจาก **Threads พื้นฐาน** (`std::thread`, `join`, move closure) ซึ่งเป็นก้าวแรกสู่การเขียนโปรแกรมที่ทำงานหลายอย่างพร้อมกันอย่างปลอดภัย ก่อนจะไปเรียน channel (Part 38), Mutex/Arc (Part 39) และ Send/Sync (Part 40) ปิดโมดูล 2 แล้วเข้าสู่โมดูล 3 (Advanced) ที่เริ่มด้วย unsafe Rust ใน Part 41

---

**Part ก่อนหน้า:** [Cargo ขั้นสูง](part-035-cargo-advanced.md) | **Part ถัดไป:** [Threads พื้นฐาน](part-037-threads-basics.md)
