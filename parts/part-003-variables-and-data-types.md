# Part 3: ตัวแปร ชนิดข้อมูลพื้นฐาน และ Mutability

> โมดูล: เริ่มต้นใช้งาน Rust (Getting Started) | ระดับ: พื้นฐาน | เวลาโดยประมาณ: 120 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- ประกาศตัวแปรด้วย `let` และอธิบายได้ว่าทำไม Rust จึงเลือกให้ตัวแปร **immutable โดย default** ต่างจากภาษาอื่นส่วนใหญ่
- ใช้ `mut` เพื่อทำให้ตัวแปรเปลี่ยนค่าได้ และแยกแยะความแตกต่างระหว่าง `mut` กับ **shadowing** ได้อย่างชัดเจน
- ใช้ `const` และ `static` ได้อย่างถูกต้อง และอธิบายความแตกต่างเชิงลึกจาก `let` ได้
- เลือกใช้ชนิดข้อมูล scalar (integer, float, bool, char) ให้เหมาะกับงาน พร้อมเข้าใจขนาดหน่วยความจำและขอบเขตค่าของแต่ละชนิด
- ใช้ชนิดข้อมูล compound คือ tuple และ array ได้ ทั้งการสร้าง เข้าถึงข้อมูล และ destructure
- เข้าใจว่า Rust ใช้ type inference เมื่อไหร่ และต้องระบุ type annotation ชัดเจนเมื่อไหร่
- อธิบายพฤติกรรมของ integer overflow ที่ต่างกันระหว่าง debug build กับ release build ได้ และเลือกใช้ `wrapping_*`,
  `checked_*`, `saturating_*`, `overflowing_*` ได้ถูกสถานการณ์
- แปลงชนิดข้อมูลตัวเลขด้วย `as` และด้วย `TryFrom`/`TryInto` ได้อย่างปลอดภัย และรู้ว่าเมื่อไหร่ควรใช้แบบไหน

## ความรู้ที่ต้องมีมาก่อน

- **Part 1**: ติดตั้ง Rust toolchain, เขียนและรันโปรแกรม `Hello, World!` ได้แล้ว, รู้จักโครงสร้าง `fn main() { ... }`
  และ macro `println!`
- **Part 2**: สร้างและรันโปรเจกต์ด้วย `cargo new`, `cargo run`, `cargo check` ได้แล้ว (บทนี้จะใช้ `cargo run` /
  `cargo run --release` ในการทดลองโค้ดทุกตัวอย่าง)

ถ้ายังไม่เคยสร้างโปรเจกต์ cargo มาก่อน แนะนำให้สร้างโปรเจกต์ทดลองไว้สักหนึ่งตัว เช่น
`cargo new playground` แล้ววาง snippet โค้ดแต่ละตัวอย่างในบทนี้ลงใน `src/main.rs` เพื่อรันดูผลจริงตามไปด้วย
เพราะบทนี้เป็นบทที่ "จับ Rust syntax ตัวจริง" เป็นครั้งแรก การรันโค้ดขนานไปกับการอ่านจะช่วยให้ concept ติดแน่นกว่าอ่านเฉย ๆ มาก

## เนื้อหา

### 3.1 `let` bindings และทำไม Rust เลือก immutable-by-default

ใน Rust เราประกาศตัวแปรด้วย keyword `let`:

```rust
fn main() {
    let apple_count = 5;
    println!("มีแอปเปิล {apple_count} ลูก");
}
```

จนถึงตรงนี้อาจดูไม่ต่างจาก Python หรือ JavaScript เลย แต่มีสิ่งหนึ่งที่ต่างอย่างมาก: **ตัวแปรที่สร้างด้วย `let`
เฉย ๆ (ไม่มี `mut`) จะไม่สามารถกำหนดค่าใหม่ได้อีก** ลองดูโค้ดนี้:

```rust
fn main() {
    let x = 5;
    x = 6; // พยายามกำหนดค่าใหม่ให้ x
    println!("{x}");
}
```

โค้ดนี้ **compile ไม่ผ่าน** ถ้าคุณลองรันด้วย `cargo run` จะได้ error หน้าตาแบบนี้ (ข้อความจริงจาก `rustc`):

```
error[E0384]: cannot assign twice to immutable variable `x`
 --> src/main.rs:3:5
  |
2 |     let x = 5;
  |         - first assignment to `x`
3 |     x = 6;
  |     ^^^^^ cannot assign twice to immutable variable
  |
help: consider making this binding mutable
  |
2 |     let mut x = 5;
  |         +++
```

สังเกตว่า compiler ไม่ได้แค่บอกว่า "ผิด" แต่ยังบอกบรรทัดที่ประกาศครั้งแรก (`first assignment to x`) และเสนอวิธีแก้ให้เลย
(`help: consider making this binding mutable`) — นี่คือปรัชญาการออกแบบ error message ของ Rust ที่เน้นให้ compiler
เป็น "เพื่อนช่วยสอน" ไม่ใช่แค่ผู้ตัดสิน ซึ่งจะเป็นแบบนี้ตลอดทั้งหลักสูตร

#### ทำไม Rust ถึงเลือกให้ immutable เป็นค่าเริ่มต้น

ภาษาโปรแกรมมิ่งส่วนใหญ่ที่คุณอาจคุ้นเคย (Python, JavaScript, Java, C, C++, Go) ให้ตัวแปร **mutable โดย default**
คุณต้องเขียน keyword เพิ่มเสริม (เช่น `final` ใน Java, `const` ใน JavaScript) ถ้าต้องการห้ามเปลี่ยนค่า Rust กลับเดินสวนทาง
โดยตั้งใจ และนี่ไม่ใช่ความเข้มงวดที่ไม่มีเหตุผล แต่มาจาก 3 เหตุผลเชิงวิศวกรรมที่ลึกซึ้ง:

**1. ความปลอดภัยและการลด bug จาก state ที่เปลี่ยนแปลงโดยไม่คาดคิด**

บั๊กจำนวนมากในโปรแกรมขนาดใหญ่เกิดจาก "ตัวแปรถูกเปลี่ยนค่าที่ไหนสักที่โดยที่เราไม่รู้ตัว" เช่นฟังก์ชัน A เขียนค่าใน list
ที่ฟังก์ชัน B ถืออยู่ ทำให้ B ทำงานผิดพลาดโดยไม่มีสาเหตุชัดเจนตอน debug เมื่อ Rust บอกว่าตัวแปร "ไม่ mutable" เว้นแต่คุณจะ
ประกาศไว้อย่างชัดเจน (`mut`) มันคือการสร้าง **"สัญญา" ที่ compiler ช่วยตรวจสอบให้** — เมื่อคุณเห็นตัวแปรที่ไม่มี `mut`
คุณมั่นใจได้ 100% ว่าค่าของมันจะไม่เปลี่ยนไปตลอด scope ที่มันมีชีวิตอยู่ ไม่ต้องไล่อ่านโค้ดทั้งไฟล์เพื่อเช็ค

**2. ทำให้ "อ่านและเข้าใจโค้ด" ง่ายขึ้นมาก (reasoning about code)**

เวลาคุณอ่านโค้ดคนอื่น (หรือโค้ดตัวเองเมื่อ 6 เดือนก่อน) การเห็น `let total = calculate_total(&items);` บอกคุณทันทีว่า
`total` คือค่าคงที่หลังจากบรรทัดนี้ ไม่ต้องกังวลว่าจะมีที่ไหนแก้ไขมันทีหลังในฟังก์ชันเดียวกัน สิ่งนี้สำคัญมากขึ้นเรื่อย ๆ
เมื่อโค้ดมีความซับซ้อนสูง หรือมีหลาย thread ทำงานพร้อมกัน (concurrency — เราจะเจาะลึกใน Part 37-40) เพราะ **ค่าที่ไม่เปลี่ยนแปลง
(immutable) ไม่มีทางเกิด data race ได้เลย** ไม่ว่าจะมีกี่ thread เข้ามาอ่านพร้อมกันก็ตาม

**3. เปิดโอกาสให้ compiler optimize ได้ดีกว่า**

เมื่อ compiler รู้แน่นอนว่าค่าหนึ่งจะไม่เปลี่ยนแปลงตลอดช่วงชีวิตของมัน มันสามารถทำ optimization หลายอย่างที่ทำไม่ได้ถ้าไม่มั่นใจ
เรื่อง mutability เช่น cache ค่าไว้ใน register แทนการอ่านจาก memory ซ้ำ ๆ, ทำ constant folding/propagation, หรือจัดเรียง
คำสั่งใหม่ (instruction reordering) โดยไม่ต้องกังวลเรื่อง side effect ผลลัพธ์คือโค้ดที่ compile ผ่านมักจะได้ machine code
ที่มีประสิทธิภาพสูงกว่าโดยที่คุณไม่ต้องทำอะไรเพิ่มเลย นี่คือส่วนหนึ่งของแนวคิด **zero-cost abstraction** ที่เราพูดถึงใน Part 1

พูดง่าย ๆ คือ Rust มองว่า "mutable เป็นกรณีพิเศษ ไม่ใช่กรณีปกติ" — ถ้าคุณต้องเปลี่ยนค่าได้จริง ๆ คุณต้องขอ (`mut`)
อย่างชัดเจน ซึ่งเป็นการบอก compiler (และคนอ่านโค้ด) ว่า "ระวังนะ ค่านี้เปลี่ยนได้"

#### `mut`: เมื่อคุณต้องการเปลี่ยนค่าจริง ๆ

แก้โค้ดด้านบนให้ compile ผ่านได้ง่าย ๆ โดยเพิ่ม `mut`:

```rust
fn main() {
    let mut x = 5;
    println!("ค่าเริ่มต้น: {x}");
    x = 6;
    println!("ค่าใหม่: {x}");
}
```

ผลลัพธ์:

```
ค่าเริ่มต้น: 5
ค่าใหม่: 6
```

`mut` ไม่ได้แปลว่า "ชนิดข้อมูลเปลี่ยนได้" (type ของตัวแปรที่ประกาศด้วย `let` จะถูกกำหนดตายตัวตั้งแต่ตอน binding แรก
เปลี่ยนชนิดทีหลังไม่ได้แม้จะมี `mut`) มันแปลว่า **"ค่า (value) ที่อยู่ใน memory ตำแหน่งเดิมนี้ เปลี่ยนแปลงได้"** เท่านั้น
ลองดูตัวอย่างที่แสดง error ถ้าพยายามเปลี่ยนชนิด:

```rust
fn main() {
    let mut count = 5;   // count เป็น i32
    count = "ห้า";        // พยายามใส่ &str ลงในตัวแปรที่เป็น i32
}
```

จะได้ error:

```
error[E0308]: mismatched types
 --> src/main.rs:3:13
  |
2 |     let mut count = 5;
  |                     - expected due to this value
3 |     count = "ห้า";
  |             ^^^^^^ expected integer, found `&str`
```

นี่ตอกย้ำว่า Rust เป็นภาษาที่ **statically typed** (ตรวจสอบ type ตอน compile time) และแต่ละตัวแปรมี **type เดียวตายตัว**
ไปตลอดชีวิตของ binding นั้น ไม่ว่าจะมี `mut` หรือไม่ก็ตาม — `mut` แค่อนุญาตให้เปลี่ยน "ค่า" ภายใน type เดิมเท่านั้น

#### เปรียบเทียบกับภาษาอื่นที่คุณอาจรู้จัก

| ภาษา | ค่าเริ่มต้นของตัวแปร | ต้องเขียนอะไรเพิ่มถ้าต้องการ immutable |
|---|---|---|
| Python | mutable (เปลี่ยนค่าได้เสมอ) | ไม่มีทางบังคับจริง ๆ (ใช้ convention ตัวพิมพ์ใหญ่ + `Final` type hint เท่านั้น) |
| JavaScript | mutable (`let`) | ใช้ `const` (แต่ `const obj = {}` ยัง mutate property ภายในได้) |
| Java | mutable | ใช้ `final` |
| Rust | **immutable (ค่าเริ่มต้น)** | ไม่ต้องเขียนอะไรเลย — ต้องเขียน `mut` เพิ่มถ้าต้องการให้ mutable ต่างหาก |

จะเห็นว่า Rust เป็นภาษาเดียวในตารางนี้ที่ "สลับ default" การเขียนโค้ดสไตล์ idiomatic Rust ในโปรเจกต์จริงจึงมักมีตัวแปร
ที่เป็น `mut` เป็นสัดส่วนน้อยกว่าตัวแปรธรรมดามาก เพราะเราจะพยายามออกแบบให้ค่าคำนวณเสร็จแล้วก็จบ ไม่ค่อยมีการ "ปรับค่าไปเรื่อย ๆ"
แบบภาษาอื่น (สไตล์นี้ใกล้เคียงกับ functional programming ผสมกับ imperative)

### 3.2 Shadowing: คนละเรื่องกับ `mut`

Rust มี feature ที่เรียกว่า **shadowing** ซึ่งให้คุณประกาศตัวแปรชื่อเดิมซ้ำด้วย `let` ใหม่ได้ โดย binding ใหม่จะ
"บัง" (shadow) ตัวเก่าไปตั้งแต่จุดนั้นเป็นต้นไป:

```rust
fn main() {
    let x = 5;
    let x = x + 1;   // สร้าง binding ใหม่ชื่อ x จากค่า x เดิม
    let x = x * 2;   // สร้าง binding ใหม่อีกครั้ง
    println!("ค่าสุดท้ายของ x: {x}"); // 12
}
```

ผลลัพธ์คือ `12` (5 + 1 = 6, แล้ว 6 * 2 = 12) แต่จุดสำคัญคือ **นี่ไม่ใช่การ mutate** — แต่ละบรรทัด `let x = ...`
คือการสร้างตัวแปรใหม่ทั้งหมด (ในทาง compiler มันคือ binding คนละตัวที่ใช้ชื่อซ้ำกันเฉย ๆ) ตัวแปร `x` ตัวแรกที่ค่า 5
ยังคง "เป็น immutable ตลอดชีวิตของมัน" มันไม่เคยถูกเปลี่ยนค่า มันแค่ถูก "บัง" ด้วยตัวแปรใหม่ที่ชื่อเดียวกัน

#### shadowing ต่างจาก `mut` อย่างไร

ความแตกต่างที่สำคัญที่สุดคือ **shadowing อนุญาตให้เปลี่ยน type ได้** เพราะมันคือการสร้างตัวแปรใหม่ทั้งหมด ไม่ใช่การเขียนค่า
ทับตัวแปรเดิม ตัวอย่างคลาสสิกที่แสดงประโยชน์ของมันชัดที่สุด — การแปลงชนิดข้อมูลของค่าที่มีชื่อความหมายเดียวกัน:

```rust
fn main() {
    let spaces = "   ";        // spaces เป็น &str (สตริงของช่องว่าง 3 ตัว)
    let spaces = spaces.len(); // spaces ตัวใหม่เป็น usize (ความยาวของสตริง)
    println!("จำนวนช่องว่าง: {spaces}"); // 3
}
```

โค้ดนี้ compile ผ่านสบาย ๆ เพราะ `let spaces = spaces.len();` คือการสร้างตัวแปรใหม่ (type `usize`) ที่บังตัวแปรเก่า
(type `&str`) ไปเลย ทั้งสองตัวไม่มีความเกี่ยวข้องกันในเชิง memory อีกต่อไป — ในตัวอย่างนี้เราไม่จำเป็นต้องคิดชื่อใหม่
อย่าง `space_count` ทั้ง ๆ ที่มันคือ "แนวคิดเดียวกัน" เพียงแค่คนละรูปแบบข้อมูล (สตริงของช่องว่าง เทียบกับ จำนวนช่องว่าง)

ลองเปรียบเทียบกับการพยายามทำแบบเดียวกันด้วย `mut` ดูว่าทำไมมันทำไม่ได้:

```rust
fn main() {
    let mut spaces = "   ";  // spaces เป็น &str
    spaces = spaces.len();   // พยายามใส่ usize ลงในตัวแปรที่เป็น &str
    println!("{spaces}");
}
```

ได้ error ทันที:

```
error[E0308]: mismatched types
 --> src/main.rs:3:14
  |
2 |     let mut spaces = "   ";
  |                      ----- expected due to this value
3 |     spaces = spaces.len();
  |              ^^^^^^^^^^^^ expected `&str`, found `usize`
```

นี่คือหลักฐานชัดเจนว่า `mut` แค่อนุญาตให้เปลี่ยน**ค่า**ภายใน**ชนิดเดิม** แต่ shadowing คือการสร้างตัวแปรใหม่ทั้งชนิดข้อมูล
เลือกใช้แบบไหนขึ้นกับความหมาย:

- ใช้ **`mut`** เมื่อแนวคิดคือ "ค่าตัวเดียวกัน กำลังถูกปรับเปลี่ยนไปตามเวลา" (เช่น ตัวนับ, accumulator ใน loop)
- ใช้ **shadowing** เมื่อแนวคิดคือ "แปลงข้อมูลจากรูปแบบหนึ่งไปอีกรูปแบบหนึ่ง" โดยที่ชื่อตัวแปรยังสื่อความหมายเดิมได้ดีที่สุด
  (เช่น string → number ที่ parse แล้ว, ค่า raw input → ค่าที่ validate แล้ว)

#### shadowing ใช้ใน scope ย่อยได้ และคืนตัวเดิมเมื่อออกจาก scope

shadowing ยังทำงานร่วมกับ block `{ }` ได้อย่างน่าสนใจ:

```rust
fn main() {
    let x = 5;

    {
        let x = x * 2; // shadow x ภายใน block นี้เท่านั้น
        println!("ค่า x ภายใน block: {x}"); // 10
    }

    println!("ค่า x ภายนอก block: {x}"); // 5 (กลับไปเป็นตัวแปรเดิม)
}
```

ผลลัพธ์:

```
ค่า x ภายใน block: 10
ค่า x ภายนอก block: 5
```

เพราะตัวแปร `x` ตัวที่สองมี**scope**จำกัดอยู่ภายใน `{ }` เท่านั้น เมื่อออกจาก block ตัวแปรตัวนั้นก็หมดอายุ (จะเจาะลึกเรื่อง
scope และการ drop ค่าใน Part 6 เรื่อง Ownership) ตัวแปร `x` ตัวแรกที่อยู่นอก block จึงยังคงมีค่า `5` เหมือนเดิมไม่เปลี่ยนแปลง
— นี่คืออีกหลักฐานว่า shadowing ไม่ใช่การ mutate ตัวแปรเดิมเลยแม้แต่นิดเดียว มันคนละตัวแปรกันโดยสิ้นเชิง

### 3.3 Constants (`const`) และ `static`

นอกจาก `let` แล้ว Rust ยังมี `const` และ `static` สำหรับประกาศค่าคงที่ ซึ่งมีความหมายและข้อจำกัดต่างจาก `let` พอสมควร

#### `const`

```rust
const MAX_LOGIN_ATTEMPTS: u32 = 5;
const SITE_NAME: &str = "ร้านค้าออนไลน์ของฉัน";

fn main() {
    println!("อนุญาตให้ login ผิดได้ไม่เกิน {MAX_LOGIN_ATTEMPTS} ครั้ง");
    println!("ยินดีต้อนรับสู่ {SITE_NAME}");
}
```

`const` แตกต่างจาก `let` ในสาระสำคัญ 3 อย่าง:

1. **ห้ามใช้ `mut` กับ `const` เด็ดขาด** — `const` immutable เสมอ ไม่มีข้อยกเว้น (ต่างจาก `let` ที่ยังเลือกได้ว่าจะ `mut`
   หรือไม่)
2. **ต้องระบุ type annotation เสมอ** — `let x = 5;` ให้ compiler infer type ได้ แต่ `const X = 5;` (ไม่ระบุ type)
   จะ compile ไม่ผ่าน ต้องเขียน `const X: i32 = 5;` เท่านั้น
3. **ค่าต้องคำนวณได้ตั้งแต่ compile time (compile-time evaluable)** — ค่าฝั่งขวาของ `const` ต้องเป็น constant expression
   ที่ compiler คำนวณผลลัพธ์ได้ทันทีตอน compile โดยไม่ต้องรันโปรแกรมจริง จะเรียกฟังก์ชันธรรมดา (ที่ไม่ใช่ `const fn`) ไม่ได้

ลองดูตัวอย่างที่ compile ไม่ผ่านเพราะข้อจำกัดข้อ 3:

```rust
fn get_max_users() -> u32 {
    100
}

const MAX_USERS: u32 = get_max_users(); // เรียกฟังก์ชันธรรมดาใน const ไม่ได้

fn main() {
    println!("{MAX_USERS}");
}
```

จะได้ error:

```
error[E0015]: cannot call non-const function `get_max_users` in constants
 --> src/main.rs:5:25
  |
5 | const MAX_USERS: u32 = get_max_users();
  |                         ^^^^^^^^^^^^^^^
  |
note: function `get_max_users` is not const
 --> src/main.rs:1:1
  |
1 | fn get_max_users() -> u32 {
  | ^^^^^^^^^^^^^^^^^^^^^^^^^
  = note: calls in constants are limited to constant functions, tuple structs and tuple variants
```

เหตุผลที่ Rust บังคับเรื่องนี้เพราะ `const` ถูกออกแบบมาให้ **compiler แทนที่ (inline) ค่าคงที่ลงไปตรง ๆ ทุกจุดที่ใช้งาน**
เหมือนการ copy-paste ค่าไปวางไว้ ณ ตำแหน่งที่เรียกใช้ ไม่ต่างจาก `#define` ใน C แต่ปลอดภัยกว่าเพราะมี type checking เต็มรูปแบบ
ด้วยความที่มันถูก "แทนที่" แบบนี้ compiler จำเป็นต้องรู้ค่าจริงตั้งแต่ตอน compile ไม่สามารถรอผลจากการรันโปรแกรมได้ — วิธีแก้
ตัวอย่างข้างบนคือเปลี่ยน `get_max_users` ให้เป็น `const fn` (ฟังก์ชันที่ประกาศว่าคำนวณได้ตอน compile time) ซึ่งเป็นเรื่องขั้นสูง
ที่จะเจาะลึกใน Part หลัง ๆ ตอนนี้จำแค่ว่า: **`const` เหมาะกับค่าที่รู้ตายตัวอยู่แล้วตั้งแต่เขียนโค้ด** เช่นค่าคงที่ทางฟิสิกส์/
คณิตศาสตร์, ขนาด buffer สูงสุด, จำนวนครั้งที่พยายามซ้ำได้สูงสุด

#### ข้อตกลงการตั้งชื่อ: SCREAMING_SNAKE_CASE

สังเกตว่าเราตั้งชื่อ `const` เป็นตัวพิมพ์ใหญ่ทั้งหมดคั่นด้วย underscore เช่น `MAX_LOGIN_ATTEMPTS` — นี่ไม่ใช่แค่ความชอบส่วนตัว
แต่เป็น **convention บังคับโดย clippy** (linter มาตรฐานที่เราจะเรียนใน Part 5) ถ้าตั้งชื่อ `const` เป็นตัวพิมพ์เล็กหรือ
camelCase จะได้ warning จาก compiler ตรง ๆ เลย:

```
warning: constant `maxLoginAttempts` should have an upper case name
 --> src/main.rs:1:7
  |
1 | const maxLoginAttempts: u32 = 5;
  |       ^^^^^^^^^^^^^^^^ help: convert the identifier to upper case: `MAX_LOGIN_ATTEMPTS`
```

การตั้งชื่อแบบนี้ช่วยให้เห็นชัดตาทันทีเมื่ออ่านโค้ดว่านี่คือ "ค่าคงที่ระดับ global ที่ไม่มีทางเปลี่ยน" ต่างจากตัวแปรทั่วไป
ที่ใช้ snake_case (เช่น `max_attempts`)

#### `static`

`static` คล้าย `const` ตรงที่เป็นค่าระดับ global ที่ประกาศได้นอก `fn` แต่ต่างกันในเชิง memory model:

```rust
static APP_VERSION: &str = "1.0.0";

fn main() {
    println!("เวอร์ชันแอป: {APP_VERSION}");
    println!("ตำแหน่งใน memory: {:p}", &APP_VERSION);
}
```

ความแตกต่างหลักระหว่าง `const` กับ `static`:

| ประเด็น | `const` | `static` |
|---|---|---|
| ตำแหน่งใน memory | ไม่มีตำแหน่งตายตัว — compiler inline ค่าไปวางทุกจุดที่ใช้ (เหมือน copy-paste) | มีตำแหน่งเดียวตายตัวใน memory ตลอดการรันโปรแกรม (`'static` lifetime) |
| Mutability | ห้าม mutable เด็ดขาด | ปกติ immutable เช่นกัน แต่**สามารถ**ประกาศเป็น `static mut` ได้ ซึ่งต้องอยู่ใน `unsafe` block เท่านั้น |
| ใช้บ่อยแค่ไหน | บ่อยมาก — ใช้แทนค่าคงที่ตัวเลข/สตริงทั่วไป | น้อยกว่า — มักใช้เมื่อต้องการ "การันตีว่ามีตำแหน่งเดียวใน memory จริง ๆ" เช่น shared config ขนาดใหญ่ |

`static mut` เป็นเรื่องที่ควรหลีกเลี่ยงในโค้ดทั่วไป เพราะการมีตัวแปร global ที่ mutable ได้จากหลายที่พร้อมกัน (โดยเฉพาะข้าม
thread) คือสาเหตุคลาสสิกของ data race ใน C/C++ — Rust จึงบังคับให้ต้องเขียนอยู่ใน `unsafe` block เสมอ (จะเจาะลึกเรื่อง
`unsafe` ใน Part 41) เพื่อเป็นสัญญาณเตือนว่า **compiler ไม่สามารถการันตีความปลอดภัยให้คุณได้อีกต่อไปในจุดนี้**
ในทางปฏิบัติ โปรแกรม Rust สมัยใหม่แทบไม่ใช้ `static mut` เลย — ถ้าต้องการ shared mutable state จะใช้ `Mutex`/`Arc`
(Part 39) หรือ `OnceLock`/`OnceCell` แทน ซึ่งปลอดภัยกว่ามาก

**สรุปสั้น ๆ สำหรับตอนนี้**: ในโค้ดระดับเริ่มต้น ให้ใช้ `const` เป็นหลักสำหรับค่าคงที่ ใช้ `static` เฉพาะเมื่อต้องการ
สตริง/ค่าที่มีตำแหน่งเดียวจริง ๆ ใน memory (เช่นสำหรับ FFI หรือ embedded programming ที่จะเรียนใน Part หลัง ๆ)

### 3.4 ชนิดข้อมูล Scalar: จำนวนเต็ม (Integers)

Rust เป็นภาษาที่ **statically typed** — ทุกค่าต้องมี type ที่รู้แน่นอนตอน compile time ชนิดข้อมูลพื้นฐานที่สุด
(scalar types) มี 4 กลุ่ม: integer, floating-point, boolean, และ character เริ่มจาก integer ก่อน

Rust มี integer type ให้เลือกมากถึง 12 ชนิด แบ่งเป็น 2 กลุ่มตาม signed/unsigned และแบ่งย่อยตามขนาดบิต:

| ขนาด (bit) | Signed (ค่าติดลบได้) | Unsigned (ค่าติดลบไม่ได้) |
|---|---|---|
| 8-bit | `i8` | `u8` |
| 16-bit | `i16` | `u16` |
| 32-bit | `i32` | `u32` |
| 64-bit | `i64` | `u64` |
| 128-bit | `i128` | `u128` |
| ตามขนาด pointer ของเครื่อง | `isize` | `usize` |

**Signed** (`i*`) เก็บได้ทั้งค่าบวกและลบ ใช้ two's complement representation แบบเดียวกับภาษาอื่นส่วนใหญ่ **Unsigned**
(`u*`) เก็บได้แต่ค่าบวกและศูนย์เท่านั้น แต่แลกกับการได้ขอบเขตค่าบวกที่กว้างขึ้นเป็น 2 เท่าในจำนวนบิตเท่ากัน

ขอบเขตค่าของแต่ละชนิดคำนวณจากสูตรมาตรฐาน: signed n-bit เก็บได้ตั้งแต่ −(2^(n−1)) ถึง 2^(n−1)−1 ส่วน unsigned n-bit
เก็บได้ตั้งแต่ 0 ถึง 2^n−1

| Type | ขอบเขตค่า |
|---|---|
| `i8` | −128 ถึง 127 |
| `u8` | 0 ถึง 255 |
| `i16` | −32,768 ถึง 32,767 |
| `u16` | 0 ถึง 65,535 |
| `i32` | −2,147,483,648 ถึง 2,147,483,647 (ประมาณ ±2.1 พันล้าน) |
| `u32` | 0 ถึง 4,294,967,295 (ประมาณ 4.2 พันล้าน) |
| `i64` / `u64` | ±9.2 ล้านล้านล้าน (10^18 ระดับ) |
| `i128` / `u128` | ±1.7×10^38 (ใหญ่มากจนแทบไม่มีใครใช้ในงานทั่วไป — เหมาะกับ cryptography, ตัวเลขทางการเงินขนาดยักษ์) |

ถ้าไม่ระบุชนิดของ integer เลย Rust จะ**default เป็น `i32`** เสมอ:

```rust
fn main() {
    let x = 42; // ไม่ได้ระบุ type — compiler จะ infer เป็น i32
    println!("type ของ x คือ i32 โดย default, ค่า = {x}");
}
```

`i32` ถูกเลือกเป็น default เพราะเป็นจุดที่สมดุลที่สุดในทางปฏิบัติ — เร็วเท่ากับ (หรือเร็วกว่า) `i64` บน CPU ส่วนใหญ่
ในปัจจุบัน แต่มีขอบเขตค่าที่กว้างพอสำหรับงานทั่วไปเกินกว่า 99% ของกรณีใช้งาน

#### `isize` และ `usize`: ขนาดผูกกับ pointer ของเครื่อง

`isize` และ `usize` เป็นชนิดพิเศษที่ **ขนาดบิตไม่ตายตัว** แต่ขึ้นกับ **architecture ของเครื่องที่ compile ไป** —
บนเครื่อง 64-bit (เกือบทุกเครื่องในปัจจุบัน) `usize`/`isize` จะมีขนาด 64 บิต (เท่ากับ `u64`/`i64`) แต่ถ้า compile ไปยัง
target แบบ 32-bit (เช่น embedded บางรุ่น หรือ WebAssembly แบบ wasm32) มันจะกลายเป็น 32 บิตทันทีโดยอัตโนมัติ

```rust
fn main() {
    println!("ขนาด usize บนเครื่องนี้: {} bytes", std::mem::size_of::<usize>());
    println!("ขนาด isize บนเครื่องนี้: {} bytes", std::mem::size_of::<isize>());
}
```

บนเครื่อง 64-bit ทั่วไปจะได้ผลลัพธ์:

```
ขนาด usize บนเครื่องนี้: 8 bytes
ขนาด isize บนเครื่องนี้: 8 bytes
```

เหตุผลที่ต้องมีชนิดที่ผูกกับขนาด pointer แบบนี้ก็เพราะ **`usize` คือชนิดที่ Rust ใช้สำหรับ index และความยาวของ collection
เสมอ** (เช่น `.len()` ของ array, `Vec`, `String` คืนค่าเป็น `usize` ทั้งหมด) เพราะในทางทฤษฎี ขนาดของ collection ที่ใหญ่ที่สุด
ที่โปรแกรมสามารถ address ได้ ก็ถูกจำกัดด้วยขนาด address space ของเครื่องอยู่แล้ว (เครื่อง 64-bit address ได้สูงสุด
2^64 ตำแหน่ง) การผูก `usize` เข้ากับขนาด pointer จึงสมเหตุสมผลที่สุดทั้งในเชิงประสิทธิภาพและความถูกต้องเชิง logic:

```rust
fn main() {
    let numbers = [10, 20, 30, 40, 50];
    let index: usize = 2; // การ index array/slice ต้องเป็น usize เท่านั้น
    println!("ตำแหน่งที่ {index}: {}", numbers[index]);
    println!("จำนวนสมาชิกทั้งหมด: {}", numbers.len()); // .len() คืนค่าเป็น usize
}
```

ถ้าคุณลองใช้ integer type อื่น (เช่น `i32`) ไปเป็น index ตรง ๆ จะเจอ error ทันที เพราะ `Index` trait ของ array/slice
รับเฉพาะ `usize`:

```rust
fn main() {
    let numbers: [i32; 3] = [10, 20, 30];
    let index: i32 = 1;
    println!("{}", numbers[index]);
}
```

```
error[E0277]: the type `[i32]` cannot be indexed by `i32`
 --> src/main.rs:4:22
  |
4 |     println!("{}", numbers[index]);
  |                      ^^^^^^^^^^^^ slice indices are of type `usize` or ranges of `usize`
```

วิธีแก้คือประกาศ `index` เป็น `usize` ตั้งแต่แรก หรือแปลงด้วย `as usize` (จะพูดถึง `as` แบบละเอียดในหัวข้อ 3.13)

#### รูปแบบการเขียนตัวเลขในโค้ด (numeric literals)

Rust รองรับการเขียนตัวเลขหลายฐาน และใส่ `_` คั่นหลักเพื่อความอ่านง่าย (compiler จะมองข้าม `_` ไปเลย ไม่มีผลต่อค่า):

```rust
fn main() {
    let decimal = 98_222;       // ฐาน 10 พร้อม underscore แบ่งหลักพัน
    let hex = 0xff;              // ฐาน 16 (hexadecimal) = 255
    let octal = 0o77;            // ฐาน 8 (octal) = 63
    let binary = 0b1111_0000;    // ฐาน 2 (binary) = 240
    let byte = b'A';              // byte literal (เฉพาะ u8) = 65

    println!("{decimal} {hex} {octal} {binary} {byte}");
}
```

และสามารถระบุ type ต่อท้ายตัวเลขได้ตรง ๆ โดยไม่ต้องเขียน type annotation แยก (เรียกว่า type suffix):

```rust
fn main() {
    let a = 255u8;      // เทียบเท่า let a: u8 = 255;
    let b = 1_000_000i64;
    let c = 3.14f32;
    println!("{a} {b} {c}");
}
```

### 3.5 ชนิดข้อมูล Scalar: ทศนิยม (Floating-Point)

Rust มีชนิดข้อมูลทศนิยม 2 แบบ: `f32` (single precision, 32 บิต) และ `f64` (double precision, 64 บิต) ทั้งคู่ implement
มาตรฐาน **IEEE 754** เหมือนภาษาส่วนใหญ่ (Java, JavaScript, Python, C, C++) `f64` คือ **default** เมื่อไม่ระบุ type
เพราะบน CPU สมัยใหม่ ความเร็วของ `f64` ใกล้เคียงกับ `f32` มาก แต่ให้ความแม่นยำสูงกว่ามาก (`f32` มีความแม่นยำประมาณ 6-9
หลักทศนิยม ส่วน `f64` มีประมาณ 15-17 หลัก)

```rust
fn main() {
    let x = 2.5;        // f64 โดย default
    let y: f32 = 3.0;   // ระบุ f32 ชัดเจน
    println!("{x} {y}");
}
```

#### ปัญหาความไม่เที่ยงตรงของ floating point (สำคัญมาก — ต้องเข้าใจก่อนใช้งานจริง)

เนื่องจาก IEEE 754 เก็บทศนิยมในรูปแบบฐาน 2 (binary fraction) ตัวเลขทศนิยมฐาน 10 บางค่า **ไม่สามารถแทนค่าได้เป๊ะ ๆ**
ในระบบฐาน 2 เลย (เหมือนที่ 1/3 เขียนเป็นทศนิยมฐาน 10 ไม่จบไม่ลง) นี่ไม่ใช่บั๊กของ Rust แต่เป็นข้อจำกัดของมาตรฐาน IEEE 754
ที่ทุกภาษาที่ใช้มันเจอเหมือนกันหมด (Python, JavaScript, Java, C ก็เจอปัญหานี้เช่นกัน):

```rust
fn main() {
    let sum = 0.1 + 0.2;
    println!("{sum}");        // 0.30000000000000004  (ไม่ใช่ 0.3 เป๊ะ ๆ!)
    println!("{}", sum == 0.3); // false
}
```

ผลลัพธ์:

```
0.30000000000000004
false
```

**บทเรียนสำคัญ**: ห้ามเปรียบเทียบค่า float ด้วย `==` โดยตรงเพื่อเช็คความเท่ากันแบบเป๊ะ ๆ (ยกเว้นกรณีพิเศษที่มั่นใจว่าค่า
มาจากการคำนวณที่เหมือนกันเป๊ะทุกขั้นตอน) วิธีที่ถูกต้องคือเช็คว่าค่าต่างกันน้อยกว่า threshold ที่ยอมรับได้ (epsilon):

```rust
fn main() {
    let sum: f64 = 0.1 + 0.2;
    let target: f64 = 0.3;
    let epsilon: f64 = 1e-10; // ค่าความต่างที่ยอมรับได้

    let is_close_enough = (sum - target).abs() < epsilon;
    println!("ใกล้เคียง 0.3 พอหรือไม่: {is_close_enough}"); // true
}
```

ด้วยเหตุนี้ **ในงานที่เกี่ยวกับเงิน (การเงิน, ราคาสินค้า) ควรหลีกเลี่ยงการใช้ float โดยตรง** และใช้ integer แทนเสมอ
(เช่น เก็บหน่วยเป็น "สตางค์" แทน "บาท" ที่มีทศนิยม) เราจะเห็นแนวทางนี้ชัดเจนในตัวอย่างระบบสินค้าคงคลังท้ายบทนี้

### 3.6 ชนิดข้อมูล Scalar: Boolean

`bool` มีแค่ 2 ค่า: `true` และ `false` มีขนาด 1 byte เท่านั้น:

```rust
fn main() {
    let is_active = true;
    let has_discount: bool = false;
    println!("Active: {is_active}, Discount: {has_discount}");
}
```

ข้อแตกต่างสำคัญจากภาษาอย่าง Python, JavaScript, C คือ **Rust ไม่มี "truthy/falsy" ใด ๆ ทั้งสิ้น** — เงื่อนไขใน `if`
ต้องเป็นค่า `bool` เท่านั้น เขียนตรง ๆ ไม่ได้:

```rust
fn main() {
    let count = 5;
    if count { // พยายามใช้ integer เป็นเงื่อนไขตรง ๆ แบบ Python/JS/C
        println!("มีของ");
    }
}
```

```
error[E0308]: mismatched types
 --> src/main.rs:3:8
  |
3 |     if count {
  |        ^^^^^ expected `bool`, found integer
```

ต้องเขียนเงื่อนไขที่ให้ผลลัพธ์เป็น `bool` อย่างชัดเจน เช่น `if count > 0 { ... }` ความเข้มงวดนี้ (เหมือนกับกรณี `mut`
และ integer overflow ที่จะเห็นต่อไป) คือแนวคิดเดียวกัน: Rust เลือก **ความชัดเจนและป้องกันบั๊กที่มาจากความคลุมเครือ**
มากกว่าความสะดวกในการเขียนสั้น ๆ เราจะเจาะลึกเรื่อง `if`/`else` และ control flow แบบเต็ม ๆ ใน **Part 4** ถัดไป

### 3.7 ชนิดข้อมูล Scalar: Character (`char`)

`char` ใน Rust แทน **1 Unicode Scalar Value** และมีขนาดคงที่ **4 bytes เสมอ** ไม่ว่าตัวอักษรนั้นจะเป็นอักษรละติน,
อักษรไทย, อักษรจีน, หรือ emoji ก็ตาม:

```rust
fn main() {
    let letter = 'A';
    let thai_char = 'ก';
    let emoji = '😻';
    let chinese = '中';

    println!("{letter} {thai_char} {emoji} {chinese}");
    println!("ขนาดของ char คือ {} bytes เสมอ", std::mem::size_of::<char>());
}
```

ผลลัพธ์:

```
A ก 😻 中
ขนาดของ char คือ 4 bytes เสมอ
```

สังเกตว่า `char` ใช้ **single quote** (`'A'`) ต่างจาก string ที่ใช้ **double quote** (`"A"`) — นี่คือความแตกต่างที่ทำให้
มือใหม่สับสนบ่อยตอนย้ายมาจาก Python/JavaScript ที่ single/double quote ใช้แทนกันได้กับ string ทุกความยาว แต่ใน Rust
`'A'` คือ `char` ตัวเดียว ส่วน `"A"` คือ `&str` (string slice ที่จะเรียนใน Part 8) ที่มีความยาว 1 ตัวอักษร — เป็นชนิดข้อมูล
คนละชนิดกันโดยสิ้นเชิง ใช้แทนกันไม่ได้

#### ทำไม Rust เลือกให้ `char` เป็น 4 bytes เสมอ — เปรียบเทียบกับภาษาอื่น

การเลือก 4 bytes มาจากข้อเท็จจริงที่ว่า Unicode มีขอบเขต code point ตั้งแต่ `U+0000` ถึง `U+10FFFF` ซึ่งต้องใช้อย่างน้อย
21 บิตในการแทนค่าทั้งหมด — Rust จึงปัดขึ้นเป็น 4 bytes (32 บิต) เพื่อให้ **`char` หนึ่งตัวแทน 1 Unicode Scalar Value
ได้เสมอโดยไม่มีข้อยกเว้น** ไม่ว่าตัวอักษรนั้นจะอยู่ใน Basic Multilingual Plane หรือนอกเหนือจากนั้น (เช่น emoji ส่วนใหญ่
และอักษรจีนโบราณบางตัว)

เทียบกับภาษาอื่น:

| ภาษา | ขนาด "char" | ปัญหาที่พบ |
|---|---|---|
| **Rust** | 4 bytes เสมอ, 1 char = 1 Unicode Scalar Value เต็มรูปแบบ | ไม่มี — แต่กินพื้นที่มากกว่าตัวอักษร ASCII ที่จริงใช้แค่ 1 byte |
| **Java** | `char` คือ 2 bytes (UTF-16 code unit) | ตัวอักษรนอก Basic Multilingual Plane (เช่น emoji 😻) ต้องใช้ 2 `char` ประกอบกัน (surrogate pair) — 1 `char` เดียวแทนตัวอักษรไม่ได้เสมอไป |
| **C (แบบดั้งเดิม)** | `char` คือ 1 byte | แทนได้แค่ ASCII/Latin-1 ตรง ๆ ตัวอักษรที่ไม่ใช่ภาษาอังกฤษ (เช่นไทย, จีน) ต้องเข้ารหัสแบบ multi-byte (UTF-8) ซึ่ง 1 `char` ไม่เท่ากับ 1 ตัวอักษรที่มองเห็น |
| **Python 3** | ไม่มี type "char" แยก — string 1 ตัวอักษรก็ยังเป็น `str` | ใกล้เคียง Rust ในเชิงแนวคิด (Python str คือลำดับของ Unicode code point) แต่ไม่มี type ระดับตัวอักษรเดี่ยวแยกจาก string เลย |

ข้อดีของแนวทาง Rust คือ **นักพัฒนาไม่ต้องกังวลเรื่อง surrogate pair หรือการตัดตัวอักษรกลาง ๆ แบบผิด ๆ เหมือนใน Java/JS
เลย** — `char` หนึ่งตัวคือหนึ่งตัวอักษร Unicode เต็มรูปแบบเสมอ อย่างไรก็ตาม สิ่งที่ต้องระมัดระวังคือ **`String`/`&str`
ใน Rust ไม่ได้เก็บข้อมูลเป็น array ของ `char`** แต่เก็บเป็น UTF-8 bytes ดิบ ๆ (แต่ละตัวอักษรอาจใช้ 1-4 bytes ต่างกันไป)
ทำให้การ index string ด้วยตำแหน่งตัวเลขตรง ๆ (`s[0]`) ทำไม่ได้เลยใน Rust ซึ่งเป็นเรื่องที่เราจะเจาะลึกอย่างละเอียดใน
**Part 14** เรื่อง String และ UTF-8 — ตอนนี้จำแค่ว่า `char` (ตัวอักษรเดี่ยว) กับ `String`/`&str` (ลำดับของ bytes ที่เข้ารหัส
แบบ UTF-8) เป็นคนละเรื่องกันในเชิง memory layout

### 3.8 ชนิดข้อมูล Compound: Tuple

Tuple คือการรวมค่าหลาย ๆ ค่า **ที่อาจมีชนิดข้อมูลต่างกัน** ไว้ในตัวแปรเดียว มีขนาดคงที่ตั้งแต่ตอนประกาศ (เปลี่ยนจำนวน
สมาชิกทีหลังไม่ได้เลย):

```rust
fn main() {
    let person: (&str, i32, f64) = ("สมชาย", 30, 65.5);
    println!("{:?}", person);
}
```

ผลลัพธ์ (ใช้ `{:?}` เพราะ tuple ไม่ implement `Display` โดย default แต่ implement `Debug` ให้ใช้ debug print ได้เสมอ
ถ้าสมาชิกทุกตัวก็ implement `Debug`):

```
("สมชาย", 30, 65.5)
```

#### การเข้าถึงสมาชิกด้วย `.0`, `.1`, `.2`

ใช้ dot notation ตามด้วยตำแหน่ง (index) เริ่มจาก 0:

```rust
fn main() {
    let person: (&str, i32, f64) = ("สมชาย", 30, 65.5);

    let name = person.0;
    let age = person.1;
    let weight = person.2;

    println!("ชื่อ: {name}, อายุ: {age}, น้ำหนัก: {weight} กก.");
}
```

#### Destructuring: แตกค่าออกมาเป็นตัวแปรแยกในบรรทัดเดียว

วิธีที่นิยมมากกว่าการเข้าถึงด้วย `.0`/`.1`/`.2` (เพราะอ่านง่ายกว่ามาก) คือการ **destructure** ผ่าน pattern matching
ในฝั่งซ้ายของ `let`:

```rust
fn main() {
    let person = ("สมชาย", 30, 65.5);
    let (name, age, weight) = person; // destructure ทีเดียวได้ทั้ง 3 ตัวแปร

    println!("ชื่อ: {name}, อายุ: {age}, น้ำหนัก: {weight} กก.");
}
```

โค้ดสองแบบด้านบนทำงานเหมือนกันทุกประการ แต่แบบ destructure สื่อความหมายชัดกว่าเมื่อมีสมาชิกหลายตัว เพราะตั้งชื่อที่สื่อ
ความหมายให้แต่ละค่าได้ทันที ไม่ต้องจำว่า `.1` คืออะไร

ถ้าต้องการข้ามสมาชิกบางตัวที่ไม่ใช้ ใช้ `_` (underscore) แทนได้:

```rust
fn main() {
    let coordinate = (10.5, 20.3, 100.0); // (x, y, z)
    let (x, y, _) = coordinate; // ไม่สนใจค่า z
    println!("x = {x}, y = {y}");
}
```

#### Unit Tuple `()`

Tuple ที่ไม่มีสมาชิกเลย `()` เรียกว่า **unit type** ซึ่งมีความหมายพิเศษมากใน Rust — มันคือ type ที่ใช้แทน "ไม่มีค่าอะไร
ที่มีความหมาย" (คล้าย `void` ใน C/Java แต่ต่างกันที่ `()` เป็น type จริงที่มีค่าจริง ไม่ใช่แค่การไม่มี return type):

```rust
fn main() {
    let unit: () = ();
    println!("{:?}", unit); // ()

    // ฟังก์ชันที่ไม่ระบุ return type จริง ๆ แล้ว return () โดย implicit
    let result = print_hello();
    println!("{:?}", result); // ()
}

fn print_hello() {
    println!("Hello!");
    // ไม่มี return statement → คืนค่า () โดยอัตโนมัติ
}
```

ทุกฟังก์ชันใน Rust ที่ไม่ได้ระบุ `-> Type` จะมี return type เป็น `()` โดยปริยาย นี่คือเหตุผลที่ `fn main() { ... }`
ที่เราเขียนมาตั้งแต่ Part 1 ไม่มี `-> ()` ต่อท้าย — เพราะมันเป็นค่า default ที่ compiler เติมให้เอง เราจะพูดถึงเรื่อง
return type ของฟังก์ชันอย่างละเอียดใน **Part 4**

### 3.9 ชนิดข้อมูล Compound: Array

Array คือชุดของค่าที่มี **ชนิดข้อมูลเดียวกันทั้งหมด** และมี **ความยาวคงที่ที่รู้แน่นอนตั้งแต่ compile time** เขียนด้วย
syntax `[T; N]` โดย `T` คือชนิดข้อมูลของสมาชิก และ `N` คือจำนวนสมาชิก (ต้องเป็นค่าคงที่ ไม่ใช่ตัวแปรที่เปลี่ยนได้):

```rust
fn main() {
    let numbers: [i32; 5] = [1, 2, 3, 4, 5];
    println!("{:?}", numbers);
    println!("ความยาว: {}", numbers.len());
    println!("สมาชิกตัวแรก: {}", numbers[0]);
    println!("สมาชิกตัวสุดท้าย: {}", numbers[numbers.len() - 1]);
}
```

ผลลัพธ์:

```
[1, 2, 3, 4, 5]
ความยาว: 5
สมาชิกตัวแรก: 1
สมาชิกตัวสุดท้าย: 5
```

ถ้าต้องการสร้าง array ที่สมาชิกทุกตัวมีค่าเดียวกัน ใช้ syntax ย่อ `[value; length]` (สังเกตว่าคล้าย type syntax
`[T; N]` มาก แต่ตำแหน่งซ้ายของ `;` เป็นค่าจริง ไม่ใช่ type):

```rust
fn main() {
    let zeros = [0; 10];       // array ของเลข 0 จำนวน 10 ตัว, type คือ [i32; 10]
    let default_grades = ["ยังไม่ได้ประเมิน"; 5]; // array ของ &str เดียวกัน 5 ตัว

    println!("{:?}", zeros);
    println!("{:?}", default_grades);
}
```

#### Array vs Vec: เมื่อไหร่ควรใช้อะไร (teaser สำหรับ Part 13)

ความจำกัดที่สำคัญที่สุดของ array คือ **ความยาวต้องรู้ตายตัวตอน compile time และเปลี่ยนแปลงไม่ได้เลยตลอดการรันโปรแกรม**
คุณ push หรือ pop สมาชิกเข้า/ออกจาก array ไม่ได้ ถ้าต้องการ collection ที่ขนาดเปลี่ยนแปลงได้ตอน runtime (เพิ่ม/ลบสมาชิก
ไปเรื่อย ๆ ตามข้อมูลที่รับเข้ามา) ต้องใช้ **`Vec<T>`** ซึ่งเป็น collection ที่จองพื้นที่บน heap และขยาย/หดขนาดได้แบบ dynamic
— เราจะเรียน `Vec<T>` แบบละเอียดเต็ม ๆ ใน **Part 13**

ตอนนี้จำหลักการเลือกใช้ง่าย ๆ ไว้ก่อน:

| สถานการณ์ | ควรใช้ |
|---|---|
| รู้จำนวนสมาชิกแน่นอนตั้งแต่เขียนโค้ด และไม่เปลี่ยนแปลงเลย (เช่น กระดานหมากรุก 8x8, สีของ RGB 3 ช่อง) | `[T; N]` (array) |
| ไม่รู้จำนวนสมาชิกล่วงหน้า หรือจำนวนเปลี่ยนไปตาม runtime (เช่น รายการสินค้าในตะกร้า, ผลลัพธ์จากการอ่านไฟล์) | `Vec<T>` |

การใช้ array เมื่อขนาดตายตัวจริง ๆ ยังมีข้อดีด้าน performance เพิ่มเติมคือ **ไม่ต้อง allocate memory บน heap เลย**
array ถูกจองพื้นที่บน stack (หรือ inline อยู่ใน struct ที่ครอบมันอยู่) ทำให้เร็วกว่าการเข้าถึง heap ผ่าน `Vec` เล็กน้อย
และไม่มี overhead จาก dynamic allocation

#### การเข้าถึงสมาชิกเกินขอบเขต (out-of-bounds) — panic ตอน runtime

Rust ตรวจสอบขอบเขตของ array **ทุกครั้ง** ก่อนเข้าถึงสมาชิกด้วย `[]` ถ้า index เกินขอบเขต โปรแกรมจะ **panic** (หยุดทำงาน
ทันทีพร้อม error message) แทนที่จะอ่าน memory ที่ไม่ควรอ่าน (ต่างจาก C/C++ ที่การอ่านเกินขอบเขต array คือ **undefined
behavior** ที่อาจทำให้โปรแกรม crash แบบสุ่ม หรือแม้แต่กลายเป็นช่องโหว่ความปลอดภัยได้เลย):

```rust
fn read_at(numbers: &[i32; 3], index: usize) -> i32 {
    numbers[index]
}

fn main() {
    let numbers = [1, 2, 3];
    let result = read_at(&numbers, 5); // index 5 เกินขอบเขต (มีแค่ 0, 1, 2)
    println!("{result}");
}
```

โปรแกรมจะ compile ผ่าน (เพราะ `index` เป็นค่าที่มาจาก parameter ของฟังก์ชัน compiler ไม่รู้ค่าแน่นอนตอน compile time)
แต่ตอนรันจะ panic ทันทีด้วยข้อความ:

```
thread 'main' panicked at src/main.rs:2:5:
index out of bounds: the len is 3 but the index is 5
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

การที่ Rust เลือก **panic ทันทีอย่างชัดเจน** แทนการปล่อยให้อ่าน memory เพี้ยน ๆ ต่อไปเงียบ ๆ คือหลักการออกแบบเดียวกับ
เรื่อง overflow ที่เราจะเห็นในหัวข้อ 3.12 — Rust เลือก **"fail fast and loud"** (พังทันทีและพังให้เห็นชัด ๆ) มากกว่า
"เดินหน้าต่อไปด้วยข้อมูลที่อาจผิดเพี้ยน" เพราะบั๊กที่ตรวจจับได้ทันทีนั้นแก้ง่ายกว่าบั๊กที่แฝงอยู่เงียบ ๆ แล้วโผล่มาสร้างปัญหา
ที่จุดอื่นของโปรแกรมในเวลาต่อมามาก

### 3.10 Type Inference กับ Type Annotation: Rust รู้ type ให้เมื่อไหร่ และต้องบอกเมื่อไหร่

Rust มี **type inference** ที่ทรงพลังมาก ในหลายกรณีคุณไม่ต้องเขียน type annotation เลยเพราะ compiler เดาได้จาก context:

```rust
fn main() {
    let x = 5;              // infer เป็น i32 (default)
    let y = 5.0;             // infer เป็น f64 (default)
    let name = "สมชาย";       // infer เป็น &str
    let is_ready = true;     // infer เป็น bool
    let scores = vec![90, 85, 77]; // infer เป็น Vec<i32> จากค่าภายใน
}
```

อย่างไรก็ตาม มีหลายสถานการณ์ที่ compiler **เดาไม่ได้** และคุณ**ต้อง**ระบุ type annotation อย่างชัดเจน ไม่งั้นจะ compile
ไม่ผ่าน สถานการณ์คลาสสิกที่สุดคือการใช้ `.parse()` เพื่อแปลง string เป็นตัวเลข:

```rust
fn main() {
    let input = "42";
    let number = input.parse(); // compiler ไม่รู้จะแปลงเป็น type ไหน!
    println!("{number:?}");
}
```

```
error[E0284]: type annotations needed for `Result<_, _>`
 --> src/main.rs:3:9
  |
3 |     let number = input.parse();
  |         ^^^^^^         ----- type must be known at this point
  |
  = note: cannot satisfy `<_ as FromStr>::Err == _`
help: consider giving `number` an explicit type, where the type for type parameter `F` is specified
  |
3 |     let number: Result<F, _> = input.parse();
  |               ++++++++++++++
```

เหตุผลคือ `.parse()` เป็น **generic method** ที่แปลง string ไปเป็น type ไหนก็ได้ที่ implement trait `FromStr`
(อาจเป็น `i32`, `f64`, `u8`, หรือแม้แต่ `bool`) compiler จึงไม่มีทางรู้ได้เลยว่าคุณต้องการผลลัพธ์เป็น type ไหน
ถ้าไม่มีข้อมูลอื่นมาช่วยบอก มีวิธีแก้ 2 แบบ:

**วิธีที่ 1: ระบุ type ให้ตัวแปรที่รับผลลัพธ์**

```rust
fn main() {
    let input = "42";
    let number: i32 = input.parse().unwrap(); // ระบุ type ผ่านตัวแปร
    println!("{number}");
}
```

**วิธีที่ 2: ระบุ type ผ่าน generic parameter ตรง ๆ ด้วย `::<Type>` (เรียกกันเล่น ๆ ว่า "turbofish")**

```rust
fn main() {
    let input = "42";
    let number = input.parse::<i32>().unwrap(); // ระบุ type ผ่าน ::<i32>
    println!("{number}");
}
```

ทั้งสองวิธีให้ผลลัพธ์เหมือนกันทุกประการ ต่างกันแค่สไตล์การเขียน — วิธีที่ 1 นิยมกว่าเมื่อมีตัวแปรรับผลลัพธ์อยู่แล้ว
ส่วนวิธีที่ 2 (turbofish `::<>`) มีประโยชน์มากเมื่อคุณต้องการเรียกใช้ค่าต่อ (chain) ทันทีโดยไม่อยากสร้างตัวแปรแยก
เช่น `input.parse::<i32>().unwrap() * 2`

(หมายเหตุ: `.unwrap()` ในตัวอย่างข้างบนใช้ดึงค่าออกจาก `Result<T, E>` ที่ `.parse()` คืนกลับมา — ถ้า parse ไม่สำเร็จ
(เช่น string ไม่ใช่ตัวเลข) โปรแกรมจะ panic ทันที นี่เป็นวิธีเขียนแบบง่ายที่สุดสำหรับตอนนี้ เราจะเรียนการจัดการ error
แบบเหมาะสมกว่านี้ด้วย `Result<T, E>` และ `?` operator แบบละเอียดใน **Part 12**)

#### กรณี ambiguous อีกแบบ: `.collect()`

อีกตัวอย่างคลาสสิกของ ambiguity คือ `.collect()` ซึ่งแปลง iterator เป็น collection ชนิดใดก็ได้ที่ implement
`FromIterator` (เช่น `Vec<T>`, `HashSet<T>`, `String`):

```rust
fn main() {
    let numbers = [1, 2, 3, 4, 5];
    let doubled: Vec<i32> = numbers.iter().map(|x| x * 2).collect();
    println!("{doubled:?}"); // [2, 4, 6, 8, 10]
}
```

ถ้าเอา `: Vec<i32>` ออกไปโดยไม่มีข้อมูลอื่นบอก type เลย จะเจอ error คล้าย ๆ กับกรณี `.parse()`:

```
error[E0283]: type annotations needed
 --> src/main.rs:3:9
  |
3 |     let doubled = numbers.iter().map(|x| x * 2).collect();
  |         ^^^^^^^                                 ------- type must be known at this point
  |
  = note: cannot satisfy `_: FromIterator<i32>`
help: consider giving `doubled` an explicit type
  |
3 |     let doubled: Vec<_> = numbers.iter().map(|x| x * 2).collect();
  |                ++++++++
```

(เราจะเจาะลึกเรื่อง iterator, `.map()`, `.collect()` และ trait `FromIterator` แบบเต็ม ๆ ใน **Part 25-26** ตอนนี้แค่
ให้เห็นภาพว่าทำไม type inference ถึงต้องการความช่วยเหลือในบางจุด)

**หลักการจำง่าย ๆ**: compiler ต้องการ "จุดยึด" (anchor) อย่างน้อยหนึ่งจุดในการอนุมาน type เสมอ ถ้าไม่มีเลย (เช่นค่าที่ได้
มาจาก generic function ที่คืนได้หลาย type และไม่มีการใช้งานต่อที่บอก type ชัดเจน) คุณต้องระบุเองที่จุดใดจุดหนึ่ง
ไม่จำเป็นต้องเป็นที่ตัวแปรเสมอไป — อาจเป็นที่ parameter ของฟังก์ชันที่รับค่านั้นไปใช้ต่อก็ได้ ถ้า context นั้นบอก type ชัดเจน

### 3.11 พฤติกรรม Integer Overflow: Debug vs Release

นี่คือหัวข้อที่สร้างความประหลาดใจให้มือใหม่ Rust บ่อยที่สุดหัวข้อหนึ่ง เพราะพฤติกรรม**ต่างกันจริง ๆ ระหว่าง debug build
กับ release build**

ลองดูตัวอย่างนี้: `u8` เก็บค่าได้สูงสุด 255 ถ้าบวกเกินจะเกิดอะไรขึ้น? (สังเกตว่าตัวอย่างนี้ห่อค่าไว้ในฟังก์ชัน `add_one`
แทนการเขียน `255u8 + 1` ตรง ๆ ใน `main` — เหตุผลคือถ้าเขียนค่าคงที่ตรง ๆ ที่ compiler รู้ผลลัพธ์ได้ทันทีตอน compile time
Rust จะจับ overflow ได้ตั้งแต่ตอน compile และฟ้อง error แข็ง ๆ เลยทั้งใน debug และ release ไม่ต่างกัน — พฤติกรรมที่ต่างกัน
ระหว่าง debug/release ที่เรากำลังจะพูดถึงนี้ เกิดขึ้นเฉพาะกับค่าที่ compiler **รู้ไม่ได้ล่วงหน้า** ว่าจะ overflow หรือไม่
เช่นค่าที่มาจาก parameter ของฟังก์ชัน ซึ่งเป็นสถานการณ์ที่พบเจอจริงในโค้ด production เกือบทั้งหมดอยู่แล้ว):

```rust
fn add_one(x: u8) -> u8 {
    x + 1 // พยายามบวกเกินขอบเขตของ u8 ถ้า x = 255
}

fn main() {
    let x: u8 = 255;
    let y = add_one(x);
    println!("{y}");
}
```

ถ้ารันด้วย `cargo run` (ซึ่ง build แบบ **debug** โดย default):

```
thread 'main' panicked at src/main.rs:2:5:
attempt to add with overflow
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

โปรแกรม**panic ทันที** แต่ถ้ารันแบบ `cargo run --release` (build แบบ **release**) โค้ด**เดียวกันนี้**จะไม่ panic เลย
แต่กลับได้ผลลัพธ์:

```
0
```

#### ทำไมถึงต่างกัน?

Rust ตั้งใจออกแบบให้พฤติกรรมนี้ต่างกันระหว่าง 2 profile โดยดูจาก flag `overflow-checks` ใน `Cargo.toml` (ค่า default
ต่างกันตาม profile):

- **Debug profile** (`cargo build` / `cargo run` แบบปกติ): `overflow-checks = true` โดย default — compiler จะแทรก
  การตรวจสอบ overflow ทุกครั้งที่มีการคำนวณเลขจำนวนเต็ม ถ้าเกิด overflow จะ **panic ทันที** เพื่อให้คุณเจอบั๊กเร็วที่สุด
  ตั้งแต่ตอนพัฒนา/เทส
- **Release profile** (`cargo build --release` / `cargo run --release`): `overflow-checks = false` โดย default —
  ไม่มีการตรวจสอบ overflow เลย เพื่อประสิทธิภาพสูงสุด (การเช็คทุกครั้งมี cost ด้าน CPU) เมื่อเกิด overflow ค่าจะ
  **"wrap around"** ตามหลัก **two's complement** เหมือนที่ C/C++ ทำมาโดยตลอด (255 + 1 กลับไปเป็น 0, เหมือนมาตรวัดที่วน
  กลับไปเลขต่ำสุดเมื่อเกินเลขสูงสุด)

เหตุผลเชิงปรัชญาการออกแบบคือ Rust มองว่า **integer overflow มักเป็นสัญญาณของบั๊กด้าน logic** (เช่น ลืมเช็คขอบเขต,
คำนวณผิดสูตร) การให้มัน panic ทันทีในช่วงพัฒนา/เทส (debug build) ช่วยให้เจอบั๊กเร็วที่สุด แต่การเช็คนี้มี runtime cost
เล็กน้อยในทุกการคำนวณเลข ซึ่งไม่คุ้มที่จะแบกไว้ใน production build (release) ที่ควรจะผ่านการเทสมาอย่างดีแล้ว จึงเลือก
ปิดการเช็คใน release เพื่อความเร็วสูงสุด — **แต่นี่หมายความว่าคุณห้ามพึ่งพาพฤติกรรม overflow แบบใดแบบหนึ่งโดยปริยาย
ในโค้ด production เด็ดขาด** ถ้าโค้ดของคุณมีโอกาส overflow ได้จริงในบางสถานการณ์ (ไม่ใช่บั๊กที่ต้องแก้ที่ logic)
คุณต้องเลือกพฤติกรรมที่ต้องการอย่างชัดเจนด้วย method ต่อไปนี้ ไม่ใช่ปล่อยให้พฤติกรรม default (ที่ต่างกันตาม build mode)
ตัดสินใจแทนคุณ

#### `wrapping_*`: บวก/ลบ/คูณ แบบ wrap around เสมอ (ไม่สน profile)

ถ้าคุณ**ต้องการ**พฤติกรรม wrap around แบบ two's complement เสมอ ไม่ว่าจะ debug หรือ release ใช้ method `wrapping_add`,
`wrapping_sub`, `wrapping_mul`:

```rust
fn main() {
    let x: u8 = 255;
    let y = x.wrapping_add(1);
    println!("{y}"); // 0 (เสมอ ไม่ว่า debug หรือ release)

    let z: u8 = 0;
    let w = z.wrapping_sub(1);
    println!("{w}"); // 255 (0 - 1 วนกลับไปค่าสูงสุด)
}
```

เหมาะกับกรณีที่ wrap around คือพฤติกรรมที่ต้องการจริง ๆ เช่น hash function บางแบบ, checksum, หรือตัวนับ (counter) ที่
ต้องการวนกลับไปศูนย์เมื่อครบรอบโดยไม่ panic (เช่น เกม progress bar ที่วนซ้ำ)

#### `checked_*`: คืน `Option<T>` — `None` เมื่อ overflow

ถ้าต้องการ**รู้ว่า overflow เกิดขึ้นหรือไม่** และจัดการมันอย่างชัดเจน (ไม่ panic ไม่ wrap เงียบ ๆ) ใช้ `checked_add`,
`checked_sub`, `checked_mul` ซึ่งคืนค่าเป็น `Option<T>`: `Some(ผลลัพธ์)` ถ้าไม่ overflow, `None` ถ้า overflow

```rust
fn main() {
    let x: u8 = 255;

    match x.checked_add(1) {
        Some(result) => println!("บวกสำเร็จ: {result}"),
        None => println!("เกิด overflow! ไม่สามารถบวกได้"),
    }

    let y: u8 = 200;
    match y.checked_add(50) {
        Some(result) => println!("บวกสำเร็จ: {result}"),
        None => println!("เกิด overflow! ไม่สามารถบวกได้"),
    }
}
```

ผลลัพธ์:

```
เกิด overflow! ไม่สามารถบวกได้
บวกสำเร็จ: 250
```

(`Option<T>` และ `match` เป็นเรื่องที่เราจะเรียนอย่างละเอียดใน Part 10-11 — ตอนนี้จำแค่ว่า `Some(x)` แปลว่า "มีค่า x"
และ `None` แปลว่า "ไม่มีค่า" ซึ่งเป็นวิธีที่ Rust ใช้แทนที่แนวคิด "null" ของภาษาอื่นแบบปลอดภัย) `checked_*` เหมาะที่สุด
เมื่อ overflow เป็นสถานการณ์ที่ **ต้องจัดการอย่างมีเหตุผลทางธุรกิจ** เช่น การตรวจสอบว่าจำนวนสินค้าในสต็อกจะเกินขีดจำกัดที่
ระบบรับได้หรือไม่ ก่อนที่จะยืนยันการทำรายการจริง

#### `saturating_*`: หยุดที่ค่าขอบเขตสูงสุด/ต่ำสุด ไม่ panic ไม่ wrap

`saturating_add`, `saturating_sub`, `saturating_mul` จะ**"เกาะติดขอบ"** ไว้ที่ค่าสูงสุดหรือต่ำสุดของ type นั้นเมื่อ
คำนวณเกินขอบเขต แทนที่จะ panic หรือ wrap กลับไปด้านตรงข้าม:

```rust
fn main() {
    let x: u8 = 255;
    let y = x.saturating_add(10);
    println!("{y}"); // 255 (ไม่ใช่ 9 แบบ wrap, ไม่ panic — หยุดที่ค่าสูงสุดของ u8)

    let z: u8 = 0;
    let w = z.saturating_sub(10);
    println!("{w}"); // 0 (หยุดที่ค่าต่ำสุดของ u8 คือ 0, ไม่ติดลบ)
}
```

เหมาะกับสถานการณ์ที่ "ค่าสูงสุด/ต่ำสุดที่เป็นไปได้" นั้นก็คือคำตอบที่สมเหตุสมผลอยู่แล้ว เช่น progress bar ที่ไม่ควรเกิน 100%
หรือ HP ของตัวละครในเกมที่ไม่ควรติดลบ (จะเห็นตัวเลือกนี้อีกครั้งในตัวอย่างระบบสินค้าคงคลังท้ายบท ตอนคำนวณส่วนลด)

#### `overflowing_*`: คืนทั้งผลลัพธ์และ flag ว่า overflow หรือไม่

`overflowing_add`, `overflowing_sub`, `overflowing_mul` คืนค่าเป็น **tuple** `(ผลลัพธ์แบบ wrap, bool บอกว่า overflow หรือไม่)`
— เป็นตัวเลือกที่ให้ข้อมูลมากที่สุด เพราะได้ทั้งค่าที่ wrap แล้วและรู้ด้วยว่ามัน overflow จริงหรือไม่ ในคำเรียกเดียว:

```rust
fn main() {
    let x: u8 = 255;
    let (result, did_overflow) = x.overflowing_add(1);
    println!("ผลลัพธ์: {result}, overflow เกิดขึ้นหรือไม่: {did_overflow}");
    // ผลลัพธ์: 0, overflow เกิดขึ้นหรือไม่: true

    let y: u8 = 100;
    let (result2, did_overflow2) = y.overflowing_add(50);
    println!("ผลลัพธ์: {result2}, overflow เกิดขึ้นหรือไม่: {did_overflow2}");
    // ผลลัพธ์: 150, overflow เกิดขึ้นหรือไม่: false
}
```

#### สรุปตารางเปรียบเทียบ 4 วิธีจัดการ overflow

| Method | คืนค่า | เมื่อ overflow ทำอะไร | เหมาะกับ |
|---|---|---|---|
| `+`, `-`, `*` (operator ปกติ) | ค่าตรง ๆ | debug: panic / release: wrap (พึ่งพาไม่ได้ ไม่ควรใช้ถ้ามีโอกาส overflow จริง) | การคำนวณที่มั่นใจว่าไม่มีทาง overflow |
| `wrapping_*` | ค่าตรง ๆ (wrap แล้ว) | wrap around เสมอ ทุก profile | ต้องการ wrap จริง ๆ เช่น hash, checksum |
| `checked_*` | `Option<T>` | คืน `None` | ต้องการรู้และจัดการ error อย่างชัดเจน |
| `saturating_*` | ค่าตรง ๆ (เกาะขอบ) | เกาะที่ค่า MIN/MAX ของ type | ค่าที่มี "เพดาน/พื้น" ตามธรรมชาติอยู่แล้ว |
| `overflowing_*` | `(ค่า, bool)` | คืนทั้ง wrap value และ flag | ต้องการทั้งค่าและสถานะในคำเรียกเดียว |

**คำแนะนำเชิงปฏิบัติ**: อย่าใช้ operator ธรรมดา (`+`, `-`, `*`) กับ integer ในจุดที่มีโอกาส overflow ได้จริงในข้อมูล
production (เช่น รับ input จากผู้ใช้ หรือคำนวณจากค่าที่ไม่รู้ขอบเขตล่วงหน้า) เพราะพฤติกรรมจะไม่แน่นอนระหว่าง debug/release
— ให้เลือกใช้ `checked_*`, `saturating_*`, หรือ `wrapping_*` ตามความหมายทางธุรกิจของโค้ดนั้นอย่างตั้งใจเสมอ

### 3.12 การแปลงชนิดข้อมูลตัวเลข: `as` กับ `TryFrom`/`TryInto`

Rust ไม่มีการแปลงชนิดข้อมูลตัวเลขแบบอัตโนมัติ (implicit conversion) เลย — ต่างจาก C/C++/Java ที่ยอมให้ `int` บวกกับ
`long` ได้ตรง ๆ โดย compiler แปลงให้อัตโนมัติ ใน Rust คุณต้องแปลง type ด้วยตัวเองอย่างชัดเจนเสมอ:

```rust
fn main() {
    let x: i32 = 5;
    let y: i64 = 10;
    let z = x + y; // พยายามบวก i32 กับ i64 ตรง ๆ
    println!("{z}");
}
```

```
error[E0308]: mismatched types
 --> src/main.rs:4:18
  |
4 |     let z = x + y;
  |                 ^ expected `i32`, found `i64`
```

การที่ Rust เข้มงวดขนาดนี้ (แม้จะดู "เข้มงวดเกินไป" ในสายตาคนที่มาจาก C/Java) เป็นเพราะการแปลง type ตัวเลขแบบอัตโนมัติ
คือหนึ่งในแหล่งบั๊กที่พบบ่อยที่สุดในภาษาอื่น (เช่นการบวก `int` กับ `long` แล้วผลลัพธ์เกินขอบเขตของ `int` แบบเงียบ ๆ)
Rust เลือกให้**ทุกการแปลง type ต้องเขียนออกมาให้เห็นชัด** เพื่อให้ผู้เขียนโค้ด (และผู้อ่านโค้ด) รู้ตัวเสมอว่ากำลังมีการ
แปลงข้อมูลเกิดขึ้น ณ จุดไหน

มีวิธีแปลง type หลัก ๆ 2 แบบ ที่ให้ trade-off ต่างกันชัดเจน:

#### `as`: แปลงตรงไปตรงมา แต่เสี่ยง "lossy" (สูญข้อมูล) แบบเงียบ ๆ

```rust
fn main() {
    let x: i32 = 5;
    let y: i64 = 10;
    let z = x as i64 + y; // แปลง x เป็น i64 ก่อนบวก
    println!("{z}"); // 15
}
```

`as` ใช้งานง่ายและเร็ว แต่มี**อันตรายสำคัญ**: ถ้าแปลงจาก type ที่มีขอบเขตค่ากว้างกว่าไปเป็น type ที่ขอบเขตแคบกว่า
(narrowing conversion) และค่าจริงเกินขอบเขตปลายทาง **`as` จะไม่เตือนอะไรเลย มันแค่ตัดบิตส่วนเกินทิ้งไปเงียบ ๆ**
(truncation) ผลลัพธ์ที่ได้อาจผิดเพี้ยนไปจากที่คาดหวังโดยสิ้นเชิง:

```rust
fn main() {
    let big: i64 = 300;
    let small = big as i8; // i8 เก็บได้แค่ -128 ถึง 127 แต่ 300 เกินขอบเขตไปมาก
    println!("{small}"); // 44 (ไม่ใช่ 300! และไม่มี warning หรือ error ใด ๆ)
}
```

ทำไมได้ `44`? เพราะ `as` ตัดเหลือแค่ 8 บิตต่ำสุดของค่า 300 เท่านั้น (300 ในเลขฐาน 2 คือ `1_0010_1100` ซึ่งใช้ 9 บิต
ตัดเหลือ 8 บิตต่ำสุดจะได้ `0010_1100` = 44 ในฐาน 10 และเพราะบิตซ้ายสุดเป็น 0 ผลลัพธ์จึงยังเป็นบวก) ลองดูอีกตัวอย่างที่
อันตรายกว่า — ค่าบวกที่กลายเป็นค่าลบไปเลยหลังแปลง:

```rust
fn main() {
    let big: i64 = 200;
    let small = big as i8;
    println!("{small}"); // -56 (!!) ค่าบวก 200 กลายเป็นค่าลบหลังแปลงแบบ as
}
```

`200` ในฐาน 2 คือ `1100_1000` (8 บิตพอดี) แต่เมื่อตีความ 8 บิตนี้แบบ **signed** (`i8`) บิตซ้ายสุดที่เป็น `1` จะถูกตีความ
ว่าเป็นเครื่องหมายลบตามหลัก two's complement ทำให้ผลลัพธ์กลายเป็น `-56` ไปเลย — นี่คือ**อันตรายตัวจริงของ `as`**:
มันไม่ใช่แค่ "ตัดข้อมูลบางส่วนหาย" แต่ผลลัพธ์อาจกลายเป็นค่าที่ผิดความหมายไปเลย (บวกกลายเป็นลบ) โดยไม่มี warning หรือ
error ใด ๆ ทั้งสิ้นตอน compile หรือ runtime — เป็นบั๊กแบบ "เงียบที่สุด" ที่หาสาเหตุยากมากถ้าเกิดในระบบจริง เช่นระบบคำนวณ
เงินที่แปลง type ผิดพลาดโดยไม่รู้ตัว

**เมื่อไหร่ใช้ `as` ได้อย่างปลอดภัย**: เมื่อคุณมั่นใจ 100% ว่าค่าที่แปลงจะไม่เกินขอบเขตปลายทางแน่นอน (เช่น แปลง `u8`
เป็น `u32` — เป็น widening conversion ที่ไม่มีทางสูญข้อมูล เพราะ `u32` เก็บได้กว้างกว่า `u8` เสมอ) หรือกรณีที่ตั้งใจ
ต้องการพฤติกรรม truncation จริง ๆ (พบน้อยมากในโค้ดทั่วไป)

#### `TryFrom` / `TryInto`: แปลงแบบปลอดภัย คืน `Result` ให้ตรวจสอบได้

ถ้าไม่มั่นใจว่าค่าจะพอดีกับ type ปลายทางหรือไม่ ควรใช้ `TryFrom`/`TryInto` ซึ่งคืนค่าเป็น `Result<T, E>` — บอกชัดเจนว่า
สำเร็จ (`Ok`) หรือล้มเหลว (`Err`) แทนที่จะเงียบ ๆ ตัดบิตทิ้งแบบ `as`:

```rust
fn main() {
    let big: i64 = 300;

    let result: Result<i8, _> = i8::try_from(big);

    match result {
        Ok(value) => println!("แปลงสำเร็จ: {value}"),
        Err(e) => println!("แปลงไม่ได้: {e}"),
    }
}
```

ผลลัพธ์:

```
แปลงไม่ได้: out of range integral type conversion attempted
```

(ตั้งแต่ Rust edition 2021 เป็นต้นไป trait `TryFrom`/`TryInto` อยู่ใน prelude อัตโนมัติแล้ว จึงไม่ต้องเขียน
`use std::convert::TryFrom;` เพิ่มเองอีก — เรียกใช้ได้ตรง ๆ เลยแบบในตัวอย่างนี้)

ลองเทียบกับกรณีที่ค่าพอดีกับขอบเขต:

```rust
fn main() {
    let ok_value: i64 = 100;

    let result: Result<i8, _> = i8::try_from(ok_value);

    match result {
        Ok(value) => println!("แปลงสำเร็จ: {value}"), // แปลงสำเร็จ: 100
        Err(e) => println!("แปลงไม่ได้: {e}"),
    }
}
```

`TryInto` คือ trait ฝั่งกลับด้าน ให้เขียนแบบ method call แทนได้ ความหมายเดียวกันทุกประการ:

```rust
fn main() {
    let big: i64 = 300;

    let result: Result<i8, _> = big.try_into();

    match result {
        Ok(value) => println!("แปลงสำเร็จ: {value}"),
        Err(e) => println!("แปลงไม่ได้: {e}"),
    }
}
```

`i8::try_from(big)` และ `big.try_into()` ทำงานเหมือนกันเป๊ะ ๆ — เลือกใช้ตามความอ่านง่ายในบริบทนั้น ๆ (`try_into()` มัก
อ่านง่ายกว่าเมื่อ chain ต่อจาก expression อื่น ส่วน `try_from()` มักอ่านง่ายกว่าเมื่อเริ่มต้นจาก type ที่ต้องการชัดเจน)

#### เมื่อไหร่ควรใช้ `as` เมื่อไหร่ควรใช้ `TryFrom`/`TryInto`

| สถานการณ์ | ควรใช้ |
|---|---|
| แปลงจาก type เล็กไป type ใหญ่กว่าเสมอ (`u8` → `u32`, `i16` → `i64`) — ไม่มีทางสูญข้อมูล | `as` (ปลอดภัยและชัดเจนอยู่แล้ว) |
| แปลงจาก type ใหญ่ไป type เล็กกว่า และ**ไม่มั่นใจ**ว่าค่าจะพอดีเสมอ (เช่น รับ input จากผู้ใช้, ข้อมูลจากภายนอก) | `TryFrom`/`TryInto` (ตรวจสอบและจัดการ error ได้) |
| ต้องการ truncation แบบตั้งใจจริง ๆ (พบน้อย) | `as` (แต่ต้องเขียน comment อธิบายเจตนาให้ชัดเจนเสมอ) |
| แปลงระหว่าง integer กับ float (`i32 as f64`, `f64 as i32`) | `as` (แต่ float → int จะตัดทศนิยมทิ้งเสมอ ไม่ใช่ round ปัดเศษ) |

กฎง่าย ๆ ที่ควรจำไว้ตลอดการเขียน Rust: **ถ้าค่าที่แปลงมาจากแหล่งที่คุณไม่ได้ควบคุม 100% (input ผู้ใช้, ข้อมูลจาก network,
ไฟล์, database) ให้ใช้ `TryFrom`/`TryInto` เสมอ** ส่วน `as` ควรสงวนไว้ใช้กับค่าที่มาจาก logic ภายในโปรแกรมที่คุณมั่นใจ
เรื่องขอบเขตอยู่แล้วเท่านั้น

### 3.13 ตัวอย่างจริง: ระบบคำนวณยอดขายสินค้าคงคลังขนาดเล็ก

มาลองรวมทุกอย่างที่เรียนมาในบทนี้เข้าด้วยกัน ผ่านตัวอย่างที่ใกล้เคียงงานจริง — ระบบคำนวณยอดขายของร้านอุปกรณ์คอมพิวเตอร์
เล็ก ๆ ที่ต้องจัดการราคา จำนวนสินค้า และยอดรวม โดยต้องระวังทั้งเรื่องความเที่ยงตรงของตัวเงิน (จึงเก็บเป็นสตางค์แบบ integer
ไม่ใช่ float) และเรื่อง overflow (เผื่อกรณีสั่งซื้อจำนวนมากผิดปกติ):

```rust
fn main() {
    // แทนสินค้าแต่ละตัวด้วย tuple: (ชื่อสินค้า, ราคาต่อหน่วยเป็น "สตางค์", จำนวนที่ขายได้)
    // เก็บราคาเป็นจำนวนเต็ม (สตางค์) แทนทศนิยม (บาท) เพื่อเลี่ยงปัญหาความไม่เที่ยงตรงของ floating point
    // ที่เราเห็นไปแล้วในหัวข้อ 3.5 (0.1 + 0.2 ไม่เท่ากับ 0.3 เป๊ะ ๆ)
    let inventory: [(&str, u32, u32); 3] = [
        ("เมาส์ไร้สาย", 29_900, 15),          // 299.00 บาท x 15 ชิ้น
        ("คีย์บอร์ดเมคานิคอล", 89_000, 8),     // 890.00 บาท x 8 ชิ้น
        ("แผ่นรองเมาส์เกมมิ่ง", 15_000, 40),   // 150.00 บาท x 40 ชิ้น
    ];

    // destructure แต่ละแถวออกมาเป็นตัวแปรที่สื่อความหมาย
    let (name_a, price_a, qty_a) = inventory[0];
    let (name_b, price_b, qty_b) = inventory[1];
    let (name_c, price_c, qty_c) = inventory[2];

    // price และ qty เป็น u32 ทั้งคู่ — ในทางทฤษฎีการคูณกันอาจ overflow ขอบเขตของ u32 ได้
    // (ราคาสูงมาก x จำนวนมากมาย) จึงยกระดับไปคำนวณเป็น u64 ก่อนคูณ ด้วย checked_mul
    // เพื่อความปลอดภัย แม้ในทางปฏิบัติของร้านเล็ก ๆ นี้จะไม่เกิดขึ้นจริงก็ตาม
    let subtotal_a: u64 = (price_a as u64)
        .checked_mul(qty_a as u64)
        .expect("คำนวณ subtotal ของสินค้า A เกิด overflow");
    let subtotal_b: u64 = (price_b as u64)
        .checked_mul(qty_b as u64)
        .expect("คำนวณ subtotal ของสินค้า B เกิด overflow");
    let subtotal_c: u64 = (price_c as u64)
        .checked_mul(qty_c as u64)
        .expect("คำนวณ subtotal ของสินค้า C เกิด overflow");

    // รวมยอดทั้งหมดด้วย checked_add ต่อ ๆ กัน ป้องกัน overflow ตอนบวกยอดรวมด้วยเช่นกัน
    let grand_total_cents: u64 = subtotal_a
        .checked_add(subtotal_b)
        .and_then(|sum| sum.checked_add(subtotal_c))
        .expect("ยอดรวมทั้งหมดเกินขนาดที่ u64 รับได้");

    // พิมพ์รายละเอียดแต่ละสินค้า: แปลงสตางค์กลับเป็นบาท.สตางค์ ด้วยการหาร/มอดสิบ (100 สตางค์ = 1 บาท)
    println!(
        "{name_a:<24} {:>6}.{:02} บาท x {qty_a:>3} ชิ้น = {:>10}.{:02} บาท",
        price_a / 100, price_a % 100, subtotal_a / 100, subtotal_a % 100
    );
    println!(
        "{name_b:<24} {:>6}.{:02} บาท x {qty_b:>3} ชิ้น = {:>10}.{:02} บาท",
        price_b / 100, price_b % 100, subtotal_b / 100, subtotal_b % 100
    );
    println!(
        "{name_c:<24} {:>6}.{:02} บาท x {qty_c:>3} ชิ้น = {:>10}.{:02} บาท",
        price_c / 100, price_c % 100, subtotal_c / 100, subtotal_c % 100
    );

    println!("{}", "-".repeat(68));
    println!("ยอดรวมทั้งหมด: {}.{:02} บาท", grand_total_cents / 100, grand_total_cents % 100);

    // สมมติร้านให้ส่วนลดพิเศษ 5000 บาท (500,000 สตางค์) สำหรับยอดซื้อนี้
    // ใช้ saturating_sub เพื่อไม่ให้ยอดสุทธิติดลบ ถ้าส่วนลดมากกว่ายอดซื้อจริง (เกาะที่ 0)
    const DISCOUNT_CENTS: u64 = 500_000;
    let net_total_cents = grand_total_cents.saturating_sub(DISCOUNT_CENTS);

    println!(
        "หลังหักส่วนลด {}.{:02} บาท เหลือชำระ: {}.{:02} บาท",
        DISCOUNT_CENTS / 100,
        DISCOUNT_CENTS % 100,
        net_total_cents / 100,
        net_total_cents % 100
    );
}
```

ผลลัพธ์เมื่อรัน:

```
เมาส์ไร้สาย                 299.00 บาท x  15 ชิ้น =       4485.00 บาท
คีย์บอร์ดเมคานิคอล          890.00 บาท x   8 ชิ้น =       7120.00 บาท
แผ่นรองเมาส์เกมมิ่ง         150.00 บาท x  40 ชิ้น =       6000.00 บาท
--------------------------------------------------------------------
ยอดรวมทั้งหมด: 17605.00 บาท
หลังหักส่วนลด 5000.00 บาท เหลือชำระ: 12605.00 บาท
```

(หมายเหตุ: การจัดคอลัมน์ด้วย `{:<24}` นับความยาวเป็นจำนวน Unicode scalar value/char ไม่ใช่ความกว้างที่มองเห็นจริงบนจอ —
ข้อความไทยที่มีวรรณยุกต์/สระลอย เช่น "ไร้สาย" นับ char มากกว่าความกว้างที่แสดงจริง ทำให้คอลัมน์ดูเยื้องไม่เท่ากันเล็กน้อย
แม้ค่าตัวเลขทุกตัวจะถูกต้องเป๊ะก็ตาม — รายละเอียดเรื่องนี้เกี่ยวกับ UTF-8 และ `char` ที่เราเรียนในหัวข้อ 3.7 จะเจาะลึกอีกครั้ง
ใน **Part 14**)

**สิ่งที่ตัวอย่างนี้รวมไว้จากทั้งบท**: tuple สำหรับเก็บข้อมูลสินค้าแต่ละตัว (`(&str, u32, u32)`), array ที่ความยาวคงที่
รู้แน่นอน (`[(&str, u32, u32); 3]`), destructuring ผ่าน pattern ใน `let`, การเลือก type `u32`/`u64` อย่างมีเหตุผล
(เก็บสตางค์เป็น integer เพื่อเลี่ยงปัญหา floating point), การ cast ด้วย `as` แบบปลอดภัย (widening `u32` → `u64` ที่ไม่มี
ทางสูญข้อมูล), การใช้ `checked_mul`/`checked_add` เพื่อป้องกัน overflow ในจุดที่คำนวณจริง, และ `saturating_sub` เพื่อ
จัดการส่วนลดแบบไม่ให้ยอดติดลบ — ครบทุกแนวคิดหลักของบทนี้ในโค้ดเดียว โดยยังไม่ต้องพึ่งพา `for` loop, `struct`, หรือ
`Vec<T>` ที่ยังไม่ได้เรียน (เราจะกลับมาเขียนโค้ดแบบนี้ให้กระชับขึ้นมากด้วย `for` loop ใน **Part 4** และด้วย `struct`
ใน **Part 9**)

## กับดักที่พบบ่อย (Common Pitfalls)

**1. ลืมว่า `let` immutable โดย default แล้วพยายาม reassign**

มือใหม่ที่ย้ายมาจาก Python/JavaScript มักเขียนโค้ดแบบนี้โดยอัตโนมัติเพราะความเคยชิน:

```rust
fn main() {
    let total = 0;
    total = total + 10; // ผิดพลาดคลาสสิก
    println!("{total}");
}
```

```
error[E0384]: cannot assign twice to immutable variable `total`
```

**วิธีแก้**: ถ้าตั้งใจให้ค่าเปลี่ยนแปลงได้จริง (เช่นตัวสะสมผลรวมใน loop) ให้เติม `mut`: `let mut total = 0;`
ถ้าไม่ได้ตั้งใจให้เปลี่ยน ให้กลับไปดู logic ว่าทำไมโค้ดพยายาม reassign ทั้ง ๆ ที่ไม่ควร — บางทีนี่คือสัญญาณว่าคุณควรใช้
shadowing แทนหากมันคือการแปลงค่าไปอีกรูปแบบ ไม่ใช่การสะสมค่า

**2. สับสนระหว่าง shadowing กับ mutation แล้วคิดว่า scope เดียวกันคือ "ตัวแปรตัวเดียวกัน" เสมอ**

```rust
fn main() {
    let count = 5;
    {
        let count = count + 100; // นี่คือตัวแปรใหม่ ไม่ใช่การแก้ count เดิม
        println!("ใน block: {count}"); // 105
    }
    println!("นอก block: {count}"); // 5 — มือใหม่หลายคนคาดว่าจะเป็น 105!
}
```

มือใหม่ที่คุ้นกับภาษาที่ scope แบบ function-level (เช่น JavaScript แบบเก่าที่ใช้ `var`) อาจคาดว่าค่า `count` จะเปลี่ยนไป
ถาวรหลังออกจาก block แต่ Rust ใช้ **block-scoping** อย่างเคร่งครัด (คล้าย `let`/`const` ใน JavaScript สมัยใหม่ หรือ
ตัวแปรใน C++/Java) ตัวแปรที่ shadow ไว้ภายใน `{ }` จะมีชีวิตอยู่แค่ใน block นั้นเท่านั้น เมื่อออกจาก block ตัวแปรเดิม
ภายนอกจะกลับมาเป็นค่าตัวเองอีกครั้งเสมอ **วิธีแก้**: ถ้าต้องการให้ค่าที่คำนวณใน block ส่งผลต่อภายนอกจริง ๆ ต้องใช้ `mut`
กับตัวแปรภายนอก และ assign ค่าใหม่ให้มันอย่างชัดเจน หรือให้ block คืนค่าออกมา (block ใน Rust คืนค่าได้ — จะเรียนเรื่องนี้
เพิ่มใน Part 4)

**3. ตัวเลขเกินขอบเขต (overflow) แล้วพฤติกรรมต่างกันระหว่างตอน dev กับตอน deploy จริง**

```rust
fn calculate_bonus(base_score: u8, multiplier: u8) -> u8 {
    base_score * multiplier // ไม่มีการเช็ค overflow เลย
}

fn main() {
    let bonus = calculate_bonus(200, 3); // 200 * 3 = 600 เกินขอบเขต u8 (สูงสุด 255) มาก
    println!("{bonus}");
}
```

ตอนเทสด้วย `cargo run` (debug) โค้ดนี้จะ panic ทันทีด้วย `attempt to multiply with overflow` ทำให้ผู้พัฒนาเจอบั๊กเร็ว
แต่ถ้าใครลืมเทสเคสนี้ตอน dev แล้วสร้าง binary release ไปใช้งานจริงด้วย `cargo build --release` โค้ดเดียวกันจะ **ไม่ panic
เลย** แต่คำนวณผลลัพธ์ผิดเงียบ ๆ (wrap around ได้ค่าที่ไม่ตรงกับความเป็นจริงทางธุรกิจเลย) ซึ่งอันตรายกว่าการ panic มาก
เพราะระบบจะทำงานต่อไปด้วยข้อมูลที่ผิด **วิธีแก้**: ในจุดคำนวณที่มีโอกาสเกินขอบเขตได้จริงจากข้อมูล input (ไม่ใช่แค่ตอน
เทสด้วยเคสสุดโต่ง) ให้ใช้ `checked_mul`/`saturating_mul` และเขียน logic จัดการ error/ขอบเขตอย่างชัดเจนเสมอ อย่าปล่อยให้
operator ธรรมดาตัดสินพฤติกรรมแทนคุณ

**4. ผสม integer type ต่างชนิดกันตรง ๆ โดยไม่ cast**

```rust
fn main() {
    let items_in_cart: usize = 3;
    let discount_percent: i32 = 10;
    let result = items_in_cart + discount_percent; // ผสม usize กับ i32
    println!("{result}");
}
```

```
error[E0308]: mismatched types
  |
  |     let result = items_in_cart + discount_percent;
  |                                  ^^^^^^^^^^^^^^^^^ expected `usize`, found `i32`
```

มือใหม่ที่มาจาก Python/JavaScript (ที่ตัวเลขทุกตัวเป็น type เดียวกันหมด ไม่ต้องคิดเรื่อง type ต่างกัน) จะเจอ error แบบนี้
บ่อยมากในช่วงแรก **วิธีแก้**: ต้อง cast ให้ type ตรงกันก่อนเสมอ เลือกว่าจะ cast ฝั่งไหนขึ้นกับความหมายทางธุรกิจ เช่น
`items_in_cart + discount_percent as usize` (ถ้ามั่นใจว่า `discount_percent` ไม่ติดลบ) หรือใช้ `TryFrom`/`TryInto`
ถ้าต้องการความปลอดภัยกว่านั้น การออกแบบที่ดีคือ **เลือก type ให้เหมาะกับความหมายตั้งแต่แรก** — ในตัวอย่างนี้ อาจจะไม่ควร
บวก "จำนวนสินค้า" กับ "เปอร์เซ็นต์ส่วนลด" ตรง ๆ อยู่แล้วในเชิง logic ด้วยซ้ำ (นี่คือตัวอย่างที่ error ของ type ช่วยจับ
บั๊กทาง logic ได้ตั้งแต่ก่อนรันจริง)

**5. เปรียบเทียบค่า float ด้วย `==` โดยตรง**

```rust
fn main() {
    // สมมติแบ่งจ่าย 3 งวด: งวดละ 0.1, 0.2, 0.3 ของยอดเต็ม ตั้งใจให้รวมกันได้ 0.6 พอดี
    let paid_ratio = 0.1 + 0.2 + 0.3;
    if paid_ratio == 0.6 {
        println!("จ่ายครบตามสัดส่วนที่กำหนด");
    } else {
        println!("สัดส่วนไม่ตรง!"); // นี่คือสิ่งที่จะเกิดขึ้นจริง
    }
}
```

โค้ดนี้ compile ผ่านสบาย ๆ (ไม่มี error ใด ๆ) แต่พิมพ์ "สัดส่วนไม่ตรง!" ออกมาอย่างไม่คาดคิด เพราะการคำนวณ floating point
สะสมความไม่เที่ยงตรงเล็ก ๆ ไว้ (ตามที่อธิบายในหัวข้อ 3.5) ทำให้ `0.1 + 0.2 + 0.3` ไม่เท่ากับ `0.6` เป๊ะ ๆ ในระดับ bit
(ได้ค่าประมาณ `0.6000000000000001` แทน)
**วิธีแก้**: ใช้การเปรียบเทียบแบบ epsilon (`(a - b).abs() < epsilon`) เสมอเมื่อต้องเทียบค่า float สองค่าว่า "เท่ากันหรือไม่"
หรือถ้าเป็นไปได้ให้ออกแบบระบบให้เก็บค่าที่ต้องการความเที่ยงตรงสูง (เช่นเงิน) เป็น integer ตั้งแต่แรกแบบที่ทำในตัวอย่าง
ระบบสินค้าคงคลังของหัวข้อ 3.13

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** โค้ดต่อไปนี้ compile ไม่ผ่าน ให้แก้ไขให้ถูกต้อง **2 วิธีที่แตกต่างกัน** (วิธีที่ 1 ใช้ `mut`, วิธีที่ 2
   ใช้ shadowing) แล้วอธิบายด้วยคำพูดของตัวเอง 2-3 บรรทัดว่าทั้งสองวิธีต่างกันอย่างไรในเชิงความหมาย (ไม่ใช่แค่ syntax):
   ```rust
   fn main() {
       let age = 25;
       age = age + 1;
       println!("อายุปีหน้า: {age}");
   }
   ```
   (hint: วิธีที่ 1 คือเติม `mut` แล้ว `age = age + 1;` ทำงานได้ตรง ๆ ส่วนวิธีที่ 2 คือเปลี่ยนบรรทัดที่สองเป็น
   `let age = age + 1;` — สังเกตว่าทั้งสองวิธีให้ output เหมือนกัน แต่วิธีที่ 2 สร้างตัวแปรใหม่จริง ๆ ในเชิง compiler)

2. **[ง่าย-กลาง]** เขียนโปรแกรมที่มีตัวแปร `let big_number: i64 = 1_000_000;` แล้วทดลองแปลงเป็น `i16` ด้วย 2 วิธี:
   (ก) ด้วย `as` แล้วพิมพ์ผลลัพธ์ที่ได้ออกมาดู (มันจะผิดเพี้ยนไปจากค่าจริงมาก) และ (ข) ด้วย `TryInto` แล้วจัดการผลลัพธ์
   ด้วย `match` พิมพ์ข้อความที่ต่างกันระหว่างกรณีสำเร็จกับล้มเหลว จากนั้นลองเปลี่ยนค่า `big_number` เป็น `500` (ซึ่งพอดี
   กับขอบเขตของ `i16`) แล้วสังเกตว่าผลลัพธ์จากทั้งสองวิธีตรงกันหรือไม่ในกรณีนี้
   (hint: ขอบเขตของ `i16` คือ -32,768 ถึง 32,767 — ค่า `500` ยังอยู่ในขอบเขตนี้พอดี)

3. **[กลาง]** ขยายตัวอย่างระบบสินค้าคงคลังในหัวข้อ 3.13 โดยเพิ่มสินค้าตัวที่ 4 เข้าไปใน array (เปลี่ยนขนาด array จาก
   `[(&str, u32, u32); 3]` เป็น `[(&str, u32, u32); 4]`) และเพิ่มการคำนวณ **ภาษีมูลค่าเพิ่ม 7%** จากยอดรวมก่อนหักส่วนลด
   โดยต้องเลือก type ของตัวเลขที่ใช้คำนวณเปอร์เซ็นต์ให้เหมาะสม (คิดว่าจะใช้ integer หรือ float คำนวณภาษี และทำไม
   — ลองคำนวณด้วย integer arithmetic ล้วน ๆ โดยคูณด้วย 107 แล้วหารด้วย 100 ดูว่าได้ผลลัพธ์สมเหตุสมผลหรือไม่)
   (hint: `let total_with_vat = grand_total_cents.checked_mul(107).and_then(|v| Some(v / 100));` — ลองคิดว่าทำไม
   ต้องคูณก่อนแล้วค่อยหาร ไม่ใช่หารก่อนแล้วค่อยคูณ ในการคำนวณด้วย integer)

4. **[ยาก/ประยุกต์]** จำลองระบบ "บัญชีธนาคารง่าย ๆ" ที่มีตัวแปร `balance_cents: i64` เริ่มต้นที่ 100_000 (1,000 บาท)
   เขียนฟังก์ชัน (หรือโค้ดใน `main` ถ้ายังไม่มั่นใจเรื่องฟังก์ชัน) ที่จำลองการ "ฝากเงิน" และ "ถอนเงิน" หลายครั้งโดยใช้
   `checked_add` สำหรับฝาก และ `checked_sub` สำหรับถอน (เพื่อป้องกันถอนเงินเกินยอดที่มี ซึ่งควรได้ `None` แล้วพิมพ์ข้อความ
   ปฏิเสธการทำรายการ ไม่ใช่ยอมให้ยอดติดลบแบบเงียบ ๆ) จากนั้นตอบคำถามเชิงออกแบบสั้น ๆ (3-5 บรรทัด): **ทำไมตัวอย่างนี้ถึง
   เลือกใช้ `i64` (signed) แทน `u64` (unsigned) สำหรับเก็บยอดเงินในบัญชี ทั้ง ๆ ที่ยอดเงินปกติไม่ควรติดลบ?**
   (hint: คิดถึงสถานการณ์ overdraft หรือการบันทึกรายการที่ยอดกลายเป็นลบชั่วคราวระหว่างการประมวลผลแบบ batch — ถ้าใช้ `u64`
   แล้วมีการคำนวณกลางทางที่ผลลัพธ์ติดลบเพียงชั่วครู่ก่อนจะบวกกลับมาเป็นบวกอีกที จะเกิดอะไรขึ้นกับ `u64` ทันทีที่ค่าติดลบ
   แม้จะแค่ชั่วขณะ?)

## สรุป

ในบทนี้เราได้เจาะลึกหัวใจสำคัญที่สุดอย่างหนึ่งของภาษา Rust: ระบบตัวแปรและชนิดข้อมูล เราเริ่มจากทำความเข้าใจว่าทำไม
`let` ถึงทำให้ตัวแปร **immutable โดย default** (เพื่อความปลอดภัย ความง่ายในการอ่านโค้ด และโอกาสให้ compiler optimize)
และต้องใช้ `mut` อย่างตั้งใจเมื่อต้องการเปลี่ยนค่าจริง ๆ เราแยกแยะความแตกต่างระหว่าง **`mut`** (เปลี่ยนค่าในตัวแปรเดิม
type เดิม) กับ **shadowing** (สร้างตัวแปรใหม่ที่บังตัวเก่า เปลี่ยน type ได้) และรู้จัก `const`/`static` สำหรับค่าคงที่
ระดับ global ที่ต้องคำนวณได้ตอน compile time

เราสำรวจชนิดข้อมูล scalar ทั้ง 4 กลุ่ม (integer หลากหลายขนาดทั้ง signed/unsigned, floating point ตามมาตรฐาน IEEE 754,
boolean ที่เข้มงวดไม่มี truthy/falsy, และ char ที่เป็น Unicode Scalar Value 4 bytes เสมอ) และชนิดข้อมูล compound
สองแบบ (tuple สำหรับรวมค่าต่าง type, array สำหรับเก็บค่า type เดียวกันจำนวนคงที่) พร้อมเข้าใจว่า type inference ของ
Rust ทำงานได้ทรงพลังแค่ไหน และต้องช่วย compiler ด้วย type annotation เมื่อไหร่

ส่วนที่สำคัญที่สุดในเชิงความปลอดภัยของโปรแกรมคือการเข้าใจพฤติกรรม **integer overflow** ที่ต่างกันระหว่าง debug กับ
release build และรู้จักเลือกใช้ `wrapping_*`, `checked_*`, `saturating_*`, `overflowing_*` ให้เหมาะกับความหมายทางธุรกิจ
ของโค้ดแต่ละจุด รวมถึงการแปลง type ตัวเลขอย่างปลอดภัยด้วย `TryFrom`/`TryInto` แทนการใช้ `as` แบบไม่ระมัดระวัง ทั้งหมดนี้
ถูกรวบรวมไว้ในตัวอย่างระบบสินค้าคงคลังท้ายบท ซึ่งแสดงให้เห็นว่าแนวคิดพื้นฐานเหล่านี้ประกอบกันเป็นโค้ดที่ใช้งานได้จริง
ได้อย่างไร

ใน **Part 4** เราจะนำตัวแปรและชนิดข้อมูลที่เรียนมาในบทนี้ไปใช้ควบคุมการทำงานของโปรแกรมผ่าน **ฟังก์ชันและ Control Flow**
— การประกาศฟังก์ชันที่รับ parameter และคืนค่าอย่างเป็นทางการ, เงื่อนไข `if`/`else`, และ loop ทั้งสามแบบ (`loop`, `while`,
`for`) ซึ่งจะทำให้เราเขียนตัวอย่างระบบสินค้าคงคลังในบทนี้ให้กระชับขึ้นได้อย่างเห็นได้ชัด แทนที่จะต้องเขียนโค้ดซ้ำ ๆ
ทีละสินค้าแบบที่ทำในหัวข้อ 3.13

---

**Part ก่อนหน้า:** [Cargo และโครงสร้างโปรเจกต์](part-002-cargo-and-project-structure.md) | **Part ถัดไป:** [ฟังก์ชันและ Control Flow](part-004-functions-and-control-flow.md)
