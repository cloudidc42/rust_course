# Part 30: Error Handling ขั้นสูง (custom error types, From/Into, error chains)

> โมดูล: ระดับกลาง (Intermediate) | ระดับ: กลาง | เวลาโดยประมาณ: 200 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายข้อจำกัดของ `Box<dyn Error>` (ทางลัด pragmatic ที่เรียนไปแล้วใน Part 12) ได้อย่างชัดเจนว่ามันเสียข้อมูล
  ชนิดที่แน่นอนของ error ไปตรงไหน และทำไมข้อจำกัดนี้ถึงสำคัญมากสำหรับโค้ดที่เป็น **library** มากกว่าโค้ดที่เป็น
  script/application ทั่วไป
- สร้าง **custom error enum** ของตัวเองตั้งแต่ต้น พร้อม implement `std::fmt::Display` (สำหรับข้อความที่มนุษย์อ่านได้)
  และ `std::error::Error` (marker trait ที่ทำให้ error type ของคุณ "เข้ากันได้" กับระบบ error handling ทั้งหมดของ
  Rust) ได้อย่างถูกต้องตามหลักการ
- อธิบายบทบาทของ trait `From<T>` ที่ทำให้ `?` operator "แปลงชนิด error ข้ามกันได้อัตโนมัติ" อย่างเป็นทางการและ
  ละเอียดกว่าที่ Part 12 แนะนำไว้ พร้อมอ่าน error `E0277` ที่เกิดขึ้นเมื่อไม่มี `impl From` ที่จำเป็นได้อย่างแม่นยำ
- implement **error chain** ผ่าน method `source()` เพื่อให้ผู้เรียกสามารถ "ไล่" สาเหตุที่แท้จริงของความล้มเหลวได้
  ทีละชั้น จากชั้นบนสุด (เช่น "โหลด config ไม่สำเร็จ") ลงไปถึงชั้นล่างสุด (เช่น "ตัวเลขที่ parse ไม่ถูกต้อง")
- ออกแบบ error type ที่ "ห่อ" (wrap) error หลายชนิดจากหลายแหล่งไว้ใน enum เดียว (เช่น ทั้ง I/O error และ parse
  error) พร้อม `impl From` ให้แต่ละชนิด เพื่อให้ `?` ใช้งานได้อย่างอิสระในทุกจุดที่เรียกใช้
- ใช้ `#[non_exhaustive]` บน public error enum ได้อย่างเข้าใจเหตุผลเชิง API design เบื้องหลัง และรู้ว่ามันเปลี่ยน
  พฤติกรรมของ `match` ที่ฝั่งผู้เรียก (จากคนละ crate) อย่างไรบ้างจริง ๆ
- เขียน `Display` ที่ **ดี** สำหรับ error type ของตัวเอง — ข้อความที่บอกทั้ง "เกิดอะไรขึ้น" และ "ควรทำอย่างไรต่อ" —
  และอธิบายได้ว่าทำไม error message คือส่วนหนึ่งของ API surface ของโค้ด ไม่ใช่แค่รายละเอียดภายในที่มองข้ามได้

## ความรู้ที่ต้องมีมาก่อน

- **Part 12 (Result<T,E> และ Error Handling เบื้องต้น)**: บทนี้คือภาคต่อโดยตรงของ Part 12 และอ้างอิงกลับไปที่นั่น
  ตลอดทั้งบท คุณต้องคุ้นเคยกับสิ่งเหล่านี้มาก่อนแล้ว: `Result<T, E>` ในฐานะ enum ธรรมดา, `?` operator และการที่มัน
  desugar เป็น `match { Ok(v) => v, Err(e) => return Err(From::from(e)) }`, บทบาทเบื้องต้นของ `From` ที่ `?` ใช้
  แปลง error, และ `Box<dyn std::error::Error>` ในฐานะทางลัด pragmatic ที่ Part 12 หัวข้อ 12.7-12.8 แนะนำไว้เป็น
  ค่า default สำหรับ script/prototype/`main()` — บทนี้จะ **ไม่สอนพื้นฐานพวกนี้ซ้ำ** แต่จะพาไปลึกกว่าเดิมมาก โดย
  เฉพาะจุดที่ Part 12 บอกไว้ตรง ๆ ว่า "จะเจาะลึกใน Part 30": การออกแบบ custom error type แบบเต็มรูปแบบด้วยมือ
  ก่อนที่จะเห็น `thiserror`/`anyhow` ลดโค้ดให้สั้นลงใน Part 31
- **Part 19 (Traits เบื้องต้น)**: `std::error::Error` เป็น trait ธรรมดาที่คุณ `impl` เองได้ เหมือนกับ trait ใด ๆ
  ที่เรียนมาจาก Part 19 — แนวคิดเรื่อง **trait object** (`dyn Trait`) ที่ Part 19 แนะนำไว้ (และ Part 12 ใช้ผ่าน
  `Box<dyn Error>`) จะกลับมาอีกหลายครั้งในบทนี้ ทั้งตอนอธิบาย `source()` ที่คืน `Option<&(dyn Error + 'static)>`
  และตอนเขียนฟังก์ชันที่รับ `&dyn Error` เพื่อไล่ error chain
- **Part 21 (Traits ขั้นสูง)**: บทนี้ใช้แนวคิด **supertrait** ที่ Part 21 สอนไว้โดยตรง — นิยามจริงของ
  `std::error::Error` คือ `pub trait Error: Debug + Display` ซึ่งแปลว่า `Debug` และ `Display` เป็น **supertrait**
  ของ `Error` (ทุก type ที่จะ `impl Error` ต้อง `impl Debug` และ `impl Display` ให้ครบก่อนเสมอ) — ถ้าคุณยังไม่แน่น
  เรื่อง supertrait ควรทวน Part 21 ก่อน เพราะกับดักที่พบบ่อยที่สุดข้อหนึ่งในบทนี้มาจากการลืมกฎนี้ตรง ๆ
- **Part 18/22 (Generics เบื้องต้น/ขั้นสูง)**: `impl From<T> for MyError` คือการ implement trait ที่มี generic
  type parameter (`From<T>`) ให้กับ type ของเราเอง — ถ้าคุณคุ้นเคยกับ syntax `impl<...> Trait<...> for Type`
  จาก Part 18/22 มาแล้ว จะอ่านโค้ดในบทนี้ได้ลื่นกว่ามาก

## เนื้อหา

### 30.1 ทวนความจำ: `Box<dyn Error>` แก้ปัญหาได้ดีแค่ไหน และพังตรงไหน

จาก Part 12 คุณรู้จัก `Box<dyn std::error::Error>` มาแล้วในฐานะ "ทางลัดที่ pragmatic" — มันแก้ปัญหา error type
ไม่ตรงกันระหว่างฟังก์ชันย่อยหลายตัวได้อย่างรวดเร็ว โดยไม่ต้องเขียน `impl From` เองทีละคู่ เพราะ standard library
เขียน `impl<E: Error> From<E> for Box<dyn Error>` ไว้ให้ล่วงหน้าแล้ว ครอบคลุมทุก error type ที่ implement `Error`
ไว้ทีเดียว นี่คือเหตุผลที่ `?` ใช้งานได้ตรง ๆ กับ `Box<dyn Error>` แทบทุกสถานการณ์

แต่ลองมาดูสิ่งที่เกิดขึ้นจริงเมื่อ **ผู้เรียก** ฟังก์ชันที่คืน `Box<dyn Error>` พยายามจะ "ตอบสนองต่าง ๆ กันไปตาม
สาเหตุของความล้มเหลว" — ซึ่งเป็นสิ่งที่โปรแกรมจริงต้องทำบ่อยมาก (เช่น "ถ้าหาผู้ใช้ไม่เจอ ให้สร้างใหม่อัตโนมัติ แต่
ถ้าเป็น error อื่นให้แจ้งเตือนแล้วหยุด"):

```rust
use std::error::Error;
use std::fmt;

#[derive(Debug)]
struct NotFoundError;

impl fmt::Display for NotFoundError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "ไม่พบผู้ใช้ที่ระบุ")
    }
}
impl Error for NotFoundError {}

fn look_up(id: u32) -> Result<String, Box<dyn Error>> {
    if id == 0 {
        return Err(Box::new(NotFoundError));
    }
    Ok(format!("user-{id}"))
}

fn main() {
    match look_up(0) {
        Ok(name) => println!("found: {name}"),
        Err(e) => {
            // ทำได้แค่พิมพ์ ไม่รู้ว่าเป็น error ชนิดไหนกันแน่ ถ้าไม่ downcast
            println!("error: {e}");

            // downcast_ref ทำได้ แต่ต้องรู้ชนิดที่ "เดา" ไว้ล่วงหน้า และยังต้องเขียน if/else ไล่ทีละชนิด
            if e.downcast_ref::<NotFoundError>().is_some() {
                println!("-> ลองสร้าง user ใหม่ให้อัตโนมัติ");
            } else {
                println!("-> error ประเภทอื่นที่ไม่คาดคิด: {e}");
            }
        }
    }
}
```

ผลลัพธ์:

```
error: ไม่พบผู้ใช้ที่ระบุ
-> ลองสร้าง user ใหม่ให้อัตโนมัติ
```

**สิ่งที่น่าสังเกตอย่างยิ่งจากโค้ดนี้คือ `if e.downcast_ref::<NotFoundError>().is_some()` ทำงานได้จริง** — `dyn
Error` มี method `.downcast_ref::<T>()` ให้ (ผ่านกลไกภายในที่คล้าย `Any`) ที่พยายาม "แปลงกลับ" จาก trait object
ไปเป็น concrete type ที่คุณเดาไว้ ถ้าเดาถูกก็ได้ `Some(&T)` ถ้าเดาผิดก็ได้ `None` — **แต่นี่คือสิ่งที่ยืนยันปัญหา
มากกว่าจะแก้มัน**:

1. **คุณต้อง "เดา" ชนิดที่แน่นอนไว้ล่วงหน้าทุกครั้ง** — ผู้เรียกฟังก์ชัน `look_up` ต้องรู้ (จากการอ่าน source code
   หรือ documentation) ว่า error ที่อาจเกิดขึ้นได้คือ `NotFoundError` เท่านั้น ซึ่งเป็นข้อมูลที่ **หายไปจาก
   signature ของฟังก์ชันโดยสิ้นเชิง** — `Result<String, Box<dyn Error>>` บอกแค่ว่า "อาจล้มเหลวด้วย error อะไร
   ก็ได้ที่ implement `Error`" ไม่ได้บอกว่า error ที่เป็นไปได้จริง ๆ มีกี่ชนิด อะไรบ้าง เหมือนกับปัญหาที่ Part 12
   หัวข้อ 12.1 อธิบายไว้เรื่อง exception ในภาษาอื่นที่ "มองไม่เห็นจาก signature" — `Box<dyn Error>` แก้ปัญหานั้น
   ได้แค่ครึ่งเดียว: มันบอกว่า **"อาจล้มเหลวได้"** (ต่างจาก exception ที่ไม่บอกอะไรเลย) แต่ไม่บอกว่า **"ล้มเหลว
   ได้ด้วยสาเหตุอะไรบ้าง"** ซึ่งเป็นข้อมูลที่ผู้เรียกต้องมีถ้าจะตอบสนองแยกกรณีได้จริง
2. **ไม่มี exhaustiveness checking ใด ๆ เลย** — ถ้าอนาคตฟังก์ชัน `look_up` เพิ่มความล้มเหลวแบบใหม่ (เช่น
   `PermissionDeniedError`) compiler **ไม่มีทางเตือนโค้ดที่เรียก `.downcast_ref::<NotFoundError>()`** ว่า "ยังมี
   กรณีอื่นที่ยังไม่ได้จัดการ" เพราะจากมุมมองของ type system ทั้งหมดยังเป็น `Box<dyn Error>` เหมือนเดิมทุก
   ประการ — ต่างจาก `match` บน enum ที่คุณคุ้นเคยมาตั้งแต่ Part 10 ที่ compiler จะฟ้อง `E0004` ทันทีถ้า enum
   เพิ่ม variant ใหม่แล้วคุณลืม handle
3. **`if/else` ไล่ downcast ทีละชนิดอ่านยากขึ้นเรื่อย ๆ เมื่อจำนวน error type ที่เป็นไปได้เพิ่มขึ้น** — เทียบกับ
   `match` บน enum ที่อ่านเป็นรายการ case ชัดเจนตั้งแต่แรก โค้ดที่ไล่ `downcast_ref` หลาย ๆ ชนิดจะกลายเป็น
   `if let Some(_) = e.downcast_ref::<A>() { ... } else if let Some(_) = e.downcast_ref::<B>() { ... } else if
   ...` ที่ยาวและเสี่ยงพลาดมากกว่า `match` แบบ exhaustive มาก

จุดนี้คือหัวใจของบททั้งบทนี้: **`Box<dyn Error>` เหมาะกับสถานการณ์ที่ผู้เรียกไม่จำเป็นต้องแยกกรณี error (แค่
log/แสดงข้อความ/หยุดโปรแกรม) แต่ถ้าผู้เรียกต้องตัดสินใจแตกต่างกันไปตามสาเหตุของความล้มเหลว — ซึ่งเป็นเรื่องปกติมาก
สำหรับโค้ดที่เป็น library ที่คนอื่นจะเอาไปใช้ต่อ — คุณต้องคืน enum ที่ compiler รู้จักโครงสร้างเต็มรูปแบบ ไม่ใช่
trait object ที่ซ่อนโครงสร้างไว้** ตารางเปรียบเทียบสั้น ๆ ก่อนเข้าเนื้อหาหลัก:

| คุณสมบัติ | `Box<dyn Error>` | Custom error enum |
|---|---|---|
| เขียนเร็วแค่ไหน | เร็วมาก แทบไม่ต้องเขียนอะไรเพิ่ม | ต้องเขียน `Display`/`Error`/`From` เอง (ก่อนเรียน Part 31) |
| ผู้เรียก `match` แยกกรณีได้ไหม | ไม่ได้ตรง ๆ (ต้อง downcast ซึ่งไม่ idiomatic) | ได้เต็มรูปแบบ พร้อม exhaustiveness checking |
| compiler เตือนเมื่อเพิ่มกรณี error ใหม่ไหม | ไม่เตือนผู้เรียกเลย | เตือนทันที (E0004) ถ้าไม่ใส่ `#[non_exhaustive]` |
| เหมาะกับ | script, prototype, `main()`, application ระดับบน | library, ระบบที่ผู้เรียกต้องตัดสินใจตามสาเหตุ error |

บทนี้ทั้งบทคือการเรียนรู้ว่า **"custom error enum" ในคอลัมน์ขวาของตารางนี้สร้างขึ้นมาอย่างไรให้ถูกต้องตามหลักการ
ของ Rust ทุกประการ** — ตั้งแต่ trait ที่ต้อง implement ไปจนถึงกลไกที่ทำให้ `?` ยังใช้งานได้สะดวกเหมือนเดิม

### 30.2 นิยามที่แท้จริงของ `std::error::Error`: trait ธรรมดาที่มี supertrait

Part 12 แนะนำ `std::error::Error` ไว้แบบสั้น ๆ ว่า "เป็น trait ที่ error type ควร implement" — ตอนนี้เราจะดู
นิยามของมันแบบเต็ม (ฉบับย่อเพื่อการสอน ตัวจริงใน std มี method ที่ deprecated แล้วปนอยู่บ้างซึ่งไม่จำเป็นต้องรู้):

```
pub trait Error: std::fmt::Debug + std::fmt::Display {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        None
    }
}
```

(นี่ไม่ใช่โค้ดที่ต้องพิมพ์เอง — มันคือนิยามที่มีอยู่แล้วใน `std::error` ยกมาแสดงเพื่อให้เห็นโครงสร้างจริงเท่านั้น
เหมือนกับที่ Part 12 ยกนิยามของ `Result<T, E>` มาแสดง)

สามจุดสำคัญที่ต้องเข้าใจให้แม่นจากนิยามนี้:

1. **`Error: Debug + Display`** — นี่คือ **supertrait** ตามที่ Part 21 สอนไว้ แปลว่า **type ใดก็ตามที่จะ
   `impl Error` ต้อง `impl Debug` (มักใช้ `#[derive(Debug)]` ก็พอ) และ `impl Display` (ต้องเขียนเอง เพราะ
   `Display` ไม่มี `#[derive]` ให้ — จะอธิบายเหตุผลในหัวข้อ 30.8) ให้ครบทั้งสองก่อนเสมอ** ถ้าขาดตัวใดตัวหนึ่งไป
   compiler จะปฏิเสธทันทีตั้งแต่บรรทัด `impl Error for ...` (จะเห็น error จริงในหัวข้อ "กับดักที่พบบ่อย")
   เหตุผลที่ Rust เลือกบังคับทั้งสอง trait นี้มีเหตุผลเชิง design ที่ชัดเจน: **`Debug` ใช้สำหรับนักพัฒนา** (แสดง
   โครงสร้างภายในดิบ ๆ เพื่อ debug, เหมือนที่เห็นใน panic message ของ `.unwrap()` จาก Part 12) ส่วน **`Display`
   ใช้สำหรับผู้ใช้ปลายทาง/log ที่มนุษย์อ่าน** (ข้อความที่จัดรูปแบบให้อ่านง่าย) — error type ที่ดีต้องรองรับทั้งสอง
   กรณีการใช้งาน ไม่ใช่แค่กรณีเดียว
2. **`fn source(&self) -> Option<&(dyn Error + 'static)>`** — นี่คือ method ที่มี **default implementation**
   (คืน `None` เสมอ) ตามหลักการ default method ที่ Part 21 สอนไว้ — แปลว่า **คุณไม่จำเป็นต้อง override มันเลยก็
   `impl Error for MyType {}` (แบบไม่มี body) ได้ตรง ๆ** เหมือนที่ Part 12 หัวข้อ 12.10 ทำกับ `ConfigParseError`
   — แต่ถ้า error ของคุณ "ห่อ" error อื่นไว้เป็นข้อมูลภายใน (เช่น เกิดจากการ parse ล้มเหลว) การ override
   `source()` ให้คืน `Some(&underlying_error)` จะเปิดทางให้ผู้เรียก **ไล่สาเหตุที่แท้จริงทั้ง chain** ได้ ซึ่งเป็น
   หัวข้อหลักของ 30.5
3. **`&(dyn Error + 'static)`** — สังเกต syntax `dyn Error + 'static` ที่มี **lifetime bound** แนบอยู่กับ trait
   object ตรง ๆ (concept ที่ Part 23 เรื่อง lifetime ขั้นสูงอาจพูดถึง แต่ในที่นี้แค่ต้องรู้ว่ามันหมายความว่า
   "error ที่ถูกห่อไว้ต้องไม่มี reference ที่ยืมมาจากที่อื่นอีกทีอยู่ภายใน — มันเป็นเจ้าของข้อมูลตัวเองทั้งหมด"
   ซึ่ง error type ส่วนใหญ่ที่คุณเขียนหรือเจอจาก standard library (`ParseIntError`, `std::io::Error`) เป็นแบบนี้
   อยู่แล้วโดยธรรมชาติ) — ถ้าเขียน signature ของ `source()` ผิดจาก `'static` ที่กำหนดไว้ compiler จะฟ้อง error
   ที่ดูสับสนพอสมควรถ้าไม่รู้จุดนี้มาก่อน (จะเห็นตัวอย่างจริงในหัวข้อกับดักที่พบบ่อย)

จุดที่ควรทวนให้แน่นก่อนไปต่อ: **`Error` ไม่ได้ทำอะไร "วิเศษ" เป็นพิเศษเลย มันคือ trait ธรรมดาที่คุณ `impl` เองได้
เหมือน trait ใด ๆ ที่เรียนมาจาก Part 19/21** สิ่งที่ทำให้มันมีประโยชน์คือ **standard library และ ecosystem ทั้งหมด
"เห็นพ้อง" กันว่าจะใช้ trait ตัวนี้เป็นจุดร่วมสำหรับ error type ทุกชนิด** — ฟังก์ชันที่รับ `Box<dyn Error>`,
`impl From<E> for Box<dyn Error>` ที่ standard library เขียนไว้ให้, และ crate อย่าง `thiserror`/`anyhow` (Part 31)
ล้วนพึ่งพา trait ตัวนี้เป็นแกนกลางทั้งหมด

### 30.3 สร้าง Custom Error Enum ตั้งแต่ต้น: `ConfigError`

มาสร้าง error enum จริงตั้งแต่ต้น สำหรับโดเมนที่เราจะใช้ตลอดทั้งบทนี้: **ระบบโหลดไฟล์ config อย่างง่าย** ที่อาจ
ล้มเหลวได้หลายสาเหตุ — เริ่มจากเวอร์ชันพื้นฐานที่สุดก่อน:

```rust
use std::error::Error;
use std::fmt;
use std::path::PathBuf;

#[derive(Debug)]
enum ConfigError {
    MissingField(String),
    InvalidNumber(String),
    FileNotFound(PathBuf),
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ConfigError::MissingField(field) => {
                write!(f, "ไม่พบฟิลด์ที่จำเป็น '{field}' ในไฟล์ config")
            }
            ConfigError::InvalidNumber(field) => {
                write!(f, "ฟิลด์ '{field}' ต้องเป็นตัวเลข แต่ค่าที่ให้มาไม่ใช่ตัวเลขที่ถูกต้อง")
            }
            ConfigError::FileNotFound(path) => {
                write!(f, "ไม่พบไฟล์ config ที่ path: {}", path.display())
            }
        }
    }
}

impl Error for ConfigError {}

fn main() {
    let errors = vec![
        ConfigError::MissingField("port".to_string()),
        ConfigError::InvalidNumber("timeout".to_string()),
        ConfigError::FileNotFound(PathBuf::from("/etc/myapp/config.toml")),
    ];

    for e in &errors {
        println!("{e}");
    }

    // exhaustiveness checking ทำงานเต็มรูปแบบ เพราะ ConfigError เป็น enum ธรรมดาที่ compiler รู้จักครบ
    for e in &errors {
        match e {
            ConfigError::MissingField(_) => println!("-> ต้องแจ้งผู้ใช้ให้เติมฟิลด์"),
            ConfigError::InvalidNumber(_) => println!("-> ต้องแจ้งผู้ใช้ให้แก้ตัวเลข"),
            ConfigError::FileNotFound(_) => println!("-> ต้องสร้างไฟล์ default ให้"),
        }
    }
}
```

ผลลัพธ์:

```
ไม่พบฟิลด์ที่จำเป็น 'port' ในไฟล์ config
ฟิลด์ 'timeout' ต้องเป็นตัวเลข แต่ค่าที่ให้มาไม่ใช่ตัวเลขที่ถูกต้อง
ไม่พบไฟล์ config ที่ path: /etc/myapp/config.toml
-> ต้องแจ้งผู้ใช้ให้เติมฟิลด์
-> ต้องแจ้งผู้ใช้ให้แก้ตัวเลข
-> ต้องสร้างไฟล์ default ให้
```

**อธิบายโค้ดทีละส่วน:**

- `#[derive(Debug)] enum ConfigError { ... }` — เรากำหนดให้ error ของเรามี **สามสาเหตุที่แยกจากกันชัดเจน** โดยแต่
  ละ variant เก็บข้อมูลที่จำเป็นสำหรับอธิบายสาเหตุนั้นไว้ (ชื่อฟิลด์ที่ผิด, path ที่หาไม่เจอ) นี่คือหลักการเดียว
  กับที่ Part 12 หัวข้อ 12.3 อธิบายไว้เรื่อง `ParseIntError { kind: InvalidDigit }` ของ standard library เอง:
  **error type ที่ดีคือ structured data ที่ผู้เรียกตรวจสอบได้ ไม่ใช่ string เดียวที่บอกทุกอย่างปนกัน**
  `#[derive(Debug)]` ให้เราได้ implement ครึ่งหนึ่งของ supertrait requirement (`Debug`) แบบอัตโนมัติ
- `impl fmt::Display for ConfigError` — implement supertrait อีกตัว (`Display`) ด้วยมือ เพราะ `Display` ไม่มี
  `#[derive]` ให้ (เหตุผลเชิงลึกอยู่ในหัวข้อ 30.8) — สังเกตว่าแต่ละ variant มีข้อความที่ **อธิบายสาเหตุต่างกัน
  อย่างเจาะจง** ไม่ใช่ข้อความกลาง ๆ แบบเดียวกันหมด
- `impl Error for ConfigError {}` — บรรทัดนี้ดู "ว่างเปล่า" แต่มีความหมายมาก: มันคือการ **ประกาศให้ compiler รู้
  ว่า `ConfigError` พร้อมใช้งานร่วมกับระบบ error handling ทั้งหมดของ Rust แล้ว** (ใช้กับ `Box<dyn Error>`,
  `?`, และ ecosystem ทั้งหมดที่ยึด trait นี้เป็นจุดร่วม) เพราะ `source()` มี default implementation (คืน `None`)
  เราจึงไม่ต้องเขียนอะไรเพิ่มถ้ายังไม่มี error ที่ถูกห่ออยู่ภายใน (จะเห็นตัวอย่างที่ override `source()` จริง
  ในหัวข้อ 30.5)
- `match e { ... }` ที่ท้ายโค้ด — นี่คือสิ่งที่ `Box<dyn Error>` **ทำไม่ได้เลย** ตามที่อธิบายไว้ในหัวข้อ 30.1:
  compiler รู้จักโครงสร้างของ `ConfigError` ครบทุก variant จึงบังคับ exhaustiveness checking ได้เต็มรูปแบบ — ถ้า
  เราเพิ่ม variant ที่ 4 เข้าไปในอนาคตแล้วลืมแก้ `match` นี้ compiler จะฟ้อง `E0004` ทันที (เหมือนที่ Part 10 และ
  Part 12 อธิบายไว้) นี่คือประโยชน์ที่จับต้องได้ของการลงทุนเขียน custom enum แทน `Box<dyn Error>`

### 30.4 `From<T>` และ `Into<T>` ทำให้ `?` "ทำงานข้าม error type" ได้อย่างไร — ฉบับเป็นทางการ

Part 12 หัวข้อ 12.7 แนะนำไว้สั้น ๆ ว่า `?` desugar เป็น `return Err(From::from(error))` และแก้ปัญหา error type
ไม่ตรงกันได้ด้วยการเขียน `impl From` เอง — ตอนนี้เรามาทำความเข้าใจกลไกนี้ให้เป็นทางการและครบถ้วนที่สุด

**กฎที่แม่นยำของ `?` (ทวนจาก Part 12 แต่เขียนให้ชัดเจนกว่าเดิม):** เมื่อ compiler เจอ `expr?` ในฟังก์ชันที่คืน
`Result<T, E>`, มันจะแปลงเป็นโค้ดที่เทียบเท่ากับ:

```
match expr {
    Ok(value) => value,
    Err(error) => return Err(<E as From<ErrorTypeOfExpr>>::from(error)),
}
```

พูดให้ตรงที่สุด: **compiler เรียก `E::from(error)` เสมอ ไม่ว่า `ErrorTypeOfExpr` จะเหมือนกับ `E` หรือไม่ก็ตาม**
(ในกรณีที่เหมือนกัน มันใช้ `impl<T> From<T> for T` ที่ standard library ให้ทุก type โดย default — เรียกว่า
"identity conversion" ที่ Part 12 หัวข้อ 12.6 อธิบายไว้แล้ว) สิ่งที่ทำให้ `?` "แก้ปัญหา error type ไม่ตรงกัน" ได้
คือ **compiler ต้องหา `impl From<ErrorTypeOfExpr> for E` ที่มีอยู่จริงให้ได้เสมอ** — ถ้าหาไม่ได้ compile ไม่ผ่าน

ลองดูตัวอย่างที่ **ไม่มี** `impl From` ที่จำเป็นให้เห็น error เต็มรูปแบบก่อน (ฉบับที่ครอบคลุมกว่าที่ Part 12 แสดง
ไว้สั้น ๆ):

```rust
use std::error::Error;
use std::fmt;
use std::path::PathBuf;

#[derive(Debug)]
enum ConfigError {
    MissingField(String),
    InvalidNumber(String),
    FileNotFound(PathBuf),
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ConfigError::MissingField(field) => write!(f, "ไม่พบฟิลด์ '{field}'"),
            ConfigError::InvalidNumber(field) => write!(f, "ฟิลด์ '{field}' ไม่ใช่ตัวเลข"),
            ConfigError::FileNotFound(path) => write!(f, "ไม่พบไฟล์: {}", path.display()),
        }
    }
}

impl Error for ConfigError {}

fn parse_timeout(raw: &str) -> Result<u32, ConfigError> {
    let timeout = raw.parse::<u32>()?; // raw.parse() คืน Result<u32, ParseIntError>
    Ok(timeout)
}

fn main() {
    println!("{:?}", parse_timeout("30"));
}
```

โค้ดนี้ **compile ไม่ผ่าน** และนี่คือ error message จริงที่ได้ (ทดสอบด้วย `rustc --edition 2021`):

```
error[E0277]: `?` couldn't convert the error to `ConfigError`
  --> src/main.rs:25:37
   |
24 | fn parse_timeout(raw: &str) -> Result<u32, ConfigError> {
   |                                ------------------------ expected `ConfigError` because of this
25 |     let timeout = raw.parse::<u32>()?;
   |                       --------------^ the trait `From<ParseIntError>` is not implemented for `ConfigError`
   |                       |
   |                       this can't be annotated with `?` because it has type `Result<_, ParseIntError>`
   |
note: `ConfigError` needs to implement `From<ParseIntError>`
  --> src/main.rs:6:1
   |
 6 | enum ConfigError {
   | ^^^^^^^^^^^^^^^^
   = note: the question mark operation (`?`) implicitly performs a conversion on the error value using the `From` trait

error: aborting due to 1 previous error

For more information about this error, try `rustc --explain E0277`.
```

**อ่าน error message นี้อย่างละเอียดทีละบรรทัด** เพราะมันบอกกลไกภายในของ `?` ตรง ๆ:

- `` `?` couldn't convert the error to `ConfigError` `` — บอกตรง ๆ ว่า `?` "พยายามแปลง" error แล้วไม่สำเร็จ
  (ไม่ใช่ปัญหาเรื่อง type ทั่วไป แต่เป็นปัญหาเรื่อง **การแปลง** โดยเฉพาะ)
- `` this can't be annotated with `?` because it has type `Result<_, ParseIntError>` `` — บอกชนิดของ error ฝั่ง
  ต้นทาง (`ParseIntError` จาก `.parse::<u32>()`) ให้เห็นตรง ๆ
- `` the trait `From<ParseIntError>` is not implemented for `ConfigError` `` — **นี่คือประโยคที่สำคัญที่สุด**
  บอกชัดเจนว่าปัญหาคือ "ไม่มี `impl From<ParseIntError> for ConfigError`" ซึ่งตรงกับกฎที่เราอธิบายไว้ข้างบนเป๊ะ ๆ
- `` note: `ConfigError` needs to implement `From<ParseIntError>` `` — compiler บอกวิธีแก้ตรง ๆ โดยไม่ต้องเดา
- `` the question mark operation (`?`) implicitly performs a conversion on the error value using the `From`
  trait `` — บรรทัดสุดท้ายนี้คือ compiler ยืนยันกลไก desugar ที่เราอธิบายไว้ตอนต้นหัวข้อด้วยคำพูดของตัวเอง
  ตรงตัวเลย: **"`?` แปลง error โดยปริยายผ่าน `From` trait"**

**วิธีแก้: implement `From<ParseIntError> for ConfigError`**

```rust
use std::error::Error;
use std::fmt;
use std::num::ParseIntError;
use std::path::PathBuf;

#[derive(Debug)]
enum ConfigError {
    MissingField(String),
    InvalidNumber(String),
    FileNotFound(PathBuf),
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ConfigError::MissingField(field) => write!(f, "ไม่พบฟิลด์ '{field}'"),
            ConfigError::InvalidNumber(field) => write!(f, "ฟิลด์ '{field}' ไม่ใช่ตัวเลข"),
            ConfigError::FileNotFound(path) => write!(f, "ไม่พบไฟล์: {}", path.display()),
        }
    }
}

impl Error for ConfigError {}

// สอนให้ ConfigError "รู้วิธีแปลงตัวเอง" จาก ParseIntError โดยอัตโนมัติ
impl From<ParseIntError> for ConfigError {
    fn from(e: ParseIntError) -> Self {
        // หมายเหตุ: ณ จุดนี้ From ไม่รู้ "ชื่อฟิลด์" ที่กำลัง parse อยู่เลย
        // เพราะมันรับรู้แค่ตัว error เอง (ข้อจำกัดที่จะแก้ในหัวข้อ 30.5-30.6 ด้วยการ "ห่อ" error)
        ConfigError::InvalidNumber(e.to_string())
    }
}

fn parse_timeout(raw: &str) -> Result<u32, ConfigError> {
    // ตอนนี้ ? compile ผ่านแล้ว เพราะ compiler รู้ว่าจะเรียก
    // ConfigError::from(the_parse_int_error) ให้เองโดยอัตโนมัติเมื่อเจอ Err
    let timeout = raw.parse::<u32>()?;
    Ok(timeout)
}

fn main() {
    println!("{:?}", parse_timeout("30"));
    println!("{:?}", parse_timeout("abc"));
}
```

ผลลัพธ์:

```
Ok(30)
Err(InvalidNumber("invalid digit found in string"))
```

จุดสำคัญที่ต้องเน้นย้ำ (ต่อยอดจาก Part 12 หัวข้อ 12.7): **โค้ดในฟังก์ชัน `parse_timeout` เองไม่มีการเรียก
`.from()`/`.into()`/`.map_err()` ให้เห็นตรง ๆ เลย** ทุกอย่างถูกทำโดย `?` แบบ **implicit** ตามกฎ desugar — คุณ
เขียน `impl From` แค่ **ครั้งเดียว** ที่ระดับ type `ConfigError` แล้ว `?` ทุกจุดในโปรแกรมทั้งหมดที่มีสถานการณ์
"เจอ `ParseIntError` แล้วต้องแปลงเป็น `ConfigError`" จะทำงานได้ทันทีโดยไม่ต้องเขียนโค้ดแปลงซ้ำเลย — นี่คือพลังที่
แท้จริงของการผสาน `?` กับ `From`: **การลงทุนเขียน `impl From` หนึ่งครั้ง ให้ผลตอบแทนที่ทุกจุดเรียกใช้ตลอดทั้ง
โปรแกรม**

แต่สังเกต comment ที่แทรกไว้ในโค้ด: **`From::from(error)` รับรู้แค่ตัว `error` เพียงอย่างเดียว ไม่มีทาง "ส่ง
context เพิ่มเติม" เข้าไปได้** (เช่น "กำลัง parse ฟิลด์ไหนอยู่ตอนนั้น") เพราะ `fn from(e: ParseIntError) -> Self`
มี parameter เดียวคือ `e` เท่านั้น — นี่คือข้อจำกัดที่แท้จริงของการพึ่งพา `?`+`From` แบบ implicit ล้วน ๆ ซึ่งเป็น
เหตุผลที่หัวข้อถัดไปจะแนะนำวิธี **"ห่อ" error ไว้เป็นข้อมูล** (แทนที่จะแปลงมันให้หายไปเป็น `String`) เพื่อให้ทั้ง
context และตัว error ดั้งเดิมไม่สูญหาย

### 30.5 Error Chains: การไล่สาเหตุที่แท้จริงด้วย `source()`

จากหัวข้อก่อน เราเห็นข้อจำกัดของการแปลง error ด้วย `From` แบบตรง ๆ: มันเปลี่ยน `ParseIntError` ให้กลายเป็นแค่
`String` ข้อความ ทำให้ **ข้อมูลต้นฉบับของ error หายไป** (เช่น `ParseIntError` มี `kind` ที่แยกแยะ `InvalidDigit`
จาก `PosOverflow` ได้ตามที่ Part 12 หัวข้อ 12.3 อธิบายไว้ แต่พอแปลงเป็น `String` แล้ว ข้อมูลนั้นก็จมอยู่ในข้อความ
ล้วน ๆ ตรวจสอบด้วย pattern matching ต่อไม่ได้อีก)

วิธีที่ดีกว่าคือ **เก็บ error ดั้งเดิมไว้เป็นข้อมูลภายใน variant เลย** (ไม่ใช่แปลงทิ้งเป็น `String`) แล้ว override
method `source()` ให้ชี้กลับไปยัง error ที่เก็บไว้นั้น — นี่คือสิ่งที่เรียกว่า **error chain** (โซ่ของ error ที่
เชื่อมกันเป็นชั้น ๆ): error ชั้นบนสุด "เกิดจาก" error ชั้นล่างกว่า ซึ่งอาจ "เกิดจาก" error ชั้นล่างกว่านั้นอีกที
ไปเรื่อย ๆ จนถึงต้นตอที่แท้จริง

```rust
use std::error::Error;
use std::fmt;
use std::num::ParseIntError;

#[derive(Debug)]
enum ConfigError {
    MissingField(String),
    // เปลี่ยนจาก InvalidNumber(String) เป็น struct variant ที่เก็บ ParseIntError ดั้งเดิมไว้จริง ๆ
    InvalidNumber { field: String, source: ParseIntError },
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ConfigError::MissingField(field) => write!(f, "ไม่พบฟิลด์ '{field}'"),
            ConfigError::InvalidNumber { field, .. } => {
                write!(f, "ฟิลด์ '{field}' ไม่ใช่ตัวเลขที่ถูกต้อง")
            }
        }
    }
}

impl Error for ConfigError {
    // override source() เพื่อ "ชี้กลับ" ไปยัง error ดั้งเดิมที่เก็บไว้
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            ConfigError::InvalidNumber { source, .. } => Some(source),
            ConfigError::MissingField(_) => None, // ไม่มี error ต้นตออื่นให้ชี้ไป
        }
    }
}

#[derive(Debug)]
enum AppError {
    Config(ConfigError),
}

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AppError::Config(_) => write!(f, "โหลดการตั้งค่าแอปพลิเคชันล้มเหลว"),
        }
    }
}

impl Error for AppError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            AppError::Config(e) => Some(e),
        }
    }
}

impl From<ConfigError> for AppError {
    fn from(e: ConfigError) -> Self {
        AppError::Config(e)
    }
}

fn parse_field(field: &str, raw: &str) -> Result<u32, ConfigError> {
    raw.parse::<u32>().map_err(|source| ConfigError::InvalidNumber {
        field: field.to_string(),
        source,
    })
}

fn load_timeout(raw: &str) -> Result<u32, AppError> {
    let timeout = parse_field("timeout_seconds", raw)?; // ConfigError -> AppError ผ่าน From
    Ok(timeout)
}

// ฟังก์ชันที่ "ไล่" error chain ทั้งหมด พิมพ์ทุกชั้นตั้งแต่บนสุดลงไปถึงต้นตอ
fn print_error_chain(err: &dyn Error) {
    println!("error: {err}");
    let mut source = err.source();
    while let Some(cause) = source {
        println!("caused by: {cause}");
        source = cause.source();
    }
}

fn main() {
    match load_timeout("abc") {
        Ok(t) => println!("timeout = {t}"),
        Err(e) => print_error_chain(&e),
    }
}
```

ผลลัพธ์:

```
error: โหลดการตั้งค่าแอปพลิเคชันล้มเหลว
caused by: ฟิลด์ 'timeout_seconds' ไม่ใช่ตัวเลขที่ถูกต้อง
caused by: invalid digit found in string
```

**อธิบายกลไกทีละส่วน:**

- **โครงสร้างของ chain ในตัวอย่างนี้มี 3 ชั้น**: `AppError::Config(ConfigError::InvalidNumber { .. })` (ชั้นบน
  สุด — ระดับแอปพลิเคชัน) → `ConfigError::InvalidNumber` (ชั้นกลาง — ระดับการตั้งค่า) → `ParseIntError` (ชั้น
  ล่างสุด — ต้นตอที่แท้จริงระดับ standard library) — สังเกตว่า `ParseIntError` **ไม่ต้อง override `source()`
  เอง** เพราะมันไม่ได้ห่อ error อื่นไว้ข้างใน `source()` ของมันจึงใช้ default implementation ที่คืน `None`
  เสมอ ซึ่งเป็นสัญญาณบอก `print_error_chain` ว่า **นี่คือจุดสิ้นสุดของ chain แล้ว** (`while let Some(cause) = ...`
  จะหยุด loop เมื่อได้ `None`)
- `InvalidNumber { field: String, source: ParseIntError }` — เราเปลี่ยนจาก tuple variant (`InvalidNumber(String)`
  ในหัวข้อก่อน) เป็น **struct variant** ที่มีชื่อ field ชัดเจน สังเกตว่าตั้งชื่อ field ว่า `source` ตรง ๆ (ไม่ใช่
  ข้อบังคับทาง syntax แต่เป็น **ธรรมเนียมที่นิยมมากในโค้ด Rust จริง** — เมื่อคนอ่านเห็น field ชื่อ `source` ใน
  error enum จะเข้าใจทันทีว่ามันคือ error ดั้งเดิมที่ `source()` จะคืนออกไป — ธรรมเนียมนี้สำคัญพอที่ crate
  `thiserror` ใน Part 31 จะมี attribute พิเศษชื่อ `#[source]`/`#[from]` ที่ผูกกับชื่อธรรมเนียมนี้โดยตรง)
- `fn source(&self) -> Option<&(dyn Error + 'static)> { match self { ... Some(source) ... } }` — นี่คือหัวใจ
  ของ error chain: การ override `source()` ให้ **คืน reference ไปยัง field ที่เก็บ error ดั้งเดิมไว้** สังเกต
  ว่า `Some(source)` ที่นี่ `source` คือชื่อ field (ผ่าน pattern `ConfigError::InvalidNumber { source, .. }`)
  ไม่ใช่ `self.source()` เรียกตัวเอง — Rust ยอมให้ field และ method ชื่อเดียวกันอยู่ร่วมกันได้เพราะ syntax
  ต่างกันชัดเจน (`source` เดี่ยว ๆ ใน pattern คือตัวแปร ส่วน `x.source()` ต้องมี `.()` เรียก method เท่านั้น)
- `impl Error for AppError { fn source(&self) -> ... { match self { AppError::Config(e) => Some(e) } } }` — ชั้น
  บนสุดก็ override `source()` เหมือนกัน โดยชี้ไปยัง `ConfigError` ที่ห่อไว้ — **แต่ละชั้นรับผิดชอบแค่ "ชี้ไปยังชั้น
  ที่อยู่ต่ำกว่าตัวเองหนึ่งชั้น" เท่านั้น ไม่ต้องรู้เรื่องชั้นที่ลึกกว่านั้นเลย** (`AppError` ไม่ต้องรู้จัก
  `ParseIntError` เลยแม้แต่นิดเดียว) — การไล่ทั้ง chain จนถึงต้นตอเป็นความรับผิดชอบของ **ฟังก์ชันที่ไล่ loop**
  (`print_error_chain`) ไม่ใช่ความรับผิดชอบของ error type แต่ละตัว — นี่คือการแบ่งหน้าที่ที่สะอาดมาก: แต่ละชั้น
  รู้แค่ "เพื่อนบ้านของตัวเอง" พอ
- `print_error_chain(err: &dyn Error)` — ฟังก์ชันนี้รับ **`&dyn Error` แบบทั่วไป** ไม่ผูกกับ `AppError` หรือ
  `ConfigError` โดยเฉพาะ ทำให้มันใช้ไล่ chain ของ error type **ใดก็ได้** ที่ implement `Error` อย่างถูกต้อง —
  `let mut source = err.source();` เริ่มด้วยชั้นถัดจากชั้นบนสุด แล้ว `while let Some(cause) = source { ...
  source = cause.source(); }` ไล่ลงไปทีละชั้นจนกว่าจะได้ `None` (จุดสิ้นสุดของ chain) — pattern การเขียนแบบนี้
  (`while let` ไล่ `.source()` ซ้ำ ๆ) เป็น pattern มาตรฐานที่ใช้กันทั่วไปในโค้ด Rust จริงสำหรับพิมพ์ error chain
  แบบเต็ม (และเป็นสิ่งที่ macro ของ `thiserror`/`anyhow` ใน Part 31 มักมี helper ทำให้อัตโนมัติ)

**ทำไม error chain สำคัญ**: ลองนึกภาพระบบจริงที่มีหลายชั้น (application → service layer → database layer →
network layer) — ถ้า error ทุกชั้น "แปลง" error ของชั้นล่างให้เป็นแค่ `String` ทันทีที่รับมา (เหมือนที่หัวข้อ 30.4
ทำ) เมื่อ error ไปถึงชั้นบนสุด **ข้อมูลของทุกชั้นตรงกลางจะถูกกลืนหายไปในข้อความเดียว** ทำให้ debug ยากมาก (รู้แค่
"database บอกว่า...", ไม่รู้ว่า connection เสียตั้งแต่ layer ไหน) การเก็บ error ดั้งเดิมไว้เป็นข้อมูล (ผ่าน
`source: SomeError` field) แทนการแปลงทิ้งทันที ทำให้ **ทุกชั้นของ context ยังอยู่ครบ** พร้อมให้ระบบ log
(หรือคนอ่าน) ไล่ดูได้ทุกรายละเอียดเมื่อจำเป็น โดยที่ `Display` ของชั้นบนสุดยังแสดงข้อความสั้น กระชับ อ่านง่ายให้
ผู้ใช้ทั่วไปได้ตามปกติ (เพราะ `Display` ของ `AppError` ไม่ได้พิมพ์รายละเอียดทุกชั้นออกมาปนกันในบรรทัดเดียว — นั่น
เป็นหน้าที่ของ `print_error_chain` ที่ไล่ `source()` เอง)

#### เปรียบเทียบกับภาษาอื่น: exception chaining ที่ทำงานอัตโนมัติกว่า แต่ตรวจสอบได้น้อยกว่า

แนวคิด "error chain" ไม่ใช่ของใหม่ที่ Rust คิดขึ้นเอง — ภาษาที่ใช้ exception ก็มีกลไกคล้ายกันมาก แต่ทำงานต่างกัน
ในรายละเอียดที่สำคัญ ลองดู Python ก่อน:

```python
# ภาษา Python — ตัวอย่างเปรียบเทียบเท่านั้น ไม่ใช่ Rust
def parse_timeout(raw):
    try:
        return int(raw)
    except ValueError as e:
        raise ConfigError("timeout ไม่ถูกต้อง") from e  # "from e" ผูก __cause__ ให้อัตโนมัติ

try:
    parse_timeout("abc")
except ConfigError as e:
    print(f"error: {e}")
    print(f"caused by: {e.__cause__}")  # Python เก็บ __cause__ ให้เองจาก "raise ... from ..."
```

Java มีกลไกคล้ายกันผ่าน constructor ที่รับ `cause`:

```java
// ภาษา Java — ตัวอย่างเปรียบเทียบเท่านั้น ไม่ใช่ Rust
try {
    Integer.parseInt(raw);
} catch (NumberFormatException e) {
    throw new ConfigException("timeout ไม่ถูกต้อง", e); // constructor เก็บ cause ให้อัตโนมัติ
}
// ...
} catch (ConfigException e) {
    System.out.println("error: " + e.getMessage());
    System.out.println("caused by: " + e.getCause());
}
```

ทั้ง Python (`raise ... from ...` ตั้งค่า `__cause__`) และ Java (constructor ของ `Throwable` ที่รับ `cause`
เป็น parameter) มี **runtime เก็บ "สาเหตุ" ให้อัตโนมัติเป็นส่วนหนึ่งของกลไก exception เอง** — คุณไม่ต้องเขียน
method เหมือน `source()` ด้วยมือเลย เพียงแค่ "โยน exception ใหม่พร้อมอ้างถึง exception เดิม" ทั้งสองภาษาก็จัดการ
เก็บ chain ให้เอง ดูเผิน ๆ เหมือนสะดวกกว่า Rust ที่ต้องเขียน `source()` เองทุก type

**แต่มีความต่างเชิง design ที่สำคัญสองข้อ**:

1. **`source()` ของ Rust ถูกตรวจสอบตอน compile time ว่ามีอยู่จริงหรือไม่** (ผ่าน supertrait requirement ของ
   `Error` ตามหัวข้อ 30.2) ส่วน `__cause__`/`getCause()` ของ Python/Java เป็น "ข้อมูลที่อาจมีหรือไม่มีก็ได้"
   (`None`/`null` ถ้าไม่ได้ตั้งไว้) ที่ตรวจสอบได้แค่ตอน runtime — ถ้าโปรแกรมเมอร์ Java ลืมส่ง `cause` เข้าไปใน
   constructor (`throw new ConfigException("timeout ไม่ถูกต้อง")` โดยไม่มี `e`) โค้ดก็ยัง compile ผ่านได้สบาย ๆ
   แต่ chain จะขาดไปเงียบ ๆ โดยไม่มีอะไรเตือน ขณะที่ใน Rust การเขียน `fn source(&self) -> ... { None }` ทั้งที่
   ตัวเองมี field เก็บ error ต้นตออยู่จริงเป็นสิ่งที่ **ต้องตั้งใจเขียนผิดเท่านั้นถึงจะเกิดขึ้น** — ไม่ใช่ค่า
   default ที่ "ลืมได้ง่าย" แบบเดียวกัน (เพราะคุณต้อง pattern match กับ field ของตัวเองอยู่ดี)
2. **Python/Java เดินทาง "ขึ้น" ผ่าน stack unwinding โดยอัตโนมัติเสมอเมื่อ throw** (ตามที่ Part 12 หัวข้อ 12.1
   อธิบายไว้เรื่องกลไก exception) ส่วน chain ที่เก็บไว้ (`__cause__`/`getCause()`) เป็นแค่ **ข้อมูลเสริม** ที่ติด
   มากับ exception object เท่านั้น ไม่เกี่ยวกับกลไก control flow เลย — ใน Rust ไม่มีการ unwind ข้าม `return`
   ธรรมดา (`?` ก็คือ `return` ตามที่ Part 12 อธิบายไว้) `source()` จึงเป็นแค่ **ความสัมพันธ์ระหว่างข้อมูล** ที่
   ไม่ผูกกับกลไก control flow ใด ๆ เลยแม้แต่นิดเดียว — สอดคล้องกับหลักการ zero-cost abstraction ที่ Part 12
   ย้ำไว้ตลอดทั้งบท: การมี error chain ใน Rust ไม่ได้เพิ่มต้นทุนอะไรให้กับ control flow ปกติเลย มันคือแค่ field
   ธรรมดาที่ `source()` คืนออกไปเฉย ๆ

พูดโดยสรุป: **Rust แลกความสะดวกทาง syntax (ต้องเขียน `source()` เองทุก type) กับความปลอดภัยที่ตรวจสอบได้ตอน
compile time (supertrait bound บังคับให้ `Error` ต้องมี `Debug`/`Display` เสมอ แม้ `source()` จะ optional ก็ตาม)**
— นี่คือ trade-off แบบเดียวกับที่เราเห็นซ้ำ ๆ ตลอดทั้งหลักสูตรนี้ระหว่างภาษาที่ใช้ runtime ตรวจสอบเป็นหลัก กับ
Rust ที่ผลักการตรวจสอบให้มากที่สุดไปที่ compile time

### 30.6 การห่อ (Wrapping) Error หลายชนิดไว้ใน Enum เดียว

ในระบบจริง ฟังก์ชันหนึ่งมักล้มเหลวได้จาก **แหล่งที่มาหลายแบบที่แตกต่างกันโดยสิ้นเชิง** ไม่ใช่แค่ "parse ผิด" อย่าง
เดียว — ตัวอย่างที่พบบ่อยที่สุดคือ **I/O error** (อ่านไฟล์ไม่ได้) ผสมกับ **parse/validation error** (อ่านไฟล์ได้
แต่เนื้อหาผิด) เรามาออกแบบ `AppError` ที่ "ห่อ" ทั้งสองแหล่งไว้ในที่เดียว:

```rust
use std::error::Error;
use std::fmt;
use std::fs;
use std::io;
use std::num::ParseIntError;

#[derive(Debug)]
enum ConfigError {
    MissingField(String),
    InvalidNumber { field: String, source: ParseIntError },
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ConfigError::MissingField(field) => write!(f, "ไม่พบฟิลด์ '{field}'"),
            ConfigError::InvalidNumber { field, .. } => write!(f, "ฟิลด์ '{field}' ไม่ใช่ตัวเลข"),
        }
    }
}
impl Error for ConfigError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            ConfigError::InvalidNumber { source, .. } => Some(source),
            ConfigError::MissingField(_) => None,
        }
    }
}

// AppError ห่อ error ได้สองแหล่ง: จาก I/O (อ่านไฟล์ไม่ได้) และจาก ConfigError (เนื้อหาผิด)
#[derive(Debug)]
enum AppError {
    Config(ConfigError),
    Io(io::Error),
}

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AppError::Config(e) => write!(f, "การตั้งค่าไม่ถูกต้อง: {e}"),
            AppError::Io(e) => write!(f, "อ่านไฟล์ config ไม่สำเร็จ: {e}"),
        }
    }
}

impl Error for AppError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            AppError::Config(e) => Some(e),
            AppError::Io(e) => Some(e),
        }
    }
}

// impl From ให้ทั้งสองแหล่ง เพื่อให้ ? ใช้งานได้อิสระกับ error ทั้งสองชนิดในฟังก์ชันเดียว
impl From<ConfigError> for AppError {
    fn from(e: ConfigError) -> Self {
        AppError::Config(e)
    }
}

impl From<io::Error> for AppError {
    fn from(e: io::Error) -> Self {
        AppError::Io(e)
    }
}

fn load_config_file(path: &str) -> Result<String, AppError> {
    let content = fs::read_to_string(path)?; // io::Error -> AppError ผ่าน From ที่เพิ่งเขียน
    Ok(content)
}

fn main() {
    match load_config_file("/path/that/does/not/exist.conf") {
        Ok(content) => println!("อ่านสำเร็จ: {content}"),
        Err(e) => println!("error: {e}"),
    }
}
```

ผลลัพธ์ (ระบบไม่มีไฟล์นี้อยู่จริงในเครื่อง จึงเจอ I/O error ตามที่คาดไว้):

```
error: อ่านไฟล์ config ไม่สำเร็จ: No such file or directory (os error 2)
```

**อธิบายจุดสำคัญ:**

- `enum AppError { Config(ConfigError), Io(io::Error) }` — แต่ละ variant ห่อ error type **ที่มีอยู่แล้ว**ไว้
  ตรง ๆ เป็นข้อมูล (ไม่ใช่แปลงเป็น `String`) — `ConfigError` ที่เราสร้างเองจากหัวข้อก่อน และ `io::Error` ของ
  standard library เอง (ที่ implement `Error` ให้แล้วโดย default) ทั้งสองถูกห่อไว้ในระดับเดียวกัน โดยที่
  `AppError` ไม่ต้องรู้รายละเอียดภายในของทั้งสองเลย
- `impl From<ConfigError> for AppError` และ `impl From<io::Error> for AppError` — **นี่คือ pattern หลักของหัวข้อ
  นี้**: เขียน `impl From` แยกกันสำหรับ**แต่ละ**แหล่งที่มาของ error ที่อาจห่อไว้ ทำให้ `?` ใช้งานได้กับ**ทั้ง
  สองแหล่ง**อย่างอิสระในฟังก์ชันเดียวกัน โดยไม่ต้องเขียน `.map_err(AppError::Io)`/`.map_err(AppError::Config)`
  ด้วยมือทุกจุด — ถ้าฟังก์ชันของคุณมี 5 จุดที่อาจเกิด `io::Error` และ 3 จุดที่อาจเกิด `ConfigError` การเขียน
  `impl From` แค่สองครั้งนี้ก็ครอบคลุมทั้ง 8 จุดพร้อมกัน
- `fs::read_to_string(path)?` — บรรทัดเดียวนี้แสดงผลลัพธ์ของการลงทุนเขียน `impl From<io::Error>`: `?` แปลง
  `io::Error` เป็น `AppError` ให้อัตโนมัติโดยไม่มีโค้ดแปลงให้เห็นตรงจุดนี้เลย เหมือนกับที่หัวข้อ 30.4 อธิบายไว้
  กับ `ParseIntError`
- **ข้อสังเกตเชิงออกแบบที่สำคัญ**: enum ที่ "ห่อ" error หลายชนิดแบบนี้ (บางครั้งเรียกว่า **wrapper error
  enum**) เป็น pattern ที่พบมากที่สุดในโค้ด Rust จริงระดับ application — แทนที่จะพยายามยัดทุกสาเหตุความล้มเหลว
  ไว้ใน enum แบบ "flat" เดียว (เช่นพยายามให้ `AppError` มี variant `MissingField`/`InvalidNumber`/`FileNotFound`
  ของตัวเองซ้ำกับ `ConfigError`) เราปล่อยให้ `ConfigError` ยังคงมีหน้าที่ของมันเองอย่างสมบูรณ์ (เกี่ยวกับการตั้งค่า
  เท่านั้น) แล้ว `AppError` ก็แค่ "ห่อ" มันไว้อีกชั้นเดียว — โครงสร้างแบบนี้ **scale ได้ดีกว่ามาก** เมื่อระบบมี
  หลาย module ย่อย เพราะแต่ละ module ดูแล error ของตัวเองได้อย่างเป็นอิสระ ไม่ต้องมากระทบ enum กลางทุกครั้งที่
  module ย่อยเปลี่ยนแปลง

### 30.7 `#[non_exhaustive]`: ป้องกัน Breaking Change เมื่อเพิ่ม Variant ใหม่ในอนาคต

ย้อนกลับไปที่หัวข้อ 30.3: เราชื่นชม `match` แบบ exhaustive บน error enum ว่าเป็นข้อดีเหนือ `Box<dyn Error>` — แต่
ข้อดีนี้มี**ด้านตรงข้าม**ที่ต้องระวังถ้า `ConfigError` เป็น **public type ของ library ที่คนอื่นเอาไปใช้**: **ถ้า
คุณ (ในฐานะผู้เขียน library) เพิ่ม variant ใหม่เข้าไปใน enum ในเวอร์ชันถัดไป โค้ดของผู้ใช้ทุกคนที่เขียน `match`
แบบ exhaustive ไว้ (ไม่มี `_ =>`) จะ **compile ไม่ผ่านทันที**** — นี่คือ **breaking change** ที่เกิดจากการเปลี่ยน
ที่ดู "ไม่น่าจะกระทบอะไร" (แค่เพิ่มความสามารถในการรายงาน error ให้ละเอียดขึ้น) แต่จริง ๆ แล้วทำลาย backward
compatibility ตาม semantic versioning

`#[non_exhaustive]` คือ attribute ที่แก้ปัญหานี้ตรง ๆ: มันบอก compiler ว่า **"enum/struct นี้อาจมี variant/field
เพิ่มขึ้นได้ในอนาคต โดยไม่ถือว่าเป็น breaking change — โค้ดจากนอก crate นี้ต้องเผื่อไว้เสมอ"** ผลที่ตามมาคือ
**เมื่อ enum ที่มี `#[non_exhaustive]` ถูก `match` จากโค้ดที่อยู่ **คนละ crate** กับที่นิยาม enum ไว้ compiler
จะบังคับให้ต้องมี `_ =>` เสมอ แม้จะ list ครบทุก variant ที่มีอยู่ ณ ปัจจุบันแล้วก็ตาม**

**สำคัญ**: effect ของ `#[non_exhaustive]` เกิดขึ้น**เฉพาะกับโค้ดที่อยู่คนละ crate**กับที่นิยาม type ไว้เท่านั้น
— ถ้า `match` อยู่ใน crate เดียวกับที่นิยาม enum (เหมือนตัวอย่างส่วนใหญ่ในบทนี้ที่ `main()` อยู่ไฟล์เดียวกับ
`enum`) `#[non_exhaustive]` จะ**ไม่มีผลอะไรเลย** — เพราะเหตุนี้ เราจะต้องจำลองสองไฟล์แยกกันเป็นคนละ crate เพื่อ
ให้เห็น effect จริง ๆ

ไฟล์แรก — สมมติว่านี่คือ library ชื่อ `config_lib` (`config_lib.rs`, compile ด้วย `--crate-type lib`):

```rust
#[non_exhaustive]
#[derive(Debug)]
pub enum ConfigError {
    MissingField(String),
    InvalidNumber(String),
}
```

ไฟล์ที่สอง — สมมติว่านี่คือโค้ดของ**ผู้ใช้** library ตัวนี้ (`downstream.rs`, compile แยกโดยอ้าง `--extern
config_lib=...`):

```rust
extern crate config_lib;
use config_lib::ConfigError;

fn describe(e: &ConfigError) -> &'static str {
    match e {
        ConfigError::MissingField(_) => "missing",
        ConfigError::InvalidNumber(_) => "invalid number",
        // ไม่มี _ => ... เพราะ list ครบทั้ง 2 variant ที่มีอยู่ ณ ตอนนี้แล้ว
    }
}

fn main() {
    let e = ConfigError::MissingField("port".to_string());
    println!("{}", describe(&e));
}
```

โค้ด `downstream.rs` นี้ **compile ไม่ผ่าน** แม้จะ list ครบทุก variant ที่มีอยู่จริงในตอนนี้ก็ตาม — นี่คือ error
message จริงที่ได้ (ทดสอบจริงด้วยการ compile สอง crate แยกกัน):

```
error[E0004]: non-exhaustive patterns: `&_` not covered
 --> downstream.rs:5:11
  |
5 |     match e {
  |           ^ pattern `&_` not covered
  |
note: `ConfigError` defined here
 --> config_lib.rs:3:1
  |
3 | pub enum ConfigError {
  | ^^^^^^^^^^^^^^^^^^^^
  = note: the matched value is of type `&ConfigError`
  = note: `ConfigError` is marked as non-exhaustive, so a wildcard `_` is necessary to match exhaustively
help: ensure that all possible cases are being handled by adding a match arm with a wildcard pattern or an explicit pattern as shown
  |
7 ~         ConfigError::InvalidNumber(_) => "invalid number",
8 ~         &_ => todo!(),
  |

error: aborting due to 1 previous error
```

สังเกตบรรทัด `` `ConfigError` is marked as non-exhaustive, so a wildcard `_` is necessary to match exhaustively``
— compiler บอกเหตุผลตรง ๆ ว่าทำไม `match` ที่ดู "ครบ" แล้วยังพังอยู่ดี **วิธีแก้คือเติม `_ =>` เข้าไป:**

```rust
extern crate config_lib;
use config_lib::ConfigError;

fn describe(e: &ConfigError) -> &'static str {
    match e {
        ConfigError::MissingField(_) => "missing",
        ConfigError::InvalidNumber(_) => "invalid number",
        _ => "unknown (variant ใหม่ที่เพิ่มมาทีหลัง)",
    }
}

fn main() {
    let e = ConfigError::MissingField("port".to_string());
    println!("{}", describe(&e));
}
```

ผลลัพธ์เมื่อรัน: `missing`

**ทำไม pattern นี้มีคุณค่าในการออกแบบ library จริง**: ลองนึกภาพว่า `config_lib` ออกเวอร์ชันใหม่ที่เพิ่ม variant
`ConfigError::PermissionDenied(String)` เข้าไป (เพื่อรายงาน error ได้ละเอียดขึ้น ซึ่งควรถือเป็นการเปลี่ยนแปลงแบบ
**minor** ตาม semantic versioning ไม่ใช่ **breaking**) — **โค้ด `downstream.rs` เวอร์ชันที่มี `_ =>` อยู่แล้ว
จะยัง compile ผ่านได้ทันทีโดยไม่ต้องแก้ไขอะไรเลย** (แค่ตกไปที่ arm `_` แทน) ในขณะที่ถ้า `config_lib` **ไม่ได้**
ใส่ `#[non_exhaustive]` ไว้ตั้งแต่แรก การเพิ่ม variant ใหม่แบบนี้จะทำให้โค้ดผู้ใช้ทุกคนที่ `match` แบบ exhaustive
ไว้ **compile ไม่ผ่านทันที** ซึ่งขัดกับความคาดหวังของ semantic versioning ที่ว่า "การเพิ่ม minor version ไม่ควร
ทำให้โค้ดที่ใช้งานอยู่พังกะทันหัน"

**หลักปฏิบัติที่แนะนำ**: ใส่ `#[non_exhaustive]` บน public error enum **ทุกตัวที่คุณคาดว่าในอนาคตอาจต้องเพิ่ม
สาเหตุความล้มเหลวแบบใหม่เข้าไปอีก** (ซึ่งสำหรับ error enum แล้ว มักจะเป็น "ทุกตัว" เพราะความล้มเหลวรูปแบบใหม่มัก
โผล่มาเรื่อย ๆ ตามที่ระบบพัฒนาไป) — ข้อแลกเปลี่ยนที่ต้องรับคือผู้ใช้ library ของคุณต้องเขียน `_ =>` เสมอ (เขียน
`match` แบบ "ครบทุกกรณีที่รู้จัก บวกทางออกสำหรับกรณีที่ไม่รู้จัก" แทนที่จะ "ครบทุกกรณีแบบสมบูรณ์") ซึ่งเป็นภาระ
เล็กน้อยที่คุ้มค่ามากเมื่อเทียบกับความเสี่ยงที่จะทำให้โค้ดของผู้ใช้ library พังโดยไม่ตั้งใจในทุกครั้งที่คุณอัปเดต
minor version

### 30.8 การเขียน `Display` ที่ดี: Error Message คือส่วนหนึ่งของ API

Part 12 หัวข้อ 12.3 ทิ้งหลักการไว้ว่า **"error type ที่ดีควรเป็น structured data ที่ผู้เรียกตรวจสอบและตัดสินใจ
ต่อได้ ไม่ใช่แค่ข้อความสำหรับมนุษย์อ่านเท่านั้น"** — บทนี้ทั้งบทแสดงให้เห็นวิธี "structured" ไปแล้ว (enum,
`source()`, wrapping) แต่ **ส่วน `Display`** ก็ยังสำคัญไม่แพ้กัน เพราะมันคือสิ่งที่ **มนุษย์** (ผู้ใช้ปลายทาง,
คนอ่าน log, นักพัฒนาที่กำลัง debug ตอนดึก) จะเห็นจริง ๆ

ลองเทียบ error message สองแบบสำหรับสถานการณ์เดียวกัน:

```rust
use std::fmt;

// แบบไม่ดี: บอกแค่ "มีอะไรผิดพลาด" กับตัวเลขที่ไม่มีความหมายต่อผู้อ่าน
struct BadError(u32);
impl fmt::Display for BadError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "Error {}", self.0)
    }
}

// แบบดี: บอกทั้ง "เกิดอะไรขึ้น" (WHAT) และ "ทำไม/ควรทำอย่างไรต่อ" (WHY/HOW)
struct GoodError {
    field: String,
    reason: String,
}
impl fmt::Display for GoodError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(
            f,
            "ฟิลด์ '{}' ไม่ถูกต้อง: {} (กรุณาตรวจสอบไฟล์ config แล้วแก้ค่าตามคำแนะนำ)",
            self.field, self.reason
        )
    }
}

fn main() {
    let bad = BadError(42);
    let good = GoodError {
        field: "timeout_seconds".to_string(),
        reason: "ต้องเป็นจำนวนเต็มบวก แต่พบค่าติดลบ".to_string(),
    };
    println!("แบบไม่ดี: {bad}");
    println!("แบบดี: {good}");
}
```

ผลลัพธ์:

```
แบบไม่ดี: Error 42
แบบดี: ฟิลด์ 'timeout_seconds' ไม่ถูกต้อง: ต้องเป็นจำนวนเต็มบวก แต่พบค่าติดลบ (กรุณาตรวจสอบไฟล์ config แล้วแก้ค่าตามคำแนะนำ)
```

**"Error 42" ไร้ประโยชน์แทบทั้งหมด**: ผู้อ่านต้องไปเปิด documentation (ถ้ามี) หรือ source code เพื่อหาว่า `42`
หมายถึงอะไร — เหมือนกับ error code แบบ `errno` ในภาษา C ที่ต้องเปิด manual คู่กันเสมอ ในขณะที่ **`GoodError`
ตอบคำถามที่ผู้อ่านต้องการรู้ครบทั้งสามข้อในบรรทัดเดียว**: "เกิดอะไรขึ้น" (ฟิลด์ `timeout_seconds` ผิด), "ทำไมถึง
ผิด" (ค่าติดลบ ทั้งที่ต้องเป็นบวก), และ "ควรทำอย่างไรต่อ" (ไปแก้ไฟล์ config)

**หลักการเขียน `Display` ที่ดีสำหรับ error type สรุปได้ 4 ข้อ:**

1. **ระบุ "อะไร" ที่ผิดให้เจาะจงที่สุด** — ชื่อ field ที่ผิด, path ที่หาไม่เจอ, ค่าที่ได้รับจริง ๆ (ไม่ใช่แค่
   "input ผิดรูปแบบ" กว้าง ๆ)
2. **ระบุ "ทำไม" ถ้าเป็นไปได้** — เงื่อนไขที่ละเมิด (เช่น "ต้องเป็นค่าบวก แต่ได้ค่าติดลบ" ดีกว่า "ค่าไม่ถูกต้อง")
3. **แนะนำวิธีแก้เมื่อทำได้** — ไม่ใช่ทุก error จะแนะนำวิธีแก้ได้ตรง ๆ แต่ถ้าทำได้ (เช่น "กรุณาตรวจสอบไฟล์ config")
   จะช่วยผู้ใช้ปลายทางได้มากโดยไม่ต้องเปิด documentation เพิ่ม
4. **ห้ามเขียนซ้ำสิ่งที่ `source()`/`Debug` จะแสดงอยู่แล้วโดยไม่จำเป็น** — ถ้า error ของคุณมี `source` ที่จะถูก
   ไล่แสดงต่อด้วย `print_error_chain` (หัวข้อ 30.5) `Display` ของชั้นนั้นควรอธิบายแค่ **มุมมองของชั้นตัวเอง** พอ
   (เช่น "การตั้งค่าไม่ถูกต้อง") ปล่อยให้รายละเอียดของ `source` แสดงเป็นชั้นถัดไปเอง ไม่ต้องพิมพ์ปนกันในบรรทัด
   เดียว (ดูตัวอย่างจริงในหัวข้อ 30.6 ที่ `AppError::Config` และ `AppError::Io` มีข้อความสั้นระดับตัวเองพอ)

เชื่อมโยงกับหลักการของ Part 12: ฟังก์ชันที่คืน `Result<T, E>` ประกาศ "สัญญา" ว่าอาจล้มเหลวได้ผ่าน type ของมันเอง
— **`Display` ของ `E` ก็เป็นส่วนหนึ่งของสัญญานั้นเหมือนกัน** เพราะมันคือสิ่งที่ผู้เรียก (หรือผู้ใช้ปลายทางของ
โปรแกรมที่ผู้เรียกสร้างขึ้น) จะได้เห็นจริงเมื่อสัญญานั้น "ผิดคำ" — การละเลยคุณภาพของ `Display` จึงเท่ากับละเลย
คุณภาพของ API ทั้งหมด แม้ว่า type system จะถูกต้องสมบูรณ์แบบทุกประการก็ตาม

### 30.8b `Into<T>`: อีกฝั่งของเหรียญเดียวกันกับ `From<T>`

ชื่อบทนี้พูดถึงทั้ง "From/Into" แต่ตลอดหัวข้อ 30.4 เราพูดถึง `From` เพียงฝั่งเดียว เพราะ `?` operator ใช้
`From::from` ตรง ๆ ตามกฎ desugar — แต่ยังมีอีกครึ่งหนึ่งของภาพที่ควรเข้าใจให้ครบ: trait `Into<T>`

standard library เขียน **blanket implementation** (impl ที่ครอบคลุมทุก type ที่ตรงเงื่อนไข ซึ่ง Part 22 อาจ
แนะนำไว้บ้างเรื่อง trait bound แบบนี้) ไว้ดังนี้:

```
impl<T, U> Into<U> for T
where
    U: From<T>,
{
    fn into(self) -> U {
        U::from(self)
    }
}
```

พูดเป็นภาษาคน: **ทุกครั้งที่คุณเขียน `impl From<A> for B` คุณจะได้ `A: Into<B>` มาโดยอัตโนมัติทันทีโดยไม่ต้องเขียน
อะไรเพิ่มเลย** — `From` และ `Into` จึงเป็น "สองมุมมองของความสัมพันธ์เดียวกัน": `From` มองจากฝั่ง "ปลายทาง"
(`B::from(a)` — "B รู้วิธีสร้างตัวเองจาก A") ส่วน `Into` มองจากฝั่ง "ต้นทาง" (`a.into()` — "a รู้วิธีแปลงตัวเอง
เป็น B") ทั้งสองทำสิ่งเดียวกันทุกประการ ต่างกันแค่ว่าใครเป็นผู้เรียก method

**คำถามที่ตามมาคือ: แล้วเมื่อไหร่ควรเขียน `.into()` เอง ทั้งที่ `?` ก็แปลงให้อัตโนมัติอยู่แล้ว?** คำตอบคือ
สถานการณ์ที่คุณ**ไม่ได้ใช้ `?`** แต่ยังต้องการแปลง error type ด้วยวิธีอื่น (เช่น ใน `.map_err()`, หรือตอนสร้าง
`Err(...)` ด้วยมือ) — `.into()` ทำให้เขียนโค้ดแบบนั้นได้กระชับโดยไม่ต้องเอ่ยชื่อ type ปลายทางตรง ๆ (compiler เดา
ให้จาก context เช่น return type ของฟังก์ชัน):

```rust
use std::error::Error;
use std::fmt;
use std::num::ParseIntError;

#[derive(Debug)]
enum ConfigError {
    InvalidNumber(String),
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ConfigError::InvalidNumber(msg) => write!(f, "ค่าตัวเลขไม่ถูกต้อง: {msg}"),
        }
    }
}
impl Error for ConfigError {}

impl From<ParseIntError> for ConfigError {
    fn from(e: ParseIntError) -> Self {
        ConfigError::InvalidNumber(e.to_string())
    }
}

// เพราะเรา impl From<ParseIntError> for ConfigError ไว้แล้ว
// standard library ให้ ParseIntError: Into<ConfigError> มาโดยอัตโนมัติ (blanket impl ข้างบน)
fn convert_manually(raw: &str) -> Result<u32, ConfigError> {
    match raw.parse::<u32>() {
        Ok(v) => Ok(v),
        // .into() ที่นี่เทียบเท่ากับ ConfigError::from(e) หรือใช้ raw.parse::<u32>()? ทุกประการ
        // compiler รู้ว่าต้องแปลงเป็น ConfigError เพราะ return type ของฟังก์ชันบอกไว้ชัดเจน
        Err(e) => Err(e.into()),
    }
}

// ฟังก์ชัน generic ที่ยอมรับ error ชนิดใดก็ได้ "ที่แปลงเป็น ConfigError ได้" — ใช้ trait bound Into<ConfigError>
// ตามหลักการ generics + trait bound ที่ Part 18/22 สอนไว้
fn build_config_error<E: Into<ConfigError>>(e: E) -> ConfigError {
    e.into()
}

fn main() {
    println!("{:?}", convert_manually("42"));
    println!("{:?}", convert_manually("xyz"));

    let parse_err = "xyz".parse::<u32>().unwrap_err();
    println!("{:?}", build_config_error(parse_err));
}
```

ผลลัพธ์:

```
Ok(42)
Err(InvalidNumber("invalid digit found in string"))
InvalidNumber("invalid digit found in string")
```

**อธิบายจุดสำคัญ:**

- `Err(e.into())` ใน `convert_manually` — เราไม่ได้ใช้ `?` ในฟังก์ชันนี้เลย (ใช้ `match` แบบเต็มแทนโดยตั้งใจ เพื่อ
  แสดงให้เห็นว่า `.into()` ใช้แยกจาก `?` ได้) แต่ผลลัพธ์เหมือนกับที่ `?` จะทำให้ทุกประการ เพราะภายใน `?` ก็เรียก
  กลไกแปลงชนิดแบบเดียวกันนี้อยู่ดี — `.into()` จึงเป็นเครื่องมือที่มีประโยชน์เมื่อต้อง **แปลง error แบบ manual
  โดยไม่ต้องการ early return ทันที** เช่น ใน `.map_err(Into::into)` ที่ใช้แปลง error กลางทาง (ระหว่าง chain
  ของ combinator จาก Part 12 หัวข้อ 12.9) โดยไม่ต้องเขียน closure ยาว ๆ
- `fn build_config_error<E: Into<ConfigError>>(e: E) -> ConfigError` — นี่คือประโยชน์อีกแบบของ `Into`: การเขียน
  ฟังก์ชัน **generic ที่รับ error ชนิดใดก็ได้ที่มีทางแปลงเป็น `ConfigError`** โดยไม่ต้องผูกกับชนิดใดชนิดหนึ่ง
  ตรง ๆ — ถ้าคุณมี `impl From<ParseIntError>` และ `impl From<io::Error>` ให้ `ConfigError` ทั้งคู่ ฟังก์ชันนี้จะ
  เรียกได้กับ error ทั้งสองชนิดโดยไม่ต้อง overload หรือเขียนฟังก์ชันซ้ำ — pattern นี้พบได้บ่อยใน library ที่มี
  API รับ "อะไรก็ได้ที่แปลงเป็น error ของเราได้" (คล้ายกับที่ `impl Into<String>` มักใช้เป็น parameter type ของ
  ฟังก์ชันที่รับ string ได้หลายรูปแบบ)
- **สรุปสั้น ๆ**: `From` คือสิ่งที่คุณ **implement** (สอน type ปลายทางให้รู้วิธีสร้างตัวเองจากอะไร) ส่วน `Into`
  คือสิ่งที่คุณ **ได้มาโดยอัตโนมัติ** (และเรียกใช้ผ่าน `.into()` เมื่อต้องการความสะดวกทาง syntax) — ในทางปฏิบัติ
  แล้วโค้ด Rust ส่วนใหญ่ (รวมถึงบทนี้ทั้งบท) **implement `From` เท่านั้น** แล้วปล่อยให้ `?`, `.into()`, และ trait
  bound `Into<T>` ในโค้ดคนอื่นใช้ประโยชน์จากมันโดยอัตโนมัติ แทบไม่มีเหตุผลที่จะต้อง `impl Into` ตรง ๆ ด้วยมือเอง
  เลยในทางปฏิบัติ (เพราะ blanket impl ครอบคลุมไว้หมดแล้ว)

### 30.9 เปรียบเทียบ 3 แนวทาง: `Box<dyn Error>` vs Hand-Rolled Enum vs `thiserror` (Part 31)

มาสรุปทุกอย่างที่เรียนมาด้วยการเปรียบเทียบ 3 แนวทางสำหรับสถานการณ์เดียวกัน: ฟังก์ชัน `load_port` ที่ parse string
เป็น `u16` แล้ว validate ว่าต้องไม่เป็น 0

**แนวทางที่ 1: `Box<dyn Error>` (ทางลัดจาก Part 12)**

```rust
use std::error::Error;
use std::fmt;

#[derive(Debug)]
struct ValidationError(String);
impl fmt::Display for ValidationError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", self.0)
    }
}
impl Error for ValidationError {}

fn load_port(raw: &str) -> Result<u16, Box<dyn Error>> {
    let port: u16 = raw.parse()?; // ParseIntError -> Box<dyn Error> อัตโนมัติ
    if port == 0 {
        return Err(Box::new(ValidationError("port ต้องไม่เป็น 0".to_string())));
    }
    Ok(port)
}

fn main() {
    match load_port("0") {
        Ok(p) => println!("port: {p}"),
        // รู้แค่ว่า error เกิด ไม่รู้ว่าเป็น parse หรือ validation กันแน่ (ไม่ downcast)
        Err(e) => println!("error: {e}"),
    }
    match load_port("abc") {
        Ok(p) => println!("port: {p}"),
        Err(e) => println!("error: {e}"),
    }
}
```

ผลลัพธ์:

```
error: port ต้องไม่เป็น 0
error: invalid digit found in string
```

**แนวทางที่ 2: Hand-Rolled Enum (แนวทางของบทนี้)**

```rust
use std::error::Error;
use std::fmt;
use std::num::ParseIntError;

#[derive(Debug)]
enum PortError {
    Parse(ParseIntError),
    Zero,
}

impl fmt::Display for PortError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            PortError::Parse(e) => write!(f, "port ไม่ใช่ตัวเลขที่ถูกต้อง: {e}"),
            PortError::Zero => write!(f, "port ต้องไม่เป็น 0"),
        }
    }
}

impl Error for PortError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            PortError::Parse(e) => Some(e),
            PortError::Zero => None,
        }
    }
}

impl From<ParseIntError> for PortError {
    fn from(e: ParseIntError) -> Self {
        PortError::Parse(e)
    }
}

fn load_port(raw: &str) -> Result<u16, PortError> {
    let port: u16 = raw.parse()?;
    if port == 0 {
        return Err(PortError::Zero);
    }
    Ok(port)
}

fn main() {
    match load_port("0") {
        Ok(p) => println!("port: {p}"),
        // ตอนนี้ match แยกกรณีได้จริง! ตอบสนองต่างกันไปตามสาเหตุที่แท้จริง
        Err(PortError::Parse(e)) => println!("รูปแบบผิด: {e}"),
        Err(PortError::Zero) => println!("port เป็น 0 ไม่ได้ — กรุณาระบุ port ที่ถูกต้อง"),
    }
    match load_port("abc") {
        Ok(p) => println!("port: {p}"),
        Err(PortError::Parse(e)) => println!("รูปแบบผิด: {e}"),
        Err(PortError::Zero) => println!("port เป็น 0 ไม่ได้ — กรุณาระบุ port ที่ถูกต้อง"),
    }
}
```

ผลลัพธ์:

```
port เป็น 0 ไม่ได้ — กรุณาระบุ port ที่ถูกต้อง
รูปแบบผิด: invalid digit found in string
```

สังเกตความต่างที่จับต้องได้: เวอร์ชันที่ 2 มี `match` ที่แยก `PortError::Parse` กับ `PortError::Zero` ออกจากกัน
**ได้จริง** ทำให้ตอบสนองข้อความไม่เหมือนกันไปตามสาเหตุ — สิ่งที่เวอร์ชันที่ 1 (`Box<dyn Error>`) ทำไม่ได้เลยโดยไม่
ต้อง downcast แต่แลกมาด้วยโค้ดที่ยาวขึ้นมาก: ต้องเขียน `enum`, `impl Display` (พร้อม `match` ข้างใน), `impl
Error` (พร้อม `source()`), และ `impl From` แยกกันทั้งหมด

**แนวทางที่ 3: `thiserror` (จะเรียนแบบเต็มใน Part 31 — ตัวอย่างด้านล่างเป็นแนวคิดเท่านั้น ยังไม่ compile เพราะ
ต้องมี external crate ที่บทนี้ยังไม่ใช้)**

```text
// (ตัวอย่างแนวคิด — ต้องเพิ่ม thiserror ใน Cargo.toml ก่อนจึง compile ได้จริง — Part 31)
#[derive(Debug, thiserror::Error)]
enum PortError {
    #[error("port ไม่ใช่ตัวเลขที่ถูกต้อง: {0}")]
    Parse(#[from] std::num::ParseIntError),

    #[error("port ต้องไม่เป็น 0")]
    Zero,
}
```

สังเกตว่า enum, `#[derive(Debug)]`, และ `match` ฝั่งผู้เรียกใน "แนวทางที่ 3" นี้ **เหมือนกับแนวทางที่ 2 ทุก
ประการ** — สิ่งที่ `thiserror` ทำคือ **สร้าง `impl Display`, `impl Error`, และ `impl From<ParseIntError>` ให้
อัตโนมัติจาก attribute `#[error(...)]` และ `#[from]`** ที่เราเขียนไว้บน enum เพียงไม่กี่บรรทัด — พูดให้ชัดที่สุด:
**`thiserror` ไม่ได้เปลี่ยนโครงสร้างของโค้ดหรือวิธีที่ `?`/`match` ทำงานเลยแม้แต่นิดเดียว มันแค่ generate โค้ดที่
เราเขียนด้วยมือในแนวทางที่ 2 ให้เราโดยอัตโนมัติจาก syntax ที่กระชับกว่า** — นี่คือเหตุผลที่บทนี้ต้องมาก่อน Part 31
เสมอ: **ถ้าไม่เข้าใจว่าแนวทางที่ 2 ทำงานอย่างไรและทำไมต้องมีแต่ละส่วน (Display/Error/From/source) การใช้
`thiserror` จะเป็นแค่ "ท่องจำ attribute" โดยไม่เข้าใจว่ามันแทนที่โค้ดอะไรอยู่จริง ๆ**

ตารางสรุปทั้ง 3 แนวทาง:

| แนวทาง | บรรทัดโค้ดโดยประมาณ | match แยกกรณีได้ | ต้องเขียนเอง | เหมาะกับ |
|---|---|---|---|---|
| `Box<dyn Error>` | สั้นที่สุด (~10 บรรทัด) | ไม่ได้ตรง ๆ | แทบไม่ต้องเขียน | script, prototype, `main()` |
| Hand-Rolled Enum | ยาวที่สุด (~35 บรรทัด) | ได้เต็มรูปแบบ | `Display`, `Error`, `From` ทั้งหมด | เรียนรู้กลไก, ระบบเล็กที่ไม่อยากเพิ่ม dependency |
| `thiserror` (Part 31) | สั้น (~10 บรรทัด) เหมือน `Box<dyn Error>` | ได้เต็มรูปแบบ เหมือน Hand-Rolled | แค่ attribute บน enum | library และระบบจริงส่วนใหญ่ |

### 30.10 ตัวอย่างจริงแบบเต็ม: Config File Parser พร้อม Error Chain ครบวงจร

มาประกอบทุกอย่างที่เรียนมาในบทนี้เข้าด้วยกันเป็นตัวอย่างเดียวที่สมบูรณ์: parser สำหรับไฟล์ config รูปแบบ
`key=value` (คนละบรรทัด) ที่ล้มเหลวได้ **3 แบบที่แตกต่างกันอย่างชัดเจน**: (1) อ่านไฟล์ไม่ได้ (I/O error), (2)
ขาดฟิลด์ที่จำเป็น, (3) ค่าของฟิลด์ไม่ถูกต้อง (ทั้งตัวเลขและ boolean) — พร้อมแสดง error chain แบบเต็มเมื่อล้มเหลว

```rust
use std::collections::HashMap;
use std::error::Error;
use std::fmt;
use std::fs;
use std::io;
use std::num::ParseIntError;

#[derive(Debug)]
struct AppConfig {
    port: u16,
    timeout_seconds: u32,
    verbose: bool,
}

#[derive(Debug)]
enum ConfigError {
    MissingField(String),
    InvalidNumber { field: String, source: ParseIntError },
    InvalidBool { field: String, value: String },
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ConfigError::MissingField(field) => write!(
                f,
                "ไม่พบฟิลด์ที่จำเป็น '{field}' กรุณาเพิ่มบรรทัด '{field}=...' ในไฟล์ config"
            ),
            ConfigError::InvalidNumber { field, source } => {
                write!(f, "ฟิลด์ '{field}' ต้องเป็นตัวเลข: {source}")
            }
            ConfigError::InvalidBool { field, value } => write!(
                f,
                "ฟิลด์ '{field}' ต้องเป็น true หรือ false แต่พบค่า '{value}'"
            ),
        }
    }
}

impl Error for ConfigError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            ConfigError::InvalidNumber { source, .. } => Some(source),
            _ => None,
        }
    }
}

#[derive(Debug)]
enum AppError {
    Io(io::Error),
    Config(ConfigError),
}

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AppError::Io(_) => write!(f, "ไม่สามารถอ่านไฟล์ config ได้"),
            AppError::Config(_) => write!(f, "ไฟล์ config มีข้อมูลไม่ถูกต้อง"),
        }
    }
}

impl Error for AppError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            AppError::Io(e) => Some(e),
            AppError::Config(e) => Some(e),
        }
    }
}

impl From<io::Error> for AppError {
    fn from(e: io::Error) -> Self {
        AppError::Io(e)
    }
}
impl From<ConfigError> for AppError {
    fn from(e: ConfigError) -> Self {
        AppError::Config(e)
    }
}

// ดึงค่าฟิลด์จาก HashMap หรือคืน ConfigError::MissingField ถ้าไม่พบ
fn get_field<'a>(map: &'a HashMap<String, String>, field: &str) -> Result<&'a str, ConfigError> {
    map.get(field)
        .map(|s| s.as_str())
        .ok_or_else(|| ConfigError::MissingField(field.to_string()))
}

// แปลง "key=value" ทีละบรรทัดให้เป็น HashMap (บรรทัดที่ผิดรูปแบบหรือว่างเปล่าจะถูกข้าม)
fn parse_lines(content: &str) -> HashMap<String, String> {
    let mut map = HashMap::new();
    for line in content.lines() {
        let line = line.trim();
        if line.is_empty() {
            continue;
        }
        if let Some((key, value)) = line.split_once('=') {
            map.insert(key.trim().to_string(), value.trim().to_string());
        }
    }
    map
}

fn parse_config(content: &str) -> Result<AppConfig, ConfigError> {
    let map = parse_lines(content);

    let port_raw = get_field(&map, "port")?;
    let port: u16 = port_raw.parse().map_err(|source| ConfigError::InvalidNumber {
        field: "port".to_string(),
        source,
    })?;

    let timeout_raw = get_field(&map, "timeout_seconds")?;
    let timeout_seconds: u32 =
        timeout_raw.parse().map_err(|source| ConfigError::InvalidNumber {
            field: "timeout_seconds".to_string(),
            source,
        })?;

    let verbose_raw = get_field(&map, "verbose")?;
    let verbose = match verbose_raw {
        "true" => true,
        "false" => false,
        other => {
            return Err(ConfigError::InvalidBool {
                field: "verbose".to_string(),
                value: other.to_string(),
            })
        }
    };

    Ok(AppConfig { port, timeout_seconds, verbose })
}

fn load_app_config(path: &str) -> Result<AppConfig, AppError> {
    let content = fs::read_to_string(path)?; // io::Error -> AppError
    let config = parse_config(&content)?; // ConfigError -> AppError
    Ok(config)
}

fn print_error_chain(err: &dyn Error) {
    println!("error: {err}");
    let mut source = err.source();
    while let Some(cause) = source {
        println!("caused by: {cause}");
        source = cause.source();
    }
}

fn main() {
    // กรณีที่ 1: ไฟล์ไม่พบ -> io::Error ห่อใน AppError::Io
    match load_app_config("/nonexistent/path/app.conf") {
        Ok(cfg) => println!("{cfg:?}"),
        Err(e) => print_error_chain(&e),
    }

    // กรณีที่ 2-4: เนื้อหาไฟล์ในรูปแบบต่าง ๆ (ทดสอบผ่าน parse_config ตรง ๆ)
    let cases = [
        "port=8080\ntimeout_seconds=30\nverbose=true",   // ถูกต้องสมบูรณ์
        "port=8080\ntimeout_seconds=30",                 // ขาดฟิลด์ verbose
        "port=abc\ntimeout_seconds=30\nverbose=true",     // port ไม่ใช่ตัวเลข
        "port=8080\ntimeout_seconds=30\nverbose=yes",     // verbose ไม่ใช่ true/false
    ];

    for case in cases {
        match parse_config(case) {
            Ok(cfg) => println!("โหลดสำเร็จ: {cfg:?}"),
            Err(e) => print_error_chain(&e),
        }
    }
}
```

ผลลัพธ์:

```
error: ไม่สามารถอ่านไฟล์ config ได้
caused by: No such file or directory (os error 2)
โหลดสำเร็จ: AppConfig { port: 8080, timeout_seconds: 30, verbose: true }
error: ไม่พบฟิลด์ที่จำเป็น 'verbose' กรุณาเพิ่มบรรทัด 'verbose=...' ในไฟล์ config
error: ฟิลด์ 'port' ต้องเป็นตัวเลข: invalid digit found in string
caused by: invalid digit found in string
error: ฟิลด์ 'verbose' ต้องเป็น true หรือ false แต่พบค่า 'yes'
```

**เดินผ่านผลลัพธ์ทีละกรณีเพื่อให้เห็นภาพรวมทั้งบท:**

- **กรณีที่ 1** (`load_app_config` กับไฟล์ที่ไม่มีจริง): `fs::read_to_string` คืน `io::Error` → `?` แปลงเป็น
  `AppError::Io` ผ่าน `From` → `print_error_chain` พิมพ์ทั้งข้อความระดับ `AppError` (`"ไม่สามารถอ่านไฟล์ config
  ได้"`) และ `caused by:` ของ `io::Error` ดั้งเดิม (`"No such file or directory (os error 2)"`) — chain มีแค่
  2 ชั้นเพราะ `io::Error` ไม่ได้ห่อ error อื่นไว้ต่อ (`source()` ของมันคืน `None`)
- **กรณีที่ 2** (ขาดฟิลด์ `verbose`): `get_field(&map, "verbose")?` คืน `ConfigError::MissingField` — สังเกตว่า
  chain มีแค่ **1 ชั้น** (`error: ...` โดยไม่มี `caused by:` ตามมาเลย) เพราะ `ConfigError::MissingField` ไม่มี
  error ต้นตออื่นให้ชี้ไป (`source()` ของ variant นี้คืน `None` ตามที่ออกแบบไว้ในหัวข้อ 30.5) — นี่คือพฤติกรรมที่
  ถูกต้อง: ไม่ใช่ทุก error จะมี chain ยาวเสมอไป บาง error ก็ **เป็นต้นตอของตัวเองอยู่แล้ว** ตั้งแต่แรก
- **กรณีที่ 3** (`port=abc`): เกิด chain ที่ครบ 3 ชั้นเหมือนตัวอย่างในหัวข้อ 30.5 — **แต่สังเกตให้ดีว่าข้อความ
  `invalid digit found in string` ปรากฏ **สองครั้ง**** ครั้งแรกฝังอยู่ในข้อความของ `ConfigError::InvalidNumber`
  เอง (เพราะ `Display` เขียน `write!(f, "... {source}")` แทรก `source` ไว้ในประโยคตรง ๆ) และครั้งที่สองมาจาก
  `caused by:` ที่ `print_error_chain` พิมพ์ต่อให้อีกที **นี่คือ trade-off ที่ควรรู้ในการออกแบบ `Display`**: ถ้า
  ต้องการให้ error message แบบสั้น (ไม่เรียก `print_error_chain`) อ่านเข้าใจได้ในตัวเองทันที ควรฝังข้อความของ
  `source` ไว้ใน `Display` (แบบที่ทำอยู่) แต่ถ้าต้องการหลีกเลี่ยงข้อความซ้ำตอนแสดง chain แบบเต็ม อาจเลือกให้
  `Display` พิมพ์แค่ระดับตัวเอง (`"ฟิลด์ 'port' ต้องเป็นตัวเลข"` โดยไม่มี `{source}`) แล้วปล่อยให้
  `print_error_chain` เป็นผู้แสดงรายละเอียดทั้งหมด — ไม่มีคำตอบที่ถูกเพียงหนึ่งเดียว ขึ้นอยู่กับว่าโค้ดส่วนใหญ่
  ของระบบคุณเรียก `Display` ตรง ๆ (ไม่ไล่ chain) หรือเรียกผ่าน `print_error_chain` เป็นหลัก
- **กรณีที่ 4** (`verbose=yes`): `ConfigError::InvalidBool` ไม่มี error ต้นตอ (มันตรวจสอบ string ตรง ๆ ไม่ได้
  เรียก `.parse()` ที่คืน error type อื่น) จึง chain มีแค่ 1 ชั้นเหมือนกรณีที่ 2

ตัวอย่างนี้แสดงให้เห็น **ทุกองค์ประกอบของบทนี้ทำงานร่วมกันในระบบเดียว**: custom error enum สองระดับ
(`ConfigError`/`AppError`), `impl Display`/`impl Error`/`impl From` ที่ครบถ้วนสำหรับทั้งคู่, `source()` ที่ทำงาน
ถูกต้อง (คืน `Some`/`None` ตามความเหมาะสมของแต่ละ variant), การห่อ error จากสองแหล่งที่มา (`io::Error` และ
`ConfigError`), และฟังก์ชันไล่ chain ที่ใช้ซ้ำได้กับ error type ใดก็ตามที่ implement `Error` อย่างถูกต้อง — นี่คือ
"วิธีที่ยาก" (the hard way) ที่ Part 31 จะแสดงให้เห็นว่า `thiserror` ลดโค้ดตรงไหนได้บ้าง โดยที่โครงสร้างและ
พฤติกรรมทั้งหมดยังเหมือนกันทุกประการ

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. ใช้ `?` ข้าม Error Type โดยไม่มี `impl From` ที่จำเป็น (E0277)

```rust
#[derive(Debug)]
struct ConfigError {
    message: String,
}

fn parse_timeout(raw: &str) -> Result<u32, ConfigError> {
    let timeout = raw.parse::<u32>()?; // ParseIntError -> ConfigError ไม่มีทางแปลง
    Ok(timeout)
}

fn main() {
    println!("{:?}", parse_timeout("30"));
}
```

Error ที่ได้ (อธิบายละเอียดแล้วในหัวข้อ 30.4):

```
error[E0277]: `?` couldn't convert the error to `ConfigError`
 --> src/main.rs:7:34
  |
6 | fn parse_timeout(raw: &str) -> Result<u32, ConfigError> {
  |                                ------------------------ expected `ConfigError` because of this
7 |     let timeout = raw.parse::<u32>()?;
  |                       --------------^ the trait `From<ParseIntError>` is not implemented for `ConfigError`
```

**สาเหตุ**: กฎของ `?` (หัวข้อ 30.4) คือ compiler ต้องหา `impl From<ErrorTypeต้นทาง> for ErrorTypeปลายทาง` ที่มี
อยู่จริงเสมอเมื่อสองชนิดไม่ตรงกัน — ถ้าหาไม่ได้ compile ไม่ผ่านทันที **วิธีแก้**: เขียน `impl
From<ParseIntError> for ConfigError` (หัวข้อ 30.4) หรือถ้าไม่ต้องการให้ `?` แปลงอัตโนมัติทุกจุด ใช้
`.map_err(|e| ConfigError { message: e.to_string() })` แทนที่จุดนั้นเฉพาะจุดเดียว (เหมือนที่ Part 12 หัวข้อ
12.9 แนะนำเรื่อง `.map_err()` ไว้) กับดักนี้พบบ่อยที่สุดตอนเพิ่มการเรียกฟังก์ชันใหม่ที่คืน error type ที่ยังไม่
เคยเจอมาก่อนเข้าไปในฟังก์ชันที่มี error type ของตัวเองอยู่แล้ว

### 2. ลืม Implement `Display` — Supertrait Bound ของ `Error` ไม่ผ่าน (E0277)

```rust
use std::error::Error;

#[derive(Debug)]
struct MyError;

impl Error for MyError {} // ไม่มี impl Display for MyError !
```

Error ที่ได้:

```
error[E0277]: `MyError` doesn't implement `std::fmt::Display`
 --> src/main.rs:6:16
  |
6 | impl Error for MyError {}
  |                ^^^^^^^ unsatisfied trait bound
  |
help: the trait `std::fmt::Display` is not implemented for `MyError`
 --> src/main.rs:4:1
  |
4 | struct MyError;
  | ^^^^^^^^^^^^^^
note: required by a bound in `std::error::Error`
```

**สาเหตุ**: ตามหัวข้อ 30.2 `Error` มี **supertrait requirement** ว่า `Error: Debug + Display` (ตรงกับหลักการ
supertrait จาก Part 21) — การ `impl Error for MyError` โดยที่ `MyError` ยังไม่ `impl Display` เป็นการละเมิด
supertrait bound ตรง ๆ compiler จึงปฏิเสธตั้งแต่บรรทัด `impl Error for MyError {}` เอง (ไม่ใช่ตอนใช้งานจริงที
หลัง) **วิธีแก้**: เขียน `impl std::fmt::Display for MyError { ... }` ก่อนบรรทัด `impl Error` เสมอ (`Display`
ไม่มี `#[derive]` ให้ ต้องเขียนด้วยมือทุกครั้ง ต่างจาก `Debug` ที่ `#[derive(Debug)]` จัดการให้ได้)

### 3. ลืม `#[derive(Debug)]` — Supertrait อีกตัวของ `Error` ไม่ผ่าน (E0277)

```rust
use std::error::Error;
use std::fmt;

struct MyError; // ไม่มี #[derive(Debug)] และไม่มี impl Debug ด้วยมือ

impl fmt::Display for MyError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "my error")
    }
}

impl Error for MyError {}
```

Error ที่ได้:

```
error[E0277]: `MyError` doesn't implement `Debug`
  --> src/main.rs:12:16
   |
12 | impl Error for MyError {}
   |                ^^^^^^^ the trait `Debug` is not implemented for `MyError`
   |
   = note: add `#[derive(Debug)]` to `MyError` or manually `impl Debug for MyError`
note: required by a bound in `std::error::Error`
help: consider annotating `MyError` with `#[derive(Debug)]`
```

**สาเหตุ**: เหมือนกับกับดักที่ 2 แต่เป็น supertrait อีกตัว (`Debug` แทน `Display`) — สังเกตว่าครั้งนี้ compiler
เขียนไว้ตรง ๆ ว่า `impl Error` คือจุดที่ตรวจ bound ทั้งสองตัว ไม่ใช่แค่ตัวเดียว **วิธีแก้**: เติม
`#[derive(Debug)]` ไว้บน struct/enum เกือบทุกครั้งที่จะเป็น error type ก็เพียงพอ (ยกเว้นบางกรณีที่ field ภายใน
ไม่ implement `Debug` เอง ซึ่งต้อง `impl Debug` ด้วยมือแทน) **ข้อสังเกตเชิงปฏิบัติ**: เพราะทั้งกับดักที่ 2 และ 3
มาจาก supertrait bound เดียวกัน วิธีป้องกันที่ตรงจุดที่สุดคือจำสูตรสำเร็จไว้ตั้งแต่สร้าง error type ใหม่ทุกครั้ง:
**`#[derive(Debug)]` เหนือ enum เสมอ ตามด้วย `impl Display` ที่เขียนเอง ก่อนจะ `impl Error` ได้**

### 4. Signature ของ `source()` ผิด — ลืม `+ 'static` (Incompatible Signature)

```rust
use std::error::Error;
use std::fmt;

#[derive(Debug)]
struct MyError;

impl fmt::Display for MyError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "my error")
    }
}

impl Error for MyError {
    fn source(&self) -> Option<&dyn Error> { // ลืม + 'static !
        None
    }
}
```

Error ที่ได้:

```
error: `impl` item signature doesn't match `trait` item signature
  --> src/main.rs:14:5
   |
14 |     fn source(&self) -> Option<&dyn Error> {
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ found `fn(&'1 MyError) -> Option<&'1 (dyn std::error::Error + '1)>`
   |
   = note: expected `fn(&'1 MyError) -> Option<&'1 (dyn std::error::Error + 'static)>`
   = note: expected signature `fn(&'1 MyError) -> Option<&'1 (dyn std::error::Error + 'static)>`
              found signature `fn(&'1 MyError) -> Option<&'1 (dyn std::error::Error + '1)>`
   = help: the lifetime requirements from the `impl` do not correspond to the requirements in the `trait`
```

**สาเหตุ**: ตามหัวข้อ 30.2 นิยามจริงของ `source()` คือ `fn source(&self) -> Option<&(dyn Error + 'static)>` —
ถ้าเขียน `-> Option<&dyn Error>` โดยไม่มี `+ 'static` ชัดเจน กฎ **lifetime elision** (ที่ Part 20/23 อธิบายไว้)
จะเดา lifetime ของ `dyn Error` ให้ **ผูกกับ `&self`** โดยอัตโนมัติ (กลายเป็น `dyn Error + '1` ที่ผูกกับ lifetime
ของ `self`) ซึ่ง**ต่างจากที่ trait ต้นฉบับกำหนดไว้** (`'static` แปลว่า "ไม่ผูกกับ lifetime ของ `self` เลย")
ทำให้ signature ไม่ตรงกัน compiler จึงปฏิเสธ **วิธีแก้**: เขียน `Option<&(dyn Error + 'static)>` แบบเต็มเสมอเมื่อ
override `source()` (คัดลอก signature จากนิยามต้นฉบับมาตรง ๆ ปลอดภัยที่สุด) กับดักนี้พบบ่อยเพราะ IDE/editor บาง
ตัวแนะนำ auto-complete แบบสั้น (`Option<&dyn Error>`) ที่ดูถูกแต่จริง ๆ ขาด `+ 'static` ไป

### 5. `#[non_exhaustive]` ไม่มีผลกับ `match` ที่อยู่ใน crate เดียวกัน — เข้าใจผิดว่าไม่ทำงาน

```rust
#[non_exhaustive]
#[derive(Debug)]
enum ConfigError {
    MissingField(String),
    InvalidNumber(String),
}

fn describe(e: &ConfigError) -> &'static str {
    // match แบบนี้ compile ผ่านได้สบาย ๆ แม้ไม่มี _ => เพราะอยู่ crate เดียวกับที่นิยาม enum
    match e {
        ConfigError::MissingField(_) => "missing",
        ConfigError::InvalidNumber(_) => "invalid number",
    }
}

fn main() {
    let e = ConfigError::MissingField("port".to_string());
    println!("{}", describe(&e));
}
```

โค้ดนี้ **compile ผ่านได้ปกติ** (ไม่มี error เลย) ผลลัพธ์: `missing`

**สาเหตุ**: ตามที่อธิบายไว้ในหัวข้อ 30.7 **`#[non_exhaustive]` มีผลเฉพาะกับโค้ดที่อยู่คนละ crate กับที่นิยาม
type ไว้เท่านั้น** — เพราะ compiler ถือว่าโค้ดใน crate เดียวกันสามารถ "เห็น" การเปลี่ยนแปลงของ enum ได้พร้อมกัน
เสมอ (คุณแก้ enum กับแก้ `match` ในที่เดียวกันได้ในคราวเดียว ไม่มีปัญหาเรื่อง version ไม่ตรงกันระหว่างสองฝ่าย)
กับดักนี้ทำให้ผู้เริ่มต้นหลายคนคิดว่า `#[non_exhaustive]` "ไม่ทำงาน" หรือ "ไม่มีประโยชน์" ทั้งที่จริง ๆ มันทำงาน
ถูกต้องตามที่ออกแบบไว้ **วิธีทดสอบที่ถูกต้อง**: ต้องแยก enum ไปอยู่ใน crate/library คนละตัวกับโค้ดที่ `match`
มัน (เหมือนตัวอย่างสองไฟล์ในหัวข้อ 30.7) จึงจะเห็น effect จริง — ถ้าคุณกำลังเขียน library ที่จะถูก publish ให้คน
อื่นใช้ (เช่นผ่าน crates.io หรือ workspace ที่มีหลาย crate) `#[non_exhaustive]` จะมีผลกับผู้ใช้เหล่านั้นแน่นอน แม้
จะดู "ไม่มีผลอะไร" ตอนคุณทดสอบเองใน crate เดียวกันก็ตาม

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** ต่อยอดจาก `ConfigError` เวอร์ชันพื้นฐานในหัวข้อ 30.3 (สามตัว: `MissingField`, `InvalidNumber`,
   `FileNotFound`) ให้เพิ่ม variant ใหม่ชื่อ `EmptyValue(String)` สำหรับสถานการณ์ที่ฟิลด์นั้นมีอยู่ในไฟล์
   แต่ค่าของมันเป็น string ว่างเปล่า (เช่น `port=`) — ต้องแก้ทั้ง `impl Display` (เพิ่ม match arm ใหม่พร้อม
   ข้อความที่ดีตามหลักการหัวข้อ 30.8) และ `match` ใดๆ ที่ทดสอบ enum นี้อยู่แล้วในโค้ดของคุณ (hint: เพราะยังไม่ใส่
   `#[non_exhaustive]` การเพิ่ม variant ใหม่จะทำให้ `match` เดิมที่ไม่มี `_ =>` compile ไม่ผ่านทันที — สังเกต
   ปรากฏการณ์นี้ด้วยตัวเองแล้วอธิบายว่าทำไม compiler ถึงฟ้อง ก่อนจะแก้ให้ครบ)

2. **[กลาง]** ปรับ `ConfigError::InvalidNumber` จากรูปแบบ `InvalidNumber(String)` (แปลง `ParseIntError` เป็น
   `String` ทิ้งข้อมูลต้นฉบับ) ให้เป็นรูปแบบที่เก็บ error ต้นฉบับไว้จริงแบบหัวข้อ 30.5 คือ `InvalidNumber { field:
   String, source: ParseIntError }` จากนั้น (ก) เขียน/แก้ `impl From<ParseIntError> for ConfigError` ให้เข้ากับ
   โครงสร้างใหม่ (ข) override `source()` ให้คืน `Some(source)` สำหรับ variant นี้ และ (ค) เขียนฟังก์ชันทดสอบที่
   เรียก `?` เพื่อแปลง `ParseIntError` เป็น `ConfigError` โดยอัตโนมัติ แล้วพิมพ์ error chain ด้วยฟังก์ชันแบบ
   `print_error_chain` จากหัวข้อ 30.5 (hint: สังเกตว่า `impl From` เวอร์ชันใหม่นี้ยังคง "ไม่รู้ชื่อฟิลด์" อยู่ดี
   เพราะ `From::from` รับพารามิเตอร์เดียว — ถ้าต้องการชื่อฟิลด์ด้วย ต้องใช้ `.map_err()` เขียน context เพิ่ม
   ตรงจุดที่เรียก แทนที่จะพึ่ง `?`+`From` ล้วน ๆ เหมือนที่ฟังก์ชัน `parse_field`/`get_field` ในหัวข้อ 30.5/30.10
   ทำ — ลองเปรียบเทียบทั้งสองวิธีดูว่าต่างกันตรงไหน)

3. **[ยาก]** ออกแบบระบบใหม่ทั้งหมดสำหรับโดเมนที่ต่างจากตัวอย่างในบทนี้ (เช่น ระบบอ่านไฟล์สต๊อกสินค้าที่เก็บเป็น
   บรรทัด `sku,quantity` คั่นด้วย comma) โดยต้องมี **wrapper error enum** ระดับบนสุด (เช่น `InventoryError`) ที่
   ห่อ error ได้อย่างน้อย 2 แหล่ง: I/O error จากการอ่านไฟล์ (`std::io::Error`) และ parse/validation error ของ
   ตัวเอง (เช่น `sku` ว่างเปล่า, `quantity` parse ไม่ได้, หรือ `quantity` ติดลบ) ต้อง implement `Display`,
   `Error` (พร้อม `source()` ที่ถูกต้องทุก variant), และ `From` ให้ครบทั้งสองแหล่ง แล้วเขียนฟังก์ชันที่ไล่พิมพ์
   error chain แบบเต็มเมื่อล้มเหลว ทดสอบด้วยอย่างน้อย 3 กรณีที่ทำให้ล้มเหลวคนละแบบ (hint: เลียนแบบโครงสร้างของ
   `AppError`/`ConfigError` ในหัวข้อ 30.6 และ 30.10 แต่เปลี่ยนโดเมนให้ไม่เหมือนกันตรง ๆ เพื่อฝึกออกแบบเอง ไม่ใช่
   copy โค้ดมาปรับชื่อ)

4. **[ยาก/ประยุกต์ใช้งานจริง]** จำลองสถานการณ์ "library กับ downstream" แบบหัวข้อ 30.7 ด้วยตัวเอง: สร้างไฟล์
   `bank_lib.rs` ที่นิยาม `pub enum WithdrawError` (มีอย่างน้อย 2 variant เช่น `InsufficientFunds` และ
   `AccountFrozen`) พร้อม `#[non_exhaustive]`, `impl Display`, และ `impl Error` ให้ครบ แล้ว compile เป็น
   `--crate-type lib` จากนั้นสร้างไฟล์ที่สอง `bank_app.rs` ที่ `extern crate bank_lib;` และเขียนฟังก์ชันที่
   `match` บน `WithdrawError` **ครบทุก variant พร้อม `_ =>`** compile ด้วย `--extern bank_lib=...` ให้ผ่าน แล้ว
   ลองกลับไปเพิ่ม variant ที่ 3 เข้าไปใน `bank_lib.rs` (เช่น `DailyLimitExceeded`) แล้ว **compile lib ใหม่อย่าง
   เดียว** (ไม่แก้ `bank_app.rs`) ยืนยันว่า `bank_app.rs` เดิมยัง compile ผ่านได้โดยไม่ต้องแก้อะไรเลย (hint:
   ต้อง compile ทีละไฟล์ด้วย `rustc` ตรง ๆ ตามลำดับที่หัวข้อ 30.7 สาธิตไว้ — ทดลองเอา `#[non_exhaustive]` ออก
   แล้วทำซ้ำขั้นตอนเดิม เพื่อดูว่าครั้งนี้ `bank_app.rs` เดิม compile ไม่ผ่านหลังเพิ่ม variant ต่างจากตอนมี
   `#[non_exhaustive]` อย่างไร)

## สรุป

บทนี้พาไปลึกกว่าสิ่งที่ Part 12 ทิ้งไว้ตรง ๆ ว่า "จะเจาะลึกใน Part 30" — เราเริ่มจากข้อจำกัดที่แท้จริงของ
`Box<dyn Error>`: มันทำให้ `?` ใช้งานสะดวกมาก แต่**ลบข้อมูลชนิดที่แน่นอนของ error ออกไปจากมุมมองของผู้เรียก**
ทำให้ `match`/แยกกรณีตอบสนองต่างกันไม่ได้โดยตรง (ต้อง `downcast_ref` ซึ่งไม่ idiomatic และไม่มี exhaustiveness
checking) — ทางออกคือสร้าง **custom error enum** ที่ compiler รู้จักโครงสร้างเต็มรูปแบบ ผ่านการ implement
`std::fmt::Display` (ข้อความสำหรับมนุษย์) และ `std::error::Error` (marker trait ที่มี `Debug + Display` เป็น
**supertrait** ตามหลักการจาก Part 21 บวก method เสริม `source()` ที่มี default implementation)

เราทำความเข้าใจกลไกของ `From<T>` ที่ `?` operator ใช้แปลง error ข้ามชนิดกันแบบเป็นทางการ อ่าน error `E0277`
ที่เกิดเมื่อไม่มี `impl From` ที่จำเป็นได้อย่างละเอียดทุกบรรทัด แล้วเรียนรู้ว่า **`source()`** เปิดทางให้สร้าง
**error chain** — การเก็บ error ต้นฉบับไว้เป็นข้อมูล (ไม่แปลงทิ้งเป็น `String`) แล้วให้ผู้เรียกไล่สาเหตุที่แท้จริง
ได้ทีละชั้นจนถึงต้นตอ ต่อด้วยการออกแบบ **wrapper error enum** ที่ห่อ error จากหลายแหล่ง (เช่น I/O และ parse
error) ไว้ในที่เดียวพร้อม `impl From` ให้แต่ละแหล่ง และ `#[non_exhaustive]` ที่ป้องกัน breaking change เมื่อ
library ต้องเพิ่มสาเหตุความล้มเหลวใหม่ในอนาคต ปิดท้ายด้วยหลักการเขียน `Display` ที่ดี การเปรียบเทียบ 3 แนวทาง
(`Box<dyn Error>` vs hand-rolled enum vs พิมพ์เขียวของ `thiserror`) และตัวอย่าง config parser แบบเต็มที่ผสาน
ทุกองค์ประกอบเข้าด้วยกัน

ทุกอย่างที่เขียนในบทนี้ — `enum`, `impl Display`, `impl Error`, `impl From` หลายตัว, `source()` ที่ต้อง `match`
เอง — คือ **โค้ดที่ crate `thiserror` จะ generate ให้อัตโนมัติจาก attribute เพียงไม่กี่บรรทัด** และ crate
`anyhow` จะให้ทางเลือกที่สะดวกกว่า `Box<dyn Error>` (พร้อม `.context()` แนบข้อความระหว่าง propagate) สำหรับ
โค้ดระดับ application ที่ไม่ต้องแยกกรณี error ละเอียดเท่า library — ใน **Part 31** เราจะเอา `ConfigError`/
`AppError` ตัวเดิมจากบทนี้ไปเขียนใหม่ด้วย `#[derive(thiserror::Error)]` เพื่อให้เห็น **บรรทัดต่อบรรทัด** ว่า
attribute ไหนแทนที่โค้ดส่วนไหนที่เราเขียนด้วยมือในบทนี้ พร้อมแนะนำ `anyhow::Error` และ `.context()` สำหรับ
โค้ดระดับ application — เมื่อเข้าใจกลไกพื้นฐานจากบทนี้แล้ว การเรียนรู้ทั้งสอง crate นั้นจะเป็นเรื่องของ "การลด
boilerplate ที่รู้อยู่แล้วว่าทำอะไร" มากกว่า "ท่องจำ syntax ใหม่ที่ไม่รู้ว่าทำงานอย่างไร"

---

**Part ก่อนหน้า:** [Smart Pointers: Weak<T>, Cow<T>](part-029-smart-pointers-weak-cow.md) | **Part ถัดไป:** [thiserror และ anyhow ในโปรเจกต์จริง](part-031-thiserror-anyhow.md)
