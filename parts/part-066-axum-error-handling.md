# Part 66: Axum: Error Handling แบบมืออาชีพ

> โมดูล: การพัฒนาเว็บแอปพลิเคชัน (Web Development) | ระดับ: สูง | เวลาโดยประมาณ: 260 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้อย่างเจาะจงว่าทำไมการคืน `(StatusCode, String)` หรือ `impl IntoResponse` แยกกันคนละแบบในแต่ละ
  handler (ที่ Part 63-65 ใช้เป็นทางลัดเพื่อโฟกัสหัวข้ออื่น) ถึง **เอาไม่อยู่** เมื่อ API โตขึ้นจริง — เห็นรูปแบบ
  ที่ไม่สอดคล้องกันจริงจากโค้ดของ Part 65 เอง แล้วออกแบบทางแก้ที่เป็นระบบ ไม่ใช่แค่ "เขียนให้สั้นลง"
- ออกแบบ **`AppError`** — enum เดียวที่เป็นจุดศูนย์กลางของ error ทั้งแอป — ด้วย `#[derive(thiserror::Error)]`
  ต่อยอดจากหลักการ custom error type/`From`/`Into`/error chain ที่ Part 30 สอนไว้ และ `thiserror`/`anyhow`
  ที่ Part 31 สอนไว้ นำมาประยุกต์ใช้ในบริบทของเว็บเป็นครั้งแรกอย่างเต็มรูปแบบ
- implement `impl IntoResponse for AppError` ครั้งเดียว ให้ error ทุกชนิดในระบบตอบกลับเป็น **JSON shape
  เดียวกันเสมอ** (`{"error": {"code": "...", "message": "..."}}`) พร้อม status code ที่ถูกต้องตามชนิด error
  — พิสูจน์ด้วย `curl` จริงทุก variant
- เขียน handler ที่ chain การทำงานที่ fail ได้หลายจุดด้วย `?` เพียงตัวเดียว (การหา resource ไม่เจอ, กฎธุรกิจ
  ขัดกัน, ระบบภายนอกล่ม) แล้วให้ทุกจุดแปลงเป็น `AppError` โดยอัตโนมัติผ่าน `#[from]` — ต่อยอด `?` operator จาก
  Part 12 ให้ไหลตลอดทางจนถึง HTTP response จริง
- ทำ **logging error แบบรวมศูนย์จุดเดียว** ด้วย `tracing::error!`/`tracing::warn!` ภายใน `IntoResponse` พร้อม
  request id ที่ไหลผ่าน tracing span (ต่อยอด Part 60 และ Part 64-65) และพิสูจน์ได้ว่า **client เห็นข้อความ
  ปลอดภัยทั่วไปเสมอ ในขณะที่ log เก็บรายละเอียดเต็มไว้** — หลักการรักษาความปลอดภัยที่สำคัญที่สุดข้อหนึ่งของบทนี้
- จัดการ **extractor rejection** (`Json`/`Path` ที่ extract ไม่ผ่าน) ให้ตอบเป็น JSON shape เดียวกับ error อื่น
  ทั้งระบบ (ไม่ใช่ plain text ตามค่า default ของ Axum) และรู้พฤติกรรมจริงของ panic ใน Axum ทั้งแบบที่ไม่มีและมี
  `tower_http::catch_panic::CatchPanicLayer` ครอบ — พิสูจน์ด้วยการรันจริงทั้งสองแบบ ไม่ใช่แค่ท่องจำ

## ความรู้ที่ต้องมีมาก่อน

- **Part 12 (Result และ Error Handling เบื้องต้น)**: บทนี้ใช้ `?` operator และกลไก desugar ของมัน
  (`return Err(From::from(error))`) ตลอดทั้งบทแบบเดียวกับที่ Part 12 ปูพื้นไว้ — ถ้าคุณลืมกลไกนี้ ควรกลับไปทวน
  ก่อน เพราะบทนี้ไม่อธิบายกลไก `?` ใหม่จากศูนย์อีก
- **Part 30 (Error Handling ขั้นสูง)**: `AppError` ในบทนี้คือ custom error enum แบบเดียวกับที่ Part 30 สอนให้
  สร้างด้วยมือทุกประการ (`impl Display`, `impl std::error::Error`, `impl From` หลายตัว, error chain ผ่าน
  `source()`) เพียงแต่บทนี้ให้ `thiserror` เขียนให้แทน และเพิ่มมิติใหม่ที่ Part 30 ยังไม่มี: การแปลง error
  enum ให้กลายเป็น HTTP response ที่ถูกต้อง ผ่าน `IntoResponse`
- **Part 31 (thiserror และ anyhow ในโปรเจกต์จริง)**: บทนี้ใช้ `#[derive(thiserror::Error)]`, `#[error("...")]`,
  `#[from]` ตรงตามที่ Part 31 สอนไว้ทุกประการ และใช้ `anyhow::Error`/`.context()` สำหรับ error ที่ไม่ต้องแยก
  กรณี (เช่น ความล้มเหลวของระบบภายนอกที่ handler ทำอะไรไม่ได้นอกจาก log แล้วตอบ 500) — ถ้ายังไม่แน่นเรื่อง
  `#[from]` implies `#[source]`, หรือความต่างระหว่าง `thiserror`/`anyhow`, ควรทวน Part 31 ก่อน
- **Part 60 (Logging และ Tracing เบื้องต้น)**: การ log ข้อความแบบ structured ด้วย `tracing::error!`/`warn!`
  พร้อม field (`error_code = ..., status = ...`) และแนวคิด span ที่ครอบ event หลายตัวไว้ด้วยกัน คือแกนหลักของ
  หัวข้อ 66.5 — บทนี้จะไม่อธิบาย `tracing` จากศูนย์อีก
- **Part 61 (HTTP Fundamentals และ REST API Concepts)**: บทนี้อ้างอิง status code (`400`, `401`, `404`,
  `409`, `500`) ตามความหมายมาตรฐานที่ Part 61 สอนไว้ตรง ๆ
- **Part 62-65 (Axum ทั้งชุด)**: บทนี้เป็น**ภาคปิด**ของชุด Axum ต่อจาก Part 62 (พื้นฐาน Axum), Part 63
  (routing/handlers/`IntoResponse` เบื้องต้น), Part 64 (`State`/extractors/`Extension`), และ Part 65
  (middleware ด้วย `tower`/`tower-http`) — capstone ท้ายบทนี้เอาระบบห้องสมุด (library API) ที่ Part 63-65
  วางโครงไว้มาปรับปรุง error handling ทั้งระบบให้เป็นมืออาชีพ คุณต้องสร้าง `Router`, ใช้ `State<T>`,
  `Path<T>`, `Json<T>`, และเขียน middleware ด้วย `axum::middleware::from_fn` ได้แล้วก่อนเริ่มบทนี้

## เนื้อหา

### 66.1 ทวนปัญหา: ทำไม error handling แบบ ad-hoc ต่อ handler ถึงเอาไม่อยู่

ตลอด Part 63-65 เราจงใจโฟกัสไปที่หัวข้อหลักของแต่ละบท (routing, state, middleware) และใช้ error handling แบบ
"เขียนพอให้ใช้งานได้" ในแต่ละ handler — ลองย้อนดูสิ่งที่เกิดขึ้นจริงถ้าเรารวมโค้ดจากสามบทนั้นเข้าด้วยกันในระบบ
เดียว (ตัดมาจาก Part 63 หัวข้อ 63.9 และ Part 65 หัวข้อ capstone ตรง ๆ ไม่ได้แต่งเติม):

**จาก Part 63 (`ApiError` ของระบบห้องสมุด):**

```rust
# use axum::{http::StatusCode, response::{IntoResponse, Response}, Json};
struct ApiError {
    status: StatusCode,
    message: String,
}

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        let body = Json(serde_json::json!({ "error": self.message }));
        (self.status, body).into_response()
    }
}
```

**จาก Part 64 (`ApiKeyRejection` ของ custom extractor):**

```rust
# use axum::{http::StatusCode, response::{IntoResponse, Response}, Json};
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
        (status, Json(serde_json::json!({ "error": message }))).into_response()
    }
}
```

**จาก Part 65 (`toy_auth_middleware` และ `borrow_book` ในระบบห้องสมุดเดียวกัน):**

```rust
# use axum::{http::StatusCode, response::{IntoResponse, Response}, Json};
# fn placeholder() -> Response {
// ปฏิเสธเพราะ auth ไม่ผ่าน -- ตอบเป็น JSON
(
    StatusCode::UNAUTHORIZED,
    Json(serde_json::json!({ "error": "unauthorized", "reason": "missing or invalid x-api-key" })),
)
    .into_response()
# }
# fn placeholder2() -> Response {
// แต่ตรรกะทางธุรกิจใน handler เดียวกันของระบบเดียวกัน -- ตอบเป็น plain text ล้วน ๆ!
(StatusCode::CONFLICT, "book is already borrowed").into_response()
# }
```

ลองนึกภาพทีมหน้าบ้าน (frontend) ที่ต้อง parse response error จาก API นี้ — endpoint หนึ่งได้
`{"error": "..."}`, อีก endpoint ได้ `{"error": {"code": "...", "message": "..."}}` (ถ้าเราผสมทั้งสามแบบ
เข้าด้วยกัน), และอีก endpoint ได้ **plain text ธรรมดา ไม่ใช่ JSON เลย** (`"book is already borrowed"`) ทั้งที่
`Content-Type` อื่น ๆ ของ API เดียวกันเป็น `application/json` หมด — ฝั่ง frontend ที่เขียน `response.json()`
แบบเดียวกันทุก endpoint (ซึ่งเป็นเรื่องปกติที่ทุกทีมทำ) **จะพังทันทีที่เจอ endpoint ที่ตอบ plain text** เพราะ
`JSON.parse("book is already borrowed")` throw exception ทันที

นี่ไม่ใช่แค่เรื่อง "ดูไม่เรียบร้อย" — มันคือปัญหาเชิงโครงสร้างสามข้อที่นับวันจะแย่ลงเมื่อ API โตขึ้น:

1. **รูปแบบ JSON ของ error ไม่สอดคล้องกันข้าม endpoint** — ตามที่เห็นข้างบน แต่ละคนที่เขียน handler ใหม่ต้อง
   "จำ" ว่ารูปแบบที่ใช้ในระบบคืออะไร (ถ้ามีการตกลงกันไว้เลย) แล้ว copy-paste มาเขียนใหม่ทุกครั้ง — เหมือนกับ
   ปัญหา DRY ที่ Part 65 หัวข้อ 65.1 อธิบายไว้เรื่อง middleware ทุกประการ แต่คราวนี้เกิดกับ error shape แทน
2. **ไม่มีจุดกลางสำหรับ log error** — `ApiError`, `ApiKeyRejection`, และ error แบบ inline ใน `borrow_book`
   ต่างคนต่างมี `impl IntoResponse` ของตัวเอง ไม่มีใครรับผิดชอบการ log แบบสม่ำเสมอ ถ้าอยากรู้ว่า "500 เกิดขึ้น
   กี่ครั้งในชั่วโมงที่แล้ว จากสาเหตุอะไร" จะไม่มีที่เดียวให้ไปดู ต้องไล่หา `println!`/`tracing::` ที่กระจัด
   กระจายอยู่ในทุก handler
3. **ไม่มีทางให้ client (โดยเฉพาะ frontend) แยกแยะสาเหตุของ error ได้อย่างเป็นระบบ** — `{"error": "book is
   already borrowed"}` เป็น string ที่มนุษย์อ่านได้ก็จริง แต่ frontend ที่ต้องการ "ถ้า error เป็นเรื่อง
   conflict ให้ขึ้น modal สีเหลือง ถ้าเป็นเรื่อง validation ให้ underline field ที่ผิด" จะทำอะไรไม่ได้เลย
   นอกจาก `if message.contains("already borrowed")` ซึ่งพังทันทีที่มีคนแก้ข้อความ (ปัญหาเดียวกับที่ Part 12
   หัวข้อ 12.3 เตือนไว้เรื่อง error ที่เป็น string ล้วน ๆ ตรวจสอบด้วย pattern matching ไม่ได้)

คำตอบของทั้งสามข้อนี้คือหลักการเดียวกันเป๊ะกับที่ Part 30 ใช้ตอนออกแบบ `AppError`/`ConfigError`: **รวม error
ทุกชนิดของแอปให้เป็น enum เดียว ที่มีจุดแปลงเป็น response แค่จุดเดียว** เพียงแต่คราวนี้ "แปลง" หมายถึงแปลงเป็น
`Response` ของ Axum ไม่ใช่แค่ `Display` — บทนี้ทั้งบทคือการสร้างสิ่งนั้นให้สมบูรณ์

### 66.2 ออกแบบ `AppError`: enum เดียวสำหรับ error ทั้งแอป

ก่อนเริ่ม เพิ่ม dependency ที่ต้องใช้ทั้งบทนี้เข้าโปรเจกต์ครั้งเดียว (เวอร์ชันจริงที่ทดสอบทั้งบทนี้ คือ axum
0.8.9 — ตัวเดียวกับ Part 62-65, บวก `thiserror`/`anyhow` เวอร์ชันเดียวกับ Part 31):

```toml
[dependencies]
axum = { version = "0.8.9", features = ["macros"] }
tokio = { version = "1.53.1", features = ["full"] }
tower-http = { version = "0.7.1", features = ["catch-panic"] }
serde = { version = "1.0.229", features = ["derive"] }
serde_json = "1.0.151"
thiserror = "2.0.21"
anyhow = "1.0.104"
tracing = "0.1.44"
tracing-subscriber = { version = "0.3.23", features = ["env-filter"] }
```

สังเกตว่า `axum` เปิด feature `macros` เพิ่มจากเดิม — เหตุผลจะเห็นในหัวข้อ 66.7 (derive macro
`#[derive(FromRequest)]`) และ 66.8 (`#[axum::debug_handler]`) และ `tower-http` เปิด feature `catch-panic`
ใหม่ (ต่างจาก Part 65 ที่เปิด `trace`/`cors`/`timeout`/`compression-gzip`/`limit`) สำหรับหัวข้อ 66.9

ตอนนี้มาออกแบบ `AppError` — ให้ตั้งคำถามก่อนว่า **สาเหตุความล้มเหลวที่เกิดขึ้นจริงในระบบ REST API มีกี่แบบ
กว้าง ๆ** คำตอบที่ครอบคลุมสถานการณ์ส่วนใหญ่ของ API ทั่วไป (ต่อยอดจาก status code ที่ Part 61 สอนไว้) คือ:

| สถานการณ์ | HTTP status ที่ควรตอบ | ตัวอย่าง |
|---|---|---|
| ไม่พบ resource ที่ขอ | `404 Not Found` | `GET /books/999` ที่ไม่มีหนังสือ id 999 |
| input ผิดรูปแบบ/ผิดเงื่อนไข | `400 Bad Request` | `title` เป็นค่าว่าง, `isbn` ไม่ครบ 13 หลัก |
| ขัดแย้งกับสถานะปัจจุบันของระบบ | `409 Conflict` | ยืมหนังสือที่ถูกยืมไปแล้ว |
| ไม่ได้รับอนุญาต | `401 Unauthorized` | ไม่มี/ผิด API key |
| เกิดข้อผิดพลาดที่ handler ควบคุมไม่ได้ | `500 Internal Server Error` | database ล่ม, service ภายนอกไม่ตอบ |

ห้าแถวนี้แปลงตรงเป็นห้า variant ของ `AppError` ได้เลย — เขียนด้วย `thiserror` ตามที่ Part 31 สอนไว้:

```rust
use serde::Serialize;
use thiserror::Error;

// field-level validation error หนึ่งจุด (ใช้ในหัวข้อ 66.6) -- ต้อง Serialize เพราะจะถูกส่งกลับเป็น JSON ตรงๆ
#[derive(Debug, Clone, Serialize)]
struct FieldError {
    field: String,
    message: String,
}

#[derive(Debug, Error)]
enum AppError {
    #[error("ไม่พบข้อมูลที่ต้องการ")]
    NotFound,

    #[error("ข้อมูลไม่ถูกต้อง: {0}")]
    Validation(String),

    #[error("ข้อมูลไม่ผ่านการตรวจสอบ")]
    ValidationErrors(Vec<FieldError>),

    #[error("ขัดแย้งกับสถานะปัจจุบัน: {0}")]
    Conflict(String),

    #[error("ไม่ได้รับอนุญาต")]
    Unauthorized,

    // #[from] ให้ ? แปลง anyhow::Error -> AppError โดยอัตโนมัติทุกจุดในโปรแกรม (ตาม Part 31 หัวข้อ 31.4)
    // และ implies #[source] ให้ด้วย -- error chain ของ anyhow ยังไล่ต่อได้เต็มรูปแบบผ่าน source()
    #[error("เกิดข้อผิดพลาดภายในระบบ")]
    Internal(#[from] anyhow::Error),
}
```

**อธิบายการออกแบบทีละ variant และเหตุผลที่มันเป็นแบบนี้:**

- **`NotFound`** — ไม่มี field เลย เพราะข้อความ "ไม่พบข้อมูลที่ต้องการ" ใช้ได้กับทุกกรณีที่หาไม่เจอ (หนังสือ,
  user, order ฯลฯ) โดยไม่ต้องรู้รายละเอียด — ถ้าต้องการข้อความที่เจาะจงกว่านี้ (เช่น "ไม่พบหนังสือ id 999")
  สามารถเปลี่ยนเป็น `NotFound(String)` ได้ แต่ต้องระมัดระวังเรื่องการรั่วข้อมูลภายใน (จะพูดถึงในหัวข้อ 66.5)
- **`Validation(String)`** เทียบกับ **`ValidationErrors(Vec<FieldError>)`** — นี่คือจุดที่ตั้งใจแยกสอง variant
  ออกจากกันทั้งที่สถานะ HTTP เหมือนกัน (`400 Bad Request`) เพราะ**การใช้งานต่างกัน**: `Validation(String)` ใช้
  กับข้อผิดพลาดที่เป็น "ข้อความเดียวพอสื่อความหมาย" (เช่น extractor rejection จากหัวข้อ 66.7 ที่ Axum เองให้
  ข้อความมาเป็น string อยู่แล้ว) ส่วน `ValidationErrors(Vec<FieldError>)` ใช้กับ **การตรวจสอบ input ที่ผู้ใช้
  ส่งมาแบบ field-by-field** ที่ frontend จำเป็นต้องรู้ว่า field ไหนผิดเพื่อ highlight ให้ผู้ใช้แก้ (หัวข้อ 66.6)
  — ทั้งสอง variant นี้คือตัวอย่างที่ดีของหลักการจาก Part 30 หัวข้อ 30.3 เรื่อง "structured data ที่ผู้เรียก
  ตรวจสอบได้ ไม่ใช่ string เดียวที่บอกทุกอย่างปนกัน" แม้จะอยู่ใน HTTP status เดียวกันก็ตาม
- **`Conflict(String)`** — เก็บข้อความอธิบายไว้เป็น `String` เพราะสาเหตุของ conflict เปลี่ยนไปตามบริบท
  (ยืมซ้ำ, อีเมลซ้ำ, สถานะไม่ตรงเงื่อนไข ฯลฯ) แต่ทั้งหมดยัง**คืนสถานะ HTTP เดียวกัน** (`409`) — เหมาะกับการ
  ให้ caller ระบุข้อความตอนสร้าง error ตรง ๆ เหมือนที่ `write!()` ใน Part 30 อธิบายไว้
- **`Unauthorized`** — ไม่มี field เพราะในระบบส่วนใหญ่ ข้อความ "ไม่ได้รับอนุญาต" เพียงพอแล้ว (ไม่ควรบอกเหตุผล
  ละเอียดว่า "username ถูกแต่ password ผิด" เพราะข้อมูลนั้นช่วย attacker เดา credential ได้ง่ายขึ้น — หลักการ
  security ที่จะพูดถึงอีกครั้งในหัวข้อ 66.5)
- **`Internal(#[from] anyhow::Error)`** — นี่คือ variant ที่เชื่อม `AppError` (ใช้เมื่อผู้เรียกต้องแยกกรณี
  — ตามหลัก thiserror จาก Part 31 หัวข้อ 31.8) เข้ากับ `anyhow::Error` (ใช้เมื่อไม่ต้องแยกกรณี เพราะ handler
  ทำอะไรกับมันไม่ได้นอกจาก log แล้วตอบ 500 อยู่ดี) **ทั้งสอง philosophy อยู่ร่วมกันในระบบเดียวได้จริง** ตามที่
  Part 31 หัวข้อ 31.8 ทิ้งท้ายไว้ว่าโปรเจกต์จริงมักใช้ทั้งคู่ — `#[from]` ทำให้ `?` แปลง `anyhow::Error` ใดก็ตาม
  (ซึ่งตัวมันเองรับ error ได้ทุกชนิดที่ implement `std::error::Error` ตามที่ Part 31 หัวข้อ 31.10 อธิบายไว้)
  ให้กลายเป็น `AppError::Internal` โดยอัตโนมัติทันที — นี่คือ "ทางระบาย" สุดท้ายสำหรับความล้มเหลวที่ไม่คาดคิด
  ที่คุณไม่อยากเขียน variant เจาะจงให้ทุกสาเหตุที่เป็นไปได้ในจักรวาล

สังเกตว่า **`AppError` ยังไม่ได้ implement `IntoResponse` เลย** ณ จุดนี้ — มันเป็นแค่ error type ธรรมดาที่
Part 30-31 สอนไว้ทุกประการ (ยัง match ได้, ยัง log ผ่าน `{}`/`{:?}` ได้) หัวข้อถัดไปคือขั้นที่ทำให้มันกลายเป็น
"error type ที่ Axum เข้าใจ"

### 66.3 `impl IntoResponse for AppError`: JSON shape เดียวกันทั้งระบบ

Part 63 หัวข้อ 63.8 แนะนำไว้แล้วว่า `IntoResponse` คือ trait ที่ Axum ใช้แปลง "อะไรก็ตามที่ handler คืน" ให้
กลายเป็น `Response` จริง — สิ่งที่บทนี้ทำคือ implement มันให้ `AppError` **ครั้งเดียว** แล้วทุก handler ที่คืน
`Result<T, AppError>` จะได้ error response ที่ถูกต้องโดยอัตโนมัติทันที ไม่ต้องเขียนโค้ดแปลงซ้ำเลยสักจุด:

```rust
use axum::{http::StatusCode, response::{IntoResponse, Response}, Json};
use serde_json::json;
# use serde::Serialize;
# use thiserror::Error;
# #[derive(Debug, Clone, Serialize)]
# struct FieldError { field: String, message: String }
# #[derive(Debug, Error)]
# enum AppError {
#     #[error("ไม่พบข้อมูลที่ต้องการ")]
#     NotFound,
#     #[error("ข้อมูลไม่ถูกต้อง: {0}")]
#     Validation(String),
#     #[error("ข้อมูลไม่ผ่านการตรวจสอบ")]
#     ValidationErrors(Vec<FieldError>),
#     #[error("ขัดแย้งกับสถานะปัจจุบัน: {0}")]
#     Conflict(String),
#     #[error("ไม่ได้รับอนุญาต")]
#     Unauthorized,
#     #[error("เกิดข้อผิดพลาดภายในระบบ")]
#     Internal(#[from] anyhow::Error),
# }

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        // ขั้นที่ 1: จับคู่ variant กับ (status code, error code สั้นๆ สำหรับ client ใช้แยกกรณี)
        let (status, code): (StatusCode, &str) = match &self {
            AppError::NotFound => (StatusCode::NOT_FOUND, "NOT_FOUND"),
            AppError::Validation(_) => (StatusCode::BAD_REQUEST, "VALIDATION_ERROR"),
            AppError::ValidationErrors(_) => (StatusCode::BAD_REQUEST, "VALIDATION_ERROR"),
            AppError::Conflict(_) => (StatusCode::CONFLICT, "CONFLICT"),
            AppError::Unauthorized => (StatusCode::UNAUTHORIZED, "UNAUTHORIZED"),
            AppError::Internal(_) => (StatusCode::INTERNAL_SERVER_ERROR, "INTERNAL_ERROR"),
        };

        // ขั้นที่ 2: สร้าง body -- ValidationErrors ได้ field "fields" เพิ่ม ตัวอื่นได้ "message" ธรรมดา
        // (self ถูก consume ตรงนี้ทีเดียว ไม่ต้อง borrow ต่อแล้ว เพราะ match ก่อนหน้าจับคู่ผ่าน &self ไปแล้ว)
        let body = match self {
            AppError::ValidationErrors(fields) => json!({
                "error": { "code": code, "message": "ข้อมูลไม่ผ่านการตรวจสอบ", "fields": fields }
            }),
            other => json!({
                "error": { "code": code, "message": other.to_string() }
            }),
        };

        (status, Json(body)).into_response()
    }
}
```

**อธิบายกลไกทีละส่วน:**

- **match สองรอบ** — รอบแรก (`match &self`) จับคู่ variant กับ `(StatusCode, &str)` โดย**ยืม** `self` เท่านั้น
  (ไม่ move) เพราะเรายังต้องใช้ `self` อีกครั้งในรอบที่สอง รอบที่สอง (`match self`) ค่อย **consume** `self`
  จริง เพื่อดึง `Vec<FieldError>` ออกมาแบบ owned (ไม่ต้อง clone) สำหรับ variant `ValidationErrors` — pattern
  "match ผ่าน reference ก่อน แล้ว match ผ่าน owned value ทีหลัง" นี้เป็นเทคนิคที่ใช้บ่อยเมื่อ enum มีข้อมูล
  ที่ทั้งต้อง "อ่านอย่างเดียว" (สำหรับเลือก status) และ "ย้ายออกมาใช้จริง" (สำหรับสร้าง body) อยู่ในตัวเดียวกัน
- **`other.to_string()`** — เรียก `Display` ที่ `thiserror` generate ให้ (ตาม `#[error("...")]` ของแต่ละ
  variant) — สังเกตว่า `AppError::Internal` มีข้อความคงที่ `"เกิดข้อผิดพลาดภายในระบบ"` **ไม่มี placeholder
  `{0}` เลย** นี่ไม่ใช่ความบังเอิญ — มันคือการตัดสินใจด้านความปลอดภัยที่ตั้งใจทำ: ไม่ว่า `anyhow::Error` ที่ห่อ
  อยู่ข้างในจะมีข้อความอะไรก็ตาม (อาจมีรายละเอียด เช่น connection string, path บนเครื่อง server, หรือ query
  SQL ที่ fail) **`.to_string()` ของ `AppError::Internal` จะไม่แสดงมันออกมาเด็ดขาด** เพราะ `thiserror` ไม่ได้
  ถูกสั่งให้ interpolate มันเข้าไปในข้อความ — หัวข้อ 66.5 จะพิสูจน์เรื่องนี้ด้วยการรันจริงทั้งสองแบบ (มี
  placeholder กับไม่มี) ให้เห็นความต่างชัด ๆ
- **JSON shape ที่ได้**: ทุก error ในระบบตอบเป็น `{"error": {"code": "...", "message": "..."}}` เสมอ (ยกเว้น
  `ValidationErrors` ที่เพิ่ม `"fields"` เข้ามา) — นี่คือคำตอบตรง ๆ ของปัญหาข้อ 1 ในหัวข้อ 66.1: frontend เขียน
  `response.json().error.code` ได้เหมือนกันทุก endpoint โดยไม่ต้องรู้ล่วงหน้าว่า endpoint นั้นจะ fail ด้วย
  สาเหตุอะไร

มาพิสูจน์ด้วยเซิร์ฟเวอร์จริง — เพิ่ม handler ทดสอบง่าย ๆ ที่คืน error แต่ละ variant ตรง ๆ:

```rust
# use axum::{extract::{Path, State}, http::StatusCode, response::IntoResponse, routing::get, Json, Router};
# use serde::Serialize;
# #[derive(Debug, Clone, Serialize)]
# struct Book { id: u32, title: String }
# struct AppState;
# type SharedState = std::sync::Arc<AppState>;
# #[derive(Debug, thiserror::Error)]
# enum AppError { #[error("x")] NotFound }
# impl axum::response::IntoResponse for AppError {
#     fn into_response(self) -> axum::response::Response { StatusCode::NOT_FOUND.into_response() }
# }
async fn get_book(
    State(_state): State<SharedState>,
    Path(id): Path<u32>,
) -> Result<Json<Book>, AppError> {
    // สมมติว่าไม่เจอหนังสือ id นี้ในฐานข้อมูล -- แปลงเป็น AppError::NotFound ด้วย .ok_or()
    let book: Option<Book> = None; // จำลองผลลัพธ์ที่หาไม่เจอ
    let book = book.ok_or(AppError::NotFound)?;
    Ok(Json(book))
}
```

รันจริง (เซิร์ฟเวอร์เต็มรูปแบบพร้อม endpoint ทดสอบทุก variant อยู่ในหัวข้อ 66.4-66.6 ที่จะค่อย ๆ ประกอบต่อไป)
แล้วยิง `curl` ไปที่หนังสือ id ที่ไม่มีอยู่จริง — ผลลัพธ์จริงที่ได้ตรงตามที่ออกแบบไว้เป๊ะ:

```bash
curl -sS -i http://127.0.0.1:3101/books/999
```

```
HTTP/1.1 404 Not Found
content-type: application/json
content-length: 106
date: Sun, 27 Sep 2026 00:04:28 GMT

{"error":{"code":"NOT_FOUND","message":"ไม่พบข้อมูลที่ต้องการ"}}
```

และผลลัพธ์จริงของทุก variant เมื่อทดสอบครบ (เซิร์ฟเวอร์ตัวเดียวกัน ต่างแค่ endpoint ที่ยิง — จะประกอบเต็มใน
หัวข้อ 66.10):

```bash
curl -sS -i -X POST http://127.0.0.1:3101/books/1/borrow   # ยืมซ้ำหนังสือที่ถูกยืมไปแล้ว
```
```
HTTP/1.1 409 Conflict
content-type: application/json
content-length: 203

{"error":{"code":"CONFLICT","message":"ขัดแย้งกับสถานะปัจจุบัน: หนังสือ 'The Rust Programming Language' ถูกยืมไปแล้ว"}}
```

```bash
curl -sS -i http://127.0.0.1:3101/demo/unauthorized
```
```
HTTP/1.1 401 Unauthorized
content-type: application/json
content-length: 91

{"error":{"code":"UNAUTHORIZED","message":"ไม่ได้รับอนุญาต"}}
```

```bash
curl -sS -i http://127.0.0.1:3101/demo/internal
```
```
HTTP/1.1 500 Internal Server Error
content-type: application/json
content-length: 117

{"error":{"code":"INTERNAL_ERROR","message":"เกิดข้อผิดพลาดภายในระบบ"}}
```

ทุก response มี `content-type: application/json` และโครงสร้าง `{"error": {"code": ..., "message": ...}}`
เหมือนกันหมด ไม่ว่า status code จะเป็น `404`, `409`, `401`, หรือ `500` — นี่คือสิ่งที่ Part 65 หัวข้อ capstone
**ทำไม่ได้** (จำ `(StatusCode::CONFLICT, "book is already borrowed")` ที่เป็น plain text ได้ไหม) เพราะไม่มี
จุดรวมแบบนี้

### 66.4 ใช้ `?` ไล่ chain การทำงานที่ fail ได้หลายจุดใน handler เดียว

ตอนนี้ `AppError: From<anyhow::Error>` ใช้งานได้แล้ว (ผ่าน `#[from]`) เราสามารถเขียน handler ที่ทำงานหลาย
ขั้นตอน **แต่ละขั้นตอน fail ได้ด้วยสาเหตุคนละแบบ** แล้วให้ `?` แปลงทุกอย่างเป็น `AppError` โดยอัตโนมัติ — นี่คือ
จุดที่ Part 12's `?` operator, Part 30's `From`, และ Part 31's `#[from]` มาบรรจบกันจริงในบริบทของเว็บ:

```rust
# use axum::{extract::{Path, State}, response::IntoResponse, Json};
# use serde::Serialize;
# use std::collections::HashMap;
# use std::sync::{Arc, Mutex};
# #[derive(Debug, Clone, Serialize)]
# struct Book { id: u32, title: String, available: bool }
# struct AppState { books: Mutex<HashMap<u32, Book>> }
# type SharedState = Arc<AppState>;
# #[derive(Debug, thiserror::Error)]
# enum AppError {
#     #[error("ไม่พบข้อมูลที่ต้องการ")]
#     NotFound,
#     #[error("ขัดแย้งกับสถานะปัจจุบัน: {0}")]
#     Conflict(String),
#     #[error("เกิดข้อผิดพลาดภายในระบบ")]
#     Internal(#[from] anyhow::Error),
# }
# impl axum::response::IntoResponse for AppError {
#     fn into_response(self) -> axum::response::Response { axum::http::StatusCode::OK.into_response() }
# }
// จำลองงาน "เขียน audit log" ที่อาจล้มเหลวได้ (เช่น เขียนไฟล์/ยิง event ไปคิว) -- คืน anyhow::Result ธรรมดา
// เพราะ handler ไม่ต้องรู้รายละเอียดว่า audit-log ล่มเพราะอะไร แค่ต้องรู้ว่า "ล่ม" ก็คือ 500 ทันที
fn write_audit_log(book_id: u32, action: &str) -> anyhow::Result<()> {
    if book_id == 42 {
        anyhow::bail!("ไม่สามารถเชื่อมต่อ audit-log service ได้ (connection refused)");
    }
    tracing::debug!(book_id, action, "เขียน audit log สำเร็จ");
    Ok(())
}

async fn borrow_book(
    State(state): State<SharedState>,
    Path(id): Path<u32>,
) -> Result<Json<Book>, AppError> {
    let mut books = state.books.lock().unwrap();

    // (1) หาไม่เจอ -> Option::ok_or(AppError::NotFound) -> ? ส่งต่อ AppError ตรงๆ (ไม่ต้องแปลงอะไรเพิ่ม)
    let book = books.get_mut(&id).ok_or(AppError::NotFound)?;

    // (2) กฎธุรกิจขัดกัน -> AppError::Conflict ตรงๆ ผ่าน early return (ไม่ใช้ ? เพราะไม่มี Result ต้นทาง)
    if !book.available {
        return Err(AppError::Conflict(format!("หนังสือ '{}' ถูกยืมไปแล้ว", book.title)));
    }

    book.available = false;
    let title = book.title.clone();
    drop(books); // ปล่อย lock ก่อนเรียกงานที่ไม่เกี่ยวกับ HashMap ต่อ (หลีกเลี่ยง lock ค้างนานเกินจำเป็น)

    // (3) ระบบภายนอกล้มเหลว -> anyhow::Result -> ? แปลงเป็น AppError::Internal โดยอัตโนมัติผ่าน #[from]
    //     บรรทัดนี้ไม่มีการเรียก .into()/.map_err() ให้เห็นเลย -- เหมือนที่ Part 30 หัวข้อ 30.4 อธิบายไว้ว่า
    //     ? desugar เป็น "return Err(From::from(error))" เสมอ ไม่ว่า error ต้นทางจะเป็นชนิดไหนก็ตาม
    write_audit_log(id, "borrow")?;

    tracing::info!(book_id = id, title = %title, "ยืมหนังสือสำเร็จ");
    let books = state.books.lock().unwrap();
    Ok(Json(books.get(&id).unwrap().clone()))
}
```

**สิ่งที่ควรสังเกตให้ชัดจากโค้ดนี้**: handler เดียวนี้ fail ได้จาก**สามสาเหตุที่ไม่เกี่ยวข้องกันเลย**
(`Option::None` จาก `HashMap::get_mut`, business rule ที่เขียนด้วยมือ, และ `anyhow::Error` จากฟังก์ชันอื่นที่
เรียกใช้) แต่ signature ของ handler สะอาดมาก — `Result<Json<Book>, AppError>` เพียงบรรทัดเดียวบอกครบว่า "ถ้า
สำเร็จได้ `Book`, ถ้าไม่สำเร็จได้ `AppError` ที่คุณ (compiler และคนอ่าน) รู้จักโครงสร้างเต็มรูปแบบ" — นี่คือ
สิ่งที่ Part 30 หัวข้อ 30.1 บอกว่า `Box<dyn Error>` ทำไม่ได้ (สูญเสีย "ล้มเหลวได้ด้วยสาเหตุอะไรบ้าง") แต่
`AppError` enum ทำได้เต็มรูปแบบ

ทดสอบ flow ปกติ (ยืมสำเร็จ) แล้วยืมซ้ำ (ชน `Conflict`) จริง:

```bash
curl -sS -i -X POST http://127.0.0.1:3101/books/1/borrow
```
```
HTTP/1.1 200 OK
content-type: application/json
content-length: 114

{"id":1,"title":"The Rust Programming Language","author":"Steve Klabnik","isbn":"9781593278281","available":false}
```

```bash
curl -sS -i -X POST http://127.0.0.1:3101/books/1/borrow   # ยืมซ้ำอีกครั้ง
```
```
HTTP/1.1 409 Conflict
content-type: application/json
content-length: 203

{"error":{"code":"CONFLICT","message":"ขัดแย้งกับสถานะปัจจุบัน: หนังสือ 'The Rust Programming Language' ถูกยืมไปแล้ว"}}
```

หัวข้อ 66.10 (capstone) จะแสดงกรณีที่ `write_audit_log` ล้มเหลวจริง (เมื่อ `book_id == 42`) พร้อม log เต็ม
รูปแบบ เพราะกรณีนั้นต้องผสานกับหัวข้อ 66.5 (logging) ก่อน — มาดูหัวข้อนั้นกันต่อ

### 66.5 Log ที่จุดเดียว: `tracing::error!` ภายใน `IntoResponse` + request id ผ่าน tracing span

Part 65 หัวข้อ 65.1 บอกไว้ว่าปัญหาของโค้ดแบบไม่มี middleware คือ "ไม่มีที่เดียวให้ไปดูว่า error เกิดขึ้นตรง
ไหนบ้าง" — ตอนนี้เรามี `impl IntoResponse for AppError` เป็น**จุดเดียว**ที่ทุก error ในระบบไหลผ่านก่อนกลายเป็น
response แล้ว เราจึงสามารถใส่ `tracing::error!`/`tracing::warn!` ไว้ที่จุดนี้เพียงจุดเดียว ให้ทุก handler ใน
ระบบได้ log ที่สอดคล้องกันโดยไม่ต้องเขียน `tracing::` ซ้ำในทุก handler เอง:

```rust
# use axum::{http::StatusCode, response::{IntoResponse, Response}, Json};
# use serde_json::json;
# use serde::Serialize;
# #[derive(Debug, Clone, Serialize)]
# struct FieldError { field: String, message: String }
# #[derive(Debug, thiserror::Error)]
# enum AppError {
#     #[error("ไม่พบข้อมูลที่ต้องการ")] NotFound,
#     #[error("ข้อมูลไม่ถูกต้อง: {0}")] Validation(String),
#     #[error("ข้อมูลไม่ผ่านการตรวจสอบ")] ValidationErrors(Vec<FieldError>),
#     #[error("ขัดแย้งกับสถานะปัจจุบัน: {0}")] Conflict(String),
#     #[error("ไม่ได้รับอนุญาต")] Unauthorized,
#     #[error("เกิดข้อผิดพลาดภายในระบบ")] Internal(#[from] anyhow::Error),
# }
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, code): (StatusCode, &str) = match &self {
            AppError::NotFound => (StatusCode::NOT_FOUND, "NOT_FOUND"),
            AppError::Validation(_) => (StatusCode::BAD_REQUEST, "VALIDATION_ERROR"),
            AppError::ValidationErrors(_) => (StatusCode::BAD_REQUEST, "VALIDATION_ERROR"),
            AppError::Conflict(_) => (StatusCode::CONFLICT, "CONFLICT"),
            AppError::Unauthorized => (StatusCode::UNAUTHORIZED, "UNAUTHORIZED"),
            AppError::Internal(_) => (StatusCode::INTERNAL_SERVER_ERROR, "INTERNAL_ERROR"),
        };

        // *** จุดศูนย์กลางของ logging ทั้งระบบ *** -- ทุก AppError ที่เกิดขึ้นไหลผ่านจุดนี้เพียงจุดเดียว
        match &self {
            // AppError::Internal ได้ log ระดับ ERROR พร้อมรายละเอียดเต็ม (source error ทั้ง chain)
            // -- ข้อมูลนี้ "ไม่เคย" ถูกส่งกลับไปให้ client เห็นเลย (ดูขั้นตอนสร้าง body ด้านล่าง)
            AppError::Internal(source) => {
                tracing::error!(
                    error_code = code,
                    status = status.as_u16(),
                    error = %source,       // Display -- ข้อความสรุประดับบนสุดของ error chain
                    error_debug = ?source, // Debug -- ถ้า source มี context หลายชั้น (anyhow .context())
                                            // จะเห็น "Caused by:" ไล่ทุกชั้นในนี้ด้วย
                    "internal error เกิดขึ้นระหว่างประมวลผล request"
                );
            }
            // error ที่เหลือเป็น "client error" ธรรมดา (ผู้ใช้ทำอะไรผิด ไม่ใช่ระบบพัง) -- log ระดับ WARN พอ
            other => {
                tracing::warn!(
                    error_code = code,
                    status = status.as_u16(),
                    error = %other,
                    "request ล้มเหลวด้วย client error"
                );
            }
        }

        let body = match self {
            AppError::ValidationErrors(fields) => json!({
                "error": { "code": code, "message": "ข้อมูลไม่ผ่านการตรวจสอบ", "fields": fields }
            }),
            other => json!({
                "error": { "code": code, "message": other.to_string() }
            }),
        };

        (status, Json(body)).into_response()
    }
}
```

จุดที่สำคัญที่สุดของหัวข้อนี้คือความต่างระหว่างสองสิ่งที่เกิดขึ้นในโค้ดข้างบน สำหรับ `AppError::Internal`:

1. **สิ่งที่ถูก log** (`tracing::error!`): ใช้ `%source` (Display) และ `?source` (Debug) ของ
   **`anyhow::Error` ตัวจริงที่ห่ออยู่ข้างใน** — ได้รายละเอียดเต็ม รวมทั้ง error chain ทั้งหมดถ้ามันถูกสร้าง
   ด้วย `.context()` (ตาม Part 31 หัวข้อ 31.12)
2. **สิ่งที่ถูกส่งกลับไปให้ client** (`body`): ใช้ `other.to_string()` ของ **`AppError` เอง** ซึ่งสำหรับ
   `Internal` คือข้อความคงที่ `"เกิดข้อผิดพลาดภายในระบบ"` เท่านั้น — ไม่มีทางที่ค่าจาก `source` จะรั่วไปถึง
   body เลย เพราะ `to_string()` ของ `AppError::Internal` ไม่ได้ไปแตะ `source` แม้แต่นิดเดียว (คนละ field กัน
   คนละ path การทำงานกัน)

**นี่คือหลักการความปลอดภัยที่สำคัญที่สุดข้อหนึ่งของทั้งบทนี้: "log เก็บทุกอย่าง, response บอกแค่พอ"** —
รายละเอียดภายใน (connection string, stack trace, ชื่อตารางในฐานข้อมูล, path บนเครื่อง server) เป็นข้อมูลที่
**มีประโยชน์กับทีมพัฒนาเท่านั้น** และ**เป็นอันตรายถ้าหลุดไปถึงมือ attacker** (มันบอก attacker ว่าระบบใช้
เทคโนโลยีอะไร โครงสร้างภายในเป็นยังไง ซึ่งช่วยวางแผนโจมตีต่อได้ง่ายขึ้น) — การแยกสอง path นี้ให้ชัดเจนตั้งแต่
ระดับ type (ไม่ใช่แค่ "จำไว้ว่าต้องระวัง") คือสิ่งที่การออกแบบ `AppError` แบบนี้ทำให้ได้โดยธรรมชาติ

#### เพิ่ม request id ผ่าน tracing span (ต่อยอด Part 64/65)

ตาม Part 65 หัวข้อ 65.9-65.10 เราเขียน middleware ที่แนบ `RequestId` เข้า `Extension` ได้ — แต่ในบทนี้เราใช้
เทคนิคที่ **ทรงพลังกว่าและใช้กันจริงในโค้ด production มากกว่า**: ครอบการประมวลผลทั้ง request ไว้ใน
**tracing span** เดียวที่มี field `request_id` ติดอยู่ — เพราะ span "ครอบ" ทุก event ที่เกิดขึ้นภายในช่วงเวลา
ของมัน (ตามที่ Part 60 สอนไว้) **`tracing::error!`/`warn!` ที่เรียกจากภายใน `into_response()` จะถูกนับเป็น
ส่วนหนึ่งของ span นั้นโดยอัตโนมัติ** โดยไม่ต้องส่ง request id ผ่านพารามิเตอร์ของ `AppError` เองเลยแม้แต่นิดเดียว:

```rust
# use axum::{response::Response, extract::Request, middleware::Next};
use std::sync::atomic::{AtomicU64, Ordering};
use tracing::Instrument;

static REQUEST_COUNTER: AtomicU64 = AtomicU64::new(1);

async fn request_id_middleware(req: Request, next: Next) -> Response {
    let id = REQUEST_COUNTER.fetch_add(1, Ordering::Relaxed);
    let request_id = format!("req-{id}");

    // สร้าง span ที่มี field request_id -- ครอบการทำงานทั้งหมดของ request นี้ไว้ข้างใน
    let span = tracing::info_span!("request", request_id = %request_id);

    // .instrument(span) จาก trait tracing::Instrument ใช้ได้กับ future ใดๆ (ไม่จำเพาะ Axum เลย)
    // ทำให้ next.run(req) -- ซึ่งครอบคลุมทั้ง handler และ AppError::into_response() ที่อาจถูกเรียก
    // ข้างในนั้น -- ทำงานอยู่ "ภายใน" span นี้ตลอดเวลา จน future เสร็จสมบูรณ์
    next.run(req).instrument(span).await
}
```

จุดที่ควรทำความเข้าใจให้แม่น: **`AppError::into_response()` ไม่รู้จัก `request_id` เลยแม้แต่นิดเดียว** —
มันไม่มี field `request_id` อยู่ใน enum ด้วยซ้ำ แต่เพราะ `tracing::error!`/`warn!` ที่เรียกจากข้างในมันทำงาน
**อยู่ภายใน span ที่ `request_id_middleware` สร้างไว้** (ผ่าน `.instrument()`) log ที่ออกมาจึงมี field
`request_id` ติดมาโดยอัตโนมัติเสมอ — นี่คือพลังของการแยก "ข้อมูลบริบทของ request" (request id) ออกจาก
"ตรรกะของ error" (`AppError`) อย่างสมบูรณ์ ตามหลักการ separation of concerns เดียวกับที่ Part 65 อธิบายไว้
เรื่อง middleware ทั้งบท

ประกอบ router พร้อม middleware นี้:

```rust
# use axum::{Router, routing::get};
# async fn get_book() -> &'static str { "ok" }
# async fn request_id_middleware(req: axum::extract::Request, next: axum::middleware::Next) -> axum::response::Response { next.run(req).await }
let app: Router<()> = Router::new()
    .route("/books/{id}", get(get_book))
    .layer(axum::middleware::from_fn(request_id_middleware));
```

ทดสอบชุดเดียวกับหัวข้อ 66.3-66.4 ทั้งหมด (`NotFound`, `ValidationErrors`, สร้างหนังสือสำเร็จ, ยืมสำเร็จ, ยืม
ซ้ำ, `Unauthorized`, `Internal`) ในลำดับเดียวติดต่อกัน แล้วดู log จริงที่ได้ (คัดลอกตรงจากการรันจริง ตัด ANSI
color code ออกเพื่อความอ่านง่าย — คำสั่ง `curl` ที่ยิงแต่ละอันเหมือนหัวข้อ 66.3-66.4 ทุกประการ):

```
2026-09-27T00:04:20.911640Z  INFO app_error_core: app_error_core ฟังอยู่ที่ http://127.0.0.1:3101
2026-09-27T00:04:28.766618Z  WARN request{request_id=req-1}: app_error_core: request ล้มเหลวด้วย client error error_code="NOT_FOUND" status=404 error=ไม่พบข้อมูลที่ต้องการ
2026-09-27T00:04:28.773126Z  WARN request{request_id=req-2}: app_error_core: request ล้มเหลวด้วย client error error_code="VALIDATION_ERROR" status=400 error=ข้อมูลไม่ผ่านการตรวจสอบ
2026-09-27T00:04:28.787536Z DEBUG request{request_id=req-4}: app_error_core: เขียน audit log สำเร็จ book_id=1 action="borrow"
2026-09-27T00:04:28.787570Z  INFO request{request_id=req-4}: app_error_core: ยืมหนังสือสำเร็จ book_id=1 title=The Rust Programming Language
2026-09-27T00:04:28.793309Z  WARN request{request_id=req-5}: app_error_core: request ล้มเหลวด้วย client error error_code="CONFLICT" status=409 error=ขัดแย้งกับสถานะปัจจุบัน: หนังสือ 'The Rust Programming Language' ถูกยืมไปแล้ว
2026-09-27T00:04:28.799577Z  WARN request{request_id=req-6}: app_error_core: request ล้มเหลวด้วย client error error_code="UNAUTHORIZED" status=401 error=ไม่ได้รับอนุญาต
2026-09-27T00:04:28.805091Z ERROR request{request_id=req-7}: app_error_core: internal error เกิดขึ้นระหว่างประมวลผล request error_code="INTERNAL_ERROR" status=500 error=ไม่สามารถขอ connection จาก database pool ได้ error_debug=ไม่สามารถขอ connection จาก database pool ได้

Caused by:
    connection refused (10.0.4.12:5432)
```

สังเกตทีละจุด:

- **`request{request_id=req-N}:`** ปรากฏหน้าทุกบรรทัด log ที่เกิดขึ้นระหว่างประมวลผล request นั้น — นี่คือ
  span ที่ `request_id_middleware` สร้างไว้ กำลังครอบ event ทุกตัวที่เกิดข้างใน ตามที่อธิบายไว้ข้างบน (แม้แต่
  `เขียน audit log สำเร็จ` ที่ log จากฟังก์ชันอื่น ไม่ใช่จาก `AppError::into_response()` เลย ก็ยังมี
  `request_id` ติดมาด้วย เพราะมันทำงานอยู่ภายใน `next.run(req).instrument(span)` เดียวกัน)
- **บรรทัดสุดท้าย (`req-7`, `500`)**: นี่คือกรณีที่จำลอง database pool ล้มเหลว โดยใช้ฟังก์ชันที่สร้าง error
  ด้วย `anyhow::Context` (ตาม Part 31 หัวข้อ 31.12):

  ```rust
  fn simulate_db_pool_error() -> anyhow::Result<()> {
      use anyhow::Context;
      let root_cause = std::io::Error::new(
          std::io::ErrorKind::ConnectionRefused,
          "connection refused (10.0.4.12:5432)",
      );
      Err(root_cause).context("ไม่สามารถขอ connection จาก database pool ได้")
  }
  ```

  สังเกตว่า **`error=` (Display) แสดงแค่ข้อความชั้นบนสุด** (`"ไม่สามารถขอ connection จาก database pool
  ได้"`) แต่ **`error_debug=` (Debug) แสดงทั้ง chain** รวมถึงบรรทัด `Caused by: connection refused
  (10.0.4.12:5432)` ที่มาจาก `root_cause` ดั้งเดิม — นี่คือกลไก error chain เดียวกับที่ Part 30 หัวข้อ 30.5
  สอนผ่าน `source()` เพียงแต่ `anyhow`/`{:?}` ทำให้ไม่ต้องเขียน loop ไล่เองตามที่ Part 31 หัวข้อ 31.12 อธิบาย
  ไว้ — **ทีม operations ที่อ่าน log นี้เห็นครบทุกรายละเอียดที่ต้องใช้ debug** (ปัญหาอยู่ที่ database pool,
  สาเหตุจริงคือ connection refused ที่ IP นี้)
- **สิ่งที่ client ได้กลับ (`curl` ในหัวข้อ 66.3) คือแค่**: `{"error":{"code":"INTERNAL_ERROR","message":
  "เกิดข้อผิดพลาดภายในระบบ"}}` — ไม่มีคำว่า "database", "pool", "connection refused", หรือ IP address `
  10.0.4.12` หลุดออกไปแม้แต่ตัวอักษรเดียว — พิสูจน์ตรงกับที่อธิบายไว้ข้างบนทุกประการ: log เก็บทุกอย่าง,
  response บอกแค่พอ

### 66.6 Validation แบบ field-level: `ValidationErrors` + `FieldError`

Section 66.2 นิยาม `AppError::ValidationErrors(Vec<FieldError>)` ไว้แล้ว — ตอนนี้มาดูการใช้งานจริงกับ
สถานการณ์ที่พบบ่อยที่สุดของ REST API: รับข้อมูลสร้าง resource ใหม่จาก client แล้วต้องตรวจสอบหลาย field
พร้อมกัน และรายงาน**ทุกจุดที่ผิด**กลับไปในครั้งเดียว (ไม่ใช่แค่จุดแรกที่เจอ — ซึ่งบีบให้ client ต้อง submit
ซ้ำหลายรอบกว่าจะรู้ว่าผิดกี่จุด):

```rust
use serde::Deserialize;
# use serde::Serialize;
# #[derive(Debug, Clone, Serialize)]
# struct FieldError { field: String, message: String }
# #[derive(Debug, thiserror::Error)]
# enum AppError {
#     #[error("ข้อมูลไม่ผ่านการตรวจสอบ")]
#     ValidationErrors(Vec<FieldError>),
# }

#[derive(Debug, Deserialize)]
struct NewBook {
    title: String,
    author: String,
    isbn: String,
}

fn validate_new_book(input: &NewBook) -> Result<(), AppError> {
    let mut errors = Vec::new();

    if input.title.trim().is_empty() {
        errors.push(FieldError {
            field: "title".to_string(),
            message: "ต้องไม่เป็นค่าว่าง".to_string(),
        });
    }
    if input.author.trim().is_empty() {
        errors.push(FieldError {
            field: "author".to_string(),
            message: "ต้องไม่เป็นค่าว่าง".to_string(),
        });
    }
    if input.isbn.len() != 13 || !input.isbn.chars().all(|c| c.is_ascii_digit()) {
        errors.push(FieldError {
            field: "isbn".to_string(),
            message: "ต้องเป็นตัวเลข 13 หลัก".to_string(),
        });
    }

    if errors.is_empty() {
        Ok(())
    } else {
        Err(AppError::ValidationErrors(errors))
    }
}
```

**อธิบายการออกแบบ**: `validate_new_book` ไม่ return ทันทีที่เจอ field แรกที่ผิด (ไม่มี `?` คั่นระหว่างการ
เช็คแต่ละ field) — มันสะสม `FieldError` ทุกจุดที่ผิดไว้ใน `Vec` ก่อน แล้วค่อยตัดสินใจตอนจบว่าจะ `Ok(())` หรือ
`Err(AppError::ValidationErrors(errors))` — ต่างจาก validation แบบ `?` ทั่วไปที่หยุดทันทีที่เจอจุดแรก (แบบที่
ใช้ในหัวข้อ 66.4 กับ `NotFound`/`Conflict`) เพราะ**ธรรมชาติของงานต่างกัน**: การหา resource/เช็คกฎธุรกิจเป็น
ลำดับขั้นที่พึ่งพากัน (ถ้าหาไม่เจอ ก็ไม่มีประโยชน์จะเช็คกฎธุรกิจต่อ) แต่การตรวจสอบ field ของ input เป็นงานที่
**แต่ละ field ตรวจสอบอิสระจากกัน** ผู้ใช้ได้ประโยชน์มากกว่าถ้ารู้ทุกจุดที่ผิดในครั้งเดียว

ใช้ใน handler สร้างหนังสือ:

```rust
# use axum::{extract::State, http::StatusCode, Json};
# use serde::{Deserialize, Serialize};
# use std::sync::{Arc, Mutex};
# #[derive(Debug, Clone, Serialize)]
# struct Book { id: u32, title: String, author: String, isbn: String, available: bool }
# #[derive(Debug, Deserialize)]
# struct NewBook { title: String, author: String, isbn: String }
# struct AppState { next_id: Mutex<u32>, books: Mutex<std::collections::HashMap<u32, Book>> }
# type SharedState = Arc<AppState>;
# #[derive(Debug, thiserror::Error)]
# enum AppError { #[error("x")] ValidationErrors(Vec<()>) }
# impl axum::response::IntoResponse for AppError {
#     fn into_response(self) -> axum::response::Response { StatusCode::OK.into_response() }
# }
# fn validate_new_book(_i: &NewBook) -> Result<(), AppError> { Ok(()) }
async fn create_book(
    State(state): State<SharedState>,
    Json(new_book): Json<NewBook>,
) -> Result<(StatusCode, Json<Book>), AppError> {
    validate_new_book(&new_book)?; // ValidationErrors ถ้ามี field ผิด -- หยุดทันที ไม่สร้างหนังสือ

    let mut next_id = state.next_id.lock().unwrap();
    let id = *next_id;
    *next_id += 1;

    let book = Book {
        id,
        title: new_book.title,
        author: new_book.author,
        isbn: new_book.isbn,
        available: true,
    };
    state.books.lock().unwrap().insert(id, book.clone());
    Ok((StatusCode::CREATED, Json(book)))
}
```

ทดสอบด้วย input ที่ผิดสามจุดพร้อมกัน (`title` ว่าง, `author` ว่าง, `isbn` ไม่ครบ 13 หลัก):

```bash
curl -sS -i -X POST http://127.0.0.1:3101/books \
  -H 'Content-Type: application/json' \
  -d '{"title":"","author":"","isbn":"123"}'
```

ผลลัพธ์จริง — **ได้ครบทั้งสามจุดในครั้งเดียว** ไม่ต้อง submit ซ้ำสามรอบ:

```
HTTP/1.1 400 Bad Request
content-type: application/json
content-length: 389

{"error":{"code":"VALIDATION_ERROR","fields":[{"field":"title","message":"ต้องไม่เป็นค่าว่าง"},{"field":"author","message":"ต้องไม่เป็นค่าว่าง"},{"field":"isbn","message":"ต้องเป็นตัวเลข 13 หลัก"}],"message":"ข้อมูลไม่ผ่านการตรวจสอบ"}}
```

จัด format ให้อ่านง่าย (ตัวจริงที่ server ส่งกลับเป็นบรรทัดเดียว ไม่มีการเว้นบรรทัด):

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "ข้อมูลไม่ผ่านการตรวจสอบ",
    "fields": [
      { "field": "title", "message": "ต้องไม่เป็นค่าว่าง" },
      { "field": "author", "message": "ต้องไม่เป็นค่าว่าง" },
      { "field": "isbn", "message": "ต้องเป็นตัวเลข 13 หลัก" }
    ]
  }
}
```

ฝั่ง frontend ที่ได้ JSON นี้ทำสิ่งที่ทำไม่ได้เลยกับ error แบบ string เดียวใน Part 65: วน `fields` แล้ว
`document.querySelector(`[name="${f.field}"]`)` เพื่อ highlight ทุก input ที่ผิดพร้อมกันในครั้งเดียว — นี่คือ
เหตุผลที่ต้องแยก `ValidationErrors(Vec<FieldError>)` ออกจาก `Validation(String)` ตั้งแต่ระดับ type ตามที่
อธิบายไว้ในหัวข้อ 66.2

ทดสอบสร้างหนังสือด้วยข้อมูลถูกต้องเพื่อยืนยันว่า path สำเร็จยังทำงานปกติ:

```bash
curl -sS -i -X POST http://127.0.0.1:3101/books \
  -H 'Content-Type: application/json' \
  -d '{"title":"Programming Rust","author":"Jim Blandy","isbn":"9781492052593"}'
```
```
HTTP/1.1 201 Created
content-type: application/json
content-length: 97

{"id":2,"title":"Programming Rust","author":"Jim Blandy","isbn":"9781492052593","available":true}
```

### 66.7 Extractor rejection ที่สม่ำเสมอ: `AppJson<T>` ด้วย `#[derive(FromRequest)]`

จนถึงตอนนี้ทุก error ที่มาจาก**ตรรกะของ handler เอง** (เราเขียน `Err(AppError::...)` ตรง ๆ) ตอบเป็น JSON
shape ที่สอดคล้องกันแล้ว — แต่ยังมี error อีกแหล่งที่ยังไม่ถูกจัดการ: **error ที่เกิดจาก extractor เอง** ก่อน
ที่ handler จะได้เริ่มทำงานด้วยซ้ำ — เช่น body ที่ไม่ใช่ JSON ที่ถูกต้อง, หรือ `Content-Type` ที่ผิด

#### พฤติกรรม default ของ Axum: plain text ไม่ใช่ JSON

ลองส่ง body ที่ไม่ใช่ JSON ที่ถูกต้องไปยัง handler ที่ใช้ `axum::Json<T>` ตรง ๆ (ตามที่ Part 63-64 สอนไว้):

```rust
# use axum::Json;
# use serde::{Deserialize, Serialize};
# #[derive(Debug, Deserialize, Serialize)]
# struct EchoBody { message: String }
async fn echo_default(Json(body): Json<EchoBody>) -> Json<EchoBody> {
    Json(body)
}
```

```bash
curl -sS -i -X POST http://127.0.0.1:3102/default/echo \
  -H 'Content-Type: application/json' -d '{"message": invalid}'
```

ผลลัพธ์จริง — **ไม่ใช่ JSON เลย**:

```
HTTP/1.1 400 Bad Request
content-type: text/plain; charset=utf-8
content-length: 85

Failed to parse the request body as JSON: message: expected value at line 1 column 13
```

และถ้าลืมส่ง header `Content-Type: application/json` ไปเลย:

```bash
curl -sS -i -X POST http://127.0.0.1:3102/default/echo -d '{"message":"hello"}'
```
```
HTTP/1.1 415 Unsupported Media Type
content-type: text/plain; charset=utf-8
content-length: 54

Expected request with `Content-Type: application/json`
```

ทั้งสองกรณีนี้เป็น **`400`/`415` แบบ plain text ล้วน ๆ** ทั้งที่ endpoint อื่นในระบบเดียวกันตอบ JSON หมด —
ตรงกับปัญหาข้อ 1 ในหัวข้อ 66.1 เป๊ะ ๆ เพียงแต่คราวนี้เกิดจาก**กลไกภายในของ Axum เอง** ไม่ใช่จากโค้ดที่เราเขียน
— `Json<T>` ของ Axum มี rejection type ของตัวเอง (`axum::extract::rejection::JsonRejection`) ที่ implement
`IntoResponse` ไว้ให้แล้วเป็น plain text ตามค่า default ของ framework

#### ทางแก้: extractor wrapper ที่แปลง rejection เป็น `AppError`

Axum (ตั้งแต่มี feature `macros`) มี derive macro `#[derive(FromRequest)]` ที่ให้เราสร้าง **extractor ของ
ตัวเอง ที่ "ห่อ" extractor ที่มีอยู่แล้ว** (`axum::Json` ในที่นี้) แล้วกำหนดว่าเมื่อมัน rejection ให้แปลงเป็น
type ไหน — ตรงตามที่ Part 64 หัวข้อ 64.7 สอนไว้เรื่อง custom `FromRequestParts` เพียงแต่คราวนี้ให้ macro
เขียน `impl` ให้แทนการเขียนด้วยมือทั้งหมด (เหมือนที่ `thiserror` เขียน `impl Display`/`impl Error` แทนเราใน
Part 31):

```rust
use axum::extract::{rejection::JsonRejection, FromRequest};
# use serde::{Deserialize, Serialize};
# #[derive(Debug, thiserror::Error)]
# enum AppError {
#     #[error("ข้อมูลไม่ถูกต้อง: {0}")]
#     Validation(String),
# }
# impl axum::response::IntoResponse for AppError {
#     fn into_response(self) -> axum::response::Response { axum::http::StatusCode::OK.into_response() }
# }

// สอนให้ AppError "รู้วิธีแปลงตัวเอง" จาก JsonRejection ของ Axum -- เหมือน impl From ปกติทุกประการ
// (rejection.body_text() คือข้อความ plain text แบบเดียวกับที่เห็นในหัวข้อก่อน แค่เอามาห่อเป็น AppError)
impl From<JsonRejection> for AppError {
    fn from(rejection: JsonRejection) -> Self {
        AppError::Validation(rejection.body_text())
    }
}

// extractor ของเราเอง -- ใช้แทน axum::Json<T> ได้ทุกที่ในระบบ
// via(axum::Json)   บอกว่า "ให้ axum::Json<T> ทำงานหนักให้เหมือนเดิม แค่ยืมมันมาใช้"
// rejection(AppError) บอกว่า "ถ้า axum::Json rejection ให้แปลงเป็น AppError ผ่าน impl From ข้างบน"
#[derive(FromRequest)]
#[from_request(via(axum::Json), rejection(AppError))]
struct AppJson<T>(T);
```

**อธิบายกลไก**: `#[derive(FromRequest)]` อ่าน attribute `#[from_request(...)]` แล้ว generate
`impl<S, T> FromRequest<S> for AppJson<T>` ให้อัตโนมัติ ซึ่งภายในทำสิ่งที่เทียบเท่ากับ:

```rust
# use axum::extract::{FromRequest, Request};
# use serde::de::DeserializeOwned;
# struct AppJson<T>(T);
# #[derive(Debug, thiserror::Error)]
# enum AppError { #[error("x")] Validation(String) }
# impl From<axum::extract::rejection::JsonRejection> for AppError {
#     fn from(r: axum::extract::rejection::JsonRejection) -> Self { AppError::Validation(r.body_text()) }
# }
impl<T, S> FromRequest<S> for AppJson<T>
where
    T: DeserializeOwned,
    S: Send + Sync,
{
    type Rejection = AppError;

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        let axum::Json(value) = axum::Json::<T>::from_request(req, state).await?; // ? ใช้ impl From ข้างบน
        Ok(AppJson(value))
    }
}
```

(นี่คือโค้ดที่เทียบเท่าเพื่อการอธิบาย — โค้ดจริงที่ compile คือ derive macro ข้างบนเพียง 3 บรรทัด) สังเกตว่า
`?` ใน `from_request` ทำงานตามกฎเดียวกับที่ Part 30 หัวข้อ 30.4 อธิบายไว้ทุกประการ: `JsonRejection` แปลงเป็น
`AppError` โดยอัตโนมัติเพราะมี `impl From<JsonRejection> for AppError` ที่เราเขียนไว้แล้ว — **derive macro
ของ Axum พึ่งพากลไก `From`/`?` แบบเดียวกับที่คุณรู้จักมาตั้งแต่ Part 30 ทุกประการ ไม่มีอะไรใหม่ที่ต้องท่องจำ
เพิ่ม**

ใช้แทน `axum::Json<T>` ในทุก handler ที่ต้องการ:

```rust
# use axum::extract::FromRequest;
# use serde::{Deserialize, Serialize};
# #[derive(Debug, Deserialize, Serialize)]
# struct EchoBody { message: String }
# #[derive(FromRequest)]
# #[from_request(via(axum::Json), rejection(axum::http::StatusCode))]
# struct AppJson<T>(T);
# impl<T: Serialize> axum::response::IntoResponse for AppJson<T> {
#     fn into_response(self) -> axum::response::Response {
#         axum::Json(self.0).into_response()
#     }
# }
async fn echo_custom(AppJson(body): AppJson<EchoBody>) -> AppJson<EchoBody> {
    AppJson(body)
}
```

(สำหรับ response ก็ต้อง `impl IntoResponse for AppJson<T>` เองเช่นกัน เพราะ `AppJson` เป็น type ใหม่ที่เรา
สร้าง ไม่ใช่ `axum::Json` — ทำได้ง่าย ๆ ด้วยการ delegate ไปที่ `axum::Json(self.0).into_response()` เหมือนที่
Part 63 หัวข้อ 63.8 สอนไว้เรื่องการห่อ type ที่มี `IntoResponse` อยู่แล้ว)

ทดสอบ **"ก่อน" กับ "หลัง" เทียบกันตรง ๆ** ด้วย body เดิมทุกประการ:

```bash
# "ก่อน" -- axum::Json ตรงๆ
curl -sS -i -X POST http://127.0.0.1:3102/default/echo \
  -H 'Content-Type: application/json' -d '{"message": invalid}'
```
```
HTTP/1.1 400 Bad Request
content-type: text/plain; charset=utf-8
content-length: 85

Failed to parse the request body as JSON: message: expected value at line 1 column 13
```

```bash
# "หลัง" -- AppJson แทน
curl -sS -i -X POST http://127.0.0.1:3102/custom/echo \
  -H 'Content-Type: application/json' -d '{"message": invalid}'
```
```
HTTP/1.1 400 Bad Request
content-type: application/json
content-length: 185

{"error":{"code":"VALIDATION_ERROR","message":"ข้อมูลไม่ถูกต้อง: Failed to parse the request body as JSON: message: expected value at line 1 column 13"}}
```

status code เดิม (`400`) แต่ `content-type` เปลี่ยนจาก `text/plain` เป็น `application/json` และ body เป็น
JSON shape เดียวกับ error อื่นทั้งระบบ — client ที่เขียน `response.json()` แบบเดียวกันทุก endpoint ใช้งานได้
โดยไม่ต้องเขียน special case สำหรับ "endpoint นี้ตอบ error เป็น text" อีกต่อไป และกรณีลืม `Content-Type` ก็
เช่นกัน:

```bash
curl -sS -i -X POST http://127.0.0.1:3102/custom/echo -d '{"message":"hello"}'
```
```
HTTP/1.1 400 Bad Request
content-type: application/json
content-length: 154

{"error":{"code":"VALIDATION_ERROR","message":"ข้อมูลไม่ถูกต้อง: Expected request with `Content-Type: application/json`"}}
```

> **หมายเหตุเรื่อง trade-off**: สังเกตว่ากรณีนี้ Axum เดิมตอบ `415 Unsupported Media Type` แต่เวอร์ชัน
> `AppJson` ของเราตอบ `400 Bad Request` เสมอ (เพราะ `AppError::Validation` ผูกกับ `400` ตายตัวตามที่ออกแบบไว้
> ในหัวข้อ 66.3) — นี่คือการตัดสินใจที่ตั้งใจแลก **ความสอดคล้องของ JSON shape** กับ **ความแม่นยำของ status
> code ในบางกรณีขอบ ๆ** ถ้าโปรเจกต์คุณต้องการรักษา status code เดิมของ Axum ไว้เป๊ะ ๆ ทำได้โดยอ่าน
> `rejection.status()` ก่อนแปลง แล้วเก็บมันไว้เป็น field เพิ่มใน `AppError::Validation` (เช่น
> `Validation { status: StatusCode, message: String }`) แทนที่จะผูก `400` ตายตัว — เป็น judgment call ที่
> ต้องเลือกตามความต้องการของแต่ละระบบ

#### Path/Query extractor ก็ทำแบบเดียวกันได้ — แต่ derive คนละตัว

`Path<T>`/`Query<T>` ทำงานผ่าน **`FromRequestParts`** (อ่านแค่ header/URL, ไม่แตะ body — ตามที่ Part 64
หัวข้อ 64.6 อธิบายไว้) ไม่ใช่ `FromRequest` เต็มรูปแบบแบบ `Json<T>` — จึงต้องใช้ derive คนละตัว:
`#[derive(FromRequestParts)]` แทน `#[derive(FromRequest)]`:

```rust
use axum::extract::{rejection::PathRejection, FromRequestParts, Path};
# #[derive(Debug, thiserror::Error)]
# enum AppError {
#     #[error("ข้อมูลไม่ถูกต้อง: {0}")]
#     Validation(String),
# }
# impl axum::response::IntoResponse for AppError {
#     fn into_response(self) -> axum::response::Response { axum::http::StatusCode::OK.into_response() }
# }

impl From<PathRejection> for AppError {
    fn from(rejection: PathRejection) -> Self {
        AppError::Validation(rejection.body_text())
    }
}

#[derive(FromRequestParts)]
#[from_request(via(axum::extract::Path), rejection(AppError))]
struct AppPath<T>(T);
```

ทดสอบด้วย path parameter ที่ผิดรูปแบบ (`abc` ไม่ใช่ `u32`):

```bash
curl -sS -i http://127.0.0.1:3102/default/books/abc   # axum::Path ตรงๆ
```
```
HTTP/1.1 400 Bad Request
content-type: text/plain; charset=utf-8
content-length: 42

Invalid URL: Cannot parse `abc` to a `u32`
```

```bash
curl -sS -i http://127.0.0.1:3102/custom/books/abc   # AppPath แทน
```
```
HTTP/1.1 400 Bad Request
content-type: application/json
content-length: 142

{"error":{"code":"VALIDATION_ERROR","message":"ข้อมูลไม่ถูกต้อง: Invalid URL: Cannot parse `abc` to a `u32`"}}
```

ตรรกะเดียวกันเป๊ะ — แค่เปลี่ยน derive macro ให้ตรงกับว่า extractor ต้นทางทำงานผ่าน `FromRequest` (แตะ body
ได้ ใช้ได้ครั้งเดียวต่อ request) หรือ `FromRequestParts` (แตะแค่ header/URL ใช้ซ้ำได้หลายครั้ง)

### 66.8 `#[axum::debug_handler]` และ fallback สำหรับ route ที่ไม่ตรงกับอะไรเลย

#### `#[axum::debug_handler]`: error message ที่อ่านง่ายขึ้นตอน compile

Axum มีกฎที่ Part 64 หัวข้อ 64.4 พูดถึงสั้น ๆ ไว้แล้ว: **extractor ที่ "กิน" request body (เช่น
`Json<T>`/`AppJson<T>`) ต้องเป็น parameter ตัวสุดท้ายของ handler เสมอ** เพราะ body อ่านได้ครั้งเดียว
extractor ตัวอื่นที่มาหลังจากมันจะไม่มี body เหลือให้อ่าน — ลองเขียนโค้ดที่ผิดกฎนี้ (จงใจสลับลำดับ):

```rust
# use axum::{extract::State, Json};
# use serde::Deserialize;
# use std::sync::Arc;
# #[derive(Clone)] struct AppState;
# #[derive(Deserialize)] struct NewBook { title: String }
async fn broken_order(Json(_body): Json<NewBook>, State(_state): State<Arc<AppState>>) -> &'static str {
    "ok"
}
```

ถ้า**ไม่มี** `#[axum::debug_handler]` แล้วเอาไปใช้กับ `.route("/books", post(broken_order))` compiler ตอบ
error แบบนี้จริง (ทดสอบด้วย `cargo build` จริง — เห็นเฉพาะ error หลัก ตัดส่วน `note`/`help` ที่ซ้ำออก):

```
error[E0277]: the trait bound `fn(Json<NewBook>, State<...>) -> ... {broken_order}: Handler<_, _>` is not satisfied
   --> src/bin/debug_handler_before.rs:27:31
    |
 27 |         .route("/books", post(broken_order))
    |                          ---- ^^^^^^^^^^^^ unsatisfied trait bound
    |                          |
    |                          required by a bound introduced by this call
    |
    = help: the trait `Handler<_, _>` is not implemented for fn item `fn(Json<NewBook>, State<Arc<AppState>>) -> impl Future<Output = &'static str> {broken_order}`
    = note: Consider using `#[axum::debug_handler]` to improve the error message
```

error นี้ **ไม่บอกตรง ๆ ว่าปัญหาคือลำดับ extractor ผิด** — มันบอกแค่ "`Handler` trait ไม่ผ่าน" แล้วเดา
ประเภทเต็ม ๆ ของฟังก์ชันมาแสดง ซึ่งสำหรับ handler ที่มี extractor หลายตัวจริงในโปรเจกต์จริง ประเภทนี้จะยาว
มากจนอ่านไม่ออกเลย — สังเกตว่า compiler เอง**แนะนำ**ให้ใช้ `#[axum::debug_handler]` ไว้ในบรรทัดสุดท้ายแล้ว
ลองทำตาม — เพิ่ม attribute นี้เข้าไปบรรทัดเดียว (โค้ดที่เหลือเหมือนกันทุกตัวอักษร):

```rust
# use axum::{extract::State, Json};
# use serde::Deserialize;
# use std::sync::Arc;
# #[derive(Clone)] struct AppState;
# #[derive(Deserialize)] struct NewBook { title: String }
#[axum::debug_handler]
async fn broken_order(Json(_body): Json<NewBook>, State(_state): State<Arc<AppState>>) -> &'static str {
    "ok"
}
```

คราวนี้ compiler ให้ error **ที่บอกสาเหตุตรงตัวเลย**:

```
error: `Json<_>` consumes the request body and thus must be the last argument to the handler function
  --> src/bin/debug_handler_after.rs:19:36
   |
19 | async fn broken_order(Json(_body): Json<NewBook>, State(_state): State<Arc<AppState>>) -> &'static str {
   |                                    ^^^^
```

(error `E0277` ตัวเดิมยังปรากฏต่อจากนี้ด้วย เพราะ handler ยังผิดอยู่จริง แต่ตอนนี้คุณมีข้อความที่บอกสาเหตุ
ตรงตัวมาก่อนแล้ว) **`#[axum::debug_handler]` ทำงานแค่ตอน compile time เท่านั้น** — มันไม่มี runtime cost
เลยแม้แต่นิดเดียว (ไม่ต่างจาก `#[derive(...)]` ใด ๆ ที่เรียนมาตลอดหลักสูตร) สิ่งที่มันทำคือ generate โค้ด
เสริมที่ตรวจสอบ**เงื่อนไขทั่วไปที่ทำให้ handler ผิดพลาด** (ลำดับ extractor ผิด, ใส่ extractor สองตัวที่กิน
body พร้อมกัน, ลืม `async`, ฯลฯ) แล้วรายงาน error ที่ตรงประเด็นกว่าให้ตั้งแต่ตอนพัฒนา — **แนวทางที่แนะนำ**: ใส่
`#[axum::debug_handler]` ไว้กับ handler ที่ซับซ้อน (มี extractor หลายตัว) ระหว่างพัฒนา แล้วเอาออกได้ทีหลังถ้า
ต้องการ (หรือจะเก็บไว้ตลอดก็ได้ เพราะไม่มีต้นทุนตอน runtime)

#### Fallback: route ที่ไม่ตรงกับอะไรเลยต้องได้ JSON shape เดียวกัน

Part 63 หัวข้อ 63.4 บอกไว้สั้น ๆ ว่า Axum มี fallback ของตัวเองเมื่อไม่มี route ไหนตรงกับ request — ค่า
default ของมันคือ `404 Not Found` พร้อม **body ว่างเปล่า** (ไม่ใช่ JSON) ซึ่งขัดกับหลักการ "ทุก error ตอบ
JSON shape เดียวกัน" ของบทนี้ — แก้ได้ง่าย ๆ ด้วย `.fallback(...)` ที่คืน `AppError::NotFound` ตรง ๆ:

```rust
# #[derive(Debug, thiserror::Error)]
# enum AppError { #[error("ไม่พบข้อมูลที่ต้องการ")] NotFound }
# impl axum::response::IntoResponse for AppError {
#     fn into_response(self) -> axum::response::Response { axum::http::StatusCode::NOT_FOUND.into_response() }
# }
// handler ไม่ต้องรับ parameter อะไรเลย และคืน AppError ตรงๆ (ไม่ใช่ Result) เพราะ AppError
// implement IntoResponse อยู่แล้ว -- Axum เรียก .into_response() ให้เองไม่ว่าจะคืนผ่าน Ok/Err หรือคืนตรงๆ
async fn app_fallback() -> AppError {
    AppError::NotFound
}
```

```rust
# use axum::Router;
# #[derive(Debug, thiserror::Error)]
# enum AppError { #[error("x")] NotFound }
# impl axum::response::IntoResponse for AppError {
#     fn into_response(self) -> axum::response::Response { axum::http::StatusCode::NOT_FOUND.into_response() }
# }
# async fn app_fallback() -> AppError { AppError::NotFound }
let app: Router<()> = Router::new()
    // .route(...) ของจริงทั้งหมดของระบบ
    .fallback(app_fallback);
```

ทดสอบด้วย path ที่ไม่มี route ไหนตรงเลย:

```bash
curl -sS -i http://127.0.0.1:3102/does/not/exist
```
```
HTTP/1.1 404 Not Found
content-type: application/json
content-length: 106

{"error":{"code":"NOT_FOUND","message":"ไม่พบข้อมูลที่ต้องการ"}}
```

path ที่ไม่มีอยู่จริงก็ยังได้ JSON shape เดียวกับ error อื่นทั้งระบบ — client ไม่ต้องเขียน special case
สำหรับ "404 ที่มาจาก routing" แยกจาก "404 ที่มาจาก resource ไม่เจอ" อีกต่อไป (ทั้งสองกรณีตอบเหมือนกันเป๊ะ
เพราะทั้งคู่คือ `AppError::NotFound` ตัวเดียวกัน)

### 66.9 Panic ใน handler: พฤติกรรม default เทียบกับ `CatchPanicLayer`

ทุกอย่างจนถึงตอนนี้คือ error ที่**คาดการณ์ไว้แล้ว** (เขียน `Err(AppError::...)` เอง) — แต่โปรแกรมจริงมี bug ที่
ทำให้ handler **panic** ได้เสมอ (index ออกนอกขอบ, `.unwrap()` ที่พลาด, arithmetic overflow ฯลฯ) คำถามสำคัญ
คือ: **panic ใน handler เดียวทำให้ทั้งเซิร์ฟเวอร์ล้มไปด้วยหรือไม่?** อย่าเดา — มาพิสูจน์ด้วยการรันจริง

#### พฤติกรรม default: ไม่มี `CatchPanicLayer`

```rust
use axum::{routing::get, Router};

async fn health() -> &'static str {
    "ok"
}

async fn boom() -> &'static str {
    panic!("จำลอง bug ร้ายแรงใน handler นี้");
}

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt().init();
    let app = Router::new()
        .route("/health", get(health))
        .route("/boom", get(boom));
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3103").await.unwrap();
    axum::serve(listener, app.into_make_service()).await.unwrap();
}
```

รันจริงแล้วยิง `/health` → `/boom` → `/health` อีกครั้งติดต่อกัน:

```bash
curl -sS -i http://127.0.0.1:3103/health
```
```
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 2

ok
```

```bash
curl -sS -i http://127.0.0.1:3103/boom
```
```
curl: (52) Empty reply from server
```

**ไม่ใช่ `500`! ไม่มี response เลยแม้แต่บรรทัดเดียว** — connection ถูกปิดกลางทางทันทีที่ handler panic ฝั่ง
client เห็นแค่ "Empty reply from server" (curl error code 52) เพราะ TCP connection ถูกตัดโดยไม่มี HTTP
response ใด ๆ ถูกส่งออกมาเลย — ลองดู `/health` อีกครั้งทันทีหลังจากนั้น:

```bash
curl -sS -i http://127.0.0.1:3103/health
```
```
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 2

ok
```

**เซิร์ฟเวอร์ยังตอบปกติ!** — คำตอบของคำถามข้างบนคือ: **panic ใน handler เดียวไม่ทำให้ทั้งเซิร์ฟเวอร์ล้ม แต่
ทำให้ connection ของ request นั้นถูกตัดทิ้งแบบไม่มี response กลับไปเลย** เหตุผลเชิงลึก (ต่อยอด Part 48-49
เรื่อง Tokio tasks): แต่ละ connection ที่ Axum/hyper รับเข้ามาถูกจัดการอยู่ใน **task** ของ Tokio ที่แยกจากกัน
และ Tokio runtime **ครอบทุก task ด้วย `std::panic::catch_unwind`** ที่ระดับ task boundary (เห็นได้จาก
`std::panicking::catch_unwind` ที่ปรากฏใน stack trace จริงของ panic ด้านล่าง) — เมื่อ task หนึ่ง panic,
Tokio **จับมันไว้ตรงนั้น ไม่ปล่อยให้ panic ลามข้ามไปยัง task อื่น** (task อื่นที่กำลังรัน connection อื่นอยู่
พร้อมกันจึงไม่ได้รับผลกระทบ, worker thread เดิมก็กลับไปรับงานต่อได้ปกติ) แต่**เพราะ task ที่ panic คือ task
ที่กำลังเขียน HTTP response กลับไปให้ client อยู่พอดี** เมื่อมันถูกฆ่ากลางทาง **ไม่มีใครเขียน response ต่อให้
เสร็จ** — connection จึงถูกปิดโดยไม่มี response ใด ๆ ส่งออกไปเลย

log ที่ server print ออกมา (stderr) ยืนยันว่า panic เกิดขึ้นจริงและ runtime จับมันไว้:

```
thread 'tokio-rt-worker' (9337) panicked at src/bin/panic_before.rs:8:5:
จำลอง bug ร้ายแรงใน handler นี้
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

**นี่คือปัญหาจริงสองข้อที่ต้องแก้**: (1) client ได้รับ connection error ที่ดูเหมือนเครือข่ายมีปัญหา ไม่ใช่
HTTP error ที่ปกติ ทำให้ debug ยากและ client library บางตัวอาจ retry ผิดวิธี และ (2) ไม่มี log ระดับ `ERROR`
ที่ tracing subscriber จัดรูปแบบให้ (มีแต่ raw panic message ที่พิมพ์ไปที่ stderr ตรง ๆ โดย Rust panic hook
เอง ไม่ผ่านระบบ `tracing` เลย) ทำให้ยากต่อการ monitor/alert แบบเดียวกับ error อื่นในระบบ

#### แก้ด้วย `tower_http::catch_panic::CatchPanicLayer`

`tower-http` มี middleware สำเร็จรูปที่แก้ปัญหานี้ตรง ๆ — ครอบ handler ด้วย `catch_unwind` อีกชั้นที่ระดับ
`tower::Service` (ก่อนที่ panic จะไปถึงจุดที่ทำลาย connection) แล้วแปลงมันเป็น `500 Internal Server Error`
ปกติ พร้อม log ผ่าน `tracing::error!`:

```toml
tower-http = { version = "0.7.1", features = ["catch-panic"] }
```

```rust
use axum::{routing::get, Router};
use tower_http::catch_panic::CatchPanicLayer;

async fn health() -> &'static str {
    "ok"
}

async fn boom() -> &'static str {
    panic!("จำลอง bug ร้ายแรงใน handler นี้");
}

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt().init();
    let app = Router::new()
        .route("/health", get(health))
        .route("/boom", get(boom))
        .layer(CatchPanicLayer::new()); // เพิ่มบรรทัดเดียว -- ไม่ต้องแก้ handler เลย
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3104").await.unwrap();
    axum::serve(listener, app.into_make_service()).await.unwrap();
}
```

รันจริงด้วยลำดับ curl เดียวกันเป๊ะ:

```bash
curl -sS -i http://127.0.0.1:3104/boom
```

ผลลัพธ์จริง — **คราวนี้ได้ HTTP response ปกติ**:

```
HTTP/1.1 500 Internal Server Error
content-type: text/plain; charset=utf-8
content-length: 16

Service panicked
```

```bash
curl -sS -i http://127.0.0.1:3104/health   # ยืนยันว่าเซิร์ฟเวอร์ยังทำงานปกติเหมือนเดิม
```
```
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 2

ok
```

และ log ที่ได้ (เห็นทั้ง panic message ดิบจาก Rust panic hook **และ** log ที่ `CatchPanicLayer` เขียนผ่าน
`tracing::error!` ให้อัตโนมัติ):

```
thread 'tokio-rt-worker' (9570) panicked at src/bin/panic_after.rs:9:5:
จำลอง bug ร้ายแรงใน handler นี้
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
2026-09-27T00:07:08.766952Z ERROR tower_http::catch_panic: Service panicked: จำลอง bug ร้ายแรงใน handler นี้
```

สังเกตว่า `CatchPanicLayer` log ผ่าน target `tower_http::catch_panic` (ไม่ใช่ target ของแอปเรา) ที่ระดับ
`ERROR` — เชื่อมกับ Part 60 เรื่องการกรอง log ด้วย `RUST_LOG`: ถ้าตั้ง `RUST_LOG=my_app=info` เฉย ๆ แบบเดียว
กับกับดักคลาสสิกของ `TraceLayer` ใน Part 65 หัวข้อ "กับดักที่พบบ่อย" ข้อ 1 คุณจะ**ยังเห็น** log นี้ (เพราะเป็น
ระดับ `ERROR` ซึ่งสูงกว่า `info` เสมอ ไม่เหมือนกรณี `TraceLayer` ที่ default เป็น `DEBUG`) — แต่ยังต้องระบุ
`tower_http=error` (หรือกว้างกว่า) ใน `RUST_LOG` ถ้าคุณตั้ง filter แบบเจาะจงทุก target ไว้

**Body `"Service panicked"` เป็นข้อความทั่วไปตายตัว ไม่มีรายละเอียดของ panic message จริงเลย** (ต่างจาก log
ที่มีข้อความ panic ตัวจริง `"จำลอง bug ร้ายแรงใน handler นี้"` ให้ทีมพัฒนาเห็น) — สอดคล้องกับหลักการ "log เก็บ
ทุกอย่าง, response บอกแค่พอ" ที่หัวข้อ 66.5 อธิบายไว้ทุกประการ แม้จะเป็น middleware สำเร็จรูปจาก `tower-http`
ไม่ใช่โค้ดที่เราเขียนเองก็ตาม — ถ้าต้องการให้ response ของ panic ตอบเป็น JSON shape เดียวกับ `AppError` อื่น
ทั้งระบบด้วย `CatchPanicLayer` มี `.custom_response(...)` ให้ปรับแต่ง response ที่คืนได้ (รับ panic payload
เป็น `Box<dyn Any + Send>` แล้วให้เราแปลงเป็น `Response` เอง) — รายละเอียดอยู่นอกสโคปของบทนี้ แต่หลักการเดียว
กับที่ `impl IntoResponse for AppError` ทำก็นำไปประยุกต์ใช้ได้ตรง ๆ

**สรุปเปรียบเทียบชัด ๆ:**

| | ไม่มี `CatchPanicLayer` | มี `CatchPanicLayer` |
|---|---|---|
| Client เห็นอะไร | connection ถูกตัด (`curl: (52) Empty reply from server`) | `500 Internal Server Error` พร้อม body ปกติ |
| Log | raw panic message ไปที่ stderr ตรงๆ ไม่ผ่าน `tracing` | log ผ่าน `tracing::error!` ที่ target `tower_http::catch_panic` |
| เซิร์ฟเวอร์ตัวอื่น (connection อื่น) กระทบไหม | ไม่กระทบ (Tokio task isolation ป้องกันไว้อยู่แล้ว) | ไม่กระทบเหมือนกัน |
| ต้องแก้ handler ไหม | - | ไม่ต้อง แค่เพิ่ม `.layer(CatchPanicLayer::new())` |

**ข้อสรุปสำคัญที่ต้องจำ**: panic ไม่เคยทำให้ "เซิร์ฟเวอร์ทั้งตัวล้ม" ในทั้งสองกรณี (ต่างจากความเข้าใจผิดที่พบ
บ่อยว่า "panic เดียวพังทั้งเซิร์ฟเวอร์") — สิ่งที่ `CatchPanicLayer` แก้ไม่ใช่ "การป้องกันไม่ให้เซิร์ฟเวอร์ล้ม"
(มันไม่ล้มอยู่แล้วโดย design ของ Tokio) แต่คือ **การเปลี่ยน "connection ที่ถูกตัดอย่างเงียบ ๆ" ให้กลายเป็น
"HTTP response ที่ถูกต้องตามมาตรฐาน พร้อม log ที่ตรวจสอบได้"** — ซึ่งสำคัญมากสำหรับ production เพราะ client
จริง (browser, mobile app, service อื่น) คาดหวัง HTTP response เสมอ ไม่ใช่ connection ที่ถูกตัดกลางทางแบบไม่
มีคำอธิบาย

### 66.10 Capstone: Library API พร้อม `AppError` แบบสมบูรณ์

มาประกอบทุกหัวข้อของบทนี้เข้ากับระบบห้องสมุด (library API) ที่ Part 63-65 วางโครงไว้ต่อเนื่องกันมา —
capstone นี้มี `AppError` เดียวครอบคลุมทุกเส้นทางความล้มเหลว, request id ผ่าน tracing span, custom `AppJson`
extractor, fallback ที่สอดคล้องกับระบบ, และ `CatchPanicLayer` ครบทุกจุด:

```rust
use axum::{
    extract::{rejection::JsonRejection, FromRequest, Path, State},
    http::StatusCode,
    middleware::Next,
    response::{IntoResponse, Response},
    routing::{get, post},
    Json, Router,
};
use serde::{Deserialize, Serialize};
use serde_json::json;
use std::collections::HashMap;
use std::sync::{
    atomic::{AtomicU64, Ordering},
    Arc, Mutex,
};
use thiserror::Error;
use tower_http::catch_panic::CatchPanicLayer;
use tracing::Instrument;

// ============================== AppError: จุดรวมของทั้งระบบ ==============================

#[derive(Debug, Clone, Serialize)]
struct FieldError {
    field: String,
    message: String,
}

#[derive(Debug, Error)]
enum AppError {
    #[error("ไม่พบหนังสือที่ต้องการ")]
    NotFound,
    #[error("ข้อมูลไม่ถูกต้อง: {0}")]
    Validation(String),
    #[error("ข้อมูลไม่ผ่านการตรวจสอบ")]
    ValidationErrors(Vec<FieldError>),
    #[error("ขัดแย้งกับสถานะปัจจุบัน: {0}")]
    Conflict(String),
    #[error("เกิดข้อผิดพลาดภายในระบบ")]
    Internal(#[from] anyhow::Error),
}

impl From<JsonRejection> for AppError {
    fn from(rejection: JsonRejection) -> Self {
        AppError::Validation(rejection.body_text())
    }
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, code) = match &self {
            AppError::NotFound => (StatusCode::NOT_FOUND, "NOT_FOUND"),
            AppError::Validation(_) => (StatusCode::BAD_REQUEST, "VALIDATION_ERROR"),
            AppError::ValidationErrors(_) => (StatusCode::BAD_REQUEST, "VALIDATION_ERROR"),
            AppError::Conflict(_) => (StatusCode::CONFLICT, "CONFLICT"),
            AppError::Internal(_) => (StatusCode::INTERNAL_SERVER_ERROR, "INTERNAL_ERROR"),
        };

        match &self {
            AppError::Internal(source) => {
                tracing::error!(error_code = code, status = status.as_u16(), error = %source, "internal error");
            }
            other => {
                tracing::warn!(error_code = code, status = status.as_u16(), error = %other, "request ล้มเหลว");
            }
        }

        let body = match self {
            AppError::ValidationErrors(fields) => json!({
                "error": { "code": code, "message": "ข้อมูลไม่ผ่านการตรวจสอบ", "fields": fields }
            }),
            other => json!({
                "error": { "code": code, "message": other.to_string() }
            }),
        };

        (status, Json(body)).into_response()
    }
}

#[derive(FromRequest)]
#[from_request(via(axum::Json), rejection(AppError))]
struct AppJson<T>(T);

// ============================== Domain: Library ==============================

#[derive(Debug, Clone, Serialize)]
struct Book {
    id: u32,
    title: String,
    author: String,
    isbn: String,
    available: bool,
}

#[derive(Debug, Deserialize)]
struct NewBook {
    title: String,
    author: String,
    isbn: String,
}

fn validate_new_book(input: &NewBook) -> Result<(), AppError> {
    let mut errors = Vec::new();
    if input.title.trim().is_empty() {
        errors.push(FieldError { field: "title".into(), message: "ต้องไม่เป็นค่าว่าง".into() });
    }
    if input.author.trim().is_empty() {
        errors.push(FieldError { field: "author".into(), message: "ต้องไม่เป็นค่าว่าง".into() });
    }
    if input.isbn.len() != 13 || !input.isbn.chars().all(|c| c.is_ascii_digit()) {
        errors.push(FieldError { field: "isbn".into(), message: "ต้องเป็นตัวเลข 13 หลัก".into() });
    }
    if errors.is_empty() { Ok(()) } else { Err(AppError::ValidationErrors(errors)) }
}

struct AppState {
    books: Mutex<HashMap<u32, Book>>,
    next_id: Mutex<u32>,
}

type SharedState = Arc<AppState>;

fn seed_books() -> HashMap<u32, Book> {
    let mut m = HashMap::new();
    m.insert(1, Book {
        id: 1, title: "The Rust Programming Language".to_string(),
        author: "Steve Klabnik".to_string(), isbn: "9781593278281".to_string(), available: true,
    });
    m.insert(2, Book {
        id: 2, title: "Zero To Production In Rust".to_string(),
        author: "Luca Palmieri".to_string(), isbn: "9798512359590".to_string(), available: true,
    });
    // book id 42 มีไว้เพื่อจำลอง "internal error" ตอนยืม (audit-log service ล่ม) โดยเฉพาะ
    m.insert(42, Book {
        id: 42, title: "Rust for Rustaceans".to_string(),
        author: "Jon Gjengset".to_string(), isbn: "9781718501850".to_string(), available: true,
    });
    m
}

fn write_audit_log(book_id: u32, action: &str) -> anyhow::Result<()> {
    if book_id == 42 {
        anyhow::bail!("ไม่สามารถเชื่อมต่อ audit-log service ได้ (connection refused)");
    }
    tracing::debug!(book_id, action, "เขียน audit log สำเร็จ");
    Ok(())
}

async fn list_books(State(state): State<SharedState>) -> Json<Vec<Book>> {
    let books = state.books.lock().unwrap();
    Json(books.values().cloned().collect())
}

async fn get_book(State(state): State<SharedState>, Path(id): Path<u32>) -> Result<Json<Book>, AppError> {
    let books = state.books.lock().unwrap();
    let book = books.get(&id).cloned().ok_or(AppError::NotFound)?;
    Ok(Json(book))
}

async fn create_book(
    State(state): State<SharedState>,
    AppJson(new_book): AppJson<NewBook>,
) -> Result<(StatusCode, Json<Book>), AppError> {
    validate_new_book(&new_book)?;

    let mut next_id = state.next_id.lock().unwrap();
    let id = *next_id;
    *next_id += 1;

    let book = Book {
        id, title: new_book.title, author: new_book.author, isbn: new_book.isbn, available: true,
    };
    state.books.lock().unwrap().insert(id, book.clone());
    Ok((StatusCode::CREATED, Json(book)))
}

async fn borrow_book(State(state): State<SharedState>, Path(id): Path<u32>) -> Result<Json<Book>, AppError> {
    let mut books = state.books.lock().unwrap();
    let book = books.get_mut(&id).ok_or(AppError::NotFound)?;

    if !book.available {
        return Err(AppError::Conflict(format!("หนังสือ '{}' ถูกยืมไปแล้ว", book.title)));
    }

    book.available = false;
    let title = book.title.clone();
    drop(books);

    write_audit_log(id, "borrow")?; // book id 42 -> AppError::Internal

    tracing::info!(book_id = id, title = %title, "ยืมหนังสือสำเร็จ");
    let books = state.books.lock().unwrap();
    Ok(Json(books.get(&id).unwrap().clone()))
}

// เผื่อ bug จริง: index ที่ "ไม่ควรเกิดขึ้น" ทำให้ panic เพื่อพิสูจน์ CatchPanicLayer ใน capstone
async fn crash_demo(Path(id): Path<u32>) -> Json<serde_json::Value> {
    let v: Vec<i32> = vec![1, 2, 3];
    let out_of_bounds = v[id as usize]; // panic ถ้า id >= 3
    Json(json!({ "value": out_of_bounds }))
}

async fn app_fallback() -> AppError {
    AppError::NotFound
}

// ============================== Middleware: Request ID ผ่าน tracing span ==============================

static REQUEST_COUNTER: AtomicU64 = AtomicU64::new(1);

async fn request_id_middleware(req: axum::extract::Request, next: Next) -> Response {
    let id = REQUEST_COUNTER.fetch_add(1, Ordering::Relaxed);
    let request_id = format!("req-{id}");
    let span = tracing::info_span!("request", request_id = %request_id);
    next.run(req).instrument(span).await
}

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt()
        .with_env_filter(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| tracing_subscriber::EnvFilter::new("capstone=debug,tower_http=info")),
        )
        .init();

    let state: SharedState = Arc::new(AppState {
        books: Mutex::new(seed_books()),
        next_id: Mutex::new(3),
    });

    let app = Router::new()
        .route("/books", get(list_books).post(create_book))
        .route("/books/{id}", get(get_book))
        .route("/books/{id}/borrow", post(borrow_book))
        .route("/books/{id}/crash", get(crash_demo))
        .fallback(app_fallback)
        .layer(axum::middleware::from_fn(request_id_middleware))
        .layer(CatchPanicLayer::new())
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3105").await.unwrap();
    tracing::info!("capstone library API ฟังอยู่ที่ http://127.0.0.1:3105");
    axum::serve(listener, app.into_make_service()).await.unwrap();
}
```

โค้ดนี้ผ่าน `cargo build` จริงโดยไม่มี error เลย (ทดสอบด้วย axum 0.8.9, tower-http 0.7.1, thiserror 2.0.21,
anyhow 1.0.104) มาทดสอบ end-to-end ด้วย `curl` จริงครบทุกเส้นทางความล้มเหลว ตามลำดับที่รันจริง:

**1) List หนังสือทั้งหมด (path สำเร็จปกติ):**

```bash
curl -sS -i http://127.0.0.1:3105/books
```
```
HTTP/1.1 200 OK
content-type: application/json
content-length: 330

[{"id":2,"title":"Zero To Production In Rust","author":"Luca Palmieri","isbn":"9798512359590","available":true},{"id":42,"title":"Rust for Rustaceans","author":"Jon Gjengset","isbn":"9781718501850","available":true},{"id":1,"title":"The Rust Programming Language","author":"Steve Klabnik","isbn":"9781593278281","available":true}]
```

**2) หาหนังสือไม่เจอ → `AppError::NotFound`:**

```bash
curl -sS -i http://127.0.0.1:3105/books/999
```
```
HTTP/1.1 404 Not Found
content-type: application/json
content-length: 109

{"error":{"code":"NOT_FOUND","message":"ไม่พบหนังสือที่ต้องการ"}}
```

**3) สร้างหนังสือด้วยข้อมูลถูกต้อง:**

```bash
curl -sS -i -X POST http://127.0.0.1:3105/books \
  -H 'Content-Type: application/json' \
  -d '{"title":"Programming Rust","author":"Jim Blandy","isbn":"9781492052593"}'
```
```
HTTP/1.1 201 Created
content-type: application/json
content-length: 97

{"id":3,"title":"Programming Rust","author":"Jim Blandy","isbn":"9781492052593","available":true}
```

**4) สร้างหนังสือด้วยข้อมูลผิดหลายจุด → `AppError::ValidationErrors`:**

```bash
curl -sS -i -X POST http://127.0.0.1:3105/books \
  -H 'Content-Type: application/json' \
  -d '{"title":"","author":"","isbn":"xx"}'
```
```
HTTP/1.1 400 Bad Request
content-type: application/json
content-length: 389

{"error":{"code":"VALIDATION_ERROR","fields":[{"field":"title","message":"ต้องไม่เป็นค่าว่าง"},{"field":"author","message":"ต้องไม่เป็นค่าว่าง"},{"field":"isbn","message":"ต้องเป็นตัวเลข 13 หลัก"}],"message":"ข้อมูลไม่ผ่านการตรวจสอบ"}}
```

**5) ยืมหนังสือสำเร็จ:**

```bash
curl -sS -i -X POST http://127.0.0.1:3105/books/1/borrow
```
```
HTTP/1.1 200 OK
content-type: application/json
content-length: 114

{"id":1,"title":"The Rust Programming Language","author":"Steve Klabnik","isbn":"9781593278281","available":false}
```

**6) ยืมซ้ำ → `AppError::Conflict`:**

```bash
curl -sS -i -X POST http://127.0.0.1:3105/books/1/borrow
```
```
HTTP/1.1 409 Conflict
content-type: application/json
content-length: 203

{"error":{"code":"CONFLICT","message":"ขัดแย้งกับสถานะปัจจุบัน: หนังสือ 'The Rust Programming Language' ถูกยืมไปแล้ว"}}
```

**7) ยืมหนังสือ id 42 → audit-log service ล่ม → `AppError::Internal`:**

```bash
curl -sS -i -X POST http://127.0.0.1:3105/books/42/borrow
```
```
HTTP/1.1 500 Internal Server Error
content-type: application/json
content-length: 117

{"error":{"code":"INTERNAL_ERROR","message":"เกิดข้อผิดพลาดภายในระบบ"}}
```

**8) `GET /books/10/crash` → panic จริง → `CatchPanicLayer` จับไว้:**

```bash
curl -sS -i http://127.0.0.1:3105/books/10/crash
```
```
HTTP/1.1 500 Internal Server Error
content-type: text/plain; charset=utf-8
content-length: 16

Service panicked
```

**9) Path ที่ไม่มี route ไหนตรงเลย → fallback:**

```bash
curl -sS -i http://127.0.0.1:3105/no/such/route
```
```
HTTP/1.1 404 Not Found
content-type: application/json
content-length: 109

{"error":{"code":"NOT_FOUND","message":"ไม่พบหนังสือที่ต้องการ"}}
```

**10) เซิร์ฟเวอร์ยังทำงานปกติหลังจาก panic ในข้อ 8:**

```bash
curl -sS -i http://127.0.0.1:3105/books/1
```
```
HTTP/1.1 200 OK
content-type: application/json
content-length: 114

{"id":1,"title":"The Rust Programming Language","author":"Steve Klabnik","isbn":"9781593278281","available":false}
```

และ log ทั้งหมดที่เกิดขึ้นตลอดการทดสอบ (คัดลอกตรงจากการรันจริง เรียงตามลำดับเวลา — ตัด ANSI color code ออก):

```
2026-09-27T00:08:18.707436Z  INFO capstone: capstone library API ฟังอยู่ที่ http://127.0.0.1:3105
2026-09-27T00:08:28.701672Z  WARN request{request_id=req-2}: capstone: request ล้มเหลว error_code="NOT_FOUND" status=404 error=ไม่พบหนังสือที่ต้องการ
2026-09-27T00:08:28.714456Z  WARN request{request_id=req-4}: capstone: request ล้มเหลว error_code="VALIDATION_ERROR" status=400 error=ข้อมูลไม่ผ่านการตรวจสอบ
2026-09-27T00:08:28.720734Z DEBUG request{request_id=req-5}: capstone: เขียน audit log สำเร็จ book_id=1 action="borrow"
2026-09-27T00:08:28.720769Z  INFO request{request_id=req-5}: capstone: ยืมหนังสือสำเร็จ book_id=1 title=The Rust Programming Language
2026-09-27T00:08:28.726287Z  WARN request{request_id=req-6}: capstone: request ล้มเหลว error_code="CONFLICT" status=409 error=ขัดแย้งกับสถานะปัจจุบัน: หนังสือ 'The Rust Programming Language' ถูกยืมไปแล้ว
2026-09-27T00:08:28.731959Z ERROR request{request_id=req-7}: capstone: internal error error_code="INTERNAL_ERROR" status=500 error=ไม่สามารถเชื่อมต่อ audit-log service ได้ (connection refused)

thread 'tokio-rt-worker' (10629) panicked at src/bin/capstone.rs:222:26:
index out of bounds: the len is 3 but the index is 10
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
2026-09-27T00:08:28.737756Z ERROR tower_http::catch_panic: Service panicked: index out of bounds: the len is 3 but the index is 10
2026-09-27T00:08:28.743095Z  WARN request{request_id=req-9}: capstone: request ล้มเหลว error_code="NOT_FOUND" status=404 error=ไม่พบหนังสือที่ต้องการ
```

สังเกตความสอดคล้องทั้งระบบที่จับต้องได้ทุกจุด: **ทุก client error (404/400/409) log ที่ระดับ `WARN`**, **ทุก
internal error (500 จาก `AppError::Internal`) log ที่ระดับ `ERROR` พร้อมรายละเอียดจริง** (`ไม่สามารถเชื่อมต่อ
audit-log service ได้ (connection refused)` — แต่ client เห็นแค่ข้อความทั่วไปในข้อ 7), **panic ก็ log ที่
ระดับ `ERROR` เหมือนกัน** (ผ่าน `tower_http::catch_panic`) แม้จะเป็นความล้มเหลวที่คนละกลไกกันเลย (`AppError`
เทียบกับ panic) — และ **request id (`req-N`) ไล่ทุก request ตามลำดับ** รวมถึง request ที่สำเร็จ (ไม่มี log
แต่ counter ก็เพิ่มด้วย เห็นได้จาก `req-2`, `req-4`, `req-5`, ... ที่ไม่ต่อเนื่องกันเพราะ request ที่สำเร็จ
ระหว่างนั้นไม่ log อะไรแต่ counter ยังเพิ่มขึ้นตามจริง) นี่คือ error handling ที่ **สอดคล้องกันทั้งระบบจริง**
ตามเป้าหมายที่วางไว้ตั้งแต่หัวข้อ 66.1

## กับดักที่พบบ่อย (Common Pitfalls)

**1. ลืม `#[from]` บน field ของ variant ที่ใช้กับ `?` — compiler ฟ้อง `E0277` แบบเดียวกับ Part 30**

```rust
# use thiserror::Error;
#[derive(Debug, Error)]
enum AppError {
    #[error("ไม่พบข้อมูลที่ต้องการ")]
    NotFound,
    // ลืม #[from] -- เขียนแค่ field ธรรมดา
    #[error("เกิดข้อผิดพลาดภายในระบบ")]
    Internal(anyhow::Error),
}
```

```rust
# use thiserror::Error;
# #[derive(Debug, Error)]
# enum AppError { #[error("x")] Internal(anyhow::Error) }
fn risky_operation() -> anyhow::Result<()> { anyhow::bail!("boom") }

async fn handler() -> Result<&'static str, AppError> {
    risky_operation()?; // ? ต้องแปลง anyhow::Error -> AppError แต่ไม่มี impl From ให้
    Ok("ok")
}
```

error จริงจาก compiler (เหมือนกับที่ Part 30 หัวข้อ 30.4 อธิบายไว้เป๊ะ เพียงแต่คราวนี้เกิดกับ `AppError` ของ
เว็บแอปจริง):

```
error[E0277]: `?` couldn't convert the error to `AppError`
  --> src/bin/pitfall_missing_from.rs:29:22
   |
29 |     risky_operation()?; // ? ต้องแปลง anyhow::Error -> AppError แต่ไม่มี impl From ให้
   |     -----------------^ the trait `From<anyhow::Error>` is not implemented for `AppError`
   |     |
   |     this can't be annotated with `?` because it has type `Result<_, anyhow::Error>`
   |
note: `AppError` needs to implement `From<anyhow::Error>`
```

วิธีแก้: เติม `#[from]` หน้า field (`Internal(#[from] anyhow::Error)`) ตามที่ Part 31 หัวข้อ 31.4 สอนไว้ —
`thiserror` จะ generate `impl From<anyhow::Error> for AppError` ให้อัตโนมัติ

**2. ใส่ placeholder `{0}` ในข้อความของ `Internal` variant — รั่วรายละเอียดภายในระบบไปให้ client เห็นตรง ๆ**

นี่คือกับดักด้าน**ความปลอดภัย**ที่อันตรายที่สุดของบทนี้ และตรวจจับได้ยากเพราะ**compile ผ่านสบาย ๆ ไม่มี
warning เลย**:

```rust
# use thiserror::Error;
#[derive(Debug, Error)]
enum AppError {
    // ผิด! ใส่ {0} เข้าไป -- ทำให้ Display ของ Internal แสดงเนื้อหาจริงของ anyhow::Error ที่ห่ออยู่
    #[error("เกิดข้อผิดพลาดภายในระบบ: {0}")]
    Internal(#[from] anyhow::Error),
}
```

ทดสอบด้วย error ที่มีรายละเอียดจริง (`anyhow::Context` แบบเดียวกับหัวข้อ 66.5) แล้วยิง `curl` จริง:

```bash
curl -sS -i http://127.0.0.1:3101/demo/internal
```

ผลลัพธ์จริง — **รายละเอียดภายใน (`database pool`) รั่วออกไปให้ client เห็นตรง ๆ**:

```
HTTP/1.1 500 Internal Server Error
content-type: application/json
content-length: 197

{"error":{"code":"INTERNAL_ERROR","message":"เกิดข้อผิดพลาดภายในระบบ: ไม่สามารถขอ connection จาก database pool ได้"}}
```

เทียบกับเวอร์ชันที่ถูกต้อง (ไม่มี `{0}`) ในหัวข้อ 66.5 ที่ client เห็นแค่ `"เกิดข้อผิดพลาดภายในระบบ"` — ข้อความ
`"database pool"` บอก attacker ว่าระบบใช้ connection pool จริง (เดา stack เทคโนโลยีได้ง่ายขึ้น) และในระบบจริง
ที่ error message อาจมี connection string, path บนเครื่อง, หรือ query SQL เต็ม ๆ ความเสียหายจะรุนแรงกว่านี้
มาก — วิธีป้องกัน: **ทบทวน `#[error("...")]` ของทุก variant ที่ห่อ error จากภายนอก (`#[from]`) อย่างเคร่งครัด
ว่ามี placeholder ที่ interpolate ข้อมูลจาก error ต้นทางหรือไม่** ถ้ามีและ variant นั้นถูกส่งกลับเป็น HTTP
response ตรง ๆ (ไม่ผ่านการกรองใด ๆ) ให้ลบ placeholder ออก แล้วพึ่งพา `tracing::error!` ในการเก็บรายละเอียด
เต็มไว้ที่ log แทนเสมอ

**3. ลำดับ extractor ผิด (`Json<T>`/`AppJson<T>` ไม่ได้อยู่ตัวสุดท้าย) — error message งงมากถ้าไม่มี
`#[axum::debug_handler]`**

```
error[E0277]: the trait bound `fn(Json<NewBook>, State<...>) -> ... {broken_order}: Handler<_, _>` is not satisfied
   --> src/bin/debug_handler_before.rs:27:31
    |
 27 |         .route("/books", post(broken_order))
    |                          ---- ^^^^^^^^^^^^ unsatisfied trait bound
    = help: the trait `Handler<_, _>` is not implemented for fn item `fn(Json<NewBook>, State<Arc<AppState>>) -> impl Future<Output = &'static str> {broken_order}`
    = note: Consider using `#[axum::debug_handler]` to improve the error message
```

error นี้ไม่บอกสาเหตุตรง ๆ เลย — วิธีแก้ตามที่หัวข้อ 66.8 พิสูจน์ไว้: เพิ่ม `#[axum::debug_handler]` เข้าไปที่
handler แล้ว compile ใหม่ จะได้ error ที่บอกตรง ๆ ว่า `` `Json<_>` consumes the request body and thus must be
the last argument to the handler function `` แล้วย้าย extractor ที่กิน body ไปไว้ **parameter สุดท้าย** เสมอ

**4. ลืมว่า `RUST_BACKTRACE=1` ทำให้ log ของ `error_debug = ?source` บวมด้วย stack trace เต็มรูปแบบ**

ถ้า environment ตั้ง `RUST_BACKTRACE=1` ไว้ (พบได้บ่อยใน container/CI ที่ตั้งไว้เพื่อ debug ปัญหาอื่น) การ log
`?source` (Debug ของ `anyhow::Error`) ใน `impl IntoResponse for AppError` จะพ่วง **stack backtrace เต็มความ
ยาวหลายสิบบรรทัด** ต่อทุก internal error หนึ่งครั้ง (ตัวอย่างที่พบจริงระหว่างเขียนบทนี้ มีมากกว่า 100 บรรทัด
ต่อ error เดียว รวม path เต็มของ registry cache บนเครื่อง) — สิ่งนี้ไม่ผิดในเชิงความปลอดภัย (อยู่ใน log ฝั่ง
server เท่านั้น ไม่หลุดไปที่ client) แต่**ทำให้ log ที่ค้นหา/เก็บยากขึ้นมากในระบบ production จริง** (ปริมาณ log
พุ่งขึ้นหลายเท่า ค่าใช้จ่ายของ log aggregation service เพิ่มตาม และหา event ที่ต้องการยากขึ้นเพราะ backtrace
บังบรรทัดอื่น) — ทางเลือกที่เหมาะกับ production: ตั้ง `RUST_BACKTRACE=0` (หรือไม่ตั้งเลย) ตามค่า default แล้ว
เปิดเฉพาะตอน debug ปัญหาเจาะจงชั่วคราว หรือ log แค่ `%source` (Display อย่างเดียว ไม่มี backtrace) แล้วพึ่งพา
`.context()` ของ `anyhow` (ตาม Part 31 หัวข้อ 31.12) สร้าง error chain ที่มีรายละเอียดพอแล้วโดยไม่ต้องพึ่ง
backtrace ดิบเลย

**5. คิดว่า panic ใน handler ทำให้ "ทั้งเซิร์ฟเวอร์ล้ม" แล้วไม่ใส่ `CatchPanicLayer` เพราะคิดว่า "เดี๋ยว Tokio
จัดการให้เอง"**

ทั้งสองความเข้าใจผิดพลาดคนละแบบแต่มาบรรจบที่ผลลัพธ์เดียวกัน (การไม่ป้องกัน panic อย่างเหมาะสม): หัวข้อ 66.9
พิสูจน์แล้วว่า panic **ไม่ทำให้เซิร์ฟเวอร์ล้ม** (Tokio task isolation ป้องกันไว้อยู่แล้วโดย design) — แต่ถ้า
ไม่มี `CatchPanicLayer` ผลลัพธ์ที่ client ได้คือ **`curl: (52) Empty reply from server`** (connection ถูกตัด
กลางทางแบบไม่มี HTTP response เลย) ไม่ใช่ `500` ตามที่หลายคนเข้าใจผิดว่า "Rust/Tokio จัดการ panic ให้เป็น 500
โดยอัตโนมัติอยู่แล้ว" — ถ้า production API ของคุณไม่มี `CatchPanicLayer` (หรือ mechanism เทียบเท่า) ครอบอยู่
ทุก route, bug เล็ก ๆ อย่าง index ที่คำนวณผิดพลาดในบางเงื่อนไข (edge case ที่ทดสอบไม่ครอบคลุม) จะทำให้ client
จริงเจอ connection error ที่ดูเหมือนเครือข่ายมีปัญหา ทำให้ debug ยากขึ้นมากและ client library หลายตัว retry
ผิดวิธี (เข้าใจว่าเป็น network flake ไม่ใช่ server bug) — เพิ่ม `.layer(CatchPanicLayer::new())` ให้ครอบทุก
route ตั้งแต่วันแรกของ production API เสมอ

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เพิ่ม variant `AppError::RateLimited` เข้า enum ที่หัวข้อ 66.2 ออกแบบไว้ ผูกกับ
   `StatusCode::TOO_MANY_REQUESTS` (`429`) และ `error_code = "RATE_LIMITED"` เขียน handler ทดสอบที่คืน
   variant นี้ตรง ๆ แล้วยิง `curl -i` พิสูจน์ว่าได้ JSON shape เดียวกับ error อื่นทั้งระบบ (`{"error":
   {"code": "RATE_LIMITED", "message": "..."}}`)
   - Hint: เพิ่ม arm ใน**ทั้งสอง** `match` ภายใน `impl IntoResponse for AppError` (จับคู่ status/code และจับคู่
     สร้าง body) — ลืมจุดใดจุดหนึ่งจะทำให้ compiler ฟ้อง "non-exhaustive patterns" ทันที (ตามหลัก
     exhaustiveness checking ที่ Part 30 หัวข้อ 30.3 อธิบายไว้)

2. **(กลาง)** เพิ่ม header `Retry-After: 5` เข้า response ของ `AppError::RateLimited` ที่สร้างในข้อ 1 (โดย
   ที่ variant อื่นไม่มี header นี้เลย) แล้วพิสูจน์ด้วย `curl -i` ว่า header ปรากฏเฉพาะ variant นี้เท่านั้น
   - Hint: `impl IntoResponse for AppError` คืน `Response` เต็มรูปแบบอยู่แล้ว — เรียก
     `.headers_mut().insert(...)` บน `Response` ที่ได้จาก `(status, Json(body)).into_response()` ก่อน
     `return` ออกไป (เหมือนที่ Part 63 หัวข้อ 63.8 ทำกับ `CreatedTicket`) แต่ต้องทำ**ก่อน**สร้าง body สำหรับ
     variant อื่น หรือใส่เงื่อนไข `if matches!(...)` แยกออกมาให้ชัด

3. **(ยาก)** เพิ่ม endpoint `DELETE /books/{id}` ในระบบห้องสมุดจากหัวข้อ 66.10 ที่ลบหนังสือได้ **เฉพาะเมื่อ
   `available == true`** (ห้ามลบหนังสือที่มีคนยืมอยู่ — ต้องตอบ `AppError::Conflict`) เขียน handler ที่ chain
   `?` ครบสามจุด: หาไม่เจอ (`NotFound`), มีคนยืมอยู่ (`Conflict`), และลบสำเร็จ (เขียน audit log ด้วย
   `write_audit_log(id, "delete")?` เหมือนหัวข้อ 66.4) แล้วทดสอบทั้งสาม path ด้วย `curl` พร้อมดู log ที่ได้
   ว่ามี `request_id` ครบทุกบรรทัดจริง
   - Hint: ใช้ `HashMap::remove(&id)` แทน `get_mut` ตอนลบจริง แต่ต้องเช็ค `available` **ก่อน** `remove` เสมอ
     (ถ้า `remove` ไปแล้วค่อยเช็คจะสายเกินไป เพราะข้อมูลหายไปจาก `HashMap` แล้ว)

4. **(ยากมาก / ประยุกต์ใช้งานจริง)** ปรับ `CatchPanicLayer` ในหัวข้อ 66.9/66.10 ให้ตอบ **JSON shape เดียวกับ
   `AppError::Internal`** แทนข้อความ `"Service panicked"` แบบ plain text (ใช้
   `CatchPanicLayer::custom(...)` หรือ `.custom_response(...)` ตามที่เอกสารของ `tower_http::catch_panic`
   ให้ไว้ — อ่าน panic payload เป็น `Box<dyn std::any::Any + Send>` แล้ว downcast เป็น `&str`/`String` ถ้าทำ
   ได้ เพื่อ log ข้อความ panic จริงผ่าน `tracing::error!` เหมือนที่ `AppError::Internal` ทำ) พิสูจน์ด้วย
   `curl -i` ว่า response ของ panic ตอนนี้มี `content-type: application/json` และ shape
   `{"error": {"code": "INTERNAL_ERROR", "message": "เกิดข้อผิดพลาดภายในระบบ"}}` เหมือนกับ error 500 ปกติ
   ทุกประการ — ระบบตอนนี้จะสอดคล้องกัน **100%** ไม่มี response ไหนหลุด shape เลยแม้แต่กรณี panic
   - Hint: `Box<dyn Any + Send>::downcast_ref::<&str>()` และ `::downcast_ref::<String>()` ทั้งสองแบบต้องลอง
     (panic message จาก `panic!("...")` เป็น `&str` แต่บาง crate อาจ panic ด้วย `String` ที่สร้างจาก
     `format!()`) — ทบทวนเรื่อง `downcast_ref` จาก Part 30 หัวข้อ 30.1 ถ้าจำไม่ได้

## สรุป

บทนี้ปิดชุด Axum (Part 62-66) ด้วยการรวมทุกอย่างที่หลักสูตรสอนเรื่อง error handling มาตั้งแต่ Part 12
(`Result`/`?`), Part 30 (custom error enum/`From`/error chain), และ Part 31 (`thiserror`/`anyhow`) เข้ากับ
Axum อย่างเป็นระบบ ผ่าน **`AppError`** — enum เดียวที่เป็นจุดศูนย์กลางของทุก error ในแอป — เราออกแบบมันด้วย
`#[derive(thiserror::Error)]` ให้ครอบคลุมสถานการณ์ความล้มเหลวที่พบจริงในระบบ REST (`NotFound`, `Validation`,
`ValidationErrors`, `Conflict`, `Internal`) แล้ว implement `IntoResponse` **ครั้งเดียว** ให้ error ทุกชนิดตอบ
JSON shape เดียวกันเสมอ (`{"error": {"code": "...", "message": "..."}}`) พิสูจน์ด้วย `curl` จริงว่าปัญหา
"รูปแบบ error ไม่สอดคล้องกัน" ที่ Part 63-65 ทิ้งไว้ (บาง endpoint ตอบ JSON, บาง endpoint ตอบ plain text) หมด
ไปโดยสิ้นเชิง

จากนั้นเราเขียน handler ที่ chain การทำงานหลายจุดด้วย `?` ตัวเดียว ให้ทั้ง `Option::None`, business rule, และ
`anyhow::Error` จากระบบภายนอกไหลรวมเป็น `AppError` เดียวกันโดยอัตโนมัติผ่าน `#[from]` — ทำ **logging แบบรวม
ศูนย์จุดเดียว** ภายใน `IntoResponse` พร้อม request id ที่ไหลผ่าน tracing span (ไม่ต้องส่งผ่านพารามิเตอร์ของ
`AppError` เองเลย) และพิสูจน์หลักการความปลอดภัยที่สำคัญที่สุดข้อหนึ่ง: **log เก็บรายละเอียดเต็ม, response บอก
แค่ข้อความปลอดภัยทั่วไป** ด้วยการรันจริงทั้งสองฝั่งเทียบกัน — ต่อด้วยการทำ validation แบบ field-level
(`ValidationErrors` + `FieldError`) ที่รายงานทุกจุดผิดในครั้งเดียว, แปลง extractor rejection ของ Axum เอง
(`JsonRejection`/`PathRejection`) ให้กลายเป็น JSON shape เดียวกับ error อื่นด้วย `#[derive(FromRequest)]`/
`#[derive(FromRequestParts)]`, ใช้ `#[axum::debug_handler]` ให้ compiler ช่วย debug handler ที่ผิดลำดับ
extractor, ทำ fallback ที่ตอบสอดคล้องกับระบบ, และพิสูจน์ด้วยการรันจริงว่า panic ใน handler **ไม่ทำให้ทั้ง
เซิร์ฟเวอร์ล้ม** แต่ทำให้ connection ถูกตัดแบบไม่มี response เลยถ้าไม่มี `tower_http::catch_panic::
CatchPanicLayer` ครอบไว้ — ปิดท้ายด้วย capstone ที่รวมทุกอย่างเข้ากับระบบห้องสมุดจาก Part 63-65 พร้อมทดสอบ
end-to-end ครบทุกเส้นทางความล้มเหลว (`404`/`400`/`409`/`500`/panic/fallback) ในระบบเดียว

ด้วยบทนี้ ชุด Axum ทั้งห้าบท (Part 62-66) ได้ครอบคลุมทุกส่วนที่จำเป็นสำหรับสร้าง REST API ระดับ production
ด้วย Rust: พื้นฐาน routing/handlers, state management/extractors, middleware, และตอนนี้คือ error handling
ที่สอดคล้องกันทั้งระบบ — Part ถัดไปจะแนะนำ **Actix-web** ซึ่งเป็น web framework อีกตัวที่ได้รับความนิยมสูงใน
ระบบนิเวศ Rust ให้เห็นมุมมองที่ต่างจาก Axum และเข้าใจว่าหลักการ error handling/middleware/extractor ที่เรียน
มาตลอดชุดนี้เป็น**แนวคิดสากลของ web framework ใน Rust** ไม่ได้ผูกติดกับ Axum ตัวเดียว แม้ syntax และรายละเอียด
การ implement จะต่างกันไปตามแต่ละ framework

---

**Part ก่อนหน้า:** [Axum: Middleware (tower, tower-http)](part-065-axum-middleware.md) | **Part ถัดไป:** [แนะนำ Actix-web Framework](part-067-actix-web-intro.md)
