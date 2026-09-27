# Part 79: GraphQL ด้วย async-graphql

> โมดูล: การพัฒนาเว็บแอปพลิเคชัน (Web Development) | ระดับ: สูง | เวลาโดยประมาณ: 260 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้อย่างเป็นรูปธรรม (ไม่ใช่แค่ท่องนิยาม) ว่า GraphQL แก้ปัญหา **over-fetching** และ **under-fetching/N+1**
  ที่ REST เจอได้อย่างไร โดยพิสูจน์ด้วยตัวอย่างจริงบนโดเมนห้องสมุด/ระบบยืม-จองหนังสือที่ใช้ต่อเนื่องมาตั้งแต่
  Part 61-78 — เห็นภาพว่า client ที่ต้องการแค่ "ชื่อหนังสือ + ชื่อผู้แต่ง" ต้องยิง REST กี่ครั้ง เทียบกับ GraphQL
  ที่ยิงครั้งเดียว
- อธิบายแนวคิดหลักของ GraphQL ได้ครบ: **schema** (type, query, mutation, subscription), โมเดล
  **single endpoint** (`POST /graphql` ตัวเดียวรับทุกอย่าง ต่างจาก REST ที่มีหลาย endpoint ตาม resource) และ
  **resolver** ในฐานะโค้ด Rust ที่เป็น "ผู้ทำงานจริง" อยู่หลัง field แต่ละตัวของ schema
- ติดตั้งและเขียน GraphQL API ด้วย `async-graphql` ผสาน `async-graphql-axum` ได้ครบวงจร: สร้าง `Query` root
  ด้วย `#[Object]`, เลือกใช้ `#[derive(SimpleObject)]` เทียบกับ `#[Object]` ให้ถูกสถานการณ์, รับ query
  argument, เขียน mutation ที่ validate input และแปลง error เป็นรูปแบบ GraphQL error ที่มี extension code
- **ระบุและแก้ปัญหา N+1 ที่เกิดขึ้นข้างใน GraphQL เอง** (คนละปัญหากับ N+1 ระดับ REST ในหัวข้อ 79.1) ด้วย
  `async_graphql::dataloader::DataLoader` — พิสูจน์ด้วยตัวเลข query count จริงจากฐานข้อมูล PostgreSQL จริงว่า
  resolver ที่เขียนไม่ระมัดระวังทำให้เกิด query เพิ่มขึ้นเป็นเส้นตรงตามจำนวนแถว และ DataLoader ลดมันกลับมาเหลือ
  query คงที่ได้อย่างไร
- เชื่อมต่อ resolver กับ PostgreSQL ผ่าน `ctx.data::<PgPool>()` และเข้าใจว่าเป็นกลไกแบบเดียวกันกับ
  `State<T>` extractor ของ Axum (Part 64) ในบทบาทที่ต่างกันแค่ "ใครเป็นคนส่ง dependency เข้ามาให้"
- ประเมินได้อย่างตรงไปตรงมาว่า**เมื่อไหร่ความยืดหยุ่นของ GraphQL คุ้มกับความซับซ้อนที่แลกมา** เทียบกับ REST
  (Part 61-78) ที่หลักสูตรนี้ยังคงใช้เป็นหลักในบทถัด ๆ ไป — GraphQL และ gRPC (Part 80) เป็นเครื่องมือที่ "ควรรู้จัก
  และรู้วิธีใช้" ไม่ใช่ตัวเลือกหลักของหลักสูตร

## ความรู้ที่ต้องมีมาก่อน

- **Part 61 (HTTP Fundamentals และ REST API Concepts)**: บทนี้ใช้ความเข้าใจเรื่อง HTTP method, status code,
  และโดยเฉพาะโมเดล "REST คือหลาย endpoint ต่อ resource" เป็นจุดเทียบตลอดทั้งบท — ถ้ายังไม่แน่นเรื่อง REST
  ควรกลับไปทวน Part 61 ก่อน เพราะทุกข้อดี/ข้อเสียของ GraphQL ที่บทนี้อธิบายอ้างอิงกับ REST เป็นฐานเปรียบเทียบ
  โดยตรง
- **Part 62-64 (Axum: Intro, Routing/Handlers, State/Extractors)**: `async-graphql-axum` คือสิ่งที่ห่อ Axum
  handler ปกติไว้อีกชั้น — เราจะยังใช้ `Router`, `Extension`, และแนวคิด shared state จาก Part 64 ทั้งหมด
  เพียงแค่มี handler แค่ตัวเดียว (`POST /graphql`) ที่รับทุก query/mutation แทนที่จะมีหลาย handler ต่อ route
- **Part 44-45 (Procedural Macros/Attribute Macros)**: `#[Object]`, `#[derive(SimpleObject)]`, และ
  `#[Subscription]` **คือ proc macro/attribute macro ทั้งหมด** ตามที่ Part 44-45 สอนไว้ — บทนี้ไม่สอนกลไก macro
  ซ้ำ แต่จะชี้ให้เห็นตรง ๆ ว่าโค้ดที่ macro พวกนี้ generate ให้ทำอะไรอยู่ เพื่อให้เข้าใจว่า "มายากล" ที่เห็นไม่ใช่
  เวทมนตร์ลึกลับ แต่เป็นโค้ด Rust ธรรมดาที่ macro เขียนแทนให้
- **Part 57 (Serde เบื้องต้น)**: `#[derive(SimpleObject)]` ทำหน้าที่คล้าย `#[derive(Serialize)]` มาก — คือ
  "แปลง struct หนึ่งให้เป็นตัวแทนอีกรูปแบบหนึ่งโดยอัตโนมัติ" (JSON ในกรณี serde, GraphQL type ในกรณีนี้) การ
  เข้าใจ derive macro ของ serde มาก่อนจะช่วยให้เข้าใจ `SimpleObject` ได้เร็วขึ้นมาก
- **Part 63 (Axum: Handlers และ Extractors)**: หัวข้อ query argument (79.5) จะเทียบ mechanism การรับ input
  ของ GraphQL กับ extractor ของ Axum ตรง ๆ — ทั้งสองแก้ปัญหาเดียวกัน ("รับ input ที่ type ถูกต้องจาก request")
  ด้วยกลไกที่ต่างกันโดยสิ้นเชิง
- **Part 66 (Error Handling ใน Axum: `AppError` แบบมืออาชีพ)**: หัวข้อ mutation (79.7) จะแปลง `AppError`
  เดียวกันในเชิงแนวคิดที่ Part 66 สอนไว้ (แต่คนละ enum ในบทนี้ เพราะบริบทต่างกัน) ให้เป็น GraphQL error ผ่าน
  `async_graphql::ErrorExtensions` แทนการแปลงเป็น HTTP status code แบบ REST
- **Part 70-71 (SQLx และ PostgreSQL, Transaction)**: resolver ทุกตัวในบทนี้คุยกับ PostgreSQL จริงผ่าน `PgPool`
  แบบเดียวกับที่ Part 70 สอน — หัวข้อ mutation ใช้ `pool.begin()`/`commit()` ตรงตาม pattern transaction ที่
  Part 70 วางไว้ทุกประการ
- **Part 76 (RBAC และ Authorization)**: หัวข้อ 79.11 จะอ้างอิงความเข้าใจเรื่อง per-resource authorization
  check จาก Part 76 เพื่ออธิบายว่าทำไม GraphQL ทำ authorization ยากกว่า REST ในบางมิติ
- **Part 77 (WebSocket)**: หัวข้อ 79.10 (subscription) จะอ้างอิงกลไก WebSocket upgrade ที่ Part 77 สอนไว้ตรง ๆ
  — GraphQL subscription ส่งข้อมูลผ่าน WebSocket connection เดียวกันกับที่ Part 77 อธิบาย ไม่ใช่ protocol ใหม่
- **Part 78 (RESTful API Design Best Practices)**: เป็น Part ก่อนหน้าตามลำดับเนื้อหา สรุปแนวทางออกแบบ REST
  API ที่ดีไว้ — บทนี้ต่อยอดจากจุดนั้นด้วยการถามคำถามว่า "แล้วถ้า REST ที่ออกแบบมาดีที่สุดแล้วก็ยังมีข้อจำกัดบาง
  อย่างในตัวมันเองอยู่ดี จะทำอย่างไร" ถ้าคุณยังไม่ได้อ่าน Part 78 ไม่เป็นไร บทนี้ไม่ต้องพึ่งรายละเอียดของ Part 78
  โดยตรง แค่รู้ว่า Part 78 คือ REST API design ที่ "ดีที่สุดในกรอบของ REST" ก็พอสำหรับอ่านบทนี้ต่อ

## หมายเหตุเรื่องการตรวจสอบเนื้อหา (สำคัญ — อ่านก่อนเริ่ม)

ก่อนเขียนบทนี้ ผู้เขียนตรวจสอบว่ามี PostgreSQL จริงให้ทดสอบหรือไม่ (ตามที่ Part 70/71 เคยตั้งค่าไว้) และพบว่า
**cluster PostgreSQL 16.13 ยังออนไลน์อยู่จริง** จึงสร้างฐานข้อมูล scratch แยกชื่อ `part79_graphql_scratch`
(ไม่แตะฐานข้อมูลของบทอื่น) และทดสอบทุกตัวอย่างในบทนี้แบบ **compile และ run จริง เชื่อมต่อฐานข้อมูลจริง** ทั้งหมด
รายละเอียดสิ่งที่ทดสอบจริง:

- ตั้งโปรเจกต์ scratch แยกไว้นอก repo (ลบทิ้งหลังเขียนบทเสร็จ) เพิ่ม `async-graphql = "7.2.1"` และ
  `async-graphql-axum = "7.2.1"` (เวอร์ชันปัจจุบันจาก crates.io ณ วันที่เขียนบทนี้) พร้อม `axum = "0.8.9"`,
  `sqlx = "0.9.0"` (ตรงกับเวอร์ชันที่ Part 70 ใช้), `tokio`, `serde`, `thiserror`
- สร้างตาราง `authors`/`books`/`bookings` จริงในฐานข้อมูล ใส่ข้อมูลจริง 3 authors, 5 books
- คอมไพล์และรัน Axum server จริงที่ mount GraphQL endpoint จริง ยิง query/mutation ด้วย `curl -X POST` จริง
  (ตามที่หัวข้อ 79.3 จะอธิบายว่าทำไม `curl` ธรรมดาก็เพียงพอสำหรับ GraphQL — ไม่ต้องมี client พิเศษ) แล้วคัดลอก
  JSON response ที่ได้จริงมาแสดงในบทนี้ทุกจุด ไม่มีการแต่ง output
- **พิสูจน์ N+1 problem จริงด้วยตัวเลข query count จริง**: เขียน resolver แบบ naive ที่ resolve `author`
  ทีละ book แยกกัน วัด query count จริงด้วย counter และ `tracing`/`sqlx` statement log (`RUST_LOG=sqlx=debug`)
  ได้ผลจริงคือ **6 query** (1 สำหรับ list + 5 สำหรับ author แต่ละเล่ม) จากนั้นแก้ด้วย
  `async_graphql::dataloader::DataLoader` วัดซ้ำได้ **2 query** (1 สำหรับ list + 1 query แบบ batched
  `WHERE id = ANY($1)`) — ตัวเลขทั้งสองนี้คัดลอกจาก log จริงที่รันได้จริง ไม่ใช่ตัวเลขสมมติ
- ทดสอบ mutation `createBooking` จริงที่ใช้ transaction จริง (ลด `available_copies` + insert `bookings`)
  รวมถึงกรณี error จริง 3 แบบ (validation ล้มเหลว, ไม่พบหนังสือ, หนังสือหมด) และตรวจค่าในฐานข้อมูลก่อน/หลังจริง
  ด้วย `psql` เพื่อยืนยันว่า transaction ทำงานถูกต้อง
- ทดสอบ error message จริงที่ compiler/runtime ให้มา สำหรับกับดักท้ายบททุกข้อ (ไม่มีข้อความ error ที่แต่งขึ้นเอง)
- ลบฐานข้อมูล `part79_graphql_scratch` และโปรเจกต์ scratch ทิ้งทั้งหมดหลังเขียนบทนี้เสร็จ ไม่มีสิ่งใดหลงเหลือ
  ในเครื่องหรือกระทบไฟล์อื่นในหลักสูตร

ทุกที่ที่มีการอ้างผลลัพธ์ (`cargo build`, error message, JSON response, ตัวเลข query count) ในบทนี้คือผลลัพธ์ที่
**รันจริงแล้วคัดลอกมา** — ถ้าเจอความต่างเล็กน้อยตอนคุณลองรันเอง (เช่น เวอร์ชัน `async-graphql`/`axum` ใหม่กว่าที่
เขียนไว้ตรงนี้) ให้ยึด `cargo add` ที่รันในเครื่องคุณเป็นความจริงล่าสุดเสมอ ตามหลักการเดียวกับที่ Part 70 วางไว้

## เนื้อหา

### 79.1 ปัญหาที่ REST เจอ: Over-fetching และ Under-fetching

#### ทวนความเข้าใจ: REST คือ "หลาย endpoint ต่อหนึ่ง resource"

จาก Part 61 คุณรู้แล้วว่า REST คือสถาปัตยกรรมที่จัดโครงสร้าง API รอบ **resource** — แต่ละ resource มี URL ของ
ตัวเอง (`/api/v1/books`, `/api/v1/books/{id}`, `/api/v1/authors/{id}`) และ client เรียก HTTP method ที่
เหมาะสม (`GET`, `POST`, ...) ไปยัง URL นั้น ๆ โมเดลนี้เรียบง่ายและเข้าใจง่ายมาก — แต่มันมี **ข้อจำกัดในตัวเอง 2
แบบ** ที่ปรากฏชัดที่สุดตอนหน้าจอ (frontend, mobile app) ต้องการข้อมูลที่ **ไม่ตรงพอดี** กับ shape ที่แต่ละ
endpoint ออกแบบไว้ล่วงหน้า

#### Over-fetching: ได้มากกว่าที่ต้องการ

สมมติระบบห้องสมุดของเรา (ตามโดเมนที่ใช้มาตลอด Part 61-78) มี endpoint `GET /api/v1/books/{id}` ที่ REST API
ที่ออกแบบมาอย่างดี (ตาม Part 78) จะคืน "resource เต็มรูปแบบ" ของหนังสือเล่มนั้นเสมอ (เพราะ REST ออกแบบรอบ
resource ไม่ใช่รอบ "หน้าจอที่ client ต้องการ"):

```json
{
  "id": 3,
  "isbn": "978-0-00-000003",
  "title": "Norwegian Wood",
  "author_id": 2,
  "status": "available",
  "total_copies": 4,
  "available_copies": 1,
  "published_year": 1987,
  "created_at": "2026-01-15T09:00:00Z",
  "updated_at": "2026-03-02T14:22:10Z"
}
```

ทีนี้สมมติว่าหน้าจอที่กำลังพัฒนาอยู่คือ**หน้ารายการหนังสือแบบย่อ** (book list card) ที่ต้องการแค่ `title` กับ
`author` — สอง field จาก field ทั้งหมด 10 field ที่ตอบกลับมา ไม่มีทางบอก REST endpoint นี้ว่า "ขอแค่ 2 field
พอ" ได้เลย (นอกจากจะออกแบบ endpoint แยกต่างหากสำหรับ use case นี้โดยเฉพาะ ซึ่งจะพาไปสู่ปัญหาอีกชุดที่กล่าวถึง
ด้านล่าง) — นี่คือ **over-fetching**: client **ต้องรับข้อมูลเกินที่ต้องการใช้จริงเสมอ** เพราะ shape ของ
response ถูกกำหนดโดย server (ตาม resource) ไม่ใช่โดย client (ตาม use case ณ ขณะนั้น)

ผลกระทบของ over-fetching ไม่ใช่แค่ "เสีย bandwidth นิดหน่อย" — ในสถานการณ์จริงที่ response มี field ซับซ้อน
กว่านี้มาก (เช่น nested object, array ยาว ๆ) หรือ client เป็น mobile app ที่ทำงานบนเครือข่ายมือถือที่ไม่แน่นอน
ผลกระทบนี้ทวีคูณขึ้นเรื่อย ๆ ตามจำนวน field ที่ไม่ได้ใช้และจำนวน request ที่ยิงบ่อย ๆ (เช่น รายการหนังสือที่
scroll ได้เรื่อย ๆ)

#### Under-fetching และ N+1 ระดับ REST: ต้องยิงหลายรอบเพื่อประกอบข้อมูลที่ต้องการ

ปัญหาตรงข้ามเกิดขึ้นเมื่อ client ต้องการข้อมูลที่ **กระจายอยู่คนละ resource** สมมติหน้าจอเดียวกัน (book list
card) ต้องโชว์ **ชื่อหนังสือ + ชื่อผู้แต่ง** — แต่ endpoint `GET /api/v1/books` คืนแค่ `author_id` (ตัวเลข
foreign key) ไม่คืนชื่อผู้แต่งจริง (เพราะ "ผู้แต่ง" เป็น resource คนละตัวตาม REST convention) client ต้อง:

1. เรียก `GET /api/v1/books?status=available&limit=5` → ได้ list ของหนังสือ 5 เล่ม พร้อม `author_id` ของแต่ละ
   เล่ม (สมมติได้ `author_id` ที่ไม่ซ้ำกัน 3 ค่า: 1, 2, 3)
2. เรียก `GET /api/v1/authors/1`, `GET /api/v1/authors/2`, `GET /api/v1/authors/3` **แยกกัน 3 ครั้ง** เพื่อได้
   ชื่อผู้แต่งแต่ละคน

รวมแล้ว client ต้องยิง **4 request** (1 + 3) เพื่อประกอบข้อมูลที่ "ควรจะเป็นแค่ 1 request เดียว" ในเชิงตรรกะทาง
ธุรกิจ (client แค่ต้องการ "หนังสือกับชื่อผู้แต่ง" ไม่ได้อยากรู้เรื่อง resource boundary ของ backend) — นี่คือ
รูปแบบคลาสสิกของปัญหาที่เรียกกันในวงการว่า **N+1 problem ระดับ REST**: 1 request สำหรับ list หลัก บวก N
request เพิ่ม (N = จำนวน related resource ที่ไม่ซ้ำกัน) เพื่อได้ข้อมูลที่เกี่ยวข้องมาครบ

**ข้อสังเกตสำคัญ**: บางทีทีม backend พยายามแก้ปัญหานี้ด้วยการเพิ่ม query parameter แบบ `?include=author` ให้
`GET /api/v1/books` คืน author แนบมาด้วยเลย (`embedding`/`expansion` pattern) — วิธีนี้ **แก้ปัญหาได้จริงในระดับ
หนึ่ง** แต่ก็เป็นการ**เพิ่ม endpoint variant** ขึ้นมาอีกชุด (endpoint เดิมที่ไม่ include, endpoint ที่ include
author, บางทีต้อง include ทั้ง author และ borrow history พร้อมกันอีก) — จำนวน combination ของ "field ที่อยาก
ได้" โตขึ้นเรื่อย ๆ ตามจำนวนหน้าจอที่ frontend ต้องการ และ backend ต้องคอยเพิ่ม query parameter ใหม่ทุกครั้งที่
มี use case ใหม่เกิดขึ้น — นี่คือจุดที่ GraphQL เข้ามาเสนอวิธีคิดที่ต่างออกไปโดยสิ้นเชิง

#### สิ่งที่ GraphQL เสนอ: ให้ Client เป็นคนกำหนด Shape ของ Response เอง

แนวคิดหลักของ GraphQL คือ**พลิกความรับผิดชอบ**: แทนที่ server จะตัดสินใจ shape ของ response ล่วงหน้า (แบบ
REST) **client เขียน query ที่บอกตรง ๆ ว่าต้องการ field อะไรบ้าง จากที่ไหนบ้าง ในคำขอเดียว** ตัวอย่าง query ที่
ตอบโจทย์ทั้ง over-fetching และ under-fetching ข้างบนพร้อมกันในคำขอ**เดียว**:

```graphql
{
  books(status: "available", limit: 5) {
    title
    author {
      name
    }
  }
}
```

query นี้บอกชัดเจนว่า: เอาหนังสือที่ `status = "available"` มา 5 เล่ม, ต่อเล่มขอแค่ `title` (ไม่เอา `isbn`,
`total_copies`, `created_at`, ... ที่ไม่ได้ใช้ — แก้ over-fetching), และของแต่ละเล่มขอ `author.name` ต่อไปในคำ
ขอเดียวกัน (ไม่ต้องยิง request แยกไปหา author endpoint — แก้ under-fetching/N+1 **ในมุมของ client**) — ผลลัพธ์
ที่ได้กลับมามี shape **ตรงกับที่ query ขอเป๊ะ** ไม่มี field เกิน ไม่มี field ขาด:

```json
{
  "data": {
    "books": [
      { "title": "A Wizard of Earthsea", "author": { "name": "Ursula K. Le Guin" } },
      { "title": "The Left Hand of Darkness", "author": { "name": "Ursula K. Le Guin" } }
    ]
  }
}
```

(ตัวอย่าง JSON ด้านบนคือผลลัพธ์จริงที่ทดสอบไว้ในหัวข้อ 79.5 — จะเห็นตรงกันเป๊ะทุกไบต์)

สิ่งสำคัญที่ต้องเข้าใจให้ชัดตั้งแต่ต้นบท (และจะย้ำอีกครั้งในหัวข้อ 79.6): **GraphQL แก้ปัญหา N+1 "ระดับ HTTP
round-trip จาก client"** ได้จริง (client ยิงครั้งเดียว ไม่ใช่ 4 ครั้งแบบ REST ข้างบน) แต่**ไม่ได้แก้ปัญหา N+1
ระดับ "จำนวน query ไปฐานข้อมูล" ให้อัตโนมัติ** — ถ้า resolver ของ `author` เขียนไม่ระมัดระวัง เซิร์ฟเวอร์เองก็
ยังยิง SQL query แยกไปหาฐานข้อมูลทีละเล่มอยู่ดี เพียงแต่ client มองไม่เห็นความแตกต่างนี้เพราะมันเกิดขึ้น
**หลังบ้าน** ทั้งหมด — นี่คือ N+1 ปัญหาที่สองที่หัวข้อ 79.6 จะพิสูจน์ด้วยตัวเลขจริง และเป็นเหตุผลที่ประโยคที่
มักพูดกันว่า "GraphQL แก้ N+1" **ไม่ถูกทั้งหมด** ถ้าไม่มีบริบทกำกับให้ชัดว่ากำลังพูดถึง N+1 ระดับไหน

### 79.2 GraphQL คืออะไรกันแน่: Schema, Single Endpoint, Resolver

#### Schema: สัญญาที่ตายตัวระหว่าง Client กับ Server

หัวใจของ GraphQL คือ **schema** — เอกสารที่นิยาม**ทุกอย่าง**ที่ client สามารถถามหรือสั่งได้ ประกอบด้วยสาม
ส่วนหลัก:

- **Query**: อ่านข้อมูล (เทียบได้กับ `GET` ใน REST — ไม่ควรมี side effect ตามหลัก safe ที่ Part 61 สอนไว้)
- **Mutation**: เปลี่ยนสถานะข้อมูล (เทียบได้กับ `POST`/`PUT`/`PATCH`/`DELETE` ใน REST รวมกันหมดเป็นกลุ่มเดียว —
  GraphQL **ไม่แยก** ระดับความหมายแบบ HTTP method ให้ ทุก mutation เท่ากันหมดในสายตาของ protocol เอง)
- **Subscription**: สมัครรับข้อมูลแบบ real-time เมื่อมีเหตุการณ์เกิดขึ้น (เทียบได้กับ WebSocket connection ที่
  Part 77 สอนไว้ — หัวข้อ 79.10 จะอธิบายว่าจริง ๆ แล้ว subscription **คือ** WebSocket connection ที่ห่อ
  protocol ของ GraphQL ไว้อีกชั้น ไม่ใช่กลไกใหม่ทั้งหมด)

schema เขียนด้วยภาษาของตัวเอง เรียกว่า **SDL (Schema Definition Language)** ตัวอย่าง schema แบบง่าย ๆ สำหรับ
โดเมนห้องสมุด (เขียนแบบ SDL — ในบทนี้เราจะไม่เขียน SDL ตรง ๆ ด้วยมือ แต่ให้ `async-graphql` generate จากโค้ด
Rust ให้อัตโนมัติ ตามที่หัวข้อ 79.3 จะอธิบาย):

```graphql
type Author {
  id: ID!
  name: String!
  country: String
}

type Book {
  id: ID!
  isbn: String!
  title: String!
  status: String!
  publishedYear: Int
  isAvailable: Boolean!
  author: Author!
}

type Query {
  book(id: ID!): Book
  books(status: String, limit: Int): [Book!]!
}

type Mutation {
  createBooking(bookId: ID!, borrowerName: String!): Booking!
}
```

สังเกตเครื่องหมาย `!` ต่อท้าย type (เช่น `String!`, `[Book!]!`) — นี่คือ **non-null marker** บอกว่า field นั้น
**ไม่มีวันเป็น `null`** (ถ้าไม่มี `!` แปลว่า field นั้นเป็น nullable ได้ เทียบตรงกับ `Option<T>` ของ Rust ที่
Part 11 สอนไว้) — `[Book!]!` อ่านจากในออกนอก: "array ที่ตัวมันเองไม่เป็น null, แต่ละ element ในนั้นก็ไม่เป็น
null เหมือนกัน" schema จึงเป็น **สัญญาที่ตายตัวและตรวจสอบได้** ระหว่าง client กับ server — client รู้ล่วงหน้า
แน่นอนว่า field ไหนมีจริง type อะไร nullable หรือไม่ ก่อนที่จะยิง query จริงเสียอีก (ผ่านกลไกที่เรียกว่า
**introspection** ซึ่งหัวข้อ 79.9 จะพูดถึงตอนอธิบาย GraphiQL)

#### Single Endpoint: `POST /graphql` รับทุกอย่าง

ความต่างที่ชัดที่สุดระหว่าง GraphQL กับ REST ในระดับ HTTP คือ **จำนวน endpoint** — REST (Part 61-78) มี
endpoint จำนวนมากตามจำนวน resource (`/books`, `/books/{id}`, `/authors/{id}`, `/bookings`, ...) แต่ GraphQL มี
**endpoint เดียว** เสมอ (ตามธรรมเนียมทั่วไปคือ `/graphql`) รับทุกอย่างผ่าน HTTP method เดียว (`POST` เกือบตลอด
— มีสเปกให้ใช้ `GET` สำหรับ query อย่างเดียวได้ในบางกรณี แต่ `POST` คือค่าที่ใช้กันเป็นมาตรฐานจริงในทางปฏิบัติ
เพราะ query string ของ GraphQL มักยาวเกินกว่าจะใส่ใน URL ของ `GET` ได้สะดวก)

รูปแบบของ HTTP request หนึ่งครั้งของ GraphQL คือ **`POST` ไปที่ `/graphql`** พร้อม body เป็น JSON ที่มี field
`query` (string ของ GraphQL query/mutation) และ `variables` (object ของค่าตัวแปรที่ query อ้างถึง — ตัวเลือก):

```json
{
  "query": "query($id: ID!) { book(id: $id) { title } }",
  "variables": { "id": 3 }
}
```

**ข้อสังเกตสำคัญที่เชื่อมกับ Part 61**: request ข้างบนนี้ **ไม่มีอะไรใหม่ในระดับ wire เลย** — มันก็คือ HTTP
request ปกติที่มี method `POST`, header `Content-Type: application/json`, และ body เป็น JSON string ตามที่
Part 61 หัวข้อ 61.1 อธิบายโครงสร้างไว้ทุกประการ — **GraphQL ไม่ใช่ protocol ใหม่ที่แทนที่ HTTP** มันคือ
"รูปแบบข้อตกลง (convention) ของ body และ routing" ที่วางอยู่**บน** HTTP อีกชั้นหนึ่ง เหมือนกับที่ REST ก็เป็น
convention บน HTTP เช่นกัน (ตามที่ Part 61 อธิบายว่า REST เป็น architectural style ไม่ใช่ protocol) — นี่คือ
เหตุผลตรง ๆ ที่หัวข้อ 79.3 จะพิสูจน์ว่า `curl` ธรรมดา ๆ ยิง GraphQL ได้เลยโดยไม่ต้องมี client พิเศษอะไรเลย

ผลที่ตามมาจากโมเดล single-endpoint นี้มีทั้งข้อดีและข้อเสียที่ควรรู้ไว้ตั้งแต่ตอนนี้ (หัวข้อ 79.11 จะขยายให้
ครบ): ข้อดีคือ client ไม่ต้องจำ URL หลายสิบ/หลายร้อยตัวเหมือน REST API ขนาดใหญ่ — จำแค่ `/graphql` ตัวเดียว
แล้วเปลี่ยน "คำสั่ง" ข้างในตัว query ไปเรื่อย ๆ แต่ข้อเสียคือ **HTTP-level caching ที่พึ่ง URL เป็น key** (เช่น
CDN cache แบบที่ REST ใช้ `GET /books/3` เป็น cache key ได้ตรง ๆ) **ใช้กับ GraphQL ไม่ได้เลย** เพราะทุก query
ที่ต่างกัน (ต่างกันแค่ field ที่ขอ) ก็ยัง `POST` ไปที่ URL เดียวกันเป๊ะ — นี่คือหนึ่งใน trade-off ที่สำคัญที่สุด
ที่หัวข้อ 79.11 จะพูดถึง

#### Resolver: โค้ด Rust ที่อยู่หลัง Field แต่ละตัวของ Schema

schema บอกแค่ว่า "field นี้มี type อะไร" แต่**ไม่ได้บอกว่าค่าของ field นั้นมาจากไหน** — หน้าที่นั้นเป็นของ
**resolver** ซึ่งในบทนี้คือ**ฟังก์ชัน Rust ธรรมดา** ที่ผูกกับ field หนึ่งตัวใน schema แต่ละครั้งที่ client
ขอ field นั้นใน query, GraphQL engine (ของ `async-graphql`) จะเรียก resolver function ที่ผูกไว้ ให้มันคำนวณ
หรือดึงค่ามาคืน — resolver ของ field `title` อาจแค่คืนค่าจาก struct ตรง ๆ (ไม่มี logic อะไรเลย) ในขณะที่
resolver ของ field `author` อาจต้องไปยิง SQL query หา author จากฐานข้อมูล — **resolver แต่ละตัวทำงานเป็น
อิสระจากกัน** ไม่รู้ว่า field ข้างเคียงกำลังทำอะไรอยู่ (นี่คือรากของปัญหา N+1 ในหัวข้อ 79.6: ถ้า `author`
resolver ถูกเรียกแยกกัน 5 ครั้งสำหรับหนังสือ 5 เล่ม มันจะยิง SQL 5 ครั้งแยกกันโดยไม่รู้ตัวว่ามีเล่มอื่นเรียกพร้อม
กันอยู่ — เว้นแต่จะมีกลไกอย่าง DataLoader มาช่วยรวมมันเข้าด้วยกัน)

ในหัวข้อถัดไปเราจะเห็นว่า `async-graphql` ผูก resolver เข้ากับ schema ผ่าน **attribute macro** — `#[Object]`
ที่ครอบ `impl` block ของ struct หนึ่งตัว จะแปลง**แต่ละ method** ข้างในให้กลายเป็น**แต่ละ field ของ GraphQL
type นั้น** โดยอัตโนมัติ — นี่คือจุดที่ Part 44-45 (proc macro) เข้ามาเกี่ยวข้องตรง ๆ: `#[Object]` เป็นตัวอย่าง
จริงของ attribute macro ที่ **generate โค้ดจำนวนมาก** (การ implement trait ของ `async-graphql` ที่ผูก schema
เข้ากับ resolver function) จากโค้ดสั้น ๆ ที่คุณเขียน

### 79.3 ติดตั้งโปรเจกต์: `async-graphql` + `async-graphql-axum`

#### เพิ่ม Dependency

```bash
cargo add async-graphql
cargo add async-graphql-axum
```

ผลลัพธ์จริง (ทดสอบในเครื่องผู้เขียนบทนี้ ณ วันที่เขียน — เวอร์ชันปัจจุบันบน crates.io คือ `7.2.1` ทั้งสอง crate):

```text
    Updating crates.io index
      Adding async-graphql v7.2.1 to dependencies
             Features:
             + dynamic-schema
             + email-validator
             + fast_chemail
             + graphiql
             + handlebars
             + playground
             + tempfile
             41 deactivated features
    Updating crates.io index
     Locking 108 packages to latest Rust 1.94.1 compatible versions
```

พร้อมกับ dependency พื้นฐานที่ต้องมีควบคู่กันตามที่ Part 62/70 สอนไว้แล้ว:

```bash
cargo add axum
cargo add tokio --features full
cargo add serde --features derive
cargo add sqlx --features runtime-tokio,postgres,macros,chrono
```

`Cargo.toml` ส่วน `[dependencies]` ที่ได้ (ทดสอบจริง คอมไพล์ผ่านทั้งหมด):

```toml
[dependencies]
async-graphql = "7.2.1"
async-graphql-axum = "7.2.1"
axum = "0.8.9"
chrono = { version = "0.4.45", features = ["serde"] }
serde = { version = "1.0.229", features = ["derive"] }
sqlx = { version = "0.9.0", features = ["runtime-tokio", "postgres", "macros", "chrono"] }
tokio = { version = "1.53.1", features = ["full"] }
```

**ข้อสังเกตเรื่อง feature ของ `async-graphql`**: สังเกตว่า feature `graphiql` และ `playground` เปิดมาให้เป็น
**ค่า default** อยู่แล้ว (สองตัวนี้คือ interactive schema explorer สองเจ้าที่ต่างกัน — หัวข้อ 79.9 จะพูดถึง
`graphiql`) ถ้าต้องการลดขนาด binary/compile time สำหรับ production build ที่ไม่ต้องมี explorer ฝังในตัว
สามารถปิดได้ด้วย `default-features = false` แล้วเปิดเฉพาะ feature ที่ใช้จริง — บทนี้ปล่อยเป็นค่า default
เพราะกำลังจะใช้ `graphiql` จริงในหัวข้อ 79.9

#### `Query` Root แรก: `#[Object]` ในฐานะ Proc Macro

จาก Part 44-45 คุณรู้แล้วว่า attribute macro (เช่น `#[tokio::main]` ที่ใช้มาตลอดตั้งแต่ Part 48) ทำงานโดยรับ
โค้ดที่มันครอบอยู่ แปลงเป็น token stream แล้ว **generate โค้ด Rust ใหม่ทั้งหมด** มาแทนที่ — `#[Object]` ของ
`async-graphql` ทำงานแบบเดียวกันเป๊ะ เพียงแต่สิ่งที่มัน generate คือ implementation ของ trait ภายในของ
`async-graphql` ที่ผูก field ของ GraphQL type เข้ากับ method ของ Rust struct ที่มันครอบอยู่

```rust
use async_graphql::{EmptySubscription, Object, Schema};

// Query root: struct เปล่า ๆ ไม่มี field เลยก็ได้ (มันเป็นแค่ "ที่แขวน" resolver แต่ละตัว)
struct Query;

#[Object]
impl Query {
    // ทุก method ใน impl block ที่มี #[Object] ครอบอยู่ กลายเป็น field หนึ่งตัวของ
    // GraphQL type "Query" โดยอัตโนมัติ -- ชื่อ field ใน schema จะถูกแปลงจาก
    // snake_case เป็น camelCase ให้ (นี่คือ apiVersion ใน schema แม้ใน Rust เขียน
    // api_version -- ดูกับดักข้อ 5 ท้ายบท)
    async fn api_version(&self) -> &str {
        "1.0"
    }
}

// Schema ต้องรู้ทั้งสามส่วน (Query, Mutation, Subscription) แม้จะยังไม่มี mutation/
// subscription จริงก็ต้องใส่ placeholder -- async-graphql ให้ EmptyMutation/
// EmptySubscription มาสำหรับกรณีนี้
type ApiSchema = Schema<Query, async_graphql::EmptyMutation, EmptySubscription>;

fn build_schema() -> ApiSchema {
    Schema::build(Query, async_graphql::EmptyMutation, EmptySubscription).finish()
}
```

โค้ดนี้**คอมไพล์และรันได้จริง** — ทุก `async fn` ที่ `#[Object]` ครอบ**ต้อง**เป็น `async fn` เสมอ (แม้ resolver
นั้นจะไม่มี `.await` ข้างในเลยก็ตาม เช่น `api_version` ข้างบน) เพราะ `async-graphql` เรียก resolver ทุกตัวผ่าน
`Future` แบบเดียวกันหมด ไม่ว่าจะมี I/O จริงหรือไม่ — ข้อบังคับนี้เป็นราคาเล็ก ๆ ที่ต้องจ่ายเพื่อความสม่ำเสมอของ
interface ภายในของ engine

#### เชื่อมกับ Axum: `GraphQLRequest`/`GraphQLResponse`

`async-graphql-axum` ให้ extractor และ response type สองตัวที่ทำให้ handler ของ Axum (ตาม Part 63) คุยกับ
schema ของ `async-graphql` ได้ตรง ๆ:

```rust
use async_graphql::{EmptySubscription, Object, Schema};
use async_graphql_axum::{GraphQLRequest, GraphQLResponse};
use axum::{Extension, Router};
use axum::routing::post;

struct Query;

#[Object]
impl Query {
    async fn api_version(&self) -> &str {
        "1.0"
    }
}

type ApiSchema = Schema<Query, async_graphql::EmptyMutation, EmptySubscription>;

// handler ตัวเดียวรับ "ทุกอย่าง" -- ไม่ว่า client จะขอ apiVersion หรือ field ใหม่ที่
// เพิ่มเข้ามาทีหลัง handler นี้ไม่ต้องแก้เลย เพราะมัน "แค่" ส่ง request ต่อให้ schema
// ตัดสินใจว่าจะ resolve อย่างไร -- คนละวิธีคิดกับ Axum route ปกติที่ Part 62-64 สอนไว้
// ที่มี handler แยกกันต่อ route
async fn graphql_handler(
    schema: Extension<ApiSchema>,
    req: GraphQLRequest,
) -> GraphQLResponse {
    // GraphQLRequest มาจากการ parse body JSON ของ request ผ่าน extractor ของมันเอง
    // (คล้าย Json<T> extractor ที่ Part 63 สอน แต่ parse เป็น GraphQL query object
    // แทน struct ของแอปเรา) .execute() คือจุดที่ resolver ทุกตัวถูกเรียกจริง
    schema.execute(req.into_inner()).await.into()
}

fn app(schema: ApiSchema) -> Router {
    Router::new()
        .route("/graphql", post(graphql_handler))
        .layer(Extension(schema))
}
```

สังเกตว่า `graphql_handler` มี**พารามิเตอร์แบบ extractor 2 ตัว** ตรงตาม pattern ที่ Part 63 สอนไว้เป๊ะ —
`Extension<ApiSchema>` ดึง schema ที่แชร์ไว้ (ผ่าน `.layer(Extension(schema))` — คล้าย `State<T>` แต่ใช้
`Extension` เพราะเป็น pattern ที่นิยมใช้คู่กับ `async-graphql-axum` โดยเฉพาะ) และ `GraphQLRequest` parse body
JSON ให้เป็น GraphQL query object โดยอัตโนมัติ — **handler ตัวนี้ทั้งไฟล์ไม่มีวันต้องแก้เลย** ไม่ว่า schema
จะเพิ่ม field ใหม่กี่ตัวก็ตาม เพราะ logic ทั้งหมดของ "field ไหนทำอะไร" ถูกย้ายไปอยู่ที่ resolver ของแต่ละ type
แทน — นี่คือความต่างเชิงโครงสร้างที่สำคัญที่สุดจาก REST ที่ Part 62-64 สอนไว้ (REST เพิ่ม endpoint ใหม่ ต้อง
เพิ่ม route ใหม่เสมอ)

#### ตัวอย่างจริงเต็มรูปแบบ: Query ที่คุยกับ PostgreSQL จริง

มาต่อยอดให้ resolver คุยกับฐานข้อมูลจริงผ่าน `PgPool` (ตาม Part 70) — โครงสร้างตารางที่ใช้ตลอดบทนี้มีดังนี้
(สร้างจริงในฐานข้อมูล scratch ตามที่ระบุไว้ในหมายเหตุการตรวจสอบเนื้อหาด้านบน):

```sql
CREATE TABLE authors (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    country TEXT
);

CREATE TABLE books (
    id BIGSERIAL PRIMARY KEY,
    isbn TEXT NOT NULL UNIQUE,
    title TEXT NOT NULL,
    author_id BIGINT NOT NULL REFERENCES authors(id),
    status TEXT NOT NULL DEFAULT 'available',
    total_copies INTEGER NOT NULL DEFAULT 1 CHECK (total_copies >= 0),
    available_copies INTEGER NOT NULL DEFAULT 1 CHECK (available_copies >= 0),
    published_year INTEGER
);

CREATE TABLE bookings (
    id BIGSERIAL PRIMARY KEY,
    book_id BIGINT NOT NULL REFERENCES books(id),
    borrower_name TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

ใส่ข้อมูลจริง 3 authors, 5 books (2 เล่มของ Le Guin, 2 เล่มของ Murakami, 1 เล่มของนักเขียนไทย) — ตัวเลขนี้จะ
สำคัญมากตอนหัวข้อ 79.6 ที่พิสูจน์ N+1 ด้วยตัวเลขจริง

โค้ด resolver แบบง่ายที่สุดที่คุยกับฐานข้อมูล — query `book(id)` เดี่ยว ๆ (query ที่มี argument จะอธิบายเต็ม ๆ
ในหัวข้อ 79.5):

```rust
use async_graphql::{Context, Object};
use sqlx::PgPool;

#[derive(Debug, Clone, sqlx::FromRow)]
struct BookRow {
    id: i64,
    isbn: String,
    title: String,
    author_id: i64,
    status: String,
    available_copies: i32,
    published_year: Option<i32>,
}

struct Book(BookRow);

struct Query;

#[Object]
impl Query {
    async fn book(&self, ctx: &Context<'_>, id: i64) -> async_graphql::Result<Option<Book>> {
        // ctx.data::<PgPool>() คือกลไกที่หัวข้อ 79.8 จะอธิบายเทียบกับ State<T> ของ
        // Axum โดยตรง -- ตอนนี้จำแค่ว่ามันคือวิธีดึง "dependency ที่แชร์ไว้ล่วงหน้า"
        // (เหมือน pool ที่ Part 70 เก็บใน AppState) ออกมาใช้ใน resolver
        let pool = ctx.data::<PgPool>()?;
        let row = sqlx::query_as::<_, BookRow>(
            "SELECT id, isbn, title, author_id, status, available_copies, published_year
             FROM books WHERE id = $1",
        )
        .bind(id)
        .fetch_optional(pool)
        .await?;
        Ok(row.map(Book))
    }
}
```

**ทดสอบจริง**: รัน Axum server ที่ mount schema ข้างบนไว้ที่ `127.0.0.1:8079` แล้วยิงด้วย `curl` ตรง ๆ — นี่คือ
จุดที่ควรสังเกตให้ชัด: **`curl` ธรรมดา ไม่มี library หรือ client พิเศษของ GraphQL เลย** ก็ยิง GraphQL ได้ เพราะ
ตามที่หัวข้อ 79.2 อธิบายไว้ GraphQL คือ HTTP `POST` + JSON body ธรรมดา — คำสั่งที่รันจริง:

```bash
curl -s -X POST http://127.0.0.1:8079/graphql \
  -H 'Content-Type: application/json' \
  -d '{"query":"{ apiVersion }"}'
```

output จริงที่ได้กลับมา:

```json
{"data":{"apiVersion":"1.0"}}
```

และเมื่อยิง query ที่มี field ซ้อนกัน (nested field, `author` ที่เชื่อมไปอีกตาราง — resolver เต็มรูปแบบจะโชว์
ในหัวข้อ 79.4):

```bash
curl -s -X POST http://127.0.0.1:8079/graphql \
  -H 'Content-Type: application/json' \
  -d '{"query":"{ book(id: 3) { id title isAvailable status author { name country } } }"}'
```

output จริง:

```json
{"data":{"book":{"id":3,"title":"Norwegian Wood","isAvailable":true,"status":"available","author":{"name":"Haruki Murakami","country":"Japan"}}}}
```

สังเกตว่า response ตอบกลับมาตรงกับ**โครงสร้างของ query เป๊ะ** — เรียงลำดับ field ตามที่ query ขอ (`id`,
`title`, `isAvailable`, `status`, `author { name country }`) ไม่มี field อื่นแอบมาด้วยเลย นี่คือรูปธรรมของ
"client กำหนด shape" ที่หัวข้อ 79.1 อธิบายไว้ในเชิงทฤษฎี — ตอนนี้เห็นเป็น JSON จริงที่รันได้จริงแล้ว

### 79.4 นิยาม Object Type: `SimpleObject` กับ `#[Object]` เลือกใช้ให้ถูกสถานการณ์

`async-graphql` ให้สองวิธีหลักในการนิยาม GraphQL object type จาก struct ของ Rust — ทั้งสองวิธีแก้ปัญหาเดียวกัน
("แปลง struct ให้เป็น GraphQL type") แต่เหมาะกับสถานการณ์ต่างกัน

#### `#[derive(SimpleObject)]`: เมื่อ Field แต่ละตัว Map ตรง ๆ กับ Field ของ Struct

ถ้า struct ของคุณ**ไม่ต้องมี logic พิเศษต่อ field เลย** (ทุก field ของ GraphQL type ตรงกับ field ของ struct
Rust แบบ 1:1 ไม่มีการคำนวณเพิ่ม ไม่มีการยิง query เพิ่ม) `#[derive(SimpleObject)]` คือตัวเลือกที่สั้นและตรง
ประเด็นที่สุด:

```rust
use async_graphql::SimpleObject;

// เทียบได้ตรงกับ #[derive(Serialize)] ของ serde (Part 57): ทั้งสองแปลง struct หนึ่งตัว
// ให้เป็น "ตัวแทน" อีกรูปแบบหนึ่งโดยอัตโนมัติ -- ต่างกันแค่ปลายทาง (JSON กับ GraphQL
// type) โครงสร้างของทั้งสอง derive macro นี้ก็คล้ายกันมาก: อ่าน field ของ struct
// ตอน compile time แล้ว generate โค้ดที่ map field เหล่านั้นเข้ากับรูปแบบเป้าหมาย
#[derive(Debug, Clone, SimpleObject, sqlx::FromRow)]
struct Author {
    id: i64,
    name: String,
    country: Option<String>,
}
```

struct `Author` ข้างบนได้ประโยชน์สามอย่างพร้อมกันจาก derive macro สามตัวที่ผสานกัน: `sqlx::FromRow` (Part 70)
ทำให้ map จากแถวข้อมูล PostgreSQL ได้ตรง ๆ, `SimpleObject` ทำให้กลายเป็น GraphQL type ได้ตรง ๆ, และ
`Debug`/`Clone` เป็น utility พื้นฐานของ Rust — สังเกตว่า `country: Option<String>` แปลงเป็น GraphQL field
ที่ **nullable** โดยอัตโนมัติ (ไม่มี `!` ต่อท้าย type ใน schema ที่ generate ให้) ตรงตามที่หัวข้อ 79.2 อธิบายไว้
ว่า `Option<T>` ของ Rust คู่กับ nullable field ของ GraphQL แบบตรงไปตรงมา

#### `#[Object]`: เมื่อ Field ต้องมี Logic การคำนวณหรือดึงข้อมูลเพิ่ม

ทันทีที่ field ใด field หนึ่งของ type ต้อง**คำนวณจากข้อมูลอื่น** หรือ**ยิง query เพิ่ม** `SimpleObject` ใช้ไม่
ได้แล้ว (มันไม่มีช่องให้เขียน logic เลย — มันแค่ copy field ตรง ๆ) ต้องสลับไปใช้ `#[Object]` ซึ่งให้เขียน
resolver เป็น method จริง ๆ ได้:

```rust
use async_graphql::{Context, Object};
use sqlx::PgPool;

#[derive(Debug, Clone, sqlx::FromRow)]
struct BookRow {
    id: i64,
    isbn: String,
    title: String,
    author_id: i64,
    status: String,
    available_copies: i32,
    published_year: Option<i32>,
}

// Book เก็บ "ข้อมูลดิบจากฐานข้อมูล" (BookRow) ไว้ข้างใน แล้ว #[Object] ครอบ impl
// block ที่ expose field แต่ละตัวออกมาเป็น method ของตัวเอง -- แยก "รูปร่างข้อมูลจาก
// DB" ออกจาก "รูปร่างที่ GraphQL client เห็น" อย่างชัดเจน (ทั้งสองรูปร่างไม่จำเป็นต้อง
// เหมือนกันเป๊ะเสมอไป)
struct Book(BookRow);

#[Object]
impl Book {
    async fn id(&self) -> i64 {
        self.0.id
    }

    async fn isbn(&self) -> &str {
        &self.0.isbn
    }

    async fn title(&self) -> &str {
        &self.0.title
    }

    async fn status(&self) -> &str {
        &self.0.status
    }

    async fn published_year(&self) -> Option<i32> {
        self.0.published_year
    }

    // field นี้ "ไม่มีอยู่จริง" เป็น column ในตาราง books เลย -- มันคือค่าที่คำนวณสด ๆ
    // จาก available_copies ตอน resolve เท่านั้น นี่คือตัวอย่างชัดเจนที่สุดว่าทำไม
    // SimpleObject ใช้ไม่ได้กับ field นี้: ไม่มี field ชื่อ is_available ให้ copy ตรง ๆ
    async fn is_available(&self) -> bool {
        self.0.available_copies > 0
    }

    // field นี้ต้องยิง SQL query เพิ่ม (ไปหาตาราง authors) ทุกครั้งที่ client ขอ field
    // นี้ -- ทำงานถูกต้องสมบูรณ์เมื่อขอ Book แค่เล่มเดียว แต่จะกลายเป็นต้นตอของ N+1
    // ทันทีที่ resolve field นี้ให้กับ "list ของ Book หลายเล่ม" (หัวข้อ 79.6)
    async fn author(&self, ctx: &Context<'_>) -> async_graphql::Result<Author> {
        let pool = ctx.data::<PgPool>()?;
        let author = sqlx::query_as::<_, Author>(
            "SELECT id, name, country FROM authors WHERE id = $1",
        )
        .bind(self.0.author_id)
        .fetch_one(pool)
        .await?;
        Ok(author)
    }
}
```

**ตารางสรุปการเลือกใช้:**

| สถานการณ์ | ใช้ | เหตุผล |
|---|---|---|
| field ทุกตัว copy ตรงจาก struct field | `#[derive(SimpleObject)]` | สั้นที่สุด ไม่มี boilerplate ไม่มี logic ให้เขียนผิด |
| มี field ที่คำนวณจาก field อื่น (เช่น `is_available` จาก `available_copies`) | `#[Object]` | ต้องเขียน method ที่มี logic คำนวณ |
| มี field ที่ต้องยิง query/เรียก service อื่นเพิ่ม (เช่น `author`) | `#[Object]` | ต้องมี `async fn` ที่เข้าถึง `ctx` และทำ I/O |
| มี field ที่ผู้ใช้บาง role ไม่ควรเห็น (เชื่อม Part 76 RBAC) | `#[Object]` | ต้องเขียน logic ตรวจสอบสิทธิ์ก่อนคืนค่า หรือคืน `null` |

จุดที่ควรสังเกต: **ทั้งสองวิธีผสมกันในโปรเจกต์เดียวได้ตามธรรมชาติ** (เช่น `Author` ใช้ `SimpleObject` เพราะ
เรียบง่ายจริง ๆ ในขณะที่ `Book` ใช้ `#[Object]` เพราะมี field พิเศษ) — ไม่มีกฎว่าต้องเลือกแบบเดียวทั้งโปรเจกต์
เลือกตาม field ของ type นั้น ๆ เป็นราย type ไป และเปลี่ยนจาก `SimpleObject` เป็น `#[Object]` ได้เสมอในอนาคตถ้า
type นั้นเริ่มมี field ที่ต้องคำนวณเพิ่มขึ้นมาทีหลัง (schema ฝั่ง client ไม่เห็นความต่างเลยว่า field มาจากวิธี
ไหน — นี่เป็นรายละเอียด implementation ฝั่ง Rust เท่านั้น)

### 79.5 Query Arguments: รับ Input แบบมี Type จาก Client

query ในหัวข้อก่อน ๆ (`apiVersion`, `book(id)`) ยังจำกัดอยู่แค่ไม่มี argument หรือมีแค่ 1 argument — ในหัวข้อนี้
มาดู resolver ที่รับ**หลาย argument พร้อมกัน โดยบาง argument เป็น optional**:

```rust
use async_graphql::{Context, Object};
use sqlx::PgPool;

# struct Book(());
# struct Query;
#[Object]
impl Query {
    /// สอง argument, ทั้งคู่ optional ในความหมายที่ client ไม่ใส่ก็ได้ -- ฝั่ง Rust ให้
    /// ค่า default เอง (limit = 50 ถ้าไม่ระบุ, ไม่ filter status ถ้าไม่ระบุ)
    async fn books(
        &self,
        ctx: &Context<'_>,
        status: Option<String>,
        limit: Option<i32>,
    ) -> async_graphql::Result<Vec<Book>> {
        let pool = ctx.data::<PgPool>()?;
        let limit = limit.unwrap_or(50).clamp(1, 100);
        // (โค้ด query จริงดูเต็มในหัวข้อ 79.3/79.6 -- ตัดมาแสดงเฉพาะ signature ตรงนี้)
        # let _ = (pool, status, limit);
        # Ok(vec![])
    }
}
```

**เทียบกับ Axum extractor (Part 63) โดยตรง**: Part 63 สอนว่า Axum ใช้ extractor (`Query<T>`, `Path<T>`,
`Json<T>`) เพื่อดึง input จาก request แล้วแปลงเป็น type ที่ compiler ตรวจสอบได้ — ถ้า client ส่ง type ผิด
(เช่น ส่ง `"abc"` ให้ parameter ที่คาด `i32`) Axum ตอบ `400 Bad Request` กลับไปโดยที่ handler function ไม่ต้อง
เขียน validation เองเลย GraphQL argument ทำงานตาม**หลักการเดียวกันเป๊ะ** เพียงแต่กลไกต่างกันโดยสิ้นเชิง: schema
กำหนด type ของ argument ไว้ล่วงหน้า (`status: String`, `limit: Int`) และ engine ของ `async-graphql`
**ตรวจสอบ type ของ argument ที่ client ส่งมาก่อนที่จะเรียก resolver ด้วยซ้ำ** — ถ้า type ไม่ตรง client จะได้
GraphQL error กลับไปทันที (ไม่มีวันเข้าไปถึงตัว resolver function เลย) พูดให้เห็นภาพ: **Axum extractor ตรวจ
type ตรงจุดที่ HTTP request มาถึง route handler / GraphQL argument ตรวจ type ตรงจุดที่ query มาถึง resolver
— คนละจุดในสถาปัตยกรรม แต่แก้ปัญหาเดียวกัน คือ "รับประกันว่า input ที่มาถึง business logic เป็น type ที่
ถูกต้องแล้วเสมอ"**

ทดสอบจริงด้วย query ที่ใช้ argument ทั้งสองตัว:

```bash
curl -s -X POST http://127.0.0.1:8079/graphql \
  -H 'Content-Type: application/json' \
  -d '{"query":"{ books(status: \"available\", limit: 2) { title author { name } } }"}'
```

output จริง:

```json
{"data":{"books":[{"title":"A Wizard of Earthsea","author":{"name":"Ursula K. Le Guin"}},{"title":"The Left Hand of Darkness","author":{"name":"Ursula K. Le Guin"}}]}}
```

ผลลัพธ์ตรงกับที่ query ขอเป๊ะ (`status = "available"` กรองหนังสือที่ยืมได้ออกมา 2 เล่มแรกตาม `limit: 2`) —
สังเกตว่า argument ทั้งสองตัวเขียนอยู่ **ในตัว query string เดียวกัน** ไม่ใช่ query parameter ของ URL แบบ REST
(`?status=available&limit=2`) นี่คือความต่างเชิงรูปแบบที่มาจากโมเดล single-endpoint ในหัวข้อ 79.2: เพราะไม่มี
URL หลายตัวให้ใส่ query string ต่อท้าย ทุก argument จึงต้องอยู่ในตัว query ของ GraphQL เองทั้งหมด

**ทางเลือกที่ดีกว่าในโปรเจกต์จริง: `variables`** — ตัวอย่างข้างบนฝัง argument ไว้ในตัว query string ตรง ๆ
(เรียกว่า **inline argument**) ซึ่งใช้ได้ดีสำหรับตัวอย่างสั้น ๆ แต่ในโปรเจกต์จริง client มักส่งค่าที่มาจาก
input ผู้ใช้ (เช่น ค่าจากช่อง search) ผ่าน field `variables` แยกออกมาจาก `query` เพื่อไม่ต้องประกอบ string ของ
query ใหม่ทุกครั้งที่ค่าเปลี่ยน (และป้องกัน injection-like bug ที่มาจากการต่อ string เอง — คล้ายเหตุผลที่
Part 70 อธิบายว่าทำไม SQL ต้อง parameterize ด้วย `$1` แทนการต่อ string):

```bash
curl -s -X POST http://127.0.0.1:8079/graphql \
  -H 'Content-Type: application/json' \
  -d '{
        "query": "query($status: String, $limit: Int) { books(status: $status, limit: $limit) { title } }",
        "variables": { "status": "available", "limit": 2 }
      }'
```

ทั้งสองรูปแบบให้ผลลัพธ์เหมือนกัน — `variables` เป็นแค่วิธีที่**ปลอดภัยกว่าและจัดการง่ายกว่า**ในทางปฏิบัติเมื่อ
argument มาจาก input ที่เปลี่ยนไปเรื่อย ๆ

### 79.6 ปัญหา N+1 *ภายใน* GraphQL เอง และการแก้ด้วย DataLoader

นี่คือหัวข้อที่สำคัญที่สุดของบทนี้ในทางปฏิบัติจริง — เป็น **"กับดักการผลิต (production gotcha)"** อันดับหนึ่ง
ที่ทีมที่เพิ่งเริ่มใช้ GraphQL เจอกันเกือบทุกทีมโดยไม่รู้ตัว

#### ทวนความแตกต่าง: N+1 ระดับ REST เทียบกับ N+1 ระดับ Resolver

หัวข้อ 79.1 อธิบาย N+1 **ระดับ HTTP round-trip จาก client** (client ยิง REST หลายครั้งเพื่อประกอบข้อมูล) —
GraphQL แก้ปัญหานั้นได้จริงเพราะ client ยิงครั้งเดียว **แต่**ปัญหา N+1 แบบเดิมทุกประการสามารถ**ย้ายที่อยู่**
ไปเกิดขึ้น**ระหว่าง resolver กับฐานข้อมูล**แทน โดยที่ client มองไม่เห็นเลยว่ามันเกิดขึ้น — เพราะ **resolver
ของแต่ละ field ทำงานเป็นอิสระจากกันสนิท** (ตามที่หัวข้อ 79.2 อธิบายไว้) ถ้า query ขอ `books { author { name
} }` engine ของ `async-graphql` จะ:

1. เรียก resolver ของ `books` → ได้ list ของ `Book` กลับมา (สมมติ 5 เล่ม)
2. สำหรับ**แต่ละเล่ม**ใน list นั้น เรียก resolver ของ field `author` **แยกกันทีละเล่ม**

ถ้า resolver ของ `author` เขียนแบบหัวข้อ 79.4 (ยิง `SELECT ... FROM authors WHERE id = $1` ตรง ๆ ทุกครั้งที่
ถูกเรียก) นั่นแปลว่าฐานข้อมูลจะได้รับ **5 query แยกกัน** สำหรับ author เพียงเพราะมีหนังสือ 5 เล่ม — บวกกับ 1
query สำหรับ list หนังสือเอง รวมเป็น **6 query** ทั้งที่ query GraphQL ที่ client ส่งมามีแค่ **1 คำขอเดียว**

#### พิสูจน์ N+1 ด้วยตัวเลขจริง: Resolver แบบ Naive

โค้ด resolver แบบ naive (ทดสอบจริง คอมไพล์และรันจริงกับฐานข้อมูล PostgreSQL จริง — มีการเพิ่ม counter
`Arc<AtomicUsize>` เข้าไปนับจำนวน query แบบตรงไปตรงมา เพื่อให้เห็นตัวเลขที่แน่นอน ไม่ต้องเดาจาก log):

```rust
use async_graphql::{Context, Object};
use sqlx::PgPool;
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;

# #[derive(Debug, Clone, sqlx::FromRow, async_graphql::SimpleObject)]
# struct Author { id: i64, name: String, country: Option<String> }
# #[derive(Debug, Clone, sqlx::FromRow)]
# struct BookRow { id: i64, title: String, author_id: i64 }
struct Book(BookRow);

#[Object]
impl Book {
    async fn id(&self) -> i64 {
        self.0.id
    }

    async fn title(&self) -> &str {
        &self.0.title
    }

    // NAIVE: ยิง SELECT แยกกันทุกครั้งที่ field นี้ถูกเรียก โดยไม่รู้ตัวเลยว่ามีหนังสือ
    // เล่มอื่นถูก resolve field เดียวกันนี้อยู่ในเวลาเดียวกันหรือไม่ -- resolver นี้
    // "ถูกต้อง" ในเชิง logic 100% (ได้ค่า author ที่ถูกเสมอ) แต่ "ไม่มีประสิทธิภาพ"
    // ในเชิงจำนวน round-trip ไปฐานข้อมูล
    async fn author(&self, ctx: &Context<'_>) -> async_graphql::Result<Author> {
        let pool = ctx.data::<PgPool>()?;
        let counter = ctx.data::<Arc<AtomicUsize>>()?;
        counter.fetch_add(1, Ordering::SeqCst);
        let author = sqlx::query_as::<_, Author>(
            "SELECT id, name, country FROM authors WHERE id = $1",
        )
        .bind(self.0.author_id)
        .fetch_one(pool)
        .await?;
        Ok(author)
    }
}

struct Query;

#[Object]
impl Query {
    async fn books(&self, ctx: &Context<'_>) -> async_graphql::Result<Vec<Book>> {
        let pool = ctx.data::<PgPool>()?;
        let counter = ctx.data::<Arc<AtomicUsize>>()?;
        counter.fetch_add(1, Ordering::SeqCst); // นับ query ของ list เองด้วย
        let rows = sqlx::query_as::<_, BookRow>(
            "SELECT id, title, author_id FROM books ORDER BY id",
        )
        .fetch_all(pool)
        .await?;
        Ok(rows.into_iter().map(Book).collect())
    }

    async fn query_count(&self, ctx: &Context<'_>) -> i64 {
        ctx.data::<Arc<AtomicUsize>>()
            .map(|c| c.load(Ordering::SeqCst) as i64)
            .unwrap_or(-1)
    }
}
```

**รันจริง**: ยิง query ที่ขอ `books { title author { name } }` (5 เล่ม ตามข้อมูลที่ seed ไว้) แล้วต่อด้วย
`queryCount` เพื่ออ่านตัวเลข counter:

```bash
curl -s -X POST http://127.0.0.1:8081/graphql -H 'Content-Type: application/json' \
  -d '{"query":"{ books { title author { name } } }"}'
```

```json
{"data":{"books":[{"title":"A Wizard of Earthsea","author":{"name":"Ursula K. Le Guin"}},{"title":"The Left Hand of Darkness","author":{"name":"Ursula K. Le Guin"}},{"title":"Norwegian Wood","author":{"name":"Haruki Murakami"}},{"title":"Kafka on the Shore","author":{"name":"Haruki Murakami"}},{"title":"ช่างสำราญ","author":{"name":"เดือนวาด พิมวนา"}}]}}
```

```bash
curl -s -X POST http://127.0.0.1:8081/graphql -H 'Content-Type: application/json' \
  -d '{"query":"{ queryCount }"}'
```

```json
{"data":{"queryCount":6}}
```

**6 query จริง** สำหรับ 1 คำขอ GraphQL เดียว — และเพื่อยืนยันว่าไม่ใช่ตัวเลขที่มาจาก counter ที่เขียนผิด นี่คือ
log จริงจาก `sqlx` (เปิดด้วย `RUST_LOG=sqlx=debug` ตามที่ Part 70 แนะนำไว้เรื่อง statement logging):

```text
DEBUG sqlx::query: summary="SELECT id, title, author_id …" db.statement="\n\nSELECT id, title, author_id FROM books ORDER BY id\n" rows_affected=5 rows_returned=5 elapsed=2.331111ms
DEBUG sqlx::query: summary="SELECT id, name, country …" db.statement="\n\nSELECT id, name, country FROM authors WHERE id = $1\n" rows_affected=1 rows_returned=1 elapsed=1.155854ms
DEBUG sqlx::query: summary="SELECT id, name, country …" db.statement="\n\nSELECT id, name, country FROM authors WHERE id = $1\n" rows_affected=1 rows_returned=1 elapsed=1.691076ms
DEBUG sqlx::query: summary="SELECT id, name, country …" db.statement="\n\nSELECT id, name, country FROM authors WHERE id = $1\n" rows_affected=1 rows_returned=1 elapsed=1.780774ms
DEBUG sqlx::query: summary="SELECT id, name, country …" db.statement="\n\nSELECT id, name, country FROM authors WHERE id = $1\n" rows_affected=1 rows_returned=1 elapsed=1.747104ms
DEBUG sqlx::query: summary="SELECT id, name, country …" db.statement="\n\nSELECT id, name, country FROM authors WHERE id = $1\n" rows_affected=1 rows_returned=1 elapsed=1.687466ms
```

เห็นชัดเจนตรง ๆ ในบรรทัด log: 1 query แรกดึง `books` มา 5 แถว จากนั้น query เดียวกันเป๊ะ
(`SELECT id, name, country FROM authors WHERE id = $1`) ซ้ำอีก **5 ครั้ง** — คนละ `$1` (คนละ `author_id`) แต่
เป็น SQL text แบบเดียวกัน ยิงแยกกันทีละครั้ง — นี่คือ N+1 ปัญหาตัวจริงที่เกิดขึ้น**ข้างในเซิร์ฟเวอร์** โดยที่
client ที่ยิง query GraphQL เพียงครั้งเดียวไม่มีทางรู้เลยว่ามันเกิดขึ้น (เห็นแค่ response ที่ถูกต้อง แค่อาจจะ
"ช้าเกินคาด" ถ้าจำนวนแถวมากขึ้น) — ในระบบจริงที่ list มีหลักร้อย/หลักพันแถว ตัวเลข query ที่เพิ่มเป็นเส้นตรง
แบบนี้คือสาเหตุอันดับหนึ่งของ GraphQL API ที่ "ช้าอย่างไม่มีเหตุผล" ที่ทีม production เจอกันจริง

#### แก้ด้วย `async_graphql::dataloader::DataLoader`

กลไกแก้ปัญหานี้คือ **DataLoader** — แนวคิดที่ยืมมาจาก DataLoader ของ Facebook (ต้นกำเนิดเดียวกับ GraphQL เอง)
หลักการคือ: **อย่ายิง query ทันทีที่ resolver ถูกเรียก** แต่ให้**เก็บ key ที่ต้องการทั้งหมดไว้ก่อน** (รอจนกว่า
resolver ทุกตัวใน "รอบ" เดียวกันจะถูกเรียกจบ) แล้ว**ยิง query เดียวที่ดึงข้อมูลของทุก key พร้อมกัน** จากนั้น
แจกจ่ายผลลัพธ์กลับไปให้ resolver ที่รออยู่แต่ละตัว — ทั้งหมดนี้เกิดขึ้น**โดยอัตโนมัติ** ผ่านกลไก batching ที่
`async-graphql` จัดการให้ ผู้เขียน resolver แค่เรียก `.load_one(key)` เหมือนกำลังขอค่าทีละตัวตามปกติ

ก่อนใช้งานต้องเพิ่ม feature `dataloader` ให้ `async-graphql` (ค่า default ไม่เปิดมาให้ — ดูกับดักข้อ 1 ท้ายบท
ที่พิสูจน์ error จริงถ้าลืมเปิด):

```bash
cargo add async-graphql --features dataloader
```

โค้ดที่แก้แล้ว (ทดสอบจริง คอมไพล์และรันจริง — โครงสร้าง schema เหมือนเดิมทุกอย่าง เปลี่ยนแค่วิธี resolve
`author`):

```rust
use async_graphql::dataloader::{DataLoader, Loader};
use async_graphql::{Context, Object};
use sqlx::PgPool;
use std::collections::HashMap;
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;

# #[derive(Debug, Clone, async_graphql::SimpleObject, sqlx::FromRow)]
# struct Author { id: i64, name: String, country: Option<String> }
# #[derive(Debug, Clone, sqlx::FromRow)]
# struct BookRow { id: i64, title: String, author_id: i64 }
struct Book(BookRow);

// ตัว loader เก็บสิ่งที่จำเป็นสำหรับ "ยิง query แบบ batch" ไว้ข้างใน -- ในที่นี้คือ
// PgPool (เพื่อคุยกับฐานข้อมูล) และ counter (สำหรับพิสูจน์ตัวเลขในบทนี้เท่านั้น ไม่ใช่
// ส่วนที่จำเป็นในโค้ด production จริง)
struct AuthorLoader {
    pool: PgPool,
    query_count: Arc<AtomicUsize>,
}

#[derive(Debug, Clone, thiserror::Error)]
enum LoaderError {
    #[error("database error: {0}")]
    Db(String),
}

// นี่คือหัวใจของ DataLoader: async-graphql เก็บทุก key ที่ .load_one() ถูกเรียกในรอบ
// เดียวกัน แล้วเรียก load() ครั้งเดียวพร้อม key ทั้งหมด (เป็น &[i64] ไม่ใช่ i64 เดี่ยว ๆ)
impl Loader<i64> for AuthorLoader {
    type Value = Author;
    type Error = LoaderError;

    async fn load(&self, keys: &[i64]) -> Result<HashMap<i64, Self::Value>, Self::Error> {
        self.query_count.fetch_add(1, Ordering::SeqCst);
        // ANY($1) รับ array ของ id ทั้งหมดในครั้งเดียว -- ไม่ว่า keys จะมี 1 หรือ 1000
        // ตัวก็ยัง "หนึ่ง query" เสมอ (ต่างจากสร้าง SQL หลาย $N ตามจำนวน key ซึ่งจะเสีย
        // ประโยชน์ของ prepared statement caching ไป)
        let authors = sqlx::query_as::<_, Author>(
            "SELECT id, name, country FROM authors WHERE id = ANY($1)",
        )
        .bind(keys)
        .fetch_all(&self.pool)
        .await
        .map_err(|e| LoaderError::Db(e.to_string()))?;
        // ต้องคืนเป็น HashMap<key, value> -- async-graphql ใช้ map นี้จับคู่ผลลัพธ์
        // กลับไปให้ .load_one(key) แต่ละตัวที่รออยู่ ไม่สนใจลำดับที่ query คืนกลับมา
        Ok(authors.into_iter().map(|a| (a.id, a)).collect())
    }
}

#[Object]
impl Book {
    async fn id(&self) -> i64 {
        self.0.id
    }

    async fn title(&self) -> &str {
        &self.0.title
    }

    // FIXED: ขอ DataLoader ให้ช่วยดึงค่าแทนการยิง query ตรง ๆ เอง -- โค้ดฝั่ง resolver
    // ดู "เหมือนกับ" กำลังขอค่าทีละตัวปกติ (load_one) แต่เบื้องหลัง async-graphql
    // จะรวบรวม author_id ของทุกเล่มที่ถูก resolve ในรอบเดียวกัน แล้วเรียก
    // AuthorLoader::load ครั้งเดียวพร้อม key ทั้งหมด
    async fn author(&self, ctx: &Context<'_>) -> async_graphql::Result<Option<Author>> {
        let loader = ctx.data::<DataLoader<AuthorLoader>>()?;
        let author = loader.load_one(self.0.author_id).await?;
        Ok(author)
    }
}
```

ตอน setup schema ต้องสร้าง `DataLoader` แล้วส่งเข้าไปเป็น context data (เหมือน `PgPool` ทุกประการ):

```rust
# use async_graphql::dataloader::DataLoader;
# use sqlx::PgPool;
# use std::sync::atomic::AtomicUsize;
# use std::sync::Arc;
# struct AuthorLoader { pool: PgPool, query_count: Arc<AtomicUsize> }
# async fn setup(pool: PgPool, query_count: Arc<AtomicUsize>) {
let author_loader = DataLoader::new(
    AuthorLoader {
        pool: pool.clone(),
        query_count: query_count.clone(),
    },
    tokio::spawn, // DataLoader ต้องรู้วิธี spawn task ของ runtime ที่ใช้ (Tokio ในบทนี้)
);
// จากนั้น .data(author_loader) เข้า schema เหมือนกับ .data(pool) ปกติ
# let _ = author_loader;
# }
```

**รันจริงซ้ำด้วย query เดิมเป๊ะ** (`{ books { title author { name } } }`) กับ server เวอร์ชันที่ใช้ DataLoader:

```bash
curl -s -X POST http://127.0.0.1:8082/graphql -H 'Content-Type: application/json' \
  -d '{"query":"{ books { title author { name } } }"}'
```

```json
{"data":{"books":[{"title":"A Wizard of Earthsea","author":{"name":"Ursula K. Le Guin"}},{"title":"The Left Hand of Darkness","author":{"name":"Ursula K. Le Guin"}},{"title":"Norwegian Wood","author":{"name":"Haruki Murakami"}},{"title":"Kafka on the Shore","author":{"name":"Haruki Murakami"}},{"title":"ช่างสำราญ","author":{"name":"เดือนวาด พิมวนา"}}]}}
```

**ผลลัพธ์ JSON เหมือนกันทุกไบต์กับเวอร์ชัน naive** (ถูกต้องแล้ว — DataLoader ไม่เปลี่ยนความหมายของข้อมูลเลย
แค่เปลี่ยนวิธีดึงข้อมูล) แต่ตัวเลข query count ต่างกันอย่างสิ้นเชิง:

```bash
curl -s -X POST http://127.0.0.1:8082/graphql -H 'Content-Type: application/json' \
  -d '{"query":"{ queryCount }"}'
```

```json
{"data":{"queryCount":2}}
```

**จาก 6 query เหลือ 2 query จริง** — ยืนยันด้วย log จริงจาก `sqlx` อีกครั้ง:

```text
DEBUG sqlx::query: summary="SELECT id, title, author_id …" db.statement="\n\nSELECT id, title, author_id FROM books ORDER BY id\n" rows_affected=5 rows_returned=5 elapsed=1.933069ms
DEBUG sqlx::query: summary="SELECT id, name, country …" db.statement="\n\nSELECT id, name, country FROM authors WHERE id = ANY($1)\n" rows_affected=3 rows_returned=3 elapsed=848.371µs
```

query ที่สองเปลี่ยนจาก `WHERE id = $1` (ยิงซ้ำ 5 ครั้ง) เป็น `WHERE id = ANY($1)` (ยิง**ครั้งเดียว**) และ
`rows_returned=3` (ไม่ใช่ 5) เพราะหนังสือ 5 เล่มมี author ที่**ไม่ซ้ำกันจริง**แค่ 3 คน (Le Guin ซ้ำ 2 เล่ม,
Murakami ซ้ำ 2 เล่ม) — DataLoader มี**deduplication ในตัว**: ถ้า key ซ้ำกันหลายครั้งในรอบเดียว มันจะยิง query
ขอแค่ key ที่ไม่ซ้ำเท่านั้น แล้วแจกผลลัพธ์เดียวกันกลับไปให้ทุกจุดที่ขอ key นั้นซ้ำ — นี่คือเหตุผลที่ query
count ไม่ได้แค่ "ลดจาก N เหลือ 1" แต่ยังประหยัดมากกว่านั้นอีกในกรณีที่มี key ซ้ำกันบ่อย (สถานการณ์ปกติมากใน
โลกจริง เช่น หนังสือหลายเล่มโดยผู้แต่งคนเดียวกัน หรือ comment หลายร้อยรายการจาก user คนเดียวกัน)

**ตารางสรุปผลการทดลองจริง:**

| | Naive resolver | DataLoader |
|---|---|---|
| Query สำหรับ list หนังสือ | 1 | 1 |
| Query สำหรับ author (5 เล่ม, 3 author ไม่ซ้ำ) | 5 (ยิงซ้ำทุกเล่ม) | 1 (batched, deduplicated) |
| รวม query ทั้งหมด | **6** | **2** |
| ความซับซ้อนของโค้ด resolver | ต่ำ (`fetch_one` ตรง ๆ) | สูงขึ้นเล็กน้อย (ต้องเขียน `Loader` trait) |
| Scaling ตามจำนวนแถว | เส้นตรง (`O(n)` query) | คงที่ (`O(1)` query ต่อรอบ ไม่ว่า n จะเท่าไหร่) |

**ข้อคิดสำคัญที่ต้องจำไว้เป็นกฎทองของ GraphQL production**: **ทุกครั้งที่เขียน `#[Object]` resolver ที่ยิง
query/เรียก service ข้ามระบบ ให้ถามตัวเองก่อนเสมอว่า "field นี้จะถูก resolve กี่ครั้งต่อ 1 คำขอจาก client"**
— ถ้าคำตอบคือ "ขึ้นกับจำนวนแถวใน list ที่ล้อมมันอยู่" (แปลว่ามันคือ field ของ type ที่ปรากฏอยู่ใน list) แทบจะ
เดาได้ทันทีว่าต้องใช้ DataLoader ไม่ใช่การยิง query ตรง ๆ — field ที่ไม่ได้อยู่ใน list (เช่น `book(id)` เดี่ยว ๆ
ในหัวข้อ 79.3) ไม่มีปัญหานี้ เพราะถูก resolve แค่ครั้งเดียวอยู่แล้วโดยธรรมชาติ

### 79.7 Mutations: `createBooking` พร้อม Validation และ Error Mapping

#### `#[Object]` สำหรับ Mutation Root: เหมือน Query Root ทุกประการ

Mutation root เขียนด้วย `#[Object]` แบบเดียวกับ Query root เป๊ะ — ความต่างมีแค่**ที่ตำแหน่งใน
`Schema::build(Query, Mutation, Subscription)`** และ**ธรรมเนียมการตั้งชื่อ** (มักขึ้นต้นด้วยคำกริยา เช่น
`createBooking`, `cancelBooking`, `updateBook`) — engine ของ `async-graphql` ไม่บังคับว่า mutation ต้องมี
side effect จริง (ไม่มีการตรวจสอบระดับ type system) แต่**ธรรมเนียมที่ทุกทีมควรทำตาม**คือ query ใช้อ่านอย่าง
เดียว mutation ใช้เปลี่ยนสถานะเท่านั้น — คล้ายกับหลัก safe/idempotent ของ HTTP method ที่ Part 61 สอนไว้ เพียง
แต่ GraphQL ไม่มีกลไกบังคับระดับ protocol เหมือน HTTP method ให้ (แยกแค่ query/mutation สองกลุ่มกว้าง ๆ ไม่ได้
แยกละเอียดเป็น `PUT` vs `POST` vs `DELETE` แบบ REST)

#### Domain: `createBooking` — ยืมหนังสือด้วย Transaction จริง

`createBooking` ต้องทำสองอย่างให้ atomic พร้อมกัน (แบบเดียวกับตัวอย่าง borrow record ของ Part 70): ลด
`available_copies` ของหนังสือลง 1 และสร้างแถวใหม่ในตาราง `bookings` — ใช้ `pool.begin()`/`.commit()` ตาม
pattern transaction ที่ Part 70/71 สอนไว้ทุกประการ:

```rust
use async_graphql::{Context, Object, SimpleObject};
use sqlx::PgPool;

#[derive(Debug, Clone, SimpleObject, sqlx::FromRow)]
struct Booking {
    id: i64,
    book_id: i64,
    borrower_name: String,
}

struct Mutation;

#[Object]
impl Mutation {
    async fn create_booking(
        &self,
        ctx: &Context<'_>,
        book_id: i64,
        borrower_name: String,
    ) -> async_graphql::Result<Booking> {
        // validate ก่อนแตะฐานข้อมูลเลย -- fail-fast แบบเดียวกับที่ Part 66 สอนไว้เรื่อง
        // การตรวจ input ที่ผิดแน่ ๆ ตั้งแต่ต้น ก่อนเสียเวลาเปิด transaction
        if borrower_name.trim().is_empty() {
            return Err(AppError::Validation("borrowerName must not be empty".into()).extend());
        }

        let pool = ctx.data::<PgPool>()?;
        let mut tx = pool.begin().await.map_err(AppError::Database)?;

        // FOR UPDATE ล็อกแถวนี้ไว้จนกว่า transaction จะ commit/rollback -- ป้องกัน race
        // condition ที่สอง request พร้อมกันอ่าน available_copies เดิม แล้วลดค่าซ้อนกัน
        // (แนวคิดเดียวกับที่ Part 71 อธิบายเรื่อง row-level locking)
        let available: Option<i32> = sqlx::query_scalar(
            "SELECT available_copies FROM books WHERE id = $1 FOR UPDATE",
        )
        .bind(book_id)
        .fetch_optional(&mut *tx)
        .await
        .map_err(AppError::Database)?;

        let available = match available {
            Some(n) => n,
            None => return Err(AppError::BookNotFound(book_id).extend()),
        };

        if available <= 0 {
            return Err(AppError::NoCopiesAvailable(book_id).extend());
        }

        sqlx::query("UPDATE books SET available_copies = available_copies - 1 WHERE id = $1")
            .bind(book_id)
            .execute(&mut *tx)
            .await
            .map_err(AppError::Database)?;

        let booking = sqlx::query_as::<_, Booking>(
            "INSERT INTO bookings (book_id, borrower_name) VALUES ($1, $2)
             RETURNING id, book_id, borrower_name",
        )
        .bind(book_id)
        .bind(&borrower_name)
        .fetch_one(&mut *tx)
        .await
        .map_err(AppError::Database)?;

        tx.commit().await.map_err(AppError::Database)?;
        Ok(booking)
    }
}
```

#### แปลง `AppError` เป็น GraphQL Error ด้วย `ErrorExtensions`

Part 66 สอนไว้ว่า REST handler แปลง `AppError` เป็น HTTP status code ที่เหมาะสม (404, 409, 422, ...) ผ่าน
`impl IntoResponse for AppError` — GraphQL **ไม่มีแนวคิด "status code หลากหลาย" แบบนั้นเลย** (ดูกับดักข้อ 3
ท้ายบทที่อธิบายเรื่องนี้ให้ครบ) สิ่งที่ GraphQL มีให้แทนคือ **extension field** บน error object — ค่า key-value
เพิ่มเติมที่แนบไปกับ error message เพื่อให้ client แยกแยะ "ชนิดของ error" ได้โดยไม่ต้อง parse ข้อความ:

```rust
use async_graphql::ErrorExtensions;

#[derive(Debug, thiserror::Error)]
enum AppError {
    #[error("book {0} not found")]
    BookNotFound(i64),
    #[error("book {0} has no available copies")]
    NoCopiesAvailable(i64),
    #[error("validation failed: {0}")]
    Validation(String),
    #[error("database error: {0}")]
    Database(#[from] sqlx::Error),
}

// นี่คือ "REST status code mapping" ของ Part 66 เวอร์ชัน GraphQL: แทนที่จะเลือก
// (404, 409, 422, 500) เราติด `code` string ที่ client อ่านได้เข้าไปใน error
// extensions แทน -- แนวคิดเดียวกัน (แยกประเภท error ให้ client จัดการต่อได้อย่างมี
// โครงสร้าง) กลไกที่ต่างกันเพราะข้อจำกัดของ GraphQL error model เอง
impl ErrorExtensions for AppError {
    fn extend(&self) -> async_graphql::Error {
        async_graphql::Error::new(self.to_string()).extend_with(|_, e| {
            let code = match self {
                AppError::BookNotFound(_) => "BOOK_NOT_FOUND",
                AppError::NoCopiesAvailable(_) => "NO_COPIES_AVAILABLE",
                AppError::Validation(_) => "VALIDATION_ERROR",
                AppError::Database(_) => "INTERNAL_ERROR",
            };
            e.set("code", code);
        })
    }
}
```

`.extend()` คืนค่า `async_graphql::Error` ที่เอาไปใช้เป็น `Err(...)` ของ resolver ได้ตรง ๆ (ตามที่เห็นในโค้ด
`create_booking` ข้างบน) — สังเกตว่า `AppError::Database` มาจาก `#[from] sqlx::Error` (ตาม pattern
`impl From<sqlx::Error> for AppError` ที่ Part 70 สอนไว้) แต่ในโค้ดข้างบนเราเรียก `.map_err(AppError::Database)`
ตรง ๆ แทนใช้ `?` เพราะจุด return เป็น `async_graphql::Result` ไม่ใช่ `Result<_, AppError>` — ยังใช้
`#[from]`/`thiserror` (Part 30-31) เป็นฐานเหมือนเดิมทุกประการ เพียงแต่จุดแปลง error สุดท้ายเปลี่ยนปลายทางจาก
"HTTP status" (Part 66) เป็น "GraphQL error extension" เท่านั้น

#### ทดสอบจริงครบทุกกรณี: สำเร็จ, Validation ผิด, ไม่พบหนังสือ, หนังสือหมด

**ก่อนทำ booking**: ตรวจสอบ `available_copies` ของหนังสือ id 1 ในฐานข้อมูลจริง:

```text
 id |         title        | available_copies
----+-----------------------+------------------
  1 | A Wizard of Earthsea  |                2
```

**สำเร็จ**:

```bash
curl -s -X POST http://127.0.0.1:8083/graphql -H 'Content-Type: application/json' \
  -d '{"query":"mutation($bookId: Int!, $name: String!) { createBooking(bookId: $bookId, borrowerName: $name) { id bookId borrowerName } }","variables":{"bookId":1,"name":"สมชาย ใจดี"}}'
```

```json
{"data":{"createBooking":{"id":1,"bookId":1,"borrowerName":"สมชาย ใจดี"}}}
```

**ตรวจฐานข้อมูลอีกครั้งหลัง mutation** — `available_copies` ลดจาก 2 เหลือ 1 จริง (transaction ทำงานถูกต้อง):

```text
 id |        title         | available_copies
----+-----------------------+------------------
  1 | A Wizard of Earthsea  |                1
```

**Validation ล้มเหลว** (`borrowerName` เป็นค่าว่าง):

```bash
curl -s -X POST http://127.0.0.1:8083/graphql -H 'Content-Type: application/json' \
  -d '{"query":"mutation { createBooking(bookId: 1, borrowerName: \"\") { id } }"}'
```

```json
{"data":null,"errors":[{"message":"validation failed: borrowerName must not be empty","locations":[{"line":1,"column":12}],"path":["createBooking"],"extensions":{"code":"VALIDATION_ERROR"}}]}
```

**ไม่พบหนังสือ** (`bookId: 9999` ไม่มีอยู่จริง):

```bash
curl -s -X POST http://127.0.0.1:8083/graphql -H 'Content-Type: application/json' \
  -d '{"query":"mutation { createBooking(bookId: 9999, borrowerName: \"Somchai\") { id } }"}'
```

```json
{"data":null,"errors":[{"message":"book 9999 not found","locations":[{"line":1,"column":12}],"path":["createBooking"],"extensions":{"code":"BOOK_NOT_FOUND"}}]}
```

**หนังสือหมด** (`bookId: 4` มี `available_copies = 0` ตามข้อมูลที่ seed ไว้):

```bash
curl -s -X POST http://127.0.0.1:8083/graphql -H 'Content-Type: application/json' \
  -d '{"query":"mutation { createBooking(bookId: 4, borrowerName: \"Somchai\") { id } }"}'
```

```json
{"data":null,"errors":[{"message":"book 4 has no available copies","locations":[{"line":1,"column":12}],"path":["createBooking"],"extensions":{"code":"NO_COPIES_AVAILABLE"}}]}
```

สังเกตรูปแบบที่**เหมือนกันทุก error**: `data` เป็น `null`, `errors` เป็น array ที่มีอย่างน้อย 1 object ข้างใน
มี `message` (ข้อความอธิบาย), `locations` (ตำแหน่งใน query ที่ทำให้เกิด error), `path` (field ไหนที่ error
เกิดขึ้น), และ `extensions.code` (รหัสที่เราติดเข้าไปเองผ่าน `ErrorExtensions`) — client ที่ดีจะเช็ค
`extensions.code` เพื่อตัดสินใจ (เช่น โชว์ข้อความ "หนังสือเล่มนี้ถูกยืมหมดแล้ว" เมื่อเจอ `NO_COPIES_AVAILABLE`)
แทนการ parse `message` ที่เป็นข้อความสำหรับมนุษย์อ่านเป็นหลัก ไม่ใช่ contract ที่ควร parse ด้วยโค้ด

### 79.8 เชื่อมต่อฐานข้อมูล: `ctx.data::<PgPool>()` เทียบกับ `State<T>` ของ Axum

ตลอดบทนี้ resolver ทุกตัวดึง `PgPool` ผ่าน `ctx.data::<PgPool>()` — คำถามที่ควรถามตรง ๆ คือ: **นี่ต่างจาก
`State<T>` extractor ของ Axum (Part 64) อย่างไร ในเมื่อทั้งคู่ทำสิ่งเดียวกันในทางแนวคิด (ดึง dependency ที่แชร์
ไว้ล่วงหน้าออกมาใช้)?**

**จุดที่เหมือนกัน**: ทั้งสองใช้หลักการเดียวกันคือ**สร้าง resource ที่แพง (เช่น connection pool) ไว้ล่วงหน้า
ครั้งเดียวตอน startup แล้วแชร์ใช้ซ้ำข้าม request** ตามที่ Part 39/70 อธิบายเหตุผลไว้แล้ว — ทั้ง `State<T>` และ
`ctx.data::<T>()` ทำงานอยู่บนฐาน `Arc` (หรือกลไก type-erased ที่คล้ายกัน) เพื่อแชร์ค่าเดียวกันข้าม task/thread
อย่างปลอดภัย (Part 39-40)

**จุดที่ต่างกัน**:

| | Axum `State<T>` (REST, Part 64) | `async-graphql` `ctx.data::<T>()` |
|---|---|---|
| ใครเป็นคน "ส่ง" dependency เข้ามา | `Router::with_state(app_state)` — ผูกกับ router ตอน build | `Schema::build(...).data(value).finish()` — ผูกกับ schema ตอน build |
| การตรวจสอบ type | **Compile-time เต็มรูปแบบ** — `State<T>` ผิด type คือ compile error ทันที | **Runtime** — `ctx.data::<T>()` คืน `Result`, ถ้า type ที่ขอไม่ตรงกับที่ลงทะเบียนไว้ จะได้ `Err` ตอนรัน ไม่ใช่ compile error (ดูกับดักข้อ 3 ท้ายบท) |
| จำนวนจุดที่เข้าถึงได้ | ทุก handler ที่ประกาศ `State<T>` เป็น parameter | ทุก resolver (`#[Object]` method) ที่มี `ctx: &Context<'_>` เป็น parameter |
| ต้องประกาศ dependency กี่ตัว | หนึ่ง `AppState` struct ที่รวมทุกอย่าง (ตาม pattern Part 64) หรือหลาย `State<T>` แยกกันก็ได้ | เรียก `.data(x)` ได้หลายครั้ง ต่างชนิดกัน ไม่ต้องรวมเป็น struct เดียว |

เหตุผลที่ `ctx.data::<T>()` ต้องเป็น runtime check (ต่างจาก `State<T>`) มาจากข้อจำกัดของ `#[Object]` macro
เอง: schema ถูก generate แบบ generic ครอบคลุมทุก resolver พร้อมกัน โดยไม่รู้ล่วงหน้าว่า resolver ตัวไหนจะขอ
type ไหนจาก context บ้าง (ต่างจาก Axum ที่แต่ละ handler function ประกาศ parameter ของตัวเองชัดเจนตอน compile
ทำให้ router ตรวจสอบได้ตั้งแต่ตอน build) — นี่คือ trade-off ที่ยอมรับได้ในทางปฏิบัติ เพราะจำนวน type ที่ resolver
ต้องการ (`PgPool`, `DataLoader<AuthorLoader>`, ...) มักคงที่และรู้ล่วงหน้าตั้งแต่ตอนออกแบบ schema แล้ว — ความ
เสี่ยงจะเกิดขึ้นก็ตอนที่**ลืมลงทะเบียน** type ใด type หนึ่งเข้า schema เท่านั้น (ซึ่งกับดักข้อ 3 จะพิสูจน์ error
จริงให้เห็น)

### 79.9 GraphiQL: เครื่องมือ Explore แบบ Interactive

การพัฒนา REST API (Part 61-78) มักใช้ `curl` หรือ Postman ยิงทดสอบทีละ endpoint — สำหรับ GraphQL มีเครื่องมือ
เฉพาะทางที่ช่วยได้มากกว่านั้นมาก เพราะ schema ของ GraphQL **บอกตัวเองได้ว่ามันมี field อะไรบ้าง** ผ่านกลไกที่
เรียกว่า **introspection** (schema ตอบคำถามเกี่ยวกับตัวเองได้ผ่าน query พิเศษที่ engine สร้างให้อัตโนมัติ ไม่
ต้องเขียนเอง) — **GraphiQL** คือหน้าเว็บ interactive ที่ใช้ introspection นี้มาสร้างประสบการณ์คล้าย IDE:

- **Schema explorer แบบ sidebar**: เห็นทุก type, ทุก field, ทุก argument พร้อม description (ถ้ามีการเขียนไว้ใน
  โค้ด Rust ผ่าน doc comment — `async-graphql` อ่าน `///` เหนือ resolver แล้วใส่เป็น description ของ field นั้น
  ใน schema โดยอัตโนมัติ)
- **Autocomplete**: พิมพ์ query ไปครึ่งทาง แล้วกด autocomplete จะเห็น field ที่เป็นไปได้ทั้งหมดของ type ณ
  ตำแหน่งนั้น (รู้ได้จาก schema ที่ introspect มา) — ลดโอกาสพิมพ์ชื่อ field ผิดได้มาก (ปัญหาที่กับดักข้อ 5 จะ
  พิสูจน์ error จริงถ้าพิมพ์ผิดโดยไม่มี autocomplete ช่วย)
- **Query history และ variable editor**: มีช่องแยกสำหรับเขียน `variables` เป็น JSON แทนต้องฝัง inline ในตัว
  query (ตามที่หัวข้อ 79.5 อธิบายว่าทำไม `variables` ดีกว่า inline argument ในโปรเจกต์จริง)
- **แสดง response พร้อม syntax highlighting**: เห็น JSON ที่ตอบกลับมาจัดรูปแบบสวยงาม อ่านง่ายกว่า raw JSON ที่
  `curl` พ่นออกมาตรง ๆ มาก

การ mount GraphiQL เข้ากับ Axum ทำผ่าน `async_graphql::http::GraphiQLSource` — มันคืนแค่ HTML string ตัวหนึ่ง
(หน้าเว็บ static ที่ฝัง JavaScript ของ GraphiQL ไว้ข้างใน ไม่ต้องพึ่ง asset แยกไฟล์) แล้วเราแค่ห่อมันด้วย
`axum::response::Html` เพื่อให้ Axum ส่ง content-type ที่ถูกต้อง:

```rust
use async_graphql::http::GraphiQLSource;
use axum::response::{Html, IntoResponse};

async fn graphiql() -> impl IntoResponse {
    Html(GraphiQLSource::build().endpoint("/graphql").finish())
}
```

`.endpoint("/graphql")` บอก GraphiQL ว่าเวลามันยิง query จาก UI ให้ยิงไปที่ URL ไหน (ตรงกับ handler
`graphql_handler` ในหัวข้อ 79.3) — วิธีที่นิยมที่สุดคือ mount ทั้งสอง handler ไว้ที่ **path เดียวกัน** แยกกันด้วย
HTTP method: `GET /graphql` คืนหน้า GraphiQL (สำหรับเปิดผ่าน browser), `POST /graphql` รับ query จริง (สำหรับ
ทั้ง browser ที่คลิกจาก GraphiQL และ client อื่น ๆ เช่น `curl`, mobile app):

```rust
# use axum::routing::get;
# use axum::Router;
# fn app() -> Router {
Router::new().route("/graphql", get(/* graphiql handler */ async || "").post(/* graphql handler */ async || ""))
# }
```

**ทดสอบจริง**: ยิง `GET /graphql` ด้วย `curl` แล้วดู header ของ response (ไม่ดึงตัว HTML เต็มมาแสดงในบทนี้
เพราะยาวเกินความจำเป็น — แค่ยืนยันว่ามันตอบกลับมาเป็นหน้าเว็บจริง):

```bash
curl -s -D - -o /dev/null http://127.0.0.1:8079/graphql
```

```text
HTTP/1.1 200 OK
content-type: text/html; charset=utf-8
content-length: 1730
date: Sun, 27 Sep 2026 01:17:08 GMT
```

`content-type: text/html` ยืนยันว่ามันคือหน้าเว็บจริง (ไม่ใช่ JSON แบบ endpoint GraphQL ปกติ) — เปิด URL
`http://127.0.0.1:8079/graphql` ผ่าน browser จริงจะเห็นหน้า GraphiQL แบบเต็มรูปแบบ มี panel สามส่วนเรียงกัน
(sidebar schema explorer ทางซ้าย, editor เขียน query ตรงกลาง พร้อม autocomplete แบบ IDE, panel แสดงผลลัพธ์
ทางขวา) — เพราะบทเรียนนี้เป็น text-only ไม่มีภาพหน้าจอมาแสดง แต่ลักษณะการทำงานตรงตามที่อธิบายไว้ข้างบนทุก
ประการ (พิสูจน์แล้วว่า endpoint ตอบ HTML จริง ไม่ใช่แค่คำอธิบายเชิงทฤษฎี)

**คำเตือนสำหรับ production**: GraphiQL (และ introspection ที่มันพึ่งพา) **เปิดเผยโครงสร้าง schema ทั้งหมด**
ให้ใครก็ตามที่เข้าถึง endpoint ได้ — ในหลายทีมจึงปิด GraphiQL (และ/หรือปิด introspection query เอง) ใน
production environment โดยเปิดไว้แค่ dev/staging เท่านั้น (คล้ายกับที่หลายทีมปิด Swagger UI ของ REST API ใน
production แต่ด้วยเหตุผลที่หนักกว่าเล็กน้อย เพราะ introspection ของ GraphQL ละเอียดกว่า OpenAPI spec ทั่วไปมาก
— เห็นทุก field ทุก type ที่มีอยู่ในระบบ แม้ field นั้นจะยังไม่มี client ตัวไหนใช้จริงก็ตาม)

### 79.10 Subscriptions โดยสังเขป: GraphQL บน WebSocket ที่ Part 77 สอนไว้แล้ว

ส่วนที่สามของ schema (นอกจาก Query กับ Mutation) คือ **Subscription** — ใช้เมื่อ client ต้องการ **รับข้อมูล
แบบ real-time ทุกครั้งที่มีเหตุการณ์เกิดขึ้น** โดยไม่ต้อง poll (ยิง query ซ้ำ ๆ เป็นระยะ) เอง — ตัวอย่างในโดเมน
ห้องสมุด: client อยากรู้ **ทันทีที่หนังสือเล่มหนึ่งมีคนคืนแล้วว่างให้ยืมได้อีก** โดยไม่ต้องกด refresh หน้าเว็บ
เอง

**ข้อควรเข้าใจให้ชัดที่สุด**: subscription ของ GraphQL **ไม่ใช่ protocol ใหม่** — มันคือ **WebSocket
connection** (ตามที่ Part 77 สอนไว้ทั้งหมด: HTTP upgrade handshake, full-duplex message passing) ที่ห่อ
message ด้วยรูปแบบของ GraphQL อีกชั้นหนึ่งเท่านั้น เมื่อ client เปิด subscription จริง ๆ สิ่งที่เกิดขึ้นคือ:

1. Client ยิง WebSocket upgrade request ไปยัง endpoint เดียวกัน (`/graphql`) — Axum ทำ upgrade เดียวกันเป๊ะกับ
   ที่ Part 77 สอนไว้เรื่อง `WebSocketUpgrade` extractor
2. เมื่อ upgrade สำเร็จ ทั้งสองฝั่งคุยกันด้วย **sub-protocol ของ GraphQL over WebSocket** (มีสองมาตรฐานหลักที่
   ใช้กันในวงการ: `graphql-ws` และ `subscriptions-transport-ws` รุ่นเก่ากว่า — `async-graphql-axum` รองรับทั้ง
   คู่ผ่าน `GraphQLSubscription` handler)
3. Client ส่ง message บอกว่า "อยากสมัครรับ subscription นี้ พร้อม argument เท่านี้"
4. Server เก็บ subscription นั้นไว้ แล้ว**ส่ง message ใหม่กลับไปทุกครั้งที่ resolver ของ subscription นั้น
   ผลิตค่าใหม่ออกมา** (ผ่าน `Stream` ของ Rust — ตาม Part 46-48 ที่สอนเรื่อง async stream)

ฝั่ง resolver ของ subscription เขียนด้วย `#[Subscription]` (ไม่ใช่ `#[Object]`) และ**ต้องคืนค่าเป็น
`impl Stream<Item = T>`** แทน `T` เฉย ๆ — ตัวอย่างที่เรียบง่ายที่สุดที่ยัง**compile ได้จริง** (ทดสอบแล้ว) เพื่อ
ให้เห็นรูปร่างของโค้ด (ในระบบจริง จะเชื่อมกับ channel ภายในที่ mutation อย่าง `createBooking` ส่งเหตุการณ์เข้า
มา แทนการ poll ด้วย timer แบบตัวอย่างนี้):

```rust
use async_graphql::{Context, Subscription};
use futures_util::stream::Stream;
use std::time::Duration;
use tokio_stream::StreamExt as _;

struct SubscriptionRoot;

#[Subscription]
impl SubscriptionRoot {
    /// ตัวอย่างแบบง่ายที่สุด: ส่งค่า `book_id` กลับไปทุก 1 วินาที รวม 3 ครั้งแล้วปิด
    /// stream เอง -- ระบบจริงจะแทนที่ interval ด้วยการฟัง event จาก mutation (เช่นผ่าน
    /// `tokio::sync::broadcast` channel ที่ createBooking/returnBook ส่งเข้ามาเมื่อ
    /// available_copies เปลี่ยนค่า) แทนการ poll ด้วยเวลา
    async fn book_status_changed(
        &self,
        _ctx: &Context<'_>,
        book_id: i64,
    ) -> impl Stream<Item = i32> {
        tokio_stream::wrappers::IntervalStream::new(tokio::time::interval(Duration::from_secs(1)))
            .map(move |_| book_id as i32)
            .take(3)
    }
}
```

โค้ดนี้ compile ผ่านจริง (ทดสอบด้วยการ build schema ที่มี `SubscriptionRoot` เป็นส่วนที่สามแทน
`EmptySubscription` ที่ใช้มาตลอดบทนี้) แต่**บทนี้จะไม่ลงรายละเอียดการต่อ WebSocket ให้ครบวงจร** (การ mount
`GraphQLSubscription` handler เข้า Axum router, การจัดการ connection lifecycle, reconnect logic ฝั่ง client)
เพราะนั่นคือการนำความรู้ WebSocket ทั้งหมดของ Part 77 มาประกอบกับ schema/resolver ของบทนี้ — สิ่งที่สำคัญที่สุด
ที่ควรจำจากหัวข้อนี้คือ **subscription ไม่ใช่ความรู้ใหม่ทั้งหมด มันคือจุดที่ Part 77 (WebSocket) กับ Part 79
(GraphQL) มาบรรจบกัน** — ถ้าเข้าใจทั้งสองบทแยกกันดีแล้ว การต่อทั้งสองเข้าด้วยกันในโปรเจกต์จริงเป็นแค่งาน
integration ไม่ใช่แนวคิดใหม่ที่ต้องเรียนจากศูนย์

### 79.11 เมื่อไหร่ควรใช้ GraphQL เทียบกับ REST: มุมมองที่ตรงไปตรงมา

หลังจากเห็นทั้งข้อดี (แก้ over/under-fetching, client กำหนด shape เอง, schema ที่ตรวจสอบได้) และภาระที่แลกมา
(N+1 ระดับ resolver ที่ต้องคอยระวัง, ไม่มี HTTP-level caching, error model ที่ไม่มี status code หลากหลาย) มาถึง
คำถามที่สำคัญที่สุดของบทนี้: **แล้วเมื่อไหร่ควรเลือก GraphQL จริง ๆ ในโปรเจกต์จริง?**

#### ตารางเปรียบเทียบเชิงตัดสินใจ

| ปัจจัย | REST (Part 61-78) | GraphQL (บทนี้) |
|---|---|---|
| Client มีหน้าจอ/use case ที่ต้องการ field ต่างกันมาก (web, mobile, partner API) | ต้องออกแบบ endpoint variant หรือ query parameter เพิ่มเรื่อย ๆ ตามความต้องการที่โตขึ้น | Client เขียน query ของตัวเองได้ ไม่ต้องรอ backend เพิ่ม endpoint ใหม่ทุกครั้ง |
| จำนวน resource ที่เกี่ยวข้องกันเป็น graph ซับซ้อน (book → author → other books → reviews → ...) | เสี่ยง under-fetching/N+1 ระดับ client สูง (ตามหัวข้อ 79.1) | ยิงครั้งเดียวได้ข้อมูลทั้ง graph ตามความลึกที่ต้องการ |
| ต้องการ HTTP-level caching (CDN, browser cache ตาม URL, `ETag`/`Cache-Control` ที่ Part 61 สอน) | ทำได้ตรงไปตรงมา เพราะ URL แต่ละตัวคือ cache key ธรรมชาติ | ทำได้ยากกว่ามาก เพราะทุก query ไปที่ URL เดียวกัน (`POST /graphql`) ต้อง cache ด้วยกลไกอื่น เช่น cache ที่ query hash หรือ persisted query — ซับซ้อนกว่า REST ล้วน ๆ |
| Authorization ต้องเช็คระดับ field/resource ละเอียด (เชื่อม Part 76 RBAC) | เช็คที่ endpoint เป็นหลัก (เช่น "role นี้เรียก `DELETE /books/{id}` ได้ไหม") ตรงไปตรงมา จำนวนจุดเช็คเท่ากับจำนวน endpoint | ต้องเช็คได้ที่**ทุก field** ที่อาจ sensitive (เพราะ client ผสม field ไหนกับ field ไหนก็ได้ในคำขอเดียว) ทำให้จำนวนจุดที่ต้องเช็ค RBAC เยอะกว่ามาก และง่ายที่จะพลาดจุดใดจุดหนึ่งไป |
| ทีม/นักพัฒนาที่คุ้นเคย | REST เป็นที่รู้จักกว้างขวางที่สุด เครื่องมือ (monitoring, API gateway, load balancer) รองรับสมบูรณ์ที่สุด | ต้องเรียนรู้แนวคิดเพิ่ม (schema, resolver, N+1 เฉพาะของ GraphQL) และเครื่องมือ ecosystem ยังไม่กว้างเท่า REST ในหลายด้าน (เช่น API gateway บางตัวยังรองรับ REST ได้ดีกว่า GraphQL) |
| ขนาดและความซับซ้อนของ domain | เหมาะกับ CRUD ตรงไปตรงมา หรือ API สาธารณะที่ผู้ใช้ภายนอกจำนวนมากต้องเข้าใจง่าย | เหมาะกับ domain ที่ relationship ซับซ้อนจริง ๆ และมี client หลายแบบที่ต้องการ field ต่างกันมาก (เช่น social network graph, e-commerce ที่มี product/review/inventory/pricing เชื่อมกันซับซ้อน) |

#### เหตุผลเชิงลึกที่อยู่หลังตาราง: Authorization ที่ซับซ้อนขึ้นจริง ๆ

Part 76 สอนว่า RBAC ที่ดีต้องเช็ค "role นี้ทำ action นี้กับ resource นี้ได้ไหม" ก่อน handler จะทำงาน — ใน REST
จุดเช็คนี้ทำได้ที่ **middleware ระดับ route** (Part 65) เพราะรู้ล่วงหน้าแน่นอนว่า request หนึ่งครั้งเรียก
resource ตัวเดียว action เดียว (เช่น `DELETE /books/42` แปลว่า "ต้องการลบ resource `book` id 42") แต่ใน
GraphQL คำขอเดียวสามารถผสม field จากหลาย type ที่มีระดับความอ่อนไหวต่างกันได้ในครั้งเดียว เช่น query เดียว
อาจขอทั้ง `book.title` (สาธารณะ ใครดูก็ได้) และ `book.internalCostPrice` (ควรเห็นแค่ staff เท่านั้น) พร้อมกัน —
ไม่มีจุดเดียวที่ middleware เช็คได้ครบ ต้องเช็คสิทธิ์**ที่ตัว resolver ของแต่ละ field ที่อ่อนไหวเอง** (เช่น
resolver ของ `internal_cost_price` ต้องอ่าน role จาก `ctx` แล้วคืน `null` หรือ error ถ้าไม่มีสิทธิ์) — จำนวน
จุดที่ต้องเขียน authorization logic จึงมากกว่า REST อย่างมีนัยสำคัญในระบบที่มี field ละเอียดอ่อนกระจายอยู่ทั่ว
schema และการพลาดเช็คสิทธิ์ที่ field เดียวก็รั่วข้อมูลได้ทันที (ต่างจาก REST ที่พลาด middleware ที่ endpoint
เดียวก็ยังจำกัดความเสียหายอยู่ที่ endpoint นั้นเท่านั้น)

#### จุดยืนของหลักสูตรนี้: REST/Axum ยังคงเป็นตัวเลือกหลัก

ตามแนวทางเดียวกับที่ Part 69 (เทียบ web framework), Part 72-73 (เทียบ ORM/database library) วางไว้ — บทนี้
**ไม่ได้สอน GraphQL เพื่อเปลี่ยนให้หลักสูตรเลิกใช้ REST** สิ่งที่ Part 61-78 สอนไว้ทั้งหมด (HTTP semantics,
Axum, SQLx, error handling, authentication/authorization, API design) **ยังคงเป็นฐานหลักของหลักสูตรต่อจากนี้
ไป** — GraphQL (บทนี้) และ gRPC (Part 80 ถัดไป) เป็นสิ่งที่นักพัฒนา Rust backend ระดับมืออาชีพ**ควรรู้จักและ
รู้วิธีเขียนได้จริง** (เพราะจะเจอในโปรเจกต์จริงแน่นอนไม่ช้าก็เร็ว โดยเฉพาะถ้าทำงานกับทีม frontend ที่มีหลาย
client ต่างกัน หรือทำงานกับระบบ microservice ที่คุยกันเองภายใน) แต่**ไม่ใช่เครื่องมือหลักที่หลักสูตรจะใช้สอน
concept ใหม่ ๆ ต่อจากนี้** — บทถัดไปหลังจาก gRPC จะกลับไปใช้ REST/Axum เป็นฐานเหมือนเดิม ตามหลักการที่ว่า
"เลือกเครื่องมือที่เหมาะกับปัญหาจริง ไม่ใช่เลือกเพราะมันใหม่หรือดูซับซ้อนกว่า"

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. ลืมเปิด Feature `dataloader` แล้ว Import ไม่ผ่าน

`async_graphql::dataloader` module **ไม่ได้เปิดมาโดย default** — ถ้าเขียนโค้ดใช้ `DataLoader`/`Loader` โดยไม่
ได้เพิ่ม feature ก่อน จะเจอ compile error จริงแบบนี้ (ทดสอบจริง):

```text
error[E0432]: unresolved import `async_graphql::dataloader`
 --> src/bin/dataloader.rs:6:20
  |
6 | use async_graphql::dataloader::{DataLoader, Loader};
  |                    ^^^^^^^^^^ could not find `dataloader` in `async_graphql`
  |
note: found an item that was configured out
 --> .../async-graphql-7.2.1/src/lib.rs:201:9
  |
199 | #[cfg(feature = "dataloader")]
200 | #[cfg_attr(docsrs, doc(cfg(feature = "dataloader")))]
201 | pub mod dataloader;
  |         ^^^^^^^^^^
```

**วิธีแก้**: `cargo add async-graphql --features dataloader` (หรือแก้ `Cargo.toml` ให้มี
`features = [..., "dataloader"]`) — สังเกตว่า compiler เองบอกสาเหตุตรง ๆ ในบรรทัด `note:` ว่า item นี้ "ถูก
configure out" เพราะ feature ไม่เปิด นี่คือรูปแบบ error ที่ Rust แสดงเสมอเมื่อโค้ดพยายามใช้ของที่ถูกซ่อนไว้ด้วย
`#[cfg(feature = "...")]` — เจอ pattern นี้กับ crate อื่นได้อีกบ่อยมาก ไม่ใช่แค่ `async-graphql`

### 2. เขียน Resolver ที่ยิง Query ต่อแถวโดยไม่รู้ตัวว่าอยู่ใน List (N+1 ที่มองไม่เห็นจนสายเกินไป)

นี่คือกับดักที่**อันตรายที่สุด**เพราะ**ไม่มีอาการเตือนตอน compile หรือแม้แต่ตอน dev ด้วยข้อมูลน้อย ๆ** — โค้ด
ในหัวข้อ 79.6 ที่เขียน `author` resolver ยิง `SELECT ... WHERE id = $1` ตรง ๆ **compile ผ่านสมบูรณ์ ทำงาน
ถูกต้อง 100%** ทดสอบด้วยข้อมูล 5 แถวก็ดูเร็วดี (ผลต่างระหว่าง 6 query กับ 2 query ในการทดลองจริงของบทนี้วัดได้
เป็น millisecond เท่านั้น) — ปัญหาจะปะทุขึ้นก็ตอนข้อมูล production โตขึ้นเป็นหลักพัน/หมื่นแถว ตัวเลข query ที่
โตเป็นเส้นตรงตามจำนวนแถวจะกลายเป็นคอขวดจริงที่วัดผลกระทบได้ชัดเจนตอนนั้นแล้ว **วิธีป้องกัน**: ทุกครั้งที่เขียน
resolver ของ field ที่จะปรากฏใน list (ไม่ใช่ query เดี่ยว ๆ) และ resolver นั้นมี I/O (query ฐานข้อมูล, เรียก
service ภายนอก) ให้ตั้งคำถามเรื่อง DataLoader ตั้งแต่วันแรกที่เขียนโค้ด — อย่ารอให้ query count โตจนสังเกตได้
ด้วยตาก่อนถึงจะแก้ (การ retrofit DataLoader เข้าไปทีหลังในระบบใหญ่มักต้องแก้โครงสร้าง resolver หลายจุดพร้อมกัน)

### 3. เข้าใจผิดว่า GraphQL มี HTTP Status Code แบบ REST — ทุก Response คือ `200 OK`

นักพัฒนาที่คุ้นกับ REST (Part 61) มักคาดหวังว่า mutation ที่ validate ล้มเหลวจะได้ `400`/`422` กลับมา หรือ
mutation ที่หา resource ไม่เจอจะได้ `404` — **ความจริงคือ GraphQL response ส่งกลับมาเป็น `200 OK` เกือบทุกครั้ง
ไม่ว่าจะสำเร็จหรือ error** (ยกเว้น error ระดับ transport เช่น body ไม่ใช่ JSON เลย ที่จะได้ `400` จริง — แต่
error ทางธุรกิจ เช่น "ไม่พบหนังสือ" หรือ "validation ล้มเหลว" ไม่นับเป็นระดับนี้) พิสูจน์ด้วย `curl -i` จริง:

```text
HTTP/1.1 200 OK
content-type: application/graphql-response+json
content-length: 154

{"data":null,"errors":[{"message":"Unknown field \"is_available\" on type \"Book\". Did you mean \"isAvailable\"?","locations":[{"line":1,"column":17}]}]}
```

response นี้**เป็น error เต็มรูปแบบ** (field ชื่อผิด) แต่ status line คือ `HTTP/1.1 200 OK` ตรง ๆ — เหตุผลคือ
GraphQL มองว่า "การประมวลผล GraphQL request สำเร็จแล้ว" (HTTP layer ทำงานถูกต้องสมบูรณ์) แม้ผลลัพธ์ทาง
**ธุรกิจ**ของมันจะเป็น error ก็ตาม — สอง concept นี้ (HTTP transport สำเร็จ vs GraphQL operation สำเร็จ) แยก
จากกันโดยสิ้นเชิงใน GraphQL **วิธีแก้ที่ถูกต้อง**: โค้ดฝั่ง client (หรือ integration test) **ต้องเช็ค field
`errors` ใน body เสมอ** ไม่ใช่เช็คแค่ HTTP status code แบบที่ทำกับ REST — ถ้าใครในทีมยังเขียนโค้ดเช็ค
`if response.status() == 200 { /* สำเร็จ */ }` กับ GraphQL client จะมี bug ที่มองไม่เห็นทันที เพราะ error จริง
ก็ผ่านเงื่อนไขนี้ไปด้วย

### 4. `ctx.data::<T>()` หา Type ไม่เจอ เพราะลืมลงทะเบียนด้วย `.data(...)`

ต่างจาก Axum `State<T>` ที่ผิด type เป็น **compile error** (Part 64) — `ctx.data::<T>()` ของ `async-graphql`
เป็น**runtime check** (ตามที่หัวข้อ 79.8 อธิบายเหตุผลไว้) ถ้าลืม `.data(pool)` ตอน build schema แต่ resolver
ไปเรียก `ctx.data::<PgPool>()` โค้ดจะ**compile ผ่านสมบูรณ์** แล้วพังตอน runtime เท่านั้น — ทดสอบจริงด้วยการ
สร้าง schema ที่ไม่มี `.data(pool)` เลย แล้วเรียก resolver ที่ต้องการ `PgPool`:

```json
{"data":null,"errors":[{"message":"Data `sqlx_core::pool::Pool<sqlx_postgres::database::Postgres>` does not exist.","locations":[{"line":1,"column":3}],"path":["broken"]}]}
```

error message บอกชื่อ Rust type แบบเต็ม (`sqlx_core::pool::Pool<sqlx_postgres::database::Postgres>`) ตรงกับที่
resolver ขอ (`PgPool` เป็น type alias ของ `Pool<Postgres>`) — **วิธีป้องกัน**: ทำ checklist ทุกครั้งที่เพิ่ม
resolver ใหม่ที่เรียก `ctx.data::<T>()` ตัวใหม่ ให้ตรวจว่า `.data(...)` ของ type นั้นถูกเรียกตอน build schema
แล้วหรือยัง — เขียน integration test ที่ยิง query ทดสอบทุก resolver อย่างน้อยครั้งเดียว (ตาม Part 32-33) จะช่วย
จับ error แบบนี้ได้ตั้งแต่ CI ก่อนขึ้น production แทนที่จะไปพังตอน client จริงเรียกใช้

### 5. ลืมว่า `#[Object]`/`SimpleObject` แปลง `snake_case` เป็น `camelCase` ให้อัตโนมัติ

ธรรมเนียมของ GraphQL คือชื่อ field ใน schema เป็น **camelCase** (`isAvailable`, `publishedYear`) แต่ธรรมเนียม
ของ Rust คือ **snake_case** (`is_available`, `published_year`, ตาม Part 5 เรื่อง rustfmt/naming convention) —
`async-graphql` **แปลงให้อัตโนมัติ** (method ชื่อ `is_available` ใน Rust กลายเป็น field `isAvailable` ใน
schema) ซึ่งสะดวกมาก แต่ทำให้พลาดได้ง่ายถ้า query เขียนด้วยชื่อ snake_case ตามความเคยชินจากฝั่ง Rust — ทดสอบ
จริง:

```bash
curl -s -X POST http://127.0.0.1:8079/graphql -H 'Content-Type: application/json' \
  -d '{"query":"{ book(id: 3) { published_year } }"}'
```

```json
{"data":null,"errors":[{"message":"Unknown field \"published_year\" on type \"Book\". Did you mean \"publishedYear\"?","locations":[{"line":1,"column":17}]}]}
```

ข่าวดีคือ `async-graphql` **ใจดีพอที่จะแนะนำชื่อที่ถูกต้องให้** (`Did you mean "publishedYear"?`) — แต่ถ้า
ไม่มี GraphiQL/autocomplete ช่วยเตือนตั้งแต่ตอนเขียน query (หัวข้อ 79.9) อาจเสียเวลาสับสนได้พักหนึ่งว่าทำไม field
ที่ "เห็นชัด ๆ ว่ามีอยู่ในโค้ด Rust" กลับหาไม่เจอ — จำหลักไว้ว่า: **เขียน resolver ด้วย snake_case ตาม Rust
convention ตามปกติ แต่เขียน query ด้วย camelCase ตาม GraphQL convention เสมอ**

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เพิ่ม query ใหม่ชื่อ `authorCount` ใน `Query` root ที่คืนจำนวน author ทั้งหมดในตาราง `authors`
   เป็น `i64` (ใช้ `SELECT COUNT(*) FROM authors` ผ่าน `sqlx::query_scalar`) — ทดสอบด้วย `curl` ยิง
   `{ authorCount }` แล้วเทียบผลลัพธ์กับ `SELECT COUNT(*) FROM authors;` ที่รันตรงผ่าน `psql`
   *Hint*: `query_scalar` คืน `Result<i64, sqlx::Error>` ตรง ๆ ไม่ต้องมี struct ห่อ เหมือนที่ใช้ใน
   `create_booking` ตอนอ่าน `available_copies`

2. **(กลาง)** เพิ่ม field `books: Vec<Book>` ให้กับ type `Author` (จาก `SimpleObject` เดิม ต้องเปลี่ยนเป็น
   `#[Object]` เพราะ field นี้ต้องยิง query เพิ่ม ไม่ใช่ copy ตรงจาก struct) เพื่อให้ query แบบ
   `{ books { author { name books { title } } } }` ทำงานได้ (ผู้แต่งคนหนึ่งมีหนังสือได้หลายเล่ม) — ทดสอบว่า
   query นี้เกิด N+1 ปัญหาใหม่ที่ **ต่างทิศทาง** จากหัวข้อ 79.6 หรือไม่ (จาก book → author → books แทน
   book → author) แล้วลองแก้ด้วย DataLoader ตัวใหม่ (`BooksByAuthorLoader`) ที่ key เป็น `author_id` และ
   value เป็น `Vec<BookRow>`
   *Hint*: `Loader<i64>` ที่ `Value = Vec<BookRow>` ต้อง group ผลลัพธ์จาก query เดียวด้วย `author_id` เป็น
   `HashMap<i64, Vec<BookRow>>` เอง (ไม่ใช่ 1 key ต่อ 1 value ตรง ๆ แบบ `AuthorLoader` ในบทนี้)

3. **(ยาก)** เพิ่ม mutation `returnBook(bookingId: ID!)` ที่ทำสิ่งตรงข้ามกับ `createBooking`: หา `bookings` row
   จาก `bookingId`, เพิ่ม `available_copies` ของ `books` ที่เกี่ยวข้องขึ้น 1 (แต่ห้ามเกิน `total_copies` — ต้อง
   validate), แล้วลบแถวใน `bookings` ทิ้ง (หรือจะเก็บ field `returned_at` แทนการลบก็ได้ ถ้าต้องการเก็บ history)
   ทั้งหมดต้องอยู่ใน transaction เดียว และแปลง error กรณี `bookingId` ไม่พบ หรือ `available_copies` จะเกิน
   `total_copies` เป็น GraphQL error ที่มี `extensions.code` ตามแนวทางหัวข้อ 79.7
   *Hint*: ต้อง `JOIN` หรือ query สองรอบภายใน transaction เดียวกัน (หา `book_id` จาก `bookings` ก่อน แล้วค่อย
   `UPDATE books`) — ระวังเรื่อง `FOR UPDATE` บนแถว `books` เหมือนที่ `createBooking` ทำ เพื่อป้องกัน race
   condition แบบเดียวกัน

4. **(ยาก/ประยุกต์ใช้งานจริง)** เขียน middleware-like authorization check สำหรับ field `internalNotes` ที่เพิ่ม
   เข้าไปใน type `Book` (สมมติเป็นข้อความที่ staff เขียนโน้ตภายในเกี่ยวกับสภาพหนังสือ ผู้ใช้ทั่วไปไม่ควรเห็น) —
   ให้ resolver ของ field นี้อ่าน "role ปลอมของผู้ใช้" ที่ส่งผ่าน HTTP header `X-Role` (รับผ่าน
   `ctx.data::<HeaderRole>()` ที่ประกาศ `.data(...)` ตอน build schema จาก header ของ request จริง — จะต้องเขียน
   middleware ของ Axum ที่ดึง header ออกมาก่อนส่งต่อให้ `graphql_handler`) แล้วคืน `Some(notes)` เมื่อ role คือ
   `"staff"` และคืน `None` เมื่อไม่ใช่ — ทดสอบด้วย `curl` สองครั้งที่มี/ไม่มี header `X-Role: staff` แล้วเทียบ
   response ว่า field นี้เป็น `null`/มีค่าจริงตามที่คาด นี่คือรูปธรรมของสิ่งที่หัวข้อ 79.11 อธิบายไว้ว่า
   authorization ของ GraphQL ต้องเช็คที่ระดับ field ไม่ใช่ระดับ endpoint เหมือน REST (เชื่อมกับ Part 76 RBAC)
   *Hint*: ใช้ `Extension` หรือเขียน `axum::middleware::from_fn` ที่ดึง header แล้ว insert เข้า request
   extensions ก่อนส่งต่อให้ handler ของ GraphQL — จากนั้นใน `graphql_handler` ดึงค่านั้นมา `.data(role)` เข้ากับ
   `Request` ของ `async-graphql` ก่อน `.execute()` (ดู `GraphQLRequest::into_inner()` ที่ให้ปรับแต่ง request
   ก่อน execute ได้)

## สรุป

บทนี้เริ่มจากคำถามที่ REST (Part 61-78) ตอบได้ไม่เต็มที่: **over-fetching** (client ได้ field เกินที่ใช้จริง
เสมอ เพราะ shape ของ response ถูกกำหนดโดย server) และ **under-fetching/N+1 ระดับ client** (client ต้องยิงหลาย
request แยกกันเพื่อประกอบข้อมูลที่กระจายอยู่คนละ resource) — GraphQL แก้ทั้งสองปัญหานี้ด้วยแนวคิดที่ตรงไปตรงมา
มาก: **ให้ client เขียน query กำหนด shape ของ response ที่ต้องการเอง ในคำขอเดียว ผ่าน endpoint เดียว
(`POST /graphql`)** โดยมี **schema** เป็นสัญญาที่ตายตัวว่า field ไหนมี type อะไร และ **resolver** (เขียนด้วย
`#[Object]`/`#[derive(SimpleObject)]` ซึ่งเป็น proc macro ตาม Part 44-45) เป็นโค้ด Rust จริงที่อยู่หลัง field
แต่ละตัว

จุดที่สำคัญที่สุดในทางปฏิบัติที่บทนี้ย้ำหลายครั้งคือ: **GraphQL แก้ N+1 ระดับ client ได้ แต่ไม่ได้แก้ N+1 ระดับ
resolver-to-database ให้อัตโนมัติ** — เราพิสูจน์ด้วยตัวเลขจริงว่า resolver ที่เขียนไม่ระวังทำให้เกิด 6 query
สำหรับ 1 คำขอ (5 เล่ม + author ของแต่ละเล่มแยกกัน) และ `async_graphql::dataloader::DataLoader` ลดมันกลับมา
เหลือ 2 query จริง (1 list + 1 batched query แบบ `WHERE id = ANY($1)`) ผ่านการรวม (batch) และการลดจำนวนซ้ำ
(deduplicate) key ที่ resolver ขอในรอบเดียวกัน — นี่คือความรู้ที่**จำเป็นต้องมี**ก่อนเอา GraphQL ไปใช้งานจริง
ในระบบที่มีข้อมูลมากกว่าตัวอย่างเล็ก ๆ ในบทนี้

เราเชื่อมทุกความรู้เดิมของหลักสูตรเข้ากับ GraphQL ตลอดบท: `ctx.data::<PgPool>()` เทียบกับ `State<T>` (Part 64),
query argument เทียบกับ Axum extractor (Part 63), `ErrorExtensions` เทียบกับ `AppError`/HTTP status mapping
(Part 66), transaction ใน mutation ตรงกับ pattern ของ Part 70-71, และ authorization ระดับ field ที่ซับซ้อนกว่า
REST เพราะเชื่อมกับ RBAC ของ Part 76 — ปิดท้ายด้วยตารางตัดสินใจที่ตรงไปตรงมาว่า **GraphQL คุ้มกับความซับซ้อนที่
แลกมาเมื่อ domain มี relationship graph ซับซ้อนจริง ๆ และมี client หลายแบบที่ต้องการ field ต่างกันมาก** แต่
สำหรับ CRUD ตรงไปตรงมาหรือ API สาธารณะที่ต้องการ HTTP-level caching REST ยังคงเป็นตัวเลือกที่เรียบง่ายและ
เพียงพอกว่า — หลักสูตรนี้ยังคงใช้ REST/Axum เป็นฐานหลักต่อจากนี้ โดยมี GraphQL (บทนี้) และ gRPC (Part 80 ถัดไป)
เป็นเครื่องมือที่รู้จักและใช้เป็น ไม่ใช่ตัวเลือกหลักของหลักสูตร

Part ถัดไป (Part 80) จะพา RPC-style API อีกแบบมาเทียบ — **gRPC ด้วย Tonic** ที่ใช้ Protocol Buffers เป็น schema
และ HTTP/2 เป็น transport ต่างจากทั้ง REST และ GraphQL โดยสิ้นเชิงในหลายมิติ (binary serialization แทน JSON,
strongly-typed service contract แทน schema แบบ dynamic query) — เหมาะกับการสื่อสารระหว่าง microservice ภายใน
องค์กรที่ต้องการ performance สูงและ type safety ข้ามภาษาแบบเข้มงวด

---

**Part ก่อนหน้า:** [RESTful API Design Best Practices](part-078-rest-api-design.md) | **Part ถัดไป:** [gRPC ด้วย Tonic](part-080-grpc-tonic.md)
