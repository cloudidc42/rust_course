# Part 45: Procedural Macros: Derive Macros ขั้นสูง

> โมดูล: ระดับสูง (Advanced) | ระดับ: สูง | เวลาโดยประมาณ: 260 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อ่าน `syn::Data` และ `syn::Fields` ได้ครบทุก variant (`Data::Struct`/`Data::Enum`/`Data::Union` และ
  `Fields::Named`/`Fields::Unnamed`/`Fields::Unit`) และรู้ว่าต้อง generate โค้ดต่างกันอย่างไรสำหรับ
  tuple struct, unit struct และ enum ที่มี variant ผสมกันทั้งสามแบบ — ไม่ใช่แค่ named-field struct
  แบบง่ายที่ทำใน Part 44
- อธิบายได้ว่าทำไม derive macro ที่ generate `impl` ให้ type ที่มี generic parameter ของตัวเอง
  **ต้อง**เติม trait bound ให้ generated `impl` block เอง และรู้จักใช้ `syn::Generics::split_for_impl()`
  กับ `type_params_mut()` เพื่อทำสิ่งนี้อย่างถูกต้อง พร้อมอธิบายได้ว่าทำไม bug นี้ถึง**ไม่โผล่ตอน derive
  บน type ธรรมดา** แต่โผล่เฉพาะตอน derive บน type ที่มี generics
- อธิบายกลไกของ "helper attribute" (แบบที่ `serde` ใช้กับ `#[serde(rename = "...")]`) และเขียน
  derive macro ที่รับ custom field attribute ของตัวเอง (`#[describe(skip)]`) ได้จริงด้วย
  `#[proc_macro_derive(_, attributes(_))]` และ `syn::Attribute::parse_nested_meta`
- เขียน error handling ระดับ production ให้ proc macro: ใช้ `syn::Error` กับ `span` ที่ถูกต้อง แทน
  `panic!`/`unwrap()` เพื่อให้ผู้ใช้ macro เห็น compile error ที่ชี้ตำแหน่งถูกและอ่านเข้าใจได้ทันที
- รู้จักปัญหาของการทดสอบ proc macro (macro ทำงานกับ `TokenStream` ไม่ใช่ค่า runtime ธรรมดา) และรู้จัก
  แนวทางที่ระบบจริงใช้ (`trybuild`) สำหรับเทส compile-time behavior/compile error message
- ออกแบบและ implement `#[derive(Builder)]` แบบเต็มรูปแบบที่ generate builder pattern (struct
  ตัวกลาง, setter method ที่ chain ได้, และ `.build() -> Result<T, String>`) ให้ struct ใด ๆ ที่มี
  named field ได้จริง พร้อมทดสอบกับ consumer crate จริงจนรันได้ผลลัพธ์ถูกต้อง
- ตัดสินใจได้อย่างมีหลักการว่าเมื่อไหร่ควรเขียน proc macro ของตัวเอง กับเมื่อไหร่ควรใช้ derive macro
  สำเร็จรูปจาก crates.io (`serde`, `thiserror`, `clap`) แทน — ปิดวงคำถาม "เมื่อไหร่ควรใช้ macro" ที่เปิด
  ไว้ตั้งแต่ Part 36 ในระดับที่ลึกที่สุด

## ความรู้ที่ต้องมีมาก่อน

- **Part 44 (Procedural Macros เบื้องต้น) — สำคัญที่สุด** บทนี้เป็น**ภาคต่อโดยตรง**ไม่ใช่บทแยก
  ใน Part 44 เราสร้าง `#[derive(Describe)]` ตัวแรกด้วย `syn`/`quote` แต่จำกัดตัวเองไว้แค่กรณีที่ง่ายที่สุด
  คือ struct ที่มี named field ล้วน ๆ ไม่มี generic ของตัวเอง ไม่มี attribute พิเศษ และตอน error ก็แค่
  `panic!`/`.unwrap()` แบบหยาบ ๆ Part 44 บอกไว้ชัดว่านี่คือ "เวอร์ชันง่ายที่สุดเพื่อให้เข้าใจกลไกก่อน" และ
  บอกไว้ว่าจะมาขยายความ "เทคนิค derive macro ขั้นสูง" ในบทถัดไป — บทนี้คือบทนั้น ถ้าคุณจำโครงสร้างพื้นฐาน
  ของ `#[proc_macro_derive]`, `syn::DeriveInput`, `quote!`, และวิธีใช้ `cargo` แบบ 2-crate (proc-macro
  crate + consumer crate) จาก Part 44 ไม่ได้แล้ว ควรย้อนกลับไปอ่านซ้ำก่อน เพราะบทนี้จะไม่สอนพื้นฐานเหล่านั้น
  ซ้ำอีก
- **Part 10 (Enums และ Pattern Matching)** — หัวข้อ 45.4 จะจับคู่ `syn::Fields` ของ**แต่ละ variant**ของ
  enum แล้ว generate `match` arm ที่ต่างกันตามรูปแบบ (unit/tuple/struct-like variant) ต้องเข้าใจโครงสร้าง
  enum ทั้งสามแบบนี้ในภาษา Rust จริงก่อน ถึงจะเข้าใจว่าทำไม `syn` ต้องแยก case ให้ตรงกัน
- **Part 19 (Traits เบื้องต้น)** — เข้าใจว่า `impl Trait for Type` คืออะไร เพราะสิ่งที่ derive macro
  ทุกตัวทำในที่สุดคือ generate `impl` block แบบนี้ให้อัตโนมัติ
- **Part 22 (Generics ขั้นสูง)** — แนวคิด trait bound และ `where` clause ที่หัวข้อ 45.6-45.7 จะนำมาใช้
  ตอนเติม bound ให้ generated code เอง
- **Part 31 (thiserror และ anyhow)** — คุณเคยใช้ `#[derive(Error)]` จาก `thiserror` มาแล้วในฐานะ
  "ผู้ใช้" derive macro บทนี้จะเฉลยว่ากลไกเบื้องหลัง `#[error("...")]` (ซึ่งเป็น helper attribute
  แบบเดียวกับที่เราจะสร้างในหัวข้อ 45.8-45.9) ทำงานอย่างไรจริง ๆ
- **Part 36 (Macros: Declarative Macros)** — หลักการ "macro ควรเป็นตัวเลือกสุดท้าย" ที่ Part 36 สอนไว้
  ในระดับ `macro_rules!` vs generics/trait บทนี้จะยกระดับคำถามเดียวกันขึ้นไปอีกชั้น: "เขียน proc macro
  เองเมื่อไหร่ vs ใช้ derive macro สำเร็จรูปจาก ecosystem เมื่อไหร่" (หัวข้อ 45.15)
- แวบไปดูล่วงหน้า: **Part 52 (Design Patterns: Builder)** จะสอน builder pattern แบบเขียนมือ (ทีละ field)
  ส่วนบทนี้ (หัวข้อ 45.13) จะสอน**การ generate โค้ด builder pattern แบบอัตโนมัติ**ด้วย derive macro —
  สองบทนี้เสริมกัน ไม่ทับกัน — และ **Part 59 (CLI ด้วย clap)** ที่คุณจะได้ใช้ `#[derive(Parser)]` ของ
  `clap` ซึ่งใช้เทคนิคหลักเดียวกันกับที่เราสร้างเองในบทนี้ทุกอย่าง (helper attribute, generics support,
  compile-time validation)

## เนื้อหา

### 45.1 ทวนความจำจาก Part 44 และภาพรวมของบทนี้

ใน Part 44 เราสร้าง `#[derive(Describe)]` เวอร์ชันแรกที่ทำสิ่งเดียวคือ: รับ struct ที่มี named field
แล้ว generate **inherent method** `pub fn describe(&self)` ที่ `println!` ชื่อ struct และชื่อของทุก
field ออกมาตรง ๆ (ไม่คืนค่าอะไรกลับ ไม่มี `trait Describe` เกี่ยวข้องเลยด้วยซ้ำ — เป็นแค่ `impl
StructName { pub fn describe(&self) { ... } }` ธรรมดา) โค้ดหน้าตาประมาณนี้ (สรุปจาก Part 44 หัวข้อ
44.6):

```rust
// สรุปแนวคิดจาก Part 44 — รองรับแค่ named-field struct เท่านั้น และ describe() print ตรง ๆ
// ไม่มี trait, ไม่คืนค่า
#[proc_macro_derive(Describe)]
pub fn derive_describe(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    let struct_name = &input.ident;

    let field_names: Vec<String> = match &input.data {
        Data::Struct(data_struct) => match &data_struct.fields {
            Fields::Named(fields) => fields
                .named
                .iter()
                .map(|f| f.ident.as_ref().unwrap().to_string())
                .collect(),
            _ => return syn::Error::new_spanned(struct_name, "รองรับแค่ named field")
                .to_compile_error(),
        },
        _ => return syn::Error::new_spanned(struct_name, "รองรับแค่ struct").to_compile_error(),
    };

    quote! {
        impl #struct_name {
            pub fn describe(&self) {
                println!("struct ชื่อ: {}", stringify!(#struct_name));
                // ... print ชื่อ field แต่ละตัว ...
            }
        }
    }
}
```

Part 44 ทำสิ่งที่ถูกต้องอย่างหนึ่งไปแล้วคือใช้ `syn::Error`/`to_compile_error()` แทน `panic!` ตั้งแต่
ต้น (หัวข้อ 44.9) — บทนี้จะไม่สอนเรื่องนั้นซ้ำ แต่จะ**ขยาย**มันให้ครอบคลุมทุกจุดของ macro ไม่ใช่แค่จุด
ปฏิเสธ input ผิดเงื่อนไข (หัวข้อ 45.10-45.11)

**การปรับเปลี่ยนที่บทนี้จะทำกับ `Describe` และเหตุผล:** เพื่อให้บทนี้พาไปถึงหัวข้อที่ซับซ้อนกว่า
(enum ที่มี field ให้ format, generics ที่ต้องมี bound, helper attribute) ได้อย่างเป็นธรรมชาติ เราจะ
ปรับ `Describe` สองจุดจากเวอร์ชันของ Part 44:

1. **เปลี่ยนจาก inherent method (`impl StructName { fn describe(&self) }`) เป็น trait
   (`impl Describe for StructName`)** — เหตุผล: เมื่อต้อง generate โค้ดให้ทั้ง struct และ enum ที่มี
   variant หลายแบบ (หัวข้อ 45.4) การมี `trait Describe { fn describe(&self) -> String; }` ที่ตายตัว
   ทำให้เขียนโค้ดที่**เรียกใช้ `describe()` ซ้อนกัน**ได้ง่ายขึ้นมาก (เช่น field ตัวหนึ่งเป็น struct อีก
   ตัวที่ implement `Describe` ไว้แล้ว ก็เรียก `.describe()` ซ้อนได้ทันทีถ้ามี trait ที่แน่นอน) และเป็น
   รูปแบบที่ตรงกับ derive macro ระดับ production ส่วนใหญ่ (`Debug`, `Serialize`, `Error`) ที่ล้วน
   generate `impl Trait for Type` ทั้งนั้น ไม่ใช่ inherent method
2. **เปลี่ยนจากการ `println!` ตรง ๆ (คืนค่า `()`) เป็นการคืนค่า `String`** — เหตุผล: การคืนค่ากลับทำให้
   ผลลัพธ์**ทดสอบได้ง่ายขึ้นมาก** (`assert_eq!(p.describe(), "...")` โดยไม่ต้อง capture stdout) และทำให้
   ผู้ใช้เอาผลลัพธ์ไปต่อกับอย่างอื่นได้ (เช่น เก็บ log, ส่งผ่าน network) ไม่ใช่ผูกติดกับการ print ไปที่
   stdout เพียงอย่างเดียว — Part 44 หัวข้อ 44.6.1 ก็แสดงให้เห็นแล้วว่าการทดสอบ token ที่ expand ออกมา
   ด้วย `.to_string()` แล้วเช็ค substring เป็นวิธีที่ใช้ได้ แต่การให้ฟังก์ชันที่ generate คืนค่าที่เทียบ
   ได้ตรง ๆ ยิ่งทดสอบง่ายกว่าไปอีกขั้น

ทั้งสองจุดนี้**ไม่ใช่การแก้ไขความผิดของ Part 44** (เวอร์ชันเดิมถูกต้องสมบูรณ์สำหรับเป้าหมายของมันคือสอน
กลไกพื้นฐาน) แต่เป็นการ**ปรับ design ให้เหมาะกับเนื้อหาที่ลึกขึ้น**ของบทนี้ — เช่นเดียวกับที่ Part 21
ปรับปรุง trait จาก Part 19 ให้รองรับ `dyn Trait`/default method ได้ ไม่ใช่บอกว่า Part 19 สอนผิด

นี่คือจุดเริ่มต้นที่ดีสำหรับการเข้าใจกลไกพื้นฐาน แต่ในโลกจริง แทบไม่มี derive macro ระดับ production
ตัวไหนหยุดอยู่แค่นี้เลย ลองนึกถึง `#[derive(Debug)]` ของ standard library เอง — มันต้อง derive ได้กับ

- struct ที่มี named field (`struct Point { x: i32, y: i32 }`)
- struct ที่เป็น tuple struct (`struct Coord(f64, f64)`)
- struct ที่เป็น unit struct (`struct Marker;`)
- enum ที่มี variant ผสมกันทุกแบบ (`enum Shape { Empty, Circle(f64), Rectangle { w: f64, h: f64 } }`)
- struct/enum ที่มี generic parameter ของตัวเอง (`struct Wrapper<T> { value: T }`)

และ `#[derive(Serialize)]` ของ `serde` ต้องรองรับทั้งหมดข้างบน **บวก**การให้ผู้ใช้ปรับแต่งพฤติกรรมต่อ
field ด้วย attribute อย่าง `#[serde(rename = "...")]` หรือ `#[serde(skip)]` **บวก**การรายงาน compile
error ที่อ่านเข้าใจได้ทันทีเมื่อใช้ macro ผิดวิธี (ไม่ใช่ panic ที่บอกแค่ "internal error: entered
unreachable code" แบบดิบ ๆ)

บทนี้จะไล่แก้ทุกข้อจำกัดของ Part 44 ทีละข้อ โดยยึด `#[derive(Describe)]` ตัวเดิมเป็นตัวอย่างหลัก
(ขยายให้รองรับทุก data shape + generics + helper attribute + error handling ที่ดี) แล้วปิดท้ายด้วย
case study ใหญ่ตัวใหม่คือ `#[derive(Builder)]` ที่ generate builder pattern ให้ struct ใด ๆ ได้จริง

ทุกตัวอย่างโค้ดในบทนี้ผ่านการ `cargo build`/`cargo run` จริงในสอง crate (proc-macro crate + consumer
crate) เหมือนที่ Part 44 สอนไว้ ผลลัพธ์ที่แสดงในบทความทุกจุดคือ output จริงจาก terminal ไม่ใช่ output
ที่เดาไว้

### 45.2 กายวิภาคของ `syn::Data`: เมื่อ struct ไม่ใช่แค่ named-field struct

จุดเริ่มต้นของทุกอย่างในบทนี้คือ enum `syn::Data` ที่ `DeriveInput::data` เก็บไว้ ทวนจาก Part 44:
`syn::DeriveInput` คือผลลัพธ์ของการ parse item ที่ derive macro ถูกแปะไว้ (`struct`, `enum`, หรือ
`union`) และ field `data: syn::Data` คือส่วนที่บอกว่า item นั้นเป็น**รูปร่างอะไร**:

```rust
// (นี่คือนิยามจริงใน syn — ไม่ต้องเขียนเอง แค่ import มาใช้ match)
pub enum Data {
    Struct(DataStruct),
    Enum(DataEnum),
    Union(DataUnion),
}
```

- **`Data::Struct(DataStruct)`** — item เป็น `struct` (ไม่ว่าจะ named field, tuple, หรือ unit)
  `DataStruct.fields: Fields` บอกว่าเป็น field แบบไหน
- **`Data::Enum(DataEnum)`** — item เป็น `enum` `DataEnum.variants: Punctuated<Variant, Comma>`
  คือรายการ variant ทั้งหมด และ**แต่ละ variant ก็มี `fields: Fields` ของตัวเองอีกชั้น** (เพราะแต่ละ
  variant ของ enum เดียวกันเป็นได้ทั้ง unit/tuple/struct-like ผสมกันได้ ดู Part 10)
- **`Data::Union(DataUnion)`** — item เป็น `union` (จาก Part 41-42 เรื่อง unsafe/raw pointer memory
  layout ถ้าคุณเรียนมาแล้วจะรู้ว่า `union` แชร์ memory เดียวกันระหว่าง field ทั้งหมด อ่านผิด field
  ได้ undefined behavior) — derive macro ส่วนใหญ่**ไม่รองรับ** union เพราะไม่มีทางรู้ว่า field ไหน
  active อยู่โดยไม่ต้องพึ่ง `unsafe` — จุดที่น่าสังเกตเชิง type: `DataUnion.fields` มี type เป็น
  **`FieldsNamed` ตรง ๆ** (ไม่ใช่ `Fields` enum แบบ `DataStruct`) เพราะไวยากรณ์ของ `union` ในภาษา Rust
  **บังคับให้มี named field เท่านั้นเสมอ** (`union Payload { i: i32, f: f32 }` — ไม่มี tuple union หรือ
  unit union ให้เขียนได้เลยตามกฎภาษา) `syn` จึงเลือก type ที่ตรงกับไวยากรณ์จริงให้ตั้งแต่ระดับ AST เพื่อ
  ป้องกันไม่ให้เขียนโค้ดที่พยายาม match `Fields::Unnamed`/`Fields::Unit` กับ union ได้ตั้งแต่ตอน compile
  ตัว macro เอง (compiler ของ Rust เองช่วยคุณจับบั๊กเชิงตรรกะแบบนี้ได้ตั้งแต่ก่อนรันด้วยซ้ำ)

ส่วน `syn::Fields` ที่ทั้ง `DataStruct` และ `Variant` ใช้ร่วมกัน คือตัวที่บอกรูปร่างจริง ๆ ของ field:

```rust
// (นิยามจริงใน syn)
pub enum Fields {
    Named(FieldsNamed),     // { x: i32, y: i32 }
    Unnamed(FieldsUnnamed),  // (f64, f64)
    Unit,                    // ไม่มีวงเล็บ ไม่มีอะไรเลย
}
```

จับคู่กับตัวอย่างจริงในภาษา Rust (ทวนจาก Part 9-10):

| รูปแบบ | ตัวอย่าง | `Fields` variant |
|---|---|---|
| Named-field struct | `struct Point { x: i32, y: i32 }` | `Fields::Named` |
| Tuple struct | `struct Coord(f64, f64);` | `Fields::Unnamed` |
| Unit struct | `struct Marker;` | `Fields::Unit` |
| Unit variant | `enum E { A }` | `Fields::Unit` |
| Tuple variant | `enum E { A(i32) }` | `Fields::Unnamed` |
| Struct-like variant | `enum E { A { x: i32 } }` | `Fields::Named` |

จุดสำคัญที่ทำให้หลายคนเขียน derive macro พลาดตอนเริ่มต้น คือคิดว่า "struct กับ enum เป็นคนละเรื่องกัน
เลย แยก match ใหญ่สองอันไปเลย" ทั้งที่จริง ๆ **`Fields` เป็น type เดียวกัน ใช้ร่วมกันได้ทั้ง struct
และแต่ละ variant ของ enum** — ดังนั้นแนวทางที่ดีที่สุดคือเขียนฟังก์ชันที่รับ `&Fields` แล้ว generate
โค้ดให้ (ใช้ซ้ำได้ทั้งสองที่) แทนที่จะ copy-paste logic เดียวกันสองรอบ ซึ่งเป็นแนวทางที่เราจะใช้ใน
หัวข้อถัดไป

### 45.3 ขยาย `Describe` ให้รองรับ tuple struct และ unit struct

มาขยาย `#[derive(Describe)]` จาก Part 44 กันทีละก้าว เริ่มจากส่วน struct ก่อน (enum ทำในหัวข้อถัดไป)
เป้าหมาย: `describe()` ต้อง generate โค้ดที่ถูกต้องสำหรับทั้งสามรูปแบบของ `Fields`

```rust
// describe_derive/src/lib.rs (ส่วนที่จัดการ struct)
use proc_macro2::TokenStream as TokenStream2;
use quote::quote;
use syn::{DataStruct, Fields, Ident, Index};

fn describe_struct_body(name: &Ident, data: &DataStruct) -> syn::Result<TokenStream2> {
    let type_name = name.to_string();
    match &data.fields {
        Fields::Named(fields) => {
            let mut format_parts = Vec::new();
            let mut format_args = Vec::new();
            for field in &fields.named {
                let field_ident = field.ident.clone().unwrap();
                // field ที่เป็น Fields::Named การันตีว่า .ident เป็น Some เสมอ
                format_parts.push(format!("{}: {{:?}}", field_ident));
                format_args.push(quote! { self.#field_ident });
            }
            let fmt_string = format!("{type_name} {{{{ {} }}}}", format_parts.join(", "));
            Ok(quote! { format!(#fmt_string, #(#format_args),*) })
        }
        Fields::Unnamed(fields) => {
            let mut format_parts = Vec::new();
            let mut format_args = Vec::new();
            for (i, _field) in fields.unnamed.iter().enumerate() {
                let idx = Index::from(i); // "field 0" ไม่มีชื่อ ต้องเข้าถึงด้วย self.0, self.1, ...
                format_parts.push("{:?}".to_string());
                format_args.push(quote! { self.#idx });
            }
            let fmt_string = format!("{type_name}({})", format_parts.join(", "));
            Ok(quote! { format!(#fmt_string, #(#format_args),*) })
        }
        Fields::Unit => {
            // ไม่มี field ให้ format เลย คืนแค่ชื่อ type
            Ok(quote! { #type_name.to_string() })
        }
    }
}
```

มีจุดที่ควรอธิบายเชิงลึกสามจุด:

**1) tuple struct เข้าถึง field ด้วย `self.0`, `self.1` ไม่ใช่ `self.field_name`** — เพราะ tuple
struct ไม่มีชื่อ field (ตาม Part 9) การอ้างอิง field ต้องใช้ `syn::Index` (ไม่ใช่ตัวเลขธรรมดา หรือ
`Ident`) เพราะ `self.0` ในภาษา Rust เป็น token คนละแบบกับ identifier — `Index` เก็บทั้งค่าตัวเลขและ
`Span` ไว้ (สำหรับ error message ที่ชี้ตำแหน่งถูก) `quote!` รู้จัก `Index` และ render เป็น `self.0`
ให้อัตโนมัติเมื่อใช้ `#idx` ข้าง `self.` — ถ้าคุณลองใช้ตัวเลข `usize` ธรรมดาแทน `Index` ตรงนี้ (เช่น
เขียน `let idx = i;` แล้ว `quote! { self.#idx }`) จะได้ compile error จาก `quote`/`syn` ทันทีเพราะ
`usize` ไม่ implement `ToTokens` ในแบบที่ทำให้กลายเป็น field access ได้ (มันจะกลายเป็น integer literal
เปล่า ๆ ซึ่งผิดไวยากรณ์ตอนตามหลัง `.`)

**2) unit struct ไม่มี field ให้ generate เลย** — สังเกตว่า `Fields::Unit` ไม่มี field ย่อยให้ loop
ผ่านเลย โค้ดในกรณีนี้ตรงไปตรงมาที่สุด: คืนชื่อ type ตรง ๆ นี่คือเหตุผลที่การเขียน `match` ให้ครบทุก
`Fields` variant สำคัญมาก — ถ้า Part 44 เขียนแบบ `Fields::Named(f) => ..., _ => panic!(...)` (จับ
`Fields::Unnamed` และ `Fields::Unit` รวมเป็น `_` แล้ว panic) ผู้ใช้ที่ derive กับ tuple struct หรือ
unit struct จะได้ panic ที่งง ๆ ทันที (เราจะเห็นตัวอย่าง panic message จริงในหัวข้อ 45.11)

**3) ทำไมต้องแยกฟังก์ชันย่อยแยกจาก `describe_enum_body`** — เพราะแม้ `Fields` เป็น type เดียวกัน แต่
"สิ่งที่ห่ออยู่ข้างนอก" ต่างกัน: struct มี `self.field` ตรง ๆ ในขณะที่ enum ต้อง `match self { Variant
{ field } => ... }` ก่อนถึงจะได้ `field` มา (ดูหัวข้อถัดไป) ดังนั้นแม้ logic การ format field คล้ายกัน
มาก แต่การ**เข้าถึง**ค่าของ field ต่างกันโดยพื้นฐาน จึงแยกฟังก์ชันแทนที่จะพยายามยัดรวมกันแบบ generic
เกินไป (ความชัดเจนสำคัญกว่าการลดโค้ดซ้ำเล็กน้อยในกรณีนี้)

### 45.4 enum: unit / tuple / struct-like variant ผสมกัน

ส่วนที่ซับซ้อนที่สุดของบทนี้คือ enum เพราะ (ทวนจาก Part 10) enum เดียวสามารถมี variant ที่เป็น unit,
tuple, และ struct-like ผสมกันได้ในตัวเดียว เช่น

```rust
enum Shape {
    Empty,                              // unit variant
    Circle(f64),                        // tuple variant
    Rectangle { width: f64, height: f64 }, // struct-like variant
}
```

การ generate `describe()` สำหรับ enum ต้อง generate `match self { ... }` ที่มี arm หนึ่งอันต่อหนึ่ง
variant และ**รูปแบบของ arm ต่างกันตาม `Fields` ของ variant นั้น** เหมือนที่ Part 10 สอนไว้เรื่อง
pattern matching กับ enum ที่มี field ต่างชนิดกัน:

```rust
// describe_derive/src/lib.rs (ส่วนที่จัดการ enum)
use quote::format_ident;
use syn::{DataEnum, Fields, Ident};

fn describe_enum_body(name: &Ident, data: &DataEnum) -> syn::Result<TokenStream2> {
    let type_name = name.to_string();
    let mut arms = Vec::new();

    for variant in &data.variants {
        let variant_ident = &variant.ident;
        let variant_name = variant_ident.to_string();

        match &variant.fields {
            Fields::Named(fields) => {
                // struct-like variant: ต้อง destructure { field1, field2, ... } ออกมาก่อน
                let field_idents: Vec<_> =
                    fields.named.iter().map(|f| f.ident.clone().unwrap()).collect();
                let format_parts: Vec<_> = field_idents
                    .iter()
                    .map(|id| format!("{}: {{:?}}", id))
                    .collect();
                let fmt_string = format!(
                    "{type_name}::{variant_name} {{{{ {} }}}}",
                    format_parts.join(", ")
                );
                arms.push(quote! {
                    #name::#variant_ident { #(#field_idents),* } => {
                        format!(#fmt_string, #(#field_idents),*)
                    }
                });
            }
            Fields::Unnamed(fields) => {
                // tuple variant: ต้องตั้งชื่อ binder เอง (field_0, field_1, ...)
                // เพราะ tuple field ไม่มีชื่อให้ destructure ตรง ๆ
                let binders: Vec<_> = (0..fields.unnamed.len())
                    .map(|i| format_ident!("field_{}", i))
                    .collect();
                let format_parts = vec!["{:?}".to_string(); binders.len()];
                let fmt_string =
                    format!("{type_name}::{variant_name}({})", format_parts.join(", "));
                arms.push(quote! {
                    #name::#variant_ident( #(#binders),* ) => {
                        format!(#fmt_string, #(#binders),*)
                    }
                });
            }
            Fields::Unit => {
                // unit variant: ไม่มี field ให้ destructure เลย
                let fmt_string = format!("{type_name}::{variant_name}");
                arms.push(quote! {
                    #name::#variant_ident => #fmt_string.to_string(),
                });
            }
        }
    }

    Ok(quote! {
        match self {
            #(#arms)*
        }
    })
}
```

สามจุดที่ควรสังเกตให้ลึก:

**1) `format_ident!` สำหรับสร้างชื่อตัวแปรใหม่** — tuple field ไม่มีชื่อให้ใช้ตรง ๆ (ต่างจาก named
field ที่มี `field.ident` ให้เลย) เราจึง**ต้องคิดชื่อขึ้นมาเอง**เพื่อ bind ค่าตอน destructure macro
`format_ident!("field_{}", i)` (มาจาก `quote` crate) ทำงานคล้าย `format!` แต่คืน `syn::Ident` ที่ใช้
ใน `quote!` ได้ตรง ๆ แทนการคืน `String` ธรรมดา (ซึ่งถ้าใช้ `String` ตรง ๆ ใน `quote!` จะกลายเป็น string
literal ไม่ใช่ identifier — เป็นอีกจุดที่มือใหม่งงบ่อย)

**2) generated code ของ enum คือ `match self { ... }` ซ้อนอีกชั้นในฟังก์ชันที่ generate `match`
เอง** — นี่คือจุดที่แสดงพลังของ macro ได้ชัดที่สุด: โค้ด**ที่เราเขียน**ใช้ `for variant in
&data.variants` (วน loop ตอน**compile time**ของ derive macro) เพื่อสร้าง token ของ `match` arm
**แต่ละอัน** ส่วนโค้ด**ที่ generate ออกมา**มี `match self { ... }` ซึ่งจะรันจริงตอน **runtime** ของ
โปรแกรมผู้ใช้ ทั้งสองเป็นคนละ "เวลา" กันโดยสิ้นเชิง — ระวังสับสนสองสิ่งนี้ตอนอ่านโค้ด macro (Part 44
เกริ่นเรื่องนี้ไว้สั้น ๆ แล้ว บทนี้เห็นภาพชัดขึ้นเพราะมี loop ซ้อน loop จริง ๆ)

**3) การจับคู่ arm กับ `Fields` variant ต้องตรงกันเป๊ะกับไวยากรณ์จริง** — สังเกตว่า `Fields::Named`
arm ต้อง generate pattern แบบ `Variant { a, b }` (มีวงเล็บปีกกา) ในขณะที่ `Fields::Unnamed` ต้อง
generate `Variant(a, b)` (วงเล็บกลม) และ `Fields::Unit` ไม่มีวงเล็บอะไรเลย ถ้าสลับกัน (เช่น generate
`Variant(a, b)` ให้ variant ที่จริง ๆ เป็น struct-like) จะได้ compile error จาก generated code ทันที
ซึ่งเราจะเห็นตัวอย่างจริงในหัวข้อกับดัก

**เรื่องที่ไม่ต้องกังวล: `enum` แบบ C-like ที่มี discriminant ชัดเจน** ทวนจาก Part 10 ว่า enum บางแบบ
กำหนดค่าตัวเลขให้ variant ตรง ๆ ได้ (เช่น `enum HttpStatus { Ok = 200, NotFound = 404 }`) `syn::Variant`
เก็บส่วนนี้ไว้ใน field `.discriminant: Option<(Token![=], Expr)>` (ค่า `Expr` หลังเครื่องหมาย `=` ถ้ามี) —
macro ของเราใน 45.4 **ไม่ได้แตะ `.discriminant` เลย** และนั่นถูกต้องแล้ว เพราะการ generate `match self
{ ... }` ไม่ได้สนใจว่า variant มีค่าตัวเลขกำกับไว้หรือไม่ (`match` จับคู่ตาม**ชื่อ**ของ variant เสมอ
ไม่ใช่ค่าตัวเลข) discriminant มีความหมายก็ต่อเมื่อคุณ cast enum เป็นตัวเลขด้วย `as i32` เท่านั้น ซึ่งเป็น
concern คนละเรื่องกับ macro ที่ generate โค้ดจาก field ของแต่ละ variant ไม่เกี่ยวข้องกัน — ถ้า derive
macro ของคุณต้องการอ่านค่า discriminant จริง ๆ (เช่น derive macro สำหรับแปลง enum เป็นค่าตัวเลข) จึงจะ
ต้องเข้าไปอ่าน `.discriminant` เพิ่ม ซึ่งอยู่นอกขอบเขตของบทนี้

### 45.5 รันจริง: ผลลัพธ์ของ `Describe` เวอร์ชันขยาย

มาดูผลลัพธ์จริงของทุกกรณีที่คุยกันมา (struct ทั้งสามแบบ + enum ผสม) รวมกันในโปรแกรมเดียว ก่อนจะไป
เพิ่มเรื่อง generics และ helper attribute ในหัวข้อถัดไป:

```rust
// describe_consumer/src/main.rs
use describe_derive::Describe;

// trait นี้อยู่ในฝั่ง consumer เหมือนใน Part 44 — ไม่ใช่ในตัว proc-macro crate เอง
// เพราะ crate ที่ตั้ง proc-macro = true จะ export ได้แค่ macro เท่านั้น จะ export
// item ธรรมดา (trait, struct, fn) จากมันไม่ได้เลย (crate ประเภทนี้ห้ามมี item ธรรมดาปนอยู่)
trait Describe {
    fn describe(&self) -> String;
}

#[derive(Describe)]
struct Point {
    x: i32,
    y: i32,
}

#[derive(Describe)]
struct Coord(f64, f64);

#[derive(Describe)]
struct Marker;

#[derive(Describe)]
enum Shape {
    Empty,
    Circle(f64),
    Rectangle { width: f64, height: f64 },
}

fn main() {
    let p = Point { x: 3, y: 4 };
    println!("{}", p.describe());

    let c = Coord(1.5, -2.5);
    println!("{}", c.describe());

    let m = Marker;
    println!("{}", m.describe());

    let shapes = vec![
        Shape::Empty,
        Shape::Circle(2.0),
        Shape::Rectangle { width: 3.0, height: 4.0 },
    ];
    for s in &shapes {
        println!("{}", s.describe());
    }
}
```

รันจริงด้วย `cargo run` ได้ผลลัพธ์นี้ (คัดลอกจาก terminal ตรง ๆ):

```
Point { x: 3, y: 4 }
Coord(1.5, -2.5)
Marker
Shape::Empty
Shape::Circle(2.0)
Shape::Rectangle { width: 3.0, height: 4.0 }
```

สังเกตว่า format ของ output ตรงกับ `#[derive(Debug)]` จริงของ standard library เกือบทุกจุด (มีความ
ตั้งใจ — เพราะ `Debug` ของ std ก็ต้องแก้ปัญหาเดียวกันกับที่เราแก้ทุกอย่างในหัวข้อนี้) นี่คือหลักฐาน
ที่ยืนยันว่าการแยก `Fields::Named`/`Fields::Unnamed`/`Fields::Unit` ให้ generate โค้ดคนละแบบ ทำงานได้
ถูกต้องครบทั้ง 6 กรณี (struct 3 แบบ + enum ที่มี variant 3 แบบผสมกัน) โดยไม่ต้อง panic หรือเดา

### 45.6 เมื่อ struct/enum มี generics ของตัวเอง

ทีนี้มาถึงจุดที่ derive macro ส่วนใหญ่ที่เขียนแบบมือใหม่พลาด: **type ที่ derive macro ถูกแปะไว้ อาจมี
generic parameter ของตัวเอง** เช่น

```rust
struct Wrapper<T> {
    value: T,
}
```

ลองคิดดูว่า `#[derive(Describe)]` ควร generate `impl` อะไรให้ `Wrapper<T>` — คำตอบที่ถูกคือ:

```rust
impl<T: std::fmt::Debug> Describe for Wrapper<T> {
    fn describe(&self) -> String {
        format!("Wrapper {{ value: {:?} }}", self.value)
    }
}
```

สังเกต **`<T: std::fmt::Debug>`** ตรงหลัง `impl` — นี่คือ trait bound ที่**เรา (คนเขียน macro) ต้อง
เพิ่มเข้าไปเอง** เหตุผลคือ: โค้ดข้างในใช้ `{:?}` กับ `self.value` ซึ่งมี type เป็น `T` (generic
parameter ที่มาจาก struct ต้นทาง) การใช้ `{:?}` (Debug formatting) กับค่าที่ type เป็น `T` แบบเปล่า ๆ
โดยไม่มี bound ใด ๆ **compile ไม่ผ่านแน่นอน** เพราะ compiler ไม่รู้ว่า `T` มี trait `Debug` implement
อยู่หรือไม่ (Part 22 สอนหลักการนี้ไว้แล้ว: "compiler ตรวจสอบ generic function/impl block **ตอนนิยาม**
ไม่ใช่ตอนใช้งานจริง (monomorphization)" — ต่างจาก C++ template ที่ตรวจตอน instantiate เท่านั้น)

ถ้าเราลืมเพิ่ม bound นี้ตอน generate `impl` (เขียนแค่ `impl<T> Describe for Wrapper<T> { ... }` เฉย ๆ
โดยไม่มี `T: Debug`) โค้ดที่ generate ออกมาจะ**compile ไม่ผ่านเลย ไม่ว่า `T` จะเป็น type อะไรก็ตาม**
แม้แต่ตอน derive กับ `Wrapper<i32>` ที่ `i32` implement `Debug` อยู่แล้วชัด ๆ ก็ตาม — เพราะ compiler
เช็ค bound ที่ประกาศไว้ตรง `impl<T> ...` เท่านั้น ไม่แคร์ว่าจริง ๆ แล้ว `T` จะถูก instantiate เป็น type
ไหนภายหลัง นี่คือบั๊กจริงที่เราจะจำลองและดู error message จริงในหัวข้อถัดไป

มาดูวิธีเพิ่ม bound อย่างถูกต้องด้วย `syn::Generics`:

```rust
/// เติม `T: std::fmt::Debug` ให้กับ generic type parameter ทุกตัวของ struct/enum ต้นทาง
/// เพราะโค้ดที่เรา generate ข้างในใช้ `{:?}` กับค่าที่มี type เป็น `T` ตรง ๆ
/// ถ้าไม่เติม bound นี้ impl block ที่ generate ออกมาจะไม่ compile
fn add_debug_bound(generics: &mut syn::Generics) {
    for param in generics.type_params_mut() {
        param.bounds.push(syn::parse_quote!(std::fmt::Debug));
    }
}
```

- **`generics.type_params_mut()`** — คืน iterator ของ generic type parameter ทั้งหมด (ไม่รวม
  lifetime parameter อย่าง `'a` หรือ const parameter อย่าง `const N: usize` — เพราะ trait bound
  แบบ `T: Debug` ใช้ได้แค่กับ type parameter เท่านั้น) แต่เป็นแบบ **mutable** เพื่อให้แก้ไข bound
  ของแต่ละตัวได้ตรง ๆ
- **`param.bounds.push(...)`** — `bounds` เป็น `Punctuated<TypeParamBound, Plus>` (รายการ bound
  คั่นด้วย `+` เช่น `T: Clone + Debug`) การ `push` เพิ่ม bound ใหม่เข้าไปโดยไม่ลบของเดิม ถ้า struct
  ต้นทางมี bound อยู่แล้ว (เช่น `struct Wrapper<T: Clone> { value: T }`) ผลลัพธ์จะกลายเป็น
  `T: Clone + std::fmt::Debug` โดยอัตโนมัติ (ไม่ overwrite ของเดิม)
- **`syn::parse_quote!(std::fmt::Debug)`** — เป็น macro จาก `syn` ที่ทำงานคล้าย `quote!` แต่**parse**
  token กลับเป็น syntax tree type ที่ต้องการ (ในที่นี้คือ `TypeParamBound`) ทันที แทนที่จะได้
  `TokenStream` เปล่า ๆ ที่ต้อง parse เองอีกที สะดวกมากเวลาต้องการ "เขียน syntax เป็นข้อความแล้วแปลง
  เป็น syntax tree node ทันที" แบบนี้

จากนั้นใน `expand_describe` เราต้องใช้ `Generics` ที่แก้ไขแล้วผ่าน `split_for_impl()`:

```rust
fn expand_describe(input: DeriveInput) -> syn::Result<TokenStream2> {
    let name = input.ident.clone();
    let body = /* ... (ตามหัวข้อ 45.3-45.4) ... */;

    let mut generics = input.generics.clone();
    add_debug_bound(&mut generics);
    let (impl_generics, ty_generics, where_clause) = generics.split_for_impl();

    let expanded = quote! {
        impl #impl_generics Describe for #name #ty_generics #where_clause {
            fn describe(&self) -> String {
                #body
            }
        }
    };

    Ok(expanded)
}
```

`Generics::split_for_impl()` คืน tuple สามชิ้นที่**ออกแบบมาเพื่อใช้ใน `impl` block โดยเฉพาะ**:

- **`impl_generics`** — ส่วนที่ตามหลัง `impl` เช่น `<T: std::fmt::Debug>` (รวม bound ทั้งหมดที่เราเพิ่ง
  เพิ่มเข้าไป)
- **`ty_generics`** — ส่วนที่ตามหลังชื่อ type เช่น `<T>` (**ไม่มี** bound เพราะตรงนี้แค่บอกว่า "type
  `Wrapper` ใช้กับ `T` ตัวไหน" ไม่ใช่ตอนประกาศ bound — การใส่ bound ซ้ำตรงนี้จะเป็น syntax error)
  สังเกตความต่างระหว่าง `impl<T: Debug> Describe for Wrapper<T>` — `<T: Debug>` (มี bound) ใช้
  `impl_generics`, `<T>` (ไม่มี bound) ใช้ `ty_generics`
- **`where_clause`** — ส่วน `where` clause ถ้ามี (เช่น `where T: SomeComplexBound` ที่ยาวเกินจะเขียน
  inline) ถ้าไม่มี `where` clause จะ render เป็นค่าว่าง ไม่ error

การมีฟังก์ชันสำเร็จรูปนี้ให้ใช้เป็นเหตุผลว่าทำไมไม่ควรพยายาม string-interpolate `impl<T> ... for
Name<T>` เองด้วยมือ (เช่น เขียน `quote! { impl<#generic_params> ... }` ตรง ๆ) — `split_for_impl()`
จัดการ edge case ยาก ๆ ให้หมดแล้ว (multiple type parameter, lifetime parameter ผสมกับ type parameter,
const generic, where clause ที่มีอยู่แล้ว ฯลฯ) การเขียนเองมีโอกาสพลาดสูงกว่ามาก

### 45.7 บั๊กจริง: ลืมเติม trait bound แล้ว compile fail เฉพาะตอนใช้กับ generic

มาดู error message จริงเมื่อลืมเรียก `add_debug_bound` (บั๊กที่พบบ่อยที่สุดในบทนี้ และเป็นเหตุผลที่
โจทย์กำหนดให้ต้องมีหัวข้อนี้แยกออกมาเลย) สมมติว่าคนเขียน macro เขียนแบบนี้โดยไม่ได้ตั้งใจ (ลืม call
ฟังก์ชันเติม bound ไปเฉย ๆ):

```rust
// เวอร์ชันที่มีบั๊ก — ลืมเรียก add_debug_bound(&mut generics)
fn expand_describe(input: DeriveInput) -> syn::Result<TokenStream2> {
    let name = input.ident.clone();
    let body = /* ... */;

    let generics = input.generics.clone();
    // ไม่ได้เรียก add_debug_bound(&mut generics) ตรงนี้!
    let (impl_generics, ty_generics, where_clause) = generics.split_for_impl();

    Ok(quote! {
        impl #impl_generics Describe for #name #ty_generics #where_clause {
            fn describe(&self) -> String { #body }
        }
    })
}
```

จุดสำคัญ: struct ธรรมดาที่ไม่มี generics (เช่น `Point`, `Coord`, `Marker`, `Shape` ในหัวข้อก่อนหน้า)
**ยัง compile ผ่านได้ปกติทุกอย่าง** เพราะ `generics.type_params_mut()` จะว่างเปล่า ไม่มี bound ให้
เพิ่มอยู่แล้ว บั๊กนี้จึง**นิ่งเงียบสนิทจนกว่าจะมีใครเอา macro ไปใช้กับ type ที่มี generic parameter**
— และนี่คือเหตุผลที่บั๊กประเภทนี้อันตรายในโลกจริงมาก: คนเขียน macro เทสกับ struct ธรรมดาผ่านหมด
มั่นใจว่า macro ใช้ได้ ก่อนจะ publish ขึ้น crates.io แล้วมีคน (หรืออนาคตของตัวเอง) เอาไปใช้กับ struct
ที่มี generics แล้วเจอ compile error ที่**ดูเหมือนไม่เกี่ยวกับโค้ดของตัวเองเลย**

ลองเอา macro เวอร์ชันบั๊กไปใช้กับ `Wrapper<T>`:

```rust
#[derive(Describe)]
struct Wrapper<T> {
    value: T,
}
```

ผลลัพธ์จริงจาก `cargo build` (คัดลอก error message ตรงจาก terminal):

```
error[E0277]: `T` doesn't implement `Debug`
  --> describe_consumer/src/main.rs:34:10
   |
34 | #[derive(Describe)]
   |          ^^^^^^^^ `T` cannot be formatted using `{:?}` because it doesn't implement `Debug`
   |
   = note: this error originates in the macro `$crate::__export::format_args` which comes from the expansion of the derive macro `Describe` (in Nightly builds, run with -Z macro-backtrace for more info)
help: consider restricting type parameter `T` with trait `Debug`
   |
35 | struct Wrapper<T: std::fmt::Debug> {
   |                 +++++++++++++++++

For more information about this error, try `rustc --explain E0277`.
error: could not compile `describe_consumer` (bin "describe_consumer") due to 1 previous error
```

สังเกตสามอย่างจาก error message นี้:

**1) error ชี้ไปที่ `#[derive(Describe)]` ไม่ใช่บรรทัดที่ใช้ `{:?}` จริง ๆ ในตัว macro** — เพราะ
`syn`/`quote` เก็บ span ของ token ที่ generate ไว้ให้ตรงกับตำแหน่งที่ผู้ใช้เขียน `#[derive(...)]`
(ไม่ใช่ตำแหน่งในไฟล์ `describe_derive/src/lib.rs` ที่เราเขียน `quote!` — เพราะผู้ใช้ macro มองไม่เห็น
และไม่ควรต้องรู้จักไฟล์นั้นเลย) นี่เป็นพฤติกรรม**ที่ถูกต้องและตั้งใจ**ของ proc macro ที่ดี — เทียบกับ
Part 44 ที่ error จาก `panic!`/`unwrap()` มักจะชี้ผิดตำแหน่งไปเลย

**2) rustc แนะนำวิธีแก้ให้ตรงเป๊ะ** — `help: consider restricting type parameter T with trait
Debug` พร้อม diff แนะนำให้เพิ่ม `T: std::fmt::Debug` — แต่ปัญหาคือ**นี่คือ struct ที่ผู้ใช้เขียน
ไม่ใช่โค้ดของ macro** ผู้ใช้อาจไม่รู้เลยว่าทำไม struct ธรรมดา ๆ ของตัวเองถึงต้องมี bound `Debug` (เพราะ
เขาไม่ได้ใช้ `Debug` ที่ไหนในโค้ดของเขาเองเลย — bound นี้จำเป็น**เพราะ macro ที่เขา derive ไปต่างหาก**)
นี่คือสิ่งที่ทำให้บั๊กนี้สร้างความสับสนให้ผู้ใช้ปลายทางได้มาก ถ้าคนเขียน macro ไม่จัดการให้ถูกต้องเอง
ตั้งแต่แรก ภาระจะตกไปที่ผู้ใช้ macro ที่ไม่รู้อีโหน่อีเหน่

**3) แก้บั๊กนี้ต้องแก้ที่ macro ไม่ใช่ที่ struct ของผู้ใช้** — วิธีแก้ที่ถูกต้องคือกลับไปเรียก
`add_debug_bound(&mut generics)` ในโค้ด macro (ตามหัวข้อ 45.6) **ไม่ใช่**บอกให้ผู้ใช้ macro ไปเพิ่ม
`T: Debug` เองตามที่ rustc แนะนำ (แม้จะแก้ปัญหาได้เหมือนกัน แต่นั่นคือการผลักภาระการดูแล generic bound
ที่ macro ควรจัดการเองไปให้ผู้ใช้ทุกคนที่ derive macro นี้ ซึ่งขัดกับเจตนาทั้งหมดของการเขียน derive
macro ตั้งแต่แรก — derive macro มีไว้เพื่อ**ลด**ภาระผู้ใช้ ไม่ใช่โยนภาระไปให้)

หลังจากเพิ่ม `add_debug_bound` กลับเข้าไป (โค้ดในหัวข้อ 45.6) แล้วรันใหม่:

```
Wrapper { value: 42 }
Wrapper { value: "hello" }
```

compile ผ่านและทำงานถูกต้องทั้งกับ `Wrapper<i32>` และ `Wrapper<String>` — ยืนยันว่าการเติม bound ให้
generated `impl` เองคือวิธีแก้ที่ถูกต้องและครบถ้วน (ไม่ต้องพึ่ง `T: Debug` ที่ struct ต้นทางเลย)

### 45.7.1 แล้ว lifetime parameter ล่ะ? (`struct Ref<'a, T> { value: &'a T }`)

ก่อนไปต่อ ควรตอบคำถามที่ค้างไว้: ถ้า struct มี **lifetime parameter** ปนกับ type parameter (ทวนจาก
Part 20/23) เช่น

```rust
#[derive(Describe)]
struct Ref<'a, T> {
    value: &'a T,
}
```

`add_debug_bound` (หัวข้อ 45.6) ต้องทำอะไรเพิ่มไหมกับ `'a`? คำตอบคือ**ไม่ต้อง** และนี่คือเหตุผลที่
ควรเข้าใจให้ชัด: `generics.type_params_mut()` คืนแค่ **type parameter** (`T`, `U`, ...) เท่านั้น ไม่รวม
**lifetime parameter** (`'a`, `'b`, ...) หรือ **const parameter** (`const N: usize`) เพราะ trait bound
แบบ `T: Debug` มีความหมายกับ type parameter เท่านั้น — lifetime ไม่มี "trait" ให้ bound แบบนั้น (มันมี
`'a: 'b` ซึ่งเป็นความสัมพันธ์ระหว่าง lifetime สองตัว คนละเรื่องกับ trait bound โดยสิ้นเชิง Part 23
อธิบายเรื่องนี้ไว้แล้ว)

field `value: &'a T` มี type เป็น**reference** (`&'a T`) ไม่ใช่ `T` ตรง ๆ — และ standard library มี
`impl<T: Debug> Debug for &T` ให้อยู่แล้ว (reference implement `Debug` ได้ทันทีถ้าตัวที่มันชี้ไปมี
`Debug`) ดังนั้น bound ที่เราเติมให้ `T` (`T: std::fmt::Debug`) ก็เพียงพอให้ `{:?}` ใช้กับ `self.value`
(ซึ่งมี type `&'a T`) ได้ทันที**โดยไม่ต้องเติมอะไรเกี่ยวกับ `'a` เลย** `split_for_impl()` (หัวข้อ 45.6)
จะนำ `'a` ไปวางไว้ใน `impl_generics`/`ty_generics` ให้ถูกตำแหน่งโดยอัตโนมัติอยู่แล้ว (เรียงตามลำดับที่
struct ต้นทางประกาศไว้ — ปกติ lifetime มาก่อน type parameter เสมอตามกฎไวยากรณ์ของ Rust) โดยที่โค้ด
macro ของเราไม่ต้องรู้จักหรือแยกแยะ `'a` ออกจาก `T` เลยแม้แต่นิดเดียว

ทดสอบจริง:

```rust
#[derive(Describe)]
struct Ref<'a, T> {
    value: &'a T,
}

fn main() {
    let x = 99;
    let r = Ref { value: &x };
    println!("{}", r.describe());
}
```

ผลลัพธ์จริง:

```
Ref { value: 99 }
```

compile ผ่านและทำงานถูกต้องทันที ยืนยันว่า `type_params_mut()` "มองข้าม" lifetime parameter ไปอย่าง
ถูกต้องโดยอัตโนมัติ — นี่คือเหตุผลที่ชื่อ method คือ `type_params_mut()` (เจาะจงว่า **type** parameter)
ไม่ใช่ `generic_params_mut()` แบบกว้าง ๆ ถ้า `syn` ออกแบบให้เราต้อง "เดา" ว่า generic parameter ตัวไหน
เป็น lifetime ตัวไหนเป็น type เอง จะเสี่ยงเขียนโค้ดผิดได้ง่ายกว่านี้มาก

### 45.8 Helper attributes: กลไกเบื้องหลัง `#[serde(rename = "...")]`

ถ้าคุณเคยใช้ `serde` (จะได้เจอเต็ม ๆ ใน Part 57-58) คุณเคยเห็นโค้ดแบบนี้:

```rust
#[derive(serde::Serialize)]
struct User {
    #[serde(rename = "user_name")]
    name: String,
    #[serde(skip)]
    password: String,
}
```

`#[serde(rename = "...")]` และ `#[serde(skip)]` **ไม่ใช่ attribute มาตรฐานของภาษา Rust** (ไม่เหมือน
`#[derive(...)]` หรือ `#[allow(...)]` ที่ compiler รู้จักเอง) มันคือ **helper attribute** — attribute
ที่ **derive macro เป็นคนประกาศว่ามีอยู่** แล้ว `syn` parse มันออกมาให้ macro นำไปใช้ตัดสินใจตอน
generate โค้ด กลไกทั้งหมดเริ่มจากบรรทัดเดียวตรง `#[proc_macro_derive(...)]`:

```rust
#[proc_macro_derive(Describe, attributes(describe))]
//                            ^^^^^^^^^^^^^^^^^^^^^ ประกาศ helper attribute ชื่อ "describe"
pub fn derive_describe(input: TokenStream) -> TokenStream {
    /* ... */
}
```

`attributes(describe)` บอก compiler ว่า "เวลามีคนแปะ `#[derive(Describe)]` ให้อนุญาตให้เขาแปะ
`#[describe(...)]` บน field/variant ต่าง ๆ ได้ด้วย โดยไม่ต้อง error ว่า attribute ไม่รู้จัก" — ถ้าไม่มี
บรรทัดนี้ compiler จะปฏิเสธ `#[describe(...)]` ทันทีตั้งแต่ก่อนที่ macro ของเราจะได้ทำงานด้วยซ้ำ (จะ
เห็น error จริงในหัวข้อกับดัก) สำคัญมาก: `attributes(describe)` แค่**เปิดสิทธิ์ให้ syntax นี้ผ่านการ
parse ได้** เท่านั้น — มันไม่ได้ทำอะไรกับ attribute นั้นอัตโนมัติ ตัว macro (โค้ดของเราเอง) ต้องไป
**parse ค่า attribute นั้นเองทั้งหมด** จาก `field.attrs: Vec<syn::Attribute>` ที่ `syn` เก็บไว้ให้ทุก
field/variant

เทียบกับ `serde`: `serde_derive` (proc-macro crate ของ serde) ประกาศ `attributes(serde)` แล้วภายใน
โค้ดของมันเอง parse `#[serde(rename = "...")]`, `#[serde(skip)]`, `#[serde(default)]` และอีกหลายสิบ
option ด้วยมือทั้งหมด — ไม่มีเวทมนตร์อะไรมากไปกว่ากลไกเดียวกันกับที่เรากำลังจะสร้างในหัวข้อถัดไป
เพียงแต่ `serde` รองรับ option เยอะกว่าและซับซ้อนกว่ามาก

### 45.9 สร้าง `#[describe(skip)]` ของเราเอง

มาสร้างเวอร์ชันง่ายของเราเองที่รองรับ helper attribute เดียว: `#[describe(skip)]` บน field ที่ต้องการ
ไม่ให้ปรากฏใน output ของ `describe()`

ก่อนอื่นต้องเขียนฟังก์ชันตรวจสอบว่า field มี attribute นี้อยู่หรือไม่:

```rust
/// ตรวจสอบว่า field มี attribute `#[describe(skip)]` หรือไม่
/// นี่คือกลไก "helper attribute" แบบเดียวกับที่ serde ใช้กับ #[serde(rename = "...")]
fn field_should_skip(attrs: &[syn::Attribute]) -> syn::Result<bool> {
    let mut skip = false;
    for attr in attrs {
        if !attr.path().is_ident("describe") {
            continue; // ไม่ใช่ attribute ของเรา ปล่อยผ่าน (อาจเป็น attribute ของ derive อื่น เช่น #[serde(...)])
        }
        attr.parse_nested_meta(|meta| {
            if meta.path.is_ident("skip") {
                skip = true;
                Ok(())
            } else {
                Err(meta.error("รู้จักแค่ #[describe(skip)] เท่านั้น"))
            }
        })?;
    }
    Ok(skip)
}
```

ไล่ทีละส่วน:

- **`field.attrs: Vec<syn::Attribute>`** — `syn::Field` (และ `syn::Variant`, `syn::DeriveInput` ด้วย)
  เก็บ attribute ทุกตัวที่แปะอยู่บน item นั้นไว้ใน field ชื่อ `attrs` เสมอ (attribute ในความหมายกว้าง
  รวมทั้ง `#[derive(...)]`, `#[doc = "..."]`, `#[serde(...)]`, และ `#[describe(...)]` ของเราทั้งหมด
  ปนกันมา — งานของเราคือกรองเอาแต่ตัวที่เกี่ยวกับเรา)
- **`attr.path().is_ident("describe")`** — เช็คว่า attribute นี้ชื่อ `describe` หรือไม่ (คือส่วนก่อน
  วงเล็บ `#[describe(...)]` เทียบเป็น string "describe") ถ้าไม่ตรงให้ `continue` ข้ามไปเฉย ๆ — จุดนี้
  สำคัญมาก เพราะ field หนึ่ง ๆ อาจมี attribute จากหลาย derive macro ปนกัน (เช่น `#[serde(skip)]` กับ
  `#[describe(skip)]` บน field เดียวกัน) macro ของเราต้อง**ไม่ไปแทรกแซง**attribute ของ macro อื่น
- **`attr.parse_nested_meta(|meta| { ... })`** — เป็น API ของ `syn` 2.x สำหรับ parse โครงสร้าง
  `#[name(key1, key2 = value2, ...)]` ที่อยู่ **ข้างใน**วงเล็บของ attribute โดย closure จะถูกเรียก
  หนึ่งครั้งต่อ "รายการ" ข้างใน (ในตัวอย่างนี้มีแค่ `skip` รายการเดียว) `meta.path` คือชื่อของรายการนั้น
  (`skip`) — ถ้า `serde` ต้องการ parse `rename = "user_name"` ก็จะเช็ค `meta.path.is_ident("rename")`
  แล้วเรียก `meta.value()?` เพื่อดึงค่าฝั่งขวาของ `=` ต่อ (มีตัวอย่างแบบเต็มในหัวข้อ 45.13 ที่เราจะเห็น
  การใช้ attribute ที่ซับซ้อนกว่าอีกแบบหนึ่ง — ไม่ใช่ในบทนี้ แต่แนวคิดเดียวกัน)
- **การคืน `Err(meta.error(...))` เมื่อเจอ key ที่ไม่รู้จัก** — ถ้าผู้ใช้เขียนพลาดเป็น
  `#[describe(skpi)]` (สลับตัวอักษร) macro ของเราจะคืน error ที่ชี้ตำแหน่งไปที่ `skpi` เป๊ะ พร้อมข้อความ
  ชัดเจนว่า "รู้จักแค่ skip เท่านั้น" — ดีกว่าการเงียบ ๆ ปล่อยผ่านไปเฉย ๆ (ซึ่งจะทำให้ผู้ใช้คิดว่า
  `skip` ทำงานทั้งที่จริง ๆ พิมพ์ผิดและไม่มีผลอะไรเลย)

จากนั้นเอาฟังก์ชันนี้ไปใช้ตอน generate โค้ดจริง (ตัวอย่างสำหรับ `Fields::Named` ของ struct):

```rust
Fields::Named(fields) => {
    let mut format_parts = Vec::new();
    let mut format_args = Vec::new();
    for field in &fields.named {
        if field_should_skip(&field.attrs)? {
            continue; // ข้าม field นี้ไปเลย ไม่เอาไปแสดงใน describe()
        }
        let field_ident = field.ident.clone().unwrap();
        format_parts.push(format!("{}: {{:?}}", field_ident));
        format_args.push(quote! { self.#field_ident });
    }
    // ... (ต่อด้วย fmt_string เหมือนหัวข้อ 45.3)
}
```

ทดสอบจริง:

```rust
#[derive(Describe)]
struct User {
    username: String,
    #[describe(skip)]
    password: String,
}

fn main() {
    let u = User {
        username: "gapper".to_string(),
        password: "super-secret".to_string(),
    };
    println!("{}", u.describe());
}
```

ผลลัพธ์จริงจาก `cargo run`:

```
User { username: "gapper" }
```

`password` หายไปจาก output ตามที่ตั้งใจ — และสิ่งสำคัญคือ field `password` **ยังมีอยู่จริงใน struct**
(ยัง compile และเก็บค่าได้ปกติทุกอย่าง) เราแค่ควบคุมว่า `describe()` ควร**แสดง**field ไหนเท่านั้น
ไม่ได้เปลี่ยนโครงสร้างของ struct จริงแต่อย่างใด — หลักการเดียวกันนี้คือสิ่งที่ทำให้
`#[serde(skip)]` ใช้งานได้: field ยังอยู่ใน struct ปกติ แค่ `Serialize`/`Deserialize` ที่ generate
ออกมาไม่ไปแตะมันเท่านั้นเอง

### 45.10 Error handling ระดับ production: `syn::Error` กับ span ที่ถูกต้อง

Part 44 แนะนำการใช้ `syn::Error`/`.to_compile_error()` ไว้สั้น ๆ ตอนพูดถึงการจัดการ error พื้นฐาน
บทนี้จะขยายความให้ครบเป็น pattern ที่ใช้งานได้จริงในทุกจุดของ macro ไม่ใช่แค่จุดเดียว

หลักการสำคัญที่สุดของ error handling ใน proc macro คือ**ห้าม panic เด็ดขาดในโค้ด production** ให้
ฟังก์ชันภายในทุกตัวคืน `syn::Result<T>` (คือ `Result<T, syn::Error>`) แทน แล้วให้ฟังก์ชันที่เป็น entry
point ของ macro (ตัวที่มี `#[proc_macro_derive(...)]`) เป็นจุดเดียวที่แปลง `Result` เป็น `TokenStream`
สุดท้าย:

```rust
#[proc_macro_derive(Describe, attributes(describe))]
pub fn derive_describe(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);

    match expand_describe(input) {
        Ok(tokens) => tokens.into(),
        Err(err) => err.to_compile_error().into(),
        //          ^^^^^^^^^^^^^^^^^^^^^^^ แปลง syn::Error เป็น TokenStream ของ compile_error!(...)
    }
}

fn expand_describe(input: DeriveInput) -> syn::Result<TokenStream2> {
    // ทุกจุดที่เคย panic!/unwrap() ใน Part 44 ให้เปลี่ยนเป็น return Err(...) แทน
    // ...
}
```

`syn::Error::to_compile_error()` แปลง error เป็น `TokenStream` ที่มีแค่ macro call เดียวข้างใน:
`compile_error!("ข้อความ error ของเรา")` — `compile_error!` เป็น macro มาตรฐานของ Rust ที่ทำให้
compiler แสดง error message ตรง ๆ ตาม string ที่ให้ **โดยไม่ต้อง panic ตัว proc macro process เลย**
proc macro คืน token stream ที่ compile ไม่ผ่านแบบ "สุภาพ" กลับไปให้ compiler จัดการรายงาน error ต่อ
เอง เทียบกับการ panic ที่ทำให้ proc macro **process ทั้งตัวล่มลง** ซึ่ง compiler จะรายงานเป็น "proc-
macro derive panicked" แทน (ดูตัวอย่างเทียบกันจริงในหัวข้อ 45.11)

ส่วนที่สำคัญไม่แพ้กันคือการเลือก **span** ให้ error message ชี้ตำแหน่งถูก ลองดูสามวิธีสร้าง
`syn::Error` เทียบกัน:

```rust
// วิธีที่ 1: ใช้ Span::call_site() — ชี้ไปที่ตำแหน่งที่ #[derive(...)] ถูกเรียก (กว้างสุด)
syn::Error::new(proc_macro2::Span::call_site(), "ข้อความ error")

// วิธีที่ 2: ใช้ span ของ syntax tree node ใดก็ได้ที่มีอยู่แล้ว (แม่นยำกว่า)
syn::Error::new_spanned(&data_union.union_token, "ข้อความ error")

// วิธีที่ 3: ใช้ span จาก parse_nested_meta ที่ syn จับตำแหน่งของ token ผิดพลาดให้อัตโนมัติ
meta.error("ข้อความ error") // ภายในคือ syn::Error::new(meta.path.span(), ...)
```

`syn::Error::new_spanned(node, msg)` คือตัวที่ควรใช้บ่อยที่สุด เพราะมันดึง `Span` จาก **node ของ
syntax tree ที่เรามีอยู่แล้วในมือ** (เช่น `union_token` ที่บอกตำแหน่งของ keyword `union` เป๊ะ ๆ) แทนที่
จะใช้ `Span::call_site()` ที่แม่นยำน้อยกว่า (ชี้ไปที่ตำแหน่งของ `#[derive(...)]` ทั้งก้อน ไม่ใช่จุดที่
ปัญหาเกิดจริง) — หลักการทั่วไป: **ยิ่ง span แคบและตรงจุดเท่าไหร่ ผู้ใช้ macro ยิ่งแก้ปัญหาได้เร็วเท่านั้น**
เพราะ IDE/editor จะขีดเส้นใต้สีแดงตรงตำแหน่งที่ span ชี้ ไม่ใช่ทั้งบรรทัด

### 45.10.1 รายงาน error หลายจุดพร้อมกันด้วย `syn::Error::combine`

โค้ดทุกจุดที่เราเขียนมาจนถึงตอนนี้ใช้ `?` เพื่อ**หยุดทันทีที่เจอ error จุดแรก** (early return) ซึ่งเพียง
พอสำหรับกรณีทั่วไป แต่ลองนึกภาพสถานการณ์นี้: struct หนึ่งมี field ที่เขียน `#[describe(...)]` ผิดอยู่
**สองจุดพร้อมกัน** ถ้า macro หยุดรายงานแค่จุดแรกที่เจอ ผู้ใช้จะต้องแก้ทีละจุด compile ใหม่ แก้อีกจุด
compile ใหม่อีกรอบ — ช้ากว่าที่ควรจะเป็นมาก `syn::Error` มี method `.combine(other)` ที่รวม error
หลายตัวเข้าด้วยกันเป็นก้อนเดียว โดยที่ compiler ยังคง**รายงานทุกจุดพร้อมกันในการ compile ครั้งเดียว**:

```rust
Fields::Named(fields) => {
    let mut format_parts = Vec::new();
    let mut format_args = Vec::new();
    let mut collected_error: Option<syn::Error> = None;

    for field in &fields.named {
        let skip = match field_should_skip(&field.attrs) {
            Ok(skip) => skip,
            Err(err) => {
                // เจอ error ที่ field นี้ — สะสมไว้ก่อน ไม่ return ทันที เพื่อให้ตรวจ field
                // ที่เหลือต่อไปได้ครบ (จะได้เจอ error อื่น ๆ ที่อาจซ่อนอยู่ในรอบเดียวกัน)
                match &mut collected_error {
                    Some(existing) => existing.combine(err),
                    None => collected_error = Some(err),
                }
                continue;
            }
        };
        if skip {
            continue;
        }
        let field_ident = field.ident.clone().unwrap();
        format_parts.push(format!("{}: {{:?}}", field_ident));
        format_args.push(quote! { self.#field_ident });
    }

    // ตรวจ error ที่สะสมไว้ทั้งหมดหลัง loop จบ — ถ้ามี ให้ return ออกไปทีเดียว
    if let Some(err) = collected_error {
        return Err(err);
    }

    let fmt_string = /* ... เหมือนหัวข้อ 45.3 ... */;
    Ok(quote! { format!(#fmt_string, #(#format_args),*) })
}
```

ทดสอบด้วย struct ที่เขียน helper attribute ผิดสองจุดพร้อมกัน:

```rust
#[derive(Describe)]
struct Config {
    #[describe(bad_key1)]
    host: String,
    #[describe(bad_key2)]
    port: u16,
}

fn main() {}
```

ผลลัพธ์จริงจาก `cargo build` (คัดลอกจาก terminal ตรง ๆ — สังเกตว่า error ทั้งสองจุดปรากฏพร้อมกันใน
การ compile ครั้งเดียว):

```
error: รู้จักแค่ #[describe(skip)] เท่านั้น
 --> src/main.rs:9:16
  |
9 |     #[describe(bad_key1)]
  |                ^^^^^^^^

error: รู้จักแค่ #[describe(skip)] เท่านั้น
  --> src/main.rs:11:16
   |
11 |     #[describe(bad_key2)]
   |                ^^^^^^^^

error: could not compile `describe_consumer` (bin "describe_consumer") due to 2 previous errors
```

ผู้ใช้เห็นทั้งสองจุดที่ต้องแก้ในการ compile ครั้งเดียว ไม่ต้องแก้ทีละจุดแล้ว compile ซ้ำหลายรอบ — นี่คือ
รายละเอียดเล็ก ๆ ที่ทำให้ experience ของผู้ใช้ macro ดีขึ้นอย่างเห็นได้ชัด และเป็นเทคนิคที่ `serde_derive`
ใช้จริงเวลามีคน parse struct ที่มี attribute ผิดหลายจุดพร้อมกัน (ลองดูตัวเองได้ด้วยการเขียน
`#[serde(rename = )]` ผิด syntax สองจุดในไฟล์เดียว แล้วสังเกตว่า `rustc` รายงานทั้งสอง error พร้อมกัน)
หลักการเลือกใช้: ใช้ `?` (early return) เมื่อ error จุดแรกทำให้ตรวจต่อไม่ได้อยู่แล้ว (เช่น input ไม่ใช่
struct เลย) แต่ใช้ `combine` เมื่อ error แต่ละจุด**เป็นอิสระจากกัน** (เช่น ตรวจ field ทีละตัวในลูปเดียวกัน
แบบนี้) เพื่อให้ผู้ใช้เห็นทุกปัญหาที่ตรวจพบได้ในรอบเดียว

### 45.11 panic vs compile error: เทียบให้เห็นจริง

มาดูความต่างระหว่างสองแนวทางนี้แบบเห็นภาพจริง ด้วยกรณี `union` (ที่ `Describe` ไม่รองรับ) เป็นตัวอย่าง

**เวอร์ชันแย่ (panic แบบดิบ ๆ)** — สมมติว่าคนเขียน macro ปล่อยให้ตกไปที่ `unreachable!()` เมื่อเจอ
`Data::Union` (บั๊กแบบที่ Part 44 เตือนไว้สั้น ๆ ว่าไม่ควรทำ):

```rust
let body = match &input.data {
    Data::Struct(data_struct) => describe_struct_body(&name, data_struct)?,
    Data::Enum(data_enum) => describe_enum_body(&name, data_enum)?,
    Data::Union(_) => unreachable!("ยังไม่รองรับ union"),
};
```

ผู้ใช้ที่เผลอ derive กับ union:

```rust
#[derive(Describe)]
union Payload {
    i: i32,
    f: f32,
}
```

ผลลัพธ์จริงจาก `cargo build`:

```
error: proc-macro derive panicked
 --> describe_consumer/src/main.rs:7:10
  |
7 | #[derive(Describe)]
  |          ^^^^^^^^
  |
  = help: message: internal error: entered unreachable code: ยังไม่รองรับ union

error: could not compile `describe_consumer` (bin "describe_consumer") due to 1 previous error
```

สังเกตว่า error message เป็น "internal error: entered unreachable code" — ข้อความนี้**บอกผู้ใช้ macro
ว่ามันคือบั๊กภายในของ macro** (คำว่า "internal error" ฟังดูน่ากลัวและทำให้ผู้ใช้คิดว่า macro พัง ไม่ใช่
ว่าตัวเองใช้ผิด) ไม่มีคำแนะนำว่าควรทำอย่างไรต่อ ไม่มีข้อมูลว่า `Describe` ไม่รองรับ `union` โดยเจตนา
ผู้ใช้ต้องไปเปิดซอร์สโค้ดของ macro เองเพื่อเข้าใจว่าเกิดอะไรขึ้น

**เวอร์ชันดี (compile error แบบมีเจตนา)** — ใช้ `syn::Error::new_spanned` ตามหัวข้อ 45.10:

```rust
Data::Union(data_union) => {
    return Err(syn::Error::new_spanned(
        data_union.union_token,
        "#[derive(Describe)] ไม่รองรับ union เพราะ union ไม่การันตีว่า field ใด \
         กำลัง active อยู่ ณ runtime การอ่าน field แบบ field-by-field จึงไม่ปลอดภัย \
         (ต้องใช้ unsafe และรู้ tag เองถึงจะอ่านได้ ซึ่งไม่ใช่สิ่งที่ derive macro ทั่วไปควรเดาแทนผู้ใช้)",
    ));
}
```

ผลลัพธ์จริงกับ union เดิม:

```
error: #[derive(Describe)] ไม่รองรับ union เพราะ union ไม่การันตีว่า field ใด กำลัง active อยู่ ณ runtime การอ่าน field แบบ field-by-field จึงไม่ปลอดภัย (ต้องใช้ unsafe และรู้ tag เองถึงจะอ่านได้ ซึ่งไม่ใช่สิ่งที่ derive macro ทั่วไปควรเดาแทนผู้ใช้)
 --> describe_consumer/src/main.rs:8:1
  |
8 | union Payload {
  | ^^^^^

error: could not compile `describe_consumer` (bin "describe_consumer") due to 1 previous error
```

เทียบกันชัด ๆ:

| | panic แบบดิบ | `syn::Error` + `to_compile_error()` |
|---|---|---|
| ประเภท error ที่ compiler แสดง | `error: proc-macro derive panicked` | `error: <ข้อความของเรา>` |
| span ที่ชี้ | ตำแหน่ง `#[derive(Describe)]` เท่านั้น | ตำแหน่ง `union` keyword ตรง ๆ (แม่นยำกว่า) |
| ข้อความอธิบาย | ไม่มี บอกแค่ตำแหน่งใน source ของ macro เอง | อธิบายเหตุผล + เจตนาการออกแบบครบ |
| ความรู้สึกของผู้ใช้ | "macro นี้พัง/มี bug" | "ฉันใช้ macro นี้ผิดวิธี รู้แล้วว่าต้องแก้ยังไง" |
| debug ต่อได้ไหม | ต้องเปิด source ของ macro เอง | ไม่ต้องเปิดอะไรเพิ่ม อ่านจบในที่เดียว |

นี่คือความต่างเดียวกันกับที่ Part 12 (`?` operator) และ Part 30 (custom error types) สอนไว้ในระดับ
runtime error (`panic!` vs `Result<T, E>`) — proc macro ก็มีหลักการเดียวกันแบบเป๊ะ ๆ เพียงแต่ "runtime"
ของ proc macro คือ**ตอน compile time ของโปรแกรมผู้ใช้** ไม่ใช่ตอนโปรแกรมรันจริง `panic!` ในโค้ด macro
ทำให้**process ของ compiler ล่มระหว่าง compile** (แม้จะ catch ได้และไม่ทำให้ compiler เองพังจริง ๆ
แต่ก็เป็นสัญญาณว่ามีอะไรผิดปกติที่ไม่ได้ตั้งใจ) ในขณะที่ `syn::Error` เป็นการรายงาน "นี่คือการใช้งาน
ที่ไม่ถูกต้องซึ่งฉันตรวจพบและออกแบบมาให้ปฏิเสธอย่างสุภาพ"

### 45.12 การทดสอบ proc macro: ทำไม `#[test]` ธรรมดาไม่พอ และ `trybuild` เข้ามาช่วยยังไง

Part 32-33 สอนการเขียน `#[test]`/integration test สำหรับโค้ด Rust ทั่วไปไว้อย่างละเอียด และ Part 44
หัวข้อ 44.6.1 ก็แสดงให้เห็นแล้วว่าเราเขียน `#[test]` ทดสอบ `expand_describe()` ได้จริง (โดยแยก entry
point ที่ "บาง" ออกจาก logic ที่ "หนา" ตามที่ Part 44 สอนไว้) ด้วยการสร้าง `DeriveInput` จาก
`syn::parse_str::<DeriveInput>("struct Customer { ... }")` ตรง ๆ แล้วเช็คว่า
`expand_describe(input).to_string()` มี substring ที่ควรมีอยู่จริง (เช่น `.contains("impl Customer")`)
— เทคนิคนี้ใช้งานได้จริงและเราใช้แนวทางเดียวกันในการพัฒนา `describe_derive`/`builder_derive` ของบทนี้
เองตลอดทั้งบท

แต่เทคนิคนั้นตอบได้แค่คำถามว่า **"token ที่ generate ออกมาหน้าตาถูกไหม"** (มี string ที่ควรมีอยู่จริง
หรือไม่) มันตอบ**ไม่ได้**อีกคำถามที่สำคัญไม่แพ้กัน คือ **"เอา token นี้ไป compile จริงแล้วมันผ่านไหม
และถ้าควร fail (เช่น กรณี `union`) มันควร fail ด้วย error message ที่ถูกต้องเป๊ะหรือไม่"** เหตุผลคือ
`expand_describe()` ทำงานบน `proc_macro2::TokenStream` ล้วน ๆ (ตามที่ Part 44 อธิบายไว้) ซึ่งเป็นแค่
"ข้อมูล" ในหน่วยความจำ — การจะรู้ว่ามันคอมไพล์ผ่านจริงหรือไม่ ต้องเอา token นั้นไปเขียนเป็นไฟล์ `.rs`
แล้วเรียก `rustc` แยกกระบวนการอีกรอบเสมอ ซึ่ง `#[test]` ธรรมดาไม่ได้ทำสิ่งนี้ให้อัตโนมัติ

นี่คือช่องว่างที่ทำให้ ecosystem proc macro ของ Rust สร้างเครื่องมือเฉพาะทางขึ้นมา ที่ได้รับความนิยม
สูงสุดคือ **`trybuild`** (สร้างโดย David Tolnay ผู้เขียน `syn`/`quote`/`serde` เองด้วย) หลักการทำงาน
ของ `trybuild` คือ: เขียนไฟล์ `.rs` ตัวอย่างเล็ก ๆ แยกไว้ (เรียกว่า "UI test case") แล้วให้ `trybuild`
เรียก `rustc` compile ไฟล์นั้นแยกกระบวนการจริง ๆ จากนั้นเช็คว่าผลลัพธ์ตรงกับที่คาดไว้หรือไม่ — แบ่งเป็น
สองแบบ:

- **`t.pass("path/to/file.rs")`** — คาดว่าไฟล์นี้ต้อง compile ผ่าน (ใช้ยืนยันว่า use case ที่ควรทำงาน
  ได้ ยังทำงานได้อยู่ — ป้องกัน regression)
- **`t.compile_fail("path/to/file.rs")`** — คาดว่าไฟล์นี้ต้อง compile **ไม่ผ่าน** และถ้ามีไฟล์
  `.stderr` คู่กันอยู่ (ชื่อเดียวกันแต่นามสกุล `.stderr`) จะเทียบ error message ที่ได้กับเนื้อหาในไฟล์
  นั้นด้วย (ใช้ยืนยันว่า error message ยังคงเป็นแบบที่ตั้งใจ ไม่ได้เปลี่ยนไปโดยไม่รู้ตัวตอนแก้โค้ด macro
  ในอนาคต)

โครงสร้างไฟล์ทั่วไปสำหรับใช้ `trybuild`:

```
describe_derive/
├── src/
│   └── lib.rs
└── tests/
    ├── ui.rs                       <- ไฟล์ที่มี #[test] เรียก trybuild
    └── trybuild_cases/
        ├── named_struct_ok.rs      <- ตัวอย่างที่ควร compile ผ่าน
        ├── union_fails.rs          <- ตัวอย่างที่ควร compile ไม่ผ่าน
        └── union_fails.stderr      <- error message ที่คาดไว้ (สร้างอัตโนมัติได้)
```

```toml
# describe_derive/Cargo.toml
[dev-dependencies]
trybuild = "1"
```

```rust
// describe_derive/tests/ui.rs
#[test]
fn ui() {
    let t = trybuild::TestCases::new();
    t.pass("tests/trybuild_cases/named_struct_ok.rs");
    t.compile_fail("tests/trybuild_cases/union_fails.rs");
}
```

```rust
// describe_derive/tests/trybuild_cases/named_struct_ok.rs
use describe_derive::Describe;

trait Describe {
    fn describe(&self) -> String;
}

#[derive(Describe)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p = Point { x: 1, y: 2 };
    assert_eq!(p.describe(), "Point { x: 1, y: 2 }");
}
```

```rust
// describe_derive/tests/trybuild_cases/union_fails.rs
use describe_derive::Describe;

trait Describe {
    fn describe(&self) -> String;
}

#[derive(Describe)]
union Payload {
    i: i32,
    f: f32,
}

fn main() {}
```

รันด้วย `cargo test` ตามปกติ (Part 32 สอนไว้แล้ว) `trybuild` จะจัดการเรียก `rustc` แยกกระบวนการให้ทุก
ไฟล์ที่ระบุ พร้อมสรุปผล pass/fail รวมเป็น test เดียว การรันครั้งแรกกับไฟล์ที่ยังไม่มี `.stderr` คู่กัน
(`union_fails.stderr`) สามารถสั่งให้ `trybuild` สร้างไฟล์นั้นให้อัตโนมัติจาก error จริงที่ได้ ด้วย
environment variable `TRYBUILD=overwrite cargo test` — เมื่อตรวจดูแล้วว่า error message ที่ได้ถูกต้อง
ตามที่ตั้งใจ ก็ commit ไฟล์ `.stderr` นั้นเข้า repository เพื่อใช้เทียบในการรันครั้งต่อ ๆ ไป

**ข้อจำกัดที่ควรรู้ (ระดับความเข้าใจ ไม่ต้องลงลึก):** `trybuild` เรียก `rustc` แยกกระบวนการจริงในทุก
test case ทำให้ test suite ที่มี case จำนวนมากใช้เวลารันนานกว่า unit test ปกติมาก (วินาทีต่อ case
เทียบกับมิลลิวินาที) และ error message ของ `rustc` อาจเปลี่ยนแปลงเล็กน้อยระหว่าง Rust version ทำให้
ไฟล์ `.stderr` ต้องอัปเดตเป็นระยะเมื่อ Rust รุ่นใหม่เปลี่ยนคำเตือน/ข้อความ error (ไม่ใช่เพราะ macro
ของเราพัง) โปรเจกต์ production อย่าง `serde`, `thiserror`, `async-trait` ใช้ `trybuild` แบบนี้จริงใน
CI ของตัวเอง — ถ้าคุณเปิด repository ของ `thiserror` บน GitHub จะเห็นโฟลเดอร์ `tests/ui/` ที่มีไฟล์
`.rs`/`.stderr` เป็นสิบ ๆ คู่ ทำงานตามหลักการเดียวกับที่อธิบายมาทั้งหมดนี้

### 45.13 Case study: `#[derive(Builder)]` ตัวเต็ม

มาถึง case study ใหญ่ของบทนี้ — สร้าง `#[derive(Builder)]` ที่ generate builder pattern แบบเต็ม
รูปแบบให้ struct ใด ๆ ที่มี named field ได้จริง (Part 52 จะสอน builder pattern แบบเขียนมือทีละ field
บทนี้สอนวิธี**generate**โค้ดแบบนั้นให้อัตโนมัติ)

**เป้าหมาย:** จาก struct ธรรมดา

```rust
#[derive(Builder)]
pub struct Config {
    pub host: String,
    pub port: u16,
    pub timeout_ms: u64,
    pub tags: Vec<String>,
}
```

อยาก generate โค้ดที่ทำให้เขียนแบบนี้ได้:

```rust
let config = Config::builder()
    .host("localhost".to_string())
    .port(8080)
    .timeout_ms(3000)
    .tags(vec!["prod".to_string()])
    .build()?; // Result<Config, String> — Err ถ้ามี field ที่ยังไม่ set
```

**การออกแบบโค้ดที่ต้อง generate:** ก่อนเขียน macro ควรเขียนโค้ดเป้าหมายด้วยมือก่อนเสมอ (เทคนิคนี้
ช่วยได้มากตอนออกแบบ macro ที่ generate โค้ดซับซ้อน — เขียนโค้ดที่อยาก**ได้**ก่อน แล้วค่อยเขียน macro
ที่ generate โค้ดแบบนั้น):

```rust
pub struct ConfigBuilder {
    host: Option<String>,
    port: Option<u16>,
    timeout_ms: Option<u64>,
    tags: Option<Vec<String>>,
}

impl Config {
    pub fn builder() -> ConfigBuilder {
        ConfigBuilder { host: None, port: None, timeout_ms: None, tags: None }
    }
}

impl ConfigBuilder {
    pub fn host(mut self, value: String) -> Self {
        self.host = Some(value);
        self
    }
    pub fn port(mut self, value: u16) -> Self {
        self.port = Some(value);
        self
    }
    pub fn timeout_ms(mut self, value: u64) -> Self {
        self.timeout_ms = Some(value);
        self
    }
    pub fn tags(mut self, value: Vec<String>) -> Self {
        self.tags = Some(value);
        self
    }

    pub fn build(self) -> Result<Config, String> {
        Ok(Config {
            host: self.host.ok_or_else(|| "missing field `host`".to_string())?,
            port: self.port.ok_or_else(|| "missing field `port`".to_string())?,
            timeout_ms: self.timeout_ms.ok_or_else(|| "missing field `timeout_ms`".to_string())?,
            tags: self.tags.ok_or_else(|| "missing field `tags`".to_string())?,
        })
    }
}
```

สังเกตโครงสร้าง 3 ส่วนที่ต้อง generate: (1) struct `ConfigBuilder` ที่ทุก field ห่อด้วย `Option<T>`,
(2) `Config::builder()` ที่สร้าง `ConfigBuilder` เริ่มต้นด้วยทุก field เป็น `None`, (3) setter method
หนึ่งตัวต่อ field ที่คืน `Self` (เพื่อ chain แบบ builder pattern ได้ — Part 24 พูดถึงหลักการ "method
ที่คืน `Self` เพื่อ chain" ไว้บ้างแล้วตอนคุยเรื่อง iterator adapter) และ `build()` ที่แปลงทุก
`Option<T>` กลับเป็น `T` พร้อมเช็คว่าไม่มี field ไหนเป็น `None` หลงเหลือ

**เขียน macro ที่ generate โค้ดข้างบนทั้งหมด:**

```rust
// builder_derive/src/lib.rs
use proc_macro::TokenStream;
use proc_macro2::TokenStream as TokenStream2;
use quote::{format_ident, quote};
use syn::{parse_macro_input, punctuated::Punctuated, token::Comma, Data, DeriveInput, Field, Fields};

#[proc_macro_derive(Builder)]
pub fn derive_builder(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    match expand_builder(input) {
        Ok(tokens) => tokens.into(),
        Err(err) => err.to_compile_error().into(),
    }
}

fn expand_builder(input: DeriveInput) -> syn::Result<TokenStream2> {
    let struct_name = &input.ident;
    let builder_name = format_ident!("{}Builder", struct_name);

    // ตรวจสอบก่อนว่าเป็น struct ที่มี named field เท่านั้น — ถ้าไม่ใช่ ให้ compile error ที่อธิบาย
    // เหตุผลไว้ตรง ๆ (ตามแนวทาง error handling ในหัวข้อ 45.10) แทนการปล่อยให้พังแบบงง ๆ ทีหลัง
    let fields: &Punctuated<Field, Comma> = match &input.data {
        Data::Struct(data_struct) => match &data_struct.fields {
            Fields::Named(fields) => &fields.named,
            Fields::Unnamed(_) => {
                return Err(syn::Error::new_spanned(
                    &input.ident,
                    "#[derive(Builder)] รองรับเฉพาะ struct ที่มี named fields เท่านั้น \
                     (tuple struct ไม่มีชื่อ field ให้ตั้งชื่อ setter method ให้ได้)",
                ));
            }
            Fields::Unit => {
                return Err(syn::Error::new_spanned(
                    &input.ident,
                    "#[derive(Builder)] ใช้กับ unit struct ไม่ได้ เพราะไม่มี field ให้ build",
                ));
            }
        },
        Data::Enum(data_enum) => {
            return Err(syn::Error::new_spanned(
                data_enum.enum_token,
                "#[derive(Builder)] รองรับเฉพาะ struct เท่านั้น ไม่รองรับ enum",
            ));
        }
        Data::Union(data_union) => {
            return Err(syn::Error::new_spanned(
                data_union.union_token,
                "#[derive(Builder)] ไม่รองรับ union",
            ));
        }
    };

    let field_idents: Vec<_> = fields.iter().map(|f| f.ident.clone().unwrap()).collect();
    let field_types: Vec<_> = fields.iter().map(|f| f.ty.clone()).collect();

    // struct XxxBuilder { field1: Option<T1>, field2: Option<T2>, ... }
    let builder_fields = field_idents
        .iter()
        .zip(&field_types)
        .map(|(ident, ty)| quote! { #ident: Option<#ty> });

    // ค่าเริ่มต้นทุก field เป็น None ตอนสร้าง Builder ใหม่
    let builder_defaults = field_idents.iter().map(|ident| quote! { #ident: None });

    // setter method หนึ่งตัวต่อ field: รับ value ตรง type ของ field คืน Self เพื่อ chain ได้
    let setters = field_idents.iter().zip(&field_types).map(|(ident, ty)| {
        quote! {
            pub fn #ident(mut self, value: #ty) -> Self {
                self.#ident = Some(value);
                self
            }
        }
    });

    // ตอน build: ถ้า field ไหนยังเป็น None ให้คืน Err ระบุชื่อ field ที่ขาดไปตรง ๆ
    let build_fields = field_idents.iter().map(|ident| {
        let missing_msg = format!("missing field `{}`", ident);
        quote! {
            #ident: self.#ident.ok_or_else(|| #missing_msg.to_string())?
        }
    });

    // นำ generics ของ struct ต้นทางมาใส่ในทั้ง struct Builder และ impl block ทุกตัว
    // (ไม่ต้องเติม bound เพิ่มเองแบบ Describe เพราะ Option<T> ใช้ได้กับทุก T โดยไม่มีเงื่อนไข)
    let generics = &input.generics;
    let (impl_generics, ty_generics, where_clause) = generics.split_for_impl();

    let expanded = quote! {
        pub struct #builder_name #impl_generics #where_clause {
            #(#builder_fields,)*
        }

        impl #impl_generics #struct_name #ty_generics #where_clause {
            pub fn builder() -> #builder_name #ty_generics {
                #builder_name {
                    #(#builder_defaults,)*
                }
            }
        }

        impl #impl_generics #builder_name #ty_generics #where_clause {
            #(#setters)*

            pub fn build(self) -> Result<#struct_name #ty_generics, String> {
                Ok(#struct_name {
                    #(#build_fields,)*
                })
            }
        }
    };

    Ok(expanded)
}
```

จุดที่ควรอธิบายเพิ่มเติมจากโค้ดข้างบน:

**1) `format_ident!("{}Builder", struct_name)`** — สร้างชื่อ struct builder ใหม่จากชื่อ struct เดิม
บวกคำว่า `Builder` ต่อท้าย (`Config` → `ConfigBuilder`) นี่คือ pattern การตั้งชื่อมาตรฐานที่ crate
`derive_builder` จริงบน crates.io ก็ใช้แนวทางเดียวกันนี้เป๊ะ

**2) ทำไม `Option<T>` คือกลไกหลักของ builder pattern ที่ generate** — ทุก field ใน `XxxBuilder`
ห่อด้วย `Option<T>` เพราะ ณ ตอนสร้าง `Builder` เราไม่รู้ว่าผู้ใช้จะ set field ไหนก่อนหลัง หรือจะลืม
set field ไหนไปเลย `Option<T>` (Part 11) คือเครื่องมือธรรมชาติที่สุดสำหรับแทน "ค่านี้อาจจะยังไม่มี"
— ตอน `build()` เราค่อยแปลง `Option<T>` ทุกตัวกลับเป็น `T` (ด้วย `.ok_or_else(...)?`) พร้อมเช็คครบ
ทุก field ในจุดเดียว ถ้า field ไหนยังเป็น `None` การ `?` จะ early-return `Err` ออกจาก `build()` ทันที
(ทวนหลักการ `?` operator จาก Part 12)

**3) การไม่ต้องเติม bound เพิ่มเอง ต่างจาก `Describe`** — สังเกตว่า `Builder` **ไม่ได้เรียกฟังก์ชัน
คล้าย `add_debug_bound`** เลย เพราะ `Option<T>` ใช้ได้กับ `T` **ทุกชนิดโดยไม่มีเงื่อนไขใด ๆ** (ไม่ต้อง
`T: Debug`, ไม่ต้อง `T: Clone`, อะไรเลย) ในขณะที่ `Describe` ต้องเติม `T: Debug` เพราะโค้ดที่ generate
เรียกใช้ `{:?}` ตรง ๆ กับค่าที่ type เป็น `T` — บทเรียนสำคัญจากการเทียบสองตัวนี้: **การที่ derive
macro ต้องเติม trait bound เพิ่มเองหรือไม่ ขึ้นอยู่กับว่าโค้ดที่ generate ออกมาเรียกใช้ trait method
อะไรกับค่าที่ type เป็น generic parameter บ้าง** ถ้าไม่เรียกอะไรเลย (แค่เก็บ/ย้ายค่าไปมาเฉย ๆ แบบที่
`Builder` ทำ) ก็ไม่ต้องเติม bound ใด ๆ แต่ bound เดิมที่ struct ต้นทางประกาศไว้แล้ว (ถ้ามี) ยังต้อง
ถูกส่งต่อไปยัง generated code ผ่าน `split_for_impl()` เหมือนเดิมเสมอ

**4) `pub struct`/`pub fn` ทุกจุดในโค้ดที่ generate** — เพื่อความง่าย macro เวอร์ชันนี้ตั้งให้ทุกอย่าง
เป็น `pub` เสมอ (ไม่ได้ดู visibility จริงของ field ต้นทาง) ซึ่งเป็นการลดความซับซ้อนที่ยอมรับได้สำหรับ
บทเรียนนี้ — ใน production จริง `derive_builder` (crate จริงบน crates.io) จะดู `field.vis` และ
`input.vis` เพื่อ generate visibility ให้ตรงกับต้นฉบับ ซึ่งเป็นแบบฝึกหัดที่ดีสำหรับผู้ที่อยากขยาย
macro นี้ต่อ (ดูแบบฝึกหัดข้อ 4)

**ทดสอบ Builder จริง** ด้วย consumer crate:

```rust
// builder_consumer/src/main.rs
use builder_derive::Builder;

#[derive(Builder, Debug)]
pub struct Config {
    pub host: String,
    pub port: u16,
    pub timeout_ms: u64,
    pub tags: Vec<String>,
}

fn main() {
    // กรณีปกติ: set ครบทุก field แล้ว build สำเร็จ
    let config = Config::builder()
        .host("localhost".to_string())
        .port(8080)
        .timeout_ms(3000)
        .tags(vec!["prod".to_string(), "eu-west".to_string()])
        .build()
        .expect("ควร build สำเร็จเมื่อ set ครบทุก field");
    println!("{config:?}");

    // กรณี build ไม่ครบ field: ต้องได้ Err พร้อมข้อความชัดเจนว่าขาด field ไหน
    let incomplete = Config::builder().host("only-host-set".to_string()).build();
    println!("{incomplete:?}");
}
```

ผลลัพธ์จริงจาก `cargo run` (คัดลอกจาก terminal ตรง ๆ):

```
Config { host: "localhost", port: 8080, timeout_ms: 3000, tags: ["prod", "eu-west"] }
Err("missing field `port`")
```

บรรทัดแรกยืนยันว่า `.host(...).port(...).timeout_ms(...).tags(...).build()` ทำงานถูกต้องครบทุก field
ที่มี type ต่างกันสี่แบบ (`String`, `u16`, `u64`, `Vec<String>`) — ครอบคลุมข้อกำหนด "multiple fields
of different types" อย่างเป็นรูปธรรม บรรทัดที่สองยืนยันว่าเมื่อ set ไม่ครบ (ตั้งใจไม่ set `port`,
`timeout_ms`, `tags`) `build()` คืน `Err` พร้อมชื่อ field ที่ขาดตัว**แรก**ที่เจอ (`port`) ตรงตาม
ลำดับการเช็คใน `build_fields` — ไม่ crash ไม่ panic ทำงานตามที่ออกแบบไว้ทุกจุด

### 45.14 ทดสอบ Builder กับ struct ที่มี generics ของตัวเอง

เพื่อพิสูจน์ว่าการดึง `generics`/`split_for_impl()` มาใช้ใน `Builder` (หัวข้อ 45.13 ข้อ 3) ทำงานถูกต้อง
จริง มาทดสอบกับ struct ที่มี generic parameter ของตัวเอง:

```rust
#[derive(Builder, Debug)]
pub struct Pair<T: Clone + std::fmt::Debug> {
    pub left: T,
    pub right: T,
}

fn main() {
    // ... (โค้ดจากหัวข้อ 45.13) ...

    // Builder ใช้กับ struct ที่มี generic parameter ของตัวเองได้ด้วย
    let pair = Pair::builder().left(1).right(2).build().unwrap();
    println!("{pair:?}");
}
```

ผลลัพธ์จริง:

```
Pair { left: 1, right: 2 }
```

macro generate `PairBuilder<T: Clone + std::fmt::Debug>`, `impl<T: Clone + std::fmt::Debug> Pair<T>`,
และ `impl<T: Clone + std::fmt::Debug> PairBuilder<T>` โดยอัตโนมัติ — bound `Clone + Debug` ที่ผู้ใช้
ประกาศไว้บน `Pair<T>` เอง (เพื่อรองรับ `#[derive(Debug)]` ตัวที่สองที่แปะไว้คู่กัน) ถูก**ส่งต่อ**ไปยัง
ทุก `impl` block ที่ generate โดยที่ macro ของเรา**ไม่ต้องรู้อะไรเกี่ยวกับ bound เหล่านั้นเลย** —
นี่คือข้อดีของการใช้ `split_for_impl()` เทียบกับการพยายามเขียน `impl<T> ...` ตายตัวด้วยมือ (ซึ่งจะทำให้
โค้ดที่ generate ผิดทันทีถ้า struct ต้นทางมี bound อะไรก็ตามที่ macro ไม่รู้จักมาก่อน)

### 45.15 เมื่อไหร่ควรเขียน proc macro เอง เมื่อไหร่ควรใช้ของสำเร็จจากคนอื่น

ตลอดบทนี้เราเห็นแล้วว่าการเขียน derive macro ที่ "ทำถูกต้องจริง ๆ" ต้องดูแลอะไรมากกว่าที่คิดตอนแรกมาก:
data shape ทั้งหมด (struct 3 แบบ + enum ที่มี variant ผสมกันได้ไม่จำกัด), generics และ trait bound,
helper attribute, error handling ที่ดี, และการทดสอบที่ต้องพึ่งเครื่องมือเฉพาะทางอย่าง `trybuild` —
เมื่อเทียบเวลาที่เราใช้ไปกับเวลาที่ `#[derive(Serialize)]` ตัวเดียวจาก `serde` ให้เราได้ฟรี (พร้อม
รองรับ format สิบกว่าแบบ, edge case ที่สั่งสมมาหลายปี, และการดูแลรักษาต่อเนื่องจากทีมที่เชี่ยวชาญ)
คำตอบของคำถาม "ควรเขียน proc macro เองไหม" เกือบทุกครั้งคือ **"ไม่ ควรใช้ของสำเร็จรูปก่อน"**

หลักการตัดสินใจที่ควรใช้จริง (ต่อยอดจาก Part 36 ที่บอกว่า "macro ควรเป็นตัวเลือกสุดท้าย" ขึ้นมาอีก
ระดับหนึ่ง):

**ใช้ derive macro สำเร็จรูปจาก crates.io เมื่อ:**

- ปัญหาที่คุณเจอเป็นปัญหา**มาตรฐาน**ที่คนอื่นในโลกเจอมาก่อนแล้วแน่นอน (serialize/deserialize เป็น
  JSON/YAML → `serde`; สร้าง error type ที่มี `Display`/`From` ให้ครบ → `thiserror`, Part 31; parse
  command-line argument → `clap`, Part 59) crate เหล่านี้ผ่านการทดสอบจากผู้ใช้จริงหลายล้านโปรเจกต์
  ครอบคลุม edge case ที่คุณคาดไม่ถึงว่ามีอยู่
- คุณต้องการ**ความเข้ากันได้กับ ecosystem** — ถ้า struct ของคุณต้อง serialize ด้วย `serde` เพื่อส่ง
  ผ่าน `axum` (Part 62-66) หรือเก็บลง database ด้วย `sqlx` (Part 70-71) การ derive macro ของตัวเองที่
  "คล้าย serde" จะใช้ร่วมกับ library อื่นในโลกไม่ได้เลย ต้องใช้ `serde` ตัวจริงเท่านั้น
- ทีมของคุณมีขนาดใหญ่ หรือจะมีคนอื่นมา maintain โค้ดต่อในอนาคต — macro สำเร็จรูปมีเอกสาร มีตัวอย่าง
  มีคนตอบคำถามบน Stack Overflow เป็นพัน ๆ กระทู้ ในขณะที่ macro ที่คุณเขียนเอง**มีคุณคนเดียว**ที่เข้าใจ
  มันอย่างถ่องแท้

**เขียน proc macro ของตัวเอง (เมื่อคุ้มค่าจริง ๆ) เมื่อ:**

- มี boilerplate ที่**ซ้ำกันจริงข้าม type จำนวนมาก**ในโดเมนที่**เฉพาะเจาะจงกับธุรกิจของคุณเอง** ไม่มี
  crate สำเร็จรูปที่แก้ปัญหานี้ได้ตรงจุด เช่น ทุก error type ในระบบของบริษัทต้อง generate HTTP status
  code ที่สัมพันธ์กันแบบเฉพาะของคุณเอง (ไม่ใช่แค่ `Display`/`From` ธรรมดาที่ `thiserror` ให้ได้แล้ว)
  หรือทุก request/response type ต้อง generate การ validate field แบบเฉพาะทางของ domain ธุรกิจ
- คุณลองแก้ปัญหาด้วย generics/trait (Part 18-22) หรือ declarative macro (Part 36) แล้วพบว่าทำไม่ได้
  จริง ๆ เพราะปัญหาต้องการ**รู้โครงสร้างของ type ที่จะ generate โค้ดให้** (ชื่อ field, type ของ field,
  จำนวน field) ซึ่งเป็นสิ่งที่ trait ทำไม่ได้ (trait ทำงานกับ "อินเทอร์เฟซ" ไม่ใช่ "โครงสร้างภายใน")
  และ `macro_rules!` ทำได้ยากมากถ้าไม่รู้ syntax ล่วงหน้าแน่นอนแบบ struct ทั่วไปที่ผู้ใช้เขียนขึ้นมาเอง
- คุณมีเวลาและกำลังคนพอที่จะดูแล proc-macro crate นี้ต่อไปในระยะยาว (เขียน test ด้วย `trybuild`,
  อัปเดตตาม `syn`/`quote` version ใหม่, เขียน documentation ให้คนในทีมเข้าใจ) — ถ้าไม่มีเวลาสำหรับสิ่ง
  เหล่านี้ การมี proc macro ที่ดูแลไม่ดีจะกลายเป็นหนี้ทางเทคนิค (technical debt) ที่แพงกว่าการเขียน
  boilerplate ซ้ำ ๆ ด้วยมือเสียอีก

สรุปเป็นคำถามเดียวที่ควรถามตัวเองก่อนเริ่มเขียน proc macro ทุกครั้ง (ในทำนองเดียวกับคำถามของ Part 36):
**"ปัญหานี้มี crate สำเร็จรูปที่แก้ได้ตรงจุดอยู่แล้วหรือไม่ และถ้าไม่มี ปัญหานี้ใหญ่พอ/ซ้ำพอที่จะคุ้ม
กับต้นทุนการดูแล proc-macro crate ไปตลอดชีวิตของโปรเจกต์หรือไม่"** — `Describe` และ `Builder` ที่เรา
สร้างในบทนี้เป็นตัวอย่างการศึกษาที่ดีเพื่อเข้าใจกลไกทั้งหมด แต่ในโปรเจกต์จริง ถ้าคุณต้องการ builder
pattern แบบทั่วไป ควรพิจารณาใช้ crate `derive_builder` (บน crates.io) ที่ทำสิ่งเดียวกันนี้แบบสมบูรณ์
กว่ามาก (รองรับ default value, optional field ที่ไม่ต้อง set, validation แบบกำหนดเอง ฯลฯ) มากกว่าเขียน
เองจากศูนย์แบบในบทนี้

### 45.15.1 ทุกเทคนิคในบทนี้ อยู่ที่ไหนใน derive macro ที่คุณใช้อยู่แล้ว

เพื่อให้เห็นภาพว่าเทคนิคทั้งหมดที่เราเขียนเองในบทนี้ไม่ใช่ของเล่นทางวิชาการ แต่เป็นสิ่งที่ derive macro
ที่คุณใช้งานจริง (หรือจะได้ใช้ในบทถัดไปของหลักสูตร) ทำอยู่จริงทุกวัน มาดูตารางเทียบกัน:

| เทคนิคที่เราสร้างเอง (หัวข้อ) | ชื่อกลไกทั่วไป | ตัวอย่างจริงที่ใช้เทคนิคเดียวกัน |
|---|---|---|
| แยก `Fields::Named`/`Unnamed`/`Unit` (45.2-45.4) | Data-shape dispatch | `#[derive(Debug)]`, `#[derive(Serialize)]` ต้องรองรับทุก shape เพื่อใช้กับ struct/enum อะไรก็ได้ |
| เติม trait bound ให้ generic (45.6-45.7) | Bound propagation | `#[derive(Clone)]` ของ std เติม `T: Clone` ให้ทุก type parameter ของ struct ที่ derive อยู่โดยอัตโนมัติแบบเดียวกัน |
| Helper attribute (`#[describe(skip)]`, 45.8-45.9) | Field-level customization | `#[serde(rename/skip/default)]` (Part 57-58), `#[error("...")]` ของ `thiserror` (Part 31), `#[arg(short, long)]` ของ `clap` (Part 59) |
| `syn::Error` + span ที่ถูกต้อง (45.10-45.11) | Graceful diagnostics | ทุก derive macro คุณภาพดีบน crates.io ใช้แนวทางนี้ — ลองสังเกต error message ตอนคุณใช้ `#[derive(Serialize)]`/`#[derive(Parser)]` ผิดวิธีดู จะเห็นข้อความที่ชี้ตำแหน่งแม่นยำแบบเดียวกัน |
| `trybuild` (45.12) | Compile-time UI testing | `serde`, `thiserror`, `async-trait`, `tokio` ใช้จริงใน CI ของตัวเองทั้งหมด |
| Builder pattern generation (45.13-45.14) | Code generation for a design pattern | `derive_builder` (บน crates.io), และ `clap` ก็ generate โครงสร้างคล้าย builder ภายในให้ `#[derive(Parser)]` เช่นกัน |

ครั้งต่อไปที่คุณเปิด error message จาก `#[derive(Serialize)]` ที่ไม่ผ่าน หรืออ่าน documentation ของ
`#[serde(...)]`/`#[error(...)]`/`#[arg(...)]` คุณจะรู้ทันทีว่าเบื้องหลังมันไม่มีอะไรที่ "เวทมนตร์" เกิน
กว่าที่คุณเพิ่งเขียนเองในบทนี้เลย — ต่างกันแค่ระดับความครบถ้วนของ edge case ที่ผ่านการทดสอบมาหลายปีจาก
ผู้ใช้จริงหลายล้านคนเท่านั้น

## กับดักที่พบบ่อย (Common Pitfalls)

**1. ลืมเติม trait bound ให้ generated `impl` — compile fail เฉพาะตอนใช้กับ generic type**

นี่คือกับดักที่อันตรายที่สุดในบทนี้ เพราะ**นิ่งเงียบสนิทตอนเทสกับ type ธรรมดา** (ดูหัวข้อ 45.7 แบบ
เต็ม) ถ้า macro generate `impl<T> Trait for Wrapper<T> { ... }` โดยใช้ `{:?}`/`{}`/method ของ trait
อื่นกับค่าที่ type เป็น `T` ตรง ๆ ข้างใน แต่ไม่เติม bound ที่จำเป็นให้ `impl<T>` ตรงนั้น จะได้:

```
error[E0277]: `T` doesn't implement `Debug`
  --> src/main.rs:34:10
   |
34 | #[derive(Describe)]
   |          ^^^^^^^^ `T` cannot be formatted using `{:?}` because it doesn't implement `Debug`
```

**วิธีป้องกัน:** ทุกครั้งที่ derive macro generate `impl` ที่ใช้ trait method ใด ๆ กับค่าที่ type เป็น
generic parameter ของ struct ต้นทาง ให้ถามตัวเองว่า "trait method นี้ต้องการ bound อะไรบ้าง" แล้วเติม
bound นั้นด้วย `generics.type_params_mut()` + `.bounds.push(syn::parse_quote!(...))` เสมอ ก่อน
`split_for_impl()` — และ**ทดสอบ macro กับ type ที่มี generic parameter อย่างน้อยหนึ่งตัวเสมอ** ไม่ใช่
เทสแค่กับ struct ธรรมดา (นี่คือเหตุผลที่ `Wrapper<T>` ควรอยู่ใน test suite ของทุก derive macro ที่
generate โค้ดแบบนี้)

**2. ใช้ `panic!`/`.unwrap()` แทน `syn::Error` — ได้ error message ที่งงและ debug ยาก**

ดูหัวข้อ 45.11 แบบเต็ม สรุปสั้น ๆ: `unreachable!()`/`panic!()` ในโค้ด macro ทำให้ compiler รายงาน
`error: proc-macro derive panicked` พร้อม span ที่ชี้กว้าง ๆ (แค่ตำแหน่ง `#[derive(...)]`) และไม่มี
คำอธิบายเหตุผลเลย:

```
error: proc-macro derive panicked
 --> src/main.rs:7:10
  |
7 | #[derive(Describe)]
  |          ^^^^^^^^
  |
  = help: message: internal error: entered unreachable code: ยังไม่รองรับ union
```

**วิธีป้องกัน:** ให้ทุกฟังก์ชันภายใน macro คืน `syn::Result<T>` และใช้ `syn::Error::new_spanned(node,
msg)` เสมอเมื่อพบ input ที่ไม่ถูกต้อง จากนั้นแปลงเป็น `TokenStream` ด้วย `.to_compile_error().into()`
ที่จุดเดียวคือ entry point ของ `#[proc_macro_derive]` (ดูหัวข้อ 45.10)

**3. ไม่จัดการ `Fields::Unnamed`/`Fields::Unit` — ใช้ได้แค่กับ named-field struct เท่านั้นโดยไม่รู้ตัว**

ถ้า `match &data.fields` เขียนแค่ `Fields::Named(f) => { ... }, _ => panic!("...")` (จับ
`Fields::Unnamed` และ `Fields::Unit` รวมเป็น `_` ทั้งคู่) macro จะดู "ใช้งานได้" ตอนเทสกับ struct ที่
เขียนขึ้นมาเองเท่านั้น (ซึ่งมักจะเขียนเป็น named-field struct เพราะอ่านง่ายกว่า) แต่พังทันทีที่มีคนเอา
ไปใช้กับ tuple struct หรือ unit struct — ในทางกลับกัน ถ้าอ้างอิง field ด้วยชื่อตรง ๆ (`field.ident.
unwrap()`) ใน branch ที่ควรเป็น `Fields::Unnamed` (ซึ่ง field ไม่มีชื่อ `.ident` จะเป็น `None`) จะได้:

```
thread 'main' panicked at 'called `Option::unwrap()` on a `None` value'
```

panic ที่**ไม่มี span ไปยัง source code ของผู้ใช้เลย** เพราะเป็น panic ธรรมดาของ Rust ไม่ใช่ compile
error — ทำให้ debug ยากกว่ากรณีอื่น ๆ ในบทนี้ทั้งหมด (ผู้ใช้จะเห็น panic message ที่ไม่มีบรรทัด/ไฟล์
ของตัวเองเกี่ยวข้องเลย)

**วิธีป้องกัน:** เขียน `match &data.fields` ให้ครบทั้งสาม variant เสมอ (`Fields::Named`,
`Fields::Unnamed`, `Fields::Unit`) แม้บาง variant จะ generate โค้ดเรียบง่ายมาก (เช่น `Fields::Unit`
มักจะแค่คืนชื่อ type ตรง ๆ ไม่มี field ให้ประมวลผลเลย) — การเขียนให้ครบ `match` ทุก arm (ไม่ใช้ `_`
รวบ) ทำให้ compiler ของ**ตัว macro เอง**เตือนทันทีถ้าเพิ่ม variant ใหม่ในอนาคตแล้วลืมจัดการ (exhaustive
match ทวนจาก Part 10)

**4. ลืม escape เครื่องหมายปีกกาสองชั้นตอนสร้าง format string ที่จะไปสร้าง format string อีกที**

นี่คือกับดักที่ดูเผิน ๆ เหมือนเรื่องเล็ก แต่พบได้บ่อยมากเวลาสร้าง derive macro ที่ generate โค้ดเรียก
`format!`/`println!` ปัญหาคือมี "ชั้นของการ escape" สองชั้นซ้อนกัน: ชั้นแรกคือ `format!` ที่**เราเรียก
เองตอนเขียนโค้ด macro** (เพื่อประกอบ string ของ format string ที่จะ generate) ชั้นที่สองคือ `format!`
ที่**macro generate ออกมาให้ผู้ใช้** (ซึ่งจะรันตอน runtime ของโปรแกรมผู้ใช้) ถ้าเขียนแบบนี้ (ดูผิดที่
ชวนพลาด):

```rust
// ผิด! ใช้ {{ }} single-escape ระดับเดียว ทั้งที่ต้อง double-escape
let fmt_string = format!("{type_name} {{ {} }}", format_parts.join(", "));
```

`format!` ในบรรทัดนี้ (ชั้นแรก) จะตีความ `{{` เป็นเครื่องหมาย `{` literal หนึ่งตัว (escape แค่ชั้นเดียว)
ทำให้ `fmt_string` ที่ได้กลายเป็น string ที่มี `{`/`}` เดี่ยว ๆ (ไม่ใช่คู่ `{{`/`}}`) เมื่อ string นี้
ถูกเอาไปใช้เป็น **format string ชั้นที่สอง** ใน generated code (`format!(#fmt_string, ...)`) `{` เดี่ยว
ที่หลงเหลืออยู่จะถูกตีความเป็นจุดเริ่มต้นของ placeholder ทันที ทำให้ได้ compile error แบบนี้:

```
error: invalid format string: expected `}`, found `x`
  --> src/main.rs:11:10
   |
11 | #[derive(Describe)]
   |          ^^^^^^^^ expected `}` in format string
   |
   = note: if you intended to print `{`, you can escape it using `{{`
```

**วิธีป้องกัน:** เมื่อต้อง generate string literal ที่**ตัวมันเองต้องมี `{{`/`}}` แบบ escape อยู่ข้างใน**
(เพื่อให้ format! ชั้นที่สองตีความเป็น literal `{`/`}`) ต้อง escape ให้**ครบสองชั้น**ตอนเขียนโค้ด macro
เอง — คือใช้ `{{{{`/`}}}}` (สี่ตัว) แทน `{{`/`}}` (สองตัว):

```rust
// ถูก! {{{{ }}}} (สี่ชั้น) เพราะ format! ชั้นแรกจะลดเหลือ {{ }} (สองชั้น) ซึ่งคือสิ่งที่
// format! ชั้นที่สอง (ใน generated code) ต้องเห็นเพื่อตีความเป็น { } literal ตัวเดียว
let fmt_string = format!("{type_name} {{{{ {} }}}}", format_parts.join(", "));
```

วิธีเช็คง่าย ๆ ที่ช่วยได้เสมอ: เขียนค่าที่**ต้องการให้ `fmt_string` มีอยู่จริง**ออกมาบนกระดาษก่อน (เช่น
`"Point {{ x: {:?}, y: {:?} }}"` — สังเกตว่ามันมี `{{`/`}}` คู่ ล้อมรอบ `x: {:?}, y: {:?}` ซึ่งใช้ตัว
เดี่ยวเพราะเป็น placeholder จริงของชั้นที่สอง) แล้วค่อยย้อนกลับไปคิดว่าต้อง escape ยังไงในชั้นแรกเพื่อ
ให้ได้ผลลัพธ์นั้นเป๊ะ ๆ — บั๊กนี้เกิดขึ้นจริงระหว่างพัฒนาตัวอย่างในบทนี้เอง (ก่อนแก้ไข) เพื่อให้เห็นภาพ
ว่าแม้แต่ตัวอย่างในหลักสูตรก็พลาดจุดนี้ได้ง่ายถ้าไม่ระมัดระวัง

**5. ลืมประกาศ `attributes(...)` — helper attribute ที่ตั้งใจสร้างใช้งานไม่ได้เลย**

ถ้าลืมเขียน `attributes(describe)` ใน `#[proc_macro_derive(Describe, attributes(describe))]` (เผลอ
เขียนแค่ `#[proc_macro_derive(Describe)]`) แล้วผู้ใช้ลองใช้ `#[describe(skip)]` บน field จะได้:

```
error: cannot find attribute `describe` in this scope
  --> src/main.rs:10:7
   |
10 |     #[describe(skip)]
   |       ^^^^^^^^
```

สังเกตว่า error นี้เกิด**ก่อน**ที่ macro ของเราจะได้ทำงานด้วยซ้ำ (compiler ปฏิเสธ syntax ตั้งแต่ขั้น
parse attribute เพราะไม่รู้จักชื่อ `describe` เป็น attribute ที่ derive macro ตัวไหนประกาศไว้)
**วิธีป้องกัน:** ทุกครั้งที่ออกแบบ helper attribute ใหม่ ต้องเพิ่มชื่อมันเข้าไปใน `attributes(...)`
ของ `#[proc_macro_derive(...)]` เสมอ — ถ้ามี helper attribute หลายชื่อ ใส่คั่นด้วย comma ได้ เช่น
`attributes(describe, describe_extra)`

**6. Builder derive กับ field ที่ type ซับซ้อน (เช่น `Option<T>` เดิมอยู่แล้ว) — เกิด `Option<Option<T>>`**

ถ้า struct ต้นทางมี field ที่เป็น `Option<T>` อยู่แล้ว (เช่น `timeout: Option<u64>` หมายถึง "ไม่บังคับ
ต้อง set") macro `Builder` ในบทนี้จะ generate `timeout: Option<Option<u64>>` ใน `XxxBuilder` (ห่อ
`Option` ซ้ำสองชั้นโดยไม่ตั้งใจ) เพราะ macro เขียนแบบ "ห่อทุก field ด้วย `Option<T>` เสมอ โดยไม่ดูว่า
`T` เดิมเป็น `Option` อยู่แล้วหรือไม่" ผลคือ `build()` จะบังคับให้ต้อง `.timeout(Some(30))` เสมอ (ต้อง
เรียก setter ก่อน `build()` ถึงจะไม่ error) ทั้งที่เจตนาเดิมของ field คือ "ไม่ set ก็ได้ ค่าเริ่มต้น
เป็น `None`" — นี่ไม่ใช่ error แต่เป็น**การออกแบบที่ไม่ตรงกับความคาดหวังของผู้ใช้** ซึ่งอันตรายกว่า
compile error เสียอีก เพราะ compile ผ่านหมด ดูเหมือนใช้งานได้ปกติ

**วิธีป้องกัน (แนวทาง ไม่ implement เต็มในบทนี้ ทิ้งไว้เป็นแบบฝึกหัดข้อ 4):** เช็ค `field.ty` ว่าเป็น
`syn::Type::Path` ที่ path segment สุดท้ายชื่อ `Option` หรือไม่ ถ้าใช่ ให้ generate setter/build logic
คนละแบบ (setter รับ `T` ตรง ๆ ไม่ต้องห่อด้วย `Some` ซ้ำ, และ `build()` ใช้ `.unwrap_or(None)` แทน
`.ok_or_else(...)?` เพราะ field แบบนี้ไม่บังคับต้อง set) — crate `derive_builder` จริงบน crates.io
จัดการเรื่องนี้ด้วย helper attribute แบบ `#[builder(default)]` ซึ่งใช้กลไก helper attribute เดียวกัน
กับที่เราสร้างในหัวข้อ 45.8-45.9 เป๊ะ

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** ขยาย `describe_struct_body` (หัวข้อ 45.3) ให้รองรับ `#[describe(skip)]` บน field ของ
   tuple struct ด้วย (ปัจจุบันตัวอย่างในบทนี้ทำให้ `Fields::Named` รองรับ skip ไว้แล้ว แต่
   `Fields::Unnamed` ยังไม่รองรับ) ทดสอบด้วย `struct Secret(String, #[describe(skip)] String);`
   แล้วยืนยันว่า `describe()` แสดงแค่ field แรก (hint: ต้องดู `field.attrs` ของแต่ละ element ใน
   `fields.unnamed` เหมือนที่ทำกับ `fields.named` แล้ว `continue` ข้าม index นั้นไปตอนสร้าง
   `format_args` แต่ต้องคิดด้วยว่า index ที่เหลือ (field ที่ไม่ได้ skip) ยังต้องใช้ `self.0`, `self.1`
   ตาม position เดิมใน struct เสมอ ไม่ใช่ตำแหน่งใหม่หลัง skip)

2. **[กลาง]** เพิ่ม helper attribute ใหม่ `#[describe(rename = "...")]` ให้ `Describe` (คล้าย
   `#[serde(rename = "...")]`) ที่เปลี่ยนชื่อ field ที่แสดงใน output ของ `describe()` โดยไม่ต้องเปลี่ยน
   ชื่อ field จริงในโค้ด Rust เช่น `struct User { #[describe(rename = "full_name")] name: String }`
   ควรให้ `describe()` print เป็น `User { full_name: "..." }` ไม่ใช่ `User { name: "..." }` (hint: ต้อง
   ใช้ `meta.value()?.parse::<syn::LitStr>()?` ภายใน closure ของ `parse_nested_meta` เพื่อดึงค่า string
   หลัง `=` — ลองอ่าน error message ที่ได้ถ้า parse ผิด type ดูว่า `syn` ช่วย debug ให้แค่ไหน)

3. **[ยาก]** เขียน `trybuild` test suite แบบเต็มให้ `describe_derive` (ตามโครงสร้างในหัวข้อ 45.12)
   ที่มีอย่างน้อย 4 test case: (ก) named-field struct compile ผ่าน, (ข) enum ที่มี variant ผสมกัน
   compile ผ่าน, (ค) union compile ไม่ผ่านพร้อม error message ที่ถูกต้อง (ง) การใช้
   `#[describe(bad_key)]` ที่ไม่รู้จัก compile ไม่ผ่านพร้อม error message จาก `meta.error(...)` ที่เรา
   เขียนไว้ — ใช้ `TRYBUILD=overwrite cargo test` เพื่อสร้างไฟล์ `.stderr` ให้อัตโนมัติในรอบแรก แล้ว
   ตรวจสอบเนื้อหาก่อน commit

4. **[ยาก/ประยุกต์ใช้งานจริง]** ขยาย `#[derive(Builder)]` (หัวข้อ 45.13) ให้จัดการ field ที่ type เป็น
   `Option<T>` อยู่แล้วอย่างถูกต้อง (แก้กับดักข้อ 6 ในหัวข้อกับดัก) กล่าวคือ: ถ้า field ต้นทางเป็น
   `Option<T>` ให้ (ก) setter รับ `T` ตรง ๆ (ไม่ต้องห่อด้วย `Some` เอง ผู้ใช้เขียน `.timeout(30)` ไม่ใช่
   `.timeout(Some(30))`) และ (ข) `build()` ไม่บังคับว่าต้อง set field นี้ก่อน (ถ้าไม่ set ให้เป็น `None`
   ไปเลยไม่ error) ในขณะที่ field ที่**ไม่ใช่** `Option<T>` ยังคงบังคับต้อง set เหมือนเดิมทุกอย่าง
   เขียนโปรแกรมทดสอบที่มี struct ผสมทั้ง field บังคับและ field แบบ `Option<T>` แล้วยืนยันว่า
   `build()` สำเร็จได้แม้ไม่ set field แบบ `Option<T>` เลย (hint: Part 44 หัวข้อ 44.5 สอนเทคนิคการ
   "มองเข้าไปข้างใน" `syn::Type` ไว้แล้วผ่านฟังก์ชัน `is_option_of` ที่เช็ค `Type::Path` → segment
   สุดท้ายชื่อ `"Option"` → ดึง `T` ข้างในออกมาผ่าน `PathArguments::AngleBracketed` + `GenericArgument::
   Type` — เอาฟังก์ชันนั้นมาปรับใช้ตรงนี้ได้เกือบทั้งดุ้น แค่เปลี่ยนจากการ print ผลลัพธ์ ไปเป็นการใช้
   `Type` ที่ดึงได้เป็น parameter type ของ setter method แทน)

## สรุป

บทนี้ปิดวงคำถามที่ Part 44 เปิดไว้ตอนท้าย: "ทำไงให้ derive macro ใช้ได้กับกรณีจริง ไม่ใช่แค่กรณีง่าย
ที่สุด" เราไล่แก้ข้อจำกัดของ Part 44 ทีละข้อ — เริ่มจากการอ่าน `syn::Data`/`syn::Fields` ให้ครบทุก
variant (struct 3 แบบ + enum ที่มี variant ผสมกันได้ไม่จำกัด) แล้วขยาย `#[derive(Describe)]` ให้
generate โค้ดถูกต้องสำหรับทุกกรณี จากนั้นเรียนรู้ปัญหาที่อันตรายที่สุดของ derive macro (การลืมเติม
trait bound ให้ generated `impl` เมื่อ type ต้นทางมี generics ของตัวเอง — บั๊กที่นิ่งเงียบตอนเทสกับ
type ธรรมดาแต่ระเบิดทันทีที่มีคนใช้กับ generic type) พร้อมดู error message จริงและวิธีแก้ด้วย
`syn::Generics::split_for_impl()`

จากนั้นเราเปิดฝากล่องของ `#[serde(rename = "...")]` และ attribute แบบเดียวกันที่ `thiserror` ใช้
(Part 31) พบว่ากลไกทั้งหมดเริ่มจาก `attributes(...)` ใน `#[proc_macro_derive]` ที่เปิดสิทธิ์ให้ syntax
attribute แบบใหม่ผ่านการ parse ได้ ก่อนที่ macro ของเราจะไป parse เนื้อหาข้างในด้วย
`parse_nested_meta` เอง — เราสร้าง `#[describe(skip)]` ของตัวเองจนใช้งานได้จริง แล้วเรียนรู้ error
handling ระดับ production: เปรียบเทียบ `panic!`/`unreachable!()` (ที่ทำให้ compiler รายงาน "proc-macro
derive panicked" อันน่าสับสน) กับ `syn::Error::new_spanned` + `.to_compile_error()` (ที่ให้ compile
error สุภาพ ชี้ตำแหน่งถูก พร้อมคำอธิบายที่ผู้ใช้เข้าใจได้ทันที) และปิดท้ายด้วยความรู้ระดับความเข้าใจ
เรื่องการทดสอบ proc macro ด้วย `trybuild` — เครื่องมือที่ `serde`, `thiserror`, `async-trait` ใช้จริง
ในการยืนยันว่า compile error message ของตัวเองยังถูกต้องอยู่เสมอ

ไฮไลต์ใหญ่ของบทคือการสร้าง `#[derive(Builder)]` แบบเต็มรูปแบบ ตั้งแต่ออกแบบโค้ดเป้าหมายด้วยมือก่อน
ไปจนถึง generate มันด้วย `syn`/`quote` จริง — เรายืนยันว่าทำงานถูกต้องทั้งกรณีปกติ (`build()` สำเร็จ
เมื่อ set ครบทุก field ของ type ต่างกันสี่แบบ) กรณี error (`build()` คืน `Err` พร้อมชื่อ field ที่ขาด
เมื่อ set ไม่ครบ) และกรณี struct ที่มี generic parameter ของตัวเอง (`Pair<T>`) — พร้อมเห็นความต่างที่
สำคัญระหว่าง `Describe` (ต้องเติม `T: Debug` เอง เพราะเรียก `{:?}` กับค่า type `T` ตรง ๆ) กับ
`Builder` (ไม่ต้องเติม bound อะไรเลย เพราะ `Option<T>` ใช้ได้กับทุก `T`) ซึ่งเป็นบทเรียนที่สรุปหลักการ
ทั้งหมดของหัวข้อ generics ในบทนี้ได้กระชับที่สุด

ปิดท้ายด้วยคำถามที่สำคัญที่สุดของทั้งบท ซึ่งยกระดับหลักการ "macro ควรเป็นตัวเลือกสุดท้าย" ของ Part 36
ขึ้นไปอีกชั้น: **การเขียน proc macro เองมีต้นทุนสูงกว่าที่คิดมาก** (ต้องดูแล data shape ทุกแบบ,
generics, helper attribute, error handling, และการทดสอบเฉพาะทาง) เมื่อเทียบกับการใช้ `serde`,
`thiserror`, `clap` ที่แก้ปัญหามาตรฐานเดียวกันได้ดีกว่าและได้รับการดูแลจากทีมที่เชี่ยวชาญมาแล้ว ทางเลือก
ที่ถูกต้องในโปรเจกต์จริงเกือบทุกครั้งคือใช้ของสำเร็จรูปก่อน แล้วเขียน proc macro เองเฉพาะตอนที่มี
boilerplate ซ้ำจริงในโดเมนของธุรกิจตัวเองเท่านั้น — และนี่คือการปิดวงเนื้อหาเรื่อง macro ทั้งหมดของ
หลักสูตร ตั้งแต่ `macro_rules!` (Part 36) ผ่าน proc macro พื้นฐาน (Part 44) มาจนถึง derive macro ขั้น
สูง (บทนี้)

**Module 3 (ระดับสูง)** จะเปลี่ยนทิศทางไปสู่หัวข้อใหญ่ถัดไปที่สำคัญไม่แพ้กัน: **Part 46: Async/Await
เบื้องต้น** จุดเริ่มต้นของการเขียนโปรแกรมแบบ asynchronous ใน Rust — `async fn`, `.await`, `Future`
trait และความแตกต่างพื้นฐานระหว่าง concurrency แบบ thread (ที่เราเรียนไปแล้วใน Part 37-40) กับ
concurrency แบบ async ก่อนจะขยายความลึกไปเรื่อย ๆ ตลอด Part 47-51 (futures/executors ภายใน, Tokio
runtime, I/O/networking, async channels, และ atomics) ซึ่งเป็นทักษะที่จำเป็นที่สุดสำหรับการเขียน
web service และระบบ network ในโมดูล 4 ที่จะตามมา

---

**Part ก่อนหน้า:** [Procedural Macros เบื้องต้น](part-044-proc-macros-basics.md) | **Part ถัดไป:**
[Async/Await เบื้องต้น](part-046-async-await-basics.md)
