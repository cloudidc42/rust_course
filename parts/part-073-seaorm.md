# Part 73: SeaORM เบื้องต้น

> โมดูล: การพัฒนาเว็บแอปพลิเคชัน (Web Development) | ระดับ: สูง | เวลาโดยประมาณ: 240 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายจุดยืนของ **SeaORM** ในภาพรวมสามตัวเลือกของ Rust ecosystem (SQLx จาก Part 70-71, Diesel จาก Part 72, SeaORM บทนี้) ได้อย่างถูกต้อง — โดยเฉพาะข้อเท็จจริงเชิงสถาปัตยกรรมที่สำคัญที่สุด: SeaORM เป็น **async-native ตั้งแต่การออกแบบ** (ไม่ต้องใช้ pattern `spawn_blocking` แบบที่ Diesel ต้องใช้) เพราะมันถูกสร้าง**บนฐานของ SQLx โดยตรง** (`sea-orm` ประกาศ `sqlx`/`sqlx-core` เป็น dependency จริงใน `Cargo.lock` ไม่ใช่แค่แนวคิดคล้ายกัน) — พร้อมอธิบายว่าการมี "ชั้น abstraction เพิ่ม" นี้แลกอะไรมาแลกอะไรไป
- ใช้ `sea-orm-cli generate entity` **introspect ฐานข้อมูลจริงที่มีอยู่แล้ว** เพื่อ generate struct `Entity`/`Model`/`Column`/`Relation`/`ActiveModel` ให้อัตโนมัติ (workflow แบบ **database-first**) และอธิบายความต่างจาก Diesel's `schema.rs` (Part 72 — introspect เหมือนกันแต่ยังต้องเขียน struct model มือ) และจาก SQLx (Part 70 — ไม่มีการ generate อะไรเลย เขียน `#[derive(FromRow)]` มือทั้งหมด)
- อธิบายวงคำศัพท์หลักของ SeaORM ให้ถูกต้อง: `Entity` (ตัวแทนตาราง), `Model` (struct อ่านอย่างเดียว, พร้อมใช้), `ActiveModel` (struct ที่ track ว่า field ไหน "ถูกแก้ไข" บ้าง แล้ว save เฉพาะ field ที่เปลี่ยนจริง — **dirty tracking**), `Column` (enum ระบุคอลัมน์แบบ type-safe), `Relation` (enum ระบุความสัมพันธ์ระหว่างตาราง) — และพิสูจน์ด้วย SQL จริงที่ observe ได้ว่า `.update()` ส่ง `UPDATE` แค่คอลัมน์ที่เปลี่ยนจริงเท่านั้น ไม่ใช่ทุกคอลัมน์
- เขียน CRUD เต็มรูปแบบด้วย ActiveRecord-style API บน domain `books`/`borrow_records` เดียวกับ Part 70 (`Entity::find().all()`, `Entity::find_by_id().one()`, สร้างผ่าน `ActiveModel { ..Default::default() }` แล้ว `.insert()`, อัปเดตผ่าน `.into_active_model()` แล้ว `.update()`, ลบผ่าน `.delete()`) พร้อมเทียบความยาว/ระดับ abstraction ของโค้ดชุดเดียวกันกับที่ Part 70 เขียนด้วย SQLx ตรง ๆ
- ใช้ query builder ของ SeaORM (`.filter()`, `.order_by_asc()`/`.order_by_desc()`, `.paginate()`) รวมถึงโหลดข้อมูลเชื่อมโยงแบบ one-to-many ทั้งแบบ eager (`.find_with_related()`) และแบบ lazy (`.find_related()`) — พร้อมอธิบายว่า `.paginate()` ที่มีมาให้ในตัวคือความสะดวกที่ SQLx (Part 71) ต้องเขียน `LIMIT`/`OFFSET` มือเอง
- ใช้ transaction ผ่าน `db.begin()`/`.commit()`/`.rollback()` ทำ operation แบบ atomic บน domain เดิม (ยืมหนังสือ) พร้อมพิสูจน์ rollback ด้วยโค้ดจริงที่จงใจทำให้ query ที่สองใน transaction ล้มเหลว และเขียน migration แบบ code-first ด้วย `sea-orm-migration` พร้อมอธิบาย workflow ที่แท้จริงระหว่างการ migrate schema กับการ generate entity (สองทิศทางที่ SeaORM รองรับทั้งคู่)
- แปลง `DbErr` เป็น `AppError` ตาม pattern เดียวกับ Part 66/70 ได้ถูกต้อง รวมถึงตรวจจับ unique-constraint violation จริงจาก `DbErr::Query` ที่ห่อ `sqlx::Error::Database` ไว้ข้างใน และต่อ `DatabaseConnection` เข้ากับ `AppState` ของ Axum ทำ endpoint จริงที่รันแล้วยิงด้วย `curl` ได้ครบทุก status code (`200`/`201`/`404`/`409`) ตาม pattern เดียวกับ Part 70.10
- สรุปตารางเปรียบเทียบ SQLx/Diesel/SeaORM แบบตรงไปตรงมา (async-native, ระดับ abstraction, compile-time guarantee, learning curve, ความยาวโค้ดของ operation เดียวกัน) พร้อมยืนยันจุดยืนของหลักสูตรว่าจะใช้ SQLx เป็นตัวหลักต่อไปจาก Part 74 เป็นต้นไป — และรู้จัก `MockDatabase` เครื่องมือทดสอบของ SeaORM ที่ไม่ต้องมีฐานข้อมูลจริงเลย เป็นทางเลือกเสริมสำหรับ unit test ที่ไม่อยากพึ่งพา infrastructure จริง

## ความรู้ที่ต้องมีมาก่อน

- **Part 70-71 (SQLx: PostgreSQL, CRUD/Query Builder เพิ่มเติม)**: บทนี้ใช้ domain เดียวกันทุกประการ (`books`/`borrow_records` ที่มี `UNIQUE`/`CHECK`/`FOREIGN KEY` constraint ตรงตามที่ Part 70 สร้างไว้) เพื่อให้เทียบโค้ดชุดเดียวกันข้ามสามไลบรารีได้ตรงจุด ถ้ายังไม่ได้อ่าน Part 70 อย่างน้อยต้องเข้าใจว่า connection pool คืออะไร (`PgPool`), `DATABASE_URL` ใช้ยังไง, และ transaction ทำงานอย่างไรในระดับ SQL เพราะบทนี้จะไม่อธิบาย mechanism พื้นฐานเหล่านี้ซ้ำ
- **Part 72 (Diesel ORM เบื้องต้น)**: บทนี้เปรียบเทียบกับ Diesel ตลอดทั้งบท (โดยเฉพาะเรื่อง sync-first vs async-native, และ `schema.rs` vs `sea-orm-cli generate entity`) — ควรอ่าน Part 72 ก่อนเพื่อให้การเทียบมีความหมาย แต่ถ้าข้ามมาอ่านบทนี้ก่อนก็ยังเข้าใจ SeaORM เองได้ครบ (การเทียบเป็นส่วนเสริม ไม่ใช่เนื้อหาหลัก)
- **Part 46-48 (Async/Await, Futures, Tokio Runtime)**: SeaORM เป็น async ตั้งแต่ต้น ทุก method ที่คุยกับฐานข้อมูลคืน `Future` ที่ต้องมี executor (Tokio) มา poll — บทนี้ใช้ความเข้าใจเรื่อง `async fn`/`.await`/`#[tokio::main]` ตรง ๆ ตามที่ Part 46-48 สอนไว้
- **Part 66 (Axum Error Handling)**: การแปลง `DbErr` เป็น `AppError` ที่ implement `IntoResponse` ใช้ pattern เดียวกับที่ Part 66 สอน (enum ที่รวม error ทุกแหล่งของแอปไว้ที่เดียว แปลงแต่ละ variant เป็น HTTP status/body ต่างกัน)
- **Part 44-45 (Proc Macros: Basics, Derive)**: `#[derive(DeriveEntityModel)]` ที่ `sea-orm-cli` generate ให้ทำงานในระดับเดียวกับ derive macro อื่น ๆ ที่เคยเรียนมา (generate `impl` block จาก struct definition ตอน compile time) — บทนี้จะไม่อธิบาย mechanism ของ proc macro ซ้ำ แค่อธิบายว่า macro ตัวนี้ generate อะไรให้บ้าง
- **Part 12, 30-31 (Error Handling)**: `impl From<DbErr> for AppError` และ `match` แบบ exhaustive ตรงตามแนวทางที่ Part 12/30-31 สอนไว้
- **พื้นฐาน SQL และ Part 70.5 (Migration ด้วย sqlx-cli)**: เพื่อเทียบ workflow migration ของ `sea-orm-migration` กับ `sqlx-cli` ได้อย่างมีความหมาย

## หมายเหตุเรื่องการตรวจสอบเนื้อหา (สำคัญ — อ่านก่อนเริ่ม)

ก่อนเขียนบทนี้ ผู้เขียนตรวจสอบว่ามี PostgreSQL จริงให้ทดสอบหรือไม่ และพบว่า**มี PostgreSQL 16.13 รันอยู่จริงในเครื่อง** (cluster เดียวกับที่ Part 70 ใช้ — `pg_lsclusters` ยืนยันว่า online อยู่แล้วที่ port 5432) จึงสามารถทดสอบตัวอย่างส่วนใหญ่ในบทนี้แบบ **compile และ run จริง เชื่อมต่อฐานข้อมูลจริง** ได้ ไม่ใช่แค่เขียนโค้ดที่ "ควรจะถูก" ตามทฤษฎี

สิ่งที่ทดสอบจริงและยืนยันแล้ว (ทำในโปรเจกต์ scratch แยกนอก repo ทั้งหมด ลบทิ้งหลังเขียนบทเสร็จ):

- ติดตั้ง `sea-orm-cli` เวอร์ชัน **2.0.3** (ปัจจุบันบน crates.io ณ วันที่เขียน) และเพิ่ม `sea-orm = "2.0.3"` พร้อม feature `sqlx-postgres`, `runtime-tokio-rustls`, `macros` ตามที่โจทย์กำหนด
- **ตรวจสอบ `Cargo.lock` จริง** ยืนยันว่า `sea-orm v2.0.3` มี `sqlx` และ `sqlx-core` (เวอร์ชัน `0.9.0` — เวอร์ชันเดียวกับที่ Part 70 ใช้ตรง ๆ) เป็น dependency ตรงไม่ใช่ dependency ทางอ้อมผ่านตัวอื่น และพบว่า `sea-orm` เอง `pub use sqlx;` (re-export ตรงในโค้ด `lib.rs` บรรทัด 740) — นี่คือหลักฐานที่แน่นหนากว่าคำโฆษณาใน docs มาก
- สร้างตาราง `books`/`borrow_records` จริงด้วย schema เดียวกับ Part 70 (`BIGSERIAL`, `UNIQUE`, `CHECK`, `FOREIGN KEY`) แล้วรัน `sea-orm-cli generate entity` introspect จริง ได้ไฟล์ `books.rs`/`borrow_records.rs`/`mod.rs`/`prelude.rs` จริง (เนื้อหาที่แสดงในหัวข้อ 73.2 คือไฟล์ที่ generate จริง ไม่ใช่ตัวอย่างที่แต่งขึ้น)
- คอมไพล์และรันโค้ด CRUD เต็มรูปแบบจริง (`find().all()`, `find_by_id().one()`, `.insert()`, `.into_active_model()` + `.update()`, `.delete()`) รวมถึง `.filter()`, `.order_by_asc()`, `.paginate()`, `.find_related()`, `.find_with_related()` — ผลลัพธ์ที่แสดงในบทนี้คือ output จริงจากการรัน
- **พิสูจน์ dirty tracking ด้วยการเปิด SQL logging จริง** (ผ่าน `tracing-subscriber` + `sqlx::query` log target) เห็น SQL จริงที่ SeaORM ส่งให้ PostgreSQL ตอนเรียก `.update()` หลังแก้แค่ field เดียว — ยืนยันว่า `UPDATE` มีแค่คอลัมน์ที่เปลี่ยนจริงใน `SET` clause ไม่ใช่ทุกคอลัมน์
- พิสูจน์ transaction rollback จริงด้วยการเทียบค่า `available_copies` ก่อน/หลัง เหมือน pattern ที่ Part 70 ใช้ (จงใจละเมิด foreign key constraint กลาง transaction)
- ทดสอบ `sea-orm-migration` จริงแบบ end-to-end: `sea-orm-cli migrate init` สร้าง migration crate, เขียน migration จริง, รัน `up`/`status`/`down` จริงกับฐานข้อมูลจริง แล้ว `sea-orm-cli generate entity` จากฐานข้อมูลที่ migrate เสร็จ (พิสูจน์ chain การทำงานทั้งสองทิศทาง)
- พิสูจน์ error จริงตอน insert ISBN ซ้ำ ได้ `DbErr::Query(RuntimeErr::SqlxError(...))` ที่ข้างในห่อ `sqlx::Error::Database` ที่มี `.is_unique_violation()`/`.message()` เหมือนที่ Part 70 ใช้กับ SQLx ตรง ๆ — อ่าน source code จริงของ `sea-orm-2.0.3/src/error.rs` ยืนยัน enum `DbErr`/`RuntimeErr` ทั้งหมด
- สร้าง Axum server จริง (route `/books`, `/books/{id}`) ที่มี `DatabaseConnection` ของ SeaORM อยู่ใน `AppState` แบบเดียวกับ pattern `PgPool` ของ Part 70.10 แล้ว**รันจริงและยิงด้วย `curl` จริง** ครบทุกกรณี (`GET` list, `POST` สร้างใหม่, `GET` by id, `404` ตอนไม่พบ, `409` ตอน ISBN ซ้ำ) — ไม่ใช่แค่ compile ผ่านเฉย ๆ
- ทดสอบ `MockDatabase` (feature `mock`) จริง — สร้าง mock connection ที่ไม่ต้องมี PostgreSQL จริงเลย แล้วดู SQL ที่ SeaORM "จะส่งจริง" ผ่าน `.into_transaction_log()` ยืนยันว่า SeaORM `SELECT` คอลัมน์ทีละชื่อ (ไม่ใช่ `SELECT *`)

ทุกส่วนของบทนี้ผ่านการทดสอบแบบ live run ครบ — ไม่มีส่วนใดที่เป็น "documented แต่ไม่ได้ทดสอบจริง" เพราะ PostgreSQL มีให้ใช้งานจริงตลอดการเขียนบทนี้

## เนื้อหา

### 73.1 SeaORM คืออะไร และทำไมมันถึง "Async-Native" ได้จริง

จาก Part 70 คุณรู้จักตารางสรุปสามตัวเลือกของ Rust ecosystem สำหรับคุยกับฐานข้อมูลเชิงสัมพันธ์ไปแล้ว: SQLx (SQL ดิบ + compile-time check), Diesel (Rust DSL ที่ generate SQL ให้ + sync-first), และ SeaORM ที่บทนี้จะลงรายละเอียด — ตำแหน่งของ SeaORM ในสามตัวนี้คือ **ORM แบบ ActiveRecord เต็มรูปแบบ** (คล้าย Entity Framework ของ C#, ActiveRecord ของ Ruby on Rails, Eloquent ของ Laravel) ที่ให้คุณทำงานกับ "entity" และ "model" แทนการเขียน SQL หรือ query DSL ตรง ๆ

#### ข้อเท็จจริงที่สำคัญที่สุด: SeaORM สร้างบน SQLx จริง ไม่ใช่แค่ "คล้ายกัน"

หลายคนเข้าใจผิดว่า SQLx, Diesel, SeaORM เป็นสามทางเลือกที่**แยกจากกันโดยสิ้นเชิง** เหมือนเป็นคู่แข่งสามค่ายที่ไม่เกี่ยวกัน — ความจริงคือ **SeaORM ใช้ SQLx เป็น database driver ชั้นล่างจริง ๆ** ผู้เขียนตรวจสอบเรื่องนี้อย่างละเอียดก่อนเขียนบทนี้ ไม่ใช่แค่เชื่อคำโฆษณาใน docs:

```bash
# ใน Cargo.lock ของโปรเจกต์ scratch ที่เพิ่ม sea-orm = "2.0.3" (feature: sqlx-postgres)
$ grep -A 20 '^name = "sea-orm"$' Cargo.lock | grep -i sqlx
```

ผลลัพธ์จริง:

```
 "sea-query-sqlx",
 "sqlx",
 "sqlx-core",
```

และเมื่อดู dependency ของ `sqlx` เองใน `Cargo.lock` เดียวกัน:

```
name = "sqlx"
version = "0.9.0"
```

**`sea-orm` ประกาศ `sqlx`/`sqlx-core` เป็น dependency ตรง** (ไม่ใช่ dependency ทางอ้อมที่บังเอิญเจอผ่านตัวอื่น) และเวอร์ชัน `sqlx` ที่ใช้ (`0.9.0`) คือเวอร์ชันเดียวกับที่ Part 70 ใช้เขียน SQLx ตรง ๆ เลย — ยิ่งชัดกว่านั้น ผู้เขียนเปิดดู source code จริงของ `sea-orm-2.0.3/src/lib.rs` (บรรทัด 740) พบว่า:

```rust
// sea-orm-2.0.3/src/lib.rs (บรรทัด 740, ยืนยันจาก source code จริง)
pub use sqlx;
```

SeaORM **re-export `sqlx` ตรง ๆ** ให้คุณเข้าถึงผ่าน `sea_orm::sqlx::...` ได้เลย และใน `sea-orm-2.0.3/src/error.rs`:

```rust
#[cfg(feature = "sqlx-dep")]
pub use sqlx::error::Error as SqlxError;

#[cfg(feature = "sqlx-postgres")]
pub use sqlx::postgres::PgDatabaseError as SqlxPostgresError;
```

Error type ของ SeaORM เองก็ห่อ `sqlx::Error`/`sqlx::postgres::PgDatabaseError` ไว้ข้างในตรง ๆ (หัวข้อ 73.10 จะลงรายละเอียดเรื่องนี้) — สรุปคือ **นี่ไม่ใช่แค่ "แนวคิดคล้ายกัน" แต่เป็นความจริงระดับ dependency graph**: connection pool ที่ SeaORM ใช้ข้างใต้คือ `sqlx::PgPool` ตัวเดียวกันกับที่ Part 70 สอนสร้างด้วย `PgPoolOptions` ทุกประการ, การส่ง query ไปยัง PostgreSQL ก็ผ่าน `sqlx-postgres` driver ตัวเดียวกัน, และ error ที่เกิดขึ้นก็คือ `sqlx::Error` ที่ SeaORM แค่ห่อไว้อีกชั้นเท่านั้น

#### ทำไมข้อเท็จจริงนี้สำคัญ: มันคือเหตุผลที่ SeaORM เป็น async-native ได้จริง

นี่คือจุดที่ต่างจาก Diesel อย่างสิ้นเชิง — Diesel (Part 72) เดิมออกแบบมาเป็น **synchronous ตั้งแต่รากฐาน** เพราะฉะนั้นเวลาเอาไปใช้ในเว็บแอป async (เช่น Axum) คุณต้องห่อทุกการเรียก Diesel ด้วย `tokio::task::spawn_blocking` เพื่อไม่ให้ query ที่ blocking จริง ๆ ไปกีดขวาง thread ของ Tokio runtime (ปัญหาเดียวกับที่ Part 48 อธิบายไว้เรื่อง "ห้ามเรียก blocking code ใน async context ตรง ๆ") — แม้ตอนหลังจะมี `diesel-async` เพิ่มมาก็ยังไม่ใช่ค่า default ของ ecosystem หลัก

SeaORM **ไม่ต้องทำแบบนั้นเลย** เพราะมันไม่ได้เขียน blocking I/O เอง — มันแค่**เรียก SQLx** ที่เป็น async ตั้งแต่ต้นอยู่แล้ว (ตามที่ Part 70 อธิบายไว้ละเอียด: ทุก method ของ SQLx คืน `Future` ที่ทำงานผ่าน non-blocking network I/O) พูดให้เป็นภาพ: SeaORM คือชั้น**ActiveRecord API** ที่วางอยู่**บนบ่า**ของ SQLx อีกที — ทุกครั้งที่คุณเรียก `Entity::find().all(&db).await`, ข้างใต้ SeaORM จะสร้าง SQL string (ผ่าน `sea-query`, query builder ที่ SeaORM ใช้แยกเป็น crate ของตัวเอง) แล้วส่งให้ SQLx รันจริงผ่าน `sqlx::query()`/`.fetch_all()` — คุณได้ async-native ของ SQLx มาแบบ "ฟรี" เพราะสถาปัตยกรรมพาไปเอง ไม่ต้องมีคนมาออกแบบ async ให้ SeaORM ใหม่ตั้งแต่ต้น

**สิ่งที่คุณแลกมา (trade-off ที่ตรงไปตรงมา)**: การมีชั้น ActiveRecord วางอยู่บน SQLx อีกที หมายความว่ามี **abstraction เพิ่มขึ้นอีกชั้น** — ทุก query ที่คุณเขียนผ่าน SeaORM ต้องผ่านการแปลงจาก entity/column DSL เป็น SQL (ผ่าน `sea-query`) ก่อนที่จะไปถึง SQLx จริง ๆ ต่างจาก SQLx ที่คุณเขียน SQL ตรง ๆ ไม่มีชั้นแปลงระหว่างกลาง — สิ่งนี้ไม่ใช่ overhead ที่มากมายในทางปฏิบัติ (การสร้าง SQL string เป็น operation ที่เร็วมากเทียบกับ round-trip เครือข่ายไปฐานข้อมูล) แต่มันหมายถึง**คุณต้องเรียนรู้ DSL ของ SeaORM เพิ่มเติม** และการ debug บางครั้งต้องคิดสองชั้น (ทำไม entity ทำงานแบบนี้ → มันแปลงเป็น SQL อะไร → ทำไม SQL นั้นได้ผลลัพธ์แบบนี้) ต่างจาก SQLx ที่ SQL ที่คุณเห็นในโค้ดคือ SQL ที่รันจริงตรง ๆ ไม่มีการแปลงชั้นกลาง

#### ตารางสรุปตำแหน่งของ SeaORM (อัปเดตจากตารางใน Part 70/72)

| | SQLx (Part 70-71) | Diesel (Part 72) | **SeaORM (บทนี้)** |
|---|---|---|---|
| วิธีเขียน query | SQL จริงในรูป string literal | Rust DSL (method chain) generate SQL | **Entity/ActiveModel API** (คล้าย ActiveRecord) |
| Async รองรับ | Async-native ตั้งแต่ต้น | Sync-first (ต้อง `spawn_blocking` หรือใช้ `diesel-async` แยก) | **Async-native — เพราะสร้างบน SQLx จริง (ยืนยันจาก `Cargo.lock`)** |
| ระดับ abstraction | ต่ำ — ใกล้ SQL ดิบที่สุด | กลาง — DSL ที่ยัง map ใกล้ SQL | **สูง — เข้าใกล้ ORM/entity แบบเต็มรูปแบบ** |
| Workflow สร้าง struct | เขียน `#[derive(FromRow)]` มือ | Introspect ผ่าน `diesel print-schema` แต่ยังเขียน model struct มือ | **Generate `Entity`/`Model`/`ActiveModel` อัตโนมัติผ่าน `sea-orm-cli generate entity`** |
| Migration | `sqlx-cli` (`sqlx migrate add/run`) | `diesel migration` (`diesel_cli`) | `sea-orm-migration` (code-first) **หรือ** introspect DB ที่มีอยู่แล้ว (database-first) — รองรับทั้งสองทาง |

หัวข้อถัดไปจะเริ่มจากจุดที่ SeaORM ต่างจากทั้งสองตัวชัดที่สุด: **การ generate entity จากฐานข้อมูลจริงโดยอัตโนมัติ**

### 73.2 ติดตั้งและ Generate Entity จากฐานข้อมูลจริง

#### เพิ่ม `sea-orm` เข้าโปรเจกต์

```bash
cargo add sea-orm --features sqlx-postgres,runtime-tokio-rustls,macros
```

`Cargo.toml` ที่ได้ (ทดสอบจริง — เวอร์ชันปัจจุบันบน crates.io ณ วันที่เขียนบทนี้คือ `2.0.3`):

```toml
[dependencies]
sea-orm = { version = "2.0.3", features = ["sqlx-postgres", "runtime-tokio-rustls", "macros"] }
```

**อธิบาย feature ที่เปิด:**

- **`sqlx-postgres`** — บอก SeaORM ให้ใช้ SQLx เป็น driver สำหรับ PostgreSQL โดยเฉพาะ (ชื่อ feature มีคำว่า `sqlx` อยู่ตรง ๆ นี่คือหลักฐานอีกชิ้นที่เห็นได้จากหน้า `Cargo.toml` เองเลยว่า SeaORM ผูกกับ SQLx อย่างเปิดเผย — ต่างจาก Diesel ที่ไม่มี feature ชื่อคล้าย SQLx เพราะไม่ได้พึ่งพากันแบบนี้) SeaORM รองรับ MySQL/SQLite ด้วยผ่าน `sqlx-mysql`/`sqlx-sqlite`
- **`runtime-tokio-rustls`** — ผูกกับ Tokio runtime (ตรงกับ Part 48) พร้อม TLS backend แบบ `rustls` — สังเกตว่าชื่อ feature นี้**ผูก runtime กับ TLS backend ไว้ด้วยกันในชื่อเดียว** ต่างจาก SQLx เวอร์ชันปัจจุบันที่แยก TLS backend ออกมาเป็น feature อื่น (Part 70 หัวข้อ 70.2 อธิบายไว้) — SeaORM ยังคงใช้ convention แบบเดิม (`runtime-tokio-rustls`/`runtime-tokio-native-tls`) ที่รวมสองอย่างไว้ในชื่อเดียว เพราะ SeaORM เป็นชั้นที่วางอยู่บน SQLx และเลือก convention การตั้งชื่อ feature ของตัวเองแยกจาก SQLx
- **`macros`** — เปิด derive macro `DeriveEntityModel`/`DeriveActiveModel`/`DeriveRelation` ที่ `sea-orm-cli generate entity` ใช้ generate โค้ดให้ (หัวข้อถัดไปจะเห็นตรง ๆ)

#### ติดตั้ง `sea-orm-cli`

```bash
cargo install sea-orm-cli
```

ผลลัพธ์จริง (ทดสอบในเครื่องผู้เขียนบทนี้ — จำกัด feature ให้ตรงกับที่จะใช้เพื่อลดเวลา compile เหมือนที่ Part 70 ทำกับ `sqlx-cli`):

```bash
$ cargo install sea-orm-cli --no-default-features --features codegen,runtime-tokio-rustls,sqlx-postgres
```

```
    Finished `release` profile [optimized] target(s) in 1m 19s
  Installing /root/.cargo/bin/sea
  Installing /root/.cargo/bin/sea-orm-cli
   Installed package `sea-orm-cli v2.0.3` (executables `sea`, `sea-orm-cli`)
```

#### `sea-orm-cli generate entity`: Database-First Workflow

นี่คือจุดที่ SeaORM แตกต่างจาก workflow ที่คุณเคยเห็นมาทั้งใน Part 70 (SQLx) และ Part 72 (Diesel) อย่างชัดเจนที่สุด — ทั้งสามเครื่องมือ "อ่าน" schema ของฐานข้อมูลได้ (ทุกตัวรู้ว่าตารางมีคอลัมน์อะไร) แต่สิ่งที่ทำกับข้อมูลนั้นต่างกันโดยสิ้นเชิง:

- **SQLx**: ไม่ generate struct ให้เลย — macro `query_as!` แค่**ตรวจสอบ** ว่า struct ที่คุณเขียนมือตรงกับ schema จริงไหมตอน compile (ตามที่ Part 70 สอน) คุณต้องเขียน `#[derive(FromRow)]` struct ทุกตัวด้วยมือเอง
- **Diesel**: `diesel print-schema` (ที่ `diesel migration run` เรียกให้อัตโนมัติ) generate ไฟล์ `schema.rs` ที่มี macro `table!` บอกโครงสร้างตาราง — แต่ **model struct ที่ map เข้ากับตารางนั้น (ที่มี `#[derive(Queryable)]`) คุณยังต้องเขียนเองอยู่ดี** เป็นคนละไฟล์กับ `schema.rs`
- **SeaORM**: `sea-orm-cli generate entity` generate **struct `Model` ที่ใช้งานได้จริงทันที** พร้อม `ActiveModel`, `Column`, `Relation` ทั้งหมดในไฟล์เดียว — ไม่มีขั้นตอน "เขียน struct เองอีกที" เหลืออยู่เลย

มาดูจริงว่ามันทำงานอย่างไร — สมมติว่าคุณมีตาราง `books`/`borrow_records` อยู่แล้ว (schema เดียวกับ Part 70 หัวข้อ 70.5 ทุกประการ — สร้างด้วย SQL ตรงนี้เพื่อความสมบูรณ์):

```sql
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

รันคำสั่ง generate entity ชี้ไปยังฐานข้อมูลจริง:

```bash
sea-orm-cli generate entity \
  -u "postgres://postgres:postgres@127.0.0.1:5432/seaorm_scratch" \
  -o src/entities
```

ผลลัพธ์จริงที่ผู้เขียนได้ (รันจริง introspect ตารางที่สร้างไว้ข้างบน):

```
Connecting to Postgres ...
Discovering schema ...
... discovered.
Generating books.rs
    > Column `id`: i64, auto_increment, not_null
    > Column `isbn`: String, not_null, unique
    > Column `title`: String, not_null
    > Column `author`: String, not_null
    > Column `total_copies`: i32, not_null
    > Column `available_copies`: i32, not_null
    > Column `published_year`: Option<i32>
    > Column `created_at`: DateTimeWithTimeZone, not_null
Generating borrow_records.rs
    > Column `id`: i64, auto_increment, not_null
    > Column `book_id`: i64, not_null
    > Column `borrower_name`: String, not_null
    > Column `borrowed_at`: DateTimeWithTimeZone, not_null
    > Column `returned_at`: Option<DateTimeWithTimeZone>
Writing src/entities/books.rs
Writing src/entities/borrow_records.rs
Writing src/entities/mod.rs
Writing src/entities/prelude.rs
... Done.
```

สังเกตสิ่งสำคัญสองอย่างในผลลัพธ์: (1) `sea-orm-cli` **เดา type ของ Rust ให้ถูกจากคอลัมน์ nullable/not-null เอง** (`published_year`/`returned_at` กลายเป็น `Option<i32>`/`Option<DateTimeWithTimeZone>` อัตโนมัติ ตรงตามที่ Part 70 หัวข้อ 70.8 สอนเรื่อง `Option<T>` กับ nullable column — ไม่ต้องบอกมันเองว่าคอลัมน์ไหน nullable) และ (2) **มันตรวจ foreign key constraint เจอเองด้วย** (`book_id BIGINT NOT NULL REFERENCES books(id)`) แล้ว generate `Relation` enum ให้อัตโนมัติทั้งสองฝั่ง — หัวข้อ 73.6 จะลงรายละเอียดเรื่องนี้

`src/entities/books.rs` ที่ generate จริง (ไฟล์เต็ม ไม่ได้ตัด ไม่ได้แก้ไขอะไรเลย):

```rust
//! `SeaORM` Entity, @generated by sea-orm-codegen 2.0

use sea_orm::entity::prelude::*;

#[derive(Clone, Debug, PartialEq, Eq, DeriveEntityModel)]
#[sea_orm(table_name = "books")]
pub struct Model {
    #[sea_orm(primary_key)]
    pub id: i64,
    #[sea_orm(column_type = "Text", unique)]
    pub isbn: String,
    #[sea_orm(column_type = "Text")]
    pub title: String,
    #[sea_orm(column_type = "Text")]
    pub author: String,
    pub total_copies: i32,
    pub available_copies: i32,
    pub published_year: Option<i32>,
    pub created_at: DateTimeWithTimeZone,
}

#[derive(Copy, Clone, Debug, EnumIter, DeriveRelation)]
pub enum Relation {
    #[sea_orm(has_many = "super::borrow_records::Entity")]
    BorrowRecords,
}

impl Related<super::borrow_records::Entity> for Entity {
    fn to() -> RelationDef {
        Relation::BorrowRecords.def()
    }
}

impl ActiveModelBehavior for ActiveModel {}
```

`src/entities/borrow_records.rs` ที่ generate จริง:

```rust
//! `SeaORM` Entity, @generated by sea-orm-codegen 2.0

use sea_orm::entity::prelude::*;

#[derive(Clone, Debug, PartialEq, Eq, DeriveEntityModel)]
#[sea_orm(table_name = "borrow_records")]
pub struct Model {
    #[sea_orm(primary_key)]
    pub id: i64,
    pub book_id: i64,
    #[sea_orm(column_type = "Text")]
    pub borrower_name: String,
    pub borrowed_at: DateTimeWithTimeZone,
    pub returned_at: Option<DateTimeWithTimeZone>,
}

#[derive(Copy, Clone, Debug, EnumIter, DeriveRelation)]
pub enum Relation {
    #[sea_orm(
        belongs_to = "super::books::Entity",
        from = "Column::BookId",
        to = "super::books::Column::Id",
        on_update = "NoAction",
        on_delete = "NoAction"
    )]
    Books,
}

impl Related<super::books::Entity> for Entity {
    fn to() -> RelationDef {
        Relation::Books.def()
    }
}

impl ActiveModelBehavior for ActiveModel {}
```

และไฟล์ประกอบอีกสองไฟล์ที่ generate มาด้วย:

```rust
// src/entities/mod.rs
pub mod prelude;

pub mod books;
pub mod borrow_records;
```

```rust
// src/entities/prelude.rs
pub use super::books::Entity as Books;
pub use super::borrow_records::Entity as BorrowRecords;
```

`prelude.rs` ทำหน้าที่เป็นทางลัด — เวลาใช้งานจริงคุณ `use entities::prelude::*;` แล้วอ้าง `Books::find()`/`BorrowRecords::find()` ได้ตรง ๆ โดยไม่ต้องพิมพ์ `books::Entity`/`borrow_records::Entity` เต็ม ๆ ทุกครั้ง

#### อธิบาย attribute ของ `Relation` enum ทีละส่วน

ก่อนไปหัวข้อถัดไป มาดูว่าแต่ละ attribute ใน `Relation` enum ที่ generate มาให้หมายถึงอะไรบ้าง — ดู `borrow_records::Relation` (ฝั่ง "belongs to") เพราะเป็นฝั่งที่มี attribute มากกว่า:

```rust
#[sea_orm(
    belongs_to = "super::books::Entity",   // (1) ปลายทางของความสัมพันธ์
    from = "Column::BookId",               // (2) คอลัมน์ฝั่งนี้ (borrow_records) ที่ใช้ join
    to = "super::books::Column::Id",       // (3) คอลัมน์ฝั่งโน้น (books) ที่ใช้ join
    on_update = "NoAction",                // (4) พฤติกรรมเมื่อ books.id ถูก UPDATE
    on_delete = "NoAction"                 // (5) พฤติกรรมเมื่อ books.id ถูก DELETE
)]
Books,
```

- **(1) `belongs_to`** — บอกว่าแถวของ `borrow_records` หนึ่งแถว "เป็นของ" `books` หนึ่งแถว (ทิศทาง many-to-one จากมุมมองของ `borrow_records`) ตรงข้ามกับ `has_many` ที่ generate ไว้ฝั่ง `books::Relation` (หนึ่ง book มีได้หลาย borrow_records) — นี่คือสองฝั่งของความสัมพันธ์ one-to-many เดียวกัน มองจากมุมต่างกัน
- **(2)-(3) `from`/`to`** — ระบุคอลัมน์ที่ใช้ join กันตรง ๆ (`borrow_records.book_id = books.id`) ตรงกับ `FOREIGN KEY` constraint ที่มีอยู่จริงในฐานข้อมูล (`sea-orm-cli generate entity` อ่านค่านี้มาจาก constraint จริงตอน introspect ไม่ได้เดาสุ่ม)
- **(4)-(5) `on_update`/`on_delete`** — มาจาก `ON UPDATE`/`ON DELETE` clause ของ `FOREIGN KEY` constraint จริงในฐานข้อมูล — ในตัวอย่างนี้เป็น `NoAction` เพราะ SQL ตอนสร้างตาราง (`REFERENCES books(id)` เฉย ๆ ไม่มี `ON DELETE CASCADE`/`ON DELETE SET NULL` ต่อท้าย) ไม่ได้ระบุพฤติกรรมพิเศษไว้ — ค่านี้เป็นแค่**ข้อมูลที่ SeaORM เก็บไว้อ่านอย่างเดียว** ไม่ได้ไปเปลี่ยนพฤติกรรมจริงของฐานข้อมูล (พฤติกรรมจริงตอน `DELETE`/`UPDATE` ถูกกำหนดโดย constraint ในฐานข้อมูลเอง) — ถ้าต้องการให้ลบ `books` แล้วลบ `borrow_records` ที่เกี่ยวข้องอัตโนมัติ ต้องแก้ที่ระดับ SQL/migration (`ON DELETE CASCADE`) ไม่ใช่แก้ค่านี้ในไฟล์ entity

ฝั่ง `books::Relation` (has-many ตรงข้ามของ belongs-to) สั้นกว่ามาก เพราะไม่ต้องระบุ `from`/`to`/`on_update`/`on_delete` เอง (SeaORM หาให้จากฝั่ง `belongs_to` ที่มีข้อมูลครบอยู่แล้ว):

```rust
#[sea_orm(has_many = "super::borrow_records::Entity")]
BorrowRecords,
```

**สิ่งสำคัญที่ต้องเข้าใจ**: `#[derive(DeriveEntityModel)]` ตัวเดียวใน struct `Model` คือจุดที่ generate ทุกอย่างที่คุณจะใช้ในบทนี้ — จาก Part 44-45 คุณรู้ว่า derive macro คือ proc macro ที่ parse struct definition แล้ว generate `impl` block ให้ — `DeriveEntityModel` ทำงานหนักกว่า derive macro ปกติมาก มัน generate ให้ทั้ง: struct `Entity` (marker type ที่แทนตาราง), enum `Column` (แต่ละ variant คือหนึ่งคอลัมน์ เข้าถึงแบบ type-safe), enum `PrimaryKey`, struct `ActiveModel` (คู่กับ `Model` แต่ทุก field เป็น `ActiveValue<T>` แทน `T` ตรง ๆ), และ `impl` ทุกตัวที่ทำให้ `Entity`/`Model`/`ActiveModel` ทำงานร่วมกันได้ (`EntityTrait`, `ModelTrait`, `ActiveModelTrait`, ฯลฯ) — นี่คือเหตุผลว่าทำไมไฟล์ที่ generate มาดู "สั้น" มาก (20-35 บรรทัด) แต่ให้ API เต็มรูปแบบ: โค้ดส่วนใหญ่ไม่ได้อยู่ในไฟล์นี้ แต่ถูก generate ซ่อนอยู่หลัง macro ตอน compile

### 73.3 วงคำศัพท์หลักของ SeaORM: Entity, Model, ActiveModel, Column, Relation

ก่อนเขียน CRUD จริง ต้องเข้าใจให้ชัดว่าแต่ละคำหมายถึงอะไร เพราะนี่คือมโนทัศน์ (mental model) ที่ต่างจากทั้ง SQLx และ Diesel

| คำศัพท์ | คืออะไร | ใช้ตอนไหน |
|---|---|---|
| **`Entity`** (เช่น `books::Entity`, alias `Books`) | struct เปล่า (unit struct) ที่เป็น "ตัวแทน" ของตาราง ไม่ได้เก็บข้อมูลแถวไหนเลย | จุดเริ่มต้นของทุก query: `Books::find()`, `Books::find_by_id(1)`, `Books::insert(...)` |
| **`Model`** | struct ที่เก็บข้อมูล**หนึ่งแถวจริง** อ่านอย่างเดียว (fields เป็น `T`/`Option<T>` ตรง ๆ ไม่ใช่ `ActiveValue`) | ผลลัพธ์ของการ `SELECT` — สิ่งที่ `.all()`/`.one()` คืนมา |
| **`ActiveModel`** | struct คู่กับ `Model` แต่ทุก field เป็น `ActiveValue<T>` (จะเป็น `Set(v)`, `Unchanged(v)`, หรือ `NotSet`) — **track ว่า field ไหน "ถูกแก้ไข" บ้าง** | ใช้ตอนจะ **เขียน** ข้อมูล (insert/update) — เป็นตัวกลางระหว่างข้อมูลที่คุณต้องการเปลี่ยนกับคำสั่ง SQL ที่ SeaORM จะสร้างให้ |
| **`Column`** | enum ที่แต่ละ variant คือหนึ่งคอลัมน์ (เช่น `books::Column::Title`, `books::Column::AvailableCopies`) | ใช้ระบุคอลัมน์แบบ type-safe ใน `.filter()`, `.order_by()`, `.col_expr()` — พิมพ์ชื่อคอลัมน์ผิดจะ**ไม่ compile** เพราะเป็น enum จริง ไม่ใช่ string |
| **`Relation`** | enum ที่แต่ละ variant คือความสัมพันธ์หนึ่งเส้นไปยังตารางอื่น (เช่น `books::Relation::BorrowRecords`) | ใช้ตอน join/โหลดข้อมูลที่เชื่อมโยงกัน (`.find_with_related()`, `.find_related()`) |

**จุดที่ต้องเข้าใจให้แน่น: ทำไมต้องมีทั้ง `Model` และ `ActiveModel` แยกกัน**

ความแตกต่างนี้ไม่ใช่ความซับซ้อนที่ไม่มีประโยชน์ — มันสะท้อนความจริงเชิงตรรกะสองอย่างที่ต่างกัน: **"ข้อมูลที่อ่านมาแล้ว" (`Model`)** กับ **"ความตั้งใจจะเขียนอะไรลงไป" (`ActiveModel`)** เป็นคนละเรื่องกัน — เวลาคุณ `SELECT` แถวหนึ่งมา คุณได้ค่าทุกคอลัมน์แน่นอนอยู่แล้ว (`Model` ไม่ต้องมีสถานะ "ยังไม่ตั้งค่า" เพราะมันมาจากแถวจริงที่มีค่าครบทุกคอลัมน์) แต่เวลาคุณจะ **insert** แถวใหม่ บาง field อาจ "ยังไม่ตั้งใจใส่ค่า" (เช่น `id` ที่เป็น auto-increment — ให้ database คิดให้) หรือ **update** แถวเดิม บาง field อาจ "ไม่ได้แก้เลย" (ต้องแยกจาก field ที่แก้จริง เพื่อสร้าง `UPDATE ... SET` ที่มีแค่คอลัมน์ที่เปลี่ยนจริง) — `ActiveValue<T>` มีสามสถานะพอดีสำหรับความคลุมเครือนี้:

```rust
// นิยามคร่าว ๆ ของ ActiveValue (มาจาก source code จริงของ sea-orm-2.0.3)
pub enum ActiveValue<V>
where
    V: Into<Value>,
{
    Set(V),        // ตั้งค่านี้ชัดเจน (insert ก็ใส่ค่านี้ / update ก็เปลี่ยนเป็นค่านี้)
    Unchanged(V),  // มีค่านี้อยู่ แต่ "ไม่ได้ถูกแก้ไข" — ไม่ต้องส่งไปใน UPDATE
    NotSet,        // ยังไม่ได้ตั้งค่าอะไรเลย (ปล่อยให้ database ใช้ default/auto-increment)
}
```

นี่คือกลไกที่แท้จริงของ **"dirty tracking"** — คำที่ใช้ในโลก ORM หมายถึง "การจดจำว่า field ไหนถูกแก้ไข (dirty) บ้าง เพื่อ save เฉพาะส่วนที่เปลี่ยนจริง" หัวข้อ 73.5 จะพิสูจน์กลไกนี้ด้วย SQL จริงที่ observe ได้ ไม่ใช่แค่อธิบายเชิงทฤษฎี

### 73.4 CRUD เต็มรูปแบบ: ActiveModel Pattern เทียบกับ SQLx

มาเขียน CRUD เต็มรูปแบบบน domain เดียวกับ Part 70 กัน ทุกตัวอย่างในหัวข้อนี้ผู้เขียน**คอมไพล์และรันจริง** ต่อฐานข้อมูล PostgreSQL จริงที่มีข้อมูล seed ไว้แล้ว (หนังสือ 2 เล่ม)

#### Setup: การเชื่อมต่อฐานข้อมูล

```rust
mod entities;
use entities::prelude::*;
use entities::books;
use sea_orm::{Database, DbErr, EntityTrait, ActiveModelTrait, ActiveValue::Set, IntoActiveModel, ModelTrait};

#[tokio::main]
async fn main() -> Result<(), DbErr> {
    let db = Database::connect("postgres://postgres:postgres@127.0.0.1:5432/seaorm_scratch").await?;
    // db มี type เป็น sea_orm::DatabaseConnection — ข้างในห่อ sqlx::PgPool ไว้จริง ๆ (ตามที่หัวข้อ 73.1 พิสูจน์)
    // ...
    Ok(())
}
```

`Database::connect(url).await` คือจุดเทียบเท่า `PgPoolOptions::new().connect(url).await` ของ Part 70 หัวข้อ 70.3 — ต่างกันที่ SeaORM ซ่อน `PgPoolOptions` ไว้ข้างในให้ค่า default ที่สมเหตุสมผล (ถ้าต้องการปรับ `max_connections`/`acquire_timeout` เอง ใช้ `ConnectOptions` ที่ SeaORM เตรียมให้แทน ซึ่งท้ายที่สุดก็ map ไปตั้งค่า `PgPoolOptions` ตัวเดียวกันข้างใต้อยู่ดี)

#### ปรับแต่ง Connection Pool ผ่าน `ConnectOptions`

ถ้าใช้ `Database::connect(url)` ตรง ๆ แบบข้างบน คุณได้ค่า default ของ `PgPoolOptions` ทั้งหมด (เชื่อมกับตารางใน Part 70 หัวข้อ 70.3: `max_connections` default ของ SQLx เอง, `min_connections = 0`, ฯลฯ) — ในระบบจริงที่ต้องคุม pool ให้เหมาะกับ workload มักต้องปรับค่าเหล่านี้เอง ทำผ่าน `ConnectOptions`:

```rust
use sea_orm::{ConnectOptions, Database};
use std::time::Duration;

async fn create_connection(database_url: &str) -> Result<sea_orm::DatabaseConnection, sea_orm::DbErr> {
    let mut opt = ConnectOptions::new(database_url.to_owned());
    opt.max_connections(10)
        .min_connections(1)
        .connect_timeout(Duration::from_secs(5))
        .acquire_timeout(Duration::from_secs(5))
        .idle_timeout(Duration::from_secs(600))
        .sqlx_logging(true); // เปิด/ปิด SQL logging ผ่าน tracing (ใช้ในหัวข้อ 73.5)

    Database::connect(opt).await
}
```

**สังเกตความคล้ายกันแบบเกือบเป๊ะ** กับตาราง `PgPoolOptions` ของ Part 70 หัวข้อ 70.3 (`.max_connections()`, `.min_connections()`, `.acquire_timeout()`, `.idle_timeout()` — ชื่อ method เดียวกันตรงตัว) เพราะ `ConnectOptions` ของ SeaORM ท้ายที่สุดก็แค่**แปลงค่าที่ตั้งไว้ไปตั้งให้ `PgPoolOptions` ของ SQLx ข้างใต้อีกที** — สิ่งเดียวที่ SeaORM เพิ่มเข้ามาเองจริง ๆ คือ `.sqlx_logging(true/false)` ที่คุมว่าจะให้ log SQL statement ผ่าน `tracing`/`log` หรือไม่ (ค่า default เป็น `true` ที่ log level `INFO` ตาม doc comment จริงของ source code — ยืนยันตรงกับ log ที่เห็นจริงในหัวข้อ 73.5 ที่ขึ้นต้นด้วย `INFO` พอดี) ต่างจาก SQLx เปล่า ๆ ที่ต้องตั้ง `tracing_subscriber` เองเสมอจึงจะเห็น SQL — แต่ log target ที่ปรากฏจริงยังคงเป็น `sqlx::query` ตามที่หัวข้อ 73.5 พิสูจน์ไว้ ไม่ใช่ log target ของ SeaORM เอง (มี `.sqlx_logging_level(...)` ให้ปรับ level เองได้ถ้าต้องการ)

#### 1. อ่านข้อมูลทั้งหมด: `Entity::find().all()`

```rust
let all_books = Books::find().all(&db).await?;
println!("all_books count = {}", all_books.len());
for b in &all_books {
    println!("  book: id={} title={} available={}", b.id, b.title, b.available_copies);
}
```

ผลลัพธ์จริง (จากฐานข้อมูลที่ seed ไว้ล่วงหน้า):

```
all_books count = 2
  book: id=1 title=Programming Rust available=3
  book: id=2 title=Rust in Action available=2
```

**เทียบกับ SQLx (Part 70)**: โค้ดเทียบเท่าด้วย SQLx คือ

```rust
let all_books = sqlx::query_as!(Book, "SELECT * FROM books").fetch_all(&pool).await?;
```

หนึ่งบรรทัด (`sqlx::query_as!` + SQL string) เทียบกับหนึ่งบรรทัด (`Books::find()`) — สำหรับ query ง่าย ๆ แบบนี้ความยาวโค้ด**ไม่ต่างกันมาก** แต่สังเกตว่า SeaORM ไม่ต้องเขียน SQL string เลย (`find()` ไม่มี argument อะไรที่เป็น SQL) ในขณะที่ SQLx ต้องมี SQL literal อยู่ในโค้ดเสมอ — นี่คือความต่างเชิง philosophy ที่ชัดที่สุด: SeaORM ซ่อน SQL ไว้ทั้งหมด ให้คุณคิดเป็น "entity operation" แทน "SQL statement"

#### 2. อ่านทีละแถวด้วย Primary Key: `Entity::find_by_id().one()`

```rust
let one = Books::find_by_id(1_i64).one(&db).await?;
println!("find_by_id(1) -> {:?}", one.map(|m| m.title));
```

ผลลัพธ์จริง:

```
find_by_id(1) -> Some("Programming Rust")
```

`.one(&db).await` คืน `Result<Option<Model>, DbErr>` — สังเกตว่าเป็น `Option<Model>` ไม่ใช่ error เมื่อไม่พบแถว (ต่างจาก SQLx's `.fetch_one()` ที่คืน `Err(sqlx::Error::RowNotFound)` เมื่อไม่พบแถวเลย ตามที่ Part 70 สอน) — SeaORM เลือกให้ "ไม่พบ" เป็น `None` ที่ต้องจัดการด้วย pattern matching ปกติ (เชื่อมกับ Part 11 เรื่อง `Option<T>`) ในขณะที่ SQLx เลือกให้เป็น error variant หนึ่ง ทั้งสองแนวคิดถูกทั้งคู่ เพียงแต่เป็นการออกแบบ API ที่ต่างกัน — เวลาแปลงเข้า `AppError` (หัวข้อ 73.10) ต้องระวังจุดนี้ให้ดี: กับ SeaORM คุณต้อง `.ok_or(AppError::NotFound)?` เองตรง ๆ ไม่มี variant `RecordNotFound` ที่ SeaORM จะคืนให้อัตโนมัติจากการ query แถวเดียวที่ไม่เจอ (ต่าง context กับ `DbErr::RecordNotFound` ที่ SeaORM ใช้ในสถานการณ์อื่น เช่น `.update()` ที่ไม่เจอแถวให้แก้)

#### 3. สร้างข้อมูลใหม่: `ActiveModel { ..Default::default() }` แล้ว `.insert()`

```rust
let new_book = books::ActiveModel {
    isbn: Set("978-0-13-468599-1".to_owned()),
    title: Set("The Rust Programming Language".to_owned()),
    author: Set("Steve Klabnik".to_owned()),
    total_copies: Set(5),
    available_copies: Set(5),
    published_year: Set(Some(2023)),
    ..Default::default() // field ที่ไม่ระบุ (เช่น id, created_at) จะเป็น NotSet — ให้ database ใส่ default ให้
};
let inserted = new_book.insert(&db).await?;
println!("inserted book id={} title={}", inserted.id, inserted.title);
```

ผลลัพธ์จริง:

```
inserted book id=3 title=The Rust Programming Language
```

**อธิบาย `..Default::default()`**: `ActiveModel` ที่ SeaORM generate ให้ implement `Default` เสมอ (ทุก field เริ่มเป็น `ActiveValue::NotSet`) — คุณระบุแค่ field ที่**ตั้งใจจะใส่ค่าเอง** ด้วย `Set(...)` ส่วนที่เหลือ (`id` ที่เป็น `BIGSERIAL`/auto-increment, `created_at` ที่มี `DEFAULT now()`) จะเป็น `NotSet` แล้ว SeaORM จะ**ไม่ใส่คอลัมน์นั้นเข้า `INSERT` เลย** ปล่อยให้ PostgreSQL ใช้ default ของตัวเอง (ตรงกับ SQL ที่คุณจะเขียนมือด้วย SQLx ว่า `INSERT INTO books (isbn, title, ...) VALUES ($1, $2, ...)` โดยไม่ระบุ `id`/`created_at` เลย)

`.insert(&db).await` คืน `Model` ที่มี `id` จริงที่ database สร้างให้กลับมาแล้ว (ข้างใต้ SeaORM ใช้ `RETURNING` clause ของ PostgreSQL แบบเดียวกับที่ Part 70 สอน — ไม่ต้อง query แยกไปหา `id` ที่เพิ่ง insert เอง)

**เทียบกับ SQLx (Part 70)**:

```rust
let inserted: Book = sqlx::query_as!(
    Book,
    r#"INSERT INTO books (isbn, title, author, total_copies, available_copies, published_year)
       VALUES ($1, $2, $3, $4, $5, $6)
       RETURNING id, isbn, title, author, total_copies, available_copies, published_year, created_at"#,
    "978-0-13-468599-1", "The Rust Programming Language", "Steve Klabnik", 5, 5, Some(2023)
)
.fetch_one(&pool)
.await?;
```

7 บรรทัดของ SQLx (ต้องพิมพ์ชื่อคอลัมน์ครบสองรอบ — ทั้งใน `INSERT INTO` และ `RETURNING`) เทียบกับ 8 บรรทัดของ SeaORM (`ActiveModel { ... }` + `.insert()`) — ความยาวใกล้เคียงกันสำหรับตารางที่มีคอลัมน์ไม่มาก แต่ **SeaORM ไม่ต้องพิมพ์ชื่อคอลัมน์ซ้ำสองรอบ** (field name ของ `ActiveModel` ตรงกับ `Column` โดยอัตโนมัติ ไม่ต้องเขียนชื่อคอลัมน์ใน `RETURNING` เอง) และไม่มีความเสี่ยงเรื่อง `$1`/`$2`/... เรียงผิดตำแหน่ง (SQLx ต้องนับตำแหน่ง parameter ให้ตรงมือ ในขณะที่ SeaORM ผูก field เข้ากับคอลัมน์ด้วยชื่อ ไม่ใช่ตำแหน่ง)

#### 4. อัปเดตข้อมูล: `.into_active_model()` แล้วแก้ field แล้ว `.update()`

```rust
let mut am = inserted.clone().into_active_model();
am.title = Set("The Rust Programming Language (2nd Ed.)".to_owned());
let updated = am.update(&db).await?;
println!("updated title -> {}", updated.title);
```

ผลลัพธ์จริง:

```
updated title -> The Rust Programming Language (2nd Ed.)
```

**อธิบายทีละขั้น**: `inserted` เป็น `Model` (ข้อมูลที่อ่านมาแล้ว) — `.into_active_model()` (trait `IntoActiveModel` ที่ SeaORM generate ให้อัตโนมัติ) แปลง `Model` เป็น `ActiveModel` โดยให้**ทุก field เป็น `ActiveValue::Unchanged(v)`** (มีค่าอยู่ แต่ยังไม่ถือว่า "ถูกแก้ไข") จากนั้นเราแก้เฉพาะ `am.title = Set(...)` — บรรทัดนี้เปลี่ยนแค่ field `title` จาก `Unchanged` เป็น `Set` (แปลว่า field นี้ "dirty" แล้ว) field อื่นทั้งหมดยังเป็น `Unchanged` เหมือนเดิม — นี่คือหัวใจของ dirty tracking: `ActiveModel` "รู้" ว่ามีแค่ `title` ที่เปลี่ยน ส่วนที่เหลือไม่เปลี่ยน

หัวข้อ 73.5 จะพิสูจน์ด้วย SQL จริงว่า `UPDATE` ที่เกิดขึ้นมีแค่คอลัมน์ `title` ใน `SET` clause จริง ๆ ไม่ใช่ทุกคอลัมน์

**เทียบกับ SQLx (Part 70)**: การอัปเดตแบบเดียวกันด้วย SQLx ต้องเขียน SQL ที่ระบุคอลัมน์ที่จะแก้เองตรง ๆ:

```rust
let updated: Book = sqlx::query_as!(
    Book,
    "UPDATE books SET title = $1 WHERE id = $2 RETURNING id, isbn, title, author, total_copies, available_copies, published_year, created_at",
    "The Rust Programming Language (2nd Ed.)",
    inserted.id
)
.fetch_one(&pool)
.await?;
```

ตรงนี้ SQLx **ชนะในเชิงความชัดเจน** — คุณเห็นตรง ๆ ว่า `UPDATE` แก้แค่ `title` เพราะคุณเขียน SQL เอง ในขณะที่ SeaORM ต้อง "เชื่อ" ว่า dirty tracking ทำงานถูกต้อง (ซึ่งมันทำงานถูกจริงตามที่หัวข้อ 73.5 จะพิสูจน์ — แต่ต้องเข้าใจกลไกนี้ก่อนจะ "เชื่อ" ได้อย่างมีเหตุผล) นี่คือตัวอย่างที่เป็นรูปธรรมของ trade-off เรื่อง abstraction: SeaORM สั้นกว่าเล็กน้อยและไม่ต้องเขียนชื่อคอลัมน์ซ้ำ แต่ต้องเข้าใจกลไกภายในเพิ่มก่อนจะมั่นใจว่า SQL ที่เกิดขึ้นจริงถูกต้อง

#### 5. ลบข้อมูล: `.delete()`

```rust
let br_inserted = /* Model ของ borrow_record ที่ insert ไว้ก่อนหน้า */;
let del_result = br_inserted.delete(&db).await?;
println!("deleted rows = {}", del_result.rows_affected);
```

ผลลัพธ์จริง:

```
deleted rows = 1
```

`.delete(&db)` เรียกตรงบน `Model` (ไม่ต้องแปลงเป็น `ActiveModel` ก่อน เพราะ SeaORM ใช้ primary key ของ `Model` หา `WHERE` clause ให้เองตรง ๆ — ต่างจาก update ที่ต้องผ่าน `ActiveModel` เพื่อบอกว่า field ไหนจะเปลี่ยน) คืน `DeleteResult` ที่มี `.rows_affected: u64` — เทียบกับ SQLx ที่คืน `PgQueryResult` ที่มี `.rows_affected()` เหมือนกัน (concept เดียวกัน ชื่อ method ต่างกันเล็กน้อย)

### 73.5 พิสูจน์ Dirty Tracking ด้วย SQL จริงที่ Observe ได้

หัวข้อ 73.3-73.4 อธิบายว่า `ActiveModel` "ควรจะ" ส่ง `UPDATE` แค่คอลัมน์ที่เปลี่ยนจริง แต่การ "เชื่อคำอธิบาย" ไม่เท่ากับการพิสูจน์จริง — ผู้เขียนเปิด SQL logging จริงผ่าน `tracing-subscriber` (SeaORM ใช้ `tracing`/`log` ผ่าน SQLx ข้างใต้ ยืนยันอีกครั้งว่ามันคือ SQLx จริง ๆ ไม่ใช่ระบบ logging แยกของตัวเอง) แล้วรัน `.update()` จากตัวอย่างหัวข้อ 73.4:

```rust
tracing_subscriber::fmt()
    .with_env_filter("sea_orm=debug,sqlx=debug")
    .init();
// ... ต่อ db แล้วรัน update ตามหัวข้อ 73.4
```

SQL จริงที่ observe ได้จาก log (ตัดมาเฉพาะบรรทัดที่เกี่ยวข้อง — timestamp/formatting ของ `tracing` ตัดออกเพื่อความอ่านง่าย):

```
INFO sqlx::query: summary="UPDATE \"books\" SET \"title\" …"
  db.statement="

UPDATE \"books\" SET \"title\" = $1 WHERE \"books\".\"id\" = $2
RETURNING \"id\", \"isbn\", \"title\", \"author\", \"total_copies\",
          \"available_copies\", \"published_year\", \"created_at\"
"
  rows_affected=1 rows_returned=1 elapsed=1.139011ms
```

**นี่คือหลักฐานที่ชี้ขาด**: `SET` clause มีแค่ `"title" = $1` ตัวเดียว — ทั้งที่ `ActiveModel` ตัวนี้มาจาก `.into_active_model()` ที่เดิมมีค่าอยู่ครบทุก field (`isbn`, `author`, `total_copies`, `available_copies`, `published_year`, `created_at` ทั้งหมดเป็น `Unchanged(v)` ไม่ใช่ `NotSet`) — ถ้า SeaORM ไม่มี dirty tracking จริง มันจะต้องส่ง `UPDATE` ที่มีทุกคอลัมน์ใน `SET` clause (เพราะทุก field มีค่าอยู่) แต่ผลลัพธ์จริงพิสูจน์ว่า SeaORM **แยกแยะ `Set` ออกจาก `Unchanged` ได้จริงตอนสร้าง SQL** — เอาแค่ field ที่เป็น `Set` (คือ `title` ที่เราแก้ตรง ๆ) เข้า `SET` clause เท่านั้น field ที่เหลือที่เป็น `Unchanged` ถูกละไว้ทั้งหมด

ข้อดีเชิงปฏิบัติของสิ่งนี้ชัดเจนมาก: ในตารางที่มีหลายสิบคอลัมน์ การอัปเดตแค่ 1-2 field ด้วย dirty tracking จะส่ง `UPDATE` ที่มีแค่ 1-2 คอลัมน์ใน `SET` clause จริง ๆ (ประหยัด bandwidth เล็กน้อย และที่สำคัญกว่าคือ**ลดโอกาสเขียนทับค่าที่คนอื่นเพิ่งแก้ไปโดยไม่ตั้งใจ** ถ้าไปเทียบกับการเขียนโค้ดที่ดึงข้อมูลเก่ามาทั้งแถวแล้ว `UPDATE` ทุกคอลัมน์กลับไปทั้งที่ตั้งใจแก้แค่คอลัมน์เดียว — ปัญหา "lost update" คลาสสิกที่เกิดจาก concurrent write)

**ข้อสังเกตเพิ่มเติมจาก log เดียวกัน**: log target ที่ปรากฏคือ `sqlx::query` ตรง ๆ (ไม่ใช่ `sea_orm::query` หรืออะไรที่เป็นของ SeaORM เอง) — นี่คือหลักฐานเชิงพฤติกรรมอีกชิ้นที่ยืนยันหัวข้อ 73.1: SQL ที่รันจริงกับฐานข้อมูลถูก log โดย SQLx เอง เพราะ SQLx คือตัวที่ส่ง query ไปจริง ๆ ในระดับ implementation ไม่ใช่แค่ SeaORM ประกาศ dependency ไว้เฉย ๆ

### 73.6 Query Building: Filter, Order, Pagination

#### `.filter()` ด้วย `Column`

```rust
use sea_orm::{ColumnTrait, QueryFilter, QueryOrder};

let filtered = Books::find()
    .filter(books::Column::AvailableCopies.gt(0))
    .order_by_asc(books::Column::Title)
    .all(&db)
    .await?;
```

ผลลัพธ์จริง (มีหนังสือ 3 เล่มในฐานข้อมูลตอนนี้ — 2 เล่ม seed ไว้ + 1 เล่มที่ insert ในหัวข้อ 73.4):

```
filtered (available > 0) count = 3
  Programming Rust
  Rust in Action
  The Rust Programming Language (2nd Ed.)
```

`books::Column::AvailableCopies.gt(0)` มาจาก trait `ColumnTrait` ที่ SeaORM ให้ method เปรียบเทียบมาครบ (`.eq()`, `.ne()`, `.gt()`, `.gte()`, `.lt()`, `.lte()`, `.like()`, `.is_in()`, `.is_null()`, ...) — สังเกตว่า**พิมพ์ชื่อคอลัมน์ผิดจะไม่ compile เลย** เพราะ `Column` เป็น enum จริงที่ generate มาจาก schema จริง (ต่างจาก SQLx ที่พิมพ์ชื่อคอลัมน์ผิดใน SQL string ธรรมดา `sqlx::query()` แบบไม่มี `!` จะไม่รู้จนกว่าจะรันจริง — แต่ถ้าใช้ `query!`/`query_as!` ตามที่ Part 70 สอน ก็ตรวจตอน compile ได้เหมือนกัน เพียงแต่ผ่านกลไกคนละแบบ: SQLx ต่อฐานข้อมูลจริงตอน compile เพื่อตรวจ SQL string, SeaORM ตรวจผ่าน Rust type system ตรง ๆ เพราะ `Column` ถูก generate เป็น enum ไปแล้วตั้งแต่ตอน `sea-orm-cli generate entity`)

`.order_by_asc(...)`/`.order_by_desc(...)` เทียบเท่า `ORDER BY ... ASC`/`ORDER BY ... DESC` ตรง ๆ

#### `.paginate()`: Pagination ที่มีมาให้ในตัว

Part 71 (SQLx query builder เพิ่มเติม) ต้องเขียน `LIMIT`/`OFFSET` มือเองสำหรับ pagination — SeaORM มี helper `.paginate()` ให้ในตัวที่จัดการเรื่องนี้ให้ทั้งหมด:

```rust
use sea_orm::PaginatorTrait;

let paginator = Books::find().order_by_asc(books::Column::Id).paginate(&db, 2); // page_size = 2
let num_pages = paginator.num_pages().await?;
println!("num_pages (page_size=2) = {num_pages}");

use futures_util::stream::StreamExt;
let mut page_num = 0;
let mut pages = paginator.into_stream();
while let Some(page) = pages.next().await {
    let page = page?;
    println!("page {page_num}: {} rows", page.len());
    page_num += 1;
}
```

ผลลัพธ์จริง (3 เล่มทั้งหมด, page_size = 2):

```
num_pages (page_size=2) = 2
page 0: 2 rows
page 1: 1 rows
```

**อธิบาย**: `.paginate(&db, page_size)` คืน `Paginator` ที่มี method ให้ครบ: `.num_pages()` (คำนวณจำนวนหน้าทั้งหมดจาก `COUNT(*)` ที่มันยิง query แยกให้เอง), `.fetch_page(n)` (ดึงหน้าที่ `n` โดยตรง), หรือ `.into_stream()` (แบบที่ใช้ข้างบน — stream ทีละหน้าต่อเนื่องกัน เหมาะกับ scenario ที่ต้อง process ทุกหน้าตามลำดับ) ข้างใต้ SeaORM สร้าง SQL ที่มี `LIMIT`/`OFFSET` ให้เองตามเลขหน้า — คุณไม่ต้องคำนวณ `OFFSET = (page - 1) * page_size` เองแบบที่ SQLx (Part 71) สอนให้ทำมือ นี่คือตัวอย่างที่เป็นรูปธรรมของ "ORM ระดับสูงประหยัดโค้ด boilerplate ที่พบซ้ำ ๆ บ่อยมากในเว็บแอปจริง" — pagination เป็นสิ่งที่ทุกระบบ list ข้อมูลต้องมี การมี helper สำเร็จรูปแบบนี้ลดโอกาสเขียนสูตร `OFFSET` ผิด (บั๊กคลาสสิกคือคำนวณ page เริ่มจาก 0 หรือ 1 สลับกันจนข้อมูลซ้ำ/ขาดไปหนึ่งแถว)

**ข้อสังเกตเรื่อง `.count()` vs `.num_items()`**: SeaORM มีสอง method ที่ฟังดูคล้ายกันแต่ความหมายต่างกันชัดเจน (อ่านจาก doc comment จริงของ source code) — `PaginatorTrait::count(self, db)` นับจำนวนแถวที่ query **ตามที่เป็นอยู่** (ถ้า query มี `.limit()`/`.offset()` ติดมาด้วยอยู่แล้ว จะนับแค่ในกรอบนั้น ผ่าน `SELECT COUNT(*) FROM (...) AS subquery`) ในขณะที่ `Paginator::num_items()` (ที่ `.num_pages()` เรียกใช้ข้างใต้) จะ**ล้าง limit/offset ออกก่อนนับ** เพื่อให้ได้ยอดรวมทั้งหมดสำหรับคำนวณจำนวนหน้า — ถ้าใช้ผิดตัว (เช่น เผลอใช้ `.count()` กับ query ที่ตั้ง `.limit(10)` ไว้ก่อนแล้ว) จะได้ตัวเลขที่ดู "ถูก" แต่ผิดความหมายที่ต้องการ (นับได้สูงสุดแค่ 10 ทั้งที่ข้อมูลจริงมีมากกว่านั้น) — ต้องเลือกใช้ให้ตรงกับคำถามที่ต้องการตอบ: "มีทั้งหมดกี่แถว (เพื่อ paginate)" ใช้ `num_pages()`/`num_items()`, "query นี้ตามที่ตั้งเงื่อนไขไว้นับได้กี่แถว" ใช้ `.count()`

### 73.7 ความสัมพันธ์ (Relations): One-to-Many ระหว่าง Books และ Borrow Records

จากหัวข้อ 73.2 คุณเห็นแล้วว่า `sea-orm-cli generate entity` ตรวจ foreign key constraint (`borrow_records.book_id REFERENCES books.id`) แล้ว generate `Relation` enum ให้อัตโนมัติทั้งสองฝั่ง — หัวข้อนี้มาดูว่าเอา `Relation` ไปใช้โหลดข้อมูลที่เชื่อมโยงกันยังไง (mirroring แนวคิด association ของ Diesel Part 72 — Diesel ก็มี `#[belongs_to]`/`joinable!` แต่ต้องประกาศเองในไฟล์ที่แยกจาก `schema.rs`, SeaORM generate ให้ในไฟล์เดียวจาก introspection เลย)

#### Eager Loading: `.find_with_related()`

```rust
use sea_orm::prelude::*;

let with_related = Books::find()
    .find_with_related(BorrowRecords)
    .all(&db)
    .await?;

for (book, records) in &with_related {
    println!("book '{}' -> {} borrow_records (find_with_related)", book.title, records.len());
}
```

ผลลัพธ์จริง (มี 1 เล่มที่มีการยืมอยู่ จากหัวข้อ 73.4):

```
book 'Programming Rust' -> 0 borrow_records (find_with_related)
book 'Rust in Action' -> 0 borrow_records (find_with_related)
book 'The Rust Programming Language (2nd Ed.)' -> 1 borrow_records (find_with_related)
```

`.find_with_related(BorrowRecords)` คืน `Vec<(books::Model, Vec<borrow_records::Model>)>` — **หนึ่ง query ที่โหลดทั้ง books ทั้งหมดและ borrow_records ที่เกี่ยวข้องมาพร้อมกัน** (ข้างใต้ SeaORM ใช้ `LEFT JOIN` แล้วจัดกลุ่มผลลัพธ์เป็น book → records ให้เองตามฝั่ง Rust — ไม่ต้องเขียน `GROUP BY`/aggregate ฝั่ง SQL เอง) นี่คือ**eager loading** — โหลดข้อมูลเชื่อมโยงมาล่วงหน้าในครั้งเดียว เหมาะกับสถานการณ์ที่รู้แน่ชัดว่าจะต้องใช้ข้อมูลที่เชื่อมโยงกันทุกแถวแน่ ๆ (เช่น list หน้าเว็บที่ต้องโชว์จำนวนครั้งที่หนังสือถูกยืมของทุกเล่มพร้อมกัน) — ป้องกันปัญหา **N+1 query** (ปัญหาคลาสสิกของ ORM ที่ query แถวหลักมาก่อนแล้วค่อย loop query ข้อมูลที่เกี่ยวข้องทีละแถว กลายเป็น 1 + N query ทั้งที่ควรทำเป็น 1-2 query)

#### Lazy Loading: `.find_related()`

```rust
use sea_orm::ModelTrait;

let related = inserted.find_related(BorrowRecords).all(&db).await?;
println!("book {} has {} borrow_records (find_related)", inserted.id, related.len());
```

ผลลัพธ์จริง:

```
book 3 has 1 borrow_records (find_related)
```

`.find_related(BorrowRecords)` เรียกตรงบน `Model` ตัวเดียว (ไม่ใช่บน `Entity` ทั้งตาราง) — คืน query builder ที่โหลดเฉพาะ `borrow_records` ของ book ตัวนี้เท่านั้น (ยิง query แยกอีกครั้งตอนที่คุณเรียก `.all()`/`.one()`) นี่คือ**lazy loading** — โหลดข้อมูลเชื่อมโยงแค่ตอนที่ต้องใช้จริง ๆ เหมาะกับสถานการณ์ที่ไม่แน่ใจว่าจะต้องใช้ข้อมูลเชื่อมโยงหรือไม่ (ประหยัดกว่าถ้าไม่ต้องใช้จริง แต่ถ้าเรียกใน loop ของหลาย ๆ book จะกลายเป็นปัญหา N+1 ทันที — **ต้องเลือกใช้ตามสถานการณ์**: ใช้ `.find_with_related()` เมื่อรู้แน่ว่าต้องใช้ข้อมูลเชื่อมโยงของทุกแถว, ใช้ `.find_related()` เมื่อทำกับแถวเดียวหรือไม่แน่ใจว่าจะต้องใช้)

**ข้อสังเกตเรื่อง `find_also_related`**: SeaORM มี `.find_also_related()` ด้วย ซึ่งคล้าย `.find_with_related()` มาก แต่ต่างกันตรงที่ `.find_also_related()` คืน `Vec<(Model, Option<RelatedModel>)>` (ใช้กับความสัมพันธ์แบบ **one-to-one**/**belongs-to** ที่แต่ละแถวมีความสัมพันธ์ได้แค่หนึ่งแถวหรือไม่มีเลย) ในขณะที่ `.find_with_related()` คืน `Vec<(Model, Vec<RelatedModel>)>` (ใช้กับ **one-to-many** ที่หนึ่งแถวมีความสัมพันธ์ได้หลายแถว — กรณี books-has-many-borrow_records ในบทนี้) เลือกตัวที่ตรงกับ cardinality ของความสัมพันธ์จริง

#### เทียบวิธีโหลดข้อมูลเชื่อมโยงข้ามสามไลบรารี

| | SQLx (Part 70-71) | Diesel (Part 72) | SeaORM (บทนี้) |
|---|---|---|---|
| Eager loading (โหลดพร้อมกันในคราวเดียว) | เขียน `JOIN` เองใน SQL แล้ว map ผลลัพธ์ที่ซ้ำแถวเป็นโครงสร้างที่ต้องการมือ | `.left_join()` + จัดกลุ่มผลลัพธ์เองหลัง query (หรือ query สองรอบแล้ว `.grouped_by()`) | `.find_with_related()`/`.find_also_related()` — จัดกลุ่มให้อัตโนมัติ |
| Lazy loading (โหลดตอนต้องใช้) | เขียน query แยกเองตรง ๆ ไม่มี helper พิเศษ | เขียน query แยกเอง (ไม่มี lazy-load helper ในตัว) | `.find_related()` — เรียกตรงบน `Model` ได้เลย |
| การนิยามความสัมพันธ์ | ไม่มี concept นี้ในระดับ library (คุณคุมทุก join เองผ่าน SQL) | `#[belongs_to]`/`joinable!` ประกาศเองแยกจาก `schema.rs` | `Relation` enum + `Related<T>` — generate อัตโนมัติจาก foreign key ตอน introspect |
| ความเสี่ยง N+1 query | เกิดได้เท่ากับที่โค้ดเขียนเอง (ไม่มี abstraction มาซ่อนปัญหา แต่ก็ไม่มีอะไรมาป้องกันให้อัตโนมัติ) | เกิดได้เหมือนกันถ้าเรียก query แยกใน loop | เกิดได้ถ้าใช้ `.find_related()` ผิดที่ (ตามกับดักข้อ 5) แต่มี `.find_with_related()` ให้เลือกป้องกันได้ตรง ๆ |

ตารางนี้ตอกย้ำภาพที่เห็นมาตลอดบท: ความสัมพันธ์ระหว่างตารางเป็น concept ที่ **SQLx ไม่มีเลยในระดับ library** (เพราะ SQLx ไม่รู้จัก "ตาราง"/"ความสัมพันธ์" อะไรเลย รู้จักแค่ SQL string กับผลลัพธ์ที่ map เข้า struct) ในขณะที่ Diesel และ SeaORM ทั้งคู่มี concept นี้ในระดับ type system แต่ต่างกันที่ Diesel ต้องประกาศ association เองแยกไฟล์ ส่วน SeaORM ได้มาอัตโนมัติจากการ introspect foreign key constraint จริง

### 73.8 Transaction: `db.begin()`, Commit, Rollback

หลักการเหมือนกับที่ Part 70 หัวข้อ 70.9 สอนเรื่อง SQLx เป๊ะ (เพราะข้างใต้คือ transaction ของ SQLx จริง ๆ ตามที่หัวข้อ 73.1 พิสูจน์ไว้) เพียงแต่ API ฝั่ง SeaORM ใช้ ActiveModel แทนการเขียน SQL — มาพิสูจน์ด้วยสถานการณ์เดียวกับ Part 70: การยืมหนังสือต้อง (1) ลด `available_copies` และ (2) สร้าง `borrow_records` พร้อมกันแบบ atomic

#### Transaction ที่สำเร็จ

```rust
use sea_orm::{TransactionTrait, sea_query::Expr, ExprTrait, QueryFilter, ColumnTrait};

let txn = db.begin().await?;

Books::update_many()
    .col_expr(books::Column::AvailableCopies, Expr::col(books::Column::AvailableCopies).sub(1))
    .filter(books::Column::Id.eq(inserted.id))
    .exec(&txn)
    .await?;

borrow_records::ActiveModel {
    book_id: Set(inserted.id),
    borrower_name: Set("Suda".to_owned()),
    ..Default::default()
}
.insert(&txn)
.await?;

txn.commit().await?;
```

ผลลัพธ์จริง (ตรวจสอบค่าหลัง commit):

```
after successful tx: available_copies = 4
```

**อธิบาย**: `db.begin().await?` คืน `DatabaseTransaction` — ใช้แทน `&db` ตรง ๆ ในทุก operation ที่ต้องอยู่ "ใน" transaction เดียวกัน (เทียบเท่า `pool.begin()` คืน `Transaction<'_, Postgres>` ของ SQLx ที่ Part 70 สอน — ต้องส่ง `&mut *tx` ไม่ใช่ `&pool`) `Books::update_many()` ใช้อัปเดตหลายแถวพร้อมกันแบบ bulk (ต่างจาก `.update()` ที่ทำงานกับ `ActiveModel` ตัวเดียว) `.col_expr(column, expr)` ให้เขียน expression ทาง SQL ตรง ๆ ได้ (`available_copies - 1` ไม่ใช่แค่ตั้งค่าคงที่) ผ่าน `Expr::col(...).sub(1)` (ต้อง `use sea_orm::ExprTrait;` เพื่อให้ method `.sub()` ใช้ได้ — ถ้าลืม import trait นี้จะเจอ compiler error ที่บอกตรง ๆ ว่ามี trait ที่ implement ไว้แต่ไม่ได้ import เข้า scope — ดูหัวข้อกับดักท้ายบท)

#### พิสูจน์ Rollback ด้วยโค้ดจริง (mirroring Part 70.9)

เหมือนกับที่ Part 70 พิสูจน์ rollback ด้วยการเทียบค่าก่อน/หลัง — จงใจทำให้คำสั่งที่สองใน transaction ล้มเหลว (`book_id` ที่ไม่มีจริง ละเมิด foreign key):

```rust
let before_rollback = Books::find_by_id(inserted.id).one(&db).await?.unwrap().available_copies;

let txn2 = db.begin().await?;

Books::update_many()
    .col_expr(books::Column::AvailableCopies, Expr::col(books::Column::AvailableCopies).sub(1))
    .filter(books::Column::Id.eq(inserted.id))
    .exec(&txn2)
    .await?;

let bad = borrow_records::ActiveModel {
    book_id: Set(999_999), // ไม่มีอยู่จริง — ละเมิด foreign key constraint
    borrower_name: Set("Ghost".to_owned()),
    ..Default::default()
}
.insert(&txn2)
.await;

match bad {
    Ok(_) => println!("ไม่ควรสำเร็จ"),
    Err(e) => {
        println!("insert ที่สองล้มเหลวตามคาด: {e}");
        txn2.rollback().await?;
    }
}

let after_rollback = Books::find_by_id(inserted.id).one(&db).await?.unwrap().available_copies;
assert_eq!(before_rollback, after_rollback);
```

ผลลัพธ์จริง:

```
insert ที่สองล้มเหลวตามคาด: Query Error: error returned from database: insert or update on table "borrow_records" violates foreign key constraint "borrow_records_book_id_fkey" at line 2608
before_rollback=4 after_rollback=4
```

`before_rollback` และ `after_rollback` **เท่ากันจริง** (`4 == 4`) — พิสูจน์ว่าการ `update_many()` ที่สำเร็จไปแล้วก่อนคำสั่งที่สองจะ fail ถูกยกเลิกไปด้วยจริงตอน `.rollback()` เหมือนกับที่ SQLx พิสูจน์ไว้ใน Part 70 ทุกประการ (เพราะกลไก transaction ข้างใต้เป็น SQLx transaction ตัวเดียวกัน — ผลลัพธ์เหมือนกันไม่ใช่เรื่องบังเอิญ)

**ข้อสังเกตสำคัญ**: error message ที่เห็น (`Query Error: error returned from database: insert or update on table "borrow_records" violates foreign key constraint ...`) ขึ้นต้นด้วย `Query Error:` ซึ่งคือ `Display` implementation ของ `DbErr::Query(...)` variant (ตามที่หัวข้อ 73.10 จะอธิบายละเอียด) ส่วนข้อความหลัง `error returned from database:` คือ**ข้อความเดียวกันตรงตัว**กับที่ SQLx ส่งมาตรง ๆ (เทียบกับ Part 70 หัวข้อ 70.9 ที่ได้ error message เดียวกันเป๊ะ) — นี่คือหลักฐานเชิงพฤติกรรมอีกชิ้นที่ยืนยันว่า SeaORM ไม่ได้สร้าง error message ของตัวเองใหม่ทั้งหมด แค่ห่อ error ของ SQLx ไว้อีกชั้นเท่านั้น

**เหมือนกับ SQLx**: ถ้าไม่เรียก `.commit()` เลย (เช่น `?` return early ก่อนถึง `.commit()`) transaction จะ**rollback อัตโนมัติ**ตอน `DatabaseTransaction` ถูก drop (ผ่าน `Drop` implementation ที่ส่งคำสั่ง `ROLLBACK` ให้ก่อน connection ถูกคืนกลับ pool — พฤติกรรมเดียวกับ SQLx's `Transaction` ตามที่ Part 70 อธิบาย เพราะข้างใต้เป็นกลไกเดียวกัน)

#### ทางเลือกที่สะดวกกว่า: `.transaction()` Closure Helper

นอกจาก `db.begin()`/`.commit()`/`.rollback()` แบบ manual ที่ตรงกับ SQLx ทุกประการ SeaORM ยังมี helper ที่ SQLx ไม่มีในรูปแบบเดียวกัน: `TransactionTrait::transaction()` — รับ closure ที่คืน `Future` แล้ว**จัดการ commit/rollback ให้อัตโนมัติตามผลลัพธ์ของ closure นั้น** (ไม่ต้องเขียน `match`/`.commit()`/`.rollback()` เองเลย):

```rust
use sea_orm::{TransactionError, TransactionTrait};

let result = db
    .transaction::<_, (), DbErr>(|txn| {
        Box::pin(async move {
            Books::update_many()
                .col_expr(books::Column::AvailableCopies, Expr::col(books::Column::AvailableCopies).sub(1))
                .filter(books::Column::Id.eq(book_id))
                .exec(txn)
                .await?;

            borrow_records::ActiveModel {
                book_id: Set(book_id),
                borrower_name: Set("ClosureTest".to_owned()),
                ..Default::default()
            }
            .insert(txn)
            .await?;

            Ok(())
        })
    })
    .await;

match result {
    Ok(_) => println!("closure transaction committed successfully"),
    Err(TransactionError::Connection(e)) => println!("connection error: {e}"),
    Err(TransactionError::Transaction(e)) => println!("transaction rolled back due to: {e}"),
}
```

**กลไก**: ถ้า closure คืน `Ok(_)` — SeaORM `.commit()` ให้อัตโนมัติ ถ้าคืน `Err(_)` — SeaORM `.rollback()` ให้อัตโนมัติ (ไม่ต้องพึ่งพฤติกรรม default ตอน drop เหมือนวิธี manual) ผู้เขียนทดสอบจริงทั้งสอง path — path ที่สำเร็จ:

```
closure transaction committed successfully
```

และ path ที่จงใจให้ query ที่สองล้มเหลว (`book_id = 999_999` ที่ไม่มีจริง เหมือนหัวข้อก่อนหน้า) เขียนโค้ดเดียวกันเป๊ะแต่เปลี่ยนแค่ `book_id` ที่ insert — **ไม่ต้องเรียก `.rollback()` เองเลยแม้แต่บรรทัดเดียว**:

```
auto-rollback แล้วเพราะ: Query Error: error returned from database: insert or update on table "borrow_records" violates foreign key constraint "borrow_records_book_id_fkey" at line 2608
available_copies after both attempts = 4
```

`available_copies after both attempts = 4` (ลดจาก `5` เหลือ `4` แค่ครั้งเดียวจาก transaction แรกที่สำเร็จ ไม่ใช่ `3`) พิสูจน์ว่า transaction ที่สอง (ที่ closure คืน `Err` เพราะ foreign key violation) **rollback การลด `available_copies` ของมันไปด้วยจริง** ไม่ทิ้งผลข้างเคียงไว้เลย แม้จะไม่ได้เขียน `.rollback()` มือสักบรรทัด — ระวังจุดหนึ่ง: type ของ error ที่ได้จาก `.transaction()` คือ `TransactionError<E>` (ไม่ใช่ `DbErr` ตรง ๆ) ที่มีสอง variant คือ `Connection(DbErr)` (เชื่อมต่อ/begin ผิดพลาดตั้งแต่แรก) กับ `Transaction(E)` (closure คืน error ชนิด `E` ที่คุณกำหนด — ในตัวอย่างนี้คือ `DbErr` เพราะ query ข้างในคืน `DbErr`) ต้อง `match` ให้ครบทั้งสอง variant เพื่อดึง error จริงออกมา

**เลือกใช้แบบไหนดี**: ใช้ `.transaction()` closure เมื่อ logic ตรงไปตรงมา (ทำตามลำดับ ถ้า error ที่ไหนก็ rollback ทั้งหมด) เพราะโค้ดสั้นกว่าและ**ไม่มีทางลืมเรียก `.rollback()`** ใช้ `db.begin()`/`.commit()`/`.rollback()` แบบ manual เมื่อ logic ซับซ้อนกว่านั้น (เช่น ต้องตรวจเงื่อนไขบางอย่างแล้ว**เลือกเอง**ว่าจะ commit หรือ rollback โดยไม่ได้ผ่าน `Result` ธรรมดา หรือต้องทำอะไรเพิ่มเติมระหว่างทางที่ไม่ใช่แค่ "สำเร็จ/ล้มเหลว")

### 73.9 Migration ด้วย `sea-orm-migration`: Code-First Direction

หัวข้อ 73.2 แสดง workflow แบบ**database-first** (มีฐานข้อมูลอยู่แล้ว → generate entity) — SeaORM ยังรองรับทิศทางตรงข้ามคือ**code-first**: เขียน migration เป็นโค้ด Rust → รันสร้างฐานข้อมูล → (ถ้าต้องการ) generate entity จากฐานข้อมูลที่สร้างขึ้นมา ผ่าน crate แยก `sea-orm-migration`

#### สร้าง migration crate

```bash
sea-orm-cli migrate init -d migration
```

ผลลัพธ์จริง:

```
Initializing migration directory...
Creating file `migration/src/lib.rs`
Creating file `migration/src/m20220101_000001_create_table.rs`
Creating file `migration/src/main.rs`
Creating file `migration/Cargo.toml`
Creating file `migration/README.md`
Done!
```

`Cargo.toml` ที่ได้ (ต้องเปิด feature `runtime-tokio-rustls`/`sqlx-postgres` เองก่อนจะ build ได้ — template ที่ generate มาเว้น feature ไว้เป็น comment):

```toml
[dependencies.sea-orm-migration]
version = "2.0.0"
features = [
    "runtime-tokio-rustls",
    "sqlx-postgres",
]
```

#### เขียน migration จริง

`migration/src/m20220101_000001_create_table.rs` (แก้จาก template ที่ generate มา — เขียน schema `books` เหมือนหัวข้อ 73.2 แต่สร้างผ่านโค้ด Rust ไม่ใช่ SQL string):

```rust
use sea_orm_migration::{prelude::*, schema::*};

#[derive(DeriveMigrationName)]
pub struct Migration;

#[async_trait::async_trait]
impl MigrationTrait for Migration {
    async fn up(&self, manager: &SchemaManager) -> Result<(), DbErr> {
        manager
            .create_table(
                Table::create()
                    .table(Books::Table)
                    .if_not_exists()
                    .col(pk_auto(Books::Id))
                    .col(string(Books::Isbn).unique_key())
                    .col(string(Books::Title))
                    .col(string(Books::Author))
                    .col(integer(Books::TotalCopies))
                    .col(integer(Books::AvailableCopies))
                    .col(integer_null(Books::PublishedYear))
                    .col(timestamp_with_time_zone(Books::CreatedAt).default(Expr::current_timestamp()))
                    .to_owned(),
            )
            .await
    }

    async fn down(&self, manager: &SchemaManager) -> Result<(), DbErr> {
        manager
            .drop_table(Table::drop().table(Books::Table).to_owned())
            .await
    }
}

#[derive(DeriveIden)]
enum Books {
    Table,
    Id,
    Isbn,
    Title,
    Author,
    TotalCopies,
    AvailableCopies,
    PublishedYear,
    CreatedAt,
}
```

**อธิบาย**: `#[derive(DeriveIden)]` บน enum `Books` generate ให้แต่ละ variant กลายเป็นชื่อ identifier ทาง SQL แบบ type-safe (`Books::Table` → `"books"`, `Books::Isbn` → `"isbn"`) — แทนการเขียน string literal ชื่อตาราง/คอลัมน์ตรง ๆ ที่พิมพ์ผิดได้ง่าย ฟังก์ชัน helper อย่าง `pk_auto()`, `string()`, `integer()`, `integer_null()`, `timestamp_with_time_zone()` มาจาก `sea_orm_migration::schema::*` เป็นตัวช่วยสร้างนิยามคอลัมน์แบบสั้น (ไม่ต้องเขียน `ColumnDef::new(...).integer().not_null()` ยาว ๆ เอง)

#### รัน migration จริง

```bash
export DATABASE_URL="postgres://postgres:postgres@127.0.0.1:5432/seaorm_migration_scratch"
cargo run -- up
```

ผลลัพธ์จริง:

```
Applying all pending migrations
Applying migration 'm20220101_000001_create_table'
Migration 'm20220101_000001_create_table' has been applied
```

ตรวจสอบ schema จริงที่ได้ด้วย `psql`:

```
                                         Table "public.books"
      Column      |           Type           | Collation | Nullable |             Default
------------------+--------------------------+-----------+----------+----------------------------------
 id               | integer                  |           | not null | generated by default as identity
 isbn             | character varying        |           | not null |
 title            | character varying        |           | not null |
 author           | character varying        |           | not null |
 total_copies     | integer                  |           | not null |
 available_copies | integer                  |           | not null |
 published_year   | integer                  |           |          |
 created_at       | timestamp with time zone |           | not null | CURRENT_TIMESTAMP
Indexes:
    "books_pkey" PRIMARY KEY, btree (id)
    "books_isbn_key" UNIQUE CONSTRAINT, btree (isbn)
```

**ข้อสังเกตสำคัญที่พบจริงตอนทดสอบ**: `pk_auto()` ที่ helper `schema::*` ให้มาสร้าง primary key เป็น `INTEGER` (ผ่าน `generated by default as identity`) **ไม่ใช่** `BIGINT`/`BIGSERIAL` แบบที่ Part 70 หัวข้อ 70.5 เลือกใช้ตอนเขียน SQL มือ — ถ้าต้องการ `BIGINT` ต้องใช้ helper อีกตัว (`pk_bigint`) หรือประกาศ `ColumnDef` เองแบบเต็ม ๆ นี่คือรายละเอียดเล็ก ๆ ที่มือใหม่มักพลาด (สมมติว่า helper ที่ "ดูเหมือนจะเป็นค่า default ที่ดี" จะให้ type เดียวกับที่ตัวเองเคยใช้มือ — ควรอ่าน signature ของ helper แต่ละตัวให้แน่ใจก่อนใช้จริงในระบบ production เสมอ)

ตรวจสอบสถานะ migration และ rollback ได้เหมือน `sqlx-cli`:

```bash
$ cargo run -- status
Checking migration status
Migration 'm20220101_000001_create_table'... Applied

$ cargo run -- down
Rolling back 1 applied migrations
Rolling back migration 'm20220101_000001_create_table'
Migration 'm20220101_000001_create_table' has been rolled back
```

**เทียบกับ `sqlx-cli` (Part 70)**: ทั้งสองมี concept เดียวกัน (`up`/`down`, ตารางเก็บ metadata ของ migration ที่รันไปแล้ว) — `sqlx-cli` เก็บ metadata ไว้ในตาราง `_sqlx_migrations` และไฟล์ migration เป็น SQL ดิบ (`.up.sql`/`.down.sql`) ส่วน `sea-orm-migration` เก็บไว้ในตาราง `seaql_migrations` และไฟล์ migration เป็น**โค้ด Rust** (ไม่ใช่ SQL string) — ข้อดีของแบบ Rust คือ portable ข้าม database backend ได้มากกว่า (โค้ดเดียวกันสร้าง schema บน PostgreSQL/MySQL/SQLite ได้ ถ้าไม่ได้ใช้ SQL เฉพาะทางของ backend ใดบักหนึ่ง) ข้อเสียคือต้องเรียนรู้ DSL ของ `sea_orm_migration::schema` เพิ่มเติม แทนที่จะเขียน SQL ที่คุ้นเคยอยู่แล้วตรง ๆ

#### Workflow ที่แนะนำ: ทั้งสองทิศทางไปด้วยกันได้

คำถามที่คนใหม่มักสงสัย: "ควรเขียน entity ก่อนหรือ migration ก่อน" — คำตอบที่ verified จริงในบทนี้คือ**ทั้งสองทิศทางใช้ได้จริงและต่อกันได้เป็น loop เดียวกัน**:

1. **Code-first (แนะนำสำหรับโปรเจกต์ใหม่)**: เขียน `sea-orm-migration` migration → รัน `up` สร้าง schema จริง → รัน `sea-orm-cli generate entity` introspect schema ที่เพิ่งสร้างเพื่อได้ `Entity`/`Model`/`ActiveModel` มาใช้ในแอป (ผู้เขียนทดสอบจริง: generate entity จากฐานข้อมูลที่ migrate ด้วย `pk_auto()` ข้างบน ได้ `id: i32` ตรงกับที่ migration สร้างไว้จริง — พิสูจน์ว่า chain นี้ทำงานสอดคล้องกันจริง ไม่ใช่แค่ทฤษฎี)
2. **Database-first (เหมาะกับฐานข้อมูลที่มีอยู่แล้วก่อน SeaORM เข้ามา)**: มีฐานข้อมูล production/legacy อยู่แล้ว → รัน `sea-orm-cli generate entity` introspect ตรง ๆ (ตามหัวข้อ 73.2) → ไม่ต้องเขียน migration เองเลยถ้าไม่มีความจำเป็นต้องเปลี่ยน schema ต่อไป (แต่ถ้าจะแก้ schema เพิ่มในอนาคต แนะนำเริ่มเขียน migration จากจุดนี้เป็นต้นไป เพื่อให้มีประวัติการเปลี่ยน schema แบบมีเวอร์ชันต่อจากนี้)

จุดที่ต้องระวัง: **entity ที่ generate ไว้ไม่ sync อัตโนมัติกับ migration ใหม่ที่เพิ่มเข้ามาทีหลัง** — ถ้าเขียน migration เพิ่มคอลัมน์ใหม่แล้วลืมรัน `sea-orm-cli generate entity` ซ้ำ entity ในโค้ดจะไม่รู้จักคอลัมน์ใหม่นั้นเลย (compile ผ่านปกติเพราะ Rust ไม่รู้ว่าฐานข้อมูลมีคอลัมน์เพิ่ม แต่โค้ดจะใช้ประโยชน์จากคอลัมน์ใหม่ไม่ได้จนกว่าจะ regenerate) — ต้องจำให้ได้ว่า **migration และ entity เป็นสองไฟล์แยกกันที่ต้อง sync มือ** ไม่ใช่ระบบเดียวที่อัปเดตกันอัตโนมัติ (ต่างจาก Diesel ที่ `schema.rs` ถูก regenerate อัตโนมัติทุกครั้งที่ `diesel migration run` แต่ก็ยังต้องไปแก้ model struct เองอยู่ดีตามที่ Part 72 อธิบาย)

### 73.10 Error Handling: `DbErr` และการแปลงเป็น `AppError`

จาก Part 66/70 คุณมี pattern `AppError` ที่รวม error ทุกแหล่งของแอปไว้ที่เดียวแล้ว implement `IntoResponse` ให้ Axum แปลงเป็น HTTP response ที่เหมาะสม — หัวข้อนี้เชื่อม `DbErr` เข้ากับ pattern เดียวกัน

#### โครงสร้างจริงของ `DbErr` (จาก source code `sea-orm-2.0.3/src/error.rs`)

```rust
// ตัดมาเฉพาะ variant ที่สำคัญที่สุด (enum เต็มมีมากกว่านี้ — #[non_exhaustive] ด้วย)
#[derive(Error, Debug, Clone)]
#[non_exhaustive]
pub enum DbErr {
    #[error("Failed to acquire connection from pool: {0}")]
    ConnectionAcquire(#[source] ConnAcquireErr),
    #[error("Connection Error: {0}")]
    Conn(#[source] RuntimeErr),
    #[error("Execution Error: {0}")]
    Exec(#[source] RuntimeErr),
    #[error("Query Error: {0}")]
    Query(#[source] RuntimeErr),
    #[error("RecordNotFound Error: {0}")]
    RecordNotFound(String),
    #[error("None of the records are updated")]
    RecordNotUpdated,
    // ... อีกหลาย variant (Migration, Custom, Type, Json, ...)
}

#[derive(Error, Debug, Clone)]
pub enum RuntimeErr {
    #[cfg(feature = "sqlx-dep")]
    #[error("{0}")]
    SqlxError(Arc<sqlx::error::Error>), // ห่อ sqlx::Error ไว้ตรง ๆ — ยืนยัน 73.1 อีกครั้ง
    #[error("{0}")]
    Internal(String),
}
```

**ข้อสังเกตสำคัญ**: `#[non_exhaustive]` บน `DbErr` หมายความว่า `match` ที่ครอบทุก variant ต้องมี `_ => ...` เผื่อไว้เสมอ (SeaORM เตือนไว้ตรง ๆ ว่าอาจเพิ่ม variant ใหม่ในเวอร์ชันถัดไปได้โดยไม่ถือเป็น breaking change) — ต่างจาก `sqlx::Error` ที่ Part 70 ใช้ (ก็เป็น `#[non_exhaustive]` เหมือนกันจริง ๆ แล้ว ทั้งสอง error type นี้ต้องเขียน `match` แบบเผื่อ `_` เสมอ)

#### ตรวจจับ Unique Constraint Violation จริง

เช่นเดียวกับ Part 70 หัวข้อ 70.7 มาดูว่าการ insert ISBN ซ้ำได้ error อะไรจริง:

```rust
let dup = books::ActiveModel {
    isbn: Set("978-0-13-468599-1".to_owned()), // ซ้ำกับที่ insert ไปแล้วในหัวข้อ 73.4
    title: Set("Duplicate ISBN Test".to_owned()),
    author: Set("Nobody".to_owned()),
    total_copies: Set(1),
    available_copies: Set(1),
    published_year: Set(None),
    ..Default::default()
}
.insert(&db)
.await;

match dup {
    Ok(_) => println!("ไม่ควรสำเร็จ (ISBN ซ้ำ)"),
    Err(DbErr::Query(sea_orm::RuntimeErr::SqlxError(sqlx_err))) => {
        println!("DbErr::Query (unique violation) = {sqlx_err}");
    }
    Err(e) => println!("DbErr variant อื่น: {e:?}"),
}
```

ผลลัพธ์จริง:

```
DbErr::Query (unique violation) = error returned from database: duplicate key value violates unique constraint "books_isbn_key"
```

ข้อความ**เดียวกันตรงตัว**กับที่ Part 70 หัวข้อ 70.7 ได้จาก SQLx ตรง ๆ (`duplicate key value violates unique constraint "books_isbn_key"`) — เพราะเป็น error ตัวเดียวกันที่ SeaORM แค่ห่อไว้อีกชั้น (`DbErr::Query(RuntimeErr::SqlxError(Arc<sqlx::Error>))`)

#### แปลงเป็น `AppError` พร้อมตรวจ `.is_unique_violation()` (mirroring Part 70.7)

```rust
use sea_orm::{DbErr, RuntimeErr};

#[derive(Debug)]
enum AppError {
    NotFound,
    Conflict(String),
    Internal(String),
}

impl From<DbErr> for AppError {
    fn from(err: DbErr) -> Self {
        // เช็คก่อนว่าเป็น Query error ที่ห่อ sqlx::Error::Database ที่เป็น unique violation หรือไม่
        if let DbErr::Query(RuntimeErr::SqlxError(sqlx_err)) = &err {
            if let sea_orm::sqlx::Error::Database(db_err) = sqlx_err.as_ref() {
                if db_err.is_unique_violation() {
                    return AppError::Conflict(db_err.message().to_string());
                }
            }
        }
        match err {
            DbErr::RecordNotFound(msg) => {
                eprintln!("record not found: {msg}");
                AppError::NotFound
            }
            other => AppError::Internal(other.to_string()),
        }
    }
}
```

ผลลัพธ์จริงจากการรัน (insert ที่ซ้ำ ISBN แล้วแปลงผ่าน `.into()`):

```
mapped AppError = Conflict("duplicate key value violates unique constraint \"books_isbn_key\"")
```

**อธิบายจุดสำคัญ**: `.is_unique_violation()`/`.message()` ที่เรียกในโค้ดนี้คือ method ตัวเดียวกันจาก trait `sqlx::error::DatabaseError` ที่ Part 70 หัวข้อ 70.7 ใช้กับ SQLx ตรง ๆ ทุกประการ — เข้าถึงได้ผ่าน `sea_orm::sqlx::Error::Database(...)` เพราะ SeaORM re-export `sqlx` ให้ตรง ๆ (ตามที่หัวข้อ 73.1 พิสูจน์) นี่คือประโยชน์เชิงปฏิบัติที่จับต้องได้จากการที่ SeaORM สร้างบน SQLx: **ความรู้เรื่อง error handling ที่เรียนจาก Part 70 (SQLx) ใช้ต่อกับ SeaORM ได้ตรง ๆ โดยไม่ต้องเรียนรู้ระบบ error ใหม่ทั้งหมด** — ต่างระดับกับ Diesel (Part 72) ที่มี error type ของตัวเอง (`diesel::result::Error`) ที่ไม่ได้ห่อ SQLx ไว้เลย (เพราะ Diesel ไม่ได้ใช้ SQLx เป็น driver)

**สิ่งที่ต้องระวัง**: `RecordNotFound(String)` ของ `DbErr` **ไม่ใช่**สิ่งที่คุณจะเจอจากการเรียก `.find_by_id().one()` ที่ไม่พบแถว (กรณีนั้นคืน `Ok(None)` ตามหัวข้อ 73.4) — `RecordNotFound` เกิดจาก operation อื่นที่ SeaORM คาดหวังว่าต้องมีแถวอยู่แน่นอนแล้วไม่เจอ (เช่น เรียก `.update()` บน `ActiveModel` ที่ primary key ไม่ตรงกับแถวไหนในตารางเลย) — ต้องแยกให้ถูกว่า "ไม่พบแถวตอน SELECT" (`Option::None`, จัดการด้วย `.ok_or(AppError::NotFound)?`) กับ "ไม่พบแถวตอน UPDATE/DELETE ที่คาดว่าต้องมี" (`DbErr::RecordNotFound`, จัดการผ่าน `impl From<DbErr>`) เป็นคนละกรณีกัน

### 73.11 รวมกับ Axum: `DatabaseConnection` ใน `AppState`

เชื่อมทุกอย่างที่เรียนมาเข้ากับ Axum ตาม pattern เดียวกับ Part 70 หัวข้อ 70.10 (`PgPool` ใน `AppState`) — ต่างกันแค่ type ที่เก็บใน state เปลี่ยนจาก `sqlx::PgPool` เป็น `sea_orm::DatabaseConnection` เท่านั้น ผู้เขียน**รัน Axum server จริงและยิงด้วย `curl` จริง**ทุก endpoint (ไม่ใช่แค่ compile ผ่าน) ต่อฐานข้อมูล PostgreSQL จริง

**ข้อสังเกตแรก**: entity ที่ generate มาตอน 73.2 (`#[derive(DeriveEntityModel)]` เปล่า ๆ) **ไม่มี** `#[derive(Serialize)]` ให้ — ถ้าจะส่ง `Model` กลับเป็น JSON ผ่าน Axum ต้องเพิ่ม derive เอง หรือใช้ flag ของ `sea-orm-cli` ตอน generate:

```bash
sea-orm-cli generate entity -u "$DATABASE_URL" -o src/entities --with-serde both
```

ผู้เขียนทดสอบจริง — ผลลัพธ์ `books.rs` ที่ได้ต่างจากหัวข้อ 73.2 ตรงบรรทัด derive:

```rust
use serde::{Deserialize, Serialize};

#[derive(Clone, Debug, PartialEq, Eq, DeriveEntityModel, Serialize, Deserialize)]
#[sea_orm(table_name = "books")]
pub struct Model {
    // ... fields เหมือนหัวข้อ 73.2 ทุกประการ
}
```

`--with-serde` รับค่าได้สี่แบบ: `none` (ค่า default — ไม่เพิ่ม derive อะไรเลย), `serialize` (เพิ่มแค่ `Serialize` — พอสำหรับ handler ที่แค่**อ่าน**แล้วส่งกลับ JSON), `deserialize` (เพิ่มแค่ `Deserialize` — ใช้กับ payload ที่รับเข้ามา), หรือ `both` (ตัวอย่างข้างบน) — เลือกให้ตรงกับการใช้งานจริง ปกติ `Model` (อ่านอย่างเดียว) มักต้องการแค่ `Serialize` ในขณะที่ payload รับเข้า (เช่น `CreateBookRequest`) ควรเป็น struct แยกที่มี `Deserialize` เพราะ payload จาก client ไม่ควรมี field อย่าง `id`/`created_at` ที่ database ต้องเป็นคนกำหนดเองอยู่ดี (เชื่อมกับ Part 57 เรื่อง Serde: การแยก request DTO ออกจาก entity model เป็น pattern ที่ดีเสมอ ไม่ผูก API contract เข้ากับ database schema ตรง ๆ)

**โค้ดเต็ม** (`AppError` ใช้ pattern จากหัวข้อ 73.10 ตรง ๆ, `AppState` derive `Clone` เพราะ Axum ต้อง clone state ให้ทุก handler — `DatabaseConnection` clone ได้ถูกต้องเพราะข้างในเป็น `Arc<...>` เหมือนที่ Part 70 พิสูจน์ไว้กับ `sqlx::PgPool`):

```rust
use axum::{
    extract::{Path, State},
    http::StatusCode,
    response::{IntoResponse, Json, Response},
    routing::get,
    Router,
};
use entities::{books, prelude::*};
use sea_orm::{
    ActiveModelTrait, ActiveValue::Set, ColumnTrait, Database, DatabaseConnection, DbErr,
    EntityTrait, QueryFilter, RuntimeErr,
};
use serde::Deserialize;

#[derive(Clone)]
struct AppState {
    db: DatabaseConnection,
}

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

impl From<DbErr> for AppError {
    fn from(err: DbErr) -> Self {
        if let DbErr::Query(RuntimeErr::SqlxError(sqlx_err)) = &err {
            if let sea_orm::sqlx::Error::Database(db_err) = sqlx_err.as_ref() {
                if db_err.is_unique_violation() {
                    return AppError::Conflict(db_err.message().to_string());
                }
            }
        }
        AppError::Internal(err.to_string())
    }
}

async fn list_books(State(state): State<AppState>) -> Result<Json<Vec<books::Model>>, AppError> {
    let all = Books::find().all(&state.db).await?;
    Ok(Json(all))
}

async fn get_book(
    State(state): State<AppState>,
    Path(id): Path<i64>,
) -> Result<Json<books::Model>, AppError> {
    let book = Books::find_by_id(id)
        .one(&state.db)
        .await?
        .ok_or(AppError::NotFound)?; // Option<Model> -> AppError::NotFound ตามที่หัวข้อ 73.4 อธิบาย
    Ok(Json(book))
}

#[derive(Deserialize)]
struct CreateBook {
    isbn: String,
    title: String,
    author: String,
    total_copies: i32,
    published_year: Option<i32>,
}

async fn create_book(
    State(state): State<AppState>,
    Json(payload): Json<CreateBook>,
) -> Result<(StatusCode, Json<books::Model>), AppError> {
    let am = books::ActiveModel {
        isbn: Set(payload.isbn),
        title: Set(payload.title),
        author: Set(payload.author),
        total_copies: Set(payload.total_copies),
        available_copies: Set(payload.total_copies), // หนังสือใหม่ ยังไม่มีใครยืม เท่ากับ total_copies
        published_year: Set(payload.published_year),
        ..Default::default()
    };
    let created = am.insert(&state.db).await?; // ISBN ซ้ำ -> DbErr -> AppError::Conflict (409) อัตโนมัติผ่าน `?`
    Ok((StatusCode::CREATED, Json(created)))
}

fn app(db: DatabaseConnection) -> Router {
    let state = AppState { db };
    Router::new()
        .route("/books", get(list_books).post(create_book))
        .route("/books/{id}", get(get_book))
        .with_state(state)
}

#[tokio::main]
async fn main() -> Result<(), DbErr> {
    let db = Database::connect("postgres://postgres:postgres@127.0.0.1:5432/seaorm_scratch2").await?;
    let router = app(db);
    let listener = tokio::net::TcpListener::bind("127.0.0.1:8973").await.unwrap();
    axum::serve(listener, router).await.unwrap();
    Ok(())
}
```

**ผลลัพธ์จริงจากการรัน server แล้วยิงด้วย `curl` จริง** (ฐานข้อมูลว่างเปล่าตอนเริ่ม):

```bash
$ curl -s http://127.0.0.1:8973/books
[]

$ curl -s -X POST http://127.0.0.1:8973/books -H "Content-Type: application/json" \
    -d '{"isbn":"978-1-59327-828-1","title":"Programming Rust","author":"Jim Blandy","total_copies":3,"published_year":2021}'
{"id":1,"isbn":"978-1-59327-828-1","title":"Programming Rust","author":"Jim Blandy","total_copies":3,"available_copies":3,"published_year":2021,"created_at":"2026-09-27T00:48:31.136762Z"}

$ curl -s http://127.0.0.1:8973/books/1
{"id":1,"isbn":"978-1-59327-828-1","title":"Programming Rust","author":"Jim Blandy","total_copies":3,"available_copies":3,"published_year":2021,"created_at":"2026-09-27T00:48:31.136762Z"}

$ curl -s -w "\nHTTP_STATUS:%{http_code}\n" http://127.0.0.1:8973/books/999
{"error":"not found"}
HTTP_STATUS:404

$ curl -s -w "\nHTTP_STATUS:%{http_code}\n" -X POST http://127.0.0.1:8973/books -H "Content-Type: application/json" \
    -d '{"isbn":"978-1-59327-828-1","title":"Dup","author":"X","total_copies":1,"published_year":null}'
{"error":"duplicate key value violates unique constraint \"books_isbn_key\""}
HTTP_STATUS:409
```

ทุกกรณีทำงานตรงตามที่ตั้งใจ: `GET /books` คืน array เปล่าตอนไม่มีข้อมูล, `POST /books` สร้างสำเร็จคืน `201 Created` พร้อม `id`/`created_at` ที่ database กำหนดให้ (สังเกตว่า client ไม่ได้ส่ง `id`/`created_at`/`available_copies` มาเลย — เป็นไปตามที่ `CreateBook` DTO ออกแบบไว้), `GET /books/999` ที่ไม่มีจริงคืน `404` ผ่าน `AppError::NotFound`, และ `POST` ที่ ISBN ซ้ำคืน `409 Conflict` พร้อมข้อความ error ที่มาจาก PostgreSQL ตรง ๆ (ผ่าน `impl From<DbErr> for AppError` ที่หัวข้อ 73.10 เขียนไว้) — โครงสร้างนี้**เหมือนกับที่ Part 70 หัวข้อ 70.10 สอนไว้ทุกประการในระดับ pattern** ต่างกันแค่ชนิดของ state (`DatabaseConnection` แทน `PgPool`) และวิธีเขียน query (ActiveModel แทน SQL string) — ถ้าเข้าใจ Part 70 มาก่อนแล้ว การต่อ SeaORM เข้ากับ Axum ไม่มีอะไรใหม่ในเชิงโครงสร้างเลย

### 73.12 ทดสอบโดยไม่ต้องมีฐานข้อมูลจริง: `MockDatabase`

SeaORM มี feature ที่ SQLx และ Diesel ไม่มีในรูปแบบเดียวกัน: **`MockDatabase`** — connection ปลอมที่ตั้งค่าล่วงหน้าได้ว่า query ไหนควรคืนผลลัพธ์อะไร โดย**ไม่ต้องต่อฐานข้อมูลจริงเลย** เหมาะสำหรับ unit test ที่อยากทดสอบ logic ของ handler/service โดยไม่อยากพึ่งพา PostgreSQL จริงตอนรัน test (ต่างจาก `#[sqlx::test]` ของ Part 70 หัวข้อ 70.13 ที่ยังต้องมีฐานข้อมูลจริงอยู่ดี เพียงแต่จัดการ isolation ให้อัตโนมัติ)

เปิดใช้ผ่าน feature `mock`:

```bash
cargo add sea-orm --features sqlx-postgres,runtime-tokio-rustls,macros,mock
```

ตัวอย่างที่ผู้เขียน**คอมไพล์และรันจริง** (ไม่มี PostgreSQL เกี่ยวข้องเลยในการรันนี้):

```rust
use sea_orm::{DbBackend, EntityTrait, MockDatabase, MockExecResult};

#[tokio::main]
async fn main() {
    let db = MockDatabase::new(DbBackend::Postgres)
        .append_query_results([vec![books::Model {
            id: 1,
            isbn: "978-1-59327-828-1".to_owned(),
            title: "Programming Rust".to_owned(),
            author: "Jim Blandy".to_owned(),
            total_copies: 3,
            available_copies: 3,
            published_year: Some(2021),
            created_at: chrono::Utc::now().into(),
        }]])
        .append_exec_results([MockExecResult {
            last_insert_id: 2,
            rows_affected: 1,
        }])
        .into_connection();

    let books = Books::find().all(&db).await.unwrap();
    println!("mock find().all() -> {} rows, first title = {}", books.len(), books[0].title);

    // ดู SQL ที่ SeaORM "จะส่งจริง" โดยไม่ต้องมีฐานข้อมูลจริงเลย
    let log = db.into_transaction_log();
    for stmt in &log {
        println!("logged statement: {stmt:?}");
    }
}
```

ผลลัพธ์จริงจากการรัน:

```
mock find().all() -> 1 rows, first title = Programming Rust
logged statement: Transaction { stmts: [Statement { sql: "SELECT \"books\".\"id\", \"books\".\"isbn\", \"books\".\"title\", \"books\".\"author\", \"books\".\"total_copies\", \"books\".\"available_copies\", \"books\".\"published_year\", \"books\".\"created_at\" FROM \"books\"", values: Some(Values([])), db_backend: Postgres }] }
```

**อธิบาย**: `.append_query_results([...])` ตั้งค่าล่วงหน้าว่า query ตัวถัดไปที่เรียกผ่าน connection นี้ (ไม่สนว่า SQL จริงจะเป็นอะไร) จะได้ผลลัพธ์เป็น `Vec<Model>` ที่กำหนดไว้ — `Books::find().all(&db)` จึงคืนข้อมูลปลอมที่ตั้งไว้ทันที ไม่มีการต่อเครือข่ายไปที่ไหนเลย และ `.into_transaction_log()` (เรียกได้ก็ต่อเมื่อ `db` ไม่ได้ถูกใช้งานที่อื่นแล้ว — consume `db` ไปเลย) คืน log ของทุก SQL statement ที่ SeaORM **สร้างขึ้นจริง**ตอนเรียก query ต่าง ๆ — **นี่คือหลักฐานเพิ่มเติมที่ตอบ Exercise 4 ได้ตรง ๆ**: SQL ที่ log ออกมาคือ `SELECT "books"."id", "books"."isbn", ..., "books"."created_at" FROM "books"` — **SeaORM ระบุชื่อคอลัมน์ทุกตัวตรง ๆ เสมอ ไม่ได้ใช้ `SELECT *`** (ต่างจากที่อาจสันนิษฐานไปเองโดยไม่ตรวจสอบ) เพราะ `Model` ที่ generate มารู้ชื่อคอลัมน์ทุกตัวอยู่แล้วตั้งแต่ตอน `sea-orm-cli generate entity` จึงระบุชื่อคอลัมน์ตรง ๆ ได้เลยโดยไม่ต้องพึ่ง `SELECT *`

**ข้อจำกัดที่ต้องเข้าใจ**: `MockDatabase` ทดสอบได้แค่ "SeaORM สร้าง SQL ที่ถูกต้องไหม" และ "โค้ดที่เรียก SeaORM ทำงานถูกกับผลลัพธ์ที่ควรได้ไหม" — มันไม่ได้ทดสอบว่า SQL นั้นรันกับ PostgreSQL จริงแล้วได้ผลลัพธ์ถูกจริงไหม (เช่น constraint violation จริง, join ที่ query ผิดเงื่อนไข) การทดสอบแบบนั้นยังต้องมีฐานข้อมูลจริงอยู่ดี (ผ่าน `#[sqlx::test]`-style pattern หรือ container แบบ testcontainers) — `MockDatabase` เหมาะกับการทดสอบ **business logic ระดับ Rust** ที่ประกอบ query หลายจุดเข้าด้วยกัน (เช่น "ถ้า available_copies เป็น 0 ห้าม insert borrow_record") มากกว่าทดสอบว่า SQL ทำงานถูกกับ database จริง

### 73.13 ตารางเปรียบเทียบสุดท้าย: SQLx vs Diesel vs SeaORM

ถึงจุดนี้คุณได้เห็นทั้งสามตัวเลือกทำงานบน domain เดียวกันแล้ว (SQLx: Part 70-71, Diesel: Part 72, SeaORM: บทนี้) — มาสรุปเป็นตารางตัดสินใจที่ตรงไปตรงมา ไม่โฆษณาเกินจริงตัวใดตัวหนึ่ง:

| มิติ | SQLx | Diesel | SeaORM |
|---|---|---|---|
| Async-native | ใช่ ตั้งแต่ต้น | ไม่ใช่ (sync-first, ต้อง `spawn_blocking` หรือ `diesel-async` แยก) | **ใช่ — เพราะสร้างบน SQLx โดยตรง (พิสูจน์แล้วจาก `Cargo.lock`)** |
| ระดับ abstraction | ต่ำสุด — SQL ดิบ | กลาง — Rust DSL ที่ map ใกล้ SQL | สูงสุด — ActiveRecord/Entity เต็มรูปแบบ |
| Compile-time guarantee | สูง (ต่อ SQL string จริงกับ schema จริงผ่าน macro/`.sqlx` cache) | สูงมาก (schema เป็น Rust type ทั้งระบบ ผิด column/type ไม่ compile แน่นอน) | กลาง (`Column` เป็น enum type-safe ป้องกันพิมพ์ชื่อคอลัมน์ผิดได้ แต่ query ที่ generate ไม่ได้ verify กับ DB จริงตอน compile แบบ SQLx) |
| Learning curve | ต่ำ (รู้ SQL อยู่แล้วก็ใช้ได้เกือบทันที) | สูง (ต้องเรียน DSL เต็มระบบ + lifetime ของ query builder ที่ซับซ้อน) | กลาง (ต้องเรียน entity/ActiveModel/dirty tracking แต่ concept คล้าย ORM ภาษาอื่นที่คนจำนวนมากคุ้นเคยอยู่แล้ว) |
| Migration | `sqlx-cli` (SQL ดิบ) | `diesel_cli` (SQL ดิบ + auto-gen `schema.rs`) | `sea-orm-migration` (โค้ด Rust, portable ข้าม backend) **หรือ** introspect ตรง (ไม่ต้องมี migration เลยก็ได้) |
| Workflow สร้าง struct | เขียนมือทั้งหมด | Introspect schema แต่ model struct เขียนมือ | **Generate ให้ครบอัตโนมัติ** |

**ทางออกเมื่อ ActiveModel/query builder เขียน query ที่ต้องการไม่ได้**: บางครั้ง query ซับซ้อนเกินกว่าที่ `.filter()`/`.find_with_related()` จะเขียนได้สะดวก (เช่น subquery ซับซ้อน, window function, CTE) — SeaORM ไม่ได้ปิดทางเขียน SQL ดิบเลย มี escape hatch ผ่าน `Statement::from_sql_and_values()` ร่วมกับ `ConnectionTrait::query_all()`/`.query_one()`:

```rust
use sea_orm::{ConnectionTrait, DatabaseBackend, Statement};

let stmt = Statement::from_sql_and_values(
    DatabaseBackend::Postgres,
    "SELECT id, title FROM books WHERE available_copies > $1 ORDER BY title",
    [1_i32.into()],
);
let rows = db.query_all(stmt).await?; // คืน Vec<QueryResult> — ดึงค่าออกทีละคอลัมน์ด้วย .try_get() เหมือน sqlx::Row
```

นี่คือจุดที่สาม abstraction มาบรรจบกันอีกครั้ง: `query_all()`/`query_one()` ของ SeaORM ท้ายที่สุดก็ส่ง SQL ผ่าน SQLx ข้างใต้เหมือนทุก query อื่นในบทนี้ — แค่ให้คุณ**ข้ามชั้น** entity/ActiveModel ไปเขียน SQL ตรง ๆ ได้เมื่อจำเป็นจริง ๆ โดยไม่ต้องออกจาก SeaORM ทั้งระบบไปพึ่ง SQLx โดยตรง (แม้จะทำแบบนั้นได้เหมือนกันเพราะ `sea_orm::sqlx` re-export ไว้ให้ตามหัวข้อ 73.1)

#### เทียบความยาวโค้ดของ operation เดียวกัน: "Insert หนึ่ง book ใหม่"

ตารางข้างบนเป็นภาพกว้าง — มาดูตัวเลขจริงที่จับต้องได้: ความยาวโค้ด (ไม่รวม import/struct definition) ของ operation เดียวกันเป๊ะ — "insert หนึ่ง book ใหม่พร้อมได้ `id` กลับมา":

**SQLx** (Part 70) — 7 บรรทัด:

```rust
sqlx::query_as!(
    Book,
    r#"INSERT INTO books (isbn, title, author, total_copies, available_copies, published_year)
       VALUES ($1, $2, $3, $4, $5, $6)
       RETURNING id, isbn, title, author, total_copies, available_copies, published_year, created_at"#,
    isbn, title, author, total_copies, available_copies, published_year
)
.fetch_one(&pool).await?
```

**Diesel** (แนวคิดจาก Part 72 — DSL แบบ `insert_into` + `values`) — ประมาณ 10 บรรทัด (ต้องเขียน struct `NewBook` แยกที่มี `#[derive(Insertable)]` ก่อน แล้วค่อยเรียก):

```rust
let new_book = NewBook { isbn, title, author, total_copies, available_copies, published_year };
diesel::insert_into(books::table)
    .values(&new_book)
    .returning(Book::as_returning())
    .get_result(&mut conn)?
```

(หมายเหตุ: Diesel เป็น sync API — ต้องมี `.get_result()` แบบ sync หรือห่อด้วย `spawn_blocking` ถ้าอยู่ใน context async ตามที่ Part 72 อธิบาย ความยาวโค้ดในตารางนี้นับแค่ตัว query เอง ไม่รวมส่วน `spawn_blocking` ที่ต้องเพิ่มถ้าใช้ใน Axum handler)

**SeaORM** (บทนี้) — 8 บรรทัด:

```rust
books::ActiveModel {
    isbn: Set(isbn),
    title: Set(title),
    author: Set(author),
    total_copies: Set(total_copies),
    available_copies: Set(available_copies),
    published_year: Set(published_year),
    ..Default::default()
}
.insert(&db).await?
```

**ข้อสังเกตที่ตรงไปตรงมา**: ความยาวโค้ดของ operation ง่าย ๆ แบบนี้**ไม่ต่างกันมาก** ระหว่างสามตัว (7-10 บรรทัด) — ความต่างที่แท้จริงไม่ได้อยู่ที่ "จำนวนบรรทัด" แต่อยู่ที่ **สิ่งที่คุณต้องรู้ล่วงหน้าเพื่อเขียนโค้ดนี้ได้ถูก**: SQLx ต้องรู้ SQL และชื่อคอลัมน์ครบ (พิมพ์ผิดจับได้ตอน compile ถ้ามี `DATABASE_URL`), Diesel ต้องรู้ DSL ของตัวเองและนิยาม `NewBook`/`Insertable` แยกออกมาก่อน, SeaORM ต้องรู้จัก `ActiveModel`/`ActiveValue::Set`/`NotSet` และเชื่อกลไก dirty tracking ที่ทำงานอยู่ข้างใต้ — สำหรับ query ซับซ้อนกว่านี้ (join หลายตาราง, aggregate, subquery) ความต่างเรื่องความยาวโค้ดและความง่ายจะเห็นชัดขึ้นมาก (SQLx เขียน SQL ตรงได้เสมอไม่ว่าซับซ้อนแค่ไหน แต่ Diesel/SeaORM DSL อาจเขียนบาง query ซับซ้อนได้ยากกว่าหรือต้อง fallback ไปเขียน raw SQL ผ่าน escape hatch ของแต่ละตัว)

#### จุดยืนของหลักสูตรจากนี้ไป

ตามเหตุผลเดียวกับที่ Part 69 (framework comparison สำหรับ Axum/Actix-web) และ Part 72 (Diesel) วางไว้: **หลักสูตรนี้จะใช้ SQLx เป็นตัวหลักต่อไปตั้งแต่ Part 74 เป็นต้นไป** สำหรับทุกตัวอย่างที่ต้องคุยกับฐานข้อมูล เหตุผลไม่ใช่เพราะ Diesel/SeaORM "แย่กว่า" — ทั้งสองตัวเป็นตัวเลือกที่ใช้งานได้จริงในโปรเจกต์จริงจำนวนมาก และมีข้อดีเฉพาะตัวที่ SQLx ไม่มี (Diesel: compile-time guarantee ที่แน่นกว่ามากในระดับ type system, SeaORM: ประสบการณ์แบบ ORM เต็มรูปแบบที่คนจากภาษาอื่นคุ้นเคยเร็ว) แต่เพื่อ**ความสม่ำเสมอของหลักสูตร** (ไม่ต้องสลับ mental model ไปมาระหว่าง "เขียน SQL" กับ "เขียน DSL" กับ "เขียน ActiveModel" ในทุกบทที่เหลือ) และเพราะ SQLx สอนให้เข้าใจ**สิ่งที่เกิดขึ้นจริงระดับ SQL** ซึ่งเป็นความรู้พื้นฐานที่ถ่ายทอดไปใช้กับเครื่องมือ/ภาษาอื่นได้เสมอ ไม่ผูกติดกับ DSL ของ library ใด library หนึ่ง

Diesel และ SeaORM ในหลักสูตรนี้จึงอยู่ในสถานะ **"รู้ว่ามีอยู่ รู้คร่าว ๆ ว่าทำงานอย่างไร และรู้ว่าควรเลือกใช้เมื่อไร"** ไม่ใช่เครื่องมือหลักที่จะใช้ต่อในบทถัด ๆ ไป — ถ้าในงานจริงคุณเจอโปรเจกต์ที่ใช้ Diesel หรือ SeaORM อยู่แล้ว ความเข้าใจจาก Part 72-73 นี้เพียงพอให้อ่านโค้ดที่มีอยู่และเขียนต่อได้ โดยไม่ต้องเรียนรู้ใหม่ทั้งหมด

## กับดักที่พบบ่อย (Common Pitfalls)

**1. ลืม import trait ที่ method ต้องใช้ — `no method named 'sub' found`**

`sea_orm::sea_query::Expr` มี method อย่าง `.sub()`/`.add()`/`.mul()` มาจาก trait `ExprTrait` ที่ต้อง import เข้า scope เอง — ผู้เขียนพลาดจุดนี้จริงตอนเขียนตัวอย่าง transaction แล้วเจอ error จริง:

```
error[E0599]: no method named `sub` found for enum `sea_orm::sea_query::Expr` in the current scope
    |
    = help: items from traits can only be used if the trait is in scope
help: trait `ExprTrait` which provides `sub` is implemented but not in scope; perhaps you want to import it
    |
  1 + use sea_orm::ExprTrait;
```

**วิธีแก้**: เพิ่ม `use sea_orm::ExprTrait;` — compiler บอกวิธีแก้ตรง ๆ อยู่แล้ว แต่คนที่เจอครั้งแรกมักงงว่าทำไม method ที่เห็นใน docs กลับใช้ไม่ได้ (เพราะ Rust ไม่ import trait method อัตโนมัติ ต้องมี trait อยู่ใน scope เสมอ — เชื่อมกับหลักการที่ Part 10 สอนเรื่อง trait และการเรียก method ผ่าน trait)

**2. เข้าใจผิดว่า `.find_by_id().one()` ที่ไม่พบแถวคือ error**

มือใหม่ที่มาจาก SQLx (Part 70) อาจเขียน:

```rust
let book = Books::find_by_id(id).one(&db).await?; // ชนิดเป็น Option<Model> ไม่ใช่ Model
println!("{}", book.title); // compile error — ไม่มี field title บน Option<Model>
```

Compiler จะบอกตรง ๆ ว่า `Option<Model>` ไม่มี field `title` — **วิธีแก้**: ต้อง unwrap/match ก่อนใช้ (`book.ok_or(AppError::NotFound)?.title` หรือ pattern match เต็ม) ต่างจาก SQLx ที่ `.fetch_one()` คืน `Err(RowNotFound)` ตรง ๆ เมื่อไม่พบแถว — ทั้งสองแบบไม่ผิด แค่ต้องรู้ว่ากำลังใช้ library ไหนอยู่ อย่าสับสน mental model ข้ามกัน

**3. แก้ `ActiveModel` โดยไม่ผ่าน `.into_active_model()` ก่อน แล้วงงว่าทำไม field เป็น `NotSet` หมด**

```rust
// ผิด — สร้าง ActiveModel เปล่าใหม่แล้วตั้งใจจะ "update" แค่ title
let mut am = books::ActiveModel::default(); // ทุก field เป็น NotSet
am.id = Set(existing_id);
am.title = Set("New Title".to_owned());
am.update(&db).await?; // compile ผ่าน แต่พฤติกรรมไม่ใช่ที่ตั้งใจ
```

โค้ดนี้ **compile ผ่าน** และแม้จะรันได้ (เพราะ `id` ถูก `Set` แล้วเจอแถวจริงให้ update) แต่แนวคิดสับสน — `ActiveModel::default()` ทุก field เป็น `NotSet` ไม่ใช่ `Unchanged` การ `Set` เฉพาะ `id`/`title` แล้วเรียก `.update()` ยัง**ใช้ได้จริง** เพราะ `.update()` สร้าง `WHERE` จาก primary key ที่ `Set`/`Unchanged` (ไม่ใช่ `NotSet`) แล้วเอาแค่ field ที่ `Set` เข้า `SET` clause — ผลลัพธ์จริง**เหมือนกับ**การใช้ `.into_active_model()` แล้วแก้ field เดียวในกรณีนี้พอดี **แต่**ถ้าคุณมี field อื่นที่ตั้งใจจะ "คงค่าเดิมไว้แต่ยังอยากอ่านค่าปัจจุบันของมันได้ในโค้ดจุดนี้ด้วย" (เช่น validate ค่าเดิมก่อนจะเปลี่ยน) การไม่ผ่าน `.into_active_model()` (ที่ query แถวเดิมมาก่อน) จะทำให้คุณไม่มีค่าเดิมให้ตรวจสอบเลย ต้อง query แยกเองอีกที — **แนวทางที่ปลอดภัยกว่าเสมอ**: ถ้ามี `Model` อยู่แล้ว (จาก query ก่อนหน้า) ใช้ `.into_active_model()` เสมอแทนสร้าง `ActiveModel::default()` เปล่าขึ้นมาเอง

**4. ลืมว่า entity ที่ generate ไว้ไม่ sync อัตโนมัติกับ migration ใหม่**

หลังเขียน migration เพิ่มคอลัมน์ใหม่ (เช่น `books.publisher TEXT`) แล้วรัน `up` สำเร็จ แต่ลืมรัน `sea-orm-cli generate entity` ซ้ำ — โค้ดยัง compile ผ่านปกติ (Rust ไม่รู้ว่าฐานข้อมูลมีคอลัมน์ใหม่) แต่พยายามเข้าถึง `books::Column::Publisher` จะได้:

```
error[E0599]: no variant named `Publisher` found for enum `Column`
```

**วิธีแก้**: รัน `sea-orm-cli generate entity` ซ้ำทุกครั้งหลัง schema เปลี่ยน (ไม่ว่าจะเปลี่ยนผ่าน `sea-orm-migration` หรือ SQL มือตรง ๆ ก็ตาม) — ไม่มีระบบ auto-sync ระหว่าง migration กับ entity ต้องจำสั่งรันเองเสมอ (ควรทำเป็นส่วนหนึ่งของ CI/CD script หรือ pre-commit hook ในทีมจริง เพื่อกันความผิดพลาดจากการลืม)

**5. สับสนระหว่าง `.find_related()` (lazy) กับ `.find_with_related()` (eager) จนเกิด N+1 query**

```rust
// ผิด — เรียก .find_related() ใน loop ของหลาย book กลายเป็น N+1 query จริง
let books = Books::find().all(&db).await?; // 1 query
for book in &books {
    let records = book.find_related(BorrowRecords).all(&db).await?; // ยิง query แยกทุกรอบ loop!
    println!("{}: {} records", book.title, records.len());
}
```

โค้ดนี้ **compile ผ่านและทำงานถูก** แต่ถ้ามี 100 books จะยิง query ไปฐานข้อมูล **101 ครั้ง** (1 ครั้งดึง books + 100 ครั้งดึง records ทีละ book) — ปัญหา N+1 query แบบคลาสสิกของ ORM ทุกตัว **วิธีแก้**: ใช้ `.find_with_related()` แทนเมื่อรู้ว่าต้องใช้ข้อมูลเชื่อมโยงของทุกแถวแน่ ๆ (ตามหัวข้อ 73.7) — ยิงแค่ 1-2 query รวมทุก book แทน

**6. ลืมเปิด feature `macros` แล้ว derive macro หายไปทั้งชุด**

ถ้าเพิ่ม `sea-orm` โดยไม่เปิด feature `macros` (เช่นพิมพ์ `cargo add sea-orm --features sqlx-postgres,runtime-tokio-rustls` ลืม `macros`) โค้ดที่ `sea-orm-cli generate entity` generate มาจะ **compile ไม่ผ่านทั้งไฟล์** เพราะ `#[derive(DeriveEntityModel)]`/`DeriveRelation`/`EnumIter` ทั้งหมดมาจาก feature นี้:

```
error[E0433]: failed to resolve: use of undeclared crate or module `sea_orm`
  --> src/entities/books.rs:5:10
   |
 5 | #[derive(Clone, Debug, PartialEq, Eq, DeriveEntityModel)]
   |          ^^^^^^ could not find `DeriveEntityModel` in this scope
```

(error message จริงจะแตกต่างกันไปเล็กน้อยตามเวอร์ชัน แต่สาระคือ "ไม่รู้จัก derive macro นี้") **วิธีแก้**: ตรวจ `Cargo.toml` ว่ามี `macros` อยู่ใน `features` list ของ `sea-orm` เสมอ — ต่างจาก SQLx (Part 70) ที่ลืม feature `macros` แค่ทำให้ใช้ `query!`/`query_as!` ไม่ได้ (ยังใช้ `query`/`query_as` แบบไม่มี `!` ได้ปกติ) กับ SeaORM การลืม `macros` กระทบทุกอย่างเพราะ `Entity`/`Model`/`ActiveModel` ทั้งระบบพึ่งพา derive macro นี้เป็นรากฐาน ไม่มี "โหมดสำรอง" ที่ใช้ได้โดยไม่มี macro เหมือน SQLx

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เขียนฟังก์ชัน `async fn find_book_by_isbn(db: &DatabaseConnection, isbn: &str) -> Result<Option<books::Model>, DbErr>` ที่ใช้ `Books::find().filter(books::Column::Isbn.eq(isbn)).one(&db).await` — Hint: `.filter()` รับ `Condition`/expression จาก `ColumnTrait` ตรงตามที่หัวข้อ 73.6 สอน ผลลัพธ์ควรเป็น `Option` เพราะอาจไม่พบ ISBN นั้นเลย

2. **(กลาง)** เขียน endpoint (สมมติเป็น handler เปล่า ๆ ไม่ต้องต่อ Axum จริง) `async fn update_available_copies(db: &DatabaseConnection, book_id: i64, delta: i32) -> Result<books::Model, AppError>` ที่ดึง `Model` ปัจจุบันมาก่อน (คืน `AppError::NotFound` ถ้าไม่เจอ) แล้วใช้ `.into_active_model()` แก้แค่ `available_copies` เป็นค่าเดิม + `delta` แล้ว `.update()` — Hint: ต้องเช็คว่าผลลัพธ์ไม่ติดลบก่อน (`CHECK` constraint ของ database จะปฏิเสธถ้าไม่เช็คฝั่ง Rust ก่อน แต่การเช็คฝั่ง Rust ก่อนช่วยให้ error message มีความหมายกว่าการรอ error จาก database)

3. **(ยาก)** เขียนฟังก์ชัน `async fn borrow_book(db: &DatabaseConnection, book_id: i64, borrower_name: &str) -> Result<(), DbErr>` ที่ทำ transaction เต็มรูปแบบ: (1) ตรวจว่า `available_copies > 0` ก่อน (ถ้าไม่ ให้ `rollback` แล้วคืน error ที่มีความหมาย ไม่ใช่ปล่อยให้ constraint จัดการเอง), (2) ลด `available_copies` ลง 1 ผ่าน `update_many().col_expr(...)`, (3) insert แถวใหม่ใน `borrow_records` — Hint: ใช้ `db.begin()` แล้วทำทุกคำสั่งผ่าน `&txn` เหมือนหัวข้อ 73.8 ถ้าเช็คเงื่อนไขข้อ (1) ไม่ผ่าน ให้เรียก `txn.rollback().await?` แล้ว return `Err(DbErr::Custom(...))` ก่อนจะไปถึงขั้นตอนถัดไป

4. **(ยาก/ประยุกต์ใช้งานจริง)** เขียนฟังก์ชันเปรียบเทียบสมรรถนะแบบเบื้องต้น: benchmark การ query "หา book ทั้งหมดที่มี `available_copies > 0` เรียงตาม `title`" ด้วยทั้งสามวิธี (SQLx ตรง ๆ ตาม Part 70, SeaORM ตามหัวข้อ 73.6) วัดเวลาด้วย `std::time::Instant` รันแต่ละแบบ 100 ครั้งแล้วหาค่าเฉลี่ย — Hint: ผลลัพธ์ที่คาดหวังคือความต่างของเวลาน้อยมาก (เพราะ query ที่ SeaORM generate ท้ายที่สุดก็รันผ่าน SQLx ตัวเดียวกันข้างใต้ ตามที่บทนี้พิสูจน์ไว้) — ถ้าเจอความต่างมากผิดปกติ ให้ตรวจสอบว่า SQL ที่ SeaORM generate จริง (ผ่าน SQL logging แบบหัวข้อ 73.5 หรือผ่าน `MockDatabase`/`.into_transaction_log()` แบบหัวข้อ 73.13) ต่างจาก SQL ที่เขียนมือตรงไหน — ผู้เขียนตรวจสอบจริงแล้วพบว่า SeaORM `SELECT` คอลัมน์ทีละชื่อเสมอ (ไม่ใช่ `SELECT *`) เพราะฉะนั้นความต่างที่อาจเจอได้จริงคือจำนวนคอลัมน์ที่ดึงมา ถ้าโค้ด SQLx มือเลือก `SELECT` เฉพาะบางคอลัมน์ที่ต้องใช้จริง ๆ ในขณะที่ SeaORM ดึงมาครบทุกคอลัมน์ของ `Model` เสมอ

## สรุป

บทนี้พาไปรู้จัก **SeaORM** ตัวที่สามและตัวสุดท้ายของสามไลบรารีคุยฐานข้อมูลใน Rust ที่หลักสูตรนี้ครอบคลุม — ข้อเท็จจริงที่สำคัญที่สุดที่บทนี้พิสูจน์ด้วยหลักฐานจริง (`Cargo.lock`, source code, SQL logging) คือ **SeaORM ไม่ใช่คู่แข่งที่แยกจาก SQLx โดยสิ้นเชิง แต่สร้างอยู่บน SQLx โดยตรง** — นี่คือเหตุผลที่มันเป็น async-native ได้แบบเดียวกับ SQLx (Part 70) โดยไม่ต้องมี `spawn_blocking` เหมือน Diesel (Part 72) และเป็นเหตุผลที่ error handling/connection pool/transaction ที่เรียนจาก SQLx ถ่ายทอดมาใช้กับ SeaORM ได้ตรง ๆ โดยไม่ต้องเรียนรู้ใหม่ทั้งหมด

คุณได้เห็น workflow **database-first** ที่ `sea-orm-cli generate entity` introspect ฐานข้อมูลจริงแล้ว generate `Entity`/`Model`/`ActiveModel`/`Column`/`Relation` ให้ครบอัตโนมัติ (ต่างจาก Diesel ที่ introspect ได้แต่ยังต้องเขียน model struct มือ) พร้อม workflow **code-first** ผ่าน `sea-orm-migration` ที่เขียน schema เป็นโค้ด Rust แล้ว generate entity จากผลลัพธ์ที่ migrate ได้เหมือนกัน — คุณเขียน CRUD เต็มรูปแบบผ่าน ActiveRecord pattern (`ActiveModel` + `.insert()`/`.update()`/`.delete()`) และพิสูจน์ด้วย SQL logging จริงว่า **dirty tracking** ทำงานได้จริง (`.update()` ส่งแค่คอลัมน์ที่เปลี่ยนจริงเข้า `SET` clause) พร้อม query building (`.filter()`, `.order_by()`, `.paginate()`), relation loading ทั้ง eager (`.find_with_related()`) และ lazy (`.find_related()`), transaction (`db.begin()`/`.commit()`/`.rollback()`) ที่พิสูจน์ rollback จริงด้วยการเทียบค่าก่อน/หลัง เหมือนที่ Part 70 ทำกับ SQLx และปิดท้ายด้วยการแปลง `DbErr` เป็น `AppError` รวมถึงตรวจจับ unique-constraint violation จริง

ตารางเปรียบเทียบสุดท้ายในหัวข้อ 73.13 สรุปจุดยืนของทั้งสามไลบรารีอย่างตรงไปตรงมา ไม่มีตัวใดที่ "ดีที่สุดในทุกมิติ" — SQLx ให้การควบคุมสูงสุดและ compile-time safety ที่ตรงกับ SQL จริง, Diesel ให้ compile-time guarantee ที่แน่นที่สุดในระดับ type system, SeaORM ให้ประสบการณ์แบบ ORM เต็มรูปแบบที่พัฒนาเร็วสำหรับ CRUD ทั่วไป — หลักสูตรนี้เลือกเดินหน้าต่อด้วย **SQLx** เป็นตัวหลักตั้งแต่ Part 74 เป็นต้นไป เพื่อความสม่ำเสมอและเพราะมันสอนความเข้าใจระดับ SQL ที่ถ่ายทอดข้ามเครื่องมือได้เสมอ — Part 74 จะเปลี่ยนโฟกัสไปเรื่อง **Authentication: JWT** สร้างระบบยืนยันตัวตนที่ต่อกับตาราง `users` ในฐานข้อมูลจริงที่เพิ่งเรียนรู้วิธีคุยด้วยมาตลอด Part 70-73

---

**Part ก่อนหน้า:** [Diesel ORM เบื้องต้น](part-072-diesel-orm.md) | **Part ถัดไป:** [Authentication: JWT](part-074-jwt-authentication.md)
