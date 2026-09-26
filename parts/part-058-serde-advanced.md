# Part 58: Serde ขั้นสูง (custom Serialize/Deserialize)

> โมดูล: ระดับสูง (Advanced) | ระดับ: สูง | เวลาโดยประมาณ: 240 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้อย่างชัดเจนว่า **เมื่อไหร่** `#[derive(Serialize, Deserialize)]` จาก Part 57 ไม่พอ และ
  ต้องลงมือ implement trait เอง — แยกได้สามสถานการณ์หลัก: type จาก external crate ที่แก้ derive ไม่ได้
  (ผูกกับ orphan rule จาก Part 21), รูปแบบ JSON/format ที่ต้องการไม่ตรงกับ shape ของ struct ใน Rust
  เลย, และ validation ที่ต้องรันระหว่างแปลงข้อมูลก่อนค่าจะกลายเป็น Rust value ที่ถูกต้อง
- อ่าน signature `fn serialize<S: Serializer>(&self, serializer: S) -> Result<S::Ok, S::Error>` แล้ว
  อธิบายได้ว่าทำไม `S` ต้องเป็น **generic parameter** ไม่ใช่ concrete type ตายตัว (เชื่อมกับหลักการ
  format-independence ที่ Part 57 สอนไว้จากมุมของ "ผู้ใช้" มาดูจากมุมของ "คนเขียน implementation" เอง)
  แล้ว implement `Serialize` ด้วยมือให้ type ที่มี custom representation จริง (`Color` → hex string)
- อธิบายได้ว่าทำไม `Deserialize` ซับซ้อนกว่า `Serialize` โดยธรรมชาติ (ต้องรู้จัก **Visitor pattern**)
  เพราะการ deserialize ต้องรับมือกับ "ข้อมูลที่ format บอกมาว่าเป็นอะไร" ก่อนจะรู้วิธีตีความ — แล้ว
  implement `Deserialize` แบบเต็มรูปแบบให้ `Color` เดิม พร้อม error handling ที่ถูกต้องสำหรับ input
  ที่ผิดรูปแบบ โดยใช้ `serde::de::Error::custom`
- ใช้ `#[serde(with = "module")]`, `#[serde(serialize_with = "fn")]`, `#[serde(deserialize_with =
  "fn")]` เป็นทางเลือกกลางระหว่าง derive เต็มรูปแบบกับ manual implementation เต็มรูปแบบ — ปรับแค่
  field เดียวโดยที่ field อื่น ๆ ยังใช้ derive ตามปกติ
- อธิบายเทคนิค **zero-copy deserialization** ด้วย `Deserialize<'de>` และ `&'de str` — รู้ว่าทำไมมันเร็ว
  กว่าการ allocate `String` ใหม่ทุก field และรู้ข้อจำกัดด้าน lifetime ที่แลกมา (ผูกกับ Part 20/23)
- เขียน validation ที่บังคับให้ข้อมูลที่ผิด**ไม่มีทางกลายเป็น Rust value ที่ถูกต้องได้เลย** ผ่าน
  `Deserialize`/`deserialize_with` (ผูกกับ pattern "newtype + validation" จาก Part 53 ให้เข้ากับ serde
  โดยตรง) และอ่าน error message จริงที่ serde ส่งกลับเมื่อ reject ข้อมูล
- รู้จัก `#[serde(flatten)]` และ `#[serde(transparent)]` ในระดับที่ใช้งานได้จริง พร้อมรู้ข้อจำกัดของ
  แต่ละตัว และปิดท้ายด้วย case study ขนาดใหญ่: type `Money` ที่ต้อง serialize เป็น string แบบ `"$19.99"`
  เสมอ (ไม่ใช่ float ดิบ ๆ) พร้อม round-trip และ validation แบบเต็มรูปแบบ

## ความรู้ที่ต้องมีมาก่อน

- **Part 57 (Serialization: Serde เบื้องต้น) — สำคัญที่สุด** บทนี้เป็น**ภาคต่อโดยตรง** ไม่ใช่บทแยก
  Part 57 สอน `#[derive(Serialize, Deserialize)]`, attribute มาตรฐาน (`rename`, `rename_all`, `skip`,
  `default`, tagging ของ enum), การทำงานกับหลาย format (JSON/`serde_json`, และรูปแบบอื่น ๆ) และ
  `serde_json::Value` สำหรับข้อมูลที่ shape ไม่ตายตัว ถ้าคุณจำหลักการ "Serde แยก data model จาก wire
  format" และวิธีอ่าน field attribute พื้นฐานจาก Part 57 ไม่ได้แล้ว ควรย้อนกลับไปอ่านซ้ำก่อน เพราะบทนี้
  จะไม่สอนพื้นฐานเหล่านั้นซ้ำ — บทนี้ตั้งต้นจากคำถามที่ Part 57 ทิ้งไว้ตอนท้าย: "แล้วถ้า derive ไม่พอ
  ต้องทำยังไง"
- **Part 21 (Traits ขั้นสูง)** — เข้าใจ **orphan rule** (implement trait ให้ type ได้ก็ต่อเมื่อ trait
  หรือ type อย่างน้อยหนึ่งอันเป็นของ crate ตัวเอง) เพราะสถานการณ์ที่ 1 ในหัวข้อ 58.1 คือกรณีที่ทั้ง
  `Serialize`/`Deserialize` (ของ serde) และ type ที่ต้องการ serialize (ของ crate อื่น) ไม่ใช่ของเราเลย
  ทั้งคู่ — ต้องเข้าใจ orphan rule เป๊ะถึงจะเข้าใจว่าทำไมแก้ปัญหานี้ด้วย newtype pattern เท่านั้น
- **Part 44-45 (Procedural Macros พื้นฐานและขั้นสูง)** — คุณเคยเห็น derive macro (ที่เราเขียนเอง) สร้าง
  `impl Trait for Type` จาก `syn`/`quote` มาแล้ว บทนี้จะแสดงให้เห็นว่า `#[derive(Serialize)]`/
  `#[derive(Deserialize)]` ของ `serde_derive` ก็เป็น proc macro แบบเดียวกันเป๊ะ (ไม่มีเวทมนตร์อะไร
  พิเศษ) มัน generate `impl Serialize for YourType { fn serialize(...) { ... } }` ให้อัตโนมัติเท่านั้น
  — บทนี้แค่สอนให้คุณเขียน `impl` แบบเดียวกันด้วยมือเอง ในจุดที่ macro ทำให้ไม่ได้
- **Part 19-21 (Traits)** — `Serialize`/`Deserialize` เป็นแค่ trait ธรรมดา ต้องเข้าใจ `impl Trait for
  Type`, associated type (`S::Ok`, `S::Error`), และ generic trait bound มาก่อน
- **Part 20/23 (Lifetimes เบื้องต้นและขั้นสูง)** — หัวข้อ 58.5 (zero-copy deserialization) ใช้
  `Deserialize<'de>` ซึ่งเป็น trait ที่มี lifetime parameter ของตัวเอง ต้องเข้าใจว่า lifetime ผูกอายุ
  ของ reference เข้ากับข้อมูลต้นทางอย่างไร (borrow checker error ที่จะเห็นในหัวข้อนี้เป็น error แบบ
  เดียวกับที่ Part 20 สอนไว้ทุกตัวอักษร)
- **Part 12 และ Part 30 (Error Handling เบื้องต้นและขั้นสูง)** — หัวข้อ 58.3 และ 58.6 ต้องสร้าง error
  ระหว่าง deserialize ด้วย `serde::de::Error::custom(...)` ซึ่งเป็นวิธีสร้าง error type ที่ implement
  `std::error::Error` โดยธรรมชาติ (Part 12 สอน `Result<T, E>` และ `?` เป็นพื้นฐาน Part 30 สอนการออกแบบ
  error type เอง — `serde::de::Error` เป็น trait ที่ออกแบบตามหลักการเดียวกัน)
- **Part 53 (Design Patterns: Newtype, Typestate, RAII)** — หัวข้อ 58.6 เอา pattern "newtype ที่บังคับ
  invariant ผ่าน constructor" จาก Part 53 มาผสานกับ serde โดยตรง (`Percentage(f64)` ที่ค่าต้องอยู่ใน
  ช่วง 0.0-100.0 เสมอ ไม่มีทางสร้างค่าที่ผิดได้แม้จะมาจาก deserialize ก็ตาม)
- แวบไปดูล่วงหน้า: **Part 59 (CLI Applications ด้วย clap)** ที่คุณจะเจอ `#[derive(Parser)]` ของ `clap`
  ซึ่งใช้กลไก derive macro + helper attribute หลักการเดียวกันกับ `serde_derive` ทุกอย่าง — บทนี้ปูทาง
  ให้คุณเข้าใจว่าเบื้องหลัง derive macro สำเร็จรูปพวกนี้ทำงานอย่างไรจริง ๆ

## เนื้อหา

### 58.1 ทวนจาก Part 57 และภาพรวม: เมื่อไหร่ derive ไม่พอ

Part 57 สอนไว้ว่า `#[derive(Serialize, Deserialize)]` จัดการเคสส่วนใหญ่ของโลกจริงได้หมด — struct ปกติ,
enum, `Option<T>`, `Vec<T>`, field attribute อย่าง `rename`/`skip`/`default` และ enum tagging ทุกแบบที่
serde มีให้ในตัว สิ่งเหล่านี้ครอบคลุมสถานการณ์ที่พบเจอบ่อยที่สุดในโปรเจกต์จริงจริง ๆ ประมาณ 90% ขึ้นไป
แต่มีสามสถานการณ์เฉพาะที่ derive **ทำไม่ได้เลยไม่ว่าจะเพิ่ม attribute อะไรก็ตาม** เพราะข้อจำกัดของมันไม่
ได้อยู่ที่ attribute แต่อยู่ที่**สมมติฐานพื้นฐาน**ของ derive macro เอง

**สถานการณ์ที่ 1: type ที่มาจาก external crate ที่คุณไม่ได้เป็นเจ้าของ**

สมมติคุณใช้ crate `image` (หรือ crate ไหนก็ตามที่ไม่ใช่ของคุณ) ที่มี type ชื่อ `Rgb` และคุณต้องการ
serialize มันเป็น JSON แต่ crate `image` ไม่ได้ทำ `#[derive(Serialize)]` ให้ `Rgb` ไว้ (เพราะทีมพัฒนา
`image` ไม่ได้ตั้งใจให้ crate ของเขาผูกกับ `serde` โดยตรง หรือ derive ไว้แต่ไม่ตรงกับ shape ที่คุณ
ต้องการ) คุณแก้ source code ของ crate `image` ไม่ได้ (มันไม่ใช่ของคุณ) แล้วคุณจะเขียน

```rust
impl Serialize for image::Rgb { /* ... */ }
```

เองตรง ๆ ในโปรเจกต์ของคุณได้ไหม? **ไม่ได้** — และนี่คือจุดที่ทวนจาก Part 21 ให้แม่นเป๊ะ: **orphan rule**
ของ Rust บอกว่าคุณจะ implement `impl Trait for Type` ได้ก็ต่อเมื่อ **trait หรือ type อย่างน้อยหนึ่งอัน
เป็นของ crate ปัจจุบัน** ในกรณีนี้ `Serialize` เป็นของ crate `serde` (ไม่ใช่ของคุณ) และ `Rgb` เป็นของ
crate `image` (ก็ไม่ใช่ของคุณเหมือนกัน) — ทั้งสองฝั่งเป็น "ของนอก" หมด จึงชนกับ orphan rule ทันที เรา
จะเห็น compiler error จริงของกรณีนี้ในหัวข้อกับดัก (58.9.1) วิธีแก้มาตรฐานคือ **newtype pattern** (ที่
Part 21 แนะนำไว้แล้วสำหรับปัญหา orphan rule โดยทั่วไป): สร้าง `struct MyRgb(image::Rgb);` ที่เป็นของ
crate คุณเอง แล้ว `impl Serialize for MyRgb` ตรงนั้นแทน — เพราะตอนนี้ `MyRgb` เป็นของ crate คุณ (ฝั่งใด
ฝั่งหนึ่งเป็น "ของใน" แล้ว) orphan rule จึงอนุญาต

**สถานการณ์ที่ 2: shape ของข้อมูลที่ serialize ต้องการ ไม่ตรงกับ shape ของ struct ใน Rust เลย**

นี่คือกรณีที่ derive ไม่ได้ "ทำไม่ได้" แต่ "ทำได้แค่แบบเดียว" — คือแปลง struct/enum ตรง ๆ เป็น JSON
shape ที่ตรงกับ field ของมันเป๊ะ (บวกกับ attribute ที่ Part 57 สอน ซึ่งปรับได้แค่ชื่อ field/การข้าม
field/ค่า default เท่านั้น) ลองดูตัวอย่างจริงสองแบบ:

```rust
use serde::Serialize;

// จำลองรูปร่างภายในของ std::time::Duration (secs + subsec nanos)
// เพื่อโชว์ว่า derive แบบตรง ๆ ให้ shape ที่ตรงกับ struct ภายใน ไม่ใช่รูปแบบที่มนุษย์อ่านง่าย
#[derive(Debug, Serialize)]
struct DurationLike {
    secs: u64,
    nanos: u32,
}

fn main() {
    let d = DurationLike { secs: 330, nanos: 0 };
    println!("derive ตรง ๆ: {}", serde_json::to_string(&d).unwrap());
}
```

รันจริงได้:

```
derive ตรง ๆ: {"secs":330,"nanos":0}
```

แต่สิ่งที่คุณต้องการจริง ๆ อาจเป็น `"5m30s"` (string ที่มนุษย์อ่านง่าย เหมาะกับ config file หรือ log ที่
คนต้องอ่านตรง ๆ) ไม่มี attribute ไหนของ derive ที่ทำให้ struct สอง field แปลงเป็น string เดี่ยว ๆ แบบนี้
ได้ — เพราะ derive macro (ตามที่ Part 45 สอนไว้) generate โค้ดจาก**โครงสร้าง field ที่มันเห็น** มันไม่มี
ทางรู้ว่า "0.33 นาที" ควร format เป็นอะไร นั่นเป็น**ตรรกะทางธุรกิจ (business logic)** ที่ต้องเขียนเอง

อีกตัวอย่างที่พบบ่อยกว่า: enum ที่มี data ใน variant

```rust
use serde::Serialize;

#[derive(Debug, Serialize)]
enum AuditEvent {
    Login { user: String },
    Logout,
}

fn main() {
    let events = vec![
        AuditEvent::Login { user: "alice".to_string() },
        AuditEvent::Logout,
    ];
    for e in &events {
        println!("default external tagging: {}", serde_json::to_string(e).unwrap());
    }
}
```

รันจริงได้:

```
default external tagging: {"Login":{"user":"alice"}}
default external tagging: "Logout"
```

นี่คือ**default external tagging** ที่ Part 57 สอนไว้แล้ว (แต่ทวนสั้น ๆ ให้ครบบริบท) — variant ที่มี
data กลายเป็น object ชั้นเดียวที่ key คือชื่อ variant Part 57 สอน attribute `#[serde(tag = "...")]`,
`#[serde(tag = "...", content = "...")]`, และ `#[serde(untagged)]` ที่ปรับรูปแบบ tagging ได้หลายแบบ
มาแล้ว แต่ถ้าคุณต้องการรูปแบบที่**ไม่ตรงกับตัวเลือกมาตรฐานตัวไหนของ serde เลย** เช่น `"LOGIN:alice"` /
`"LOGOUT"` (ตัวพิมพ์ใหญ่ทั้งหมด คั่นด้วย `:` เพื่อ interop กับระบบ log เดิมที่มีอยู่แล้วในบริษัท) — นี่
คือ scheme ที่ไม่มี attribute มาตรฐานตัวไหนสร้างให้ได้ ต้องเขียน `Serialize`/`Deserialize` ด้วยมือเท่านั้น

**สถานการณ์ที่ 3: validation ที่ต้องรันระหว่าง deserialize ไม่ใช่หลังจากนั้น**

สมมติคุณมี `struct Percentage(f64)` ที่ค่าต้องอยู่ในช่วง 0.0-100.0 เสมอ (ทวนจาก Part 53: newtype ที่
บังคับ invariant ผ่าน constructor) ถ้า derive `Deserialize` ตรง ๆ ให้ `Percentage(f64)` มันจะรับค่า
`f64` **อะไรก็ได้**เข้ามา รวมถึง `-999.0` หรือ `1e300` ด้วย — เพราะ derive ไม่รู้จัก invariant ทางธุรกิจ
ของคุณเลย มันรู้แค่ว่า "field เดียว type `f64`" วิธีแก้ผิด ๆ ที่มือใหม่มักทำคือ deserialize แบบปกติก่อน
แล้วค่อยเช็คทีหลังด้วย `if` — ปัญหาคือ **ค่าที่ผิดได้กลายเป็น `Percentage` ที่ใช้งานได้ไปแล้วชั่วขณะหนึ่ง**
ก่อนจะถูกจับได้ (ถ้าคุณลืมเช็คสักจุดเดียวในโค้ด ค่าผิดจะหลุดรอดไปได้เงียบ ๆ) วิธีที่ถูกต้องคือทำให้
**การ deserialize เองปฏิเสธค่าที่ผิดตั้งแต่จุดกำเนิด** — ไม่มีทางสร้าง `Percentage` ที่มีค่าผิดได้เลย
ไม่ว่าทางไหน (constructor ปกติก็เช็ค, deserialize ก็เช็ค) — หัวข้อ 58.6 จะสอนวิธีทำสิ่งนี้อย่างถูกต้อง

ตารางสรุปทั้งสามสถานการณ์และวิธีแก้:

| สถานการณ์ | อาการ | วิธีแก้ |
|---|---|---|
| Type จาก external crate | `impl Serialize for ExternalType` ผิด orphan rule | newtype pattern (Part 21) |
| Shape ไม่ตรงกับ struct field | derive ให้ได้แค่ shape ที่ตรงกับ field ตรง ๆ | เขียน `Serialize`/`Deserialize` เอง หรือ `with =`/`serialize_with =` |
| ต้อง validate ระหว่าง deserialize | ข้อมูลผิดกลายเป็นค่าที่ใช้งานได้ชั่วคราว | เขียน `Deserialize` เอง หรือ `deserialize_with =` ที่ reject ค่าผิด |

บทนี้จะไล่แก้ทั้งสามสถานการณ์ทีละข้อ โดยเริ่มจากการเข้าใจ trait `Serialize`/`Deserialize` ให้ลึกถึงระดับ
signature จริงก่อน (58.2-58.3) แล้วค่อยขยายไปที่ทางลัดที่ปฏิบัติได้จริงในโปรเจกต์ (58.4), เทคนิค
performance เฉพาะทาง (58.5), validation (58.6), attribute เสริมสองตัว (58.7), และปิดท้ายด้วย case
study ขนาดใหญ่ที่รวมทุกอย่างเข้าด้วยกัน (58.8)

### 58.2 Anatomy ของ trait `Serialize`: ทำไม `Serializer` ต้องเป็น generic

มาดู signature จริงของ trait `Serialize` จาก crate `serde` (ไม่ต้องเขียนเอง แค่ทำความเข้าใจ):

```rust
pub trait Serialize {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer;
}
```

จุดที่ต้องเข้าใจให้ลึกที่สุดในบรรทัดนี้คือ **`S` เป็น generic type parameter ของฟังก์ชัน `serialize`
เอง ไม่ใช่ของ `struct`/`enum` ที่ implement trait นี้** ลองเทียบกับสิ่งที่**ผิด**ก่อน เพื่อให้เห็นว่าทำไม
การออกแบบแบบนี้สำคัญ — สมมติ (ผิด) ว่า serde ออกแบบ trait นี้แบบ hardcode format ไว้ตรง ๆ:

```rust
// ถ้า serde ออกแบบแบบนี้ (สมมติเพื่อเปรียบเทียบ — serde จริงไม่ทำแบบนี้)
trait SerializeToJsonOnly {
    fn serialize(&self) -> serde_json::Value;
}
```

ถ้า `Serialize` ถูกออกแบบแบบนี้ ทุก type ที่ implement มันจะผูกติดกับ JSON ตายตัว ต้องการ serialize เป็น
YAML, TOML, MessagePack, หรือ format อื่น ๆ ไม่ได้เลยแม้แต่นิดเดียว ต้องเขียน trait ใหม่ทุกครั้งที่มี
format ใหม่ และ type เดิมต้อง implement ซ้ำสำหรับแต่ละ format — นี่คือสิ่งที่ Part 57 สอนไว้แล้วว่า
serde **ไม่ทำแบบนี้** จาก**มุมของผู้ใช้** (คุณ derive ครั้งเดียว ใช้กับ `serde_json`, `serde_yaml`,
`bincode` ได้หมด) บทนี้กำลังอธิบายจุดเดียวกันแต่จาก**มุมของคนเขียน implementation**: เหตุผลที่มันทำงาน
แบบนั้นได้คือเพราะ `serialize<S: Serializer>` รับ **ตัว `Serializer` เป็น parameter แบบ generic** ตัว
`Serialize::serialize` ของคุณ**ไม่รู้จักและไม่สนใจ**ว่า `S` คือ JSON serializer, YAML serializer, หรือ
อะไร มันแค่เรียก method ของ `S` (เช่น `serializer.serialize_str(...)`, `serializer.serialize_i64(...)`)
ตามชนิดข้อมูลที่ตัวเองมี ส่วน `S` แต่ละตัว (ที่ crate `serde_json`, `serde_yaml` implement `Serializer`
trait ให้) จะเป็นคนแปลง "ค่านี้เป็น string" ให้กลายเป็น bytes ที่ตรงกับ format ของตัวเองต่อไป

พูดอีกแบบ: **`Serialize::serialize` คือ "บอกว่าค่านี้มีรูปร่างอะไร" ส่วน `Serializer` คือ "รู้ว่าจะเขียน
รูปร่างนั้นลงไปในรูปแบบไบต์ที่ตัวเองรับผิดชอบยังไง"** สอง concern นี้ถูกแยกออกจากกันอย่างสมบูรณ์ผ่าน
generic parameter — หลักการเดียวกันกับที่ Part 21 สอนเรื่อง trait object/`dyn Trait` แยก "สิ่งที่ทำได้"
ออกจาก "ทำยังไงจริง ๆ" เพียงแต่ตรงนี้ serde เลือกใช้ **static dispatch ผ่าน generic** (ไม่ใช่
`dyn Serializer`) เพราะต้องการ **zero-cost**: compiler จะ monomorphize `serialize::<JsonSerializer>`
แยกเป็นโค้ดเครื่องคนละชุดกับ `serialize::<YamlSerializer>` (ทวนจาก Part 22) ไม่มี virtual dispatch หรือ
`Box` เกิดขึ้นเลยตอน runtime

`S::Ok` และ `S::Error` เป็น **associated type** ของ trait `Serializer` — สิ่งที่คืนกลับตอน serialize
สำเร็จ (`Ok`) และ error type ตอน fail (`Error`) ก็เป็น generic ตาม `S` เช่นกัน (JSON serializer อาจคืน
`S::Ok = ()` และ `S::Error = serde_json::Error`, format อื่นอาจคืน type ต่างกัน) — เราไม่ต้องรู้ล่วงหน้า
ว่ามันคือ type อะไรกันแน่ เขียนโค้ดผ่าน `Result<S::Ok, S::Error>` ตรง ๆ ได้เสมอ

มาลอง implement `Serialize` ด้วยมือให้ type จริงตัวหนึ่งกัน: `Color` ที่เก็บ `r`, `g`, `b` เป็น `u8` แยก
กัน (ตรงกับ scenario 2 ในหัวข้อก่อนหน้า — derive ตรง ๆ จะได้ `{"r":26,"g":43,"b":60}` แต่เราต้องการ
`"#1A2B3C"` แทน):

```rust
use serde::{Serialize, Serializer};

#[derive(Debug, Clone, Copy, PartialEq)]
struct Color {
    r: u8,
    g: u8,
    b: u8,
}

impl Serialize for Color {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        let hex = format!("#{:02X}{:02X}{:02X}", self.r, self.g, self.b);
        serializer.serialize_str(&hex)
    }
}
```

สังเกตทีละส่วน:

- **`fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error> where S: Serializer`** — เขียน
  ตรงตาม signature ของ trait เป๊ะ (Rust บังคับให้ signature ของ `impl` ตรงกับ trait declaration เสมอ —
  ทวนจาก Part 19) `&self` คือ `Color` ที่เรากำลังจะแปลง `serializer: S` คือ "ตัวเขียน" ที่รับมาจากภายนอก
  (เราไม่ได้สร้างมันเอง คนเรียก `serde_json::to_string(&color)` เป็นคนสร้าง `S` แล้วส่งเข้ามาให้)
- **`format!("#{:02X}{:02X}{:02X}", ...)`** — นี่คือ**ตรรกะทางธุรกิจ**ล้วน ๆ ไม่มีอะไรเกี่ยวกับ serde
  เลย: แปลง `u8` สามตัวเป็น hex string สองหลักต่อค่า (`{:02X}` คือ format specifier ที่บังคับ 2 หลัก
  ตัวพิมพ์ใหญ่ เติม `0` ข้างหน้าถ้าจำเป็น — เช่น `r = 0x1a` จะได้ `"1A"` ไม่ใช่ `"1A"` ที่ขาดเลขศูนย์)
- **`serializer.serialize_str(&hex)`** — นี่คือจุดที่เรา**มอบหมาย**ให้ `Serializer` ทำงานต่อ
  `serialize_str` เป็น method หนึ่งใน trait `Serializer` (มี `serialize_i64`, `serialize_bool`,
  `serialize_map` ฯลฯ ให้เลือกตามชนิดข้อมูลที่เรามี — ในที่นี้เราตัดสินใจแล้วว่า representation
  ที่ถูกต้องของ `Color` คือ "string" ไม่ใช่ "struct/map" จึงเรียก `serialize_str` ไม่ใช่ `serialize_struct`)
  ค่าที่ return กลับมาคือ `S::Ok`/`S::Error` ของ `serializer` ตัวนั้นตรง ๆ — เราไม่ต้องแปลงอะไรเพิ่ม

ลองรันจริงกับ `serde_json`:

```rust
fn main() {
    let c = Color { r: 0x1a, g: 0x2b, b: 0x3c };
    let json = serde_json::to_string(&c).unwrap();
    println!("serialized: {json}");
}
```

ผลลัพธ์จริงจาก terminal:

```
serialized: "#1A2B3C"
```

ยืนยันว่า custom `Serialize` ทำงานถูกต้อง — ได้ `"#1A2B3C"` (string เดี่ยว ๆ) ไม่ใช่
`{"r":26,"g":43,"b":60}` แบบที่ derive จะให้ ข้อสังเกตสำคัญ: **โค้ดของเราไม่มีคำว่า `json` หรือ
`serde_json` อยู่เลยแม้แต่คำเดียว** ถ้าพรุ่งนี้มีคนเอา `Color` ไปใช้กับ `serde_yaml` หรือ `bincode`
`impl Serialize for Color` ตัวนี้ก็ยังใช้ได้เหมือนเดิมทุกตัวอักษร ไม่ต้องแก้ — นี่คือผลลัพธ์จริงของการ
ออกแบบ `S: Serializer` เป็น generic parameter ตามที่อธิบายไว้ข้างบน

### 58.2.1 คำศัพท์เต็มของ `Serializer`: ไม่ใช่แค่ `serialize_str`

`Color` ใช้ `serialize_str` เพราะ representation ที่เราเลือกคือ string แต่ trait `Serializer` มี method
ให้เลือกมากกว่านั้นมาก ครอบคลุมทุก "รูปร่างข้อมูลพื้นฐาน" ที่ format ส่วนใหญ่รองรับ:

| Method | ใช้เมื่อ representation คือ |
|---|---|
| `serialize_bool` | `bool` |
| `serialize_i8`...`serialize_i64` | จำนวนเต็มมีเครื่องหมาย |
| `serialize_u8`...`serialize_u64` | จำนวนเต็มไม่มีเครื่องหมาย |
| `serialize_f32`/`serialize_f64` | ทศนิยม |
| `serialize_char` | ตัวอักษรเดี่ยว |
| `serialize_str` | string (ใช้ใน `Color`, `Money`) |
| `serialize_bytes` | raw byte array (เร็วกว่า serialize เป็น seq ของ `u8` ทีละตัว) |
| `serialize_none`/`serialize_some` | `Option<T>` |
| `serialize_seq` | list/array ที่ความยาวไม่ตายตัว (คืน `S::SerializeSeq` ให้เรียก `.serialize_element(...)` ทีละตัว) |
| `serialize_map` | key-value map ที่ความยาวไม่ตายตัว |
| `serialize_struct` | struct ที่มีจำนวน field ตายตัว รู้ชื่อ field ล่วงหน้า (คืน `S::SerializeStruct`) |
| `serialize_tuple`/`serialize_tuple_struct` | tuple/tuple struct |

จุดที่ควรสังเกต: `serialize_struct` **แยกจาก** `serialize_map` แม้ผลลัพธ์ JSON หน้าตาเหมือนกัน (ทั้งคู่
กลายเป็น `{...}`) เพราะ format ที่ไม่ self-describing เท่า JSON (เช่น บาง binary format) ต้องรู้
**ล่วงหน้า**ว่าจำนวน field ตายตัวหรือไม่ตายตัว เพื่อเลือกวิธี encode ให้ประหยัดที่สุด — `serialize_map`
ต้อง encode ความยาวหรือ marker บอกจบ เพราะไม่รู้ล่วงหน้าว่ามีกี่ key แต่ `serialize_struct` รู้จำนวน
field แน่นอนตั้งแต่ compile time (เพราะ derive macro รู้จาก `DataStruct.fields` ทวนจาก Part 45) จึง
ไม่ต้อง encode ความยาวซ้ำอีกในบาง format ได้ นี่คือรายละเอียดที่ `serde_json` มองไม่เห็นความต่าง (มัน
render ทั้งคู่เป็น `{...}` เหมือนกัน) แต่ format binary ที่เน้น compact size ใช้ประโยชน์จากความต่างนี้จริง

### 58.2.2 พิสูจน์ว่า derive macro ก็แค่เรียก method พวกนี้เหมือนกัน

ทวนจาก Part 44-45: `#[derive(Serialize)]` เป็น proc macro ที่ generate `impl Serialize for YourType`
ให้อัตโนมัติ ไม่มีเวทมนตร์อะไรที่เราเข้าไม่ถึง — มาพิสูจน์ด้วยการเขียน `impl Serialize` ด้วยมือให้ struct
`Point` แบบเดียวกับที่ `#[derive(Serialize)]` จะ generate ให้ (ใช้ `serialize_struct` จากตารางข้างบน)
แล้วเทียบผลลัพธ์กับเวอร์ชัน derive ตรง ๆ:

```rust
use serde::ser::SerializeStruct;
use serde::{Serialize, Serializer};

#[derive(Serialize)]
struct PointDerived {
    x: i32,
    y: i32,
}

// เวอร์ชันมือ ที่ทำสิ่งเดียวกันกับที่ #[derive(Serialize)] ทำให้ PointDerived ทุกประการ
struct PointManual {
    x: i32,
    y: i32,
}

impl Serialize for PointManual {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        // "PointManual" คือชื่อ struct ส่งไปเผื่อ format ที่ต้องใช้ชื่อ type (เช่น บาง binary format)
        // 2 คือจำนวน field ที่จะเรียก serialize_field ทั้งหมด (derive macro รู้เลขนี้จาก
        // fields.named.len() ตอน compile time — ทวนจาก Part 45 หัวข้อ 45.2-45.3)
        let mut state = serializer.serialize_struct("PointManual", 2)?;
        state.serialize_field("x", &self.x)?;
        state.serialize_field("y", &self.y)?;
        state.end()
    }
}

fn main() {
    let derived = PointDerived { x: 3, y: 4 };
    let manual = PointManual { x: 3, y: 4 };
    println!("derived: {}", serde_json::to_string(&derived).unwrap());
    println!("manual:  {}", serde_json::to_string(&manual).unwrap());
}
```

ผลลัพธ์จริงจาก terminal:

```
derived: {"x":3,"y":4}
manual:  {"x":3,"y":4}
```

ผลลัพธ์**เหมือนกันทุกตัวอักษร** — ยืนยันว่า `#[derive(Serialize)]` ของ `serde_derive` ไม่ได้ทำอะไรที่
พิเศษไปกว่าการเรียก `serializer.serialize_struct(...)` แล้ว `state.serialize_field(...)` วนตาม field
ของ struct ทีละตัว (ตามที่ Part 45 สอนว่า derive macro generate `impl` block จาก `syn::Fields` ที่มัน
อ่านได้ — ในที่นี้คือวน `for field in &fields.named` แล้ว generate
`state.serialize_field(#field_name_str, &self.#field_ident)?;` หนึ่งบรรทัดต่อหนึ่ง field) ตัว
`SerializeStruct` (ที่ `serialize_struct` คืนมา) เป็น trait ย่อยที่มี method `serialize_field` และ `end`
— ออกแบบให้เป็น "handle ชั่วคราว" สำหรับเขียน field ทีละตัวแล้วปิดท้ายด้วย `end()` เพื่อบอก serializer
ว่า struct นี้เขียนครบแล้ว (บาง format ต้องรู้จุดจบชัดเจน เช่น ปิด `}` หรือเขียน byte สุดท้ายของ struct)

จุดสำคัญที่ทำให้เห็นภาพรวมของทั้งบท: **ทุกครั้งที่คุณเขียน `impl Serialize` ด้วยมือ (แบบ `Color` ใน
58.2 หรือแบบ `Point` ตรงนี้) คุณกำลังทำสิ่งเดียวกันกับที่ derive macro ทำให้อัตโนมัติ เพียงแต่คุณ
**เลือก representation เองได้** ว่าจะเรียก `serialize_str` (เมื่อต้องการ representation ที่ไม่ตรงกับ
field shape เลย อย่าง `Color`/`Money`) หรือ `serialize_struct` (เมื่อต้องการ shape ที่ตรงกับ field
เป๊ะ แต่มีเหตุผลอื่นที่ทำให้ derive ใช้ไม่ได้ เช่น orphan rule ในสถานการณ์ที่ 1 ของหัวข้อ 58.1)** — นี่
คือคำตอบที่สมบูรณ์ของคำถาม "derive macro ทำงานยังไงเบื้องหลัง" ที่ Part 44-45 ปูพื้นไว้ตลอดสองบทนั้น

### 58.3 Anatomy ของ trait `Deserialize`: ทำไมต้องมี Visitor pattern

ถ้า `Serialize` ง่ายเพราะเรา (คนเขียน `impl`) **รู้อยู่แล้ว**ว่า `self` มี field อะไรบ้าง — `Deserialize`
กลับยากกว่าโดยธรรมชาติ เพราะสถานการณ์ตรงข้ามกันสนิท: **เรายังไม่รู้ว่าข้อมูลที่กำลังจะมาถึงคือรูปร่าง
อะไร** จนกว่า deserializer จะบอกเรา

ลองนึกภาพ input JSON `"#1A2B3C"` (string) เข้ามา — deserializer (ของ `serde_json`) พอเห็น byte แรกเป็น
`"` ก็รู้ว่า "นี่คือ string แน่นอน" แต่ code ของเรา (คนเขียน `Deserialize for Color`) ไม่ได้เป็นคน parse
JSON เอง (นั่นเป็นงานของ `serde_json`) เราแค่**บอก**ว่า "ถ้าเจอ string ให้ทำอะไร" — และเพราะ format
ต่าง ๆ ส่งข้อมูลมาให้เราในรูปแบบที่ต่างกัน (JSON อาจส่งมาเป็น string, number, bool, null, array, object
— TOML อาจไม่มี concept ของ null เลย — MessagePack แยก integer ที่ signed/unsigned ชัดเจนกว่า JSON)
serde จึงต้องมีกลไกที่ให้เรา**ประกาศว่าเรารับข้อมูลรูปแบบไหนได้บ้าง** แล้วให้ deserializer เป็นคนเลือก
เรียก callback ที่ตรงกับรูปแบบที่มันเห็นจริง ๆ ในข้อมูล — กลไกนี้คือ **Visitor pattern**

trait `Visitor` (จาก `serde::de`) มี method สำหรับแต่ละรูปแบบข้อมูลที่เป็นไปได้ — `visit_str`,
`visit_i64`, `visit_bool`, `visit_map`, `visit_seq`, `visit_none`, ฯลฯ — ทุก method มี **default
implementation ที่ error ทันที** ("ฉันไม่รองรับรูปแบบนี้") คุณ override เฉพาะ method ที่ type ของคุณ
ยินดีรับเข้ามาเท่านั้น ที่เหลือปล่อยให้ default จัดการ (นี่คล้ายกับหลักการ "helper attribute" ใน
`#[proc_macro_derive(_, attributes(_))]` ที่ Part 45 สอนไว้ — เราไม่ต้องจัดการทุก case เอง แค่ประกาศ
ว่า case ไหนที่เรารองรับ)

trait `Deserialize<'de>` (สังเกตว่ามี **lifetime parameter `'de`** — จะอธิบายลึกในหัวข้อ 58.5) มี
หน้าที่แค่**เรียก method ของ `Deserializer` ที่บอกว่า "ฉันคาดหวังรูปแบบไหน" แล้วส่ง `Visitor` ของตัวเอง
เข้าไปให้**:

```rust
pub trait Deserialize<'de>: Sized {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: Deserializer<'de>;
}
```

มาดูขั้นตอนแบบเต็มด้วยการ implement `Deserialize` ให้ `Color` เดิม (parse hex string กลับเป็น struct):

```rust
use serde::de::{self, Visitor};
use serde::{Deserialize, Deserializer};
use std::fmt;

struct ColorVisitor;

impl<'de> Visitor<'de> for ColorVisitor {
    type Value = Color;

    fn expecting(&self, formatter: &mut fmt::Formatter) -> fmt::Result {
        formatter.write_str("a hex color string like \"#RRGGBB\"")
    }

    fn visit_str<E>(self, value: &str) -> Result<Color, E>
    where
        E: de::Error,
    {
        let s = value.strip_prefix('#').ok_or_else(|| {
            E::custom(format!("expected color string to start with '#', got {:?}", value))
        })?;
        if s.len() != 6 {
            return Err(E::custom(format!(
                "expected 6 hex digits after '#', got {} digits in {:?}",
                s.len(),
                value
            )));
        }
        let r = u8::from_str_radix(&s[0..2], 16)
            .map_err(|e| E::custom(format!("invalid red component in {:?}: {}", value, e)))?;
        let g = u8::from_str_radix(&s[2..4], 16)
            .map_err(|e| E::custom(format!("invalid green component in {:?}: {}", value, e)))?;
        let b = u8::from_str_radix(&s[4..6], 16)
            .map_err(|e| E::custom(format!("invalid blue component in {:?}: {}", value, e)))?;
        Ok(Color { r, g, b })
    }
}

impl<'de> Deserialize<'de> for Color {
    fn deserialize<D>(deserializer: D) -> Result<Color, D::Error>
    where
        D: Deserializer<'de>,
    {
        deserializer.deserialize_str(ColorVisitor)
    }
}
```

อธิบายทีละส่วนอย่างละเอียด:

**1) `type Value = Color;`** — associated type ที่บอกว่า "ถ้า visitor ตัวนี้สำเร็จ จะได้ค่าชนิดอะไร
กลับมา" ทุก `visit_*` method ของ `Visitor` ตัวนี้ต้อง return `Result<Self::Value, E>` คือ
`Result<Color, E>` ตรงกัน

**2) `fn expecting(&self, formatter: &mut fmt::Formatter) -> fmt::Result`** — นี่คือ method **บังคับ**
(ไม่มี default) ของ `Visitor` มีหน้าที่เดียว: อธิบายเป็นคำพูดว่า visitor ตัวนี้ "คาดหวัง" อะไร ข้อความนี้
จะถูกใช้**อัตโนมัติ**เมื่อ deserializer เจอข้อมูลรูปแบบที่เราไม่ได้ override method ไว้รับ (เช่น ถ้า
input เป็น number แต่เรา override แค่ `visit_str` — serde จะ generate error message ที่รวมข้อความจาก
`expecting()` เข้าไปให้เองโดยอัตโนมัติ เราจะเห็นตัวอย่างจริงด้านล่าง)

**3) `fn visit_str<E>(self, value: &str) -> Result<Color, E> where E: de::Error`** — นี่คือ callback ที่
เราเลือก override เพราะ `Color` ของเรารับข้อมูลจาก **string** เท่านั้น (`value: &str` คือ string ที่
deserializer ส่งมาให้ — มาจาก JSON string ที่ parse แล้ว) สังเกตว่า `E` เป็น generic parameter ของ
method นี้ (ไม่ใช่ของ `impl` block) ต้อง implement trait `de::Error` — เหตุผลเดียวกับที่ `S` ใน
`Serialize::serialize` เป็น generic: error type ต่างกันไปตามแต่ละ `Deserializer` (JSON มี error type
ของตัวเอง, YAML มีของตัวเอง) `visit_str` ของเราไม่ผูกติดกับ error type ไหนโดยเฉพาะ ใช้ `E::custom(...)`
สร้าง error ผ่าน trait method ที่ทุก error type ต้อง implement ได้เหมือนกัน

**4) การจัดการ error แบบ chain ด้วย `?`** — สังเกตว่าโค้ดใช้ `ok_or_else` และ `map_err` ร่วมกับ `?`
ตลอดทั้งฟังก์ชัน (ทวนจาก Part 12/30) แต่ละจุดที่ parse ผิดพลาดได้ (ไม่มี `#`, จำนวนหลักผิด, hex digit
parse ไม่ผ่าน) จะสร้าง error message ที่**อธิบายเจาะจงว่าอะไรผิดตรงไหน** แทนที่จะ panic หรือคืน error
ทั่ว ๆ ไปแบบ "invalid input" — นี่คือหลักการเดียวกับที่ Part 30 สอนเรื่องการออกแบบ error message ที่
"บอกสาเหตุจริง ไม่ใช่แค่บอกว่าผิด"

**5) `impl Deserialize<'de> for Color` เพียงแค่ 3 บรรทัด** — งานทั้งหมดของ `Deserialize::deserialize`
คือเรียก `deserializer.deserialize_str(ColorVisitor)` — บอก deserializer ว่า "ฉันคาดหวัง string เข้ามา
ถ้าเจอ ช่วยเรียก `ColorVisitor::visit_str` ให้ที" ส่วนตรรกะจริงทั้งหมดอยู่ใน `Visitor` implementation
ข้างบน `Deserializer::deserialize_str` (ตรงข้ามกับ `deserialize_any`) เป็นการ**บอก hint** ให้ format
ที่ self-describing น้อยกว่า JSON (เช่น บาง binary format) รู้ว่าควร parse ต่อยังไงโดยไม่ต้องเดา — ส่วน
JSON เองรู้ชนิดข้อมูลจาก syntax อยู่แล้ว (`"..."` คือ string เสมอ) จึงไม่กระทบพฤติกรรมมากนักในกรณีนี้
แต่เป็น best practice ที่ถูกต้องเสมอ

มาดูผลลัพธ์จริงแบบครบวงจร (round-trip + error case):

```rust
fn main() {
    let c = Color { r: 0x1a, g: 0x2b, b: 0x3c };
    let json = serde_json::to_string(&c).unwrap();
    println!("serialized: {json}");

    let back: Color = serde_json::from_str(&json).unwrap();
    println!("deserialized: {back:?}");
    assert_eq!(c, back);

    let bad = serde_json::from_str::<Color>("\"not-a-color\"");
    println!("bad hex error: {}", bad.unwrap_err());

    let bad_len = serde_json::from_str::<Color>("\"#ABC\"");
    println!("bad length error: {}", bad_len.unwrap_err());

    let wrong_type = serde_json::from_str::<Color>("42");
    println!("wrong type error: {}", wrong_type.unwrap_err());
}
```

ผลลัพธ์จริงจาก terminal (คัดลอกตรง ๆ ไม่มีตัดต่อ):

```
serialized: "#1A2B3C"
deserialized: Color { r: 26, g: 43, b: 60 }
bad hex error: expected color string to start with '#', got "not-a-color" at line 1 column 13
bad length error: expected 6 hex digits after '#', got 3 digits in "#ABC" at line 1 column 6
wrong type error: invalid type: integer `42`, expected a hex color string like "#RRGGBB" at line 1 column 2
```

สามบรรทัดสุดท้ายคือหลักฐานสำคัญที่สุดของหัวข้อนี้:

- **บรรทัดที่ 3-4** มาจาก `E::custom(...)` ที่เราเขียนเอง — ข้อความตรงกับที่เราสร้างไว้เป๊ะ พร้อม
  `at line 1 column N` ที่ `serde_json` **เติมให้เองอัตโนมัติ** (เราไม่ต้องคำนวณตำแหน่งเอง —
  `serde_json::Error` แนบตำแหน่งใน input buffer ที่กำลัง parse ไว้ให้ทุก error ไม่ว่าจะมาจาก error
  ภายในของมันเองหรือจาก `E::custom` ของเรา)
- **บรรทัดสุดท้าย** คือ error ที่เรา**ไม่ได้เขียนเอง** — เกิดจากการที่ input เป็น `42` (number) แต่
  `ColorVisitor` ไม่ได้ override `visit_i64`/`visit_u64` เลย จึงตกไปใช้ default implementation ของ
  `Visitor` ที่ generate ข้อความ `"invalid type: integer `42`, expected ..."` โดยเอาข้อความจาก
  `expecting()` ของเรา (`"a hex color string like \"#RRGGBB\""`) ไปต่อท้ายให้อัตโนมัติ — นี่คือเหตุผล
  ที่ `expecting()` เป็น method บังคับ ไม่มี default ให้ข้าม: ถ้าไม่มีมัน serde จะไม่มีข้อความให้ต่อท้าย
  ตรงจุดนี้เลย

จุดสังเกตเชิงลึกสุดท้าย: **เราไม่ต้อง override `visit_i64` เพื่อ reject number เอง** — การไม่ override
เลยก็ทำให้ปฏิเสธค่าที่ผิดชนิดได้อัตโนมัติแล้ว (ผ่าน default implementation ของ `Visitor` trait) นี่คือ
พลังของ Visitor pattern: **คุณต้องเขียนโค้ดสำหรับ shape ที่ยอมรับเท่านั้น shape ที่ไม่ยอมรับถูกปฏิเสธ
ให้ฟรี ๆ โดยไม่ต้องเขียนอะไรเพิ่มเลย**

### 58.3.1 มองลึกอีกชั้น: `MapAccess` และสิ่งที่ `#[derive(Deserialize)]` generate ให้ named-field struct

หัวข้อ 58.2.2 พิสูจน์ไปแล้วว่า `#[derive(Serialize)]` ก็แค่เรียก `serialize_struct` เหมือนที่เราเขียนมือ
ได้ ฝั่ง `Deserialize` ก็มีคำตอบแบบเดียวกัน แต่ซับซ้อนกว่าเล็กน้อยเพราะ (ตามที่ 58.3 อธิบายไว้) ต้องรับมือ
กับ **key ที่มาไม่เรียงลำดับ** ได้ด้วย (JSON object ไม่บังคับว่า key ต้องมาตามลำดับที่ struct ประกาศไว้ —
`{"y":4,"x":3}` ก็ต้อง parse ได้ถูกต้องเหมือน `{"x":3,"y":4}`) กลไกที่ทำสิ่งนี้คือ **`MapAccess`** —
trait ที่ให้ `Visitor` เรียก `next_key()`/`next_value()` วนไปทีละคู่จนกว่าจะหมด ผสานกับ **field enum**
ที่แปลง key string เป็น enum variant ก่อน (เร็วกว่า match string ตรง ๆ ทุกครั้ง)

```rust
use serde::de::{self, MapAccess, Visitor};
use serde::{Deserialize, Deserializer};
use std::fmt;

#[derive(Debug, Deserialize)]
struct PointDerived {
    x: i32,
    y: i32,
}

// เวอร์ชันมือ ที่จำลองแนวคิดของสิ่งที่ #[derive(Deserialize)] generate ให้ PointDerived
#[derive(Debug)]
struct PointManual {
    x: i32,
    y: i32,
}

// field enum: แทนชื่อ field แต่ละตัวด้วย variant (derive macro generate enum นี้ให้เองจาก
// fields.named ทวนจาก Part 45 — ชื่อ field ทุกตัวกลายเป็น variant หนึ่งของ enum นี้)
enum Field {
    X,
    Y,
}

impl<'de> Deserialize<'de> for Field {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: Deserializer<'de>,
    {
        struct FieldVisitor;
        impl<'de> Visitor<'de> for FieldVisitor {
            type Value = Field;
            fn expecting(&self, f: &mut fmt::Formatter) -> fmt::Result {
                f.write_str("`x` or `y`")
            }
            fn visit_str<E>(self, value: &str) -> Result<Field, E>
            where
                E: de::Error,
            {
                match value {
                    "x" => Ok(Field::X),
                    "y" => Ok(Field::Y),
                    other => Err(de::Error::unknown_field(other, &["x", "y"])),
                }
            }
        }
        // deserialize_identifier บอก deserializer ว่า "ค่านี้คือชื่อ key ของ map/struct"
        // (ต่างจาก deserialize_str ธรรมดา — บาง format optimize การอ่านชื่อ field ได้ต่างจากการ
        // อ่าน string value ทั่วไป)
        deserializer.deserialize_identifier(FieldVisitor)
    }
}

struct PointManualVisitor;

impl<'de> Visitor<'de> for PointManualVisitor {
    type Value = PointManual;

    fn expecting(&self, f: &mut fmt::Formatter) -> fmt::Result {
        f.write_str("struct PointManual")
    }

    fn visit_map<A>(self, mut map: A) -> Result<PointManual, A::Error>
    where
        A: MapAccess<'de>,
    {
        let mut x: Option<i32> = None;
        let mut y: Option<i32> = None;
        // วน map ทีละคู่ key-value จนกว่า next_key จะคืน None (หมด map)
        // key ไม่ต้องมาตามลำดับ x, y เสมอ — loop นี้รองรับลำดับไหนก็ได้
        while let Some(key) = map.next_key::<Field>()? {
            match key {
                Field::X => {
                    if x.is_some() {
                        return Err(de::Error::duplicate_field("x"));
                    }
                    x = Some(map.next_value()?);
                }
                Field::Y => {
                    if y.is_some() {
                        return Err(de::Error::duplicate_field("y"));
                    }
                    y = Some(map.next_value()?);
                }
            }
        }
        let x = x.ok_or_else(|| de::Error::missing_field("x"))?;
        let y = y.ok_or_else(|| de::Error::missing_field("y"))?;
        Ok(PointManual { x, y })
    }
}

impl<'de> Deserialize<'de> for PointManual {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: Deserializer<'de>,
    {
        deserializer.deserialize_struct("PointManual", &["x", "y"], PointManualVisitor)
    }
}

fn main() {
    let json = r#"{"x":3,"y":4}"#;
    let derived: PointDerived = serde_json::from_str(json).unwrap();
    let manual: PointManual = serde_json::from_str(json).unwrap();
    println!("derived: {derived:?}");
    println!("manual:  {manual:?}");

    let missing = serde_json::from_str::<PointManual>(r#"{"x":3}"#);
    println!("missing y: {}", missing.unwrap_err());
}
```

ผลลัพธ์จริงจาก terminal:

```
derived: PointDerived { x: 3, y: 4 }
manual:  PointManual { x: 3, y: 4 }
missing y: missing field `y` at line 1 column 7
```

อธิบายทีละส่วนที่สำคัญ:

- **`Field` enum + `Deserialize` ของมันเอง** — สังเกตว่า `Field` เป็น**อีก type หนึ่งที่ implement
  `Deserialize` เอง แยกจาก `PointManual`** นี่คือรูปแบบที่ derive macro ของ serde ใช้จริง: มันไม่ match
  string ตรง ๆ ใน `visit_map` แต่สร้าง helper type เล็ก ๆ ขึ้นมาก่อน ให้ deserializer "แปลง" key string
  เป็น enum ให้เสร็จก่อน ค่อยเอา enum นั้นมา `match` ต่อ (เร็วกว่าและอ่านง่ายกว่าการเทียบ string ยาว ๆ
  ในลูปหลัก)
- **`map.next_key::<Field>()?` แล้ว `match`** — นี่คือรูปแบบเดียวกับ pattern matching ปกติที่ Part 10
  สอนไว้ เพียงแต่ตรงนี้ pattern มาจาก loop ที่ deserializer ควบคุมจำนวนรอบ ไม่ใช่ data structure ที่เรา
  มีอยู่แล้ว — `x`/`y` เป็น `Option<i32>` ระหว่างวน (ยังไม่รู้ว่าจะเจอ key ไหนก่อน) แล้วค่อยแปลงเป็น
  `i32` ที่ไม่ใช่ `Option` ด้วย `ok_or_else` ตอนจบ (ทวนจาก Part 11: `Option<T>` เป็นตัวแทน "ยังไม่มีค่า"
  ที่ปลอดภัยกว่า sentinel value)
- **`de::Error::duplicate_field`/`missing_field`/`unknown_field`** — เป็น constructor สำเร็จรูปที่
  `serde::de::Error` เตรียมไว้ให้ (เหมือนกับ `Error::custom` ที่ใช้มาตลอดบท แต่เจาะจงกว่า — สร้าง error
  message มาตรฐานที่ผู้ใช้ serde ทุกคนคุ้นเคยรูปแบบ ไม่ต้องเขียน message เองด้วย `format!`)
  ผลลัพธ์ `missing field \`y\` at line 1 column 7` มาจาก `missing_field("y")` ตรง ๆ — ข้อความเดียวกัน
  กับที่ derive จริงให้เมื่อ struct ธรรมดาขาด field ที่จำเป็น

**ข้อจำกัดของตัวอย่างนี้ที่ควรรู้ตรง ๆ (เพื่อความถูกต้อง ไม่ให้เข้าใจผิด):** เวอร์ชันมือด้านบน**เข้มงวด
กว่า** derive จริงเล็กน้อย — ถ้าลองส่ง key ที่ไม่รู้จักเข้ามา (เช่น `{"x":3,"y":4,"z":5}`)
`PointManual` จะ error ทันที (เพราะ `FieldVisitor::visit_str` เจอ `"z"` แล้วคืน
`Err(unknown_field(...))` ทันที) แต่ `PointDerived` (derive จริง โดยไม่มี
`#[serde(deny_unknown_fields)]`) จะ**ยอมรับและข้าม** field ที่ไม่รู้จักไปเงียบ ๆ ตามค่า default ของ
serde (ทวนจาก Part 57 — `deny_unknown_fields` เป็น attribute ที่ทำให้พฤติกรรมนี้เข้มงวดขึ้น) เหตุผล
ที่ derive จริงทำแบบนั้นได้คือ field enum ที่มันสร้างจริงมี variant พิเศษเพิ่มมาอีกตัว (สำหรับ "key
ที่ไม่ตรงกับ field ไหนเลย ให้ข้ามค่านั้นไปด้วย `serde::de::IgnoredAny`") ซึ่งเราตัดออกในตัวอย่างนี้เพื่อ
ความกระชับ — แก่นของกลไก (field enum + `MapAccess` loop) เหมือนกันเป๊ะ มีแค่รายละเอียดปลีกย่อยเรื่อง
unknown-field handling ที่เราตัดออกไปเพื่อให้โค้ดอ่านง่ายขึ้น

โครงสร้าง "field enum + `MapAccess` loop" นี้คือคำตอบสมบูรณ์ของคำถามที่ตั้งไว้ตอนต้นหัวข้อ 58.3: ทำไม
`Deserialize` ต้องมี `Visitor` — เพราะ deserializer ต้อง**ค้นพบ**ว่าเจอ key ไหนบ้างระหว่างวน map แบบ
real-time (ไม่รู้ล่วงหน้า) แล้วส่งกลับให้ visitor ตัดสินใจทีละ key — ต่างจาก `Serialize` ที่เรา (คนเขียน)
รู้ field ทั้งหมดอยู่แล้วตั้งแต่ก่อนเริ่มเขียนโค้ดเลย

### 58.4 ทางลัดที่ปฏิบัติได้จริง: `#[serde(with = "module")]`, `serialize_with`, `deserialize_with`

การ implement `Serialize`/`Deserialize` เต็มรูปแบบแบบหัวข้อ 58.2-58.3 คุ้มค่าเมื่อ**ทั้ง type**
ต้องการ representation พิเศษ (เช่น `Color` ทั้งตัวเป็น string) แต่ในโปรเจกต์จริง สถานการณ์ที่พบบ่อยกว่า
คือ: struct มีหลาย field และ**field เดียว**ต้องการ custom logic ส่วน field อื่น ๆ derive ตามปกติได้
สบาย ๆ — ถ้าต้อง implement `Serialize`/`Deserialize` เต็มรูปแบบทั้ง struct แค่เพราะ field เดียว จะเสีย
ประโยชน์ของ derive สำหรับ field ที่เหลือไปหมด (ต้องเขียน `serialize_struct` เปิด field ทีละตัวเอง —
Part 57 ไม่ได้สอนราย ละเอียดของ `serialize_struct` เพราะมันยาวและ error-prone มาก)

serde จึงมี attribute พิเศษสามตัวที่ทำงานร่วมกับ derive: `#[serde(with = "module")]`,
`#[serde(serialize_with = "path::to::fn")]`, และ `#[serde(deserialize_with = "path::to::fn")]` —
attribute เหล่านี้บอก derive macro ว่า **"field นี้ ให้เรียกฟังก์ชันนี้แทนการ generate โค้ดปกติ"**
ส่วน field อื่นในโครงสร้างเดียวกันยัง derive ตามปกติทุกอย่าง

`with = "module"` คือทางลัดที่สะดวกที่สุดเมื่อทั้ง serialize และ deserialize logic อยู่ในที่เดียวกัน —
มันเทียบเท่ากับเขียน `serialize_with = "module::serialize"` และ `deserialize_with =
"module::deserialize"` พร้อมกันทั้งคู่ (module ต้องมีฟังก์ชันชื่อ `serialize` และ `deserialize`
signature ตรงตามที่ derive คาดหวังเป๊ะ)

มาดูตัวอย่างจริงที่พบบ่อยที่สุดในโลก production: field ที่เป็น `chrono::DateTime<Utc>` ที่ต้องการ format
เป็น string เฉพาะ (ไม่ใช่ ISO-8601 default ที่ `chrono` มี `serde` feature ให้อยู่แล้ว):

```rust
use chrono::{DateTime, TimeZone, Utc};
use serde::{Deserialize, Serialize};

mod custom_datetime_format {
    use chrono::{DateTime, NaiveDateTime, Utc};
    use serde::{Deserialize, Deserializer, Serializer};

    const FORMAT: &str = "%Y-%m-%d %H:%M:%S";

    pub fn serialize<S>(date: &DateTime<Utc>, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        let s = date.format(FORMAT).to_string();
        serializer.serialize_str(&s)
    }

    pub fn deserialize<'de, D>(deserializer: D) -> Result<DateTime<Utc>, D::Error>
    where
        D: Deserializer<'de>,
    {
        let s = String::deserialize(deserializer)?;
        let naive = NaiveDateTime::parse_from_str(&s, FORMAT).map_err(serde::de::Error::custom)?;
        Ok(naive.and_utc())
    }
}

#[derive(Debug, Serialize, Deserialize)]
struct LogEntry {
    message: String,
    #[serde(with = "custom_datetime_format")]
    created_at: DateTime<Utc>,
}

fn main() {
    let entry = LogEntry {
        message: "server started".to_string(),
        created_at: Utc.with_ymd_and_hms(2026, 9, 26, 10, 30, 0).unwrap(),
    };
    let json = serde_json::to_string_pretty(&entry).unwrap();
    println!("{json}");

    let back: LogEntry = serde_json::from_str(&json).unwrap();
    println!("{back:?}");
    assert_eq!(entry.created_at, back.created_at);
}
```

ผลลัพธ์จริงจาก terminal:

```
{
  "message": "server started",
  "created_at": "2026-09-26 10:30:00"
}
LogEntry { message: "server started", created_at: 2026-09-26T10:30:00Z }
```

จุดที่ต้องเข้าใจให้ลึก:

**1) signature ของฟังก์ชันใน module ต้องตรงกับที่ `serde_derive` คาดหวังเป๊ะ** — `serialize` ต้องรับ
`&T` (ไม่ใช่ `T`) เป็น argument แรก และ `S: Serializer` เป็น argument ที่สอง คืน `Result<S::Ok,
S::Error>` — นี่คือ signature เดียวกันกับ `Serialize::serialize` ทุกตัวอักษร เพียงแต่เขียนเป็นฟังก์ชัน
แยกต่างหาก ไม่ใช่ method ใน `impl Serialize for T` เหตุผลที่ derive macro ต้องการ signature นี้เป๊ะคือ
มันจะ generate โค้ดประมาณ `custom_datetime_format::serialize(&self.created_at, serializer_for_this_field)?`
เข้าไปแทนตำแหน่งที่ปกติจะ generate `self.created_at.serialize(serializer_for_this_field)?` — สลับจาก
method call เป็น free function call ตรง ๆ เท่านั้น ถ้า signature ไม่ตรง compiler จะฟ้อง type mismatch
ทันที (ดูตัวอย่าง error จริงในหัวข้อกับดัก 58.9.5)

**2) `deserialize` เขียนโดยใช้ `String::deserialize(deserializer)?` ก่อน แล้ว parse ต่อด้วยมือ** — นี่
คือเทคนิคที่ใช้ได้บ่อยมาก: ไม่ต้องเขียน `Visitor` เองทุกครั้ง ถ้า representation บน wire เป็น string
อยู่แล้ว ให้ deserialize เป็น `String` (type ที่ serde มี `Deserialize` ให้แล้ว) ก่อน แล้ว parse
string นั้นด้วยตรรกะของตัวเอง — สั้นกว่าการเขียน `Visitor` เต็มรูปแบบมาก (เหมาะกับกรณีที่ input เป็น
string เท่านั้นแน่ ๆ ไม่ต้องรองรับหลาย shape เหมือน `Color` ในหัวข้อก่อน)

**3) `NaiveDateTime::parse_from_str(...).map_err(serde::de::Error::custom)?`** — เห็นรูปแบบเดียวกันกับ
หัวข้อ 58.3: แปลง error จาก library ภายนอก (`chrono::ParseError`) ให้กลายเป็น error type ที่ deserializer
ต้องการ (`D::Error`) ผ่าน `serde::de::Error::custom` — `map_err` รับ function pointer ตรง ๆ ได้เพราะ
`Error::custom` เป็น generic function ที่รับ `T: Display` (และ `chrono::ParseError` implement
`Display`) จึง pass เป็น `fn` ตรง ๆ ได้โดยไม่ต้องเขียน closure `|e| serde::de::Error::custom(e)`

**4) field อื่น (`message: String`) ไม่ได้รับผลกระทบเลย** — derive ยัง generate โค้ดปกติให้มันทุกอย่าง
นี่คือประโยชน์หลักของ `with =`: ได้ทั้ง**ความสะดวกของ derive สำหรับ field ส่วนใหญ่** และ**ความยืดหยุ่น
ของ manual implementation สำหรับ field ที่ต้องการมันจริง ๆ** ในไฟล์เดียวกัน

ถ้าต้องการแค่ทิศทางเดียว (เช่น serialize ปรับแต่ง แต่ deserialize ใช้ default ปกติได้) ใช้
`#[serde(serialize_with = "...")]` เดี่ยว ๆ ได้ (ไม่ต้องมี `with =` ทั้ง module) — Part 57 ไม่ได้ลง
รายละเอียดตรงนี้เพราะมันคาบเกี่ยวกับการเขียน `Serialize`/`Deserialize` เอง ซึ่งเป็นเนื้อหาของบทนี้

### 58.5 Deserialize กับ Lifetime: Zero-Copy Deserialization

ทวนจาก Part 20/23: reference (`&'a T`) ผูกอายุการใช้งานไว้กับข้อมูลต้นทาง — ยืมได้แต่ต้องไม่อยู่นานกว่า
เจ้าของข้อมูล เรื่องนี้เกี่ยวข้องกับ `Deserialize` โดยตรง เพราะ trait จริงของมันมี **lifetime parameter
ของตัวเอง**:

```rust
pub trait Deserialize<'de>: Sized {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: Deserializer<'de>;
}
```

`'de` คือ lifetime ของ**input buffer ที่กำลังถูก deserialize** — เหตุผลที่มันสำคัญคือ serde เปิดช่อง
ให้ type ที่ deserialize ออกมา**ยืม (borrow)** ข้อมูลตรงจาก input buffer ได้เลย โดยไม่ต้อง allocate
หน่วยความจำใหม่ — เทคนิคนี้เรียกว่า **zero-copy deserialization**

ลองเทียบสองแบบให้เห็นภาพ: การ deserialize field ที่เป็น string ปกติ (`String`) ทุกครั้งที่
`serde_json` เจอ JSON string มันต้อง (1) หา boundary ของ string ใน input buffer, (2) unescape
`\"`/`\\`/`\n` ฯลฯ ถ้ามี, (3) **copy** ข้อมูลที่ unescape แล้วไปไว้ใน heap allocation ใหม่ (`String`)
ขั้นตอนที่ 3 คือต้นทุนที่หลีกเลี่ยงได้ในหลายกรณี — ถ้า string **ไม่มี escape sequence เลย** (กรณีส่วน
ใหญ่ในโลกจริง เช่น ชื่อ, ID, tag) ข้อมูลที่ต้องการก็คือ**ช่วง byte ต่อเนื่องเดียวกันกับที่อยู่ใน input
buffer อยู่แล้ว** — การ copy มันไปที่อื่นจึงเป็นงานที่ไม่จำเป็น

`Deserialize<'de>` เปิดช่องให้ทำแบบนี้ได้ผ่าน field ที่เป็น `&'de str` (หรือ `&'de [u8]`) แทน `String`:

```rust
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct Event<'a> {
    name: &'a str,
    tags: Vec<&'a str>,
}

fn main() {
    let input = String::from(r#"{"name":"deploy","tags":["prod","urgent"]}"#);
    let event: Event = serde_json::from_str(&input).unwrap();
    println!("{event:?}");

    let buf_start = input.as_ptr() as usize;
    let buf_end = buf_start + input.len();
    let name_ptr = event.name.as_ptr() as usize;
    println!(
        "name ptr inside original buffer: {}",
        name_ptr >= buf_start && name_ptr < buf_end
    );
}
```

ผลลัพธ์จริงจาก terminal:

```
Event { name: "deploy", tags: ["prod", "urgent"] }
name ptr inside original buffer: true
```

บรรทัดสุดท้ายคือ**หลักฐานที่จับต้องได้จริง** ไม่ใช่แค่คำอธิบายเชิงทฤษฎี: เราเอา pointer ของ
`event.name` มาเทียบกับช่วง address ของ `input` (buffer ต้นทาง) แล้วพบว่า `name_ptr` อยู่**ภายใน**
ช่วงนั้นจริง ๆ — แปลว่า `event.name` **ไม่ได้ copy ข้อมูลไปที่ไหนเลย** มันคือ slice ที่ชี้กลับไปยัง
ตำแหน่งเดิมใน `input` ตรง ๆ ต่างจาก `struct Event { name: String }` ที่จะต้อง allocate heap block ใหม่
แล้ว copy ตัวอักษร `"deploy"` ไปไว้ที่นั่น — สำหรับ struct ที่มีหลาย string field และต้อง deserialize
เอกสารจำนวนมาก (เช่น parsing log file เป็นล้านบรรทัด) ผลต่างด้าน performance จากการไม่ allocate/copy
ซ้ำ ๆ นี้มีความหมายจริงในระดับ production — นี่คือเหตุผลที่ crate อย่าง `simd-json` หรือโค้ด
performance-critical จำนวนมากพยายาม design struct ให้ borrow จาก input buffer ให้ได้มากที่สุด

**สิ่งที่ต้องแลกมา: constraint ด้าน lifetime ที่เข้มงวดขึ้น** — เพราะ `event.name` ยืม memory มาจาก
`input` ตรง ๆ **`event` ใช้งานได้นานสุดแค่ที่ `input` ยังมีชีวิตอยู่เท่านั้น** เหมือน reference ธรรมดา
ทุกประการ (Part 20 สอนกฎนี้ไว้แล้ว) ลองดูว่าเกิดอะไรขึ้นถ้าเราพยายาม return `Event` ที่ยืมมาจาก `String`
ที่สร้างขึ้นในฟังก์ชันเดียวกัน (แล้วจะถูกทำลายทันทีที่ฟังก์ชัน return):

```rust
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct Event<'a> {
    name: &'a str,
}

fn parse_event() -> Event<'static> {
    let input = String::from(r#"{"name":"deploy"}"#);
    let event: Event = serde_json::from_str(&input).unwrap();
    event
}

fn main() {
    let e = parse_event();
    println!("{e:?}");
}
```

compile error จริง (คัดลอกจาก terminal ตรง ๆ):

```
error[E0515]: cannot return value referencing local variable `input`
  --> src/main.rs:11:5
   |
10 |     let event: Event = serde_json::from_str(&input).unwrap();
   |                                             ------ `input` is borrowed here
11 |     event
   |     ^^^^^ returns a value referencing data owned by the current function
```

borrow checker จับได้ทันทีตอน compile: `input` เป็น local variable ที่ถูกทำลาย (dropped) ทันทีที่
`parse_event` return แต่ `event` ยืม memory จากมันอยู่ — ถ้าปล่อยให้ compile ผ่าน `event` ที่ return
ออกไปจะกลายเป็น **dangling reference** (ชี้ไปยัง memory ที่ถูกคืนไปแล้ว) ทันที — นี่คือ error ตระกูล
เดียวกันกับที่ Part 20 สอนไว้ทุกตัวอักษร (return reference ที่ชี้ไปยัง local variable) เพียงแต่ตอนนี้
เกิดผ่าน `Deserialize<'de>` แทนที่จะเป็น struct/function ที่เขียนมือปกติ — **ยืนยันว่า lifetime
constraint ของ zero-copy deserialization คือ borrow checker rule เดียวกันเป๊ะ ไม่ใช่กฎพิเศษของ serde**

วิธีแก้ในสถานการณ์จริงคือ**ปรับ ownership ของ caller** — ให้ caller เป็นเจ้าของ `input` (`String`)
แล้วส่ง `&str` เข้าไปให้ฟังก์ชันที่ deserialize เท่านั้น (แทนที่ฟังก์ชันจะสร้าง `String` เองแล้วพยายาม
return ค่าที่ยืมจากมัน) — เหมือนตัวอย่างแรกในหัวข้อนี้ที่ `input` อยู่ใน `main` และ `event` ใช้งานอยู่ใน
scope เดียวกันตลอด ไม่มีปัญหาเรื่อง lifetime เลย

**เมื่อไหร่ควรใช้ zero-copy กับเมื่อไหร่ควรใช้ `String` ปกติ:** ใช้ `&'a str` เมื่อ (1) คุณควบคุม
lifetime ของ input buffer ได้ง่าย (เก็บ buffer ไว้ตราบเท่าที่ struct ที่ deserialize ออกมายังใช้อยู่),
และ (2) performance ของการ deserialize จำนวนมากสำคัญจริง ๆ (เช่น parsing log/message queue ปริมาณสูง)
ถ้า struct ที่ deserialize ออกมาต้องถูกเก็บไว้นาน ส่งข้าม thread, หรือ ownership ซับซ้อน — `String`
ปกติเรียบง่ายกว่าและปลอดภัยกว่ามาก ไม่ต้องพก lifetime parameter ติดไปทุกที่ที่ type นั้นถูกใช้ (ทวนจาก
Part 23: lifetime parameter ที่ผูกกับ struct จะ "แพร่กระจาย" ไปทุกที่ที่ struct นั้นถูกใช้เสมอ) —
นี่คือ trade-off ระหว่าง performance กับความเรียบง่ายที่ต้องชั่งน้ำหนักตามบริบทจริงของแต่ละโปรเจกต์

### 58.6 Custom Validation ระหว่าง Deserialize

ทวนจาก Part 53: **newtype ที่บังคับ invariant ผ่าน constructor** คือ pattern ที่ทำให้ค่าที่ผิดกฎไม่
สามารถถูกสร้างขึ้นมาได้เลยตั้งแต่ต้น (เช่น `struct Percentage(f64)` ที่มี `fn new(v: f64) ->
Result<Self, String>` เป็นทางเดียวในการสร้างค่า ไม่มี public constructor อื่นให้ bypass ได้) หัวข้อนี้
จะเอา pattern เดียวกันมาผสานกับ serde — เพราะ **derive `Deserialize` ตรง ๆ จะ bypass invariant นี้
ทันที** (มันสร้างค่าผ่าน field ตรง ๆ ไม่ผ่าน constructor ที่คุณเขียนไว้เลย)

```rust
use serde::{Deserialize, Deserializer, Serialize};

#[derive(Debug, Clone, Copy, PartialEq, Serialize)]
struct Percentage(f64);

impl<'de> Deserialize<'de> for Percentage {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: Deserializer<'de>,
    {
        let value = f64::deserialize(deserializer)?;
        if !(0.0..=100.0).contains(&value) {
            return Err(serde::de::Error::custom(format!(
                "percentage must be between 0.0 and 100.0, got {value}"
            )));
        }
        Ok(Percentage(value))
    }
}
```

จุดสำคัญ: **`Percentage` ไม่มี `#[derive(Deserialize)]` เลย** เราตั้งใจ implement เองเพราะเหตุผล
ทางธุรกิจ (validation) ไม่ใช่เพราะ shape ไม่ตรง (shape ตรงกันเป๊ะ — เป็น field เดียว `f64`) ข้อความนี้
สำคัญ: **derive กับ manual implementation ไม่ใช่ทางเลือกที่ผูกกับ "shape ตรงหรือไม่ตรง" เท่านั้น
validation ล้วน ๆ ก็เป็นเหตุผลที่ดีพอที่จะเขียนมือได้เช่นกัน** แม้ shape จะเหมือนกับที่ derive จะสร้าง
ให้ทุกประการ

โค้ดข้างในตรงไปตรงมา: `f64::deserialize(deserializer)?` ใช้ `Deserialize` ของ `f64` (ที่ serde มีให้
built-in) เพื่อดึงค่า `f64` ธรรมดาออกมาก่อน แล้วเช็ค range ด้วย `!(0.0..=100.0).contains(&value)`
(ทวนจาก Part 13: range pattern `a..=b` ใน Rust ใช้กับ `.contains()` ได้ตรง ๆ) ถ้าอยู่นอกช่วง —
**ไม่มีทางสร้าง `Percentage` ได้เลย** เพราะฟังก์ชันคืน `Err` ก่อนจะถึงบรรทัด `Ok(Percentage(value))`

ผลลัพธ์จริง:

```
ok = Percentage(42.5)
err = percentage must be between 0.0 and 100.0, got 142.5
err2 = percentage must be between 0.0 and 100.0, got -5
```

สังเกต `err2`: ค่า `-5.0` ถูก format เป็น `-5` (ไม่ใช่ `-5.0`) — นี่ไม่ใช่บั๊ก แต่เป็นพฤติกรรมมาตรฐานของ
`Display` สำหรับ `f64` ใน Rust (ไม่แสดง `.0` ต่อท้ายถ้าค่าเป็นจำนวนเต็มพอดี) เป็นรายละเอียดเล็ก ๆ ที่ควร
รู้ไว้เมื่ออ่าน error message ที่มี float ปนอยู่

**ทางเลือกที่สอง: ใช้ `deserialize_with` เมื่อ struct มีหลาย field และมีแค่ field เดียวที่ต้อง
validate** (ผสานกับหัวข้อ 58.4):

```rust
fn validate_percentage<'de, D>(deserializer: D) -> Result<f64, D::Error>
where
    D: Deserializer<'de>,
{
    let value = f64::deserialize(deserializer)?;
    if !(0.0..=100.0).contains(&value) {
        return Err(serde::de::Error::custom(format!(
            "progress must be between 0.0 and 100.0, got {value}"
        )));
    }
    Ok(value)
}

#[derive(Debug, Deserialize)]
struct Task {
    name: String,
    #[serde(deserialize_with = "validate_percentage")]
    progress: f64,
}
```

ตรงนี้ `Task::progress` ยังเป็น `f64` ธรรมดา (ไม่ต้องห่อเป็น newtype `Percentage` ถ้าไม่ต้องการ
ใช้ type นี้ที่อื่นด้วย) — ฟังก์ชัน `validate_percentage` ทำหน้าที่เดียวกับ `Deserialize::deserialize`
ของ `Percentage` เป๊ะ เพียงแต่เป็น free function เพื่อให้ derive macro ของ `Task` เรียกใช้แทน field
นี้เพียงจุดเดียว ส่วน `name: String` derive ตามปกติ

ผลลัพธ์จริง:

```
Task { name: "build", progress: 75.0 }
bad task error: progress must be between 0.0 and 100.0, got 999 at line 1 column 33
```

สังเกตว่า error message ของ `Task` มี `at line 1 column 33` ต่อท้าย (มาจาก `serde_json` ที่ห่อ error
ของเราด้วยตำแหน่งใน input อัตโนมัติ อีกครั้ง) ในขณะที่ error message ของ `Percentage` เดี่ยว ๆ ก่อนหน้า
ก็มีพฤติกรรมเดียวกัน (ตัดออกจากตัวอย่างเพื่อความกระชับ แต่ทำงานเหมือนกันทุกจุดที่ deserialize ผ่าน
`serde_json`) — ไม่ว่าจะเลือกวิธีไหน (newtype เต็มรูปแบบ หรือ `deserialize_with`) **ผลลัพธ์ด้านความ
ปลอดภัยเหมือนกัน: ไม่มีทางที่ค่าที่ผิด invariant จะกลายเป็น Rust value ที่ compile ผ่านและใช้งานได้เลย**
— นี่คือการปิดวงคำถามจาก Part 53 อย่างสมบูรณ์: newtype validation ที่เคย "ปิดประตูหน้า" (constructor)
ตอนนี้ "ปิดประตูหลัง" (deserialize) ได้ด้วยหลักการเดียวกันแล้ว

### 58.7 `#[serde(flatten)]` และ `#[serde(transparent)]`

สองหัวข้อนี้เป็น attribute จาก derive (ไม่ใช่ manual implementation) แต่วางไว้ในบทนี้เพราะทั้งคู่ปรับ
**shape ของข้อมูลบน wire ให้ต่างจาก shape ของ struct ใน Rust** เหมือนกับใจความหลักของบทนี้ เพียงแต่ทำ
ผ่าน attribute สำเร็จรูป (คุ้มค่ากว่าการเขียน manual implementation มาก เมื่อรูปแบบที่ต้องการตรงกับ
สิ่งที่ attribute เหล่านี้ทำได้พอดี)

**`#[serde(flatten)]`** — รวม field ของ struct ที่ซ้อนอยู่ ("nested") เข้ากับ field ของ struct แม่โดยตรง
ในระดับเดียวกัน ไม่มี wrapper object คั่นกลาง สถานการณ์ที่พบบ่อยที่สุด: API ที่มี field พื้นฐานร่วมกัน
(เช่น `id`, `created_at`) ที่อยากแยกเป็น struct กลางเพื่อ reuse ในหลาย response type แต่ JSON ที่ต้อง
ส่งออกจริงต้องการให้ field พวกนี้อยู่ "แบน" (flat) ในระดับเดียวกับ field อื่น ไม่ใช่ nested object:

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct Base {
    id: u64,
    name: String,
}

#[derive(Debug, Serialize, Deserialize)]
struct ExtendedUser {
    #[serde(flatten)]
    base: Base,
    role: String,
}

fn main() {
    let user = ExtendedUser {
        base: Base { id: 1, name: "Alice".to_string() },
        role: "admin".to_string(),
    };
    let json = serde_json::to_string(&user).unwrap();
    println!("flatten json: {json}");

    let back: ExtendedUser = serde_json::from_str(&json).unwrap();
    println!("{back:?}");
}
```

ผลลัพธ์จริง:

```
flatten json: {"id":1,"name":"Alice","role":"admin"}
ExtendedUser { base: Base { id: 1, name: "Alice" }, role: "admin" }
```

สังเกตว่า JSON ที่ได้**ไม่มี** `"base": {"id":1,"name":"Alice"}` ห่ออยู่เลย — `id` และ `name` ถูกยกขึ้น
มาอยู่ระดับเดียวกับ `role` ตรง ๆ (deserialize กลับก็ทำงานย้อนทาง: อ่าน field ที่ระดับบนสุดแล้ว "เติม"
เข้าไปใน `base` ให้ถูก struct) เบื้องหลังกลไกนี้คือ `#[serde(flatten)]` เปลี่ยนวิธีที่ derive macro
generate การอ่าน/เขียน field นั้น จากการอ่าน "หนึ่ง key ที่ตรงชื่อ field" เป็นการอ่าน **ทุก key ที่เหลือ
ในระดับเดียวกัน แล้วส่งต่อให้ `Deserialize` ของ type ข้างในไปตีความเอง** — field ที่ flatten ได้จึง
ต้องเป็น type ที่ deserialize/serialize เป็น **map หรือ struct เท่านั้น** (ไม่ใช่ scalar อย่าง `i32`
หรือ `String`) เราจะเห็น error จริงเมื่อละเมิดกฎนี้ในหัวข้อกับดัก 58.9.4

**`#[serde(transparent)]`** — ทำให้ struct ที่มี **field เดียว** serialize/deserialize เป็นค่าของ field
นั้นตรง ๆ **ไม่มี wrapper ให้เห็นใน JSON เลยแม้แต่นิดเดียว** นี่คือแนวคิดที่ใกล้เคียงกับ
`#[repr(transparent)]` ที่ Part 21 สอนไว้ (บอก compiler ว่า struct ที่มี field เดียวมี memory layout
เหมือนกับ field ข้างในทุกประการ ไม่มี overhead ใด ๆ) เพียงแต่ `#[serde(transparent)]` ทำงานที่ระดับ
**การ serialize** (บอก serde ว่า struct นี้ไม่ควรมี "รูปร่างของตัวเอง" บน wire เลย ควรมองทะลุไปที่ field
ข้างในตรง ๆ):

```rust
#[derive(Debug, Serialize, Deserialize, PartialEq)]
#[serde(transparent)]
struct UserId(u64);

fn main() {
    let id = UserId(42);
    let idjson = serde_json::to_string(&id).unwrap();
    println!("transparent json: {idjson}");
    let idback: UserId = serde_json::from_str(&idjson).unwrap();
    println!("{idback:?}");
    assert_eq!(id, idback);
}
```

ผลลัพธ์จริง:

```
transparent json: 42
UserId(42)
```

`UserId(42)` (tuple struct newtype ตัวเดียว ทวนจาก Part 9 และ Part 53 เรื่อง newtype pattern)
serialize ออกมาเป็น **`42` ตรง ๆ** ไม่มี `{"0": 42}` หรือ `[42]` ห่ออยู่เลย — เหมาะมากกับ newtype ที่
สร้างไว้เพื่อความปลอดภัยด้าน type ในฝั่ง Rust เท่านั้น (เช่น กัน `UserId` สลับกับ `ProductId` ที่ต่างก็
เป็น `u64` — ทวนจาก Part 53) แต่ **ไม่อยากให้ผู้บริโภค API ฝั่งอื่น (ที่อาจเขียนด้วยภาษาอื่น) ต้องรับรู้
ว่ามี wrapper type นี้อยู่เลย** — จาก JSON consumer มองไม่เห็นความแตกต่างระหว่าง `UserId` กับ `u64`
ธรรมดาแม้แต่นิดเดียว ในขณะที่ฝั่ง Rust ยังได้ type safety เต็มรูปแบบ (ส่ง `UserId` ผิดที่ที่ควรเป็น
`ProductId` จะ compile ไม่ผ่าน) — นี่คือตัวอย่างที่ดีของการที่ **abstraction ฝั่ง Rust ไม่จำเป็นต้อง
รั่วไหลออกไปสู่ wire format เสมอไป**

ข้อจำกัดสำคัญของ `#[serde(transparent)]`: ใช้ได้กับ struct ที่มี field เดียวเท่านั้น (newtype หรือ
struct ที่มี field อื่นเป็น `PhantomData`/marker type ที่ไม่กระทบ serialize) — ถ้ามีมากกว่าหนึ่ง field
ที่มีข้อมูลจริง จะไม่รู้ว่าควร "มองทะลุ" ไปที่ field ไหน จึงต้อง compile error ออกมา

### 58.8 Case Study เต็มรูปแบบ: `Money` ที่ serialize เป็น string เสมอ

ทวนจาก Part 3: **ในงานที่เกี่ยวกับเงินไม่ควรใช้ float โดยตรง** เพราะ `f64`/`f32` ไม่สามารถเก็บค่าทศนิยม
บางค่าได้เที่ยงตรง 100% (มาจากข้อจำกัดของ IEEE 754 floating point — `0.1 + 0.2` ไม่เท่ากับ `0.3` เป๊ะ ๆ
ในระดับ bit) Part 3 แก้ปัญหานี้ในตัวอย่างระบบสินค้าคงคลังด้วยการเก็บราคาเป็น **จำนวนเต็ม (สตางค์/cents)**
แทนทศนิยม (บาท/ดอลลาร์) หัวข้อนี้จะเอาหลักการเดียวกันมาสร้าง type `Money` ที่ปิดประตูปัญหานี้ให้สนิท
ยิ่งขึ้นไปอีกขั้น: **ปิดกั้นไม่ให้ JSON ที่มี float ดิบ ๆ กลายเป็น `Money` ได้เลยแม้แต่กรณีเดียว** — ต้อง
เป็น string รูปแบบ `"$19.99"` เท่านั้น ไม่มีทางอื่น

การตัดสินใจออกแบบมีเหตุผลสามชั้นซ้อนกัน (ทั้งสามเหตุผลนี้คือสามสถานการณ์ที่หัวข้อ 58.1 เปิดไว้ พบมารวมกัน
ในตัวอย่างเดียว):

1. **เก็บภายในเป็น `cents: u64`** (จำนวนเต็ม ไม่ใช่ float) — แก้ปัญหาความไม่เที่ยงตรงตั้งแต่ระดับ
   representation ในหน่วยความจำ (สถานการณ์ 2: shape ภายในไม่ตรงกับ shape บน wire — `cents: u64` เป็น
   representation ภายในที่ถูกต้องด้านความแม่นยำ แต่ไม่ใช่รูปแบบที่มนุษย์อ่านง่ายหรือที่ API ต้องการ)
2. **serialize ออกเป็น string `"$X.XX"` เสมอ ไม่ใช่ number** — ป้องกันไม่ให้ผู้บริโภค API (โดยเฉพาะถ้า
   เขียนด้วยภาษาที่ JSON number แปลงเป็น float ทันที เช่น JavaScript) นำค่าไปคำนวณต่อด้วย floating
   point โดยไม่รู้ตัว (สถานการณ์ 2 อีกครั้ง จากอีกทิศทาง: บังคับ representation บน wire ให้ปลอดภัยกว่า
   default ที่ derive จะให้)
3. **reject การ deserialize จาก JSON number ทันที** (ไม่ยอมแปลง `19.99` เป็น `Money` แม้จะดูสมเหตุสมผล
   ก็ตาม) — เพราะการยอมรับ JSON number เข้ามาแม้แค่ทางใดทางหนึ่ง เท่ากับเปิดช่องให้ความไม่เที่ยงตรงของ
   float หลุดเข้ามาในระบบได้อยู่ดี (สถานการณ์ 3: validation ที่ต้องรันระหว่าง deserialize)

```rust
use serde::de::{self, Visitor};
use serde::{Deserialize, Deserializer, Serialize, Serializer};
use std::fmt;

#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
struct Money {
    cents: u64,
}

impl Money {
    fn from_dollars_cents(dollars: u64, cents: u8) -> Self {
        assert!(cents < 100, "cents component must be 0-99");
        Money { cents: dollars * 100 + cents as u64 }
    }

    fn to_dollar_string(&self) -> String {
        format!("${}.{:02}", self.cents / 100, self.cents % 100)
    }

    fn parse_dollar_string(s: &str) -> Result<Self, String> {
        let s = s
            .strip_prefix('$')
            .ok_or_else(|| format!("money string must start with '$', got {:?}", s))?;
        let (whole, frac) = s
            .split_once('.')
            .ok_or_else(|| format!("money string must contain a decimal point, got {:?}", s))?;
        if frac.len() != 2 {
            return Err(format!(
                "expected exactly 2 digits after the decimal point, got {:?}",
                s
            ));
        }
        let whole: u64 = whole
            .parse()
            .map_err(|_| format!("invalid whole-dollar part in {:?}", s))?;
        let frac: u64 = frac
            .parse()
            .map_err(|_| format!("invalid cents part in {:?}", s))?;
        Ok(Money { cents: whole * 100 + frac })
    }
}
```

ส่วน "ตรรกะทางธุรกิจ" ล้วน ๆ ข้างบนนี้**ไม่มีคำว่า serde แม้แต่คำเดียว** — ตั้งใจแยกออกจากกันชัดเจน: การ
แปลง `Money` ↔ `String` (`to_dollar_string`/`parse_dollar_string`) เป็นความรู้เรื่อง "เงินคืออะไร" ส่วน
การเชื่อมมันเข้ากับ serde เป็นอีกชั้นแยกต่างหาก (นี่คือหลักการ separation of concerns เดียวกับที่ Part
21 สอนไว้เรื่องการออกแบบ trait ที่ดี) — ทำให้ `Money::parse_dollar_string` ทดสอบได้ตรง ๆ ด้วย
`assert_eq!` ธรรมดา โดยไม่ต้องพึ่ง `serde_json` เลยด้วยซ้ำ ถ้าต้องการ

ตอนนี้ค่อยเชื่อมกับ serde:

```rust
impl Serialize for Money {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        serializer.serialize_str(&self.to_dollar_string())
    }
}

struct MoneyVisitor;

impl<'de> Visitor<'de> for MoneyVisitor {
    type Value = Money;

    fn expecting(&self, f: &mut fmt::Formatter) -> fmt::Result {
        f.write_str("a money string like \"$19.99\"")
    }

    fn visit_str<E>(self, v: &str) -> Result<Money, E>
    where
        E: de::Error,
    {
        Money::parse_dollar_string(v).map_err(E::custom)
    }
}

impl<'de> Deserialize<'de> for Money {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: Deserializer<'de>,
    {
        deserializer.deserialize_str(MoneyVisitor)
    }
}
```

โครงสร้างนี้ตรงกับ `Color` (58.2-58.3) และ `Percentage` (58.6) เป๊ะทุกจุด — เพราะทั้งสามคือรูปแบบเดียวกัน
ของปัญหา: "ค่านี้ควรมี representation บน wire เป็น string ที่ format เฉพาะตัว ไม่ใช่ shape ตรงตัวจาก
field ภายใน" สังเกตจุดสำคัญที่ทำให้ข้อกำหนดข้อ 3 (reject JSON number) เป็นจริงโดยธรรมชาติ: **เราไม่ได้
override `visit_f64`/`visit_i64`/`visit_u64` เลยแม้แต่ตัวเดียว** ดังนั้น input ที่เป็น JSON number ใด ๆ
จะตกไปที่ default implementation ของ `Visitor` ที่ปฏิเสธทันที — เหมือนกับที่เกิดกับ `Color`/`42` ใน
หัวข้อ 58.3 ทุกประการ **นี่คือ "ปฏิเสธโดยไม่ต้องเขียนโค้ดปฏิเสธเอง" อีกครั้ง — จุดแข็งที่สุดของ Visitor
pattern**

มาดู `Invoice` ที่ใช้ `Money` เป็น field หนึ่ง (field อื่น derive ปกติ ตามหลักการหัวข้อ 58.4):

```rust
#[derive(Debug, Serialize, Deserialize)]
struct Invoice {
    item: String,
    price: Money,
}

fn main() {
    let inv = Invoice {
        item: "Rust course".into(),
        price: Money::from_dollars_cents(19, 99),
    };
    let json = serde_json::to_string(&inv).unwrap();
    println!("{json}");
    let back: Invoice = serde_json::from_str(&json).unwrap();
    println!("{back:?}");
    assert_eq!(inv.price, back.price);

    let raw_float = serde_json::from_str::<Invoice>(r#"{"item":"x","price":19.99}"#);
    println!("raw float rejected: {}", raw_float.unwrap_err());

    for bad in ["19.99", "$19.9", "$19", "$abc.de"] {
        let json = format!(r#"{{"item":"x","price":"{bad}"}}"#);
        let r = serde_json::from_str::<Invoice>(&json);
        println!("bad {bad:?} -> {}", r.unwrap_err());
    }
}
```

ผลลัพธ์จริงจาก terminal — ครบทั้ง round-trip ที่สำเร็จ และทุกกรณี validation ที่ต้อง reject:

```
{"item":"Rust course","price":"$19.99"}
Invoice { item: "Rust course", price: Money { cents: 1999 } }
raw float rejected: invalid type: floating point `19.99`, expected a money string like "$19.99" at line 1 column 25
bad "19.99" -> money string must start with '$', got "19.99" at line 1 column 27
bad "$19.9" -> expected exactly 2 digits after the decimal point, got "19.9" at line 1 column 27
bad "$19" -> money string must contain a decimal point, got "19" at line 1 column 25
bad "$abc.de" -> invalid whole-dollar part in "abc.de" at line 1 column 29
```

วิเคราะห์แต่ละกรณีที่ reject:

- **`raw float rejected`** — ยืนยันข้อกำหนดที่ 3: JSON number `19.99` ถูกปฏิเสธที่ระดับ Visitor ก่อน
  จะถึงตรรกะ parse string ด้วยซ้ำ (error มาจาก default implementation ของ `Visitor::visit_f64` ไม่ใช่
  จาก `parse_dollar_string`) — เห็นชัดว่า "รูปร่างผิด" (wrong shape) กับ "เนื้อหาผิด" (wrong content)
  เป็น error คนละชั้นกัน serde จัดการชั้นแรกให้อัตโนมัติ ส่วนชั้นที่สองเราต้องเขียนเอง
- **`"19.99"` (string แต่ไม่มี `$`)** — ผ่านชั้น "เป็น string" ของ Visitor ไปได้ แต่ตกที่ชั้น "เนื้อหา
  ของ string ถูกต้องไหม" ที่เราเขียนใน `parse_dollar_string` — error message บอกสาเหตุตรงจุดที่สุด
- **`"$19.9"` / `"$19"` / `"$abc.de"`** — ตัวอย่างที่ครอบคลุมทุก failure mode ของการ parse: จำนวนหลัก
  ทศนิยมผิด, ไม่มีจุดทศนิยมเลย, ส่วนจำนวนเต็มไม่ใช่ตัวเลข — สังเกตว่าทุก error message **ระบุเจาะจงว่า
  อะไรผิดตรงไหน** ไม่ใช่ error กำกวมแบบ "invalid money format" เพียวๆ — นี่คือมาตรฐานการออกแบบ error
  message ที่ดีตาม Part 30

case study นี้แสดงให้เห็นภาพรวมของทั้งบทในตัวอย่างเดียว: representation ภายในที่แม่นยำ (`u64` cents),
representation บน wire ที่ปลอดภัยและมนุษย์อ่านง่าย (string `"$X.XX"`), และ validation ที่ปฏิเสธข้อมูล
ผิดตั้งแต่จุดกำเนิดโดยไม่มีทางบายพาสได้เลย — ครบทั้งสามสถานการณ์ที่หัวข้อ 58.1 ตั้งคำถามไว้ตอนต้นบท

## กับดักที่พบบ่อย (Common Pitfalls)

**1. พยายาม `impl Serialize`/`Deserialize` ให้ type จาก external crate ตรง ๆ — ชน orphan rule**

โค้ดที่ผิด (ทั้ง `Serialize` และ `chrono::Duration` เป็นของ crate อื่นทั้งคู่ ไม่ใช่ของเรา):

```rust
use serde::{Serialize, Serializer};

impl Serialize for chrono::Duration {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        serializer.serialize_i64(self.num_seconds())
    }
}
```

compile error จริง:

```
error[E0117]: only traits defined in the current crate can be implemented for types defined outside of the crate
 --> src/main.rs:5:1
  |
5 | impl Serialize for chrono::Duration {
  | ^^^^^^^^^^^^^^^^^^^----------------
  |                    |
  |                    `TimeDelta` is not defined in the current crate
  |
  = note: impl doesn't have any local type before any uncovered type parameters
  = note: for more information see https://doc.rust-lang.org/reference/items/implementations.html#orphan-rules
  = note: define and implement a trait or new type instead
```

(สังเกตว่า compiler เรียก `chrono::Duration` ด้วยชื่อภายในจริงของมันคือ `TimeDelta` — เป็น type alias
ที่ `chrono` เปลี่ยนชื่อในเวอร์ชันใหม่ แต่ orphan rule ยังใช้กับ type ที่แท้จริงอยู่ดี ไม่เกี่ยวกับชื่อ
ที่มองเห็น) **วิธีแก้:** สร้าง newtype ของตัวเอง (ทวนจาก Part 21 และ Part 53) แล้ว implement บน
newtype นั้นแทน:

```rust
struct MyDuration(chrono::Duration); // ของ crate เราเอง — orphan rule อนุญาต

impl Serialize for MyDuration {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        serializer.serialize_i64(self.0.num_seconds())
    }
}
```

**2. `Deserialize<'de>` แบบ zero-copy return ค่าที่ยืมจาก local variable — dangling reference**

โค้ดที่ผิด:

```rust
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct Event<'a> {
    name: &'a str,
}

fn parse_event() -> Event<'static> {
    let input = String::from(r#"{"name":"deploy"}"#);
    let event: Event = serde_json::from_str(&input).unwrap();
    event
}
```

compile error จริง:

```
error[E0515]: cannot return value referencing local variable `input`
  --> src/main.rs:11:5
   |
10 |     let event: Event = serde_json::from_str(&input).unwrap();
   |                                             ------ `input` is borrowed here
11 |     event
   |     ^^^^^ returns a value referencing data owned by the current function
```

**วิธีแก้:** อย่าสร้าง input buffer ไว้ในฟังก์ชันที่ต้อง return ค่าที่ยืมจากมัน — ให้ caller เป็น
เจ้าของ buffer แล้วส่ง `&str` ให้ฟังก์ชัน deserialize ยืมใช้ในระดับเดียวกัน (ดูหัวข้อ 58.5) หรือถ้า
ownership ซับซ้อนเกินไป ให้ deserialize เป็น `String` ปกติแทน (เสีย zero-copy แต่ปลอดภัยและง่ายกว่า)

**3. ลืมว่า Visitor ปฏิเสธ shape ที่ไม่ได้ override ให้อัตโนมัติ — เข้าใจผิดว่าต้อง handle ทุก type เอง**

มือใหม่หลายคนพยายาม override `visit_i64`, `visit_u64`, `visit_f64` ทุกตัวเพื่อ "return error" เอง
ทั้งที่ **ไม่ override เลยก็ได้ผลเหมือนกัน** เพราะ default implementation ของ `Visitor` ปฏิเสธให้อยู่แล้ว
พร้อมข้อความที่อ้างอิง `expecting()` ของเราด้วย ตัวอย่างจริงจากหัวข้อ 58.3:

```
wrong type error: invalid type: integer `42`, expected a hex color string like "#RRGGBB" at line 1 column 2
```

การเขียน `visit_i64` เพิ่มเพื่อ `return Err(...)` เองเป็นโค้ดซ้ำซ้อนที่ไม่จำเป็น (แถมข้อความ error
อาจไม่สอดคล้องกับรูปแบบมาตรฐานของ serde ที่ผู้ใช้ชินอ่านจาก library อื่น ๆ ด้วย) **หลักการที่ควรจำ:**
override เฉพาะ `visit_*` ที่คุณ**ยอมรับ**เท่านั้น ปล่อยให้ default จัดการกรณีที่เหลือทั้งหมด

**4. flatten field ที่ type ไม่ใช่ struct/map — `can only flatten structs and maps`**

โค้ดที่ผิด (พยายาม flatten field ที่เป็น scalar `i32`):

```rust
#[derive(Debug, Serialize)]
struct BadFlatten {
    #[serde(flatten)]
    n: i32,
    other: u8,
}
```

รันจริง (compile ผ่าน แต่ fail ตอน serialize จริง เพราะ serde ตรวจ shape ตอน runtime สำหรับ error นี้):

```
serialize error: can only flatten structs and maps (got an integer)
```

**เหตุผล:** `#[serde(flatten)]` ทำงานโดยการรวม **key-value pair** ของ field ที่ flatten เข้ากับ
key-value pair ของ struct แม่ — scalar อย่าง `i32` ไม่มี key-value pair ให้รวม (มันคือค่าเดี่ยว ไม่ใช่
map) จึง fail **วิธีแก้:** ใช้ flatten กับ field ที่เป็น struct หรือ `HashMap<String, Value>` เท่านั้น
(ดูตัวอย่างที่ถูกต้องในหัวข้อ 58.7 — `Base` เป็น struct ธรรมดา ใช้ flatten ได้)

**5. `serialize_with`/`deserialize_with` ชี้ไปยังฟังก์ชันที่ signature ไม่ตรงกับที่ derive ต้องการ**

โค้ดที่ผิด (ฟังก์ชัน `bad_serialize` รับ `&u32` ตัวเดียว ไม่รับ `Serializer`, และคืน `String` ไม่ใช่
`Result<S::Ok, S::Error>`):

```rust
fn bad_serialize(value: &u32) -> String {
    value.to_string()
}

#[derive(Debug, Serialize, Deserialize)]
struct Wrong {
    #[serde(serialize_with = "bad_serialize")]
    count: u32,
}
```

compile error จริง:

```
error[E0061]: this function takes 1 argument but 2 arguments were supplied
  --> src/main.rs:11:30
   |
 9 | #[derive(Debug, Serialize, Deserialize)]
   |                 --------- unexpected argument #2 of type `__S`
10 | struct Wrong {
11 |     #[serde(serialize_with = "bad_serialize")]
   |                              ^^^^^^^^^^^^^^^

error[E0308]: mismatched types
  --> src/main.rs:11:30
   |
 9 | #[derive(Debug, Serialize, Deserialize)]
   |                 --------- expected `Result<<__S as _::_serde::Serializer>::Ok, <__S as _::_serde::Serializer>::Error>` because of return type
10 | struct Wrong {
11 |     #[serde(serialize_with = "bad_serialize")]
   |                              ^^^^^^^^^^^^^^^ expected `Result<<__S as Serializer>::Ok, ...>`, found `String`
```

สังเกตว่า error ชี้ไปที่บรรทัด attribute (`#[serde(serialize_with = ...)]`) ไม่ใช่บรรทัดที่นิยาม
`bad_serialize` — เพราะ derive macro generate โค้ดที่**เรียก** `bad_serialize` ตรงตำแหน่งนั้น (ทวนจาก
Part 45: span ของโค้ดที่ generate มักผูกกับตำแหน่งที่ผู้ใช้เขียน attribute ไม่ใช่ตำแหน่งภายใน macro)
**วิธีแก้:** ฟังก์ชันที่ใช้กับ `serialize_with` ต้องมี signature ตรงกับ `Serialize::serialize` เป๊ะ —
`fn(&T, S) -> Result<S::Ok, S::Error> where S: Serializer` (ดูตัวอย่างที่ถูกต้องในหัวข้อ 58.4)

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** implement `Serialize` (แบบมือ ไม่ใช้ derive) ให้ `struct Temperature { celsius: f64 }`
   ให้ serialize เป็น string รูปแบบ `"25.5°C"` (ทศนิยม 1 ตำแหน่งเสมอ ต่อด้วย `°C`) แทน
   `{"celsius":25.5}` แบบที่ derive จะให้ (hint: ใช้ `format!("{:.1}°C", self.celsius)` แล้ว
   `serializer.serialize_str(...)` — เหมือนกับ `Color` ในหัวข้อ 58.2 ทุกจุด เพียงแค่เปลี่ยน logic
   การ format)

2. **[กลาง]** implement `Deserialize` ให้ `Temperature` จากข้อ 1 ให้ parse string `"25.5°C"` กลับเป็น
   `Temperature { celsius: 25.5 }` และต้อง reject กรณีที่ไม่มี `°C` ต่อท้าย หรือส่วนตัวเลขก่อนหน้า parse
   เป็น `f64` ไม่ได้ ด้วย error message ที่บอกสาเหตุเจาะจง (hint: เขียน `Visitor` ที่ override
   `visit_str` เหมือน `ColorVisitor`/`MoneyVisitor` — ใช้ `value.strip_suffix("°C")` แทน
   `strip_prefix('#')` แล้ว parse ส่วนที่เหลือด้วย `.parse::<f64>()`) ทดสอบด้วย
   `serde_json::from_str::<Temperature>("\"100C\"")` (ไม่มี `°`) แล้วดูว่า error message อ่านเข้าใจได้
   ไหม

3. **[ยาก]** เขียน `struct SemVer { major: u32, minor: u32, patch: u32 }` ที่ serialize เป็น string
   `"MAJOR.MINOR.PATCH"` (เช่น `"1.4.20"`) และ deserialize กลับได้ถูกต้อง — โดยต้อง reject รูปแบบที่ไม่
   ตรง 3 ส่วนคั่นด้วยจุด (เช่น `"1.4"` หรือ `"1.4.20-beta"` ต้อง error) จากนั้นเพิ่ม field ใหม่ให้
   `SemVer` คือ `pre_release: Option<String>` ที่ทำให้ format กลายเป็น `"1.4.20-beta"` เมื่อมีค่า และ
   `"1.4.20"` เมื่อไม่มี (hint: ต้องแก้ทั้ง `Serialize` และ `Deserialize` ให้ split ด้วย `-` ก่อน
   parse ส่วนตัวเลข สังเกตว่านี่คือ "shape ที่ไม่ตรงกับ struct field" ตามสถานการณ์ที่ 2 ในหัวข้อ 58.1
   อย่างเป๊ะที่สุด — struct มี 4 field แต่ representation บน wire เป็น string เดี่ยว)

4. **[ยาก/ประยุกต์ใช้งานจริง]** ขยาย `Money` จากหัวขัด 58.8 ให้รองรับ **สกุลเงินหลายแบบ**:
   `struct Money { cents: i64, currency: Currency }` โดย `Currency` เป็น
   `enum Currency { Usd, Thb, Eur }` ที่ serialize เป็น string 3 ตัวอักษรมาตรฐาน (`"USD"`, `"THB"`,
   `"EUR"`) ผ่าน `#[derive(Serialize, Deserialize)]` ธรรมดา (ใช้ `#[serde(rename = "...")]` ทวนจาก
   Part 57 ถ้าต้องการชื่อ variant ต่างจากที่ประกาศ) ส่วน `Money` ทั้งก้อนต้อง serialize เป็น string
   รูปแบบ `"$19.99"` (USD), `"฿19.99"` (THB), `"€19.99"` (EUR) — และ `cents` ตอนนี้เป็น `i64` (รองรับ
   ค่าติดลบสำหรับ refund/credit) โดยค่าติดลบต้อง serialize เป็น `"-$19.99"` (เครื่องหมายลบอยู่หน้า
   สัญลักษณ์เงิน) เขียนทั้ง `Serialize`/`Deserialize` ให้ทำงาน round-trip ถูกต้องกับทั้ง 3 สกุลเงิน
   และค่าทั้งบวก/ลบ พร้อมเขียน validation test ที่ยืนยันว่า string ที่มีสัญลักษณ์เงินผิดสกุล (เช่น
   `"€19.99"` ที่ parse เป็น USD) ถูก reject ด้วย error message ที่ชัดเจน (hint: แยกฟังก์ชัน parse
   symbol ก่อน parse ตัวเลข คล้ายกับที่ `Money::parse_dollar_string` แยก `strip_prefix('$')` ออกมา
   ก่อนแล้วค่อย parse ส่วนที่เหลือ — ต่างกันแค่ต้อง map สัญลักษณ์ไปเป็น `Currency` ก่อน)

## สรุป

บทนี้ปิดวง mini-arc เรื่อง Serde ที่ Part 57 เปิดไว้: Part 57 สอนวิธี**ใช้** serde ผ่าน derive และ
attribute มาตรฐานสำหรับ 90% ของสถานการณ์ที่พบในโลกจริง ส่วนบทนี้สอนว่าจะทำอย่างไรกับอีก 10% ที่เหลือ —
เมื่อ derive ไม่พอ เราแยกได้สามสถานการณ์ชัดเจน: type จาก external crate ที่ชนกับ orphan rule (Part 21),
รูปแบบข้อมูลที่ต้องการไม่ตรงกับ shape ของ struct เลย, และ validation ที่ต้องรันระหว่าง deserialize
ไม่ใช่หลังจากนั้น

จากนั้นเราเจาะลึกถึงระดับ trait signature จริง: `Serialize::serialize<S: Serializer>` ที่ใช้ generic
parameter เพื่อให้ implementation เดียวทำงานได้กับทุก wire format (หลักการเดียวกับ format-independence
ของ Part 57 มองจากมุมคนเขียน implementation) และ `Deserialize`/`Visitor` pattern ที่ซับซ้อนกว่าโดย
ธรรมชาติเพราะต้องรับมือกับความไม่รู้ล่วงหน้าว่า input จะมาในรูปแบบไหน — implement ทั้งสอง trait ให้
`Color` แบบเต็มรูปแบบ แล้วเรียนรู้ทางลัดที่ปฏิบัติได้จริงในโปรเจกต์จริง (`#[serde(with = "module")]`,
`serialize_with`, `deserialize_with`) ที่ให้ปรับแค่ field เดียวโดยไม่เสียประโยชน์ของ derive สำหรับ field
ที่เหลือ

เทคนิคเฉพาะทางสองเรื่องที่สำคัญมากในโลก production: **zero-copy deserialization** ผ่าน
`Deserialize<'de>` และ `&'de str` (ประหยัด allocation ได้จริงเมื่อต้อง deserialize ข้อมูลปริมาณมาก
แลกมาด้วย lifetime constraint ที่เข้มงวดขึ้นตามกฎ borrow checker เดิมจาก Part 20/23) และ **custom
validation ระหว่าง deserialize** ที่เอา pattern "newtype + validation" จาก Part 53 มาผสานกับ serde
โดยตรง ทำให้ค่าที่ผิด invariant ไม่มีทางกลายเป็น Rust value ที่ compile ผ่านได้เลยไม่ว่าทางไหน — ปิด
ท้ายด้วย `#[serde(flatten)]`/`#[serde(transparent)]` (attribute สำเร็จรูปสำหรับกรณีที่ตรงกับ pattern
มาตรฐาน) และ case study ขนาดใหญ่ `Money` ที่รวมทั้งสามสถานการณ์เข้าด้วยกันในตัวอย่างเดียว: representation
ภายในที่แม่นยำแบบ integer cents (ทวนจาก Part 3), representation บน wire ที่ปลอดภัยแบบ string, และ
validation ที่ reject float ดิบและ string ผิดรูปแบบทุกกรณีตั้งแต่จุดกำเนิด

**Part 59: CLI Applications ด้วย clap** จะพา derive macro pattern ที่เราเห็นตลอดสอง Part นี้ (helper
attribute, generic ที่ผูกกับ format ต่าง ๆ, validation ที่ integrate เข้ากับ derive) ไปใช้ในบริบทใหม่:
`#[derive(Parser)]` ของ `clap` ที่แปลง struct ธรรมดาเป็น command-line argument parser แบบเต็มรูปแบบ
— คุณจะเห็นว่า `#[arg(...)]` ของ `clap` ทำงานด้วยกลไกเดียวกับ `#[serde(...)]` ทุกประการ (ทั้งคู่เป็น
helper attribute ที่ derive macro parse ผ่าน `syn::Attribute::parse_nested_meta` ตามที่ Part 45 สอนไว้)
เพียงแต่เปลี่ยนจาก "แปลงข้อมูลระหว่าง Rust กับ wire format" เป็น "แปลงข้อมูลระหว่าง Rust กับ
command-line arguments ที่ผู้ใช้พิมพ์ในเทอร์มินัล" — เป็นอีกตัวอย่างที่ตอกย้ำว่าทำไม Part 44-45 ถึงคุ้มค่า
ที่จะเรียนให้ลึก: มันคือกลไกเบื้องหลัง derive macro ระดับ production ทุกตัวที่คุณจะเจอต่อจากนี้ในสาย
Rust ทั้งหมด

---

**Part ก่อนหน้า:** [Serialization: Serde เบื้องต้น](part-057-serde-basics.md) | **Part ถัดไป:**
[CLI Applications ด้วย clap](part-059-clap-cli.md)
