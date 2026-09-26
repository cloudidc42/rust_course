# Part 52: Design Patterns ใน Rust (Builder, Strategy, Observer)

> โมดูล: Design Patterns และสถาปัตยกรรมโค้ด | ระดับ: สูง | เวลาโดยประมาณ: 240 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้ว่า **Design Pattern แบบ Gang of Four (GoF)** ถูกออกแบบมาสำหรับภาษา class-based OOP (Java/C++) และ
  รู้ว่า pattern ไหน "ย้ายมาตรง ๆ" ได้ใน Rust, pattern ไหนต้อง **ตีความใหม่ (reinterpret)** เพราะ Rust ไม่มี
  inheritance แต่มี ownership/borrowing และระบบ trait+enum ที่ทรงพลังกว่า
- ออกแบบและเขียน **Builder pattern** แบบเต็มรูปแบบด้วยมือ ทั้งแบบ consuming builder (`self -> Self`) และแบบ
  `&mut self` builder พร้อมอธิบาย tradeoff ของทั้งสองแบบได้อย่างถูกต้อง, ทำ validation ตอน `.build()` ที่คืน
  `Result<T, BuilderError>` ได้จริง และเข้าใจว่า `#[derive(Builder)]` จาก Part 45 automate ขั้นตอนไหนให้เราบ้าง
- อธิบายแนวคิด **typestate-adjacent builder** (ใช้ type ต่างกันแทนสถานะ "ยังไม่ครบ" กับ "พร้อม build") ในระดับ
  เกริ่นนำ ก่อนไปเจาะลึกเต็มรูปแบบใน Part 53
- ออกแบบ **Strategy pattern** ได้ทั้งสามแบบ — `Box<dyn Trait>` (dynamic dispatch), generic + trait bound
  (static dispatch), และ closure parameter (`Fn`) — พร้อมเลือกแบบที่เหมาะสมกับสถานการณ์ได้อย่างมีเหตุผล
  โดยอ้างอิงกรอบคิด dispatch จาก Part 21 โดยตรง
- ออกแบบ **Observer pattern** แบบดั้งเดิมด้วย `Vec<Box<dyn Observer>>` และรู้จักทางเลือกที่ "เป็น Rust" มากกว่า
  คือการใช้ **channel** (`mpsc`/`broadcast`) แทนการเก็บ reference ที่ใช้ร่วมกันระหว่าง publisher กับ subscriber
- ผสมทั้งสาม pattern (Builder + Strategy + Observer) เข้าด้วยกันในระบบตัวอย่างจริงจัง (ระบบคำสั่งซื้อ e-commerce)
  และเห็นว่า pattern เหล่านี้ "ประกอบกันเอง" ได้อย่างเป็นธรรมชาติในโค้ด Rust จริง ไม่ใช่แค่ตัวอย่างแยกส่วนสั้น ๆ

## ความรู้ที่ต้องมีมาก่อน

- **Part 21 (Traits ขั้นสูง) — จำเป็นที่สุดสำหรับบทนี้**: บทนี้ใช้ **trait object** (`&dyn Trait`, `Box<dyn Trait>`),
  **object safety/dyn compatibility**, และกรอบคิด **static dispatch เทียบ dynamic dispatch** (monomorphization,
  vtable, fat pointer) จาก Part 21 ตลอดทั้งบท โดยเฉพาะหัวข้อ Strategy pattern ที่เป็นการนำกรอบคิดนั้นมาใช้ตัดสินใจ
  จริงในสถานการณ์ที่มีชื่อเรียก ("นี่คือ pattern อะไร") ถ้าจำ vtable/fat pointer/object safety ไม่ได้ ควรย้อนไปทวน
  Part 21 ก่อน
- **Part 19 (Traits เบื้องต้น)**: การนิยาม `trait`, `impl Trait for Type`, default method — ใช้เป็นพื้นฐานของ
  ทุก pattern ในบทนี้
- **Part 18/22 (Generics)**: จำเป็นสำหรับ Strategy pattern แบบ static dispatch (`fn calculate<S: PricingStrategy>`)
- **Part 9 (Structs)** และ **Part 4 (ฟังก์ชันและ parameter)**: Part 9 หัวข้อ 9.13 เกริ่น method chaining และ
  บอกตรง ๆ ว่า "Builder pattern เต็มรูปแบบจะเรียนใน Part 52" — **นี่คือบทนั้น** เช่นเดียวกับ Part 4 ที่อธิบายว่า
  Rust ไม่มี default parameter/named parameter เหมือน Python/Kotlin ทำให้ constructor ที่มีหลาย parameter
  optional กลายเป็นปัญหาที่ Builder pattern เข้ามาแก้
- **Part 19 หัวข้อท้ายบท**: เกริ่น Builder pattern สั้น ๆ อีกครั้งก่อนส่งต่อไปยัง Part 21 (trait object) และ
  Part 52 (pattern เต็มรูปแบบ)
- **Part 45 (Derive Macros ขั้นสูง)**: หัวข้อ 45.13 เขียน `#[derive(Builder)]` ที่ **generate** โค้ด builder
  แบบเดียวกับที่บทนี้จะสอนให้เขียนด้วยมือ — บทนี้จะอธิบายว่าโค้ดที่ macro นั้น generate ให้ตรงกับอะไรที่เราเขียน
  เองทุกประการ (มาจากมุมกลับกัน)
- **Part 24 (Closures)**: `Fn`, `FnMut`, `FnOnce` และการ capture ตัวแปร — ใช้เป็นทางเลือกที่เบากว่าของ Strategy
  pattern เมื่อ "กลยุทธ์" คือฟังก์ชันเดียว ไม่ใช่กลุ่มพฤติกรรมที่เกี่ยวข้องกัน
- **Part 11-12 (Option/Result)** และ **Part 30 (Error Handling ขั้นสูง)**: ใช้ทำ validation ใน `.build()` ที่คืน
  `Result<T, BuilderError>` พร้อม custom error type ตามแนวทาง Part 30
- **Part 27-29 (Smart Pointers)**: ความคุ้นเคยกับ `Box<T>` (โดยเฉพาะการเก็บค่าบน heap ผ่าน owned pointer) ช่วยให้
  เข้าใจ `Box<dyn Trait>` ในบทนี้ได้เร็วขึ้น แม้ Part 21 จะสอนกลไกเจาะลึกไปแล้วก็ตาม
- **Part 38 (Channels)**: `std::sync::mpsc` — ใช้เป็นทางเลือกของ Observer pattern ในหัวข้อ 52.10

## เนื้อหา

### 52.1 กรอบคิด: Design Pattern แบบ GoF กับ Rust คุยกันอย่างไร

ปี 1994 หนังสือ *Design Patterns: Elements of Reusable Object-Oriented Software* โดยผู้เขียนสี่คนที่รู้จักกันในชื่อ
**Gang of Four (GoF)** รวบรวม pattern การออกแบบซอฟต์แวร์ 23 แบบที่พบเจอบ่อยในโค้ด C++/Smalltalk สมัยนั้น
สิ่งสำคัญที่ต้องเข้าใจให้ชัดก่อนเริ่มบทนี้คือ **pattern เหล่านั้นถูกออกแบบมาเพื่อ "ชดเชย" สิ่งที่ภาษา class-based OOP
ในยุคนั้นทำไม่ได้โดยตรง** — พูดตรง ๆ กว่านั้นคือ หลาย pattern คือ **"วิธีเลี่ยงข้อจำกัดของภาษา"** มากกว่าจะเป็น
"ความจริงสากลของการออกแบบซอฟต์แวร์" เช่น Strategy pattern แบบดั้งเดิมในหนังสือ GoF เกิดขึ้นเพราะ Java/C++ ยุคนั้น
**ไม่มี first-class function** (ส่งฟังก์ชันเป็นค่าไปมาไม่ได้ตรง ๆ) จึงต้องห่อ "พฤติกรรม" ไว้ในรูปของ object ที่มี
method เดียว (`interface Strategy { void execute(); }`) แทน — นี่คือเหตุผลที่ทำให้ pattern จำนวนมากในหนังสือ GoF
มี "รูปร่าง" เป็น interface + concrete class เสมอ เพราะเป็นเครื่องมือเดียวที่ภาษาเหล่านั้นมีให้ใช้

Rust มีเครื่องมือที่ทรงพลังกว่านั้นมาก: **closure ที่เป็น first-class value จริง** (Part 24), **enum ที่เป็น sum
type แบบเต็มรูปแบบ** (Part 10) พร้อม pattern matching, **trait ที่แยก "สัญญา" ออกจาก "การสืบทอด class"** อย่าง
สิ้นเชิง (Part 19/21), และระบบ **ownership/borrowing** ที่ทำให้ปัญหาบางอย่างที่ pattern ของ GoF ต้องแก้ (เช่น
การจัดการ lifecycle ของ object ที่ใครเป็นเจ้าของ ใครแค่ยืมดู) ถูกบังคับให้คิดถูกตั้งแต่ compile time โดยไม่ต้องมี
pattern พิเศษมาช่วยเลย ผลลัพธ์คือ **pattern บางตัวของ GoF "หายไปเลย" ในโค้ด Rust ทั่วไป** เพราะไม่จำเป็นอีกต่อไป
(เช่น Singleton pattern มักถูกแทนที่ด้วย `static` + `OnceLock`/module-level constant ธรรมดา ไม่ต้องมี class พิเศษ
คอยป้องกันการสร้าง instance ซ้ำ) ในขณะที่ **pattern บางตัวยังจำเป็นอยู่ แต่ "หน้าตา" ต่างจากตำราเดิมไปมาก** เพราะ
ใช้เครื่องมือของ Rust แทนที่จะพยายามเลียนแบบ class hierarchy

เพื่อให้เห็นภาพรวมก่อนเริ่มเจาะลึก มาดูตารางสั้น ๆ ว่า pattern บางตัวจากตำรา GoF (ที่ไม่ใช่หัวข้อหลักของบทนี้) มา
ปรากฏอย่างไรในโค้ด Rust — จะเห็นความหลากหลายของ "ระดับการแปล" ที่ต่างกันไปในแต่ละ pattern:

| Pattern (GoF) | ใน Java/C++ ดั้งเดิม | ใน Rust ที่เป็น idiomatic |
|---|---|---|
| Singleton | class พิเศษที่ป้องกัน constructor ปกติ เก็บ instance เดียวไว้เป็น static field | มักไม่จำเป็นเป็น pattern แยกเลย — ใช้ `static` ธรรมดา หรือ `std::sync::OnceLock`/`LazyLock` ตรง ๆ |
| Factory Method | interface `Factory` + subclass ที่ override วิธีสร้าง object | ฟังก์ชัน associated function ธรรมดา (`Type::new(...)`) หรือ `Box<dyn Trait>` ที่ return ตามเงื่อนไข (หัวข้อ 21.5 ที่ทวนไว้แล้ว) |
| Iterator | interface `Iterator` ที่มี `hasNext()`/`next()` แยกกันสองขั้นตอน | `trait Iterator` ในตัวภาษา (Part 16) ที่รวมสองขั้นตอนเป็น `next() -> Option<T>` เดียว — เนียนกว่าเพราะใช้ `Option` แทนสอง method |
| Decorator | ห่อ object ชั้นแล้วชั้นเพื่อเพิ่มพฤติกรรม โดยยังคง interface เดิม | มักใช้ trait ผสม default method หรือ newtype wrapper (Part 53) — บางกรณี generic ที่ compose กันตรง ๆ ก็ทำหน้าที่แทนได้ |
| Strategy, Builder, Observer | interface/class ตามที่ตำราสอน | **หัวข้อหลักของบทนี้** — มีทั้งที่ตรงกับตำรา (Builder) และที่มีทางเลือกอื่นที่เป็น Rust มากกว่า (Strategy, Observer) |

บทนี้และ Part 53 จะไม่สอน pattern แบบ "แปลจาก Java ทีละบรรทัด" แต่จะสอน **pattern อย่างที่มันปรากฏจริงในโค้ด Rust
idiomatic** — บางครั้งจะตรงกับตำรา GoF เกือบทั้งหมด (เช่น Builder pattern), บางครั้งจะมีสองสามรูปแบบให้เลือกตาม
สถานการณ์ (เช่น Strategy pattern ที่มีทั้งแบบ `dyn Trait`, generic, และ closure), และบางครั้งจะมีทางเลือกที่ "เป็น
Rust มากกว่า" ทางเลือกดั้งเดิมไปเลย (เช่น Observer pattern ที่ channel มักเหมาะกว่า `Vec<Box<dyn Observer>>` ใน
หลายสถานการณ์) — Part 53 จะไปต่อกับ pattern ที่ **ไม่มีอยู่ในตำรา GoF เลย** เพราะเป็น pattern ที่เกิดขึ้นเฉพาะจาก
type system และ ownership ของ Rust เท่านั้น (Newtype, Typestate, RAII)

### 52.2 ปัญหาที่ Builder Pattern แก้: Constructor ยักษ์ที่มี Argument จำนวนมาก

ทวนจาก **Part 4**: Rust ไม่มี **default parameter** (กำหนดค่า default ให้ parameter แล้วไม่ต้องส่งก็ได้) และไม่มี
**named/keyword argument** (เรียกฟังก์ชันโดยระบุชื่อ parameter แทนตำแหน่ง) แบบที่ Python (`def f(a, b=1, c=2)`)
หรือ Kotlin ทำได้ — ทุก argument ใน Rust ต้องส่งครบตามตำแหน่ง (positional) เสมอ ปัญหานี้ไม่รุนแรงถ้าฟังก์ชันมี 2-3
parameter แต่จะกลายเป็นปัญหาจริงเมื่อ struct มี field จำนวนมาก โดยเฉพาะเมื่อหลาย field เป็น **optional** (มีค่า
default ที่สมเหตุสมผลได้ ไม่บังคับต้องระบุ) มาดูปัญหานี้ให้เห็นภาพชัดด้วยตัวอย่างจริง: การสร้าง HTTP request

```rust
// 52.2 - ปัญหา: constructor ที่มี argument จำนวนมาก อ่านยาก จำลำดับไม่ได้ ผิดง่าย
#[derive(Debug)]
struct HttpRequest {
    method: String,
    url: String,
    headers: Vec<(String, String)>,
    body: Option<String>,
    timeout_secs: u64,
    follow_redirects: bool,
    max_retries: u32,
}

impl HttpRequest {
    // constructor เดียวที่รับทุก field เป็น positional argument — ยิ่ง field เยอะ ยิ่งอ่านไม่รู้เรื่อง
    fn new(
        method: String,
        url: String,
        headers: Vec<(String, String)>,
        body: Option<String>,
        timeout_secs: u64,
        follow_redirects: bool,
        max_retries: u32,
    ) -> Self {
        HttpRequest { method, url, headers, body, timeout_secs, follow_redirects, max_retries }
    }
}

fn main() {
    // อ่านบรรทัดนี้แล้วบอกได้ไหมว่า true ตัวที่สองคืออะไร กับ 3 ตัวสุดท้ายคือ retry กี่ครั้ง timeout กี่วิ?
    // ต้องเปิดไปดู signature ของ new() ทุกครั้งที่เรียก — และถ้าสลับลำดับ true/false กับตัวเลขผิด
    // compiler ก็ตรวจไม่ได้เลยเพราะ type ตรงกันหมด (มีแต่ตำแหน่งที่ผิด)
    let req = HttpRequest::new(
        "GET".to_string(),
        "https://api.example.com/users".to_string(),
        vec![],
        None,
        30,
        true,
        3,
    );
    println!("{:?}", req);
}
```

ผลลัพธ์:

```
HttpRequest { method: "GET", url: "https://api.example.com/users", headers: [], body: None, timeout_secs: 30, follow_redirects: true, max_retries: 3 }
```

โปรแกรมนี้ compile และรันได้ถูกต้องสมบูรณ์ — **แต่นั่นคือปัญหาที่แท้จริง**: มันดู "ถูก" ในสายตา compiler ทั้งที่จริง ๆ
แล้วโค้ดที่เรียกมันอ่านไม่รู้เรื่องเลยว่า `true` หมายถึง `follow_redirects` และ `3` หมายถึง `max_retries` ถ้าสลับ
`follow_redirects: bool` กับ field `bool` อีกตัวที่อาจถูกเพิ่มเข้ามาในอนาคต (เช่น `verify_ssl: bool`) compiler จะ
ไม่มีทางจับได้เลยเพราะ type ตรงกันทุกประการ ยิ่งไปกว่านั้น ถ้าอนาคตต้องเพิ่ม field ใหม่อีก 2 field (เช่น
`user_agent: Option<String>`, `verify_ssl: bool`) ทุกจุดในโค้ดที่เรียก `HttpRequest::new(...)` ต้องถูกแก้ไขให้ส่ง
argument เพิ่มครบทุกจุด — เป็น **breaking change** ขนาดใหญ่ทุกครั้งที่ struct โตขึ้น แม้ field ใหม่จะเป็น optional
โดยธรรมชาติก็ตาม (เช่น "ถ้าไม่ระบุ user agent ก็ใช้ค่า default" — แต่ constructor แบบ positional บังคับให้ทุกคน
ต้องส่งค่าอยู่ดี)

ทางแก้ไขแรกที่มือใหม่มักคิดถึงคือใช้ `Option<T>` ทำให้ทุก field เป็น optional จริง ๆ ในตัว struct เอง แต่นั่นทำให้
ทุกจุดที่**ใช้** `HttpRequest` ต้อง `.unwrap()`/pattern match ทุก field ที่จริง ๆ แล้ว "ควรจะมีค่าแน่นอน" หลังสร้าง
เสร็จ (เช่น `method` และ `url` ไม่ควรเป็น `Option` เลย เพราะ HTTP request ที่ไม่มี method/url ไม่มีความหมาย) —
Rust ไม่มี default parameter ให้ใช้แก้ปัญหานี้แบบภาษาอื่น ทางออกที่ idiomatic ในโค้ด Rust จริงคือ **Builder pattern**
ซึ่งแยกปัญหาออกเป็นสองช่วงเวลาอย่างชัดเจน: **ช่วง "กำลังสร้าง" (field ไหนจะเป็น `Option<T>` ก็ได้ ยังไม่ครบก็ได้)**
กับ **ช่วง "สร้างเสร็จแล้วใช้งานจริง" (field ที่จำเป็นต้องมีค่าแน่นอน ไม่ใช่ `Option` อีกต่อไป)** — หัวข้อถัดไปจะสร้าง
ทางออกนี้ให้เห็นแบบเต็มรูปแบบ

### 52.3 Builder Pattern แบบ Consuming (`self -> Self`): `HttpRequestBuilder`

แนวคิดหลักของ Builder pattern มีสามส่วนที่ประกอบกัน (ตรงกับที่ Part 45 หัวข้อ 45.13 ออกแบบไว้ก่อนเขียน macro):

1. **struct ที่สอง** แยกจาก struct จริง เรียกว่า "builder" (ในที่นี้คือ `HttpRequestBuilder` แยกจาก `HttpRequest`)
   ที่ทุก field ที่ "อาจยังไม่ถูกกำหนด" ห่อด้วย `Option<T>` ไว้ก่อน
2. **method setter หนึ่งตัวต่อ field** ที่รับค่าเข้าไปตั้งใน builder แล้ว **คืน builder กลับออกมาเพื่อ chain
   ต่อได้** (`.method("GET").header(...).body(...)`)
3. **method `.build()`** ที่ตรวจสอบว่า field ที่ "บังคับต้องมี" (required) ถูกกำหนดแล้วครบทุกตัวหรือยัง ถ้าไม่ครบ
   คืน `Err` (ตาม pattern error handling จาก Part 30) ถ้าครบก็แปลง `Option<T>` ทุกตัวกลับเป็น `T` แล้วสร้าง struct
   จริงคืนกลับไป

มาดูเวอร์ชันแรก — **consuming builder** ที่ setter รับ `mut self` (ไม่ใช่ `&mut self`) และคืน `Self` แบบ **by
value** (ไม่ใช่ reference):

```rust
// 52.3 - Builder pattern แบบเต็มรูปแบบ: consuming builder (self -> Self)
use std::fmt;

#[derive(Debug)]
struct HttpRequest {
    method: String,
    url: String,
    headers: Vec<(String, String)>,
    body: Option<String>,
    timeout_secs: u64,
    follow_redirects: bool,
    max_retries: u32,
}

// Error type ตามแนวทาง Part 30: enum ที่บอกเหตุผลของความล้มเหลวได้ชัดเจน ไม่ใช่ String ธรรมดา
#[derive(Debug)]
enum BuilderError {
    MissingField(&'static str),
}

impl fmt::Display for BuilderError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            BuilderError::MissingField(name) => {
                write!(f, "field ที่จำเป็นยังไม่ถูกกำหนด: `{}`", name)
            }
        }
    }
}

impl std::error::Error for BuilderError {}

// builder struct: field ที่ "จำเป็น" (required) เก็บเป็น Option<T> ชั่วคราวระหว่างสร้าง
// field ที่ "ไม่บังคับ" (optional) มีค่า default ที่สมเหตุสมผลอยู่แล้ว ไม่ต้องเป็น Option ก็ได้
struct HttpRequestBuilder {
    method: Option<String>,       // required — ต้องเรียก .method(...) ก่อน .build() เสมอ
    url: Option<String>,          // required — ต้องเรียก .url(...) ก่อน .build() เสมอ
    headers: Vec<(String, String)>, // optional — ค่า default คือ vec เปล่า ไม่ต้องเป็น Option
    body: Option<String>,         // optional โดยธรรมชาติของ HTTP (GET ไม่มี body ก็ได้) — Option ที่นี่มีความหมาย
                                   // เป็น "ไม่มี body จริง ๆ" ไม่ใช่ "ยังไม่ถูกกำหนด"
    timeout_secs: u64,            // optional — มีค่า default 30 วินาที
    follow_redirects: bool,       // optional — มีค่า default true
    max_retries: u32,             // optional — มีค่า default 0
}

impl HttpRequestBuilder {
    // จุดเริ่มต้นของ builder: กำหนดค่า default ให้ทุก field ที่ optional ไว้ตรงนี้เลย
    // field ที่ required ตั้งเป็น None เพื่อบังคับให้ผู้ใช้ต้องเรียก setter ก่อน build()
    fn new() -> Self {
        HttpRequestBuilder {
            method: None,
            url: None,
            headers: Vec::new(),
            body: None,
            timeout_secs: 30,
            follow_redirects: true,
            max_retries: 0,
        }
    }

    // setter แบบ consuming: รับ "mut self" (ไม่ใช่ &mut self) แปลว่าฟังก์ชันนี้ "ยึด" ความเป็นเจ้าของ builder
    // เข้ามา แก้ไขมัน แล้วคืนความเป็นเจ้าของกลับออกไปเป็น Self (by value ไม่ใช่ reference)
    fn method(mut self, method: impl Into<String>) -> Self {
        self.method = Some(method.into());
        self
    }

    fn url(mut self, url: impl Into<String>) -> Self {
        self.url = Some(url.into());
        self
    }

    // header ใช้ &str สองตัวแยกกัน แล้วค่อย push เข้า Vec ภายใน — เรียกซ้ำได้หลายครั้งเพื่อเพิ่มหลาย header
    fn header(mut self, key: impl Into<String>, value: impl Into<String>) -> Self {
        self.headers.push((key.into(), value.into()));
        self
    }

    fn body(mut self, body: impl Into<String>) -> Self {
        self.body = Some(body.into());
        self
    }

    fn timeout_secs(mut self, secs: u64) -> Self {
        self.timeout_secs = secs;
        self
    }

    fn follow_redirects(mut self, follow: bool) -> Self {
        self.follow_redirects = follow;
        self
    }

    fn max_retries(mut self, retries: u32) -> Self {
        self.max_retries = retries;
        self
    }

    // build() คือจุดเดียวที่ validate — แปลง Option<T> ของ field required กลับเป็น T
    // ด้วย ok_or() แล้ว propagate error ด้วย ? (ตาม Part 12/30) ถ้า field ไหนขาด จะได้ Err ทันที
    fn build(self) -> Result<HttpRequest, BuilderError> {
        let method = self.method.ok_or(BuilderError::MissingField("method"))?;
        let url = self.url.ok_or(BuilderError::MissingField("url"))?;

        Ok(HttpRequest {
            method,
            url,
            headers: self.headers,
            body: self.body,
            timeout_secs: self.timeout_secs,
            follow_redirects: self.follow_redirects,
            max_retries: self.max_retries,
        })
    }
}

impl HttpRequest {
    // entry point: HttpRequest::builder() แทนที่จะเรียก HttpRequestBuilder::new() ตรง ๆ
    // (idiom ที่นิยมมากในโค้ด Rust จริง — ให้ struct จริงเป็นจุดเริ่มต้นเสมอ ผู้ใช้ไม่ต้องรู้จักชื่อ builder เลย)
    fn builder() -> HttpRequestBuilder {
        HttpRequestBuilder::new()
    }
}

fn main() -> Result<(), BuilderError> {
    // อ่านง่ายกว่า constructor เดิมมาก: แต่ละบรรทัดบอกชื่อ field ตรง ๆ ไม่ต้องจำลำดับ
    // ไม่ต้องระบุ field ที่ใช้ค่า default (timeout_secs, follow_redirects, max_retries)
    let req = HttpRequest::builder()
        .method("GET")
        .url("https://api.example.com/users")
        .header("Authorization", "Bearer token123")
        .header("Accept", "application/json")
        .max_retries(3)
        .build()?;

    println!("{:#?}", req);

    // ลองสร้างแบบที่ขาด field required (url) — ต้องได้ Err กลับมา ไม่ panic ไม่ crash
    let missing = HttpRequest::builder().method("POST").build();
    match missing {
        Ok(_) => println!("ไม่ควรมาถึงจุดนี้"),
        Err(e) => println!("สร้างไม่สำเร็จตามที่คาด: {}", e),
    }

    Ok(())
}
```

ผลลัพธ์:

```
HttpRequest {
    method: "GET",
    url: "https://api.example.com/users",
    headers: [
        (
            "Authorization",
            "Bearer token123",
        ),
        (
            "Accept",
            "application/json",
        ),
    ],
    body: None,
    timeout_secs: 30,
    follow_redirects: true,
    max_retries: 3,
}
สร้างไม่สำเร็จตามที่คาด: field ที่จำเป็นยังไม่ถูกกำหนด: `url`
```

สังเกตรายละเอียดสำคัญหลายจุด:

**เหตุผลที่ setter รับ `impl Into<String>` แทน `String` ตรง ๆ**: ทำให้เรียก `.method("GET")` ด้วย `&str` literal
ได้โดยตรง โดยไม่ต้องเขียน `.method("GET".to_string())` ทุกครั้ง (`&str` implement `Into<String>` อยู่แล้วผ่าน
`From<&str> for String` ที่มีมาตั้งแต่ standard library) — นี่คือ idiom ที่พบบ่อยมากในโค้ด builder ของ Rust จริง
เพราะช่วยลด "พิธีกรรม" (ceremony) ที่ผู้ใช้ builder ต้องทำโดยไม่จำเป็น

**เหตุผลที่ `BuilderError` ควรพิจารณาใส่ `#[non_exhaustive]` ถ้าเป็น public API**: ทวนจาก **Part 30 หัวข้อ 30.7**
— ถ้า `HttpRequestBuilder` เป็นส่วนหนึ่งของ library ที่ crate อื่นจะ `match` บน `BuilderError` (เช่นเพื่อแสดงข้อความ
ที่ต่างกันตามชนิด error) การเพิ่ม variant ใหม่ในอนาคต (เช่น `BuilderError::InvalidUrl` ตอนเพิ่มการ validate รูปแบบ
URL) จะเป็น **breaking change** ทันทีสำหรับทุก crate ที่ `match` แบบ exhaustive ไว้กับ `BuilderError` เดิม — ใส่
`#[non_exhaustive]` ไว้บน enum ตั้งแต่แรกจะบังคับให้ผู้ใช้จาก crate อื่นต้องมี `_ =>` เผื่อไว้เสมอ ทำให้เพิ่ม variant
ใหม่ในอนาคตได้โดยไม่ทำลายโค้ดที่มีอยู่ — รายละเอียดเต็มรูปแบบและตัวอย่าง error E0004 จริงอยู่ใน Part 30 หัวข้อ 30.7
บทนี้ขอเพียงเชื่อมโยงให้เห็นว่าหลักการเดียวกันนำมาใช้กับ error type ของ builder ได้โดยตรง

**เหตุผลที่ `header()` push เข้า `Vec` แทนการเก็บเป็น `Option<Vec<...>>`**: เพราะ "ไม่มี header เลย" เป็นค่าที่
สมเหตุสมผลอยู่แล้ว (`Vec::new()` ว่างเปล่า) ไม่จำเป็นต้องแยกแยะระหว่าง "ยังไม่ถูกตั้งค่า" กับ "ตั้งค่าเป็นค่าง่าย"
เหมือนกับ `method`/`url` ที่ "ไม่มีค่า" ไม่สมเหตุสมผล (HTTP request ที่ไม่มี URL ไม่มีความหมายอะไรเลย) — **นี่คือ
เกณฑ์สำคัญที่ต้องคิดตอนออกแบบ builder จริง**: field ที่ "การไม่มีค่าเป็นสถานะที่ใช้งานได้จริง" ไม่ต้องห่อด้วย
`Option<T>` ในตัว builder เลย ใส่ค่า default ตรง ๆ ใน `new()` ได้ทันที ส่วน field ที่ "จำเป็นต้องมีค่าที่มีความหมาย
เท่านั้น ไม่มีค่า default ที่สมเหตุสมผล" (เช่น URL ปลายทาง) จึงค่อยห่อด้วย `Option<T>` เพื่อบังคับให้ validate ตอน
`.build()`

**เหตุผลที่ `main()` return `Result<(), BuilderError>`**: ใช้ `?` operator (Part 12) ส่ง error ต่อออกไปได้ตรง ๆ
โดยไม่ต้อง `.unwrap()` หรือเขียน `match` ยาว ๆ ทุกจุดที่เรียก `.build()` — ฟังก์ชัน `main` ที่ return `Result`
เป็น feature มาตรฐานของ Rust ที่ Part 12 สอนไว้แล้ว

#### ทำไมต้อง "รับ `self` แล้วคืน `Self`" แทน `&mut self` แล้วคืน `&mut Self`

นี่คือคำถามที่สำคัญที่สุดของหัวข้อนี้ — Part 9 หัวข้อ 9.13 โชว์ method chaining แบบ `&mut self -> &mut Self` ไปแล้ว
(ตัวอย่าง `Product::with_quantity`) ทำไม Builder pattern เต็มรูปแบบถึงเปลี่ยนมาใช้ `self -> Self` (by value) แทน?
คำตอบคือทั้งสองแบบมี **tradeoff ที่ต่างกันจริง** ไม่ใช่แบบหนึ่ง "ถูก" อีกแบบ "ผิด" — มาดูทั้งสองแบบเทียบกันตรง ๆ

### 52.4 Builder Pattern แบบ `&mut self`: ข้อดี-ข้อเสียเทียบกับ Consuming Builder

```rust
// 52.4 - Builder pattern แบบ &mut self (ไม่ consume ตัวเอง คืน &mut Self แทน)
#[derive(Debug)]
struct HttpRequest {
    method: String,
    url: String,
    timeout_secs: u64,
}

#[derive(Debug)]
struct BuilderError(&'static str);

struct HttpRequestBuilderRef {
    method: Option<String>,
    url: Option<String>,
    timeout_secs: u64,
}

impl HttpRequestBuilderRef {
    fn new() -> Self {
        HttpRequestBuilderRef { method: None, url: None, timeout_secs: 30 }
    }

    // setter แบบ &mut self: "ยืม" ตัวเองแบบ mutable แล้วคืน &mut Self กลับไป (ไม่ยึดความเป็นเจ้าของ)
    fn method(&mut self, method: impl Into<String>) -> &mut Self {
        self.method = Some(method.into());
        self
    }

    fn url(&mut self, url: impl Into<String>) -> &mut Self {
        self.url = Some(url.into());
        self
    }

    fn timeout_secs(&mut self, secs: u64) -> &mut Self {
        self.timeout_secs = secs;
        self
    }

    // build() รับ &self (ไม่ยึดความเป็นเจ้าของ) — เรียกซ้ำได้หลายครั้งจาก builder ตัวเดิม
    // แต่ต้อง clone ค่าข้างใน เพราะ &self ไม่สามารถ "ย้าย" ข้อมูลออกไปจาก reference ได้
    fn build(&self) -> Result<HttpRequest, BuilderError> {
        let method = self.method.clone().ok_or(BuilderError("method"))?;
        let url = self.url.clone().ok_or(BuilderError("url"))?;
        Ok(HttpRequest { method, url, timeout_secs: self.timeout_secs })
    }
}

fn main() {
    // ข้อดีของ &mut self: เก็บ builder ไว้ในตัวแปรได้ แล้วค่อยเรียก method เพิ่มทีละบรรทัดคนละที่กันได้
    // (ต่างจาก consuming builder ที่ทำแบบนี้ไม่ได้ง่าย ๆ — ดูหัวข้อถัดไป)
    let mut builder = HttpRequestBuilderRef::new();
    builder.method("GET"); // เรียกทีละ statement แยกกันได้ ไม่ต้อง chain ในบรรทัดเดียว
    if true {
        // เงื่อนไข runtime บางอย่างที่ตัดสินใจว่าจะตั้ง url แบบไหน — ทำได้ง่ายเพราะ builder ยังอยู่ในตัวแปรเดิม
        builder.url("https://api.example.com/v1/users");
    } else {
        builder.url("https://api.example.com/v2/users");
    }
    builder.timeout_secs(60);

    // build() ไม่ consume builder — เรียกซ้ำได้อีกถ้าต้องการ (เช่นสร้างหลาย request จาก base เดียวกัน)
    let req1 = builder.build().unwrap();
    println!("{:?}", req1);

    // แก้ timeout แล้ว build() อีกครั้งจาก builder ตัวเดิม — ทำได้เพราะ build() ยืมแค่ &self
    builder.timeout_secs(120);
    let req2 = builder.build().unwrap();
    println!("{:?}", req2);
}
```

ผลลัพธ์:

```
HttpRequest { method: "GET", url: "https://api.example.com/v1/users", timeout_secs: 60 }
HttpRequest { method: "GET", url: "https://api.example.com/v1/users", timeout_secs: 120 }
```

ตาราง tradeoff เปรียบเทียบทั้งสองแบบโดยตรง:

| ประเด็น | Consuming (`self -> Self`) | Reference (`&mut self -> &mut Self`) |
|---|---|---|
| Chain ในนิพจน์เดียว (`.a().b().c()`) | ทำได้เป็นธรรมชาติ อ่านเหมือนประโยคเดียว | ทำได้เหมือนกัน แต่ `&mut self` มีข้อจำกัดเรื่อง temporary (ดูด้านล่าง) |
| แยกเรียกทีละ statement คนละที่กัน (เช่นใน `if`/`else` แยก branch) | ต้องเก็บ `let builder = ...` แล้ว reassign ทุกครั้ง เพราะแต่ละ method "ยึด" ตัวแปรไปแล้วคืนตัวใหม่ | ทำได้ตรง ๆ เพราะตัวแปรเดิมยังอยู่ ไม่ถูกยึดไป |
| เรียก `.build()` ซ้ำได้จาก builder ตัวเดิม | ทำไม่ได้ — `.build(self)` ยึด (consume) builder ไปแล้ว ใช้ต่อไม่ได้อีก | ทำได้ — `.build(&self)` แค่ยืมดู เรียกซ้ำได้เรื่อย ๆ |
| ต้อง `clone()` ข้อมูลข้างในตอน `build()` | ไม่ต้อง — ย้าย (move) ค่าออกจาก builder ที่กำลังจะถูกทิ้งไปพอดี ไม่มีต้นทุน copy เพิ่ม | ต้อง — เพราะ `&self` ยืมดูเท่านั้น ดึงข้อมูลออกมาแบบ own ไม่ได้ ต้อง `.clone()` แทน |
| นิยมใช้ในโค้ด production จริง | **นิยมมากที่สุด** (เช่น `reqwest::RequestBuilder`, `std::process::Command`) | พบน้อยกว่า มักใช้เมื่อต้องสร้างหลาย variant จาก base เดียวกันจริง ๆ |

**ทำไม consuming builder ถึงเป็นที่นิยมมากกว่าในโค้ด Rust จริง**: เหตุผลหลักคือ **ไม่ต้อง `clone()` ข้อมูลตอน
`build()`** — เพราะ `self` (by value) หมายความว่า builder ตัวนั้น "กำลังจะถูกทิ้งไปพอดี" หลังจาก `build()` return
Rust จึง **ย้าย (move)** ข้อมูลข้างในออกไปสร้าง struct จริงได้ตรง ๆ โดยไม่มีต้นทุนใด ๆ เพิ่ม (ทวนจาก Part 6-7:
move ไม่มีการ copy หน่วยความจำเกิดขึ้นจริงสำหรับ heap-allocated data อย่าง `String`/`Vec`) ในทางกลับกัน `&mut self`
builder ต้อง `.clone()` ทุก field ที่เป็น `String`/`Vec` ตอน `build()` เพราะ `&self` แค่ "ยืมดู" ไม่สามารถย้ายข้อมูล
ออกจาก reference ได้ (ทวนจาก Part 6: ย้ายค่าออกจาก reference ที่คนอื่นถืออยู่จะทำให้ reference นั้นชี้ไปยัง
หน่วยความจำที่ไม่สมบูรณ์ — Rust ห้ามแบบนี้เด็ดขาด) — ต้นทุน `.clone()` นี้อาจเล็กน้อยสำหรับ struct เล็ก ๆ แต่ถ้า
builder มี field ขนาดใหญ่ (เช่น `Vec` ยาว ๆ) การ clone ทุกครั้งที่ build() ถูกเรียกซ้ำอาจกลายเป็นต้นทุนที่จับต้องได้

ในทางกลับกัน **ข้อดีที่แท้จริงของ `&mut self` builder** คือความยืดหยุ่นในการเขียนโค้ดที่ setter ถูกเรียกแบบมี
เงื่อนไขซับซ้อน กระจายอยู่หลาย statement หรือหลาย function เช่น "ถ้า config บอกว่า verbose ให้เพิ่ม header X" —
กับ consuming builder ทำแบบนี้ได้เหมือนกัน แต่ต้อง reassign ตัวแปรทุกครั้ง (`builder = builder.method(...)`)
ซึ่งเขียนยากกว่าเล็กน้อยเมื่อ logic ซับซ้อนขึ้น (ดูตัวอย่างเปรียบเทียบในกับดักที่พบบ่อยข้อ 2 ท้ายบท)

**กฎที่ใช้ได้จริง**: เริ่มจาก consuming builder (`self -> Self`) เป็น default เสมอ เพราะเข้ากับ idiom ของ
ecosystem Rust ส่วนใหญ่ และไม่มีต้นทุน clone แฝงอยู่ — เปลี่ยนไปใช้ `&mut self` เฉพาะเมื่อมีความจำเป็นจริง ๆ ที่ต้อง
เก็บ builder ไว้ในตัวแปรระยะยาว แก้ไขทีละนิดจากหลายจุดในโค้ด หรือสร้างหลาย variant จาก base เดียวกันซ้ำ ๆ

### 52.5 ความสัมพันธ์กับ `#[derive(Builder)]` ของ Part 45: สิ่งที่ Macro Generate ให้เราคือ 52.3 นี้เอง

ตอนนี้เราเขียน `HttpRequestBuilder` เต็มรูปแบบด้วยมือไปแล้ว มาย้อนดู Part 45 หัวข้อ 45.13 อีกครั้ง — struct
`Config`/`ConfigBuilder` ที่ macro generate ให้นั้น **มีโครงสร้างเดียวกันกับ `HttpRequest`/`HttpRequestBuilder`
ทุกประการ**: struct builder ที่ทุก field เป็น `Option<T>`, setter หนึ่งตัวต่อ field ที่รับ `mut self` คืน `Self`,
และ `build()` ที่แปลง `Option<T>` กลับเป็น `T` ด้วย `ok_or_else()` — ต่างกันแค่รายละเอียดเล็กน้อยคือ Part 45
เลือกให้ **ทุก field เป็น required เสมอ** (เพื่อให้ macro เขียนง่าย ใช้ได้กับทุก struct โดยไม่ต้องรู้ว่า field
ไหน "ควรมีค่า default" — เพราะนั่นเป็นการตัดสินใจทางธุรกิจที่ macro ทั่วไปรู้ไม่ได้) ในขณะที่ตัวอย่าง 52.3 ของเรา
แยก required (`method`, `url`) กับ optional ที่มีค่า default (`timeout_secs`, `follow_redirects`, `max_retries`)
ออกจากกันอย่างตั้งใจ — **นี่คือเหตุผลที่ builder ที่เขียนด้วยมือยังมีที่ใช้อยู่แม้จะมี derive macro ให้ใช้แล้ว**:
macro ทำงานได้ดีที่สุดกับ struct ที่ทุก field required เท่ากันหมด ส่วน struct ที่มี field default ค่าต่างกัน
(ธุรกิจจริงมักเป็นแบบนี้) ยังต้องเขียน builder เองหรือใช้ crate สำเร็จรูปอย่าง `derive_builder` ที่รองรับ attribute
พิเศษสำหรับกำหนดค่า default ต่อ field ได้

พูดให้ครบวงจร: **Part 9 เกริ่นปัญหา (method chaining แบบพื้นฐาน) → Part 19/21 ให้เครื่องมือ (trait, `Option`,
`Result`) → Part 45 สอนวิธี generate โค้ด builder อัตโนมัติด้วย proc macro (มุมมองของคน "เขียนเครื่องมือ") → Part
52 (บทนี้) สอนวิธีเขียน builder เองด้วยมือให้ถูกต้องสมบูรณ์ (มุมมองของคน "ใช้เครื่องมือ" หรือกรณีที่เครื่องมือ
สำเร็จรูปไม่พอ)** — ทั้งสองมุมมองเสริมกัน ไม่ใช่แข่งกัน: เข้าใจโค้ดที่เขียนมือได้ ทำให้เข้าใจว่า macro "ควร"
generate อะไรออกมา และในทางกลับกัน เข้าใจกลไก macro ทำให้รู้ว่าเมื่อไหร่ควร generate อัตโนมัติ เมื่อไหร่ควรเขียนมือ
เพื่อควบคุมรายละเอียด (เช่น field ไหนควร default เป็นอะไร) ให้ตรงกับ business logic จริง

### 52.6 Typestate-adjacent Builder: เปลี่ยน Runtime Error เป็น Compile Error (ตัวอย่างเกริ่นก่อน Part 53)

Builder pattern แบบ 52.3 มีข้อจำกัดหนึ่งที่ต้องยอมรับตรง ๆ: **การตรวจสอบว่า field required ครบหรือยังเกิดขึ้น
ตอน runtime เท่านั้น** (`.build()` คืน `Result` ที่ต้องจัดการตอนรัน ไม่ใช่ตอน compile) ถ้าเผลอเรียก `.build()`
โดยลืมตั้ง `url` โปรแกรมจะ compile ผ่านสมบูรณ์ แล้วค่อยพัง (คืน `Err`) ตอนรันจริง — นี่คือจุดที่บางไลบรารีเลือก
ใช้เทคนิคที่ก้าวหน้าไปอีกขั้น เรียกว่า **typestate pattern** (Part 53 จะสอนเต็มรูปแบบ) หลักการสั้น ๆ ที่พอเห็นภาพ
ได้ตอนนี้คือ: **ใช้ type ที่ต่างกันแทน "สถานะ" ของ builder** เช่น `NoUrlSet`/`UrlSet` เป็น type parameter ที่
บอกว่า field `url` ถูกตั้งค่าแล้วหรือยัง — ทำให้ `.build()` **มีอยู่เฉพาะตอนที่ type parameter บอกว่า "ครบแล้ว"
เท่านั้น** ถ้าลืมตั้ง `url` แล้วเรียก `.build()` จะเป็น **compile error** ทันที ไม่ใช่ runtime `Err` อีกต่อไป
มาดูตัวอย่างง่าย ๆ พอให้เห็นกลไก (ไม่ลงรายละเอียดครบทุกกรณีเหมือน Part 53):

```rust
// 52.6 - Typestate-adjacent builder: ใช้ type parameter บอกสถานะ "ยังไม่ครบ" กับ "ครบแล้ว"
// (นี่คือตัวอย่างเกริ่นนำ — Part 53 จะสอนเทคนิคนี้แบบเต็มรูปแบบพร้อมชื่อ marker type ที่เป็นระบบมากกว่านี้)
use std::marker::PhantomData;

// marker type สองตัว ไม่มี field ข้างใน ใช้แค่เป็น "แท็ก" บอกสถานะที่ตำแหน่ง type parameter
struct NoUrl;
struct HasUrl;

struct SimpleRequestBuilder<UrlState> {
    method: String,
    url: Option<String>,
    // PhantomData<UrlState> ไม่กิน memory จริง (ขนาด 0 byte) มีไว้แค่บอก compiler ว่า
    // "struct นี้ผูกกับ type parameter UrlState เชิงตรรกะ" แม้ไม่มี field ไหนเก็บค่าชนิดนั้นจริง ๆ
    _marker: PhantomData<UrlState>,
}

impl SimpleRequestBuilder<NoUrl> {
    fn new(method: impl Into<String>) -> Self {
        SimpleRequestBuilder { method: method.into(), url: None, _marker: PhantomData }
    }

    // url() มีอยู่เฉพาะบน SimpleRequestBuilder<NoUrl> — เรียกแล้วได้ SimpleRequestBuilder<HasUrl> กลับมา
    // (type parameter "เปลี่ยน" จาก NoUrl เป็น HasUrl ตอนนี้เอง เป็นการ "ย้ายสถานะ" ผ่านการเปลี่ยน type)
    fn url(self, url: impl Into<String>) -> SimpleRequestBuilder<HasUrl> {
        SimpleRequestBuilder { method: self.method, url: Some(url.into()), _marker: PhantomData }
    }
}

impl SimpleRequestBuilder<HasUrl> {
    // build() มีอยู่เฉพาะบน SimpleRequestBuilder<HasUrl> เท่านั้น — ไม่มีให้เรียกเลยตอน type parameter เป็น NoUrl
    // ผลคือ: ถ้าลืมเรียก .url(...) ก่อน .build() จะเป็น "method ไม่มีอยู่จริง" (compile error) ไม่ใช่ runtime Err
    fn build(self) -> String {
        format!("{} {}", self.method, self.url.expect("HasUrl การันตีว่ามีค่าเสมอ"))
    }
}

fn main() {
    // ลำดับที่ถูก: new() -> url() -> build() — type parameter ไล่จาก NoUrl -> HasUrl ตามลำดับที่ต้อง
    let request = SimpleRequestBuilder::new("GET")
        .url("https://api.example.com/users")
        .build();
    println!("{}", request);

    // ลองสลับ: เรียก .build() ตรงจาก new() โดยไม่เรียก .url() ก่อน — ดูคอมเมนต์ด้านล่าง
    // let broken = SimpleRequestBuilder::new("GET").build();
    // ปลดคอมเมนต์บรรทัดบนแล้ว compile จะ error ทันที: no method named `build` found for struct
    // `SimpleRequestBuilder<NoUrl>` — เพราะ build() ถูก impl ไว้เฉพาะบน SimpleRequestBuilder<HasUrl> เท่านั้น
}
```

ผลลัพธ์:

```
GET https://api.example.com/users
```

ถ้าปลดคอมเมนต์บรรทัด `let broken = ...` compiler จะปฏิเสธทันทีด้วย error แนวนี้:

```
error[E0599]: no method named `build` found for struct `SimpleRequestBuilder<NoUrl>` in the current scope
  --> src/main.rs:37:44
   |
37 |     let broken = SimpleRequestBuilder::new("GET").build();
   |                                                    ^^^^^ method not found in `SimpleRequestBuilder<NoUrl>`
```

**นี่คือความแตกต่างเชิงคุณภาพที่สำคัญมาก**: builder จาก 52.3 จับ "field ขาด" ได้ตอน **runtime** ผ่าน `Result::Err`
(ต้องรันโปรแกรมจริงหรือรัน test เพื่อเจอปัญหา) ส่วน typestate-adjacent builder จับปัญหาเดียวกันได้ตอน **compile
time** ผ่าน error `method not found` (เจอปัญหาทันทีตอนพิมพ์โค้ด ก่อนโปรแกรมรันด้วยซ้ำ) — ข้อแลกเปลี่ยนคือความ
ซับซ้อนของโค้ดที่เพิ่มขึ้น (ต้องมี marker type, `PhantomData`, และ `impl` block แยกตาม type parameter) และยิ่งมี
field required หลายตัว ก็ยิ่งต้องมี marker type ผสมกันมากขึ้นตามไปด้วย (เช่น 3 field required จะต้องมีการผสม
type parameter ที่ซับซ้อนขึ้นมาก) — **Part 53 จะสอนเทคนิคนี้แบบเต็มรูปแบบ** ทั้งการจัดการ field required
หลายตัวพร้อมกัน และการออกแบบ API ที่ไม่ทำให้ผู้ใช้ต้องเจอ type signature ที่อ่านยากเกินไป ตอนนี้แค่เข้าใจ**หลักการ
แกนกลาง**ก็เพียงพอ: **"ใช้ type ที่ต่างกันแทนสถานะที่ต่างกัน ทำให้ compiler ปฏิเสธการใช้งานผิดลำดับได้เองโดย
อัตโนมัติ โดยไม่ต้องเขียน check ด้วยมือเลย"**

### 52.7 Strategy Pattern: ปัญหาที่ต้องแก้และภาพรวมสามวิธีใน Rust

**Strategy pattern** แก้ปัญหาที่พบบ่อยมาก: มีขั้นตอนการทำงานหลักที่ **คงที่** (เช่น "คำนวณราคาสุทธิของคำสั่งซื้อ")
แต่ **รายละเอียดของอัลกอริทึมย่อยหนึ่งจุด** ต้องเปลี่ยนได้ (เช่น "วิธีคิดส่วนลด" — ลูกค้าทั่วไปไม่มีส่วนลด, ลูกค้า VIP
ลด 10%, ช่วงโปรโมชันลดตามเงื่อนไขพิเศษ) ตำรา GoF ดั้งเดิมแก้ปัญหานี้ด้วย interface ที่มี method เดียว
(`interface DiscountStrategy { double apply(double price); }`) แล้วสร้าง concrete class แยกต่างหากสำหรับแต่ละ
อัลกอริทึม — Rust มีวิธีทำสิ่งนี้ได้ **สามแบบ** ที่แต่ละแบบเหมาะกับสถานการณ์ต่างกัน:

1. **`Box<dyn Trait>`** — dynamic dispatch เลือก strategy ได้ตอน **runtime** (เช่นอ่านจาก config, database,
   หรือเงื่อนไขที่รู้แค่ตอนโปรแกรมทำงาน) — ตรงกับ Strategy pattern แบบดั้งเดิมของ GoF มากที่สุด
2. **Generic + trait bound** — static dispatch เลือก strategy ได้ตอน **compile time** เมื่อรู้ล่วงหน้าแล้วว่า
   จะใช้ strategy ไหน (zero-cost, ไม่มีต้นทุน vtable indirection ตาม Part 21)
3. **Closure parameter (`Fn`)** — เมื่อ "strategy" คือฟังก์ชันเดียวจริง ๆ ไม่ใช่กลุ่มพฤติกรรมที่เกี่ยวข้องกันหลาย
   method — เบากว่าการสร้าง trait+struct ทั้งชุดสำหรับกรณีง่าย ๆ

มาดูแต่ละแบบเจาะลึกทีละแบบ โดยใช้ปัญหาเดียวกันตลอด — **การคำนวณส่วนลดของคำสั่งซื้อ**

### 52.8 Strategy แบบ `Box<dyn Trait>`: เลือกอัลกอริทึมตอน Runtime

```rust
// 52.8 - Strategy pattern แบบ Box<dyn Trait>: dynamic dispatch, เลือก strategy ตอน runtime
// ทวนจาก Part 21: dyn Trait ใช้ vtable + fat pointer, ต้นทุนคือ indirect call หนึ่งครั้งต่อการเรียก method

// trait ที่เป็น "สัญญา" ของทุกกลยุทธ์คำนวณส่วนลด — มี method เดียว ตรงตามเจตนาของ GoF Strategy ดั้งเดิม
trait DiscountStrategy {
    fn apply(&self, original_price: f64) -> f64;
    fn describe(&self) -> String;
}

// กลยุทธ์ที่ 1: ไม่มีส่วนลดเลย (ลูกค้าทั่วไป)
struct NoDiscount;

impl DiscountStrategy for NoDiscount {
    fn apply(&self, original_price: f64) -> f64 {
        original_price
    }
    fn describe(&self) -> String {
        "ไม่มีส่วนลด".to_string()
    }
}

// กลยุทธ์ที่ 2: ลดตามเปอร์เซ็นต์คงที่ (ลูกค้า VIP)
struct PercentageDiscount {
    percent: f64,
}

impl DiscountStrategy for PercentageDiscount {
    fn apply(&self, original_price: f64) -> f64 {
        original_price * (1.0 - self.percent / 100.0)
    }
    fn describe(&self) -> String {
        format!("ลด {:.0}%", self.percent)
    }
}

// กลยุทธ์ที่ 3: ลดแบบเงื่อนไขซับซ้อน (โปรโมชันซื้อครบ 1000 ลดทันที 100 บาท)
struct ThresholdDiscount {
    threshold: f64,
    flat_discount: f64,
}

impl DiscountStrategy for ThresholdDiscount {
    fn apply(&self, original_price: f64) -> f64 {
        if original_price >= self.threshold {
            (original_price - self.flat_discount).max(0.0)
        } else {
            original_price
        }
    }
    fn describe(&self) -> String {
        format!("ซื้อครบ {:.0} ลด {:.0} บาท", self.threshold, self.flat_discount)
    }
}

// ฟังก์ชันหลักที่ "ไม่รู้และไม่สนใจ" concrete type ของ strategy — รับผ่าน &dyn Trait เท่านั้น
// เลือก strategy ตัวไหนมาส่งเข้าฟังก์ชันนี้ เป็นเรื่องที่ตัดสินใจได้ตอน runtime อย่างอิสระ
fn checkout(price: f64, strategy: &dyn DiscountStrategy) -> f64 {
    let final_price = strategy.apply(price);
    println!(
        "ราคาเดิม {:.2} บาท -> {} -> ราคาสุทธิ {:.2} บาท",
        price,
        strategy.describe(),
        final_price
    );
    final_price
}

fn main() {
    let price = 1500.0;

    // เลือก strategy ตอน "runtime" จริง ๆ ผ่านเงื่อนไขที่รู้แค่ตอนโปรแกรมทำงาน (เช่น สถานะสมาชิกจาก database)
    let customer_tier = "vip"; // จำลองค่าที่มาจากภายนอกตอน runtime

    // สร้าง Vec<Box<dyn DiscountStrategy>> ได้ เพราะทุก concrete type ถูกซ่อนหลัง fat pointer เดียวกัน (Part 21)
    let strategy: Box<dyn DiscountStrategy> = match customer_tier {
        "vip" => Box::new(PercentageDiscount { percent: 10.0 }),
        "promo" => Box::new(ThresholdDiscount { threshold: 1000.0, flat_discount: 100.0 }),
        _ => Box::new(NoDiscount),
    };

    checkout(price, strategy.as_ref());

    // heterogeneous collection: strategy หลาย concrete type ปนกันใน Vec เดียว (สิ่งที่ generic ทำไม่ได้ตาม Part 21)
    let all_strategies: Vec<Box<dyn DiscountStrategy>> = vec![
        Box::new(NoDiscount),
        Box::new(PercentageDiscount { percent: 15.0 }),
        Box::new(ThresholdDiscount { threshold: 1000.0, flat_discount: 200.0 }),
    ];

    println!("\nเปรียบเทียบทุกกลยุทธ์กับราคาเดียวกัน:");
    for s in &all_strategies {
        checkout(price, s.as_ref());
    }
}
```

ผลลัพธ์:

```
ราคาเดิม 1500.00 บาท -> ลด 10% -> ราคาสุทธิ 1350.00 บาท

เปรียบเทียบทุกกลยุทธ์กับราคาเดียวกัน:
ราคาเดิม 1500.00 บาท -> ไม่มีส่วนลด -> ราคาสุทธิ 1500.00 บาท
ราคาเดิม 1500.00 บาท -> ลด 15% -> ราคาสุทธิ 1275.00 บาท
ราคาเดิม 1500.00 บาท -> ซื้อครบ 1000 ลด 200 บาท -> ราคาสุทธิ 1300.00 บาท
```

จุดสำคัญของตัวอย่างนี้ (ที่เชื่อมกับ Part 21 โดยตรง): `checkout()` รับ `&dyn DiscountStrategy` ไม่ใช่ generic
`<S: DiscountStrategy>` เพราะ **strategy ที่จะใช้ตัดสินใจตอน runtime จริง ๆ** (จาก `customer_tier` ที่อ่านมาจาก
ภายนอก) — ถ้าใช้ generic `checkout<S: DiscountStrategy>(price: f64, strategy: &S)` compiler จะต้อง monomorphize
แยกเวอร์ชันสำหรับทุก concrete type ที่เป็นไปได้ตั้งแต่ compile time ซึ่งขัดกับความจริงที่ว่า **เรายังไม่รู้ตอน
compile time ว่า customer_tier จะเป็นค่าไหน** และยิ่งสำคัญกว่านั้นคือ `Vec<Box<dyn DiscountStrategy>>` ทำให้เก็บ
`NoDiscount`, `PercentageDiscount`, `ThresholdDiscount` ปนกันใน collection เดียวได้ (heterogeneous collection —
สิ่งที่ `Vec<S: DiscountStrategy>` ทำไม่ได้เลยตามกฎ generics ที่ Part 21 อธิบายไว้อย่างละเอียด)

### 52.9 Strategy แบบ Generic: Static Dispatch เมื่อรู้ Strategy ตอน Compile Time

ถ้าสถานการณ์ต่างออกไป — **รู้อยู่แล้วตอน compile time ว่าจะใช้ strategy ตัวไหน** (เช่น เขียนโค้ดคนละเวอร์ชันสำหรับ
"โหมดทดสอบ" กับ "โหมด production" อย่างชัดเจนในโค้ด ไม่ใช่ตัดสินใจจาก input ตอน runtime) — generic + trait bound
ให้ผลลัพธ์เดียวกันโดยไม่มีต้นทุนของ dynamic dispatch เลย (ทวนจาก Part 21: static dispatch ผ่าน monomorphization
ทำให้ compiler generate โค้ดแยกสำหรับแต่ละ concrete type และ **inline** การเรียก method ได้เต็มที่ — เร็วกว่า
ทฤษฎีแต่แลกกับขนาด binary ที่โตขึ้นถ้ามีหลาย concrete type):

```rust
// 52.9 - Strategy pattern แบบ generic: static dispatch, zero-cost ตาม Part 21
trait DiscountStrategy {
    fn apply(&self, original_price: f64) -> f64;
    fn describe(&self) -> String;
}

struct NoDiscount;
impl DiscountStrategy for NoDiscount {
    fn apply(&self, original_price: f64) -> f64 {
        original_price
    }
    fn describe(&self) -> String {
        "ไม่มีส่วนลด".to_string()
    }
}

struct PercentageDiscount {
    percent: f64,
}
impl DiscountStrategy for PercentageDiscount {
    fn apply(&self, original_price: f64) -> f64 {
        original_price * (1.0 - self.percent / 100.0)
    }
    fn describe(&self) -> String {
        format!("ลด {:.0}%", self.percent)
    }
}

// generic function: <S: DiscountStrategy> ทำให้ compiler สร้างโค้ดแยกสำหรับทุก concrete type ที่เรียกจริง
// (monomorphization ตาม Part 18/21) — เรียก strategy.apply()/describe() เป็น direct call ล้วน ๆ ไม่มี vtable
fn checkout_static<S: DiscountStrategy>(price: f64, strategy: &S) -> f64 {
    let final_price = strategy.apply(price);
    println!(
        "[static] ราคาเดิม {:.2} บาท -> {} -> ราคาสุทธิ {:.2} บาท",
        price,
        strategy.describe(),
        final_price
    );
    final_price
}

fn main() {
    let price = 1500.0;

    // รู้ชัดเจนตอนเขียนโค้ดแล้วว่าจะใช้ strategy ไหน ไม่ต้องรอ input ตอน runtime — เหมาะกับ static dispatch
    let vip_strategy = PercentageDiscount { percent: 10.0 };
    checkout_static(price, &vip_strategy);

    let regular_strategy = NoDiscount;
    checkout_static(price, &regular_strategy);

    // ข้อจำกัดที่ต้องยอมรับ: ทำแบบนี้ไม่ได้ —
    // let strategies = vec![&vip_strategy, &regular_strategy]; // จะ error: type ไม่ตรงกัน
    // เพราะ checkout_static::<PercentageDiscount> กับ checkout_static::<NoDiscount> เป็นคนละฟังก์ชันกันไปแล้ว
    // หลัง monomorphization — Vec ต้องมีสมาชิกเป็น "หนึ่ง concrete type เดียว" เสมอตามกฎ generics (Part 21)
}
```

ผลลัพธ์:

```
[static] ราคาเดิม 1500.00 บาท -> ลด 10% -> ราคาสุทธิ 1350.00 บาท
[static] ราคาเดิม 1500.00 บาท -> ไม่มีส่วนลด -> ราคาสุทธิ 1500.00 บาท
```

สังเกตว่าโค้ดหน้าตาคล้าย 52.8 มาก (เรียก `strategy.apply()`/`strategy.describe()` เหมือนกัน) แต่ **กลไกภายในต่างกัน
โดยสิ้นเชิง**: `checkout_static::<PercentageDiscount>` และ `checkout_static::<NoDiscount>` คือ**สองฟังก์ชันที่
ต่างกันจริง ๆ** หลัง monomorphization การเรียก `strategy.apply()` ข้างในแต่ละเวอร์ชันเป็น **direct call** ไปยัง
`PercentageDiscount::apply` หรือ `NoDiscount::apply` ตรง ๆ ที่ compiler รู้ตำแหน่งแน่นอนแล้ว (เปิดโอกาสให้ inline
ได้เต็มที่) ไม่มีการอ่าน vtable หรือ indirect call เกิดขึ้นเลยแม้แต่ครั้งเดียว — แลกกับข้อจำกัดที่เห็นในคอมเมนต์:
**ไม่สามารถเก็บ `PercentageDiscount` กับ `NoDiscount` ปนกันใน `Vec` เดียวได้** เพราะมันเป็นคนละ concrete type กัน
ตามกฎ generics พื้นฐาน

**หมายเหตุเรื่องไวยากรณ์**: ทวนจาก **Part 19 หัวข้อ 19.6** — `fn checkout_static<S: DiscountStrategy>(price: f64,
strategy: &S)` กับ `fn checkout_static(price: f64, strategy: &impl DiscountStrategy)` เป็น**โค้ดเดียวกันเป๊ะ**หลัง
compile (`impl Trait` ใน parameter position เป็นแค่ **น้ำตาลไวยากรณ์** ที่ compiler แปลงเป็น generic ให้อัตโนมัติ
ทั้งคู่ใช้ static dispatch เหมือนกันทุกประการ) ความต่างมีแค่รูปแบบการเขียน: `impl Trait` อ่านง่ายกว่าเมื่อมี type
parameter เดียวและไม่ต้องอ้างชื่อ `S` ที่ไหนอีกในฟังก์ชัน ส่วน `<S: Trait>` จำเป็นต้องใช้เมื่อต้องอ้างชื่อ type
parameter ซ้ำ (เช่น `strategy: &S, backup: &S` ที่ต้องเป็น type เดียวกัน — `impl Trait` เขียนแบบนี้ไม่ได้เพราะแต่ละ
`impl Trait` ที่ปรากฏถือเป็นชื่อไม่ผูกกัน) หรือเมื่อต้อง return type นั้นออกไปด้วย (`impl Trait` ใช้ได้ทั้ง
parameter และ return position แต่คนละความหมายกัน — Part 19 หัวข้อ 19.8 อธิบายไว้แล้ว)

### 52.10 Strategy แบบ Closure (`Fn`): เมื่อ "กลยุทธ์" คือฟังก์ชันเดียว

ทั้งสองแบบข้างบน (`dyn Trait` และ generic) เหมาะกับสถานการณ์ที่ **strategy เป็นกลุ่มพฤติกรรมที่มีมากกว่าหนึ่ง
method ที่เกี่ยวข้องกัน** (`apply()` และ `describe()` ต้องอยู่คู่กันเสมอ) แต่ถ้า "กลยุทธ์" จริง ๆ แล้วคือ**ฟังก์ชัน
เดียวแค่ตัวเดียว** (คำนวณราคาอย่างเดียว ไม่ต้องมี `describe()` ประกอบ) การสร้าง `trait` + `struct` หนึ่งชุดต่อ
กลยุทธ์หนึ่งแบบอาจเป็นพิธีกรรม (ceremony) ที่มากเกินความจำเป็น — **closure parameter** (ทวนจาก Part 24) เป็น
ทางเลือกที่เบากว่าและ idiomatic กว่าในสถานการณ์แบบนี้:

```rust
// 52.10 - Strategy pattern แบบ closure: เบากว่า trait+struct เมื่อ "กลยุทธ์" คือฟังก์ชันเดียว
// ทวนจาก Part 24: Fn(f64) -> f64 คือ trait bound ที่บอกว่า "รับได้ทั้ง fn ธรรมดา, closure ที่ไม่ capture,
// และ closure ที่ capture ตัวแปรจากสิ่งแวดล้อมแบบ borrow" — เรียกได้ไม่จำกัดจำนวนครั้ง (Fn ไม่ใช่ FnOnce)

// รับ closure ตรง ๆ ผ่าน generic + Fn bound (impl Fn(...) -> ... ก็เขียนได้เหมือนกัน แบบนี้ชัดกว่าเรื่อง generic)
fn checkout_with<F>(price: f64, discount_fn: F) -> f64
where
    F: Fn(f64) -> f64,
{
    let final_price = discount_fn(price);
    println!("ราคาเดิม {:.2} บาท -> ราคาสุทธิ {:.2} บาท", price, final_price);
    final_price
}

fn main() {
    let price = 1500.0;

    // closure ที่ไม่ capture อะไรเลย — เทียบเท่ากับ NoDiscount struct แต่ไม่ต้องประกาศ struct ใหม่เลย
    checkout_with(price, |p| p);

    // closure ที่ capture ตัวแปรจากสิ่งแวดล้อมแบบ borrow (Part 24) — เทียบเท่ากับ PercentageDiscount { percent }
    let percent = 10.0;
    checkout_with(price, |p| p * (1.0 - percent / 100.0));

    // เก็บ closure หลายตัวไว้ใน Vec ได้ ผ่าน Box<dyn Fn(...)> (ทวนจาก Part 21: Box<dyn Fn> ก็คือ trait object
    // ธรรมดาตัวหนึ่ง ไม่มีอะไรพิเศษไปกว่า Box<dyn Shape>) — นี่คือจุดที่ closure กับ dyn Trait มาบรรจบกัน
    let strategies: Vec<Box<dyn Fn(f64) -> f64>> = vec![
        Box::new(|p| p),                              // ไม่มีส่วนลด
        Box::new(|p| p * 0.85),                       // ลด 15%
        Box::new(|p| if p >= 1000.0 { p - 200.0 } else { p }), // ซื้อครบ 1000 ลด 200
    ];

    println!("\nเปรียบเทียบทุก closure strategy กับราคาเดียวกัน:");
    for s in &strategies {
        checkout_with(price, s);
    }
}
```

ผลลัพธ์:

```
ราคาเดิม 1500.00 บาท -> ราคาสุทธิ 1500.00 บาท
ราคาเดิม 1500.00 บาท -> ราคาสุทธิ 1350.00 บาท

เปรียบเทียบทุก closure strategy กับราคาเดียวกัน:
ราคาเดิม 1500.00 บาท -> ราคาสุทธิ 1500.00 บาท
ราคาเดิม 1500.00 บาท -> ราคาสุทธิ 1275.00 บาท
ราคาเดิม 1500.00 บาท -> ราคาสุทธิ 1300.00 บาท
```

สังเกตว่า `Vec<Box<dyn Fn(f64) -> f64>>` ในตัวอย่างนี้แก้ปัญหา "heterogeneous strategy collection" แบบเดียวกับ
`Vec<Box<dyn DiscountStrategy>>` ใน 52.8 ได้เหมือนกันทุกประการ — เพราะ **closure ก็เป็น trait object ได้เหมือน
struct ทั่วไป** (Part 21 อธิบายไว้แล้วว่า `Box<dyn Fn(...)>` ไม่มีอะไรพิเศษไปกว่า `Box<dyn Trait>` ตัวอื่น ๆ)
ความต่างเดียวที่สำคัญคือ closure ไม่มี method `describe()` ประกบมาด้วย — ถ้าต้องการคำอธิบายของแต่ละกลยุทธ์
(เช่นเพื่อแสดงในใบเสร็จ) ก็ต้องกลับไปใช้ `trait DiscountStrategy` แบบ 52.8/52.9 เพราะนั่นคือสถานการณ์ที่ "กลยุทธ์"
มีมากกว่าหนึ่งพฤติกรรมที่ต้องเดินทางไปด้วยกันเป็นชุดจริง ๆ

### 52.11 กรอบการเลือก Strategy: เมื่อไหร่ใช้แบบไหน

ตารางสรุปกรอบคิดการเลือกที่นำกรอบคิด dispatch ของ Part 21 มาใช้ตัดสินใจจริงในสถานการณ์ที่มีชื่อเรียก:

| สถานการณ์ | ทางเลือกที่เหมาะสม | เหตุผล |
|---|---|---|
| Strategy ถูกเลือกจาก config/database/input ผู้ใช้ตอน runtime | `Box<dyn Trait>` | ต้อง dynamic dispatch เพราะไม่รู้ concrete type ตอน compile time |
| ต้องเก็บ strategy หลายชนิดปนกันใน collection เดียว | `Box<dyn Trait>` (หรือ `Box<dyn Fn>`) | generic ทำ heterogeneous collection ไม่ได้ตามกฎพื้นฐานของ Part 21 |
| รู้ strategy ตอน compile time แน่นอน, อยู่ใน hot path ที่ performance สำคัญมาก | Generic + trait bound | Zero-cost, ไม่มี indirect call, เปิดโอกาส inline เต็มที่ |
| Strategy คือฟังก์ชันเดียว ไม่มีพฤติกรรมอื่นประกบ | Closure (`Fn`) | เบากว่า ไม่ต้องสร้าง trait+struct ทั้งชุดสำหรับ logic ง่าย ๆ |
| Strategy มีหลาย method ที่ต้องมาด้วยกันเสมอ (เช่น `apply()` + `describe()` + `priority()`) | `trait` (แล้วเลือก `dyn`/generic ตามบริบท runtime/compile time) | closure เดียวเก็บได้แค่ฟังก์ชันเดียว ไม่พอสำหรับกลุ่มพฤติกรรม |
| ไม่แน่ใจ เริ่มจากอะไรก่อนดี | เริ่มจาก `Box<dyn Trait>` แล้ววัด performance จริงก่อนเปลี่ยน | ตรงกับหลัก "อย่า optimize ก่อนรู้ว่าช้าจริง" ที่ Part 21 สอนไว้แล้ว |

### 52.12 Observer Pattern: ปัญหาที่ต้องแก้และเวอร์ชันดั้งเดิมด้วย Trait Object

**Observer pattern** แก้ปัญหาการ **notify หลายส่วนของระบบเมื่อมีการเปลี่ยนแปลงเกิดขึ้น โดยไม่ให้ต้นทาง (publisher)
ผูกติดแน่น (tightly coupled) กับปลายทาง (subscriber) แต่ละตัว** — ตัวอย่างคลาสสิกคือระบบราคาหุ้น: มี `StockPrice`
เป็น publisher ที่ราคาเปลี่ยนแปลงได้ และมี subscriber หลายตัว (เช่น จอแสดงราคา, ระบบแจ้งเตือน, ระบบบันทึก log) ที่
ต้องรู้ทันทีเมื่อราคาเปลี่ยน โดยที่ `StockPrice` **ไม่ควรรู้จัก concrete type ของ subscriber แต่ละตัวเลย** (เพิ่ม
subscriber ใหม่ในอนาคตได้โดยไม่ต้องแก้โค้ด `StockPrice`)

```rust
// 52.12 - Observer pattern แบบดั้งเดิม: Vec<Box<dyn Observer>> — เหมือนระบบ plugin จาก Part 21
trait Observer {
    fn on_price_changed(&self, symbol: &str, old_price: f64, new_price: f64);
}

// subscriber แบบที่ 1: แสดงราคาบนหน้าจอ
struct PriceDisplay {
    name: String,
}

impl Observer for PriceDisplay {
    fn on_price_changed(&self, symbol: &str, old_price: f64, new_price: f64) {
        println!(
            "[{}] จอแสดงราคา {}: {:.2} -> {:.2}",
            self.name, symbol, old_price, new_price
        );
    }
}

// subscriber แบบที่ 2: แจ้งเตือนเฉพาะเมื่อราคาเปลี่ยนแปลงเกินเกณฑ์ที่กำหนด
struct PriceAlert {
    threshold_percent: f64,
}

impl Observer for PriceAlert {
    fn on_price_changed(&self, symbol: &str, old_price: f64, new_price: f64) {
        let change_percent = ((new_price - old_price) / old_price * 100.0).abs();
        if change_percent >= self.threshold_percent {
            println!(
                "*** แจ้งเตือน! {} เปลี่ยนแปลง {:.2}% (เกินเกณฑ์ {:.2}%) ***",
                symbol, change_percent, self.threshold_percent
            );
        }
    }
}

// subscriber แบบที่ 3: บันทึก log ทุกการเปลี่ยนแปลง
struct PriceLogger;

impl Observer for PriceLogger {
    fn on_price_changed(&self, symbol: &str, old_price: f64, new_price: f64) {
        println!("[LOG] {} price_change old={:.2} new={:.2}", symbol, old_price, new_price);
    }
}

// publisher: เก็บ observer ไว้ใน Vec<Box<dyn Observer>> — ไม่รู้จัก concrete type ของ observer แต่ละตัวเลย
struct StockPrice {
    symbol: String,
    current_price: f64,
    observers: Vec<Box<dyn Observer>>,
}

impl StockPrice {
    fn new(symbol: impl Into<String>, initial_price: f64) -> Self {
        StockPrice { symbol: symbol.into(), current_price: initial_price, observers: Vec::new() }
    }

    // เพิ่ม observer ใหม่ได้เรื่อย ๆ โดยไม่ต้องแก้โค้ด StockPrice เลย — นี่คือหัวใจของ "loose coupling"
    fn subscribe(&mut self, observer: Box<dyn Observer>) {
        self.observers.push(observer);
    }

    // เมื่อราคาเปลี่ยน: วน notify observer ทุกตัวที่ subscribe ไว้ ตามลำดับที่ subscribe (synchronous ล้วน ๆ)
    fn set_price(&mut self, new_price: f64) {
        let old_price = self.current_price;
        self.current_price = new_price;
        for observer in &self.observers {
            observer.on_price_changed(&self.symbol, old_price, new_price);
        }
    }
}

fn main() {
    let mut stock = StockPrice::new("RUST", 100.0);

    stock.subscribe(Box::new(PriceDisplay { name: "หน้าจอหลัก".to_string() }));
    stock.subscribe(Box::new(PriceAlert { threshold_percent: 5.0 }));
    stock.subscribe(Box::new(PriceLogger));

    println!("-- เปลี่ยนราคาครั้งที่ 1 (เปลี่ยนน้อย ไม่ควรเตือน) --");
    stock.set_price(102.0);

    println!("\n-- เปลี่ยนราคาครั้งที่ 2 (เปลี่ยนมาก ควรเตือน) --");
    stock.set_price(115.0);
}
```

ผลลัพธ์:

```
-- เปลี่ยนราคาครั้งที่ 1 (เปลี่ยนน้อย ไม่ควรเตือน) --
[หน้าจอหลัก] จอแสดงราคา RUST: 100.00 -> 102.00
[LOG] RUST price_change old=100.00 new=102.00

-- เปลี่ยนราคาครั้งที่ 2 (เปลี่ยนมาก ควรเตือน) --
[หน้าจอหลัก] จอแสดงราคา RUST: 102.00 -> 115.00
*** แจ้งเตือน! RUST เปลี่ยนแปลง 12.75% (เกินเกณฑ์ 5.00%) ***
[LOG] RUST price_change old=102.00 new=115.00
```

สังเกตว่านี่คือ pattern เดียวกันกับ "ระบบ plugin" ที่ Part 21 แนะนำไว้ตอนท้ายบท (`Vec<Box<dyn Trait>>` ที่รับ
type ใหม่เข้ามาได้เรื่อย ๆ โดยไม่ต้องแก้โค้ดเดิม) — เพียงแค่มาปรากฏในบริบทที่มีชื่อเรียกเฉพาะว่า "Observer pattern"
เท่านั้น `StockPrice::subscribe()` รับ `Box<dyn Observer>` ใดก็ได้ ทำให้เพิ่ม observer ชนิดใหม่ในอนาคต (เช่น
`PriceToDatabase` ที่บันทึกลง database) ได้โดยไม่ต้องแก้โค้ด `StockPrice` แม้แต่บรรทัดเดียว — นี่คือความหมายของ
"loose coupling": publisher กับ subscriber เชื่อมกันผ่าน **สัญญา** (`trait Observer`) เท่านั้น ไม่รู้จัก
รายละเอียดกันและกันเลย

### 52.13 ข้อจำกัดของ Observer แบบ Trait Object และทางเลือกที่ "เป็น Rust" มากกว่า: Channel

Observer pattern แบบ 52.12 ทำงานได้ดีมากในสถานการณ์ **synchronous, in-process** (ทุกอย่างรันใน thread เดียว
ไม่มีการข้าม thread) แต่มีข้อจำกัดที่สำคัญเมื่อระบบเริ่มซับซ้อนขึ้น:

- **การเรียก observer ทุกตัวเป็นแบบ blocking**: `set_price()` ต้องรอ observer ทุกตัวทำงานเสร็จตามลำดับก่อนจะ
  return กลับไป — ถ้า observer ตัวใดตัวหนึ่งทำงานช้า (เช่นเขียนไฟล์ หรือส่ง network request) จะบล็อกทุกอย่างที่
  ต่อจากนี้
- **ownership ซับซ้อนขึ้นทันทีถ้าต้องข้าม thread**: ถ้าอยากให้ observer ทำงานใน thread แยก (Part 37) ต้องเอา
  `Vec<Box<dyn Observer>>` ไปแบ่งกันใช้ระหว่าง thread ซึ่งต้องพึ่ง `Arc<Mutex<...>>` (Part 39) ทำให้โค้ดซับซ้อน
  ขึ้นมาก ทั้งที่จริง ๆ แล้ว "การส่งข้อความบอกว่ามีอะไรเปลี่ยนแปลง" ไม่จำเป็นต้อง share memory ระหว่าง publisher
  กับ subscriber เลยด้วยซ้ำ
- **publisher ต้องรู้จัก trait `Observer` และเก็บ list ของมันไว้เอง**: ผูกความรับผิดชอบเรื่อง "จัดการรายชื่อผู้ฟัง"
  เข้ากับตัว publisher โดยตรง

Rust มีเครื่องมือที่ **ถูกออกแบบมาสำหรับ "ส่งข้อความบอกว่ามีอะไรเปลี่ยนแปลง" โดยเฉพาะ** อยู่แล้ว คือ **channel**
ที่เรียนมาตั้งแต่ **Part 38** (`std::sync::mpsc`) — แนวคิดคือเปลี่ยนจาก "publisher เก็บ list ของ observer แล้ว
เรียก method ของมันตรง ๆ" มาเป็น **"publisher ส่งข้อความเข้า channel แล้วปล่อยให้แต่ละ subscriber (ที่อาจอยู่คนละ
thread) ไปรับข้อความเอาเองตามจังหวะของตัวเอง"** — ข้อดีที่สำคัญคือ **publisher กับ subscriber ไม่ต้องแบ่ง memory
ร่วมกันเลย** (ไม่มี `Arc<Mutex<...>>` ที่ต้อง lock, ไม่มี reference ที่ต้องมี lifetime ตรงกัน) ส่งข้อความผ่าน
channel แล้วจบ — ตรงกับ**หลักการ ownership ของ Rust ที่สนับสนุน "ส่งข้อมูลไป" มากกว่า "แบ่งกันใช้ข้อมูลเดียวกัน"**
มาตั้งแต่ Part 37-38 อยู่แล้ว

```rust
// 52.13 - Observer pattern ด้วย channel (std::sync::mpsc จาก Part 38) — ทางเลือกที่ "เป็น Rust" มากกว่า
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

// ข้อความที่ publisher ส่งผ่าน channel — แทนที่ trait Observer ด้วย "ชนิดข้อมูล" ที่ส่งผ่านไปได้ตรง ๆ
#[derive(Debug, Clone)]
struct PriceChanged {
    symbol: String,
    old_price: f64,
    new_price: f64,
}

struct StockPrice {
    symbol: String,
    current_price: f64,
    // เก็บ Sender หลายตัว (หนึ่งตัวต่อ subscriber) — เทียบเท่ากับ Vec<Box<dyn Observer>> เดิม
    // แต่ตอนนี้เก็บแค่ "endpoint สำหรับส่งข้อความ" ไม่ได้เก็บ subscriber ตรง ๆ
    senders: Vec<mpsc::Sender<PriceChanged>>,
}

impl StockPrice {
    fn new(symbol: impl Into<String>, initial_price: f64) -> Self {
        StockPrice { symbol: symbol.into(), current_price: initial_price, senders: Vec::new() }
    }

    // subscribe คืน Receiver ให้ผู้เรียกไปวนอ่านเอาเอง (ไม่ต้อง implement trait ใด ๆ เลย)
    fn subscribe(&mut self) -> mpsc::Receiver<PriceChanged> {
        let (tx, rx) = mpsc::channel();
        self.senders.push(tx);
        rx
    }

    fn set_price(&mut self, new_price: f64) {
        let old_price = self.current_price;
        self.current_price = new_price;
        let event = PriceChanged { symbol: self.symbol.clone(), old_price, new_price };

        // ส่งข้อความให้ทุก subscriber — send() ไม่ block เลย (ต่างจากการเรียก observer.on_price_changed() ตรง ๆ
        // ที่ publisher ต้องรอ logic ข้างในทำงานจนเสร็จ) ผลคือ set_price() คืนตัวทันทีไม่ว่า subscriber จะ
        // ประมวลผลช้าแค่ไหนก็ตาม
        self.senders.retain(|tx| tx.send(event.clone()).is_ok());
    }
}

fn main() {
    let mut stock = StockPrice::new("RUST", 100.0);

    // subscriber ตัวที่ 1: จอแสดงราคา — รันใน thread แยกของตัวเอง (จำลอง component ที่แยก process/thread กัน)
    let rx1 = stock.subscribe();
    let display_handle = thread::spawn(move || {
        for event in rx1 {
            println!(
                "[thread แสดงราคา] {}: {:.2} -> {:.2}",
                event.symbol, event.old_price, event.new_price
            );
        }
    });

    // subscriber ตัวที่ 2: ระบบแจ้งเตือน — รันใน thread แยกอีกตัวหนึ่ง ทำงานอย่างอิสระจากตัวแรกโดยสิ้นเชิง
    let rx2 = stock.subscribe();
    let alert_handle = thread::spawn(move || {
        for event in rx2 {
            let change_percent = ((event.new_price - event.old_price) / event.old_price * 100.0).abs();
            if change_percent >= 5.0 {
                println!(
                    "*** [thread แจ้งเตือน] {} เปลี่ยนแปลง {:.2}% ***",
                    event.symbol, change_percent
                );
            }
        }
    });

    // set_price() คืนตัวทันที ไม่รอ thread ผู้รับทำงานเสร็จ — ต่างจาก Vec<Box<dyn Observer>> ที่บล็อกจนกว่า
    // observer ทุกตัวจะ return
    stock.set_price(102.0);
    stock.set_price(115.0);

    // ต้อง drop StockPrice (หรือทำให้ senders หมดไป) ก่อน เพื่อให้ for event in rx จบ loop ได้
    // (channel จะปิดเองเมื่อไม่มี Sender เหลืออยู่เลย — ทวนจาก Part 38)
    drop(stock);

    // รอให้ทั้งสอง thread ประมวลผลข้อความที่เหลือให้ครบก่อนโปรแกรมจบ
    display_handle.join().unwrap();
    alert_handle.join().unwrap();

    // หน่วงเล็กน้อยเพื่อให้ output ของสอง thread ไม่ปนกันในผลลัพธ์ที่แสดงในเอกสาร (ไม่จำเป็นในโค้ด production จริง)
    thread::sleep(Duration::from_millis(10));
}
```

ผลลัพธ์ (ลำดับของสอง thread อาจสลับกันได้เล็กน้อยในการรันจริง เพราะเป็น concurrent execution แต่ข้อความจากแต่ละ
thread จะเรียงตามลำดับเวลาเสมอ):

```
[thread แสดงราคา] RUST: 100.00 -> 102.00
[thread แสดงราคา] RUST: 102.00 -> 115.00
*** [thread แจ้งเตือน] RUST เปลี่ยนแปลง 12.75% ***
```

สังเกตความแตกต่างเชิงโครงสร้างที่สำคัญ: `StockPrice` ใน 52.13 **ไม่รู้จัก trait `Observer` เลย** — มันรู้จักแค่
`mpsc::Sender<PriceChanged>` ซึ่งเป็น type ธรรมดาจาก standard library ไม่ต้องมี trait พิเศษให้ผู้ subscribe ต้อง
implement เลย แค่ต้องรับ `Receiver<PriceChanged>` แล้ววนอ่านเอาเอง (`for event in rx`) — และที่สำคัญที่สุดคือ
**publisher กับ subscriber ไม่ต้อง share memory กันเลยแม้แต่ไบต์เดียว** ข้อมูลถูก **ส่ง (move)** ผ่าน channel
ไปเป็นของ subscriber โดยสมบูรณ์ (ทวนจาก Part 38: channel implement "ownership transfer" ไม่ใช่ "shared access")
ทำให้ไม่ต้องมี `Arc<Mutex<Vec<Box<dyn Observer>>>>` ที่ซับซ้อนและมีโอกาสเกิด deadlock (Part 39) เลย

ในกรณีที่ subscriber ต้องการ **ทุกคน** ได้รับข้อความเดียวกัน (broadcast แทน point-to-point) และอยู่ในบริบท async
(เช่น เขียนด้วย tokio ตาม Part 46-50) `tokio::sync::broadcast::channel()` ที่ Part 50 สอนไว้เป็นเครื่องมือที่ตรง
กับสถานการณ์นี้มากกว่า `mpsc` ธรรมดา (`mpsc` คือ "หลายคนส่ง คนรับเดียวรับ" ในความหมายเดิม แต่ในที่นี้เราใช้ pattern
"หนึ่ง channel ต่อ subscriber หนึ่งตัว" ทำให้ทุกคนได้รับ event เดียวกันผ่านคนละ channel — ใช้ได้ผลแต่สร้าง channel
ซ้ำเยอะเมื่อ subscriber มีจำนวนมาก) `broadcast::channel::<T>(capacity)` ให้ `Sender<T>` ตัวเดียวที่ `.send()` ครั้ง
เดียวแล้วทุก `Receiver` ที่ `.subscribe()` ไว้ได้รับสำเนาข้อความเดียวกันหมด (Part 50 อธิบายกลไกไว้ครบแล้ว) — เหมาะ
กับระบบ Observer ที่ต้อง broadcast ไปยัง subscriber จำนวนมากในบริบท async โดยเฉพาะ

### 52.14 เมื่อไหร่ Observer แบบ Trait Object ยังสมเหตุสมผล เมื่อไหร่ควรใช้ Channel

ทั้งสองแบบมีที่ใช้งานของตัวเอง ไม่มีแบบไหน "ถูกกว่า" อีกแบบเสมอไป — ตารางสรุปการเลือก:

| ปัจจัย | Trait Object (`Vec<Box<dyn Observer>>`) | Channel (`mpsc`/`broadcast`) |
|---|---|---|
| ทุกอย่างรันใน thread เดียว (synchronous) | เหมาะสมและง่ายกว่า | overhead เกินจำเป็น (ต้องสร้าง channel, thread, loop รับข้อความ) |
| ต้องข้าม thread หรือทำงานแบบ async | ต้องพึ่ง `Arc<Mutex<...>>` ซับซ้อนขึ้นมาก | เหมาะสมโดยธรรมชาติ ไม่ต้อง share memory เลย |
| ต้องการผลลัพธ์จาก observer กลับมาทันที (เช่น observer ต้อง validate แล้วอาจ reject การเปลี่ยนแปลง) | ทำได้ตรง ๆ (method เรียกแบบ synchronous คืนค่าได้) | ทำยากกว่า ต้องมี channel ตอบกลับอีกชุด (request-response pattern) |
| จำนวน observer เปลี่ยนแปลงบ่อย เพิ่ม/ลบขณะโปรแกรมรัน | ทำได้ตรง ๆ ผ่าน `Vec::push`/`retain` | ทำได้เหมือนกัน แต่ต้องจัดการ `Sender` ที่ตายแล้ว (`send()` คืน `Err`) เอง |
| ต้องการ decouple แบบเต็มที่ (publisher ไม่ควรรู้จัก "การมีอยู่" ของ subscriber เลย ไม่ใช่แค่ concrete type) | ยังคง coupling ระดับหนึ่ง (publisher เก็บ list ของ observer ไว้ตรง ๆ) | decouple มากกว่า (publisher แค่ส่งเข้า channel ไม่รู้ด้วยซ้ำว่ามีใครกำลังฟังอยู่) |
| ระบบเล็ก, in-process, ไม่ซับซ้อน | **เลือกอันนี้** — เข้าใจง่าย debug ง่าย | เกินความจำเป็น |
| ระบบใหญ่, cross-thread/cross-service, async-heavy | ซับซ้อนเกินจำเป็นถ้าฝืนใช้ | **เลือกอันนี้** — ตรงกับธรรมชาติของปัญหามากกว่า |

**สรุปเป็นกฎที่ใช้ได้จริง**: ถ้า observer ทั้งหมดรันในบริบทเดียวกัน (thread เดียว, ไม่มีการรอ I/O ยาว ๆ) และ
ต้องการความเรียบง่ายที่สุด — `Vec<Box<dyn Observer>>` แบบดั้งเดิมยังเป็นตัวเลือกที่ดีที่สุด (ทั้ง GoF และ Rust
เห็นตรงกันในสถานการณ์นี้) แต่ทันทีที่ต้อง**ข้าม thread**, ทำงานแบบ **async**, หรือต้องการ **decouple แบบเต็มที่**
ระหว่าง publisher กับ subscriber — channel คือทางเลือกที่ "เป็น Rust" มากกว่า เพราะสอดคล้องกับหลักการ ownership
transfer ที่ภาษาสนับสนุนมาตั้งแต่ต้น (Part 6-7) มากกว่าการพยายามแบ่ง memory ร่วมกันแบบภาษาอื่น

### 52.15 ตัวอย่างรวม: ระบบคำสั่งซื้อ E-commerce ที่ผสมทั้งสาม Pattern

มาถึงตัวอย่างสุดท้ายของบทนี้ — ระบบคำสั่งซื้อที่ผสม **Builder** (สร้าง `Order`), **Strategy** (`PricingStrategy`
คำนวณราคาสุทธิตามเงื่อนไขลูกค้า), และ **Observer** (แจ้งเตือนเมื่อสถานะคำสั่งซื้อเปลี่ยน) เข้าด้วยกันในระบบเดียว
เพื่อให้เห็นว่า pattern เหล่านี้ **ประกอบกันเองตามธรรมชาติ** ในโค้ด Rust จริง ไม่ใช่แค่ตัวอย่างแยกส่วนสั้น ๆ:

```rust
// 52.15 - ตัวอย่างรวม: ระบบคำสั่งซื้อ e-commerce ผสม Builder + Strategy + Observer
use std::fmt;

// ===== ส่วนที่ 1: Strategy pattern — PricingStrategy คำนวณราคาสุทธิ =====

trait PricingStrategy {
    fn calculate(&self, subtotal: f64) -> f64;
    fn name(&self) -> &str;
}

struct StandardPricing;
impl PricingStrategy for StandardPricing {
    fn calculate(&self, subtotal: f64) -> f64 {
        subtotal
    }
    fn name(&self) -> &str {
        "ราคามาตรฐาน"
    }
}

struct MemberPricing {
    discount_percent: f64,
}
impl PricingStrategy for MemberPricing {
    fn calculate(&self, subtotal: f64) -> f64 {
        subtotal * (1.0 - self.discount_percent / 100.0)
    }
    fn name(&self) -> &str {
        "ราคาสมาชิก"
    }
}

struct BulkOrderPricing {
    threshold: f64,
    flat_discount: f64,
}
impl PricingStrategy for BulkOrderPricing {
    fn calculate(&self, subtotal: f64) -> f64 {
        if subtotal >= self.threshold {
            (subtotal - self.flat_discount).max(0.0)
        } else {
            subtotal
        }
    }
    fn name(&self) -> &str {
        "ราคาสั่งซื้อจำนวนมาก"
    }
}

// ===== ส่วนที่ 2: Observer pattern — แจ้งเตือนเมื่อสถานะคำสั่งซื้อเปลี่ยน =====

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
enum OrderStatus {
    Pending,
    Paid,
    Shipped,
    Delivered,
    Cancelled,
}

impl fmt::Display for OrderStatus {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        let text = match self {
            OrderStatus::Pending => "รอดำเนินการ",
            OrderStatus::Paid => "ชำระเงินแล้ว",
            OrderStatus::Shipped => "จัดส่งแล้ว",
            OrderStatus::Delivered => "ส่งถึงแล้ว",
            OrderStatus::Cancelled => "ยกเลิก",
        };
        write!(f, "{}", text)
    }
}

trait OrderObserver {
    fn on_status_changed(&self, order_id: u32, old_status: OrderStatus, new_status: OrderStatus);
}

struct EmailNotifier {
    customer_email: String,
}
impl OrderObserver for EmailNotifier {
    fn on_status_changed(&self, order_id: u32, old_status: OrderStatus, new_status: OrderStatus) {
        println!(
            "[อีเมลถึง {}] คำสั่งซื้อ #{}: {} -> {}",
            self.customer_email, order_id, old_status, new_status
        );
    }
}

struct WarehouseNotifier;
impl OrderObserver for WarehouseNotifier {
    fn on_status_changed(&self, order_id: u32, _old_status: OrderStatus, new_status: OrderStatus) {
        if new_status == OrderStatus::Paid {
            println!("[คลังสินค้า] เตรียมจัดสินค้าสำหรับคำสั่งซื้อ #{}", order_id);
        }
    }
}

struct AuditLogger;
impl OrderObserver for AuditLogger {
    fn on_status_changed(&self, order_id: u32, old_status: OrderStatus, new_status: OrderStatus) {
        println!("[AUDIT] order_id={} {:?} -> {:?}", order_id, old_status, new_status);
    }
}

// ===== ส่วนที่ 3: Builder pattern — OrderBuilder สร้าง Order =====

#[derive(Debug, Clone)]
struct OrderLine {
    product_name: String,
    unit_price: f64,
    quantity: u32,
}

impl OrderLine {
    fn line_total(&self) -> f64 {
        self.unit_price * self.quantity as f64
    }
}

struct Order {
    id: u32,
    customer_name: String,
    lines: Vec<OrderLine>,
    pricing_strategy: Box<dyn PricingStrategy>,
    status: OrderStatus,
    observers: Vec<Box<dyn OrderObserver>>,
}

impl Order {
    fn subtotal(&self) -> f64 {
        self.lines.iter().map(OrderLine::line_total).sum()
    }

    fn final_price(&self) -> f64 {
        self.pricing_strategy.calculate(self.subtotal())
    }

    // เปลี่ยนสถานะคำสั่งซื้อ แล้ว notify observer ทุกตัว — จุดที่ Observer pattern เข้ามาทำงาน
    fn set_status(&mut self, new_status: OrderStatus) {
        let old_status = self.status;
        self.status = new_status;
        for observer in &self.observers {
            observer.on_status_changed(self.id, old_status, new_status);
        }
    }

    fn print_receipt(&self) {
        println!("=== ใบเสร็จคำสั่งซื้อ #{} ({}) ===", self.id, self.customer_name);
        for line in &self.lines {
            println!(
                "  {} x{} @ {:.2} = {:.2} บาท",
                line.product_name, line.quantity, line.unit_price, line.line_total()
            );
        }
        println!("  ยอดรวม: {:.2} บาท", self.subtotal());
        println!(
            "  กลยุทธ์ราคา: {} -> ยอดสุทธิ: {:.2} บาท",
            self.pricing_strategy.name(),
            self.final_price()
        );
        println!("  สถานะ: {}", self.status);
    }
}

#[derive(Debug)]
enum OrderBuilderError {
    NoLines,
    MissingCustomerName,
}

impl fmt::Display for OrderBuilderError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            OrderBuilderError::NoLines => write!(f, "คำสั่งซื้อต้องมีสินค้าอย่างน้อยหนึ่งรายการ"),
            OrderBuilderError::MissingCustomerName => write!(f, "ต้องระบุชื่อลูกค้า"),
        }
    }
}

impl std::error::Error for OrderBuilderError {}

// OrderBuilder: consuming builder ตามหลักการที่เรียนในหัวข้อ 52.3
struct OrderBuilder {
    id: u32,
    customer_name: Option<String>,
    lines: Vec<OrderLine>,
    // ค่า default เป็น StandardPricing (ไม่มีส่วนลด) ถ้าไม่ระบุ strategy — ไม่บังคับต้องเลือกทุกครั้ง
    pricing_strategy: Box<dyn PricingStrategy>,
    observers: Vec<Box<dyn OrderObserver>>,
}

impl OrderBuilder {
    fn new(id: u32) -> Self {
        OrderBuilder {
            id,
            customer_name: None,
            lines: Vec::new(),
            pricing_strategy: Box::new(StandardPricing),
            observers: Vec::new(),
        }
    }

    fn customer_name(mut self, name: impl Into<String>) -> Self {
        self.customer_name = Some(name.into());
        self
    }

    fn add_line(mut self, product_name: impl Into<String>, unit_price: f64, quantity: u32) -> Self {
        self.lines.push(OrderLine { product_name: product_name.into(), unit_price, quantity });
        self
    }

    // รับ strategy ผ่าน Box<dyn PricingStrategy> — เลือกได้ตอน runtime ตามเงื่อนไขลูกค้า (หัวข้อ 52.8)
    fn pricing_strategy(mut self, strategy: Box<dyn PricingStrategy>) -> Self {
        self.pricing_strategy = strategy;
        self
    }

    // ผูก observer เข้ากับ order ตั้งแต่ตอนสร้าง (หัวข้อ 52.12) — เรียกซ้ำได้หลายครั้งเพื่อเพิ่มหลาย observer
    fn add_observer(mut self, observer: Box<dyn OrderObserver>) -> Self {
        self.observers.push(observer);
        self
    }

    fn build(self) -> Result<Order, OrderBuilderError> {
        if self.lines.is_empty() {
            return Err(OrderBuilderError::NoLines);
        }
        let customer_name = self.customer_name.ok_or(OrderBuilderError::MissingCustomerName)?;

        Ok(Order {
            id: self.id,
            customer_name,
            lines: self.lines,
            pricing_strategy: self.pricing_strategy,
            status: OrderStatus::Pending,
            observers: self.observers,
        })
    }
}

fn main() -> Result<(), OrderBuilderError> {
    // ลูกค้าสมาชิก VIP สั่งซื้อสินค้าหลายรายการ — ใช้ MemberPricing เป็น strategy
    let mut order = OrderBuilder::new(1001)
        .customer_name("คุณสมชาย")
        .add_line("คีย์บอร์ดกลไก", 2500.0, 1)
        .add_line("เมาส์ไร้สาย", 890.0, 2)
        .pricing_strategy(Box::new(MemberPricing { discount_percent: 10.0 }))
        .add_observer(Box::new(EmailNotifier { customer_email: "somchai@example.com".to_string() }))
        .add_observer(Box::new(WarehouseNotifier))
        .add_observer(Box::new(AuditLogger))
        .build()?;

    order.print_receipt();

    println!();
    // เปลี่ยนสถานะคำสั่งซื้อไปตามขั้นตอนจริง — ทุกครั้งที่เปลี่ยน observer ทั้งสามตัวจะถูก notify ตามลำดับ
    order.set_status(OrderStatus::Paid);
    order.set_status(OrderStatus::Shipped);
    order.set_status(OrderStatus::Delivered);

    println!();
    // คำสั่งซื้อที่สอง: ลูกค้าทั่วไปสั่งซื้อจำนวนมาก — ใช้ BulkOrderPricing แทน ไม่ต้องผูกกับ EmailNotifier
    // (สังเกตว่าแต่ละ Order เลือก strategy และ observer ของตัวเองได้อย่างอิสระ ผ่าน builder เดียวกัน)
    let mut order2 = OrderBuilder::new(1002)
        .customer_name("คุณสมหญิง")
        .add_line("จอมอนิเตอร์ 27 นิ้ว", 6500.0, 3)
        .pricing_strategy(Box::new(BulkOrderPricing { threshold: 15000.0, flat_discount: 1500.0 }))
        .add_observer(Box::new(AuditLogger))
        .build()?;

    order2.print_receipt();
    println!();
    order2.set_status(OrderStatus::Cancelled);

    Ok(())
}
```

ผลลัพธ์:

```
=== ใบเสร็จคำสั่งซื้อ #1001 (คุณสมชาย) ===
  คีย์บอร์ดกลไก x1 @ 2500.00 = 2500.00 บาท
  เมาส์ไร้สาย x2 @ 890.00 = 1780.00 บาท
  ยอดรวม: 4280.00 บาท
  กลยุทธ์ราคา: ราคาสมาชิก -> ยอดสุทธิ: 3852.00 บาท
  สถานะ: รอดำเนินการ

[อีเมลถึง somchai@example.com] คำสั่งซื้อ #1001: รอดำเนินการ -> ชำระเงินแล้ว
[คลังสินค้า] เตรียมจัดสินค้าสำหรับคำสั่งซื้อ #1001
[AUDIT] order_id=1001 Pending -> Paid
[อีเมลถึง somchai@example.com] คำสั่งซื้อ #1001: ชำระเงินแล้ว -> จัดส่งแล้ว
[AUDIT] order_id=1001 Paid -> Shipped
[อีเมลถึง somchai@example.com] คำสั่งซื้อ #1001: จัดส่งแล้ว -> ส่งถึงแล้ว
[AUDIT] order_id=1001 Shipped -> Delivered

=== ใบเสร็จคำสั่งซื้อ #1002 (คุณสมหญิง) ===
  จอมอนิเตอร์ 27 นิ้ว x3 @ 6500.00 = 19500.00 บาท
  ยอดรวม: 19500.00 บาท
  กลยุทธ์ราคา: ราคาสั่งซื้อจำนวนมาก -> ยอดสุทธิ: 18000.00 บาท
  สถานะ: รอดำเนินการ

[AUDIT] order_id=1002 Pending -> Cancelled
```

**สังเกตว่าทั้งสาม pattern ทำงานร่วมกันโดยไม่มีจุดใดขัดแย้งกันเลย**: `OrderBuilder` (Builder) รับผิดชอบแค่ "ประกอบ
`Order` ให้ครบถูกต้อง" — มันไม่สนใจว่า `PricingStrategy` ที่ส่งเข้ามาเป็นตัวไหน (Strategy) หรือ `OrderObserver`
ที่เพิ่มเข้ามาจะทำอะไรตอนสถานะเปลี่ยน (Observer) เพียงแค่**เก็บ**สิ่งที่ผู้ใช้ส่งเข้ามาให้ครบถ้วนแล้วประกอบเป็น
`Order` เดียว — ส่วน `Order::final_price()` เรียก `self.pricing_strategy.calculate(...)` โดยไม่รู้ concrete type
ของ strategy เลย (dynamic dispatch ตาม Part 21) และ `Order::set_status()` วน notify `self.observers` โดยไม่รู้
concrete type ของ observer เลยเช่นกัน — **นี่คือเหตุผลที่ pattern เหล่านี้ "ประกอบกันได้" ในโค้ดจริง**: แต่ละ
pattern แก้ปัญหาคนละมุมของระบบ (การสร้าง, การเลือกอัลกอริทึม, การแจ้งเตือน) โดยไม่ก้าวก่ายความรับผิดชอบของกันและกัน
เลย — หลักการ **separation of concerns** ที่เป็นแกนกลางของการออกแบบซอฟต์แวร์ที่ดี ไม่ว่าจะเขียนด้วยภาษาไหนก็ตาม

## กับดักที่พบบ่อย (Common Pitfalls)

**1. ลืมว่า consuming builder "ยึด" ตัวแปรไปแล้ว พยายามเรียก method บนตัวแปรเดิมอีก**

```rust
struct Builder {
    value: i32,
}

impl Builder {
    fn new() -> Self {
        Builder { value: 0 }
    }
    fn set(mut self, v: i32) -> Self {
        self.value = v;
        self
    }
}

fn main() {
    let builder = Builder::new();
    let b1 = builder.set(1); // builder ถูก "ยึด" (moved) เข้าไปใน set() แล้ว
    let b2 = builder.set(2); // error: use of moved value: `builder`
}
```

Error จริง:

```
error[E0382]: use of moved value: `builder`
  --> src/main.rs:18:14
   |
16 |     let builder = Builder::new();
   |         ------- move occurs because `builder` has type `Builder`, which does not implement the `Copy` trait
17 |     let b1 = builder.set(1); // builder ถูก "ยึด" (moved) เข้าไปใน set() แล้ว
   |                      ------ `builder` moved due to this method call
18 |     let b2 = builder.set(2); // error: use of moved value: `builder`
   |              ^^^^^^^ value used here after move
   |
note: `Builder::set` takes ownership of the receiver `self`, which moves `builder`
  --> src/main.rs:9:16
   |
 9 |     fn set(mut self, v: i32) -> Self {
   |                ^^^^
```

วิธีแก้: ถ้าต้องใช้ builder ตัวเดิมสร้างหลาย variant จริง ๆ ให้เปลี่ยนไปใช้ `&mut self` builder (หัวข้อ 52.4)
หรือถ้าต้องการแค่สร้างครั้งเดียวแบบ chain ต่อกันในนิพจน์เดียว ให้เขียน `Builder::new().set(1).set(2)` ในบรรทัด
เดียวโดยไม่เก็บตัวแปรกลางทาง

**2. เขียน consuming builder แบบมีเงื่อนไข (`if`/`else`) โดยลืม reassign ตัวแปร — คิดว่าจะได้ผลลัพธ์ผิดแบบเงียบ ๆ
แต่จริง ๆ แล้ว borrow checker จับได้เสมอ**

```rust
struct Builder {
    verbose: bool,
}

impl Builder {
    fn new() -> Self {
        Builder { verbose: false }
    }
    fn verbose(mut self, v: bool) -> Self {
        self.verbose = v;
        self
    }
}

fn main() {
    let debug_mode = true;
    let mut builder = Builder::new();
    if debug_mode {
        builder.verbose(true); // ผลลัพธ์ของ .verbose(true) หายไปเลย! ไม่ได้ reassign กลับเข้า builder
    }
    println!("{}", builder.verbose); // ตั้งใจจะอ่านค่าต่อ แต่ builder ถูก "ยึด" ไปแล้วในบรรทัดข้างบน
}
```

ผู้ที่เคยเขียนภาษาอื่นที่มี mutable object (เช่น Java/Python) อาจคาดหวังว่าโปรแกรมนี้ **"compile ผ่านแต่ได้ผลลัพธ์
ผิด"** (พิมพ์ `false` ทั้งที่ตั้งใจให้เป็น `true`) เพราะคิดว่า `builder.verbose(true)` แค่ "เรียกแล้วทิ้งผลลัพธ์"
เหมือนเรียก method ที่ mutate ตัวเองแบบภาษาอื่น — **แต่ Rust ไม่ยอมให้ถึงจุดนั้นด้วยซ้ำ**: เพราะ `verbose(mut self,
...)` รับ `self` แบบ **by value** การเรียก `builder.verbose(true)` จึง **ยึด (move) ตัวแปร `builder` เข้าไปในฟังก์ชัน
ทันที** ไม่ว่าค่าที่ return ออกมาจะถูกเก็บไว้หรือถูกทิ้งก็ตาม — ผลคือบรรทัดถัดมาที่พยายามอ่าน `builder.verbose` จะ
เจอ **compile error** ก่อนโปรแกรมจะได้รันด้วยซ้ำ:

```
error[E0382]: borrow of moved value: `builder`
  --> src/main.rs:21:20
   |
17 |     let mut builder = Builder::new();
   |         ----------- move occurs because `builder` has type `Builder`, which does not implement the `Copy` trait
18 |     if debug_mode {
19 |         builder.verbose(true); // ผลลัพธ์ของ .verbose(true) หายไปเลย! ไม่ได้ reassign กลับเข้า builder
   |                 ------------- `builder` moved due to this method call
20 |     }
21 |     println!("{}", builder.verbose); // ตั้งใจจะอ่านค่าต่อ แต่ builder ถูก "ยึด" ไปแล้วในบรรทัดข้างบน
   |                    ^^^^^^^^^^^^^^^ value borrowed here after move
   |
note: `Builder::verbose` takes ownership of the receiver `self`, which moves `builder`
  --> src/main.rs:9:20
   |
 9 |     fn verbose(mut self, v: bool) -> Self {
   |                    ^^^^
```

**นี่คือจุดที่ควรเห็นเป็นข้อดีของ Rust ไม่ใช่ความน่ารำคาญ**: ในภาษาที่ไม่มี ownership tracking บั๊กแบบนี้ (เรียก
method ที่คืนค่าใหม่แล้วลืมเก็บผลลัพธ์) จะกลายเป็น **silent logic bug** ที่ต้อง debug เองตอน runtime แต่ใน Rust
compiler ปฏิเสธโค้ดตั้งแต่ก่อนรันเสมอ ไม่ว่าเงื่อนไข `if` จะซับซ้อนแค่ไหนก็ตาม (สังเกตว่า error เกิดแม้ `if
debug_mode` จะมีโอกาสเป็น `false` ตอน runtime ก็ตาม — Rust วิเคราะห์แบบ **conservative**: ถ้ามี code path ไหนที่
อาจเกิดการ move ตัวแปรไปแล้ว การใช้ตัวแปรนั้นต่อหลังจุดที่ branch มารวมกันจะถือว่าอาจไม่ปลอดภัยเสมอ) — วิธีแก้คือ
ต้องเขียน `builder = builder.verbose(true);` เสมอ (reassign ตัวแปรทุกครั้งที่เรียก setter บน consuming builder)
หรือถ้า logic มีเงื่อนไขซับซ้อนแบบนี้บ่อย ควรพิจารณาเปลี่ยนไปใช้ `&mut self` builder แทนตั้งแต่ต้น เพราะจะไม่มี
ปัญหานี้เลย (`&mut self` builder mutate ตัวเองในที่เดิม ไม่ต้อง reassign)

**3. ทำให้ trait ของ Strategy ไม่ dyn-compatible โดยไม่ตั้งใจ แล้วงงว่าทำไมใช้ `Box<dyn Trait>` ไม่ได้**

```rust
trait PricingStrategy {
    // ผิดพลาด: เขียน method ที่ return Self แบบ by-value เข้าไปในสัญญาโดยไม่จำเป็น
    fn with_extra_discount(&self, percent: f64) -> Self;
    fn calculate(&self, subtotal: f64) -> f64;
}

struct StandardPricing;
impl PricingStrategy for StandardPricing {
    fn with_extra_discount(&self, _percent: f64) -> Self {
        StandardPricing
    }
    fn calculate(&self, subtotal: f64) -> f64 {
        subtotal
    }
}

fn use_strategy(s: Box<dyn PricingStrategy>) {
    let _ = s.calculate(100.0);
}
```

Error จริง (ตรงกับกฎที่ Part 21 หัวข้อ 21.4 อธิบายไว้):

```
error[E0038]: the trait `PricingStrategy` is not dyn compatible
  --> src/main.rs:17:24
   |
17 | fn use_strategy(s: Box<dyn PricingStrategy>) {
   |                        ^^^^^^^^^^^^^^^^^^^ `PricingStrategy` is not dyn compatible
   |
note: for a trait to be dyn compatible it needs to allow building a vtable
      for more information, visit <https://doc.rust-lang.org/reference/items/traits.html#dyn-compatibility>
  --> src/main.rs:3:52
   |
 1 | trait PricingStrategy {
   |       --------------- this trait is not dyn compatible...
 2 |     // ผิดพลาด: เขียน method ที่ return Self แบบ by-value เข้าไปในสัญญาโดยไม่จำเป็น
 3 |     fn with_extra_discount(&self, percent: f64) -> Self;
   |                                                    ^^^^ ...because method `with_extra_discount` references the `Self` type in its return type
   = help: consider moving `with_extra_discount` to another trait
   = help: only type `StandardPricing` implements `PricingStrategy`; consider using it directly instead.
```

วิธีแก้: ถ้า method แบบนี้จำเป็นต้องมี ให้เพิ่ม `where Self: Sized` (เทคนิคจาก Part 21 หัวข้อ 21.4) เพื่อยกเว้น
method นั้นออกจาก vtable โดยไม่ทำให้ trait ทั้งก้อนใช้ `dyn` ไม่ได้ หรือถ้าไม่จำเป็นจริง ๆ ให้ตัด method นั้นออก
จาก trait หลักไปเลย แล้วแยกเป็น trait/ฟังก์ชันอื่นต่างหาก — บทเรียนสำคัญคือ **ก่อนออกแบบ trait สำหรับ Strategy
pattern ที่ตั้งใจจะใช้กับ `Box<dyn Trait>` ต้องเช็คกฎ dyn compatibility ทั้งสามข้อจาก Part 21 ก่อนเสมอ**

**4. ใช้ Observer แบบ `Vec<Box<dyn Observer>>` ข้าม thread โดยไม่ผ่าน `Arc<Mutex<...>>` ให้ถูกต้อง**

```rust
use std::thread;

trait Observer: Send {
    fn notify(&self, msg: &str);
}

struct Logger;
impl Observer for Logger {
    fn notify(&self, msg: &str) {
        println!("{}", msg);
    }
}

fn main() {
    let observers: Vec<Box<dyn Observer>> = vec![Box::new(Logger)];

    let handle = thread::spawn(move || {
        for obs in &observers {
            obs.notify("จาก thread ใหม่");
        }
    });
    // พยายามใช้ observers ต่อใน main thread หลังจาก move เข้า closure ไปแล้ว
    for obs in &observers {
        obs.notify("จาก main thread");
    }
    handle.join().unwrap();
}
```

Error จริง:

```
error[E0382]: borrow of moved value: `observers`
  --> src/main.rs:23:16
   |
15 |     let observers: Vec<Box<dyn Observer>> = vec![Box::new(Logger)];
   |         --------- move occurs because `observers` has type `Vec<Box<dyn Observer>>`, which does not implement the `Copy` trait
16 |
17 |     let handle = thread::spawn(move || {
   |                                ------- value moved into closure here
18 |         for obs in &observers {
   |                     --------- variable moved due to use in closure
...
23 |     for obs in &observers {
   |                ^^^^^^^^^^ value borrowed here after move
```

นี่คือตัวอย่างที่ตอกย้ำหัวข้อ 52.13-52.14 ตรง ๆ: การพยายามใช้ `Vec<Box<dyn Observer>>` ร่วมกันข้าม thread ต้อง
ผ่าน `Arc<Mutex<Vec<Box<dyn Observer>>>>` (Part 39) เสมอ ไม่สามารถ `move` ตัวแปรเดิมเข้า thread เดียวแล้วยังใช้ที่
main thread ต่อได้ — และนี่คือเหตุผลเชิงโครงสร้างที่ทำให้ channel (ที่ **ออกแบบมาสำหรับส่งข้อมูลข้าม thread โดย
เฉพาะ**) เหมาะกับสถานการณ์แบบนี้มากกว่า `Vec<Box<dyn Observer>>` ที่ต้องพึ่ง shared mutable state ทุกครั้งที่ข้าม
thread — ถ้าจำเป็นต้องใช้ trait object ข้าม thread จริง ๆ ต้องห่อด้วย `Arc<Mutex<...>>` (หรือ `Arc<RwLock<...>>`
ถ้าอ่านบ่อยกว่าเขียน) และต้อง `.clone()` ตัว `Arc` ก่อนส่งเข้า `move` closure เสมอ ไม่ใช่ move ตัวแปรเดิมไปตรง ๆ

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** เขียน consuming builder สำหรับ struct `Pizza` ที่มี field `size: String` (required), `crust:
   String` (required), `toppings: Vec<String>` (optional, default เป็น vec เปล่า), และ `extra_cheese: bool`
   (optional, default `false`) ให้มี method `.build() -> Result<Pizza, String>` ที่ตรวจสอบว่า `size` และ
   `crust` ถูกกำหนดแล้วหรือยัง (hint: โครงสร้างเหมือน `HttpRequestBuilder` ในหัวข้อ 52.3 ทุกประการ เปลี่ยนแค่ชื่อ
   field และชนิดข้อมูล)

2. **[กลาง]** เขียน trait `SortStrategy` ที่มี method `sort(&self, data: &mut Vec<i32>)` แล้วสร้างสาม
   implementation: `BubbleSort`, `QuickSortWrapper` (ใช้ `data.sort()` ของ standard library ข้างในได้ ไม่ต้อง
   เขียน quicksort จริงจากศูนย์), และ `ReverseSort` (เรียงจากมากไปน้อย) จากนั้นเขียนฟังก์ชัน
   `sort_with(strategy: &dyn SortStrategy, data: &mut Vec<i32>)` ที่รับ strategy ผ่าน `&dyn Trait` แล้วทดสอบ
   เรียกทั้งสามกลยุทธ์กับข้อมูลชุดเดียวกัน เปรียบเทียบผลลัพธ์ที่ได้ (hint: อ้างอิงโครงสร้างจากหัวข้อ 52.8 โดยตรง
   เปลี่ยนจาก "คำนวณราคา" เป็น "เรียงข้อมูล")

3. **[ยาก]** ขยายตัวอย่าง Observer แบบ channel ในหัวข้อ 52.13 ให้ subscriber ตัวหนึ่ง (เช่น `PriceAlert`) สามารถ
   **บอกให้ publisher หยุดส่งข้อความมาให้ตัวเองได้** (unsubscribe ตัวเอง) โดยไม่กระทบ subscriber ตัวอื่น (hint:
   ใช้ข้อเท็จจริงที่ว่า `Sender::send()` คืน `Err` เมื่อฝั่ง `Receiver` ถูก drop ไปแล้ว — ให้ subscriber ปล่อย
   `Receiver` ของตัวเองทิ้งเมื่อต้องการ unsubscribe แล้วให้ `StockPrice::set_price()` ใช้ `.retain()` ที่มีอยู่
   แล้วในหัวข้อ 52.13 ลบ `Sender` ที่ส่งไม่สำเร็จออกจาก list โดยอัตโนมัติ)

4. **[ยาก/ประยุกต์ใช้งานจริง]** ขยายตัวอย่างรวมในหัวข้อ 52.15 (ระบบคำสั่งซื้อ) ให้ `OrderBuilder` ใช้เทคนิค
   typestate-adjacent จากหัวข้อ 52.6: สร้าง marker type สองตัว (`Incomplete`, `Complete`) ที่ทำให้ `.build()`
   เรียกได้เฉพาะเมื่อทั้ง `customer_name` และ `add_line` (อย่างน้อยหนึ่งครั้ง) ถูกเรียกแล้วเท่านั้น — ถ้าลืมเรียก
   อย่างใดอย่างหนึ่ง ต้องเป็น **compile error** ไม่ใช่ `Result::Err` เหมือนเดิม (hint: ต้องใช้ type parameter
   สองตัวแยกกัน หนึ่งตัวสำหรับสถานะ `customer_name`, อีกตัวสำหรับสถานะ "มีสินค้าอย่างน้อยหนึ่งรายการ" — ลองคิดดูว่า
   จะออกแบบ `impl` block ให้ method ที่ถูกต้องปรากฏเฉพาะตอนที่ type parameter ทั้งสองอยู่ในสถานะ "ครบ" พร้อมกัน
   ได้อย่างไร งานนี้เป็นตัวอย่างเกริ่นนำของสิ่งที่ Part 53 จะสอนเต็มรูปแบบ)

## สรุป

บทนี้พา Rust ไปเจอกับ pattern คลาสสิกจากโลก OOP สามตัว แล้วแสดงให้เห็นว่า **แต่ละ pattern ต้อง "ตีความใหม่" ผ่าน
เลนส์ของ ownership, trait system, และ closure ของ Rust ไม่ใช่แปลตรงจากตำรา Java** — **Builder pattern** แก้ปัญหา
constructor ที่มี argument จำนวนมากด้วยการแยกช่วง "กำลังสร้าง" (`Option<T>` ยอมให้ยังไม่ครบ) กับ "สร้างเสร็จแล้ว"
(`T` ที่การันตีว่ามีค่าเสมอ) ออกจากกัน โดยมีให้เลือกทั้งแบบ consuming (`self -> Self`, นิยมที่สุด, ไม่มีต้นทุน
clone) และแบบ `&mut self` (ยืดหยุ่นกว่าเมื่อต้องแก้ทีละนิดจากหลายจุด) พร้อมทั้งเห็นความเชื่อมโยงกับ
`#[derive(Builder)]` จาก Part 45 และเกริ่น typestate-adjacent builder ที่เปลี่ยน runtime `Err` เป็น compile error
ได้ — **Strategy pattern** มีสามรูปแบบให้เลือกตามสถานการณ์ (`Box<dyn Trait>` เมื่อเลือกตอน runtime, generic เมื่อ
รู้ตอน compile time และต้องการ zero-cost, closure เมื่อกลยุทธ์เป็นฟังก์ชันเดียว) โดยใช้กรอบคิด dispatch จาก Part
21 เป็นเกณฑ์ตัดสินใจตรง ๆ — **Observer pattern** แบบดั้งเดิม (`Vec<Box<dyn Observer>>`) ยังใช้ได้ดีในสถานการณ์
synchronous แบบง่าย แต่ channel (`mpsc` จาก Part 38, `broadcast` จาก Part 50) เป็นทางเลือกที่ "เป็น Rust" มากกว่า
เมื่อต้องข้าม thread หรือทำงานแบบ async เพราะสอดคล้องกับหลักการ ownership transfer มากกว่าการแบ่ง memory ร่วมกัน
สุดท้าย ตัวอย่างระบบคำสั่งซื้อ e-commerce แสดงให้เห็นว่าทั้งสาม pattern **ประกอบกันได้เองตามธรรมชาติ** เพราะแต่ละ
ตัวรับผิดชอบคนละส่วนของระบบอย่างชัดเจน

**Part 53** จะไปต่อกับ pattern อีกสามตัวที่ไม่มีอยู่ในตำรา GoF เลย เพราะเกิดขึ้นเฉพาะจาก type system และ ownership
ของ Rust เท่านั้น: **Newtype pattern** (ห่อ type เดิมด้วย struct ใหม่เพื่อความปลอดภัยเชิง type และแก้ orphan rule
ที่เกริ่นไว้ใน Part 21), **Typestate pattern** (ขยายจากหัวข้อ 52.6 ในบทนี้ให้เต็มรูปแบบ — ใช้ type แทนสถานะเพื่อให้
compiler ปฏิเสธการใช้งานผิดลำดับได้เอง), และ **RAII pattern** (ผูก resource cleanup เข้ากับ lifetime ของ object
ผ่าน `Drop` trait ที่ Part 27-29 เกริ่นไว้บ้างแล้ว) — ทั้งสาม pattern นี้คือสิ่งที่ทำให้โค้ด Rust "ปลอดภัยโดย
โครงสร้าง" ในแบบที่ภาษาอื่นทำได้ยากกว่ามาก

---

**Part ก่อนหน้า:** [Atomics และ Lock-free Programming](part-051-atomics-lockfree.md) | **Part ถัดไป:**
[Design Patterns ใน Rust (Newtype, Typestate, RAII)](part-053-design-patterns-2.md)
