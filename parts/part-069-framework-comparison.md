# Part 69: เปรียบเทียบ Axum vs Actix-web vs Rocket

> โมดูล: การพัฒนาเว็บแอปพลิเคชัน (Web Development) | ระดับ: กลาง | เวลาโดยประมาณ: 200 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายปรัชญาการออกแบบของ **Rocket** ได้อย่างถูกต้อง — รวมถึงข้อเท็จจริงที่หลายคนยังเข้าใจผิดว่า Rocket
  "ต้องใช้ nightly Rust เท่านั้น" (เป็นความจริงในอดีต แต่ไม่ใช่ความจริงในปัจจุบันแล้ว) — และเขียนเซิร์ฟเวอร์
  Rocket ตัวแรกที่ compile และรันได้จริงบน **stable Rust** ด้วย `#[launch]`, `#[get("/")]`, และ
  `rocket::build()`
- เขียนเซิร์ฟเวอร์ Rocket ที่มี path parameter (`#[get("/tasks/<id>")]`), query parameter
  (`#[get("/tasks?<done>")]`), คืนค่า JSON ผ่าน `rocket::serde::json::Json<T>`, และแชร์ state ระหว่าง
  request ด้วย `rocket::State<T>` ที่ผูกผ่าน `.manage(...)` — ทดสอบทุก endpoint ด้วย `curl` จริง
- เทียบโค้ด endpoint "ดึงข้อมูลหนึ่งชิ้นตาม ID แล้วคืนเป็น JSON" ที่เขียนด้วย Axum, Actix-web, และ Rocket
  วางเคียงข้างกัน แล้วอธิบายความแตกต่างเชิง idiom, แนวคิดเบื้องหลัง (extractor เทียบกับ request guard), และ
  พฤติกรรมจริงที่สังเกตได้เมื่อ input ผิดรูปแบบ (เช่น status code ที่แต่ละเฟรมเวิร์กเลือกตอบกลับเมื่อ path
  parameter parse ไม่ผ่าน — ทั้งสามตอบต่างกัน และเราจะพิสูจน์ด้วย `curl` จริง)
- อ่านตารางเปรียบเทียบสถาปัตยกรรมเชิงลึกของทั้งสามเฟรมเวิร์ก (routing, extractor/guard, middleware, state
  management, error handling, compile-time route checking, การพึ่งพา `tower`/`hyper`) และอธิบายได้ว่าทำไม
  แต่ละจุดต่างกันจึงเป็นผลมาจาก **การตัดสินใจด้านสถาปัตยกรรมที่ต่างกันตั้งแต่ต้น** ไม่ใช่ว่าเฟรมเวิร์กใด
  "ทำได้ไม่ดี"
- ใช้ decision framework (ตาราง/flowchart เชิงข้อความ) เพื่อเลือกเฟรมเวิร์กที่เหมาะกับโปรเจกต์จริงของตัวเอง
  ในอนาคต — โดยไม่ยึดติดกับตัวเลข benchmark เพียงอย่างเดียว และเข้าใจว่าอะไรคือ "portable" (ย้าย
  เฟรมเวิร์กได้ง่าย) กับอะไรคือ "framework-specific" (ย้ายยาก) ในแอปพลิเคชัน Rust ทั่วไป
- อธิบายได้ว่าทำไมหลักสูตรนี้เลือกเดินหน้าต่อด้วย **Axum** เป็นเฟรมเวิร์กหลักตั้งแต่ Part 70 เป็นต้นไป — และ
  เข้าใจว่านี่เป็น **การตัดสินใจเชิงหลักสูตร** ไม่ใช่การอ้างว่า Axum เป็นเฟรมเวิร์กที่ "ดีที่สุด" แบบสัมบูรณ์

## ความรู้ที่ต้องมีมาก่อน

- **Part 61 (HTTP Fundamentals และ REST API Concepts)**: บทนี้อ้างอิงกลับไปตลอดเรื่อง HTTP method, status
  code, request/response cycle — ถ้าพื้นฐานนี้ไม่แน่น การเปรียบเทียบเรื่อง status code ที่แต่ละเฟรมเวิร์ก
  เลือกตอบต่างกันจะดูเป็นเรื่องจุกจิกไม่มีเหตุผล ทั้งที่จริงมันสะท้อนปรัชญาการออกแบบที่ต่างกัน
- **Part 62-66 (Axum เต็มรูปแบบ)**: บทนี้ถือว่าคุณเขียนแอป Axum จริงมาแล้วหลายตัว รู้จัก `Router`,
  extractor (`Path`, `State`, `Json`), middleware ผ่าน `tower`/`tower-http`, และ error handling แบบมือ
  อาชีพผ่าน `IntoResponse` ที่ implement เอง — บทนี้จะไม่สอนแนวคิดเหล่านี้ใหม่ แต่จะใช้เป็น "จุดอ้างอิง" ตลอด
  เวลาเทียบกับ Actix-web และ Rocket
- **Part 67-68 (Actix-web เต็มรูปแบบ)**: เช่นเดียวกัน บทนี้ถือว่าคุณเขียน handler, route, middleware ด้วย
  Actix-web มาแล้วจริง รู้จัก `web::Data`, `web::Path`, `web::Json`, และระบบ `App`/`HttpServer` — บทนี้จะไม่
  ทวนไวยากรณ์พื้นฐานของ Actix-web ใหม่ แต่จะสรุปเชิงสถาปัตยกรรมและเทียบกับอีกสองเฟรมเวิร์ก
- **Part 48-49 (Tokio: Runtime และ I/O/Networking)**: จำเป็นสำหรับเข้าใจว่า "async runtime" คืออะไร เพราะ
  ความแตกต่างที่สำคัญที่สุดข้อหนึ่งระหว่างสามเฟรมเวิร์กคือวิธีที่แต่ละตัวใช้งาน async runtime — Axum และ
  Rocket วิ่งอยู่บน Tokio ตรงๆ ส่วน Actix-web มี runtime ของตัวเอง (`actix-rt`) ที่สร้างอยู่บน Tokio อีกที
  แต่มีโมเดล worker/thread ที่ต่างออกไป
- **Part 57-58 (Serde เบื้องต้นและขั้นสูง)**: ทั้งสามเฟรมเวิร์กใช้ `serde`/`serde_json` เป็นกลไก
  serialize/deserialize JSON เหมือนกันหมด — struct ที่ derive `Serialize`/`Deserialize` แบบเดียวกันใช้ข้าม
  เฟรมเวิร์กได้ทันที นี่คือหนึ่งในเหตุผลหลักที่บทท้ายๆของบทนี้จะบอกว่า "domain model ย้ายเฟรมเวิร์กได้ง่าย"
- **Part 39 (Smart Pointers: `Arc`/`Mutex`)**: ตัวอย่าง state ที่ใช้เทียบกันในบทนี้ (Tasks API) ใช้
  `Arc<Mutex<Vec<Task>>>` แบบเดียวกับ Part 62 — ถ้าไม่แน่นเรื่องนี้ กลับไปทวนก่อน เพราะบทนี้ไม่อธิบาย
  mechanism ของ `Mutex`/`Arc` ซ้ำอีก
- **Part 30-31 (Error Handling: `thiserror`/`anyhow`)**: ใช้อ้างอิงตอนพูดถึง error handling ergonomics ของ
  แต่ละเฟรมเวิร์ก

## เนื้อหา

### 69.1 ทวนทางที่เดินมา: จาก Axum และ Actix-web สู่คำถาม "แล้วควรเลือกอันไหน"

ถึงจุดนี้คุณได้เขียนเซิร์ฟเวอร์ HTTP จริงด้วยมือสองครั้งด้วยสองเฟรมเวิร์กที่ต่างกันโดยสิ้นเชิงในเชิง
สถาปัตยกรรม — **Axum** (Part 62-66) ที่สร้างอยู่บน `tower` + `hyper` และเน้นความ "explicit" ผ่าน
trait-composition, และ **Actix-web** (Part 67-68) ที่มี runtime และระบบ routing เป็นของตัวเอง เน้น macro
attribute (`#[get(...)]`) และ actor-model ที่สืบทอดมาจากรากฐานเดิมของโปรเจกต์ ถ้าคุณลงมือเขียนโค้ดทั้งสอง
เฟรมเวิร์กจริงมาแล้ว คุณน่าจะสัมผัสได้ถึงความแตกต่างในบรรยากาศการเขียนโค้ด (feel) ทั้งที่ทั้งคู่แก้ปัญหา
เดียวกัน — "รับ HTTP request แล้วตอบ HTTP response ให้ถูกต้องและเร็ว"

คำถามที่เกิดขึ้นตามธรรมชาติคือ **"แล้วควรเลือกอันไหนสำหรับโปรเจกต์จริง?"** — คำถามนี้ตอบได้ไม่ดีถ้าตอบจาก
เฟรมเวิร์กเดียว เพราะคุณไม่มี baseline สำหรับเทียบ บทนี้จึงทำสองอย่างพร้อมกัน:

1. **แนะนำเฟรมเวิร์กที่สามคือ Rocket** ในระดับที่พอให้คุณเห็นภาพจริง (compile และรันจริง ไม่ใช่แค่อ่านเชิง
   ทฤษฎี) — Rocket ไม่ได้ถูกสอนเป็นบทเต็มในหลักสูตรนี้เหมือน Axum/Actix-web เพราะเป้าหมายของบทนี้ไม่ใช่ให้
   คุณเชี่ยวชาญ Rocket แต่ให้คุณมี**ข้อมูลอ้างอิงที่สาม** ที่มากพอจะเข้าใจสเปกตรัมของการออกแบบเฟรมเวิร์กใน
   Rust ทั้งหมด (Axum อยู่ปลายด้าน "explicit/generic-heavy", Rocket อยู่ปลายด้าน "ergonomic/macro-heavy",
   Actix-web อยู่กลางๆ แต่เอียงไปทาง performance-first ด้วย runtime ของตัวเอง)
2. **สังเคราะห์ (synthesize)** สิ่งที่คุณเรียนมาทั้งหมดให้เป็นกรอบการตัดสินใจที่ใช้ได้จริง ไม่ใช่แค่ตาราง
   feature เทียบกันแบบตื้นๆ

บทนี้จึงมีบรรยากาศต่างจาก Part 62-68 ที่ผ่านมา — บทก่อนหน้าสอน "วิธีเขียน" เฟรมเวิร์กหนึ่งตัวให้ลึก ส่วนบท
นี้สอน "วิธีคิด" เมื่อต้องเลือกระหว่างเฟรมเวิร์กหลายตัวที่ล้วนใช้งานได้จริงในระดับ production ทั้งหมด — ไม่มี
เฟรมเวิร์กไหนใน "สามตัวนี้" ที่เป็นตัวเลือกผิด เพียงแต่แต่ละตัวเหมาะกับสถานการณ์ที่ต่างกัน

### 69.2 แนะนำ Rocket: ปรัชญา "batteries-included" และประวัติศาสตร์เรื่อง nightly Rust

**Rocket** เป็นเว็บเฟรมเวิร์กของ Rust ที่เก่าแก่มากตัวหนึ่ง (เริ่มพัฒนาตั้งแต่ปี 2016 โดย Sergio Benitez)
และมีชื่อเสียงในสองเรื่องที่ตรงกันข้ามกันในทางประวัติศาสตร์:

**เรื่องที่หนึ่ง — ergonomics ที่ดีมากตั้งแต่วันแรก**: Rocket ใช้ attribute macro อย่างหนักเพื่อให้โค้ดที่
เขียนออกมา "อ่านแล้วรู้เลยว่า route นี้ทำอะไร" — `#[get("/tasks/<id>")]` บอกทั้ง HTTP method และ path
parameter ในบรรทัดเดียว ต่างจาก Axum ที่ต้องเขียน `.route("/tasks/{id}", get(handler))` แยกเป็นสองส่วน
(attribute ของ route กับการลงทะเบียนเข้า `Router`) ปรัชญานี้ทำให้ Rocket ถูกอธิบายบ่อยครั้งว่าเป็นเฟรมเวิร์ก
ที่ **"ให้ framework ทำงานให้มากที่สุด"** — ไม่ใช่แค่ routing แต่รวมถึง response type ที่มีให้ครบ (form
parsing, cookie, session, TLS, ไปจนถึง CSRF protection แบบ built-in) มากกว่าที่ Axum หรือ Actix-web ให้มา
โดยตรงในตัวเฟรมเวิร์กเอง (ทั้งสองมักผลักงานเหล่านี้ไปให้ crate เสริม)

**เรื่องที่สอง — ประวัติศาสตร์การพึ่งพา nightly Rust**: ในช่วงหลายปีแรก (0.1 ถึง 0.4) Rocket ใช้ฟีเจอร์ของ
compiler ที่ยังไม่ stabilize เข้า stable Rust (โดยเฉพาะ procedural macro บางรูปแบบ และ
`#[proc_macro_attribute]` ยุคแรกๆที่ API ยังไม่เสถียร) ทำให้ผู้ใช้ **ต้องสับ toolchain ไปที่ nightly Rust**
เพื่อ compile โปรเจกต์ที่ใช้ Rocket ได้ — นี่คือเหตุผลที่ Rocket มีชื่อเสียง (หรือ "ชื่อเสีย" ในมุมของทีมที่
ต้องการความเสถียรของ production) ในเรื่องนี้มาอย่างยาวนาน และเป็นเหตุผลสำคัญที่หลายทีมเลือก Actix-web หรือ
Warp แทนในช่วงปี 2019-2021 เพราะทั้งสองรันบน stable Rust ได้ตั้งแต่แรก

**ข้อเท็จจริงปัจจุบัน (ที่ต้องแก้ความเข้าใจผิดที่ยังพบเห็นบ่อย)**: ตั้งแต่ **Rocket 0.5** (release อย่างเป็น
ทางการปลายปี 2023) Rocket **compile และรันได้บน stable Rust เต็มรูปแบบ** ไม่ต้องใช้ nightly toolchain
อีกต่อไป — ทีมงานปรับ codebase ทั้งหมดให้ใช้เฉพาะฟีเจอร์ที่ stabilize แล้ว การเปลี่ยนแปลงนี้เป็นเรื่องใหญ่
มากในประวัติศาสตร์ของ Rocket เพราะลบข้อจำกัดที่เคยเป็นเหตุผลหลักที่คนไม่เลือกใช้มันออกไปทั้งหมด บทนี้จะ
พิสูจน์ข้อเท็จจริงนี้ให้เห็นจริงในหัวข้อถัดไป — เราจะสร้างโปรเจกต์ Rocket ใหม่ **โดยไม่แก้ toolchain เป็น
nightly เลย** แล้ว compile รันได้จริงด้วย stable Rust ตัวเดียวกันที่ใช้เขียน Axum/Actix-web มาตลอดทั้ง
หลักสูตร

อย่างไรก็ตาม ยังมีบางฟีเจอร์ของ Rocket (ที่ระบุไว้ชัดใน documentation ของโปรเจกต์เอง) ที่**ยังต้องพึ่งพา
nightly** อยู่จนถึงปัจจุบัน เช่นฟีเจอร์ทดลองบางตัวที่ยังรอ Rust compiler stabilize API ที่เกี่ยวข้อง — แต่
นี่เป็นสถานะเดียวกับที่ crate อื่นๆจำนวนมากในระบบนิเวศ Rust เป็นอยู่ (ฟีเจอร์ทดลองผลักไปไว้หลัง nightly-only
flag) ไม่ใช่ว่า **การใช้งาน Rocket พื้นฐาน** ต้องพึ่งพา nightly แบบที่เคยเป็นในอดีต — Hello World, routing,
JSON, state management (ทุกอย่างที่บทนี้จะสอน) ทำงานได้ครบบน stable ทั้งหมด

### 69.3 ติดตั้ง Rocket และ Hello World ตัวจริง

มาพิสูจน์ข้อเท็จจริงในหัวข้อ 69.2 ด้วยการลงมือทำจริง — สร้างโปรเจกต์ใหม่ (บทนี้สร้างและทดสอบด้วย
`rustc`/`cargo` เวอร์ชัน stable ตัวเดียวกันที่ใช้ทั้งหลักสูตร ไม่มีการสับ toolchain):

```bash
cargo new hello_rocket
cd hello_rocket
cargo add rocket --features json
```

`Cargo.toml` ที่ได้:

```toml
[package]
name = "hello_rocket"
version = "0.1.0"
edition = "2021"

[dependencies]
rocket = { version = "0.5.1", features = ["json"] }
```

(เวอร์ชัน `0.5.1` คือเวอร์ชันล่าสุดบน crates.io ณ เวลาที่เขียนบทนี้ — ตรงกับที่ Part 62 อธิบายไว้เรื่อง
`cargo add` ดึงเวอร์ชันล่าสุดเสมอ)

โค้ด Hello World ที่เล็กที่สุดที่ยังทำงานได้จริง:

```rust
// เปิดใช้ attribute macro ของ Rocket ทั้งหมดผ่าน #[macro_use]
// (Rocket ยังใช้ pattern แบบ macro_use ของ Rust รุ่นเก่ากว่า use แบบ path-based ล้วนๆ
//  ด้วยเหตุผลด้าน internal macro hygiene — ไม่ใช่ปัญหา แค่เป็นสไตล์ที่ต่างจาก Axum/Actix-web เล็กน้อย)
#[macro_use]
extern crate rocket;

// #[get("/")] คือ attribute macro ที่ประกาศพร้อมกันทั้ง HTTP method (GET) และ path ("/")
// ในบรรทัดเดียว — ต่างจาก Axum ที่แยก .route("/", get(handler)) ออกจากตัว handler function
#[get("/")]
fn index() -> &'static str {
    "Hello, World!"
}

// #[launch] คือ attribute macro ที่ครอบฟังก์ชันซึ่งคืนค่า Rocket instance ที่ configure แล้ว
// มันแทนที่ #[tokio::main] + fn main() ของ Axum ไปในตัว — Rocket "ซ่อน" การสร้าง async runtime
// ไว้ข้างในให้ทั้งหมด ผู้ใช้ไม่ต้องเขียน #[tokio::main] เองเลย
#[launch]
fn rocket() -> _ {
    // rocket::build() สร้าง instance เปล่า, .mount(prefix, routes) ผูก route เข้ากับ prefix
    // routes![index] คือ macro ที่รวบรวม route function หลายตัว (ที่นี่มีแค่ index) เป็น list เดียว
    rocket::build().mount("/", routes![index])
}
```

สังเกตความแตกต่างจาก Axum ตั้งแต่บรรทัดแรก: Rocket ไม่มี `#[tokio::main]`, ไม่มี `async fn main()`, ไม่มี
`TcpListener::bind()` ที่ต้องเขียนเอง — ทั้งหมดถูกซ่อนไว้หลัง `#[launch]` ทั้งก้อน นี่คือตัวอย่างที่ชัดที่สุด
ของปรัชญา "batteries included" ที่หัวข้อ 69.2 อธิบายไว้: Axum ให้คุณเห็น (และควบคุม) `TcpListener` ตรงๆ
เพราะมันไม่อยากซ่อนอะไรที่ไม่จำเป็น ส่วน Rocket เลือกซ่อนรายละเอียดนี้ไปเพื่อให้ผู้ใช้โฟกัสที่ route logic
ล้วนๆ

รันเซิร์ฟเวอร์:

```bash
cargo run
```

ผลลัพธ์จริงจากการ compile และรัน (capture จริงจากการทดสอบบน stable Rust — **ไม่มีการเปลี่ยน toolchain เป็น
nightly เลยตลอดทั้งกระบวนการ**):

```
   Compiling hello_rocket v0.1.0 (/path/to/hello_rocket)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 1m 13s
     Running `target/debug/hello_rocket`
Configured for debug.
   >> address: 127.0.0.1
   >> port: 8000
   >> workers: 4
   >> max blocking threads: 512
   >> ident: Rocket
   >> IP header: X-Real-IP
   >> limits: bytes = 8KiB, data-form = 2MiB, file = 1MiB, form = 32KiB, json = 1MiB, msgpack = 1MiB, string = 8KiB
   >> temp dir: /tmp
   >> http/2: true
   >> keep-alive: 5s
   >> tls: disabled
   >> shutdown: ctrlc = true, force = true, signals = [SIGTERM], grace = 2s, mercy = 3s
   >> log level: normal
   >> cli colors: true
Routes:
   >> (index) GET /
Fairings:
   >> Shield (liftoff, response, singleton)
Shield:
   >> X-Content-Type-Options: nosniff
   >> X-Frame-Options: SAMEORIGIN
   >> Permissions-Policy: interest-cohort=()
Rocket has launched from http://127.0.0.1:8000
```

สังเกตสิ่งที่ Rocket **บอกคุณโดยไม่ต้องขอ** ตอน startup — นี่คือหลักฐานที่จับได้ชัดของปรัชญา
"batteries-included" อีกครั้ง:

- **`Routes:`** — พิมพ์ route table ทั้งหมดที่ mount ไว้ให้เห็นตรงๆตอน startup (Axum ไม่ทำแบบนี้ให้
  อัตโนมัติ — ต้องเขียน logging middleware เองถ้าต้องการเห็นสิ่งนี้)
- **`Fairings: >> Shield`** — **Fairing** คือชื่อเรียก middleware ของ Rocket (เทียบเท่ากับ `tower::Layer`
  ของ Axum หรือ `Transform`/middleware ของ Actix-web) และ **Shield** คือ fairing ด้าน security ที่ Rocket
  ติดมาให้ **โดยอัตโนมัติตั้งแต่ default** — ไม่ต้องเพิ่ม dependency ใดๆเพิ่ม ไม่ต้องเขียน middleware เอง
  แม้แต่บรรทัดเดียว เทียบกับ Axum ที่ security header แบบนี้ต้องเพิ่ม `tower-http` แล้วเขียน
  `SetResponseHeaderLayer` เองทุกตัว (Part 65 สอนวิธีเพิ่ม middleware เองแบบนี้ไว้เต็มรูปแบบ)

ทดสอบด้วย `curl` จริง (เปิด terminal ใหม่ เพราะ `cargo run` จะ block terminal เดิมไว้ตราบใดที่เซิร์ฟเวอร์ยัง
รันอยู่ — พฤติกรรมเดียวกับ `axum::serve(...).await` ที่ Part 62 อธิบายไว้):

```bash
curl -i http://127.0.0.1:8000/
```

output จริง:

```
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
server: Rocket
x-content-type-options: nosniff
x-frame-options: SAMEORIGIN
permissions-policy: interest-cohort=()
content-length: 13
date: Sat, 26 Sep 2026 23:58:23 GMT

Hello, World!
```

เทียบกับ output ของ Axum Hello World ใน Part 62 (`HTTP/1.1 200 OK`, `content-type: text/plain`,
`content-length: 13`, `date:` header) — เนื้อหาหลักเหมือนกันเป๊ะ (เพราะ handler คืนค่าเป็น string ธรรมดา
พอๆกัน) แต่ Rocket เพิ่ม header สาม header ที่ Axum ไม่มีให้อัตโนมัติ: `server: Rocket` (ระบุตัวเฟรมเวิร์ก
ตรงๆ), และสาม header จาก **Shield** fairing (`x-content-type-options`, `x-frame-options`,
`permissions-policy`) — นี่คือหลักฐานที่จับได้ตรงๆอีกครั้งของปรัชญา batteries-included: Rocket
**เพิ่ม defense-in-depth header ให้ทุก response โดยไม่ต้องขอ** ส่วน Axum **ไม่เพิ่มอะไรเลย**นอกจากสิ่งที่
จำเป็นต่อการทำงานของ HTTP (`content-type`, `content-length`, `date`)

ลองยิง path ที่ไม่มี route ผูกไว้:

```bash
curl -i http://127.0.0.1:8000/nope
```

output จริง:

```
HTTP/1.1 404 Not Found
content-type: text/html; charset=utf-8
server: Rocket
x-content-type-options: nosniff
x-frame-options: SAMEORIGIN
permissions-policy: interest-cohort=()
content-length: 435
date: Sat, 26 Sep 2026 23:58:23 GMT

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta name="color-scheme" content="light dark">
    <title>404 Not Found</title>
</head>
<body align="center">
    <div role="main" align="center">
        <h1>404: Not Found</h1>
        <p>The requested resource could not be found.</p>
        <hr />
    </div>
    <div role="contentinfo" align="center">
        <small>Rocket</small>
    </div>
</body>
</html>
```

ความแตกต่างที่สำคัญมากอีกจุด: Axum ตอบ 404 ด้วย body ว่างเปล่า (`content-length: 0`) ส่วน Rocket ตอบ 404
ด้วย **หน้า HTML ที่จัดรูปแบบไว้ให้แล้ว** โดยอัตโนมัติ — นี่คือ default catcher ของ Rocket (จะอธิบายเพิ่ม
ในหัวข้อ error handling) ที่แสดงให้เห็นว่า Rocket ตั้งใจออกแบบมาให้ "ใช้งานได้ดีทันทีที่ติดตั้ง" แม้แต่ error
page ที่ยังไม่ได้ customize เอง — ข้อดีคือ debug ง่ายกว่าเยอะตอน dev เพราะเห็นข้อความชัดในเบราว์เซอร์ ข้อเสีย
คือถ้าไม่ได้ปิด/เปลี่ยน default catcher เอง production API อาจส่ง HTML กลับไปให้ client ที่คาดหวัง JSON
ล้วนๆ (จะกล่าวถึงในหัวข้อกับดัก)

### 69.4 กายวิภาคของ Rocket: Request Guard คือ "Extractor" ของ Rocket

ก่อนเขียนตัวอย่างที่ซับซ้อนขึ้น ต้องเข้าใจแนวคิดหลักหนึ่งอย่างของ Rocket ที่เทียบเท่ากับ **extractor** ของ
Axum และ **extractor** ของ Actix-web (ที่ Part 64 และ 67 สอนไว้แล้วตามลำดับ) — Rocket เรียกแนวคิดนี้ว่า
**request guard** (บางเอกสารเรียกสั้นๆว่า "guard")

หลักการเดียวกันทั้งสามเฟรมเวิร์ก: **parameter ของ handler function บอกเองว่ามันต้องการข้อมูลอะไรจาก
request** โดยไม่ต้องเขียน parsing logic เอง — เฟรมเวิร์กจะ "ดึง" ค่าที่ตรงกับ type ที่ประกาศไว้มาให้ก่อน
เรียก handler จริง ถ้าดึงไม่สำเร็จ (เช่น parse type ผิด, header ที่ต้องการหาไม่เจอ) เฟรมเวิร์กจะตัดจบ
กระบวนการเองโดยไม่เข้า body ของ handler เลย

ความแตกต่างเชิง**คำศัพท์**และ**กลไกภายใน**ระหว่างสามเฟรมเวิร์ก:

| แนวคิด | Axum | Actix-web | Rocket |
|---|---|---|---|
| ชื่อเรียก | Extractor | Extractor | Request Guard |
| Trait ที่ต้อง implement | `FromRequest` / `FromRequestParts` | `FromRequest` | `FromRequest` (trait ของ Rocket เอง — คนละตัวกับของ Axum/Actix-web แม้ชื่อคล้าย) |
| วิธีประกาศใน handler | เป็น parameter ธรรมดาของ `async fn` | เป็น parameter ธรรมดาของ `async fn` | เป็น parameter ธรรมดาของ `fn`/`async fn` **แต่ชื่อ parameter ต้องตรงกับชื่อใน path/query attribute** |
| จุดที่ผูกกับ route | แยกจาก path string (`Path<u32>` แค่บอก "ฉันต้องการ path param") | แยกจาก path string เช่นกัน (`web::Path<u32>`) | **ผูกกับ path string โดยตรงผ่านชื่อตัวแปร** (`<id>` ใน path ต้องมี parameter ชื่อ `id`) |
| เมื่อดึงไม่สำเร็จ | เรียก `IntoResponse` ของ rejection type (ปกติ `400`) | ส่งกลับตาม error ที่เกิด (มักเป็น `404` เมื่อ route ไม่ match) | เรียก **catcher** ของ status code ที่เหมาะสม (ปกติ `422` สำหรับ type ไม่ตรง) |

จุดที่**ต่างจาก Axum/Actix-web มากที่สุด**คือแถวที่สี่: ใน Rocket ชื่อของ parameter ใน function signature
**ต้องตรงกับชื่อตัวแปรใน path/query attribute เป๊ะๆ** — นี่ไม่ใช่ความสะดวกเฉยๆ แต่เป็นกลไกที่ Rocket ใช้
**ตรวจสอบความถูกต้องตอน compile time** (จะพิสูจน์ให้เห็นจริงในหัวข้อ 69.6) เทียบกับ Axum ที่ `Path<u32>`
เป็นแค่ "บอกว่าต้องการ path param ตัวหนึ่ง" โดยไม่ผูกชื่อกับ path string เลย (การจับคู่ path parameter หลาย
ตัวกับ tuple ใน Axum ใช้ "ลำดับ" ไม่ใช่ "ชื่อ")

### 69.5 สร้าง Tasks API เวอร์ชัน Rocket: Path Guard, Query Guard, JSON, Managed State

มาสร้างตัวอย่างเดียวกันกับที่ Part 62 (Axum) และ Part 67-68 (Actix-web) ใช้ตลอด — ระบบ **Tasks API** ขนาด
เล็กที่มี endpoint สำหรับดูรายการงาน, ดูงานตัวเดียวตาม ID, และกรองงานตามสถานะ `done` — คราวนี้เขียนด้วย
Rocket โดยตั้งใจใช้ domain เดียวกันเพื่อให้เทียบโค้ดกันตรงๆได้ในหัวข้อ 69.7

ติดตั้ง dependency (feature `json` จำเป็นเพื่อเปิดใช้ `rocket::serde::json::Json<T>`):

```bash
cargo add rocket --features json
```

โค้ดเต็ม:

```rust
#[macro_use]
extern crate rocket;

use rocket::serde::json::Json;
use rocket::serde::Serialize;
use rocket::State;
use std::sync::Mutex;

// Task struct: ต้อง derive Serialize เหมือนกับที่ Axum/Actix-web ต้องการ (serde ตัวเดียวกันเป๊ะ)
// #[serde(crate = "rocket::serde")] จำเป็นเพราะ Rocket re-export serde ของตัวเองผ่าน rocket::serde
// (เพื่อรับประกันว่า minor version ของ serde ที่ Rocket ใช้ภายในตรงกับที่ผู้ใช้ derive อยู่เสมอ)
#[derive(Debug, Clone, Serialize)]
#[serde(crate = "rocket::serde")]
struct Task {
    id: u32,
    title: String,
    done: bool,
}

// AppState: เหมือนกับ Part 62 เป๊ะ — Vec<Task> อยู่หลัง Mutex เพราะหลาย request อาจวิ่งพร้อมกัน
struct AppState {
    tasks: Mutex<Vec<Task>>,
}

#[get("/health")]
fn health() -> &'static str {
    "ok"
}

// &State<AppState> คือ request guard ที่ Rocket ให้มาสำหรับดึงค่าที่ผูกไว้ผ่าน .manage(...)
// เทียบเท่า State<Arc<AppState>> ของ Axum และ web::Data<AppState> ของ Actix-web
// สังเกตว่าไม่ต้องห่อ AppState ด้วย Arc เอง — Rocket จัดการ sharing ให้ภายใน State<T> เอง
#[get("/tasks/<id>")]
fn get_task(state: &State<AppState>, id: u32) -> Result<Json<Task>, rocket::http::Status> {
    let tasks = state.tasks.lock().unwrap();
    match tasks.iter().find(|t| t.id == id) {
        Some(task) => Ok(Json(task.clone())),
        // rocket::http::Status implement Responder เอง — คืน status code เปล่าไม่มี body
        None => Err(rocket::http::Status::NotFound),
    }
}

// query parameter ประกาศด้วย ?<name> ใน path attribute — Option<bool> ทำให้ query param
// เป็น "ไม่บังคับ" (ไม่ส่ง query มาก็ได้ ค่าจะเป็น None) เทียบกับ Query<T> ของ Axum ที่ต้องห่อ
// struct เองแล้ว derive Deserialize
#[get("/tasks?<done>")]
fn filter_tasks(state: &State<AppState>, done: Option<bool>) -> Json<Vec<Task>> {
    let tasks = state.tasks.lock().unwrap();
    match done {
        Some(d) => Json(tasks.iter().filter(|t| t.done == d).cloned().collect()),
        None => Json(tasks.clone()),
    }
}

#[launch]
fn rocket() -> _ {
    let state = AppState {
        tasks: Mutex::new(vec![
            Task { id: 1, title: "เขียนบท Rocket".to_string(), done: false },
            Task { id: 2, title: "ทดสอบ curl".to_string(), done: true },
        ]),
    };

    rocket::build()
        // .manage(state) ผูก state เข้ากับ instance — เทียบเท่า .with_state() ของ Axum
        // และ .app_data() ของ Actix-web
        .manage(state)
        .mount("/", routes![health, get_task, filter_tasks])
}
```

`cargo build` ผ่านสำเร็จโดยไม่มี warning ที่เกี่ยวข้อง แล้วรันด้วย `cargo run` ได้ route table แบบนี้จริง:

```
Routes:
   >> (filter_tasks) GET /tasks?<done>
   >> (health) GET /health
   >> (get_task) GET /tasks/<id>
```

ทดสอบทุก endpoint ด้วย `curl` จริง:

```bash
curl -i http://127.0.0.1:8000/tasks
```

output จริง (ไม่ส่ง query `done` มา — ตรงกับ branch `None` ในโค้ด):

```
HTTP/1.1 200 OK
content-type: application/json
server: Rocket
x-content-type-options: nosniff
x-frame-options: SAMEORIGIN
permissions-policy: interest-cohort=()
content-length: 114
date: Sat, 26 Sep 2026 23:59:25 GMT

[{"id":1,"title":"เขียนบท Rocket","done":false},{"id":2,"title":"ทดสอบ curl","done":true}]
```

```bash
curl -i "http://127.0.0.1:8000/tasks?done=true"
```

output จริง (กรองเฉพาะ `done == true`):

```
HTTP/1.1 200 OK
content-type: application/json
server: Rocket
x-content-type-options: nosniff
x-frame-options: SAMEORIGIN
permissions-policy: interest-cohort=()
content-length: 53
date: Sat, 26 Sep 2026 23:59:25 GMT

[{"id":2,"title":"ทดสอบ curl","done":true}]
```

```bash
curl -i http://127.0.0.1:8000/tasks/1
```

output จริง:

```
HTTP/1.1 200 OK
content-type: application/json
server: Rocket
x-content-type-options: nosniff
x-frame-options: SAMEORIGIN
permissions-policy: interest-cohort=()
content-length: 60
date: Sat, 26 Sep 2026 23:59:25 GMT

{"id":1,"title":"เขียนบท Rocket","done":false}
```

```bash
curl -i http://127.0.0.1:8000/tasks/999
```

output จริง (ไม่มี task id 999 — เข้า branch `None` แล้วคืน `Status::NotFound`):

```
HTTP/1.1 404 Not Found
content-type: text/html; charset=utf-8
server: Rocket
x-content-type-options: nosniff
x-frame-options: SAMEORIGIN
permissions-policy: interest-cohort=()
content-length: 435
date: Sat, 26 Sep 2026 23:59:25 GMT

<!DOCTYPE html>
...(หน้า HTML default catcher เดียวกับหัวข้อ 69.3)...
```

ที่น่าสนใจที่สุดคือกรณีสุดท้าย:

```bash
curl -i http://127.0.0.1:8000/tasks/abc
```

output จริง (`abc` parse เป็น `u32` ไม่ได้ — request guard ล้มเหลว **ก่อน**เข้า body ของ `get_task` เลย):

```
HTTP/1.1 422 Unprocessable Entity
content-type: text/html; charset=utf-8
server: Rocket
x-content-type-options: nosniff
x-frame-options: SAMEORIGIN
permissions-policy: interest-cohort=()
content-length: 496
date: Sat, 26 Sep 2026 23:59:25 GMT

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <meta name="color-scheme" content="light dark">
    <title>422 Unprocessable Entity</title>
</head>
<body align="center">
    <div role="main" align="center">
        <h1>422: Unprocessable Entity</h1>
        <p>The request was well-formed but was unable to be followed due to semantic errors.</p>
        <hr />
    </div>
    <div role="contentinfo" align="center">
        <small>Rocket</small>
    </div>
</body>
</html>
```

เก็บผลลัพธ์นี้ไว้ในใจ — หัวข้อ 69.7 จะเทียบผลลัพธ์นี้กับ Axum และ Actix-web ที่ทดสอบสถานการณ์เดียวกัน
(`path param parse ไม่ผ่าน`) แล้วจะเห็นว่า**ทั้งสามเฟรมเวิร์กตอบ status code ต่างกันโดยสิ้นเชิง** ทั้งที่โค้ด
ทำสิ่งเดียวกัน — นี่คือตัวอย่างที่จับต้องได้ที่สุดของ "ปรัชญาการออกแบบที่ต่างกันนำไปสู่พฤติกรรม default ที่
ต่างกัน"

### 69.6 Compile-Time Route Checking: จุดขายที่ยังเป็นจริงของ Rocket

หัวข้อ 69.4 กล่าวไว้ว่าชื่อ parameter ของ Rocket ต้องตรงกับชื่อตัวแปรใน path attribute — นี่ไม่ใช่ข้อจำกัด
ที่สร้างความรำคาญเฉยๆ แต่คือกลไกที่ทำให้ Rocket **ตรวจจับ route ที่เขียนผิดได้ตั้งแต่ compile time** ซึ่งเป็น
จุดขายที่ Rocket เน้นย้ำมาตั้งแต่เปิดตัวโปรเจกต์ และ**ยังเป็นจริงอยู่จนถึง Rocket 0.5.1 ปัจจุบัน**

มาพิสูจน์ให้เห็นจริง — ลองเขียนโค้ดที่ path attribute ประกาศ `<id>` แต่ function parameter ใช้ชื่อ `user_id`
(เผลอพิมพ์ผิด หรือ refactor ชื่อตัวแปรแล้วลืมแก้ path):

```rust
#[macro_use]
extern crate rocket;

// จงใจเขียนผิด: path ประกาศ <id> แต่ function parameter ชื่อ user_id
#[get("/tasks/<id>")]
fn get_task(user_id: u32) -> String {
    format!("{}", user_id)
}

#[launch]
fn rocket() -> _ {
    rocket::build().mount("/", routes![get_task])
}
```

`cargo build` ให้ error จริงแบบนี้ (capture มาจริง ไม่ได้แต่งขึ้น):

```
error: unused parameter
 --> src/main.rs:5:7
  |
5 | #[get("/tasks/<id>")]
  |       ^^^^^^^^^^^^^

error: [note] expected argument named `id` here
 --> src/main.rs:6:12
  |
6 | fn get_task(user_id: u32) -> String {
  |            ^^^^^^^^^^^^^^

error[E0422]: cannot find struct, variant or union type `get_task` in this scope
  --> src/main.rs:12:40
   |
12 |     rocket::build().mount("/", routes![get_task])
   |                                        ^^^^^^^^ not found in this scope
```

**นี่คือหลักฐานที่จับได้ตรงๆ**ว่า attribute macro `#[get(...)]` ของ Rocket ไม่ได้แค่ "สร้าง route registration
ให้อัตโนมัติ" — มันตรวจสอบ**ความสอดคล้อง**ระหว่าง path string กับ function signature ตั้งแต่ตอน compile
error message บอกตรงๆว่า "unused parameter" (คือ `<id>` ใน path ไม่มีใครมารับ) พร้อม `note` ที่บอกว่า
"expected argument named `id` here" ชี้ไปที่ signature ของฟังก์ชันเป๊ะๆ — ผู้เขียนโค้ดรู้ทันทีตอน `cargo
build` ว่ามีปัญหา ไม่ต้องรอให้ deploy ไปแล้วเจอ runtime error ตอนมีคน request เข้ามาจริง

เทียบกับ Axum: ถ้าคุณเขียน `.route("/tasks/{id}", get(handler))` แล้ว `handler` มี `Path<u32>` parameter
ที่ไม่ตรงกับชื่อ (เช่น เผลอเขียน `.route("/tasks/{user_id}", ...)`) **Axum จะไม่ error ตอน compile เลย**
เพราะ `Path<u32>` ของ Axum ไม่ผูกกับชื่อ path parameter — มันรับค่า path parameter ตัวแรกที่เจอ (หรือถ้ามี
หลายตัว ต้องใช้ tuple ตามลำดับ) ไม่ว่า path segment นั้นชื่ออะไรก็ตาม โปรแกรมจะ compile ผ่านและทำงานได้
ถูกต้องเพราะ Axum แค่สนใจ**ตำแหน่ง**ของ path parameter ไม่สนใจชื่อ — นี่หมายความว่าถ้าคุณเขียน route ผิด
ลักษณะอื่น (เช่น สลับลำดับ path parameter หลายตัวใน `Path<(u32, String)>`) **compiler จะไม่จับให้** ต้องรอ
เจอ runtime behavior ที่ผิดคาดแทน — นี่คือความแตกต่างเชิง "safety net" ที่แท้จริงที่ Rocket ยังคงรักษาไว้ได้
ดีกว่า และเป็นเหตุผลที่คนจำนวนไม่น้อยยังเลือก Rocket สำหรับโปรเจกต์ที่ route จำนวนมากและอยากได้ safety net
เพิ่มจาก compiler

ต้องพูดให้ชัดว่านี่**ไม่ใช่การรับประกันความถูกต้อง 100%** ของทั้งระบบ — compile-time route checking ของ
Rocket ตรวจแค่ความสอดคล้องระหว่าง path/query string กับชื่อ-ประเภทของ parameter ในฟังก์ชันเท่านั้น มันไม่
ตรวจ business logic ผิด, ไม่ตรวจว่า route ไป overlap กับ route อื่นอย่างที่ไม่ตั้งใจ (ปัญหานี้ยังเป็น
runtime behavior เหมือนกันทั้งสามเฟรมเวิร์ก), และไม่ตรวจว่า handler จะ panic หรือไม่ — มันคือ safety net
ที่แคบแต่มีประโยชน์จริง ไม่ใช่ "ระบบ type-safe routing แบบสมบูรณ์" ตามที่บางบทความทางการตลาดอาจพูดเกินจริง

### 69.7 Side-by-Side: Endpoint เดียวกันในสามเฟรมเวิร์ก

ถึงหัวข้อที่มีค่าที่สุดของบทนี้ — วางโค้ดของ endpoint เดียวกันเป๊ะ (`GET /tasks/<id>` → คืน `Task` เป็น JSON
ถ้าเจอ, คืน 404-ish ถ้าไม่เจอ) เขียนด้วยสามเฟรมเวิร์กเคียงข้างกัน โค้ดทั้งสามชุดนี้ compile ผ่านจริงและทดสอบ
ด้วย `curl` จริงมาแล้วทั้งหมด (Axum: ยืนยันจาก Part 62 และทดสอบซ้ำสำหรับบทนี้, Actix-web: ทดสอบจริงสำหรับ
บทนี้, Rocket: ทดสอบจริงในหัวข้อ 69.5)

**Axum:**

```rust
use axum::extract::{Path, State};
use axum::http::StatusCode;
use axum::routing::get;
use axum::{Json, Router};
use std::sync::{Arc, Mutex};

async fn get_task(
    State(state): State<Arc<AppState>>,
    Path(id): Path<u32>,
) -> Result<Json<Task>, (StatusCode, String)> {
    let tasks = state.tasks.lock().unwrap();
    match tasks.iter().find(|t| t.id == id) {
        Some(task) => Ok(Json(task.clone())),
        None => Err((StatusCode::NOT_FOUND, format!("ไม่พบ task id {id}"))),
    }
}

// การลงทะเบียน route แยกจากตัว handler โดยสิ้นเชิง
let app = Router::new()
    .route("/tasks/{id}", get(get_task))
    .with_state(state);
```

**Actix-web:**

```rust
use actix_web::{get, web, HttpResponse, Responder};

#[get("/tasks/{id}")]
async fn get_task(state: web::Data<AppState>, path: web::Path<u32>) -> impl Responder {
    let id = path.into_inner();
    let tasks = state.tasks.lock().unwrap();
    match tasks.iter().find(|t| t.id == id) {
        Some(task) => HttpResponse::Ok().json(task.clone()),
        None => HttpResponse::NotFound().body(format!("ไม่พบ task id {id}")),
    }
}

// #[get("/tasks/{id}")] ผูก path ไว้กับ handler โดยตรง ไม่ต้องแยกลงทะเบียนอีกที
// (แค่ .service(get_task) ตอนสร้าง App)
```

**Rocket:**

```rust
use rocket::serde::json::Json;
use rocket::State;

#[get("/tasks/<id>")]
fn get_task(state: &State<AppState>, id: u32) -> Result<Json<Task>, rocket::http::Status> {
    let tasks = state.tasks.lock().unwrap();
    match tasks.iter().find(|t| t.id == id) {
        Some(task) => Ok(Json(task.clone())),
        None => Err(rocket::http::Status::NotFound),
    }
}

// #[get("/tasks/<id>")] ผูก path ไว้กับ handler ตรงๆเหมือน Actix-web
// แต่ id ต้องตรงชื่อกับ <id> ใน path เป๊ะ (หัวข้อ 69.6)
```

**สิ่งที่เหมือนกันทั้งสามชุด** (คุ้มค่าที่จะสังเกตก่อนพูดถึงความต่าง): logic ภายในเกือบเหมือนกันทุกตัวอักษร
— `.lock().unwrap()`, `.iter().find(...)`, `match Some/None` — เพราะ**นี่คือ business logic ธรรมดาของ
Rust ที่ไม่เกี่ยวกับเฟรมเวิร์กเลย** (Part 39 สอน `Mutex`/`Arc`, ไม่ใช่เนื้อหาเว็บเฟรมเวิร์ก) ความต่างทั้งหมด
อยู่ที่ **"เปลือก" รอบนอก** — วิธีประกาศ route, วิธีดึง state, วิธีคืน error — นี่คือข้อสังเกตสำคัญที่จะ
กลับมาขยายในหัวข้อ 69.14 เรื่อง migration

**สิ่งที่ต่างกันเชิง idiom**:

| จุดเทียบ | Axum | Actix-web | Rocket |
|---|---|---|---|
| การประกาศ route กับ handler | แยกกัน (`.route()` แยกจาก `fn`) | รวมกันผ่าน attribute (`#[get(...)]` เหนือ `fn` เลย) | รวมกันผ่าน attribute เหมือน Actix-web |
| การดึง path parameter | `Path<u32>` — ไม่ผูกชื่อกับ path string | `web::Path<u32>` — ไม่ผูกชื่อกับ path string | `id: u32` ตรงๆ — **ต้อง**ผูกชื่อกับ `<id>` ใน path |
| การดึง state | `State<Arc<AppState>>` — ต้องห่อด้วย `Arc` เอง | `web::Data<AppState>` — ห่อด้วย `Arc` ให้ภายใน | `&State<AppState>` — ห่อ sharing ให้ภายในเช่นกัน |
| การคืน error พร้อม status | `(StatusCode, String)` implement `IntoResponse` เอง | `HttpResponse::NotFound().body(...)` สร้าง response ตรงๆ | `rocket::http::Status` implement `Responder` (คืน status ล้วนไม่มี body ในตัวอย่างนี้) |
| จำนวนบรรทัด "เปลือก" (ไม่รวม logic) | มากที่สุด (ต้องเขียน `Router`, `.route()`, `.with_state()` แยก) | กลางๆ | น้อยที่สุด (attribute ทำงานหลายอย่างในบรรทัดเดียว) |

**พฤติกรรมจริงที่ต่างกันเมื่อ path parameter parse ไม่ผ่าน** (ทดสอบจริงด้วย `curl
http://.../tasks/abc` กับทั้งสามเซิร์ฟเวอร์):

| เฟรมเวิร์ก | Status Code ที่ตอบจริง | เหตุผลเชิงกลไก |
|---|---|---|
| Axum | `400 Bad Request` | `Path<u32>` rejection มี `IntoResponse` default เป็น `400` — extractor ล้มเหลวถูกมองเป็น "client ส่ง request ผิดรูปแบบ" |
| Actix-web | `404 Not Found` | `web::Path<u32>` guard ล้มเหลว → route ไม่ถูกมองว่า "match" เลย → ตกไปที่ default handler ของ "ไม่พบ route" ซึ่งคือ 404 |
| Rocket | `422 Unprocessable Entity` | request guard ล้มเหลวเพราะ type ไม่ตรง (path *รูปแบบ* ถูกแล้ว มี segment ตรงตำแหน่ง แต่ *เนื้อหา* parse เป็น `u32` ไม่ได้) → Rocket จัดว่าเป็น semantic error ไม่ใช่ syntax error ของ request |

นี่คือตัวอย่างที่จับต้องได้ที่สุดในบทนี้ว่า **"ปรัชญาการออกแบบที่ต่างกันจริงๆนำไปสู่พฤติกรรม default ที่ต่าง
กันจริงๆ"** ไม่ใช่แค่คำพูดลอยๆ — logic ของแอปพลิเคชันเหมือนกัน 100% (ทั้งสามคืน "ไม่พบ" เมื่อ id ไม่ตรง
รูปแบบ) แต่ status code ที่ client ได้รับต่างกันถึงสามแบบ ถ้าทีมของคุณมี API contract ที่ระบุ status code
ตายตัวสำหรับกรณีนี้ (เช่น ต้องเป็น 400 เท่านั้นตามสเปกที่ตกลงกับทีม frontend) **นี่คือรายละเอียดที่ต้อง
ทดสอบเอง**ไม่ใช่แค่เชื่อว่าเฟรมเวิร์กไหนก็ตอบเหมือนกัน — บทเรียนเชิงปฏิบัตินี้สำคัญกว่าที่มองแวบแรก เพราะ
มันคือสิ่งที่ integration test (Part 63 สอนพื้นฐานการเทสต์ Axum ไว้) ควรครอบคลุมเสมอเมื่อเปลี่ยนหรือเลือก
เฟรมเวิร์กใหม่

### 69.8 Architecture Comparison: เจาะลึกทุกมิติ

ตารางด้านล่างสรุปความต่างเชิงสถาปัตยกรรมของทั้งสามเฟรมเวิร์กในทุกมิติสำคัญ — อ่านทีละแถวช้าๆ เพราะแต่ละ
แถวมีเหตุผลเชิงลึกรออธิบายต่อด้านล่าง ไม่ใช่แค่ "ข้อมูล feature" ให้ท่องจำ

| มิติ | Axum | Actix-web | Rocket |
|---|---|---|---|
| **Routing style** | Method chaining บน `Router` (`.route(path, method(handler))`), แยกจาก handler function | Attribute macro เหนือ handler (`#[get(path)]`) ผูกกับ `.service()` | Attribute macro เหนือ handler (`#[get(path)]`) ผูกกับ `routes![...]` |
| **Extractor/Guard model** | `FromRequest`/`FromRequestParts`, ไม่ผูกชื่อกับ path string, static dispatch เต็มรูปแบบ | `FromRequest` (trait ของ Actix-web เอง), ไม่ผูกชื่อกับ path string | `FromRequest` (trait ของ Rocket เอง — คนละ trait), **ผูกชื่อกับ path/query string บังคับ** |
| **Middleware system** | `tower::Layer`/`tower::Service` — นามธรรมกลางที่ใช้ร่วมกับ ecosystem อื่นได้ (gRPC, RPC) | `Transform`/`Service` ของ Actix-web เอง — ออกแบบเฉพาะสำหรับ Actix-web ไม่ใช้ข้ามระบบนิเวศ | **Fairing** — ระบบ hook ของตัวเอง (on-ignite, on-request, on-response, on-liftoff) |
| **State management** | `State<T>` extractor + `.with_state()`, ต้องห่อ `Arc` เอง | `web::Data<T>` extractor + `.app_data()`, ห่อ `Arc` ให้ภายใน | `State<T>` guard + `.manage()`, ห่อ sharing ให้ภายใน |
| **Error handling ergonomics** | `IntoResponse` บน error type เอง — ยืดหยุ่นเต็มที่ (Part 66 สอนเต็มรูปแบบ) แต่ต้อง implement เอง | `ResponseError` trait — ค่อนข้าง verbose แต่ mapping ชัดเจน | **Catcher** (`#[catch(code)]`) ผูกกับ status code ตรงๆ — ปรับ error page/response ทั้งแอปในจุดเดียว |
| **Compile-time route checking** | ไม่มี — path parameter ผูกด้วยตำแหน่ง ไม่ตรวจชื่อ | ไม่มี — เหมือน Axum | **มี** — ตรวจความสอดคล้องชื่อ/ประเภทระหว่าง path/query string กับ parameter (หัวข้อ 69.6) |
| **Community/ecosystem maturity** | ใหญ่และเติบโตเร็วที่สุดในปัจจุบัน (ทีม Tokio ดูแล, ผูกกับ `tower` ecosystem ที่ crate จำนวนมากรองรับ) | ใหญ่และเก่าแก่ที่สุดในสามตัว (production track record ยาวนานที่สุดในกลุ่มนี้) | เล็กกว่าอีกสองตัวชัดเจน แต่ยังมีการพัฒนาต่อเนื่องและ community เฉพาะกลุ่มที่แน่น |
| **การพึ่งพา `tower`/`hyper`** | **ใช้ `hyper` เป็น HTTP engine + `tower` เป็นนามธรรม middleware กลางตรงๆ** (ยืนยันจาก `cargo tree`/build log: `hyper v1.11.1`, `tower v0.5.3` เป็น dependency ตรง) | **ไม่ใช้ `tower` เลย** — มี `actix-http`/`actix-service` เป็นของตัวเอง (ยืนยันจาก build log: ไม่มี `tower` compile เข้ามาเลยตอน `cargo build`) | **ไม่ใช้ `tower`** — มี `rocket_http` เป็นของตัวเอง และภายในยังใช้ `hyper` เวอร์ชันเก่ากว่า (`hyper v0.14.32` ที่พบใน build log) เพื่อ TCP/HTTP transport เท่านั้น ไม่ใช่แบบ `tower::Service` |

**อธิบายเชิงลึกแถวสำคัญที่สุด — การพึ่งพา `tower`/`hyper`**: ระหว่างเขียนบทนี้ ได้ตรวจสอบ dependency graph
จริงของทั้งสามเฟรมเวิร์กผ่าน `cargo build` (สังเกตชื่อ crate ที่ compiler ดึงมา compile จริง ไม่ใช่แค่อ่าน
เอกสาร) — ผลที่ได้ตรงกับที่ Part 62 อธิบายไว้เรื่อง Axum ทุกประการ (`hyper v1.11.1`, `tower v0.5.3` เป็น
dependency ตรงของ `axum`) ในขณะที่ตอน build โปรเจกต์ Actix-web และ Rocket **ไม่มี crate ชื่อ `tower` ถูก
compile เข้ามาแม้แต่ครั้งเดียว** — นี่คือหลักฐานที่จับได้ตรงๆว่า Actix-web และ Rocket **ไม่ได้ใช้ `tower`
เป็นนามธรรมกลางเลย** ทั้งสองมีระบบ `Service`/middleware ของตัวเองที่ออกแบบมาเฉพาะสำหรับตัวเองเท่านั้น

ผลที่ตามมาในทางปฏิบัติสำคัญมาก: **middleware ที่เขียนขึ้นสำหรับ `tower-http`** (เช่น `CorsLayer`,
`TraceLayer`, `TimeoutLayer` ที่ Part 65 สอนใช้กับ Axum) **ใช้กับ Actix-web หรือ Rocket ไม่ได้เลยโดยตรง**
เพราะมันคาดหวัง trait `tower::Layer`/`tower::Service` ซึ่งทั้งสองเฟรมเวิร์กไม่มี ต้องเขียน middleware แบบ
`Transform` ของ Actix-web หรือ `Fairing` ของ Rocket ขึ้นมาใหม่เอง (ต่อให้ logic ภายในเหมือนกัน — เช่น
"ล็อก request/response" — แต่ "เปลือก" ที่ต้อง implement ต่างกันโดยสิ้นเชิง) — จุดนี้คือหนึ่งในเหตุผลเชิง
ปฏิบัติที่สำคัญที่สุดที่จะกล่าวถึงต่อในหัวข้อ 69.12 ว่าทำไมหลักสูตรนี้เลือกเดินหน้าด้วย Axum

**อธิบายเชิงลึก — Middleware system ที่ต่างกันสามระบบ**: แม้ทั้งสามเฟรมเวิร์กจะมีแนวคิด "middleware"
เหมือนกัน (ทำงานบางอย่างก่อน/หลัง handler จริงรัน เช่น logging, auth check, compression) แต่**นามธรรม
ที่ใช้ implement มันต่างกันโดยสิ้นเชิง**:

- **Axum/`tower`**: middleware คือ struct ที่ implement `tower::Service<Request>` ห่อ (wrap) `Service`
  อีกตัวไว้ข้างใน — เป็นรูปแบบ "onion" ที่ composable สูงมาก (ห่อกันได้หลายชั้นไม่จำกัด) และที่สำคัญคือ
  นามธรรมนี้**ไม่ผูกกับ HTTP เลย** — `tower::Service` เอาไปใช้กับ gRPC หรือ protocol อื่นก็ได้ (Part 65
  อธิบายพื้นฐานนี้ไว้)
- **Actix-web**: middleware implement trait `Transform` แล้วคืน `Service` ที่ Actix-web กำหนดไว้เอง — 
  แนวคิดคล้าย `tower` (ห่อ service ซ้อนกัน) แต่เป็น trait ที่ผูกกับ Actix-web โดยเฉพาะ ออกแบบมาให้ทำงาน
  ร่วมกับ actor-based runtime เดิมของโปรเจกต์
- **Rocket/Fairing**: ต่างจากอีกสองแบบชัดเจนที่สุด — Fairing ไม่ใช่ "ห่อ Service ซ้อนกัน" แต่เป็น
  **callback ที่ hook เข้ากับจุดเฉพาะ**ของ lifecycle (`on_ignite` ตอน build เซิร์ฟเวอร์, `on_request` ก่อน
  routing, `on_response` หลัง handler รันจบ, `on_liftoff` ตอนเซิร์ฟเวอร์เริ่มรับ connection จริง) — ง่ายกว่า
  ต่อการเขียนสำหรับงานพื้นฐาน (logging, header injection แบบที่ Shield ทำ) แต่ไม่ composable ในระดับเดียว
  กับ `tower::Layer` เพราะไม่ได้ออกแบบให้ "ห่อ" ตัวเองซ้อนกันหลายชั้นแบบ Service pattern

**อธิบายเชิงลึก — Error handling ergonomics**: Rocket มีจุดเด่นเฉพาะตัวคือ **catcher**
(`#[catch(404)] fn not_found() -> ... { ... }`) ที่ผูก error page/response เข้ากับ **status code**
โดยตรงในจุดเดียวทั้งแอป ต่างจาก Axum ที่ error handling กระจายอยู่ที่ทุก handler (ผ่าน `Result<T, E>` ที่
`E` implement `IntoResponse` เอง — Part 66) และ Actix-web ที่ใช้ `ResponseError` trait บน error type ของ
ตัวเอง — ทั้งสามวิธีถูกต้องและใช้งานได้จริงหมด แต่ catcher ของ Rocket เหมาะกับกรณีที่ต้องการ "หน้า error ที่
สม่ำเสมอทั้งแอปตาม status code" (เช่น 404 หน้าเดียวใช้ทั้งระบบ) ในขณะที่ `IntoResponse`/`ResponseError`
เหมาะกับกรณีที่ error message ต้องมีรายละเอียดเฉพาะของแต่ละ domain error (เช่น "ไม่พบ task" ต่างจาก "ไม่พบ
user" ในเนื้อหา JSON ที่ตอบกลับ)

### 69.9 Worker/Task Model: มรดกจาก Actor System ของ Actix-web เทียบกับ Task-per-Connection ของ Tokio

หัวข้อ 69.8 พูดถึงความต่างเรื่อง `tower`/`hyper` ไปแล้ว แต่มีมิติสถาปัตยกรรมอีกชั้นหนึ่งที่ลึกกว่านั้นและ
ส่งผลกับพฤติกรรมของ state ในหน่วยความจำโดยตรง — **โมเดล concurrency ระดับ process/thread ที่แต่ละเฟรมเวิร์ก
เลือกใช้**

**Axum และ Rocket: task-per-connection บน Tokio runtime เดียวกัน** — ทั้งสองเฟรมเวิร์กไม่มีแนวคิดเรื่อง
"worker" ของตัวเองเลย พวกมันฝากทุกอย่างไว้กับ Tokio scheduler ตรงๆ (Part 48 สอนไว้เต็มรูปแบบ) — ทุก
connection ที่เข้ามาจะถูก `tokio::spawn` เป็น task ใหม่หนึ่งตัว ซึ่ง Tokio's work-stealing scheduler จะเลือก
เอาไปรันบน OS thread ไหนก็ได้ในกลุ่ม worker thread ของ runtime (ปกติเท่ากับจำนวน logical CPU core — ตรงกับ
ที่ Part 48 อธิบายเรื่อง `rt-multi-thread` ไว้) — task สามารถถูกย้ายข้าม thread ได้ระหว่างที่ยัง await อยู่
(เพราะ future เป็นแค่ state machine ที่ Send ข้าม thread ได้ ตามที่ Part 46 สอนไว้) โมเดลนี้ให้ load
balancing ที่ดีมากโดยอัตโนมัติ เพราะ scheduler จัดสรร task ให้ thread ที่ว่างที่สุดเสมอ ไม่ต้องคิดเรื่องการ
กระจายงานเอง

**Actix-web: มรดกจาก actor-model เดิม + worker process ที่ตรึงกับ thread** — ชื่อ "Actix" มาจาก **actor
model** ตรงๆ (โปรเจกต์เริ่มต้นจาก crate ชื่อ `actix` ที่ implement ระบบ actor แบบ Erlang/Akka สำหรับ Rust —
มีแนวคิดเรื่อง `Actor`, `Addr` (ที่อยู่สำหรับส่งข้อความหา actor), `System`, และ `Arbiter` ซึ่ง Actix-web
เวอร์ชันแรกๆสร้างอยู่บนระบบนี้ทั้งหมด) — **Actix-web เวอร์ชันปัจจุบัน (4.x) ไม่บังคับให้เขียน actor
เองแล้ว** คุณเขียน `async fn handler` ธรรมดาได้เหมือน Axum/Rocket ทุกประการ (ตัวอย่างทั้งหมดในบทนี้และ Part
67-68 ไม่มี actor ปรากฏเลย) — แต่ **โมเดล worker ที่สืบทอดมาจากยุค actor ยังคงอยู่** และเป็นความต่างเชิง
สถาปัตยกรรมที่สำคัญที่สุดกับ Tokio task-per-connection:

`HttpServer::new(factory)` ของ Actix-web เมื่อเรียก `.run()` จะสร้าง **worker thread จำนวนคงที่** (ค่า
default คือจำนวน logical CPU core ของเครื่อง ปรับได้ผ่าน `.workers(n)`) — จุดสำคัญคือ **factory closure
(`move || { App::new()... }`) ถูกเรียกแยกกันหนึ่งครั้งต่อหนึ่ง worker thread** ไม่ใช่เรียกครั้งเดียวแล้วแชร์
ทุก thread แบบที่คนอาจคาดหวัง — ผลคือ `App` instance ที่มี routing table ถูกสร้างขึ้นใหม่ **N ชุด** (N คือ
จำนวน worker) แต่ละ worker thread ผูก (pin) กับ connection ที่ตัวเองรับผิดชอบไปตลอดชีวิตของ connection นั้น
ไม่ย้ายข้าม thread แบบ Tokio task — นี่คือเหตุผลที่ `web::Data<T>` (ที่ห่อ `Arc` ไว้ภายในให้อัตโนมัติ ตามที่
หัวข้อ 69.5-69.7 กล่าวถึง) ต้องถูก **`.clone()` ก่อนส่งเข้า closure** เสมอ (สังเกตบรรทัด `let state =
web::Data::new(...)` แล้ว `move || { App::new().app_data(state.clone())... }` ในตัวอย่างหัวข้อ 69.7) —
เพราะ closure นั้นถูกเรียกซ้ำ N รอบ (หนึ่งรอบต่อ worker) แต่ละรอบต้องมี `Arc` ที่ point ไปที่ข้อมูลชุดเดียวกัน

**ผลกระทบเชิงปฏิบัติของความต่างนี้**:

- **CPU cache locality**: การตรึง connection กับ worker thread คงที่ของ Actix-web ให้ cache locality ที่
  ดีกว่าในสถานการณ์ที่ connection เดิมมี request ต่อเนื่องจำนวนมาก (keep-alive connection ที่ใช้งานหนัก)
  เพราะข้อมูลที่เกี่ยวข้องกับ connection นั้นๆมักอยู่ใน CPU cache ของ thread เดิมเสมอ ในขณะที่ Tokio
  task-per-connection ของ Axum/Rocket อาจย้าย task ข้าม thread ได้ (แลกกับ load balancing ที่ดีกว่าใน
  สถานการณ์ที่ workload ไม่สมดุลระหว่าง connection)
- **หน่วยความจำที่ใช้ตอน startup**: เพราะ `App` (พร้อม routing table และ middleware ที่ configure ไว้)
  ถูกสร้างซ้ำ N ครั้ง (N = จำนวน worker) Actix-web จึงใช้หน่วยความจำสำหรับ routing table มากกว่า
  Axum/Rocket ที่มี `Router`/route table เพียงชุดเดียวที่ทุก task แชร์กันผ่าน `Arc` ภายใน — ในทางปฏิบัติ
  ความต่างนี้มีขนาดเล็กมากสำหรับแอปทั่วไป (routing table ไม่ใช่โครงสร้างข้อมูลที่ใหญ่) แต่เป็นรายละเอียดที่
  ควรรู้เมื่อ debug memory usage ของเซิร์ฟเวอร์ที่มี route จำนวนมากมาก
- **state ที่ไม่ใช่ `Arc`-wrapped จะพังทันที**: เพราะ factory closure รันซ้ำ N รอบ ถ้าเผลอสร้าง state ที่
  "ไม่แชร์กัน" ไว้ข้างใน closure ตรงๆ (เช่น `Mutex::new(vec![])` ที่สร้างใหม่ *ข้างใน* closure โดยไม่ผ่าน
  `web::Data` จากข้างนอก) แต่ละ worker จะได้ state คนละชุดแยกกันโดยไม่รู้ตัว — ทำให้ endpoint สอง endpoint
  ที่ควรเห็นข้อมูลชุดเดียวกันกลับเห็นข้อมูลคนละชุด ขึ้นอยู่กับว่า request ไปตกที่ worker ไหน (bug ประเภทนี้
  ตรวจยากมากเพราะทำงาน "ถูก" บางครั้งและ "ผิด" บางครั้งแบบสุ่มตาม load balancer ของ OS ว่าส่ง connection ไป
  worker ไหน) — นี่คือกับดักที่กล่าวถึงเพิ่มในหัวข้อกับดักท้ายบท

**สรุปเชิงเปรียบเทียบสั้นๆ**: Tokio task-per-connection (Axum/Rocket) คือโมเดล "M งาน กระจายอัตโนมัติบน N
thread โดย scheduler ตัดสินใจแบบ dynamic" ส่วน Actix-web worker model คือโมเดล "N thread คงที่ แต่ละตัว
เป็นอิสระเกือบสมบูรณ์ (มี App instance ของตัวเอง) รับ connection ที่ OS จัดสรรให้แบบ static ต่อ thread" — ทั้ง
สองโมเดลใช้งานได้ดีในระดับ production เหมือนกัน ความต่างมักไม่ปรากฏชัดจนกว่าจะทำงานกับ workload ที่ไม่สมดุล
มากๆระหว่าง connection หรือจนกว่าจะเขียน state management ผิดแบบที่อธิบายไว้ข้างต้น

### 69.10 Middleware สามแบบ เขียนโค้ดเทียบกันจริง: `tower::Layer`, `Transform`, `Fairing`

หัวข้อ 69.8 อธิบายแนวคิดของ middleware ทั้งสามระบบไว้เชิงทฤษฎีแล้ว — หัวข้อนี้แสดงให้เห็น **โครงร่างโค้ดจริง**
ของ middleware แบบเดียวกัน ("ล็อกเวลาที่ใช้ประมวลผล request แต่ละตัว") เขียนด้วยสามระบบ เพื่อให้เห็นภาพว่า
"เปลือก" ของแต่ละระบบมีรูปร่างต่างกันแค่ไหนจริงๆ ไม่ใช่แค่คำอธิบายลอยๆ (โค้ดในหัวข้อนี้เป็นโครงร่างเพื่อการ
ศึกษาโครงสร้าง — ไม่ได้ curl-test เพราะไม่ใช่ endpoint แต่ syntax ตรงตาม API จริงของแต่ละเฟรมเวิร์กที่ Part
65 และ 68 สอนไว้)

**Axum: `tower::Layer` + `tower::Service`** — ต้อง implement สอง trait ซ้อนกัน (`Layer` สร้าง `Service`,
`Service` ทำงานจริง) เพราะนามธรรมของ `tower` ออกแบบให้ "ห่อ" service เดิมไว้ข้างในแบบ onion:

```rust
use std::task::{Context, Poll};
use tower::{Layer, Service};

#[derive(Clone)]
struct TimingLayer;

impl<S> Layer<S> for TimingLayer {
    type Service = TimingMiddleware<S>;
    fn layer(&self, inner: S) -> Self::Service {
        TimingMiddleware { inner }
    }
}

#[derive(Clone)]
struct TimingMiddleware<S> {
    inner: S,
}

impl<S, Req> Service<Req> for TimingMiddleware<S>
where
    S: Service<Req> + Send + 'static,
    Req: Send + 'static,
{
    type Response = S::Response;
    type Error = S::Error;
    type Future = std::pin::Pin<Box<dyn std::future::Future<Output = Result<S::Response, S::Error>> + Send>>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        self.inner.poll_ready(cx)
    }

    fn call(&mut self, req: Req) -> Self::Future {
        let start = std::time::Instant::now();
        let future = self.inner.call(req);
        Box::pin(async move {
            let result = future.await;
            println!("ใช้เวลา: {:?}", start.elapsed());
            result
        })
    }
}

// ใช้งานผ่าน .layer(TimingLayer) บน Router — Part 65 สอนไว้เต็มรูปแบบ
```

**Actix-web: `Transform` + `Service`** — แนวคิดคล้ายกัน (สอง trait ซ้อนกันแบบ onion เหมือนกัน) แต่เป็น
trait ที่ Actix-web กำหนดเอง ไม่ใช่ `tower`:

```rust
use actix_web::dev::{Service, ServiceRequest, ServiceResponse, Transform};
use actix_web::Error;
use futures_util::future::LocalBoxFuture;
use std::future::{ready, Ready};

struct TimingMiddlewareFactory;

impl<S, B> Transform<S, ServiceRequest> for TimingMiddlewareFactory
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Transform = TimingMiddleware<S>;
    type InitError = ();
    type Future = Ready<Result<Self::Transform, Self::InitError>>;

    fn new_transform(&self, service: S) -> Self::Future {
        ready(Ok(TimingMiddleware { service }))
    }
}

struct TimingMiddleware<S> {
    service: S,
}

impl<S, B> Service<ServiceRequest> for TimingMiddleware<S>
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Future = LocalBoxFuture<'static, Result<Self::Response, Self::Error>>;

    actix_web::dev::forward_ready!(service);

    fn call(&self, req: ServiceRequest) -> Self::Future {
        let start = std::time::Instant::now();
        let fut = self.service.call(req);
        Box::pin(async move {
            let res = fut.await?;
            println!("ใช้เวลา: {:?}", start.elapsed());
            Ok(res)
        })
    }
}

// ใช้งานผ่าน .wrap(TimingMiddlewareFactory) บน App — Part 68 สอนไว้เต็มรูปแบบ
```

**Rocket: `Fairing`** — สั้นกว่าอีกสองแบบมาก เพราะไม่ใช่ pattern "ห่อ service ซ้อนกัน" แต่เป็น callback ที่
hook เข้ากับจุดที่กำหนดไว้ล่วงหน้าตายตัวของ lifecycle:

```rust
use rocket::fairing::{Fairing, Info, Kind};
use rocket::{Data, Request};
use std::time::Instant;

struct Timing;

#[rocket::async_trait]
impl Fairing for Timing {
    fn info(&self) -> Info {
        Info { name: "Timing", kind: Kind::Request | Kind::Response }
    }

    async fn on_request(&self, req: &mut Request<'_>, _: &mut Data<'_>) {
        req.local_cache(|| Instant::now());
    }

    async fn on_response<'r>(&self, req: &'r Request<'_>, _: &mut rocket::Response<'r>) {
        let start = req.local_cache(|| Instant::now());
        println!("ใช้เวลา: {:?}", start.elapsed());
    }
}

// ใช้งานผ่าน .attach(Timing) บน rocket::build()
```

**สังเกตความยาวของโค้ดทั้งสามชุด**: Axum และ Actix-web ต้อง implement สอง trait ที่ซับซ้อนใกล้เคียงกัน
(เพราะทั้งคู่ใช้ pattern "ห่อ service ซ้อนกัน" ที่ทำให้ composable สูงมาก แลกกับ boilerplate ที่มากกว่า) ส่วน
Rocket implement แค่ trait เดียว (`Fairing`) ที่มี method callback ตรงไปตรงมา — นี่คือหลักฐานโค้ดจริงของสิ่ง
ที่หัวข้อ 69.8 อธิบายไว้เชิงทฤษฎี: **นามธรรมแบบ `Service`-composition (Axum/Actix-web) ให้พลังการ compose
middleware ซ้อนกันได้ไม่จำกัดชั้นและใช้ร่วมกับ ecosystem อื่นได้ (เฉพาะฝั่ง `tower`) แต่แลกมาด้วย
boilerplate ที่มากกว่า** ในขณะที่ **นามธรรมแบบ callback-hook ของ Rocket เขียนง่ายกว่ามากสำหรับ use case
พื้นฐาน แต่ไม่มีสิทธิ "ปฏิเสธที่จะเรียก inner service" แบบที่ `Service::call` ทำได้** (เช่น short-circuit
request โดยไม่ให้ไปถึง handler เลย — ทำได้ใน `tower`/Actix-web middleware ตรงๆ ผ่านการไม่เรียก
`inner.call()` แต่ Fairing's `on_request` ไม่สามารถ "ยกเลิก" การ routing ที่กำลังจะเกิดขึ้นได้ในลักษณะ
เดียวกัน — ต้องใช้กลไกอื่นของ Rocket เช่น request guard ที่คืน `Outcome::Error`/`Outcome::Forward` แทนถ้า
ต้องการ short-circuit)

### 69.11 Performance: อย่าเลือกเฟรมเวิร์กจากตัวเลข Benchmark เพียงอย่างเดียว

คำถามที่มักถูกถามตอนเทียบเว็บเฟรมเวิร์กคือ "แล้วอันไหนเร็วกว่ากัน" — คำตอบที่ซื่อสัตย์ที่สุดคือ: **ตัวเลข
benchmark สาธารณะที่มีชื่อเสียง (เช่นกลุ่ม benchmark แบบ TechEmpower ที่เทียบเว็บเฟรมเวิร์กหลายภาษา/หลาย
framework ด้วย workload มาตรฐานอย่าง "plaintext", "JSON serialization", "database query")** มักแสดงให้
เห็นว่าเว็บเฟรมเวิร์กที่เขียนด้วย Rust (ไม่ว่า Axum, Actix-web, หรือ Rocket) **อยู่ในกลุ่มบนของตารางเทียบกับ
ภาษาอื่นอย่างชัดเจน** เพราะทั้งหมดไม่มี garbage collector, ทั้งหมดใช้ async I/O แบบ zero-cost abstraction
(Part 46-49 สอนพื้นฐานนี้ไว้เต็มรูปแบบ) — **แต่ความต่างของตัวเลขระหว่างสามเฟรมเวิร์ก Rust นี้เอง (Axum
เทียบ Actix-web เทียบ Rocket) มักเล็กกว่าความต่างระหว่าง Rust กับภาษาอื่นมาก** และตัวเลขที่แม่นยำเปลี่ยนไป
ทุกครั้งที่มีการ release เวอร์ชันใหม่ของแต่ละเฟรมเวิร์ก (โดยเฉพาะ Axum ที่มีการปรับปรุง performance ของ
`matchit` router และ `hyper` อยู่เรื่อยๆ) — บทนี้จะ**ไม่อ้างตัวเลขที่เฉพาะเจาะจง**เพราะไม่มีทางยืนยันความ
ถูกต้อง ณ เวลาที่คุณอ่านบทนี้จริง (ตัวเลขที่ยืนยันได้วันนี้อาจล้าสมัยไปแล้วตอนคุณอ่าน) — สิ่งที่ยืนยันได้และ
ยังจริงเสมอคือ**รูปแบบทั่วไป** (general shape) ของผลลัพธ์: เว็บเฟรมเวิร์ก Rust ทั้งสามอยู่ในระดับ throughput
สูงมากเมื่อเทียบข้ามภาษา และผลต่างภายในกลุ่ม Rust เองมักอยู่ในช่วงที่ workload จริงของแอปพลิเคชันไม่ค่อย
รู้สึกถึงความต่าง

**เหตุผลเชิงลึกที่สำคัญกว่าตัวเลข**: Part 61 อธิบายไว้ว่า HTTP request หนึ่งตัวต้องผ่านหลายขั้น (parse
protocol, routing, extract data, รัน business logic, serialize response) — งานที่**เฟรมเวิร์กควบคุมได้**
(parsing, routing, extraction) ใช้เวลาระดับ **ไมโครวินาที** ในทุกเฟรมเวิร์ก Rust สมัยใหม่ ในขณะที่แอป
พลิเคชันจริงส่วนใหญ่ (โดยเฉพาะที่ Part 70 กำลังจะสอนต่อ — เชื่อมต่อ PostgreSQL ด้วย SQLx) ใช้เวลาส่วนใหญ่
ไปกับ **I/O ที่ต้องรอ** — round-trip ไปยัง database, เรียก external API, อ่าน/เขียน disk — ซึ่งใช้เวลาระดับ
**มิลลิวินาที** (สูงกว่า framework overhead หลายร้อยถึงหลายพันเท่า) ผลคือในแอปพลิเคชันจริงที่มี database
เกี่ยวข้อง **ความต่างของ throughput ระหว่าง Axum, Actix-web, Rocket มักถูกกลบด้วยเวลาที่รอ database
เกือบทั้งหมด** — เลือกเฟรมเวิร์กที่เร็วกว่าอีกตัว 5% แต่ยัง query database แบบไม่มี index หรือไม่ใช้
connection pool (Part 70 จะสอนเรื่อง connection pool ของ SQLx โดยตรง) ให้ผลลัพธ์แย่กว่าการเลือกเฟรมเวิร์ก
ที่ "ช้ากว่า" แต่ query database อย่างถูกต้องมาก

**ข้อสรุปเชิงปฏิบัติ**: อย่าเลือกเฟรมเวิร์กจากตัวเลข benchmark synthetic (ที่มักทดสอบ endpoint แบบ
"plaintext" หรือ "JSON" ล้วนๆที่ไม่มี I/O จริงเลย) เพียงอย่างเดียว — ให้เลือกจากปัจจัยอื่นที่ตารางในหัวข้อ
69.8 และ decision framework ในหัวข้อ 69.13 ระบุไว้ (ergonomics ที่ทีมถนัด, ระบบนิเวศ middleware ที่ต้องใช้,
ความคุ้นเคยของทีม) แล้ว**ค่อยไปโฟกัสที่การ optimize จุดที่ส่งผลจริงกับ performance ของแอปคุณ** — ซึ่งมักคือ
database query pattern, connection pooling, และ caching strategy ไม่ใช่การเลือกเฟรมเวิร์ก HTTP ตัวไหน

### 69.12 Ecosystem และ Interop: ทำไมหลักสูตรนี้เดินหน้าต่อด้วย Axum

หัวข้อนี้ตอบคำถามตรงๆที่ผู้เรียนหลายคนคงสงสัยตั้งแต่ Part 62: **"ทำไมหลักสูตรสอนทั้ง Axum และ Actix-web
เต็มรูปแบบ แต่พอถึง Part 70 (database) เป็นต้นไป กลับใช้ Axum เป็นหลักตัวเดียว?"**

**เหตุผลที่ 1 — ความเข้ากันกับ `tower`/Tokio ecosystem ที่หลักสูตรสอนมาตั้งแต่ต้น**: หัวข้อ 69.8 พิสูจน์ให้
เห็นแล้วว่า Axum ใช้ `tower`/`hyper` ตรงๆ ในขณะที่ Actix-web และ Rocket มีระบบของตัวเอง — หลักสูตรนี้ใช้เวลา
สอน **Tokio เต็มรูปแบบ (Part 48-49)** และแนวคิด **`Service`/middleware แบบ `tower`** ผ่าน Part 65 มาแล้ว
การเดินหน้าต่อด้วย Axum หมายความว่าความรู้เรื่อง `tower`/`tower-http` middleware ที่เรียนไปแล้วนำไปใช้ต่อ
ได้ทันทีในบททุกบทถัดไป (เช่น Part 77 เรื่อง WebSocket ที่ Axum มี extractor สำหรับ WebSocket built-in
ผ่าน `axum::extract::ws::WebSocketUpgrade` ตรงๆ, และ Part 85 เรื่อง OpenAPI generation ที่ crate อย่าง
`utoipa` มี integration package เฉพาะสำหรับ Axum ที่ maintain แข็งแรง — `utoipa-axum`) — การใช้ Axum ต่อจึง
เป็นการ**ลดจำนวนแนวคิดใหม่ที่ต้องแนะนำ**ในบทถัดไป เพราะพื้นฐาน (`tower`, Tokio, extractor pattern) ถูก
สอนไปแล้วอย่างละเอียดตลอด Part 62-66

**เหตุผลที่ 2 — โมเมนตัมของ community และทิศทางของ ecosystem ปัจจุบัน**: ในช่วงหลัง (ที่หลักสูตรนี้เขียน)
Axum เป็นเฟรมเวิร์กที่ crate เสริมใหม่ๆในระบบนิเวศ Rust จำนวนมากเลือกทำ integration ให้ก่อนเป็นอันดับแรก —
ส่วนหนึ่งเป็นผลจากการที่ทีม Tokio ดูแล Axum เอง ทำให้มันอยู่ตรงกลางของ "stack" ที่ crate เสริมส่วนใหญ่
(เช่น driver ของ database, crate สำหรับ observability, crate สำหรับ authentication) เขียน adapter ให้
โดยธรรมชาติ เพราะ adapter สำหรับ `tower::Service` ใช้ได้กับทุกเฟรมเวิร์กที่พูดภาษา `tower` (ซึ่งรวมถึง
Axum และเฟรมเวิร์กอื่นอย่าง Warp/Tonic ด้วย ไม่ใช่ผูกกับ Axum เพียงตัวเดียว) — นี่ไม่ได้แปลว่า Actix-web หรือ
Rocket "ตาย" หรือไม่มีคนใช้ (ทั้งสองยังมี production user จำนวนมากและได้รับการดูแลต่อเนื่อง) แต่การ integration
กับ crate ใหม่ๆมักมาช้ากว่าหรือต้องเขียน adapter เพิ่มเอง

**เหตุผลที่ 3 — WebSocket และ OpenAPI ที่หลักสูตรจะสอนต่อ**: Part 77 (WebSocket) ในหลักสูตรนี้จะสอนผ่าน
Axum โดยตรง เพราะ `axum::extract::ws` เป็นส่วนหนึ่งของตัวเฟรมเวิร์กเอง ไม่ต้องเพิ่ม dependency แยก (Actix-web
มี `actix-web-actors` เป็น crate เสริมสำหรับ WebSocket ที่ต้องเรียนรู้ actor pattern เพิ่มเติม ส่วน Rocket
ยังไม่มี WebSocket support ที่เป็นทางการในตัวเฟรมเวิร์กหลักในระดับเดียวกัน) — Part 85 (OpenAPI ผ่าน
`utoipa`) จะใช้ `utoipa-axum` ที่ integration แน่นกับ `Router` ของ Axum ตรงๆ ทำให้ generate เอกสาร API จาก
โค้ด route จริงได้สะดวก การเลือกเดินหน้าด้วย Axum จึงทำให้บททั้งสองนี้สอนได้ตรงประเด็นโดยไม่ต้องสอน
ecosystem คู่ขนานสองชุด

**ตัวอย่างที่จับต้องได้ — ฉีด `sqlx::PgPool` เข้าเป็น shared state เตรียมไว้สำหรับ Part 70**: ไม่ว่าจะใช้
เฟรมเวิร์กไหน รูปแบบการฉีด database connection pool เข้าเป็น state ที่ทุก handler เข้าถึงได้ก็ทำตามหลักการ
เดียวกับ `AppState` ที่บทนี้ใช้มาตลอด (Part 70 จะสอนเรื่อง `sqlx::PgPool` เต็มรูปแบบ บทนี้แค่โชว์ "เปลือก" ที่
ต่างกันของแต่ละเฟรมเวิร์กล่วงหน้า):

```rust
// Axum — ห่อ PgPool ด้วย Arc เอง (หรือ clone ตรงๆก็ได้เพราะ PgPool เป็น Arc-based อยู่แล้วภายใน)
async fn list_tasks(State(pool): State<sqlx::PgPool>) -> Json<Vec<Task>> { /* ... */ }
let app = Router::new().route("/tasks", get(list_tasks)).with_state(pool);

// Actix-web — ห่อด้วย web::Data ตามปกติ
#[get("/tasks")]
async fn list_tasks(pool: web::Data<sqlx::PgPool>) -> impl Responder { /* ... */ }
// .app_data(web::Data::new(pool.clone())) ตอนสร้าง App ในแต่ละ worker (หัวข้อ 69.9)

// Rocket — ผูกผ่าน .manage() เหมือน state อื่นๆ
#[get("/tasks")]
fn list_tasks(pool: &State<sqlx::PgPool>) -> Json<Vec<Task>> { /* ... */ }
// rocket::build().manage(pool)
```

สังเกตว่า **`sqlx::PgPool` ตัวเดียวกันเป๊ะ** ใช้ได้กับทั้งสามเฟรมเวิร์กโดยไม่ต้องแก้ driver หรือ query ใดๆ
เลย — เพราะ SQLx เป็น crate ที่ทำงานอิสระจากเว็บเฟรมเวิร์กโดยสมบูรณ์ (ตรงกับหลักการ "database layer
portable" ที่หัวข้อ 69.14 จะอธิบายเต็มรูปแบบ) ความต่างมีแค่ **วิธีห่อและดึงค่า pool ออกมา** ซึ่งเป็นรูปแบบ
เดียวกับที่หัวข้อ 69.7 เปรียบเทียบ `State<T>`/`web::Data<T>`/`&State<T>` ไว้แล้วทุกประการ — นี่คือเหตุผลที่
Part 70 (SQLx) เขียนแยกจากเรื่อง "เฟรมเวิร์กไหน" ได้อย่างเป็นธรรมชาติ

**ตารางเทียบ WebSocket support** (foreshadow Part 77 — ยังไม่ต้องเข้าใจโค้ด WebSocket ตอนนี้ แค่รู้ระดับการ
รองรับของแต่ละเฟรมเวิร์กไว้ก่อน):

| เฟรมเวิร์ก | WebSocket support | หมายเหตุ |
|---|---|---|
| Axum | Built-in ในตัวเฟรมเวิร์กหลัก ผ่าน `axum::extract::ws::WebSocketUpgrade` | ไม่ต้องเพิ่ม dependency แยก — Part 77 จะสอนใช้ตรงนี้ |
| Actix-web | ผ่าน crate เสริม `actix-web-actors` (หรือทำผ่าน `actix-ws` ที่เป็น crate ชุมชนอีกตัว) | ต้องเรียนรู้ actor pattern เพิ่ม (สำหรับ `actix-web-actors`) หรือเรียนรู้ API ของ crate เสริมอีกชุด |
| Rocket | ไม่มี WebSocket support ในตัวเฟรมเวิร์กหลักที่เทียบเท่ากับอีกสองตัว ณ ปัจจุบัน | ต้องพึ่งพา crate ชุมชนเพิ่มเติมที่ maturity ต่ำกว่า |

**ข้อความสำคัญที่ต้องเข้าใจให้ถูก**: การเลือกนี้ **เป็นการตัดสินใจเชิงหลักสูตร (curriculum design
decision)** ที่พิจารณาจาก (ก) ความต่อเนื่องกับสิ่งที่สอนไปแล้ว (Tokio, `tower`) (ข) ทิศทางที่ ecosystem crate
ใหม่ๆกำลังไปในช่วงที่เขียนหลักสูตรนี้ และ (ค) ต้องเลือกเฟรมเวิร์กเดียวเพื่อไม่ให้เนื้อหาบทหลังบวมด้วยการสอน
สาม ecosystem คู่ขนาน — **นี่ไม่ใช่การอ้างว่า Axum เป็นเฟรมเวิร์กที่ "ดีที่สุด" แบบสัมบูรณ์หรือเหมาะกับทุก
โปรเจกต์เสมอไป** ทีมที่มีระบบเดิมเป็น Actix-web อยู่แล้ว หรือทีมที่ต้องการ ergonomics แบบ Rocket สำหรับ
โปรเจกต์ขนาดเล็กถึงกลาง ก็มีเหตุผลที่สมเหตุสมผลอย่างสมบูรณ์ในการเลือกเฟรมเวิร์กอื่น — หัวข้อ 69.13 จะให้
กรอบการตัดสินใจที่ใช้ได้จริงสำหรับสถานการณ์ของคุณเอง ไม่ใช่ของหลักสูตรนี้

### 69.13 Decision Framework: จะเลือกเฟรมเวิร์กไหนสำหรับโปรเจกต์ถัดไปของคุณ

ตารางด้านล่างนี้ไม่ใช่ flowchart แบบภาพ แต่เป็น **decision table เชิงข้อความ** ที่ให้คุณอ่านจากบนลงล่าง —
หาแถวที่ตรงกับสถานการณ์ของโปรเจกต์คุณที่สุด (อาจตรงมากกว่าหนึ่งแถว — ในกรณีนั้นให้ชั่งน้ำหนักตามลำดับความ
สำคัญของทีมคุณเอง):

| สถานการณ์ของคุณ | เฟรมเวิร์กที่ควรพิจารณาก่อน | เหตุผล |
|---|---|---|
| ทีมมีระบบเดิมที่ใช้ Actix-web อยู่แล้ว และแอปทำงานได้ดี | **Actix-web** (คงไว้) | ต้นทุนการย้ายเฟรมเวิร์กมักสูงกว่าประโยชน์ที่ได้ (หัวข้อ 69.14) เว้นแต่มีปัญหาเชิงสถาปัตยกรรมที่แก้ไม่ได้ในเฟรมเวิร์กปัจจุบันจริงๆ |
| ต้องการ ecosystem middleware ที่ใช้ร่วมกับ gRPC/RPC ได้ในระบบเดียวกัน (เช่นใช้ `tonic` สำหรับ gRPC ควบคู่ REST API) | **Axum** | ทั้ง Axum และ `tonic` พูดภาษา `tower::Service` เดียวกัน — middleware เขียนครั้งเดียวใช้ได้ทั้งสองฝั่ง |
| ทีมเล็ก โปรเจกต์ขนาดเล็กถึงกลาง ต้องการ ergonomics สูงสุด ลด boilerplate ให้มากที่สุด (form parsing, cookie, TLS built-in) | **Rocket** | ปรัชญา batteries-included ลดเวลาที่ต้องเขียน infrastructure code เองสำหรับงานที่ Rocket มีให้ในตัว |
| ต้องการ compile-time safety net สำหรับ route ที่มีจำนวนมากและเปลี่ยนแปลงบ่อย (ทีมใหญ่ หลาย feature team แก้ route ร่วมกัน) | **Rocket** | ระบบตรวจสอบ path/parameter ตอน compile time (หัวข้อ 69.6) ช่วยจับ route ที่เขียนผิดก่อน deploy — มีประโยชน์มากขึ้นตามขนาดทีมและความถี่ของการเปลี่ยน route |
| ต้องการความคุ้นเคยกับ Tokio ecosystem ที่กว้างที่สุด ต้องผสมกับ crate อื่นในระบบนิเวศ Tokio จำนวนมาก | **Axum** | ทีม Tokio ดูแล Axum เอง การผสานกับ crate อื่นในวง Tokio (เช่น `tokio-tungstenite`, `tower-http`) มักตรงไปตรงมาที่สุด |
| กำลังเรียน/สอน Rust ใหม่และอยากให้ error message ตรงประเด็นที่สุดตอนเขียนผิด | **Rocket** (สำหรับ error ด้าน routing) หรือ **Axum** (ถ้าใช้ `#[axum::debug_handler]` ควบคู่ — Part 62.5) | ทั้งสองมีจุดแข็งด้าน error message คนละแบบ — Rocket จับ route/parameter ผิดตอน compile ได้ดี ส่วน Axum มี `debug_handler` ช่วยแปล trait bound error ที่ซับซ้อนให้อ่านง่ายขึ้น |
| ต้องการ throughput สูงสุดในสถานการณ์ที่ CPU-bound จริง (ไม่ใช่ I/O-bound) และมีทีมที่พร้อม benchmark เฉพาะ workload ของตัวเอง | **ต้อง benchmark เองด้วย workload จริง** ไม่มีคำตอบตายตัว | หัวข้อ 69.11 อธิบายไว้แล้วว่าตัวเลข synthetic benchmark ทั่วไปมักไม่สะท้อน workload จริงของคุณ |
| ไม่แน่ใจ ไม่มีข้อจำกัดพิเศษ อยากได้ตัวเลือกที่ปลอดภัยที่สุดสำหรับผสานกับ ecosystem Rust ในวงกว้างระยะยาว | **Axum** | ความเข้ากันกับ `tower`/Tokio ที่กว้างที่สุด ณ ปัจจุบัน ลดความเสี่ยงที่จะต้อง "ค้นหา adapter เอง" เมื่อ crate ใหม่ๆออกมา |

**ข้อควรระวังสำคัญเกี่ยวกับตารางนี้**: ตารางนี้ให้ "จุดเริ่มต้นที่สมเหตุสมผล" ไม่ใช่คำตอบสุดท้ายที่ใช้แทน
การวิเคราะห์จริงของทีมคุณ — ปัจจัยที่สำคัญที่สุดในทางปฏิบัติซึ่งไม่มีตารางไหนแทนได้คือ **ทีมของคุณคุ้นเคย
กับอะไรอยู่แล้ว** — ถ้าทีมทั้งทีมเขียน Actix-web มาสามปีแล้วทำงานได้ดี การเปลี่ยนไป Axum เพราะ "บทความบอก
ว่าดีกว่า" มักเป็นการตัดสินใจที่แพงเกินไปเทียบกับประโยชน์ที่ได้จริง เว้นแต่มีเหตุผลเชิงเทคนิคที่หนักแน่นจริงๆ
(เช่นต้องผสาน gRPC เข้าไปในระบบเดียวกัน)

### 69.14 Migration: ย้ายระหว่างเฟรมเวิร์กยากแค่ไหนจริงๆ

หัวข้อ 69.7 ให้เห็นแล้วว่า business logic ภายใน handler (`.lock()`, `.find()`, `match`) เหมือนกันทุก
ตัวอักษรระหว่างสามเฟรมเวิร์ก — นี่ไม่ใช่เรื่องบังเอิญ แต่สะท้อนหลักการสำคัญที่ควรจำไว้เสมอเมื่อออกแบบ
โครงสร้างโปรเจกต์เว็บใน Rust (หรือภาษาไหนก็ตาม): **แยก "domain logic" ออกจาก "เปลือกของเฟรมเวิร์ก" ให้
ชัดเจนที่สุดเท่าที่ทำได้**

**สิ่งที่ portable (ย้ายข้ามเฟรมเวิร์กได้ง่ายมาก แทบไม่ต้องแก้)**:

- **โครงสร้างข้อมูล (struct/enum) ที่ derive `Serialize`/`Deserialize`** — `Task`, `CreateTaskRequest` และ
  struct อื่นๆที่ Part 57-58 สอนไว้ใช้ `serde` ตัวเดียวกันหมดไม่ว่าเฟรมเวิร์กไหน (ข้อยกเว้นเล็กน้อยคือ
  Rocket ต้องเพิ่ม `#[serde(crate = "rocket::serde")]` เพราะ re-export ของตัวเอง — แก้แค่ attribute เดียว
  ไม่ต้องแก้ logic)
- **Business logic ที่ไม่รู้จักเฟรมเวิร์กเลย** — ฟังก์ชันอย่าง "หา task ที่ id ตรงกัน", "validate ว่า title
  ไม่ว่างเปล่า", "คำนวณราคารวมของ order" ถ้าเขียนเป็นฟังก์ชัน Rust ธรรมดาที่รับ/คืนค่าเป็น domain type
  (ไม่ใช่ `Request`/`Response` ของเฟรมเวิร์กใดตรงๆ) ย้ายได้ 100% โดยไม่ต้องแก้แม้แต่บรรทัดเดียว — นี่คือ
  เหตุผลเชิงสถาปัตยกรรมที่สำคัญที่สุดที่ทำให้โปรเจกต์เว็บที่ออกแบบดี **แยก layer "service"/"repository"
  ออกจาก layer "route handler"** อย่างชัดเจน (แนวคิดคล้าย Clean Architecture/Hexagonal Architecture ที่พบ
  ในหลายภาษา ไม่ใช่เฉพาะ Rust)
- **Database layer** (ที่ Part 70 กำลังจะสอนผ่าน SQLx) — connection pool, query, และ struct ที่ map จาก
  แถวข้อมูลกลับมาเป็น Rust type ไม่ผูกกับเว็บเฟรมเวิร์กเลย เพราะ SQLx เป็น crate ที่ทำงานได้อิสระจาก
  HTTP layer โดยสิ้นเชิง — ย้ายเว็บเฟรมเวิร์กไม่กระทบ database layer เลยถ้าแยก layer ไว้ถูกต้อง

**สิ่งที่ framework-specific (ต้องเขียนใหม่เมื่อย้าย)**:

- **Route registration และ handler signature** — ต้องเขียนใหม่ทั้งหมดตามรูปแบบของเฟรมเวิร์กปลายทาง (จาก
  `.route("/tasks/{id}", get(handler))` ของ Axum ไปเป็น `#[get("/tasks/<id>")]` ของ Rocket) แม้ *เนื้อหา*
  ของ handler (บรรทัดข้างในฟังก์ชัน) ย้ายได้เกือบทั้งหมดถ้าแยก business logic ไว้ดีตามข้อก่อนหน้า สิ่งที่
  ต้องแก้จริงๆคือ**ส่วนหัวของฟังก์ชัน** (parameter ที่เป็น extractor/guard) และ**การลงทะเบียน route**
- **Extractor/Guard ที่ทำงานเฉพาะเฟรมเวิร์ก** — `State<T>` ของ Axum, `web::Data<T>` ของ Actix-web,
  `&State<T>` ของ Rocket ล้วนเป็น type ที่ผูกกับเฟรมเวิร์กนั้นๆตรงๆ ย้ายไม่ได้ ต้องเขียนใหม่เสมอ
- **Middleware** — ตามที่หัวข้อ 69.8 อธิบายไว้ `tower::Layer` ของ Axum, `Transform` ของ Actix-web, และ
  `Fairing` ของ Rocket เป็นนามธรรมคนละตัวกันโดยสิ้นเชิง **ย้ายไม่ได้แม้แต่นิดเดียว** — ต้องเขียน middleware
  ใหม่ทั้งหมดตาม pattern ของเฟรมเวิร์กปลายทาง แม้ *สิ่งที่ middleware นั้นทำ* (เช่น "ล็อก request ทุกตัว")
  จะเหมือนกันก็ตาม
- **Error handling wiring** — `IntoResponse` ของ Axum, `ResponseError` ของ Actix-web, `Responder` +
  catcher ของ Rocket เป็น trait คนละตัวกัน แม้ error enum ที่ derive `thiserror` (Part 30-31) ย้ายได้ตรงๆ
  (มันเป็น Rust ธรรมดา) แต่ **ส่วนที่ผูก error enum นั้นเข้ากับ HTTP response** ต้องเขียนใหม่เสมอ

**ประเมินความยากของการย้ายจริงในทางปฏิบัติ**: สำหรับแอปพลิเคชันที่แยก layer ไว้ดี (business logic +
database layer อยู่คนละ module จาก route handler อย่างชัดเจน ตามที่ Part 16 สอนเรื่อง module ไว้) การย้าย
จากเฟรมเวิร์กหนึ่งไปอีกเฟรมเวิร์กหนึ่งในสามตัวนี้ **มักเป็นงานระดับ "เขียน route/handler/middleware layer
ใหม่ทั้งหมด แต่ธุรกิจหลัก (core logic) ไม่ต้องแตะ"** — งานที่ใหญ่แต่มีขอบเขตชัดเจนและคาดเดาได้ ไม่ใช่งานที่
ต้อง "เขียนแอปใหม่ทั้งหมดตั้งแต่ต้น" ในทางกลับกัน สำหรับแอปที่เขียนแบบ "handler function ทำทุกอย่างในตัว"
(query database ตรงในบรรทัดเดียวกับที่ parse request, ไม่มีการแยก layer เลย) การย้ายเฟรมเวิร์กจะเจ็บปวดกว่า
มากเพราะ business logic กับเปลือกเฟรมเวิร์กพันกันจนแยกไม่ออก — นี่คือเหตุผลเชิงปฏิบัติอีกข้อที่สนับสนุนการ
แยก layer ให้ดีตั้งแต่ต้น ไม่ใช่แค่เพื่อความสวยงามของโค้ด แต่เพื่อ**ลดความเสี่ยงของการผูกติดกับเฟรมเวิร์กหนึ่ง
ตัวตลอดไป** (vendor lock-in เชิง framework)

## กับดักที่พบบ่อย (Common Pitfalls)

**1. เข้าใจผิดว่า Rocket "ยังต้องใช้ nightly Rust" — ข้อมูลล้าสมัยที่ยังเวียนอยู่ในหลายบทความเก่า**

บทความ/คำแนะนำเก่าจำนวนมาก (เขียนก่อน Rocket 0.5 release ปลายปี 2023) ยังบอกว่า Rocket ต้องใช้ nightly
toolchain — ถ้าเจอบทความแบบนี้ **ให้เช็ควันที่เขียนก่อนเชื่อ** หัวข้อ 69.2-69.3 ของบทนี้พิสูจน์ให้เห็นแล้วว่า
Rocket 0.5.1 compile และรันได้บน stable Rust เต็มรูปแบบ ไม่ต้องรัน `rustup default nightly` หรือเพิ่ม
`rust-toolchain.toml` ที่ระบุ nightly เลยแม้แต่นิดเดียว — ถ้าคุณเจอ error ที่บอกว่าต้องใช้ nightly ตอนใช้
Rocket ให้เช็คก่อนว่าคุณเผลอเปิดใช้ฟีเจอร์ทดลอง (experimental feature) ของ Rocket ที่ยังรอ stabilize อยู่
หรือไม่ (มักระบุชื่อฟีเจอร์ที่ต้องเปิดชัดเจนใน `Cargo.toml`) ไม่ใช่การใช้งานพื้นฐานทั่วไป

**2. คิดว่า request guard ของ Rocket ใช้แทน extractor ของ Axum ได้แบบ 1:1 โดยไม่ต้องปรับ**

หัวข้อ 69.4 อธิบายไว้ว่าทั้งสองแนวคิดคล้ายกันในภาพรวม (ดึงข้อมูลจาก request แบบ type-safe ก่อนเข้า
handler) แต่ **trait ที่ต้อง implement เป็นคนละตัวกันโดยสิ้นเชิง** (`FromRequest` ของ Rocket ไม่ใช่
`FromRequest` ของ Axum แม้ชื่อ trait เหมือนกัน) — ถ้าเขียน custom extractor ไว้แล้วสำหรับ Axum แล้วคิดว่า
"เอาไปแปะใน Rocket ได้เลย" จะเจอ compile error ทันทีเพราะ signature ของ trait method ต่างกัน (Axum ใช้
`async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection>`, Rocket ใช้
`async fn from_request(request: &'r Request<'_>) -> Outcome<Self, Self::Error>` ที่มี `Outcome` เป็น
enum สามทาง `Success`/`Error`/`Forward` ไม่ใช่ `Result` สองทางแบบ Axum) — ต้องเขียนใหม่เสมอเมื่อย้าย
เฟรมเวิร์ก ตรงกับที่หัวข้อ 69.14 สรุปไว้

**3. ลืมว่า middleware ของ `tower-http` ใช้กับ Actix-web หรือ Rocket ไม่ได้เลย แล้วพยายาม `cargo add
tower-http` เข้าโปรเจกต์ Actix-web/Rocket**

จะได้ crate มา compile ผ่าน (เพราะ `tower-http` เป็น crate อิสระ ไม่ผูกกับ Axum ตรงๆ) แต่พอพยายามเอา
`Layer` ของมัน (เช่น `CorsLayer`) มาต่อกับ `App` ของ Actix-web หรือ `rocket::build()` ของ Rocket จะเจอ
compile error ทันทีเพราะ `App`/`Rocket` ไม่มี method ที่รับ `tower::Layer` — หัวข้อ 69.8 อธิบายไว้แล้วว่า
ทั้งสองเฟรมเวิร์กไม่ได้ใช้ `tower` เป็นนามธรรมกลาง วิธีแก้คือใช้ crate เฉพาะของแต่ละเฟรมเวิร์ก
(`actix-cors` สำหรับ Actix-web ที่มี `Transform` ของตัวเอง หรือเขียน Fairing เองสำหรับ Rocket)

**4. เลือกเฟรมเวิร์กจากตัวเลข benchmark synthetic เพียงอย่างเดียว โดยไม่ทดสอบกับ workload จริงของตัวเอง**

หัวข้อ 69.11 อธิบายไว้ละเอียดแล้ว — benchmark แบบ "plaintext"/"JSON" ล้วนๆไม่มี I/O จริงเลย ไม่ได้สะท้อน
แอปพลิเคชันจริงที่ต้องคุยกับ database เกือบทุกครั้ง (Part 70 กำลังจะสอนเรื่องนี้ต่อ) — ถ้าต้องตัดสินใจจาก
performance จริงๆ ให้เขียน prototype เล็กๆด้วย workload ที่ใกล้เคียงระบบจริงที่สุด (รวม database call จริง)
แล้ว benchmark เอง ไม่ใช่เชื่อตัวเลขจาก benchmark สาธารณะที่ไม่มี context ตรงกับระบบของคุณ

**5. เผลอส่ง HTML default catcher ของ Rocket กลับไปให้ client ที่คาดหวัง JSON**

หัวข้อ 69.3 และ 69.5 แสดงให้เห็นว่า Rocket ตอบ 404/422 เป็นหน้า HTML โดยอัตโนมัติถ้าไม่ได้ custom catcher
เอง — สำหรับ REST API ที่ client (เช่น frontend JavaScript หรือ mobile app) คาดหวัง JSON เสมอ การส่ง HTML
กลับไปโดยไม่ตั้งใจจะทำให้ client-side parsing พังทันที (`response.json()` ฝั่ง JavaScript จะ throw
error เพราะ body ไม่ใช่ JSON) — ต้องเขียน custom catcher ด้วย `#[catch(code)]` แล้ว register ผ่าน
`.register("/", catchers![...])` เพื่อให้ error response เป็น JSON เสมอสำหรับ API ที่ตั้งใจให้เป็น JSON-only
— นี่คือตัวอย่างจริงที่ "ความสะดวกของ default" กลายเป็น "กับดักที่ต้องรู้ทันและแก้เอง"

**6. คิดว่า compile-time route checking ของ Rocket รับประกันว่า route จะไม่มี bug เลย**

หัวข้อ 69.6 เตือนไว้แล้วว่านี่คือ safety net ที่แคบ — มันตรวจแค่ความสอดคล้องของชื่อ/ประเภทระหว่าง path/query
string กับ parameter เท่านั้น มันไม่ตรวจ business logic, ไม่ตรวจ route ที่ overlap กันโดยไม่ตั้งใจ
(เช่น `/tasks/<id>` กับ `/tasks/done` ชนกันในบางกรณีถ้าออกแบบไม่ดี — ปัญหานี้เกิดได้ในทุกเฟรมเวิร์ก ไม่ใช่
แค่ Rocket) และไม่ตรวจว่า handler จะ panic ตอน runtime หรือไม่ — integration test (Part 63 สอนพื้นฐาน
การเทสต์ route ของ Axum ไว้) ยังจำเป็นเสมอไม่ว่าจะใช้เฟรมเวิร์กไหน

**7. สร้าง state ข้างใน factory closure ของ Actix-web แทนที่จะสร้างข้างนอกแล้ว `.clone()` เข้าไป**

หัวข้อ 69.9 อธิบายไว้ว่า `HttpServer::new(factory)` เรียก closure นั้นซ้ำหนึ่งครั้งต่อ worker thread — ถ้า
เผลอเขียนแบบนี้:

```rust
// ผิด: Mutex::new(vec![]) ถูกสร้างใหม่ทุกครั้งที่ worker เรียก closure
// -> แต่ละ worker ได้ state คนละชุด ไม่แชร์กัน
HttpServer::new(|| {
    let state = web::Data::new(AppState { tasks: Mutex::new(vec![]) });
    App::new().app_data(state).service(list_tasks)
})
```

worker แต่ละตัวจะได้ `AppState` คนละชุดที่ไม่เกี่ยวข้องกันเลย — request ที่ไปตกที่ worker คนละตัวกันจะเห็น
ข้อมูลไม่ตรงกัน (เพิ่ม task ผ่าน worker หนึ่ง แต่ query จาก worker อื่นแล้วไม่เจอ) วิธีแก้คือสร้าง state
**นอก** closure ครั้งเดียว แล้ว `.clone()` (ซึ่งแค่เพิ่ม reference count ของ `Arc` ภายใน `web::Data` ไม่ได้
copy ข้อมูลจริง) เข้าไปในแต่ละครั้งที่ closure ถูกเรียก ตรงกับ pattern ที่ตัวอย่างในหัวข้อ 69.7 และ 69.9
เขียนไว้ถูกต้องแล้ว (`let state = web::Data::new(...); HttpServer::new(move || { App::new()
.app_data(state.clone())... })`) — บั๊กแบบนี้อันตรายเป็นพิเศษเพราะทำงาน "ถูก" เวลาทดสอบด้วย `-w 1` (worker
เดียว) แล้วค่อยพังตอน deploy จริงที่ใช้ worker หลายตัวตามค่า default

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** เพิ่ม endpoint `POST /tasks` เข้าไปในโปรเจกต์ Rocket จากหัวข้อ 69.5 ที่รับ JSON body เป็น
   `CreateTaskRequest { title: String }` แล้วสร้าง `Task` ใหม่เพิ่มเข้า `state.tasks` — คืน `(Status::Created,
   Json<Task>)` กลับไป
   - *Hint*: ใช้ `#[post("/tasks", data = "<payload>")]` และ parameter `payload: Json<CreateTaskRequest>`
     — สังเกตว่า `data = "<name>"` ใน attribute คือวิธีที่ Rocket บอกว่า parameter ตัวไหน "กิน" request
     body (เทียบเท่ากับกฎ "extractor ที่กิน body ต้องอยู่ท้ายสุด" ของ Axum ที่ Part 62.5 สอนไว้ — Rocket
     ก็มีกฎคล้ายกันแต่ระบุผ่าน `data = "<...>"` แทนตำแหน่งใน signature)

2. **[กลาง]** เขียน Fairing ง่ายๆสำหรับ Rocket ที่ log ทุก request (method + path) ก่อนเข้า handler โดยใช้
   `on_request` hook แล้วเทียบโค้ดกับ `tower-http`'s `TraceLayer` ที่ Part 65 สอนไว้สำหรับ Axum — เขียน
   สรุปสั้นๆ (3-5 บรรทัด) ว่าความแตกต่างเชิง ergonomics ระหว่างสองวิธีนี้คืออะไร
   - *Hint*: implement trait `rocket::fairing::Fairing` แล้วใช้ `#[rocket::async_trait]` เหนือ `impl`
     block (Rocket ยังต้องใช้ crate `async-trait` แบบ explicit สำหรับ trait method บางตัวที่เป็น async ใน
     trait — ต่างจาก Axum ที่ trait อย่าง `IntoResponse` ไม่ต้องใช้ `async_trait` เพราะออกแบบให้เป็น
     synchronous function ที่คืนค่าได้ทันที)

3. **[ยาก]** พอร์ต Tasks API เวอร์ชันเต็ม (ที่มี `GET /tasks`, `GET /tasks/{id}`, `POST /tasks`,
   `PUT /tasks/{id}`, `DELETE /tasks/{id}` — ถ้า Part 62-66 ของคุณมีเวอร์ชันเต็มอยู่แล้วให้ใช้เวอร์ชันนั้น
   เป็นต้นแบบ) จาก Axum ไปเป็น Rocket ทั้งหมด แล้วนับจำนวนบรรทัดของแต่ละเวอร์ชัน (ไม่รวม comment/blank
   line) เทียบกัน — เขียนสรุปว่าจุดไหนที่ Rocket เขียนสั้นกว่า Axum ชัดเจน และจุดไหนที่ Axum เขียนสั้นกว่า
   Rocket ชัดเจน (ถ้ามี)
   - *Hint*: ให้แยก business logic (การจัดการ `Vec<Task>`) ออกเป็นฟังก์ชันอิสระที่ไม่รู้จักทั้งสองเฟรมเวิร์ก
     ก่อน (ตามหลักการหัวข้อ 69.14) แล้วค่อยเขียนส่วน route/handler ของแต่ละเฟรมเวิร์กแยกเรียกฟังก์ชันนั้น —
     วิธีนี้จะทำให้เห็นชัดว่า "เปลือก" ของแต่ละเฟรมเวิร์กมีขนาดเท่าไหร่จริงๆ แยกจาก business logic ที่เท่ากัน
     เป๊ะทั้งสองฝั่ง

4. **[ยาก/ประยุกต์]** สมมติสถานการณ์: ทีมของคุณมี 6 คน กำลังจะสร้างระบบจองตั๋วภาพยนตร์ใหม่ตั้งแต่ต้น ต้อง
   รองรับทั้ง REST API สำหรับ mobile app และ gRPC service ภายในสำหรับติดต่อกับระบบ inventory ที่มีอยู่แล้ว
   (เขียนด้วย `tonic`) ทีมมีประสบการณ์ Rust ปานกลาง (เขียนมา 6 เดือน) ไม่มีใครเคยใช้เว็บเฟรมเวิร์กตัวไหนมา
   ก่อน — ใช้ decision framework ในหัวข้อ 69.13 เขียน "architecture decision record" สั้นๆ (ไม่เกิน 1 หน้า)
   สรุปว่าทีมนี้ควรเลือกเฟรมเวิร์กไหน พร้อมเหตุผล 3-4 ข้อที่อ้างอิงจากตารางเปรียบเทียบในบทนี้ (ไม่ใช่แค่ความ
   เห็นส่วนตัวลอยๆ)
   - *Hint*: สังเกตคำว่า "gRPC ผ่าน `tonic`" ในโจทย์ให้ดี — ตารางในหัวข้อ 69.13 มีแถวที่พูดถึงสถานการณ์นี้
     ตรงๆ อย่าลืมเหตุผลเรื่อง `tower::Service` ที่ใช้ร่วมกันได้ระหว่าง Axum และ `tonic`

## สรุป

บทนี้ไม่ได้สอนไวยากรณ์ใหม่แบบ Part 62-68 ที่ผ่านมา — เป้าหมายคือการสังเคราะห์ประสบการณ์จริงที่คุณสั่งสมมา
จากการเขียน Axum และ Actix-web เต็มรูปแบบ เข้ากับข้อมูลใหม่จาก Rocket ที่เพิ่งได้ลงมือเขียนและทดสอบจริงเป็น
ครั้งแรกในบทนี้ ประเด็นสำคัญที่สุดที่ควรติดตัวไปคือ:

- **Rocket ใช้งานได้บน stable Rust เต็มรูปแบบแล้วตั้งแต่เวอร์ชัน 0.5** — ข้อมูลเก่าเรื่อง "ต้องใช้
  nightly" ล้าสมัยไปแล้ว และปรัชญา "batteries-included" ของมัน (Shield security header อัตโนมัติ, default
  error catcher, compile-time route checking) ทำให้เหมาะกับสถานการณ์ที่ต้องการ ergonomics สูงสุดและ safety
  net ตอน compile time
- **ความต่างเชิงสถาปัตยกรรมที่แท้จริงระหว่างสามเฟรมเวิร์กคือการพึ่งพา `tower`** — Axum ใช้ `tower`/`hyper`
  ตรงๆ (ยืนยันจาก dependency graph จริง) ส่วน Actix-web และ Rocket มีระบบ `Service`/middleware ของตัวเอง
  ที่ไม่เข้ากับ `tower-http` เลย — นี่คือจุดที่ส่งผลกระทบเชิงปฏิบัติมากที่สุดตอนต้องเลือก middleware หรือ
  ผสมกับ crate อื่นในระบบนิเวศ
- **อย่าเลือกเฟรมเวิร์กจากตัวเลข benchmark เพียงอย่างเดียว** — ในแอปพลิเคชันจริงที่มี database เกี่ยวข้อง
  (ซึ่งคือแอปส่วนใหญ่ที่จะสร้างต่อจากนี้ในหลักสูตร) ความต่างของ throughput ระหว่างเฟรมเวิร์ก HTTP มักถูก
  กลบด้วยเวลาที่รอ I/O เกือบทั้งหมด
- **domain logic ย้ายเฟรมเวิร์กได้ง่าย แต่ routing/extractor/middleware ย้ายไม่ได้เลย** — การแยก layer ให้
  ดีตั้งแต่ต้นคือวิธีป้องกัน vendor lock-in เชิงเฟรมเวิร์กที่ได้ผลจริงที่สุด
- หลักสูตรนี้เลือกเดินหน้าต่อด้วย **Axum** ตั้งแต่ Part 70 เป็นต้นไป ด้วยเหตุผลเรื่องความต่อเนื่องกับ
  Tokio/`tower` ที่สอนมาแล้ว และทิศทางของ ecosystem crate ที่จะใช้ในบทต่อๆไป (SQLx ที่ Part 70,
  WebSocket ที่ Part 77, OpenAPI ผ่าน `utoipa` ที่ Part 85) — เป็นการตัดสินใจเชิงหลักสูตรที่มีเหตุผลรองรับ
  ไม่ใช่การอ้างว่า Axum เหนือกว่าเฟรมเวิร์กอื่นในทุกสถานการณ์

Part ถัดไปจะเริ่มเชื่อมต่อกับฐานข้อมูลจริงเป็นครั้งแรกในหลักสูตรฝั่งเว็บ — **PostgreSQL ผ่าน SQLx** — ซึ่ง
เป็นจุดที่หัวข้อ 69.11 เตือนไว้แล้วว่าจะกลายเป็นปัจจัยด้าน performance ที่สำคัญกว่าเฟรมเวิร์ก HTTP ที่เลือกใช้
เสียอีก

---

**Part ก่อนหน้า:** [Actix-web: Routing, Handlers, Middleware](part-068-actix-web-routing-middleware.md) | **Part ถัดไป:** [Database: เชื่อมต่อ PostgreSQL ด้วย SQLx](part-070-sqlx-postgresql.md)
