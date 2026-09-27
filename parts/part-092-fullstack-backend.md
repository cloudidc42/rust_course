# Part 92: Full-Stack Project (1/3): ออกแบบและสร้าง Backend

> โมดูล: โปรเจกต์ Capstone (Full-Stack) | ระดับ: มืออาชีพ | เวลาโดยประมาณ: 320 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- ออกแบบ **API contract ที่แน่นอนและไม่กำกวม** สำหรับระบบจริงระบบหนึ่ง (ห้องสมุด/ยืม-คืนหนังสือ) ก่อนเขียนโค้ด
  แม้แต่บรรทัดเดียว — เห็นว่าทำไม contract ที่ชัดเจน (method, path, auth requirement, request/response
  shape, status code ที่เป็นไปได้ทุกกรณี) คือสิ่งที่ทำให้ทีม frontend ทำงานคู่ขนานกับทีม backend ได้จริง
  โดยไม่ต้องรอให้อีกฝั่งเขียนเสร็จก่อน — Part 93 (frontend) และ Part 94 (deployment) ที่จะตามมาจะ**อ่าน
  contract จากบทนี้เพียงอย่างเดียว** ไม่มีการ "เดา" หรือ "ถามกลับ" ใดๆ ทั้งสิ้น
- จัดโครงสร้างโปรเจกต์ Axum ขนาดจริง (ไม่ใช่ `main.rs` ไฟล์เดียวอีกต่อไป) เป็นหลาย module
  (`routes/`, `models/`, `auth/`, `error.rs`, `state.rs`) ตามหลักการแบ่ง module จาก Part 16-17 — และอธิบาย
  ได้ว่าทำไมการแบ่งแบบนี้ถึง "scale" ได้เมื่อโปรเจกต์โตขึ้น ต่างจากการยัดทุกอย่างไว้ที่เดียว
- สร้างฐานข้อมูล PostgreSQL จริงพร้อม migration ที่มี constraint/index ถูกต้อง (`users`, `books`, `borrows`)
  ตามหลักการ SQLx migration จาก Part 71 และเขียน handler ทุกตัวด้วย `sqlx::query!`/`query_as!` ที่ตรวจสอบ
  SQL กับ schema จริงตอน compile time
- implement authentication ด้วย JWT (Part 74) + argon2 password hashing แบบสมบูรณ์ (register/login) และ
  authorization แบบ RBAC (Part 76) สำหรับ endpoint ที่ต้องเป็น admin เท่านั้น พร้อมพิสูจน์ทุกเส้นทางด้วย
  `curl` จริง
- แก้ปัญหา **race condition ของการยืมหนังสือพร้อมกันหลาย request** ด้วย transaction + atomic conditional
  update (ต่อยอด Part 70) — และ**พิสูจน์ด้วยการยิง 10 request พร้อมกันจริง**ว่าหนังสือที่มี 3 สำเนาจะมีคน
  ยืมสำเร็จได้แค่ 3 คนเท่านั้น ไม่มีทางที่ `available_copies` จะติดลบ
- เขียน error handling แบบรวมศูนย์ (`AppError` ตาม Part 66) ที่ครอบคลุมทุกสถานการณ์ที่ API นี้เป็นไปได้ และ
  ตัดสินใจเลือก HTTP status ที่ถูกต้อง (โดยเฉพาะการเลือกระหว่าง 403/404/409 ในสถานการณ์ที่กำกวม) ตามหลักการ
  จาก Part 76
- ผลิตเอกสาร OpenAPI ที่ตรงกับโค้ดจริง 100% ด้วย `utoipa` + เปิด Swagger UI ให้ทดสอบได้จากเบราว์เซอร์ (Part 85)
  และเขียน integration test suite จริงที่ครอบคลุม flow หลักทั้งหมด (Part 32/33/71) พร้อมผลการรันจริงทุกเทส

## ความรู้ที่ต้องมีมาก่อน

บทนี้เป็น **capstone project บทที่ 1 จาก 3 บท** ของโมดูล Full-Stack — มันไม่สอนเทคนิคใหม่ทีละชิ้นแบบบทอื่น ๆ
แต่เอาเทคนิคที่คุณเรียนมาแล้วทั้งหมดจาก Module 4 (Web Development) มา**ประกอบเข้าด้วยกันเป็นระบบจริงระบบเดียว**
ดังนั้นความรู้ที่ต้องมีมาก่อนจึงมีจำนวนมากกว่าปกติ:

- **Part 16-17 (Modules, Packages/Crates/Workspaces)**: บทนี้จัดโครงสร้างโปรเจกต์เป็นหลาย module
  (`mod routes; mod models; mod auth;` ฯลฯ) ตามหลักการที่ Part 16 สอนไว้ทุกประการ — ถ้ายังไม่แน่นเรื่อง
  `mod`/`pub mod`/`use crate::...` ควรทวนก่อน
- **Part 32-33 (Testing: Unit และ Integration)**: integration test ท้ายบท (หัวข้อ 92.9) ใช้ pattern
  `#[sqlx::test]` และเรียก `Router` ตรง ๆ ผ่าน `tower::ServiceExt::oneshot` แทนการเปิด TCP listener จริง —
  เป็นรูปแบบเดียวกับ integration test ที่ Part 33 สอนไว้ เพียงแต่ประยุกต์กับ Axum
- **Part 61-66 (HTTP Fundamentals ถึง Axum Error Handling)**: `AppError` ในบทนี้ต่อยอด `AppError` ที่
  Part 66 ออกแบบไว้ตรง ๆ (enum เดียว, `impl IntoResponse`, JSON shape `{"error": {"code", "message"}}`)
  เพียงเพิ่ม variant ใหม่และ `impl From<sqlx::Error>` — ถ้าจำ mechanism ของ Part 66 ไม่ชัด ควรกลับไปทวนก่อน
  เพราะบทนี้จะไม่อธิบาย mechanism พื้นฐานซ้ำ
- **Part 70-71 (SQLx: PostgreSQL, Queries และ Migrations)**: `PgPool`, `AppState`, transaction
  (`pool.begin()`/`tx.commit()`), `sqlx::query!`/`query_as!` ที่ตรวจสอบกับ schema จริงตอน compile,
  `QueryBuilder` สำหรับ filter แบบ dynamic, และการจัดการ migration ด้วย `sqlx-cli` — บทนี้ใช้ทั้งหมดนี้เป็น
  "ไวยากรณ์พื้นฐาน" โดยไม่อธิบายซ้ำ
- **Part 74 (JWT Authentication)**: `Claims`, `jsonwebtoken::encode`/`decode`, argon2 password hashing,
  custom extractor `FromRequestParts` สำหรับอ่าน `Authorization: Bearer <token>` — บทนี้ใช้ pattern เดียวกัน
  ทุกประการ เพียงย้ายจาก `HashMap` ในหน่วยความจำ (ที่ Part 74 ใช้เพื่อโฟกัสเรื่อง JWT ล้วน ๆ) มาเป็น `PgPool`
  จริงตามที่ Part 74 แบบฝึกหัดข้อ 2 บอกไว้ว่าให้ลองทำ
- **Part 76 (Authorization: RBAC)**: `AppError::Forbidden` (403), การตัดสินใจ 403 vs 404 สำหรับ resource ที่
  ไม่ใช่เจ้าของ, และ RBAC check ระดับ handler (`if !current_user.is_admin() { return Err(...) }`) — บทนี้ใช้
  ทั้งสองแนวคิดนี้ตรงในหัวข้อ borrow/return และ create book
- **Part 78 (REST API Design)**: pagination (`page`/`per_page`), การออกแบบ response shape ที่มี `data` +
  `meta`, และการเลือก HTTP status code ที่ถูกต้องตามความหมายมาตรฐาน
- **Part 85 (OpenAPI ด้วย utoipa)**: `#[derive(ToSchema)]`, `#[utoipa::path(...)]`, `#[derive(OpenApi)]`,
  `SwaggerUi` — บทนี้ใช้ทั้งหมดนี้ในโปรเจกต์ที่แบ่งหลาย module จริง (ต่างจาก Part 85 ที่สอนแบบไฟล์เดียวเพื่อ
  โฟกัสที่ utoipa ล้วน ๆ)
- **Part 65 (Axum Middleware)**: `CorsLayer` — บทนี้ต้องเปิด CORS ให้ frontend dev server (Part 93) เรียกได้
  จริง ไม่ใช่แค่ทำ demo ในเครื่องเดียว

## หมายเหตุเรื่องการตรวจสอบเนื้อหา (สำคัญ — อ่านก่อนเริ่ม)

**บทนี้ต่างจากบททั่วไปของหลักสูตร**: มันคือจุดเริ่มต้นของ capstone 3 บท (Part 92 → 93 → 94) ที่ Part 93 และ
94 จะเขียนโดย agent คนละคน **โดยอ่าน API contract จากบทนี้เพียงอย่างเดียว** เพื่อสร้าง frontend และเชื่อมระบบ
เข้าด้วยกันจริง — เพราะฉะนั้นความถูกต้องแม่นยำของบทนี้จึงสำคัญกว่าปกติมาก ทุกโค้ด ทุก `curl` transcript ทุกผล
การทดสอบในบทนี้ **มาจากการสร้างโปรเจกต์ Rust จริง รันจริงกับ PostgreSQL จริงบนเครื่อง แล้ว capture ผลลัพธ์มา
ทั้งหมด** ไม่มีการแต่งขึ้นเองแม้แต่จุดเดียว รวมถึง error message จริงตอน compile ที่เจอระหว่างพัฒนา

เวอร์ชัน crate ที่ใช้จริงทั้งบทนี้ (ตรงกับที่ Part 65/66/70/71/74/76/78/85 ใช้ทุกประการ เพื่อให้ทั้งหลักสูตร
สอดคล้องกัน):

```toml
[package]
name = "library_api"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = { version = "0.8.9", features = ["macros"] }
tokio = { version = "1.53.1", features = ["full"] }
tower = { version = "0.5.3", features = ["util"] }
tower-http = { version = "0.7.1", features = ["cors", "trace"] }
serde = { version = "1.0.229", features = ["derive"] }
serde_json = "1.0.151"
sqlx = { version = "0.9.0", features = ["runtime-tokio", "postgres", "macros", "chrono", "migrate"] }
thiserror = "2.0.21"
anyhow = "1.0.104"
tracing = "0.1.44"
tracing-subscriber = { version = "0.3.23", features = ["env-filter"] }
jsonwebtoken = { version = "11.1.0", features = ["rust_crypto"] }
argon2 = "0.6.0"
chrono = { version = "0.4.45", features = ["serde"] }
utoipa = { version = "6.0.0", features = ["axum_extras", "chrono"] }
utoipa-swagger-ui = { version = "10.0.1", features = ["axum", "vendored"] }
```

**สภาพแวดล้อมที่ทดสอบจริง**: PostgreSQL 16 รันอยู่ที่ `127.0.0.1:5432`, สร้างฐานข้อมูลชื่อ `library_api_dev`,
รัน migration จริงด้วย `sqlx-cli` 0.9.0, รันเซิร์ฟเวอร์จริงด้วย `cargo run`, ยิง `curl` จริงกว่า 30 คำสั่งไปยัง
เซิร์ฟเวอร์ที่รันอยู่จริงที่ `http://127.0.0.1:8092`, และรัน `cargo test` จริงกับ integration test 5 ตัวที่
ผ่านทั้งหมด — โปรเจกต์ scratch ที่ใช้ทดสอบถูกลบทิ้งหลังตรวจสอบเสร็จ (ไม่กระทบไฟล์ใด ๆ ในหลักสูตร) แต่**โค้ด
ทุกไฟล์ในบทนี้คือโค้ดตัวจริงที่ compile ผ่านและรันได้ 100%** — ผู้อ่านสามารถคัดลอกโค้ดทั้งหมดในบทนี้ไปสร้าง
โปรเจกต์เดียวกันขึ้นมาใหม่ได้ทันที

## เนื้อหา

### 92.1 ภาพรวม Capstone: ระบบห้องสมุด/ยืม-คืนหนังสือ

ตลอด Module 4-5 หลักสูตรนี้ใช้ระบบห้องสมุด/ระบบจองตั๋วเป็นตัวอย่างซ้ำ ๆ (Part 63 วางโครง REST API เบื้องต้น,
Part 66 ทำ error handling, Part 70-71 ต่อ PostgreSQL จริง, Part 74 ทำ auth, Part 76 ทำ RBAC, Part 78 ออกแบบ
REST ให้เป็นมาตรฐาน, Part 85 เขียนเอกสาร) — **บทนี้คือจุดที่เอาทุกอย่างมาประกอบเป็นระบบเดียวที่สมบูรณ์และรันได้
จริงตั้งแต่ต้นจนจบ** ไม่ใช่ตัวอย่างแยกส่วนอีกต่อไป

ก่อนเขียนโค้ดสักบรรทัด สิ่งสำคัญที่สุดคือ**กำหนดสโคปให้ชัดเจน** เพราะ Part 93 (frontend) และ Part 94
(deployment) จะสร้างต่อจากสิ่งที่บทนี้สร้างไว้ "ตามที่มันเป็น" — ถ้าสโคปกำกวม ทุกอย่างที่ตามมาจะกำกวมไปด้วย
สโคปของ capstone นี้ (ตัดสินใจไว้ชัดเจน ไม่ใช่ "แนวทาง" ที่ยังเปลี่ยนได้):

1. ผู้ใช้ (ยังไม่ login) เรียกดูรายการหนังสือและรายละเอียดหนังสือได้ — public, ไม่ต้อง auth
2. ผู้ใช้สมัครสมาชิกและ login ได้ — ได้ JWT access token กลับมา
3. ผู้ใช้ที่ login แล้ว (member) ยืมและคืนหนังสือได้ และดูรายการที่ตัวเองยืมค้างอยู่ได้
4. ผู้ใช้ที่มี role `admin` เท่านั้นที่สร้างหนังสือใหม่เข้าระบบได้
5. การยืมหนังสือต้อง**ปลอดภัยจาก race condition** — สำเนาหนังสือมีจำกัด ถ้ามีคนยืมพร้อมกันหลายคนตอนที่เหลือ
   สำเนาน้อย ต้องมีคนได้ยืมพอดีตามจำนวนสำเนาที่เหลือเท่านั้น ไม่มากไม่น้อยกว่านั้น

**สิ่งที่ตั้งใจ "ไม่รวม" ไว้ในบทนี้ (deferred ไปยัง Part 93/94 หรือนอกสโคป capstone นี้โดยตั้งใจ)**: การแก้ไข/
ลบหนังสือ (`PATCH`/`DELETE /books/{id}`), การจัดการผู้ใช้โดย admin, refresh token ที่ revoke ได้จริง (Part 74
sketch แนวคิดไว้แต่ไม่ implement เต็ม), rate limiting (Part 78 สอนแยกไว้แล้ว), การจอง/คิวรอ (waitlist) — ทั้งหมด
นี้เป็นหัวข้อดีสำหรับแบบฝึกหัดขยายผล (ดูหัวข้อแบบฝึกหัดท้ายบท) แต่**ไม่อยู่ในสัญญา (contract) ที่ Part 93/94
ต้องรองรับ** เพื่อไม่ให้สโคปของ capstone บวมจนควบคุมไม่ได้

#### API Contract — เอกสารที่สำคัญที่สุดของบทนี้

ตารางนี้คือ **สัญญา (contract) ที่แน่นอนที่สุดของทั้ง 3 บท capstone** — Part 93 (frontend) จะเรียก endpoint
เหล่านี้ตรงตามที่ระบุไว้นี้ทุกประการ ไม่มีการเปลี่ยนแปลง path, shape, หรือ status code ใด ๆ อีกในบทต่อ ๆ ไป
โดยไม่ได้แก้ไขบทนี้ก่อน:

| # | Method | Path | ต้อง Auth? | Request Body | Response Body (สำเร็จ) | Status สำเร็จ | Status ล้มเหลวที่เป็นไปได้ |
|---|--------|------|-----------|---------------|------------------------|---------------|------------------------------|
| 1 | GET | `/health` | ไม่ | - | `{"status":"ok"}` | 200 | - |
| 2 | POST | `/api/v1/auth/register` | ไม่ | `{"username":string, "email":string, "password":string}` | `UserResponse` | 201 | 400 (validation), 409 (username/email ซ้ำ) |
| 3 | POST | `/api/v1/auth/login` | ไม่ | `{"username":string, "password":string}` | `LoginResponse` | 200 | 401 (username/password ผิด) |
| 4 | GET | `/api/v1/me` | **ต้อง** | - | `UserResponse` | 200 | 401 |
| 5 | GET | `/api/v1/books` | ไม่ | query: `page`, `per_page`, `category`, `q`, `available_only` (ทุกตัว optional) | `BookListResponse` | 200 | - |
| 6 | GET | `/api/v1/books/{id}` | ไม่ | - | `BookResponse` | 200 | 404 |
| 7 | POST | `/api/v1/books` | **ต้อง + admin** | `CreateBookRequest` | `BookResponse` | 201 | 400, 401, 403 (ไม่ใช่ admin), 409 (isbn ซ้ำ) |
| 8 | POST | `/api/v1/books/{id}/borrow` | **ต้อง** | - | `BorrowResponse` | 201 | 401, 404 (ไม่มีหนังสือนี้), 409 (ไม่มีสำเนาว่าง หรือยืมค้างอยู่แล้ว) |
| 9 | POST | `/api/v1/books/{id}/return` | **ต้อง** | - | `BorrowResponse` | 200 | 401, 403 (ไม่ใช่ผู้ยืม), 404 (ไม่มีหนังสือนี้), 409 (ไม่มีการยืมค้างอยู่) |
| 10 | GET | `/api/v1/me/borrowed` | **ต้อง** | - | `BorrowListResponse` | 200 | 401 |

**Shape ของแต่ละ type ที่อ้างถึงในตาราง** (JSON field ทุกตัว, ชนิดข้อมูล, และความหมาย — Part 93 ใช้ตารางนี้
กำหนด type ฝั่ง frontend ได้ตรง ๆ):

```
UserResponse:
  id: number, username: string, email: string, role: "member" | "admin", created_at: string (ISO-8601)

LoginResponse:
  access_token: string, token_type: "Bearer", expires_in: number (วินาที = 3600), user: UserResponse

BookResponse:
  id: number, title: string, author: string, isbn: string, category: string,
  total_copies: number, available_copies: number, created_at: string (ISO-8601)

CreateBookRequest (request เท่านั้น):
  title: string, author: string, isbn: string (10 หรือ 13 หลัก), category: string, total_copies: number (> 0)

BookListResponse:
  data: BookResponse[], meta: { page: number, per_page: number, total_items: number, total_pages: number }

BorrowResponse:
  id: number, book_id: number, user_id: number,
  borrowed_at: string (ISO-8601), due_at: string (ISO-8601), returned_at: string | null

BorrowListResponse:
  data: BorrowResponse[]

Error shape (ทุก endpoint ที่ล้มเหลว ไม่ว่า status ไหน — ตรงกับ AppError ของ Part 66):
  { "error": { "code": string, "message": string, "fields"?: [{ "field": string, "message": string }] } }
  ("fields" ปรากฏเฉพาะตอน code === "VALIDATION_ERROR" เท่านั้น)
```

**การยืนยันตัวตน**: endpoint ที่ระบุ "ต้อง Auth" ต้องแนบ header `Authorization: Bearer <access_token>` ที่ได้
จาก endpoint #3 (login) — access token มีอายุ **3600 วินาที (1 ชั่วโมง)** นับจากตอนออก ไม่มี refresh token ใน
สโคปนี้ (ผู้ใช้ต้อง login ใหม่หลัง token หมดอายุ — ดูหัวข้อแบบฝึกหัดข้อ 3 ถ้าต้องการต่อยอดเรื่อง refresh token)

**ทำไม endpoint #1 และ #4 ไม่ได้อยู่ใน requirement เดิมของ capstone แต่ถูกเพิ่มเข้ามา**: `GET /health` เป็น
มาตรฐานที่ Part 94 (deployment) ต้องใช้แน่นอนสำหรับ health check ของ container/load balancer — เพิ่มตั้งแต่
ตอนนี้เพื่อไม่ให้ Part 94 ต้องกลับมาแก้ backend, ส่วน `GET /api/v1/me` เป็น endpoint พื้นฐานที่ frontend ทุก
ระบบที่มี auth ต้องมี (เพื่อรู้ว่า "ใคร login อยู่" ตอนโหลดหน้าเว็บใหม่ หรือ refresh หน้า) — การไม่เพิ่มมันไว้
ตั้งแต่ตอนนี้จะบีบให้ Part 93 ต้องขอเพิ่ม endpoint กลางทาง

### 92.2 โครงสร้างโปรเจกต์: จัดโค้ดเป็น module อย่างมีระบบ (ต่อยอด Part 16-17)

โปรเจกต์ระดับนี้ (10 endpoint, auth, DB, docs, tests) ถ้ายัดทุกอย่างไว้ใน `main.rs` ไฟล์เดียวจะกลายเป็นไฟล์
หลักพันบรรทัดที่หาอะไรไม่เจอทันที — ตามหลักการแบ่ง module จาก Part 16 (แบ่งตาม **ความรับผิดชอบ** ไม่ใช่แบ่ง
ตาม "ขนาดไฟล์") โครงสร้างที่ใช้ในบทนี้คือ:

```
library_api/
├── Cargo.toml
├── migrations/
│   ├── 0001_create_users.sql
│   ├── 0002_create_books.sql
│   └── 0003_create_borrows.sql
├── src/
│   ├── main.rs           -- entry point: อ่าน config, สร้าง PgPool, รัน migration, เปิด server
│   ├── lib.rs             -- build_app(): ประกอบ Router ทั้งหมด (ให้ทั้ง main.rs และ tests ใช้ร่วมกัน)
│   ├── state.rs            -- AppState (PgPool + jwt_secret)
│   ├── error.rs            -- AppError (ศูนย์กลาง error handling ของทั้งแอป)
│   ├── auth/
│   │   ├── mod.rs
│   │   ├── jwt.rs           -- Claims, create_access_token, decode_access_token
│   │   ├── password.rs      -- hash_password, verify_password (argon2)
│   │   └── extractor.rs     -- CurrentUser: FromRequestParts extractor
│   ├── models/
│   │   ├── mod.rs
│   │   ├── user.rs           -- UserRow, UserResponse, RegisterRequest, LoginRequest, LoginResponse
│   │   ├── book.rs            -- BookResponse, CreateBookRequest, ListBooksQuery, BookListResponse
│   │   └── borrow.rs           -- BorrowResponse, BorrowListResponse
│   ├── routes/
│   │   ├── mod.rs              -- build_router(): รวม route ทั้งหมด
│   │   ├── health.rs
│   │   ├── auth.rs               -- register, login, me
│   │   ├── books.rs               -- list_books, get_book, create_book
│   │   └── borrows.rs              -- borrow_book, return_book, my_borrowed_books
│   └── docs.rs                     -- utoipa ApiDoc (OpenAPI spec)
└── tests/
    └── api_tests.rs                  -- integration test suite เต็มรูปแบบ
```

**เหตุผลของการแบ่งแบบนี้ทีละจุด**:

- **`models/` แยกจาก `routes/`** — struct ที่แทนข้อมูล (`BookResponse`, `UserResponse` ฯลฯ) ถูกใช้ทั้งใน
  handler, ใน `docs.rs` (สำหรับ `#[derive(ToSchema)]`), และใน `tests/` — ถ้าฝัง struct เหล่านี้ไว้ในไฟล์
  handler จะเกิด circular reference เมื่อ `docs.rs` ต้อง `use` มันมาสร้าง schema
- **`auth/` เป็น module ของตัวเอง แยกจาก `routes/auth.rs`** — `auth/jwt.rs`, `auth/password.rs`,
  `auth/extractor.rs` เป็น "กลไกของ authentication" (การเซ็น/ตรวจ JWT, การ hash password, การ extract
  `CurrentUser` จาก request) ในขณะที่ `routes/auth.rs` เป็น "endpoint ที่ใช้กลไกเหล่านั้น" (`register`,
  `login`, `me`) — สองอย่างนี้เปลี่ยนแปลงด้วยเหตุผลคนละแบบ (กลไก JWT เปลี่ยนน้อยมาก ส่วน endpoint อาจเพิ่ม/
  แก้บ่อยกว่า) แยกกันจึงชัดเจนกว่า
- **`error.rs` เป็นไฟล์เดียว ไม่ใช่ module ที่มีไฟล์ย่อย** — เพราะ `AppError` enum ต้องเป็นจุดศูนย์กลางเดียว
  ที่ทุกที่ใน codebase มองเห็นและ `use` ได้ง่าย การแตกเป็นหลายไฟล์จะทำให้ผู้อ่านโค้ดต้องไล่หา variant ทั้งหมด
  จากหลายที่โดยไม่ได้ประโยชน์อะไรเพิ่ม
- **`lib.rs` แยกจาก `main.rs`** — นี่คือจุดสำคัญสำหรับการเทสต์: `lib.rs` export ฟังก์ชัน `build_app(state,
  frontend_origin) -> Router` ที่ประกอบ route + middleware ทั้งหมด แล้ว `main.rs` (ที่มีแค่หน้าที่อ่าน env,
  เปิด `PgPool`, รัน migration, เปิด TCP listener) และ `tests/api_tests.rs` (ที่ต้องการ `Router` ไปทดสอบผ่าน
  `tower::ServiceExt::oneshot` โดยไม่ต้องเปิด TCP listener จริง) ต่างก็เรียก `build_app` ตัวเดียวกัน — ไม่มี
  การเขียน "ประกอบ Router" ซ้ำสองที่ ตามหลักการ DRY เดียวกับที่ Part 65 อธิบายไว้เรื่อง middleware

### 92.3 Database Schema และ Migrations (ต่อยอด Part 71)

สร้างโปรเจกต์และเริ่มจากฐานข้อมูลก่อนเสมอ — ต่อยอด `sqlx-cli` จาก Part 70-71:

```bash
cargo new library_api
cd library_api
cargo add axum --features macros
cargo add tokio --features full
cargo add tower --features util
cargo add tower-http --features cors,trace
cargo add serde --features derive
cargo add serde_json
cargo add sqlx --features runtime-tokio,postgres,macros,chrono,migrate
cargo add thiserror
cargo add anyhow
cargo add tracing
cargo add tracing-subscriber --features env-filter
cargo add jsonwebtoken --features rust_crypto
cargo add argon2
cargo add chrono --features serde
cargo add utoipa --features axum_extras,chrono
cargo add utoipa-swagger-ui --features axum,vendored

cargo install sqlx-cli --no-default-features --features rustls,postgres

createdb library_api_dev   # หรือ: psql -c "CREATE DATABASE library_api_dev;"
export DATABASE_URL="postgres://postgres:postgres@127.0.0.1:5432/library_api_dev"

sqlx migrate add -r create_users
sqlx migrate add -r create_books
sqlx migrate add -r create_borrows
```

(`sqlx migrate add -r <name>` สร้างไฟล์คู่ `up`/`down` — บทนี้แสดงเฉพาะ `up` เพราะเป็นไฟล์ที่ต้องเข้าใจ schema
จริง `down` เพียงแค่ `DROP TABLE` กลับตามลำดับย้อนกัน ตามที่ Part 71 สอนไว้แล้ว)

**Migration 1 — `users`**:

```sql
-- migrations/0001_create_users.sql
CREATE TABLE users (
    id            BIGSERIAL PRIMARY KEY,
    username      TEXT NOT NULL UNIQUE,
    email         TEXT NOT NULL UNIQUE,
    password_hash TEXT NOT NULL,
    role          TEXT NOT NULL DEFAULT 'member' CHECK (role IN ('member', 'admin')),
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_username ON users (username);
```

**อธิบายการออกแบบ**: `username`/`email` เป็น `UNIQUE` ทั้งคู่ (constraint ระดับฐานข้อมูล — ไม่พึ่งแค่การเช็ค
ในโค้ด Rust เพราะมี race condition เดียวกับที่จะพูดถึงในหัวข้อ borrow/return ถ้าสอง request สมัครด้วย
username เดียวกัน "พร้อมกัน" การเช็ค `SELECT ... WHERE username = ?` ก่อน `INSERT` ในโค้ดอย่างเดียวไม่ปลอดภัย
100% — ต้องให้ database constraint เป็นตัวรับประกันสุดท้ายเสมอ), `role` เป็น `TEXT` ธรรมดาพร้อม `CHECK`
constraint จำกัดค่าที่เป็นไปได้แค่สองค่า (`member`/`admin`) — เลือก `TEXT` + `CHECK` แทน Postgres `ENUM` เพราะ
`ALTER TYPE ... ADD VALUE` ของ Postgres enum มีข้อจำกัดหลายอย่างเวลาต้องเพิ่ม role ใหม่ในอนาคต (เช่น `staff`)
ในขณะที่แก้ `CHECK` constraint ทำได้ตรงไปตรงมากว่า, `idx_users_username` เพิ่มเข้ามาแม้ `UNIQUE` จะสร้าง index
ให้อัตโนมัติอยู่แล้ว (index นี้จึงซ้ำซ้อนในทางเทคนิค — ใส่ไว้เพื่อความชัดเจนในการสอนว่า `username` คือคอลัมน์
ที่ query บ่อยที่สุด (ทุกครั้งที่ login) แต่ในโปรเจกต์จริงไม่ต้องสร้างซ้ำเพราะ Postgres ทำให้แล้ว)

**Migration 2 — `books`**:

```sql
-- migrations/0002_create_books.sql
CREATE TABLE books (
    id                BIGSERIAL PRIMARY KEY,
    title             TEXT NOT NULL,
    author            TEXT NOT NULL,
    isbn              TEXT NOT NULL UNIQUE,
    category          TEXT NOT NULL,
    total_copies      INT NOT NULL CHECK (total_copies >= 0),
    available_copies  INT NOT NULL CHECK (available_copies >= 0),
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT available_not_more_than_total CHECK (available_copies <= total_copies)
);

CREATE INDEX idx_books_category ON books (category);
CREATE INDEX idx_books_title ON books (title);
```

**อธิบายการออกแบบ**: `isbn UNIQUE` ป้องกันหนังสือซ้ำ (endpoint สร้างหนังสือใน 92.6 จะพึ่ง constraint นี้เพื่อ
ตอบ `409 Conflict` โดยไม่ต้อง `SELECT` เช็คก่อนเลย), `total_copies`/`available_copies` มี `CHECK (>= 0)` แยก
กันสองอัน**บวก** constraint ที่สาม `available_not_more_than_total` — นี่คือ**ตาข่ายนิรภัยชั้นสุดท้าย**ระดับ
ฐานข้อมูล ถ้าโค้ด Rust มีบั๊กที่พยายามลด `available_copies` ต่ำกว่า 0 หรือมากกว่า `total_copies` (ไม่ว่าจะบั๊ก
จากตรงไหนก็ตาม รวมถึงบั๊กที่ยังไม่รู้จักในอนาคต) ฐานข้อมูลจะปฏิเสธ `UPDATE`/`INSERT` นั้นทันทีด้วย error
`check_violation` แทนที่จะยอมให้ข้อมูลเพี้ยนแบบเงียบ ๆ — นี่คือหลักการ **defense in depth** เดียวกับที่ Part 71
พูดถึงเรื่อง constraint ระดับฐานข้อมูลไม่ใช่ "ทางเลือก" แต่เป็น "ชั้นป้องกันสุดท้ายที่โค้ด Rust ไม่ควรเป็นชั้น
เดียว"

**Migration 3 — `borrows`**:

```sql
-- migrations/0003_create_borrows.sql
CREATE TABLE borrows (
    id           BIGSERIAL PRIMARY KEY,
    book_id      BIGINT NOT NULL REFERENCES books (id),
    user_id      BIGINT NOT NULL REFERENCES users (id),
    borrowed_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    due_at       TIMESTAMPTZ NOT NULL,
    returned_at  TIMESTAMPTZ NULL
);

CREATE INDEX idx_borrows_user_active ON borrows (user_id) WHERE returned_at IS NULL;
CREATE INDEX idx_borrows_book_active ON borrows (book_id) WHERE returned_at IS NULL;

-- ผู้ใช้คนเดียวยืมหนังสือเล่มเดียวกัน "พร้อมกัน" ได้แค่ 1 รายการที่ยัง active
-- (ป้องกันการกดยืมซ้ำสองครั้งติดกันจนเกิดแถว active สองแถวสำหรับคู่ book_id/user_id เดียวกัน)
CREATE UNIQUE INDEX idx_borrows_one_active_per_user_book
    ON borrows (book_id, user_id)
    WHERE returned_at IS NULL;
```

**อธิบายการออกแบบทีละจุด — นี่คือตารางที่ซับซ้อนที่สุดของทั้งสามตาราง**:

- **ไม่มีคอลัมน์ `status` (เช่น `'borrowed'`/`'returned'`)** — ใช้ **`returned_at IS NULL` แทนการมี field
  status แยก** เพราะข้อมูลนี้ "เป็น" เวลาที่คืนอยู่แล้ว การมี field `status` ควบคู่ไปด้วยจะสร้างความเป็นไปได้ที่
  `status = 'returned'` แต่ `returned_at` เป็น `NULL` (หรือกลับกัน) ซึ่งเป็นสถานะที่ไม่สมเหตุสมผลแต่ database
  ไม่ป้องกันให้เอง — การเลือกให้ `returned_at` เป็น**แหล่งข้อมูลเดียว** (single source of truth) ของสถานะการ
  คืนตัดปัญหานี้ทั้งหมด (ตรงกับหลักการ "avoid impossible states" ที่ type system ของ Rust เองก็สอนไว้ตลอด
  หลักสูตรผ่าน `enum`/`Option` — ที่นี่ประยุกต์หลักการเดียวกันไปที่ระดับ schema ฐานข้อมูล)
- **`idx_borrows_user_active`/`idx_borrows_book_active` เป็น partial index (`WHERE returned_at IS NULL`)**
  — คำสั่งที่ query บ่อยที่สุดของทั้งระบบคือ "หา active borrow ของ user/book นี้" (ใช้ในทุก handler ของ
  borrow/return) ซึ่งกรองด้วย `returned_at IS NULL` เสมอ — partial index จะเล็กกว่า index ปกติมาก (มีแค่แถว
  ที่ active เท่านั้น ไม่รวมแถวเก่าที่คืนไปแล้วเป็นพัน ๆ แถวเมื่อระบบใช้งานมานาน) และเร็วกว่าสำหรับ query
  pattern นี้โดยเฉพาะ
- **`idx_borrows_one_active_per_user_book` เป็น unique partial index** — นี่คือ constraint ที่ตั้งใจเพิ่มเข้า
  มา**นอกเหนือ**จาก requirement เดิม (การกันยืมซ้ำถูกเช็คในโค้ด Rust อยู่แล้วในหัวข้อ 92.7 ด้วย) เพื่อเป็น
  ตาข่ายนิรภัยชั้นที่สองสำหรับกรณี **race condition ระดับ "ยืมซ้ำ"**: ถ้าผู้ใช้คนเดียวกันกดปุ่ม "ยืม" สองครั้ง
  เร็ว ๆ (double-click, หรือ retry จาก network เดิม) ก่อนที่ transaction แรกจะ commit เสร็จ การเช็คใน
  application code (`SELECT EXISTS(...)` ก่อน `INSERT`) อาจไม่ทันเห็นแถวที่ transaction คู่ขนานยัง insert ไม่
  เสร็จ — unique index ระดับฐานข้อมูลจะปฏิเสธการ `INSERT` ที่สองด้วย `23505 unique_violation` เสมอไม่ว่า
  timing จะบังเอิญตรงกันแค่ไหนก็ตาม (`AppError`'s `From<sqlx::Error>` ในหัวข้อ 92.4 แปลง error code นี้เป็น
  `409 Conflict` ให้อัตโนมัติอยู่แล้ว)

รัน migration และตรวจสอบ:

```bash
$ sqlx migrate run
Applied 1/migrate create users (8.805901ms)
Applied 2/migrate create books (5.333591ms)
Applied 3/migrate create borrows (8.874261ms)

$ sqlx migrate info
1/installed create users
2/installed create books
3/installed create borrows
```

### 92.4 AppState และ AppError (ต่อยอด Part 66 + 70 + 76)

```rust
// src/state.rs
use std::sync::Arc;

use sqlx::PgPool;

/// AppState ของทั้งแอป — เก็บ PgPool (ตาม Part 70) และ jwt_secret (ตาม Part 74)
/// Clone ได้ราคาถูกเพราะ PgPool ข้างในเป็น Arc อยู่แล้ว (Part 70 หัวข้อ 70.3) และ jwt_secret
/// เป็น Arc<str> เพื่อไม่ต้อง clone ตัว String จริงทุกครั้งที่ extractor ต้องใช้
#[derive(Clone)]
pub struct AppState {
    pub db: PgPool,
    pub jwt_secret: Arc<str>,
}
```

ต่อไปนี้คือ `AppError` ฉบับเต็มของบทนี้ — เทียบกับ `AppError` ของ Part 66 มีการเปลี่ยนแปลงสามจุด: (1) เพิ่ม
variant `Forbidden` ตามที่ Part 76 สอนไว้ (2) `NotFound`/`Unauthorized` เปลี่ยนจากไม่มี field เป็นมี
`String` เพื่อให้ handler ระบุรายละเอียดที่ปลอดภัยจะบอก client ได้ (เช่น "ไม่พบหนังสือ id 999") และ (3) เพิ่ม
`impl From<sqlx::Error>` ที่แปลง `unique_violation` เป็น `409` โดยอัตโนมัติ — เป็นจุดเชื่อมระหว่าง Part 66
(error handling) กับ Part 70-71 (SQLx) ที่ทั้งสองบทยังไม่ได้ทำให้เห็นเต็มรูปแบบ:

```rust
// src/error.rs
use axum::{
    http::StatusCode,
    response::{IntoResponse, Response},
    Json,
};
use serde::Serialize;
use serde_json::json;
use thiserror::Error;

/// field-level validation error หนึ่งจุด — ใช้กับ AppError::ValidationErrors
#[derive(Debug, Clone, Serialize, utoipa::ToSchema)]
pub struct FieldError {
    pub field: String,
    pub message: String,
}

/// enum เดียวสำหรับ error ทั้งแอป — ต่อยอด Part 66 (AppError) + Part 76 (variant Forbidden)
#[derive(Debug, Error)]
pub enum AppError {
    #[error("ไม่พบข้อมูลที่ต้องการ: {0}")]
    NotFound(String),

    #[error("ข้อมูลไม่ถูกต้อง: {0}")]
    Validation(String),

    #[error("ข้อมูลไม่ผ่านการตรวจสอบ")]
    ValidationErrors(Vec<FieldError>),

    #[error("ขัดแย้งกับสถานะปัจจุบัน: {0}")]
    Conflict(String),

    #[error("ไม่ได้รับอนุญาต: {0}")]
    Unauthorized(String),

    #[error("ไม่มีสิทธิ์ทำรายการนี้: {0}")]
    Forbidden(String),

    #[error("เกิดข้อผิดพลาดภายในระบบ")]
    Internal(#[from] anyhow::Error),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, code): (StatusCode, &str) = match &self {
            AppError::NotFound(_) => (StatusCode::NOT_FOUND, "NOT_FOUND"),
            AppError::Validation(_) => (StatusCode::BAD_REQUEST, "VALIDATION_ERROR"),
            AppError::ValidationErrors(_) => (StatusCode::BAD_REQUEST, "VALIDATION_ERROR"),
            AppError::Conflict(_) => (StatusCode::CONFLICT, "CONFLICT"),
            AppError::Unauthorized(_) => (StatusCode::UNAUTHORIZED, "UNAUTHORIZED"),
            AppError::Forbidden(_) => (StatusCode::FORBIDDEN, "FORBIDDEN"),
            AppError::Internal(_) => (StatusCode::INTERNAL_SERVER_ERROR, "INTERNAL_ERROR"),
        };

        // *** จุดศูนย์กลางของ logging ทั้งระบบ — ทุก AppError ไหลผ่านจุดนี้ก่อนกลายเป็น response ***
        match &self {
            AppError::Internal(source) => {
                tracing::error!(
                    error_code = code,
                    status = status.as_u16(),
                    error = %source,
                    error_debug = ?source,
                    "internal error เกิดขึ้นระหว่างประมวลผล request"
                );
            }
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
            // AppError::Internal ไม่เคยรั่วรายละเอียดของ anyhow::Error ไปให้ client เห็นเลย
            // (Display ของ variant นี้เป็นข้อความคงที่ ไม่มี placeholder {0})
            other => json!({
                "error": { "code": code, "message": other.to_string() }
            }),
        };

        (status, Json(body)).into_response()
    }
}

// แปลง sqlx::Error -> AppError โดยอัตโนมัติผ่าน ? ตาม Part 70/71
// (unique_violation/foreign_key_violation ของ Postgres แปลงเป็น Conflict, อย่างอื่นเป็น Internal)
impl From<sqlx::Error> for AppError {
    fn from(err: sqlx::Error) -> Self {
        if let sqlx::Error::Database(db_err) = &err {
            // 23505 = unique_violation ใน Postgres (isbn ซ้ำ, username/email ซ้ำ, borrow ซ้ำ)
            if db_err.code().as_deref() == Some("23505") {
                return AppError::Conflict("ข้อมูลนี้ซ้ำกับที่มีอยู่แล้วในระบบ".to_string());
            }
        }
        AppError::Internal(anyhow::Error::new(err))
    }
}

impl From<argon2::password_hash::Error> for AppError {
    fn from(err: argon2::password_hash::Error) -> Self {
        AppError::Internal(anyhow::anyhow!("password hashing error: {err}"))
    }
}
```

**อธิบายจุดที่ต่อยอดจาก Part 66 มากที่สุด**: `impl From<sqlx::Error> for AppError` คือสิ่งที่ทำให้ handler
ทุกตัวในบทนี้ที่เขียน `sqlx::query!(...).fetch_one(&pool).await?` **ไม่ต้องเขียน `.map_err(...)` เลยแม้แต่
จุดเดียว** — `?` desugar เป็น `From::from(error)` ตามที่ Part 12/30/31 สอนไว้ ทำให้ `sqlx::Error` ใดก็ตาม
กลายเป็น `AppError` ที่ถูกต้องโดยอัตโนมัติ (ถ้าเป็น unique_violation กลายเป็น `Conflict`/409, ถ้าเป็นอย่างอื่น
กลายเป็น `Internal`/500) — สังเกตว่านี่คือรูปแบบเดียวกับ `#[from] anyhow::Error` ของ Part 66 เพียงแต่คราวนี้
เขียน `impl From` ด้วยมือ (ไม่ใช้ `#[from]` ของ `thiserror`) เพราะต้อง**ตรวจสอบเนื้อหาของ error ก่อนเลือก
variant ปลายทาง** (`#[from]` ของ `thiserror` แปลงตรงไปยัง variant เดียวเสมอ ไม่มีทาง "แยกกรณี" แบบนี้ได้)

### 92.5 Authentication: JWT + Argon2 (ต่อยอด Part 74)

โมดูล `auth/` มีสามไฟล์ — `jwt.rs` (การเซ็น/ตรวจ token), `password.rs` (การ hash/verify password), และ
`extractor.rs` (การดึง `CurrentUser` จาก request):

```rust
// src/auth/jwt.rs
use chrono::Utc;
use jsonwebtoken::{decode, encode, Algorithm, DecodingKey, EncodingKey, Header, Validation};
use serde::{Deserialize, Serialize};

/// อายุของ access token — 1 ชั่วโมง (ต่อยอด Part 74 หัวข้อ 74.5)
pub const ACCESS_TOKEN_TTL_SECONDS: i64 = 60 * 60;

/// Claims ของ access token — ผสาน standard claims (sub, iat, exp) กับ custom claims
/// (username, role) ตาม RFC 7519 และ Part 74 หัวข้อ 74.5
#[derive(Debug, Serialize, Deserialize, Clone)]
pub struct Claims {
    /// subject — user id แปลงเป็น String ตามธรรมเนียม JWT
    pub sub: String,
    pub username: String,
    /// "member" หรือ "admin" — ใช้เช็ค RBAC ตาม Part 76
    pub role: String,
    pub iat: usize,
    pub exp: usize,
}

pub fn create_access_token(
    secret: &[u8],
    user_id: i64,
    username: &str,
    role: &str,
) -> anyhow::Result<String> {
    let now = Utc::now().timestamp();
    let claims = Claims {
        sub: user_id.to_string(),
        username: username.to_string(),
        role: role.to_string(),
        iat: now as usize,
        exp: (now + ACCESS_TOKEN_TTL_SECONDS) as usize,
    };
    let header = Header::new(Algorithm::HS256);
    let key = EncodingKey::from_secret(secret);
    encode(&header, &claims, &key).map_err(|e| anyhow::anyhow!("ออก JWT ไม่สำเร็จ: {e}"))
}

pub fn decode_access_token(secret: &[u8], token: &str) -> Result<Claims, jsonwebtoken::errors::Error> {
    let decoding_key = DecodingKey::from_secret(secret);
    let validation = Validation::new(Algorithm::HS256);
    decode::<Claims>(token, &decoding_key, &validation).map(|data| data.claims)
}
```

```rust
// src/auth/password.rs
use argon2::{
    password_hash::{phc::PasswordHash, PasswordHasher, PasswordVerifier},
    Argon2,
};

/// hash password ด้วย argon2id (default ของ crate `argon2`) — `hash_password` (ฟีเจอร์
/// `getrandom` ที่เปิดเป็น default อยู่แล้ว) สุ่ม salt ใหม่ให้เองทุกครั้งที่เรียก ตาม Part 74
/// หัวข้อ 74.6 — ห้ามเก็บ/เทียบ password เป็น plaintext เด็ดขาด
pub fn hash_password(password: &str) -> Result<String, argon2::password_hash::Error> {
    let argon2 = Argon2::default();
    Ok(argon2.hash_password(password.as_bytes())?.to_string())
}

/// verify password กับ hash ที่เก็บไว้ — คืน false ถ้าไม่ตรง (ไม่ propagate error ธรรมดา
/// เพราะ "password ผิด" ไม่ใช่ error ของระบบ เป็นผลลัพธ์ปกติของการ login ที่ผู้ใช้พิมพ์ผิด)
pub fn verify_password(password: &str, hash: &str) -> bool {
    let parsed_hash = match PasswordHash::new(hash) {
        Ok(h) => h,
        Err(_) => return false,
    };
    Argon2::default()
        .verify_password(password.as_bytes(), &parsed_hash)
        .is_ok()
}
```

```rust
// src/auth/extractor.rs
use axum::{
    extract::FromRequestParts,
    http::{request::Parts, StatusCode},
    response::{IntoResponse, Response},
    Json,
};
use serde_json::json;

use crate::{auth::jwt::decode_access_token, state::AppState};

/// ผู้ใช้ที่ผ่านการตรวจ JWT แล้ว — handler ที่ต้องการ authentication ใส่ `CurrentUser`
/// เป็นพารามิเตอร์ได้ตรง ๆ (ตาม pattern FromRequestParts จาก Part 64 + Part 74 หัวข้อ 74.7)
#[derive(Debug, Clone)]
pub struct CurrentUser {
    pub user_id: i64,
    pub username: String,
    pub role: String,
}

impl CurrentUser {
    pub fn is_admin(&self) -> bool {
        self.role == "admin"
    }
}

/// error ตอน extract ล้มเหลว — implement IntoResponse เอง ให้ตรง JSON shape เดียวกับ AppError
/// ของ Part 66 (แยกจาก AppError เพราะ extractor rejection เกิด "ก่อน" handler เริ่มทำงาน)
pub struct AuthRejection(pub String);

impl IntoResponse for AuthRejection {
    fn into_response(self) -> Response {
        tracing::warn!(reason = %self.0, "JWT authentication ล้มเหลว");
        (
            StatusCode::UNAUTHORIZED,
            Json(json!({ "error": { "code": "UNAUTHORIZED", "message": self.0 } })),
        )
            .into_response()
    }
}

impl FromRequestParts<AppState> for CurrentUser {
    type Rejection = AuthRejection;

    async fn from_request_parts(
        parts: &mut Parts,
        state: &AppState,
    ) -> Result<Self, Self::Rejection> {
        let header = parts
            .headers
            .get(axum::http::header::AUTHORIZATION)
            .and_then(|v| v.to_str().ok())
            .ok_or_else(|| AuthRejection("ไม่พบ header Authorization".to_string()))?;

        let token = header
            .strip_prefix("Bearer ")
            .ok_or_else(|| AuthRejection("Authorization header ต้องเป็นรูปแบบ 'Bearer <token>'".to_string()))?;

        let claims = decode_access_token(state.jwt_secret.as_bytes(), token)
            .map_err(|e| AuthRejection(format!("token ไม่ถูกต้องหรือหมดอายุ: {e}")))?;

        let user_id: i64 = claims
            .sub
            .parse()
            .map_err(|_| AuthRejection("token มี claim 'sub' ที่ผิดรูปแบบ".to_string()))?;

        Ok(CurrentUser {
            user_id,
            username: claims.username,
            role: claims.role,
        })
    }
}
```

```rust
// src/auth/mod.rs
pub mod extractor;
pub mod jwt;
pub mod password;

pub use extractor::CurrentUser;
pub use jwt::Claims;
```

### 92.6 Models: struct ที่ผูก Request/Response เข้ากับแถวจริงของฐานข้อมูล

```rust
// src/models/user.rs
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use utoipa::ToSchema;

/// แถวจริงในตาราง users — มี password_hash ซึ่ง**ไม่เคย** ถูกส่งออกไปใน response ใดๆ
#[derive(Debug, Clone, sqlx::FromRow)]
pub struct UserRow {
    pub id: i64,
    pub username: String,
    pub email: String,
    pub password_hash: String,
    pub role: String,
    pub created_at: DateTime<Utc>,
}

/// รูปแบบ user ที่ปลอดภัยสำหรับส่งออกไปให้ client — ไม่มี password_hash เด็ดขาด
#[derive(Debug, Clone, Serialize, ToSchema)]
pub struct UserResponse {
    pub id: i64,
    pub username: String,
    pub email: String,
    pub role: String,
    pub created_at: DateTime<Utc>,
}

impl From<UserRow> for UserResponse {
    fn from(row: UserRow) -> Self {
        UserResponse {
            id: row.id,
            username: row.username,
            email: row.email,
            role: row.role,
            created_at: row.created_at,
        }
    }
}

#[derive(Debug, Deserialize, ToSchema)]
pub struct RegisterRequest {
    /// 3-32 ตัวอักษร, a-z/A-Z/0-9/underscore เท่านั้น
    pub username: String,
    pub email: String,
    /// อย่างน้อย 8 ตัวอักษร
    pub password: String,
}

#[derive(Debug, Deserialize, ToSchema)]
pub struct LoginRequest {
    pub username: String,
    pub password: String,
}

#[derive(Debug, Serialize, ToSchema)]
pub struct LoginResponse {
    pub access_token: String,
    pub token_type: String,
    /// อายุ token เป็นวินาที
    pub expires_in: i64,
    pub user: UserResponse,
}
```

**อธิบายการออกแบบที่สำคัญที่สุดของไฟล์นี้**: มีสอง struct สำหรับ "user" — `UserRow` (มี `password_hash`,
map ตรงจากแถวในตาราง `users` ด้วย `#[derive(sqlx::FromRow)]`) และ `UserResponse` (ไม่มี `password_hash`,
ใช้ `Serialize` ส่งออกไปยัง client) — **การแยกสอง type นี้ไม่ใช่การเขียนซ้ำที่ไม่มีประโยชน์** มันคือการทำให้
"ไม่ส่ง `password_hash` ออกไปให้ client" เป็นสิ่งที่ **compiler การันตีให้แทนการ "จำไว้ว่าต้องระวัง"** — ถ้า
`UserRow` ทั้ง struct ถูกส่งให้ `Json(...)` ตรง ๆ (ไม่ผ่าน `.into()`) `password_hash` (แม้จะ hash แล้วก็ตาม)
จะรั่วออกไปด้วย เพราะ `UserRow` ไม่มี `#[derive(Serialize)]` เลย **`cargo build` จะ error ทันทีถ้ามีใครพยายาม
ทำแบบนั้น** — นี่คือตัวอย่างที่ดีของการใช้ type system ป้องกันข้อผิดพลาดด้านความปลอดภัยที่ Part 30/57 พูดถึงไว้

```rust
// src/models/book.rs
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use utoipa::ToSchema;

#[derive(Debug, Clone, sqlx::FromRow, Serialize, ToSchema)]
pub struct BookResponse {
    pub id: i64,
    pub title: String,
    pub author: String,
    pub isbn: String,
    pub category: String,
    pub total_copies: i32,
    pub available_copies: i32,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, ToSchema)]
pub struct CreateBookRequest {
    pub title: String,
    pub author: String,
    /// ต้องเป็นตัวเลข 10 หรือ 13 หลัก
    pub isbn: String,
    pub category: String,
    /// จำนวนสำเนาทั้งหมด — ต้องมากกว่า 0
    pub total_copies: i32,
}

#[derive(Debug, Deserialize, ToSchema, Default)]
pub struct ListBooksQuery {
    pub page: Option<i64>,
    pub per_page: Option<i64>,
    pub category: Option<String>,
    /// ค้นหาใน title/author แบบ case-insensitive substring match
    pub q: Option<String>,
    /// true = แสดงเฉพาะหนังสือที่ available_copies > 0
    pub available_only: Option<bool>,
}

#[derive(Debug, Serialize, ToSchema)]
pub struct PaginationMeta {
    pub page: i64,
    pub per_page: i64,
    pub total_items: i64,
    pub total_pages: i64,
}

#[derive(Debug, Serialize, ToSchema)]
pub struct BookListResponse {
    pub data: Vec<BookResponse>,
    pub meta: PaginationMeta,
}
```

`BookResponse` เป็น struct เดียวที่ใช้ทั้ง map จากฐานข้อมูล (`sqlx::FromRow`) และ serialize ออกไปเป็น response
(`Serialize`) เพราะ**ไม่มี field ใดใน `books` ที่เป็นความลับ** ต่างจาก `users` ที่มี `password_hash` — เมื่อไม่
มีอะไรต้องซ่อน การมี struct เดียวใช้ทั้งสองด้านลดความซ้ำซ้อนได้โดยไม่เสียความปลอดภัยอะไรไป

```rust
// src/models/borrow.rs
use chrono::{DateTime, Utc};
use serde::Serialize;
use utoipa::ToSchema;

#[derive(Debug, Clone, sqlx::FromRow, Serialize, ToSchema)]
pub struct BorrowResponse {
    pub id: i64,
    pub book_id: i64,
    pub user_id: i64,
    pub borrowed_at: DateTime<Utc>,
    pub due_at: DateTime<Utc>,
    pub returned_at: Option<DateTime<Utc>>,
}

#[derive(Debug, Serialize, ToSchema)]
pub struct BorrowListResponse {
    pub data: Vec<BorrowResponse>,
}
```

```rust
// src/models/mod.rs
pub mod book;
pub mod borrow;
pub mod user;

pub use book::*;
pub use borrow::*;
pub use user::*;
```

### 92.7 Auth Endpoints: register / login / me

```rust
// src/routes/auth.rs
use axum::{extract::State, http::StatusCode, Json};

use crate::{
    auth::{
        jwt::{create_access_token, ACCESS_TOKEN_TTL_SECONDS},
        password::{hash_password, verify_password},
        CurrentUser,
    },
    error::{AppError, FieldError},
    models::{LoginRequest, LoginResponse, RegisterRequest, UserResponse, UserRow},
    state::AppState,
};

fn validate_register(input: &RegisterRequest) -> Result<(), AppError> {
    let mut errors = Vec::new();

    let username_ok = (3..=32).contains(&input.username.len())
        && input
            .username
            .chars()
            .all(|c| c.is_ascii_alphanumeric() || c == '_');
    if !username_ok {
        errors.push(FieldError {
            field: "username".to_string(),
            message: "ต้องมี 3-32 ตัวอักษร และเป็น a-z, A-Z, 0-9 หรือ _ เท่านั้น".to_string(),
        });
    }

    if !input.email.contains('@') || input.email.starts_with('@') || input.email.ends_with('@') {
        errors.push(FieldError {
            field: "email".to_string(),
            message: "รูปแบบอีเมลไม่ถูกต้อง".to_string(),
        });
    }

    if input.password.len() < 8 {
        errors.push(FieldError {
            field: "password".to_string(),
            message: "ต้องมีอย่างน้อย 8 ตัวอักษร".to_string(),
        });
    }

    if errors.is_empty() {
        Ok(())
    } else {
        Err(AppError::ValidationErrors(errors))
    }
}

/// POST /api/v1/auth/register — สมัครสมาชิกใหม่ (role เริ่มต้นเป็น "member" เสมอ ไม่มีทาง
/// สมัครเป็น admin ได้เองผ่าน endpoint นี้ — สร้าง admin ต้องทำผ่าน DB โดยตรง ดูหัวข้อ 92.9)
#[utoipa::path(
    post,
    path = "/api/v1/auth/register",
    tag = "auth",
    request_body = RegisterRequest,
    responses(
        (status = 201, description = "สมัครสมาชิกสำเร็จ", body = UserResponse),
        (status = 400, description = "ข้อมูลไม่ผ่านการตรวจสอบ"),
        (status = 409, description = "username หรือ email ถูกใช้ไปแล้ว"),
    ),
)]
pub async fn register(
    State(state): State<AppState>,
    Json(req): Json<RegisterRequest>,
) -> Result<(StatusCode, Json<UserResponse>), AppError> {
    validate_register(&req)?;

    // *** ห้ามเก็บ/เทียบ password เป็น plaintext เด็ดขาด — hash ด้วย argon2 ก่อนเก็บเสมอ ***
    let password_hash = hash_password(&req.password)?;

    let row = sqlx::query_as!(
        UserRow,
        r#"
        INSERT INTO users (username, email, password_hash, role)
        VALUES ($1, $2, $3, 'member')
        RETURNING id, username, email, password_hash, role, created_at
        "#,
        req.username,
        req.email,
        password_hash,
    )
    .fetch_one(&state.db)
    .await?; // unique_violation (username/email ซ้ำ) ถูกแปลงเป็น AppError::Conflict ผ่าน From<sqlx::Error>

    Ok((StatusCode::CREATED, Json(row.into())))
}

/// POST /api/v1/auth/login — ตรวจสอบ username/password แล้วออก JWT access token
#[utoipa::path(
    post,
    path = "/api/v1/auth/login",
    tag = "auth",
    request_body = LoginRequest,
    responses(
        (status = 200, description = "login สำเร็จ", body = LoginResponse),
        (status = 401, description = "username หรือ password ไม่ถูกต้อง"),
    ),
)]
pub async fn login(
    State(state): State<AppState>,
    Json(req): Json<LoginRequest>,
) -> Result<Json<LoginResponse>, AppError> {
    let row = sqlx::query_as!(
        UserRow,
        r#"SELECT id, username, email, password_hash, role, created_at FROM users WHERE username = $1"#,
        req.username,
    )
    .fetch_optional(&state.db)
    .await?
    // *** ข้อความ error เดียวกันทั้งกรณี "ไม่มี username นี้" และ "password ผิด" ***
    // (ไม่บอกว่าฝั่งไหนผิด — ป้องกัน attacker enumerate username ที่มีจริงในระบบ)
    .ok_or_else(|| AppError::Unauthorized("username หรือ password ไม่ถูกต้อง".to_string()))?;

    if !verify_password(&req.password, &row.password_hash) {
        return Err(AppError::Unauthorized(
            "username หรือ password ไม่ถูกต้อง".to_string(),
        ));
    }

    let access_token = create_access_token(state.jwt_secret.as_bytes(), row.id, &row.username, &row.role)?;

    Ok(Json(LoginResponse {
        access_token,
        token_type: "Bearer".to_string(),
        expires_in: ACCESS_TOKEN_TTL_SECONDS,
        user: row.into(),
    }))
}

/// GET /api/v1/me — ข้อมูลผู้ใช้ปัจจุบันตาม token ที่แนบมา (ต้อง login)
#[utoipa::path(
    get,
    path = "/api/v1/me",
    tag = "auth",
    security(("bearer_auth" = [])),
    responses(
        (status = 200, description = "ข้อมูลผู้ใช้ปัจจุบัน", body = UserResponse),
        (status = 401, description = "ไม่ได้ login หรือ token ไม่ถูกต้อง"),
    ),
)]
pub async fn me(
    State(state): State<AppState>,
    current_user: CurrentUser,
) -> Result<Json<UserResponse>, AppError> {
    let row = sqlx::query_as!(
        UserRow,
        r#"SELECT id, username, email, password_hash, role, created_at FROM users WHERE id = $1"#,
        current_user.user_id,
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or_else(|| AppError::NotFound("ไม่พบผู้ใช้นี้ในระบบแล้ว".to_string()))?;

    Ok(Json(row.into()))
}
```

**พิสูจน์ด้วยเซิร์ฟเวอร์จริง** (เซิร์ฟเวอร์เต็มรูปแบบรันที่ `http://127.0.0.1:8092` — ประกอบครบทุก route ในหัวข้อ
92.10):

```bash
$ curl -sS -i -X POST http://127.0.0.1:8092/api/v1/auth/register \
  -H 'Content-Type: application/json' \
  -d '{"username":"nan","email":"nan@example.com","password":"password123"}'
```
```
HTTP/1.1 201 Created
content-type: application/json
access-control-allow-origin: http://127.0.0.1:5173
content-length: 110

{"id":1,"username":"nan","email":"nan@example.com","role":"member","created_at":"2026-09-27T03:19:24.424255Z"}
```

สมัครด้วย username เดิมซ้ำ — unique constraint ของ migration 0001 ทำงานผ่าน `From<sqlx::Error>` ที่เขียนไว้
ในหัวข้อ 92.4 ทันที:

```bash
$ curl -sS -i -X POST http://127.0.0.1:8092/api/v1/auth/register \
  -H 'Content-Type: application/json' \
  -d '{"username":"nan","email":"other@example.com","password":"password123"}'
```
```
HTTP/1.1 409 Conflict
content-type: application/json

{"error":{"code":"CONFLICT","message":"ขัดแย้งกับสถานะปัจจุบัน: ข้อมูลนี้ซ้ำกับที่มีอยู่แล้วในระบบ"}}
```

ส่ง input ที่ผิดสามจุดพร้อมกัน — ได้ครบทั้งสามจุดในการตอบครั้งเดียว (ตาม pattern `ValidationErrors` จาก
Part 66 หัวข้อ 66.6):

```bash
$ curl -sS -i -X POST http://127.0.0.1:8092/api/v1/auth/register \
  -H 'Content-Type: application/json' \
  -d '{"username":"a","email":"not-an-email","password":"123"}'
```
```
HTTP/1.1 400 Bad Request
content-type: application/json

{"error":{"code":"VALIDATION_ERROR","fields":[
  {"field":"username","message":"ต้องมี 3-32 ตัวอักษร และเป็น a-z, A-Z, 0-9 หรือ _ เท่านั้น"},
  {"field":"email","message":"รูปแบบอีเมลไม่ถูกต้อง"},
  {"field":"password","message":"ต้องมีอย่างน้อย 8 ตัวอักษร"}
],"message":"ข้อมูลไม่ผ่านการตรวจสอบ"}}
```

Login ด้วย credential ที่ถูกต้อง ได้ JWT access token กลับมา (โครงสร้าง JWT เต็มรูปแบบตามที่ Part 74 อธิบาย
ไว้ — `header.payload.signature` แยกด้วยจุด):

```bash
$ curl -sS -X POST http://127.0.0.1:8092/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"nan","password":"password123"}'
```
```json
{"access_token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxIiwidXNlcm5hbWUiOiJuYW4iLCJyb2xlIjoibWVtYmVyIiwiaWF0IjoxNzkwNDc5MTc3LCJleHAiOjE3OTA0ODI3Nzd9.IJ0ach8QQOihz_9RRQ_IgBsJ48sJJIm5ZWz8sqiUw0A","token_type":"Bearer","expires_in":3600,"user":{"id":1,"username":"nan","email":"nan@example.com","role":"member","created_at":"2026-09-27T03:19:24.424255Z"}}
```

Login ด้วย password ผิด และเรียก `/api/v1/me` โดยไม่แนบ token — ทั้งสองกรณีตอบ `401` แต่ด้วยข้อความคนละแบบ
(เพราะเกิดจากจุดคนละจุดในระบบ — อันแรกจาก `AppError::Unauthorized` ใน handler, อันหลังจาก `AuthRejection`
ใน extractor — แต่ **JSON shape เดียวกันทั้งคู่** ตามที่ Part 66 ออกแบบไว้):

```bash
$ curl -sS -i -X POST http://127.0.0.1:8092/api/v1/auth/login \
  -H 'Content-Type: application/json' -d '{"username":"nan","password":"wrongpassword"}'
```
```
HTTP/1.1 401 Unauthorized
{"error":{"code":"UNAUTHORIZED","message":"ไม่ได้รับอนุญาต: username หรือ password ไม่ถูกต้อง"}}
```
```bash
$ curl -sS -i http://127.0.0.1:8092/api/v1/me
```
```
HTTP/1.1 401 Unauthorized
{"error":{"code":"UNAUTHORIZED","message":"ไม่พบ header Authorization"}}
```

### 92.8 Books Endpoints: List (Pagination/Filter), Get, Create (Admin-Only)

```rust
// src/routes/books.rs
use axum::{
    extract::{Path, Query, State},
    http::StatusCode,
    Json,
};
use sqlx::{Postgres, QueryBuilder};

use crate::{
    auth::CurrentUser,
    error::{AppError, FieldError},
    models::{BookListResponse, BookResponse, CreateBookRequest, ListBooksQuery, PaginationMeta},
    state::AppState,
};

/// เติมเงื่อนไข WHERE ที่ใช้ร่วมกันทั้ง query หลัก (SELECT ... ) และ query นับจำนวน (COUNT(*))
/// เขียนเป็นฟังก์ชันแยกเพื่อไม่ให้เงื่อนไขสองฝั่งเพี้ยนไปจากกัน (ตาม Part 71 หัวข้อ 71.2-71.3 —
/// ทุกค่าที่มาจาก query string ผ่าน `.push_bind()` เท่านั้น ไม่มีจุดใดต่อ string ตรงๆ)
fn push_book_filters(qb: &mut QueryBuilder<Postgres>, params: &ListBooksQuery) {
    qb.push(" WHERE 1 = 1 ");
    if let Some(category) = &params.category {
        qb.push(" AND category = ").push_bind(category.clone());
    }
    if let Some(q) = &params.q {
        let pattern = format!("%{}%", q.to_lowercase());
        qb.push(" AND (LOWER(title) LIKE ").push_bind(pattern.clone());
        qb.push(" OR LOWER(author) LIKE ").push_bind(pattern);
        qb.push(") ");
    }
    if params.available_only == Some(true) {
        qb.push(" AND available_copies > 0 ");
    }
}

/// GET /api/v1/books — list พร้อม pagination (page/per_page) และ filter (category, q, available_only)
/// ตาม Part 78 หัวข้อ pagination + Part 71 หัวข้อ dynamic filter ด้วย QueryBuilder
#[utoipa::path(
    get,
    path = "/api/v1/books",
    tag = "books",
    params(
        ("page" = Option<i64>, Query, description = "หน้าที่ต้องการ (เริ่มที่ 1, default 1)"),
        ("per_page" = Option<i64>, Query, description = "จำนวนต่อหน้า (default 20, สูงสุด 100)"),
        ("category" = Option<String>, Query, description = "กรองด้วย category ตรงตัว"),
        ("q" = Option<String>, Query, description = "ค้นหาใน title/author (substring, ไม่สนตัวพิมพ์เล็ก/ใหญ่)"),
        ("available_only" = Option<bool>, Query, description = "true = แสดงเฉพาะที่ available_copies > 0"),
    ),
    responses((status = 200, description = "รายการหนังสือ", body = BookListResponse)),
)]
pub async fn list_books(
    State(state): State<AppState>,
    Query(params): Query<ListBooksQuery>,
) -> Result<Json<BookListResponse>, AppError> {
    let page = params.page.unwrap_or(1).max(1);
    let per_page = params.per_page.unwrap_or(20).clamp(1, 100);
    let offset = (page - 1) * per_page;

    let mut count_qb: QueryBuilder<Postgres> = QueryBuilder::new("SELECT COUNT(*) FROM books");
    push_book_filters(&mut count_qb, &params);
    let total_items: i64 = count_qb
        .build_query_scalar()
        .fetch_one(&state.db)
        .await?;

    let mut qb: QueryBuilder<Postgres> = QueryBuilder::new(
        "SELECT id, title, author, isbn, category, total_copies, available_copies, created_at FROM books",
    );
    push_book_filters(&mut qb, &params);
    qb.push(" ORDER BY id ASC LIMIT ").push_bind(per_page);
    qb.push(" OFFSET ").push_bind(offset);

    let data: Vec<BookResponse> = qb.build_query_as().fetch_all(&state.db).await?;

    let total_pages = if total_items == 0 {
        0
    } else {
        (total_items + per_page - 1) / per_page
    };

    Ok(Json(BookListResponse {
        data,
        meta: PaginationMeta {
            page,
            per_page,
            total_items,
            total_pages,
        },
    }))
}

/// GET /api/v1/books/{id}
#[utoipa::path(
    get,
    path = "/api/v1/books/{id}",
    tag = "books",
    params(("id" = i64, Path, description = "book id")),
    responses(
        (status = 200, description = "รายละเอียดหนังสือ", body = BookResponse),
        (status = 404, description = "ไม่พบหนังสือ id นี้"),
    ),
)]
pub async fn get_book(
    State(state): State<AppState>,
    Path(id): Path<i64>,
) -> Result<Json<BookResponse>, AppError> {
    let book = sqlx::query_as!(
        BookResponse,
        r#"SELECT id, title, author, isbn, category, total_copies, available_copies, created_at
           FROM books WHERE id = $1"#,
        id,
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or_else(|| AppError::NotFound(format!("ไม่พบหนังสือ id {id}")))?;

    Ok(Json(book))
}

fn validate_create_book(input: &CreateBookRequest) -> Result<(), AppError> {
    let mut errors = Vec::new();

    if input.title.trim().is_empty() {
        errors.push(FieldError { field: "title".into(), message: "ต้องไม่เป็นค่าว่าง".into() });
    }
    if input.author.trim().is_empty() {
        errors.push(FieldError { field: "author".into(), message: "ต้องไม่เป็นค่าว่าง".into() });
    }
    let isbn_digits = input.isbn.chars().all(|c| c.is_ascii_digit());
    if !isbn_digits || !(input.isbn.len() == 10 || input.isbn.len() == 13) {
        errors.push(FieldError { field: "isbn".into(), message: "ต้องเป็นตัวเลข 10 หรือ 13 หลัก".into() });
    }
    if input.category.trim().is_empty() {
        errors.push(FieldError { field: "category".into(), message: "ต้องไม่เป็นค่าว่าง".into() });
    }
    if input.total_copies <= 0 {
        errors.push(FieldError { field: "total_copies".into(), message: "ต้องมากกว่า 0".into() });
    }

    if errors.is_empty() {
        Ok(())
    } else {
        Err(AppError::ValidationErrors(errors))
    }
}

/// POST /api/v1/books — admin เท่านั้น (ตรวจด้วย CurrentUser::is_admin() ตาม Part 76)
#[utoipa::path(
    post,
    path = "/api/v1/books",
    tag = "books",
    security(("bearer_auth" = [])),
    request_body = CreateBookRequest,
    responses(
        (status = 201, description = "สร้างหนังสือสำเร็จ", body = BookResponse),
        (status = 400, description = "ข้อมูลไม่ผ่านการตรวจสอบ"),
        (status = 401, description = "ไม่ได้ login"),
        (status = 403, description = "ไม่ใช่ admin"),
        (status = 409, description = "isbn ซ้ำกับหนังสือที่มีอยู่แล้ว"),
    ),
)]
pub async fn create_book(
    State(state): State<AppState>,
    current_user: CurrentUser,
    Json(req): Json<CreateBookRequest>,
) -> Result<(StatusCode, Json<BookResponse>), AppError> {
    // *** RBAC check ระดับ handler ตาม Part 76 หัวข้อ 76.6 — ผ่าน authentication (มี CurrentUser)
    // ไม่ได้แปลว่ามีสิทธิ์ทำทุกอย่าง ต้องเช็ค role อีกชั้นก่อนทำ business logic จริง ***
    if !current_user.is_admin() {
        return Err(AppError::Forbidden(
            "ต้องเป็น admin เท่านั้นที่สร้างหนังสือได้".to_string(),
        ));
    }

    validate_create_book(&req)?;

    let book = sqlx::query_as!(
        BookResponse,
        r#"
        INSERT INTO books (title, author, isbn, category, total_copies, available_copies)
        VALUES ($1, $2, $3, $4, $5, $5)
        RETURNING id, title, author, isbn, category, total_copies, available_copies, created_at
        "#,
        req.title,
        req.author,
        req.isbn,
        req.category,
        req.total_copies,
    )
    .fetch_one(&state.db)
    .await?; // isbn ซ้ำ -> unique_violation -> AppError::Conflict ผ่าน From<sqlx::Error>

    Ok((StatusCode::CREATED, Json(book)))
}
```

**สังเกตจุดที่ `INSERT INTO books (...) VALUES ($1, $2, $3, $4, $5, $5)`** — พารามิเตอร์ `$5` ถูกใช้**สอง
ครั้ง**ในประโยค SQL เดียว (สำหรับทั้ง `total_copies` และ `available_copies`) เพราะ**ตอนสร้างหนังสือใหม่
จำนวนที่ว่างต้องเท่ากับจำนวนทั้งหมดเสมอ** (ยังไม่มีใครยืมไปเลย) — Postgres รองรับการอ้าง placeholder ซ้ำได้
ปกติ และ `sqlx::query_as!` macro จะตรวจสอบให้ตรงว่าจำนวน argument ที่ส่งมา (5 ตัว) ตรงกับ placeholder
**ที่ไม่ซ้ำ** สูงสุด (`$5`) ไม่ใช่จำนวนครั้งที่ปรากฏ

**พิสูจน์ด้วยเซิร์ฟเวอร์จริง** — สร้าง admin ก่อน (วิธีเดียวที่ตั้งใจไว้ในสโคปนี้: **ไม่มี endpoint สมัคร
เป็น admin ได้เอง** — ต้อง register แบบธรรมดาก่อนแล้ว promote role ผ่านฐานข้อมูลตรง ๆ ซึ่งเป็นการตัดสินใจ
ด้านความปลอดภัยที่ตั้งใจทำ ไม่ใช่ทางลัดที่ลืมทำ):

```bash
$ psql -d library_api_dev -c "UPDATE users SET role='admin' WHERE username='nan';"
UPDATE 1
```

member ธรรมดาพยายามสร้างหนังสือ (มี token ที่ผ่าน authentication แล้ว แต่ role ไม่ใช่ admin) — ตอบ `403`
ไม่ใช่ `401` เพราะระบบ**รู้ว่าใครกำลังเรียก** (ผ่าน authentication) เพียงแต่คนนั้น**ไม่มีสิทธิ์**ทำ action นี้
(ตรงตามหลักการแยก 401/403 ของ Part 76 หัวข้อ 76.6):

```bash
$ curl -sS -i -X POST http://127.0.0.1:8092/api/v1/books \
  -H "Authorization: Bearer <token ของ member ธรรมดา>" -H 'Content-Type: application/json' \
  -d '{"title":"x","author":"y","isbn":"1234567890123","category":"z","total_copies":1}'
```
```
HTTP/1.1 403 Forbidden
{"error":{"code":"FORBIDDEN","message":"ไม่มีสิทธิ์ทำรายการนี้: ต้องเป็น admin เท่านั้นที่สร้างหนังสือได้"}}
```

admin สร้างหนังสือสำเร็จ:

```bash
$ curl -sS -i -X POST http://127.0.0.1:8092/api/v1/books \
  -H "Authorization: Bearer <admin token>" -H 'Content-Type: application/json' \
  -d '{"title":"The Rust Programming Language","author":"Steve Klabnik","isbn":"9781593278281","category":"programming","total_copies":2}'
```
```
HTTP/1.1 201 Created
{"id":1,"title":"The Rust Programming Language","author":"Steve Klabnik","isbn":"9781593278281","category":"programming","total_copies":2,"available_copies":2,"created_at":"2026-09-27T03:20:03.322657Z"}
```

isbn ซ้ำ, และ input ผิดห้าจุดพร้อมกัน:

```bash
$ curl -sS -i -X POST http://127.0.0.1:8092/api/v1/books \
  -H "Authorization: Bearer <admin token>" -H 'Content-Type: application/json' \
  -d '{"title":"Dup","author":"Someone","isbn":"9781593278281","category":"programming","total_copies":1}'
```
```
HTTP/1.1 409 Conflict
{"error":{"code":"CONFLICT","message":"ขัดแย้งกับสถานะปัจจุบัน: ข้อมูลนี้ซ้ำกับที่มีอยู่แล้วในระบบ"}}
```
```bash
$ curl -sS -i -X POST http://127.0.0.1:8092/api/v1/books \
  -H "Authorization: Bearer <admin token>" -H 'Content-Type: application/json' \
  -d '{"title":"","author":"","isbn":"123","category":"","total_copies":0}'
```
```
HTTP/1.1 400 Bad Request
{"error":{"code":"VALIDATION_ERROR","fields":[
  {"field":"title","message":"ต้องไม่เป็นค่าว่าง"},
  {"field":"author","message":"ต้องไม่เป็นค่าว่าง"},
  {"field":"isbn","message":"ต้องเป็นตัวเลข 10 หรือ 13 หลัก"},
  {"field":"category","message":"ต้องไม่เป็นค่าว่าง"},
  {"field":"total_copies","message":"ต้องมากกว่า 0"}
],"message":"ข้อมูลไม่ผ่านการตรวจสอบ"}}
```

list พร้อม filter (สร้างหนังสือเล่มที่สองเพิ่มเข้ามาก่อนหน้านี้ — "Zero To Production In Rust" มี 1 สำเนา):

```bash
$ curl -sS "http://127.0.0.1:8092/api/v1/books?q=rust&per_page=1&page=1"
```
```json
{"data":[{"id":1,"title":"The Rust Programming Language", "...": "..."}],"meta":{"page":1,"per_page":1,"total_items":2,"total_pages":2}}
```
```bash
$ curl -sS -i "http://127.0.0.1:8092/api/v1/books/999"
```
```
HTTP/1.1 404 Not Found
{"error":{"code":"NOT_FOUND","message":"ไม่พบข้อมูลที่ต้องการ: ไม่พบหนังสือ id 999"}}
```

### 92.9 Borrow/Return: Transaction, Race Condition, และการเลือก 403 vs 404 vs 409

นี่คือหัวข้อที่ซับซ้อนที่สุดของทั้งบท — การยืมหนังสือต้องรับมือกับ**สองปัญหาที่แยกกันโดยสิ้นเชิง**: (1) ปัญหา
เชิง concurrency (สำเนาหนังสือมีจำกัด หลาย request แข่งกันยืมพร้อมกันได้) และ (2) ปัญหาเชิง authorization
(ใครมีสิทธิ์คืนหนังสือเล่มไหน)

```rust
// src/routes/borrows.rs
use axum::{
    extract::{Path, State},
    http::StatusCode,
    Json,
};
use chrono::{Duration, Utc};

use crate::{
    auth::CurrentUser,
    error::AppError,
    models::{BorrowListResponse, BorrowResponse},
    state::AppState,
};

/// จำนวนวันที่ยืมได้ต่อครั้ง — ค่าคงที่ตายตัวสำหรับ capstone นี้ (ระบบจริงอาจตั้งต่อ category/role ได้)
const BORROW_PERIOD_DAYS: i64 = 14;

/// POST /api/v1/books/{id}/borrow — ยืมหนังสือ (ต้อง login)
///
/// **หัวใจของ endpoint นี้คือการป้องกัน race condition ตอน available_copies ลดฮวบต่ำกว่า 0**
/// เมื่อมีหลาย request ยืมหนังสือเล่มเดียวกันมาถึง "พร้อมกัน" จริง ๆ (ตาม Part 70 หัวข้อ transaction) —
/// วิธีที่ใช้คือ `UPDATE ... SET available_copies = available_copies - 1 WHERE id = $1 AND
/// available_copies > 0` ภายใน transaction เดียว: Postgres ล็อกแถวนั้นระหว่างทำ UPDATE ให้เอง
/// (row-level lock ที่เกิดจาก MVCC) request ที่มาทีหลังจะ "รอ" ให้ transaction แรกจบก่อนเสมอ
/// ไม่มีทางที่ available_copies จะติดลบ หรือสอง request "แข่งกันอ่านค่าเดิม" แล้วเขียนทับกันได้เลย
#[utoipa::path(
    post,
    path = "/api/v1/books/{id}/borrow",
    tag = "borrows",
    security(("bearer_auth" = [])),
    params(("id" = i64, Path, description = "book id ที่ต้องการยืม")),
    responses(
        (status = 201, description = "ยืมสำเร็จ", body = BorrowResponse),
        (status = 401, description = "ไม่ได้ login"),
        (status = 404, description = "ไม่พบหนังสือ id นี้"),
        (status = 409, description = "ไม่มีสำเนาว่างให้ยืม หรือคุณยืมเล่มนี้ค้างอยู่แล้ว"),
    ),
)]
pub async fn borrow_book(
    State(state): State<AppState>,
    current_user: CurrentUser,
    Path(id): Path<i64>,
) -> Result<(StatusCode, Json<BorrowResponse>), AppError> {
    let mut tx = state.db.begin().await?;

    // (1) ต้องมีหนังสือ id นี้อยู่จริงก่อน — แยกจาก "มีแต่ไม่มีสำเนาว่าง" เพราะเป็นสาเหตุคนละแบบ
    //     (404 ต่างจาก 409 ตามความหมายของ Part 61: หาไม่เจอ vs ขัดแย้งกับสถานะปัจจุบัน)
    let book_exists = sqlx::query_scalar!("SELECT EXISTS(SELECT 1 FROM books WHERE id = $1)", id)
        .fetch_one(&mut *tx)
        .await?
        .unwrap_or(false);
    if !book_exists {
        return Err(AppError::NotFound(format!("ไม่พบหนังสือ id {id}")));
    }

    // (2) ผู้ใช้คนนี้ยืมเล่มนี้ "ค้างอยู่" (ยังไม่คืน) หรือยัง — กันการกดยืมซ้ำสองครั้งติดกัน
    let already_borrowed = sqlx::query_scalar!(
        "SELECT EXISTS(SELECT 1 FROM borrows WHERE book_id = $1 AND user_id = $2 AND returned_at IS NULL)",
        id,
        current_user.user_id,
    )
    .fetch_one(&mut *tx)
    .await?
    .unwrap_or(false);
    if already_borrowed {
        return Err(AppError::Conflict(
            "คุณยืมหนังสือเล่มนี้ค้างอยู่แล้ว ต้องคืนก่อนยืมซ้ำ".to_string(),
        ));
    }

    // (3) *** จุดสำคัญที่สุด: atomic conditional decrement ป้องกัน race condition ***
    //     WHERE available_copies > 0 ทำให้ถ้ามีคนอื่นยืมไปพร้อมกันจนเหลือ 0 ก่อนถึงคิวเรา
    //     UPDATE นี้จะไม่ match แถวไหนเลย (0 rows affected) แทนที่จะเขียนค่าติดลบ
    let updated = sqlx::query_scalar!(
        "UPDATE books SET available_copies = available_copies - 1
         WHERE id = $1 AND available_copies > 0
         RETURNING available_copies",
        id,
    )
    .fetch_optional(&mut *tx)
    .await?;

    if updated.is_none() {
        return Err(AppError::Conflict(
            "ไม่มีสำเนาว่างให้ยืมในขณะนี้".to_string(),
        ));
    }

    let due_at = Utc::now() + Duration::days(BORROW_PERIOD_DAYS);
    let borrow = sqlx::query_as!(
        BorrowResponse,
        r#"
        INSERT INTO borrows (book_id, user_id, due_at)
        VALUES ($1, $2, $3)
        RETURNING id, book_id, user_id, borrowed_at, due_at, returned_at
        "#,
        id,
        current_user.user_id,
        due_at,
    )
    .fetch_one(&mut *tx)
    .await?;

    tx.commit().await?;

    tracing::info!(book_id = id, user_id = current_user.user_id, "ยืมหนังสือสำเร็จ");
    Ok((StatusCode::CREATED, Json(borrow)))
}

/// POST /api/v1/books/{id}/return — คืนหนังสือ (ต้อง login, ต้องเป็นคนที่ยืมเองเท่านั้น)
///
/// **การตัดสินใจ 403 vs 404 vs 409 ในฟังก์ชันนี้ (ต่อยอด Part 76 หัวข้อ 403-vs-404):**
/// - ไม่มีหนังสือ id นี้เลย -> 404 (ไม่มีอะไรให้ทำ)
/// - มีหนังสือ แต่ไม่มีใครยืมอยู่ตอนนี้เลย -> 409 Conflict (สถานะขัดกับ action "คืน" — เป็นข้อมูล
///   สาธารณะอยู่แล้วว่าหนังสือเล่มนี้ "ว่าง" ผ่าน available_copies ของ GET /books/{id} จึงไม่มีอะไรต้องปิดบัง)
/// - มีคนยืมอยู่ แต่เป็น "คนอื่น" ไม่ใช่ current_user -> 403 Forbidden (authenticate ผ่านแล้ว รู้ว่าเป็นใคร
///   แต่ไม่มีสิทธิ์คืนแทนคนอื่น — ไม่ใช้ 404 เพราะการรู้ว่า "หนังสือเล่มนี้มีคนยืมอยู่" ไม่ใช่ข้อมูลลับ
///   ต่างจากกรณีอย่างเช่นข้อมูล order ส่วนตัวของคนอื่นที่ Part 76 หัวข้อ 76.7 แนะนำให้ซ่อนด้วย 404)
#[utoipa::path(
    post,
    path = "/api/v1/books/{id}/return",
    tag = "borrows",
    security(("bearer_auth" = [])),
    params(("id" = i64, Path, description = "book id ที่ต้องการคืน")),
    responses(
        (status = 200, description = "คืนสำเร็จ", body = BorrowResponse),
        (status = 401, description = "ไม่ได้ login"),
        (status = 403, description = "ไม่ใช่ผู้ยืมเล่มนี้"),
        (status = 404, description = "ไม่พบหนังสือ id นี้"),
        (status = 409, description = "หนังสือเล่มนี้ไม่มีการยืมค้างอยู่"),
    ),
)]
pub async fn return_book(
    State(state): State<AppState>,
    current_user: CurrentUser,
    Path(id): Path<i64>,
) -> Result<Json<BorrowResponse>, AppError> {
    let mut tx = state.db.begin().await?;

    let book_exists = sqlx::query_scalar!("SELECT EXISTS(SELECT 1 FROM books WHERE id = $1)", id)
        .fetch_one(&mut *tx)
        .await?
        .unwrap_or(false);
    if !book_exists {
        return Err(AppError::NotFound(format!("ไม่พบหนังสือ id {id}")));
    }

    // หา active borrow (returned_at IS NULL) ของหนังสือเล่มนี้ "ไม่ว่าใครยืม" ก่อน เพื่อแยกกรณี
    // 409 (ไม่มีใครยืมเลย) ออกจาก 403 (มีคนยืมแต่ไม่ใช่เรา) ให้ถูกต้องตามที่ doc comment อธิบายไว้ข้างบน
    // FOR UPDATE ล็อกแถวนี้ไว้กันคนอื่นมา "คืน" แถวเดียวกันซ้ำพร้อมกัน
    let active_borrow = sqlx::query!(
        r#"SELECT id, user_id FROM borrows WHERE book_id = $1 AND returned_at IS NULL FOR UPDATE"#,
        id,
    )
    .fetch_optional(&mut *tx)
    .await?;

    let active_borrow = match active_borrow {
        None => {
            return Err(AppError::Conflict(
                "หนังสือเล่มนี้ไม่มีการยืมค้างอยู่".to_string(),
            ))
        }
        Some(row) => row,
    };

    if active_borrow.user_id != current_user.user_id {
        return Err(AppError::Forbidden(
            "คุณไม่ใช่ผู้ยืมหนังสือเล่มนี้ จึงคืนแทนไม่ได้".to_string(),
        ));
    }

    let borrow = sqlx::query_as!(
        BorrowResponse,
        r#"
        UPDATE borrows SET returned_at = now()
        WHERE id = $1
        RETURNING id, book_id, user_id, borrowed_at, due_at, returned_at
        "#,
        active_borrow.id,
    )
    .fetch_one(&mut *tx)
    .await?;

    sqlx::query!(
        "UPDATE books SET available_copies = available_copies + 1 WHERE id = $1",
        id,
    )
    .execute(&mut *tx)
    .await?;

    tx.commit().await?;

    tracing::info!(book_id = id, user_id = current_user.user_id, "คืนหนังสือสำเร็จ");
    Ok(Json(borrow))
}

/// GET /api/v1/me/borrowed — รายการหนังสือที่ current user ยืมค้างอยู่ (ยังไม่คืน)
#[utoipa::path(
    get,
    path = "/api/v1/me/borrowed",
    tag = "borrows",
    security(("bearer_auth" = [])),
    responses(
        (status = 200, description = "รายการที่ยืมค้างอยู่ของผู้ใช้ปัจจุบัน", body = BorrowListResponse),
        (status = 401, description = "ไม่ได้ login"),
    ),
)]
pub async fn my_borrowed_books(
    State(state): State<AppState>,
    current_user: CurrentUser,
) -> Result<Json<BorrowListResponse>, AppError> {
    let data = sqlx::query_as!(
        BorrowResponse,
        r#"
        SELECT id, book_id, user_id, borrowed_at, due_at, returned_at
        FROM borrows
        WHERE user_id = $1 AND returned_at IS NULL
        ORDER BY borrowed_at DESC
        "#,
        current_user.user_id,
    )
    .fetch_all(&state.db)
    .await?;

    Ok(Json(BorrowListResponse { data }))
}
```

**ทำไม early return ข้างในฟังก์ชันที่ถือ `tx: Transaction` ไว้ถึงปลอดภัย**: สังเกตว่าทั้ง `borrow_book` และ
`return_book` มี `return Err(...)` หลายจุด**ก่อน**ถึง `tx.commit().await?` — คำถามที่ควรผ่านหัวคือ "แล้ว
transaction ที่เปิดไว้ด้วย `state.db.begin()` จะเป็นยังไงถ้าฟังก์ชัน return ก่อนถึง `commit`?" คำตอบคือ
**ปลอดภัย 100%** เพราะ `sqlx::Transaction` implement `Drop` ที่ **rollback อัตโนมัติ**ถ้ามันถูก drop โดยไม่
เคย `commit()` มาก่อน (ตามหลักการ RAII ที่ Rust ใช้ตลอดทั้งหลักสูตร ตั้งแต่ Part 6 เรื่อง ownership) — ทุก
early return ในสองฟังก์ชันนี้จึงแปลว่า "ไม่มีการเปลี่ยนแปลงอะไรเกิดขึ้นกับฐานข้อมูลเลย" โดยอัตโนมัติ ไม่ต้อง
เขียน rollback ด้วยมือแม้แต่จุดเดียว

**พิสูจน์ flow เต็มด้วยเซิร์ฟเวอร์จริง** — สร้างผู้ใช้ `fah` และ `beau`, admin สร้างหนังสือ "Zero To Production
In Rust" ที่มีแค่ 1 สำเนา (id = 2):

```bash
$ curl -sS -i -X POST http://127.0.0.1:8092/api/v1/books/2/borrow -H "Authorization: Bearer <fah token>"
```
```
HTTP/1.1 201 Created
{"id":1,"book_id":2,"user_id":3,"borrowed_at":"2026-09-27T03:20:32.813156Z","due_at":"2026-10-11T03:20:32.815161Z","returned_at":null}
```

`beau` พยายามยืมเล่มเดียวกันต่อ — ไม่มีสำเนาว่างแล้ว (`available_copies` เหลือ 0):

```bash
$ curl -sS -i -X POST http://127.0.0.1:8092/api/v1/books/2/borrow -H "Authorization: Bearer <beau token>"
```
```
HTTP/1.1 409 Conflict
{"error":{"code":"CONFLICT","message":"ขัดแย้งกับสถานะปัจจุบัน: ไม่มีสำเนาว่างให้ยืมในขณะนี้"}}
```

`fah` พยายามยืมเล่มเดิมซ้ำ (ยังไม่คืน) — ชน unique partial index จากหัวข้อ 92.3:

```bash
$ curl -sS -i -X POST http://127.0.0.1:8092/api/v1/books/2/borrow -H "Authorization: Bearer <fah token>"
```
```
HTTP/1.1 409 Conflict
{"error":{"code":"CONFLICT","message":"ขัดแย้งกับสถานะปัจจุบัน: คุณยืมหนังสือเล่มนี้ค้างอยู่แล้ว ต้องคืนก่อนยืมซ้ำ"}}
```

`beau` พยายามคืนหนังสือที่ `fah` ยืมอยู่ — 403 ไม่ใช่ 404 ตามที่ doc comment อธิบายไว้:

```bash
$ curl -sS -i -X POST http://127.0.0.1:8092/api/v1/books/2/return -H "Authorization: Bearer <beau token>"
```
```
HTTP/1.1 403 Forbidden
{"error":{"code":"FORBIDDEN","message":"ไม่มีสิทธิ์ทำรายการนี้: คุณไม่ใช่ผู้ยืมหนังสือเล่มนี้ จึงคืนแทนไม่ได้"}}
```

`fah` คืนหนังสือสำเร็จ แล้วพยายามคืนซ้ำ (ไม่มี active borrow เหลือแล้ว) — 409 ไม่ใช่ 404 เพราะหนังสือยัง
มีอยู่จริง แค่ "ไม่มีการยืมค้างอยู่" ต่างหาก:

```bash
$ curl -sS -i -X POST http://127.0.0.1:8092/api/v1/books/2/return -H "Authorization: Bearer <fah token>"
```
```
HTTP/1.1 200 OK
{"id":1,"book_id":2,"user_id":3,"borrowed_at":"2026-09-27T03:20:32.813156Z","due_at":"2026-10-11T03:20:32.815161Z","returned_at":"2026-09-27T03:20:32.942684Z"}
```
```bash
$ curl -sS -i -X POST http://127.0.0.1:8092/api/v1/books/2/return -H "Authorization: Bearer <fah token>"
```
```
HTTP/1.1 409 Conflict
{"error":{"code":"CONFLICT","message":"ขัดแย้งกับสถานะปัจจุบัน: หนังสือเล่มนี้ไม่มีการยืมค้างอยู่"}}
```

ตอนนี้ `beau` ยืมได้แล้ว (สำเนาว่างกลับมาเพราะ `fah` คืนแล้ว):

```bash
$ curl -sS -i -X POST http://127.0.0.1:8092/api/v1/books/2/borrow -H "Authorization: Bearer <beau token>"
```
```
HTTP/1.1 201 Created
{"id":2,"book_id":2,"user_id":4,"borrowed_at":"2026-09-27T03:20:32.985897Z","due_at":"2026-10-11T03:20:32.986651Z","returned_at":null}
```

#### พิสูจน์ race condition จริง: ยิง 10 request พร้อมกันไปยังหนังสือที่มี 3 สำเนา

คำอธิบายเรื่อง "atomic conditional update ป้องกัน race condition" ในหัวข้อนี้จะเป็นแค่คำกล่าวลอย ๆ ถ้าไม่มี
การพิสูจน์จริง — สร้างหนังสือใหม่ id 4 ที่มี `total_copies = 3`, สมัครผู้ใช้ 10 คน (`racer1`...`racer10`),
แล้วยิง `POST /api/v1/books/4/borrow` **พร้อมกันจริง ๆ** ด้วย `curl` 10 ตัวที่รันเป็น background process
พร้อมกัน (`&` ใน bash แล้ว `wait`):

```bash
for i in $(seq 1 10); do
  TOKEN=$(cat /tmp/racer${i}_token.txt)
  (curl -sS -o /dev/null -w "%{http_code}" -X POST http://127.0.0.1:8092/api/v1/books/4/borrow \
    -H "Authorization: Bearer $TOKEN" > /tmp/race_status_$i.txt) &
done
wait
```

ผลลัพธ์จริงที่ได้จากแต่ละ request (สถานะ HTTP ของ `racer1`...`racer10` ตามลำดับที่ curl แต่ละตัวคืนกลับมา —
ลำดับที่ "ชนะ" การแข่งไปยืมสำเร็จไม่แน่นอน ขึ้นกับ scheduler ของ OS/tokio ตอนนั้น ซึ่งเป็นธรรมชาติของ
concurrency แบบนี้อยู่แล้ว):

```
racer1: 201
racer2: 201
racer3: 409
racer4: 409
racer5: 201
racer6: 409
racer7: 409
racer8: 409
racer9: 409
racer10: 409
```

```bash
$ grep -l 201 /tmp/race_status_*.txt | wc -l   # จำนวนคนที่ยืมสำเร็จ
3
$ grep -l 409 /tmp/race_status_*.txt | wc -l   # จำนวนคนที่โดน conflict
7
$ curl -sS "http://127.0.0.1:8092/api/v1/books/4" | python3 -m json.tool
```
```json
{
    "id": 4,
    "title": "Race Condition Test Book",
    "author": "Test",
    "isbn": "1112223334446",
    "category": "test",
    "total_copies": 3,
    "available_copies": 0,
    "created_at": "2026-09-27T03:20:59.985696Z"
}
```

**ผลลัพธ์ตรงตามที่ออกแบบไว้เป๊ะ**: หนังสือมี 3 สำเนา, ยิง 10 request พร้อมกันจริง, **ได้คนยืมสำเร็จ (`201`)
พอดี 3 คนเท่านั้น** อีก 7 คนได้ `409 Conflict`, และ `available_copies` ลงเหลือ **`0` พอดี ไม่ติดลบ** — นี่คือ
สิ่งที่การใช้ `UPDATE books SET available_copies = available_copies - 1 WHERE ... AND available_copies > 0`
ภายใน transaction ให้การันตี: ไม่ว่า 10 request จะมาถึงเซิร์ฟเวอร์ในเวลาไล่เลี่ยกันแค่ไหน Postgres จะประมวลผล
`UPDATE` ทีละตัวเสมอสำหรับแถวเดียวกัน (row-level lock ที่เกิดขึ้นเองจากการ `UPDATE`) ทำให้ request ที่ 4 เป็น
ต้นไปเห็น `available_copies` ที่ลดลงมาแล้วจาก request ก่อนหน้าเสมอ ไม่มีทางที่สอง request จะ "อ่านค่า 1 พร้อม
กัน แล้วลดเหลือ 0 พร้อมกันทั้งคู่" ได้เลย

**เทียบกับโค้ดที่ "ดูเหมือนถูก" แต่มี race condition จริง** (เพื่อให้เห็นว่าทำไมต้องเขียนแบบ `UPDATE ...
WHERE available_copies > 0` เท่านั้น ไม่ใช่แยกเป็นสองคำสั่ง):

```rust
// *** อย่าทำแบบนี้ — มี race condition จริง แม้จะอยู่ใน transaction ก็ตาม ***
let available: i32 = sqlx::query_scalar!("SELECT available_copies FROM books WHERE id = $1", id)
    .fetch_one(&mut *tx)
    .await?;

if available <= 0 {
    return Err(AppError::Conflict("ไม่มีสำเนาว่างให้ยืมในขณะนี้".to_string()));
}

// *** ช่วงเวลาระหว่างบรรทัดนี้กับบรรทัดข้างบน คือช่องโหว่ ***
// ถ้ามี transaction คู่ขนานอีกตัวอ่าน available_copies ค่าเดิม (เช่น = 1) ไปพร้อมกัน
// ก่อนที่ทั้งคู่จะมาถึง UPDATE ด้านล่าง ทั้งสอง transaction จะเห็นว่า "ยังมีสำเนาว่าง" ทั้งคู่
// แล้วต่างคน ต่างลดค่าจาก 1 เหลือ 0 — สุดท้ายมีคนยืมสำเร็จ "สองคน" ทั้งที่มีสำเนาว่างแค่ 1
sqlx::query!("UPDATE books SET available_copies = available_copies - 1 WHERE id = $1", id)
    .execute(&mut *tx)
    .await?;
```

ปัญหาของโค้ดนี้คือการแยก **"อ่าน" (`SELECT`) กับ "เขียน" (`UPDATE`) เป็นสองคำสั่งที่ไม่ atomic ต่อกัน** — แม้
ทั้งสองอยู่ใน transaction เดียวกัน (ซึ่งการันตีแค่ atomicity ของ transaction ทั้งก้อน ไม่ได้การันตีว่าไม่มี
transaction อื่นมาอ่าน/เขียนแถวเดียวกันระหว่างสองคำสั่งนี้ — เว้นแต่ใช้ isolation level ที่สูงกว่า default
`READ COMMITTED` ของ Postgres ซึ่งมีข้อแลกเปลี่ยนด้าน performance) — วิธีที่ปลอดภัยและง่ายที่สุดคือรวม "เช็ค"
กับ "เขียน" ให้เป็น**คำสั่งเดียว** (`UPDATE ... WHERE available_copies > 0`) แบบที่โค้ดจริงในบทนี้ทำ ซึ่ง
ใช้ atomicity ที่ Postgres การันตีให้กับทุก `UPDATE` เดี่ยว ๆ อยู่แล้วโดยไม่ต้องพึ่ง isolation level พิเศษเลย

### 92.10 API Documentation: utoipa + Swagger UI (ต่อยอด Part 85)

```rust
// src/docs.rs
use utoipa::{
    openapi::security::{HttpAuthScheme, HttpBuilder, SecurityScheme},
    Modify, OpenApi,
};

use crate::{
    error::FieldError,
    models::{
        BookListResponse, BookResponse, BorrowListResponse, BorrowResponse, CreateBookRequest,
        LoginRequest, LoginResponse, PaginationMeta, RegisterRequest, UserResponse,
    },
};

struct SecurityAddon;

impl Modify for SecurityAddon {
    fn modify(&self, openapi: &mut utoipa::openapi::OpenApi) {
        if let Some(components) = openapi.components.as_mut() {
            components.add_security_scheme(
                "bearer_auth",
                SecurityScheme::Http(
                    HttpBuilder::new()
                        .scheme(HttpAuthScheme::Bearer)
                        .bearer_format("JWT")
                        .build(),
                ),
            );
        }
    }
}

#[derive(OpenApi)]
#[openapi(
    paths(
        crate::routes::health::health_check,
        crate::routes::auth::register,
        crate::routes::auth::login,
        crate::routes::auth::me,
        crate::routes::books::list_books,
        crate::routes::books::get_book,
        crate::routes::books::create_book,
        crate::routes::borrows::borrow_book,
        crate::routes::borrows::return_book,
        crate::routes::borrows::my_borrowed_books,
    ),
    components(schemas(
        UserResponse, RegisterRequest, LoginRequest, LoginResponse,
        BookResponse, CreateBookRequest, BookListResponse, PaginationMeta,
        BorrowResponse, BorrowListResponse, FieldError,
    )),
    tags(
        (name = "health", description = "health check endpoint"),
        (name = "auth", description = "สมัครสมาชิก / login / ข้อมูลผู้ใช้ปัจจุบัน"),
        (name = "books", description = "ค้นหา/ดูรายละเอียด/สร้างหนังสือ (สร้างได้เฉพาะ admin)"),
        (name = "borrows", description = "ยืม/คืน/ดูรายการที่ยืมค้างอยู่"),
    ),
    modifiers(&SecurityAddon),
)]
pub struct ApiDoc;
```

**จุดที่ต่างจาก Part 85 ที่สำคัญที่สุด — การอ้าง handler ข้าม module**: Part 85 สอนตัวอย่างที่ handler กับ
`#[derive(OpenApi)]` อยู่ไฟล์เดียวกัน ทำให้ `paths(get_book)` เขียนชื่อฟังก์ชันตรง ๆ ได้เลย แต่บทนี้แยก
`routes::books::get_book` ออกจาก `docs.rs` โดยสิ้นเชิง (ตามโครงสร้าง module ในหัวข้อ 92.2) — ถ้าเขียน
`paths(get_book)` เฉย ๆ (แม้จะ `use crate::routes::books::get_book;` มาก่อนก็ตาม) จะเจอ compile error ทันที
เพราะ macro `#[utoipa::path(...)]` สร้าง marker type ที่ชื่อ `__path_get_book` ไว้ **ในโมดูลเดียวกับฟังก์ชัน**
(ไม่ใช่ในโมดูลที่ import ฟังก์ชันไปใช้) และ `#[derive(OpenApi)]` จะไปหา marker type ตัวนี้จาก**path ของชื่อที่
เขียนใน `paths(...)` เป๊ะ** — วิธีแก้คือเขียน **path แบบเต็ม** (`crate::routes::books::get_book`) ในทุกจุดของ
`paths(...)` เพื่อให้ macro คำนวณ path ของ `__path_get_book` ให้ตรงกับตำแหน่งจริงของมัน (ดูรายละเอียด error
จริงที่เจอตอนพัฒนาบทนี้ในหัวข้อ "กับดักที่พบบ่อย" ข้อ 3 ท้ายบท)

ต่อไปนี้คือ `lib.rs` ที่ประกอบทุกอย่างเข้าด้วยกัน — Router, CORS, Swagger UI, TraceLayer:

```rust
// src/lib.rs
pub mod auth;
pub mod docs;
pub mod error;
pub mod models;
pub mod routes;
pub mod state;

use axum::http::{header, HeaderValue, Method};
use axum::Router;
use tower_http::cors::CorsLayer;
use tower_http::trace::TraceLayer;
use utoipa::OpenApi;
use utoipa_swagger_ui::SwaggerUi;

use state::AppState;

/// สร้าง Router เต็มรูปแบบของทั้งแอป — แยกออกจาก main() เพื่อให้ integration test (tests/api_tests.rs)
/// เรียก build_app(state) ตรง ๆ ได้โดยไม่ต้องเปิด TCP listener จริง (ตาม pattern Part 32/33)
///
/// `frontend_origin`: origin ของ frontend dev server ที่จะอนุญาตให้เรียก API นี้ผ่าน CORS
/// (Part 93 จะรัน frontend dev server แยก origin จาก backend นี้เสมอ — ต่อยอด Part 65 หัวข้อ CorsLayer)
pub fn build_app(state: AppState, frontend_origin: &str) -> Router {
    let cors = CorsLayer::new()
        .allow_origin(
            frontend_origin
                .parse::<HeaderValue>()
                .expect("frontend_origin ต้องเป็น URL ที่ valid"),
        )
        // เฉพาะ method ที่ endpoint จริงในบทนี้ใช้ (GET/POST) — Part 93/94 ที่จะเพิ่ม endpoint
        // ใหม่ (เช่น PATCH/DELETE) ต้องเติม method นั้นเข้ามาที่นี่ด้วย ไม่อย่างนั้น browser จะ
        // บล็อก request จริงตอน preflight แม้ route จะรับ method นั้นอยู่แล้วก็ตาม
        .allow_methods([Method::GET, Method::POST, Method::OPTIONS])
        .allow_headers([header::CONTENT_TYPE, header::AUTHORIZATION]);

    routes::build_router()
        .merge(SwaggerUi::new("/swagger-ui").url("/api-docs/openapi.json", docs::ApiDoc::openapi()))
        .layer(cors)
        .layer(TraceLayer::new_for_http())
        .with_state(state)
}
```

```rust
// src/routes/health.rs
use axum::Json;
use serde_json::{json, Value};

#[utoipa::path(
    get,
    path = "/health",
    tag = "health",
    responses((status = 200, description = "เซิร์ฟเวอร์พร้อมใช้งาน", body = serde_json::Value)),
)]
pub async fn health_check() -> Json<Value> {
    Json(json!({ "status": "ok" }))
}
```

```rust
// src/routes/mod.rs
pub mod auth;
pub mod books;
pub mod borrows;
pub mod health;

use axum::{
    routing::{get, post},
    Router,
};

use crate::state::AppState;

pub fn build_router() -> Router<AppState> {
    Router::new()
        .route("/health", get(health::health_check))
        .route("/api/v1/auth/register", post(auth::register))
        .route("/api/v1/auth/login", post(auth::login))
        .route("/api/v1/me", get(auth::me))
        .route("/api/v1/me/borrowed", get(borrows::my_borrowed_books))
        .route("/api/v1/books", get(books::list_books).post(books::create_book))
        .route("/api/v1/books/{id}", get(books::get_book))
        .route("/api/v1/books/{id}/borrow", post(borrows::borrow_book))
        .route("/api/v1/books/{id}/return", post(borrows::return_book))
}
```

```rust
// src/main.rs
use std::sync::Arc;
use std::time::Duration;

use sqlx::postgres::PgPoolOptions;

use library_api::{build_app, state::AppState};

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    tracing_subscriber::fmt()
        .with_env_filter(tracing_subscriber::EnvFilter::from_default_env())
        .init();

    let database_url = std::env::var("DATABASE_URL")
        .unwrap_or_else(|_| "postgres://postgres:postgres@127.0.0.1:5432/library_api_dev".to_string());
    let jwt_secret = std::env::var("JWT_SECRET")
        .unwrap_or_else(|_| "dev-only-secret-change-me-in-production".to_string());
    let frontend_origin =
        std::env::var("FRONTEND_ORIGIN").unwrap_or_else(|_| "http://127.0.0.1:5173".to_string());
    let bind_addr = std::env::var("BIND_ADDR").unwrap_or_else(|_| "127.0.0.1:8092".to_string());

    let db = PgPoolOptions::new()
        .max_connections(10)
        .acquire_timeout(Duration::from_secs(5))
        .connect(&database_url)
        .await?;

    sqlx::migrate!("./migrations").run(&db).await?;

    let state = AppState {
        db,
        jwt_secret: Arc::from(jwt_secret),
    };

    let app = build_app(state, &frontend_origin);

    let listener = tokio::net::TcpListener::bind(&bind_addr).await?;
    tracing::info!("library_api ฟังอยู่ที่ http://{bind_addr}");
    tracing::info!("Swagger UI: http://{bind_addr}/swagger-ui");
    axum::serve(listener, app).await?;

    Ok(())
}
```

**พิสูจน์ Swagger UI และ OpenAPI JSON จริง**:

```bash
$ curl -sS http://127.0.0.1:8092/api-docs/openapi.json | head -c 400
{"openapi":"3.1.0","info":{"title":"library_api","description":"","license":{"name":""},"version":"0.1.0"},"paths":{"/api/v1/auth/login":{"post":{"tags":["auth"],"summary":"POST /api/v1/auth/login — ตรวจสอบ username/password แล้วออก JWT access token","operationId":"login", ...

$ curl -sS -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8092/swagger-ui/
200
```

Swagger UI ที่เปิดผ่านเบราว์เซอร์ที่ `http://127.0.0.1:8092/swagger-ui/` แสดง endpoint ทั้ง 10 ตัวจัดกลุ่มตาม
tag (`health`, `auth`, `books`, `borrows`) พร้อมปุ่ม "Authorize" ที่ให้กรอก JWT token ครั้งเดียวแล้วทดสอบ
endpoint ที่ต้อง auth ได้ทุกตัวจากหน้าเว็บโดยตรง (มาจาก `SecurityAddon`/`bearer_auth` scheme ที่ผูกไว้ในหัวข้อ
นี้) — นี่คือเอกสารตัวเดียวกันที่ Part 93 (frontend) จะใช้อ้างอิง shape ของ request/response ทุก endpoint

### 92.11 Testing: Integration Test Suite เต็มรูปแบบ (ต่อยอด Part 32/33/71)

ตาม pattern จาก Part 71 หัวข้อ 71.10 integration test ของแอปที่ใช้ SQLx ใช้ `#[sqlx::test]` — macro นี้สร้าง
ฐานข้อมูลทดสอบใหม่ที่แยกจากกันสนิทให้แต่ละเทส (จาก template database ที่รัน migration ไว้ล่วงหน้าครั้งเดียว)
แล้วส่ง `PgPool` ที่ต่อฐานข้อมูลนั้นเข้ามาเป็นพารามิเตอร์ — ทำให้เทสแต่ละตัวรันแบบ isolate สมบูรณ์ ไม่ชนกัน
แม้จะรันพร้อมกันหลายเทสก็ตาม:

```rust
// tests/api_tests.rs
use std::sync::Arc;

use axum::{
    body::{to_bytes, Body},
    http::{Request, StatusCode},
};
use serde_json::{json, Value};
use sqlx::PgPool;
use tower::ServiceExt; // .oneshot()

use library_api::{build_app, state::AppState};

const TEST_JWT_SECRET: &str = "test-secret-not-for-production";
const FRONTEND_ORIGIN: &str = "http://127.0.0.1:5173";

fn app(pool: PgPool) -> axum::Router {
    let state = AppState {
        db: pool,
        jwt_secret: Arc::from(TEST_JWT_SECRET),
    };
    build_app(state, FRONTEND_ORIGIN)
}

async fn body_json(response: axum::response::Response) -> Value {
    let bytes = to_bytes(response.into_body(), usize::MAX).await.unwrap();
    serde_json::from_slice(&bytes).unwrap_or(Value::Null)
}

async fn register(app: &axum::Router, username: &str, email: &str, password: &str) -> Value {
    let req = Request::builder()
        .method("POST")
        .uri("/api/v1/auth/register")
        .header("content-type", "application/json")
        .body(Body::from(
            json!({ "username": username, "email": email, "password": password }).to_string(),
        ))
        .unwrap();
    let res = app.clone().oneshot(req).await.unwrap();
    assert_eq!(res.status(), StatusCode::CREATED, "register ควรได้ 201");
    body_json(res).await
}

async fn login(app: &axum::Router, username: &str, password: &str) -> Value {
    let req = Request::builder()
        .method("POST")
        .uri("/api/v1/auth/login")
        .header("content-type", "application/json")
        .body(Body::from(json!({ "username": username, "password": password }).to_string()))
        .unwrap();
    let res = app.clone().oneshot(req).await.unwrap();
    assert_eq!(res.status(), StatusCode::OK, "login ควรได้ 200");
    body_json(res).await
}

/// flow เต็มรูปแบบ: register -> login -> admin สร้างหนังสือ -> member ยืม -> member คืน
#[sqlx::test]
async fn test_register_login_borrow_return_flow(pool: PgPool) {
    let app = app(pool.clone());

    // 1) register member ธรรมดา
    let user = register(&app, "nan", "nan@example.com", "password123").await;
    assert_eq!(user["role"], "member");

    // 2) login ได้ access_token
    let login_body = login(&app, "nan", "password123").await;
    let token = login_body["access_token"].as_str().unwrap().to_string();
    assert_eq!(login_body["token_type"], "Bearer");

    // 3) สร้าง admin: ไม่มี endpoint สมัครเป็น admin ได้เองเด็ดขาด (register เขียน role='member'
    //    ตายตัวเสมอ) วิธีเดียวที่ตั้งใจไว้คือ register แบบธรรมดาก่อน แล้ว promote role ผ่าน DB ตรง ๆ
    register(&app, "admin2", "admin2@example.com", "adminpass1").await;
    sqlx::query!("UPDATE users SET role = 'admin' WHERE username = 'admin2'")
        .execute(&pool)
        .await
        .unwrap();
    let admin_login = login(&app, "admin2", "adminpass1").await;
    let admin_token = admin_login["access_token"].as_str().unwrap().to_string();

    // 4) admin สร้างหนังสือ
    let create_req = Request::builder()
        .method("POST")
        .uri("/api/v1/books")
        .header("content-type", "application/json")
        .header("authorization", format!("Bearer {admin_token}"))
        .body(Body::from(
            json!({
                "title": "The Rust Programming Language",
                "author": "Steve Klabnik",
                "isbn": "9781593278281",
                "category": "programming",
                "total_copies": 2
            })
            .to_string(),
        ))
        .unwrap();
    let res = app.clone().oneshot(create_req).await.unwrap();
    assert_eq!(res.status(), StatusCode::CREATED);
    let book = body_json(res).await;
    let book_id = book["id"].as_i64().unwrap();
    assert_eq!(book["available_copies"], 2);

    // 5) member (nan) ยืมหนังสือเล่มนี้
    let borrow_req = Request::builder()
        .method("POST")
        .uri(format!("/api/v1/books/{book_id}/borrow"))
        .header("authorization", format!("Bearer {token}"))
        .body(Body::empty())
        .unwrap();
    let res = app.clone().oneshot(borrow_req).await.unwrap();
    assert_eq!(res.status(), StatusCode::CREATED, "borrow ควรสำเร็จ");
    let borrow = body_json(res).await;
    assert!(borrow["returned_at"].is_null());

    // available_copies ต้องลดลงเหลือ 1
    let get_req = Request::builder()
        .uri(format!("/api/v1/books/{book_id}"))
        .body(Body::empty())
        .unwrap();
    let res = app.clone().oneshot(get_req).await.unwrap();
    let book_after_borrow = body_json(res).await;
    assert_eq!(book_after_borrow["available_copies"], 1);

    // 6) me/borrowed ต้องมีเล่มนี้อยู่ 1 รายการ
    let mine_req = Request::builder()
        .uri("/api/v1/me/borrowed")
        .header("authorization", format!("Bearer {token}"))
        .body(Body::empty())
        .unwrap();
    let res = app.clone().oneshot(mine_req).await.unwrap();
    let mine = body_json(res).await;
    assert_eq!(mine["data"].as_array().unwrap().len(), 1);

    // 7) nan คืนหนังสือ
    let return_req = Request::builder()
        .method("POST")
        .uri(format!("/api/v1/books/{book_id}/return"))
        .header("authorization", format!("Bearer {token}"))
        .body(Body::empty())
        .unwrap();
    let res = app.clone().oneshot(return_req).await.unwrap();
    assert_eq!(res.status(), StatusCode::OK, "return ควรสำเร็จ");
    let returned = body_json(res).await;
    assert!(!returned["returned_at"].is_null());

    // available_copies ต้องกลับมาเป็น 2
    let get_req = Request::builder()
        .uri(format!("/api/v1/books/{book_id}"))
        .body(Body::empty())
        .unwrap();
    let res = app.clone().oneshot(get_req).await.unwrap();
    let book_after_return = body_json(res).await;
    assert_eq!(book_after_return["available_copies"], 2);
}

/// non-admin พยายามสร้างหนังสือ -> ต้องโดน 403 Forbidden ไม่ใช่ 401
#[sqlx::test]
async fn test_create_book_forbidden_for_non_admin(pool: PgPool) {
    let app = app(pool);
    register(&app, "member1", "member1@example.com", "password123").await;
    let login_body = login(&app, "member1", "password123").await;
    let token = login_body["access_token"].as_str().unwrap();

    let req = Request::builder()
        .method("POST")
        .uri("/api/v1/books")
        .header("content-type", "application/json")
        .header("authorization", format!("Bearer {token}"))
        .body(Body::from(
            json!({
                "title": "x", "author": "y", "isbn": "1234567890123",
                "category": "z", "total_copies": 1
            })
            .to_string(),
        ))
        .unwrap();
    let res = app.clone().oneshot(req).await.unwrap();
    assert_eq!(res.status(), StatusCode::FORBIDDEN);
    let body = body_json(res).await;
    assert_eq!(body["error"]["code"], "FORBIDDEN");
}

/// ยืมหนังสือโดยไม่แนบ token -> 401 Unauthorized
#[sqlx::test]
async fn test_borrow_without_token_is_unauthorized(pool: PgPool) {
    let app = app(pool.clone());
    sqlx::query!(
        "INSERT INTO books (title, author, isbn, category, total_copies, available_copies)
         VALUES ('t', 'a', '1112223334445', 'c', 1, 1)"
    )
    .execute(&pool)
    .await
    .unwrap();

    let req = Request::builder()
        .method("POST")
        .uri("/api/v1/books/1/borrow")
        .body(Body::empty())
        .unwrap();
    let res = app.clone().oneshot(req).await.unwrap();
    assert_eq!(res.status(), StatusCode::UNAUTHORIZED);
    let body = body_json(res).await;
    assert_eq!(body["error"]["code"], "UNAUTHORIZED");
}

/// หนังสือมีสำเนาว่าง 1 เล่ม — user คนแรกยืมสำเร็จ, user คนที่สองยืมต่อ -> ต้องโดน 409 Conflict
#[sqlx::test]
async fn test_borrow_conflict_when_no_copies_available(pool: PgPool) {
    let app = app(pool.clone());

    register(&app, "alice", "alice@example.com", "password123").await;
    register(&app, "bob", "bob@example.com", "password123").await;
    let alice_token = login(&app, "alice", "password123").await["access_token"]
        .as_str()
        .unwrap()
        .to_string();
    let bob_token = login(&app, "bob", "password123").await["access_token"]
        .as_str()
        .unwrap()
        .to_string();

    sqlx::query!(
        "INSERT INTO books (id, title, author, isbn, category, total_copies, available_copies)
         VALUES (100, 'Only One Copy', 'author', '9999999999999', 'rare', 1, 1)"
    )
    .execute(&pool)
    .await
    .unwrap();

    // alice ยืมสำเร็จ (เหลือ 0)
    let req = Request::builder()
        .method("POST")
        .uri("/api/v1/books/100/borrow")
        .header("authorization", format!("Bearer {alice_token}"))
        .body(Body::empty())
        .unwrap();
    let res = app.clone().oneshot(req).await.unwrap();
    assert_eq!(res.status(), StatusCode::CREATED);

    // bob ยืมต่อ -> ไม่มีสำเนาว่างแล้ว -> 409
    let req = Request::builder()
        .method("POST")
        .uri("/api/v1/books/100/borrow")
        .header("authorization", format!("Bearer {bob_token}"))
        .body(Body::empty())
        .unwrap();
    let res = app.clone().oneshot(req).await.unwrap();
    assert_eq!(res.status(), StatusCode::CONFLICT);
    let body = body_json(res).await;
    assert_eq!(body["error"]["code"], "CONFLICT");

    // bob พยายามคืนหนังสือที่ alice ยืมอยู่ -> 403 Forbidden (ไม่ใช่ผู้ยืม)
    let req = Request::builder()
        .method("POST")
        .uri("/api/v1/books/100/return")
        .header("authorization", format!("Bearer {bob_token}"))
        .body(Body::empty())
        .unwrap();
    let res = app.clone().oneshot(req).await.unwrap();
    assert_eq!(res.status(), StatusCode::FORBIDDEN);
}

/// GET /health ไม่ต้อง auth และตอบ 200 เสมอ
#[sqlx::test]
async fn test_health_check(pool: PgPool) {
    let app = app(pool);
    let req = Request::builder().uri("/health").body(Body::empty()).unwrap();
    let res = app.oneshot(req).await.unwrap();
    assert_eq!(res.status(), StatusCode::OK);
}
```

**อธิบายจุดสำคัญของ test suite นี้**:

- **`app: axum::Router` (ไม่ใช่ `Router<AppState>`)** — เพราะ `build_app` เรียก `.with_state(state)` ให้
  เรียบร้อยแล้ว `Router` ที่ได้กลับมาจึงเป็น `Router<()>` (Axum เรียกสั้น ๆ ว่า `Router`) ที่พร้อมรับ request
  ได้ทันทีโดยไม่ต้องมี state เพิ่มเติมจากใครอีก — สิ่งนี้คือประโยชน์ตรงของการแยก `build_app` ออกจาก `main.rs`
  ตามที่อธิบายไว้ในหัวข้อ 92.2
- **`.oneshot(req)`** จาก `tower::ServiceExt` — ส่ง request ตัวเดียวเข้า `Router` โดยตรงในหน่วยความจำ **ไม่มี
  การเปิด TCP socket จริงเลย** ต่างจากการทดสอบด้วย `curl` ที่ต้องมีเซิร์ฟเวอร์รันอยู่จริง — นี่คือรูปแบบ
  integration test ที่เร็วกว่าและเสถียรกว่า (ไม่มีปัญหาเรื่อง port ชนกัน) ตามที่ Part 33 แนะนำไว้สำหรับเทส
  ระดับ HTTP handler โดยตรง
- **`#[sqlx::test]` สร้างฐานข้อมูลแยกให้ทุกเทส** — ทำให้ `test_borrow_conflict_when_no_copies_available` ที่
  `INSERT` หนังสือด้วย `id = 100` ตรง ๆ ไม่ชนกับข้อมูลของเทสอื่นเลย แต่ละเทสเห็นฐานข้อมูลที่ "สะอาด" (มีแค่
  schema จาก migration แต่ไม่มีข้อมูล) เสมอไม่ว่าจะรันเทสไหนก่อนหรือหลัง — ทั้งหมดรันพร้อมกันได้อย่างปลอดภัย

รันจริงทั้ง 5 เทส:

```bash
$ cargo test
   Compiling library_api v0.1.0
    Finished `test` profile [unoptimized + debuginfo] target(s) in 1.76s
     Running unittests src/lib.rs (target/debug/deps/library_api-...)

running 0 tests
test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running tests/api_tests.rs (target/debug/deps/api_tests-...)

running 5 tests
test test_health_check ... ok
test test_borrow_without_token_is_unauthorized ... ok
test test_create_book_forbidden_for_non_admin ... ok
test test_borrow_conflict_when_no_copies_available ... ok
test test_register_login_borrow_return_flow ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 2.57s

   Doc-tests library_api

running 0 tests
test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**ทั้ง 5 เทสผ่านหมด** ครอบคลุม flow หลักที่ Part 94 (deployment/CI) จะใช้เป็น regression test ก่อน deploy
ทุกครั้ง: register→login→borrow→return แบบสมบูรณ์, admin สร้างหนังสือ, unauthorized borrow ถูกปฏิเสธ, borrow
ตอนไม่มีสำเนาว่างถูกปฏิเสธด้วย 409 (พร้อมพิสูจน์ 403 บน return ที่ไม่ใช่ผู้ยืมไปในตัว), และ health check

### 92.12 รันเซิร์ฟเวอร์จริง, CORS, และค่า config สำหรับ Local Development

เซิร์ฟเวอร์ของบทนี้ **รันที่ `http://127.0.0.1:8092` เป็นค่าเริ่มต้น** (ปรับได้ผ่าน environment variable
`BIND_ADDR`) — พอร์ต `8092` เลือกให้ตรงกับเลข Part เพื่อให้จำง่ายและไม่ชนกับพอร์ตที่บทอื่น ๆ ในหลักสูตรใช้ (Part
63/65/85 ใช้ `3xxx`, Part 76 ใช้ `3177` ฯลฯ) — **Part 93 (frontend) และ Part 94 (deployment) ต้องใช้ค่านี้
เป๊ะ** เว้นแต่จะมีการแก้ไขบทนี้อย่างชัดเจน

```bash
$ export DATABASE_URL="postgres://postgres:postgres@127.0.0.1:5432/library_api_dev"
$ export JWT_SECRET="dev-only-secret-change-me-in-production"
$ export FRONTEND_ORIGIN="http://127.0.0.1:5173"
$ export BIND_ADDR="127.0.0.1:8092"
$ cargo run
```
```
2026-09-27T03:19:02.632980Z  INFO library_api: library_api ฟังอยู่ที่ http://127.0.0.1:8092
2026-09-27T03:19:02.633009Z  INFO library_api: Swagger UI: http://127.0.0.1:8092/swagger-ui
```

**ตัวแปร environment ทั้งสี่ตัว** (ทุกตัวมีค่า default ที่ใช้งานได้ทันทีถ้าไม่ตั้งไว้ — สังเกตจาก `main.rs`
ในหัวข้อ 92.10):

| ตัวแปร | ค่า default | ความหมาย |
|---|---|---|
| `DATABASE_URL` | `postgres://postgres:postgres@127.0.0.1:5432/library_api_dev` | connection string ของ PostgreSQL |
| `JWT_SECRET` | `dev-only-secret-change-me-in-production` | secret สำหรับเซ็น/ตรวจ JWT (HS256) — **ต้องเปลี่ยนใน production เสมอ** |
| `FRONTEND_ORIGIN` | `http://127.0.0.1:5173` | origin ที่ `CorsLayer` อนุญาตให้เรียก API นี้ |
| `BIND_ADDR` | `127.0.0.1:8092` | address:port ที่เซิร์ฟเวอร์ฟัง |

**เรื่อง CORS ที่ Part 93 ต้องรู้**: `FRONTEND_ORIGIN` ค่า default คือ `http://127.0.0.1:5173` — **ตัวเลข 5173
นี้คือพอร์ต default ของ Vite dev server** (เครื่องมือ build ที่ frontend framework ยอดนิยมของ Rust/WASM หลาย
ตัวใช้ร่วมกับ `wasm-pack`/`trunk`) เลือกไว้ล่วงหน้าเพื่อให้ Part 93 เสียบเข้ามาได้ทันทีโดยไม่ต้องแก้ค่านี้ —
ถ้า frontend dev server ของ Part 93 ใช้พอร์ตอื่น ต้องตั้ง `FRONTEND_ORIGIN` ให้ตรงก่อน backend จะอนุญาต CORS
ให้เรียกได้ พิสูจน์ preflight request จริง:

```bash
$ curl -sS -i -X OPTIONS http://127.0.0.1:8092/api/v1/books \
  -H "Origin: http://127.0.0.1:5173" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: authorization, content-type"
```
```
HTTP/1.1 200 OK
access-control-allow-methods: GET,POST,OPTIONS
access-control-allow-headers: content-type,authorization
access-control-allow-origin: http://127.0.0.1:5173
allow: GET,HEAD,POST
content-length: 0
```

`CorsLayer` ตอบ preflight `OPTIONS` request จบในตัวเอง (ตามที่ Part 65 หัวข้อ CorsLayer พิสูจน์ไว้แล้วว่า
`OPTIONS` ถูกสกัดไว้ตั้งแต่ชั้น middleware ไม่ลงไปถึง handler จริงเลย) พร้อม header สามตัวที่ browser ต้องเห็น
ก่อนจะยอมให้ JavaScript ของ frontend ยิง request จริงตามมา: `access-control-allow-origin` (origin ที่
อนุญาต), `access-control-allow-methods` (method ที่อนุญาต — จำกัดแค่ `GET`/`POST`/`OPTIONS` ตามที่ endpoint
จริงในบทนี้ใช้), และ `access-control-allow-headers` (header ที่ frontend ส่งมาได้ — `content-type` สำหรับ
body แบบ JSON และ `authorization` สำหรับแนบ JWT)

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. `jsonwebtoken` panic ตอน runtime เพราะไม่ได้เลือก crypto backend

เหมือนที่ Part 74 หัวข้อ 74.3 เตือนไว้ — ถ้า `cargo add jsonwebtoken` โดยไม่เปิด feature ให้ถูกต้อง จะเจอ
panic ทันทีตอนเรียก `encode`/`decode` ครั้งแรก (ไม่ใช่ compile error):

```
thread 'main' panicked at .../jsonwebtoken-11.1.0/src/crypto/mod.rs:124:40:

Could not automatically determine the process-level CryptoProvider from jsonwebtoken crate features.
Call CryptoProvider::install_default() before this point to select a provider manually, or make sure
exactly one of the 'rust_crypto' and 'aws_lc_rs' features is enabled.
```

**วิธีแก้**: เปิด feature `rust_crypto` ตั้งแต่ตอนเพิ่ม dependency (`cargo add jsonwebtoken --features
rust_crypto` หรือระบุใน `Cargo.toml` ตรง ๆ ตามที่บทนี้ทำไว้แล้วในหัวข้อ 92 (หมายเหตุการตรวจสอบเนื้อหา)) —
กับดักนี้ร้ายกาจเพราะ `cargo build` **ผ่านสนิท** ไม่มีคำเตือนอะไรเลย จนกว่าจะมีคนเรียก endpoint ที่ใช้ JWT
จริงตอน runtime

### 2. `sqlx::QueryBuilder` เปลี่ยน signature ใน sqlx 0.9 — ไม่มี lifetime parameter อีกแล้ว

ระหว่างพัฒนาโค้ดในหัวข้อ 92.8 (list_books ที่ใช้ `QueryBuilder` สำหรับ filter แบบ dynamic) การเขียนตาม
รูปแบบเก่าที่คุ้นเคยจาก sqlx เวอร์ชันก่อนหน้า (`QueryBuilder<'a, Postgres>`) ทำให้เจอ compile error ทันที:

```
error[E0107]: struct takes 0 lifetime arguments but 1 lifetime argument was supplied
  --> src/routes/books.rs:18:35
   |
18 | fn push_book_filters<'a>(qb: &mut QueryBuilder<'a, Postgres>, params: &'a ListBooksQuery) {
   |                                   ^^^^^^^^^^^^ -- help: remove the lifetime argument
   |
note: struct defined here, with 0 lifetime parameters
  --> .../sqlx-core-0.9.0/src/query_builder.rs:28:12
   |
28 | pub struct QueryBuilder<DB>
   |            ^^^^^^^^^^^^
```

**วิธีแก้**: sqlx 0.9 เปลี่ยน `QueryBuilder` ให้เก็บ SQL string ไว้ใน `Arc<String>` ภายในแทนการ borrow แบบ
เดิม จึงไม่ต้องมี lifetime parameter อีกต่อไป — เขียนเป็น `QueryBuilder<Postgres>` เฉย ๆ (ไม่มี `'a`) และค่า
ที่ผ่าน `.push_bind(...)` ต้องเป็น**ค่าที่ owned** (เช่น `category.clone()` แทน `category` ที่เป็น `&String`)
เพราะ `push_bind<'t, T>` รับ `T: Encode<'t, DB> + Type<DB>` แบบ by-value ไม่ใช่ by-reference — นี่คือตัวอย่าง
ที่ดีว่าทำไมต้องอ่าน changelog/docs ของเวอร์ชันที่ใช้จริงเสมอ แทนที่จะเชื่อความจำจากเวอร์ชันเก่า แม้แต่ crate
ที่คุ้นเคยมากอย่าง `sqlx` ก็เปลี่ยน public API ข้าม major version ได้

### 3. `#[derive(OpenApi)]` หา `__path_<handler>` ไม่เจอ เมื่อแยก handler ข้าม module

นี่คือกับดักที่เกิดขึ้นจริงระหว่างพัฒนาหัวข้อ 92.10 (docs.rs) — ตอนเขียน `paths(register, login, me, ...)`
โดย `use` handler function เข้ามาก่อน (แบบเดียวกับตัวอย่างไฟล์เดียวของ Part 85):

```rust
use crate::routes::{auth::{login, me, register}, /* ... */};

#[derive(OpenApi)]
#[openapi(paths(register, login, me, /* ... */))]
pub struct ApiDoc;
```

เจอ compile error ทันที (ตัดมาบางส่วน):

```
error[E0433]: failed to resolve: use of unresolved module or unlinked crate `__path_register`
error[E0425]: cannot find type `__path_register` in this scope
```

**สาเหตุ**: `#[utoipa::path(...)]` สร้าง marker struct ชื่อ `__path_<handler_fn>` ไว้**ในโมดูลเดียวกับตัว
ฟังก์ชัน** เสมอ (ไม่ใช่ในโมดูลที่ import ฟังก์ชันไปใช้) — เมื่อเขียน `paths(register)` (ชื่อเปล่า ไม่มี module
prefix) `#[derive(OpenApi)]` จะสร้างโค้ดที่พยายามอ้าง `__path_register` แบบไม่มี prefix เช่นกัน ซึ่งไม่มีอยู่
ในสโคปของ `docs.rs` เพราะ `use register;` ธรรมดาไม่ได้พา sibling item ที่ macro สร้างขึ้นมาด้วย

**วิธีแก้**: เขียน path แบบเต็ม (fully-qualified) ในทุกจุดของ `paths(...)` แทนการ `use` ชื่อเปล่ามาก่อน —
`paths(crate::routes::auth::register, crate::routes::auth::login, ...)` ตามที่บทนี้ทำไว้จริงในหัวข้อ 92.10
— วิธีนี้ทำให้ macro คำนวณ path ของ `__path_register` เป็น `crate::routes::auth::__path_register` ซึ่งตรงกับ
ตำแหน่งจริงที่มันถูกสร้างไว้ (ทางเลือกอีกทางคือใช้ crate `utoipa-axum` ที่ Part 85 หัวข้อ 85.8 แนะนำไว้สำหรับ
โปรเจกต์ที่แยก `#[derive(OpenApi)]` ข้ามโมดูลจำนวนมาก แต่สำหรับ 10 endpoint ของบทนี้ fully-qualified path
ตรงไปตรงมาและเพียงพอ)

### 4. `sqlx::query!`/`query_as!` fail ตอน compile เพราะลืมรัน migration ก่อน `cargo build`

ระหว่างพัฒนาบทนี้ หลังเขียนโค้ด handler เสร็จแต่ยังไม่ได้รัน `sqlx migrate run` กับฐานข้อมูลที่ `DATABASE_URL`
ชี้ไป `cargo build` fail ทันทีด้วย error จากฐานข้อมูลจริง (ไม่ใช่ error จาก Rust compiler):

```
error: error returned from database: relation "users" does not exist at line 1449
  --> src/routes/auth.rs:72:15
   |
72 |       let row = sqlx::query_as!(
   |  _______________^
73 | |         UserRow,
74 | |         r#"
75 | |         INSERT INTO users (username, email, password_hash, role)
   | |_____^
```

**วิธีแก้**: `sqlx::query!`/`query_as!` (ในโหมด default ที่**ไม่ใช้** offline mode/`.sqlx/` cache) เชื่อมต่อ
ฐานข้อมูลจริงตอน**compile time**เพื่อตรวจสอบ SQL กับ schema จริง (ตามที่ Part 70 หัวข้อ 70.1 อธิบายไว้) —
ลำดับที่ถูกต้องเสมอคือ **สร้าง/migrate ฐานข้อมูลให้เสร็จก่อน แล้วค่อย `cargo build`** ไม่ใช่ตรงกันข้าม กับดัก
นี้พบบ่อยเป็นพิเศษเมื่อทำงานเป็นทีม (เพื่อนร่วมทีม pull โค้ดที่มี query ใหม่มา แต่ลืมรัน migration ใหม่ที่มา
คู่กันก่อน build) — Part 71 หัวข้อ pitfall ข้อ 3 อธิบายกับดักเดียวกันนี้ไว้แล้วอย่างละเอียด บทนี้เจอมันจริง
ระหว่างพัฒนาเช่นกัน ยืนยันว่าเป็นกับดักที่เกิดขึ้นได้กับทุกโปรเจกต์จริง ไม่ใช่แค่ตัวอย่างสมมติ

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** เพิ่ม endpoint `GET /api/v1/books?category=fiction` ให้ทำงานถูกต้องเมื่อไม่มีหนังสือใน
   category นั้นเลย (ควรได้ `data: []` กับ `total_items: 0` ไม่ใช่ error) — ทดสอบด้วย `curl` จริงกับ
   เซิร์ฟเวอร์ที่คุณสร้างจากบทนี้ แล้วเพิ่ม integration test ยืนยันเคสนี้ในไฟล์ `tests/api_tests.rs`
   (hint: ดูฟังก์ชัน `push_book_filters` ในหัวข้อ 92.8 — ควรทำงานถูกต้องอยู่แล้ว โจทย์นี้คือการ**พิสูจน์**ว่า
   มันถูกต้องจริงด้วยเทสของคุณเอง)

2. **[กลาง]** เพิ่ม endpoint ใหม่ `PATCH /api/v1/books/{id}` (admin เท่านั้น) ให้แก้ `total_copies` ได้ —
   ถ้าลด `total_copies` ต่ำกว่าจำนวนที่ถูกยืมอยู่ตอนนี้ (`total_copies - available_copies`) ต้องปฏิเสธด้วย
   `409 Conflict` ไม่ใช่ยอมให้ทำแล้วข้อมูลเพี้ยน (hint: ต้องคำนวณ `currently_borrowed = total_copies -
   available_copies` ก่อน แล้วตั้ง `available_copies` ใหม่ = `new_total_copies - currently_borrowed` ทั้งหมด
   ต้องอยู่ใน transaction เดียวเพื่อป้องกัน race condition แบบเดียวกับหัวข้อ 92.9 — และต้องอัปเดต CORS
   `allow_methods` ในหัวข้อ 92.11 ให้รองรับ `PATCH` ด้วย ไม่อย่างนั้น browser จะบล็อก preflight)

3. **[ยาก]** implement refresh token flow เต็มรูปแบบ (ที่ Part 74 sketch แนวคิดไว้แต่บทนี้ตัดออกจากสโคป) —
   เพิ่มตาราง `refresh_tokens` (เก็บ token แบบ hash ไม่ใช่ plaintext, ผูกกับ `user_id`, มี `expires_at` และ
   `revoked_at`), เพิ่ม endpoint `POST /api/v1/auth/refresh` ที่รับ refresh token แล้วออก access token ใหม่
   ให้, และเพิ่ม endpoint `POST /api/v1/auth/logout` ที่ revoke refresh token นั้น (hint: การ revoke ต้องเช็ค
   `revoked_at IS NULL AND expires_at > now()` ทุกครั้งที่ใช้ refresh token — คิดดูว่าทำไม access token เดิม
   ที่ยังไม่หมดอายุถึง**ยังใช้ได้ต่อจนกว่าจะหมดอายุของมันเอง** แม้ refresh token จะถูก revoke ไปแล้ว และนี่คือ
   ข้อแลกเปลี่ยนที่ต้องยอมรับเมื่อไม่มีระบบ token blacklist แยก เช่น Redis ที่ Part 83 จะสอน)

4. **[ยาก/ประยุกต์]** ออกแบบและ implement "ระบบจองคิว" (waitlist): เมื่อผู้ใช้พยายามยืมหนังสือที่ไม่มีสำเนา
   ว่าง ให้เพิ่ม endpoint `POST /api/v1/books/{id}/waitlist` ที่ให้ผู้ใช้เข้าคิวรอได้ (ต้องคิดเรื่อง: ผู้ใช้คน
   เดียวเข้าคิวซ้ำเล่มเดียวกันได้ไหม, ลำดับคิวเรียงตามอะไร) และแก้ `return_book` ให้ตรวจสอบคิวก่อนว่าจะให้
   สำเนาที่ว่างกลับไปเพิ่ม `available_copies` ตรง ๆ หรือ "จอง" ให้คนแรกในคิวโดยอัตโนมัติ (สร้าง `borrows`
   record ให้เขาทันทีแทนการเพิ่ม `available_copies`) — โจทย์นี้ไม่มีคำตอบเดียวที่ถูก เป้าหมายคือฝึกออกแบบ
   business logic ที่ซับซ้อนขึ้นด้วย transaction เดียวกันกับที่หัวข้อ 92.9 สอนไว้ (hint: นี่คือจุดที่แนวคิด
   message queue จาก Part 82 จะมีประโยชน์มากถ้าต้องแจ้งเตือนผู้ใช้ว่า "ถึงคิวคุณแล้ว" แบบ asynchronous ไม่ใช่
   ให้ผู้ใช้ต้อง poll เอง — แต่การ implement เต็มรูปแบบนั้นเกินสโคปของแบบฝึกหัดนี้ ให้โฟกัสที่ database logic
   ก่อน)

## สรุป

บทนี้สร้าง backend ที่สมบูรณ์และรันได้จริง 100% สำหรับระบบห้องสมุด/ยืม-คืนหนังสือ — 10 endpoint ครอบคลุม
authentication (JWT + argon2), authorization (RBAC แบบ admin-only), CRUD หนังสือพร้อม pagination/filter,
และหัวใจของ business logic (ยืม/คืน) ที่ปลอดภัยจาก race condition ด้วย transaction + atomic conditional
update ซึ่งพิสูจน์แล้วจริงด้วยการยิง 10 request พร้อมกัน — ทุกความล้มเหลวที่เป็นไปได้ตอบกลับเป็น JSON shape
เดียวกันผ่าน `AppError` ที่ต่อยอดจาก Part 66 และเลือก HTTP status code (401/403/404/409) ตามหลักการที่ Part
76 สอนไว้อย่างเคร่งครัด ทั้งหมดมีเอกสาร OpenAPI ที่ตรงกับโค้ดจริงผ่าน `utoipa` และมี integration test suite
5 ตัวที่ผ่านทั้งหมดคอยยืนยัน regression

**สิ่งที่ Part 93 (Full-Stack Project 2/3: สร้าง Frontend) ต้องใช้จากบทนี้**:

- **Base URL**: `http://127.0.0.1:8092` (ปรับผ่าน `BIND_ADDR`)
- **API contract**: ตารางเต็มในหัวข้อ 92.1 — 10 endpoint, ทุก request/response shape, ทุก status code ที่
  เป็นไปได้ — Part 93 เรียก endpoint เหล่านี้ตรงตามที่กำหนดไว้ ไม่มีการเปลี่ยนแปลงโดยไม่แก้บทนี้ก่อน
- **CORS**: backend อนุญาต origin ที่ตั้งใน `FRONTEND_ORIGIN` (default `http://127.0.0.1:5173` — พอร์ต
  มาตรฐานของ Vite dev server) ด้วย method `GET`/`POST`/`OPTIONS` และ header `Content-Type`/`Authorization`
  — ถ้า Part 93 ใช้พอร์ต dev server อื่น ต้องตั้ง `FRONTEND_ORIGIN` ให้ตรงก่อนเรียก API ได้
- **Authentication**: แนบ `Authorization: Bearer <access_token>` ที่ได้จาก `POST /api/v1/auth/login` —
  token มีอายุ 3600 วินาที ไม่มี refresh token ในสโคปนี้ (ผู้ใช้ต้อง login ใหม่หลังหมดอายุ)
- **Error shape**: `{"error": {"code": string, "message": string, "fields"?: [...]}}` เสมอทุก endpoint ที่
  ล้มเหลว ไม่ว่า status จะเป็นอะไร

Part 94 (Full-Stack Project 3/3: Integration และ Deployment) จะใช้ `GET /health` สำหรับ health check,
integration test suite ในหัวข้อ 92.11 เป็น regression test ก่อน deploy, และ environment variable ทั้งสี่ตัว
ในหัวข้อ 92.12 สำหรับตั้งค่าตอน deploy จริง — ทั้งสามบท (92, 93, 94) ประกอบกันเป็น capstone เดียวที่แสดงให้
เห็นว่าทุกเทคนิคที่เรียนมาตลอด Module 4-5 ของหลักสูตรนี้ทำงานร่วมกันเป็นระบบจริงได้อย่างไร

---

**Part ก่อนหน้า:** [Server-Side Rendering (SSR) ด้วย Rust](part-091-ssr.md) | **Part ถัดไป:** [Full-Stack Project (2/3): สร้าง Frontend](part-093-fullstack-frontend.md)
