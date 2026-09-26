# Part 62: แนะนำ Axum Framework

> โมดูล: การพัฒนาเว็บแอปพลิเคชัน (Web Development) | ระดับ: กลาง | เวลาโดยประมาณ: 210 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้อย่างละเอียดว่า **Axum** คือ layer ไหนของ web stack ใน Rust — ทำไมมันไม่ได้ "เขียน HTTP parser เอง"
  แบบที่ Part 61 ทำ แต่สร้างอยู่บน **`hyper`** (HTTP implementation ระดับต่ำ) และ **`tower`**/**`tower-http`**
  (นามธรรมของ "service" และ middleware) โดยทั้งหมดวิ่งอยู่บน **Tokio runtime** ที่ Part 48-49 สอนไว้ — เข้าใจ
  ว่า Axum "ไม่ได้ผูกขาด" ecosystem แต่เป็นเพียงชั้นบางๆที่ประกอบ `hyper` + `tower` เข้าด้วยกันให้ใช้งานสะดวก
- ติดตั้ง Axum เข้าโปรเจกต์ด้วย `cargo add axum tokio --features tokio/full` แล้วเขียนเซิร์ฟเวอร์ "Hello, World"
  ตัวแรกที่รันได้จริง ใช้ `Router`, `.route()`, handler แบบ `async fn`, และ `axum::serve()` ร่วมกับ
  `tokio::net::TcpListener` (ตัวเดียวกับที่ Part 49 สอน) — ทดสอบด้วย `curl` จริงและเห็น response กลับมาจริง
- อธิบายกลไกเบื้องหลังที่ทำให้ `async fn handler() -> impl IntoResponse` ใช้งานได้ — ผ่าน trait `Handler` และ
  trait `IntoResponse` (การประยุกต์ trait-based dispatch จาก Part 21 และ generic trait bound จาก Part 22)
  ไม่ใช่แค่ "ก็อปโค้ดมาแล้วมันทำงาน" — รู้ว่าอะไรบ้าง implement `IntoResponse` ได้ (`&str`, `String`,
  `(StatusCode, T)`, `Json<T>`, tuple อื่นๆ) และอ่าน compiler error จริงที่เกิดขึ้นเมื่อ handler เขียนผิดได้
- ออกแบบ routing พื้นฐานด้วยหลาย `.route()`, การ nest หนึ่ง `Router` เข้าไปใน `Router` อื่นด้วย `.nest()`,
  และคืนค่า JSON จริงด้วย `axum::Json<T>` ที่ผูกตรงกับ `Serialize` trait ที่ Part 57-58 สอนไว้ — ทดสอบด้วย
  `curl` แล้วอ่าน JSON body จริงที่เซิร์ฟเวอร์ตอบกลับมา
- ใช้ `StatusCode` ร่วมกับ tuple response (`(StatusCode, Json<T>)`) เพื่อควบคุม HTTP status code ที่ส่งกลับ
  ตามสถานการณ์ (เชื่อมกับตาราง status code เต็มรูปแบบที่ Part 61 สอนไว้) และแยกความแตกต่างระหว่าง error ที่
  Axum จัดการเองอัตโนมัติ (เช่น path parse ผิด) กับ error ที่คุณต้องส่งกลับเอง
- สร้างเซิร์ฟเวอร์ API ขนาดเล็กแต่ทำงานได้จริงครบวงจร (ระบบ Tasks API) ที่มี state ที่แชร์ระหว่าง request ด้วย
  `Arc<Mutex<Vec<T>>>` (ตรงกับที่ Part 39 สอนไว้) — ทดสอบทุก endpoint ด้วย `curl -i` จริง เห็น header/body
  ครบถ้วน และเข้าใจว่านี่เป็นแค่ **ขั้นบันไดแรก** ของการจัดการ state ใน Axum (Part 64 จะสอนรูปแบบที่ถูกต้อง
  กว่านี้ผ่าน extractor)

## ความรู้ที่ต้องมีมาก่อน

- **Part 61 (HTTP Fundamentals และ REST API Concepts)**: บทนี้พึ่งพา Part 61 โดยตรง — คุณต้องเข้าใจ HTTP
  request/response, method (`GET`/`POST`/...), status code, header, และเหตุผลที่การ parse HTTP ด้วยมือ
  (แบบที่ Part 61 ให้ลองเขียนเองบน raw `TcpStream`) นำไปสู่ปัญหาที่ framework อย่าง Axum แก้ให้ — ถ้ายังไม่ผ่าน
  Part 61 บทนี้จะดูเหมือน "มายากล" เพราะไม่รู้ว่า Axum ซ่อนงานอะไรไว้ข้างหลัง
- **Part 48 (Tokio: Runtime และ Tasks)** และ **Part 49 (Tokio: I/O และ Networking)**: Axum **ไม่มี** runtime
  ของตัวเอง — มันวิ่งอยู่บน Tokio runtime ที่ Part 48 สอนสร้างด้วย `#[tokio::main]` เต็มๆ และ `TcpListener`
  ที่ Axum ใช้ bind port ก็คือ `tokio::net::TcpListener` ตัวเดียวกับที่ Part 49 สอนสร้าง echo server — บทนี้
  จะไม่อธิบายกลไก async/await หรือ Tokio scheduler ซ้ำจากศูนย์ แต่จะอ้างอิงกลับไปตลอด
- **Part 39 (Smart Pointers: Mutex และ Arc)**: ตัวอย่าง Tasks API ในบทนี้ใช้ `Arc<Mutex<Vec<Task>>>` เพื่อแชร์
  state ระหว่าง request handler หลายตัวที่รันพร้อมกัน — ถ้า Part 39 (โดยเฉพาะเรื่องทำไมต้อง `Arc` ไม่ใช่ `Rc`
  ข้าม thread, และทำไม `.lock()` คืน `Result`) ยังไม่แน่น กลับไปทวนก่อน เพราะบทนี้ใช้ pattern นี้ตรงๆโดยไม่
  อธิบาย mechanism ของ `Mutex` ใหม่
- **Part 21 (Traits ขั้นสูง)** และ **Part 22 (Generics ขั้นสูง)**: หัวข้อ "ทำไม handler ทำงานได้" ในบทนี้อาศัย
  ความเข้าใจเรื่อง trait bound บน generic function, blanket implementation, และการที่ trait หนึ่งตัวสามารถถูก
  implement ให้กับ tuple/type หลายแบบพร้อมกันได้ — ถ้าไม่เคยเห็น blanket impl หรือ trait bound ที่ซับซ้อนกว่า
  `T: SomeTrait` ตัวเดียวมาก่อน ให้ทวน Part 21-22 ก่อน เพราะ error message ของ Axum ที่บทนี้จะแสดงให้ดูจริง
  (`Handler<_, _>` is not implemented) จะอ่านไม่เข้าใจถ้าไม่มีพื้นฐานนี้
- **Part 57 (Serde เบื้องต้น)** และ **Part 58 (Serde ขั้นสูง)**: `axum::Json<T>` ใช้ `Serialize`/`Deserialize`
  trait จาก `serde` ตรงๆ — struct ที่คุณจะคืนเป็น JSON ในบทนี้ต้อง `#[derive(Serialize)]` แบบเดียวกับที่
  Part 57 สอน ไม่มีอะไรพิเศษเพิ่มเติมจาก Axum เอง มันแค่เรียก `serde_json::to_vec()` ให้อัตโนมัติเบื้องหลัง
- **Part 30-31 (Error Handling ขั้นสูง, thiserror/anyhow)**: บทนี้แนะนำ error handling แบบง่ายๆด้วย
  `Result<T, (StatusCode, String)>` เพื่อให้เห็นภาพเร็วที่สุด — แต่การออกแบบ error type ที่ implement
  `IntoResponse` เองอย่างมืออาชีพ (ผูกกับ `thiserror` ที่ Part 30-31 สอน) จะเป็นเนื้อหาเต็มรูปแบบของ **Part 66**
  บทนี้จึงจงใจใช้วิธีง่ายที่สุดก่อน เพื่อไม่ให้หัวข้อ error handling บวมจนกลบเนื้อหาหลักของบท
- **Part 16 (Modules)**: หัวข้อท้ายบทเรื่อง "โครงสร้างโปรเจกต์ที่โตขึ้น" อ้างอิงกลับไปที่ระบบ module ของ Rust
  ที่ Part 16 สอนไว้ตรงๆ — การแยก route ออกเป็นไฟล์/module ต่างๆ ไม่ใช่ฟีเจอร์พิเศษของ Axum แต่เป็นการใช้
  `mod` ธรรมดาจัดระเบียบ `Router` ที่โตขึ้น

## เนื้อหา

### 62.1 ทวนสิ่งที่ Part 61 ทิ้งไว้: เราต้องการอะไรมากกว่า raw `TcpListener`

ใน Part 61 คุณเห็นแล้วว่า HTTP คือแค่ข้อความรูปแบบหนึ่งที่วิ่งอยู่บน TCP stream — request line, header,
บรรทัดว่าง, แล้วอาจจะมี body ตามมา คุณอาจได้ลอง parse มันด้วยมือบน `TcpStream` ที่ Part 49 สอนไว้ (อ่าน
byte, หา `\r\n\r\n`, แยก method/path/version ด้วย `split()`) แล้วก็เห็นว่าแค่จะรองรับ HTTP ให้ถูกต้อง 100%
ตามสเปกจริง (เช่น `Content-Length` เทียบกับ `Transfer-Encoding: chunked`, `Connection: keep-alive`,
pipelining, HTTP/1.1 เทียบกับ HTTP/2) เป็นงานที่ใหญ่มาก ไม่มีทางที่ทีมพัฒนาส่วนใหญ่จะเขียนสิ่งนี้เองซ้ำทุก
โปรเจกต์ได้อย่างสมเหตุสมผล

นี่คือเหตุผลที่ทุกภาษาที่มี ecosystem web ที่โตแล้วจะมี **HTTP library ระดับต่ำ** (low-level) ที่ implement
โปรโตคอลให้ถูกต้องครบถ้วนตามสเปก แล้วมี **web framework** สร้างอยู่ *บนนั้น* อีกชั้นเพื่อให้ประสบการณ์การ
เขียนโค้ดสะดวกขึ้น — เทียบกับภาษาอื่นที่คุณอาจคุ้นเคย:

| ภาษา | HTTP library ระดับต่ำ | Web framework ที่สร้างอยู่บนนั้น |
|---|---|---|
| Rust | `hyper` | Axum, Actix-web (ใช้เอนจินของตัวเอง), Warp |
| Python | (built-in `http.server` ง่ายเกินใช้จริง) | Flask/Django ใช้ WSGI server เช่น Gunicorn |
| Node.js | `http` module ใน Node core | Express, Fastify, Koa |
| Go | `net/http` ใน standard library | Gin, Echo (ทั้งคู่สร้างอยู่บน `net/http`) |
| Java | Servlet API | Spring Boot (สร้างอยู่บน Servlet container เช่น Tomcat) |

Rust ไม่มี HTTP library ใน standard library เลย (ตั้งใจ — standard library ของ Rust เก็บให้เล็กและเสถียร
ที่สุด งานเฉพาะทางอย่าง HTTP ปล่อยให้ ecosystem crate จัดการ) ตำแหน่งของ `hyper` ใน Rust จึงคล้ายกับ
`net/http` ของ Go หรือ Node core's `http` module มากที่สุด — เป็นรากฐานที่ framework ตัวอื่นสร้างต่อ

### 62.2 Axum คืออะไรกันแน่: สร้างโดยทีม Tokio, บน `hyper` + `tower`

**Axum** เป็น web framework ที่สร้างและดูแลโดย**ทีมเดียวกันกับที่สร้าง Tokio** (โปรเจกต์ tokio-rs บน GitHub)
นี่ไม่ใช่รายละเอียดที่ไม่สำคัญ — มันหมายความว่า Axum ถูกออกแบบมาให้เข้ากันสนิทกับ Tokio ตั้งแต่วันแรก ไม่ใช่
framework ที่พยายาม "รองรับหลาย runtime" แล้วต้องประนีประนอมออกแบบ

โครงสร้างแบบเป็นชั้น (layer) ของ Axum เขียนเป็นภาพได้ดังนี้:

```
┌─────────────────────────────────────────┐
│         โค้ดแอปพลิเคชันของคุณ            │  <- Router, handler function ที่คุณเขียน
├─────────────────────────────────────────┤
│                 Axum                     │  <- ให้ ergonomics: Router, extractor, IntoResponse
├─────────────────────────────────────────┤
│          tower / tower-http              │  <- นามธรรม "Service" กลาง + middleware สำเร็จรูป
├─────────────────────────────────────────┤
│                 hyper                    │  <- HTTP/1.1 และ HTTP/2 protocol implementation
├─────────────────────────────────────────┤
│           Tokio runtime                  │  <- async executor, TcpListener, task scheduling (Part 48-49)
├─────────────────────────────────────────┤
│      OS: epoll (Linux) / kqueue (BSD/macOS) / IOCP (Windows)  │
└─────────────────────────────────────────┘
```

ไล่จากล่างขึ้นบน:

**Tokio runtime** (Part 48-49) คือชั้นที่ทำให้ "รอ I/O พร้อมกันหลายพันการเชื่อมต่อโดยไม่ต้องใช้พันเธรด" เป็น
ไปได้ — ผ่าน task scheduler แบบ cooperative multitasking ที่คุณเรียนไปแล้ว Axum เองไม่ได้เขียน event loop
ของตัวเองเลยแม้แต่นิดเดียว มันฝากทุกอย่างไว้กับ Tokio ทั้งหมด — เมื่อคุณเรียก `#[tokio::main]` เหนือ `main()`
ในเซิร์ฟเวอร์ Axum นี่คือ Tokio runtime ตัวเดียวกันเป๊ะกับที่ Part 48 สอน ไม่มีอะไรพิเศษเพิ่มเข้ามา

**`hyper`** คือ crate ที่ implement โปรโตคอล HTTP/1.1 และ HTTP/2 อย่างถูกต้องตามสเปก RFC เต็มรูปแบบ — parse
request line, header, จัดการ `Content-Length`/chunked encoding, keep-alive connection, HTTP/2 multiplexing
ฯลฯ ทั้งหมดที่ Part 61 บอกว่า "ยากและมีรายละเอียดปลีกย่อยมากถ้าจะเขียนเอง" คือสิ่งที่ `hyper` แก้ให้เรียบร้อย
แล้ว `hyper` เป็น crate ที่โฟกัสมากที่สุดในเรื่องความถูกต้องและ performance ของ HTTP โดยตัวมันเองไม่มีแนวคิด
เรื่อง "routing" หรือ "handler function" เลย — มันแค่รับ byte stream แล้วให้คุณได้ request/response object
ที่ parse แล้ว กับ raw byte stream อีกด้านให้คุณเขียน response กลับไป งานของ "จะจับคู่ path ไหนกับฟังก์ชันไหน"
ไม่ใช่หน้าที่ของ `hyper`

**`tower`** คือ crate ที่นิยาม trait กลางตัวหนึ่งชื่อ `Service` ซึ่งเป็นนามธรรมของ "อะไรก็ตามที่รับ request
แล้วคืน response แบบ async" — คิดว่ามันคือเวอร์ชันทั่วไปกว่าของแนวคิด "HTTP handler" แต่ไม่ผูกกับ HTTP โดย
เฉพาะ (`Service` ใช้กับ RPC, gRPC หรือ protocol อื่นก็ได้) จุดสำคัญคือ `tower` ยังให้แนวคิด **middleware**
แบบ composable — คุณสามารถ "ห่อ" `Service` หนึ่งตัวด้วย `Service` อีกตัวเพื่อเพิ่มพฤติกรรม (เช่น timeout,
retry, logging) โดยไม่ต้องแก้โค้ดตัวใน `tower-http` คือชุด middleware สำเร็จรูปที่สร้างอยู่บน `tower` เฉพาะ
สำหรับงาน HTTP (CORS, compression, tracing, request timeout ฯลฯ) — Part 65 (Axum: Middleware) จะสอนเรื่อง
`tower`/`tower-http` แบบเต็มรูปแบบ บทนี้แค่ให้รู้ว่ามันอยู่ตรงไหนของ stack พอ

**Axum เอง** คือชั้นบางๆที่อยู่บนสุด — มันประกอบ `hyper` (สำหรับ HTTP protocol) กับ `tower` (สำหรับนามธรรม
`Service` และ middleware) เข้าด้วยกัน แล้วเพิ่ม ergonomics ที่ทำให้เขียนโค้ดสะดวกมาก: `Router` สำหรับจับคู่
path กับฟังก์ชัน, ระบบ **extractor** (จะเจอใน Part 64) สำหรับดึงข้อมูลจาก request แบบ type-safe, และ trait
`IntoResponse` สำหรับแปลงค่า Rust ธรรมดาให้เป็น HTTP response — สิ่งเหล่านี้คือ "หน้าตา" ของ Axum ที่คุณจะ
เขียนโค้ดจริงในบทนี้และบทถัดไป

**ทำไมเรื่องนี้สำคัญในทางปฏิบัติ?** เพราะมันอธิบายได้ว่าทำไม error message บางอันของ Axum จะพูดถึง `tower`
หรือ `hyper` ตรงๆ (คุณจะเห็นตัวอย่างจริงในหัวข้อ 62.5) — และทำไม middleware ของ `tower`/`tower-http` ใช้กับ
Axum ได้ทันทีโดยไม่ต้องมี "เวอร์ชันพิเศษสำหรับ Axum" — เพราะ Axum ก็คือ `tower::Service` ตัวหนึ่งเช่นกัน
(ทุก `Router` implement trait `Service` ของ `tower`) นี่คือจุดที่ตอบคำถามล่วงหน้าไว้ก่อนที่ Part 65 จะขยาย
ต่อ: middleware ของ `tower-http` "เสียบเข้ากับ" Axum ได้ตรงๆเพราะทั้งคู่พูดภาษาเดียวกันคือ `tower::Service`

### 62.3 ติดตั้ง Axum เข้าโปรเจกต์

สร้างโปรเจกต์ใหม่แล้วเพิ่ม dependency ด้วย `cargo add`:

```bash
cargo new hello_axum
cd hello_axum
cargo add axum
cargo add tokio --features full
```

หรือระบุ feature ของ `tokio` แบบเจาะจงกว่านี้ก็ได้ (ถ้าอยากประหยัดเวลา compile — ทวนจาก Part 48 ว่า
`--features full` ดึงมาทุก feature รวมถึงส่วนที่อาจไม่ได้ใช้) — สำหรับเซิร์ฟเวอร์ Axum พื้นฐาน feature ที่
จำเป็นจริงๆของ `tokio` คือ `rt-multi-thread` (runtime แบบ multi-thread), `macros` (สำหรับ `#[tokio::main]`),
และ `net` (สำหรับ `TcpListener`):

```bash
cargo add tokio --features rt-multi-thread,macros,net
```

บทนี้จะใช้ `--features full` ในตัวอย่างเพื่อความเรียบง่าย (ตรงกับที่ Part 48 แนะนำไว้ตอนเริ่มต้น) แต่รู้ไว้ว่า
ในโปรเจกต์ production จริง การเจาะจง feature list มีผลต่อเวลา compile ไม่น้อยเมื่อโปรเจกต์โตขึ้น

`Cargo.toml` ที่ได้:

```toml
[package]
name = "hello_axum"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = "0.8.9"
tokio = { version = "1.53.1", features = ["full"] }
```

(เวอร์ชันที่ปรากฏคือเวอร์ชันล่าสุดบน crates.io ณ เวลาที่เขียนบทนี้ — `cargo add` จะดึงเวอร์ชันล่าสุดที่มีอยู่
ให้เสมอ ไม่ต้องพิมพ์เลขเวอร์ชันตามนี้เป๊ะๆ)

#### พิสูจน์หัวข้อ 62.2 ด้วย `cargo tree`: เห็น `hyper`/`tower` จริงในโปรเจกต์ตัวเอง

หัวข้อ 62.2 อธิบายว่า Axum สร้างอยู่บน `hyper` และ `tower` — ไม่ต้องเชื่อคำอธิบายเฉยๆ ลองพิสูจน์ด้วยคำสั่ง
`cargo tree` (มาพร้อม cargo อยู่แล้ว ไม่ต้องติดตั้งเพิ่ม) ที่แสดง dependency graph ทั้งหมดของโปรเจกต์:

```bash
cargo tree -p axum
```

output จริง (ตัดมาเฉพาะส่วนที่เกี่ยวข้อง — dependency tree เต็มยาวกว่านี้มาก):

```
axum v0.8.9
├── axum-core v0.5.6
│   ├── bytes v1.12.1
│   ├── futures-core v0.3.34
│   ├── http v1.5.0
│   │   ├── bytes v1.12.1
│   │   └── itoa v1.0.18
│   ├── http-body v1.1.0
│   │   ├── bytes v1.12.1
│   │   └── http v1.5.0 (*)
│   ├── http-body-util v0.1.5
│   │   ├── bytes v1.12.1
│   │   ├── futures-core v0.3.34
│   │   ├── http v1.5.0 (*)
│   │   ├── http-body v1.1.0 (*)
│   │   └── pin-project-lite v0.2.17
...
├── hyper v1.11.1
├── hyper-util v0.1.21
│   ├── hyper v1.11.1 (*)
├── tower v0.5.3
```

เห็น `hyper v1.11.1` และ `tower v0.5.3` เป็น dependency ตรงของ `axum` เป๊ะตามที่หัวข้อ 62.2 อธิบายไว้ —
`http` (crate ที่นิยาม `StatusCode`, `Request`, `Response` ที่ทั้ง `hyper` และ `axum` ใช้ร่วมกัน) ก็ปรากฏ
เป็น dependency ร่วมที่ทั้งสองฝั่งอิงถึง type เดียวกัน (นี่คือเหตุผลที่ `axum::http::StatusCode` ที่จะใช้ใน
หัวข้อ 62.8 คือ re-export ของ `http::StatusCode` ตรงๆ ไม่ใช่ type ที่ Axum นิยามขึ้นมาใหม่เอง) — คำสั่ง
`cargo tree` นี้มีประโยชน์มากเวลาต้อง debug ปัญหาเรื่อง "เวอร์ชันของ crate ตัวกลางไม่ตรงกัน" ในโปรเจกต์ใหญ่
ที่มีหลาย dependency พึ่งพา `hyper`/`tower` คนละเวอร์ชันกัน (ปัญหาที่ Part 35 เรื่อง Cargo ขั้นสูงกล่าวถึงไว้
บ้างแล้วเรื่อง dependency resolution)

### 62.4 Hello World เซิร์ฟเวอร์ตัวแรก

มาเขียนเซิร์ฟเวอร์ Axum ที่เล็กที่สุดที่ยังทำงานได้จริง:

```rust
use axum::routing::get;
use axum::Router;

// handler function: ฟังก์ชัน async ธรรมดาที่คืนค่าอะไรบางอย่างที่แปลงเป็น HTTP response ได้
async fn hello() -> &'static str {
    "Hello, World!"
}

#[tokio::main]
async fn main() {
    // Router::new() สร้าง router เปล่า แล้ว .route() ผูก path เข้ากับ handler
    let app = Router::new().route("/", get(hello));

    // TcpListener ตัวเดียวกับที่ Part 49 สอน — Axum ไม่ได้มี networking ของตัวเอง
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    println!("listening on {}", listener.local_addr().unwrap());

    // axum::serve() รับ listener + router แล้ววิ่ง event loop รับ connection ไปเรื่อยๆ
    axum::serve(listener, app).await.unwrap();
}
```

ไล่ทีละส่วน:

- **`#[tokio::main]`** — attribute macro ตัวเดียวกับ Part 48 สอนไว้ทุกประการ มันแปลง `async fn main()` เป็น
  `fn main()` ธรรมดาที่สร้าง Tokio runtime (ค่าเริ่มต้นคือ `multi_thread` flavor) แล้ว `block_on` future ที่
  เป็น body ของ `main` เดิม — Axum ไม่มีอะไรพิเศษตรงนี้เลย มันคือ Tokio runtime ปกติ 100%
- **`Router::new()`** — สร้าง router เปล่า (ยังไม่มี route ไหนถูกจับคู่)
- **`.route("/", get(hello))`** — ผูก path `"/"` เข้ากับ handler `hello` สำหรับ HTTP method `GET`
  (`axum::routing::get` เป็นฟังก์ชันที่ห่อ handler function ให้กลายเป็นสิ่งที่ `Router` ยอมรับได้ เฉพาะเมื่อ
  method ของ request ตรงกับ `GET`) — Part 63 จะสอนฟังก์ชันคู่กัน `post`, `put`, `delete` ฯลฯ ครบทุก method
- **`tokio::net::TcpListener::bind(...)`** — `TcpListener` ตัวเดียวกับ Part 49 เป๊ะๆ ไม่ใช่ type ของ Axum
  เอง นี่คือหลักฐานที่จับได้ชัดที่สุดว่า Axum "ยืม" networking layer ของ Tokio มาใช้ตรงๆ ไม่ได้เขียนของ
  ตัวเอง
- **`axum::serve(listener, app)`** — ฟังก์ชันที่รับ `TcpListener` กับ `Router` แล้วเริ่ม accept connection
  loop: สำหรับทุก connection ที่เข้ามา มันจะ spawn task ใหม่ (ผ่าน `tokio::spawn` เบื้องหลัง — คู่เทียบกับที่
  Part 49 สอนให้ทำเองตอน "รองรับหลาย client พร้อมกัน") เพื่อจัดการ connection นั้นแบบ concurrent กับ
  connection อื่นๆ โดยไม่ block กัน — เป็น pattern เดียวกับที่ Part 49 สอนแค่ Axum ทำให้อัตโนมัติ

รันเซิร์ฟเวอร์:

```bash
cargo run
```

ผลลัพธ์จริงจาก compile และรัน:

```
   Compiling hello_axum v0.1.0 (/path/to/hello_axum)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 18.19s
     Running `target/debug/hello_axum`
listening on 127.0.0.1:3000
```

เปิด terminal ใหม่ (เซิร์ฟเวอร์ยังรันอยู่ที่ terminal เดิม เพราะ `axum::serve(...).await` จะไม่ return จนกว่า
เซิร์ฟเวอร์จะปิด) แล้วยิง `curl` เข้าไปจริง:

```bash
curl -i http://127.0.0.1:3000/
```

output จริง:

```
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 13
date: Sat, 26 Sep 2026 23:27:07 GMT

Hello, World!
```

สังเกตสิ่งที่ Axum**ทำให้อัตโนมัติ**โดยที่เราไม่ได้เขียนโค้ดเพิ่มแม้แต่บรรทัดเดียว:

- **`content-type: text/plain; charset=utf-8`** — เพราะ handler คืนค่าเป็น `&'static str`, Axum รู้ (ผ่าน
  `IntoResponse` implementation ของ `&str` ที่จะอธิบายในหัวข้อถัดไป) ว่าข้อความธรรมดาควรมี content-type นี้
- **`content-length: 13`** — Axum นับความยาวของ body (`"Hello, World!"` มี 13 byte) ให้อัตโนมัติ ไม่ต้องคำนวณเอง
- **`date:`** header — เพิ่มให้อัตโนมัติตาม RFC 7231 ที่บอกว่า response ควรมี timestamp
- **status line `HTTP/1.1 200 OK`** — เพราะไม่ได้บอก status code อะไรเป็นพิเศษ ค่า default คือ `200 OK`

ลองยิง path ที่ไม่มี route ผูกไว้:

```bash
curl -i http://127.0.0.1:3000/nope
```

output จริง:

```
HTTP/1.1 404 Not Found
content-length: 0
date: Sat, 26 Sep 2026 23:27:07 GMT

```

Axum จัดการ fallback 404 ให้อัตโนมัติเมื่อไม่มี route ไหนตรงกับ path — เทียบกับ Part 61 ที่คุณต้องเขียนโค้ด
เช็คเองว่า "ถ้า path ไม่ตรงกับที่รู้จัก ให้ตอบ 404" นี่คือตัวอย่างแรกที่เห็นชัดว่า framework ประหยัดงานให้
เท่าไหร่ — ทั้ง Content-Length, Date header, และ 404 fallback ล้วนเป็นสิ่งที่ต้องเขียนเองถ้าทำแบบ Part 61
แต่ Axum ให้มาโดยไม่ต้องขอ

### 62.5 กายวิภาคของ Handler: ทำไม `async fn -> impl IntoResponse` ใช้งานได้

หัวข้อนี้คือใจสำคัญที่สุดของบท — เพราะถ้าไม่เข้าใจกลไกนี้ คุณจะแค่ "ก็อปโค้ดจากตัวอย่างมาแก้" โดยไม่รู้ว่าทำไม
บางอย่างทำงานได้และบางอย่างไม่ได้ พอเจอ error message ที่ซับซ้อนของ Axum (จะเห็นจริงในหัวข้อนี้) ก็จะงงสนิท

#### สอง trait ที่เป็นหัวใจของทั้งระบบ: `Handler` และ `IntoResponse`

Axum ตัดสินใจว่า `.route(path, handler)` จะรับ**อะไรก็ได้**ที่ implement trait `Handler<T, S>` (`T` และ `S`
คือ generic parameter ที่ Axum ใช้ภายในสำหรับ extractor และ state — Part 64 จะอธิบาย `S` แบบเต็มรูปแบบตอน
สอน state) แทนที่จะบังคับ signature ตายตัวแบบภาษาอื่น (เช่น Express ของ Node.js ที่บังคับ
`(req, res) => {...}` เป๊ะๆ) — Axum ใช้ **blanket implementation** (concept จาก Part 21-22) เพื่อ implement
`Handler` ให้กับฟังก์ชัน async ที่มี signature หลากหลายรูปแบบโดยอัตโนมัติ ตราบใดที่:

1. พารามิเตอร์แต่ละตัวของฟังก์ชัน implement trait `FromRequest`/`FromRequestParts` (เรียกว่า "extractor" —
   Part 64 จะสอนแบบเต็ม บทนี้ยังไม่ต้องเขียน extractor เอง)
2. ค่าที่ฟังก์ชันคืนกลับมา implement trait `IntoResponse`

นี่คือเหตุผลที่ handler function เขียนได้หลากหลายรูปแบบ — `async fn hello() -> &'static str`,
`async fn hello() -> String`, `async fn hello(Path(id): Path<u32>) -> Json<Task>` ล้วนถูกต้องเพราะ Axum
ไม่ได้เช็ค signature แบบตายตัว แต่เช็คผ่าน trait bound ว่า "แต่ละส่วนของ signature นี้ implement trait ที่
ถูกต้องหรือไม่"

trait `IntoResponse` นิยามไว้ประมาณนี้ (ย่อให้เข้าใจแนวคิด ไม่ใช่นิยามเต็มจริงของ Axum):

```rust
// นี่คือแนวคิดของ trait ที่ Axum ใช้จริง (ประมาณ) ไม่ต้องเขียนเองในโค้ดจริง
trait IntoResponse {
    fn into_response(self) -> axum::response::Response;
}
```

แค่นั้นเลย — อะไรก็ตามที่รู้วิธีแปลงตัวเองเป็น `Response` (struct ที่แทน HTTP response หนึ่งชุด: status,
header, body) ก็ "เป็น response ได้" ตาม Axum Axum เขียน `impl IntoResponse` ให้กับ type พื้นฐานหลายสิบตัว
ไว้ล่วงหน้าให้แล้ว — นี่คือตารางสรุป type ที่ใช้บ่อยที่สุดที่ implement `IntoResponse` มาให้:

| Type | Response ที่ได้ | ตัวอย่าง |
|---|---|---|
| `&'static str` / `String` | `200 OK`, `content-type: text/plain; charset=utf-8` | `"Hello"` |
| `()` (unit type) | `200 OK` ไม่มี body | `async fn h() {}` |
| `StatusCode` | status นั้นๆ ไม่มี body | `StatusCode::NO_CONTENT` |
| `(StatusCode, T)` โดย `T: IntoResponse` | status ตามที่กำหนด + body จาก `T` | `(StatusCode::CREATED, "created")` |
| `(StatusCode, [(HeaderName, &str); N], T)` | status + header เพิ่มเติม + body | ใช้ตอนต้องเพิ่ม custom header |
| `axum::Json<T>` โดย `T: Serialize` | `200 OK`, `content-type: application/json`, body คือ JSON ของ `T` | `Json(vec![1, 2, 3])` |
| `Result<T, E>` โดยทั้ง `T` และ `E` implement `IntoResponse` | `T`'s response ถ้า `Ok`, `E`'s response ถ้า `Err` | `Result<Json<Task>, (StatusCode, String)>` |
| `Html<T>` | `200 OK`, `content-type: text/html; charset=utf-8` | `Html("<h1>hi</h1>")` |
| `Redirect` | `3xx` พร้อม `Location` header | `Redirect::to("/login")` |

จุดที่น่าสนใจที่สุดในตารางนี้คือแถว **`Result<T, E>`** — มันหมายความว่าฟังก์ชัน handler สามารถคืนค่า
`Result` ธรรมดา (แบบเดียวกับที่ Part 12 สอนใช้ทั่วทั้งหลักสูตร) และ Axum จะเลือก response ที่เหมาะสมให้
อัตโนมัติตามว่าเป็น `Ok` หรือ `Err` — นี่คือกลไกที่ตัวอย่าง Tasks API ในหัวข้อ 62.9 จะใช้จริง

#### พิสูจน์ว่า Axum ใช้ trait bound จริงๆ ไม่ใช่ signature ตายตัว: ตัวอย่างที่ compile ไม่ผ่าน

มาดู error message จริงที่เกิดขึ้นเมื่อ handler คืนค่าเป็น type ที่**ไม่ได้** implement `IntoResponse` —
สมมติเผลอคืนค่าเป็น `u32` ตรงๆ (คนอาจคิดว่า "เลขก็ควรแปลงเป็น text ได้เอง"):

```rust
use axum::routing::get;
use axum::Router;

// ตัวอย่าง "ผิด": คืนค่า u32 ตรงๆ ซึ่งไม่มี impl IntoResponse for u32
async fn broken_handler() -> u32 {
    42
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/", get(broken_handler));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

รัน `cargo build` แล้วเจอ error จริงแบบนี้ (capture มาจริง ไม่ได้แต่งขึ้น):

```
error[E0277]: the trait bound `fn() -> impl Future<Output = u32> {broken_handler}: Handler<_, _>` is not satisfied
   --> src/main.rs:11:44
    |
 11 |     let app = Router::new().route("/", get(broken_handler));
    |                                        --- ^^^^^^^^^^^^^^ the trait `Handler<_, _>` is not implemented for fn item `fn() -> impl Future<Output = u32> {broken_handler}`
    |                                        |
    |                                        required by a bound introduced by this call
    |
    = note: Consider using `#[axum::debug_handler]` to improve the error message
note: required by a bound in `axum::routing::get`
   --> .../axum-0.8.9/src/routing/method_routing.rs:167:16
    |
167 |             H: Handler<T, S>,
    |                ^^^^^^^^^^^^^ required by this bound in `get`
...
441 | top_level_handler_fn!(get, GET);
    | -------------------------------
    | |                     |
    | |                     required by a bound in this function
    | in this macro invocation
    = note: this error originates in the macro `top_level_handler_fn` (in Nightly builds, run with -Z macro-backtrace for more info)

For more information about this error, try `rustc --explain E0277`.
error: could not compile `axum_verify` (bin "axum_verify") due to 1 previous error
```

นี่คือ error message ที่**ขึ้นชื่อ**ในหมู่ผู้ใช้ Axum ว่าอ่านยากมากสำหรับผู้เริ่มต้น — เหตุผลเชิงลึกคือ: `get()`
ไม่ได้เขียน error message แบบ "คุณคืนค่า type ที่ผิด" ตรงๆ เพราะในทางเทคนิคมันไม่รู้เลยว่าปัญหาคือส่วนไหนของ
signature — มันแค่รู้ว่า "trait bound `H: Handler<T, S>` ไม่ผ่าน" (ตรงกับที่ Part 22 สอนว่า trait bound ที่
ซับซ้อนบน generic function พอไม่ผ่านจะรายงานที่**จุดเรียกใช้** ไม่ใช่จุดที่ต้นเหตุจริงอยู่) เพราะ `Handler`
เป็น blanket trait ที่ implement ผ่าน macro ภายในของ Axum เอง (`impl_handler!` แบบเดียวกับที่ Part 36 สอน
declarative macro ใช้สร้าง implementation ซ้ำๆหลายแบบ) compiler จึงมองเห็นแค่ "trait ไม่ครบ" โดยไม่รู้จะ
บอกจุดที่แท้จริงยังไง

**Axum แก้ปัญหานี้ให้เอง**ด้วยข้อความ `note: Consider using #[axum::debug_handler]` — ลองทำตาม โดยเปิด
feature `macros` ของ `axum` (`cargo add axum --features macros`) แล้วเพิ่ม attribute เหนือฟังก์ชัน:

```rust
#[axum::debug_handler]
async fn broken_handler() -> u32 {
    42
}
```

compile อีกครั้ง คราวนี้ได้ error ที่ชัดเจนกว่าเดิมมาก (แสดงมาก่อน error เดิม — error เดิมยังคงอยู่ด้วย เพราะ
`get(broken_handler)` ก็ยัง fail อยู่ดี):

```
error[E0277]: the trait bound `u32: IntoResponse` is not satisfied
 --> src/main.rs:6:30
  |
6 | async fn broken_handler() -> u32 {
  |                              ^^^ the trait `IntoResponse` is not implemented for `u32`
  |
  = help: the following other types implement trait `IntoResponse`:
            &'static [u8; N]
            &'static [u8]
            &'static str
            ()
            (R,)
            (Response<()>, R)
            (Response<()>, T1, R)
            (Response<()>, T1, T2, R)
          and 120 others
note: required by a bound in `__axum_macros_check_broken_handler_into_response::{closure#0}::check`
 --> src/main.rs:6:30
  |
6 | async fn broken_handler() -> u32 {
  |                              ^^^ required by this bound in `check`
```

เห็นความแตกต่างชัดมาก — คราวนี้ error บอกตรงๆว่า **`u32` ไม่ implement `IntoResponse`** ชี้ไปที่บรรทัด
`-> u32` เป๊ะๆ พร้อม `help:` แนะนำ type อื่นที่ implement ได้ (list ยาวมาก จึงมีคำว่า "and 120 others" ต่อท้าย
— เป็นการยืนยันจากตัวคอมไพเลอร์เองว่า Axum implement `IntoResponse` ให้กับ type จำนวนมากจริงๆ) — วิธีแก้คือ
เปลี่ยนให้คืนค่าเป็น type ที่ implement `IntoResponse` เช่นแปลงเป็น string ก่อน:

```rust
async fn fixed_handler() -> String {
    42.to_string()
}
```

**บทเรียนสำคัญจากหัวข้อนี้**: ทุกครั้งที่เขียน handler แล้ว Axum ฟ้อง error ยาวๆที่พูดถึง `Handler<_, _>`
`is not satisfied` แบบงงๆ — **ให้ใส่ `#[axum::debug_handler]` เหนือฟังก์ชันนั้นก่อนเสมอ** แล้ว compile ใหม่
จะได้ error ที่ชี้จุดจริงตรงกว่ามาก (ต้องเปิด feature `macros` ของ `axum` ก่อน — ถ้า error บอก
`cannot find attribute debug_handler` แสดงว่ายังไม่ได้เปิด feature นี้) — attribute นี้ใช้แค่ตอน debug
เท่านั้น ลบออกได้เมื่อ handler compile ผ่านแล้ว (หรือจะเก็บไว้ก็ได้ ไม่มีผลต่อ runtime behavior แต่มีผลต่อ
เวลา compile เล็กน้อยเพราะมันเพิ่ม check พิเศษ)

#### ลำดับของ Parameter ใน Handler มีความหมาย: Extractor ที่ "กิน" Body ต้องอยู่ตัวสุดท้าย

อีกจุดหนึ่งที่มาจาก mechanism ของ `Handler` trait ที่ควรรู้ไว้ล่วงหน้าก่อนถึง Part 64 (ที่จะสอน extractor
เต็มรูปแบบ): handler function รับ parameter ได้หลายตัว โดยแต่ละตัวต้อง implement `FromRequestParts` (ดึง
ข้อมูลจากส่วน "หัว" ของ request เช่น path, header, query string — ดึงได้หลายตัวและดึงกี่รอบก็ได้เพราะไม่ได้
"บริโภค" อะไรที่มีอยู่ตัวเดียว) **ยกเว้นตัวสุดท้ายตัวเดียว**ที่อนุญาตให้ implement `FromRequest` เต็มรูปแบบ
(ดึงข้อมูลจากทั้ง request รวม body — ทำได้แค่ครั้งเดียวเพราะ body ของ HTTP request เป็น stream ที่อ่านแล้ว
อ่านซ้ำไม่ได้ ตรงกับที่ Part 49 อธิบายเรื่อง `TcpStream` เป็น byte stream ที่อ่านไปแล้วก็หายไป)

ในตัวอย่างหัวข้อ 62.9 `get_task(State(state): State<Arc<AppState>>, Path(id): Path<u32>)` ทั้งสอง
extractor (`State`, `Path`) เป็นกลุ่มที่ดึงจาก "หัว" ของ request เท่านั้น ไม่มีตัวไหนอ่าน body เลย จึงเรียง
ลำดับกันได้อย่างอิสระ — แต่ถ้ามี extractor ที่อ่าน body (เช่น `Json<T>` ตอนรับ request body เป็น JSON —
Part 63-64 จะสอนใช้จริงตอนทำ `POST`/`PUT`) extractor ตัวนั้นต้องอยู่**ท้ายสุด**ของ parameter list เสมอ:

```rust
// ถูก: Json<T> (กิน body) อยู่ท้ายสุด
async fn create_task(
    State(state): State<Arc<AppState>>,
    Json(payload): Json<CreateTaskRequest>,
) -> StatusCode {
    // ...
    StatusCode::CREATED
}
```

ถ้าสลับลำดับผิด (เอา `Json<T>` ไว้ก่อน `State<T>`) จะได้ error message จาก `#[axum::debug_handler]` ที่บอก
ตรงๆว่า "extractor ตัวนี้ต้องเป็นตัวสุดท้าย" — จุดนี้เป็นเหตุผลอีกข้อที่ควรใส่
`#[axum::debug_handler]` ไว้เสมอตอนพัฒนา เพราะ error ธรรมดา (ไม่มี `debug_handler`) จะรายงานปัญหานี้ผ่าน
`Handler<_, _> is not satisfied` แบบเดียวกับตัวอย่างในหัวข้อนี้ ซึ่งไม่บอกสาเหตุตรงๆเลย

#### เดินตามรอย Request หนึ่งตัว: เกิดอะไรขึ้นบ้างตั้งแต่ `curl` ยิงไปจนได้คำตอบกลับมา

เพื่อสรุปกลไกทั้งหมดของหัวข้อนี้ให้เห็นภาพต่อเนื่อง มาไล่ทีละขั้นว่าเกิดอะไรขึ้นจริงๆตอน
`curl http://127.0.0.1:3000/tasks/1` ถูกยิงเข้าไปที่เซิร์ฟเวอร์ Tasks API ในหัวข้อ 62.9:

1. **OS ระดับ kernel**: connection TCP ใหม่มาถึง socket ที่ bind ไว้ที่ port 3000 — Tokio runtime (ผ่าน
   `epoll` บน Linux ตามที่ Part 48-49 อธิบายไว้) ปลุก task ที่รอ `accept()` อยู่บน `TcpListener`
2. **`axum::serve`**: accept connection สำเร็จ ได้ `TcpStream` ใหม่ตัวหนึ่ง — spawn task ใหม่ (ผ่าน
   `tokio::spawn` เบื้องหลัง) เพื่อจัดการ connection นี้แบบ concurrent กับ connection อื่น
3. **`hyper`**: อ่าน byte จาก `TcpStream` แล้ว parse เป็น HTTP request ที่มีโครงสร้าง (method `GET`, path
   `/tasks/1`, header ต่างๆ) — งาน parsing ทั้งหมดที่ Part 61 ให้ลองเขียนเองเกิดขึ้นที่ชั้นนี้
4. **Axum's `Router`**: รับ request ที่ parse แล้วจาก `hyper` มาเทียบกับ route ที่ประกาศไว้ทั้งหมด — เจอว่า
   `/tasks/{id}` ตรงกับ `/tasks/1` (โดย `id` capture ค่า `"1"` เป็น string ไว้ก่อน)
5. **Extractor layer**: ก่อนเรียก body ของ `get_task` เลย Axum พยายามสร้างค่าให้กับ parameter แต่ละตัวของ
   handler — `State(state): State<Arc<AppState>>` clone `Arc` ที่ผูกไว้ตอน `.with_state()` ออกมา,
   `Path(id): Path<u32>` พยายาม parse string `"1"` เป็น `u32` (สำเร็จ ได้ `1u32`) — **ถ้า parse ไม่สำเร็จ
   (เช่น path เป็น `/tasks/abc`) กระบวนการหยุดตรงนี้เลย** แล้วตอบ `400 Bad Request` กลับไปทันทีโดยไม่เข้า
   body ของ `get_task` เลยแม้แต่นิดเดียว — ตรงกับ output จริงที่เห็นในหัวข้อ 62.10 ข้อ 5
6. **Body ของ handler รันจริง**: `get_task` เริ่มทำงาน — `.lock()` บน `Mutex`, ค้นหา task ที่ `id == 1`,
   เจอ, คืน `Ok(Json(task.clone()))`
7. **`IntoResponse`**: ค่าที่ handler คืนกลับมา (`Result<Json<Task>, ...>`) ถูกแปลงเป็น `axum::Response`
   ผ่าน `into_response()` — เรียก `serde_json::to_vec()` แปลง `Task` เป็น JSON byte, ตั้ง status `200`,
   ตั้ง header `content-type: application/json`
8. **`hyper`**: แปลง `Response` object กลับเป็น byte ตามสเปก HTTP/1.1 (status line, header, บรรทัดว่าง,
   body) เขียนกลับไปที่ `TcpStream`
9. **`curl`**: อ่าน byte ที่ตอบกลับมา parse เป็น HTTP response แสดงผลลัพธ์ที่เห็นในหัวข้อ 62.10

การไล่ทีละขั้นนี้ควรทำให้เห็นชัดว่า Axum "ทำงานหนักให้" ตรงไหนบ้าง (ขั้น 3, 4, 5, 7, 8) เทียบกับสิ่งที่คุณ
เขียนเองแค่ขั้น 6 เท่านั้น — และเห็นว่าทำไม error ที่เกิดจาก extractor (เช่น parse `u32` ไม่ผ่าน) จึงไม่ผ่าน
ไปถึง body ของ handler เลย เพราะมันถูกจับตั้งแต่ขั้น 5 ก่อนที่โค้ดของเราจะได้รันด้วยซ้ำ

### 62.6 Routing พื้นฐาน: หลาย Route และการ Nest Router

`Router` รองรับการต่อ `.route()` หลายครั้งด้วย method chaining (แต่ละ `.route()` คืนค่า `Router` ตัวใหม่ที่
เพิ่ม route เข้ามา — pattern แบบ builder ที่คุ้นเคยจาก Part 35 เรื่อง `cargo` advanced หรือ Part 59 เรื่อง
`clap` builder-style):

```rust
use axum::routing::get;
use axum::Router;

async fn home() -> &'static str {
    "หน้าแรก"
}

async fn about() -> &'static str {
    "เกี่ยวกับเรา"
}

async fn contact() -> &'static str {
    "ติดต่อเรา"
}

fn app() -> Router {
    Router::new()
        .route("/", get(home))
        .route("/about", get(about))
        .route("/contact", get(contact))
}
```

สังเกตว่าดึงการสร้าง `Router` ออกมาเป็นฟังก์ชัน `app()` แยกจาก `main()` — เป็น pattern ที่มีประโยชน์มากตอน
เขียนเทสต์ (Part 63 จะสอนวิธีเทสต์ `Router` โดยไม่ต้องเปิด TCP port จริงด้วย `tower::ServiceExt::oneshot`)

#### `.nest()`: รวม Router ย่อยเข้าเป็น Router ใหญ่

พอแอปพลิเคชันโตขึ้น การมี route ทั้งหมดแบนอยู่ใน `Router` เดียวจะเริ่มอ่านยาก — `.nest(prefix, sub_router)`
ให้คุณสร้าง `Router` ย่อยที่มี path ของตัวเอง แล้ว "เสียบ" เข้าไปใน `Router` หลักโดยเติม prefix ให้อัตโนมัติ:

```rust
use axum::routing::get;
use axum::Router;

async fn list_users() -> &'static str {
    "รายชื่อผู้ใช้ทั้งหมด"
}

async fn list_orders() -> &'static str {
    "รายการคำสั่งซื้อทั้งหมด"
}

fn users_router() -> Router {
    Router::new().route("/", get(list_users))
}

fn orders_router() -> Router {
    Router::new().route("/", get(list_orders))
}

fn app() -> Router {
    Router::new()
        .nest("/users", users_router())
        .nest("/orders", orders_router())
}
```

ผลคือ `users_router()` ที่ประกาศ route `"/"` เอง จะถูก mount ที่ `/users/` จริง (คือเข้าถึงได้ผ่าน
`GET /users/`) และ `orders_router()` mount ที่ `/orders/` — ประโยชน์คือแต่ละ `_router()` function
สามารถเขียนแยกกันโดยไม่ต้องรู้ว่าตัวเองจะถูก mount ที่ prefix ไหนในระบบใหญ่ (decoupling — เชื่อมกับแนวคิด
module boundary ของ Part 16) เมื่อระบบโตขึ้น การแยก `users_router()`/`orders_router()` ไปอยู่คนละไฟล์
(`src/routes/users.rs`, `src/routes/orders.rs`) เป็นขั้นตอนต่อไปที่เป็นธรรมชาติมาก — จะกล่าวถึงในหัวข้อ 62.14

#### เมื่อ Path Literal กับ Path Parameter ชนกัน: Literal ชนะเสมอ

คำถามที่มักเกิดขึ้นตอนออกแบบ routing คือ: ถ้าประกาศทั้ง `.route("/tasks/{id}", ...)` และ
`.route("/tasks/done", ...)` พร้อมกัน แล้ว `GET /tasks/done` จะถูกจับคู่กับ route ไหน? — คำตอบคือ **Axum
(ผ่าน matcher ภายในชื่อ `matchit` — crate ที่ Axum ใช้ทำ routing แบบ radix tree ที่เร็วมาก) ให้ path
segment ที่เป็น literal ตรงตัวชนะเสมอ ไม่ว่าจะประกาศ route ไหนก่อนหรือหลัง**:

```rust
use axum::routing::get;
use axum::Router;

async fn get_task_by_id() -> &'static str {
    "task ตัวเดียวตาม id"
}

async fn get_done_tasks() -> &'static str {
    "task ที่ทำเสร็จแล้วทั้งหมด"
}

fn app() -> Router {
    Router::new()
        // ลำดับการประกาศไม่มีผล — literal "/tasks/done" ชนะ "/tasks/{id}" เสมอ
        .route("/tasks/{id}", get(get_task_by_id))
        .route("/tasks/done", get(get_done_tasks))
}
```

`GET /tasks/done` จะถูกจับคู่กับ `get_done_tasks` เสมอ (ไม่ใช่ `get_task_by_id` ที่พยายาม parse `"done"`
เป็น type ของ `id`) เพราะ literal segment มีความ "เจาะจง" มากกว่า path parameter ในสายตาของ matcher — นี่
คือหลักการเดียวกับที่ระบบ routing ของเกือบทุก framework เว็บใช้ (Express, Rails, Django ทำแบบเดียวกัน) ไม่
ต้องกังวลเรื่องลำดับการเขียน `.route()` ในโค้ด — เขียนตามลำดับไหนก็ได้ตามความสะดวกในการอ่าน

Part 63 (Axum: Routing และ Handlers) จะขยายเรื่อง routing แบบเต็มรูปแบบ — path parameter หลายตัว, wildcard,
method อื่นๆนอกจาก `GET`, และการจัดการ route ที่ชนกันแบบซับซ้อนกว่านี้ บทนี้ให้เห็นแค่พื้นฐานที่พอใช้เขียน
ตัวอย่างจริงต่อไปได้

### 62.7 คืนค่า JSON ด้วย `axum::Json<T>`

งาน API เกือบทั้งหมดในปัจจุบันสื่อสารกันด้วย JSON — Axum ให้ wrapper type `axum::Json<T>` ที่ implement
`IntoResponse` ไว้แล้ว โดย**เงื่อนไขเดียว**คือ `T` ต้อง implement `serde::Serialize` (trait เดียวกันเป๊ะ
กับที่ Part 57 สอน — Axum ไม่ได้คิดกลไก serialize ของตัวเองขึ้นมาใหม่เลย มันเรียก
`serde_json::to_vec(&value)` ให้เบื้องหลังตรงๆ)

```rust
use axum::routing::get;
use axum::{Json, Router};
use serde::Serialize;

#[derive(Debug, Clone, Serialize)]
struct Item {
    id: u32,
    name: String,
    price: f64,
}

async fn list_items() -> Json<Vec<Item>> {
    let items = vec![
        Item { id: 1, name: "เมาส์ไร้สาย".to_string(), price: 299.0 },
        Item { id: 2, name: "แป้นพิมพ์กลไก".to_string(), price: 1490.0 },
        Item { id: 3, name: "หูฟัง Bluetooth".to_string(), price: 890.0 },
    ];
    Json(items)
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/items", get(list_items));
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

`Json(items)` ห่อค่า `Vec<Item>` ไว้ — พอ Axum เห็นค่านี้คืนจาก handler มันจะ:

1. เรียก `serde_json::to_vec(&items)` แปลงเป็น byte ของ JSON (จุดเดียวกันเป๊ะกับที่ Part 57 สอนเรื่อง
   `serde_json::to_string`/`to_vec`)
2. ตั้ง header `content-type: application/json`
3. ใส่ byte ที่ได้เป็น response body

ยิง `curl` เข้าไปจริง:

```bash
curl -i http://127.0.0.1:3000/items
```

output จริง:

```
HTTP/1.1 200 OK
content-type: application/json
content-length: 187
date: Sat, 26 Sep 2026 23:27:43 GMT

[{"id":1,"name":"เมาส์ไร้สาย","price":299.0},{"id":2,"name":"แป้นพิมพ์กลไก","price":1490.0},{"id":3,"name":"หูฟัง Bluetooth","price":890.0}]
```

สังเกตว่า `content-type: application/json` มาให้อัตโนมัติ (ไม่ต้องตั้งเอง) และ JSON array/field ตรงกับชื่อ
field ของ struct เป๊ะๆ (`id`, `name`, `price`) — เพราะ `#[derive(Serialize)]` ตาม default ของ `serde`
(Part 57.5 สอน `rename`/`rename_all` ถ้าต้องการเปลี่ยนชื่อ field ตอนออก JSON เช่นเป็น camelCase สำหรับ
frontend JavaScript ที่คาดหวัง convention นั้น)

### 62.8 Status Code ใน Axum: ควบคุมสถานะการตอบกลับ

Part 61 สอนความหมายของ status code แต่ละกลุ่มไว้แล้ว (`2xx` สำเร็จ, `4xx` client error, `5xx` server error
ฯลฯ) — คำถามตอนนี้คือ "เขียนโค้ด Axum ยังไงให้ควบคุม status code ที่ตอบกลับ" คำตอบคือ enum
`axum::http::StatusCode` (มาจาก crate `http` ที่ทั้ง `hyper` และ Axum ใช้ร่วมกัน — เชื่อมกับหัวข้อ 62.2 ที่
บอกว่า Axum สร้างอยู่บน `hyper`) ที่มีค่าคงที่ให้ครบทุก status code มาตรฐาน:

```rust
use axum::http::StatusCode;

// ตัวอย่างค่าคงที่ที่ใช้บ่อย — ชื่อคือ SCREAMING_SNAKE_CASE ของชื่อ status code
StatusCode::OK;                    // 200
StatusCode::CREATED;                // 201
StatusCode::NO_CONTENT;             // 204
StatusCode::BAD_REQUEST;            // 400
StatusCode::UNAUTHORIZED;           // 401
StatusCode::FORBIDDEN;              // 403
StatusCode::NOT_FOUND;              // 404
StatusCode::UNPROCESSABLE_ENTITY;   // 422
StatusCode::INTERNAL_SERVER_ERROR;  // 500
```

จากตารางใน 62.5 เราเห็นแล้วว่า `StatusCode` เองก็ implement `IntoResponse` (คืน status นั้นแบบไม่มี body)
และ tuple `(StatusCode, T)` ก็ implement ด้วย (status ตามที่กำหนด + body จาก `T`) — นี่คือ pattern ที่ใช้
บ่อยที่สุดในทางปฏิบัติ:

```rust
use axum::http::StatusCode;
use axum::Json;
use serde::Serialize;

#[derive(Serialize)]
struct Item {
    id: u32,
    name: String,
}

async fn create_item() -> (StatusCode, Json<Item>) {
    let new_item = Item { id: 99, name: "สินค้าใหม่".to_string() };
    (StatusCode::CREATED, Json(new_item))
}

async fn health_check() -> (StatusCode, &'static str) {
    (StatusCode::OK, "ok")
}
```

ยิง `curl` เข้าไปจริงกับ handler ที่คืน `(StatusCode::OK, &'static str)`:

```bash
curl -i http://127.0.0.1:3000/health
```

output จริง:

```
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 2
date: Sat, 26 Sep 2026 23:27:43 GMT

ok
```

#### `Result<T, E>` และ Status Code: pattern ที่ใช้จริงในตัวอย่างท้ายบท

เพราะ `Result<T, E>` implement `IntoResponse` เมื่อทั้ง `T` และ `E` implement `IntoResponse` เอง วิธีที่
สะดวกที่สุดในการคืน error พร้อม status code ที่ถูกต้องคือใช้ `(StatusCode, String)` เป็น error type ตรงๆ:

```rust
use axum::http::StatusCode;
use axum::Json;

async fn get_item_by_id(id: u32) -> Result<Json<String>, (StatusCode, String)> {
    if id == 0 {
        // (StatusCode, String) implement IntoResponse -> ใช้เป็น Err ได้เลย
        return Err((StatusCode::BAD_REQUEST, "id ต้องมากกว่า 0".to_string()));
    }
    Ok(Json(format!("item-{id}")))
}
```

นี่คือวิธีที่ง่ายที่สุดในการเริ่มต้น error handling ใน Axum — **หมายเหตุสำคัญ**: วิธีนี้ยังไม่ใช่วิธีที่
"มืออาชีพ" เต็มรูปแบบ (ในระบบจริงมักอยากมี custom error enum ที่ implement `IntoResponse` เองพร้อม logic
ที่ซับซ้อนกว่า — เชื่อมกับ `thiserror` จาก Part 30-31) — **Part 66 (Axum: Error Handling แบบมืออาชีพ)**
จะสอนวิธีที่ถูกต้องเต็มรูปแบบ บทนี้ใช้ `(StatusCode, String)` เพราะเป็นวิธีเร็วที่สุดที่ยังคง type-safe และ
พอเพียงสำหรับตัวอย่างเรียนรู้ในระดับนี้

### 62.9 ตัวอย่างเต็มรูปแบบ: Tasks API พร้อม In-Memory State

มาประกอบทุกอย่างที่เรียนมาเข้าด้วยกันเป็นเซิร์ฟเวอร์จริงที่ทำงานได้ครบวงจร — ระบบ "Tasks API" ขนาดเล็กที่มี
3 endpoint (`GET /health`, `GET /tasks`, `GET /tasks/{id}`) เก็บข้อมูลไว้ใน memory ด้วย
`Arc<Mutex<Vec<Task>>>` (ตรงกับ pattern ที่ Part 39 สอนไว้เต็มรูปแบบ):

```rust
use axum::extract::{Path, State};
use axum::http::StatusCode;
use axum::routing::get;
use axum::{Json, Router};
use serde::Serialize;
use std::sync::{Arc, Mutex};

// Task struct: derive Serialize เพื่อให้ axum::Json<Task> ใช้ได้ (Part 57)
#[derive(Debug, Clone, Serialize)]
struct Task {
    id: u32,
    title: String,
    done: bool,
}

// AppState: state ที่ทุก handler เข้าถึงได้ร่วมกัน — เก็บ Vec<Task> ไว้หลัง Mutex
// เหตุผลที่ต้องใช้ Mutex: หลาย request อาจวิ่งมาพร้อมกันจริง (Axum spawn task ต่อ connection)
// เหตุผลที่ต้องใช้ Arc: state ต้องแชร์ระหว่างหลาย task/thread โดยไม่ copy ข้อมูลจริง (Part 39)
struct AppState {
    tasks: Mutex<Vec<Task>>,
}

// GET /tasks -> คืนรายการ Task ทั้งหมดเป็น JSON
async fn list_tasks(State(state): State<Arc<AppState>>) -> Json<Vec<Task>> {
    let tasks = state.tasks.lock().unwrap();
    Json(tasks.clone())
}

// GET /tasks/{id} -> คืน Task ตัวเดียวถ้าเจอ, 404 ถ้าไม่เจอ
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

// GET /health -> health check endpoint ง่ายๆ สำหรับตรวจสอบว่าเซิร์ฟเวอร์ยังทำงาน
async fn health() -> (StatusCode, &'static str) {
    (StatusCode::OK, "ok")
}

#[tokio::main]
async fn main() {
    let state = Arc::new(AppState {
        tasks: Mutex::new(vec![
            Task { id: 1, title: "เขียนบท Axum".to_string(), done: false },
            Task { id: 2, title: "ทดสอบ curl".to_string(), done: true },
        ]),
    });

    let app = Router::new()
        .route("/health", get(health))
        .route("/tasks", get(list_tasks))
        .route("/tasks/{id}", get(get_task))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}
```

ส่วนใหม่ที่ยังไม่เคยเจอในบทนี้คือ **`State<T>`** extractor และ **`.with_state(state)`** — อธิบายคร่าวๆก่อน
(รายละเอียดเต็มคือเนื้อหาของ **Part 64: Axum State Management และ Extractors**):

- `.with_state(state)` ผูกค่า `state` (ในที่นี้คือ `Arc<AppState>`) เข้ากับ `Router` — Axum เก็บค่านี้ไว้แล้ว
  "ส่งเข้า" ให้ handler ทุกตัวที่ขอมันผ่าน parameter `State(state): State<Arc<AppState>>`
- แต่ละครั้งที่ request เข้ามา Axum จะ **clone `Arc<AppState>`** (การ clone `Arc` คือแค่เพิ่ม reference
  count แบบ atomic — ไม่ได้ copy ข้อมูลจริงข้างใน ตรงกับที่ Part 39 อธิบายเรื่อง `Arc` ไว้) แล้วส่งให้
  handler ตัวนั้นใช้งาน — นี่คือเหตุผลที่ `AppState` ต้องอยู่หลัง `Arc` ตั้งแต่แรก: ถ้าไม่ใช้ `Arc` การส่ง
  state เข้า handler แต่ละตัวจะต้อง copy หรือ move ข้อมูลทั้งชุด ซึ่งไม่สมเหตุสมผลเมื่อ request มาพร้อมกัน
  หลายตัว

**ข้อควรระวังที่สำคัญมาก** (จะกลับมาย้ำในหัวข้อกับดัก): เพราะ handler อาจถูกเรียกพร้อมกันจากหลาย request
จริง (Axum spawn task แยกต่อ connection) การ `.lock()` บน `Mutex` ใน handler เหล่านี้ต้อง**ปลดล็อกให้เร็ว
ที่สุด** — สังเกตว่าในตัวอย่างข้างบน เราเรียก `.lock()`, ทำงานให้จบ (`clone()` หรือ `find()`), แล้วให้
`MutexGuard` หลุด scope (drop) ทันทีก่อนที่จะมี `.await` ใดๆเกิดขึ้นหลังจากนั้น — **ไม่มีจุดไหนที่ถือ lock
ค้างไว้ข้าม `.await`** ซึ่งเป็นกับดักเชิง performance ที่ร้ายแรงมากในโค้ด async (จะอธิบายลึกกว่าในหัวข้อกับดัก
ข้อที่ 4)

**หมายเหตุสำคัญ**: ตัวอย่างนี้เป็นแค่ **ขั้นบันไดแรก** ของการจัดการ state ใน Axum เท่านั้น — การใช้
`Mutex<Vec<T>>` ตรงๆแบบนี้ใช้ได้ดีสำหรับตัวอย่างเรียนรู้และ prototype เล็กๆ แต่ในระบบจริงที่มีข้อมูลจำนวนมาก
หรือต้องเขียนพร้อมกันหนักๆ มักจะเปลี่ยนไปใช้ database จริง (เชื่อมไปยัง Part 70+ เรื่อง SQLx/PostgreSQL) หรือ
โครงสร้าง concurrent-friendly กว่า `Mutex` ธรรมดา — **Part 64** จะสอนรูปแบบ state management ที่ครบถ้วนและ
เป็นระบบกว่านี้ รวมถึง extractor ประเภทอื่นๆที่ Axum มีให้ (`Query`, `Json` แบบ request body, `Extension`)

### 62.10 ทดสอบทุก Endpoint ด้วย `curl -i` จริง

รันเซิร์ฟเวอร์ Tasks API ด้วย `cargo run` แล้วเปิด terminal อีกอันมาทดสอบทีละ endpoint — นี่คือ output จริง
ที่ capture มาจากการรันเซิร์ฟเวอร์ตัวอย่างข้างบนจริงๆ (ไม่ได้แต่งขึ้น):

**1. Health check:**

```bash
curl -i http://127.0.0.1:3000/health
```

```
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 2
date: Sat, 26 Sep 2026 23:27:43 GMT

ok
```

**2. รายการ task ทั้งหมด:**

```bash
curl -i http://127.0.0.1:3000/tasks
```

```
HTTP/1.1 200 OK
content-type: application/json
content-length: 112
date: Sat, 26 Sep 2026 23:27:43 GMT

[{"id":1,"title":"เขียนบท Axum","done":false},{"id":2,"title":"ทดสอบ curl","done":true}]
```

**3. Task ตัวเดียวที่มีอยู่จริง (`id=1`):**

```bash
curl -i http://127.0.0.1:3000/tasks/1
```

```
HTTP/1.1 200 OK
content-type: application/json
content-length: 58
date: Sat, 26 Sep 2026 23:27:43 GMT

{"id":1,"title":"เขียนบท Axum","done":false}
```

**4. Task ที่ไม่มีอยู่จริง (`id=999`) — ควรได้ 404 จากโค้ดของเราเอง:**

```bash
curl -i http://127.0.0.1:3000/tasks/999
```

```
HTTP/1.1 404 Not Found
content-type: text/plain; charset=utf-8
content-length: 27
date: Sat, 26 Sep 2026 23:27:43 GMT

ไม่พบ task id 999
```

สังเกตว่านี่คือ 404 ที่**โค้ดของเราเอง**สร้างขึ้น (จากบรรทัด
`Err((StatusCode::NOT_FOUND, format!("ไม่พบ task id {id}")))`) — ต่างจาก 404 ใน 62.4 ที่ Axum สร้างให้
อัตโนมัติตอน path ไม่มี route ผูกไว้เลย ทั้งสองกรณีได้ status code เดียวกัน (404) แต่มาจากจุดที่ต่างกันโดย
สิ้นเชิงในโค้ด — คนละความหมาย: "path นี้ไม่มีอยู่ในระบบเลย" เทียบกับ "path นี้มีอยู่ แต่ resource ที่ขอไม่พบ"

**5. path parameter ที่ parse เป็น `u32` ไม่ได้ (`id=abc`) — ดูว่า Axum จัดการยังไง:**

```bash
curl -i http://127.0.0.1:3000/tasks/abc
```

```
HTTP/1.1 400 Bad Request
content-type: text/plain; charset=utf-8
content-length: 42
date: Sat, 26 Sep 2026 23:27:43 GMT

Invalid URL: Cannot parse `abc` to a `u32`
```

นี่คือตัวอย่างที่สำคัญมาก — เราไม่ได้เขียนโค้ดจัดการ error นี้เลย! เพราะ handler `get_task` ระบุ signature
เป็น `Path(id): Path<u32>` — Axum พยายาม parse ส่วนของ path เป็น `u32` **ก่อน**ที่จะเรียก body ของฟังก์ชัน
เราเลยด้วยซ้ำ พอ parse ไม่ได้ (`"abc"` ไม่ใช่จำนวนเต็ม) มันปฏิเสธ request ด้วย `400 Bad Request` ทันทีที่
extractor layer (ก่อนเข้าโค้ดเราเลย) — นี่คือประโยชน์ของระบบ extractor ที่ type-safe: การ validate
"path parameter นี้ต้องเป็นเลขจริงๆ" เกิดขึ้นแค่จากการประกาศ type `Path<u32>` เท่านั้น ไม่ต้องเขียน
`.parse::<u32>()` แล้วเช็ค `Result` เองเลยแม้แต่บรรทัดเดียว — Part 64 จะอธิบายกลไก extractor นี้แบบเต็ม
รูปแบบ

### 62.11 Graceful Shutdown เบื้องต้น: ปิดเซิร์ฟเวอร์อย่างสุภาพด้วย Ctrl+C

เซิร์ฟเวอร์ทุกตัวในบทนี้จนถึงตอนนี้ ตอนกด `Ctrl+C` จะถูกฆ่าทันที (SIGINT ทำให้ process จบแบบไม่มีการเตือน) —
ในระบบจริงที่กำลังมี request ค้างอยู่ตอนนั้น (เช่นกำลังเขียนข้อมูลลง database) การถูกตัดกลางคันแบบนี้อาจทำให้
ข้อมูลเสียหายหรือ client ได้รับ connection ที่ถูกปิดกลางทางแบบไม่คาดคิด — **graceful shutdown** คือแนวคิดที่
บอกให้เซิร์ฟเวอร์ "หยุดรับ connection ใหม่ แต่รอให้ request ที่กำลังทำอยู่เสร็จก่อน แล้วค่อยปิดตัวจริงๆ"

Axum รองรับผ่าน `.with_graceful_shutdown(signal_future)` ที่ต่อจาก `axum::serve(...)` — รับ future
อะไรก็ได้ที่จะ resolve เมื่อถึงเวลาที่ควรปิดเซิร์ฟเวอร์ (ปกติคือรอสัญญาณ `Ctrl+C` จาก OS ผ่าน
`tokio::signal::ctrl_c()` ที่ Tokio ให้มา):

```rust
use axum::routing::get;
use axum::Router;

async fn hello() -> &'static str {
    "Hello, World!"
}

async fn shutdown_signal() {
    tokio::signal::ctrl_c()
        .await
        .expect("ติดตั้ง Ctrl+C handler ไม่สำเร็จ");
    println!("ได้รับสัญญาณ Ctrl+C กำลังปิดเซิร์ฟเวอร์อย่างสุภาพ...");
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/", get(hello));
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await.unwrap();

    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await
        .unwrap();
}
```

รันเซิร์ฟเวอร์นี้แล้วกด `Ctrl+C` (หรือส่ง signal `SIGINT` ด้วย `kill -INT <pid>` จาก terminal อื่น) — output
จริงที่ได้ (แทนที่จะถูกฆ่าเงียบๆแบบไม่มีข้อความอะไร เหมือนตัวอย่างก่อนหน้าในบทนี้ทั้งหมด):

```
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.04s
     Running `target/debug/axum_shutdown`
ได้รับสัญญาณ Ctrl+C กำลังปิดเซิร์ฟเวอร์อย่างสุภาพ...
```

`shutdown_signal()` เป็น future ที่ `.await` ค้างอยู่บน `tokio::signal::ctrl_c()` ตลอดเวลาที่เซิร์ฟเวอร์รัน
อยู่ปกติ — พอมันเสร็จ (คือมี `Ctrl+C` เข้ามาจริง) `axum::serve(...).with_graceful_shutdown(...)` จะรู้ว่า
ถึงเวลาต้องหยุด: มันจะสั่งให้ `TcpListener` เลิก accept connection ใหม่ทันที แต่ยังให้ connection ที่กำลัง
ทำงานอยู่ (ถ้ามี) ทำจนจบตามธรรมชาติก่อนที่ `.await` บน `axum::serve(...)` ทั้งก้อนจะ return จริง — เทียบ
กับ Part 48.12 ที่อธิบายว่าการ cancel future ที่ถูกต้องคือ "รอให้จบตามธรรมชาติหรือถึงจุดที่ปลอดภัยจะหยุด"
ไม่ใช่ "ตัดตรงกลางคันแบบดิบๆ" — graceful shutdown ของ Axum ก็ใช้หลักการเดียวกันนี้เพื่อความปลอดภัยของข้อมูล

ในระบบจริง (โดยเฉพาะที่ deploy บน container/Kubernetes) มักต้องดักทั้ง `SIGINT` (Ctrl+C ตอน dev) และ
`SIGTERM` (สัญญาณที่ orchestrator ส่งมาบอกว่า "กำลังจะปิด container นี้") ร่วมกัน — เนื้อหาการดัก signal
ทั้งสองแบบพร้อมกันแบบ production-ready เต็มรูปแบบจะอยู่ในบทที่เกี่ยวกับ deployment ในโมดูลถัดๆไปของหลักสูตร
บทนี้แนะนำแค่แนวคิดพื้นฐานที่สุด (`Ctrl+C` เพียงอย่างเดียว) ให้รู้จักไว้ก่อน

### 62.12 Loop การพัฒนา: ทดสอบด้วยมือ vs `cargo watch`

ระหว่างพัฒนา คุณจะต้อง หยุดเซิร์ฟเวอร์ (`Ctrl+C`), แก้โค้ด, แล้ว `cargo run` ใหม่ ซ้ำไปเรื่อยๆ — วนแบบนี้เอง
ก็ใช้ได้ (และเป็นสิ่งที่บทนี้ทำมาตลอดในการ capture output จริงทุกตัวอย่าง) แต่ในทางปฏิบัติ มี tool ชื่อ
**`cargo watch`** ที่ช่วยให้ loop นี้เร็วขึ้นได้ — มันคือ cargo subcommand (ติดตั้งด้วย
`cargo install cargo-watch`) ที่เฝ้าดูไฟล์ source แล้วรัน command ที่กำหนดให้อัตโนมัติทุกครั้งที่ไฟล์เปลี่ยน:

```bash
cargo install cargo-watch
cargo watch -x run
```

`cargo watch -x run` แปลว่า "ทุกครั้งที่ไฟล์ `.rs` เปลี่ยน ให้รัน `cargo run` ใหม่อัตโนมัติ" — เซิร์ฟเวอร์จะ
build และรันใหม่ทุกครั้งที่ save ไฟล์ ไม่ต้องกด `Ctrl+C` แล้ว `cargo run` เองด้วยมือ (เทียบได้กับ
`nodemon` ใน Node.js ecosystem หรือ `air` ใน Go ecosystem ถ้าคุณคุ้นเคยจากภาษาอื่น) — บทนี้ระบุไว้แค่ให้รู้
ว่า tool นี้มีอยู่และช่วยประหยัดเวลาได้จริงในการพัฒนาต่อเนื่อง แต่**ไม่ใช่ของบังคับ**สำหรับเรียนบทนี้ — ทุก
ตัวอย่างในบทนี้ทดสอบด้วยการรัน `cargo run` ธรรมดาสลับกับ `curl` จาก terminal อีกอันก็ใช้งานได้เหมือนกันทุก
ประการ (`cargo watch` แค่ลดจำนวนครั้งที่ต้องพิมพ์คำสั่งด้วยมือ)

### 62.13 Axum vs Actix-web vs Rocket: มุมมองคร่าวๆก่อนเจาะลึก

Rust มี web framework หลักอยู่สามตัวที่ได้รับความนิยมสูงในปัจจุบัน — บทนี้ให้ภาพรวมสั้นๆพอให้เข้าใจตำแหน่ง
ของแต่ละตัว ไม่ลงรายละเอียดเจาะจึก (ตารางเปรียบเทียบแบบละเอียดครบทุกมุม รวม benchmark และตัวอย่างโค้ดคู่กัน
คือเนื้อหาของ **Part 69: เปรียบเทียบ Axum vs Actix-web vs Rocket** — **Part 67-68** จะสอน Actix-web แบบ
hands-on เต็มรูปแบบก่อนถึงจุดนั้น):

| Framework | จุดเด่น | สร้างอยู่บน | สไตล์ |
|---|---|---|---|
| **Axum** (บทนี้) | ผูกกับ Tokio/tower แน่นมาก, type-safe extractor, ระบบ error handling ที่ยืดหยุ่นสูง, ทีม Tokio ดูแลเอง | `hyper` + `tower` | function-based handler, ergonomics จาก trait system ของ Rust |
| **Actix-web** | เร็วที่สุดใน benchmark ส่วนใหญ่ (TechEmpower), เก่าแก่ที่สุด (เสถียร, ecosystem ใหญ่), มี actor model ในตัว (แม้ปัจจุบันไม่บังคับใช้) | เอนจิน HTTP ของตัวเอง (ไม่ใช้ `hyper`) | macro attribute ต่อ route (`#[get("/")]`) |
| **Rocket** | เน้น developer experience/ergonomics สูงสุด, error message อ่านง่ายมาก, มี request guard ที่ทรงพลัง | `hyper` (ตั้งแต่เวอร์ชัน 0.5 เป็นต้นมา) | macro attribute ต่อ route คล้าย Actix-web แต่มี compile-time check เข้มกว่า |

**Axum** (ที่เรียนในบทนี้) เหมาะที่สุดถ้าคุณต้องการความสอดคล้องแน่นกับ Tokio ecosystem ที่เหลือ (ซึ่งเป็นสิ่งที่
หลักสูตรนี้สอนมาตั้งแต่ Part 46-50) และชอบสไตล์ "function ธรรมดาที่ type บอกทุกอย่าง" มากกว่า attribute
macro ต่อ route — มันคือทางเลือกที่คนเขียน Rust จำนวนมากแนะนำให้เริ่มต้นด้วยในปัจจุบัน เพราะ mental model
ที่ต้องเรียนรู้เพิ่มมีน้อยที่สุด (ส่วนใหญ่คือการรู้จัก trait หลักไม่กี่ตัว: `Handler`, `IntoResponse`,
extractor trait) เมื่อเทียบกับ macro attribute แบบ Actix-web/Rocket ที่ซ่อนงานไว้เยอะกว่า

**Actix-web** (Part 67-68) น่าสนใจถ้าประสิทธิภาพระดับสูงสุดคือเป้าหมายหลัก หรือทีมคุ้นเคยกับ actor model
มาก่อน — แลกกับการที่มันมี concept เฉพาะตัวเพิ่มขึ้น (เช่น `App`/`HttpServer` ที่ต่างจาก Axum's `Router`)

**Rocket** เหมาะกับทีมที่ให้ความสำคัญกับ compile-time safety และ developer experience สูงสุด — แต่ปัจจุบัน
มี ecosystem/community เล็กกว่า Axum และ Actix-web พอสมควร ทำให้หาตัวอย่าง/ความช่วยเหลือจาก community ยาก
กว่าเล็กน้อยเมื่อเจอปัญหาเฉพาะทาง

หลักสูตรนี้เลือกสอน **Axum ก่อน** (Part 62-66) เพราะมันต่อยอดจาก Tokio ที่สอนไปแล้วโดยตรงที่สุด แล้วค่อยไป
เรียน **Actix-web** (Part 67-68) เพื่อเห็นสไตล์ที่ต่างออกไป ก่อนจะสรุปเปรียบเทียบทั้งสามแบบละเอียดใน
**Part 69**

### 62.14 โครงสร้างโปรเจกต์: เมื่อ `main.rs` เดียวไม่พอแล้ว

ทุกตัวอย่างในบทนี้ใส่ทุกอย่าง — struct, handler, `main()` — ไว้ใน `src/main.rs` ไฟล์เดียว ซึ่ง**ใช้ได้ดี
มากสำหรับตัวอย่างเรียนรู้และโปรเจกต์เล็กๆ** (ไม่มีข้อเสียอะไรถ้าโปรเจกต์ยังมี route ไม่เกิน 5-10 ตัว) — แต่
พอแอปพลิเคชันโตขึ้นถึงจุดที่มี handler หลายสิบตัว, หลาย resource (users, orders, products, ...) การรวมทุก
อย่างไว้ไฟล์เดียวจะเริ่มอ่านยากและ merge conflict บ่อยขึ้นถ้าทำงานเป็นทีม

Rust ไม่มีอะไรพิเศษเฉพาะของ Axum สำหรับเรื่องนี้ — วิธีแก้คือใช้ระบบ **module** ธรรมดาที่ Part 16 สอนไว้
ตรงๆ pattern ที่นิยมมากในโปรเจกต์ Axum จริงคือแยกตาม "resource" (สอดคล้องกับที่ REST API ออกแบบตาม resource
อยู่แล้ว — Part 61 สอนแนวคิดนี้ไว้):

```
src/
├── main.rs           # แค่ setup Router หลัก + เริ่ม server
├── state.rs          # นิยาม AppState และ logic ที่เกี่ยวกับ state
└── routes/
    ├── mod.rs         # pub mod tasks; pub mod users; ...
    ├── tasks.rs        # handler + router สำหรับ /tasks/*
    └── users.rs        # handler + router สำหรับ /users/*
```

ตัวอย่างคร่าวๆของการแยก (แสดงแค่โครงเพื่อให้เห็นภาพ ไม่ใช่ตัวอย่างที่ต้อง copy ไปรันจริงในบทนี้):

```rust
// src/routes/tasks.rs
use axum::routing::get;
use axum::Router;
// ... use ที่จำเป็นอื่นๆ

pub fn router() -> Router<crate::state::SharedState> {
    Router::new()
        .route("/", get(list_tasks))
        .route("/{id}", get(get_task))
}

async fn list_tasks(/* ... */) { /* ... */ }
async fn get_task(/* ... */) { /* ... */ }
```

```rust
// src/main.rs
mod routes;
mod state;

#[tokio::main]
async fn main() {
    let state = state::build();

    let app = axum::Router::new()
        .nest("/tasks", routes::tasks::router())
        .nest("/users", routes::users::router())
        .with_state(state);

    // ... bind + serve เหมือนเดิม
}
```

สังเกตว่า `.nest()` (จากหัวข้อ 62.6) คือกลไกที่ทำให้การแยกไฟล์แบบนี้ทำงานได้อย่างเป็นธรรมชาติ — แต่ละไฟล์
route ประกาศ `Router` ของตัวเองแบบไม่ต้องรู้ prefix ของตัวเอง แล้ว `main.rs` เป็นจุดเดียวที่ตัดสินใจว่าแต่
ละกลุ่ม route จะ mount ที่ path ไหน — **นี่ไม่ใช่ฟีเจอร์พิเศษของ Axum แต่เป็นแค่การใช้ `mod`/`pub fn` ธรรมดา
ของ Rust ร่วมกับ `Router` ที่เป็นค่าธรรมดาที่ pass ไปมาได้** (เพราะ `Router` implement `Clone` และเป็นแค่
`struct` ธรรมดา ไม่มีอะไรวิเศษ) — บทนี้ไม่ลงรายละเอียดการแยกไฟล์แบบนี้เพิ่ม เพราะตัวอย่างในบทยังเล็กพอที่จะ
อยู่ไฟล์เดียวได้สบายๆ แต่รู้ไว้ล่วงหน้าว่านี่คือทิศทางที่โปรเจกต์จริงจะเดินไปเมื่อโตขึ้น

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. Handler ไม่ implement `Handler` trait — error message ที่งงที่สุดของ Axum

อาการ: compile error ยาวมากที่พูดถึง `Handler<_, _>` `is not satisfied` ไม่บอกตรงๆว่าปัญหาอยู่ที่ไหนของ
signature — นี่คือ error จริงที่เจอตอน handler คืน `u32` ตรงๆ (ดูรายละเอียดเต็มในหัวข้อ 62.5):

```
error[E0277]: the trait bound `fn() -> impl Future<Output = u32> {broken_handler}: Handler<_, _>` is not satisfied
   --> src/main.rs:11:44
    |
 11 |     let app = Router::new().route("/", get(broken_handler));
    |                                        --- ^^^^^^^^^^^^^^ the trait `Handler<_, _>` is not implemented for fn item ...
    = note: Consider using `#[axum::debug_handler]` to improve the error message
```

**วิธีแก้**: ใส่ `#[axum::debug_handler]` เหนือฟังก์ชัน handler ที่มีปัญหา (ต้องเปิด feature `macros` ของ
`axum` ก่อน: `cargo add axum --features macros`) แล้ว compile ใหม่ — จะได้ error ที่ชี้จุดจริงชัดเจนกว่ามาก
เช่น `the trait bound `u32: IntoResponse` is not satisfied` ที่ชี้ตรงไปยัง return type ที่ผิด สาเหตุที่พบ
บ่อยที่สุดของ error กลุ่มนี้: (ก) คืนค่า type ที่ไม่ implement `IntoResponse`, (ข) parameter ของ handler ใช้
extractor ผิด order (extractor ที่ consume request body เช่น `Json<T>` ต้องอยู่**ตัวสุดท้าย**เสมอ — เรื่องนี้
Part 64 จะอธิบายลึกกว่านี้), (ค) ลืม `.await` บางจุดทำให้ return type ผิดไปจากที่คาดหวัง

### 2. Path parameter syntax เก่า (`:id`) ใช้กับ Axum 0.8+ ไม่ได้แล้ว

Axum เปลี่ยน syntax ของ path parameter ระหว่างเวอร์ชัน 0.7 กับ 0.8 — จากเดิม `:id` เป็น `{id}` (ทำตาม
มาตรฐาน OpenAPI/path template ทั่วไปมากขึ้น) ถ้าเขียนตามตัวอย่างเก่าจาก blog/tutorial ที่ยังใช้ syntax เดิม
(เช่นจากคู่มือหรือ blog post ที่เขียนไว้ตั้งแต่ยุค Axum 0.6-0.7):

```rust
let app = Router::new().route("/tasks/:id", get(get_task));
```

โค้ดนี้ **compile ผ่าน** (เพราะ `.route()` รับ `&str` ธรรมดาเป็น path — compiler ไม่รู้ syntax path จนกว่า
จะรันจริง) แต่พอรันจริงจะ **panic ทันทีตอน `.route()` ถูกเรียก** ด้วย panic message จริงแบบนี้:

```
thread 'main' panicked at src/main.rs:10:29:
Path segments must not start with `:`. For capture groups, use `{capture}`. If you meant to literally match a segment starting with a colon, call `without_v07_checks` on the router.
```

**วิธีแก้**: เปลี่ยน `:id` เป็น `{id}` ทุกจุด (`"/tasks/:id"` → `"/tasks/{id}"`) — panic message เองก็บอก
วิธีแก้ตรงๆอยู่แล้ว (เป็นตัวอย่างที่ดีของ error message ที่ออกแบบมาให้ actionable) หมายเหตุ: ถ้าจำเป็นต้อง
รองรับ syntax เก่าจริงๆ (เช่นย้าย codebase ใหญ่มาจากเวอร์ชันก่อน) มีเมธอด `.without_v07_checks()` ที่ปิด
การเช็คนี้ได้ แต่ **ไม่แนะนำ**สำหรับโค้ดใหม่ — เขียนด้วย syntax `{id}` ตั้งแต่แรกดีกว่าเสมอ

### 3. `Router` ที่มี `.with_state(...)` แล้วยังใส่ `State<T>` ผิด Type

ถ้า `.with_state(state)` ถูกเรียกด้วยค่าประเภทหนึ่ง แต่ handler ขอ `State<T>` เป็นอีกประเภท (เช่น ลืม `Arc`,
หรือชนิดไม่ตรงกัน) จะได้ compile error ที่บอกตรงๆว่า state type ไม่ตรงกัน ตัวอย่างเช่นถ้า handler เขียนว่า
`State(state): State<AppState>` (ไม่มี `Arc`) แต่ `main()` เรียก `.with_state(Arc::new(AppState { ... }))`
จะได้ errorประมาณ:

```
error[E0308]: mismatched types
  expected `AppState`, found `Arc<AppState>`
```

**วิธีแก้**: ให้ type ที่ `.with_state(...)` ส่งเข้าไป กับ type parameter ของ `State<T>` ในทุก handler
**ตรงกันเป๊ะ** — ถ้าใช้ `Arc<AppState>` ตอน `.with_state()` ทุก handler ต้องขอ `State<Arc<AppState>>`
เหมือนกันหมด (แนวทางที่แนะนำ: ตัดสินใจตั้งแต่แรกว่าจะใส่ `Arc` ไว้ *ข้างใน* `AppState` เอง (field เป็น
`Arc<Mutex<...>>` ต่อ field) หรือห่อ `AppState` ทั้งก้อนด้วย `Arc` จากนอก (แบบที่บทนี้ทำ) — สองแบบนี้ทำงาน
ได้ทั้งคู่แต่ห้ามผสมกันโดยไม่ตั้งใจ)

### 4. ถือ `MutexGuard` ค้างข้าม `.await` — Deadlock หรือ Performance ที่แย่ลงมาก

ถ้าเขียนโค้ดแบบนี้ (ดูดีๆ ต่างจากตัวอย่างในหัวข้อ 62.9 เล็กน้อยแต่อันตรายมาก):

```rust
async fn bad_handler(State(state): State<Arc<AppState>>) -> String {
    let tasks = state.tasks.lock().unwrap(); // ล็อกไว้
    some_async_operation().await;             // <-- lock ยังถือค้างอยู่ตรงนี้!
    format!("{} tasks", tasks.len())
}
```

โค้ดนี้ **compile ผ่าน** เพราะ `std::sync::Mutex`'s `MutexGuard` ไม่ได้ผูกกับ concept ของ async โดยตรง —
แต่ในทางปฏิบัติมันคือกับดักร้ายแรง: ระหว่างที่ task นี้ `.await` อยู่ (อาจถูก scheduler สลับไปทำ task อื่น
บน worker thread เดียวกัน — ตรงกับกลไก cooperative multitasking ที่ Part 48 สอนไว้) lock ยังถูกถือค้างอยู่
— ถ้า task อื่นที่ถูกสลับมาทำงานพยายาม `.lock()` ตัวเดียวกัน มันจะต้องรอจนกว่า task แรกจะกลับมาทำงานต่อและ
ปลดล็อก ซึ่งถ้า worker thread ตันหรือ task แรกใช้เวลานานผิดปกติ (`some_async_operation()` ที่ทำ I/O ช้า)
ทุก request ที่ต้องการ lock ตัวนี้จะติดคอขวดรอกันเป็นแถว — ในกรณีที่ซับซ้อนกว่านี้ (เช่นถือ lock ข้าม
`.await` แล้ว future ที่รอถูก cancel กลางทางแบบ Part 48.12 สอนไว้) อาจนำไปสู่ deadlock จริงได้

**วิธีแก้**: ทำตามที่ตัวอย่างหัวข้อ 62.9 ทำไว้แล้ว — ล็อก, ทำงานที่ต้องใช้ข้อมูลให้จบ (clone ข้อมูลออกมาถ้า
จำเป็น), แล้วปล่อยให้ `MutexGuard` หลุด scope (drop) **ก่อน** จุดที่มี `.await` ใดๆเกิดขึ้น — ถ้าจำเป็นต้อง
ทำงาน async ระหว่างที่ต้องใช้ข้อมูลจาก state จริงๆ ให้พิจารณาใช้ `tokio::sync::Mutex` แทน (Part 50 สอนไว้ว่า
`tokio::sync::Mutex` ออกแบบมาให้ปลอดภัยกว่าตอนถือข้าม `.await` — แต่ก็ยังควรถือให้สั้นที่สุดเสมออยู่ดี ไม่ใช่
ใบอนุญาตให้ถือยาวๆได้อย่างสบายใจ)

### 5. ใช้ `#[axum::debug_handler]` โดยไม่ได้เปิด feature `macros` — `E0433`

หัวข้อ 62.5 แนะนำให้ใส่ `#[axum::debug_handler]` เพื่อดู error message ที่ชัดเจนกว่า — แต่ attribute นี้อยู่
หลัง feature flag `macros` ของ `axum` ที่**ไม่ได้เปิดมาให้ตาม default** ถ้า `Cargo.toml` มีแค่
`axum = "0.8.9"` (ไม่มี `features = ["macros"]`) แล้วใส่ attribute นี้ไปตรงๆ จะได้ error จริงแบบนี้:

```
error[E0433]: failed to resolve: could not find `debug_handler` in `axum`
 --> src/main.rs:4:9
  |
4 | #[axum::debug_handler]
  |         ^^^^^^^^^^^^^ could not find `debug_handler` in `axum`
  |
note: found an item that was configured out
 --> .../axum-0.8.9/src/lib.rs:526:23
  |
525 | #[cfg(feature = "macros")]
  |       ------------------ the item is gated behind the `macros` feature
526 | pub use axum_macros::{debug_handler, debug_middleware};
  |                       ^^^^^^^^^^^^^
```

สังเกตว่า compiler ใจดีมากในกรณีนี้ — มันบอกตรงๆเลยว่า item นี้ "ถูก configure out" เพราะ feature `macros`
ไม่ได้เปิด (ต่างจากกับดักข้อ 1 ที่ error message ไม่บอกสาเหตุตรงๆ) **วิธีแก้**: เปิด feature ให้ครบด้วย
`cargo add axum --features macros` (หรือแก้ `Cargo.toml` ตรงๆเป็น
`axum = { version = "0.8.9", features = ["macros"] }`) — ทวนจาก Part 17/35 เรื่อง feature flag: การที่
Axum ซ่อน `debug_handler` ไว้หลัง feature ที่ต้องเปิดเองแบบนี้ (ไม่ใช่เปิดมาให้ default) เป็นการตัดสินใจ
ออกแบบเพื่อให้ dependency tree ของ Axum เองไม่ดึง `axum-macros` (ซึ่งมี dependency ของตัวเองเช่น `syn`)
เข้ามาโดยไม่จำเป็นสำหรับโปรเจกต์ที่ไม่ได้ใช้ macro พวกนี้เลย — ประหยัดเวลา compile ให้กับโปรเจกต์ที่ไม่ต้องการ

### 6. เรียกฟังก์ชัน Blocking ตรงๆ ใน Handler — บล็อก Worker Thread ทั้งเส้น

กับดักนี้ไม่ใช่เรื่องใหม่ของ Axum โดยตรง แต่เป็นกับดักของ Tokio ที่ Part 48.11 อธิบายไว้เต็มรูปแบบแล้ว — เพียง
แค่**อันตรายกว่าเดิมมากในบริบทของเว็บเซิร์ฟเวอร์** เพราะ handler ทุกตัวรันอยู่บน worker thread pool เดียวกัน
ที่ต้องรองรับ**ทุก request ที่เข้ามาพร้อมกัน** ถ้า handler ตัวหนึ่งเรียกโค้ด blocking (เช่น
`std::thread::sleep`, การอ่านไฟล์แบบ synchronous ด้วย `std::fs::read`, หรือ query database แบบ blocking):

```rust
async fn slow_handler() -> &'static str {
    std::thread::sleep(std::time::Duration::from_secs(5)); // อันตราย! บล็อก worker thread ทั้งเส้น
    "เสร็จแล้ว"
}
```

โค้ดนี้ **compile ผ่านและทำงานได้** — request ที่เรียก `/slow` จะได้คำตอบกลับมาถูกต้องหลัง 5 วินาที แต่
ปัญหาคือ**ระหว่าง 5 วินาทีนั้น worker thread ที่ทำ task นี้จะทำอะไรอื่นไม่ได้เลย** (ตรงกับที่ Part 48.11
อธิบายไว้ว่า cooperative multitasking ต้องพึ่ง task ที่ "ให้ความร่วมมือ" คืน control กลับให้ scheduler ที่
จุด `.await` — แต่ `std::thread::sleep` ไม่มีจุด `.await` เลย มันบล็อก OS thread ตรงๆ) — ถ้าเซิร์ฟเวอร์รัน
ด้วย `multi_thread` runtime (ค่า default ของ `#[tokio::main]`) ที่มี worker thread เท่าจำนวน CPU core และ
มีคนยิง request `/slow` พร้อมกันมากกว่าจำนวน worker thread ที่มี **request อื่นๆทั้งหมด (แม้จะเป็น
endpoint ที่เร็วมากอย่าง `/health`) จะค้างรอไม่ได้ตอบเลย** จนกว่า worker thread ที่ถูกบล็อกจะว่าง

**วิธีแก้**: ใช้ async-native alternative เสมอสำหรับงาน I/O (เช่น `tokio::time::sleep` แทน
`std::thread::sleep`, `tokio::fs::read` แทน `std::fs::read` — ตรงกับที่ Part 49.2 สอนไว้เรื่อง async file
I/O) และถ้าเลี่ยงไม่ได้จริงๆที่ต้องเรียกโค้ด CPU-bound หรือ blocking library ที่ไม่มีเวอร์ชัน async ให้ห่อ
มันด้วย `tokio::task::spawn_blocking(...)` (Part 48.11 สอนวิธีใช้ไว้เต็มรูปแบบ) เพื่อย้ายงาน blocking นั้น
ไปทำบน thread pool แยกที่ Tokio จัดไว้เฉพาะสำหรับงานประเภทนี้ ไม่ให้กระทบ worker thread ที่ต้อง serve
request อื่นๆ

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เพิ่ม route ใหม่ `GET /about` ให้กับเซิร์ฟเวอร์ Hello World ในหัวข้อ 62.4 ที่คืนข้อความ
   `"เซิร์ฟเวอร์นี้เขียนด้วย Axum"` แบบ `&'static str` แล้วทดสอบด้วย `curl -i` ว่า status code, header, และ
   body ถูกต้อง — *hint*: เพิ่ม handler function ใหม่ แล้วต่อ `.route(...)` อีกครั้งบน `Router` ที่มีอยู่
   (ใช้ pattern เดียวกับหัวข้อ 62.6 ที่มีหลาย `.route()` ต่อกัน)

2. **(กลาง)** ขยายตัวอย่าง Tasks API ในหัวข้อ 62.9 ให้มี endpoint ใหม่ `GET /tasks/done` ที่คืนเฉพาะ task
   ที่ `done == true` เป็น JSON array (ไม่ใช่ task ทั้งหมด) — *hint*: ระวังเรื่อง route ordering — Axum
   จับคู่ path แบบตายตัว (ไม่ใช่ regex) ดังนั้น `/tasks/done` กับ `/tasks/{id}` เป็นสอง route คนละตัวที่ไม่
   ชนกัน (Axum จะจับคู่ path ตรงตัวก่อน path parameter เสมอ) ลองเพิ่ม route นี้แล้วดูว่า `curl` ไปที่
   `/tasks/done` กับ `/tasks/1` ยังคงตอบถูกทั้งคู่หรือไม่

3. **(ยาก)** เพิ่ม endpoint ใหม่ `GET /tasks/stats` ที่คืน JSON object รูปแบบ
   `{"total": N, "done": M, "pending": K}` (นับจาก state ที่มีอยู่) — ต้องนิยาม struct ใหม่
   `#[derive(Serialize)] struct Stats { total: u32, done: u32, pending: u32 }` แล้วคำนวณค่าจาก
   `state.tasks.lock().unwrap()` — *hint*: อย่าลืมว่า `.route("/tasks/stats", ...)` ต้องถูกประกาศ**ก่อน**
   หรือ**หลัง** `.route("/tasks/{id}", ...)` ก็ได้ (ไม่มีผลต่อผลลัพธ์เพราะ literal segment ชนะ path
   parameter เสมอตามที่ข้อ 2 อธิบายไว้) — ทดสอบว่า `curl http://127.0.0.1:3000/tasks/stats` ไม่ถูก
   `get_task` (ที่คาดหวัง `Path<u32>`) ไปแย่งจับคู่โดยไม่ตั้งใจ

4. **(ยาก/ประยุกต์)** เปลี่ยนตัวอย่าง Tasks API ให้ handler `get_task` คืน error message เป็น**JSON**แทน
   plain text เมื่อไม่พบ task (ปัจจุบันคืน `(StatusCode::NOT_FOUND, String)` ที่ตอบเป็น
   `content-type: text/plain`) — ให้ตอบ `(StatusCode::NOT_FOUND, Json<ErrorBody>)` โดย
   `ErrorBody { error: String }` แทน แล้วทดสอบด้วย `curl -i` ว่า `content-type` เปลี่ยนเป็น
   `application/json` และ body เป็น `{"error":"ไม่พบ task id 999"}` — *hint*: ต้องเปลี่ยน return type ของ
   `Result<Json<Task>, (StatusCode, String)>` ทั้งก้อนเป็น
   `Result<Json<Task>, (StatusCode, Json<ErrorBody>)>` และเปลี่ยนบรรทัด `Err((StatusCode::NOT_FOUND, ...))`
   ให้ห่อ message ด้วย `Json(ErrorBody { error: ... })` แทน `String` ตรงๆ — นี่คือจุดเริ่มต้นของแนวคิดที่
   Part 66 จะขยายเป็นระบบ error handling ที่สมบูรณ์กว่านี้

## สรุป

บทนี้พาไปทำความรู้จัก **Axum** ในระดับที่เขียนโค้ดจริงได้แล้ว — เริ่มจากเข้าใจว่า Axum ไม่ใช่สิ่งวิเศษที่
ทำงานอย่างมายากล แต่เป็นชั้นบางๆที่ประกอบ `hyper` (HTTP protocol) กับ `tower`/`tower-http` (นามธรรม
`Service` + middleware) เข้าด้วยกัน โดยทั้งหมดวิ่งอยู่บน Tokio runtime ที่ Part 48-49 สอนไว้ตรงๆ ไม่มี
runtime หรือ networking layer ของตัวเอง — จากนั้นเขียนเซิร์ฟเวอร์ Hello World ตัวแรกด้วย `Router`,
`.route()`, `axum::serve()` และ `tokio::net::TcpListener` ตัวเดียวกับ Part 49 พิสูจน์ด้วย `curl` จริง

หัวใจสำคัญที่สุดของบทคือกลไกเบื้องหลัง `async fn handler() -> impl IntoResponse` — ผ่าน trait `Handler`
และ `IntoResponse` ที่ implement ให้กับ type จำนวนมาก (`&str`, `String`, `StatusCode`, tuple, `Json<T>`,
`Result<T, E>`) โดยใช้ blanket implementation แบบเดียวกับที่ Part 21-22 สอน — และได้เห็น compiler error
จริงที่เกิดขึ้นเมื่อ handler ผิด พร้อมวิธีอ่านมันด้วย `#[axum::debug_handler]` จากนั้นได้ฝึก routing
พื้นฐาน (`.route()` หลายตัว, `.nest()`), คืนค่า JSON ผ่าน `axum::Json<T>` ที่ผูกกับ `Serialize` trait จาก
Part 57-58 ตรงๆ, และควบคุม status code ผ่าน `StatusCode` และ tuple response

ตัวอย่างท้ายบท — Tasks API ที่ใช้ `Arc<Mutex<Vec<Task>>>` (Part 39) เป็น state — แสดงให้เห็นว่าเซิร์ฟเวอร์
ขนาดเล็กที่ทำงานได้จริงประกอบขึ้นจากส่วนต่างๆที่เรียนมาได้อย่างไร พร้อมทดสอบทุก endpoint ด้วย `curl -i`
จริงจนเห็นทั้ง header, status code, และ JSON body ครบถ้วน — แต่ก็ย้ำไว้ชัดเจนว่านี่เป็นแค่ **ขั้นบันไดแรก**
ของการจัดการ state: **Part 63** จะขยายเรื่อง routing และ handler ให้ลึกกว่านี้ (path parameter หลายตัว,
ทุก HTTP method, การจัดการ route ที่ชนกัน), **Part 64** จะสอนระบบ extractor และ state management แบบ
เต็มรูปแบบ, **Part 65** จะสอน middleware ผ่าน `tower`/`tower-http` ที่บทนี้แค่แนะนำให้รู้จักตำแหน่งของมัน
ใน stack, และ **Part 66** จะสอนการออกแบบ error handling แบบมืออาชีพที่ต่อยอดจาก `(StatusCode, String)`
อย่างง่ายที่บทนี้ใช้ไปสู่ custom error type ที่ implement `IntoResponse` เอง

---

**Part ก่อนหน้า:** [HTTP Fundamentals และ REST API Concepts](part-061-http-fundamentals.md) | **Part ถัดไป:** [Axum: Routing และ Handlers](part-063-axum-routing-handlers.md)
