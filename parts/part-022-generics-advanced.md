# Part 22: Generics ขั้นสูง (trait bounds, where clauses, PhantomData)

> โมดูล: ระดับกลาง (Intermediate) | ระดับ: กลาง | เวลาโดยประมาณ: 220 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- เขียนและอ่าน **`where` clause** ได้อย่างคล่องแคล่วในทุกกรณี ทั้งกรณีที่ใช้แค่เพื่อความอ่านง่าย (เทียบเท่า
  inline bound ทุกประการ) และกรณีที่ `where` เป็น**ทางเดียว**ที่ใช้ได้ เช่นการ bound บน associated type
  หรือบน type expression ที่ไม่ใช่ type parameter เปล่า ๆ (เช่น `Option<T>: PartialOrd`)
- อธิบายและเขียน **trait bound บน `impl` block** ได้อย่างเป็นระบบมากกว่าที่ Part 19 แนะนำไว้สั้น ๆ เข้าใจว่าทำไม
  generic struct ตัวเดียวกัน (`Wrapper<T>`) จึงมี method ต่างกันไปตามชนิดข้อมูลจริงที่ใช้ และอ่าน error
  `E0599` (method not found เพราะ trait bound ไม่ครบ) ได้อย่างถูกต้องแม่นยำ
- ออกแบบ generic type ที่มี **type parameter หลายตัวที่มี bound อิสระจากกัน** ได้ (เช่น
  `Cache<K: Hash + Eq, V: Clone>`) โดยเชื่อมโยงกับข้อกำหนดของ key ใน `HashMap<K, V>` จาก Part 15 ได้อย่างถูกต้อง
- แยกแยะความแตกต่างระหว่าง **associated type** (`type Item;`) กับ **generic type parameter บน trait**
  (`trait Container<Item>`) ได้อย่างชัดเจน อธิบายได้ว่าทำไม `Iterator` ของ standard library เลือกใช้
  associated type และรู้จัก error `E0119` (conflicting implementations) ที่เกิดจากการ implement trait ที่มี
  associated type ให้ type เดียวกันซ้ำสองครั้ง
- ใช้ **`PhantomData<T>`** แก้ปัญหา type parameter ที่ต้องมีไว้เพื่อความปลอดภัยของ type system ตอน compile time
  แต่ไม่มีข้อมูลจริงเก็บอยู่ใน struct เลย พร้อมพิสูจน์ด้วย `std::mem::size_of` ว่ามันไม่กิน memory เพิ่มแม้แต่
  ไบต์เดียว และแก้ error `E0392` ที่ Part 18 ทีเซอร์ไว้ให้เห็นครบทั้งปัญหาและทางแก้
- เขียนและอ่าน **lifetime bound แบบ `T: 'a`** ได้ในระดับที่เพียงพอสำหรับใช้งานจริง และเข้าใจ
  **blanket implementation** ในระดับที่ลึกพอจะรู้ว่าเมื่อไหร่ที่ **orphan rule** จะปฏิเสธมัน พร้อม error จริง
  (`E0210`)

## ความรู้ที่ต้องมีมาก่อน

- **Part 18 (Generics เบื้องต้น)**: บทนี้คือภาคต่อโดยตรงของ Part 18 ทุกหัวข้อในบทนี้สร้างอยู่บนพื้นฐานที่ Part 18
  วางไว้ทั้งหมด — generic function/struct/enum, trait bound แบบ inline (`T: PartialOrd + Copy`),
  monomorphization, const generics เบื้องต้น และที่สำคัญที่สุดคือ**สองเรื่องที่ Part 18 เอ่ยชื่อไว้แต่จงใจไม่
  อธิบายลึก**: หัวข้อ 18.11 แนะนำ `where` clause แบบสั้น ๆ แล้วบอกตรง ๆ ว่า "เรื่องนี้จะกลับมาสำคัญอีกครั้งอย่าง
  เป็นทางการใน Part 22" และกับดักข้อ 3 ของ Part 18 (error `E0392`) บอกว่า `PhantomData` "เป็นเทคนิคขั้นสูงกว่าที่
  จะเจาะลึกใน Part 22" — บทนี้คือจุดที่เราเฉลยทั้งสองเรื่องอย่างเต็มรูปแบบ ถ้าคุณจำหัวข้อเหล่านี้จาก Part 18 ไม่
  ชัดแล้ว แนะนำให้ย้อนไปทวนก่อนอ่านบทนี้
- **Part 15 (HashMap, HashSet)**: เราจะย้อนกลับไปใช้ข้อกำหนดที่ Part 15 วางไว้ว่า key ของ `HashMap<K, V>` ต้อง
  implement ทั้ง `Hash` และ `Eq` เพื่อออกแบบ `Cache<K, V>` ของเราเองในหัวข้อ 22.5 — ถ้าลืมว่าทำไม key ต้องมีสอง
  trait นี้ ควรย้อนทวน Part 15 หัวข้อการสร้าง `HashMap` ก่อน
- **Part 19 (Traits เบื้องต้น)**: เราจะขยาย **conditional method implementation**
  (`impl<T: Bound> Type<T> { ... }`) และ **blanket implementation** (`impl<T: Display> ToString for T`) ที่
  Part 19 แนะนำไว้ในฐานะ "รูปแบบขั้นสูงของ `impl` block" ให้ลึกและเป็นระบบมากขึ้นในบทนี้ รวมถึงต้องเข้าใจว่า
  trait คืออะไร, `impl Trait for Type` ทำงานอย่างไร มาก่อน
- **Part 20 (Lifetimes เบื้องต้น)**: จำเป็นสำหรับหัวข้อ 22.8 ที่ผสม trait bound กับ lifetime เข้าด้วยกัน
  (`T: Display + 'a`) — คุณต้องเข้าใจว่า lifetime คืออะไร และ `&'a T` หมายความว่าอะไรมาก่อน
- **Part 21 (Traits ขั้นสูง)**: บทนี้จะ**ไม่**พูดถึง `dyn Trait`/trait object เลย เพราะเป็นเนื้อหาหลักของ
  Part 21 แต่จะอ้างถึงเป็นทางเลือกเปรียบเทียบสั้น ๆ ในบางจุดที่เกี่ยวข้อง (เช่นตอนพูดถึง object safety ของ
  associated type) — ถ้ายังไม่ได้อ่าน Part 21 ไม่ต้องกังวล เพราะบทนี้ไม่ได้พึ่งพาเนื้อหานั้นโดยตรง

## เนื้อหา

### 22.1 ทวนจาก Part 18: สองเรื่องที่ถูก "แขวน" ไว้รอบทนี้

ก่อนเริ่มเนื้อหาใหม่ ลองย้อนกลับไปดูสองจุดที่ Part 18 จงใจเปิดค้างไว้ให้บทนี้มาปิด เพราะมันคือกรอบความคิดที่ทำให้
เนื้อหาทั้งบทนี้เชื่อมกันเป็นเรื่องเดียว ไม่ใช่หัวข้อแยกกันสามหัวข้อที่ไม่เกี่ยวข้องกัน

**เรื่องที่หนึ่ง — `where` clause**: หัวข้อ 18.11 แสดงให้เห็นว่า

```rust
fn describe_and_compare<T: PartialOrd + Copy + Display>(a: T, b: T) -> String { /* ... */ }
```

เขียนเป็นอีกแบบได้โดยความหมายเหมือนกันทุกประการ:

```rust
fn describe_and_compare<T>(a: T, b: T) -> String
where
    T: PartialOrd + Copy + Display,
{
    /* ... */
}
```

Part 18 อธิบายแค่ว่า "ต่างกันด้านความอ่านง่ายเมื่อ bound ยาวขึ้น" ซึ่ง**ถูกแล้ว แต่ไม่ครบ** — มันไม่ได้บอกว่ามี
**สถานการณ์ที่การเขียน bound แบบ inline ทำไม่ได้เลยในทางไวยากรณ์** ต้องใช้ `where` เท่านั้น ซึ่งเราจะเห็นตัวอย่าง
จริงที่จับต้องได้ในหัวข้อ 22.3

**เรื่องที่สอง — `PhantomData<T>`**: กับดักข้อ 3 ของ Part 18 แสดง error นี้:

```
error[E0392]: type parameter `T` is never used
 --> src/main.rs:1:16
  |
1 | struct Wrapper<T> {
  |                ^ unused type parameter
  |
  = help: consider removing `T`, referring to it in a field, or using a marker such as `PhantomData`
```

Part 18 บอกแค่ว่า "ถ้าตั้งใจให้ `T` เป็นแค่ป้ายกำกับประเภทโดยไม่มีข้อมูลเก็บอยู่จริง จะต้องใช้ `PhantomData`" แต่
ไม่ได้อธิบายว่า **ทำไมสถานการณ์แบบนั้นถึงมีความหมายในทางวิศวกรรมเลย** — ถ้า `T` ไม่ถูกใช้เก็บข้อมูลอะไรสักไบต์
เดียว แล้วมันมีประโยชน์อะไร? คำตอบคือ **ประโยชน์ทั้งหมดอยู่ที่ compile time ไม่ใช่ runtime** ซึ่งเราจะเห็นตัวอย่าง
ที่ใช้จริงในหัวข้อ 22.7

ทั้งสองเรื่องนี้ดูเหมือนไม่เกี่ยวข้องกัน แต่จริง ๆ แล้วมันคือ**สองด้านของเหรียญเดียวกัน**: `where` clause ทำให้เรา
**ประกาศเงื่อนไขที่ซับซ้อนกว่าที่ inline bound ทำได้** ส่วน `PhantomData` ทำให้เรา**ใช้ type parameter เพื่อสื่อ
ความหมายและความปลอดภัยของ type ล้วน ๆ โดยไม่ต้องแบกข้อมูลจริง** — ทั้งสองเรื่องคือการ**ขยายขอบเขตของสิ่งที่
generics ทำได้** ให้ไกลกว่า "แทนชนิดข้อมูลที่เก็บอยู่ใน field" แบบที่ Part 18 สอนไว้เป็นจุดเริ่มต้น

### 22.2 `where` clause เชิงลึก: จาก inline bound สู่ syntax ที่เทียบเท่ากันทุกประการ

มาเริ่มจากสถานการณ์ที่สมจริงกว่าตัวอย่างเดียวใน Part 18: ฟังก์ชันที่มี **type parameter สองตัว** ซึ่งแต่ละตัวมี
bound หลายเงื่อนไข ลองเขียนฟังก์ชันที่เทียบค่าสองค่าชนิดเดียวกัน (`T`) แล้วพิมพ์ผลพร้อม label ที่เป็นชนิดข้อมูล
อื่น (`U`) ออกมา:

```rust
use std::fmt::{Debug, Display};

fn process_pair<T: Clone + Debug + PartialOrd, U: Display + Default>(a: T, b: T, label: U) -> T {
    let winner = if a > b { a.clone() } else { b.clone() };
    println!("[{label}] a={a:?}, b={b:?} -> winner={winner:?}");
    winner
}

fn main() {
    let w = process_pair(10, 20, "รอบที่ 1");
    println!("winner = {w}");
}
```

ผลลัพธ์:

```
[รอบที่ 1] a=10, b=20 -> winner=20
winner = 20
```

**อธิบายโค้ดทีละส่วน:**

- **`T: Clone + Debug + PartialOrd`** — `T` ต้องคัดลอกได้ (`Clone`, เพราะเราต้อง `.clone()` ค่าที่ชนะออกมาโดย
  ไม่ทำลายต้นฉบับ), พิมพ์แบบ debug ได้ (`Debug`, สำหรับ `{a:?}`), และเปรียบเทียบลำดับได้ (`PartialOrd`, สำหรับ
  `a > b`) — สามเงื่อนไขที่ **ไม่เกี่ยวข้องกันเลยในเชิงความหมาย** แต่ทั้งสามจำเป็นสำหรับให้ฟังก์ชันนี้ compile
  ผ่านและทำงานถูกต้อง
- **`U: Display + Default`** — `U` ต้องพิมพ์แบบปกติได้ (`Display`, สำหรับ `{label}`) และมีค่าเริ่มต้นได้
  (`Default`) — bound `Default` ในตัวอย่างนี้ยังไม่ได้ใช้งานจริงในเนื้อฟังก์ชัน แต่ตั้งใจใส่ไว้เพื่อจำลอง
  สถานการณ์ที่ signature มี bound สะสมมากขึ้นเรื่อย ๆ ตามที่ requirement ของฟังก์ชันขยายตัว (สถานการณ์ที่เกิดขึ้น
  จริงมากในโค้ดที่ดูแลมานาน)

สังเกตว่า signature บรรทัดแรก (`fn process_pair<T: Clone + Debug + PartialOrd, U: Display + Default>(a: T, b: T, label: U) -> T`)
มีความยาวเกิน 90 ตัวอักษรไปแล้ว และการอ่านว่า "ฟังก์ชันนี้รับอะไร คืนอะไร" ต้องปนกับการอ่านเงื่อนไขของ `T` และ
`U` ไปพร้อมกันในบรรทัดเดียว ยิ่งมี type parameter เพิ่มขึ้นหรือ bound เพิ่มขึ้น ความยากในการอ่านจะเพิ่มแบบไม่เป็น
เชิงเส้น (ไม่ใช่แค่ "ยาวขึ้นเป็นเส้นตรง" แต่ "อ่านยากขึ้นแบบทวีคูณ" เพราะสายตาต้องแยกแยะว่า bound ไหนเป็นของ
parameter ไหน)

มาเขียนฟังก์ชันเดียวกันนี้ด้วย **`where` clause**:

```rust
use std::fmt::{Debug, Display};

fn process_pair<T, U>(a: T, b: T, label: U) -> T
where
    T: Clone + Debug + PartialOrd,
    U: Display + Default,
{
    let winner = if a > b { a.clone() } else { b.clone() };
    println!("[{label}] a={a:?}, b={b:?} -> winner={winner:?}");
    winner
}

fn main() {
    let w = process_pair(10, 20, "รอบที่ 1");
    println!("winner = {w}");
}
```

ผลลัพธ์ (เหมือนกันทุกตัวอักษรกับเวอร์ชันก่อนหน้า เพราะเป็นฟังก์ชันเดียวกันทุกประการในสายตา compiler):

```
[รอบที่ 1] a=10, b=20 -> winner=20
winner = 20
```

**สิ่งที่เปลี่ยนไปมีแค่การจัดวาง ไม่มีอะไรเปลี่ยนด้าน behavior เลย**: `fn process_pair<T, U>(a: T, b: T, label: U) -> T`
บรรทัดแรกตอนนี้เหลือแค่ **"ฟังก์ชันนี้ชื่ออะไร รับ parameter อะไร คืนอะไร"** — อ่านจบในสายตาเดียวได้ทันทีโดยไม่
ต้องสนใจเงื่อนไขของ `T`/`U` เลย จากนั้น `where` clause ที่ตามมาแยกรายการเงื่อนไขออกเป็น**คนละบรรทัดต่อ type
parameter หนึ่งตัว** ทำให้อ่านง่ายขึ้นทันทีว่า "`T` ต้องมีอะไรบ้าง" และ "`U` ต้องมีอะไรบ้าง" แยกจากกันชัดเจน โดย
ไม่ต้องนับ `+` ทีละตัวปนกันในบรรทัดเดียว

ประเด็นสำคัญที่สุดที่ต้องเข้าใจให้แน่นคือ: **`fn f<T: A + B>(...)` และ `fn f<T>(...) where T: A + B { ... }`
compiler มองเป็นสิ่งเดียวกันเป๊ะ** ไม่มีความแตกต่างด้าน type-checking, ด้าน monomorphization (ตามหลักการจาก
Part 18 หัวข้อ 18.7), หรือด้านประสิทธิภาพตอน runtime เลยแม้แต่นิดเดียว — `where` clause **ไม่ใช่ feature ใหม่ที่
ทำอะไรเพิ่มเติมได้มากกว่า inline bound** มันเป็นแค่ **ตำแหน่งอื่นที่เขียนข้อมูลเดียวกันได้** (ยกเว้นกรณีพิเศษที่
เราจะเห็นในหัวข้อถัดไป ซึ่ง `where` ทำได้มากกว่าจริง ๆ ในเชิงไวยากรณ์)

#### กฎการจัดวาง: `where` มาหลังชนิดข้อมูลที่คืนค่า ก่อนวงเล็บปีกกาเปิด

สังเกต pattern ที่ตายตัวของ `where` clause: มันอยู่**หลัง** return type (`-> T`) และ**ก่อน** วงเล็บปีกกาเปิดของ
ฟังก์ชัน (`{`) เสมอ ไม่ว่าฟังก์ชันจะซับซ้อนแค่ไหน:

```rust
fn some_function<T, U>(param1: T, param2: U) -> bool
where
    T: SomeTrait,
    U: AnotherTrait,
{
    // เนื้อฟังก์ชัน
    true
}

trait SomeTrait {}
trait AnotherTrait {}
```

รูปแบบนี้ใช้ได้กับ `struct`, `enum`, `impl` block เช่นกัน (เราจะเห็นตัวอย่างการใช้กับ `impl` block ในหัวขัดถัด ๆ
ไป) — ธรรมเนียมนิยม (convention) ที่ `rustfmt` (จาก Part 5) จัดรูปแบบให้อัตโนมัติคือ **แต่ละ bound ขึ้นบรรทัดใหม่
คั่นด้วยจุลภาค** เมื่อมีมากกว่าหนึ่ง trait bound ในกลุ่ม `where` เพื่อให้ไล่อ่านได้ทีละบรรทัด

### 22.3 เมื่อ `where` ไม่ใช่แค่สไตล์ แต่เป็น "ทางเดียว" ที่ใช้ได้

ตอนนี้มาถึงจุดที่สำคัญที่สุดของหัวข้อนี้ — สถานการณ์ที่การเขียน bound แบบ inline (ในวงเล็บมุมหลังชื่อ type
parameter) **ทำไม่ได้เลยในทางไวยากรณ์ของภาษา** ไม่ใช่เรื่องของความชอบส่วนตัวหรือความอ่านง่ายอีกต่อไป

ลองพิจารณาฟังก์ชันที่เปรียบเทียบค่าสองตัวที่ห่อด้วย `Option<T>` (จาก Part 11) — ต้องการหาว่าตัวไหน "มากกว่า" กัน
ระหว่างสองค่าที่อาจว่างเปล่าได้:

```rust
fn max_option<T>(a: Option<T>, b: Option<T>) -> Option<T>
where
    Option<T>: PartialOrd,
{
    if a > b {
        a
    } else {
        b
    }
}

fn main() {
    let a = Some(5);
    let b = Some(9);
    println!("{:?}", max_option(a, b));
}
```

ผลลัพธ์:

```
Some(9)
```

**อธิบายจุดสำคัญของ bound นี้**: สังเกตว่า bound ที่เราต้องการคือ **`Option<T>: PartialOrd`** — ไม่ใช่
`T: PartialOrd` (แม้ในทางปฏิบัติ `Option<T>` จะ implement `PartialOrd` ก็ต่อเมื่อ `T: PartialOrd` เท่านั้น เพราะ
standard library เขียน blanket implementation ไว้แบบนั้น แต่**สิ่งที่เราต้องการประกาศคือ bound บน `Option<T>`
โดยตรง** ไม่ใช่บน `T` เปล่า ๆ) นี่คือความแตกต่างเชิงไวยากรณ์ที่สำคัญมาก: `Option<T>` เป็น **type expression ที่
สร้างจาก `T`** ไม่ใช่ **type parameter ที่ถูกประกาศไว้เอง** และ syntax inline bound (`<T: Trait>`) รองรับเฉพาะ
การแนบ bound ให้กับ**ชื่อ type parameter ที่ประกาศอยู่ในวงเล็บมุมเดียวกันนั้น**เท่านั้น

ลองพิสูจน์ว่า inline bound แบบนี้ทำไม่ได้จริง ๆ ด้วยการพยายามเขียนมันดู:

```rust
fn max_option<T, Option<T>: PartialOrd>(a: Option<T>, b: Option<T>) -> Option<T> {
    if a > b { a } else { b }
}

fn main() {}
```

```
error: expected one of `,`, `:`, `=`, or `>`, found `<`
 --> src/main.rs:1:24
  |
1 | fn max_option<T, Option<T>: PartialOrd>(a: Option<T>, b: Option<T>) -> Option<T> {
  |                        ^ expected one of `,`, `:`, `=`, or `>`
```

นี่ไม่ใช่ error ที่เกี่ยวกับ type-checking เลย (ไม่มีเลขรหัส `E____` เพราะมันเป็น **syntax error** — parser
ปฏิเสธตั้งแต่ยังไม่ทันวิเคราะห์ type ด้วยซ้ำ) — parser ของ Rust เห็นคำว่า `Option` ตรงตำแหน่งที่ **คาดหวังชื่อ
type parameter ใหม่** (หลังจุลภาคที่ตามหลัง `T`) แล้วงงว่าทำไมชื่อนั้นมี `<T>` ต่อท้ายอีก เพราะในตำแหน่งนั้น
ไวยากรณ์อนุญาตแค่ **"ชื่อตัวแปรเปล่า ๆ หนึ่งชื่อ"** เท่านั้น (เหมือนการประกาศ parameter ของฟังก์ชันธรรมดาที่รับ
แค่ชื่อ ไม่รับ expression ซับซ้อน) วงเล็บมุมหลังชื่อฟังก์ชันคือ**ที่สำหรับประกาศตัวแปรชนิดข้อมูลใหม่**เท่านั้น
ไม่ใช่ที่สำหรับเขียนเงื่อนไขบนชนิดข้อมูลที่ประกอบขึ้นจากตัวแปรเหล่านั้น

ในทางกลับกัน `where` clause **ไม่มีข้อจำกัดแบบนี้เลย** — มันอนุญาตให้เขียน **"type อะไรก็ได้ (ไม่จำเป็นต้องเป็น
type parameter เปล่า ๆ) ต้อง implement trait อะไร"** ทำให้ `where Option<T>: PartialOrd` ถูกต้องตามไวยากรณ์
สมบูรณ์ นี่คือสิ่งที่ Part 18 หัวข้อ 18.11 ไม่ได้บอกไว้: **`where` ไม่ใช่แค่ "ย้ายตำแหน่งเดียวกันไปเขียนที่อื่น"
แต่มันมี "อำนาจในการแสดงออก" (expressive power) ที่มากกว่า inline bound จริง ๆ** — inline bound เป็น**เซตย่อย**
ของสิ่งที่ `where` ทำได้ ไม่ใช่คนละความหมายที่ทับซ้อนกันพอดี

#### กรณีที่พบบ่อยกว่า: bound บน associated type

อีกสถานการณ์หนึ่งที่พบบ่อยมากในโค้ด Rust จริง คือการ bound บน **associated type** ของ trait อื่น (เราจะเรียน
เรื่อง associated type อย่างเต็มรูปแบบในหัวข้อ 22.6 — ตอนนี้ขอให้เข้าใจแค่ว่ามันคือ "ชนิดข้อมูลที่ผูกอยู่กับ
trait หนึ่ง ๆ" พอก่อน) ตัวอย่างคลาสสิกที่สุดคือการรับ `Iterator` (ที่เรียนเบื้องต้นอย่างเป็นทางการใน Part 25)
แล้วต้องการให้ค่าที่ iterate ออกมา (`Item`) พิมพ์ได้:

```rust
use std::fmt::Display;

fn print_all<I>(iter: I)
where
    I: Iterator,
    I::Item: Display,
{
    for item in iter {
        println!("- {item}");
    }
}

fn main() {
    print_all(vec![1, 2, 3].into_iter());
    print_all(vec!["สวัสดี", "โลก"].into_iter());
}
```

ผลลัพธ์:

```
- 1
- 2
- 3
- สวัสดี
- โลก
```

**อธิบายทำไมต้องใช้ `where` ที่นี่**: `I::Item` คือ**ชื่อของ associated type** ที่ผูกอยู่กับ `I` ผ่าน trait
`Iterator` — มันคือ "path" (เส้นทางเข้าถึง type) ไม่ใช่ชื่อ type parameter เปล่า ๆ เหมือน `T` แนวเดียวกับที่
`Option<T>` ในตัวอย่างก่อนหน้าไม่ใช่ type parameter เปล่า ๆ การเขียน `I::Item: Display` จึงต้องอยู่ใน `where`
clause (นอกเหนือจากรูปแบบพิเศษเฉพาะที่ Rust รุ่นใหม่เพิ่มมาให้เขียน `I: Iterator<Item: Display>` แบบ inline ได้
สำหรับ associated type ที่เป็น**ทางตรง**ของ trait นั้น ๆ เท่านั้น — แต่รูปแบบนี้ไม่ใช่ทุกคนที่คุ้นเคย และไม่
ครอบคลุมทุกกรณี เช่น associated type ที่ซ้อนกันหลายชั้น หรือ bound ที่เชื่อมโยงหลาย type parameter เข้าด้วยกัน)
**`where` clause จึงเป็นรูปแบบที่ครอบคลุมที่สุด อ่านง่ายที่สุด และเป็นที่นิยมมากที่สุดในโค้ดจริง** สำหรับการ
bound บน associated type ไม่ว่า Rust รุ่นที่ใช้จะรองรับ syntax ทางลัดแบบไหนเพิ่มมาก็ตาม

สรุปหลักการของหัวข้อนี้ให้ชัด: **ใช้ inline bound เมื่อ bound นั้นแนบกับ type parameter เปล่า ๆ ตรง ๆ และมีไม่
กี่ตัว (เพื่อความกระชับ) และเปลี่ยนไปใช้ `where` เมื่อ (1) bound เยอะจนอ่านยาก, หรือ (2) ต้อง bound บน type
expression ที่ไม่ใช่ type parameter เปล่า ๆ (เช่น `Option<T>`, `Vec<T>`, หรือ associated type อย่าง `I::Item`)
ซึ่งกรณีที่สองนี้ **ไม่มีทางเลือกอื่นเลยนอกจาก `where`**

### 22.4 Trait bound บน `impl` block: conditional method อย่างเป็นระบบ

Part 19 แนะนำแนวคิดนี้ไว้สั้น ๆ ในฐานะ "รูปแบบขั้นสูงของ `impl` block" — บทนี้จะขยายให้เห็นภาพครบทุกมุม รวมถึง
error จริงที่เกิดขึ้นเมื่อละเมิดเงื่อนไข

หลักการคือ: คุณเขียน generic struct ตัวหนึ่งขึ้นมา แล้วเขียน `impl` block ที่ผูก bound เข้ากับ type parameter
ของมัน — method ที่อยู่ใน `impl` block นั้น**จะมีอยู่จริงก็ต่อเมื่อ**ชนิดข้อมูลที่ใส่ไปตรงกับ bound เท่านั้น
ลองดูตัวอย่างที่ชัดเจน:

```rust
use std::fmt::Display;

struct Wrapper<T> {
    value: T,
}

impl<T: Display> Wrapper<T> {
    fn show(&self) {
        println!("value = {}", self.value);
    }
}

fn main() {
    let w = Wrapper { value: 42 };
    w.show();
}
```

ผลลัพธ์:

```
value = 42
```

**อธิบายโค้ดทีละส่วน**: `struct Wrapper<T> { value: T }` เป็น generic struct ธรรมดาที่ **ไม่มีเงื่อนไขอะไรเลย**
กับ `T` — คุณสร้าง `Wrapper<i32>`, `Wrapper<String>`, `Wrapper<NoDisplay>` (struct อะไรก็ได้ในโลกนี้) ได้หมด
ไม่มี compiler ปฏิเสธ แต่ `impl<T: Display> Wrapper<T>` คือการบอกว่า **"method `show` มีอยู่จริงก็ต่อเมื่อ `T`
implement `Display`"** — สิ่งนี้ทำให้ `Wrapper<T>` มีลักษณะพิเศษที่ภาษาที่ไม่มี generics แบบ static (เช่น Python)
ทำไม่ได้เลย คือ **struct ตัวเดียวกัน มีหน้าตา (method ที่เรียกได้) แตกต่างกันไปตามชนิดข้อมูลที่ใส่จริง**

ลองดูว่าเกิดอะไรขึ้นเมื่อพยายามเรียก `.show()` กับ `Wrapper<T>` ที่ `T` **ไม่** implement `Display`:

```rust
use std::fmt::Display;

struct Wrapper<T> {
    value: T,
}

impl<T: Display> Wrapper<T> {
    fn show(&self) {
        println!("value = {}", self.value);
    }
}

struct NoDisplay;

fn main() {
    let w = Wrapper { value: 42 };
    w.show();

    let w2 = Wrapper { value: NoDisplay };
    w2.show();
}
```

```
error[E0599]: the method `show` exists for struct `Wrapper<NoDisplay>`, but its trait bounds were not satisfied
  --> src/main.rs:20:8
   |
 3 | struct Wrapper<T> {
   | ----------------- method `show` not found for this struct
...
13 | struct NoDisplay;
   | ---------------- doesn't satisfy `NoDisplay: std::fmt::Display`
...
20 |     w2.show();
   |        ^^^^ method cannot be called on `Wrapper<NoDisplay>` due to unsatisfied trait bounds
   |
note: trait bound `NoDisplay: std::fmt::Display` was not satisfied
  --> src/main.rs:7:9
   |
 7 | impl<T: Display> Wrapper<T> {
   |         ^^^^^^^  ----------
   |         |
   |         unsatisfied trait bound introduced here
```

**อ่าน error นี้ให้เข้าใจลึกจริง ๆ**: ข้อความที่สำคัญที่สุดคือ **"the method `show` exists for struct
`Wrapper<NoDisplay>`, but its trait bounds were not satisfied"** — สังเกตคำว่า **"exists"** compiler กำลังบอก
เราว่า **method `show` มีอยู่จริงในซอร์สโค้ด (ไม่ได้พิมพ์ผิดชื่อ ไม่ได้ลืมเขียน)** เพียงแต่ `Wrapper<NoDisplay>`
โดยเฉพาะ**ไม่ผ่านเงื่อนไข**ที่จะเรียกมันได้ — นี่ต่างจาก error `E0599` แบบทั่วไปที่บอกว่า "ไม่มี method ชื่อนี้
เลย" (ซึ่งมักเกิดจากพิมพ์ชื่อผิดหรือลืม `use` trait — Part 19 มีตัวอย่างแบบนั้น) กรณีนี้ **method มีอยู่จริง
แต่ถูก "ล็อก" ไว้ด้วย trait bound ของ `impl` block** ที่ `Wrapper<NoDisplay>` ไม่ผ่านเงื่อนไข

ทำไม compiler ถึงต้องออกแบบมาแบบนี้? ลองนึกภาพว่าถ้า Rust อนุญาตให้เรียก `.show()` กับ `Wrapper<NoDisplay>` ได้
โดยไม่มีการเช็ค bound — โปรแกรมจะพยายาม compile บรรทัด `println!("value = {}", self.value)` โดยที่ `self.value`
เป็นชนิด `NoDisplay` ซึ่ง**ไม่มี implementation ของ `{}` formatting เลย** จะเกิด error ที่จุดอื่นในโค้ด (ข้างใน
`show`) ซึ่งสร้างความสับสนมากกว่า เพราะดูเหมือน `show` เขียนผิด ทั้งที่จริง ๆ ตัว `Wrapper<NoDisplay>` เองไม่มี
สิทธิ์เรียก `show` ตั้งแต่แรก — การรายงาน error ที่ "จุดเรียกใช้" (`w2.show()`) แทนที่จะรายงานที่ "จุดนิยาม"
(ข้างใน `show`) ทำให้ debug ง่ายกว่ามาก เพราะชี้ตรงไปที่สาเหตุจริง

**วิธีแก้**: มีสองทางหลัก ทางแรกคือทำให้ `NoDisplay` implement `Display` เอง (ถ้าสมเหตุสมผลกับความหมายของมัน):

```rust
use std::fmt;

struct NoDisplay;

impl fmt::Display for NoDisplay {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "NoDisplay")
    }
}
```

ทางที่สองคือยอมรับว่า `Wrapper<NoDisplay>` **ไม่ควรมี** `.show()` ตั้งแต่ต้น (เพราะไม่มีทางพิมพ์ค่าที่พิมพ์ไม่ได้)
แล้วออกแบบ method อื่นที่ไม่ต้องพึ่ง `Display` แทน (เช่นถ้าต้องการ debug อย่างเดียว ใช้ bound `Debug` แทน `Display`
ซึ่งเป็น trait ที่ derive ได้ง่ายกว่ามาก)

#### `impl` block หลายแบบซ้อนกันสำหรับ struct เดียวกัน

หลักการเดียวกับ Part 18 หัวข้อ 18.3 (ที่มี `impl<T> Point<T>`, `impl<T: Display> Point<T>`, และ
`impl Point<f64>` สามแบบซ้อนกัน) ใช้ได้กับ bound ที่ซับซ้อนกว่านี้ด้วย ลองดูตัวอย่างที่มี `impl` block สามระดับ
ความเข้มงวดต่างกัน:

```rust
use std::fmt::{Debug, Display};

struct Wrapper<T> {
    value: T,
}

// ระดับ 1: ไม่มีเงื่อนไขเลย ใช้ได้กับ T ทุกชนิด
impl<T> Wrapper<T> {
    fn new(value: T) -> Self {
        Wrapper { value }
    }
}

// ระดับ 2: ต้องการ Debug เท่านั้น (bound ที่ derive ได้ง่าย)
impl<T: Debug> Wrapper<T> {
    fn inspect(&self) {
        println!("[debug] {:?}", self.value);
    }
}

// ระดับ 3: ต้องการทั้ง Display และ Debug (bound เข้มงวดกว่า)
impl<T: Display + Debug> Wrapper<T> {
    fn show_both(&self) {
        println!("display = {}, debug = {:?}", self.value, self.value);
    }
}

fn main() {
    let w = Wrapper::new(42);
    w.inspect();
    w.show_both();
}
```

ผลลัพธ์:

```
[debug] 42
display = 42, debug = 42
```

`Wrapper<T>` ตัวเดียวมี method ที่ "เปิดใช้งาน" ไม่เท่ากันตาม `T` — ถ้า `T` implement แค่ `Debug` (ไม่ implement
`Display`) จะเรียกได้แค่ `new()` และ `inspect()` แต่เรียก `show_both()` ไม่ได้ (จะได้ `E0599` แบบเดียวกับที่เห็น
ไปแล้ว) — นี่คือรูปแบบที่พบเจอบ่อยมากในโค้ด Rust ระดับกลาง-สูง: **ออกแบบ method พื้นฐานให้ bound หลวมที่สุดที่
เป็นไปได้ แล้วค่อยเพิ่ม method พิเศษที่ต้องการ bound เข้มงวดกว่าใน `impl` block แยก** ทำให้ผู้ใช้ type ของคุณ
ได้รับความสามารถมากที่สุดที่ `T` ของเขารองรับได้จริง โดยไม่ถูกบังคับให้ต้อง implement trait ที่ไม่จำเป็นสำหรับ
การใช้งานพื้นฐาน

### 22.5 Type parameter หลายตัวที่มี bound อิสระจากกัน: ออกแบบ `Cache<K, V>`

Part 18 หัวข้อ 18.4 สอน `Point<T, U>` ที่ `T` และ `U` ไม่มีเงื่อนไขอะไรเลย เป็นแค่ "อิสระจากกันในเรื่องชนิด
ข้อมูล" ตอนนี้เรามาผสมกับ trait bound: type parameter หลายตัวที่**แต่ละตัวต้องการ bound คนละแบบ**ตามหน้าที่ของ
มันในโครงสร้างข้อมูล — ตัวอย่างที่สมจริงที่สุดคือ **cache แบบง่าย** ที่ใช้ `HashMap<K, V>` (จาก Part 15) เป็น
โครงสร้างเก็บข้อมูลเบื้องหลัง:

```rust
use std::collections::HashMap;
use std::hash::Hash;

struct Cache<K, V> {
    store: HashMap<K, V>,
    capacity: usize,
}

impl<K, V> Cache<K, V>
where
    K: Hash + Eq + Clone,
    V: Clone,
{
    fn new(capacity: usize) -> Self {
        Cache {
            store: HashMap::new(),
            capacity,
        }
    }

    fn put(&mut self, key: K, value: V) {
        if !self.store.contains_key(&key) && self.store.len() >= self.capacity {
            if let Some(first_key) = self.store.keys().next().cloned() {
                self.store.remove(&first_key);
            }
        }
        self.store.insert(key, value);
    }

    fn get(&self, key: &K) -> Option<V> {
        self.store.get(key).cloned()
    }

    fn len(&self) -> usize {
        self.store.len()
    }
}

fn main() {
    let mut cache: Cache<String, i32> = Cache::new(2);
    cache.put(String::from("a"), 1);
    cache.put(String::from("b"), 2);
    println!("get a = {:?}", cache.get(&String::from("a")));

    cache.put(String::from("c"), 3); // เกิน capacity ต้อง evict ตัวหนึ่งออก
    println!("len after eviction = {}", cache.len());
}
```

ผลลัพธ์:

```
get a = Some(1)
len after eviction = 2
```

**อธิบายว่าทำไม `K` และ `V` ต้องการ bound คนละชุด**:

- **`K: Hash + Eq + Clone`** — นี่คือ**ข้อกำหนดเดียวกันเป๊ะ**กับที่ Part 15 สอนไว้ว่า key ของ `HashMap<K, V>`
  ต้องมี: `Hash` (เพื่อคำนวณ hash value สำหรับหาตำแหน่งเก็บข้อมูล — ตามหลักการจากหัวข้อ 15.1 เรื่อง hash-based
  collection) และ `Eq` (เพื่อเทียบว่าสอง key นี้ "เหมือนกันจริง ๆ" หรือแค่ hash ชนกันโดยบังเอิญ — ปัญหา hash
  collision ที่ทุก hash-based collection ต้องจัดการ) ส่วน `Clone` เป็นข้อกำหนดเพิ่มเติมที่**เราเองเป็นคนใส่**
  เพราะ method `put` ต้องการ `.cloned()` key ตัวแรกออกมาก่อนจะ `.remove()` มัน (ทำแบบนี้เพื่อเลี่ยงปัญหา borrow
  checker: `self.store.keys().next()` คืน reference ที่ยืม `self.store` อยู่ เราไม่สามารถ `.remove()` ขณะที่ยัง
  ยืมอยู่ได้ ตามหลักการ borrowing จาก Part 7 — ต้อง `.cloned()` ออกมาเป็นค่าใหม่ที่ไม่ผูกกับ borrow เดิมก่อน)
- **`V: Clone`** — ต่างจาก `K` โดยสิ้นเชิง เพราะ `V` **ไม่จำเป็นต้องเป็น key ของอะไรเลย** มันแค่ต้องคัดลอกได้
  เพื่อให้ method `get` คืนค่าเป็นเจ้าของใหม่ (`Option<V>`) แทนการคืน reference (`Option<&V>`) — การออกแบบแบบนี้
  ทำให้ผู้ใช้ `Cache` ไม่ต้องยุ่งกับ lifetime ของ reference ที่ผูกกับตัว cache เอง (ใช้งานง่ายกว่าในโค้ดเรียกใช้
  จริง แลกกับต้นทุนการ clone ทุกครั้งที่ `get` — trade-off ที่สมเหตุสมผลสำหรับ cache ขนาดเล็กถึงกลาง)

**จุดที่สำคัญที่สุดของหัวข้อนี้**: `K` และ `V` มี bound ที่ **"อิสระจากกันโดยสิ้นเชิง"** เหมือนที่ `Point<T, U>`
ใน Part 18 มี type ที่อิสระจากกัน แต่ตอนนี้ "อิสระ" หมายถึง **อิสระในเรื่องเงื่อนไข (bound) ด้วย** ไม่ใช่แค่อิสระ
ในเรื่องชนิดข้อมูลที่ใส่จริง — `K` ต้องรับผิดชอบเรื่อง "ใช้เป็น key ในโครงสร้างแบบ hash ได้" ส่วน `V` ต้องรับ
ผิดชอบแค่เรื่อง "คัดลอกได้" เท่านั้น การผสม bound ที่มีความหมายต่างกันคนละชุดแบบนี้เข้ากับ type parameter คนละตัว
คือทักษะสำคัญของการออกแบบ generic type ที่ใช้งานได้จริง ไม่ใช่แค่ตัวอย่างการศึกษาเปล่า ๆ

ถ้าคุณลองสร้าง `Cache<Product, i32>` โดยที่ `Product` (struct ที่มี field `name: String`) ไม่ได้ derive `Hash`
และ `Eq` ไว้ compiler จะปฏิเสธทันที — เราจะเห็น error จริงของกรณีนี้ในหัวข้อ "กับดักที่พบบ่อย" ข้อ 4

### 22.6 Associated type เทียบกับ generic type parameter บน trait

นี่คือหัวข้อที่ลึกและสำคัญที่สุดของบทนี้ — ความแตกต่างระหว่าง**สองวิธี**ในการทำให้ trait "รับ" ชนิดข้อมูลเข้ามา
เกี่ยวข้องด้วย ซึ่งดูคล้ายกันมากในตอนแรก แต่มีความหมายเชิง design ที่ต่างกันอย่างสิ้นเชิง

#### วิธีที่หนึ่ง: associated type (`type Item;`)

```rust
trait ContainerAssoc {
    type Item;
    fn get(&self, i: usize) -> Option<&Self::Item>;
}

struct IntBox(Vec<i32>);

impl ContainerAssoc for IntBox {
    type Item = i32;

    fn get(&self, i: usize) -> Option<&i32> {
        self.0.get(i)
    }
}

fn main() {
    let ib = IntBox(vec![1, 2, 3]);
    println!("{:?}", ib.get(1));
}
```

ผลลัพธ์:

```
Some(2)
```

**อธิบายโค้ด**: `type Item;` ภายใน trait คือการประกาศว่า **"ทุก type ที่จะ implement `ContainerAssoc` ต้อง
เลือกชนิดข้อมูลหนึ่งชนิดมาผูกกับชื่อ `Item`"** — ไม่ใช่ type parameter ที่ผู้เรียก trait เป็นคนกำหนด (เหมือน
generic parameter ปกติ) แต่เป็น**ชนิดข้อมูลที่ผู้ implement trait เป็นคนกำหนดไว้ตายตัว** ตอน `impl` เมื่อ `IntBox`
เขียน `type Item = i32;` มันหมายความว่า **"สำหรับ `IntBox` โดยเฉพาะ `Item` คือ `i32` เสมอ ไม่มีทางเป็นอย่างอื่น"**
เมื่อเรียก `ib.get(1)` compiler รู้ทันทีว่า return type คือ `Option<&i32>` โดยไม่ต้องมีใครระบุ type parameter
เพิ่มเติมเลย (ไม่ต้องเขียน `ib.get::<i32>(1)` แบบ turbofish จาก Part 18 หัวข้อกับดักข้อ 4)

#### วิธีที่สอง: generic type parameter บน trait (`trait Container<Item>`)

```rust
trait ContainerGeneric<Item> {
    fn get(&self, i: usize) -> Option<&Item>;
}

struct MultiBox(Vec<i32>, Vec<String>);

impl ContainerGeneric<i32> for MultiBox {
    fn get(&self, i: usize) -> Option<&i32> {
        self.0.get(i)
    }
}

impl ContainerGeneric<String> for MultiBox {
    fn get(&self, i: usize) -> Option<&String> {
        self.1.get(i)
    }
}

fn main() {
    let mb = MultiBox(vec![10, 20], vec![String::from("a"), String::from("b")]);

    let x: Option<&i32> = ContainerGeneric::<i32>::get(&mb, 0);
    let y: Option<&String> = ContainerGeneric::<String>::get(&mb, 1);

    println!("{:?} {:?}", x, y);
}
```

ผลลัพธ์:

```
Some(10) Some("b")
```

**อธิบายโค้ด**: คราวนี้ `Item` เป็น **generic type parameter ของ trait เอง** (เหมือน `T` ของ struct/ฟังก์ชันที่
เรียนมาตลอด Part 18) ทำให้ `MultiBox` สามารถ **implement `ContainerGeneric` ได้มากกว่าหนึ่งครั้ง** — ครั้งหนึ่ง
สำหรับ `ContainerGeneric<i32>` (ดึงข้อมูลจาก field แรก) และอีกครั้งสำหรับ `ContainerGeneric<String>` (ดึงข้อมูล
จาก field ที่สอง) เพราะในสายตา compiler `ContainerGeneric<i32>` และ `ContainerGeneric<String>` คือ **trait ที่
ต่างกันโดยสิ้นเชิง** (เหมือนที่ `Point<i32>` และ `Point<f64>` เป็น type ต่างกันตามหลักการ Part 18 หัวข้อ 18.4)
— นี่คือเหตุผลที่ต้องเรียกผ่าน syntax `ContainerGeneric::<i32>::get(&mb, 0)` เพื่อบอก compiler ว่ากำลังเรียก
`get` จาก implementation ตัวไหน (ถ้ามีแค่ implementation เดียว compiler infer ให้อัตโนมัติได้จาก context ปกติ
โดยไม่ต้องเขียนยาวแบบนี้)

#### ทำไม `Iterator` ของ std เลือกใช้ associated type ไม่ใช่ generic parameter?

นี่คือคำถามที่สำคัญที่สุดของหัวข้อนี้ นิยามจริงของ `Iterator` ใน standard library คือ:

```
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
    // ... method อื่น ๆ อีกมาก (จะเรียนเต็มรูปแบบใน Part 25-26)
}
```

ทำไมไม่ออกแบบเป็น `trait Iterator<Item> { fn next(&mut self) -> Option<Item>; }` แทน? คำตอบคือ**ความหมายทาง
ธรรมชาติของสิ่งที่ `Iterator` กำลังสร้างแบบจำลอง**: **type หนึ่งตัวควรจะเป็น iterator ของชนิดข้อมูลเดียวเท่านั้น
ในเวลาเดียวกัน** ลองนึกภาพ `struct Counter` ที่นับเลข `0, 1, 2, 3, ...` ไปเรื่อย ๆ — มันควรจะเป็น `Iterator`
ที่ให้ค่า `i32` ออกมาเสมอ **ไม่ควรมีทางที่ `Counter` จะเป็นได้ทั้ง "iterator ของ `i32`" และ "iterator ของ
`String`" ในเวลาเดียวกัน** เพราะนั่นทำให้ความหมายของ "การวนซ้ำครั้งต่อไปคืออะไร" กำกวมทันที: ถ้า `Counter`
implement `ContainerGeneric<i32>` และ `ContainerGeneric<String>` (แบบวิธีที่สอง) พร้อมกัน แล้วเขียน
`for x in counter` (ใช้งานผ่าน trait `IntoIterator` ที่จะเรียนใน Part 25) — compiler จะไม่รู้ว่าต้องเลือก
implementation ตัวไหน เพราะไม่มี context เพิ่มเติมมาบอกว่าต้องการ `i32` หรือ `String` ในจุดนั้น

การเลือกใช้ **associated type** จึงเป็นการ**บังคับกฎทางความหมาย**ไว้ตั้งแต่ระดับ type system: **"type หนึ่งตัว
implement `Iterator` ได้แค่หนึ่งครั้งเท่านั้น และเมื่อ implement แล้ว `Item` ของมันตายตัวตลอดไป"** — นี่ไม่ใช่
ข้อจำกัดที่ไม่มีเหตุผล แต่เป็นการเขียนกฎที่เราต้องการอยู่แล้ว ("iterator ควรให้ค่าชนิดเดียวเสมอ") ให้กลายเป็น
สิ่งที่ **compiler ตรวจสอบให้อัตโนมัติ** เราจะเห็นในหัวข้อถัดไปว่าถ้าคุณพยายาม implement trait ที่มี associated
type ให้ type เดียวกันสองครั้ง compiler จะปฏิเสธด้วย error ที่ยืนยันกฎนี้ตรง ๆ

ตรงกันข้าม บาง trait ต้องการความหมายแบบ **"type หนึ่งตัวทำได้กับหลายชนิดข้อมูล"** โดยธรรมชาติ — ตัวอย่างจาก std
เองคือ trait `From<T>` (ที่จะเรียนใน Part 30): `String` implement `From<&str>`, `From<char>`, `From<i32>` (ผ่าน
`.to_string()` ที่พึ่ง `Display`), และอื่น ๆ อีกมาก **พร้อมกันในเวลาเดียวกัน** เพราะความหมายของ "แปลงจาก `T`
เป็น `String`" สมเหตุสมผลที่จะทำได้กับหลาย `T` — นี่คือเหตุผลที่ `From<T>` ออกแบบด้วย **generic parameter**
(`trait From<T> { fn from(value: T) -> Self; }`) ไม่ใช่ associated type

**สรุปหลักการเลือก**: ถามตัวเองว่า **"type ที่ implement trait นี้ ควรผูกกับชนิดข้อมูลที่เกี่ยวข้องแค่หนึ่งแบบ
ตายตัวหรือไม่?"** ถ้าใช่ (เช่น "iterator นี้ให้ค่าชนิดอะไร", "container นี้เก็บชนิดข้อมูลอะไร") ให้ใช้
**associated type** ถ้าไม่ใช่ — ต้องการให้ implement ได้หลายแบบพร้อมกันสำหรับหลายชนิดข้อมูล (เช่น "แปลงจากอะไร
เป็น type นี้ได้บ้าง") ให้ใช้ **generic parameter บน trait**

#### พิสูจน์ด้วย error จริง: E0119 เมื่อพยายาม implement associated-type trait ซ้ำ

ลองพยายามทำในสิ่งที่กฎ "associated type ตายตัวต่อ type" ห้ามไว้ — implement `ContainerAssoc` (จากตัวอย่างแรก
ของหัวข้อนี้) ให้ type เดียวกันสองครั้งด้วย `Item` คนละชนิด:

```rust
trait ContainerAssoc {
    type Item;
    fn get(&self, i: usize) -> Option<&Self::Item>;
}

struct Dual;

impl ContainerAssoc for Dual {
    type Item = i32;

    fn get(&self, _i: usize) -> Option<&i32> {
        None
    }
}

impl ContainerAssoc for Dual {
    type Item = String;

    fn get(&self, _i: usize) -> Option<&String> {
        None
    }
}

fn main() {}
```

```
error[E0119]: conflicting implementations of trait `ContainerAssoc` for type `Dual`
  --> src/main.rs:15:1
   |
 8 | impl ContainerAssoc for Dual {
   | ---------------------------- first implementation here
...
15 | impl ContainerAssoc for Dual {
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^ conflicting implementation for `Dual`
```

**อธิบาย error นี้ให้ลึก**: compiler มองว่า `impl ContainerAssoc for Dual` ทั้งสองบล็อกนี้เป็นการ **"implement
trait เดียวกันให้ type เดียวกัน"** ซ้ำสองครั้ง (ต่างจากตัวอย่าง `ContainerGeneric<i32>` กับ `ContainerGeneric<String>`
ที่เป็น**trait คนละตัว**ในสายตา compiler เพราะมี generic parameter ต่างกัน) — สังเกตว่า error message **ไม่ได้
พูดถึง `Item` เลยแม้แต่คำเดียว** เพราะปัญหาไม่ได้อยู่ที่ `Item` ต่างกัน แต่อยู่ที่ **`ContainerAssoc for Dual`
(ไม่มี generic parameter ให้แยกความแตกต่าง) ถูกประกาศซ้ำ** — นี่คือ**หลักฐานที่จับต้องได้**ว่า associated type
ไม่ได้เป็น "ส่วนหนึ่งของ identity ของ trait implementation" แบบที่ generic parameter เป็น มันเป็นแค่**ผลลัพธ์
ที่ตายตัว**ของการ implement ครั้งเดียวเท่านั้น ถ้าคุณต้องการให้ `Dual` รองรับทั้ง `i32` และ `String` จริง ๆ ต้อง
เปลี่ยนไปออกแบบด้วย generic parameter (`ContainerGeneric<Item>`) แทน ไม่มีทางแก้อื่นสำหรับ associated type
เพราะนี่คือกฎพื้นฐานของมัน ไม่ใช่ข้อจำกัดที่แก้ได้ด้วยการปรับ syntax เล็กน้อย

### 22.7 `PhantomData<T>`: ติดตามชนิดข้อมูลตอน compile time โดยไม่เก็บข้อมูลจริง

กลับไปที่ error `E0392` จาก Part 18 ที่เราทวนไว้ในหัวข้อ 22.1 — ทีนี้มาดูสถานการณ์จริงที่ทำให้เราอยาก
"ประกาศ type parameter ที่ไม่ได้เก็บข้อมูลจริง" ตั้งแต่แรก

#### ปัญหา: unit-safe wrapper ที่ป้องกันการผสมหน่วยวัดผิด

ลองจินตนาการระบบที่ทำงานกับระยะทางทั้งหน่วยเมตร (meters) และฟุต (feet) — ทั้งสองเก็บเป็น `f64` เหมือนกัน แต่
**การบวกเมตรกับฟุตตรง ๆ โดยไม่แปลงหน่วยก่อนเป็นบั๊กร้ายแรง** (5.0 เมตร + 5.0 ฟุต ไม่เท่ากับ 10.0 อะไรเลยที่มี
ความหมาย) เราต้องการให้ **compiler ปฏิเสธการผสมหน่วยผิดตั้งแต่ compile time** โดยไม่ต้องเก็บ "แท็กบอกหน่วย" ไว้
ใน struct ตอน runtime เลย (เพราะหน่วยรู้อยู่แล้วตั้งแต่ตอนเขียนโค้ด ไม่จำเป็นต้องเสียเนื้อที่ memory มาเก็บมันซ้ำ)

ลองเขียนแบบ**ไม่มี** `PhantomData` ดูก่อนว่าเกิดอะไรขึ้น:

```rust
struct Distance<Unit> {
    value: f64,
}

fn main() {
    println!("compiled");
}
```

```
error[E0392]: type parameter `Unit` is never used
 --> src/main.rs:1:17
  |
1 | struct Distance<Unit> {
  |                 ^^^^ unused type parameter
  |
  = help: consider removing `Unit`, referring to it in a field, or using a marker such as `PhantomData`
  = help: if you intended `Unit` to be a const parameter, use `const Unit: /* Type */` instead
```

**ทำไม compiler ต้องปฏิเสธ**: จากหลักการ monomorphization ใน Part 18 หัวข้อ 18.7 — compiler ต้องรู้ว่า "ขนาด
และ layout ใน memory" ของ `Distance<Unit>` แต่ละแบบคืออะไร ถ้า `Unit` ไม่ถูกใช้เก็บข้อมูลที่ field ไหนเลยสักไบต์
เดียว `Distance<Meters>` และ `Distance<Feet>` จะมี layout ใน memory **เหมือนกันทุกประการ** (มีแค่ `value: f64`
แปดไบต์) ซึ่งทำให้ Rust ตั้งคำถามว่า "แล้วทำไมต้องมี `Unit` เป็น type parameter ด้วยตั้งแต่แรก ถ้ามันไม่ส่งผลอะไร
เลยต่อตัว struct จริง ๆ?" — Rust จึงบังคับให้คุณ**ยืนยันเจตนา**อย่างชัดเจนว่าต้องการ "ป้ายกำกับที่ไม่มีข้อมูล
จริง" จริง ๆ ผ่านการใช้ `PhantomData`

#### ทางแก้: `std::marker::PhantomData<T>`

`PhantomData<T>` คือ **zero-sized type** (type ที่มีขนาด 0 ไบต์เสมอ) จาก standard library ที่มีหน้าที่**เดียว**
คือ "หลอก" compiler ให้เห็นว่า `T` ถูกใช้งานอยู่จริงใน struct (เพื่อผ่านการตรวจสอบ `E0392`) โดยไม่เพิ่มขนาด
memory ให้ struct เลยแม้แต่ไบต์เดียว:

```rust
use std::marker::PhantomData;
use std::ops::Add;

struct Meters;
struct Feet;

struct Distance<Unit> {
    value: f64,
    _unit: PhantomData<Unit>,
}

impl<Unit> Distance<Unit> {
    fn new(value: f64) -> Self {
        Distance {
            value,
            _unit: PhantomData,
        }
    }
}

impl<Unit> Add for Distance<Unit> {
    type Output = Distance<Unit>;

    fn add(self, other: Distance<Unit>) -> Distance<Unit> {
        Distance::new(self.value + other.value)
    }
}

fn main() {
    let a: Distance<Meters> = Distance::new(5.0);
    let b: Distance<Meters> = Distance::new(3.0);
    let c = a + b;
    println!("c = {} meters", c.value);

    let d: Distance<Feet> = Distance::new(12.0);
    println!("d = {} feet", d.value);

    println!(
        "size_of Distance<Meters> = {} bytes",
        std::mem::size_of::<Distance<Meters>>()
    );
    println!(
        "size_of Distance<Feet>   = {} bytes",
        std::mem::size_of::<Distance<Feet>>()
    );
    println!("size_of f64 alone        = {} bytes", std::mem::size_of::<f64>());
}
```

ผลลัพธ์:

```
c = 8 meters
d = 12 feet
size_of Distance<Meters> = 8 bytes
size_of Distance<Feet>   = 8 bytes
size_of f64 alone        = 8 bytes
```

**อธิบายโค้ดทีละส่วนอย่างละเอียด:**

- **`struct Meters; struct Feet;`** — สอง **unit struct** (จาก Part 9) ที่**ไม่มี field เลย** หน้าที่ของมันไม่ใช่
  เก็บข้อมูล แต่เป็นแค่ **"ชื่อ" ที่แยกแยะประเภท** ใช้เป็น "ป้ายกำกับ" ที่ส่งเข้าไปเป็น type parameter ของ
  `Distance<Unit>` เท่านั้น
- **`_unit: PhantomData<Unit>`** — field พิเศษที่ใช้ `Unit` จริง (ทำให้ผ่านการตรวจสอบ `E0392` เพราะ `Unit`
  "ถูกใช้" ใน field นี้แล้ว) แต่เพราะ `PhantomData<Unit>` มีขนาด **0 ไบต์เสมอไม่ว่า `Unit` จะเป็นชนิดข้อมูลอะไร**
  มันไม่เพิ่มขนาดของ `Distance<Unit>` เลยแม้แต่ไบต์เดียว — ชื่อขึ้นต้นด้วย underscore (`_unit`) เป็นธรรมเนียม
  นิยมทั่วไปเพื่อบอกว่า field นี้ "ไม่ได้ใช้งานจริงในเชิง logic" (ป้องกัน warning `unused field` บางกรณี)
- **ตัวเลขจาก `size_of` พิสูจน์คำกล่าวนี้อย่างเป็นรูปธรรม**: `Distance<Meters>` และ `Distance<Feet>` มีขนาด
  **8 ไบต์เท่ากันทุกประการ** — เท่ากับขนาดของ `f64` เปล่า ๆ พอดี (`size_of f64 alone = 8 bytes`) แปลว่า
  `PhantomData<Unit>` **ไม่เพิ่มขนาดอะไรเลยจริง ๆ** ไม่ใช่แค่ทฤษฎีที่บอกไว้ แต่พิสูจน์ได้ด้วยตัวเลขจริงจาก
  compiler — นี่คือหลักฐานที่จับต้องได้แบบเดียวกับที่ Part 18 หัวข้อ 18.7 ใช้ `nm` พิสูจน์ monomorphization
- **`impl<Unit> Add for Distance<Unit>`** — เขียน `Add` (trait สำหรับ operator `+`) ให้กับ `Distance<Unit>`
  **แบบ generic ทุกหน่วยเลย** (ไม่ต้องเขียนแยกให้ `Meters` และ `Feet` คนละ `impl` block) เพราะตรรกะการบวก
  (บวกค่า `value` ตรง ๆ) เหมือนกันไม่ว่าหน่วยจะเป็นอะไร — สังเกตว่า **`Output = Distance<Unit>`** (หน่วยเดียว
  กับ input ทั้งสองฝั่ง) นี่คือจุดสำคัญที่ทำให้ type-safety เกิดขึ้นจริง

ทีนี้ลองดูว่าเกิดอะไรขึ้นถ้าพยายามบวกหน่วยที่ไม่ตรงกัน:

```rust
use std::marker::PhantomData;
use std::ops::Add;

struct Meters;
struct Feet;

struct Distance<Unit> {
    value: f64,
    _unit: PhantomData<Unit>,
}

impl<Unit> Distance<Unit> {
    fn new(value: f64) -> Self {
        Distance {
            value,
            _unit: PhantomData,
        }
    }
}

impl<Unit> Add for Distance<Unit> {
    type Output = Distance<Unit>;

    fn add(self, other: Distance<Unit>) -> Distance<Unit> {
        Distance::new(self.value + other.value)
    }
}

fn main() {
    let a: Distance<Meters> = Distance::new(5.0);
    let f: Distance<Feet> = Distance::new(10.0);
    let bad = a + f;
    println!("{}", bad.value);
}
```

```
error[E0308]: mismatched types
  --> src/main.rs:28:19
   |
28 |     let bad = a + f;
   |                   ^ expected `Distance<Meters>`, found `Distance<Feet>`
   |
   = note: expected struct `Distance<Meters>`
              found struct `Distance<Feet>`
```

**นี่คือหัวใจสำคัญที่สุดของ `PhantomData`**: `a + f` ถูกปฏิเสธตั้งแต่ compile time เพราะ `impl<Unit> Add for
Distance<Unit>` กำหนดว่า `add(self, other: Distance<Unit>)` ต้องรับ `other` ที่เป็น **`Unit` เดียวกัน** กับ
`self` เท่านั้น (`Unit` ตัวเดียวกันถูกใช้ทั้งสองฝั่ง) เมื่อ `a` เป็น `Distance<Meters>` compiler จึง monomorphize
`add` ตัวนั้นให้รับแค่ `Distance<Meters>` เท่านั้น การส่ง `Distance<Feet>` เข้ามาจึงผิด type ทันที — **ทั้งหมดนี้
เกิดขึ้นโดยที่ runtime ไม่ต้องเช็คอะไรเลยแม้แต่นิดเดียว** ไม่มีการเก็บ "แท็กหน่วย" ไว้เทียบกันตอนรัน ไม่มี `if
self.unit != other.unit { panic!() }` แบบที่ภาษาที่ไม่มี type system เข้มงวดต้องทำ — **ความปลอดภัยทั้งหมดอยู่ที่
compile time ล้วน ๆ ด้วยต้นทุน runtime เท่ากับศูนย์** ตรงกับปรัชญา zero-cost abstraction ที่ Part 18 พิสูจน์ไว้
ด้วย monomorphization ทุกประการ เพียงแต่คราวนี้สิ่งที่ "เป็น zero-cost" คือ**ความปลอดภัยของหน่วยวัด** ไม่ใช่แค่
ความเร็วของฟังก์ชัน generic

#### PhantomData ในรูปแบบ typestate pattern

อีกการใช้งานที่พบบ่อยมากคือ **typestate pattern** — ใช้ type parameter (ที่ไม่มีข้อมูลจริง) แทน "สถานะ" ของ
object แล้วให้ `impl` block ที่ต่างกัน (ตามหลักการหัวข้อ 22.4) เปิด/ปิดการเรียก method ตามสถานะนั้น เราจะฝึก
เขียน pattern นี้ในแบบฝึกหัดข้อ 3 ท้ายบท — สำหรับตอนนี้ให้เข้าใจแค่หลักการว่า **`PhantomData<State>` ทำให้
"สถานะ" กลายเป็นส่วนหนึ่งของ type ได้ โดยไม่ต้องเก็บ enum บอกสถานะไว้ตรวจสอบตอน runtime เลย** — compiler เป็น
คนตรวจสอบ "การเปลี่ยนสถานะที่ผิดกฎ" ให้แทน ก่อนโปรแกรมจะรันจริงเสียอีก

#### ตัวแปรของ `PhantomData` ที่ควรรู้จักไว้: covariance และ auto trait

ในระดับที่ลึกกว่านี้ (เกินขอบเขตของบทนี้แต่ควรรู้จักชื่อไว้) `PhantomData<T>` ยังมีผลต่อ **variance** (ว่า
`Distance<Meters>` เกี่ยวข้องกับ `Distance<Feet>` ในเชิง subtyping อย่างไรผ่าน lifetime) และ **auto trait**
อย่าง `Send`/`Sync` (ว่า `Distance<Unit>` จะเป็น `Send`/`Sync` ได้หรือไม่ ขึ้นอยู่กับว่า `Unit` เป็น `Send`/`Sync`
หรือไม่ แม้ `Unit` จะไม่มีข้อมูลจริงเก็บอยู่เลยก็ตาม) — เรื่องนี้ลึกเกินความจำเป็นสำหรับบทนี้ แต่ควรจำไว้ว่า
`PhantomData<T>` ไม่ใช่แค่ "ตัวหลอก compiler ให้ผ่าน `E0392`" เพียว ๆ มันยังสื่อความหมายเรื่อง ownership/variance
เชิงตรรกะของ `T` ให้กับ compiler ด้วย (เช่นถ้าคุณต้องการบอกว่า struct ของคุณ "เป็นเจ้าของ `T` เชิงตรรกะ" แม้จะ
ไม่ได้เก็บ `T` จริง ให้ใช้ `PhantomData<T>`; ถ้าต้องการบอกว่า "แค่ยืม `T` เชิงตรรกะ" จะใช้ `PhantomData<fn() -> T>`
หรือรูปแบบอื่นที่ต่างออกไป — รายละเอียดนี้จะลึกเกินไปสำหรับบทนี้)

### 22.8 Trait bound ผสม lifetime: `T: 'a`

Part 20 สอนเรื่อง lifetime ของ reference (`&'a T`) ไปแล้ว ทีนี้เรามาดู bound รูปแบบพิเศษที่ผสม trait bound กับ
lifetime bound เข้าด้วยกัน:

```rust
use std::fmt::Display;

fn print_with_prefix<'a, T: Display + 'a>(prefix: &'a str, value: &'a T) {
    println!("{prefix}: {value}");
}

fn main() {
    let s = String::from("Rust");
    print_with_prefix("ภาษา", &s);
}
```

ผลลัพธ์:

```
ภาษา: Rust
```

**อธิบายความหมายของ `T: 'a`**: สังเกต bound `T: Display + 'a` — ส่วน `Display` คือ trait bound แบบที่เรียนมา
ตลอด (บอกว่า `T` ต้องพิมพ์ด้วย `{}` ได้) แต่ **`'a`** ตรงนี้**ไม่ใช่ trait** มันคือ **lifetime bound** ที่แปลว่า
**"ทุก reference ที่อาจซ่อนอยู่ข้างใน `T` (ถ้ามี) ต้องมีชีวิตอยู่ไม่สั้นกว่า `'a`"** — พูดให้ตรงกว่านั้นคือ
**`T` ต้องไม่ถือ reference ที่หมดอายุเร็วกว่า `'a`**

ทำไมถึงต้องมี bound นี้? ลองดูฟังก์ชันของเรา: `value: &'a T` คือ reference ที่มีชีวิต `'a` **ไปยัง** ค่าชนิด `T`
ถ้า `T` เอง**ก็เก็บ reference อยู่ข้างใน** (เช่น `T` อาจเป็น `struct Note<'b> { text: &'b str }`) — เราต้อง
มั่นใจว่า reference ข้างใน `T` นั้น (`'b`) **มีชีวิตอยู่ไม่สั้นกว่า `'a`** เพราะถ้า `'b` สั้นกว่า `'a` ได้
ฟังก์ชันของเราอาจถือ `&'a T` ที่ยังไม่หมดอายุ แต่ข้างในมันมี reference (`'b`) ที่หมดอายุไปแล้ว — นั่นคือ
**dangling reference** ที่ Rust ต้องป้องกันไม่ให้เกิดขึ้นได้เลยตามหลักการพื้นฐานที่สุดของ borrow checker จาก
Part 7 และ Part 20

ในตัวอย่างของเรา `T = String` ซึ่งเป็นชนิดข้อมูลที่**เป็นเจ้าของข้อมูลเอง 100%** (ไม่มี reference ซ่อนอยู่ข้างใน
เลย ตามหลักการ ownership จาก Part 6) ดังนั้น `String: 'a` จะเป็นจริง**เสมอไม่ว่า `'a` จะเป็นอะไรก็ตาม** (เพราะไม่
มี reference ข้างในให้ตรวจว่าหมดอายุเร็วไปหรือไม่) — นี่คือเหตุผลที่ **ชนิดข้อมูลที่ไม่มี reference ข้างในเลย
("owned" ทั้งหมด อย่าง `i32`, `String`, `Vec<T>` ที่ `T` เป็น owned) จะผ่าน bound `T: 'a` ได้ตลอดโดยไม่ต้องคิด
อะไรเพิ่มเลย** bound นี้จะมีผลจริง ๆ ก็ต่อเมื่อ `T` เองมี lifetime parameter ซ่อนอยู่ (เช่น `T = Note<'b>`) ซึ่ง
เป็นสถานการณ์ที่จะพบมากขึ้นเมื่อคุณเขียน struct ที่มีทั้ง generic type parameter และ lifetime parameter รวมกัน
ในบทเดียว (เนื้อหานี้จะลึกขึ้นอีกใน **Part 23: Lifetimes ขั้นสูง** ที่พูดถึง lifetime elision และ struct ที่มี
lifetime หลายตัว)

**ข้อสังเกตเพิ่มเติม**: ในโค้ด Rust จริงจำนวนมาก คุณจะเห็น `T: 'static` (แทน `T: 'a` ทั่วไป) ซึ่งแปลว่า **"`T`
ต้องไม่มี reference ที่หมดอายุก่อนสิ้นสุดโปรแกรมเลย"** — เป็นกรณีพิเศษที่เข้มงวดที่สุดของ `T: 'a` (เพราะ
`'static` คือ lifetime ที่ยาวที่สุดเท่าที่มีอยู่) พบได้บ่อยเมื่อเขียนโค้ดที่เกี่ยวกับ thread หรือ trait object
(`dyn Trait + 'static`) ที่จะเรียนใน Part 21 และ Part 37-40 — สำหรับบทนี้ขอให้จำแค่หลักการทั่วไปของ `T: 'a`
ไว้พอเพียง

### 22.9 Blanket implementation เชิงลึก: coherence และ orphan rule

Part 19 แนะนำ blanket implementation ผ่านตัวอย่าง `impl<T: Display> ToString for T` ของ standard library
ไว้สั้น ๆ — ตอนนี้เรามาเขียน blanket implementation **ของตัวเอง** และเจาะลึกกฎที่ควบคุมว่าเมื่อไหร่ทำได้/ทำไม่ได้

#### เขียน blanket implementation ของตัวเอง

สมมติว่าเรามี trait `Summary` (ให้ความสามารถ "สรุปตัวเองเป็นข้อความสั้น ๆ") และต้องการให้ **ทุกอย่างที่
`Summary` ได้ ได้ความสามารถ `Describable` (สรุปแบบมีคำนำหน้า) มาฟรี ๆ โดยไม่ต้อง implement `Describable`
ทีละ type**:

```rust
trait Summary {
    fn summary(&self) -> String;
}

trait Describable {
    fn describe(&self) -> String;
}

// blanket implementation: "ทุก type T ที่ implement Summary จะได้ Describable มาโดยอัตโนมัติ"
impl<T: Summary> Describable for T {
    fn describe(&self) -> String {
        format!("[สรุป] {}", self.summary())
    }
}

struct Article {
    title: String,
}

impl Summary for Article {
    fn summary(&self) -> String {
        format!("บทความเรื่อง \"{}\"", self.title)
    }
}

fn main() {
    let a = Article {
        title: String::from("Rust คืออะไร"),
    };
    println!("{}", a.describe());
}
```

ผลลัพธ์:

```
[สรุป] บทความเรื่อง "Rust คืออะไร"
```

**อธิบายโค้ด**: สังเกตว่า `Article` implement **แค่ `Summary`** เท่านั้น — ไม่มีที่ไหนเขียน `impl Describable
for Article` เลยแม้แต่บรรทัดเดียว แต่เรายังเรียก `a.describe()` ได้ เพราะ `impl<T: Summary> Describable for T`
บอก compiler ว่า **"สำหรับ `T` ตัวไหนก็ตามที่ implement `Summary` ให้ถือว่า `T` นั้น implement `Describable`
ด้วยเสมอ โดยอัตโนมัติ"** — `Article` implement `Summary` แล้ว จึงได้ `Describable` มาโดยอัตโนมัติผ่าน blanket
implementation นี้ทันที นี่คือกลไกเดียวกันเป๊ะกับที่ทำให้ทุกชนิดข้อมูลที่ implement `Display` ได้ `ToString`
มาฟรี ๆ ตามที่ Part 19 แนะนำไว้

#### Coherence และ orphan rule: ทำไม Rust ไม่ให้เขียน blanket implementation ได้ทุกกรณี

blanket implementation เป็นเครื่องมือที่ทรงพลังมาก แต่ Rust มีกฎที่ควบคุมไม่ให้มันสร้างความกำกวมได้ เรียกว่า
**coherence** (ความสอดคล้องกันของ trait implementation ทั้งระบบ) และกฎที่บังคับ coherence ที่สำคัญที่สุดคือ
**orphan rule**: **การ implement trait `Tr` ให้กับ type `Ty` ทำได้ก็ต่อเมื่ออย่างน้อยหนึ่งใน `Tr` หรือ `Ty`
เป็น "ของ local" (ประกาศอยู่ใน crate ปัจจุบันของคุณ)**

ลองดูว่าเกิดอะไรขึ้นถ้าพยายามละเมิดกฎนี้ — สมมติเราอยากให้ทุก type ที่ implement `Summary` (trait ของเราเอง)
ได้ `Display` (trait **ของ standard library**, ไม่ใช่ของเรา) มาโดยอัตโนมัติ:

```rust
use std::fmt;

trait Summary {
    fn summary(&self) -> String;
}

impl<T: Summary> fmt::Display for T {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{}", self.summary())
    }
}

fn main() {}
```

```
error[E0210]: type parameter `T` must be used as the type parameter for some local type (e.g., `MyStruct<T>`)
 --> src/main.rs:7:6
  |
7 | impl<T: Summary> fmt::Display for T {
  |      ^ type parameter `T` must be used as the type parameter for some local type
  |
  = note: implementing a foreign trait is only possible if at least one of the types for which it is implemented is local
  = note: only traits defined in the current crate can be implemented for a type parameter
```

**อธิบาย error นี้ให้ลึก**: `fmt::Display` เป็น trait **ของ standard library** (ไม่ใช่ local — ไม่ได้ประกาศใน
crate ของเรา) และ `T` ที่เราพยายาม implement ให้ก็เป็น **generic type parameter เปล่า ๆ** (ไม่ใช่ type ที่
เป็น local เจาะจงตัวใดตัวหนึ่ง) — ทั้ง `Display` และ `T` จึง **ไม่มีฝั่งใดฝั่งหนึ่งเป็น local เลย** ละเมิด
orphan rule ตรง ๆ error message บอกวิธีแก้ไว้ในตัว: **"type parameter `T` must be used as the type parameter
for some local type (e.g., `MyStruct<T>`)"** — ถ้าจะทำแบบนี้ได้ ต้อง implement ให้กับ struct ของเราเองที่หุ้ม
`T` ไว้ (เช่น `impl<T: Summary> fmt::Display for MyWrapper<T>`) ไม่ใช่ให้กับ `T` เปล่า ๆ ตรง ๆ

**ทำไม Rust ต้องมีกฎนี้?** ลองนึกภาพว่าถ้าไม่มี orphan rule: สมมติ crate ของคุณ (`my_crate`) เขียน
`impl<T: Summary> fmt::Display for T` ได้จริง แล้ววันหนึ่งมี crate อื่น (`other_crate`) ที่คุณก็ใช้อยู่ในโปรเจกต์
เดียวกัน**ก็เขียน** `impl<T: SomeOtherTrait> fmt::Display for T` ของตัวเองด้วย (เขียนแบบเดียวกัน แต่ trait bound
คนละตัว) ถ้ามี type ตัวหนึ่ง (`Foo`) ที่ implement ทั้ง `Summary` และ `SomeOtherTrait` พร้อมกัน — **compiler
จะรู้ได้อย่างไรว่าจะใช้ `Display` implementation จากไหน?** ทั้งสอง crate ไม่รู้จักกันเลย ไม่มีใครผิดในเชิง
โค้ดของตัวเอง แต่รวมกันแล้วเกิดความกำกวมที่แก้ไม่ได้ — orphan rule ป้องกันสถานการณ์นี้**ตั้งแต่ต้นทาง** โดย
บังคับว่า การจะเขียน `impl Foreign Trait for SomeType` ได้ ต้องมีสิ่งหนึ่งที่ "คุณเป็นเจ้าของ" (คุณเขียน trait
เอง หรือคุณเขียน type เอง) เท่านั้น — ทำให้ **แค่คนเดียวในระบบทั้งหมดที่มีสิทธิ์เขียน implementation คู่นั้นได้**
ไม่มีทางที่สอง crate ที่ไม่รู้จักกันจะชนกันได้เลย

ในตัวอย่าง `Describable for T where T: Summary` ที่เราเขียนไว้ก่อนหน้า **ไม่ละเมิดกฎนี้** เพราะ `Describable`
เป็น trait **local** (เราประกาศเองใน crate นี้) แม้ `T` จะเป็น generic parameter เปล่า ๆ ก็ตาม — orphan rule
ผ่านได้ทันทีที่**อย่างน้อยหนึ่งฝั่ง**เป็น local ซึ่งในกรณีนี้คือฝั่ง trait (`Describable`)

**ข้อควรระวังเชิงปฏิบัติ**: แม้ blanket implementation ที่ไม่ละเมิด orphan rule จะ compile ผ่าน แต่ **coherence
ยังมีกฎอีกชั้นที่ป้องกันไม่ให้มี blanket implementation สองตัวที่ทับซ้อนกันจนกำกวมภายใน crate เดียวกัน** — เช่น
ถ้าคุณเขียน `impl<T: Summary> Describable for T` ไว้แล้ว คุณจะ**เขียน** `impl Describable for Article` แยกอีก
บล็อกหนึ่งไม่ได้อีกเลย (จะได้ `E0119` แบบเดียวกับหัวข้อ 22.6 ทันที เพราะ compiler มองว่าทั้งสองบล็อกอาจ apply
กับ `Article` พร้อมกัน) — นี่คือเหตุผลที่ blanket implementation ต้องออกแบบอย่างระมัดระวัง: มันเป็นการ
**"ตัดสินใจแทนทุก type ที่จะมาในอนาคต"** ว่าจะได้ trait นี้มาแบบเดียวกันหมด ไม่มีทางให้ type ใด type หนึ่ง
"ปรับแต่งเฉพาะตัว" ได้อีกถ้ามี blanket implementation ครอบไว้แล้ว

### 22.10 ตัวอย่างจริงขนาดใหญ่: `Repository<T>` ที่ผสมทุกเทคนิคในบทนี้

มาปิดท้ายเนื้อหาด้วยตัวอย่างที่สมจริงและผสมทุกอย่างที่เรียนมาในบทนี้เข้าด้วยกัน: **`Repository<T>`** — pattern
ที่พบได้บ่อยมากในโค้ดจัดการข้อมูลจริง (ทั้งระบบฝั่ง backend, แอปพลิเคชันจัดการข้อมูล, หรือแม้แต่โปรแกรมขนาดเล็ก
ที่ต้องเก็บ/ค้นหา record) — เราจะใช้ `where` clause (หัวข้อ 22.2-22.3), trait bound บน `impl` block (หัวข้อ
22.4), และ associated type (หัวข้อ 22.6) ร่วมกันทั้งหมด

```rust
use std::fmt::Debug;

// trait ที่บอกว่า "ชนิดข้อมูลนี้มี id เอกลักษณ์" — ทุก entity ในระบบต้องมี
trait Entity {
    fn id(&self) -> u32;
}

#[derive(Debug, Clone)]
struct User {
    id: u32,
    name: String,
}

impl Entity for User {
    fn id(&self) -> u32 {
        self.id
    }
}

// Repository<T> เก็บ entity ชนิด T ไว้เป็น Vec — ใช้ where clause เพราะมี bound สามตัว
// ผูกกับ struct เดียวกันทั้งตอนประกาศ struct และตอน impl (ต้องเขียนคู่กันเสมอ)
struct Repository<T>
where
    T: Entity + Clone + Debug,
{
    items: Vec<T>,
}

impl<T> Repository<T>
where
    T: Entity + Clone + Debug,
{
    fn new() -> Self {
        Repository { items: Vec::new() }
    }

    fn add(&mut self, item: T) {
        self.items.push(item);
    }

    fn find_by_id(&self, id: u32) -> Option<&T> {
        self.items.iter().find(|item| item.id() == id)
    }
}

// trait ของเราเองที่มี associated type — ใช้แทน "การเปิดให้เข้าถึงข้อมูลทั้งหมดแบบอ่านอย่างเดียว"
// เลือก associated type เพราะ Repository<T> แต่ละตัวควรมี Item ตายตัวแค่ชนิดเดียว (ตามหลักการหัวข้อ 22.6)
trait RepoView {
    type Item;
    fn all(&self) -> &[Self::Item];
}

impl<T> RepoView for Repository<T>
where
    T: Entity + Clone + Debug,
{
    type Item = T;

    fn all(&self) -> &[T] {
        &self.items
    }
}

fn main() {
    let mut repo: Repository<User> = Repository::new();
    repo.add(User {
        id: 1,
        name: String::from("Alice"),
    });
    repo.add(User {
        id: 2,
        name: String::from("Bob"),
    });

    if let Some(u) = repo.find_by_id(2) {
        println!("Found: {u:?}");
    }

    for u in repo.all() {
        println!("- {u:?}");
    }
}
```

ผลลัพธ์:

```
Found: User { id: 2, name: "Bob" }
- User { id: 1, name: "Alice" }
- User { id: 2, name: "Bob" }
```

**อธิบายภาพรวมของตัวอย่างนี้ ทีละจุดที่เชื่อมกับเนื้อหาที่เรียนมา:**

- **`struct Repository<T> where T: Entity + Clone + Debug`** — ใช้ `where` clause ตั้งแต่ตอนประกาศ struct
  (ไม่ใช่แค่ฟังก์ชัน) เพราะมี bound สามตัวพร้อมกัน (`Entity`, `Clone`, `Debug`) — ถ้าเขียนแบบ inline
  (`struct Repository<T: Entity + Clone + Debug>`) ก็ทำได้เหมือนกัน แต่ `where` ทำให้บรรทัดแรกอ่านง่ายกว่า
  ตามหลักการหัวข้อ 22.2 สังเกตว่า **ต้องเขียน bound ชุดเดียวกันซ้ำอีกครั้งใน `impl<T> Repository<T> where ...`**
  — นี่ไม่ใช่ความซ้ำซ้อนที่ไม่มีเหตุผล แต่เป็นกฎของภาษา: bound ของ struct (ที่ควบคุมว่า `Repository<T>` เป็น
  type ที่ถูกต้องได้เมื่อไหร่) และ bound ของ `impl` block (ที่ควบคุมว่า method ไหนใช้ได้กับ `T` แบบไหน) เป็น
  คนละที่ประกาศกันในทางไวยากรณ์ แม้ในทางปฏิบัติมักจะเขียนให้ตรงกันเพื่อความสอดคล้อง
- **`trait Entity { fn id(&self) -> u32; }`** — trait ธรรมดาที่ไม่มี associated type เลย ใช้เป็น bound
  พื้นฐานที่สุดที่ `Repository<T>` ต้องการ (ต้องหา id ได้ ไม่งั้น `find_by_id` เขียนไม่ได้)
- **`trait RepoView { type Item; fn all(&self) -> &[Self::Item]; }`** — นี่คือจุดที่ใช้ associated type
  ตามหลักการหัวข้อ 22.6: `Repository<User>` ควรมี `Item = User` **ตายตัว** เสมอ ไม่ใช่ `Repository<User>`
  ที่เป็นได้ทั้ง "view ของ `User`" และ "view ของอะไรอื่น" พร้อมกัน — ตรงกับความหมายทางธรรมชาติที่ associated
  type ถูกออกแบบมาให้ (เหมือนที่ `Iterator::Item` ตายตัวต่อ type หนึ่ง ๆ)
- **`impl<T> RepoView for Repository<T> where T: Entity + Clone + Debug { type Item = T; ... }`** — ทุก
  `Repository<T>` (ไม่ว่า `T` จะเป็นอะไรก็ตาม ตราบใดที่ผ่าน bound) ได้ `RepoView` มาโดยที่ `Item` ผูกกับ `T`
  ตัวเดียวกันเสมอโดยอัตโนมัติ — สังเกตว่านี่**ไม่ใช่** blanket implementation (ซึ่งจะเป็น `impl<T> RepoView
  for T` แบบไม่มี `Repository<...>` ครอบ) แต่เป็นการ implement ให้กับ **`Repository<T>` โดยเจาะจง** สำหรับทุก
  `T` ที่เป็นไปได้ ซึ่งไม่ละเมิด orphan rule เลยเพราะ `Repository` เป็น local type ของเราเองอยู่แล้ว

ตัวอย่างนี้แสดงให้เห็นว่าเนื้อหาทั้งบทนี้ไม่ใช่หัวข้อที่แยกจากกัน — `where` clause, trait bound บน `impl`,
associated type ทำงานประสานกันเป็นเครื่องมือชุดเดียวสำหรับออกแบบ generic type ที่ใช้งานได้จริงในระดับที่พบใน
โค้ด production จริง

## กับดักที่พบบ่อย (Common Pitfalls)

**1. เรียก method ที่มีอยู่จริงในซอร์สโค้ด แต่ trait bound ของ `impl` block ไม่ผ่าน**

```rust
use std::fmt::Display;

struct Wrapper<T> {
    value: T,
}

impl<T: Display> Wrapper<T> {
    fn show(&self) {
        println!("value = {}", self.value);
    }
}

struct RawData(Vec<u8>);

fn main() {
    let w = Wrapper { value: RawData(vec![1, 2, 3]) };
    w.show();
}
```

```
error[E0599]: the method `show` exists for struct `Wrapper<RawData>`, but its trait bounds were not satisfied
  --> src/main.rs:17:7
   |
 3 | struct Wrapper<T> {
   | ----------------- method `show` not found for this struct
...
13 | struct RawData(Vec<u8>);
   | ----------------------- doesn't satisfy `RawData: std::fmt::Display`
...
17 |     w.show();
   |       ^^^^ method cannot be called on `Wrapper<RawData>` due to unsatisfied trait bounds
   |
note: trait bound `RawData: std::fmt::Display` was not satisfied
```

**เหตุผลเชิงลึก**: เหมือนที่อธิบายไว้ในหัวข้อ 22.4 — `show` มีอยู่จริงในซอร์สโค้ด แต่ถูก "ล็อก" ไว้ด้วย bound
`T: Display` ของ `impl` block ที่มันอยู่ `RawData` ไม่ implement `Display` (มันเป็น byte buffer ดิบ ๆ ที่ไม่มี
วิธี "พิมพ์" ที่สมเหตุสมผลด้วย `{}`) จึงเรียกไม่ได้ ข้อควรระวังคือ**อย่าสับสน error นี้กับกรณี "ไม่มี method ชื่อ
นี้เลย"** — สังเกตคำว่า "exists ... but its trait bounds were not satisfied" ต่างจาก error ทั่วไปที่บอก "no
method named ... found" (ซึ่งมักเกิดจากพิมพ์ชื่อผิดหรือลืม `use` trait ที่ประกาศ method นั้น)

**วิธีแก้**: implement `Debug` ให้ `RawData` แล้วเปลี่ยน bound ของ `impl` block จาก `Display` เป็น `Debug`
(เพราะข้อมูลแบบ byte buffer มักเหมาะกับการ debug-print มากกว่า format แบบปกติ):

```rust
#[derive(Debug)]
struct RawData(Vec<u8>);

impl<T: std::fmt::Debug> Wrapper<T> {
    fn show(&self) {
        println!("value = {:?}", self.value);
    }
}
```

(ในโค้ดจริงต้องเลือกอย่างใดอย่างหนึ่ง ไม่สามารถมี `impl<T: Display> Wrapper<T> { fn show(&self) {...} }` และ
`impl<T: Debug> Wrapper<T> { fn show(&self) {...} }` พร้อมกันสองบล็อกได้ เพราะจะเป็นการประกาศ method ชื่อซ้ำที่
อาจ apply กับ `T` ตัวเดียวกันได้พร้อมกัน ซึ่งกำกวมและ compiler จะปฏิเสธ)

**2. Implement trait ที่มี associated type ให้ type เดียวกันมากกว่าหนึ่งครั้ง**

```rust
trait Storage {
    type Value;
    fn store(&mut self, value: Self::Value);
}

struct Box1;

impl Storage for Box1 {
    type Value = i32;
    fn store(&mut self, _value: i32) {}
}

impl Storage for Box1 {
    type Value = String;
    fn store(&mut self, _value: String) {}
}

fn main() {}
```

```
error[E0119]: conflicting implementations of trait `Storage` for type `Box1`
  --> src/main.rs:14:1
   |
 8 | impl Storage for Box1 {
   | ---------------------- first implementation here
...
14 | impl Storage for Box1 {
   | ^^^^^^^^^^^^^^^^^^^^^^ conflicting implementation for `Box1`
```

**เหตุผลเชิงลึก**: ตามหลักการหัวข้อ 22.6 — associated type ต้อง**ตายตัวต่อ type ที่ implement** เสมอ `Box1`
implement `Storage` ได้แค่**ครั้งเดียว**เท่านั้น ไม่ว่าจะพยายามให้ `Value` เป็นชนิดข้อมูลกี่แบบก็ตาม เพราะ
`impl Storage for Box1` (ไม่มี generic parameter ใด ๆ มาแยกความแตกต่าง) ถูกมองว่าเป็น "การประกาศเดียวกัน" ไม่
ว่า `type Value` ข้างในจะกำหนดเป็นอะไร

**วิธีแก้**: ถ้าต้องการให้ `Box1` เก็บได้หลายชนิดข้อมูลจริง ๆ ต้องเปลี่ยน `Storage` ให้เป็น **generic parameter
บน trait** แทน (ตามหลักการหัวข้อ 22.6):

```rust
trait Storage<Value> {
    fn store(&mut self, value: Value);
}

struct Box1;

impl Storage<i32> for Box1 {
    fn store(&mut self, _value: i32) {}
}

impl Storage<String> for Box1 {
    fn store(&mut self, _value: String) {}
}
```

**3. ประกาศ type parameter ที่ไม่ได้ใช้เก็บข้อมูลจริงในโครงสร้าง โดยไม่ตั้งใจ**

```rust
struct Order<Status> {
    total: f64,
}

fn main() {
    println!("compiled");
}
```

```
error[E0392]: type parameter `Status` is never used
 --> src/main.rs:1:14
  |
1 | struct Order<Status> {
  |              ^^^^^^ unused type parameter
  |
  = help: consider removing `Status`, referring to it in a field, or using a marker such as `PhantomData`
  = help: if you intended `Status` to be a const parameter, use `const Status: /* Type */` instead
```

**เหตุผลเชิงลึก**: นี่คือ error เดียวกันกับที่ Part 18 ทีเซอร์ไว้ และเราเจาะลึกเต็มรูปแบบในหัวข้อ 22.7 — ถ้า
`Status` ตั้งใจใช้เป็น "ป้ายกำกับสถานะ" แบบ typestate pattern (เช่น `Order<Pending>` กับ `Order<Shipped>` ที่มี
method ต่างกัน) จำเป็นต้องมี field `PhantomData<Status>` ไว้เพื่อ "ยืนยันเจตนา" ให้ compiler อนุญาต

**วิธีแก้**:

```rust
use std::marker::PhantomData;

struct Pending;
struct Shipped;

struct Order<Status> {
    total: f64,
    _status: PhantomData<Status>,
}

impl Order<Pending> {
    fn new(total: f64) -> Self {
        Order { total, _status: PhantomData }
    }

    fn ship(self) -> Order<Shipped> {
        Order { total: self.total, _status: PhantomData }
    }
}
```

(ถ้าไม่ได้ตั้งใจแบบนี้จริง ๆ — เผลอลืมใส่ field ที่ควรมี `Status` เก็บอยู่ — วิธีแก้ที่ตรงไปตรงมากว่าคือกลับไป
เพิ่ม field ที่ใช้ `Status` เป็นชนิดข้อมูลจริง ตามที่ Part 18 แนะนำไว้)

**4. ใช้ struct ของตัวเองเป็น key ของ `Cache<K, V>` (หรือ `HashMap<K, V>`) โดยไม่ derive `Hash`/`Eq`**

```rust
use std::collections::HashMap;
use std::hash::Hash;

struct Cache<K, V> {
    store: HashMap<K, V>,
}

impl<K: Hash + Eq, V> Cache<K, V> {
    fn new() -> Self {
        Cache { store: HashMap::new() }
    }
}

struct Product {
    name: String,
}

fn main() {
    let _cache: Cache<Product, i32> = Cache::new();
}
```

```
error[E0277]: the trait bound `Product: Hash` is not satisfied
  --> src/main.rs:19:39
   |
19 |     let _cache: Cache<Product, i32> = Cache::new();
   |                                       ^^^^^^^^^^^^ the trait `Hash` is not implemented for `Product`
   |
note: required by a bound in `Cache::<K, V>::new`
help: consider annotating `Product` with `#[derive(Hash)]`

error[E0277]: the trait bound `Product: Eq` is not satisfied
  --> src/main.rs:19:39
   |
19 |     let _cache: Cache<Product, i32> = Cache::new();
   |                                       ^^^^^^^^^^^^ the trait `Eq` is not implemented for `Product`
   |
note: required by a bound in `Cache::<K, V>::new`
help: consider annotating `Product` with `#[derive(Eq)]`
```

**เหตุผลเชิงลึก**: ตรงกับหลักการหัวข้อ 22.5 ที่เชื่อมโยงกับข้อกำหนด key ของ `HashMap<K, V>` จาก Part 15 — key
ต้อง implement `Hash` (คำนวณ hash value เพื่อหาตำแหน่งเก็บข้อมูล) และ `Eq` (เทียบความเท่ากันแบบสมบูรณ์ เพื่อ
จัดการ hash collision) โดยอัตโนมัติแล้ว **compiler ไม่มีทางเดาเอาเองได้ว่า "เทียบ `Product` สองตัวว่าเท่ากัน"
ควรทำอย่างไร** (เทียบทุก field? เทียบแค่ `name`? กรณีนี้มีแค่ field เดียวจึงดูตรงไปตรงมา แต่ compiler ก็ยังไม่
ยอมเดาเองอยู่ดี เพื่อความสอดคล้องของกฎ) จึงต้องประกาศให้ชัดเจนเสมอ

**วิธีแก้**: derive ทั้งสอง trait (พร้อม `PartialEq` ที่ `Eq` ต้องพึ่งพา เหมือนหลักการ `PartialOrd` ต้องพึ่ง
`PartialEq` ที่ Part 18 อธิบายไว้):

```rust
#[derive(Hash, PartialEq, Eq)]
struct Product {
    name: String,
}
```

**5. ผสมค่าที่มี type parameter ต่างกันเข้าด้วยกัน ทั้งที่ตั้งใจให้ `PhantomData` ป้องกันไว้อยู่แล้ว**

```rust
use std::marker::PhantomData;

struct Meters;
struct Feet;

struct Distance<Unit> {
    value: f64,
    _unit: PhantomData<Unit>,
}

impl<Unit> Distance<Unit> {
    fn new(value: f64) -> Self {
        Distance { value, _unit: PhantomData }
    }
}

fn total_meters(distances: &[Distance<Meters>]) -> f64 {
    distances.iter().map(|d| d.value).sum()
}

fn main() {
    let mixed = vec![
        Distance::<Meters>::new(5.0),
        Distance::<Feet>::new(3.0),
    ];
    println!("{}", total_meters(&mixed));
}
```

```
error[E0308]: mismatched types
  --> src/main.rs:22:9
   |
20 |     let mixed = vec![
   |                 ---- expected because of this value's type
21 |         Distance::<Meters>::new(5.0),
22 |         Distance::<Feet>::new(3.0),
   |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected `Distance<Meters>`, found `Distance<Feet>`
   |
   = note: expected struct `Distance<Meters>`
              found struct `Distance<Feet>`
```

**เหตุผลเชิงลึก**: `vec![...]` (จาก Part 13) ต้องการให้ทุกสมาชิกเป็น**ชนิดข้อมูลเดียวกัน** — และตามหลักการ
หัวข้อ 22.7 `Distance<Meters>` กับ `Distance<Feet>` เป็นชนิดข้อมูลที่**ต่างกันโดยสิ้นเชิงในสายตา compiler**
(แม้ข้างในเก็บแค่ `f64` เหมือนกัน) การพยายามใส่ทั้งสองไว้ใน `Vec` เดียวกันจึงผิด type ตั้งแต่ตอนสร้าง `vec!`
ก่อนจะไปถึงจุดที่เรียก `total_meters` เสียอีก — **นี่คือ `PhantomData` ทำงานตามที่ออกแบบไว้เป๊ะ**: มันป้องกัน
การผสมหน่วยผิดได้ตั้งแต่จุดแรกที่พยายามรวมค่าเข้าด้วยกัน ไม่ใช่แค่ตอนบวกกันแบบหัวข้อ 22.7 เท่านั้น

**วิธีแก้**: แยก `Vec` ตามหน่วยให้ถูกต้องตั้งแต่ต้น (ซึ่งเป็นพฤติกรรมที่ถูกต้องอยู่แล้ว ไม่ใช่ข้อจำกัดที่ต้อง
"หลีกเลี่ยง" แต่เป็นสิ่งที่ระบบ type ควรบังคับให้ทำ):

```rust
let meters_only: Vec<Distance<Meters>> = vec![
    Distance::<Meters>::new(5.0),
    Distance::<Meters>::new(2.0),
];
println!("{}", total_meters(&meters_only));
```

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** เขียนฟังก์ชัน generic ชื่อ `fn describe_all<T>(items: &[T]) -> String` ที่รับ slice ของค่าใด ๆ
   แล้วคืนข้อความที่เอาแต่ละค่ามาต่อกันด้วย `", "` (ใช้ `.to_string()` กับแต่ละค่า แล้ว `.join(", ")`) โดย
   **บังคับให้เขียน trait bound ด้วย `where` clause** (แม้จะมีแค่ bound เดียว ก็ให้ฝึกไวยากรณ์นี้ให้คุ้นมือ)
   จากนั้นทดสอบเรียกกับ `Vec<i32>` และ `Vec<&str>` ในโปรแกรมเดียวกัน
   (hint: ต้องการ bound `T: Display` เพื่อใช้ `.to_string()` ได้ — เขียนแบบ `fn describe_all<T>(items: &[T])
   -> String where T: std::fmt::Display { ... }` แล้วใช้ `items.iter().map(|item| item.to_string())
   .collect::<Vec<String>>().join(", ")`)

2. **[กลาง]** ออกแบบ trait ชื่อ `Converter<Output>` ที่มี generic parameter (**ไม่ใช่** associated type — ให้
   ฝึกความแตกต่างจากหัวข้อ 22.6) ประกาศ method เดียวคือ `fn convert(&self) -> Output` จากนั้นสร้าง struct ชื่อ
   `Money(f64)` แล้ว implement `Converter<String>` (แปลงเป็นข้อความรูปแบบ `"XXX.XX บาท"`) และ
   `Converter<f64>` (คืนค่าตัวเลขดิบ) **ให้กับ `Money` พร้อมกันทั้งสองแบบ** พิสูจน์ให้เห็นว่า type เดียวกัน
   implement trait ที่มี generic parameter ได้มากกว่าหนึ่งครั้งจริง แล้วลองอธิบายด้วยคำพูดของตัวเองว่าทำไมถ้า
   เปลี่ยน `Converter<Output>` ให้เป็น associated type (`trait Converter { type Output; ... }`) แทน จะทำแบบ
   นี้ไม่ได้อีกต่อไป (ควรได้ error อะไรถ้าลอง)
   (hint: `impl Converter<String> for Money { fn convert(&self) -> String { format!("{:.2} บาท", self.0) } }`
   และ `impl Converter<f64> for Money { fn convert(&self) -> f64 { self.0 } }` — ตอนเรียกใช้ต้องประกาศ type
   ของตัวแปรที่รับผลลัพธ์ให้ชัดเจน เช่น `let text: String = m.convert();` เพื่อให้ compiler เลือก
   implementation ที่ถูกต้อง)

3. **[ยาก]** โค้ดต่อไปนี้ compile ไม่ผ่านด้วย `E0392` (ตั้งใจให้ `State` เป็น typestate แต่ยังไม่มี field ที่ใช้
   `State` จริง) ให้แก้ไขโดยเพิ่ม `PhantomData<State>` แล้วออกแบบ **สอง `impl` block** ที่ผูกกับ `Ticket<Unpaid>`
   และ `Ticket<Paid>` แยกกัน (ตามหลักการหัวข้อ 22.4 เรื่อง `impl` block เจาะจงชนิดข้อมูล): `Ticket<Unpaid>` มี
   method `fn pay(self) -> Ticket<Paid>` (จ่ายเงินแล้วได้ตั๋วที่จ่ายแล้วกลับมา) ส่วน `Ticket<Paid>` มี method
   `fn enter(&self)` (เข้าประตูได้) — ที่สำคัญคือ `Ticket<Unpaid>` **ต้องไม่มี** `.enter()` และ `Ticket<Paid>`
   **ต้องไม่มี** `.pay()` (ลองพิสูจน์ด้วยการเขียนโค้ดที่เรียกผิดลำดับดูว่าได้ error `E0599` จริงหรือไม่):
   ```rust
   struct Unpaid;
   struct Paid;

   struct Ticket<State> {
       price: f64,
   }

   fn main() {
       println!("compiled");
   }
   ```
   (hint: ดูโครงสร้างคำตอบแบบเต็มได้จากตัวอย่าง typestate ในหัวข้อ 22.7 ที่พูดถึง `Order<Pending>`/
   `Order<Shipped>` แล้วปรับชื่อ/method ให้ตรงกับโจทย์ตั๋วนี้)

4. **[ยาก/ประยุกต์]** ขยายตัวอย่าง `Repository<T>` ในหัวข้อ 22.10 โดยเพิ่ม trait ใหม่ชื่อ `Describable` (แบบ
   blanket implementation ตามหลักการหัวข้อ 22.9) ที่ให้ทุกชนิดข้อมูลที่ implement `Entity` ได้ method
   `fn describe(&self) -> String` มาโดยอัตโนมัติ (คืนข้อความรูปแบบ `"Entity #<id>"`) จากนั้นเพิ่ม method ใหม่
   ให้ `Repository<T>` ชื่อ `fn describe_all(&self) -> Vec<String>` ที่เรียก `.describe()` กับทุก item ใน
   repository แล้วคืนเป็น `Vec<String>` สุดท้ายอธิบายด้วยคำพูดของตัวเองว่า `impl<T: Entity> Describable for T`
   นี้ไม่ละเมิด orphan rule เพราะอะไร (อ้างอิงหลักการจากหัวข้อ 22.9)
   (hint: `impl<T: Entity> Describable for T { fn describe(&self) -> String { format!("Entity #{}",
   self.id()) } }` แล้วใน `impl<T> Repository<T> where T: Entity + Clone + Debug` เพิ่ม method
   `fn describe_all(&self) -> Vec<String> { self.items.iter().map(|item| item.describe()).collect() }` —
   สังเกตว่า method นี้ใช้ได้เพราะ `T: Entity` อยู่ใน bound ของ `Repository<T>` อยู่แล้ว ทำให้ `Describable`
   (ที่ได้มาจาก blanket implementation) มีให้ใช้เสมอโดยอัตโนมัติ)

## สรุป

บทนี้เป็นภาคต่อโดยตรงของ Part 18 และปิดสองเรื่องที่ถูกแขวนไว้: **`where` clause** และ **`PhantomData<T>`** เรา
เริ่มจากการพิสูจน์ว่า `where` clause กับ inline trait bound **เหมือนกันทุกประการ** ในกรณีทั่วไป (ไม่มีผลต่อ
type-checking หรือ monomorphization เลย ต่างกันแค่ความอ่านง่าย) ก่อนจะเจาะลึกไปถึงกรณีที่ `where` เป็น**ทางเดียว**
ที่ใช้ได้จริง — การ bound บน type expression ที่ไม่ใช่ type parameter เปล่า ๆ อย่าง `Option<T>: PartialOrd`
(ซึ่งเราพิสูจน์ด้วยการพยายามเขียน inline แล้วได้ syntax error จริง) และการ bound บน associated type อย่าง
`I::Item: Display`

จากนั้นเราขยาย **trait bound บน `impl` block** ที่ Part 19 แนะนำไว้สั้น ๆ ให้เป็นระบบมากขึ้น เห็น error
`E0599` จริงเมื่อพยายามเรียก method ที่ "มีอยู่จริงแต่ถูกล็อกด้วย bound" และเรียนรู้ pattern การเขียน `impl`
block หลายระดับความเข้มงวดซ้อนกันสำหรับ struct เดียวกัน — ต่อด้วยการออกแบบ **`Cache<K, V>`** ที่มี type
parameter สองตัวซึ่งมี bound คนละชุดกันโดยเจตนา (`K: Hash + Eq + Clone` ผูกกับข้อกำหนดของ `HashMap` จาก Part 15,
`V: Clone` ผูกกับความต้องการคืนค่าเป็นเจ้าของ)

หัวใจสำคัญที่สุดของบทนี้คือความแตกต่างระหว่าง **associated type** (`type Item;`) กับ **generic parameter บน
trait** (`trait Container<Item>`) — เราพิสูจน์ด้วย error `E0119` จริงว่า type หนึ่งตัว implement trait ที่มี
associated type ได้แค่**ครั้งเดียว**เท่านั้น (ตรงกับเหตุผลที่ `Iterator` ของ std เลือกใช้ associated type เพราะ
type หนึ่งตัวควรเป็น iterator ของชนิดข้อมูลเดียวเท่านั้น) ในขณะที่ generic parameter บน trait implement ซ้ำได้
หลายครั้งสำหรับหลายชนิดข้อมูล (เหมาะกับ trait อย่าง `From<T>` ที่ต้องการความหมายแบบ "แปลงจากหลายอย่างได้")

เราแก้ error `E0392` จาก Part 18 อย่างเต็มรูปแบบด้วย **`PhantomData<T>`** ผ่านตัวอย่าง `Distance<Unit>` ที่
ป้องกันการผสมหน่วยเมตร/ฟุตผิดตั้งแต่ compile time พร้อมพิสูจน์ด้วย `size_of` ว่ามันไม่กิน memory เพิ่มเลยแม้แต่
ไบต์เดียว (zero-cost เหมือน monomorphization ที่ Part 18 พิสูจน์ไว้) ต่อด้วย lifetime bound แบบ `T: 'a` ในระดับ
ที่พอเพียงสำหรับตอนนี้ และปิดท้ายด้วย **blanket implementation** เชิงลึก — เขียนของตัวเอง (`Describable for
T: Summary`) และเข้าใจ **orphan rule/coherence** ผ่าน error `E0210` จริงที่เกิดจากการพยายาม implement foreign
trait ให้ generic parameter เปล่า ๆ

ตัวอย่าง `Repository<T>` ในหัวข้อสุดท้ายผสมทุกเทคนิคของบทนี้เข้าด้วยกัน — `where` clause, trait bound บน
`impl` block, และ associated type ทำงานร่วมกันเป็นเครื่องมือชุดเดียวสำหรับออกแบบ generic type ที่ใช้งานได้จริง

ใน **Part 23 (Lifetimes ขั้นสูง)** เราจะกลับไปเจาะลึกเรื่อง lifetime ที่บทนี้แค่แตะผิวเผินไว้ในหัวข้อ 22.8
(`T: 'a`) ให้ครบทุกมิติ: lifetime elision rules ทำงานอย่างไรจริง ๆ, higher-ranked trait bound (`for<'a>`),
และ struct ที่มี lifetime parameter หลายตัวพร้อมกัน — ความเข้าใจเรื่อง trait bound และ `where` clause ที่แน่น
จากบทนี้จะทำให้ syntax ที่ผสม lifetime กับ trait bound เข้าด้วยกันใน Part 23 ไม่ใช่เรื่องใหม่อีกต่อไป

---

**Part ก่อนหน้า:** [Traits ขั้นสูง](part-021-traits-advanced.md) | **Part ถัดไป:**
[Lifetimes ขั้นสูง](part-023-lifetimes-advanced.md)
