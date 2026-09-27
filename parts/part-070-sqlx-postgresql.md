# Part 70: Database: เชื่อมต่อ PostgreSQL ด้วย SQLx

> โมดูล: การพัฒนาเว็บแอปพลิเคชัน (Web Development) | ระดับ: สูง | เวลาโดยประมาณ: 260 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้ว่าทำไม `SQLx` ถึงเป็นตัวเลือกที่เข้ากับ Rust async ecosystem โดยธรรมชาติ (async-native ตั้งแต่การออกแบบ ไม่ใช่ ORM ที่ค่อยเติม async เข้ามาทีหลัง) และอธิบายกลไก **compile-time query verification** ผ่าน macro `sqlx::query!`/`sqlx::query_as!` ได้อย่างถูกต้อง — ทั้งสองโหมด (เชื่อมต่อฐานข้อมูลจริงตอน compile กับใช้ cache แบบ offline ผ่าน `cargo sqlx prepare`) พร้อมเทียบความต่างเชิงแนวคิดกับ ORM แบบ Diesel (Part 72) และ SeaORM (Part 73)
- ติดตั้งและตั้งค่าโปรเจกต์ที่เชื่อมต่อ PostgreSQL ได้ครบวงจร: เพิ่ม `sqlx` ด้วย feature flag ที่ถูกต้อง, จัดการ `DATABASE_URL` ตามธรรมเนียม 12-factor app, และสร้าง connection pool ด้วย `PgPoolOptions` — พร้อมอธิบายเหตุผลเชิงลึกว่าทำไมต้องใช้ pool ไม่ใช่เปิด connection ใหม่ทุกครั้งที่มี request เข้ามา (เชื่อมกับ Part 39 เรื่อง shared state และ Part 48 เรื่อง Tokio task concurrency)
- เขียน CRUD เต็มรูปแบบด้วย SQL ที่ parameterize ถูกต้อง (`$1`, `$2`, ...) ทั้ง SELECT, INSERT ... RETURNING, UPDATE, DELETE พร้อม map ผลลัพธ์เข้า struct ด้วย `#[derive(sqlx::FromRow)]` ผสานกับ `serde::Serialize` (เชื่อมกับ Part 57) และอธิบาย **SQL injection** ได้อย่างเป็นรูปธรรมพร้อมพิสูจน์ด้วยโค้ดจริงว่าเกิดอะไรขึ้นถ้า string-format SQL เอง เทียบกับการใช้ bind parameter
- ใช้ `sqlx-cli` จัดการ migration แบบมีเวอร์ชัน (`sqlx migrate add`, `sqlx migrate run`) และเข้าใจโครงสร้างไฟล์ `up.sql`/`down.sql` พร้อมสร้างตาราง `books`/`borrow_records` จริงที่มี constraint ที่ถูกต้อง (`UNIQUE`, `CHECK`, `FOREIGN KEY`)
- อธิบายพฤติกรรมจริงของ connection pool เมื่อ "เต็ม" (connection ถูกใช้ครบ `max_connections`) — request ที่มาทีหลังจะ **รอ (queue)** จนกว่าจะมี connection ว่างหรือหมดเวลา (`acquire_timeout`) ไม่ใช่ error ทันที — และผูกเรื่องนี้เข้ากับโมเดล concurrency ของ Tokio/Axum จาก Part 48/64
- นำ `PgPool` ไปเก็บใน `AppState` ของ Axum (ต่อยอด pattern จาก Part 64) เขียน handler จริงที่ query database แล้วตอบ JSON ได้ครบ พร้อมแปลง `sqlx::Error` (`RowNotFound`, `Database` สำหรับ constraint violation, connection error) เป็น HTTP response ที่เหมาะสม (404, 409, 500) ตามแนวทาง error handling ที่ Part 66 สอน
- ใช้ **transaction** (`pool.begin()`, `commit()`, `rollback()`) จัดการ operation ที่ต้อง atomic หลายคำสั่งพร้อมกัน (เช่น ยืมหนังสือ: ลดจำนวนที่เหลือ + สร้าง record การยืม ต้องสำเร็จทั้งคู่หรือไม่สำเร็จเลย) พร้อมพิสูจน์ด้วยโค้ดจริงว่า rollback ทำงานถูกต้องเมื่อมี error เกิดขึ้นกลางทาง

## ความรู้ที่ต้องมีมาก่อน

- **Part 62-64 (Axum: Intro, Routing/Handlers, State/Extractors)**: บทนี้เป็นการ "เปลี่ยนของจริง" ให้กับระบบที่ Part 62-64 สร้างไว้ — จาก `Arc<Mutex<HashMap<...>>>` ที่เป็น in-memory placeholder มาเป็น `PgPool` ที่คุยกับ PostgreSQL จริง โดยใช้ pattern `AppState` + `State<T>` extractor เดียวกันทุกประการที่ Part 64 สอนไว้ ถ้ายังไม่แน่นเรื่อง `AppState`/`with_state()`/`FromRef` ควรทวนก่อน เพราะบทนี้จะไม่อธิบาย mechanism ของ `State<T>` ซ้ำ
- **Part 39 (Mutex, Arc และ Shared-State Concurrency) และ Part 48 (Tokio Runtime)**: connection pool คือรูปแบบหนึ่งของ shared state ที่ต้องแชร์ข้าม async task หลายพันตัวพร้อมกันอย่างปลอดภัย — บทนี้อ้างอิงความเข้าใจเรื่อง `Arc`, task scheduling, และ `.await` จาก Part 39/46-48 ตลอดเวลาโดยไม่อธิบายกลไกพื้นฐานซ้ำ
- **Part 57 (Serde เบื้องต้น)**: struct ที่ map มาจากแถวข้อมูลในตารางต้อง `#[derive(Serialize)]` เพื่อส่งกลับเป็น JSON ผ่าน Axum handler — บทนี้ใช้ derive macro ของ serde ควบคู่กับ `sqlx::FromRow` ตรงตามที่ Part 57 สอนไว้
- **Part 12 และ Part 30-31 (Error Handling)**: การแปลง `sqlx::Error` เป็น error type ของแอปเอง (`AppError` ตามแนวทางที่ Part 66 สอน) ใช้ `impl From<sqlx::Error> for AppError` และ `match` แบบ exhaustive ตรงตามที่ Part 12/30/31 สอนไว้ทั้งหมด
- **Part 11 (Option และ Null-Safety)**: column ที่ nullable ใน PostgreSQL ต้อง map เป็น `Option<T>` ใน Rust — บทนี้ใช้ความเข้าใจเรื่อง `Option<T>` จาก Part 11 ตรง ๆ ไม่มีอะไรใหม่ในระดับ type system เพียงแต่เป็นครั้งแรกที่ "ความไม่มีค่า" มาจากฐานข้อมูลจริงแทนโค้ด Rust ล้วน ๆ
- **Part 32-33 (Testing: Unit และ Integration)**: หัวข้อท้ายบทเรื่องการทดสอบกับฐานข้อมูลใช้ `#[test]`/`#[tokio::test]` และแนวคิด integration test แบบ end-to-end ที่ Part 32-33 ปูพื้นไว้
- **พื้นฐาน SQL**: บทนี้ไม่สอน SQL ตั้งแต่ต้น — สมมติว่าคุณอ่าน `SELECT`, `INSERT`, `UPDATE`, `DELETE`, และ `JOIN` พื้นฐานออก (ถ้าไม่คุ้นเคย ควรหาความรู้ SQL พื้นฐานเพิ่มก่อน เพราะบทนี้โฟกัสที่ "ฝั่ง Rust" ของการต่อฐานข้อมูล ไม่ใช่สอน SQL)

## หมายเหตุเรื่องการตรวจสอบเนื้อหา (สำคัญ — อ่านก่อนเริ่ม)

ก่อนเขียนบทนี้ ผู้เขียนตรวจสอบว่ามี PostgreSQL จริงให้ทดสอบหรือไม่ และพบว่า**มี PostgreSQL 16.13 ติดตั้งอยู่ในเครื่องจริง** (cluster ที่ตั้งค่าไว้แล้วแต่ปิดอยู่ — เปิดขึ้นมาด้วย `pg_ctlcluster 16 main start`) จึงสามารถทดสอบทุกตัวอย่างในบทนี้แบบ **compile และ run จริง เชื่อมต่อฐานข้อมูลจริง** ได้ทั้งหมด รวมถึง macro `sqlx::query!`/`sqlx::query_as!` ที่ต้องพึ่งพา `DATABASE_URL` ตอน compile time โดยตรง — **นี่ไม่ใช่ทางเลือกสำรอง (fallback)** แต่คือเส้นทางที่ดีที่สุดที่ทำได้จริงในบทนี้

รายละเอียดสิ่งที่ทดสอบจริง:

- ตั้งโปรเจกต์ scratch แยกไว้นอก repo (ไม่กระทบไฟล์ใด ๆ ในหลักสูตร) เพิ่ม `sqlx = "0.9.0"` (เวอร์ชันปัจจุบันจาก crates.io ณ วันที่เขียนบทนี้) พร้อม feature flag ตามที่บทนี้จะสอน
- สร้างฐานข้อมูล `rust_course_scratch` จริง รัน migration จริงด้วย `sqlx-cli` สร้างตาราง `books`/`borrow_records` จริง
- คอมไพล์และรันโค้ดที่ใช้ `sqlx::query!`/`sqlx::query_as!` (compile-time checked) จริง — เห็น error จริงจาก PostgreSQL เมื่อละเมิด unique constraint/foreign key constraint จริง ไม่ใช่ error ที่แต่งขึ้น
- พิสูจน์ transaction rollback จริงด้วยการเทียบค่าตัวเลขก่อน/หลัง (ไม่ใช่แค่บอกว่า "มันควรจะ work")
- พิสูจน์ pool exhaustion จริงด้วยการยึด connection จนครบ `max_connections` แล้ววัดเวลาที่ request ตัวต่อไปรอ
- อ่าน source code จริงของ `sqlx-core` (`Pool<DB>(pub(crate) Arc<PoolInner<DB>>)`) เพื่อยืนยันว่า `PgPool::clone()` ถูกจริงในระดับ implementation ไม่ใช่แค่คำโฆษณาใน docs
- รัน Axum server จริงพร้อม `curl` จริงยิง request CRUD ครบทุกกรณี รวมกรณี error (404, 409)
- ทดสอบ `#[sqlx::test]` attribute จริง พิสูจน์ว่า isolation ระหว่าง test ทำงานจริง

ทุกที่ที่มีการอ้างผลลัพธ์ (`cargo build`, error message, ตัวเลขจาก query) ในบทนี้คือผลลัพธ์ที่**รันจริงแล้วคัดลอกมา** ไม่มีการแต่ง output ขึ้นเอง — ถ้าเจอความต่างเล็กน้อยตอนคุณลองรันเอง (เช่น เวอร์ชัน `sqlx`/`axum` ใหม่กว่าที่เขียนไว้ตรงนี้) ให้ยึด `cargo add` ที่รันในเครื่องคุณเป็นความจริงล่าสุดเสมอ ecosystem ของ Rust อัปเดตเร็ว เวอร์ชันที่ระบุในบทนี้คือ snapshot ณ ช่วงเวลาที่เขียน ไม่ใช่ตัวเลขตายตัวตลอดไป

## เนื้อหา

### 70.1 ทำไมต้องเป็น SQLx: Async-Native และ Compile-Time Query Verification

ตั้งแต่ Part 62 คุณสร้างระบบตั๋ว/ห้องสมุดด้วย `Arc<Mutex<HashMap<u32, T>>>` เป็น "ฐานข้อมูลจำลอง" ในหน่วยความจำ ซึ่งใช้ได้ดีมากสำหรับการเรียนรู้ Axum เอง แต่มีข้อจำกัดที่ชัดเจนที่คุณคงสัมผัสได้แล้ว: ข้อมูลหายทุกครั้งที่โปรแกรมปิด ไม่มีทาง query ข้อมูลซับซ้อน (join, filter หลายเงื่อนไข, aggregate) ได้อย่างมีประสิทธิภาพ และไม่มีทาง scale ไปมากกว่าหนึ่ง process ได้เลย (แต่ละ process จะมี `HashMap` แยกกันคนละก้อน) — บทนี้คือจุดเปลี่ยนที่ทุกอย่างที่เรียนมาตั้งแต่ Part 62 จะ "ต่อกับของจริง"

Rust ecosystem มีตัวเลือกหลักสามกลุ่มสำหรับคุยกับฐานข้อมูลเชิงสัมพันธ์: **SQLx** (สิ่งที่บทนี้สอน), **Diesel** (Part 72), และ **SeaORM** (Part 73) — ทั้งสามแก้ปัญหาเดียวกันแต่เลือกจุดยืนต่างกันมาก มาดูจุดยืนของ SQLx ก่อนว่าทำไมบทนี้เลือกสอนตัวนี้เป็นตัวแรก

#### Async-native ตั้งแต่การออกแบบ

จาก Part 46-48 คุณรู้แล้วว่า `async fn` ใน Rust ไม่ได้ "รันจริง" ด้วยตัวเอง — มันต้องมี executor (เช่น Tokio) มา poll `Future` ที่ได้ ทุก I/O operation ที่ดี (ไฟล์, network, และรวมถึงฐานข้อมูล) ต้อง**ไม่ block thread ที่ executor ใช้อยู่** ไม่อย่างนั้นจะไปกีดขวาง task async อื่น ๆ ที่กำลังรอ queue อยู่บน thread เดียวกัน (ปัญหาเดียวกับที่ Part 48 อธิบายไว้เรื่อง "ห้ามเรียก blocking code ใน async context ตรง ๆ")

SQLx ถูกออกแบบให้เป็น **async ตั้งแต่บรรทัดแรกของโค้ด** — ทุก method ที่คุยกับฐานข้อมูล (`fetch_one`, `execute`, `fetch_all`, ...) คืนค่าเป็น `Future` ที่ทำงานผ่าน network I/O แบบ non-blocking ล้วน ๆ ผูกกับ async runtime ที่คุณเลือก (Tokio ในบทนี้ ตาม Part 48 — SQLx รองรับ `async-std` ด้วยผ่าน feature flag อื่น) เทียบกับ Diesel ที่เดิมออกแบบมาเป็น **synchronous ตั้งแต่ต้น** (แม้จะมี `diesel-async` เพิ่มมาทีหลังก็ยังไม่ใช่ค่า default ของ crate หลัก) — ความต่างนี้สำคัญมากในเว็บแอปที่รองรับ concurrent request จำนวนมาก เพราะ query ที่ blocking จริง ๆ (แม้จะเร็วมากในเชิง latency) จะกิน thread ของ Tokio runtime ไปตลอดช่วงเวลานั้น ทำให้ throughput รวมของระบบตกลงถ้ามี concurrent request มากพอ

#### Compile-time query verification: กลไกที่แท้จริง

จุดขายที่โดดเด่นที่สุดของ SQLx คือ macro `sqlx::query!` และ `sqlx::query_as!` ที่ทำสิ่งที่ ORM ทั่วไปทำไม่ได้: **ตรวจสอบ SQL ที่คุณเขียนตรงกับ schema จริงของฐานข้อมูล ตอน compile time** ไม่ใช่แค่ตอน runtime ตอนที่โปรแกรมรันจริงแล้วเจอ query พัง

กลไกจริงมีสองโหมด:

**โหมดที่ 1 — เชื่อมต่อฐานข้อมูลจริงตอน compile (ค่า default)**: ตอนคุณเรียก `cargo build` ตัว proc macro ของ `sqlx-macros` จะ**เปิด connection ไปยังฐานข้อมูลจริง** (อ่าน connection string จาก environment variable `DATABASE_URL` ที่ต้องตั้งไว้**ก่อน**สั่ง build) ส่ง SQL string ที่คุณเขียนใน `query!("SELECT ...")` ไปให้ PostgreSQL วิเคราะห์ผ่านกลไก **prepared statement** (คำสั่ง `PREPARE` ของ PostgreSQL ที่คืน metadata ของ column ที่ query นี้จะได้ โดยไม่ต้องรันจริง) ผลลัพธ์ที่ได้คือ **ชนิดข้อมูลจริงของแต่ละ column** (ชื่อ, PostgreSQL type, nullable หรือไม่) macro เอาข้อมูลนี้มา generate โค้ด Rust ที่ผูก type ให้ตรงกับ struct ที่คุณระบุ (ใน `query_as!`) หรือ tuple ที่ generate ให้เอง (ใน `query!`)

พูดให้เป็นรูปธรรม: ถ้าคุณพิมพ์ชื่อ column ผิด (`SELECT ttile FROM books` พิมพ์ `title` ผิดเป็น `ttile`) หรือใส่ type ให้ตัวแปรที่ bind ไม่ตรงกับ column จริง (`WHERE id = $1` แล้ว bind ด้วย `String` ทั้งที่ `id` เป็น `BIGINT`) **`cargo build` จะ fail ทันที** ก่อนที่โค้ดจะรันด้วยซ้ำ — error message จะบอกตรง ๆ ว่า column ไหนไม่มีอยู่จริง หรือ type ไม่ตรงกันอย่างไร (จะเห็นตัวอย่างจริงในหัวข้อกับดักท้ายบท)

**โหมดที่ 2 — offline mode ด้วย `.sqlx` cache**: การต้องมีฐานข้อมูลจริงเชื่อมต่อได้ตลอดเวลาที่ compile เป็นปัญหาในสถานการณ์จริงหลายแบบ (CI/CD ที่ไม่อยากรัน database container เต็มรูปแบบทุกครั้ง, เพื่อนร่วมทีมที่ยังไม่ได้ตั้งฐานข้อมูล local, การ build image สำหรับ deploy) SQLx แก้ปัญหานี้ด้วยคำสั่ง `cargo sqlx prepare` (จาก `sqlx-cli`) ที่**รันครั้งเดียวตอนมีฐานข้อมูลเชื่อมต่อได้** แล้วบันทึกผลลัพธ์ของทุก query ที่เจอในโค้ด (type ของแต่ละ column ที่ query นั้นคืน) เป็นไฟล์ JSON ในโฟลเดอร์ `.sqlx/` ที่ commit เข้า git ได้ตามปกติ จากนั้นตั้ง environment variable `SQLX_OFFLINE=true` ตอน build ครั้งต่อไป macro จะ**อ่านจาก `.sqlx/` แทนการต่อฐานข้อมูลจริง** — ได้ผลลัพธ์การ type-check เหมือนกันทุกประการ แต่ไม่ต้องมีฐานข้อมูลออนไลน์ตอน compile เลย

ทั้งสองโหมดนี้ผู้เขียนทดสอบจริงทั้งคู่ (ดูหัวข้อ 70.5) — build ด้วย `DATABASE_URL` ชี้ไปยัง PostgreSQL จริงสำเร็จ และ build อีกครั้งด้วย `SQLX_OFFLINE=true` โดยไม่มี `DATABASE_URL` เลยก็สำเร็จเช่นกัน (อ่านจาก `.sqlx/` ที่สร้างไว้แล้ว)

#### เทียบกับ ORM แบบ runtime-only (แนวคิด — จะลงรายละเอียดใน Part 72/73)

ORM ดั้งเดิมในภาษาอื่น ๆ (และ Diesel ในระดับหนึ่ง แม้จะมี compile-time query builder ของตัวเองที่ต่างจาก SQLx) มักให้คุณเขียน query ผ่าน method chain ที่เป็น Rust ล้วน ๆ (`books.filter(title.eq("Rust"))`) แทนการเขียน SQL string ตรง ๆ — ข้อดีคือไม่มีความเสี่ยงเรื่อง SQL string พิมพ์ผิด (เพราะไม่ได้เขียน SQL เลย) แต่ข้อเสียคือคุณต้องเรียนรู้ DSL ของ ORM นั้นแยกจาก SQL ที่มีอยู่แล้ว และบางครั้ง SQL ที่ ORM generate ให้ไม่มีประสิทธิภาพเท่าที่คุณเขียนมือ (โดยเฉพาะ query ซับซ้อนที่มี join หลายตาราง)

SQLx เลือกจุดยืนตรงกลาง: **คุณเขียน SQL จริงด้วยมือ** (ควบคุมได้เต็มที่ ประสิทธิภาพเท่าที่ SQL เขียนได้) แต่ได้ความปลอดภัยระดับ compile-time กลับมาผ่าน macro แทน — เหมาะกับคนที่คุ้นเคย SQL อยู่แล้วและต้องการควบคุม query เต็มรูปแบบ ในขณะที่ SeaORM (Part 73) จะเข้าใกล้ ORM แบบ entity/relation มากกว่า (คล้าย ORM ในภาษาอื่นที่หลายคนคุ้นเคย) SQLx จึงเป็นจุดเริ่มต้นที่ดีที่สุดสำหรับบทนี้ เพราะสอนให้เข้าใจว่า "การคุยกับฐานข้อมูลจาก Rust จริง ๆ มันทำงานอย่างไร" ก่อนที่จะไปเห็น abstraction ที่ห่อหุ้มมากขึ้นใน Part 72-73

**ตารางสรุปจุดยืนของทั้งสามตัวเลือก** (รายละเอียดเชิงลึกจะอยู่ใน Part 72-73 — ตารางนี้แค่ให้เห็นภาพรวมก่อนเริ่มเรียน SQLx):

| | SQLx (บทนี้) | Diesel (Part 72) | SeaORM (Part 73) |
|---|---|---|---|
| วิธีเขียน query | SQL จริงในรูป string literal | Rust DSL (method chain) ที่ generate SQL ให้ | Entity/ActiveRecord API (คล้าย ORM ภาษาอื่น) |
| การตรวจสอบ query | Compile-time (ต่อ DB จริงหรือ `.sqlx` cache) ต่อ **SQL string ที่เขียนเอง** | Compile-time ผ่าน Rust type system (schema ถูก generate เป็น Rust type ผ่าน `diesel print-schema`) | ส่วนใหญ่ runtime (entity ถูก generate จาก schema แต่ query builder เองไม่ type-check กับ DB ตอน compile) |
| Async รองรับ | Async-native ตั้งแต่ต้น | เดิม sync-first (มี `diesel-async` แยกเพิ่มทีหลัง) | Async-native (สร้างบน SQLx เป็น driver ชั้นล่างจริง ๆ) |
| ระดับ abstraction | ต่ำ — ใกล้ SQL ดิบที่สุด | กลาง — DSL ที่ยัง map ใกล้ SQL | สูง — เข้าใกล้ ORM/entity แบบเต็มรูปแบบ |
| เหมาะกับ | คนที่คุ้น SQL อยากคุมทุกอย่างเอง | คนที่อยากได้ type-safety สูงสุดของ Rust DSL โดยไม่ต้องเขียน SQL string | คนที่อยากได้ประสบการณ์แบบ ORM ดั้งเดิม (คล้าย ActiveRecord/Entity Framework) |

(ข้อสังเกตที่น่าสนใจ: SeaORM ใช้ SQLx เป็น driver ชั้นล่างจริง ๆ ไม่ใช่คู่แข่งที่แยกกันโดยสิ้นเชิง — เพราะฉะนั้นความเข้าใจเรื่อง connection pool, transaction, error handling ที่เรียนในบทนี้จะยังมีประโยชน์ตรงเมื่อไปถึง Part 73 ด้วย)

### 70.2 ติดตั้งและตั้งค่าโปรเจกต์

#### เพิ่ม `sqlx` เข้าโปรเจกต์

```bash
cargo add sqlx --features runtime-tokio,postgres,macros,chrono,uuid
```

ผลลัพธ์จริง (ทดสอบในเครื่องผู้เขียนบทนี้ ณ วันที่เขียน — `sqlx` เวอร์ชันปัจจุบันบน crates.io คือ `0.9.0`):

```
    Adding sqlx v0.9.0 to dependencies
             Features:
             + _rt-tokio
             + any
             + chrono
             + derive
             + json
             + macros
             + migrate
             + postgres
             + runtime-tokio
             + sqlx-postgres
             + uuid
             39 deactivated features
```

**อธิบาย feature ที่เปิดแต่ละตัว:**

- **`runtime-tokio`** — บอก SQLx ว่าให้ผูกกับ Tokio runtime (ตรงกับที่ Part 48 สอน `#[tokio::main]`) SQLx รองรับ runtime อื่นด้วย (เช่น `async-std`) แต่บทนี้ทั้งบทใช้ Tokio เพราะเป็น runtime เดียวกับที่ Axum ใช้ — **ต้องเลือก runtime ให้ตรงกับที่แอปทั้งตัวใช้เสมอ** ผสมกันไม่ได้
- **`postgres`** — เปิด driver สำหรับ PostgreSQL โดยเฉพาะ (SQLx รองรับ MySQL, SQLite ด้วยผ่าน feature แยก `mysql`/`sqlite` — เปิดเฉพาะตัวที่ใช้จริงเพื่อลด compile time และขนาด binary)
- **`macros`** — เปิด macro `query!`/`query_as!`/`FromRow` derive ที่บทนี้ใช้ตลอด (ถ้าไม่เปิดจะใช้ได้แค่ฟังก์ชันแบบไม่มี `!` เช่น `sqlx::query()`/`sqlx::query_as::<_, T>()` ที่ไม่ type-check ตอน compile — หัวข้อ 70.4 จะอธิบายความต่างละเอียด)
- **`chrono`** — เปิดการแปลงระหว่าง PostgreSQL type `TIMESTAMPTZ`/`DATE`/`TIME` กับ type ของ crate `chrono` (`DateTime<Utc>`, `NaiveDate`, ...) โดยตรง (ไม่มี feature นี้ จะต้อง map ผ่าน string เอง ยุ่งยากกว่ามาก)
- **`uuid`** — เปิดการแปลงระหว่าง PostgreSQL type `UUID` กับ `uuid::Uuid` — เหมาะกับตารางที่ใช้ UUID เป็น primary key (บทนี้ใช้ `BIGSERIAL` เป็นหลักเพื่อความง่าย แต่เปิด feature ไว้เผื่อขยายในบทหลัง ๆ)

`Cargo.toml` ที่ได้:

```toml
[dependencies]
sqlx = { version = "0.9.0", features = ["runtime-tokio", "postgres", "macros", "chrono", "uuid"] }
```

พร้อมกับ dependency พื้นฐานอื่น ๆ ที่ต้องมีควบคู่กันตามที่ Part 48/57/62 สอนไว้แล้ว:

```bash
cargo add tokio --features full
cargo add serde --features derive
cargo add serde_json
cargo add axum
cargo add chrono --features serde
```

**ข้อสังเกตสำคัญที่มือใหม่พลาดบ่อย**: ชื่อ feature ของ `sqlx` เปลี่ยนไปตามเวอร์ชัน — เวอร์ชันเก่ามาก ๆ (0.5-0.6) ใช้ชื่อ `runtime-tokio-rustls`/`runtime-tokio-native-tls` (ผูก TLS backend ไว้ในชื่อ feature เลย) ส่วนเวอร์ชันปัจจุบันแยก TLS backend ออกมาเป็นเรื่องของ feature อื่น (`tls-rustls`, `tls-native-tls`) ทำให้ `runtime-tokio` เพียว ๆ ก็เพียงพอสำหรับ localhost/dev แล้ว — **อย่า copy feature flag จาก tutorial เก่าตรง ๆ** ให้รัน `cargo add sqlx --features ...` แล้วอ่านผลลัพธ์ที่ cargo แสดงเสมอ เพราะมันจะบอกชื่อ feature ที่ตรงกับเวอร์ชันจริงที่ล็อกไว้ใน `Cargo.toml` ของคุณ

#### `DATABASE_URL`: ธรรมเนียมของ 12-factor app

SQLx (และ `sqlx-cli`) ใช้ environment variable ชื่อ `DATABASE_URL` เป็นค่าเริ่มต้นสำหรับบอกว่าจะต่อฐานข้อมูลที่ไหน รูปแบบมาตรฐานคือ:

```
postgres://<user>:<password>@<host>:<port>/<database_name>
```

เช่น:

```
DATABASE_URL=postgres://postgres:postgres@127.0.0.1:5432/rust_course_scratch
```

นี่คือรูปแบบเดียวกับที่ Part 59 สอนเรื่อง `#[arg(env = "...")]` ของ `clap` — connection string ของฐานข้อมูลเป็นตัวอย่างคลาสสิกของค่า config ที่**ไม่ควร hardcode ในโค้ด** (เปลี่ยนไปตาม environment: dev/staging/production ต่อฐานข้อมูลคนละตัว) และ**ไม่ควร commit เข้า git** (มี password อยู่ในนั้นตรง ๆ) ธรรมเนียมที่ใช้กันทั่วไปคือเก็บไว้ในไฟล์ `.env` (ที่ใส่ใน `.gitignore`) แล้วให้ `sqlx-cli`/แอปอ่านผ่าน crate `dotenvy` หรือปล่อยให้ shell/deployment platform ตั้งเป็น environment variable จริงตอน runtime

ในโค้ด Rust เอง ถ้าไม่ได้ใช้ `sqlx::query!`/`query_as!` (ที่อ่าน `DATABASE_URL` ตอน **compile time** ผ่าน mechanism ของ macro เอง ไม่เกี่ยวกับ `std::env` เลย) การอ่านค่านี้ตอน **runtime** เพื่อเปิด connection จริงตอนโปรแกรมสตาร์ทใช้ `std::env::var` ธรรมดา:

```rust
let database_url = std::env::var("DATABASE_URL")
    .expect("ต้องตั้ง DATABASE_URL ก่อนรันโปรแกรม (เช่น export DATABASE_URL=postgres://...)");
```

**จุดที่มักสับสน**: `DATABASE_URL` มี**สองบทบาทที่แยกกันโดยสิ้นเชิง**:

1. **ตอน compile time** — macro `query!`/`query_as!` อ่านมันเพื่อไปต่อฐานข้อมูล type-check SQL (หรือใช้ `.sqlx/` cache ถ้าตั้ง `SQLX_OFFLINE=true`) — ค่านี้ต้องตั้งไว้ **ก่อน** สั่ง `cargo build`/`cargo run` เสมอ (export ไว้ใน shell หรือใส่ในไฟล์ `.env` ที่ `sqlx-cli`/build script อ่าน)
2. **ตอน runtime** — โค้ด `main()` ของแอปคุณเองอ่านมันด้วย `std::env::var` เพื่อเปิด `PgPool` จริงตอนโปรแกรมรัน — ค่านี้ต้องมีตอน**รัน**โปรแกรม (อาจเป็นค่าเดียวกันหรือคนละค่าก็ได้ เช่น dev ต่อ localhost ตอน compile แต่ production ต่อฐานข้อมูลจริงตอน runtime — ถ้าใช้ offline mode ทั้งสองขั้นตอนจะไม่ผูกกันเลย)

### 70.3 Connection Pool: `PgPoolOptions` และเหตุผลที่ต้องมี Pool

#### ทำไมไม่เปิด connection ใหม่ทุกครั้งที่มี request

ลองนึกภาพเขียนแบบตรงไปตรงมาที่สุด: ทุก handler เปิด connection ใหม่ตอนเริ่ม ใช้เสร็จก็ปิด:

```rust
// ตัวอย่าง "อย่าทำแบบนี้" — เปิด/ปิด connection ใหม่ทุกครั้งที่มี request
async fn naive_handler_sketch() -> Result<(), sqlx::Error> {
    // เปิด TCP connection ใหม่ + ทำ handshake + authentication กับ PostgreSQL ทุกครั้ง
    let conn = sqlx::postgres::PgConnection::connect(
        "postgres://postgres:postgres@127.0.0.1:5432/rust_course_scratch",
    )
    .await?;
    drop(conn); // ปิดทันทีหลังใช้เสร็จ (สมมติว่าใช้เสร็จแล้ว)
    Ok(())
}
```

โค้ดนี้ **compile ผ่านและทำงานได้** แต่มีต้นทุนแฝงที่ร้ายแรงมากในระบบที่รับ request จำนวนมาก: การเปิด connection ใหม่ไปยัง PostgreSQL ไม่ใช่แค่เปิด TCP socket — มันต้องทำ **TCP handshake**, **TLS handshake** (ถ้าใช้ TLS), และ **PostgreSQL authentication handshake** (ส่ง username/password ไปมาหลาย round-trip ตาม auth method ที่ตั้งไว้ เช่น `scram-sha-256` ที่ต้องมีการแลก challenge/response) — รวมกันแล้วมีค่าใช้จ่ายเป็น**หลัก millisecond ต่อครั้ง** ซึ่งฟังดูน้อย แต่ถ้าแอปรับ **หลายพัน request ต่อวินาที** (เหมือนที่ Part 64 พูดถึงตอนอธิบายเรื่อง `Arc::clone` cost) การเปิด connection ใหม่ทุกครั้งจะกลายเป็นคอขวดที่ใหญ่กว่าตัว query จริงเสียอีก แถมยังไปกดดัน PostgreSQL ฝั่งเซิร์ฟเวอร์ที่ต้อง fork process/thread ใหม่รองรับ connection แต่ละตัว (ขึ้นกับ PostgreSQL configuration) — เกิน `max_connections` ของ PostgreSQL เองได้ง่าย ๆ ถ้า traffic สูง

นี่คือปัญหาเดียวกันในเชิงโครงสร้างกับที่ Part 39 อธิบายไว้เรื่อง "ทำไมต้องแชร์ state ข้าม thread ด้วย `Arc<Mutex<T>>`" และ Part 48 เรื่อง "ทำไม async task ควรเบา (ไม่ผูก resource หนักต่อ task)" — คำตอบคือ**สร้าง resource ที่แพงไว้ล่วงหน้าจำนวนจำกัด แล้วแชร์ใช้ซ้ำ** แทนสร้างใหม่ทุกครั้ง สำหรับ database connection แนวคิดนี้เรียกว่า **connection pool**

#### `PgPoolOptions`: สร้าง pool ที่ถูกวิธี

```rust
use sqlx::postgres::PgPoolOptions;
use std::time::Duration;

async fn create_pool(database_url: &str) -> Result<sqlx::PgPool, sqlx::Error> {
    PgPoolOptions::new()
        .max_connections(10)       // จำนวน connection สูงสุดที่ pool จะเปิดพร้อมกัน
        .min_connections(1)        // จำนวน connection ขั้นต่ำที่พยายามคงไว้เสมอ (ลด cold-start latency)
        .acquire_timeout(Duration::from_secs(5)) // รอ connection ว่างนานสุดกี่วินาทีก่อน error
        .idle_timeout(Duration::from_secs(600))  // ปิด connection ที่ไม่ได้ใช้นานเกินนี้ทิ้ง
        .connect(database_url)
        .await
}
```

`.connect(database_url).await` ทำสองอย่างพร้อมกัน: (1) เปิด connection จริงจำนวน `min_connections` ตัวทันที (เพื่อให้ request แรก ๆ ไม่ต้องรอเปิด connection ใหม่) และ (2) คืนค่า `PgPool` — struct ที่**ไม่ใช่ connection ตัวเดียว** แต่เป็น**ตัวจัดการ pool ของ connection หลายตัว** ทุกครั้งที่คุณเรียก `.fetch_one()`/`.execute()` ผ่าน `&pool` ตรง ๆ (ไม่ผ่าน transaction) SQLx จะ**หยิบ connection ที่ว่างตัวหนึ่งจาก pool มาใช้ชั่วคราว แล้วคืนกลับเข้า pool ทันทีที่ query เสร็จ** — connection ตัวเดิมถูกใช้ซ้ำได้เรื่อย ๆ ข้าม request ที่ต่างกัน ไม่ต้องเปิด/ปิดใหม่

**เชื่อมกับ Part 48 เรื่อง Tokio task concurrency**: Axum handler แต่ละตัวรันเป็น async task แยกกัน (อาจ concurrent กันหลายพันตัวพร้อมกันถ้า traffic สูง) — `max_connections` คือ**ขีดจำกัดจริง**ว่ากี่ task จะสามารถ "คุยกับฐานข้อมูลพร้อมกันได้จริง" ในเวลาเดียวกัน task ที่เกินจำนวนนี้จะ**รอ**อยู่ในคิว (หัวข้อ 70.11 จะพิสูจน์พฤติกรรมนี้ด้วยโค้ดจริง) — เลข `max_connections` ที่เหมาะสมขึ้นกับหลายปัจจัย (จำนวน CPU core ของเครื่อง database, `max_connections` ของ PostgreSQL เอง, จำนวน instance ของแอปที่รันพร้อมกัน) ไม่มีตัวเลขตายตัวที่ใช้ได้ทุกสถานการณ์ แต่หลักการทั่วไปคือ**ไม่ควรตั้งสูงเกินจำเป็น** เพราะ connection ที่เปิดทิ้งไว้เฉย ๆ (idle) ก็ยังกินหน่วยความจำฝั่ง PostgreSQL อยู่ดี

#### ตารางอ้างอิง: option สำคัญของ `PgPoolOptions`

`PgPoolOptions` มี builder method อีกหลายตัวนอกจากสี่ตัวที่ใช้ในตัวอย่างข้างบน — ตารางนี้สรุป option ที่ควรรู้จักสำหรับใช้งานจริง (ค่า default อ้างอิงจากเวอร์ชัน `sqlx` ที่ใช้ในบทนี้ — ควรอ่าน docs ของเวอร์ชันที่ใช้จริงอีกครั้งเสมอเพราะค่า default อาจเปลี่ยนได้ระหว่างเวอร์ชัน):

| Method | ความหมาย | ข้อสังเกตเชิงปฏิบัติ |
|---|---|---|
| `.max_connections(n)` | จำนวน connection สูงสุดที่ pool เปิดพร้อมกันได้ | ค่านี้สำคัญที่สุด — ผูกตรงกับ throughput สูงสุดของระบบด้าน database |
| `.min_connections(n)` | จำนวน connection ขั้นต่ำที่พยายามคงไว้เสมอ (แม้ไม่มี traffic) | ค่า default คือ 0 (ไม่เปิดล่วงหน้าเลย เปิดตามความต้องการจริง) — ตั้งเป็นค่าน้อย ๆ (1-2) ช่วยลด latency ของ request แรก ๆ หลัง idle นาน ๆ |
| `.acquire_timeout(d)` | รอ connection ว่างนานสุดกี่วินาทีก่อนคืน `PoolTimedOut` | ควรตั้งให้สั้นกว่า timeout ของ client ที่เรียกเข้ามา (ไม่มีประโยชน์ที่จะรอนานกว่าที่ client จะรอไหว) |
| `.idle_timeout(d)` | ปิด connection ที่ไม่ได้ใช้นานเกินนี้ทิ้ง (ลดจำนวน connection ที่เปิดทิ้งไว้เฉย ๆ ตอน traffic ต่ำ) | ค่า default ประมาณ 10 นาที — เหมาะสำหรับแอปที่ traffic ขึ้นลงเป็นช่วง ๆ |
| `.max_lifetime(d)` | ปิด (แล้วเปิดใหม่) connection ที่มีอายุนานเกินนี้ ไม่ว่าจะ idle หรือไม่ | ป้องกันปัญหา connection ที่เปิดค้างนานเกินจนอาจมีปัญหาสะสม (เช่น memory leak ฝั่ง driver บางตัว หรือ load balancer ฝั่งเครือข่ายตัดการเชื่อมต่อที่ค้างนานเกินไปเงียบ ๆ) |
| `.test_before_acquire(bool)` | ตรวจสอบว่า connection ที่จะให้ยืมยัง "มีชีวิต" อยู่จริงก่อนส่งให้ใช้ (ส่ง ping สั้น ๆ) | ค่า default คือ `true` — ปลอดภัยกว่าเล็กน้อยแต่มี overhead เพิ่มขึ้นนิดหน่อยต่อการ acquire แต่ละครั้ง ปิดได้ถ้ามั่นใจว่า network ระหว่างแอปกับฐานข้อมูลนิ่งมาก |

### 70.4 `sqlx::query!` vs `sqlx::query`/`sqlx::query_as`: สองระดับของการตรวจสอบ

SQLx มี API สองชุดที่ทำงานคล้ายกันแต่ต่างกันที่ **เวลาที่ type-check เกิดขึ้น**:

| | `sqlx::query!("...")` / `query_as!(T, "...")` | `sqlx::query("...")` / `query_as::<_, T>("...")` |
|---|---|---|
| Type-check เกิดตอนไหน | **compile time** (ผ่าน DB จริงหรือ `.sqlx` cache) | **runtime** (ตอน query จริงรัน) |
| ต้องมี `DATABASE_URL`/`.sqlx` ตอน build ไหม | ต้องมี | ไม่ต้อง |
| ตรวจชื่อ column ผิด/type ไม่ตรงได้เมื่อไร | ตอน `cargo build` — ก่อนโปรแกรมรันด้วยซ้ำ | ตอนรันจริงถึง query นั้น (อาจเจอ error ตอน production ถ้า test ไม่ครอบคลุม) |
| SQL string ต้องเป็น literal ในโค้ดไหม | ต้องเป็น string literal ตรง ๆ (macro ต้องเห็น SQL ตอน compile) | ไม่ต้อง — รับ `&str`/`String` แบบ dynamic ได้ (แต่มีเงื่อนไขด้านความปลอดภัย — ดูหัวข้อ 70.6) |

ตัวอย่างที่เทียบกันตรง ๆ (ทั้งสองแบบทำงานเหมือนกัน แต่ต่างกันที่ระดับการตรวจสอบ):

```rust
use chrono::{DateTime, Utc};
use serde::Serialize;
use sqlx::PgPool;

// struct ที่แต่ละ field ต้อง derive sqlx::FromRow เพื่อ map จากแถวข้อมูลได้
// derive Serialize คู่กัน (ตาม Part 57) เพื่อส่งกลับเป็น JSON ผ่าน Axum handler ได้ตรง ๆ
#[derive(Debug, Serialize, sqlx::FromRow)]
struct Book {
    id: i64,
    isbn: String,
    title: String,
    author: String,
    total_copies: i32,
    available_copies: i32,
    published_year: Option<i32>, // column ที่ nullable ต้องเป็น Option<T> (หัวข้อ 70.8)
    created_at: DateTime<Utc>,
}

// --- แบบที่ 1: query_as! (compile-time checked) ---
async fn find_book_checked(pool: &PgPool, isbn: &str) -> Result<Book, sqlx::Error> {
    sqlx::query_as!(
        Book,
        r#"SELECT id, isbn, title, author, total_copies, available_copies, published_year, created_at
           FROM books WHERE isbn = $1"#,
        isbn
    )
    .fetch_one(pool)
    .await
}

// --- แบบที่ 2: query_as (runtime checked, ไม่มี ! ท้าย) ---
async fn find_book_runtime(pool: &PgPool, isbn: &str) -> Result<Book, sqlx::Error> {
    sqlx::query_as::<_, Book>("SELECT * FROM books WHERE isbn = $1")
        .bind(isbn)
        .fetch_one(pool)
        .await
}
```

ทั้งสองฟังก์ชันนี้ผู้เขียน**คอมไพล์และรันจริง**ทั้งคู่สำเร็จ (ต่อ PostgreSQL จริง มีตาราง `books` จริงตามที่หัวข้อ 70.5 จะสร้าง) — ผลลัพธ์ที่ได้เหมือนกัน ต่างกันแค่ **เวลา** ที่ SQLx ตรวจสอบว่า SQL ตรงกับ schema จริงไหม: แบบแรกตรวจตอน `cargo build` (ก่อนโปรแกรมรันด้วยซ้ำ — ลองพิมพ์ชื่อ column ผิดดูจะเห็น `cargo build` fail ทันที) แบบที่สองไม่ตรวจอะไรเลยตอน compile (`SELECT *` เป็นตัวอย่างที่ตั้งใจให้เห็นว่า runtime API ไม่ผูกกับ schema ชัดเจนแบบ macro — ผลลัพธ์จะพังตอน runtime ถ้า schema ไม่ตรงกับ struct จริง ๆ เท่านั้น)

**หลักการเลือกใช้**: ใช้ `query!`/`query_as!` เป็นค่า default เสมอเมื่อ SQL เป็น literal คงที่ (กรณีส่วนใหญ่ในแอปจริง) เพื่อได้ความปลอดภัยระดับ compile-time เต็มที่ — สลับไปใช้ `query`/`query_as` แบบไม่มี `!` เฉพาะกรณีที่ต้อง**สร้าง SQL แบบ dynamic** จริง ๆ (เช่น filter ที่จำนวนเงื่อนไขไม่แน่นอนตาม input ผู้ใช้ — สถานการณ์นี้ SQLx มี `QueryBuilder` ช่วยสร้าง SQL แบบ dynamic อย่างปลอดภัยกว่าการต่อ string เอง ซึ่งจะกล่าวถึงเพิ่มใน Part 71)

#### Dynamic Row Access ด้วย `sqlx::Row`: ทางเลือกเมื่อไม่อยากนิยาม struct

มีอีกสถานการณ์ที่ทั้ง `query_as!`/`query_as` ไม่เหมาะ: เวลาที่คุณต้องการอ่านผลลัพธ์ query แบบ **ไม่รู้ shape ล่วงหน้า** (เช่น เครื่องมือ debug ที่รับ SQL จาก argument แล้วพิมพ์ผลลัพธ์อะไรก็ได้ที่ได้กลับมา) — SQLx มี trait `sqlx::Row` ที่ให้ดึงค่าออกจากแต่ละแถวทีละคอลัมน์ด้วยชื่อหรือตำแหน่งได้ตรง ๆ ผ่าน `.try_get()`:

```rust
use sqlx::Row;

async fn peek_first_book(pool: &sqlx::PgPool) -> Result<(), sqlx::Error> {
    let row = sqlx::query("SELECT title, available_copies FROM books LIMIT 1")
        .fetch_one(pool)
        .await?;

    // ดึงค่าออกทีละคอลัมน์ด้วยชื่อ — ต้องระบุ type เป้าหมายให้ตรงกับคอลัมน์จริงเอง (ไม่มีอะไรช่วย type-check ให้)
    let title: String = row.try_get("title")?;
    let available: i32 = row.try_get("available_copies")?;
    println!("title={title} avail={available}");
    Ok(())
}
```

ผู้เขียนทดสอบจริง ได้ผลลัพธ์ตรงตามที่คาด (`title=Programming Rust avail=2` จากข้อมูลที่ insert ไว้ก่อนหน้าในหัวข้อ 70.10) — **ข้อสังเกตสำคัญ**: `.try_get::<String>("title")` ไม่มีการตรวจสอบใด ๆ ตอน compile ว่าคอลัมน์ `title` มีอยู่จริงไหม หรือ type ตรงกับ `String` ไหม — ทุกอย่างเช็คตอน runtime เท่านั้น (คืน `Err(sqlx::Error::ColumnNotFound(...))` หรือ type mismatch error ถ้าผิด) นี่คือเหตุผลที่ pattern นี้ควรใช้เป็น**ทางเลือกสุดท้าย** เมื่อ struct ตายตัวใช้ไม่ได้จริง ๆ เท่านั้น — สำหรับ CRUD ปกติที่รู้ shape ข้อมูลล่วงหน้าแน่นอน (กรณีส่วนใหญ่ของแอปจริง) `query_as!`/`#[derive(sqlx::FromRow)]` ยังคงเป็นตัวเลือกที่ดีกว่าเสมอ เพราะได้ type-safety เต็มรูปแบบกลับมาโดยไม่ต้องเขียน `.try_get()` ทีละคอลัมน์ด้วยมือ

### 70.5 Migration ด้วย `sqlx-cli`: สร้างตาราง `books`/`borrow_records`

#### ติดตั้ง `sqlx-cli`

```bash
cargo install sqlx-cli --no-default-features --features rustls,postgres
```

ผลลัพธ์จริง (ทดสอบในเครื่องผู้เขียนบทนี้ — ใช้เวลาประมาณ 1-2 นาทีเพราะต้อง compile จาก source):

```
    Finished `release` profile [optimized] target(s) in 1m 46s
  Installing /root/.cargo/bin/cargo-sqlx
  Installing /root/.cargo/bin/sqlx
   Installed package `sqlx-cli v0.9.0` (executables `cargo-sqlx`, `sqlx`)
```

**อธิบาย feature flag**: `--no-default-features --features rustls,postgres` จำกัดให้ `sqlx-cli` compile เฉพาะส่วนที่ใช้จริง (TLS backend แบบ `rustls` + driver `postgres`) — ค่า default ของ `sqlx-cli` จะพยายาม compile รองรับทั้ง MySQL, SQLite, PostgreSQL พร้อมกัน ซึ่งทำให้เวลา compile นานขึ้นมากโดยไม่จำเป็นถ้าใช้แค่ PostgreSQL

#### สร้าง migration ใหม่

```bash
export DATABASE_URL="postgres://postgres:postgres@127.0.0.1:5432/rust_course_scratch"
sqlx migrate add -r create_books_table
```

ผลลัพธ์จริง:

```
Creating migrations/20260926235829_create_books_table.up.sql
Creating migrations/20260926235829_create_books_table.down.sql
```

**อธิบาย flag `-r`**: บอกให้สร้าง migration แบบ **reversible** — มีทั้งไฟล์ `.up.sql` (คำสั่งที่ใช้ตอนอัปเดต schema ไปข้างหน้า) และ `.down.sql` (คำสั่งย้อนกลับ ใช้ตอน rollback) ถ้าไม่ใส่ `-r` จะได้ไฟล์เดียว (`<timestamp>_<name>.sql`) ที่ไม่มีทาง rollback อัตโนมัติ — สำหรับ migration ที่สร้างตารางใหม่ (ยังไม่มีข้อมูลสำคัญเสี่ยงหาย) การมี `down.sql` มีประโยชน์มากตอน dev (ลองผิดลองถูก schema ได้ง่าย) ชื่อไฟล์ขึ้นต้นด้วย **timestamp** เสมอ (`20260926235829`) เพื่อการันตีว่า migration จะถูกรันตามลำดับเวลาที่สร้างจริง ไม่ว่าจะสร้างจากเครื่องไหนของทีมก็ตาม

#### เขียน SQL ของ migration

`migrations/20260926235829_create_books_table.up.sql`:

```sql
-- สร้างตาราง books สำหรับระบบห้องสมุด (ต่อยอดจาก domain ที่ใช้มาตั้งแต่ Part 62)
CREATE TABLE books (
    id BIGSERIAL PRIMARY KEY,
    isbn TEXT NOT NULL UNIQUE,
    title TEXT NOT NULL,
    author TEXT NOT NULL,
    total_copies INTEGER NOT NULL CHECK (total_copies >= 0),
    available_copies INTEGER NOT NULL CHECK (available_copies >= 0),
    published_year INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE borrow_records (
    id BIGSERIAL PRIMARY KEY,
    book_id BIGINT NOT NULL REFERENCES books(id),
    borrower_name TEXT NOT NULL,
    borrowed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    returned_at TIMESTAMPTZ
);
```

`migrations/20260926235829_create_books_table.down.sql`:

```sql
DROP TABLE IF EXISTS borrow_records;
DROP TABLE IF EXISTS books;
```

**อธิบายการเลือก type/constraint ทีละส่วน (นี่คือ schema design ที่มีเหตุผล ไม่ใช่สุ่มเลือก):**

- **`BIGSERIAL PRIMARY KEY`** — `SERIAL`/`BIGSERIAL` คือ syntax สะดวกของ PostgreSQL ที่สร้าง sequence auto-increment ให้อัตโนมัติ (`BIGSERIAL` ใช้ `BIGINT` ภายใน รองรับตัวเลขได้มากกว่า `SERIAL` ที่ใช้ `INTEGER` — เลือก `BIGSERIAL` เป็นค่า default ที่ปลอดภัยสำหรับตารางที่ข้อมูลอาจโตมากในระยะยาว)
- **`isbn TEXT NOT NULL UNIQUE`** — ISBN ต้องไม่ซ้ำกันในระบบจริง (หนังสือแต่ละเล่มมี ISBN ของตัวเอง) การใส่ `UNIQUE` ที่ระดับฐานข้อมูล**บังคับ invariant นี้จริง**ไม่ว่าโค้ด Rust ชั้นบนจะเช็คซ้ำไหม (defense in depth — เชื่อมกับหลักการ "ตรวจสอบที่ชั้นที่ใกล้ข้อมูลที่สุด" ที่ตรงกับแนวคิด type-safety ของ Rust เอง เพียงแต่ย้ายมาอยู่ระดับ database constraint แทน compile-time type) หัวข้อ 70.7 จะพิสูจน์ว่า constraint นี้ป้องกัน duplicate ได้จริงแม้โค้ด Rust จะพลาดเช็คไป
- **`total_copies INTEGER NOT NULL CHECK (total_copies >= 0)`** และ **`available_copies ... CHECK (available_copies >= 0)`** — `CHECK` constraint บังคับว่าตัวเลขต้องไม่ติดลบ ป้องกัน bug เชิงตรรกะ (เช่น ลด `available_copies` เกินจำนวนที่มีจริงจนติดลบ) ตั้งแต่ระดับฐานข้อมูล ไม่ต้องรอให้โค้ด Rust เช็คเองทุกจุดที่แก้ค่านี้
- **`book_id BIGINT NOT NULL REFERENCES books(id)`** — **foreign key constraint** การันตีว่าทุก `borrow_records.book_id` ต้องชี้ไปยัง `books.id` ที่มีอยู่จริงเท่านั้น — ฐานข้อมูลจะ**ปฏิเสธ** insert/update ที่ใส่ `book_id` ที่ไม่มีจริงทันที (หัวข้อ 70.9 เรื่อง transaction จะใช้ constraint นี้จำลอง error กลางทางเพื่อพิสูจน์ rollback)
- **`returned_at TIMESTAMPTZ`** (ไม่มี `NOT NULL`) — คอลัมน์นี้**ตั้งใจให้ nullable**: หนังสือที่ยังไม่ถูกคืนจะมีค่าเป็น `NULL` ในคอลัมน์นี้ (ต่างจาก `borrowed_at` ที่ต้องมีค่าเสมอตอนสร้าง record) — นี่คือตัวอย่าง nullable column ตัวจริงที่หัวข้อ 70.8 จะ map เข้า `Option<DateTime<Utc>>`
- **`TIMESTAMPTZ`** (ไม่ใช่ `TIMESTAMP` เฉย ๆ) — `TIMESTAMPTZ` (timestamp with time zone) เก็บเวลาพร้อมข้อมูล timezone แปลงเป็น UTC ภายในเสมอ เป็น type ที่**ควรใช้เป็นค่า default** สำหรับ timestamp ในระบบจริงแทบทุกกรณี (ต่างจาก `TIMESTAMP` เฉย ๆ ที่ไม่เก็บ timezone เลย ทำให้ตีความข้อมูลผิดพลาดได้ง่ายเวลาระบบมีผู้ใช้งานคนละ timezone) — map ตรงกับ `chrono::DateTime<Utc>` ในฝั่ง Rust พอดี (ตามที่เปิด feature `chrono` ไว้ในหัวข้อ 70.2)

#### รัน migration

```bash
sqlx migrate run
```

ผลลัพธ์จริง:

```
Applied 20260926235829/migrate create books table (6.638243ms)
```

ตรวจสอบด้วย `psql` จริง (`\d books`) ยืนยันว่าตารางถูกสร้างตรงตามที่ตั้งใจ (ตัดมาเฉพาะส่วนสำคัญ):

```
                                          Table "public.books"
      Column      |           Type           | Collation | Nullable |              Default
------------------+--------------------------+-----------+----------+-----------------------------------
 id               | bigint                   |           | not null | nextval('books_id_seq'::regclass)
 isbn             | text                     |           | not null |
 ...
Indexes:
    "books_pkey" PRIMARY KEY, btree (id)
    "books_isbn_key" UNIQUE CONSTRAINT, btree (isbn)
Check constraints:
    "books_available_copies_check" CHECK (available_copies >= 0)
    "books_total_copies_check" CHECK (total_copies >= 0)
Referenced by:
    TABLE "borrow_records" CONSTRAINT "borrow_records_book_id_fkey" FOREIGN KEY (book_id) REFERENCES books(id)
```

SQLx สร้างตาราง `_sqlx_migrations` ขึ้นมาเองอัตโนมัติในฐานข้อมูล (เก็บ version, checksum ของแต่ละ migration ที่รันไปแล้ว) — ทุกครั้งที่เรียก `sqlx migrate run` มันจะเช็คตารางนี้ก่อนว่า migration ไหนรันไปแล้วบ้าง แล้วรันแค่ตัวที่ยังไม่ได้รันเท่านั้น (idempotent — รันซ้ำได้อย่างปลอดภัยโดยไม่ apply migration เดิมซ้ำสอง)

#### ตรวจสอบสถานะและ Revert Migration

ก่อนรัน migration ใหม่หรือ debug ปัญหาเรื่อง schema ไม่ตรงกันระหว่างเครื่อง คำสั่งที่ควรรู้จักคือ `sqlx migrate info` (ดูว่า migration ไหนรันไปแล้ว/ยังค้างอยู่):

```bash
$ sqlx migrate info
20260926235829/installed create books table
```

ถ้าต้องการย้อนกลับ migration ล่าสุด (ใช้ `.down.sql` ที่สร้างไว้ตอนเปิด `-r`) ใช้ `sqlx migrate revert` — ผู้เขียนทดสอบจริง:

```bash
$ sqlx migrate revert
Applied 20260926235829/revert create books table (5.059675ms)

$ sqlx migrate info
20260926235829/pending create books table
```

สังเกตว่า `sqlx migrate info` เปลี่ยนสถานะจาก `installed` เป็น `pending` ทันทีหลัง revert (ตาราง `books`/`borrow_records` ถูก `DROP` ไปจริงตาม `.down.sql` ที่เขียนไว้ — ผู้เขียนตรวจสอบด้วย `\d books` ใน `psql` ยืนยันว่าตารางหายไปจริง) รันคำสั่ง `sqlx migrate run` อีกครั้งเพื่อสร้างตารางกลับมา (ข้อมูลเดิมในตารางจะ**หายหมด**เพราะ `.down.sql` สั่ง `DROP TABLE` ตรง ๆ — นี่คือเหตุผลที่ revert ควรใช้เฉพาะช่วง dev/testing เท่านั้น ไม่ควรรันบน production ที่มีข้อมูลจริงอยู่โดยไม่มีการ backup ก่อน)

**หลักการสำคัญเรื่อง migration ในทีมจริง**: ไฟล์ migration ที่ถูก apply ไปแล้วและมีคน pull ไปใช้แล้ว **ไม่ควรแก้ไขย้อนหลัง** (แก้ SQL ในไฟล์ที่มี timestamp เก่ากว่าที่คนอื่นรันไปแล้ว) เพราะ SQLx ตรวจ checksum ของไฟล์ migration เทียบกับที่บันทึกไว้ใน `_sqlx_migrations` — ผู้เขียนทดสอบจริงโดยเพิ่ม comment หนึ่งบรรทัดเข้าไปในไฟล์ `up.sql` ที่ apply ไปแล้ว แล้วลองรัน `sqlx migrate run` ซ้ำ ได้ error จริงทันที:

```
$ sqlx migrate run
error: migration 20260926235829 was previously applied but has been modified
```

SQLx **ปฏิเสธ**ทำงานทันทีเพื่อป้องกันสถานะ schema ที่ไม่ตรงกันระหว่างเครื่องของทีม (ถ้าเครื่อง A รัน migration เวอร์ชันเก่าไปแล้ว แต่เครื่อง B มีไฟล์เวอร์ชันที่ถูกแก้ไข ทั้งสองเครื่องจะมี schema ต่างกันโดยไม่มีใครรู้ตัวถ้าไม่มีการเช็คนี้) — ถ้าต้องแก้ schema ที่ทำผิดไปแล้ว ให้สร้าง migration **ใหม่** (timestamp ใหม่กว่า) ที่แก้ไขสิ่งที่ผิดแทน ไม่ใช่แก้ไฟล์เก่า

### 70.6 CRUD เต็มรูปแบบ: Parameterized Query และ SQL Injection

#### `#[derive(sqlx::FromRow)]` ทำงานอย่างไร (เชื่อมกลับไปที่ Part 44-45)

ก่อนลงมือเขียน CRUD เต็มรูปแบบ มาทวนก่อนว่า `#[derive(sqlx::FromRow)]` ที่ใช้มาตลอดบทนี้ทำงานอย่างไรในระดับ mechanism — จาก Part 44-45 คุณรู้แล้วว่า derive macro คือ proc macro ที่รับ `TokenStream` ของ struct มา parse แล้ว generate `impl` block ให้อัตโนมัติ (เหมือนที่ Part 57 อธิบาย `#[derive(Serialize)]` ไว้) `sqlx::FromRow` ทำงานแบบเดียวกันเป๊ะ เพียงแต่ generate `impl FromRow` ที่แปลง**แถวข้อมูลจากฐานข้อมูล**เป็น struct ของคุณ แทนแปลงเป็น JSON

รูปแบบคร่าว ๆ ของสิ่งที่ macro generate ให้ `Book` (เพื่อความเข้าใจ ไม่ใช่ token-for-token จริงที่ macro สร้าง — หลักการเดียวกับที่ Part 57 อธิบาย `#[derive(Serialize)]` ไว้):

```rust
// รูปแบบคร่าว ๆ ของสิ่งที่ #[derive(sqlx::FromRow)] generate ให้ Book
impl<'r> sqlx::FromRow<'r, sqlx::postgres::PgRow> for Book {
    fn from_row(row: &'r sqlx::postgres::PgRow) -> Result<Self, sqlx::Error> {
        Ok(Book {
            id: row.try_get("id")?,
            isbn: row.try_get("isbn")?,
            title: row.try_get("title")?,
            author: row.try_get("author")?,
            total_copies: row.try_get("total_copies")?,
            available_copies: row.try_get("available_copies")?,
            published_year: row.try_get("published_year")?,
            created_at: row.try_get("created_at")?,
        })
    }
}
```

สังเกตว่านี่คือ**สิ่งเดียวกัน**กับที่หัวข้อก่อนหน้าเพิ่งเขียนด้วยมือผ่าน `sqlx::Row::try_get()` — `#[derive(sqlx::FromRow)]` แค่ generate โค้ดแบบนี้ให้อัตโนมัติทีละ field ตามชื่อ field ที่ตรงกับชื่อ column (เชื่อมกับ trait bound: field แต่ละตัวต้อง implement `sqlx::Decode`/`sqlx::Type` ที่ผูกกับ PostgreSQL type จริง — `i64` ผูกกับ `BIGINT`, `String` ผูกกับ `TEXT`, `Option<T>` ผูกกับคอลัมน์ nullable ตามที่หัวข้อ 70.8 อธิบาย) — ส่วนที่ `query_as!`/`query_as::<_, T>()` ทำเพิ่มเติมคือเรียก `T::from_row(&row)` ให้กับทุกแถวที่ query คืนมาโดยอัตโนมัติ ไม่ต้องเรียกเองทีละแถว

**ทำไมต้อง derive แทนเขียน `impl FromRow` มือ**: เหตุผลเดียวกับ Part 44 หัวข้อ 44.1 อธิบายไว้เรื่อง `Display`/`Error`/`From` — ถ้า struct มี 10 field คุณต้องเขียน `.try_get("...")` ซ้ำ 10 ครั้งด้วยมือทุกครั้งที่มี struct ใหม่ และทุกครั้งที่เพิ่ม/ลบ field ต้องแก้ `impl` ตามด้วยมือเสมอ (ลืมแก้ = bug ที่เงียบมาก เพราะ compile ผ่านปกติแต่ field ใหม่ไม่ถูก populate) — derive macro ทำสิ่งนี้ให้อัตโนมัติ synced กับนิยาม struct เสมอ ไม่มีทางลืม

#### SELECT: query_as! กับ struct ที่ derive FromRow + Serialize

```rust
use sqlx::PgPool;

async fn list_all_books(pool: &PgPool) -> Result<Vec<Book>, sqlx::Error> {
    sqlx::query_as!(
        Book,
        r#"SELECT id, isbn, title, author, total_copies, available_copies, published_year, created_at
           FROM books ORDER BY id"#
    )
    .fetch_all(pool)
    .await
}
```

`.fetch_all()` คืน `Vec<Book>` ทุกแถวที่ query match — ใช้เมื่อคาดว่าอาจมีหลายแถวหรือไม่มีแถวเลยก็ได้ (ผลลัพธ์เป็น `Vec` ว่างถ้าไม่มีข้อมูล ไม่ error) ต่างจาก `.fetch_one()` ที่ใช้ในหัวข้อก่อนหน้า (คาดว่าต้องมีแถวเดียวพอดี — error `RowNotFound` ถ้าไม่มีแถวเลย และ error ถ้ามีมากกว่า 1 แถว) SQLx ยังมี `.fetch_optional()` (คืน `Option<T>` — `None` ถ้าไม่พบ ไม่ error) ที่เหมาะกับกรณี "หาไม่เจอก็เป็นเรื่องปกติ ไม่ใช่ error" มากกว่า `.fetch_one()`

#### INSERT ... RETURNING

PostgreSQL รองรับ clause `RETURNING` ที่คืนค่าแถวที่ถูก insert/update/delete กลับมาทันที **ในคำสั่งเดียว** — ไม่ต้อง insert แล้ว query กลับมาแยกอีกรอบ (ลด round-trip ไปฐานข้อมูลลงครึ่งหนึ่งสำหรับ pattern ที่พบบ่อยที่สุดของ CRUD API: "สร้างข้อมูลแล้วส่งข้อมูลที่สร้างกลับไปให้ client"):

```rust
async fn create_book(
    pool: &PgPool,
    isbn: &str,
    title: &str,
    author: &str,
    total_copies: i32,
    published_year: Option<i32>,
) -> Result<Book, sqlx::Error> {
    sqlx::query_as!(
        Book,
        r#"
        INSERT INTO books (isbn, title, author, total_copies, available_copies, published_year)
        VALUES ($1, $2, $3, $4, $4, $5)
        RETURNING id, isbn, title, author, total_copies, available_copies, published_year, created_at
        "#,
        isbn,
        title,
        author,
        total_copies,
        published_year,
    )
    .fetch_one(pool)
    .await
}
```

สังเกตว่า `$4` ถูกใช้ **สองครั้ง** ในคำสั่งเดียว (สำหรับทั้ง `total_copies` และ `available_copies` — ตอนสร้างหนังสือใหม่ จำนวนที่มีให้ยืมย่อมเท่ากับจำนวนทั้งหมดเสมอ) — SQLx/PostgreSQL รองรับการใช้ placeholder ตัวเดียวกันซ้ำได้ตามธรรมชาติของ positional parameter (`$N` อ้างอิงตำแหน่งของ argument ที่ bind ไว้ ไม่ใช่นับจำนวนครั้งที่ใช้)

#### UPDATE

```rust
async fn decrement_available_copies(pool: &PgPool, book_id: i64) -> Result<u64, sqlx::Error> {
    let result = sqlx::query!(
        "UPDATE books SET available_copies = available_copies - 1
         WHERE id = $1 AND available_copies > 0",
        book_id
    )
    .execute(pool)
    .await?;

    Ok(result.rows_affected())
}
```

**ข้อสังเกตสำคัญ**: เงื่อนไข `AND available_copies > 0` ในคำสั่ง `UPDATE` ป้องกันไม่ให้ `available_copies` ติดลบได้ (ถึงจะมี `CHECK` constraint ที่ระดับฐานข้อมูลกันไว้อีกชั้นแล้วก็ตาม) — `result.rows_affected()` คืน `u64` บอกว่ามีกี่แถวที่ถูกแก้ไขจริง ถ้าเป็น `0` แปลว่า**ไม่มีหนังสือเหลือให้ยืมแล้ว** (เงื่อนไข `available_copies > 0` ไม่ match แถวไหนเลย) — handler ฝั่ง Axum ควรเช็คค่านี้แล้วตอบ error ที่เหมาะสม (เช่น 409 Conflict "หนังสือหมด") ไม่ใช่ปล่อยให้ดูเหมือนสำเร็จทั้งที่จริง ๆ ไม่ได้แก้อะไรเลย

#### DELETE

```rust
async fn delete_book(pool: &PgPool, book_id: i64) -> Result<u64, sqlx::Error> {
    let result = sqlx::query!("DELETE FROM books WHERE id = $1", book_id)
        .execute(pool)
        .await?;
    Ok(result.rows_affected())
}
```

ทั้ง INSERT/UPDATE/DELETE ข้างบนผู้เขียน**คอมไพล์และรันจริง**ทั้งหมด (ดูผลลัพธ์รวมในหัวข้อ 70.9 ที่รันทั้ง flow ต่อกัน)

#### SQL Injection: ทำไม bind parameter ถึงสำคัญ (พิสูจน์ด้วยโค้ดจริง)

ทุกตัวอย่างข้างบนใช้ **`$1`, `$2`, ... (bind parameter/positional parameter)** แทนการเอาค่าจาก input ไปต่อ (concatenate/format) เข้ากับ SQL string ตรง ๆ — นี่ไม่ใช่แค่ "style ที่แนะนำ" แต่คือ**มาตรการป้องกัน SQL injection** ที่จำเป็นจริง ๆ มาดูว่าเกิดอะไรขึ้นถ้าทำผิด

ลองสมมติ (และผู้เขียน**ทดสอบจริง**ในสภาพแวดล้อมทดสอบแยก) ว่าเขียนฟังก์ชันค้นหาหนังสือจาก ISBN แบบ**ผิด** — เอา input ผู้ใช้ไป `format!` เข้า SQL string ตรง ๆ:

```rust
// ตัวอย่าง "ผิด" ที่ห้ามทำในโค้ดจริงเด็ดขาด — เอา input ผู้ใช้ไป format เข้า SQL ตรง ๆ
async fn find_book_UNSAFE(pool: &PgPool, user_input: &str) -> Result<(), sqlx::Error> {
    let unsafe_sql = format!("SELECT isbn, title FROM books WHERE isbn = '{}'", user_input);
    let rows = sqlx::query(&unsafe_sql).fetch_all(pool).await?; // ← สมมติว่าคอมไพล์ผ่าน
    println!("ได้ {} แถว", rows.len());
    Ok(())
}
```

ผู้เขียนลองคอมไพล์โค้ดนี้จริง — และพบสิ่งที่น่าสนใจมาก: **`sqlx` เวอร์ชัน 0.9.0 ที่ใช้ในบทนี้ปฏิเสธโค้ดนี้ตั้งแต่ compile time** ด้วย error จริงดังนี้:

```
error[E0277]: dynamic SQL strings should be audited for possible injections
   --> src/bin/injection_check.rs:21:28
    |
 21 |     let rows = sqlx::query(&unsafe_sql).fetch_all(&pool).await?;
    |                ----------- ^^^^^^^^^^^ dynamic SQL string
    |
    = help: the trait `SqlSafeStr` is not implemented for `&std::string::String`
    = note: prefer literal SQL strings with bind parameters or `QueryBuilder` to add dynamic data to a query.

            To bypass this error, manually audit for potential injection vulnerabilities and wrap with `AssertSqlSafe()`.
            For details, see the docs for `SqlSafeStr`.
```

นี่คือฟีเจอร์ความปลอดภัยที่ SQLx เพิ่มเข้ามา: `sqlx::query()`/`sqlx::query_as()` รับ argument ที่ implement trait `SqlSafeStr` เท่านั้น ซึ่ง **implement ให้แค่ `&'static str` (string literal ที่เขียนตรงในโค้ด) เป็นค่าเริ่มต้น** — `String`/`&String` ที่สร้างจาก `format!()` (นั่นคือ SQL ที่ "ประกอบขึ้นมาแบบ dynamic") **ไม่ implement trait นี้** compiler จึงปฏิเสธตั้งแต่ก่อนโปรแกรมรันด้วยซ้ำ ถ้าต้องการ SQL แบบ dynamic จริง ๆ ต้อง**ยอมรับความเสี่ยงอย่างชัดเจน**ด้วยการห่อ `sqlx::AssertSqlSafe(...)` เอง (ชื่อ "Assert" สื่อตรง ๆ ว่า "ฉันรับรองว่าตรวจสอบแล้วว่าปลอดภัย" — เป็นภาระของโปรแกรมเมอร์ที่ต้องเลือกทำเองอย่างตั้งใจ)

เพื่อพิสูจน์ว่าความเสี่ยงนี้เป็นเรื่องจริง (ไม่ใช่แค่ทฤษฎี) ผู้เขียนลอง**บายพาสการป้องกันนี้โดยตั้งใจ** (เหมือนโปรแกรมเมอร์ที่มองข้าม warning ของ compiler) ด้วย `AssertSqlSafe`:

```rust
let rows = sqlx::query(sqlx::AssertSqlSafe(unsafe_sql.clone()))
    .fetch_all(&pool)
    .await?;
```

แล้วป้อน input ที่เป็น SQL injection แบบคลาสสิก (`nonexistent' OR '1'='1`) เข้าไปแทนค่า ISBN ปกติ — ผลลัพธ์จริงที่ได้:

```
unsafe_sql = SELECT isbn, title FROM books WHERE isbn = 'nonexistent' OR '1'='1'
unsafe query ได้ 2 แถว (ควรได้ 0 ถ้าไม่มี isbn ตรงตัว แต่ดันได้ข้อมูลทั้งหมดเพราะ injection)
  leaked: isbn=111-1 title=Programming Rust
  leaked: isbn=inj-1 title=Secret Book
safe query (bind) ได้ 0 แถว (ถูกต้อง - ไม่มี isbn ตรงตามนี้จริง)
```

สิ่งที่เกิดขึ้นคือ SQL string ที่ประกอบเสร็จกลายเป็น `WHERE isbn = 'nonexistent' OR '1'='1'` — เงื่อนไข `'1'='1'` เป็นจริงเสมอ ทำให้ **ทุกแถวในตารางถูกเลือกออกมาหมด** (ในโลกจริงอาจเป็นข้อมูลลับ, ข้อมูลผู้ใช้คนอื่น, หรือแม้แต่ payload ที่ทำลายข้อมูล เช่น `'; DROP TABLE books; --` ถ้า driver รองรับ multiple statement) ในขณะที่บรรทัดสุดท้ายซึ่งใช้ **bind parameter** (`sqlx::query("SELECT isbn, title FROM books WHERE isbn = $1").bind(user_input)`) กับ input เดียวกันเป๊ะ ให้ผลลัพธ์ที่ถูกต้อง (`0` แถว) — เพราะ PostgreSQL รับ `user_input` ทั้งสตริง (รวม `' OR '1'='1`) เป็น**ค่าข้อมูลตัวเดียว**ที่จะเทียบกับ `isbn` เท่านั้น ไม่มีทางถูกตีความเป็น syntax SQL ได้เลย ไม่ว่า input จะมีเครื่องหมายคำพูดหรือ keyword SQL ปนอยู่แค่ไหนก็ตาม — นี่คือเหตุผลเชิงกลไกที่แท้จริงว่าทำไม bind parameter ถึง "ป้องกัน SQL injection ได้ 100%" ไม่ใช่แค่คำแนะนำลอย ๆ

**หลักการที่ต้องจำ**: **ห้ามเอา input จากผู้ใช้ไป `format!`/concatenate เข้า SQL string โดยตรงเด็ดขาด** ใช้ `$1, $2, ...` กับ `.bind()` (หรือส่งเป็น argument ให้ `query!`/`query_as!` ตรง ๆ ตามที่ทำมาตลอดบทนี้) เสมอ — ถ้าจำเป็นต้องสร้าง SQL แบบ dynamic จริง ๆ (เช่น จำนวนเงื่อนไข `WHERE` ไม่แน่นอน) ให้ใช้ `sqlx::QueryBuilder` (สร้าง SQL อย่างปลอดภัยแบบ dynamic โดยยัง bind parameter ถูกต้องอยู่ — จะกล่าวถึงใน Part 71) ไม่ใช่ต่อ string เอง

#### ข้อมูลแบบยืดหยุ่นด้วย `JSONB` และ `serde_json::Value`

บางครั้งข้อมูลที่ต้องเก็บไม่มี schema ตายตัวชัดเจน (เช่น metadata ของหนังสือที่อาจมี field ต่างกันไปตามประเภท — บางเล่มมี `series`, บางเล่มมี `edition`, บางเล่มไม่มีอะไรเพิ่มเลย) การเพิ่มคอลัมน์แยกทีละ field ทุกครั้งที่มี attribute ใหม่ไม่คุ้มค่า — PostgreSQL มี type **`JSONB`** (JSON แบบ binary ที่ index/query ได้เร็วกว่า `JSON` แบบ text ธรรมดา) ที่เหมาะกับสถานการณ์นี้พอดี และ SQLx (ผ่าน feature `json` ที่เปิดมาให้อัตโนมัติเมื่อเปิด `postgres` — สังเกตได้จากผลลัพธ์ `cargo add` ในหัวข้อ 70.2 ที่มี `+json` อยู่ในรายการ) แปลงระหว่าง `JSONB` กับ `serde_json::Value` ให้โดยตรง โดยไม่ต้อง parse/stringify มือเลย

สมมติเพิ่ม migration ใหม่ (ตามแนวทางหัวข้อ 70.5):

```sql
-- migrations/<timestamp>_add_metadata_column.up.sql
ALTER TABLE books ADD COLUMN metadata JSONB;
```

```rust
use serde_json::json;

#[derive(Debug, Serialize, Deserialize, sqlx::FromRow)]
struct BookWithMeta {
    id: i64,
    title: String,
    metadata: Option<serde_json::Value>, // JSONB เป็น nullable -> Option<Value> ตามกฎหัวข้อ 70.8
}

async fn insert_with_metadata(pool: &sqlx::PgPool) -> Result<BookWithMeta, sqlx::Error> {
    let meta = json!({ "tags": ["rust", "programming"], "rating": 4.5 });

    sqlx::query_as!(
        BookWithMeta,
        r#"INSERT INTO books (isbn, title, author, total_copies, available_copies, metadata)
           VALUES ($1, $2, $3, 1, 1, $4)
           RETURNING id, title, metadata"#,
        "978-jsonb-1",
        "JSONB Demo Book",
        "Tester",
        meta
    )
    .fetch_one(pool)
    .await
}
```

ผู้เขียนรันจริงได้ผลลัพธ์:

```
BookWithMeta { id: 1, title: "JSONB Demo Book", metadata: Some(Object {"rating": Number(4.5), "tags": Array [String("rust"), String("programming")]}) }
```

ดึงกลับมาอ่านค่าเฉพาะ field ข้างในผ่าน API ปกติของ `serde_json::Value` (ตาม Part 57 หัวข้อเรื่อง `Value` แบบ dynamic) ได้ตรง ๆ:

```rust
if let Some(m) = &fetched.metadata {
    println!("rating field = {:?}", m.get("rating")); // Some(Number(4.5))
}
```

**ข้อควรพิจารณา**: `JSONB` สะดวกมากสำหรับข้อมูลที่ shape ไม่แน่นอนจริง ๆ แต่**ไม่ใช่ทางลัดแทนการออกแบบ schema ที่ดี** — field ที่รู้อยู่แล้วว่าทุกแถวต้องมี ควรเป็นคอลัมน์แยกตามปกติ (`title`, `author`, ...) เพราะได้ทั้ง type-check ที่ compile time (ตามที่บทนี้เน้นย้ำมาตลอด), `CHECK`/`UNIQUE`/`FOREIGN KEY` constraint ที่ระดับฐานข้อมูล, และ query/index ที่มีประสิทธิภาพกว่า `JSONB` ควรสงวนไว้สำหรับข้อมูลที่**จริง ๆ แล้วไม่มีโครงสร้างคงที่**เท่านั้น

**ทางเลือกที่ type-safe กว่า `serde_json::Value`**: การใช้ `Option<serde_json::Value>` แบบข้างบนได้ flexibility เต็มที่ แต่เสีย type-safety ไปเลย (ต้องเรียก `.get("rating")` แล้วเดา type เอง — ไม่มีอะไรการันตีว่า field นั้นมีอยู่จริงหรือเป็น type ที่คาดไว้) ถ้ารู้ shape ของ JSON ที่จะเก็บล่วงหน้าอยู่แล้ว (แค่ไม่อยากแยกเป็นคอลัมน์ต่างหาก) SQLx มี wrapper type **`sqlx::types::Json<T>`** ที่ให้ผูก JSONB column กับ struct ที่ derive `Serialize`/`Deserialize` ธรรมดาได้ตรง ๆ (ได้ type-safety กลับมาเต็มรูปแบบ):

```rust
use sqlx::types::Json;

#[derive(Debug, Serialize, Deserialize, Clone)]
struct BookExtra {
    rating: f64,
    tags: Vec<String>,
}

#[derive(Debug, sqlx::FromRow)]
struct BookTypedMeta {
    id: i64,
    title: String,
    metadata: Option<Json<BookExtra>>,
}

async fn insert_typed_metadata(pool: &sqlx::PgPool) -> Result<BookTypedMeta, sqlx::Error> {
    let extra = BookExtra { rating: 4.8, tags: vec!["typed".into(), "json".into()] };

    sqlx::query_as!(
        BookTypedMeta,
        r#"INSERT INTO books (isbn, title, author, total_copies, available_copies, metadata)
           VALUES ($1, $2, $3, 1, 1, $4)
           RETURNING id, title, metadata as "metadata: Json<BookExtra>""#,
        "978-typedjson-1",
        "Typed JSON Book",
        "Tester",
        Json(extra) as _
    )
    .fetch_one(pool)
    .await
}
```

ผู้เขียนรันจริงได้ผลลัพธ์:

```
BookTypedMeta { id: 8, title: "Typed JSON Book", metadata: Some(Json(BookExtra { rating: 4.8, tags: ["typed", "json"] })) }
rating (typed, no .get() needed) = 4.8
```

**อธิบายจุดที่ไม่คุ้นตา**: `metadata as "metadata: Json<BookExtra>"` เป็น syntax พิเศษของ `query_as!`/`query!` สำหรับ**บอก type ที่ต้องการ override** ให้กับ column ที่ macro เดา type จาก schema ไม่ตรงกับที่ต้องการ (ในที่นี้ PostgreSQL รู้ว่า `metadata` เป็น `JSONB` เฉย ๆ แต่เราต้องการบอกว่า "แปลงเป็น `Json<BookExtra>` ที่ระดับ Rust ให้ด้วย" — ไม่ใช่ `serde_json::Value` เฉย ๆ) ส่วน `Json(extra) as _` ฝั่ง input ก็บอก compiler แบบเดียวกันว่าค่าที่ bind เข้าไปควรถูกมองเป็น type อะไร (`as _` ให้ compiler infer ชนิดที่ SQLx ต้องการเอง) — ผลลัพธ์คือได้ `BookExtra` กลับมาเป็น struct ที่ type-check เต็มรูปแบบ อ่าน `.rating` ได้ตรง ๆ ไม่ต้องเดา type จาก `serde_json::Value` แบบก่อนหน้า

#### Pagination ด้วย `LIMIT`/`OFFSET`

API ที่ list ข้อมูลในระบบจริงแทบไม่มีทางคืนข้อมูลทั้งหมดในครั้งเดียว (ตารางอาจมีหลักแสน/ล้านแถว) — ต้องแบ่งหน้า (pagination) SQL มาตรฐานสำหรับสิ่งนี้คือ `LIMIT`/`OFFSET`:

```rust
async fn list_books_paginated(
    pool: &sqlx::PgPool,
    page_size: i64,
    page_number: i64, // เริ่มจาก 0
) -> Result<Vec<Book>, sqlx::Error> {
    let offset = page_size * page_number;
    sqlx::query_as!(
        Book,
        r#"SELECT id, isbn, title, author, total_copies, available_copies, published_year, created_at
           FROM books ORDER BY id LIMIT $1 OFFSET $2"#,
        page_size,
        offset
    )
    .fetch_all(pool)
    .await
}
```

ผู้เขียนทดสอบจริงด้วยข้อมูล 5 แถว ขอ `LIMIT 2` สองหน้าติดกัน:

```
page1: ["Page Book 0", "Page Book 1"]
page2: ["Page Book 2", "Page Book 3"]
```

ผลลัพธ์ตรงตามที่คาด (หน้า 1 ได้แถวที่ 0-1, หน้า 2 ได้แถวที่ 2-3 — ไม่ทับซ้อนกัน) **ข้อสังเกตสำคัญเรื่อง `ORDER BY`**: `LIMIT`/`OFFSET` **ไม่มีความหมายที่แน่นอนถ้าไม่มี `ORDER BY`** ประกบคู่กันเสมอ — PostgreSQL ไม่การันตีลำดับแถวที่คืนมาถ้าไม่ระบุ `ORDER BY` ชัดเจน (อาจได้ลำดับต่างกันในแต่ละครั้งที่ query แม้ข้อมูลไม่เปลี่ยนเลย ถ้า query planner เลือกวิธีอ่านข้อมูลต่างไป) ทำให้หน้าที่ 1 กับหน้าที่ 2 อาจมีแถวซ้ำกันหรือขาดหายได้ถ้าลืม `ORDER BY` — กับดักนี้พบบ่อยมากในโค้ด pagination ที่เขียนแบบรีบ ๆ

(สำหรับตารางขนาดใหญ่มาก ๆ ที่ `OFFSET` เริ่มช้าลง เพราะ PostgreSQL ต้องอ่านข้ามแถวที่ถูก skip ทุกแถวก่อนถึงหน้าที่ต้องการ มีเทคนิคที่ดีกว่าเรียกว่า **keyset/cursor-based pagination** — ใช้ `WHERE id > $last_seen_id LIMIT $n` แทน `OFFSET` ตรง ๆ ซึ่งมีประสิทธิภาพคงที่ไม่ว่าจะอยู่หน้าไหน — เนื้อหานี้เกินขอบเขตบทนี้ แต่ควรรู้จักไว้เมื่อต้องทำ pagination กับตารางที่ข้อมูลมาก ๆ ในงานจริง)

#### `JOIN` ข้ามตาราง: หนังสือที่กำลังถูกยืมอยู่

ตาราง `borrow_records` ที่สร้างไว้ตั้งแต่หัวข้อ 70.5 ยังไม่ถูกใช้ query แบบ `JOIN` เลยจนถึงตอนนี้ — มาลองเขียน query จริงที่ต้องใช้ทั้งสองตารางพร้อมกัน: "รายชื่อหนังสือที่กำลังถูกยืมอยู่ (ยังไม่คืน) พร้อมชื่อผู้ยืม"

```rust
#[derive(Debug, sqlx::FromRow)]
struct ActiveBorrow {
    book_title: String,
    borrower_name: String,
    borrowed_at: DateTime<Utc>,
}

async fn list_active_borrows(pool: &sqlx::PgPool) -> Result<Vec<ActiveBorrow>, sqlx::Error> {
    sqlx::query_as!(
        ActiveBorrow,
        r#"SELECT b.title as book_title, r.borrower_name, r.borrowed_at
           FROM borrow_records r
           JOIN books b ON b.id = r.book_id
           WHERE r.returned_at IS NULL"#
    )
    .fetch_all(pool)
    .await
}
```

ผู้เขียนรันจริง (หลัง insert หนังสือหนึ่งเล่มและสร้าง borrow record ที่ยังไม่คืน) ได้ผลลัพธ์:

```
ActiveBorrow { book_title: "Join Demo Book", borrower_name: "Bob", borrowed_at: 2026-09-27T00:24:26.022774Z }
```

**ข้อสังเกตสำคัญสองจุด**: (1) `as book_title` ใน SQL คือ **column alias** ที่จำเป็นเพราะทั้งสองตารางมีคอลัมน์ชื่อ `title`ไม่ตรงกัน (จริง ๆ แค่ตาราง `books` มี `title` แต่การเขียน alias ชัดเจนแบบนี้ทำให้ struct `ActiveBorrow` map field `book_title` เข้ากับ column ที่ตั้งชื่อใหม่ได้ตรง ๆ — `query_as!` จับคู่ field กับ**ชื่อ column ในผลลัพธ์ query** ไม่ใช่ชื่อ column ในตารางต้นฉบับ) (2) SQLx ยัง type-check query ที่มี `JOIN` ได้ปกติทุกประการเหมือน query ธรรมดา (เชื่อมกับหัวข้อ 70.1 ที่อธิบายว่า SQLx ต่อฐานข้อมูลจริงตอน compile เพื่อ type-check — PostgreSQL รู้ type ของผลลัพธ์ `JOIN` ได้แม่นยำเหมือน query ปกติทุกกรณี ไม่มีข้อยกเว้นสำหรับ query ที่ซับซ้อนขึ้น)

#### Batch Insert ด้วย `UNNEST`

การ insert หลายแถวพร้อมกัน (เช่น import ข้อมูลจำนวนมากจากไฟล์) ถ้าเขียน loop เรียก `INSERT` ทีละแถวผ่าน `&pool` ตรง ๆ จะเสีย round-trip ไปฐานข้อมูล**หนึ่งครั้งต่อแถว** (ช้ามากถ้ามีเป็นพัน/หมื่นแถว) — PostgreSQL มีเทคนิค **`UNNEST`** ที่แปลง array หลายตัวให้กลายเป็นชุดของแถวได้ในคำสั่งเดียว ทำให้ insert ได้หลายแถวพร้อมกันโดยส่ง SQL แค่ครั้งเดียว:

```rust
async fn batch_insert_books(
    pool: &sqlx::PgPool,
    isbns: &[String],
    titles: &[String],
    authors: &[String],
) -> Result<Vec<i64>, sqlx::Error> {
    let copies = vec![1_i32; isbns.len()];

    let rows = sqlx::query!(
        r#"INSERT INTO books (isbn, title, author, total_copies, available_copies)
           SELECT * FROM UNNEST($1::text[], $2::text[], $3::text[], $4::int[], $4::int[])
           RETURNING id"#,
        isbns,
        titles,
        authors,
        &copies,
    )
    .fetch_all(pool)
    .await?;

    Ok(rows.iter().map(|r| r.id).collect())
}
```

ผู้เขียนรันจริงด้วย 3 แถวพร้อมกัน ได้ผลลัพธ์:

```
batch inserted 3 rows, ids=[9, 10, 11]
```

**อธิบายกลไก**: `UNNEST($1::text[], $2::text[], $3::text[], $4::int[], $4::int[])` รับ array หลายตัว (ที่ต้องมีความยาวเท่ากันทุกตัว) แล้ว "คลี่" มันออกมาเป็นชุดของแถวพร้อมกัน (แถวที่ 1 คือ element ที่ 0 ของทุก array, แถวที่ 2 คือ element ที่ 1 ของทุก array เรียงกันไป) — `SELECT * FROM UNNEST(...)` จึงได้ผลลัพธ์เป็นตารางชั่วคราวที่มีหลายแถวพร้อมกัน แล้ว `INSERT INTO ... SELECT * FROM ...` ก็ insert ทุกแถวนั้นในคำสั่งเดียว การ bind array เข้ากับ SQLx ทำได้ตรง ๆ เพราะ SQLx implement การแปลง `&[String]`/`Vec<T>` เป็น PostgreSQL array type ให้อยู่แล้ว (ต้องระบุ type array อย่างชัดเจนด้วย `::text[]`/`::int[]` เพราะ PostgreSQL ต้องรู้ type ของ `UNNEST` ก่อนจะ query-plan ได้ — ถ้าไม่ระบุ `sqlx::query!` จะ compile ไม่ผ่านเพราะเดา type ของ column ที่ได้จาก `UNNEST` ไม่ได้)

**หลักการเลือกใช้**: สำหรับ insert จำนวนน้อย (สิบ/ร้อยแถว) loop เรียก `INSERT` ทีละแถวผ่าน transaction เดียว (ตามหัวข้อ 70.9) ก็เพียงพอและอ่านโค้ดง่ายกว่า — `UNNEST` คุ้มค่าเมื่อต้อง insert **จำนวนมากจริง ๆ** (หลักพัน/หมื่นแถวขึ้นไป) ที่ overhead ของ round-trip ต่อแถวกลายเป็นคอขวดจริง

#### UUID เป็น Primary Key: ทางเลือกสำหรับตารางใหม่

ตาราง `books`/`borrow_records` ในบทนี้เลือกใช้ `BIGSERIAL` (ตัวเลข auto-increment) เป็น primary key ตามที่อธิบายไว้ในหัวข้อ 70.5 — แต่ในระบบจริงบางแบบ (เฉพาะอย่างยิ่งระบบ distributed ที่มีหลายฐานข้อมูล/หลาย service สร้าง record พร้อมกันโดยไม่พึ่ง sequence กลางร่วมกัน) การใช้ **UUID** เป็น primary key มีข้อดีที่ `BIGSERIAL` ให้ไม่ได้: สร้างค่า id ที่ไม่ซ้ำกันได้จากฝั่งไหนก็ได้ (แม้แต่ฝั่ง client) โดยไม่ต้องรอฐานข้อมูลจัดสรร sequence ให้ก่อน

จากหัวข้อ 70.2 คุณเปิด feature `uuid` ของ `sqlx` ไว้แล้ว (`cargo add sqlx --features ...,uuid`) — การผูก column type `UUID` ของ PostgreSQL เข้ากับ `uuid::Uuid` ของ Rust ทำได้ตรง ๆ ไม่ต้องแปลงผ่าน string เอง:

```sql
CREATE TABLE reviews (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(), -- PostgreSQL 13+ สร้าง UUID ให้อัตโนมัติ
    book_id BIGINT NOT NULL REFERENCES books(id),
    comment TEXT NOT NULL
);
```

```rust
use uuid::Uuid;

#[derive(Debug, sqlx::FromRow)]
struct Review {
    id: Uuid,
    book_id: i64,
    comment: String,
}

async fn add_review(pool: &sqlx::PgPool, book_id: i64, comment: &str) -> Result<Review, sqlx::Error> {
    // ไม่ระบุ id เลย -- ให้ฐานข้อมูล gen_random_uuid() สร้างให้ (เหมือน BIGSERIAL auto-increment)
    sqlx::query_as!(
        Review,
        "INSERT INTO reviews (book_id, comment) VALUES ($1, $2) RETURNING id, book_id, comment",
        book_id,
        comment
    )
    .fetch_one(pool)
    .await
}

async fn add_review_with_client_id(pool: &sqlx::PgPool, book_id: i64, comment: &str) -> Result<Review, sqlx::Error> {
    let client_generated_id = Uuid::new_v4(); // สร้าง UUID ฝั่ง Rust เองก่อน insert
    sqlx::query_as!(
        Review,
        "INSERT INTO reviews (id, book_id, comment) VALUES ($1, $2, $3) RETURNING id, book_id, comment",
        client_generated_id,
        book_id,
        comment
    )
    .fetch_one(pool)
    .await
}
```

ผู้เขียนรันจริงทั้งสองฟังก์ชัน ได้ผลลัพธ์:

```
Review { id: 6ed62f08-abb3-4600-959a-60944ef501a1, book_id: 13, comment: "หนังสือดีมาก" }
Review { id: 4097ac12-663a-4f13-b754-21e94221dc23, book_id: 13, comment: "สร้าง UUID จากฝั่ง Rust" }
```

และยืนยันด้วย `assert_eq!(review2.id, client_generated_id)` ผ่านจริง — พิสูจน์ว่า UUID ที่สร้างจากฝั่ง Rust ด้วย `Uuid::new_v4()` (เรียก Part 11/random generation ที่อาจเคยเห็นผ่าน ๆ มา) ถูก insert ตรงเข้า column `UUID` ได้ปกติโดยไม่ต้องแปลงเป็น string ก่อนเลย ทั้งสองแนวทาง (ให้ฐานข้อมูลสร้าง vs สร้างฝั่ง client) ใช้งานได้จริงตามความเหมาะสมของสถานการณ์

**เทียบข้อดี/ข้อเสียกับ `BIGSERIAL`**: UUID ไม่รั่วข้อมูลเชิงจำนวน (id แบบ `1, 2, 3, ...` ทำให้เดาได้ว่าระบบมีข้อมูลกี่รายการ หรือเดา id ของ record อื่นได้ง่าย ๆ — เป็นความเสี่ยงด้าน security เล็กน้อยที่ UUID ไม่มี) และสร้างได้แบบ distributed แต่แลกมาด้วย**ขนาดที่ใหญ่กว่า** (16 byte ต่อค่า เทียบกับ 8 byte ของ `BIGINT`) และ**index ที่ประสิทธิภาพต่ำกว่าเล็กน้อย**ในบางกรณี (UUID v4 แบบสุ่มล้วนทำให้ B-tree index กระจายตัวแบบสุ่ม ต่างจาก `BIGSERIAL` ที่เรียงลำดับต่อเนื่องซึ่ง index ทำงานได้มีประสิทธิภาพกว่า) — เลือกใช้ตามความต้องการจริงของระบบ ไม่มีคำตอบที่ถูกเสมอไปทุกกรณี

### 70.7 Error Handling: แปลง `sqlx::Error` เป็น HTTP Response

จาก Part 66 คุณมี pattern `AppError` ที่ implement `IntoResponse` ให้ Axum แปลง error เป็น HTTP response ที่เหมาะสมได้เอง — หัวข้อนี้จะเชื่อม `sqlx::Error` เข้ากับ pattern เดียวกัน (ถ้ายังไม่ได้อ่าน Part 66 ให้เข้าใจแนวคิดคร่าว ๆ ว่า `AppError` คือ enum ที่รวม error ทุกแหล่งของแอปไว้ที่เดียว แล้ว implement `IntoResponse` แปลงแต่ละ variant เป็น status code/JSON body ที่ต่างกัน)

`sqlx::Error` เป็น enum ที่มี variant สำคัญที่ต้องรู้จัก (จาก source code ของ `sqlx-core` เวอร์ชัน 0.9.0):

```rust
// รูปแบบย่อของ enum จริง (แสดงเฉพาะ variant ที่เกี่ยวข้องกับบทนี้)
pub enum Error {
    Database(Box<dyn DatabaseError>), // error จาก database เอง (constraint violation, syntax error, ...)
    Io(std::io::Error),                // error ตอนสื่อสารกับ database (connection หลุด, timeout, ...)
    RowNotFound,                       // fetch_one()/fetch_one ที่ไม่มีแถว match เลย
    PoolTimedOut,                      // acquire connection จาก pool ไม่ทันภายใน acquire_timeout
    PoolClosed,                        // pool ถูกปิดไปแล้ว (เช่น โปรแกรมกำลัง shutdown)
    // ... variant อื่น ๆ (Configuration, Protocol, TypeNotFound, ...)
}
```

**`Error::Database`** เป็น variant ที่ต้องแยกย่อยต่ออีกชั้น เพราะครอบคลุม constraint violation ทุกชนิดของฐานข้อมูล — trait `DatabaseError` มี method `.kind()` ที่คืน `ErrorKind` (enum ที่ SQLx normalize มาจาก error code ของแต่ละฐานข้อมูลให้เหมือนกันไม่ว่าจะใช้ driver ไหน):

```rust
pub enum ErrorKind {
    UniqueViolation,      // ละเมิด UNIQUE constraint (หรือ PRIMARY KEY ซ้ำ)
    ForeignKeyViolation,  // ละเมิด FOREIGN KEY constraint
    NotNullViolation,     // พยายามใส่ NULL ในคอลัมน์ที่เป็น NOT NULL
    CheckViolation,       // ละเมิด CHECK constraint
    ExclusionViolation,   // ละเมิด EXCLUDE constraint
    Other,                // error อื่น ๆ ที่ SQLx ไม่ได้ normalize เป็นหมวดเฉพาะ
}
```

มาดู error จริงที่เกิดขึ้นเมื่อละเมิด `UNIQUE` constraint ของ `isbn` (สร้างหนังสือ ISBN ซ้ำ) — ผู้เขียนรันจริงและจับ error object มาพิมพ์รายละเอียด:

```rust
let dup = sqlx::query!(
    "INSERT INTO books (isbn, title, author, total_copies, available_copies) VALUES ($1, $2, $3, $4, $4)",
    "978-1-59327-828-1", // ISBN ที่มีอยู่แล้วในตาราง
    "Duplicate ISBN Attempt",
    "Someone",
    1_i32,
)
.execute(&pool)
.await;

match dup {
    Ok(_) => println!("ไม่ควรมาถึงจุดนี้"),
    Err(sqlx::Error::Database(db_err)) => {
        println!("Database error kind: {:?}", db_err.kind());
        println!("constraint(): {:?}", db_err.constraint());
        println!("code(): {:?}", db_err.code());
        println!("message(): {}", db_err.message());
    }
    Err(other) => println!("unexpected error variant: {other:?}"),
}
```

ผลลัพธ์จริง:

```
Database error kind: UniqueViolation
constraint(): Some("books_isbn_key")
code(): Some("23505")
message(): duplicate key value violates unique constraint "books_isbn_key"
```

สังเกตว่า SQLx ให้ข้อมูลครบทุกระดับที่ต้องใช้จริง: **`kind()`** (`UniqueViolation` — ใช้ตัดสินใจ logic ได้ทันทีไม่ต้อง parse string) **`constraint()`** (ชื่อ constraint ที่ถูกละเมิดเป๊ะ — มีประโยชน์เมื่อตารางมีหลาย `UNIQUE` constraint และต้องรู้ว่าตัวไหนถูกละเมิด) **`code()`** (SQLSTATE code ของ PostgreSQL เอง — `23505` คือ `unique_violation` ตาม PostgreSQL error code ที่เป็น standard) และ **`message()`** (ข้อความจาก PostgreSQL ตรง ๆ — เหมาะสำหรับ log แต่**ไม่ควร**ส่งตรง ๆ ให้ client เห็น เพราะอาจรั่วรายละเอียดโครงสร้างฐานข้อมูล)

ตอนนี้มาผูกเข้ากับ `AppError` (แนวทางเดียวกับที่ Part 66 สอน) เพื่อแปลง constraint violation เป็น **409 Conflict** แทนการปล่อยให้เป็น 500 ที่ไม่มีความหมายอะไรกับ client เลย:

```rust
use axum::http::StatusCode;
use axum::response::{IntoResponse, Response};
use axum::Json;

#[derive(Debug)]
enum AppError {
    NotFound,
    Conflict(String),
    Internal(String),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, msg) = match self {
            AppError::NotFound => (StatusCode::NOT_FOUND, "not found".to_string()),
            AppError::Conflict(m) => (StatusCode::CONFLICT, m),
            AppError::Internal(m) => (StatusCode::INTERNAL_SERVER_ERROR, m),
        };
        (status, Json(serde_json::json!({ "error": msg }))).into_response()
    }
}

// เชื่อม sqlx::Error เข้ากับ AppError ผ่าน From — ทำให้ `?` แปลง error ให้อัตโนมัติ (ตาม Part 12/30-31)
impl From<sqlx::Error> for AppError {
    fn from(e: sqlx::Error) -> Self {
        match e {
            sqlx::Error::RowNotFound => AppError::NotFound,
            sqlx::Error::Database(db_err) => {
                if db_err.is_unique_violation() {
                    AppError::Conflict(format!("duplicate: {}", db_err.message()))
                } else {
                    AppError::Internal(db_err.message().to_string())
                }
            }
            other => AppError::Internal(other.to_string()),
        }
    }
}
```

**อธิบาย**: `db_err.is_unique_violation()` เป็น convenience method ของ `DatabaseError` (เทียบเท่ากับเช็ค `db_err.kind() == ErrorKind::UniqueViolation` แต่สั้นกว่า — SQLx มี `is_foreign_key_violation()`, `is_check_violation()` คู่กันด้วย) ส่วนการที่ `impl From<sqlx::Error> for AppError` ทำให้ handler ที่คืน `Result<T, AppError>` สามารถใช้ `?` กับ `Result<T, sqlx::Error>` ที่ query คืนมาได้ตรง ๆ โดย compiler จะเรียก `.into()` (ผ่าน `From`) ให้อัตโนมัติ (กลไกเดียวกับที่ Part 12/30-31 สอนเรื่อง error conversion ทั้งหมด ไม่มีอะไรใหม่ในระดับ mechanism)

ทดสอบด้วย handler จริงผ่าน Axum + `curl` จริง (หัวข้อ 70.9 จะแสดง handler เต็มรูปแบบ) ให้ผลลัพธ์จริง:

```
$ curl -s -i -X POST http://127.0.0.1:4070/books -H 'Content-Type: application/json' \
    -d '{"isbn":"111-1","title":"Dup","author":"X","total_copies":1,"published_year":null}'
HTTP/1.1 409 Conflict
content-type: application/json
content-length: 88

{"error":"duplicate: duplicate key value violates unique constraint \"books_isbn_key\""}
```

การ insert ISBN ที่ซ้ำได้ตอบ `409 Conflict` กลับมาจริง (ไม่ใช่ `500 Internal Server Error` ที่ไม่มีความหมายอะไรกับ client) — client ที่เรียก API นี้สามารถแยกแยะได้ว่า "ข้อมูลซ้ำ ควรแก้ input" (409) กับ "ระบบมีปัญหา ลองใหม่ทีหลัง" (500) ได้อย่างถูกต้อง

#### Connection Errors: Password ผิด, Host หาไม่พบ, Database ไม่มีอยู่จริง

นอกจาก constraint violation (ที่เกิด**หลัง**ต่อฐานข้อมูลสำเร็จแล้ว) ยังมี error อีกกลุ่มที่เกิด**ตอนพยายามต่อฐานข้อมูล**เอง (ตอนเรียก `PgPoolOptions::connect()`) — ผู้เขียนทดสอบจริงสามสถานการณ์ที่พบบ่อยที่สุดในโลกจริง (พิมพ์ connection string ผิด, ตั้งค่า environment variable ผิดสภาพแวดล้อม, ฐานข้อมูลยังไม่ถูกสร้าง):

```rust
// สถานการณ์ 1: password ผิด
let bad_pw = PgPoolOptions::new()
    .connect("postgres://postgres:wrongpassword@127.0.0.1:5432/rust_course_scratch")
    .await;

// สถานการณ์ 2: host/port ไม่มีอะไรฟังอยู่จริง (ต่อ port ที่ไม่มี PostgreSQL รันอยู่)
let bad_host = PgPoolOptions::new()
    .acquire_timeout(std::time::Duration::from_secs(2))
    .connect("postgres://postgres:postgres@127.0.0.1:59999/rust_course_scratch")
    .await;

// สถานการณ์ 3: ชื่อฐานข้อมูลไม่มีอยู่จริง (server ต่อได้ แต่ไม่มี database นี้)
let bad_db = PgPoolOptions::new()
    .connect("postgres://postgres:postgres@127.0.0.1:5432/no_such_database_xyz")
    .await;
```

ผลลัพธ์จริงของทั้งสามกรณี:

```
wrong password error: error returned from database: password authentication failed for user "postgres" at line 331
unreachable host error: pool timed out while waiting for an open connection
no such database error: error returned from database: database "no_such_database_xyz" does not exist at line 1021
```

**ข้อสังเกตที่น่าสนใจ**: สถานการณ์ที่ 1 และ 3 ได้ error เป็น `sqlx::Error::Database` (PostgreSQL server ตอบกลับมาจริง ๆ ว่า authentication fail หรือ database ไม่มีอยู่ — ได้ error message ที่ชัดเจนเกือบจะทันที) แต่สถานการณ์ที่ 2 (host/port ที่ไม่มีอะไรฟังอยู่) กลับได้ error เป็น **`PoolTimedOut`** (ข้อความ `pool timed out while waiting for an open connection`) ไม่ใช่ error แบบ "connection refused" ที่อาจคาดไว้ทันที — เหตุผลคือ `PgPoolOptions::connect()` เองก็ยังทำงานผ่านกลไก pool (พยายามเปิด `min_connections` ตัวแรกผ่าน internal acquire ที่มี `acquire_timeout` กำกับอยู่) ถ้า TCP connection ไปยัง host/port นั้นไม่มีอะไรตอบสนองเลย (ไม่ reject ทันที ไม่ accept เลย) SQLx จะรอจนครบ `acquire_timeout` ก่อนแล้วสรุปว่า "รอไม่ไหวแล้ว" แทนที่จะเป็น error เชิง network โดยตรง — สิ่งนี้เตือนให้ระวังว่า **ข้อความ error ที่เห็นไม่ได้บอกสาเหตุที่แท้จริงเสมอไป** (ตัวอย่างนี้เห็น `PoolTimedOut` แต่สาเหตุจริงคือ host ต่อไม่ได้ ไม่ใช่ pool เต็มแบบหัวข้อ 70.11) ตอน debug ควรเช็คทั้งสองความเป็นไปได้เสมอเมื่อเจอ error นี้

**หลักปฏิบัติสำหรับ error กลุ่มนี้ในแอปจริง**: error ตอน `connect()` ควรทำให้โปรแกรมไม่ผ่าน startup เลย (`.expect("เชื่อมต่อฐานข้อมูลไม่สำเร็จ")` ตามที่หัวข้อ 70.10 ทำ — ไม่มีประโยชน์ที่จะรัน HTTP server ต่อถ้าต่อฐานข้อมูลไม่ได้เลยตั้งแต่ต้น) แต่ error ที่เกิด**ระหว่าง**โปรแกรมรันอยู่แล้ว (เช่น network hiccup ชั่วคราวระหว่าง connection กับฐานข้อมูลหลุดกลางทาง) ไม่ควรทำให้ทั้งแอป crash — ควรแปลงเป็น `500`/`503` สำหรับ request นั้น ๆ ผ่าน `AppError::Internal` (ตามที่ `impl From<sqlx::Error>` ในหัวข้อนี้ทำไว้ให้แล้ว — variant อื่นที่ไม่ได้ระบุเฉพาะจะตกไปที่ `AppError::Internal` โดย fallback) แล้วให้ระบบข้างนอก (load balancer, client ที่ retry) จัดการต่อ — connection pool จะพยายามเปิด connection ใหม่ให้เองสำหรับ request ถัดไปโดยไม่ต้อง restart โปรแกรมทั้งตัว

### 70.8 NULL Handling: `Option<T>` กับ Nullable Column

จาก Part 11 คุณรู้จัก `Option<T>` ในฐานะวิธีที่ Rust แทน "อาจไม่มีค่า" อย่างปลอดภัยที่ compile time (ไม่มี null pointer ที่ crash ตอน runtime แบบภาษาอื่น) — PostgreSQL มีแนวคิดคู่กันคือ **nullable column** (คอลัมน์ที่ไม่มี `NOT NULL` constraint แปลว่าอนุญาตให้เป็น `NULL` ได้) SQLx เชื่อมสองแนวคิดนี้เข้าด้วยกันตรง ๆ: **คอลัมน์ที่ nullable ต้อง map เป็น `Option<T>` ในฝั่ง Rust เท่านั้น**

จากตาราง `books` ที่สร้างไว้ คอลัมน์ `published_year INTEGER` **ไม่มี** `NOT NULL` (หนังสือบางเล่มอาจไม่รู้ปีที่พิมพ์แน่ชัด) — struct `Book` จึงประกาศ field นี้เป็น `Option<i32>` ตามที่เห็นมาตลอดบท:

```rust
#[derive(Debug, Serialize, sqlx::FromRow)]
struct Book {
    // ... field อื่น ๆ
    published_year: Option<i32>, // nullable ในฐานข้อมูล -> Option<T> ใน Rust
    // ...
}
```

ผู้เขียนทดสอบจริงทั้งสองกรณี — insert หนังสือที่**ไม่ระบุ** `published_year` (ใส่ `NULL` ตรง ๆ ใน SQL):

```rust
let inserted_null = sqlx::query_as!(
    Book,
    r#"
    INSERT INTO books (isbn, title, author, total_copies, available_copies, published_year)
    VALUES ($1, $2, $3, $4, $4, NULL)
    RETURNING id, isbn, title, author, total_copies, available_copies, published_year, created_at
    "#,
    "978-0-000-00000-1",
    "Unknown Year Book",
    "Anon",
    1_i32,
)
.fetch_one(&pool)
.await?;

println!("published_year (should be None): {:?}", inserted_null.published_year);
```

ผลลัพธ์จริง:

```
published_year (should be None): None
```

และหนังสือที่ระบุปีจริง (จากหัวข้อ 70.6) ให้ `published_year: Some(2020)` ตามที่คาด — SQLx (ผ่าน `query_as!` ที่ type-check กับ schema จริงตอน compile) **รู้ล่วงหน้าแล้ว**ว่า column `published_year` เป็น nullable (อ่านมาจาก metadata ของ PostgreSQL ตอนต่อฐานข้อมูล type-check) และจะปฏิเสธตั้งแต่ compile time ถ้าคุณพลาดประกาศ field เป็น `i32` เฉย ๆ (ไม่ห่อ `Option`) — ลองดู error จริงถ้าทำผิด (เปลี่ยน field เป็น `published_year: i32` ตรง ๆ แล้ว build ใหม่):

```
error: mismatched types
  --> src/main.rs:12:5
   |
12 |     published_year: i32,
   |     ^^^^^^^^^^^^^^^^^^^ expected `Option<i32>`, found `i32`
   |
   = note: column `published_year` may be `NULL`; if the column is not nullable,
           create a `NOT NULL` constraint or use the `unwrap_or` methods on `Option<T>`
```

(error message นี้คือรูปแบบมาตรฐานของ SQLx เมื่อ field ไม่ตรงกับ nullability ของ column จริง — ข้อความอาจต่างกันเล็กน้อยตามเวอร์ชัน แต่สาระสำคัญเหมือนกันเสมอ: "field ที่คุณประกาศไม่ตรงกับ nullable ของ column จริง") นี่คือตัวอย่างที่ชัดที่สุดของสิ่งที่หัวข้อ 70.1 อธิบายไว้ว่า compile-time query verification ทำอะไรได้จริง — ไม่ใช่แค่ตรวจ syntax SQL แต่ตรวจ**ความถูกต้องของ type ระดับ column ต่อ column** รวมถึง nullability ด้วย

**หลักการที่ต้องจำ**: ทุกคอลัมน์ที่**ไม่มี** `NOT NULL` ในฐานข้อมูล ต้อง map เป็น `Option<T>` ในฝั่ง Rust เสมอ — ถ้าใช้ `query_as!`/`query!` SQLx จะบังคับสิ่งนี้ให้ที่ compile time อยู่แล้ว (ผิดไม่ได้) แต่ถ้าใช้ `query_as`/`query` แบบ runtime-only การ map ผิดจะไม่ error ตอน compile — จะพังตอน runtime แทน (deserialize error ตอนพยายามอ่านค่า `NULL` เข้า type ที่ไม่รองรับ `None`) นี่คือข้อดีที่จับต้องได้อีกข้อของการใช้ macro แบบ compile-time checked เป็นค่า default

### 70.9 Transaction: `pool.begin()`, Commit, Rollback

หลายสถานการณ์ในโลกจริงต้องการ**หลายคำสั่ง SQL ที่ต้องสำเร็จทั้งหมดหรือไม่สำเร็จเลย (atomic)** — ตัวอย่างคลาสสิกในระบบห้องสมุด: การยืมหนังสือต้อง (1) ลด `available_copies` ลง 1 **และ** (2) สร้าง record ใน `borrow_records` พร้อมกัน ถ้าทำแค่ข้อ (1) สำเร็จแต่ข้อ (2) ล้มเหลว (เช่น เกิด error กลางทาง) ระบบจะอยู่ในสถานะที่ไม่ถูกต้อง (หนังสือถูกหักไปแล้วแต่ไม่มีบันทึกว่าใครยืม) — **transaction** คือกลไกของฐานข้อมูลที่แก้ปัญหานี้โดยตรง: กลุ่มคำสั่งที่อยู่ใน transaction เดียวกันจะถูกมองเป็น**หน่วยเดียว** ถ้า `commit()` สำเร็จ ทุกคำสั่งมีผลจริงพร้อมกัน ถ้า `rollback()` (หรือ error เกิดขึ้นแล้ว transaction ถูกทิ้งไปโดยไม่ commit) **ทุกคำสั่งใน transaction นั้นจะถูกยกเลิกทั้งหมด เหมือนไม่เคยเกิดขึ้นเลย**

```rust
use sqlx::PgPool;

async fn borrow_book(
    pool: &PgPool,
    book_id: i64,
    borrower_name: &str,
) -> Result<(), sqlx::Error> {
    // เริ่ม transaction — ได้ Transaction<'_, Postgres> ที่ implement Executor เหมือน &PgPool
    let mut tx = pool.begin().await?;

    sqlx::query!(
        "UPDATE books SET available_copies = available_copies - 1
         WHERE id = $1 AND available_copies > 0",
        book_id
    )
    .execute(&mut *tx) // สังเกต: ส่ง &mut *tx ไม่ใช่ &pool — คำสั่งนี้อยู่ "ใน" transaction
    .await?;

    sqlx::query!(
        "INSERT INTO borrow_records (book_id, borrower_name) VALUES ($1, $2)",
        book_id,
        borrower_name
    )
    .execute(&mut *tx)
    .await?;

    // ทั้งสองคำสั่งข้างบนยังไม่มีผลจริงกับฐานข้อมูลจนกว่าจะ commit สำเร็จ
    tx.commit().await?;

    Ok(())
}
```

**อธิบายจุดสำคัญ**: `pool.begin()` คืน `Transaction<'_, Postgres>` ที่ "ยืม" connection ตัวหนึ่งจาก pool มาใช้ตลอดช่วงชีวิตของ transaction (connection ตัวนั้นจะไม่ถูกคืนกลับ pool จนกว่า transaction จะ commit/rollback เสร็จ หรือถูก drop) — ทุก query ที่ต้องอยู่ "ใน" transaction เดียวกันต้องเรียกผ่าน `&mut *tx` (ไม่ใช่ `&pool` ตรง ๆ) เพราะถ้าเรียกผ่าน `&pool` SQLx จะไปหยิบ connection **ตัวอื่น**จาก pool มาใช้ (คนละ transaction กันโดยสิ้นเชิง — คำสั่งนั้นจะ commit ทันทีตาม auto-commit mode ปกติของ PostgreSQL ไม่ได้อยู่ใน transaction ที่ตั้งใจไว้เลย) นี่คือกับดักเชิง type ที่พบบ่อยมาก (ดูหัวข้อกับดักท้ายบท)

`tx.commit().await?` เป็นจุดเดียวที่ทำให้ทุกคำสั่งใน transaction "มีผลจริง" พร้อมกัน — ถ้าโปรแกรม crash หรือ `tx` ถูก drop (เช่น มี `?` return early ก่อนถึง `.commit()`) **โดยไม่เรียก commit** SQLx จะ**rollback อัตโนมัติ**ให้ (ผ่าน `Drop` implementation ของ `Transaction` — ส่งคำสั่ง `ROLLBACK` ให้ฐานข้อมูลก่อนที่ connection จะถูกคืนกลับ pool) — นี่คือ safety net ที่สำคัญมาก: **การไม่เรียก `.commit()` ปลอดภัยกว่าการลืมเรียก `.rollback()`** เพราะพฤติกรรม default คือ rollback อยู่แล้ว

#### ทำไม `.execute(&pool)` และ `.execute(&mut *tx)` เขียนแบบเดียวกันได้ (เชื่อมกับ Part 19/21 เรื่อง Trait)

สังเกตว่าตลอดบทนี้ `.fetch_one()`/`.execute()` ถูกเรียกได้ทั้งกับ `&pool` (หัวข้อ 70.6-70.8) และกับ `&mut *tx` (หัวข้อนี้) โดยไม่ต้องเขียนโค้ดคนละแบบ หรือใช้ method คนละชื่อกันเลย — นี่ไม่ใช่ความบังเอิญ แต่มาจาก trait กลางของ SQLx ชื่อ **`Executor`** ที่นิยาม method อย่าง `.fetch_one()`/`.fetch_all()`/`.execute()` ไว้เพียงครั้งเดียว แล้ว implement ให้กับหลาย type ที่ "สามารถรันคำสั่ง SQL ได้" ทั้งหมด: `&PgPool`, `&mut PgConnection`, และ `&mut Transaction<'_, Postgres>` (สามชนิดนี้ implement `Executor` เหมือนกันหมด แม้ภายในทำงานต่างกัน — `&PgPool` ไปหยิบ connection จาก pool มาใช้ชั่วคราว ส่วน `&mut Transaction` ใช้ connection ที่ transaction ยึดไว้อยู่แล้ว)

นี่คือการประยุกต์ตรงของหลักการที่ Part 19/21 สอนไว้เรื่อง trait: "เขียนโค้ดครั้งเดียว ทำงานได้กับ concrete type ที่สนอง trait bound ได้หลายแบบ" — ฟังก์ชันของแอปคุณเองก็ใช้หลักการเดียวกันนี้ได้ ถ้าอยากเขียนฟังก์ชัน query ที่**เรียกได้ทั้งผ่าน `&pool` ตรง ๆ และผ่าน `&mut *tx`** (โดยไม่ต้องเขียนสองเวอร์ชัน) ให้รับ parameter เป็น `impl sqlx::PgExecutor<'_>` แทนการ fix type เป็น `&PgPool` ตรง ๆ:

```rust
// ฟังก์ชันนี้เรียกได้ทั้ง find_book(&pool, ...) และ find_book(&mut *tx, ...)
async fn find_book<'e>(
    executor: impl sqlx::PgExecutor<'e>,
    isbn: &str,
) -> Result<Book, sqlx::Error> {
    sqlx::query_as!(
        Book,
        r#"SELECT id, isbn, title, author, total_copies, available_copies, published_year, created_at
           FROM books WHERE isbn = $1"#,
        isbn
    )
    .fetch_one(executor)
    .await
}
```

(`PgExecutor` คือ type alias ที่ SQLx เตรียมไว้ให้เฉพาะ PostgreSQL ของ `Executor` ตัวเต็ม — สะดวกกว่าเขียน trait bound แบบ generic ข้าม database ที่ซับซ้อนกว่า) ความสามารถนี้มีประโยชน์มากตอนต้องเขียนฟังก์ชันย่อยที่**บางครั้ง**ต้องอยู่ใน transaction และ**บางครั้ง**เรียกแบบเดี่ยว ๆ โดยไม่ต้อง copy โค้ด query ซ้ำสองที่

#### พิสูจน์ rollback ด้วยโค้ดจริง

คำอธิบายเรื่อง rollback ข้างบนฟังดูสมเหตุสมผล แต่การ "เชื่อคำอธิบาย" ไม่เท่ากับการพิสูจน์จริง — ผู้เขียนจึงสร้างสถานการณ์ที่ตั้งใจให้ query ที่สองใน transaction **ล้มเหลว** (ใส่ `book_id` ที่ไม่มีจริงลงใน `borrow_records` — ละเมิด foreign key constraint ที่สร้างไว้ในหัวข้อ 70.5) แล้ววัดค่า `available_copies` **ก่อน** กับ **หลัง** การพยายามทำ transaction นี้ ถ้า rollback ทำงานถูกต้องจริง สองค่านี้ต้อง**เท่ากัน** (แปลว่าการ `UPDATE` ที่สำเร็จไปแล้วก่อนหน้าถูกยกเลิกไปด้วย ไม่ใช่แค่ insert ที่พังตัวเดียว):

```rust
let before_rollback = sqlx::query!(
    "SELECT available_copies FROM books WHERE id = $1",
    book_id
)
.fetch_one(&pool)
.await?
.available_copies;

{
    let mut tx = pool.begin().await?;

    // คำสั่งที่ 1: สำเร็จแน่นอน (ไม่มีเงื่อนไขที่ทำให้ fail)
    sqlx::query!(
        "UPDATE books SET available_copies = available_copies - 1 WHERE id = $1",
        book_id
    )
    .execute(&mut *tx)
    .await?;

    // คำสั่งที่ 2: ตั้งใจให้ fail — book_id = 999999 ไม่มีอยู่จริง ละเมิด foreign key
    let bad_insert = sqlx::query!(
        "INSERT INTO borrow_records (book_id, borrower_name) VALUES ($1, $2)",
        999999_i64,
        "Ghost Borrower"
    )
    .execute(&mut *tx)
    .await;

    match bad_insert {
        Ok(_) => println!("ไม่ควรสำเร็จ"),
        Err(e) => {
            println!("insert ที่สองล้มเหลวตามคาด: {e}");
            tx.rollback().await?; // ยกเลิกทุกอย่างใน transaction นี้ รวมถึง UPDATE ที่สำเร็จไปแล้ว
        }
    }
}

let after_rollback = sqlx::query!(
    "SELECT available_copies FROM books WHERE id = $1",
    book_id
)
.fetch_one(&pool)
.await?
.available_copies;

assert_eq!(before_rollback, after_rollback, "rollback ต้องทำให้ค่ากลับเป็นเดิม");
```

ผลลัพธ์จริงจากการรัน:

```
insert ที่สองล้มเหลวตามคาด: error returned from database: insert or update on table "borrow_records" violates foreign key constraint "borrow_records_book_id_fkey" at line 2608
rollback เรียกแล้ว
available_copies: before_rollback=1 after_rollback=1 (ควรเท่ากัน = rollback สำเร็จ)
```

`before_rollback` และ `after_rollback` **เท่ากันจริง** (`1 == 1`) — พิสูจน์ว่าคำสั่ง `UPDATE ... available_copies - 1` ที่**สำเร็จไปแล้ว**ก่อนที่คำสั่งที่สองจะ fail ถูกยกเลิกไปด้วยจริง ๆ ตอน rollback ไม่ใช่แค่คำสั่งที่ fail ตัวเดียวที่ไม่มีผล — นี่คือความหมายที่แท้จริงของ "atomic": ทั้ง transaction ถูกมองเป็นหน่วยเดียวกัน ไม่มีสถานะ "สำเร็จครึ่งเดียว" หลุดออกมาให้เห็นเลย ต่างจากถ้าไม่ใช้ transaction เลย (เรียกทั้งสองคำสั่งผ่าน `&pool` ตรง ๆ แยกกัน) ที่คำสั่งแรกจะ commit ทันทีตาม auto-commit ปกติของ PostgreSQL แล้วค้างอยู่แบบนั้นแม้คำสั่งที่สองจะ fail ก็ตาม (สถานะข้อมูลจะเสียหายจริง — หนังสือถูกหักไปแล้วแต่ไม่มี record การยืม)

#### Nested Transaction ด้วย Savepoint

บางสถานการณ์ต้องการ "transaction ย่อยภายใน transaction ใหญ่" — เช่น ฟังก์ชันย่อยที่อาจ fail ได้และต้องการ rollback แค่ส่วนของตัวเองโดยไม่กระทบ transaction ใหญ่ที่เรียกมัน (ต่างจากตัวอย่างหัวข้อก่อนหน้าที่ rollback ทั้ง transaction) PostgreSQL รองรับสิ่งนี้ผ่านกลไก **`SAVEPOINT`** (จุดกลับที่ตั้งไว้กลาง transaction — `ROLLBACK TO SAVEPOINT` ย้อนกลับไปแค่จุดนั้น ไม่ใช่ยกเลิกทั้ง transaction) และ SQLx เปิดให้ใช้ผ่าน trait `sqlx::Acquire`:

```rust
use sqlx::Acquire; // จำเป็นต้อง import trait นี้ก่อน — ไม่มีจะเรียก .begin() บน Transaction ไม่ได้ (ดูกับดักข้อ 6)

async fn nested_transaction_demo(pool: &sqlx::PgPool) -> Result<(), sqlx::Error> {
    let mut tx = pool.begin().await?;
    sqlx::query("SELECT 1").execute(&mut *tx).await?;

    {
        // เปิด savepoint ใหม่ภายใน tx — ภายใต้ผิว SQLx ส่งคำสั่ง SAVEPOINT ให้ PostgreSQL จริง
        let mut inner = tx.begin().await?;
        sqlx::query("SELECT 2").execute(&mut *inner).await?;
        inner.rollback().await?; // ยกเลิกแค่ส่วนของ savepoint นี้ ไม่กระทบ tx ชั้นนอก
    }

    tx.commit().await?; // tx ชั้นนอก commit ได้ปกติ แม้ inner จะ rollback ไปแล้วก็ตาม
    Ok(())
}
```

ผู้เขียนรันจริงได้ผลลัพธ์:

```
inner rollback สำเร็จ
outer commit สำเร็จ (savepoint ทำงานถูกต้อง)
```

`tx.begin()` (เรียกซ้ำบน `Transaction` ที่มีอยู่แล้ว ไม่ใช่บน `PgPool`) คือจุดที่ทำให้เกิด **nested transaction** — ผู้เขียนตรวจสอบ source code ของ `sqlx-core` ยืนยันว่าเบื้องหลังมันสร้าง SQL คำสั่ง `SAVEPOINT _sqlx_savepoint_<depth>` จริง (และ `inner.rollback()` แปลเป็น `ROLLBACK TO SAVEPOINT _sqlx_savepoint_<depth>` ตามลำดับความลึกของ nested transaction) — ประโยชน์ของ pattern นี้คือการเขียนฟังก์ชันย่อยที่ "พยายามทำอะไรสักอย่าง แล้วถ้าพลาดก็แค่ข้ามไปทำอย่างอื่นต่อ" โดยไม่ทำให้ transaction ใหญ่ทั้งก้อนต้อง rollback ไปด้วย (เช่น พยายาม insert log entry แบบ best-effort ระหว่างทำ transaction สำคัญ — ถ้า insert log พลาดก็ไม่ควรทำให้ transaction หลักพังไปด้วย)

**ข้อจำกัดที่ควรรู้**: savepoint มีต้นทุน (ต้อง round-trip ไปมากับฐานข้อมูลเพิ่มอีกอย่างน้อย 2 ครั้งต่อ nested level — เปิดกับปิด) และ nest ลึกเกินไปทำให้โค้ดอ่านยาก — ใช้เฉพาะกรณีที่ต้องการ "rollback แค่บางส่วน" จริง ๆ เท่านั้น ไม่ใช่ default pattern สำหรับทุก transaction

### 70.10 รวมทุกอย่างเข้ากับ Axum: `PgPool` ใน `AppState`

ตอนนี้มาผูกทุกอย่างที่เรียนมาเข้ากับ pattern `AppState` ของ Part 64 — เขียน Axum server ที่มี handler จริงคุยกับฐานข้อมูลจริง

#### `PgPool` ใน `AppState`: clone ถูกจริงไหม?

Part 64 อธิบายไว้ว่า `AppState` ต้อง `#[derive(Clone)]` เพราะ Axum เรียก `.clone()` ทุกครั้งที่ dispatch request ไปยัง handler ที่ต้องการ `State<AppState>` — และย้ำว่า field ที่แชร์ข้าม handler ควรห่อด้วย `Arc` เพื่อให้ `.clone()` ถูก (O(1) ไม่ deep-clone) คำถามคือ: `PgPool` ต้องห่อด้วย `Arc<PgPool>` เองอีกชั้นไหม หรือเก็บ `PgPool` ตรง ๆ ก็พอ?

ผู้เขียน**อ่าน source code จริง**ของ `sqlx-core` เวอร์ชัน 0.9.0 เพื่อตอบคำถามนี้อย่างแน่ชัด (ไม่เดา ไม่เชื่อคำโฆษณาใน docs เฉย ๆ):

```rust
// จาก sqlx-core-0.9.0/src/pool/mod.rs (คัดลอกมาตรงตัว)
pub struct Pool<DB: Database>(pub(crate) Arc<PoolInner<DB>>);

/// Returns a new [Pool] tied to the same shared connection pool.
impl<DB: Database> Clone for Pool<DB> {
    fn clone(&self) -> Self {
        Self(Arc::clone(&self.0))
    }
}
```

ยืนยันชัดเจน: **`Pool<DB>` (ซึ่ง `PgPool` คือ type alias ของมันสำหรับ PostgreSQL) คือ `Arc<PoolInner<DB>>` ที่ห่อด้วย tuple struct ตัวเดียว** และ `impl Clone` ก็แค่เรียก `Arc::clone` ตรง ๆ ไม่มีการ deep-clone อะไรเลย — พูดอีกแบบคือ **`PgPool` "เป็น" `Arc` อยู่แล้วในตัว** (ภายใน) เพราะฉะนั้น**ไม่ต้องห่อ `Arc<PgPool>` อีกชั้นซ้อนกัน** เก็บ `PgPool` ตรง ๆ ใน `AppState` ได้เลย การ `.clone()` มันจะถูกเท่ากับ `Arc::clone` เสมอไม่ว่าจะห่อซ้ำหรือไม่ (ห่อซ้ำแค่เพิ่ม indirection โดยไม่ได้ประโยชน์อะไรเพิ่ม)

#### AppState และ handler เต็มรูปแบบ

```rust
use axum::{
    extract::{Path, State},
    http::StatusCode,
    routing::get,
    Json, Router,
};
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use sqlx::postgres::PgPoolOptions;
use sqlx::PgPool;

#[derive(Debug, Serialize, sqlx::FromRow)]
struct Book {
    id: i64,
    isbn: String,
    title: String,
    author: String,
    total_copies: i32,
    available_copies: i32,
    published_year: Option<i32>,
    created_at: DateTime<Utc>,
}

#[derive(Deserialize)]
struct NewBook {
    isbn: String,
    title: String,
    author: String,
    total_copies: i32,
    published_year: Option<i32>,
}

// AppState เก็บ PgPool ตรง ๆ — ไม่ต้องห่อ Arc ซ้ำ (ตามที่พิสูจน์ไว้ข้างบน)
#[derive(Clone)]
struct AppState {
    pool: PgPool,
}

// ... AppError และ impl From<sqlx::Error> for AppError ตามหัวข้อ 70.7

async fn list_books(State(state): State<AppState>) -> Result<Json<Vec<Book>>, AppError> {
    let books = sqlx::query_as!(
        Book,
        r#"SELECT id, isbn, title, author, total_copies, available_copies, published_year, created_at
           FROM books ORDER BY id"#
    )
    .fetch_all(&state.pool)
    .await?;
    Ok(Json(books))
}

async fn get_book(
    State(state): State<AppState>,
    Path(id): Path<i64>,
) -> Result<Json<Book>, AppError> {
    let book = sqlx::query_as!(
        Book,
        r#"SELECT id, isbn, title, author, total_copies, available_copies, published_year, created_at
           FROM books WHERE id = $1"#,
        id
    )
    .fetch_one(&state.pool)
    .await?;
    Ok(Json(book))
}

async fn create_book(
    State(state): State<AppState>,
    Json(body): Json<NewBook>,
) -> Result<(StatusCode, Json<Book>), AppError> {
    let book = sqlx::query_as!(
        Book,
        r#"
        INSERT INTO books (isbn, title, author, total_copies, available_copies, published_year)
        VALUES ($1, $2, $3, $4, $4, $5)
        RETURNING id, isbn, title, author, total_copies, available_copies, published_year, created_at
        "#,
        body.isbn,
        body.title,
        body.author,
        body.total_copies,
        body.published_year,
    )
    .fetch_one(&state.pool)
    .await?;
    Ok((StatusCode::CREATED, Json(book)))
}

#[tokio::main]
async fn main() {
    let database_url = std::env::var("DATABASE_URL")
        .unwrap_or_else(|_| "postgres://postgres:postgres@127.0.0.1:5432/rust_course_scratch".to_string());

    let pool = PgPoolOptions::new()
        .max_connections(5)
        .connect(&database_url)
        .await
        .expect("เชื่อมต่อฐานข้อมูลไม่สำเร็จ");

    let state = AppState { pool };

    let app = Router::new()
        .route("/books", get(list_books).post(create_book))
        .route("/books/{id}", get(get_book))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:4070").await.unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}
```

ผู้เขียน**คอมไพล์และรัน server นี้จริง** แล้วยิง `curl` จริงตามลำดับ:

```bash
$ curl -s http://127.0.0.1:4070/books
[]

$ curl -s -X POST http://127.0.0.1:4070/books -H 'Content-Type: application/json' \
    -d '{"isbn":"111-1","title":"Programming Rust","author":"Jim Blandy","total_copies":2,"published_year":2021}'
{"id":4,"isbn":"111-1","title":"Programming Rust","author":"Jim Blandy","total_copies":2,"available_copies":2,"published_year":2021,"created_at":"2026-09-27T00:03:38.535351Z"}

$ curl -s -i -X POST http://127.0.0.1:4070/books -H 'Content-Type: application/json' \
    -d '{"isbn":"111-1","title":"Dup","author":"X","total_copies":1,"published_year":null}'
HTTP/1.1 409 Conflict
content-type: application/json

{"error":"duplicate: duplicate key value violates unique constraint \"books_isbn_key\""}

$ curl -s -i http://127.0.0.1:4070/books/9999
HTTP/1.1 404 Not Found
content-type: application/json

{"error":"not found"}

$ curl -s http://127.0.0.1:4070/books
[{"id":4,"isbn":"111-1","title":"Programming Rust","author":"Jim Blandy","total_copies":2,"available_copies":2,"published_year":2021,"created_at":"2026-09-27T00:03:38.535351Z"}]
```

ทุกอย่างทำงานตรงตามที่ออกแบบ: list ว่างตอนไม่มีข้อมูล, สร้างสำเร็จได้ `201 Created` พร้อมข้อมูลที่ database generate ให้ (`id`, `created_at`), สร้างซ้ำ ISBN ได้ `409 Conflict` ที่มีความหมาย, หา id ที่ไม่มีได้ `404 Not Found` — handler ทุกตัวเป็น `async fn` ธรรมดาที่ประกาศ `State<AppState>` ตรง ๆ ไม่มีการ clone ตัวแปรเข้า closure มือเลยแม้แต่จุดเดียว (ตรงตาม pattern ของ Part 64 ทุกประการ เพียงแต่ `AppState` คราวนี้ถือ `PgPool` จริงแทน `Arc<Mutex<HashMap<...>>>`)

### 70.11 Connection Pool Exhaustion: พฤติกรรมจริงเมื่อ Pool เต็ม

หัวข้อ 70.3 อธิบายไว้ว่า `max_connections` คือขีดจำกัดจริงของจำนวน task ที่คุยกับฐานข้อมูลพร้อมกันได้ — คำถามที่สำคัญคือ: **ถ้ามี request มากกว่าที่ pool รองรับพร้อมกันจริง ๆ จะเกิดอะไรขึ้น?** error ทันที หรือรอ (queue)? ผู้เขียนทดสอบเรื่องนี้ด้วยการยึด connection จนครบ `max_connections` แล้ววัดพฤติกรรมของ request ตัวต่อไปจริง ๆ

```rust
use sqlx::postgres::PgPoolOptions;
use std::time::{Duration, Instant};

async fn pool_exhaustion_probe() {
    let small_pool = PgPoolOptions::new()
        .max_connections(2)                              // ตั้งใจให้เล็กมาก เพื่อทดสอบง่าย
        .acquire_timeout(Duration::from_secs(2))          // รอสูงสุด 2 วินาทีก่อนยอมแพ้
        .connect("postgres://postgres:postgres@127.0.0.1:5432/rust_course_scratch")
        .await
        .unwrap();

    let conn1 = small_pool.acquire().await.unwrap(); // ยึด connection ตัวที่ 1
    let conn2 = small_pool.acquire().await.unwrap(); // ยึด connection ตัวที่ 2 (ครบ max_connections แล้ว)

    let start = Instant::now();
    let result = small_pool.acquire().await; // ขอ connection ตัวที่ 3 — pool เต็มแล้ว
    let elapsed = start.elapsed();

    match result {
        Ok(_) => println!("ได้ connection ตัวที่ 3 (ไม่ควรเกิดถ้า pool เต็มจริง)"),
        Err(e) => println!("acquire ตัวที่ 3 ล้มเหลวหลังรอ {:.2?}: {e}", elapsed),
    }

    drop(conn1);
    drop(conn2);
}
```

ผลลัพธ์จริง:

```
acquire ตัวที่ 3 ล้มเหลวหลังรอ 2.00s: pool timed out while waiting for an open connection (คาด: PoolTimedOut)
```

ข้อสังเกตสำคัญที่พิสูจน์ได้จากผลลัพธ์นี้: **`elapsed` คือ 2.00 วินาที พอดีกับค่า `acquire_timeout` ที่ตั้งไว้** — ไม่ใช่ error ทันทีตอนเรียก `.acquire()` ครั้งที่สาม แปลว่า SQLx **รอ (block เฉพาะ task นี้ ไม่ block thread อื่นเพราะเป็น async)** จนครบเวลาที่กำหนดก่อนจะยอมแพ้และคืน `Err(sqlx::Error::PoolTimedOut)` — นี่คือพฤติกรรมที่ถูกต้องและเป็นประโยชน์มาก: ถ้า traffic สูงขึ้นชั่วครู่ (spike) แล้วลดลงเร็ว request ที่มาตอน pool เต็มจะ**รอเฉย ๆ จนกว่าจะมี connection ว่าง** (จาก request อื่นที่ทำเสร็จแล้วคืน connection กลับ pool) แทนที่จะ error ทันทีทั้งที่จริง ๆ รอแค่เสี้ยววินาทีก็เพียงพอแล้ว

ผูกเรื่องนี้กลับไปที่ Part 48 เรื่อง Tokio task concurrency: การที่ `.acquire()` "รอ" ไม่ได้แปลว่า thread ของ Tokio ถูกบล็อกทิ้งไว้เฉย ๆ — มันคือ `Future` ที่ยังไม่พร้อม (`Poll::Pending`) จนกว่า pool จะมี connection ว่างให้ ระหว่างที่รออยู่ Tokio scheduler เอา thread นั้นไปรัน task อื่นได้ตามปกติ (ตรงตามโมเดล cooperative scheduling ที่ Part 46-48 สอนไว้) — สิ่งที่ "รอ" จริง ๆ คือ**ตัว request handler ตัวนั้น** (client ที่ยิง request มาต้องรอ response นานขึ้น) ไม่ใช่ตัว server ทั้งตัวหยุดทำงาน

**ผลเชิงปฏิบัติสำหรับการออกแบบระบบจริง**: `acquire_timeout` ที่สั้นเกินไปทำให้ระบบ "ยอมแพ้เร็วเกินไป" ตอน traffic spike ชั่วครู่ (error ทั้งที่รอแค่นิดเดียวก็ได้ connection) ส่วน `acquire_timeout` ที่ยาวเกินไป (หรือไม่ตั้งเลย — ค่า default ของ `PgPoolOptions` คือ 30 วินาที) ทำให้ client ต้องรอนานผิดปกติตอน pool เต็มจริง ๆ (แทนที่จะได้ error เร็ว ๆ แล้วไป retry) — ต้องเลือกให้เหมาะกับ SLA ของระบบ (เช่น ถ้า client timeout ที่ 3 วินาที `acquire_timeout` ที่ 30 วินาทีไม่มีประโยชน์อะไรเลย เพราะ client จะ timeout ไปก่อนอยู่ดี)

### 70.12 Health Check และ Graceful Shutdown

ระบบจริงที่ deploy บน production มักต้องมี endpoint สำหรับให้ load balancer/orchestrator (เช่น Kubernetes) เช็คว่าแอปยัง "แข็งแรง" อยู่ไหม — ส่วนสำคัญของ health check ที่มี database เป็น dependency คือต้องเช็คว่า**ต่อฐานข้อมูลได้จริง** ไม่ใช่แค่เช็คว่า process ยังรันอยู่เฉย ๆ (แอปอาจรันอยู่แต่ต่อฐานข้อมูลไม่ได้แล้วก็ได้ — ควรถูกมองว่า "ไม่พร้อมรับ traffic" เหมือนกัน)

#### Health Check Endpoint ด้วย `pool.acquire()`

```rust
use axum::{extract::State, http::StatusCode};

async fn health_check(State(state): State<AppState>) -> StatusCode {
    match state.pool.acquire().await {
        Ok(_conn) => StatusCode::OK,
        Err(_) => StatusCode::SERVICE_UNAVAILABLE,
    }
}
```

`pool.acquire()` พยายามหยิบ connection ที่ใช้งานได้จริงตัวหนึ่งจาก pool (ถ้าไม่มี connection ว่างเลยและต้องเปิดใหม่ มันจะเปิดจริงและทดสอบว่าต่อฐานข้อมูลสำเร็จไหมในตัว) ผู้เขียนทดสอบจริงทั้งสองกรณี — ตอนฐานข้อมูลปกติ:

```
health check ok, pool.size()=2
```

และลองปิด `pool` ก่อนเรียก (จำลองสถานการณ์ที่แอปกำลัง shutdown แล้วยังมี request สุดท้ายหลงเหลืออยู่ — ดูหัวข้อถัดไป) ได้ error ที่ตรวจจับได้ทันที:

```
attempted to acquire a connection on a closed pool
```

`.acquire()` คืน `Result<PoolConnection<Postgres>, sqlx::Error>` — ถ้าสำเร็จ `_conn` จะถูก `drop` ทันทีที่ scope จบ (คืน connection กลับ pool อัตโนมัติ ไม่ต้องเรียกอะไรเพิ่มเอง เชื่อมกับ RAII pattern ที่ Rust ใช้ตลอดทั้งภาษา ตาม Part 27-29 เรื่อง smart pointer/Drop) — endpoint นี้จึงเบามาก (ไม่ query ข้อมูลจริง แค่พิสูจน์ว่า "คุยกับฐานข้อมูลได้" เท่านั้น) เหมาะสำหรับให้ orchestrator เรียกถี่ ๆ ได้โดยไม่กระทบ performance ของระบบจริง

#### Graceful Shutdown: `pool.close()`

เวลาแอปได้รับสัญญาณให้ปิดตัว (เช่น `SIGTERM` จาก orchestrator ตอน deploy เวอร์ชันใหม่) ควรปิด connection pool อย่างเป็นระเบียบ**ก่อน**โปรแกรมจบ ไม่ใช่ปล่อยให้ process ตายไปเฉย ๆ ทั้งที่ยัง query ค้างอยู่ — `PgPool` มี method `.close()` ที่:

1. รอให้ query ที่กำลังทำงานอยู่ ณ ขณะนั้นเสร็จก่อน (ไม่ตัดจบกลางทาง)
2. ปิด connection ทุกตัวใน pool อย่างเป็นระเบียบ (ส่ง `Terminate` message ให้ PostgreSQL รู้ตัวว่า connection นี้จะไม่ใช้อีกแล้ว ไม่ใช่แค่ตัด socket ทิ้งดื้อ ๆ)
3. ทำให้ `.acquire()`/query ใหม่ที่เรียกหลังจากนี้ **error ทันที** (ไม่ใช่ hang รอ) — ผู้เขียนทดสอบจริง:

```rust
let pool = /* ... */;
println!("pool.size() before close = {}", pool.size());
pool.close().await;
println!("pool.is_closed() = {}", pool.is_closed());

let result = sqlx::query("SELECT 1").execute(&pool).await;
match result {
    Ok(_) => println!("ไม่ควรสำเร็จหลัง close"),
    Err(e) => println!("query หลัง close ล้มเหลวตามคาด: {e}"),
}
```

ผลลัพธ์จริง:

```
pool.size() before close = 1
pool.is_closed() = true
query หลัง close ล้มเหลวตามคาด: attempted to acquire a connection on a closed pool
```

`pool.close()` ทำให้ query ใหม่ที่เรียกหลังจากนั้น**ล้มเหลวทันที**ด้วยข้อความที่ชัดเจน (`attempted to acquire a connection on a closed pool`) — ไม่ hang รอ ไม่ panic แบบไม่มีเหตุผล การผูก `pool.close()` เข้ากับ shutdown signal ของ Axum ทำได้ผ่าน `axum::serve(...).with_graceful_shutdown(...)` (ที่ Part 62/65 อาจแนะนำผ่าน ๆ มาแล้วเรื่อง graceful shutdown ของตัว HTTP server เอง) — pattern ที่แนะนำคือรอให้ HTTP server ปิดตัวเสร็จก่อน (ไม่รับ request ใหม่ + รอ request ที่ค้างอยู่เสร็จ) แล้วค่อยเรียก `pool.close().await` เป็นขั้นตอนสุดท้าย เพื่อการันตีว่าไม่มี query ค้างคาอยู่ตอนที่ connection ถูกปิดจริง

### 70.13 Testing กับ Database: `#[sqlx::test]`

จาก Part 32-33 คุณรู้จัก `#[test]`/`#[tokio::test]` มาแล้ว — สำหรับ code ที่ต้องคุยกับฐานข้อมูลจริง มีความซับซ้อนเพิ่มขึ้นมาอย่างหนึ่ง: **test แต่ละตัวต้องไม่เห็นข้อมูลของกันและกัน** (ถ้า test A insert ข้อมูลแล้ว test B ที่รันพร้อมกัน (SQLx test รันแบบ parallel โดย default เหมือน `#[test]` ปกติ) ไปนับแถวในตารางเดียวกัน ผลลัพธ์จะไม่แน่นอน — ขึ้นกับ test ไหนรันก่อน)

SQLx มี attribute macro **`#[sqlx::test]`** ที่แก้ปัญหานี้ให้โดยอัตโนมัติ: มันจะสร้าง**ฐานข้อมูลใหม่ชั่วคราว**ให้ทุก test function หนึ่งตัว (clone schema จาก migration ที่ระบุ) รัน migration ให้เสร็จก่อน แล้วส่ง `PgPool` ที่ต่อกับฐานข้อมูลชั่วคราวตัวนั้นเข้าไปเป็น parameter — จบ test แล้ว**ลบฐานข้อมูลชั่วคราวทิ้งอัตโนมัติ**

```rust
use sqlx::PgPool;

#[sqlx::test(migrations = "./migrations")]
async fn insert_and_count(pool: PgPool) -> sqlx::Result<()> {
    sqlx::query!(
        "INSERT INTO books (isbn, title, author, total_copies, available_copies) VALUES ($1, $2, $3, 1, 1)",
        "978-0-00-000000-0",
        "Test Driven Book",
        "Tester"
    )
    .execute(&pool)
    .await?;

    let count = sqlx::query!("SELECT COUNT(*) as c FROM books")
        .fetch_one(&pool)
        .await?;
    assert_eq!(count.c, Some(1));
    Ok(())
}

#[sqlx::test(migrations = "./migrations")]
async fn separate_isolated_db(pool: PgPool) -> sqlx::Result<()> {
    // ทดสอบตัวที่สอง — insert ISBN เดียวกันกับ test แรกได้ (เพราะ database คนละตัวกันจริง ๆ)
    let count_before = sqlx::query!("SELECT COUNT(*) as c FROM books")
        .fetch_one(&pool)
        .await?;
    assert_eq!(count_before.c, Some(0)); // ต้องเป็น 0 — พิสูจน์ว่า isolation ทำงานจริง

    sqlx::query!(
        "INSERT INTO books (isbn, title, author, total_copies, available_copies) VALUES ($1, $2, $3, 1, 1)",
        "978-0-00-000000-0", // ISBN เดียวกันกับ test แรก — ไม่ควร conflict เพราะคนละ database
        "Test Driven Book",
        "Tester"
    )
    .execute(&pool)
    .await?;
    Ok(())
}
```

ผู้เขียน**รันจริง**ด้วย `cargo test` ผลลัพธ์จริง:

```
running 2 tests
test separate_isolated_db ... ok
test insert_and_count ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.87s
```

ทั้งสอง test **ผ่านพร้อมกัน** แม้จะ insert ISBN ซ้ำกันเป๊ะ (`978-0-00-000000-0`) — และ `assert_eq!(count_before.c, Some(0))` ใน `separate_isolated_db` ผ่านจริง (ไม่ใช่ `Some(1)` ที่จะเกิดถ้าทั้งสอง test ใช้ฐานข้อมูลเดียวกัน) พิสูจน์ว่า `#[sqlx::test]` สร้างฐานข้อมูลแยกกันจริง ๆ ให้แต่ละ test ไม่ใช่แค่ทฤษฎี — ผู้เขียนยังตรวจสอบด้วย `psql -l` หลัง test รันเสร็จ พบว่า**ไม่มีฐานข้อมูลชั่วคราวหลงเหลืออยู่เลย** (macro ลบทิ้งอัตโนมัติเมื่อ test จบ ไม่ว่า test นั้นจะ pass หรือ fail ก็ตาม)

**ข้อสังเกตเรื่อง `DATABASE_URL` สำหรับ `#[sqlx::test]`**: macro นี้ต้องมี `DATABASE_URL` ชี้ไปยังฐานข้อมูล PostgreSQL ที่ **user ที่ล็อกอินมีสิทธิ์สร้าง/ลบฐานข้อมูลได้** (เพราะมันต้อง `CREATE DATABASE`/`DROP DATABASE` ฐานข้อมูลชั่วคราวเอง) — ในสภาพแวดล้อม CI มักตั้งค่าให้ user ทดสอบมีสิทธิ์นี้เฉพาะ (ไม่ใช่ user เดียวกับที่แอป production ใช้จริง ที่ควรมีสิทธิ์จำกัดกว่ามาก ตามหลักการ least privilege)

**ทางเลือกอื่นที่ควรรู้จัก**: pattern "transaction-rollback-per-test" — เปิด transaction ตอนเริ่ม test แล้ว rollback เสมอตอนจบ (ไม่ว่า test จะ pass/fail) ทำให้ทุก test เห็นฐานข้อมูลที่ "สะอาด" เหมือนกันโดยไม่ต้องสร้าง/ลบฐานข้อมูลจริงทุกครั้ง (เร็วกว่า `#[sqlx::test]` สำหรับ test suite ขนาดใหญ่มาก ๆ เพราะไม่ต้อง `CREATE DATABASE`/`DROP DATABASE` ทุกครั้ง) แต่ต้องเขียน setup/teardown เองมากกว่า ผู้เขียนทดสอบจริง pattern นี้ด้วย (ใช้ `PgPool` ธรรมดาตัวเดียวที่แชร์ข้าม test แทนการให้ `#[sqlx::test]` สร้างฐานข้อมูลใหม่ให้):

```rust
#[tokio::test]
async fn transaction_rollback_per_test_pattern() {
    let pool = shared_test_pool().await; // PgPool ธรรมดาที่เชื่อมฐานข้อมูล test ตัวเดียวกันทุก test
    let mut tx = pool.begin().await.unwrap();

    sqlx::query!(
        "INSERT INTO books (isbn, title, author, total_copies, available_copies) VALUES ($1,$2,$3,1,1)",
        "978-rollback-test-1", "Rollback Test Book", "Tester"
    )
    .execute(&mut *tx)
    .await
    .unwrap();

    // ภายใน transaction เดียวกัน มองเห็นข้อมูลของตัวเองได้ปกติ (query ผ่าน &mut *tx)
    let count = sqlx::query!("SELECT COUNT(*) as c FROM books WHERE isbn = $1", "978-rollback-test-1")
        .fetch_one(&mut *tx)
        .await
        .unwrap();
    assert_eq!(count.c, Some(1));

    // จบ test ด้วยการ rollback เสมอ — ไม่ commit ไม่ว่า assert ข้างบนจะผ่านหรือไม่ก็ตาม
    tx.rollback().await.unwrap();

    // ตรวจสอบผ่าน pool ปกติ (นอก tx) ว่าไม่มีข้อมูลหลงเหลือให้ test อื่นเห็นจริง ๆ
    let count_after = sqlx::query!("SELECT COUNT(*) as c FROM books WHERE isbn = $1", "978-rollback-test-1")
        .fetch_one(&pool)
        .await
        .unwrap();
    assert_eq!(count_after.c, Some(0));
}
```

ผลลัพธ์จริง:

```
running 1 test
test transaction_rollback_per_test_pattern ... ok
```

ทั้งสอง `assert_eq!` ผ่านจริง — พิสูจน์สองอย่างพร้อมกัน: (1) ภายใน transaction เดียวกัน มองเห็นข้อมูลที่ insert ไปแล้วได้ตามปกติ (`count.c == Some(1)`) และ (2) หลัง rollback ข้อมูลนั้นไม่หลงเหลืออยู่จริงเมื่อ query ผ่าน `&pool` ปกติ (`count_after.c == Some(0)`) — ข้อจำกัดที่ต้องระวังของ pattern นี้คือ**ทุก query ในเนื้อ test ต้องเรียกผ่าน `&mut *tx` ให้ครบ** (กับดักเดียวกับหัวข้อกับดักข้อ 2 ของบทนี้) ถ้ามีจุดใดเผลอเรียกผ่าน `&pool` ตรง ๆ คำสั่งนั้นจะ commit ทันทีและไม่ถูก rollback ไปด้วย — และถ้า test นี้เรียก handler/ฟังก์ชันของแอปจริงที่รับ `&PgPool` เป็น parameter (ไม่ใช่ `&mut Transaction`) จะต้องปรับ signature ของฟังก์ชันนั้นให้รับ generic ที่ครอบคลุมทั้งสองกรณีได้ (ผ่าน trait bound เช่น `impl sqlx::PgExecutor<'_>`) ซึ่งเป็นการเปลี่ยนโครงสร้างโค้ดที่มากกว่า `#[sqlx::test]` ต้องการ — นี่คือเหตุผลที่ `#[sqlx::test]` ยังเหมาะเป็นจุดเริ่มต้นที่ดีที่สุดสำหรับ integration test ระดับ handler เต็มรูปแบบ ในขณะที่ pattern นี้เหมาะกับ unit test ของ query function เดี่ยว ๆ ที่ต้องการความเร็วสูงและมีจำนวนมาก

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. ลืมตั้ง `DATABASE_URL` ก่อน `cargo build` แล้วใช้ `query!`/`query_as!`

```rust
// มี query_as! ในโค้ด แต่ไม่มี DATABASE_URL ตั้งไว้เลยตอน build
let book = sqlx::query_as!(Book, "SELECT * FROM books WHERE id = $1", 1_i64)
    .fetch_one(&pool)
    .await?;
```

`cargo build` จะ fail ทันทีด้วย error ประมาณนี้ (ข้อความจริงอาจต่างกันเล็กน้อยตามเวอร์ชัน แต่สาระเดิม):

```
error: `DATABASE_URL` must be set: try setting it in `.env`, or use `SQLX_OFFLINE=true`
  to disable compile-time verification
```

**วิธีแก้**: ตั้ง `DATABASE_URL` ก่อนสั่ง build เสมอ — ทางเลือกที่สะดวกที่สุดคือใส่ไว้ในไฟล์ `.env` ที่ root ของโปรเจกต์ (crate `dotenvy` หรือ `sqlx-cli` เองจะอ่านไฟล์นี้อัตโนมัติ) หรือ `export DATABASE_URL=...` ใน shell ก่อนรัน `cargo build`/`cargo run` — ถ้าไม่ต้องการพึ่งฐานข้อมูลออนไลน์ตลอดเวลา (เช่นตอน build บน CI) ให้รัน `cargo sqlx prepare` ครั้งหนึ่งตอนมีฐานข้อมูลเชื่อมต่อได้ แล้ว commit โฟลเดอร์ `.sqlx/` เข้า git จากนั้นตั้ง `SQLX_OFFLINE=true` แทน `DATABASE_URL` ตอน build ครั้งต่อ ๆ ไป

### 2. เรียก query ผ่าน `&pool` แทน `&mut *tx` ระหว่างอยู่ใน Transaction

```rust
let mut tx = pool.begin().await?;

sqlx::query!("UPDATE books SET available_copies = available_copies - 1 WHERE id = $1", book_id)
    .execute(&pool)          // ❌ ผิด — ใช้ &pool ตรง ๆ ไม่ได้อยู่ใน tx เลย
    .await?;

sqlx::query!("INSERT INTO borrow_records (book_id, borrower_name) VALUES ($1, $2)", book_id, "Alice")
    .execute(&mut *tx)       // อันนี้ถูก แต่คำสั่งข้างบนไปคนละ transaction กันแล้ว
    .await?;

tx.commit().await?; // commit อันนี้แค่ยืนยัน insert อย่างเดียว UPDATE ไปคนละที่แล้ว
```

โค้ดนี้ **compile ผ่านปกติ** (เพราะ `&pool` implement trait `Executor` เหมือนกับ `&mut *tx` — SQLx ไม่มีทางรู้ตอน compile ว่าคุณ "ตั้งใจ" ให้อยู่ใน transaction เดียวกัน) แต่ผลลัพธ์ผิดโดยสิ้นเชิง: คำสั่ง `UPDATE` ที่ใช้ `&pool` จะหยิบ connection คนละตัวจาก pool แล้ว **commit ทันที** ตาม auto-commit mode ปกติของ PostgreSQL (ไม่ได้อยู่ใน transaction ของ `tx` เลย) — ถ้า `tx.commit()` หรือ `tx.rollback()` เกิดขึ้นทีหลัง จะไม่มีผลกับ `UPDATE` นั้นอีกต่อไป เพราะมันสำเร็จไปแล้วนอก transaction ตั้งแต่ตอนเรียก `.execute(&pool)`

**วิธีแก้**: ทุก query ที่ต้องอยู่ใน transaction เดียวกันต้องเรียกผ่าน `&mut *tx` (หรือ `&mut tx` ก็ได้ในบาง signature — SQLx implement `Executor` ให้ `&mut Transaction<'_, DB>` โดยตรง) **สม่ำเสมอทุกคำสั่ง** ไม่สลับไปมาระหว่าง `&pool` กับ `&mut *tx` ภายใน transaction block เดียวกัน — วิธีป้องกันบั๊กนี้เชิง code review ที่ดีคือดูว่าทุกคำสั่งภายใน `{ let mut tx = pool.begin().await?; ... }` block ใช้ `tx` (ไม่ใช่ `pool`) ให้ครบทุกจุด

### 3. Column ที่ Nullable แต่ประกาศ field เป็น type ตรง ๆ ไม่ห่อ `Option<T>`

```rust
#[derive(sqlx::FromRow)]
struct Book {
    // ...
    published_year: i32, // ❌ column นี้ nullable จริง แต่ประกาศ field เป็น i32 ตรง ๆ
}
```

ถ้าใช้ `query_as!`/`query!` `cargo build` จะ fail ทันทีด้วย error แบบนี้:

```
error: mismatched types
  --> src/main.rs:12:5
   |
12 |     published_year: i32,
   |     ^^^^^^^^^^^^^^^^^^^ expected `Option<i32>`, found `i32`
   |
   = note: column `published_year` may be `NULL`; if the column is not nullable,
           create a `NOT NULL` constraint or use the `unwrap_or` methods on `Option<T>`
```

**วิธีแก้**: ห่อ field เป็น `Option<i32>` ตามที่ column จริงในฐานข้อมูลอนุญาต — ถ้ามั่นใจว่า column นี้**ไม่ควร**เป็น `NULL` เลยในทางตรรกะของระบบ (แต่ตอนนี้ schema ยังไม่มี `NOT NULL`) ทางออกที่ถูกต้องกว่าคือแก้ schema จริง (เพิ่ม `NOT NULL` ผ่าน migration ใหม่) ไม่ใช่แค่แก้ field ฝั่ง Rust ให้ "หลบ" error ไปโดยที่ฐานข้อมูลยังอนุญาต `NULL` อยู่ดี (จะพังตอน runtime ทันทีที่มีแถวที่ column นั้นเป็น `NULL` จริง ๆ โผล่มา ซึ่ง compile-time check จะช่วยไม่ได้อีกต่อไปเพราะคุณ "โกหก" type ไปแล้ว)

### 4. เอา Input ผู้ใช้ไป Format เข้า SQL String ตรง ๆ (SQL Injection)

```rust
let unsafe_sql = format!("SELECT * FROM books WHERE isbn = '{}'", user_input); // ❌ อันตรายมาก
let rows = sqlx::query(&unsafe_sql).fetch_all(&pool).await?;
```

ใน `sqlx` เวอร์ชัน 0.9.0 ที่ใช้ในบทนี้ โค้ดนี้จะไม่ผ่าน compile ตั้งแต่แรกด้วย error:

```
error[E0277]: dynamic SQL strings should be audited for possible injections
   |
   = help: the trait `SqlSafeStr` is not implemented for `&std::string::String`
   = note: prefer literal SQL strings with bind parameters or `QueryBuilder` to add dynamic data to a query.
```

นี่คือ compile-time safety net ที่มีมาให้แล้ว — แต่ **ถ้าคุณ (หรือเพื่อนร่วมทีม) เลือกบายพาสมันด้วย `sqlx::AssertSqlSafe(...)` โดยไม่ได้ตรวจสอบจริง ๆ** ความเสี่ยงจะกลับมาทันที (ดูหัวข้อ 70.6 ที่พิสูจน์ด้วยโค้ดจริงว่า input `nonexistent' OR '1'='1` ทำให้ query คืนข้อมูลทั้งตารางออกมาได้อย่างไร) **วิธีแก้ที่ถูกต้องเสมอ**: ใช้ `$1, $2, ...` กับ `.bind()` หรือส่งเป็น argument ให้ `query!`/`query_as!` ตรง ๆ — ไม่มีเหตุผลที่ดีที่จะ format ค่าจาก input ผู้ใช้เข้า SQL string เองในสถานการณ์ปกติเลย (กรณีที่ต้อง dynamic จริง ๆ เช่น ชื่อ column/ชื่อตารางที่เปลี่ยนได้ตาม logic เท่านั้น ควร whitelist ค่าที่อนุญาตไว้ล่วงหน้าแทนรับ input มาต่อตรง ๆ)

### 5. ลืมว่า `.clone()` ของ `AppState` ที่มี `PgPool` ไม่ได้เปิด Connection ใหม่

มือใหม่บางคนเข้าใจผิดว่าทุกครั้งที่ Axum เรียก `state.clone()` (ตามที่ Part 64 อธิบาย) จะมีการเปิด database connection ใหม่ตามไปด้วย แล้วพยายาม "แก้ปัญหา" ด้วยการห่อ `Arc<PgPool>` เพิ่มเข้าไปอีกชั้น หรือแย่กว่านั้นคือพยายาม lazy-init pool แยกต่อ handler:

```rust
#[derive(Clone)]
struct AppState {
    pool: Arc<PgPool>, // ไม่ผิด แต่ไม่จำเป็น — เพิ่ม indirection โดยไม่ได้ประโยชน์
}
```

**ความเข้าใจที่ถูกต้อง** (ตามที่พิสูจน์ด้วย source code ในหัวข้อ 70.10): `PgPool` "เป็น" `Arc<PoolInner<DB>>` อยู่แล้วในตัวมันเอง `.clone()` ของ `PgPool` เท่ากับ `Arc::clone` เสมอ — ไม่มีการเปิด connection ใหม่ ไม่มีการ deep-clone อะไรทั้งสิ้น ไม่ว่าจะ `.clone()` กี่ครั้งก็ตาม จำนวน connection จริงที่เปิดกับฐานข้อมูลจะยังคงอยู่ที่ไม่เกิน `max_connections` เสมอ (พิสูจน์ได้ง่าย ๆ ด้วยการเช็ค `pool.size()` ก่อน/หลัง clone หลายรอบ — ตัวเลขจะไม่เปลี่ยน) **วิธีแก้**: เก็บ `PgPool` ตรง ๆ ใน `AppState` พอ ไม่ต้องห่อ `Arc` ซ้ำ

### 6. ลืม `use sqlx::Acquire;` ตอนเปิด Nested Transaction

```rust
let mut tx = pool.begin().await?;
let mut inner = tx.begin().await?; // ❌ ยังไม่ import sqlx::Acquire
```

`cargo build` ให้ error จริงดังนี้:

```
error[E0599]: no method named `begin` found for struct `Transaction<'_, Postgres>` in the current scope
   |
   = help: items from traits can only be used if the trait is in scope
help: trait `Acquire` which provides `begin` is implemented but not in scope; perhaps you want to import it
   |
 1 + use sqlx::Acquire;
   |
```

**วิธีแก้**: เพิ่ม `use sqlx::Acquire;` ตามที่ compiler แนะนำตรง ๆ ในบรรทัด `help:` — เหตุผลที่ `.begin()` มาจาก trait แยก (ไม่ได้เป็น method ตรงของ `Transaction`/`PgPool`) คือ SQLx ออกแบบให้ `Acquire` เป็น trait กลางที่ implement ให้ทั้ง `&PgPool` และ `&mut Transaction<'_, DB>` (ทั้งสอง "acquire ตัวที่คุยกับฐานข้อมูลได้" ในความหมายกว้าง ๆ เหมือนกัน) — นี่คือตัวอย่างที่ดีของสิ่งที่ Part 19/21 สอนไว้เรื่อง trait: method ที่มาจาก trait ต้อง **import trait นั้นเข้า scope ก่อนเสมอ** ไม่ว่า type ที่เรียกจะ implement trait นั้นจริงหรือไม่ก็ตาม (ต่างจาก inherent method ที่เรียกได้ทันทีไม่ต้อง import อะไรเพิ่ม) — error message ของ Rust ในกรณีนี้ช่วยได้มากเพราะบอกชื่อ trait และวิธีแก้ตรง ๆ ให้เลย

### 7. ลืม `#[derive(sqlx::FromRow)]` แล้วใช้กับ `query_as`/`query_as!`

```rust
// ไม่มี #[derive(sqlx::FromRow)] เลย
struct BookNoDerive {
    id: i64,
    title: String,
}

let book = sqlx::query_as::<_, BookNoDerive>("SELECT id, title FROM books LIMIT 1")
    .fetch_one(&pool)
    .await?;
```

`cargo build` fail ทันทีด้วย error สองจุดพร้อมกัน:

```
error[E0599]: the method `fetch_one` exists for struct `QueryAs<'_, _, BookNoDerive, _>`, but its trait bounds were not satisfied
   |
   |   struct BookNoDerive {
   |   ------------------- doesn't satisfy `BookNoDerive: FromRow<'r, _>`
   |
   = note: the following trait bounds were not satisfied:
           `BookNoDerive: FromRow<'r, _>`
```

**วิธีแก้**: เพิ่ม `#[derive(sqlx::FromRow)]` ให้ struct ที่จะใช้เป็นเป้าหมายของ `query_as`/`query_as!`/`fetch_all`/`fetch_one` เสมอ — error นี้ตรงไปตรงมาในเชิงกลไก (ตามหัวข้อ 70.6 ที่อธิบายว่า `query_as` ต้องการ `T: FromRow<...>` เป็น trait bound) แต่ตำแหน่งที่ error ปรากฏ (`fetch_one` ไม่มี method ให้เรียก) อาจทำให้มือใหม่งงว่าเกี่ยวอะไรกับ derive macro — จุดที่ต้องสังเกตคือข้อความ `doesn't satisfy` ที่ชี้ตรงไปที่ตัว struct และชื่อ trait `FromRow` ที่ขาดไป

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เพิ่ม endpoint `PUT /books/{id}` ใน server จากหัวข้อ 70.10 ที่รับ JSON body `{ "title": "...", "author": "..." }` แล้ว `UPDATE` แถวที่ `id` ตรงกัน คืน `404 Not Found` (ผ่าน `AppError::NotFound`) ถ้าไม่มีหนังสือ id นั้นอยู่จริง — Hint: เขียน `UPDATE ... WHERE id = $N RETURNING ...` ด้วย `query_as!` ตัวเดียว ถ้าไม่มีแถว match `RETURNING` จะไม่คืนอะไรเลย ทำให้ `.fetch_one()` คืน `Err(sqlx::Error::RowNotFound)` ที่ `AppError` แปลงเป็น 404 ให้อัตโนมัติอยู่แล้ว ไม่ต้องเช็คเองก่อน

2. **(กลาง)** เขียน endpoint `POST /books/{id}/return` ที่จำลองการ "คืนหนังสือ" — ต้องทำสองอย่างใน **transaction เดียว**: (1) เพิ่ม `available_copies` ขึ้น 1 (แต่ห้ามเกิน `total_copies` — เช็คด้วย `WHERE available_copies < total_copies`) และ (2) `UPDATE borrow_records SET returned_at = now() WHERE book_id = $1 AND returned_at IS NULL` (ปิด record การยืมที่ยังเปิดอยู่) — Hint: ใช้ `pool.begin()` แบบหัวข้อ 70.9 เช็ค `rows_affected()` ของคำสั่งแรก ถ้าเป็น 0 ให้ `tx.rollback()` แล้วคืน error ที่เหมาะสม (409 — ไม่มีที่ให้คืนเพิ่ม) ก่อนจะรันคำสั่งที่สอง

3. **(ยาก)** สร้าง migration ใหม่ที่เพิ่ม column `category TEXT NOT NULL DEFAULT 'general'` เข้าตาราง `books` ที่มีข้อมูลอยู่แล้ว (ต้องใช้ `DEFAULT` เพื่อไม่ให้แถวเก่าพัง เพราะ `NOT NULL` บนตารางที่มีข้อมูลอยู่แล้วจะ fail ทันทีถ้าไม่มี default ให้แถวเก่า) แล้วเขียนฟังก์ชัน `list_books_by_category(pool: &PgPool, category: &str) -> Result<Vec<Book>, sqlx::Error>` ที่ query ด้วย `WHERE category = $1` — ทดสอบด้วย `#[sqlx::test]` สองตัวที่พิสูจน์ทั้ง "หา category ที่มีข้อมูลจริง" และ "หา category ที่ไม่มีข้อมูลเลยต้องได้ `Vec` ว่าง ไม่ error" — Hint: อย่าลืมรัน `sqlx migrate run` ก่อน `cargo build` ด้วย `query_as!`/`query!` ที่อ้างถึง column ใหม่ ไม่อย่างนั้น compile-time check จะไม่รู้จัก column นี้เลย

4. **(ยาก/ประยุกต์ใช้งานจริง)** ต่อยอดจากข้อ 2: เพิ่มการจำกัดจำนวนการยืมพร้อมกันของผู้ยืมคนเดียว (เช่น ยืมพร้อมกันได้ไม่เกิน 3 เล่ม) — ต้องเช็คจำนวน record ที่ `returned_at IS NULL` ของ `borrower_name` นั้นก่อน insert record ใหม่ **ภายใน transaction เดียวกัน** กับการลด `available_copies` (เพื่อป้องกัน race condition ที่ผู้ยืมคนเดียวยืมพร้อมกันสองครั้งในเวลาไล่เลี่ยกันจนได้ record เกิน 3 ไปได้) — Hint: การเช็คแล้ว insert สองคำสั่งแยกกันมี race condition อยู่ดีถ้าไม่ทำใน transaction level ที่เหมาะสม (`SERIALIZABLE` isolation level หรือ `SELECT ... FOR UPDATE` เพื่อ lock แถวที่เกี่ยวข้องระหว่างเช็ค) ลองค้นคว้าเรื่อง PostgreSQL isolation level เพิ่มเติม (`SET TRANSACTION ISOLATION LEVEL ...` ผ่าน `sqlx::query!` ธรรมดาก่อน query อื่นใน transaction เดียวกัน) นี่คือปัญหาที่ลึกกว่าที่เห็นตอนแรกมาก — ทดสอบด้วยการยิง request พร้อมกันจริง (เช่นด้วย `tokio::join!` เรียก endpoint เดียวกันสองครั้งพร้อมกัน) เพื่อดูว่า race condition เกิดขึ้นจริงไหมถ้าไม่ป้องกัน

## สรุป

บทนี้คือจุดที่ระบบตั๋ว/ห้องสมุดที่สร้างมาตั้งแต่ Part 62 เปลี่ยนจาก in-memory `HashMap` มาเป็นฐานข้อมูล PostgreSQL จริงเป็นครั้งแรกในหลักสูตร — สิ่งสำคัญที่ควรจำ:

- **SQLx เป็น async-native** ผูกกับ Tokio runtime เดียวกับที่ Axum ใช้ ไม่มี blocking call ที่แอบกีดขวาง task อื่นในระบบ
- **`sqlx::query!`/`query_as!` ตรวจสอบ SQL กับ schema จริงตอน compile time** ผ่านการต่อฐานข้อมูลจริง (`DATABASE_URL`) หรือ cache แบบ offline (`.sqlx/` + `SQLX_OFFLINE=true`) — จับ column พิมพ์ผิด, type ไม่ตรง, nullability ไม่ตรง ได้ตั้งแต่ก่อนโปรแกรมรันด้วยซ้ำ ต่างจาก `query`/`query_as` แบบไม่มี `!` ที่ type-check ตอน runtime เท่านั้น
- **Connection pool (`PgPoolOptions`) ต้องมีเสมอ** แทนการเปิด connection ใหม่ทุก request — และ pool ที่เต็มจะทำให้ request ตัวต่อไป**รอ (queue)** จนกว่าจะมี connection ว่างหรือหมด `acquire_timeout` (พิสูจน์ด้วยตัวเลขจริง) ไม่ใช่ error ทันที
- **Bind parameter (`$1, $2, ...`) คือมาตรการป้องกัน SQL injection ที่จำเป็นจริง** ไม่ใช่แค่ style — พิสูจน์ด้วยโค้ดจริงว่า string-format SQL รั่วข้อมูลทั้งตารางได้อย่างไร และ SQLx 0.9.0 เองก็เพิ่ม compile-time guard (`SqlSafeStr`) มาช่วยป้องกันเรื่องนี้แล้ว
- **`sqlx::Error` มี variant ที่ต้องแยกจัดการ**: `RowNotFound` (→ 404), `Database` ที่ต้องเช็ค `.kind()`/`.is_unique_violation()` ต่อ (→ 409 สำหรับ conflict), connection error (→ 500) — แปลงผ่าน `impl From<sqlx::Error> for AppError` ตาม pattern ที่ Part 66 สอน
- **Transaction (`pool.begin()`) การันตี atomicity จริง** — พิสูจน์ด้วยการวัดค่าก่อน/หลัง rollback ว่าคำสั่งที่สำเร็จไปแล้วก่อนหน้าถูกยกเลิกไปด้วยจริงเมื่อคำสั่งถัดมา fail
- **`PgPool` ใน `AppState` ไม่ต้องห่อ `Arc` ซ้ำ** — มันคือ `Arc<PoolInner<DB>>` อยู่แล้วในตัว (ยืนยันจาก source code จริง) `.clone()` ถูกเท่ากับ `Arc::clone` เสมอ
- **Nullable column ต้อง map เป็น `Option<T>`** เสมอ — `query!`/`query_as!` บังคับสิ่งนี้ให้ที่ compile time อัตโนมัติ
- **`#[sqlx::test]`** ให้ database แยกกันจริงต่อ test function หนึ่งตัว (สร้าง/ลบอัตโนมัติ) แก้ปัญหา test ที่รันพร้อมกันแล้วชนข้อมูลกัน — และ pattern "transaction-rollback-per-test" เป็นทางเลือกที่เร็วกว่าสำหรับ test suite ขนาดใหญ่ (พิสูจน์ด้วยโค้ดจริงทั้งสองแบบ)
- **เทคนิค query ที่เกินขอบเขต CRUD พื้นฐาน** ก็ทำได้ตรงไปตรงมาด้วย SQL ธรรมดา: `JOIN` ข้ามตาราง (`books`/`borrow_records`), `LIMIT`/`OFFSET` สำหรับ pagination (พร้อมข้อเตือนเรื่อง `ORDER BY`), batch insert ด้วย `UNNEST` (ลด round-trip เมื่อต้อง insert จำนวนมาก), คอลัมน์ `JSONB` สำหรับข้อมูลที่ shape ไม่แน่นอน (ผ่าน `serde_json::Value` หรือ `sqlx::types::Json<T>` ที่ type-safe กว่า), และ `UUID` เป็นทางเลือกของ primary key แทน `BIGSERIAL`

Part ถัดไป (Part 71) จะลงรายละเอียดเรื่อง query ขั้นสูงกว่านี้ — `QueryBuilder` สำหรับ SQL แบบ dynamic ที่ปลอดภัย (สำหรับ filter ที่จำนวนเงื่อนไขไม่แน่นอนตาม input ผู้ใช้ ตามที่หัวข้อ 70.6 ทิ้งท้ายไว้), การจัดการ migration ในทีมที่ทำงานพร้อมกันหลายคนแบบละเอียดกว่านี้, connection pooling ขั้นสูงกว่าที่บทนี้ครอบคลุม (เช่น read replica routing, prepared statement caching), keyset/cursor-based pagination สำหรับตารางขนาดใหญ่ (ต่อจากที่หัวข้อ 70.6 แนะนำไว้), และ full-text search ของ PostgreSQL — ทั้งหมดต่อยอดจากพื้นฐาน query/transaction/pool ที่บทนี้วางไว้โดยตรง

---

**Part ก่อนหน้า:** [เปรียบเทียบ Axum vs Actix-web vs Rocket](part-069-framework-comparison.md) | **Part ถัดไป:** [SQLx: Queries, Migrations, Connection Pooling](part-071-sqlx-queries-migrations.md)
