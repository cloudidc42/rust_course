# Part 65: Axum: Middleware (tower, tower-http)

> โมดูล: การพัฒนาเว็บแอปพลิเคชัน (Web Development) | ระดับ: สูง | เวลาโดยประมาณ: 260 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้ว่าทำไม "cross-cutting concern" อย่าง logging, การจับเวลา request, การเติม CORS header, การบีบอัด
  response, การจำกัดขนาด body, หรือการเช็ค auth **ไม่ควร**ถูกเขียนซ้ำอยู่ในทุก handler — และรู้จักคำตอบของ
  ปัญหานี้ในโลก Rust: **`tower`** (ระบบนิเวศ middleware ที่เป็นภาษากลาง ไม่ผูกกับ web framework ตัวใดตัวหนึ่ง)
  และ **`tower-http`** (ชุด middleware สำเร็จรูปสำหรับงาน HTTP โดยเฉพาะ ที่ Axum ใช้เป็นมาตรฐาน)
- อธิบายกลไกของ **`tower::Service`** และ **`tower::Layer`** ได้ในระดับแนวคิดที่แม่นยำ — เข้าใจว่า `Router`
  ของ Axum เอง**คือ** `Service` ตัวหนึ่ง (พิสูจน์ได้จริงจาก source code ของ Axum), เข้าใจโมเดล "ชั้นหัวหอม"
  (onion model) ที่ request ไหลเข้าทีละชั้นและ response ไหลกลับออกทีละชั้นในลำดับย้อนกลับ และเชื่อมโยงกลไก
  `poll_ready`/`call` แบบ future-based เข้ากับความเข้าใจเรื่อง polling/executor จาก Part 46-47 ที่คุณมีอยู่แล้ว
- ใช้ middleware สำเร็จรูปจาก `tower-http` ได้จริงในโปรเจกต์: `TraceLayer` (ต่อยอดจาก `tracing` ใน Part 60),
  `CorsLayer` (ทำ CORS ให้ถูกต้องจริงตามที่ Part 61 พูดถึงไว้ พร้อมพิสูจน์พฤติกรรม preflight `OPTIONS` จริง),
  `CompressionLayer`, `TimeoutLayer`, และ `RequestBodyLimitLayer` — พร้อมตัวเลขและ log จริงจากการรันจริง
- เข้าใจ**ลำดับการ `.layer()`** อย่างลึกซึ้งพอที่จะทำนายพฤติกรรมได้ถูกต้อง 100% ไม่ใช่แค่จำสูตร — พิสูจน์ด้วยการ
  รันจริงว่าลำดับต่างกันทำให้ observable behavior ต่างกันจริง (ไม่ใช่แค่ทฤษฎี) รวมถึงเข้าใจความแตกต่างเฉพาะ
  ระหว่าง `.layer()` กับ `.route_layer()` ที่ทำให้ 404 กลายเป็น 401 ได้ (หรือไม่ได้) ขึ้นกับตัวที่เลือกใช้
- เขียน **custom middleware ของตัวเอง** ด้วย `axum::middleware::from_fn` และ `from_fn_with_state` ได้อย่าง
  ถูกต้อง ครอบคลุมทั้งกรณีจับเวลา + log (ต่อยอด Part 60), กรณี auth-gate ที่ short-circuit คำขอด้วย response
  ทันทีโดยไม่เรียก handler จริง, และกรณีแนบข้อมูลเข้า request ผ่าน `Extension` ให้ handler ปลายทางดึงไปใช้
  (ต่อยอด `Extension<T>` extractor จาก Part 64)
- ประกอบ middleware stack เต็มรูปแบบให้กับ API จริง (ต่อยอดจากระบบห้องสมุด/การจองที่ Part 63-64 วางโครงไว้)
  พร้อมทดสอบ end-to-end ด้วย `curl` จริง ยืนยันว่า header ที่ควรมี, log ที่ควรออก, และ response ที่ควรถูกปฏิเสธ
  เกิดขึ้นตรงตามที่ออกแบบไว้ทุกจุด

## ความรู้ที่ต้องมีมาก่อน

- **Part 46 (Async/Await เบื้องต้น)** และ **Part 47 (Futures และ Executors)**: `tower::Service` มีรูปร่างเป็น
  ฟังก์ชันที่คืน future แบบเดียวกับที่ Part 46-47 อธิบายไว้ (poll-based execution ที่ executor เป็นผู้เรียก
  `poll` ซ้ำ ๆ จนกว่าจะพร้อม) — บทนี้จะไม่อธิบายกลไก `Future`/`poll` ใหม่จากศูนย์ แต่จะชี้ให้เห็นว่า
  `tower::Service::call` คือการนำแนวคิดเดียวกันมาประยุกต์กับ "การจัดการ request หนึ่งตัว"
- **Part 48-50 (Tokio: Runtime, Tasks, I/O, Networking, Sync)**: middleware ที่จะเขียนในบทนี้ (เช่น
  timing middleware) เป็น async function ที่รันอยู่ใน Tokio runtime เหมือนกับ handler ทุกตัวที่คุณเขียนมา
  ตั้งแต่ Part 62-64
- **Part 60 (Logging และ Tracing เบื้องต้น)**: บทนี้อ้างอิง `tracing`/`tracing-subscriber` และแนวคิด
  `RUST_LOG` ตลอดทั้งบท โดยเฉพาะตอนอธิบาย `TraceLayer` ที่สร้าง log และ span ให้อัตโนมัติทุก request — ถ้า
  Part 60 (โดยเฉพาะเรื่อง log level และการกรองด้วย `RUST_LOG=module=level`) ยังไม่แน่น ให้กลับไปทวนก่อน
  เพราะกับดักที่พบบ่อยที่สุดข้อหนึ่งของบทนี้คือ "ตั้ง `RUST_LOG` แล้วทำไมไม่เห็น log จาก middleware เลย"
  ซึ่งใช้ความเข้าใจเรื่อง log level filtering จาก Part 60 ตรง ๆ ในการวินิจฉัย
- **Part 61 (HTTP Fundamentals และ REST API Concepts)**: บทนี้อ้างอิงแนวคิด CORS, HTTP status code
  (`401`, `404`, `408`, `413`), และ preflight request (`OPTIONS`) ที่ Part 61 ปูพื้นไว้ — บทนี้จะไม่อธิบาย
  ว่า CORS คืออะไรจากศูนย์ แต่จะพาไปดูว่า Axum/`tower-http` implement มันจริงอย่างไร
- **Part 62 (แนะนำ Axum Framework), Part 63 (Axum: Routing และ Handlers), Part 64 (Axum: State
  Management และ Extractors)**: บทนี้สมมติว่าคุณสร้าง `Router`, เขียน handler ที่รับ extractor (`Path`,
  `Query`, `Json`, `State<T>`, `Extension<T>`) ได้แล้ว และมี `AppState` ที่แชร์ผ่าน `State` extractor ตามที่
  Part 64 สอนไว้ — ตัวอย่าง capstone ท้ายบทนี้ต่อยอดจากระบบห้องสมุด/การจองที่ Part 63-64 เริ่มวางโครงไว้
  โดยตรง เพิ่ม middleware stack เต็มรูปแบบเข้าไป
- **Part 12 (Result และ Error Handling เบื้องต้น)** และ **Part 30-31 (Error Handling ระดับโปรเจกต์)**:
  หลักการ "แยกส่วนที่ทำหน้าที่ต่างกันออกจากกัน อย่าเขียนโค้ดซ้ำ" ที่ Part 30-31 ใช้ออกแบบ error type เป็น
  หลักการเดียวกันที่ทำให้ middleware มีเหตุผลในการมีอยู่ — บทนี้จะโยงกลับไปที่หลักการนี้ตรง ๆ ในหัวข้อแรก
  ส่วนกลไก error handling แบบละเอียดของ Axum (การแปลง `Result<T, E>` เป็น response ที่ถูกต้อง) จะเป็นเนื้อหา
  หลักของ **Part 66** ที่ตามมา — บทนี้จะแค่พอให้เข้าใจว่า middleware ที่ปฏิเสธคำขอได้ต้องคืน "response ที่ถูก
  ต้อง" ไม่ใช่ "error ที่ปล่อยลอย"

## เนื้อหา

### 65.1 ทำไมต้องมี Middleware: Cross-Cutting Concerns ที่ทุก API จริงต้องมี

ลองนึกภาพ API ระบบห้องสมุดที่ Part 63-64 เริ่มสร้างไว้ — มี handler สำหรับ `list_books`, `get_book`,
`borrow_book` และอีกหลายสิบ endpoint ที่จะเพิ่มขึ้นเรื่อย ๆ ตามที่ระบบเติบโต ทีนี้ลองนึกถึงความต้องการที่เกิด
ขึ้นจริงกับ API ทุกตัวที่ขึ้น production:

1. **อยาก log ทุก request** ที่เข้ามา — method, path, และตอนจบ อยาก log status code กับเวลาที่ใช้ไปด้วย
2. **อยากรู้ว่า request ไหนช้าผิดปกติ** — เพื่อ debug performance
3. **อยากให้ browser จากหน้าเว็บ frontend คนละ origin เรียก API นี้ได้** — ต้องเติม CORS header ที่ถูกต้อง
4. **อยากบีบอัด response ที่มีขนาดใหญ่** ด้วย gzip เพื่อประหยัด bandwidth
5. **อยาก timeout request ที่ค้างนานเกินไป** ไม่ให้ค้างตลอดกาลจนกิน connection ของ server ไปเรื่อย ๆ
6. **อยากจำกัดขนาด body ที่ client ส่งมา** ไม่ให้ client ส่ง payload ขนาดหลาย GB มาทำให้ server ล้ม
7. **อยากเช็คว่า request มี API key/token ที่ถูกต้องก่อนถึง handler จริง** สำหรับ endpoint ที่ต้องป้องกัน

ทั้งเจ็ดข้อนี้มีลักษณะร่วมกันที่สำคัญมาก: **มันไม่เกี่ยวอะไรเลยกับ "ตรรกะทางธุรกิจ" ของ endpoint นั้น ๆ**
`borrow_book` มีหน้าที่แค่ตรวจว่าหนังสือว่างอยู่หรือไม่แล้วบันทึกการยืม — มันไม่ควรต้องรู้เรื่อง CORS, ไม่ควร
ต้องรู้เรื่องการบีบอัด response, และไม่ควรต้องเขียนโค้ดจับเวลาตัวเองซ้ำกับทุก handler อื่นในระบบ

#### ถ้าไม่มี Middleware: เขียนซ้ำในทุก Handler

ลองดูว่าถ้าไม่มี middleware แล้วจะต้องทำอะไรบ้าง (โค้ดนี้จงใจเขียนแบบ "ผิดหลักการ" เพื่อให้เห็นปัญหาชัด ๆ):

```rust
use axum::{extract::State, response::IntoResponse, http::StatusCode};
use std::time::Instant;

async fn list_books_bad(State(state): State<AppState>) -> impl IntoResponse {
    let start = Instant::now(); // (1) ต้องจับเวลาเอง
    tracing::info!("เริ่ม request: list_books");

    let books = state.catalog.lock().unwrap().values().cloned().collect::<Vec<_>>();

    tracing::info!(elapsed_ms = start.elapsed().as_millis(), "จบ request: list_books"); // (2) log ซ้ำ
    axum::Json(books)
}

async fn borrow_book_bad(
    State(state): State<AppState>,
    axum::extract::Path(id): axum::extract::Path<u32>,
    headers: axum::http::HeaderMap,
) -> impl IntoResponse {
    let start = Instant::now(); // (1) จับเวลาซ้ำอีกครั้ง — copy-paste จาก handler บน
    tracing::info!("เริ่ม request: borrow_book");

    // (3) เช็ค auth ซ้ำ — ต้อง copy-paste โค้ดนี้ไปทุก handler ที่ต้อง auth
    let key = headers.get("x-api-key").and_then(|v| v.to_str().ok());
    if key != Some("secret123") {
        tracing::warn!("ปฏิเสธ: auth ไม่ผ่าน");
        return (StatusCode::UNAUTHORIZED, "unauthorized").into_response();
    }

    // ... ตรรกะจริงของการยืมหนังสือ (โดนกลบด้วยโค้ด infrastructure ทั้งหมดข้างบน) ...
    tracing::info!(elapsed_ms = start.elapsed().as_millis(), "จบ request: borrow_book");
    (StatusCode::OK, "borrowed").into_response()
}
# struct AppState;
```

ปัญหาไม่ได้อยู่ที่โค้ดนี้ "compile ไม่ได้" — มันคอมไพล์ได้สบาย ๆ ปัญหาคือ**หลักการ DRY (Don't Repeat
Yourself) ที่ Part 30-31 เน้นตอนออกแบบ error type ถูกละเมิดอย่างรุนแรง**: ทุก handler ใหม่ที่เพิ่มเข้ามาต้อง
copy-paste โค้ดจับเวลา, โค้ด log, โค้ดเช็ค auth ซ้ำทุกครั้ง — ถ้าวันหนึ่งอยากเปลี่ยนรูปแบบ log (เช่น เพิ่ม
`request_id` เข้าไปทุกบรรทัด) ต้องไปแก้ทุก handler ที่มีอยู่ทั้งหมดในระบบ และถ้าลืมแก้ handler ใดไปสักตัว
ระบบ log ก็จะไม่สอดคล้องกันแบบเงียบ ๆ โดยไม่มี compiler เตือนเลยแม้แต่นิดเดียว (เพราะทางไวยากรณ์มันถูกต้อง
สมบูรณ์แบบ)

#### คำตอบ: แยก "สิ่งที่ทำกับทุก request เหมือนกัน" ออกจาก "ตรรกะทางธุรกิจของ endpoint นั้น ๆ"

นี่คือหลักการเดียวกับที่ Part 30-31 ใช้ตอนออกแบบ error type ที่ดี — "แยกสิ่งที่เปลี่ยนบ่อยออกจากสิ่งที่ไม่ควร
เปลี่ยน" เพียงแต่คราวนี้แกนที่แยกคือ "โค้ดที่เฉพาะเจาะจงกับ endpoint" กับ "โค้ดที่ใช้ร่วมกันข้าม endpoint"
**Middleware คือกลไกที่ให้คุณเขียนโค้ดจับเวลา, log, เช็ค auth เพียง**ครั้งเดียว**แล้วให้มันครอบ (wrap) ทุก
handler ที่ต้องการโดยอัตโนมัติ** — handler เองไม่ต้องรู้เรื่องพวกนี้เลยแม้แต่นิดเดียว เขียนแค่ตรรกะทางธุรกิจ
บริสุทธิ์ ๆ

ในโลก Rust/Axum กลไกที่ทำให้เรื่องนี้เป็นไปได้อย่างเป็นระบบ ไม่ใช่แค่ "เทคนิคเฉพาะของ Axum" แต่เป็นระบบนิเวศ
ทั้งชุดที่ Axum เองก็สร้างอยู่บนมัน — คือ crate ชื่อ **`tower`** และชุด middleware สำเร็จรูปสำหรับ HTTP ที่
ชื่อ **`tower-http`** บทนี้จะพาไปดูทั้งกลไกเบื้องหลัง (`tower::Service`/`tower::Layer`) และวิธีใช้งานจริง

### 65.2 `tower::Service`: หน่วยพื้นฐานที่สุดของทุกอย่างใน Axum

ก่อนจะไปถึง middleware สำเร็จรูป ต้องเข้าใจ trait ที่เป็นรากฐานของทั้งระบบก่อน — **`tower::Service`** คือ
trait ที่ `tower` (และทั้ง `tower-http`, และ Axum เอง) สร้างขึ้นมาเพื่อนิยามคำว่า **"อะไรก็ตามที่รับ request
หนึ่งตัวแล้วคืน response หนึ่งตัวแบบ async"** อย่างเป็นทางการ ไม่ว่าสิ่งนั้นจะเป็น handler ของคุณ, HTTP client,
gRPC client, หรือ middleware ตัวหนึ่งก็ตาม — ทุกอย่างที่ "รับ request แล้วคืน response" ใน ecosystem นี้คือ
`Service` ทั้งหมด

รูปร่างของ trait (ตัดรายละเอียดที่ไม่จำเป็นออกเพื่อให้เห็นแกนหลัก) หน้าตาประมาณนี้:

```rust
trait Service<Request> {
    type Response;
    type Error;
    type Future: std::future::Future<Output = Result<Self::Response, Self::Error>>;

    fn poll_ready(
        &mut self,
        cx: &mut std::task::Context<'_>,
    ) -> std::task::Poll<Result<(), Self::Error>>;

    fn call(&mut self, req: Request) -> Self::Future;
}
```

สังเกตสองเมธอดหลัก:

- **`call(&mut self, req: Request) -> Self::Future`** — รับ request หนึ่งตัว แล้วคืน**future** ที่เมื่อ
  await แล้วจะได้ `Result<Response, Error>` — นี่คือรูปร่างเดียวกับ `async fn handler(req: Request) ->
  Result<Response, Error>` ที่คุณคุ้นเคยมาตั้งแต่ Part 46 เพียงแต่เขียนในรูปแบบ trait method แทน `async fn`
  ตรง ๆ (เหตุผลที่ต้องเขียนแบบนี้แทน `async fn` ธรรมดา คือ Rust ยังไม่รองรับ `async fn` ใน trait ได้อย่าง
  สมบูรณ์ในทุกสถานการณ์ตอนที่ `tower` ถูกออกแบบ — จึงต้องเขียน `Future` เป็น associated type แยกออกมาเอง)
- **`poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Error>>`** — นี่คือจุดที่เชื่อมกับความ
  เข้าใจเรื่อง **poll-based execution จาก Part 47** ตรง ๆ: `poll_ready` ให้ service ตอบว่า **"ตอนนี้ฉันพร้อม
  รับ request ใหม่หรือยัง"** ก่อนที่ caller จะเรียก `call` จริง — สำหรับ service ธรรมดาส่วนใหญ่ (เช่น handler
  ของคุณ) คำตอบคือ "พร้อมเสมอ" (`Poll::Ready(Ok(()))`) แต่สำหรับ service ที่มี**backpressure** จริง ๆ (เช่น
  connection pool ที่มี connection จำกัด, หรือ rate limiter) `poll_ready` คือจุดที่ทำให้ caller รอได้ก่อนที่
  จะยิง request ไปโดยไม่ต้องเสี่ยงถูกปฏิเสธทีหลัง — กลไกนี้คือเหตุผลเดียวกับที่ Part 47 อธิบายว่าทำไม
  `Future` ต้องถูก `poll` ซ้ำ ๆ จนกว่าจะพร้อม แทนที่จะบล็อกรอตรง ๆ

**สิ่งที่สำคัญที่สุดที่ต้องเข้าใจ:** ทั้ง `Router` ของ Axum เอง, ทั้ง handler ทุกตัวที่คุณเขียน, ทั้ง
middleware ทุกตัวที่จะเรียนในบทนี้ — **ทุกอย่างคือ `Service` ทั้งหมด** เราสามารถพิสูจน์ข้อความนี้ได้จริงจาก
source code ของ Axum เวอร์ชัน 0.8.9 (ไม่ใช่การเดา) — `Router<()>` ถูก implement ตรงตัวว่า:

```rust
// นี่คือ implementation จริงจาก axum-0.8.9 (ตัดส่วน bound บางส่วนออกเพื่อความกระชับ)
impl<B> tower::Service<axum::http::Request<B>> for axum::Router<()> {
    type Response = axum::response::Response;
    type Error = std::convert::Infallible; // Router รับประกันว่าไม่ error เลย — คืน response เสมอ
    type Future = /* ... */;

    fn poll_ready(&mut self, _: &mut std::task::Context<'_>) -> std::task::Poll<Result<(), Self::Error>> {
        std::task::Poll::Ready(Ok(()))
    }

    fn call(&mut self, req: axum::http::Request<B>) -> Self::Future {
        // ค้นหา route ที่ตรงกับ path/method แล้วเรียก handler ที่ผูกไว้
        // ...
    }
}
```

สังเกตว่า `type Error = std::convert::Infallible` — แปลว่า **`Router` การันตีว่าจะไม่มีทาง "error" ออกมา
เลย มันจะคืน `Response` เสมอ** (ถ้า path ไม่ตรงกับ route ไหนเลย มันก็แค่คืน response ที่เป็น `404 Not Found`
— นั่นก็ยังเป็น "response ที่สำเร็จ" ในมุมของ `Service`, ไม่ใช่ "error") ข้อเท็จจริงนี้จะสำคัญมากในหัวข้อ 65.9
ตอนพูดถึง error handling ของ middleware — เพราะ layer ไหนที่คุณจะ `.layer()` เข้า `Router` **ต้อง**คงเงื่อนไข
`Error: Into<Infallible>` นี้ไว้ ไม่งั้น compiler จะปฏิเสธตั้งแต่ตอน compile เลย

ส่วน**handler** (ฟังก์ชัน async ที่คุณเขียนแล้วส่งให้ `get()`/`post()`/ฯลฯ) เองไม่ได้ implement
`tower::Service` ตรง ๆ (มันอยู่ภายใต้ trait ของ Axum เองชื่อ `Handler` ซึ่งมีรูปร่างคล้ายกันแต่ปรับให้ทำงาน
กับ extractor ได้) — แต่ Axum มีฟังก์ชัน `.into_service()` ที่แปลง `Handler` ให้กลายเป็น `Service` ได้จริง
(ผ่าน type ชื่อ `HandlerService` ภายใน) เพื่อให้มันเข้ากับระบบ `tower` ได้อย่างสมบูรณ์ — นี่คือที่มาของคำว่า
**"handler กลายเป็น Service"**: มันไม่ใช่ Service ในตัวเองเป๊ะ ๆ แต่มี adapter บาง ๆ แปลงให้เป็น Service ได้
เสมอ ซึ่งหมายความว่า**middleware ที่ทำงานกับ `Service` ใด ๆ ก็ทำงานกับ handler ทุกตัวได้โดยอัตโนมัติ** — นี่
คือสิ่งที่ทำให้ `.layer()` ใช้ได้กับทั้ง `Router` ทั้งก้อน และใช้ได้กับ handler ตัวเดียวผ่าน `.layer()` ของ
`MethodRouter`/`Handler` ก็ได้เช่นกัน

### 65.3 `tower::Layer`: กลไกการห่อ Service ให้กลายเป็น Service ตัวใหม่

ถ้า `Service` คือ "หน่วยที่รับ request แล้วคืน response" แล้ว **`tower::Layer`** คือ "โรงงานที่รับ `Service`
ตัวหนึ่งแล้วคืน `Service` ตัวใหม่ที่ทำงานมากขึ้น" — นี่คือกลไกการประกอบ middleware ทั้งหมดของ `tower`:

```rust
trait Layer<S> {
    type Service;

    fn layer(&self, inner: S) -> Self::Service;
}
```

รับ `inner: S` (service เดิม) แล้วคืน `Self::Service` (service ใหม่ที่ "ห่อ" ตัวเดิมไว้) — สังเกตว่า
`Self::Service` เองก็ต้องเป็น `Service` ด้วย (ปกติแล้วจะ implement `Service` ให้มันเองในอีก `impl` บล็อก
หนึ่ง) พูดง่าย ๆ คือ: **`Layer` ไม่ได้ "เพิ่มพฤติกรรม" เข้าไปใน service เดิมโดยตรง แต่สร้าง service ตัวใหม่ที่
มี service เดิมซ่อนอยู่ข้างในแล้วเรียกใช้มันตอนที่เหมาะสม**

#### พิสูจน์กลไกนี้ด้วยโค้ดที่ compile และรันได้จริง

มาเขียน `Layer`/`Service` เองแบบเปลือย ๆ ไม่พึ่ง HTTP หรือ Axum เลย เพื่อเห็นกลไกล้วน ๆ:

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use tower::{Layer, Service, ServiceBuilder, ServiceExt};

#[derive(Debug, Clone)]
struct DemoRequest {
    path: String,
}

#[derive(Debug)]
struct DemoResponse {
    status: u16,
}

// Service ตัวใน (inner service) — "handler" จริงที่ตอบกลับ 200 เสมอ
#[derive(Clone)]
struct EchoService;

impl Service<DemoRequest> for EchoService {
    type Response = DemoResponse;
    type Error = std::convert::Infallible;
    type Future = Pin<Box<dyn Future<Output = Result<Self::Response, Self::Error>> + Send>>;

    fn poll_ready(&mut self, _cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        Poll::Ready(Ok(()))
    }

    fn call(&mut self, req: DemoRequest) -> Self::Future {
        Box::pin(async move {
            println!("  [EchoService] จัดการ path={}", req.path);
            Ok(DemoResponse { status: 200 })
        })
    }
}

// Middleware Service ที่ log ก่อน/หลังเรียก inner service — นี่คือ "ชั้นหัวหอม" หนึ่งชั้น
#[derive(Clone)]
struct LoggingService<S> {
    inner: S,
    label: &'static str,
}

impl<S> Service<DemoRequest> for LoggingService<S>
where
    S: Service<DemoRequest, Response = DemoResponse, Error = std::convert::Infallible>
        + Send
        + 'static,
    S::Future: Send,
{
    type Response = DemoResponse;
    type Error = std::convert::Infallible;
    type Future = Pin<Box<dyn Future<Output = Result<Self::Response, Self::Error>> + Send>>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        self.inner.poll_ready(cx)
    }

    fn call(&mut self, req: DemoRequest) -> Self::Future {
        let label = self.label;
        println!("[{label}] ก่อนเรียก inner (path={})", req.path);
        let fut = self.inner.call(req);
        Box::pin(async move {
            let res = fut.await;
            println!("[{label}] หลังเรียก inner (status={:?})", res.as_ref().map(|r| r.status));
            res
        })
    }
}

// Layer คือ "โรงงานผลิต Service ตัวใหม่จาก Service เดิม" — กลไก middleware composition ทั้งหมดของ tower
#[derive(Clone)]
struct LoggingLayer {
    label: &'static str,
}

impl<S> Layer<S> for LoggingLayer {
    type Service = LoggingService<S>;

    fn layer(&self, inner: S) -> Self::Service {
        LoggingService { inner, label: self.label }
    }
}

#[tokio::main]
async fn main() {
    let mut svc = ServiceBuilder::new()
        .layer(LoggingLayer { label: "ADDED-FIRST" })
        .layer(LoggingLayer { label: "ADDED-SECOND" })
        .service(EchoService);

    println!("=== เรียก request 1 ===");
    let resp = svc
        .ready()
        .await
        .unwrap()
        .call(DemoRequest { path: "/books".to_string() })
        .await
        .unwrap();
    println!("ผลลัพธ์สุดท้าย: {resp:?}");
}
```

รันจริงได้ output (คัดลอกตรงจากการรันจริงในสภาพแวดล้อมทดสอบ):

```
=== เรียก request 1 ===
[ADDED-FIRST] ก่อนเรียก inner (path=/books)
[ADDED-SECOND] ก่อนเรียก inner (path=/books)
  [EchoService] จัดการ path=/books
[ADDED-SECOND] หลังเรียก inner (status=Ok(200))
[ADDED-FIRST] หลังเรียก inner (status=Ok(200))
ผลลัพธ์สุดท้าย: DemoResponse { status: 200 }
```

นี่คือ **"onion model" (โมเดลชั้นหัวหอม)** ที่เป็นภาพจำสำคัญที่สุดของบทนี้ทั้งบท วาดเป็นภาพในหัวได้ดังนี้:

```
                     ┌─────────────────────────────────────┐
                     │           ADDED-FIRST                │  ← ชั้นนอกสุด
                     │   ┌───────────────────────────────┐   │
request ────────────▶│   │        ADDED-SECOND           │   │────────────▶ EchoService
                     │   │   ┌───────────────────────┐   │   │
                     │   │   │      EchoService      │   │   │
                     │   │   └───────────────────────┘   │   │
response ◀───────────│   │                               │   │◀────────────
                     │   └───────────────────────────────┘   │
                     └─────────────────────────────────────┘
```

Request **ไหลเข้า**จากชั้นนอกสุดเข้าไปทีละชั้น (`ADDED-FIRST` เห็นก่อน แล้ว `ADDED-SECOND` แล้วค่อยถึง
`EchoService` จริง ๆ) และ response **ไหลกลับออก**ในลำดับ**ย้อนกลับ**เป๊ะ ๆ (`EchoService` ตอบก่อน,
`ADDED-SECOND` ประมวลผลก่อน, แล้ว `ADDED-FIRST` เป็นคนสุดท้ายที่เห็น response ก่อนส่งกลับไปยังผู้เรียก) —
สังเกตว่า `ServiceBuilder::new().layer(A).layer(B).service(svc)` ให้ผลว่า **`A` (ที่เพิ่มก่อน) กลายเป็นชั้น
นอกสุด** เอกสารทางการของ `tower::builder::ServiceBuilder` เขียนกฎนี้ไว้ตรง ๆ ว่า:

> "The order in which layers are added impacts how requests are handled. Layers that are added first will
> be called with the request first."

จำกฎนี้ไว้ให้แม่น เพราะหัวข้อ 65.6 จะพิสูจน์ให้เห็นว่ากฎนี้ใช้ได้กับ `ServiceBuilder` เท่านั้น — ถ้าคุณเรียก
`.layer()` **ตรง ๆ ซ้อนกันหลายครั้งบน `Router`** (ไม่ผ่าน `ServiceBuilder`) กฎจะ**กลับด้าน**! นี่คือกับดักที่
พบบ่อยที่สุดข้อหนึ่งของทั้งบทนี้ และเราจะพิสูจน์มันด้วยการรันจริง ไม่ใช่แค่พูดลอย ๆ

### 65.4 Middleware ที่ปฏิเสธ Request กลางทาง (Short-Circuit)

สิ่งสำคัญอีกอย่างที่ต้องเห็นจากตัวอย่างข้างบนคือ middleware **ไม่จำเป็นต้องเรียก inner service เสมอไป** —
`LoggingService::call` ในตัวอย่างข้างบนเรียก `self.inner.call(req)` เสมอ แต่ในความเป็นจริง middleware
สามารถ**ตรวจสอบ request ก่อน**แล้ว**เลือกที่จะไม่เรียก inner เลย** ถ้าเงื่อนไขไม่ผ่าน — แทนที่จะเรียก inner
มันแค่สร้าง response ของตัวเอง (เช่น `401 Unauthorized`) แล้วคืนออกไปตรง ๆ นี่คือกลไกที่ทำให้ middleware
ทำหน้าที่เป็น **"ยามหน้าประตู" (gatekeeper)** ได้ — เรียกว่า **short-circuit** และคือกลไกหลักที่ auth
middleware ในหัวข้อ 65.9 ใช้

### 65.5 `tower-http`: ชุด Middleware สำเร็จรูปสำหรับงาน HTTP

การเขียน `Service`/`Layer` เองจากศูนย์แบบหัวข้อ 65.3 มีประโยชน์มากในการเข้าใจกลไก แต่ในงานจริงแทบไม่มีใคร
เขียน `Layer` เองสำหรับงานพื้นฐานอย่าง logging, CORS, compression, timeout — เพราะทีม **`tower-http`**
(ทีมเดียวกับที่ดูแล `tower` และ `axum`) เขียนตัวสำเร็จรูปให้แล้ว ผ่านการทดสอบมาอย่างละเอียด และ Axum ทั้ง
framework ก็ใช้ตัวเดียวกันนี้เป็นมาตรฐานแนะนำในเอกสารของตัวเอง

เพิ่มเข้าโปรเจกต์ด้วย (ต้องเลือก feature ที่ต้องใช้ — `tower-http` แยก feature ต่อ middleware เพื่อไม่ให้
ต้อง compile โค้ดที่ไม่ได้ใช้):

```bash
cargo add tower-http --features trace,cors,timeout,compression-gzip,limit
```

ได้ใน `Cargo.toml`:

```toml
[dependencies]
axum = "0.8.9"
tokio = { version = "1.53.1", features = ["full"] }
tower = { version = "0.5.3", features = ["util"] }
tower-http = { version = "0.7.1", features = ["trace", "cors", "timeout", "compression-gzip", "limit"] }
tracing = "0.1.44"
tracing-subscriber = { version = "0.3.23", features = ["env-filter"] }
serde = { version = "1.0.229", features = ["derive"] }
serde_json = "1.0.151"
```

มาดูตัวหลัก ๆ ที่ใช้บ่อยที่สุดในงานจริงทีละตัว พร้อมพิสูจน์พฤติกรรมด้วยการรันเซิร์ฟเวอร์จริงแล้วยิง `curl`
จริง (server ตัวอย่างนี้มีสี่ endpoint: `/health`, `/slow` ที่ sleep 3 วินาที, `/big-data` ที่คืน JSON ขนาด
ใหญ่, และ `/echo` ที่รับ body แล้วสะท้อนความยาวกลับ):

```rust
use axum::{
    extract::DefaultBodyLimit,
    http::{HeaderValue, Method, StatusCode},
    response::IntoResponse,
    routing::{get, post},
    Json, Router,
};
use serde::Serialize;
use std::time::Duration;
use tower_http::{
    compression::CompressionLayer, cors::CorsLayer, limit::RequestBodyLimitLayer,
    timeout::TimeoutLayer, trace::TraceLayer,
};

#[derive(Serialize)]
struct Health {
    status: &'static str,
}

async fn health() -> impl IntoResponse {
    Json(Health { status: "ok" })
}

// endpoint ที่จงใจช้าเกิน timeout ที่ตั้งไว้ (1 วินาที) เพื่อพิสูจน์ TimeoutLayer
async fn slow() -> impl IntoResponse {
    tokio::time::sleep(Duration::from_secs(3)).await;
    "ตอบช้ามาก ๆ"
}

// endpoint ที่คืนข้อมูลขนาดใหญ่พอให้ CompressionLayer เห็นผลจริง (gzip-able ได้ดี)
async fn big_data() -> impl IntoResponse {
    let items: Vec<String> = (0..500)
        .map(|i| format!("รายการหนังสือหมายเลข {i} - เรื่อง Rust Programming เล่มที่ {i}"))
        .collect();
    Json(items)
}

async fn echo_body(body: String) -> impl IntoResponse {
    format!("ได้รับ body ยาว {} bytes", body.len())
}

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt()
        .with_env_filter(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                tracing_subscriber::EnvFilter::new("tower_http_demo=info,tower_http=info")
            }),
        )
        .init();

    let cors = CorsLayer::new()
        .allow_origin("http://localhost:5173".parse::<HeaderValue>().unwrap())
        .allow_methods([Method::GET, Method::POST])
        .allow_headers([axum::http::header::CONTENT_TYPE, "x-api-key".parse().unwrap()]);

    let app = Router::new()
        .route("/health", get(health))
        .route("/slow", get(slow))
        .route("/big-data", get(big_data))
        .route("/echo", post(echo_body))
        .layer(DefaultBodyLimit::disable()) // ปิดลิมิตเดิมของ axum ก่อน ให้ RequestBodyLimitLayer เป็นคนคุมแทน
        .layer(RequestBodyLimitLayer::new(1024)) // จำกัด body ไม่เกิน 1024 bytes
        .layer(TimeoutLayer::with_status_code(
            StatusCode::REQUEST_TIMEOUT,
            Duration::from_secs(1),
        ))
        .layer(CompressionLayer::new())
        .layer(cors)
        .layer(TraceLayer::new_for_http());

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3002").await.unwrap();
    axum::serve(listener, app.into_make_service()).await.unwrap();
}
```

โค้ดนี้ compile ผ่านสะอาดจริงด้วย `cargo build` (เวอร์ชันจริงที่ทดสอบ: axum 0.8.9, tower-http 0.7.1)

#### `TraceLayer`: ต่อยอด `tracing` จาก Part 60 โดยตรง

`TraceLayer::new_for_http()` คือตัวที่ทำสิ่งที่ Part 60 สอนให้ทำด้วยมือ (สร้าง span, log เริ่ม/จบ request)
**ให้อัตโนมัติทุก request โดยไม่ต้องเขียนโค้ดเพิ่มสักบรรทัด** — แต่มีจุดที่ต้องระวังตรงกับกับดักคลาสสิกของ
Part 60 พอดี: **ค่าเริ่มต้นของระดับ log ที่ `TraceLayer` ใช้คือ `DEBUG` ไม่ใช่ `INFO`** ถ้าตั้ง
`RUST_LOG=info` เฉย ๆ (แม้จะรวม `tower_http=info` ด้วย) คุณจะ**ไม่เห็น log จาก `TraceLayer` เลยแม้แต่บรรทัด
เดียว** ต้องตั้งเป็น `tower_http=debug` ถึงจะเห็น — พิสูจน์ด้วยการรันจริง:

```bash
RUST_LOG="tower_http_demo=info,tower_http=debug" cargo run --bin tower_http_demo
```

แล้วยิง `curl` สามครั้ง (`/health`, `/big-data`, และ path ที่ไม่มีอยู่จริง) ได้ log จริงดังนี้:

```
2026-09-26T23:38:06.027672Z  INFO tower_http_demo: ฟังอยู่ที่ http://127.0.0.1:3002
2026-09-26T23:38:07.034879Z DEBUG request{method=GET uri=/health version=HTTP/1.1}: tower_http::trace::on_request: started processing request
2026-09-26T23:38:07.035103Z DEBUG request{method=GET uri=/health version=HTTP/1.1}: tower_http::trace::on_response: finished processing request latency=0 ms status=200
2026-09-26T23:38:07.035165Z DEBUG request{method=GET uri=/health version=HTTP/1.1}: tower_http::trace::on_eos: end of stream stream_duration=0 ms
2026-09-26T23:38:07.042309Z DEBUG request{method=GET uri=/big-data version=HTTP/1.1}: tower_http::trace::on_request: started processing request
2026-09-26T23:38:07.044739Z DEBUG request{method=GET uri=/big-data version=HTTP/1.1}: tower_http::trace::on_response: finished processing request latency=2 ms status=200
2026-09-26T23:38:07.046769Z DEBUG request{method=GET uri=/big-data version=HTTP/1.1}: tower_http::trace::on_eos: end of stream stream_duration=1 ms
2026-09-26T23:38:07.054628Z DEBUG request{method=GET uri=/does-not-exist version=HTTP/1.1}: tower_http::trace::on_request: started processing request
2026-09-26T23:38:07.054929Z DEBUG request{method=GET uri=/does-not-exist version=HTTP/1.1}: tower_http::trace::on_response: finished processing request latency=0 ms status=404
```

สังเกตสามอย่างที่ตรงกับความเข้าใจที่คุณมีอยู่แล้วจาก Part 60:

1. แต่ละ request ได้ **span** ของตัวเอง (`request{method=GET uri=/health ...}`) ที่ครอบทุก event ของ
   request นั้น — นี่คือกลไก span เดียวกับที่ `#[tracing::instrument]` สร้างให้ ใช้แก้ปัญหาเดียวกันคือ
   "log จากหลาย request ที่ทำงานสลับกันต้องแยกแยะที่มาได้" ตามที่ Part 60 อธิบายไว้ — เพียงแต่คราวนี้
   `TraceLayer` สร้าง span นี้ให้อัตโนมัติทุก request โดยไม่ต้องเขียน `#[instrument]` เอง
2. path ที่**ไม่ตรงกับ route ไหนเลย** (`/does-not-exist`, ได้ `404`) **ก็ยังถูก trace เหมือนกัน** — เพราะ
   `TraceLayer` ถูก apply ด้วย `.layer()` (ไม่ใช่ `.route_layer()`) ซึ่งครอบทั้ง fallback ของ `Router` ด้วย
   (จะอธิบายความแตกต่างนี้ละเอียดในหัวข้อ 65.6)
3. `latency=2 ms` สำหรับ `/big-data` (endpoint ที่สร้าง JSON 500 รายการ) เทียบกับ `latency=0 ms` สำหรับ
   `/health` — นี่คือตัวเลขจริงจากการรันจริง ไม่ใช่ตัวเลขสมมติ

#### `CorsLayer`: CORS ที่ถูกต้องจริง พร้อมพิสูจน์ Preflight `OPTIONS`

Part 61 พูดถึง CORS ไว้ในแง่แนวคิด — บทนี้มาดูว่า Axum implement มันยังไง เริ่มจากทดสอบ request ปกติที่มี
`Origin` header ตรงกับที่อนุญาต:

```bash
curl -sS -i http://127.0.0.1:3002/health -H "Origin: http://localhost:5173"
```

ผลลัพธ์จริง:

```
HTTP/1.1 200 OK
content-type: application/json
access-control-allow-origin: http://localhost:5173
content-length: 15
date: Sat, 26 Sep 2026 23:34:51 GMT

{"status":"ok"}
```

เห็น header `access-control-allow-origin: http://localhost:5173` ถูกเติมเข้ามาให้อัตโนมัติ — นี่คือสิ่งที่
`CorsLayer` ทำ: **เติม response header ที่บอก browser ว่า origin ไหนได้รับอนุญาตให้อ่าน response นี้**

จุดที่สำคัญมากและมักเข้าใจผิดคือ **ใครเป็นคนบล็อกจริง ๆ** — ลองยิง request จาก origin ที่**ไม่ได้**อยู่ใน
รายการอนุญาต:

```bash
curl -sS -i http://127.0.0.1:3002/health -H "Origin: http://evil.example.com"
```

ผลลัพธ์จริง (สังเกตให้ดี):

```
HTTP/1.1 200 OK
content-type: application/json
access-control-allow-origin: http://localhost:5173
content-length: 15
date: Sat, 26 Sep 2026 23:34:51 GMT

{"status":"ok"}
```

**Server ยังคืน `200 OK` พร้อม body เหมือนเดิมทุกอย่าง!** — server **ไม่ได้ปฏิเสธ** request นี้เลยที่ระดับ
HTTP มันแค่เติม header `access-control-allow-origin: http://localhost:5173` ไปแบบตายตัว (เพราะเราตั้งค่า
`allow_origin` เป็นค่าคงที่ค่าเดียว ไม่ใช่การ "เทียบแล้วเลือก") — เหตุผลที่ CORS ยังป้องกันอะไรได้เลยคือ
**บราวเซอร์ (ไม่ใช่ server) เป็นคนตรวจสอบ**: ถ้าหน้าเว็บที่รันอยู่บน origin `http://evil.example.com` ยิง
`fetch()` มาที่ endpoint นี้ บราวเซอร์จะเห็นว่า `access-control-allow-origin` ที่ได้กลับมาคือ
`http://localhost:5173` ซึ่ง**ไม่ตรง**กับ origin ของหน้าเว็บที่กำลังรันอยู่ (`evil.example.com`) — บราวเซอร์
จึง**บล็อกไม่ให้ JavaScript เข้าถึง response นั้น** (แม้ว่า response จะถูกส่งมาถึงเครื่อง client จริง ๆ แล้ว
ก็ตาม) — `curl` ไม่ใช่บราวเซอร์ จึงไม่มีการบล็อกอะไรให้เห็น นี่คือเหตุผลที่การทดสอบ CORS ด้วย `curl` เพียง
อย่างเดียว**ไม่พอ**ที่จะยืนยันว่า CORS ทำงานถูกต้อง — ต้องทดสอบด้วย browser จริง หรือเข้าใจกลไกนี้ให้แม่น
ก่อนสรุปผล

เทียบกับตั้ง `allow_origin` เป็น `tower_http::cors::Any` (อนุญาตทุก origin) — ผลลัพธ์จริงต่างกันชัดเจนตรง
ค่า header:

```
HTTP/1.1 200 OK
content-type: application/json
access-control-allow-origin: *
content-length: 15
```

`access-control-allow-origin: *` คือสัญลักษณ์บอกบราวเซอร์ว่า **"origin ไหนก็อ่าน response นี้ได้ทั้งหมด"**
— สะดวกตอน develop แต่ในงาน production จริงที่มี endpoint ที่ต้องพก credential (เช่น cookie) `*` ใช้ไม่ได้
ตามสเปกของ CORS เอง (บราวเซอร์จะปฏิเสธ `*` คู่กับ `credentials: include` โดยอัตโนมัติ) — ควรระบุ origin ที่
อนุญาตเจาะจงเสมอสำหรับ production

##### Preflight `OPTIONS`: พิสูจน์ด้วยการรันจริง

Browser จะส่ง **preflight request** แบบ `OPTIONS` ไปก่อน request จริงเสมอ เมื่อ request ที่จะส่งไม่ใช่
"simple request" ตามสเปก CORS (เช่น มี custom header อย่าง `x-api-key`, หรือ method ที่ไม่ใช่ `GET`/`POST`
แบบพื้นฐาน) — พิสูจน์ด้วยการยิง `OPTIONS` ตรง ๆ ด้วย `curl -X OPTIONS`:

```bash
curl -sS -i -X OPTIONS http://127.0.0.1:3002/health \
  -H "Origin: http://localhost:5173" \
  -H "Access-Control-Request-Method: GET" \
  -H "Access-Control-Request-Headers: content-type"
```

ผลลัพธ์จริง:

```
HTTP/1.1 200 OK
access-control-allow-methods: GET,POST
access-control-allow-headers: content-type,x-api-key
access-control-allow-origin: http://localhost:5173
allow: GET,HEAD
content-length: 0
date: Sat, 26 Sep 2026 23:34:51 GMT
```

สังเกตสิ่งสำคัญ: **request `OPTIONS` นี้ไม่มี body และได้ response กลับมาโดยที่ handler `health()` ไม่ถูก
เรียกเลย** (ไม่มี log อะไรเกี่ยวกับ handler ปรากฏขึ้น) — `CorsLayer` **สกัดคำขอ preflight ไว้ตั้งแต่ชั้น
middleware** แล้วตอบกลับด้วย header ที่บอกว่า method/header ไหนได้รับอนุญาตบ้าง โดยไม่ปล่อยให้คำขอไหลลึกเข้า
ไปถึง route/handler จริงเลย — นี่คือพฤติกรรมที่ถูกต้องตามสเปก CORS: browser จะดู response ของ preflight นี้
ก่อน ถ้าเงื่อนไขผ่านหมด (origin ตรง, method ที่จะใช้อยู่ใน `access-control-allow-methods`, header ที่จะส่ง
อยู่ใน `access-control-allow-headers`) จึงจะยิง request จริงตามมา

#### `TimeoutLayer`: จำกัดเวลาที่ Request ค้างได้นานสุด

ในเวอร์ชันปัจจุบันของ `tower-http` (0.7.1) `TimeoutLayer::new(duration)` ถูก**เลิกใช้แล้ว** (deprecated)
เปลี่ยนเป็น `TimeoutLayer::with_status_code(status_code, duration)` แทน — ตอน compile โค้ดที่ใช้ `::new()`
จะได้ warning จริงแบบนี้:

```
warning: use of deprecated associated function `tower_http::timeout::TimeoutLayer::new`: Use `TimeoutLayer::with_status_code` instead
  --> src/bin/tower_http_demo.rs:62:30
   |
62 |         .layer(TimeoutLayer::new(Duration::from_secs(1)))
   |                              ^^^
```

จุดที่น่าสนใจกว่า warning คือ**เหตุผลที่มันถูกออกแบบใหม่**: `tower::timeout::Timeout` (ตัวดั้งเดิมใน
`tower` เอง ไม่ใช่ `tower-http`) ทำงานโดยเปลี่ยน**type ของ error** เป็น `BoxError` เมื่อ timeout เกิดขึ้น —
ซึ่งขัดกับกฎที่ Router ต้องการ (`Error: Into<Infallible>`, ตามที่อธิบายไว้ในหัวข้อ 65.2) ทำให้ต้องใช้
`HandleErrorLayer` แปลง error เป็น response เอง (รายละเอียดเต็มอยู่ในหัวข้อ 65.9 และ Part 66) — ส่วน
`tower_http::timeout::TimeoutLayer` ถูกออกแบบมาให้ **ไม่เปลี่ยน error type เลย** — เมื่อ timeout มันสร้าง
`Response` ที่มี status code ตามที่กำหนด (ค่าเริ่มต้นของ `::new()` ก่อนถูกเลิกใช้คือ `408 Request Timeout`)
คืนออกไปตรง ๆ เหมือนเป็น response ปกติ ทำให้ใช้กับ `Router::layer()` ได้โดยไม่ต้องพ่วง `HandleErrorLayer`
เลย — นี่คือตัวอย่างที่ดีว่าทำไมควรใช้ตัวสำเร็จรูปจาก `tower-http` แทนตัวดิบจาก `tower` เมื่อทำงานกับ Axum

พิสูจน์ด้วยการรันจริง ยิงไปที่ `/slow` (sleep 3 วินาที) ที่มี `TimeoutLayer` ตั้งไว้ที่ 1 วินาที พร้อมจับเวลา
ด้วย `time`:

```bash
time curl -sS -i http://127.0.0.1:3002/slow
```

ผลลัพธ์จริง:

```
HTTP/1.1 408 Request Timeout
access-control-allow-origin: http://localhost:5173
content-length: 0
date: Sat, 26 Sep 2026 23:35:46 GMT

real	0m1.008s
user	0m0.004s
sys	0m0.003s
```

request จบลงใน **1.008 วินาที** (ตรงกับ timeout 1 วินาทีที่ตั้งไว้) แม้ว่า handler `slow()` เขียนไว้ว่าจะ
sleep เต็ม 3 วินาที — `TimeoutLayer` ตัดคำขอทิ้งกลางทาง**โดยที่ handler ยังทำงานต่อไปเบื้องหลังจนครบ 3 วินาที
เหมือนเดิม** (แค่ response ที่ client ได้รับถูกส่งไปก่อนตาม timeout) นี่เป็นจุดที่ต้องเข้าใจให้ถูก: `Timeout`
middleware ไม่ได้ "ยกเลิก" งานที่ handler ทำอยู่จริง ๆ (cancel การทำงาน) มันแค่**ไม่รอผลลัพธ์จริงอีกต่อไป**
และตอบ client ไปก่อนเท่านั้น — ถ้าต้องการยกเลิกงานจริง ๆ (เช่น หยุด database query ที่กำลังรันอยู่) ต้องใช้
กลไกอื่นเพิ่ม เช่น `tokio::select!` ร่วมกับ cancellation token ภายใน handler เอง ซึ่งเป็นรายละเอียดที่ลึกกว่า
สโคปของบทนี้

#### `CompressionLayer`: บีบอัด Response ด้วย gzip

พิสูจน์ด้วยการยิง `/big-data` (JSON 500 รายการ) สองครั้ง — ครั้งแรกบอกว่ารับ gzip ได้ ครั้งที่สองบอกว่าไม่
รับการบีบอัด (`Accept-Encoding: identity`):

```bash
curl -sS -i http://127.0.0.1:3002/big-data -H "Accept-Encoding: gzip" -o /tmp/gz.bin -D -
curl -sS -i http://127.0.0.1:3002/big-data -H "Accept-Encoding: identity" -o /tmp/plain.bin -D -
```

ผลลัพธ์จริง (header ของแต่ละคำขอ):

```
# Accept-Encoding: gzip
HTTP/1.1 200 OK
content-type: application/json
vary: accept-encoding
content-encoding: gzip
access-control-allow-origin: http://localhost:5173
transfer-encoding: chunked

# Accept-Encoding: identity
HTTP/1.1 200 OK
content-type: application/json
vary: accept-encoding
access-control-allow-origin: http://localhost:5173
content-length: 65281
```

และขนาดไฟล์จริงที่บันทึกไว้:

```
-rw-r--r-- 1 root root  2990 บีบอัดแล้ว (gzip)
-rw-r--r-- 1 root root 65467 ไม่บีบอัด (plain)
```

จาก **65,467 bytes เหลือ 2,990 bytes** — บีบอัดได้ประมาณ **95%** ของขนาดเดิม (เพราะข้อมูลเป็น JSON ที่มี
รูปแบบซ้ำ ๆ สูง ซึ่ง gzip ทำได้ดีเป็นพิเศษกับข้อมูลลักษณะนี้) สังเกตว่า:

- `CompressionLayer` **เช็ค `Accept-Encoding` header ของ request เองอัตโนมัติ** — ถ้า client บอกว่ารับ
  `gzip` ได้ (ตามที่บราวเซอร์ทุกตัวส่งมาเป็นค่าเริ่มต้นเสมอ) มันถึงจะบีบอัด ถ้า client ไม่รับ (`identity`)
  มันจะปล่อย response ตามเดิมโดยไม่บีบอัด — เป็นการตัดสินใจ**ต่อ request** ไม่ใช่ตั้งค่าตายตัวทั้งเซิร์ฟเวอร์
- Header `vary: accept-encoding` ถูกเติมมาให้ทั้งสองกรณี — บอก proxy/cache ที่อยู่ระหว่างทางว่า response
  นี้อาจต่างกันได้ตาม `Accept-Encoding` ของ request จึงห้าม cache แบบไม่สนใจ header ตัวนี้
- เมื่อบีบอัด `content-length` หายไปแล้วกลายเป็น `transfer-encoding: chunked` แทน — เพราะขนาดที่บีบอัดแล้ว
  รู้ล่วงหน้าไม่ได้ก่อนที่ stream การบีบอัดจะจบ (streaming compression) จึงต้องส่งเป็น chunk ไปเรื่อย ๆ

#### `RequestBodyLimitLayer`: จำกัดขนาด Body ที่รับได้

พิสูจน์ด้วยการยิง body เล็ก (ผ่าน limit 1024 bytes ที่ตั้งไว้) แล้วยิง body ใหญ่กว่า (2000 bytes):

```bash
curl -sS -i -X POST http://127.0.0.1:3002/echo -d "hello world"
```

ผลลัพธ์จริง:

```
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 42

ได้รับ body ยาว 11 bytes
```

```bash
python3 -c "print('x'*2000)" > /tmp/bigbody.txt
curl -sS -i -X POST http://127.0.0.1:3002/echo --data-binary @/tmp/bigbody.txt
```

ผลลัพธ์จริง:

```
HTTP/1.1 413 Payload Too Large
content-type: text/plain; charset=utf-8
content-length: 21

length limit exceeded
```

`413 Payload Too Large` พร้อมข้อความ **`length limit exceeded`** คือ error message จริงที่ `tower-http`
คืนมา (ไม่ใช่ข้อความที่เราเขียนเอง) — สังเกตว่าในโค้ดตัวอย่างมี `.layer(DefaultBodyLimit::disable())` วางไว้
**ก่อน** `RequestBodyLimitLayer` — เหตุผลคือ **Axum เองมี body limit เริ่มต้นติดมาด้วยอยู่แล้ว** (ผ่าน
`DefaultBodyLimit`, ค่าเริ่มต้นคือ 2 MB) ถ้าไม่ปิดของเดิมก่อน จะมีสอง limit ทำงานซ้อนกันโดยไม่ตั้งใจ — ใน
โปรเจกต์จริงต้องตัดสินใจให้ชัดว่าจะใช้ค่าเริ่มต้นของ Axum เอง หรือปิดแล้วตั้งเองผ่าน
`RequestBodyLimitLayer` (มีประโยชน์เมื่อต้องการ limit ที่ต่างกันในแต่ละกลุ่ม route)

### 65.6 ลำดับการ `.layer()` มีผลจริง: พิสูจน์ด้วยการรันสองแบบเปรียบเทียบกัน

นี่คือหัวข้อที่สำคัญที่สุดของทั้งบทในเชิงปฏิบัติ — และเป็นจุดที่นักพัฒนา Axum จำนวนมากเข้าใจผิดหรือจำสูตรผิด
เพราะกฎมัน**ขึ้นกับวิธีที่คุณเรียก `.layer()`** ไม่ใช่กฎตายตัวกฎเดียว

#### ตั้งสมมติฐาน: Logging กับ Auth ใครควรอยู่ชั้นนอก?

สมมติมี middleware สองตัว: `log_mw` (log ก่อน/หลังทุก request) และ `auth_mw` (เช็ค header `x-api-key` ถ้าไม่
ถูกต้อง จะ short-circuit คืน `401` ทันทีโดยไม่เรียก handler จริง) คำถามคือ: **`.layer(log_mw).layer(auth_mw)`
กับ `.layer(auth_mw).layer(log_mw)` ให้ผลต่างกันจริงหรือไม่ ต่างกันอย่างไร?**

```rust
use axum::{
    extract::Request,
    http::StatusCode,
    middleware::{self, Next},
    response::{IntoResponse, Response},
    routing::get,
    Router,
};

async fn log_mw(req: Request, next: Next) -> Response {
    let method = req.method().clone();
    let uri = req.uri().clone();
    tracing::info!(%method, %uri, "LOG_MW: ก่อนส่งต่อ (before next)");
    let response = next.run(req).await;
    tracing::info!(status = %response.status(), "LOG_MW: ได้ response กลับมาแล้ว (after next)");
    response
}

async fn auth_mw(req: Request, next: Next) -> Response {
    let has_valid_key = req
        .headers()
        .get("x-api-key")
        .map(|v| v == "secret123")
        .unwrap_or(false);

    if !has_valid_key {
        tracing::warn!("AUTH_MW: ไม่มี/ผิด x-api-key -> ปฏิเสธที่นี่เลย (short-circuit)");
        return (StatusCode::UNAUTHORIZED, "unauthorized: missing or invalid x-api-key").into_response();
    }
    tracing::info!("AUTH_MW: x-api-key ถูกต้อง -> ส่งต่อ");
    next.run(req).await
}

async fn list_orders() -> &'static str {
    tracing::info!("HANDLER: list_orders กำลังทำงาน");
    "[\"order-1\", \"order-2\"]"
}
```

**กรณี A — `.layer(log_mw).layer(auth_mw)` (auth ถูกเพิ่ม*ทีหลัง*):**

```rust
# use axum::{routing::get, Router, middleware};
# async fn list_orders() -> &'static str { "" }
# async fn log_mw(req: axum::extract::Request, next: axum::middleware::Next) -> axum::response::Response { next.run(req).await }
# async fn auth_mw(req: axum::extract::Request, next: axum::middleware::Next) -> axum::response::Response { next.run(req).await }
let app: Router = Router::new()
    .route("/orders", get(list_orders))
    .layer(middleware::from_fn(log_mw))
    .layer(middleware::from_fn(auth_mw));
```

รันจริงแล้วยิง `curl` สองครั้ง (มี key ที่ถูกต้อง / ไม่มี key เลย) ได้ log จริงดังนี้ (คัดลอกตรงจากการรันจริง):

```
2026-09-26T23:32:27.044554Z  INFO order_demo: เริ่มเซิร์ฟเวอร์ order_demo order=auth-outer
2026-09-26T23:32:27.044833Z  INFO order_demo: ฟังอยู่ที่ http://127.0.0.1:3001 (ORDER=auth-outer)
2026-09-26T23:32:28.049782Z  INFO order_demo: AUTH_MW: x-api-key ถูกต้อง -> ส่งต่อ
2026-09-26T23:32:28.049832Z  INFO order_demo: LOG_MW: ก่อนส่งต่อ (before next) method=GET uri=/orders
2026-09-26T23:32:28.049856Z  INFO order_demo: HANDLER: list_orders กำลังทำงาน
2026-09-26T23:32:28.049873Z  INFO order_demo: LOG_MW: ได้ response กลับมาแล้ว (after next) status=200 OK
2026-09-26T23:32:28.056635Z  WARN order_demo: AUTH_MW: ไม่มี/ผิด x-api-key -> ปฏิเสธที่นี่เลย (short-circuit)
```

สังเกตให้ดี — สำหรับ request **ที่ไม่มี key** (บรรทัดสุดท้าย): มีแค่บรรทัด `AUTH_MW: ไม่มี/ผิด...` ปรากฏขึ้น
มาเท่านั้น **ไม่มีบรรทัดของ `LOG_MW` เลยแม้แต่บรรทัดเดียว** — เพราะ `auth_mw` (ที่ถูกเพิ่มเป็นตัวสุดท้าย)
กลายเป็น**ชั้นนอกสุด** ปฏิเสธ request ก่อนที่มันจะได้มีโอกาสไหลลึกเข้าไปถึง `log_mw` เลย — **ผลลัพธ์เชิงธุรกิจ
ที่ตามมา: request ที่ถูกปฏิเสธด้วย auth จะไม่ถูก log เลย** — ถ้าทีม operations ต้องการรู้ว่ามีใครพยายามยิง
request โดยไม่มี key เข้ามาบ้าง (สำคัญมากสำหรับ security monitoring) การจัดลำดับแบบนี้จะทำให้ข้อมูลนั้น
**หายไปเลย**

**กรณี B — `.layer(auth_mw).layer(log_mw)` (log ถูกเพิ่ม*ทีหลัง*):**

```rust
# use axum::{routing::get, Router, middleware};
# async fn list_orders() -> &'static str { "" }
# async fn log_mw(req: axum::extract::Request, next: axum::middleware::Next) -> axum::response::Response { next.run(req).await }
# async fn auth_mw(req: axum::extract::Request, next: axum::middleware::Next) -> axum::response::Response { next.run(req).await }
let app: Router = Router::new()
    .route("/orders", get(list_orders))
    .layer(middleware::from_fn(auth_mw))
    .layer(middleware::from_fn(log_mw));
```

รันจริงด้วย request สองคำขอเดิม ได้ log จริง (สลับเฉพาะการเรียง `.layer()`):

```
2026-09-26T23:32:42.640700Z  INFO order_demo: เริ่มเซิร์ฟเวอร์ order_demo order=log-outer
2026-09-26T23:32:42.640951Z  INFO order_demo: ฟังอยู่ที่ http://127.0.0.1:3001 (ORDER=log-outer)
2026-09-26T23:32:43.646467Z  INFO order_demo: LOG_MW: ก่อนส่งต่อ (before next) method=GET uri=/orders
2026-09-26T23:32:43.646527Z  INFO order_demo: AUTH_MW: x-api-key ถูกต้อง -> ส่งต่อ
2026-09-26T23:32:43.646548Z  INFO order_demo: HANDLER: list_orders กำลังทำงาน
2026-09-26T23:32:43.646567Z  INFO order_demo: LOG_MW: ได้ response กลับมาแล้ว (after next) status=200 OK
2026-09-26T23:32:43.653232Z  INFO order_demo: LOG_MW: ก่อนส่งต่อ (before next) method=GET uri=/orders
2026-09-26T23:32:43.653308Z  WARN order_demo: AUTH_MW: ไม่มี/ผิด x-api-key -> ปฏิเสธที่นี่เลย (short-circuit)
2026-09-26T23:32:43.653337Z  INFO order_demo: LOG_MW: ได้ response กลับมาแล้ว (after next) status=401 Unauthorized
```

คราวนี้สำหรับ request **ที่ไม่มี key** เห็น `LOG_MW` ทั้งก่อนและหลัง (`status=401 Unauthorized`) — เพราะ
`log_mw` (เพิ่มเป็นตัวสุดท้าย) กลายเป็น**ชั้นนอกสุด** เห็น**ทุก**request ที่ผ่านเข้ามา ไม่ว่า `auth_mw`
(ที่อยู่ชั้นในกว่า) จะปฏิเสธมันหรือไม่ก็ตาม

#### สรุปกฎที่พิสูจน์แล้วจากการรันจริงสำหรับ `Router::layer()` แบบเรียงต่อกันตรง ๆ

**เมื่อเรียก `.layer()` ซ้อนกันหลายครั้งตรง ๆ บน `Router` (ไม่ผ่าน `ServiceBuilder`) — `.layer()` ตัว
ล่าสุดที่เรียก (ตัวที่เขียนอยู่ล่างสุดในโค้ด) จะกลายเป็นชั้นนอกสุด** เหตุผลเชิงกลไกมาจาก source code ของ
Axum เอง (`axum-0.8.9/src/routing/path_router.rs`): แต่ละครั้งที่เรียก `.layer(new_layer)` มันทำ
`endpoint.layer(new_layer)` — เอา layer ใหม่ไปห่อ (`wrap`) รอบ ๆ **สิ่งที่มีอยู่แล้วทั้งหมด** ทำให้ layer ที่
เพิ่งเพิ่มเข้ามาอยู่วงนอกสุดเสมอ นี่คือกฎที่**ตรงข้าม**กับกฎของ `ServiceBuilder` ในหัวข้อ 65.3 (ที่เพิ่มก่อน =
ชั้นนอกสุด) พอดี ๆ!

**คำแนะนำเชิงปฏิบัติที่ตรงกับเอกสารของ Axum เอง**: เพื่อไม่ต้องจำกฎที่กลับกันนี้ให้ปวดหัว **ควรประกอบ
middleware หลายตัวด้วย `tower::ServiceBuilder` แล้ว `.layer()` เข้า `Router` เพียง "ครั้งเดียว"** — ลองพิสูจน์
ด้วยการรันจริงว่าวิธีนี้ทำให้กฎกลับไปเป็นแบบ `ServiceBuilder` (เพิ่มก่อน = ชั้นนอกสุด) ตามที่คาด:

```rust
use tower::ServiceBuilder;
# use axum::{routing::get, Router, middleware};
# async fn list_orders() -> &'static str { "" }
# async fn log_mw(req: axum::extract::Request, next: axum::middleware::Next) -> axum::response::Response { next.run(req).await }
# async fn auth_mw(req: axum::extract::Request, next: axum::middleware::Next) -> axum::response::Response { next.run(req).await }

let app: Router = Router::new().route("/orders", get(list_orders)).layer(
    ServiceBuilder::new()
        .layer(middleware::from_fn(log_mw))  // เพิ่มก่อน -> ชั้นนอกสุด (ตาม ServiceBuilder)
        .layer(middleware::from_fn(auth_mw)), // เพิ่มทีหลัง -> ชั้นในกว่า
);
```

รันจริงแล้วยิง request ทั้งสองแบบเดิม ได้ log จริง:

```
2026-09-26T23:34:16.366349Z  INFO order_demo: เริ่มเซิร์ฟเวอร์ order_demo order=service-builder
2026-09-26T23:34:17.373514Z  INFO order_demo: LOG_MW: ก่อนส่งต่อ (before next) method=GET uri=/orders
2026-09-26T23:34:17.373575Z  INFO order_demo: AUTH_MW: x-api-key ถูกต้อง -> ส่งต่อ
2026-09-26T23:34:17.373612Z  INFO order_demo: HANDLER: list_orders กำลังทำงาน
2026-09-26T23:34:17.373635Z  INFO order_demo: LOG_MW: ได้ response กลับมาแล้ว (after next) status=200 OK
2026-09-26T23:34:17.380547Z  INFO order_demo: LOG_MW: ก่อนส่งต่อ (before next) method=GET uri=/orders
2026-09-26T23:34:17.380603Z  WARN order_demo: AUTH_MW: ไม่มี/ผิด x-api-key -> ปฏิเสธที่นี่เลย (short-circuit)
2026-09-26T23:34:17.380621Z  INFO order_demo: LOG_MW: ได้ response กลับมาแล้ว (after next) status=401 Unauthorized
```

ได้ผลลัพธ์**เหมือนกับกรณี B เป๊ะ** (log เห็นทุก request รวมถึงที่ถูกปฏิเสธ) เพราะตอนนี้ `log_mw` ถูกเพิ่ม
**ก่อน** `auth_mw` เข้า `ServiceBuilder` — และตาม `ServiceBuilder`, "เพิ่มก่อน = ชั้นนอกสุด" — `log_mw` จึง
เป็นชั้นนอกสุดเหมือนเดิม ทั้งสองวิธี (Router chaining แบบย้อนกฎ, หรือ ServiceBuilder แบบปกติ) **ให้ผลลัพธ์
สุดท้ายเหมือนกันได้ถ้าจัดลำดับให้ถูก** — แต่ `ServiceBuilder` ปลอดภัยกว่าเพราะกฎมันตรงกับสัญชาตญาณ (อ่านจาก
บนลงล่าง เพิ่มก่อนคือวงนอก) และยังมีข้อดีด้าน performance คือ Axum ต้องเรียก `.layer()` ที่ `Router` เพียง
ครั้งเดียวแทนที่จะสร้าง wrapper ซ้อนกันหลายชั้นทีละขั้น

**บทเรียนสำหรับงานจริง**: วาง middleware ที่ทำ**observability** (เช่น `TraceLayer`, logging middleware)
ไว้เป็น**ชั้นนอกสุดเสมอ** เพื่อให้เห็น**ทุก**request รวมถึงที่ถูกปฏิเสธโดย middleware ชั้นในกว่า (เช่น auth,
rate limiting) — นี่คือเหตุผลที่ตัวอย่าง capstone ท้ายบทจะวาง `TraceLayer` ไว้เป็นตัวสุดท้ายที่ `.layer()`
เข้า `Router` เสมอ

### 65.7 `.layer()` กับ `.route_layer()`: ต่างกันตรงจุดที่ 404 กลายเป็นอย่างอื่นหรือไม่

Axum มีเมธอดสองตัวที่ดูคล้ายกันมากแต่มีความหมายต่างกันโดยพื้นฐาน — และความต่างนี้มีผลจริงต่อพฤติกรรมของ
API เมื่อ path ไม่ตรงกับ route ใดเลย มาพิสูจน์ด้วยเซิร์ฟเวอร์เล็ก ๆ ที่มี middleware ตัวเดียวที่**ปฏิเสธทุก
คำขอเสมอ** (จำลอง auth middleware ที่เข้มงวดที่สุดเท่าที่จะเป็นไปได้ เพื่อให้เห็นผลชัด ๆ):

```rust
use axum::{routing::get, Router};
use axum::http::StatusCode;
use axum::response::IntoResponse;
use axum::extract::Request;
use axum::middleware::{self, Next};
use axum::response::Response;

async fn always_reject(_req: Request, _next: Next) -> Response {
    (StatusCode::UNAUTHORIZED, "always_reject: ไม่ผ่าน").into_response()
}

async fn ok_handler() -> &'static str {
    "ok"
}

#[tokio::main]
async fn main() {
    let mode = std::env::var("MODE").unwrap_or_else(|_| "layer".to_string());

    let app = match mode.as_str() {
        "route_layer" => Router::new()
            .route("/foo", get(ok_handler))
            .route_layer(middleware::from_fn(always_reject)),
        _ => Router::new()
            .route("/foo", get(ok_handler))
            .layer(middleware::from_fn(always_reject)),
    };

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3005").await.unwrap();
    axum::serve(listener, app.into_make_service()).await.unwrap();
}
```

**โหมด `.layer()` (ค่าเริ่มต้น)** — ทดสอบทั้ง path ที่มีจริง (`/foo`) และ path ที่ไม่มีอยู่เลย
(`/does-not-exist`):

```
### /foo (มี route จริง) ###
HTTP/1.1 401 Unauthorized
content-length: 36

always_reject: ไม่ผ่าน

### /does-not-exist (ไม่มี route ไหนตรงเลย) ###
HTTP/1.1 401 Unauthorized
content-length: 36

always_reject: ไม่ผ่าน
```

สังเกต: **ทั้งสอง path ได้ `401` เหมือนกัน!** แม้ `/does-not-exist` ไม่มี route ไหนตรงกับมันเลยก็ตาม —
เพราะ `.layer()` ห่อ**ทั้ง path_router และ fallback ของ Router** (ตามที่พิสูจน์ไว้แล้วในหัวข้อ 65.2 จาก
source code จริง) ทำให้ middleware ถูกเรียกก่อนที่ Axum จะรู้ด้วยซ้ำว่า path นี้ไม่มี route ไหนรองรับ — ผล
คือ **404 ที่ควรจะเป็นถูกกลืนกลายเป็น 401 ไปเสีย** ซึ่งอาจไม่ใช่พฤติกรรมที่ตั้งใจ (client ที่พิมพ์ URL ผิดจะ
เห็น "unauthorized" ทั้งที่ปัญหาจริงคือ URL ผิด ไม่ใช่เรื่อง auth เลย)

**โหมด `.route_layer()`** — รันเซิร์ฟเวอร์เดียวกันด้วย `MODE=route_layer`:

```
### /foo (มี route จริง) ###
HTTP/1.1 401 Unauthorized
content-length: 36

always_reject: ไม่ผ่าน

### /does-not-exist (ไม่มี route ไหนตรงเลย) ###
HTTP/1.1 404 Not Found
content-length: 0
```

คราวนี้ `/does-not-exist` ได้ **`404 Not Found`** ตามที่ควรจะเป็น! — เพราะ `.route_layer()` ห่อ**เฉพาะ
route ที่ประกาศไว้จริง**เท่านั้น (`path_router`) **ไม่ห่อ fallback** เอกสารทางการของ Axum เขียนอธิบายไว้
ตรงประเด็นนี้เป๊ะ:

> "This is useful for middleware that return early (such as authorization) which might otherwise convert
> a `404 Not Found` into a `401 Unauthorized`."

**บทสรุปเชิงปฏิบัติ**: ถ้า middleware ของคุณมีลักษณะ "ปฏิเสธคำขอได้" (เช่น auth, rate limiting เฉพาะ
บาง route) และไม่ต้องการให้มันไปเปลี่ยนพฤติกรรมของ 404 (path ที่ไม่มีอยู่จริง) ให้ใช้ **`.route_layer()`**
แต่ถ้า middleware เป็นแบบ "สังเกตการณ์" อย่างเดียว ไม่ปฏิเสธอะไร (เช่น `TraceLayer` ที่อยากเห็น 404 ด้วยเพื่อ
ตรวจสอบว่ามีคนยิง path ผิดบ่อยแค่ไหน) ใช้ **`.layer()`** ตามปกติ — และทั้งสองตัวมีข้อจำกัดร่วมกันที่สำคัญ
คือ **middleware จะถูกใส่ให้เฉพาะ route ที่ประกาศไว้ "ก่อน" เรียก `.layer()`/`.route_layer()` เท่านั้น**
route ที่เพิ่มเข้ามา**หลังจาก**เรียกสองเมธอดนี้แล้วจะ**ไม่มี**middleware ตัวนั้นครอบอยู่ (นี่คือเหตุผลที่ควร
ประกาศ route ทั้งหมดของกลุ่มนั้นให้ครบก่อน แล้วค่อยเรียก `.layer()`/`.route_layer()` ปิดท้าย)

### 65.8 การแบ่งกลุ่ม Route ให้มี Middleware ต่างกันด้วย Nested/Merged Router

ในโปรเจกต์จริง ไม่ใช่ทุก route ต้องการ middleware ชุดเดียวกัน — เช่น endpoint สาธารณะ (`GET /books`) ไม่
ควรต้อง auth แต่ endpoint ที่แก้ไขข้อมูล (`POST /books/{id}/borrow`) ต้อง auth วิธีที่สะอาดที่สุดคือแยก
`Router` เป็นกลุ่ม ๆ ใส่ middleware ที่ต่างกันในแต่ละกลุ่ม แล้ว `.merge()` เข้าด้วยกัน:

```rust
use axum::{routing::{get, post}, Router, middleware};
# use axum::extract::State;
# #[derive(Clone)] struct AppState;
# async fn list_books() -> &'static str { "" }
# async fn get_book() -> &'static str { "" }
# async fn borrow_book() -> &'static str { "" }
# async fn toy_auth_middleware(
#     State(_state): State<AppState>,
#     req: axum::extract::Request,
#     next: axum::middleware::Next,
# ) -> axum::response::Response { next.run(req).await }
# fn build(state: AppState) -> Router {
// เส้นทางสาธารณะ: ไม่ต้อง auth เลย
let public_routes = Router::new()
    .route("/books", get(list_books))
    .route("/books/{id}", get(get_book));

// เส้นทางที่ต้อง auth: layer เฉพาะกลุ่มนี้ *ก่อน* merge เข้า router หลัก
let protected_routes = Router::new()
    .route("/books/{id}/borrow", post(borrow_book))
    .layer(middleware::from_fn_with_state(state.clone(), toy_auth_middleware));

// merge เข้าด้วยกัน — เฉพาะ protected_routes เท่านั้นที่มี auth middleware ครอบอยู่
public_routes.merge(protected_routes).with_state(state)
# }
```

จุดสำคัญคือ **`.layer()` ถูกเรียกบน `protected_routes` ก่อนที่จะ `.merge()` เข้ากับ `public_routes`** — ทำให้
middleware `toy_auth_middleware` ครอบอยู่แค่รอบ ๆ `protected_routes` เท่านั้น ไม่กระทบ `public_routes` เลย
แม้จะถูก merge เข้าเป็น `Router` เดียวกันแล้วก็ตาม (การ merge ไม่ได้ทำให้ layer ของฝั่งหนึ่งลามไปอีกฝั่ง) —
นี่คือรูปแบบมาตรฐานสำหรับแยก middleware ตามกลุ่ม route ในโปรเจกต์จริง ต่อยอดจากแนวคิด `nest()`/`merge()`
ที่ Part 63 สอนไว้ตรง ๆ

### 65.9 เขียน Custom Middleware ด้วย `axum::middleware::from_fn`

ตัวอย่างในหัวข้อก่อน ๆ ใช้ `axum::middleware::from_fn` ไปแล้วหลายครั้งโดยยังไม่ได้อธิบายอย่างเป็นทางการ —
นี่คือวิธี**ที่แนะนำที่สุด**สำหรับเขียน middleware กำหนดเองใน Axum เพราะไม่ต้องลงไปเขียน `Service`/`Layer`
เองเหมือนหัวข้อ 65.3 เลย — แค่เขียน**ฟังก์ชัน async** ที่มีรูปแบบตายตัวแบบนี้:

```rust
use axum::{extract::Request, middleware::Next, response::Response};

async fn my_middleware(req: Request, next: Next) -> Response {
    // (1) ทำอะไรกับ request ก่อนส่งต่อได้ที่นี่
    let response = next.run(req).await; // (2) เรียก middleware/handler ชั้นถัดไป
    // (3) ทำอะไรกับ response ก่อนส่งกลับได้ที่นี่
    response
}
```

พารามิเตอร์ **`next: Next`** คือตัวแทนของ "ทุกอย่างที่อยู่ชั้นในกว่า" (middleware ตัวต่อไป หรือ handler ปลาย
ทาง) — เรียก `next.run(req).await` เมื่อไหร่ก็เมื่อนั้นที่ request จะไหลลึกเข้าไปต่อ ถ้า**ไม่**เรียก
`next.run(...)` เลย (คืน `Response` ของตัวเองไปตรง ๆ) คือการ **short-circuit** ตามที่อธิบายไว้ในหัวข้อ 65.4

#### ตัวอย่างเต็ม: Timing Middleware ที่ log ผ่าน `tracing` (ต่อยอด Part 60)

```rust
use axum::{extract::Request, middleware::Next, response::Response};
use std::time::Instant;

async fn timing_middleware(req: Request, next: Next) -> Response {
    let method = req.method().clone();
    let path = req.uri().path().to_string();
    let start = Instant::now();

    let response = next.run(req).await;

    let elapsed = start.elapsed();
    tracing::info!(
        %method,
        %path,
        status = response.status().as_u16(),
        elapsed_ms = elapsed.as_millis() as u64,
        "request เสร็จสิ้น"
    );
    response
}
```

สังเกตว่าเราจับ `method`/`path` ไว้ **ก่อน** เรียก `next.run(req)` เพราะ `req` ถูก**ย้าย (move)** เข้าไปใน
`next.run(req)` ทั้งตัว — หลังจากบรรทัดนั้น `req` เดิมใช้ไม่ได้แล้ว (ตามกฎ ownership จาก Part 4-5) ต้องดึง
ข้อมูลที่ต้องใช้ทีหลัง (เพื่อ log ตอนจบ) ออกมาเก็บไว้ล่วงหน้าเสมอ — นี่คือกับดักการเขียน middleware ที่พบบ่อย
มากสำหรับคนที่เพิ่งเริ่มเขียน (จะกลับมาย้ำในหัวข้อกับดัก)

#### ตัวอย่างเต็ม: Toy Auth-Gate Middleware ที่ Short-Circuit ด้วย 401

```rust
use axum::{
    extract::Request,
    http::StatusCode,
    middleware::Next,
    response::{IntoResponse, Response},
};

async fn toy_auth_gate(req: Request, next: Next) -> Response {
    let has_valid_key = req
        .headers()
        .get("x-api-key")
        .and_then(|v| v.to_str().ok())
        .map(|v| v == "secret123")
        .unwrap_or(false);

    if !has_valid_key {
        // short-circuit: คืน response ตรงนี้เลย ไม่เรียก next.run() — handler ปลายทางไม่ถูกเรียกแม้แต่นิดเดียว
        return (StatusCode::UNAUTHORIZED, "missing or invalid x-api-key").into_response();
    }

    next.run(req).await
}
```

นี่คือ**ตัวอย่างสาธิตกลไก** ไม่ใช่ระบบ auth ที่ปลอดภัยสำหรับ production จริง (เทียบ API key แบบ string
เปรียบเทียบตรง ๆ แบบนี้เสี่ยงต่อ timing attack และไม่มีการจัดการ user/role อะไรเลย) — **Part 74 (JWT)** และ
**Part 76 (Authorization และ RBAC)** จะสอนวิธีเช็ค auth ที่ถูกต้องและปลอดภัยจริง แต่**กลไกที่ใช้ห่อ
middleware เข้ากับ route ก็ยังเป็นแบบเดียวกับที่เห็นในบทนี้เป๊ะ** — สิ่งที่เปลี่ยนแค่ "ตรรกะภายในของการ
ตรวจสอบ" เท่านั้น

### 65.10 `from_fn_with_state`: เมื่อ Middleware ต้องเข้าถึง `AppState`

Middleware ในหัวข้อก่อนใช้ค่าคงที่ (`"secret123"`) ฝังอยู่ในโค้ดตรง ๆ — ในงานจริง ค่าที่ต้องใช้เช็ค (เช่น
API key ที่จริง, connection pool ของ database) ต้องมาจาก **`AppState`** เดียวกับที่ Part 64 สอนให้แชร์ผ่าน
`State<T>` extractor นั่นเอง `axum::middleware::from_fn_with_state` คือตัวที่ให้ middleware เข้าถึง state
นั้นได้:

```rust
use axum::{
    extract::{Request, State},
    http::StatusCode,
    middleware::Next,
    response::{IntoResponse, Response},
};
use std::sync::{atomic::{AtomicU64, Ordering}, Arc};

#[derive(Clone)]
struct AppState {
    valid_api_key: Arc<String>,
    request_counter: Arc<AtomicU64>,
}

async fn auth_gate_with_state(
    State(state): State<AppState>, // extractor ตัวแรกของ signature — ดึงจาก state ที่ผูกไว้กับ layer นี้
    mut req: Request,
    next: Next,
) -> Response {
    let count = state.request_counter.fetch_add(1, Ordering::SeqCst) + 1;
    tracing::debug!(count, "auth_gate_with_state: request ลำดับที่");

    let provided_key = req
        .headers()
        .get("x-api-key")
        .and_then(|v| v.to_str().ok())
        .map(|s| s.to_string());

    match provided_key {
        Some(key) if key == *state.valid_api_key => next.run(req).await,
        Some(_) => (StatusCode::UNAUTHORIZED, "invalid x-api-key").into_response(),
        None => (StatusCode::UNAUTHORIZED, "missing x-api-key header").into_response(),
    }
}
```

ผูกเข้ากับ router ด้วย `middleware::from_fn_with_state(state.clone(), auth_gate_with_state)` แทน
`middleware::from_fn(...)` ธรรมดา — สังเกตว่าต้อง `.clone()` state ก่อนส่งเข้า `from_fn_with_state` เพราะ
ค่านี้จะถูก**ฝัง (capture) เข้าไปในตัว middleware โดยตรง** ไม่ได้พึ่งพา generic state ของ `Router` เอง — ผล
คือ middleware ตัวนี้ใช้ได้แม้ `Router` ยังไม่ได้ `.with_state(...)` เลยตอนที่ `.layer()` ถูกเรียก (ตราบใดที่
สุดท้ายมีการเรียก `.with_state()` ให้ type ตรงกันก่อนส่งเข้า `axum::serve`)

พิสูจน์ด้วยการรันจริงและยิง `curl` สี่แบบ (public / protected ไม่มี key / protected key ผิด / protected
key ถูก):

```
=== public (no auth needed) ===
HTTP/1.1 200 OK
หน้านี้เข้าได้โดยไม่ต้อง auth

=== protected WITHOUT api key ===
HTTP/1.1 401 Unauthorized
missing x-api-key header

=== protected WITH wrong api key ===
HTTP/1.1 401 Unauthorized
invalid x-api-key

=== protected WITH correct api key ===
HTTP/1.1 200 OK
สวัสดีคุณ phutjirakul คุณเข้าถึงหน้าที่ต้อง auth ได้แล้ว
```

และ log จริงที่ได้ (สังเกต `count` ที่เพิ่มขึ้นทุกครั้งที่ auth middleware ถูกเรียก — พิสูจน์ว่า state ถูก
แชร์และเปลี่ยนแปลงจริงข้าม request):

```
2026-09-26T23:38:29.671490Z  INFO from_fn_demo: request เสร็จสิ้น method=GET path=/public status=200 elapsed_ms=0
2026-09-26T23:38:29.680266Z DEBUG from_fn_demo: auth_gate_with_state: request ลำดับที่ count=1
2026-09-26T23:38:29.680312Z  WARN from_fn_demo: auth_gate_with_state: ไม่มี x-api-key -> 401
2026-09-26T23:38:29.680341Z  INFO from_fn_demo: request เสร็จสิ้น method=GET path=/protected status=401 elapsed_ms=0
2026-09-26T23:38:29.687225Z DEBUG from_fn_demo: auth_gate_with_state: request ลำดับที่ count=2
2026-09-26T23:38:29.687299Z  WARN from_fn_demo: auth_gate_with_state: x-api-key ผิด -> 401
2026-09-26T23:38:29.687326Z  INFO from_fn_demo: request เสร็จสิ้น method=GET path=/protected status=401 elapsed_ms=0
2026-09-26T23:38:29.693262Z DEBUG from_fn_demo: auth_gate_with_state: request ลำดับที่ count=3
2026-09-26T23:38:29.693350Z  INFO from_fn_demo: request เสร็จสิ้น method=GET path=/protected status=200 elapsed_ms=0
```

สังเกตว่า `count` เดินหน้าต่อเนื่อง (1, 2, 3) ข้ามทั้งสาม request ที่ยิงไปที่ `/protected` — พิสูจน์ว่า
`Arc<AtomicU64>` ที่แชร์ผ่าน `AppState` ทำงานถูกต้องข้าม request เหมือนที่ Part 64 อธิบายไว้เรื่อง
`Arc`/`Mutex` ใน `AppState` — และ `/public` ไม่กระทบ counter นี้เลย (เพราะ auth middleware ไม่ได้ครอบมัน)

### 65.11 แนบข้อมูลเข้า Request ให้ Handler ปลายทางใช้ต่อผ่าน `Extension`

จากตัวอย่างข้างบน สังเกตว่าถ้า auth ผ่านแล้ว มันก็แค่เรียก `next.run(req).await` — handler ปลายทางไม่รู้
เลยว่า "ใคร" เป็นคนยิง request มา รู้แค่ว่ามันผ่าน auth เท่านั้น ในงานจริงมักอยากส่งต่อ**ข้อมูลที่ middleware
ตรวจสอบมาแล้ว** (เช่น `user_id`, `role`) ให้ handler ใช้ต่อได้โดยไม่ต้องตรวจซ้ำ — กลไกที่ทำสิ่งนี้คือการแนบ
ข้อมูลเข้า **request extensions** ก่อนเรียก `next.run(...)` แล้วให้ handler ดึงออกมาผ่าน **`Extension<T>`
extractor** ที่ Part 64 แนะนำไว้แล้ว:

```rust
use axum::{
    extract::Request,
    http::StatusCode,
    middleware::Next,
    response::{IntoResponse, Response},
    Extension,
};

// ข้อมูลผู้ใช้ปัจจุบันที่ middleware ตรวจสอบแล้วแนบเข้า request ให้ handler ปลายทางดึงไปใช้ได้
#[derive(Clone, Debug)]
struct CurrentUser {
    username: String,
}

async fn auth_gate_attach_user(mut req: Request, next: Next) -> Response {
    let key_ok = req
        .headers()
        .get("x-api-key")
        .map(|v| v == "secret123")
        .unwrap_or(false);

    if !key_ok {
        return (StatusCode::UNAUTHORIZED, "unauthorized").into_response();
    }

    // แนบ CurrentUser เข้า request extensions — ต้องทำ *ก่อน* เรียก next.run() เท่านั้น
    req.extensions_mut().insert(CurrentUser {
        username: "phutjirakul".to_string(),
    });

    next.run(req).await
}

// handler ที่ดึง CurrentUser ที่ middleware แนบมาให้ ผ่าน Extension extractor (Part 64)
async fn protected_handler(Extension(user): Extension<CurrentUser>) -> String {
    format!("สวัสดีคุณ {} คุณเข้าถึงหน้าที่ต้อง auth ได้แล้ว", user.username)
}
```

จุดสำคัญที่ต้องแม่น: `req.extensions_mut().insert(...)` ต้องเรียก **ก่อน** `next.run(req)` เท่านั้น — เพราะ
`req` ถูก move เข้า `next.run()` ไปแล้ว ไม่มีโอกาสแก้ไขมันอีกทีหลัง และ**ชนิดข้อมูลที่ใส่ต้องตรงกับชนิดที่
handler ขอผ่าน `Extension<T>`เป๊ะ** (`CurrentUser` ในตัวอย่างนี้) — ถ้า middleware ไม่ได้ทำงาน (เช่น ลืม
`.layer()` มันเข้า route นั้น) หรือใส่ชนิดข้อมูลผิด `Extension<CurrentUser>` extractor จะดึงไม่เจอ ซึ่งเป็น
กับดักที่จะอธิบายในหัวข้อถัดไป

### 65.12 เมื่อ Middleware คืน `Err`: จุดเชื่อมกับ Part 66

ตลอดบทนี้ ทุก middleware ที่เขียนมี**ชนิด return เป็น `Response` ตรง ๆ** (ไม่ใช่ `Result<Response, E>`) —
เพราะเราคืน `Response` ของตัวเอง (เช่น `(StatusCode::UNAUTHORIZED, "...")​.into_response()`) เสมอเมื่อ
ต้องการปฏิเสธ ไม่เคยคืน "error" ที่ปล่อยลอยไม่มีคนจับ — นี่**ไม่ใช่เรื่องบังเอิญ** แต่เป็นไปตามข้อเท็จจริงที่
พิสูจน์ไว้แล้วในหัวข้อ 65.2: **`Router` ต้องการ `Error: Into<Infallible>` เสมอ** — พูดอีกแบบคือ **Axum
บังคับ (ผ่าน type system ตอน compile) ว่า chain ของ middleware + handler ทั้งหมดต้อง "ไม่มีทาง error"
เลย ต้องคืน `Response` เสมอไม่ว่าจะกรณีไหน**

ทีนี้คำถามคือ: แล้วถ้า middleware ที่เราไม่ได้เขียนเอง (เช่น ตัวจาก `tower` ดิบ ๆ ที่ไม่ได้ออกแบบมาสำหรับ
Axum โดยเฉพาะ) มี `Error` type ที่ไม่ใช่ `Infallible` ล่ะ? เอกสารทางการของ Axum ตอบเรื่องนี้ไว้ตรง ๆ ด้วย
ตัวอย่างที่ใช้ `tower::timeout::TimeoutLayer` (ตัวดิบจาก `tower` เอง คนละตัวกับ `tower_http::timeout::
TimeoutLayer` ที่เราใช้ในหัวข้อ 65.5) ซึ่งเปลี่ยน `Error` เป็น `BoxError` เมื่อ timeout เกิดขึ้น:

```rust
use axum::{
    error_handling::HandleErrorLayer,
    http::StatusCode,
    routing::get,
    BoxError, Router,
};
use tower::{timeout::TimeoutLayer, ServiceBuilder};
use std::time::Duration;

async fn handler() {}

let app: Router = Router::new().route("/", get(handler)).layer(
    ServiceBuilder::new()
        // middleware นี้ต้องอยู่ *เหนือ* TimeoutLayer เพราะมันรับ error ที่ TimeoutLayer คืนมา
        .layer(HandleErrorLayer::new(|_: BoxError| async {
            StatusCode::REQUEST_TIMEOUT
        }))
        .layer(TimeoutLayer::new(Duration::from_secs(10))),
);
```

**`HandleErrorLayer`** คือสะพานที่แปลง `Err(E)` ใด ๆ กลับให้เป็น `Response` ที่ถูกต้อง (ในตัวอย่างนี้แปลง
`BoxError` เป็น `408 Request Timeout` เสมอ) — มันต้องอยู่ "เหนือ" (ห่ออยู่รอบนอกกว่า) ตัวที่สร้าง error
เพราะมันต้องเป็นคนแรกที่ได้เห็น error นั้นก่อนที่จะไหลลึกลงไปหา `Router` ที่ต้องการ `Infallible` เสมอ — นี่
คือเหตุผลเชิงลึกว่าทำไม `tower_http::timeout::TimeoutLayer` (ตัวที่เราใช้จริงในหัวข้อ 65.5) ถึงถูกออกแบบให้
**ไม่ต้องพ่วง `HandleErrorLayer` เลย**: มันจัดการแปลง timeout เป็น `Response` ให้เสร็จสรรพภายในตัวเอง โดย
ไม่เปลี่ยน `Error` type ออกจาก `Infallible` แม้แต่นิดเดียว — **`tower-http` ถูกออกแบบมาให้ "เข้ากับ Axum ได้
ทันทีโดยไม่ต้องใช้ `HandleErrorLayer`" เสมอ** ต่างจากตัวดิบใน `tower` เองที่ทั่วไปแล้วไม่ได้ผูกกับสมมติฐาน
เรื่อง `IntoResponse` ของ Axum

กลไก `HandleErrorLayer` เต็มรูปแบบ, การออกแบบ error type ของทั้งแอปให้แปลงเป็น `Response` ได้อย่างเป็นระบบ
(ผ่าน `IntoResponse` ของ Axum เอง), และวิธีจัดการ error ที่มาจาก extractor ที่ล้มเหลว (rejection) — คือ
เนื้อหาหลักทั้งหมดของ **Part 66** ที่กำลังจะมาถึง บทนี้แค่พอให้เข้าใจว่า**middleware ที่ปฏิเสธ request ต้อง
คืน `Response` ที่สมบูรณ์เสมอ ไม่ใช่ `Err` ที่ปล่อยลอยไม่มีใครแปลงกลับให้ถูกต้อง** — ตัวอย่าง `toy_auth_gate`/
`auth_gate_with_state` ที่เขียนมาตลอดบทนี้ทำถูกตามหลักการนี้แล้วทุกจุด (คืน `(StatusCode, &str).
into_response()` เสมอ ไม่มีจุดไหนคืน `Err` เลย)

### 65.13 Capstone: เพิ่ม Middleware Stack เต็มรูปแบบให้ระบบห้องสมุด

มาประกอบทุกอย่างที่เรียนมาในบทนี้เข้าด้วยกัน โดยต่อยอดจากระบบห้องสมุด (Library API) ที่ Part 63-64 เริ่มวาง
โครงไว้ — เพิ่ม endpoint และ middleware stack เต็มรูปแบบ: `TraceLayer` + `CorsLayer` + custom timing
middleware + custom toy-auth middleware ที่แนบ `CurrentUser` เข้า request

```rust
use axum::{
    extract::{Path, Request, State},
    http::{HeaderValue, Method, StatusCode},
    middleware::{self, Next},
    response::{IntoResponse, Response},
    routing::{get, post},
    Extension, Json, Router,
};
use serde::Serialize;
use std::collections::HashMap;
use std::sync::{Arc, Mutex};
use std::time::Instant;
use tower_http::cors::CorsLayer;
use tower_http::trace::TraceLayer;

#[derive(Clone, Debug, Serialize)]
struct Book {
    id: u32,
    title: String,
    available: bool,
}

#[derive(Clone)]
struct AppState {
    catalog: Arc<Mutex<HashMap<u32, Book>>>,
    valid_api_key: Arc<String>,
}

#[derive(Clone, Debug)]
struct CurrentUser {
    username: String,
}

fn seed_catalog() -> HashMap<u32, Book> {
    let mut m = HashMap::new();
    m.insert(1, Book { id: 1, title: "The Rust Programming Language".into(), available: true });
    m.insert(2, Book { id: 2, title: "Zero To Production In Rust".into(), available: true });
    m
}

// --- Custom middleware #1: จับเวลาทุก request แล้ว log ผ่าน tracing (Part 60) ---
async fn timing_middleware(req: Request, next: Next) -> Response {
    let method = req.method().clone();
    let path = req.uri().path().to_string();
    let start = Instant::now();
    let response = next.run(req).await;
    tracing::info!(
        %method, %path,
        status = response.status().as_u16(),
        elapsed_us = start.elapsed().as_micros() as u64,
        "timing_middleware: จบ request"
    );
    response
}

// --- Custom middleware #2: toy auth-gate — ตรวจ x-api-key แล้วแนบ CurrentUser (foreshadow Part 74/76) ---
async fn toy_auth_middleware(
    State(state): State<AppState>,
    mut req: Request,
    next: Next,
) -> Response {
    let key = req.headers().get("x-api-key").and_then(|v| v.to_str().ok());
    match key {
        Some(k) if k == state.valid_api_key.as_str() => {
            req.extensions_mut().insert(CurrentUser { username: "phutjirakul".to_string() });
            next.run(req).await
        }
        _ => {
            tracing::warn!("toy_auth_middleware: ปฏิเสธคำขอ (x-api-key ไม่ถูกต้อง/ไม่มี)");
            (
                StatusCode::UNAUTHORIZED,
                Json(serde_json::json!({ "error": "unauthorized", "reason": "missing or invalid x-api-key" })),
            )
                .into_response()
        }
    }
}

async fn health() -> &'static str {
    "ok"
}

async fn list_books(State(state): State<AppState>) -> Json<Vec<Book>> {
    let catalog = state.catalog.lock().unwrap();
    Json(catalog.values().cloned().collect())
}

async fn get_book(State(state): State<AppState>, Path(id): Path<u32>) -> Response {
    let catalog = state.catalog.lock().unwrap();
    match catalog.get(&id) {
        Some(book) => Json(book.clone()).into_response(),
        None => (StatusCode::NOT_FOUND, "book not found").into_response(),
    }
}

async fn borrow_book(
    State(state): State<AppState>,
    Path(id): Path<u32>,
    Extension(user): Extension<CurrentUser>,
) -> Response {
    let mut catalog = state.catalog.lock().unwrap();
    match catalog.get_mut(&id) {
        Some(book) if book.available => {
            book.available = false;
            tracing::info!(book_id = id, borrower = %user.username, "ยืมหนังสือสำเร็จ");
            Json(serde_json::json!({ "borrowed_by": user.username, "book": book.clone() })).into_response()
        }
        Some(_) => (StatusCode::CONFLICT, "book is already borrowed").into_response(),
        None => (StatusCode::NOT_FOUND, "book not found").into_response(),
    }
}

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt()
        .with_env_filter(
            tracing_subscriber::EnvFilter::try_from_default_env().unwrap_or_else(|_| {
                tracing_subscriber::EnvFilter::new("capstone=info,tower_http=info")
            }),
        )
        .init();

    let state = AppState {
        catalog: Arc::new(Mutex::new(seed_catalog())),
        valid_api_key: Arc::new("secret123".to_string()),
    };

    // เส้นทางสาธารณะ: ไม่ต้อง auth
    let public_routes = Router::new()
        .route("/health", get(health))
        .route("/books", get(list_books))
        .route("/books/{id}", get(get_book));

    // เส้นทางที่ต้อง auth: layer เฉพาะกลุ่มนี้ก่อน merge เข้า router หลัก
    let protected_routes = Router::new()
        .route("/books/{id}/borrow", post(borrow_book))
        .layer(middleware::from_fn_with_state(state.clone(), toy_auth_middleware));

    let app = public_routes
        .merge(protected_routes)
        // ลำดับ .layer() ด้านล่างนี้ (ไล่จากบนลงล่าง = จากในสุดไปนอกสุด ตามกฎที่พิสูจน์ไว้ในหัวข้อ 65.6):
        // 1) timing_middleware (ในสุด — วัดเวลาเฉพาะงานจริงของ handler + auth ที่อยู่ในกว่า)
        // 2) CorsLayer (ตอบ preflight OPTIONS ได้จบในตัว ไม่ต้องพึ่ง timing/trace ชั้นในเลย)
        // 3) TraceLayer (นอกสุด — เห็น "ทุก" request แม้จะถูก auth ปฏิเสธไปแล้วก็ตาม)
        .layer(middleware::from_fn(timing_middleware))
        .layer(
            CorsLayer::new()
                .allow_origin("http://localhost:5173".parse::<HeaderValue>().unwrap())
                .allow_methods([Method::GET, Method::POST])
                .allow_headers([axum::http::header::CONTENT_TYPE, "x-api-key".parse().unwrap()]),
        )
        .layer(TraceLayer::new_for_http())
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3004").await.unwrap();
    tracing::info!("Library API ฟังอยู่ที่ http://127.0.0.1:3004");
    axum::serve(listener, app.into_make_service()).await.unwrap();
}
```

โค้ดนี้ผ่าน `cargo build` จริงโดยไม่มี warning เลยแม้แต่ตัวเดียว (ทดสอบด้วย axum 0.8.9 + tower-http 0.7.1)
มาทดสอบ end-to-end ด้วย `curl` จริงทีละ scenario:

**1) endpoint สาธารณะ ใช้ได้โดยไม่ต้อง auth:**

```bash
curl -sS -i http://127.0.0.1:3004/books
```
```
HTTP/1.1 200 OK
content-type: application/json
access-control-allow-origin: http://localhost:5173
content-length: 130

[{"id":1,"title":"The Rust Programming Language","available":true},{"id":2,"title":"Zero To Production In Rust","available":true}]
```

**2) CORS preflight สำหรับ endpoint ที่ต้อง auth (ยังตอบได้โดยไม่ต้องมี `x-api-key` เลย เพราะ `CorsLayer`
อยู่ในวงที่ตื้นกว่า `toy_auth_middleware` — preflight ถูกจัดการก่อนถึงชั้น auth):**

```bash
curl -sS -i -X OPTIONS http://127.0.0.1:3004/books/1/borrow \
  -H "Origin: http://localhost:5173" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: x-api-key"
```
```
HTTP/1.1 200 OK
access-control-allow-methods: GET,POST
access-control-allow-headers: content-type,x-api-key
access-control-allow-origin: http://localhost:5173
allow: POST
content-length: 0
```

**3) ยืมหนังสือโดยไม่มี `x-api-key` — ต้องถูกปฏิเสธด้วย `401` พร้อม body ที่มีเหตุผลชัดเจน:**

```bash
curl -sS -i -X POST http://127.0.0.1:3004/books/1/borrow
```
```
HTTP/1.1 401 Unauthorized
content-type: application/json
access-control-allow-origin: http://localhost:5173
content-length: 64

{"error":"unauthorized","reason":"missing or invalid x-api-key"}
```

**4) ยืมหนังสือด้วย `x-api-key` ที่ถูกต้อง — สำเร็จ พร้อมเห็นว่า `CurrentUser` ที่ middleware แนบเข้าไปถูก
ดึงมาใช้ใน response จริง (`borrowed_by`):**

```bash
curl -sS -i -X POST http://127.0.0.1:3004/books/1/borrow -H "x-api-key: secret123"
```
```
HTTP/1.1 200 OK
content-type: application/json
access-control-allow-origin: http://localhost:5173
content-length: 103

{"book":{"available":false,"id":1,"title":"The Rust Programming Language"},"borrowed_by":"phutjirakul"}
```

**5) ยืมซ้ำอีกครั้ง (หนังสือถูกยืมไปแล้ว) — ตรรกะทางธุรกิจปกติ ไม่เกี่ยวกับ middleware เลย:**

```bash
curl -sS -i -X POST http://127.0.0.1:3004/books/1/borrow -H "x-api-key: secret123"
```
```
HTTP/1.1 409 Conflict
book is already borrowed
```

และ log จริงที่ `TraceLayer`/`timing_middleware` ผลิตให้ตลอดทั้งการทดสอบ (คัดลอกตรงจากการรันจริง — เห็นครบ
ทุก request ตามลำดับที่ยิงไป รวมทั้งตัวที่ถูก auth ปฏิเสธด้วย เพราะ `timing_middleware` และ `TraceLayer`
อยู่**นอกกว่า** `toy_auth_middleware` เสมอ ตามที่ออกแบบไว้ในหัวข้อ 65.6):

```
2026-09-26T23:38:45.538509Z  INFO capstone: Library API ฟังอยู่ที่ http://127.0.0.1:3004
2026-09-26T23:38:46.545729Z  INFO capstone: timing_middleware: จบ request method=GET path=/health status=200 elapsed_us=71
2026-09-26T23:38:46.554320Z  INFO capstone: timing_middleware: จบ request method=GET path=/books status=200 elapsed_us=86
2026-09-26T23:38:46.562100Z  INFO capstone: timing_middleware: จบ request method=GET path=/books/1 status=200 elapsed_us=42
2026-09-26T23:38:46.577946Z  WARN capstone: toy_auth_middleware: ปฏิเสธคำขอ (x-api-key ไม่ถูกต้อง/ไม่มี)
2026-09-26T23:38:46.578016Z  INFO capstone: timing_middleware: จบ request method=POST path=/books/1/borrow status=401 elapsed_us=110
2026-09-26T23:38:46.586042Z  INFO capstone: ยืมหนังสือสำเร็จ book_id=1 borrower=phutjirakul
2026-09-26T23:38:46.586128Z  INFO capstone: timing_middleware: จบ request method=POST path=/books/1/borrow status=200 elapsed_us=152
2026-09-26T23:38:46.593539Z  INFO capstone: timing_middleware: จบ request method=POST path=/books/1/borrow status=409 elapsed_us=39
2026-09-26T23:38:46.601025Z  INFO capstone: timing_middleware: จบ request method=GET path=/books/999 status=404 elapsed_us=23
```

สังเกตบรรทัดที่ห้า (`status=401 elapsed_us=110`) — นี่คือหลักฐานที่จับต้องได้ว่า **request ที่ถูก
`toy_auth_middleware` ปฏิเสธยังถูก `timing_middleware` และ `TraceLayer` มองเห็นครบทุกครั้ง** เพราะทั้งสองอยู่
วงนอกกว่า auth — สอดคล้องกับสิ่งที่พิสูจน์ไว้ในหัวข้อ 65.6 ทุกประการ และเป็นการออกแบบที่ถูกต้องสำหรับ
production จริง: ทีม operations เห็น log ของ**ทุก**ความพยายามเข้าถึงระบบ ไม่ใช่แค่ที่ผ่าน auth สำเร็จ

## กับดักที่พบบ่อย (Common Pitfalls)

**1. ตั้ง `RUST_LOG=info` แล้วงงว่าทำไมไม่เห็น log จาก `TraceLayer` เลยแม้แต่บรรทัดเดียว**

นี่คือกับดักที่พบบ่อยที่สุดของบทนี้ และเชื่อมกับความเข้าใจเรื่อง log level จาก **Part 60** ตรง ๆ —
`TraceLayer::new_for_http()` (จาก `tower-http`) ใช้ **`tracing::Level::DEBUG`** เป็นค่าเริ่มต้านสำหรับ
ทั้ง span และ event ของมัน (`tower_http::trace::mod::DEFAULT_MESSAGE_LEVEL = Level::DEBUG` ใน source code
จริงของ `tower-http` 0.7.1) — ถ้าตั้ง `RUST_LOG="my_app=info,tower_http=info"` คุณจะเห็น log ของแอปตัวเอง
แต่**ไม่เห็นอะไรเลยจาก `TraceLayer`** เพราะ `info` สูงกว่า (กรองเข้มกว่า) `debug` วิธีแก้คือต้องระบุ
`tower_http=debug` ให้ตรง ๆ:

```bash
RUST_LOG="my_app=info,tower_http=debug" cargo run
```

หรือถ้าต้องการ log ระดับ `INFO` จาก `TraceLayer` แทน สามารถปรับ level เองได้ผ่าน `.make_span_with()`,
`.on_request()`, `.on_response()` ของ `TraceLayer` (เช่น `TraceLayer::new_for_http().on_response(
DefaultOnResponse::new().level(tracing::Level::INFO))`) แทนที่จะพึ่ง default

**2. เขียน middleware ด้วย `from_fn` แล้วพยายามใช้ `req` หลังจากเรียก `next.run(req)` ไปแล้ว — compiler ปฏิเสธ**

```rust
use axum::{extract::Request, middleware::Next, response::Response};

async fn broken_middleware(req: Request, next: Next) -> Response {
    let response = next.run(req).await;
    // พยายามอ่าน req.uri() หลังจากที่ req ถูก move เข้า next.run() ไปแล้ว
    println!("path was: {}", req.uri().path());
    response
}
```

error จริงจาก compiler:

```
error[E0382]: borrow of moved value: `req`
 --> src/bin/broken_middleware.rs:6:30
  |
3 | async fn broken_middleware(req: Request, next: Next) -> Response {
  |                            --- move occurs because `req` has type `axum::http::Request<Body>`, which does not implement the `Copy` trait
4 |     let response = next.run(req).await;
  |                             --- value moved here
5 |     // พยายามอ่าน req.uri() หลังจากที่ req ถูก move เข้า next.run() ไปแล้ว
6 |     println!("path was: {}", req.uri().path());
  |                              ^^^ value borrowed here after move

For more information about this error, try `rustc --explain E0382`.
```

`next.run(req)` รับ `req` แบบ **move** (ไม่ใช่ borrow) เพราะ request ต้องถูกส่งต่อไปทั้งก้อนให้ชั้นถัดไป
วิธีแก้ตามที่ทำมาตลอดบทนี้คือ **ดึงข้อมูลที่ต้องใช้ทีหลังออกมาเก็บไว้ในตัวแปรก่อน** (`.clone()` สำหรับ
`Method`/`Uri` ซึ่ง implement `Clone` ทั้งคู่ ราคาถูกมาก) ตั้งแต่ก่อนเรียก `next.run(req)`:

```rust
async fn fixed_middleware(req: axum::extract::Request, next: axum::middleware::Next) -> axum::response::Response {
    let path = req.uri().path().to_string(); // ดึงมาเก็บไว้ก่อน
    let response = next.run(req).await;
    println!("path was: {path}"); // ใช้ตัวแปรที่เก็บไว้ ไม่ใช่ req เดิม
    response
}
```

**3. ลืมว่า `Router::layer()` ที่เรียกซ้อนกันตรง ๆ ใช้กฎ "เพิ่มทีหลัง = ชั้นนอกสุด" — สลับกับกฎของ
`ServiceBuilder` ("เพิ่มก่อน = ชั้นนอกสุด") จนพฤติกรรมผิดจากที่คาดไว้**

หัวข้อ 65.6 พิสูจน์เรื่องนี้ไว้ละเอียดแล้วด้วยการรันจริง — สรุปสั้น ๆ ในเชิงกับดัก: ถ้าเขียน
`.layer(TraceLayer::new_for_http()).layer(middleware::from_fn(auth_gate))` โดยหวังว่า `TraceLayer` จะเห็น
ทุก request รวมถึงที่ auth ปฏิเสธ — **ผิดความคาดหมาย!** เพราะ `auth_gate` ถูกเพิ่ม**ทีหลัง** จึงกลายเป็นชั้น
นอกสุดจริง ๆ (ตรงข้ามกับที่ตั้งใจ) ทำให้ request ที่ถูก auth ปฏิเสธไม่ถูก trace เลย วิธีแก้ที่ปลอดภัยที่สุด
คือใช้ `tower::ServiceBuilder` ประกอบ layer ทั้งหมดเป็นก้อนเดียวก่อน ค่อย `.layer()` เข้า `Router` เพียงครั้ง
เดียว ตามที่แนะนำไว้ในหัวข้อ 65.6 (`ServiceBuilder` ใช้กฎ "เพิ่มก่อน = ชั้นนอกสุด" ซึ่งตรงกับสัญชาตญาณการ
อ่านโค้ดจากบนลงล่างมากกว่า)

**4. ใช้ `.layer()` กับ middleware ที่ปฏิเสธคำขอได้ (เช่น auth) แล้วงงว่าทำไม path ที่พิมพ์ผิดกลายเป็น
`401` แทนที่จะเป็น `404` ตามที่ควรจะเป็น**

พิสูจน์ไว้แล้วในหัวข้อ 65.7 ด้วยการรันจริงทั้งสองโหมด — ถ้า middleware มีลักษณะปฏิเสธคำขอได้ (return early)
และคุณไม่ต้องการให้มันกลืน `404` ของ path ที่ไม่มีอยู่จริงให้กลายเป็น status code ของมันเอง ต้องใช้
**`.route_layer()`** แทน `.layer()` ธรรมดา — `.route_layer()` มีเงื่อนไขเพิ่มที่ต้องรู้ด้วย: **มันจะ panic
ทันทีถ้าเรียกตอนที่ `Router` ยังไม่มี route ไหนเลย** (เพราะถ้าไม่มี route ก็ไม่มีอะไรให้ middleware นี้ครอบ
เลย ถือเป็น bug ในโค้ด) — error message จริงเมื่อเกิดกรณีนี้:

```
thread 'main' panicked at src/bin/snippet_check4.rs:6:38:
Adding a route_layer before any routes is a no-op. Add the routes you want the layer to apply to first.
```

ต้องเรียก `.route(...)` เพิ่ม route ให้ครบก่อน แล้วค่อยเรียก `.route_layer(...)` ปิดท้ายเสมอ

**5. ใส่ `RequestBodyLimitLayer` แล้วงงว่าทำไม limit ไม่ทำงานตามที่ตั้งไว้ (ยังรับ body ใหญ่กว่าที่กำหนด
ได้อยู่ หรือ error message ดูไม่ตรงกับที่คาด)**

Axum เองมี **`DefaultBodyLimit`** ติดมาให้ทุก `Router` อยู่แล้วโดยอัตโนมัติ (ค่าเริ่มต้น 2 MB) ถ้าคุณเพิ่ม
`RequestBodyLimitLayer::new(1024)` เข้าไปโดย**ไม่ปิด**ของเดิมก่อน จะมีสอง limit ทำงานซ้อนกัน — ถ้า limit ที่
ตั้งใหม่**สูงกว่า**ของเดิม (2 MB) คุณจะเห็น request ถูกตัดที่ 2 MB (ของ `DefaultBodyLimit`) ไม่ใช่ที่ตัวเลข
ใหม่ที่ตั้งไว้ ทำให้ดูเหมือน "limit ไม่ทำงาน" ทั้งที่จริง ๆ มันทำงานอยู่ แค่ตัวที่ทำงานคือตัวเดิม วิธีแก้คือ
เรียก `.layer(DefaultBodyLimit::disable())` ก่อนเสมอเมื่อต้องการใช้ `RequestBodyLimitLayer` ควบคุมเองแทน
(ดังที่ทำไว้ในตัวอย่างหัวข้อ 65.5)

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เพิ่ม `TraceLayer::new_for_http()` เข้า `Router` ของระบบห้องสมุดในหัวขัวข้อ capstone ด้วย
   `.on_response()` ที่ปรับ level เป็น `tracing::Level::INFO` แทนค่าเริ่มต้น (`DEBUG`) แล้วรันด้วย
   `RUST_LOG="capstone=info,tower_http=info"` (ไม่ต้องพิเศษ `debug` แล้ว) พิสูจน์ว่าเห็น log จาก
   `on_response` ที่ระดับ `INFO` ได้จริง
   - Hint: ใช้ `TraceLayer::new_for_http().on_response(tower_http::trace::DefaultOnResponse::new()
     .level(tracing::Level::INFO))`

2. **(กลาง)** เขียน custom middleware ชื่อ `request_id_middleware` ที่สร้าง UUID/สุ่มตัวเลขสั้น ๆ เป็น
   "request id" ให้ทุก request ที่เข้ามา แนบเข้า `Extension<RequestId>` ให้ handler ปลายทางดึงไปใช้ log
   ต่อได้ (เชื่อมกับ Part 60 เรื่อง "แยกแยะ log จากหลาย request ที่ทำงานพร้อมกัน") แล้วพิสูจน์ด้วยการยิง
   สอง request พร้อมกัน (เช่นด้วย `curl` สองคำสั่งคู่กันหรือ `ab`/`hey`) ว่า request id ที่ log ออกมาไม่ซ้ำกัน
   - Hint: ใช้ `AtomicU64` ง่าย ๆ (เหมือน `request_counter` ในหัวข้อ 65.10) แทน UUID จริงก็ได้ถ้ายังไม่ได้
     เรียน crate `uuid`

3. **(ยาก)** ตั้งค่า `RequestBodyLimitLayer` และ `TimeoutLayer` ให้กับกลุ่ม route เดียวใน library API
   (เฉพาะ `POST /books/{id}/borrow`) โดย**ไม่กระทบ** route อื่น ๆ ในระบบเลย (route อื่นยังรับ body ขนาดใหญ่
   และไม่ timeout ได้ตามปกติ) — ต้องใช้เทคนิคการแยก sub-router ตามหัวข้อ 65.8 ร่วมกับสิ่งที่เรียนในหัวข้อ
   65.5 พิสูจน์ด้วย `curl` ว่า route ที่ไม่ได้อยู่ในกลุ่มนี้ยังรับ body ใหญ่ได้ตามปกติ
   - Hint: สร้าง `Router` ย่อยสำหรับ `/books/{id}/borrow` เพียงตัวเดียว ใส่ layer เฉพาะกลุ่มนั้นก่อน
     `.merge()` เข้า router หลัก เหมือนที่ `protected_routes` ทำกับ auth middleware

4. **(ยากมาก / ประยุกต์จริง)** เขียน **rate limiting middleware** อย่างง่ายด้วยมือเอง (ไม่ต้องใช้ crate
   สำเร็จรูป) ผ่าน `from_fn_with_state` ที่จำกัดจำนวน request ต่อ `x-api-key` ไม่ให้เกิน N ครั้งต่อวินาที
   (เก็บ state ด้วย `Arc<Mutex<HashMap<String, (Instant, u32)>>>` ใน `AppState`) — ถ้าเกิน limit ให้คืน
   `429 Too Many Requests` พร้อม header `Retry-After` พิสูจน์ด้วยการยิง `curl` ถี่ ๆ ในลูป `for` ของ bash
   แล้วนับว่ามีกี่ request ที่ได้ `200` กับกี่ request ที่ได้ `429` จริง ต้องตรงกับ limit ที่ตั้งไว้
   - Hint: การจัดการ concurrent access กับ `HashMap` ที่แชร์ต้องระวังเรื่อง lock contention — ทบทวน
     `Mutex` จาก Part 50 ถ้าจำเป็น; ระวังเรื่อง "sliding window" กับ "fixed window" ให้ชัดว่าอัลกอริทึมไหน
     ที่กำลังทำอยู่ เพราะมีผลต่อผลลัพธ์ตอนทดสอบ

## สรุป

บทนี้พาไปดู middleware ของ Axum ในระดับที่ลึกกว่าการ "copy โค้ดมาแล้วใช้" — เริ่มจากเหตุผลว่าทำไมต้องมี
middleware เลย (แยก cross-cutting concern ออกจากตรรกะทางธุรกิจ ตามหลักการเดียวกับที่ Part 30-31 สอนเรื่อง
error handling), ลงไปถึงรากฐานจริงของระบบ (`tower::Service`/`tower::Layer`) ที่พิสูจน์ได้จาก source code ของ
Axum เองว่า `Router` คือ `Service` ตัวหนึ่ง และ middleware ทุกตัวคือการห่อ `Service` ให้กลายเป็น `Service`
ใหม่ที่ทำงานมากขึ้นตามโมเดล "ชั้นหัวหอม" — จากนั้นใช้งาน middleware สำเร็จรูปจาก `tower-http` จริง
(`TraceLayer`, `CorsLayer`, `CompressionLayer`, `TimeoutLayer`, `RequestBodyLimitLayer`) พร้อมพิสูจน์
พฤติกรรมทุกตัวด้วยการรันเซิร์ฟเวอร์จริงและยิง `curl` จริง — และที่สำคัญที่สุดคือพิสูจน์ (ไม่ใช่แค่บอก) ว่า
**ลำดับการ `.layer()` มีผลจริงต่อ observable behavior** ทั้งในแง่ที่ auth-outer ทำให้ request ที่ถูกปฏิเสธ
หายไปจาก log และในแง่ที่ `.layer()` เทียบกับ `.route_layer()` ทำให้ `404` กลายเป็น `401` ได้หรือไม่ก็ได้
สุดท้ายเขียน custom middleware ของตัวเองด้วย `axum::middleware::from_fn`/`from_fn_with_state` ครบทั้งแบบ
จับเวลา, แบบ short-circuit ด้วย auth, และแบบแนบข้อมูลเข้า request ผ่าน `Extension` ให้ handler ปลายทางใช้
ต่อ ปิดท้ายด้วย capstone ที่รวมทุกอย่างเข้ากับระบบห้องสมุดจาก Part 63-64 พร้อมทดสอบ end-to-end จริงครบทุก
เส้นทาง

middleware ที่เขียนในบทนี้ยังจงใจปฏิเสธ request ด้วยการคืน `Response` ตรง ๆ เสมอ ไม่เคยปล่อย `Err` ลอย ๆ
ออกมา — นั่นเป็นเพราะกฎที่ `Router` กำหนดไว้ (`Error: Into<Infallible>`) แต่ในระบบที่ซับซ้อนขึ้น การแปลง
error ประเภทต่าง ๆ (จาก database, จาก extractor ที่ rejection, จาก validation) ให้กลายเป็น response ที่
ถูกต้องและสอดคล้องกันทั้งระบบ ต้องมีสถาปัตยกรรมที่เป็นระบบมากกว่านี้ — นั่นคือสิ่งที่ **Part 66: Axum: Error
Handling แบบมืออาชีพ** จะพาไปดูต่อ ครอบคลุมทั้ง custom error type ที่ implement `IntoResponse`, การจัดการ
extractor rejection, และ `HandleErrorLayer` ที่แนะนำไว้สั้น ๆ ในหัวข้อ 65.12 อย่างละเอียดครบถ้วน

---

**Part ก่อนหน้า:** [Axum: State Management และ Extractors](part-064-axum-state-extractors.md) | **Part ถัดไป:** [Axum: Error Handling แบบมืออาชีพ](part-066-axum-error-handling.md)
