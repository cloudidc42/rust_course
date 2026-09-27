# Part 78: RESTful API Design Best Practices

> โมดูล: การพัฒนาเว็บแอปพลิเคชัน (Web Development) | ระดับ: มืออาชีพ | เวลาโดยประมาณ: 260 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- ใช้ **กรอบการตัดสินใจ (decision framework)** ที่ชัดเจนในการตั้งชื่อ resource และเลือกว่าจะ **nest URL**
  (`/shelves/{id}/books`) หรือ **flatten ด้วย query filter** (`/books?shelf_id={id}`) — ไม่ใช่แค่ทำตาม
  ความเคยชิน แต่รู้เหตุผลเชิงความหมายว่าทำไมความสัมพันธ์แบบหนึ่งควร nest และอีกแบบไม่ควร
- เปรียบเทียบ **versioning strategy** สามแบบ (URL-based, header-based, query-param-based) ได้อย่างเป็นระบบ
  ทั้งข้อดี/ข้อเสียเชิงปฏิบัติ (debuggability, cache-friendliness, ความชัดเจนต่อผู้บริโภค API) และอธิบายได้ว่า
  ทำไมหลักสูตรนี้เลือกใช้ URL versioning (`/api/v1/...`) มาตั้งแต่ Part 61
- ออกแบบ **response envelope ที่สม่ำเสมอทั้งระบบ** ต่อยอดจาก `AppError`/`IntoResponse` ของ Part 66 ให้กลาย
  เป็น convention เดียวที่ครอบคลุมทั้ง success response (เดี่ยว/list), error response, และ pagination
  metadata — พร้อมเหตุผลว่าทำไม "wrapped" (`{"data": ...}`) มักชนะ "bare" (`[...]`) ในระบบที่ต้องรองรับ
  client หลายเวอร์ชันไปอีกหลายปี
- Implement **`Link` header pagination** ตาม RFC 8288 คู่กับ keyset pagination ที่ Part 71 สอนไว้, เขียน
  **sort-param mini-DSL** (`?sort=-created_at,title`) และ **sparse fieldsets** (`?fields=id,title`) ด้วย
  `Query<T>` extractor จาก Part 63 — ทุกตัวรันจริงผ่าน Axum server และพิสูจน์ด้วย `curl` จริง
- Implement **`Idempotency-Key`** สำหรับ `POST` ที่สร้าง resource แบบใช้งานได้จริง (ไม่ใช่แค่ simulation
  ด้วย `HashMap` แบบ Part 61) — เก็บทั้ง response ที่เคยตอบไปแล้วและ hash ของ body เพื่อตรวจจับการใช้ key ซ้ำ
  กับ request ที่ไม่เหมือนกัน (ตอบ `422`) พร้อมรู้ว่า production จริงต้องย้ายไปเก็บใน Redis (Part 83)
- ใส่ **`_links` แบบ HATEOAS ที่ใช้งานได้จริง** อย่างเลือกสรร (ไม่ใช่ full hypermedia navigation ตามที่ Part
  61 อธิบายว่า API จริงส่วนใหญ่ไม่ทำ) และออกแบบ **rate limiting contract** (`429`, `Retry-After`,
  `X-RateLimit-*`) ที่ทีม frontend เอาไปใช้ได้จริงแม้ backend ยังไม่มี distributed rate limiter (Part 83)
- แยกแยะ **breaking change** กับ **safe additive change** ได้อย่างแม่นยำจาก checklist ที่เป็นรูปธรรม และรู้จัก
  ใช้ `Deprecation`/`Sunset` header (RFC 8594) เพื่อวางแผน migration ให้ผู้บริโภค API โดยไม่ทำลาย client เดิม

## ความรู้ที่ต้องมีมาก่อน

- **Part 61 (HTTP Fundamentals และ REST API Concepts)**: บทนี้คือ "ภาคต่อระดับมืออาชีพ" ของ Part 61 ตรง ๆ —
  Part 61 ปูพื้น HTTP method/status/header, แนะนำ HATEOAS และ Richardson Maturity Model ในระดับ**แนวคิด**,
  และสาธิต Idempotency-Key ด้วย `HashMap` ธรรมดาไม่มี HTTP จริงเกี่ยวข้อง — บทนี้จะหยิบทุกแนวคิดนั้นมา
  **implement จริงด้วย Axum** และเพิ่มมิติที่ Part 61 ยังไม่ได้พูดถึงเลย (versioning, response envelope,
  Link header, rate limiting contract, breaking changes) ถ้าจำเนื้อหา Part 61 ไม่ชัด ควรกลับไปทวนก่อน
  เพราะบทนี้จะอ้างอิงกลับไปตลอดโดยไม่อธิบายพื้นฐาน HTTP method/status ซ้ำ
- **Part 63 (Axum: Routing และ Handlers)**: `Query<T>` extractor ที่หัวข้อ 78.6 ใช้ parse `?sort=...&fields=...`
  คือตัวเดียวกับที่ Part 63 สอนไว้ทุกประการ — ต้องเข้าใจว่าทำไม field ควรเป็น `Option<T>` และ Axum จัดการ
  deserialize error ของ query string อย่างไร
- **Part 64 (Axum: State Management และ Extractors)**: ตัวอย่าง Idempotency-Key และ rate limiting ในบทนี้
  ใช้ `State<AppState>` เก็บ `Arc<Mutex<HashMap<...>>>` แบบเดียวกับที่ Part 64 สอนเรื่อง shared mutable state
  ข้าม request — ถ้าลืมว่าทำไมต้องใช้ `Arc`/`Mutex` คู่กัน ควรทวนก่อน
- **Part 65 (Axum: Middleware)**: rate limiting ในทางปฏิบัติมักทำเป็น middleware ที่ครอบทุก route ไม่ใช่โค้ด
  ซ้ำในทุก handler — บทนี้เขียนแบบ handler-level เพื่อความชัดเจนในการสอน แต่จะอธิบายว่าทำไม production ควร
  ย้ายไปเป็น `tower::Layer` ตามที่ Part 65 สอนไว้
- **Part 66 (Axum: Error Handling แบบมืออาชีพ)**: หัวข้อ 78.4 ของบทนี้คือการ**ขยาย** `AppError`/
  `impl IntoResponse for AppError` ของ Part 66 ให้กลายเป็น response envelope แบบเต็มรูปแบบ — ต้องเข้าใจ
  โครงสร้าง `AppError` เดิมก่อน (โดยเฉพาะว่าทำไมมันมี error code คงที่ ไม่ใช่ข้อความอิสระ)
- **Part 71 (SQLx: Queries, Migrations, Connection Pooling)**: หัวข้อ 78.5 อ้างอิงตรงถึง**keyset/cursor-based
  pagination** ที่ Part 71 สอนไว้ (ด้วย `WHERE id > $cursor ORDER BY id LIMIT $n`) — บทนี้ไม่สอน SQL ซ้ำ
  แต่จะสอนว่า**ที่ระดับ HTTP** ควร expose cursor นั้นออกมาให้ client อย่างไรผ่าน `Link` header
- **Part 74-75 (JWT Authentication, Session/OAuth2)**: ไม่ใช่ความรู้บังคับสำหรับโค้ดในบทนี้โดยตรง (endpoint
  ตัวอย่างไม่มี auth) แต่แนวคิดเรื่อง "convention ที่สม่ำเสมอทั้งระบบ" ที่บทนี้สอนใช้กับ auth error response
  ด้วยเช่นกัน (เช่น `401`/`403` ก็ควรอยู่ใน envelope เดียวกันกับ error อื่น ๆ)
- บทนี้**ไม่แนะนำ crate ใหม่**และไม่สอน Rust syntax ใหม่ — เป็นบทสังเคราะห์ (synthesis chapter) ที่ใช้เครื่องมือ
  ทั้งหมดที่มีอยู่แล้ว (Axum, `serde`, `Query<T>`, `State<T>`) มาออกแบบ **convention ระดับ API** ที่ทำให้ API
  สม่ำเสมอ วิวัฒน์ได้โดยไม่พังของเดิม (evolvable) และใช้งานสบายสำหรับนักพัฒนาคนอื่นที่ต้องเรียก API ของเรา —
  ลักษณะบทคล้ายกับ Part 69 (บทเปรียบเทียบ framework) มากกว่าบทที่สอน syntax ใหม่ทีละก้อน

## เนื้อหา

### 78.1 จาก "ทำงานได้" สู่ "ออกแบบดี": ทำไมบทนี้ไม่มี Syntax ใหม่แม้แต่ตัวเดียว

ตลอด Part 62-77 คุณสร้าง API ที่มี routing, state, middleware, error handling, การเชื่อมฐานข้อมูล,
authentication, RBAC, และ WebSocket มาแล้วครบ — ในทางเทคนิค **คุณมีทุกเครื่องมือที่จำเป็นในการสร้าง API ที่
"ทำงานได้" แล้ว** สิ่งที่ยังไม่มีคือ**วินัยในการออกแบบ** ที่ทำให้ API ของคุณ:

1. **คาดเดาได้ (predictable)** — นักพัฒนาที่ไม่เคยเห็น API ของคุณมาก่อนควรเดาชื่อ endpoint/รูปแบบ response
   ได้ถูกโดยดูจากตัวอย่างไม่กี่ตัว ไม่ต้องเปิดเอกสารทุกครั้ง
2. **วิวัฒน์ได้โดยไม่พังของเดิม (evolvable)** — เพิ่ม field ใหม่, เปลี่ยน pagination, เพิ่ม endpoint ใหม่ได้
   โดย client เวอร์ชันเก่า**ยังใช้งานต่อได้ปกติ** ไม่ต้องบีบให้ทุกคน upgrade พร้อมกัน
3. **ใช้งานสบายสำหรับคนอื่น (developer experience)** — error message ชัดเจน, pagination ใช้ pattern เดียว
   ทั้งระบบ, ไม่ต้องเดาว่า endpoint นี้จะตอบ `200` หรือ `204` หรือ list เปล่าจะเป็น `[]` หรือ `null`

สามข้อนี้**ไม่มีข้อไหนแก้ด้วยการเขียนโค้ด Rust ที่ "ถูกต้องกว่า"** — compiler ของ Rust ไม่ช่วยคุณตัดสินว่า
`/books` ควร nest เป็น `/shelves/{id}/books` หรือไม่ ไม่ช่วยตัดสินว่า response ควร wrap ด้วย `{"data": ...}`
หรือไม่ ไม่ช่วยตัดสินว่า field ใหม่ที่คุณเพิ่มเข้าไปจะทำ client เก่าพังหรือไม่ — สิ่งเหล่านี้คือ **การตัดสินใจ
เชิงออกแบบ (design decision)** ที่ต้องอาศัยเหตุผลเชิง trade-off ไม่ใช่กฎภาษา และนี่คือเหตุผลที่บทนี้มีโค้ด
Rust **น้อยกว่า** บทอื่น ๆ ในโมดูลนี้อย่างเห็นได้ชัด (คล้ายกับ Part 69 ที่เปรียบเทียบ framework โดยไม่สอน
syntax ใหม่) — โค้ดที่มีจะเป็น**การพิสูจน์ว่า convention ที่เลือกไว้ implement ได้จริงและทำงานถูกต้อง**
มากกว่าการสอนกลไกภาษาใหม่

หลักคิดกลางที่ร้อยทุกหัวข้อของบทนี้เข้าด้วยกันคือคำถามเดียว: **"ถ้าคุณเป็นนักพัฒนาที่เพิ่งได้ API นี้มาใช้เป็น
ครั้งแรก โดยไม่มีใครอธิบายอะไรให้เลย นอกจาก endpoint ตัวอย่างสองสามตัว — คุณจะเดา endpoint/response ที่เหลือ
ถูกไหม?"** ถ้าคำตอบคือ "ถูก" แปลว่า API นั้นสม่ำเสมอเพียงพอ ถ้าคำตอบคือ "ไม่แน่ใจ" แปลว่ายังมีจุดที่ convention
ขาดความชัดเจน — ทุกหัวข้อถัดไปในบทนี้คือคำตอบสำหรับจุดที่ API มักขาดความชัดเจนบ่อยที่สุดในทางปฏิบัติ

### 78.2 การตั้งชื่อ Resource: Noun พหูพจน์ และกรอบการตัดสินใจเรื่อง Nesting

#### กฎพื้นฐาน: Resource คือคำนาม ไม่ใช่คำกริยา และเป็นพหูพจน์เสมอสำหรับ Collection

Part 61 หัวข้อ 61.7 อธิบายไว้แล้วว่า REST คิดแบบ "resource + method" ไม่ใช่ "การเรียกฟังก์ชัน" — ข้อสรุป
เชิงปฏิบัติที่ตรงไปตรงมาที่สุดของหลักการนี้คือ: **URL ของ collection ควรเป็นคำนามพหูพจน์เสมอ ไม่มีคำกริยา
ปนอยู่เลย**

| แบบที่ไม่ควรทำ | แบบที่ควรทำ | เหตุผล |
|---|---|---|
| `GET /getBooks` | `GET /books` | `GET` (method) บอกอยู่แล้วว่า "อ่าน" — คำว่า `get` ใน path ซ้ำซ้อนกับสิ่งที่ HTTP method บอกไว้แล้ว |
| `POST /createBook` | `POST /books` | เหตุผลเดียวกัน — `POST` บอกอยู่แล้วว่า "สร้าง" |
| `GET /book` (เอกพจน์) | `GET /books` (พหูพจน์) | `/books` คือ collection ทั้งชุด — endpoint เดียวกันนี้ (ไม่มี id ต่อท้าย) ควรสื่อว่า "รวมทุกเล่ม" ไม่ใช่ "เล่มเดียว" |
| `GET /book/4821` | `GET /books/4821` | แม้จะเป็นเล่มเดียว แต่ URL prefix ควรคงที่เป็นพหูพจน์เสมอไม่ว่าจะเข้าถึง collection หรือ item เดี่ยว — ไม่มีเหตุผลที่ดีที่จะสลับเอกพจน์/พหูพจน์ตามจำนวนผลลัพธ์ |
| `POST /books/deleteAll` | `DELETE /books?older_than=2020-01-01` | ถ้าต้องการ "ลบหลายรายการตามเงื่อนไข" ให้ใช้ `DELETE` กับ query filter แทนการสร้าง endpoint คำกริยาใหม่ |

เหตุผลเชิงลึกที่อยู่เบื้องหลังกฎ "ห้ามมีคำกริยาใน path": **คำกริยาใน path คือสัญญาณว่า API กำลังคิดแบบ RPC
(หัวข้อ 61.7) ปนเข้ามาใน URL ที่ควรจะเป็น resource-based ล้วน ๆ** — เมื่อคุณเขียน `POST /createBook` คุณกำลัง
บอกว่า "path นี้คือชื่อฟังก์ชัน" ซึ่งขัดกับปรัชญาที่ว่า "path คือ**ที่อยู่ของข้อมูล**, method คือ**การกระทำ**"
— ถ้าปล่อยให้คำกริยาปนเข้ามาได้ในบางที่ ทีมงานจะเริ่มไม่แน่ใจว่าเมื่อไรควรใช้ noun ล้วน ๆ เมื่อไรควรใส่กริยา
และความไม่แน่ใจนี้คือรากของ API ที่ไม่สม่ำเสมอ

#### กรณีที่ดูเหมือนต้องมีคำกริยาแต่จริง ๆ ไม่ต้อง: Action ที่ไม่ใช่ CRUD ตรง ๆ

คำถามที่พบบ่อยที่สุดคือ: "แล้วถ้าต้อง 'ยกเลิกการจอง' ล่ะ? มันไม่ใช่ CRUD ตรง ๆ (ไม่ใช่ create/read/update/
delete แบบตรงไปตรงมา) จะเลี่ยงคำกริยายังไง?" — คำตอบคือสองแนวทางหลักที่ API จริงในอุตสาหกรรมใช้กัน:

**แนวทางที่ 1 (แนะนำเมื่อ action มีความหมายเป็น state transition ที่ชัดเจน)**: มองว่า "การยกเลิก" คือการ
**เปลี่ยนสถานะของ resource เดิม** ไม่ใช่การสร้าง resource ใหม่ — ใช้ `PATCH` กับ path เดิม:

```text
PATCH /api/v1/bookings/4821
Content-Type: application/json

{"status": "cancelled"}
```

**แนวทางที่ 2 (แนะนำเมื่อ action มี side effect ที่ซับซ้อนกว่าการเปลี่ยน field เดียว)**: มองว่า "การยกเลิก"
เป็น **sub-resource ที่เป็นคำนาม** ของการกระทำนั้น (คิดว่า "cancellation" คือของที่ถูกสร้างขึ้น ไม่ใช่คำกริยา
ที่ถูกเรียก) — ใช้ `POST` ไปที่ path ย่อยที่เป็นคำนาม:

```text
POST /api/v1/bookings/4821/cancel
```

สังเกตว่า `cancel` ในแบบที่ 2 **ยังดูเหมือนคำกริยา** แต่ในทางปฏิบัติ API จริงจำนวนมาก (Stripe: `POST
/payment_intents/{id}/cancel`, GitHub: `POST /repos/{owner}/{repo}/pulls/{number}/merge`) ก็ใช้รูปแบบนี้
เพราะมันสื่อความหมายชัดกว่าการพยายามยัดทุกอย่างให้เป็น `PATCH` เมื่อ action นั้นมี business logic ซับซ้อนกว่า
"เปลี่ยนค่า field" ตรง ๆ (เช่น การยกเลิกอาจต้อง refund เงิน, ส่งอีเมล, ปลดล็อกที่นั่งให้คนอื่นจอง — logic
เยอะกว่าการ `UPDATE status = 'cancelled'` ใน database) — **หลักที่ยึดได้คือ**: ถ้า action นั้นเทียบเท่ากับ
"เปลี่ยนค่า field เดียวตรงไปตรงมา" ให้ใช้ `PATCH`; ถ้า action นั้นมี business logic/side effect ซับซ้อนพอที่
สมควรมีชื่อของตัวเองที่ทีมอื่นจำได้ทันที ให้ใช้ `POST` กับ sub-path ที่เป็นคำนามของผลลัพธ์การกระทำนั้น (ไม่ใช่
คำกริยากระทำตรง ๆ แม้จะเขียนเหมือนกริยาก็ตาม) — บทนี้จะใช้แนวทางที่ 2 กับตัวอย่าง `POST
/api/v1/bookings/{id}/cancel` ในหัวข้อ 78.8 เพราะการยกเลิกการจองมี business rule ("ยกเลิกได้เฉพาะตอนที่ยัง
`confirmed` อยู่") ที่สมควรมี endpoint ของตัวเอง

#### กรอบการตัดสินใจ: เมื่อไรควร Nest URL เมื่อไรควร Flatten + Filter

นี่คือคำถามที่มักสร้างความสับสนมากที่สุดในทีมจริง: เมื่อมีความสัมพันธ์ระหว่าง resource สองตัว ควรเขียนเป็น
`/parents/{id}/children` (nested) หรือ `/children?parent_id={id}` (flat + filter) ดี? — คำตอบไม่ใช่ "อย่างใด
อย่างหนึ่งถูกเสมอ" แต่ขึ้นกับ**ธรรมชาติของความสัมพันธ์**นั้น ตรวจสอบด้วย 3 คำถามนี้:

1. **child มีตัวตนที่มีความหมายได้โดยไม่ต้องอ้างอิง parent หรือไม่?** ถ้า "ไม่" (child ไม่มีความหมายอะไรเลย
   นอกบริบทของ parent) → เอียงไปทาง **nest**
2. **การลบ parent ควร cascade ลบ child ไปด้วยหรือไม่ (โดยธรรมชาติทางความหมาย ไม่ใช่แค่ทาง technical
   constraint)?** ถ้า "ใช่" → เอียงไปทาง **nest**
3. **child ย้ายจาก parent หนึ่งไปอีก parent หนึ่งได้หรือไม่ (reassignment) หรือมี query pattern ที่ต้องกรอง
   ข้าม parent หลายตัวพร้อมกัน?** ถ้า "ได้" → เอียงไปทาง **flat + filter**

ใช้กรอบนี้กับสองตัวอย่างจริงจากโดเมนที่ใช้ตลอดหลักสูตรนี้:

**ตัวอย่างที่ 1 — Nest: `/events/{event_id}/tickets`** (Part 61 หัวข้อ 61.7 ใช้ตัวอย่างนี้ไว้แล้ว บทนี้อธิบาย
เหตุผลเชิงลึกเพิ่ม): ตั๋วงานอีเวนต์ **ไม่มีความหมายอะไรเลยถ้าไม่รู้ว่าเป็นตั๋วของ event ไหน** (คำถาม 1 = ไม่มี
ตัวตนนอกบริบท parent) ถ้า event ถูกยกเลิก/ลบไปทั้งหมด ตั๋วทุกใบของ event นั้นก็ไม่มีความหมายอีกต่อไปแล้ว
โดยธรรมชาติ (คำถาม 2 = cascade ตามธรรมชาติ) และตั๋วใบหนึ่ง**ไม่มีทางย้ายไปเป็นของ event อื่น**ได้เลย (คำถาม 3
= ไม่มี reassignment) — สามคำถามชี้ไปทาง **nest ทั้งหมด**: `GET /api/v1/events/99/tickets` เหมาะสมกว่า
`GET /api/v1/tickets?event_id=99` เพราะ URL เองก็สื่อความหมาย "ตั๋วเหล่านี้เป็นของ event 99" ได้ตรง ๆ โดยไม่
ต้องอ่าน query string เพิ่ม

**ตัวอย่างที่ 2 — Flat + Filter: `/books?shelf_id={id}`** (ระบบห้องสมุดที่ใช้ตลอด Part 57-73): หนังสือเล่ม
หนึ่ง**มีตัวตนและความหมายที่ชัดเจนแม้ไม่รู้ว่าอยู่ชั้นไหน** — `GET /api/v1/books/4821` ที่ client บุ๊คมาร์ก
ไว้ควรใช้งานได้เสมอไม่ว่าหนังสือเล่มนั้นจะถูกย้ายไปวางชั้นไหนก็ตาม (คำถาม 1 = มีตัวตนนอกบริบท parent) การลบ
ชั้นวางหนังสือ (เช่น ปิดปรับปรุง) **ไม่ควร**ลบหนังสือทั้งชั้นไปด้วย (คำถาม 2 = ไม่ cascade) และหนังสือ**ย้าย
จากชั้นหนึ่งไปอีกชั้นได้ตามปกติ** (บรรณารักษ์จัดชั้นใหม่) แถมบางครั้งยังต้องกรองข้ามหลายชั้นพร้อมกัน เช่น
`GET /api/v1/books?shelf_id=12&status=available` (คำถาม 3 = reassignment ได้ + ต้อง query ข้าม parent) —
สามคำถามชี้ไปทาง **flat + filter ทั้งหมด**: `GET /api/v1/books?shelf_id=12` เหมาะสมกว่า
`GET /api/v1/shelves/12/books` เพราะ URL ของหนังสือควรคงที่ (`/books/4821`) ไม่ว่าจะถูกจัดวางใหม่กี่ครั้งก็ตาม
ถ้าเลือก nest แบบผิด (`/shelves/{shelf_id}/books/{book_id}`) จะเกิดปัญหาจริงตอนหนังสือถูกย้ายชั้น: URL เดิมที่
client เคยบุ๊คมาร์กไว้ (`/shelves/12/books/4821`) จะใช้ไม่ได้อีกต่อไปทันทีที่ย้ายไปชั้น 15 — ทั้งที่หนังสือเล่ม
นั้นยังมีอยู่ ไม่ได้ถูกลบไปไหน

#### กับดัก Nesting ที่ลึกเกินไป

แม้บาง relationship เข้าเงื่อนไข "ควร nest" จริง ๆ ก็ยังมีขีดจำกัดว่า **ลึกได้แค่ไหน**: หลักปฏิบัติที่ยึดกัน
ทั่วไปคือ **ไม่ nest เกิน 2 ระดับ** ตัวอย่างที่แสดงว่าทำไมการ nest ลึกเกินไปกลายเป็นปัญหา:

```text
# ลึกเกินไป — อ่านยาก, เขียน client code ยาก, และเปลี่ยน URL scheme ยากมากในอนาคต
GET /api/v1/events/99/tickets/4821/attendees/17/checkins/3

# ดีกว่า — flatten ระดับที่ลึกเกินไป ใช้ id ตรง ๆ (ตัวตนของ resource ปลายสุดไม่ผันแปรตาม parent เส้นทางไหน)
GET /api/v1/checkins/3
```

เหตุผล: เมื่อ URL ลึกขนาดนี้ **ทุกระดับกลายเป็นข้อมูลที่ client ต้องรู้และส่งมาถูกทุกตัวเพื่อเข้าถึง resource
ปลายสุดตัวเดียว** ทั้งที่ในทางปฏิบัติ `checkin` มักมี `id` ของตัวเองที่ unique พออยู่แล้ว (ไม่ต้องพึ่ง
`event_id`/`ticket_id`/`attendee_id` มาช่วยระบุตัวตนเลย) — กฎง่าย ๆ ที่ใช้ตัดสินคือ: **nest ได้ไม่เกิน 1 ชั้น
จาก resource ที่เป็น "เจ้าของโดยตรงแท้จริง" เท่านั้น** ถ้าความสัมพันธ์ต้องลึกกว่านั้น ให้ flatten ระดับที่เหลือ
ด้วย id ตรง ๆ แล้วใช้ query filter (`?event_id=99&ticket_id=4821`) เสริมถ้าจำเป็นต้องกรองจริง ๆ

### 78.3 Versioning Strategies: URL vs Header vs Query Param

Part 61 แนะนำ `/api/v1/...` ไว้ตั้งแต่หัวข้อแรกโดยไม่อธิบายว่าทำไมเลือกแบบนี้ — บทนี้จะอธิบายให้ครบว่ามี
ทางเลือกอะไรบ้าง และทำไมหลักสูตรนี้เลือก URL versioning เป็น default

#### แนวทางที่ 1: URL-Based Versioning

```text
GET /api/v1/books/4821
GET /api/v2/books/4821
```

เวอร์ชันของ API ถูกฝังอยู่ใน**ตัว path** ตรง ๆ — เป็นแนวทางที่หลักสูตรนี้ใช้มาตั้งแต่ Part 61

**ข้อดี:**

- **มองเห็นได้ทันที (visible)** — เปิด browser วาง URL ก็รู้ทันทีว่าเรียกเวอร์ชันไหน ไม่ต้องเปิด dev tools
  ดู header
- **debug ง่ายที่สุด** — copy URL ไปวางใน `curl`/Postman/browser ได้ทันทีโดยไม่ต้องจำ header เพิ่ม, log ของ
  server (access log ทั่วไปที่บันทึกแค่ request line) ก็เห็นเวอร์ชันที่ถูกเรียกโดยอัตโนมัติอยู่แล้ว
- **cache ง่าย** — HTTP cache (browser, CDN, reverse proxy) ทำงานบนพื้นฐาน URL เป็น cache key อยู่แล้ว
  เวอร์ชันที่ต่างกันจึงถูก cache แยกกันโดยอัตโนมัติโดยไม่ต้องตั้งค่าอะไรเพิ่ม
- **routing ง่ายที่สุดในโค้ด** — Axum `Router::new().nest("/api/v1", v1_routes).nest("/api/v2", v2_routes)`
  ตรงไปตรงมา ไม่ต้องเขียน middleware แยก header มา parse

**ข้อเสีย (ตามที่นักออกแบบ REST เคร่งครัดวิจารณ์กัน):**

- **"ปนเปื้อน" URL semantic ในทาง REST เคร่งครัด** — ในทางทฤษฎี `/api/v1/books/4821` และ `/api/v2/books/4821`
  ควรจะเป็น**คนละ resource** ในสายตาของ HTTP (เพราะ URL คือ identity ของ resource) ทั้งที่ในทางปฏิบัติมันคือ
  "หนังสือเล่มเดียวกัน มองผ่านสองรูปแบบข้อมูล" — นี่คือความขัดแย้งทางปรัชญาที่นัก REST เคร่งครัดชี้ว่า URL
  versioning "ไม่ RESTful เต็มรูปแบบ"
- **เปลี่ยนเวอร์ชันทั้ง URL ทุกจุด** — client ที่ต้องการ upgrade ต้องแก้ URL **ทุกจุดที่เรียก** ไม่ใช่แค่
  เปลี่ยนค่าใน header เดียว

#### แนวทางที่ 2: Header-Based Versioning (Content Negotiation)

```text
GET /api/books/4821
Accept: application/vnd.libraryapi.v2+json
```

เวอร์ชันถูกฝังอยู่ใน**ค่า `Accept` header** โดยใช้ **custom media type** (`vnd.` คือ prefix มาตรฐานสำหรับ
"vendor-specific media type" ตาม RFC 6838) — path เดียวกันเสมอ (`/api/books/4821`) ไม่ว่าจะขอเวอร์ชันไหน

**ข้อดี:**

- **"ถูกต้องตามหลัก REST" มากกว่าในทางทฤษฎี** — `/api/books/4821` ยังคง**เป็น resource เดียวกัน**เสมอ (ตัวตน
  ไม่เปลี่ยนไปตามเวอร์ชัน) สิ่งที่เปลี่ยนคือ**รูปแบบข้อมูล (representation)** ที่ขอ ซึ่งตรงกับความหมายดั้งเดิม
  ของคำว่า "**Re**presentational **S**tate Transfer" ตรง ๆ — content negotiation ผ่าน `Accept` คือกลไกที่
  HTTP ออกแบบมาให้ทำสิ่งนี้อยู่แล้ว (Part 61 หัวข้อ 61.4 อธิบาย `Accept` ไว้แล้วสำหรับเลือกรูปแบบ เช่น JSON
  vs XML — versioning ก็เป็นการประยุกต์ใช้กลไกเดียวกัน)
- **URL คงที่ตลอดไป** — ไม่ต้องเปลี่ยน URL ทุกจุดตอน upgrade เวอร์ชัน (แก้แค่ header เดียว)

**ข้อเสีย (เหตุผลที่หลักสูตรนี้ไม่เลือกแนวทางนี้เป็น default):**

- **debug ยากกว่ามาก** — เปิด URL ใน browser ตรง ๆ ไม่ได้บอกเวอร์ชันเลย ต้องเปิด dev tools ดู request header
  ทุกครั้ง; แชร์ URL ให้เพื่อนร่วมทีมโดยไม่แนบ header ประกอบ อีกฝ่ายจะได้เวอร์ชัน default โดยไม่รู้ตัว
- **เอกสารและตัวอย่างซับซ้อนขึ้น** — ทุกตัวอย่าง `curl` ในเอกสารต้องมี `-H "Accept: ..."` แนบเสมอ ลืมแนบครั้ง
  เดียวก็ได้เวอร์ชันผิดโดยไม่มี error บอกชัดเจน (บาง implementation คืน default version เงียบ ๆ)
- **cache/proxy บางตัวไม่ได้ตั้งค่าให้แยก cache ตาม `Accept` โดย default** — HTTP cache แยก cache key ตาม URL
  เป็นหลัก การแยกตาม header ต้องอาศัย `Vary: Accept` response header ซึ่งเป็นรายละเอียดที่ทีมมักลืมตั้งจนเกิด
  บั๊ก "cache เวอร์ชันผิดกลับมาให้ client อื่น"

#### แนวทางที่ 3: Query-Param-Based Versioning

```text
GET /api/books/4821?version=2
GET /api/books/4821?api-version=2026-09-01
```

เวอร์ชันฝังอยู่ใน query parameter — บางบริษัทใหญ่ (เช่น Stripe) ใช้รูปแบบ**วันที่**แทนเลขเวอร์ชัน
(`api-version=2026-09-01`) เพื่อสื่อว่า "พฤติกรรม API ณ วันที่นี้" ไม่ใช่ "รุ่นที่ N" — วิธีนี้มักเก็บเป็น
header แทน query param จริง ๆ ในทางปฏิบัติของ Stripe แต่หลักการ (ผูกเวอร์ชันกับวันที่) ใช้ร่วมกับทั้งสอง
mechanism ได้

**ข้อดี:** อยู่กึ่งกลางระหว่างสองแนวทางข้างบน — เห็นได้จาก URL เหมือน URL versioning (debug ง่ายกว่า header
version) แต่ path หลักยังคงที่เหมือน header versioning (ไม่ต้องเปลี่ยน path ทุกจุด แค่เปลี่ยนค่า query param)

**ข้อเสีย:** **เสี่ยงถูกมองว่าเป็น "ตัวกรอง" ธรรมดา** เพราะ query param ตามธรรมเนียม REST (หัวข้อ 61.8) มีไว้
สำหรับ filter/sort/pagination ไม่ใช่สำหรับกำหนด "ตัวตนพื้นฐาน" ของ response — การ mix เวอร์ชันเข้ากับ query
param อื่น ๆ (`?version=2&status=open&sort=title`) ทำให้แยกยากว่า param ไหนคือ "ตัวกรอง" param ไหนคือ
"ตัวกำหนดโครงสร้าง response" และมี**ความเสี่ยงเรื่อง cache**คล้ายกับ header versioning (URL ที่ query param
ต่างกันมักถูก cache แยกกันโดย default ในหลาย cache implementation ซึ่งจริง ๆ ก็คือข้อดีในกรณีนี้ แต่ proxy
บางตัวถูกตั้งค่าให้ตัด query param บางตัวออกก่อน cache เพื่อเพิ่ม cache hit rate — ถ้าตัด `version` ออกไปด้วย
โดยไม่ตั้งใจ จะเกิดบั๊กแบบเดียวกับ header versioning ทันที)

#### ตารางสรุปและคำแนะนำ

| มิติ | URL-based | Header-based | Query-param-based |
|---|---|---|---|
| Debug/แชร์ URL ตรงไปตรงมา | ดีที่สุด | แย่ที่สุด (ต้องแนบ header เสมอ) | ดี (เห็นใน URL) |
| ถูกต้องตามหลัก REST เคร่งครัด | อ่อนที่สุด | แข็งแรงที่สุด | อ่อน (คล้าย URL) |
| ต้องแก้ URL ทุกจุดตอน upgrade | ต้องแก้ | ไม่ต้องแก้ (แก้ header จุดเดียว) | ไม่ต้องแก้ path (แก้ query param) |
| Cache friendliness | ดีที่สุด (URL คือ cache key เดิม) | ต้องตั้ง `Vary` เอง มักลืม | ดี แต่เสี่ยง proxy ตัด query param ทิ้ง |
| Routing ในโค้ด Axum | ง่ายที่สุด (`nest()`) | ต้องเขียน middleware parse `Accept` เอง | ต้องเขียน extractor/middleware parse query param เอง |

**คำแนะนำเชิงปฏิบัติของบทนี้**: **URL versioning คือ default ที่เหมาะกับทีมส่วนใหญ่** เพราะข้อดีเรื่อง
debuggability, ความชัดเจนต่อผู้บริโภค API ภายนอก, และความง่ายในการ implement/cache ชนะข้อเสียเชิงปรัชญาที่
ไม่ได้ส่งผลกระทบเชิงปฏิบัติมากนักสำหรับ API ส่วนใหญ่ — ทีมที่ควรพิจารณา header versioning จริง ๆ คือทีมที่ (1)
มี API ที่เปลี่ยนแปลงบ่อยมากและต้องการให้ client เปลี่ยนเวอร์ชันแบบ **granular ต่อ resource** (ระหว่างที่
`/books` อาจอยู่ v3 แต่ `/authors` อาจยังอยู่ v1 ได้พร้อมกัน — ยากมากที่จะทำแบบนี้ด้วย URL versioning เพราะ
path ทั้ง prefix มักผูกกับเวอร์ชันเดียว) หรือ (2) มีทีม client ภายในองค์กรเดียวกันที่คุ้นเคยกับ content
negotiation อยู่แล้วและมี tooling รองรับพร้อม — สำหรับ public API ทั่วไปที่ต้องให้นักพัฒนาภายนอกใช้งานง่ายและ
debug เองได้โดยไม่ต้องอ่านเอกสารละเอียด **URL versioning ยังคงเป็นตัวเลือกที่ปลอดภัยที่สุด** และนี่คือเหตุผล
ที่หลักสูตรนี้ใช้ `/api/v1/...` มาตั้งแต่ Part 61 และจะใช้ต่อไปตลอดทุกตัวอย่างในบทนี้

### 78.4 Response Envelope: Bare vs Wrapped — ขยาย `AppError` จาก Part 66 ให้เป็น Convention เต็มรูปแบบ

#### คำถามที่ต้องตอบก่อนเขียน handler ตัวแรกของโปรเจกต์: Response ควร "เปลือย" หรือ "ห่อ"?

**Bare response** คือ response ที่ส่งข้อมูลจริงกลับไปตรง ๆ ไม่มี wrapper:

```json
[
  {"id": 1, "title": "หนังสือเล่มที่ 01"},
  {"id": 2, "title": "หนังสือเล่มที่ 02"}
]
```

**Wrapped response** คือ response ที่ห่อข้อมูลจริงไว้ใน field ชื่อ `data` เสมอ พร้อมพื้นที่สำหรับ metadata
เพิ่มเติม:

```json
{
  "data": [
    {"id": 1, "title": "หนังสือเล่มที่ 01"},
    {"id": 2, "title": "หนังสือเล่มที่ 02"}
  ],
  "meta": {
    "request_id": "req-abc-123",
    "pagination": {"limit": 20, "next_cursor": 2, "has_next": true}
  }
}
```

#### ข้อดี-ข้อเสียของแต่ละแบบ

**Bare ชนะที่**:

- **เรียบง่าย ตรงไปตรงมากว่า** — client parse ได้ตรงเข้าที่ข้อมูลจริงทันที ไม่ต้องเจาะเข้า `.data` ก่อนเสมอ
- **"RESTful ในทางจิตวิญญาณ" มากกว่า** — response คือ**ตัวแทนของ resource นั้นเอง** ไม่ใช่ "envelope ที่บรรจุ
  resource" — สอดคล้องกับแนวคิดที่ว่า `GET /books/4821` ควรได้ "หนังสือเล่มที่ 4821 เอง" กลับมาตรง ๆ ไม่ใช่
  "กล่องที่มีหนังสือเล่มนั้นอยู่ข้างใน"

**Wrapped ชนะที่**:

- **มีที่วาง metadata โดยไม่ชนกับข้อมูลจริง** — ปัญหาใหญ่ที่สุดของ bare response คือ**ไม่มีที่ใส่ pagination
  metadata** (`total_count`, `next_cursor`) โดยไม่ทำลาย client เดิม เพราะ response เป็น array ตรง ๆ — ถ้าจะ
  เติม metadata เข้าไปทีหลัง ต้องเปลี่ยน response จาก `[...]` เป็น `{...}` ซึ่ง**เป็น breaking change ระดับ
  โครงสร้างที่ใหญ่ที่สุดที่มีได้** (หัวข้อ 78.11 จะอธิบายว่าทำไม) client เดิมทุกตัวที่คาดว่า response เป็น
  array จะพังทันทีเมื่อได้ object กลับมาแทน — ในขณะที่ wrapped response **เพิ่ม field ใหม่ใน `meta` ได้ตลอด
  ไปโดยไม่กระทบ client เดิมเลย** (client เก่าที่ไม่รู้จัก field ใหม่แค่มองไม่เห็นมัน ไม่ error)
- **โครงสร้างเดียวกันทั้ง single resource และ list** — `GET /books/4821` ได้ `{"data": {...}}` (object),
  `GET /books` ได้ `{"data": [...]}` (array) — client เขียนโค้ด "อ่าน `.data` เสมอ" ได้แบบเดียวกันไม่ต้องเช็ค
  ว่า endpoint นี้เป็น list หรือ single
- **แยก success/error ได้ชัดเจนด้วยโครงสร้างเดียวกัน** — ผูกกับ `AppError` ของ Part 66 ได้ตรง ๆ (ดูหัวข้อ
  ถัดไป)

**คำแนะนำของบทนี้**: **ใช้ wrapped response เป็น default สำหรับ API ที่ต้องรองรับ pagination หรือ
metadata อื่น ๆ ในระยะยาว** (ซึ่งคือ API ส่วนใหญ่ในทางปฏิบัติ) — ต้นทุนที่เสียไป (ต้องเจาะ `.data`
ก่อนเข้าถึงข้อมูลจริง) เล็กน้อยมากเทียบกับต้นทุนของการต้องทำ breaking change ทั้งระบบทีหลังเมื่อ requirement
เรื่อง pagination/metadata โผล่มา (ซึ่งเกิดขึ้นแทบทุกโปรเจกต์จริงไม่ช้าก็เร็ว) — บทนี้จะใช้ wrapped
response ตลอดทุกตัวอย่างที่เหลือ

#### ขยาย `AppError` ของ Part 66 ให้เป็น Envelope เต็มรูปแบบ

Part 66 หัวข้อ 66.3 สร้าง `AppError` ที่ `impl IntoResponse` คืน JSON shape สำหรับ**กรณี error**ไว้แล้ว:

```rust
use axum::{http::StatusCode, response::{IntoResponse, Response}, Json};
use serde_json::json;

#[derive(Debug, thiserror::Error)]
enum AppError {
    #[error("ไม่พบข้อมูล")]
    NotFound,
    #[error("ข้อมูลไม่ผ่านการตรวจสอบ: {0}")]
    Validation(String),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, code) = match &self {
            AppError::NotFound => (StatusCode::NOT_FOUND, "NOT_FOUND"),
            AppError::Validation(_) => (StatusCode::BAD_REQUEST, "VALIDATION_ERROR"),
        };
        // *** นี่คือจุดที่บทนี้ขยายจาก Part 66: ห่อ error ด้วย field "error" แทนการส่ง object เปลือย ***
        let body = json!({
            "error": { "code": code, "message": self.to_string() }
        });
        (status, Json(body)).into_response()
    }
}
```

สิ่งที่บทนี้ทำต่อคือ **ให้ฝั่ง success ใช้โครงสร้างที่ "คู่กัน" กับฝั่ง error** — success ใช้ field `data`
(+ `meta` เสริม), error ใช้ field `error` (+ `meta` เสริมแบบเดียวกัน) — client เขียนโค้ดตรวจสอบง่าย ๆ ได้
เพียงว่า **"ถ้า response มี field `error` คือ request ล้มเหลว ถ้ามี field `data` คือสำเร็จ"** โดยไม่ต้องพึ่ง
แค่ HTTP status code เพียงอย่างเดียว (แม้ status code ยังคงสำคัญเรื่อง protocol-level semantics ตาม Part 61
หัวข้อ 61.3 — envelope นี้เป็นชั้นข้อมูลเสริมที่ **ทำงานคู่กับ** status code ไม่ใช่แทนที่มัน):

```json
// success - single resource: GET /api/v1/books/4821
{
  "data": {"id": 4821, "title": "Rust in Production", "status": "open"},
  "meta": {"request_id": "req-a1b2"}
}

// success - list: GET /api/v1/books?limit=10
{
  "data": [
    {"id": 1, "title": "หนังสือเล่มที่ 01"},
    {"id": 2, "title": "หนังสือเล่มที่ 02"}
  ],
  "meta": {
    "request_id": "req-c3d4",
    "pagination": {"limit": 10, "next_cursor": 2, "has_next": true}
  }
}

// error: GET /api/v1/books/999999 (ไม่มีจริง)
{
  "error": {"code": "NOT_FOUND", "message": "ไม่พบข้อมูล"},
  "meta": {"request_id": "req-e5f6"}
}
```

สังเกตว่า `meta.request_id` ปรากฏอยู่ **ทั้งฝั่ง success และ error เหมือนกัน** — นี่คือประโยชน์ที่สำคัญมากใน
ทางปฏิบัติที่ envelope แบบ bare ทำไม่ได้เลย: เมื่อผู้ใช้ API รายงานบั๊กมา ("เรียก endpoint นี้แล้วได้ผลลัพธ์
แปลก ๆ") ทีม support สามารถขอ `request_id` จากเขา แล้วค้นหา log ฝั่ง server ที่ตรงกับ request นั้นเป๊ะได้ทันที
โดยไม่ต้องพึ่ง timestamp/IP ที่แม่นยำน้อยกว่ามาก — field เล็ก ๆ นี้เชื่อมโยงกับ `tracing::error!` ที่ Part 66
หัวข้อ 66.5 สอนไว้เรื่อง log ที่จุดเดียวได้ตรง ๆ (ใส่ `request_id` เดียวกันทั้งใน log line และใน response
body)

โครงสร้าง Rust ที่ implement แนวคิดนี้อย่างเป็นระบบ (แทนการเขียน `json!({...})` มือทุกจุดซึ่งเสี่ยงพิมพ์
field ผิดหรือลืม field):

```rust
use serde::Serialize;

// envelope กลางสำหรับ success response ทุกชนิด — generic เพื่อรองรับทั้ง T เดี่ยวและ Vec<T>
#[derive(Debug, Serialize)]
struct Envelope<T: Serialize> {
    data: T,
    meta: Meta,
}

#[derive(Debug, Serialize, Default)]
struct Meta {
    request_id: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    pagination: Option<PaginationMeta>,
}

#[derive(Debug, Serialize)]
struct PaginationMeta {
    limit: u64,
    next_cursor: Option<u64>,
    has_next: bool,
}

impl<T: Serialize> Envelope<T> {
    fn new(data: T, request_id: impl Into<String>) -> Self {
        Self { data, meta: Meta { request_id: request_id.into(), pagination: None } }
    }

    fn with_pagination(mut self, pagination: PaginationMeta) -> Self {
        self.meta.pagination = Some(pagination);
        self
    }
}
```

`#[serde(skip_serializing_if = "Option::is_none")]` (จาก Part 57-58 ที่สอน serde attribute พวกนี้ไว้แล้ว)
คือจุดสำคัญ — endpoint ที่ตอบ single resource (ไม่มี pagination) จะได้ JSON ที่**ไม่มี field `pagination`
โผล่มาเป็น `null` ให้รก** ในขณะที่ endpoint ที่เป็น list จะมี field นี้ครบ — นี่คือตัวอย่างที่ Part 58 หัวข้อ
เรื่อง `Option<T>` กับ serde มาบรรจบกับการออกแบบ API convention โดยตรง

### 78.5 Pagination เจาะลึก: Offset vs Keyset + `Link` Header ตาม RFC 8288

#### ทวนจาก Part 71: Offset-Based vs Keyset-Based

Part 71 หัวข้อ 71.3 สอนไว้แล้วอย่างละเอียดว่า:

- **Offset-based** (`LIMIT $n OFFSET $m`) — ใช้งานง่าย, รองรับ "ไปหน้าที่ N โดยตรง" ได้ (จำเป็นสำหรับ UI ที่
  มีปุ่มเลขหน้า 1 2 3 ... 10) แต่**ช้าลงเป็นเส้นตรงเมื่อหน้าลึกขึ้น** เพราะ database ต้องอ่านข้ามแถวที่ถูก
  skip ทุกแถวก่อนถึงแถวที่ต้องการจริง (Part 71 วัดจริงได้ว่า `OFFSET 49980` ช้ากว่า keyset ที่ตำแหน่งเดียวกัน
  ถึง **5.5 เท่า**)
- **Keyset/cursor-based** (`WHERE id > $cursor ORDER BY id LIMIT $n`) — ใช้ index ของคอลัมน์ที่ sort อยู่
  โดยตรง **ความเร็วคงที่ไม่ว่าจะอยู่หน้าไหน** แต่**ไม่รองรับ "ไปหน้าที่ N โดยตรง"** (ต้องไล่ตาม cursor
  ทีละหน้าเท่านั้น เหมาะกับ UI แบบ "infinite scroll" หรือ "ปุ่มถัดไป/ก่อนหน้า" มากกว่าปุ่มเลขหน้า)

บทนี้จะไม่สอน SQL ซ้ำ (Part 71 สอนไว้ครบแล้ว) แต่จะสอน**ชั้นที่อยู่เหนือ SQL**: **ที่ระดับ HTTP ควร expose
cursor นั้นออกมาให้ client อย่างไร** — คำตอบมาตรฐานที่หลาย API จริง (GitHub API, Stripe) ใช้กันคือ
**`Link` header ตาม RFC 8288**

#### RFC 8288: `Link` Header คืออะไร

RFC 8288 นิยามรูปแบบ header สำหรับสื่อว่า "resource ที่เกี่ยวข้องกับ response นี้อยู่ตรงไหน" โดยใช้
**relation type** (`rel="..."`) บอกความสัมพันธ์ — สำหรับ pagination ใช้สองค่าหลักคือ `rel="next"` (หน้า
ถัดไป) และ `rel="prev"` (หน้าก่อนหน้า) รูปแบบ:

```text
Link: <https://api.example.com/books?cursor=20&limit=5>; rel="next", <https://api.example.com/books?limit=5>; rel="prev"
```

**ข้อดีของการใส่ pagination ผ่าน `Link` header เทียบกับใส่ใน body**: (1) **client ที่ไม่สนใจ pagination
metadata ไม่ต้อง parse body ก่อนถึงจะรู้ว่ามีหน้าถัดไปไหม** — บาง HTTP client library (เช่น browser
prefetching, บาง REST client แบบสำเร็จรูป) อ่าน `Link` header ได้โดยอัตโนมัติโดยไม่ต้องเขียนโค้ด parse JSON
เอง (2) **แยกความรับผิดชอบชัดเจน**: header คือ "ข้อมูลเกี่ยวกับการขนส่ง/นำทาง" ส่วน body คือ "ข้อมูลจริงที่
ขอ" ตรงกับปรัชญาของ HTTP header ที่ Part 61 หัวข้อ 61.4 อธิบายไว้ (3) **ใช้คู่กับ `meta.pagination` ใน body
ได้พร้อมกัน** ไม่ขัดแย้งกัน — บทนี้จะทำทั้งสองอย่างพร้อมกัน (body มี `has_next`/`next_cursor` ให้ client ที่
สะดวกอ่าน JSON, header มี URL สำเร็จรูปให้ client ที่สะดวกอ่าน `Link`)

#### Implement จริง: Axum Handler ที่ส่ง `Link` Header

โค้ดนี้รันจริงและทดสอบผ่าน `cargo build`/`cargo run` + `curl` แล้ว (โปรเจกต์ทดสอบใช้
`axum = "0.8.9"`, `tokio = { version = "1", features = ["full"] }`, `serde = { version = "1", features =
["derive"] }`, `serde_json = "1"` — เวอร์ชันเดียวกับที่ Part 62 ตั้งไว้เป็นมาตรฐานของหลักสูตร):

```rust
use axum::{
    extract::Query,
    http::{HeaderMap, HeaderValue, StatusCode},
    response::IntoResponse,
    routing::get,
    Json, Router,
};
use serde::{Deserialize, Serialize};
use serde_json::json;

#[derive(Debug, Clone, Serialize)]
struct Book {
    id: u64,
    title: String,
    status: String,
    created_at: String,
}

fn seed_books() -> Vec<Book> {
    let statuses = ["open", "borrowed"];
    (1..=25)
        .map(|i| Book {
            id: i,
            title: format!("หนังสือเล่มที่ {i:02}"),
            status: statuses[(i % 2) as usize].to_string(),
            created_at: format!("2026-01-{:02}T00:00:00Z", (i % 28) + 1),
        })
        .collect()
}

#[derive(Debug, Deserialize)]
struct ListBooksQuery {
    cursor: Option<u64>, // id ของแถวสุดท้ายในหน้าก่อน (keyset cursor — ตาม Part 71 หัวข้อ 71.3)
    limit: Option<u64>,
}

async fn list_books(Query(q): Query<ListBooksQuery>) -> impl IntoResponse {
    let books = seed_books();

    // keyset pagination: กรองเอาแต่แถวที่ id มากกว่า cursor (แบบเดียวกับ `WHERE id > $cursor` ของ Part 71)
    let limit = q.limit.unwrap_or(10).clamp(1, 100);
    let after_cursor: Vec<&Book> = match q.cursor {
        Some(c) => books.iter().filter(|b| b.id > c).collect(),
        None => books.iter().collect(),
    };
    let page: Vec<&Book> = after_cursor.iter().take(limit as usize).cloned().collect();
    let has_next = after_cursor.len() > limit as usize;
    let next_cursor = page.last().map(|b| b.id);

    // สร้าง Link header ตาม RFC 8288: rel="next" ชี้ไปหน้าถัดไปด้วย cursor ใหม่
    let mut headers = HeaderMap::new();
    let base = "http://127.0.0.1:3078/api/v1/books";
    let mut links = Vec::new();
    if has_next {
        if let Some(nc) = next_cursor {
            links.push(format!(r#"<{base}?cursor={nc}&limit={limit}>; rel="next""#));
        }
    }
    if q.cursor.is_some() {
        links.push(format!(r#"<{base}?limit={limit}>; rel="prev""#));
    }
    if !links.is_empty() {
        if let Ok(v) = HeaderValue::from_str(&links.join(", ")) {
            headers.insert("Link", v);
        }
    }

    let body = json!({
        "data": page,
        "meta": {
            "request_id": "req-demo-0001",
            "pagination": { "limit": limit, "next_cursor": next_cursor, "has_next": has_next }
        }
    });

    (StatusCode::OK, headers, Json(body))
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/api/v1/books", get(list_books));
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3078").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

จุดที่ควรสังเกตในโค้ด: **`(StatusCode::OK, headers, Json(body))`** คือการคืนค่า tuple สามตัวที่ Axum
implement `IntoResponse` ให้ไว้แล้ว (status + header map + body) — ตรงกับที่ Part 63 หัวข้อ 63.8 สอนไว้ว่า
tuple แบบนี้คือหนึ่งใน pattern การคืนค่าที่ Axum รองรับให้แบบ built-in ไม่ต้องเขียน struct wrapper เอง —
`HeaderValue::from_str(...)` คืน `Result` เพราะ header value ต้องเป็น**ASCII visible character เท่านั้น**
ตาม HTTP spec (RFC 7230) — โค้ดใช้ `if let Ok(v) = ...` เพื่อข้ามการใส่ header ไปเงียบ ๆ ถ้า URL มีตัวอักษร
แปลก ๆ ที่ทำให้ header value ผิดรูปแบบ (ในตัวอย่างนี้ URL ประกอบจากตัวเลขล้วนจึงไม่มีวันเกิดปัญหานี้จริง แต่
เป็น pattern ที่ปลอดภัยไว้ก่อนสำหรับกรณีทั่วไปที่ URL อาจมี query param ที่มาจาก user input)

#### รันจริง: ผลลัพธ์จาก `curl` เรียกสองหน้าติดกัน

รันเซิร์ฟเวอร์ด้วย `cargo run` แล้วเรียกหน้าแรกด้วย `limit=5`:

```text
$ curl -s -D - "http://127.0.0.1:3078/api/v1/books?limit=5"
HTTP/1.1 200 OK
content-type: application/json
link: <http://127.0.0.1:3078/api/v1/books?cursor=5&limit=5>; rel="next"
content-length: 702
date: Sun, 27 Sep 2026 01:17:09 GMT

{"data":[{"created_at":"2026-01-02T00:00:00Z","id":1,"status":"borrowed","title":"หนังสือเล่มที่ 01"},
{"created_at":"2026-01-03T00:00:00Z","id":2,"status":"open","title":"หนังสือเล่มที่ 02"},
{"created_at":"2026-01-04T00:00:00Z","id":3,"status":"borrowed","title":"หนังสือเล่มที่ 03"},
{"created_at":"2026-01-05T00:00:00Z","id":4,"status":"open","title":"หนังสือเล่มที่ 04"},
{"created_at":"2026-01-06T00:00:00Z","id":5,"status":"borrowed","title":"หนังสือเล่มที่ 05"}],
"meta":{"pagination":{"has_next":true,"limit":5,"next_cursor":5},"request_id":"req-demo-0001"}}
```

**ไม่ต้องเปิด body เลย** — แค่อ่าน `Link` header ก็รู้ทันทีว่า URL ของหน้าถัดไปคืออะไร ตาม `rel="next"` ที่
ระบุมา ตอนนี้เรียก URL นั้นตรง ๆ:

```text
$ curl -s -D - "http://127.0.0.1:3078/api/v1/books?cursor=5&limit=5"
HTTP/1.1 200 OK
content-type: application/json
link: <http://127.0.0.1:3078/api/v1/books?cursor=10&limit=5>; rel="next", <http://127.0.0.1:3078/api/v1/books?limit=5>; rel="prev"
content-length: 700
date: Sun, 27 Sep 2026 01:17:09 GMT

{"data":[{"created_at":"2026-01-07T00:00:00Z","id":6,"status":"open","title":"หนังสือเล่มที่ 06"},
{"created_at":"2026-01-08T00:00:00Z","id":7,"status":"borrowed","title":"หนังสือเล่มที่ 07"},
{"created_at":"2026-01-09T00:00:00Z","id":8,"status":"open","title":"หนังสือเล่มที่ 08"},
{"created_at":"2026-01-10T00:00:00Z","id":9,"status":"borrowed","title":"หนังสือเล่มที่ 09"},
{"created_at":"2026-01-11T00:00:00Z","id":10,"status":"open","title":"หนังสือเล่มที่ 10"}],
"meta":{"pagination":{"has_next":true,"limit":5,"next_cursor":10},"request_id":"req-demo-0001"}}
```

สังเกตว่าหน้าที่สองมี `rel="prev"` เพิ่มขึ้นมา (เพราะ `cursor` ถูกส่งมาในครั้งนี้ แปลว่าไม่ใช่หน้าแรก) และ
`rel="next"` ชี้ไปที่ `cursor=10` ถูกต้อง (เป็น `id` ของแถวสุดท้ายในหน้านี้เป๊ะ) — client ที่เขียนแบบ "ตาม
`Link` header ไปเรื่อย ๆ จนกว่าจะไม่มี `rel=\"next\"`" จะเดินหน้าอ่านข้อมูลทั้ง collection ได้อย่างถูกต้อง
โดยไม่ต้องรู้เรื่อง cursor/keyset ภายในเลยแม้แต่นิดเดียว — นี่คือประโยชน์สำคัญที่สุดของการ expose pagination
ผ่าน `Link` header: **implementation detail ของ pagination (offset vs keyset) ถูกซ่อนไว้หลัง URL ทั้งหมด**
ถ้าวันหนึ่งทีมเปลี่ยนจาก keyset ไปใช้กลไกอื่น client เดิมที่ "ตาม link" อย่างเดียวจะไม่ได้รับผลกระทบเลย —
ตรงกันข้ามกับ client ที่ต้องคำนวณ `offset`/`cursor` เองจาก field ใน body ซึ่งต้องรู้ implementation detail
และจะพังถ้าเปลี่ยนกลไก

### 78.6 Filtering, Sorting Mini-DSL, และ Sparse Fieldsets

#### Filtering: Query Param ตรงไปตรงมาที่สุด

Part 63 สอน `Query<T>` extractor ไว้แล้วสำหรับ pagination (`page`/`limit`) — filter ก็ใช้กลไกเดียวกัน เพิ่ม
field ใหม่เข้าไปใน struct:

```rust
#[derive(Debug, Deserialize)]
struct ListBooksQuery {
    cursor: Option<u64>,
    limit: Option<u64>,
    status: Option<String>, // ?status=open — filter ตรงไปตรงมา
    sort: Option<String>,   // ?sort=-created_at,title — mini-DSL (อธิบายต่อไป)
    fields: Option<String>, // ?fields=id,title — sparse fieldset (อธิบายต่อไป)
}
```

การ filter เองไม่มีอะไรซับซ้อน (`books.retain(|b| &b.status == status)`) — ส่วนที่น่าสนใจกว่าคือ **sort**
และ **sparse fieldsets** เพราะทั้งสองต้อง parse "mini-DSL" ที่ซับซ้อนกว่า string เดี่ยวธรรมดา

#### Sort Mini-DSL: `?sort=-created_at,title`

Convention ที่ API จำนวนมากในอุตสาหกรรมใช้กัน (JSON:API, และอีกหลาย API สาธารณะ) คือ: **ค่าของ `sort` คือ
รายชื่อ field คั่นด้วย comma, field ที่มี `-` นำหน้าคือ descending, ไม่มี `-` คือ ascending** — ตัวอย่าง
`?sort=-created_at,title` แปลว่า "เรียงตาม `created_at` จากใหม่ไปเก่าก่อน แล้วถ้าเท่ากันให้เรียงตาม `title`
จาก A ไป Z" (multi-key sort — ใช้ field แรกเป็นหลัก field ถัดไปเป็นตัวตัดสินเมื่อ field แรกเท่ากัน)

Parse function ที่แปลง string นี้เป็นโครงสร้างที่ใช้ sort ได้จริง:

```rust
#[derive(Debug, Serialize)]
struct SortKey {
    field: String,
    desc: bool,
}

fn parse_sort(raw: &str) -> Vec<SortKey> {
    raw.split(',')
        .filter(|s| !s.trim().is_empty())
        .map(|s| {
            let s = s.trim();
            if let Some(field) = s.strip_prefix('-') {
                SortKey { field: field.to_string(), desc: true }
            } else {
                SortKey { field: s.to_string(), desc: false }
            }
        })
        .collect()
}
```

จุดที่น่าสนใจคือ `str::strip_prefix('-')` (จาก Part 14 ที่สอน string method พวกนี้ไว้แล้ว) คืน `Option<&str>`
— `Some(field)` แปลว่ามี `-` นำหน้าจริง (ตัด `-` ออกให้เรียบร้อยในตัวเดียว) `None` แปลว่าไม่มี — ใช้
`if let Some(...) = ... else` ได้ตรงประเด็นโดยไม่ต้องเช็ค `.starts_with('-')` แล้ว slice เองแยกสองขั้นตอน

การนำ `Vec<SortKey>` ไปใช้จริงกับ `sort_by` (ใช้หลักการ **short-circuit multi-key comparison** — ถ้า key
แรกเท่ากันให้ไปเทียบ key ถัดไป, จบด้วย `Ordering::Equal` ถ้าทุก key เท่ากันหมด):

```rust
books.sort_by(|a, b| {
    for k in &sort_keys {
        let ord = match k.field.as_str() {
            "title" => a.title.cmp(&b.title),
            "created_at" => a.created_at.cmp(&b.created_at),
            "id" => a.id.cmp(&b.id),
            _ => std::cmp::Ordering::Equal, // field ที่ไม่รู้จัก — เพิกเฉย ไม่ error (ดูกับดักข้อ 3)
        };
        let ord = if k.desc { ord.reverse() } else { ord };
        if ord != std::cmp::Ordering::Equal {
            return ord;
        }
    }
    std::cmp::Ordering::Equal
});
```

`Ordering::reverse()` (จาก `std::cmp::Ordering` ที่ Part 26 แนะนำไว้ตอนสอน iterator `sort_by`/`cmp`) คือ
วิธีที่สะอาดที่สุดในการกลับทิศทางการเรียง — แทนที่จะเขียน `b.field.cmp(&a.field)` สลับ `a`/`b` เอง (ซึ่งอ่าน
สับสนเมื่อผสมกับ multi-key sort) `.reverse()` สื่อความหมาย "descending" ได้ตรงกว่าและไม่เสี่ยงเขียนสลับผิด

#### Sparse Fieldsets: `?fields=id,title`

**ปัญหาที่ sparse fieldset แก้**: response เต็มของหนังสือหนึ่งเล่มอาจมี field เยอะมาก (`id`, `title`,
`author`, `isbn`, `description`, `cover_image_url`, `status`, `created_at`, `updated_at`, ...) — บาง client
(เช่น mobile app ที่ต้องประหยัด bandwidth, หรือหน้า list ที่โชว์แค่ชื่อกับสถานะ) ต้องการแค่ 2-3 field จาก
ทั้งหมด การส่ง field ที่ไม่ได้ใช้กลับไปทุกครั้งคือการสิ้นเปลือง bandwidth/parsing time ที่หลีกเลี่ยงได้ —
สารพัด API จริง (Facebook Graph API เป็นตัวอย่างที่รู้จักกันดีที่สุด) แก้ด้วย **sparse fieldsets**: client
ขอเฉพาะ field ที่ต้องการผ่าน `?fields=id,title`

Implementation ที่ทำงานที่ระดับ `serde_json::Value` (แทนที่จะแก้ struct หลักให้มี field เป็น `Option<T>`
ทุกตัวซึ่งจะทำให้โค้ดฝั่ง business logic รกไปด้วย `Option` ที่ไม่เกี่ยวกับ business rule จริง ๆ เลย):

```rust
let data: Vec<serde_json::Value> = page
    .iter()
    .map(|b| {
        let full = json!({
            "id": b.id, "title": b.title, "status": b.status, "created_at": b.created_at,
        });
        match &fields_param {
            Some(f) => {
                let wanted: Vec<&str> = f.split(',').map(|s| s.trim()).collect();
                let mut obj = serde_json::Map::new();
                if let serde_json::Value::Object(map) = full {
                    for k in wanted {
                        if let Some(v) = map.get(k) {
                            obj.insert(k.to_string(), v.clone());
                        }
                    }
                }
                serde_json::Value::Object(obj)
            }
            None => full, // ไม่ระบุ ?fields= -> ส่งครบทุก field ตามปกติ
        }
    })
    .collect();
```

หลักการสำคัญ: **serialize เป็น `serde_json::Value` แบบเต็มก่อนเสมอ แล้วค่อยกรอง field ออกทีหลัง** ไม่ใช่
พยายาม "ไม่ serialize field ที่ไม่ต้องการตั้งแต่แรก" — วิธีนี้ทำให้ business logic (การสร้างข้อมูล `Book`)
กับ presentation logic (จะแสดง field ไหนให้ client เห็น) **แยกกันสนิท** ไม่ปนกัน struct `Book` เองไม่ต้องรู้
เรื่อง sparse fieldset เลยแม้แต่นิดเดียว — trade-off คือมี allocation พิเศษ (`Map` ใหม่) ต่อ request ซึ่งถูก
มองว่าคุ้มค่าเทียบกับความสะอาดของโค้ดในระบบขนาดที่ response ไม่ได้ใหญ่ระดับหลักหมื่น field

#### รันจริง: ผลลัพธ์จาก `curl` ทดสอบ Filter + Sort + Sparse Fieldsets พร้อมกัน

```text
$ curl -s -D - "http://127.0.0.1:3078/api/v1/books?status=open&sort=-created_at,title&fields=id,title&limit=3"
HTTP/1.1 200 OK
content-type: application/json
link: <http://127.0.0.1:3078/api/v1/books?cursor=20&limit=3>; rel="next"
content-length: 304
date: Sun, 27 Sep 2026 01:17:16 GMT

{"data":[{"id":24,"title":"หนังสือเล่มที่ 24"},{"id":22,"title":"หนังสือเล่มที่ 22"},
{"id":20,"title":"หนังสือเล่มที่ 20"}],
"meta":{"pagination":{"has_next":true,"limit":3,"next_cursor":20},"request_id":"req-demo-0001"}}
```

ผลลัพธ์ยืนยันครบทุกเงื่อนไข: (1) **filter** เอาแต่หนังสือที่ `status=open` (เห็นจากว่าไม่มีเล่มเลขคู่ที่เป็น
`borrowed` ปนมา — ในข้อมูลตัวอย่างเลขคี่คือ `borrowed` เลขคู่คือ `open` ตาม `seed_books()`) (2) **sort**
เรียงจาก `created_at` ใหม่ไปเก่า (`24 -> 22 -> 20` ซึ่งวันที่ `created_at` คำนวณจาก `i % 28` จึงลดหลั่นตาม
`id` ในช่วงนี้พอดี) (3) **sparse fieldset** ส่งกลับแค่ `id`/`title` เท่านั้น ไม่มี `status`/`created_at`
โผล่มาแม้จะเป็น field ที่ใช้ filter/sort ไปแล้วก็ตาม — พิสูจน์ว่า filter/sort/fields ทำงาน**อิสระจากกัน**ได้
ถูกต้องแม้ใช้พร้อมกันทั้งสามอย่าง

### 78.7 Idempotency-Key ในทางปฏิบัติ: Implement จริงด้วย Axum (ต่อยอดจาก Part 61)

#### ทวนสิ่งที่ Part 61 ยังไม่ได้ทำ

Part 61 หัวข้อ 61.9 สาธิตแนวคิด Idempotency-Key ด้วย `struct BookingServer` ที่เป็น `HashMap` ธรรมดาไม่มี
HTTP เกี่ยวข้องเลย — บทนี้จะ implement เป็น **Axum handler จริง** ที่อ่าน header `Idempotency-Key` จาก
request จริง เก็บ**ทั้ง response ที่เคยตอบไปแล้วแบบเต็ม** (ไม่ใช่แค่ `booking_id`) และเพิ่มการตรวจสอบที่
Part 61 พูดถึงไว้เป็น "ข้อควรระวัง" แต่ไม่ได้ implement จริง: **การตรวจว่า request body เหมือนกันด้วย ไม่ใช่
แค่ key ตรงกัน**

#### โครงสร้างข้อมูลและ State

```rust
use axum::{
    extract::State,
    http::{HeaderMap, HeaderValue, StatusCode},
    response::{IntoResponse, Response},
    Json,
};
use serde::{Deserialize, Serialize};
use serde_json::json;
use std::{collections::HashMap, sync::{Arc, Mutex}};

#[derive(Debug, Clone, Serialize, Deserialize)]
struct Booking {
    id: u64,
    event_name: String,
    status: String, // "confirmed" | "cancelled"
    created_at: String,
}

#[derive(Debug, Deserialize)]
struct CreateBookingRequest {
    event_name: String,
}

// เก็บทั้ง response ที่เคยตอบไปแล้ว (ไม่ใช่แค่ id) เพื่อคืนผลลัพธ์เดิมเป๊ะตอน retry
#[derive(Clone)]
struct IdempotencyRecord {
    request_body_hash: u64, // ใช้ตรวจว่า retry มา body เดิมจริงไหม (ไม่ใช่ key ชนกันโดยบังเอิญ)
    status: StatusCode,
    body: serde_json::Value,
}

#[derive(Clone)]
struct AppState {
    bookings: Arc<Mutex<HashMap<u64, Booking>>>,
    next_booking_id: Arc<Mutex<u64>>,
    idempotency_store: Arc<Mutex<HashMap<String, IdempotencyRecord>>>,
}
```

`Arc<Mutex<HashMap<...>>>` คือ pattern เดียวกับที่ Part 64 สอนเรื่อง shared mutable state ข้าม request —
`idempotency_store` เก็บแยกจาก `bookings` เพราะมันคือ**ข้อมูลระดับ HTTP protocol** (การจดจำว่า request ไหน
เคยถูกประมวลผลไปแล้ว) ไม่ใช่**ข้อมูลระดับ business** (การจองจริง) — แยกกันชัดเจนแบบนี้ทำให้เข้าใจง่ายว่าถ้า
วันหนึ่งต้องย้าย `idempotency_store` ไปเก็บใน Redis (ตามที่ Part 83 จะสอน) จะไม่กระทบโครงสร้างข้อมูลของ
`bookings` เลยแม้แต่นิดเดียว

#### `simple_hash`: ตรวจจับการใช้ Key ซ้ำกับ Body ต่างกัน

```rust
fn simple_hash(bytes: &[u8]) -> u64 {
    // FNV-1a แบบง่าย ๆ พอสำหรับตรวจว่า body เดิมหรือไม่ (ไม่ใช่ cryptographic hash — ไม่ต้องทนต่อการปลอมแปลง
    // เพราะจุดประสงค์คือตรวจจับ "ความผิดพลาดของ client" ไม่ใช่ป้องกันการโจมตี)
    let mut hash: u64 = 0xcbf29ce484222325;
    for &b in bytes {
        hash ^= b as u64;
        hash = hash.wrapping_mul(0x100000001b3);
    }
    hash
}
```

**เหตุผลที่ต้องมี hash นี้เลย**: สมมติ client A ส่ง `Idempotency-Key: abc-123` พร้อม body
`{"event_name": "Rust Conf"}` แล้วสำเร็จ — ถ้าอีกวันหนึ่ง (หรือ client อื่น เพราะ key เป็น string ที่ generate
เองไม่มีการันตีว่า unique ข้ามทุก client) ส่ง `Idempotency-Key: abc-123` เดิม**แต่ body ต่างออกไป**
(`{"event_name": "PyCon"}`) — ถ้าไม่ตรวจ hash ระบบจะ**คืนผลลัพธ์ของการจอง "Rust Conf" กลับไปทั้งที่ client
ตั้งใจจอง "PyCon"** ซึ่งเป็นบั๊กที่อันตรายกว่าไม่มี idempotency เสียอีก (เงียบ ๆ ให้ผลลัพธ์ผิดไปเลย โดยไม่มี
error อะไรบอก) — Part 61 หัวข้อ 61.9 พูดถึงเรื่องนี้ไว้เป็น "ข้อควรระวังเชิงปฏิบัติ" แต่ไม่ได้เขียนโค้ดตรวจสอบ
จริง บทนี้ implement เต็มรูปแบบ: ถ้า hash ไม่ตรงกัน **ตอบ `422 Unprocessable Entity`** (ตามหลัก Part 61
หัวข้อ 61.3 ที่ `422` ใช้กับกรณี "syntax ถูกต้องแต่ขัดกับกฎ" — ในกรณีนี้คือ "การใช้ key ซ้ำกับเจตนาที่ต่างกัน
คือขัดกับกฎการใช้งาน `Idempotency-Key`") ไม่ใช่เงียบ ๆ คืนผลลัพธ์เก่าไปให้แบบผิด ๆ

#### Handler เต็มรูปแบบ

```rust
async fn create_booking(
    State(state): State<AppState>,
    headers: HeaderMap,
    body: axum::body::Bytes, // ดึง body แบบ raw bytes ก่อน เพื่อ hash ได้ก่อนแปลงเป็น struct
) -> Response {
    let idempotency_key = headers
        .get("Idempotency-Key")
        .and_then(|v| v.to_str().ok())
        .map(|s| s.to_string());
    let body_hash = simple_hash(&body);

    let payload: CreateBookingRequest = match serde_json::from_slice(&body) {
        Ok(p) => p,
        Err(_) => {
            return (
                StatusCode::BAD_REQUEST,
                Json(json!({"error": {"code": "BAD_REQUEST", "message": "JSON body ไม่ถูกต้อง"}})),
            ).into_response()
        }
    };

    if let Some(key) = &idempotency_key {
        let store = state.idempotency_store.lock().unwrap();
        if let Some(record) = store.get(key) {
            if record.request_body_hash != body_hash {
                return (
                    StatusCode::UNPROCESSABLE_ENTITY,
                    Json(json!({"error": {
                        "code": "IDEMPOTENCY_KEY_REUSED",
                        "message": "Idempotency-Key นี้เคยถูกใช้กับ request body ที่ต่างออกไป"
                    }})),
                ).into_response();
            }
            // เจอ key เดิม + body เดิม -> คืน response เดิมเป๊ะ ไม่สร้าง booking ซ้ำ
            let mut resp = (record.status, Json(record.body.clone())).into_response();
            resp.headers_mut().insert("Idempotent-Replayed", HeaderValue::from_static("true"));
            return resp;
        }
    }

    // สร้าง booking ใหม่จริง (path นี้วิ่งเฉพาะตอนไม่เคยเห็น key นี้มาก่อน)
    let mut next_id = state.next_booking_id.lock().unwrap();
    let id = *next_id;
    *next_id += 1;
    drop(next_id); // ปล่อย lock ก่อนไปแตะ mutex ตัวอื่น (bookings) — กัน deadlock/lock contention ไม่จำเป็น

    let booking = Booking {
        id,
        event_name: payload.event_name,
        status: "confirmed".to_string(),
        created_at: "2026-09-27T10:00:00Z".to_string(),
    };
    state.bookings.lock().unwrap().insert(id, booking.clone());

    let response_body = json!({
        "data": booking_to_json(&booking),
        "meta": { "request_id": format!("req-booking-{id}") }
    });

    if let Some(key) = idempotency_key {
        state.idempotency_store.lock().unwrap().insert(key, IdempotencyRecord {
            request_body_hash: body_hash,
            status: StatusCode::CREATED,
            body: response_body.clone(),
        });
    }

    let mut resp = (StatusCode::CREATED, Json(response_body)).into_response();
    resp.headers_mut().insert("Location", HeaderValue::from_str(&format!("/api/v1/bookings/{id}")).unwrap());
    resp
}
```

จุดที่ควรสังเกตหลายจุด: **`body: axum::body::Bytes` เป็น parameter สุดท้ายเสมอ** ตามกฎที่ Part 63 หัวข้อ
63.6-63.7 สอนไว้ (extractor ที่ "กิน" body ต้องอยู่ท้ายสุด — `HeaderMap` และ `State<T>` อ่านจาก `parts`
ล้วน ๆ จึงวางไว้ก่อนได้) — เหตุผลที่ดึงเป็น `Bytes` แบบ raw ก่อนแทนที่จะใช้ `Json<CreateBookingRequest>`
ตรง ๆ คือ **ต้อง hash "ตัวบ่งบอกเนื้อ body ดิบ" ไม่ใช่ struct ที่ deserialize แล้ว** (ถ้า deserialize ก่อนแล้ว
hash struct นั้น จะมีปัญหาเรื่อง "field order ต่างกันใน JSON แต่ struct เหมือนกันควรนับว่า body เดียวกัน
หรือไม่" ซึ่งซับซ้อนกว่าที่ควรจะเป็นสำหรับตัวอย่างนี้ — hash raw bytes ตรงไปตรงมากว่าและตรงกับที่ API จริง
อย่าง Stripe อธิบายพฤติกรรมของตนเองไว้) — `drop(next_id)` ก่อนแตะ `state.bookings.lock()` คือการปล่อย
mutex guard ตัวแรกก่อนเปิด mutex guard ตัวที่สอง ตามหลักที่ Part 39 สอนไว้เรื่องการถือ lock สองตัวพร้อมกัน
เสี่ยง deadlock ถ้า handler อื่นถือ order สลับกัน — ในโค้ดนี้ order เดียวกันเสมอ (`next_booking_id` ก่อน
`bookings`) ก็ปลอดภัยอยู่แล้วแม้ไม่ `drop()` แต่การปล่อย lock ทันทีที่ไม่ต้องใช้แล้วก็เป็นนิสัยที่ดีเสมอ ลด
เวลาที่ thread อื่นต้องรอ lock นี้ให้น้อยที่สุด

#### รันจริง: ทดสอบ Create → Retry → Conflict → Key ใหม่

```text
$ curl -s -D - -X POST "http://127.0.0.1:3078/api/v1/bookings" \
    -H "Content-Type: application/json" -H "Idempotency-Key: abc-123" \
    -d '{"event_name":"Rust Conf Bangkok 2026"}'
HTTP/1.1 201 Created
content-type: application/json
location: /api/v1/bookings/1
content-length: 260

{"data":{"_links":{"cancel":{"href":"/api/v1/bookings/1/cancel","method":"POST"},
"self":{"href":"/api/v1/bookings/1"}},"created_at":"2026-09-27T10:00:00Z",
"event_name":"Rust Conf Bangkok 2026","id":1,"status":"confirmed"},"meta":{"request_id":"req-booking-1"}}
```

```text
$ curl -s -D - -X POST "http://127.0.0.1:3078/api/v1/bookings" \
    -H "Content-Type: application/json" -H "Idempotency-Key: abc-123" \
    -d '{"event_name":"Rust Conf Bangkok 2026"}'
HTTP/1.1 201 Created
content-type: application/json
idempotent-replayed: true
content-length: 260

{"data":{"_links":{"cancel":{"href":"/api/v1/bookings/1/cancel","method":"POST"},
"self":{"href":"/api/v1/bookings/1"}},"created_at":"2026-09-27T10:00:00Z",
"event_name":"Rust Conf Bangkok 2026","id":1,"status":"confirmed"},"meta":{"request_id":"req-booking-1"}}
```

**ผลลัพธ์ retry ได้ `id: 1` เดิมเป๊ะ** (ไม่ใช่ `id: 2`) และ response body **เหมือนกันทุกตัวอักษร** กับครั้ง
แรก มีเพียง header `Idempotent-Replayed: true` เพิ่มเข้ามาบอกทีมที่ debug ว่านี่คือ response ที่ replay มา
ไม่ใช่การประมวลผลใหม่ (header นี้ไม่ใช่มาตรฐาน RFC แต่เป็น convention ที่หลาย API ใช้เพื่อความสะดวกในการ
สังเกตเวลา debug — client ไม่จำเป็นต้องอ่าน header นี้เพื่อทำงานถูกต้อง แต่มีไว้ช่วยทีม operations)

ทดสอบกรณีที่ Part 61 เตือนไว้แต่ไม่มีโค้ดพิสูจน์: ใช้ key เดิมกับ body ต่างกัน

```text
$ curl -s -D - -X POST "http://127.0.0.1:3078/api/v1/bookings" \
    -H "Content-Type: application/json" -H "Idempotency-Key: abc-123" \
    -d '{"event_name":"Different Event"}'
HTTP/1.1 422 Unprocessable Entity
content-type: application/json
content-length: 167

{"error":{"code":"IDEMPOTENCY_KEY_REUSED","message":"Idempotency-Key นี้เคยถูกใช้กับ request body ที่ต่างออกไป"}}
```

และ key ใหม่ที่ต่างออกไปทั้งหมด ควรสร้าง booking ใหม่จริง:

```text
$ curl -s -D - -X POST "http://127.0.0.1:3078/api/v1/bookings" \
    -H "Content-Type: application/json" -H "Idempotency-Key: xyz-999" \
    -d '{"event_name":"PyCon Bangkok 2026"}'
HTTP/1.1 201 Created
content-type: application/json
location: /api/v1/bookings/2
content-length: 256

{"data":{...,"event_name":"PyCon Bangkok 2026","id":2,"status":"confirmed"},"meta":{"request_id":"req-booking-2"}}
```

ทั้งสี่ผลลัพธ์ยืนยันครบตามที่ออกแบบไว้: retry ด้วย key+body เดิมได้ผลลัพธ์เดิมเป๊ะ, key ซ้ำกับ body ต่างกันถูก
ปฏิเสธด้วย `422`, และ key ใหม่จริงสร้าง booking ใหม่จริง (`id: 2`) โดยไม่กระทบ booking แรกเลย

#### ข้อจำกัดของ In-Memory Store และทางไปสู่ Production

`idempotency_store: Arc<Mutex<HashMap<String, IdempotencyRecord>>>` ในตัวอย่างนี้มีข้อจำกัดสำคัญสองข้อที่
**ต้องแก้ก่อนขึ้น production จริง**: (1) **ไม่มี TTL** — ข้อมูลอยู่ใน memory ตลอดไปจนกว่า process จะ restart
ทำให้ memory โตไม่มีที่สิ้นสุดถ้ามี request จำนวนมาก (2) **ไม่รอดจาก restart และไม่ถูกแชร์ข้าม instance** —
ถ้า deploy หลาย instance ของ server เดียวกัน (ตามที่ Part 61 หัวข้อ statelessness อธิบายว่าทำไม HTTP server
ควร scale แนวนอนได้) request สองครั้งที่มี `Idempotency-Key` เดียวกันอาจไปตกที่ instance คนละตัวที่ไม่รู้จัก
key ของกันและกันเลย ทำให้ idempotency ใช้ไม่ได้ผลจริงในระบบที่มีหลาย instance — **Part 83 (Caching ด้วย
Redis)** จะแก้ทั้งสองปัญหานี้พร้อมกัน: Redis รองรับ TTL แบบ built-in (`SET key value EX seconds`) และเป็น
store กลางที่ทุก instance ของ server เข้าถึงร่วมกันได้ — โครงสร้างตรรกะของ handler (เช็ค key ก่อน, ถ้าไม่มี
ประมวลผลจริงแล้วเก็บผลลัพธ์, ถ้ามีแล้วคืนของเดิม) **ไม่ต้องเปลี่ยนเลย** เปลี่ยนแค่ชนิดของ `idempotency_store`
จาก `Arc<Mutex<HashMap<...>>>` เป็น Redis client — นี่คือตัวอย่างที่ดีว่าทำไมการแยก "protocol-level concern"
(idempotency store) ออกจาก "business logic" (การสร้าง booking) ให้ชัดเจนตั้งแต่แรกทำให้การย้าย infrastructure
ทีหลังง่ายขึ้นมาก

### 78.8 HATEOAS แบบปฏิบัติได้จริง: เลือกใส่ `_links` เฉพาะที่มีประโยชน์จริง

#### ทวนจาก Part 61: HATEOAS เต็มรูปแบบ vs ความจริงที่ API ส่วนใหญ่ทำ

Part 61 หัวข้อ 61.7 อธิบายไว้แล้วว่า HATEOAS เต็มรูปแบบ (ทุก response มี `_links` ครบทุก action ที่เป็นไปได้)
คือ**อุดมคติที่ API จริงส่วนใหญ่ไม่ได้ทำ** เพราะความซับซ้อนที่เพิ่มขึ้นไม่คุ้มกับประโยชน์ในทางปฏิบัติ — แต่
สิ่งที่ Part 61 ยังไม่ได้แสดงคือ **"ทางกลาง" ที่ API จริงจำนวนมากใช้กัน**: ใส่ `_links` **เฉพาะบางจุดที่มี
ประโยชน์ชัดเจนจริง ๆ** ไม่ใช่ทุก action ที่เป็นไปได้ทั้งหมด

#### ประโยชน์ที่แท้จริงของ `_links` แบบเลือกสรร: บอก "ทำอะไรได้ตอนนี้" โดยไม่ต้องมี Business Rule ซ้ำสองที่

ปัญหาจริงที่ `_links` แบบเลือกสรรแก้ได้ตรงจุด: สมมติกฎธุรกิจของระบบจองคือ **"ยกเลิกการจองได้เฉพาะตอนที่
status ยังเป็น `confirmed` เท่านั้น"** (ยกเลิกซ้ำ หรือยกเลิกการจองที่ถูกยกเลิกไปแล้วไม่มีความหมาย) — ถ้าไม่มี
`_links` เลย **client ต้องรู้กฎนี้เองและ implement ซ้ำ**ในโค้ดฝั่ง frontend (เช่น ซ่อนปุ่ม "ยกเลิก" เมื่อ
`status !== 'confirmed'`) — ปัญหาคือ **กฎธุรกิจนี้ถูกกำหนดไว้สองที่ที่ต้องซิงค์กันตลอดไป**: ครั้งหนึ่งใน
backend (ตอน validate ว่ายกเลิกได้จริงไหม) และอีกครั้งใน frontend (ตอนตัดสินใจว่าจะโชว์ปุ่มไหม) — ถ้าวันหนึ่ง
backend เปลี่ยนกฎ (เช่น "ยกเลิกได้จนถึง 24 ชั่วโมงก่อนงานเริ่มเท่านั้น") แต่ frontend ไม่ได้อัปเดตตาม จะเกิด
UI ที่โชว์ปุ่ม "ยกเลิก" ทั้งที่กดแล้วจะถูก backend ปฏิเสธ (หรือแย่กว่านั้นคือซ่อนปุ่มไว้ทั้งที่จริง ๆ ยกเลิกได้)

การใส่ `_links.cancel` **เฉพาะตอนที่ยกเลิกได้จริง** ย้าย "จุดที่ตัดสินกฎนี้" กลับไปอยู่ที่**เดียว**คือ backend
— frontend แค่เช็คว่า `_links.cancel` มีอยู่ไหม (`if (booking._links.cancel) { showCancelButton() }`) โดย
**ไม่ต้องรู้เงื่อนไขของกฎเลยแม้แต่นิดเดียว** — backend เปลี่ยนกฎเมื่อไรก็ได้ (`confirmed` เท่านั้น →
`confirmed` และยังไม่เลย 24 ชั่วโมงก่อนงาน) โดย frontend **ไม่ต้อง deploy ใหม่เลย** เพราะ logic การตัดสินใจ
อยู่ใน `_links` ที่ backend คำนวณให้แล้วทุกครั้ง — นี่คือประโยชน์ที่แท้จริงของ HATEOAS แบบเลือกสรร: **ไม่ใช่
"navigation แบบ hypermedia เต็มรูปแบบ" แต่คือ "ย้าย business rule ที่เกี่ยวกับ 'ทำอะไรต่อได้ไหม' จาก client
กลับไปอยู่ที่ server จุดเดียว"**

#### Implement จริง

```rust
// HATEOAS แบบเลือกเฉพาะ link ที่มีประโยชน์จริง: ใส่ _links.cancel เฉพาะตอนที่ยัง cancel ได้
fn booking_to_json(b: &Booking) -> serde_json::Value {
    let mut obj = json!({
        "id": b.id,
        "event_name": b.event_name,
        "status": b.status,
        "created_at": b.created_at,
        "_links": {
            "self": { "href": format!("/api/v1/bookings/{}", b.id) }
        }
    });
    if b.status == "confirmed" {
        obj["_links"]["cancel"] = json!({
            "href": format!("/api/v1/bookings/{}/cancel", b.id),
            "method": "POST"
        });
    }
    obj
}

async fn cancel_booking(State(state): State<AppState>, Path(id): Path<u64>) -> Response {
    let mut bookings = state.bookings.lock().unwrap();
    match bookings.get_mut(&id) {
        Some(b) => {
            b.status = "cancelled".to_string();
            (StatusCode::OK, Json(json!({
                "data": booking_to_json(b),
                "meta": {"request_id": "req-cancel-1"}
            }))).into_response()
        }
        None => (
            StatusCode::NOT_FOUND,
            Json(json!({"error": {"code": "NOT_FOUND", "message": "ไม่พบการจองนี้"}})),
        ).into_response(),
    }
}
```

**`self` ถูกใส่ไว้เสมอไม่มีเงื่อนไข** — นี่เป็น link เดียวที่ HATEOAS เต็มรูปแบบและ HATEOAS แบบเลือกสรรเห็น
พ้องกันว่าควรมีเสมอ เพราะมันไม่ใช่ business rule แต่เป็นข้อเท็จจริงที่ไม่เปลี่ยน (resource นี้อยู่ที่ URL นี้
เสมอ ไม่ว่า status จะเป็นอะไร) — ในขณะที่ `cancel` **ผูกกับเงื่อนไข `b.status == "confirmed"` ตรง ๆ** ซึ่งคือ
กฎธุรกิจของระบบนี้ — จุดสำคัญคือ**เงื่อนไขนี้เขียนอยู่ที่เดียว** (ในฟังก์ชัน `booking_to_json`) ทั้ง `GET
/bookings/{id}` และ response ของ `POST /bookings/{id}/cancel` เอง**เรียกฟังก์ชันเดียวกันนี้** จึงการันตีว่า
`_links` ที่ทั้งสอง endpoint ส่งกลับมาจะสม่ำเสมอกันเสมอ ไม่มีทางที่ endpoint หนึ่งบอกว่า cancel ได้แต่อีก
endpoint บอกว่าไม่ได้เพราะลืมอัปเดตจุดใดจุดหนึ่ง

#### รันจริง: พิสูจน์ว่า `_links.cancel` หายไปหลัง Cancel สำเร็จ

```text
$ curl -s "http://127.0.0.1:3078/api/v1/bookings/1"
{"data":{"_links":{"cancel":{"href":"/api/v1/bookings/1/cancel","method":"POST"},
"self":{"href":"/api/v1/bookings/1"}},"created_at":"2026-09-27T10:00:00Z",
"event_name":"Rust Conf Bangkok 2026","id":1,"status":"confirmed"},"meta":{"request_id":"req-get-1"}}

$ curl -s -X POST "http://127.0.0.1:3078/api/v1/bookings/1/cancel"
{"data":{"_links":{"self":{"href":"/api/v1/bookings/1"}},"created_at":"2026-09-27T10:00:00Z",
"event_name":"Rust Conf Bangkok 2026","id":1,"status":"cancelled"},"meta":{"request_id":"req-cancel-1"}}

$ curl -s "http://127.0.0.1:3078/api/v1/bookings/1"
{"data":{"_links":{"self":{"href":"/api/v1/bookings/1"}},"created_at":"2026-09-27T10:00:00Z",
"event_name":"Rust Conf Bangkok 2026","id":1,"status":"cancelled"},"meta":{"request_id":"req-get-1"}}
```

การเรียกครั้งแรก (ก่อนยกเลิก) มี `_links.cancel` ครบ — หลังยกเลิกสำเร็จ (`status` เปลี่ยนเป็น `"cancelled"`)
**`_links.cancel` หายไปทันทีทั้งใน response ของการยกเลิกเอง และใน `GET` ครั้งถัดไป** — พิสูจน์ว่า logic ทำงาน
ถูกต้องตามที่ออกแบบไว้ และสอดคล้องกันทุก endpoint ที่เกี่ยวข้อง

### 78.9 Rate Limiting Design: HTTP Contract ที่ทีม Frontend ใช้ได้จริงตั้งแต่วันนี้

#### ทำไมพูดเรื่อง "Contract" ก่อน "Implementation" จริง

หัวเรื่องบอกไว้ตรง ๆ ว่างานนี้เป็น **HTTP contract** ก่อนเป็น distributed rate limiter จริง — เหตุผลคือ
rate limiter ระดับ production ที่ทำงานถูกต้องข้ามหลาย instance ของ server (ตามที่ Part 61 อธิบายเรื่อง
statelessness) **ต้องพึ่ง store กลางอย่าง Redis** ซึ่งยังไม่ได้สอนจนกว่าจะถึง **Part 83** — แต่สิ่งที่**ทำได้
ตั้งแต่วันนี้และสำคัญไม่แพ้กัน**คือการตกลง**รูปแบบ response และ header** ที่ทีม frontend เอาไปเขียนโค้ด
handle ได้เลย โดยไม่ต้องรอ backend มี distributed rate limiter จริงก่อน — เมื่อวันหนึ่ง backend ย้ายจาก
in-memory ไป Redis, **contract ที่ frontend เห็นจะไม่เปลี่ยนแม้แต่ byte เดียว** เปลี่ยนแค่ความแม่นยำ (accurate
ข้ามหลาย instance) เท่านั้น

#### Contract มาตรฐาน: `429`, `Retry-After`, `X-RateLimit-*`

**สถานการณ์ที่ rate limit เกิน**: ตอบ **`429 Too Many Requests`** (Part 61 หัวข้อ 61.3 กล่าวถึงไว้แล้ว) พร้อม
header **`Retry-After`** (จำนวนวินาทีที่ควรรอก่อนลองใหม่ — เป็น standard HTTP header ตาม RFC 9110 ใช้ร่วม
กับทั้ง `429` และ `503`) — **ทุก response** (ไม่ว่าสำเร็จหรือถูก rate limit) ควรมีสาม header เสริมที่เป็น
**de facto convention** (ไม่ใช่ RFC มาตรฐานแต่ถูกใช้กันแพร่หลายมากจนกลายเป็นความคาดหวังโดยพฤตินัยของนัก
พัฒนา — GitHub API, Twitter/X API ใช้รูปแบบเดียวกันนี้):

- **`X-RateLimit-Limit`**: จำนวน request สูงสุดที่อนุญาตต่อ window
- **`X-RateLimit-Remaining`**: จำนวน request ที่เหลือใน window ปัจจุบัน
- **`X-RateLimit-Reset`**: อีกกี่วินาที (หรือ Unix timestamp ขึ้นกับ convention ที่แต่ละ API เลือก — ตัวอย่าง
  นี้ใช้ "อีกกี่วินาที" เพื่อความง่ายในการอ่าน) ที่ quota จะถูก reset เต็ม

**response shape ของ error**: ใช้ envelope เดียวกันกับหัวข้อ 78.4 ทุกประการ — ไม่มีเหตุผลให้ error จาก rate
limit ต่างไปจาก error อื่น ๆ ในระบบ:

```json
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "เรียก API ถี่เกินกำหนด กรุณาลองใหม่ภายหลัง"
  }
}
```

#### Implement Token Bucket แบบ In-Memory (จริง ใช้ทดสอบ Contract)

Token bucket คือ algorithm ที่ใช้กันแพร่หลายที่สุดสำหรับ rate limiting — แนวคิด: มี "ถัง" ที่เติม token ด้วย
อัตราคงที่ (`refill_per_sec`) จนถึง capacity สูงสุด (`capacity`) ทุกครั้งที่มี request เข้ามาต้อง "จ่าย" 1
token ถ้าถังมี token พอก็อนุญาต ถ้าไม่พอก็ปฏิเสธ — ข้อดีของ token bucket เทียบกับการนับ request ตรง ๆ ใน
window คงที่คือ**รองรับ "burst" ได้ในระดับที่ควบคุมได้** (ถ้าถังเต็มพอดี ยิงรัว ๆ ได้เท่ากับ capacity ในทันที
โดยไม่ต้องรอ แต่หลังจากนั้นต้องรอให้ token เติมใหม่ตามอัตราที่กำหนด):

```rust
use std::time::Instant;

struct TokenBucket {
    tokens: f64,
    capacity: f64,
    refill_per_sec: f64,
    last_refill: Instant,
}

impl TokenBucket {
    fn new(capacity: f64, refill_per_sec: f64) -> Self {
        Self { tokens: capacity, capacity, refill_per_sec, last_refill: Instant::now() }
    }

    // คืน (allowed, remaining, retry_after_secs, reset_in_secs)
    fn try_consume(&mut self) -> (bool, u64, u64, u64) {
        let now = Instant::now();
        let elapsed = now.duration_since(self.last_refill).as_secs_f64();
        self.tokens = (self.tokens + elapsed * self.refill_per_sec).min(self.capacity);
        self.last_refill = now;

        if self.tokens >= 1.0 {
            self.tokens -= 1.0;
            let remaining = self.tokens.floor().max(0.0) as u64;
            let reset_in = ((self.capacity - self.tokens) / self.refill_per_sec).ceil() as u64;
            (true, remaining, 0, reset_in)
        } else {
            let retry_after = ((1.0 - self.tokens) / self.refill_per_sec).ceil() as u64;
            (false, 0, retry_after, retry_after)
        }
    }
}
```

จุดที่น่าสนใจ: **`tokens` เป็น `f64` ไม่ใช่จำนวนเต็ม** — เพราะการเติม token คำนวณจาก `elapsed * refill_per_sec`
ซึ่งเป็นค่าต่อเนื่อง (ถ้า `refill_per_sec = 1.0` และเวลาผ่านไป 0.3 วินาที ควรเติมได้ 0.3 token ไม่ใช่ 0 หรือ 1
เต็ม ๆ ซึ่งจะทำให้ rate จริงเบี่ยงเบนจากที่ตั้งใจ) — การคำนวณ `elapsed` แบบ **lazy** (คำนวณตอนที่มี request
เข้ามาเท่านั้น ไม่มี background task คอย "เติม token" ทุก ๆ กี่มิลลิวินาที) คือเทคนิคมาตรฐานที่หลีกเลี่ยงการ
ต้องมี timer thread แยกสำหรับ rate limiter — ถูกต้องทางคณิตศาสตร์เหมือนกันเพราะไม่ว่าจะเช็คตอนไหน สูตร
`elapsed * refill_per_sec` ก็คำนวณ token ที่ควรจะเติมไปแล้วได้ถูกต้องเสมอ

#### Handler ที่ใช้ Token Bucket และส่ง Header ครบ

```rust
async fn rate_limited_endpoint(State(state): State<AppState>) -> Response {
    let (allowed, remaining, retry_after, reset_in) = {
        let mut bucket = state.rate_bucket.lock().unwrap();
        bucket.try_consume()
    };

    let mut headers = HeaderMap::new();
    headers.insert("X-RateLimit-Limit", HeaderValue::from_static("5"));
    headers.insert("X-RateLimit-Remaining", HeaderValue::from_str(&remaining.to_string()).unwrap());
    headers.insert("X-RateLimit-Reset", HeaderValue::from_str(&reset_in.to_string()).unwrap());

    if !allowed {
        headers.insert("Retry-After", HeaderValue::from_str(&retry_after.to_string()).unwrap());
        return (
            StatusCode::TOO_MANY_REQUESTS,
            headers,
            Json(json!({"error": {"code": "RATE_LIMITED", "message": "เรียก API ถี่เกินกำหนด กรุณาลองใหม่ภายหลัง"}})),
        ).into_response();
    }

    (StatusCode::OK, headers, Json(json!({"data": {"message": "ok"}, "meta": {"request_id": "req-rl-1"}}))).into_response()
}
```

สังเกตว่า `{ let mut bucket = state.rate_bucket.lock().unwrap(); bucket.try_consume() }` ใส่อยู่ในบล็อกแยก
— นี่คือเทคนิคที่ทำให้ `MutexGuard` ถูก **`drop` ทันทีที่ statement จบ** (จบ scope ของบล็อกนั้น) ก่อนที่โค้ด
ส่วนที่เหลือ (การสร้าง header/response) จะรันต่อ — ทำให้ lock ของ `rate_bucket` ถูกถือไว้สั้นที่สุดเท่าที่
จำเป็น (แค่ตอนเรียก `try_consume()` เท่านั้น) ซึ่งสำคัญมากสำหรับ endpoint ที่ถูกเรียกถี่ (rate-limited
endpoint โดยธรรมชาติของมันเองถูกออกแบบมาให้ต้องรับ request ถี่) — ยิ่ง lock นานเท่าไร thread อื่นที่ถูก
rate limit เช็คพร้อมกันต้องรอนานขึ้นเท่านั้น

#### รันจริง: ยิง 7 ครั้งติดกัน (Capacity 5, Refill 1/วินาที)

```text
=== request 1 === HTTP/1.1 200 OK | x-ratelimit-remaining: 4 | x-ratelimit-reset: 1
=== request 2 === HTTP/1.1 200 OK | x-ratelimit-remaining: 3 | x-ratelimit-reset: 2
=== request 3 === HTTP/1.1 200 OK | x-ratelimit-remaining: 2 | x-ratelimit-reset: 3
=== request 4 === HTTP/1.1 200 OK | x-ratelimit-remaining: 1 | x-ratelimit-reset: 4
=== request 5 === HTTP/1.1 200 OK | x-ratelimit-remaining: 0 | x-ratelimit-reset: 5
=== request 6 === HTTP/1.1 429 Too Many Requests | x-ratelimit-remaining: 0 | retry-after: 1
    body: {"error":{"code":"RATE_LIMITED","message":"เรียก API ถี่เกินกำหนด กรุณาลองใหม่ภายหลัง"}}
=== request 7 === HTTP/1.1 429 Too Many Requests | x-ratelimit-remaining: 0 | retry-after: 1
    body: {"error":{"code":"RATE_LIMITED","message":"เรียก API ถี่เกินกำหนด กรุณาลองใหม่ภายหลัง"}}
```

(ผลลัพธ์ข้างบนคือ header/body จริงจากการรัน `curl` เจ็ดครั้งติดกันแบบไม่หยุดพัก บันทึกแบบย่อให้อ่านง่าย —
`x-ratelimit-limit: 5` ปรากฏครบทุก response แต่ตัดออกจากตารางเพื่อความกระชับ) request ที่ 1-5 ผ่านหมด
(token bucket มี capacity 5 พอดี) `remaining` ลดลงทีละ 1 ตามลำดับ ถูกต้องตามที่ token ถูก "จ่าย" ไปทีละ 1 ต่อ
request — request ที่ 6-7 ถูกปฏิเสธด้วย `429` พร้อม `Retry-After: 1` (เพราะ refill 1 token/วินาที การรอ 1
วินาทีก็จะมี token ใหม่มาให้ใช้พอดี) — นี่คือ contract แบบสมบูรณ์ที่ frontend เอาไปเขียนโค้ด "รอตาม
`Retry-After` แล้ว retry อัตโนมัติ" หรือ "โชว์ progress bar จาก `X-RateLimit-Remaining`/`X-RateLimit-Limit`"
ได้เลยทันที โดยไม่ต้องรู้เลยว่าหลังบ้านเป็น in-memory token bucket หรือ Redis-backed sliding window

#### ทางไปสู่ Production: ทำไมต้องเป็น Middleware ไม่ใช่โค้ดในทุก Handler

ตัวอย่างข้างบนเขียน rate limiting logic อยู่**ใน handler ตรง ๆ** เพื่อความชัดเจนในการสอน — ในระบบจริงที่มี
หลาย endpoint ต้องมี rate limit เหมือนกัน (ซึ่งมักจะเป็นแทบทุก endpoint) **การเขียนซ้ำในทุก handler คือ
สัญญาณของ cross-cutting concern** ที่ Part 65 สอนไว้แล้วว่าควรทำเป็น **`tower::Layer` middleware** ครอบทุก
route ไว้ครั้งเดียว — โครงสร้าง `TokenBucket` และ logic การตัดสิน `allowed`/header ทั้งหมดที่เขียนในหัวข้อนี้
**ย้ายเข้า middleware ได้ตรง ๆ โดยไม่ต้องเปลี่ยนตรรกะเลย** เปลี่ยนแค่ตำแหน่งที่โค้ดรัน (จาก "ในตัว handler"
ไปเป็น "ก่อนถึง handler" ผ่าน `axum::middleware::from_fn`) — และเมื่อถึง **Part 83** ที่สอน Redis, ส่วนที่
เปลี่ยนคือ `state.rate_bucket: Arc<Mutex<TokenBucket>>` (ต่อ process เดียว จำกัดแค่ instance เดียว) กลายเป็น
Redis-backed counter ที่ **แชร์ quota ข้ามทุก instance ของ server พร้อมกัน** ซึ่งจำเป็นมากสำหรับระบบที่ scale
แนวนอน (ถ้ามี 3 instance และแต่ละ instance มี token bucket ของตัวเอง แปลว่า client หนึ่งตัวเรียกได้จริง 3
เท่าของ limit ที่ตั้งใจไว้ ถ้า request ของเขากระจายไปตกคนละ instance แบบสุ่ม)

### 78.10 เอกสาร API เป็นวินัยของการออกแบบ ไม่ใช่งานที่ทำทีหลังสุด

ทุกหัวข้อที่ผ่านมาในบทนี้ (การตั้งชื่อ resource, versioning, envelope, pagination, filter/sort, idempotency,
HATEOAS, rate limiting) มีสิ่งหนึ่งที่เหมือนกัน: **ทุกอย่างคือ "สัญญา" (contract) ระหว่าง server กับ client**
— และสัญญาที่ไม่ได้เขียนไว้ที่ไหนเลยคือสัญญาที่**พังง่ายที่สุด** เพราะไม่มีใครรู้แน่ชัดว่าสัญญานั้นคืออะไรกัน
แน่นอกจากคนที่เขียนโค้ดคนนั้นคนเดียว (และแม้แต่คนนั้นเองก็อาจจำรายละเอียดผิดหลังจากผ่านไปหกเดือน)

**ความเข้าใจผิดที่พบบ่อยที่สุด**: มองว่า "เอกสาร API" คือสิ่งที่เขียน**หลังจาก**โค้ด API เสร็จสมบูรณ์แล้ว (มัก
เป็นงานที่ถูกผัดไปจนถึงวันก่อน deploy หรือแย่กว่านั้นคือไม่มีเวลาเขียนเลย) — วินัยที่ทีมมืออาชีพใช้กันคือ
**คิดเรื่องเอกสารไปพร้อมกับการออกแบบ** ไม่ใช่ทำทีหลัง เหตุผลเชิงลึก: **การพยายามเขียนเอกสารของ endpoint หนึ่ง
ให้ชัดเจนคือกระบวนการเดียวกับการตรวจสอบว่าออกแบบ endpoint นั้นดีพอหรือยัง** — ถ้าคุณเขียนเอกสารของ endpoint
หนึ่งแล้วต้องอธิบายเงื่อนไขประหลาด ๆ เยอะมาก ("ปกติ status นี้คือ `200` แต่ถ้าเงื่อนไข X จะเป็น `202` แทน
และถ้า Y จะไม่มี field `data` เลยแต่มี field `partial_data` ขึ้นมาแทน") — นั่นคือสัญญาณว่า**การออกแบบมีความ
ไม่สม่ำเสมอ** ที่ควรแก้ไขตั้งแต่ตอนออกแบบ ไม่ใช่แค่จดไว้ในเอกสารแล้วปล่อยไป

องค์ประกอบที่ **ต้อง**เขียนไว้ชัดเจนสำหรับทุก endpoint (ไม่ว่าจะเขียนในรูปแบบไหนก็ตาม — comment ในโค้ด, ไฟล์
markdown แยก, หรือ OpenAPI spec):

1. **Request shape**: method, path (พร้อม path/query parameter ที่รองรับทั้งหมดตามหัวข้อ 78.2/78.6), body
   shape (ถ้ามี) พร้อมว่า field ไหน required/optional
2. **Response shape สำหรับกรณีสำเร็จ**: status code ที่เป็นไปได้ (ปกติควรมีแค่ 1-2 ตัวต่อ endpoint ไม่ใช่
   4-5 ตัวที่สื่อว่า logic ซับซ้อนเกินจำเป็น) และ envelope shape ตามหัวข้อ 78.4
3. **Response shape สำหรับทุกกรณี error ที่เป็นไปได้**: `404` เกิดเมื่อไร, `409` เกิดเมื่อไร, `422` ต่างจาก
   `400` ตรงไหนสำหรับ endpoint นี้โดยเฉพาะ (Part 61 หัวข้อ 61.3 อธิบายหลักการทั่วไปไว้แล้ว แต่แต่ละ endpoint
   มีรายละเอียดเฉพาะของตัวเองที่ต้องระบุ)
4. **Side effect ที่ไม่ชัดเจนจาก signature เพียงอย่างเดียว**: เช่น "endpoint นี้ส่งอีเมลแจ้งเตือนด้วย",
   "endpoint นี้ idempotent ผ่าน `Idempotency-Key` header (หัวข้อ 78.7)"

**บทนี้จงใจไม่ลงรายละเอียดเรื่อง tooling** (การเขียน spec ด้วย YAML/JSON แบบ OpenAPI, การ generate เอกสาร
สวย ๆ อัตโนมัติจากโค้ด) เพราะ **Part 85 (API Documentation ด้วย OpenAPI/Swagger ผ่าน `utoipa`)** ในโมดูลนี้
จะสอนเรื่องนี้แบบเต็มรูปแบบ — สิ่งที่บทนี้ต้องการปูพื้นไว้ก่อนคือ**วินัยการคิด** ("ทุก endpoint ต้องตอบคำถาม
4 ข้อข้างบนได้ชัดเจนตั้งแต่ตอนออกแบบ") ซึ่งเป็นเงื่อนไขที่ทำให้ tooling อย่าง `utoipa` **ใช้งานได้ผลจริง** —
ถ้าการออกแบบไม่สม่ำเสมอมาตั้งแต่แรก ไม่มี tooling ใดที่ generate เอกสารสวย ๆ ออกมาแล้วช่วยได้ เพราะเอกสารที่
ได้ก็จะสวยแต่**สื่อความสับสนแบบเดียวกับที่โค้ดมี**ออกมาให้ผู้อ่านเห็นชัดขึ้นเท่านั้นเอง

### 78.11 Breaking vs Non-Breaking Changes: Checklist และ Deprecation Header

#### นิยาม: "Breaking" วัดจากมุมมองของ Client เท่านั้น

หลักที่สำคัญที่สุดของหัวข้อนี้คือ: **"breaking change" ไม่ได้วัดจากว่าโค้ด server เปลี่ยนมากแค่ไหน แต่วัดจาก
ว่า client ที่เขียนโค้ดไว้แล้วตาม response shape เดิม จะพัง (throw error, parse ผิด, หรือได้ผลลัพธ์ที่ไม่ตรง
กับที่ตั้งใจ) หรือไม่** — server อาจ refactor ภายในทั้งหมด (เปลี่ยนจาก raw SQL เป็น SeaORM ตาม Part 73,
เปลี่ยน algorithm การคำนวณ) โดยไม่เป็น breaking change เลยตราบใดที่ **response shape ที่ client เห็นเหมือนกัน
เป๊ะ** — ในทางกลับกัน การเปลี่ยน field เดียวใน JSON response ก็อาจเป็น breaking change รุนแรงได้โดยไม่ต้อง
แก้โค้ด business logic เลยแม้แต่บรรทัดเดียว

#### Checklist: อะไรคือ Breaking Change

| การเปลี่ยนแปลง | Breaking? | เหตุผล |
|---|---|---|
| ลบ field ออกจาก response | ✅ Breaking | client ที่ใช้ field นั้นอยู่จะได้ `undefined`/error ทันที |
| เปลี่ยนชื่อ field (`name` → `full_name`) | ✅ Breaking | เทียบเท่าลบ field เดิม + เพิ่ม field ใหม่พร้อมกัน — client เดิมมองไม่เห็น field ที่ต้องการอีกต่อไป |
| เปลี่ยนชนิดข้อมูลของ field (`price: "100"` string → `price: 100` number) | ✅ Breaking | client ที่ทำ `.toUpperCase()` บน string หรือ `parseInt()` คาดหวัง string จะพัง แม้ค่าที่สื่อความหมายจะเหมือนกัน |
| เปลี่ยน field จาก required เป็น "อาจไม่มี" (nullable ใหม่) | ✅ Breaking | client ที่ไม่ได้เช็ค `null`/`undefined` มาก่อน (เพราะเดิม field นี้การันตีว่ามีเสมอ) จะพังตอนเจอ `null` |
| เปลี่ยน status code ของสถานการณ์เดิมที่มีอยู่แล้ว (`404` → `200` พร้อม `data: null`) | ✅ Breaking | client ที่เขียน logic แยกตาม status code (Part 61 หัวข้อ 61.3) จะตีความสถานการณ์ผิดทันที |
| เปลี่ยนความหมายของ field โดยไม่เปลี่ยนชื่อ/ชนิด (เช่น `status: "open"` เดิมหมายถึง "ว่างให้ยืม" เปลี่ยนไปหมายถึง "เปิดให้จอง") | ✅ Breaking (อันตรายที่สุดเพราะตรวจจับไม่ได้ด้วย schema validation) | client เก่ายัง parse ผ่านได้ปกติทุกประการ แต่**ตีความข้อมูลผิดโดยไม่มี error ใดๆเตือน** |
| เพิ่ม field ใหม่เข้าไปใน response (ไม่ลบ/เปลี่ยนของเดิม) | ❌ Non-breaking | client ที่ deserialize แบบ "รู้จัก field ที่คาดไว้แล้วมองข้าม field ที่ไม่รู้จัก" (พฤติกรรม default ของ `#[derive(Deserialize)]` ใน serde ที่ Part 57 สอนไว้ — ไม่มี `#[serde(deny_unknown_fields)]`) จะไม่ได้รับผลกระทบเลย |
| เพิ่ม endpoint ใหม่ | ❌ Non-breaking | ไม่กระทบ endpoint เดิมที่มีอยู่แล้วเลย |
| เพิ่ม query parameter ใหม่ที่เป็น optional | ❌ Non-breaking | client เดิมที่ไม่ส่ง parameter นี้มาจะได้ค่า default ตามที่ endpoint กำหนด (เช่นเดียวกับที่หัวข้อ 78.6 ออกแบบ `status`/`sort`/`fields` ให้เป็น `Option<T>` ทั้งหมด) |
| เปลี่ยนลำดับของ field ใน JSON object | ❌ Non-breaking (ในทางทฤษฎี) | JSON object ไม่มีลำดับที่มีความหมายทางความหมาย (unordered map) — client ที่ดีไม่ควร parse ตามตำแหน่ง อย่างไรก็ตามถ้า client บางตัวเขียนแบบ fragile (parse ด้วย regex บน raw string แทน JSON parser จริง) อาจพังได้ ซึ่งเป็นปัญหาของ client ไม่ใช่ของ server |
| ทำให้ endpoint ช้าลงอย่างมีนัยสำคัญ | ⚠️ ไม่ใช่ breaking ในทาง data shape แต่เป็น breaking ในทาง **SLA** ถ้ามี timeout อยู่ฝั่ง client | สำคัญพอที่ควรนับรวมอยู่ในการพิจารณา "ผลกระทบต่อ client" แม้ response shape จะเหมือนเดิมทุกประการ |

แถวที่อันตรายที่สุดในตารางคือแถว **"เปลี่ยนความหมายของ field โดยไม่เปลี่ยนชื่อ/ชนิด"** — เพราะมันผ่าน schema
validation ทุกชนิดได้สบาย ๆ (ชนิดข้อมูลเหมือนเดิม, ชื่อ field เหมือนเดิม) แต่ทำให้ client ตีความข้อมูลผิดโดย
เงียบ ๆ — นี่คือเหตุผลที่การตั้งชื่อ field ตั้งแต่แรก (หัวข้อ 78.2 พูดถึง resource แต่หลักการเดียวกันใช้กับชื่อ
field ได้) ต้อง**สื่อความหมายชัดเจนไม่กำกวม** ตั้งแต่ครั้งแรกที่ออกแบบ เพราะการแก้ไขความหมายทีหลังโดยไม่แก้
ชื่อคือการเปลี่ยนแปลงที่มองไม่เห็นด้วยเครื่องมือใด ๆ เลย

#### เมื่อจำเป็นต้อง Breaking Change จริง ๆ: Versioning (78.3) + Deprecation Header (RFC 8594)

บางครั้ง breaking change เป็นสิ่งที่หลีกเลี่ยงไม่ได้จริง ๆ (แก้ bug เชิงความหมายที่ฝังมานาน, ปรับ data model
ให้ถูกต้องกว่าเดิม) — เส้นทางที่ปลอดภัยคือ **ไม่แก้ของเดิมตรง ๆ แต่เปิดเวอร์ชันใหม่** ตามที่หัวข้อ 78.3 สอนไว้
(`/api/v2/...`) แล้ว **ประกาศเลิกใช้เวอร์ชันเก่าอย่างมีขั้นตอน** ไม่ใช่ปิดกะทันหัน

**RFC 8594** นิยาม header สองตัวสำหรับสื่อสารเรื่องนี้กับ client โดยตรงผ่าน HTTP (ไม่ต้องพึ่งอีเมลประกาศหรือ
เอกสารแยกอย่างเดียว):

- **`Deprecation`**: บอกว่า endpoint/เวอร์ชันนี้ **ถูก deprecate แล้ว** ค่าเป็น `true` หรือเป็น HTTP-date
  บอกว่า deprecate ตั้งแต่เมื่อไร
- **`Sunset`**: บอกว่า endpoint นี้ **จะหยุดทำงานจริง** ณ วันที่ระบุ (HTTP-date format เดียวกับที่ใช้ใน
  `Date`/`Expires` header ทั่วไป) — ต่างจาก `Deprecation` ตรงที่ `Sunset` คือ**คำมั่นสัญญาเรื่องเวลาที่แน่นอน**
  ว่าจะปิดจริง ไม่ใช่แค่ "ไม่แนะนำให้ใช้แล้ว"

```text
GET /api/v1/books/4821 HTTP/1.1

HTTP/1.1 200 OK
Content-Type: application/json
Deprecation: true
Sunset: Sat, 01 Aug 2026 00:00:00 GMT
Link: <https://api.example.com/api/v2/books/4821>; rel="successor-version"

{"data": {...}}
```

`Link` header กลับมาอีกครั้ง (คราวนี้ใช้ `rel="successor-version"` แทน `rel="next"`/`rel="prev"` ของหัวข้อ
78.5 — RFC 8288 นิยาม relation type ไว้หลายแบบ ไม่ได้จำกัดแค่ pagination) ชี้ตรงไปที่เวอร์ชันใหม่ที่ควรย้าย
ไปใช้ — client ที่เขียนโค้ดตรวจสอบ header เหล่านี้อัตโนมัติสามารถ**เตือน developer ของตัวเองในโค้ด**หรือ
**log warning ไว้** ให้ทีมที่ดูแล client รู้ตัวล่วงหน้านานพอที่จะ migrate ก่อนวันที่ `Sunset` มาถึงจริง — นี่
คือความแตกต่างที่สำคัญที่สุดระหว่าง "การปิด API กะทันหันแล้วให้ client พังไปพร้อมกันตอนนั้น" กับ "การให้เวลา
migrate อย่างมีขั้นตอนที่สื่อสารผ่าน HTTP header ที่ tooling อัตโนมัติตรวจจับได้"

**ขั้นตอนที่แนะนำเมื่อต้องทำ breaking change**: (1) เปิด endpoint/เวอร์ชันใหม่ที่มี shape ที่ถูกต้อง (2) ใส่
`Deprecation`/`Sunset`/`Link` header บนเวอร์ชันเก่า พร้อมประกาศในเอกสาร (หัวข้อ 78.10) (3) ให้เวลา migrate
นานพอสมควร (มักวัดเป็นเดือนสำหรับ public API ไม่ใช่วัน) (4) ปิดเวอร์ชันเก่าจริงเมื่อถึงวันที่ `Sunset` ที่
ประกาศไว้ — ไม่เร็วกว่านั้น เพราะนั่นคือการผิดสัญญาที่เพิ่งประกาศไปเอง

### 78.12 สังเคราะห์ทุกหลักการเป็น API Design Review Checklist เดียว

ก่อนเขียน handler ตัวแรกของ endpoint ใหม่ทุกครั้ง (หรือก่อน merge PR ที่เพิ่ม/เปลี่ยน endpoint) ให้ตอบ
คำถามต่อไปนี้ให้ครบ — checklist นี้คือการรวมทุกหัวข้อของบทนี้เข้าเป็นชุดคำถามเดียวที่ใช้ตรวจสอบตัวเองได้จริง
ในการทำงานทุกวัน:

1. **[78.2] ชื่อ resource เป็นคำนามพหูพจน์ ไม่มีคำกริยาปนใน path หรือไม่?** ถ้ามีความสัมพันธ์กับ resource
   อื่น ตอบสามคำถาม (ตัวตนนอกบริบท parent / cascade delete ตามธรรมชาติ / reassignment ได้ไหม) แล้วเลือก
   nest หรือ flat+filter ให้ตรงกับคำตอบ
2. **[78.3] เวอร์ชันของ endpoint นี้ชัดเจนหรือไม่?** อยู่ใต้ `/api/v{n}/...` ตาม convention ที่ทั้งระบบใช้
   สม่ำเสมอหรือไม่
3. **[78.4] Response ใช้ envelope เดียวกันกับ endpoint อื่นในระบบหรือไม่?** success มี `data`/`meta` ตรง
   ตามที่ตกลงกันไว้, error ใช้ `AppError` ของ Part 66 ที่ขยายแล้วหรือไม่
4. **[78.5] ถ้าเป็น list endpoint — pagination ใช้ pattern เดียวกับที่เหลือของระบบหรือไม่?** มี `Link`
   header สำหรับ `rel="next"`/`rel="prev"` ครบไหม, เลือก offset หรือ keyset ถูกกับขนาดข้อมูลที่คาดไว้ไหม
   (ตาม Part 71)
5. **[78.6] มี filter/sort/sparse-fieldset ที่จำเป็นครบไหม และตั้งชื่อ query param ตาม convention เดียวกัน
   กับ endpoint อื่นหรือไม่?** (`sort=-field` ไม่ใช่ `sortDesc=field` แบบสุ่มคิดเอง)
6. **[78.7] ถ้าเป็น `POST` ที่สร้าง resource ที่มีต้นทุนสูงถ้าซ้ำ (หักเงิน, จอง, ส่งอีเมล) — รองรับ
   `Idempotency-Key` หรือยัง?**
7. **[78.8] มี `_links` ที่ช่วย client จริงหรือไม่ (ไม่ใช่ HATEOAS เต็มรูปแบบที่ไม่มีใครใช้)?** โดยเฉพาะ
   action ที่มีเงื่อนไขทางธุรกิจว่า "ทำได้ไหม" ซึ่งควรให้ server บอก ไม่ใช่ให้ client เดา
8. **[78.9] ถ้า endpoint นี้เสี่ยงถูกเรียกถี่เกิน (public API, endpoint ที่ทำงานหนัก) — มี rate limit
   contract (`429`/`Retry-After`/`X-RateLimit-*`) กำหนดไว้แล้วหรือยัง แม้ implementation จริงจะยังไม่
   distributed?**
9. **[78.10] เอกสารของ endpoint นี้ตอบ 4 คำถามหลัก (request shape / success response / ทุก error case /
   side effect ที่ซ่อนอยู่) ได้ครบหรือยัง?**
10. **[78.11] ถ้าเป็นการแก้ endpoint ที่มีอยู่แล้ว — ผ่าน breaking-change checklist หรือไม่?** ถ้า breaking
    จริง มีแผนเปิดเวอร์ชันใหม่ + ใส่ `Deprecation`/`Sunset` บนเวอร์ชันเก่าหรือยัง?

ถ้าตอบทุกข้อได้อย่างมั่นใจ — endpoint นั้นพร้อมสำหรับการใช้งานจริงในระดับที่ทีมอื่นเอาไปพึ่งพาได้อย่างมั่นใจ
ในระยะยาว

## กับดักที่พบบ่อย (Common Pitfalls)

1. **ผสม bare และ wrapped response ในระบบเดียวกัน**: endpoint บางตัวคืน `[...]` ตรง ๆ บางตัวคืน
   `{"data": [...]}` — เกิดขึ้นบ่อยมากเมื่อทีมโตขึ้นและมีคนเขียน endpoint ใหม่โดยไม่รู้ convention เดิม หรือ
   endpoint เก่าถูกเขียนไว้ก่อนที่ทีมจะตกลง convention กัน วิธีป้องกัน: กำหนด `Envelope<T>` (หัวข้อ 78.4)
   เป็น**จุดเดียว**ที่ทุก handler ต้องใช้คืนค่า success (คล้ายกับที่ `AppError`/`IntoResponse` เป็นจุดเดียว
   สำหรับ error ตาม Part 66) — ถ้ามี handler ที่ไม่ผ่าน `Envelope<T>` ควรถูกจับได้ตอน code review ทันที
   ไม่ใช่ปล่อยให้หลุดไปจนถึง production แล้วค่อยแก้

2. **ลืมว่า `Query<T>` ของ Axum deserialize field เป็น `String` เดี่ยว ไม่ได้ split comma ให้อัตโนมัติ**:
   นักพัฒนาที่เพิ่งเขียน sort mini-DSL ครั้งแรกมักคาดหวังว่า `sort: Vec<String>` จะ parse
   `?sort=-created_at,title` ให้เป็น `vec!["-created_at", "title"]` ให้อัตโนมัติ — ความจริงคือ `Query<T>`
   ใช้ `serde_urlencoded` ซึ่ง**ไม่รู้จัก comma-separated list โดย default** (มันรองรับ `?sort=a&sort=b`
   แบบ repeated key มากกว่า) ถ้าประกาศ `sort: Option<Vec<String>>` แล้วยิง `?sort=-created_at,title` จะได้
   error แบบนี้จริง (ทดสอบแล้ว):

   ```text
   Failed to deserialize query string: sort: invalid type: string "-created_at,title", expected a sequence
   ```

   วิธีแก้ที่บทนี้ใช้: **ประกาศ field เป็น `sort: Option<String>` (string เดี่ยว) แล้ว split comma เองในโค้ด
   handler** ด้วย `parse_sort()` ตามหัวข้อ 78.6 — ไม่ต้องพึ่ง `serde` ให้ทำ parsing ที่ซับซ้อนกว่ารูปแบบ
   key-value ธรรมดาให้

3. **Sort mini-DSL เงียบ ๆ เพิกเฉยต่อ field ที่ไม่รู้จักแทนที่จะ error**: โค้ดตัวอย่างในหัวข้อ 78.6 ใช้
   `_ => Ordering::Equal` เมื่อเจอ field ที่ไม่รู้จักใน `?sort=`  — behavior นี้**เจตนา**เพื่อความง่ายของ
   ตัวอย่าง แต่ในระบบจริงเป็นกับดักที่อันตราย: ถ้า client พิมพ์ผิด (`?sort=-createdAt` แทน `?sort=-created_at`)
   ระบบจะ**เงียบ ๆ ไม่เรียงตามที่ขอเลยแต่ก็ไม่ error บอกอะไร** — client จะสงสัยว่า "ทำไม sort ไม่ทำงาน" โดย
   ไม่มี error message ช่วยชี้ทาง วิธีแก้ที่ดีกว่าสำหรับ production: ให้ `_ =>` ในโค้ด match คืน
   `Err(AppError::Validation(format!("ไม่รู้จัก sort field: {}", k.field)))` แทน แล้วตอบ `400`/`422` กลับไป
   ให้ client รู้ตัวทันทีว่าพิมพ์ field ผิด

4. **สับสนระหว่าง `Deprecation` header กับการลบ endpoint ทันที**: ทีมที่เพิ่งรู้จัก RFC 8594 บางครั้งเข้าใจ
   ผิดว่าการใส่ `Deprecation: true` **คือ**การปิดใช้งาน — ความจริงคือ `Deprecation` เป็นแค่**การสื่อสาร**
   endpoint ยังต้องทำงานได้ปกติทุกประการต่อไปจนกว่าจะถึงวันที่ระบุใน `Sunset` — การปิด endpoint จริงก่อนถึง
   วันที่ประกาศไว้ (แม้จะใส่ header เตือนไว้ล่วงหน้าแล้วก็ตาม) คือการผิดสัญญาที่ประกาศไปเอง และทำให้ client
   ที่วางแผน migrate ตามกำหนดการที่ประกาศไว้พังกะทันหันโดยไม่มีเหตุผล

5. **ใช้ `HashMap` เดียวกันสำหรับ idempotency store ข้ามหลาย endpoint ที่มี business logic ต่างกัน โดยไม่
   namespace key**: ถ้า endpoint `POST /bookings` และ `POST /payments` ใช้ `idempotency_store` ตัวเดียวกัน
   และ client (โดยบังเอิญหรือโดยตั้งใจ generate key แบบเดาได้) ส่ง `Idempotency-Key` เดียวกันไปสอง endpoint
   — ระบบในหัวข้อ 78.7 (ที่ตรวจ hash ของ body ด้วย) จะจับได้ว่า body ต่างกัน (เพราะ endpoint ต่างกัน body
   ก็ต่างกันตามธรรมชาติ) แต่ในระบบที่ไม่ได้ตรวจ hash แบบนี้ ความเสี่ยงคือ**key ชนกันข้าม endpoint แล้วคืน
   response ของ endpoint ผิดกลับไป** วิธีแก้ที่ปลอดภัยกว่า: namespace key ด้วยชื่อ endpoint เสมอ
   (`format!("{}:{}", endpoint_name, idempotency_key)`) แทนการใช้ key ดิบจาก client ตรง ๆ เป็น key ของ store

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** ระบบห้องสมุดมี resource `authors` และ `books` — ผู้เขียนคนหนึ่งเขียนหนังสือได้หลายเล่ม และ
   หนังสือเล่มหนึ่งมีผู้เขียนได้หลายคน (many-to-many) ใช้กรอบการตัดสินใจในหัวข้อ 78.2 (3 คำถาม) วิเคราะห์ว่า
   ควรออกแบบเป็น `/authors/{id}/books` (nested) หรือ `/books?author_id={id}` (flat+filter) — **hint**: ตอบ
   คำถามที่ 3 ก่อน (reassignment/many-to-many) จะเห็นคำตอบชัดที่สุด เพราะความสัมพันธ์แบบ many-to-many เป็น
   สัญญาณที่ชัดมากว่าไม่ควร nest แบบใดแบบหนึ่งเป็นเจ้าของอีกฝั่งตรง ๆ

2. **(กลาง)** ขยาย `Envelope<T>` จากหัวข้อ 78.4 ให้มี field `warnings: Vec<String>` เพิ่มเติมใน `Meta`
   (optional, skip เมื่อ empty) สำหรับกรณีที่ request สำเร็จแต่มีบางอย่างที่ client ควรรู้ (เช่น "ใช้ค่า
   default ของ `limit` เพราะค่าที่ส่งมาเกิน max ที่อนุญาต") แล้วแก้ `list_books` handler ในหัวข้อ 78.5 ให้
   เติม warning เมื่อ `limit` ที่ client ขอเกิน 100 (ถูก `.clamp(1, 100)` ปรับลงมา) — **hint**: ต้องเก็บค่า
   `limit` ดิบที่ client ส่งมาไว้ก่อน `.clamp()` เพื่อเทียบว่ามันถูกปรับหรือไม่

3. **(ยาก)** เขียน middleware (`tower::Layer` ตามที่ Part 65 สอน) ที่ครอบ rate limiting logic จากหัวข้อ
   78.9 ให้ใช้ได้กับทุก route โดยไม่ต้องเขียนโค้ดซ้ำในทุก handler — middleware ต้องดึง client identifier
   (เช่น IP address จาก `ConnectInfo<SocketAddr>` หรือ API key จาก header) มาแยก token bucket ต่อ client
   แต่ละคน (ไม่ใช่ bucket เดียวที่แชร์กันทุกคนแบบตัวอย่างในบทนี้) — **hint**: ต้องเปลี่ยน
   `rate_bucket: Arc<Mutex<TokenBucket>>` เป็น `Arc<Mutex<HashMap<String, TokenBucket>>>` ที่ key คือ client
   identifier แล้วสร้าง `TokenBucket` ใหม่ (ด้วยค่า default) ตอนเจอ client ใหม่ครั้งแรก คิดเพิ่มว่าจะทำ
   cleanup ของ client ที่ไม่ active แล้วอย่างไรไม่ให้ `HashMap` โตไม่มีที่สิ้นสุด (คำใบ้: มี TTL คล้ายกับที่
   Idempotency-Key store ต้องมี)

4. **(ยาก/ประยุกต์ใช้งานจริง)** ระบบจองตั๋วในหัวข้อ 78.8 มีกฎธุรกิจว่า "ยกเลิกได้เฉพาะตอน `status ==
   confirmed`" — เพิ่มกฎใหม่: "ยกเลิกได้เฉพาะเมื่อยังเหลือเวลามากกว่า 24 ชั่วโมงก่อนงานเริ่ม" (ต้องเพิ่ม field
   `event_starts_at` ให้ `Booking`) แก้ `booking_to_json` ให้เงื่อนไข `_links.cancel` ครอบคลุมทั้งสองกฎ แล้ว
   ตอบคำถามนี้ในเอกสารของ endpoint (ตามวินัยหัวข้อ 78.10): ถ้า client เรียก `POST
   /bookings/{id}/cancel` ตอนที่เหลือเวลาน้อยกว่า 24 ชั่วโมง (ซึ่ง `_links.cancel` ไม่ได้แสดงให้เห็นอยู่แล้ว)
   ควรตอบ status code อะไร (`403`? `409`? `422`?) พร้อมให้เหตุผลอ้างอิงหลักการเลือก status code จาก Part 61
   หัวข้อ 61.3 — **hint**: คำถามสำคัญคือ "นี่คือเรื่องสิทธิ์ (authorization) หรือเรื่องสถานะของ resource ที่
   ขัดกับ request (state conflict)?" — คำตอบนำไปสู่การเลือกระหว่าง `403` กับ `409` ได้ตรงประเด็น ลองพิจารณา
   ว่าถ้า client คนเดียวกันเรียกตอนที่ยังเหลือเวลาพอ จะได้ผลต่างกันไหม เพื่อแยกว่าเป็นเรื่อง "สิทธิ์ของ
   ผู้ใช้คนนี้" หรือ "เงื่อนไขของเวลา ณ ขณะนี้"

## สรุป

บทนี้ไม่ได้สอน syntax ใหม่ของ Rust หรือ crate ใหม่แม้แต่ตัวเดียว — สิ่งที่สอนคือ**วินัยการออกแบบ** ที่แปลง
API ที่ "ทำงานได้" (ซึ่งคุณสร้างได้มาตั้งแต่ Part 62-77) ให้กลายเป็น API ที่ **คาดเดาได้, วิวัฒน์ได้โดยไม่พัง
ของเดิม, และใช้งานสบายสำหรับนักพัฒนาคนอื่น** — ทบทวนสิ่งที่ทำไปทั้งหมด:

- **การตั้งชื่อ resource** (78.2) ใช้ noun พหูพจน์เสมอ และเลือก nest URL หรือ flat+filter ด้วยกรอบ 3
  คำถาม (ตัวตนนอกบริบท parent / cascade ตามธรรมชาติ / reassignment ได้ไหม) ไม่ใช่ตามความเคยชิน
- **Versioning** (78.3) มีสามแนวทาง (URL/header/query-param) แต่ละแบบมี trade-off ชัดเจน — URL versioning
  คือ default ที่ปฏิบัติได้ง่ายที่สุดสำหรับทีมส่วนใหญ่ ตามที่หลักสูตรนี้ใช้มาตั้งแต่ Part 61
- **Response envelope** (78.4) ที่ wrap ด้วย `data`/`meta`/`error` อย่างสม่ำเสมอ ต่อยอดจาก `AppError` ของ
  Part 66 ให้กลายเป็น convention เต็มระบบที่เผื่อที่สำหรับ metadata ในอนาคตโดยไม่ต้องทำ breaking change
- **Pagination** (78.5) ผสาน keyset pagination ของ Part 71 กับ `Link` header ตาม RFC 8288 ทำให้
  implementation detail ของการ paginate ถูกซ่อนไว้หลัง URL ทั้งหมด
- **Filter/sort/sparse fieldsets** (78.6) ใช้ `Query<T>` จาก Part 63 พร้อม mini-DSL ที่ parse เองในโค้ด
  handler เพราะ `serde_urlencoded` ไม่รองรับ comma-separated list โดย default
- **Idempotency-Key** (78.7) implement จริงด้วย Axum ต่อยอดจาก simulation ของ Part 61 — เก็บทั้ง response
  เต็มและ hash ของ body เพื่อตรวจจับการใช้ key ซ้ำผิดที่ ก่อนจะย้ายไป Redis ใน Part 83
- **HATEOAS เลือกสรร** (78.8) ใส่ `_links` เฉพาะที่ย้าย business rule กลับไปที่ server จุดเดียว ไม่ใช่ full
  hypermedia navigation ตามอุดมคติที่ Part 61 บอกว่า API จริงส่วนใหญ่ไม่ทำ
- **Rate limiting contract** (78.9) กำหนด `429`/`Retry-After`/`X-RateLimit-*` ที่ frontend ใช้ได้ตั้งแต่
  วันนี้ แม้ implementation จริงยังเป็น in-memory จนกว่าจะถึง Part 83
- **เอกสารเป็นวินัยการออกแบบ** (78.10) ไม่ใช่งานที่ทำทีหลังสุด — foreshadow เครื่องมืออัตโนมัติใน Part 85
- **Breaking vs non-breaking changes** (78.11) มี checklist ที่เป็นรูปธรรม และ `Deprecation`/`Sunset`
  header (RFC 8594) เป็นเส้นทางที่ปลอดภัยเมื่อ breaking change เป็นสิ่งที่หลีกเลี่ยงไม่ได้จริง

Part ถัดไป (**Part 79: GraphQL ด้วย async-graphql**) จะแนะนำแนวทางที่ต่างออกไปอย่างสิ้นเชิงในการแก้ปัญหา
บางส่วนที่บทนี้พูดถึง — โดยเฉพาะปัญหา **over-fetching/under-fetching** ที่ sparse fieldsets (หัวข้อ 78.6)
พยายามแก้ด้วย query param แบบง่าย ๆ นั้น GraphQL แก้ด้วยการให้ client เขียน query ที่ระบุ field ที่ต้องการ
ได้อย่างละเอียดในทุกระดับความลึกของข้อมูลที่เกี่ยวข้องกัน (ไม่ใช่แค่ field ระดับบนสุดแบบ `?fields=id,title`)
— แต่ก็มาพร้อม trade-off ของตัวเองที่ REST ไม่มี (เช่น การ cache ที่ยากกว่ามากเพราะทุก query อาจมีรูปร่างต่าง
กัน ไม่เหมือน REST ที่ endpoint เดียวกันมี URL เดียวกันให้ HTTP cache ทำงานได้ตรงไปตรงมา) — หลักการทั้งหมดที่
บทนี้สอน (versioning, error envelope, breaking changes, documentation discipline) ยังคงนำไปใช้กับ GraphQL
API ได้เกือบทั้งหมดเช่นกัน เพียงแต่ราย mechanism จะต่างออกไป

---

**Part ก่อนหน้า:** [WebSockets ด้วย Axum](part-077-websockets-axum.md) | **Part ถัดไป:** [GraphQL ด้วย async-graphql](part-079-graphql.md)
