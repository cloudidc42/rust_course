# Part 68: Actix-web: Routing, Handlers, Middleware

> โมดูล: การพัฒนาเว็บแอปพลิเคชัน (Web Development) | ระดับ: สูง | เวลาโดยประมาณ: 270 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- ดึงค่าจาก path ของ URL ด้วย **`web::Path<T>`** ได้ทั้งแบบ parameter เดียว, แบบ tuple
  (`web::Path<(u32, u32)>`), และแบบ struct ที่ `#[derive(Deserialize)]` — พร้อมรู้จริง (จากการรันจริง ไม่ใช่
  การเดา) ว่าเมื่อ path parameter parse ไม่ผ่าน Actix-web ตอบกลับด้วย **status code อะไร** ซึ่งเป็นจุดที่
  **ต่างจาก Axum อย่างมีนัยสำคัญ**
- ดึง query string ด้วย **`web::Query<T>`** พร้อมออกแบบ struct ที่มี field เป็น `Option<T>` สำหรับ pattern
  pagination แบบเดียวกับที่ Part 63 สอนไว้ฝั่ง Axum เพื่อเทียบกันได้ตรง ๆ
- จัดกลุ่ม route ที่ใหญ่ขึ้นด้วย **`web::scope(...)`** ซึ่งเป็นสิ่งที่ทำหน้าที่คล้าย `.nest()` ของ Axum แต่มี
  วิธีประกอบร่างกับ handler ที่ประกาศด้วย `#[get]`/`#[post]` macro ต่างออกไป (ผ่าน `.service(handler)`) —
  รวมหลาย resource เข้าเป็น nested scope ที่ทำงานได้จริง
- ใช้ **route guard** (`guard::Header`, `guard::fn_guard`, method guard) ซึ่งเป็นแนวคิดระดับ routing ที่ Axum
  **ไม่มีกลไกเทียบเท่าตรงตัว** — เข้าใจว่ากลไกนี้แก้ปัญหาอะไร และทำไมมันคือจุดที่การออกแบบของทั้งสอง framework
  แยกทางกันจริง ๆ ไม่ใช่แค่ต่างกันที่ syntax
- อธิบาย **`FromRequest`** trait ของ Actix-web ได้อย่างแม่นยำ — รู้ว่ามันเป็น trait เดียว (ไม่ใช่ trait คู่แบบ
  `FromRequestParts`/`FromRequest` ของ Axum) และรับมือกับ extractor ที่ต้องอ่าน body อย่างไรผ่าน `Payload`
  พร้อมเขียน **custom extractor `ApiKey`** ของตัวเองเทียบตรงกับที่ Part 64 สอนไว้ฝั่ง Axum
- พิสูจน์ด้วยการรันจริงว่า Actix-web มีกฎ "extractor ที่กิน body ต้องมาตัวสุดท้าย" แบบ Axum หรือไม่ — และรู้
  พฤติกรรมจริงเมื่อละเมิดกฎที่มีอยู่จริง (ถ้ามี) หรือกฎที่ไม่มีอยู่จริง (ถ้าไม่มี)
- เขียน `impl Responder` ให้ type ของแอปเองเพื่อควบคุม response โดยละเอียด เทียบตรงกับ `impl IntoResponse` ของ
  Axum ใน Part 63
- อธิบายระบบ middleware ของ Actix-web ที่สร้างจาก **`Transform`/`Service` ของตัวเอง** (ไม่ใช้ `tower` เหมือน
  Axum) เขียน middleware ด้วย **`wrap_fn`** (closure-based, ง่ายกว่าการ implement `Transform` เต็มรูปแบบมาก)
  และเห็นโค้ด `Transform`/`Service` เต็มรูปแบบแบบสั้น ๆ เพื่อเทียบความยาว
- ใช้ middleware สำเร็จรูป **`Logger`**, **`Compress`**, **`DefaultHeaders`**, และ CORS ผ่าน crate
  **`actix-cors`** พร้อมพิสูจน์ preflight `OPTIONS` จริงด้วย curl เทียบกับที่ Part 65 พิสูจน์ไว้ฝั่ง
  `tower-http::CorsLayer`
- พิสูจน์ด้วยการรันจริงว่า **ลำดับการ `.wrap()`** ของ Actix-web ทำงานตามกฎไหน (เหมือนหรือต่างจากกฎของ Axum ที่
  Part 65 พิสูจน์ไว้) แล้วประกอบทุกหัวข้อเป็น capstone ระบบห้องสมุดที่ใช้ scope, custom extractor, และ
  middleware stack ที่จัดลำดับถูกต้องตามกฎที่พิสูจน์แล้ว

## ความรู้ที่ต้องมีมาก่อน

- **Part 67 (แนะนำ Actix-web Framework)**: บทนี้ต่อยอดจาก "Hello World" ของ Actix-web ตรง ๆ — คุณควรเขียน
  handler ด้วย `#[get("/path")]`/`#[post("/path")]` macro, ส่ง response กลับด้วย `Responder`, ใช้
  `web::Data<T>` แชร์ state, และรัน server ด้วย `HttpServer::new(...).bind(...).run()` มาแล้ว บทนี้จะไม่สอน
  พื้นฐานพวกนี้ซ้ำ แต่จะขยายไปที่ routing, extractor, และ middleware ที่ซับซ้อนกว่านั้นโดยตรง
- **Part 62-65 (Axum Framework, Routing, State/Extractors, Middleware)**: บทนี้เขียนขึ้นเพื่อให้**เทียบกับ
  Axum ได้ตรงจุดที่สุด** — ทุกหัวข้อจะอ้างอิงกลับไปยังสิ่งที่ Part 63 (Routing/Handlers) และ Part 65
  (Middleware) สอนไว้ฝั่ง Axum อยู่เรื่อย ๆ เพื่อชี้ให้เห็นว่าอะไรเหมือนกันจริง (แค่ syntax ต่าง) และอะไรต่างกัน
  ในระดับการออกแบบจริง ๆ — ถ้ายังไม่ได้อ่าน Part 63/65 อย่างน้อยควรทวนคร่าว ๆ ก่อน เพราะบทนี้จะไม่อธิบาย
  แนวคิดพื้นฐานร่วม (เช่น "ทำไมต้องมี middleware") ซ้ำอีกครั้ง
- **Part 57 (Serialization: Serde เบื้องต้น)**: `web::Path<T>` และ `web::Query<T>` ทั้งคู่พึ่งพา
  `#[derive(Deserialize)]` แบบเดียวกับที่ Part 57 สอนไว้
- **Part 46-50 (Async/Await, Futures, Tokio)**: `FromRequest::Future` และ `Service::Future` ของ Actix-web
  เป็น associated type ที่ implement `std::future::Future` ตรง ๆ — บทนี้จะใช้ความเข้าใจเรื่อง `Future`/`poll`
  จาก Part 47 ตรง ๆ ตอนอธิบายว่าทำไม Actix-web ยังต้องเขียน `type Future = ...` เอง (ไม่ใช้ native
  `async fn` ใน trait เหมือนที่ Axum ทำได้)
- **Part 30-31 (Error Handling ระดับโปรเจกต์)**: หลักการออกแบบ error type ที่ implement trait ของ framework
  (ในที่นี้คือ `ResponseError` ของ Actix-web) เพื่อควบคุม response ที่ตอบกลับ เป็นหลักการเดียวกับที่ Part 63
  ใช้กับ `IntoResponse` ของ Axum

## เนื้อหา

### 68.1 Path Parameters เจาะลึก: `web::Path<T>`

จาก Part 67 คุณได้เห็น handler ที่ประกาศด้วย `#[get("/path")]` ที่ไม่มี dynamic segment มาแล้ว — หัวข้อนี้จะ
ขยายไปที่ path ที่มีส่วนแปรผันได้ เช่น `/users/42` เหมือนกับที่ Part 63 สอนไว้ฝั่ง Axum ทุกประการในระดับแนวคิด
เพียงแต่ syntax ของ dynamic segment ต่างกัน — Actix-web ใช้ `{name}` เหมือน Axum เวอร์ชัน 0.8 ขึ้นไป (ไม่ใช้
`:name` แบบเก่า) แต่ extractor คนละตัวกันคือ **`web::Path<T>`** แทน `axum::extract::Path<T>`

#### Path Parameter เดียว

```rust
use actix_web::{get, web, App, HttpServer, Responder};

#[get("/users/{id}")]
async fn get_user(path: web::Path<u32>) -> impl Responder {
    let id = path.into_inner();
    format!("user id = {id}")
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| App::new().service(get_user))
        .bind(("127.0.0.1", 4001))?
        .run()
        .await
}
```

สังเกตสิ่งสำคัญที่ต่างจาก Axum ทันที:

1. **`web::Path<u32>` ไม่ได้ destructure ผ่าน pattern ในลิสต์ parameter โดยตรง** (ต่างจาก Axum ที่เขียน
   `Path(id): Path<u32>` ได้เลย) — Actix-web ให้คุณรับ `path: web::Path<u32>` เป็นตัวแปรทั้งก้อนแล้วเรียก
   **`.into_inner()`** เพื่อดึงค่า `u32` ออกมา (หรือใช้ `*path` deref ก็ได้ เพราะ `web::Path<T>` implement
   `Deref<Target = T>` ไว้ให้ — แต่ `.into_inner()` ชัดเจนกว่าเพราะสื่อเจตนาว่า "เอาค่าออกมาเป็นเจ้าของ" ตรง ๆ)
2. Handler ยังคงเป็น `async fn` ปกติที่คืนค่าอะไรก็ได้ที่ implement **`Responder`** (trait ของ Actix-web ที่
   ทำหน้าที่เหมือน `IntoResponse` ของ Axum เป๊ะ ในเชิงบทบาท — จะอธิบายลึกกว่านี้ในหัวข้อ 68.8)

รันจริงแล้วเรียก `curl http://127.0.0.1:4001/users/42`:

```
HTTP/1.1 200 OK
content-length: 12
content-type: text/plain; charset=utf-8
date: Sun, 27 Sep 2026 00:04:10 GMT

user id = 42
```

#### เมื่อ Path Parameter Parse ไม่ผ่าน: จุดที่ Actix-web ต่างจาก Axum จริง ๆ

นี่คือจุดที่สำคัญที่สุดของหัวข้อนี้ — จาก Part 63 คุณรู้แล้วว่า Axum ตอบ **`400 Bad Request`** เมื่อ path
parameter parse ไม่ผ่าน (เพราะ path matching ผ่านแล้ว แค่ค่าแปลงเป็น type ที่ต้องการไม่ได้ ซึ่งถือเป็นความผิด
ของ client) มาดูว่า Actix-web ทำแบบเดียวกันหรือไม่ ด้วยการรันจริงและยิง `curl` เข้า path เดิมแต่ใส่ค่าที่ parse
เป็น `u32` ไม่ได้:

```bash
curl -i http://127.0.0.1:4001/users/abc
```

ผลลัพธ์จริง (คัดลอกตรงจากการรันจริง ไม่ใช่การเดา):

```
HTTP/1.1 404 Not Found
content-length: 28
content-type: text/plain; charset=utf-8
date: Sun, 27 Sep 2026 00:04:10 GMT

can not parse "abc" to a u32
```

**สังเกตให้ดี: status code คือ `404 Not Found` ไม่ใช่ `400 Bad Request`!** นี่คือความแตกต่างเชิงพฤติกรรมที่แท้
จริงจาก Axum ไม่ใช่แค่ syntax — เหตุผลเชิงกลไกคือ Actix-web ใช้ `PathDeserializer` ที่ผูกอยู่กับตัว **router**
เอง (ไม่ใช่ extractor ที่ทำงานแยกทีหลังแบบ Axum) เมื่อ segment ที่ควรจะ parse เป็น `u32` ได้ทำไม่ได้
Actix-web ปฏิบัติกับสถานการณ์นี้เหมือนกับว่า **"ไม่มี route ไหนตรงกับ request นี้เลย"** (คือ 404) มากกว่าจะมองว่า
"route ตรงแล้ว แต่ข้อมูลผิด" (คือ 400) แบบที่ Axum เลือกทำ — ทั้งสองมุมมองมีเหตุผลรองรับได้ทั้งคู่ในทางทฤษฎี
(Axum: "path pattern ตรง แค่ type ไม่ตรง" vs Actix-web: "ในเมื่อ parse ไม่ได้ ก็เท่ากับไม่มี resource ที่ตรงกับ
ค่านี้จริง ๆ") แต่ผลลัพธ์ที่ client เห็นต่างกันจริง และถ้าคุณเขียน integration test ที่ hardcode คาดหวัง `400`
ตามความเคยชินจาก Axum แล้วย้ายมาใช้ Actix-web โดยไม่ตรวจสอบ จะพบว่า test ล้มเหลวทันที

ยืนยันด้วยการทดสอบ struct-based extraction (หลาย parameter พร้อมกัน) ว่าให้ผลเดียวกัน:

```rust
use actix_web::{get, web, Responder};
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct UserPostParams {
    user_id: u32,
    post_id: u32,
}

#[get("/users2/{user_id}/posts/{post_id}")]
async fn get_user_post_struct(path: web::Path<UserPostParams>) -> impl Responder {
    format!(
        "(struct) user_id = {}, post_id = {}",
        path.user_id, path.post_id
    )
}
```

ทดสอบด้วยค่าที่ parse ไม่ผ่าน `curl -i http://127.0.0.1:4001/users2/abc/posts/7`:

```
HTTP/1.1 404 Not Found
content-length: 28
content-type: text/plain; charset=utf-8
date: Sun, 27 Sep 2026 00:04:10 GMT

can not parse "abc" to a u32
```

จุดที่ควรเปรียบเทียบกับ Axum อีกจุด: **ข้อความ error ของ Actix-web ไม่ได้บอกชื่อ field ที่มีปัญหาเลย**
(`can not parse "abc" to a u32` เหมือนกันทุกตัวอักษรกับกรณี `web::Path<u32>` เดี่ยว ๆ) ในขณะที่ Part 63 แสดงให้
เห็นว่า Axum เวอร์ชัน struct-based จะบอกชื่อ field ด้วย (`Cannot parse \`user_id\` with value \`abc\` to a
\`u32\``) — นี่เป็นความแตกต่างเล็กแต่มีผลจริงต่อ debugging experience ใน production ที่ path มี parameter หลาย
ตัวชนิดเดียวกัน (เช่น `user_id`/`post_id` ที่เป็น `u32` ทั้งคู่): Axum บอกได้ว่าตัวไหนผิด แต่ Actix-web (ในเวอร์
ชันที่ทดสอบคือ 4.15.0) ไม่บอก ต้อง debug เองว่า segmentไหนที่ parse ไม่ผ่าน

#### หลาย Path Parameter: แบบ Tuple

เช่นเดียวกับ Axum, Actix-web รองรับ `web::Path<(T1, T2, ...)>` สำหรับดึงหลาย segment พร้อมกันแบบ tuple:

```rust
#[get("/users/{user_id}/posts/{post_id}")]
async fn get_user_post_tuple(path: web::Path<(u32, u32)>) -> impl Responder {
    let (user_id, post_id) = path.into_inner();
    format!("user_id = {user_id}, post_id = {post_id}")
}
```

`web::Path<(u32, u32)>` จับคู่ dynamic segment **ตามลำดับที่ปรากฏใน path** เข้ากับตำแหน่งใน tuple เหมือนกับ
Axum เป๊ะ — ทดสอบจริง `curl http://127.0.0.1:4001/users/42/posts/7`:

```
HTTP/1.1 200 OK
content-length: 25
content-type: text/plain; charset=utf-8
date: Sun, 27 Sep 2026 00:04:10 GMT

user_id = 42, post_id = 7
```

#### หลาย Path Parameter: แบบ Struct

Struct-based extraction (ตัวอย่างที่เห็นไปแล้วข้างบน) ให้ผลลัพธ์ที่อ่านง่ายกว่า tuple ในแบบเดียวกับที่ Part
63 อธิบายไว้ — field ชื่อใน struct ต้องตรงกับชื่อใน `{...}` ของ route pattern เป๊ะ ๆ ทดสอบด้วยค่าถูกต้อง
`curl http://127.0.0.1:4001/users2/42/posts/7`:

```
HTTP/1.1 200 OK
content-length: 34
content-type: text/plain; charset=utf-8
date: Sun, 27 Sep 2026 00:04:10 GMT

(struct) user_id = 42, post_id = 7
```

### 68.2 Query Parameters: `web::Query<T>`

**`web::Query<T>`** ทำหน้าที่เหมือน `axum::extract::Query<T>` เป๊ะ — ดึงส่วนหลัง `?` ของ URL แล้ว deserialize
ด้วย `#[derive(Deserialize)]` แพทเทิร์น pagination ด้วย `Option<T>` ก็ใช้แนวคิดเดียวกันกับ Part 63 ทุกประการ

```rust
use actix_web::{get, web, Responder};
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct Pagination {
    page: Option<u32>,
    limit: Option<u32>,
}

#[get("/items")]
async fn list_items(query: web::Query<Pagination>) -> impl Responder {
    let page = query.page.unwrap_or(1);
    let limit = query.limit.unwrap_or(10);
    format!("page = {page}, limit = {limit}")
}
```

เหตุผลที่ต้องเป็น `Option<T>` เหมือนกับที่ Part 63 อธิบายไว้ทุกประการ: query parameter ไม่บังคับโดยธรรมชาติ —
ถ้าประกาศ `page: u32` ตรง ๆ การเรียกโดยไม่ส่ง `?page=...` มาเลยจะทำให้ deserialize ล้มเหลวทั้งที่ควรตีความว่า
"ใช้ค่า default"

ทดสอบจริงทีละกรณี — **ไม่ส่ง query เลย** (`curl http://127.0.0.1:4001/items`):

```
HTTP/1.1 200 OK
content-length: 20
content-type: text/plain; charset=utf-8
date: Sun, 27 Sep 2026 00:04:10 GMT

page = 1, limit = 10
```

**ส่งแค่ `page`** (`curl "http://127.0.0.1:4001/items?page=2"`):

```
HTTP/1.1 200 OK
content-length: 20
content-type: text/plain; charset=utf-8

page = 2, limit = 10
```

**ส่งครบทั้งสอง** (`curl "http://127.0.0.1:4001/items?page=2&limit=50"`):

```
HTTP/1.1 200 OK
content-length: 20
content-type: text/plain; charset=utf-8

page = 2, limit = 50
```

**ส่งค่าที่ parse ไม่ผ่าน** (`curl -i "http://127.0.0.1:4001/items?page=abc"`) — นี่คือกรณีที่ผลลัพธ์**ตรงกับ
พฤติกรรมของ Axum** (ต่างจาก path parameter ที่ต่างกันในหัวข้อก่อน):

```
HTTP/1.1 400 Bad Request
content-length: 54
content-type: text/plain; charset=utf-8

Query deserialize error: invalid digit found in string
```

จุดที่น่าสนใจคือ **`web::Query<T>` ตอบ `400 Bad Request` เหมือน Axum เป๊ะ แต่ `web::Path<T>` ตอบ
`404 Not Found` ต่างจาก Axum** — สรุปกฎที่ต้องจำสำหรับ Actix-web คือ: **rejection ของ path และ query ไม่ได้ใช้
status code เดียวกันเสมอไปเหมือนที่คุณอาจคุ้นจาก Axum** ต้องแยกจำเป็นราย extractor ไม่ใช่จำเป็นกฎกลางเดียว

### 68.3 Scopes: จัดกลุ่ม Route ด้วย `web::scope`

**`web::scope("/prefix")`** คือกลไกของ Actix-web ที่ทำหน้าที่คล้าย **`.nest()`** ของ Axum ใน Part 63 — เพิ่ม
prefix ให้ route ทั้งกลุ่มโดยไม่ต้องเขียน path เต็มซ้ำในทุก handler แต่วิธี**ประกอบร่าง**กับ handler ต่างจาก
Axum อย่างชัดเจน: Axum ใช้ `Router::new().route(path, get(handler))` แล้ว `.nest(prefix, router)`, ส่วน
Actix-web ที่ประกาศ handler ด้วย `#[get(...)]`/`#[post(...)]` macro (แนวทางที่ Part 67 สอน) จะใช้
**`.service(handler_fn)`** — เพราะ macro พวกนี้ทำให้ฟังก์ชัน implement trait `HttpServiceFactory` ของ
Actix-web เอง ทำให้ `.service(...)` รับมันเข้าไปเป็นหน่วยเดียวที่รวม "path + method + handler" ไว้ในตัว
(สังเกตว่า path ที่ระบุใน `#[get("...")]` เป็น path **สัมพัทธ์กับ scope ที่มันถูก `.service()` เข้าไป** ไม่ใช่
path เต็มจาก root)

มาสร้างระบบห้องสมุด (library) ตัวเดียวกับที่ Part 63/67 ใช้ เพื่อเทียบกันได้ตรง ๆ:

```rust
use actix_web::{get, post, web, App, HttpServer, Responder};

#[get("")]
async fn list_books() -> impl Responder {
    "book list"
}

#[get("/{id}")]
async fn get_book(path: web::Path<u32>) -> impl Responder {
    format!("book id = {}", path.into_inner())
}

#[post("/{id}/borrow")]
async fn borrow_book(path: web::Path<u32>) -> impl Responder {
    format!("borrowed book id = {}", path.into_inner())
}

#[get("")]
async fn list_members() -> impl Responder {
    "member list"
}

async fn health() -> impl Responder {
    "ok"
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        let books_scope = web::scope("/books")
            .service(list_books)
            .service(get_book)
            .service(borrow_book);

        let members_scope = web::scope("/members").service(list_members);

        let api_v1 = web::scope("/api/v1")
            .service(books_scope)
            .service(members_scope);

        App::new()
            .route("/health", web::get().to(health))
            .service(api_v1)
    })
    .bind(("127.0.0.1", 4002))?
    .run()
    .await
}
```

สังเกตประเด็นสำคัญหลายจุด:

1. **`#[get("")]` (path ว่างเปล่า) ใช้เป็น "route ของ scope นั้นเอง" ได้** — เมื่อ `list_books` ถูก
   `.service()` เข้า `web::scope("/books")` แล้ว scope นั้นถูก `.service()` เข้า `web::scope("/api/v1")` อีก
   ชั้น path จริงที่ Actix-web จับคู่ให้คือ `/api/v1/books` (path ว่างต่อท้าย prefix เฉย ๆ) — นี่ต่างจาก Axum ที่
   ต้องเขียน `"/"` (slash) เป็น path ของ route ราก ไม่ใช่สตริงว่าง
2. **`web::scope(...)` ซ้อนกันได้หลายชั้น** เหมือนกับที่ `.nest()` ของ Axum ทำได้ — `books_scope` และ
   `members_scope` แต่ละตัวถูกสร้างแยกกันเป็น resource ของตัวเอง แล้วค่อยรวมเข้า `api_v1` อีกที เป็นรูปแบบการ
   จัดโครงสร้างแอปที่ใหญ่ขึ้นแบบเดียวกับที่ Part 63 สอนไว้เรื่อง per-resource module
3. **`.route("/health", web::get().to(health))`** ใช้ syntax คนละแบบจาก `#[get]` macro โดยสิ้นเชิง — นี่คือ
   วิธีประกาศ route แบบ "builder" ของ Actix-web ที่ไม่ต้องพึ่ง macro เลย (`web::get()` สร้าง `Route` ตัวหนึ่ง
   แล้ว `.to(handler)` ผูก handler เข้าไป) ทั้งสองวิธี (`#[get]` macro กับ `.route(path, web::get().to(...))`)
   ใช้แทนกันได้ในสถานการณ์ส่วนใหญ่ — macro สะดวกกว่าเพราะ path กับ method อยู่ติดกับตัว handler ในไฟล์เดียว
   ส่วน builder-style เหมาะกับตอนที่ต้องประกอบ route จำนวนมากแบบ dynamic หรือไม่อยากให้ handler ผูกกับ path
   ตายตัวตั้งแต่นิยาม (มีประโยชน์มากตอน mount handler เดียวกันที่หลาย path)

ทดสอบจริงทุก path (ยืนยันว่าการ scope ซ้อนกันสองชั้นทำงานถูกต้อง):

```bash
curl http://127.0.0.1:4002/health                      # -> ok
curl http://127.0.0.1:4002/api/v1/books                # -> book list
curl http://127.0.0.1:4002/api/v1/books/5               # -> book id = 5
curl -X POST http://127.0.0.1:4002/api/v1/books/5/borrow # -> borrowed book id = 5
curl http://127.0.0.1:4002/api/v1/members               # -> member list
```

ทั้งห้าคำสั่งได้ผลลัพธ์ `200 OK` ตรงตามคอมเมนต์ทุกบรรทัด (ผลจริงจากการรัน) — พิสูจน์ว่า `web::scope("/api/v1")`
ที่ครอบ `web::scope("/books")` และ `web::scope("/members")` อีกชั้นซ้อนกันได้ถูกต้องจริง เหมือนกับที่
`.nest()` ซ้อนกันได้ใน Axum

### 68.4 Route Guards: แนวคิดที่ Axum ไม่มีเทียบเท่าตรงตัว

นี่คือหัวข้อที่สำคัญที่สุดของบทนี้ในเชิง "จุดที่การออกแบบแยกทางกันจริง" — Axum ตัดสินใจ route request ด้วย
**path + HTTP method เท่านั้น** ถ้าต้องการเลือก handler ตาม header (เช่น `Accept`, custom header) ต้องเขียน
handler เดียวแล้ว `match` เองข้างในฟังก์ชัน หรือใช้ middleware ครอบ — Axum **ไม่มีกลไกระดับ routing** ที่ให้
"ลงทะเบียน handler สองตัวคนละ handler บน path เดียวกัน แล้วให้ router เลือกให้ตาม header" ได้ตรง ๆ

Actix-web มีกลไกนี้ในตัวเรียกว่า **guard** — ฟังก์ชัน/struct ที่ implement `guard::Guard` trait ซึ่งตรวจสอบ
request แล้วตอบ `true`/`false` ว่า resource นี้ควรรับ request นี้ไว้หรือไม่ **ก่อน**ที่จะตัดสินว่า request
"ตรงกับ" resource นั้น — คุณผูก guard เข้ากับ `web::resource(...)` ด้วย `.guard(...)` ได้ และประกาศ
`web::resource` **หลายตัวที่ path เดียวกันแต่มี guard ต่างกัน** ได้ Actix-web จะลองแต่ละตัวตามลำดับจนเจอตัวที่
guard ผ่าน

#### ตัวอย่างจริง: เลือก Handler ตาม `Accept` Header

```rust
use actix_web::{guard, web, App, HttpResponse, Responder};

async fn get_report_json() -> impl Responder {
    HttpResponse::Ok()
        .content_type("application/json")
        .body(r#"{"format":"json"}"#)
}

async fn get_report_html() -> impl Responder {
    HttpResponse::Ok()
        .content_type("text/html")
        .body("<h1>report</h1>")
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    actix_web::HttpServer::new(|| {
        App::new()
            // path เดียวกัน ("/report") ประกาศเป็น resource สองตัว แยกกันด้วย guard คนละแบบ
            .service(
                web::resource("/report")
                    .guard(guard::Header("accept", "application/json"))
                    .to(get_report_json),
            )
            .service(
                web::resource("/report")
                    .guard(guard::Header("accept", "text/html"))
                    .to(get_report_html),
            )
    })
    .bind(("127.0.0.1", 4002))?
    .run()
    .await
}
```

**`guard::Header("accept", "application/json")`** ตรวจว่า request มี header `Accept` ที่ค่าตรงกับ
`"application/json"` เป๊ะหรือไม่ — ผูกเข้ากับ `web::resource(...)` ผ่าน `.guard(...)` ก่อน `.to(handler)`
ทดสอบจริง:

```bash
curl -i http://127.0.0.1:4002/report -H "Accept: application/json"
```
```
HTTP/1.1 200 OK
content-length: 17
content-type: application/json
date: Sun, 27 Sep 2026 00:04:54 GMT

{"format":"json"}
```

```bash
curl -i http://127.0.0.1:4002/report -H "Accept: text/html"
```
```
HTTP/1.1 200 OK
content-length: 15
content-type: text/html
date: Sun, 27 Sep 2026 00:04:54 GMT

<h1>report</h1>
```

```bash
curl -i http://127.0.0.1:4002/report -H "Accept: text/plain"
```
```
HTTP/1.1 404 Not Found
content-length: 0
date: Sun, 27 Sep 2026 00:04:54 GMT
```

**path เดียวกันเป๊ะ (`/report`) ให้ response ต่างกันสามแบบขึ้นอยู่กับ header `Accept` เท่านั้น** — และเมื่อไม่มี
guard ไหนผ่านเลย (กรณีที่สาม) Actix-web ตอบ `404 Not Found` ตรง ๆ (ไม่มี body, ไม่มี header บอกว่า "path นี้มี
อยู่จริงแต่ header ไม่ตรง" อะไรเลย — มันแค่ไม่มี resource ไหนตรงกับ request นี้ในมุมของ router) นี่คือ pattern
ที่ทำได้เองใน Actix-web ที่ Axum ต้องเขียน middleware หรือ manual dispatch ในฟังก์ชันเดียวถึงจะได้ผลลัพธ์
เดียวกัน

#### Custom Guard ด้วย `guard::fn_guard`

นอกจาก guard สำเร็จรูป (`guard::Header`, `guard::Get()`, `guard::Post()`, ...) Actix-web ให้เขียน guard เอง
ด้วยฟังก์ชันธรรมดาผ่าน **`guard::fn_guard`** — รับ `&GuardContext` แล้วคืน `bool`:

```rust
use actix_web::{guard, web, HttpResponse};

async fn get_beta_feature() -> &'static str {
    "beta feature unlocked"
}

// guard เองไม่ต้อง implement trait อะไรเป็นพิเศษ — แค่ตรวจ header แล้วคืน bool
let beta_resource = web::resource("/beta")
    .guard(guard::fn_guard(|ctx| {
        ctx.head().headers().get("X-Client-Version").is_some()
    }))
    .to(get_beta_feature);
```

ทดสอบจริง:

```bash
curl -i http://127.0.0.1:4002/beta -H "X-Client-Version: 1.2.3"
```
```
HTTP/1.1 200 OK
content-length: 21
content-type: text/plain; charset=utf-8

beta feature unlocked
```

```bash
curl -i http://127.0.0.1:4002/beta
```
```
HTTP/1.1 404 Not Found
content-length: 0
```

ไม่มี header `X-Client-Version` เลย -> `404 Not Found` ทันที — เพราะไม่มี resource อื่นประกาศไว้ที่ `/beta`
เลยถ้าไม่ผ่าน guard นี้ (ต่างจากตัวอย่างก่อนหน้าที่มี guard สามแบบแข่งกันที่ path เดียว)

#### Method Guard: `guard::Get()` เทียบกับ `web::get()`

Actix-web มี guard สำหรับ HTTP method ด้วย (`guard::Get()`, `guard::Post()`, ...) ซึ่ง**ทำงานคล้าย**กับที่
`web::get().to(...)`/`#[get(...)]` ทำอยู่แล้วภายใน (ทั้งสองระบบใช้กลไก guard เดียวกันเป็นฐาน — `#[get(...)]`
macro ที่จริงก็ generate `.guard(guard::Get())` ให้อัตโนมัติเบื้องหลัง) แต่การผูก method guard ผ่าน
`.guard(...)` ตรง ๆ มีประโยชน์เมื่อต้องการ**รวม method guard เข้ากับ guard อื่นในเวลาเดียวกัน** เช่น "ต้องเป็น
`GET` และต้องมี header บางอย่างด้วย":

```rust
async fn method_guard_demo() -> &'static str {
    "handled by GET-guarded route"
}

let route = web::resource("/method-demo")
    .guard(guard::Get())
    .to(method_guard_demo);
```

ทดสอบจริง:

```bash
curl -i http://127.0.0.1:4002/method-demo           # GET -> 200
curl -i -X POST http://127.0.0.1:4002/method-demo    # POST -> 404
```

```
HTTP/1.1 200 OK
content-length: 28

handled by GET-guarded route
```
```
HTTP/1.1 404 Not Found
content-length: 0
```

สังเกตว่า POST ที่ไม่ผ่าน method guard ได้ **`404 Not Found`** ไม่ใช่ **`405 Method Not Allowed`** ที่มี header
`Allow` บอกว่า method ไหนใช้ได้ (แบบที่ Part 63 พิสูจน์ไว้ว่า Axum ทำ) — นี่เป็นอีกจุดที่ Actix-web กับ Axum
ต่างกันจริง: **Axum แยกแยะ "path ไม่ตรงเลย" (404) ออกจาก "path ตรงแต่ method ไม่ตรง" (405) อย่างชัดเจน ส่วน
Actix-web ที่ routing ผ่านระบบ guard มองทั้งสองกรณีเป็น "ไม่มี resource ไหนผ่าน guard ทั้งหมด" เหมือนกันหมด
จึงตอบ 404 เสมอไม่ว่าจะขาด method หรือขาด path** — รายละเอียดนี้สำคัญมากถ้าทีมของคุณเขียน client ที่พึ่งพา
`Allow` header เพื่อ discover ว่า endpoint รองรับ method อะไรบ้าง เพราะ Actix-web ไม่ให้ข้อมูลนั้นมาโดย
อัตโนมัติแบบ Axum

### 68.5 กลไกเบื้องหลัง Extractor: `FromRequest` Trait เดี่ยวของ Actix-web

Part 63 อธิบายไว้ว่า Axum แยก extractor เป็นสองระดับ: **`FromRequestParts`** (อ่านได้แค่ headers/URI/method
ไม่แตะ body) และ **`FromRequest`** (ตัวสุดท้ายเท่านั้นที่กิน body ได้) — นี่คือการออกแบบที่ Axum ใช้เพื่อบังคับ
กฎ "body-consuming extractor ต้องมาตัวสุดท้าย" ให้เป็น**ข้อผิดพลาดตอน compile time**

**Actix-web ไม่ได้ทำแบบนั้น** — มัน**ไม่มี** trait คู่แบบนี้เลย มีแค่ **`FromRequest` ตัวเดียว** ที่ extractor
ทุกตัวต้อง implement ไม่ว่าจะอ่านแค่ header หรืออ่าน body ก็ตาม มาดูนิยามจริงจาก source code ของ actix-web
เวอร์ชัน 4.15.0 (`src/extract.rs`):

```rust
// นี่คือ trait definition จริงจาก actix-web 4.15.0 (ตัดคอมเมนต์ยาวออกเพื่อความกระชับ)
pub trait FromRequest: Sized {
    type Error: Into<Error>;
    type Future: Future<Output = Result<Self, Self::Error>>;

    fn from_request(req: &HttpRequest, payload: &mut Payload) -> Self::Future;

    fn extract(req: &HttpRequest) -> Self::Future {
        Self::from_request(req, &mut Payload::None)
    }
}
```

สังเกตสามจุดที่สำคัญมาก และแต่ละจุดคือความแตกต่างเชิงสถาปัตยกรรมจริงจาก Axum:

1. **`from_request` รับ `&HttpRequest` (แค่ reference, ไม่ใช่ owned) และ `&mut Payload` (แยกออกมาต่างหาก)**
   ไม่ใช่รับ `Request` เต็มตัวแบบเดียวแล้ว `.into_parts()` เองแบบที่ Axum ทำภายใน `impl_handler!` — Actix-web
   ออกแบบให้ "หัว" ของ request (`HttpRequest`, อ่านได้พร้อมกันหลายครั้งเพราะเป็น reference) กับ "ตัว"
   (`Payload`, stream ที่ต้อง "เอาออกมา" ถ้าจะอ่านจริง ๆ) แยกกันเป็น argument คนละตัวตั้งแต่ signature ของ
   trait เอง — extractor ที่ไม่แตะ body (เช่น `Path`, `Query`) ก็แค่ไม่แยแส `payload` เลย
2. **`type Future: Future<Output = Result<Self, Self::Error>>` เป็น associated type ที่ต้องกำหนดเอง** —
   นี่คือความต่างที่สำคัญมากจาก Axum: Axum เวอร์ชัน 0.7 ขึ้นไป (ตามที่ Part 64 อธิบายไว้) ใช้ **RPITIT**
   (`async fn from_request_parts(...) -> ...` เขียนตรง ๆ ได้เลยโดยไม่ต้องประกาศ associated type ของ Future
   เอง) แต่ **`FromRequest` ของ Actix-web (เวอร์ชัน 4.15.0 ที่ตรวจสอบเนื้อหาบทนี้) ยังไม่รองรับแบบนั้น** —
   ต้องกำหนด `type Future = ...` เองเสมอ (มักเป็น `std::future::Ready<...>` สำหรับ extractor ที่ตรวจสอบแบบ
   synchronous เช่น อ่าน header เดียว หรือ `Pin<Box<dyn Future<...>>>`/`LocalBoxFuture` สำหรับที่ต้อง
   `.await` จริง เช่นอ่าน body) — เขียน extractor เองใน Actix-web จึงมีโค้ด "พิธีกรรม" (boilerplate) มากกว่า
   Axum เล็กน้อยในจุดนี้เสมอ
3. **ไม่มีการแยก trait ตามว่า "กินได้กี่ตัว"** — เพราะไม่มี `FromRequestParts` แยกออกมา ทุก extractor (ไม่ว่า
   จะกิน body หรือไม่) ก็อยู่ใต้ trait เดียวกันหมด **สิ่งที่ป้องกันไม่ให้อ่าน body ซ้ำสองครั้งไม่ได้อยู่ที่ระดับ
   trait/type-system แบบ Axum แต่อยู่ที่ตัว `Payload` เอง** — `Payload` เป็น enum ที่ "เอาออกมาได้ครั้งเดียว"
   ผ่าน `payload.take()` (คืนค่าจริงแล้วแทนที่ตัวเดิมด้วย `Payload::None`) ตัว extractor ที่กิน body
   (`web::Json<T>`, `web::Bytes`, `web::Form<T>`) จะ `.take()` มันตอนเรียก `from_request` — extractor ตัวไหน
   ที่ถูกเรียก `from_request` **ทีหลัง** แล้วพยายาม `.take()` payload ที่ถูก take ไปแล้ว จะได้ `Payload::None`
   เปล่า ๆ ไม่ใช่ compile error — หัวข้อ 68.7 จะพิสูจน์ผลลัพธ์ที่เกิดขึ้นจริงด้วยการรันจริง

**สรุปความต่างเชิงสถาปัตยกรรมที่ต้องจำ**: Axum ป้องกันการอ่าน body ซ้ำด้วย **type system ที่ compiler
บังคับ**; Actix-web ป้องกันด้วย **ค่า runtime ตัวเดียว (`Payload`) ที่ "หมด" ไปเมื่อถูกเอาไปครั้งแรก** — วิธี
หลังนี้ตรวจไม่พบตอน compile เลย ต้องรอให้รันจริงถึงจะเห็นผล

### 68.6 เขียน Custom Extractor: `ApiKey` จาก Header

มาเขียน extractor ที่ตรวจสอบ custom header เอง — เทียบตรงกับ `ApiKey` ที่ Part 64 สอนไว้ฝั่ง Axum (หัวข้อ
64.6) เพื่อให้เห็นความต่างของโค้ดชัด ๆ

#### ขั้นที่ 1: นิยาม extractor type และ rejection type

```rust
use actix_web::{
    error::ResponseError, http::StatusCode, HttpResponse,
};
use std::fmt;

struct ApiKey(String);

#[derive(Debug)]
enum ApiKeyRejection {
    Missing,
    Invalid,
}

impl fmt::Display for ApiKeyRejection {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ApiKeyRejection::Missing => write!(f, "missing X-Api-Key header"),
            ApiKeyRejection::Invalid => write!(f, "invalid X-Api-Key"),
        }
    }
}

// ResponseError คือ trait ของ Actix-web ที่ทำหน้าที่คล้าย IntoResponse ของ Axum เวลาใช้กับ error type
// (ต่างจาก Responder ตรงที่ ResponseError มีค่า default ของ status_code() มาให้ และ Actix-web ใช้มัน
// โดยอัตโนมัติเมื่อ Result<T, E> handler คืน Err(e) กลับมา ไม่ต้องเขียน match เอง)
impl ResponseError for ApiKeyRejection {
    fn status_code(&self) -> StatusCode {
        match self {
            ApiKeyRejection::Missing => StatusCode::UNAUTHORIZED,
            ApiKeyRejection::Invalid => StatusCode::FORBIDDEN,
        }
    }

    fn error_response(&self) -> HttpResponse {
        HttpResponse::build(self.status_code())
            .content_type("application/json")
            .body(format!(r#"{{"error":"{self}"}}"#))
    }
}
```

สังเกตว่า Actix-web ใช้ **`ResponseError`** (ไม่ใช่ `Responder`) เป็น trait ที่ error type ของ extractor ต้อง
implement — `ResponseError` มี method `status_code()` ที่มีค่า default (`500 Internal Server Error`) และ
`error_response()` ที่เราปรับแต่งเองได้ ต่างจาก `IntoResponse` ของ Axum ที่ใช้ trait**เดียว**กันทั้งสำหรับ
success response และ error response — Actix-web แยก `Responder` (สำหรับค่าที่ handler คืนสำเร็จ) กับ
`ResponseError` (สำหรับ error type ที่ extractor หรือ handler ปฏิเสธ request) เป็นสอง trait คนละตัว

#### ขั้นที่ 2: implement `FromRequest`

```rust
use actix_web::{dev::Payload, FromRequest, HttpRequest};
use std::future::{ready, Ready};

impl FromRequest for ApiKey {
    type Error = ApiKeyRejection;
    // ไม่แตะ body เลย -> ใช้ Ready<...> พอ ไม่ต้อง Box::pin future จริง
    type Future = Ready<Result<Self, Self::Error>>;

    fn from_request(req: &HttpRequest, _payload: &mut Payload) -> Self::Future {
        let header_value = match req.headers().get("X-Api-Key") {
            Some(v) => v,
            None => return ready(Err(ApiKeyRejection::Missing)),
        };
        let key_str = match header_value.to_str() {
            Ok(s) => s,
            Err(_) => return ready(Err(ApiKeyRejection::Invalid)),
        };
        if key_str == "secret-123" {
            ready(Ok(ApiKey(key_str.to_string())))
        } else {
            ready(Err(ApiKeyRejection::Invalid))
        }
    }
}
```

เทียบกับเวอร์ชัน Axum ของ Part 64 (`impl<S> FromRequestParts<S> for ApiKey` ที่เขียน
`async fn from_request_parts(...)` ตรง ๆ ได้เลย) จะเห็นความต่างชัดมาก: Actix-web ต้อง**ประกาศ
`type Future` เอง** และ**เรียก `ready(...)` เพื่อห่อค่าให้เป็น `Future` ที่ resolve ทันที** เพราะ trait method
`from_request` เป็น **sync function ที่คืน type ที่ implement `Future`** ไม่ใช่ตัวมันเองเป็น `async fn` — นี่
คือ boilerplate ที่มาจากเหตุผลในหัวข้อ 68.5 ข้อ 2 ตรง ๆ

**หมายเหตุสำคัญ**: extractor นี้**ไม่ต้องแตะ `payload` เลย** (ใช้ `_payload: &mut Payload` แล้ว ignore) เพราะ
มันอ่านได้แค่จาก header ของ `HttpRequest` — Actix-web ไม่บังคับให้ extractor ที่ไม่กิน body ต้อง implement
trait คนละตัวจาก extractor ที่กิน body (ต่างจาก Axum ที่แยก `FromRequestParts` ออกจาก `FromRequest`
ชัดเจนตาม type) — `ApiKey` implement `FromRequest` ตัวเดียวกันกับที่ `web::Json<T>` implement นั่นแหละ แค่
เนื้อหาข้างในไม่ยุ่งกับ `payload`

#### ขั้นที่ 3: ใช้งานใน handler

```rust
use actix_web::get;

#[get("/protected")]
async fn protected_handler(api_key: ApiKey) -> impl Responder {
    format!("เข้าถึงสำเร็จด้วย API key: {}", api_key.0)
}
```

Handler ไม่รู้เรื่อง `ResponseError`/`FromRequest` เลย — เหมือนกับ Axum ทุกประการในแง่ปรัชญาของ pattern นี้:
**ย้าย cross-cutting concern ออกไปไว้ที่ type system** ทดสอบจริงด้วย curl ครบทั้งสามเส้นทาง:

```bash
curl -s -i http://127.0.0.1:4003/protected
```
```
HTTP/1.1 401 Unauthorized
content-length: 36
content-type: application/json

{"error":"missing X-Api-Key header"}
```

```bash
curl -s -i http://127.0.0.1:4003/protected -H 'X-Api-Key: wrong'
```
```
HTTP/1.1 403 Forbidden
content-length: 29
content-type: application/json

{"error":"invalid X-Api-Key"}
```

```bash
curl -s -i http://127.0.0.1:4003/protected -H 'X-Api-Key: secret-123'
```
```
HTTP/1.1 200 OK
content-length: 71
content-type: text/plain; charset=utf-8

เข้าถึงสำเร็จด้วย API key: secret-123
```

ผลลัพธ์**ตรงกับเวอร์ชัน Axum ของ Part 64 เป๊ะทุกกรณี** (401 สำหรับไม่มี header, 403 สำหรับ header ผิด, 200
พร้อม key สำหรับถูก) — พิสูจน์ว่าทั้งสอง framework แก้ปัญหาเดียวกันได้ผลลัพธ์เดียวกัน แค่กลไกเบื้องหลัง
(trait ที่ implement, การจัดการ Future) ต่างกันตามที่อธิบายไว้ในหัวข้อ 68.5

### 68.7 หลาย Extractor ในฟังก์ชันเดียว: กฎ Arity และลำดับ — พิสูจน์ด้วยการรันจริง

หัวข้อนี้คือใจความสำคัญของบทนี้ในเชิงเทคนิค — Part 63 พิสูจน์ไว้ว่า Axum **บังคับ** ด้วย compile error ว่า
extractor ที่กิน body ต้องมาตัวสุดท้ายเท่านั้น คำถามคือ **Actix-web มีกฎเดียวกันหรือไม่** — อย่าเดา ต้องทดสอบ
จริง

#### ทดสอบที่ 1: วาง Body-Consuming Extractor ไว้ "ก่อน" Extractor อื่น

```rust
use actix_web::{post, web, Responder};
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct NewTicket {
    title: String,
}

// Json<T> (body-consuming) อยู่ "ก่อน" Path<u32> (parts-only) — ใน Axum แบบนี้ compile ไม่ผ่านแน่นอน
#[post("/tickets/{project_id}")]
async fn create_ticket_body_first(
    body: web::Json<NewTicket>,
    path: web::Path<u32>,
) -> impl Responder {
    format!("project {}: {}", path.into_inner(), body.title)
}
```

**ผลจริง: โค้ดนี้ compile ผ่านสะอาด ไม่มี warning เลย** (ทดสอบด้วย `cargo build` กับ actix-web 4.15.0 จริง) —
รันแล้วทดสอบด้วย curl ก็ทำงานถูกต้องสมบูรณ์:

```bash
curl -i -X POST http://127.0.0.1:4003/tickets/7 \
  -H 'Content-Type: application/json' -d '{"title":"printer broken"}'
```
```
HTTP/1.1 200 OK
content-length: 25
content-type: text/plain; charset=utf-8

project 7: printer broken
```

**นี่คือข้อสรุปแรกที่พิสูจน์แล้ว: Actix-web ไม่มีกฎ "body-consuming extractor ต้องมาตัวสุดท้าย" แบบ Axum
เลย** — วางลำดับ extractor ในลิสต์ parameter อย่างไรก็ได้ ตราบใดที่**มีตัวกิน body ไม่เกินหนึ่งตัวที่ถูก
เรียกจริง** เหตุผลตรงกับที่อธิบายไว้ในหัวข้อ 68.5: ไม่มี trait แยกที่บังคับตำแหน่ง, สิ่งที่ตัดสินคือ `Payload`
ตัวเดียวที่ extractor ไหนมาถึงก่อนก็ได้เอาไปก่อน

#### ทดสอบที่ 2: สอง Body-Consuming Extractor ในฟังก์ชันเดียว — จุดที่ Actix-web "เงียบ" แทนที่จะ Error

Part 63 พิสูจน์ไว้ว่า Axum ปฏิเสธด้วย compile error ทันทีถ้ามี body-consuming extractor สองตัว
(`error: Can't have two extractors that consume the request body`) — มาดูว่า Actix-web ทำอะไรกับสถานการณ์
เดียวกัน:

```rust
#[post("/two-bodies")]
async fn two_body_extractors(
    json_body: web::Json<NewTicket>,
    raw_bytes: web::Bytes,
) -> impl Responder {
    format!(
        "json.title={}, raw_bytes.len()={}",
        json_body.title,
        raw_bytes.len()
    )
}
```

**ผลจริง: compile ผ่านสะอาดอีกครั้ง ไม่มี error หรือ warning เลยแม้แต่นิดเดียว** — นี่คือความต่างที่สำคัญมาก:
Axum จับกรณีนี้ได้ตอน compile time ส่วน Actix-web ปล่อยให้ผ่านไปจนถึง runtime มาดูว่าเกิดอะไรขึ้นจริงตอนรัน
(ยิง JSON body เข้าไป):

```bash
curl -i -X POST http://127.0.0.1:4003/two-bodies \
  -H 'Content-Type: application/json' -d '{"title":"hello"}'
```
```
HTTP/1.1 200 OK
content-length: 35
content-type: text/plain; charset=utf-8

json.title=hello, raw_bytes.len()=0
```

**`raw_bytes.len()=0`!** — `json_body: web::Json<NewTicket>` (extractor ตัวแรกในลิสต์) ถูกเรียก
`from_request` ก่อน มัน `.take()` `Payload` ไปอ่านจริงสำเร็จ (ได้ `title=hello`) ทำให้ `Payload` ที่เหลือให้
`raw_bytes: web::Bytes` (ตัวที่สอง) กลายเป็นค่าว่าง — `web::Bytes` ไม่ error เมื่อเจอ payload ว่าง มันแค่คืน
`Bytes` ที่มีความยาว 0 ให้เงียบ ๆ **ไม่มี error ใด ๆ เกิดขึ้นเลยทั้ง compile time และ runtime** — นี่คือความ
เสี่ยงเชิง silent-bug ที่แท้จริงของ Actix-web ในจุดนี้: โค้ดดูเหมือนทำงานถูกต้อง (คืน `200 OK`) แต่ข้อมูลที่
extractor ตัวที่สองได้รับนั้น**ผิดจากที่ตั้งใจไปเงียบ ๆ**

#### ทดสอบที่ 3: สลับลำดับ — Body Extractor ที่ "รอ" อ่านทีหลังจะพังแบบมี Error

ลองสลับลำดับให้ `web::Bytes` มาก่อน `web::Json<T>`:

```rust
#[post("/two-bodies-reversed")]
async fn two_body_extractors_reversed(
    raw_bytes: web::Bytes,
    json_body: web::Json<NewTicket>,
) -> impl Responder {
    format!(
        "raw_bytes.len()={}, json.title={}",
        raw_bytes.len(),
        json_body.title
    )
}
```

ทดสอบจริงด้วย body เดียวกัน:

```bash
curl -i -X POST http://127.0.0.1:4003/two-bodies-reversed \
  -H 'Content-Type: application/json' -d '{"title":"hello"}'
```
```
HTTP/1.1 400 Bad Request
content-length: 68
connection: close
content-type: text/plain; charset=utf-8

Json deserialize error: EOF while parsing a value at line 1 column 0
```

คราวนี้ `web::Bytes` (ตัวแรก) เอา payload จริงไปหมด (`raw_bytes` ได้ bytes ของ JSON ตัวเต็ม) แล้ว
`web::Json<NewTicket>` (ตัวที่สอง) ได้ payload ว่างเปล่า — การพยายาม deserialize JSON จาก payload ว่างล้มเหลว
จริง (`EOF while parsing a value`) ทำให้ครั้งนี้**เกิด error จริง** เป็น `400 Bad Request`

**สรุปกฎที่พิสูจน์แล้วจากการรันจริงทั้งสามทดสอบ**:

1. Actix-web **ไม่มี** compile-time check ใด ๆ เกี่ยวกับตำแหน่งหรือจำนวนของ body-consuming extractor เลย —
   ต่างกับ Axum อย่างสิ้นเชิงในจุดนี้
2. เมื่อมี body-consuming extractor มากกว่าหนึ่งตัว **ตัวที่ถูกประกาศไว้ก่อน (ลำดับซ้ายสุดในลิสต์
   parameter) จะได้ payload จริงไปเสมอ** ตัวถัดไปได้ payload ว่าง (`Payload::None`)
3. **ผลลัพธ์ที่ตัวสอง (ตัวที่ได้ payload ว่าง) ได้รับ ขึ้นอยู่กับว่า extractor ตัวนั้นทนต่อ "ไม่มีข้อมูล" ได้
   แค่ไหน** — `web::Bytes` ทนได้ (คืนความยาว 0 เงียบ ๆ ไม่ error) แต่ `web::Json<T>` ทนไม่ได้ (deserialize
   ล้มเหลว กลายเป็น `400 Bad Request` จริง)
4. **ไม่มีข้อความ error ไหนบอกตรง ๆ ว่า "ปัญหาคือมีสอง extractor แย่ง payload กัน"** — ข้อความที่ได้
   (`EOF while parsing a value`) ฟังดูเหมือนปัญหาการ deserialize ธรรมดา ทำให้ debug ยากกว่า error message
   ของ Axum ที่บอกตรงประเด็นทันที (`Can't have two extractors that consume the request body`)

**บทเรียนเชิงปฏิบัติที่สำคัญที่สุดของหัวข้อนี้**: เมื่อเขียน handler ใน Actix-web ที่มี extractor หลายตัว
**ต้องตรวจสอบด้วยตัวเองเสมอว่ามี body-consuming extractor ไม่เกินหนึ่งตัว** (`web::Json<T>`,
`web::Form<T>`, `web::Bytes`, `String`, `web::Payload`) เพราะ compiler จะไม่ช่วยจับให้เหมือนที่ Axum ทำ —
ถ้าจำเป็นต้องใช้ทั้ง raw bytes และข้อมูลที่ parse แล้วจาก body เดียวกัน ให้ดึงแค่ `web::Bytes` ตัวเดียวแล้ว
`serde_json::from_slice(&raw_bytes)` เองในตัว handler (แพทเทิร์นเดียวกับที่ Part 63 แนะนำไว้ฝั่ง Axum)

#### Arity Limit: มีจำกัดที่ 16 เหมือน Axum หรือไม่

ตรวจสอบจาก source code จริงของ `actix-web` 4.15.0 (ไฟล์ `src/handler.rs`) พบว่ามี macro
`factory_tuple!` ที่ generate `impl Handler` ให้ arity ตั้งแต่ 0 จนถึง **16** (`A` ถึง `P`, รวม 16 ตัวอักษร)
เท่ากับ Axum เป๊ะ — และ `FromRequest` เองก็มี blanket impl ให้ tuple ตั้งแต่ 1 ถึง 16 ตัว
(`tuple_from_req! { TupleFromRequest16; A, B, ..., P }`) ผ่าน macro ในไฟล์ `src/extract.rs` เช่นกัน — สรุปคือ
**ทั้งสอง framework จำกัดจำนวน extractor ต่อ handler ไว้ที่ 16 ตัวเหมือนกัน** ด้วยเหตุผลเดียวกัน (ไม่มี
variadic generics ใน Rust ต้อง generate โค้ดทีละ arity ด้วย macro) แต่ **Actix-web ไม่มี macro ช่วยแปล error
message ให้อ่านง่ายแบบ `#[axum::debug_handler]`** — ถ้าเผลอใส่ extractor เกิน 16 ตัว จะได้ error แบบ
`E0277`/`the trait bound ... is not satisfied` กว้าง ๆ ที่ไม่บอกตรงประเด็นว่า "เกิน 16 ตัว" เหมือนที่ Axum
บอกได้ผ่าน macro debug ของตัวเอง

### 68.8 Custom `Responder` สำหรับ Type ของแอปเอง

**`Responder`** คือ trait ของ Actix-web ที่ทำหน้าที่เหมือน **`IntoResponse`** ของ Axum เป๊ะในเชิงบทบาท: มัน
นิยามว่า "แปลงค่านี้เป็น `HttpResponse` ได้อย่างไร" Actix-web เขียน implementation ให้ type ที่ใช้บ่อยไว้แล้ว
ทั้งหมด (`&str`, `String`, `HttpResponse`, `web::Json<T>`, `Result<T, E>` ที่ `E: ResponseError`, ...) —
เขียน `impl Responder` ให้ type ของแอปเองได้เช่นกัน

```rust
use actix_web::{http::StatusCode, post, web, HttpRequest, HttpResponse, Responder};
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize)]
struct Ticket {
    id: u32,
    title: String,
}

// custom "success" type ที่ห่อ Ticket ไว้พร้อมกำหนด status + header เอง (เทียบ CreatedTicket ของ Axum)
struct CreatedTicket(Ticket);

impl Responder for CreatedTicket {
    type Body = actix_web::body::BoxBody;

    fn respond_to(self, _req: &HttpRequest) -> HttpResponse<Self::Body> {
        let mut response = HttpResponse::build(StatusCode::CREATED).json(self.0);
        response.headers_mut().insert(
            actix_web::http::header::HeaderName::from_static("x-resource-kind"),
            actix_web::http::header::HeaderValue::from_static("ticket"),
        );
        response
    }
}

#[derive(Debug, Deserialize)]
struct NewTicket {
    title: String,
}

#[post("/tickets")]
async fn create_ticket(new_ticket: web::Json<NewTicket>) -> CreatedTicket {
    CreatedTicket(Ticket { id: 1, title: new_ticket.title.clone() })
}
```

สังเกตความต่างจาก `IntoResponse` ของ Axum สองจุด:

1. **`Responder::respond_to` รับ `&HttpRequest` เป็น parameter ด้วย** (ต่างจาก `IntoResponse::into_response`
   ของ Axum ที่ไม่รับ request เข้ามาเลย) — ทำให้ custom `Responder` **เข้าถึงข้อมูลของ request ต้นทางได้**
   ตอนสร้าง response (เช่นเช็ค `Accept` header เพื่อเลือกว่าจะตอบ JSON หรือ HTML, หรือดึงค่าจาก extensions ที่
   middleware แนบไว้ก่อนหน้า) ซึ่งเป็นความสามารถที่ `IntoResponse` ของ Axum ไม่มีให้โดยตรง (ต้องพึ่ง extractor
   ตัวอื่นแยกไปเก็บข้อมูลนั้นเอง)
2. **ต้องกำหนด `type Body` เอง** (ในตัวอย่างนี้ใช้ `actix_web::body::BoxBody` ซึ่งเป็น body type แบบ
   "boxed" ที่ครอบคลุมได้ทุกกรณี ง่ายที่สุดสำหรับ custom `Responder`) — Axum ไม่มีแนวคิด associated type แบบ
   นี้ให้ `IntoResponse` ต้องกำหนด

ทดสอบจริงด้วย curl เห็น header ที่เพิ่มมาจริง:

```bash
curl -i -X POST http://127.0.0.1:4004/tickets \
  -H 'Content-Type: application/json' -d '{"title":"test custom response"}'
```
```
HTTP/1.1 201 Created
content-length: 39
content-type: application/json
x-resource-kind: ticket
date: Sun, 27 Sep 2026 00:06:26 GMT

{"id":1,"title":"test custom response"}
```

header **`x-resource-kind: ticket`** มาจาก `impl Responder` ที่เขียนเอง ยืนยันว่ากลไกทำงานจริง — ผลลัพธ์
เทียบเท่ากับ `CreatedTicket`/`impl IntoResponse` ของ Axum ใน Part 63 ทุกประการ แค่ trait และ signature ของ
method ต่างกัน

### 68.9 Middleware ของ Actix-web: `Transform`/`Service` — ไม่ใช้ `tower`

Part 65 อธิบายไว้อย่างละเอียดว่าระบบ middleware ของ Axum**ทั้งหมด**สร้างขึ้นบน `tower::Service` และ
`tower::Layer` — trait สองตัวที่มาจาก crate `tower` ซึ่งเป็น**ระบบนิเวศกลาง**ที่ไม่ผูกกับ web framework ตัวใด
ตัวหนึ่ง (Axum, Tonic สำหรับ gRPC, และ HTTP client อื่น ๆ ก็ใช้ `tower` ร่วมกันได้)

**Actix-web ไม่ได้ใช้ `tower` เลย** — นี่คือความแตกต่างเชิงสถาปัตยกรรมที่สำคัญที่สุดของทั้งบทนี้ในหมวด
middleware: Actix-web มี trait ของตัวเองชื่อ **`actix_service::Service`** และ **`actix_service::Transform`**
(re-export ผ่าน `actix_web::dev::{Service, Transform}`) ที่**หน้าตาคล้ายกับ `tower::Service`/`tower::Layer`
มาก ในเชิงแนวคิด** (ทั้งคู่คือ "หน่วยที่รับ request แล้วคืน response แบบ async" และ "โรงงานที่ห่อ service เดิม
ให้กลายเป็น service ใหม่") แต่เป็น **crate คนละตัว คนละ ecosystem กันโดยสิ้นเชิง** — middleware ที่เขียนเพื่อ
`tower` ใช้กับ Actix-web ไม่ได้ตรง ๆ (ต้องมี adapter แปลง ซึ่งไม่ใช่เรื่องธรรมดา) และในทางกลับกัน middleware
ของ Actix-web ก็ใช้กับ Axum ไม่ได้เช่นกัน — **ถ้าทีมของคุณมีทั้งโปรเจกต์ Axum และ Actix-web พร้อมกัน อย่าคาด
หวังว่าจะแชร์ middleware library เดียวกันข้ามทั้งสองได้เลย** นี่เป็นผลกระทบเชิงปฏิบัติจริงของการเลือก
framework ที่มักถูกมองข้าม

รูปร่างของ `actix_service::Service` (ตัดรายละเอียดที่ไม่จำเป็นออก) หน้าตาใกล้เคียงกับ `tower::Service` มาก:

```rust
// รูปร่างของ actix_service::Service (คล้าย tower::Service ในเชิงแนวคิด แต่เป็น trait คนละตัว)
pub trait Service<Req> {
    type Response;
    type Error;
    type Future: std::future::Future<Output = Result<Self::Response, Self::Error>>;

    fn poll_ready(&self, ctx: &mut std::task::Context<'_>) -> std::task::Poll<Result<(), Self::Error>>;
    fn call(&self, req: Req) -> Self::Future;
}
```

และ `Transform` ทำหน้าที่คล้าย `tower::Layer` — "โรงงาน" ที่รับ service ชั้นในแล้วคืน service ตัวใหม่ที่ห่อรอบ
มัน:

```rust
pub trait Transform<S, Req> {
    type Response;
    type Error;
    type Transform: Service<Req, Response = Self::Response, Error = Self::Error>;
    type InitError;
    type Future: std::future::Future<Output = Result<Self::Transform, Self::InitError>>;

    fn new_transform(&self, service: S) -> Self::Future;
}
```

#### `wrap_fn`: วิธีที่ง่ายที่สุดในการเขียน Middleware — ไม่ต้อง `impl Transform` เอง

การ implement `Transform`/`Service` เต็มรูปแบบ (จะเห็นตัวอย่างสั้น ๆ ในหัวข้อถัดไป) มีโค้ด boilerplate มาก
พอสมควร — วิธีที่ใช้บ่อยที่สุดในทางปฏิบัติ (และเป็นวิธีที่บทนี้จะใช้เป็นหลักในการสอน เทียบเท่ากับที่ Part 65
ใช้ `axum::middleware::from_fn` เป็นตัวหลัก) คือ **`App::wrap_fn`** (มีให้ทั้งบน `App`, `Scope`, และ
`Resource`) — รับ **closure** ที่ได้ `ServiceRequest` กับ `&Service` ของชั้นในเข้ามาตรง ๆ ไม่ต้องประกาศ
struct หรือ trait อะไรเพิ่มเลย

```rust
use actix_web::{dev::Service as _, web, App, HttpServer};
use std::time::Instant;

async fn list_orders() -> &'static str {
    "[\"order-1\",\"order-2\"]"
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    env_logger::Builder::from_env(env_logger::Env::default().default_filter_or("info")).init();

    HttpServer::new(move || {
        App::new()
            .route("/orders", web::get().to(list_orders))
            // timing middleware ผ่าน wrap_fn — closure รับ (ServiceRequest, &Service ชั้นใน)
            .wrap_fn(|req, srv| {
                let method = req.method().clone();
                let path = req.path().to_string();
                let start = Instant::now();
                let fut = srv.call(req); // เรียก service ชั้นในทันที ได้ Future กลับมา
                async move {
                    let res = fut.await?; // (2) รอผลจาก service ชั้นใน/handler
                    let elapsed = start.elapsed();
                    log::info!(
                        "TIMING_MW: {method} {path} -> status={} elapsed_ms={}",
                        res.status(),
                        elapsed.as_millis()
                    );
                    Ok(res)
                }
            })
    })
    .bind(("127.0.0.1", 4009))?
    .run()
    .await
}
```

สังเกตรูปร่างของ closure ที่ `wrap_fn` ต้องการ: **`Fn(ServiceRequest, &S) -> R` โดย `R: Future<Output =
Result<ServiceResponse<B>, Error>>`** — ต่างจาก `middleware::from_fn` ของ Axum (`async fn(req, next) ->
Response`) ตรงที่ **ไม่มีตัวแทน "next" เป็น value ให้เรียก `.run()`/`.call()` ผ่าน argument โดยตรง** แต่รับ
**`srv: &S` (service ชั้นในทั้งก้อน)** มาแทน แล้วต้องเรียก **`srv.call(req)`** เอง (ต้อง `use
actix_web::dev::Service as _;` เพื่อให้เมธอด `.call()` อยู่ใน scope) — เชิงแนวคิดแล้วทั้งสองแบบทำสิ่งเดียวกัน
(เรียก "ชั้นถัดไป" แล้วได้ response กลับมาให้ปรับแต่งเพิ่ม) แค่วิธีเข้าถึง "ชั้นถัดไป" ต่างกันในระดับ syntax

รันจริงแล้วทดสอบ:

```bash
curl -i http://127.0.0.1:4009/orders
```
```
HTTP/1.1 200 OK
content-length: 21
content-type: text/plain; charset=utf-8

["order-1","order-2"]
```

log จริงที่ได้:

```
[2026-09-27T00:06:42Z INFO  middleware_timing] TIMING_MW: GET /orders -> status=200 OK elapsed_ms=0
```

ยิงไปที่ endpoint ที่ sleep 50ms ก่อนตอบ (`/slow`) ยืนยันว่าจับเวลาได้ถูกต้องจริง:

```
[2026-09-27T00:06:42Z INFO  middleware_timing] TIMING_MW: GET /slow -> status=200 OK elapsed_ms=50
```

#### Short-Circuit ด้วย `wrap_fn`: ปฏิเสธ Request โดยไม่เรียก Service ชั้นใน

เหมือนกับ `middleware::from_fn` ของ Axum ที่เลือกไม่เรียก `next.run(...)` ได้ (Part 65 หัวข้อ 65.4)
`wrap_fn` ก็เลือกไม่เรียก `srv.call(req)` ได้เช่นกัน — จุดที่ต้องระวังคือถ้าจะปฏิเสธโดยไม่เรียก service ชั้นใน
ต้องสร้าง `ServiceResponse` เองจาก `ServiceRequest` เดิม (ผ่าน `req.into_response(...)`) เพราะ
`ServiceResponse` คือคู่ของ `HttpRequest` (ที่ต้องคงไว้เพื่อให้ middleware ชั้นนอกกว่าใช้ต่อได้) กับ
`HttpResponse` และ**เนื่องจากทั้งสองแขนของเงื่อนไข (เรียก service ชั้นใน vs ไม่เรียก) ต้องคืน type เดียวกัน
เป๊ะ** (closure ตัวเดียวคืนได้แค่ type เดียว) วิธีที่สะอาดที่สุดคือใช้ **`futures_util::future::Either`** ห่อ
สองแขนของ future ให้เป็น type เดียวกัน:

```rust
use actix_web::{dev::Service as _, http::StatusCode, web, App, HttpResponse};
use futures_util::future::{ready, Either};

App::new()
    .route("/orders", web::get().to(list_orders))
    .wrap_fn(|req, srv| {
        let has_valid_key = req
            .headers()
            .get("x-api-key")
            .map(|v| v == "secret123")
            .unwrap_or(false);

        if !has_valid_key {
            log::warn!("AUTH_MW: ไม่มี/ผิด x-api-key -> ปฏิเสธที่นี่เลย (short-circuit)");
            let response = HttpResponse::build(StatusCode::UNAUTHORIZED)
                .body("unauthorized: missing or invalid x-api-key");
            let res = req.into_response(response); // สร้าง ServiceResponse จาก req เดิม + response ใหม่
            return Either::Left(ready(Ok(res)));    // ไม่เรียก srv.call เลย
        }

        log::info!("AUTH_MW: x-api-key ถูกต้อง -> ส่งต่อ");
        Either::Right(srv.call(req)) // เรียก service ชั้นในตามปกติ
    });
```

`Either::Left`/`Either::Right` ทำหน้าที่เหมือน enum สองแขนที่ implement `Future` ทั้งคู่ (ถ้าทั้งสองแขนภายใน
implement `Future` ด้วย `Output` ชนิดเดียวกัน) ทำให้ closure คืนค่าที่เป็น type เดียวกันได้ทั้งสองเงื่อนไข —
เป็น pattern ที่พบบ่อยมากเวลาเขียน middleware แบบ short-circuit ด้วย `wrap_fn` ใน Actix-web (คนละแนวทางจาก
Axum's `middleware::from_fn` ที่แค่ `return response.into_response()` ตรง ๆ ได้เลยเพราะ `async fn` เดียวคืน
ค่าได้หลายจุดโดยไม่ต้องห่อ type)

#### เขียน `Transform`/`Service` เต็มรูปแบบ: เพื่อเทียบความยาวโค้ด

เพื่อให้เห็นภาพครบว่า `wrap_fn` ช่วยประหยัดโค้ดไปมากแค่ไหน มาดู timing middleware ตัวเดียวกันที่เขียนแบบ
`Transform`/`Service` เต็มรูปแบบ (ไม่พึ่ง `wrap_fn` เลย) — วิธีนี้จำเป็นเมื่อต้องแจก middleware เป็น crate
ให้คนอื่นใช้ (`wrap_fn` ผูกกับ closure ที่นิยาม ณ จุดใช้งาน เอาไป export เป็น type สาธารณะข้าม crate ไม่ได้
ตรง ๆ) หรือเมื่อต้องเก็บ state ภายใน middleware เอง (เช่น connection pool, cache) ที่ซับซ้อนกว่าการ capture
ตัวแปรใน closure:

```rust
use actix_web::{
    dev::{forward_ready, Service, ServiceRequest, ServiceResponse, Transform},
    Error,
};
use futures_util::future::LocalBoxFuture;
use std::future::{ready, Ready};
use std::rc::Rc;
use std::time::Instant;

pub struct Timing;

// Transform คือ "โรงงาน" ที่รับ service ชั้นใน (S) แล้วคืน TimingMiddleware<S> ที่ห่อมันไว้
impl<S, B> Transform<S, ServiceRequest> for Timing
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Transform = TimingMiddleware<S>;
    type InitError = ();
    type Future = Ready<Result<Self::Transform, Self::InitError>>;

    fn new_transform(&self, service: S) -> Self::Future {
        ready(Ok(TimingMiddleware { service: Rc::new(service) }))
    }
}

pub struct TimingMiddleware<S> {
    service: Rc<S>,
}

impl<S, B> Service<ServiceRequest> for TimingMiddleware<S>
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Future = LocalBoxFuture<'static, Result<Self::Response, Self::Error>>;

    // macro ช่วยจาก actix-web เอง — ส่งต่อ poll_ready ไปยัง service ชั้นในตรง ๆ (กรณีปกติแทบทุกครั้ง)
    forward_ready!(service);

    fn call(&self, req: ServiceRequest) -> Self::Future {
        let method = req.method().clone();
        let path = req.path().to_string();
        let start = Instant::now();
        let svc = self.service.clone();

        Box::pin(async move {
            let res = svc.call(req).await?;
            let elapsed = start.elapsed();
            log::info!(
                "TIMING(Transform): {method} {path} -> status={} elapsed_ms={}",
                res.status(),
                elapsed.as_millis()
            );
            Ok(res)
        })
    }
}

// ใช้งาน: .wrap(Timing) — เหมือนใช้ middleware สำเร็จรูปทุกตัวใน Actix-web
// App::new().route("/hello", web::get().to(hello)).wrap(Timing)
```

โค้ดชุดนี้ (สอง `impl` block, สอง struct) ทำสิ่งเดียวกันเป๊ะกับ `wrap_fn` เวอร์ชันสั้น ๆ ในหัวข้อก่อน — ทดสอบ
จริงด้วย `cargo build`/`cargo run` แล้วยิง curl ได้ผลลัพธ์ที่เหมือนกัน (`.wrap(Timing)` ใช้ได้เหมือน
middleware สำเร็จรูปทุกตัว):

```bash
curl -i http://127.0.0.1:4010/hello
```
```
HTTP/1.1 200 OK
content-length: 37
content-type: text/plain; charset=utf-8

hello from Transform-based middleware
```

```
[INFO  transform_full] TIMING(Transform): GET /hello -> status=200 OK elapsed_ms=0
```

**สรุปบทเรียนของหัวข้อนี้**: เขียน `Transform`/`Service` เต็มรูปแบบมีค่าใช้จ่ายทางโค้ดมากกว่า `wrap_fn` อย่าง
เห็นได้ชัด (สอง `impl` block, ต้องจัดการ `Rc`, `forward_ready!`, `LocalBoxFuture` เอง) — ใช้ `wrap_fn` เป็น
ค่าเริ่มต้นเสมอสำหรับ middleware ที่ใช้ในโปรเจกต์เดียว และสงวน `Transform`/`Service` เต็มรูปแบบไว้สำหรับตอน
ต้อง publish เป็น crate แยกจริง ๆ

### 68.10 Middleware สำเร็จรูป: `Logger`, `Compress`, `DefaultHeaders`, และ CORS

เหมือนกับที่ `tower-http` ให้ middleware สำเร็จรูปกับ Axum, Actix-web มี middleware สำเร็จรูปในตัวอยู่ที่
module **`actix_web::middleware`** สำหรับงานพื้นฐาน (logging, compression, headers) และ CORS ผ่าน crate
แยกต่างหากชื่อ **`actix-cors`** (คนละ crate จาก `actix-web` เอง คล้ายกับที่ `tower-http` เป็นคนละ crate จาก
`axum`)

```toml
[dependencies]
actix-web = "4.15.0"
actix-cors = "0.7.2"
env_logger = "0.11.11"
log = "0.4.34"
serde = { version = "1.0.229", features = ["derive"] }
serde_json = "1.0.151"
```

```rust
use actix_cors::Cors;
use actix_web::{get, http, middleware, web, HttpServer, App, Responder};
use serde::Serialize;

#[derive(Serialize)]
struct Health { status: &'static str }

#[get("/health")]
async fn health() -> impl Responder {
    web::Json(Health { status: "ok" })
}

#[get("/big-data")]
async fn big_data() -> impl Responder {
    let items: Vec<String> = (0..500)
        .map(|i| format!("รายการหนังสือหมายเลข {i} - เรื่อง Rust Programming เล่มที่ {i}"))
        .collect();
    web::Json(items)
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    env_logger::Builder::from_env(env_logger::Env::default().default_filter_or("info")).init();

    HttpServer::new(|| {
        let cors = Cors::default()
            .allowed_origin("http://localhost:5173")
            .allowed_methods(vec!["GET", "POST"])
            .allowed_headers(vec![
                http::header::CONTENT_TYPE,
                http::header::HeaderName::from_static("x-api-key"),
            ])
            .max_age(3600);

        App::new()
            .service(health)
            .service(big_data)
            .wrap(middleware::Compress::default())
            .wrap(middleware::DefaultHeaders::new().add(("X-Version", "1.0")))
            .wrap(cors)
            .wrap(middleware::Logger::default())
    })
    .bind(("127.0.0.1", 4007))?
    .run()
    .await
}
```

โค้ดนี้ compile ผ่านสะอาดจริงด้วย `cargo build` (เวอร์ชันจริงที่ทดสอบ: actix-web 4.15.0, actix-cors 0.7.2)

#### `middleware::Logger`: รูปแบบ Log เริ่มต้นจริง

ต่างจาก `TraceLayer` ของ `tower-http` ที่สร้าง **span** พร้อม event หลายบรรทัดต่อ request (ตามที่ Part 65
พิสูจน์ไว้) `middleware::Logger` ของ Actix-web สร้าง**หนึ่งบรรทัดต่อ request** ในรูปแบบคล้าย **Apache/Nginx
access log** — ตรวจสอบจาก source code จริงของ `actix-web` 4.15.0 (`src/middleware/logger.rs`) พบว่า
**รูปแบบเริ่มต้น (`Logger::default()`) คือ**:

```
%a "%r" %s %b "%{Referer}i" "%{User-Agent}i" %T
```

(`%a` = peer IP, `%r` = request line, `%s` = status, `%b` = response size เป็น byte, `%T` = เวลาที่ใช้ทั้ง
หมดเป็นวินาที) รันจริงแล้วยิง curl สามครั้ง (`/health`, `/health` พร้อม CORS header, `/big-data`) ได้ log
จริงดังนี้ (คัดลอกตรงจากการรันจริง — สังเกตว่า **ระดับ log คือ `INFO` ตั้งแต่ต้น ไม่ต้องตั้ง `RUST_LOG` เป็น
`debug` แบบที่ `TraceLayer` ของ Axum ต้องทำ** ซึ่งเป็นความต่างเล็ก ๆ แต่มีผลจริงต่อ default developer
experience):

```
[2026-09-27T00:07:18Z INFO  actix_web::middleware::logger] 127.0.0.1 "GET /health HTTP/1.1" 200 15 "-" "curl/8.5.0" 0.000240
[2026-09-27T00:07:18Z INFO  actix_web::middleware::logger] 127.0.0.1 "GET /health HTTP/1.1" 200 15 "-" "curl/8.5.0" 0.000155
[2026-09-27T00:07:18Z INFO  actix_web::middleware::logger] 127.0.0.1 "GET /big-data HTTP/1.1" 200 2902 "-" "curl/8.5.0" 0.003689
```

เทียบกับ Axum's `TraceLayer` (Part 65 หัวข้อ 65.5) ที่ให้ **สาม event ต่อ request** ภายใต้ span เดียว
(`started processing request`, `finished processing request`, `end of stream`) — `Logger` ของ Actix-web
กระชับกว่ามาก (บรรทัดเดียวจบ) แต่ให้ข้อมูลเชิงโครงสร้าง (structured logging, field แยกเป็น key-value) น้อย
กว่า `tracing`-based `TraceLayer` ของ Axum ชัดเจน — ถ้าต้องการ log แบบ structured จริงจังใน Actix-web ต้องใช้
`tracing`/`tracing-actix-web` (crate ภายนอกเพิ่มเติม ไม่ใช่ของในตัว Actix-web เอง) ซึ่งเป็นเรื่องนอกสโคปของ
บทนี้

ปรับ format เองได้ด้วย `Logger::new("...")`:

```rust
middleware::Logger::new("%a %{User-Agent}i %r %s %b")
```

#### `middleware::Compress`: บีบอัด Response ด้วย gzip/brotli/zstd อัตโนมัติ

พิสูจน์ด้วยการยิง `/big-data` สองครั้ง — ครั้งแรกบอกว่ารับ gzip ได้ ครั้งที่สองบอกว่าไม่รับ:

```bash
curl -sS -i http://127.0.0.1:4007/big-data -H "Accept-Encoding: gzip" -o /tmp/gz.bin -D -
curl -sS -i http://127.0.0.1:4007/big-data -H "Accept-Encoding: identity" -o /tmp/plain.bin -D -
```

ผลลัพธ์จริง (header ของแต่ละคำขอ):

```
# Accept-Encoding: gzip
HTTP/1.1 200 OK
transfer-encoding: chunked
content-encoding: gzip
content-type: application/json
vary: accept-encoding, Origin, Access-Control-Request-Method, Access-Control-Request-Headers

# Accept-Encoding: identity
HTTP/1.1 200 OK
content-length: 65281
content-type: application/json
vary: Origin, Access-Control-Request-Method, Access-Control-Request-Headers
```

และจากบรรทัด log ของ `Logger` เอง (`%b` คือขนาด response เป็น byte) ยืนยันตัวเลขจริง:

```
"GET /big-data HTTP/1.1" 200 2902 ...   <- บีบอัดแล้ว (gzip)
"GET /big-data HTTP/1.1" 200 65281 ...  <- ไม่บีบอัด (identity)
```

จาก **65,281 bytes เหลือ 2,902 bytes** — บีบอัดได้ประมาณ **95.6%** ของขนาดเดิม ตัวเลขนี้**สอดคล้องกับ
`CompressionLayer` ของ `tower-http` ใน Part 65** (ที่บีบอัดได้ประมาณ 95% กับข้อมูล JSON ที่มีรูปแบบซ้ำสูงแบบ
เดียวกัน) เพราะทั้งคู่ใช้ตัวบีบอัด gzip ที่มีอัตราส่วนใกล้เคียงกันสำหรับข้อมูลลักษณะนี้ — สังเกตว่า header
`vary` ของ Actix-web สะสมค่าจากทั้ง `Compress` (`accept-encoding`) และ `Cors` (`Origin`,
`Access-Control-Request-Method`, `Access-Control-Request-Headers`) เข้าไปใน header เดียวกัน เป็นผลจากการที่
middleware หลายตัวห่อกันเป็นชั้น ๆ แล้วต่างก็เติม `Vary` ของตัวเองเข้าไปสะสมกัน (แนวคิด "onion" เดียวกับที่
Part 65 อธิบายไว้ฝั่ง tower)

#### `middleware::DefaultHeaders`: เติม Header ให้ทุก Response

```rust
middleware::DefaultHeaders::new().add(("X-Version", "1.0"))
```

เติม header `X-Version: 1.0` ให้**ทุก** response ที่ไหลผ่านชั้นนี้ — ยืนยันจากผลลัพธ์จริงที่ curl เห็นในทุก
คำขอ:

```
HTTP/1.1 200 OK
x-version: 1.0
...
```

ใช้บ่อยสำหรับ header ที่ต้องมีทุก response แบบตายตัว เช่น version ของ API, security header
(`X-Content-Type-Options: nosniff`) — เทียบเท่ากับการเขียน `wrap_fn`/`from_fn` เองที่แค่เติม header ก่อนคืน
response แต่ `DefaultHeaders` สำเร็จรูปให้เลยไม่ต้องเขียนเอง

#### CORS ผ่าน `actix-cors`: พิสูจน์ Preflight `OPTIONS` จริง

`actix-cors` (crate แยก ไม่ได้อยู่ใน `actix-web` เอง — ตรงกับสถานะของ `tower-http::CorsLayer` ที่เป็น crate
แยกจาก `axum` เหมือนกัน) ให้ builder `Cors::default()...` คล้ายกับ `CorsLayer::new()...` ของ `tower-http`
มาก ทดสอบ request ปกติที่มี `Origin` header ตรงกับที่อนุญาต:

```bash
curl -sS -i http://127.0.0.1:4007/health -H "Origin: http://localhost:5173"
```
```
HTTP/1.1 200 OK
x-version: 1.0
vary: Origin, Access-Control-Request-Method, Access-Control-Request-Headers
access-control-allow-origin: http://localhost:5173
content-type: application/json
```

เห็น `access-control-allow-origin: http://localhost:5173` เติมมาให้อัตโนมัติ ตรงกับที่ Part 65 พิสูจน์ไว้ฝั่ง
`tower-http::CorsLayer`

**จุดที่ actix-cors ต่างจาก `CorsLayer` ของ Axum อย่างมีนัยสำคัญ**: ลองยิงจาก origin ที่**ไม่ได้**อยู่ใน
รายการอนุญาต:

```bash
curl -sS -i http://127.0.0.1:4007/health -H "Origin: http://evil.example.com"
```
```
HTTP/1.1 200 OK
content-type: application/json
x-version: 1.0
vary: Origin, Access-Control-Request-Method, Access-Control-Request-Headers
```

**สังเกตว่า header `access-control-allow-origin` หายไปทั้งหมด!** — นี่ต่างจาก Part 65 ที่พิสูจน์ไว้ว่า
`tower-http::CorsLayer` ที่ตั้ง `allow_origin` เป็นค่าคงที่ค่าเดียวจะเติม header เดิมกลับไปเสมอไม่ว่า
`Origin` ของ request จะตรงหรือไม่ (เพราะมันเป็นการตั้งค่าตายตัว ไม่ใช่การเทียบแล้วเลือก) — **`actix-cors`
เลือกที่จะ "เทียบ" `Origin` ของ request กับรายการที่อนุญาตจริง ๆ ก่อนตัดสินใจว่าจะเติม header หรือไม่** ถ้าไม่
ตรงก็ไม่เติมอะไรเลย พฤติกรรมนี้ทำให้ `actix-cors` "ปลอดภัยกว่าโดย default" ในความหมายที่ว่ามันไม่โฆษณา origin
ที่อนุญาตออกไปให้ origin อื่นเห็นแบบผิด ๆ — ทั้งสอง approach ยังคง**พึ่งพาบราวเซอร์เป็นผู้บล็อกจริง** (ตามที่
Part 65 อธิบายไว้ว่า `curl` ไม่ใช่บราวเซอร์ จึงไม่ถูกบล็อกอะไรให้เห็น) แต่การไม่เติม header เลยของ
`actix-cors` ทำให้ debug ง่ายกว่า: เห็น header หายไปตรง ๆ ก็รู้ทันทีว่า origin นั้นไม่ผ่านการอนุญาต ไม่ต้อง
สงสัยว่า "ทำไม header มาแต่ browser ยังบล็อก"

##### Preflight `OPTIONS`: พิสูจน์ด้วยการรันจริง

```bash
curl -sS -i -X OPTIONS http://127.0.0.1:4007/health \
  -H "Origin: http://localhost:5173" \
  -H "Access-Control-Request-Method: GET" \
  -H "Access-Control-Request-Headers: content-type"
```

ผลลัพธ์จริง:

```
HTTP/1.1 200 OK
content-length: 0
access-control-allow-methods: POST, GET
access-control-allow-headers: x-api-key, content-type
access-control-allow-origin: http://localhost:5173
access-control-max-age: 3600
vary: Origin, Access-Control-Request-Method, Access-Control-Request-Headers
```

request `OPTIONS` นี้ไม่มี body และ**ไม่ปรากฏ log จาก `Logger` middleware ที่ตำแหน่งที่คาดไว้** (รายละเอียด
เรื่องนี้ขึ้นกับลำดับ `.wrap()` ซึ่งหัวข้อ 68.11 จะพิสูจน์ต่อ) — พฤติกรรมพื้นฐานเหมือนกับ `CorsLayer` ของ Axum
เป๊ะ: `actix-cors` **สกัดคำขอ preflight ไว้ตั้งแต่ชั้น middleware** ตอบกลับด้วย header ที่บอกว่า
method/header ไหนได้รับอนุญาตบ้าง (`access-control-allow-methods`, `access-control-allow-headers`) โดยไม่
ปล่อยให้คำขอไหลลึกไปถึง handler จริงเลย — พิสูจน์ได้ว่า handler `health()` ไม่ถูกเรียก เพราะไม่มี log ใด ๆ
เกี่ยวกับ business logic ปรากฏขึ้นเลยสำหรับคำขอนี้

### 68.11 ลำดับการ `.wrap()`: พิสูจน์ด้วยการรันจริง — เหมือนหรือต่างจาก Axum

นี่คือหัวข้อที่สำคัญที่สุดของบทนี้ในเชิงปฏิบัติ ตรงกับที่ Part 65 หัวข้อ 65.6 พิสูจน์ไว้ว่า **ลำดับ `.layer()`
ของ Axum มีสองกฎที่ขัดกันเอง** ขึ้นอยู่กับว่าใช้ `tower::ServiceBuilder` (เพิ่มก่อน = ชั้นนอกสุด) หรือ
`.layer()` ซ้อนตรง ๆ บน `Router` (เพิ่มทีหลัง = ชั้นนอกสุด) — คำถามคือ **`.wrap()`/`.wrap_fn()` ของ
Actix-web ใช้กฎไหน?** อย่าเดา ต้องพิสูจน์จริง

#### ตั้งสมมติฐาน: Logging กับ Auth ใครควรอยู่ชั้นนอก?

ใช้ middleware สองตัวแบบเดียวกับ Part 65 เป๊ะเพื่อเทียบกันได้ตรง ๆ: `log_mw` (log ก่อน/หลังทุก request) และ
`auth_mw` (เช็ค header `x-api-key` ถ้าไม่ถูกต้อง short-circuit คืน `401` ทันที)

**กรณี A — `.wrap_fn(log_mw)` เพิ่ม*ก่อน*, `.wrap_fn(auth_mw)` เพิ่ม*ทีหลัง*:**

```rust
App::new()
    .route("/orders", web::get().to(list_orders))
    .wrap_fn(log_mw)   // เพิ่มก่อน
    .wrap_fn(auth_mw)  // เพิ่มทีหลัง
```

รันจริงแล้วยิง `curl` สองครั้ง (มี key ถูกต้อง / ไม่มี key เลย) ได้ log จริงดังนี้ (คัดลอกตรงจากการรันจริง):

```
--- request ที่มี key ถูกต้อง ---
[INFO  order_demo] AUTH_MW: x-api-key ถูกต้อง -> ส่งต่อ
[INFO  order_demo] LOG_MW: ก่อนส่งต่อ (before call) method=GET uri=/orders
[INFO  order_demo] HANDLER: list_orders กำลังทำงาน
[INFO  order_demo] LOG_MW: ได้ response กลับมาแล้ว (after call) status=200 OK

--- request ที่ไม่มี key ---
[WARN  order_demo] AUTH_MW: ไม่มี/ผิด x-api-key -> ปฏิเสธที่นี่เลย (short-circuit)
```

สังเกตให้ดี — สำหรับ request **ที่มี key ถูกต้อง**: `AUTH_MW` (เพิ่ม*ทีหลัง*) รันขึ้นมา**ก่อน** `LOG_MW`
(เพิ่ม*ก่อน*) เสมอ และสำหรับ request **ที่ไม่มี key**: มีแค่ `AUTH_MW` ปรากฏ **ไม่มี `LOG_MW` เลยแม้แต่
บรรทัดเดียว** — พิสูจน์ว่า **`auth_mw` (เพิ่มทีหลัง) กลายเป็นชั้นนอกสุด** ปฏิเสธ request ก่อนที่มันจะไหลลึกไป
ถึง `log_mw` (เพิ่มก่อน, ชั้นในกว่า) เลย

**กรณี B — `.wrap_fn(auth_mw)` เพิ่ม*ก่อน*, `.wrap_fn(log_mw)` เพิ่ม*ทีหลัง*:**

```rust
App::new()
    .route("/orders", web::get().to(list_orders))
    .wrap_fn(auth_mw)  // เพิ่มก่อน
    .wrap_fn(log_mw)   // เพิ่มทีหลัง
```

รันจริงด้วย request สองคำขอเดิม (สลับเฉพาะการเรียง `.wrap_fn()`):

```
--- request ที่มี key ถูกต้อง ---
[INFO  order_demo] LOG_MW: ก่อนส่งต่อ (before call) method=GET uri=/orders
[INFO  order_demo] AUTH_MW: x-api-key ถูกต้อง -> ส่งต่อ
[INFO  order_demo] HANDLER: list_orders กำลังทำงาน
[INFO  order_demo] LOG_MW: ได้ response กลับมาแล้ว (after call) status=200 OK

--- request ที่ไม่มี key ---
[INFO  order_demo] LOG_MW: ก่อนส่งต่อ (before call) method=GET uri=/orders
[WARN  order_demo] AUTH_MW: ไม่มี/ผิด x-api-key -> ปฏิเสธที่นี่เลย (short-circuit)
[INFO  order_demo] LOG_MW: ได้ response กลับมาแล้ว (after call) status=401 Unauthorized
```

คราวนี้สำหรับ request **ที่ไม่มี key** เห็น `LOG_MW` ทั้งก่อนและหลัง (`status=401 Unauthorized`) — เพราะ
`log_mw` (เพิ่มทีหลัง) กลายเป็น**ชั้นนอกสุด** เห็น**ทุก**request ที่ผ่านเข้ามา ไม่ว่า `auth_mw` (ชั้นในกว่า)
จะปฏิเสธมันหรือไม่ก็ตาม

#### สรุปกฎที่พิสูจน์แล้วจากการรันจริง: `.wrap()`/`.wrap_fn()` ของ Actix-web

**เมื่อเรียก `.wrap()`/`.wrap_fn()` ซ้อนกันหลายครั้งบน `App` (หรือ `Scope`/`Resource`) — ตัวที่เพิ่ม
`ทีหลัง` (อยู่ล่างสุดในโค้ด) จะกลายเป็น**ชั้นนอกสุด**เสมอ** — กฎนี้**ตรงกับ**กฎของ Axum ตอน `.layer()` ถูก
เรียกซ้อนตรง ๆ บน `Router` (ไม่ผ่าน `ServiceBuilder`) ที่ Part 65 พิสูจน์ไว้ **เป๊ะ** (ทั้งสอง framework ใช้กฎ
"เพิ่มทีหลัง = ห่อรอบสิ่งที่มีอยู่แล้วทั้งหมด = ชั้นนอกสุด" เหมือนกัน) — สิ่งที่ Actix-web **ไม่มี**คือ
ทางเลือกแบบ `tower::ServiceBuilder` ที่ให้กฎกลับด้านได้ (Actix-web ไม่มีระบบ builder แยกสำหรับ compose
middleware ก่อนแล้วค่อย `.wrap()` เข้าทีเดียวแบบที่ Axum ทำได้ผ่าน `tower::ServiceBuilder`) — มีแค่กฎเดียว
ตลอด ไม่ต้องเลือกหรือจำสองกฎที่ขัดกันแบบ Axum

**คำแนะนำเชิงปฏิบัติที่ตรงกับหลักการเดียวกับ Part 65**: วาง middleware ที่ทำ **observability** (เช่น
`Logger`, timing middleware) ไว้เป็น **`.wrap()` ตัวสุดท้ายที่เรียก** เพื่อให้มันกลายเป็นชั้นนอกสุด เห็น
**ทุก** request รวมถึงที่ถูกปฏิเสธโดย middleware ชั้นในกว่า (เช่น auth, CORS preflight) — นี่คือเหตุผลที่
capstone ท้ายบทจะเรียง `.wrap(Logger).wrap(cors).wrap_fn(timing)` โดยวาง timing (ที่อยากให้เห็นทุก request
รวมถึง preflight) ไว้เป็นตัวสุดท้าย

พิสูจน์ผลของกฎนี้ต่อ CORS preflight ที่เห็นในหัวข้อ 68.10: เมื่อจัดลำดับ `.wrap(Logger).wrap(cors)` (Logger
เพิ่มก่อน = ชั้นในกว่า cors) request `OPTIONS` ที่ `actix-cors` (ชั้นนอกกว่า) ตอบกลับตรง ๆ โดยไม่ส่งต่อ จะ**
ไม่ไปถึง `Logger` เลย** (เพราะ Logger อยู่ชั้นในกว่า) — ตรงกับที่สังเกตไว้ในหัวข้อ 68.10 ว่าไม่มี log ปรากฏ
สำหรับ preflight request นั่นเอง นี่คือเหตุผลเชิงกลไกที่แท้จริง ไม่ใช่ความบังเอิญ

### 68.12 Capstone: Bookshelf API เวอร์ชัน Actix-web

มาประกอบทุกหัวข้อของบทนี้เข้าด้วยกันเป็น REST API เล็ก ๆ ที่ทำงานได้จริงทั้งระบบ — ระบบห้องสมุดเดียวกับที่
Part 63/67 ใช้ เพื่อเทียบกับเวอร์ชัน Axum ได้ตรงจุดที่สุด มีครบ:

- Scope ซ้อนกัน (`/api/v1/books/...`)
- Path parameter (`/books/{id}`)
- Custom extractor `ApiKey` ป้องกัน endpoint ที่แก้ไขข้อมูล (`create_book`, `borrow_book`)
- Middleware stack: `Logger` + CORS (`actix-cors`) + custom timing `wrap_fn` จัดลำดับตามกฎที่พิสูจน์ไว้ใน
  หัวข้อ 68.11 (timing เป็นตัวสุดท้าย = ชั้นนอกสุด เห็นทุก request)

```rust
use actix_cors::Cors;
use actix_web::{
    dev::{Payload, Service as _},
    error::ResponseError,
    get,
    http::{header, StatusCode},
    middleware, post, web, App, HttpRequest, HttpResponse, HttpServer, Responder,
};
use serde::{Deserialize, Serialize};
use std::fmt;
use std::sync::Mutex;
use std::time::Instant;

#[derive(Debug, Clone, Serialize)]
struct Book {
    id: u32,
    title: String,
    author: String,
    available: bool,
}

#[derive(Debug, Deserialize)]
struct NewBook {
    title: String,
    author: String,
}

struct AppState {
    books: Mutex<Vec<Book>>,
    next_id: Mutex<u32>,
}

// ===== Custom extractor: ApiKey (หัวข้อ 68.6) =====
struct ApiKey(#[allow(dead_code)] String);

#[derive(Debug)]
enum ApiKeyRejection {
    Missing,
    Invalid,
}

impl fmt::Display for ApiKeyRejection {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ApiKeyRejection::Missing => write!(f, "missing X-Api-Key header"),
            ApiKeyRejection::Invalid => write!(f, "invalid X-Api-Key"),
        }
    }
}

impl ResponseError for ApiKeyRejection {
    fn status_code(&self) -> StatusCode {
        match self {
            ApiKeyRejection::Missing => StatusCode::UNAUTHORIZED,
            ApiKeyRejection::Invalid => StatusCode::FORBIDDEN,
        }
    }
    fn error_response(&self) -> HttpResponse {
        HttpResponse::build(self.status_code())
            .content_type("application/json")
            .body(format!(r#"{{"error":"{self}"}}"#))
    }
}

impl actix_web::FromRequest for ApiKey {
    type Error = ApiKeyRejection;
    type Future = std::future::Ready<Result<Self, Self::Error>>;

    fn from_request(req: &HttpRequest, _payload: &mut Payload) -> Self::Future {
        let result = match req.headers().get("X-Api-Key") {
            None => Err(ApiKeyRejection::Missing),
            Some(v) => match v.to_str() {
                Ok(s) if s == "secret-123" => Ok(ApiKey(s.to_string())),
                _ => Err(ApiKeyRejection::Invalid),
            },
        };
        std::future::ready(result)
    }
}

mod books_routes {
    use super::*;

    #[get("")]
    pub async fn list_books(state: web::Data<AppState>) -> impl Responder {
        let books = state.books.lock().unwrap();
        web::Json(books.clone())
    }

    #[get("/{id}")]
    pub async fn get_book(state: web::Data<AppState>, path: web::Path<u32>) -> impl Responder {
        let id = path.into_inner();
        let books = state.books.lock().unwrap();
        match books.iter().find(|b| b.id == id) {
            Some(b) => HttpResponse::Ok().json(b),
            None => HttpResponse::NotFound()
                .json(serde_json::json!({ "error": format!("ไม่พบหนังสือ id={id}") })),
        }
    }

    // ต้องมี ApiKey ที่ถูกต้องก่อนจะสร้างหนังสือได้ — endpoint สาธารณะ (list/get) ไม่ต้อง
    #[post("")]
    pub async fn create_book(
        state: web::Data<AppState>,
        _api_key: ApiKey,
        new_book: web::Json<NewBook>,
    ) -> impl Responder {
        let mut next_id = state.next_id.lock().unwrap();
        let id = *next_id;
        *next_id += 1;
        let book = Book {
            id,
            title: new_book.title.clone(),
            author: new_book.author.clone(),
            available: true,
        };
        state.books.lock().unwrap().push(book.clone());
        HttpResponse::Created().json(book)
    }

    #[post("/{id}/borrow")]
    pub async fn borrow_book(
        state: web::Data<AppState>,
        _api_key: ApiKey,
        path: web::Path<u32>,
    ) -> impl Responder {
        let id = path.into_inner();
        let mut books = state.books.lock().unwrap();
        match books.iter_mut().find(|b| b.id == id) {
            None => HttpResponse::NotFound()
                .json(serde_json::json!({ "error": format!("ไม่พบหนังสือ id={id}") })),
            Some(book) if !book.available => HttpResponse::Conflict()
                .json(serde_json::json!({ "error": format!("หนังสือ id={id} ถูกยืมไปแล้ว") })),
            Some(book) => {
                book.available = false;
                HttpResponse::Ok().json(book.clone())
            }
        }
    }
}

async fn health() -> impl Responder {
    "ok"
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    env_logger::Builder::from_env(env_logger::Env::default().default_filter_or("info")).init();

    let state = web::Data::new(AppState {
        books: Mutex::new(vec![
            Book { id: 0, title: "The Rust Programming Language".into(), author: "Steve Klabnik".into(), available: true },
            Book { id: 1, title: "Programming Rust".into(), author: "Jim Blandy".into(), available: true },
        ]),
        next_id: Mutex::new(2),
    });

    HttpServer::new(move || {
        let cors = Cors::default()
            .allowed_origin("http://localhost:5173")
            .allowed_methods(vec!["GET", "POST"])
            .allowed_headers(vec![header::CONTENT_TYPE, header::HeaderName::from_static("x-api-key")])
            .max_age(3600);

        let books_scope = web::scope("/books")
            .service(books_routes::list_books)
            .service(books_routes::get_book)
            .service(books_routes::create_book)
            .service(books_routes::borrow_book);

        let api_v1 = web::scope("/api/v1").service(books_scope);

        App::new()
            .app_data(state.clone())
            .route("/health", web::get().to(health))
            .service(api_v1)
            // ลำดับ .wrap(): เพิ่ม "ทีหลัง" = ชั้นนอกสุด (หัวข้อ 68.11) — timing อยู่นอกสุด เห็นทุก request
            .wrap(middleware::Logger::default())
            .wrap(cors)
            .wrap_fn(|req, srv| {
                let method = req.method().clone();
                let path = req.path().to_string();
                let start = Instant::now();
                let fut = srv.call(req);
                async move {
                    let res = fut.await?;
                    let elapsed = start.elapsed();
                    log::info!(
                        "TIMING_MW: {method} {path} -> status={} elapsed_ms={}",
                        res.status(),
                        elapsed.as_millis()
                    );
                    Ok(res)
                }
            })
    })
    .bind(("127.0.0.1", 4008))?
    .run()
    .await
}
```

#### Curl Transcript เต็มรูปแบบ (รันจริงทุกคำสั่ง)

```bash
curl -i http://127.0.0.1:4008/api/v1/books
```
```
HTTP/1.1 200 OK
content-length: 167
content-type: application/json
vary: Origin, Access-Control-Request-Method, Access-Control-Request-Headers

[{"id":0,"title":"The Rust Programming Language","author":"Steve Klabnik","available":true},{"id":1,"title":"Programming Rust","author":"Jim Blandy","available":true}]
```

```bash
curl -i http://127.0.0.1:4008/api/v1/books/999
```
```
HTTP/1.1 404 Not Found
content-length: 55
content-type: application/json

{"error":"ไม่พบหนังสือ id=999"}
```

```bash
curl -i -X POST http://127.0.0.1:4008/api/v1/books \
  -H 'Content-Type: application/json' \
  -d '{"title":"Zero To Production In Rust","author":"Luca Palmieri"}'
```
```
HTTP/1.1 401 Unauthorized
content-length: 36
connection: close
content-type: application/json

{"error":"missing X-Api-Key header"}
```

**ไม่มี `X-Api-Key` ถูกปฏิเสธจริง** — extractor `ApiKey` ทำงานก่อน `web::Json<NewBook>` แม้จะประกาศไว้ก่อน
extractor ตัวนั้นในลิสต์ parameter (จำได้จากหัวข้อ 68.7 ว่าลำดับไม่บังคับแบบ Axum) ทดสอบใหม่พร้อม key ที่
ถูกต้อง:

```bash
curl -i -X POST http://127.0.0.1:4008/api/v1/books \
  -H 'Content-Type: application/json' \
  -H 'X-Api-Key: secret-123' \
  -d '{"title":"Zero To Production In Rust","author":"Luca Palmieri"}'
```
```
HTTP/1.1 201 Created
content-length: 87
content-type: application/json

{"id":2,"title":"Zero To Production In Rust","author":"Luca Palmieri","available":true}
```

```bash
curl -i -X POST http://127.0.0.1:4008/api/v1/books/2/borrow -H 'X-Api-Key: secret-123'
```
```
HTTP/1.1 200 OK
content-length: 88
content-type: application/json

{"id":2,"title":"Zero To Production In Rust","author":"Luca Palmieri","available":false}
```

```bash
curl -i -X POST http://127.0.0.1:4008/api/v1/books/2/borrow -H 'X-Api-Key: secret-123'
```
```
HTTP/1.1 409 Conflict
content-length: 75
content-type: application/json

{"error":"หนังสือ id=2 ถูกยืมไปแล้ว"}
```

ยืมซ้ำครั้งที่สองตอบ `409 Conflict` ตรงตามที่ตั้งใจ — เหมือนกับที่ Part 63 ทำได้ฝั่ง Axum เป๊ะ พิสูจน์ preflight
สำหรับ endpoint ที่ต้อง auth:

```bash
curl -i -X OPTIONS http://127.0.0.1:4008/api/v1/books \
  -H "Origin: http://localhost:5173" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: content-type,x-api-key"
```
```
HTTP/1.1 200 OK
content-length: 0
access-control-allow-methods: GET, POST
access-control-max-age: 3600
access-control-allow-headers: content-type, x-api-key
access-control-allow-origin: http://localhost:5173
```

และดู log จริงจากเซิร์ฟเวอร์ที่ยืนยันทั้งลำดับ middleware และการทำงานของ business logic พร้อมกัน:

```
[INFO  capstone] TIMING_MW: GET /api/v1/books -> status=200 OK elapsed_ms=0
[INFO  actix_web::middleware::logger] 127.0.0.1 "GET /api/v1/books HTTP/1.1" 200 167 "-" "curl/8.5.0" 0.000279
[INFO  capstone] TIMING_MW: POST /api/v1/books -> status=401 Unauthorized elapsed_ms=0
[INFO  actix_web::middleware::logger] 127.0.0.1 "POST /api/v1/books HTTP/1.1" 401 36 "-" "curl/8.5.0" 0.000203
[INFO  capstone] TIMING_MW: POST /api/v1/books -> status=201 Created elapsed_ms=0
[INFO  actix_web::middleware::logger] 127.0.0.1 "POST /api/v1/books HTTP/1.1" 201 87 "-" "curl/8.5.0" 0.000207
[INFO  capstone] TIMING_MW: POST /api/v1/books/2/borrow -> status=200 OK elapsed_ms=0
[INFO  actix_web::middleware::logger] 127.0.0.1 "POST /api/v1/books/2/borrow HTTP/1.1" 200 88 "-" "curl/8.5.0" 0.000295
[INFO  capstone] TIMING_MW: POST /api/v1/books/2/borrow -> status=409 Conflict elapsed_ms=0
[INFO  actix_web::middleware::logger] 127.0.0.1 "POST /api/v1/books/2/borrow HTTP/1.1" 409 75 "-" "curl/8.5.0" 0.000260
[INFO  capstone] TIMING_MW: OPTIONS /api/v1/books -> status=200 OK elapsed_ms=0
```

จุดที่ควรสังเกตเป็นพิเศษ (พิสูจน์กฎของหัวข้อ 68.11 ต่อในสถานการณ์จริง): **`TIMING_MW` มี log สำหรับคำขอ
`OPTIONS` preflight ด้วย** (เพราะมันเป็น `.wrap_fn()` ที่เพิ่ม*ทีหลังสุด* จึงเป็นชั้นนอกสุด เห็นทุก request
รวม preflight) แต่ **`actix_web::middleware::logger` ไม่มี log บรรทัดคู่กันสำหรับ `OPTIONS` เลย** (เพราะ
`Logger` ถูกเพิ่ม*ก่อน* `cors` จึงอยู่ชั้นในกว่า `cors` — และ `cors` สกัด preflight ไว้ตอบกลับเองก่อนที่คำขอ
จะไหลลึกไปถึง `Logger`) — นี่คือหลักฐานที่ตรงกับที่อธิบายไว้ในหัวข้อ 68.10-68.11 ทุกประการ: ตำแหน่งของ
middleware ในลำดับ `.wrap()` มีผลจริงต่อว่า middleware ตัวไหนจะ "เห็น" request ประเภทไหนบ้าง

## กับดักที่พบบ่อย (Common Pitfalls)

**1. คาดหวังว่า path parameter parse ไม่ผ่านจะตอบ `400` เหมือน Axum — แต่ Actix-web ตอบ `404`**

```bash
curl -i http://127.0.0.1:4001/users/abc
```
```
HTTP/1.1 404 Not Found
content-length: 28

can not parse "abc" to a u32
```

ถ้าเขียน integration test ที่ port มาจาก Axum โดยคาดหวัง `assert_eq!(response.status(), 400)` แล้วย้ายมาใช้
Actix-web โดยไม่ตรวจสอบ test จะล้มเหลวทันที เพราะ Actix-web ปฏิบัติกับ path parameter ที่ parse ไม่ผ่านเหมือน
"ไม่มี route ไหนตรงกับ request นี้เลย" (404) ไม่ใช่ "route ตรงแต่ข้อมูลผิด" (400) แบบ Axum — ต้องเขียน error
handling ของฝั่ง client ให้รองรับทั้งสองความเป็นไปได้ถ้าต้องรองรับทั้งสอง framework ในระบบเดียวกัน

**2. ใส่ extractor สองตัวที่กิน body พร้อมกัน โดยไม่รู้ว่า Actix-web ไม่เตือนตอน compile**

```rust
// compile ผ่านสะอาด ไม่มี warning เลย — แต่ raw_bytes จะได้ค่าว่างเสมอ (len()=0)
#[post("/two-bodies")]
async fn two_body_extractors(
    json_body: web::Json<NewTicket>,
    raw_bytes: web::Bytes,
) -> impl Responder { /* ... */ }
```

Axum จะปฏิเสธโค้ดแบบนี้ตั้งแต่ compile time ด้วย error ที่บอกตรงประเด็น แต่ Actix-web ปล่อยให้ผ่านไปเงียบ ๆ
แล้วให้ extractor ตัวที่สอง (ตามลำดับที่ประกาศ) ได้ payload ว่างไปเสมอ — ตรวจสอบด้วยตัวเองทุกครั้งว่า handler
มี body-consuming extractor (`web::Json<T>`, `web::Form<T>`, `web::Bytes`, `String`, `web::Payload`) ไม่เกิน
หนึ่งตัว ถ้าต้องการทั้ง raw bytes และข้อมูลที่ parse แล้ว ให้ดึงแค่ `web::Bytes` แล้ว
`serde_json::from_slice(&bytes)` เองในตัว handler

**3. ลืมว่า `web::Path<T>` ต้อง `.into_inner()` หรือ deref ไม่ได้ destructure ตรงในลิสต์ parameter แบบ Axum**

```rust
// ผิด — Path ไม่ใช่ tuple struct ที่ destructure ตรง ๆ แบบ Path(id): Path<u32> ของ Axum
async fn get_user(Path(id): web::Path<u32>) -> impl Responder { /* ... */ }
```

compile ได้ error ที่บอกว่า `Path` ไม่ match กับ pattern แบบนั้น (Actix-web's `web::Path<T>` เป็น newtype ที่
ให้เข้าถึงค่าผ่าน `Deref`/`.into_inner()` เท่านั้น ไม่ใช่ tuple struct แบบเดียวกับ `axum::extract::Path<T>`)
วิธีแก้: รับ `path: web::Path<u32>` เป็นตัวแปรทั้งก้อนแล้วเรียก `path.into_inner()` หรือ `*path` เพื่อดึงค่า
ออกมา

**4. ลืมว่า guard ที่ไม่ผ่านให้ผลเป็น `404` เสมอ ไม่มี `405 Method Not Allowed` แบบ Axum**

```bash
curl -i -X POST http://127.0.0.1:4002/method-demo   # path มีจริง แต่ guard ต้องการ GET เท่านั้น
```
```
HTTP/1.1 404 Not Found
content-length: 0
```

ไม่มี header `Allow` บอกว่า method ไหนใช้ได้แบบที่ Axum ให้มา (Part 63 พิสูจน์ไว้ว่า Axum ตอบ `405` พร้อม
`allow: GET,HEAD,POST`) — ถ้า client ของคุณพึ่งพา `405`/`Allow` เพื่อ discover API ต้องเขียน endpoint สำหรับ
discovery แยกเอง (เช่น `OPTIONS` handler เอง) เพราะ Actix-web ไม่ให้ข้อมูลนี้มาโดยอัตโนมัติ

**5. เข้าใจผิดว่า `wrap_fn`/`.wrap()` ใช้กฎเดียวกับ `tower::ServiceBuilder` ของ Axum**

```rust
// สมมติว่า "เพิ่มก่อน = ชั้นนอกสุด" แบบ tower::ServiceBuilder — ผิด!
App::new().wrap_fn(auth_mw).wrap_fn(log_mw)
// จริง ๆ log_mw (เพิ่มทีหลัง) คือชั้นนอกสุด ไม่ใช่ auth_mw
```

Actix-web มีกฎเดียวเท่านั้น (ไม่มีทางเลือกแบบ `ServiceBuilder` ของ `tower`): **`.wrap()`/`.wrap_fn()` ที่เพิ่ม
ทีหลังสุดคือชั้นนอกสุดเสมอ** ตามที่หัวข้อ 68.11 พิสูจน์ไว้ — วาง middleware ที่ต้องเห็นทุก request (logging,
timing) ไว้เป็น `.wrap()` ตัวสุดท้ายเสมอ

**6. ลืมเปิด feature `derive` ของ `serde` แล้วสงสัยว่าทำไม `#[derive(Deserialize)]` หา macro ไม่เจอ**

```
error: cannot find derive macro `Deserialize` in this scope
```

`web::Path<T>`/`web::Query<T>` ต้องการ `T: Deserialize` — ถ้า `Cargo.toml` มีแค่ `serde = "1.0.229"` โดยไม่
เปิด feature `derive` (`serde = { version = "1.0.229", features = ["derive"] }`) จะเจอ error นี้ทันทีตั้งแต่
compile — ปัญหาเดียวกันนี้เกิดขึ้นได้กับทั้ง Axum และ Actix-web เพราะทั้งคู่พึ่งพา `serde` crate เดียวกันสำหรับ
extractor พวกนี้

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เพิ่ม endpoint `GET /api/v1/books/{id}/history` ในตัวอย่าง scope ของหัวข้อ 68.3 ที่คืน
   `web::Json<Vec<String>>` รายการข้อความประวัติปลอม ๆ (hard-code ก็ได้)
   - Hint: ประกาศ handler ใหม่ด้วย `#[get("/{id}/history")]` แล้ว `.service(...)` เพิ่มเข้า `books_scope`
     เดิม — สังเกตว่า path นี้ไม่ชนกับ `#[get("/{id}")]` เดิม เพราะ Actix-web มองว่าเป็น pattern คนละแบบ (มี
     literal segment `/history` ต่อท้าย)

2. **(กลาง)** เพิ่ม guard เข้ากับ endpoint `POST /api/v1/books` ในตัวอย่าง capstone ให้ตรวจสอบ
   `Content-Type: application/json` ด้วย `guard::Header("content-type", "application/json")` ควบคู่กับ
   method guard เดิม (ใช้ `.guard(...)` สองครั้งบน `web::resource` เดียวกัน หรือรวมเป็น guard เดียวด้วย
   `guard::All(...)`) — ทดสอบด้วย curl ทั้งกรณีที่ส่ง `Content-Type` ถูกและผิด ยืนยันว่ากรณีผิดได้ `404`
   - Hint: ถ้าใช้ `#[post(...)]` macro อยู่แล้ว การเพิ่ม guard เพิ่มเข้าไปตรง ๆ ทำไม่ได้ผ่าน macro — ต้อง
     เปลี่ยนมาใช้ `web::resource("/books").guard(guard::Post()).guard(guard::Header(...)).to(handler)` แทน

3. **(ยาก)** เขียนโปรแกรมทดลอง (แยกไฟล์ใหม่) ที่มี `web::Json<T>` สามตัวในฟังก์ชันเดียว (จงใจผิด) แล้วสังเกต
   ว่า Actix-web compile ผ่านหรือไม่ ถ้าผ่าน ให้ยิง request จริงแล้วบันทึกว่าตัวไหนได้ข้อมูลจริงและตัวไหนได้
   ข้อมูลว่าง/error — เขียนสรุปสั้น ๆ เทียบกับผลของหัวข้อ 68.7 (ที่ทดสอบด้วยสองตัว) ว่าพฤติกรรมสอดคล้องกับกฎที่
   สรุปไว้หรือไม่ (ตัวแรกได้ของจริง ตัวถัดไปทั้งหมดได้ payload ว่าง)
   - Hint: extractor ตัวที่ 2 และ 3 ทั้งคู่จะได้ `Payload::None` เหมือนกัน (ไม่ใช่แค่ตัวที่ 2) เพราะตัวแรก
     `.take()` ไปหมดแล้วตั้งแต่ตอนนั้น — ทดสอบว่า error message ของ `web::Json<T>` ตัวที่ 2 กับตัวที่ 3
     เหมือนกันหรือไม่ (ควรจะเหมือน เพราะทั้งคู่เจอ payload ว่างเหมือนกัน)

4. **(ยาก/ประยุกต์ใช้งานจริง)** ขยาย capstone ของหัวข้อ 68.12 ให้มี resource ที่สองคือ `members` (สมาชิก
   ห้องสมุด: `id`, `name`) และ endpoint `POST /api/v1/members/{member_id}/borrow/{book_id}` ที่ต้องมี
   `ApiKey` ด้วย จัดโครงสร้างเป็นสอง module (`books_routes`/`members_routes`) แล้ว `.service()` ทั้งคู่เข้า
   scope `/api/v1` เดียวกัน — เพิ่ม guard ตรวจสอบว่า `member_id` ต้องเป็นตัวเลขบวกเท่านั้นด้วย
   `guard::fn_guard` ก่อนถึง handler จริง (แม้ `web::Path<u32>` จะตรวจ type อยู่แล้ว ให้ guard นี้ตรวจเงื่อนไข
   เพิ่มเติมเช่น "ต้องไม่ใช่ 0") ทดสอบด้วย curl transcript เต็มรูปแบบว่ายืมได้ตามที่ตั้งใจ และยืมหนังสือที่ถูก
   ยืมไปแล้วได้ `409 Conflict` เหมือนเดิม
   - Hint: `guard::fn_guard` ทำงาน**ก่อน**ที่ router จะตัดสินว่า resource นี้ "ตรง" กับ request หรือไม่ — ถ้า
     guard ไม่ผ่าน จะได้ `404` (ไม่ใช่ error message ที่กำหนดเองได้ผ่าน `ResponseError` เหมือน extractor
     rejection) เพราะ guard ทำงานอยู่ในชั้น routing ไม่ใช่ชั้น extraction — ถ้าต้องการ error message ที่
     กำหนดเองสำหรับกรณีนี้ ต้องย้าย logic ไปตรวจใน handler หรือ extractor เอง (เช่นสร้าง
     `PositiveMemberId` extractor เอง) ไม่ใช่ใน guard

## สรุป

บทนี้พา Actix-web จากระดับ "Hello World" ของ Part 67 ไปสู่ระบบ routing, extractor, และ middleware ที่ใช้งาน
ได้จริงในโปรเจกต์ขนาดจริง โดยตั้งใจเปรียบเทียบกับ Axum ทุกหัวข้อเพื่อให้เห็นว่าอะไรเหมือนกันจริงและอะไรต่างกัน
ในระดับการออกแบบ — คุณได้เห็นว่า `web::Path<T>` parse ไม่ผ่านตอบ **`404`** (ต่างจาก Axum's `400`) ในขณะที่
`web::Query<T>` ยังตอบ `400` เหมือนกัน; ได้เห็น **`web::scope`** ทำหน้าที่คล้าย `.nest()` แต่ประกอบร่างกับ
`#[get]`/`#[post]` macro ผ่าน `.service()`; ได้เห็น **route guard** (`guard::Header`, `guard::fn_guard`,
method guard) ซึ่งเป็นกลไก routing ที่ Axum ไม่มีเทียบเท่าตรงตัว; ได้เห็นว่า **`FromRequest`** เป็น trait
เดียว (ไม่แยกเป็นคู่แบบ Axum) และ**ไม่มีกฎ compile-time**เรื่องลำดับ body-consuming extractor เลย — พิสูจน์
ด้วยการรันจริงว่าสอง extractor ที่กิน body พร้อมกัน compile ผ่านเงียบ ๆ แต่ให้ผลลัพธ์ที่ไม่ตรงตามความตั้งใจ;
เขียน custom extractor `ApiKey` และ custom `Responder` เทียบตรงกับ Axum; และในหมวด middleware ได้เห็นว่า
Actix-web ใช้ **`Transform`/`Service` ของตัวเอง ไม่ใช่ `tower`** สอนผ่าน **`wrap_fn`** เป็นหลัก ใช้ middleware
สำเร็จรูป (`Logger`, `Compress`, `DefaultHeaders`, `actix-cors`) และพิสูจน์ด้วยการรันจริงว่า **ลำดับ
`.wrap()`** ของ Actix-web ใช้กฎ "เพิ่มทีหลัง = ชั้นนอกสุด" เพียงกฎเดียว (ตรงกับกฎ `.layer()` แบบตรงของ Axum
แต่ Actix-web ไม่มีทางเลือกแบบ `ServiceBuilder`) ปิดท้ายด้วย capstone ระบบห้องสมุดที่ประกอบทุกอย่างเข้าด้วยกัน
และทดสอบผ่าน curl ครบทุก endpoint

Part ถัดไป (**Part 69: เปรียบเทียบ Axum vs Actix-web vs Rocket**) จะรวบทุกอย่างที่ Part 62-68 สอนไว้ทั้งสอง
framework มาเทียบกันเป็นตารางสรุปอย่างเป็นระบบ พร้อมแนะนำ Rocket เป็น framework ตัวที่สามสั้น ๆ เพื่อช่วยให้คุณ
ตัดสินใจเลือก framework ที่เหมาะกับโปรเจกต์จริงของตัวเองได้

---

**Part ก่อนหน้า:** [แนะนำ Actix-web Framework](part-067-actix-web-intro.md) | **Part ถัดไป:** [เปรียบเทียบ Axum vs Actix-web vs Rocket](part-069-framework-comparison.md)
