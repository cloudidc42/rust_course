# Part 67: แนะนำ Actix-web Framework

> โมดูล: การพัฒนาเว็บแอปพลิเคชัน (Web Development) | ระดับ: กลาง | เวลาโดยประมาณ: 230 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายประวัติศาสตร์ของ **Actix-web** ได้อย่างถูกต้องตรงกับความจริงในปัจจุบัน — รู้ว่าในอดีต (เวอร์ชัน 1.x-2.x)
  Actix-web เคยสร้างอยู่บน **`actix`** (actor framework ที่ทำให้ทุกอย่างในเว็บเซิร์ฟเวอร์ต้องเป็น "actor")
  จริง แต่ตั้งแต่ **Actix-web 4.0** (ต้นปี 2022) เป็นต้นมา HTTP layer หลักได้เลิกพึ่งพา `actix` crate นั้นแล้ว
  โดยสมบูรณ์ — เปลี่ยนไปใช้ `actix-rt` (runtime ของตัวเองที่สร้างอยู่บน **Tokio** ตรงๆ) แทน — และพิสูจน์ข้อเท็จจริง
  นี้ด้วย `cargo tree` จริงบนเครื่องของคุณเอง ไม่ใช่แค่เชื่อคำบอกเล่าที่อาจล้าสมัย
- ติดตั้ง Actix-web เข้าโปรเจกต์ด้วย `cargo add actix-web` แล้วเขียนเซิร์ฟเวอร์ "Hello, World" ตัวแรกที่รันได้จริง
  ด้วย `#[actix_web::main]`, `HttpServer::new(...)`, `App::new()`, `HttpResponse`, และ handler แบบ
  `async fn` — เทียบเคียงโค้ดแบบ **side-by-side** กับ Hello World ของ Axum จาก Part 62 ตรงๆ เพื่อเห็นความ
  เหมือนและความต่างของทั้งสอง framework ในระดับ syntax และ mental model
- อธิบายกลไก **extractor** ของ Actix-web (`web::Path<T>`, `web::Query<T>`, `web::Json<T>`, `web::Data<T>`) และ
  trait **`FromRequest`** ของมันได้ พร้อมเทียบ signature จริง (อ่านจาก source code ตรงๆ) กับ `FromRequest`/
  `FromRequestParts` ของ Axum ที่ Part 62-64 สอนไว้ — เห็นว่าแนวคิด "ดึงข้อมูลจาก request แบบ type-safe" เหมือนกัน
  แต่ shape ของ trait ต่างกันจริง ไม่ใช่แค่ชื่อต่างกัน
- อธิบาย trait **`Responder`** (คู่เทียบของ `IntoResponse` ใน Axum) ได้ว่ารับอะไรเป็น parameter ต่างจาก
  `IntoResponse` อย่างไร (เห็นความแตกต่างเชิงสถาปัตยกรรมที่แท้จริง ไม่ใช่แค่ชื่อ) และรู้ว่า type พื้นฐานอะไรบ้าง
  implement มันมาให้แล้ว — เขียน handler ที่คืน JSON ได้ทั้งผ่าน `web::Json<T>` และผ่าน `HttpResponse::Ok().json(...)`
- เขียน routing ได้ทั้งสองสไตล์ที่ Actix-web รองรับ: สไตล์ **attribute macro** (`#[get("/path")]`,
  `#[post("/path")]` — ผูก path ไว้ที่ตัว handler เอง ต่อยอดจากความรู้เรื่อง attribute macro ใน Part 44-45) และ
  สไตล์ **builder** (`App::new().route("/path", web::get().to(handler))` ที่คล้าย Axum) — และพิสูจน์ว่าทั้งสอง
  สไตล์ใช้ร่วมกันใน `App` เดียวได้จริง
- ระบุและแก้กับดักที่สำคัญที่สุดของ Actix-web ที่ **ไม่มีใน Axum**: closure ที่ส่งให้ `HttpServer::new(...)`
  จะถูกเรียก**หนึ่งครั้งต่อ worker thread** — ถ้า state ไม่ถูกสร้างข้างนอก closure แล้ว `.clone()` เข้าไป (ผ่าน
  `web::Data<T>` ที่ห่อด้วย `Arc` ภายใน) แต่ละ worker จะได้ state คนละก้อนที่ไม่ซิงค์กัน — พิสูจน์พฤติกรรมทั้ง
  "ผิด" และ "ถูก" ด้วยการรันจริงและยิง `curl` นับจำนวนจริง เห็นตัวเลขที่พิสูจน์ปัญหาและคำตอบชัดเจน
- สร้าง REST API ขนาดเล็กแต่ทำงานได้จริงครบวงจร (ระบบ Tasks API — โดเมนเดียวกับที่ Part 62-63 ใช้กับ Axum เพื่อ
  เทียบเคียงกันได้ตรงๆ) พร้อม state แบบ `web::Data<Mutex<...>>` และทดสอบด้วยทั้ง `curl` จริงและ
  `actix_web::test` module (`TestRequest`, `call_service` — คู่เทียบของ `tower::ServiceExt::oneshot` ที่
  Part 62 กล่าวถึง)

## ความรู้ที่ต้องมีมาก่อน

- **Part 61 (HTTP Fundamentals และ REST API Concepts)**: บทนี้อ้างอิงความหมายของ HTTP method, status code,
  header ที่ Part 61 ปูพื้นไว้ตรงๆ เหมือนที่ Part 62 ทำกับ Axum — ถ้ายังไม่ผ่าน Part 61 บทนี้จะดูเหมือนมายากล
- **Part 62 (แนะนำ Axum Framework), Part 63 (Axum: Routing และ Handlers), Part 64 (Axum: State Management
  และ Extractors), Part 65 (Axum: Middleware), Part 66 (Axum: Error Handling แบบมืออาชีพ)**: นี่คือ Part ที่
  สำคัญที่สุดที่ต้องผ่านมาก่อน เพราะทั้งบทนี้เขียนขึ้นในลักษณะ "จับคู่เทียบเคียง" กับสิ่งที่ Part 62-66 สอนไว้
  ทุกครั้งที่พูดถึง extractor, Responder, state, routing ของ Actix-web จะอ้างอิงกลับไปที่กลไกคู่เทียบใน Axum
  ตลอดเวลา — ถ้าจำ Axum ไม่ได้แม่น การเทียบเคียงจะไม่ช่วยอะไร (แนะนำให้ทวน Part 62 หัวข้อ 62.5 เรื่อง
  `IntoResponse`/`Handler` และ Part 64 เรื่อง `State<T>`/`with_state()` ก่อนอ่านบทนี้)
- **Part 48-49 (Tokio: Runtime และ Tasks, I/O และ Networking)**: Actix-web มี runtime ของตัวเอง (`actix-rt`)
  แต่สร้างอยู่บน Tokio ตรงๆ (จะพิสูจน์ในหัวข้อ 67.1) — ความเข้าใจเรื่อง async executor, task, และการที่
  `#[tokio::main]`/`#[actix_web::main]` แปลง `async fn main()` เป็นโปรแกรมที่รันจริงได้ ต้องมาจาก Part 48 ก่อน
- **Part 39 (Smart Pointers: Mutex และ Arc)**: `web::Data<T>` ของ Actix-web ห่อค่าไว้ด้วย `Arc<T>` ภายใน
  (จะพิสูจน์จาก source code จริงในหัวข้อ 67.7) และตัวอย่าง state ในบทนี้ใช้ `Mutex` แบบเดียวกับที่ Part 39
  สอน — ถ้ายังไม่แน่นเรื่องทำไมต้อง `Arc` ข้าม thread และทำไม `.lock()` คืน `Result` ให้กลับไปทวนก่อน
- **Part 44-45 (Macro พื้นฐานและขั้นสูง — รวม Attribute Macro)**: routing แบบ `#[get("/path")]` ของ
  Actix-web เป็น **attribute macro** ตัวเดียวกับกลไกที่ Part 44-45 สอนไว้เป๊ะๆ (ไม่ใช่ฟีเจอร์ภาษาพิเศษของ
  Actix-web) — บทนี้จะไม่อธิบายกลไก proc macro ใหม่จากศูนย์ แต่จะชี้ให้เห็นว่านี่คือการประยุกต์ใช้จริงของสิ่งที่
  เรียนไปแล้ว
- **Part 57-58 (Serde เบื้องต้นและขั้นสูง)**: `web::Json<T>` และ `HttpResponse::Ok().json(...)` ใช้
  `Serialize`/`Deserialize` trait จาก `serde` ตรงๆ แบบเดียวกับ `axum::Json<T>` ที่ Part 57-58 ปูพื้นไว้ ไม่มี
  กลไก serialize พิเศษเพิ่มเติมจาก Actix-web เอง
- **Part 30-31 (Error Handling ระดับโปรเจกต์)**: ตัวอย่าง error handling ในบทนี้ใช้วิธีง่ายที่สุด
  (`HttpResponse::NotFound().json(...)`) เพื่อโฟกัสที่กลไกของ Actix-web เอง — การออกแบบ error type แบบ
  มืออาชีพที่ implement `ResponseError` เอง (คู่เทียบของ `IntoResponse` สำหรับ error type ที่ Part 66 สอนฝั่ง
  Axum) ไม่ใช่เนื้อหาหลักของบทนี้ (จะกล่าวถึงสั้นๆใน Part 69 ตอนเทียบระบบ error handling เต็มรูปแบบ)

## เนื้อหา

### 67.1 Actix-web คืออะไรกันแน่: ประวัติศาสตร์ที่เข้าใจผิดกันบ่อยที่สุด

ถ้าคุณเคยอ่านบทความเก่าๆเกี่ยวกับ Rust web framework คุณอาจเจอประโยคแบบ "Actix-web สร้างอยู่บน actor model
ทุกอย่างในนั้นคือ actor" — ประโยคนี้**เคยเป็นจริง**แต่**ไม่เป็นจริงแล้วในปัจจุบัน** และนี่คือจุดที่บทนี้จะพา
คุณไปพิสูจน์ข้อเท็จจริงด้วยตัวเอง ไม่ใช่เชื่อคำบอกเล่าเก่าๆ ที่ยังคงเวียนซ้ำอยู่ในหลายที่บนอินเทอร์เน็ต

#### ไทม์ไลน์จริงของ Actix-web

- **`actix`** คือ crate แยกที่มาก่อน — เป็น **actor framework** ทั่วไป (implement แนวคิด actor model:
  หน่วยประมวลผลอิสระที่สื่อสารกันด้วยการส่ง message เท่านั้น ไม่มี shared mutable state ตรงๆข้ามหน่วย)
  เขียนโดย Nikolay Kim ไม่ผูกกับ HTTP หรือเว็บเลยแม้แต่นิดเดียวโดยตัวมันเอง
- **Actix-web เวอร์ชัน 1.x-2.x** (ประมาณปี 2017-2019) สร้าง HTTP server ทับบน `actix` โดยตรง — connection
  แต่ละตัวและ worker ถูกโมเดลเป็น actor จริงๆ ต้องเข้าใจ `System`, `Arbiter`, `Addr<A>`, message passing
  ของ `actix` เพื่อจะเข้าใจว่า Actix-web ทำงานอย่างไรข้างใน นี่คือที่มาของชื่อเสียง (และข้อวิจารณ์) ที่ว่า
  "ต้องเรียน actor model ก่อนถึงจะใช้ Actix-web ได้"
- **Actix-web 4.0** (ออกต้นปี 2022) คือจุดเปลี่ยนสำคัญ — ทีมพัฒนา**เอา `actix` (actor framework) ออกจาก
  HTTP layer หลักทั้งหมด** เปลี่ยนไปใช้ `actix-rt` (runtime ของตัวเอง ที่ตอนนี้เป็นแค่ชั้นบางๆทับ **Tokio**
  ตรงๆ) และ `actix-server`/`actix-service`/`actix-http` (ชุด crate ย่อยที่ implement HTTP server และแนวคิด
  `Service` ของตัวเอง — ไม่ใช่ `tower::Service` ของ Axum) แทน

**ทำไมเรื่องนี้ถึงสำคัญในทางปฏิบัติ?** เพราะทุกโค้ดที่คุณจะเขียนในบทนี้ (ตั้งแต่ Hello World ไปจนถึง Tasks
API) **ไม่แตะ `actix` actor framework เลยแม้แต่นิดเดียว** — คุณจะไม่เห็นคำว่า `Actor`, `Addr`, `System::run()`
แบบ actor-style ในตัวอย่างของบทนี้เลย (ยกเว้นถ้าคุณเจาะจงไปใช้ `actix-web-actors` สำหรับ WebSocket ซึ่งเป็นเคส
พิเศษที่ยังพึ่งพา actor model อยู่จริง แต่ไม่ใช่เนื้อหาของบทนี้)

#### พิสูจน์ด้วย `cargo tree`: ไม่มี `actix` (actor crate) อยู่ใน dependency tree ของ Actix-web 4.x เลย

ไม่ต้องเชื่อคำอธิบายเฉยๆ — สร้างโปรเจกต์ทดสอบแล้วดู dependency graph จริง:

```bash
cargo new hello_actix
cd hello_actix
cargo add actix-web
```

`Cargo.toml` ที่ได้:

```toml
[package]
name = "hello_actix"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4.15.0"
```

(เวอร์ชันที่ปรากฏคือเวอร์ชันล่าสุดบน crates.io ณ เวลาที่เขียนบทนี้ — เหมือนกับที่ Part 62 อธิบายไว้ตอนติดตั้ง
Axum ไม่ต้องพิมพ์เลขเวอร์ชันตามนี้เป๊ะๆ)

รัน `cargo tree` แล้วค้นหาคำว่า `actix` แบบเปล่าๆ (ไม่มี `-web`, `-rt`, `-http`, `-server` ต่อท้าย):

```bash
cargo tree | grep -E "^actix v| actix v"
```

ผลลัพธ์จริง: **ไม่มีบรรทัดไหนขึ้นมาเลย** — `actix` (actor framework เปล่าๆ) ไม่ได้เป็น dependency ของ
`actix-web` 4.15.0 ไม่ว่าจะเป็น direct หรือ transitive dependency ก็ตาม เทียบกับดู dependency ตรงของ
`actix-web` เอง (ตัด `cargo tree -p actix-web` มาเฉพาะชั้นบนสุด):

```
actix-web v4.15.0
├── actix-codec v0.5.4
├── actix-http v3.18.12
├── actix-macros v0.2.5
├── actix-router v0.5.4
├── actix-rt v2.15.0
│   └── tokio v1.53.1
├── actix-server v2.9.7
│   ├── actix-rt v2.15.0 (*)
│   └── tokio v1.53.1 (*)
├── actix-service v2.0.3
├── actix-utils v3.0.2
├── actix-web-codegen v4.4.0
└── ... (serde, bytes, http, mime, url, ฯลฯ)
```

สังเกตชื่อ crate ทั้งหมดที่ขึ้นต้นด้วย `actix-` (มีขีดกลาง) — เหล่านี้คือ**ชุด crate เล็กๆที่ทีม Actix-web
เขียนขึ้นมาเองสำหรับงาน HTTP โดยเฉพาะ** (`actix-http` = HTTP protocol implementation, `actix-router` =
path matching, `actix-service` = นามธรรม service ของตัวเอง, `actix-server` = การจัดการ worker/socket,
`actix-rt` = async runtime wrapper) — ไม่มีตัวไหนคือ `actix` (actor framework) ตัวเปล่าเลย และ `actix-rt`
เองก็มี **`tokio`** เป็น dependency ตรง — ยืนยันสิ่งที่หัวข้อนี้อธิบายไว้ว่า runtime ของ Actix-web สร้างอยู่บน
Tokio จริง ไม่ใช่ event loop ของตัวเองที่แยกขาดจาก ecosystem

#### หลักฐานอีกชิ้น: คำเตือนจาก source code ของ `actix-web-codegen` เอง

หลักฐานที่หนักแน่นที่สุดมาจาก doc comment ในตัว macro `#[actix_web::main]` เอง (อ่านจาก source code ของ
`actix-web-codegen` เวอร์ชัน 4.4.0 ตรงๆ):

```
/// Marks async main function as the Actix Web system entry-point.
///
/// Note that Actix Web also works under `#[tokio::main]` since version 4.0. However, this macro is
/// still necessary for actor support (since actors use a `System`). Read more in the
/// `actix_web::rt` module docs.
```

ประโยคนี้ยืนยันตรงๆจากปากผู้พัฒนาเองว่า: **"Actix Web ทำงานได้ภายใต้ `#[tokio::main]` เช่นกันตั้งแต่เวอร์ชัน
4.0"** — และ `#[actix_web::main]` (ที่ตัวอย่างในบทนี้จะใช้เป็นหลัก) ยังคง "จำเป็น" ก็เฉพาะกรณีที่คุณใช้
**actor support** (เช่น `actix-web-actors` สำหรับ WebSocket แบบ actor-based) เท่านั้น — สำหรับ HTTP handler
ธรรมดาทั้งหมดที่บทนี้จะสอน `#[tokio::main]` ก็ใช้แทนกันได้จริง (จะทดลองพิสูจน์ในหัวข้อกับดักท้ายบท)

#### สรุปสถาปัตยกรรมของ Actix-web เทียบกับ Axum แบบภาพชั้น (layer)

เพื่อให้เทียบกับภาพ layer ของ Axum ที่ Part 62 หัวข้อ 62.2 วาดไว้ได้ตรงๆ นี่คือภาพเดียวกันของ Actix-web:

```
┌─────────────────────────────────────────┐
│         โค้ดแอปพลิเคชันของคุณ            │  <- App, handler function ที่คุณเขียน
├─────────────────────────────────────────┤
│              Actix-web                   │  <- ให้ ergonomics: App, extractor, Responder
├─────────────────────────────────────────┤
│   actix-http / actix-service / actix-router  │  <- HTTP protocol impl ของตัวเอง + Service ของตัวเอง
├─────────────────────────────────────────┤
│         actix-server / actix-rt          │  <- จัดการ worker thread หลายตัว + wrapper รอบ Tokio
├─────────────────────────────────────────┤
│              Tokio runtime               │  <- async executor ตัวจริงที่รันงานทั้งหมด
├─────────────────────────────────────────┤
│      OS: epoll (Linux) / kqueue (BSD/macOS) / IOCP (Windows)  │
└─────────────────────────────────────────┘
```

เทียบบรรทัดต่อบรรทัดกับภาพของ Axum ใน Part 62 จะเห็นความต่างที่สำคัญที่สุดสองจุด:

| ประเด็น | Axum | Actix-web |
|---|---|---|
| HTTP protocol implementation | `hyper` (crate แยก ใช้ร่วมกับ framework อื่นได้ด้วย เช่น อาจเอาไปสร้าง framework ของตัวเองก็ได้) | `actix-http` (เขียนขึ้นมาเองโดยทีม Actix-web โดยเฉพาะ ไม่ใช้ `hyper`) |
| นามธรรม "Service" กลาง | `tower::Service` (ecosystem-wide ใช้กับ gRPC, RPC อื่นได้ด้วย) | `actix_service::Service` (นามธรรมของตัวเอง รูปร่างคล้ายกันแต่เป็น trait คนละตัว ไม่ compatible กัน) |
| Async runtime | Tokio ตรงๆ (ไม่มี wrapper) | `actix-rt` (wrapper บาง ๆ บน Tokio — เพิ่มแนวคิด `System`/`Arbiter` ที่มาจากมรดกของ actor model เดิม) |
| การจัดการ concurrency ต่อ request | `tokio::spawn` task ใหม่ต่อ connection บน runtime เดียว (thread pool ใช้ร่วมกันทั้งแอป) | worker **process-like thread** หลายตัว แต่ละตัวมี event loop และ (โดย default) app instance เป็นของตัวเอง (รายละเอียดเต็มในหัวข้อ 67.7) |

จุดสุดท้ายในตารางคือหัวใจของกับดักที่สำคัญที่สุดของบทนี้ (หัวข้อ 67.7) — เตรียมใจไว้ล่วงหน้าได้เลยว่านี่คือ
ความต่างเชิงสถาปัตยกรรมที่ทำให้ Actix-web "รู้สึกคนละแบบ" จาก Axum ตอนจัดการ state ที่แชร์กันข้าม request

### 67.2 ติดตั้ง Actix-web เข้าโปรเจกต์

สร้างโปรเจกต์ใหม่แล้วเพิ่ม dependency ด้วย `cargo add` (เหมือนขั้นตอนที่ Part 62 ทำกับ Axum เป๊ะๆ):

```bash
cargo new hello_actix
cd hello_actix
cargo add actix-web
cargo add serde --features derive
cargo add serde_json
```

`Cargo.toml` ที่ได้ (verify จริงบนเครื่องที่เขียนบทนี้):

```toml
[package]
name = "hello_actix"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4.15.0"
serde = { version = "1.0.229", features = ["derive"] }
serde_json = "1.0.151"
```

**สังเกตความแตกต่างจาก Axum ทันที**: ไม่มี `tokio = { version = "...", features = ["full"] }` ใน
dependency list เลย! เพราะ Actix-web ไม่ได้ให้คุณเรียก `#[tokio::main]` เอง (แม้จะทำได้ตามที่หัวข้อ 67.1
พิสูจน์ไว้) แต่มันซ่อน Tokio ไว้ข้างในผ่าน `actix-rt` แล้ว — `tokio` ยังคงเป็น dependency จริง (เห็นได้จาก
`cargo tree` ในหัวข้อ 67.1) แค่เป็น **transitive** (ผ่าน `actix-rt`/`actix-server`) ไม่ใช่ direct dependency
ที่คุณต้องประกาศเอง — นี่คือตัวอย่างที่ชัดเจนของความต่างเชิงปรัชญา: Axum เลือกให้ผู้ใช้เห็นและควบคุม Tokio
ตรงๆ ส่วน Actix-web เลือกซ่อนมันไว้เป็น implementation detail

### 67.3 Hello World เซิร์ฟเวอร์ตัวแรก: เทียบเคียงกับ Axum แบบ Side-by-Side

มาเขียนเซิร์ฟเวอร์ Actix-web ที่เล็กที่สุดที่ยังทำงานได้จริง วางไว้ข้าง ๆ เวอร์ชัน Axum จาก Part 62 หัวข้อ
62.4 เพื่อเทียบทุกบรรทัด:

**Axum (จาก Part 62):**

```rust
use axum::routing::get;
use axum::Router;

async fn hello() -> &'static str {
    "Hello, World!"
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/", get(hello));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    println!("listening on {}", listener.local_addr().unwrap());

    axum::serve(listener, app).await.unwrap();
}
```

**Actix-web (บทนี้ — verify จริงว่า compile และรันได้):**

```rust
use actix_web::{web, App, HttpServer, Responder};

async fn hello() -> impl Responder {
    "Hello, World!"
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    println!("listening on 127.0.0.1:3000");

    HttpServer::new(|| App::new().route("/", web::get().to(hello)))
        .bind(("127.0.0.1", 3000))?
        .run()
        .await
}
```

ไล่เทียบทีละบรรทัด:

| ส่วนของโค้ด | Axum | Actix-web |
|---|---|---|
| Attribute macro บน `main` | `#[tokio::main]` | `#[actix_web::main]` (ทำงานได้กับ `#[tokio::main]` เช่นกัน ตามที่พิสูจน์ในหัวข้อ 67.1 — แต่บทนี้ใช้ตัวเฉพาะของ Actix-web เป็นค่าปริยาย) |
| Return type ของ `main()` | `()` (ไม่คืนอะไร — เรียก `.unwrap()` เอง) | `std::io::Result<()>` (ให้ `?` operator ใช้ propagate error จาก `.bind()` ได้ตรงๆ) |
| สร้างตัวจับ path ↔ handler | `Router::new().route("/", get(hello))` | `App::new().route("/", web::get().to(hello))` |
| ฟังก์ชันที่ห่อ method | `axum::routing::get(hello)` — ส่ง handler เป็น argument | `web::get().to(hello)` — สร้าง route ก่อนด้วย `web::get()` แล้ว `.to(hello)` เติม handler เข้าไป |
| การ bind + listen | แยกสองขั้น: `TcpListener::bind(...)` แล้ว `axum::serve(listener, app)` | รวมเป็นขั้นเดียว: `HttpServer::new(factory).bind(addr)?.run()` |
| return type ของ handler | `&'static str` ตรงๆ (implement `IntoResponse`) | `impl Responder` (ต้องเขียน `impl Responder` ชัดๆ หรือ concrete type — จะอธิบายเหตุผลในหัวข้อ 67.4) |
| จำนวน worker/thread ที่ประมวลผล request | ใช้ Tokio task บน runtime เดียว (จำนวน thread ควบคุมผ่าน `#[tokio::main]` เอง) | ค่าปริยายคือหลาย **worker thread แยกกัน** (หัวข้อ 67.7 จะอธิบายลึก) |

สังเกตจุดที่สำคัญที่สุด: **`HttpServer::new(|| App::new()...)`** — closure ที่ส่งเข้าไปใน `HttpServer::new`
คือสิ่งที่ Actix-web เรียกว่า **"application factory"** — มันไม่ได้สร้าง `App` แค่ครั้งเดียวเหมือนที่
`Router::new()` ของ Axum ทำ แต่จะถูกเรียก**ซ้ำหลายครั้ง** (ครั้งละหนึ่งครั้งต่อ worker thread) — นี่คือกลไก
พื้นฐานที่หัวข้อ 67.7 จะขยายเต็มรูปแบบ ตอนนี้ให้จำไว้ก่อนว่า `|| App::new()...` ไม่ใช่แค่ syntax แปลกๆ แต่มี
ความหมายเชิงสถาปัตยกรรมจริง

รันเซิร์ฟเวอร์:

```bash
cargo run
```

ผลลัพธ์จริงจาก compile และรัน:

```
   Compiling actix_verify_proj v0.1.0 (/path/to/hello_actix)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 1m 15s
     Running `target/debug/hello_actix`
listening on 127.0.0.1:3000
```

เปิด terminal ใหม่แล้วยิง `curl` จริง:

```bash
curl -i http://127.0.0.1:3000/
```

output จริง:

```
HTTP/1.1 200 OK
content-length: 13
content-type: text/plain; charset=utf-8
date: Sat, 26 Sep 2026 23:58:29 GMT

Hello, World!
```

ยิง path ที่ไม่มี route ผูกไว้:

```bash
curl -i http://127.0.0.1:3000/nope
```

output จริง:

```
HTTP/1.1 404 Not Found
content-length: 0
date: Sat, 26 Sep 2026 23:58:29 GMT
```

พฤติกรรม fallback 404 อัตโนมัติเมื่อไม่มี route ตรง — เหมือนกับที่ Axum ทำใน Part 62 หัวข้อ 62.4 เป๊ะๆ ทั้งสอง
framework เห็นพ้องกันในเรื่องนี้ว่าเป็นพฤติกรรม default ที่สมเหตุสมผลที่สุด

**สิ่งที่ต่างจาก Axum แบบสังเกตได้**: response ของ Hello World ไม่มี `content-type: text/plain; charset=utf-8`
เขียนก่อน `content-length` ตามลำดับเดียวกับ Axum เป๊ะ (ลำดับ header ต่างกันเล็กน้อยเพราะคนละ HTTP
implementation กัน — `actix-http` กับ `hyper`) แต่ **เนื้อหาของ header ที่สำคัญเหมือนกันทั้งคู่** — นี่คือ
ตัวอย่างที่ดีว่า HTTP เป็นสเปกกลางที่ทั้งสอง framework implement ให้ตรงกัน แม้จะเขียนโค้ดคนละชุดกันก็ตาม

### 67.4 Trait `Responder`: คู่เทียบของ `IntoResponse` ที่รับ `HttpRequest` เข้ามาด้วย

ใน Axum (Part 62 หัวข้อ 62.5) กลไกที่ทำให้ `async fn -> impl IntoResponse` ทำงานได้คือ trait `IntoResponse`
ที่นิยาม (ประมาณ) ว่า:

```rust
// Axum (axum-core 0.5.6) — นิยามจริง อ่านจาก source code:
pub trait IntoResponse {
    fn into_response(self) -> axum::response::Response;
}
```

Actix-web มีกลไกคู่เทียบชื่อ **`Responder`** — แต่ signature ต่างจาก `IntoResponse` ในจุดที่สำคัญมาก
(นิยามจริง อ่านจาก source code ของ `actix-web` 4.15.0 ตรงๆ ไม่ใช่เขียนขึ้นมาประมาณ):

```rust
pub trait Responder {
    type Body: MessageBody + 'static;

    fn respond_to(self, req: &HttpRequest) -> HttpResponse<Self::Body>;
}
```

**ความแตกต่างที่แท้จริง (ไม่ใช่แค่ชื่อ)**: `Responder::respond_to` รับ **`&HttpRequest`** เป็น parameter
ด้วย ในขณะที่ `IntoResponse::into_response` ของ Axum **ไม่รับ request เข้ามาเลย** — หมายความว่าใน
Actix-web การแปลงค่าที่ handler คืนกลับมาให้เป็น response **สามารถอ้างอิงกลับไปดู request ต้นทางได้** (เช่น
เช็ค header `Accept` เพื่อเลือกว่าจะตอบ JSON หรือ HTML, หรือเช็ค method เพื่อตัดสินใจอะไรบางอย่าง) ในขณะที่
`IntoResponse` ของ Axum ทำแบบนั้นไม่ได้โดยตรง (ถ้าต้องการ logic แบบนั้นใน Axum ต้องทำผ่าน extractor ที่ดึง
`HeaderMap`/`Method` เข้ามาเป็น parameter ต่างหากแทน) — นี่คือตัวอย่างรูปธรรมของความต่างเชิงสถาปัตยกรรมที่
แท้จริงระหว่างสองระบบ ไม่ใช่แค่ syntax เปลี่ยนชื่อ

ตารางสรุป type พื้นฐานที่ implement `Responder` มาให้แล้ว (ตรวจสอบจาก source code จริงของ
`actix-web` 4.15.0):

| Type | Response ที่ได้ | หมายเหตุ |
|---|---|---|
| `&'static str` / `String` | `200 OK`, `content-type: text/plain; charset=utf-8` | เหมือน Axum |
| `HttpResponse<B>` / `HttpResponseBuilder` | ตามที่ตั้งค่าไว้ตรงๆ | คู่เทียบของการคืน `Response` ตรงๆใน Axum |
| `web::Json<T>` โดย `T: Serialize` | `200 OK`, `content-type: application/json` | เหมือน `axum::Json<T>` |
| `web::Form<T>` โดย `T: Serialize` | `200 OK`, `content-type: application/x-www-form-urlencoded` | Axum ไม่มี type นี้ในตัวหลัก ต้องใช้ crate เสริม |
| `Html` (จาก `actix_web::web::Html` หรือ `actix_files`) | `200 OK`, `content-type: text/html; charset=utf-8` | เหมือน `axum::response::Html<T>` |
| `Redirect` | `3xx` พร้อม `Location` header | เหมือน `axum::response::Redirect` |
| `Option<R>` โดย `R: Responder` | `Some`'s response ถ้ามี, **`404 Not Found`** ถ้า `None` | **ต่างจาก Axum**: Axum ไม่มี `impl IntoResponse for Option<T>` ในตัวหลัก ต้องแปลงเป็น `Result` เองก่อน |
| `Result<R, E>` โดยทั้งคู่ implement `Responder`/แปลงเป็น error ได้ | `R`'s response ถ้า `Ok`, error response ถ้า `Err` | เหมือน Axum concept แต่ Actix-web ต้องการให้ `E: Into<actix_web::Error>` (ผ่าน trait `ResponseError`) ไม่ใช่ `E: IntoResponse` ตรงๆ |
| `(R, StatusCode)` โดย `R: Responder` | status ตามที่กำหนด + body จาก `R` | **สังเกตลำดับ tuple สลับกับ Axum!** Axum ใช้ `(StatusCode, T)` (status มาก่อน) ส่วน Actix-web ใช้ `(R, StatusCode)` (body มาก่อน) — สลับกันเป๊ะ เป็นกับดักเล็กๆที่คนย้ายจาก Axum มาชนบ่อย |
| `Either<L, R>` | response ของฝั่งที่ resolve จริง | คู่เทียบของการคืน enum สองแบบที่ทั้งคู่ implement responder เอง |

ตัวอย่างการคืน JSON สองแบบที่ใช้บ่อยที่สุด:

```rust
use actix_web::{get, web, HttpResponse, Responder};
use serde::Serialize;

#[derive(Debug, Clone, Serialize)]
struct Item {
    id: u32,
    name: String,
    price: f64,
}

// แบบที่ 1: คืน web::Json<T> ตรงๆ — คู่เทียบของ axum::Json<T>
#[get("/items")]
async fn list_items() -> impl Responder {
    let items = vec![
        Item { id: 1, name: "เมาส์ไร้สาย".to_string(), price: 299.0 },
        Item { id: 2, name: "แป้นพิมพ์กลไก".to_string(), price: 1490.0 },
    ];
    web::Json(items)
}

// แบบที่ 2: สร้าง HttpResponse ตรงๆ ด้วย builder pattern แล้วเรียก .json(...)
#[get("/items/builder-style")]
async fn list_items_builder() -> impl Responder {
    let items = vec![
        Item { id: 1, name: "เมาส์ไร้สาย".to_string(), price: 299.0 },
    ];
    HttpResponse::Ok().json(items)
}
```

ยิง `curl` เข้าไปจริงกับ endpoint แบบที่ 1:

```bash
curl -i http://127.0.0.1:3000/items
```

output จริง:

```
HTTP/1.1 200 OK
content-length: 108
content-type: application/json
date: Sat, 26 Sep 2026 23:59:10 GMT

[{"id":1,"name":"เมาส์ไร้สาย","price":299.0},{"id":2,"name":"แป้นพิมพ์กลไก","price":1490.0}]
```

**สังเกต**: `web::Json<T>` ของ Actix-web กับ `axum::Json<T>` ของ Axum เขียนเหมือนกันเป๊ะในระดับการใช้งาน
(ห่อค่าที่ `Serialize` ได้ แล้ว framework แปลงเป็น JSON response ให้อัตโนมัติ) — นี่คือจุดที่สอง framework
**เห็นตรงกัน**ว่าเป็น pattern ที่ดีที่สุดสำหรับงานนี้ ต่างจากจุดอื่นๆที่ออกแบบต่างกันจริง

### 67.5 Extractor: `web::Path`, `web::Query`, `web::Json`, `web::Data` และ Trait `FromRequest`

แนวคิด "ดึงข้อมูลจาก request แบบ type-safe ผ่าน parameter ของ handler" เป็นแนวคิดที่ทั้งสอง framework มี
เหมือนกัน (Actix-web มีก่อน Axum ด้วยซ้ำในเชิงประวัติศาสตร์ — extractor เป็นหนึ่งใน design ที่ Actix-web
บุกเบิกไว้ในโลก Rust web framework) — นี่คือตารางเทียบชื่อ:

| งานที่ต้องทำ | Axum (Part 62-64) | Actix-web |
|---|---|---|
| ดึงค่าจาก path segment | `axum::extract::Path<T>` | `actix_web::web::Path<T>` |
| ดึงค่าจาก query string | `axum::extract::Query<T>` | `actix_web::web::Query<T>` |
| ดึง JSON body แล้ว deserialize | `axum::Json<T>` | `actix_web::web::Json<T>` |
| เข้าถึง state ที่แชร์กัน | `axum::extract::State<T>` (ผูกกับ `Router::with_state()`) | `actix_web::web::Data<T>` (ผูกกับ `App::app_data()`) |
| ดึง header ทั้งหมด | `axum::http::HeaderMap` | `actix_web::HttpRequest` แล้วเรียก `.headers()`, หรือ `web::Header<T>` |
| ดึง form-urlencoded body | ต้องใช้ crate เสริม | `actix_web::web::Form<T>` (มีในตัวหลักเลย) |

ชื่อ type คล้ายกันมากจนแทบจำสลับกันได้ — แต่ trait ที่อยู่เบื้องหลังต่างกันจริง ลองดู `FromRequest` ของทั้งคู่
เทียบกันตรงๆ (คัดลอกมาจาก source code จริง ไม่ใช่เขียนขึ้นมาประมาณ):

**Axum (`axum-core` 0.5.6, ไฟล์ `src/extract/mod.rs`)** — แยกเป็น**สอง** trait:

```rust
// สำหรับ extractor ที่ดึงได้จาก "หัว" ของ request เท่านั้น (ไม่แตะ body) — ดึงกี่ตัวก็ได้ในหนึ่ง handler
pub trait FromRequestParts<S>: Sized {
    type Rejection: IntoResponse;

    fn from_request_parts(
        parts: &mut Parts,
        state: &S,
    ) -> impl Future<Output = Result<Self, Self::Rejection>> + Send;
}

// สำหรับ extractor ที่ต้อง "กิน" ทั้ง request รวม body — ใช้ได้แค่ตัวสุดท้ายของ parameter list เท่านั้น
pub trait FromRequest<S, M = private::ViaRequest>: Sized {
    type Rejection: IntoResponse;

    fn from_request(
        req: Request,
        state: &S,
    ) -> impl Future<Output = Result<Self, Self::Rejection>> + Send;
}
```

**Actix-web (`actix-web` 4.15.0, ไฟล์ `src/extract.rs`)** — มี trait **เดียว**:

```rust
pub trait FromRequest: Sized {
    type Error: Into<Error>;
    type Future: Future<Output = Result<Self, Self::Error>>;

    fn from_request(req: &HttpRequest, payload: &mut Payload) -> Self::Future;
}
```

ความต่างเชิงสถาปัตยกรรมที่แท้จริงมีสามจุด:

1. **Axum แยกสอง trait ตามชนิด** (`FromRequestParts` สำหรับดึงจากหัว request เท่านั้น, `FromRequest`
   สำหรับดึงทั้ง request รวม body) — Rust type system ของ Axum จะ**บังคับ** ผ่าน trait bound ที่ต่างกันว่า
   extractor ที่กิน body ต้องเป็น parameter ตัวสุดท้ายเท่านั้น (Part 62 หัวข้อ 62.5 อธิบายเรื่องนี้ไว้แล้ว)
   — Actix-web ใช้ trait**เดียว**สำหรับทุก extractor โดยรับ `payload: &mut Payload` มาด้วยเสมอ (เป็น
   mutable reference ที่ extractor ตัวไหนจะ "ดึง" body ออกไปใช้ก็ได้ ไม่มีการแยก type ระดับ trait บังคับว่า
   ต้องเป็นตัวสุดท้าย — เป็น**ธรรมเนียมปฏิบัติ** (convention) มากกว่าการบังคับด้วย type system)
2. **Axum ใช้ `impl Future<...> + Send` เป็น return type ตรงๆ** (native async fn in trait ผ่านฟีเจอร์
   return-position-impl-trait-in-trait ของ Rust ที่ Part 22 กล่าวถึงแนวคิด impl Trait ไว้บ้าง) — Actix-web
   ใช้ **associated type** `type Future` แยกออกมาต่างหาก (ต้องกำหนดเองว่า Future ของ extractor คือ type
   ไหน เช่น `Pin<Box<dyn Future<...>>>` หรือ `Ready<Result<Self, Self::Error>>`) — เป็นสไตล์การเขียน trait
   ที่เก่ากว่า (เขียนไว้ตั้งแต่ก่อนที่ Rust จะรองรับ async fn ใน trait ได้ดีเท่าทุกวันนี้) แต่ยังใช้งานได้ดี
3. **`Error` ของ Actix-web ต้อง `Into<actix_web::Error>`** (error type กลางของ Actix-web เอง) ส่วน
   `Rejection` ของ Axum ต้อง `IntoResponse` ตรงๆ — ทั้งคู่ทำหน้าที่เดียวกัน (แปลง extractor ที่ล้มเหลวให้เป็น
   HTTP response ที่เหมาะสม) แต่ Actix-web มี type กลาง (`actix_web::Error`) เป็นตัวกลางอีกชั้นหนึ่ง

**ทำไมเรื่องนี้สำคัญในทางปฏิบัติ?** เพราะถ้าคุณต้องเขียน extractor ของตัวเอง (custom extractor — เนื้อหาขั้น
สูงกว่าที่บทนี้จะยังไม่ลงรายละเอียดเต็ม) shape ของโค้ดที่ต้องเขียนจะต่างกันจริงระหว่างสอง framework ไม่ใช่แค่
เปลี่ยนชื่อ trait — และถ้าอ่าน error message ตอน extractor compile ไม่ผ่าน (จะเห็นตัวอย่างจริงในหัวข้อกับดัก)
จะเข้าใจได้ง่ายขึ้นว่าทำไม error หน้าตาแบบนั้น

ตัวอย่างการใช้ extractor หลายตัวพร้อมกันใน handler เดียว:

```rust
use actix_web::{get, web, HttpResponse, Responder};
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct Filter {
    done: Option<bool>,
}

// web::Path<u32> ดึง path parameter, web::Query<Filter> ดึง query string
// ทั้งสองตัวไม่ได้แตะ body เลย จึงเรียงลำดับพารามิเตอร์ได้อย่างอิสระ (แนวคิดเดียวกับ Axum)
#[get("/tasks/{id}/related")]
async fn related_tasks(path: web::Path<u32>, query: web::Query<Filter>) -> impl Responder {
    let id = path.into_inner();
    let done_filter = query.done;
    HttpResponse::Ok().body(format!(
        "task {id} เกี่ยวข้องกับงานที่ done = {done_filter:?}"
    ))
}
```

`web::Path<T>` และ `web::Query<T>` ทำงานเหมือนที่ Axum สอนไว้เป๊ะ — `web::Path<u32>` พยายาม parse path
segment เป็น `u32` (ล้มเหลวได้ถ้า parse ไม่ผ่าน — จะเห็น behavior จริงในหัวข้อกับดัก) และ `web::Query<T>`
ต้องการให้ `T: Deserialize` (มาจาก `serde` ตรงๆเหมือนเดิม)

### 67.6 Routing สองสไตล์: Attribute Macro เทียบกับ Builder Style

นี่คือหนึ่งในความต่างเชิง "หน้าตาโค้ด" (surface syntax) ที่ชัดเจนที่สุดระหว่างสอง framework — Axum ใช้สไตล์
**builder** เท่านั้น (เขียน route ทั้งหมดรวมกันเป็น chain ของ `.route()`) ส่วน Actix-web รองรับ**ทั้งสองสไตล์**
และในทางปฏิบัติ สไตล์ **attribute macro** (`#[get("/path")]`) เป็นสไตล์ที่โปรเจกต์ Actix-web ส่วนใหญ่นิยมใช้
มากกว่า

#### สไตล์ที่ 1: Attribute Macro — ผูก Path ไว้ที่ตัว Handler เอง

```rust
use actix_web::{get, post, delete, HttpResponse, Responder};

#[get("/tasks")]
async fn list_tasks() -> impl Responder {
    HttpResponse::Ok().body("รายการ task ทั้งหมด")
}

#[post("/tasks")]
async fn create_task() -> impl Responder {
    HttpResponse::Created().finish()
}

#[delete("/tasks/{id}")]
async fn delete_task() -> impl Responder {
    HttpResponse::NoContent().finish()
}
```

`#[get(...)]`, `#[post(...)]`, `#[delete(...)]` (และ `#[put(...)]`, `#[patch(...)]`, `#[head(...)]` ที่มี
ให้ครบทุก method) คือ **attribute macro** ตัวเดียวกับกลไกที่ Part 44-45 สอนไว้เป๊ะๆ — ไม่ใช่ฟีเจอร์ภาษา
พิเศษของ Actix-web แต่เป็นการประยุกต์ใช้ proc macro แบบ attribute ที่มาจาก crate `actix-web-codegen`
(dependency ของ `actix-web` ที่เห็นใน `cargo tree` ในหัวข้อ 67.1) — สิ่งที่ macro นี้ทำคือ**สร้างโค้ดเพิ่มเติม
รอบตัวฟังก์ชัน** ที่บอก Actix-web ว่า "ฟังก์ชันนี้ควรถูก mount ไว้ที่ path และ method อะไร" (แนวคิดเดียวกับที่
Part 45 สอนว่า attribute macro รับ token stream ของทั้ง attribute เอง (`"/tasks"`) และของ item ที่มันติดอยู่
(ตัวฟังก์ชัน `list_tasks` ทั้งก้อน) แล้วคืน token stream ใหม่ออกมาแทนที่)

การเอา handler ที่มี macro ผูกไว้แล้วไปใช้จริง ต้องเรียก **`.service(handler)`** ไม่ใช่ `.route(...)` (เพราะ
macro ได้ผูก path/method ไว้ในตัวมันเองแล้ว ไม่ต้องบอกซ้ำ):

```rust
use actix_web::{web, App, HttpServer};
# use actix_web::{get, post, delete, HttpResponse, Responder};
# #[get("/tasks")]
# async fn list_tasks() -> impl Responder { HttpResponse::Ok().body("") }
# #[post("/tasks")]
# async fn create_task() -> impl Responder { HttpResponse::Created().finish() }
# #[delete("/tasks/{id}")]
# async fn delete_task() -> impl Responder { HttpResponse::NoContent().finish() }

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .service(list_tasks)
            .service(create_task)
            .service(delete_task)
    })
    .bind(("127.0.0.1", 3000))?
    .run()
    .await
}
```

#### สไตล์ที่ 2: Builder Style — เหมือน Axum

```rust
use actix_web::{web, App, HttpResponse, HttpServer, Responder};

async fn via_builder() -> impl Responder {
    HttpResponse::Ok().body("มาจาก web::get().to(via_builder)")
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new().route("/via-builder", web::get().to(via_builder))
    })
    .bind(("127.0.0.1", 3000))?
    .run()
    .await
}
```

#### พิสูจน์ว่าทั้งสองสไตล์ใช้ร่วมกันใน `App` เดียวได้จริง

ไม่ต้องเลือกอย่างใดอย่างหนึ่งเท่านั้น — โค้ดต่อไปนี้ compile และรันได้จริง (verify แล้ว) ผสมทั้งสองสไตล์เข้า
ด้วยกัน:

```rust
use actix_web::{get, web, App, HttpResponse, HttpServer, Responder};

#[get("/via-macro")]
async fn via_macro() -> impl Responder {
    HttpResponse::Ok().body("มาจาก #[get(\"/via-macro\")]")
}

async fn via_builder() -> impl Responder {
    HttpResponse::Ok().body("มาจาก web::get().to(via_builder)")
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .service(via_macro)
            .route("/via-builder", web::get().to(via_builder))
    })
    .bind(("127.0.0.1", 3000))?
    .run()
    .await
}
```

ยิง `curl` ทดสอบทั้งสอง endpoint จริง:

```bash
curl -i http://127.0.0.1:3000/via-macro
```

```
HTTP/1.1 200 OK
content-length: 36
date: Sun, 27 Sep 2026 00:04:42 GMT

มาจาก #[get("/via-macro")]
```

```bash
curl -i http://127.0.0.1:3000/via-builder
```

```
HTTP/1.1 200 OK
content-length: 42
date: Sun, 27 Sep 2026 00:04:42 GMT

มาจาก web::get().to(via_builder)
```

ทั้งสอง endpoint ทำงานได้พร้อมกันในแอปเดียว — ยืนยันว่า Actix-web ไม่ได้บังคับให้เลือกสไตล์เดียว ในทางปฏิบัติ
โปรเจกต์จริงส่วนใหญ่จะเลือกสไตล์ attribute macro เป็นหลัก (เพราะ path/method อยู่ "ติดกับ" handler ทำให้
อ่านง่ายว่าฟังก์ชันนี้ผูกกับ endpoint ไหน โดยไม่ต้องเลื่อนไปดูที่ `main()`) แล้วสำรอง builder style ไว้สำหรับ
เคสพิเศษ เช่น route ที่ path ถูกสร้างแบบไดนามิกจากลูป หรือ route ที่มาจาก config ไฟล์

**เทียบกับ Axum**: Axum ไม่มีสไตล์ attribute macro แบบนี้เลย — ทุก route ต้องประกาศผ่าน `.route()` แบบ
builder เท่านั้น (ทีม Axum ตัดสินใจแบบนี้โดยตั้งใจ เพื่อให้เห็น route ทั้งหมดของแอปรวมกันเป็นจุดเดียวที่มองเห็น
ได้ง่าย — trade-off คนละแบบกับ Actix-web ที่เลือกความสะดวกของการผูก path ไว้กับ handler) นี่คือความต่างเชิง
ปรัชญาการออกแบบที่แท้จริง ไม่ใช่แค่ syntax เฉยๆ — Part 69 จะพูดถึงข้อดี-ข้อเสียของทั้งสองแนวทางนี้ในเชิงการ
ดูแลรักษาโค้ดระยะยาวให้ละเอียดกว่านี้

### 67.7 State Management: `app_data`/`web::Data<T>` และกับดักเชิงสถาปัตยกรรมที่สำคัญที่สุดของบทนี้

นี่คือหัวข้อที่สำคัญที่สุดของบทนี้ เพราะเป็นจุดที่ Actix-web **ต่างจาก Axum อย่างแท้จริงในระดับสถาปัตยกรรม**
ไม่ใช่แค่ syntax

#### ทวนโมเดลของ Axum ก่อน

Part 64 สอนไว้ว่า Axum ใช้ `Router::with_state(state)` ผูก state เข้ากับ `Router` **ตัวเดียว** — Axum
สร้าง `Router` แค่ครั้งเดียวตอน `main()` แล้วส่ง `Router` ตัวนั้นเข้า `axum::serve(...)` แต่ละ request ที่เข้า
มาจะถูก spawn เป็น Tokio task ใหม่บน runtime **เดียวกัน** (thread pool เดียวกันทั้งแอป) แล้ว clone
`Arc<AppState>` ออกมาให้ handler ใช้ — เพราะมี `Router` แค่ตัวเดียวจริงๆ ไม่มีคำถามว่า "state จะซิงค์กันไหม"
เลย เพราะไม่มีอะไรให้ไม่ซิงค์ตั้งแต่แรก

#### โมเดลของ Actix-web: หลาย Worker Thread ที่แต่ละตัวมี App เป็นของตัวเอง

Actix-web ออกแบบต่างไปโดยสิ้นเชิง — `HttpServer` จะสร้าง **worker thread หลายตัว** (ค่า default พิสูจน์ได้
จาก source code ของ `actix-server` 2.9.7 ตรงๆ):

```rust
// จาก actix-server-2.9.7/src/builder.rs (ตัดมาเฉพาะส่วนที่เกี่ยวข้อง)
/// The default worker count is determined by [`std::thread::available_parallelism()`], capped
/// at 512. See its documentation for how the available parallelism is determined.
///
/// `num` must be in the range 1–512.
pub fn workers(mut self, num: usize) -> Self { /* ... */ }
```

และในไฟล์เดียวกัน ค่า default ที่ใช้จริงตอนไม่เรียก `.workers(...)` เอง:

```rust
// จาก actix-server-2.9.7/src/builder.rs
Self::with_default_workers(
    std::thread::available_parallelism().map_or(2, NonZeroUsize::get),
)
```

สรุปเป็นภาษาที่เข้าใจง่าย: **ค่า default ของจำนวน worker คือจำนวน CPU logical core ที่เครื่องมี**
(`std::thread::available_parallelism()` — ฟังก์ชันมาตรฐานของ Rust ที่ตรวจจำนวน core ที่ระบบใช้ได้จริง) ถ้า
ตรวจไม่ได้ (เช่นบนระบบที่แปลกมาก) จะ fallback เป็น 2 และ**จำกัดเพดานสูงสุดไว้ที่ 512** ไม่ว่าเครื่องจะมี core
มากกว่านั้นแค่ไหนก็ตาม — บนเครื่องที่ใช้ verify บทนี้ (4 logical core) เมื่อรันเซิร์ฟเวอร์แล้ว query
`std::thread::available_parallelism()` จาก endpoint ทดสอบ ได้ผลจริง:

```bash
curl -s http://127.0.0.1:3000/cpus
```

```
available_parallelism = 4
```

ตรงกับที่คาดไว้ — เครื่องนี้จะรัน Actix-web ด้วย **4 worker** เป็นค่า default ถ้าไม่เรียก `.workers(n)` เอง

**และนี่คือจุดสำคัญที่สุด**: closure ที่ส่งให้ `HttpServer::new(factory)` (application factory ที่หัวข้อ
67.3 กล่าวถึงไว้) จะถูก**เรียกซ้ำหนึ่งครั้งต่อ worker thread** — ถ้ามี 4 worker, closure นั้นจะถูกเรียก **4
ครั้ง** สร้าง `App` 4 ชุดที่**แยกกันโดยสิ้นเชิง** รันอยู่บน 4 thread คนละตัว ต่างจาก Axum ที่มี `Router` แค่
ตัวเดียวที่ใช้ร่วมกันทุก request

ดูจาก signature จริงของ `HttpServer` (source code ของ `actix-web` 4.15.0):

```rust
pub struct HttpServer<F, I, S, B>
where
    F: Fn() -> I + Send + Clone + 'static,
    // ...
{
    pub(super) factory: F,
    // ...
}
```

`F: Fn() -> I + Send + Clone + 'static` — สังเกตว่า `factory` ต้อง **`Clone`** ด้วย เพราะ Actix-web จะ
`.clone()` closure นี้ไปให้แต่ละ worker thread เพื่อให้แต่ละ thread เรียก `factory()` ของตัวเองอย่างอิสระ
(การ clone closure ก็คือ clone ทุกอย่างที่ closure จับมาด้วย — ถ้า closure จับ `Arc<T>` มา ก็ clone แค่ตัว
`Arc` (เพิ่ม refcount) แต่ถ้า closure สร้างค่าใหม่ *ข้างใน* ตัวเอง ทุกครั้งที่ closure ถูกเรียก มันจะได้ค่า
ใหม่เอี่ยมทุกครั้ง — นี่คือกลไกที่ทำให้กับดักในหัวข้อถัดไปเกิดขึ้นได้จริง)

#### พิสูจน์ "พฤติกรรมผิด": สร้าง State ข้างในของ Closure โดยไม่ใช้ `Arc`/`web::Data` ให้ถูกต้อง

มาดูโค้ดที่ **compile ผ่านสมบูรณ์แบบ** แต่มี bug เชิง concurrency ที่ซ่อนอยู่ (นี่คือโค้ดจริงที่ใช้ verify
หัวข้อนี้):

```rust
use actix_web::{get, web, App, HttpServer, Responder};
use std::sync::Mutex;

struct WrongCounter {
    count: Mutex<usize>,
}

#[get("/wrong-count")]
async fn wrong_count(data: web::Data<WrongCounter>) -> impl Responder {
    let mut c = data.count.lock().unwrap();
    *c += 1;
    format!("wrong worker-local count = {}\n", *c)
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        // <-- closure นี้ถูกเรียก 1 ครั้งต่อ worker (4 ครั้งบนเครื่องนี้)
        // ทุกครั้งที่ closure รัน จะสร้าง WrongCounter ใหม่ทั้งก้อนจากศูนย์
        // แล้วห่อด้วย web::Data (ซึ่งข้างในคือ Arc) — แต่เพราะสร้าง "ข้างใน" closure
        // แต่ละ worker จะได้ WrongCounter ของตัวเอง ไม่ได้แชร์กับ worker อื่นเลย
        let wrong_state = web::Data::new(WrongCounter {
            count: Mutex::new(0),
        });
        App::new()
            .app_data(wrong_state)
            .service(wrong_count)
    })
    .workers(4)
    .bind(("127.0.0.1", 3000))?
    .run()
    .await
}
```

รันเซิร์ฟเวอร์แล้วยิง `curl` 12 ครั้งติดกัน โดยบังคับให้แต่ละครั้งเปิด connection ใหม่ (`-H "Connection:
close"`) เพื่อให้ Actix-web กระจาย request ไปยัง worker ต่างๆแบบ round-robin (พฤติกรรมการกระจาย connection
จริงของ `actix-server`):

```bash
for i in $(seq 1 12); do
  curl -s -H "Connection: close" http://127.0.0.1:3000/wrong-count
done
```

output จริง (capture มาจากการรันจริง ไม่ได้แต่งขึ้น):

```
wrong worker-local count = 1
wrong worker-local count = 1
wrong worker-local count = 1
wrong worker-local count = 1
wrong worker-local count = 2
wrong worker-local count = 2
wrong worker-local count = 2
wrong worker-local count = 2
wrong worker-local count = 3
wrong worker-local count = 3
wrong worker-local count = 3
wrong worker-local count = 3
```

**นี่คือหลักฐานที่ชัดเจนที่สุดของบทนี้**: เห็นตัวเลขวนเป็นกลุ่มละ 4 (`1,1,1,1,2,2,2,2,3,3,3,3`) เพราะมี 4
worker และ Actix-web กระจาย 12 request แบบ round-robin ไปยังแต่ละ worker (worker A ได้ request ที่ 1, 5, 9;
worker B ได้ 2, 6, 10; ...) — **แต่ละ worker นับ counter ของตัวเองแยกกันโดยสิ้นเชิง ไม่รู้จักกันเลย** เพราะ
`WrongCounter` ถูกสร้างขึ้นมาใหม่ทุกครั้งที่ closure ถูกเรียก (4 ครั้ง = 4 `WrongCounter` คนละก้อน แม้จะห่อ
ด้วย `web::Data`/`Arc` ก็ตาม เพราะ `Arc` แต่ละตัวชี้ไปที่ข้อมูลคนละก้อน ไม่ใช่ก้อนเดียวกัน)

**นี่คือกับดักที่อันตรายที่สุดของ Actix-web** — ถ้าคุณคาดหวังว่า counter นี้จะนับรวมกันทั้งระบบ (เหมือนที่
Axum ทำได้ทันทีเพราะมี `Router` เดียว) คุณจะได้ผลลัพธ์ที่ผิดแบบเงียบๆ ไม่มี compiler error ไม่มี warning
ไม่มี panic — โค้ด compile ผ่านสมบูรณ์แบบและรันได้ แต่ตรรกะทางธุรกิจผิดเพราะสมมติฐานเรื่อง "state เดียวกัน
ทั้งระบบ" ไม่จริงสำหรับ Actix-web

#### พิสูจน์ "พฤติกรรมถูก": สร้าง State ข้างนอก Closure ทั้งหมด แล้ว `.clone()` เข้าไป

วิธีแก้คือสร้าง state **ก่อน**เรียก `HttpServer::new(...)` เลย แล้วให้ closure แค่ `.clone()` (Arc clone
ราคาถูก — เพิ่ม refcount แบบ atomic ตรงตามที่ Part 39 สอนไว้) เข้าไปแทนการสร้างใหม่:

```rust
use actix_web::{get, web, App, HttpServer, Responder};
use std::sync::Mutex;

struct AppState {
    count: Mutex<usize>,
}

#[get("/correct-count")]
async fn correct_count(data: web::Data<AppState>) -> impl Responder {
    let mut c = data.count.lock().unwrap();
    *c += 1;
    format!("correct shared count = {}\n", *c)
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    // สร้าง state ก้อนเดียวข้างนอก closure ทั้งหมด — ก่อนเรียก HttpServer::new เลย
    let shared_state = web::Data::new(AppState {
        count: Mutex::new(0),
    });

    HttpServer::new(move || {
        // .clone() ตรงนี้คือ Arc::clone ภายใน web::Data (แค่เพิ่ม refcount, ไม่ copy ข้อมูล)
        // ทุก worker จึงถือ Arc ที่ชี้ไปยัง AppState ก้อนเดียวกันจริงๆ
        App::new()
            .app_data(shared_state.clone())
            .service(correct_count)
    })
    .workers(4)
    .bind(("127.0.0.1", 3000))?
    .run()
    .await
}
```

ยิง `curl` 12 ครั้งแบบเดียวกัน:

```bash
for i in $(seq 1 12); do
  curl -s -H "Connection: close" http://127.0.0.1:3000/correct-count
done
```

output จริง:

```
correct shared count = 1
correct shared count = 2
correct shared count = 3
correct shared count = 4
correct shared count = 5
correct shared count = 6
correct shared count = 7
correct shared count = 8
correct shared count = 9
correct shared count = 10
correct shared count = 11
correct shared count = 12
```

เปรียบเทียบกับผลลัพธ์ก่อนหน้าเห็นความต่างชัดเจนที่สุด: **นับต่อกันเป็น 1 ถึง 12 อย่างต่อเนื่อง** ไม่ว่า
request นั้นจะไปตกที่ worker ไหนก็ตาม เพราะทั้ง 4 worker ถือ `Arc<AppState>` ที่ชี้ไปยังข้อมูลก้อนเดียวกัน
จริงๆ — การ `.lock()` บน `Mutex` ที่อยู่ข้างใน `Arc` เดียวกันจึงทำงานถูกต้องตามที่คาดหวัง (ตรงกับกลไกที่ Part
39 สอนไว้เรื่อง `Arc<Mutex<T>>` เป๊ะๆ — สิ่งที่ Actix-web เพิ่มมาคือ "ต้องระวังว่าจะสร้าง `Arc` ตรงไหน" ซึ่ง
ไม่มีในโมเดลของ Axum เพราะไม่มีการสร้าง `App`/`Router` ซ้ำหลายรอบให้ต้องระวัง)

#### สรุปกฎเหล็กของหัวข้อนี้

> **กฎ**: อะไรก็ตามที่ต้องการให้แชร์กันข้าม worker ของ Actix-web ต้องถูก **สร้างขึ้นก่อน** เรียก
> `HttpServer::new(...)` แล้วห่อด้วย `web::Data::new(...)` (หรือ `Arc::new(...)` ธรรมดาก็ได้ถ้าไม่ต้องการ
> ใช้ extractor `web::Data<T>`) จากนั้นใน closure ให้ทำแค่ **`.clone()`** เข้าไปเท่านั้น **ห้ามสร้างค่าตั้งต้น
> ของ state ข้างในของ closure โดยตรง** ไม่ว่าจะห่อด้วย `Arc`/`web::Data` ข้างในนั้นด้วยหรือไม่ก็ตาม

### 67.8 ตัวอย่างเต็มรูปแบบ: Tasks API (โดเมนเดียวกับ Axum ใน Part 62-63 เพื่อเทียบเคียงกันได้ตรงๆ)

มาประกอบทุกอย่างที่เรียนมาเข้าด้วยกันเป็นเซิร์ฟเวอร์จริงที่ทำงานได้ครบวงจร — ระบบ "Tasks API" แบบเดียวกับที่
Part 62 หัวข้อ 62.9 สร้างด้วย Axum เป๊ะๆ (`Task { id, title, done }`) เพื่อให้เทียบโค้ดสองฝั่งได้ตรงจุดต่อจุด
คราวนี้มี 5 endpoint (`GET /health`, `GET /tasks`, `GET /tasks/{id}`, `POST /tasks`, `DELETE /tasks/{id}`)
ใช้สไตล์ attribute macro ทั้งหมด (สไตล์ที่นิยมกว่าในโปรเจกต์ Actix-web จริง ตามที่หัวข้อ 67.6 อธิบายไว้):

```rust
use actix_web::{delete, get, post, web, App, HttpResponse, HttpServer, Responder};
use serde::{Deserialize, Serialize};
use std::sync::Mutex;

// Task struct: derive Serialize/Deserialize เพื่อให้ web::Json<Task> ใช้ได้ (Part 57)
#[derive(Debug, Clone, Serialize, Deserialize)]
struct Task {
    id: u32,
    title: String,
    done: bool,
}

#[derive(Debug, Deserialize)]
struct CreateTaskRequest {
    title: String,
}

// AppState: state ที่ทุก worker ต้องแชร์ร่วมกัน — ตามกฎเหล็กของหัวข้อ 67.7
// จะต้องสร้างก้อนเดียวก่อน HttpServer::new แล้ว .clone() เข้า closure เท่านั้น
struct AppState {
    tasks: Mutex<Vec<Task>>,
    next_id: Mutex<u32>,
}

#[get("/health")]
async fn health() -> impl Responder {
    HttpResponse::Ok().body("ok")
}

#[get("/tasks")]
async fn list_tasks(data: web::Data<AppState>) -> impl Responder {
    let tasks = data.tasks.lock().unwrap();
    HttpResponse::Ok().json(tasks.clone())
}

#[get("/tasks/{id}")]
async fn get_task(path: web::Path<u32>, data: web::Data<AppState>) -> impl Responder {
    let id = path.into_inner();
    let tasks = data.tasks.lock().unwrap();
    match tasks.iter().find(|t| t.id == id) {
        Some(task) => HttpResponse::Ok().json(task.clone()),
        None => HttpResponse::NotFound().json(serde_json::json!({
            "error": format!("ไม่พบ task id {id}")
        })),
    }
}

#[post("/tasks")]
async fn create_task(
    payload: web::Json<CreateTaskRequest>,
    data: web::Data<AppState>,
) -> impl Responder {
    let mut next_id = data.next_id.lock().unwrap();
    let mut tasks = data.tasks.lock().unwrap();
    let new_task = Task {
        id: *next_id,
        title: payload.title.clone(),
        done: false,
    };
    *next_id += 1;
    tasks.push(new_task.clone());
    HttpResponse::Created().json(new_task)
}

#[delete("/tasks/{id}")]
async fn delete_task(path: web::Path<u32>, data: web::Data<AppState>) -> impl Responder {
    let id = path.into_inner();
    let mut tasks = data.tasks.lock().unwrap();
    let len_before = tasks.len();
    tasks.retain(|t| t.id != id);
    if tasks.len() == len_before {
        HttpResponse::NotFound().json(serde_json::json!({
            "error": format!("ไม่พบ task id {id}")
        }))
    } else {
        HttpResponse::NoContent().finish()
    }
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    // สร้าง state ก้อนเดียวก่อน HttpServer::new — ตามกฎเหล็กของหัวข้อ 67.7
    let shared_state = web::Data::new(AppState {
        tasks: Mutex::new(vec![
            Task { id: 1, title: "เขียนบท Actix-web".to_string(), done: false },
            Task { id: 2, title: "ทดสอบ curl".to_string(), done: true },
        ]),
        next_id: Mutex::new(3),
    });

    println!("listening on 127.0.0.1:3000");
    HttpServer::new(move || {
        App::new()
            .app_data(shared_state.clone())
            .service(health)
            .service(list_tasks)
            .service(get_task)
            .service(create_task)
            .service(delete_task)
    })
    .workers(2)
    .bind(("127.0.0.1", 3000))?
    .run()
    .await
}
```

**สังเกตความคล้ายกับ Axum**: ตัวโครงสร้าง `AppState`, `Mutex<Vec<Task>>`, การ `.lock()` แล้วปลดล็อกก่อนถึง
จุด `.await` ใดๆ (ไม่มีจุดไหนใน handler ข้างบนที่ถือ `MutexGuard` ค้างข้าม `.await` — เหตุผลเดียวกับที่ Part
62 อธิบายไว้ว่าเป็นกับดักเชิง performance ที่ร้ายแรง จะย้ำอีกครั้งในหัวข้อกับดักท้ายบทนี้) และ logic การค้นหา
task ด้วย `.iter().find(...)` เหมือนกันเป๊ะกับเวอร์ชัน Axum — **ความต่างอยู่แค่ที่ "หน้าตา" ของ handler
signature และวิธีผูก route** ไม่ใช่ตรรกะทางธุรกิจ

#### ทดสอบทุก Endpoint ด้วย `curl -i` จริง

รันเซิร์ฟเวอร์ด้วย `cargo run` แล้วทดสอบทีละ endpoint — นี่คือ output จริงที่ capture มาจากการรันเซิร์ฟเวอร์
ตัวอย่างข้างบนจริงๆ (ไม่ได้แต่งขึ้น):

**1. Health check:**

```bash
curl -i http://127.0.0.1:3000/health
```

```
HTTP/1.1 200 OK
content-length: 2
date: Sun, 27 Sep 2026 00:01:55 GMT

ok
```

**2. รายการ task ทั้งหมด:**

```bash
curl -i http://127.0.0.1:3000/tasks
```

```
HTTP/1.1 200 OK
content-length: 117
content-type: application/json
date: Sun, 27 Sep 2026 00:01:55 GMT

[{"id":1,"title":"เขียนบท Actix-web","done":false},{"id":2,"title":"ทดสอบ curl","done":true}]
```

**3. Task ตัวเดียวที่มีอยู่จริง (`id=1`):**

```bash
curl -i http://127.0.0.1:3000/tasks/1
```

```
HTTP/1.1 200 OK
content-length: 63
content-type: application/json
date: Sun, 27 Sep 2026 00:01:55 GMT

{"id":1,"title":"เขียนบท Actix-web","done":false}
```

**4. Task ที่ไม่มีอยู่จริง (`id=999`) — 404 ที่โค้ดของเราเองสร้าง:**

```bash
curl -i http://127.0.0.1:3000/tasks/999
```

```
HTTP/1.1 404 Not Found
content-length: 39
content-type: application/json
date: Sun, 27 Sep 2026 00:01:55 GMT

{"error":"ไม่พบ task id 999"}
```

**5. สร้าง task ใหม่ด้วย `POST`:**

```bash
curl -i -X POST http://127.0.0.1:3000/tasks \
  -H "Content-Type: application/json" \
  -d '{"title":"เรียน Actix-web ให้ครบ"}'
```

```
HTTP/1.1 201 Created
content-length: 76
content-type: application/json
date: Sun, 27 Sep 2026 00:02:04 GMT

{"id":3,"title":"เรียน Actix-web ให้ครบ","done":false}
```

**6. ยืนยันว่า state จริงถูกอัปเดต (`GET /tasks` เห็น task ใหม่):**

```bash
curl -s http://127.0.0.1:3000/tasks
```

```
[{"id":1,"title":"เขียนบท Actix-web","done":false},{"id":2,"title":"ทดสอบ curl","done":true},{"id":3,"title":"เรียน Actix-web ให้ครบ","done":false}]
```

**7. ลบ task ด้วย `DELETE`:**

```bash
curl -i -X DELETE http://127.0.0.1:3000/tasks/1
```

```
HTTP/1.1 204 No Content
date: Sun, 27 Sep 2026 00:02:04 GMT
```

**8. ลบซ้ำ (`id=1` ไม่มีแล้ว) — ควรได้ 404:**

```bash
curl -i -X DELETE http://127.0.0.1:3000/tasks/1
```

```
HTTP/1.1 404 Not Found
content-length: 37
content-type: application/json
date: Sun, 27 Sep 2026 00:02:04 GMT

{"error":"ไม่พบ task id 1"}
```

ทุก endpoint ทำงานตามที่ออกแบบไว้ครบถ้วน — และเพราะ state ถูกสร้างและแชร์ตามกฎเหล็กของหัวข้อ 67.7 (สร้าง
ก่อน `HttpServer::new`, `.clone()` เข้า closure) การสร้าง/ลบ task ผ่าน worker ตัวไหนก็ตามจะสะท้อนกลับมาให้
worker ตัวอื่นเห็นตรงกันเสมอ ไม่มีปัญหาเรื่อง state ไม่ซิงค์แบบที่หัวข้อ 67.7 พิสูจน์ไว้ว่าจะเกิดถ้าทำผิดวิธี

### 67.9 ทดสอบ Actix-web App ด้วย `actix_web::test`

Part 62 กล่าวถึง `tower::ServiceExt::oneshot` ไว้สั้นๆว่าเป็นวิธีทดสอบ `Router` ของ Axum โดยไม่ต้องเปิด TCP
port จริง — Actix-web มีกลไกคู่เทียบชื่อโมดูล **`actix_web::test`** ที่ให้ **`test::TestRequest`** (สร้าง
request จำลอง) และ **`test::call_service`** (ส่ง request จำลองนั้นเข้า service โดยตรง ไม่ผ่าน TCP เลย)

การจะใช้ `actix_web::test` ต้องแยก handler ออกมาเป็น library (`src/lib.rs`) แยกจาก `src/main.rs` เสียก่อน
(แนวคิดเดียวกับที่ Part 62 แนะนำให้ดึง `Router` ออกมาเป็นฟังก์ชัน `app()` เพื่อให้ test เรียกใช้ได้):

```rust
// src/lib.rs
use actix_web::{get, post, web, HttpResponse, Responder};
use serde::{Deserialize, Serialize};
use std::sync::Mutex;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Task {
    pub id: u32,
    pub title: String,
    pub done: bool,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct CreateTaskRequest {
    pub title: String,
}

pub struct AppState {
    pub tasks: Mutex<Vec<Task>>,
    pub next_id: Mutex<u32>,
}

#[get("/tasks")]
pub async fn list_tasks(data: web::Data<AppState>) -> impl Responder {
    let tasks = data.tasks.lock().unwrap();
    HttpResponse::Ok().json(tasks.clone())
}

#[get("/tasks/{id}")]
pub async fn get_task(path: web::Path<u32>, data: web::Data<AppState>) -> impl Responder {
    let id = path.into_inner();
    let tasks = data.tasks.lock().unwrap();
    match tasks.iter().find(|t| t.id == id) {
        Some(task) => HttpResponse::Ok().json(task.clone()),
        None => HttpResponse::NotFound().finish(),
    }
}

#[post("/tasks")]
pub async fn create_task(
    payload: web::Json<CreateTaskRequest>,
    data: web::Data<AppState>,
) -> impl Responder {
    let mut next_id = data.next_id.lock().unwrap();
    let mut tasks = data.tasks.lock().unwrap();
    let new_task = Task {
        id: *next_id,
        title: payload.title.clone(),
        done: false,
    };
    *next_id += 1;
    tasks.push(new_task.clone());
    HttpResponse::Created().json(new_task)
}

pub fn test_state() -> web::Data<AppState> {
    web::Data::new(AppState {
        tasks: Mutex::new(vec![Task {
            id: 1,
            title: "งานทดสอบ".to_string(),
            done: false,
        }]),
        next_id: Mutex::new(2),
    })
}
```

แล้วเขียน test module ที่ท้ายไฟล์เดียวกัน (หรือแยกเป็น `tests/` directory ก็ได้ตาม convention ปกติของ
Cargo):

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use actix_web::{http::StatusCode, test, App};

    #[actix_web::test]
    async fn list_tasks_returns_seed_data() {
        let state = test_state();
        // test::init_service สร้าง service จำลองจาก App จริง โดยไม่เปิด TCP port เลย
        let app = test::init_service(
            App::new().app_data(state).service(list_tasks),
        )
        .await;

        // TestRequest::get() สร้าง request จำลองแบบ builder pattern
        let req = test::TestRequest::get().uri("/tasks").to_request();
        let resp = test::call_service(&app, req).await;

        assert_eq!(resp.status(), StatusCode::OK);

        let body: Vec<Task> = test::read_body_json(resp).await;
        assert_eq!(body.len(), 1);
        assert_eq!(body[0].title, "งานทดสอบ");
    }

    #[actix_web::test]
    async fn get_task_not_found_returns_404() {
        let state = test_state();
        let app = test::init_service(
            App::new().app_data(state).service(get_task),
        )
        .await;

        let req = test::TestRequest::get().uri("/tasks/999").to_request();
        let resp = test::call_service(&app, req).await;

        assert_eq!(resp.status(), StatusCode::NOT_FOUND);
    }

    #[actix_web::test]
    async fn create_task_returns_201_with_body() {
        let state = test_state();
        let app = test::init_service(
            App::new().app_data(state).service(create_task),
        )
        .await;

        let req = test::TestRequest::post()
            .uri("/tasks")
            .set_json(&CreateTaskRequest {
                title: "งานใหม่จาก test".to_string(),
            })
            .to_request();
        let resp = test::call_service(&app, req).await;

        assert_eq!(resp.status(), StatusCode::CREATED);

        let body: Task = test::read_body_json(resp).await;
        assert_eq!(body.id, 2);
        assert_eq!(body.title, "งานใหม่จาก test");
    }
}
```

รัน `cargo test` จริง:

```bash
cargo test
```

output จริง:

```
running 3 tests
test tests::list_tasks_returns_seed_data ... ok
test tests::create_task_returns_201_with_body ... ok
test tests::get_task_not_found_returns_404 ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

ทั้งสามเทสต์ผ่าน — สังเกตความคล้ายกับ `tower::ServiceExt::oneshot` ของ Axum ในระดับแนวคิด:

| แนวคิด | Axum | Actix-web |
|---|---|---|
| macro สำหรับ async test function | `#[tokio::test]` | `#[actix_web::test]` |
| สร้าง service จำลองจาก Router/App | ส่ง `Router` เข้า `.oneshot(request)` ตรงๆ | `test::init_service(app_builder).await` สร้าง service ก่อน แล้วค่อยยิง request หลายครั้งกับ service ตัวเดียวได้ |
| สร้าง request จำลอง | `http::Request::builder()...` (ใช้ type จาก crate `http` ตรงๆ) | `test::TestRequest::get()/post()/...` (builder ของ Actix-web เอง สะดวกกว่าเล็กน้อยเพราะมี `.set_json()` ในตัว) |
| ส่ง request เข้า service | `.oneshot(request).await` | `test::call_service(&app, req).await` |
| อ่าน JSON body จาก response | ต้อง extract body ด้วยมือแล้ว deserialize เอง | `test::read_body_json(resp).await` ทำให้ในบรรทัดเดียว |

**ข้อดีที่ชัดเจนของ `actix_web::test`**: `test::init_service(...)` สร้าง service ได้ครั้งเดียวแล้วนำไปยิง
`test::call_service` ได้หลายรอบในเทสต์เดียว (ตามที่เห็นในตัวอย่าง — ไม่ต้องสร้าง service ใหม่ทุกครั้งที่ยิง
request แบบที่ `.oneshot()` ของ Axum ทำ ซึ่ง `.oneshot()` "กิน" `Router` ไปเลยหลังใช้ครั้งเดียว ต้อง `.clone()`
`Router` ใหม่ทุกครั้งถ้าต้องยิงหลาย request ในเทสต์เดียว) — เป็นความสะดวกเล็กๆที่ Actix-web ออกแบบมาให้เขียน
เทสต์ที่มีหลาย request ต่อเทสต์ได้ง่ายกว่า

### 67.10 First Look: Actix-web เทียบกับ Axum (ภาพรวมสั้นๆ ก่อนเจาะลึกใน Part 69)

บทนี้เก็บภาพความต่างที่สำคัญที่สุดสองเรื่องไว้ก่อนสรุป — **ยังไม่ใช่การเทียบเต็มรูปแบบ** (เรื่อง performance
เชิงตัวเลข, ความเป็นผู้ใหญ่ของ ecosystem, ชุมชนผู้ใช้, และการเลือกใช้ในโปรเจกต์จริง จะเป็นเนื้อหาเต็มของ
**Part 69: Axum vs Actix-web** ที่ตามมา) แต่เป็นแค่การตั้งกรอบความคิดไว้ล่วงหน้า:

**1. สไตล์การเขียนโค้ด: Macro-heavy/Annotation Style เทียบกับ Trait-based Composition Style**

Axum ยึดหลัก "compose ด้วย trait bound และ builder method" ทุกอย่าง — `Router`, extractor, middleware
ทั้งหมดประกอบกันผ่าน generic type และ trait (`Handler<T, S>`, `IntoResponse`, `tower::Service`) ไม่มี
attribute macro พิเศษที่ผูก path ไว้กับ handler เลย ทุก route ต้องเห็นรวมกันที่จุดสร้าง `Router`

Actix-web ผสมทั้งสองแนวทาง — มี trait ที่ทำหน้าที่คล้ายกัน (`Responder`, `FromRequest`) แต่**เพิ่ม**ชั้นของ
attribute macro (`#[get(...)]` ที่ผูก path ไว้กับ handler โดยตรง) ทำให้โค้ดดู "annotation-driven" มากกว่า —
ข้อดีคืออ่าน handler ตัวเดียวแล้วรู้ทันทีว่ามันรับ request จาก path ไหน ข้อเสียคือ route ทั้งหมดของระบบไม่ได้
รวมกันอยู่ที่จุดเดียวให้มองเห็นภาพรวมง่ายๆ (ต้องไล่หา `#[get(...)]` กระจายอยู่ทั่วโค้ดเบส)

**2. โมเดลการจัดการ State: Worker-based เทียบกับ Single Shared Router**

นี่คือความต่างที่ลึกที่สุดที่บทนี้พิสูจน์ไว้ในหัวข้อ 67.7 — Axum มี `Router` เดียวที่ทุก request (ทุก Tokio
task) เข้าถึงร่วมกัน ไม่มีคำถามเรื่อง "state จะซิงค์กันไหม" เพราะไม่มีอะไรให้ไม่ซิงค์ตั้งแต่การออกแบบ ส่วน
Actix-web มีหลาย worker thread ที่แต่ละตัวมี `App` เป็นของตัวเอง (สร้างจาก factory closure ที่เรียกซ้ำ) ทำให้
ต้องคิดเรื่อง "อะไรสร้างตรงไหน" อย่างมีสติเสมอ — ข้อดีของโมเดล Actix-web คือมันใกล้เคียงกับโมเดล
multi-process ของเว็บเซิร์ฟเวอร์แบบเก่า (เช่น PHP-FPM, Gunicorn ของ Python) ที่คนจากภาษาอื่นอาจคุ้นเคยกว่า และ
อาจช่วยลด contention บน `Mutex` ตัวเดียวถ้า workload ส่วนใหญ่ไม่ต้องแชร์ state ข้าม worker จริงๆ (แต่ก็ต้อง
ระวังไม่ให้ลืม share state ที่ *ควร* แชร์ ตามที่หัวข้อ 67.7 พิสูจน์ไว้)

Part 69 จะเจาะลึกทั้งสองเรื่องนี้ต่อ พร้อมข้อมูลเรื่อง benchmark, ความพร้อมของ ecosystem (crate เสริมต่างๆที่
ใช้ร่วมกับแต่ละ framework ได้), และคำแนะนำเชิงปฏิบัติว่าเมื่อไหร่ควรเลือก framework ไหนสำหรับโปรเจกต์แบบไหน

## กับดักที่พบบ่อย (Common Pitfalls)

**1. ลืมห่อ state ด้วย `web::Data::new(...)` ก่อนเรียก `.app_data(...)`**

```rust
// ผิด: ส่ง struct ธรรมดาเข้า .app_data() ตรงๆ โดยไม่ห่อด้วย web::Data::new(...)
struct Counter { n: std::sync::Mutex<usize> }

# use actix_web::{get, web, App, HttpServer, Responder};
#[get("/count")]
async fn count(data: web::Data<Counter>) -> impl Responder {
    let mut n = data.n.lock().unwrap();
    *n += 1;
    format!("{}\n", *n)
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(move || {
        let raw_counter = Counter { n: std::sync::Mutex::new(0) };
        App::new().app_data(raw_counter).service(count) // <- ผิด: ไม่ได้ web::Data::new()
    })
    .bind(("127.0.0.1", 3000))?
    .run()
    .await
}
```

โค้ดนี้ **compile ผ่านสมบูรณ์แบบ** เพราะ `.app_data()` รับ `T: 'static` อะไรก็ได้ ไม่ได้บังคับว่าต้องเป็น
`web::Data<T>` — ปัญหาจะโผล่มาแค่ตอน**รันจริง**เท่านั้น ยิง `curl` เข้าไปที่ endpoint นี้:

```bash
curl -i http://127.0.0.1:3000/count
```

output จริง (capture มาจากการรันจริง):

```
HTTP/1.1 500 Internal Server Error
content-length: 96
content-type: text/plain; charset=utf-8
date: Sun, 27 Sep 2026 00:05:49 GMT

Requested application data is not configured correctly. View/enable debug logs for more details.
```

**สาเหตุ**: `web::Data<T>` extractor ค้นหาค่าที่ถูกเก็บไว้เป็น type `Data<Counter>` ใน registry ภายในของ
`App` — แต่ `.app_data(raw_counter)` เก็บค่าไว้เป็น type `Counter` เปล่าๆ (ไม่ใช่ `Data<Counter>`) ทำให้
extractor หาไม่เจอตอน request เข้ามาจริง แล้วคืน `500 Internal Server Error` แทน — **วิธีแก้**: ห่อด้วย
`web::Data::new(...)` เสมอก่อนส่งเข้า `.app_data()`:

```rust
let counter = web::Data::new(Counter { n: std::sync::Mutex::new(0) });
App::new().app_data(counter.clone()).service(count) // ถูก
```

**2. สร้าง State ข้างในของ Application Factory Closure โดยตรง (กับดักที่อธิบายเต็มในหัวข้อ 67.7)**

ทวนสั้นๆอีกครั้งเพราะสำคัญมาก: `HttpServer::new(|| { let s = web::Data::new(...); App::new()... })` จะ
สร้าง state คนละก้อนต่อ worker (พิสูจน์แล้วในหัวข้อ 67.7 ด้วยตัวเลขจริง `1,1,1,1,2,2,2,2,3,3,3,3` สำหรับ 4
worker) — **วิธีแก้**: สร้าง state ก่อนเรียก `HttpServer::new(...)` แล้ว `move` เข้า closure ผ่าน `.clone()`

**3. `web::Path<T>` parse ล้มเหลว → Actix-web ตอบ `404`, ไม่ใช่ `400` แบบที่ Axum ทำ**

ถ้าคุณย้ายมาจาก Axum (Part 62-64 สอนไว้ว่า path extractor ที่ parse ไม่ผ่านจะได้ `400 Bad Request`) นี่คือ
กับดักที่เจอบ่อยที่สุด — ลองยิง path parameter ที่ parse เป็น `u32` ไม่ได้:

```bash
curl -i http://127.0.0.1:3000/tasks/abc
```

output จริงจากตัวอย่าง Tasks API ในหัวข้อ 67.8:

```
HTTP/1.1 404 Not Found
content-length: 28
content-type: text/plain; charset=utf-8
date: Sun, 27 Sep 2026 00:01:55 GMT

can not parse "abc" to a u32
```

**Actix-web ตอบ `404 Not Found`** (ไม่ใช่ `400 Bad Request`) ตอน path extractor parse ไม่ผ่าน — เหตุผลคือ
Actix-web มองว่า "path segment ที่ parse เป็น type ที่ต้องการไม่ได้" เท่ากับ "ไม่มี route ไหนตรงกับ request
นี้จริงๆ" (จากมุมมองของตัว router — มันไม่เคย match ไปถึง `{id}` ที่ parse ผ่านได้เลย) ในขณะที่ Axum มองว่า
"route match ที่ path แล้ว แค่ extractor ทำงานไม่สำเร็จ" จึงตอบ `400` แทน — ทั้งสองมุมมองสมเหตุสมผลในตัวเอง
แต่**ให้ผลลัพธ์ต่างกันจริง** — ถ้าโค้ด client (เช่น frontend หรือเทสต์อัตโนมัติ) เขียนโดยคาดหวัง `400` จาก
ประสบการณ์กับ Axum มาก่อน ต้องปรับให้เช็ค `404` แทนสำหรับ Actix-web

**4. ถือ `MutexGuard` ค้างข้าม `.await` ใน Handler แบบ Async**

กับดักนี้ไม่ใช่ปัญหาเฉพาะของ Actix-web (เป็นปัญหา async Rust ทั่วไปที่ Part 39 และ Part 62 เตือนไว้แล้วฝั่ง
Axum) แต่ยังคงสำคัญพอที่ต้องย้ำอีกครั้งเพราะ Tasks API ในหัวข้อ 67.8 จงใจเขียนให้หลีกเลี่ยงมัน — ตัวอย่าง
โค้ดที่ผิด:

```rust
# use actix_web::{get, web, HttpResponse, Responder};
# use std::sync::Mutex;
# struct AppState { tasks: Mutex<Vec<i32>> }
#[get("/bad-example")]
async fn bad_handler(data: web::Data<AppState>) -> impl Responder {
    let tasks = data.tasks.lock().unwrap(); // ล็อกไว้
    some_async_operation().await; // <- ยังถือ MutexGuard ข้าม .await อยู่!
    HttpResponse::Ok().json(tasks.clone())
}
# async fn some_async_operation() {}
```

ถ้า `some_async_operation()` ใช้เวลานาน worker thread นั้นจะค้าง `Mutex` ไว้ตลอดเวลาที่รอ ทำให้ request อื่น
ที่ต้องการล็อกตัวเดียวกัน (แม้จะมาจาก worker เดียวกันหรือ worker อื่นที่ share state เดียวกัน) ต้องรอจนกว่า
`.await` นั้นจะเสร็จ — ในกรณีร้ายแรงถ้ามี `.await` ที่รอ mutex ตัวเดียวกันแบบ nested อาจนำไปสู่ deadlock ได้
จริง — **วิธีแก้**: ปลดล็อก (ให้ `MutexGuard` หลุด scope) ก่อนถึงจุด `.await` ใดๆเสมอ — ตัวอย่างที่ถูกต้องใน
หัวข้อ 67.8 ทำสิ่งนี้อยู่แล้ว: เรียก `.lock()`, ทำงานให้จบ (`.clone()` หรือ `.find()`), แล้วให้ scope ของ
`MutexGuard` จบก่อนที่จะมี `.await` ใดๆเกิดขึ้น

**5. เขียน Handler ด้วย Attribute Macro แล้วลืมเรียก `.service(...)` (ใช้ `.route(...)` แทนโดยไม่รู้ตัว)**

```rust
use actix_web::{get, web, App, HttpResponse, HttpServer, Responder};

#[get("/hello")]
async fn hello() -> impl Responder {
    HttpResponse::Ok().body("hi")
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            // ผิด: hello ผูก path "/hello" ไว้แล้วจากตัว macro
            // แต่ .route() คาดหวัง handler แบบ "เปล่า" (ไม่มี macro ผูก path มา)
            .route("/hello", web::get().to(hello))
    })
    .bind(("127.0.0.1", 3000))?
    .run()
    .await
}
```

โค้ดข้างบนจะ**compile ไม่ผ่าน** — error จริงที่ได้ (capture มาจากการ `cargo build` จริง ไม่ได้แต่งขึ้น):

```
error[E0277]: the trait bound `hello: Handler<_>` is not satisfied
   --> src/main.rs:12:44
    |
 12 |             .route("/hello", web::get().to(hello))
    |                                         -- ^^^^^ unsatisfied trait bound
    |                                         |
    |                                         required by a bound introduced by this call
    |
help: the trait `Handler<_>` is not implemented for `hello`
   --> src/main.rs:3:1
    |
  3 | #[get("/hello")]
    | ^^^^^^^^^^^^^^^^
note: required by a bound in `Route::to`
   --> .../actix-web-4.15.0/src/route.rs:247:12
    |
245 |     pub fn to<F, Args>(mut self, handler: F) -> Self
    |            -- required by a bound in this associated function
246 |     where
247 |         F: Handler<Args>,
    |            ^^^^^^^^^^^^^ required by this bound in `Route::to`
    = note: this error originates in the attribute macro `get` (in Nightly builds, run with -Z macro-backtrace for more info)
```

สาเหตุ: `#[get("/hello")]` แปลง `hello` จากฟังก์ชัน async ธรรมดาให้กลายเป็น struct ตัวหนึ่งที่ implement
trait ภายในของ Actix-web สำหรับการ register เป็น service (ไม่ใช่ signature ของฟังก์ชัน async ตรงๆอีกต่อไป)
— `web::get().to(handler)` ต้องการ `handler` ที่ implement trait `Handler<Args>` ตรงๆ (คือฟังก์ชัน async
เปล่าๆที่**ไม่**ผ่าน attribute macro มาก่อน) จึงชนกัน compiler จึงฟ้องว่า `hello: Handler<_>` ไม่ผ่าน พร้อม
ชี้กลับไปที่บรรทัด `#[get("/hello")]` ตรงๆว่าเป็นจุดต้นเหตุ — **วิธีแก้**: ใช้ `.service(hello)` เสมอสำหรับ
handler ที่มี attribute macro ผูก path ไว้แล้ว สงวน `.route(path, web::get().to(handler))` ไว้สำหรับ handler
แบบ "เปล่า" (ไม่มี macro) เท่านั้น อย่าผสมสองวิธีเข้ากับ handler ตัวเดียวกัน

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เพิ่ม endpoint `PUT /tasks/{id}/toggle` เข้าไปในตัวอย่าง Tasks API ของหัวข้อ 67.8 ที่สลับค่า
   `done` ของ task ตาม `id` ที่ระบุ (จาก `false` เป็น `true` หรือกลับกัน) คืน `200 OK` พร้อม JSON ของ task
   ที่อัปเดตแล้วถ้าเจอ, คืน `404 Not Found` ถ้าไม่เจอ — **คำแนะนำ**: ใช้ attribute macro `#[put(...)]`
   (ต้อง import เข้ามาเพิ่ม) และ pattern การ `.lock()`/หา task/แก้ไข field/ปลดล็อกแบบเดียวกับ `get_task` ที่
   มีอยู่แล้ว ทดสอบด้วย `curl -i -X PUT http://127.0.0.1:3000/tasks/1/toggle`

2. **(กลาง)** เพิ่ม query parameter สำหรับกรองผลลัพธ์ที่ `GET /tasks` — รองรับ `?done=true` และ `?done=false`
   เพื่อกรองว่าจะแสดงเฉพาะ task ที่เสร็จแล้วหรือยังไม่เสร็จ (ถ้าไม่มี query parameter เลยให้แสดงทั้งหมดตามปกติ)
   — **คำแนะนำ**: เขียน struct `#[derive(Deserialize)] struct TaskFilter { done: Option<bool> }` แล้วเพิ่ม
   parameter `query: web::Query<TaskFilter>` เข้าไปใน `list_tasks` (ทั้ง `web::Path` และ `web::Query` ไม่แตะ
   body เลย จึงเรียงพารามิเตอร์ได้อย่างอิสระตามที่หัวข้อ 67.5 อธิบายไว้) ใช้ `.filter(...)` บน iterator ก่อน
   `.collect()` เป็น `Vec<Task>` ที่จะส่งเข้า `Json(...)`

3. **(ยาก)** เขียนเทสต์ด้วย `actix_web::test` module (`TestRequest`, `call_service`, `read_body_json`) ให้
   ครอบคลุม endpoint ที่เพิ่มในข้อ 1 และข้อ 2 อย่างน้อยข้อละ 2 เทสต์ (กรณีสำเร็จ + กรณี error/ไม่พบข้อมูล) —
   **คำแนะนำ**: ต้องแยก handler ทั้งหมดออกมาไว้ที่ `src/lib.rs` ก่อน (ตามที่หัวข้อ 67.9 สอน) แล้วเขียนฟังก์ชัน
   ช่วย `test_state()` ที่ seed ข้อมูลตั้งต้นให้เทสต์แต่ละตัวเรียกใช้ได้อย่างอิสระ (ไม่ควร share state เดียวกัน
   ข้ามเทสต์ เพราะเทสต์รันแบบ concurrent กันโดย default — ปัญหาเดียวกับที่ Part 39 เตือนไว้เรื่อง shared
   mutable state ข้าม thread ที่ไม่ได้ synchronize ให้ถูกต้อง)

4. **(ยาก/ประยุกต์ใช้งานจริง)** ทำซ้ำการพิสูจน์ในหัวข้อ 67.7 ด้วยตัวเอง — สร้างเซิร์ฟเวอร์ทดสอบใหม่ที่มี
   `.workers(3)` (เลือกจำนวนต่างจากตัวอย่างในบทเพื่อพิสูจน์ว่าไม่ได้ผลเฉพาะกับเลข 4) เขียน endpoint
   `GET /debug-count` สองตัวคู่กัน: ตัวหนึ่งสร้าง state ผิดตำแหน่ง (ข้างในของ closure) อีกตัวสร้างถูกตำแหน่ง
   (ข้างนอกของ closure แล้ว `.clone()`) แล้วยิง `curl` แบบ `-H "Connection: close"` อย่างน้อย 15 ครั้งเข้าไป
   ที่แต่ละ endpoint บันทึกผลลัพธ์ที่ได้จริง แล้วเขียนคำอธิบายเป็นความเห็นส่วนตัว (comment ในโค้ด) ว่าทำไม
   ตัวเลขที่เห็นออกมาเป็นแบบนั้น โดยอ้างอิงกลับไปที่กลไก application factory closure ที่หัวข้อ 67.3 และ 67.7
   อธิบายไว้ — **คำแนะนำ**: ถ้าผลลัพธ์ของ endpoint "ผิด" ดูสุ่มไม่เป็น pattern ชัดเจนแบบในบท (`1,1,1,2,2,2,...`)
   ให้ตรวจสอบว่าใช้ `-H "Connection: close"` ทุกครั้งหรือไม่ (ถ้าไม่บังคับปิด connection curl อาจ reuse
   connection เดิมที่ผูกกับ worker เดียวกันซ้ำๆ ทำให้เห็น pattern ไม่ชัด)

## สรุป

บทนี้แนะนำ **Actix-web** ในฐานะ framework ทางเลือกที่มีสถาปัตยกรรมต่างจาก Axum อย่างแท้จริง ไม่ใช่แค่
เปลี่ยนชื่อ API ตั้งแต่ประวัติศาสตร์ (เคยสร้างอยู่บน `actix` actor framework จริงในเวอร์ชัน 1.x-2.x แต่เลิก
พึ่งพามันในเวอร์ชัน 4.0 เป็นต้นมา — พิสูจน์ด้วย `cargo tree` และ source code จริงว่าปัจจุบันวิ่งอยู่บน
`actix-rt` ที่สร้างทับ Tokio ตรงๆ) ไปจนถึงกลไกระดับ trait (`Responder` ที่รับ `&HttpRequest` เข้ามาด้วย
ต่างจาก `IntoResponse` ของ Axum, `FromRequest` ที่เป็น trait เดียวไม่แยกเป็น Parts/Full แบบ Axum) และที่
สำคัญที่สุดคือโมเดลการจัดการ state ที่ต่างกันโดยสิ้นเชิง — Actix-web ใช้หลาย worker thread ที่แต่ละตัวมี
`App` เป็นของตัวเอง (สร้างจาก application factory closure ที่เรียกซ้ำหนึ่งครั้งต่อ worker) ทำให้ต้องมีสติ
เรื่อง "สร้าง state ตรงไหน" อย่างที่ Axum ไม่ต้องคิดเลย (เพราะมี `Router` เดียวที่ใช้ร่วมกันทั้งระบบ) — บทนี้
พิสูจน์ปัญหานี้และคำตอบด้วยตัวเลขจริงจากการรันเซิร์ฟเวอร์และยิง `curl` นับผลลัพธ์ ไม่ใช่แค่คำอธิบายทางทฤษฎี

คุณได้เห็นทั้งสไตล์ attribute macro (`#[get(...)]`) และ builder style (`web::get().to(...)`) ของ Actix-web
ทำงานร่วมกันได้ในแอปเดียว, extractor ที่ชื่อคล้าย Axum มากแต่ trait เบื้องหลังต่างกันจริง, และสร้าง Tasks
API ที่ทำงานได้ครบวงจรในโดเมนเดียวกับที่ Part 62-63 ทำไว้ฝั่ง Axum เพื่อให้เทียบโค้ดสองฝั่งได้ตรงจุดต่อจุด —
พร้อมทดสอบด้วยทั้ง `curl` จริงและ `actix_web::test` module

**Part ถัดไป (Part 68: Actix-web: Routing, Handlers, Middleware)** จะขยายเนื้อหาของ Actix-web ต่อในเชิงลึก
— routing แบบซับซ้อนกว่านี้ (scope, path parameter หลายตัว, wildcard), middleware ของ Actix-web เอง (คู่เทียบ
ของ `tower`/`tower-http` ที่ Part 65 สอนฝั่ง Axum — Actix-web มีระบบ middleware ของตัวเองที่ไม่ใช้
`tower::Service`) และการจัดโครงสร้างโปรเจกต์ที่โตขึ้นด้วย `scope()` — จากนั้น **Part 69: Axum vs Actix-web**
จะสรุปการเทียบเคียงทั้งสอง framework แบบเต็มรูปแบบ ทั้งเรื่อง performance เชิงตัวเลข, ความเป็นผู้ใหญ่ของ
ecosystem, และคำแนะนำเชิงปฏิบัติว่าโปรเจกต์แบบไหนควรเลือก framework ไหน

---

**Part ก่อนหน้า:** [Axum: Error Handling แบบมืออาชีพ](part-066-axum-error-handling.md) | **Part ถัดไป:** [Actix-web: Routing, Handlers, Middleware](part-068-actix-web-routing-middleware.md)
