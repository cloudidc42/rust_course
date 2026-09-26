# Part 44: Procedural Macros เบื้องต้น

> โมดูล: ระดับสูง (Advanced) | ระดับ: สูง | เวลาโดยประมาณ: 220 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้อย่างถูกต้องว่า **procedural macro** ต่างจาก **declarative macro** (`macro_rules!` จาก Part 36) อย่างไรในระดับกลไก ไม่ใช่แค่ syntax และอธิบายได้ว่าทำไม `macro_rules!` ทำสิ่งที่ proc macro ทำได้ไม่ได้จริง ๆ ในหลายกรณี
- จำแนกความแตกต่างระหว่าง proc macro ทั้ง 3 ชนิด — **derive macro**, **attribute macro**, **function-like macro** — และบอกได้ว่าแต่ละชนิดใช้ตอนไหน มีข้อจำกัดอะไร
- อธิบายเหตุผลเชิงกลไกว่าทำไม proc macro ต้องอยู่ใน crate แยกที่ประกาศ `proc-macro = true` โดยเชื่อมโยงกับความรู้เรื่อง crate/compilation unit จาก Part 17
- เข้าใจว่า `TokenStream` คืออะไร และทำไมนักพัฒนา Rust แทบทุกคนไม่ parse มันเองแบบ raw แต่ใช้ **`syn`** และ **`quote`** เป็นเครื่องมือมาตรฐานของ ecosystem แทน
- เขียน derive macro ของตัวเองตั้งแต่ต้นจนจบได้ — ตั้ง Cargo.toml ให้ถูกต้อง, parse input ด้วย `syn::parse_macro_input!`, ดึงข้อมูลจาก struct, generate โค้ดด้วย `quote!{}`, และใช้งานจาก crate อื่นได้จริง
- เขียน function-like proc macro แบบง่ายที่สุดได้ และรู้ว่า entry point ของมันต่างจาก derive macro อย่างไร
- ใช้ `cargo expand` และเทคนิค `eprintln!` เพื่อ debug proc macro ได้ และเขียน proc macro ที่รายงาน compile error อย่างมีอารยะด้วย `syn::Error`/`to_compile_error()` แทนการ `panic!`

## ความรู้ที่ต้องมีมาก่อน

- **Part 36 (Macros: Declarative Macros)** — นี่คือ prerequisite ที่สำคัญที่สุดของบทนี้ คุณต้องเข้าใจมาก่อนว่า macro ทำงาน**ตอน compile time** ไม่ใช่ runtime, เข้าใจกลไก **token matching** ของ `macro_rules!`, fragment specifier (`expr`, `ident`, `ty`, `tt`, ...), repetition syntax (`$(...),*`), และที่สำคัญที่สุดคือ Part 36 ได้**จำแนกไว้แล้ว**ว่า macro มี 2 ประเภทหลัก คือ declarative และ procedural — พร้อมยกตัวอย่าง `#[derive(Debug, Clone, PartialEq)]` ไว้สั้น ๆ ว่าเป็น proc macro ประเภท derive แล้วบอกไว้ชัดเจนว่า **"เราจะเรียน procedural macro แบบเต็ม ๆ ใน Part 44-45"** — นี่คือ Part 44 ที่ Part 36 พูดถึง เราจะไม่สอนเรื่อง `macro_rules!` ซ้ำในบทนี้ แต่จะอ้างอิงกลับไปที่ Part 36 ตลอดเวลาเพื่อเปรียบเทียบ
- **Part 17 (Packages, Crates, Workspaces)** — เรื่อง crate ในฐานะ **compilation unit**, ความแตกต่างระหว่าง library crate กับ binary crate, และแนวคิด crate type ต่าง ๆ — บทนี้จะขยายความเรื่อง crate type ไปอีกแบบหนึ่งที่ Part 17 ยังไม่ได้พูดถึง คือ **`proc-macro` crate type** ซึ่งมีกฎการ build ที่ต่างจาก library crate ปกติโดยสิ้นเชิง
- **Part 19/21 (Traits พื้นฐาน/ขั้นสูง)** — เพราะตัวอย่างหลักของบทนี้คือการเขียน derive macro ที่ generate `impl` block ให้ struct โดยอัตโนมัติ คุณต้องเข้าใจ syntax และความหมายของ `impl TypeName { ... }` มาก่อนจึงจะเข้าใจว่า macro ของเรากำลัง "เขียนโค้ดแบบเดียวกับที่คุณเขียนมือ" อยู่
- **Part 9/19 (การใช้ `#[derive(Debug, Clone, ...)]`)** — คุณใช้ `#[derive(...)]` มาตั้งแต่ Part 9 โดยไม่รู้ว่ามันทำงานอย่างไรข้างใน บทนี้คือบทที่เฉลยคำถามนั้นแบบเต็ม ๆ

## เนื้อหา

### 44.1 ทวนความจำจาก Part 36 และคำถามที่ค้างไว้: ทำไม `macro_rules!` ยังไม่พอ

จาก Part 36 คุณได้เรียนมาแล้วว่า Rust มี macro 2 ประเภทหลัก:

| | Declarative macro (`macro_rules!`) | Procedural macro |
|---|---|---|
| กลไกพื้นฐาน | **จับคู่ pattern ของ token** แล้ว substitute เข้า template (เหมือน "find-and-replace ที่ฉลาดมาก") | **โค้ด Rust จริง** ที่รันตอน compile time รับ token stream เป็น input แล้วคืน token stream เป็น output — ทำอะไรก็ได้ตาม logic ที่เขียน |
| เขียนด้วยอะไร | syntax เฉพาะของ `macro_rules!` เอง (arm, fragment specifier, repetition) | ฟังก์ชัน Rust ธรรมดา ที่ต้องอยู่ใน crate พิเศษ |
| "เห็น" โครงสร้างโค้ดได้แค่ไหน | เห็นแค่ "รูปแบบของ token" ตาม pattern ที่ผู้เขียนกำหนดไว้ล่วงหน้าเท่านั้น | parse token stream เป็น **AST (Abstract Syntax Tree)** ที่มี struct/field/type ครบ แล้ววิเคราะห์/generate โค้ดด้วย logic โปรแกรมมิ่งเต็มรูปแบบ |
| ตัวอย่างที่คุณรู้จัก | `println!`, `vec!`, `my_vec!`, `hashmap!` (จาก Part 36) | `#[derive(Debug)]`, `#[tokio::main]`, `sqlx::query!` |

คำถามที่ควรค้างอยู่ในใจคุณตอนนี้คือ: **ถ้า `macro_rules!` ทรงพลังพอจะสร้าง `vec!`/`hashmap!` ของตัวเองได้ (Part 36 หัวข้อ 36.7) แล้วทำไมเราต้องมี macro อีกระบบหนึ่งที่ซับซ้อนกว่ามาก?** คำตอบอยู่ที่ขอบเขตของสิ่งที่ **pattern matching แบบ token-tree** ทำได้จริง เรามาดูตัวอย่างที่ชัดเจนที่สุด: การ generate เมธอด `describe()` ที่พิมพ์ชื่อ struct และชื่อทุก field ของมันโดยอัตโนมัติ — สิ่งที่คุณใช้ `#[derive(Debug)]` ทำอยู่แล้วในทางปฏิบัติ (เพียงแต่ format ต่างกัน)

**ลองด้วย `macro_rules!` ก่อน** — เพื่อดูว่ามันพอไปได้แค่ไหน:

```rust
// ความพยายามใช้ macro_rules! ให้ทำสิ่งที่ derive macro ทำ
macro_rules! struct_with_describe {
    (
        struct $name:ident {
            $( $field:ident : $ty:ty ),* $(,)?
        }
    ) => {
        struct $name {
            $( $field: $ty ),*
        }

        impl $name {
            fn describe(&self) {
                println!("struct: {}", stringify!($name));
                $(
                    println!("  field: {}", stringify!($field));
                )*
            }
        }
    };
}

struct_with_describe! {
    struct Customer {
        name: String,
        age: u32,
    }
}

fn main() {
    let c = Customer { name: "สมชาย".to_string(), age: 30 };
    c.describe();
}
```

ผลลัพธ์:

```
struct: Customer
  field: name
  field: age
```

**น่าประหลาดใจไหม? มันคอมไพล์และทำงานได้จริง!** สังเกตกลไก: เราใช้ repetition (`$( $field:ident : $ty:ty ),*`) จับคู่รายการ field ทั้งหมดพร้อมกัน แล้วขยาย (expand) ทั้ง `struct` ตัวจริงและ `impl` block ที่ generate `describe()` ออกมาพร้อมกันในการเรียกครั้งเดียว — นี่คือเทคนิคที่ถูกต้องตาม Part 36 หัวข้อ 36.7 ทุกประการ

**แต่ตัวอย่างนี้ซ่อนปัญหาใหญ่ 3 อย่างที่ทำให้แนวทางนี้ใช้งานจริงไม่ได้ในสถานการณ์ทั่วไป:**

**ปัญหาที่ 1 — คุณต้อง "เขียนนิยาม struct ใหม่ผ่าน macro" แทนการเขียน struct ปกติ** สังเกตว่าเราต้องเขียน `struct Customer { ... }` ทั้งก้อน**อยู่ภายในการเรียก macro** (`struct_with_describe! { struct Customer { ... } }`) นี่ต่างจากที่คุณคุ้นเคยมาตั้งแต่ Part 9 อย่างสิ้นเชิง ที่คุณเขียน struct ปกติ แล้ว**แค่ติด attribute** `#[derive(Debug, Clone, Describe)]` ไว้ด้านบน โดยที่ตัว struct ยังเป็น struct ปกติทุกประการ ไม่ต้องห่อด้วย macro call ใด ๆ — การห่อทั้ง struct ไว้ใน macro แบบนี้ทำให้คุณ**เสีย ergonomic ที่สำคัญที่สุดของ derive macro ไปเลย**: การที่ derive macro หลายตัว **compose กันได้อย่างอิสระ** บน struct เดียวกัน (`#[derive(Debug, Clone, Serialize, Describe)]`) โดยแต่ละตัวเป็นอิสระจากกันโดยสมบูรณ์ ถ้าคุณต้องการให้ `Customer` มีทั้ง `Debug`, `Clone`, และ `describe()` พร้อมกันด้วยแนวทาง `macro_rules!` นี้ คุณต้องเขียน macro `struct_with_describe!` ให้ generate `#[derive(Debug, Clone)]` ไว้ในตัวมันเองด้วย ซึ่งทำให้ผู้ใช้**เลือกเองไม่ได้อีกต่อไป**ว่าจะ derive อะไรบ้าง (ต้องตายตัวตามที่ macro กำหนดไว้)

**ปัญหาที่ 2 — `$ty:ty` เป็น "กล่องดำ" ที่มองเข้าไปข้างในไม่ได้** สมมติว่าคุณต้องการให้ `describe()` **ข้าม field ที่เป็น `Option<T>`** ไปโดยอัตโนมัติ (ไม่พิมพ์ field ที่เป็น optional) คุณจะเขียน logic แบบนี้ด้วย `macro_rules!` ไม่ได้เลย เพราะ `$ty:ty` **จับคู่กับ type ทั้งก้อนแบบทึบ (opaque)** — มันรู้แค่ว่า "ตรงนี้คือ type หนึ่งตัว" แต่ไม่มีทางถามได้ว่า "type ตัวนี้คือ `Option<...>` หรือเปล่า" เพราะ fragment specifier ทำงานแค่ตอน**จับคู่กับ pattern ที่ผู้เขียน macro พิมพ์เป็น literal token ไว้ล่วงหน้า** ไม่ใช่การ**วิเคราะห์โครงสร้าง**ของสิ่งที่จับคู่ได้ ต่างจาก proc macro ที่ parse type เป็น `syn::Type` ซึ่งเป็น `enum` ที่มี variant เช่น `Type::Path` ให้คุณเช็คชื่อ segment แรกของ path ได้จริง ๆ ว่าเป็น `Option` หรือไม่ (เราจะเห็นความสามารถนี้แบบเต็ม ๆ ใน Part 45)

**ปัญหาที่ 3 — ต้อง reimplement ไวยากรณ์ของ Rust เองทุกกรณีย่อย** ตัวอย่างข้างบนรองรับแค่ struct ที่มี named field แบบง่ายที่สุด ถ้า struct มี generic parameter (`struct Container<T> { ... }`), มี lifetime (`struct Ref<'a> { ... }`), มี `where` clause, มี attribute บน field แต่ละตัว (`#[serde(default)] name: String`), เป็น tuple struct, หรือเป็น `enum` ที่มีทั้ง variant แบบไม่มีข้อมูลและมีข้อมูล — pattern ของ `macro_rules!` ต้องเขียน arm แยกสำหรับแต่ละกรณีเพิ่มขึ้นเรื่อย ๆ จนซับซ้อนจัดการไม่ได้ ในขณะที่ proc macro ใช้ **`syn::DeriveInput`** ซึ่งเป็น struct ที่ออกแบบมาให้ครอบคลุมไวยากรณ์ของ item ทุกกรณีในภาษา Rust ไว้ให้แล้ว (เพราะทีมงาน `syn` implement parser ตามไวยากรณ์จริงของ Rust ทั้งภาษา) คุณแค่ pattern-match บน `enum` ของ `syn` เพื่อดึงข้อมูลที่ต้องการออกมา ไม่ต้องเขียน parser เองเลย

**สรุปหัวข้อนี้:** `macro_rules!` เก่งเรื่อง**จับคู่รูปแบบของ token** และ**generate โค้ดตาม template** — มันเหมาะกับปัญหาที่ "รูปร่าง" ของ input ค่อนข้างตายตัวและไม่ต้องวิเคราะห์เชิงลึก (เช่น `vec!`, `hashmap!`) แต่พอปัญหาต้องการ **"อ่านและวิเคราะห์โครงสร้างของโค้ด Rust จริง ๆ"** (เช่น "สำหรับทุก field ใน struct นี้ ให้ทำสิ่งนี้" โดยต้องรู้จำนวน field, ชื่อ field, และรายละเอียดของ type แต่ละตัวอย่างแม่นยำ) `macro_rules!` จะเริ่มฝืนธรรมชาติของมันอย่างรุนแรง — และนี่คือช่องว่างที่ **procedural macro** เข้ามาเติมเต็ม: มันคือโค้ด Rust จริงที่ parse token stream เป็น AST ที่มีโครงสร้างสมบูรณ์ แล้ววิเคราะห์/generate โค้ดด้วย logic โปรแกรมมิ่งแบบเต็มรูปแบบ ไม่ใช่แค่ pattern matching แบบ token

### 44.2 สามชนิดของ Procedural Macro: Derive, Attribute, Function-like

Procedural macro ใน Rust แบ่งเป็น 3 ชนิดตาม **วิธีที่มันถูกเรียกใช้** และ **ความสามารถในการแก้ไข item ที่มันติดอยู่**

**1. Derive macro** — เรียกผ่าน `#[derive(TraitName)]` (แบบเดียวกับ `#[derive(Debug, Clone)]` ที่คุณใช้มาตลอดหลักสูตร) มันได้รับ token stream ของ item ที่มัน derive อยู่ (struct/enum/union) เป็น **input แบบอ่านอย่างเดียว** และหน้าที่ของมันคือ **generate โค้ดใหม่เพิ่มเข้ามาข้าง ๆ** (มักจะเป็น `impl Trait for TypeName { ... }`) — **มันแก้ไขหรือลบ item เดิมไม่ได้เลย** item เดิมที่ผู้ใช้เขียนไว้จะยังคงอยู่เหมือนเดิมทุกประการเสมอ สิ่งที่ derive macro ทำได้คือ "เพิ่มโค้ดใหม่ที่วางคู่กัน" เท่านั้น (เราพิสูจน์ข้อจำกัดนี้แล้วด้วยตาตัวเองในหัวข้อกับดักท้ายบท)

**2. Attribute macro** — เรียกผ่าน attribute ของตัวเอง เช่น `#[my_attribute] fn foo() { ... }` หรือที่คุณอาจเคยเห็นมาก่อนคือ `#[tokio::main]` (จะเรียนละเอียดใน Part 46) มันได้รับ **สอง TokenStream**: (ก) token ของตัว attribute เอง (เช่น argument ที่ใส่ในวงเล็บถ้ามี) และ (ข) token ของ item ที่มันติดอยู่ทั้งก้อน — และที่สำคัญที่สุดคือมัน **คืน TokenStream ใหม่ที่ใช้แทนที่ item เดิมทั้งหมด** พูดง่าย ๆ คือ attribute macro **อ่านและเขียนทับ item เดิมได้เต็มที่** ทรงพลังกว่า derive macro มาก (เช่น `#[tokio::main]` เปลี่ยนฟังก์ชัน `async fn main()` ธรรมดาให้กลายเป็น `fn main()` ที่สร้าง async runtime ขึ้นมาแล้วรัน future ข้างในให้ — มันแก้ไข **signature และ body** ของฟังก์ชันเดิมไปเลย ไม่ใช่แค่เพิ่มโค้ดข้าง ๆ)

**3. Function-like macro** — เรียกด้วย `!` แบบเดียวกับ `macro_rules!` (เช่น `my_macro!(...)`) หน้าตาการเรียกใช้เหมือนกับ declarative macro ทุกประการ แต่เบื้องหลังเป็นฟังก์ชัน proc macro เต็มรูปแบบ รับ TokenStream ของสิ่งที่อยู่ในวงเล็บ แล้วคืน TokenStream ใหม่ทั้งหมด (ไม่ได้ "แทนที่ item เดิม" เพราะไม่มี item เดิม — มันแทนที่**ตัวเองทั้งการเรียก**เหมือน `macro_rules!`) ความต่างจาก `macro_rules!` คือภายในมันใช้โค้ด Rust วิเคราะห์ token ได้อย่างอิสระเต็มที่ ไม่ใช่แค่ pattern matching (ตัวอย่างจริง: `sqlx::query!("SELECT ...")` ที่ตรวจสอบ SQL กับ database schema จริงตอน compile time — เป็นสิ่งที่ `macro_rules!` ทำไม่ได้เลยเพราะมันไม่มีทาง "เชื่อมต่อ database" ได้ในกลไก pattern matching)

ตารางเปรียบเทียบทั้ง 3 ชนิด:

| คุณสมบัติ | Derive macro | Attribute macro | Function-like macro |
|---|---|---|---|
| เรียกใช้ยังไง | `#[derive(Name)]` | `#[name]` หรือ `#[name(args)]` | `name!(...)` |
| Input ที่ได้รับ | TokenStream ของ item ที่ derive อยู่ (อ่านอย่างเดียว) | 2 TokenStream: attribute args + item ทั้งก้อน | TokenStream ของสิ่งที่อยู่ในวงเล็บ |
| แก้ไข item เดิมได้ไหม | **ไม่ได้** — เพิ่มโค้ดใหม่ข้าง ๆ เท่านั้น | **ได้เต็มที่** — คืน TokenStream ใหม่แทนที่ item เดิมทั้งหมด | ไม่มี "item เดิม" — แทนที่ตัวเองทั้งหมด |
| ระดับความยาก/พลัง | ปานกลาง — เขียนบ่อยที่สุดในบรรดา 3 ชนิด | สูง — ต้องคิดทั้ง "อ่าน" และ "เขียนทับ" | สูง — คล้าย `macro_rules!` แต่ควบคุมได้เต็มที่กว่ามาก |
| ตัวอย่างที่ใช้บ่อย | `#[derive(Debug, Clone, Serialize, Deserialize)]` | `#[tokio::main]`, (`#[test]` ก็มีคุณสมบัติคล้ายกันในเชิงแนวคิด) | `sqlx::query!`, `html!` (จาก `yew`) |
| Entry point signature | `fn(TokenStream) -> TokenStream` | `fn(TokenStream, TokenStream) -> TokenStream` | `fn(TokenStream) -> TokenStream` |
| แก้ macro นี้บ่อยไหม (ในฐานะผู้เขียน) | นักพัฒนาทั่วไปมักเขียนเองบ้างสำหรับ domain trait ของตน | ส่วนใหญ่**ใช้**มากกว่า**เขียน** (ซับซ้อนกว่ามาก) | ส่วนใหญ่**ใช้**มากกว่า**เขียน** เช่นกัน |

สังเกตว่า **signature ของ derive macro และ function-like macro เหมือนกัน** (`fn(TokenStream) -> TokenStream`) ต่างกันแค่ **attribute ที่ประกาศ** (`#[proc_macro_derive(...)]` เทียบกับ `#[proc_macro]`) และ **วิธีที่ผู้ใช้เรียก** — ส่วน attribute macro มีความพิเศษตรงที่รับ 2 input พร้อมกัน เพราะต้องแยกระหว่าง "argument ของ attribute เอง" กับ "item ที่ attribute ติดอยู่"

**ตัวอย่างสั้น ๆ ของ attribute macro ที่ใช้งานได้จริง** เพื่อให้เห็น entry point แบบ 2-input จริง ๆ (ไม่ใช่แค่คำอธิบาย) มาดู `#[log_call]` ที่ห่อฟังก์ชันใด ๆ ให้พิมพ์ log ตอนเข้าและออกจากฟังก์ชันโดยอัตโนมัติ:

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, ItemFn};

/// attribute macro: #[log_call] fn foo() { ... }
/// รับ 2 TokenStream: _attr (argument ของ attribute เอง ถ้ามี — ในตัวอย่างนี้ไม่ใช้)
/// และ item (token ของทั้งฟังก์ชันที่ attribute ติดอยู่)
/// คืน TokenStream ใหม่ที่ "แทนที่" ฟังก์ชันเดิมทั้งหมด (ต่างจาก derive macro ที่ทำแบบนี้ไม่ได้)
#[proc_macro_attribute]
pub fn log_call(_attr: TokenStream, item: TokenStream) -> TokenStream {
    let input_fn = parse_macro_input!(item as ItemFn);

    let fn_name_str = input_fn.sig.ident.to_string();
    let fn_vis = &input_fn.vis;
    let fn_sig = &input_fn.sig;
    let fn_block = &input_fn.block;

    // สร้างฟังก์ชันใหม่ทั้งก้อน โดยเก็บ signature เดิมไว้ (ชื่อ, argument, return type)
    // แต่ "ห่อ" body เดิมด้วย println! ก่อน/หลังเรียกจริง — นี่คือการ "เขียนทับ" item เดิม
    quote! {
        #fn_vis #fn_sig {
            println!("[log_call] เข้าฟังก์ชัน: {}", #fn_name_str);
            let result = (|| #fn_block)();
            println!("[log_call] ออกจากฟังก์ชัน: {}", #fn_name_str);
            result
        }
    }
    .into()
}
```

ใช้งานจาก consumer crate:

```rust
use log_call_macro::log_call;

#[log_call]
fn add(a: i32, b: i32) -> i32 {
    a + b
}

fn main() {
    let sum = add(3, 4);
    println!("ผลรวม = {sum}");
}
```

ผลลัพธ์จริง (ตรวจสอบแล้วว่าคอมไพล์และรันผ่าน):

```
[log_call] เข้าฟังก์ชัน: add
[log_call] ออกจากฟังก์ชัน: add
ผลรวม = 7
```

สังเกตสิ่งที่ **derive macro ทำไม่ได้เลย**: ฟังก์ชัน `add` ที่ compiler เห็นจริง ๆ (หลัง expand) **ไม่ใช่** `fn add(a: i32, b: i32) -> i32 { a + b }` ตามที่ผู้ใช้เขียนอีกต่อไป — มันถูก**แทนที่ทั้งหมด**ด้วยเวอร์ชันที่มี `println!` ห่ออยู่ ผู้ใช้ `#[log_call]` ไม่ต้องแก้ body ของฟังก์ชันตัวเองแม้แต่นิดเดียว เพราะ attribute macro อ่าน**ทั้งฟังก์ชัน**ผ่าน `syn::ItemFn` (ที่มี `.sig` สำหรับ signature และ `.block` สำหรับ body แยกกันชัดเจน) แล้ว generate ฟังก์ชันใหม่ทั้งก้อนขึ้นมาแทน — เทียบกับ `Describe`/`Validated` ในหัวข้อก่อนที่ทำได้แค่ "เพิ่ม `impl` ข้าง ๆ" เท่านั้น

บทนี้จะโฟกัสที่ **derive macro** เป็นหลัก (เพราะเป็นชนิดที่นักพัฒนา Rust ทั่วไปมีโอกาสต้อง**เขียนเอง**บ่อยที่สุด สำหรับ domain trait ของตัวเอง) และ**function-like macro** แบบง่าย ๆ เพื่อให้เห็นความแตกต่างของ entry point ส่วน **attribute macro** เราเห็นตัวอย่างสั้น ๆ ไปแล้วข้างบนเพื่อให้เข้าใจกลไก 2-input/เขียนทับ item เดิม แต่จะไม่ลงรายละเอียดขั้นสูง (การรองรับ `async fn`, generic, หรือ argument ที่ซับซ้อนของตัว attribute เอง) ในบทนี้ — คุณจะเห็นการ**ใช้**มันอย่างเข้าใจลึกใน Part 46 ผ่าน `#[tokio::main]` แต่มีพื้นฐานพอที่จะอ่าน source code ของมันเข้าใจได้แล้วหลังจบหัวข้อนี้

### 44.3 กฎเชิงกลไก: Proc Macro ต้องอยู่ใน Crate ของตัวเอง (`proc-macro = true`)

จาก Part 17 คุณรู้มาแล้วว่า crate คือ **compilation unit** ที่ `rustc` มองเห็นเป็นหนึ่งหน่วยการคอมไพล์ และ crate หนึ่ง ๆ มักเป็น library crate (`src/lib.rs`) หรือ binary crate (`src/main.rs`) แต่ proc macro ต้องอยู่ใน crate ที่ประกาศตัวเองเป็น **crate type พิเศษ** ผ่าน `Cargo.toml`:

```toml
[package]
name = "describe_derive"
version = "0.1.0"
edition = "2021"

[lib]
proc-macro = true    # <-- นี่คือบรรทัดที่ทำให้ crate นี้เป็น "proc-macro crate"

[dependencies]
syn = { version = "2", features = ["full"] }
quote = "1"
proc-macro2 = "1"
```

**ทำไมต้องแยก crate?** นี่ไม่ใช่กฎที่ตั้งขึ้นมาโดยไม่มีเหตุผล แต่เป็นผลจากความสัมพันธ์เชิง**เวลา build** ที่ต่างไปจากปกติโดยสิ้นเชิง ลองเทียบกับ dependency ธรรมดาก่อน:

**dependency ปกติ (เช่น `serde` เป็น library ธรรมดา):** เมื่อ crate ของคุณ (`my_app`) depend on `serde`, `rustc` จะคอมไพล์ `serde` เป็น**โค้ดเครื่องของ target platform** (เช่น ARM ถ้าคุณ cross-compile ไป Raspberry Pi) แล้ว**link** เข้ากับ binary ของ `my_app` ที่รันบน target platform เดียวกัน — ทั้งสองอย่างอยู่ใน "โลกของ target" เดียวกันหมด

**proc macro crate ทำงานคนละแบบโดยสิ้นเชิง:** เมื่อ `my_app` depend on `describe_derive` (proc macro crate) สิ่งที่เกิดขึ้นคือ:

1. `rustc`/`cargo` ต้องคอมไพล์ `describe_derive` เป็นโปรแกรมที่**รันได้บนเครื่องที่กำลัง compile อยู่ตอนนี้** (host platform) — **ไม่ใช่** target platform ของ `my_app`
2. `describe_derive` ที่คอมไพล์แล้วจะถูกโหลดเข้ามา**รันจริงเป็นส่วนหนึ่งของกระบวนการ compile** ของ `my_app` — มันทำหน้าที่เหมือน **compiler plugin** ที่ `rustc` เรียกใช้ตอนกำลัง parse โค้ดของ `my_app` เพื่อขอให้มัน "ช่วยสร้างโค้ดเพิ่ม" ก่อนที่ `my_app` จะถูกคอมไพล์ต่อไปให้เสร็จสมบูรณ์
3. เพราะฉะนั้น `describe_derive` **ต้องถูกคอมไพล์เสร็จสมบูรณ์และพร้อมรันก่อน** ที่ `my_app` จะเริ่มคอมไพล์ได้ — มันไม่ใช่ความสัมพันธ์แบบ "compile คู่กันแล้ว link เข้าด้วยกันทีหลัง" แบบ dependency ปกติ แต่เป็น "compile ให้จบสมบูรณ์ก่อน แล้วเอามา**รัน**เพื่อช่วยสร้างโค้ดของอีกฝั่ง"

นี่คือเหตุผลที่ Rust ต้องมี**ป้ายบอกชัดเจน** (`proc-macro = true`) ว่า crate นี้ **ไม่ใช่ library ธรรมดาที่จะถูก link เข้ากับ target binary** — มันคือโปรแกรมที่ compiler ต้องปฏิบัติต่อแบบพิเศษ: คอมไพล์สำหรับ **host** ไม่ใช่ **target**, โหลดมันเป็น dynamic library ที่ compiler เรียกใช้ได้ตอน compile time, และห้ามมันมี item อื่นที่ export แบบ library ปกติปนอยู่ (เราจะเห็น error จริงของกรณีนี้ในหัวข้อกับดัก)

ผลข้างเคียงที่สำคัญของกฎนี้:

- **proc macro crate ใช้ macro ของตัวเองไม่ได้เลย** ลองนึกดูว่าถ้า `describe_derive` เขียน `#[cfg(test)] mod tests { #[derive(Describe)] struct Foo { x: i32 } ... }` ไว้ในไฟล์เดียวกัน จะเกิด**ปัญหาไข่กับไก่ (chicken-and-egg)**: การจะใช้ `#[derive(Describe)]` ได้ ต้องมี `describe_derive` ที่ compile เสร็จสมบูรณ์แล้ว**ก่อน** แต่นี่คือการ compile ตัว `describe_derive` เองอยู่ — มันยังไม่มีตัวเองที่ compile เสร็จให้ใช้ได้เลย ลองจริงจะได้ error ตรงตัวแบบนี้ (ตรวจสอบแล้ว):

  ```
  error: can't use a procedural macro from the same crate that defines it
    --> src/lib.rs:21:14
     |
  21 |     #[derive(Describe)]
     |              ^^^^^^^^
     |
     = help: you can define integration tests in a directory named `tests`
  ```

  สังเกตว่า compiler เองก็แนะนำวิธีแก้ไว้ในข้อความ `help:` แล้ว: ใช้ **integration test** (โฟลเดอร์ `tests/` ที่ Part 33 สอนไว้ — ไฟล์ในนั้นถูก compile เป็น**crate แยก**ที่ `extern crate`/`use` ตัว `describe_derive` ที่ compile เสร็จแล้วเข้ามา ไม่ใช่ส่วนหนึ่งของ `describe_derive` เอง) นี่คือเหตุผลเชิงกลไกอีกข้อที่ทำให้เราต้องมี**crate consumer แยก** (`describe_consumer` ในหัวข้อ 44.6) — ไม่ใช่แค่เพื่อความสะดวก แต่เป็นข้อจำกัดที่ compiler บังคับไว้จริง ๆ อย่างไรก็ตาม ตรรกะ**ภายใน**ของ macro (ฟังก์ชันที่ทำงานบน `proc_macro2::TokenStream` ไม่ใช่ตัว entry point ที่มี `#[proc_macro_derive]` ติดอยู่) ยัง `#[test]` ได้ตามปกติในไฟล์เดียวกัน เพราะมันเป็นฟังก์ชัน Rust ธรรมดาที่ไม่ต้องพึ่งกลไก "เรียกใช้ derive macro ของตัวเอง" เลย — นี่คือสิ่งที่เราจะพิสูจน์และใช้จริงในหัวข้อ 44.6.1
- **proc macro crate มักไม่ใส่ business logic ปนเข้าไปเยอะ** เพราะมันแยกโลกกับ target platform อย่างชัดเจน (บาง project จะมี 2 crate คือ `foo` (library หลักที่ผู้ใช้ import มาใช้ทั้งหมด) กับ `foo-derive` หรือ `foo_macros` (proc macro crate ที่ `foo` เอง depend on อีกที เพื่อ re-export `#[derive(Foo)]` ผ่าน `pub use foo_derive::Foo;`) — pattern นี้พบได้ในหลาย crate ที่มีชื่อเสียง เช่น `serde`/`serde_derive`, `thiserror` ภายในก็มี proc macro crate ซ่อนอยู่ (ที่จริง `thiserror` ที่คุณใช้มาแล้วใน Part 31 คือตัวอย่างจริงของ derive macro ที่ generate `impl std::error::Error` ให้!)

**ตารางเปรียบเทียบ crate type ทั้งหมดที่เกี่ยวข้อง** (ต่อเนื่องจาก Part 17 ที่พูดถึงแค่ library crate กับ binary crate):

| Crate type | ประกาศใน `Cargo.toml` | คอมไพล์เป็นโค้ดของ platform ไหน | ใช้ทำอะไร |
|---|---|---|---|
| Binary crate | `src/main.rs` (ค่าเริ่มต้น ไม่ต้องประกาศ) | **target** | โปรแกรมที่รันได้จริงบนเครื่องปลายทาง |
| Library crate (`rlib`) | `src/lib.rs` (ค่าเริ่มต้น) | **target** | โค้ดที่ crate อื่นใน**โลกของ target เดียวกัน** import ไปใช้/link |
| `cdylib`/`staticlib` | `[lib] crate-type = ["cdylib"]` (Part 43 เรื่อง FFI) | **target** | โค้ด Rust ที่ให้ภาษาอื่น (เช่น C) เรียกใช้แบบ dynamic/static library |
| **`proc-macro`** | `[lib] proc-macro = true` | **host** (เครื่องที่กำลัง compile อยู่ — คนละแนวคิดกับที่ผ่านมาทั้งหมด) | โปรแกรมที่ `rustc` โหลดมา**รัน**เพื่อช่วย generate โค้ดให้ crate อื่นตอน compile time |

สังเกตว่า `proc-macro` เป็น crate type**เดียว**ในตารางที่ compile สำหรับ **host** ไม่ใช่ **target** — นี่สำคัญมากเวลา cross-compile (เช่นสร้างโปรแกรมสำหรับ ARM บนเครื่อง x86_64) สมมติคุณรัน `cargo build --target aarch64-unknown-linux-gnu` สำหรับโปรเจกต์ที่ใช้ `describe_derive`: `describe_consumer` (binary crate) จะถูกคอมไพล์เป็นโค้ดเครื่อง ARM ตามที่ `--target` ระบุ แต่ **`describe_derive` จะยังคงถูกคอมไพล์เป็นโค้ดเครื่อง x86_64 เสมอ** (ตาม host จริงที่กำลังรัน `cargo build` อยู่) เพราะมันต้อง**รันได้บนเครื่องที่กำลัง compile** ไม่ใช่รันบนเครื่อง ARM ปลายทาง — ถ้า Rust คอมไพล์ `describe_derive` เป็นโค้ด ARM ตาม `--target` ไปด้วย มันจะกลายเป็นโปรแกรมที่รันบนเครื่อง x86_64 ที่กำลัง build อยู่ไม่ได้เลย ทำให้กระบวนการ compile ทั้งหมดล้มเหลว — นี่คือเหตุผลเชิงลึกอีกชั้นที่ตอกย้ำว่า proc-macro crate ไม่ได้อยู่ใน "โลกของ target" เดียวกับโค้ดส่วนที่เหลือของโปรเจกต์คุณเลย

### 44.4 `TokenStream`: หัวใจของ Proc Macro ทุกตัว

ทุก proc macro ไม่ว่าชนิดไหนก็ตาม มีรูปแบบพื้นฐานเหมือนกันหมด: **รับ `TokenStream` เป็น input และคืน `TokenStream` เป็น output** `TokenStream` คือ type ที่มาจาก crate ชื่อ `proc_macro` ซึ่งเป็น crate พิเศษที่ `rustc` เตรียมไว้ให้ (ไม่ต้องเพิ่มใน `[dependencies]` แต่ต้องเป็น crate ที่มี `proc-macro = true` เท่านั้นถึงจะ `use proc_macro::TokenStream;` ได้ — ตามที่เราเห็น error จริงในหัวข้อกับดักถ้าลืมตั้งค่านี้)

**`TokenStream` คือลำดับของ "token" ของ source code Rust ดิบ ๆ** — พูดง่าย ๆ คือมันถูก**หั่น**เป็นชิ้น ๆ ตาม lexical grammar ของภาษา (identifier, literal, punctuation, delimiter) แล้วแต่**ยังไม่ถูก parse ให้เป็นโครงสร้างที่มีความหมายเชิงไวยากรณ์** (ไม่รู้ว่า "นี่คือ struct ที่มีชื่อ X และมี field 3 ตัว" — รู้แค่ว่า "มี token `struct`, ตามด้วย identifier `X`, ตามด้วย `{`, ...") เทียบกับ fragment ที่ `macro_rules!` จับคู่ได้ (เช่น `$x:expr`) ซึ่งถูกจับคู่ตาม grammar ไว้แล้วในระดับหนึ่ง — `TokenStream` แบบดิบยัง**ไม่มี**การจัดกลุ่มเชิงความหมายแบบนั้นเลย มันเป็นแค่ stream ของ token ตามลำดับที่เขียนในซอร์สโค้ด

**ในทางทฤษฎี คุณสามารถเขียน proc macro โดย parse `TokenStream` ด้วยมือทั้งหมดได้** — วน iterate ผ่านแต่ละ token, เช็คว่าเป็น `Ident`/`Literal`/`Punct`/`Group` อะไร, เขียน logic จับกลุ่มเองว่า "ตรงนี้คือ field ของ struct" — **แต่ในทางปฏิบัติ แทบไม่มีใครทำแบบนี้เลย** เพราะมันคือการ reimplement Rust parser บางส่วนด้วยมือ ซึ่งมีรายละเอียดยิบย่อยมหาศาล (attribute, generic, lifetime, doc comment, raw identifier, ฯลฯ) และมีโอกาสเขียนผิดสูงมาก

**นี่คือจุดที่ ecosystem ของ Rust สร้าง "มาตรฐานโดยพฤตินัย" (de facto standard) ขึ้นมา** เปรียบเทียบให้เห็นภาพ: การเขียนเว็บแอปด้วยการจัดการ raw HTTP request/response ด้วยมือ**ทำได้ในทางทฤษฎี** แต่ในทางปฏิบัติแทบไม่มีใครทำ ทุกคนใช้ framework (Express, Django, Axum ที่จะเรียนใน Part 62) เพราะมันจัดการรายละเอียดที่น่าเบื่อและเสี่ยงผิดพลาดให้หมดแล้ว — proc macro ก็เช่นกัน: แทนที่จะ parse `TokenStream` ดิบด้วยมือ นักพัฒนา Rust เกือบทั้งหมด**ใช้ 2 crate นี้เป็นคู่กันเสมอ**:

- **`syn`** — parse `TokenStream` ดิบให้กลายเป็น **AST ที่มี type ครบถ้วน** (เช่น `syn::DeriveInput` สำหรับ struct/enum ที่ derive อยู่, `syn::ItemFn` สำหรับฟังก์ชัน, `syn::Type` สำหรับ type) คุณสามารถ pattern-match และดึงข้อมูล (ชื่อ struct, ชื่อ field, ชนิดของ field) ออกมาได้เหมือนทำงานกับ struct/enum ปกติของ Rust
- **`quote`** — ทำงาน**ตรงข้ามกับ `syn`**: มันแปลง**โค้ด Rust ที่คุณเขียนตรง ๆ ในซอร์ส** (ผ่าน macro `quote!{ ... }`) ให้กลายเป็น `TokenStream` โดยรองรับการ**แทรกค่า** (interpolation) ผ่าน `#variable` syntax ข้อดีมหาศาลคือ คุณเขียนโค้ดที่จะ generate ในรูปแบบที่**อ่านออกเป็นโค้ด Rust จริง ๆ** (ไม่ใช่การต่อ string หรือสร้าง token object ทีละตัวด้วยมือ) แล้วปล่อยให้ `quote!` แปลงเป็น `TokenStream` ให้เอง

และมักมี **`proc-macro2`** ควบคู่มาด้วยเสมอ — เป็น wrapper รอบ `proc_macro::TokenStream` ของ `rustc` ที่ทำให้ `TokenStream` ใช้งานได้**นอก context ของ proc macro compiler plugin** ด้วย (เช่น เขียน unit test ธรรมดาสำหรับ logic ข้างในของ proc macro ได้โดยไม่ต้องผ่านกระบวนการ compile จริงทุกครั้ง) — `syn` และ `quote` ทั้งคู่ทำงานบน `proc_macro2::TokenStream` เป็นหลัก (ไม่ใช่ `proc_macro::TokenStream` ตรง ๆ) แล้วคุณ**แปลงกลับ**เป็น `proc_macro::TokenStream` (ผ่าน `.into()`) ตอนจะคืนค่าออกจากฟังก์ชัน entry point ของ proc macro เท่านั้น — นี่คือจุดที่มือใหม่สับสนบ่อยที่สุด (และเป็นกับดักข้อหนึ่งท้ายบท) เพราะทั้งสอง type ชื่อเหมือนกันเป๊ะ (`TokenStream`) แต่เป็น**คนละ type จากคนละ crate**

สรุปภาพรวม pipeline ของ proc macro แบบมาตรฐานของ ecosystem:

```
ผู้ใช้เขียนโค้ด (struct/fn/...)
        │
        ▼  compiler ส่ง raw TokenStream (proc_macro::TokenStream) มาให้
   [ syn::parse_macro_input! ]   -- แปลง raw TokenStream เป็น AST ที่มี type (เช่น DeriveInput)
        │
        ▼  โค้ด Rust ธรรมดา วิเคราะห์/ดึงข้อมูลจาก AST
   [ ตรรกะของคุณเอง ]
        │
        ▼  สร้างโค้ดใหม่ด้วย quote! ได้ proc_macro2::TokenStream
   [ quote!{ ... } ]
        │
        ▼  .into() แปลงกลับเป็น proc_macro::TokenStream
   [ คืนค่าออกจากฟังก์ชัน entry point ]
        │
        ▼
compiler เอา TokenStream ที่ได้ไปรวมเข้ากับโค้ดของผู้ใช้ แล้ว compile ต่อตามปกติ
```

### 44.5 Anatomy ของ `syn`: `DeriveInput` และการดึงข้อมูลจาก AST

มาดูโครงสร้างของ `syn::DeriveInput` แบบคร่าว ๆ ก่อนลงมือเขียนโค้ดจริง (นี่คือ struct ที่ `syn` มอบให้เมื่อ parse input ของ derive macro):

```rust
// โครงสร้างของ syn::DeriveInput (แสดงแบบย่อเพื่อความเข้าใจ ไม่ใช่ต้อง copy ไปใช้)
pub struct DeriveInput {
    pub attrs: Vec<Attribute>,   // attribute ที่ติดอยู่บน item เช่น #[derive(...)] เอง
    pub vis: Visibility,         // pub, pub(crate), หรือ private
    pub ident: Ident,            // ชื่อของ struct/enum/union (เช่น "Customer")
    pub generics: Generics,      // generic parameter, lifetime, where clause
    pub data: Data,              // <-- ส่วนสำคัญที่สุด: struct/enum/union body
}

// Data เป็น enum แยกตามชนิดของ item
pub enum Data {
    Struct(DataStruct),
    Enum(DataEnum),
    Union(DataUnion),
}

// DataStruct เก็บ field ทั้งหมดของ struct
pub struct DataStruct {
    pub fields: Fields,
    // ...
}

// Fields แยกตามรูปแบบของ struct (named / tuple / unit)
pub enum Fields {
    Named(FieldsNamed),       // struct Foo { x: i32, y: i32 }
    Unnamed(FieldsUnnamed),   // struct Foo(i32, i32);  (tuple struct)
    Unit,                     // struct Foo;
}
```

สังเกตว่านี่คือ**struct/enum ของ Rust ธรรมดา** ที่คุณอ่านเข้าใจได้ทันทีด้วยความรู้จาก Part 9/10/19 — ไม่มีอะไรพิเศษเลย เพราะ `syn` ถูกออกแบบมาให้ AST เป็น type ปกติของ Rust ที่คุณ pattern-match ได้ด้วย `match` (Part 10) ตรงไปตรงมา นี่คือความแตกต่างเชิงคุณภาพจาก `macro_rules!`: `Fields::Named(fields)` ให้คุณเข้าถึง `fields.named` ซึ่งเป็น `Punctuated<Field, Comma>` (คิดเป็น iterator ของ field ที่คั่นด้วย comma ได้เลย) แต่ละ `Field` มี `.ident: Option<Ident>` (ชื่อ field) และ `.ty: Type` (type ของ field ที่เป็น AST ต่ออีกชั้นหนึ่ง ไม่ใช่ token ดิบ) — คุณสามารถเขียนโค้ด Rust ธรรมดา (loop, `if let`, `match`) เพื่อดึงข้อมูลเหล่านี้ออกมาได้อย่างเป็นธรรมชาติ

ฟังก์ชันตัวช่วยที่คุณจะเห็นในเกือบทุก derive macro คือ `syn::parse_macro_input!` — macro ตัวนี้ (ใช่ มันเป็น `macro_rules!` macro ที่มาช่วย proc macro อีกที เป็นตัวอย่างที่ดีว่าทั้งสองระบบไม่ได้แยกกันโดยสิ้นเชิง) ทำหน้าที่:

1. เรียก `syn::parse::<DeriveInput>(input)` เพื่อ parse `TokenStream` ดิบให้เป็น `DeriveInput`
2. ถ้า parse **สำเร็จ** — คืนค่า `DeriveInput` ออกมาให้ใช้งานต่อได้ทันที
3. ถ้า parse **ล้มเหลว** (เช่น input ไม่ใช่ syntax ของ Rust ที่ถูกต้องเลย ซึ่งไม่ควรเกิดขึ้นเพราะ compiler การันตีว่า input เป็นโค้ด Rust ที่ผ่านการ parse ระดับ syntax มาแล้วในระดับหนึ่ง แต่ก็ป้องกันไว้เผื่อกรณีสุดโต่ง) — **มันจะ `return` ออกจากฟังก์ชันทันทีพร้อมสร้าง compile error ที่ถูกต้องให้เอง** (ใช้เทคนิคเดียวกับ `syn::Error::to_compile_error()` ที่เราจะเห็นในหัวข้อ 44.9) นี่คือเหตุผลที่ macro ตัวนี้ต้องเป็น "statement-like macro" ที่เรียกแบบ `let x = parse_macro_input!(input as DeriveInput);` ไม่ใช่ฟังก์ชันธรรมดา เพราะมันต้อง `return` ออกจากฟังก์ชันที่เรียกมันได้โดยตรง (ซึ่งเป็นสิ่งที่ฟังก์ชันธรรมดาทำแทนกันไม่ได้ — ต้องใช้ macro เพื่อ "แทรก" `return` เข้าไปในจุดที่เรียก)

ฝั่ง `quote!{}` ก็มีกลไกที่คล้ายกับ repetition ของ `macro_rules!` (Part 36 หัวข้อ 36.7) มาก — แทนที่จะใช้ `$(...)* ` มันใช้ `#(...)* ` แทน:

```rust
// เทียบ syntax repetition: macro_rules! ใช้ $(...)*  ส่วน quote! ใช้ #(...)*
// ตัวอย่าง: field_names เป็น Vec<String> ที่มีสมาชิกหลายตัว
let field_names: Vec<String> = vec!["name".to_string(), "age".to_string()];

// quote! วนซ้ำผ่าน field_names ทั้งหมด แทรกทีละตัว คั่นด้วย comma (,)
let tokens = quote::quote! {
    [ #(#field_names),* ]
};
// ผลลัพธ์ (เป็น TokenStream ที่ขยายแล้ว): ["name", "age"]
```

นี่คือเหตุผลที่บอกว่า Part 36 ไม่ได้ "สูญเปล่า" เลยแม้จะโฟกัสที่ declarative macro — mental model ของ repetition, fragment, และการ interpolate ตัวแปรเข้าไปในโค้ดต้นแบบที่คุณฝึกมาแล้วใน Part 36 นำมาใช้กับ `quote!` ได้เกือบตรงตัว เปลี่ยนแค่สัญลักษณ์จาก `$` เป็น `#`

**มาพิสูจน์คำกล่าวในหัวข้อ 44.1 ให้เห็นจริง** ที่บอกว่า `$ty:ty` ของ `macro_rules!` เป็น "กล่องดำ" ที่มองเข้าไปข้างในไม่ได้ แต่ `syn::Type` ทำได้ — โค้ดนี้ (โค้ด Rust ธรรมดา ไม่ใช่ proc macro แต่ใช้ `syn` ตัวเดียวกัน) แสดงให้เห็นว่าเราสามารถเขียนฟังก์ชันที่ **"มองเข้าไปข้างใน" type** เพื่อเช็คว่ามันเป็น `Option<T>` หรือไม่ แล้วดึง `T` ข้างในออกมาได้จริง:

```rust
use syn::{Type, GenericArgument, PathArguments};

// ฟังก์ชันธรรมดา (ไม่ใช่ proc macro) ที่สาธิตว่า syn::Type "มองเข้าไปข้างใน" ได้จริง
// ต่างจาก $ty:ty ของ macro_rules! ที่เป็นกล่องดำ (ตามที่อธิบายในหัวข้อ 44.1)
fn is_option_of(ty: &Type) -> Option<&Type> {
    // ขั้นที่ 1: type ต้องเป็นรูปแบบ "path" (เช่น Option<String>, std::collections::HashMap<K, V>)
    if let Type::Path(type_path) = ty {
        // ขั้นที่ 2: ดู segment สุดท้ายของ path (สำหรับ Option<String> segment สุดท้ายคือ "Option")
        let segment = type_path.path.segments.last()?;
        if segment.ident != "Option" {
            return None;
        }
        // ขั้นที่ 3: ดึง generic argument ตัวแรกออกมา (ตัว T ข้างใน Option<T>)
        if let PathArguments::AngleBracketed(args) = &segment.arguments {
            if let Some(GenericArgument::Type(inner_ty)) = args.args.first() {
                return Some(inner_ty);
            }
        }
    }
    None
}

fn main() {
    let ty1: Type = syn::parse_str("Option<String>").unwrap();
    let ty2: Type = syn::parse_str("u32").unwrap();

    match is_option_of(&ty1) {
        Some(inner) => println!("ty1 คือ Option ของ: {}", quote::quote!(#inner)),
        None => println!("ty1 ไม่ใช่ Option"),
    }

    match is_option_of(&ty2) {
        Some(inner) => println!("ty2 คือ Option ของ: {}", quote::quote!(#inner)),
        None => println!("ty2 ไม่ใช่ Option"),
    }
}
```

ผลลัพธ์จริง (ตรวจสอบแล้ว):

```
ty1 คือ Option ของ: String
ty2 ไม่ใช่ Option
```

สังเกตกลไกที่เกิดขึ้น: `syn::Type` เป็น `enum` ที่มีหลาย variant (`Type::Path`, `Type::Reference`, `Type::Tuple`, `Type::Array`, ...) — `Type::Path` เก็บ `path.segments` เป็นลำดับของชื่อ (เหมือน `Option`, หรือ `std`/`collections`/`HashMap` ถ้าเขียนแบบเต็ม) แต่ละ segment มี `.arguments` ที่เก็บ generic argument (`PathArguments::AngleBracketed` สำหรับ `<...>`) — เราไล่ pattern match ลงไปทีละชั้นด้วย `if let` ธรรมดา (Part 10) จนดึง type ข้างในออกมาได้ **นี่คือสิ่งที่ `$ty:ty` ของ `macro_rules!` ทำไม่ได้เลยในหลักการ** เพราะ fragment specifier แค่ "จับคู่และเก็บ" ทั้งก้อนไว้ ไม่มีทาง pattern-match ลงไปในโครงสร้างภายในของสิ่งที่จับคู่ได้แล้ว ในขณะที่ `syn::Type` เป็น AST ที่มีโครงสร้างสมบูรณ์ให้ `match`/`if let` ได้เหมือน enum ปกติทุกประการ (ความสามารถนี้จะถูกนำไปใช้จริงในการ generate โค้ดแบบ "ข้าม field ที่เป็น `Option`" ใน Part 45)

**หมายเหตุเชื่อมกับ Part 36 เรื่อง hygiene:** Part 36 พิสูจน์ให้เห็นว่า `macro_rules!` เป็น **hygienic** — ตัวแปรที่ macro ประกาศขึ้นมาเองจะไม่ชนกับตัวแปรของผู้เรียกใช้ แม้ชื่อจะซ้ำกันก็ตาม เพราะ compiler ติดตาม "ที่มา" (site) ของแต่ละ identifier แยกกันโดยอัตโนมัติ proc macro ที่ generate โค้ดผ่าน `quote!{}` **ไม่ได้รับ hygiene ระดับเดียวกันแบบอัตโนมัติ** — ค่าเริ่มต้นของ `quote!` ใช้ `Span::call_site()` สำหรับ identifier ที่มัน generate ขึ้น ซึ่งหมายความว่า identifier เหล่านั้นจะ**ผูกกับ scope ของจุดที่ macro ถูกเรียกใช้** (call site) ไม่ใช่ผูกกับ "โลกของตัว macro เอง" แบบ def-site hygiene ของ `macro_rules!` ในทางปฏิบัติ นี่มักไม่เป็นปัญหาสำหรับ derive macro ที่ generate `impl` block ใหม่ทั้งก้อน (เพราะ body ของ method ที่ generate เป็น scope ใหม่ของตัวเองอยู่แล้ว ไม่ทับซ้อนกับโค้ดของผู้ใช้) แต่เป็นสิ่งที่ต้องระมัดระวังมากขึ้นถ้าคุณเขียน **attribute macro ที่ต้องผสมโค้ดที่ generate เข้ากับ body เดิมของผู้ใช้โดยตรง** (แบบที่ `#[log_call]` ในหัวข้อ 44.2 ทำ) — เป็นอีกเหตุผลที่ตอกย้ำว่า attribute macro ซับซ้อนกว่า derive macro จริง ๆ ไม่ใช่แค่เรื่อง signature

### 44.6 ตัวอย่างเต็มรูปแบบ: เขียน `#[derive(Describe)]` ตั้งแต่ต้นจนจบ

ถึงเวลาลงมือเขียนจริง เราจะสร้าง derive macro ชื่อ `Describe` ที่ generate เมธอด `describe()` ให้ struct ใด ๆ โดยอัตโนมัติ — เมธอดนี้จะพิมพ์ชื่อ struct และชื่อของทุก field (ตัวอย่าง "derive macro 101" ที่พบได้บ่อยที่สุดตอนเริ่มเรียน proc macro เพราะมันครอบคลุมทุกขั้นตอนสำคัญโดยไม่ซับซ้อนเกินไป)

**โครงสร้างโปรเจกต์ที่ต้องใช้ (สำคัญมาก เพราะสอดคล้องกับกฎในหัวข้อ 44.3):**

```
proc_macro_demo/                  <-- Cargo workspace
├── Cargo.toml                    <-- ประกาศ [workspace] members
├── describe_derive/              <-- proc-macro crate (ประกาศ proc-macro = true)
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs                <-- โค้ด proc macro ทั้งหมดอยู่ที่นี่
└── describe_consumer/            <-- binary crate ธรรมดาที่ "ใช้" macro
    ├── Cargo.toml
    └── src/
        └── main.rs
```

**สังเกตว่าเราต้องมี 2 crate แยกกันเสมอ** — `describe_derive` (ที่นิยามตัว macro) และ `describe_consumer` (ที่นำ macro มาใช้กับ struct ของตัวเอง) นี่ไม่ใช่ทางเลือก แต่เป็นผลจากกฎในหัวข้อ 44.3: `describe_derive` ต้องคอมไพล์เสร็จสมบูรณ์ก่อนที่ `describe_consumer` จะเริ่ม derive ได้ — ต่อให้ทั้งสองอยู่ใน workspace เดียวกัน `cargo` ก็จะจัดลำดับ build ให้ `describe_derive` compile ก่อนโดยอัตโนมัติ (เพราะมันเห็นความสัมพันธ์ dependency ใน `Cargo.toml` ของ `describe_consumer`)

**ขั้นที่ 1: `describe_derive/Cargo.toml`**

```toml
[package]
name = "describe_derive"
version = "0.1.0"
edition = "2021"

[lib]
proc-macro = true

[dependencies]
syn = { version = "2", features = ["full"] }
quote = "1"
proc-macro2 = "1"
```

หมายเหตุเรื่อง `features = ["full"]` ของ `syn`: `syn` ออกแบบให้ compile เร็วขึ้นโดย**เปิดใช้แค่บางส่วนของ grammar เป็น default** (เช่น รองรับแค่ expression บางแบบ) ฟีเจอร์ `"full"` เปิดการรองรับไวยากรณ์ Rust แบบเต็ม (รวม item ระดับ struct/enum/fn ทั้งหมด) ซึ่ง derive macro ที่ทำงานกับ `DeriveInput` **ต้องเปิดฟีเจอร์นี้เสมอ** ไม่งั้นจะ parse struct/enum ไม่ได้ครบ

**ขั้นที่ 2: `describe_derive/src/lib.rs` — ตัวมาโครจริง**

ก่อนเขียนโค้ดจริง มีการตัดสินใจเชิงสถาปัตยกรรมหนึ่งอย่างที่นักพัฒนา proc macro ที่มีประสบการณ์มักทำเสมอ นั่นคือ **แยก entry point ที่ "บาง" (thin wrapper) ออกจาก logic จริงที่ "หนา"**: ฟังก์ชัน entry point (ที่มี `#[proc_macro_derive(...)]`) ทำหน้าที่แค่แปลง `TokenStream` ↔ `TokenStream` (คือ parse input แล้วแปลง output กลับ) ส่วน**ตรรกะทั้งหมด**ที่วิเคราะห์ AST และ generate โค้ดจะอยู่ในฟังก์ชันแยกที่ทำงานบน `proc_macro2::TokenStream` ล้วน ๆ (ไม่แตะ `proc_macro::TokenStream` เลย) เหตุผลจะอธิบายละเอียดหลังโค้ด — สรุปสั้น ๆ ก่อนคือ **เพื่อให้เขียน `#[test]` ทดสอบ logic ได้โดยไม่ต้องพึ่งกระบวนการ compile จริงของ crate อื่น**

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, Data, DeriveInput, Fields};

/// entry point ของ proc macro เอง — "บาง" ที่สุด แค่แปลง TokenStream กับ TokenStream2
/// ห่อ logic จริงที่อยู่ใน expand_describe() ซึ่งทำงานบน proc_macro2::TokenStream ล้วน ๆ
#[proc_macro_derive(Describe)]
pub fn derive_describe(input: TokenStream) -> TokenStream {
    // ขั้นที่ 1: parse TokenStream ดิบ ให้กลายเป็น syn::DeriveInput (AST ที่มีโครงสร้าง)
    let input = parse_macro_input!(input as DeriveInput);

    // ขั้นที่ 2: ส่งต่อให้ฟังก์ชัน logic จริง แล้วแปลงผลลัพธ์กลับเป็น proc_macro::TokenStream
    expand_describe(input).into()
}

/// ฟังก์ชันที่ทำงานจริงทั้งหมด — รับ DeriveInput (จาก syn) คืน proc_macro2::TokenStream
/// ไม่แตะ proc_macro::TokenStream เลยแม้แต่บรรทัดเดียว ทำให้เรียกจาก #[test] ธรรมดาได้
fn expand_describe(input: DeriveInput) -> proc_macro2::TokenStream {
    // ขั้นที่ 3: ดึงชื่อ struct ออกมา (เป็น syn::Ident แล้วแปลงเป็น String สำหรับ print)
    let struct_name = &input.ident;
    let struct_name_str = struct_name.to_string();

    // ขั้นที่ 4: ดึงชื่อ field ทั้งหมด — pattern match บน Data/Fields ตามที่เรียนในหัวข้อ 44.5
    let field_names: Vec<String> = match &input.data {
        Data::Struct(data_struct) => match &data_struct.fields {
            Fields::Named(fields) => fields
                .named
                .iter()
                .map(|f| f.ident.as_ref().unwrap().to_string())
                .collect(),
            _ => {
                // struct ที่ไม่มี named field (tuple struct/unit struct) — ปฏิเสธอย่างสุภาพ
                return syn::Error::new_spanned(
                    struct_name,
                    "Describe รองรับเฉพาะ struct ที่มี named field เท่านั้น",
                )
                .to_compile_error();
            }
        },
        _ => {
            // ไม่ใช่ struct เลย (เป็น enum หรือ union) — ปฏิเสธอย่างสุภาพเช่นกัน
            return syn::Error::new_spanned(
                struct_name,
                "Describe ใช้ได้กับ struct เท่านั้น (ไม่รองรับ enum/union)",
            )
            .to_compile_error();
        }
    };

    let field_count = field_names.len();

    // ขั้นที่ 5: สร้างโค้ดใหม่ด้วย quote! — #variable คือการ interpolate ค่าเข้าไปในโค้ดต้นแบบ
    // #(#field_names),* คือ repetition ของ quote! (เทียบเท่า $(...),*  ของ macro_rules! จาก Part 36)
    quote! {
        impl #struct_name {
            pub fn describe(&self) {
                println!("struct ชื่อ: {}", #struct_name_str);
                println!("มี field ทั้งหมด {} ตัว:", #field_count);
                let field_list: [&str; #field_count] = [#(#field_names),*];
                for name in field_list {
                    println!("  - field: {}", name);
                }
            }
        }
    }
}
```

**อธิบายโค้ดทีละส่วนอย่างละเอียด:**

- **`#[proc_macro_derive(Describe)]`** — attribute ที่บอก compiler ว่าฟังก์ชันข้างล่างนี้คือ implementation ของ derive macro ชื่อ `Describe` (ชื่อในวงเล็บนี้คือชื่อที่ผู้ใช้จะพิมพ์ใน `#[derive(Describe)]` — ไม่จำเป็นต้องตรงกับชื่อฟังก์ชัน Rust ที่ implement มันเลย สังเกตว่าฟังก์ชันเราชื่อ `derive_describe` แต่ macro ชื่อ `Describe`)
- **`fn derive_describe(input: TokenStream) -> TokenStream`** — นี่คือ entry point signature มาตรฐานของ derive macro ทุกตัว: รับ `TokenStream` เดียว (คือ token ของ item ที่ derive อยู่) คืน `TokenStream` เดียว (คือโค้ดที่จะถูกเพิ่มเข้ามา) — สังเกตว่า `TokenStream` ตรงนี้คือ `proc_macro::TokenStream` (จาก `use proc_macro::TokenStream;` ด้านบน) ไม่ใช่ `proc_macro2::TokenStream`
- **`parse_macro_input!(input as DeriveInput)`** — ดังที่อธิบายในหัวข้อ 44.5 บรรทัดนี้ทำหน้าที่ parse และจะ `return` ออกจากฟังก์ชันพร้อม compile error ให้เองถ้า parse ไม่ผ่าน
- **`expand_describe(input).into()`** — จุดที่ entry point "บาง ๆ" ส่งไม้ต่อให้ฟังก์ชัน logic จริง แล้วแปลงผลลัพธ์ (`proc_macro2::TokenStream`) กลับเป็น `proc_macro::TokenStream` ด้วย `.into()` เพียงครั้งเดียว
- **`fn expand_describe(input: DeriveInput) -> proc_macro2::TokenStream`** — ฟังก์ชันนี้คือ **โค้ด Rust ธรรมดาที่สุด** ไม่มีอะไรเกี่ยวกับ "compiler plugin" หรือ `proc_macro` เลยแม้แต่นิดเดียว มันรับ `DeriveInput` (ที่ parse ไว้แล้ว) แล้วคืน `proc_macro2::TokenStream` — เพราะมันเป็นฟังก์ชันธรรมดา คุณสามารถเรียกมันจาก `#[test]` ได้ตรง ๆ (จะเห็นในหัวข้อถัดไป)
- **`match &input.data { Data::Struct(...) => ..., _ => ... }`** — จุดที่เราปฏิเสธ input ที่ไม่ใช่ struct-with-named-fields อย่างชัดเจน โดยใช้ `syn::Error`/`to_compile_error()` (จะอธิบายลึกในหัวข้อ 44.9) แทนการ `panic!` หรือ `.unwrap()` มือ ๆ — สังเกตว่าตรงนี้เราเรียก `.to_compile_error()` **โดยไม่ใส่ `.into()` ต่อ** เพราะฟังก์ชันนี้ยังคืน `proc_macro2::TokenStream` อยู่ (การแปลงเป็น `proc_macro::TokenStream` เกิดขึ้นครั้งเดียวตรงจุดเดียวใน `derive_describe`)
- **`quote! { ... }`** — ส่วนที่น่าตื่นเต้นที่สุด: เราเขียนโค้ด Rust ที่**อยาก generate**ตรง ๆ ในรูปแบบที่อ่านออกเป็นโค้ดจริง (`impl #struct_name { ... }`) แล้วใช้ `#variable` เพื่อ**แทรกค่า** เข้าไป — `#struct_name` ถูกแทนที่ด้วยชื่อ struct จริง (เช่น `Customer`), `#struct_name_str` ถูกแทนที่ด้วย string literal `"Customer"`, `#field_count` ถูกแทนที่ด้วยตัวเลขจริง (เช่น `3`), และ `#(#field_names),*` ขยายเป็นรายการ string literal คั่นด้วย comma (เช่น `"name", "age", "email"`) ผลลัพธ์ของ `quote!{}` คือ `proc_macro2::TokenStream` เสมอ ตรงกับ return type ของฟังก์ชัน จึงไม่ต้องแปลงอะไรเพิ่ม

**ขั้นที่ 3: `describe_consumer/Cargo.toml`**

```toml
[package]
name = "describe_consumer"
version = "0.1.0"
edition = "2021"

[dependencies]
describe_derive = { path = "../describe_derive" }
```

สังเกตว่า `describe_consumer` เป็น **binary crate ธรรมดา** ไม่มีอะไรพิเศษเลย มันแค่ depend on `describe_derive` เหมือน dependency ทั่วไป (ผ่าน `path = "..."` เพราะยังไม่ได้ publish ขึ้น crates.io)

**ขั้นที่ 4: `describe_consumer/src/main.rs`**

```rust
use describe_derive::Describe;

#[derive(Describe)]
struct Customer {
    name: String,
    age: u32,
    email: String,
}

#[derive(Describe)]
struct Product {
    sku: String,
    price: f64,
}

fn main() {
    let c = Customer {
        name: "สมชาย".to_string(),
        age: 30,
        email: "somchai@example.com".to_string(),
    };
    c.describe();

    println!("---");

    let p = Product {
        sku: "SKU-001".to_string(),
        price: 199.0,
    };
    p.describe();
}
```

สังเกตว่า `Customer` และ `Product` เป็น **struct ปกติที่สุด** — ไม่มีการห่อด้วย macro call แบบที่เราพยายามทำในหัวข้อ 44.1 เลย มันคือ struct ธรรมดาที่ติด `#[derive(Describe)]` ไว้ด้านบน**เหมือนกับ `#[derive(Debug, Clone)]` ทุกประการ** — นี่คือ ergonomic ที่แท้จริงของ derive macro ที่ `macro_rules!` ให้ไม่ได้

**ผลลัพธ์จริงจากการรัน `cargo run -p describe_consumer` (ตรวจสอบแล้วว่าคอมไพล์และรันผ่านจริง):**

```
struct ชื่อ: Customer
มี field ทั้งหมด 3 ตัว:
  - field: name
  - field: age
  - field: email
---
struct ชื่อ: Product
มี field ทั้งหมด 2 ตัว:
  - field: sku
  - field: price
```

สังเกตสิ่งสำคัญ: เราเขียน `describe_derive` แค่**ครั้งเดียว** แต่มันทำงานถูกต้องกับทั้ง `Customer` (3 field) และ `Product` (2 field) โดยอัตโนมัติ — โดยที่ตัว macro **ไม่รู้ล่วงหน้า**ว่าจะถูกใช้กับ struct ไหนบ้าง มันอ่านโครงสร้างของ struct ที่ derive อยู่**ตอน compile time ของ `describe_consumer`** แล้ว generate โค้ดที่เหมาะกับ struct นั้นโดยเฉพาะ นี่คือพลังของการที่ proc macro เห็น**โครงสร้างจริง**ของ item ผ่าน AST ของ `syn` — ไม่ใช่แค่ token pattern ตายตัวแบบ `macro_rules!`

### 44.6.1 ทดสอบ Logic ของ Proc Macro ด้วย `#[test]` ธรรมดา

เพราะเราแยก `expand_describe()` ออกมาเป็นฟังก์ชันที่ทำงานบน `proc_macro2::TokenStream` ล้วน ๆ (ไม่แตะ `proc_macro::TokenStream` เลย) เราสามารถเขียน `#[test]` ทดสอบมันได้**ตรง ๆ ในไฟล์เดียวกัน** เหมือนทดสอบฟังก์ชันธรรมดาทั่วไปที่คุณคุ้นเคยมาตั้งแต่ Part 32 — โดยไม่ต้องพึ่ง crate consumer แยกต่างหากเลย:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn expands_struct_with_two_fields() {
        // สร้าง DeriveInput จากซอร์สโค้ดจริงด้วย syn::parse_str (ไม่ต้อง compile crate อื่น)
        let input: DeriveInput = syn::parse_str(
            "struct Customer { name: String, age: u32 }"
        )
        .unwrap();

        let expanded = expand_describe(input);
        let code = expanded.to_string();

        // ตรวจสอบว่าโค้ดที่ expand ออกมา "มีชิ้นส่วน" ที่ควรมี (ไม่ตรวจ byte-for-byte
        // เพราะ quote! อาจจัดช่องว่าง/รูปแบบต่างกันได้ในแต่ละเวอร์ชัน)
        assert!(code.contains("impl Customer"));
        assert!(code.contains("describe"));
        assert!(code.contains("\"name\""));
        assert!(code.contains("\"age\""));
    }

    #[test]
    fn rejects_enum_with_compile_error() {
        let input: DeriveInput = syn::parse_str(
            "enum Status { Active, Inactive }"
        )
        .unwrap();

        let expanded = expand_describe(input);
        let code = expanded.to_string();

        // เมื่อปฏิเสธ input ที่ผิดเงื่อนไข ผลลัพธ์ต้องมี compile_error! อยู่ข้างใน
        // (นี่คือสิ่งที่ to_compile_error() generate ออกมา ตามหัวข้อ 44.9)
        assert!(code.contains("compile_error"));
    }
}
```

จุดที่น่าสนใจที่สุดคือ **`syn::parse_str::<DeriveInput>("struct Customer { ... }")`** — เราสร้าง `DeriveInput` ขึ้นมาจาก**string ของซอร์สโค้ด Rust** ตรง ๆ โดยไม่ต้องผ่านกระบวนการ `cargo build` ของ crate อื่นเลยแม้แต่นิดเดียว (ต่างจากการทดสอบผ่าน `describe_consumer` ที่ต้อง compile ทั้ง crate จริง ๆ ถึงจะเห็นผล) `expanded.to_string()` แปลง `TokenStream` กลับเป็น string ของโค้ด Rust (ในรูปแบบที่ไม่สวยนัก ไม่มีการจัดบรรทัด/เว้นวรรคแบบที่มนุษย์อ่านง่าย) ทำให้เราใช้ `.contains(...)` ตรวจสอบว่า**ชิ้นส่วนที่ควรมี**ปรากฏอยู่จริงในโค้ดที่ expand ออกมาได้

รันคำสั่ง `cargo test -p describe_derive` ผลลัพธ์จริง (ตรวจสอบแล้ว):

```
running 2 tests
test tests::expands_struct_with_two_fields ... ok
test tests::rejects_enum_with_compile_error ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**นี่คือเหตุผลแท้จริงที่ ecosystem แยก `proc-macro2` ออกมาต่างหาก** (ตามที่กล่าวถึงสั้น ๆ ในหัวข้อ 44.4): ถ้า `syn`/`quote` ทำงานบน `proc_macro::TokenStream` ตรง ๆ (ไม่มี `proc-macro2` มาห่อ) คุณจะเรียกใช้มัน**นอก context ของการเป็น proc-macro compiler plugin ไม่ได้เลย** เพราะ `proc_macro::TokenStream` ผูกติดกับ compiler runtime ของ `rustc` ที่มีให้ใช้แค่ตอนกำลังรันเป็น plugin จริงเท่านั้น (`use proc_macro::TokenStream;` ใน crate ที่ไม่ใช่ `proc-macro = true` จะ error ทันทีตามที่เห็นในกับดักข้อ 1) การที่ `proc-macro2` เป็น "สำเนา" ของ `TokenStream` ที่ใช้งานได้เป็น library ธรรมดา ทำให้ทั้ง logic การ parse/generate ของคุณกลายเป็น**โค้ด Rust ธรรมดาที่ทดสอบได้แบบมาตรฐาน** — pattern "entry point บาง + logic หนาที่ทดสอบได้" แบบนี้คือสิ่งที่ proc macro crate คุณภาพดีในโลกจริงใช้กันเป็นธรรมเนียมทั่วไป (ลองเปิดดู source code ของ `serde_derive`, `thiserror-impl`, หรือ crate อื่น ๆ ที่มี proc macro จะเห็น pattern เดียวกันนี้ซ้ำ ๆ)

### 44.7 Function-like Proc Macro: `make_answer!()`

ทีนี้มาดู entry point ของ function-like macro ซึ่ง**เรียบง่ายกว่า derive macro** เพราะไม่ต้องยุ่งกับการ parse struct/enum ที่มีโครงสร้างซับซ้อน — มันรับ token อะไรก็ได้ตามที่คุณออกแบบเอง (บางตัวอาจไม่สนใจ input เลยด้วยซ้ำ)

**`answer_macro/Cargo.toml`:**

```toml
[package]
name = "answer_macro"
version = "0.1.0"
edition = "2021"

[lib]
proc-macro = true

[dependencies]
quote = "1"
proc-macro2 = "1"
```

สังเกตว่าตัวอย่างนี้**ไม่ต้องใช้ `syn` เลย** เพราะเราไม่ได้ parse โครงสร้างซับซ้อนอะไรจาก input — เป็นตัวอย่างที่ดีที่แสดงว่า `syn` ไม่ใช่สิ่งบังคับของ proc macro ทุกตัว มันจำเป็นก็ต่อเมื่อคุณต้องการ**วิเคราะห์โครงสร้าง**ของ token ที่ได้รับมา

**`answer_macro/src/lib.rs`:**

```rust
use proc_macro::TokenStream;
use quote::quote;

/// function-like proc macro — เรียกใช้แบบ make_answer!()
/// สังเกตว่า signature เหมือน derive macro เป๊ะ: fn(TokenStream) -> TokenStream
/// ต่างกันแค่ attribute (#[proc_macro] ไม่ใช่ #[proc_macro_derive(...)])
/// และในตัวอย่างนี้เราไม่สนใจ input เลย (ตั้งชื่อ _input เพื่อไม่ให้ compiler เตือน unused)
#[proc_macro]
pub fn make_answer(_input: TokenStream) -> TokenStream {
    let expanded = quote! {
        fn answer() -> u32 {
            42
        }
    };
    expanded.into()
}
```

**`answer_consumer/src/main.rs`:**

```rust
use answer_macro::make_answer;

make_answer!();

fn main() {
    println!("คำตอบคือ: {}", answer());
}
```

**ผลลัพธ์จริง (ตรวจสอบแล้วว่าคอมไพล์และรันผ่านจริงด้วย `cargo run -p answer_consumer`):**

```
คำตอบคือ: 42
```

สังเกตว่า `make_answer!()` **ไม่ได้คืนค่า 42 ตรง ๆ** แต่มัน generate **นิยามฟังก์ชันใหม่** ชื่อ `answer()` เข้ามาในโค้ด — เมื่อขยายแล้ว โค้ดจะเทียบเท่ากับ:

```rust
fn answer() -> u32 {
    42
}

fn main() {
    println!("คำตอบคือ: {}", answer());
}
```

นี่คือตัวอย่างที่ง่ายที่สุดเท่าที่จะเป็นไปได้ของ function-like proc macro โดยเจตนา เพื่อให้เห็น**ความต่างของ entry point** เมื่อเทียบกับ derive macro อย่างชัดเจน: ไม่มีขั้นตอน parse `DeriveInput`, ไม่มีการดึง field, ไม่ต้อง `syn` เลยก็ยังทำงานได้ — ในสถานการณ์จริงที่ input ของ function-like macro ซับซ้อนกว่านี้ (เช่น `sqlx::query!("SELECT ...")` ที่ต้อง parse SQL string) คุณก็จะดึง `syn`/`quote` เข้ามาช่วยแบบเดียวกับ derive macro

### 44.8 Debugging Proc Macros: `cargo expand` และ `eprintln!`

Debug proc macro **ยากกว่า** debug โค้ดปกติมาก และยากกว่าแม้แต่ debug `macro_rules!` (ที่ Part 36 ก็บอกไว้แล้วว่ายากกว่าฟังก์ชันธรรมดา) เหตุผลคือ:

**ปัญหาพื้นฐาน: proc macro รันตอน compile time ของ "คนอื่น"** เมื่อคุณเขียน `describe_derive` แล้วมีคนอื่น (หรือ `describe_consumer` ในตัวอย่างเรา) นำไปใช้ — โค้ดของคุณจะรันตอนที่**เขา**กำลัง compile โปรเจกต์ของ**เขา** ไม่ใช่ตอนที่คุณกำลังรันโปรแกรมของคุณเองแบบปกติ ถ้าคุณอยากรู้ว่า "ตัวแปร `field_names` มีค่าอะไรตอนนี้" คุณจะ `println!` ธรรมดาไม่ได้ตรง ๆ เพราะ `println!` ภายในโค้ดของ proc macro **จะพิมพ์ไปที่ stdout ของกระบวนการ `rustc`/`cargo build` ตอนกำลัง compile** — มันจะไม่ปรากฏใน output ของโปรแกรมที่รันเสร็จแล้วเลย (ต่างจากการ debug ฟังก์ชันปกติที่ `println!` เห็นผลตรง ๆ ตอนโปรแกรมรัน)

**เทคนิคหยาบแต่ใช้งานได้จริง: `eprintln!` ระหว่างการ expand** เนื่องจาก proc macro รันเป็นส่วนหนึ่งของกระบวนการ `cargo build`, การเขียน `eprintln!` (พิมพ์ไปที่ stderr) ไว้ในโค้ดของ proc macro **จะปรากฏบน terminal ตอนรัน `cargo build`/`cargo check` จริง** (เพราะ stderr ของ process ที่ทำ compile จะถูกส่งต่อมาที่ terminal ของผู้ใช้ที่รัน `cargo build`) ตัวอย่าง:

```rust
#[proc_macro_derive(Describe)]
pub fn derive_describe(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    let struct_name = &input.ident;

    // เทคนิค debug แบบหยาบ: eprintln! จะโชว์ตอน cargo build (ไม่ใช่ตอนโปรแกรมรัน)
    eprintln!("[describe_derive] กำลัง derive struct ชื่อ: {}", struct_name);

    // ... โค้ดที่เหลือเหมือนเดิม
    # unimplemented!()
}
```

เมื่อรัน `cargo build -p describe_consumer` คุณจะเห็นบรรทัด `[describe_derive] กำลัง derive struct ชื่อ: Customer` และ `[describe_derive] กำลัง derive struct ชื่อ: Product` ปรากฏบน terminal **ระหว่าง**กระบวนการ compile (ก่อนที่จะมี "Finished" หรือ error ใด ๆ) — นี่คือวิธี debug ที่หยาบมาก (ไม่มี breakpoint, ไม่มี step-through แบบ debugger จริง) แต่เป็นสิ่งที่นักพัฒนา proc macro ใช้กันจริงในทางปฏิบัติ เพราะการต่อ debugger เข้ากับกระบวนการ `rustc` เองที่กำลังรัน `describe_derive` เป็น plugin นั้นยุ่งยากกว่ามากในสถานการณ์ทั่วไป

**เครื่องมือที่สำคัญกว่า: `cargo expand`** จาก Part 36 คุณได้รู้จัก `cargo expand` มาแล้วในบริบทของ `macro_rules!` (ติดตั้งด้วย `cargo install cargo-expand` แยกจาก cargo default) มันมีประโยชน์มากกว่ากับ proc macro ด้วยซ้ำ เพราะโค้ดที่ proc macro generate ออกมา**อยู่ห่างจากซอร์สโค้ดที่คุณเห็นมากกว่า** `macro_rules!` มาก (ไม่มี template ให้เดารูปร่างคร่าว ๆ ได้เหมือน `macro_rules!` — โค้ดที่ generate มาจาก logic โปรแกรมมิ่งเต็มรูปแบบ) รันคำสั่ง:

```bash
cargo expand -p describe_consumer
```

ผลลัพธ์ (แบบคร่าว ๆ) จะแสดง `describe_consumer/src/main.rs` **ฉบับเต็มหลังจาก `#[derive(Describe)]` ทุกตัวถูกขยายเสร็จแล้ว** ซึ่งจะโชว์ให้เห็น `impl Customer { pub fn describe(&self) { ... } }` และ `impl Product { pub fn describe(&self) { ... } }` ที่ macro generate ให้แบบเต็ม ๆ ตรงตามที่เราเขียนไว้ใน `quote!{}` — เมื่อ error message ของ compiler ชี้ไปที่โค้ดที่ expand แล้วจนงง (เช่น type ไม่ตรงกันในโค้ดที่ macro generate) การเปิด `cargo expand` ดูควบคู่กันคือวิธีที่เร็วที่สุดในการเข้าใจว่า macro ของคุณสร้างโค้ดผิดตรงไหน

### 44.9 Compile Error ที่มีอารยะ: `syn::Error` และ `to_compile_error()`

หัวข้อนี้สำคัญมากในเชิง**คุณภาพของ proc macro ที่เขียนดี** เทียบกับที่เขียนไม่ดี ลองดูสิ่งที่เกิดขึ้นจริงเมื่อ proc macro รายงานข้อผิดพลาดสองแบบที่ต่างกัน

**แบบที่ไม่ดี: ใช้ `panic!` (หรือ `.unwrap()` ที่ panic แทน)**

```rust
// เวอร์ชัน "ไม่ดี" — panic! แทนการคืน syn::Error
#[proc_macro_derive(PanickyValidated)]
pub fn derive_panicky(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    let struct_name = &input.ident;

    match &input.data {
        Data::Struct(_) => {
            let expanded = quote! {
                impl #struct_name {
                    pub fn is_valid(&self) -> bool { true }
                }
            };
            expanded.into()
        }
        _ => panic!("PanickyValidated ใช้ได้กับ struct เท่านั้น"),
    }
}
```

เมื่อมีคนใช้ `#[derive(PanickyValidated)]` กับ `enum` (ผิดเงื่อนไข) **ผลลัพธ์จริงที่ได้จากการคอมไพล์ (ตรวจสอบแล้ว):**

```
error: proc-macro derive panicked
 --> src/main.rs:3:10
  |
3 | #[derive(PanickyValidated)]
  |          ^^^^^^^^^^^^^^^^
  |
  = help: message: PanickyValidated ใช้ได้กับ struct เท่านั้น

error: could not compile `panicky_consumer` (bin "panicky_consumer") due to 1 previous error
```

สังเกตปัญหา 2 อย่าง: (1) error ชี้ไปที่**ชื่อ derive macro** (`PanickyValidated` ในวงเล็บ `#[derive(...)]`) ไม่ใช่ไปที่**ตัว `enum` ที่ผิดเงื่อนไขจริง ๆ** ทำให้ผู้ใช้ต้องอ่าน `help: message:` เพื่อรู้ว่าปัญหาจริงคืออะไร (2) ข้อความ `proc-macro derive panicked` เป็นคำที่บอกว่า**เกิด panic ข้างในกระบวนการ compile plugin** ซึ่งฟังดูเหมือนเป็น**บั๊กของตัว macro เอง** มากกว่าฟังดูเหมือน "คุณใช้ macro นี้ผิดวิธี" (ในกรณีที่ macro ซับซ้อนกว่านี้และ panic เกิดลึกเข้าไปในการเรียก `.unwrap()` ของ `syn` เอง ผู้ใช้อาจได้ backtrace ที่ชี้เข้าไปใน internal ของ `syn`/`proc-macro2` เต็มไปหมด อ่านไม่รู้เรื่องเลยว่าปัญหาแท้จริงอยู่ตรงไหนในโค้ดของตัวเอง)

**แบบที่ดี: ใช้ `syn::Error::new_spanned(...).to_compile_error()`**

```rust
#[proc_macro_derive(Validated)]
pub fn derive_validated(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    let struct_name = &input.ident;

    match &input.data {
        Data::Struct(data_struct) => match &data_struct.fields {
            Fields::Named(_) => {
                let expanded = quote! {
                    impl #struct_name {
                        pub fn is_valid(&self) -> bool { true }
                    }
                };
                expanded.into()
            }
            _ => syn::Error::new_spanned(
                &input.ident,
                "Validated ต้องใช้กับ struct ที่มี named field เท่านั้น (ไม่รองรับ tuple struct)",
            )
            .to_compile_error()
            .into(),
        },
        Data::Enum(_) | Data::Union(_) => syn::Error::new_spanned(
            &input.ident,
            "Validated ใช้ได้กับ struct เท่านั้น ไม่รองรับ enum หรือ union",
        )
        .to_compile_error()
        .into(),
    }
}
```

เมื่อมีคนใช้ `#[derive(Validated)]` กับ `enum` แบบเดียวกัน **ผลลัพธ์จริงที่ได้ (ตรวจสอบแล้ว):**

```
error: Validated ใช้ได้กับ struct เท่านั้น ไม่รองรับ enum หรือ union
 --> src/main.rs:4:6
  |
4 | enum Status {
  |      ^^^^^^
```

สังเกตความต่างที่ชัดเจนมาก: (1) error message เป็น**ข้อความที่เราเขียนเองตรง ๆ** ไม่มีคำว่า "panicked" หรือคำเทคนิคภายในของ compiler ปนมาเลย อ่านแล้วเข้าใจปัญหาทันที (2) หัวลูกศร `^^^^^^` ชี้ไปที่**ชื่อ `Status`** (ตัว item ที่ผิดเงื่อนไขจริง) ไม่ใช่ชี้ไปที่ `#[derive(...)]` เหมือนแบบแรก — ผู้ใช้เห็น error แล้วรู้ทันทีว่า "ปัญหาอยู่ที่ `enum Status` ของฉัน ต้องเปลี่ยนเป็น struct"

**กลไกที่ทำให้เกิดผลลัพธ์นี้:** `syn::Error::new_spanned(tokens, message)` สร้าง `syn::Error` โดยผูก**ตำแหน่ง (span)** ของ error เข้ากับ token ที่คุณระบุ (ในตัวอย่างคือ `&input.ident` ซึ่งคือชื่อ `Status` ที่มาจาก source code จริงของผู้ใช้ พร้อมข้อมูลบรรทัด/คอลัมน์ที่แม่นยำที่ compiler เก็บไว้ตั้งแต่ parse) จากนั้น `.to_compile_error()` แปลง `syn::Error` ให้กลายเป็น **`TokenStream` ที่มี macro พิเศษของ compiler ชื่อ `compile_error!`** อยู่ข้างใน (ประมาณว่า expand เป็น `compile_error!("Validated ใช้ได้กับ struct เท่านั้น...")` ที่ตำแหน่ง span ที่กำหนด) — `compile_error!` คือ macro built-in ของ Rust ที่ทำหน้าที่เดียว คือ**บอก compiler ให้สร้าง error message ตรงตามที่ระบุ ที่ตำแหน่งที่ระบุ** โดยไม่ต้อง panic ทั้งกระบวนการ compile เลย มันคือวิธี "ส่ง error กลับไปให้ compiler แสดงอย่างสุภาพ" แทนการทำให้ทั้งกระบวนการ crash

**หลักปฏิบัติที่ควรยึดถือ:** proc macro ที่เขียนดีควรเลี่ยง `.unwrap()`/`.expect()`/`panic!()` ในทุกจุดที่ input จากผู้ใช้ (ไม่ใช่ error ภายในของตัว macro เอง) อาจทำให้ logic ล้มเหลว — ให้ใช้ `syn::Error` + `to_compile_error()` แทนเสมอ เพื่อให้ผู้ใช้ macro ของคุณได้รับ experience เดียวกับ error message มาตรฐานของ Rust compiler เอง ไม่ใช่ backtrace ที่งงงวยจากภายในกระบวนการ derive

### 44.10 เขียนบ่อยแค่ไหน? Derive vs Attribute vs Function-like ในทางปฏิบัติ

ก่อนปิดบท มาตั้งความคาดหวังที่สมจริงเกี่ยวกับว่า**นักพัฒนา Rust ทั่วไป**ควรลงทุนเวลาเรียนรู้ proc macro แต่ละชนิดมากแค่ไหน:

**Derive macro — สิ่งที่คุณมีโอกาส "เขียนเอง" มากที่สุด** ถ้าคุณทำงานในทีมที่มี domain trait ของตัวเอง (เช่น trait `Validate` สำหรับตรวจสอบข้อมูล, trait `ToSql` สำหรับแปลงเป็น SQL statement, trait `Describe` แบบที่เราเขียนในบทนี้) และพบว่าต้อง implement มันซ้ำ ๆ กันสำหรับหลาย struct ด้วยรูปแบบเดิม ๆ ทุกครั้ง (คล้ายกับที่คุณเคยเบื่อการเขียน `impl Debug`/`impl Clone` มือ ๆ ก่อนจะมี `#[derive(...)]`) — นี่คือสถานการณ์ที่**สมเหตุสมผลที่สุด**สำหรับการลงทุนเขียน derive macro ของตัวเอง เพราะมันแก้ปัญหา "โค้ดซ้ำซ้อนข้าม concrete type" ได้ตรงจุด และรูปแบบที่คุณเรียนในบทนี้ (parse ด้วย `syn::parse_macro_input!`, ดึงข้อมูล, generate ด้วย `quote!`, จัดการ error ด้วย `syn::Error`) ครอบคลุม derive macro ส่วนใหญ่ที่พบในโลกจริงแล้ว

**Attribute macro และ function-like macro ที่ซับซ้อน — สิ่งที่คุณจะ "ใช้" บ่อยกว่า "เขียน" มาก** attribute macro อย่าง `#[tokio::main]` (ที่คุณจะพบเต็ม ๆ ใน Part 46) ต้องจัดการทั้งการอ่านและ**เขียนทับ**ทั้ง item ซึ่งซับซ้อนกว่ามาก (ต้องคิดเรื่อง edge case ของ syntax ของฟังก์ชันทุกรูปแบบที่เป็นไปได้ — generic, async, return type ต่าง ๆ) ส่วน function-like macro ที่ซับซ้อนอย่าง `sqlx::query!` ต้องเชื่อมต่อกับระบบภายนอก (database connection ตอน compile time) ซึ่งเป็นงานเฉพาะทางมาก นักพัฒนา Rust ส่วนใหญ่ตลอดอาชีพการทำงานจะ**ใช้** macro ประเภทนี้ (ที่คนอื่นเขียนไว้แล้วใน crate ต่าง ๆ) มากกว่าจะต้อง**เขียน**มันขึ้นมาเอง — ต่างจาก derive macro ที่มีโอกาสสูงกว่ามากที่คุณจะต้องเขียนสำหรับ domain ของทีมตัวเอง

แม้แต่ `#[test]` ที่คุณใช้มาตั้งแต่ Part 32 ก็มีความคล้ายคลึงในเชิงแนวคิดกับ attribute macro (มันเป็น attribute ที่ compiler ปฏิบัติเป็นพิเศษ แม้จะ implement ด้วยกลไกภายในของ `rustc`/`libtest` ที่ต่างจาก proc macro ทั่วไปที่ผู้ใช้เขียนเอง ไม่ใช่ proc macro แบบที่เราสร้างในบทนี้ตรง ๆ) — สิ่งที่สำคัญคือ**แนวคิด**ของ "attribute ที่แปลง/ตรวจสอบ item ที่มันติดอยู่" ที่คุณใช้อยู่แล้วโดยไม่รู้ตัว

**ข้อสรุปเชิงปฏิบัติ:** ลงทุนเวลาให้เข้าใจ **derive macro อย่างลึก** (ตามที่บทนี้สอน) เพราะมีโอกาสได้เขียนเองจริง ส่วน attribute macro และ function-like macro ที่ซับซ้อน ให้เข้าใจ**แนวคิดและ entry point ของมัน** พอที่จะอ่าน source code ของ crate ที่คุณใช้เข้าใจได้ (เช่น ตอนเจอ error แปลก ๆ จาก `#[tokio::main]` จะรู้ว่าต้องไปดูตรงไหน) มากกว่าจะต้องเขียนมันขึ้นมาเองตั้งแต่ต้น — Part 45 จะพาคุณกลับไปที่ derive macro อีกครั้งเพื่อเจาะลึกเทคนิคขั้นสูงกว่านี้ (attribute บน field แบบ `#[describe(skip)]`, การรองรับ generic, และการจัดการ enum)

**ตัวอย่างจากโลกจริง (crate ที่คุณอาจเคยใช้มาแล้ว หรือจะได้ใช้ในโมดูลถัด ๆ ไปของหลักสูตรนี้) เพื่อให้เห็นภาพว่า proc macro ทั้ง 3 ชนิดถูกใช้แก้ปัญหาอะไรจริง ๆ ในระบบนิเวศของ Rust:**

| Crate | ชนิด proc macro ที่ใช้ | แก้ปัญหาอะไร |
|---|---|---|
| `serde` (`serde_derive`) | Derive — `#[derive(Serialize, Deserialize)]` | generate โค้ดแปลง struct/enum เป็น/จาก รูปแบบข้อมูล (JSON, YAML, ...) โดยอัตโนมัติ จากโครงสร้าง struct จริง (จะเรียนใน Part 57-58) |
| `thiserror` | Derive — `#[derive(Error)]` (ที่คุณใช้มาแล้วใน Part 31) | generate `impl std::error::Error`, `impl Display`, และ `impl From` ให้ enum error ของคุณ จากโครงสร้าง variant จริง |
| `tokio` | Attribute — `#[tokio::main]` | เปลี่ยน `async fn main()` ให้กลายเป็น `fn main()` ที่สร้าง async runtime แล้วรัน future ข้างในให้ (จะเรียนเต็ม ๆ ใน Part 46/48) |
| `async-trait` | Attribute — `#[async_trait]` | แปลง `async fn` ภายใน trait ให้ compile ได้ (แก้ข้อจำกัดของภาษาที่ trait ไม่รองรับ `async fn` ตรง ๆ ในบางเวอร์ชันของ Rust) |
| `sqlx` | Function-like — `sqlx::query!("SELECT ...")` | parse SQL string แล้ว**เชื่อมต่อ database จริงตอน compile time**เพื่อตรวจสอบว่า query ถูกต้องและ type ของ column ตรงกับที่ใช้ในโค้ด |
| `clap` (`clap_derive`) | Derive — `#[derive(Parser)]` | generate CLI argument parser ทั้งหมดจากโครงสร้าง struct ที่นิยาม field ไว้ (จะเรียนใน Part 59) |

สังเกตว่า derive macro ครองสัดส่วนมากที่สุดในตาราง — ตอกย้ำข้อสรุปที่บอกไว้ข้างบนว่ามันคือชนิดที่พบเจอ (และมีโอกาสเขียนเอง) มากที่สุดในทางปฏิบัติ

### 44.11 สรุปอ้างอิงเร็ว (Quick Reference)

ก่อนไปหัวข้อกับดัก มาสรุป "checklist" ที่ใช้อ้างอิงเร็วตอนเริ่มเขียน proc macro ใหม่ทุกครั้ง:

**`Cargo.toml` ของ proc-macro crate ต้องมีเสมอ:**

```toml
[lib]
proc-macro = true

[dependencies]
syn = { version = "2", features = ["full"] }  # "full" จำเป็นถ้าต้อง parse ItemFn/ItemImpl/ฯลฯ
quote = "1"
proc-macro2 = "1"                              # จำเป็นถ้าต้องการแยก logic ไปทดสอบแบบ 44.6.1
```

**Attribute และ signature ของแต่ละชนิด:**

| ชนิด | Attribute ที่แปะบนฟังก์ชัน | Signature ของฟังก์ชัน | เรียกใช้แบบไหน |
|---|---|---|---|
| Derive | `#[proc_macro_derive(TraitName)]` | `fn(TokenStream) -> TokenStream` | `#[derive(TraitName)]` |
| Attribute | `#[proc_macro_attribute]` | `fn(TokenStream, TokenStream) -> TokenStream` | `#[macro_name]` หรือ `#[macro_name(args)]` |
| Function-like | `#[proc_macro]` | `fn(TokenStream) -> TokenStream` | `macro_name!(...)` |

**ลำดับขั้นตอนมาตรฐานภายในฟังก์ชัน (สำหรับ derive/function-like):**

1. `parse_macro_input!(input as ประเภทที่ต้องการ)` — แปลง `TokenStream` ดิบเป็น AST ของ `syn`
2. วิเคราะห์/ดึงข้อมูลจาก AST ด้วยโค้ด Rust ธรรมดา (`match`, `if let`, `.iter()`)
3. ถ้าพบ input ที่ไม่ถูกต้อง → `return syn::Error::new_spanned(...).to_compile_error()` (แปลงเป็น `proc_macro::TokenStream` ด้วย `.into()` ถ้าฟังก์ชันนี้คือ entry point เอง)
4. สร้างโค้ดใหม่ด้วย `quote! { ... }` โดย interpolate ค่าผ่าน `#variable` และ repeat ด้วย `#(...)* `
5. `.into()` แปลง `proc_macro2::TokenStream` กลับเป็น `proc_macro::TokenStream` ก่อนคืนค่า (ครั้งเดียวที่จุดที่เป็น entry point จริง ๆ)

**เครื่องมือ debug ที่ควรมีติดตัว:** `cargo expand` (ติดตั้งด้วย `cargo install cargo-expand`) และ `eprintln!` ในโค้ด macro (แสดงตอน `cargo build`/`cargo check` ไม่ใช่ตอนโปรแกรมรัน)

## กับดักที่พบบ่อย (Common Pitfalls)

**1. ลืมประกาศ `proc-macro = true` ใน `Cargo.toml`**

```toml
# Cargo.toml ของ describe_derive — ลืมส่วน [lib] ทั้งหมด
[package]
name = "bad_no_flag"
version = "0.1.0"
edition = "2021"

[dependencies]
syn = { version = "2", features = ["full"] }
quote = "1"
```

```rust
use proc_macro::TokenStream;   // <-- ใช้ crate proc_macro ไม่ได้ถ้าไม่ตั้ง proc-macro = true
use syn::{parse_macro_input, DeriveInput};

#[proc_macro_derive(Describe)]
pub fn derive_describe(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    // ...
    # unimplemented!()
}
```

Error จริงที่ได้ (ตรวจสอบแล้ว):

```
error: the `#[proc_macro_derive]` attribute is only usable with crates of the `proc-macro` crate type
 --> src/lib.rs:5:1
  |
5 | #[proc_macro_derive(Describe)]
  | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

error[E0432]: unresolved import `proc_macro`
 --> src/lib.rs:1:5
  |
1 | use proc_macro::TokenStream;
  |     ^^^^^^^^^^ use of unresolved module or unlinked crate `proc_macro`
  |
  = help: if you wanted to use a crate named `proc_macro`, use `cargo add proc_macro` to add it to your `Cargo.toml`

For more information about this error, try `rustc --explain E0432`.
```

**สาเหตุ:** ดังที่อธิบายในหัวข้อ 44.3 — crate `proc_macro` (ที่ให้ type `TokenStream` และ attribute `#[proc_macro_derive]`) เป็น crate พิเศษที่ compiler เตรียมให้ **เฉพาะ crate ที่ประกาศ `proc-macro = true` เท่านั้น** ถึงจะเชื่อมต่อ (link) กับมันได้ ถ้า crate ไม่ได้ประกาศค่านี้ มันจะถูกปฏิบัติเป็น library ธรรมดา ซึ่งไม่มีสิทธิ์เข้าถึง compiler internals แบบนี้ **วิธีแก้:** เพิ่ม `[lib]` section พร้อม `proc-macro = true` ใน `Cargo.toml` เสมอสำหรับ crate ที่มีเป้าหมายเป็น proc macro

---

**2. ลืม `.into()` ตอนแปลง `proc_macro2::TokenStream` กลับเป็น `proc_macro::TokenStream`**

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, DeriveInput};

#[proc_macro_derive(Describe)]
pub fn derive_describe(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    let name = &input.ident;
    // ลืมเรียก .into() ตอนจบ — quote! คืน proc_macro2::TokenStream ไม่ใช่ proc_macro::TokenStream
    quote! {
        impl #name {
            pub fn describe(&self) {}
        }
    }
}
```

Error จริงที่ได้ (ตรวจสอบแล้ว):

```
error[E0308]: mismatched types
   --> src/lib.rs:10:5
    |
  6 |   pub fn derive_describe(input: TokenStream) -> TokenStream {
    |                                                 ----------- expected `proc_macro::TokenStream` because of return type
...
 10 | /     quote! {
 11 | |         impl #name {
 12 | |             pub fn describe(&self) {}
 13 | |         }
 14 | |     }
    | |_____^ expected `proc_macro::TokenStream`, found `proc_macro2::TokenStream`
    |
    = note: `proc_macro2::TokenStream` and `proc_macro::TokenStream` have similar names, but are actually distinct types
help: call `Into::into` on this expression to convert `proc_macro2::TokenStream` into `proc_macro::TokenStream`
    |
 14 |     }.into()
    |      +++++++
```

**สาเหตุ:** ดังที่อธิบายในหัวข้อ 44.4 — `quote!{}` ทำงานบน `proc_macro2::TokenStream` เสมอ (เพื่อให้ `syn`/`quote` ใช้งานได้นอก context ของ proc macro compiler plugin ด้วย เช่นในการเขียน unit test) แต่ entry point ของ proc macro ต้องคืน `proc_macro::TokenStream` (type ของ `rustc` เอง) ทั้งสอง type **ชื่อเดียวกันเป๊ะแต่มาจากคนละ crate** ทำให้เป็นกับดักที่พบบ่อยมากสำหรับมือใหม่ ที่ดีคือ compiler สมัยใหม่ (ตามที่เห็นใน error ข้างบน) ตรวจจับกรณีนี้ได้แม่นยำและ**แนะนำวิธีแก้ให้ตรงจุดเลย** (`help: call \`Into::into\`...`) **วิธีแก้:** เติม `.into()` ต่อท้ายผลลัพธ์ของ `quote!{}` เสมอก่อน `return`/ก่อนจบฟังก์ชัน

---

**3. เข้าใจผิดว่า derive macro แก้ไข/แทนที่ item เดิมได้**

```rust
// derive_derive/src/lib.rs — พยายาม "เพิ่ม field" ให้ struct เดิมโดยประกาศ struct ชื่อเดียวกันใหม่
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, DeriveInput};

#[proc_macro_derive(AddTimestamp)]
pub fn derive_add_timestamp(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    let name = &input.ident;
    quote! {
        struct #name {
            created_at: u64,
        }
    }
    .into()
}
```

```rust
// ในโค้ดของผู้ใช้ (consumer)
use bad_modify_derive::AddTimestamp;

#[derive(AddTimestamp)]
struct Event {
    name: String,
}
```

Error จริงที่ได้ (ตรวจสอบแล้ว):

```
error[E0428]: the name `Event` is defined multiple times
 --> src/main.rs:4:1
  |
3 | #[derive(AddTimestamp)]
  |          ------------ previous definition of the type `Event` here
4 | struct Event {
  | ^^^^^^^^^^^^ `Event` redefined here
  |
  = note: `Event` must be defined only once in the type namespace of this module

For more information about this error, try `rustc --explain E0428`.
```

**สาเหตุ:** นี่คือหลักฐานที่เป็นรูปธรรมที่สุดของกฎในหัวข้อ 44.2: **derive macro ไม่สามารถแก้ไขหรือแทนที่ item เดิมได้เลย** — `struct Event { name: String }` ที่ผู้ใช้เขียนไว้**ยังคงอยู่ในโค้ดจริงเสมอ** ไม่ว่า derive macro จะ generate อะไรออกมา สิ่งที่ derive macro generate จะถูก**เพิ่มเข้ามาข้าง ๆ** เท่านั้น เมื่อ macro นี้ generate `struct Event { created_at: u64 }` ออกมาอีกตัว compiler จึงเห็น `Event` ถูกประกาศ **2 ครั้ง** ในโค้ดเดียวกัน (ครั้งแรกจากผู้ใช้ ครั้งที่สองจากที่ macro generate) ซึ่งขัดกับกฎว่าชื่อ type ต้องไม่ซ้ำกันในเนมสเปซเดียว (Part 16/17) **วิธีแก้:** ถ้าต้องการ "เพิ่มพฤติกรรม" ให้ struct เดิม ให้ generate เป็น `impl` block เท่านั้น (แบบที่เราทำใน `Describe`/`Validated`) ไม่ใช่ประกาศ `struct`/`enum` ชื่อเดิมซ้ำ — ถ้าต้องการ**แก้ไข field ของ item เดิมจริง ๆ** (เช่น เพิ่ม field เข้าไปในนิยามเดิม) นั่นคือความสามารถของ **attribute macro** เท่านั้น (ซึ่งอ่านและคืน TokenStream ใหม่แทนที่ item เดิมทั้งหมดได้ ตามหัวข้อ 44.2) ไม่ใช่ความสามารถของ derive macro

---

**4. ใช้ `panic!`/`.unwrap()` แทน `syn::Error` ทำให้ error message ไม่เป็นมิตร**

```rust
#[proc_macro_derive(PanickyValidated)]
pub fn derive_panicky(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    let struct_name = &input.ident;

    match &input.data {
        Data::Struct(_) => {
            quote! {
                impl #struct_name {
                    pub fn is_valid(&self) -> bool { true }
                }
            }
            .into()
        }
        _ => panic!("PanickyValidated ใช้ได้กับ struct เท่านั้น"),  // <-- ไม่ดี
    }
}
```

เมื่อผู้ใช้ derive กับ `enum` ผิดเงื่อนไข ผลลัพธ์จริงคือ:

```
error: proc-macro derive panicked
 --> src/main.rs:3:10
  |
3 | #[derive(PanickyValidated)]
  |          ^^^^^^^^^^^^^^^^
  |
  = help: message: PanickyValidated ใช้ได้กับ struct เท่านั้น
```

เทียบกับเวอร์ชันที่ใช้ `syn::Error::new_spanned(...).to_compile_error()` (ตามหัวข้อ 44.9) ที่ให้:

```
error: Validated ใช้ได้กับ struct เท่านั้น ไม่รองรับ enum หรือ union
 --> src/main.rs:4:6
  |
4 | enum Status {
  |      ^^^^^^
```

**สาเหตุและวิธีแก้:** อธิบายละเอียดแล้วในหัวข้อ 44.9 — สรุปคือ `panic!` ทำให้ compiler ต้องรายงานว่า "โค้ด plugin ของคุณพัง" (ชี้ไปที่ `#[derive(...)]` และมีคำว่า "panicked" ซึ่งฟังดูเหมือนบั๊กของ macro เอง) ในขณะที่ `syn::Error`/`to_compile_error()` ทำให้ compiler รายงานเหมือน error ปกติที่ชี้ไปยัง**ตำแหน่งที่ถูกต้องในโค้ดของผู้ใช้**พร้อมข้อความที่คุณเขียนเองตรง ๆ — proc macro ที่เขียนดีควรใช้แบบหลังเสมอสำหรับ error ที่เกิดจาก input ของผู้ใช้ (สงวน `panic!`/`.unwrap()` ไว้เฉพาะกรณีที่เป็น bug ภายในของ macro เองจริง ๆ ที่ไม่ควรเกิดขึ้นได้เลยในทางทฤษฎี)

---

**5. ใส่ item ปกติ (function/struct ที่ไม่ใช่ proc macro) ปนไว้ใน proc-macro crate แล้วคาดหวังให้ import ใช้ได้ตรง ๆ**

```rust
// describe_derive/src/lib.rs
use proc_macro::TokenStream;

#[proc_macro_derive(Describe)]
pub fn derive_describe(input: TokenStream) -> TokenStream {
    // ...
    # unimplemented!()
}

// ฟังก์ชัน helper ธรรมดาที่ตั้งใจให้ consumer เรียกใช้ตรง ๆ ด้วย
pub fn helper_function() -> i32 {
    42
}
```

ปัญหานี้ไม่ทำให้เกิด compile error ทันที (crate ที่มี `proc-macro = true` ยัง compile item ปกติอื่น ๆ ในไฟล์เดียวกันได้อยู่) แต่เป็นความเข้าใจผิดเชิงสถาปัตยกรรมที่พบบ่อย: **proc-macro crate ถูกออกแบบมาให้ export "แค่ macro" เป็นหลัก** ในทางปฏิบัติ item ปกติที่ `pub` ไว้ในนั้นมักใช้งานได้จำกัดกว่าที่คาดสำหรับ use case บางอย่าง (เช่น cross-compilation ที่ proc-macro crate ถูก compile สำหรับ **host** ไม่ใช่ **target** ตามที่อธิบายในหัวข้อ 44.3 — ถ้า `helper_function()` ถูกเรียกใช้เป็นค่าจริงตอน runtime ของ `target`, มันจะเป็นฟังก์ชันที่ compile ผิด platform โดยไม่รู้ตัว ถ้า host/target ต่างกัน) **วิธีแก้ที่เป็นธรรมเนียมมาตรฐาน:** ถ้าต้องการแบ่ง helper function ที่ทั้ง proc macro และผู้ใช้ทั่วไปต้องเรียกใช้ ให้แยกเป็น **crate ที่สาม** (library ธรรมดา ไม่ใช่ proc-macro) ที่ทั้ง proc-macro crate และ consumer depend on ร่วมกัน — pattern แบบ `serde`/`serde_derive`/`serde` (ตัว `serde` หลักมี logic ทั่วไป ส่วน `serde_derive` มีแค่ proc macro ล้วน ๆ)

---

**6. ลืมเปิด `features = ["full"]` ของ `syn` แล้วต้อง parse item ที่ซับซ้อนกว่า struct/enum ธรรมดา**

```toml
# Cargo.toml ของ attribute macro crate — ใช้ syn โดยไม่เปิด "full"
[dependencies]
syn = "2"          # <-- ไม่มี features = ["full"]
quote = "1"
```

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, ItemFn};   // <-- ItemFn ต้องใช้ feature "full"

#[proc_macro_attribute]
pub fn log_call(_attr: TokenStream, item: TokenStream) -> TokenStream {
    let input_fn = parse_macro_input!(item as ItemFn);
    let block = &input_fn.block;
    let sig = &input_fn.sig;
    quote! { #sig #block }.into()
}
```

Error จริงที่ได้ (ตรวจสอบแล้ว):

```
error[E0432]: unresolved import `syn::ItemFn`
 --> src/lib.rs:3:30
  |
3 | use syn::{parse_macro_input, ItemFn};
  |                              ^^^^^^ no `ItemFn` in the root
  |
note: found an item that was configured out
  --> .../syn-2.0.119/src/lib.rs:432:43
   |
427 | #[cfg(feature = "full")]
   |       ---------------- the item is gated behind the `full` feature
...
432 |     ItemConst, ItemEnum, ItemExternCrate, ItemFn, ItemForeignMod, ItemImpl, ItemMacro, ItemMod,
   |                                           ^^^^^^

For more information about this error, try `rustc --explain E0432`.
```

**สาเหตุ:** `syn` ออกแบบให้ประหยัดเวลา compile โดย default เปิดใช้แค่ชุดไวยากรณ์บางส่วน (พอสำหรับ derive macro ทำงานกับ struct/enum ธรรมดาส่วนใหญ่ได้ — สังเกตว่าตัวอย่าง `Describe`/`Validated` ในบทเรียนใช้ `DeriveInput` ได้แม้ไม่เปิด `"full"` ก็ตาม เพราะ `DeriveInput`/`Data`/`Fields` อยู่ในฟีเจอร์ `derive` ที่เป็น default) แต่ type สำหรับ item ระดับอื่น ๆ ที่ครอบคลุมทั้งภาษา เช่น `syn::ItemFn` (ฟังก์ชัน), `syn::ItemImpl` (impl block), `syn::ItemMod` (module) ถูกกันไว้หลัง feature flag `"full"` เพราะมันดึงเข้ามาซึ่ง parser ของไวยากรณ์ Rust แบบเต็มทั้งภาษา (compile ช้ากว่า) **วิธีแก้:** เปิด `features = ["full"]` เสมอเมื่อ macro ของคุณต้อง parse item ที่ไม่ใช่แค่ struct/enum ธรรมดา (โดยเฉพาะ **attribute macro ที่ทำงานกับฟังก์ชัน** อย่าง `#[log_call]` ในหัวข้อ 44.2 ซึ่งต้องใช้ `syn::ItemFn` เสมอ)

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** สร้าง workspace ใหม่ที่มี proc-macro crate ชื่อ `hello_derive` ที่ implement `#[derive(SayHello)]` — เมื่อ derive กับ struct ใดก็ตาม ให้ generate เมธอด `fn say_hello(&self)` ที่พิมพ์ `"สวัสดีจาก <ชื่อ struct>!"` (ไม่ต้องอ่าน field เลย ใช้แค่ `input.ident`) ทดสอบด้วย struct 2 ตัวที่ชื่อต่างกันใน consumer crate แยก แล้วยืนยันว่า output ถูกต้องสำหรับทั้งสองตัว
   (hint: เริ่มจากโครงสร้าง `describe_derive` ในบทเรียน แต่ตัด logic การอ่าน field ทั้งหมดออก เหลือแค่ `#struct_name` ตัวเดียวใน `quote!`)

2. **[กลาง]** ขยาย derive macro `Describe` จากบทเรียนให้รองรับ **tuple struct** ด้วย (เช่น `struct Point(f64, f64);`) โดยพิมพ์ index ของแต่ละ field แทนชื่อ (เพราะ tuple struct ไม่มีชื่อ field) เช่น `Point(1.0, 2.0)` ควร print `field.0`, `field.1` เป็นต้น — ต้องแก้ `match &data_struct.fields` ให้มี arm สำหรับ `Fields::Unnamed` เพิ่มเข้ามา (ใช้ `fields.unnamed.iter().enumerate()` เพื่อได้ index) เขียน consumer ที่ทดสอบทั้ง struct แบบ named field และ tuple struct ในไฟล์เดียวกัน ยืนยันว่าทั้งสองแบบทำงานถูกต้อง

3. **[ยาก]** เขียน derive macro ชื่อ `#[derive(FieldCount)]` ที่ generate **associated constant** (ไม่ใช่ method) ชื่อ `FIELD_COUNT: usize` เข้าไปใน `impl` block (เช่น `Customer::FIELD_COUNT` ควรได้ `3`) จากนั้นเขียน function-like proc macro เพิ่มอีกตัวชื่อ `count_fields!(TypeName)` ที่รับชื่อ type เป็น argument (parse ด้วย `syn::parse_macro_input!(input as syn::Ident)` หรือ `syn::Type`) แล้ว expand เป็นการเรียก `TypeName::FIELD_COUNT` ตรง ๆ — โจทย์นี้ฝึกทั้งการสร้าง constant ผ่าน `quote!` และการเขียน function-like macro ที่ parse input จริง (ไม่ใช่ทิ้ง input แบบ `make_answer!()` ในบทเรียน)
   (hint: `quote!` รองรับการ generate `impl` ที่มี `const` ปนกับ `fn` ได้ในบล็อกเดียวกันตามปกติ เหมือนเขียน `impl` มือ)

4. **[ยาก/ประยุกต์ใช้งานจริง]** ออกแบบระบบ "inventory validation" ง่าย ๆ: เขียน derive macro ชื่อ `#[derive(NonEmpty)]` ที่ generate เมธอด `fn validate(&self) -> Result<(), String>` ให้ struct ที่มี field ชนิด `String` **ทุกตัว** — เมธอดนี้ควรวน loop ตรวจสอบว่าทุก field ที่เป็น `String` ไม่เป็นค่าว่าง (`""`) ถ้าพบ field ว่างให้คืน `Err(format!("field '{}' ต้องไม่เป็นค่าว่าง", ชื่อ field))` ถ้าผ่านหมดให้คืน `Ok(())` — ต้องใช้ `syn::Type` เพื่อเช็คว่า field แต่ละตัวเป็น `String` หรือไม่ (เทียบ path ของ type กับ `"String"`) และต้องเขียนโค้ดที่รายงาน error ด้วย `syn::Error`/`to_compile_error()` อย่างสุภาพถ้า struct ที่ derive ไม่มี named field เลย (ตามหลักการหัวข้อ 44.9) ทดสอบด้วย struct `Order { customer_name: String, note: String }` ทั้งกรณีข้อมูลถูกต้องและกรณีมี field ว่าง — โจทย์นี้ฝึกทั้งการอ่าน `syn::Type` เพื่อเช็คชนิด, generate โค้ดที่มี logic ควบคุมการทำงาน (ไม่ใช่แค่ print), และการรายงาน compile error อย่างมีอารยะพร้อมกัน
   (hint: การเช็คว่า `syn::Type` คือ `String` แบบง่ายที่สุดคือ pattern match บน `syn::Type::Path(TypePath { path, .. })` แล้วเช็คว่า `path.segments.last().unwrap().ident == "String"` — วิธีนี้ไม่สมบูรณ์แบบ 100% ในทุกกรณี generic/alias แต่เพียงพอสำหรับโจทย์นี้)

## สรุป

บทนี้เราได้ต่อยอดจาก Part 36 ในจุดที่ Part 36 ทิ้งค้างไว้อย่างตั้งใจ: การเรียน **procedural macro** ทั้ง 3 ชนิด — **derive macro** (`#[derive(TraitName)]`, generate โค้ดใหม่เพิ่มเข้ามาข้าง item เดิมเท่านั้น ไม่สามารถแก้ไข item เดิมได้), **attribute macro** (`#[my_attribute]`, อ่านและเขียนทับ item เดิมทั้งหมดได้ ทรงพลังที่สุด), และ **function-like macro** (`my_macro!(...)`, หน้าตาเหมือน `macro_rules!` แต่มีพลังของ proc macro เต็มรูปแบบอยู่ข้างใน) เราได้เห็นเหตุผลเชิงลึกว่าทำไม `macro_rules!` ที่เก่งเรื่อง pattern matching ของ token ไม่สามารถทำสิ่งที่ต้องการ**วิเคราะห์โครงสร้าง**ของโค้ด Rust จริง ๆ ได้ (เช่น มองเข้าไปข้างในว่า type ตัวหนึ่งคือ `Option<T>` หรือเปล่า) และทำไม proc macro ที่ parse token stream เป็น AST ที่มี type ครบถ้วนด้วย `syn` จึงเข้ามาเติมเต็มช่องว่างนี้ได้

เราได้เรียนกฎเชิงกลไกที่สำคัญที่สุด: proc macro ต้องอยู่ใน crate ที่ประกาศ `proc-macro = true` เพราะมันรันเป็น **compiler plugin ของ host** ที่ต้อง compile ให้เสร็จสมบูรณ์ก่อนที่ crate ผู้ใช้จะเริ่ม compile ได้ — ความสัมพันธ์เชิงเวลา build ที่ต่างจาก dependency ปกติโดยสิ้นเชิง และได้เรียนรู้ pipeline มาตรฐานของ ecosystem: `syn::parse_macro_input!` แปลง `TokenStream` ดิบเป็น AST (`DeriveInput`), โค้ด Rust ธรรมดาวิเคราะห์/ดึงข้อมูลจาก AST นั้น, `quote!{}` แปลงโค้ดที่เขียนตรง ๆ กลับเป็น `TokenStream` ผ่านการ interpolate ด้วย `#variable` (รวมถึง repetition `#(...)* ` ที่ mental model เดียวกับ `$(...),* ` ของ `macro_rules!`) เราได้เขียนและ**ทดสอบให้ทำงานได้จริง**ทั้ง derive macro เต็มรูปแบบ (`#[derive(Describe)]`) และ function-like macro แบบง่าย (`make_answer!()`) พร้อมเห็น output จริงจากการ compile และ run

ที่สำคัญไม่แพ้กันคือเรื่อง**คุณภาพของ proc macro ที่ดี**: การใช้ `syn::Error::new_spanned(...).to_compile_error()` แทน `panic!`/`.unwrap()` เพื่อให้ error message ชี้ไปยังตำแหน่งที่ถูกต้องในโค้ดของผู้ใช้ด้วยข้อความที่อ่านเข้าใจได้ทันที ต่างจาก error แบบ "proc-macro derive panicked" ที่ทำให้ผู้ใช้สับสนว่าเป็นบั๊กของ macro เองหรือของตัวเอง และเรื่องการ debug ด้วย `cargo expand`/`eprintln!` ซึ่งเป็นทักษะจำเป็นเพราะการ debug proc macro ยากกว่าโค้ดปกติมาก (มันรันตอน compile time ของคนอื่น ไม่ใช่ runtime ของตัวเอง)

สุดท้าย เราได้ตั้งความคาดหวังที่สมจริง: **derive macro คือชนิดที่คุณมีโอกาสต้องเขียนเองมากที่สุด** สำหรับ domain trait ของทีม ในขณะที่ attribute macro และ function-like macro ที่ซับซ้อนมักเป็นสิ่งที่คุณ**ใช้**มากกว่า**เขียน** — บทนี้จึงวางน้ำหนักไปที่ derive macro เป็นหลักด้วยเหตุผลนี้

ใน **Part 45** เราจะกลับมาที่ derive macro อีกครั้งเพื่อเจาะลึกเทคนิคขั้นสูงกว่านี้ที่บทนี้ยังไม่ได้แตะ: การอ่าน **attribute บน field** ที่ผู้ใช้กำหนดเอง (เช่น `#[describe(skip)]` เพื่อบอกให้ macro ข้าม field บางตัว — เทคนิคเดียวกับที่ `serde` ใช้ทำ `#[serde(rename = "...")]`), การรองรับ **generic parameter** ใน struct ที่ derive อยู่อย่างถูกต้อง (ต้องส่ง `impl<T> ... for Name<T>` พร้อม bound ที่เหมาะสม), และการรองรับ **enum** ที่มี variant หลากหลายรูปแบบ (ไม่ใช่แค่ struct แบบที่บทนี้โฟกัส) — เทคนิคเหล่านี้คือสิ่งที่ทำให้ derive macro ระดับ production อย่าง `serde`/`thiserror` ใช้งานได้ครอบคลุมสถานการณ์ที่หลากหลายอย่างที่คุณเห็นในโลกจริง

---

**Part ก่อนหน้า:** [FFI: การเชื่อมต่อกับ C](part-043-ffi-c.md) | **Part ถัดไป:** [Procedural Macros: Derive Macros ขั้นสูง](part-045-proc-macros-derive.md)
