# Part 16: Modules และการจัดระเบียบโค้ด

> โมดูล: พื้นฐานภาษา Rust (Core Language Fundamentals) | ระดับ: กลาง | เวลาโดยประมาณ: 200 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้ว่าทำไมโปรแกรมที่โตขึ้นจนมีทุกอย่างอยู่ใน `main.rs` ไฟล์เดียวถึงเริ่มควบคุมไม่ได้ และ **module** แก้ปัญหานั้น
  อย่างไรในระดับโครงสร้างโค้ด
- นิยาม module แบบ inline ด้วย `mod`, นิยาม module ซ้อนกันหลายชั้น, และอธิบาย **module tree** (ต้นไม้ของ module)
  ที่มี `crate` เป็นราก
- อธิบายได้อย่างลึกซึ้งว่าทำไม Rust เลือกให้ **ทุกอย่างเป็น private โดย default** (แม้แต่ระหว่าง sibling module
  ด้วยกันเอง) ต่างจากภาษาอื่นอย่าง Python ที่ทุกอย่างเข้าถึงได้เป็นปกติ และแก้ error เรื่อง privacy จริงที่ compiler
  แจ้งได้อย่างถูกจุด
- ใช้ **path** ทั้งแบบ absolute (`crate::...`) และ relative (`self::...`, `super::...`) ได้อย่างถูกต้อง, ใช้ `use`
  เพื่อดึง path เข้ามาในขอบเขต, ใช้ `use ... as` ตั้งชื่อเล่น, และใช้ `pub use` เพื่อ **re-export** สร้าง public API
  ที่สะอาด
- แยก module ออกเป็นไฟล์แยกต่างหากได้ทั้งสไตล์ปัจจุบัน (`src/inventory.rs`) และสไตล์เก่า (`src/inventory/mod.rs`)
  พร้อมสร้าง module tree ที่มีหลายไฟล์ซ้อนกันหลายระดับ
- ปรับความละเอียดของ visibility ด้วย `pub(crate)` และ `pub(super)` เพื่อสร้าง "internal API" ที่ใช้ได้เฉพาะบางส่วน
  ของโปรแกรม โดยไม่เปิดเผยสู่ภายนอกทั้งหมด
- ออกแบบ struct ที่ field เป็น private แม้ struct เองเป็น `pub` เพื่อบังคับให้ผู้ใช้ต้องผ่าน constructor/getter
  ที่ควบคุม invariant ได้ (ต่อยอดจาก associated function `new(...)` ที่เรียนใน Part 9)
- refactor โปรแกรมที่เขียนแบบไฟล์เดียวยุ่งเหยิงให้กลายเป็น module tree ที่จัดระเบียบดี พร้อม public API ที่สะอาด
  ผ่าน `pub use`

## ความรู้ที่ต้องมีมาก่อน

- **Part 9 (Structs)**: struct, `impl` block, associated function `new(...)` เป็น constructor, `&self`/`&mut self`
  — บทนี้จะนำแนวคิด constructor กลับมาใช้เป็นกลไกบังคับ encapsulation ผ่าน field privacy โดยตรง
- **Part 10 (Enums และ Pattern Matching)**: `enum`, `match` — ใช้ในตัวอย่างสถานะออเดอร์ (`OrderStatus`) ตลอดทั้งบท
- **Part 6-8 (Ownership, Borrowing, Slices)**: `&`, `&mut`, การยืมค่า — จำเป็นสำหรับเข้าใจ method บน struct ที่อยู่ใน
  module ต่าง ๆ ที่เรียกยืมกันไปมา
- **Part 11-15 (Option, Result, Vec, String, HashMap)**: ไม่จำเป็นต้องใช้ตรง ๆ ในบทนี้ แต่ตัวอย่างจริงจะสมมติว่าคุณ
  คุ้นเคยกับการเขียนโปรแกรมที่มีหลาย struct/enum ทำงานร่วมกันแล้ว (domain model) เพราะบทนี้คือขั้นต่อไปหลังจากออกแบบ
  domain model ได้ — คือการ **จัดระเบียบ** domain model นั้นให้อยู่ในที่ที่เหมาะสม

จนถึง Part 15 ทุกตัวอย่างในหลักสูตรนี้อยู่ใน `main.rs` ไฟล์เดียว (หรือ snippet สั้น ๆ ที่แยกจากกัน) ซึ่งเหมาะกับการเรียน
concept ทีละเรื่อง แต่ **ไม่เหมาะกับการเขียนโปรแกรมจริง** ที่มีขนาดใหญ่ขึ้นเรื่อย ๆ บทนี้คือจุดเปลี่ยนสำคัญที่เราจะเรียนรู้
วิธี "จัดบ้าน" ให้โค้ดที่เราเขียนมาตลอด 15 บท อยู่ในที่ที่ถูกต้องและขยายต่อได้ในระยะยาว

## เนื้อหา

### 16.1 ปัญหา: เมื่อ `main.rs` ไฟล์เดียวเริ่มควบคุมไม่ได้

ลองนึกภาพระบบร้านค้าออนไลน์เล็ก ๆ ที่เริ่มต้นด้วย struct และฟังก์ชันไม่กี่ตัว แล้วค่อย ๆ เพิ่มฟีเจอร์ไปเรื่อย ๆ ตามที่
เราได้ฝึกทำในหลาย ๆ Part ที่ผ่านมา — สินค้า (`Product`), ออเดอร์ (`Order`), ลูกค้า (`Customer`), การคำนวณส่วนลด,
การจัดการสต็อก ทั้งหมดนี้ยังคงอยู่ใน `main.rs` ไฟล์เดียวเหมือนที่เราทำมาตลอด:

```rust
// ตัวอย่าง "ก่อน": ทุกอย่างอยู่ใน main.rs ไฟล์เดียว ไม่มีการจัดกลุ่มใด ๆ เลย
struct Product {
    name: String,
    base_price_cents: i64,
    quantity: u32,
}

impl Product {
    fn new(name: &str, base_price_cents: i64, quantity: u32) -> Self {
        Product {
            name: name.to_string(),
            base_price_cents,
            quantity,
        }
    }

    fn total_value_cents(&self) -> i64 {
        self.base_price_cents * self.quantity as i64
    }
}

enum OrderStatus {
    Pending,
    Paid,
    Shipped,
    Cancelled,
}

struct Order {
    id: u32,
    status: OrderStatus,
    item_index: usize,
    item_quantity: u32,
}

struct Customer {
    name: String,
    is_vip: bool,
    loyalty_points: u32,
}

// ฟังก์ชันคำนวณส่วนลด อยู่ลอย ๆ ไม่ผูกกับใครเลย ทั้งที่จริง ๆ เป็นเรื่องของ "การตั้งราคา"
fn calculate_discount_percent(customer: &Customer, order_total_cents: i64) -> f64 {
    let mut discount = 0.0;
    if customer.is_vip {
        discount += 10.0;
    }
    if order_total_cents > 100_000 {
        discount += 5.0;
    }
    discount
}

fn apply_discount(total_cents: i64, discount_percent: f64) -> i64 {
    let discount_amount = (total_cents as f64) * (discount_percent / 100.0);
    total_cents - discount_amount as i64
}

// ฟังก์ชันจัดการสต็อก ก็ลอย ๆ อีกตัว ไม่รู้ว่าเกี่ยวกับ Product หรือ Order มากกว่ากัน
fn reduce_stock(catalog: &mut [Product], index: usize, amount: u32) -> bool {
    if catalog[index].quantity < amount {
        return false;
    }
    catalog[index].quantity -= amount;
    true
}

fn describe_order_status(status: &OrderStatus) -> &'static str {
    match status {
        OrderStatus::Pending => "รอดำเนินการ",
        OrderStatus::Paid => "จ่ายเงินแล้ว",
        OrderStatus::Shipped => "จัดส่งแล้ว",
        OrderStatus::Cancelled => "ยกเลิกแล้ว",
    }
}

fn main() {
    let mut catalog = vec![
        Product::new("เมาส์ไร้สาย", 29_900, 20),
        Product::new("คีย์บอร์ดเมคานิคอล", 89_000, 10),
        Product::new("จอมอนิเตอร์ 27 นิ้ว", 650_000, 5),
    ];

    let customer = Customer {
        name: String::from("สมชาย"),
        is_vip: true,
        loyalty_points: 120,
    };

    let order = Order {
        id: 1001,
        status: OrderStatus::Pending,
        item_index: 1,
        item_quantity: 2,
    };

    let unit_total = catalog[order.item_index].base_price_cents * order.item_quantity as i64;
    let discount = calculate_discount_percent(&customer, unit_total);
    let final_total = apply_discount(unit_total, discount);

    if reduce_stock(&mut catalog, order.item_index, order.item_quantity) {
        println!(
            "ลูกค้า {} (แต้มสะสม {}) สั่งซื้อ {} จำนวน {} ชิ้น",
            customer.name,
            customer.loyalty_points,
            catalog[order.item_index].name,
            order.item_quantity
        );
        println!("ออเดอร์เลขที่ {} สถานะ: {}", order.id, describe_order_status(&order.status));
        println!("ยอดรวมหลังหักส่วนลด {:.1}%: {} บาท", discount, final_total as f64 / 100.0);
    } else {
        println!("สต็อกไม่พอ");
    }

    println!(
        "มูลค่าสต็อกทั้งหมดของสินค้าตัวแรก: {} บาท",
        catalog[0].total_value_cents() as f64 / 100.0
    );
}
```

ผลลัพธ์:

```
ลูกค้า สมชาย (แต้มสะสม 120) สั่งซื้อ คีย์บอร์ดเมคานิคอล จำนวน 2 ชิ้น
ออเดอร์เลขที่ 1001 สถานะ: รอดำเนินการ
ยอดรวมหลังหักส่วนลด 15.0%: 1513 บาท
มูลค่าสต็อกทั้งหมดของสินค้าตัวแรก: 5980 บาท
```

โปรแกรมนี้ **compile ผ่านและทำงานถูกต้องทุกอย่าง** — ในมุมของ compiler ไม่มีปัญหาอะไรเลย แต่ลองมองในมุมของคนที่ต้อง
**ดูแลโค้ดนี้ต่อไปอีกหลายเดือนหรือหลายปี** จะเห็นปัญหาเชิงโครงสร้างที่ชัดเจนหลายจุด:

1. **ไม่มีการจัดกลุ่มตามความรับผิดชอบ (responsibility)** — `Product`, `Order`, `Customer` ปนกันอยู่กับฟังก์ชัน
   คำนวณส่วนลด (`calculate_discount_percent`, `apply_discount`) และฟังก์ชันจัดการสต็อก (`reduce_stock`) โดยไม่มีอะไร
   บอกเลยว่าฟังก์ชันไหน "เป็นเรื่องของ" อะไร ต้องอ่านชื่อฟังก์ชันแล้วเดาเอาเองว่ามันเกี่ยวกับเรื่องไหน
2. **ไม่มีขอบเขตการเข้าถึง (encapsulation)** — ทุก field ของทุก struct เข้าถึงได้จากทุกที่ในไฟล์ ใครก็เขียน
   `catalog[0].quantity = 999_999;` แบบข้าม logic การตรวจสอบใน `reduce_stock` ไปเลยก็ได้ ไม่มีอะไรห้าม เพราะทุกอย่าง
   อยู่ใน scope เดียวกันหมด
3. **สเกลไม่ได้เมื่อไฟล์ใหญ่ขึ้น** — ลองนึกภาพว่าระบบนี้โตขึ้นจนมี struct 30 ตัว ฟังก์ชัน 200 ตัว ทุกอย่างยังอยู่ใน
   ไฟล์เดียว การ scroll หาสิ่งที่ต้องการแก้จะกลายเป็นปัญหาใหญ่ และทีมที่ทำงานพร้อมกันหลายคนก็จะแก้ไฟล์เดียวกันจนเกิด
   merge conflict ใน git บ่อยมาก
4. **ทดสอบเป็นส่วน ๆ ไม่ได้ง่าย** — ถ้าอยากทดสอบเฉพาะ logic การคำนวณส่วนลด โดยไม่ต้องยุ่งกับ `Order`/`Customer` เลย
   ก็ทำได้ยาก เพราะไม่มีขอบเขตแยกให้เห็นชัดว่าอะไรคือ "หน่วยที่แยกทดสอบได้"
5. **ชื่อชนกันง่ายเมื่อโปรแกรมโตขึ้น** — สมมติวันหนึ่งต้องเพิ่มระบบ "การจัดส่ง" (shipping) ที่ก็มีแนวคิดเรื่อง
   `apply_discount` เป็นของตัวเอง (เช่นส่วนลดค่าส่ง) ชื่อฟังก์ชันจะชนกับ `apply_discount` ของราคาสินค้าทันที เพราะทุกอย่าง
   อยู่ใน namespace เดียวกันหมด (namespace คือขอบเขตของชื่อ — ถ้าสองสิ่งอยู่ใน namespace เดียวกัน ชื่อซ้ำกันไม่ได้)

ปัญหาทั้งหมดนี้ไม่ใช่ปัญหาทาง "ไวยากรณ์" (syntax) แต่เป็นปัญหาทาง **"โครงสร้าง" (structure)** — Rust แก้ปัญหานี้ด้วย
ระบบ **module** ที่ให้เราจัดกลุ่มโค้ดที่เกี่ยวข้องกันไว้ด้วยกัน กำหนดขอบเขตการเข้าถึง (privacy) อย่างชัดเจน และแยก
namespace ของแต่ละกลุ่มออกจากกัน — เนื้อหาที่เหลือทั้งบทนี้คือการเรียนรู้ระบบนี้อย่างละเอียด

### 16.2 `mod` Keyword: นิยาม Module แบบ Inline และ Module Tree

วิธีที่ง่ายที่สุดในการสร้าง module คือเขียน `mod ชื่อ_module { ... }` ครอบกลุ่มโค้ดที่เกี่ยวข้องกันไว้ในไฟล์เดียวกันก่อน
(ยังไม่ต้องแยกไฟล์ — เราจะเรียนการแยกไฟล์ในหัวข้อ 16.6) ลองจัดกลุ่มส่วนที่เกี่ยวกับ "สินค้าและการตั้งราคา" จากตัวอย่าง
ก่อนหน้าให้อยู่ใน module ชื่อ `inventory`:

```rust
// ตัวอย่าง mod แบบ inline: ประกาศ module ไว้ในไฟล์เดียวกันด้วย mod { ... }
mod inventory {
    // struct และ field ทุกตัวยังเป็น private ตาม default ในตอนนี้ (ยังไม่ใส่ pub)
    pub struct Product {
        pub name: String,
        pub base_price_cents: i64,
        pub quantity: u32,
    }

    impl Product {
        pub fn new(name: &str, base_price_cents: i64, quantity: u32) -> Self {
            Product {
                name: name.to_string(),
                base_price_cents,
                quantity,
            }
        }

        pub fn total_value_cents(&self) -> i64 {
            self.base_price_cents * self.quantity as i64
        }
    }

    // module ย่อยที่ซ้อนอยู่ภายใน inventory อีกชั้นหนึ่ง — module tree แบบ inventory::pricing
    pub mod pricing {
        pub fn apply_discount(total_cents: i64, discount_percent: f64) -> i64 {
            let discount_amount = (total_cents as f64) * (discount_percent / 100.0);
            total_cents - discount_amount as i64
        }
    }
}

fn main() {
    // เข้าถึงผ่าน path เต็มจาก module root: inventory::Product
    let mouse = inventory::Product::new("เมาส์ไร้สาย", 29_900, 20);
    println!("{} มูลค่ารวม {} สตางค์", mouse.name, mouse.total_value_cents());

    // เข้าถึง function ใน module ย่อยด้วย path inventory::pricing::apply_discount
    let discounted = inventory::pricing::apply_discount(mouse.total_value_cents(), 10.0);
    println!("หลังหักส่วนลด 10%: {} สตางค์", discounted);
}
```

ผลลัพธ์:

```
เมาส์ไร้สาย มูลค่ารวม 598000 สตางค์
หลังหักส่วนลด 10%: 538200 สตางค์
```

สังเกตโครงสร้างสำคัญที่เกิดขึ้น: เรามี **module tree** (ต้นไม้ของ module) หน้าตาประมาณนี้:

```
crate (main.rs)
└── inventory
    ├── Product
    └── pricing
        └── apply_discount
```

**`crate` คือรากของ module tree เสมอ** — ทุกอย่างที่เขียนอยู่นอก `mod { ... }` ใด ๆ เลย (เช่น `fn main()` ในตัวอย่างนี้)
ถือว่าอยู่ใน **module root ของ crate** (เรียกสั้น ๆ ว่า "crate root") เมื่อเราเขียน `mod inventory { ... }` เราก็สร้าง
"กิ่ง" ใหม่ในต้นไม้นี้ชื่อ `inventory` และเมื่อเขียน `pub mod pricing { ... }` ซ้อนอยู่ **ภายใน** `mod inventory` อีกที
เราก็สร้าง "กิ่งย่อย" `inventory::pricing` ซ้อนลึกเข้าไปอีกชั้นหนึ่ง

**ทำไมต้องมีแนวคิดต้นไม้แบบนี้?** เพราะมันให้ **namespace ที่ไม่ชนกัน** อย่างเป็นระบบ — ถ้าวันหนึ่งเราต้องการเขียน
`shipping::pricing::apply_discount` (ส่วนลดค่าส่ง) แยกจาก `inventory::pricing::apply_discount` (ส่วนลดราคาสินค้า)
ทั้งสองฟังก์ชันชื่อ `apply_discount` เหมือนกันเป๊ะ แต่อยู่คนละ path ในต้นไม้ จึงไม่ชนกันเลย — นี่คือประโยชน์ข้อแรกที่
module แก้ปัญหาข้อ 5 จากหัวข้อ 16.1 ได้ทันที

ข้อสังเกตด้าน syntax ที่สำคัญ:

- `mod inventory { ... }` ไม่มี `;` ต่อท้าย เพราะเนื้อหาของ module เขียนอยู่ในวงเล็บปีกกาทันที (ต่างจากรูปแบบแยกไฟล์
  ที่เราจะเรียนในหัวข้อ 16.6 ซึ่งใช้ `mod inventory;` ที่มี `;` ต่อท้ายแทน)
- ชื่อ module (`inventory`, `pricing`) เขียนด้วย **snake_case** ตาม convention เดียวกับตัวแปรและฟังก์ชัน (ต่างจากชื่อ
  struct/enum ที่ใช้ PascalCase อย่างที่เรียนมาใน Part 9-10)
- module สามารถซ้อนกันได้ **ไม่จำกัดจำนวนชั้น** — `inventory::pricing::seasonal::christmas_discount` ก็เป็นไปได้ถ้า
  ซ้อน `mod` เข้าไปเรื่อย ๆ แม้ในทางปฏิบัติการซ้อนลึกเกิน 3-4 ชั้นมักเป็นสัญญาณว่าควรจัดโครงสร้างใหม่

### 16.3 กฎ Privacy: ทุกอย่างเป็น Private โดย Default

ในตัวอย่างหัวข้อ 16.2 เราใส่ `pub` ไว้หน้า `struct Product`, หน้าทุก field, หน้า `impl` method ทุกตัว, และหน้า
`mod pricing` และ `fn apply_discount` — ลองถามตัวเองว่า **ถ้าไม่ใส่ `pub` เหล่านี้ จะเกิดอะไรขึ้น?**

```rust
mod inventory {
    // ไม่ได้ใส่ pub ไว้หน้า struct — ตาม default แล้ว Product เป็น private
    // มองเห็นได้แค่จากภายใน module inventory เท่านั้น (และ module ลูกของมัน)
    struct Product {
        name: String,
    }

    impl Product {
        fn new(name: &str) -> Self {
            Product { name: name.to_string() }
        }
    }
}

fn main() {
    // พยายามเข้าถึง inventory::Product จากภายนอก module — Product เป็น private
    let p = inventory::Product::new("เมาส์ไร้สาย");
    println!("{}", p.name);
}
```

โค้ดนี้ **compile ไม่ผ่าน** พร้อม error ถึง 3 จุดพร้อมกัน (เพราะ privacy รั่วอยู่ 3 ชั้น: ตัว struct, associated
function, และ field):

```
error[E0603]: struct `Product` is private
  --> src/main.rs:17:24
   |
17 |     let p = inventory::Product::new("เมาส์ไร้สาย");
   |                        ^^^^^^^ private struct
   |
note: the struct `Product` is defined here
  --> src/main.rs:4:5
   |
 4 |     struct Product {
   |     ^^^^^^^^^^^^^^

error[E0624]: associated function `new` is private
  --> src/main.rs:17:33
   |
 9 |         fn new(name: &str) -> Self {
   |         -------------------------- private associated function defined here
...
17 |     let p = inventory::Product::new("เมาส์ไร้สาย");
   |                                 ^^^ private associated function

error[E0616]: field `name` of struct `inventory::Product` is private
  --> src/main.rs:18:22
   |
18 |     println!("{}", p.name);
   |                      ^^^^ private field

error: aborting due to 3 previous errors
```

**นี่คือกฎพื้นฐานที่สำคัญที่สุดของระบบ module ใน Rust: ทุก item (struct, enum, function, field, module, ...) เป็น
private โดย default เสมอ ไม่มีข้อยกเว้น** ต้องใส่ `pub` อย่างจงใจเท่านั้นถึงจะมองเห็นได้จากภายนอก module ที่นิยามมัน

**ทำไม Rust ถึงเลือก default แบบนี้?** นี่เป็นการตัดสินใจเชิงปรัชญาที่สำคัญมาก และต่างจากภาษาสาย scripting หลายภาษา
อย่างชัดเจน:

- **ใน Python**: attribute และ method ของ object ทุกตัว **เข้าถึงได้จากภายนอกเป็นค่าเริ่มต้นเสมอ** (การซ่อนข้อมูลทำได้
  แค่ด้วย convention ตั้งชื่อขึ้นต้นด้วย underscore เช่น `_internal` ซึ่งเป็นแค่ "สัญญาทางใจ" ไม่มี compiler บังคับจริง
  ใครจะเขียน `obj._internal = 999` ก็ทำได้เสมอ)
- **ใน Rust**: ทุกอย่างเป็น private จนกว่าคุณจะ**ประกาศอย่างจงใจ**ว่าต้องการเปิดเผยส่วนไหนให้ใครเห็น — เป็นแนวทาง
  **"ปิดโดย default เปิดเมื่อจำเป็น" (closed by default)** ตรงข้ามกับ **"เปิดโดย default"** ของภาษาสาย dynamic
  ส่วนใหญ่

การเลือก "ปิดโดย default" มีเหตุผลเชิงวิศวกรรมซอฟต์แวร์ที่ลึกซึ้งอยู่เบื้องหลัง:

1. **การเปลี่ยนแปลง implementation ภายในโดยไม่กระทบผู้ใช้ (encapsulation ที่แท้จริง)** — ถ้า field ทั้งหมดของ
   `Product` เป็น private และมีเพียง method ที่เป็น `pub` เท่านั้นที่เข้าถึงได้ วันหนึ่งถ้าคุณต้องการเปลี่ยนวิธีเก็บ
   `quantity` จาก `u32` เป็นระบบที่ซับซ้อนขึ้น (เช่นแยกเป็น `available` กับ `reserved`) คุณสามารถทำได้โดยไม่กระทบโค้ด
   ที่เรียกใช้ `Product` จากภายนอกเลย ตราบใดที่ method แบบ `pub` ยังคืนค่าแบบเดิม — นี่คือสิ่งที่เรียกว่า **API ที่
   คงที่ (stable API) กับ implementation ที่เปลี่ยนแปลงได้อย่างอิสระ**
2. **บังคับให้ผู้ออกแบบ module คิดอย่างจงใจว่าจะเปิดเผยอะไร** — เพราะ default คือปิด การใส่ `pub` แต่ละครั้งคือการ
   ตัดสินใจที่ตั้งใจ (deliberate decision) ว่า "นี่คือส่วนหนึ่งของสัญญา (contract) ที่ฉันสัญญาว่าจะดูแลให้คงที่
   ต่อไป" ต่างจากการเปิดทุกอย่างแบบ default ที่มักนำไปสู่สถานการณ์ที่ผู้ใช้ไปพึ่งพา (depend on) รายละเอียดภายในที่
   ผู้เขียนไม่ได้ตั้งใจจะรับประกันไว้ตั้งแต่แรก แล้วพอแก้ implementation นิดเดียวก็ทำโค้ดคนอื่นพังไปทั่ว
3. **compiler ช่วยตรวจสอบขอบเขตให้แทนมนุษย์** — ในภาษาที่พึ่งพา convention (เช่น underscore ใน Python) การ "ทำผิด"
   (เข้าถึง `_internal` ตรง ๆ) ไม่มีอะไรเตือนเลยจนกว่าจะเกิดบั๊กจริงตอนรัน ในขณะที่ Rust จับได้ตั้งแต่ก่อน compile
   เสร็จด้วยซ้ำ

จำหลักการสำคัญนี้ไว้: **`pub` ไม่ใช่การ "ปลดล็อกสิ่งที่ถูกล็อกไว้ผิดพลาด" แต่เป็นการ "ประกาศ API สาธารณะอย่างตั้งใจ"**
ทุกครั้งที่คุณเห็น error เรื่อง privacy ให้ถามตัวเองก่อนว่า **"ฉันตั้งใจจริง ๆ หรือเปล่าที่จะให้จุดนี้เป็นส่วนหนึ่งของ
สิ่งที่โค้ดภายนอกเรียกใช้ได้?"** ถ้าใช่ค่อยเติม `pub` ถ้าไม่แน่ใจ การปล่อยเป็น private ไว้ก่อนมักปลอดภัยกว่าเสมอ
(เพิ่ม `pub` ทีหลังได้ง่าย แต่การเอา `pub` ออกทีหลังอาจทำโค้ดที่พึ่งพามันอยู่แล้วพังได้)

วิธีแก้ตัวอย่างข้างบนคือเติม `pub` ให้ครบทั้ง 3 จุดที่ error แจ้ง — ตรงกับที่เราเขียนไว้แล้วในหัวข้อ 16.2:

```rust
mod inventory {
    pub struct Product {
        pub name: String,
    }

    impl Product {
        pub fn new(name: &str) -> Self {
            Product { name: name.to_string() }
        }
    }
}

fn main() {
    let p = inventory::Product::new("เมาส์ไร้สาย");
    println!("{}", p.name);
}
```

ผลลัพธ์:

```
เมาส์ไร้สาย
```

**ข้อสังเกตสำคัญ**: การใส่ `pub` หน้า `struct Product` เพียงอย่างเดียว **ไม่ได้ทำให้ field ข้างในเป็น `pub` ไปด้วย
โดยอัตโนมัติ** — `pub struct` แค่บอกว่า "ชนิดข้อมูลนี้ชื่อ `Product` มองเห็นได้จากภายนอก" ส่วน field แต่ละตัวยังต้อง
ประกาศ `pub` ของตัวเองแยกกัน นี่คือกฎที่มือใหม่มักงงบ่อยที่สุด และเราจะเจาะลึกเรื่องนี้อีกครั้งอย่างละเอียดในหัวข้อ 16.9
พร้อมเหตุผลว่าทำไมการแยกความ pub ของ struct กับ pub ของ field ออกจากกันถึงมีประโยชน์มาก

### 16.4 Path: Absolute vs Relative, `use`, `use ... as`, และ `pub use`

เมื่อ module tree ซับซ้อนขึ้น เราต้องมีวิธี "ชี้" ไปยัง item ใด ๆ ในต้นไม้นั้นอย่างแม่นยำ — สิ่งนี้เรียกว่า **path**
(เส้นทาง) Rust รองรับ path 2 รูปแบบหลัก:

- **Absolute path (เส้นทางสัมบูรณ์)**: เริ่มจาก `crate` (รากของ module tree) เสมอ ไม่ว่าโค้ดที่เขียน path นี้จะอยู่
  ลึกแค่ไหนในต้นไม้ก็ตาม เช่น `crate::inventory::pricing::apply_discount`
- **Relative path (เส้นทางสัมพัทธ์)**: เริ่มจากตำแหน่งปัจจุบันของโค้ด ใช้ `self::` เพื่อชี้ไปยัง item ใน module
  ปัจจุบัน หรือ `super::` เพื่อชี้ไปยัง module พ่อ (เราจะเจาะลึก `super::` แยกในหัวข้อ 16.5)

```rust
mod inventory {
    pub struct Product {
        pub name: String,
        pub price_cents: i64,
    }

    pub mod pricing {
        // path แบบ absolute: เริ่มจาก crate (module root) เสมอ ไม่ว่าโค้ดนี้จะอยู่ลึกแค่ไหนของ tree
        // ใช้เพื่ออ้างถึง Product โดยไม่ขึ้นกับว่า pricing ซ้อนอยู่ตรงไหนของ tree
        pub fn markup_price(product: &crate::inventory::Product) -> i64 {
            product.price_cents * 120 / 100 // บวก markup 20%
        }

        // path แบบ relative ด้วย self:: — ชี้กลับไปยัง item ภายใน module ปัจจุบัน (pricing) เอง
        // ในที่นี้ self::markup_price ก็คือฟังก์ชันข้างบนตัวเดียวกัน เขียนแบบ relative แทน absolute
        pub fn markup_price_relative(product: &crate::inventory::Product) -> i64 {
            self::markup_price(product)
        }
    }
}

// pub use ใน module ใหม่ชื่อ catalog: "re-export" สร้าง path ใหม่ที่สั้นและสะอาดกว่าให้ผู้ใช้ crate
// ผู้เรียกใช้ catalog::Product ได้โดยไม่ต้องรู้เลยว่าจริง ๆ ของจริงมันมาจาก inventory::Product
mod catalog {
    pub use crate::inventory::Product;
    pub use crate::inventory::pricing::markup_price;
}

// use ธรรมดา: ดึง path มาไว้ในขอบเขตปัจจุบัน เพื่อไม่ต้องเขียน path เต็มซ้ำ ๆ ทุกครั้ง
use crate::inventory::Product;

// use ... as: ตั้งชื่อเล่นให้ path เดิม เวลาชื่อเดิมยาวเกินไป หรือชนกับชื่ออื่นในขอบเขตเดียวกัน
use crate::inventory::pricing as pr;

fn main() {
    // เพราะมี `use crate::inventory::Product;` ด้านบน จึงเขียนแค่ Product ตรง ๆ ได้เลย
    // ไม่ต้องเขียน inventory::Product::{ ... } แบบเต็มทุกครั้ง
    let mouse = Product {
        name: String::from("เมาส์ไร้สาย"),
        price_cents: 29_900,
    };

    // เรียกผ่าน alias pr ที่ตั้งด้วย use ... as
    println!("ราคาบวก markup: {} สตางค์", pr::markup_price(&mouse));
    println!(
        "ราคาบวก markup (relative path version): {} สตางค์",
        pr::markup_price_relative(&mouse)
    );

    // เรียกผ่าน path ที่ re-export มาจาก catalog — สั้นและซ่อนรายละเอียดภายในของ inventory ไว้
    let keyboard = catalog::Product {
        name: String::from("คีย์บอร์ดเมคานิคอล"),
        price_cents: 89_000,
    };
    println!(
        "{} ราคาบวก markup ผ่าน catalog::markup_price: {} สตางค์",
        keyboard.name,
        catalog::markup_price(&keyboard)
    );
}
```

ผลลัพธ์:

```
ราคาบวก markup: 35880 สตางค์
ราคาบวก markup (relative path version): 35880 สตางค์
คีย์บอร์ดเมคานิคอล ราคาบวก markup ผ่าน catalog::markup_price: 106800 สตางค์
```

มาแยกวิเคราะห์แต่ละส่วนของตัวอย่างนี้:

**`crate::` (absolute path)** — ข้อดีคือ **อ่านแล้วรู้ทันทีว่า item นั้นอยู่ที่ไหนในต้นไม้ทั้งหมด** ไม่ต้องคำนวณจาก
ตำแหน่งปัจจุบันของโค้ด เหมาะมากเมื่อโค้ดถูก copy ย้ายไปมาระหว่าง module บ่อย ๆ เพราะ absolute path ไม่เปลี่ยนความหมาย
ไม่ว่าจะ copy ไปวางตรงไหนของไฟล์ก็ตาม (ต่างจาก relative path ที่ความหมายเปลี่ยนไปตามตำแหน่งที่วาง)

**`self::` (relative path ชี้ในตัวเอง)** — ใช้ได้ไม่บ่อยเท่า `crate::` หรือ `super::` เพราะปกติแล้วการเรียก item
ในโมดูลเดียวกันไม่จำเป็นต้องเขียน prefix อะไรเลย (เขียน `markup_price(product)` ตรง ๆ ก็พอ) แต่ `self::` มีประโยชน์
เมื่อต้องการ **แยกแยะให้ชัดเจน** ว่ากำลังอ้างถึง item ในโมดูลปัจจุบัน ไม่ใช่ item ที่ชื่อเดียวกันจาก `use` ที่ดึงมาจาก
ที่อื่น (ป้องกันความสับสนเรื่อง namespace ชนกัน)

**`use`** — คือกลไก "ดึง path เข้ามาไว้ใน scope ปัจจุบัน" เพื่อไม่ต้องเขียน path เต็มซ้ำ ๆ ทุกครั้ง เทียบได้กับ `import`
ใน Python หรือ JavaScript (แนวคิดคล้ายกันมาก แต่ Rust ไม่มีผลข้างเคียงเรื่อง module-level side effect ตอน import
เหมือนบางภาษา) `use crate::inventory::Product;` ทำให้เราเขียน `Product { ... }` ตรง ๆ ได้ในบรรทัดถัดไปทั้งไฟล์
โดยไม่ต้องเขียน `inventory::Product` เต็ม ๆ ทุกครั้ง

**`use ... as`** — ตั้ง **ชื่อเล่น (alias)** ให้ path ที่ดึงมา มีประโยชน์ 2 กรณีหลัก: (1) เมื่อชื่อเดิมยาวเกินไปและ
อยากเรียกสั้น ๆ อย่าง `pr` แทน `pricing` ในตัวอย่างข้างบน และ (2) เมื่อ `use` สองอันจากคนละที่มีชื่อสุดท้ายชนกัน เช่น
ถ้ามีทั้ง `inventory::pricing` และ `shipping::pricing` ต้องใช้ `as` เพื่อแยกแยะไม่ให้ชนกัน (`use
crate::shipping::pricing as shipping_pricing;`)

**`pub use` (re-export)** — นี่คือกลไกที่ทรงพลังที่สุดในหัวข้อนี้ `pub use` ไม่ได้แค่ "ดึงเข้ามาใช้ในโมดูลปัจจุบัน"
แบบ `use` ธรรมดา แต่ยัง **"ส่งต่อ" (re-export)** ให้ path นั้นเข้าถึงได้จากภายนอก module ปัจจุบันด้วยเสมือนว่ามัน
ถูกนิยามอยู่ในโมดูลนี้เอง ในตัวอย่างข้างบน `catalog::Product` และ `catalog::markup_price` จริง ๆ แล้ว "ของจริง" อยู่
ที่ `inventory::Product` และ `inventory::pricing::markup_price` แต่ผู้ใช้ที่เรียกผ่าน `catalog::` ไม่จำเป็นต้องรู้
รายละเอียดนี้เลย — พวกเขาเห็นแค่ path `catalog::Product` ที่ดูสะอาดและเข้าใจง่าย ทั้งที่ภายในอาจมี module ย่อยซับซ้อน
กว่านั้นมาก

`pub use` คือกลไกหลักที่ทำให้เกิด **facade pattern** (สร้างหน้าตาสาธารณะที่เรียบง่าย ซ่อนความซับซ้อนภายในไว้) เรา
จะเห็นประโยชน์ของมันเต็มรูปแบบในตัวอย่าง refactor ท้ายบท (หัวข้อ 16.10) ที่ module `catalog` จะทำหน้าที่เป็น "หน้าร้าน"
รวม public API ที่ผู้ใช้ต้องการทั้งหมดไว้ที่เดียว โดยไม่ต้องเปิดเผยว่าภายในแยก module ย่อยกี่ตัว

**หมายเหตุเรื่อง glob import (`use module::*`)**: Rust รองรับการดึงทุก `pub` item ในโมดูลหนึ่งเข้ามาพร้อมกันด้วย
`use inventory::*;` ซึ่งสะดวกในบางสถานการณ์ (เช่นใน test module หรือ "prelude" module ที่ตั้งใจให้ดึงมาทั้งหมด)
แต่ **ไม่แนะนำให้ใช้เป็นปกติกับ public API ทั่วไป** เพราะทำให้ผู้อ่านโค้ดไม่รู้ว่าชื่อที่ใช้อยู่มาจากไหนกันแน่
(อ่านโค้ดแล้วต้องเดา หรือเปิด IDE ไปหาที่มาเอง) การเขียน `use` แบบระบุชื่อชัดเจนทีละตัว หรือ `use module::{A, B, C};`
(กลุ่มหลายตัวในวงเล็บปีกกา) ยังคงเป็นแนวทางที่แนะนำสำหรับโค้ดโปรดักชันทั่วไป

### 16.5 `super`: อ้างอิงกลับไปยัง Module พ่อ

`super::` คือ relative path พิเศษที่ชี้ไปยัง **module ที่เป็นพ่อของ module ปัจจุบันโดยตรง** เปรียบได้กับการเขียน
`../` ใน path ของไฟล์ระบบ (filesystem path) ที่หมายถึง "ขึ้นไปหนึ่งโฟลเดอร์" ประโยชน์ที่สำคัญที่สุดของ `super::` ไม่ใช่
แค่ความสะดวกในการเขียน path สั้นลง แต่เกี่ยวพันกับกฎ privacy โดยตรง:

**กฎที่สำคัญมาก**: item หนึ่ง ๆ ที่เป็น private (ไม่มี `pub`) จะ **มองไม่เห็นจากภายนอก module ที่นิยามมัน** ก็จริง แต่
"ภายนอก module" ในที่นี้ **ไม่รวมโมดูลลูกของมันเอง** — module ใด ๆ จะมองเห็น item private ของ **ตัวเองและของทุก
module บรรพบุรุษ (ancestor) ของมันเสมอ** พูดง่าย ๆ คือ: **ลูกมองเห็นของพ่อได้ (แม้พ่อไม่ได้ตั้งใจเปิดให้คนอื่นเห็น)
แต่คนนอกมองไม่เห็น**

ลองดูตัวอย่างที่แสดงกฎนี้อย่างเป็นรูปธรรม:

```rust
mod inventory {
    // ฟังก์ชันนี้ไม่มี pub เลย — private สนิท มองไม่เห็นจากนอก module inventory แม้แต่นิดเดียว
    // แต่ "นอก module inventory" ไม่ได้แปลว่ารวม module ลูกของ inventory เองด้วย!
    fn warehouse_code() -> &'static str {
        "WH-42"
    }

    pub mod pricing {
        // pricing เป็น module ลูกของ inventory (inventory::pricing)
        // super:: จากตรงนี้หมายถึง "กลับไปที่ module พ่อ" คือ inventory นั่นเอง
        // กฎ privacy ของ Rust ระบุว่า item ของ module ใด ๆ มองเห็นได้จากตัวมันเองและจาก
        // "ทุก module ลูกของมัน" เสมอ (ไม่ใช่แค่มองเห็นจากภายนอกไม่ได้) ดังนั้น pricing
        // ในฐานะลูกของ inventory จึงมองเห็น warehouse_code() ได้ ทั้งที่มันไม่มี pub เลย
        pub fn shipping_label(product_name: &str) -> String {
            format!("{} จัดส่งจากโกดัง {}", product_name, super::warehouse_code())
        }
    }
}

fn main() {
    println!("{}", inventory::pricing::shipping_label("เมาส์ไร้สาย"));

    // ลองพิสูจน์ว่า warehouse_code() ยังคง private จริงจากมุมมองนอก inventory ทั้งหมด
    // (คอมเมนต์บรรทัดล่างไว้ เพราะถ้า uncomment จะ compile ไม่ผ่านด้วย E0603)
    // println!("{}", inventory::warehouse_code());
}
```

ผลลัพธ์:

```
เมาส์ไร้สาย จัดส่งจากโกดัง WH-42
```

ถ้าเรา uncomment บรรทัดสุดท้ายเพื่อพิสูจน์ว่า `warehouse_code()` ยัง private จริงจากมุมมองของ `main()` (ซึ่งอยู่ที่
crate root ไม่ใช่ลูกของ `inventory`) จะได้ error ทันที:

```
error[E0603]: function `warehouse_code` is private
  --> src/main.rs:25:31
   |
25 |     println!("{}", inventory::warehouse_code());
   |                               ^^^^^^^^^^^^^^ private function
   |
note: the function `warehouse_code` is defined here
  --> src/main.rs:4:5
   |
 4 |     fn warehouse_code() -> &'static str {
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

นี่คือสิ่งที่พิสูจน์ว่า `warehouse_code()` ไม่ได้ "รั่ว" ออกไปทั่วโปรแกรม — มันยัง private สนิทจากมุมมองของ `main()`
เหมือนเดิม แต่ `pricing` (ลูกของ `inventory`) มองเห็นมันได้เพราะเป็น "คนในครอบครัวเดียวกัน" ตามกฎ ancestor ที่อธิบายไว้

**เมื่อไหร่ควรใช้ pattern นี้จริง?** สถานการณ์ที่พบบ่อยมากคือ: module พ่อมี "รายละเอียดการทำงานภายใน" (implementation
detail) ที่ต้องการให้ module ลูกที่เกี่ยวข้องกันใช้ร่วมกันได้ โดยไม่ต้องการเปิดเผยรายละเอียดนั้นออกไปสู่ภายนอกทั้งหมด
เช่นในตัวอย่างข้างบน `warehouse_code()` อาจเป็นค่าคงที่ภายในของระบบคลังสินค้าที่ทั้ง `inventory` และ module ลูกของมัน
(เช่น `pricing`, หรือถ้ามี `shipping` ซ้อนอยู่ในนั้นด้วย) ต้องใช้ร่วมกัน แต่ไม่มีเหตุผลอะไรที่โลกภายนอก (เช่น `main()`
หรือ module อื่นที่ไม่เกี่ยวข้อง) ควรรู้จักหรือเข้าถึงมันได้เลย `super::` จึงเป็นกลไกที่ให้ "การแบ่งปันแบบจำกัดวงในสาย
เครือญาติ" ได้อย่างพอดี — เราจะเห็น pattern ที่ใกล้เคียงกันแต่ปรับความละเอียดได้มากขึ้นอีกด้วย `pub(super)` ในหัวข้อ
16.8

### 16.6 แยก Module ออกเป็นไฟล์แยก: `mod inventory;` และสไตล์ปัจจุบันกับสไตล์เก่า

เมื่อ module โตขึ้นจนมีหลายร้อยบรรทัด การเก็บทุกอย่างไว้ใน `main.rs` ไฟล์เดียว (แม้จะจัดกลุ่มด้วย `mod { ... }` แล้ว
ก็ตาม) ก็ยังทำให้ไฟล์ใหญ่เทอะทะอยู่ดี Rust ให้เราแยกเนื้อหาของ module ออกไปเป็น **ไฟล์แยกต่างหาก** ได้ โดยเปลี่ยนจาก
`mod ชื่อ { ... }` (มีเนื้อหาตามหลังทันที) เป็น **`mod ชื่อ;`** (มี `;` แทน แล้วไม่มีเนื้อหาต่อ) — บรรทัดนี้บอก compiler
ว่า **"เนื้อหาของ module นี้อยู่ในไฟล์อื่น ไปหาเอาเอง"**

Rust รองรับการวางไฟล์ 2 รูปแบบสำหรับ `mod inventory;` ที่เขียนใน `src/main.rs`:

1. **สไตล์ปัจจุบัน (ตั้งแต่ Rust 2018 เป็นต้นมา)**: เนื้อหาอยู่ที่ `src/inventory.rs` — ไฟล์เดี่ยว ๆ ชื่อตรงกับ
   module วางอยู่ระดับเดียวกับ `main.rs`
2. **สไตล์เก่า (ตั้งแต่ Rust 2015)**: เนื้อหาอยู่ที่ `src/inventory/mod.rs` — สร้างโฟลเดอร์ชื่อ `inventory/` แล้ววาง
   ไฟล์ชื่อ `mod.rs` ไว้ข้างใน

ลองดูโครงสร้างไฟล์และเนื้อหาแบบสไตล์ปัจจุบันก่อน:

```
project/
└── src/
    ├── main.rs
    └── inventory.rs
```

```rust
// ไฟล์: src/main.rs
// การประกาศ `mod inventory;` (ไม่มี { ... } ตามหลัง) บอก Rust ว่า
// "เนื้อหาของ module inventory อยู่ในไฟล์แยกต่างหาก" — compiler จะไปหาไฟล์
// src/inventory.rs (สไตล์ใหม่ ตั้งแต่ edition 2018) โดยอัตโนมัติ
mod inventory;

use inventory::Product;

fn main() {
    let mouse = Product::new("เมาส์ไร้สาย", 29_900, 20);
    println!(
        "{} ราคาตั้ง {} สตางค์ จำนวน {} ชิ้น",
        mouse.name, mouse.base_price_cents, mouse.quantity
    );

    let final_price = inventory::pricing::apply_member_discount(mouse.base_price_cents, 10.0);
    println!("ราคาหลังหักส่วนลดสมาชิก 10%: {} สตางค์", final_price);
}
```

```rust
// ไฟล์: src/inventory.rs
// นี่คือเนื้อหาของ module inventory (มาแทนที่ mod inventory { ... } แบบ inline)
// สังเกตว่าในไฟล์นี้ไม่ต้องเขียน "mod inventory { ... }" ครอบอีกชั้น เพราะตัวไฟล์เองก็
// *เป็น* เนื้อหาของ module inventory อยู่แล้ว โดยอัตโนมัติจากชื่อไฟล์
pub struct Product {
    pub name: String,
    pub base_price_cents: i64,
    pub quantity: u32,
}

impl Product {
    pub fn new(name: &str, base_price_cents: i64, quantity: u32) -> Self {
        Product {
            name: name.to_string(),
            base_price_cents,
            quantity,
        }
    }
}

// ประกาศ module ลูกชื่อ pricing ที่เนื้อหาอยู่ในไฟล์แยก (รายละเอียดตำแหน่งไฟล์ในหัวข้อ 16.7)
pub mod pricing;
```

ผลลัพธ์เมื่อรันโปรแกรมนี้ (หลังจากสร้างไฟล์ `src/inventory/pricing.rs` ตามหัวข้อถัดไปด้วย):

```
เมาส์ไร้สาย ราคาตั้ง 29900 สตางค์ จำนวน 20 ชิ้น
ราคาหลังหักส่วนลดสมาชิก 10%: 26910 สตางค์
```

**ข้อสังเกตสำคัญที่สุดของหัวข้อนี้**: เนื้อหาข้างใน `src/inventory.rs` **ไม่ต้องเขียน `mod inventory { ... }` ครอบ
อีกชั้น** — สิ่งที่ต่างจากตัวอย่างหัวข้อ 16.2 (inline module) คือตอนนี้ **ตัวไฟล์เองคือเนื้อหาของ module** ไปโดยอัตโนมัติ
จากการที่ `main.rs` เขียน `mod inventory;` (ไม่มีเนื้อหาตามหลัง) ไว้ชี้มาที่ไฟล์นี้ กฎเรื่อง privacy, path, `use`,
`super::` ที่เรียนมาทั้งหมดในหัวข้อ 16.2-16.5 **ใช้งานเหมือนเดิมทุกประการ ไม่มีความแตกต่างเชิงพฤติกรรมเลย** — การแยก
ไฟล์เป็นเรื่องของ "จะเก็บเนื้อหาไว้ที่ไหน" เท่านั้น ไม่ใช่การเปลี่ยนกฎภาษาแต่อย่างใด

**ทำไมถึงมี 2 สไตล์?** สไตล์ `src/inventory/mod.rs` เป็นสไตล์ดั้งเดิมที่ Rust ใช้มาตั้งแต่ต้น (ก่อน edition 2018)
— เหตุผลตอนนั้นคือให้ path ของไฟล์บนดิสก์สอดคล้องกับ module tree แบบตรงไปตรงมาที่สุด (module มีลูกก็สร้างโฟลเดอร์
ชื่อตรงกัน แล้ว "ตัวมันเอง" อยู่ในไฟล์ `mod.rs` ข้างในโฟลเดอร์นั้น) แต่สไตล์นี้มีข้อเสียในทางปฏิบัติที่ชัดเจน:
**ถ้าเปิดหลายไฟล์พร้อมกันใน editor (เช่น `inventory/mod.rs`, `orders/mod.rs`, `catalog/mod.rs`) แท็บของ editor
ทุกแท็บจะแสดงชื่อ `mod.rs` เหมือนกันหมด** ทำให้แยกไม่ออกว่าแท็บไหนคือของ module ไหนกันแน่ ต้องเอาเมาส์ไปชี้ดู path
เต็มทุกครั้ง

ตั้งแต่ **Rust edition 2018** เป็นต้นมา จึงมีสไตล์ใหม่ที่ **ไม่บังคับให้สร้างไฟล์ชื่อ `mod.rs`** อีกต่อไป — module
ที่ไม่มีลูกเลยก็แค่เป็นไฟล์เดี่ยว ๆ ชื่อตรงกับ module (`inventory.rs`) วางเทียบชั้นกับไฟล์อื่น ทำให้แต่ละแท็บใน editor
มีชื่อไม่ซ้ำกันเลย อ่านง่ายขึ้นมาก **สไตล์นี้คือสไตล์ที่แนะนำและเป็น idiomatic (ตรงตามธรรมเนียมนิยม) สำหรับโค้ด Rust
ปัจจุบันทั้งหมด** ส่วนสไตล์ `mod.rs` ยังคงใช้งานได้อยู่ (compiler รองรับทั้งสองแบบเสมอ ไม่มีแผนจะเลิกรองรับ) แต่คุณ
จะเจอมันเป็นหลักในโค้ดเก่าที่เขียนก่อน edition 2018 หรือในโปรเจกต์ที่ยังไม่ได้ปรับปรุงสไตล์

ลองดูโครงสร้างและเนื้อหาแบบสไตล์เก่าเทียบกัน (ให้ผลลัพธ์การรันเหมือนกันทุกประการกับสไตล์ใหม่):

```
project/
└── src/
    ├── main.rs
    └── inventory/
        ├── mod.rs
        └── pricing.rs
```

```rust
// ไฟล์: src/inventory/mod.rs — สไตล์เก่า (ก่อน edition 2018): เนื้อหาของ module inventory
// ต้องอยู่ในไฟล์ mod.rs ภายในโฟลเดอร์ที่ชื่อตรงกับ module (inventory/)
pub struct Product {
    pub name: String,
    pub base_price_cents: i64,
    pub quantity: u32,
}

impl Product {
    pub fn new(name: &str, base_price_cents: i64, quantity: u32) -> Self {
        Product {
            name: name.to_string(),
            base_price_cents,
            quantity,
        }
    }
}

pub mod pricing;
```

สังเกตว่าเนื้อหาข้างในเหมือนกับ `src/inventory.rs` ของสไตล์ใหม่ทุกตัวอักษร — **ความแตกต่างมีแค่ตำแหน่งไฟล์บนดิสก์
เท่านั้น ไม่มีความแตกต่างด้าน syntax ภายในไฟล์เลยแม้แต่นิดเดียว** `main.rs` ของทั้งสองสไตล์ก็เขียน `mod inventory;`
เหมือนกันเป๊ะ — compiler เป็นผู้ตัดสินใจเองว่าจะไปหาไฟล์ที่ `src/inventory.rs` หรือ `src/inventory/mod.rs` โดยดูว่า
ไฟล์ไหนมีอยู่จริง (ถ้ามีทั้งคู่พร้อมกันจะเป็น error เพราะ ambiguous)

### 16.7 Module Tree แบบหลายไฟล์หลายระดับ

ตอนนี้เรามาต่อยอดตัวอย่างสไตล์ใหม่จากหัวข้อ 16.6 โดยเติมเนื้อหาให้ `pub mod pricing;` ที่ประกาศไว้ใน
`src/inventory.rs` — คำถามคือ: **เมื่อ module ที่แยกไฟล์แล้ว (`inventory.rs`) มี module ลูกที่ต้องแยกไฟล์ต่ออีก
(`pricing`) ไฟล์ของลูกควรอยู่ที่ไหน?**

กฎคือ: **module ลูกของ module ที่อยู่ในไฟล์ `<ชื่อ>.rs` (สไตล์ใหม่) จะต้องอยู่ในโฟลเดอร์ที่ชื่อตรงกับไฟล์นั้น (ไม่รวม
นามสกุล .rs)** เพราะ `inventory.rs` ประกาศ `pub mod pricing;` ไฟล์ของ `pricing` จึงต้องอยู่ที่ `src/inventory/pricing.rs`
(สร้างโฟลเดอร์ `inventory/` ขึ้นมาแม้ว่า `inventory` เองจะไม่ได้ใช้ไฟล์ `mod.rs` ก็ตาม) — โครงสร้างไฟล์แบบสไตล์ใหม่
ที่สมบูรณ์คือ:

```
project/
└── src/
    ├── main.rs
    ├── inventory.rs          <- เนื้อหา module inventory
    └── inventory/
        └── pricing.rs        <- เนื้อหา module inventory::pricing (module ลูก)
```

```rust
// ไฟล์: src/inventory/pricing.rs
// เนื้อหาของ module inventory::pricing — อยู่ในโฟลเดอร์ inventory/ เพราะไฟล์พ่อของมัน
// คือ src/inventory.rs (ไม่ใช่ src/inventory/mod.rs) นี่คือกฎการวาง path แบบ edition 2018+
pub fn apply_member_discount(price_cents: i64, discount_percent: f64) -> i64 {
    let discount_amount = (price_cents as f64) * (discount_percent / 100.0);
    price_cents - discount_amount as i64
}
```

สังเกตว่าไฟล์นี้ **ไม่มี `mod pricing { ... }` ครอบตัวเองเลย** เช่นเดียวกับที่ `inventory.rs` ไม่มี `mod inventory { ... }`
ครอบ — กฎเดิมจากหัวข้อ 16.6 ใช้ซ้ำกันในทุกระดับความลึกของ module tree เสมอ: **ไฟล์ที่ module ประกาศ `mod ลูก;` ชี้มา
ถึง คือเนื้อหาของ module นั้นโดยตรง ไม่ต้องมี wrapper ใด ๆ ครอบอีก**

ถ้าเป็นสไตล์เก่า (`mod.rs`) โครงสร้างจะต่างออกไปเล็กน้อย เพราะ `inventory` เองก็อยู่ในโฟลเดอร์ `inventory/` แล้ว
(ที่มีไฟล์ `mod.rs`) module ลูกของมันจึงอยู่ **ในโฟลเดอร์เดียวกัน** ไม่ต้องสร้างโฟลเดอร์ซ้อนเพิ่ม:

```
project/
└── src/
    ├── main.rs
    └── inventory/
        ├── mod.rs            <- เนื้อหา module inventory
        └── pricing.rs        <- เนื้อหา module inventory::pricing (อยู่ในโฟลเดอร์เดียวกับ mod.rs)
```

นี่คือความแตกต่างเชิงโครงสร้างไฟล์ที่ชัดเจนระหว่างสองสไตล์: สไตล์เก่า module ลูกอยู่ **โฟลเดอร์เดียวกัน** กับไฟล์
`mod.rs` ของพ่อ ส่วนสไตล์ใหม่ module ลูกอยู่ **โฟลเดอร์ที่ชื่อตรงกับไฟล์ `.rs` ของพ่อ** (ไม่ใช่โฟลเดอร์เดียวกับไฟล์
`.rs` นั้น) หากซ้อนลึกไปอีกระดับ (เช่น `pricing` มี module ลูกชื่อ `seasonal`) กฎเดียวกันนี้จะใช้ซ้ำไปเรื่อย ๆ:
สไตล์ใหม่จะได้ `src/inventory/pricing/seasonal.rs`, สไตล์เก่าจะได้ `src/inventory/pricing/seasonal.rs` เหมือนกัน
พอดี (เพราะ `pricing.rs` ในสไตล์ใหม่กับ `pricing/mod.rs` ในสไตล์เก่า ทั้งคู่ต้องมีโฟลเดอร์ `pricing/` สำหรับลูกของ
มันเองอยู่ดี) — ความแตกต่างระหว่างสองสไตล์จะเห็นชัดที่สุดตรง **module ที่ไม่มีลูกเลย** ซึ่งสไตล์ใหม่ไม่ต้องสร้าง
โฟลเดอร์ให้เปล่า ๆ เลย ประหยัดขั้นตอนไปได้มาก

**คำแนะนำเชิงปฏิบัติ**: สำหรับโปรเจกต์ใหม่ทุกโปรเจกต์ ให้ใช้สไตล์ปัจจุบัน (`<ชื่อ>.rs` + โฟลเดอร์ย่อยเมื่อมีลูก) เสมอ
— `cargo new` และเครื่องมือ tooling ทั้งหมดของ Rust ในปัจจุบันก็สร้างโครงสร้างแบบนี้ให้เป็น default อยู่แล้ว สไตล์เก่า
มีประโยชน์แค่ตอนที่ต้องอ่าน/แก้โค้ดเก่าที่มีอยู่แล้วเท่านั้น

### 16.8 ปรับความละเอียดของ Visibility: `pub(crate)` และ `pub(super)`

จนถึงตรงนี้เราเห็น visibility แค่ 2 ระดับ: private สนิท (ไม่มี `pub`) กับ public เต็มที่ (`pub` เปล่า ๆ ที่เปิดให้
ทุกคนที่เข้าถึง module นี้ได้เห็น รวมถึงคนนอก crate ถ้า crate นี้ถูกใช้เป็น library) แต่ในโลกจริงมีสถานการณ์ที่ต้องการ
ความละเอียดตรงกลางระหว่างสองขั้วนี้ — Rust มี syntax `pub(...)` ที่ระบุ **ขอบเขตที่แน่นอน** ว่าจะเปิดเผยให้ใครเห็น
ที่พบบ่อยที่สุด 2 แบบคือ:

- **`pub(crate)`**: มองเห็นได้จากทุกที่ **ภายใน crate เดียวกันนี้เท่านั้น** — ถ้า crate นี้ถูกคนอื่นเอาไปใช้เป็น
  library (dependency) ผู้ใช้จากภายนอก crate จะมองไม่เห็น item นี้เลย
- **`pub(super)`**: มองเห็นได้เฉพาะจาก **module พ่อของ module ปัจจุบันเท่านั้น** (แคบกว่า `pub(crate)` อีกขั้น)

```rust
mod inventory {
    pub struct Product {
        pub name: String,
        // pub(crate): มองเห็นได้จากทุกที่ "ภายใน crate เดียวกันนี้" เท่านั้น
        // ถ้ามีคนเอา crate นี้ไปใช้เป็น library (external consumer) จะมองไม่เห็น field นี้เลย
        // แม้ Product เองจะเป็น pub ก็ตาม
        pub(crate) internal_sku: String,
    }

    impl Product {
        pub fn new(name: &str, internal_sku: &str) -> Self {
            Product {
                name: name.to_string(),
                internal_sku: internal_sku.to_string(),
            }
        }
    }

    pub mod pricing {
        use super::Product;

        // pub(super): มองเห็นได้เฉพาะจาก module พ่อของ pricing (คือ inventory) เท่านั้น
        // แคบกว่า pub(crate) อีกขั้น — เหมาะกับฟังก์ชันช่วยภายในที่ไม่ต้องการให้ module อื่น
        // (แม้อยู่ crate เดียวกัน) มาเรียกใช้ตรง ๆ
        pub(super) fn raw_cost_cents(product: &Product) -> i64 {
            // เข้าถึง field แบบ pub(crate) ได้ตามปกติ เพราะ pricing อยู่ใน crate เดียวกันกับ inventory
            product.internal_sku.len() as i64 * 1000
        }

        pub fn quote_price(product: &Product) -> i64 {
            raw_cost_cents(product) * 130 / 100
        }
    }

    // debug_raw_cost อยู่ใน inventory ซึ่งเป็น "module พ่อ" ของ pricing โดยตรง
    // จึงเรียก pricing::raw_cost_cents (ที่เป็น pub(super)) ได้ ทั้งที่ main เรียกตรง ๆ ไม่ได้
    pub fn debug_raw_cost(product: &Product) -> i64 {
        pricing::raw_cost_cents(product)
    }
}

fn main() {
    let mouse = inventory::Product::new("เมาส์ไร้สาย", "MOU-01");

    println!("สินค้า: {}", mouse.name);
    println!("ราคาเสนอ: {} สตางค์", inventory::pricing::quote_price(&mouse));
    println!(
        "ต้นทุนดิบ (ผ่าน debug_raw_cost ใน inventory): {} สตางค์",
        inventory::debug_raw_cost(&mouse)
    );

    // pub(crate): main() อยู่ใน crate เดียวกันกับ inventory จึงยังมองเห็น internal_sku ได้
    println!("SKU ภายใน: {}", mouse.internal_sku);
}
```

ผลลัพธ์:

```
สินค้า: เมาส์ไร้สาย
ราคาเสนอ: 7800 สตางค์
ต้นทุนดิบ (ผ่าน debug_raw_cost ใน inventory): 6000 สตางค์
SKU ภายใน: MOU-01
```

ถ้าเราลองเรียก `inventory::pricing::raw_cost_cents(&mouse)` ตรง ๆ จาก `main()` (ซึ่งไม่ใช่ module พ่อของ `pricing`
— module พ่อของ `pricing` คือ `inventory` เท่านั้น) จะเจอ error ทันที แม้ `main()` จะอยู่ใน crate เดียวกันก็ตาม:

```
error[E0603]: function `raw_cost_cents` is private
  --> src/main.rs:51:25
   |
51 |     inventory::pricing::raw_cost_cents(&mouse);
   |                         ^^^^^^^^^^^^^^ private function
   |
note: the function `raw_cost_cents` is defined here
  --> src/main.rs:25:9
   |
25 |         pub(super) fn raw_cost_cents(product: &Product) -> i64 {
   |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

**เมื่อไหร่ควรใช้ `pub(crate)` จริงในโปรเจกต์จริง?** สถานการณ์คลาสสิกคือ: คุณกำลังเขียน crate ที่จะถูก publish เป็น
library ให้คนอื่นใช้ (เช่นผ่าน crates.io ซึ่งเราจะเรียนใน Part 17) และมีฟังก์ชัน/struct ที่เป็น "internal
implementation detail" ที่ต้องแบ่งปันกันระหว่างหลาย module **ภายใน** crate ของคุณเอง (เช่น helper สำหรับ validate
input, cache ภายใน, หรือ struct ที่ใช้แลกเปลี่ยนข้อมูลระหว่าง module) แต่ **ไม่ต้องการให้ผู้ใช้ library ของคุณเห็น
หรือพึ่งพามันเลย** — ถ้าใช้ `pub` เต็ม ๆ มันจะกลายเป็นส่วนหนึ่งของ public API ที่คุณต้อง "รับประกัน" ว่าจะคงที่ต่อไป
(ตามเหตุผลเรื่อง stable API จากหัวข้อ 16.3) แต่ `pub(crate)` ให้คุณแบ่งปันข้าม module ภายในได้เต็มที่ โดยยังคง
**อิสระในการเปลี่ยนแปลงมันได้ทุกเมื่อ** เพราะไม่มีใครนอก crate มองเห็นมันเลย

**เมื่อไหร่ควรใช้ `pub(super)`?** เมื่อ item นั้นเป็น "รายละเอียดการทำงานภายใน" ที่ต้องการให้ **เฉพาะ module พ่อของ
ตัวเองเท่านั้น** ใช้ได้ ไม่ต้องการให้ module อื่นแม้อยู่ crate เดียวกัน (เช่น module พี่น้อง หรือลูกของ module อื่น)
มาเรียกใช้ตรง ๆ — เหมาะกับ helper function ที่ผูกกับ logic เฉพาะของความสัมพันธ์พ่อ-ลูกคู่นั้นคู่เดียว ในตัวอย่างข้างบน
`raw_cost_cents` เป็นรายละเอียดการคำนวณภายในที่ `pricing` ต้องการให้ `inventory` (พ่อของมัน) เรียกดูได้เพื่อจุดประสงค์
เช่น debug หรือ logging แต่ไม่ต้องการให้ module อื่นที่ไม่เกี่ยวข้อง (สมมติมี `mod reports` อยู่ระดับเดียวกับ
`inventory`) มาเรียกใช้ตรง ๆ โดยไม่ผ่าน public API ที่ตั้งใจไว้ (`quote_price`)

พูดโดยสรุป ระดับ visibility ทั้งหมดของ Rust เรียงจากแคบสุดไปกว้างสุดได้ดังนี้:

| Visibility | มองเห็นได้จาก |
|---|---|
| (ไม่มี `pub`) | เฉพาะ module ปัจจุบันและ module ลูกของมันเท่านั้น |
| `pub(self)` | เหมือนไม่มี `pub` เลย (เขียนไว้เพื่อความชัดเจนเท่านั้น ใช้น้อยมากในทางปฏิบัติ) |
| `pub(super)` | module ปัจจุบัน + module พ่อโดยตรงของมัน |
| `pub(in path)` | เฉพาะ module ที่ระบุใน `path` และลูกของมัน (ใช้เมื่อต้องการระบุขอบเขตที่ไม่ใช่แค่พ่อตรง ๆ แต่ยังไม่กว้างเท่าทั้ง crate) |
| `pub(crate)` | ทุกที่ภายใน crate เดียวกันนี้ |
| `pub` | ทุกที่ ทั้งภายในและภายนอก crate (ถ้า crate นี้ถูกใช้เป็น library) |

### 16.9 Struct Field Privacy: บังคับใช้ Constructor/Getter แม้ Struct เป็น `pub`

หัวข้อนี้จะเจาะลึกสิ่งที่เราแค่แตะไว้ในหัวข้อ 16.3: **`pub struct` ไม่ได้ทำให้ field เป็น `pub` ไปด้วยโดยอัตโนมัติ**
— นี่ไม่ใช่แค่กฎที่ต้องจำ แต่เป็น **เครื่องมือออกแบบ (design tool) ที่ทรงพลังมาก** สำหรับการรักษา invariant (เงื่อนไข
ที่ต้องเป็นจริงตลอดเวลาของข้อมูล) ที่ต่อยอดโดยตรงจาก associated function `new(...)` ที่เราเรียนใน Part 9

ลองพิจารณา `BankAccount` ที่มี invariant สำคัญ: **ยอดเงินต้องไม่ติดลบ** ถ้า field `balance_cents` เป็น `pub` ธรรมดา
ใครก็ตั้งค่าให้ติดลบได้ตรง ๆ โดยไม่ผ่านการตรวจสอบใด ๆ เลย (`account.balance_cents = -999_999;` ก็ทำได้ ถ้า field
เป็น public) วิธีป้องกันคือทำให้ field เป็น **private** แม้ว่า struct เองจะเป็น `pub` ก็ตาม แล้วบังคับให้การสร้างและ
แก้ไขทุกอย่างต้องผ่าน method ที่ตรวจสอบเงื่อนไขให้เท่านั้น:

```rust
mod accounts {
    pub struct BankAccount {
        owner_name: String,
        balance_cents: i64, // ตั้งใจเก็บเป็น private เพื่อรักษา invariant: ยอดเงินต้องไม่ติดลบ
    }

    impl BankAccount {
        // constructor คือ "ประตูเดียว" ที่สร้าง BankAccount ได้ ควบคุมค่าเริ่มต้นให้ถูกต้องเสมอ
        pub fn new(owner_name: &str, initial_balance_cents: i64) -> Self {
            BankAccount {
                owner_name: owner_name.to_string(),
                balance_cents: initial_balance_cents.max(0), // บังคับไม่ให้ยอดเริ่มต้นติดลบ
            }
        }

        // getter แบบ &self คืนค่าแบบอ่านอย่างเดียว — ผู้เรียกดูค่าได้ แต่แก้ไขตรงไม่ได้
        pub fn owner_name(&self) -> &str {
            &self.owner_name
        }

        pub fn balance_cents(&self) -> i64 {
            self.balance_cents
        }

        // การแก้ไขยอดเงินทำได้ผ่าน method ที่ควบคุม logic เท่านั้น ไม่ใช่แก้ field ตรง ๆ
        pub fn deposit(&mut self, amount_cents: i64) {
            if amount_cents > 0 {
                self.balance_cents += amount_cents;
            }
        }

        pub fn try_withdraw(&mut self, amount_cents: i64) -> bool {
            if amount_cents > 0 && self.balance_cents >= amount_cents {
                self.balance_cents -= amount_cents;
                true
            } else {
                false
            }
        }
    }
}

fn main() {
    let mut acc = accounts::BankAccount::new("สมชาย", 100_000);

    println!(
        "{}: ยอดเงินเริ่มต้น {} สตางค์",
        acc.owner_name(),
        acc.balance_cents()
    );

    acc.deposit(50_000);
    let withdrew = acc.try_withdraw(30_000);
    println!(
        "ถอนสำเร็จ: {} | ยอดเงินคงเหลือ: {} สตางค์",
        withdrew,
        acc.balance_cents()
    );

    // ยังพยายามสร้างด้วยยอดติดลบผ่าน constructor เพื่อยืนยันว่า invariant ถูกบังคับใช้จริง
    let protected = accounts::BankAccount::new("มานี", -50_000);
    println!(
        "{}: ยอดเงินถูกบังคับให้ไม่ต่ำกว่า 0 -> {} สตางค์",
        protected.owner_name(),
        protected.balance_cents()
    );
}
```

ผลลัพธ์:

```
สมชาย: ยอดเงินเริ่มต้น 100000 สตางค์
ถอนสำเร็จ: true | ยอดเงินคงเหลือ: 120000 สตางค์
มานี: ยอดเงินถูกบังคับให้ไม่ต่ำกว่า 0 -> 0 สตางค์
```

สังเกตว่า `main()` **ไม่มีทางเข้าถึง `acc.owner_name` หรือ `acc.balance_cents` แบบ field ตรง ๆ ได้เลย** (`acc.balance_cents`
ในฐานะชื่อ field จะให้ error ทันทีถ้าเขียนแบบนั้น) ต้องผ่าน method `owner_name()` และ `balance_cents()` เท่านั้น —
ถ้าลองสร้างด้วยยอดเริ่มต้นติดลบ (`BankAccount::new("มานี", -50_000)`) constructor จะ **บังคับ** ให้กลายเป็น `0` ทันที
ผ่าน `.max(0)` แทนที่จะยอมให้ค่าติดลบหลุดเข้าไปในระบบ — นี่คือ invariant ที่ **การันตีได้ 100% ในทุกจุดของโปรแกรม**
เพราะไม่มีทางใดเลยที่จะสร้างหรือแก้ไข `BankAccount` โดยไม่ผ่าน method ที่ตรวจสอบเงื่อนไขนี้

ลองดูว่าถ้าพยายามอ่าน field ตรง ๆ จากนอก module (แม้ว่า `BankAccount` เป็น `pub` และสร้าง instance ผ่าน `new()`
ที่เป็น `pub` สำเร็จแล้วก็ตาม) จะเกิดอะไรขึ้น:

```rust
mod accounts {
    pub struct BankAccount {
        owner_name: String,
        balance_cents: i64,
    }

    impl BankAccount {
        pub fn new(owner_name: &str, initial_balance_cents: i64) -> Self {
            BankAccount {
                owner_name: owner_name.to_string(),
                balance_cents: initial_balance_cents,
            }
        }
    }
}

fn main() {
    let acc = accounts::BankAccount::new("สมชาย", 100_000);
    // สร้าง instance ผ่าน constructor ที่เป็น pub ได้สำเร็จ แต่ field ยังเป็น private
    // การอ่าน field ตรง ๆ จากนอก module จึงยัง compile ไม่ผ่าน
    println!("{}", acc.owner_name);
}
```

```
error[E0616]: field `owner_name` of struct `BankAccount` is private
  --> src/main.rs:21:24
   |
21 |     println!("{}", acc.owner_name);
   |                        ^^^^^^^^^^ private field
```

**นี่คือรูปแบบการออกแบบที่สำคัญที่สุดรูปแบบหนึ่งในภาษา Rust**: ใช้ **module boundary** (ขอบเขตของ module) ร่วมกับ
**field privacy** เพื่อบังคับให้ทุกการสร้างและแก้ไขข้อมูลต้องผ่าน "จุดตรวจ" ที่ผู้เขียน struct กำหนดไว้เท่านั้น — เรา
เคยเห็น associated function `new(...)` เป็น constructor มาแล้วใน Part 9 แต่ตอนนั้น `Product` ทุก field เป็น `pub`
หมด constructor เป็นแค่ **ความสะดวก** (ไม่ต้องเขียน struct literal ยาว ๆ ทุกครั้ง) แต่ยังมีช่องโหว่ที่ใครก็เข้าถึง
field ตรง ๆ ได้อยู่ดี บทนี้ทำให้ constructor กลายเป็น **ข้อบังคับที่ compiler ยืนยันให้จริง (compiler-enforced
guarantee)** ไม่ใช่แค่ความสะดวกอีกต่อไป — field ที่เป็น private ทำให้ "ประตู" เข้าออกข้อมูลมีอยู่ทางเดียวคือผ่าน
method ที่ `impl` กำหนดไว้เท่านั้น

**คำแนะนำเชิงปฏิบัติ**: เมื่อออกแบบ struct ที่จะเป็น `pub` (เผื่อว่าจะถูกใช้จากนอก module หรือนอก crate) ให้ตั้ง
คำถามกับทุก field ว่า **"ถ้าใครแก้ field นี้ตรง ๆ โดยไม่ผ่าน method ใด ๆ จะทำให้ข้อมูลอยู่ในสถานะที่ไม่สมเหตุสมผลได้
หรือไม่?"** ถ้าคำตอบคือได้ (เช่นยอดเงินติดลบ, สต็อกติดลบ, สถานะขัดแย้งกันเอง) ควรเก็บ field นั้นเป็น private แล้วเปิด
เฉพาะ method ที่ควบคุม logic การเปลี่ยนแปลงเท่านั้น ถ้าคำตอบคือไม่ได้จริง ๆ (เช่น field ที่เป็นแค่ label แสดงผล
ไม่มีเงื่อนไขอะไรเลย) การเปิด field เป็น `pub` ตรง ๆ ก็ไม่ใช่เรื่องผิดอะไร — ไม่จำเป็นต้องเขียน getter/setter ให้กับ
ทุก field เสมอไปถ้าไม่มี invariant ให้ปกป้องจริง ๆ

### 16.10 ตัวอย่างจริง: Refactor ระบบร้านค้าจากไฟล์เดียวยุ่งเหยิงสู่ Module Tree ที่จัดระเบียบดี

ตอนนี้เรามีทุกเครื่องมือที่จำเป็นแล้ว มาดูภาพรวมทั้งหมดโดยนำระบบร้านค้าจากหัวข้อ 16.1 (ไฟล์เดียวยุ่งเหยิง) มา
**refactor** ให้เป็น module tree ที่จัดระเบียบดี พร้อมใช้ `pub(crate)`, field privacy, และ `pub use` ร่วมกันแบบ
ครบวงจร

**ก่อน (before)**: ทุกอย่างอยู่ใน `src/main.rs` ไฟล์เดียว (ตัดบางส่วนที่ไม่เกี่ยวข้องออกเพื่อโฟกัสที่โครงสร้าง เทียบกับ
หัวข้อ 16.1) — `Product` เก็บ field เป็น `pub` ทุกตัว ทำให้ `reduce_stock` แก้ `quantity` ตรง ๆ ได้จากที่ไหนก็ได้
โดยไม่ผ่านการตรวจสอบใด ๆ, ทุกฟังก์ชันปนกันอยู่ในไฟล์เดียวไม่มีการจัดกลุ่ม:

```rust
// src/main.rs (ก่อน refactor — ทุกอย่างปนกันในไฟล์เดียว)
struct Product {
    name: String,
    base_price_cents: i64,
    quantity: u32, // เป็น field ธรรมดาที่ใครก็แก้ตรง ๆ ได้ ไม่มีการป้องกัน
}

enum OrderStatus {
    Pending,
    Fulfilled,
    Rejected,
}

struct Order {
    id: u32,
    quantity: u32,
    status: OrderStatus,
}

fn apply_discount(total_cents: i64, discount_percent: f64) -> i64 {
    let discount_amount = (total_cents as f64) * (discount_percent / 100.0);
    total_cents - discount_amount as i64
}

fn fulfill_order(order: &mut Order, product: &mut Product) {
    // ไม่มีอะไรห้ามโค้ดส่วนอื่นแก้ product.quantity ตรง ๆ แบบข้าม logic นี้ไปเลย
    if product.quantity >= order.quantity {
        product.quantity -= order.quantity;
        order.status = OrderStatus::Fulfilled;
    } else {
        order.status = OrderStatus::Rejected;
    }
}

fn main() {
    let mut mouse = Product {
        name: String::from("เมาส์ไร้สาย"),
        base_price_cents: 29_900,
        quantity: 20,
    };
    let mut order = Order {
        id: 1001,
        quantity: 3,
        status: OrderStatus::Pending,
    };

    fulfill_order(&mut order, &mut mouse);
    // ... ส่วนที่เหลือเหมือนหัวข้อ 16.1
}
```

โครงสร้างนี้มีปัญหาตรงตามที่วิเคราะห์ไว้ในหัวข้อ 16.1 ทุกข้อ บวกกับปัญหาเรื่อง encapsulation จากหัวข้อ 16.9: field
`quantity` ของ `Product` เป็น field ธรรมดา ไม่มีอะไรบังคับว่าการลดสต็อกต้องผ่านการตรวจสอบว่ามีของพอหรือไม่ ถ้ามีโค้ด
ส่วนอื่นในไฟล์เขียน `mouse.quantity = 99999;` ตรง ๆ ที่ไหนก็ได้ ก็ไม่มีอะไรจับได้เลย

**หลัง (after)**: แยกออกเป็น 4 ไฟล์ตามความรับผิดชอบ ใช้สไตล์การแยกไฟล์ปัจจุบันจากหัวข้อ 16.6-16.7:

```
project/
└── src/
    ├── main.rs           <- เปลือกโปรแกรม ประกอบทุก module เข้าด้วยกัน
    ├── inventory.rs       <- Product + การจัดการสต็อกภายใน
    ├── inventory/
    │   └── pricing.rs     <- การคำนวณส่วนลด/ราคา แยกจาก Product โดยตรง
    ├── orders.rs           <- Order, OrderStatus, และ logic การ fulfill ออเดอร์
    └── catalog.rs           <- "หน้าร้าน" รวม public API ที่สะอาด ด้วย pub use
```

```rust
// ไฟล์: src/inventory.rs
pub mod pricing;

pub struct Product {
    name: String,
    base_price_cents: i64,
    quantity: u32, // private: ป้องกันไม่ให้ใครแก้สต็อกตรง ๆ แบบไม่ผ่านการตรวจสอบ
}

impl Product {
    pub fn new(name: &str, base_price_cents: i64, quantity: u32) -> Self {
        Product {
            name: name.to_string(),
            base_price_cents,
            quantity,
        }
    }

    pub fn name(&self) -> &str {
        &self.name
    }

    pub fn base_price_cents(&self) -> i64 {
        self.base_price_cents
    }

    pub fn quantity(&self) -> u32 {
        self.quantity
    }

    // pub(crate): เปิดให้ module อื่นภายใน crate นี้ (เช่น orders) เรียกได้ เพราะเป็น operation
    // ที่ต้องทำงานร่วมกับ Order ตอน fulfill แต่ไม่ต้องการเปิดเผยออกไปสู่ผู้ใช้ crate ภายนอก
    // (ผู้ใช้ภายนอกควรแก้สต็อกผ่าน public API ระดับสูงกว่านี้เท่านั้น เช่นผ่าน Order)
    pub(crate) fn reduce_stock(&mut self, amount: u32) -> bool {
        if self.quantity < amount {
            return false;
        }
        self.quantity -= amount;
        true
    }
}
```

```rust
// ไฟล์: src/inventory/pricing.rs
pub fn apply_discount(total_cents: i64, discount_percent: f64) -> i64 {
    let discount_amount = (total_cents as f64) * (discount_percent / 100.0);
    total_cents - discount_amount as i64
}
```

```rust
// ไฟล์: src/orders.rs
use crate::inventory::Product;

pub enum OrderStatus {
    Pending,
    Fulfilled,
    Rejected,
}

pub struct Order {
    id: u32,
    quantity: u32,
    status: OrderStatus,
}

impl Order {
    pub fn new(id: u32, quantity: u32) -> Self {
        Order {
            id,
            quantity,
            status: OrderStatus::Pending,
        }
    }

    pub fn id(&self) -> u32 {
        self.id
    }

    pub fn status(&self) -> &OrderStatus {
        &self.status
    }

    // orders เป็น module คนละสายกับ inventory (ทั้งคู่เป็นลูกของ crate root เหมือนกัน ไม่ใช่พ่อ-ลูกกัน)
    // จึงเรียก product.reduce_stock() (ที่เป็น pub(crate)) ได้ เพราะ pub(crate) มองเห็นได้ทั่ว crate
    // แต่จะเรียกไม่ได้เลยถ้า reduce_stock เป็น private เฉย ๆ (ไม่มี pub เลย)
    pub fn fulfill(&mut self, product: &mut Product) {
        if product.reduce_stock(self.quantity) {
            self.status = OrderStatus::Fulfilled;
        } else {
            self.status = OrderStatus::Rejected;
        }
    }
}
```

```rust
// ไฟล์: src/catalog.rs
// "หน้าร้าน" ของทั้งระบบ: รวม public API ที่ผู้ใช้ทั่วไปต้องการทั้งหมดไว้ที่เดียว
// ผู้ใช้ import จาก catalog:: อย่างเดียวพอ ไม่ต้องรู้เลยว่าภายในแยกเป็น inventory/orders/pricing
// กี่ไฟล์ — นี่คือประโยชน์ของ pub use ที่อธิบายไว้ในหัวข้อ 16.4
pub use crate::inventory::pricing::apply_discount;
pub use crate::inventory::Product;
pub use crate::orders::{Order, OrderStatus};
```

```rust
// ไฟล์: src/main.rs
mod inventory;
mod orders;
mod catalog;

use catalog::{apply_discount, Order, OrderStatus, Product};

fn main() {
    let mut mouse = Product::new("เมาส์ไร้สาย", 29_900, 20);
    let mut order = Order::new(1001, 3);

    order.fulfill(&mut mouse);

    match order.status() {
        OrderStatus::Fulfilled => println!(
            "ออเดอร์ #{} สำเร็จ เหลือสต็อก {} ชิ้น",
            order.id(),
            mouse.quantity()
        ),
        OrderStatus::Rejected => println!("ออเดอร์ #{} ถูกปฏิเสธ สต็อกไม่พอ", order.id()),
        OrderStatus::Pending => println!("ออเดอร์ #{} รอดำเนินการ", order.id()),
    }

    let discounted = apply_discount(mouse.base_price_cents(), 10.0);
    println!(
        "ราคาสินค้า {} หลังหักส่วนลด 10%: {} สตางค์",
        mouse.name(),
        discounted
    );
}
```

ผลลัพธ์:

```
ออเดอร์ #1001 สำเร็จ เหลือสต็อก 17 ชิ้น
ราคาสินค้า เมาส์ไร้สาย หลังหักส่วนลด 10%: 26910 สตางค์
```

มาสรุปสิ่งที่ดีขึ้นอย่างเป็นรูปธรรมจากการ refactor นี้ เทียบกับปัญหาที่ระบุไว้ในหัวข้อ 16.1:

1. **จัดกลุ่มตามความรับผิดชอบชัดเจน** — `inventory.rs` รับผิดชอบเรื่องสินค้าและสต็อกเท่านั้น, `inventory/pricing.rs`
   รับผิดชอบเรื่องการคำนวณราคา/ส่วนลดเท่านั้น, `orders.rs` รับผิดชอบเรื่องออเดอร์และ workflow การ fulfill เท่านั้น
   ใครเปิดไฟล์ไหนก็รู้ทันทีว่าไฟล์นั้น "เป็นเรื่องของ" อะไร โดยไม่ต้องอ่านทั้งไฟล์
2. **encapsulation ที่แท้จริง** — field `quantity` ของ `Product` เป็น private สนิท การลดสต็อกทำได้ทางเดียวคือผ่าน
   `reduce_stock()` ที่ตรวจสอบว่ามีของพอก่อนเสมอ ไม่มีทางใดเลยที่โค้ดจากที่อื่นจะทำให้ `quantity` ติดลบหรือผิดพลาด
   แบบข้าม logic การตรวจสอบนี้ได้
3. **ใช้ `pub(crate)` แบ่งปันอย่างพอดี** — `reduce_stock()` เปิดให้ `orders.rs` (ซึ่งอยู่ crate เดียวกัน) เรียกใช้ได้
   เพราะการ fulfill ออเดอร์จำเป็นต้องแก้สต็อกจริง แต่ไม่เปิดเผยรายละเอียดนี้ออกไปสู่ผู้ใช้ crate จากภายนอกเลย (ถ้า
   crate นี้ถูก publish เป็น library ผู้ใช้จะเห็นแค่ `Order::fulfill()` เป็น API ระดับสูงที่ใช้งานง่าย ไม่เห็น
   `reduce_stock` เลย)
4. **`catalog` เป็นหน้าร้านที่สะอาด** — ผู้ใช้ทั่วไปเรียก `use catalog::{Product, Order, ...}` เพียงจุดเดียว โดยไม่
   จำเป็นต้องรู้เลยว่าภายในแยก module เป็น `inventory`, `inventory::pricing`, `orders` กี่ไฟล์ — ถ้าวันหนึ่งเราตัดสินใจ
   จัดโครงสร้างภายในใหม่ (เช่นย้าย `pricing` ไปเป็นลูกของ `orders` แทน) เราแค่แก้ path ข้างใน `catalog.rs` ที่จุดเดียว
   โดยไม่กระทบโค้ดที่เรียกผ่าน `catalog::` เลยแม้แต่บรรทัดเดียว — นี่คือ **stable API กับ implementation ที่เปลี่ยน
   ได้อย่างอิสระ** ตามเหตุผลที่อธิบายไว้ในหัวข้อ 16.3 อย่างเป็นรูปธรรม
5. **ทดสอบเป็นส่วน ๆ ได้ง่ายขึ้นมาก** — ถ้าต้องการทดสอบ `apply_discount` เพียงอย่างเดียว ก็เปิดแค่ไฟล์
   `inventory/pricing.rs` โดยไม่ต้องยุ่งกับ `Order` หรือ `Product` เลย (เราจะเรียนการเขียน automated test สำหรับ
   module แต่ละตัวอย่างเป็นทางการใน Part 32)

นี่คือภาพรวมที่แสดงให้เห็นว่าเนื้อหาทั้งบทนี้ — `mod`, privacy, path, `use`/`pub use`, `super`, การแยกไฟล์,
`pub(crate)`/`pub(super)`, และ field privacy — ทำงานร่วมกันเป็นระบบเดียวเพื่อเปลี่ยนโค้ดที่ "ใช้งานได้" ให้กลายเป็น
โค้ดที่ "ดูแลได้ในระยะยาว" ซึ่งเป็นทักษะที่จำเป็นมากขึ้นเรื่อย ๆ เมื่อโปรแกรมที่คุณเขียนเริ่มโตเกินกว่าไฟล์เดียวจะรับ
ไหว

## กับดักที่พบบ่อย (Common Pitfalls)

**1. ลืมใส่ `pub` แล้วเจอ error privacy หลายชั้นพร้อมกัน**

```rust
mod inventory {
    struct Product {
        name: String,
    }

    impl Product {
        fn new(name: &str) -> Self {
            Product { name: name.to_string() }
        }
    }
}

fn main() {
    let p = inventory::Product::new("เมาส์ไร้สาย");
    println!("{}", p.name);
}
```

```
error[E0603]: struct `Product` is private
  --> src/main.rs:14:24
   |
14 |     let p = inventory::Product::new("เมาส์ไร้สาย");
   |                        ^^^^^^^ private struct

error[E0624]: associated function `new` is private
  --> src/main.rs:14:33
   |
14 |     let p = inventory::Product::new("เมาส์ไร้สาย");
   |                                 ^^^ private associated function

error[E0616]: field `name` of struct `inventory::Product` is private
  --> src/main.rs:15:22
   |
15 |     println!("{}", p.name);
   |                      ^^^^ private field
```

**วิธีแก้**: เติม `pub` ให้ครบทุกชั้นที่ต้องการเปิดเผย (`pub struct`, `pub fn new`, `pub name: String`) — สังเกตว่า
error มาพร้อมกันหลายจุด เพราะ privacy เป็นกฎที่ตรวจสอบ **ทุกจุดที่เข้าถึง** แยกกัน ไม่ใช่แค่จุดแรกที่เจอ (ดูรายละเอียด
เต็มในหัวข้อ 16.3)

**2. คิดว่า `pub struct` ทำให้ field เป็น `pub` ไปด้วย**

```rust
mod accounts {
    pub struct BankAccount {
        owner_name: String, // ลืมใส่ pub — คิดว่า pub struct ครอบคลุมถึง field แล้ว
        balance_cents: i64,
    }

    impl BankAccount {
        pub fn new(owner_name: &str, initial_balance_cents: i64) -> Self {
            BankAccount {
                owner_name: owner_name.to_string(),
                balance_cents: initial_balance_cents,
            }
        }
    }
}

fn main() {
    let acc = accounts::BankAccount::new("สมชาย", 100_000);
    println!("{}", acc.owner_name); // ตั้งใจอ่าน field ตรง ๆ โดยไม่มี getter
}
```

```
error[E0616]: field `owner_name` of struct `BankAccount` is private
  --> src/main.rs:19:24
   |
19 |     println!("{}", acc.owner_name); // ตั้งใจอ่าน field ตรง ๆ โดยไม่มี getter
   |                        ^^^^^^^^^^ private field
```

**วิธีแก้**: `pub` ของ struct และ `pub` ของแต่ละ field เป็นการตัดสินใจที่**แยกจากกันอย่างสิ้นเชิง** ถ้าต้องการให้
อ่าน field ได้จากภายนอก ต้องเติม `pub` ให้ field นั้นโดยตรง หรือ (ซึ่งเป็นแนวทางที่แนะนำกว่าเมื่อมี invariant ต้อง
รักษา) เขียน getter method อย่าง `pub fn owner_name(&self) -> &str { &self.owner_name }` แทน (ดูรายละเอียดเต็ม
ในหัวข้อ 16.9)

**3. เข้าใจผิดว่า `pub(super)` หมายถึง "เปิดให้ทุกที่ใน crate เห็น"**

```rust
mod inventory {
    pub mod pricing {
        pub(super) fn raw_cost_cents(base: i64) -> i64 {
            base + 1000
        }
    }
}

fn main() {
    // main() ไม่ใช่ module พ่อของ pricing (พ่อของ pricing คือ inventory) จึงเรียกไม่ได้
    println!("{}", inventory::pricing::raw_cost_cents(29_900));
}
```

```
error[E0603]: function `raw_cost_cents` is private
  --> src/main.rs:11:40
   |
11 |     println!("{}", inventory::pricing::raw_cost_cents(29_900));
   |                                        ^^^^^^^^^^^^^^ private function
   |
note: the function `raw_cost_cents` is defined here
  --> src/main.rs:3:9
   |
 3 |         pub(super) fn raw_cost_cents(base: i64) -> i64 {
   |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

**วิธีแก้**: `pub(super)` เปิดให้เห็นเฉพาะ **module พ่อโดยตรง** เท่านั้น ไม่ใช่ทุกที่ใน crate (นั่นคือความหมายของ
`pub(crate)` ต่างหาก) ถ้าต้องการให้ `main()` (ซึ่งอยู่ที่ crate root) เรียกได้ ต้องเปลี่ยนเป็น `pub` เต็ม ๆ หรือ
`pub(crate)` แทน หรือถ้าต้องการรักษาความหมายเดิมไว้ (เฉพาะ `inventory` เรียกได้) ให้เรียกผ่านฟังก์ชันตัวกลางใน
`inventory` ที่เป็น `pub` แทน (ดูรายละเอียดเต็มในหัวข้อ 16.8)

**4. ประกาศ `mod ชื่อ;` แต่ไม่ได้สร้างไฟล์ให้ตรงตามที่ compiler คาดหวัง**

```rust
// src/main.rs
mod inventory;

fn main() {
    println!("hello");
}
```

ถ้าไม่มีทั้ง `src/inventory.rs` และ `src/inventory/mod.rs` อยู่จริง:

```
error[E0583]: file not found for module `inventory`
 --> src/main.rs:2:1
  |
2 | mod inventory;
  | ^^^^^^^^^^^^^^
  |
  = help: to create the module `inventory`, create file "inventory.rs" or "inventory/mod.rs"
  = note: if there is a `mod inventory` elsewhere in the crate already, import it with `use crate::...` instead
```

**วิธีแก้**: `mod ชื่อ;` (มี `;` ไม่มี `{ }`) เป็นการบอก compiler ว่า "ไปหาไฟล์เอาเอง" — ต้องสร้างไฟล์ `src/inventory.rs`
(สไตล์ปัจจุบัน แนะนำ) หรือ `src/inventory/mod.rs` (สไตล์เก่า) ให้ตรงกับชื่อ module จริง ๆ ก่อน มิฉะนั้น compiler จะ
หาไฟล์ไม่เจอตั้งแต่ขั้น parse โปรเจกต์เลย (ดูรายละเอียดเต็มในหัวข้อ 16.6) — สังเกตว่า error message ของ Rust ในกรณีนี้
บอกวิธีแก้ตรงจุดให้เลยทั้งสองสไตล์ที่เป็นไปได้

**5. พิมพ์ชื่อ path ผิดตอน `use` แล้วเจอ unresolved import**

```rust
mod inventory {
    pub struct Product {
        pub name: String,
    }
}

use inventory::Prodcut; // พิมพ์ผิด: Prodcut ไม่ใช่ Product

fn main() {
    let p = Prodcut { name: String::from("เมาส์ไร้สาย") };
    println!("{}", p.name);
}
```

```
error[E0432]: unresolved import `inventory::Prodcut`
 --> src/main.rs:7:5
  |
7 | use inventory::Prodcut; // พิมพ์ผิด: Prodcut ไม่ใช่ Product
  |     ^^^^^^^^^^^-------
  |     |          |
  |     |          help: a similar name exists in the module: `Product`
  |     no `Prodcut` in `inventory`
```

**วิธีแก้**: ตรวจตัวสะกดชื่อ item ให้ตรงกับที่นิยามไว้จริง — ข้อดีอย่างหนึ่งของ compiler ของ Rust คือมันมักจะ**เดา
ชื่อที่ใกล้เคียงที่สุด**ในโมดูลนั้นมาแนะนำให้เสมอ (`help: a similar name exists in the module: Product`) ทำให้แก้
ได้เร็วโดยไม่ต้องไปงมหาเองว่าพิมพ์ผิดตรงไหน — error แบบเดียวกันนี้ก็เกิดขึ้นได้ถ้า item นั้นมีชื่อถูกแล้วแต่**ไม่ได้
เป็น `pub`** ซึ่งในกรณีนั้น error message จะบอกว่า item นั้น "is private" แทนที่จะบอกว่าไม่มีชื่อนี้อยู่เลย

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** เขียน module ชื่อ `shipping` แบบ inline (ยังไม่ต้องแยกไฟล์) ที่มี `pub struct ShippingOption` เก็บ
   field `carrier_name: String` และ `fee_cents: i64` (ทั้งสอง field เป็น `pub`) พร้อม associated function
   `pub fn new(carrier_name: &str, fee_cents: i64) -> Self` จากนั้นเขียน `mod` ย่อยชื่อ `rules` ซ้อนอยู่ใน
   `shipping` ที่มีฟังก์ชัน `pub fn is_free_shipping(order_total_cents: i64) -> bool` (คืน `true` ถ้ายอดออเดอร์
   เกิน 100,000 สตางค์) แล้วเขียน `main()` เรียกใช้ทั้งสองผ่าน path เต็ม `shipping::ShippingOption::new(...)` และ
   `shipping::rules::is_free_shipping(...)`
   (hint: โครงสร้างเหมือนตัวอย่างหัวข้อ 16.2 ทุกประการ แค่เปลี่ยนชื่อ struct/module/field ให้ตรงกับโจทย์)

2. **[กลาง]** ลอง comment `pub` ออกจาก field `fee_cents` ในแบบฝึกหัดข้อ 1 (ให้เหลือ `struct` เป็น `pub` แต่ field
   นี้ private) แล้วเขียนโค้ดที่พยายามอ่าน `option.fee_cents` ตรง ๆ จาก `main()` บันทึก error message ที่ได้ (ควร
   เจอ E0616) จากนั้นแก้ด้วยการเพิ่ม getter method `pub fn fee_cents(&self) -> i64 { self.fee_cents }` ใน `impl`
   แทน แล้วยืนยันว่า compile ผ่านอีกครั้ง อธิบายด้วยคำพูดของตัวเองว่าทำไมการทำแบบนี้ (private field + getter) ดีกว่า
   การเปิด field เป็น `pub` ตรง ๆ ในกรณีที่ field นั้นมีเงื่อนไข (invariant) ที่ต้องรักษา
   (hint: ดูตัวอย่าง `BankAccount` ในหัวข้อ 16.9 เป็นต้นแบบ — ลองนึกว่า `fee_cents` ไม่ควรติดลบได้เช่นเดียวกัน)

3. **[กลาง-ยาก]** สร้างโปรเจกต์ Rust จริงด้วย `cargo new shipping_demo` แล้วแยก module `shipping` จากข้อ 1-2 ออกจาก
   `src/main.rs` ไปเป็นไฟล์ `src/shipping.rs` (สไตล์ปัจจุบัน) และแยก module ย่อย `rules` ออกไปเป็นไฟล์
   `src/shipping/rules.rs` ตามกฎที่เรียนในหัวข้อ 16.6-16.7 รัน `cargo run` เพื่อยืนยันว่าโปรแกรมยังทำงานถูกต้อง
   เหมือนก่อนแยกไฟล์ทุกประการ จากนั้นลองเปลี่ยน `pub mod rules;` ใน `shipping.rs` ให้เป็น `pub(crate) mod rules;`
   แทน แล้วสังเกตว่า public API ที่มองเห็นได้จากภายนอก crate เปลี่ยนไปอย่างไร (เขียนคำอธิบายสั้น ๆ ว่า `rules`
   จะยังใช้ได้จาก `main.rs` หรือไม่ เพราะเหตุใด)
   (hint: `main.rs` อยู่ crate เดียวกันกับ `shipping` เสมอ ดังนั้น `pub(crate) mod rules;` จะยัง**ไม่**ทำให้
   `main.rs` เรียก `shipping::rules::is_free_shipping` ไม่ได้ — แต่ถ้าเผยแพร่ crate นี้เป็น library ให้คนอื่นใช้
   ผู้ใช้ภายนอกจะมองไม่เห็น `rules` เลย)

4. **[ยาก/ประยุกต์]** ขยายตัวอย่าง refactor ในหัวข้อ 16.10 ให้สมบูรณ์ยิ่งขึ้น โดยเพิ่ม module ใหม่ชื่อ `customers`
   (แยกไฟล์ `src/customers.rs`) ที่มี `pub struct Customer` เก็บ field แบบ private ทั้งหมด (`name: String`,
   `loyalty_points: u32`) พร้อม constructor `pub fn new(name: &str) -> Self`, getter `pub fn name(&self) -> &str`
   และ `pub fn loyalty_points(&self) -> u32`, และ method `pub(crate) fn add_points(&mut self, points: u32)` ที่
   เพิ่มแต้มสะสม (ตั้งใจให้เป็น `pub(crate)` เพราะเฉพาะ module ภายในระบบ เช่น `orders`, ควรเป็นคนเพิ่มแต้มให้ ไม่ใช่
   ผู้ใช้ภายนอกเรียกเพิ่มเองตรง ๆ) จากนั้นแก้ `Order::fulfill` ใน `orders.rs` ให้รับ `customer: &mut Customer`
   เพิ่มอีกหนึ่ง parameter และเรียก `customer.add_points(...)` (เช่น 1 แต้มต่อ 1,000 สตางค์ที่ซื้อ) เมื่อออเดอร์
   สำเร็จเท่านั้น สุดท้ายเพิ่ม `pub use crate::customers::Customer;` ใน `catalog.rs` แล้วปรับ `main()` ให้ประกาศ
   ลูกค้าและพิมพ์แต้มสะสมหลังสั่งซื้อสำเร็จ
   (hint: `orders.rs` และ `customers.rs` เป็น module พี่น้องกัน (sibling) ทั้งคู่เป็นลูกของ crate root เหมือน
   `inventory` — `pub(crate)` ทำงานข้าม sibling module ได้เพราะมันมองเห็นได้ "ทั่ว crate" ไม่ใช่แค่ในสาย
   ancestor-descendant เดียวกันแบบ `pub(super)`)

## สรุป

บทนี้คือจุดเปลี่ยนสำคัญจากการเรียน "ไวยากรณ์ของภาษา" (ที่เราสะสมมาตั้งแต่ Part 6-15) ไปสู่การเรียน **"วิธีจัดโครงสร้าง
โปรแกรมขนาดใหญ่"** — เราเริ่มจากปัญหาจริงของการเขียนทุกอย่างไว้ใน `main.rs` ไฟล์เดียว (ไม่มีการจัดกลุ่ม, ไม่มีขอบเขต
การเข้าถึง, สเกลไม่ได้, ชื่อชนกันง่าย) แล้วแก้ด้วยระบบ **module** ที่ Rust ออกแบบมาอย่างเป็นระบบ

เราเรียนรู้ **`mod`** สำหรับสร้าง module และ **module tree** ที่มี `crate` เป็นราก, กฎ **privacy** ที่ทุกอย่างเป็น
private โดย default อย่างจงใจ (เพื่อ encapsulation และ stable API ที่แท้จริง ต่างจากภาษาที่เปิดทุกอย่างเป็น default
อย่าง Python), **path** ทั้งแบบ absolute (`crate::`) และ relative (`self::`, `super::`) พร้อม **`use`**,
**`use ... as`**, และ **`pub use`** สำหรับ re-export สร้าง public API ที่สะอาด, การ **แยก module ออกเป็นไฟล์แยก**
ทั้งสไตล์ปัจจุบัน (`inventory.rs`) และสไตล์เก่า (`inventory/mod.rs`) พร้อม module tree แบบหลายไฟล์หลายระดับ, การปรับ
ความละเอียดของ visibility ด้วย **`pub(crate)`** และ **`pub(super)`** สำหรับสร้าง internal API ที่แบ่งปันกันได้บาง
ส่วนโดยไม่เปิดเผยทั้งหมด, และ **field privacy** ที่ทำให้ `pub struct` ยังคงบังคับให้ผู้ใช้ผ่าน constructor/getter
ได้ ต่อยอดจาก associated function `new(...)` ที่เรียนใน Part 9 ให้กลายเป็นข้อบังคับที่ compiler การันตีให้จริง ไม่ใช่
แค่ความสะดวกอีกต่อไป

ตัวอย่าง refactor ท้ายบทแสดงให้เห็นว่าเครื่องมือทั้งหมดนี้ทำงานร่วมกันอย่างไรในระบบจริง: แยกความรับผิดชอบเป็น
`inventory`, `inventory::pricing`, `orders` ตามหน้าที่ชัดเจน, ป้องกัน invariant ของสต็อกด้วย field privacy บวก
`pub(crate)`, และรวม public API ที่สะอาดผ่าน `catalog` ด้วย `pub use` — ทั้งหมดนี้คือทักษะพื้นฐานที่จำเป็นสำหรับ
การเขียนโปรแกรม Rust ขนาดใหญ่ทุกโปรแกรมในโลกจริง

ใน **Part 17** เราจะขยับออกจากระดับ "โครงสร้างภายในไฟล์ Rust" ไปสู่ระดับ "โครงสร้างของโปรเจกต์ทั้งก้อน" — เรียนรู้จัก
**Package, Crate, และ Workspace** ความแตกต่างระหว่าง binary crate กับ library crate, วิธีจัดการ dependency จาก
crates.io, และวิธีรวมหลาย crate ที่เกี่ยวข้องกันไว้ใน workspace เดียว ซึ่งเป็นขั้นต่อไปที่จำเป็นก่อนจะเรียน Generics
(Part 18) และ Traits (Part 19) ที่จะทำให้ระบบ module ที่เราออกแบบในบทนี้ยืดหยุ่นและนำกลับมาใช้ซ้ำได้มากยิ่งขึ้นไปอีก

---

**Part ก่อนหน้า:** [Collections: HashMap, HashSet, BTreeMap/BTreeSet](part-015-hashmap-and-sets.md) | **Part ถัดไป:** [Packages, Crates, Workspaces](part-017-packages-crates-workspaces.md)
