# Part 12: Result<T,E> และ Error Handling เบื้องต้น

> โมดูล: พื้นฐานภาษา Rust (Core Language Fundamentals) | ระดับ: พื้นฐาน | เวลาโดยประมาณ: 180 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายปรัชญาการจัดการ error ของ Rust ที่ว่า "ฟังก์ชันที่ล้มเหลวได้ต้องประกาศไว้ใน return type" และเปรียบเทียบได้ว่า
  แนวทางนี้ต่างจาก exception-based error handling ของ Python/Java อย่างไร พร้อมบอกได้ว่าทำไมความต่างนี้ถึงสำคัญ
- นิยามและอธิบาย enum `Result<T, E>` ที่เป็น built-in ของ Rust ได้อย่างเป็นทางการ ใช้งานร่วมกับ `match` แบบเต็มรูปแบบ
  บนฟังก์ชันที่ล้มเหลวได้จริง เช่น `"42".parse::<i32>()`
- แยกแยะ **recoverable error** (ใช้ `Result`) กับ **unrecoverable error** (ใช้ `panic!`) ได้ถูกสถานการณ์ พร้อมอธิบาย
  หลักการ "โปรแกรมอยู่ในสถานะที่ไม่ควรเกิดขึ้นได้" ที่อยู่เบื้องหลังการเลือกใช้ `panic!`
- ใช้ `.unwrap()`/`.expect()` บน `Result` ได้อย่างเข้าใจว่าเมื่อไหร่ "ยอมรับได้" เมื่อไหร่ "ไม่ควรใช้ใน production"
  และอ่าน panic message ที่มี `Debug` ของค่า `Err` แทรกอยู่ได้
- ใช้ `?` operator แทนการเขียน `match` ซ้อนกันแบบยาว ๆ ได้อย่างคล่องแคล่ว และอธิบายได้อย่างละเอียดว่า `?` desugar
  เป็นโค้ดอะไรจริง ๆ ทำไมมันไม่มี runtime overhead และไม่เหมือนกับกลไก exception/stack unwinding ของภาษาอื่น
- เข้าใจบทบาทของ trait `From` ที่ `?` operator ใช้แปลงชนิด error โดยอัตโนมัติ แก้ปัญหา error type ไม่ตรงกันได้ทั้งด้วย
  การเขียน `impl From` เอง และด้วย `Box<dyn Error>` แบบ pragmatic พร้อมเขียน `fn main() -> Result<(), Box<dyn Error>>`
  ได้อย่างถูกต้อง
- ใช้ combinator ของ `Result` ที่คู่กับของ `Option` ที่เรียนใน Part 11 ได้ถูกสถานการณ์ (`.map()`, `.map_err()`,
  `.and_then()`, `.unwrap_or()`, `.unwrap_or_else()`, `.ok()`, `.is_ok()`/`.is_err()`) และออกแบบการ propagate error
  ข้ามหลายฟังก์ชันในระบบจริงได้อย่างสะอาด อ่านง่าย

## ความรู้ที่ต้องมีมาก่อน

- **Part 11 (Option<T> และ Null Safety)**: บทนี้ต่อยอดจาก Part 11 โดยตรง — คุณควรคุ้นเคยกับ `.unwrap()`/`.expect()`
  ของ `Option<T>` มาก่อนแล้ว เพราะเราจะเทียบพฤติกรรมของทั้งสอง method นี้บน `Result<T, E>` ว่าเหมือนและต่างจาก
  `Option<T>` อย่างไร (โดยเฉพาะเรื่อง panic message ที่ `Result` มีข้อมูล `Err` ติดมาด้วย ขณะที่ `Option` ไม่มีข้อมูล
  อะไรเลยใน `None`) และคุณควรรู้จัก `.map()`/`.and_then()` ของ `Option` มาก่อนแล้ว เพราะ `Result` มี method ชุดเดียวกัน
  ที่ทำงานคล้ายกันมาก รวมถึง `.ok_or()`/`.ok_or_else()` ที่ Part 11 ใช้แปลง `Option<T>` เป็น `Result<T, E>` ซึ่งเป็น
  "สะพาน" เชื่อมสองบทนี้เข้าด้วยกัน
- **Part 6 (Ownership)**: combinator ของ `Result` หลายตัว (เช่น `.map()`, `.and_then()`, `.unwrap_or()`, `.ok()`)
  รับ `self` แบบ **take ownership** (กินค่าเข้าไปตรง ๆ ไม่ยืม) เหมือน method ที่คุณเจอมาแล้วใน Part 6-7 — ถ้าจะใช้
  ค่าเดิมต่อหลังเรียก method เหล่านี้ ต้อง `.clone()` ก่อนเสมอ (จะเห็นตัวอย่างจริงในหัวข้อ 12.9)
- **Part 9 (Structs)** และ **Part 10 (Enums และ Pattern Matching)**: `Result<T, E>` **คือ enum ธรรมดา** ที่มี
  2 variant คือ `Ok(T)` และ `Err(E)` — ทุกอย่างที่คุณเรียนเรื่อง `match`, exhaustiveness checking, `if let`,
  destructuring ใน Part 10 ใช้กับ `Result` ได้ตรง ๆ ทุกประการ ไม่มีอะไรพิเศษเพิ่มเติม บทนี้จะไม่สอน `match` ซ้ำ
  ตั้งแต่ต้น แต่จะอ้างอิงกลับไปที่ Part 10 ตลอด
- **Part 8 (Slices)**: ตัวอย่างการ parse string และตัวอย่าง config parser ท้ายบทใช้ `&str`, `.split()`, และ
  slice indexing ที่เรียนมาจาก Part 8

ถ้าคุณยังไม่แน่นเรื่อง `match`/enum จาก Part 10 แนะนำให้ทวนก่อน เพราะ `Result<T, E>` ไม่ใช่ concept ใหม่ในเชิง
โครงสร้างข้อมูล — มันคือการ "เอา enum ที่คุณรู้จักแล้วมาใช้แก้ปัญหา error handling" เท่านั้นเอง สิ่งที่ใหม่จริง ๆ ใน
บทนี้คือ **`?` operator** และ **ปรัชญาการออกแบบ error handling ของ Rust โดยรวม** ซึ่งเป็นเนื้อหาหลักที่บทนี้ให้
ความสำคัญที่สุด

## เนื้อหา

### 12.1 ปัญหาที่ Rust พยายามแก้: Error ที่ "มองไม่เห็น" ในภาษาที่ใช้ Exception

ก่อนจะเข้าเรื่อง `Result<T, E>` เราต้องเข้าใจก่อนว่า Rust กำลังแก้ปัญหาอะไร ลองนึกภาพว่าคุณกำลังเขียนโปรแกรมที่อ่านไฟล์
config มาแปลงเป็นตัวเลข port ในภาษาที่ใช้ exception อย่าง Python:

```python
# ภาษา Python — ตัวอย่างเปรียบเทียบเท่านั้น ไม่ใช่ Rust
def parse_port(raw: str) -> int:
    return int(raw)  # อาจ raise ValueError ถ้า raw ไม่ใช่ตัวเลข

def start_server(config_line: str):
    port = parse_port(config_line)  # เรียกฟังก์ชันที่ "อาจ" raise exception
    print(f"เริ่มเซิร์ฟเวอร์ที่ port {port}")

start_server("not-a-port")
```

โค้ด Python นี้ **ไม่มีอะไรในตัว signature ของ `parse_port`** ที่บอกผู้เรียกเลยว่า "ฟังก์ชันนี้อาจล้มเหลว" — คุณต้อง
อ่าน documentation, อ่าน source code ข้างใน, หรือรอให้มันพังตอน runtime เท่านั้นถึงจะรู้ว่า `int(raw)` โยน
`ValueError` ได้ ถ้า `start_server` ไม่ได้ครอบ `parse_port(config_line)` ด้วย `try/except` โปรแกรมจะ crash ทันที
ด้วย traceback ยาว ๆ และปัญหาที่ร้ายแรงกว่านั้นคือ **ไม่มีอะไรบังคับให้คุณต้อง handle มันเลย** — คุณลืม `try/except`
ได้ง่าย ๆ โดยที่ compiler (หรือในที่นี้คือ interpreter) ไม่มีทางเตือนคุณก่อนรันจริง

ภาษา Java พยายามแก้ปัญหานี้บางส่วนด้วย **checked exceptions** ที่ต้องประกาศ `throws` ใน signature:

```java
// ภาษา Java — ตัวอย่างเปรียบเทียบเท่านั้น ไม่ใช่ Rust
int parsePort(String raw) throws NumberFormatException {
    return Integer.parseInt(raw);
}
```

แต่ `NumberFormatException` เป็น **unchecked exception** (subclass ของ `RuntimeException`) ซึ่ง Java **ไม่บังคับ**
ให้ประกาศ `throws` หรือ handle เลยแม้แต่นิดเดียว — มันแทบจะเป็นสถานการณ์เดียวกับ Python ทุกประการ: error ที่เกิดขึ้น
บ่อยที่สุดในโค้ดจริง (แปลง string เป็นตัวเลขผิด) กลับเป็นชนิดที่ compiler ไม่บังคับ handle เลย ส่วน checked exception
ที่ Java บังคับจริง ๆ (เช่น `IOException`) ก็มีปัญหาคนละแบบ: นักพัฒนา Java จำนวนมากแก้ปัญหาด้วยการเขียน
`catch (IOException e) { }` (catch เปล่า ๆ ไม่ทำอะไร) เพียงเพื่อให้ compile ผ่าน ซึ่งแย่กว่าไม่มี error handling
เลยด้วยซ้ำ เพราะมัน **ซ่อน** ว่ามี error เกิดขึ้นจริง

#### ปัญหาแกนกลาง: exception คือ "control flow ที่แยกออกจาก type system"

จุดที่ทั้ง Python และ Java (รวมถึง C++, JavaScript, Ruby, และภาษาที่ใช้ exception ส่วนใหญ่) มีปัญหาร่วมกันคือ:
**การที่ฟังก์ชันหนึ่งจะ "ล้มเหลว" ได้หรือไม่ ไม่ได้เป็นส่วนหนึ่งของ type ที่ฟังก์ชันนั้นประกาศไว้เลย** ฟังก์ชัน
`int parse_port(str raw)` (ไม่ว่าจะภาษาไหน) มี "รูปร่าง" เดียวกันไม่ว่ามันจะ throw exception ได้หรือไม่ได้ก็ตาม —
คุณต้องพึ่งพา documentation, ความจำ, หรือการอ่าน source code เพื่อรู้ความจริงนี้ และที่แย่กว่านั้นคือ:

1. **exception ที่ไม่ถูก catch จะ "เดินทาง" ขึ้นไปเรื่อย ๆ ผ่านหลาย stack frame โดยอัตโนมัติ** จนกว่าจะเจอ
   `try/catch` ที่ครอบมันไว้ หรือไปถึง top-level แล้วทำให้โปรแกรม crash — เส้นทางที่ exception เดินทางไปนี้
   **ไม่ปรากฏอยู่ในโค้ดที่อ่านตรงหน้าเลย** คุณอ่านฟังก์ชัน A ที่เรียก B ที่เรียก C ไม่มีทางรู้จากการอ่านฟังก์ชัน A
   อย่างเดียวว่า "ถ้า C พัง จะเกิดอะไรขึ้นกับ A"
2. **การเพิ่ม exception type ใหม่ในฟังก์ชันที่อยู่ลึกในระบบ ไม่บังคับให้ผู้เรียกทุกจุดต้อง handle เพิ่ม** (ยกเว้น
   checked exception ของ Java ที่ก็มีข้อเสียของตัวเองตามที่อธิบายไปแล้ว) ต่างจาก Rust ที่การเปลี่ยน return type
   ของฟังก์ชันจะถูก compiler ตรวจสอบทุกจุดที่เรียกใช้ทันที (คล้ายกับที่ Part 10 อธิบายไว้เรื่องการเพิ่ม enum variant
   ใหม่)

Rust เลือกแนวทางที่ต่างไปอย่างสิ้นเชิง โดยวางอยู่บนหลักการเดียว ที่เราจะย้ำตลอดทั้งบทนี้:

> **ฟังก์ชันที่ "ล้มเหลวได้" ต้องประกาศความจริงนี้ไว้ใน return type ของมันเอง** ไม่มีทางซ่อนมันไว้ได้ ผู้เรียกฟังก์ชัน
> จะเห็น "สัญญา" นี้ทันทีจากการอ่าน signature เพียงบรรทัดเดียว โดยไม่ต้องพึ่งพา documentation หรือความจำเลย

ฟังก์ชัน `fn parse_port(raw: &str) -> Result<u16, ParseIntError>` บอกทุกอย่างที่ผู้เรียกต้องรู้ในบรรทัดเดียว:
"ฟังก์ชันนี้ **อาจ** คืนค่า `u16` ที่ถูกต้อง (`Ok`) **หรือ** ล้มเหลวพร้อมเหตุผลที่เป็น `ParseIntError` (`Err`)" และ
ที่สำคัญที่สุดคือ **compiler จะไม่ยอมให้คุณเอาค่า `u16` ออกมาใช้ตรง ๆ โดยไม่จัดการกับความเป็นไปได้ของ `Err` ก่อน**
เราจะเห็นกลไกที่บังคับสิ่งนี้อย่างละเอียดตลอดบทนี้

### 12.2 นิยาม `Result<T, E>` อย่างเป็นทางการ

`Result<T, E>` คือ enum ที่ built-in อยู่ใน Rust prelude (ใช้ได้ทันทีทุกไฟล์โดยไม่ต้อง `use` อะไรเพิ่มเติม เหมือนกับ
`Option<T>` ที่คุณเรียนมาจาก Part 11) นิยามจริงของมันใน standard library มีรูปร่างง่าย ๆ แบบนี้:

```
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

(นี่ไม่ใช่โค้ดที่ต้องพิมพ์เอง — มันมีอยู่แล้วใน `std`/`core` ยกมาแสดงเพื่อให้เห็นโครงสร้างจริงเท่านั้น)

สังเกตว่านี่คือ **generic enum ที่มี type parameter สองตัว** คือ `T` และ `E`:

- **`T`** คือชนิดของค่าที่ได้เมื่อ**สำเร็จ** (variant `Ok`) — ย่อมาจาก "Type" ของค่าปกติที่ต้องการ
- **`E`** คือชนิดของค่าที่ได้เมื่อ**ล้มเหลว** (variant `Err`) — ย่อมาจาก "Error"

ต่างจาก `Option<T>` ที่มี type parameter เดียว (`Some(T)` / `None` — กรณีไม่มีค่าไม่ต้องมีข้อมูลอะไรเพิ่มเติม)
`Result<T, E>` มี **สอง** type parameter เพราะกรณีล้มเหลวมักมีข้อมูลสำคัญที่อยากรู้เสมอ (เช่น "ล้มเหลว **เพราะ**
อะไร") ซึ่งต่างจาก `None` ของ `Option` ที่ไม่มีข้อมูลอะไรให้เก็บเลย (ไม่มีค่า ก็ไม่มีอะไรต้องอธิบายเพิ่ม) นี่คือ
ความแตกต่างเชิง design ที่สำคัญที่สุดระหว่างสองชนิดข้อมูลนี้ และเป็นเกณฑ์ที่ใช้เลือกว่าจะใช้ตัวไหน:

> **ใช้ `Option<T>` เมื่อ "ไม่มีค่า" ก็เพียงพอต่อการอธิบายสถานการณ์แล้ว** (เช่น หาไม่เจอ, ไม่มีข้อมูล ณ ตำแหน่งนี้)
> **ใช้ `Result<T, E>` เมื่อ "ล้มเหลว" ต้องมีเหตุผลกำกับด้วยเสมอ** (เช่น ล้มเหลวเพราะ input ผิดรูปแบบ, ล้มเหลวเพราะ
> เชื่อมต่อเครือข่ายไม่ได้, ล้มเหลวเพราะสิทธิ์ไม่พอ) — ผู้เรียกจะต้องใช้เหตุผลนั้นตัดสินใจต่อว่าจะทำอะไร

มาดูการใช้งานพื้นฐานที่สุด:

```rust
fn main() {
    let ok_value: Result<i32, String> = Ok(42);
    let err_value: Result<i32, String> = Err("something went wrong".to_string());

    println!("{:?}", ok_value);
    println!("{:?}", err_value);
}
```

ผลลัพธ์:

```
Ok(42)
Err("something went wrong")
```

**อธิบายโค้ดทีละส่วน:**

- `let ok_value: Result<i32, String> = Ok(42);` — `Ok(42)` สร้างค่า variant `Ok` ที่มีข้อมูลติดตัวเป็น `42`
  (type `i32`) เราต้องระบุ type annotation เต็ม `Result<i32, String>` เพราะจากแค่ `Ok(42)` เพียงอย่างเดียว
  compiler รู้แค่ว่า `T = i32` แต่ยังไม่รู้ว่า `E` (ชนิดของ error ที่ *อาจจะ* เกิดในบริบทอื่น) ควรเป็นอะไร ถ้าเรา
  ใช้ `ok_value` ต่อในทางที่บอก type ของ `E` ชัดเจน (เช่นส่งเข้าฟังก์ชันที่รับ `Result<i32, String>` โดยเฉพาะ)
  บางครั้งก็ไม่ต้องเขียน annotation เอง (type inference จะเดาให้ได้) แต่ในตัวอย่างแบบเดี่ยว ๆ นี้ต้องระบุเอง
- `Err("something went wrong".to_string())` — สร้างค่า variant `Err` ที่มีข้อมูลติดตัวเป็น `String` เดียวกับที่
  ประกาศไว้ใน type parameter `E`
- `{:?}` — ทั้ง `Result<T, E>` เองก็ implement `Debug` ให้อัตโนมัติตราบใดที่ทั้ง `T` และ `E` implement `Debug`
  (ในที่นี้ `i32` และ `String` implement `Debug` ทั้งคู่อยู่แล้วโดย default)

เหมือนกับที่ Part 10 อธิบายไว้เรื่อง enum ทั่วไป: `Ok` และ `Err` **ไม่ใช่ type แยกจากกัน** ทั้งคู่เป็นแค่ variant ของ
type เดียวคือ `Result<T, E>` — ตัวแปร `ok_value` และ `err_value` ในตัวอย่างข้างบนมี type เดียวกันคือ
`Result<i32, String>` ทั้งคู่ ต่างกันแค่ "อยู่ใน variant ไหน" เท่านั้น ทำให้คุณส่งค่าทั้งสองผ่านฟังก์ชันเดียวกัน หรือ
เก็บใน `Vec<Result<i32, String>>` เดียวกันได้อย่างอิสระ

### 12.3 ตัวอย่างจริง: ฟังก์ชันที่ล้มเหลวได้ตามธรรมชาติ — `.parse::<i32>()`

ตัวอย่างที่ชัดที่สุดของฟังก์ชันที่ "ล้มเหลวได้ตามปกติของการทำงาน" คือการแปลง string เป็นตัวเลข — ข้อมูลที่มาจากผู้ใช้
หรือไฟล์ config ไม่มีทางรู้ก่อนว่าจะเป็นตัวเลขที่ถูกต้องเสมอไป Rust จึงออกแบบให้ `str::parse::<T>()` คืนค่าเป็น
`Result<T, T::Err>` เสมอ (ไม่ใช่คืนค่าตัวเลขตรง ๆ แล้ว panic ถ้าผิด):

```rust
fn main() {
    let input = "42";
    let parsed: Result<i32, std::num::ParseIntError> = input.parse::<i32>();

    match parsed {
        Ok(number) => println!("แปลงสำเร็จ: {number}"),
        Err(e) => println!("แปลงไม่สำเร็จ: {e}"),
    }

    let bad_input = "สี่สิบสอง";
    let parsed2 = bad_input.parse::<i32>();
    match parsed2 {
        Ok(number) => println!("แปลงสำเร็จ: {number}"),
        Err(e) => println!("แปลงไม่สำเร็จ: {e}"),
    }
}
```

ผลลัพธ์:

```
แปลงสำเร็จ: 42
แปลงไม่สำเร็จ: invalid digit found in string
```

**อธิบายโค้ดทีละส่วน:**

- `input.parse::<i32>()` — `::<i32>` คือ **turbofish syntax** (ที่คุณอาจเคยเห็นผ่านมาบ้างแล้ว) บอก compiler ว่า
  ต้องการ parse เป็น type `i32` โดยเฉพาะ ผลลัพธ์ที่ได้คือ `Result<i32, ParseIntError>` — สังเกตว่าชนิดของ `Err`
  (`ParseIntError`) **ไม่ใช่ `String` ธรรมดา** แต่เป็น struct เฉพาะที่ standard library ออกแบบมาสำหรับ error จาก
  การ parse ตัวเลขโดยเฉพาะ ซึ่งเป็นแนวทางที่ดีกว่า `String` ล้วน ๆ (จะอธิบายเหตุผลเชิงลึกในหัวข้อ 12.7)
- `Err(e) => println!("แปลงไม่สำเร็จ: {e}")` — `ParseIntError` implement trait `Display` ให้แล้ว จึงใช้ `{e}`
  ใน `println!` ได้ตรง ๆ (แสดงเป็นข้อความอ่านง่าย `"invalid digit found in string"`) ต่างจาก `{:?}` ที่ต้องใช้
  `Debug` และมักแสดงรายละเอียด structure ภายในแบบดิบกว่า
- `match parsed { Ok(number) => ..., Err(e) => ... }` — นี่คือ `match` แบบเต็มรูปแบบเหมือนที่เรียนใน Part 10
  ทุกประการ เพราะ `Result<T, E>` ก็เป็น enum ธรรมดา — compiler จะบังคับ **exhaustiveness checking** เช่นเดียวกับ
  enum อื่น ๆ คือต้อง handle ทั้ง `Ok` และ `Err` ให้ครบ (หรือมี `_`/`other` ปิดท้าย) ถ้าลืม arm ใดไป จะได้ error
  `E0004` แบบเดียวกับที่ Part 10 อธิบายไว้ทุกประการ — ลองดูตัวอย่าง:

```rust
fn describe(parsed: Result<i32, std::num::ParseIntError>) -> String {
    match parsed {
        Ok(number) => format!("ได้ตัวเลข {number}"),
        // ลืม handle Err ไปเลย!
    }
}

fn main() {}
```

โค้ดนี้ compile ไม่ผ่าน เพราะ `Result` มีแค่ 2 variant แต่ `match` ครอบคลุมแค่ 1:

```
error[E0004]: non-exhaustive patterns: `Err(_)` not covered
```

(รูปแบบ error message นี้เหมือนกับ `E0004` ที่อธิบายไว้เต็ม ๆ แล้วใน Part 10 หัวข้อ "Exhaustiveness Checking" —
ระบุ pattern ที่ไม่ครอบคลุม พร้อม `help` เสนอวิธีแก้ให้เสมอ) นี่คือจุดสำคัญที่สุดที่ต้องจำไว้ตลอดบทนี้: **`Result`
ไม่ได้มีกลไกพิเศษอะไรที่ต่างจาก enum ทั่วไปเลย มันปลอดภัยเพราะมันเป็น enum ที่ compiler บังคับ exhaustiveness
checking เหมือนกับทุก enum ที่คุณสร้างเอง** — ความปลอดภัยของ error handling ใน Rust มาจากกลไกเดียวกับที่ทำให้
`match` บน enum ของคุณเองปลอดภัย ไม่ใช่ feature พิเศษแยกต่างหาก

#### `ParseIntError`: ตัวอย่างของการออกแบบ error type ที่ดี

ทำไม `parse::<i32>()` ไม่คืน `Err(String)` ตรง ๆ ไปเลยล่ะ ทำไมต้องมี struct `ParseIntError` แยก? เหตุผลคือ
**การเก็บ error เป็น structured type แทน string ทำให้ผู้เรียกตัดสินใจต่อได้แม่นยำกว่า** ลองดูว่า `ParseIntError`
เก็บอะไรบ้างจริง ๆ ผ่าน `{:?}`:

```rust
fn main() {
    let result = "abc".parse::<i32>();
    println!("{:?}", result);

    let overflow_result = "99999999999999999999".parse::<i32>();
    println!("{:?}", overflow_result);

    let empty_result = "".parse::<i32>();
    println!("{:?}", empty_result);
}
```

ผลลัพธ์:

```
Err(ParseIntError { kind: InvalidDigit })
Err(ParseIntError { kind: PosOverflow })
Err(ParseIntError { kind: Empty })
```

สังเกตว่า `ParseIntError` มี field ภายใน (`kind`) ที่แยกแยะ**สาเหตุที่แท้จริง**ของความล้มเหลวออกจากกันชัดเจน —
"ไม่ใช่ตัวเลข" (`InvalidDigit`), "ตัวเลขใหญ่เกินขนาดที่ type รับได้" (`PosOverflow`), และ "string ว่างเปล่า"
(`Empty`) ล้วนเป็นสาเหตุคนละแบบที่โปรแกรมอาจต้องการตอบสนองต่างกัน (เช่น ถ้าเป็น `Empty` อาจแจ้งผู้ใช้ว่า "กรุณากรอก
ข้อมูล" แต่ถ้าเป็น `InvalidDigit` อาจแจ้งว่า "กรุณากรอกตัวเลขเท่านั้น") ถ้าเก็บเป็น `String` ธรรมดาตั้งแต่แรก ผู้เรียก
จะต้องมา parse ข้อความ string (`if message.contains("overflow")`) ซึ่งเปราะบางมาก (ข้อความอาจเปลี่ยนได้ในเวอร์ชัน
ใหม่ของ library) เราจะเจาะลึกเรื่องการออกแบบ error type ของตัวเองแบบนี้อย่างเต็มรูปแบบใน **Part 30** แต่ตอนนี้ควร
จำหลักการไว้ก่อน: **error type ที่ดีควรเป็น structured data ที่ผู้เรียกตรวจสอบและตัดสินใจต่อได้ ไม่ใช่แค่ข้อความ
สำหรับมนุษย์อ่านเท่านั้น**

### 12.4 Recoverable vs Unrecoverable Errors: `Result` กับ `panic!`

Rust แบ่ง error ออกเป็น 2 ประเภทใหญ่อย่างชัดเจน ซึ่งเป็นการแบ่งที่สำคัญที่สุดในการออกแบบระบบ error handling ทั้งหมด
ของภาษา:

| ประเภท | เครื่องมือ | ความหมาย | ตัวอย่าง |
|---|---|---|---|
| **Recoverable** (กู้คืนได้) | `Result<T, E>` | เหตุการณ์ที่**เป็นส่วนหนึ่งของการทำงานปกติ** โปรแกรมควรจับและตอบสนองต่อมันได้ ไม่ใช่บั๊ก | ผู้ใช้กรอกข้อมูลผิด, ไฟล์ไม่พบ, การเชื่อมต่อเครือข่ายขาด, ยอดเงินไม่พอ, สิทธิ์การเข้าถึงไม่พอ |
| **Unrecoverable** (กู้คืนไม่ได้/ไม่ควรกู้คืน) | `panic!` | โปรแกรม**อยู่ในสถานะที่ไม่ควรเกิดขึ้นได้เลย** ถ้าเกิด แปลว่ามีบั๊กในโค้ด ไม่ใช่สถานการณ์ปกติที่ควรวางแผนรับมือ | เข้าถึง array เกินขอบเขต (index ผิดที่ควรถูกเช็คมาก่อนแล้ว), หารด้วยศูนย์ที่ไม่ควรเกิด, invariant ภายในของโครงสร้างข้อมูลถูกละเมิด, contract ของฟังก์ชันถูกละเมิดโดยผู้เรียก |

หลักการตัดสินใจที่ใช้ได้จริงคือถามตัวเองว่า: **"นี่คือสิ่งที่คาดว่าจะเกิดขึ้นได้เป็นปกติในระหว่างที่โปรแกรมทำงาน
ถูกต้องหรือไม่?"** ถ้าคำตอบคือ "ใช่ นี่เป็นเรื่องปกติที่ต้องรับมือ" ให้ใช้ `Result` ถ้าคำตอบคือ "ไม่ ถ้าเกิดแบบนี้ขึ้น
แปลว่าโค้ดมีบั๊ก หรือมีใครทำผิดกฎที่ตกลงกันไว้" ให้ใช้ `panic!`

ลองดูตัวอย่างที่แยกสองกรณีนี้ให้เห็นภาพชัด — ระบบธนาคารง่าย ๆ ที่มีทั้งสองสถานการณ์อยู่ในไฟล์เดียวกัน:

```rust
// ตัวอย่าง: ฟังก์ชันที่ "ล้มเหลวได้ตามปกติของการทำงาน" ควรคืน Result
fn withdraw(balance: u64, amount: u64) -> Result<u64, String> {
    if amount > balance {
        return Err(format!(
            "ยอดเงินไม่พอ: มี {balance} บาท แต่ขอถอน {amount} บาท"
        ));
    }
    Ok(balance - amount)
}

// ตัวอย่าง: ฟังก์ชันที่ควร panic! เพราะเป็น "การละเมิดสัญญาของโปรแกรม" (bug) ไม่ใช่เหตุการณ์ปกติ
fn get_element_at(items: &[i32], index: usize) -> i32 {
    // สมมติว่าเราการันตีไว้ก่อนแล้วว่า index ต้องถูกต้องเสมอ (invariant ของฟังก์ชันนี้)
    // ถ้า index ผิด แสดงว่าโค้ดผู้เรียกมีบั๊ก ไม่ใช่ "ผู้ใช้กรอกข้อมูลผิด"
    items[index]
}

fn main() {
    match withdraw(1000, 300) {
        Ok(remaining) => println!("ถอนสำเร็จ เหลือ {remaining} บาท"),
        Err(e) => println!("ถอนไม่สำเร็จ: {e}"),
    }

    match withdraw(1000, 5000) {
        Ok(remaining) => println!("ถอนสำเร็จ เหลือ {remaining} บาท"),
        Err(e) => println!("ถอนไม่สำเร็จ: {e}"),
    }

    let items = vec![10, 20, 30];
    println!("สมาชิกตำแหน่งที่ 1: {}", get_element_at(&items, 1));
}
```

ผลลัพธ์:

```
ถอนสำเร็จ เหลือ 700 บาท
ถอนไม่สำเร็จ: ยอดเงินไม่พอ: มี 1000 บาท แต่ขอถอน 5000 บาท
สมาชิกตำแหน่งที่ 1: 20
```

**สังเกตความต่างของทั้งสองฟังก์ชัน:**

- `withdraw` คืนค่ายอดเงินไม่พอเป็น `Result` เพราะ **"ผู้ใช้พยายามถอนเงินเกินยอด" เป็นเหตุการณ์ที่คาดหวังว่าจะเกิดขึ้น
  ได้เป็นปกติ** ธนาคารจริงเจอเหตุการณ์นี้ทุกวัน โปรแกรมต้องมี logic รองรับมันอย่างสุภาพ (แจ้งผู้ใช้ ไม่ใช่ล้มโปรแกรม
  ทั้งระบบ)
- `get_element_at` ไม่มีการเช็คอะไรเลย และปล่อยให้ `items[index]` panic ตรง ๆ ถ้า `index` ผิด เพราะ**เราตั้งสมมติฐาน
  (invariant) ไว้ตั้งแต่ต้นว่าผู้เรียกจะส่ง index ที่ถูกต้องมาเสมอ** — ถ้าโค้ดที่เรียก `get_element_at` ส่ง index
  ผิดมา นั่นคือ**บั๊กของโค้ดผู้เรียก** ไม่ใช่สถานการณ์ปกติที่ควรวางแผนรับมือด้วย `Result` การใส่ `Result` ให้กับ
  ฟังก์ชันแบบนี้จะทำให้ผู้เรียก**ทุกจุด**ต้อง handle `Err` ที่ไม่มีวันเกิดขึ้นจริงถ้าโค้ดไม่มีบั๊ก ซึ่งเป็นภาระที่ไม่
  จำเป็นและซ่อนบั๊กจริงไว้เบื้องหลัง `match` ที่ไม่มีความหมาย

หัวใจของการตัดสินใจนี้คือ **panic เป็นเครื่องมือสำหรับ "หยุดโปรแกรมทันทีเมื่อสมมติฐานพื้นฐานของมันถูกละเมิด"**
เพื่อไม่ให้โปรแกรมเดินหน้าต่อไปด้วยสถานะที่ไม่ถูกต้อง (ซึ่งอาจนำไปสู่ปัญหาที่แย่กว่ามาก เช่น ข้อมูลเสียหายแบบเงียบ ๆ
หรือช่องโหว่ความปลอดภัย) แนวคิดนี้ตรงกับสิ่งที่ Part 3 อธิบายไว้เรื่อง integer overflow และ array index out of
bounds ว่า Rust เลือก **"fail fast and loud"** มากกว่าเดินหน้าต่อไปด้วยข้อมูลที่อาจผิดเพี้ยน — `panic!` คือกลไกที่
implement หลักการนี้โดยตรง

ลองดูอีกตัวอย่างของ panic ที่มาจาก operation พื้นฐานของภาษาเอง (ไม่ใช่ `panic!()` ที่เราเขียนเอง) — การหารด้วยศูนย์:

```rust
fn divide(a: i32, b: i32) -> i32 {
    a / b
}

fn main() {
    println!("{}", divide(10, 2));
    println!("{}", divide(10, 0));
}
```

บรรทัดแรกทำงานได้ปกติ (`5`) แต่บรรทัดที่สอง **panic ทันที**:

```
thread 'main' panicked at src/main.rs:2:5:
attempt to divide by zero
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

Rust เลือกให้การหารด้วยศูนย์ของ integer เป็น panic (ไม่ใช่คืนค่าพิเศษอย่าง `inf`/`NaN` แบบที่ floating-point
ทำได้ตามมาตรฐาน IEEE 754 ที่เรียนใน Part 3) เพราะสำหรับ integer แล้ว **ไม่มีค่าไหนที่สมเหตุสมผลให้คืนแทนได้เลย**
การเดินหน้าต่อไปด้วยค่าที่ไม่มีความหมาย (เช่น คืน `0` หรือค่าสุ่ม) จะซ่อนบั๊กไว้และทำให้ปัญหาไปโป่งแตกที่จุดอื่นซึ่ง
debug ยากกว่ามาก การ panic ทันที ณ จุดที่เกิดปัญหาจริง คือทางเลือกที่ปลอดภัยกว่าเสมอ — ถ้าฟังก์ชัน `divide` ของคุณ
**คาดว่า** `b` อาจเป็นศูนย์ได้ในสถานการณ์ปกติ (เช่นรับค่ามาจากผู้ใช้) นั่นคือสัญญาณว่าคุณควรออกแบบมันให้คืน
`Result<i32, String>` แทน แล้วเช็ค `b == 0` เองก่อนหาร ไม่ใช่ปล่อยให้ panic ตรง ๆ

#### เปรียบเทียบกับภาษาอื่น: exception ที่ควรเป็น panic แต่ไม่ใช่

ภาษาที่ใช้ exception เป็นหลักมักไม่มีการแบ่งแยกสองแนวคิดนี้อย่างชัดเจนในระดับภาษา — ทั้ง "ผู้ใช้กรอกข้อมูลผิด" และ
"index ผิดเพราะบั๊ก" ต่างก็แสดงออกมาเป็น exception เหมือนกันหมด (`ValueError` ทั้งคู่ใน Python, หรือ
`RuntimeException` ทั้งคู่ใน Java) ทำให้ผู้เรียก catch ทุกอย่างปนกันไปในโค้ดเดียว หรือแย่กว่านั้นคือ catch แบบกว้าง
เกินไป (`except Exception:` หรือ `catch (Exception e)`) ที่ดักจับทั้ง error ที่ควรจัดการได้ **และ** บั๊กที่ควร
ปล่อยให้โปรแกรม crash ไปพร้อมกัน — การ catch บั๊กไว้เงียบ ๆ แบบนี้คืออันตรายที่แท้จริง เพราะมันทำให้โปรแกรมเดินหน้า
ต่อไปด้วยสถานะที่เสียหายโดยไม่มีใครรู้ตัว Rust แยกสองแนวคิดนี้ออกจากกันตั้งแต่ระดับภาษา (`Result` vs `panic!`)
ทำให้นักพัฒนาต้องตัดสินใจตั้งแต่ตอนออกแบบว่า "นี่คือ error ที่คาดหวังไว้ หรือคือบั๊ก" ซึ่งเป็นคำถามที่ควรถามตั้งแต่
แรกอยู่แล้ว

### 12.5 `.unwrap()` และ `.expect()` บน `Result`

จาก Part 11 คุณรู้จัก `.unwrap()`/`.expect()` ของ `Option<T>` มาแล้ว — บน `Result<T, E>` ทั้งสอง method ทำงาน
คล้ายกันมาก: **ถ้าเป็น `Ok`, คืนค่าข้างในออกมา; ถ้าเป็น `Err`, panic ทันที** ความต่างที่สำคัญที่สุดจาก `Option` คือ
**panic message ของ `Result::unwrap()`/`.expect()` จะแสดง `Debug` ของค่า `Err` แทรกไว้ด้วย** (เพราะ `Err` มีข้อมูล
ติดตัวอยู่ ขณะที่ `None` ของ `Option` ไม่มีข้อมูลอะไรให้แสดงเลย)

```rust
fn find_user_age(username: &str) -> Result<u32, String> {
    if username == "somchai" {
        Ok(30)
    } else {
        Err(format!("user '{username}' not found"))
    }
}

fn main() {
    let age = find_user_age("somchai").unwrap();
    println!("อายุ: {age}");

    let age2 = find_user_age("unknown_user").unwrap();
    println!("อายุ: {age2}");
}
```

บรรทัดแรกทำงานปกติ (`อายุ: 30`) แต่การเรียก `.unwrap()` บน `Err` ในบรรทัดที่สอง panic ด้วยข้อความ:

```
thread 'main' panicked at src/main.rs:13:46:
called `Result::unwrap()` on an `Err` value: "user 'unknown_user' not found"
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

สังเกตส่วน **`called \`Result::unwrap()\` on an \`Err\` value: "user 'unknown_user' not found"`** — ข้อความ
`"user 'unknown_user' not found"` ที่เห็นคือผลลัพธ์ของการนำค่า `String` ที่อยู่ใน `Err` มาแสดงผ่าน `Debug`
(เห็นได้จาก quote คู่ที่ล้อมข้อความไว้ ซึ่งเป็นวิธีที่ `Debug` ของ `String` แสดงผล) นี่คือสิ่งที่ Part 11 ไม่มีให้เห็น
เพราะ `Option::None` ไม่มีข้อมูลอะไรเก็บไว้เลย panic message ของ `Option::unwrap()` บน `None` จึงสั้นกว่ามาก
(`called \`Option::unwrap()\` on a \`None\` value` โดยไม่มีข้อมูลอะไรต่อท้าย) — `Result::unwrap()` "บอกเหตุผล"
ของความล้มเหลวไว้ในตัว panic message เองโดยอัตโนมัติ ซึ่งมีประโยชน์มากตอน debug เพราะไม่ต้องเดาว่า `Err` เก็บอะไร
ไว้ (ข้อกำหนดเดียวคือ `E` ต้อง implement `Debug` ซึ่งเป็นเงื่อนไขที่ `.unwrap()` บังคับไว้ผ่าน trait bound)

`.expect()` ทำงานเหมือนกันทุกประการ แต่ให้คุณกำหนดข้อความส่วนแรกของ panic เองได้ (แทนข้อความ default
`called \`Result::unwrap()\` on an \`Err\` value:`):

```rust
fn find_user_age(username: &str) -> Result<u32, String> {
    if username == "somchai" {
        Ok(30)
    } else {
        Err(format!("user '{username}' not found"))
    }
}

fn main() {
    let age2 =
        find_user_age("unknown_user").expect("ควรหาผู้ใช้เจอเสมอเพราะเช็ค username มาก่อนแล้ว");
    println!("อายุ: {age2}");
}
```

panic message ที่ได้:

```
thread 'main' panicked at src/main.rs:11:39:
ควรหาผู้ใช้เจอเสมอเพราะเช็ค username มาก่อนแล้ว: "user 'unknown_user' not found"
```

สังเกตรูปแบบ: **`{ข้อความที่คุณกำหนด}: {Debug ของ Err}`** — ข้อความของคุณมาก่อน ตามด้วย `:` แล้วตามด้วยข้อมูลจาก
`Err` เสมอ นี่คือเหตุผลที่ (เหมือนกับที่ Part 11 อาจอธิบายไว้กับ `Option`) **ข้อความใน `.expect()` ควรเขียนในเชิง
"สิ่งที่คุณคาดหวังว่าจะเป็นจริง" ไม่ใช่ "อธิบาย error ที่เกิดขึ้น"** เพราะ error ที่เกิดขึ้นจริงจะถูกแสดงให้เห็น
อยู่แล้วโดยอัตโนมัติต่อจาก `:` — การเขียนข้อความซ้ำสิ่งที่ `Err` บอกอยู่แล้วจึงไม่มีประโยชน์ แต่การอธิบาย
**สมมติฐาน** ที่คุณตั้งไว้ตอนเขียนโค้ดจุดนี้ (เช่น "ควรหาผู้ใช้เจอเสมอเพราะเช็ค username มาก่อนแล้ว") ช่วยให้คนอ่าน
log ตอน production พังเข้าใจได้ทันทีว่า **สมมติฐานไหนที่ผิดไปจากที่คิด** ซึ่งมีประโยชน์กว่ามากตอน debug จริง

#### เมื่อไหร่ที่ `.unwrap()`/`.expect()` ยอมรับได้ (Result-specific)

หลักการเดียวกับที่ Part 11 อธิบายไว้กับ `Option` ใช้ได้กับ `Result` เช่นกัน: **ยอมรับได้เมื่อคุณมั่นใจ 100% ว่า
`Err` ไม่มีทางเกิดขึ้นจริง ณ จุดนั้น (หรือถ้าเกิด ก็ควรทำให้โปรแกรม crash ทันทีเพราะมันคือบั๊ก ไม่ใช่สถานการณ์ที่ควร
กู้คืน)** สถานการณ์ที่พบบ่อยที่สุด:

1. **โค้ดตัวอย่าง/prototype ที่ยังไม่ต้องทำ production-grade error handling** — เขียน `.unwrap()` ไปก่อนเพื่อให้
   เห็นภาพรวม logic ทำงาน แล้วค่อยกลับมาแทนที่ด้วย proper error handling ทีหลัง
2. **ค่าที่มาจาก constant หรือข้อมูลที่ hard-code ไว้ในโค้ดเอง ไม่ได้มาจาก input ภายนอก** เช่น
   `"2024-01-01".parse::<u32>()` (แค่ตัวอย่างสมมติ) ที่ string มาจากโค้ดตรง ๆ ไม่มีทางเปลี่ยนแปลงตอน runtime
3. **test code** — ใน `#[test]` function การ `.unwrap()` เป็นเรื่องปกติมาก เพราะถ้า setup ของ test ล้มเหลว
   ก็อยากให้ test fail ทันทีอยู่แล้ว ไม่มีประโยชน์ที่จะเขียน error handling ที่ซับซ้อนใน test
4. **สถานการณ์ที่คุณเช็ค invariant ไว้ก่อนหน้าแล้วจริง ๆ** เหมือนตัวอย่าง `find_user_age` ข้างบน (แม้ในตัวอย่างจริง
   นี้จะยังเสี่ยงเพราะ "เช็คไว้ก่อนแล้ว" อาจผิดพลาดได้ถ้า logic การเช็คมีบั๊ก — ในโค้ด production ควรพิจารณาใช้
   `?` หรือ `match` แทนเสมอถ้ามีทางเลือก)

**ไม่ควรใช้** ในโค้ด production ที่ error มาจากแหล่งข้อมูลภายนอกที่ควบคุมไม่ได้ (input จากผู้ใช้, การอ่านไฟล์,
การเรียก network, การ parse response จาก API ภายนอก) เพราะการ panic ทั้งโปรแกรมเพียงเพราะ error ที่คาดเดาได้ล่วง
หน้าว่าจะเกิดขึ้นเป็นปกติ ถือเป็นการออกแบบที่ผิดพลาด — ควรใช้ `Result` ร่วมกับ `?` (หัวขัดถัดไป) เพื่อส่งต่อ error
ให้ชั้นที่เหมาะสมกว่าจัดการแทน

### 12.6 `?` Operator: หัวใจของ Error Handling ใน Rust

นี่คือหัวข้อที่สำคัญที่สุดของบทนี้ ก่อนจะรู้จัก `?` มาดูก่อนว่าปัญหาที่มันแก้คืออะไร

#### ปัญหา: การ propagate error ผ่านหลายขั้นตอนด้วย `match` ล้วน ๆ นั้นยาวและซ้ำซ้อน

สมมติเราต้องเขียนฟังก์ชันที่รับ string สองตัว แปลงเป็นตัวเลขทั้งคู่ แล้วคูณกัน — ถ้าตัวใดตัวหนึ่งแปลงไม่สำเร็จ ต้อง
ส่ง error นั้นกลับออกไปทันที ไม่คำนวณต่อ เขียนด้วย `match` ล้วน ๆ (ที่คุณรู้จักมาตั้งแต่ Part 10) จะได้แบบนี้:

```rust
use std::num::ParseIntError;

// เวอร์ชัน "ยาว": ใช้ match + early return ทุกขั้นตอนที่อาจล้มเหลว
fn parse_two_and_multiply_verbose(a: &str, b: &str) -> Result<i32, ParseIntError> {
    let first = match a.parse::<i32>() {
        Ok(value) => value,
        Err(e) => return Err(e), // ล้มเหลว → คืน Err ออกจากฟังก์ชันทันที
    };

    let second = match b.parse::<i32>() {
        Ok(value) => value,
        Err(e) => return Err(e), // ล้มเหลว → คืน Err ออกจากฟังก์ชันทันที
    };

    Ok(first * second)
}

fn main() {
    println!("{:?}", parse_two_and_multiply_verbose("6", "7"));
    println!("{:?}", parse_two_and_multiply_verbose("6", "abc"));
}
```

ผลลัพธ์:

```
Ok(42)
Err(ParseIntError { kind: InvalidDigit })
```

โค้ดนี้**ทำงานถูกต้องสมบูรณ์** แต่สังเกตว่า pattern `match ... { Ok(value) => value, Err(e) => return Err(e) }`
**ซ้ำกันทุกครั้งที่มีขั้นตอนที่อาจล้มเหลว** — ถ้าฟังก์ชันมี 5 ขั้นตอนที่อาจล้มเหลว คุณจะต้องเขียน `match` แบบนี้
5 ครั้ง ซึ่งไม่ได้แค่ยาว แต่ยังทำให้ **logic ที่แท้จริงของฟังก์ชัน** (ในที่นี้คือ "เอาสองตัวเลขมาคูณกัน") จมอยู่ใต้
โค้ด boilerplate ของการจัดการ error จนมองเห็นยากขึ้นมาก

#### วิธีแก้: `?` operator

Rust มี syntax พิเศษที่เรียกว่า **`?` operator** ที่ทำหน้าที่ตรงกับ pattern ข้างบนนี้เป๊ะ ๆ แต่เขียนสั้นกว่ามาก:

```rust
use std::num::ParseIntError;

// เวอร์ชันเดียวกัน เขียนด้วย `?`
fn parse_two_and_multiply(a: &str, b: &str) -> Result<i32, ParseIntError> {
    let first = a.parse::<i32>()?;
    let second = b.parse::<i32>()?;
    Ok(first * second)
}

fn main() {
    println!("{:?}", parse_two_and_multiply("6", "7"));
    println!("{:?}", parse_two_and_multiply("6", "abc"));
}
```

ผลลัพธ์เหมือนกับเวอร์ชันยาวทุกประการ:

```
Ok(42)
Err(ParseIntError { kind: InvalidDigit })
```

`let first = a.parse::<i32>()?;` ทำสิ่งที่ `match` 5 บรรทัดข้างบนทำเป๊ะ ๆ ในบรรทัดเดียว — โค้ดที่เหลือของ
ฟังก์ชันตอนนี้เห็น logic จริงชัดเจนมาก (`a.parse()?`, `b.parse()?`, แล้วคูณกัน) โดยไม่มี boilerplate การจัดการ
error มาบังตา

#### `?` desugar เป็นอะไรจริง ๆ — เจาะลึกแบบละเอียด

`?` **ไม่ใช่ฟังก์ชัน ไม่ใช่ magic ของ runtime ใด ๆ ทั้งสิ้น** มันคือ **syntactic sugar** ที่ compiler แปลงเป็นโค้ด
ธรรมดาให้ตั้งแต่ตอน compile time (ก่อนจะไปเป็น machine code เสียอีก) กฎการแปลงของ `expr?` มีดังนี้:

1. ประเมินค่า `expr` (ซึ่งต้องมี type เป็น `Result<T, E>` — หรือ `Option<T>` ก็ใช้ `?` ได้เช่นกันตามที่ Part 11
   อาจแนะนำไว้ แต่บทนี้โฟกัสที่ `Result`)
2. ถ้าผลลัพธ์เป็น `Ok(value)` → **ค่าทั้ง expression `expr?` คือ `value`** (แค่ "แกะ" ค่าออกจาก `Ok` มาให้ใช้ต่อ)
3. ถ้าผลลัพธ์เป็น `Err(error)` → **ทำ `return Err(From::from(error))` ออกจากฟังก์ชันปัจจุบันทันที** (จะอธิบาย
   ส่วน `From::from` ในหัวข้อ 12.7 — สำหรับตอนนี้ให้มองว่ามันคือ "แปลง error ให้เข้ากับชนิดที่ฟังก์ชันประกาศไว้")

พูดง่าย ๆ คือ **`expr?` แปลว่า "ถ้าสำเร็จ เอาค่าออกมาใช้ต่อ; ถ้าล้มเหลว หยุดฟังก์ชันนี้แล้วส่ง error กลับไปให้ผู้เรียก
ทันที"** ซึ่งตรงกับ pattern `match { Ok(v) => v, Err(e) => return Err(e) }` เป๊ะ ๆ ทุกประการ — โค้ดสองเวอร์ชันข้างบน
(`parse_two_and_multiply_verbose` และ `parse_two_and_multiply`) จึง **compile ไปเป็น machine code ที่เหมือนกันทุก
ประการ** ไม่มีความต่างด้าน performance เลยแม้แต่นิดเดียว

> **สิ่งที่สำคัญที่สุดที่ต้องเข้าใจ**: `?` **ไม่ใช่กลไก exception** และไม่มีอะไรที่คล้าย `try/catch` เกี่ยวข้องเลย
> มันไม่มี "การค้นหา handler ที่เหมาะสมตาม call stack" (ซึ่งเป็นกลไกที่ภาษาอย่าง Java/Python/C++ ใช้เวลาทำงานจริง
> เรียกว่า **stack unwinding เพื่อหา catch block**) `?` เป็นแค่ **`return` แบบมีเงื่อนไข** ที่ compiler เขียนแทน
> คุณตั้งแต่ตอน compile — ไม่มี lookup table ของ handler, ไม่มีการค้นหาอะไรตอน runtime, ไม่มีต้นทุนที่มากไปกว่า
> การเช็ค `if` ธรรมดาสักครั้งเดียว นี่คือความหมายของคำว่า **zero-cost abstraction** ที่เราพูดถึงมาตั้งแต่ Part 1
> และ Part 3: `?` ให้ความสะดวกเชิง syntax เหมือนกับที่ exception ให้ในภาษาอื่น แต่ compile ไปเป็นโค้ดที่มี
> ประสิทธิภาพเท่ากับที่คุณเขียน `match` ด้วยมือทุกประการ ไม่มากไม่น้อยไปกว่านั้นเลย

เพื่อให้เห็นภาพว่า "desugar" หมายความว่าอย่างไรจริง ๆ ลองเขียนเวอร์ชัน "จำลองการขยายโค้ดด้วยมือ" ของ
`a.parse::<i32>()?` ดูตรง ๆ:

```rust
use std::num::ParseIntError;

fn parse_two_and_multiply_desugared(a: &str, b: &str) -> Result<i32, ParseIntError> {
    // บรรทัดนี้: let first = a.parse::<i32>()?;
    // compiler มองเห็นเป็นสิ่งที่เทียบเท่ากับโค้ดข้างล่างนี้ทุกประการ:
    let first = match a.parse::<i32>() {
        Ok(value) => value,
        Err(error) => return Err(From::from(error)),
    };

    let second = match b.parse::<i32>() {
        Ok(value) => value,
        Err(error) => return Err(From::from(error)),
    };

    Ok(first * second)
}

fn main() {
    println!("{:?}", parse_two_and_multiply_desugared("6", "7"));
}
```

โค้ดนี้ compile และรันได้ผลลัพธ์เหมือนกันทุกประการกับสองเวอร์ชันก่อนหน้า (`Ok(42)`) — นี่คือสิ่งที่ `?` ทำแทนคุณ
อย่างแท้จริง ในกรณีนี้ `From::from(error)` แค่คืนค่า `error` เดิม (เพราะ `E` ทั้งสองฝั่งเป็น `ParseIntError`
เหมือนกัน ทุก type implement `From<Self>` ให้ตัวเองเสมอโดย default — เป็น "การแปลงแบบไม่มีอะไรต้องแปลง") แต่ในกรณี
ที่ error type ของ `?` (ฝั่งขวา) กับ error type ที่ฟังก์ชันประกาศไว้ (ฝั่งซ้าย) **ต่างกัน** นี่คือจุดที่ `From` เข้ามา
มีบทบาทจริง ๆ ซึ่งเป็นเนื้อหาของหัวข้อถัดไป

### 12.7 `?` กับการแปลงชนิด Error โดยอัตโนมัติผ่าน `From`

ในหัวข้อก่อนเราเห็นว่า `?` desugar เป็น `return Err(From::from(error))` — คำถามคือ **ทำไมต้องมี `From::from`
แทรกอยู่ด้วย ทำไมไม่ใช่แค่ `return Err(error)` ตรง ๆ?**

คำตอบคือ: ในโปรแกรมจริง ฟังก์ชันหนึ่งมักเรียกใช้หลายฟังก์ชันย่อยที่ **แต่ละตัวคืน error type ต่างกัน** (เช่น
ฟังก์ชันหนึ่งอ่านไฟล์แล้วคืน `std::io::Error`, อีกฟังก์ชัน parse ตัวเลขแล้วคืน `ParseIntError`) แต่ฟังก์ชันแม่ต้อง
ประกาศ return type เป็น error type **เดียว** — `?` จึงต้องมีวิธี "แปลง" error จากฟังก์ชันย่อยให้เข้ากับ error type
ของฟังก์ชันแม่โดยอัตโนมัติ และ trait ที่ Rust เลือกใช้สำหรับการแปลงชนิดข้อมูลแบบนี้คือ **`From`**

ลองดูสถานการณ์ที่ `?` **compile ไม่ผ่าน** เพราะ error type ไม่ตรงกัน และไม่มีทางแปลงให้:

```rust
#[derive(Debug)]
struct ConfigError {
    message: String,
}

fn parse_port(raw: &str) -> Result<u16, ConfigError> {
    let port = raw.parse::<u16>()?;
    Ok(port)
}

fn main() {
    println!("{:?}", parse_port("8080"));
}
```

โค้ดนี้ compile ไม่ผ่าน เพราะ `raw.parse::<u16>()` คืน `Result<u16, ParseIntError>` แต่ฟังก์ชัน `parse_port`
ประกาศไว้ว่าจะคืน `Result<u16, ConfigError>` — เมื่อเจอ `Err`, `?` ต้องแปลง `ParseIntError` เป็น `ConfigError`
แต่ไม่มีใครสอนให้มันแปลงได้เลย:

```
error[E0277]: `?` couldn't convert the error to `ConfigError`
 --> src/main.rs:7:34
  |
6 | fn parse_port(raw: &str) -> Result<u16, ConfigError> {
  |                             ------------------------ expected `ConfigError` because of this
7 |     let port = raw.parse::<u16>()?;
  |                    --------------^ the trait `From<ParseIntError>` is not implemented for `ConfigError`
  |                    |
  |                    this can't be annotated with `?` because it has type `Result<_, ParseIntError>`
  |
note: `ConfigError` needs to implement `From<ParseIntError>`
 --> src/main.rs:2:1
  |
2 | struct ConfigError {
  | ^^^^^^^^^^^^^^^^^^
  = note: the question mark operation (`?`) implicitly performs a conversion on the error value using the `From` trait

error: aborting due to 1 previous error
```

นี่คือ error `E0277` — "the trait bound is not satisfied" — ตัวจริง สังเกตว่า compiler บอกตรงมากว่าปัญหาคืออะไร:
`the trait \`From<ParseIntError>\` is not implemented for \`ConfigError\`` และยังบอกวิธีแก้ในบรรทัด `note` ด้วย:
`ConfigError needs to implement From<ParseIntError>` — นี่คือกลไกที่แท้จริงของ `?`: **มันเรียก
`ConfigError::from(the_parse_int_error)` โดยอัตโนมัติทุกครั้งที่เจอ `Err`** ถ้า `ConfigError` ไม่มี `impl
From<ParseIntError>` ให้ compiler ก็ไม่รู้ว่าจะแปลงยังไง จึง compile ไม่ผ่าน

#### วิธีแก้ที่ 1: เขียน `impl From` เอง

วิธีที่ตรงไปตรงมาที่สุดคือสอนให้ `ConfigError` "รู้วิธีแปลงตัวเอง" จาก `ParseIntError`:

```rust
use std::num::ParseIntError;

#[derive(Debug)]
struct ConfigError {
    message: String,
}

// สอนให้ ConfigError "แปลงมาจาก" ParseIntError ได้ ผ่าน trait From
impl From<ParseIntError> for ConfigError {
    fn from(e: ParseIntError) -> Self {
        ConfigError {
            message: format!("port ไม่ถูกต้อง: {e}"),
        }
    }
}

fn parse_port(raw: &str) -> Result<u16, ConfigError> {
    // ตอนนี้ `?` compile ผ่านแล้ว เพราะ compiler รู้ว่าจะเรียก
    // ConfigError::from(parse_int_error) ให้เองโดยอัตโนมัติเมื่อเจอ Err
    let port = raw.parse::<u16>()?;
    Ok(port)
}

fn main() {
    match parse_port("8080") {
        Ok(port) => println!("port: {port}"),
        Err(e) => println!("error: {}", e.message),
    }

    match parse_port("not-a-port") {
        Ok(port) => println!("port: {port}"),
        Err(e) => println!("error: {}", e.message),
    }
}
```

ผลลัพธ์:

```
port: 8080
error: port ไม่ถูกต้อง: invalid digit found in string
```

ตอนนี้ `?` compile ผ่านแล้ว เพราะทุกครั้งที่ `raw.parse::<u16>()` คืน `Err(ParseIntError)`, `?` จะเรียก
`ConfigError::from(that_error)` ให้เองโดยอัตโนมัติ (ตามกฎ desugar ที่อธิบายไว้ในหัวข้อ 12.6: `return
Err(From::from(error))`) แล้วส่ง `ConfigError` ที่ได้กลับออกไปแทน — จุดที่น่าสนใจคือ **โค้ดในฟังก์ชัน `parse_port`
เองไม่มีการเรียก `.from()` หรือ `.into()` ให้เห็นตรง ๆ เลย** ทั้งหมดถูกซ่อนไว้ในกลไกของ `?` — นี่คือพลังของการ
ผสาน `?` กับ `From`: คุณเขียน `impl From` แค่ครั้งเดียว แล้ว `?` ทุกจุดในโค้ดของคุณที่มีสถานการณ์แบบนี้จะแปลง error
ให้อัตโนมัติตลอดไป โดยไม่ต้องเขียน `.map_err(...)` ซ้ำ ๆ ทุกจุดที่เรียก `.parse()`

#### วิธีแก้ที่ 2: `Box<dyn Error>` — ทางลัดที่ pragmatic สำหรับตอนนี้

การเขียน `impl From` ให้ทุก error type คู่ที่อาจเกิดขึ้นได้นั้นดีมากสำหรับโค้ด production ที่ต้องการ error
handling ที่ชัดเจนและตรวจสอบได้ (จะเจาะลึกการออกแบบ custom error type แบบเต็มรูปแบบ พร้อม crate อย่าง
`thiserror` ใน **Part 30**) แต่ในหลายสถานการณ์ (โดยเฉพาะโค้ดขนาดเล็ก, script, หรือ prototype) การเขียน `impl
From` ให้ทุกคู่ที่เป็นไปได้อาจดูเวอร์เกินไป Rust standard library มีทางลัดที่ใช้ได้จริงและเป็นที่ยอมรับกว้างขวางคือ
**`Box<dyn std::error::Error>`**:

```rust
use std::error::Error;

fn parse_port(raw: &str) -> Result<u16, Box<dyn Error>> {
    // ParseIntError implement std::error::Error ให้แล้ว จึงแปลงเป็น Box<dyn Error>
    // ได้อัตโนมัติผ่าน `?` (เพราะ Box<dyn Error> implement From<E> ให้ทุก E: Error)
    let port = raw.parse::<u16>()?;
    Ok(port)
}

fn main() {
    match parse_port("8080") {
        Ok(port) => println!("port ที่ใช้งาน: {port}"),
        Err(e) => println!("แปลง port ไม่สำเร็จ: {e}"),
    }

    match parse_port("not-a-port") {
        Ok(port) => println!("port ที่ใช้งาน: {port}"),
        Err(e) => println!("แปลง port ไม่สำเร็จ: {e}"),
    }
}
```

ผลลัพธ์:

```
port ที่ใช้งาน: 8080
แปลง port ไม่สำเร็จ: invalid digit found in string
```

**อธิบายว่า `Box<dyn Error>` ทำงานอย่างไร:**

- `std::error::Error` คือ **trait** ที่ standard library กำหนดให้ error type ที่ "เหมาะสม" ควร implement
  (ต้อง implement `Display` และ `Debug` ด้วย) — `ParseIntError`, `std::io::Error`, และ error type มาตรฐานส่วน
  ใหญ่ implement มันให้แล้ว
- `dyn Error` คือ **trait object** (แนวคิดที่จะเจาะลึกใน Part 19 เรื่อง Traits) หมายถึง "อะไรก็ได้ที่ implement
  `Error` trait โดยไม่ต้องรู้ชนิดที่แน่นอนตอน compile time" — เพราะ error ต่าง ๆ ที่คุณอาจเจอ (`ParseIntError`,
  `std::io::Error`, custom error ของคุณเอง) มีขนาดและโครงสร้างต่างกัน ไม่สามารถเก็บไว้ใน type เดียวกันตรง ๆ ได้
  จึงต้องห่อด้วย `Box<...>` (สร้าง pointer ไปยัง heap ที่ขนาดคงที่ ไม่ว่า error จริงข้างในจะมีขนาดเท่าไหร่)
- **`Box<dyn Error>` implement `From<E>` ให้กับทุก type `E` ที่ implement `Error`** — นี่คือเหตุผลที่ `?` ใช้ได้
  ตรง ๆ โดยไม่ต้องเขียน `impl From` เองเลย เพราะ standard library เขียน `impl From` ตัวกลาง ๆ ที่ครอบคลุมทุก
  error type ที่ implement `Error` ไว้ให้แล้วล่วงหน้า

**ข้อแลกเปลี่ยน (trade-off) ของ `Box<dyn Error>`**: มันสะดวกมาก แต่แลกมาด้วยการ**เสียข้อมูลชนิดที่แน่นอนของ
error** ไป — ผู้เรียกฟังก์ชันที่คืน `Box<dyn Error>` จะรู้แค่ว่า "มัน implement `Error` trait" แต่ไม่รู้ว่ามันคือ
`ParseIntError` หรือ `std::io::Error` หรืออะไรกันแน่ (นอกจากจะพยายาม downcast ซึ่งยุ่งยากและไม่ค่อย idiomatic)
ทำให้ **match แยกกรณีตาม error type ที่แท้จริงไม่ได้อีกต่อไป** ถ้าโปรแกรมต้องการ logic ที่ตอบสนองต่าง error type
ต่างกัน (เช่น "ถ้าเป็น network error ให้ retry แต่ถ้าเป็น validation error ให้แจ้งผู้ใช้ทันที") การใช้ custom error
enum (ที่ implement `From` เองแบบหัวข้อก่อน หรือใช้ crate `thiserror` ที่จะเรียนใน Part 30) จะเหมาะสมกว่ามาก

**สรุปแนวทางเลือกใช้สำหรับตอนนี้**:

| สถานการณ์ | แนวทางที่เหมาะสม |
|---|---|
| Prototype, script สั้น ๆ, โปรแกรมขนาดเล็กที่ผู้เรียกไม่ต้องแยกกรณี error | `Box<dyn Error>` — เขียนเร็ว ได้ `?` ใช้งานทันที |
| Library หรือระบบที่ผู้เรียกต้องแยกกรณี error เพื่อตัดสินใจต่อ | Custom error type ที่ `impl From` เอง (จะเห็นเวอร์ชันเต็มพร้อม `thiserror` ใน Part 30) |
| `main()` ของโปรแกรมระดับ application ที่ไม่มีใครเรียก `main()` ต่ออีก | `Box<dyn Error>` มักเพียงพอ (หัวข้อถัดไป) |

### 12.8 `?` ใน `fn main()`: `Result<(), Box<dyn Error>>`

หนึ่งใน pattern ที่พบมากที่สุดในโค้ด Rust จริงคือการให้ `main()` เองคืนค่าเป็น `Result` แทนที่จะเป็น `()` เฉย ๆ
(อย่างที่คุณเขียนมาตลอดตั้งแต่ Part 1) เพื่อให้ใช้ `?` ได้ตรง ๆ ใน `main()` เอง โดยไม่ต้อง `.unwrap()`/`match`
ทุกครั้ง:

```rust
use std::error::Error;

fn parse_port(raw: &str) -> Result<u16, Box<dyn Error>> {
    let port = raw.parse::<u16>()?;
    Ok(port)
}

fn main() -> Result<(), Box<dyn Error>> {
    let port = parse_port("8080")?;
    println!("เริ่มเซิร์ฟเวอร์ที่ port {port}");
    Ok(())
}
```

ผลลัพธ์เมื่อรัน:

```
เริ่มเซิร์ฟเวอร์ที่ port 8080
```

และโปรแกรมออกด้วย **exit code 0** (สำเร็จ) เพราะ `main()` คืน `Ok(())`

ลองเปลี่ยน input ให้ parse ไม่สำเร็จ:

```rust
use std::error::Error;

fn parse_port(raw: &str) -> Result<u16, Box<dyn Error>> {
    let port = raw.parse::<u16>()?;
    Ok(port)
}

fn main() -> Result<(), Box<dyn Error>> {
    let port = parse_port("not-a-port")?;
    println!("เริ่มเซิร์ฟเวอร์ที่ port {port}");
    Ok(())
}
```

โปรแกรมนี้ **ไม่ panic** แต่พิมพ์ข้อความไปที่ **stderr** และออกด้วย **exit code 1** (ไม่สำเร็จ):

```
Error: ParseIntError { kind: InvalidDigit }
```

**อธิบายกลไกที่เกิดขึ้น:**

- Rust ยอมให้ `fn main()` คืนค่าเป็น type ใดก็ได้ที่ implement trait พิเศษชื่อ `std::process::Termination`
  — ซึ่ง `()` (ที่คุณใช้มาตลอด) และ `Result<T, E>` (โดย `E: Debug`) ทั้งคู่ implement ให้แล้วโดย default
- ถ้า `main()` คืน `Ok(())` → runtime ของ Rust ถือว่าโปรแกรม**สำเร็จ** จบด้วย **exit code 0** เหมือนกับ `main()`
  แบบเดิมที่ไม่มี return type เลย
- ถ้า `main()` คืน `Err(e)` → runtime จะ **พิมพ์ `Debug` ของ `e` ออกไปที่ stderr โดยขึ้นต้นด้วยคำว่า `Error: `**
  โดยอัตโนมัติ แล้วจบโปรแกรมด้วย **exit code 1** (ไม่สำเร็จ) — สังเกตว่า **ไม่มีการ panic เกิดขึ้นเลย** ไม่มี
  `thread 'main' panicked at ...`, ไม่มี stack backtrace ทั้งหมดนี้ต่างจาก `.unwrap()` อย่างสิ้นเชิงในเชิงว่า
  "ดูสุภาพกว่า" และเป็นพฤติกรรมที่คาดหวังได้ (ไม่ใช่ crash)

**ทำไม pattern นี้มีประโยชน์มาก:** โปรแกรมบรรทัดคำสั่ง (CLI) จำนวนมากต้องอ่านไฟล์, parse argument, เชื่อมต่อ
เครือข่าย — ล้วนเป็นขั้นตอนที่ล้มเหลวได้ตามปกติ (recoverable) แต่ถ้ามันล้มเหลวใน `main()` เอง โดยทั่วไปก็ "ไม่มีอะไร
ทำต่อได้แล้ว" นอกจากแจ้งผู้ใช้ว่าเกิดอะไรขึ้นแล้วจบโปรแกรม การเขียน `fn main() -> Result<(), Box<dyn Error>>`
พร้อม `?` ทำให้คุณเขียน `main()` ที่อ่านง่ายเป็น "ลำดับขั้นตอนที่อาจล้มเหลว" ตรง ๆ โดยไม่ต้องมี `match`/`.unwrap()`
รกโค้ด และยังได้ exit code ที่ถูกต้อง (`0` = สำเร็จ, ไม่ใช่ `0` = ล้มเหลว) ซึ่งสำคัญมากสำหรับโปรแกรมที่ถูกเรียกใช้
จาก shell script หรือระบบ automation ที่เช็ค exit code เพื่อตัดสินใจขั้นตอนถัดไป (เช่น CI/CD pipeline)

**หมายเหตุ**: ข้อความ `Error: ParseIntError { kind: InvalidDigit }` ที่เห็นมาจากการพิมพ์ `Debug` ของ error โดยตรง
ซึ่งอาจไม่เป็นมิตรกับผู้ใช้ทั่วไปมากนัก (มันคือรายละเอียดภายในของ struct) ถ้าต้องการควบคุมข้อความที่แสดงให้สวยงามกว่า
นี้ในโปรแกรมจริง วิธีที่นิยมคือ `match` ผลลัพธ์ของฟังก์ชันหลักเองใน `main()` แล้วเรียก `std::process::exit(1)`
พร้อมข้อความที่จัดรูปแบบเอง (จะกล่าวถึงใน 12.12) หรือใช้ crate `anyhow` (Part 31) ที่ทำให้ error message ที่แสดง
ออกมาอ่านง่ายกว่า `Debug` ธรรมดามาก

### 12.9 Combinators ของ `Result`: เทียบเคียงกับ `Option`

Part 11 สอนคุณเรื่อง combinator ของ `Option<T>` มาแล้ว (`.map()`, `.and_then()`, `.unwrap_or()`,
`.unwrap_or_else()`) — `Result<T, E>` มี combinator ชุดเดียวกันที่ทำงานคล้ายกันมาก บวกกับตัวที่มีเฉพาะ `Result`
เท่านั้นเพราะมันมีข้อมูลใน `Err` ด้วย (`.map_err()`) มาดูทั้งหมดในตัวอย่างเดียว:

```rust
fn main() {
    let ok_val: Result<i32, String> = Ok(10);
    let err_val: Result<i32, String> = Err("boom".to_string());

    // map: แปลงค่าใน Ok เฉย ๆ ไม่แตะ Err
    // (หมายเหตุ: combinator เหล่านี้ "กิน" ค่า self เข้าไปตรง ๆ (take ownership)
    //  เหมือน method อื่น ๆ ที่เรียนใน Part 6-7 จึงต้อง .clone() ถ้าจะใช้ค่าเดิมต่อ)
    println!("{:?}", ok_val.clone().map(|v| v * 2)); // Ok(20)
    println!("{:?}", err_val.clone().map(|v| v * 2)); // Err("boom")

    // map_err: แปลงค่าใน Err เฉย ๆ ไม่แตะ Ok
    let mapped_err: Result<i32, usize> = err_val.clone().map_err(|e| e.len());
    println!("{:?}", mapped_err); // Err(4)

    // and_then: เหมือน map แต่ closure คืน Result เอง (สำหรับ chain หลายขั้นตอน)
    let chained: Result<i32, String> = ok_val.clone().and_then(|v| {
        if v > 0 {
            Ok(v * 100)
        } else {
            Err("ค่าต้องมากกว่า 0".to_string())
        }
    });
    println!("{:?}", chained); // Ok(1000)

    // unwrap_or: ให้ค่า default ถ้าเป็น Err (ประเมิน default ทันทีเสมอ)
    println!("{}", err_val.clone().unwrap_or(-1)); // -1

    // unwrap_or_else: เหมือนกันแต่ default มาจาก closure (คำนวณเฉพาะตอนจำเป็น)
    println!("{}", err_val.clone().unwrap_or_else(|e| e.len() as i32)); // 4

    // ok(): แปลง Result<T, E> เป็น Option<T> (ทิ้งข้อมูล error ไปเลย)
    println!("{:?}", ok_val.clone().ok()); // Some(10)
    println!("{:?}", err_val.clone().ok()); // None

    // is_ok / is_err: เช็คสถานะแบบไม่ดึงค่าออกมา (ยืม &self เฉย ๆ ไม่กินค่า)
    println!("{} {}", ok_val.is_ok(), ok_val.is_err()); // true false
    println!("{} {}", err_val.is_ok(), err_val.is_err()); // false true
}
```

ผลลัพธ์:

```
Ok(20)
Err("boom")
Err(4)
Ok(1000)
-1
4
Some(10)
None
true false
false true
```

**ตารางสรุป combinator ทั้งหมด เทียบกับ `Option` ที่เรียนจาก Part 11:**

| Method | ทำงานกับ `Ok`/`Some` | ทำงานกับ `Err`/`None` | มีเฉพาะ `Result` เท่านั้น? |
|---|---|---|---|
| `.map(f)` | แปลงค่าข้างในด้วย `f` | ปล่อยผ่านเหมือนเดิม | ไม่ (มีคู่กับ `Option::map`) |
| `.map_err(f)` | ปล่อยผ่านเหมือนเดิม | แปลงค่า error ข้างในด้วย `f` | **ใช่** (เพราะ `Option::None` ไม่มีข้อมูลให้แปลง) |
| `.and_then(f)` | เรียก `f` (ที่คืน `Result`/`Option` เอง) เพื่อ chain ต่อ | ปล่อยผ่านเหมือนเดิม | ไม่ (มีคู่กับ `Option::and_then`) |
| `.unwrap_or(default)` | คืนค่าข้างใน | คืน `default` (evaluate ทันทีเสมอไม่ว่าจะใช้จริงหรือไม่) | ไม่ |
| `.unwrap_or_else(f)` | คืนค่าข้างใน | เรียก `f` เพื่อคำนวณ default (evaluate เฉพาะตอนจำเป็น) | ไม่ |
| `.ok()` | แปลงเป็น `Some(value)` | แปลงเป็น `None` (**ทิ้งข้อมูล error ไปเลย**) | **ใช่** (`Option` ไม่มีอะไรให้ "ok" เพราะไม่มี error ตั้งแต่แรก) |
| `.is_ok()` / `.is_err()` | เช็คสถานะแบบ bool ไม่ดึงค่า | เช็คสถานะแบบ bool ไม่ดึงค่า | ไม่ (คู่กับ `.is_some()`/`.is_none()`) |

จุดที่ควรสังเกตเชิงลึกจากตัวอย่างที่รันมา:

- **`.map_err()` มีประโยชน์มากที่สุดตอนต้องการแปลง error type ให้เข้ากับ signature ของฟังก์ชัน** เป็นอีกวิธีหนึ่ง
  (นอกจาก `impl From` ในหัวข้อ 12.7) ในการแก้ปัญหา error type ไม่ตรงกัน โดยเขียนแบบ **explicit** ตรงจุดที่เกิด
  แทนที่จะพึ่งพา `?`+`From` ที่ทำงานแบบ implicit — เหมาะกับกรณีที่ต้องการแปลงแบบเฉพาะจุดเดียว ไม่อยากให้กระทบ
  ทุกจุดที่ใช้ `?` กับ error type คู่นั้นทั่วทั้งโปรแกรม
- **`.unwrap_or()` กับ `.unwrap_or_else()` ต่างกันสำคัญมากเรื่อง "เวลาที่ evaluate ค่า default"** — เราจะเห็นผล
  ที่เป็นรูปธรรมของความต่างนี้ในหัวข้อ "กับดักที่พบบ่อย" ด้านล่าง เพราะเป็นจุดที่มือใหม่พลาดบ่อยมาก
- **`.ok()` เป็นสะพานที่ตรงข้ามกับ `.ok_or()`/`.ok_or_else()` ที่ Part 11 สอน** — `Option::ok_or()` แปลง
  `Option<T>` **ไปเป็น** `Result<T, E>` (ต้องเติม error เข้าไปเพราะ `None` ไม่มีข้อมูล error ให้ใช้) ส่วน
  `Result::ok()` แปลง `Result<T, E>` **กลับมาเป็น** `Option<T>` (ทิ้งข้อมูล error ที่มีอยู่แล้วไปเลย) ทั้งสอง
  method นี้คือ "สะพานสองทาง" ที่เชื่อม `Option` กับ `Result` เข้าด้วยกัน — เลือกใช้ตามทิศทางที่ต้องการแปลง

### 12.10 ตัวอย่างจริง: การ Propagate Error ข้ามหลายขั้นตอนด้วย Config Parser

มาดูตัวอย่างที่ใกล้เคียงการใช้งานจริงมากขึ้น: parse "บรรทัด config" ที่มีรูปแบบ `"port=8080,verbose=true"` ให้เป็น
struct `Config` ที่มี field `port: u16` และ `verbose: bool` — งานนี้มีขั้นตอนที่อาจล้มเหลวได้หลายจุด (แยก field
ผิดจำนวน, key ไม่ตรงชื่อ, ค่าไม่ใช่ตัวเลข, ค่าไม่ใช่ boolean) เหมาะมากที่จะเห็นความต่างระหว่างเขียนด้วย `match`
ซ้อนกันล้วน ๆ เทียบกับใช้ `?`

#### เวอร์ชัน `match` ซ้อนกันล้วน ๆ (verbose)

```rust
use std::error::Error;
use std::fmt;

#[derive(Debug)]
struct Config {
    port: u16,
    verbose: bool,
}

#[derive(Debug)]
struct ConfigParseError(String);

impl fmt::Display for ConfigParseError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "config parse error: {}", self.0)
    }
}
impl Error for ConfigParseError {}

// รูปแบบ input ที่คาดไว้: "port=8080,verbose=true"
fn parse_config_verbose(line: &str) -> Result<Config, ConfigParseError> {
    let parts: Vec<&str> = line.split(',').collect();
    if parts.len() != 2 {
        return Err(ConfigParseError(format!(
            "ต้องมี 2 ฟิลด์คั่นด้วย comma แต่พบ {} ฟิลด์",
            parts.len()
        )));
    }

    let port_part = parts[0];
    let verbose_part = parts[1];

    let port_kv: Vec<&str> = port_part.split('=').collect();
    if port_kv.len() != 2 || port_kv[0] != "port" {
        return Err(ConfigParseError(format!("รูปแบบฟิลด์ port ผิด: '{port_part}'")));
    }

    let port: u16 = match port_kv[1].parse() {
        Ok(value) => value,
        Err(e) => return Err(ConfigParseError(format!("port ไม่ใช่ตัวเลขที่ถูกต้อง: {e}"))),
    };

    let verbose_kv: Vec<&str> = verbose_part.split('=').collect();
    if verbose_kv.len() != 2 || verbose_kv[0] != "verbose" {
        return Err(ConfigParseError(format!(
            "รูปแบบฟิลด์ verbose ผิด: '{verbose_part}'"
        )));
    }

    let verbose: bool = match verbose_kv[1].parse() {
        Ok(value) => value,
        Err(e) => {
            return Err(ConfigParseError(format!(
                "verbose ต้องเป็น true/false: {e}"
            )))
        }
    };

    Ok(Config { port, verbose })
}

fn show(result: Result<Config, ConfigParseError>) {
    match result {
        Ok(cfg) => println!(
            "โหลด config สำเร็จ: port={}, verbose={}",
            cfg.port, cfg.verbose
        ),
        Err(e) => println!("โหลด config ล้มเหลว: {e}"),
    }
}

fn main() {
    show(parse_config_verbose("port=8080,verbose=true"));
    show(parse_config_verbose("port=abc,verbose=true"));
    show(parse_config_verbose("port=8080"));
}
```

ผลลัพธ์:

```
โหลด config สำเร็จ: port=8080, verbose=true
โหลด config ล้มเหลว: config parse error: port ไม่ใช่ตัวเลขที่ถูกต้อง: invalid digit found in string
โหลด config ล้มเหลว: config parse error: ต้องมี 2 ฟิลด์คั่นด้วย comma แต่พบ 1 ฟิลด์
```

โค้ดนี้ทำงานถูกต้องสมบูรณ์ แต่สังเกตว่าฟังก์ชัน `parse_config_verbose` ยาวถึง **เกือบ 30 บรรทัด** และมี `match ...
{ Ok(value) => value, Err(e) => return Err(...) }` ซ้ำอยู่ 2 ครั้ง (สำหรับ `port` และ `verbose`) ซึ่งเป็น pattern
เดียวกันกับที่หัวข้อ 12.6 อธิบายไว้ว่า `?` แก้ปัญหานี้ได้โดยตรง

#### เวอร์ชันเดียวกัน เขียนด้วย `?`

```rust
use std::error::Error;
use std::fmt;

#[derive(Debug)]
struct Config {
    port: u16,
    verbose: bool,
}

#[derive(Debug)]
struct ConfigParseError(String);

impl fmt::Display for ConfigParseError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "config parse error: {}", self.0)
    }
}
impl Error for ConfigParseError {}

// ให้ ConfigParseError แปลงมาจาก error ชนิดต่าง ๆ ได้ผ่าน From
// เพื่อให้ `?` ใช้งานข้าม error type ได้อย่างอิสระในฟังก์ชันเดียว
impl From<std::num::ParseIntError> for ConfigParseError {
    fn from(e: std::num::ParseIntError) -> Self {
        ConfigParseError(format!("ตัวเลขไม่ถูกต้อง: {e}"))
    }
}
impl From<std::str::ParseBoolError> for ConfigParseError {
    fn from(e: std::str::ParseBoolError) -> Self {
        ConfigParseError(format!("ค่า boolean ไม่ถูกต้อง: {e}"))
    }
}

fn field_value<'a>(field: &'a str, expected_key: &str) -> Result<&'a str, ConfigParseError> {
    let kv: Vec<&str> = field.split('=').collect();
    if kv.len() != 2 || kv[0] != expected_key {
        return Err(ConfigParseError(format!(
            "รูปแบบฟิลด์ '{expected_key}' ผิด: '{field}'"
        )));
    }
    Ok(kv[1])
}

// รูปแบบ input ที่คาดไว้: "port=8080,verbose=true"
fn parse_config(line: &str) -> Result<Config, ConfigParseError> {
    let parts: Vec<&str> = line.split(',').collect();
    if parts.len() != 2 {
        return Err(ConfigParseError(format!(
            "ต้องมี 2 ฟิลด์คั่นด้วย comma แต่พบ {} ฟิลด์",
            parts.len()
        )));
    }

    let port: u16 = field_value(parts[0], "port")?.parse()?;
    let verbose: bool = field_value(parts[1], "verbose")?.parse()?;

    Ok(Config { port, verbose })
}

fn show(result: Result<Config, ConfigParseError>) {
    match result {
        Ok(cfg) => println!(
            "โหลด config สำเร็จ: port={}, verbose={}",
            cfg.port, cfg.verbose
        ),
        Err(e) => println!("โหลด config ล้มเหลว: {e}"),
    }
}

fn main() {
    show(parse_config("port=8080,verbose=true"));
    show(parse_config("port=abc,verbose=true"));
    show(parse_config("port=8080"));
}
```

ผลลัพธ์ **เหมือนกันทุกประการ** กับเวอร์ชัน verbose:

```
โหลด config สำเร็จ: port=8080, verbose=true
โหลด config ล้มเหลว: config parse error: ตัวเลขไม่ถูกต้อง: invalid digit found in string
โหลด config ล้มเหลว: config parse error: ต้องมี 2 ฟิลด์คั่นด้วย comma แต่พบ 1 ฟิลด์
```

**สิ่งที่เปลี่ยนไปและทำไมมันสำคัญ:**

- บรรทัด `let port: u16 = field_value(parts[0], "port")?.parse()?;` มี **`?` ถึงสองตัวต่อกันในบรรทัดเดียว**:
  ตัวแรกจัดการความล้มเหลวของ `field_value(...)` (ที่คืน `ConfigParseError` อยู่แล้วโดยตรง ไม่ต้องแปลง) ตัวที่สอง
  จัดการความล้มเหลวของ `.parse()` (ที่คืน `ParseIntError` ซึ่งต้องแปลงเป็น `ConfigParseError` ผ่าน `impl From`
  ที่เราเขียนไว้ด้านบน) — โค้ดบรรทัดเดียวนี้แทนที่ `match` ซ้อนกัน 2 ชั้นในเวอร์ชัน verbose ได้ทั้งหมด
- เราเขียน `impl From<ParseIntError>` และ `impl From<ParseBoolError>` ไว้**ครั้งเดียว**ที่ระดับ type
  `ConfigParseError` แล้วใช้ `?` ได้อย่างอิสระในทุกฟังก์ชันที่คืน `Result<_, ConfigParseError>` ทั่วทั้งโปรแกรม
  โดยไม่ต้องเขียน error handling ซ้ำ ๆ ทุกจุดที่ parse ตัวเลข/boolean — นี่คือประโยชน์ระยะยาวของการลงทุนเขียน
  `impl From` เมื่อโปรแกรมมีหลายจุดที่ต้อง parse ชนิดข้อมูลเดียวกันซ้ำ ๆ
- ฟังก์ชัน `parse_config` เวอร์ชัน `?` เหลือแค่ **4 บรรทัดของ logic จริง** (เช็คจำนวน field, parse port, parse
  verbose, สร้าง `Config`) เทียบกับเวอร์ชัน verbose ที่มีทั้ง logic จริงและ boilerplate การจัดการ error ปนกันอยู่
  ตลอดทั้งฟังก์ชัน — นี่คือประโยชน์ที่จับต้องได้ของ `?`: **โค้ดที่เหลืออยู่คือ "สิ่งที่โปรแกรมทำจริง ๆ" ไม่ใช่
  "วิธีจัดการกับความล้มเหลวของแต่ละขั้นตอน"** ซึ่งทำให้อ่านและ maintain ได้ง่ายขึ้นมากในระบบที่มีหลาย function
  ที่ต้อง propagate error แบบนี้

### 12.11 มองไปข้างหน้า: `thiserror` และ `anyhow`

ตัวอย่าง `ConfigParseError` ที่เขียนไว้ในหัวข้อก่อนต้องเขียน `struct`, `impl fmt::Display`, `impl Error`, และ
`impl From` เองทั้งหมดด้วยมือ — ซึ่งเริ่มมีโค้ด boilerplate ที่ซ้ำ ๆ กันไม่น้อย (โดยเฉพาะ `impl fmt::Display` ที่
เขียนคล้ายกันมากในทุก error type) และการใช้ `Box<dyn Error>` ก็แลกมาด้วยการเสียข้อมูลชนิดที่แน่นอนไปตามที่อธิบาย
ไว้ในหัวข้อ 12.7

Rust ecosystem มี 2 crate ที่ได้รับความนิยมสูงมากสำหรับแก้ปัญหาทั้งสองด้านนี้ ซึ่งเราจะเจาะลึกแบบเต็มรูปแบบใน
**Part 30-31**:

- **`thiserror`**: ช่วยลด boilerplate ของการสร้าง custom error type (แบบ `ConfigParseError` ข้างบน) ด้วย
  `#[derive(thiserror::Error)]` — เขียน `impl Display`/`impl Error`/`impl From` ให้อัตโนมัติจาก attribute ที่
  ประกาศไว้บน enum เพียงไม่กี่บรรทัด เหมาะมากสำหรับเขียน **library** ที่ผู้เรียกต้องแยกกรณี error ได้อย่างแม่นยำ
- **`anyhow`**: ให้ type `anyhow::Error` ที่ทำงานคล้าย `Box<dyn Error>` (เก็บ error อะไรก็ได้โดยไม่ต้องรู้ชนิด
  ล่วงหน้า) แต่สะดวกกว่ามาก มี method เสริมอย่าง `.context()` ที่แนบข้อความอธิบายเพิ่มเติมไปกับ error ตอน
  propagate ผ่านหลายชั้น เหมาะมากสำหรับเขียน **application** (เช่น `main()` และโค้ดระดับบนของโปรแกรม) ที่ไม่
  จำเป็นต้องแยกกรณี error อย่างละเอียดเท่า library

รูปแบบที่พบบ่อยที่สุดในโปรแกรม Rust จริงคือ: **library ใช้ `thiserror` สร้าง error type ของตัวเอง ส่วน
application ที่เรียกใช้ library หลายตัวรวมกัน ใช้ `anyhow::Error` (หรือ `Box<dyn Error>`) เป็น error type กลาง
ของ `main()`** — แต่สำหรับตอนนี้ **`Box<dyn Error>` ที่เรียนไปแล้วในหัวข้อ 12.7-12.8 ก็เพียงพอสมบูรณ์แบบสำหรับ
งานส่วนใหญ่ที่คุณจะเจอในหลักสูตรจนถึง Part 30** ไม่ต้องรีบไปติดตั้ง crate เพิ่มตอนนี้

### 12.12 `panic!` แบบเจาะลึก: Unwinding, Abort, และ `std::process::exit`

กลับมาเจาะลึกฝั่ง unrecoverable error ให้ครบถ้วนยิ่งขึ้น เพราะ `panic!` มีรายละเอียดที่ควรรู้มากกว่าที่เห็นในหัวข้อ
12.4

#### `panic!()` macro พื้นฐาน

`panic!()` คือ macro ที่คุณเรียกเองได้ตรง ๆ เมื่อต้องการหยุดโปรแกรมทันทีเพราะเจอสถานะที่ไม่ควรเกิดขึ้น:

```rust
fn charge_credit_card(amount_cents: i64) {
    if amount_cents < 0 {
        // amount ติดลบคือ "บั๊กของโค้ดผู้เรียก" ไม่ใช่ข้อมูลจากผู้ใช้ที่ผิดพลาดตามปกติ
        panic!("จำนวนเงินต้องไม่ติดลบ แต่ได้รับ {amount_cents}");
    }
    println!("เก็บเงิน {amount_cents} สตางค์สำเร็จ");
}

fn main() {
    charge_credit_card(500);
    charge_credit_card(-100);
}
```

ผลลัพธ์:

```
เก็บเงิน 500 สตางค์สำเร็จ

thread 'main' panicked at src/main.rs:4:9:
จำนวนเงินต้องไม่ติดลบ แต่ได้รับ -100
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

`panic!()` รับ argument แบบเดียวกับ `format!()`/`println!()` (รองรับ `{}`/`{variable}` แบบ interpolation) และเมื่อ
ถูกเรียก โปรแกรมจะพิมพ์ข้อความไปที่ stderr พร้อมตำแหน่งที่เกิด แล้วเริ่มกระบวนการที่เรียกว่า **unwinding**

#### Unwinding: พฤติกรรม default ของ panic

เมื่อ panic เกิดขึ้น Rust (ใน mode default) จะทำ **stack unwinding** — คือการ "ไล่คลาย" call stack กลับขึ้นไป
ทีละ stack frame โดย **เรียก destructor (`Drop::drop`) ของทุกตัวแปรที่ยังอยู่ใน scope ตามลำดับ** ก่อนจะปิดโปรแกรม
จริง ๆ ทำให้ resource ต่าง ๆ (ไฟล์ที่เปิดไว้, memory ที่ allocate ไว้, lock ที่ถืออยู่) ถูกคืนกลับอย่างถูกต้อง
แม้ในสถานการณ์ที่โปรแกรมกำลังจะพัง — นี่เป็นข้อดีของ unwinding: **แม้จะ panic โปรแกรมก็ยัง "cleanup ตัวเองอย่าง
สุภาพ"** ก่อนจบการทำงาน (ต่างจาก crash แบบดิบ ๆ ในภาษาที่ไม่มีกลไกนี้)

ลองดูตัวอย่างที่แสดงให้เห็นว่า `Drop` ทำงานแม้ตอนโปรแกรมจบแบบปกติ:

```rust
struct Receipt {
    id: u32,
}

impl Drop for Receipt {
    fn drop(&mut self) {
        println!("บันทึกใบเสร็จ #{} ลงไฟล์ก่อนปิดโปรแกรม (Drop ทำงาน)", self.id);
    }
}

fn main() {
    let _receipt = Receipt { id: 1 };
    println!("จบการทำงานตามปกติ กำลังจะออกจาก main()");
    // ออกจาก scope ปกติ -> Drop ของ _receipt ทำงานแน่นอน
}
```

ผลลัพธ์:

```
จบการทำงานตามปกติ กำลังจะออกจาก main()
บันทึกใบเสร็จ #1 ลงไฟล์ก่อนปิดโปรแกรม (Drop ทำงาน)
```

(เรื่อง `Drop` และ scope เต็มรูปแบบเรียนมาแล้วใน Part 6) ถ้าคุณแทนที่ `println!("จบการทำงาน...")` ด้วย `panic!(...)`
ในตัวอย่างนี้ `Drop::drop` ของ `_receipt` ก็ยังจะถูกเรียกอยู่ดี **ก่อน** โปรแกรมจะปิดตัวจริง เพราะ unwinding ไล่
cleanup ทุก scope ที่ผ่านระหว่างทางกลับขึ้นไป

#### `panic = "abort"`: ทางเลือกที่ข้าม unwinding ไปเลย

Unwinding มีต้นทุน (ทั้งขนาดของ binary ที่ต้องเก็บข้อมูลสำหรับ "รู้วิธี cleanup" ของทุกจุดในโค้ด และเวลาที่ใช้ไล่
คลาย stack) ในบางสถานการณ์ (เช่น embedded system ที่มี memory จำกัดมาก, หรือโปรแกรมที่อยากให้ binary มีขนาดเล็ก
ที่สุด) นักพัฒนาอาจต้องการให้ panic **จบโปรแกรมทันที** โดยไม่เสียเวลา cleanup เลย — Rust รองรับสิ่งนี้ผ่าน
การตั้งค่าใน `Cargo.toml`:

```toml
[profile.release]
panic = "abort"
```

เมื่อตั้งค่านี้ ทุก panic ใน build โหมด release จะเรียก `abort()` ของระบบปฏิบัติการทันที (ปิดโปรแกรมทันทีแบบ
ดิบ ๆ) **โดยไม่เรียก `Drop::drop` ของตัวแปรใด ๆ เลย** ไม่มีการ cleanup ใด ๆ ทั้งสิ้น แลกกับ binary ที่มีขนาดเล็กลง
และ panic ที่เกิดขึ้นเร็วขึ้นเล็กน้อย (ไม่ต้องเสียเวลาไล่ unwind) การตั้งค่านี้เหมาะกับโปรแกรมที่ยอมรับได้ว่า "ถ้า
panic ก็ไม่ต้องพยายาม cleanup อะไรแล้ว เพราะโปรแกรมกำลังจะปิดอยู่ดี" ซึ่งเป็นทางเลือกทาง engineering ที่ต้องชั่ง
น้ำหนักตามบริบทของแต่ละโปรเจกต์ (เราจะเจาะลึกเรื่อง Cargo profile แบบเต็มรูปแบบในบทที่เกี่ยวกับการจัดการ project
ขนาดใหญ่ในโมดูลถัด ๆ ไป)

#### `std::process::exit()`: ทางเลือกที่คุณสั่งเอง แทน `panic!`

Rust มีอีกวิธีในการจบโปรแกรมทันทีที่**ไม่ใช่** panic เลย นั่นคือ `std::process::exit(code)` — เรียกได้จากที่ไหนก็
ได้ในโปรแกรม จบโปรแกรมทันทีด้วย exit code ที่คุณกำหนด **โดยข้าม unwinding ไปเลยเหมือนกับ `panic = "abort"`**
(ไม่ว่าโปรเจกต์จะตั้งค่า `panic = "unwind"` หรือ `"abort"` ก็ตาม `process::exit()` ข้าม `Drop` เสมอ) ลองเทียบ
โค้ดสองเวอร์ชันนี้ดู:

```rust
use std::process;

struct Receipt {
    id: u32,
}

impl Drop for Receipt {
    fn drop(&mut self) {
        println!("บันทึกใบเสร็จ #{} ลงไฟล์ก่อนปิดโปรแกรม (Drop ทำงาน)", self.id);
    }
}

fn main() {
    let _receipt = Receipt { id: 1 };
    println!("กำลังจะเรียก process::exit(0) แบบฉุกเฉิน");
    process::exit(0);
    // โค้ดหลังจุดนี้ไม่มีวันถูกรัน และ Drop ของ _receipt จะไม่ถูกเรียกเลย
}
```

ผลลัพธ์ (สังเกตให้ดี — ข้อความจาก `Drop` **ไม่ปรากฏเลย**):

```
กำลังจะเรียก process::exit(0) แบบฉุกเฉิน
```

เทียบกับเวอร์ชันที่ออกจาก `main()` ตามปกติในหัวข้อก่อน ที่ข้อความจาก `Drop` ปรากฏครบถ้วน — นี่คือความต่างที่สำคัญ
ที่สุดระหว่างการ "ออกจาก scope ตามปกติ" (หรือแม้แต่ panic แบบ unwind) กับ `process::exit()`: **`process::exit()`
ไม่รอให้ destructor ทำงานเลย ไม่ว่ากรณีใดก็ตาม** ซึ่งเป็นเรื่องสำคัญมากที่ต้องระวัง — เราจะเจาะลึกอันตรายของสิ่งนี้
ในหัวข้อ "กับดักที่พบบ่อย" ด้านล่าง

**เมื่อไหร่ควรใช้ `process::exit()` แทน `panic!()` หรือ `return Err(...)` จาก `main()`:**

- เมื่อต้องการ **กำหนด exit code เอง** ที่ไม่ใช่แค่ `0`/`1` (เช่น โปรแกรม CLI ที่ตกลงกับระบบภายนอกว่า exit code
  `2` แปลว่า "ไฟล์ config ไม่พบ", `3` แปลว่า "สิทธิ์ไม่พอ" เป็นต้น) — `panic!()` และ `main() -> Result<(),
  E>` ให้คุณควบคุม exit code ได้จำกัดกว่านี้มาก (`panic!` จบด้วย code `101` เสมอ, `Err` จาก `main()` จบด้วย
  code `1` เสมอ)
- เมื่อ**ตั้งใจ**ให้โปรแกรมจบทันทีจากที่ไหนก็ได้ในโค้ด โดยไม่ต้องพึ่ง unwinding ผ่านหลายชั้นฟังก์ชัน (บางครั้งใช้ใน
  ฟังก์ชันที่ handle signal หรือ error ระดับวิกฤตที่ตัดสินใจแล้วว่า **ไม่ต้องการ** cleanup ใด ๆ)
- **ไม่ควร**ใช้แทน `panic!`/`Result` เป็นค่า default ทั่วไป เพราะการข้าม `Drop` ไปดื้อ ๆ อาจทำให้ resource สำคัญ
  (ไฟล์ที่ยังไม่ flush, การเชื่อมต่อ database ที่ยังไม่ปิดอย่างถูกต้อง, lock ที่ยังไม่ปลด) หลุดค้างอยู่ในสถานะที่
  ไม่ถูกต้อง — จะอธิบายเป็นตัวอย่างที่จับต้องได้ในหัวข้อถัดไป

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. ใช้ `?` ในฟังก์ชันที่ไม่ได้ประกาศ return type เป็น `Result`/`Option`

```rust
fn print_doubled(input: &str) {
    let number: i32 = input.parse()?; // ใช้ ? ในฟังก์ชันที่ return () ไม่ใช่ Result/Option
    println!("{}", number * 2);
}

fn main() {
    print_doubled("21");
}
```

Error ที่ได้:

```
error[E0277]: the `?` operator can only be used in a function that returns `Result` or `Option` (or another type that implements `FromResidual`)
 --> src/main.rs:2:36
  |
1 | fn print_doubled(input: &str) {
  | ----------------------------- this function should return `Result` or `Option` to accept `?`
2 |     let number: i32 = input.parse()?; // ใช้ ? ในฟังก์ชันที่ return () ไม่ใช่ Result/Option
  |                                    ^ cannot use the `?` operator in a function that returns `()`
  |
help: consider adding return type
  |
1 ~ fn print_doubled(input: &str) -> Result<(), Box<dyn std::error::Error>> {
2 |     let number: i32 = input.parse()?; // ใช้ ? ในฟังก์ชันที่ return () ไม่ใช่ Result/Option
3 |     println!("{}", number * 2);
4 +     Ok(())
  |
```

**สาเหตุ**: จากหัวข้อ 12.6 เราอธิบายว่า `?` desugar เป็น `return Err(...)` (หรือ `return None` สำหรับ `Option`)
— ถ้าฟังก์ชันประกาศว่าคืน `()` แล้วมี `return Err(...)` แทรกอยู่ ก็ขัดแย้งกับ return type ที่ประกาศไว้ตรง ๆ
compiler จึงต้องปฏิเสธ **วิธีแก้**: เปลี่ยน return type ของฟังก์ชันให้เป็น `Result<T, E>` (หรือ `Option<T>`) ที่
เหมาะสม — สังเกตว่า compiler ในตัวอย่างนี้ **เสนอวิธีแก้ให้ตรงกับ pattern ที่เราสอนไปแล้วในหัวข้อ 12.8 เป๊ะ**
(`Result<(), Box<dyn std::error::Error>>` พร้อม `Ok(())` ปิดท้าย) ซึ่งยืนยันว่านี่คือ pattern ที่ Rust เองก็แนะนำ
เป็นค่า default เมื่อไม่มีข้อมูลอื่นเพิ่มเติม

### 2. ใช้ `?` ข้าม Error Type ที่ไม่มีทางแปลงกันได้ (ไม่มี `From`)

```rust
#[derive(Debug)]
struct ConfigError {
    message: String,
}

fn parse_port(raw: &str) -> Result<u16, ConfigError> {
    let port = raw.parse::<u16>()?;
    Ok(port)
}

fn main() {
    println!("{:?}", parse_port("8080"));
}
```

Error ที่ได้ (อธิบายละเอียดแล้วในหัวข้อ 12.7):

```
error[E0277]: `?` couldn't convert the error to `ConfigError`
 --> src/main.rs:7:34
  |
6 | fn parse_port(raw: &str) -> Result<u16, ConfigError> {
  |                             ------------------------ expected `ConfigError` because of this
7 |     let port = raw.parse::<u16>()?;
  |                    --------------^ the trait `From<ParseIntError>` is not implemented for `ConfigError`
```

**สาเหตุ**: `?` ต้องเรียก `ConfigError::from(parse_int_error)` โดยอัตโนมัติ แต่ `ConfigError` ไม่มี `impl
From<ParseIntError>` ให้ **วิธีแก้**: เขียน `impl From<ParseIntError> for ConfigError` เอง (หัวข้อ 12.7) หรือ
เปลี่ยน return type ของฟังก์ชันเป็น `Result<u16, Box<dyn Error>>` ที่รับ error หลายชนิดได้อัตโนมัติ กับดักนี้พบบ่อย
มากตอนเริ่มเขียนฟังก์ชันที่เรียกหลายฟังก์ชันย่อยที่คืน error คนละชนิดกัน แล้วลืมสอนให้ error type หลักของตัวเอง
"รู้จัก" แปลงจาก error ย่อยพวกนั้น

### 3. `.unwrap_or()` ประเมิน argument ทันทีเสมอ แม้ค่านั้นจะไม่ถูกใช้ — ต่างจาก `.unwrap_or_else()`

```rust
fn expensive_default() -> i32 {
    println!("...กำลังคำนวณค่า default (ทำงานหนัก)...");
    0
}

fn main() {
    let good: Result<i32, String> = Ok(99);

    // unwrap_or รับ "ค่า" ตรง ๆ -> argument ถูก evaluate ทันทีเสมอ ไม่ว่า Result จะเป็น Ok หรือไม่
    println!("ทดสอบ unwrap_or:");
    let v1 = good.clone().unwrap_or(expensive_default());
    println!("ได้ {v1}\n");

    // unwrap_or_else รับ "closure" -> closure ถูกเรียกเฉพาะตอนเป็น Err เท่านั้น
    println!("ทดสอบ unwrap_or_else:");
    let v2 = good.unwrap_or_else(|_| expensive_default());
    println!("ได้ {v2}");
}
```

ผลลัพธ์:

```
ทดสอบ unwrap_or:
...กำลังคำนวณค่า default (ทำงานหนัก)...
ได้ 99

ทดสอบ unwrap_or_else:
ได้ 99
```

**สาเหตุ**: `good` เป็น `Ok(99)` ทั้งคู่ — ค่า `expensive_default()` จึงไม่ควรต้องถูกเรียกเลยในทางทฤษฎี แต่สังเกตว่า
ตอนเรียก `.unwrap_or(expensive_default())` ข้อความ `"...กำลังคำนวณค่า default..."` **ก็ยังพิมพ์ออกมา** เพราะ Rust
ประเมิน argument ของฟังก์ชัน/method **ก่อน** ที่จะเรียก method นั้นเสมอ (evaluation แบบ eager ตามปกติของภาษา) —
`.unwrap_or(x)` รับ **ค่าที่คำนวณเสร็จแล้ว** เป็น argument ดังนั้น `x` (ในที่นี้คือ `expensive_default()`) จึงถูก
เรียกก่อนเสมอไม่ว่าผลลัพธ์จะถูกใช้จริงหรือไม่ ส่วน `.unwrap_or_else(f)` รับ **closure** ที่ยังไม่ได้ถูกเรียก จะถูก
เรียกก็ต่อเมื่อ method ตัดสินใจแล้วว่าจำเป็นต้องใช้ค่า default จริง ๆ (คือเมื่อเป็น `Err` เท่านั้น) จึงเห็นว่าเวอร์ชัน
`.unwrap_or_else()` ไม่พิมพ์ข้อความ "กำลังคำนวณ" เลย **วิธีแก้/หลักปฏิบัติ**: ถ้าค่า default คำนวณได้เร็วและไม่มี
side effect (เช่น literal ธรรมดา `.unwrap_or(0)`) ใช้ `.unwrap_or()` ได้สบาย ๆ อ่านง่ายกว่า แต่ถ้าค่า default
ต้องเรียกฟังก์ชันที่ทำงานหนัก (เช่น query database, อ่านไฟล์, allocate memory ขนาดใหญ่) หรือมี side effect (เช่น
`println!` เหมือนตัวอย่างนี้) **ต้องใช้ `.unwrap_or_else()` เสมอ** เพื่อไม่ให้งานนั้นถูกทำแบบสิ้นเปลืองเมื่อ `Result`
เป็น `Ok` อยู่แล้ว — clippy (linter จาก Part 5) มี lint ชื่อ `unwrap_or` ที่ช่วยเตือนกรณีนี้ได้บางส่วนถ้า argument
ที่ส่งเข้าไปมีรูปแบบเป็นการเรียกฟังก์ชันตรง ๆ

### 4. `std::process::exit()` ข้าม `Drop` ไปเสมอ — ทำให้ resource ไม่ถูก cleanup

```rust
use std::process;

struct Receipt {
    id: u32,
}

impl Drop for Receipt {
    fn drop(&mut self) {
        println!("บันทึกใบเสร็จ #{} ลงไฟล์ก่อนปิดโปรแกรม (Drop ทำงาน)", self.id);
    }
}

fn main() {
    let _receipt = Receipt { id: 1 };
    println!("กำลังจะเรียก process::exit(0) แบบฉุกเฉิน");
    process::exit(0);
}
```

ผลลัพธ์ (ไม่มีบรรทัด "บันทึกใบเสร็จ..." เลย ตามที่แสดงไว้แล้วในหัวข้อ 12.12):

```
กำลังจะเรียก process::exit(0) แบบฉุกเฉิน
```

**สาเหตุ**: นี่**ไม่ใช่ compiler error หรือ panic** แต่เป็น **behavioral pitfall** ที่ compiler ไม่มีทางเตือนคุณ
ได้เลย เพราะโค้ด compile ผ่านสมบูรณ์และรันโดยไม่มี error ใด ๆ ในหน้าจอ — ความเสียหายที่เกิดคือความเงียบ: ถ้า
`Receipt` ตัวนี้เป็นตัวแทนของ resource สำคัญ (เช่น `BufWriter` ที่ยังไม่ `.flush()` ข้อมูลลงไฟล์, connection ไป
database ที่ยังไม่ commit transaction, lock ที่ยังไม่ปลด) การเรียก `process::exit()` ก่อนที่ resource เหล่านี้จะ
ถูก `Drop` อย่างถูกต้อง อาจทำให้ **ข้อมูลสูญหายอย่างเงียบ ๆ โดยไม่มี error แจ้งเลย** ซึ่งเป็นบั๊กที่ตรวจจับยากมาก
เพราะไม่มี panic, ไม่มี error message ใด ๆ ทั้งสิ้น มีแค่ "ผลลัพธ์ที่ควรเกิดแต่ไม่เกิด" **วิธีแก้/หลักปฏิบัติ**:
ก่อนเรียก `process::exit()` ให้ตรวจสอบเสมอว่าไม่มี resource สำคัญที่ยังรอ cleanup อยู่ (เรียก `.flush()`,
`.commit()`, หรือปิด resource ด้วยมือก่อน) หรือพิจารณาใช้ `return Err(...)` จาก `fn main() -> Result<(), E>`
(หัวข้อ 12.8) แทน เพราะการ `return` ตามปกติ (แม้จาก `main()`) ยังคง unwind ผ่าน scope ต่าง ๆ และเรียก `Drop`
ให้ครบถ้วนก่อนโปรแกรมจะปิดจริง

### 5. เขียนข้อความ `.expect()` ผิดทาง — อธิบาย error ซ้ำ แทนอธิบายสมมติฐาน

```rust
fn find_user_age(username: &str) -> Result<u32, String> {
    if username == "somchai" {
        Ok(30)
    } else {
        Err(format!("user '{username}' not found"))
    }
}

fn main() {
    // ตัวอย่างข้อความที่ "ผิดทาง" — ซ้ำกับสิ่งที่ panic message จะแสดงอยู่แล้ว
    let bad = find_user_age("unknown").expect("user not found");

    println!("{bad}");
}
```

panic message ที่ได้ (ถ้ารันจริง จะ panic ก่อนถึงบรรทัด `println!`):

```
thread 'main' panicked at src/main.rs:11:40:
user not found: "user 'unknown' not found"
```

**สาเหตุ**: ข้อความที่ใส่ใน `.expect("user not found")` **ซ้ำซ้อน**กับข้อมูลที่ `Err` มีอยู่แล้ว (`"user 'unknown'
not found"`) ทำให้ผู้ที่อ่าน log ตอน production panic เห็นข้อความ `user not found: "user 'unknown' not found"`
ซึ่งไม่ได้ให้ข้อมูลอะไรเพิ่มเติมเลยเหนือกว่าสิ่งที่ `Err` บอกอยู่แล้ว **วิธีแก้/หลักปฏิบัติ**: ตามที่อธิบายไว้ใน
หัวข้อ 12.5 ข้อความใน `.expect()` ควรอธิบาย **"สมมติฐานที่คุณตั้งไว้ ณ จุดนี้ในโค้ด"** ไม่ใช่คาดเดา/ทวนซ้ำสิ่งที่
error บอกอยู่แล้ว เช่น เปลี่ยนเป็น `.expect("ควรมี user 'somchai' อยู่ในระบบเสมอเพราะสร้างไว้ตอน seed data")` —
เมื่อ panic เกิดขึ้นจริง ผู้ที่อ่าน log จะได้เห็นทั้ง **"สมมติฐานที่ผิดไปจากที่คิด"** (จากข้อความของคุณ) **และ**
**"รายละเอียด error จริง"** (จาก `Err` ที่แสดงต่อท้ายอัตโนมัติ) ในข้อความเดียว ซึ่งช่วย debug ได้เร็วกว่าการอธิบาย
ซ้ำสิ่งเดียวกันสองครั้งมาก

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** เขียนฟังก์ชัน `fn parse_positive(input: &str) -> Result<i32, String>` ที่ทำ 2 อย่าง: (1) parse
   `input` เป็น `i32` และ (2) ตรวจสอบว่าค่าที่ได้ต้องเป็นบวก (`> 0`) ถ้า parse ไม่สำเร็จหรือค่าไม่เป็นบวก ให้คืน
   `Err` ที่มีข้อความอธิบายสาเหตุที่ชัดเจน (ใช้ `match` แบบเต็มรูปแบบ ไม่ใช้ `?` ในข้อนี้ เพื่อฝึกความเข้าใจ
   โครงสร้างพื้นฐานก่อน) ทดสอบด้วย input 3 แบบ: `"42"` (ควรได้ `Ok(42)`), `"-5"` (ควรได้ `Err` เพราะไม่เป็นบวก),
   และ `"abc"` (ควรได้ `Err` เพราะ parse ไม่สำเร็จ) — hint: `.parse::<i32>()` คืน `Result<i32, ParseIntError>`
   ต้อง `match` แยกกรณีให้ครบทั้ง "parse ไม่สำเร็จ" และ "parse สำเร็จแต่ค่าไม่เป็นบวก"

2. **[กลาง]** เขียนฟังก์ชัน `parse_positive` ข้อ 1 **ใหม่** โดยใช้ `?` operator แทน `match` ให้ได้มากที่สุด (hint:
   ส่วน "parse ไม่สำเร็จ" ใช้ `?` ได้ตรง ๆ ถ้า error type ตรงกัน ส่วน "ค่าไม่เป็นบวก" ยังต้องเขียน `if`/`return
   Err(...)` เองเพราะไม่มี method สำเร็จรูปที่คืน `Result` ให้ตรวจสอบเงื่อนไขแบบนี้) เปรียบเทียบความยาวและความอ่าน
   ง่ายระหว่างสองเวอร์ชัน แล้วลองเปลี่ยน error type ของฟังก์ชันจาก `String` เป็น custom struct ของคุณเอง (เช่น
   `struct PositiveParseError(String)`) แล้วดูว่าต้องแก้อะไรเพิ่มเพื่อให้ `?` ยังใช้งานได้ (ทวนหัวข้อ 12.7)

3. **[ยาก]** ขยายตัวอย่าง config parser จากหัวข้อ 12.10 ให้รองรับ field ที่สาม: `timeout_seconds=<ตัวเลข>` (เช่น
   input เต็มรูปแบบ `"port=8080,verbose=true,timeout_seconds=30"`) โดยต้องแก้ทั้ง struct `Config` (เพิ่ม field
   `timeout_seconds: u32`) และฟังก์ชัน `parse_config` ให้ parse field ที่สามด้วย `?` เหมือน field อื่น ๆ (hint:
   เพราะ `impl From<ParseIntError> for ConfigParseError` มีอยู่แล้วจากหัวข้อ 12.10 การเพิ่ม field ที่เป็นตัวเลข
   อีกตัวไม่ต้องเขียน `impl From` เพิ่มเลย — สังเกตดูว่าทำไม) ทดสอบด้วย input ที่ถูกต้อง, input ที่ field ที่สาม
   parse ไม่ผ่าน, และ input ที่ขาด field ที่สามไปเลย

4. **[ยาก/ประยุกต์ใช้งานจริง]** ออกแบบระบบถอนเงินจากบัญชีธนาคารง่าย ๆ ที่มี 3 ฟังก์ชันเรียงเป็น chain:
   `fn validate_amount(raw: &str) -> Result<u64, Box<dyn Error>>` (parse string เป็นจำนวนเงิน ต้องเป็นค่าบวก),
   `fn check_balance(account_balance: u64, amount: u64) -> Result<(), Box<dyn Error>>` (เช็คว่ายอดเงินพอไหม คืน
   custom error ของคุณเองถ้าไม่พอ ที่ implement `std::error::Error`), และ `fn withdraw(account_balance: u64, raw_amount:
   &str) -> Result<u64, Box<dyn Error>>` ที่เรียกทั้งสองฟังก์ชันข้างบนด้วย `?` แล้วคืนยอดเงินที่เหลือ จากนั้นเขียน
   `fn main() -> Result<(), Box<dyn Error>>` ที่เรียก `withdraw` และใช้ `?` เพื่อ propagate error ออกไปให้ runtime
   จัดการเอง (ตามหัวข้อ 12.8) ลองรันด้วย input ที่ทำให้ล้มเหลวทั้งจาก `validate_amount` (เช่น `"abc"` หรือ `"-5"`)
   และจาก `check_balance` (ยอดไม่พอ) แล้วสังเกต exit code ของโปรแกรมด้วยคำสั่ง `echo $?` หลังรัน (บน Linux/macOS)
   ว่าเป็น `1` ทั้งสองกรณีหรือไม่ (hint: ต้องออกแบบ custom error สำหรับ "ยอดเงินไม่พอ" ให้ implement
   `std::fmt::Display`, `std::fmt::Debug`, และ `std::error::Error` เพื่อให้แปลงเข้ากับ `Box<dyn Error>` ได้ผ่าน
   `?` — ทวนหัวข้อ 12.7 เรื่อง `Box<dyn Error>` ที่ implement `From<E>` ให้ทุก `E: Error`)

## สรุป

บทนี้เริ่มจากคำถามพื้นฐานที่สุดของการเขียนโปรแกรม: "เมื่อฟังก์ชันล้มเหลว โปรแกรมควรรู้เรื่องนี้ได้อย่างไร" เราเทียบ
แนวทาง exception-based ของ Python/Java (ที่ error "มองไม่เห็น" ใน function signature และไม่มีอะไรบังคับให้ handle)
กับแนวทางของ Rust ที่ยึดหลัก **"ฟังก์ชันที่ล้มเหลวได้ต้องประกาศไว้ใน return type"** ผ่าน enum `Result<T, E>` ที่
เป็น enum ธรรมดา (มี `Ok(T)`/`Err(E)` สองตัว) ซึ่งใช้กลไก `match` และ exhaustiveness checking แบบเดียวกับที่เรียน
มาแล้วใน Part 10 ทุกประการ — ความปลอดภัยของ error handling ใน Rust จึงไม่ใช่ feature พิเศษแยกต่างหาก แต่มาจาก
การนำ enum ที่แข็งแรงอยู่แล้วมาใช้แก้ปัญหานี้โดยเฉพาะ

เราแยกแยะ **recoverable error** (`Result` — สำหรับเหตุการณ์ปกติที่คาดหวังว่าจะเกิดขึ้นได้) กับ **unrecoverable
error** (`panic!` — สำหรับสถานะที่ไม่ควรเกิดขึ้นได้เลย/บั๊ก) และเจาะลึก `.unwrap()`/`.expect()` ที่ panic message
มี `Debug` ของ `Err` แทรกอยู่ (ต่างจาก `Option` ที่ไม่มีข้อมูลให้แสดง) จุดศูนย์กลางของบทนี้คือ **`?` operator** ที่
เราอธิบายแบบละเอียดว่ามันคือ syntactic sugar ที่ desugar เป็น `match { Ok(v) => v, Err(e) => return
Err(From::from(e)) }` — **ไม่มีต้นทุน runtime เพิ่มเติมเลย** ต่างจากกลไก exception/stack unwinding เพื่อค้นหา
handler ของภาษาอื่นอย่างสิ้นเชิง เราเห็นบทบาทของ trait `From` ที่ทำให้ `?` แปลงชนิด error ข้ามกันได้อัตโนมัติ
(หรือ compile ไม่ผ่านด้วย `E0277` ถ้าไม่มีทางแปลง) และสองทางออกที่ใช้ได้จริง: เขียน `impl From` เอง หรือใช้
`Box<dyn Error>` เป็นทางลัด pragmatic (พร้อมข้อแลกเปลี่ยนที่ต้องเข้าใจ) รวมถึง pattern `fn main() ->
Result<(), Box<dyn Error>>` ที่ทำให้ `main()` ใช้ `?` ได้ตรง ๆ และได้ exit code ที่ถูกต้องโดยอัตโนมัติ ปิดท้ายด้วย
combinator ของ `Result` ที่คู่กับของ `Option`, ตัวอย่าง config parser ที่แสดงความต่างระหว่าง `match` ซ้อนกันกับ
`?` แบบจับต้องได้ และรายละเอียดเชิงลึกของ `panic!` (unwinding, `panic = "abort"`, และ `process::exit()` ที่ข้าม
`Drop` ไปเสมอ)

ใน **Part 13** เราจะย้ายจากพื้นฐานภาษาไปสู่ **Collections** เริ่มด้วย `Vec<T>` — โครงสร้างข้อมูลที่เก็บค่าหลายตัว
ที่มี type เดียวกันแบบ dynamic (ขยาย/หดขนาดได้ตอน runtime ต่างจาก array ที่ขนาดคงที่ตั้งแต่ Part 3) ซึ่งเป็น
โครงสร้างข้อมูลที่ใช้บ่อยที่สุดในโปรแกรม Rust จริง — และคุณจะเห็น `Result`/`Option` ที่เรียนมาจากสองบทนี้ปรากฏอยู่
ใน method ของ `Vec<T>` หลายตัว (เช่น `.get()` ที่คืน `Option` แทนการ panic ตรง ๆ แบบ indexing operator `[]`)
ซึ่งเป็นเครื่องยืนยันว่าทั้ง `Option<T>` และ `Result<T, E>` ที่เรียนมาจนถึงตอนนี้คือรากฐานที่ Rust standard library
ใช้ทั่วทั้งระบบ ไม่ใช่แค่ concept แยกเดี่ยว

---

**Part ก่อนหน้า:** [Option<T> และ Null Safety](part-011-option-and-null-safety.md) | **Part ถัดไป:** [Collections: Vec<T>](part-013-vec.md)
