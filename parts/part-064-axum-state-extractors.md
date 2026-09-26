# Part 64: Axum: State Management และ Extractors

> โมดูล: การพัฒนาเว็บแอปพลิเคชัน (Web Development) | ระดับ: กลาง-สูง | เวลาโดยประมาณ: 240 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้ว่าทำไม pattern การ capture `Arc<Mutex<T>>` ใน closure ตรง ๆ (ที่อาจใช้กันมาตั้งแต่ Part 63)
  ถึง**ไม่scale**พอแอปมีหลายชิ้นข้อมูลที่ต้องแชร์ (inventory, config, connection pool ที่จะเจอเต็ม ๆ ใน
  Part 70) และมีหลาย handler function ที่ต้องการ**คนละส่วน**ของข้อมูลเหล่านั้น — แล้วแก้ด้วยแพทเทิร์น
  `AppState` + `State<T>` extractor ที่เป็นมาตรฐานของ Axum
- ออกแบบ `AppState` เป็น struct เดียวที่รวมข้อมูลที่แชร์ทั้งหมดของแอป ห่อด้วย `Arc` ให้ถูกวิธี ผูกเข้ากับ
  `Router` ด้วย `.with_state()` แล้ว extract ออกมาใช้ในทุก handler ด้วย `State<AppState>` — พร้อมพิสูจน์
  ด้วยตัวอย่าง CRUD API ที่ compile และรันได้จริง ทดสอบด้วย `curl` จริง
- อธิบายได้อย่างละเอียดว่าทำไม `AppState` ต้องห่อด้วย `Arc` (ผูกกับต้นทุนของ `Clone` ที่ Part 56 วัดไว้แล้วว่า
  `Arc::clone` ถูกกว่าการ deep-clone มากแค่ไหน) และอ่าน**compiler error จริง**ที่เกิดขึ้นถ้าลืม
  `#[derive(Clone)]` หรือลืมห่อ `Arc` รอบ field ที่เป็น `Mutex`
- แยก state ก้อนใหญ่ออกเป็น "substate" ที่แต่ละกลุ่ม route เห็นแค่ส่วนที่ตัวเองต้องใช้ ผ่าน trait `FromRef`
  โดยไม่ต้องแยก `Router` เป็นหลายตัว
- อธิบายระบบ extractor ของ Axum ในระดับ trait ได้ทั้งสองฝั่ง — `FromRequestParts` (extractor ที่ไม่แตะ
  body เช่น `Path`, `Query`, `State`, header) กับ `FromRequest` (extractor ที่แตะ/กิน body เช่น `Json`,
  `Bytes`, `Form`) — และอธิบายได้ว่าทำไมกฎ "extractor ที่กิน body ต้องมาตัวสุดท้าย" ที่ Part 63 สอนไว้แบบ
  ท่องจำ ถึงเป็นผลตามธรรมชาติของการมี trait สองตัวนี้แยกกัน
- เขียน **custom extractor ของตัวเอง** โดย implement `FromRequestParts` ให้กับ type ที่อ่านและตรวจสอบ
  custom header (เช่น API key) พร้อม rejection response ของตัวเอง — เตรียมพื้นฐานให้ Part 74/76 เรื่อง
  authentication/authorization เต็มรูปแบบ
- เลือกใช้ `Extension<T>` ให้ถูกจุด เข้าใจว่าต่างจาก `State<T>` อย่างไร (compile-time guarantee vs
  runtime-only) และห่อ extractor ใด ๆ ด้วย `Option<T>`/`Result<T, T::Rejection>` เพื่อจัดการความล้มเหลว
  ของการ extract เองแทนปล่อยให้ auto-reject

## ความรู้ที่ต้องมีมาก่อน

- **Part 61 (HTTP Fundamentals และ REST API Concepts)**: บทนี้ใช้คำศัพท์ HTTP method, status code,
  header พื้นฐานตลอดทั้งบทโดยไม่อธิบายซ้ำ
- **Part 62 (แนะนำ Axum Framework)** และ **Part 63 (Axum: Routing และ Handlers)**: บทนี้พึ่งพา Part 62-63
  อย่างหนัก — สมมติว่าคุณเขียน `Router`, ผูก handler ด้วย `.route()`, ใช้ `Path<T>`/`Query<T>` extractor,
  และคืนค่าที่ implement `IntoResponse` มาแล้ว รวมถึงกฎ "extractor ที่กิน body ต้องมาเป็นตัวสุดท้ายในลิสต์
  parameter" ที่ Part 63 สอนไว้ (บทนี้จะอธิบายว่า**ทำไม**กฎนี้ถึงมีอยู่ในระดับ trait)
- **Part 15 (Collections: HashMap, HashSet, BTreeMap/BTreeSet)**: ตัวอย่างทั้งบทใช้ `HashMap` เป็น
  in-memory store จำลอง database
- **Part 28 (Smart Pointers: Rc<T> และ RefCell<T>)** และ **Part 39 (Mutex, Arc และ Shared-State
  Concurrency)**: หัวใจของบทนี้คือการแชร์ state ข้าม handler/thread ด้วย `Arc<Mutex<T>>` — ถ้า Part 39
  (โดยเฉพาะว่าทำไม `Mutex` ต้องมาคู่กับ `Arc` ในโลก multi-thread) ยังไม่แน่น กลับไปทวนก่อน เพราะบทนี้จะไม่
  อธิบายกลไก `Mutex`/`Arc` ซ้ำจากศูนย์
- **Part 46-50 (Async/Await, Futures, Tokio Runtime/Tasks/I/O, Async Channels)**: Axum เป็น async
  framework เต็มรูปแบบ handler ทุกตัวเป็น `async fn` ที่รันบน Tokio runtime — บทนี้จะอ้างอิงความเข้าใจ
  เรื่อง task/`.await` จาก Part 46-50 ตลอดเวลา
- **Part 56 (Memory Management ขั้นสูงและ Zero-cost Abstractions)**: บทนี้อ้างอิงกลับไปที่ตัวเลขจริงที่
  Part 56 วัดไว้ว่า `Arc::clone` (เพิ่ม reference count) ถูกกว่าการ deep-clone โครงสร้างข้อมูลใหญ่มากแค่ไหน
  — เป็นเหตุผลหลักที่ `AppState` ควรห่อด้วย `Arc`
- **Part 57 (Serialization: Serde เบื้องต้น)**: handler ที่ตอบ/รับ JSON ในบทนี้ใช้ `#[derive(Serialize,
  Deserialize)]` ที่ Part 57 สอนไว้ตลอด
- **Part 12 (Result<T,E> และ Error Handling เบื้องต้น)**: rejection ของ custom extractor คือรูปแบบหนึ่งของ
  `Result<T, E>` ที่ `E` (Rejection) ต้อง implement `IntoResponse` — ใช้ความเข้าใจ `Result` จาก Part 12 ตรง ๆ

## เนื้อหา

### 64.1 ปัญหา: `Arc<Mutex<Vec<T>>>` ใน Closure ไม่ Scale พอแอปโตขึ้น

จาก Part 63 คุณอาจเคยเห็น (หรือเขียนเอง) แอป Axum เล็ก ๆ ที่แชร์ state ด้วยการ `clone()` ตัวแปร `Arc<Mutex<T>>`
เข้าไปใน closure ตรง ๆ ก่อนสร้าง handler แต่ละตัว ลองดูรูปแบบนี้ก่อน (นี่คือโค้ดที่ "ใช้งานได้" แต่กำลังจะเจอ
ปัญหาเมื่อแอปโตขึ้น):

```rust
use axum::{routing::get, Router};
use std::sync::{Arc, Mutex};

async fn naive_pattern_sketch() {
    let inventory = Arc::new(Mutex::new(Vec::<String>::new()));

    // ต้อง clone() ตัวแปรที่ต้องใช้ทุกครั้งก่อนสร้าง closure ให้ handler แต่ละตัว
    let inventory_for_list = inventory.clone();
    let list_handler = move || async move {
        let items = inventory_for_list.lock().unwrap();
        format!("{items:?}")
    };

    let inventory_for_add = inventory.clone();
    let add_handler = move || async move {
        inventory_for_add.lock().unwrap().push("สินค้าใหม่".to_string());
        "added".to_string()
    };

    let _app: Router = Router::new()
        .route("/list", get(list_handler))
        .route("/add", get(add_handler));
}
```

ตอนที่แอปมี state แค่ตัวเดียว (`inventory` ตัวเดียว) pattern นี้พอไหว แต่ลองนึกภาพแอปจริงที่ต้องแชร์**หลายก้อน
ข้อมูล**พร้อมกัน — ตัวอย่างเช่นระบบจองตั๋วจริงที่ Part 63 เริ่มสร้างไว้ ถ้าขยายต่อจะมีอย่างน้อย:

1. **Ticket store** (`Arc<Mutex<HashMap<u32, Ticket>>>`) — ข้อมูลตั๋วที่จองแล้ว
2. **Database connection pool** (`Arc<PgPool>` ที่ Part 70 จะสอนเต็มรูปแบบ) — connection ไปยัง PostgreSQL
3. **Config** (`Arc<AppConfig>`) — ค่าคงที่เช่น จำนวนที่นั่งสูงสุดต่อการจอง, ชื่อเว็บไซต์
4. **Cache** (`Arc<Mutex<HashMap<String, CachedResult>>>`) — cache ผลลัพธ์ query ที่แพง

แต่ handler แต่ละตัวไม่ได้ต้องการทุกอย่างพร้อมกัน — `list_tickets` ต้องการแค่ ticket store, `health_check`
ต้องการแค่ config (ดู site name), `search_events` ต้องการทั้ง database pool และ cache แต่ไม่ต้องการ ticket
store เลย ถ้ายังใช้ pattern "clone ตัวแปรที่ต้องใช้เข้า closure ตรง ๆ" คุณจะเจอปัญหาสามข้อพร้อมกัน:

**1. Boilerplate ทวีคูณตามจำนวน handler × จำนวน state** — ทุก handler ที่ต้องการ state ตัวไหน ต้อง `.clone()`
ตัวแปรนั้นก่อนสร้าง closure เสมอ ถ้ามี 4 ก้อน state และ 15 handler ในแอปจริง คุณอาจต้องเขียน `.clone()` ซ้ำ ๆ
เป็นสิบ ๆ ครั้งกระจายอยู่ทั่วไฟล์ `main.rs` — โค้ดที่ควรจะอ่านง่าย (นี่คือจุด setup ของแอป) กลับรกด้วยการ clone
ตัวแปรก่อนสร้าง handler แต่ละตัว

**2. Handler function ธรรมดาเขียนไม่ได้ ต้องเป็น closure เท่านั้น** — สังเกตในโค้ดตัวอย่างข้างบนว่า handler ต้อง
เป็น closure (`move || async move { ... }`) เพื่อ capture ตัวแปรได้ ถ้าอยากแยก handler ออกเป็น `async fn`
ธรรมดา (ซึ่งอ่านง่ายกว่า, test ง่ายกว่า, ใส่ doc comment ได้ปกติ) คุณทำไม่ได้เลยด้วย pattern นี้ เพราะ `async
fn` ธรรมดารับ parameter ตามที่ signature กำหนดเท่านั้น ไม่มีทาง "capture" ตัวแปรจากภายนอกแบบ closure

**3. เพิ่ม state ใหม่ = ต้องแก้ทุก handler ที่เกี่ยวข้องด้วยมือ** — ถ้าวันหนึ่งต้องเพิ่ม cache เข้าไปในระบบ
คุณต้องไปหาทุก handler ที่ต้องใช้ cache แล้วเพิ่มขั้นตอน clone ตัวแปร cache เข้า closure ของมันทีละตัว ไม่มี
โครงสร้างกลางที่บอกว่า "แอปนี้มี state อะไรทั้งหมด" — ข้อมูลกระจัดกระจายอยู่ในตัวแปร local ของ `main()`

Axum แก้ปัญหาทั้งสามข้อนี้ด้วยแพทเทิร์นเดียว: รวมทุกอย่างที่ต้องแชร์ไว้ใน **struct เดียว** ที่มักเรียกว่า
`AppState` แล้วให้ handler แต่ละตัว**ประกาศใน signature ตรง ๆ**ว่าต้องการ state (ทั้งหมดหรือบางส่วน) ผ่าน
extractor ที่ชื่อ `State<T>` — Axum เป็นคน "เสิร์ฟ" state ตัวนี้ให้ handler ทุกตัวโดยอัตโนมัติตอนมี request
เข้ามา ไม่ต้อง clone ผ่าน closure ด้วยมือเลยแม้แต่บรรทัดเดียว

### 64.2 `State<T>` Extractor และ `Router::with_state()`

#### โครงสร้างพื้นฐาน

แพทเทิร์นมาตรฐานมีสามส่วน:

1. นิยาม struct ที่รวม state ทั้งหมดของแอป (ตามธรรมเนียมชื่อ `AppState`)
2. สร้าง instance ของมันหนึ่งตัวตอนเริ่มโปรแกรม
3. ผูกมันเข้ากับ `Router` ด้วย `.with_state(state)` — จากนั้น handler ตัวไหนก็ตามที่มี parameter เป็น
   `State<AppState>` จะได้รับ**สำเนา**ของ state ตัวนี้โดยอัตโนมัติทุกครั้งที่มี request เข้ามา

มาเขียนตัวอย่างจริง — ระบบจองตั๋ว (ต่อยอดจากตัวอย่างที่ Part 63 เริ่มไว้) ที่มี CRUD เต็มรูปแบบผ่าน `AppState`
เดียว:

```rust
use axum::{
    extract::{Path, State},
    routing::get,
    Json, Router,
};
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

#[derive(Clone, Serialize, Deserialize)]
struct Ticket {
    id: u32,
    event: String,
    seats: u32,
}

// ข้อมูลที่ "แก้ไขได้" ทั้งหมดของระบบตั๋วอยู่รวมกันในจุดเดียว หลัง Mutex เดียว
#[derive(Default)]
struct TicketStore {
    tickets: Mutex<HashMap<u32, Ticket>>,
    next_id: Mutex<u32>,
}

// AppState คือสิ่งเดียวที่ handler ทุกตัวขอผ่าน State<AppState>
#[derive(Clone)]
struct AppState {
    store: Arc<TicketStore>,
}

#[derive(Deserialize)]
struct NewTicket {
    event: String,
    seats: u32,
}

// handler ธรรมดา -- ไม่ต้องเป็น closure, ไม่ต้อง .clone() อะไรก่อนสร้างมันเลย
// แค่ประกาศว่า "ฉันต้องการ AppState" ผ่าน parameter ตัวแรก
async fn list_tickets(State(state): State<AppState>) -> Json<Vec<Ticket>> {
    let tickets = state.store.tickets.lock().unwrap();
    Json(tickets.values().cloned().collect())
}

async fn create_ticket(
    State(state): State<AppState>,
    Json(body): Json<NewTicket>,
) -> Json<Ticket> {
    let mut next_id = state.store.next_id.lock().unwrap();
    let id = *next_id;
    *next_id += 1;

    let ticket = Ticket { id, event: body.event, seats: body.seats };
    state.store.tickets.lock().unwrap().insert(id, ticket.clone());
    Json(ticket)
}

async fn get_ticket(
    State(state): State<AppState>,
    Path(id): Path<u32>,
) -> Result<Json<Ticket>, axum::http::StatusCode> {
    state
        .store
        .tickets
        .lock()
        .unwrap()
        .get(&id)
        .cloned()
        .map(Json)
        .ok_or(axum::http::StatusCode::NOT_FOUND)
}

#[tokio::main]
async fn main() {
    let state = AppState { store: Arc::new(TicketStore::default()) };

    let app = Router::new()
        .route("/tickets", get(list_tickets).post(create_ticket))
        .route("/tickets/{id}", get(get_ticket))
        .with_state(state); // <- จุดที่ผูก AppState เข้ากับ Router ทั้งตัว

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3064").await.unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}
```

ทดสอบด้วย `curl` จริง (รันโปรแกรมแล้วยิง request ตามลำดับ):

```bash
$ curl -s http://127.0.0.1:3064/tickets
[]

$ curl -s -X POST http://127.0.0.1:3064/tickets \
    -H 'Content-Type: application/json' \
    -d '{"event":"Rust Conf 2026","seats":100}'
{"id":0,"event":"Rust Conf 2026","seats":100}

$ curl -s -X POST http://127.0.0.1:3064/tickets \
    -H 'Content-Type: application/json' \
    -d '{"event":"Thai Rust Meetup","seats":40}'
{"id":1,"event":"Thai Rust Meetup","seats":40}

$ curl -s http://127.0.0.1:3064/tickets
[{"id":0,"event":"Rust Conf 2026","seats":100},{"id":1,"event":"Thai Rust Meetup","seats":40}]

$ curl -s http://127.0.0.1:3064/tickets/0
{"id":0,"event":"Rust Conf 2026","seats":100}

$ curl -s -i http://127.0.0.1:3064/tickets/99
HTTP/1.1 404 Not Found
content-length: 0
date: Sat, 26 Sep 2026 23:28:04 GMT
```

ทุกอย่างทำงานตรงตามที่คาดไว้ — และสิ่งสำคัญที่สุดที่ต้องสังเกตคือ **`list_tickets`, `create_ticket`,
`get_ticket` ไม่มีการ `.clone()` ตัวแปรใด ๆ ด้วยมือเลย** ไม่มี closure ห่อ ไม่ต้อง capture ตัวแปรจากภายนอก —
ทุกตัวเป็น `async fn` ธรรมดาที่ประกาศชัดเจนใน signature ว่าต้องการอะไรบ้าง (`State<AppState>`, `Path<u32>`,
`Json<NewTicket>`) และ Axum เป็นคนจัดหาให้เองทั้งหมดตอน dispatch request

#### `State<T>` ทำงานอย่างไรภายใน (แนวคิดระดับสูง)

ตอนคุณเรียก `.with_state(state)` จริง ๆ แล้ว Axum เก็บ `state` ไว้เป็นส่วนหนึ่งของ `Router` (ผ่าน type
parameter `Router<S>` ที่ Part 62-63 อาจแนะนำผ่าน ๆ มาแล้ว) พอมี request เข้ามาตรงกับ route ไหน ก่อนเรียก
handler จริง Axum จะ:

1. ดูว่า handler นั้นมี parameter ชนิด `State<AppState>` อยู่หรือไม่ (ผ่านการ dispatch แบบ generic ที่ผูกกับ
   trait `Handler` — Part 63 อาจแนะนำ trait นี้มาแล้วผ่าน ๆ)
2. ถ้ามี ก็ `state.clone()` (เรียก `Clone::clone()` บน `AppState` ที่คุณเก็บไว้) แล้วห่อผลลัพธ์เป็น `State(...)`
   ส่งให้ handler เป็น argument ตัวนั้น

นี่คือเหตุผลที่ `AppState` **ต้อง implement `Clone`** เสมอ — ทุกครั้งที่มี request ที่ handler ต้องการ state
เข้ามา จะมีการ `clone()` เกิดขึ้นหนึ่งครั้ง (ไม่ใช่ share reference ตรง ๆ แบบ `&AppState` เพราะ handler แต่ละตัว
รันเป็น task async แยกกัน อาจ concurrent กัน — การส่งค่าที่ own ไปให้แต่ละ task ปลอดภัยกว่าการยืม reference
ข้าม `.await` point ตามที่ Part 46-50 อธิบายไว้เรื่อง lifetime ของ future) — หัวข้อถัดไปจะอธิบายว่าทำไมการ
`clone()` นี้ถึง"ถูกมาก"ถ้าออกแบบ `AppState` ถูกวิธี

### 64.3 ทำไม `AppState` ต้องห่อด้วย `Arc`

#### ต้นทุนของการ `Clone` — เชื่อมกับตัวเลขจริงจาก Part 56

Part 56 (Memory Management ขั้นสูงและ Zero-cost Abstractions) ได้เคยวัดความต่างระหว่าง `Arc::clone()`
(เพิ่ม atomic reference count หนึ่งครั้ง — เป็น operation ที่เร็วมาก คงที่ไม่ว่าข้อมูลภายในจะใหญ่แค่ไหน) กับ
การ deep-clone โครงสร้างข้อมูลขนาดใหญ่ (ที่ต้อง allocate memory ใหม่และ copy ข้อมูลทุก byte — ต้นทุนโตตาม
ขนาดข้อมูล) ไว้แล้วว่าต่างกันหลายลำดับขนาดสำหรับข้อมูลที่มีขนาดใหญ่พอสมควร

ทุก request ที่เข้ามาที่ handler ซึ่งต้องการ `State<AppState>` จะทำให้ Axum เรียก `state.clone()` หนึ่งครั้ง —
ในระบบเว็บจริงที่รับ**หลายพัน request ต่อวินาที** ถ้า `.clone()` ตัวนี้แพง (เพราะ `AppState` deep-clone
`HashMap` ที่มีข้อมูลเป็นหมื่นรายการทุกครั้ง) ระบบจะช้าลงอย่างมีนัยสำคัญ แถมยังมีปัญหาซ้อนอีกชั้น: ถ้า
`AppState` เก็บ `HashMap` ตรง ๆ (ไม่ใช่ผ่าน `Arc`) การ clone แต่ละครั้งจะได้ **สำเนาข้อมูลที่แยกจากกันโดย
สิ้นเชิง** — handler ตัวหนึ่งเขียนข้อมูลเพิ่มเข้าไปใน `HashMap` ของสำเนาตัวเอง จะ**ไม่มีผล**กับสำเนาที่ handler
ตัวอื่นเห็นเลย ระบบจะดูเหมือน "เขียนข้อมูลไปแล้ว แต่พอ query กลับมาไม่เจอ" ซึ่งเป็นบั๊กที่ทำให้งงมาก

การห่อทุกอย่างที่ต้องแชร์**จริง ๆ**ด้วย `Arc` (แบบที่ตัวอย่างในหัวข้อ 64.2 ทำกับ `TicketStore`) แก้ทั้งสอง
ปัญหาพร้อมกัน:

- **`.clone()` ถูกมาก** — clone แค่ pointer + เพิ่ม reference count เป็น O(1) เสมอ ไม่ว่า `TicketStore`
  ภายในจะมีข้อมูลกี่รายการก็ตาม (ตรงกับที่ Part 56 วัดไว้)
- **ข้อมูลจริง ๆ มีแค่ก้อนเดียวที่ทุก clone ชี้ไปที่เดียวกัน** — handler ตัวหนึ่งเขียนผ่าน `Mutex::lock()`
  แล้ว handler ตัวอื่น (ที่ได้ `AppState` เป็นอีก clone หนึ่ง) จะเห็นข้อมูลที่เปลี่ยนไปทันที เพราะ `Arc` ชี้ไป
  ที่ heap allocation เดียวกันเสมอ

#### รูปแบบที่ถูกต้อง: `Arc` ครอบทั้งก้อน ไม่ใช่ครอบทีละ field

สังเกตในตัวอย่าง 64.2 ว่า `AppState` มี field เดียวคือ `store: Arc<TicketStore>` และ `TicketStore` (ข้างใน)
ค่อยมี `Mutex` แยกทีละ field อีกที — นี่คือรูปแบบที่แนะนำ: **`Arc` ห่อรอบ struct ใหญ่ทั้งก้อนเพียงครั้งเดียว**
แทนที่จะห่อ `Arc` แยกทีละ field เล็ก ๆ (เช่น `Arc<Mutex<HashMap<...>>>` กับ `Arc<Mutex<u32>>` แยกกันสองตัว) —
ทั้งสองแบบทำงานถูกต้อง แต่แบบ "ห่อทั้งก้อนครั้งเดียว" จัดการง่ายกว่าตอนแอปมี state หลายสิบ field เพราะ
`.clone()` ที่เกิดตอน dispatch request มีแค่ครั้งเดียว (clone `AppState` ทั้งก้อน) ไม่ใช่ต้อง clone `Arc`
แยกทีละตัว

#### สิ่งที่เกิดขึ้นจริงถ้าลืม `#[derive(Clone)]`

มาดู compiler error จริงที่เกิดขึ้นถ้าลืมสิ่งที่อธิบายไปทั้งหมดนี้ — เริ่มจากลืม `#[derive(Clone)]` บน
`AppState`:

```rust
use axum::{extract::State, routing::get, Router};
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

// สังเกต: ไม่มี #[derive(Clone)] บน AppState ตัวนี้
struct AppState {
    inventory: Arc<Mutex<HashMap<String, u32>>>,
}

async fn handler(State(_state): State<AppState>) -> &'static str {
    "ok"
}

#[tokio::main]
async fn main() {
    let state = AppState { inventory: Arc::new(Mutex::new(HashMap::new())) };
    let app = Router::new().route("/", get(handler)).with_state(state);
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3065").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

`cargo build` ให้ผลลัพธ์จริงดังนี้ (ตัดบางส่วนที่ซ้ำออก):

```
error[E0277]: the trait bound `fn(State<AppState>) -> impl Future<Output = &'static str> {handler}: Handler<_, _>` is not satisfied
   --> src/bin/forgot_clone.rs:18:44
    |
 18 |     let app = Router::new().route("/", get(handler)).with_state(state);
    |                                        --- ^^^^^^^ the trait `Handler<_, _>` is not implemented for fn item `fn(State<AppState>) -> impl Future<Output = &'static str> {handler}`
    |                                        |
    |                                        required by a bound introduced by this call
    |
    = note: Consider using `#[axum::debug_handler]` to improve the error message
note: required by a bound in `axum::routing::get`
   ...

error[E0277]: the trait bound `AppState: Clone` is not satisfied
   --> src/bin/forgot_clone.rs:18:15
    |
 18 |     let app = Router::new().route("/", get(handler)).with_state(state);
    |               ^^^^^^^^^^^^^ the trait `Clone` is not implemented for `AppState`
    |
note: required by a bound in `Router::<S>::new`
   --> .../axum-0.8.9/src/routing/mod.rs:140:8
    |
140 |     S: Clone + Send + Sync + 'static,
    |        ^^^^^ required by this bound in `Router::<S>::new`
    ...
help: consider annotating `AppState` with `#[derive(Clone)]`
    |
  6 + #[derive(Clone)]
  7 | struct AppState {
    |

error[E0277]: the trait bound `AppState: Clone` is not satisfied
   --> src/bin/forgot_clone.rs:18:40
    |
 18 |     let app = Router::new().route("/", get(handler)).with_state(state);
    |                                        ^^^^^^^^^^^^ the trait `Clone` is not implemented for `AppState`
    |
help: consider annotating `AppState` with `#[derive(Clone)]`
    ...

error[E0277]: the trait bound `AppState: Clone` is not satisfied
   --> src/bin/forgot_clone.rs:18:54
    |
 18 |     let app = Router::new().route("/", get(handler)).with_state(state);
    |                                                      ^^^^^^^^^^ the trait `Clone` is not implemented for `AppState`
    |
note: required by a bound in `Router::<S>::with_state`
   --> .../axum-0.8.9/src/routing/mod.rs:140:8
    |
140 |     S: Clone + Send + Sync + 'static,
    |        ^^^^^ required by this bound in `Router::<S>::with_state`
    ...
help: consider annotating `AppState` with `#[derive(Clone)]`
```

สังเกตสองอย่าง: (1) error เกิด**หลายจุดพร้อมกัน** (`Router::new()`, `.route()`, `.with_state()`) เพราะ
**ทุกเมธอดของ `Router<S>` ต้องการ `S: Clone + Send + Sync + 'static`** เสมอ ไม่ใช่แค่ `.with_state()`
เท่านั้น — บอกได้ว่า requirement นี้อยู่ในระดับ type parameter ของ `Router` เอง ไม่ใช่แค่จุดเดียว (2) compiler
**บอกวิธีแก้ตรง ๆ** ผ่าน `help:` — เพิ่ม `#[derive(Clone)]` ให้ `AppState` แก้ปัญหาได้ทันที นี่คือหนึ่งใน
ตัวอย่างที่ Rust compiler ให้ error message ที่ actionable มากที่สุด

#### สิ่งที่เกิดขึ้นจริงถ้า `derive(Clone)` แล้ว แต่ "ลืม" ห่อ `Arc`

ทีนี้สมมติว่าคุณจำได้ว่าต้อง `#[derive(Clone)]` แต่ลืมว่าทำไมต้องมี `Arc` เลยเก็บ `Mutex` ไว้ตรง ๆ ใน struct
โดยไม่ห่อ `Arc`:

```rust
use axum::{extract::State, routing::get, Router};
use std::collections::HashMap;
use std::sync::Mutex;

// derive(Clone) แล้ว แต่ "ลืม" ห่อ Mutex ด้วย Arc — เก็บ Mutex ตรง ๆ ใน struct
#[derive(Clone)]
struct AppState {
    inventory: Mutex<HashMap<String, u32>>,
}
# async fn handler(State(_state): State<AppState>) -> &'static str { "ok" }
```

`cargo build` ให้ error ที่**ต่างจากข้อก่อนหน้าโดยสิ้นเชิง** — คราวนี้สั้นและชี้เป้าตรงจุดกว่ามาก:

```
error[E0277]: the trait bound `std::sync::Mutex<HashMap<String, u32>>: Clone` is not satisfied
 --> src/bin/forgot_arc.rs:8:5
  |
6 | #[derive(Clone)]
  |          ----- in this derive macro expansion
7 | struct AppState {
8 |     inventory: Mutex<HashMap<String, u32>>,
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ the trait `Clone` is not implemented for `std::sync::Mutex<HashMap<String, u32>>`
```

นี่คือ error ที่เกิดที่**การ derive** ไม่ใช่ที่จุดเรียก `.with_state()` เหมือนข้อก่อน — เหตุผลคือ `#[derive
(Clone)]` ทำงานโดยกำหนดว่า struct จะ `Clone` ได้ก็ต่อเมื่อ**ทุก field**ของมัน `Clone` ได้ (macro ที่ Part 44
สอนไว้เรื่อง derive macro ทำงานแบบนี้เป๊ะ — generate `impl Clone for AppState { fn clone(&self) -> Self {
Self { inventory: self.inventory.clone() } } }` แล้วปล่อยให้ compiler เช็คว่าทุก `.clone()` ข้างในถูก
implement จริงไหม) และ **`std::sync::Mutex<T>` (รวมถึง `tokio::sync::Mutex<T>`) ไม่ implement `Clone` เลย
ไม่ว่า `T` จะเป็นอะไรก็ตาม** — เหตุผลเชิงออกแบบคือ `Mutex` มีไว้ป้องกันการเข้าถึงข้อมูล**เดียวกัน**พร้อมกัน
จากหลาย thread; การ "clone a mutex" จะสร้าง lock ใหม่ที่แยกจากตัวเดิมโดยสมบูรณ์ ทำให้ mutual exclusion ที่
ตั้งใจไว้พังทันที (สอง handler ที่ควรแย่ง lock ตัวเดียวกัน จะกลายเป็นแย่ง lock คนละตัวที่ไม่รู้จักกัน) — Rust
เลือกไม่ implement `Clone` ให้ `Mutex` เลยเพื่อป้องกัน bug ประเภทนี้ตั้งแต่ compile time

**นี่คือเหตุผลที่แท้จริงที่ `Arc` ต้องมา**: `Arc<Mutex<T>>` implement `Clone` ได้ (เพราะ `Arc::clone()` แค่
เพิ่ม reference count ไม่ได้ clone ข้อมูลภายใน) และผลลัพธ์คือทุก clone ของ `Arc<Mutex<T>>` ยังคง**ชี้ไปที่
`Mutex` ตัวเดียวกัน** — mutual exclusion ที่ตั้งใจไว้จึงยังทำงานถูกต้อง 100% วิธีแก้คือกลับไปห่อ `Arc` รอบ
`Mutex` (หรือรอบ struct ที่มี `Mutex` ข้างใน) แบบตัวอย่างในหัวข้อ 64.2:

```rust
use std::sync::{Arc, Mutex};
use std::collections::HashMap;

#[derive(Clone)]
struct AppState {
    inventory: Arc<Mutex<HashMap<String, u32>>>, // ห่อด้วย Arc แล้ว -- Clone ได้ถูกต้อง
}
```

#### สิ่งที่เกิดขึ้นจริงถ้า "ลืม" เรียก `.with_state()` เลย

กับดักที่สามที่พบบ่อยไม่แพ้กัน: เขียน handler ที่ขอ `State<AppState>` ถูกต้อง สร้าง state ถูกต้อง แต่ลืมต่อ
`.with_state(state)` เข้ากับ `Router` เลย:

```rust
use axum::{extract::State, routing::get, Router};
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

#[derive(Clone)]
struct AppState {
    inventory: Arc<Mutex<HashMap<String, u32>>>,
}

async fn handler(State(_state): State<AppState>) -> &'static str {
    "ok"
}

#[tokio::main]
async fn main() {
    let _state = AppState { inventory: Arc::new(Mutex::new(HashMap::new())) };

    // ไม่ระบุ type ให้ app เลย ปล่อยให้ compiler infer เอง แล้ว "ลืม" .with_state(state)
    let app = Router::new().route("/", get(handler));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3068").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

ตรงนี้น่าสนใจ: `Router::new().route("/", get(handler))` เอง **compile ผ่าน** — เพราะ Axum ยัง"ไม่รู้"ว่า
`AppState` ตัวจริงหน้าตาเป็นอย่างไร มันแค่ต้องการ `S: Clone + Send + Sync + 'static` (ตัวแปร generic ใด ๆ
ที่สนอง trait bound พวกนี้ก็พอ) — error จริง ๆ ไปเกิดตอนพยายามเอา `Router<AppState>` (ที่ยังไม่ถูก
`.with_state()` แปลงเป็น `Router<()>`) ไปส่งให้ `axum::serve()`:

```
error[E0277]: the trait bound `for<'a> Router<AppState>: tower_service::Service<IncomingStream<'a, tokio::net::TcpListener>>` is not satisfied
   --> src/bin/forgot_with_state2.rs:22:27
    |
 22 |     axum::serve(listener, app).await.unwrap();
    |     -----------           ^^^ the trait `for<'a> tower_service::Service<IncomingStream<'a, tokio::net::TcpListener>>` is not implemented for `Router<AppState>`
    |
help: the following other types implement trait `tower_service::Service<Request>`
   --> .../axum-0.8.9/src/routing/mod.rs:549:5
    |
549 | /     impl<L> Service<serve::IncomingStream<'_, L>> for Router<()>
    | |___________________________^ `Router` implements `tower_service::Service<IncomingStream<'_, L>>`
    ...

error[E0277]: `Serve<tokio::net::TcpListener, Router<AppState>, _>` is not a future
   --> src/bin/forgot_with_state2.rs:22:32
    |
 22 |     axum::serve(listener, app).await.unwrap();
    |     -------------------------- ^^^^^ `Serve<tokio::net::TcpListener, Router<AppState>, _>` is not a future
```

จุดที่ต้องเข้าใจให้ลึก: `Router<S>` เป็น**generic type** ที่ `S` แทน "ชนิดของ state ที่ยังไม่ถูกผูก" —
`Router::new()` เริ่มต้นเป็น `Router<AppState>` (เพราะ handler ที่ผูกด้วย `.route()` บอกใบ้ชนิด state ที่
ต้องการ) และมีแค่ **`Router<()>`** (คือ `S = ()` แปลว่า "ไม่มี state ที่ยังไม่ผูกเหลืออยู่แล้ว") เท่านั้นที่
implement `tower_service::Service` — ซึ่งเป็น trait ที่จำเป็นสำหรับส่งให้ `axum::serve()` รัน หน้าที่ของ
`.with_state(state)` คือแปลง `Router<AppState>` เป็น `Router<()>` (ด้วยการฝัง `state` ที่ให้มาเข้าไปข้างใน
ให้ Axum ใช้ตอน dispatch) — ถ้าไม่เรียกมัน `Router` จะติดอยู่ที่ `Router<AppState>` ตลอด และไม่มีทางเอาไป
`serve()` ได้ วิธีแก้คือเติม `.with_state(state)` กลับเข้าไป (และห้ามลืมใช้ `state` จริง ไม่ใช่ `_state` ที่
ไม่ได้ใช้งาน):

```rust
let app = Router::new().route("/", get(handler)).with_state(state); // ใส่ state จริง ไม่มี underscore
```

### 64.4 Substates ด้วย `FromRef`: แยก State ตามกลุ่ม Route

พอแอปโตขึ้น `AppState` มักจะมีหลาย field ที่**ไม่ใช่ทุก handler ต้องการทั้งหมด** — เช่นแอปที่มีทั้ง config
(อ่านอย่างเดียว ไม่ต้อง lock) และตัวนับที่ต้อง mutate ผ่าน `Mutex` route กลุ่มหนึ่ง (เช่น endpoint แสดงข้อมูล
เว็บไซต์) อาจต้องการแค่ config route อีกกลุ่ม (เช่น endpoint จองตั๋ว) อาจต้องการแค่ตัวนับ — การให้ทุก handler
รับ `State<AppState>` เต็มก้อนแล้วเข้าไปหยิบ field ที่ตัวเองต้องการเองก็ใช้ได้ แต่ signature ของ handler จะ
"โกหก" — บอกว่าต้องการ `AppState` ทั้งก้อน ทั้งที่จริง ๆ ใช้แค่ field เดียว ทำให้อ่านโค้ดแล้วเดายากว่า handler
ตัวนี้พึ่งพาอะไรบ้าง

Axum แก้ปัญหานี้ด้วย trait **`FromRef<S>`** — บอกว่า "จาก state ก้อนใหญ่ `S` ดึง substate ชนิดนี้ออกมาได้
อย่างไร" พอ implement ให้ครบ handler แต่ละตัวสามารถขอ `State<Substate>` (ไม่ใช่ `State<AppState>` เต็มก้อน)
ได้ตรง ๆ:

```rust
use axum::{
    extract::{FromRef, State},
    routing::get,
    Router,
};
use std::sync::{Arc, Mutex};

// --- ส่วนที่ 1: config อ่านอย่างเดียว ไม่ต้อง lock อะไรเลย ---
#[derive(Clone)]
struct AppConfig {
    site_name: String,
    max_seats_per_booking: u32,
}

// --- ส่วนที่ 2: ตัวนับการจองที่ handler กลุ่ม booking ต้องแก้ไขได้ ---
#[derive(Clone)]
struct BookingCounter {
    count: Arc<Mutex<u32>>,
}

// --- AppState ใหญ่ที่รวมทุกอย่าง ---
#[derive(Clone)]
struct AppState {
    config: AppConfig,
    booking_counter: BookingCounter,
}

// บอก Axum ว่า "จาก AppState ตัวเต็ม ดึง AppConfig ออกมายังไง"
impl FromRef<AppState> for AppConfig {
    fn from_ref(state: &AppState) -> AppConfig {
        state.config.clone()
    }
}

// บอก Axum ว่า "จาก AppState ตัวเต็ม ดึง BookingCounter ออกมายังไง"
impl FromRef<AppState> for BookingCounter {
    fn from_ref(state: &AppState) -> BookingCounter {
        state.booking_counter.clone()
    }
}

// handler นี้รู้จักแค่ AppConfig เท่านั้น ไม่รู้เรื่อง booking_counter เลย
async fn site_info(State(config): State<AppConfig>) -> String {
    format!(
        "ยินดีต้อนรับสู่ {} (จองได้สูงสุด {} ที่นั่ง/ครั้ง)",
        config.site_name, config.max_seats_per_booking
    )
}

// handler นี้รู้จักแค่ BookingCounter เท่านั้น ไม่รู้เรื่อง config เลย
async fn booking_count(State(counter): State<BookingCounter>) -> String {
    let mut count = counter.count.lock().unwrap();
    *count += 1;
    format!("มีการจองรวมทั้งหมด {} ครั้งแล้ว (นับครั้งนี้ด้วย)", *count)
}

#[tokio::main]
async fn main() {
    let state = AppState {
        config: AppConfig {
            site_name: "Rust Ticket".to_string(),
            max_seats_per_booking: 4,
        },
        booking_counter: BookingCounter { count: Arc::new(Mutex::new(0)) },
    };

    // Router<AppState> เดียว แต่ handler แต่ละตัว extract คนละ substate กัน
    // ทำได้เพราะ AppConfig: FromRef<AppState> และ BookingCounter: FromRef<AppState>
    let app = Router::new()
        .route("/site-info", get(site_info))
        .route("/booking-count", get(booking_count))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3069").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

ทดสอบจริง:

```bash
$ curl -s http://127.0.0.1:3069/site-info
ยินดีต้อนรับสู่ Rust Ticket (จองได้สูงสุด 4 ที่นั่ง/ครั้ง)

$ curl -s http://127.0.0.1:3069/booking-count
มีการจองรวมทั้งหมด 1 ครั้งแล้ว (นับครั้งนี้ด้วย)

$ curl -s http://127.0.0.1:3069/booking-count
มีการจองรวมทั้งหมด 2 ครั้งแล้ว (นับครั้งนี้ด้วย)

$ curl -s http://127.0.0.1:3069/booking-count
มีการจองรวมทั้งหมด 3 ครั้งแล้ว (นับครั้งนี้ด้วย)
```

สังเกตว่า `AppState` ยังมีตัวเดียว ผูกกับ `Router` ครั้งเดียวด้วย `.with_state()` ครั้งเดียว — ไม่ต้องแยก
`Router` ออกเป็นหลายตัวแล้วค่อย `.merge()` กันเพื่อให้แต่ละกลุ่ม route เห็น state ต่างกัน (แม้ว่าจะทำแบบนั้น
ได้เหมือนกัน และมีประโยชน์เวลาต้องแยก module ทั้งกลุ่ม) `FromRef` ทำให้แต่ละ **handler** เลือกได้เองว่าจะขอ
มุมมองไหนของ state ใหญ่ โดยที่ signature ของ handler สื่อสารตรง ๆ ว่ามันพึ่งพาอะไรจริง ๆ — เป็นประโยชน์มาก
ตอนอ่านโค้ดทีหลัง หรือตอนต้องเขียน unit test ที่ mock แค่ substate เดียวโดยไม่ต้องประกอบ `AppState` เต็มก้อน

ข้อสังเกตเชิงเทคนิค: มี `impl<S> FromRef<S> for S` (identity) มาให้กับทุก type อยู่แล้วในตัว Axum เอง (ทุก
state เป็น substate ของตัวเองได้เสมอ) — นี่คือเหตุผลที่ `State<AppState>` ตรง ๆ (แบบหัวข้อ 64.2) ก็ยังใช้ได้
ปกติควบคู่กับ substate อื่น ๆ ในแอปเดียวกัน ไม่ขัดกัน

### 64.5 ระบบ Extractor เชิงลึก: `FromRequestParts` vs `FromRequest`

Part 63 สอนกฎ "extractor ที่กิน request body (เช่น `Json<T>`) ต้องมาเป็น parameter ตัวสุดท้ายในลิสต์ของ
handler เท่านั้น" ไว้แบบท่องจำ — ตอนนี้มาดูว่ากฎนี้มาจากไหนจริง ๆ ในระดับ trait

Axum นิยาม extractor ผ่านสอง trait หลัก:

```rust
// (นี่คือรูปแบบย่อของ trait จริงใน axum-core เพื่อให้เห็นภาพ ไม่ต้องพิมพ์ตามนี้ตรง ๆ)
pub trait FromRequestParts<S>: Sized {
    type Rejection: axum::response::IntoResponse;

    fn from_request_parts(
        parts: &mut axum::http::request::Parts,
        state: &S,
    ) -> impl std::future::Future<Output = Result<Self, Self::Rejection>> + Send;
}

pub trait FromRequest<S>: Sized {
    type Rejection: axum::response::IntoResponse;

    fn from_request(
        req: axum::extract::Request,
        state: &S,
    ) -> impl std::future::Future<Output = Result<Self, Self::Rejection>> + Send;
}
```

ความต่างที่สำคัญที่สุดอยู่ที่ **argument ตัวแรก**:

- **`FromRequestParts<S>`** รับ `&mut Parts` — `Parts` คือส่วนของ HTTP request ที่**ไม่รวม body** (method,
  URI, headers, extensions, version) การอ่าน `Parts` ไม่ทำลายอะไรเลย เรียกได้หลายครั้งโดยไม่มีปัญหา — นี่คือ
  extractor กลุ่ม `Path`, `Query`, `State`, `HeaderMap`, `Extension` (ที่จะเจอในหัวข้อ 64.7), และ custom
  extractor ที่กำลังจะเขียนในหัวข้อถัดไป
- **`FromRequest<S>`** รับ `Request` (หรือ `axum::extract::Request`) **ทั้งตัว รวม body** — และการอ่าน
  body ของ HTTP request (ซึ่งภายในเป็น async stream ของ byte ที่ไหลมาทาง network) เป็นการ**บริโภคทิ้ง**
  (consume) — อ่านครั้งเดียวจบ ไม่มีทาง "อ่านซ้ำ" ได้ เหมือนการอ่าน `Iterator` ที่พออ่านผ่านไปแล้ว
  ต้องเริ่มใหม่ไม่ได้ (concept เดียวกับที่ Part 25-26 สอนเรื่อง `Iterator` ที่ consume ตัวเองไปทีละ item) —
  นี่คือ extractor กลุ่ม `Json<T>`, `Bytes`, `String`, `Form<T>`

ตอนนี้กฎ "body extractor ต้องมาตัวสุดท้าย" ก็สมเหตุสมผลในระดับ mechanism แล้ว: Axum dispatch handler โดย
เดิน**ทีละ parameter ตามลำดับที่เขียนไว้ใน signature** — parameter แต่ละตัวที่เป็น `FromRequestParts` จะถูก
extract จาก `&mut Parts` ไปเรื่อย ๆ (หยิบแค่ metadata ไม่แตะ body เลย ทำกี่ตัวก็ได้ไม่มีปัญหา) จนกว่าจะเจอ
parameter ตัวแรกที่เป็น `FromRequest` (กิน body ทั้งตัว) — ตัวนั้นต้องเป็น**ตัวสุดท้าย** เพราะหลังจากมันกิน
body ไปแล้ว **ไม่มี body เหลือให้ extractor ตัวต่อไปอ่านอีก** ถ้าคุณเขียน parameter อีกตัวหลังจากนั้น (ไม่ว่า
จะเป็น `FromRequestParts` หรือ `FromRequest` อีกตัว) มันจะไม่มีอะไรให้อ่านอย่างสมเหตุสมผล — Axum จึงปฏิเสธ
ตั้งแต่ compile time ไม่ปล่อยให้เขียนโค้ดแบบนี้ผ่านไปได้เลย

#### พิสูจน์ด้วย compiler error จริง

```rust
use axum::{extract::Path, routing::post, Json, Router};
use serde::Deserialize;

#[derive(Deserialize)]
struct NewBook {
    title: String,
}

// ผิดลำดับ: Json<T> (FromRequest, กินไป body) มาก่อน Path<T> (FromRequestParts)
async fn create_book(Json(body): Json<NewBook>, Path(shelf_id): Path<u32>) -> String {
    format!("shelf={shelf_id} title={}", body.title)
}
# fn wire_up() -> axum::Router {
#     Router::new().route("/shelves/{shelf_id}/books", post(create_book))
# }
```

`cargo build` ให้ error ทันที ยืนยันว่านี่คือ**compile-time error** ไม่ใช่ปัญหาที่โผล่มาตอน runtime:

```
error[E0277]: the trait bound `fn(Json<NewBook>, Path<u32>) -> ... {create_book}: Handler<_, _>` is not satisfied
   --> src/bin/bad_extractor_order.rs:16:69
    |
 16 |     let app = Router::new().route("/shelves/{shelf_id}/books", post(create_book));
    |                                                                ---- ^^^^^^^^^^^ the trait `Handler<_, _>` is not implemented for fn item `fn(Json<NewBook>, Path<u32>) -> ... {create_book}`
    |
    = note: Consider using `#[axum::debug_handler]` to improve the error message
```

error message นี้เอง "ตรงจุด" น้อยมาก — บอกแค่ว่า `Handler` trait ไม่ implement ให้ฟังก์ชันนี้ ไม่ได้บอกว่า
"ทำไม" แต่ Axum แนะนำวิธีดูให้ชัดขึ้นมาในตัว: attribute macro `#[axum::debug_handler]` (ต้องเปิด feature
`macros` ของ crate `axum` — `cargo add axum --features macros`) ลองแปะไว้เหนือ handler ตัวเดียวกัน:

```rust
#[axum::debug_handler]
async fn create_book(Json(body): Json<NewBook>, Path(shelf_id): Path<u32>) -> String {
    format!("shelf={shelf_id} title={}", body.title)
}
```

คราวนี้ error ชี้เป้าตรงจุดทันที:

```
error: `Json<_>` consumes the request body and thus must be the last argument to the handler function
  --> src/bin/bad_extractor_order.rs:11:34
   |
11 | async fn create_book(Json(body): Json<NewBook>, Path(shelf_id): Path<u32>) -> String {
   |                                  ^^^^
```

**`#[axum::debug_handler]` เป็นเครื่องมือ debug ที่ทรงคุณค่ามาก** ทุกครั้งที่เจอ error `Handler<_, _> is not
implemented` ที่อ่านไม่รู้เรื่อง — แปะ attribute นี้ไว้ก่อนแล้ว compile ใหม่ มันจะแปลง error กว้าง ๆ ให้เป็น
ข้อความที่ชัดเจนเจาะจงปัญหาจริงเสมอ (แนะนำให้แปะไว้ระหว่าง debug แล้วลบออกทีหลังก็ได้ ไม่มีผลต่อ runtime
behavior ของ handler เลย)

### 64.6 เขียน Custom Extractor: `ApiKey` จาก Header

ตอนนี้มาถึงหัวข้อที่ทรงคุณค่าที่สุดของบทนี้ — เขียน extractor ของตัวเองที่ตรวจสอบ custom header เพื่อยืนยัน
ว่าผู้เรียกมี API key ที่ถูกต้อง (foreshadow ระบบ authentication เต็มรูปแบบด้วย JWT ใน Part 74 และ
authorization/RBAC ใน Part 76 — ที่นี่คือเวอร์ชันง่ายที่สุดของแนวคิดเดียวกัน)

Extractor ที่ดีที่สุดคือ extractor ที่ทำให้ handler **ไม่ต้องรู้เรื่อง logic การตรวจสอบเลย** — handler เห็น
แค่ parameter ชนิด `ApiKey` แล้วมั่นใจได้ทันทีว่าถ้าโค้ดใน body ของ handler รันถึง ก็แปลว่า key ผ่านการ
ตรวจสอบแล้วเรียบร้อย (แนวคิดเดียวกับ "typestate pattern" ที่ Part 53 สอนไว้ — ใช้ type system เป็นตัวการันตี
ว่า invariant บางอย่างเป็นจริงแล้ว โดยไม่ต้องเช็คซ้ำด้วยมือใน business logic)

#### ขั้นที่ 1: นิยาม extractor type และ rejection type

```rust
use axum::{
    extract::FromRequestParts,
    http::{request::Parts, StatusCode},
    response::{IntoResponse, Response},
    Json,
};
use serde_json::json;

/// Extractor ที่ดึงและตรวจสอบ header `X-Api-Key` เอง
/// ไม่ต้องแก้ signature ของ handler ที่ใช้มันเลย -- แค่ใส่ `ApiKey` เป็น parameter
struct ApiKey(String);

/// Rejection ของเรา -- ต้อง implement IntoResponse เพื่อบอกว่าถ้า extract ไม่ผ่าน จะตอบกลับยังไง
enum ApiKeyRejection {
    Missing,
    Invalid,
}

impl IntoResponse for ApiKeyRejection {
    fn into_response(self) -> Response {
        let (status, message) = match self {
            ApiKeyRejection::Missing => (StatusCode::UNAUTHORIZED, "ไม่พบ header X-Api-Key"),
            ApiKeyRejection::Invalid => (StatusCode::FORBIDDEN, "X-Api-Key ไม่ถูกต้อง"),
        };
        (status, Json(json!({ "error": message }))).into_response()
    }
}
```

สังเกตว่า `Rejection` ไม่จำเป็นต้องเป็น type เดียวตายตัว (เช่น `String` หรือ `StatusCode`) — มันเป็น
**enum ของเราเอง** ที่แยกแยะสถานการณ์ที่ล้มเหลวได้หลายแบบ (ไม่มี header เลย vs มี header แต่ค่าผิด) แล้วให้
แต่ละแบบตอบกลับด้วย status code ต่างกัน (`401 Unauthorized` กับ `403 Forbidden`) — นี่คือรูปแบบเดียวกับ
custom error type ที่ Part 30 สอนไว้ (`enum` ที่ implement `Display`/`Error`) แค่คราวนี้ target คือ
`IntoResponse` ของ Axum แทน `std::error::Error`

#### ขั้นที่ 2: implement `FromRequestParts`

```rust
# use axum::{extract::FromRequestParts, http::request::Parts};
# struct ApiKey(String);
# enum ApiKeyRejection { Missing, Invalid }
// FromRequestParts เพราะ ApiKey ต้องการแค่ header (parts) ไม่ต้องแตะ body เลย
// สังเกตว่าไม่มี #[async_trait] เพราะ axum 0.8 (และ 0.7 ขึ้นไป) ใช้ native async fn
// in trait (RPITIT: Return-Position impl Trait In Trait) ที่ Rust รองรับมาตั้งแต่ 1.75
impl<S> FromRequestParts<S> for ApiKey
where
    S: Send + Sync,
{
    type Rejection = ApiKeyRejection;

    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        let header_value = parts
            .headers
            .get("X-Api-Key")
            .ok_or(ApiKeyRejection::Missing)?;

        let key_str = header_value.to_str().map_err(|_| ApiKeyRejection::Invalid)?;

        // ในระบบจริง จุดนี้จะไป query database/cache (foreshadow Part 70-71, 74-76)
        // ในตัวอย่างนี้ hardcode ค่าที่ถูกต้องไว้ก่อนเพื่อความง่าย
        if key_str == "secret-123" {
            Ok(ApiKey(key_str.to_string()))
        } else {
            Err(ApiKeyRejection::Invalid)
        }
    }
}
```

ข้อสังเกตสำคัญเรื่อง syntax ที่ต่างจากที่หลายคนคุ้นเคยจาก tutorial เก่า: **ไม่ต้องมี `#[async_trait]` ครอบ
`impl` block นี้เลย** เขียน `async fn from_request_parts(...)` ตรง ๆ ได้ ทำงานถูกต้อง — สาเหตุคือ trait
`FromRequestParts` (ตั้งแต่ Axum 0.7 ขึ้นไป) นิยาม method ด้วย **RPITIT** (`fn from_request_parts(...) ->
impl Future<Output = ...> + Send`) ซึ่งเป็นฟีเจอร์ของภาษา Rust เองที่ stable มาตั้งแต่เวอร์ชัน 1.75 — ทำให้
`impl` ของ trait นี้เขียนเป็น `async fn` ปกติได้เลยโดยไม่ต้องพึ่ง macro จาก crate `async-trait` ที่จำเป็นใน
Axum เวอร์ชันเก่า ๆ (0.6 และก่อนหน้า) — ถ้าเห็นตัวอย่างเก่าที่มี `#[async_trait]` ครอบ `impl FromRequest`
แสดงว่ากำลังดู tutorial ที่เขียนไว้สำหรับ Axum เวอร์ชันเก่ากว่านี้มาก

#### ขั้นที่ 3: ใช้งานใน handler — ไม่มีอะไรพิเศษเลย

```rust
# use axum::{routing::get, Router};
# struct ApiKey(String);
async fn protected_handler(ApiKey(key): ApiKey) -> String {
    format!("เข้าถึงสำเร็จด้วย API key: {key}")
}

# fn wire_up() -> Router {
Router::new().route("/protected", get(protected_handler))
# }
```

handler ไม่รู้เรื่อง `IntoResponse` ของ rejection เลย ไม่รู้ว่า header ชื่ออะไร ไม่รู้เรื่อง `400`/`401`/`403`
อะไรทั้งนั้น — ทุกอย่างถูกจัดการให้เสร็จสรรพก่อนที่โค้ดใน `protected_handler` จะได้รันด้วยซ้ำ นี่คือพลังของ
extractor pattern: **ย้าย cross-cutting concern (การตรวจสอบสิทธิ์) ออกจาก business logic ไปไว้ที่ type
system** — ยิ่งมี handler ที่ต้องการการตรวจสอบแบบเดียวกันมากเท่าไหร่ ยิ่งประหยัดโค้ดซ้ำมากเท่านั้น (เขียน
`impl FromRequestParts for ApiKey` ครั้งเดียว ใช้ได้กับทุก handler ที่ใส่ `ApiKey` เป็น parameter)

#### ทดสอบด้วย `curl` จริง: ทั้งเส้นทางสำเร็จและถูกปฏิเสธ

```bash
# ไม่ส่ง header เลย -> 401
$ curl -s -i http://127.0.0.1:3070/protected
HTTP/1.1 401 Unauthorized
content-type: application/json
content-length: 44

{"error":"ไม่พบ header X-Api-Key"}

# ส่ง header แต่ค่าผิด -> 403
$ curl -s -i http://127.0.0.1:3070/protected -H 'X-Api-Key: wrong'
HTTP/1.1 403 Forbidden
content-type: application/json
content-length: 52

{"error":"X-Api-Key ไม่ถูกต้อง"}

# ส่ง header ถูก -> 200 พร้อม handler รันจริง
$ curl -s -i http://127.0.0.1:3070/protected -H 'X-Api-Key: secret-123'
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 71

เข้าถึงสำเร็จด้วย API key: secret-123
```

ครบทั้งสามเส้นทาง (ไม่มี header, header ผิด, header ถูก) ตรงตาม logic ที่เขียนไว้ทุกจุด — และเพราะ
`ApiKeyRejection` implement `IntoResponse` เอง เราควบคุมรูปแบบ error response (JSON, status code) ได้เต็มที่
โดยไม่ต้องพึ่ง default rejection ของ Axum เลย

### 64.7 `OriginalUri`, `Extension<T>` เทียบกับ `State<T>`

#### `OriginalUri`: URI ดั้งเดิมก่อนถูก route matching ตัด path ออก

extractor เล็ก ๆ อีกตัวที่มีประโยชน์คือ `OriginalUri` — คืน URI เต็มของ request ตามที่ client ส่งมาจริง ๆ
ต่างจาก `Uri` เปล่า ๆ ที่ (ในบางสถานการณ์ เช่นตอนอยู่ใต้ `.nest()`) อาจถูก Axum ปรับ path ให้สั้นลงตาม
prefix ที่ match ไปแล้ว `OriginalUri` มีประโยชน์มากตอนต้องการ log หรือสร้าง URL อ้างอิงกลับไปยัง endpoint
เดิมที่ client เรียกมาแบบไม่บิดเบือน:

```rust
use axum::extract::OriginalUri;

async fn handler_with_uri(OriginalUri(uri): OriginalUri) -> String {
    format!("คุณเรียก URI: {uri}")
}
```

#### `Extension<T>`: การแชร์ค่าแบบ runtime-only เทียบกับ `State<T>` แบบ compile-time-checked

ก่อน Axum จะเสนอ `State<T>` แบบที่เห็นทั้งบทนี้ (และในหลาย framework/library อื่นในโลก Rust ที่มาก่อน)
วิธีมาตรฐานในการแชร์ค่าคือ `Extension<T>` — ทำงานคล้ายกันตรงที่ extract ค่าที่ "แชร์" ออกมาให้ handler แต่
กลไกภายในต่างกันโดยพื้นฐาน:

- **`State<T>`**: ค่าที่ให้ผ่าน `.with_state()` ถูกผูกเข้ากับ **type parameter ของ `Router`** ตรง ๆ — ถ้า
  handler ขอ `State<AppState>` แต่ไม่มีการ `.with_state(state_ที่เป็น_AppState)` ที่ไหนเลย โปรแกรม**จะไม่
  compile** (ตามที่พิสูจน์ไว้แล้วในหัวข้อ 64.3) — เป็น**compile-time guarantee**
- **`Extension<T>`**: ค่าถูกเก็บไว้ใน `http::Extensions` ของ request/response (เป็น type-erased map ภายใน
  ที่เก็บค่าได้หลายชนิดพร้อมกัน โดย key คือ `TypeId`) — อะไรก็ตามที่มีสิทธิ์แก้ไข request (ปกติคือ
  **middleware**, สิ่งที่ Part 65 จะสอนเต็มรูปแบบ) สามารถ `.insert()` ค่าเข้าไปได้ตอนไหนก็ได้ระหว่างทาง และ
  handler ที่ขอ `Extension<T>` (สำหรับ `T` ชนิดที่ตรงกัน) จะดึงมันออกมา — **compiler ไม่รู้เลยว่ามี
  middleware ตัวไหน insert ค่าชนิดนี้ไว้จริงหรือไม่** ผลคือถ้าไม่มีใคร insert ไว้จริง ๆ error จะเกิด
  **ตอน runtime** (request จริงมาถึง แล้ว extract ล้มเหลว) ไม่ใช่ตอน compile

มาดูตัวอย่างจริงที่แสดงทั้งสองด้าน — middleware แทรก `RequestId` ผ่าน `Extension`, ควบคู่กับ `State<AppState>`
ปกติ:

```rust
use axum::{
    extract::{Extension, Request, State},
    middleware::{self, Next},
    response::Response,
    routing::get,
    Router,
};
use std::sync::{Arc, Mutex};

#[derive(Clone)]
struct AppState {
    hit_count: Arc<Mutex<u32>>,
}

// ค่าที่ middleware แทรกเข้ามาระหว่างทาง โดยที่ handler ไม่รู้ตอน compile ว่ามันจะถูกแทรกมาจริงหรือไม่
#[derive(Clone)]
struct RequestId(String);

async fn inject_request_id(mut req: Request, next: Next) -> Response {
    let id = RequestId("req-abc-123".to_string());
    req.extensions_mut().insert(id);
    next.run(req).await
}

async fn handler_with_extension(Extension(request_id): Extension<RequestId>) -> String {
    format!("Request ID จาก middleware ผ่าน Extension: {}", request_id.0)
}

async fn handler_with_state(State(state): State<AppState>) -> String {
    let mut count = state.hit_count.lock().unwrap();
    *count += 1;
    format!("เรียกมาแล้ว {count} ครั้ง (ผ่าน State ที่ผูกกับ Router ตอน compile)")
}

#[tokio::main]
async fn main() {
    let state = AppState { hit_count: Arc::new(Mutex::new(0)) };

    let app = Router::new()
        .route("/ext", get(handler_with_extension))
        .layer(middleware::from_fn(inject_request_id))
        .route("/state", get(handler_with_state))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3072").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

รันจริง:

```bash
$ curl -s http://127.0.0.1:3072/ext
Request ID จาก middleware ผ่าน Extension: req-abc-123

$ curl -s http://127.0.0.1:3072/state
เรียกมาแล้ว 1 ครั้ง (ผ่าน State ที่ผูกกับ Router ตอน compile)
```

ทำงานถูกต้องทั้งคู่ — เพราะ `.layer(middleware::from_fn(inject_request_id))` ถูกวางไว้**ก่อน**
`.route("/state", ...)` (เรื่องลำดับการวาง `.layer()` เทียบกับ `.route()` และผลที่ตามมา Part 65 จะอธิบาย
ลึกกว่านี้อีกมาก) มันจึงมีผลกับ route `/ext` เท่านั้นในตัวอย่างนี้ — ทีนี้มาดูว่าเกิดอะไรขึ้นถ้า handler ขอ
`Extension<T>` โดยที่**ไม่มี middleware ตัวไหน insert ค่าชนิดนั้นไว้เลย**:

```rust
use axum::{extract::Extension, routing::get, Router};

#[derive(Clone)]
struct RequestId(String);

// handler นี้ขอ Extension<RequestId> แต่ไม่มี middleware ไหนแทรกมันเข้ามาเลย
// -- compiler ปล่อยผ่านสนิท เพราะ Extension<T> ไม่ผูกกับ Router ตอน compile เหมือน State<T>
async fn handler(Extension(request_id): Extension<RequestId>) -> String {
    format!("request id: {}", request_id.0)
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/no-ext", get(handler));
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3073").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

โปรแกรมนี้ **compile ผ่านโดยไม่มี error หรือ warning ใด ๆ เลย** — สมมติฐานที่ผิด (ไม่มี middleware ไหน
insert `RequestId` ไว้จริง) ไม่ถูกจับได้เลยตอน compile time พอรันจริงแล้วยิง request:

```bash
$ curl -s -i http://127.0.0.1:3073/no-ext
HTTP/1.1 500 Internal Server Error
content-type: text/plain; charset=utf-8
content-length: 143

Missing request extension: Extension of type `extension_missing::RequestId` was not found. Perhaps you forgot to add it? See `axum::Extension`.
```

**500 Internal Server Error** — และปัญหานี้จะไม่มีทางถูกจับได้จนกว่าจะมี request จริงมาถึง route นี้ (ซึ่ง
อาจนานถึงตอน deploy ไป production แล้วก็ได้ ถ้า test ไม่ครอบคลุมทุก route) นี่คือความต่างเชิงคุณภาพที่สำคัญ
ที่สุดระหว่าง `State<T>` กับ `Extension<T>`:

| | `State<T>` | `Extension<T>` |
|---|---|---|
| ผูกกับ state ตอนไหน | Compile time (type parameter ของ `Router`) | Runtime (type-erased map ต่อ request) |
| ถ้าค่าที่ต้องการไม่มีจริง | **Compile error** ทันที | **500 Internal Server Error** ตอนมี request จริง |
| ใครเป็นคน "ใส่" ค่า | `.with_state()` ที่จุดสร้าง `Router` เท่านั้น | Middleware ใดก็ได้ ที่จุดไหนก็ได้ระหว่างทาง |
| เหมาะกับ | Application state หลักที่รู้ล่วงหน้าตอนเขียนโค้ด | ค่าที่ middleware สร้าง/คำนวณขึ้นมาระหว่างทาง (request id, authenticated user หลัง auth middleware, ฯลฯ) |

**ข้อสรุปเชิงปฏิบัติที่ใช้ได้จริง**: สำหรับ **application state ของคุณเอง** (database pool, config,
in-memory store) ที่คุณรู้ตอนเขียนโค้ดอยู่แล้วว่าต้องมีแน่ ๆ — **ใช้ `State<T>` เสมอ** เพื่อได้
compile-time guarantee ส่วน `Extension<T>` ยังมีที่ใช้จริงอยู่ในสถานการณ์เดียวคือ**ค่าที่ middleware สร้าง
ขึ้นมาระหว่างทาง** ที่ไม่มีทางรู้ตอน compile ว่ามันมีอยู่แน่นอน (เช่น "authenticated user" ที่ auth
middleware แทรกเข้ามาหลังตรวจสอบ token สำเร็จ — ถ้า route ไหนลืมแปะ auth middleware ไว้ ก็สมควรได้ 500 เป็น
สัญญาณเตือนว่า setup ผิด) — โค้ด Axum รุ่นเก่า (ก่อนที่ `State<T>` เต็มรูปแบบจะแพร่หลาย) มักใช้
`Extension<T>` สำหรับทุกอย่างรวมถึง database pool ด้วย ถ้าเห็นโค้ดแบบนั้นในโปรเจกต์เก่าหรือ tutorial เก่า
ให้เข้าใจว่าเป็น pattern ที่ล้าสมัยไปแล้วสำหรับ use case นั้น

### 64.8 Optional Extractors: `Option<T>` และ `Result<T, T::Rejection>`

พฤติกรรมปกติของ Axum คือถ้า extractor ตัวไหน extract ไม่สำเร็จ (rejection เกิดขึ้น) **request จะถูกปฏิเสธ
ทันที** — handler ตัวจริงจะไม่ได้รันเลยแม้แต่บรรทัดเดียว response ที่ client ได้รับคือ response ของ
`Rejection` (ที่เราควบคุมได้เต็มที่ถ้าเป็น custom extractor แบบหัวข้อ 64.6) แต่บางสถานการณ์ ความล้มเหลวของ
การ extract ไม่ควรทำให้ทั้ง request ถูกปฏิเสธ — เช่น endpoint สาธารณะที่ "ถ้ามี API key ที่ถูกต้องก็ให้เห็น
ข้อมูลพิเศษเพิ่ม แต่ถ้าไม่มีก็ยังใช้งานได้ปกติแบบผู้เยี่ยมชมทั่วไป"

Axum รองรับสถานการณ์นี้โดยให้ห่อ extractor ด้วย **`Option<T>`** หรือ **`Result<T, T::Rejection>`** แทนที่จะ
ใช้ `T` ตรง ๆ

#### `Result<T, T::Rejection>`: ได้เหตุผลของความล้มเหลวมาจัดการเอง

ฝั่งนี้ทำงานตรงไปตรงมา — Axum มี blanket implementation ที่ทำให้ `Result<T, T::Rejection>` เป็น extractor
ได้เสมอ (ไม่ต้องเขียนอะไรเพิ่มเลย) ทุกครั้งที่ `T::from_request_parts`/`from_request` คืน `Err(rejection)`
ค่านั้นจะถูกส่งเข้า handler เป็น `Err(rejection)` แทนที่จะ auto-reject ทันที:

```rust
# use axum::{http::StatusCode, response::{IntoResponse, Response}};
# struct ApiKey(String);
# enum ApiKeyRejection { Missing, Invalid }
async fn diagnostic_handler(api_key: Result<ApiKey, ApiKeyRejection>) -> Response {
    match api_key {
        Ok(ApiKey(key)) => format!("ok key={key}").into_response(),
        Err(_rejection) => {
            // เลือกจัดการเองแทนที่จะให้ rejection ตอบกลับตรง ๆ เช่น log แล้วค่อยตอบ 200 เสมอ
            println!("diagnostic_handler: extraction ล้มเหลว (จัดการเองไม่ auto-reject)");
            (StatusCode::OK, "รับทราบคำขอแล้ว แม้ API key จะมีปัญหา").into_response()
        }
    }
}
```

ทดสอบจริง (ไม่ส่ง header เลย):

```bash
$ curl -s -i http://127.0.0.1:3071/diagnostic
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 91

รับทราบคำขอแล้ว แม้ API key จะมีปัญหา
```

สังเกตว่าปกติ (ไม่ห่อ `Result`) การไม่ส่ง header จะได้ `401 Unauthorized` (ตามหัวข้อ 64.6) แต่พอห่อด้วย
`Result<ApiKey, ApiKeyRejection>` handler เลือกที่จะตอบ `200 OK` เองแทนได้ — เห็น log บรรทัด
`diagnostic_handler: extraction ล้มเหลว (จัดการเองไม่ auto-reject)` ที่ฝั่ง server ยืนยันว่าโค้ดใน branch
`Err` ถูกรันจริง

#### `Option<T>`: ต้อง opt-in ผ่าน `OptionalFromRequestParts` (รายละเอียดที่เปลี่ยนไปใน Axum 0.8)

ฝั่งนี้มีรายละเอียดที่เปลี่ยนไประหว่าง Axum เวอร์ชันต่าง ๆ ที่สำคัญมากพอจะพูดถึงตรง ๆ: **ใน Axum 0.7 และ
ก่อนหน้า** มี blanket implementation ที่ทำให้ `Option<T>` เป็น extractor ได้เสมอสำหรับทุก `T: FromRequestParts`
โดยอัตโนมัติ (แปลง `Err(_)` เป็น `None` แบบตรงไปตรงมา) แต่ **Axum 0.8 (เวอร์ชันที่ใช้ตรวจสอบเนื้อหาทั้งบทนี้
คือ 0.8.9) เปลี่ยนพฤติกรรมนี้** — ไม่มี blanket impl ให้ `Option<T>` ตรง ๆ อีกต่อไป แต่ต้อง**เขียน opt-in
เอง**ผ่าน trait ใหม่ชื่อ **`OptionalFromRequestParts<S>`** (เหตุผลของทีม Axum คือให้ผู้เขียน extractor เป็น
คนตัดสินใจเองว่า "ความล้มเหลวแบบไหนควรกลายเป็น `None`" เพราะบางแบบ เช่น header มีอยู่แต่ format ผิด อาจ
ควรยังคงเป็น error จริง ๆ ไม่ใช่แค่ "ไม่มี" เฉย ๆ)

มาดูว่าเขียน opt-in นี้อย่างไร (ต่อจาก `ApiKey`/`ApiKeyRejection` ในหัวข้อ 64.6):

```rust
# use axum::{extract::{FromRequestParts, OptionalFromRequestParts}, http::request::Parts};
# struct ApiKey(String);
# enum ApiKeyRejection { Missing, Invalid }
# impl<S> FromRequestParts<S> for ApiKey where S: Send + Sync {
#     type Rejection = ApiKeyRejection;
#     async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
#         unimplemented!()
#     }
# }
// axum 0.8 เปลี่ยนพฤติกรรม Option<T>: ไม่ blanket-impl FromRequestParts ให้ Option<T> ทุกตัวแบบ 0.7 อีกต่อไป
// ต้อง opt-in เองผ่าน OptionalFromRequestParts<S> จึงจะใช้ `Option<ApiKey>` เป็น extractor ได้
impl<S> OptionalFromRequestParts<S> for ApiKey
where
    S: Send + Sync,
{
    type Rejection = ApiKeyRejection;

    async fn from_request_parts(
        parts: &mut Parts,
        state: &S,
    ) -> Result<Option<Self>, Self::Rejection> {
        // ไม่มี header เลย -> ถือว่าเป็น "ไม่มี" (None) ไม่ใช่ error
        if !parts.headers.contains_key("X-Api-Key") {
            return Ok(None);
        }
        // มี header แต่ค่าอาจผิด -> ยังใช้ logic เดิมของ FromRequestParts แล้วห่อเป็น Some
        <ApiKey as FromRequestParts<S>>::from_request_parts(parts, state)
            .await
            .map(Some)
    }
}
```

จุดที่ควรสังเกตให้ดี: ตัวเลือกออกแบบข้างบนคือ **"ไม่มี header เลย" → `None`** แต่ **"มี header แต่ค่าผิด" →
ยังคง `Err(ApiKeyRejection::Invalid)` เหมือนเดิม (ไม่ใช่ `None`)** — นี่คือการตัดสินใจที่สมเหตุสมผลในทาง
ปฏิบัติ: ถ้าผู้เรียกไม่ได้ตั้งใจส่ง API key มาเลย แสดงว่าเขาต้องการเข้าถึงแบบผู้เยี่ยมชมทั่วไปจริง ๆ (สมควร
เป็น `None`) แต่ถ้าเขา**ตั้งใจ**ส่ง key มาแล้วมันผิด (พิมพ์ผิด, key หมดอายุ) นั่นคือสถานการณ์ที่ควรแจ้ง error
กลับไปตรง ๆ ไม่ควรเงียบแล้วปฏิบัติเหมือนไม่มี key มา (ผู้ใช้จะงงว่าทำไม "ข้อมูลพิเศษ" ที่ควรเห็นไม่โผล่ ทั้งที่
คิดว่าส่ง key ถูกต้องไปแล้ว) — เขียน handler ที่ใช้งาน:

```rust
# struct ApiKey(String);
async fn public_handler(api_key: Option<ApiKey>) -> String {
    match api_key {
        Some(ApiKey(key)) => format!("สวัสดีสมาชิก (key={key}) -- คุณได้เห็นข้อมูลพิเศษเพิ่ม"),
        None => "สวัสดีผู้เยี่ยมชมทั่วไป -- ไม่มี/ไม่ถูกต้อง API key ก็ยังใช้งานได้ปกติ".to_string(),
    }
}
```

ทดสอบจริงครบทั้งสามเส้นทาง:

```bash
# ไม่ส่ง header -> None -> ผู้เยี่ยมชมทั่วไป (200)
$ curl -s -i http://127.0.0.1:3071/public
HTTP/1.1 200 OK
content-length: 182

สวัสดีผู้เยี่ยมชมทั่วไป -- ไม่มี/ไม่ถูกต้อง API key ก็ยังใช้งานได้ปกติ

# ส่ง header แต่ค่าผิด -> ยังเป็น Err จริง ๆ (403) ไม่ใช่ None
$ curl -s -i http://127.0.0.1:3071/public -H 'X-Api-Key: wrong'
HTTP/1.1 403 Forbidden
content-length: 52

{"error":"X-Api-Key ไม่ถูกต้อง"}

# ส่ง header ถูก -> Some -> สมาชิก (200)
$ curl -s -i http://127.0.0.1:3071/public -H 'X-Api-Key: secret-123'
HTTP/1.1 200 OK
content-length: 135

สวัสดีสมาชิก (key=secret-123) -- คุณได้เห็นข้อมูลพิเศษเพิ่ม
```

ผลลัพธ์ตรงตามที่ตั้งใจออกแบบไว้ทุกจุด — บทเรียนสำคัญจากหัวข้อนี้: **`Option<T>` ไม่ได้แปลว่า "กลืน error
ทุกแบบให้เงียบ"** มันเป็นแค่กลไกที่**คุณควบคุมได้เอง**ว่าความล้มเหลวแบบไหนสมควรเป็น "ไม่มีค่า" (ปกติ ไม่ใช่
ปัญหา) กับแบบไหนยังสมควรเป็น error จริง ๆ ที่ควรถูกรายงานออกไป — และถ้าใช้ crate Axum เวอร์ชันเก่ากว่า 0.8
(ที่มี blanket impl ให้ `Option<T>` อัตโนมัติ) การเลือกนี้จะไม่มี ทุก `Err` จะกลายเป็น `None` เสมอไม่มี
ข้อยกเว้น — ต่างจาก 0.8 ที่ให้ความคุมได้ละเอียดขึ้นแต่ต้องเขียน `OptionalFromRequestParts` เพิ่มเอง

## กับดักที่พบบ่อย (Common Pitfalls)

**1. ลืม `#[derive(Clone)]` บน `AppState`** — error ที่ได้คือ `the trait bound `AppState: Clone` is not
satisfied` ผสมกับ `the trait `Handler<_, _>` is not implemented` เกิดขึ้นพร้อมกันหลายจุด (ที่
`Router::new()`, `.route()`, `.with_state()`) เพราะทุกเมธอดของ `Router<S>` ต้องการ `S: Clone + Send +
Sync + 'static` — วิธีแก้ตรงไปตรงมา: เติม `#[derive(Clone)]` ไว้เหนือ `AppState` เสมอ (compiler มักแนะนำ
วิธีแก้นี้ให้ตรง ๆ ผ่าน `help:` อยู่แล้ว)

**2. เก็บ `Mutex<T>`/`RwLock<T>` ตรง ๆ ใน `AppState` โดยไม่ห่อ `Arc`** — ถึงจะ `#[derive(Clone)]` แล้วก็ตาม
error ที่ได้คือ `the trait bound `std::sync::Mutex<...>: Clone` is not satisfied` เพราะ `Mutex`/`RwLock`
ไม่ implement `Clone` เลยไม่ว่ากรณีใด (การ clone จะทำให้ mutual exclusion ที่ตั้งใจไว้พังทันที) — วิธีแก้:
ห่อด้วย `Arc<Mutex<T>>` เสมอเมื่อ field นั้นต้อง mutate ได้ (`Arc::clone()` แค่เพิ่ม reference count ทำให้
ทุก clone ของ `AppState` ยังชี้ไปที่ `Mutex` ตัวเดียวกัน)

**3. ลืมเรียก `.with_state(state)`** — โปรแกรม**อาจ compile ผ่านได้ในบางจุด** (เช่น `Router::new()`,
`.route()`) เพราะ Axum ยังไม่รู้ชนิดจริงของ state ตอนนั้น แต่จะพังตอนพยายามส่ง `Router<AppState>` ที่ยังไม่
ถูก "ปิด" (ยังไม่ใช่ `Router<()>`) เข้า `axum::serve()` — error ที่ได้คือ `the trait
`tower_service::Service<...>` is not implemented for `Router<AppState>`` พร้อม help ที่บอกว่า
`Router<()>` เท่านั้นที่ implement `Service` — วิธีแก้: ตรวจสอบว่าทุก `Router` ที่ส่งเข้า `axum::serve()`
ผ่าน `.with_state(...)` มาแล้วครบทุกจุดที่มี handler ขอ `State<T>`

**4. เขียน body-consuming extractor (`Json<T>`, `Bytes`, `Form<T>`) ไม่ใช่ตัวสุดท้ายใน parameter list** —
error คือ `the trait `Handler<_, _>` is not implemented` ที่อ่านไม่รู้เรื่องว่าปัญหาคืออะไร — แก้ด้วยการ
แปะ `#[axum::debug_handler]` (ต้องเปิด feature `macros` ของ `axum`) เหนือ handler ตัวนั้นแล้ว compile ใหม่
จะได้ error ที่ชัดเจนตรงจุด: `` `Json<_>` consumes the request body and thus must be the last argument
to the handler function `` — สาเหตุคือ `Json<T>` implement `FromRequest` (กิน body ทั้ง `Request`) ในขณะที่
`Path`/`Query`/`State`/header extractor implement `FromRequestParts` (อ่านแค่ `&mut Parts` ไม่แตะ body) —
extractor ที่กิน body ได้แค่ตัวเดียวต่อ handler และต้องมาตัวสุดท้ายเสมอ

**5. สับสนระหว่าง `Option<T>` กับ auto-reject ปกติ (โดยเฉพาะข้ามเวอร์ชัน Axum)** — ถ้าเห็น tutorial เก่าที่
บอกว่า `Option<T>` "ใช้ได้กับทุก extractor อัตโนมัติ" นั่นคือพฤติกรรมของ Axum 0.7 และก่อนหน้า — ใน Axum 0.8
(เวอร์ชันปัจจุบันที่ใช้ตรวจสอบบทนี้) custom extractor ของคุณต้อง implement `OptionalFromRequestParts<S>`
(หรือ `OptionalFromRequest<S>` สำหรับ extractor ที่กิน body) เองก่อน จึงจะใช้ `Option<YourType>` เป็น
handler parameter ได้ — ถ้าลืม จะได้ error แบบเดียวกับข้อ 4 (`Handler<_, _> is not implemented`) ที่ชี้ไปว่า
extractor นี้ยังไม่มี implementation ที่ Axum ต้องการ

**6. คิดว่า `Extension<T>` ปลอดภัยเท่า `State<T>`** — โค้ดที่ handler ขอ `Extension<T>` โดยไม่มี middleware
ตัวไหน insert ค่าชนิดนั้นไว้จริง จะ **compile ผ่านสนิทไม่มี warning เลย** แต่พังตอน runtime ด้วย `500
Internal Server Error` พร้อมข้อความ `Missing request extension: Extension of type `...` was not found.
Perhaps you forgot to add it? See `axum::Extension`.` — ต่างจาก `State<T>` ที่ปัญหาแบบเดียวกัน (ไม่มี state
ที่ต้องการจริง) จะถูกจับได้ตั้งแต่ compile time เสมอ — บทเรียน: ใช้ `State<T>` สำหรับ application state
หลักที่รู้แน่ชัดตอนเขียนโค้ด สงวน `Extension<T>` ไว้เฉพาะค่าที่ middleware สร้างขึ้นระหว่างทางเท่านั้น

**7. ใส่ `FromRef` ไม่ครบทุก substate ที่ handler ต้องการ** — ถ้า handler ขอ `State<SubState>` แต่ไม่มี
`impl FromRef<AppState> for SubState` เลย จะได้ compile error ประเภทเดียวกับข้อ 3/4 (`Handler<_, _> is not
implemented`) เพราะ Axum หาทางแปลง `AppState` เป็น `SubState` ไม่ได้ — แก้ด้วยการเพิ่ม `impl
FromRef<AppState> for SubState { ... }` ให้ครบทุก substate ที่มี handler ไหนขอใช้งานจริง

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เพิ่ม endpoint `DELETE /tickets/{id}` เข้าไปในตัวอย่าง `AppState`/`TicketStore` ของหัวข้อ
   64.2 — ให้คืน `204 No Content` ถ้าลบสำเร็จ และ `404 Not Found` ถ้าไม่มี id นั้นอยู่จริง ทดสอบด้วย
   `curl -X DELETE` ทั้งสองเส้นทาง (มีอยู่จริง/ไม่มีอยู่จริง)
   *Hint*: ใช้ `HashMap::remove()` ที่คืน `Option<Ticket>` — แปลง `Some`/`None` เป็น status code ที่
   ต้องการด้วย `match` หรือ `.map_or()`

2. **(กลาง)** ขยายตัวอย่าง `FromRef` ในหัวข้อ 64.4 ให้มี substate ที่สามชนิด — `RequestLog` (เก็บ
   `Arc<Mutex<Vec<String>>>` ของ log message) แล้วเขียน handler ใหม่ `GET /logs` ที่ขอ `State<RequestLog>`
   ตรง ๆ คืนรายการ log ทั้งหมดเป็น JSON array และแก้ `site_info`/`booking_count` ให้ log ทุกครั้งที่ถูกเรียก
   เข้า `RequestLog` ด้วย (handler เหล่านี้ต้องขอทั้ง substate เดิมของมัน + `State<RequestLog>` พร้อมกัน)
   *Hint*: `impl FromRef<AppState> for RequestLog` เหมือนที่ทำกับ `AppConfig`/`BookingCounter` — handler
   หนึ่งตัวขอ `State<T>` ได้มากกว่าหนึ่งครั้งถ้าเป็นคนละ substate กัน (`State(config): State<AppConfig>,
   State(log): State<RequestLog>` เป็น parameter สองตัวแยกกันได้ปกติ)

3. **(ยาก)** เขียน custom extractor ชื่อ `AdminUser` ที่ทำงานสองชั้น: อ่าน header `X-Api-Key` แบบเดียวกับ
   หัวข้อ 64.6 ก่อน แล้วถ้าค่าตรงกับ key พิเศษ (`"admin-secret-999"`) ค่อยสำเร็จเป็น `AdminUser` ถ้าเป็น key
   อื่นที่ถูกต้องแบบ user ทั่วไป (`"secret-123"`) ให้ reject ด้วย `403 Forbidden` (บอกว่า "ต้องเป็น admin
   เท่านั้น") ถ้าไม่มี/ผิดเลย ให้ reject แบบเดิม (`401`) — ใช้ extractor นี้ป้องกัน endpoint ใหม่ `DELETE
   /tickets/{id}` ที่เขียนไว้ในข้อ 1 ทดสอบด้วย `curl` ครบทั้งสามเส้นทาง (ไม่มี key / key ธรรมดา / admin key)
   *Hint*: เขียน enum `AdminRejection` แยก 3 สถานการณ์ให้ `IntoResponse` คนละแบบ ข้างใน
   `from_request_parts` ทำ logic ตรวจ header เหมือนเดิมก่อน แล้วค่อยเช็คเพิ่มว่า key ที่ผ่านมาคือ admin key
   หรือเปล่า

4. **(ยาก/ประยุกต์)** รวมทุกอย่างในบทนี้เข้าด้วยกันเป็นระบบเดียว: ต่อยอด capstone ระบบห้องสมุด (หัวข้อ
   ถัดไปในบทนี้) ให้มี `AppState` ที่แยกเป็น substate สองส่วนผ่าน `FromRef` — `LibraryStore` (หนังสือ) กับ
   `AccessLog` (บันทึกว่า client id ไหนเรียก endpoint อะไรบ้าง เก็บเป็น `Arc<Mutex<Vec<String>>>`) แก้ทุก
   handler ที่มีอยู่ให้บันทึก log ลง `AccessLog` ทุกครั้งที่ถูกเรียก (ไม่ใช่แค่ `println!` เหมือนในตัวอย่าง
   เดิม) แล้วเพิ่ม endpoint ใหม่ `GET /admin/logs` ที่ต้องผ่าน `AdminUser` extractor จากข้อ 3 ก่อนถึงจะเห็น
   log ทั้งหมดได้ ทดสอบทั้งระบบด้วย `curl` transcript แบบเดียวกับหัวข้อ 64.10 ให้ครบทุกเส้นทาง (สำเร็จ,
   ปฏิเสธเพราะไม่ใช่ admin, ปฏิเสธเพราะไม่มี key เลย)
   *Hint*: นี่คือการรวม `FromRef` (ข้อ 2) + custom extractor สองชั้น (ข้อ 3) + CRUD เดิม (ข้อ 1) เข้าด้วยกัน
   ทั้งหมด — เริ่มจากออกแบบ `AppState` ก่อนว่าจะมี field อะไรบ้าง แล้วค่อยเขียน `impl FromRef` ให้ครบทุก
   substate ก่อนแก้ handler

### 64.9 (ต่อ) รวมทุกอย่างเข้าด้วยกัน: Capstone ระบบห้องสมุด

มาปิดบทด้วยตัวอย่างที่รวมทุกแนวคิดของบทนี้เข้าด้วยกันเป็นระบบเดียวที่สมบูรณ์ — **ระบบห้องสมุด** ที่มี:

- `AppState` ที่ห่อ `Arc<LibraryStore>` (แบบหัวข้อ 64.2 — `Mutex` สองตัวรวมกันหลัง `Arc` เดียว)
- Custom extractor `ClientId` ที่ต้องมี header `X-Client-Id` เป็น non-empty string ในทุก request (แบบ
  หัวข้อ 64.6)
- ลำดับ parameter ที่ถูกต้องตามกฎ `FromRequestParts` ก่อน `FromRequest` เสมอ (หัวข้อ 64.5)
- CRUD เต็มรูปแบบ: list, create, get by id, checkout (ยืมหนังสือ)

```rust
use axum::{
    extract::{FromRequestParts, Path, State},
    http::{request::Parts, StatusCode},
    response::{IntoResponse, Response},
    routing::{get, patch},
    Json, Router,
};
use serde::{Deserialize, Serialize};
use serde_json::json;
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

// ===================== โดเมนของแอป: ระบบห้องสมุด =====================

#[derive(Clone, Serialize, Deserialize)]
struct Book {
    id: u32,
    title: String,
    author: String,
    available: bool,
}

#[derive(Deserialize)]
struct NewBook {
    title: String,
    author: String,
}

// เก็บทุกอย่างที่ "แก้ไขได้" ไว้หลัง Mutex เดียว ต่างจากยุคก่อนที่แต่ละ handler
// อาจต้อง capture Arc<Mutex<...>> คนละตัวผ่าน closure -- ที่นี่มีจุดเดียวที่นิยาม state ทั้งหมด
struct LibraryStore {
    books: Mutex<HashMap<u32, Book>>,
    next_id: Mutex<u32>,
}

impl LibraryStore {
    fn new() -> Self {
        Self { books: Mutex::new(HashMap::new()), next_id: Mutex::new(1) }
    }
}

// AppState คือสิ่งเดียวที่ handler ทุกตัวขอผ่าน State<AppState> -- ห่อด้วย Arc
// เพราะ Router::with_state ต้องการ S: Clone และ clone Arc คือ clone แค่ pointer + เพิ่ม
// reference count (เทียบกับ Part 56 ที่วัดว่า Arc::clone ถูกกว่าการ deep-clone HashMap มาก)
#[derive(Clone)]
struct AppState {
    library: Arc<LibraryStore>,
}

// ===================== Custom Extractor: X-Client-Id =====================

/// Extractor ที่ต้องมี header `X-Client-Id` เป็น non-empty string
/// (ในระบบจริง Part 74-76 จะขยายให้เป็น JWT/RBAC เต็มรูปแบบ -- นี่คือ "ของเล่น" ก่อนถึงจุดนั้น)
struct ClientId(String);

enum ClientIdRejection {
    Missing,
    Empty,
}

impl IntoResponse for ClientIdRejection {
    fn into_response(self) -> Response {
        let message = match self {
            ClientIdRejection::Missing => "ต้องส่ง header X-Client-Id มาด้วยทุกคำขอ",
            ClientIdRejection::Empty => "X-Client-Id ต้องไม่เป็นค่าว่าง",
        };
        (StatusCode::BAD_REQUEST, Json(json!({ "error": message }))).into_response()
    }
}

impl<S> FromRequestParts<S> for ClientId
where
    S: Send + Sync,
{
    type Rejection = ClientIdRejection;

    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        let value = parts
            .headers
            .get("X-Client-Id")
            .ok_or(ClientIdRejection::Missing)?
            .to_str()
            .map_err(|_| ClientIdRejection::Missing)?;

        if value.trim().is_empty() {
            return Err(ClientIdRejection::Empty);
        }

        Ok(ClientId(value.to_string()))
    }
}

// ===================== Handlers =====================
// สังเกตลำดับ parameter ในทุก handler: State/Path/ClientId (FromRequestParts, ไม่แตะ body)
// มาก่อนเสมอ ส่วน Json<T> (FromRequest, แตะ body) ต้องมาเป็นตัวสุดท้ายเท่านั้น

async fn list_books(State(state): State<AppState>, ClientId(client): ClientId) -> Json<Vec<Book>> {
    println!("[access] client={client} เรียกดูรายการหนังสือทั้งหมด");
    let books = state.library.books.lock().unwrap();
    Json(books.values().cloned().collect())
}

async fn create_book(
    State(state): State<AppState>,
    ClientId(client): ClientId,
    Json(body): Json<NewBook>,
) -> Json<Book> {
    let mut next_id = state.library.next_id.lock().unwrap();
    let id = *next_id;
    *next_id += 1;

    let book = Book { id, title: body.title, author: body.author, available: true };
    state.library.books.lock().unwrap().insert(id, book.clone());
    println!("[access] client={client} เพิ่มหนังสือใหม่ id={id}");
    Json(book)
}

async fn get_book(
    State(state): State<AppState>,
    ClientId(client): ClientId,
    Path(id): Path<u32>,
) -> Result<Json<Book>, StatusCode> {
    println!("[access] client={client} ขอดูหนังสือ id={id}");
    state
        .library
        .books
        .lock()
        .unwrap()
        .get(&id)
        .cloned()
        .map(Json)
        .ok_or(StatusCode::NOT_FOUND)
}

async fn checkout_book(
    State(state): State<AppState>,
    ClientId(client): ClientId,
    Path(id): Path<u32>,
) -> Result<Json<Book>, (StatusCode, Json<serde_json::Value>)> {
    let mut books = state.library.books.lock().unwrap();
    match books.get_mut(&id) {
        None => Err((
            StatusCode::NOT_FOUND,
            Json(json!({ "error": format!("ไม่พบหนังสือ id={id}") })),
        )),
        Some(book) if !book.available => Err((
            StatusCode::CONFLICT,
            Json(json!({ "error": format!("หนังสือ '{}' ถูกยืมไปแล้ว", book.title) })),
        )),
        Some(book) => {
            book.available = false;
            println!("[access] client={client} ยืมหนังสือ id={id}");
            Ok(Json(book.clone()))
        }
    }
}

#[tokio::main]
async fn main() {
    let state = AppState { library: Arc::new(LibraryStore::new()) };

    let app = Router::new()
        .route("/books", get(list_books).post(create_book))
        .route("/books/{id}", get(get_book))
        .route("/books/{id}/checkout", patch(checkout_book))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3074").await.unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}
```

ทดสอบครบทุกเส้นทาง (สำเร็จและถูกปฏิเสธ) ด้วย `curl` จริง ตามลำดับ:

```bash
# 1. ไม่ส่ง X-Client-Id -> 400 (ClientIdRejection::Missing)
$ curl -s -i http://127.0.0.1:3074/books
HTTP/1.1 400 Bad Request
content-type: application/json
content-length: 92

{"error":"ต้องส่ง header X-Client-Id มาด้วยทุกคำขอ"}

# 2. ส่ง X-Client-Id เป็นค่าว่างจริง ๆ (curl ต้องใช้ syntax `;` เพื่อส่ง header ที่มีค่าว่าง
#    เพราะ `-H 'X-Client-Id: '` เฉย ๆ curl จะตัด header ทิ้งไปเลยไม่ส่งอะไรออกไป)
$ curl -s -i http://127.0.0.1:3074/books -H 'X-Client-Id;'
HTTP/1.1 400 Bad Request
content-type: application/json
content-length: 78

{"error":"X-Client-Id ต้องไม่เป็นค่าว่าง"}

# 3. ส่ง X-Client-Id ถูกต้อง -> 200 ได้ [] (ยังไม่มีหนังสือ)
$ curl -s http://127.0.0.1:3074/books -H 'X-Client-Id: lib-app-01'
[]

# 4. สร้างหนังสือเล่มที่ 1
$ curl -s -i -X POST http://127.0.0.1:3074/books \
    -H 'X-Client-Id: lib-app-01' -H 'Content-Type: application/json' \
    -d '{"title":"The Rust Programming Language","author":"Steve Klabnik"}'
HTTP/1.1 200 OK
content-type: application/json
content-length: 90

{"id":1,"title":"The Rust Programming Language","author":"Steve Klabnik","available":true}

# 5. สร้างหนังสือเล่มที่ 2
$ curl -s -i -X POST http://127.0.0.1:3074/books \
    -H 'X-Client-Id: lib-app-01' -H 'Content-Type: application/json' \
    -d '{"title":"Programming Rust","author":"Jim Blandy"}'
HTTP/1.1 200 OK
content-type: application/json
content-length: 74

{"id":2,"title":"Programming Rust","author":"Jim Blandy","available":true}

# 6. ดูรายการทั้งหมด -- เห็นทั้งสองเล่ม
$ curl -s http://127.0.0.1:3074/books -H 'X-Client-Id: lib-app-01'
[{"id":1,"title":"The Rust Programming Language","author":"Steve Klabnik","available":true},{"id":2,"title":"Programming Rust","author":"Jim Blandy","available":true}]

# 7. ดูหนังสือ id=1 เจาะจง
$ curl -s -i http://127.0.0.1:3074/books/1 -H 'X-Client-Id: lib-app-01'
HTTP/1.1 200 OK
content-type: application/json
content-length: 90

{"id":1,"title":"The Rust Programming Language","author":"Steve Klabnik","available":true}

# 8. ดูหนังสือ id=99 ที่ไม่มีอยู่จริง -> 404
$ curl -s -i http://127.0.0.1:3074/books/99 -H 'X-Client-Id: lib-app-01'
HTTP/1.1 404 Not Found
content-length: 0

# 9. ยืมหนังสือ id=1
$ curl -s -i -X PATCH http://127.0.0.1:3074/books/1/checkout -H 'X-Client-Id: lib-app-01'
HTTP/1.1 200 OK
content-type: application/json
content-length: 91

{"id":1,"title":"The Rust Programming Language","author":"Steve Klabnik","available":false}

# 10. ยืมซ้ำอีกครั้ง -- เล่มนี้ถูกยืมไปแล้ว -> 409 Conflict
$ curl -s -i -X PATCH http://127.0.0.1:3074/books/1/checkout -H 'X-Client-Id: lib-app-01'
HTTP/1.1 409 Conflict
content-type: application/json
content-length: 102

{"error":"หนังสือ 'The Rust Programming Language' ถูกยืมไปแล้ว"}
```

ทั้ง 10 เส้นทางทำงานตรงตามที่ออกแบบไว้ทุกจุด — และสังเกตว่าโค้ดทั้งหมดนี้ **ไม่มี `.clone()` ตัวแปรเข้า
closure สักครั้งเดียว** ไม่มี `move ||` handler ทุกตัวเป็น `async fn` ธรรมดาที่ประกาศ dependency ของตัวเอง
ผ่าน type ใน signature ตรง ๆ — ตรงข้ามกับปัญหาที่ตั้งไว้ในหัวข้อ 64.1 ทุกจุด: state เดียวชัดเจน (`AppState`),
ทุก handler ประกาศชัดว่าต้องการอะไร (`State<AppState>`, `ClientId`, `Path<u32>`, `Json<NewBook>`), และการ
เพิ่ม cross-cutting concern ใหม่ (การตรวจสอบ `X-Client-Id`) ทำครั้งเดียวที่ extractor แล้วนำไปใช้ซ้ำได้กับ
ทุก handler ที่ต้องการ

## สรุป

บทนี้ปิดช่องว่างสำคัญระหว่างการเขียนแอป Axum เล็ก ๆ (ที่ capture `Arc<Mutex<T>>` ตรง ๆ เข้า closure พอไหว)
กับการเขียนแอปจริงที่มีหลายก้อน state และหลาย handler ผ่านสองแนวคิดหลัก:

- **`AppState` + `State<T>` + `.with_state()`** คือแพทเทิร์นมาตรฐานสำหรับแชร์ state — รวมทุกอย่างไว้ใน
  struct เดียว ห่อด้วย `Arc` เสมอ (เพราะ `.clone()` เกิดขึ้นทุก request ที่ handler ต้องการ state และต้อง
  ถูกทั้งด้าน performance และด้าน correctness ของ shared mutable state) แล้วให้ handler แต่ละตัวประกาศ
  ชัดเจนใน signature ว่าต้องการอะไร — ลืม `#[derive(Clone)]`, ลืม `Arc`, หรือลืม `.with_state()` ทั้งสาม
  กรณีจับได้ตั้งแต่ compile time ด้วย error message ที่ (ส่วนใหญ่) ชี้ทางแก้ให้ตรง ๆ
- **`FromRef`** ให้แยก state ก้อนใหญ่เป็น substate ที่ handler แต่ละตัวขอเฉพาะส่วนที่ตัวเองต้องการได้ โดยไม่
  ต้องแยก `Router` หลายตัว
- **`FromRequestParts` vs `FromRequest`** คือรากฐานของระบบ extractor ทั้งหมด — แยกกันเพราะการอ่าน request
  body เป็นการ consume ทำได้ครั้งเดียว ส่วนการอ่าน headers/path/state ทำได้กี่ครั้งก็ได้ — และนี่คือเหตุผล
  ที่แท้จริงของกฎ "body extractor ต้องมาตัวสุดท้าย"
- **Custom extractor** (implement `FromRequestParts` เอง) คือเครื่องมือทรงพลังที่สุดของบทนี้ — ย้าย
  cross-cutting concern อย่างการตรวจสอบสิทธิ์ออกจาก business logic ไปไว้ที่ type system ทั้งหมด พร้อม
  ต่อยอดตรงไปยัง JWT authentication (Part 74) และ RBAC (Part 76)
- **`Extension<T>`** ยังมีที่ใช้จริงสำหรับค่าที่ middleware สร้างขึ้นระหว่างทาง แต่ต่างจาก `State<T>` ตรงที่
  ไม่มี compile-time guarantee เลย — ถ้าค่าที่ต้องการไม่มีจริง จะได้ `500 Internal Server Error` ตอน
  runtime เท่านั้น
- **`Option<T>`/`Result<T, T::Rejection>`** ให้จัดการความล้มเหลวของการ extract เองแทนปล่อยให้ auto-reject
  — และใน Axum 0.8 การใช้ `Option<T>` กับ custom extractor ต้อง opt-in ผ่าน `OptionalFromRequestParts`
  เอง (เปลี่ยนจากพฤติกรรม blanket-impl อัตโนมัติของ 0.7)

ตอนนี้คุณมีเครื่องมือครบสำหรับสร้างแอป Axum ที่มี state ซับซ้อนหลายชั้น และมี extractor ที่ตรวจสอบ/แปลงข้อมูล
ก่อนถึง handler ได้ตามต้องการ — **Part 65** จะพาไปดู **middleware** (ผ่าน `tower` และ `tower-http`) ซึ่งเป็น
อีกชั้นหนึ่งของการประมวลผล request/response ที่ครอบทั้ง `Router` (ไม่ใช่ต่อ handler แบบ extractor) — และจะ
อธิบายกลไกการแทรกค่าผ่าน `Extension` ที่เห็นผ่าน ๆ ในหัวข้อ 64.7 อย่างละเอียดเต็มรูปแบบ รวมถึง logging,
CORS, compression, rate limiting ที่แอปจริงทุกตัวต้องมี

---

**Part ก่อนหน้า:** [Axum: Routing และ Handlers](part-063-axum-routing-handlers.md) | **Part ถัดไป:** [Axum: Middleware (tower, tower-http)](part-065-axum-middleware.md)
