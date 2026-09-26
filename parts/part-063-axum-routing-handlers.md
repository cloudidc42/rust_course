# Part 63: Axum: Routing และ Handlers

> โมดูล: ระดับสูง (Advanced) | ระดับ: สูง | เวลาโดยประมาณ: 240 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- ดึงค่าจาก path ของ URL ด้วย **`Path<T>`** extractor ได้ทั้งแบบ parameter เดียว, แบบ tuple (`Path<(u32, u32)>`)
  และแบบ struct ที่ `#[derive(Deserialize)]` (เชื่อมกับ Part 57) พร้อมอธิบายได้ว่าเมื่อ path parameter parse
  ไม่ผ่าน (เช่นส่ง `abc` เข้าไปในที่ที่ควรเป็น `u32`) Axum จะตอบกลับด้วย HTTP status และ body อะไร**จริง ๆ**
  (ไม่ใช่แค่เดา) เพราะได้เห็น response จริงจากการรันโปรแกรมจริงมาแล้ว
- ดึง query string (ส่วนหลัง `?` ของ URL) ด้วย **`Query<T>`** extractor พร้อมออกแบบ struct ที่มี field เป็น
  `Option<T>` เพื่อรองรับ pagination (`page`, `limit`) ที่ผู้ใช้ไม่ส่งมาก็ได้ และรู้ว่าเกิด error อะไรตอน query
  string parse ไม่ผ่าน
- ประกาศ route ที่รองรับหลาย HTTP method บน path เดียวกัน (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`) ด้วย
  **`MethodRouter`** ผ่าน combinator แบบ `get(...).post(...)` และประกอบ route ทั้งหมดของ resource หนึ่งตัว
  (เช่น "tickets") ให้เป็นชุดที่อ่านเข้าใจง่าย เหมือนกับที่ REST API จริงต้องมี
- แบ่งแอปที่กำลังใหญ่ขึ้นออกเป็นกลุ่ม route ต่อ resource ด้วย **`.nest(...)`** และรวมหลาย `Router` เข้าด้วยกัน
  ด้วย **`.merge(...)`** พร้อมจัดโครงสร้างโค้ดเป็น module ต่อ resource ตามแนวคิดที่ Part 16 สอนไว้
- อธิบายกฎการจับคู่ path (path matching) ของ Axum ได้อย่างแม่นยำ — exact match ชนะ dynamic parameter, dynamic
  parameter ชนะ wildcard `{*rest}`, trailing slash **ไม่ถูก redirect ให้อัตโนมัติ** (ต่างจาก framework บาง
  ตัว) และรู้ว่าการประกาศ route ที่ชนกัน (เช่น `/users/{id}` กับ `/users/{name}`) จะทำให้โปรแกรม **panic ตอน
  build router** ไม่ใช่ตอน request เข้ามา — ทั้งหมดนี้พิสูจน์ด้วยการรันจริงและเห็น response/panic message จริง
- อธิบาย **`Handler` trait** ในระดับที่ลึกกว่า "Hello World" ของ Part 62 — ว่าทำไม handler function ถึงรับ
  extractor ได้ตั้งแต่ 0 ถึงสูงสุด 16 ตัว (มาจาก macro `impl_handler!` ที่ generate ทีละ arity), ทำไม
  extractor ตัวสุดท้ายเท่านั้นที่อนุญาตให้ "กิน" request body ได้ (เช่น `Json<T>`) และเกิดอะไรขึ้นจริง (compile
  error จริง) เมื่อละเมิดกฎนี้
- เขียน `impl IntoResponse` ให้ type ของแอปตัวเองเพื่อควบคุม response ที่ส่งกลับได้ละเอียดขึ้น (status code,
  header) และใช้ `#[axum::debug_handler]` แปลง compile error ที่งงงวยให้กลายเป็นข้อความที่อ่านแล้วรู้ทันทีว่า
  ต้องแก้อะไร — เทียบ error message จริงทั้งแบบมีและไม่มี macro นี้
- ประกอบทุกหัวข้อเข้าด้วยกันเป็น REST API เล็ก ๆ ที่ทำงานได้จริงทั้งระบบ (ระบบยืมหนังสือ) ที่มี path parameter,
  query parameter (filter + pagination), nested/merged router, และ custom response ครบ พร้อม curl transcript
  จริงยืนยันทุก endpoint

## ความรู้ที่ต้องมีมาก่อน

- **Part 62 (แนะนำ Axum Framework)**: บทนี้ต่อยอดจาก "Hello World" ของ Axum ที่ Part 62 สอนไว้ตรง ๆ — คุณควร
  เขียน `Router::new().route("/", get(handler))`, ส่ง JSON response กลับด้วย `Json<T>`, และรัน server ด้วย
  `axum::serve` มาแล้ว บทนี้จะไม่สอนพื้นฐานพวกนี้ซ้ำ แต่จะขยายไปที่ routing และ handler ที่ซับซ้อนกว่านั้น
  โดยตรง — ถ้า Part 62 ยังไม่แน่น กลับไปทวนก่อน เพราะบทนี้อ้างอิงกลับไปตลอดโดยไม่อธิบายกลไกพื้นฐานซ้ำ
- **Part 57 (Serialization: Serde เบื้องต้น)**: `Path<T>` และ `Query<T>` ทั้งคู่พึ่งพา `#[derive(Deserialize)]`
  ในแบบเดียวกับที่ Part 57 สอนไว้ทุกประการ — struct ที่คุณ derive `Deserialize` มาแปลง JSON ได้ ก็แปลง path
  segment หรือ query string ได้เหมือนกัน เพราะทั้งคู่คือ "ข้อมูลรูปแบบ key-value ที่ต้อง deserialize" เพียงแต่
  แปลงจาก URL แทน JSON
- **Part 16 (Modules และการจัดระเบียบโค้ด)**: หัวข้อ "แบ่งแอปเป็น per-resource module" ในบทนี้ใช้
  `mod` block/`pub fn` ตรงตามที่ Part 16 สอน — ถ้ายังไม่คุ้นกับ `mod`, `pub`, และการจัดไฟล์เป็นโมดูลย่อย ควร
  ทวน Part 16 ก่อน
- **Part 44 (Procedural Macros เบื้องต้น)** และ **Part 45 (Procedural Macros: Derive Macros ขั้นสูง)**:
  `#[axum::debug_handler]` เป็น **attribute macro** (`#[proc_macro_attribute]`) ตัวเดียวกับที่ Part 44 สอน
  กลไกให้เขียนเอง — บทนี้จะไม่อธิบายกลไก macro ซ้ำ แต่จะใช้ความเข้าใจนั้นอธิบายว่าทำไม macro ตัวนี้ถึงสามารถ
  "อ่าน" ลายเซ็นของฟังก์ชันตอน compile time แล้วสร้าง error message ที่เจาะจงกว่าที่ trait bound ธรรมดาทำได้
- **Part 12 (Result และ Error Handling เบื้องต้น)**: โค้ดในบทนี้ใช้ `Result<T, E>` เป็นค่าที่ handler คืนกลับ
  (เช่น `Result<Json<Book>, ApiError>`) เพื่อแทน "หาไม่พบ" หรือ "ทำไม่ได้" — ยังไม่ได้ลงรายละเอียดการออกแบบ
  error type แบบเต็มรูปแบบ (นั่นคือหน้าที่ของ **Part 66** ที่จะสอน error handling ของ Axum อย่างละเอียด) บทนี้
  ใช้แค่กลไกพื้นฐานที่สุดพอให้ capstone ทำงานได้จริง

## เนื้อหา

### 63.1 Path Parameters: ดึงค่าจากส่วนที่แปรผันของ URL

จาก Part 62 คุณได้เห็น route ที่ตรงกับ path แบบตายตัวเป๊ะ ๆ เช่น `"/"` หรือ `"/hello"` มาแล้ว — แต่ REST API
จริงต้องมี path ที่มีส่วนที่**แปรผันได้** เช่น `/users/42` ที่ `42` คือ user id ที่เปลี่ยนไปได้ทุกครั้งที่มีคน
เรียก endpoint นี้ Axum ใช้ syntax `{name}` ในตอนประกาศ route เพื่อบอกว่า "ส่วนนี้ของ path เป็นตัวแปร ชื่อ
`name`" (สำคัญมาก: axum ตั้งแต่เวอร์ชัน 0.8 เปลี่ยน syntax จาก `:name` แบบเวอร์ชันเก่ามาเป็น `{name}` — ถ้าคุณ
เห็นตัวอย่างเก่าบน internet ที่ใช้ `:id` นั่นคือ syntax ของ axum รุ่นก่อน 0.8 ซึ่งจะ**compile ไม่ผ่าน**บน axum
0.8 ขึ้นไปที่บทนี้ใช้)

#### Path Parameter เดียว

```rust
use axum::{extract::Path, routing::get, Router};

async fn get_user(Path(id): Path<u32>) -> String {
    format!("user id = {id}")
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/users/{id}", get(get_user));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3001").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

สังเกตสิ่งสำคัญสามอย่าง:

1. **`{id}` ในสตริง path** ต้องมีชื่อตรงกับสิ่งที่ extractor คาดหวัง (แต่สำหรับ `Path<u32>` เดี่ยว ๆ แบบนี้ Axum
   ไม่สนใจชื่อ เพราะมีแค่ parameter เดียวให้ดึง — ชื่อจะมีความหมายจริงจังตอนใช้ struct-based extraction ที่จะ
   เห็นถัดไป)
2. **`Path<u32>`** คือ extractor — มันบอก Axum ว่า "อ่านค่าจาก path segment แล้ว parse เป็น `u32`" การ
   `Path(id): Path<u32>` นี้คือ pattern ที่ destructure `Path<u32>` (ซึ่งเป็น tuple struct ที่มี field เดียว)
   ออกมาเป็นตัวแปร `id: u32` โดยตรงในลิสต์ parameter ของฟังก์ชัน — pattern นี้เหมือนกับที่คุณเคย destructure
   tuple struct มาตั้งแต่บทต้น ๆ ของหลักสูตร
3. **ฟังก์ชันคืนค่า `String` ตรง ๆ** — Axum แปลง `String` เป็น response ที่มี `content-type: text/plain`
   อัตโนมัติผ่าน `IntoResponse` (จะอธิบายลึกกว่านี้ในหัวข้อ 63.7)

รันจริงแล้วเรียก `curl http://127.0.0.1:3001/users/42` ได้ผลลัพธ์:

```
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 12

user id = 42
```

#### เมื่อ Path Parameter Parse ไม่ผ่าน: ดู Rejection จริง ไม่ใช่เดา

คำถามที่สำคัญมากคือ: ถ้ามีคนเรียก `/users/abc` (ที่ `abc` ไม่ใช่ตัวเลข) จะเกิดอะไรขึ้น? Axum ไม่ได้ปล่อยให้
โปรแกรม panic หรือคืน error 500 กำกวม ๆ — มันสร้าง **rejection response** ที่มีทั้ง HTTP status และข้อความ
อธิบายปัญหาให้อัตโนมัติ นี่คือผลจริงจากการรัน (ไม่ใช่การเดา):

```bash
curl -i http://127.0.0.1:3001/users/abc
```

```
HTTP/1.1 400 Bad Request
content-type: text/plain; charset=utf-8
content-length: 42

Invalid URL: Cannot parse `abc` to a `u32`
```

สามจุดสำคัญที่ต้องจำ:

- **status code คือ `400 Bad Request`** ไม่ใช่ `404` หรือ `500` — เพราะ path มันตรงกับ route pattern
  `/users/{id}` ถูกต้องแล้ว (route matching ผ่าน) แต่**ค่าที่ดึงมาไม่สามารถแปลงเป็น type ที่ handler ต้องการ
  ได้** ซึ่งตามหลัก HTTP semantics แล้วคือความผิดของ**ผู้ส่ง request** (client ส่งข้อมูลผิดรูปแบบมา) จึงเป็น
  `4xx` ไม่ใช่ `5xx`
- **ข้อความบอกตรง ๆ ว่า parse ค่าอะไรไม่ผ่าน เป็น type ไหน** — `Cannot parse \`abc\` to a \`u32\`` — มีประโยชน์
  มากตอน debug เพราะไม่ต้องเดาว่า field ไหนหรือ type ไหนที่มีปัญหา
- **โค้ด handler ของคุณไม่ถูกเรียกเลย** — ทั้งกระบวนการนี้เกิดขึ้น**ก่อน**ที่ `get_user` จะถูกรันด้วยซ้ำ เพราะ
  `Path<u32>::from_request_parts` (กลไกเบื้องหลัง extractor ที่จะอธิบายในหัวข้อ 63.6) คืน `Err` ออกมาก่อน
  Axum เห็น `Err` แล้วแปลงเป็น response ทันทีโดยไม่เรียก handler function ต่อ — นี่คือเหตุผลที่คุณไม่ต้องเขียน
  `if !id.is_numeric() { return 400 }` เองในทุก handler ที่มี path parameter

#### หลาย Path Parameter: แบบ Tuple

เมื่อ path มีมากกว่าหนึ่ง dynamic segment เช่น `/users/{user_id}/posts/{post_id}` วิธีแรกที่ง่ายที่สุดคือใช้
**tuple**:

```rust
async fn get_user_post_tuple(Path((user_id, post_id)): Path<(u32, u32)>) -> String {
    format!("user_id = {user_id}, post_id = {post_id}")
}
```

ประกาศ route ด้วยชื่อ segment สองตัวตามลำดับที่จะปรากฏใน path:

```rust
Router::new().route("/users/{user_id}/posts/{post_id}", get(get_user_post_tuple));
```

`Path<(u32, u32)>` จับคู่ dynamic segment **ตามลำดับที่ปรากฏใน path** เข้ากับตำแหน่งใน tuple — segment แรก
(`user_id`) ไปที่ตำแหน่งแรกของ tuple, segment ที่สอง (`post_id`) ไปที่ตำแหน่งที่สอง **ไม่สนใจชื่อ**ที่ตั้งไว้ใน
สตริง route เลย มีแต่ตำแหน่ง (position) ที่มีความหมาย — วิธีนี้ใช้ได้ดีตอน parameter น้อยและ type ไม่ปนกัน แต่
พอ path ซับซ้อนขึ้นจะเสี่ยงสลับตำแหน่งผิดโดยไม่มี compiler เตือน (ถ้า type ตรงกันหมด เช่นเป็น `u32` ทั้งคู่)

เรียกจริง `curl http://127.0.0.1:3001/users/42/posts/7`:

```
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 25

user_id = 42, post_id = 7
```

#### หลาย Path Parameter: แบบ Struct (แนะนำสำหรับ Path ที่ซับซ้อน)

วิธีที่ปลอดภัยและอ่านง่ายกว่าคือดึงเป็น **struct ที่ `#[derive(Deserialize)]`** เหมือนที่ Part 57 สอนไว้ทุก
ประการ — field ชื่ออะไรใน struct ต้อง**ตรงกับชื่อใน `{...}` ของ route pattern เป๊ะ ๆ**:

```rust
use serde::Deserialize;

#[derive(Deserialize)]
struct UserPostParams {
    user_id: u32,
    post_id: u32,
}

async fn get_user_post_struct(Path(params): Path<UserPostParams>) -> String {
    format!("(struct) user_id = {}, post_id = {}", params.user_id, params.post_id)
}
```

```rust
Router::new().route("/users2/{user_id}/posts/{post_id}", get(get_user_post_struct));
```

ข้อดีของวิธีนี้ที่ tuple ให้ไม่ได้: **ชื่อ field เป็นตัวจับคู่ ไม่ใช่ตำแหน่ง** ต่อให้สลับลำดับใน path pattern
(`{post_id}/{user_id}` แทน) struct ก็ยังดึงค่าถูกต้องเพราะ Axum จับคู่ด้วยชื่อ ไม่ใช่ตำแหน่งที่ปรากฏ — และเมื่อ
path มี parameter มากกว่า 2-3 ตัว การอ่านโค้ดที่ใช้ named field ย่อมชัดเจนกว่า tuple ที่นับตำแหน่งเอาเอง

ทดสอบด้วยค่าถูกต้อง `curl http://127.0.0.1:3001/users2/42/posts/7`:

```
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 34

(struct) user_id = 42, post_id = 7
```

และทดสอบ rejection กรณี struct-based ด้วยค่าผิด `curl http://127.0.0.1:3001/users2/abc/posts/7` — สังเกตว่า
ข้อความ error **ระบุชื่อ field ที่ parse ไม่ผ่านด้วย** ต่างจากกรณี `Path<u32>` เดี่ยว ๆ ที่ไม่มีชื่อ field ให้
บอก:

```
HTTP/1.1 400 Bad Request
content-type: text/plain; charset=utf-8
content-length: 63

Invalid URL: Cannot parse `user_id` with value `abc` to a `u32`
```

ข้อความนี้ชัดกว่าเวอร์ชัน tuple มาก — ในระบบที่มี path parameter หลายตัวและมีมากกว่าหนึ่งตัวเป็น type เดียวกัน
(เช่นทั้งคู่เป็น `u32`) การรู้ว่า**field ไหน**ที่มีปัญหาช่วยลดเวลา debug ได้มาก นี่คือเหตุผลเชิงปฏิบัติที่ทำให้
struct-based extraction เป็นตัวเลือกที่แนะนำเมื่อ path มี parameter มากกว่า 1 ตัว

#### กับดักที่ต้องรู้ล่วงหน้า: ห้ามใช้ `Path<T>` ซ้ำหลายตัวใน handler เดียว

ความผิดพลาดที่พบบ่อยของคนใหม่คือคิดว่าจะดึง path parameter หลายตัวได้ด้วยการใส่ `Path<T>` **หลายพารามิเตอร์**
ในฟังก์ชันเดียว:

```rust
// ผิด — ห้ามทำแบบนี้
async fn get_user_post_wrong(
    Path(user_id): Path<u32>,
    Path(post_id): Path<u32>,
) -> String {
    format!("user_id = {user_id}, post_id = {post_id}")
}
```

โค้ดนี้ compile ไม่ผ่าน (จะเห็น error message จริงในหัวข้อ "กับดักที่พบบ่อย" ท้ายบท) — คำตอบที่ถูกคือใช้ tuple
หรือ struct แบบที่เพิ่งเห็นไปเท่านั้น เพราะ path มี**ค่าเดียว**ให้ดึง (ทั้ง URL path) การพยายามดึงมันสองครั้ง
แยกกันไม่สมเหตุสมผลในทางออกแบบของ Axum — Axum ผลักดันให้คุณดึง "ทุกอย่างที่ต้องการจาก path" ในครั้งเดียวผ่าน
tuple/struct เพื่อให้ compiler ช่วยเช็คว่าจำนวนและ type ตรงกับ pattern ที่ประกาศไว้

### 63.2 Query Parameters: ดึงค่าจากส่วนหลัง `?` ของ URL

Path parameter เหมาะกับข้อมูลที่**บ่งบอกตัวตนของ resource** (เช่น "หนังสือเล่มไหน") แต่ query parameter เหมาะ
กับข้อมูลที่**ปรับพฤติกรรมของการค้นหา/แสดงผล** (เช่น "หน้าไหน", "จำกัดกี่รายการ", "กรองด้วยเงื่อนไขอะไร") —
URL อย่าง `/items?page=2&limit=20` มี query string คือส่วนหลัง `?` ที่ Axum ดึงออกมาให้ด้วย **`Query<T>`**
extractor

```rust
use axum::{extract::Query, routing::get, Router};
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct Pagination {
    page: Option<u32>,
    limit: Option<u32>,
}

async fn list_items(Query(pagination): Query<Pagination>) -> String {
    let page = pagination.page.unwrap_or(1);
    let limit = pagination.limit.unwrap_or(10);
    format!("page = {page}, limit = {limit}")
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/items", get(list_items));
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3002").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

จุดที่สำคัญที่สุดของแพทเทิร์นนี้คือ **field ทั้งสองเป็น `Option<u32>` ไม่ใช่ `u32` ตรง ๆ** — เหตุผลคือ query
parameter นั้น**ไม่บังคับ**โดยธรรมชาติของมัน (ผู้ใช้ไม่ส่ง `?page=...` มาเลยก็ต้องเป็น request ที่ valid ได้)
ถ้าคุณประกาศเป็น `page: u32` ตรง ๆ การเรียก `/items` แบบไม่มี query string เลยจะทำให้ `Query<Pagination>`
deserialize ล้มเหลว (เพราะ field ที่ required ไม่มีค่ามาให้) ทั้งที่ในทางความหมายแล้ว "ไม่ส่ง page มา" ควรแปล
ว่า "ใช้ค่า default" ไม่ใช่ "request ผิด" — การใช้ `Option<T>` แล้วเรียก `.unwrap_or(ค่า_default)` ในโค้ด
handler เองคือวิธีที่ถูกต้องในการรองรับ "ค่า default ที่ผู้ใช้ข้ามได้"

ทดสอบจริงทีละกรณี:

**ไม่ส่ง query เลย** — `curl http://127.0.0.1:3002/items`:

```
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 20

page = 1, limit = 10
```

**ส่งแค่ `page`** — `curl "http://127.0.0.1:3002/items?page=2"`:

```
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 20

page = 2, limit = 10
```

**ส่งครบทั้งสอง** — `curl "http://127.0.0.1:3002/items?page=2&limit=50"`:

```
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 20

page = 2, limit = 50
```

**ส่งค่าที่ parse ไม่ผ่าน** — `curl "http://127.0.0.1:3002/items?page=abc"`:

```
HTTP/1.1 400 Bad Request
content-type: text/plain; charset=utf-8
content-length: 71

Failed to deserialize query string: page: invalid digit found in string
```

สังเกตว่าข้อความ rejection ของ `Query<T>` ("Failed to deserialize query string: ...") **มีรูปแบบต่างจาก**
ข้อความ rejection ของ `Path<T>` ("Invalid URL: Cannot parse ...") ที่เห็นในหัวข้อก่อน — ทั้งสองใช้กลไก
`FromRequestParts` เดียวกันในระดับ trait แต่แต่ละ extractor implement การ format ข้อความ error ของตัวเองต่าง
กัน (เพราะ `Path` ใช้ deserializer ที่เขียนเฉพาะสำหรับ URL path segments ส่วน `Query` ใช้
`serde_urlencoded` เป็นตัว deserialize query string ซึ่งมีข้อความ error เป็นของตัวเอง) — สิ่งที่เหมือนกันคือ
ทั้งคู่ตอบ `400 Bad Request` เสมอเมื่อข้อมูลจาก client ไม่ตรงกับ type ที่ handler ต้องการ

### 63.3 Route Methods ที่มากกว่า GET: MethodRouter และ CRUD จริง

Part 62 สอนแค่ `get(handler)` — แต่ REST API จริงต้องรองรับ `POST` (สร้างใหม่), `PUT`/`PATCH` (แก้ไข), และ
`DELETE` (ลบ) บน path เดียวกันหรือ path ที่เกี่ยวข้องกัน Axum ให้ประกาศได้ผ่าน **`MethodRouter`** — ค่าที่คืน
จาก `get(...)`, `post(...)`, `put(...)`, `patch(...)`, `delete(...)` ที่ **chain ต่อกันได้** เพื่อรวมหลาย
method เข้า path เดียว:

```rust
use axum::routing::get;

// path เดียวกัน รองรับทั้ง GET (list) และ POST (create)
Router::new().route("/tickets", get(list_tickets).post(create_ticket));

// path ที่มี id รองรับ GET (อ่านตัวเดียว), PATCH (แก้ไขบางส่วน), DELETE (ลบ)
Router::new().route(
    "/tickets/{id}",
    get(get_ticket).patch(update_ticket).delete(delete_ticket),
);
```

`get(list_tickets)` คืนค่า type `MethodRouter<S>` — การเรียก `.post(create_ticket)` ต่อจากมันไม่ได้สร้าง
`MethodRouter` ใหม่ แต่**เพิ่ม** handler สำหรับ method `POST` เข้าไปใน `MethodRouter` ตัวเดิม ผลคือ object
ตัวเดียวที่รู้วิธีจัดการทั้ง `GET` และ `POST` บน path นั้น — `.route("/tickets", ...)` แค่ผูก path เข้ากับ
`MethodRouter` ตัวนี้เท่านั้น

#### ตัวอย่างจริง: Ticket API แบบ CRUD สมบูรณ์ (state เป็นแค่ placeholder ชั่วคราว)

มาประกอบเป็นระบบเล็ก ๆ ที่ใช้ทุก method — เนื่องจาก **บทนี้ยังไม่ใช่บทที่สอน state management อย่างจริงจัง**
(นั่นคือหน้าที่ของ **Part 64** ที่จะสอน `State<T>` extractor และการออกแบบ shared state อย่างละเอียด) ตัวอย่างนี้
จะใช้ `Arc<Mutex<Vec<Ticket>>>` แบบง่ายที่สุดเท่าที่จำเป็นเพื่อให้ endpoint ทำงานได้จริงแบบ end-to-end — อย่า
มองว่านี่คือวิธี "ที่ถูกต้อง" สำหรับจัดการ state ในแอปจริง มันเป็นเพียง placeholder ที่พอเพียงสำหรับจุดโฟกัสของ
บทนี้คือ**การประกอบ route** เท่านั้น:

```rust
use axum::{
    extract::{Path, State},
    http::StatusCode,
    routing::get,
    Json, Router,
};
use serde::{Deserialize, Serialize};
use std::sync::{Arc, Mutex};

#[derive(Debug, Clone, Serialize)]
struct Ticket {
    id: u32,
    title: String,
    done: bool,
}

#[derive(Debug, Deserialize)]
struct NewTicket {
    title: String,
}

#[derive(Debug, Deserialize)]
struct UpdateTicket {
    title: Option<String>,
    done: Option<bool>,
}

#[derive(Default)]
struct AppState {
    tickets: Mutex<Vec<Ticket>>,
    next_id: Mutex<u32>,
}

type SharedState = Arc<AppState>;

async fn list_tickets(State(state): State<SharedState>) -> Json<Vec<Ticket>> {
    let tickets = state.tickets.lock().unwrap();
    Json(tickets.clone())
}

async fn get_ticket(
    State(state): State<SharedState>,
    Path(id): Path<u32>,
) -> Result<Json<Ticket>, StatusCode> {
    let tickets = state.tickets.lock().unwrap();
    tickets
        .iter()
        .find(|t| t.id == id)
        .cloned()
        .map(Json)
        .ok_or(StatusCode::NOT_FOUND)
}

async fn create_ticket(
    State(state): State<SharedState>,
    Json(new_ticket): Json<NewTicket>,
) -> (StatusCode, Json<Ticket>) {
    let mut next_id = state.next_id.lock().unwrap();
    let id = *next_id;
    *next_id += 1;
    let ticket = Ticket { id, title: new_ticket.title, done: false };
    state.tickets.lock().unwrap().push(ticket.clone());
    (StatusCode::CREATED, Json(ticket))
}

async fn update_ticket(
    State(state): State<SharedState>,
    Path(id): Path<u32>,
    Json(patch): Json<UpdateTicket>,
) -> Result<Json<Ticket>, StatusCode> {
    let mut tickets = state.tickets.lock().unwrap();
    let ticket = tickets.iter_mut().find(|t| t.id == id).ok_or(StatusCode::NOT_FOUND)?;
    if let Some(title) = patch.title {
        ticket.title = title;
    }
    if let Some(done) = patch.done {
        ticket.done = done;
    }
    Ok(Json(ticket.clone()))
}

async fn delete_ticket(State(state): State<SharedState>, Path(id): Path<u32>) -> StatusCode {
    let mut tickets = state.tickets.lock().unwrap();
    let len_before = tickets.len();
    tickets.retain(|t| t.id != id);
    if tickets.len() == len_before {
        StatusCode::NOT_FOUND
    } else {
        StatusCode::NO_CONTENT
    }
}

#[tokio::main]
async fn main() {
    let state = SharedState::new(AppState::default());

    let app = Router::new()
        .route("/tickets", get(list_tickets).post(create_ticket))
        .route(
            "/tickets/{id}",
            get(get_ticket).patch(update_ticket).delete(delete_ticket),
        )
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3003").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

จุดที่ควรสังเกตในโค้ดนี้ (จะกลับมาอธิบายลึกกว่านี้ในหัวข้อ 63.6-63.7):

- **`State<SharedState>` มาก่อน `Path<u32>`/`Json<T>` เสมอ** ในลิสต์ parameter — นี่ไม่ใช่เรื่องบังเอิญ จะ
  อธิบายกฎนี้อย่างละเอียดในหัวข้อ 63.7
- **ค่าที่คืนจาก handler มีหลายรูปแบบ**: `Json<Vec<Ticket>>` ตรง ๆ, `Result<Json<Ticket>, StatusCode>`,
  `(StatusCode, Json<Ticket>)` (tuple ที่มีทั้ง status และ body), และ `StatusCode` เดี่ยว ๆ — ทั้งหมดนี้ทำได้
  เพราะทุก type เหล่านี้ implement `IntoResponse` (หัวข้อ 63.7 จะอธิบายว่าทำไม Axum ยอมให้ signature ของ
  handler แต่ละตัวคืนค่าคนละ type กันได้อย่างอิสระขนาดนี้)

รันจริงแล้วทดสอบทุก endpoint ตามลำดับ (สร้าง → อ่าน → แก้ → ลบ):

```bash
curl -i http://127.0.0.1:3003/tickets
```
```
HTTP/1.1 200 OK
content-type: application/json
content-length: 2

[]
```

```bash
curl -i -X POST http://127.0.0.1:3003/tickets \
  -H 'Content-Type: application/json' \
  -d '{"title":"เครื่องพิมพ์เสีย"}'
```
```
HTTP/1.1 201 Created
content-type: application/json
content-length: 80

{"id":0,"title":"เครื่องพิมพ์เสีย","done":false}
```

```bash
curl -i http://127.0.0.1:3003/tickets/999
```
```
HTTP/1.1 404 Not Found
content-length: 0
```

```bash
curl -i -X PATCH http://127.0.0.1:3003/tickets/0 \
  -H 'Content-Type: application/json' \
  -d '{"done":true}'
```
```
HTTP/1.1 200 OK
content-type: application/json
content-length: 79

{"id":0,"title":"เครื่องพิมพ์เสีย","done":true}
```

```bash
curl -i -X DELETE http://127.0.0.1:3003/tickets/0
```
```
HTTP/1.1 204 No Content
```

ทุก response ด้านบนนี้คือผลลัพธ์จริงจากการรันโปรแกรมและยิง `curl` เข้าไปจริง ๆ ไม่ใช่การเดาว่า Axum "ควร" ตอบ
อะไร — `PATCH` ที่ส่งแค่ `{"done":true}` (ไม่มี `title`) ทำงานถูกต้องเพราะ `UpdateTicket::title` เป็น
`Option<String>` ที่เป็น `None` ได้เมื่อไม่ส่งมา (แพทเทิร์นเดียวกับ `Pagination` ในหัวข้อ query parameter)

#### เมธอดที่ไม่ได้ประกาศไว้: 405 Method Not Allowed

ถ้ามีคนยิง method ที่ path นั้นไม่ได้ประกาศไว้เลย เช่น `PUT /tickets` (ทั้งที่ `/tickets` ประกาศไว้แค่ `GET`
กับ `POST`) Axum จะตอบ:

```bash
curl -i -X PUT http://127.0.0.1:3003/tickets
```
```
HTTP/1.1 405 Method Not Allowed
allow: GET,HEAD,POST
content-length: 0
```

สังเกต header **`allow: GET,HEAD,POST`** — Axum บอกกลับมาตรง ๆ ว่า path นี้รองรับ method อะไรบ้าง (ซึ่งเป็น
พฤติกรรมที่ HTTP spec แนะนำไว้สำหรับ response `405`) และ **`HEAD` โผล่มาโดยที่คุณไม่ได้ประกาศเอง** — Axum เพิ่ม
`HEAD` ให้อัตโนมัติทุกครั้งที่มี `GET` handler เพราะตาม HTTP semantics แล้ว `HEAD` ควรทำงานเหมือน `GET` เป๊ะ
เพียงแต่ไม่ส่ง body กลับมา (ใช้เช็คว่า resource มีอยู่ไหมโดยไม่ต้องโหลด body ทั้งก้อน) — นี่คือความสะดวกที่
Axum จัดการให้โดยอัตโนมัติ ไม่ต้องเขียน handler แยกสำหรับ `HEAD` เอง

### 63.4 Route Nesting และ Grouping: จัดระเบียบแอปที่ใหญ่ขึ้น

ตัวอย่างในหัวข้อ 63.3 มีแค่ resource เดียว (`tickets`) พอแอปจริงมีหลาย resource (users, tickets, orders, ...)
การเขียนทุก `.route(...)` เรียงต่อกันยาว ๆ ใน `main()` จะเริ่มอ่านยากขึ้นเรื่อย ๆ Axum มีสองกลไกสำหรับจัดการ
เรื่องนี้: **`.nest(...)`** (เพิ่ม prefix ให้ router ย่อยทั้งชุด) และ **`.merge(...)`** (รวมสอง router ที่มี
prefix เดียวกัน/ไม่มี prefix เข้าด้วยกัน)

#### `.nest(prefix, router)`: เติม Prefix ให้ Router ย่อยทั้งก้อน

```rust
let api_v1 = Router::new()
    .nest("/users", users_routes::router())
    .nest("/tickets", tickets_routes::router());

let app = Router::new().nest("/api/v1", api_v1);
```

ทุก route ที่ประกาศไว้ใน `users_routes::router()` (เช่น `"/"` หรือ `"/{id}"`) จะถูก**เติม prefix** เข้าไปข้าง
หน้าโดยอัตโนมัติ — route `"/"` ภายใน `users_routes::router()` เมื่อ nest เข้า `"/users"` แล้ว nest เข้า
`"/api/v1"` อีกชั้น จะกลายเป็น path จริงคือ `/api/v1/users` (ไม่ใช่ `/api/v1/users/`) — เรื่อง trailing slash
นี้สำคัญมากและจะพิสูจน์ด้วยการรันจริงในหัวข้อถัดไป (63.5)

**เหตุผลที่ `.nest()` มีประโยชน์มากกว่าการต่อสตริง path เองด้วยมือ**: router ย่อย (`users_routes::router()`)
ไม่จำเป็นต้องรู้เลยว่าตัวเองจะถูก mount ที่ prefix อะไรในระบบสุดท้าย — คุณเขียน route ภายในโมดูล `users_routes`
โดยคิดแค่ว่า "ถ้าฉันเป็น router เดี่ยว ๆ ของตัวเอง ฉันมี `/` และ `/{id}`" แล้วปล่อยให้ `main()` (หรือ router
ระดับบนสุด) เป็นคนตัดสินใจว่าจะ mount โมดูลนี้ไว้ที่ไหน (`/api/v1/users`? `/v2/users`? หรือ mount สองครั้งที่
คนละ prefix เพื่อรองรับ API สองเวอร์ชันพร้อมกัน?) — นี่คือหลักการ **separation of concerns** เดียวกับที่ Part
16 สอนไว้เรื่องการแบ่ง module: โมดูลย่อยไม่ควรรู้เรื่อง "บริบทที่ตัวเองถูกใช้งาน" มากเกินความจำเป็น

#### `.merge(router)`: รวมสอง Router ที่เป็นระดับเดียวกัน

`.merge()` ต่างจาก `.nest()` ตรงที่**ไม่เติม prefix ใด ๆ** — มันแค่เอา route ทั้งหมดของ router อีกตัวมารวมเข้า
กับ router ปัจจุบัน ตรง ๆ ตามที่ประกาศไว้:

```rust
let health_router = Router::new().route("/health", get(health));

let app = Router::new()
    .route("/", get(root))
    .nest("/api/v1", api_v1)
    .merge(health_router);  // "/health" ยังคงเป็น "/health" ไม่ใช่ "/api/v1/health"
```

ใช้ `.merge()` เมื่อ router สองตัวควรอยู่ "ระดับเดียวกัน" ของ path space จริง ๆ (เช่น route สำหรับ health
check หรือ metrics endpoint ที่ควรอยู่นอก versioned API prefix) ส่วน `.nest()` ใช้เมื่อต้องการให้ router
ย่อยทั้งก้อนอยู่ภายใต้ prefix เดียวกัน — เลือกผิดจะได้ path ที่ไม่ตรงกับที่ตั้งใจ (ลืม `.merge()` แล้วใช้
`.nest("/health", health_router)` โดยไม่ตั้งใจ จะได้ path จริงเป็น `/health/health` เพราะภายใน
`health_router` มี route `"/health"` อยู่แล้ว ซ้ำกับ prefix ที่เพิ่งเติมเข้าไป)

#### จัดโครงสร้างเป็น Module ต่อ Resource (เชื่อมกับ Part 16)

แนวทางที่ทำให้แอปจริงจัดการได้ง่ายคือแบ่งแต่ละ resource เป็น**โมดูลของตัวเอง** ที่ export ฟังก์ชันสร้าง
`Router` ออกมา — ในโปรเจกต์จริง แต่ละโมดูลแบบนี้มักแยกเป็นไฟล์ต่างหาก (เช่น `src/routes/users.rs`,
`src/routes/tickets.rs`) ตามที่ Part 16 สอนเรื่องการแบ่งไฟล์เป็นโมดูล แต่เพื่อความกระชับของตัวอย่างในบทเรียน
นี้ จะเขียนเป็น `mod` block หลายตัวไว้ในไฟล์เดียว (โครงสร้างเชิงตรรกะเหมือนกันทุกประการกับการแยกไฟล์จริง):

```rust
use axum::{routing::get, Router};

mod users_routes {
    use axum::{routing::get, Router};

    async fn list_users() -> &'static str {
        "user list"
    }

    async fn get_user() -> &'static str {
        "one user"
    }

    pub fn router() -> Router {
        Router::new()
            .route("/", get(list_users))
            .route("/{id}", get(get_user))
    }
}

mod tickets_routes {
    use axum::{routing::get, Router};

    async fn list_tickets() -> &'static str {
        "ticket list"
    }

    pub fn router() -> Router {
        Router::new().route("/", get(list_tickets))
    }
}

async fn health() -> &'static str {
    "ok"
}

async fn root() -> &'static str {
    "root"
}

#[tokio::main]
async fn main() {
    let api_v1 = Router::new()
        .nest("/users", users_routes::router())
        .nest("/tickets", tickets_routes::router());

    let health_router = Router::new().route("/health", get(health));

    let app = Router::new()
        .route("/", get(root))
        .nest("/api/v1", api_v1)
        .merge(health_router);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3004").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

โครงสร้างนี้อ่านง่ายกว่าการเขียนทุก `.route(...)` เรียงยาว ๆ ใน `main()` เดียวมาก และในโปรเจกต์จริง แต่ละ
`pub fn router() -> Router` แบบนี้จะย้ายไปอยู่ในไฟล์ของตัวเอง (`users_routes.rs`) แล้ว `main.rs` แค่
`mod users_routes;` แล้วเรียก `users_routes::router()` — โครงสร้างโค้ดเหมือนกันเป๊ะ เปลี่ยนแค่ตำแหน่งไฟล์

ทดสอบจริงทุก path (ยืนยันว่าการ nest ซ้อนกันสองชั้นทำงานถูกต้อง):

```bash
curl http://127.0.0.1:3004/                    # -> root
curl http://127.0.0.1:3004/health               # -> ok
curl http://127.0.0.1:3004/api/v1/users         # -> user list
curl http://127.0.0.1:3004/api/v1/users/42      # -> one user
curl http://127.0.0.1:3004/api/v1/tickets       # -> ticket list
```

ทั้งห้าคำสั่งได้ผลลัพธ์ `200 OK` ตรงตามคอมเมนต์ทุกบรรทัด — พิสูจน์ว่า `.nest("/api/v1", ...)` ที่ครอบ
`.nest("/users", ...)` อีกชั้นซ้อนกันได้ถูกต้องจริง ให้ path สุดท้ายเป็น `/api/v1/users` ตามที่คาดไว้

### 63.5 Path Matching อย่างแม่นยำ: Exact, Wildcard, Trailing Slash, และลำดับความสำคัญ

หัวข้อนี้จะพิสูจน์ด้วยการรันจริงว่า Axum ตัดสินใจว่า request หนึ่งตัว "ตรงกับ route ไหน" อย่างไร เมื่อมีหลาย
pattern ที่**อาจจะ**ตรงกันได้พร้อมกัน

#### Exact Match ชนะ Dynamic Parameter เสมอ

ถ้าคุณประกาศทั้ง route ที่ตรงตัวเป๊ะ (`/users/me`) และ route ที่มี dynamic parameter ครอบคลุมกรณีนั้นด้วย
(`/users/{id}`) Axum จะ**เลือก exact match ก่อนเสมอ** ไม่ว่าจะประกาศ route ไหนก่อนในโค้ด:

```rust
use axum::{extract::Path, routing::get, Router};

async fn get_me() -> &'static str {
    "static: /users/me"
}

async fn get_user_by_id(Path(id): Path<String>) -> String {
    format!("dynamic: /users/{{id}} matched id={id}")
}

let app = Router::new()
    .route("/users/me", get(get_me))
    .route("/users/{id}", get(get_user_by_id));
```

ทดสอบจริง:

```bash
curl http://127.0.0.1:3011/users/me
```
```
static: /users/me
```

```bash
curl http://127.0.0.1:3011/users/42
```
```
dynamic: /users/{id} matched id=42
```

`/users/me` ไปที่ handler ของ exact match แม้ว่า `/users/{id}` ก็สามารถ "จับคู่" กับ `me` ได้ในทางเทคนิค
(เพราะ `me` เป็น string ที่ valid สำหรับ `Path<String>`) — Axum ให้**ความจำเพาะเจาะจงของ pattern** (specificity)
เป็นตัวตัดสิน ไม่ใช่ลำดับการประกาศ นี่คือพฤติกรรมที่สมเหตุสมผลและเป็นสิ่งที่นักพัฒนาคาดหวังจาก router ส่วนใหญ่:
เส้นทางที่ "เจาะจงกว่า" ควรมาก่อนเส้นทางที่ "ครอบคลุมกว้างกว่า" เสมอ ไม่ว่าจะเขียนโค้ดในลำดับใด

#### Dynamic Parameter ชนะ Wildcard `{*rest}`

Wildcard (`{*name}`) จับคู่กับ**ส่วนที่เหลือทั้งหมด**ของ path ตั้งแต่จุดนั้นเป็นต้นไป (รวมทุก `/` ที่ตามมา) ใช้
บ่อยสำหรับ serve static file ที่ path ย่อยลึกได้ไม่จำกัด:

```rust
use axum::{extract::Path, routing::get, Router};

async fn exact_about() -> &'static str {
    "exact: /about"
}

async fn catch_all(Path(rest): Path<String>) -> String {
    format!("catch-all matched, rest = {rest:?}")
}

async fn specific_file() -> &'static str {
    "specific: /static/logo.png"
}

let app = Router::new()
    .route("/about", get(exact_about))
    .route("/static/logo.png", get(specific_file))
    .route("/static/{*rest}", get(catch_all));
```

ทดสอบจริงทีละกรณี:

```bash
curl http://127.0.0.1:3005/static/logo.png       # exact match ชนะ wildcard
```
```
specific: /static/logo.png
```

```bash
curl http://127.0.0.1:3005/static/css/app.css    # ไม่มี exact match -> wildcard จับ
```
```
catch-all matched, rest = "css/app.css"
```

สังเกตว่า `rest` ที่ `Path<String>` ดึงออกมาจาก wildcard คือ `"css/app.css"` (**ไม่มี** `/` นำหน้า) — นี่คือ
รูปแบบที่ `{*name}` ให้มาเสมอ: ทุก segment ที่เหลือถูกรวมเป็น string เดียวคั่นด้วย `/` โดยไม่มี `/` นำหน้า

**กรณีที่ต้องระวัง**: wildcard ต้องมีอย่างน้อยหนึ่ง segment ตามหลัง `/static/` เสมอ ทดสอบเรียก path ที่ไม่มี
ส่วนต่อท้ายเลย:

```bash
curl -i http://127.0.0.1:3005/static/
```
```
HTTP/1.1 404 Not Found
content-length: 0
```

```bash
curl -i http://127.0.0.1:3005/static
```
```
HTTP/1.1 404 Not Found
content-length: 0
```

ทั้งสองกรณี**404 จริง** — `{*rest}` ไม่จับคู่กับ "ค่าว่าง" ให้ ต้องมีตัวอักษรอย่างน้อยหนึ่งตัวหลัง `/static/`
เสมอ ถ้าต้องการให้ `/static` (ไม่มี segment ต่อ) ก็ตรงกับ handler บางตัวด้วย ต้องประกาศ route แยกสำหรับ
`/static` (ไม่มี `{*rest}`) เพิ่มเข้ามาต่างหาก

#### Trailing Slash: Axum ไม่ Redirect ให้อัตโนมัติ — 404 ตรง ๆ

นี่คือพฤติกรรมที่คนย้ายมาจาก framework อื่น (เช่น Express.js หรือ Flask ที่มักมี option ให้ redirect
trailing slash อัตโนมัติ) มักเข้าใจผิด ต้องพิสูจน์ด้วยการรันจริงเท่านั้น เพราะเป็นรายละเอียดเชิงพฤติกรรมที่
เปลี่ยนไปได้ตามเวอร์ชัน — จากการทดสอบจริงกับ **axum 0.8.9**: ทดสอบกับ router จากหัวข้อ 63.4 ที่มี route
`/api/v1/users` (จาก `.nest("/users", ...)` ที่ข้างในมี route `"/"`) โดยตั้งใจเรียกแบบมี trailing slash และ
ไม่มี trailing slash:

```bash
curl -i http://127.0.0.1:3004/api/v1/users
```
```
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 9

user list
```

```bash
curl -i http://127.0.0.1:3004/api/v1/users/
```
```
HTTP/1.1 404 Not Found
content-length: 0
```

**ผลจริง 100%: `/api/v1/users` (ไม่มี `/` ท้าย) ตอบ `200 OK` แต่ `/api/v1/users/` (มี `/` ท้าย) ตอบ
`404 Not Found` ตรง ๆ — ไม่มีการ redirect ให้เลย** นี่คือความแตกต่างสำคัญจาก framework บางตัวที่ค่า default
คือ redirect `301`/`308` ระหว่างสองรูปแบบนี้ให้อัตโนมัติ — **Axum ถือว่า path ที่มี/ไม่มี trailing slash คือ
path คนละอันเด็ดขาด** ถ้าคุณต้องการให้ทั้งสองรูปแบบใช้งานได้ ต้องประกาศ route ทั้งสองแบบเอง หรือใช้
middleware เฉพาะสำหรับ normalize path (เรื่อง middleware จะสอนในบทหลัง ๆ ของหลักสูตร) — ผลกระทบเชิงปฏิบัติ:
ถ้า client (เช่น browser หรือ mobile app อื่น) เผลอเติม `/` ท้าย URL ที่เรียก API ของคุณ จะได้ `404` ทันที
ไม่ใช่ redirect ไปที่ path ที่ถูกต้องให้อัตโนมัติ — ควรสื่อสารรูปแบบ URL ที่ถูกต้องให้ชัดเจนใน API
documentation ของทีมเสมอ

#### Route ที่ชนกัน: Panic ตอน Build Router ไม่ใช่ตอน Request

ถ้าคุณประกาศ dynamic parameter สองตัวที่ตำแหน่งเดียวกันของ path (เช่น `/users/{id}` และ `/users/{name}` — ทั้ง
คู่มี dynamic segment ที่ตำแหน่งเดียวกัน แค่ชื่อต่างกัน) Axum **ไม่รอให้ request เข้ามาแล้วค่อยสับสน** — มันจะ
**panic ทันทีตอนเรียก `.route(...)`** (ตอน build router ตั้งแต่ยัง `main()` ยังไม่เริ่ม `serve` เลย):

```rust
async fn a() -> &'static str { "a" }
async fn b() -> &'static str { "b" }

let app = Router::new()
    .route("/users/{id}", get(a))
    .route("/users/{name}", get(b));   // <- panic ตรงนี้
```

รันจริงได้ panic message (พร้อม backtrace เต็มที่ตัดมาแค่ส่วนสำคัญ):

```
thread 'main' panicked at src/bin/conflict.rs:10:10:
Invalid route "/users/{name}": Insertion failed due to conflict with previously registered route: /users/{id}
```

นี่คือการตัดสินใจทางออกแบบที่ตั้งใจของ Axum: **การมี route สองตัวที่กำกวมกันเองในตำแหน่งเดียวกันคือ bug ของ
โค้ด ไม่ใช่สถานการณ์ที่ควรปล่อยให้เกิดขึ้นตอน runtime** ป้องกันได้ตั้งแต่ startup (fail-fast) ดีกว่าให้แอป
รันอยู่แล้วมี request บางตัวพฤติกรรมกำกวมโดยไม่มีใครรู้ล่วงหน้า — ข้อสังเกตเชิงปฏิบัติ: error message บอกชื่อ
route ทั้งสองตัวที่ชนกันตรง ๆ ทำให้ debug ง่าย แค่เปลี่ยนชื่อ parameter ให้ตรงกัน (ทั้งคู่ใช้ `{id}`) หรือรวม
เป็น route เดียวแล้วจัดการความแตกต่างในตัว handler เอง

### 63.6 `Handler` Trait อย่างเจาะลึก: ทำไม Handler รับ Extractor ได้ 0 ถึง 16 ตัว

Part 62 บอกไว้แค่ระดับผิวเผินว่า "ฟังก์ชัน async ที่มี signature เหมาะสมกลายเป็น handler ได้" — หัวข้อนี้จะ
อธิบายกลไกจริงเบื้องหลังคำว่า "เหมาะสม" นั้น

#### `Handler` Trait คืออะไร

Axum มี trait ชื่อ `Handler<T, S>` (`T` คือ marker type ที่บอกว่า handler นี้มี "รูปแบบ" extractor อย่างไร,
`S` คือ state type) — `get()`, `post()`, และ method อื่น ๆ ทั้งหมดของ `MethodRouter` ต้องการ argument ที่
implement trait นี้ ถ้าฟังก์ชันที่คุณส่งเข้าไปไม่ implement `Handler<T, S>` สำหรับ `T`, `S` อะไรเลย จะเกิด
compile error (จะเห็นตัวอย่างจริงในหัวข้อถัดไป) — คำถามคือ **ฟังก์ชันของคุณ implement trait นี้ได้ยังไง
ทั้งที่คุณไม่ได้เขียน `impl Handler for ...` เองเลย?**

#### `impl_handler!`: Macro ที่ Generate Implementation ให้ทุก Arity

คำตอบอยู่ใน source code ของ axum เอง (ไฟล์ `src/handler/mod.rs`) — มี macro ชื่อ `impl_handler!` ที่
generate `impl<F, ...> Handler<(...), S> for F` ให้ทุกจำนวน extractor ตั้งแต่ 0 ถึง 16 ตัว โดยเรียกผ่าน macro
อีกตัวชื่อ `all_the_tuples!` ที่ขยายเป็นรายการ arity ทีละระดับ (`[]`, `[T1]`, `[T1, T2]`, ... จนถึง `[T1..T15]`
พร้อม `T16` ตัวสุดท้าย) — โครงสร้างของ implementation แต่ละ arity หน้าตาประมาณนี้ (ย่อจาก source จริงของ axum
0.8.9 เพื่ออธิบายหลักการ ไม่ใช่โค้ดที่ต้องเขียนเอง):

```rust
// (ประมาณโครงสร้างจริงจาก axum::handler — เพื่ออธิบายหลักการเท่านั้น ไม่ต้องเขียนเอง)
impl<F, Fut, S, Res, $($ty,)* $last> Handler<(...), S> for F
where
    F: FnOnce($($ty,)* $last,) -> Fut + Clone + Send + Sync + 'static,
    Fut: Future<Output = Res> + Send,
    Res: IntoResponse,
    $( $ty: FromRequestParts<S> + Send, )*   // <- extractor ทุกตัว "ก่อน" ตัวสุดท้าย
    $last: FromRequest<S, M> + Send,          // <- extractor ตัวสุดท้ายเท่านั้น
{
    fn call(self, req: Request, state: S) -> Self::Future {
        let (mut parts, body) = req.into_parts();  // แยก request เป็นสองส่วน: headers/path/... กับ body
        Box::pin(async move {
            // ดึงค่าจาก "parts" (ไม่แตะ body เลย) ทีละตัวตามลำดับที่ประกาศในฟังก์ชัน
            $(
                let $ty = match $ty::from_request_parts(&mut parts, &state).await {
                    Ok(value) => value,
                    Err(rejection) => return rejection.into_response(),
                };
            )*
            // ประกอบ request กลับมาเพื่อดึง "body" ให้ extractor ตัวสุดท้ายเท่านั้น
            let req = Request::from_parts(parts, body);
            let $last = match $last::from_request(req, &state).await {
                Ok(value) => value,
                Err(rejection) => return rejection.into_response(),
            };
            self($($ty,)* $last,).await.into_response()
        })
    }
}
```

จากโค้ดต้นทางนี้ เราเห็น**เหตุผลเชิงลึกสามข้อ**ที่อธิบายพฤติกรรมทั้งหมดที่เห็นมาตลอดบทนี้ได้:

1. **ทำไมมี arity limit ที่ 16**: เพราะ `all_the_tuples!` เป็น macro ที่ต้อง**เขียนรายการ tuple ไว้ตายตัว**
   ทีละระดับ (`[T1]`, `[T1, T2]`, ..., `[T1, ..., T15], T16`) — เนื่องจาก Rust ไม่มี variadic generics (generic
   ที่รับจำนวน type parameter ไม่จำกัด) ทีมงาน axum จึงต้อง generate โค้ดสำหรับ arity แต่ละระดับแยกกันทีละ
   ระดับด้วย macro (คล้ายกับที่ `std` เองก็ implement trait บางตัวให้ tuple แค่ถึงขนาดหนึ่งเท่านั้น เช่น
   `Debug` ให้ tuple ถึง 12 element) — axum เลือกหยุดที่ 16 เป็นจำนวนที่มากเกินพอสำหรับการใช้งานจริงเกือบทั้งหมด
   (ถ้า handler ตัวไหนต้องการ extractor เกิน 16 ตัวจริง ๆ นั่นมักเป็นสัญญาณว่า handler นั้นทำหน้าที่มากเกินไป
   ควรแยกความรับผิดชอบ หรือรวม extractor หลายตัวเป็น struct เดียวผ่าน `#[derive(FromRequestParts)]` — เทคนิค
   ระดับสูงกว่าที่จะเห็นในบทถัด ๆ ไป)
2. **ทำไม extractor ทุกตัว "ก่อน" ตัวสุดท้าย ต้อง implement `FromRequestParts`**: เพราะโค้ดในบล็อกแรก
   (`$( let $ty = ... )*`) ทำงานกับ **`parts`** เท่านั้น (`&mut parts`) — `parts` คือ ส่วนหัวของ request
   (method, URI, headers) ที่**ไม่มี body รวมอยู่** เลย ในทางเทคนิคของ `http` crate ที่ axum ใช้เป็นฐาน
   `Request` ถูกแยกเป็นสองส่วนได้ด้วย `.into_parts()` — ส่วน `Parts` (headers/URI/method) กับส่วน body
   (`Body`) — extractor ที่ implement แค่ `FromRequestParts` (เช่น `Path<T>`, `Query<T>`, `HeaderMap`,
   `State<T>`) จึงดึงค่าได้จาก `parts` โดยไม่ต้องแตะ body เลย ดึงกี่ตัวก็ได้ไม่มีปัญหา เพราะ `parts` ยังอยู่
   ครบไม่ได้ถูก "กิน" ไปไหน
3. **ทำไมมีแค่ตัวสุดท้ายเท่านั้นที่ implement `FromRequest` (ไม่ใช่ `FromRequestParts`)**: เพราะโค้ดต้อง
   **ประกอบ `parts` กับ `body` กลับเป็น `Request` เต็มตัวอีกครั้ง** (`Request::from_parts(parts, body)`)
   ก่อนจะเรียก `$last::from_request(req, &state)` — และ `body` ใน HTTP นั้นเป็น **stream ที่อ่านได้ครั้งเดียว**
   (คล้ายกับ `Iterator` ที่ครั้งหนึ่งดึงค่าออกมาแล้วก็หายไปจากตัว iterator เดิม ไม่ใช่ข้อมูลที่ copy ไปมาได้อย่าง
   `parts`) — ถ้ามี extractor สองตัวที่ต้องการอ่าน body พร้อมกัน ตัวที่สองจะอ่านได้ "ไม่มีอะไรเหลือ" เพราะตัว
   แรกอ่านไปหมดแล้ว นี่คือเหตุผลพื้นฐานที่สุดว่าทำไม **extractor ที่กิน body ได้แค่ตัวเดียวต่อ handler
   และต้องเป็นตัวสุดท้ายเท่านั้น** — เป็นข้อจำกัดที่มาจากธรรมชาติของ HTTP body เอง ไม่ใช่ข้อจำกัดที่ Axum
   สร้างขึ้นมาเองโดยไม่มีเหตุผล

หัวข้อ 63.7 จะพิสูจน์ทั้งสามข้อนี้ด้วย compile error จริง

### 63.7 Extractor Order สำคัญ: Body-Consuming Extractor ต้องมาตัวสุดท้ายเท่านั้น

จากหลักการในหัวข้อ 63.6 สรุปเป็นกฎที่ต้องจำได้ทันที: **extractor ที่ "กิน" request body ได้ (`Json<T>`,
`Form<T>`, `String`, `Bytes`, `axum::body::Body`) ต้องเป็น parameter สุดท้ายของ handler เท่านั้น** extractor
ตัวอื่นทั้งหมด (`Path<T>`, `Query<T>`, `State<T>`, `HeaderMap`, ...) ที่อ่านได้จาก `parts` เพียงอย่างเดียว
วางไว้ตำแหน่งไหนก่อนตัวสุดท้ายก็ได้ (แม้ order ระหว่างพวกนี้เองจะไม่สำคัญเชิง type-checking แต่ก็ยังมีผลต่อ
"ลำดับที่แต่ละ extractor ถูก`.await`" อยู่ดี ซึ่งปกติไม่กระทบผลลัพธ์เพราะพวกมันแค่อ่าน ไม่ได้แก้ไข state ที่มี
side effect ข้าม extractor)

#### ตัวอย่างที่ผิด: วาง `Json<T>` ไว้ก่อน `Path<T>`

```rust
use axum::{routing::post, Json, Router};
use serde::Deserialize;

#[derive(Deserialize)]
struct CreateTicket {
    title: String,
}

// ผิด: Json<T> (body-consuming) อยู่ก่อน Path (parts-only) — ต้องสลับให้ Path มาก่อน
async fn create_ticket(
    Json(payload): Json<CreateTicket>,
    axum::extract::Path(project_id): axum::extract::Path<u32>,
) -> String {
    format!("project {project_id}: {}", payload.title)
}

let app: Router = Router::new().route("/projects/{project_id}/tickets", post(create_ticket));
```

compile จริงได้ error (ยังไม่ใส่ `#[axum::debug_handler]`):

```
error[E0277]: the trait bound `fn(Json<CreateTicket>, ...) -> ... {create_ticket}: Handler<_, _>` is not satisfied
   --> src/bin/bad_order.rs:20:82
    |
 20 |     let app: Router = Router::new().route("/projects/{project_id}/tickets", post(create_ticket));
    |                                                                             ---- ^^^^^^^^^^^^^ the trait `Handler<_, _>` is not implemented for fn item `fn(Json<CreateTicket>, Path<u32>) -> ... {create_ticket}`
    |                                                                             |
    |                                                                             required by a bound introduced by this call
    |
    = note: Consider using `#[axum::debug_handler]` to improve the error message
note: required by a bound in `post`
```

error message นี้**บอกแค่ว่า** `create_ticket` ไม่ implement `Handler<_, _>` — ไม่ได้บอกตรง ๆ ว่า "เพราะ
`Json` อยู่ผิดตำแหน่ง" เลย ผู้เริ่มต้นที่เจอ error นี้ครั้งแรกมักงงว่าโค้ดผิดตรงไหน (เพราะ signature ของ
ฟังก์ชันดู "ถูกต้อง" ในสายตาคนอ่านทั่วไป — มี `Json<T>` มี `Path<T>` ครบ) แต่ตัว compiler เห็นแค่ว่าไม่มี
implementation ของ `Handler` ตัวไหนที่ตรงกับ signature นี้เลย (เพราะ `impl_handler!` generate ให้เฉพาะ
รูปแบบที่ตัวสุดท้ายเป็น `FromRequest` เท่านั้น — ในที่นี้ตัวสุดท้ายคือ `Path<u32>` ที่ implement แค่
`FromRequestParts` ไม่ใช่ `FromRequest` เต็มรูปแบบ ผลคือไม่มี arity ไหนใน `impl_handler!` ที่ match ได้)

สังเกตว่า error message เองก็แนะนำตรง ๆ ว่า **"Consider using `#[axum::debug_handler]` to improve the
error message"** — มาลองทำตามคำแนะนำนี้กันในหัวข้อถัดไป

#### `#[axum::debug_handler]`: แปลง Error ให้อ่านออกทันที

`#[axum::debug_handler]` เป็น **attribute macro** (concept เดียวกับที่ Part 44 สอน `#[proc_macro_attribute]`)
ที่มาจาก crate `axum-macros` (ต้องเปิด feature `"macros"` ของ `axum` — `cargo add axum --features macros`
ถ้ายังไม่มี) มันไม่ได้เปลี่ยนพฤติกรรมของโปรแกรมตอน runtime เลยแม้แต่นิดเดียว — งานของมันคือ**ตรวจสอบ
signature ของฟังก์ชันตอน compile time แบบเจาะจง** (แทนที่จะรอให้ trait bound `Handler<T, S>` ล้มเหลวแบบ
กว้าง ๆ) แล้ว emit compile error ของตัวเองที่อธิบายปัญหาตรงประเด็นกว่า — แนวคิดนี้เหมือนกับที่ Part 44/45
อธิบายว่า attribute macro "อ่าน" AST ของ item ที่มันประดับอยู่ได้ทั้งหมดก่อนที่โค้ดจริงจะถูก generate ออกมา จึง
สามารถเช็คกฎเฉพาะทาง (เช่น "extractor ตัวไหนกิน body ได้") ที่ trait bound ทั่วไปเช็คไม่ได้ตรง ๆ

ใส่ attribute เข้าไปกับฟังก์ชันเดิม (**เหมือนเดิมทุกอย่าง แค่เพิ่มบรรทัดเดียว**):

```rust
#[axum::debug_handler]
async fn create_ticket(
    Json(payload): Json<CreateTicket>,
    axum::extract::Path(project_id): axum::extract::Path<u32>,
) -> String {
    format!("project {project_id}: {}", payload.title)
}
```

compile จริงได้ error ที่ชัดเจนขึ้นมาก (โผล่มา**ก่อน** error `E0277` เดิมที่ยังคงอยู่):

```
error: `Json<_>` consumes the request body and thus must be the last argument to the handler function
  --> src/bin/bad_order_debug.rs:11:20
   |
11 |     Json(payload): Json<CreateTicket>,
   |                    ^^^^
```

ข้อความนี้บอกตรงประเด็น 100%: **ปัญหาคือ `Json<_>`, ปัญหาคือมันกิน body, และคำตอบคือย้ายมันไปตัวสุดท้าย** —
ไม่ต้องนั่งไล่อ่าน trait bound ยาว ๆ หรือเดาเอาเองว่า compiler พยายามบอกอะไร นี่คือเหตุผลที่แนะนำให้ใส่
`#[axum::debug_handler]` ไว้กับ handler ทุกตัวระหว่างพัฒนา (ถอดออกได้ตอน production ถ้าต้องการ เพราะมันไม่มี
runtime cost แต่ก็ทิ้งไว้ได้เพราะไม่กระทบ performance เลย เนื่องจากมันไม่ generate โค้ดเพิ่มเข้าไปในกรณีที่
ไม่มีปัญหา)

#### วิธีแก้: ย้าย `Path<T>` (parts-only) มาก่อน `Json<T>` (body-consuming)

```rust
use axum::{extract::Path, routing::post, Json, Router};
use serde::Deserialize;

#[derive(Deserialize)]
struct CreateTicket {
    title: String,
}

#[axum::debug_handler]
async fn create_ticket(
    Path(project_id): Path<u32>,   // parts-only extractor มาก่อน
    Json(payload): Json<CreateTicket>,  // body-consuming extractor มาตัวสุดท้าย
) -> String {
    format!("project {project_id}: {}", payload.title)
}

let app: Router = Router::new().route("/projects/{project_id}/tickets", post(create_ticket));
```

compile ผ่านสะอาด ไม่มี error หรือ warning เลย และรันจริงทดสอบด้วย curl:

```bash
curl -i -X POST http://127.0.0.1:3006/projects/7/tickets \
  -H 'Content-Type: application/json' -d '{"title":"test"}'
```
```
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 15

project 7: test
```

#### กรณีที่มี Body-Consuming Extractor สองตัว: ผิดแน่นอน ไม่ว่าจะจัดตำแหน่งยังไง

ถ้าลอง**ใส่ extractor ที่กิน body สองตัว**ในฟังก์ชันเดียว (ไม่ว่าจะเรียงลำดับไหนก็ตาม) เช่น `Json<T>` คู่กับ
`Bytes`:

```rust
#[axum::debug_handler]
async fn create_ticket(
    Json(payload): Json<CreateTicket>,
    body_bytes: axum::body::Bytes,
) -> String {
    format!("{} bytes, title={}", body_bytes.len(), payload.title)
}
```

compile จริงได้ error ที่ตรงประเด็นอีกครั้ง:

```
error: Can't have two extractors that consume the request body. `Json<_>` and `Bytes` both do that.
  --> src/bin/two_bodies.rs:11:5
   |
11 |     Json(payload): Json<CreateTicket>,
   |     ^^^^
```

ข้อความนี้ตรงกับเหตุผลเชิงลึกในหัวข้อ 63.6 ข้อ 3 เป๊ะ ๆ: body เป็น stream ที่อ่านได้ครั้งเดียวเท่านั้น มีตัว
"กิน" ได้ตัวเดียวต่อ request — ถ้าโค้ดของคุณต้องการทั้ง raw bytes และข้อมูลที่ parse แล้วจาก body เดียวกัน
วิธีที่ถูกคือดึงมาแค่ตัวเดียว (เช่น `Bytes` ตัวเดียว) แล้ว parse เองในตัว handler ไม่ใช่ขอสอง extractor
พร้อมกัน

#### Arity Limit ตัวจริง: เกิน 16 Argument แล้วเกิดอะไรขึ้น

ทดสอบ arity limit จริงด้วย handler ที่มี 17 extractor (ใช้ extractor เปล่า ๆ ที่ implement
`FromRequestParts` เอง เพื่อไม่ให้ชนกับ lint อื่นของ `Path` ที่ตรวจจับการใช้ `Path<_>` ซ้ำหลายตัว):

```
error: Handlers cannot take more than 16 arguments. Use `(a, b): (ExtractorA, ExtractorA)` to further nest extractors
  --> src/bin/arity17.rs:22:5
```

(ข้อความนี้มาจาก `#[axum::debug_handler]` เช่นเดิม — ไม่มี macro นี้จะเห็นแค่ `E0277` แบบเดิมที่บอกไม่ตรง
ประเด็นว่าปัญหาคืออะไร) ข้อความ**บอกวิธีแก้ไว้ตรง ๆ ด้วย**: ถ้าต้องการ extractor เกิน 16 ตัวจริง ๆ ให้รวม
บางตัวเข้าเป็น tuple (`(a, b): (ExtractorA, ExtractorB)`) เพื่อลดจำนวน "parameter ระดับบนสุด" ของฟังก์ชันลง
— แต่ในทางปฏิบัติ การมาถึงจุดที่ handler ต้องการ extractor 16+ ตัวมักหมายความว่า handler นั้นทำงานหลายอย่าง
เกินไปในฟังก์ชันเดียว ควรพิจารณาแยกความรับผิดชอบออกเป็นหลายฟังก์ชัน/หลาย endpoint มากกว่าพยายามยัดทุกอย่าง
เข้า handler เดียว

### 63.8 คืนค่า Response ที่ซับซ้อนขึ้น: `impl IntoResponse` สำหรับ Type ของแอปเอง

ตลอดบทนี้ handler คืนค่าได้หลายรูปแบบ (`String`, `Json<T>`, `(StatusCode, Json<T>)`, `Result<Json<T>,
StatusCode>`) เพราะทุก type เหล่านี้ implement trait **`IntoResponse`** — trait ที่นิยามว่า "แปลงค่านี้เป็น
`Response` (โครงสร้าง response ของ HTTP ที่มี status, headers, body) ได้อย่างไร" Axum เขียน implementation ของ
`IntoResponse` ให้ type ที่ใช้บ่อยไว้แล้วทั้งหมด (`String`, `&'static str`, `StatusCode`, `Json<T>`, tuple ที่
รวม `StatusCode` กับ body, `Result<T, E>` ที่ทั้ง `T` และ `E` implement `IntoResponse`, ...) — แต่คุณสามารถ
implement `IntoResponse` ให้ **type ของแอปตัวเอง** ได้เช่นกัน เพื่อควบคุมว่า response ที่ส่งกลับหน้าตาเป็น
อย่างไรอย่างละเอียด (เช่นเติม header พิเศษ)

```rust
use axum::{
    http::StatusCode,
    response::{IntoResponse, Response},
    routing::post,
    Json, Router,
};
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize)]
struct Ticket {
    id: u32,
    title: String,
}

// custom "success" response ที่ห่อ Ticket ไว้พร้อมกำหนด status code + header เอง
struct CreatedTicket(Ticket);

impl IntoResponse for CreatedTicket {
    fn into_response(self) -> Response {
        let mut response = (StatusCode::CREATED, Json(self.0)).into_response();
        response
            .headers_mut()
            .insert("X-Resource-Kind", "ticket".parse().unwrap());
        response
    }
}

#[derive(Deserialize)]
struct NewTicket {
    title: String,
}

async fn create_ticket(Json(new_ticket): Json<NewTicket>) -> CreatedTicket {
    CreatedTicket(Ticket { id: 1, title: new_ticket.title })
}

let app = Router::new().route("/tickets", post(create_ticket));
```

สังเกตว่า `impl IntoResponse for CreatedTicket` **เรียกใช้ `IntoResponse` ของ `(StatusCode, Json<Ticket>)`
ที่ Axum เขียนไว้ให้แล้ว** (`(StatusCode::CREATED, Json(self.0)).into_response()`) แล้ว**ปรับแต่งเพิ่ม**บน
`Response` ที่ได้มา (`response.headers_mut().insert(...)`) — นี่คือแพทเทิร์นทั่วไปที่ดีมากในการเขียน
`IntoResponse` เอง: **ไม่ต้องสร้าง `Response` จากศูนย์** แค่ประกอบ type ที่มี `IntoResponse` อยู่แล้วแล้ว
ปรับแต่งผลลัพธ์เพิ่มเท่าที่ต้องการ

รันจริงและทดสอบด้วย curl เห็น header ที่เพิ่มมาจริง:

```bash
curl -i -X POST http://127.0.0.1:3009/tickets \
  -H 'Content-Type: application/json' -d '{"title":"test custom response"}'
```
```
HTTP/1.1 201 Created
content-type: application/json
x-resource-kind: ticket
content-length: 39

{"id":1,"title":"test custom response"}
```

header **`x-resource-kind: ticket`** มาจาก `impl IntoResponse` ที่เขียนเอง ยืนยันว่ากลไกทำงานจริง — นี่คือแค่
"ทางเลือกที่แสดงกลไก" ของการคืน success response แบบกำหนดเอง ในบทนี้จะยังไม่ลงรายละเอียดการออกแบบ error
response ทั้งระบบ (เช่นทำให้ error ทุกแบบของแอปแปลงเป็น JSON error format ที่สอดคล้องกันหมด) เพราะนั่นคือหัวข้อ
เต็มรูปแบบของ **Part 66** ที่จะสอน error handling ของ Axum โดยเฉพาะ — capstone ท้ายบทนี้ (หัวข้อ 63.9) จะใช้
`impl IntoResponse` แบบง่าย ๆ สำหรับ error type หนึ่งตัวพอให้ endpoint ตอบ error เป็น JSON ที่อ่านง่ายได้ แต่ยัง
ไม่ใช่การออกแบบ error handling แบบสมบูรณ์

### 63.9 Capstone: Bookshelf API — ระบบยืมหนังสือแบบสมบูรณ์

มาประกอบทุกหัวข้อของบทนี้เข้าด้วยกันเป็น REST API เล็ก ๆ ที่ทำงานได้จริงทั้งระบบ: **ระบบยืมหนังสือ** ที่มี:

- Path parameter (`/books/{id}`)
- Query parameter สำหรับ filter + pagination (`?author=...&page=...&limit=...`)
- Route หลาย method บน resource เดียว (`GET`/`POST` ที่ `/books`, `GET` ที่ `/books/{id}`, `POST` ที่
  `/books/{id}/borrow`)
- Nested router (`/api/v1/books/...`) รวมกับ router อื่นด้วย `.merge()` (`/health`)
- Custom `IntoResponse` สำหรับ error (`ApiError`) ที่ทำให้ error ทุกจุดในระบบตอบกลับเป็น JSON รูปแบบเดียวกัน

```rust
use axum::{
    extract::{Path, Query, State},
    http::StatusCode,
    response::{IntoResponse, Response},
    routing::get,
    Json, Router,
};
use serde::{Deserialize, Serialize};
use std::sync::{Arc, Mutex};

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

#[derive(Debug, Deserialize)]
struct BookQuery {
    author: Option<String>,
    page: Option<u32>,
    limit: Option<u32>,
}

struct AppState {
    books: Mutex<Vec<Book>>,
    next_id: Mutex<u32>,
}

type SharedState = Arc<AppState>;

// error type ของแอป: implement IntoResponse ครั้งเดียว ใช้ได้กับทุก endpoint ที่คืน Result<_, ApiError>
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

mod books_routes {
    use super::*;

    async fn list_books(
        State(state): State<SharedState>,
        Query(query): Query<BookQuery>,
    ) -> Json<Vec<Book>> {
        let books = state.books.lock().unwrap();
        let filtered: Vec<Book> = books
            .iter()
            .filter(|b| match &query.author {
                Some(author) => b.author.eq_ignore_ascii_case(author),
                None => true,
            })
            .cloned()
            .collect();

        let page = query.page.unwrap_or(1).max(1);
        let limit = query.limit.unwrap_or(10).max(1);
        let start = ((page - 1) * limit) as usize;
        let page_slice: Vec<Book> =
            filtered.into_iter().skip(start).take(limit as usize).collect();
        Json(page_slice)
    }

    async fn get_book(
        State(state): State<SharedState>,
        Path(id): Path<u32>,
    ) -> Result<Json<Book>, ApiError> {
        let books = state.books.lock().unwrap();
        books
            .iter()
            .find(|b| b.id == id)
            .cloned()
            .map(Json)
            .ok_or(ApiError {
                status: StatusCode::NOT_FOUND,
                message: format!("ไม่พบหนังสือ id={id}"),
            })
    }

    async fn create_book(
        State(state): State<SharedState>,
        Json(new_book): Json<NewBook>,
    ) -> (StatusCode, Json<Book>) {
        let mut next_id = state.next_id.lock().unwrap();
        let id = *next_id;
        *next_id += 1;
        let book = Book { id, title: new_book.title, author: new_book.author, available: true };
        state.books.lock().unwrap().push(book.clone());
        (StatusCode::CREATED, Json(book))
    }

    async fn borrow_book(
        State(state): State<SharedState>,
        Path(id): Path<u32>,
    ) -> Result<Json<Book>, ApiError> {
        let mut books = state.books.lock().unwrap();
        let book = books.iter_mut().find(|b| b.id == id).ok_or(ApiError {
            status: StatusCode::NOT_FOUND,
            message: format!("ไม่พบหนังสือ id={id}"),
        })?;
        if !book.available {
            return Err(ApiError {
                status: StatusCode::CONFLICT,
                message: format!("หนังสือ id={id} ถูกยืมไปแล้ว"),
            });
        }
        book.available = false;
        Ok(Json(book.clone()))
    }

    pub fn router() -> Router<SharedState> {
        Router::new()
            .route("/", get(list_books).post(create_book))
            .route("/{id}", get(get_book))
            .route("/{id}/borrow", axum::routing::post(borrow_book))
    }
}

async fn health() -> &'static str {
    "ok"
}

#[tokio::main]
async fn main() {
    let state = SharedState::new(AppState {
        books: Mutex::new(vec![
            Book {
                id: 0,
                title: "The Rust Programming Language".into(),
                author: "Steve Klabnik".into(),
                available: true,
            },
            Book {
                id: 1,
                title: "Programming Rust".into(),
                author: "Jim Blandy".into(),
                available: true,
            },
        ]),
        next_id: Mutex::new(2),
    });

    let api_v1 = Router::new().nest("/books", books_routes::router());
    let health_router: Router<SharedState> = Router::new().route("/health", get(health));

    let app = Router::new()
        .nest("/api/v1", api_v1)
        .merge(health_router)
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3010").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

ข้อสังเกตเกี่ยวกับโครงสร้าง type ของ `Router` ที่สำคัญ: `books_routes::router()` คืน **`Router<SharedState>`**
(ระบุ type parameter ของ state ตรง ๆ) ไม่ใช่ `Router<()>` เฉย ๆ — เพราะ handler ภายในโมดูลนี้ใช้
`State<SharedState>` extractor ซึ่งต้องการให้ router ที่ handler ถูกผูกอยู่มี state type ตรงกัน `.with_state(state)`
เรียกที่ระดับบนสุดเพียงครั้งเดียวหลังจากรวม (`nest`/`merge`) ทุกอย่างเข้าด้วยกันแล้ว เพื่อ "เติม" state ค่าจริง
ให้ router ทั้งก้อนกลายเป็น `Router` (ไม่มี type parameter ค้าง) ที่ `axum::serve` รับได้ — รายละเอียดเรื่อง
state ทั้งหมดนี้ Part 64 จะอธิบายลึกกว่านี้อีกมาก บทนี้แค่แสดงว่ามันทำงานร่วมกับ nest/merge ได้จริง

#### Curl Transcript เต็มรูปแบบ (รันจริงทุกคำสั่ง)

```bash
curl -i http://127.0.0.1:3010/health
```
```
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 2

ok
```

```bash
curl -i http://127.0.0.1:3010/api/v1/books
```
```
HTTP/1.1 200 OK
content-type: application/json
content-length: 167

[{"id":0,"title":"The Rust Programming Language","author":"Steve Klabnik","available":true},{"id":1,"title":"Programming Rust","author":"Jim Blandy","available":true}]
```

```bash
curl -i "http://127.0.0.1:3010/api/v1/books?author=Jim%20Blandy"
```
```
HTTP/1.1 200 OK
content-type: application/json
content-length: 76

[{"id":1,"title":"Programming Rust","author":"Jim Blandy","available":true}]
```

```bash
curl -i "http://127.0.0.1:3010/api/v1/books?limit=1&page=2"
```
```
HTTP/1.1 200 OK
content-type: application/json
content-length: 76

[{"id":1,"title":"Programming Rust","author":"Jim Blandy","available":true}]
```

```bash
curl -i -X POST http://127.0.0.1:3010/api/v1/books \
  -H 'Content-Type: application/json' \
  -d '{"title":"Zero To Production In Rust","author":"Luca Palmieri"}'
```
```
HTTP/1.1 201 Created
content-type: application/json
content-length: 87

{"id":2,"title":"Zero To Production In Rust","author":"Luca Palmieri","available":true}
```

```bash
curl -i http://127.0.0.1:3010/api/v1/books/999
```
```
HTTP/1.1 404 Not Found
content-type: application/json
content-length: 55

{"error":"ไม่พบหนังสือ id=999"}
```

```bash
curl -i -X POST http://127.0.0.1:3010/api/v1/books/2/borrow
```
```
HTTP/1.1 200 OK
content-type: application/json
content-length: 88

{"id":2,"title":"Zero To Production In Rust","author":"Luca Palmieri","available":false}
```

```bash
curl -i -X POST http://127.0.0.1:3010/api/v1/books/2/borrow
```
```
HTTP/1.1 409 Conflict
content-type: application/json
content-length: 75

{"error":"หนังสือ id=2 ถูกยืมไปแล้ว"}
```

ทุกบรรทัดด้านบนคือผลลัพธ์จริงจากการรัน binary นี้ด้วย `cargo run` แล้วยิง `curl` เข้าไปตามลำดับ — สังเกตการ
ยืมซ้ำครั้งที่สอง (`borrow` id=2 อีกครั้ง) ตอบ `409 Conflict` พร้อมข้อความ JSON ที่มาจาก `ApiError` ที่เขียน
`impl IntoResponse` ไว้เพียงครั้งเดียวแต่ใช้ซ้ำได้กับทุก endpoint ที่คืน `Result<_, ApiError>` — นี่คือ
ประโยชน์เชิงปฏิบัติของการเขียน `impl IntoResponse` ให้ error type ของแอปเอง: ทุก endpoint ตอบ error ด้วย
รูปแบบ JSON ที่สอดคล้องกันหมดโดยไม่ต้องเขียนโค้ดแปลง error เป็น JSON ซ้ำในทุกฟังก์ชัน

## กับดักที่พบบ่อย (Common Pitfalls)

**1. ใส่ `Path<T>` หลายตัวแยกกันในฟังก์ชันเดียว แทนที่จะใช้ tuple/struct**

```rust
// ผิด
async fn get_user_post_wrong(
    Path(user_id): Path<u32>,
    Path(post_id): Path<u32>,
) -> String {
    format!("user_id = {user_id}, post_id = {post_id}")
}
```

ใช้ `#[axum::debug_handler]` แล้ว compile จริงได้ error ที่บอกตรงประเด็น:

```
error: Multiple parameters must be extracted with a tuple `Path<(_, _)>` or a struct `Path<YourParams>`, not by applying multiple `Path<_>` extractors
```

วิธีแก้: รวมเป็น `Path((user_id, post_id)): Path<(u32, u32)>` หรือ struct-based extraction ตามที่สอนไว้ใน
หัวข้อ 63.1 — Axum มองว่า path คือ "ข้อมูลชุดเดียว" ที่ต้องดึงครบในครั้งเดียว ไม่ใช่ดึงทีละส่วนแยกกันหลาย
extractor

**2. ลืมว่า body-consuming extractor ต้องมาตัวสุดท้าย**

```rust
// ผิด: Json มาก่อน Path
async fn create_ticket(
    Json(payload): Json<CreateTicket>,
    axum::extract::Path(project_id): axum::extract::Path<u32>,
) -> String { /* ... */ }
```

error จริงที่ได้ (ไม่มี `#[axum::debug_handler]`):

```
error[E0277]: the trait bound `fn(Json<CreateTicket>, ...) -> ... {create_ticket}: Handler<_, _>` is not satisfied
```

หรือ (มี `#[axum::debug_handler]`):

```
error: `Json<_>` consumes the request body and thus must be the last argument to the handler function
```

วิธีแก้: ย้าย `Json<T>`/`Form<T>`/`Bytes`/`String` ไปเป็น parameter สุดท้ายเสมอ — ใส่
`#[axum::debug_handler]` ไว้ตอน dev เพื่อเห็น error แบบที่สองเสมอ ประหยัดเวลา debug ได้มาก (ต้องเปิด feature
`"macros"` ของ `axum` ก่อน ไม่งั้นจะเจอ error คนละแบบ: `error[E0433]: failed to resolve: could not find
\`debug_handler\` in \`axum\`` — เพราะฟังก์ชันนี้ถูก `#[cfg(feature = "macros")]` กันไว้ ไม่ compile เข้ามา
เลยถ้าไม่เปิด feature)

**3. คาดหวังว่า trailing slash จะ redirect ให้อัตโนมัติ**

```
curl http://host/api/v1/users    -> 200 OK
curl http://host/api/v1/users/   -> 404 Not Found  (ไม่ redirect!)
```

Axum ถือว่า path ที่มี/ไม่มี `/` ท้ายเป็น path คนละอันเด็ดขาด ไม่มีการ redirect ให้อัตโนมัติเหมือนบาง
framework — ถ้าต้องการรองรับทั้งสองแบบ ต้องประกาศ route ทั้งสองรูปแบบเอง หรือจัดการผ่าน middleware (จะสอนใน
บทหลัง ๆ) เขียน API documentation ให้ชัดเจนเรื่อง URL ที่ถูกต้องเสมอเพื่อลดโอกาสเจอปัญหานี้ในทีม

**4. ประกาศ route ที่ dynamic segment ชนกันโดยไม่ตั้งใจ**

```rust
Router::new()
    .route("/users/{id}", get(a))
    .route("/users/{name}", get(b));   // panic ทันทีตอน build router
```

```
thread 'main' panicked at ...:
Invalid route "/users/{name}": Insertion failed due to conflict with previously registered route: /users/{id}
```

นี่คือ panic ตอน **startup** ไม่ใช่ตอนมี request เข้ามา — ตรวจสอบให้ dynamic parameter ที่ตำแหน่งเดียวกันของ
path ทุกจุดใช้ชื่อเดียวกันเสมอ หรือรวมเป็น route เดียวแล้วแยก logic ในตัว handler

**5. ลืม `with_state` หรือเรียก `with_state` ผิดจุดในโครงสร้างที่มี `.nest()`/`.merge()`**

ถ้าเรียก `.with_state(state)` กับ router ย่อยก่อนที่จะ `.nest()`/`.merge()` เข้า router หลัก อาจเจอ compile
error ยาว ๆ เกี่ยวกับ type mismatch ของ `Router<S>` กับ `Router<()>` — วิธีที่ปลอดภัยที่สุดคือ **ประกาศ
router ย่อยทั้งหมดให้มี type parameter ของ state ตรงกัน (เช่น `Router<SharedState>` ทุกตัว) แล้วเรียก
`.with_state(...)` ครั้งเดียวที่ router ระดับบนสุดหลังรวมทุกอย่างเข้าด้วยกันแล้ว** ตามที่ capstone ในหัวข้อ
63.9 ทำ — Part 64 จะอธิบายเรื่อง state และ type parameter ของ `Router<S>` อย่างละเอียดกว่านี้มาก

**6. สับสนระหว่าง `Query<T>` rejection กับ `Path<T>` rejection**

ทั้งสองตอบ `400 Bad Request` เหมือนกัน แต่**รูปแบบข้อความต่างกัน**:

```
# Path<T> parse ไม่ผ่าน
Invalid URL: Cannot parse `abc` to a `u32`

# Query<T> deserialize ไม่ผ่าน
Failed to deserialize query string: page: invalid digit found in string
```

ถ้าเขียน test ที่เช็คข้อความ error ตรง ๆ (string matching) ต้องรู้ว่าข้อความมาจาก extractor ตัวไหน เพราะแต่ละ
ตัวใช้ deserializer คนละตัวกันภายใน ข้อความจึงไม่เหมือนกัน แม้จะเป็น "การ deserialize ล้มเหลว" แบบเดียวกันใน
เชิงแนวคิด

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เพิ่ม endpoint `GET /tickets/{id}/history` เข้าไปในตัวอย่าง Ticket CRUD ของหัวข้อ 63.3 ที่คืน
   `Json<Vec<String>>` รายการข้อความประวัติปลอม ๆ (hard-code ก็ได้ เช่น `vec!["created".to_string(),
   "updated".to_string()]`) — โจทย์นี้ฝึกแค่การเพิ่ม route ใหม่ที่มี `Path<u32>` ตัวเดียวเข้าไปใน
   `MethodRouter` ที่มีอยู่แล้ว
   - Hint: ต้องเพิ่ม `.route("/tickets/{id}/history", get(get_history))` เป็น route แยกจาก
     `/tickets/{id}` เดิม (คนละ path กัน แม้จะมี `{id}` ร่วมกัน) ไม่ใช่เพิ่ม method เข้าไปใน `MethodRouter`
     เดิม

2. **(กลาง)** เพิ่ม query parameter `done: Option<bool>` ให้ `list_tickets` ของหัวข้อ 63.3 เพื่อกรองว่าจะ
   แสดงแต่ ticket ที่ `done == true`, `done == false`, หรือทั้งหมด (ถ้าไม่ส่ง `done` มา) — ทดสอบด้วย curl ทั้ง
   สามกรณี (`?done=true`, `?done=false`, ไม่ส่ง query เลย) แล้วยืนยันว่าผลลัพธ์ตรงกับที่คาดไว้จริง
   - Hint: เพิ่ม field ใน struct query ที่มีอยู่เดิม เป็น `Option<bool>` แล้วใช้ `.filter(|t| match
     query.done { Some(d) => t.done == d, None => true })` แบบเดียวกับที่ capstone กรองด้วย `author`

3. **(ยาก)** เขียนโปรแกรมทดลอง (แยกไฟล์ใหม่) ที่จงใจสร้าง handler ที่มี extractor 17 ตัวขึ้นไป (ใช้
   `Path<u32>` สลับกับ extractor คนละชนิดเพื่อไม่ให้ชนกับ lint "ห้ามใช้ `Path<_>` ซ้ำ") แล้ว compile ดู error
   จริงทั้งแบบมีและไม่มี `#[axum::debug_handler]` — เขียนสรุปสั้น ๆ ว่า error สองแบบต่างกันอย่างไร และทำไม
   axum เลือก generate implementation ของ `Handler` ไว้แค่ถึง 16 arity ไม่ใช่มากกว่านั้นหรือไม่จำกัดเลย
   - Hint: ใช้เทคนิคเดียวกับที่บทนี้ใช้ทดสอบ arity limit — เขียน extractor เปล่า ๆ ที่ implement
     `FromRequestParts` เองผ่าน const generic (`struct Noop<const N: u8>`) เพื่อได้ type ที่ต่างกันจริง 17
     ตัวโดยไม่ต้องเขียน struct ซ้ำ 17 ชื่อ

4. **(ยาก/ประยุกต์ใช้งานจริง)** ขยาย Bookshelf API ของหัวข้อ 63.9 ให้มี resource ที่สองคือ `members`
   (สมาชิกห้องสมุด: `id`, `name`) และ endpoint `POST /api/v1/members/{member_id}/borrow/{book_id}` ที่
   บันทึกว่าสมาชิกคนไหนยืมหนังสือเล่มไหนอยู่ (เก็บใน `HashMap<u32, u32>` ที่ map จาก `book_id` ไปยัง
   `member_id` ก็พอสำหรับ placeholder แบบเดียวกับที่บทนี้ใช้) จัดโครงสร้างเป็นสอง module
   (`books_routes`/`members_routes`) แล้ว `.nest()` ทั้งคู่เข้า `/api/v1` — ทดสอบด้วย curl transcript เต็ม
   รูปแบบว่ายืมได้ตามที่ตั้งใจ และยืมหนังสือที่ถูกยืมไปแล้วจะได้ `409 Conflict` เหมือนกับที่ `borrow_book`
   เดิมทำ
   - Hint: path parameter สองตัวในหนึ่ง route (`{member_id}` กับ `{book_id}`) ต้องใช้ tuple หรือ struct
     ตามหัวข้อ 63.1 ไม่ใช่ `Path<T>` สองตัวแยกกัน — และต้องตัดสินใจว่า `ApiError` ที่เขียนไว้แล้วจะใช้ซ้ำกับ
     `members_routes` ได้อย่างไร (ทำเป็น type กลางที่ทั้งสองโมดูล `use super::ApiError` เหมือนที่ capstone
     ทำกับ `use super::*`)

## สรุป

บทนี้ขยาย Axum จากระดับ "Hello World" ของ Part 62 ไปสู่ระบบ routing และ handler ที่ใช้งานได้จริงในโปรเจกต์
ขนาดจริง — คุณได้เรียนการดึงข้อมูลจาก path ด้วย `Path<T>` ทั้งแบบเดี่ยว, tuple, และ struct พร้อมเห็น rejection
response จริงเมื่อ parse ไม่ผ่าน; การดึง query string ด้วย `Query<T>` และแพทเทิร์น `Option<T>` สำหรับค่าที่ไม่
บังคับ; การประกอบ route หลาย HTTP method ด้วย `MethodRouter` ให้เป็น CRUD resource ที่สมบูรณ์; การจัดระเบียบ
แอปที่ใหญ่ขึ้นด้วย `.nest()`/`.merge()` และโครงสร้างโมดูลต่อ resource; กฎการจับคู่ path ที่แม่นยำ (exact ชนะ
dynamic, dynamic ชนะ wildcard, ไม่มี trailing-slash redirect, route ที่ชนกัน panic ตอน startup); เหตุผลเชิง
ลึกจาก source code จริงว่าทำไม `Handler` trait รับ extractor ได้ 0-16 ตัวและทำไม body-consuming extractor
ต้องมาตัวสุดท้ายเท่านั้น; วิธีเขียน `impl IntoResponse` ให้ type ของแอปเอง; และการใช้
`#[axum::debug_handler]` แปลง compile error ที่งงงวยให้อ่านออกทันที ปิดท้ายด้วย capstone ระบบยืมหนังสือที่
ทำงานได้จริงทั้งระบบ ผ่านการทดสอบด้วย curl ครบทุก endpoint

Part ถัดไป (**Part 64: Axum: State Management และ Extractors**) จะกลับไปขยายเรื่อง `State<T>` ที่บทนี้ใช้
แค่แบบง่ายที่สุด (`Arc<Mutex<...>>`) ให้ลึกขึ้นมาก — การออกแบบ shared state ที่ปรับขนาดได้, การเขียน custom
extractor ของตัวเอง (`impl FromRequestParts`/`impl FromRequest`) แบบเดียวกับที่บทนี้แค่แสดงกลไกเบื้องหลังผ่าน
`Noop<N>`, และรูปแบบการจัดการ state ที่ใช้จริงในระบบ production เช่น connection pool ของ database

---

**Part ก่อนหน้า:** [แนะนำ Axum Framework](part-062-axum-intro.md) | **Part ถัดไป:** [Axum: State Management และ Extractors](part-064-axum-state-extractors.md)
