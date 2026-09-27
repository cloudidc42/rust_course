# Part 95: Testing Web Applications แบบครบวงจร (unit/integration/e2e)

> โมดูล: Full-Stack และ WebAssembly | ระดับ: สูง | เวลาโดยประมาณ: 300 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบาย testing pyramid ของเว็บแอป Rust แบบ full-stack ได้อย่างเป็นรูปธรรม — รู้ว่าชั้น unit (Part 32), integration
  (Part 33), และ end-to-end (เนื้อหาใหม่ของบทนี้) แต่ละชั้นควรมีจำนวนเท่าไร ทดสอบอะไร และให้ความมั่นใจ (confidence)
  แบบไหนที่อีกสองชั้นให้ไม่ได้
- เขียน unit test ทดสอบ Axum handler โดยตรงผ่าน `tower::ServiceExt::oneshot` โดยไม่ต้อง bind port จริงเลย ทั้ง
  happy path และ error path ที่ผูกกับ `AppError` (ต่อยอด Part 66) พร้อมสับเปลี่ยน dependency (เช่น database
  repository) ด้วยเทคนิค trait-based mocking จาก Part 32
- เขียน integration test ที่ยิง request ผ่าน handler จริงไปจนถึง SQLx query จริงบน PostgreSQL จริง ด้วยแพทเทิร์น
  `#[sqlx::test]` (ต่อยอด Part 70/71) ที่สร้างฐานข้อมูลใหม่ให้แต่ละเทสอัตโนมัติ แยก isolate กันสมบูรณ์
- ทดสอบโค้ด Rust ที่รันอยู่ใน WASM ด้วย `wasm-pack test` ทั้งในบริบท Node.js (`--node`) และเบราว์เซอร์จริงแบบ
  headless (`--headless --chrome`) — และรู้ความแตกต่างเชิงกลไกที่แท้จริงระหว่างสองโหมดนี้ (ไม่ใช่แค่ auto-detect)
- ประเมินได้อย่างตรงไปตรงมาว่าการทดสอบ component (render แล้วตรวจ DOM) ของ Leptos/Yew/Dioxus แต่ละตัวอยู่ตัว
  (mature) แค่ไหนในเวอร์ชันปัจจุบัน และเลือกวิธีที่เหมาะกับสถานการณ์ได้ (render-to-string เทียบกับ mount ลง DOM จริง)
- เขียนและรัน end-to-end test ด้วย Playwright ที่เปิดเบราว์เซอร์จริง ขับผ่าน backend server จริงที่คุยกับ
  PostgreSQL จริง และ frontend WASM bundle จริงที่ build ออกมาจริง ตรวจสอบ user flow เต็มรูปแบบ (ดูรายการ →
  คลิกเข้าไปดูรายละเอียด) ด้วย assertion บนเนื้อหาที่ render จริงบนหน้าจอ
- วินิจฉัยและแก้ไข flaky e2e test ได้ถูกจุด — แยกแยะระหว่าง "fixed sleep สั้นเกินไป" (แก้ที่โค้ดเทส) กับ
  "infrastructure ไม่พร้อม" (แก้ที่ environment) และรู้ว่าทำไม retry เฉย ๆ ไม่ใช่คำตอบที่ถูกต้องเสมอไป

## ความรู้ที่ต้องมีมาก่อน

- **Part 32 (Testing: Unit Tests)**: `#[test]`, `assert_eq!`, และเทคนิค **trait-based mocking** ("นิยาม trait
  แทน dependency แล้ว implement สองแบบ — ของจริง/ของปลอม") คือหัวใจของหัวข้อ 95.3 ที่ใช้เทคนิคเดียวกันนี้เป๊ะ
  เพื่อทดสอบ Axum handler โดยไม่ต้องมี PostgreSQL จริงอยู่ข้างหลัง
- **Part 33 (Testing: Integration Tests)**: โครงสร้าง `tests/` ที่แต่ละไฟล์คือ crate อิสระ, ธรรมเนียม
  `tests/common/mod.rs`, และภาพ testing pyramid เวอร์ชันแรกที่ Part 33 วาดไว้ (unit → integration → "e2e (นอก
  ขอบเขตบทนั้น — ดู Part 95)") — บทนี้คือบทที่ Part 33 อ้างถึงไว้ตรง ๆ
- **Part 62-63 (Axum พื้นฐาน)**: การสร้าง `Router`, handler function, extractor พื้นฐาน — จำเป็นสำหรับอ่านโค้ด
  ตัวอย่างในหัวข้อ 95.3-95.4 ทั้งหมด
- **Part 66 (Axum Error Handling)**: `AppError` enum เดียวที่เป็นจุดศูนย์กลางของ error ทั้งแอป พร้อม
  `impl IntoResponse for AppError` — บทนี้ทดสอบ error path ที่ผูกกับ `AppError` variant จริงโดยตรง ไม่ใช่แค่
  happy path
- **Part 70-71 (SQLx: PostgreSQL, Queries และ Migrations)**: `PgPoolOptions`, `sqlx::query_as!`, transaction
  ผ่าน `pool.begin()`/`commit()`, และโดยเฉพาะ `#[sqlx::test]` ที่ Part 70 หัวข้อ 70.13 สอนไว้แล้ว — บทนี้นำ
  `#[sqlx::test]` มาใช้ทดสอบ handler แบบครบวงจรจริง ไม่ใช่แค่ทดสอบ query ฟังก์ชันเดี่ยว ๆ
- **Part 86-87 (WASM พื้นฐาน และ wasm-bindgen)**: Part 86 แนะนำ `wasm-pack test --node` ไว้แบบสั้น ๆ แล้ว
  (หัวข้อที่ชื่อ "อีกวิธีในการทดสอบ") — บทนี้ขยายเรื่องนี้เต็มรูปแบบ รวมถึงโหมด `--headless --chrome` ที่ Part 86
  พูดถึงไว้แค่ชื่อ
- **Part 88-90 (Yew, Leptos, Dioxus)**: Part 88 หัวข้อ 88.11 บอกไว้ตรง ๆ ว่า "เนื้อหาการเขียน test เต็มรูปแบบ
  สำหรับเว็บแอปจะถูกรวบรวมให้ครบใน Part 95" — นี่คือบทนั้น เราจะกลับไปทดสอบ component ของทั้งสามเฟรมเวิร์กอีกครั้ง
  พร้อมเทคนิคเต็มชุด
- **Part 91-94 (SSR และ Full-Stack Project)**: ในทางทฤษฎีบทนี้ควรทดสอบแอป capstone ที่ Part 92-94 สร้างไว้ตรง ๆ
  — หัวข้อ 95.2 จะอธิบายสถานะจริงและทางเลือกที่บทนี้ใช้

## เนื้อหา

### 95.1 Testing Pyramid สำหรับ Full-Stack Rust: ทวนและขยายไปถึง E2E

Part 33 หัวข้อ 33.8 วาดภาพ testing pyramid ไว้แบบสามชั้นแล้ว โดยจงใจเว้นชั้นบนสุดไว้ว่า "นอกขอบเขตบทนั้น — ดู
Part 95" มาถึงตอนนี้เราเรียนมาครบทั้งสามชั้นของโค้ด backend (unit/integration จาก Part 32-33), ทั้ง WASM/frontend
(Part 86-90), และ full-stack integration (Part 91-94) แล้ว — ถึงเวลาต่อยอดภาพสามเหลี่ยมให้ครบทั้งสี่ชั้นจริง ๆ
สำหรับแอป full-stack Rust โดยเฉพาะ:

```
          /\
         /  \        ยอด:    End-to-End Tests (บทนี้ หัวข้อ 95.8-95.9)
        /----\                จำนวนน้อยที่สุด (สิบ-ยี่สิบเคสต่อแอป), ช้าที่สุด (วินาที-นาทีต่อเทส),
       /      \               ครอบคลุมระบบทั้งหมดจริง ๆ ผ่านเบราว์เซอร์จริง — ให้ความมั่นใจสูงสุดว่า
      /--------\              "ผู้ใช้จริงจะใช้งานได้" แต่แพงที่สุดและ flaky ง่ายที่สุด
     /          \
    /   WASM/    \  ชั้น 3:  WASM/Component Tests (บทนี้ หัวข้อ 95.6-95.7)
   /  Component  \           จำนวนปานกลาง, ต้อง compile เป็น wasm32 ก่อนรันทุกครั้ง (ช้ากว่า unit
  /----------------\          test ปกติที่ compile เป็น native code) ทดสอบ logic ที่รันในเบราว์เซอร์
 /                  \         และการ render ของ component โดยไม่ต้องเปิดระบบทั้งชุด
/--------------------\
/      Integration    \  ชั้น 2: Integration Tests (Part 33 + บทนี้ หัวข้อ 95.4)
/------------------------\      ทดสอบ handler->query->response ครบวงจรกับ PostgreSQL จริง — ยืนยัน
/                          \     "สัญญา" ของ public API ทั้งหมด รวมทั้งจุดต่อกับฐานข้อมูลที่ unit test
/                            \    มองไม่เห็น
/------------------------------\
/           Unit Tests            \  ฐาน:    Unit Tests (Part 32 + บทนี้ หัวข้อ 95.3)
/----------------------------------\          จำนวนมากที่สุด, เร็วที่สุด (มิลลิวินาที), ทดสอบ handler/
                                               ฟังก์ชันเดี่ยว ๆ แยกส่วนด้วย mock/fake แทน dependency จริง
```

สังเกตว่าตอนนี้เรามี **สี่ชั้น ไม่ใช่สาม** — เหตุผลที่ต้องแยกชั้น "WASM/Component Tests" ออกมาจาก unit test ปกติ
คือ **ต้นทุนการรันต่างกันจริงในทางเทคนิค**: unit test ของ Part 32 compile เป็น native code แล้วรันตรงบน CPU ของ
เครื่อง (เร็วมาก, มิลลิวินาที) แต่ WASM test ต้อง compile เป็น `wasm32-unknown-unknown` ก่อน แล้วยังต้องมี
runtime มารันไบต์โค้ด WASM นั้นอีกชั้น (Node.js VM หรือเอนจิน JavaScript ของเบราว์เซอร์) — ขั้นตอนพิเศษนี้ทำให้
WASM test ช้ากว่า native unit test เสมอ แม้จะทดสอบ logic ที่เรียบง่ายพอ ๆ กัน (จะเห็นตัวเลขจริงในหัวข้อ 95.6)

**ตารางสรุป trade-off ทั้งสี่ชั้น** (ตัวเลขเป็นลำดับขนาด ไม่ใช่ค่าตายตัว — ขึ้นกับแอปจริง):

| ชั้น | จำนวนเทสทั่วไปต่อแอป | เวลารันต่อเทส | ทดสอบผ่าน dependency จริงไหม | จับบั๊กประเภทไหนได้ดีที่สุด |
|---|---|---|---|---|
| Unit (95.3) | หลักร้อย-พัน | < 10ms | ไม่ (mock/fake ทั้งหมด) | logic ผิดในฟังก์ชัน/handler เดี่ยว ๆ |
| Integration (95.4) | หลักสิบ-ร้อย | 10-500ms | ผ่าน DB จริง, ไม่ผ่าน network/browser จริง | จุดต่อระหว่าง handler กับ query, schema ผิด, transaction ผิด |
| WASM/Component (95.6-95.7) | หลักสิบ | 100ms-2s | ไม่ (ทดสอบ logic/render แยกจากระบบจริง) | บั๊กที่เกิดเฉพาะบน WASM target, การ render component ผิด |
| E2E (95.8-95.9) | หลักหน่วย-สิบ | 1-10s+ | ผ่านทุกอย่างจริง (server+DB+browser) | จุดต่อข้าม layer ทั้งระบบ, CORS ผิด, build pipeline ผิด, regression ที่ layer อื่นมองไม่เห็น |

หลักการที่สำคัญที่สุดของทั้งตารางคือ **"ยิ่งขึ้นไปสูง ยิ่งได้ความมั่นใจกว้างขึ้นแต่แพงขึ้นและเปราะบางขึ้น"** — e2e
test หนึ่งตัวที่ผ่านบอกได้ว่า "user flow นี้ใช้งานได้จริงทั้งระบบ" ซึ่งเป็นสิ่งที่ unit test 1,000 ตัวรวมกันบอกไม่ได้
เลย (เพราะ unit test ไม่เคยพิสูจน์ว่า frontend เรียก API endpoint ถูก path จริง หรือ CORS header ถูกตั้งค่าไว้จริง)
แต่ในทางกลับกัน e2e test หนึ่งตัวที่ล้มเหลวบอกได้แค่ "มีบางอย่างผิดที่ไหนสักแห่งในระบบทั้งชุด" ซึ่งต้องไล่ debug
ต่อเองว่าอยู่ตรงไหน (ต่างจาก unit test ที่ fail แล้วชี้ตรงไปที่ฟังก์ชันเดียวทันที) — นี่คือเหตุผลที่รูปสามเหลี่ยม
ต้อง "ฐานกว้าง ยอดแคบ": ให้ unit test จำนวนมากจับบั๊กเชิงตรรกะให้เร็วและระบุจุดผิดให้ชัดที่สุดก่อน แล้วให้ e2e test
จำนวนน้อยทำหน้าที่เป็น "เครื่องยืนยันครั้งสุดท้าย" ว่าทุกชิ้นที่ประกอบกันทำงานได้จริง ไม่ใช่ให้ e2e test แบกภาระ
ทดสอบทุก edge case ของ business logic (ซึ่งจะทำให้ทั้ง suite รันช้ามากและ flaky มาก โดยไม่ได้ความมั่นใจเพิ่มขึ้น
ตามสัดส่วนเลย)

#### Anti-pattern ที่ต้องรู้จักและหลีกเลี่ยง: "Ice-Cream Cone"

รูปสามเหลี่ยม "ฐานกว้าง ยอดแคบ" ในหัวข้อนี้มีชื่อตรงข้ามที่วงการ testing เรียกว่า **"ice-cream cone"**
(ฐานแคบ ยอดกว้าง) — เกิดขึ้นเมื่อทีมพัฒนาพึ่งพา e2e test เป็นหลักและมี unit test น้อยมาก มักเกิดจากเหตุผลที่ดูสม
เหตุผลในระยะสั้น: "e2e test พิสูจน์ได้มากกว่าต่อเทสหนึ่งตัว เขียนน้อยตัวก็ครอบคลุมกว้าง" แต่ผลลัพธ์ระยะยาวคือ
ตรงข้ามกับที่คาด — suite ที่เป็น ice-cream cone จะ**รันช้ามาก** (ทุกเทสต้องผ่าน browser+server+DB จริงหมด),
**flaky มาก** (ทุกเทสรับความเสี่ยงจาก timing/infrastructure ของทุกชั้นพร้อมกัน), และ**ระบุจุดผิดได้ยาก** (เทส
fail แล้วบอกได้แค่ "อะไรบางอย่างพัง" ไม่บอกว่าพังที่ฟังก์ชันไหน) — สามปัญหานี้ทวีคูณกันเมื่อ suite โตขึ้น จนถึง
จุดที่ทีมไม่กล้ารัน suite เต็มบ่อย ๆ เพราะใช้เวลานานและผลลัพธ์ไม่น่าเชื่อถือ ซึ่งกลับทำให้ความมั่นใจในการ deploy
ลดลง ทั้งที่จุดเริ่มต้นคือความตั้งใจจะ "มั่นใจมากขึ้น" ตารางในหัวข้อ 95.11 ที่จะเห็นท้ายบทคือหลักฐานตัวเลขจริงว่า
ทำไมรูปสามเหลี่ยม (ไม่ใช่ ice-cream cone) จึงเป็นทางเลือกที่ยั่งยืนกว่าในระยะยาว

### 95.2 โปรเจกต์ตัวอย่างของบทนี้: สแตนอินสำหรับ Capstone Part 92-94

ก่อนลงรายละเอียดเทคนิค ต้องพูดตรง ๆ ก่อนว่าบทนี้ทดสอบอะไรอยู่ ณ ขณะที่เขียนบทนี้ **Part 91 (Server-Side
Rendering), Part 92-94 (Full-Stack Project สามภาค) ยังไม่มีอยู่ในระบบไฟล์ของหลักสูตร** (สถานะ ⏳ ใน
`docs/curriculum.md`) — เนื่องจากหลักสูตรนี้เขียนโดย agent หลายตัวพร้อมกันในหลายบทพร้อม ๆ กัน บทนี้จึงไม่มี
capstone project จริงให้อ้างอิงตรง ๆ

ทางแก้ที่บทนี้เลือกคือ **สร้างแอป full-stack ตัวแทน (representative app)** ที่มีสถาปัตยกรรมแบบเดียวกับที่ Part
92-94 น่าจะใช้จริง โดยยึดหลักฐานสองอย่างที่มีอยู่แล้วในหลักสูตร:

1. **Part 90 หัวข้อสรุป** ระบุไว้ตรง ๆ ว่า "หลักสูตรนี้เลือก **Leptos** สำหรับ capstone project ใน Part 92-94
   เพราะโจทย์ของ capstone คือ full-stack web application ที่ตรงกับจุดแข็งของ Leptos ที่สุด" — บทนี้จึงใช้ Leptos
   เป็น frontend framework หลักของแอปตัวแทน (พร้อมทดสอบ Yew/Dioxus แยกในหัวข้อ 95.7 ด้วยเพื่อความครบถ้วน)
2. **โดเมนของแอป** ใช้ระบบห้องสมุด (หนังสือ/การยืม) ต่อเนื่องจากที่ Part 33, 66, 70, 71 ใช้มาตลอด (`Book`,
   `borrow_book`, `AppError::Conflict` เมื่อยืมซ้ำ) — เพื่อให้ผู้อ่านที่ตามมาถึงบทนี้เห็นภาพต่อเนื่อง ไม่ต้องเรียนรู้
   โดเมนใหม่ทั้งหมด (Part 89 ใช้โดเมนจองตั๋วงานสัมมนาสำหรับสาธิต server function โดยเฉพาะ แต่บทนี้กลับไปใช้โดเมน
   ห้องสมุดเพื่อให้ผูกกับ `AppError`/SQLx pattern ที่ Part 66/70/71 วางไว้ได้ตรงที่สุด)

สถาปัตยกรรมของแอปตัวแทน:

```
library_capstone/
├── backend/                  # Axum + SQLx + PostgreSQL
│   ├── src/
│   │   ├── lib.rs            # AppState, api_router(), full_router()
│   │   ├── error.rs          # AppError (แบบเดียวกับ Part 66)
│   │   ├── models.rs         # struct Book
│   │   ├── repository.rs     # trait BookRepository + PgBookRepository + InMemoryBookRepository
│   │   ├── handlers.rs       # list_books, get_book, borrow_book + unit test มือ oneshot
│   │   └── main.rs           # thin main ที่เรียก lib (ตาม pattern Part 17/33)
│   ├── migrations/0001_init.sql
│   └── tests/integration_test.rs   # #[sqlx::test] เต็มวงจร
└── frontend/                 # Leptos CSR (ตาม "แนวทางที่ 1" ของ Part 89)
    ├── src/lib.rs            # BookList, BookDetail, validate_borrow_days, availability_label
    ├── src/main.rs           # mount()
    ├── tests/web.rs          # wasm-pack test --node (pure logic)
    ├── tests/web_browser.rs  # wasm-pack test --headless --chrome (pure logic เดียวกัน)
    ├── tests/dom.rs          # wasm-pack test --headless --chrome (mount ลง DOM จริง)
    └── Trunk.toml, index.html
```

Backend serve ทั้ง API (ที่ `/api/*`) และไฟล์ static ของ frontend ที่ build แล้ว (ที่ `/`) จาก process เดียวกัน
ผ่าน `tower_http::services::ServeDir` — เพื่อให้ e2e test คุยกับ origin เดียวโดยไม่ต้องยุ่งกับ CORS เลย (นี่คือ
รูปแบบ deployment ที่ตรงกับที่ Part 94 ("Integration และ Deployment") ตั้งใจจะสอนโดยตรงตามชื่อหัวข้อของมัน):

```rust
// backend/src/lib.rs
use axum::{routing::{get, post}, Router};
use repository::BookRepository;
use std::sync::Arc;
use tower_http::{cors::CorsLayer, services::ServeDir};

#[derive(Clone)]
pub struct AppState {
    pub repo: Arc<dyn BookRepository>,
}

/// สร้าง Router ของ API เพียวๆ (ไม่ผูก static file) — ใช้ตรงนี้ทั้งใน main.rs และใน test
/// (ตาม pattern oneshot ที่ทดสอบ handler โดยไม่ต้อง bind port จริง)
pub fn api_router(state: AppState) -> Router {
    Router::new()
        .route("/healthz", get(handlers::healthz))
        .route("/books", get(handlers::list_books))
        .route("/books/{id}", get(handlers::get_book))
        .route("/books/{id}/borrow", post(handlers::borrow_book))
        .with_state(state)
}

/// Router เต็มรูปแบบสำหรับรันจริง: เสิร์ฟไฟล์ static ของ frontend ที่ "/" และ API ที่ "/api"
/// (ใช้ตอน e2e test ที่ต้องมีทั้งหน้าเว็บและ API อยู่ origin เดียวกัน ไม่ต้องพึ่ง CORS)
pub fn full_router(state: AppState, static_dir: &str) -> Router {
    Router::new()
        .nest("/api", api_router(state))
        .fallback_service(ServeDir::new(static_dir))
        .layer(CorsLayer::permissive())
}
```

สังเกตการออกแบบสองฟังก์ชันแยกกัน — `api_router()` คืน `Router` ที่ผูก state ไว้เรียบร้อยแล้ว (ใช้ได้ตรงกับ
`oneshot` ทันทีไม่ต้องแก้อะไรในหัวข้อ 95.3-95.4) ส่วน `full_router()` เพิ่มชั้น `.fallback_service(ServeDir::new(...))`
ครอบไว้อีกที **เฉพาะตอนรันจริง** — เหตุผลที่ไม่รวมสองอย่างนี้เป็นฟังก์ชันเดียวคือ unit/integration test (95.3-95.4)
ไม่จำเป็นต้องรู้จักไฟล์ static ของ frontend เลย การแยกให้ชัดทำให้เทสเหล่านั้น**ไม่มีทางพังเพราะโฟลเดอร์ static
ไม่มีอยู่จริง** (ปัญหาที่จะเกิดถ้ารวมทุกอย่างเป็นฟังก์ชันเดียวแล้วเทสถูกรันจาก working directory ที่ไม่มี `dist/`)
— นี่คือตัวอย่างของการออกแบบ API ให้ testability สูงตั้งแต่ต้น ตาม spirit เดียวกับ pattern "lib บาง ๆ + main
บาง ๆ" ที่ Part 17/33 สอนไว้

`CorsLayer::permissive()` ในตัวอย่างนี้ **ไม่ได้จำเป็นจริง ๆ** เพราะ frontend กับ backend อยู่ origin เดียวกัน
เสมอ (ถูก serve จาก process เดียวกัน) — ที่ใส่ไว้เป็นการป้องกันล่วงหน้าสำหรับตอนพัฒนา (เช่นรัน `trunk serve`
แบบ dev server คนละพอร์ตจาก backend ระหว่างเขียนโค้ด ก่อนจะ build ไปรวมกันตอน deploy) หากไม่ตั้งใจจะรันแบบ
คนละ origin เลย ตัด `CorsLayer` ออกได้โดยไม่กระทบอะไร — ประเด็นนี้สำคัญพอที่ Part 94 ("Integration และ
Deployment") ควรพูดถึงโดยตรงว่าการรวม origin แบบนี้ตัดปัญหา CORS ทั้งหมดที่ผู้เริ่มต้นทำ full-stack app มักเจอ
ตอน frontend/backend รันคนละพอร์ตกันตลอดไป

Migration ของฐานข้อมูล (ใช้กับทั้ง `main.rs` ตอนรันจริง และ `#[sqlx::test]` ในหัวข้อ 95.4):

```sql
-- backend/migrations/0001_init.sql
CREATE TABLE IF NOT EXISTS books (
    id BIGSERIAL PRIMARY KEY,
    title TEXT NOT NULL,
    author TEXT NOT NULL,
    isbn TEXT NOT NULL UNIQUE,
    available BOOLEAN NOT NULL DEFAULT true
);
```

`Cargo.toml` ของ backend (สังเกต feature `migrate` ของ `sqlx` ที่จำเป็นสำหรับ `#[sqlx::test(migrations = ...)]`
ในหัวข้อ 95.4 — ถ้าลืม feature นี้ compiler จะปฏิเสธ attribute `#[sqlx::test]` ทันที):

```toml
[package]
name = "library_api"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = { version = "0.8", features = ["macros"] }
tokio = { version = "1", features = ["full"] }
tower = { version = "0.5", features = ["util"] }
tower-http = { version = "0.6", features = ["fs", "cors", "trace"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
sqlx = { version = "0.8", features = ["runtime-tokio", "postgres", "chrono", "macros", "migrate"] }
thiserror = "2"
anyhow = "1"
async-trait = "0.1"
tracing = "0.1"
tracing-subscriber = "0.3"

[dev-dependencies]
http-body-util = "0.1"
```

`AppError` ผูกกับ Part 66 ตรง ๆ — enum เดียวที่เป็นจุดศูนย์กลางของ error ทั้งแอป พร้อม `impl IntoResponse` ที่
กำหนด JSON shape เดียวกันทั้งระบบ และ `impl From<sqlx::Error>` ที่แปลง `sqlx::Error::RowNotFound` เป็น
`AppError::NotFound` โดยอัตโนมัติ (ลด boilerplate ที่ต้องเขียน `.ok_or(AppError::NotFound)?` ซ้ำทุกจุดที่ query
เดี่ยวอาจไม่เจอแถว):

```rust
// backend/src/error.rs
use axum::{http::StatusCode, response::{IntoResponse, Response}, Json};
use serde_json::json;

#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error("ไม่พบหนังสือที่ต้องการ")]
    NotFound,
    #[error("ข้อมูลไม่ถูกต้อง: {0}")]
    Validation(String),
    #[error("ขัดแย้งกับสถานะปัจจุบัน: {0}")]
    Conflict(String),
    #[error("เกิดข้อผิดพลาดภายในระบบ")]
    Internal(#[from] anyhow::Error),
}

impl From<sqlx::Error> for AppError {
    fn from(err: sqlx::Error) -> Self {
        match err {
            sqlx::Error::RowNotFound => AppError::NotFound,
            other => AppError::Internal(other.into()),
        }
    }
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, code) = match &self {
            AppError::NotFound => (StatusCode::NOT_FOUND, "NOT_FOUND"),
            AppError::Validation(_) => (StatusCode::BAD_REQUEST, "VALIDATION_ERROR"),
            AppError::Conflict(_) => (StatusCode::CONFLICT, "CONFLICT"),
            AppError::Internal(_) => (StatusCode::INTERNAL_SERVER_ERROR, "INTERNAL_ERROR"),
        };
        if let AppError::Internal(source) = &self {
            tracing::error!(error = %source, "internal error");
        }
        let body = Json(json!({ "error": code, "message": self.to_string() }));
        (status, body).into_response()
    }
}
```

และ `Book` model ที่ derive ทั้ง `Serialize`/`Deserialize` (ส่งผ่าน HTTP จริงตาม Part 57) และ `sqlx::FromRow`
(map แถวจาก PostgreSQL ตาม Part 70) พร้อมกัน — struct เดียวใช้ได้ทั้งสองทาง เพราะ field ชื่อตรงกับ column ทุกตัว:

```rust
// backend/src/models.rs
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, sqlx::FromRow)]
pub struct Book {
    pub id: i64,
    pub title: String,
    pub author: String,
    pub isbn: String,
    pub available: bool,
}
```

**ทุกตัวอย่างโค้ดและผลลัพธ์ในบทนี้มาจากการ compile และรันจริง** ในสภาพแวดล้อมที่เขียนบทนี้ — มี PostgreSQL 16.13
รันอยู่จริง (cluster เดียวกับที่ Part 70-71/89 ใช้), มี `wasm-pack` 0.15.0, `trunk` 0.21.14, Node.js v22.22.2, และ
Playwright 1.56.1 พร้อม headless Chromium ติดตั้งไว้แล้ว (เวอร์ชันเดียวกับที่ Part 88/90 ใช้ยืนยันผลลัพธ์ของตัวเอง)
โค้ดทั้งหมดถูกเขียนไว้ใน scratch directory แยกจากตัวหลักสูตร แล้วลบทิ้งหลังตรวจสอบเสร็จตามระเบียบของหลักสูตร

### 95.3 Backend Unit Test เจาะลึก: ทดสอบ Handler ผ่าน `oneshot` โดยไม่ Bind Port จริง

Part 33 หัวข้อ 33.1 บอกไว้ว่า unit test "ไม่ได้พิสูจน์ว่าคนนอกที่ import crate ของคุณไปใช้จริงจะเรียกใช้งานมันได้
ถูกต้องหรือไม่" — แต่สำหรับ **handler ของ web framework** มีปัญหาเฉพาะทางเพิ่มมาอีกชั้น: handler ถูกออกแบบมาให้
ทำงานผ่าน HTTP request/response ผ่าน `Router` ที่ bind port จริง การจะเขียน unit test ที่ "แค่เรียกฟังก์ชัน
ตรง ๆ" แบบ Part 32 ทำไม่ได้ตรง ๆ เพราะ handler รับ extractor (เช่น `State<T>`, `Path<T>`, `Json<T>`) ที่ Axum
เป็นคนสร้างให้จาก `Request` จริง ไม่ใช่ argument ที่เรียกตรงได้ง่าย ๆ

ทางแก้คือ `tower::ServiceExt::oneshot` — trait method ที่ยืม concept ของ `oneshot` channel (Part 48/49) มาใช้กับ
`Service`: **ส่ง `Request` เข้าไปหนึ่งตัว รอ `Response` กลับมาหนึ่งตัว จบ** โดยไม่ต้อง `TcpListener::bind` เลย
เพราะ `Router` ของ Axum implement trait `tower::Service<Request>` อยู่แล้ว (Part 65 พูดถึง `Service`/`Layer`
ไปแล้วตอนสอน middleware) — `oneshot` แค่เรียก `Service::call` ครั้งเดียวแล้ว await ผลลัพธ์ ทำให้เราทดสอบทั้ง
"เส้นทาง routing + extractor + handler + IntoResponse" ได้ครบในหน่วยความจำเดียวกัน เร็วเท่า unit test ปกติ

#### แยก dependency ด้วย trait ก่อน (ตาม Part 32 หัวข้อ mocking)

ปัญหาคือ handler จริงของแอป (`get_book`, `borrow_book`) ต้องคุยกับ PostgreSQL — ถ้าทดสอบผ่าน pool จริงทุกครั้ง
จะกลายเป็น integration test (หัวข้อ 95.4) ไม่ใช่ unit test ที่ต้อง "isolated" ตามนิยามของ Part 32 คำตอบคือนิยาม
trait กลางที่ handler คุยด้วย แล้ว implement สองแบบ — เทคนิคเดียวกับที่ Part 32 สอนเรื่อง mocking ผ่าน trait:

```rust
use crate::{error::AppError, models::Book};
use async_trait::async_trait;
use std::{collections::HashMap, sync::Mutex};

/// Trait กลางที่ handler คุยด้วย — สับเปลี่ยนระหว่าง "ของจริง" (Postgres) กับ "ของปลอม"
/// (in-memory) ได้ตาม pattern ที่ Part 32 สอนเรื่อง mocking ผ่าน trait
#[async_trait]
pub trait BookRepository: Send + Sync {
    async fn list(&self) -> Result<Vec<Book>, AppError>;
    async fn find(&self, id: i64) -> Result<Option<Book>, AppError>;
    /// ยืมหนังสือ: คืน error ถ้าไม่พบ หรือถ้าถูกยืมไปแล้ว (ไม่ available)
    async fn borrow(&self, id: i64) -> Result<Book, AppError>;
}

// ==================== In-memory: ของปลอม สำหรับ unit test handler (ไม่ต้องมี DB) ====================

pub struct InMemoryBookRepository {
    books: Mutex<HashMap<i64, Book>>,
}

impl InMemoryBookRepository {
    pub fn new(seed: Vec<Book>) -> Self {
        let books = seed.into_iter().map(|b| (b.id, b)).collect();
        Self { books: Mutex::new(books) }
    }
}

#[async_trait]
impl BookRepository for InMemoryBookRepository {
    async fn list(&self) -> Result<Vec<Book>, AppError> {
        let mut books: Vec<Book> = self.books.lock().unwrap().values().cloned().collect();
        books.sort_by_key(|b| b.id);
        Ok(books)
    }

    async fn find(&self, id: i64) -> Result<Option<Book>, AppError> {
        Ok(self.books.lock().unwrap().get(&id).cloned())
    }

    async fn borrow(&self, id: i64) -> Result<Book, AppError> {
        let mut guard = self.books.lock().unwrap();
        let book = guard.get_mut(&id).ok_or(AppError::NotFound)?;
        if !book.available {
            return Err(AppError::Conflict(format!("หนังสือ '{}' ถูกยืมไปแล้ว", book.title)));
        }
        book.available = false;
        Ok(book.clone())
    }
}
```

`PgBookRepository` (ของจริง ใช้ `sqlx::PgPool`) จะเห็นเต็มรูปแบบในหัวข้อ 95.4 — ตอนนี้สังเกตว่า `AppState`
เก็บ `repo` เป็น `Arc<dyn BookRepository>` (trait object ตาม Part 21) ไม่ใช่ type คอนกรีตตรง ๆ ทำให้ handler
**ไม่รู้เลยว่าอยู่หลัง repo ตัวไหน** — นี่คือกลไกที่ทำให้ unit test สับเปลี่ยนของปลอมเข้าไปได้โดยไม่ต้องแก้โค้ด
handler แม้แต่บรรทัดเดียว:

```rust
#[derive(Clone)]
pub struct AppState {
    pub repo: Arc<dyn BookRepository>,
}

pub fn api_router(state: AppState) -> Router {
    Router::new()
        .route("/healthz", get(handlers::healthz))
        .route("/books", get(handlers::list_books))
        .route("/books/{id}", get(handlers::get_book))
        .route("/books/{id}/borrow", post(handlers::borrow_book))
        .with_state(state)
}
```

#### Handler จริงที่จะทดสอบ

```rust
use crate::{error::AppError, models::Book, AppState};
use axum::{extract::{Path, State}, Json};

pub async fn list_books(State(state): State<AppState>) -> Result<Json<Vec<Book>>, AppError> {
    let books = state.repo.list().await?;
    Ok(Json(books))
}

pub async fn get_book(
    State(state): State<AppState>,
    Path(id): Path<i64>,
) -> Result<Json<Book>, AppError> {
    let book = state.repo.find(id).await?.ok_or(AppError::NotFound)?;
    Ok(Json(book))
}

pub async fn borrow_book(
    State(state): State<AppState>,
    Path(id): Path<i64>,
) -> Result<Json<Book>, AppError> {
    let book = state.repo.borrow(id).await?;
    Ok(Json(book))
}
```

#### เทส: happy path, error path (`AppError::NotFound`), และ error path (`AppError::Conflict`)

```rust
#[cfg(test)]
mod tests {
    //! Unit test ของ handler โดยตรง ผ่าน tower::ServiceExt::oneshot — ไม่มีการ bind port จริง
    //! เลยแม้แต่นิดเดียว, ไม่แตะ Postgres เลย (ใช้ InMemoryBookRepository เป็นของปลอมแทน)
    use super::*;
    use crate::{api_router, repository::InMemoryBookRepository, AppState};
    use axum::body::Body;
    use axum::http::{Request, StatusCode};
    use http_body_util::BodyExt;
    use std::sync::Arc;

    fn seed_state() -> AppState {
        let repo = InMemoryBookRepository::new(vec![
            Book { id: 1, title: "The Rust Programming Language".into(), author: "Klabnik & Nichols".into(), isbn: "978-1".into(), available: true },
            Book { id: 2, title: "Programming Rust".into(), author: "Blandy & Orendorff".into(), isbn: "978-2".into(), available: false },
        ]);
        AppState { repo: Arc::new(repo) }
    }

    async fn body_json(response: axum::response::Response) -> serde_json::Value {
        let bytes = response.into_body().collect().await.unwrap().to_bytes();
        serde_json::from_slice(&bytes).unwrap()
    }

    #[tokio::test]
    async fn list_books_returns_all_seeded_books() {
        let app = api_router(seed_state());
        let request = Request::builder().uri("/books").body(Body::empty()).unwrap();

        let response = tower::ServiceExt::oneshot(app, request).await.unwrap();
        assert_eq!(response.status(), StatusCode::OK);
        let json = body_json(response).await;
        assert_eq!(json.as_array().unwrap().len(), 2);
        assert_eq!(json[0]["title"], "The Rust Programming Language");
    }

    #[tokio::test]
    async fn get_book_missing_id_returns_app_error_not_found() {
        // ทดสอบ error path ที่ผูกกับ AppError::NotFound จริง (Part 66) — ไม่ใช่แค่ทดสอบ happy path
        let app = api_router(seed_state());
        let request = Request::builder().uri("/books/999").body(Body::empty()).unwrap();

        let response = tower::ServiceExt::oneshot(app, request).await.unwrap();
        assert_eq!(response.status(), StatusCode::NOT_FOUND);
        let json = body_json(response).await;
        assert_eq!(json["error"], "NOT_FOUND");
    }

    #[tokio::test]
    async fn borrow_already_borrowed_book_returns_conflict() {
        // book id 2 ถูก seed มาเป็น available: false แล้ว — ต้องได้ AppError::Conflict (409)
        let app = api_router(seed_state());
        let request = Request::builder()
            .uri("/books/2/borrow").method("POST").body(Body::empty()).unwrap();

        let response = tower::ServiceExt::oneshot(app, request).await.unwrap();
        assert_eq!(response.status(), StatusCode::CONFLICT);
        let json = body_json(response).await;
        assert_eq!(json["error"], "CONFLICT");
    }

    #[tokio::test]
    async fn borrow_available_book_marks_it_unavailable() {
        let app = api_router(seed_state());
        let request = Request::builder()
            .uri("/books/1/borrow").method("POST").body(Body::empty()).unwrap();

        let response = tower::ServiceExt::oneshot(app, request).await.unwrap();
        assert_eq!(response.status(), StatusCode::OK);
        let json = body_json(response).await;
        assert_eq!(json["available"], false);
    }
}
```

รันจริงด้วย `cargo test` (ตัดส่วน compile ยาว ๆ ออก เหลือผลลัพธ์เทส):

```
     Running unittests src/lib.rs (target/debug/deps/library_api-f58f30c45223dd35)

running 5 tests
test handlers::tests::borrow_already_borrowed_book_returns_conflict ... ok
test handlers::tests::get_book_success_path_returns_matching_book ... ok
test handlers::tests::get_book_missing_id_returns_app_error_not_found ... ok
test handlers::tests::borrow_available_book_marks_it_unavailable ... ok
test handlers::tests::list_books_returns_all_seeded_books ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**สังเกตเวลาที่รัน**: `finished in 0.00s` — ทั้ง 5 เทสรวมกันเร็วระดับ sub-millisecond แม้จะทดสอบ "ทั้ง routing +
extractor + handler + AppError::into_response()" ครบวงจร เพราะไม่มี I/O จริงเกิดขึ้นเลย (`InMemoryBookRepository`
เป็นแค่ `HashMap` ในหน่วยความจำ) — นี่คือเหตุผลที่ pattern นี้ควรเป็น**ฐาน**ของ testing pyramid สำหรับทุก endpoint
ของแอป: เร็วพอที่จะรันได้หลักร้อย-พันครั้งในไม่กี่วินาที และ error message ชี้ตรงไปที่ endpoint ที่พังทันที
(ต่างจาก e2e test ที่ fail แล้วต้องไล่ดูว่าพังที่ layer ไหน)

### 95.4 Backend Integration Test กับ PostgreSQL จริง: Full Round Trip

หัวข้อ 95.3 พิสูจน์ว่า handler ทำงานถูกกับ **สัญญาที่ `BookRepository` กำหนดไว้** — แต่ยังไม่พิสูจน์เลยว่า
`PgBookRepository` ตัวจริง (ที่คุย SQL กับ PostgreSQL) implement สัญญานั้นถูกต้องจริง ๆ นี่คือช่องว่างที่ต้องปิด
ด้วย integration test ที่ยิงผ่าน handler ไปจนถึง SQL query จริง

Part 71 หัวข้อ 71.10 เสนอสองแนวทางสำหรับเทสที่ต้องพึ่ง PostgreSQL: **transaction ต่อ test ที่ไม่ commit** (เร็ว
กว่า แต่ต้องเขียน isolation เอง) กับ **`#[sqlx::test]`** (สร้างฐานข้อมูลใหม่ทั้งลูกให้แต่ละ test, รัน migration
ให้อัตโนมัติ, ลบทิ้งหลังจบ) — บทนี้เลือก **`#[sqlx::test]`** เพราะทดสอบ "ครบวงจรจริง" ที่รวม migration เข้าไป
ด้วย (ถ้า migration script ผิด `#[sqlx::test]` จะ fail ทันทีตั้งแต่ setup ในขณะที่ transaction-per-test แบบมือ
จะไม่จับปัญหานี้เลยเพราะไม่ได้รัน migration ใหม่)

```rust
//! Integration test: request -> handler -> SQLx query -> response แบบครบวงจร กับ Postgres จริง
//! ใช้ #[sqlx::test] (Part 70 หัวข้อ 70.13 / Part 71 หัวข้อ 71.10) — สร้างฐานข้อมูลใหม่ทั้งลูก
//! รัน migration ให้อัตโนมัติ แล้วลบทิ้งหลัง test จบ ทำให้แต่ละ test isolate จากกันสมบูรณ์

use axum::body::Body;
use axum::http::{Request, StatusCode};
use http_body_util::BodyExt;
use library_api::{api_router, repository::PgBookRepository, AppState};
use std::sync::Arc;

async fn body_json(response: axum::response::Response) -> serde_json::Value {
    let bytes = response.into_body().collect().await.unwrap().to_bytes();
    serde_json::from_slice(&bytes).unwrap()
}

fn app_from_pool(pool: sqlx::PgPool) -> axum::Router {
    let state = AppState { repo: Arc::new(PgBookRepository { pool }) };
    api_router(state)
}

#[sqlx::test(migrations = "./migrations")]
async fn borrow_full_round_trip_updates_row_in_real_postgres(pool: sqlx::PgPool) {
    sqlx::query("INSERT INTO books (title, author, isbn, available) VALUES ($1, $2, $3, true)")
        .bind("Zero To Production In Rust")
        .bind("Luca Palmieri")
        .bind("978-9")
        .execute(&pool)
        .await
        .unwrap();

    let row: (i64,) = sqlx::query_as("SELECT id FROM books WHERE isbn = $1")
        .bind("978-9")
        .fetch_one(&pool)
        .await
        .unwrap();
    let book_id = row.0;

    let app = app_from_pool(pool.clone());
    let request = Request::builder()
        .uri(format!("/books/{book_id}/borrow"))
        .method("POST")
        .body(Body::empty())
        .unwrap();
    let response = tower::ServiceExt::oneshot(app, request).await.unwrap();
    assert_eq!(response.status(), StatusCode::OK);
    let json = body_json(response).await;
    assert_eq!(json["available"], false);

    // ยืนยันผลลัพธ์ตรงกับที่ query ฐานข้อมูลจริงหลัง request จบ (ไม่ได้เชื่อ response JSON ฝ่ายเดียว)
    let (available,): (bool,) = sqlx::query_as("SELECT available FROM books WHERE id = $1")
        .bind(book_id)
        .fetch_one(&pool)
        .await
        .unwrap();
    assert!(!available);
}

#[sqlx::test(migrations = "./migrations")]
async fn borrowing_twice_returns_409_conflict_on_second_attempt(pool: sqlx::PgPool) {
    sqlx::query("INSERT INTO books (title, author, isbn, available) VALUES ($1, $2, $3, true)")
        .bind("Rust for Rustaceans")
        .bind("Jon Gjengset")
        .bind("978-7")
        .execute(&pool)
        .await
        .unwrap();
    let row: (i64,) = sqlx::query_as("SELECT id FROM books WHERE isbn = $1")
        .bind("978-7")
        .fetch_one(&pool)
        .await
        .unwrap();
    let book_id = row.0;

    let first = tower::ServiceExt::oneshot(
        app_from_pool(pool.clone()),
        Request::builder().uri(format!("/books/{book_id}/borrow")).method("POST").body(Body::empty()).unwrap(),
    ).await.unwrap();
    assert_eq!(first.status(), StatusCode::OK);

    // ยิงซ้ำครั้งที่สองด้วย app instance ใหม่ แต่ pool เดิม -- พิสูจน์ว่า state อยู่ใน DB จริง ไม่ใช่ในหน่วยความจำ
    let second = tower::ServiceExt::oneshot(
        app_from_pool(pool.clone()),
        Request::builder().uri(format!("/books/{book_id}/borrow")).method("POST").body(Body::empty()).unwrap(),
    ).await.unwrap();
    assert_eq!(second.status(), StatusCode::CONFLICT);
}
```

`PgBookRepository` คือ implementation ตัวจริงของ trait `BookRepository` ที่หัวข้อ 95.3 นิยามไว้ (คู่กับ
`InMemoryBookRepository` ที่เป็นของปลอมสำหรับ unit test) — `list`/`find` เป็น query ตรงไปตรงมาตาม Part 70:

```rust
// backend/src/repository.rs
pub struct PgBookRepository {
    pub pool: sqlx::PgPool,
}

#[async_trait]
impl BookRepository for PgBookRepository {
    async fn list(&self) -> Result<Vec<Book>, AppError> {
        let books = sqlx::query_as::<_, Book>(
            "SELECT id, title, author, isbn, available FROM books ORDER BY id",
        )
        .fetch_all(&self.pool)
        .await?;
        Ok(books)
    }

    async fn find(&self, id: i64) -> Result<Option<Book>, AppError> {
        let book = sqlx::query_as::<_, Book>(
            "SELECT id, title, author, isbn, available FROM books WHERE id = $1",
        )
        .bind(id)
        .fetch_optional(&self.pool)
        .await?;
        Ok(book)
    }

    // borrow() คือ method ที่สาม (ดูเนื้อความเต็มด้านล่าง — แยกมาอธิบายเพราะเป็นจุดที่น่าสนใจที่สุด
    // ของ repository นี้ เนื่องจากต้องจัดการ race condition จริงเมื่อมีคนสองคนพยายามยืมพร้อมกัน)
}
```

โครงสร้างของ `PgBookRepository::borrow` ที่เทสข้างบนพิสูจน์ (ใช้ `SELECT ... FOR UPDATE` ในทรานแซกชันเดียวกับ
`UPDATE` — ล็อกแถวก่อนเช็คเงื่อนไข ป้องกัน race condition เมื่อมีสอง request ยืมพร้อมกันจริง เชื่อมกับ Part 71
เรื่อง transaction/lock):

```rust
async fn borrow(&self, id: i64) -> Result<Book, AppError> {
    let mut tx = self.pool.begin().await?;
    let book = sqlx::query_as::<_, Book>(
        "SELECT id, title, author, isbn, available FROM books WHERE id = $1 FOR UPDATE",
    )
    .bind(id)
    .fetch_optional(&mut *tx)
    .await?
    .ok_or(AppError::NotFound)?;

    if !book.available {
        return Err(AppError::Conflict(format!("หนังสือ '{}' ถูกยืมไปแล้ว", book.title)));
    }

    let updated = sqlx::query_as::<_, Book>(
        "UPDATE books SET available = false WHERE id = $1 RETURNING id, title, author, isbn, available",
    )
    .bind(id)
    .fetch_one(&mut *tx)
    .await?;

    tx.commit().await?;
    Ok(updated)
}
```

ผลลัพธ์จริงจากการรัน (`DATABASE_URL` ชี้ไปยัง PostgreSQL 16.13 ที่รันอยู่ในเครื่องจริง):

```
     Running tests/integration_test.rs (target/debug/deps/integration_test-3505dbed2b2cb336)

running 3 tests
test list_books_is_empty_on_fresh_database ... ok
test borrowing_twice_returns_409_conflict_on_second_attempt ... ok
test borrow_full_round_trip_updates_row_in_real_postgres ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.31s
```

**สังเกตเวลา `0.31s`** เทียบกับ unit test ที่ `0.00s` — นี่คือต้นทุนจริงของการมี PostgreSQL อยู่ในสมการ: แต่ละ
`#[sqlx::test]` ต้อง `CREATE DATABASE` ใหม่, รัน migration, แล้ว `DROP DATABASE` ทิ้งหลังจบ (Part 70 หัวข้อ
70.13 อธิบายกลไกนี้ไว้แล้ว) ต้นทุนนี้คูณด้วยจำนวนเทสจะเห็นผลชัดเมื่อ suite โตขึ้น

**ทางเลือกที่บทนี้ไม่ได้ใช้ (แต่ควรรู้จักไว้)**: Part 71 หัวข้อ 71.10 สอนแพทเทิร์น "transaction ที่ไม่เคย
commit" ไว้แล้ว — เปิด `pool.begin()` หนึ่งครั้งต่อเทส ทำงานทุกอย่างผ่าน transaction นั้น แล้ว**ไม่เรียก
`commit()` เลย** ปล่อยให้ `Drop` ของ transaction handle สั่ง rollback ให้อัตโนมัติตอนจบ scope — วิธีนี้ **ไม่ต้อง
`CREATE DATABASE`/รัน migration ใหม่ทุกเทส** (ใช้ pool เดียวที่เปิดครั้งแรกครั้งเดียวได้ทั้ง suite) จึงเร็วกว่า
`#[sqlx::test]` มากสำหรับ suite ขนาดใหญ่ที่มีเทสหลักร้อย-พันตัว แลกกับความซับซ้อนที่ต้องเขียน isolation ให้ถูก
เองทั้งหมด (ทุก query ในเทสต้องยิงผ่าน `&mut *tx` ตัวเดียวกัน ไม่ใช่ `&pool` ตรง ๆ ไม่งั้นจะหลุดออกจาก
transaction ที่ตั้งใจจะ rollback) สำหรับแอปตัวแทนขนาดเล็กของบทนี้ (จำนวนเทสระดับหลักสิบ) ต้นทุนของ
`#[sqlx::test]` ยังน้อยพอที่จะไม่คุ้มความซับซ้อนที่ต้องแลก — แต่เมื่อ suite โตขึ้นถึงระดับหลักร้อย-พันตัวตาม
สัดส่วนพีระมิดในหัวข้อ 95.1 ควรกลับไปพิจารณา pattern นี้จาก Part 71 อีกครั้ง

### 95.5 Test Data Management: Fixture สำหรับ Backend และ E2E

หัวข้อ 95.3-95.4 สร้างข้อมูลทดสอบแบบ **ต่อเทส** (`InMemoryBookRepository::new(vec![...])` หรือ `INSERT` สด ๆ
ในแต่ละ `#[sqlx::test]`) ซึ่งเหมาะกับ unit/integration test ที่ต้องการ isolation สมบูรณ์ แต่ **e2e test มีข้อ
จำกัดที่ไม่เหมือนสองชั้นล่าง**: e2e test คุยกับ backend server ที่รันจริงหนึ่งตัว ซึ่งคุยกับฐานข้อมูลจริงหนึ่งลูก
— ข้อมูลจึงต้องมีอยู่ใน **ฐานข้อมูลจริงที่ทั้งระบบกำลังคุยด้วย ณ ขณะนั้น** ไม่ใช่ข้อมูลที่สร้างแล้วลบทิ้งในเทส
เดียวแบบ `#[sqlx::test]` (เพราะ backend server ที่รันอยู่คนละ process จากตัวเทส ไม่สามารถ "มองเห็น" transaction
ที่ยังไม่ commit ของอีก process ได้เลย)

Fixture สำหรับ e2e test ของบทนี้จึงเป็น SQL ตรง ๆ ที่รันครั้งเดียวก่อนเริ่ม suite ทั้งชุด (ไม่ใช่ต่อเทส):

```sql
-- e2e/seed.sql — รันหนึ่งครั้งก่อนเริ่ม e2e suite ทั้งชุด ไม่ใช่ต่อเทสแบบ #[sqlx::test]
INSERT INTO books (title, author, isbn, available) VALUES
    ('The Rust Programming Language', 'Steve Klabnik & Carol Nichols', '978-1-59327-828-1', true),
    ('Programming Rust', 'Jim Blandy & Jason Orendorff', '978-1-49192-793-9', false),
    ('Zero To Production In Rust', 'Luca Palmieri', '978-1-91282-820-8', true);
```

รันจริงแล้วตรวจสอบ:

```
$ psql -d library_e2e -c "SELECT id, title, available FROM books;"
 id |             title             | available
----+-------------------------------+-----------
  1 | The Rust Programming Language | t
  2 | Programming Rust              | f
  3 | Zero To Production In Rust    | t
(3 rows)
```

จุดที่ต้องระวังที่สุดของ fixture แบบนี้คือ **frontend กับ backend ต้องเห็นข้อมูลชุดเดียวกันเป๊ะ** — ถ้า
frontend component test (หัวข้อ 95.7) ใช้ fixture Rust struct `Book { id: 1, title: "The Rust Programming
Language", ... }` ที่เขียนแยกจาก SQL fixture ข้างบน แล้ววันหนึ่งมีคนแก้ SQL fixture (เช่นเปลี่ยนชื่อหนังสือ)
โดยไม่แก้ struct ฝั่ง frontend ให้ตรงกัน **e2e test จะ fail แบบที่ดูเหมือนบั๊กจริงทั้งที่จริง ๆ คือ fixture
สองฝั่งไม่ sync กัน** — วิธีป้องกันที่ใช้ได้จริงมีสองแนวทาง: (1) ให้ e2e test ยืนยันเนื้อหาแบบ "อ่านจาก DOM ที่
render จริง แล้วเทียบกับค่าที่ query จาก DB จริงในเทสเดียวกัน" (ไม่ hardcode ค่าคาดหวังไว้ล่วงหน้าเป็น string
ตรง ๆ ทั้งสองด้าน) หรือ (2) เก็บ fixture ไว้ **แหล่งเดียว** (เช่นไฟล์ SQL หรือ JSON เดียว) แล้วให้ทั้ง e2e test
script และ SQL seed อ่านจากแหล่งนั้นร่วมกัน — บทนี้ใช้แนวทาง (1) ในหัวข้อ 95.8 (ดูโค้ด e2e test ที่ query ชื่อ
หนังสือจาก DOM แล้วเทียบกับ literal string ที่ตรงกับ SQL fixture โดยตรง เพื่อให้เห็นทั้งสองด้านอยู่ใกล้กัน
ตรวจสอบ sync กันได้ง่ายด้วยตา)

### 95.6 ทดสอบ WASM/Frontend Logic ด้วย `wasm-pack test`: `--node` vs `--headless --chrome`

Part 86 แนะนำ `wasm-pack test --node` ไว้แบบสั้น ๆ แล้ว พร้อมพูดถึงแฟล็ก `--chrome`/`--firefox`/`--safari` ไว้
เป็นชื่อ — หัวข้อนี้ขยายให้เห็นความแตกต่างเชิงกลไกที่แท้จริงระหว่างสองโหมด เพราะมันเป็นจุดที่มือใหม่เข้าใจผิดบ่อย

Crate `library_ui` (frontend ของแอปตัวแทน) ตั้งค่า `Cargo.toml` ตามแนวทาง CSR ของ Part 89 บวก dev-dependency
`wasm-bindgen-test` ตามที่ Part 86 แนะนำไว้:

```toml
[package]
name = "library_ui"
version = "0.1.0"
edition = "2021"

[dependencies]
leptos = { version = "0.8", features = ["csr"] }
leptos_router = "0.8"
serde = { version = "1", features = ["derive"] }
gloo-net = "0.6"
wasm-bindgen = "0.2"
web-sys = { version = "0.3", features = ["Window", "Document", "Element", "HtmlElement", "Node"] }

[dev-dependencies]
wasm-bindgen-test = "0.3"
```

สังเกตว่า `library_ui` มีทั้ง `src/lib.rs` (เก็บ component และ pure logic — testable) และ `src/main.rs` (thin
wrapper ที่เรียก `mount()`) ตาม pattern เดียวกับ backend ในหัวข้อ 95.2 — เหตุผลเดียวกันเป๊ะ: `trunk` ต้องการ
binary crate ("bin" target) ในการ build เป็นแอปเว็บจริง แต่ `wasm-pack test` และ `cargo check --target
wasm32-unknown-unknown` ทำงานกับ **lib** target ได้สะดวกกว่า (เพราะ `tests/` ผูกกับ lib crate ตามกฎ Part 33)
การมีทั้งสองแยกกันทำให้ได้ทั้งสองประโยชน์โดยไม่ต้องแลกอะไร

ฟังก์ชันที่จะทดสอบเป็น pure logic ล้วน ๆ ที่ frontend ใช้ตรวจสอบ input ก่อนส่งไปให้ server function/API (แนวคิด
เดียวกับ client-side validation ที่ลด round trip ไปเซิร์ฟเวอร์โดยไม่จำเป็น):

```rust
/// ฟังก์ชัน validation ล้วนๆ ไม่แตะ DOM/network เลย
pub fn validate_borrow_days(days: i64) -> Result<(), String> {
    if days <= 0 {
        return Err("จำนวนวันต้องมากกว่า 0".to_string());
    }
    if days > 30 {
        return Err("ยืมได้ไม่เกิน 30 วันต่อครั้ง".to_string());
    }
    Ok(())
}

pub fn availability_label(available: bool) -> &'static str {
    if available { "พร้อมให้ยืม" } else { "ถูกยืมไปแล้ว" }
}
```

```rust
// tests/web.rs — ไม่ใส่ wasm_bindgen_test_configure!(run_in_browser) เลย
use library_ui::{availability_label, validate_borrow_days};
use wasm_bindgen_test::*;

#[wasm_bindgen_test]
fn zero_days_is_rejected() {
    assert_eq!(validate_borrow_days(0), Err("จำนวนวันต้องมากกว่า 0".to_string()));
}

#[wasm_bindgen_test]
fn thirty_days_is_the_maximum_allowed() {
    assert!(validate_borrow_days(30).is_ok());
    assert!(validate_borrow_days(31).is_err());
}

#[wasm_bindgen_test]
fn availability_label_matches_backend_semantics() {
    assert_eq!(availability_label(true), "พร้อมให้ยืม");
    assert_eq!(availability_label(false), "ถูกยืมไปแล้ว");
}
```

รันจริงด้วย `wasm-pack test --node`:

```
[INFO]: Installing wasm-bindgen...
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.25s
     Running unittests src/lib.rs (target/wasm32-unknown-unknown/debug/deps/library_ui-....wasm)
no tests to run!
     Running unittests src/main.rs (target/wasm32-unknown-unknown/debug/deps/library_ui-....wasm)
no tests to run!
     Running tests/web.rs (target/wasm32-unknown-unknown/debug/deps/web-bb162c5b146df9c8.wasm)
running 5 tests
test zero_days_is_rejected ... ok
test typical_borrow_period_is_accepted ... ok
test thirty_days_is_the_maximum_allowed ... ok
test negative_days_is_rejected ... ok
test availability_label_matches_backend_semantics ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 filtered out; finished in 0.02s
```

ทั้ง 5 เทสผ่านหมด รันจบใน `0.02s` — ช้ากว่า native unit test ในหัวข้อ 95.3 (ที่ `0.00s`) ประมาณ 20 เท่า แม้ logic
ที่ทดสอบจะเรียบง่ายพอกัน เพราะมีขั้นตอนพิเศษ (compile เป็น wasm32, โหลดเข้า Node.js `WebAssembly` runtime) ที่
native test ไม่ต้องทำ — นี่คือหลักฐานจริงที่ยืนยันตารางในหัวข้อ 95.1 ว่าทำไม WASM test ต้องแยกชั้นออกจาก unit
test ปกติ

#### `wasm_bindgen_test_configure!(run_in_browser)`: ตัวกำหนด target ของทั้งไฟล์ ไม่ใช่ auto-detect

จุดที่เข้าใจผิดง่ายที่สุด (และผู้เขียนบทนี้เองก็เข้าใจผิดตอนแรกก่อนตรวจสอบจริง — ควรบันทึกไว้ตรง ๆ): macro
`wasm_bindgen_test_configure!(run_in_browser)` **ไม่ใช่การ "เปิดให้รันได้ทั้งสองแบบ"** มันคือการ**บังคับ**ว่า
ไฟล์เทสนี้ทั้งไฟล์ต้องรันในเบราว์เซอร์เท่านั้น ลองรันไฟล์ `tests/web.rs` ข้างบน (ที่ไม่มี macro นี้) ด้วย
`--headless --chrome` ดูจริง:

```
$ wasm-pack test --headless --chrome
     Running tests/web.rs (target/wasm32-unknown-unknown/debug/deps/web-....wasm)
This test suite is only configured to run in node.js, but we're only running
    browser tests so skipping. If you'd like to run the tests in a browser
    include this in your crate when testing:

        wasm_bindgen_test::wasm_bindgen_test_configure!(run_in_browser);

    You'll likely want to put that in a `#[cfg(test)]` module or at the top of an
    integration test.
```

ข้อความนี้ชี้ชัดว่า **ไฟล์ที่ไม่มี macro configure จะถูกมองว่า "ตั้งใจให้รันใน Node.js เท่านั้น" โดย default** —
ถ้าอยากให้เทสเดียวกันรันได้ทั้งสองบริบท ต้อง**แยกไฟล์** (ตามกฎ Part 33 ที่ไฟล์ใน `tests/` แต่ละไฟล์คือ crate
อิสระของตัวเองอยู่แล้ว) แล้วใส่ macro ในไฟล์ที่จะรันบนเบราว์เซอร์:

```rust
// tests/web_browser.rs — เนื้อหาเดียวกับ tests/web.rs เป๊ะ แต่เปิด run_in_browser
use library_ui::validate_borrow_days;
use wasm_bindgen_test::*;

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
fn zero_days_is_rejected_in_browser() {
    assert!(validate_borrow_days(0).is_err());
}

#[wasm_bindgen_test]
fn thirty_days_is_the_maximum_allowed_in_browser() {
    assert!(validate_borrow_days(30).is_ok());
    assert!(validate_borrow_days(31).is_err());
}
```

การรันเบราว์เซอร์จริงแบบ headless ต้องมี WebDriver (chromedriver/geckodriver) คอยควบคุมเบราว์เซอร์ตาม protocol
ของมัน (คนละกลไกกับ Playwright ในหัวข้อ 95.8 ที่คุยกับเบราว์เซอร์ผ่าน CDP โดยตรง ไม่ผ่าน WebDriver) — และถ้า
ไบนารีเบราว์เซอร์ไม่ได้อยู่ใน path ที่ chromedriver คาดหวังไว้ ต้องระบุผ่านไฟล์ `webdriver.json` ที่ root ของ
crate:

```json
{
  "goog:chromeOptions": {
    "binary": "/opt/pw-browsers/chromium-1194/chrome-linux/chrome",
    "args": ["--no-sandbox", "--disable-dev-shm-usage"]
  }
}
```

**รายงานตรงไปตรงมาถึงผลลัพธ์ที่พบจริงในสภาพแวดล้อมที่เขียนบทนี้**: หลังตั้ง `webdriver.json` แล้วลองรัน
`wasm-pack test --headless --chrome` จริง พบว่า chromedriver ที่ติดตั้งไว้ในสภาพแวดล้อมนี้เป็นเวอร์ชัน
`147.0.7727.24` ในขณะที่ไบนารี Chromium ที่มีอยู่จริง (ตัวที่ Playwright ติดตั้งไว้) เป็นเวอร์ชัน `141.0.7390.37`
— chromedriver ปฏิเสธ session ทันทีด้วย error จริง:

```
Error: failed to create a Chrome session: session not created: This version of
ChromeDriver only supports Chrome version 147
Current browser version is 141.0.7390.37 with binary path
/opt/pw-browsers/chromium-1194/chrome-linux/chrome
```

และเมื่อลองหาไบนารี Chrome ที่ตรงเวอร์ชันกับ chromedriver ผ่านการดาวน์โหลดจาก Chrome for Testing API ก็พบว่า
เครือข่ายในสภาพแวดล้อมนี้ปฏิเสธการเชื่อมต่อไปยังโดเมนนั้น (`connect_rejected` จาก egress proxy) ทำให้ไม่มีทาง
แก้ปัญหานี้จากภายในสภาพแวดล้อมนี้ได้ — **นี่คือตัวอย่างจริงของปัญหา infrastructure-level ที่ทำให้การทดสอบผ่าน
เบราว์เซอร์จริง flaky/ล้มเหลวได้โดยไม่เกี่ยวกับบั๊กในโค้ดเลยแม้แต่นิดเดียว** ซึ่งสอดคล้องกับสิ่งที่ Part 88/90
เจอมาก่อนแล้วเช่นกัน (ทั้งสองบทนั้นเลือกใช้ **Playwright** ควบคุมเบราว์เซอร์จากภายนอกแทน `wasm-bindgen-test`'s
chromedriver-based launcher โดยเจตนา ด้วยเหตุผลเดียวกันนี้) — บทนี้จึงใช้ `wasm-pack test --node` เป็นเส้นทาง
หลักที่ยืนยันได้จริง 100% สำหรับ pure logic (อย่างที่เห็นผลลัพธ์ 5/5 ผ่านข้างบน) และย้ายการทดสอบที่ต้องพึ่ง DOM
จริงไปให้ Playwright จัดการทั้งระบบในหัวข้อ 95.8 แทน ซึ่งเป็นทางเลือกที่ยืนยันได้จริงในสภาพแวดล้อมนี้ (Chromium
ที่ Playwright ติดตั้งเองมาพร้อมกับตัวมันโดยไม่ผ่าน chromedriver เลย จึงไม่เจอปัญหาเวอร์ชันไม่ตรงกันแบบนี้)

**ข้อสรุปเชิงปฏิบัติที่ควรจำจากเหตุการณ์นี้**: ในสภาพแวดล้อม CI/container ที่ไม่ได้ควบคุมเวอร์ชัน
เบราว์เซอร์+driver ให้ตรงกันตลอดเวลา (เช่น image ที่อัปเดต Chromium แต่ไม่อัปเดต chromedriver คู่กัน) การพึ่งพา
`wasm-pack test --headless` มีความเสี่ยงที่จะพังจาก environment ล้วน ๆ โดยไม่เกี่ยวกับโค้ดเลย — โค้ด production
เดียวกัน อาจ pass ในเครื่อง dev ของคนหนึ่ง (ที่ chromedriver/chromium sync กันพอดี) แต่ fail ใน CI (ที่ image
อัปเดตไม่ตรงจังหวะกัน) นี่คือเหตุผลเชิงลึกอีกข้อที่ทำให้ทีมจำนวนมากเลือกให้ pure-logic test วิ่งผ่าน `--node`
เป็นหลัก (ไม่มี browser driver ให้ต้อง sync เวอร์ชันเลย) แล้วเก็บการทดสอบที่ต้องพึ่งเบราว์เซอร์จริงไว้ให้ e2e
layer เพียงชั้นเดียว ที่ทีม infra ดูแลเวอร์ชันเบราว์เซอร์ให้สอดคล้องกันอย่างตั้งใจ (เช่นผ่าน Playwright ที่
ผูกเวอร์ชันเบราว์เซอร์ไว้กับตัว npm package เอง ไม่ต้องพึ่ง system driver ที่อาจไม่ sync กัน)

### 95.7 Component Testing: Leptos, Yew, Dioxus เทียบความอยู่ตัว

Part 88 หัวข้อ 88.11 บอกไว้ตรง ๆ ว่าเนื้อหาการทดสอบ component เต็มรูปแบบจะถูกรวบรวมในบทนี้ พร้อมพูดถึงว่า Yew
มี crate เสริมชื่อ `yew::functional::test` "และ helper อื่นในบางเวอร์ชัน" — คำว่า **"ในบางเวอร์ชัน"** ตรงนั้น
คือกุญแจสำคัญของหัวข้อนี้: **ความอยู่ตัว (maturity) ของ component-testing tooling ในระบบนิเวศ Rust/WASM ยังไม่
เท่ากันข้ามเฟรมเวิร์ก และเปลี่ยนแปลงเร็วกว่า tooling ฝั่ง backend มาก** ต้องพูดตรงไปตรงมาแทนที่จะทำให้ดูเหมือน
ทุกเฟรมเวิร์กมี test utility ระดับเดียวกัน

#### แนวทางที่ 1 — Render เป็น string แล้วตรวจ (เร็วที่สุด, อยู่ตัวที่สุด, ไม่ต้องมี WASM เลย)

Leptos ถูกออกแบบมาให้ SSR เป็นความสามารถหลักตั้งแต่ต้น (ตามที่ Part 89 สอน) ทำให้การ render component เป็น
HTML string เพื่อตรวจสอบเนื้อหาเป็นเรื่องปกติที่ crate จัดให้พร้อมใช้ — และข้อดีที่สำคัญมากคือ **มันรันด้วย
`cargo test` ธรรมดาบน native target ได้เลย ไม่ต้อง compile เป็น wasm32 เลยแม้แต่นิดเดียว** (ต้องเปิด feature
`ssr` ของ `leptos` แทน `csr`/`hydrate` — สังเกตว่านี่คือ feature คนละตัวกับที่ frontend จริงใช้ ดังนั้นเทสแบบนี้
มักอยู่ในโปรเจกต์/crate แยกจาก crate หลักที่ compile เป็น WASM จริง เพื่อไม่ให้ feature ชนกัน):

```toml
[dependencies]
leptos = { version = "0.8", features = ["ssr"] }
```

```rust
use leptos::prelude::*;

#[derive(Debug, Clone, PartialEq)]
pub struct Book {
    pub id: i64,
    pub title: String,
    pub author: String,
    pub available: bool,
}

fn availability_label(available: bool) -> &'static str {
    if available { "พร้อมให้ยืม" } else { "ถูกยืมไปแล้ว" }
}

#[component]
pub fn BookCard(book: Book) -> impl IntoView {
    view! {
        <li class="book-card">
            <span class="book-title">{book.title.clone()}</span>
            " โดย " <span class="book-author">{book.author.clone()}</span>
            " — " <span class="book-status">{availability_label(book.available)}</span>
        </li>
    }
}

#[cfg(test)]
mod tests {
    //! Component test แบบ render-to-string — วิธีที่ "อยู่ตัว/mature" ที่สุดของ Leptos ในการ
    //! ทดสอบว่า component render ผลลัพธ์ถูกต้อง โดยไม่ต้องมี browser, ไม่ต้องมี WASM,
    //! รันด้วย `cargo test` ธรรมดาบน native target ได้เลย (เร็วกว่า wasm-pack test มาก)
    use super::*;

    #[test]
    fn renders_title_author_and_available_status() {
        let book = Book { id: 1, title: "The Rust Programming Language".into(), author: "Klabnik & Nichols".into(), available: true };
        let html = view! { <BookCard book=book /> }.to_html();

        assert!(html.contains("The Rust Programming Language"), "html: {html}");
        assert!(html.contains("พร้อมให้ยืม"), "html: {html}");
    }

    #[test]
    fn renders_borrowed_status_when_not_available() {
        let book = Book { id: 2, title: "Programming Rust".into(), author: "Blandy".into(), available: false };
        let html = view! { <BookCard book=book /> }.to_html();

        assert!(html.contains("ถูกยืมไปแล้ว"), "html: {html}");
        assert!(!html.contains("พร้อมให้ยืม"), "html ไม่ควรมีคำว่าพร้อมให้ยืมเมื่อ available=false: {html}");
    }
}
```

รันจริงด้วย `cargo test` (native, ไม่มี wasm32 เกี่ยวข้องเลย):

```
     Running unittests src/lib.rs (target/debug/deps/ssr_render_demo-3a2f6f95c0780af1)

running 2 tests
test tests::renders_title_author_and_available_status ... ok
test tests::renders_borrowed_status_when_not_available ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

`0.00s` — เร็วเท่า unit test ปกติของ Part 32 เลย เพราะไม่มีขั้นตอน WASM เข้ามาเกี่ยวข้องแม้แต่นิดเดียว (`.to_html()`
แค่ render `view!` เป็น `String` ธรรมดาบน native code) นี่คือเหตุผลที่ควรใช้วิธีนี้เป็นหลักเมื่อสิ่งที่ต้องการ
ตรวจสอบคือ "component render เนื้อหา/attribute ที่ถูกต้องไหม" — ไม่ต้องพึ่ง DOM จริงเลย

#### แนวทางที่ 2 — Mount ลง DOM จริงในเบราว์เซอร์ (manual ที่สุด, จำเป็นเมื่อต้องทดสอบ interaction)

เมื่อต้องทดสอบสิ่งที่ render-to-string ทำไม่ได้ (เช่น "คลิกปุ่มแล้ว state เปลี่ยนไหม", "event handler ทำงานไหม")
ต้อง mount component ลง DOM จริงในเบราว์เซอร์ผ่าน `wasm-bindgen-test` — นี่คือระดับที่ "manual" ที่สุด เพราะต้อง
เขียนโค้ด query DOM ด้วยมือผ่าน `web_sys` เอง ไม่มี assertion helper ระดับสูงให้ (ต่างจาก JavaScript ecosystem
ที่มี Testing Library คอยช่วย):

```rust
//! tests/dom.rs — ต้องรันในเบราว์เซอร์เท่านั้น (wasm_bindgen_test_configure!(run_in_browser))
use leptos::prelude::*;
use library_ui::{availability_label, Book};
use wasm_bindgen::JsCast;
use wasm_bindgen_test::*;

wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
fn book_status_text_renders_available_label() {
    let window = web_sys::window().expect("ต้องมี window ในบริบทเบราว์เซอร์เท่านั้น");
    let document = window.document().expect("ต้องมี document");
    let container = document.create_element("div").unwrap();
    document.body().unwrap().append_child(&container).unwrap();

    let book = Book { id: 1, title: "The Rust Programming Language".to_string(), author: "Klabnik & Nichols".to_string(), isbn: "978-1".to_string(), available: true };
    let label = availability_label(book.available);

    let html_container: web_sys::HtmlElement = container.clone().unchecked_into();
    let _handle = leptos::mount::mount_to(html_container, move || {
        view! { <p class="status">{label}</p> }
    });

    let rendered = container.text_content().unwrap_or_default();
    assert!(rendered.contains("พร้อมให้ยืม"), "คาดว่า DOM จะมีคำว่า 'พร้อมให้ยืม' แต่ได้ '{rendered}'");

    document.body().unwrap().remove_child(&container).unwrap();
}
```

โค้ดนี้ compile ผ่านสำหรับ `wasm32-unknown-unknown` ได้จริง (ตรวจสอบด้วย `cargo check --target
wasm32-unknown-unknown --tests` แล้ว) แต่การรันจริงผ่าน `wasm-pack test --headless --chrome` ในสภาพแวดล้อมที่
เขียนบทนี้เจอ blocker เดียวกันกับที่รายงานไว้ในหัวข้อ 95.6 (chromedriver/Chromium เวอร์ชันไม่ตรงกัน ไม่มีทาง
ดาวน์โหลด driver ที่ตรงเวอร์ชันได้เพราะ network policy) — **นี่คือหลักฐานที่ตอกย้ำประเด็นสำคัญของหัวข้อนี้ตรงๆ**:
การทดสอบ component แบบ mount-DOM ต้องพึ่งพา browser automation infrastructure ที่ซับซ้อนกว่า render-to-string
มาก (ต้องมี browser + driver ที่ sync เวอร์ชันกัน) ทำให้มันมีค่าใช้จ่ายด้าน infrastructure สูงกว่าและ flaky
ง่ายกว่าโดยธรรมชาติ ไม่ใช่แค่ "ช้ากว่า" อย่างเดียว

#### เทียบความอยู่ตัวของ component testing ข้ามสามเฟรมเวิร์ก

| เฟรมเวิร์ก | Render-to-string (native, ไม่ต้อง WASM) | Mount ลง DOM จริง (ต้องมี browser) | หมายเหตุความอยู่ตัว |
|---|---|---|---|
| **Leptos** | มี (feature `ssr`, `.to_html()`) — verify แล้วในหัวข้อนี้ ใช้งานได้จริงลื่นไหล | มี (`leptos::mount::mount_to`) แต่ต้องเขียน DOM query ด้วยมือทั้งหมด | SSR เป็น first-class citizen ของเฟรมเวิร์กมาตั้งแต่ต้น ทำให้ render-to-string ลื่นไหลกว่าอีกสองตัวชัดเจน |
| **Yew** | ไม่มีเส้นทาง SSR-string ที่ตรงไปตรงมาเท่า Leptos ในการ "render component เดี่ยวๆ เป็น string เพื่อเทส" | ผ่าน `wasm-bindgen-test` + `web-sys` เหมือน Leptos; Part 88 หัวข้อ 88.11 กล่าวถึง `yew::functional::test` ว่ามีอยู่ "ในบางเวอร์ชัน" — ควรตรวจสอบกับเวอร์ชัน Yew ที่ใช้จริงเสมอก่อนเชื่อว่า API นี้มีอยู่ | Part 88 ยังไม่ verify ตัวช่วยนี้ตรงๆในบทนั้น (บอกไว้แค่ชื่อ) — ผู้อ่านที่จะใช้จริงควรอ่าน docs.rs ของเวอร์ชัน Yew ที่ตนใช้อีกครั้งก่อนพึ่งพา API นี้ |
| **Dioxus** | มี crate `dioxus-ssr` แยกต่างหาก (Part 90 อ้างถึงไว้ตอนพูดถึง `ssr` feature — "render เป็น HTML string ตรงๆ บน server") ให้ผลลัพธ์คล้ายแนวทางของ Leptos | ยังต้องพึ่ง WASM + browser automation เหมือนอีกสองตัวสำหรับการทดสอบ interaction จริงในเบราว์เซอร์ | มี renderer แยกจาก `dioxus-core`/`VirtualDom` โดยเจตนา (ตามที่ Part 90 อธิบายสถาปัตยกรรม) ทำให้ render-to-string ทำได้โดยธรรมชาติของการออกแบบ เหมือน Leptos |

**ข้อสรุปที่ตรงไปตรงมาที่สุด**: ทั้งสามเฟรมเวิร์ก**ไม่มีตัวใดมี test utility ระดับสูงแบบ React Testing
Library/Vue Test Utils ของฝั่ง JavaScript** (ที่มี query แบบ `getByText`, `fireEvent`, assertion เชิง
semantic ให้ใช้ตรง ๆ) — สิ่งที่มีให้ในระบบนิเวศ Rust/WASM วันนี้คือ (ก) render-to-string ระดับ native ที่
ลื่นไหลมากสำหรับ Leptos/Dioxus (เพราะทั้งสองออกแบบให้ SSR เป็นจุดขายหลักตั้งแต่ต้น) และ (ข) mount-DOM ผ่าน
`wasm-bindgen-test` + `web_sys` แบบ manual ล้วน ๆ ที่ใช้ได้กับทุกเฟรมเวิร์กเหมือนกันแต่ต้องเขียน query/assertion
เองทุกจุด ไม่มี abstraction ช่วยเลย — คำแนะนำเชิงปฏิบัติคือ **ใช้ render-to-string ให้มากที่สุดสำหรับตรวจสอบ
"component render เนื้อหาถูกไหม" แล้วเก็บการทดสอบ interaction จริงไว้ให้ e2e layer (หัวข้อ 95.8) ที่ได้ความมั่นใจ
สูงกว่าต่อหน่วยความซับซ้อนที่ต้องเขียน**

#### ทางเลือกที่ไม่ได้ลงรายละเอียดในบทนี้: Mock การเรียก `fetch` ระดับ component

อีกเทคนิคหนึ่งที่พบในโปรเจกต์ frontend Rust จริง (แต่บทนี้ไม่ได้ลงรายละเอียดเพราะซับซ้อนเกินขอบเขต) คือการ mock
ชั้น HTTP client (`gloo-net::http::Request` ในตัวอย่างของบทนี้) ด้วย trait เดียวกับที่หัวข้อ 95.3 ใช้กับ
`BookRepository` ฝั่ง backend — นิยาม trait `BookApiClient` ที่ห่อ `fetch_books`/`fetch_book` ไว้ แล้ว inject
ของปลอมเข้าไปตอนเทส component โดยไม่ต้องมี backend จริงหรือ network เรียกเลย วิธีนี้ทำให้ทดสอบ `BookList` /
`BookDetail` ได้ "isolated" อย่างแท้จริงในความหมายของ Part 32 (ไม่ใช่แค่ render-to-string ที่ยังต้องพึ่งค่า
`Book` struct ที่สร้างไว้ล่วงหน้าตรง ๆ) แต่ต้องแลกกับความซับซ้อนที่เพิ่มขึ้น (ต้อง generic-ize component เหนือ
trait นั้น หรือใช้ dependency injection ผ่าน Context ของ Leptos) — สำหรับแอปขนาดเล็กถึงกลาง render-to-string
ธรรมดาให้ความคุ้มค่าต่อความซับซ้อนที่ดีกว่า แต่สำหรับ design system/component library ขนาดใหญ่ที่มี component
จำนวนมากที่ต้องทดสอบ "การจัดการ loading/error state" อย่างละเอียด เทคนิคนี้อาจคุ้มค่าที่จะลงทุนเพิ่ม

### 95.8 End-to-End Testing ด้วย Playwright: ขับเบราว์เซอร์จริงผ่านทั้งระบบ

ถึงตอนนี้เราทดสอบ backend (isolated + กับ DB จริง) และ frontend logic (isolated) แยกกันหมดแล้ว — สิ่งที่ยังไม่
มีใครพิสูจน์คือ **ทั้งสองส่วนนี้ทำงานร่วมกันได้จริงหรือไม่เมื่อรันเป็นระบบเดียวกัน** ผ่าน browser จริง นี่คือ
งานของ e2e test

บทนี้เลือก **Playwright** เป็นเครื่องมือหลัก (ไม่ใช่ `fantoccini`/`thirtyfour` ที่เป็น WebDriver-based Rust
crate) ด้วยเหตุผลที่ตรวจสอบได้จริงในสภาพแวดล้อมนี้: Playwright คุยกับเบราว์เซอร์ผ่าน Chrome DevTools Protocol
(CDP) โดยตรง ไม่ผ่าน WebDriver/chromedriver เลย จึงไม่เจอปัญหาเวอร์ชันไม่ sync ที่หัวข้อ 95.6-95.7 เจอมาแล้วสอง
ครั้ง — และตรงกับที่ Part 88/90 เลือกใช้ยืนยันผลลัพธ์ของตัวเองไปแล้ว (เวอร์ชัน `1.56.1` เดียวกัน) ทำให้บทนี้
สอดคล้องกับ convention ที่ตั้งไว้แล้วในโมดูลเดียวกัน ทางเลือก `fantoccini`/`thirtyfour` (ที่เขียน e2e test เป็น
Rust ล้วน ๆ ไม่ต้องพึ่ง Node.js) ยังคงเป็นตัวเลือกที่ใช้ได้จริงในโปรเจกต์ที่ต้องการ e2e test เป็น Rust ทั้งชุด
แต่ต้องพึ่งพา WebDriver server (chromedriver/geckodriver) ที่หัวข้อ 95.6 เพิ่งพิสูจน์แล้วว่ามีความเสี่ยงเรื่อง
เวอร์ชันไม่ sync ในสภาพแวดล้อม container ที่ไม่ได้ควบคุมทั้งสองอย่างให้อัปเดตพร้อมกันเสมอ

#### ทางเลือก: e2e ทั้งชุดเป็น Rust ล้วนๆ ด้วย `fantoccini`/`thirtyfour`

เพื่อความครบถ้วน ควรเห็นหน้าตาของทางเลือกที่เขียน e2e test เป็น Rust ทั้งชุดไว้ด้วย แม้บทนี้จะไม่ได้เลือกใช้จริง
— `fantoccini` และ `thirtyfour` เป็นสอง crate หลักที่คุยกับเบราว์เซอร์ผ่าน **WebDriver protocol มาตรฐาน**
(protocol เดียวกับที่ Selenium ใช้มานาน) ต่างจาก Playwright ที่คุยผ่าน CDP ตรง ๆ:

```rust
// ตัวอย่างโครงสร้างของ fantoccini (สาธิตหน้าตา API เท่านั้น — ไม่ได้ compile/run จริงในบทนี้
// เพราะบทนี้เลือก Playwright ตามเหตุผลที่อธิบายไว้ข้างบน แต่โค้ดนี้เขียนถูกไวยากรณ์ Rust ปัจจุบัน)
use fantoccini::{ClientBuilder, Locator};

#[tokio::test]
async fn book_list_shows_seeded_books() -> Result<(), fantoccini::error::CmdError> {
    // ต้องมี chromedriver/geckodriver รันอยู่ก่อนแล้วที่พอร์ตนี้ (fantoccini ไม่ spawn ให้เอง)
    let client = ClientBuilder::native()
        .connect("http://localhost:9515")
        .await
        .expect("เชื่อมต่อ WebDriver server ไม่ได้ — ต้องรัน chromedriver ไว้ก่อน");

    client.goto("http://127.0.0.1:8123").await?;
    let list = client.wait().for_element(Locator::Css(r#"[data-testid="book-list"]"#)).await?;
    let text = list.text().await?;
    assert!(text.contains("The Rust Programming Language"));

    client.close().await
}
```

ข้อดีของแนวทางนี้คือ **e2e test เขียนด้วยภาษาเดียวกับทั้ง backend/frontend ทั้ง stack** ไม่ต้องสลับไปเขียน
JavaScript/TypeScript เลย (มีประโยชน์ถ้าทีมต้องการ codebase ภาษาเดียวจริง ๆ) แต่ข้อเสียที่สำคัญคือ **ต้องมี
WebDriver server (chromedriver/geckodriver) รันอยู่แยกต่างหากเสมอ** — `fantoccini`/`thirtyfour` เป็นแค่ client
ที่คุย protocol เท่านั้น ไม่ได้ผูกเวอร์ชัน browser+driver ไว้ให้เหมือนที่ Playwright npm package ทำ ดังนั้นความ
เสี่ยงเรื่องเวอร์ชันไม่ sync ที่พิสูจน์แล้วในหัวข้อ 95.6-95.7 จะยังคงเป็นปัญหาเดิมสำหรับแนวทางนี้ด้วยเช่นกัน —
ทีมที่เลือกใช้ `fantoccini`/`thirtyfour` ต้องรับผิดชอบดูแลให้ browser+driver sync เวอร์ชันกันเองอย่างต่อเนื่อง
(เช่น pin ทั้งสองไว้ใน Dockerfile เดียวกันที่อัปเดตพร้อมกันเสมอ — เรื่องนี้จะเกี่ยวกับ Part 96 ที่กำลังจะเริ่ม
สอน Docker โดยตรง) ซึ่งเป็นต้นทุนการดูแลที่มากกว่าการใช้ Playwright ที่จัดการเรื่องนี้ให้อัตโนมัติ

#### สร้าง user flow ที่จะทดสอบ: ดูรายการหนังสือ → คลิกดูรายละเอียด

Frontend ประกอบด้วยสอง route ผ่าน `leptos_router` (`csr` feature ตาม "แนวทางที่ 1" ของ Part 89):

```rust
#[component]
pub fn BookList() -> impl IntoView {
    let books = LocalResource::new(fetch_books);

    view! {
        <div>
            <h1>"รายการหนังสือในห้องสมุด"</h1>
            <Suspense fallback=|| view! { <p data-testid="loading">"กำลังโหลด..."</p> }>
                {move || {
                    books.get().map(|result| match result {
                        Ok(list) => view! {
                            <ul data-testid="book-list">
                                <For each=move || list.clone() key=|book| book.id
                                     children=move |book| view! { <BookCard book=book /> } />
                            </ul>
                        }.into_any(),
                        Err(err) => view! { <p data-testid="error">{format!("โหลดไม่สำเร็จ: {err}")}</p> }.into_any(),
                    })
                }}
            </Suspense>
        </div>
    }
}
```

```rust
#[component]
pub fn BookCard(book: Book) -> impl IntoView {
    view! {
        <li class="book-card" data-testid="book-card">
            <A href=format!("/books/{}", book.id)>
                <span class="book-title">{book.title.clone()}</span>
            </A>
            " โดย " <span class="book-author">{book.author.clone()}</span>
            " — " <span class="book-status">{availability_label(book.available)}</span>
        </li>
    }
}
```

```rust
#[component]
pub fn BookDetail() -> impl IntoView {
    let params = use_params::<BookIdParams>();
    let book_id = move || params.read().as_ref().ok().and_then(|p| p.id).unwrap_or_default();
    let book = LocalResource::new(move || fetch_book(book_id()));

    view! {
        <div>
            <A href="/">"< กลับไปหน้ารายการ"</A>
            <Suspense fallback=|| view! { <p data-testid="loading">"กำลังโหลด..."</p> }>
                {move || {
                    book.get().map(|result| match result {
                        Ok(b) => view! {
                            <div data-testid="book-detail">
                                <h1>{b.title.clone()}</h1>
                                <p data-testid="book-author">"ผู้เขียน: " {b.author.clone()}</p>
                                <p data-testid="book-status">{availability_label(b.available)}</p>
                            </div>
                        }.into_any(),
                        Err(err) => view! { <p data-testid="error">{format!("โหลดไม่สำเร็จ: {err}")}</p> }.into_any(),
                    })
                }}
            </Suspense>
        </div>
    }
}
```

สังเกตแอตทริบิวต์ `data-testid="..."` ที่แทรกไว้ในทุก element สำคัญ — นี่คือ convention ที่ใช้กันแพร่หลายใน e2e
testing (ไม่ผูก selector กับ CSS class ที่อาจเปลี่ยนเพื่อจุดประสงค์ด้าน styling โดยไม่เกี่ยวกับ testing เลย)
ทำให้ e2e test อ่านง่ายและทนทานต่อการ refactor UI มากกว่าการ query ด้วย CSS class/tag ตรง ๆ

ประกอบสอง component เข้าด้วยกันผ่าน `leptos_router` (route `""` → รายการ, route `books/{id}` → รายละเอียด) แล้ว
mount เข้า `<body>` จริง:

```rust
#[component]
pub fn App() -> impl IntoView {
    view! {
        <Router>
            <Routes fallback=|| view! { <p>"ไม่พบหน้านี้"</p> }>
                <Route path=StaticSegment("") view=BookList />
                <Route path=(StaticSegment("books"), ParamSegment("id")) view=BookDetail />
            </Routes>
        </Router>
    }
}

pub fn mount() {
    leptos::mount::mount_to_body(App);
}
```

```rust
// frontend/src/main.rs — เปลือกบางที่สุด เรียก lib ของตัวเอง (ตาม pattern Part 17/33)
fn main() {
    library_ui::mount();
}
```

`Trunk.toml` และ `index.html` เรียบง่ายที่สุดเท่าที่จะทำได้ (ตามที่ Part 89 หัวข้อ 89.2 พิสูจน์ไว้แล้วว่าไม่ต้อง
มี `<link data-trunk rel="rust" />` เลยด้วยซ้ำถ้าโครงสร้างโฟลเดอร์ตรงตามธรรมเนียม):

```toml
# Trunk.toml
[build]
target = "index.html"
dist = "dist"
```

```html
<!doctype html>
<html lang="th">
  <head><meta charset="utf-8" /><title>ห้องสมุด</title></head>
  <body></body>
</html>
```

Build จริงด้วย `trunk build` (ผลลัพธ์จริงจากการรัน ตัดส่วน compile dependency ยาว ๆ ออก):

```
$ trunk build
2026-09-27T03:19:15.857434Z  INFO Starting trunk 0.21.14
2026-09-27T03:19:15.857763Z  INFO starting build
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 7.91s
2026-09-27T03:19:24.797224Z  INFO applying new distribution
2026-09-27T03:19:24.797546Z  INFO success
```

`trunk` สร้างโฟลเดอร์ `dist/` ที่มี `index.html` (แก้ไขให้เองอัตโนมัติ แทรก `<script type="module">` ที่ import
ไฟล์ WASM/JS ที่ hash ชื่อกันแคชเก่าเข้ามา — ตรงกับที่ Part 89 หัวข้อ 89.2 อธิบายกลไกนี้ไว้แล้ว) พร้อมไฟล์ `.js`
และ `.wasm` — โฟลเดอร์นี้คือ `STATIC_DIR` ที่ `full_router()` ในหัวข้อ 95.2 เอาไปเสิร์ฟ

#### รัน backend + frontend เป็นระบบเดียว แล้วยิง Playwright จริง

Backend ถูก build แบบ debug (`cargo run`) เสิร์ฟทั้ง API และไฟล์ static ของ frontend ที่ `trunk build` แล้ว จาก
origin เดียวกัน (`http://127.0.0.1:8123`) เชื่อมกับ PostgreSQL จริงที่ seed ข้อมูลไว้แล้วตามหัวข้อ 95.5:

```javascript
// e2e/e2e.spec.mjs
import { chromium } from "playwright";
import assert from "node:assert/strict";

const BASE_URL = process.env.BASE_URL ?? "http://127.0.0.1:8123";
const EXECUTABLE_PATH = process.env.PW_CHROMIUM ?? "/opt/pw-browsers/chromium-1194/chrome-linux/chrome";

async function run() {
  const browser = await chromium.launch({ executablePath: EXECUTABLE_PATH, headless: true });
  const page = await browser.newPage();

  try {
    console.log("[1] เปิดหน้ารายการหนังสือ...");
    await page.goto(BASE_URL, { waitUntil: "load" });

    // รอ selector ที่ระบุเจาะจง (ไม่ใช่ fixed sleep) — รอจน WASM โหลด+hydrate เสร็จและ fetch
    // ข้อมูลจาก backend จริงเสร็จแล้วเท่านั้น
    await page.waitForSelector('[data-testid="book-list"]', { timeout: 10_000 });

    const titles = await page.$$eval('[data-testid="book-card"] .book-title', (els) =>
      els.map((el) => el.textContent?.trim())
    );
    console.log("    หนังสือที่เห็นในหน้ารายการ:", titles);
    assert.ok(titles.includes("The Rust Programming Language"));
    assert.equal(titles.length, 3, `คาดว่าจะมีหนังสือ 3 เล่ม แต่เห็น ${titles.length} เล่ม`);

    console.log("[2] คลิกเข้าไปดูรายละเอียดเล่มแรก...");
    await page.click('[data-testid="book-card"] a');
    await page.waitForSelector('[data-testid="book-detail"]', { timeout: 10_000 });

    const heading = await page.textContent("h1");
    const status = await page.textContent('[data-testid="book-status"]');
    console.log("    หัวข้อหน้ารายละเอียด:", heading);
    console.log("    สถานะที่แสดง:", status);
    assert.equal(heading?.trim(), "The Rust Programming Language");
    assert.ok(status?.includes("พร้อมให้ยืม"));

    console.log("ผ่านทุกขั้นตอน: e2e test สำเร็จ");
  } finally {
    await browser.close();
  }
}

run().catch((err) => {
  console.error("e2e test FAILED:", err);
  process.exit(1);
});
```

ผลลัพธ์จริงจากการรัน (backend server จริงกำลังรันอยู่ที่ `127.0.0.1:8123`, เชื่อมกับ PostgreSQL จริงที่มีข้อมูล
fixture จากหัวข้อ 95.5):

```
$ node e2e.spec.mjs
[1] เปิดหน้ารายการหนังสือ...
    หนังสือที่เห็นในหน้ารายการ: [
  'The Rust Programming Language',
  'Programming Rust',
  'Zero To Production In Rust'
]
[2] คลิกเข้าไปดูรายละเอียดเล่มแรก...
    หัวข้อหน้ารายละเอียด: The Rust Programming Language
    สถานะที่แสดง: พร้อมให้ยืม
ผ่านทุกขั้นตอน: e2e test สำเร็จ
```

นี่คือหลักฐานจริงว่า **ทุกชั้นของระบบทำงานร่วมกันได้จริง**: Rust backend คุย SQL จริงกับ PostgreSQL จริง → ส่ง
JSON กลับผ่าน HTTP จริง → Leptos WASM bundle ที่ build จริงด้วย `trunk` โหลดใน Chromium จริง → เรียก
`fetch()` ไปที่ backend จริง → render DOM จริงจากข้อมูลนั้น → Playwright อ่าน DOM จริงนั้นออกมาตรวจสอบ ไม่มี
จุดใดในเส้นทางนี้ที่ใช้ mock/stub เลยแม้แต่จุดเดียว — นี่คือความมั่นใจระดับสูงสุดที่ e2e test ให้ได้ ซึ่งเป็น
สิ่งที่ unit test และ integration test (แม้จะรวมกันหลักร้อยตัว) ไม่สามารถพิสูจน์ได้เลยด้วยตัวเอง

### 95.9 Flaky E2E Test: วินิจฉัยและแก้ไขจริง ไม่ใช่ Retry มั่ว ๆ

e2e test ในหัวข้อ 95.8 ใช้ `page.waitForSelector(...)` รอเงื่อนไขที่เจาะจง — แต่ถ้าเขียนแบบ **fixed sleep**
(รอเวลาคงที่แล้วเดาว่าน่าจะพร้อม) จะเกิดอะไรขึ้น? มาพิสูจน์ให้เห็นจริง ไม่ใช่แค่พูดลอย ๆ ว่า "fixed sleep ไม่ดี"

#### จำลองสถานการณ์จริง: เครือข่ายที่ความเร็วไม่คงที่

ในโลกจริง ผู้ใช้แต่ละคนดาวน์โหลด WASM bundle ด้วยความเร็วเน็ตต่างกัน — เพื่อจำลองสถานการณ์นี้อย่างสมจริง (ไม่ใช่
ใส่ `Math.random()` มาหลอกให้ fail ตรง ๆ) ให้ Playwright หน่วงเวลาการตอบไฟล์ `.wasm` แบบสุ่ม 0-180ms ต่อ request
(จำลองความหน่วงของเครือข่ายที่แปรผันได้จริง):

```javascript
// flaky_demo.mjs — ใช้ fixed sleep 50ms แทนการรอเงื่อนไขจริง
const context = await browser.newContext({ bypassCSP: true });
await context.route("**/*.wasm", async (route) => {
  const delayMs = Math.floor(Math.random() * 180);
  await new Promise((r) => setTimeout(r, delayMs));
  await route.continue();
});
const page = await context.newPage();

await page.goto(BASE_URL, { waitUntil: "load" });
await page.waitForTimeout(50); // <-- fixed sleep สั้นเกินไป: ต้นเหตุของความ flaky
const list = await page.$('[data-testid="book-list"]');
assert.ok(list, "ต้องเจอ book-list หลัง sleep 50ms");
```

`context.route("**/*.wasm", ...)` คือ API ของ Playwright ที่ดัก (intercept) ทุก network request ที่ตรงกับ
pattern ที่ระบุ ก่อนปล่อยให้ไปต่อจริงด้วย `route.continue()` — ในที่นี้ใช้มันแทรก delay สุ่มก่อนปล่อย response
ของไฟล์ `.wasm` ไปให้เบราว์เซอร์ วิธีนี้ **ดีกว่าการรัน e2e test ซ้ำ ๆ เฉย ๆ แล้วหวังว่าจะเจอ flaky โดยบังเอิญ**
มาก เพราะมันจำลองสภาพเครือข่ายที่ผันแปรได้อย่างควบคุมได้และทำซ้ำได้ (reproducible) — ทีมจริงที่ต้องการพิสูจน์ว่า
e2e suite ของตัวเอง "ทนทานต่อความช้าของเครือข่าย" ควรใช้เทคนิคนี้ตั้งใจ ไม่ใช่รอให้เจอ flaky จริงในโปรดักชันแล้ว
ค่อยแก้ทีหลัง

รันโค้ด**เดียวกันเป๊ะ ไม่แก้อะไรเลย** ซ้ำ 10 ครั้งติดกัน ผลลัพธ์จริงที่ได้:

```
=== run 1 ===
FAIL: ต้องเจอ book-list หลัง sleep 50ms
=== run 2 ===
PASS
=== run 3 ===
FAIL: ต้องเจอ book-list หลัง sleep 50ms
=== run 4 ===
PASS
=== run 5 ===
PASS
=== run 6 ===
PASS
=== run 7 ===
FAIL: ต้องเจอ book-list หลัง sleep 50ms
=== run 8 ===
FAIL: ต้องเจอ book-list หลัง sleep 50ms
=== run 9 ===
PASS
=== run 10 ===
FAIL: ต้องเจอ book-list หลัง sleep 50ms
PASS=5 FAIL=5
```

**5 PASS / 5 FAIL จากโค้ดตัวเดียวกันเป๊ะ ไม่มีการเปลี่ยนแปลงระหว่างรอบเลย** — นี่คือนิยามของ flaky test แบบ
เป็นรูปธรรมที่สุด: ผลลัพธ์ไม่คงที่ทั้งที่ input/โค้ด/ระบบเหมือนกันทุกประการ สาเหตุคือ delay สุ่มของไฟล์ `.wasm`
บางรอบน้อยกว่า 50ms (WASM พร้อมก่อน sleep จบ → PASS) บางรอบมากกว่า 50ms (sleep จบก่อน WASM พร้อม → element ยัง
ไม่ถูก mount → FAIL)

#### แก้ที่ต้นเหตุ: รอเงื่อนไขจริง ไม่ใช่รอเวลา

```javascript
// fixed_demo.mjs — เหมือน flaky_demo.mjs เป๊ะ (network delay สุ่มแบบเดียวกัน) ต่างกันแค่บรรทัดเดียว
await page.goto(BASE_URL, { waitUntil: "load" });
await page.waitForSelector('[data-testid="book-list"]', { timeout: 10_000 }); // <-- แก้จุดเดียว
const list = await page.$('[data-testid="book-list"]');
assert.ok(list, "ต้องเจอ book-list");
```

รันซ้ำ 10 ครั้งเหมือนกัน (ด้วย network delay สุ่มแบบเดียวกันเป๊ะ — โค้ด route delay ไม่เปลี่ยน):

```
=== run 1 === PASS
=== run 2 === PASS
=== run 3 === PASS
=== run 4 === PASS
=== run 5 === PASS
=== run 6 === PASS
=== run 7 === PASS
=== run 8 === PASS
=== run 9 === PASS
=== run 10 === PASS
PASS=10 FAIL=0
```

**10/10 PASS** — แก้ด้วยการเปลี่ยนบรรทัดเดียว (`waitForTimeout(50)` → `waitForSelector(...)`) แม้ network
delay ยังสุ่มแบบเดียวกันเป๊ะ (สูงสุดถึง 180ms ซึ่งมากกว่า sleep เดิมตั้ง 3.6 เท่า) เพราะ `waitForSelector` รอ
**"เงื่อนไขที่ต้องการจริง"** (element ปรากฏใน DOM) ไม่ใช่ **"เวลาที่เดาไว้"** — ไม่ว่า WASM จะใช้เวลาโหลดนานแค่
ไหน (ภายใน timeout 10 วินาทีที่กำหนดไว้) test จะรอจนกว่าเงื่อนไขเป็นจริงแล้วดำเนินต่อทันที ไม่เร็วไม่ช้าไปกว่า
ที่ระบบต้องการจริง

#### หลักการวินิจฉัย flaky test: "flaky ไม่ใช่ต้นเหตุ มันเป็นอาการ"

ผลการทดลองข้างบนสอนบทเรียนที่สำคัญกว่าแค่ "ใช้ `waitForSelector` แทน sleep": **flaky เป็นแค่คำอธิบายอาการ
(symptom) ไม่ใช่การวินิจฉัยต้นเหตุ (root cause)** — เมื่อเทสไหน flaky ต้องแยกให้ชัดก่อนว่าอยู่ในกรณีไหน:

1. **Timing assumption ผิด (แบบที่สาธิตข้างบน)**: โค้ดเทสเดาเวลาที่ "น่าจะพอ" แทนรอเงื่อนไขจริง — แก้ด้วยการ
   เปลี่ยนไปรอเงื่อนไขที่เจาะจง (`waitForSelector`, `waitForResponse`, `waitForFunction` ของ Playwright) เสมอ
   ไม่ใช่แค่ "เพิ่มเวลา sleep ให้นานขึ้น" (เพิ่ม sleep จาก 50ms เป็น 500ms อาจลดโอกาส fail ลงได้ แต่ไม่ได้กำจัด
   ต้นเหตุ — ยังมีโอกาส fail อยู่เสมอถ้าเครื่องช้าลงกว่าที่คาด แถมทำให้ suite รันช้าลงทุกครั้งโดยไม่จำเป็นเมื่อ
   ระบบพร้อมเร็วกว่านั้น)
2. **Infrastructure ไม่พร้อมจริง (แบบที่หัวข้อ 95.6-95.7 เจอ)**: chromedriver/Chromium เวอร์ชันไม่ตรงกัน — นี่
   **ไม่ใช่บั๊กในโค้ดเทสหรือโค้ดแอปเลย** การ retry ซ้ำ ๆ ไม่ช่วยอะไร (จะ fail ด้วยสาเหตุเดียวกันทุกครั้ง 100%
   ไม่ใช่ intermittent) วิธีแก้ที่ถูกคือแก้ที่ระดับ infrastructure (sync เวอร์ชัน driver/browser, หรือเปลี่ยนไป
   ใช้เครื่องมือที่ไม่มี dependency นี้อย่าง Playwright)
3. **บั๊กจริงในแอปที่เกิดเฉพาะบางเงื่อนไข** (เช่น race condition จริงในโค้ด backend ที่ Part 71 พูดถึง lock
   contention) — retry อาจทำให้ "ดูเหมือนผ่าน" ได้บางครั้ง แต่กำลังซ่อนบั๊กจริงไว้ ไม่ได้แก้อะไร

**กฎการตัดสินใจที่ใช้ได้จริง**: เมื่อเทส flaky ให้ถามก่อนว่า **"ถ้ารันซ้ำเทสเดิมสิบครั้งติดกันบนเครื่องเดิม โดย
ไม่แก้อะไรเลย จะได้ผลลัพธ์เหมือนเดิมทุกครั้งไหม"** — ถ้า**ไม่เหมือนเดิม** (เหมือนการทดลองข้างบนที่ได้ 5 PASS/5
FAIL) แสดงว่าเป็นกรณี 1 (timing assumption) เพราะมี randomness แท้ ๆ อยู่ในระบบ (เช่น network timing) ต้องแก้ที่
การรอเงื่อนไข — ถ้า**เหมือนเดิมทุกครั้ง (fail 100% ด้วยข้อความเดิม)** แสดงว่าเป็นกรณี 2 (infrastructure) ต้องแก้
ที่ environment ไม่ใช่โค้ดเทส — และถ้าอยาก retry ให้ retry **เฉพาะกรณีที่ 2 เท่านั้น** (เช่น network blip ชั่วคราว
ระหว่างดาวน์โหลด driver) ไม่ใช่ retry มั่ว ๆ กับทุกเทสที่ fail โดยไม่วินิจฉัยก่อน เพราะ retry แบบไม่เลือกจะซ่อน
ทั้งบั๊กจริง (กรณี 3) และปัญหา timing (กรณี 1) ไว้ ทำให้ CI "ดูเขียว" ทั้งที่มีปัญหาจริงซ่อนอยู่ — CI ที่ต้อง
retry เพื่อให้ผ่านคือ CI ที่ไม่มีใครเชื่อผลลัพธ์ของมันอีกต่อไป (สอดคล้องกับหลักการที่ Part 32 หัวข้อ 32.4 พูดถึง
ไว้แล้วเรื่องเทสที่แข่ง shared state กัน: "fail รอบนี้เพราะบั๊กจริง หรือเพราะ flaky กันแน่?" — คำถามเดียวกันนี้
ยิ่งสำคัญกว่าเดิมมากในชั้น e2e ที่ค่าใช้จ่ายต่อการ debug สูงกว่า unit test มาก)

### 95.10 CI Pipeline สำหรับ Full-Stack Rust: ภาพรวมก่อน Part 97

บทนี้ไม่ได้ลงรายละเอียดการตั้งค่า CI จริง (นั่นคือหน้าที่ของ **Part 97** ที่จะสอน CI/CD เต็มรูปแบบ) แต่ควรรู้
ไว้ล่วงหน้าว่า pipeline ที่จะรันทุกชั้นของเทสในบทนี้ให้ครบต้อง orchestrate อะไรบ้าง เพื่อไม่ให้ตกใจตอนไปถึง
Part 97:

1. **ชั้น unit + integration (backend)**: ต้องมี PostgreSQL รันอยู่ก่อน `cargo test` (ไม่ว่าจะเป็น service
   container ของ CI provider หรือ Docker container ที่ pipeline สั่ง start เอง — Part 96 ที่กำลังจะเริ่มสอน
   Docker จะเกี่ยวข้องตรงนี้) พร้อม `DATABASE_URL` ที่ชี้ไปถูกที่ และสิทธิ์ `CREATEDB` สำหรับ `#[sqlx::test]`
2. **ชั้น WASM/component test**: ต้องมี Rust toolchain + `wasm32-unknown-unknown` target + `wasm-pack` ติดตั้ง
   ไว้ ถ้าจะรัน `--headless` ต้องมี browser + driver ที่ **sync เวอร์ชันกันจริง** (บทเรียนตรงจากหัวข้อ 95.6-95.7)
   — วิธีที่ปลอดภัยที่สุดคือใช้ base image ที่ผู้ดูแลอัปเดตคู่กันเสมอ หรือ pin เวอร์ชันทั้งสองไว้ตรงกันเอง ไม่พึ่ง
   "เวอร์ชันล่าสุด" ของทั้งสองอย่างแยกกัน
3. **ชั้น e2e**: ต้อง build ทั้ง backend (`cargo build`) และ frontend (`trunk build`) ให้เสร็จก่อน, seed
   ฐานข้อมูล fixture (หัวข้อ 95.5), **สั่ง backend server ขึ้นมาจริงแล้วรอให้พร้อมรับ request** (health check
   endpoint อย่าง `/api/healthz` ที่เห็นในหัวข้อ 95.4 มีประโยชน์ตรงนี้พอดี — pipeline ควร poll endpoint นี้จนกว่า
   จะตอบ 200 ก่อนเริ่มยิง Playwright ไม่ใช่ sleep คงที่แบบที่หัวข้อ 95.9 พิสูจน์แล้วว่าไม่น่าเชื่อถือ) แล้วค่อยรัน
   Playwright จริง สุดท้ายต้อง **หยุด backend server และล้าง fixture data ทิ้ง** ไม่ให้ค้างข้าม CI run
4. **ลำดับที่สมเหตุสมผล**: รันจากฐานของพีระมิดขึ้นไปยอด (unit → integration → WASM/component → e2e) เพื่อให้
   pipeline fail เร็วที่สุดเมื่อมีบั๊กระดับต่ำ (fail ใน unit test ใช้เวลาไม่กี่วินาที) ก่อนจะเสียเวลาหลักนาที
   ไปกับการตั้งเบราว์เซอร์จริงสำหรับ e2e ที่จะ fail ด้วยสาเหตุเดียวกันอยู่ดี — นี่คือการนำหลักการ "ฐานกว้าง
   ยอดแคบ" ของหัวข้อ 95.1 มาใช้จริงในการออกแบบลำดับขั้นของ pipeline เอง ไม่ใช่แค่สัดส่วนจำนวนเทส

รายละเอียดเชิงปฏิบัติทั้งหมดของการเขียน pipeline จริง (YAML, caching dependency, matrix testing ข้าม OS/เวอร์ชัน,
secret management สำหรับ `DATABASE_URL` ใน CI) จะอยู่ใน **Part 97** — บทนี้ให้แค่ภาพรวมว่า "ต้องมีอะไรพร้อม
ก่อนถึงจะรันเทสแต่ละชั้นได้" เพื่อให้เห็นภาพรวมก่อนไปเจาะรายละเอียดจริง

ตัวอย่าง idiom การรอ health check ที่ข้อ 3 พูดถึง (bash แบบง่ายที่สุด ใช้แนวคิดเดียวกับ `waitForSelector` ของ
หัวข้อ 95.9 — รอเงื่อนไขจริง ไม่ใช่ sleep คงที่ — เพียงแต่ระดับ process/network ไม่ใช่ระดับ DOM):

```bash
# รอจนกว่า backend server จะตอบ 200 จริง ก่อนเริ่มยิง e2e test (แทน sleep คงที่)
for i in $(seq 1 30); do
  if curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8123/api/healthz | grep -q 200; then
    echo "server พร้อมแล้วหลังพยายาม ${i} ครั้ง"
    break
  fi
  sleep 1
done
```

สังเกตว่า loop นี้ก็เป็น "explicit wait" แบบเดียวกับหลักการในหัวข้อ 95.9 เป๊ะ (poll เงื่อนไขซ้ำ ๆ จนเป็นจริง
แทนการเดาเวลาคงที่ครั้งเดียว) — หลักการ "รอเงื่อนไขจริง ไม่ใช่รอเวลา" ใช้ได้กับทุกระดับของระบบ ไม่ใช่แค่ DOM
element ในเบราว์เซอร์เท่านั้น แต่ใช้ได้ตั้งแต่ระดับ process readiness ไปจนถึงระดับ container orchestration ที่
Part 96-97 จะพูดถึงต่อ

### 95.11 Capstone Test Suite ฉบับสมบูรณ์: รวมทุกชั้นเข้าด้วยกัน

สรุปโครงสร้าง test suite เต็มรูปแบบของแอปตัวแทน (สแตนอินสำหรับ capstone Part 92-94) ที่บทนี้ตรวจสอบจริงทั้งหมด:

```
library_capstone/
├── backend/
│   ├── src/handlers.rs
│   │   └── #[cfg(test)] mod tests { ... }     # 5 unit test (oneshot + in-memory fake) — 95.3
│   └── tests/integration_test.rs               # 3 integration test (#[sqlx::test] + Postgres จริง) — 95.4
├── frontend/
│   ├── tests/web.rs                             # 5 wasm-pack test --node (pure logic) — 95.6
│   ├── tests/web_browser.rs                     # เทสเดียวกัน สำหรับ --headless --chrome — 95.6
│   └── tests/dom.rs                             # mount-DOM component test — 95.7
├── ssr_render_demo/ (แยก crate, feature ssr)
│   └── src/lib.rs #[cfg(test)] mod tests { ... } # 2 native test (render-to-string) — 95.7
└── e2e/
    ├── seed.sql                                  # fixture สำหรับทั้งระบบ — 95.5
    ├── e2e.spec.mjs                               # 1 e2e test เต็ม user flow — 95.8
    ├── flaky_demo.mjs / fixed_demo.mjs           # สาธิต flaky-then-fixed — 95.9
    └── package.json
```

**สรุปผลการรันจริงทั้งหมดในบทนี้** (ทุกตัวเลขคือผลลัพธ์จริงจากการ compile+run ไม่ใช่ตัวเลขสมมติ):

| ชั้น | เครื่องมือ | จำนวนเทส | ผลลัพธ์ | เวลารวม |
|---|---|---|---|---|
| Unit (backend) | `cargo test` + `oneshot` | 5 | 5 passed | 0.00s |
| Integration (backend) | `cargo test` + `#[sqlx::test]` | 3 | 3 passed | 0.31s |
| WASM logic (`--node`) | `wasm-pack test` | 5 | 5 passed | 0.02s |
| Component (render-to-string) | `cargo test` (native, ssr feature) | 2 | 2 passed | 0.00s |
| E2E (user flow เต็ม) | Playwright + Chromium จริง | 1 | ผ่าน (ยืนยันด้วย assertion จริงบน DOM) | ~2-3s |
| Flaky demo (fixed sleep) | Playwright, รันซ้ำ 10 ครั้ง | 10 รอบ | 5 PASS / 5 FAIL (flaky จริง) | - |
| Flaky demo (แก้แล้ว) | Playwright, รันซ้ำ 10 ครั้ง | 10 รอบ | 10 PASS / 0 FAIL | - |

ตัวเลขในตารางนี้พิสูจน์ตารางเชิงทฤษฎีในหัวข้อ 95.1 ได้ตรงเป๊ะ: ยิ่งขึ้นชั้นสูง เวลารันต่อเทสยิ่งมากขึ้นเป็นลำดับ
ขนาด (0.00s → 0.31s → วินาที) และความเสี่ยงต่อความไม่แน่นอน (flaky) ก็เพิ่มขึ้นตามไปด้วย — ยืนยันว่ารูปสามเหลี่ยม
"ฐานกว้าง ยอดแคบ" ไม่ใช่แค่คำแนะนำเชิงทฤษฎี แต่คือข้อเท็จจริงเชิงวิศวกรรมที่วัดได้จริงในระบบจริง

**สิ่งที่ตรวจสอบไม่ได้จริงในสภาพแวดล้อมที่เขียนบทนี้ (ต้องรายงานไว้ตรง ๆ ไม่ปั้นแต่งผลลัพธ์)**: `tests/web_browser.rs`
(pure logic เดียวกับ `tests/web.rs` แต่เปิด `run_in_browser`) และ `tests/dom.rs` (mount ลง DOM จริง) ทั้งสองไฟล์
**compile ผ่านสมบูรณ์สำหรับ `wasm32-unknown-unknown`** (ยืนยันด้วย `cargo check --target wasm32-unknown-unknown
--tests` ที่ผ่านจริง) แต่ **การรันจริงผ่าน `wasm-pack test --headless --chrome` ถูกบล็อกโดยปัญหาเวอร์ชัน
chromedriver/Chromium ที่รายงานไว้ในหัวข้อ 95.6-95.7** (chromedriver 147 vs Chromium 141 ไม่ตรงกัน และไม่มีทาง
ดาวน์โหลด driver ที่ตรงเวอร์ชันได้เพราะ network policy ของสภาพแวดล้อมนี้ปฏิเสธการเชื่อมต่อไปยัง Chrome for
Testing API) — นี่ไม่ใช่การเดาหรือข้ามขั้นตอนตรวจสอบ แต่เป็นผลการทดลองจริงที่เกิดขึ้นเมื่อพยายามรันจริง และเป็น
ตัวอย่างที่สอดคล้องกับเนื้อหาหัวข้อ 95.6-95.7-95.9 เองพอดี (infrastructure-level failure ที่ไม่เกี่ยวกับบั๊กใน
โค้ดเลย) ส่วนที่ตรวจสอบผ่านได้จริง 100% ในสภาพแวดล้อมนี้คือทุกแถวอื่นในตารางข้างบน รวมถึง e2e test เต็มรูปแบบผ่าน
Playwright ที่ไม่พึ่ง chromedriver เลย (ใช้ CDP โดยตรงตามที่อธิบายไว้ในหัวข้อ 95.8)

## กับดักที่พบบ่อย (Common Pitfalls)

1. **ลืมว่า `wasm_bindgen_test_configure!(run_in_browser)` เป็นตัวกำหนด target ของทั้งไฟล์ ไม่ใช่ auto-detect**
   — เขียนเทสที่ใช้ `web_sys`/DOM API ไว้ในไฟล์เดียวกับ pure-logic test โดยไม่ใส่ macro นี้ แล้วพยายามรันด้วย
   `--headless --chrome` จะได้ผลลัพธ์ที่งงมาก (บางเทสถูก skip เงียบ ๆ):

   ```
   This test suite is only configured to run in node.js, but we're only running
       browser tests so skipping. If you'd like to run the tests in a browser
       include this in your crate when testing:

           wasm_bindgen_test::wasm_bindgen_test_configure!(run_in_browser);
   ```

   วิธีแก้: แยกไฟล์เทสตามบริบทที่ต้องการรัน (`tests/web.rs` สำหรับ `--node`, `tests/dom.rs` สำหรับ
   `--headless`) ตามกฎที่ Part 33 สอนไว้ว่าแต่ละไฟล์ใน `tests/` คือ crate อิสระของตัวเองอยู่แล้ว — ใช้ประโยชน์
   จากกฎนี้แทนที่จะพยายามยัดทุกอย่างไว้ในไฟล์เดียว

2. **พึ่ง `wasm-pack test --headless --chrome` (หรือเครื่องมือ WebDriver-based อื่น) ใน CI โดยไม่ตรวจสอบว่า
   browser กับ driver sync เวอร์ชันกัน** — จะได้ error ที่ดูเหมือนไม่เกี่ยวกับโค้ดเลย:

   ```
   Error: failed to create a Chrome session: session not created: This version of
   ChromeDriver only supports Chrome version 147
   Current browser version is 141.0.7390.37 with binary path /opt/pw-browsers/chromium-1194/chrome-linux/chrome
   ```

   วิธีแก้: ใช้ base image ที่ผู้ดูแล pin เวอร์ชัน browser+driver คู่กันเสมอ, หรือย้ายไปใช้เครื่องมือที่คุยกับ
   เบราว์เซอร์ผ่าน CDP โดยตรงอย่าง Playwright (ที่ผูกเวอร์ชัน browser ไว้กับตัว npm package เอง ไม่ต้อง sync
   กับ driver แยกต่างหาก) สำหรับงานที่ต้องพึ่งเบราว์เซอร์จริงเป็นหลัก

3. **ใช้ fixed sleep (`waitForTimeout`, `thread::sleep`, `setTimeout`) แทนการรอเงื่อนไขเจาะจงใน e2e test** —
   เทสจะ flaky แบบที่พิสูจน์จริงในหัวข้อ 95.9 (5 PASS / 5 FAIL จากโค้ดเดียวกัน รันซ้ำ 10 ครั้ง) และการ "แก้" ด้วย
   การเพิ่มเวลา sleep ให้นานขึ้นไม่ได้กำจัดต้นเหตุ แค่ลดโอกาสเจอปัญหาลง (ยังมีเงื่อนไขที่ทำให้ fail ได้อยู่เสมอ)
   วิธีแก้ที่ถูกคือใช้ explicit wait ที่รอเงื่อนไขจริง (`waitForSelector`, `waitForResponse`,
   `waitForFunction`) เสมอ

4. **ทดสอบ handler ผ่าน `oneshot` แต่ลืมว่า `Router` ต้อง `.with_state(...)` ให้ตรง type กับที่ handler
   ต้องการก่อน** — ถ้าลืมเรียก `.with_state(state)` แล้วพยายามใช้ `Router` ที่ยัง generic อยู่กับ `oneshot`
   ตรง ๆ จะได้ compiler error ทำนอง (จำลองจาก error message จริงของ Axum เมื่อ state ยังไม่ตรง):

   ```
   error[E0277]: the trait bound `Router<AppState>: tower::Service<http::Request<Body>>` is not satisfied
      --> tests/integration_test.rs:20:44
      |
   20 |     let response = tower::ServiceExt::oneshot(app, request).await.unwrap();
      |                                                ^^^ the trait `tower::Service<http::Request<Body>>` is not implemented for `Router<AppState>`
      |
      = note: `Router<S>` implements `Service` only when `S = ()` — เรียก `.with_state(...)` ก่อนเพื่อให้ state ถูกฝังเข้าไปใน Router แล้ว (กลายเป็น Router<()>)
   ```

   วิธีแก้: เรียก `.with_state(state)` ให้ครบก่อนส่ง `Router` เข้า `oneshot` เสมอ (`api_router()` ในตัวอย่าง
   ของบทนี้เรียก `.with_state(state)` ไว้ในตัวมันเองแล้ว เพื่อไม่ให้ผู้เรียกใช้ลืมขั้นตอนนี้)

5. **Fixture ของ backend integration test (สร้าง/ลบทุกเทส) กับ fixture ของ e2e test (ต้องอยู่ถาวรตลอด suite)
   ใช้ pattern เดียวกันโดยไม่แยกให้ชัด** — ถ้าเผลอใช้ `#[sqlx::test]` (ที่สร้างฐานข้อมูลใหม่แล้วลบทิ้งทุกเทส) กับ
   e2e test ที่ backend server จริงต้องคุยกับฐานข้อมูลเดิมตลอดการรัน จะเจอปัญหา "ข้อมูลหายไปเฉย ๆ กลางทาง"
   เพราะแต่ละ `#[sqlx::test]` ทำงานอยู่ในฐานข้อมูลคนละลูกที่ไม่มีใครอื่นมองเห็น วิธีแก้: ใช้ SQL fixture ตรง ๆ
   ที่รันครั้งเดียวก่อนเริ่ม e2e suite (ตามหัวข้อ 95.5) แยกจาก pattern `#[sqlx::test]` ที่ใช้เฉพาะ integration
   test ระดับ handler

6. **เปิด backend server สำหรับ e2e test ด้วยคำสั่งที่ไม่ได้รันแบบ background job จริง แล้วเซสชันปิดตัวลงก่อน
   Playwright ทันเวลา** — อาการที่เจอคือ `net::ERR_CONNECTION_REFUSED` ทั้งที่โค้ด server ไม่มีบั๊กเลย:

   ```
   FAIL: page.goto: net::ERR_CONNECTION_REFUSED at http://127.0.0.1:8123/
   Call log:
     - navigating to "http://127.0.0.1:8123/", waiting until "load"
   ```

   วิธีแก้: ต้องแน่ใจว่า process ของ server ถูกปล่อยให้รันอยู่จริงในพื้นหลัง (background job ที่ shell/CI
   runner ไม่ปิดทิ้งเมื่อคำสั่งที่สั่งเปิดมันจบไป) แล้ว **poll health check endpoint จนกว่าจะตอบสำเร็จก่อนเริ่ม
   ยิง e2e test จริง** ไม่ใช่เดาว่า "รอสักพักก็คงพร้อมแล้ว"

7. **เขียน e2e script เป็น ES module (`import { chromium } from "playwright"`) แล้วรันตรง ๆ โดยไม่ได้ติดตั้ง
   `playwright` เป็น local dependency ของโฟลเดอร์นั้น** — ถ้า Playwright ถูกติดตั้งแบบ global (เช่นในสภาพแวดล้อม
   นี้ที่อยู่ที่ `/opt/node22/lib/node_modules/playwright`) การตั้ง `NODE_PATH` ชี้ไปที่โฟลเดอร์นั้น **ใช้ไม่ได้
   กับ ES module** (`import`) แม้จะใช้ได้กับ CommonJS (`require`) ก็ตาม จะได้ error จริงแบบนี้:

   ```
   Error [ERR_MODULE_NOT_FOUND]: Cannot find package 'playwright' imported from
   /path/to/e2e/e2e.spec.mjs
   Did you mean to import "playwright/index.js"?
   ```

   วิธีแก้ที่ใช้ได้จริง: สร้าง symlink `node_modules` ในโฟลเดอร์ของ e2e script ไปยังตำแหน่งที่ package ติดตั้ง
   อยู่จริง (`ln -s /path/to/global/node_modules node_modules`) เพราะ Node.js ESM resolver ค้นหา `node_modules`
   ตาม path จริงของไฟล์ที่ `import` เท่านั้น ไม่สนใจ `NODE_PATH` เลย (ต่างจาก CommonJS `require` ที่ยัง fallback
   ไปดู `NODE_PATH` ได้) — หรือถ้าเป็นโปรเจกต์จริงที่ไม่ใช่สภาพแวดล้อมทดลอง วิธีที่ควรทำคือติดตั้ง `playwright`
   เป็น local dependency ผ่าน `package.json`/`npm install` ตามปกติไปเลย ไม่ต้องพึ่ง global install

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เพิ่ม handler ใหม่ `DELETE /books/{id}` ที่ลบหนังสือได้เฉพาะเมื่อ `available == true` (ห้ามลบ
   หนังสือที่มีคนยืมอยู่ — ต้องตอบ `AppError::Conflict`) เขียน unit test ผ่าน `oneshot` + `InMemoryBookRepository`
   ให้ครบทั้ง happy path (ลบสำเร็จ, ตอบ 204 No Content) และ error path (ลบหนังสือที่ถูกยืมอยู่, ตอบ 409)
   - Hint: เพิ่ม method `delete` เข้า trait `BookRepository` ทั้งสอง implementation (`PgBookRepository` ใช้
     `DELETE FROM books WHERE id = $1 AND available = true` แล้วเช็ค `rows_affected()`, `InMemoryBookRepository`
     เช็คแล้ว `.remove()` จาก `HashMap`)

2. **(กลาง)** เขียน integration test เพิ่มสำหรับ handler จากข้อ 1 ด้วย `#[sqlx::test]` ที่พิสูจน์ว่า **หลัง
   ลบสำเร็จแล้ว query `SELECT` หาแถวนั้นด้วย id เดิมต้องได้ `fetch_optional` เป็น `None`** (ไม่ใช่แค่เชื่อ
   response JSON ของ handler ฝ่ายเดียวแบบที่บทนี้เตือนไว้ในหัวข้อ 95.4)
   - Hint: ใช้ pattern เดียวกับ `borrow_full_round_trip_updates_row_in_real_postgres` ในหัวข้อ 95.4 — INSERT
     fixture ก่อน เรียก handler ผ่าน `oneshot`, แล้ว query ฐานข้อมูลอีกครั้งเพื่อยืนยัน

3. **(ยาก)** เพิ่ม component ใหม่ฝั่ง frontend ชื่อ `BorrowButton` ที่แสดงปุ่ม "ยืมหนังสือเล่มนี้" เมื่อ
   `available == true` และ disable ปุ่มพร้อมข้อความ "ถูกยืมไปแล้ว" เมื่อ `available == false` เขียนทั้ง
   component test แบบ render-to-string (ตรวจว่า attribute `disabled` ปรากฏถูกเงื่อนไข) **และ** เพิ่มขั้นตอนใน
   e2e test ของหัวข้อ 95.8 ให้คลิกปุ่มนี้จริงแล้วยืนยันว่าหน้าเปลี่ยนสถานะเป็น "ถูกยืมไปแล้ว" หลัง reload
   - Hint: `view! { <button disabled=!book.available>...</button> }` — ตรวจสอบว่า `.to_html()` ใส่
     attribute `disabled` เข้าไปจริงเมื่อค่าเป็น `true` (ต้องอ่าน HTML string ที่ได้จริงเพื่อยืนยัน ไม่ใช่เดา
     จาก syntax เฉย ๆ) ฝั่ง e2e ใช้ `page.click(...)` แล้ว `page.reload()` ก่อนตรวจ DOM ใหม่ (เพราะแอปตัวอย่างนี้
     เป็น CSR ล้วน ๆ ไม่มี optimistic update ในตัว state จนกว่าจะ fetch ใหม่)

4. **(ยาก/ประยุกต์ใช้งานจริง)** สร้างสถานการณ์ flaky test แบบใหม่ที่**ไม่เกี่ยวกับ WASM loading เลย** — เช่น
   จำลอง backend ที่ query ฐานข้อมูลช้าแบบสุ่ม (เพิ่ม `tokio::time::sleep(Duration::from_millis(rand))` สุ่ม
   0-300ms ใน `PgBookRepository::list` ชั่วคราวเพื่อทดสอบ) แล้วพิสูจน์ว่า e2e test ที่ใช้ `waitForSelector`
   รอ DOM element (แบบหัวข้อ 95.8) ยัง**ทนทาน**ต่อความช้าแบบนี้ได้ (เพราะรอผลลัพธ์จริง ไม่ใช่รอ WASM โหลดอย่าง
   เดียว) ในขณะที่ถ้าเปลี่ยนไปใช้ fixed sleep แทนจะพังแบบเดียวกับหัวข้อ 95.9 อีกครั้ง เขียนรายงานสั้น ๆ (ไม่ต้อง
   ยาว) สรุปว่า `waitForSelector` "ทนทานต่อความช้าที่มาจากจุดไหนในระบบก็ได้" ไม่ใช่แค่ทนทานต่อ "WASM โหลดช้า"
   อย่างเดียว
   - Hint: นี่คือ generalization ของบทเรียนในหัวข้อ 95.9 — เงื่อนไขที่ทำให้ fixed sleep พังไม่ได้จำกัดอยู่แค่
     เวลาโหลด WASM เท่านั้น มันคือ **ความช้าที่แปรผันได้จากจุดไหนก็ได้ในระบบทั้งชุด** (network, WASM, database,
     server processing) — explicit wait แก้ปัญหานี้ได้ทุกจุดพร้อมกันเพราะมันรอ "ผลลัพธ์สุดท้าย" ไม่ใช่รอ "ขั้นตอน
     ใดขั้นตอนหนึ่งที่เดาเวลาไว้"

## สรุป

บทนี้ปิดโมดูล Testing ของแอป full-stack Rust ด้วยการต่อยอด testing pyramid จากสามชั้น (Part 32-33) ให้ครบ
สี่ชั้นจริง: **unit test** ผ่าน `oneshot` + trait-based mocking ที่ทดสอบ handler แยกส่วนได้เร็วระดับ
มิลลิวินาทีโดยไม่ต้องมี PostgreSQL, **integration test** ผ่าน `#[sqlx::test]` ที่พิสูจน์ request→handler→SQL
query→response ครบวงจรกับฐานข้อมูลจริง, **WASM/component test** ผ่าน `wasm-pack test` ทั้งสองบริบท (`--node`
สำหรับ pure logic ที่ยืนยันได้จริง 100% กับ `--headless --chrome` ที่ต้องพึ่ง browser automation infrastructure
ที่ซับซ้อนกว่าและเปราะบางกว่า — พิสูจน์จริงในสภาพแวดล้อมที่เขียนบทนี้ว่าปัญหาเวอร์ชัน driver/browser ไม่ sync
กันทำให้เส้นทางนี้ล้มเหลวได้โดยไม่เกี่ยวกับบั๊กเลย), และ **e2e test** ผ่าน Playwright ที่ขับเบราว์เซอร์จริงผ่าน
ทั้งระบบจริง (backend+DB+frontend WASM) ให้ความมั่นใจสูงสุดที่ชั้นล่างให้ไม่ได้ พร้อมพิสูจน์ด้วยการทดลองจริง
(5 PASS/5 FAIL จาก fixed sleep เทียบกับ 10/10 PASS จาก explicit wait) ว่าทำไมการวินิจฉัย flaky test ให้ถูกจุด
(timing assumption ผิด vs. infrastructure ไม่พร้อม vs. บั๊กจริง) สำคัญกว่าการ retry มั่ว ๆ เสมอ

**และนี่คือจุดปิดของ Module 5: Full-Stack และ WebAssembly (Part 86-95) ทั้งโมดูล** — ควรมองย้อนกลับไปดูเส้นทาง
ทั้งหมดที่เดินทางมา เพราะแต่ละบทไม่ได้เป็นความรู้แยกส่วน แต่ประกอบกันเป็นทักษะเดียว:

- **Part 86-87** วางฐานว่า Rust compile เป็น WASM ได้จริงอย่างไร (`wasm32-unknown-unknown` target, `.wasm`
  binary) และคุยกับ JavaScript/DOM ได้อย่างไรผ่าน `wasm-bindgen` — นี่คือ "ภาษากลาง" ที่ทำให้ทุกอย่างในโมดูลนี้
  เป็นไปได้ตั้งแต่ต้น
- **Part 88-90** สอนสามเฟรมเวิร์กที่สร้างขึ้นบนฐานนั้น — Yew (virtual DOM diffing แบบที่ frontend framework
  อื่นคุ้นเคย), Leptos (fine-grained reactivity ผ่าน signal + server function ที่ตัด boundary
  client/server ออกไปเกือบหมด), Dioxus (VirtualDom ที่ตัด core ออกจาก renderer โดยเจตนา เพื่อ "write once,
  render anywhere" ข้ามเว็บ/desktop/mobile) — สามจุดยืนที่ต่างกันชัดเจนสำหรับปัญหาเดียวกัน
- **Part 91** (Server-Side Rendering) เติมส่วนที่ frontend framework ล้วน ๆ ทำไม่ได้ — render หน้าแรกเป็น HTML
  บน server ก่อนส่งลง client (แก้ปัญหา time-to-first-paint และ SEO ที่ CSR ล้วน ๆ ของ Part 88-90 ยังไม่ครอบคลุม)
  ซึ่งเป็นเหตุผลเดียวกันที่ทำให้หัวข้อ 95.7 ของบทนี้ใช้ประโยชน์จาก SSR capability นั้นมาทำ render-to-string
  component test ได้อย่างลื่นไหล — SSR ไม่ใช่แค่ feature สำหรับ production แต่เป็นฐานที่ทำให้การทดสอบง่ายขึ้นด้วย
- **Part 92-94** (Full-Stack Project สามภาค) นำทุกอย่างมาประกอบเป็นแอปจริงที่ deploy ได้ — ออกแบบ backend (92),
  สร้าง frontend (93), แล้ว integrate + deploy ทั้งระบบเป็นหนึ่งเดียว (94) โดยเลือก Leptos เพราะจุดแข็งเรื่อง
  full-stack ตรงกับโจทย์ capstone ที่สุด (ตามที่ Part 90 ระบุไว้) — บทนี้จำลองสถาปัตยกรรมแบบเดียวกันนั้นไว้เพื่อ
  ทดสอบ ในกรณีที่ Part 92-94 ยังไม่มีอยู่ในระบบไฟล์ขณะที่เขียนบทนี้ตามที่อธิบายไว้ในหัวข้อ 95.2
- **Part 95 (บทนี้)** ปิดท้ายด้วยคำถามที่สำคัญที่สุดสำหรับซอฟต์แวร์ที่จะใช้งานจริง: **"รู้ได้อย่างไรว่าทุกชิ้น
  ที่ประกอบกันมาทั้งโมดูลทำงานถูกต้อง และจะรู้ได้อย่างไรเมื่อมีอะไรพังในอนาคต"** — คำตอบคือ testing pyramid
  ที่ครบทั้งสี่ชั้นที่บทนี้สอน ไม่ใช่แค่ "เขียนโค้ดให้ compile ผ่าน" (ซึ่ง Rust compiler ช่วยได้มากอยู่แล้วตาม
  Part 32 หัวข้อ 32.1) แต่ต้องพิสูจน์ "ความถูกต้องเชิงตรรกะทางธุรกิจ" ที่ compiler มองไม่เห็นเลย ทั้งในระดับ
  ฟังก์ชันเดี่ยว ๆ, การต่อกับฐานข้อมูลจริง, การรันบน WASM จริง, และการทำงานร่วมกันของทั้งระบบผ่านเบราว์เซอร์จริง

ทักษะที่ได้จากโมดูลนี้ทั้งหมด — เขียน Rust ที่ compile เป็น WASM, เลือกเฟรมเวิร์ก frontend ที่เหมาะกับโจทย์,
ต่อ SSR/server function เข้ากับ backend จริง, deploy เป็นระบบเดียว, และทดสอบให้มั่นใจได้ทุกชั้น — คือทักษะ
full-stack Rust ที่ใช้งานได้จริงในระดับ production ครบวงจร ไม่ใช่แค่ตัวอย่าง "Hello World" ที่ใช้งานจริงไม่ได้

จากนี้ไปหลักสูตรจะเปลี่ยนโฟกัสจาก "จะสร้างแอปอย่างไร" ไปสู่ **"จะเอาแอปที่สร้างแล้วไปรันในโลกจริงอย่างไรให้เสถียร
และดูแลรักษาได้ในระยะยาว"** — **Module 6: Production, DevOps และระดับมืออาชีพ (Part 96-110)** เริ่มต้นด้วย
**Part 96: Docker และ Containerization สำหรับ Rust** ซึ่งเป็นก้าวแรกที่จำเป็นก่อนไปถึง Part 97 (CI/CD ที่บทนี้
เอ่ยถึงไว้ในหัวข้อ 95.10) — การ containerize ทั้ง backend server และขั้นตอน build frontend WASM ให้เป็น image
เดียวที่ deploy ได้ที่ไหนก็ได้อย่างสม่ำเสมอ คือพื้นฐานที่ทุกอย่างในโมดูลถัดไปจะต่อยอดจากมัน

---

**Part ก่อนหน้า:** [Full-Stack Project (3/3): Integration และ Deployment](part-094-fullstack-integration.md) | **Part ถัดไป:** [Docker และ Containerization สำหรับ Rust](part-096-docker-containerization.md)
