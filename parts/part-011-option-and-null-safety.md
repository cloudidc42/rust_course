# Part 11: Option<T> และ Null Safety

> โมดูล: พื้นฐานภาษา Rust (Core Language Fundamentals) | ระดับ: พื้นฐาน | เวลาโดยประมาณ: 150 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้อย่างลึกซึ้งว่าทำไม Tony Hoare เรียก null reference ว่า "billion dollar mistake" และ Rust แก้ปัญหานี้
  อย่างไรด้วยการทำให้ "การไม่มีค่า" เป็น type ที่ compiler บังคับให้เช็คเสมอ ไม่ใช่ค่าที่แอบแฝงมาโดยไม่มีสัญญาณเตือน
- จดจำได้ว่า `Option<T>` โผล่มาให้เห็นตรงไหนบ้างในโค้ด Rust จริง เช่น `Vec::first()`, `.get(i)`, `str::find()`,
  และฟังก์ชันค้นหาที่เราออกแบบเอง
- ใช้ `.unwrap()` และ `.expect("...")` ได้อย่างมีสติ รู้ว่า panic message หน้าตาเป็นอย่างไร และตัดสินใจได้ว่าเมื่อไหร่
  ควรใช้ (prototype, test, invariant ที่พิสูจน์แล้วว่าเป็นไปไม่ได้ที่จะเป็น `None`) กับเมื่อไหร่ที่มันคือ code smell
  ในโค้ด production
- ดึงค่าจาก `Option<T>` แบบไม่ panic ได้หลายวิธี (`.unwrap_or()`, `.unwrap_or_else()`, `.unwrap_or_default()`) และ
  เข้าใจความต่างเรื่อง eager กับ lazy evaluation ว่าทำไมมันสำคัญกับ performance และ side effect
- แปลงค่าข้างใน `Option<T>` โดยไม่ต้อง unwrap ก่อนด้วย `.map()` และ `.and_then()` และรู้ว่าเมื่อไหร่ต้องใช้ตัวไหน
- ใช้ `.is_some()`, `.is_none()`, `.as_ref()`, `.as_mut()`, `.or()`, `.or_else()`, `.filter()`, `.zip()` ได้ถูก
  สถานการณ์ พร้อมเข้าใจผลกระทบเรื่อง ownership ของแต่ละ method
- แปลง `Option<T>` เป็น `Result<T, E>` ด้วย `.ok_or()`/`.ok_or_else()` และใช้ `?` operator กับฟังก์ชันที่คืน
  `Option<T>` ได้ — เตรียมพร้อมสำหรับ Part 12 ที่จะเจาะลึก `Result`/error handling เต็มรูปแบบ
- แยกแยะความแตกต่างระหว่าง `Option<&T>` กับ `&Option<T>` ได้ และรู้ว่าแต่ละรูปแบบโผล่มาจากสถานการณ์ไหน

## ความรู้ที่ต้องมีมาก่อน

- **Part 10 (Enums และ Pattern Matching)**: บทนี้แนะนำ `Option<T>` ไปแล้วในฐานะ enum ที่ Rust สร้างไว้ให้อัตโนมัติ
  (`enum Option<T> { Some(T), None }`) พร้อม `match` พื้นฐาน และปิดท้ายด้วยการบอกตรง ๆ ว่าจะเก็บรายละเอียดเรื่อง
  method อย่าง `.unwrap_or()`, `.map()`, `.and_then()` ไว้ให้ Part นี้ — ถ้าคุณยังไม่คุ้นกับรูปร่างของ `Option<T>`
  ในฐานะ enum, การ `match`/`if let`/`while let` บนมัน แนะนำให้กลับไปทวนหัวข้อ 10.9-10.10 ก่อน เพราะบทนี้จะไม่สอน
  พื้นฐานพวกนั้นซ้ำอีก
- **Part 6 (Ownership)** และ **Part 7 (Borrowing)**: หลาย method ของ `Option<T>` ในบทนี้ (โดยเฉพาะ `.as_ref()`,
  `.as_mut()`, และความแตกต่างระหว่าง `Option<&T>`/`&Option<T>`) เกี่ยวข้องกับ ownership/borrowing โดยตรง — ต้องเข้าใจ
  ว่าทำไม `.unwrap()` "ยึด" ค่าออกไป (move) ในขณะที่ `.as_ref()` แค่ "ยืมดู" เท่านั้น
- **Part 9 (Structs)**: ตัวอย่างท้ายบทจะใช้ struct ที่มี field เป็น `Option<T>` และ array ของ struct คงที่ (const
  array) แบบเดียวกับที่เรียนมาแล้ว

## เนื้อหา

### 11.1 ทวนความจำสั้น ๆ: `Option<T>` คือคำตอบของ Rust ต่อ "Billion Dollar Mistake"

Part 10 เล่าไปแล้วว่า Tony Hoare ผู้คิดค้น null reference ขึ้นในภาษา ALGOL W เมื่อปี 1965 เรียกมันว่า **"ความผิดพลาด
มูลค่าพันล้านดอลลาร์"** ในการบรรยายปี 2009 เพราะปัญหาของ null ไม่ใช่แค่ "มันทำให้โปรแกรม crash" แต่คือ **ในภาษาส่วนใหญ่
ตัวแปรที่ประกาศเป็น type `T` (เช่น `User`, `String`) แอบเป็น `null` ได้เสมอโดยที่ type ของมันไม่ได้บอกความเป็นไปได้นี้
ไว้ตรง ๆ เลย** — คุณเห็น `User user = ...` ใน Java แล้วไม่มีทางรู้จาก signature ล้วน ๆ ว่ามันอาจเป็น `null` หรือไม่
ต้องอาศัยความจำ, documentation, หรือประสบการณ์ล้วน ๆ ถ้าลืมเช็คสักครั้งเดียว โปรแกรมก็ล่มตอน runtime ทันที ความเสียหาย
สะสมจากบั๊กประเภทนี้ทั่วอุตสาหกรรมตลอด 60 ปีที่ผ่านมาจึงถูกประเมินไว้ในระดับ "พันล้านดอลลาร์" ไม่ใช่คำพูดเกินจริง

Rust เลือกวิธีแก้ที่สุดโต่งกว่าภาษาส่วนใหญ่ที่พยายามแก้ปัญหานี้ทีหลัง (เช่น `Optional<T>` ของ Java ที่เป็นแค่ class
เสริม หรือ `strictNullChecks` ของ TypeScript ที่เป็นแค่ชั้นตรวจสอบทับบน JavaScript ที่มี `null` อยู่แล้ว): **Rust ไม่มี
`null` หลงเหลืออยู่ในภาษาเลยแม้แต่นิดเดียว (ในส่วนของ safe Rust)** ทุกครั้งที่ต้องแทน "อาจมีค่า หรืออาจไม่มีค่า" Rust
ใช้ enum ธรรมดา ๆ ที่ชื่อ `Option<T>` ซึ่งมีแค่ 2 variant คือ `Some(T)` (มีค่า) กับ `None` (ไม่มีค่า) และเพราะมันเป็น
type ที่ **ต่างจาก `T` อย่างสมบูรณ์ในเชิง type system** คุณไม่มีทางใช้ `Option<T>` เหมือนเป็น `T` ตรง ๆ ได้เลยจนกว่าจะ
ผ่านการ `match`/`if let`/method ที่จัดการทั้งสองกรณีก่อน ลองดู error ที่เราเห็นมาแล้วใน Part 10 อีกครั้งเพื่อตอกย้ำ:

```rust
fn find_user_age(name: &str) -> Option<u32> {
    if name == "สมชาย" {
        Some(30)
    } else {
        None
    }
}

fn birthday_greeting(age: u32) -> String {
    format!("สุขสันต์วันเกิดอายุ {age} ปี")
}

fn main() {
    let age = find_user_age("สมชาย");
    println!("{}", birthday_greeting(age)); // ส่ง Option<u32> เข้าฟังก์ชันที่ต้องการ u32 ตรง ๆ
}
```

```
error[E0308]: mismatched types
  --> src/main.rs:15:38
   |
15 |     println!("{}", birthday_greeting(age));
   |                    ----------------- ^^^ expected `u32`, found `Option<u32>`
   |                    |
   |                    arguments to this function are incorrect
   |
   = note: expected type `u32`
              found enum `Option<u32>`
```

**นี่คือหัวใจของบทนี้**: `age` มี type เป็น `Option<u32>` ไม่ใช่ `u32` — compiler บังคับให้คุณ "เปิดกล่อง" `Option`
ก่อนใช้งานเสมอ ไม่มีทางลืมเช็คแล้ว compile ผ่านได้เลย ต่างจากภาษาที่มี null ที่ปล่อยให้ error แบบนี้ไปโป่งแตกตอน
runtime เท่านั้น Part 10 สอนวิธี "เปิดกล่อง" แบบพื้นฐานที่สุดไปแล้วคือ `match` และ `if let`/`while let` — บทนี้จะ
เจาะลึกเครื่องมือที่ครบมือกว่านั้นมาก ทั้ง method สั้น ๆ ที่ไม่ต้องเขียน `match` ทุกครั้ง (`.unwrap_or()`, `.map()`,
`.and_then()`, ...) และวิธีคิดว่าเมื่อไหร่ควรใช้เครื่องมือตัวไหนในโค้ดจริง

### 11.2 `Option<T>` โผล่มาให้เห็นทั่วไปในโค้ด Rust จริง

ก่อนจะไปเรียน method ต่าง ๆ ให้เห็นภาพก่อนว่า `Option<T>` ไม่ใช่แค่สิ่งที่คุณต้องประกาศเองเวลาเขียนฟังก์ชัน —
standard library ของ Rust ใช้ `Option<T>` เป็น return type ของ method ที่ "อาจไม่มีคำตอบให้" อยู่ทั่วไปแทบทุกที่
มาดูตัวอย่างที่คุณจะเจอบ่อยที่สุด:

```rust
fn main() {
    let numbers = vec![10, 20, 30];

    println!("{:?}", numbers.first());  // สมาชิกตัวแรก (ถ้ามี)
    println!("{:?}", numbers.get(1));   // สมาชิกตำแหน่งที่ 1 (ถ้ามีตำแหน่งนั้นจริง)
    println!("{:?}", numbers.get(99));  // ตำแหน่ง 99 ไม่มีอยู่จริงใน Vec นี้

    let text = "hello, rust world";
    println!("{:?}", text.find("rust"));   // ตำแหน่งที่พบ substring (ถ้าพบ)
    println!("{:?}", text.find("python")); // ไม่พบ substring นี้เลย
}
```

ผลลัพธ์:

```
Some(10)
Some(20)
None
Some(7)
None
```

**อธิบายทีละบรรทัด:**

- `numbers.first()` — คืน `Option<&i32>` (`Some(&10)` ในที่นี้ ซึ่งเวลา debug print ด้วย `{:?}` จะโชว์แค่ `Some(10)`
  เพราะ `Debug` ของ reference จะแสดงค่าที่ชี้ไปโดยไม่ใส่ `&` ให้เห็น) — ถ้า `Vec` ว่างเปล่า จะคืน `None` แทน ไม่ panic
  แต่อย่างใด นี่คือเหตุผลที่ `.first()` ปลอดภัยกว่าการเขียน `numbers[0]` ตรง ๆ ซึ่งจะ panic ทันทีถ้า `Vec` ว่าง
  (เราจะเจาะลึก `Vec<T>` เต็มรูปแบบใน **Part 13** — ตอนนี้จำแค่ว่า method ที่ "อาจไม่มีคำตอบ" ของมันคืน `Option`
  เสมอ)
- `numbers.get(1)` — เหมือน `.first()` แต่รับตำแหน่ง (index) ที่ต้องการเอง คืน `Some(&value)` ถ้าตำแหน่งนั้นมีอยู่จริง
  หรือ `None` ถ้าเกินขอบเขต (`numbers.get(99)` คืน `None` เพราะ `numbers` มีสมาชิกแค่ 3 ตัว ไม่มีตำแหน่งที่ 99)
- `text.find("rust")` — คืน `Option<usize>` เป็นตำแหน่ง byte แรกที่พบ substring ที่ค้นหา (`Some(7)` เพราะ `"rust"`
  เริ่มที่ตำแหน่ง byte ที่ 7 ของ `"hello, rust world"`) ถ้าไม่พบเลยก็คืน `None` — เราจะเจาะลึก `String`/`&str` และ
  UTF-8 เต็มรูปแบบใน **Part 14**

จะเห็น pattern ที่ซ้ำกันชัดเจน: **ทุกครั้งที่ method หนึ่ง "อาจมีคำตอบ หรืออาจไม่มี" ขึ้นอยู่กับข้อมูล ณ ขณะนั้น
(ไม่ใช่ error ที่ผิดปกติ แต่เป็นผลลัพธ์ปกติที่คาดหวังได้) Rust จะเลือกคืน `Option<T>` แทนการ panic หรือคืนค่า
placeholder ที่คลุมเครือ** (เช่น `-1` ใน C ที่บอก "ไม่พบ" แต่ยังเป็น `int` ปนกับค่าจริงอยู่ดี ต้องจำเองว่า `-1`
มีความหมายพิเศษ) เราจะเจอ pattern นี้อีกครั้งกับ `HashMap::get()` ใน **Part 15** ที่คืน `Option<&V>` เมื่อค้นหาด้วย
key ที่อาจไม่มีอยู่จริงในตาราง

#### เขียนฟังก์ชันของเราเองที่ "อาจไม่มีคำตอบ" ด้วย `Option<T>`

รูปแบบเดียวกันนี้ใช้ได้กับฟังก์ชันที่เราออกแบบเองด้วย — เมื่อไหร่ที่ตรรกะของฟังก์ชันมีทางเป็นไปได้ที่จะ "หาไม่พบ"
โดยที่ไม่ใช่ความผิดพลาดร้ายแรง (แค่ผลลัพธ์ปกติของการค้นหา) ให้ประกาศ return type เป็น `Option<T>` เสมอ:

```rust
struct User {
    id: u32,
    name: String,
}

fn find_user_by_id(users: &[User], id: u32) -> Option<&User> {
    for user in users {
        if user.id == id {
            return Some(user);
        }
    }
    None
}

fn main() {
    let users = vec![
        User { id: 1, name: "สมชาย".to_string() },
        User { id: 2, name: "สมหญิง".to_string() },
    ];

    match find_user_by_id(&users, 2) {
        Some(u) => println!("พบผู้ใช้: {}", u.name),
        None => println!("ไม่พบผู้ใช้"),
    }

    match find_user_by_id(&users, 99) {
        Some(u) => println!("พบผู้ใช้: {}", u.name),
        None => println!("ไม่พบผู้ใช้"),
    }
}
```

ผลลัพธ์:

```
พบผู้ใช้: สมหญิง
ไม่พบผู้ใช้
```

สังเกต signature `fn find_user_by_id(users: &[User], id: u32) -> Option<&User>` อย่างละเอียด: รับ `&[User]`
(slice ที่ยืมมา ตามที่เรียนใน Part 8) และคืน `Option<&User>` (reference ที่ยืมมาจากสมาชิกใน slice นั้น ไม่ใช่
`User` ตัวจริงที่ยึด ownership ออกไป) การใช้ `&User` แทน `User` ตรง ๆ สำคัญมาก เพราะถ้าคืน `Option<User>` (ไม่มี `&`)
ฟังก์ชันจะต้อง **ยึด ownership ของ `User` ตัวนั้นออกจาก slice ที่แค่ยืมมา** ซึ่งเป็นไปไม่ได้เลยตามกฎ ownership
จาก Part 6-7 (คุณไม่มีทาง move ค่าออกจากสิ่งที่คุณแค่ยืมมาอ่าน) — เราจะเห็นข้อผิดพลาดแบบนี้ชัดเจนขึ้นในหัวข้อ
"กับดักที่พบบ่อย" ท้ายบท ตอนนี้จำหลักไว้ก่อนว่า **ฟังก์ชันค้นหาที่รับข้อมูลมาแบบยืม ควรคืนเป็น `Option<&T>`
(ยืมกลับไป) ไม่ใช่ `Option<T>` (ยึดออกมา)** เพื่อให้ผู้เรียกยังเป็นเจ้าของข้อมูลต้นฉบับอยู่เหมือนเดิม

### 11.3 `Option<&T>` ไม่มีต้นทุนเพิ่ม: Null Pointer Optimization

ก่อนไปเรียน method ต่าง ๆ ของ `Option<T>` ต่อ มีข้อเท็จจริงเชิง performance ที่น่าสนใจมากอย่างหนึ่งที่ต่อยอดจาก
เรื่อง **niche filling optimization** ที่ Part 10 แนะนำไว้สั้น ๆ ตอนพูดถึงขนาดของ enum ที่มีข้อมูล (หัวข้อ 10.3):
`Option<&T>` **ไม่กินพื้นที่ memory เพิ่มเลยแม้แต่ byte เดียว** เมื่อเทียบกับ `&T` เปล่า ๆ ทั้งที่ในทางความหมาย
`Option<&T>` ต้องเก็บข้อมูลเพิ่ม ("มีค่าหรือไม่มี") มากกว่า `&T` อยู่แล้ว

```rust
fn main() {
    println!("size_of &i32               = {}", std::mem::size_of::<&i32>());
    println!("size_of Option<&i32>       = {}", std::mem::size_of::<Option<&i32>>());
    println!("size_of i32                = {}", std::mem::size_of::<i32>());
    println!("size_of Option<i32>        = {}", std::mem::size_of::<Option<i32>>());
    println!("size_of Box<i32>           = {}", std::mem::size_of::<Box<i32>>());
    println!("size_of Option<Box<i32>>   = {}", std::mem::size_of::<Option<Box<i32>>>());
}
```

ผลลัพธ์ (บนเครื่อง 64-bit ทั่วไป):

```
size_of &i32               = 8
size_of Option<&i32>       = 8
size_of i32                = 4
size_of Option<i32>        = 8
size_of Box<i32>           = 8
size_of Option<Box<i32>>   = 8
```

สังเกตสองคู่ที่น่าสนใจที่สุด:

- **`&i32` และ `Option<&i32>` มีขนาดเท่ากันเป๊ะ ๆ คือ 8 ไบต์** ทั้งที่ `Option<&i32>` ต้องแยกแยะได้ระหว่าง `Some(&x)`
  กับ `None` ด้วย ในขณะที่ `&i32` เดี่ยว ๆ มีแต่กรณี "มีค่า" อย่างเดียว
- **`Box<i32>` และ `Option<Box<i32>>` ก็มีขนาดเท่ากันเช่นกัน** (`Box<T>` คือ smart pointer ที่จะเรียนเต็มรูปแบบใน
  Part 27 — ตอนนี้มองว่าเป็น pointer ที่ชี้ไปยังข้อมูลบน heap ก็เพียงพอ)
- ในทางกลับกัน **`i32` (4 ไบต์) กับ `Option<i32>` (8 ไบต์) มีขนาดต่างกัน** — `Option<i32>` ต้องใหญ่ขึ้นเพื่อเก็บ
  "tag" บอกว่าเป็น `Some`/`None` เพิ่มเติม (ในทางปฏิบัติ compiler จัดให้ครบ 8 ไบต์เพื่อรักษา alignment ที่เหมาะสม
  บนเครื่อง 64-bit)

เหตุผลที่ `Option<&T>` (และ `Option<Box<T>>`) ไม่มีต้นทุนเพิ่มคือสิ่งที่เรียกว่า **null pointer optimization**
(กรณีพิเศษของ niche filling optimization ที่ Part 10 พูดถึง) — **ในภาษา Rust ที่ปลอดภัย (`&T` ที่ไม่ใช่ raw
pointer) ไม่มีทางเป็นค่า pointer ที่เป็น 0 (null) ได้เลย** compiler จึงรู้ว่า **bit pattern ที่เป็น 0 ทั้งหมด**
ของ pointer นั้น "ว่างอยู่" ไม่มีความหมายใด ๆ ในตัว `&T` เอง — มันจึงยืม bit pattern นั้นมาใช้แทน `None` ได้เลย
โดยไม่ต้องเพิ่มพื้นที่ tag แยกต่างหาก ผลลัพธ์คือ: `Some(x)` ถูกเก็บเป็น pointer ตัวจริงที่ชี้ไปยัง `x` (ไม่มีวัน
เป็น 0), และ `None` ถูกเก็บเป็น pointer ที่มีค่า 0 ไปเลย — การเช็คว่าเป็น `Some`/`None` ที่ระดับ machine code
จึงแปลงเป็นแค่ "เช็คว่า pointer เป็น null หรือไม่" ซึ่งเป็นการเปรียบเทียบ 1 คำสั่งเดียวที่เร็วมาก

นี่คือเหตุผลเชิงวิศวกรรมที่แท้จริงว่าทำไมการออกแบบ API ให้คืน `Option<&T>` (ตามที่เห็นใน `.first()`, `.get()`,
`.as_ref()`, และ `find_by_sku`/`find_user_by_id` ในบทนี้) **ไม่มีข้อเสียด้าน performance เลยเมื่อเทียบกับการคืน
pointer ดิบที่อาจเป็น null ได้ในภาษาอื่นอย่าง C/C++** — คุณได้ความปลอดภัยที่ compiler บังคับให้เช็คเสมอ (ตามที่
เรียนมาตั้งแต่หัวข้อ 11.1) โดยไม่ต้องแลกกับ memory หรือความเร็วที่มากขึ้นแม้แต่นิดเดียว สอดคล้องกับแนวคิด
**zero-cost abstraction** ที่หลักสูตรนี้พูดถึงมาตั้งแต่ Part 1 และ Part 10 ได้เห็นตัวอย่างที่คล้ายกันมาแล้วกับ
enum ที่มีข้อมูลติดตัว (`PaymentMethod` ที่มี `String` — ต่างกันตรงที่กรณีนั้นไม่มี niche ให้ใช้ฟรี เพราะ `String`
ไม่มี bit pattern ที่ "เป็นไปไม่ได้" ให้ยืมมาใช้แบบเดียวกับ pointer)

### 11.4 `.unwrap()` และ `.expect()`: ทางลัดที่ต้องใช้อย่างมีสติ

วิธีที่เร็วที่สุดในการ "เอาค่าออกมาจาก `Option<T>`" คือ `.unwrap()` — มันเอาค่าข้างใน `Some(T)` ออกมาให้ตรง ๆ
แต่ถ้าค่าเป็น `None` มันจะ **panic ทันที** (หยุดโปรแกรมทั้งหมด ไม่ใช่แค่คืนค่า error):

```rust
fn main() {
    let some_value: Option<i32> = Some(5);
    println!("unwrap ค่าที่มีอยู่: {}", some_value.unwrap());

    let none_value: Option<i32> = None;
    println!("unwrap ค่าที่ไม่มี: {}", none_value.unwrap());
}
```

บรรทัดแรกทำงานปกติ ได้ผลลัพธ์:

```
unwrap ค่าที่มีอยู่: 5
```

แต่บรรทัดที่สองไม่มีวันรันถึง `println!` เพราะ `.unwrap()` panic ไปก่อนตั้งแต่ตอนประเมินค่า argument ของ `println!`
เอง โปรแกรมจะหยุดทำงานทันทีพร้อมข้อความ (นี่คือ panic message จริงจาก `rustc`/`cargo run`):

```
unwrap ค่าที่มีอยู่: 5

thread 'main' panicked at src/main.rs:6:46:
called `Option::unwrap()` on a `None` value
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

สังเกตว่า panic message ระบุตำแหน่งที่แน่นอน (`src/main.rs:6:46` — บรรทัดและ column ที่เรียก `.unwrap()`) และ
ข้อความ `called \`Option::unwrap()\` on a \`None\` value` — บอกตรง ๆ ว่า **ทำอะไร** (`unwrap()`) แต่ **ไม่ได้บอก
เลยว่าทำไม** ค่านั้นถึงเป็น `None` ในทางธุรกิจ (เช่น "ไม่พบ user", "ยังไม่ได้ตั้งค่า config", "index เกินขอบเขต")
นี่คือข้อจำกัดสำคัญของ `.unwrap()` ที่ `.expect("...")` ช่วยแก้ได้

#### `.expect("...")`: `.unwrap()` ที่มีข้อความอธิบายเอง

`.expect()` ทำงานเหมือน `.unwrap()` เป๊ะ ๆ (คืนค่าถ้าเป็น `Some`, panic ถ้าเป็น `None`) แต่ให้คุณกำหนด **ข้อความ
panic ของตัวเอง** แทนข้อความ generic ของ `.unwrap()`:

```rust
fn main() {
    let config_value: Option<&str> = None;
    let value = config_value.expect("ตัวแปร CONFIG_PATH ต้องถูกตั้งค่าไว้เสมอก่อนเริ่มโปรแกรม");
    println!("{value}");
}
```

```
thread 'main' panicked at src/main.rs:3:30:
ตัวแปร CONFIG_PATH ต้องถูกตั้งค่าไว้เสมอก่อนเริ่มโปรแกรม
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

เทียบสองข้อความ panic ดูให้ชัด — `.unwrap()` ให้คุณได้แค่ `called \`Option::unwrap()\` on a \`None\` value`
ซึ่งบอกแค่ "มีอะไรบางอย่างเป็น `None` ตรงนี้" ส่วน `.expect("ตัวแปร CONFIG_PATH ...")` บอกตรง ๆ ว่า **นี่คือ
`CONFIG_PATH` ที่ไม่ได้ตั้งค่า** — เมื่อโปรแกรมมีจุดเรียก `.unwrap()`/`.expect()` อยู่หลายร้อยจุด (เรื่องปกติใน
โปรเจกต์ใหญ่) การเห็น panic message ที่ระบุ**บริบทที่แท้จริง**ทันทีช่วยประหยัดเวลา debug ได้มาก ไม่ต้องไล่เปิดโค้ด
ไปดูว่าบรรทัดที่ panic นั้นหมายถึงอะไรกันแน่ — นี่คือเหตุผลที่ **นักพัฒนา Rust ที่มีประสบการณ์แทบทุกคนแนะนำให้ใช้
`.expect()` แทน `.unwrap()` เสมอเมื่อต้องเลือกใช้ตัวใดตัวหนึ่ง** เพราะต้นทุนในการเขียน (พิมพ์ข้อความเพิ่มอีกนิดเดียว)
ต่ำกว่าประโยชน์ที่ได้ตอน debug มากในระยะยาว

#### เมื่อไหร่ควรใช้ `.unwrap()`/`.expect()` — และเมื่อไหร่คือ code smell

จุดสำคัญที่สุดของหัวข้อนี้ไม่ใช่แค่ "syntax ของมันคืออะไร" แต่คือ **การตัดสินใจว่าเมื่อไหร่ควรใช้มันในโค้ดจริง**
เพราะทั้งสอง method นี้คือการ **บอก compiler ตรง ๆ ว่า "ฉันมั่นใจ 100% ว่าค่านี้จะไม่เป็น `None` ณ จุดนี้ ถ้าฉันคิดผิด
ให้โปรแกรม panic ไปเลย"** — มันคือการสละความปลอดภัยที่ `Option<T>` มอบให้โดยสมบูรณ์ เพื่อความสะดวกในการเขียน
มาดูกรณีที่ **เหมาะสม** กับที่ **ไม่เหมาะสม**:

**กรณีที่ยอมรับได้ (acceptable):**

1. **ตอน prototype/exploratory coding** — เขียนโค้ดทดลองไอเดียเร็ว ๆ ที่ยังไม่ต้อง production-ready การจัดการ
   `None` อย่างถูกต้องทุกจุดจะทำให้เสียเวลาคิดไปกับสิ่งที่ยังไม่ใช่ประเด็นหลัก
2. **ในโค้ด test** — ถ้า test เขียนไว้ให้ค่าที่ query มาต้องมีอยู่แน่นอน (เพราะ test เตรียมข้อมูลเองในบรรทัดก่อนหน้า)
   การ panic ทันทีเมื่อไม่เป็นไปตามที่คาดคือพฤติกรรมที่ต้องการอยู่แล้ว (test ควร fail ทันทีถ้า assumption ผิด)
3. **Invariant ที่พิสูจน์ได้จริงว่าเป็นไปไม่ได้ที่จะเป็น `None`** — เช่น คุณเพิ่งเช็ค `if !vec.is_empty()` ไปหมาด ๆ
   แล้วเรียก `vec.first().unwrap()` ในบรรทัดถัดมา (รู้แน่นอนว่าไม่ว่างแล้ว) กรณีนี้ `.expect("ตรวจสอบแล้วว่า vec
   ไม่ว่างก่อนหน้านี้")` สื่อสารกับคนอ่านโค้ดว่า "นี่ไม่ใช่การมองข้ามปัญหา แต่เป็นการยืนยันสิ่งที่การันตีไว้แล้ว"

**กรณีที่เป็น code smell (ไม่เหมาะสมในโค้ด production):**

1. **เมื่อค่าที่ query มาจากภายนอกระบบ** (input ผู้ใช้, ผลลัพธ์จาก network request, ข้อมูลจากไฟล์/database) —
   สิ่งเหล่านี้ **ไม่มีทางการันตีได้ล่วงหน้า** ว่าจะเป็น `Some` เสมอ การ `.unwrap()` ตรงนี้แปลว่า "ถ้า input ผิดปกติ
   ให้ทั้งโปรแกรม/service ล่มไปเลย" ซึ่งมักไม่ใช่พฤติกรรมที่ต้องการในระบบจริง (ควรจัดการ `None` แล้วตอบ error
   กลับไปอย่างสุภาพแทน)
2. **เมื่อ "ความมั่นใจ" มาจากการเดา ไม่ใช่การพิสูจน์** — ถ้าคุณคิดว่า "ปกติมันจะมีค่าอยู่แล้วนะ" แต่ไม่มีโค้ดบรรทัด
   ไหนที่การันตีสิ่งนั้นจริง ๆ (เช่นไม่ได้เช็ค `is_empty()` มาก่อน) นี่คือสัญญาณเตือนว่ากำลังเอาความสะดวกมาก่อน
   ความถูกต้อง
3. **เมื่อมีทางเลือกที่ปลอดภัยกว่าและไม่ได้ซับซ้อนขึ้นมากนัก** — หัวข้อ 11.5-11.9 ที่จะเรียนต่อไปมี method
   หลากหลายที่จัดการ `None` ได้อย่างสวยงามโดยไม่ต้อง panic เลย ถ้าเลือกใช้ได้ ควรเลือกใช้ก่อนเสมอ

กฎง่าย ๆ ที่จำได้ทันที: **ถ้าคุณอธิบายไม่ได้ว่า "ทำไมค่านี้จะไม่เป็น `None` ณ จุดนี้" ด้วยเหตุผลที่ชัดเจนพิสูจน์ได้
(ไม่ใช่แค่ความรู้สึก) ห้ามใช้ `.unwrap()`/`.expect()` เด็ดขาด** ให้ใช้ method อื่นที่จัดการทั้งสองกรณีอย่างชัดเจน
แทน ซึ่งเป็นเนื้อหาของหัวข้อต่อไปนี้ทั้งหมด

### 11.5 ดึงค่าแบบไม่ Panic: `.unwrap_or()`, `.unwrap_or_else()`, `.unwrap_or_default()`

เมื่อไหร่ที่ "ไม่มีค่า" ไม่ใช่ข้อผิดพลาดร้ายแรง แต่มี **ค่า default ที่สมเหตุสมผล** ให้ใช้แทนได้เลย เราไม่จำเป็นต้อง
panic หรือเขียน `match` เต็มรูปแบบ — ตระกูล method `unwrap_or*` ทำหน้าที่นี้โดยเฉพาะ

```rust
fn compute_default_discount() -> u32 {
    println!("  (กำลังคำนวณส่วนลด default ... ทำงานหนักจำลองไว้)");
    5
}

fn main() {
    let discount: Option<u32> = Some(20);
    let no_discount: Option<u32> = None;

    // unwrap_or: รับค่า default ตรง ๆ — ค่านี้ถูกคำนวณเสมอไม่ว่า Option จะเป็น Some หรือ None (eager)
    println!("discount.unwrap_or(0) = {}", discount.unwrap_or(0));
    println!("no_discount.unwrap_or(0) = {}", no_discount.unwrap_or(0));

    // unwrap_or_else: รับ closure/ฟังก์ชันที่จะถูกเรียกก็ต่อเมื่อเป็น None เท่านั้น (lazy)
    println!(
        "no_discount.unwrap_or_else(...) = {}",
        no_discount.unwrap_or_else(compute_default_discount)
    );

    // unwrap_or_default: ใช้ค่า default ของ type นั้น ๆ ผ่าน Default trait
    let maybe_count: Option<u32> = None;
    println!("maybe_count.unwrap_or_default() = {}", maybe_count.unwrap_or_default());

    let maybe_name: Option<String> = None;
    println!("maybe_name.unwrap_or_default() = {:?}", maybe_name.unwrap_or_default());
}
```

ผลลัพธ์:

```
discount.unwrap_or(0) = 20
no_discount.unwrap_or(0) = 0
  (กำลังคำนวณส่วนลด default ... ทำงานหนักจำลองไว้)
no_discount.unwrap_or_else(...) = 5
maybe_count.unwrap_or_default() = 0
maybe_name.unwrap_or_default() = ""
```

**อธิบายทีละ method:**

- **`.unwrap_or(default_value)`** — ถ้าเป็น `Some(x)` คืน `x`, ถ้าเป็น `None` คืน `default_value` ที่ระบุไว้
  จุดสำคัญที่สุดที่ต้องเข้าใจคือ **`default_value` ต้องถูกสร้าง/คำนวณขึ้นมาก่อนเสมอ ไม่ว่า `Option` ต้นทางจะเป็น
  `Some` หรือ `None`** — สังเกตจากผลลัพธ์บรรทัดที่ 1 (`discount.unwrap_or(0)`) เราแค่ใส่ literal `0` ไปตรง ๆ
  ซึ่งไม่มีต้นทุนอะไรให้กังวล แต่ถ้า argument นั้นคือการเรียกฟังก์ชันที่ทำงานหนักหรือมี side effect นี่คือจุดที่
  ต้องระวังมาก (จะเห็นตัวอย่างชัดเจนในหัวข้อ "กับดักที่พบบ่อย")
- **`.unwrap_or_else(closure)`** — ทำงานเหมือน `.unwrap_or()` แต่รับ **closure/ฟังก์ชันที่ไม่มี argument** แทน
  ค่าตรง ๆ closure นี้จะถูกเรียก **ก็ต่อเมื่อ `Option` เป็น `None` จริง ๆ เท่านั้น** (lazy evaluation) สังเกตจาก
  ผลลัพธ์ว่าข้อความ `"(กำลังคำนวณส่วนลด default ...)"` ปรากฏขึ้นก็ต่อตอนเรียก `no_discount.unwrap_or_else(...)`
  เท่านั้น (เพราะ `no_discount` เป็น `None`) — ถ้าเราเรียก `discount.unwrap_or_else(compute_default_discount)`
  (ที่ `discount` เป็น `Some(20)`) ฟังก์ชัน `compute_default_discount` จะ**ไม่ถูกเรียกเลย** เพราะไม่จำเป็นต้องใช้
  ค่า default — นี่คือความต่างที่สำคัญที่สุดระหว่างสองตัวนี้ ในโค้ดข้างบนเราส่งชื่อฟังก์ชัน `compute_default_discount`
  ตรง ๆ (ไม่ใส่ `()`) เพราะ `.unwrap_or_else()` ต้องการ "สิ่งที่เรียกได้" (`FnOnce() -> T`) ซึ่งชื่อฟังก์ชันเฉย ๆ
  ก็ถือเป็นสิ่งที่เรียกได้ในตัวมันเองอยู่แล้ว ไม่ต้องเขียนเป็น closure `|| compute_default_discount()` ก็ได้ (ทั้ง
  สองแบบทำงานเหมือนกัน แต่แบบไม่มี `||` กระชับกว่าเมื่อไม่ต้องส่ง argument เพิ่ม)
- **`.unwrap_or_default()`** — ใช้ค่า default ของ type นั้นโดยอัตโนมัติ ผ่าน trait ที่ชื่อ **`Default`** (จะเรียน
  เต็มรูปแบบใน Part หลัง ๆ — ตอนนี้จำแค่ว่า type พื้นฐานส่วนใหญ่ของ Rust implement `Default` ให้มาแล้ว: ตัวเลข
  ทุกชนิด default เป็น `0`, `bool` default เป็น `false`, `String` default เป็นสตริงว่าง `""`, `Vec<T>` default
  เป็น vector ว่าง เป็นต้น) `.unwrap_or_default()` สะดวกมากเมื่อ "ไม่มีค่า" ตีความได้ตรงกับ "ค่าว่าง/ศูนย์" ของ
  type นั้นอย่างเป็นธรรมชาติอยู่แล้ว (เช่น จำนวนสินค้าที่ไม่พบข้อมูล = 0 ชิ้น, ชื่อที่ไม่มีข้อมูล = สตริงว่าง)

สังเกตว่า `.unwrap_or_default()` **ไม่ต้องรับ argument เลย** ต่างจากอีกสองตัว เพราะมันไม่ได้ถามคุณว่า "อยากได้ค่า
default อะไร" — มันตัดสินใจแทนคุณโดยอ้างอิงจาก type เพียงอย่างเดียว (ผ่าน `T::default()`) นี่คือเหตุผลที่มันเหมาะกับ
กรณีทั่วไปที่ไม่ต้องการค่า default แบบพิเศษ แต่ไม่เหมาะกับกรณีที่ต้องการค่า default ที่มีความหมายทางธุรกิจเฉพาะ
(เช่น "ราคา default คือ 100 บาท" — กรณีนี้ต้องใช้ `.unwrap_or(100)` แทน เพราะ `100` ไม่ใช่ค่า default ของ `u32`
ตาม `Default` trait ซึ่งคือ `0`)

### 11.6 แปลงค่าข้างใน `Option<T>` โดยไม่ต้อง Unwrap ก่อน: `.map()` และ `.and_then()`

หลายครั้งเราไม่ได้ต้องการ "เอาค่าออกมาใช้ทันที" แต่ต้องการ **แปลงค่าข้างในให้เป็นอีกรูปแบบหนึ่ง โดยที่ยังคงความเป็น
`Option` เอาไว้** (ถ้าเดิมเป็น `None` ผลลัพธ์ก็ยังเป็น `None` ต่อไป ไม่ต้อง unwrap มาเช็คก่อนแล้วค่อยห่อกลับเข้า
`Option` ใหม่ด้วยมือ) นี่คือจุดที่ `.map()` เข้ามาช่วย

ก่อนไปดูโค้ด ให้รู้จัก syntax สั้น ๆ ที่ชื่อ **closure** ก่อน — `|x| expression` คือการเขียน "ฟังก์ชันนิรนาม" แบบ
รวบรัด อ่านว่า "รับ argument ชื่อ `x` แล้วคำนวณ `expression`" (เทียบเท่ากับ arrow function `x => expression` ใน
JavaScript หรือ lambda `lambda x: expression` ใน Python) เราจะเรียน closure แบบเต็มรูปแบบ (การ capture ตัวแปร
จากภายนอก, `Fn`/`FnMut`/`FnOnce`) ใน **Part 24** — ตอนนี้จำแค่ว่ามันคือวิธีเขียนฟังก์ชันสั้น ๆ ไว้ใช้ครั้งเดียว
ตรงจุดที่ต้องใช้เลย โดยไม่ต้องประกาศฟังก์ชันแยกด้วย `fn` ก่อน

```rust
fn parse_price(input: &str) -> Option<f64> {
    input.trim().parse::<f64>().ok()
}

fn apply_vat(price: f64) -> f64 {
    price * 1.07
}

fn find_discount_percent(price: f64) -> Option<f64> {
    if price >= 1000.0 {
        Some(10.0)
    } else {
        None
    }
}

fn main() {
    // .map(): แปลงค่าข้างใน Option โดยไม่ต้อง unwrap ก่อน — ถ้าเป็น None ก็ยังเป็น None ต่อไป
    let price_with_vat = parse_price("500.0").map(apply_vat);
    println!("{price_with_vat:?}");

    let invalid_with_vat = parse_price("ไม่ใช่ตัวเลข").map(apply_vat);
    println!("{invalid_with_vat:?}");

    // Chain .map() หลายชั้นต่อกันได้เรื่อย ๆ
    let final_price = parse_price("1200")
        .map(apply_vat)
        .map(|p| p.round())
        .unwrap_or(0.0);
    println!("ราคาสุทธิ (ปัดเศษ): {final_price}");

    // .and_then(): ใช้เมื่อ closure เองก็คืน Option<T> — ป้องกันการซ้อน Option<Option<T>>
    let discount = parse_price("1500").and_then(|p| find_discount_percent(p));
    println!("{discount:?}");

    let discount_low_price = parse_price("100").and_then(|p| find_discount_percent(p));
    println!("{discount_low_price:?}");

    // เทียบให้เห็นภาพ: ถ้าใช้ .map() แทน .and_then() กับ closure ที่คืน Option
    // จะได้ Option<Option<f64>> ซ้อนกันสองชั้น ซึ่งใช้งานต่อลำบากโดยไม่มีประโยชน์
    let nested = parse_price("1500").map(|p| find_discount_percent(p));
    println!("{nested:?}");
}
```

ผลลัพธ์:

```
Some(535.0)
None
ราคาสุทธิ (ปัดเศษ): 1284
Some(10.0)
None
Some(Some(10.0))
```

**อธิบายทีละส่วน:**

- `parse_price("500.0").map(apply_vat)` — `parse_price` คืน `Option<f64>` (`Some(500.0)` ในกรณีนี้) `.map(apply_vat)`
  จะ **เรียก `apply_vat` กับค่าข้างในให้เอง ก็ต่อเมื่อเป็น `Some` เท่านั้น** แล้วห่อผลลัพธ์กลับเป็น `Some(...)` ใหม่
  ให้อัตโนมัติ (`500.0 * 1.07 = 535.0` จึงได้ `Some(535.0)`) — เราไม่ต้องเขียน `match`/`if let` เพื่อแกะค่าออกมา
  ก่อนแล้วค่อยห่อกลับเข้า `Some` ด้วยมือเองเลย
- `parse_price("ไม่ใช่ตัวเลข").map(apply_vat)` — เมื่อ `parse_price` คืน `None` (parse ไม่สำเร็จ) `.map()` จะ
  **ข้ามการเรียก `apply_vat` ไปเลย** และคืน `None` ต่อไปเฉย ๆ — นี่คือพฤติกรรมสำคัญที่สุดของ `.map()`: มันไม่มีวัน
  ทำให้เกิด panic หรือ error ใหม่ ต่อให้ค่าตั้งต้นเป็น `None` ก็ตาม ผลลัพธ์จะแค่ "ไหลผ่าน" เป็น `None` ต่อไปเรื่อย ๆ
- Chain `.map(apply_vat).map(|p| p.round()).unwrap_or(0.0)` — แสดงให้เห็นว่า `.map()` ต่อกันได้หลายชั้น แต่ละชั้น
  แปลงค่าไปอีกขั้นหนึ่ง จนจบด้วย `.unwrap_or(0.0)` เพื่อดึงค่าจริงออกมาแบบไม่ panic ในขั้นตอนสุดท้าย — นี่คือ
  idiom ที่พบบ่อยมากในโค้ด Rust จริง: **ประมวลผลข้อมูลทั้ง pipeline ผ่าน `Option` โดยไม่ต้อง unwrap กลางทางเลย
  แล้วค่อย unwrap/ใส่ default แค่ครั้งเดียวตอนจบ**
- `.and_then(|p| find_discount_percent(p))` — จุดสำคัญคือ `find_discount_percent` **คืนค่าเป็น `Option<f64>`
  เอง ไม่ใช่ `f64` ตรง ๆ** ถ้าเราใช้ `.map(|p| find_discount_percent(p))` แทน ผลลัพธ์ที่ได้จะเป็น `Option<Option<f64>>`
  (ดูตัวอย่างสุดท้าย `nested` ที่ได้ `Some(Some(10.0))`) — ซึ่งซ้อนกันสองชั้นแบบไม่มีประโยชน์ เพราะ `Option<Option<T>>`
  ใช้งานยากกว่า `Option<T>` เปล่า ๆ (ต้อง unwrap สองรอบ) `.and_then()` แก้ปัญหานี้โดย **"แบน" (flatten) ชั้นซ้อนออก
  ให้อัตโนมัติ**: ถ้าค่าตั้งต้นเป็น `Some(x)`, มันจะเรียก closure แล้ว**คืนผลลัพธ์ของ closure ตรง ๆ** (ซึ่งเป็น
  `Option<f64>` อยู่แล้ว ไม่ต้องห่อ `Some` เพิ่ม) แต่ถ้าค่าตั้งต้นเป็น `None` มันจะคืน `None` ทันทีโดยไม่เรียก closure
  เลย (เหมือน `.map()`) รูปแบบนี้บางภาษาเรียกว่า **"flat-map"** และในทาง functional programming เรียกว่า
  **monadic bind** (ไม่ต้องจำศัพท์นี้ก็ได้ แต่ถ้าเคยเจอใน Haskell/Scala จะคุ้นแนวคิดทันที)

**กฎจำง่าย ๆ ในการเลือกระหว่าง `.map()` กับ `.and_then()`**: ให้ดูที่ **return type ของ closure/ฟังก์ชันที่จะส่ง
เข้าไป** — ถ้าฟังก์ชันนั้นคืนค่า `T` ธรรมดา (ไม่ใช่ `Option`) ให้ใช้ `.map()`, แต่ถ้าฟังก์ชันนั้นคืนค่าเป็น
`Option<T>` เองอยู่แล้ว (เช่นเป็นอีกฟังก์ชันค้นหาที่อาจไม่พบผลลัพธ์) ให้ใช้ `.and_then()` เสมอ เพื่อไม่ให้เกิด
`Option<Option<T>>` ที่ซ้อนกันโดยไม่จำเป็น

### 11.7 ถามคำถามเกี่ยวกับ `Option<T>` โดยไม่ยึดค่า: `.is_some()`, `.is_none()`, `.as_ref()`, `.as_mut()`

บางครั้งเราต้องการแค่ **"ถามคำถาม" ว่า `Option` มีค่าอยู่หรือไม่ โดยไม่ต้องดึงค่าออกมาใช้เลย** หรือต้องการ **"ยืมดู"
ค่าข้างในชั่วคราวโดยไม่ยึด ownership ออกไป** ทั้งสองความต้องการนี้มี method รองรับโดยเฉพาะ

```rust
struct Profile {
    nickname: Option<String>,
}

fn main() {
    let profile = Profile {
        nickname: Some("มานะ".to_string()),
    };

    // is_some / is_none: เช็คสถานะ ได้คำตอบเป็น bool ตรง ๆ ไม่แตะค่าข้างในเลย
    println!("มี nickname หรือไม่: {}", profile.nickname.is_some());
    println!("ไม่มี nickname หรือไม่: {}", profile.nickname.is_none());

    // as_ref: แปลง &Option<T> เป็น Option<&T> — ยืมดูค่าข้างในโดยไม่ move และไม่ clone
    let nickname_ref: Option<&String> = profile.nickname.as_ref();
    match nickname_ref {
        Some(n) => println!("nickname (ยืมมาดูเฉย ๆ): {n}"),
        None => println!("ไม่มี nickname"),
    }

    // profile.nickname ยังเป็นเจ้าของค่าเดิมอยู่ครบ เพราะเราไม่ได้ยึด ownership ออกไปเลย
    println!("nickname ยังอยู่ที่ profile: {:?}", profile.nickname);

    let mut counter: Option<i32> = Some(10);

    // as_mut: แปลง &mut Option<T> เป็น Option<&mut T> เพื่อแก้ค่าข้างในตรง ๆ โดยไม่ต้อง unwrap/reassign ทั้งตัว
    if let Some(value) = counter.as_mut() {
        *value += 5;
    }
    println!("counter หลังแก้ผ่าน as_mut: {counter:?}");
}
```

ผลลัพธ์:

```
มี nickname หรือไม่: true
ไม่มี nickname หรือไม่: false
nickname (ยืมมาดูเฉย ๆ): มานะ
nickname ยังอยู่ที่ profile: Some("มานะ")
counter หลังแก้ผ่าน as_mut: Some(15)
```

**อธิบายทีละ method พร้อมผลกระทบเชิง ownership:**

- **`.is_some()` / `.is_none()`** — คืน `bool` ตรง ๆ โดยไม่แตะต้องค่าข้างในเลย (ไม่ move, ไม่ borrow อะไรเพิ่มเติม
  นอกจากการยืม `&self` ชั่วคราวตอนเรียก) เหมาะมากกับตอนที่ logic ต้องการแค่รู้ "มี/ไม่มี" โดยไม่สนใจค่าจริง เช่นใช้
  ใน `if profile.nickname.is_some() { ... }` — แต่ข้อสังเกตสำคัญคือ **ถ้าคุณจะใช้ค่าจริงต่อในบล็อกนั้นด้วย
  มักจะเขียนด้วย `if let Some(n) = ...` ตรง ๆ กระชับกว่า** (ตามที่เรียนใน Part 10) `.is_some()`/`.is_none()`
  เหมาะกับกรณีที่ไม่ต้องใช้ค่าจริงเลยจริง ๆ
- **`.as_ref()`** — นี่คือ method ที่สำคัญที่สุดในหัวข้อนี้ มันแปลง `&Option<T>` (reference ไปยัง `Option` ทั้งกล่อง)
  ให้เป็น `Option<&T>` (Option ที่ถ้ามีค่า ค่านั้นจะเป็น reference ไปยังข้อมูลข้างใน ไม่ใช่ข้อมูลตัวจริง) สังเกต
  ว่า `nickname_ref` มี type เป็น `Option<&String>` ไม่ใช่ `Option<String>` — **เราได้แค่ "ยืมดู" `String` ข้างใน
  ผ่าน reference เท่านั้น ไม่ได้ clone หรือ move มันออกมาเลย** ทำให้ `profile.nickname` ยังคงเป็นเจ้าของค่าเดิม
  ครบถ้วนหลังจากนั้น (ตามที่พิสูจน์ด้วยการ print `profile.nickname` อีกครั้งในบรรทัดต่อมา) เทียบกับถ้าเราเรียก
  `profile.nickname.unwrap()` ตรง ๆ (โดยที่ `profile.nickname` ไม่ใช่ `mut` และอยู่หลัง field access ปกติ) จะเกิด
  การ **ยึด ownership ของ `String` ข้างในออกจาก `profile` ไปเลย** ทำให้ `profile.nickname` ใช้งานต่อไม่ได้อีก
  (เราจะเห็น error จริงของกรณีนี้ในหัวข้อ "กับดักที่พบบ่อย") `.as_ref()` จึงเป็นเครื่องมือสำคัญที่สุดเมื่อคุณ
  ต้องการ "ดูค่าข้างใน `Option` โดยไม่ทำลาย ownership ของเจ้าของเดิม" ซึ่งเกิดขึ้นบ่อยมากเมื่อทำงานกับ field ของ
  struct ที่คุณแค่ยืมมา (`&self`) หรือเมื่อวนลูปด้วย `for item in &collection`
- **`.as_mut()`** — เหมือน `.as_ref()` แต่ให้ **สิทธิ์แก้ไข** ค่าข้างใน: แปลง `&mut Option<T>` เป็น `Option<&mut T>`
  ในตัวอย่าง `counter.as_mut()` คืน `Option<&mut i32>` ทำให้เราใช้ `if let Some(value) = counter.as_mut() { *value
  += 5; }` แก้ค่า `i32` ข้างในได้ตรง ๆ ผ่าน `*value` (การ dereference เพื่อเข้าถึงค่าจริงที่ `value` ชี้ไป — ตามหลัก
  Part 7) โดยไม่ต้อง unwrap ค่าเดิมออกมา คำนวณ แล้ว reassign กลับเข้า `counter` ทั้งตัวใหม่ (ซึ่งจะยาวกว่าและ
  เสี่ยงพลาดมากกว่า) — `.as_mut()` เหมาะมากเมื่อ field เป็น `Option<T>` ที่ต้องแก้ไขค่าข้างในบ่อย ๆ โดยไม่อยาก
  เขียน `match`/`if let` แล้ว reassign ใหม่ทุกครั้ง

### 11.8 รวม `Option<T>` หลายตัวเข้าด้วยกัน: `.or()`, `.or_else()`, `.filter()`, `.zip()`

นอกจากแปลงค่าและถามสถานะแล้ว `Option<T>` ยังมี method สำหรับ "รวม" หรือ "กรอง" Option เข้าด้วยกันในรูปแบบต่าง ๆ
ที่พบบ่อยในโค้ดจริง:

```rust
fn main() {
    // .or(): ถ้าตัวแรกเป็น None ให้ใช้ตัวสำรองที่เตรียมไว้ล่วงหน้าแล้ว (eager — สร้าง fallback ไว้ก่อนเสมอ)
    let primary: Option<&str> = None;
    let fallback: Option<&str> = Some("ค่าสำรอง");
    println!("{:?}", primary.or(fallback));

    // .or_else(): เหมือน .or() แต่รับ closure — คำนวณค่าสำรองแบบ lazy เฉพาะเมื่อจำเป็นจริง ๆ
    let cached_config: Option<String> = None;
    let loaded = cached_config.or_else(|| Some("โหลดจากไฟล์ config".to_string()));
    println!("{loaded:?}");

    // .filter(): เก็บค่าไว้ต่อก็ต่อเมื่อผ่านเงื่อนไขที่กำหนด ไม่ผ่านก็กลายเป็น None
    let age: Option<u32> = Some(15);
    let adult_age = age.filter(|&a| a >= 18);
    println!("{adult_age:?}");

    let age2: Option<u32> = Some(25);
    let adult_age2 = age2.filter(|&a| a >= 18);
    println!("{adult_age2:?}");

    // .zip(): รวมสอง Option เข้าด้วยกันเป็น Option<(A, B)> — ได้ Some ก็ต่อเมื่อทั้งคู่เป็น Some
    let name: Option<&str> = Some("สมชาย");
    let score: Option<u32> = Some(88);
    println!("{:?}", name.zip(score));

    let missing_score: Option<u32> = None;
    println!("{:?}", name.zip(missing_score));
}
```

ผลลัพธ์:

```
Some("ค\u{e48}าสำรอง")
Some("โหลดจากไฟล\u{e4c} config")
None
Some(25)
Some(("สมชาย", 88))
None
```

(หมายเหตุ: บรรทัดที่ 1-2 แสดง `\u{e48}` และ `\u{e4c}` เพราะ `Debug` ของ `str`/`String` escape ตัวอักษร Unicode
combining mark เสมอ — สระ/วรรณยุกต์ไทยหลายตัวเข้าข่ายนี้ — ตามที่อธิบายไว้แล้วอย่างละเอียดใน Part 9 หัวข้อ
"ข้อความไทยกับ `{:?}`" นี่ไม่ใช่บั๊ก แค่พฤติกรรมจริงของ `Debug`)

**อธิบายทีละ method:**

- **`.or(other)`** — ถ้าตัวเองเป็น `Some` คืนตัวเอง ถ้าเป็น `None` คืน `other` ที่ระบุไว้ (เหมือน `.unwrap_or()`
  แต่**ไม่ unwrap** — ผลลัพธ์ยังเป็น `Option<T>` อยู่ ไม่ใช่ `T` ตรง ๆ) เหมาะกับกรณีที่ต้องการ "chain แหล่งข้อมูล
  สำรองหลายแหล่ง" เช่น `env_config.or(file_config).or(default_config)` ที่ยังต้อง handle `None` ต่อในขั้นตอน
  ถัดไปอยู่ (ต่างจาก `.unwrap_or()` ที่จบด้วยค่า `T` แน่นอนแล้ว)
- **`.or_else(closure)`** — เหมือน `.or()` แต่รับ closure แบบ lazy (คำนวณก็ต่อเมื่อจำเป็น) หลักการเดียวกับ
  `.unwrap_or_else()` ในหัวข้อ 11.5 ทุกประการ เหมาะกับกรณีที่การสร้างค่าสำรองมีต้นทุนสูง (เช่นต้องเปิดไฟล์อ่าน)
- **`.filter(predicate)`** — รับ closure ที่คืน `bool` (เรียกว่า **predicate** — ฟังก์ชันที่ตอบ `true`/`false`)
  ถ้าค่าเป็น `Some(x)` **และ** `predicate(x)` เป็น `true` จะคงเป็น `Some(x)` ต่อไป แต่ถ้า `predicate(x)` เป็น
  `false` (หรือถ้าตั้งต้นเป็น `None` อยู่แล้ว) ผลลัพธ์จะกลายเป็น `None` — สังเกตว่า `age.filter(|&a| a >= 18)`
  ใน closure เขียน `|&a|` (มี `&` นำหน้า) เพราะ `.filter()` ส่ง reference ของค่าข้างใน (`&u32`) เข้าไปให้ predicate
  ไม่ใช่ค่าตรง ๆ การเขียน `&a` ใน pattern ของ parameter คือการ destructure reference นั้นออกมาเป็นค่า `u32`
  ธรรมดาทันที (ทำงานคล้าย match ergonomics ที่เรียนใน Part 10) ถ้าไม่ใส่ `&` จะต้องเขียน `|a: &u32| *a >= 18` แทน
  ซึ่งยาวกว่า `.filter()` เหมาะมากกับสถานการณ์ "มีค่าอยู่ก็จริง แต่ต้องเช็คว่าค่านั้น 'ผ่านเงื่อนไข' ทางธุรกิจ
  ด้วยหรือไม่" เช่นตัวอย่างนี้: "มีอายุอยู่ก็จริง แต่ต้อง ≥ 18 ปีถึงจะถือว่าเป็นผู้ใหญ่"
- **`.zip(other)`** — รวม `Option<A>` กับ `Option<B>` เป็น `Option<(A, B)>` เดียว — ได้ `Some((a, b))` ก็ต่อเมื่อ
  **ทั้งสองตัวเป็น `Some` พร้อมกัน** เท่านั้น ถ้าตัวใดตัวหนึ่งเป็น `None` ผลลัพธ์รวมจะเป็น `None` ทันที เหมาะกับ
  กรณีที่ต้องใช้ค่าสองตัวคู่กันเสมอ (เช่น "ชื่อ" กับ "คะแนน" ต้องมีครบทั้งคู่ถึงจะแสดงผลได้ ถ้าอย่างใดอย่างหนึ่งขาด
  ไปก็ยังแสดงผลไม่ได้อยู่ดี)

#### ตัวอย่างจริง: ลำดับความสำคัญของค่า config หลายแหล่ง

รูปแบบที่พบบ่อยที่สุดของ `.or_else()` ในโค้ด production คือการไล่หาค่า config จาก**หลายแหล่งตามลำดับความสำคัญ**
เช่น "ลองหาจาก environment variable ก่อน ถ้าไม่มีค่อยลองหาจากไฟล์ config ถ้ายังไม่มีอีกค่อยใช้ค่า default ที่
ฝังไว้ในโค้ด" — เขียนเป็น chain เดียวได้อย่างสวยงามโดยไม่ต้องมี `if`/`else if` ซ้อนกันหลายชั้น:

```rust
fn from_env(key: &str) -> Option<String> {
    // จำลองว่าอ่านจาก environment variable แล้วไม่พบเลย
    let _ = key;
    None
}

fn from_config_file(key: &str) -> Option<String> {
    if key == "log_level" {
        Some("debug".to_string())
    } else {
        None
    }
}

fn hardcoded_default(key: &str) -> String {
    match key {
        "log_level" => "info".to_string(),
        _ => "unknown".to_string(),
    }
}

fn resolve_setting(key: &str) -> String {
    from_env(key)
        .or_else(|| from_config_file(key))
        .unwrap_or_else(|| hardcoded_default(key))
}

fn main() {
    // ไม่มีใน env, มีใน config file -> ได้ค่าจาก config file
    println!("log_level = {}", resolve_setting("log_level"));

    // ไม่มีทั้ง env และ config file -> ตกไปที่ hardcoded default
    println!("theme = {}", resolve_setting("theme"));
}
```

ผลลัพธ์:

```
log_level = debug
theme = unknown
```

สังเกตว่า `resolve_setting` อ่านเป็นเส้นตรงจากบนลงล่างตามลำดับความสำคัญที่ต้องการเป๊ะ ๆ: **ลองแหล่งที่สำคัญที่สุด
ก่อน (`from_env`) ถ้าไม่มีค่อยลองแหล่งถัดไป (`from_config_file`) ผ่าน `.or_else()` และถ้าไม่มีเลยทุกแหล่งค่อยจบด้วย
ค่า fallback สุดท้ายผ่าน `.unwrap_or_else()`** ทั้งสอง method ในที่นี้เป็น **lazy** ทั้งคู่ (ใช้ `.or_else()` ไม่ใช่
`.or()`, ใช้ `.unwrap_or_else()` ไม่ใช่ `.unwrap_or()`) เพราะการอ่านไฟล์ config หรือคำนวณค่า default อาจมีต้นทุน
ที่ไม่อยากให้เกิดขึ้นถ้าไม่จำเป็น (เช่น ถ้า `from_env` เจอค่าอยู่แล้ว ก็ไม่มีเหตุผลต้องเสียเวลาเปิดไฟล์ config อ่าน
เพิ่มเลย) — นี่คือตัวอย่างที่แสดงให้เห็นว่าหลักการ eager/lazy จากหัวข้อ 11.5 นำไปใช้ตัดสินใจเลือก method ได้จริง
ไม่ใช่แค่ทฤษฎี

### 11.9 แปลง `Option<T>` เป็น `Result<T, E>`: `.ok_or()` และ `.ok_or_else()`

Part 10 แนะนำ `Result<T, E>` ไปสั้น ๆ แล้วว่าคล้าย `Option<T>` มาก เพียงแต่กรณี "ไม่สำเร็จ" (`Err`) เก็บ **เหตุผล**
ของความล้มเหลวไว้ด้วย ต่างจาก `None` ที่บอกแค่ว่า "ไม่มีค่า" โดยไม่มีรายละเอียดใด ๆ เพิ่ม — เมื่อไหร่ที่คุณกำลัง
propagate ค่า `None` ขึ้นไปให้ผู้เรียกใช้ฟังก์ชันจัดการ และผู้เรียกใช้ **จำเป็นต้องรู้ว่า "ทำไม" มันถึงไม่มีค่า**
(ไม่ใช่แค่รู้ว่ามันไม่มี) นี่คือสัญญาณว่าถึงเวลาแปลง `Option<T>` เป็น `Result<T, E>` แล้ว — `.ok_or()`/`.ok_or_else()`
ทำหน้าที่นี้โดยเฉพาะ

```rust
fn get_setting(name: &str) -> Option<&'static str> {
    match name {
        "theme" => Some("dark"),
        "language" => Some("th"),
        _ => None,
    }
}

fn main() {
    // .ok_or_else(): สร้างค่า error แบบ lazy (คำนวณก็ต่อเมื่อเป็น None จริง ๆ) — เหมาะกับ error ที่ต้องใช้ format!
    let theme: Result<&str, String> =
        get_setting("theme").ok_or_else(|| "ไม่พบการตั้งค่า theme".to_string());
    println!("{theme:?}");

    let missing: Result<&str, String> =
        get_setting("font_size").ok_or_else(|| "ไม่พบการตั้งค่า font_size".to_string());
    println!("{missing:?}");

    // .ok_or(): ค่า error สร้างไว้ล่วงหน้าแล้ว (eager) เหมาะกับ error ที่เป็น literal ราคาถูก
    let timeout: Result<u32, &str> = None.ok_or("ไม่พบค่า timeout ที่ตั้งไว้");
    println!("{timeout:?}");
}
```

ผลลัพธ์:

```
Ok("dark")
Err("ไม\u{e48}พบการต\u{e31}\u{e49}งค\u{e48}า font_size")
Err("ไม\u{e48}พบค\u{e48}า timeout ท\u{e35}\u{e48}ต\u{e31}\u{e49}งไว\u{e49}")
```

**อธิบายทีละ method:**

- **`.ok_or(err)`** — ถ้าเป็น `Some(x)` คืน `Ok(x)`, ถ้าเป็น `None` คืน `Err(err)` โดย `err` ต้องสร้างไว้ล่วงหน้า
  แล้วเสมอ (eager evaluation — หลักการเดียวกับ `.unwrap_or()`) เหมาะกับกรณีที่ค่า error เป็น literal ธรรมดาที่
  ไม่มีต้นทุนการสร้างสูง (เช่น `&str` คงที่, หรือ enum variant ที่ไม่มีข้อมูลติดตัว)
- **`.ok_or_else(closure)`** — เหมือน `.ok_or()` แต่รับ closure แบบ lazy — สร้างค่า error ก็ต่อเมื่อเป็น `None`
  จริง ๆ เท่านั้น เหมาะมากเมื่อค่า error ต้องสร้างด้วย `format!`/`String::from` หรือมีการคำนวณเพิ่มเติม (เช่นใน
  ตัวอย่าง `"ไม่พบการตั้งค่า {name}".to_string()` ที่ต้อง allocate `String` ใหม่ — ไม่มีประโยชน์ที่จะ allocate
  มันทุกครั้งถ้าค่าที่หาส่วนใหญ่เจอเป็น `Some` อยู่แล้ว)

ทำไมการแปลงนี้สำคัญ? ลองเทียบสองสถานการณ์: ฟังก์ชันที่คืน `Option<&Setting>` เมื่อ caller เจอ `None` จะรู้แค่ว่า
"ไม่มี" แต่ฟังก์ชันที่คืน `Result<&Setting, String>` เมื่อ caller เจอ `Err("ไม่พบการตั้งค่า theme")` จะรู้ทันที
**ว่าอะไรที่ขาดไป** — ข้อมูลนี้มีประโยชน์มากตอนแสดง error message ให้ผู้ใช้เห็น หรือตอนเขียน log สำหรับ debug
นี่คือสะพานเชื่อมไปสู่ **Part 12** ที่จะเจาะลึก `Result<T, E>` และการจัดการ error แบบเต็มรูปแบบ — ตอนนี้จำแค่ว่า
`.ok_or()`/`.ok_or_else()` คือประตูที่เชื่อมโลกของ `Option<T>` (ไม่มีคำอธิบาย) เข้ากับโลกของ `Result<T, E>`
(มีคำอธิบายเสมอ) ได้อย่างเป็นธรรมชาติ

### 11.10 `?` Operator กับฟังก์ชันที่คืน `Option<T>`

Part 10 ยังไม่ได้พูดถึง `?` operator เลย (จะเจาะลึกเต็มรูปแบบคู่กับ `Result<T, E>` ใน **Part 12**) แต่มีจุดหนึ่งที่
ควรรู้ไว้ตั้งแต่ตอนนี้: **`?` ใช้กับ `Option<T>` ได้เช่นกัน ไม่ใช่แค่กับ `Result<T, E>`** — เมื่อใช้ในฟังก์ชันที่คืน
`Option<T>`, `?` จะทำสิ่งนี้ให้อัตโนมัติ: **ถ้าค่าเป็น `Some(x)` ให้ดึง `x` ออกมาใช้ต่อในบรรทัดนั้นทันที (ไม่ต้อง
unwrap ด้วยมือ) แต่ถ้าเป็น `None` ให้ `return None;` ออกจากฟังก์ชันทั้งหมดทันที**

```rust
fn first_char_uppercase(input: &str) -> Option<char> {
    let first = input.chars().next()?; // ถ้า None ให้ return None ออกจากฟังก์ชันทันที
    Some(first.to_ascii_uppercase())
}

fn last_two_digits_sum(input: &str) -> Option<u32> {
    let len = input.len();
    if len < 2 {
        return None;
    }
    let last_two = &input[len - 2..];
    let n: u32 = last_two.parse().ok()?; // ? ใช้กับ Option ได้ตรงในฟังก์ชันที่คืน Option
    Some(n / 10 + n % 10)
}

fn main() {
    println!("{:?}", first_char_uppercase("rust"));
    println!("{:?}", first_char_uppercase(""));

    println!("{:?}", last_two_digits_sum("order42"));
    println!("{:?}", last_two_digits_sum("x"));
    println!("{:?}", last_two_digits_sum("orderab"));
}
```

ผลลัพธ์:

```
Some('R')
None
Some(6)
None
None
```

**อธิบายทีละส่วน:**

- `input.chars().next()?` — `.chars()` (จะเรียนเต็มใน Part 14) คืน iterator ของตัวอักษร, `.next()` ดึงตัวอักษรตัว
  แรกออกมาเป็น `Option<char>` (`None` ถ้าสตริงว่างเปล่า) การเขียน `?` ต่อท้ายเทียบเท่ากับการเขียน:
  ```rust
  let first = match input.chars().next() {
      Some(c) => c,
      None => return None,
  };
  ```
  แต่กระชับกว่ามาก — สังเกตว่า `first_char_uppercase("")` คืน `None` ทันทีโดยไม่ไปถึงบรรทัด `Some(first...)`
  เลย เพราะ `?` ดัก `None` แล้ว `return None;` ออกจากฟังก์ชันไปตั้งแต่บรรทัดแรก
- `last_two.parse().ok()?` — `.parse()` คืน `Result<u32, _>` (จะเรียนเต็มใน Part 12) `.ok()` แปลง `Result<T, E>`
  เป็น `Option<T>` (ทิ้งรายละเอียด error ไป เก็บไว้แค่ "สำเร็จหรือไม่") แล้ว `?` ดึงค่าออกมาถ้าเป็น `Some` หรือ
  return `None` ทันทีถ้า parse ไม่สำเร็จ (`orderab` -> `"ab"` parse เป็น `u32` ไม่ได้ -> `None`)
- ทั้งสองฟังก์ชันนี้ **ต้องประกาศ return type เป็น `Option<T>` เท่านั้น** จึงใช้ `?` แบบนี้ได้ — ถ้าฟังก์ชันคืน
  `u32`/`String`/type อื่นตรง ๆ (ไม่ใช่ `Option`/`Result`) จะ compile ไม่ผ่านทันที ซึ่งเราจะเห็น error จริงของ
  กรณีนี้ในหัวข้อ "กับดักที่พบบ่อย"

`?` ใน context ของ `Option<T>` มีประโยชน์มากที่สุดเมื่อฟังก์ชันต้อง "unwrap หลายขั้นตอนต่อกัน" ที่แต่ละขั้นตอนอาจ
เป็น `None` ได้ — เขียนด้วย `?` ทำให้โค้ดอ่านเป็นเส้นตรง (คล้ายกับที่ `let else` ทำใน Part 10) โดยไม่ต้อง `match`/
`if let` ซ้อนกันหลายชั้นให้อ่านยาก Part 12 จะแสดงให้เห็นว่า `?` ทรงพลังกว่านี้อีกมากเมื่อใช้กับ `Result<T, E>`
ที่ต้อง propagate ทั้งค่าและ error ไปพร้อมกัน

### 11.11 `Option<&T>` vs `&Option<T>`: รูปร่างสองแบบที่มักสับสน

นี่คือจุดที่มือใหม่ Rust สับสนบ่อยที่สุดเมื่อทำงานกับ `Option<T>` ร่วมกับ ownership/borrowing — **`Option<&T>`**
กับ **`&Option<T>`** ดูคล้ายกันมาก (มีทั้ง `Option`, `&`, และ `T` ปนกัน) แต่มีความหมายต่างกันโดยสิ้นเชิง:

- **`Option<&T>`** อ่านว่า "ตัวเลือกของ reference" — คือ `Option` ที่ **ถ้ามีค่า ค่านั้นจะเป็น reference ไปยัง
  `T`** ตัวอย่างที่เจอบ่อยที่สุดคือผลลัพธ์จาก `.first()`, `.get()`, `.as_ref()` ที่เรียนมาแล้วในหัวข้อก่อนหน้า
- **`&Option<T>`** อ่านว่า "reference ไปยังตัวเลือก" — คือ reference ที่ชี้ไปยัง**กล่อง `Option<T>` ทั้งกล่อง**
  (ตัวกล่องเองอาจเป็น `Some(T)` หรือ `None` ก็ได้ แต่เรามีแค่สิทธิ์ยืมดูกล่องนั้น ไม่ได้ยึด ownership) พบบ่อยที่สุด
  ตอนรับ `&self` ของ struct ที่มี field เป็น `Option<T>`, หรือตอนวนลูปด้วย `for item in &collection` ที่ `item`
  เป็น field ชนิด `Option<T>`

```rust
struct Session {
    user_note: Option<String>,
}

// รับ &Option<T>: อ้างอิงถึงกล่อง Option ทั้งกล่อง (เช่น field ของ struct ที่ยืมมา)
fn describe_ref_to_option(note: &Option<String>) -> String {
    match note {
        Some(text) => format!("มีบันทึกไว้ว่า: {text}"),
        None => "ไม่มีบันทึกไว้".to_string(),
    }
}

// รับ Option<&T>: ตัวเลือกที่ (ถ้ามี) จะเป็น reference ไปยังค่า — พบบ่อยจาก .first()/.get()/.as_ref()
fn describe_option_of_ref(note: Option<&String>) -> String {
    match note {
        Some(text) => format!("มีบันทึกไว้ว่า: {text}"),
        None => "ไม่มีบันทึกไว้".to_string(),
    }
}

fn main() {
    let session = Session {
        user_note: Some("ลูกค้าต้องการติดต่อกลับ".to_string()),
    };

    // &session.user_note มี type เป็น &Option<String>
    println!("{}", describe_ref_to_option(&session.user_note));

    // session.user_note.as_ref() แปลงจาก &Option<String> เป็น Option<&String>
    println!("{}", describe_option_of_ref(session.user_note.as_ref()));
}
```

ผลลัพธ์:

```
มีบันทึกไว้ว่า: ลูกค้าต้องการติดต่อกลับ
มีบันทึกไว้ว่า: ลูกค้าต้องการติดต่อกลับ
```

ทั้งสองฟังก์ชันให้ผลลัพธ์เหมือนกัน (เพราะทั้งคู่แค่ต้องการ "อ่าน" ค่าข้างในเฉย ๆ) แต่ **type ของ parameter ต่างกัน
โดยสิ้นเชิง** — `describe_ref_to_option` รับ `&Option<String>` (ต้องส่ง `&session.user_note` เข้าไปตรง ๆ) ส่วน
`describe_option_of_ref` รับ `Option<&String>` (ต้องแปลงด้วย `.as_ref()` ก่อน ไม่สามารถส่ง `&session.user_note`
เข้าไปตรง ๆ ได้ — จะเกิด type mismatch ซึ่งเราจะเห็น error จริงในหัวข้อ "กับดักที่พบบ่อย")

**เมื่อไหร่ควรออกแบบฟังก์ชันให้รับแบบไหน?** — ในทางปฏิบัติ **`Option<&T>` เป็นรูปแบบที่นิยมใช้มากกว่า** เมื่อ
ออกแบบ API เอง เพราะมันคือ type ที่ `.as_ref()`, `.first()`, `.get()` คืนมาให้อยู่แล้วโดยธรรมชาติ และการ `match`
บน `Option<&T>` ตรง ๆ ก็ทำงานได้อย่างเป็นธรรมชาติโดยไม่ต้องพึ่ง match ergonomics เพิ่มเติมแบบ `&Option<T>` (ที่
ต้อง match บน reference ของ enum อีกที) `&Option<T>` มักจะโผล่มาแบบ "ให้ต้องรับมือ" มากกว่า "ให้ตั้งใจออกแบบ" เช่น
ตอนที่คุณมี `&self` ของ struct ที่มี field เป็น `Option<T>` อยู่แล้ว — ในกรณีนั้น `&self.some_field` โดยธรรมชาติ
จะมี type เป็น `&Option<T>` และคุณมักจะแปลงมันเป็น `Option<&T>` ด้วย `.as_ref()` ทันทีก่อนส่งต่อหรือประมวลผล
เพื่อให้ทำงานร่วมกับ method อื่น ๆ ของ `Option` (`.map()`, `.filter()`, ...) ได้สะดวกกว่า

### 11.12 ตัวอย่างจริง: ค้นหาสินค้าในคลังด้วย SKU

มาปิดท้ายบทนี้ด้วยตัวอย่างที่รวมทุกอย่างที่เรียนมาเข้าด้วยกัน — ระบบค้นหาสินค้าด้วยรหัส SKU (Stock Keeping Unit
รหัสเฉพาะของสินค้าแต่ละตัวที่ใช้ในระบบคลังสินค้าจริง) จาก catalog ขนาดเล็กที่เก็บเป็น **array คงที่ของ struct**
(ทวนจาก Part 9) เนื่องจาก `Vec<T>` และ `HashMap<K, V>` ยังไม่ได้เรียนอย่างเป็นทางการ (จะเรียนใน Part 13 และ
Part 15) เราจะค้นหาด้วย **loop ธรรมดา** เหมือนที่ทำใน `find_user_by_id` ตอนต้นบท

```rust
#[derive(Debug)]
struct Product {
    sku: &'static str,
    name: &'static str,
    price_cents: u32,
    stock: u32,
}

const CATALOG: [Product; 4] = [
    Product { sku: "A100", name: "เมาส์ไร้สาย", price_cents: 29900, stock: 12 },
    Product { sku: "B200", name: "คีย์บอร์ดกลไก", price_cents: 89900, stock: 0 },
    Product { sku: "C300", name: "หูฟังบลูทูธ", price_cents: 59900, stock: 5 },
    Product { sku: "D400", name: "แท่นชาร์จไร้สาย", price_cents: 39900, stock: 20 },
];

fn find_by_sku(sku: &str) -> Option<&'static Product> {
    for product in CATALOG.iter() {
        if product.sku == sku {
            return Some(product);
        }
    }
    None
}

fn format_price(cents: u32) -> String {
    format!("{}.{:02} บาท", cents / 100, cents % 100)
}

// ใช้ .map() + .unwrap_or() เมื่อ "ไม่พบ = ราคา 0" เป็นค่าที่สมเหตุสมผลพอสำหรับ caller (ไม่ต้อง panic)
fn price_of(sku: &str) -> u32 {
    find_by_sku(sku).map(|p| p.price_cents).unwrap_or(0)
}

// ใช้ .map() + .unwrap_or_else() เมื่อค่า fallback ต้องสร้างด้วย format! (มีต้นทุน จึงควร lazy)
fn display_name(sku: &str) -> String {
    find_by_sku(sku)
        .map(|p| p.name.to_string())
        .unwrap_or_else(|| format!("ไม่พบสินค้ารหัส {sku}"))
}

// ใช้ .ok_or_else() ต่อด้วย ? เมื่อ caller ต้องรู้ "เหตุผล" ที่ล้มเหลว ไม่ใช่แค่ None เฉย ๆ
fn require_in_stock(sku: &str) -> Result<&'static Product, String> {
    let product = find_by_sku(sku).ok_or_else(|| format!("ไม่พบสินค้ารหัส '{sku}' ในระบบ"))?;
    if product.stock == 0 {
        return Err(format!("สินค้า '{}' หมดสต๊อกแล้ว", product.name));
    }
    Ok(product)
}

fn main() {
    // 1) match แบบเต็มรูปแบบ — เหมาะกับตอนต้องแยกแยะทั้งสองกรณีอย่างชัดเจนในโค้ดหลัก
    match find_by_sku("A100") {
        Some(p) => println!("พบสินค้า: {} ราคา {}", p.name, format_price(p.price_cents)),
        None => println!("ไม่พบสินค้า"),
    }

    match find_by_sku("Z999") {
        Some(p) => println!("พบสินค้า: {} ราคา {}", p.name, format_price(p.price_cents)),
        None => println!("ไม่พบสินค้า"),
    }

    // 2) .map() + unwrap_or — เมื่อ "ไม่พบ" มีค่า default ที่สมเหตุสมผล ไม่ต้อง match เต็มรูปแบบ
    println!("ราคาของ B200: {}", format_price(price_of("B200")));
    println!("ราคาของ Z999 (ไม่มีจริง): {}", format_price(price_of("Z999")));

    println!("{}", display_name("C300"));
    println!("{}", display_name("Z999"));

    // 3) ok_or_else + ? — แปลงเป็น Result เมื่อ caller ต้องรู้เหตุผลที่แท้จริงของความล้มเหลว
    match require_in_stock("A100") {
        Ok(p) => println!("สั่งซื้อได้: {} (คงเหลือ {} ชิ้น)", p.name, p.stock),
        Err(e) => println!("สั่งซื้อไม่ได้: {e}"),
    }

    match require_in_stock("B200") {
        Ok(p) => println!("สั่งซื้อได้: {} (คงเหลือ {} ชิ้น)", p.name, p.stock),
        Err(e) => println!("สั่งซื้อไม่ได้: {e}"),
    }

    match require_in_stock("Z999") {
        Ok(p) => println!("สั่งซื้อได้: {} (คงเหลือ {} ชิ้น)", p.name, p.stock),
        Err(e) => println!("สั่งซื้อไม่ได้: {e}"),
    }
}
```

ผลลัพธ์:

```
พบสินค้า: เมาส์ไร้สาย ราคา 299.00 บาท
ไม่พบสินค้า
ราคาของ B200: 899.00 บาท
ราคาของ Z999 (ไม่มีจริง): 0.00 บาท
หูฟังบลูทูธ
ไม่พบสินค้ารหัส Z999
สั่งซื้อได้: เมาส์ไร้สาย (คงเหลือ 12 ชิ้น)
สั่งซื้อไม่ได้: สินค้า 'คีย์บอร์ดกลไก' หมดสต๊อกแล้ว
สั่งซื้อไม่ได้: ไม่พบสินค้ารหัส 'Z999' ในระบบ
```

**อธิบายภาพรวมของการออกแบบ:**

- `const CATALOG: [Product; 4] = [...]` — array คงที่ (ทวนจาก Part 3/9) ที่ประกาศระดับ global เก็บ struct
  `Product` ไว้ 4 รายการ ใช้ `&'static str` (ไม่ใช่ `String`) สำหรับ `sku`/`name` เพราะข้อมูลเหล่านี้เป็น string
  literal ที่ฝังอยู่ใน binary ตั้งแต่ compile time ไม่มีการสร้าง/ทำลายระหว่างการรันโปรแกรมเลย (จะเรียนความแตกต่าง
  ระหว่าง `&str` กับ `String` แบบเต็มใน Part 14)
- `find_by_sku` คือฟังก์ชันแกนหลักของทั้งระบบ — ใช้ **loop ธรรมดาแบบ manual** (ทวนจากรูปแบบเดียวกับ
  `find_user_by_id` ในหัวข้อ 11.2) วนดู `CATALOG` ทีละตัว ถ้าเจอ `sku` ที่ตรงกันให้ `return Some(product)` ทันที
  ถ้าวนจนหมด array แล้วไม่เจอเลยให้ `None` — ฟังก์ชันนี้คือ "จุดกำเนิด" ของ `Option<&Product>` ที่ทุกฟังก์ชันอื่น
  ในตัวอย่างนี้ต่อยอดมาจากมัน
- `price_of` และ `display_name` แสดงให้เห็นว่า **ฟังก์ชันเดียวกัน (`find_by_sku`) นำไปใช้ต่อได้หลายรูปแบบ** ผ่าน
  `.map()`/`.unwrap_or()`/`.unwrap_or_else()` แทนการเขียน `match`/`if let` ซ้ำทุกครั้ง — สังเกตว่า `price_of`
  ใช้ `.unwrap_or(0)` (eager, ค่า `0` ไม่มีต้นทุนสร้าง) ในขณะที่ `display_name` ใช้ `.unwrap_or_else(|| format!(...))`
  (lazy, เพราะ `format!` ต้อง allocate `String` ใหม่ ไม่ควรทำทุกครั้งถ้าไม่จำเป็น) — นี่คือการนำหลักการจากหัวข้อ
  11.4 มาใช้ตัดสินใจจริงในโค้ด ไม่ใช่แค่ท่องจำ syntax
- `require_in_stock` แสดงการ **chain `.ok_or_else()` ต่อด้วย `?`** ในฟังก์ชันเดียว: แปลง `Option<&Product>` เป็น
  `Result<&Product, String>` ก่อน (ถ้าไม่พบสินค้าเลย ให้ error บอกว่า "ไม่พบสินค้ารหัสนี้") แล้วใช้ `?` ดึงค่า
  `&Product` ออกมาใช้ต่อทันทีถ้าสำเร็จ จากนั้นเช็คเงื่อนไขทางธุรกิจเพิ่ม (`stock == 0`) ที่ **ไม่เกี่ยวกับ `Option`
  เลย** (เป็นการเช็คแยกหลังจากได้ `&Product` มาแล้ว) ก่อนคืน `Ok(product)` ในที่สุด — แสดงให้เห็นว่า `Option<T>`
  และ `Result<T, E>` ทำงานประสานกันได้อย่างลื่นไหลในฟังก์ชันเดียว ซึ่งเป็นรูปแบบที่พบบ่อยมากในโค้ด Rust จริง

ตัวอย่างนี้สรุปภาพรวมของทั้งบทได้ครบ: **`match` เมื่อต้อง handle ทั้งสองกรณีอย่างชัดเจนในโค้ดหลัก, `.map()`/
`.unwrap_or()`/`.unwrap_or_else()` เมื่อแค่ต้องการแปลง/ดึงค่าแบบมีทางออกที่ไม่ panic, และ `.ok_or_else()`/`?`
เมื่อต้อง propagate เหตุผลของความล้มเหลวขึ้นไปให้ผู้เรียกใช้จัดการต่อ** — เลือกเครื่องมือให้ตรงกับสถานการณ์ ไม่ใช่
ใช้ตัวเดียวกันตลอดทุกที่

## กับดักที่พบบ่อย (Common Pitfalls)

**1. `.unwrap()` บนค่า `None` — panic ที่พบบ่อยที่สุดในโค้ด Rust ของมือใหม่**

```rust
fn main() {
    let user_input: Option<i32> = None;
    let value = user_input.unwrap();
    println!("{value}");
}
```

```
thread 'main' panicked at src/main.rs:3:29:
called `Option::unwrap()` on a `None` value
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

**วิธีแก้**: อย่าใช้ `.unwrap()` กับค่าที่มาจากภายนอกระบบ (input ผู้ใช้, ผลลัพธ์จากการค้นหา, การ parse) เว้นแต่
พิสูจน์ได้จริงว่าเป็นไปไม่ได้ที่จะเป็น `None` ณ จุดนั้น (ตามที่อธิบายในหัวข้อ 11.4) ให้ใช้ `match`/`if let`/
`.unwrap_or()`/`.unwrap_or_else()` แทนเสมอในโค้ด production หรืออย่างน้อยเปลี่ยนเป็น `.expect("ข้อความอธิบาย
บริบท")` เพื่อให้ debug ง่ายขึ้นถ้ายังจำเป็นต้อง panic จริง ๆ

**2. พยายาม `.unwrap()` ค่าที่อยู่หลัง reference — ย้าย (move) ค่าออกจากสิ่งที่ยืมมาไม่ได้ (E0507)**

```rust
struct Ticket {
    note: Option<String>,
}

fn main() {
    let tickets = vec![Ticket { note: Some("โน้ตของตั๋ว".to_string()) }];

    for ticket in &tickets {
        let note_owned: String = ticket.note.unwrap();
        println!("{note_owned}");
    }
}
```

```
error[E0507]: cannot move out of `ticket.note` which is behind a shared reference
 --> src/main.rs:9:34
  |
9 |         let note_owned: String = ticket.note.unwrap();
  |                                  ^^^^^^^^^^^ -------- `ticket.note` moved due to this method call
  |                                  |
  |                                  move occurs because `ticket.note` has type `Option<String>`, which does not implement the `Copy` trait
  |
help: consider cloning the value if the performance cost is acceptable
  |
9 -         let note_owned: String = ticket.note.unwrap();
9 +         let note_owned: String = ticket.note.clone().unwrap();
  |
```

เกิด error เพราะ `for ticket in &tickets` ทำให้ `ticket` มี type เป็น `&Ticket` (ยืมมา ไม่ใช่เจ้าของ) `.unwrap()`
ต้อง **ยึด ownership ของ `String` ข้างใน `Option` ออกมา** (ตามหลัก Part 6) ซึ่งทำไม่ได้เลยกับค่าที่อยู่หลัง
`&` เพียงตัวเดียว **วิธีแก้ที่ถูกต้องที่สุด**: ใช้ `.as_ref()` ก่อน (ตามหัวข้อ 11.7) เพื่อแปลงเป็น `Option<&String>`
แทนการยึดออกมาทั้งตัว:

```rust
struct Ticket {
    note: Option<String>,
}

fn main() {
    let tickets = vec![Ticket { note: Some("โน้ตของตั๋ว".to_string()) }];

    for ticket in &tickets {
        let note_ref: Option<&String> = ticket.note.as_ref();
        match note_ref {
            Some(n) => println!("มีโน้ต: {n}"),
            None => println!("ไม่มีโน้ต"),
        }
    }
}
```

```
มีโน้ต: โน้ตของตั๋ว
```

**3. `.unwrap_or(...)` กับ argument ที่มี side effect หรือคำนวณหนัก — ทำงานทุกครั้งแม้ไม่จำเป็น**

```rust
fn compute_default_discount() -> u32 {
    println!("  (กำลังคำนวณส่วนลด default ... ทำงานหนักจำลองไว้)");
    5
}

fn main() {
    let discount: Option<u32> = Some(20);

    // ข้อควรระวัง: unwrap_or ประเมินค่า argument เสมอ แม้ discount จะเป็น Some อยู่แล้วก็ตาม
    println!("เรียก unwrap_or พร้อมฟังก์ชันที่มี side effect:");
    let result = discount.unwrap_or(compute_default_discount());
    println!("ผลลัพธ์: {result}");
}
```

```
เรียก unwrap_or พร้อมฟังก์ชันที่มี side effect:
  (กำลังคำนวณส่วนลด default ... ทำงานหนักจำลองไว้)
ผลลัพธ์: 20
```

โค้ดนี้ **compile ผ่านและรันได้ปกติ ไม่มี error/warning เตือนเลย** ทำให้เป็นกับดักที่ตรวจจับยากกว่าสองข้อก่อนหน้า
มาก — สังเกตว่าข้อความ `"(กำลังคำนวณส่วนลด default ...)"` ปรากฏขึ้น**ทั้งที่ `discount` เป็น `Some(20)` อยู่แล้ว**
(ไม่จำเป็นต้องใช้ค่า default เลย) เพราะ `compute_default_discount()` ถูกเรียก (ประเมินค่า argument) **ก่อน**ที่
`.unwrap_or()` จะรู้ด้วยซ้ำว่าต้องใช้ผลลัพธ์นั้นหรือไม่ — นี่คือกฎการประเมิน argument ปกติของ Rust (ประเมินทุก
argument ก่อนเรียกฟังก์ชัน ไม่ว่าฟังก์ชันจะใช้มันหรือไม่) ซึ่งใช้กับ method call ทุกตัวเหมือนกัน ไม่ใช่ข้อยกเว้น
ของ `.unwrap_or()` เอง **วิธีแก้**: ถ้าค่า default มีต้นทุนการคำนวณสูง (query database, เรียก API, allocate
หน่วยความจำใหญ่) หรือมี side effect ที่ไม่ควรเกิดขึ้นโดยไม่จำเป็น (เช่น `println!` ใน log จริง) ให้เปลี่ยนไปใช้
`.unwrap_or_else(|| compute_default_discount())` เสมอ เพราะมันจะเรียกฟังก์ชันก็ต่อเมื่อเป็น `None` จริง ๆ เท่านั้น
(ตามที่พิสูจน์แล้วในหัวข้อ 11.5) — กฎจำง่าย: **ถ้า default value ไม่ใช่ literal ธรรมดา (ตัวเลข/string คงที่/
ตัวแปรที่มีอยู่แล้ว) ให้สงสัยไว้ก่อนว่าอาจต้องใช้ `_or_else` แทน `_or`**

**4. ใช้ `.map()` ทั้งที่ควรใช้ `.and_then()` — ได้ `Option<Option<T>>` ซ้อนกันโดยไม่ตั้งใจ**

```rust
fn find_discount_percent(price: f64) -> Option<f64> {
    if price >= 1000.0 { Some(10.0) } else { None }
}

fn main() {
    let price: Option<f64> = Some(1500.0);

    // ใช้ .map() กับ closure ที่คืน Option<f64> เอง — ได้ Option<Option<f64>> ซ้อนกันสองชั้น
    let nested: Option<Option<f64>> = price.map(|p| find_discount_percent(p));
    println!("{nested:?}");

    // ปัญหา: ต้อง unwrap สองรอบ (หรือ .flatten() ซึ่งเรียนเต็มใน Part 25-26) ก่อนใช้ค่าจริงได้
    // ถ้าลืมและพยายามใช้ nested เหมือนเป็น Option<f64> ตรง ๆ จะเจอ E0308 ทันที
}
```

```
Some(Some(10.0))
```

โค้ดนี้ **compile ผ่าน ไม่มี error** เพราะในเชิง type มันถูกต้องสมบูรณ์ (`Option<Option<f64>>` เป็น type ที่มีจริง
ไม่ใช่ error) แต่มันคือ "กับดักเชิง design" — `Option<Option<f64>>` ใช้งานยากกว่า `Option<f64>` มาก (ต้อง match/
unwrap สองชั้นซ้อนกันเพื่อเข้าถึงค่าจริง) และมักไม่ใช่สิ่งที่ตั้งใจออกแบบไว้ตั้งแต่แรก **วิธีแก้**: ทุกครั้งที่
closure ที่ส่งเข้า `.map()` คืนค่าเป็น `Option<U>` เอง ให้เปลี่ยนไปใช้ `.and_then()` แทนทันที (`price.and_then(|p|
find_discount_percent(p))` จะได้ `Option<f64>` ธรรมดา ไม่ซ้อนกัน ตามที่พิสูจน์แล้วในหัวข้อ 11.6) — สัญญาณเตือนที่
สังเกตได้ง่ายคือ **ถ้า type ที่ compiler infer ให้มีคำว่า `Option<Option<...>>` ซ้อนกัน ให้สงสัยไว้ก่อนว่าน่าจะลืม
เปลี่ยน `.map()` เป็น `.and_then()` ตรงไหนสักจุด**

**5. ใช้ `?` operator ในฟังก์ชันที่ไม่ได้คืน `Option`/`Result` (E0277)**

```rust
fn parse_and_double(input: &str) -> u32 {
    let n: u32 = input.parse().ok()?;
    n * 2
}

fn main() {
    println!("{}", parse_and_double("21"));
}
```

```
error[E0277]: the `?` operator can only be used in a function that returns `Result` or `Option` (or another type that implements `FromResidual`)
 --> src/main.rs:2:36
  |
1 | fn parse_and_double(input: &str) -> u32 {
  | --------------------------------------- this function should return `Result` or `Option` to accept `?`
2 |     let n: u32 = input.parse().ok()?;
  |                                    ^ cannot use the `?` operator in a function that returns `u32`
```

`?` (ตามที่เรียนในหัวข้อ 11.10) ทำงานโดย "return ค่าที่แสดงความล้มเหลวออกจากฟังก์ชันทั้งหมดทันที" (`None` สำหรับ
`Option`, `Err(e)` สำหรับ `Result`) — มันจึง**ต้องรู้ว่าจะ return อะไรกันแน่เมื่อล้มเหลว** ซึ่งเป็นไปไม่ได้ถ้า
ฟังก์ชันประกาศ return type เป็น `u32` ตรง ๆ (ไม่มีทาง "return ความล้มเหลว" ออกมาเป็น `u32` ได้อย่างสมเหตุสมผล)
**วิธีแก้**: เปลี่ยน return type ของฟังก์ชันให้เป็น `Option<u32>` (แล้วห่อค่าสุดท้ายด้วย `Some(n * 2)`) หรือถ้า
ต้องการคง return type เป็น `u32` จริง ๆ ให้เปลี่ยนไปใช้ `.unwrap_or(...)`/`match` แทน `?` เพื่อดึงค่า `u32` ที่
แน่นอนออกมาโดยไม่ propagate ความล้มเหลวออกไป

**6. สับสนระหว่าง `Option<&T>` กับ `&Option<T>` — ส่ง type ผิดรูปร่างเข้าฟังก์ชัน (E0308)**

```rust
struct Session {
    user_note: Option<String>,
}

fn describe_option_of_ref(note: Option<&String>) -> String {
    match note {
        Some(text) => format!("มีบันทึกไว้ว่า: {text}"),
        None => "ไม่มีบันทึกไว้".to_string(),
    }
}

fn main() {
    let session = Session {
        user_note: Some("ลูกค้าต้องการติดต่อกลับ".to_string()),
    };

    println!("{}", describe_option_of_ref(&session.user_note));
}
```

```
error[E0308]: mismatched types
  --> src/main.rs:16:43
   |
16 |     println!("{}", describe_option_of_ref(&session.user_note));
   |                    ---------------------- ^^^^^^^^^^^^^^^^^^ expected `Option<&String>`, found `&Option<String>`
   |                    |
   |                    arguments to this function are incorrect
   |
   = note:   expected enum `Option<&String>`
           found reference `&Option<String>`
help: try using `.as_ref()` to convert `&Option<String>` to `Option<&String>`
   |
16 -     println!("{}", describe_option_of_ref(&session.user_note));
16 +     println!("{}", describe_option_of_ref(session.user_note.as_ref()));
   |
```

ตามที่อธิบายในหัวข้อ 11.11 — `&session.user_note` มี type เป็น `&Option<String>` (reference ไปยังกล่อง `Option`
ทั้งกล่อง) แต่ฟังก์ชันต้องการ `Option<&String>` (ตัวเลือกของ reference) ทั้งสองรูปร่างนี้**ไม่ใช่ type เดียวกัน**
แม้จะ "ประกอบด้วยส่วนเดียวกัน" ก็ตาม สังเกตว่า compiler แนะนำวิธีแก้ให้ตรง ๆ อีกแล้ว (`help: try using .as_ref()`)
**วิธีแก้**: เรียก `.as_ref()` แปลงรูปร่างก่อนส่งเข้าฟังก์ชันเสมอ (`session.user_note.as_ref()`) — error แบบนี้
เป็นสัญญาณที่ดีมากว่าคุณต้องใช้ `.as_ref()`/`.as_mut()` ณ จุดนั้น ไม่ใช่สัญญาณว่าการออกแบบฟังก์ชันผิด

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** เขียนฟังก์ชัน `fn first_even(nums: &[i32]) -> Option<i32>` ที่วนหา**เลขคู่ตัวแรก**ใน slice ด้วย
   loop ธรรมดา (แบบเดียวกับ `find_user_by_id`/`find_by_sku` ในบทนี้) คืน `Some(เลขนั้น)` ถ้าพบ หรือ `None` ถ้า
   ไม่มีเลขคู่เลยในลิสต์ จากนั้นใช้ `.map()` แปลงผลลัพธ์เป็นข้อความ `"พบเลขคู่: {n}"` และ `.unwrap_or_else()`
   ใส่ข้อความ fallback `"ไม่พบเลขคู่เลย"` เมื่อไม่พบ (hint: ผลลัพธ์สุดท้ายควรเป็น `String` เดียว พร้อมสำหรับ
   `println!` ตรง ๆ)

2. **[กลาง]** เขียนฟังก์ชัน `fn parse_and_validate_age(input: &str) -> Option<u32>` ที่ `.parse::<u32>().ok()`
   สตริงที่รับมาก่อน (ได้ `Option<u32>`) แล้ว `.filter()` ต่อว่าค่านั้นต้องอยู่ในช่วง 0-150 ปีเท่านั้นถึงจะถือว่า
   สมเหตุสมผล (ใช้ `.and_then()` เพื่อรวม parse กับ filter เข้าด้วยกันในบรรทัดเดียวก็ได้ ลองคิดดูว่าทำไมต้องใช้
   `.and_then()` ไม่ใช่ `.map()` ตรงจุดที่ต่อกับผลลัพธ์ของ `.parse()`) ทดสอบด้วย input หลายแบบ: `"25"` (ผ่าน),
   `"200"` (ไม่ผ่านเพราะเกิน 150), `"abc"` (parse ไม่สำเร็จเลย) — ทั้งสามกรณีควรได้ `None` เหมือนกันสำหรับสอง
   กรณีหลัง แม้จะ "ล้มเหลว" ด้วยเหตุผลคนละแบบ

3. **[ยาก]** สร้าง struct `Employee { id: u32, name: String, manager_id: Option<u32> }` (พนักงานบางคนไม่มีหัวหน้า
   จึง `manager_id` เป็น `Option<u32>`) สร้าง array คงที่ของพนักงาน 4-5 คน (บางคนมี `manager_id`, บางคนไม่มี)
   เขียนฟังก์ชัน `fn find_employee(employees: &[Employee], id: u32) -> Option<&Employee>` ด้วย loop ธรรมดา
   จากนั้นเขียนฟังก์ชัน `fn find_manager_name(employees: &[Employee], employee_id: u32) -> Result<String, String>`
   ที่ต้อง: (1) หาพนักงานด้วย `employee_id` ก่อน ถ้าไม่พบให้ error ว่า "ไม่พบพนักงาน" (2) ถ้าพบแต่ `manager_id`
   เป็น `None` ให้ error ว่า "พนักงานคนนี้ไม่มีหัวหน้า" (3) ถ้ามี `manager_id` ให้ค้นหาชื่อหัวหน้าต่อ ถ้าหา
   `manager_id` นั้นไม่พบใน `employees` เลย (ข้อมูลไม่สมบูรณ์) ให้ error อีกแบบ (hint: ต้อง chain `.ok_or_else()`
   กับ `?` สองรอบ — รอบแรกสำหรับหาพนักงาน รอบสองสำหรับหาหัวหน้าจาก `manager_id` ที่ได้มา)

4. **[ยาก/ประยุกต์ใช้งานจริง]** ขยายตัวอย่าง catalog ท้ายบทนี้ (หัวข้อ 11.12): เพิ่ม field `category: &'static str`
   ให้ `Product` เขียนฟังก์ชัน `fn cheapest_in_stock(category: &str) -> Option<&'static Product>` ที่หาสินค้า
   **ราคาถูกที่สุด** ในหมวดหมู่ที่ระบุ **โดยต้องมี `stock > 0` เท่านั้น** (ใช้ loop ธรรมดาวนเก็บตัวที่ถูกที่สุด
   ไว้ในตัวแปร `Option<&Product>` แล้วอัปเดตทุกครั้งที่เจอตัวที่ถูกกว่า — ลองคิดว่าจะใช้ `match`/`.map_or()`
   หรือ `if let` ตรวจสอบตัวแปรสะสมนั้นในแต่ละรอบ loop อย่างไร) จากนั้นเขียนฟังก์ชัน `fn checkout_message(sku:
   &str) -> String` ที่ chain `.filter()` (ต้องมี stock เหลือ) กับ `.map()` (แปลงเป็นข้อความ) และ `.unwrap_or_else()`
   (ข้อความ fallback ที่บอกสาเหตุต่างกันระหว่าง "ไม่พบสินค้า" กับ "สินค้าหมด" — ลองคิดว่าจะแยกสองกรณีนี้ด้วย
   `Option` เพียงตัวเดียวได้จริงหรือไม่ หรือต้องใช้ `match`/`Result` แทนถึงจะแยกได้ครบ)

## สรุป

บทนี้เจาะลึก `Option<T>` ต่อจากที่ Part 10 แนะนำไว้แบบผิวเผิน โดยเริ่มจากการทวนว่า Rust แก้ "billion dollar mistake"
ของ null อย่างไร (ทำให้การไม่มีค่าเป็น type ที่ compiler บังคับให้เช็คเสมอ) แล้วไล่ดูว่า `Option<T>` โผล่มาให้เห็น
ทั่วไปในโค้ดจริงตรงไหนบ้าง (`Vec::first()`, `.get()`, `str::find()`, ฟังก์ชันค้นหาที่เราออกแบบเอง) จากนั้นเจาะลึก
API ทั้งหมดของ `Option<T>` อย่างเป็นระบบ: `.unwrap()`/`.expect()` สำหรับดึงค่าแบบเสี่ยง panic (พร้อมเกณฑ์ตัดสินใจ
ว่าเมื่อไหร่ยอมรับได้กับเมื่อไหร่คือ code smell), ตระกูล `.unwrap_or*()` สำหรับดึงค่าแบบปลอดภัย (พร้อมความต่างเรื่อง
eager/lazy evaluation), `.map()`/`.and_then()` สำหรับแปลงค่าโดยไม่ต้อง unwrap, `.is_some()`/`.is_none()`/`.as_ref()`/
`.as_mut()` สำหรับถามคำถามและยืมดูค่าโดยไม่ทำลาย ownership, `.or()`/`.or_else()`/`.filter()`/`.zip()` สำหรับรวม
Option หลายตัว, `.ok_or()`/`.ok_or_else()` สำหรับแปลงเป็น `Result<T, E>` เมื่อต้อง propagate เหตุผลของความล้มเหลว,
`?` operator กับฟังก์ชันที่คืน `Option<T>`, และปิดท้ายด้วยความแตกต่างสำคัญระหว่าง `Option<&T>` กับ `&Option<T>`
ตัวอย่างค้นหาสินค้าด้วย SKU ท้ายบทแสดงให้เห็นว่าเครื่องมือทั้งหมดนี้ทำงานร่วมกันได้อย่างไรในโค้ดจริง — เลือกใช้
`match` เมื่อต้อง handle ทั้งสองกรณีอย่างชัดเจน, ใช้ `.map()`/`.unwrap_or*()` เมื่อแค่ต้องแปลง/ดึงค่าแบบไม่ panic,
และใช้ `.ok_or_else()`/`?` เมื่อต้อง propagate เหตุผลของความล้มเหลวขึ้นไปให้ผู้เรียกใช้จัดการต่อ

ใน **Part 12** เราจะเจาะลึก `Result<T, E>` แบบเต็มรูปแบบต่อจากที่แนะนำไว้สั้น ๆ ในหัวข้อ 11.9 — วิธี `match`/
`if let` บน `Result`, `?` operator แบบเต็มรูปแบบ (รวมการแปลง error type ด้วย `From`), การออกแบบฟังก์ชันที่คืน
`Result` อย่าง idiomatic, และความแตกต่างเชิงการออกแบบระหว่าง "ใช้ `Option<T>` เมื่อไม่มีค่า" กับ "ใช้ `Result<T, E>`
เมื่อล้มเหลวพร้อมเหตุผล" ที่เราเกริ่นไว้ในบทนี้

---

**Part ก่อนหน้า:** [Enums และ Pattern Matching](part-010-enums-and-pattern-matching.md) | **Part ถัดไป:** [Result<T,E> และ Error Handling เบื้องต้น](part-012-result-and-error-handling.md)
