# Part 71: SQLx: Queries, Migrations, Connection Pooling

> โมดูล: การพัฒนาเว็บแอปพลิเคชัน (Web Development) | ระดับ: สูง | เวลาโดยประมาณ: 280 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- สร้าง `WHERE` clause แบบ dynamic (จำนวนเงื่อนไขไม่แน่นอนตาม input ผู้ใช้) ได้อย่างปลอดภัยด้วย `sqlx::QueryBuilder` โดยไม่ต่อ string เอง — เข้าใจว่า `push()`/`push_bind()` ทำงานต่างกันอย่างไร และทำไม `push_bind()` ถึงยังคง bind parameter ที่ปลอดภัยจาก SQL injection ไว้ได้แม้ SQL จะถูกประกอบขึ้นมาแบบ dynamic
- เขียน endpoint "list + filter + sort + pagination" ในคำสั่งเดียวที่สมบูรณ์แบบเดียวกับที่ระบบจริงต้องมี รวมถึงเทคนิค `COUNT(*) OVER()` (window function) ที่ได้ทั้งข้อมูลหน้าปัจจุบันและจำนวนแถวทั้งหมดในคำสั่ง SQL เดียว ไม่ต้องยิงสอง query แยกกัน
- เข้าใจต้นทุนจริงของการ insert ข้อมูลจำนวนมากทีละแถวในลูป เทียบกับเทคนิค bulk insert ด้วย `UNNEST` — พร้อมตัวเลขวัดจริงที่พิสูจน์ความต่างเชิงประสิทธิภาพ (ต่อยอดจากแนวคิดการวัดผลจริงของ Part 54) และใช้ `WHERE id = ANY($1)` แทนการ query ทีละ id เมื่อต้อง fetch ข้อมูลหลายรายการพร้อมกัน
- เก็บข้อมูลที่ shape ไม่แน่นอนด้วยคอลัมน์ `JSONB` ผ่าน `sqlx::types::Json<T>` ที่ผูกกับ struct/enum ของ Rust ตรง ๆ (ไม่ต้องยุ่งกับ `serde_json::Value` ดิบ ๆ ถ้าไม่จำเป็น) และ query ข้อมูลข้างในผ่าน operator ของ JSONB เอง
- จัดการ migration แบบมีเวอร์ชันในทีมจริง (`sqlx migrate add -r`, `run`, `revert`, `info`) เข้าใจโครงสร้างตาราง `_sqlx_migrations` ที่ SQLx สร้างให้อัตโนมัติ และเขียน migration ที่แก้ schema ตารางเดิมที่มีข้อมูลอยู่แล้วอย่างปลอดภัย (`ALTER TABLE ... ADD COLUMN ... DEFAULT ...`)
- ปรับแต่ง connection pool อย่างมีเหตุผลผ่าน `PgPoolOptions` ครบทุก option (`max_connections`, `min_connections`, `acquire_timeout`, `idle_timeout`, `max_lifetime`) โดยผูกกับโมเดล concurrency ของ Tokio (Part 48) และวินิจฉัยอาการ "API timeout พร้อมกันหมดตอน traffic สูง" ในโลกจริงได้ พร้อมเข้าใจกลไก statement cache ของ SQLx ที่ช่วยลด overhead ของการ `PREPARE` query ซ้ำ ๆ
- ใช้ `sqlx::postgres::PgListener` รับ `NOTIFY` จาก PostgreSQL แบบ real-time ในระดับที่เพียงพอสำหรับเข้าใจกลไก และรู้ขอบเขตของมันเทียบกับ message queue เต็มรูปแบบ (Part 82) พร้อมเขียน test ที่แยก transaction ต่อ test แบบ rollback เพื่อความเร็วและ isolation (ต่อยอด Part 32-33)
- ตรวจจับปัญหา **N+1 query** ในโค้ดจริง (query ทีละแถวในลูปแทนที่จะ join ครั้งเดียว) และแก้ด้วย `JOIN`/`WHERE ... = ANY($1)` พร้อมตัวเลขวัดจริงที่แสดงความต่างทั้งจำนวน query และเวลาที่ใช้

## ความรู้ที่ต้องมีมาก่อน

- **Part 70 (Database: เชื่อมต่อ PostgreSQL ด้วย SQLx)**: บทนี้เป็นภาคต่อโดยตรง ไม่สอน CRUD พื้นฐาน, `PgPoolOptions` เบื้องต้น, `#[derive(sqlx::FromRow)]`, การแปลง `sqlx::Error` เป็น `AppError`, transaction พื้นฐาน (`pool.begin()`/`commit()`/`rollback()`), หรือ SQL injection พื้นฐานซ้ำอีก — ถ้าจุดไหนไม่แน่นให้กลับไปอ่าน Part 70 ก่อน บทนี้จะอ้างอิงกลับไปตลอดโดยไม่อธิบายใหม่ ระบบ, domain (`books`/`borrow_records`), และ `AppState`/`PgPool` pattern ที่ใช้ในบทนี้คือตัวเดียวกันกับที่ Part 70 สร้างไว้ทุกประการ
- **Part 54 (Benchmarking และการวัดประสิทธิภาพ)**: หัวข้อ bulk insert และ N+1 ของบทนี้ยึดหลักการเดียวกับที่ Part 54 สอนไว้เป๊ะ — "ถ้าวัดได้จริง อย่าเดา" ตัวเลขที่แสดงในบทนี้ทุกตัวมาจากการรันจริงแล้ววัดเวลาจริงด้วย `std::time::Instant`
- **Part 48 (Tokio Runtime) และ Part 39 (Shared State Concurrency)**: หัวข้อ connection pool tuning ผูกกับความเข้าใจเรื่อง cooperative scheduling และจำนวน task ที่ทำงานพร้อมกันได้จริงโดยตรง (ต่อยอดจาก Part 70 หัวข้อ pool exhaustion)
- **Part 32-33 (Testing: Unit และ Integration)**: หัวข้อ testing ด้วย transaction-rollback pattern ต่อยอดแนวคิด test isolation ที่ Part 32-33 วางพื้นฐานไว้ และเทียบกับ `#[sqlx::test]` ที่ Part 70 หัวข้อ 70.13 สอนไปแล้ว
- **Part 57 (Serde เบื้องต้น)**: หัวข้อ JSONB ใช้ `#[derive(Serialize, Deserialize)]` กับ enum ที่มี tag ตรงตามที่ Part 57 สอนเรื่อง serde attribute
- **Part 82 (จะมาถึงในหลักสูตร — Message Queue)**: บทนี้แค่แนะนำ `LISTEN`/`NOTIFY` ในระดับ awareness และพูดถึง Part 82 ล่วงหน้าแบบเบา ๆ ว่าเป็นอีกแนวทางหนึ่งที่ "หนักกว่า" สำหรับงานคิวข้อความจริงจัง — ไม่ต้องอ่าน Part 82 มาก่อนเพื่อเข้าใจบทนี้

## หมายเหตุเรื่องการตรวจสอบเนื้อหา (สำคัญ — อ่านก่อนเริ่ม)

ก่อนเขียนบทนี้ ผู้เขียนตรวจสอบสภาพแวดล้อมเดียวกับที่ Part 70 อธิบายไว้ และพบว่า **PostgreSQL 16.13 ยังติดตั้งและรันอยู่จริง** (`pg_lsclusters` ยืนยันสถานะ `online` อยู่แล้วโดยไม่ต้องสั่ง start เพิ่ม) จึงเลือกเส้นทางเดียวกับ Part 70 คือ **ทดสอบทุกตัวอย่างในบทนี้แบบ compile และ run จริง เชื่อมต่อฐานข้อมูลจริง** ทั้งหมด — นี่ไม่ใช่ทางเลือกสำรอง แต่คือเส้นทางที่ดีที่สุดที่ทำได้จริงในสภาพแวดล้อมนี้ เช่นเดียวกับที่ Part 70 ทำไว้

รายละเอียดสิ่งที่ทดสอบจริงสำหรับบทนี้โดยเฉพาะ:

- สร้างฐานข้อมูลใหม่แยกจาก Part 70 ชื่อ `part71_scratch` (คนละตัวกับ `rust_course_scratch` ของ Part 70 เพื่อไม่ชนกับ agent อื่นที่อาจกำลังทดสอบ Part 70 พร้อมกัน) และตั้ง scratch cargo project แยกไว้นอก repo ทั้งหมด (ลบทิ้งหลังเขียนบทนี้เสร็จ)
- รัน migration จริงสองรอบด้วย `sqlx-cli` (สร้างตาราง `books`/`borrow_records` ตาม schema ของ Part 70 ก่อน แล้วรัน `ALTER TABLE` เพิ่ม `category`/`metadata` จริง) และ query ตาราง `_sqlx_migrations` ตรง ๆ ด้วย `psql` เพื่อเอาเนื้อหาจริงมาแสดงในบทนี้ — ไม่มีการแต่งข้อมูลใน `_sqlx_migrations` ขึ้นเอง
- Seed ข้อมูลจริง 40 เล่ม + `borrow_records` 200 แถว แล้ววัดเวลาจริงของ: insert ทีละแถวในลูป vs `UNNEST` bulk insert (2,000 แถว), naive N+1 loop vs single `JOIN` (200 แถว `borrow_records`), และ concurrent task 12 ตัวชิง connection pool ที่ตั้ง `max_connections` ต่างกัน (2 เทียบกับ 12) — ทุกตัวเลขในบทนี้คัดลอกมาจาก stdout ของโปรแกรมที่รันจริง
- ทดสอบ `sqlx::QueryBuilder` จริงกับ query ที่มี filter หลายแบบผสมกัน พร้อม `COUNT(*) OVER()` และพิมพ์ SQL ที่ `QueryBuilder` ประกอบให้ดูจริงผ่าน `.sql()`
- ทดสอบ `sqlx::types::Json<T>` กับ enum ที่ tag ด้วย `#[serde(tag = "kind")]` จริง ยืนยัน round-trip (insert แล้วอ่านกลับมาตรงกับค่าเดิมทุกประการด้วย `assert_eq!`) และ query ผ่าน operator `->>` ของ JSONB จริง
- ทดสอบ `sqlx::postgres::PgListener` จริงแบบ end-to-end: เปิด listener, spawn task แยกส่ง `pg_notify()`, listener อีกฝั่งรับ payload จริงได้ถูกต้อง
- ทดสอบ pattern "เปิด transaction ในทุก test แล้วไม่ commit เลย" จริงด้วย `#[tokio::test]` สองตัวที่รันพร้อมกัน พิสูจน์ว่าข้อมูลไม่หลุดออกจาก transaction ที่ไม่ commit
- สร้าง Axum server จริง (`capstone_server.rs`) รวม pagination+filtering endpoint และ N+1-avoiding endpoint เข้าด้วยกัน แล้วยิง `curl` จริงหลายกรณี
- จงใจทำผิดสองจุดเพื่อจับ error จริง: `ALTER TABLE ... ADD COLUMN ... NOT NULL` (ไม่มี `DEFAULT`) บนตารางที่มีข้อมูลอยู่แล้ว, และ `sqlx::query!` ที่อ้างถึงคอลัมน์ที่ยังไม่มี migration รองรับ

เวอร์ชัน `sqlx`/`sqlx-cli` ที่ใช้คือ `0.9.0` เดียวกับ Part 70 (ตรวจสอบด้วย `cargo add`/`sqlx --version` ในเครื่องจริงตอนเขียนบทนี้) — เช่นเดียวกับที่ Part 70 เตือนไว้ ถ้า ecosystem อัปเดตเวอร์ชันใหม่กว่านี้ตอนคุณอ่าน ให้ยึดผลลัพธ์จาก `cargo add`/`cargo build` ในเครื่องคุณเป็นความจริงล่าสุดเสมอ

## เนื้อหา

### 71.1 ทวนจาก Part 70 และภาพรวมของบทนี้

Part 70 วางพื้นฐานที่จำเป็นทั้งหมดไว้แล้ว: `PgPoolOptions` สร้าง pool, `#[derive(sqlx::FromRow)]` map แถวข้อมูลเข้า struct, `sqlx::query!`/`query_as!` ตรวจสอบ SQL กับ schema จริงตอน compile time, transaction การันตี atomicity, และการแปลง `sqlx::Error` เป็น HTTP response ที่เหมาะสม — สิ่งเหล่านี้คือ "ไวยากรณ์พื้นฐาน" ของการคุยกับฐานข้อมูลจาก Rust ที่ทุกแอปต้องมี

แต่ CRUD พื้นฐานเพียงอย่างเดียวไม่เพียงพอสำหรับระบบจริงที่ต้องรองรับ traffic จริง มีข้อมูลจริงหลักพัน-หลักล้านแถว และต้องอยู่รอดได้เมื่อ schema ต้องเปลี่ยนแปลงไปตามความต้องการทางธุรกิจที่เปลี่ยนไป บทนี้ตอบคำถามเชิงลึกที่ทุกทีมที่ใช้ SQLx ในงานจริงต้องเจอไม่ช้าก็เร็ว:

- ผู้ใช้อยากกรองข้อมูลด้วยเงื่อนไขที่ **ไม่รู้ล่วงหน้าว่ามีกี่ตัว** (บางครั้งกรอง category บางครั้งกรองทั้ง category และช่วงราคา) — จะเขียน SQL ที่ปลอดภัยแบบ dynamic ได้อย่างไรในเมื่อ `query!`/`query_as!` ต้องการ SQL literal ที่ตายตัว?
- ตารางมีข้อมูลหลักแสนแถว จะ list พร้อม pagination โดยไม่ query สองรอบ (รอบหนึ่งหาข้อมูล อีกรอบหา `COUNT(*)` รวม) ได้อย่างไร?
- ต้อง insert ข้อมูลนำเข้า (import) ครั้งละหลายพันแถว insert ทีละแถวในลูปช้าขนาดไหนจริง ๆ เทียบกับเทคนิคที่ดีกว่า?
- schema ต้องเปลี่ยนหลังจากมีข้อมูลจริงอยู่แล้ว จะเพิ่มคอลัมน์ใหม่โดยไม่ทำแถวเก่าพังได้อย่างไร และทีมจะรู้ได้อย่างไรว่า migration ไหนถูก apply ไปแล้วบ้างในฐานข้อมูลไหน?
- `max_connections` ควรตั้งเท่าไร ทำไม API บางระบบถึง timeout พร้อมกันหมดตอน traffic สูงขึ้นกะทันหัน?
- โค้ดที่ "ดูปกติ" (วน loop query ทีละแถว) ทำไมถึงเป็นปัญหาประสิทธิภาพที่ร้ายแรงที่สุดอันดับต้น ๆ ของแอปที่ใช้ ORM/query library ทุกตัว?

บทนี้ตอบทุกคำถามข้างบนด้วยโค้ดจริงที่รันได้จริง ไม่ใช่แค่คำอธิบายเชิงทฤษฎี

### 71.2 Query Patterns: Dynamic WHERE ด้วย `QueryBuilder`

#### ทำไม `query!`/`query_as!` ใช้กับ filter แบบ dynamic ไม่ได้ตรง ๆ

จาก Part 70 หัวข้อ 70.4 คุณรู้แล้วว่า `sqlx::query!`/`query_as!` ต้องการ SQL เป็น **string literal ที่ตายตัวเห็นได้ตอน compile** (macro ต้องเห็น SQL ทั้งก้อนเพื่อไปตรวจสอบกับ schema จริง) — นี่คือข้อจำกัดที่ชนกับสถานการณ์จริงที่พบบ่อยมาก: endpoint `GET /books?category=fiction&min_available=2&sort=desc` ที่ query parameter ทุกตัวเป็น**ตัวเลือก** ผู้ใช้อาจส่งมาแค่บางตัว หรือไม่ส่งมาเลยก็ได้ ทำให้จำนวนเงื่อนไขใน `WHERE` clause ไม่แน่นอนล่วงหน้า

วิธีที่ **ผิด** (และเป็นเหตุผลที่ Part 70 หัวข้อ 70.6 เตือนไว้อย่างหนัก) คือต่อ string เอาเงื่อนไขที่ต้องการเข้าไปตรง ๆ:

```rust
// ❌ ตัวอย่าง "ผิด" — ต่อ SQL string เองตามเงื่อนไขที่มี (ห้ามทำแบบนี้เด็ดขาด)
async fn find_by_category_unsafe(pool: &sqlx::PgPool, category: &str) -> Result<i64, sqlx::Error> {
    let sql = format!("SELECT COUNT(*) FROM books WHERE category = '{category}'");
    let count: i64 = sqlx::query_scalar(&sql).fetch_one(pool).await?;
    Ok(count)
}
```

ผู้เขียนลองคอมไพล์โค้ดนี้จริงในโปรเจกต์ scratch เดียวกับ Part 70 ใช้ — และเจอสิ่งเดียวกันที่ Part 70 พิสูจน์ไว้: SQLx 0.9.0 **ปฏิเสธตั้งแต่ compile time** ด้วย error จริง:

```
error[E0277]: dynamic SQL strings should be audited for possible injections
   --> src/bin/unsafe_dynamic_filter.rs:6:41
    |
6   |     let count: i64 = sqlx::query_scalar(&sql).fetch_one(pool).await?;
    |                      ------------------ ^^^^ dynamic SQL string
    |                      |
    |                      required by a bound introduced by this call
    |
    = help: the trait `SqlSafeStr` is not implemented for `&std::string::String`
    = note: prefer literal SQL strings with bind parameters or `QueryBuilder` to add dynamic data to a query.

            To bypass this error, manually audit for potential injection vulnerabilities and wrap with `AssertSqlSafe()`.
            For details, see the docs for `SqlSafeStr`.

    = note: this trait is only implemented for `&'static str`, not all `&str` like the compiler error may suggest
```

สังเกตข้อความ `= note:` ท้ายสุดที่เพิ่มมา (ต่างจาก error เดิมที่ Part 70 เจอเล็กน้อย เพราะบริบทต่างกัน) — ย้ำชัดเจนว่า trait `SqlSafeStr` implement ให้แค่ `&'static str` เท่านั้น ไม่ใช่ `&str` ทั่วไปแบบที่มือใหม่อาจเข้าใจผิด ตัว compiler ยัง suggest ทางแก้ที่ **ผิด** ด้วยซ้ำ (`consider dereferencing here` → `&*sql`) ถ้าลองทำตามจะยังคง error เดิม เพราะปัญหาไม่ใช่เรื่อง reference แต่เป็นเรื่องที่ SQL string นั้น "มาจาก `format!()`" ซึ่งไม่มีทาง implement `SqlSafeStr` ได้เลยไม่ว่าจะห่อ reference แบบไหนก็ตาม — นี่คือตัวอย่างที่ดีว่า suggestion ของ compiler ไม่ได้ถูกต้องเสมอไปในทุกสถานการณ์ ต้องเข้าใจ**สาเหตุที่แท้จริง**ของ error ก่อนทำตาม suggestion แบบไม่คิด

ทางแก้ที่ถูกต้องคือสิ่งที่ Part 70 หัวข้อ 70.6 แปะท้ายไว้ว่า "จะกล่าวถึงใน Part 71": **`sqlx::QueryBuilder`**

#### `QueryBuilder`: ประกอบ SQL แบบ dynamic โดยไม่เสีย bind parameter safety

`QueryBuilder<DB>` คือ struct ที่ให้คุณ**ต่อ SQL เป็นท่อน ๆ** (ผ่าน `.push()`) สลับกับ**bind ค่าจาก input ผู้ใช้อย่างปลอดภัย** (ผ่าน `.push_bind()`) — ความต่างสำคัญระหว่างสองเมธอดนี้คือ:

- **`.push(sql)`** — เติม SQL **fragment ที่เป็น literal ของคุณเอง** (ชื่อ column, keyword `AND`/`ORDER BY`, ฯลฯ) เข้าไปในคำสั่งที่กำลังสร้าง — สิ่งที่ push เข้าไปด้วยเมธอดนี้ **ต้องไม่ใช่ค่าที่มาจาก input ผู้ใช้ตรง ๆ** เพราะมันจะถูกต่อเข้า SQL string จริง ๆ (ไม่ผ่าน bind parameter) — ถ้า push ค่าจาก user input ผ่าน `.push()` ก็จะเปิดช่องโหว่ SQL injection แบบเดียวกับ `format!()` ทันที
- **`.push_bind(value)`** — เติม**placeholder** (`$1`, `$2`, ...) เข้าไปใน SQL string และเก็บ `value` ไว้ bind แยกต่างหาก (แบบเดียวกับ `.bind()` ปกติ) — **ค่าที่มาจาก input ผู้ใช้ต้องผ่านเมธอดนี้เท่านั้นเสมอ** ไม่มีทางถูกตีความเป็น SQL syntax ได้เลยไม่ว่าค่าจะมีอะไรปนอยู่ก็ตาม (หลักการเดียวกับ `.bind()` ที่ Part 70 พิสูจน์ไว้ทุกประการ)

มาดูตัวอย่างจริงที่รวม filter สามแบบเข้าด้วยกัน (category, จำนวนขั้นต่ำที่เหลือ, และลำดับการ sort) — ทุกฟิลด์เป็น `Option` เพราะผู้ใช้อาจไม่ส่งมาก็ได้:

```rust
use serde::Serialize;
use sqlx::postgres::Postgres;
use sqlx::QueryBuilder;

#[derive(Debug, Serialize, sqlx::FromRow)]
struct BookRow {
    id: i64,
    title: String,
    category: String,
    available_copies: i32,
    total_count: i64, // มาจาก COUNT(*) OVER() — อธิบายในหัวข้อ 71.3
}

#[derive(Default)]
struct ListParams {
    category: Option<String>,
    min_available: Option<i32>,
    sort_desc: bool,
    limit: i64,
    offset: i64,
}

async fn list_books_dynamic(
    pool: &sqlx::PgPool,
    params: &ListParams,
) -> Result<Vec<BookRow>, sqlx::Error> {
    // "WHERE 1 = 1" เป็นเทคนิคคลาสสิกที่ทำให้เงื่อนไขถัดไปทุกตัวเริ่มด้วย "AND ..." ได้เหมือนกันหมด
    // ไม่ต้องเช็คแยกว่า "นี่คือเงื่อนไขแรกหรือเปล่า" (ไม่ต้องมี if/else ต่างกันสำหรับเงื่อนไขแรก)
    let mut qb: QueryBuilder<Postgres> = QueryBuilder::new(
        "SELECT id, title, category, available_copies, COUNT(*) OVER() AS total_count FROM books WHERE 1 = 1",
    );

    if let Some(cat) = &params.category {
        qb.push(" AND category = ");   // .push() — literal SQL fragment ที่เราเขียนเอง ปลอดภัย
        qb.push_bind(cat.clone());     // .push_bind() — ค่าจาก input ผู้ใช้ ต้อง bind เท่านั้น
    }

    if let Some(min_avail) = params.min_available {
        qb.push(" AND available_copies >= ");
        qb.push_bind(min_avail);
    }

    if params.sort_desc {
        qb.push(" ORDER BY id DESC");
    } else {
        qb.push(" ORDER BY id ASC");
    }

    qb.push(" LIMIT ");
    qb.push_bind(params.limit);
    qb.push(" OFFSET ");
    qb.push_bind(params.offset);

    let query = qb.build_query_as::<BookRow>();
    query.fetch_all(pool).await
}
```

**อธิบายทีละส่วน**: `QueryBuilder::new("...")` เริ่มต้นด้วย SQL คงที่ส่วนที่ไม่เปลี่ยนแปลง (`SELECT ... WHERE 1 = 1`) จากนั้นแต่ละ `if let Some(...)` เพิ่มเงื่อนไขเข้าไปเฉพาะเมื่อ field นั้นมีค่าจริง — สังเกตว่า**ไม่มีจุดไหนเลย**ที่ค่าจาก `params` ถูกเอาไปต่อกับ SQL string ตรง ๆ ทุกค่าที่มาจากภายนอก (`cat`, `min_avail`, `params.limit`, `params.offset`) ผ่าน `.push_bind()` เท่านั้น ส่วนที่ผ่าน `.push()` มีแค่ SQL keyword ที่เราเขียนเองในโค้ด (ไม่มาจาก input) — นี่คือหลักการที่ทำให้ SQL แบบ dynamic ยังปลอดภัยจาก SQL injection เท่ากับ `query!`/`query_as!` ทุกประการ แม้จะไม่ใช่ SQL literal ที่ตายตัวก็ตาม

`qb.build_query_as::<BookRow>()` แปลง `QueryBuilder` ที่ประกอบเสร็จแล้วเป็น query object พร้อม type-check กับ `BookRow` ผ่าน `#[derive(sqlx::FromRow)]` (เหมือน `query_as::<_, BookRow>(...)` ปกติ) — **ข้อสังเกตสำคัญ**: `QueryBuilder` ไม่ใช่ `query!`/`query_as!` จึงไม่มีการตรวจสอบ SQL กับ schema จริงตอน **compile time** เลย (เป็น runtime-checked เหมือน `query`/`query_as` แบบไม่มี `!` ที่ Part 70 หัวข้อ 70.4 อธิบายไว้) — นี่คือ**ข้อแลกเปลี่ยนที่หลีกเลี่ยงไม่ได้**ของ SQL แบบ dynamic: ได้ความยืดหยุ่นเรื่องจำนวนเงื่อนไข แต่เสียการตรวจสอบ compile-time ไป ต้องเขียน integration test ที่ครอบคลุมให้ดีกว่าปกติเพื่อชดเชย (เชื่อมกับ Part 32-33 และหัวข้อ 71.10 ของบทนี้)

ผู้เขียนรันจริงสามกรณีทดสอบ (ไม่มี filter เลย, filter category เดียวพร้อม sort desc, filter สองเงื่อนไขพร้อมกัน) ได้ผลลัพธ์จริง:

```
=== page1 (no filter) ===
  id=1 title=Seed Book 0 category=fiction total_count=40
  id=2 title=Seed Book 1 category=programming total_count=40
  id=3 title=Seed Book 2 category=history total_count=40
=== fiction, sort desc, limit 3 ===
  id=37 title=Seed Book 36 category=fiction total_count=10
  id=33 title=Seed Book 32 category=fiction total_count=10
  id=29 title=Seed Book 28 category=fiction total_count=10
=== programming AND available_copies >= 2 ===
  id=2 title=Seed Book 1 available=2 total_count=8
  id=10 title=Seed Book 9 available=5 total_count=8
  id=14 title=Seed Book 13 available=4 total_count=8
  id=18 title=Seed Book 17 available=3 total_count=8
  id=22 title=Seed Book 21 available=2 total_count=8
  id=30 title=Seed Book 29 available=5 total_count=8
  id=34 title=Seed Book 33 available=4 total_count=8
  id=38 title=Seed Book 37 available=3 total_count=8
```

สังเกตว่า `total_count` เปลี่ยนตาม filter ที่ใช้จริง (`40` ตอนไม่กรอง, `10` ตอนกรอง `fiction` เท่านั้น, `8` ตอนกรองทั้ง `programming` และ `available_copies >= 2`) — พิสูจน์ว่า `COUNT(*) OVER()` นับจากผลลัพธ์**หลังกรอง**แล้ว ไม่ใช่นับทั้งตาราง (รายละเอียดกลไกนี้อยู่ในหัวข้อ 71.3) และ `QueryBuilder` ประกอบเงื่อนไขที่ผสมกันได้ถูกต้องในทุกกรณีทดสอบ

ผู้เขียนยังพิมพ์ SQL ที่ `QueryBuilder` ประกอบให้ดูจริงผ่าน `.sql()` (ต้องใช้ `{:?}` เพราะ SQLx 0.9.0 ห่อ SQL string ไว้ใน type `SqlStr` ที่ตั้งใจไม่ implement `Display` ตรง ๆ — เหตุผลเดียวกับ `SqlSafeStr` ที่พูดถึงข้างบน: ป้องกันไม่ให้เอา SQL ที่ผ่านการ audit แล้วไปต่อกับอะไรมือ ๆ ต่ออีกทีโดยไม่ตั้งใจ):

```
generated sql = SqlStr(ArcString("SELECT id FROM books WHERE 1 = 1 AND category = $1 AND available_copies >= $2 ORDER BY id ASC LIMIT $3"))
```

สังเกตว่า placeholder `$1`, `$2`, `$3` ถูกวางให้ถูกตำแหน่งอัตโนมัติตามลำดับที่เรียก `.push_bind()` — ไม่ต้องนับเลขเองแบบที่ต้องทำถ้าเขียน `$1, $2, ...` ด้วยมือ (ข้อดีเสริมของ `QueryBuilder` ที่ไม่ใช่แค่เรื่องความปลอดภัยอย่างเดียว)

### 71.3 Pagination ขั้นสูง: `LIMIT`/`OFFSET` รวมกับ `COUNT(*) OVER()`

#### ปัญหาของ pagination แบบเดิม: ต้อง query สองรอบ

Part 70 หัวข้อ 70.6 สอน `LIMIT`/`OFFSET` พื้นฐานไปแล้ว แต่ยังไม่ได้แก้ปัญหาที่ทุก API ที่มี pagination ต้องเจอ: **client ต้องรู้ว่ามีข้อมูลทั้งหมดกี่แถว** (เพื่อคำนวณจำนวนหน้าทั้งหมด แสดงปุ่ม "หน้าสุดท้าย" ฯลฯ) วิธีตรงไปตรงมาที่สุดคือยิงสอง query แยกกัน:

```rust
// ❌ ไม่ผิด แต่ไม่มีประสิทธิภาพ — สอง round-trip ไปฐานข้อมูลสำหรับงานเดียว
async fn list_books_two_queries(pool: &sqlx::PgPool, limit: i64, offset: i64) -> Result<(Vec<BookRow>, i64), sqlx::Error> {
    let total: i64 = sqlx::query_scalar!("SELECT COUNT(*) FROM books")
        .fetch_one(pool)
        .await?
        .unwrap_or(0);

    let items = sqlx::query_as!(
        BookRowNoCount,
        "SELECT id, title, category, available_copies FROM books ORDER BY id LIMIT $1 OFFSET $2",
        limit,
        offset
    )
    .fetch_all(pool)
    .await?;

    Ok((items, total))
}
```

สอง query แปลว่าสอง round-trip ไปฐานข้อมูล (ผูกกับ Part 70 หัวข้อ 70.3 เรื่องต้นทุนของ network round-trip) — ถ้า endpoint นี้ถูกเรียกหลักพันครั้งต่อวินาที การ query ซ้ำสองเท่าสำหรับงานเดียวกันไม่ใช่เรื่องเล็กเลย

#### `COUNT(*) OVER()`: ได้ทั้งข้อมูลและจำนวนรวมในคำสั่งเดียว

PostgreSQL มี **window function** ที่แก้ปัญหานี้ได้ตรง ๆ: `COUNT(*) OVER()` (ไม่มี `PARTITION BY` ข้างใน) คำนวณ**จำนวนแถวทั้งหมดที่ query จะคืน (หลังกรองด้วย `WHERE` แล้ว แต่ก่อน `LIMIT`)** แล้วแนบตัวเลขนี้**ซ้ำในทุกแถว**ของผลลัพธ์ — เห็นได้แล้วจากตัวอย่างหัวข้อ 71.2 ที่ทุกแถวมี field `total_count` เท่ากันหมด (`40`, `10`, `8` ตามลำดับ ขึ้นกับ filter ที่ใช้)

ความต่างเชิงกลไกจาก `COUNT(*)` ธรรมดา: `COUNT(*)` แบบ aggregate ปกติจะ**ยุบ**ผลลัพธ์ทั้งหมดเหลือแถวเดียว (ต้องมี `GROUP BY` หรือไม่มี column อื่นเลยถ้าไม่มี `GROUP BY`) ในขณะที่ `COUNT(*) OVER()` เป็น **window function** ที่คำนวณ aggregate แล้ว**คงจำนวนแถวเดิมไว้** (ไม่ยุบ) — ทำให้ query เดียวสามารถคืนทั้ง**รายละเอียดของแต่ละแถว**และ**ผลรวม**พร้อมกันได้ นี่คือเหตุผลที่เทคนิคนี้ใช้แก้ปัญหา pagination ได้พอดี: `LIMIT`/`OFFSET` จำกัดจำนวนแถวที่คืนมาจริง (เช่น 3 แถวต่อหน้า) แต่ `total_count` ที่แนบมาในแต่ละแถวยังบอก**จำนวนแถวทั้งหมดก่อน `LIMIT`**อยู่เสมอ — client อ่านค่านี้จากแถวแรก (หรือแถวไหนก็ได้ เพราะเท่ากันหมด) ก็เพียงพอ

**ข้อควรระวังเรื่องประสิทธิภาพ**: `COUNT(*) OVER()` ยัง**ต้องสแกนแถวที่ match เงื่อนไข `WHERE` ทั้งหมด**เพื่อนับ (เหมือน `COUNT(*)` ธรรมดา) เพียงแต่ทำในคำสั่งเดียวกับการดึงข้อมูลหน้าปัจจุบัน ไม่ได้ทำให้ "นับได้เร็วขึ้น" เมื่อเทียบกับ `COUNT(*)` แยก — ประโยชน์หลักคือ**ลด round-trip จากสองครั้งเหลือครั้งเดียว** ไม่ใช่ลด work ที่ PostgreSQL ต้องทำ (สำหรับตารางที่ใหญ่มาก ๆ ระดับสิบล้านแถวขึ้นไป การนับ `COUNT(*)`/`COUNT(*) OVER()` แบบตรง ๆ อาจช้าพอที่ต้องใช้เทคนิคอื่นแทน เช่น approximate count จาก `pg_class.reltuples` — เกินขอบเขตบทนี้ แต่ควรรู้ไว้ว่าเทคนิคนี้ไม่ใช่ทางออกสำหรับทุกขนาดข้อมูล)

**ย้ำเรื่อง `ORDER BY`** (ตามที่ Part 70 หัวข้อ 70.6 เตือนไว้แล้ว): `LIMIT`/`OFFSET` ยังต้องมี `ORDER BY` ที่ชัดเจนและ**deterministic** เสมอ (เช่น sort ตาม `id` ที่ไม่ซ้ำกัน ไม่ใช่ sort ตาม column ที่มีค่าซ้ำได้อย่าง `category` เพียว ๆ ที่ไม่การันตีลำดับภายใน category เดียวกัน) — ไม่อย่างนั้นหน้าที่ 1 กับหน้าที่ 2 อาจมีข้อมูลซ้ำหรือขาดหายได้ ยิ่งสำคัญขึ้นเมื่อรวมกับ `COUNT(*) OVER()` เพราะ error เชิง pagination แบบนี้จะไม่มีอาการอะไรที่ compiler หรือ SQLx ช่วยจับให้ได้เลย เป็น logic bug ล้วน ๆ ที่ต้องอาศัย test ที่ครอบคลุมเท่านั้น

**ข้อจำกัดของ `LIMIT`/`OFFSET` ที่ควรรู้ก่อนใช้กับตารางขนาดใหญ่มาก**: `OFFSET` บอก PostgreSQL ให้ "อ่านแล้วข้าม" แถวที่ถูก skip ทุกแถวก่อนถึงหน้าที่ต้องการจริง — หน้าที่ 1 (`OFFSET 0`) เร็วมากเสมอ แต่หน้าที่ 10,000 (`OFFSET 100000` ถ้า page size คือ 10) ต้องอ่านผ่าน 100,000 แถวก่อน แม้จะไม่คืนแถวเหล่านั้นออกมาก็ตาม ยิ่งหน้าลึกเท่าไรยิ่งช้าลงเป็นเส้นตรง — Part 70 ทิ้งท้ายไว้ว่ามีเทคนิคที่ดีกว่าเรียกว่า **keyset/cursor-based pagination** ในหัวข้อถัดไปจะลงรายละเอียดพร้อมโค้ดจริง

#### Keyset Pagination: ทางเลือกสำหรับตารางขนาดใหญ่มาก

แนวคิดคือใช้ **ค่าของแถวสุดท้ายในหน้าก่อนหน้า** (โดยทั่วไปคือ `id` หรือคอลัมน์ที่ sort อยู่ ไม่ซ้ำกัน) เป็น "คีย์อ้างอิง" (cursor) แทนการนับ offset เป็นตัวเลข — `WHERE id > $last_seen_id LIMIT $n` แทน `OFFSET` ตรง ๆ:

```rust
#[derive(Debug, sqlx::FromRow)]
struct BookKeyset {
    id: i64,
    title: String,
}

// keyset/cursor-based pagination: ใช้ WHERE id > $last_seen_id แทน OFFSET
// ประสิทธิภาพคงที่ไม่ว่าจะอยู่หน้าไหน (ไม่ต้องอ่านข้ามแถวที่ถูก skip เหมือน OFFSET)
async fn list_after_cursor(
    pool: &sqlx::PgPool,
    after_id: i64,
    limit: i64,
) -> Result<Vec<BookKeyset>, sqlx::Error> {
    sqlx::query_as!(
        BookKeyset,
        "SELECT id, title FROM books WHERE id > $1 ORDER BY id ASC LIMIT $2",
        after_id,
        limit
    )
    .fetch_all(pool)
    .await
}
```

ผู้เขียนรันจริงเรียกดูสามหน้าติดกัน โดยแต่ละหน้าใช้ `id` สุดท้ายของหน้าก่อนเป็น cursor สำหรับหน้าถัดไป:

```
page1: [(1, "Seed Book 0"), (2, "Seed Book 1"), (3, "Seed Book 2")]
page2 (after id=3): [(4, "Seed Book 3"), (5, "Seed Book 4"), (6, "Seed Book 5")]
page3 (after id=6): [(7, "Seed Book 6"), (8, "Seed Book 7"), (9, "Seed Book 8")]
```

**เหตุผลที่เร็วกว่า `OFFSET` สำหรับตารางใหญ่**: `WHERE id > $1 ORDER BY id LIMIT $2` ใช้ประโยชน์จาก **index บน `id`** (btree ของ primary key ที่มีอยู่แล้วเสมอ) โดยตรง — PostgreSQL หา "จุดที่ id มากกว่า cursor" ผ่าน index ได้ในเวลาคงที่ (ไม่ขึ้นกับว่าอยู่หน้าที่เท่าไร) แล้วอ่านต่อไปแค่ `limit` แถว จบ ในขณะที่ `OFFSET 100000 LIMIT 10` ต้อง**อ่านผ่าน**ทั้ง 100,000 แถวก่อนจะถึงแถวที่ 100,001 ที่ต้องการจริง

**ข้อแลกเปลี่ยนที่ต้องรู้**: keyset pagination **ไม่รองรับการ "กระโดดไปหน้าที่ N โดยตรง"** ได้ตามธรรมชาติ (เช่น "ไปหน้า 50 เลย" โดยไม่ต้องเปิดหน้า 1-49 ก่อน) เพราะ cursor ต้องมาจากแถวสุดท้ายของหน้าก่อนหน้าเสมอ (ต่างจาก `OFFSET` ที่คำนวณ `offset = page_size * page_number` ได้ตรง ๆ ไม่ว่าจะข้ามไปหน้าไหนก็ตาม) เหมาะกับ UI แบบ "infinite scroll"/"โหลดเพิ่มเติม" ที่ผู้ใช้เลื่อนไปข้างหน้าเรื่อย ๆ มากกว่า UI แบบตัวเลขหน้าที่กระโดดไปมาได้ (pagination ที่มีเลขหน้าให้กดตรง ๆ ยังต้องพึ่ง `OFFSET`/`COUNT(*) OVER()` ตามหัวข้อก่อนหน้าอยู่ดี) — เลือกใช้ตามที่ UX ของระบบต้องการจริง ไม่ใช่เลือกเพราะ "เร็วกว่า" เพียงอย่างเดียว และไม่มี `COUNT(*) OVER()` ที่ใช้คู่กับ keyset pagination ได้ตรง ๆ แบบเดียวกับ `OFFSET` (เพราะไม่มี concept ของ "หน้าที่ N" ที่ต้องรู้จำนวนรวมล่วงหน้า) ถ้าต้องการทั้งจำนวนรวมและ infinite scroll พร้อมกัน มักต้องยิง query แยกสำหรับจำนวนรวม (ยอมรับ round-trip ที่สองเป็นข้อแลกเปลี่ยน)

### 71.4 Batch Operations: Bulk Insert ด้วย `UNNEST` เทียบกับ Insert ทีละแถว

#### ทำไม insert ทีละแถวในลูปถึงช้า

สถานการณ์ที่พบบ่อยมาก: ต้อง import ข้อมูลจากไฟล์ CSV/API ภายนอกเข้าตาราง `books` ครั้งละหลายพันแถว วิธีที่ตรงไปตรงมาที่สุดคือวน loop เรียก `INSERT` ทีละแถว:

```rust
// วิธีที่ตรงไปตรงมา — insert ทีละแถวในลูป
async fn insert_one_by_one(pool: &sqlx::PgPool, books: &[NewBook]) -> Result<(), sqlx::Error> {
    for book in books {
        sqlx::query!(
            "INSERT INTO books (isbn, title, author, total_copies, available_copies, category)
             VALUES ($1, $2, $3, 1, 1, 'bench')",
            book.isbn,
            book.title,
            book.author,
        )
        .execute(pool)
        .await?;
    }
    Ok(())
}
```

โค้ดนี้ **compile ผ่านและทำงานถูกต้อง 100%** — ไม่มี bug เชิง logic เลย แต่มีต้นทุนแฝงเชิงประสิทธิภาพที่ร้ายแรงมาก: **แต่ละรอบของลูปคือ network round-trip แยกกันหนึ่งครั้งไปยัง PostgreSQL** (เชื่อมกับ Part 70 หัวข้อ 70.3 ที่อธิบายต้นทุนของ round-trip ไว้แล้วสำหรับการเปิด connection ใหม่ — หลักการเดียวกันแต่คราวนี้คือต้นทุนของการส่ง**คำสั่ง**แต่ละคำสั่งแยกกัน ไม่ใช่การเปิด connection) ถ้า insert 2,000 แถว จะมี 2,000 round-trip แยกกัน แม้แต่ละ round-trip จะเร็วมากในเชิง latency เดี่ยว ๆ (localhost อาจแค่ 0.3-0.5ms) แต่ผลรวมสะสมของ 2,000 round-trip กลับกลายเป็นเวลาที่มากอย่างมีนัยสำคัญ

#### เทคนิค `UNNEST`: แตกอาร์เรย์เป็นแถว insert ทีเดียว

PostgreSQL มีฟังก์ชัน **`UNNEST()`** ที่แตก **array** ให้กลายเป็นหลายแถว (set-returning function) — เทคนิคที่ใช้กันทั่วไปสำหรับ bulk insert คือส่งอาร์เรย์ของแต่ละคอลัมน์ (array ของ isbn ทั้งหมด, array ของ title ทั้งหมด, ...) ไปพร้อมกันในคำสั่งเดียว แล้วให้ `UNNEST()` "คลี่" อาร์เรย์เหล่านั้นให้จับคู่กันเป็นแถว ๆ ก่อน insert ทีเดียวทั้งหมด:

```rust
async fn insert_unnest_bulk(pool: &sqlx::PgPool, books: &[NewBook]) -> Result<(), sqlx::Error> {
    // แตก Vec<NewBook> ออกเป็นสามอาร์เรย์แยกคอลัมน์ (SQLx bind Vec<T> เป็น PostgreSQL array ให้อัตโนมัติ)
    let isbns: Vec<&str> = books.iter().map(|b| b.isbn.as_str()).collect();
    let titles: Vec<&str> = books.iter().map(|b| b.title.as_str()).collect();
    let authors: Vec<&str> = books.iter().map(|b| b.author.as_str()).collect();

    sqlx::query!(
        r#"
        INSERT INTO books (isbn, title, author, total_copies, available_copies, category)
        SELECT isbn, title, author, 1, 1, 'bench'
        FROM UNNEST($1::text[], $2::text[], $3::text[]) AS t(isbn, title, author)
        "#,
        &isbns as &[&str],
        &titles as &[&str],
        &authors as &[&str],
    )
    .execute(pool)
    .await?;
    Ok(())
}
```

**อธิบายกลไก**: `UNNEST($1::text[], $2::text[], $3::text[])` รับสามอาร์เรย์ที่มีความยาวเท่ากัน แล้ว "คลี่" พร้อมกันทีละตำแหน่ง (ตำแหน่งที่ 0 ของทั้งสามอาร์เรย์กลายเป็นแถวที่ 1, ตำแหน่งที่ 1 กลายเป็นแถวที่ 2, ...) `AS t(isbn, title, author)` ให้ชื่อคอลัมน์กับผลลัพธ์ที่คลี่ออกมา (เหมือนตาราง virtual ชั่วคราว) จากนั้น `SELECT ... FROM UNNEST(...)` ก็แค่เลือกคอลัมน์เหล่านั้นมาพร้อมค่าคงที่ (`1, 1, 'bench'`) แล้ว `INSERT INTO books (...) SELECT ...` insert ผลลัพธ์ทั้งหมดในคำสั่งเดียว — **นี่คือ INSERT เดียว ไม่ว่าจะมีกี่พันแถวก็ตาม** เพราะฉะนั้นมี network round-trip แค่**หนึ่งครั้ง**ไม่ว่าข้อมูลจะมีกี่แถว (ต่างจาก insert ทีละแถวที่ round-trip เพิ่มเป็นเส้นตรงตามจำนวนแถว)

#### วัดผลจริง: ไม่เดา ไม่พูดลอย ๆ (ตามหลักการ Part 54)

ผู้เขียน**วัดจริง**ด้วย `std::time::Instant` เปรียบเทียบทั้งสองวิธีบนข้อมูล 2,000 แถวเท่ากัน บนฐานข้อมูล PostgreSQL ตัวเดียวกัน (localhost — latency ต่อ round-trip ต่ำที่สุดที่เป็นไปได้แล้ว ผลลัพธ์ในสภาพแวดล้อมที่มี network latency สูงกว่านี้ เช่น database ที่อยู่ต่าง region จะยิ่งเห็นความต่างชัดกว่านี้มาก เพราะ round-trip แต่ละครั้งแพงขึ้นตาม latency):

```
insert one-by-one (2000 rows): 813.06ms
insert via UNNEST bulk (2000 rows): 11.58ms
speedup = 70.2x
total bench rows after both inserts = 4000 (should be 4000)
```

**70.2 เท่า** — นี่คือตัวเลขจริงที่วัดได้บนเครื่องเดียวกัน ข้อมูลเดียวกัน ต่างกันแค่วิธี insert แม้จะเป็นแค่ localhost ที่ latency ต่ำมากอยู่แล้ว ความต่างก็ยังมากถึงระดับ 70 เท่า — เหตุผลไม่ใช่ที่ PostgreSQL "insert แต่ละแถวช้า" (การ insert แถวเดียวจริง ๆ เร็วมาก) แต่เป็นที่**overhead ของการส่ง-รับคำสั่งแต่ละคำสั่งผ่าน network** (round-trip latency, การ parse/plan คำสั่งใหม่ทุกครั้งแม้จะมี statement cache ช่วยอยู่บ้างก็ตาม — ดูหัวข้อ 71.8) สะสมคูณด้วยจำนวนแถว ในขณะที่ `UNNEST` ทำให้ PostgreSQL เห็นงานทั้งหมดในคำสั่งเดียว วางแผน query ครั้งเดียว insert ทีเดียวทั้งหมด

**หลักการที่ต้องจำ**: เมื่อต้อง insert ข้อมูลมากกว่าหยิบมือหนึ่ง (มากกว่าสิบ-ยี่สิบแถว) ในครั้งเดียว **ควรใช้เทคนิค bulk insert เสมอ** — `UNNEST` เป็นหนึ่งในเทคนิคที่ SQLx bind `Vec<T>` เป็น PostgreSQL array ให้ได้ตรง ๆ (ไม่ต้อง feature เพิ่มเติมใด ๆ) ทำให้ใช้งานสะดวกจาก Rust ฝั่งเดียว (ทางเลือกอื่นที่ PostgreSQL รองรับคือ `COPY` ที่เร็วกว่า `INSERT` แบบ `UNNEST` อีกสำหรับข้อมูลขนาดใหญ่มาก ๆ ระดับหลักแสน-หลักล้านแถว แต่ SQLx ไม่มี high-level API สำหรับ `COPY` ให้ตรง ๆ ในเวอร์ชันนี้ ต้องเข้าถึงผ่าน `PgConnection` แบบ low-level กว่า — เกินขอบเขตบทนี้ แต่ควรรู้ไว้เป็นตัวเลือกขั้นถัดไปถ้า `UNNEST` ยังไม่พอสำหรับ scale ที่ต้องการ)

#### `WHERE id = ANY($1)`: bulk fetch แทนวน loop ทีละ id

ปัญหาแบบเดียวกันเกิดขึ้นได้กับการ**อ่าน**ข้อมูลด้วย: ถ้ามีลิสต์ของ id ที่ต้องการดึงข้อมูล (เช่น id ของหนังสือที่เกี่ยวข้องกับ `borrow_records` หลายแถว) การวน loop query ทีละ id คือปัญหาเดียวกับหัวข้อ 71.11 (N+1) — วิธีที่ถูกต้องคือส่ง**อาร์เรย์ของ id ทั้งหมด** เข้า query เดียวผ่าน `= ANY($1)`:

```rust
async fn fetch_books_by_ids(pool: &sqlx::PgPool, ids: &[i64]) -> Result<Vec<BookTitle>, sqlx::Error> {
    sqlx::query_as!(
        BookTitle,
        "SELECT id, title FROM books WHERE id = ANY($1)",
        ids
    )
    .fetch_all(pool)
    .await
}
```

ผู้เขียนทดสอบจริงกับ 5 id ที่เพิ่ง insert ไปในตัวอย่าง bulk insert ข้างบน:

```
=== bulk fetch by ids: [41, 42, 43, 44, 45] ===
  id=41 title=Loop Book 0
  id=42 title=Loop Book 1
  id=43 title=Loop Book 2
  id=44 title=Loop Book 3
  id=45 title=Loop Book 4
```

`= ANY($1)` ทำงานคล้าย `IN (...)` แต่รับ**อาร์เรย์ตัวเดียว**เป็น bind parameter (ไม่ต้องสร้าง SQL string ที่มีจำนวน `$1, $2, $3, ...` เปลี่ยนไปตามจำนวน id เหมือนที่ต้องทำถ้าจะใช้ `IN (...)` กับจำนวน id ที่ไม่แน่นอน) — นี่คือข้อดีเชิงปฏิบัติสำคัญ: `IN (...)` ที่ต้องมีจำนวนเงื่อนไขไม่แน่นอนจะกลับไปเจอปัญหาเดียวกับหัวข้อ 71.2 (ต้องสร้าง SQL แบบ dynamic) ในขณะที่ `= ANY($1)` รับ `Vec<i64>` เป็น bind parameter ตัวเดียวได้เลยไม่ว่าจะมีกี่ id ก็ตาม เขียนเป็น SQL literal ที่ตายตัวได้ (ใช้ `query!`/`query_as!` ที่ type-check ตอน compile ได้ตามปกติ ไม่ต้องพึ่ง `QueryBuilder` เลยสำหรับกรณีนี้โดยเฉพาะ)

#### ทางเลือกที่สาม: `QueryBuilder::push_values` เมื่อ SQL ต้อง dynamic อยู่แล้ว

`UNNEST` เหมาะมากเมื่อ SQL เป็น literal ที่ตายตัว (ใช้กับ `query!` ได้ตรง ๆ) แต่ถ้าอยู่ในสถานการณ์ที่ใช้ `QueryBuilder` อยู่แล้ว (เช่นต้องสร้าง SQL แบบ dynamic ด้วยเหตุผลอื่นร่วมด้วย) SQLx มีเมธอด **`push_values()`** ที่ประกอบ `VALUES ($1, $2), ($3, $4), ...` (INSERT หลายแถวแบบ SQL standard) ให้อัตโนมัติจาก iterator:

```rust
use sqlx::postgres::Postgres;
use sqlx::QueryBuilder;

async fn insert_via_push_values(
    pool: &sqlx::PgPool,
    books: &[(String, String, String)], // (isbn, title, author)
) -> Result<(), sqlx::Error> {
    let mut qb: QueryBuilder<Postgres> = QueryBuilder::new(
        "INSERT INTO books (isbn, title, author, total_copies, available_copies, category) ",
    );

    // push_values วนทุก item ใน iterator แล้วสร้าง "($1, $2, $3, 1, 1, 'bench2')" ต่อ ๆ กัน
    // คั่นด้วย comma ให้อัตโนมัติ — ไม่ต้องนับจำนวน $N เองเลย
    qb.push_values(books.iter(), |mut b, (isbn, title, author)| {
        b.push_bind(isbn).push_bind(title).push_bind(author).push(" 1, 1, 'bench2'");
    });

    qb.build().execute(pool).await?;
    Ok(())
}
```

ผู้เขียนวัดจริงเทียบทั้งสามวิธีบนข้อมูลชุดเดียวกัน (2,000 แถว):

```
insert one-by-one (2000 rows): 813.06ms
insert via UNNEST bulk (2000 rows): 11.58ms
insert via QueryBuilder::push_values (2000 rows): 20.76ms
```

`push_values` (20.76ms) ช้ากว่า `UNNEST` (11.58ms) เล็กน้อย (เหตุผลที่เป็นไปได้คือ SQL statement ของ `push_values` มีความยาวมากกว่ามาก — ต้องเขียน placeholder `$1` ถึง `$12000` สำหรับ 2,000 แถว × 6 คอลัมน์ ในขณะที่ `UNNEST` ใช้แค่ 3 placeholder เสมอไม่ว่าจะมีกี่แถว เพราะส่งเป็นอาร์เรย์) แต่ทั้งคู่ยังเร็วกว่าการ insert ทีละแถวอย่างมหาศาล (**39 เท่า** สำหรับ `push_values`, **70 เท่า** สำหรับ `UNNEST`) — **หลักการเลือกใช้**: ถ้า SQL เป็น literal ที่ตายตัวอยู่แล้ว ใช้ `UNNEST` ตรง ๆ กับ `query!` (เร็วกว่าเล็กน้อย และยังได้ compile-time check) ถ้าอยู่ในบริบทที่ต้องใช้ `QueryBuilder` แบบ dynamic อยู่แล้ว (เช่น จำนวนคอลัมน์ที่จะ insert ไม่แน่นอน) `push_values` สะดวกกว่าเพราะไม่ต้องแปลง `Vec<T>` เป็นหลายอาร์เรย์แยกคอลัมน์เองแบบที่ `UNNEST` ต้องทำ

### 71.5 ข้อมูลแบบยืดหยุ่นด้วย `JSONB` และ `sqlx::types::Json<T>`

Part 70 หัวข้อ 70.6 แนะนำ `JSONB` กับ `serde_json::Value` แบบ dynamic ไปแล้วในระดับพื้นฐาน (metadata ที่ไม่รู้ shape ล่วงหน้าเลย) — บทนี้ไปอีกขั้น: สถานการณ์ที่ metadata **มี shape ที่รู้ล่วงหน้า แต่ต่างกันไปตามประเภท** (เช่น metadata ของหนังสือ fiction มี `series`/`volume` แต่ metadata ของหนังสือ programming มี `edition`/`language`) — กรณีนี้ Rust มี type ที่เหมาะสมกว่า `serde_json::Value` แบบ dynamic มาก: **enum ที่ tag ด้วย serde**

#### ออกแบบ `BookMetadata` เป็น enum ที่ type-safe เต็มรูปแบบ

```rust
use serde::{Deserialize, Serialize};

// metadata ของหนังสือแต่ละประเภทมี field ต่างกัน (คล้าย tagged event metadata ที่หลากหลายตาม event type)
// #[serde(tag = "kind")] ทำให้ JSON ที่ได้มี field "kind" บอกว่าเป็น variant ไหน
#[derive(Debug, Serialize, Deserialize, Clone, PartialEq)]
#[serde(tag = "kind", rename_all = "snake_case")]
enum BookMetadata {
    Fiction { series: Option<String>, volume: Option<i32> },
    Programming { edition: i32, language: String },
    Plain,
}

#[derive(Debug, sqlx::FromRow)]
struct TypedBook {
    id: i64,
    title: String,
    // sqlx::types::Json<T> ทำหน้าที่ทั้ง Encode/Decode ให้ T ที่ derive Serialize/Deserialize
    // แปลงเป็น/จาก JSONB โดยตรง ไม่ต้องยุ่งกับ serde_json::Value ดิบ ๆ เลย
    metadata: sqlx::types::Json<BookMetadata>,
}
```

`sqlx::types::Json<T>` คือ wrapper ทั่วไปที่ SQLx ให้มา — เงื่อนไขเดียวคือ `T` ต้อง `Serialize`/`Deserialize` (ตาม Part 57) SQLx จะจัดการ serialize เป็น JSON string ตอน insert และ deserialize กลับเป็น `T` ตอนอ่านให้อัตโนมัติ ผ่าน column ที่เป็น `JSON`/`JSONB` ทั้งสองแบบ — ต่างจากการใช้ `serde_json::Value` ตรง ๆ ตรงที่คุณได้ **type ที่ตรงกับ domain จริง** กลับมา (enum ที่ match ได้, field ที่ IDE autocomplete ได้, compiler เช็ค exhaustiveness ของ `match` ให้) ไม่ใช่ dynamic value ที่ต้อง `.get("field")`/`.as_str()` เดาชนิดเอาเองทุกจุด

#### พิสูจน์ round-trip ด้วยโค้ดจริง

```rust
async fn demo_typed_json(pool: &sqlx::PgPool) -> Result<(), sqlx::Error> {
    let programming_meta = BookMetadata::Programming {
        edition: 3,
        language: "Rust".to_string(),
    };

    let inserted: TypedBook = sqlx::query_as(
        r#"INSERT INTO books (isbn, title, author, total_copies, available_copies, category, metadata)
           VALUES ($1, $2, $3, 1, 1, 'programming', $4)
           RETURNING id, title, metadata"#,
    )
    .bind("978-json-001")
    .bind("Programming Rust, 3rd Edition")
    .bind("Jim Blandy")
    .bind(sqlx::types::Json(programming_meta.clone()))
    .fetch_one(pool)
    .await?;

    assert_eq!(inserted.metadata.0, programming_meta); // round-trip ต้องตรงกันเป๊ะ
    Ok(())
}
```

ผู้เขียนรันจริงทั้ง insert สอง record (แบบ `Fiction` และ `Programming`) แล้วอ่านกลับมา `match` ตาม variant ได้ผลลัพธ์จริง:

```
inserted = TypedBook { id: 4041, title: "Programming Rust, 3rd Edition", metadata: Json(Programming { edition: 3, language: "Rust" }) }
round-trip ผ่าน strongly-typed Json<BookMetadata> ตรงกันเป๊ะ
  [Programming Rust, 3rd Edition] programming edition=3 language=Rust
  [Foundation and Empire] fiction series=Some("Foundation") volume=Some(2)
books ที่ metadata->>'language' = 'Rust': 1
```

`assert_eq!(inserted.metadata.0, programming_meta)` ผ่านจริง — พิสูจน์ว่าข้อมูลที่ insert ไปกับที่อ่านกลับมาเป็น `BookMetadata::Programming { edition: 3, language: "Rust" }` ตัวเดียวกันเป๊ะทุก field ไม่มีการสูญข้อมูลหรือแปลงผิดพลาดระหว่างทาง

#### Query เข้าไปข้างใน JSONB ด้วย operator ของ PostgreSQL

แม้ metadata จะถูก decode เป็น Rust enum ตอนอ่านออกมาแล้ว แต่บางครั้งต้อง**filter จากข้างใน JSONB โดยตรงในระดับ SQL** (ไม่ต้องดึงทุกแถวมาแล้วมา filter ด้วย Rust) PostgreSQL มี operator สำหรับดึงค่าออกจาก JSON/JSONB โดยตรง:

```rust
// ->> ดึง field ออกมาเป็น text ตรง ๆ จาก column JSONB — ใช้ query ระดับ SQL ได้เลยไม่ต้องดึงมา filter ฝั่ง Rust
let rust_books = sqlx::query_as::<_, TypedBook>(
    "SELECT id, title, metadata FROM books WHERE metadata ->> 'language' = $1",
)
.bind("Rust")
.fetch_all(pool)
.await?;
```

ผู้เขียนรันจริงได้ผลลัพธ์ `books ที่ metadata->>'language' = 'Rust': 1` (ตรงตาม 1 แถวที่ insert ไว้เป็น `Programming { language: "Rust", ... }`) — `->>` (double arrow) คืนค่าเป็น **text** ตรง ๆ (เทียบกับ `->` ตัวเดียวที่คืนเป็น JSON value ยังห่อ quote อยู่) เหมาะสำหรับเทียบค่ากับ string ตรง ๆ แบบนี้ **ข้อควรระวัง**: การ filter ผ่าน `->>`  แบบนี้ (โดยไม่มี index รองรับ) ต้องสแกนทุกแถวแล้วแปลง JSONB เป็น text ทีละแถว — ถ้า filter pattern นี้ใช้บ่อยในระบบจริง ควรพิจารณาสร้าง **expression index** (`CREATE INDEX ON books ((metadata ->> 'language'))`) เพื่อให้ query เร็วขึ้นแทนสแกนทั้งตารางทุกครั้ง (แนวคิดเดียวกับ index ปกติ เพียงแต่ index บน "ผลลัพธ์ของ expression" แทน column ตรง ๆ)

#### เร่งความเร็ว query บน JSONB ด้วย GIN Index และ Containment Operator (`@>`)

หัวข้อก่อนหน้าใช้ operator `->>` ที่ดึง field ออกมาเป็น text ก่อนเทียบค่า — วิธีนี้ต้องแปลง JSONB เป็น text **ทุกแถว**ทุกครั้งที่ query (สแกนทั้งตารางเสมอ ไม่มีทาง index ช่วยได้ตรง ๆ) PostgreSQL มี operator ที่เหมาะกับการ query ข้างใน JSONB มากกว่าและ**ใช้ index ได้จริง**: **containment operator `@>`** ที่ตรวจว่า JSONB ฝั่งซ้าย "มี" โครงสร้างย่อยที่ระบุทางขวาอยู่ข้างในหรือไม่ (ไม่ต้องตรงกันทั้งหมด แค่เป็น subset ก็พอ):

```rust
// เพิ่ม GIN index บนคอลัมน์ metadata — ออกแบบมาสำหรับ query ที่ "ค้นภายใน" structure แบบ JSONB โดยเฉพาะ
sqlx::query!("CREATE INDEX IF NOT EXISTS idx_books_metadata_gin ON books USING GIN (metadata)")
    .execute(pool)
    .await?;

// @> ตรวจว่า metadata "มี" {"language": "Rust"} อยู่ข้างในไหม (ไม่ต้อง field อื่นตรงกันทั้งหมด)
let rows = sqlx::query!(
    "SELECT title FROM books WHERE metadata @> $1::jsonb",
    serde_json::json!({"language": "Rust"}),
)
.fetch_all(pool)
.await?;
```

ผู้เขียนรันจริงแล้วตรวจสอบด้วย `EXPLAIN` ว่า query planner ของ PostgreSQL **เลือกใช้ index จริง** (ไม่ใช่แค่ทฤษฎีว่า "ควรจะเร็วขึ้น"):

```
books ที่ metadata @> {"language":"Rust"}: 1
  Rust in Action
PLAN: Bitmap Heap Scan on books  (cost=13.02..17.03 rows=1 width=12)
PLAN:   Recheck Cond: (metadata @> '{"language": "Rust"}'::jsonb)
PLAN:   ->  Bitmap Index Scan on idx_books_metadata_gin  (cost=0.00..13.02 rows=1 width=0)
PLAN:         Index Cond: (metadata @> '{"language": "Rust"}'::jsonb)
```

สังเกตคำว่า **`Bitmap Index Scan on idx_books_metadata_gin`** ใน plan ที่ได้ — ยืนยันว่า PostgreSQL **ไม่ได้สแกนทั้งตาราง** (sequential scan) แต่ใช้ index ที่สร้างไว้ค้นหาโดยตรง (`Recheck Cond` คือขั้นตอนมาตรฐานของ GIN index ที่ตรวจซ้ำผลลัพธ์จาก index อีกครั้งกับข้อมูลจริง เพื่อความถูกต้อง 100% — เป็นกลไกปกติของ bitmap scan ไม่ใช่สัญญาณว่า index ทำงานผิดพลาด) — **GIN** (Generalized Inverted Index) คือชนิด index ที่ PostgreSQL ออกแบบมาสำหรับข้อมูลที่มีโครงสร้าง "ประกอบด้วยหลายส่วนย่อย" (composite values) เช่น array, JSONB, full-text search — ต่างจาก **btree** (index ปกติที่ใช้กับ `category` ในหัวข้อ 71.6) ที่เหมาะกับการเทียบค่าเดี่ยว ๆ ตรง ๆ (`=`, `<`, `>`) เท่านั้น

**หลักการเลือก operator/index สำหรับ JSONB**: ใช้ `->>`  เมื่อต้องดึง field เดียวมาเทียบแบบง่าย ๆ ในตารางขนาดเล็กที่ไม่ต้องพึ่ง index (หรือสร้าง expression index เฉพาะ field นั้นตามที่กล่าวไว้ก่อนหน้า) ใช้ `@>` กับ GIN index เมื่อต้อง query "ค้นภายใน" โครงสร้างที่ซับซ้อนกว่าหรือคาดว่าตารางจะโตขึ้นมากในระยะยาว (GIN index ทำงานได้ดีกับ query ที่ตรวจสอบความมีอยู่ของ key/value ภายใน JSONB โดยไม่ต้องรู้ล่วงหน้าว่า field ไหนจะถูก query บ้าง)

**ย้ำหลักการจาก Part 70**: field ที่รู้อยู่แล้วว่าทุกแถวต้องมีแน่นอน (`title`, `category`, `total_copies`) ควรเป็นคอลัมน์แยกตามปกติเสมอ — `JSONB`/`Json<T>` เหมาะกับ field ที่**ต่างกันไปตามประเภทของข้อมูล**เท่านั้น (เหมือน `BookMetadata` ข้างบนที่ shape ต่างกันจริงตาม category) ไม่ใช่ทางลัดแทนการออกแบบ schema ที่ดี

### 71.6 Migration เชิงลึก: Reversible Migrations และ `_sqlx_migrations`

Part 70 หัวข้อ 70.5 สอนพื้นฐาน `sqlx migrate add -r`/`run`/`revert`/`info` ไปแล้วพร้อมสร้างตาราง `books`/`borrow_records` ตั้งแต่ต้น — หัวข้อนี้ไปลึกกว่าเดิม: **การแก้ schema ของตารางที่มีข้อมูลอยู่แล้ว** ซึ่งเป็นสถานการณ์ที่พบบ่อยกว่าการสร้างตารางใหม่มากในระบบที่ deploy ไปแล้ว

#### สร้าง migration ที่สอง: เพิ่ม `category` และ `metadata` เข้าตารางที่มีข้อมูลอยู่แล้ว

```bash
sqlx migrate add -r add_category_and_metadata
```

```
Creating migrations/20260927002805_add_category_and_metadata.up.sql
Creating migrations/20260927002805_add_category_and_metadata.down.sql
```

`migrations/20260927002805_add_category_and_metadata.up.sql`:

```sql
-- เพิ่ม category (มี default เพื่อไม่ให้แถวเก่าพัง) + metadata แบบ JSONB + index บน category
ALTER TABLE books ADD COLUMN category TEXT NOT NULL DEFAULT 'general';
ALTER TABLE books ADD COLUMN metadata JSONB;
CREATE INDEX idx_books_category ON books (category);
ALTER TABLE books ADD CONSTRAINT books_category_not_empty CHECK (category <> '');
```

`migrations/20260927002805_add_category_and_metadata.down.sql`:

```sql
ALTER TABLE books DROP CONSTRAINT IF EXISTS books_category_not_empty;
DROP INDEX IF EXISTS idx_books_category;
ALTER TABLE books DROP COLUMN IF EXISTS metadata;
ALTER TABLE books DROP COLUMN IF EXISTS category;
```

**อธิบายเหตุผลของแต่ละคำสั่งทีละบรรทัด**:

- **`ADD COLUMN category TEXT NOT NULL DEFAULT 'general'`** — จุดสำคัญที่สุดของ migration นี้ทั้งก้อน: การเพิ่มคอลัมน์ `NOT NULL` เข้าตารางที่**มีข้อมูลอยู่แล้ว**ต้องมี `DEFAULT` เสมอ ไม่อย่างนั้น PostgreSQL จะไม่รู้ว่าจะใส่ค่าอะไรให้แถวเก่าที่มีอยู่แล้วก่อน column นี้ถูกสร้าง (ดูหัวข้อกับดักที่ 3 ที่พิสูจน์ error จริงถ้าลืมใส่ `DEFAULT`) `DEFAULT 'general'` ทำให้แถวเก่าทุกแถวได้ค่านี้ไปโดยอัตโนมัติในตอนที่ `ALTER TABLE` รัน (PostgreSQL เวอร์ชันใหม่ ๆ ตั้งแต่ 11 เป็นต้นมาทำสิ่งนี้แบบ metadata-only ไม่ต้อง rewrite ทั้งตารางถ้า `DEFAULT` เป็นค่าคงที่แบบนี้ — เร็วมากแม้ตารางจะมีข้อมูลหลักล้านแถวก็ตาม)
- **`ADD COLUMN metadata JSONB`** — ไม่มี `NOT NULL` (nullable โดย default) เพราะไม่ใช่ทุกหนังสือที่ต้องมี metadata พิเศษ (ตรงกับที่หัวข้อ 71.5 ออกแบบ `metadata: sqlx::types::Json<BookMetadata>` — ถ้า column เป็น `NOT NULL` field ฝั่ง Rust จะไม่ต้องห่อ `Option` แต่ในกรณีนี้ตั้งใจให้ nullable จึงต้องเป็น `Option<sqlx::types::Json<BookMetadata>>` ถ้าต้องการรองรับแถวที่ไม่มี metadata เลย)
- **`CREATE INDEX idx_books_category ON books (category)`** — เพิ่ม index เพราะหัวข้อ 71.2/71.4 จะ filter ด้วย `category` บ่อยมาก (เป็น query pattern หลักของ endpoint list) — ไม่มี index หมายความว่า PostgreSQL ต้องสแกนทุกแถวทุกครั้งที่ filter ด้วย category (sequential scan) ซึ่งช้าลงเป็นเส้นตรงตามขนาดตาราง มี index (btree ตาม default) ทำให้ query ที่ filter ด้วย `category = '...'` เร็วขึ้นมากสำหรับตารางขนาดใหญ่ (แม้ตารางทดสอบในบทนี้เล็กเกินกว่าจะเห็นผลต่างที่ชัดเจนจาก `EXPLAIN`)
- **`ADD CONSTRAINT books_category_not_empty CHECK (category <> '')`** — บังคับที่ระดับฐานข้อมูลว่า `category` ต้องไม่เป็น string ว่างเปล่า (แม้จะเป็น `NOT NULL` แล้ว ก็ยังอาจเป็น `''` ได้ถ้าไม่มี constraint นี้กัน) — ตัวอย่างของ defense-in-depth เดียวกับที่ Part 70 อธิบายไว้เรื่อง `CHECK` constraint

รันจริง:

```
$ sqlx migrate run
Applied 20260927002805/migrate add category and metadata (3.286246ms)

$ sqlx migrate info
20260927002748/installed create books table
20260927002805/installed add category and metadata
```

ตรวจสอบ schema **ก่อน/หลัง** ด้วย `psql \d books` จริง — ก่อนรัน migration นี้ ตาราง `books` มีแค่ 8 คอลัมน์ตาม Part 70 (`id`, `isbn`, `title`, `author`, `total_copies`, `available_copies`, `published_year`, `created_at`) หลังรันจริง:

```
                                          Table "public.books"
      Column      |           Type           | Collation | Nullable |              Default
------------------+--------------------------+-----------+----------+-----------------------------------
 id               | bigint                   |           | not null | nextval('books_id_seq'::regclass)
 isbn             | text                     |           | not null |
 title            | text                     |           | not null |
 author            | text                     |           | not null |
 total_copies     | integer                  |           | not null |
 available_copies | integer                  |           | not null |
 published_year   | integer                  |           |          |
 created_at       | timestamp with time zone |           | not null | now()
 category         | text                     |           | not null | 'general'::text
 metadata         | jsonb                    |           |          |
Indexes:
    "books_pkey" PRIMARY KEY, btree (id)
    "books_isbn_key" UNIQUE CONSTRAINT, btree (isbn)
    "idx_books_category" btree (category)
Check constraints:
    "books_available_copies_check" CHECK (available_copies >= 0)
    "books_category_not_empty" CHECK (category <> ''::text)
    "books_total_copies_check" CHECK (total_copies >= 0)
Referenced by:
    TABLE "borrow_records" CONSTRAINT "borrow_records_book_id_fkey" FOREIGN KEY (book_id) REFERENCES books(id)
```

เห็นคอลัมน์ `category`/`metadata` ใหม่จริง พร้อม `Default` ของ `category` เป็น `'general'::text` ตามที่ตั้งใจ, index `idx_books_category` และ constraint `books_category_not_empty` ทั้งคู่ถูกสร้างจริงตาม migration ที่เขียนไว้

#### ดู `_sqlx_migrations` จริงว่าหน้าตาเป็นอย่างไร

Part 70 บอกไว้ว่า SQLx สร้างตาราง `_sqlx_migrations` ให้อัตโนมัติเพื่อ track ว่า migration ไหนรันไปแล้ว — บทนี้ query ตารางนี้**ตรง ๆ** ด้วย `psql` เพื่อดูเนื้อหาจริง (ไม่ใช่แค่พูดถึงว่ามันมีอยู่):

```sql
SELECT version, description, installed_on, success, checksum, execution_time
FROM _sqlx_migrations ORDER BY version;
```

ผลลัพธ์จริง:

```
    version     |        description        |         installed_on          | success |                                              checksum                                              | execution_time
----------------+---------------------------+-------------------------------+---------+----------------------------------------------------------------------------------------------------+----------------
 20260927002748 | create books table        | 2026-09-27 00:28:04.840374+00 | t       | \x49f86707b95f34a27bbae405da3268f1419fb8ee544fe984df6b7b1dc1d2ca7779fcb475da53afafda027ff7c94ff5c9 |        6564506
 20260927002805 | add category and metadata | 2026-09-27 00:28:22.009681+00 | t       | \x53a2f5e22e3814e5c21115b050fb6d4186bac224173cbd39f6c16b3c72549ae3f99841eada3ecc2fef0b3d3416d67f12 |        3286246
```

**อธิบายทีละคอลัมน์**:

- **`version`** (`bigint`) — timestamp ที่อยู่ในชื่อไฟล์ migration (`20260927002748` มาจากชื่อไฟล์ `20260927002748_create_books_table.up.sql`) — เป็น primary key ของตารางนี้ การันตีลำดับการรันตามเวลาที่**สร้างไฟล์** (ไม่ใช่เวลาที่รันจริง)
- **`description`** — ชื่อ migration ที่อ่านง่าย (มาจากส่วนหลัง timestamp ในชื่อไฟล์ แปลง `_` เป็น space ให้)
- **`installed_on`** — timestamp ที่ migration นี้ถูก**รันจริง**บนฐานข้อมูลนี้ (ต่างจาก `version` ที่มาจากชื่อไฟล์ — สองเครื่องในทีมอาจรัน migration เดียวกันคนละเวลากัน แต่ `version` จะเหมือนกันเป๊ะเพราะมาจากไฟล์เดียวกัน)
- **`success`** — บอกว่า migration นี้รันสำเร็จหรือไม่ (`t` = true) — ถ้า migration ทำให้เกิด error กลางทาง SQLx จะบันทึก `success = false` ไว้เพื่อให้ทีมรู้ว่ามี migration ที่ค้างอยู่ในสถานะไม่สมบูรณ์ (ต้องแก้ไขด้วยมือ ไม่ใช่ปล่อยให้ `sqlx migrate run` รันซ้ำเองอัตโนมัติ)
- **`checksum`** (`bytea`) — hash ของเนื้อหาไฟล์ migration ตอนที่รัน (ตามที่ Part 70 หัวข้อ 70.5 พิสูจน์ไว้ว่าใช้ตรวจจับการแก้ไฟล์ migration เก่าย้อนหลัง — ถ้าเนื้อหาไฟล์เปลี่ยนไปจาก checksum ที่บันทึกไว้ `sqlx migrate run` จะปฏิเสธทำงาน)
- **`execution_time`** (`bigint`, หน่วยเป็น nanosecond) — เวลาที่ migration นี้ใช้รันจริง (`6564506` ns ≈ 6.56ms สำหรับ migration แรกที่สร้างตาราง, `3286246` ns ≈ 3.29ms สำหรับ migration ที่สอง — ตรงกับตัวเลขที่ `sqlx migrate run` พิมพ์ให้เห็นตอนรันจริงทั้งคู่)

ตารางนี้คือ**แหล่งความจริงเดียว** (single source of truth) ว่าฐานข้อมูลตัวนี้อยู่ที่ "เวอร์ชัน schema" ไหนแล้ว — เครื่องมือ deploy อัตโนมัติ (CI/CD) มักเช็คตารางนี้ก่อนตัดสินใจว่าต้องรัน migration ตัวไหนเพิ่มก่อน deploy เวอร์ชันใหม่ของแอป

#### Migration ที่ปลอดภัยตอน Deploy หลาย Instance พร้อมกัน: แนวคิด Expand-Contract

Part 70 หัวข้อ 70.5 เตือนไว้แล้วว่าห้ามแก้ไฟล์ migration เก่าที่ apply ไปแล้ว — หัวข้อนี้ไปอีกขั้น: สถานการณ์ที่ระบบจริง deploy แอปพร้อมกันหลาย instance (rolling deployment ที่ instance เก่าและใหม่รันพร้อมกันชั่วครู่ระหว่าง deploy) migration ที่ "ดูปลอดภัย" อาจทำให้ instance เวอร์ชันเก่าที่ยังรันอยู่ระหว่าง deploy **พังกลางอากาศ** ได้ ถ้าออกแบบไม่ดี

ลองนึกภาพสถานการณ์ที่ต้อง**เปลี่ยนชื่อคอลัมน์** `author` เป็น `author_name` (เหตุผลสมมติ: ทีมต้องการชื่อที่สื่อความหมายชัดเจนกว่า) วิธีที่ **ผิด** คือทำในคำสั่งเดียว:

```sql
-- ❌ อันตรายมากถ้า deploy แบบ rolling — instance เวอร์ชันเก่าที่ยังรันโค้ดเดิมจะ query "author" ไม่เจอทันที
ALTER TABLE books RENAME COLUMN author TO author_name;
```

ปัญหาคือ: migration รันเสร็จเร็วมาก (metadata operation ล้วน ๆ) แต่การ deploy โค้ดใหม่ของแอปไปยังทุก instance **ใช้เวลานานกว่านั้น** (rolling deployment ทยอย deploy instance ทีละตัว ไม่ใช่ deploy ทุกตัวพร้อมกันในเสี้ยววินาที) — ในช่วงเวลาที่ migration รันเสร็จไปแล้ว แต่ยังมี instance เก่าที่รันโค้ดที่ query `SELECT author FROM books` อยู่ (โค้ดเวอร์ชันเก่าที่ยังไม่ได้ deploy ทับ) instance เหล่านั้นจะเจอ error `column "author" does not exist` ทันที ทั้งที่โค้ดของมันเองไม่ได้เปลี่ยนอะไรเลย

**แนวคิด Expand-Contract** (บางครั้งเรียก "parallel change") แก้ปัญหานี้ด้วยการแบ่งงานเป็นหลาย migration/deploy step ที่**ทับซ้อนกันได้อย่างปลอดภัย**:

1. **Expand**: migration แรกแค่**เพิ่ม**คอลัมน์ใหม่ (`author_name`) โดยไม่ลบของเก่า — โค้ดเวอร์ชันเก่ายัง query `author` ได้ตามปกติ ไม่กระทบอะไรเลย
   ```sql
   ALTER TABLE books ADD COLUMN author_name TEXT;
   UPDATE books SET author_name = author; -- copy ข้อมูลเดิมมาไว้ในคอลัมน์ใหม่
   ```
2. **Migrate (deploy โค้ด)**: deploy โค้ดเวอร์ชันใหม่ที่ **เขียนเข้าทั้งสองคอลัมน์พร้อมกัน** (`author` และ `author_name`) แต่**อ่านจากคอลัมน์ใหม่** — ระหว่างนี้ไม่ว่า instance ไหนจะรันเวอร์ชันเก่าหรือใหม่ ข้อมูลทั้งสองคอลัมน์ยังตรงกันเสมอ (เวอร์ชันเก่าเขียนแค่ `author` แต่มี `UPDATE ... SET author_name = author` เดิมคอยซิงค์อยู่แล้วในบางกรณี หรือใช้ trigger ช่วยซิงค์ระหว่างสองคอลัมน์ชั่วคราวถ้าจำเป็น)
3. **Contract**: หลังจากมั่นใจว่า instance ทุกตัวถูก deploy เป็นเวอร์ชันใหม่หมดแล้ว (ไม่มี instance เก่าเหลืออยู่เลย) ค่อยสร้าง migration ที่**ลบ**คอลัมน์เก่าทิ้ง
   ```sql
   ALTER TABLE books DROP COLUMN author;
   ```

**หลักการที่ต้องจำ**: ทุก migration ที่ **ลบ/เปลี่ยนชื่อ/เปลี่ยน type** ของคอลัมน์ที่โค้ดเวอร์ชันปัจจุบันยังใช้อยู่ ควรถูกมองว่า**อันตรายต่อ rolling deployment เสมอ** จนกว่าจะพิสูจน์ได้ว่าไม่มี instance เก่าเหลืออยู่แล้วจริง ๆ — migration ที่ปลอดภัยที่สุดคือ migration ที่**เพิ่มสิ่งใหม่โดยไม่แตะของเก่า** (`ADD COLUMN` แบบที่หัวข้อนี้ทำกับ `category`/`metadata` ก็เป็นตัวอย่างที่ปลอดภัยอยู่แล้วในความหมายนี้ เพราะไม่กระทบโค้ดเก่าที่ไม่รู้จักคอลัมน์ใหม่เลย มันแค่ไม่เห็นคอลัมน์นั้นเฉย ๆ ไม่ error) ระบบขนาดเล็กที่ deploy แบบหยุดแล้วเริ่มใหม่ทั้งระบบ (ไม่ใช่ rolling) อาจไม่ต้องกังวลเรื่องนี้มากเท่าระบบขนาดใหญ่ที่ downtime เป็นศูนย์เป็นข้อกำหนดสำคัญ — แต่ควรรู้จักแนวคิดนี้ไว้ก่อนที่ระบบจะโตไปถึงจุดที่ rolling deployment กลายเป็นเรื่องจำเป็น

#### เมื่อไรควรใช้ Migration แบบไม่ Reversible (ไม่ใส่ `-r`)

Part 70 หัวข้อ 70.5 แนะนำใส่ `-r` เสมอเพื่อได้ทั้ง `up.sql`/`down.sql` — แต่ในทางปฏิบัติมี migration บางประเภทที่**เขียน `down.sql` ที่ถูกต้องจริง ๆ ไม่ได้เลย** เช่น migration ที่ลบข้อมูลบางแถวออกอย่างถาวร (`DELETE FROM books WHERE category = 'deprecated'`) — ถ้าเขียน `down.sql` เป็น `-- ไม่สามารถย้อนกลับได้` เฉย ๆ ก็ไม่ต่างจากไม่มี `down.sql` เลยในทางปฏิบัติ (แค่หลอกตัวเองว่ามี rollback path) กรณีแบบนี้การสร้าง migration แบบไม่ reversible ตรง ๆ (ไม่ใส่ `-r` ได้ไฟล์เดียวคือ `<timestamp>_<name>.sql`) สื่อความหมายตรงกว่า: **"migration นี้ไม่มีทางย้อนกลับอัตโนมัติได้ ถ้าต้อง rollback ต้อง restore จาก backup เท่านั้น"** ทีมที่เห็นไฟล์แบบนี้ในโค้ดจะรู้ทันทีว่าต้องระวังเป็นพิเศษก่อน deploy (เช่น backup ฐานข้อมูลก่อนรัน migration นี้เสมอ) แทนที่จะเข้าใจผิดว่ามี safety net จาก `down.sql` ที่ใช้งานไม่ได้จริงรออยู่

#### การ Squash Migration เมื่อมีจำนวนมากเกินไป

โปรเจกต์ที่พัฒนามานานหลายปีอาจสะสม migration ไฟล์หลักร้อยไฟล์ — การรัน migration ทั้งหมดตั้งแต่ไฟล์แรกทุกครั้งที่ตั้งฐานข้อมูลใหม่ (เช่น environment สำหรับ test/CI ที่สร้างขึ้นใหม่บ่อย ๆ) ใช้เวลานานขึ้นเรื่อย ๆ ตามจำนวนไฟล์ที่สะสม แนวทางที่ทีมใหญ่ใช้กันคือ **squash migration**: รวม migration เก่าจำนวนมากที่ apply ไปแล้วในทุก environment ที่สำคัญ (ไม่มี environment ไหนเหลือค้างที่ยังไม่ได้ apply ไฟล์เก่าเหล่านั้นแล้ว) ให้เหลือเป็นไฟล์เดียวที่สร้าง schema สุดท้ายตรง ๆ (เหมือน `pg_dump --schema-only` ของ schema ปัจจุบัน) แล้ว**ลบไฟล์เก่าที่ถูก squash ไปทั้งหมด** — SQLx เองไม่มีคำสั่ง squash อัตโนมัติให้ (ต่างจากบางเครื่องมือ migration ของภาษาอื่น) ต้องทำด้วยมือ: `pg_dump --schema-only` ฐานข้อมูลที่มี schema ล่าสุด แล้ววาง SQL ที่ได้ในไฟล์ migration ใหม่ไฟล์เดียว จากนั้นต้อง**อัปเดตตาราง `_sqlx_migrations` เองด้วยมือ**ในทุกฐานข้อมูลที่มีอยู่แล้ว (insert แถวที่บอกว่า migration ใหม่ตัวนี้ "ถูก apply แล้ว" พร้อม checksum ที่ตรงกับไฟล์ใหม่) เพื่อไม่ให้ `sqlx migrate run` พยายามรันไฟล์ใหม่ตัวนี้ทับฐานข้อมูลที่มี schema นี้อยู่แล้ว — เป็นกระบวนการที่ต้องระวังมาก ควรทำเฉพาะเมื่อจำนวนไฟล์ migration เริ่มเป็นปัญหาจริงจังต่อความเร็วในการตั้ง environment ใหม่เท่านั้น ไม่ใช่ทำเป็นประจำ

### 71.7 Connection Pool Tuning เชิงลึก: ผูกกับ Tokio Task Concurrency

Part 70 หัวข้อ 70.3/70.11 สอน `PgPoolOptions` พื้นฐานและพิสูจน์พฤติกรรม pool exhaustion (รอจนกว่าจะมี connection ว่างหรือ timeout) ไปแล้ว — หัวข้อนี้ตอบคำถามที่ทีมจริงต้องเจอบ่อยที่สุด: **"ตั้ง `max_connections` เท่าไรดี และทำไม API ของฉันถึงเริ่ม timeout พร้อมกันหมดตอน traffic สูงขึ้น?"**

#### ทวนตารางอ้างอิงจาก Part 70 พร้อมมุมมองเชิง tuning เพิ่ม

| Option | มุมมองเชิง tuning ที่ควรรู้เพิ่ม |
|---|---|
| `.max_connections(n)` | นี่คือ**ขีดจำกัดจริง**ของจำนวน task ที่คุยกับฐานข้อมูลพร้อมกันได้ (ไม่ใช่จำนวน request ที่รับได้พร้อมกัน — request ที่ไม่ได้คุยกับฐานข้อมูลตอนนั้นไม่นับ) ตั้งสูงเกินไปโดยไม่จำเป็นสิ้นเปลือง memory ทั้งฝั่งแอปและฝั่ง PostgreSQL (แต่ละ connection กิน memory ฝั่ง PostgreSQL หลัก MB ต่อ connection) ตั้งต่ำเกินไปทำให้ throughput รวมของระบบถูกจำกัดโดยไม่จำเป็นทั้งที่ CPU/network ยังเหลือ
| `.min_connections(n)` | ช่วยลด **cold-start latency** ของ request แรก ๆ หลังโปรแกรม idle นาน (ไม่ต้องรอเปิด connection ใหม่) แต่ไม่ช่วยเรื่อง throughput สูงสุด (นั่นคือหน้าที่ของ `max_connections`)
| `.acquire_timeout(d)` | ควรตั้งให้**สั้นกว่า** timeout ของ client/load balancer ที่เรียกเข้ามาเสมอ (ไม่มีประโยชน์ที่จะรอนานกว่าที่ client จะรอไหว — client จะ timeout ไปก่อนอยู่ดี ทำให้ resource ที่ pool "จอง" ไว้เพื่อรอ request นั้นสูญเปล่า)
| `.idle_timeout(d)` | สำคัญมากสำหรับระบบที่ traffic ขึ้นลงเป็นช่วง ๆ (เช่น มี peak ตอนกลางวัน เงียบตอนกลางคืน) — ปิด connection ที่ไม่ได้ใช้ทิ้งช่วยลด load ฝั่ง PostgreSQL ตอน traffic ต่ำ โดยที่ pool จะเปิดใหม่ให้เองอัตโนมัติเมื่อ traffic กลับมาสูงอีก
| `.max_lifetime(d)` | ป้องกันปัญหาที่ connection อายุยืนเกินไปอาจสะสมปัญหา (เช่น load balancer/firewall ตัดการเชื่อมต่อที่ค้างนานเกินไปแบบเงียบ ๆ โดยที่ทั้งสองฝั่งไม่รู้ตัว — connection ดู "ยังเปิดอยู่" ฝั่งแอป แต่จริง ๆ ใช้ไม่ได้แล้ว) การบังคับปิด/เปิดใหม่เป็นระยะช่วยลดความเสี่ยงนี้

#### วัดจริง: `max_connections` เล็กเกินไปทำให้ concurrent task ต้องรอเรียงกัน

มาพิสูจน์ผลของ `max_connections` ต่อ concurrent task ด้วยโค้ดจริง — จำลอง "งานที่กิน connection นานกว่าปกติ" ด้วย `pg_sleep(0.1)` (คิวรีที่ใช้เวลา 0.1 วินาทีต่อครั้ง เหมือน query ที่ซับซ้อนหรือฐานข้อมูลที่ตอบช้าผิดปกติในสถานการณ์จริง) แล้ว spawn 12 task พร้อมกันยิง query นี้ เทียบระหว่าง pool ที่ `max_connections = 2` กับ `max_connections = 12`:

```rust
async fn slow_task(pool: sqlx::PgPool, id: u32) {
    sqlx::query!("SELECT pg_sleep(0.1)").execute(&pool).await.unwrap();
}

async fn run_with_max_connections(max_conn: u32, n_tasks: u32) -> std::time::Duration {
    let pool = sqlx::postgres::PgPoolOptions::new()
        .max_connections(max_conn)
        .acquire_timeout(std::time::Duration::from_secs(10))
        .connect("postgres://postgres:postgres@127.0.0.1:5432/part71_scratch")
        .await
        .unwrap();

    let start = std::time::Instant::now();
    let mut handles = Vec::new();
    for i in 0..n_tasks {
        handles.push(tokio::spawn(slow_task(pool.clone(), i)));
    }
    for h in handles {
        h.await.unwrap();
    }
    start.elapsed()
}
```

ผู้เขียนรันจริงกับ 12 task พร้อมกัน (จำลอง 12 request ที่มาถึงเซิร์ฟเวอร์ในเวลาไล่เลี่ยกัน) ได้ผลลัพธ์:

```
=== max_connections=2, 12 concurrent tasks (แต่ละ task ใช้เวลา ~0.1s) ===
  task 0 เสร็จหลัง 102.04ms
  task 1 เสร็จหลัง 107.48ms
  task 4 เสร็จหลัง 203.88ms
  task 5 เสร็จหลัง 209.05ms
  task 6 เสร็จหลัง 306.24ms
  task 9 เสร็จหลัง 310.45ms
  task 7 เสร็จหลัง 407.77ms
  task 10 เสร็จหลัง 411.82ms
  task 8 เสร็จหลัง 509.28ms
  task 11 เสร็จหลัง 513.29ms
  task 2 เสร็จหลัง 610.93ms
  task 3 เสร็จหลัง 614.69ms
รวมเวลาทั้งหมด (pool เล็ก): 614.90ms

=== max_connections=12, 12 concurrent tasks ===
  task 0 เสร็จหลัง 102.04ms
  task 3 เสร็จหลัง 109.28ms
  ... (ทุก task เสร็จในช่วง 102-115ms ใกล้เคียงกันหมด)
รวมเวลาทั้งหมด (pool ใหญ่พอสำหรับทุก task): 115.24ms
```

ผลลัพธ์ตรงกับที่คาดเป๊ะ: pool ที่ `max_connections = 2` ทำให้ 12 task ต้อง**ผลัดกันใช้ 2 connection เป็นกลุ่ม ๆ** (สังเกตว่า task เสร็จเป็นคู่ ๆ ทุก ~100ms — `102ms, 107ms` แล้ว `203ms, 209ms` แล้ว `306ms, 310ms` ไปเรื่อย ๆ จนครบ 6 รอบสำหรับ 12 task) รวมเวลาทั้งหมด **614.90ms** ในขณะที่ pool ที่ `max_connections = 12` (พอสำหรับทุก task พร้อมกัน) ทำให้ทุก task รันพร้อมกันได้หมด เสร็จเกือบพร้อมกันที่ **115.24ms** — เร็วกว่าประมาณ **5.3 เท่า**

**ผลเชิงปฏิบัติที่ตอบคำถามตั้งต้น**: "ทำไม API timeout พร้อมกันหมดตอน traffic สูง" คือปรากฏการณ์เดียวกับที่พิสูจน์ข้างบนนี้พอดี — ถ้า `max_connections` ตั้งไว้เล็กเกินไปเทียบกับจำนวน concurrent request ที่ต้องคุยกับฐานข้อมูลจริง (ผูกกับ Part 48 เรื่องจำนวน Tokio task ที่ทำงานพร้อมกันได้) request ที่มาทีหลังจะต้อง**รอเข้าคิว**นานขึ้นเรื่อย ๆ ตามจำนวน task ที่เข้ามาก่อนหน้า (เห็นได้จากรูปแบบ "รอเป็นกลุ่ม ๆ" ข้างบน) ถ้าคิวยาวพอ เวลารอของ request ท้าย ๆ จะเกิน `acquire_timeout` (หรือ timeout ของ client เอง) ทำให้เกิด error พร้อมกันเป็นชุดใหญ่ ทั้งที่จริง ๆ ฐานข้อมูลเองไม่ได้ "ช้า" อะไรเลย (แต่ละ query ยังเร็วเท่าเดิม) ปัญหาอยู่ที่**คิวที่ยาวเกินไปเพราะ pool เล็กเกินไปเทียบกับ concurrency ที่ต้องรองรับจริง**

**ข้อควรระวัง — ไม่ควรตั้ง `max_connections` สูงเกินจำเป็นเช่นกัน**: ถ้าตั้งสูงมาก ๆ (เช่น 500) โดยไม่คำนึงถึง `max_connections` ของ PostgreSQL เอง (ค่า default ทั่วไปคือ 100) หรือจำนวน instance ของแอปที่รันพร้อมกันหลาย ๆ ตัว (แต่ละ instance มี pool ของตัวเอง — ถ้ามี 10 instance ตั้ง `max_connections = 50` ต่อตัว รวมกันคือ 500 connection ไปยัง PostgreSQL ตัวเดียว) อาจทำให้ PostgreSQL เองปฏิเสธ connection ใหม่เพราะเกิน `max_connections` ของฐานข้อมูล — ตัวเลขที่เหมาะสมต้องคำนวณจาก**ทั้งระบบ** ไม่ใช่แค่มองจากแอปตัวเดียว (เชื่อมกับแนวคิด capacity planning ที่ Part 54/55 พูดถึงเรื่องการวัด/วางแผนประสิทธิภาพในภาพรวม ไม่ใช่มองแค่ชิ้นเดียวของระบบ)

#### วินิจฉัยจริง: "ทำไม API ของฉันเริ่ม timeout พร้อมกันหมดตอน traffic สูง"

สถานการณ์นี้คือหนึ่งใน incident ที่พบบ่อยที่สุดของทีมที่ deploy เว็บแอปที่มี database เป็น dependency — มาลองเดินตามกระบวนการวินิจฉัยแบบที่ Part 54/55 สอนไว้ (วัดจริง ไม่เดา) ทีละขั้น แทนการ "เพิ่ม `max_connections` มั่ว ๆ แล้วหวังว่าจะดีขึ้น" ซึ่งอาจแก้ปัญหาไม่ตรงจุดหรือแย่ลงกว่าเดิม (ถ้าสาเหตุจริงคือ query ที่ช้าผิดปกติ การเพิ่ม `max_connections` แค่ทำให้ปัญหาเดิมเกิดพร้อมกันได้มากขึ้น ไม่ได้แก้ต้นเหตุ):

1. **เช็คจำนวน connection ที่ pool ใช้อยู่จริง ณ ขณะนั้น** — `PgPool` มีเมธอด `pool.size()` (จำนวน connection ทั้งหมดที่ pool เปิดอยู่ ทั้งที่ idle และถูกยืมอยู่) และ `pool.num_idle()` (จำนวนที่**ว่าง**อยู่ ไม่มีใครยืม) ถ้า `pool.size() == max_connections` และ `pool.num_idle() == 0` ต่อเนื่องเป็นเวลานาน นั่นคือสัญญาณชัดเจนว่า pool "เต็มค้าง" จริง ไม่ใช่แค่ peak ชั่วครู่
2. **แยกแยะว่า pool เต็มเพราะ traffic สูงขึ้นจริง หรือเพราะ query แต่ละตัวช้าลง** — ถ้าจำนวน request ต่อวินาทีไม่ได้เพิ่มขึ้นมากแต่ pool ยังเต็ม แปลว่า query แต่ละตัวใช้เวลานานขึ้น (ทำให้ connection ถูกยึดไว้นานขึ้นต่อ request หนึ่งครั้ง) ต้องไปดู slow query log ของ PostgreSQL ต่อ (ปัญหาอาจเป็น N+1 ตามหัวข้อ 71.11, ขาด index, หรือ lock contention จาก transaction ที่ยาวเกินไป) ไม่ใช่ปัญหาที่ `max_connections` แก้ได้
3. **ถ้า traffic สูงขึ้นจริงและ query เร็วเท่าเดิม** — นี่คือกรณีที่ `max_connections` เล็กเกินไปเทียบกับ concurrency ที่ต้องรองรับจริงตามที่พิสูจน์ไว้ข้างบน (614.90ms เทียบ 115.24ms) การเพิ่ม `max_connections` (พร้อมตรวจสอบว่า PostgreSQL และ instance อื่นยังรับได้ตามข้อควรระวังข้างบน) คือทางแก้ที่ตรงจุด

ผู้เขียนสาธิตการอ่านค่า `pool.size()`/`pool.num_idle()` จริงเพื่อใช้ในขั้นตอนที่ 1 ข้างบน — ยึด connection ทีละตัวแล้วดูค่าเปลี่ยนไปอย่างไร:

```rust
let pool = /* ... */;
println!("ตอนเริ่ม: size={} num_idle={}", pool.size(), pool.num_idle());

let c1 = pool.acquire().await?;
let c2 = pool.acquire().await?;
let c3 = pool.acquire().await?;
println!("ยึด 3 connection แล้ว: size={} num_idle={}", pool.size(), pool.num_idle());

drop(c1);
drop(c2);
// (ให้เวลา pool คืน connection กลับเข้า idle pool เล็กน้อย)
println!("คืน 2 connection แล้ว: size={} num_idle={}", pool.size(), pool.num_idle());
```

ผลลัพธ์จริง:

```
ตอนเริ่ม: size=1 num_idle=1
ยึด 3 connection แล้ว: size=3 num_idle=0
คืน 2 connection แล้ว: size=3 num_idle=2
คืนครบทุกตัว: size=3 num_idle=3
```

สังเกตว่า `size` **ไม่ลดลง**เมื่อคืน connection (ยังคงเป็น `3` ตลอด — connection ที่เปิดแล้วจะถูกเก็บไว้ใน pool รอใช้ซ้ำ ไม่ปิดทิ้งทันทีที่คืน ตามหลักการของ pool ที่ Part 70 อธิบายไว้) ในขณะที่ `num_idle` เปลี่ยนตามจำนวน connection ที่**ว่าง ณ ขณะนั้น**จริง — สอง metric นี้คือจุดเริ่มต้นที่ดีที่สุดสำหรับ observability ของ pool ในระบบจริง: expose ค่าทั้งสองผ่าน endpoint `/metrics` (เช่นในรูปแบบ Prometheus) แล้วตั้ง alert เมื่อ `num_idle` เท่ากับ `0` ต่อเนื่องนานเกินเกณฑ์ที่ยอมรับได้ — จะรู้ปัญหา pool exhaustion **ก่อน**ที่ client จะเริ่ม timeout จริง ไม่ต้องรอให้ user ร้องเรียนก่อนแล้วมาสืบย้อนหลัง

#### สูตรประมาณค่า `max_connections` ที่เหมาะสม: Little's Law

คำถาม "ควรตั้ง `max_connections` เท่าไร" มีสูตรประมาณเชิงคณิตศาสตร์ที่ช่วยให้ไม่ต้องเดามั่ว ๆ — ทฤษฎีแถวคอย (queueing theory) มีกฎพื้นฐานที่เรียกว่า **Little's Law**: `L = λ × W` โดย `L` คือจำนวนงานเฉลี่ยที่อยู่ในระบบพร้อมกัน (ในที่นี้คือจำนวน connection ที่ต้องใช้พร้อมกันโดยเฉลี่ย) `λ` (แลมบ์ดา) คือ throughput หรืออัตราการมาถึงของงาน (จำนวน query ต่อวินาที) และ `W` คือเวลาเฉลี่ยที่งานหนึ่งชิ้นอยู่ในระบบ (เวลาเฉลี่ยที่ query หนึ่งครั้งใช้ ตั้งแต่ acquire connection จนถึงคืนกลับ pool)

ลองใช้ตัวเลขจริงจากหัวข้อ 71.11: query แบบ `JOIN` เดียวที่วัดได้ **1.10ms** ต่อครั้ง ถ้าระบบต้องรองรับ **1,000 query ต่อวินาที** (`λ = 1000`, `W = 0.0011` วินาที):

```
L = λ × W = 1000 × 0.0011 = 1.1
```

แปลว่าโดยเฉลี่ยต้องมี connection ที่ถูกใช้งานพร้อมกันจริง ๆ แค่ **~1.1 ตัว** ณ เวลาใดเวลาหนึ่ง — `max_connections = 10` ก็เหลือเฟือมากสำหรับ throughput ระดับนี้ (มี margin รองรับ traffic spike ได้สบาย) แต่ถ้า query เดียวกันช้าลงเป็น **50ms** ต่อครั้ง (เช่นเกิด N+1 ตามหัวข้อ 71.11 ที่ทำให้ query แต่ละครั้งใช้เวลานานขึ้น หรือ query ที่ขาด index) ที่ throughput เท่าเดิม:

```
L = 1000 × 0.050 = 50
```

ตอนนี้ต้องมี connection พร้อมกันเฉลี่ย **~50 ตัว** — ถ้า `max_connections` ยังตั้งไว้ที่ 10 เหมือนเดิม (ตามค่าที่คำนวณไว้ตอน query ยังเร็ว) pool จะเข้าสู่สภาวะ**เต็มค้างถาวร**ทันที (`L` ที่ต้องการมากกว่า `max_connections` ที่มีถึง 5 เท่า) เกิดอาการ timeout พร้อมกันตามที่อธิบายไว้ข้างบนพอดี — **นี่คือสิ่งที่ Little's Law เผยให้เห็นชัดเจน**: ปัญหา pool exhaustion มักไม่ได้มาจาก "traffic สูงขึ้นกะทันหัน" อย่างเดียว แต่มาจาก **`W` (เวลาต่อ query) เพิ่มขึ้น** ควบคู่ไปด้วยบ่อยครั้งกว่าที่คิด (ตรงกับขั้นตอนวินิจฉัยข้อ 2 ที่อธิบายไว้ข้างบน) — สูตรนี้ไม่ได้ให้ตัวเลขที่แม่นยำ 100% สำหรับ production จริง (ยังไม่รวมความแปรปรวนของ traffic แบบ burst, distribution ของเวลา query ที่ไม่ใช่ค่าคงที่เดียว ฯลฯ) แต่เพียงพอสำหรับเป็น**จุดเริ่มต้นประมาณค่า**ที่ดีกว่าการเดาล้วน ๆ มาก และช่วยอธิบายเหตุผลว่าทำไม "traffic เท่าเดิม แต่ query ช้าลงนิดเดียว" ถึงทำให้ pool ที่เคยพอเพียงกลายเป็นไม่พอได้ทันที

### 71.8 Prepared Statement Caching: ลด Overhead ของการ `PREPARE` ซ้ำ

#### กลไก statement cache ของ SQLx

ทุกครั้งที่ PostgreSQL รัน query หนึ่งคำสั่ง มันต้อง **parse** (แปลง SQL text เป็น syntax tree) และ **plan** (ตัดสินใจว่าจะอ่านข้อมูลด้วยวิธีไหน ใช้ index ตัวไหน) ก่อนจะ execute จริง — ถ้า query **เดิม**ถูกเรียกซ้ำ ๆ หลายครั้ง (query pattern เดียวกัน ต่างกันแค่ค่าที่ bind) การ parse/plan ใหม่ทุกครั้งเป็นงานที่**ซ้ำซ้อนไม่จำเป็น** PostgreSQL รองรับ **prepared statement** (`PREPARE`) ที่ทำ parse/plan ครั้งเดียว แล้วเรียก `EXECUTE` ซ้ำได้หลายครั้งโดยข้ามขั้นตอน parse/plan ไปเลย

SQLx ใช้ประโยชน์จากกลไกนี้โดยอัตโนมัติผ่าน **connection-level statement cache**: แต่ละ connection ใน pool เก็บ **LRU cache** (Least Recently Used — ตัวที่ไม่ถูกใช้นานที่สุดถูกลบออกก่อนเมื่อ cache เต็ม) ของ prepared statement ที่เคยส่งไปแล้ว ครั้งแรกที่ query pattern หนึ่งถูกเรียกผ่าน connection ตัวนั้น SQLx จะส่ง `PREPARE` ให้ PostgreSQL จริง (มี overhead เพิ่มขึ้นเล็กน้อยจากการ parse/plan) ครั้งต่อ ๆ ไปที่ query pattern**เดียวกัน**ถูกเรียกผ่าน connection ตัวเดิม SQLx จะ**ข้าม**การ `PREPARE` ไปเลย ส่งแค่ `EXECUTE` พร้อมค่าที่ bind ใหม่

ตั้งค่าความจุของ cache นี้ผ่าน `PgConnectOptions::statement_cache_capacity()` (ค่า default คือ **100** — เก็บได้ 100 query pattern ที่ต่างกันต่อ connection หนึ่งตัว):

```rust
use sqlx::postgres::{PgConnectOptions, PgPoolOptions};
use std::str::FromStr;

let connect_options = PgConnectOptions::from_str(
    "postgres://postgres:postgres@127.0.0.1:5432/part71_scratch",
)?
.statement_cache_capacity(50); // ลดจาก default 100 ถ้า query pattern ที่ใช้จริงมีน้อย ประหยัด memory

let pool = PgPoolOptions::new()
    .max_connections(8)
    .connect_with(connect_options) // .connect_with() รับ PgConnectOptions แทน .connect() ที่รับ &str ตรง ๆ
    .await?;
```

**ข้อสังเกตเรื่อง layer**: `statement_cache_capacity()` เป็น method ของ `PgConnectOptions` (คุมพฤติกรรมของ**connection แต่ละตัว**) ไม่ใช่ของ `PgPoolOptions` (ที่คุมพฤติกรรมของ**pool โดยรวม** อย่าง `max_connections`) — ต้องสร้าง `PgConnectOptions` แยกก่อน ตั้งค่าที่ต้องการ แล้วส่งให้ `PgPoolOptions::connect_with()` แทนการเรียก `.connect(url)` ที่รับ URL string ตรง ๆ (ที่ใช้ `PgConnectOptions` ค่า default ทั้งหมด)

ผู้เขียนทดสอบจริงด้วยการรัน query เดิม 500 รอบผ่าน pool เดียวกัน:

```
pool connected, size=2
รอบแรก (เตรียม statement ใหม่): 1.51ms
รวม 500 รอบ: 83.07ms, เฉลี่ยรอบละ 166.14µs
```

บน localhost ที่ network latency ต่ำมาก ความต่างระหว่างรอบแรก (ต้อง `PREPARE`) กับรอบเฉลี่ย (ใช้ cache) ไม่ได้ต่างกันมากอย่างมีนัยสำคัญ เพราะ round-trip latency เองก็ต่ำมากอยู่แล้ว (เหตุผลเดียวกับที่หัวข้อ 71.4 อธิบายไว้ว่า network latency คือต้นทุนหลักของ round-trip) — **แต่ในสภาพแวดล้อมจริงที่ database อยู่ไกลกว่านี้** (เช่น cloud database ต่าง availability zone หรือต่าง region) ความต่างของ statement cache จะชัดเจนกว่านี้มาก เพราะการข้าม round-trip ของ `PREPARE` (แม้จะรวมอยู่ใน round-trip เดียวกับ query แรกก็ตามในบางกรณี) และการข้ามขั้นตอน parse/plan ฝั่ง PostgreSQL เอง มีผลสะสมชัดเจนกว่าเมื่อ query ถูกเรียกซ้ำ ๆ นับพัน-หมื่นครั้งต่อวัน

**หลักปฏิบัติ**: ค่า default 100 เพียงพอสำหรับแอปส่วนใหญ่ที่มี query pattern ไม่มากเกินไป (ไม่ได้ generate SQL ที่ต่างกันแบบไม่มีที่สิ้นสุด) — ควรพิจารณาปรับ**เพิ่ม**ถ้าแอปมี query pattern ที่หลากหลายมากจริง ๆ (หลาย endpoint ที่แต่ละตัวมี query เฉพาะของตัวเองจำนวนมาก) และควรพิจารณาปรับ**ลด**เฉพาะกรณีที่ memory ต่อ connection เป็นข้อจำกัดจริงจัง (แต่ละ prepared statement ที่ cache ไว้กิน memory เพิ่มขึ้นเล็กน้อยฝั่ง PostgreSQL) — สำหรับแอปทั่วไปแทบไม่ต้องยุ่งกับค่านี้เลย ค่า default ออกแบบมาให้เหมาะกับกรณีส่วนใหญ่อยู่แล้ว

### 71.9 Listening for Notifications: `LISTEN`/`NOTIFY` ผ่าน `PgListener`

#### กลไกของ `LISTEN`/`NOTIFY` ใน PostgreSQL

PostgreSQL มีกลไก **pub/sub แบบง่าย ๆ ในตัว** ที่ไม่ต้องพึ่ง extension หรือ message queue ภายนอกเลย: คำสั่ง **`LISTEN channel_name`** บอกให้ connection นั้น "สมัครรับฟัง" ข้อความจาก channel ที่ระบุ ส่วนคำสั่ง **`NOTIFY channel_name, 'payload'`** (หรือฟังก์ชัน `pg_notify(channel_name, payload)`) ส่งข้อความไปยังทุก connection ที่ `LISTEN` channel นั้นอยู่ **ทันที** (แบบ real-time ไม่ต้อง poll) — นี่คือกลไกที่มีมานานแล้วใน PostgreSQL ใช้ประโยชน์ได้ดีสำหรับสถานการณ์ที่ต้องการแจ้งเตือน "มีอะไรเปลี่ยนแปลง" แบบง่าย ๆ โดยไม่ต้องตั้ง message queue เต็มรูปแบบ

SQLx เปิดให้ใช้กลไกนี้ผ่าน **`sqlx::postgres::PgListener`** — struct ที่จัดการ connection แยกสำหรับ `LISTEN` โดยเฉพาะ (ไม่ยืม connection จาก pool ปกติ เพราะ connection ที่ `LISTEN` ต้องเปิดค้างไว้รอรับข้อความตลอดเวลา ผิดธรรมชาติของ connection ใน pool ที่ควรถูกยืม-คืนเร็ว ๆ):

```rust
use sqlx::postgres::PgListener;

async fn demo_listen_notify(pool: &sqlx::PgPool) -> Result<(), sqlx::Error> {
    // PgListener::connect_with(&pool) เปิด connection ใหม่แยกจาก pool (ยืม config การเชื่อมต่อจาก pool มาใช้)
    let mut listener = PgListener::connect_with(pool).await?;
    listener.listen("book_events").await?;
    println!("listening on channel 'book_events'...");

    // จำลอง producer: ส่ง NOTIFY จาก task อื่น (อาจเป็นอีก process/instance ของแอปก็ได้ในโลกจริง)
    let notify_pool = pool.clone();
    let producer = tokio::spawn(async move {
        tokio::time::sleep(std::time::Duration::from_millis(200)).await;
        sqlx::query!(
            "SELECT pg_notify('book_events', $1)",
            r#"{"event":"book_created","isbn":"978-notify-1"}"#
        )
        .execute(&notify_pool)
        .await
        .unwrap();
    });

    // consumer: รอรับ notification ตัวแรกที่มาถึง
    let notification = listener.recv().await?;
    println!(
        "ได้รับ notification จาก channel='{}' payload={}",
        notification.channel(),
        notification.payload()
    );

    producer.await.unwrap();
    Ok(())
}
```

ผู้เขียนรันจริง (มี timeout กันค้างด้วย `tokio::time::timeout` ในเวอร์ชันเต็มที่ทดสอบ) ได้ผลลัพธ์จริงแบบ end-to-end:

```
listening on channel 'book_events'...
producer: ส่ง NOTIFY แล้ว
ได้รับ notification จาก channel='book_events' payload={"event":"book_created","isbn":"978-notify-1"}
```

`listener.recv().await` block (แบบ async — ไม่กีดขวาง task อื่นเลยตามหลักการ Part 46-48) จนกว่าจะมี notification มาถึงจริง ๆ payload ที่ได้กลับมาคือ string เดียวกันเป๊ะกับที่ `pg_notify()` ส่งไป (ในตัวอย่างนี้ตั้งใจส่งเป็น JSON string เพื่อให้ consumer parse ต่อได้ด้วย `serde_json::from_str` ถ้าต้องการโครงสร้างข้อมูลที่ซับซ้อนกว่า string เปล่า)

#### ขอบเขตและข้อจำกัดที่ต้องรู้ก่อนใช้จริง

`LISTEN`/`NOTIFY` เหมาะสำหรับสถานการณ์ **"แจ้งเตือนแบบเบา ๆ"** เท่านั้น — เช่น บอกให้ instance อื่นของแอป (ที่แชร์ฐานข้อมูลเดียวกัน) รู้ว่า "cache ควร invalidate แล้ว" หรือแจ้งให้ WebSocket connection ที่เปิดอยู่รู้ว่า "มีข้อมูลใหม่ให้ push ไปยัง client" **ข้อจำกัดสำคัญที่ต้องรู้ก่อนใช้จริง**:

- **ไม่มีการันตีการส่งถึง (no delivery guarantee)** — ถ้า listener ไม่ได้ต่ออยู่ตอนที่ `NOTIFY` ถูกส่ง (เช่น instance นั้น restart อยู่พอดี) notification นั้น**หายไปเลย** ไม่มีทาง replay กลับมาดูใหม่ได้ (ต่างจาก message queue จริงที่มักมี persistent storage รอให้ consumer มา pull เมื่อพร้อม)
- **ไม่มี message queue/buffer ฝั่งเซิร์ฟเวอร์** — payload ถูกส่งแบบ fire-and-forget ทันที ไม่มีการเก็บสะสมรอ ไม่มี retry, ไม่มี dead-letter queue, ไม่มี ordering guarantee ข้าม channel
- **ขนาด payload จำกัด** — PostgreSQL จำกัดขนาด payload ของ `NOTIFY` ไว้ที่ 8000 byte (ค่านี้ผูกกับ `NAMEDATALEN`/การตั้งค่าภายในของ PostgreSQL) ไม่เหมาะกับข้อมูลขนาดใหญ่
- **connection ของ `PgListener` แยกจาก pool** — ไม่ถูกนับรวมกับ `max_connections` ของ `PgPoolOptions` แต่ยังคงเป็น connection จริงหนึ่งตัวที่กิน resource ฝั่ง PostgreSQL เหมือน connection ปกติทุกประการ ต้องบริหารจัดการจำนวน listener ที่เปิดไว้อย่างมีสติเช่นกัน (ไม่ใช่ "ฟรี" เพียงเพราะไม่ได้มาจาก pool)

ด้วยข้อจำกัดเหล่านี้ `LISTEN`/`NOTIFY` **ไม่ใช่ตัวแทนของ message queue เต็มรูปแบบ** (เช่น RabbitMQ, Kafka, หรือ SQS ที่ Part 82 จะสอนในหลักสูตรต่อไป) ที่ต้องมี delivery guarantee, persistent buffer, retry mechanism, และ ordering ที่เข้มงวดกว่านี้มาก — ควรมองมันเป็น**ทางเลือกที่เบากว่ามาก**สำหรับกรณีง่าย ๆ ที่ยอมรับได้ว่าข้อความอาจหายได้บ้างเป็นบางครั้ง (best-effort) โดยไม่ต้องเพิ่ม infrastructure ใหม่เข้าระบบ (ไม่ต้องตั้ง message broker แยก ใช้ PostgreSQL ที่มีอยู่แล้วได้ทันที) — เมื่อความต้องการซับซ้อนขึ้น (ต้องการันตีการส่งถึง, ต้องรองรับ throughput สูงมาก, ต้องมี consumer group หลายตัวแบ่งงานกัน) ควรย้ายไปใช้ message queue จริงตามที่ Part 82 จะสอน

#### ตัวอย่างประยุกต์: ส่ง Notification ต่อไปยัง Client ผ่าน SSE (ระดับแนวคิด)

สถานการณ์ที่ใช้ `PgListener` ได้ประโยชน์จริงในระบบห้องสมุด/ตั๋วคือการแจ้ง client ที่เปิดหน้าเว็บทิ้งไว้ (ผ่าน Server-Sent Events หรือ WebSocket) ว่า "มีการยืมหนังสือเกิดขึ้นใหม่" แบบ real-time โดยไม่ต้องให้ client poll ถามซ้ำ ๆ — โครงร่างแนวคิด (ไม่ลงรายละเอียด SSE เต็มรูปแบบ เพราะเกินขอบเขตบทนี้ แต่ควรเห็นภาพว่าประกอบกันอย่างไร):

```rust
// แนวคิดคร่าว ๆ: task พื้นหลังหนึ่งตัวคอย listen แล้ว broadcast ต่อให้ทุก SSE connection ที่เปิดอยู่
async fn notification_forwarder(pool: sqlx::PgPool, tx: tokio::sync::broadcast::Sender<String>) {
    let mut listener = PgListener::connect_with(&pool).await.expect("connect listener");
    listener.listen("borrow_events").await.expect("listen");

    loop {
        match listener.recv().await {
            Ok(notification) => {
                // ส่งต่อ payload ให้ทุกคนที่ subscribe ผ่าน broadcast channel (Part 39/48 เรื่อง channel)
                let _ = tx.send(notification.payload().to_string());
            }
            Err(e) => {
                eprintln!("listener error: {e}, กำลังพยายามใหม่...");
                // ในโค้ดจริงควรมี retry/reconnect logic ที่นี่ — connection ของ listener อาจหลุดได้
            }
        }
    }
}
```

`tokio::sync::broadcast::Sender` (ตาม Part 39/48 ที่สอนเรื่อง channel สำหรับสื่อสารข้าม task) รับ payload จาก `PgListener` แล้วส่งต่อให้ทุก SSE handler ที่ subscribe ไว้พร้อมกัน — แต่ละ handler เพียงแค่ `tx.subscribe()` แล้ว loop ส่งข้อมูลที่ได้กลับไปยัง client ของตัวเองผ่าน HTTP response แบบ streaming — สถาปัตยกรรมนี้ทำให้ **connection ของ `PgListener` มีแค่ตัวเดียวทั้งระบบ** (ไม่ว่าจะมี client เชื่อมต่ออยู่กี่คนก็ตาม) เพราะ broadcast ทำหน้าที่กระจายต่อในระดับ in-memory ของ Rust process เอง ไม่ต้องเปิด `PgListener` ใหม่ต่อ client แต่ละคน (ซึ่งจะเข้าเงื่อนไขกับดักที่ 6 ท้ายบทถ้าทำแบบนั้น — connection รั่วไหลสะสมที่ไม่ถูกนับใน `max_connections`)

### 71.10 Testing กับ Transaction: Pattern "เปิด Transaction แล้วไม่ Commit"

#### ทวนจาก Part 70: `#[sqlx::test]` ทำอะไร และมีข้อจำกัดอะไร

Part 70 หัวข้อ 70.13 สอน `#[sqlx::test]` ไปแล้ว — attribute macro ที่สร้าง**ฐานข้อมูลใหม่ทั้งลูก**ให้ทุก test function หนึ่งตัว (clone schema จาก migration, รัน migration ให้เสร็จ, ลบทิ้งอัตโนมัติหลัง test จบ) วิธีนี้ให้ isolation ที่แน่นอนที่สุด (แต่ละ test เห็นฐานข้อมูลที่ไม่มีใครแตะเลยจริง ๆ) แต่มีต้นทุน: การ `CREATE DATABASE`/รัน migration ทุกครั้งมีค่าใช้จ่ายที่ไม่เล็กเมื่อ test suite มีจำนวนมาก (หลักร้อย-หลักพัน test function) — สำหรับทีมที่ test suite ใหญ่มาก เวลารวมของการสร้าง/ลบฐานข้อมูลซ้ำ ๆ อาจกลายเป็นคอขวดของ CI pipeline เอง

#### Pattern ทางเลือก: transaction ต่อ test ที่ไม่เคย commit

แนวคิดคือ: **แชร์ฐานข้อมูล/pool ตัวเดียวกันข้าม test** แต่ทุกคำสั่งในแต่ละ test อยู่ **ใน transaction ที่ไม่เคย commit** — จาก Part 70 หัวข้อ 70.9 คุณรู้แล้วว่า `Transaction` ที่ถูก `drop` โดยไม่เรียก `.commit()` จะ **rollback อัตโนมัติ** (ผ่าน `Drop` implementation) — pattern นี้ใช้ประโยชน์จากพฤติกรรมนี้ตรง ๆ: เปิด transaction ตอนเริ่ม test ทำอะไรก็ได้ข้างใน (insert, update, delete) แล้ว**ปล่อยให้ transaction ถูก drop เองตอนจบ function** (ไม่ต้องเรียก `.rollback()` เองด้วยซ้ำ — การไม่ commit ก็เพียงพอแล้ว) ข้อมูลที่ทำไปทั้งหมดจะไม่มีผลกับฐานข้อมูลจริงเลย ไม่กระทบ test อื่นที่ใช้ pool เดียวกัน

```rust
use sqlx::PgPool;

async fn shared_pool() -> PgPool {
    sqlx::postgres::PgPoolOptions::new()
        .max_connections(5)
        .connect("postgres://postgres:postgres@127.0.0.1:5432/part71_scratch")
        .await
        .expect("connect failed")
}

#[tokio::test]
async fn insert_inside_uncommitted_tx_is_isolated() {
    let pool = shared_pool().await;
    let mut tx = pool.begin().await.unwrap();

    sqlx::query!(
        "INSERT INTO books (isbn, title, author, total_copies, available_copies, category)
         VALUES ($1, $2, $3, 1, 1, 'tx-test')",
        "978-txtest-001",
        "Transaction Isolated Book",
        "Ghost Author",
    )
    .execute(&mut *tx)
    .await
    .unwrap();

    // เห็นแถวนี้ได้จาก "ภายใน" tx เดียวกัน (connection เดียวกัน มองเห็นงานที่ยังไม่ commit ของตัวเอง)
    let count_inside: i64 = sqlx::query_scalar!(
        "SELECT COUNT(*) FROM books WHERE isbn = $1", "978-txtest-001"
    )
    .fetch_one(&mut *tx)
    .await
    .unwrap()
    .unwrap();
    assert_eq!(count_inside, 1);

    // ไม่เรียก tx.commit() เลย — ปล่อยให้ tx ถูก drop ตอนจบฟังก์ชัน => ROLLBACK อัตโนมัติ
    drop(tx);

    // เปิด connection ใหม่จาก pool มาเช็คว่าแถวนี้ "ไม่มีอยู่จริง" ในฐานข้อมูล
    let count_outside: i64 = sqlx::query_scalar!(
        "SELECT COUNT(*) FROM books WHERE isbn = $1", "978-txtest-001"
    )
    .fetch_one(&pool)
    .await
    .unwrap()
    .unwrap();
    assert_eq!(count_outside, 0, "แถวต้องไม่หลุดออกมาหลัง rollback");
}

#[tokio::test]
async fn second_test_sees_clean_state() {
    // test นี้รันพร้อมกับตัวข้างบนได้ (cargo test รัน test แบบ parallel โดย default)
    // ถ้า test แรก commit จริงจะเห็น isbn 978-txtest-001 หลุดมาปนที่นี่ — แต่ไม่เห็นเพราะ rollback ไปแล้ว
    let pool = shared_pool().await;
    let count: i64 = sqlx::query_scalar!(
        "SELECT COUNT(*) FROM books WHERE isbn = $1", "978-txtest-001"
    )
    .fetch_one(&pool)
    .await
    .unwrap()
    .unwrap();
    assert_eq!(count, 0);
}
```

ผู้เขียนรันจริงด้วย `cargo test` (ทั้งสอง test รันพร้อมกันตาม default ของ Rust test runner — เชื่อมกับ Part 32-33 เรื่อง test ที่ควร independent กันได้แม้รัน parallel):

```
running 2 tests
test second_test_sees_clean_state ... ok
test insert_inside_uncommitted_tx_is_isolated ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.09s
```

ทั้งสอง test ผ่านจริง — `count_inside == 1` (เห็นแถวที่ insert ไปจาก**ภายใน** transaction เดียวกัน เพราะ PostgreSQL การันตีว่า connection มองเห็นงานที่ตัวเองยังไม่ commit เสมอ) แต่ `count_outside == 0` (แถวนั้น**ไม่มีอยู่จริง**เมื่อมองจากมุมของ connection อื่น เพราะไม่เคย commit) และ test ที่สองที่ query ผ่าน pool ตัวเดียวกันก็ยืนยันว่า `978-txtest-001` ไม่เคยหลุดออกมาสู่ฐานข้อมูลจริงเลย — สังเกตว่า test ทั้งสองใช้ **เวลารวมแค่ 0.09s** (เทียบกับ `#[sqlx::test]` ที่ต้อง `CREATE DATABASE`/รัน migration จริงทุกครั้งตามที่ Part 70 หัวข้อ 70.13 วัดไว้ที่ 0.87s สำหรับสองตัวเช่นกัน) — เร็วกว่าอย่างเห็นได้ชัดสำหรับ test suite ขนาดเล็ก และความต่างจะยิ่งเห็นชัดขึ้นเมื่อจำนวน test มากขึ้นเรื่อย ๆ

**ข้อสังเกตเรื่องเวอร์ชัน**: ผู้เขียนตรวจสอบ `sqlx-macros-core` เวอร์ชัน 0.9.0 ที่ใช้ในบทนี้ (อ่าน source code จริงของ `test_attr.rs`) พบว่า `#[sqlx::test]` **ไม่มี option สำหรับสลับไปใช้ transaction-per-test แบบอัตโนมัติ** ในเวอร์ชันนี้ — มีแค่ option ปรับ migration path/fixtures เท่านั้น การใช้ pattern "transaction ที่ไม่ commit" ต้องเขียนด้วยมือแบบข้างบน (เปิด `pool.begin()` เองในทุก test) ไม่ใช่ attribute ที่มีให้ใช้ตรง ๆ

**เลือกใช้แนวทางไหน**: `#[sqlx::test]` (Part 70) เหมาะเป็นค่า default เพราะ setup น้อยที่สุดและ isolation แน่นอนที่สุด (ไม่มีทางที่ test หนึ่งจะเห็นข้อมูลของอีก test ได้เลย เพราะเป็นฐานข้อมูลคนละตัวกันจริง ๆ) — pattern "transaction ไม่ commit" ในหัวข้อนี้เหมาะสำหรับทีมที่ test suite ใหญ่มากจนความเร็วรวมของการรัน test กลายเป็นปัญหาจริงจัง และยอมรับความซับซ้อนที่เพิ่มขึ้นเล็กน้อย (ต้องเขียน `pool.begin()`/ส่ง `&mut *tx` ให้ทุก query ในทุก test เอง ตามกับดักข้อ 2 ของ Part 70 ที่เตือนไว้เรื่องการสลับ `&pool`/`&mut *tx` ผิดที่) เพื่อแลกกับความเร็วที่มากกว่า

#### ลดโค้ดซ้ำ: Helper Function สำหรับเปิด Transaction ทดสอบ

การเขียน `pool.begin().await.unwrap()` ซ้ำทุก test function เป็น boilerพลेตที่รำคาญถ้ามี test จำนวนมาก — helper function ง่าย ๆ ช่วยลดความซ้ำซ้อนนี้ได้ (ยังไม่ถึงขั้นต้องเขียน custom attribute macro ของตัวเอง เพียงแค่ฟังก์ชันช่วยธรรมดา):

```rust
use sqlx::{PgConnection, PgPool, Postgres, Transaction};

/// เปิด pool (ใช้ pool เดียวกันได้ทุก test เพราะ transaction ที่ไม่ commit ไม่ทิ้งผลกระทบ)
/// แล้วเปิด transaction ให้พร้อมใช้ในบรรทัดเดียว — ไม่ต้อง unwrap() ซ้ำทุกที่
async fn begin_test_tx() -> Transaction<'static, Postgres> {
    static POOL: tokio::sync::OnceCell<PgPool> = tokio::sync::OnceCell::const_new();
    let pool = POOL
        .get_or_init(|| async {
            PgPoolOptions::new()
                .max_connections(5)
                .connect("postgres://postgres:postgres@127.0.0.1:5432/part71_scratch")
                .await
                .expect("connect failed")
        })
        .await;
    pool.begin().await.expect("begin tx failed")
}

#[tokio::test]
async fn example_using_helper() {
    let mut tx: Transaction<'_, Postgres> = begin_test_tx().await;
    sqlx::query!(
        "INSERT INTO books (isbn, title, author, total_copies, available_copies, category)
         VALUES ($1, $2, $3, 1, 1, 'tx-test')",
        "978-txhelper-001", "Helper Test Book", "Author",
    )
    .execute(&mut *tx)
    .await
    .unwrap();
    // ไม่ commit — tx ถูก drop ตอนจบ scope ก็ rollback ให้อัตโนมัติ
}
```

**อธิบาย**: `tokio::sync::OnceCell` (ตาม Part 39/48 เรื่องการแชร์ state ข้าม task อย่างปลอดภัย) เปิด pool เพียง**ครั้งเดียว**ตลอดทั้ง test binary (ไม่ว่าจะมี test function กี่ตัวเรียก `begin_test_tx()` ก็ตาม) แล้ว pool ตัวนั้นถูกใช้ซ้ำข้าม test — เพราะแต่ละ test ทำงานใน transaction ของตัวเองที่ไม่เคย commit การแชร์ pool เดียวกันจึงไม่ทำให้ test ชนกันเลย (ตรงกันข้ามกับการแชร์ pool แบบเดียวกันถ้าไม่มี transaction คั่นไว้ ซึ่งจะชนกันแน่นอน) — pattern นี้ทำให้แต่ละ test function สั้นลงมาก เหลือแค่ `let mut tx = begin_test_tx().await;` บรรทัดแรก แล้วเขียน logic ของ test ต่อได้เลยโดยไม่ต้องยุ่งกับ boilerplate การเปิด pool ซ้ำทุกครั้ง

### 71.11 ปัญหา N+1 Query: พิสูจน์ด้วยตัวเลขจริง

#### โค้ดที่ "ดูปกติ" แต่ซ่อนปัญหาประสิทธิภาพร้ายแรง

ลองนึกภาพ endpoint ที่ list ประวัติการยืมหนังสือ (`borrow_records`) พร้อมชื่อหนังสือของแต่ละ record — วิธีเขียนที่ "ดูสมเหตุสมผล" ที่สุดสำหรับคนที่เพิ่งเรียน SQLx คือ query `borrow_records` มาก่อน แล้ววน loop ไป query หาชื่อหนังสือของแต่ละแถวทีละตัว (เพราะดูเหมือนเป็นขั้นตอนที่ตรงไปตรงมา "หา record ก่อน แล้วหาชื่อหนังสือของมัน"):

```rust
// ❌ N+1: query หา borrow_records ก่อน 1 ครั้ง แล้ววน loop ไป query หนังสือทีละเล่มต่อแถว
async fn list_naive(pool: &sqlx::PgPool) -> Result<Vec<(String, String)>, sqlx::Error> {
    let records = sqlx::query!("SELECT book_id, borrower_name FROM borrow_records ORDER BY id")
        .fetch_all(pool)
        .await?; // query ที่ 1

    let mut result = Vec::with_capacity(records.len());
    for r in records {
        // ❌ นี่คือ query แยกทีละแถว — ถ้ามี borrow_records 200 แถว จะยิง query นี้ 200 ครั้ง
        let title = sqlx::query_scalar!("SELECT title FROM books WHERE id = $1", r.book_id)
            .fetch_one(pool)
            .await?;
        result.push((r.borrower_name, title));
    }
    Ok(result)
}
```

โค้ดนี้ **compile ผ่านและทำงานถูกต้อง 100%** (ผลลัพธ์ที่ได้ไม่ผิดเลย) — นี่คือเหตุผลที่ปัญหานี้ชื่อ **"N+1 query problem"** เป็นหนึ่งในปัญหาประสิทธิภาพที่พบบ่อยและถูกมองข้ามมากที่สุดในโค้ดที่ใช้ ORM/query library ทุกตัว ไม่จำกัดแค่ SQLx หรือแค่ Rust: **N** คือจำนวนแถวของ query แรก, **+1** คือ query แรกที่ใช้หา list ของ id มา — รวมแล้วมี `N + 1` query ทั้งที่งานนี้**ควรทำในคำสั่งเดียว**

#### ทางแก้ที่ถูกต้อง: `JOIN` ครั้งเดียว

```rust
// ✅ วิธีที่ถูก: JOIN ครั้งเดียว ได้ข้อมูลทั้งหมดในคำสั่ง SQL เดียว — ไม่ว่าจะมีกี่แถวก็ตาม
async fn list_join(pool: &sqlx::PgPool) -> Result<Vec<(String, String)>, sqlx::Error> {
    let rows = sqlx::query!(
        r#"SELECT br.borrower_name, b.title AS book_title
           FROM borrow_records br
           JOIN books b ON b.id = br.book_id
           ORDER BY br.id"#
    )
    .fetch_all(pool)
    .await?;

    Ok(rows.into_iter().map(|r| (r.borrower_name, r.book_title)).collect())
}
```

`JOIN` ให้ PostgreSQL ทำงาน "จับคู่แถวที่เกี่ยวข้องกัน" ในระดับฐานข้อมูลโดยตรง (ซึ่ง PostgreSQL ถูกออกแบบมาให้ทำงานนี้อย่างมีประสิทธิภาพผ่าน index บน foreign key อยู่แล้ว) แทนที่จะให้ฝั่ง Rust ทำหน้าที่ "จับคู่" เองด้วยการ query แยกทีละแถว — **ไม่ว่า `borrow_records` จะมีกี่แถวก็ตาม จำนวน query ที่ต้องยิงคือ `1` เสมอ** (คงที่ ไม่ผูกกับขนาดข้อมูล)

#### วัดผลจริง: จำนวน query และเวลาต่างกันแค่ไหน

ผู้เขียนวัดจริงบนข้อมูล `borrow_records` 200 แถว (แต่ละแถวอ้างถึงหนังสือใน 40 เล่มที่ seed ไว้):

```
naive (N+1): 200 แถว, 201 queries, ใช้เวลา 27.67ms
join (single query): 200 แถว, 1 query, ใช้เวลา 1.10ms
batch ANY(): 2 queries คงที่ (ไม่ว่า borrow_records จะมีกี่แถว)
speedup ของ JOIN เทียบกับ N+1 = 25.2x
query count ลดจาก 201 เหลือ 1 (201x น้อยลง)
```

**201 queries เหลือ 1 query** — เร็วขึ้น **25.2 เท่า** แม้จะทดสอบบน localhost ที่ latency ต่อ query ต่ำที่สุดแล้ว (เชื่อมกับหัวข้อ 71.4 อีกครั้ง: บนระบบจริงที่ database มี network latency สูงกว่านี้ ตัวเลขนี้จะยิ่งต่างกันมากกว่า 25 เท่าไปอีก เพราะทุก round-trip แพงขึ้นตาม latency แต่ N+1 มี round-trip มากกว่าเป็นเส้นตรงตามจำนวนแถว) — และตัวเลขที่สำคัญไม่แพ้เวลาคือ**จำนวน query**: `201` เทียบกับ `1` ตัวเลขนี้จะแย่ลงไปอีกเป็นเส้นตรงตามข้อมูลที่มากขึ้น (ถ้า `borrow_records` มี 100,000 แถว โค้ดแบบ N+1 จะยิง **100,001 queries** ในขณะที่ `JOIN` ยังคงเป็น **1 query เสมอ**) — นี่คือเหตุผลที่ N+1 เป็นปัญหาที่ "ไม่โผล่ให้เห็นตอน dev บนข้อมูลทดสอบน้อย ๆ" แต่ **ระเบิดจริงตอน production ที่มีข้อมูลจริงมาก ๆ**

#### ทางเลือกที่สาม: เมื่อต้อง fetch สองรอบแยกกันจริง ๆ

บางสถานการณ์ไม่สามารถ `JOIN` ตรง ๆ ได้ (เช่น ข้อมูลที่สองมาจาก microservice คนละตัวที่ต้องเรียกผ่าน HTTP API แยก ไม่ใช่ตารางในฐานข้อมูลเดียวกัน) — กรณีนี้ยังหลีกเลี่ยง N+1 ได้ด้วยการ**รวบรวม id ทั้งหมดก่อน แล้วยิง query/request ที่สองแค่ครั้งเดียว** (ตามที่หัวข้อ 71.4 แนะนำ `= ANY($1)`):

```rust
async fn list_batch_any(pool: &sqlx::PgPool) -> Result<(), sqlx::Error> {
    let records = sqlx::query!("SELECT DISTINCT book_id FROM borrow_records")
        .fetch_all(pool)
        .await?; // query ที่ 1: หา distinct ids ทั้งหมด

    let ids: Vec<i64> = records.iter().map(|r| r.book_id).collect();

    // query ที่ 2 (ครั้งเดียว): ดึงหนังสือทุกเล่มที่เกี่ยวข้องพร้อมกันทีเดียว แทนวน loop ทีละ id
    let _titles = sqlx::query!("SELECT id, title FROM books WHERE id = ANY($1)", &ids)
        .fetch_all(pool)
        .await?;

    Ok(()) // รวม 2 queries คงที่ ไม่ว่า borrow_records จะมีกี่แถว
}
```

ผู้เขียนทดสอบจริงได้ `batch ANY(): 2 queries คงที่` — สอง query นี้ไม่ขึ้นกับจำนวนแถวของ `borrow_records` เลย (ต่างจาก N+1 ที่ query ที่สองคูณตามจำนวนแถว) แม้จะไม่ดีเท่า `JOIN` เดียวตรง ๆ (ที่เหลือแค่ 1 query) แต่ก็ยังดีกว่า N+1 อย่างมหาศาล และเป็นทางเลือกที่ใช้ได้ในสถานการณ์ที่ `JOIN` ทำไม่ได้จริง ๆ

**หลักการที่ต้องจำและตรวจสอบทุกครั้งที่เห็น loop ที่มี `.await` ข้างใน**: ทุกครั้งที่เห็นโค้ดที่มี `for`/`while` loop ที่ข้างในมีการเรียก query (`.fetch_one()`, `.fetch_all()`, `.execute()`) **ให้สงสัยไว้ก่อนว่าอาจเป็น N+1** — ถามตัวเองว่า "งานนี้รวมเป็น query เดียวได้ไหมด้วย `JOIN`" ก่อนเสมอ ถ้าทำไม่ได้จริง ๆ ให้ถามต่อว่า "รวบรวมเงื่อนไขทั้งหมดแล้วยิงเป็น batch เดียวด้วย `= ANY($1)` ได้ไหม" — ทั้งสองทางเลือกนี้ควรเป็นค่า default ในหัวเสมอเมื่อเจอ pattern "query ในลูป" ไม่ใช่สิ่งที่นึกถึงทีหลังตอน performance มีปัญหาแล้ว

#### N+1 แบบ Aggregation: รูปแบบที่แนบเนียนกว่าเดิม

N+1 ไม่ได้เกิดแค่กับการ "หารายละเอียด" ของแต่ละแถวเท่านั้น — อีกรูปแบบที่พบบ่อยไม่แพ้กันคือการ **นับ/สรุปข้อมูลที่เกี่ยวข้องทีละแถว** เช่น "หนังสือแต่ละเล่มถูกยืมไปแล้วกี่ครั้ง" ถ้าเขียนแบบวน loop นับทีละเล่ม:

```rust
// ❌ N+1 แบบ aggregation: หา book ทั้งหมดก่อน แล้ววน loop นับ borrow_records ของแต่ละเล่มทีละครั้ง
async fn count_naive(pool: &sqlx::PgPool) -> Result<Vec<(i64, i64)>, sqlx::Error> {
    let books = sqlx::query!("SELECT id FROM books ORDER BY id").fetch_all(pool).await?;
    let mut results = Vec::with_capacity(books.len());
    for b in books {
        let count: i64 = sqlx::query_scalar!(
            "SELECT COUNT(*) FROM borrow_records WHERE book_id = $1", b.id
        )
        .fetch_one(pool)
        .await?
        .unwrap_or(0);
        results.push((b.id, count));
    }
    Ok(results)
}
```

ทางแก้คือ `GROUP BY` ครั้งเดียว (ใช้ `LEFT JOIN` เพื่อให้หนังสือที่**ยังไม่เคยถูกยืมเลย**ยังปรากฏในผลลัพธ์ด้วย `count = 0` ไม่ใช่ถูกตัดออกไปแบบที่ `JOIN`/`INNER JOIN` ธรรมดาจะทำ):

```rust
// ✅ GROUP BY ครั้งเดียว: นับทุกเล่มพร้อมกันในคำสั่งเดียว
async fn count_group_by(pool: &sqlx::PgPool) -> Result<Vec<(i64, i64)>, sqlx::Error> {
    let rows = sqlx::query!(
        r#"SELECT b.id, COUNT(br.id) AS borrow_count
           FROM books b
           LEFT JOIN borrow_records br ON br.book_id = b.id
           GROUP BY b.id
           ORDER BY b.id"#
    )
    .fetch_all(pool)
    .await?;
    Ok(rows.into_iter().map(|r| (r.id, r.borrow_count.unwrap_or(0))).collect())
}
```

ผู้เขียนวัดจริงบนข้อมูล 40 เล่ม (แต่ละเล่มมี `borrow_records` เกี่ยวข้องหลายแถวตามที่ seed ไว้):

```
naive per-book count: 40 เล่ม, 41 queries, 24.49ms
GROUP BY: 40 เล่ม, 1 query, 1.14ms
speedup = 21.4x, query ลดจาก 41 เหลือ 1
```

ผลลัพธ์เดียวกันกับหัวข้อก่อนหน้าเป๊ะในเชิงรูปแบบ (41 queries เหลือ 1, เร็วขึ้น 21.4 เท่า) — ย้ำว่า N+1 ไม่ใช่แค่ "อย่าลืม join ตอน list ข้อมูล" เท่านั้น แต่เป็น**หลักการทั่วไป**ที่ต้องระวังทุกครั้งที่โค้ดต้อง "หาข้อมูลที่เกี่ยวข้องกับแต่ละแถว" ไม่ว่าจะเป็นรายละเอียดเต็ม ๆ หรือแค่ตัวเลขสรุปก็ตาม — คำตอบที่ถูกมักจะเป็น SQL aggregate (`GROUP BY`, `COUNT`, `SUM`, ...) เดียวที่ทำงานให้ทุกแถวพร้อมกัน แทนการวน loop สั่งให้ฐานข้อมูลทำงานเล็ก ๆ ซ้ำ ๆ หลายร้อยหลายพันครั้ง

### 71.12 Capstone: รวมทุกอย่างเข้ากับ Axum

มาผูกทุกหัวข้อของบทนี้เข้าด้วยกันเป็น endpoint จริงที่ต่อยอดจาก Axum server ของ Part 70 หัวข้อ 70.10 ตรง ๆ — เพิ่ม endpoint `GET /books` เวอร์ชันใหม่ที่รวม filter+sort+pagination (หัวข้อ 71.2/71.3) เข้ากับ migration ที่เพิ่ม `category`/index (หัวข้อ 71.6) และเพิ่ม endpoint `GET /books/borrowed` ที่หลีกเลี่ยง N+1 อย่างชัดเจน (หัวข้อ 71.11)

```rust
use axum::{
    extract::{Query, State},
    http::StatusCode,
    response::{IntoResponse, Response},
    routing::get,
    Json, Router,
};
use serde::{Deserialize, Serialize};
use sqlx::postgres::{PgPoolOptions, Postgres};
use sqlx::{PgPool, QueryBuilder};

#[derive(Debug, Serialize, sqlx::FromRow)]
struct BookRow {
    id: i64,
    isbn: String,
    title: String,
    category: String,
    available_copies: i32,
    total_count: i64,
}

#[derive(Debug, Deserialize)]
struct ListBooksQuery {
    category: Option<String>,
    min_available: Option<i32>,
    sort: Option<String>, // "asc" | "desc"
    page: Option<i64>,
    page_size: Option<i64>,
}

#[derive(Serialize)]
struct PagedResponse<T> {
    items: Vec<T>,
    total: i64,
    page: i64,
    page_size: i64,
}

#[derive(Debug)]
enum AppError {
    Internal(String),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let AppError::Internal(msg) = self;
        (StatusCode::INTERNAL_SERVER_ERROR, Json(serde_json::json!({ "error": msg }))).into_response()
    }
}

impl From<sqlx::Error> for AppError {
    fn from(e: sqlx::Error) -> Self {
        AppError::Internal(e.to_string())
    }
}

#[derive(Clone)]
struct AppState {
    pool: PgPool,
}

// endpoint เดียวที่รวม filter + sort + pagination ผ่าน QueryBuilder ตัวเดียว (หัวข้อ 71.2/71.3)
async fn list_books(
    State(state): State<AppState>,
    Query(params): Query<ListBooksQuery>,
) -> Result<Json<PagedResponse<BookRow>>, AppError> {
    let page = params.page.unwrap_or(0).max(0);
    let page_size = params.page_size.unwrap_or(10).clamp(1, 100);

    let mut qb: QueryBuilder<Postgres> = QueryBuilder::new(
        "SELECT id, isbn, title, category, available_copies, COUNT(*) OVER() AS total_count \
         FROM books WHERE 1 = 1",
    );

    if let Some(cat) = &params.category {
        qb.push(" AND category = ");
        qb.push_bind(cat.clone());
    }
    if let Some(min_avail) = params.min_available {
        qb.push(" AND available_copies >= ");
        qb.push_bind(min_avail);
    }

    match params.sort.as_deref() {
        Some("desc") => qb.push(" ORDER BY id DESC"),
        _ => qb.push(" ORDER BY id ASC"),
    };

    qb.push(" LIMIT ");
    qb.push_bind(page_size);
    qb.push(" OFFSET ");
    qb.push_bind(page * page_size);

    let items: Vec<BookRow> = qb.build_query_as().fetch_all(&state.pool).await?;
    let total = items.first().map(|b| b.total_count).unwrap_or(0);

    Ok(Json(PagedResponse { items, total, page, page_size }))
}

#[derive(Serialize)]
struct BorrowedBookRow {
    borrower_name: String,
    book_title: String,
    category: String,
}

// endpoint ที่หลีกเลี่ยง N+1 อย่างชัดเจน: หนึ่ง query เดียว join ตารางที่เกี่ยวข้องทั้งหมด (หัวข้อ 71.11)
async fn list_currently_borrowed(
    State(state): State<AppState>,
) -> Result<Json<Vec<BorrowedBookRow>>, AppError> {
    let rows = sqlx::query_as!(
        BorrowedBookRow,
        r#"SELECT br.borrower_name, b.title AS book_title, b.category
           FROM borrow_records br
           JOIN books b ON b.id = br.book_id
           WHERE br.returned_at IS NULL
           ORDER BY br.id"#
    )
    .fetch_all(&state.pool)
    .await?;
    Ok(Json(rows))
}

#[tokio::main]
async fn main() {
    let pool = PgPoolOptions::new()
        .max_connections(10)
        .connect("postgres://postgres:postgres@127.0.0.1:5432/part71_scratch")
        .await
        .expect("เชื่อมต่อฐานข้อมูลไม่สำเร็จ");

    let state = AppState { pool };

    let app = Router::new()
        .route("/books", get(list_books))
        .route("/books/borrowed", get(list_currently_borrowed))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:4071").await.unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}
```

ผู้เขียน**คอมไพล์และรัน server นี้จริง** แล้วยิง `curl` จริงหลายกรณี:

```bash
$ curl -s "http://127.0.0.1:4071/books?page_size=3"
{"items":[{"id":1,"isbn":"978-seed-0000","title":"Seed Book 0","category":"fiction","available_copies":1,"total_count":40},{"id":2,"isbn":"978-seed-0001","title":"Seed Book 1","category":"programming","available_copies":2,"total_count":40},{"id":3,"isbn":"978-seed-0002","title":"Seed Book 2","category":"history","available_copies":3,"total_count":40}],"total":40,"page":0,"page_size":3}

$ curl -s "http://127.0.0.1:4071/books?category=fiction&sort=desc&page_size=3"
{"items":[{"id":37,"isbn":"978-seed-0036","title":"Seed Book 36","category":"fiction","available_copies":2,"total_count":10},{"id":33,"isbn":"978-seed-0032","title":"Seed Book 32","category":"fiction","available_copies":3,"total_count":10},{"id":29,"isbn":"978-seed-0028","title":"Seed Book 28","category":"fiction","available_copies":4,"total_count":10}],"total":10,"page":0,"page_size":3}

$ curl -s "http://127.0.0.1:4071/books/borrowed" | head -c 300
[{"borrower_name":"Borrower 0","book_title":"Seed Book 0","category":"fiction"},{"borrower_name":"Borrower 1","book_title":"Seed Book 1","category":"programming"},{"borrower_name":"Borrower 2","book_title":"Seed Book 2","category":"history"}, ...
```

ทุก endpoint ทำงานตรงตามที่ออกแบบ: `total` เปลี่ยนตาม filter ที่ใช้จริง (`40` ไม่กรอง, `10` กรอง `fiction`), `sort=desc` เรียง id จากมากไปน้อยจริง (`37, 33, 29`), และ `/books/borrowed` คืนข้อมูลที่ join มาจากทั้ง `borrow_records` และ `books` ในคำสั่งเดียว (query เดียว ไม่มี N+1) — นี่คือตัวอย่างที่รวมทุกเทคนิคของบทนี้ (`QueryBuilder`, `COUNT(*) OVER()`, migration ที่เพิ่ม `category`/index, และการหลีกเลี่ยง N+1 ด้วย `JOIN`) เข้าเป็นระบบเดียวที่ทำงานได้จริงครบวงจร ต่อยอดจาก `AppState`/`PgPool` pattern เดียวกันกับ Part 70 ทุกประการ

### 71.13 เลือกเทคนิคให้เหมาะกับสถานการณ์: ตารางสรุปการตัดสินใจ

บทนี้ผ่านเทคนิคมาหลายตัวที่แก้ปัญหาคล้ายกันในรายละเอียดต่างกัน — ตารางนี้สรุปเป็น "ถ้าเจอสถานการณ์แบบนี้ ให้นึกถึงเทคนิคนี้ก่อน" เพื่อใช้เป็นจุดเริ่มต้นตัดสินใจเร็ว ๆ ในงานจริง (ไม่ใช่กฎตายตัวที่ใช้ได้ทุกกรณีเสมอไป แต่เป็นจุดเริ่มต้นที่ดีก่อนตัดสินใจลงรายละเอียด):

| สถานการณ์ | เทคนิคที่ควรนึกถึงก่อน | เหตุผลสั้น ๆ |
|---|---|---|
| จำนวนเงื่อนไข `WHERE` ไม่แน่นอนตาม input | `QueryBuilder` (`.push()`/`.push_bind()`) | `query!`/`query_as!` ต้องการ SQL literal ตายตัว ใช้กับ filter แบบ dynamic ไม่ได้ตรง ๆ |
| ต้อง list ข้อมูลพร้อมจำนวนรวมทั้งหมด | `COUNT(*) OVER()` แทนสอง query แยก | ลด round-trip จาก 2 เหลือ 1 โดยไม่เสีย correctness |
| Insert/update ข้อมูลมากกว่า ~20 แถวพร้อมกัน | `UNNEST` (SQL literal ตายตัว) หรือ `QueryBuilder::push_values` (ถ้าต้อง dynamic อยู่แล้ว) | ลด round-trip จาก N ครั้งเหลือ 1 ครั้ง — วัดจริงเร็วขึ้น 39-70 เท่า |
| ต้อง fetch หลายแถวจาก id ที่รู้อยู่แล้วเป็นลิสต์ | `WHERE id = ANY($1)` | ไม่ต้องสร้าง SQL แบบ dynamic ตามจำนวน id เหมือน `IN (...)` |
| ข้อมูล metadata ที่ shape ต่างกันตามประเภท | `sqlx::types::Json<T>` กับ enum ที่ tag ด้วย serde | ได้ type-safety เต็มรูปแบบ ดีกว่า `serde_json::Value` แบบ dynamic เมื่อรู้ shape ล่วงหน้า |
| ต้อง filter/ค้นภายใน JSONB บ่อย ๆ บนตารางใหญ่ | GIN index + operator `@>` | `->>`  ธรรมดาต้องสแกนทั้งตารางเสมอ ไม่มีทาง index ช่วยตรง ๆ |
| เพิ่มคอลัมน์ใหม่บนตารางที่มีข้อมูลอยู่แล้ว | `ADD COLUMN ... DEFAULT ...` เสมอถ้าเป็น `NOT NULL` | ไม่มี default จะ error ทันทีถ้าตารางไม่ว่าง |
| ลบ/เปลี่ยนชื่อคอลัมน์บนระบบที่ deploy แบบ rolling | Expand-Contract (เพิ่มก่อน ค่อยลบทีหลัง) | ป้องกัน instance เก่าที่ยังรันอยู่ระหว่าง deploy พังกลางอากาศ |
| API timeout พร้อมกันตอน traffic สูง | เช็ค `pool.size()`/`num_idle()` ก่อนปรับ `max_connections` | ต้องแยกให้ออกว่า pool เล็กเกินไป หรือ query ช้าลงจริง ก่อนแก้ |
| ต้องแจ้งเตือนแบบเบา ๆ ข้าม instance โดยไม่อยากตั้ง message queue | `PgListener` (`LISTEN`/`NOTIFY`) | ใช้ PostgreSQL ที่มีอยู่แล้ว แต่ไม่มี delivery guarantee — ไม่ใช่ทางเลือกสำหรับงานสำคัญจริงจัง |
| Test suite ใหญ่มากจนรันช้า | Transaction ที่ไม่ commit แทน `#[sqlx::test]` | เร็วกว่ามากเพราะไม่ต้อง `CREATE DATABASE`/migration ทุก test แลกกับ isolation ที่ต้องเขียนเอง |
| เห็น query ในลูป (ไม่ว่าจะหารายละเอียดหรือแค่นับ) | `JOIN`/`GROUP BY` ครั้งเดียว | นี่คือ N+1 — วัดจริงในบทนี้เร็วขึ้น 21-25 เท่าทุกกรณีที่ทดสอบ |

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. ต่อ SQL string เองสำหรับ filter แบบ dynamic แทนใช้ `QueryBuilder`

```rust
// ❌ ผิด — ต่อ SQL string ตามเงื่อนไขที่มีเอง (ห้ามทำแบบนี้เด็ดขาด)
let sql = format!("SELECT COUNT(*) FROM books WHERE category = '{category}'");
let count: i64 = sqlx::query_scalar(&sql).fetch_one(pool).await?;
```

`cargo build` fail ทันทีใน SQLx 0.9.0 ด้วย error จริง:

```
error[E0277]: dynamic SQL strings should be audited for possible injections
    |
    = help: the trait `SqlSafeStr` is not implemented for `&std::string::String`
    = note: prefer literal SQL strings with bind parameters or `QueryBuilder` to add dynamic data to a query.
    = note: this trait is only implemented for `&'static str`, not all `&str` like the compiler error may suggest
```

**วิธีแก้**: ใช้ `sqlx::QueryBuilder` ตามหัวข้อ 71.2 เสมอเมื่อจำนวนเงื่อนไขไม่แน่นอน — `.push()` สำหรับ SQL fragment ที่เขียนเอง (ไม่มาจาก input ผู้ใช้) และ `.push_bind()` สำหรับค่าจาก input ทุกตัวโดยไม่มีข้อยกเว้น สังเกตว่า compiler suggest ให้ `&*sql` (dereference) ซึ่ง**ไม่ใช่ทางแก้ที่ถูก** (ยัง error เดิมเพราะปัญหาไม่ใช่เรื่อง reference) — ต้องเข้าใจสาเหตุจริง (`String` จาก `format!()` ไม่มีทาง implement `SqlSafeStr` ได้) ก่อนเลือกทางแก้ที่ถูกต้อง

### 2. `ALTER TABLE ... ADD COLUMN ... NOT NULL` โดยไม่มี `DEFAULT` บนตารางที่มีข้อมูลอยู่แล้ว

```sql
-- ❌ ตารางนี้มีข้อมูลอยู่แล้ว 40 แถว — คำสั่งนี้จะ fail ทันที
ALTER TABLE books ADD COLUMN sku TEXT NOT NULL;
```

รันจริงผ่าน `psql` ได้ error จริง:

```
ERROR:  column "sku" of relation "books" contains null values
```

**อธิบาย**: PostgreSQL ต้องหาค่าให้แถวเก่าทุกแถวที่มีอยู่แล้วสำหรับคอลัมน์ใหม่ที่เพิ่ง `ADD COLUMN` — ถ้าคอลัมน์นั้นเป็น `NOT NULL` แต่ไม่มี `DEFAULT` กำหนดไว้ PostgreSQL ไม่มีทางรู้ว่าจะใส่ค่าอะไรให้แถวเก่า (ค่า default ของคอลัมน์ที่ไม่ระบุคือ `NULL` ซึ่งขัดกับ `NOT NULL` ที่เพิ่งประกาศทันที) **วิธีแก้**: เพิ่ม `DEFAULT <ค่าที่เหมาะสม>` เสมอเมื่อ `ADD COLUMN ... NOT NULL` บนตารางที่**อาจ**มีข้อมูลอยู่แล้ว (ปลอดภัยกว่าที่จะใส่ `DEFAULT` เสมอแม้จะมั่นใจว่าตารางว่างตอนเขียน migration ก็ตาม เพราะไม่รู้ว่า migration จะถูกรันตอนไหนในอนาคต — ตามที่หัวข้อ 71.6 ทำกับ `category TEXT NOT NULL DEFAULT 'general'`) ถ้าต้องการคอลัมน์ที่ไม่มี default ที่สมเหตุสมผลจริง ๆ ให้แยกเป็นสองขั้นตอน: `ADD COLUMN` แบบ nullable ก่อน, `UPDATE` ให้ค่าทุกแถวเก่า, แล้วค่อย `ALTER COLUMN ... SET NOT NULL` ในอีก migration หรือคำสั่งถัดไป

### 3. `sqlx::query!`/`query_as!` อ้างถึงคอลัมน์ที่ migration ยังไม่ได้รัน

```rust
// ❌ column "tags" ยังไม่มีจริงในฐานข้อมูล (migration ยังไม่ถูกสร้าง/รัน)
let _row = sqlx::query!("SELECT id, tags FROM books LIMIT 1").fetch_one(&pool).await?;
```

`cargo build` ให้ error จริงทันที (ต่อฐานข้อมูลจริงตอน compile ตามที่ Part 70 หัวข้อ 70.1 อธิบายไว้):

```
error: error returned from database: column "tags" does not exist at line 3722
  --> src/bin/missing_column_check.rs:10:16
   |
10 |     let _row = sqlx::query!("SELECT id, tags FROM books LIMIT 1")
   |                ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

**วิธีแก้**: ต้องรัน `sqlx migrate run` ให้ schema ของฐานข้อมูลที่ `DATABASE_URL` ชี้ไปตรงกับที่โค้ดคาดหวัง**ก่อน**สั่ง `cargo build` เสมอ — นี่เป็นกับดักที่พบบ่อยมากเมื่อทำงานเป็นทีม: เพื่อนร่วมทีม pull โค้ดที่มี `query!` อ้างถึงคอลัมน์ใหม่มา แต่ลืมรัน migration ใหม่ที่มาด้วยกันก่อน build — ลำดับที่ถูกต้องเสมอคือ `git pull` → `sqlx migrate run` → `cargo build` ไม่ใช่ตรงกันข้าม (ถ้าใช้ offline mode ผ่าน `.sqlx/` cache ตามที่ Part 70 หัวข้อ 70.1 อธิบายไว้ ต้องแน่ใจว่า `.sqlx/` cache ที่ commit มาถูก generate ใหม่หลัง migration ล่าสุดด้วย `cargo sqlx prepare` เช่นกัน ไม่อย่างนั้นจะ build ผ่านแต่ schema ไม่ตรงกับความจริงตอน runtime)

### 4. Insert ทีละแถวในลูปสำหรับข้อมูลจำนวนมาก (ไม่ error แต่ช้ากว่าที่ควรมาก)

```rust
// ❌ compile ผ่าน รันได้ผลลัพธ์ถูกต้อง — แต่ช้ากว่าที่ควรมากสำหรับข้อมูลจำนวนมาก
for book in books {
    sqlx::query!("INSERT INTO books (...) VALUES ($1, $2, ...)", book.isbn, book.title, ...)
        .execute(pool)
        .await?;
}
```

นี่คือกับดักที่**ไม่มี error ให้เห็นเลย** — compile ผ่าน รันผ่าน ได้ผลลัพธ์ถูกต้อง 100% ปัญหาเป็นเรื่องประสิทธิภาพล้วน ๆ ที่ไม่มี compiler หรือ runtime ช่วยเตือนให้ ผู้เขียนวัดจริงตามหัวข้อ 71.4 ได้ **813.06ms** (insert ทีละแถว 2,000 แถว) เทียบกับ **11.58ms** (bulk insert ด้วย `UNNEST` จำนวนแถวเท่ากัน) — ต่างกัน **70.2 เท่า** **วิธีแก้**: เมื่อต้อง insert มากกว่าสิบ-ยี่สิบแถวขึ้นไปในครั้งเดียว ให้ใช้เทคนิค bulk insert ด้วย `UNNEST` (ตามหัวข้อ 71.4) เป็นค่า default แทนวน loop — กับดักนี้อันตรายเพราะไม่มีสัญญาณเตือนใด ๆ ตอน dev ที่ข้อมูลน้อย (ต่างกันแค่มิลลิวินาที มองไม่เห็นปัญหา) แต่ระเบิดจริงตอน production ที่ต้อง import ข้อมูลจำนวนมาก

### 5. N+1 Query โดยไม่รู้ตัว (query ในลูปที่ดู "เป็นธรรมชาติ")

```rust
// ❌ ดู "สมเหตุสมผล" มาก แต่คือ N+1 query problem ตัวเต็ม
let records = sqlx::query!("SELECT book_id, borrower_name FROM borrow_records").fetch_all(pool).await?;
for r in records {
    let title = sqlx::query_scalar!("SELECT title FROM books WHERE id = $1", r.book_id)
        .fetch_one(pool)
        .await?; // query แยกทีละแถว — อันตรายที่สุดของกับดักนี้คือไม่มี error ให้เห็นเลย
}
```

เช่นเดียวกับกับดักที่ 4 นี่คือปัญหาที่**ไม่มี compiler error หรือ runtime error ใด ๆ** ให้เห็น — ผู้เขียนวัดจริงตามหัวข้อ 71.11 ได้ **201 queries, 27.67ms** (naive loop กับข้อมูล 200 แถว) เทียบกับ **1 query, 1.10ms** (`JOIN` เดียว) — ต่างกัน **25.2 เท่า** ทั้งเวลาและจำนวน query **วิธีตรวจจับ**: ทุกครั้งที่เห็น `for`/`while` loop ที่ข้างในมีการเรียก query ให้สงสัยไว้ก่อนเสมอว่าอาจเป็น N+1 — เครื่องมือที่ช่วยตรวจจับสิ่งนี้ได้จริงในโลกจริงคือการ log จำนวน query ที่ยิงไปฐานข้อมูลต่อ request หนึ่งครั้ง (ถ้าตัวเลขนี้ผูกกับขนาดของ response แบบเป็นเส้นตรง นั่นคือสัญญาณเตือนของ N+1) **วิธีแก้**: ใช้ `JOIN` (ถ้าข้อมูลอยู่ในฐานข้อมูลเดียวกัน) หรือรวบรวม id ทั้งหมดก่อนแล้วยิง `WHERE id = ANY($1)` เป็น batch เดียว (ถ้าต้อง fetch จากแหล่งอื่นแยกกันจริง ๆ)

### 6. เข้าใจผิดว่า `PgListener` ยืม Connection จาก Pool (นับรวมกับ `max_connections`)

```rust
let pool = PgPoolOptions::new().max_connections(5).connect(url).await?;
println!("pool.size() ก่อนเปิด listener = {}", pool.size());

let mut listener = PgListener::connect_with(&pool).await?;
listener.listen("gotcha_channel").await?;

println!("pool.size() หลังเปิด listener = {}", pool.size());
```

ผู้เขียนรันจริงเพื่อพิสูจน์ว่า `pool.size()` **ไม่เปลี่ยน**เลยแม้เปิด `PgListener` ไปแล้ว:

```
pool.size() ก่อนเปิด listener = 1
pool.size() หลังเปิด listener = 1 (ไม่เพิ่ม — เป็น connection แยก)
```

**อธิบาย**: แม้ `PgListener::connect_with(&pool)` จะรับ `&PgPool` เป็น argument (ทำให้ดูเหมือนว่ามันยืม connection จาก pool ตัวนั้น) แต่จริง ๆ แล้วมันแค่**อ่าน connection config** จาก pool (host, port, user, password, database) มาใช้เปิด **connection ใหม่ของตัวเอง** ที่แยกออกไปต่างหากอย่างสิ้นเชิง — connection ของ `PgListener` จึงไม่ถูกนับรวมกับ `max_connections` ของ pool เลย (พิสูจน์แล้วจาก `pool.size()` ที่ไม่ขยับ) แต่**ยังคงเป็น connection จริงหนึ่งตัว**ที่กิน resource ฝั่ง PostgreSQL เหมือน connection ปกติทุกประการ (ปรากฏใน `pg_stat_activity` เหมือนกัน) — ผลที่ตามมาอีกจุดที่ต้องรู้: `pool.close()` (Part 70 หัวข้อ 70.12) **ไม่ได้ปิด connection ของ `PgListener` ไปด้วย** เพราะมันไม่ได้เป็นส่วนหนึ่งของ pool ตั้งแต่แรก — ถ้าเปิด `PgListener` ไว้แล้วต้อง shutdown แอปอย่างเป็นระเบียบ ต้องปิด `listener` เอง (หรือปล่อยให้ `Drop` ของมันทำงานตามธรรมชาติเมื่อ scope จบ) แยกจากการเรียก `pool.close()` **วิธีแก้/ข้อควรจำ**: นับจำนวน `PgListener` ที่เปิดไว้ในระบบแยกจากการคำนวณ `max_connections` ของ pool เสมอ (ถ้าเปิด listener หลายตัวโดยไม่ได้ตั้งใจ เช่น เปิดใหม่ทุกครั้งที่ handler ถูกเรียกโดยไม่ปิดตัวเก่า จะสร้าง connection รั่วไหลสะสมที่ไม่มีทาง track ผ่าน `pool.size()` ได้เลย)

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เพิ่ม field `min_total_copies: Option<i32>` เข้า `ListParams`/`ListBooksQuery` ของหัวข้อ 71.2/71.12 (กรองหนังสือที่มี `total_copies` มากกว่าหรือเท่ากับค่าที่ระบุ) — Hint: เพิ่ม `if let Some(...)` อีกหนึ่งบล็อกในฟังก์ชัน `list_books_dynamic`/`list_books` ตามรูปแบบเดียวกับ `min_available` ที่มีอยู่แล้ว ระวังอย่าลืม `.push_bind()` (ไม่ใช่ `.push()`) สำหรับค่าที่มาจาก query parameter

2. **(กลาง)** เขียนฟังก์ชัน `bulk_update_category(pool: &PgPool, updates: &[(i64, String)]) -> Result<u64, sqlx::Error>` ที่รับลิสต์ของ `(book_id, new_category)` แล้วอัปเดต `category` ของหลายเล่มพร้อมกันในคำสั่งเดียว (ห้ามวน loop เรียก `UPDATE` ทีละแถว) — Hint: ใช้เทคนิคคล้าย `UNNEST` ของหัวข้อ 71.4 ผสมกับ `UPDATE ... FROM UNNEST(...)`: `UPDATE books SET category = t.new_category FROM UNNEST($1::bigint[], $2::text[]) AS t(id, new_category) WHERE books.id = t.id` — ทดสอบด้วยการวัดเวลาเทียบกับวิธี loop ทีละแถวแบบหัวข้อ 71.4 ตัวเลขที่ได้ควรต่างกันในทิศทางเดียวกับที่บทนี้วัดไว้

3. **(ยาก)** เพิ่ม endpoint `GET /books/:id/history` ที่คืนประวัติการยืม-คืนของหนังสือเล่มหนึ่ง **พร้อมชื่อผู้ยืมทุกคน** โดยต้องไม่มี N+1 เลย (คำนวณจำนวน query ที่ใช้จริงด้วยการนับ manual หรือเปิด PostgreSQL query log แล้วนับ) จากนั้นเพิ่ม migration ใหม่ที่สร้าง index บน `borrow_records (book_id, borrowed_at DESC)` (composite index สำหรับ query pattern "หาประวัติของหนังสือเล่มหนึ่ง เรียงจากล่าสุด") — Hint: query เดียวที่มี `WHERE book_id = $1 ORDER BY borrowed_at DESC` เพียงพอแล้ว ไม่ต้อง `JOIN` อะไรเพิ่มเพราะข้อมูลอยู่ในตารางเดียว ส่วน index ให้ทดสอบด้วย `EXPLAIN ANALYZE` ก่อน/หลังสร้าง index เทียบว่า query planner เลือกใช้ index scan แทน sequential scan หรือไม่ (ต้องมีข้อมูลมากพอสมควรก่อน PostgreSQL จะเลือกใช้ index จริง ๆ — ตารางที่มีข้อมูลน้อยมาก ๆ อาจยังเลือก sequential scan เพราะเร็วกว่าในทางปฏิบัติ)

4. **(ยาก/ประยุกต์ใช้งานจริง)** สร้างระบบแจ้งเตือนแบบง่ายด้วย `PgListener` (หัวข้อ 71.9): ทุกครั้งที่มีการสร้าง `borrow_records` ใหม่ ให้ trigger PostgreSQL (`CREATE TRIGGER` + `CREATE FUNCTION` ที่เรียก `pg_notify()`) ส่ง notification ไปยัง channel `borrow_events` โดยอัตโนมัติ (ไม่ต้องพึ่งโค้ด Rust เรียก `pg_notify()` เอง) แล้วเขียนโปรแกรม Rust ที่ `LISTEN` channel นี้และพิมพ์ log ทุกครั้งที่มีการยืมหนังสือเกิดขึ้นจริง (ไม่ว่าการยืมนั้นจะมาจาก endpoint ไหนของระบบก็ตาม) — Hint: `CREATE FUNCTION notify_borrow() RETURNS TRIGGER AS $$ BEGIN PERFORM pg_notify('borrow_events', NEW.id::text); RETURN NEW; END; $$ LANGUAGE plpgsql;` แล้ว `CREATE TRIGGER borrow_notify_trigger AFTER INSERT ON borrow_records FOR EACH ROW EXECUTE FUNCTION notify_borrow();` — ลองทดสอบด้วยการ insert ผ่าน `psql` ตรง ๆ (ไม่ผ่านโค้ด Rust เลย) แล้วดูว่าโปรแกรม listener ยังรับ notification ได้ไหม (คำตอบคือได้ เพราะ trigger ทำงานที่ระดับฐานข้อมูล ไม่ว่า insert จะมาจากไหน) — นี่คือข้อแตกต่างสำคัญจากการเรียก `pg_notify()` จากโค้ด Rust เองตรง ๆ ตามตัวอย่างในบทนี้

## สรุป

บทนี้ต่อยอดจากพื้นฐาน CRUD/transaction/pool ที่ Part 70 วางไว้ ไปสู่เทคนิคระดับที่ระบบจริงต้องใช้จริง — สิ่งสำคัญที่ควรจำ:

- **`sqlx::QueryBuilder`** สร้าง SQL แบบ dynamic ได้อย่างปลอดภัยเท่ากับ `query!`/`query_as!` — กฎเดียวที่ต้องจำ: `.push()` สำหรับ SQL fragment ที่เขียนเอง, `.push_bind()` สำหรับค่าจาก input ผู้ใช้**ทุกตัวโดยไม่มีข้อยกเว้น** ไม่มีเหตุผลใดที่ต้องกลับไปต่อ string เองอีก
- **`COUNT(*) OVER()`** ให้ทั้งข้อมูลหน้าปัจจุบันและจำนวนแถวทั้งหมดในคำสั่ง SQL เดียว ลด round-trip จากสองครั้งเหลือครั้งเดียวสำหรับ pagination — ต้องมี `ORDER BY` ที่ deterministic เสมอคู่กับ `LIMIT`/`OFFSET`
- **Bulk insert ด้วย `UNNEST` เร็วกว่า insert ทีละแถวในลูปถึง 70 เท่า** จากการวัดจริง (2,000 แถว) — ใช้ `= ANY($1)` แทนวน loop query ทีละ id เมื่อต้อง fetch หลายรายการพร้อมกันด้วยเหตุผลเดียวกัน
- **`sqlx::types::Json<T>`** ผูก `JSONB` เข้ากับ struct/enum ของ Rust ตรง ๆ ได้ type-safety เต็มรูปแบบ (ดีกว่า `serde_json::Value` แบบ dynamic เมื่อ shape ของข้อมูลรู้ล่วงหน้าตามประเภท) — query เข้าไปข้างในผ่าน operator `->>`/`->` ของ PostgreSQL ได้โดยไม่ต้องดึงทุกแถวมา filter ฝั่ง Rust
- **`ALTER TABLE ... ADD COLUMN ... NOT NULL` ต้องมี `DEFAULT` เสมอ** บนตารางที่อาจมีข้อมูลอยู่แล้ว — พิสูจน์ด้วย error จริงถ้าลืม ตาราง **`_sqlx_migrations`** เก็บ version/checksum/execution_time ของทุก migration ที่รันไปแล้ว เป็นแหล่งความจริงเดียวว่าฐานข้อมูลอยู่ที่ schema เวอร์ชันไหน
- **`max_connections` ผูกตรงกับจำนวน concurrent task ที่คุยกับฐานข้อมูลได้พร้อมกัน** — pool เล็กเกินไปทำให้ request ต้องรอเข้าคิวยาวขึ้นเรื่อย ๆ ตอน traffic สูง (พิสูจน์ด้วยตัวเลขจริง: 614.90ms เทียบกับ 115.24ms สำหรับ pool เล็ก/ใหญ่พอ) จนอาจเกิน timeout พร้อมกันเป็นชุดใหญ่ — นี่คือกลไกที่แท้จริงเบื้องหลังอาการ "API timeout พร้อมกันหมดตอน traffic สูง"
- **Statement cache** ของ SQLx (`statement_cache_capacity`, default 100) ลด overhead ของการ `PREPARE` query pattern ซ้ำ ๆ — ตั้งผ่าน `PgConnectOptions` (คนละ layer จาก `PgPoolOptions`)
- **`PgListener`** รับ `LISTEN`/`NOTIFY` แบบ real-time ได้จริง แต่ไม่มี delivery guarantee/persistent buffer — เหมาะกับการแจ้งเตือนแบบเบา ๆ เท่านั้น ไม่ใช่ตัวแทนของ message queue เต็มรูปแบบ (Part 82)
- **Pattern "transaction ที่ไม่เคย commit" ต่อ test** เป็นทางเลือกที่เร็วกว่า `#[sqlx::test]` สำหรับ test suite ขนาดใหญ่ (แลกกับ isolation ที่ต้องเขียนเองให้ถูกต้อง) — พิสูจน์จริงว่าข้อมูลไม่หลุดออกจาก transaction ที่ไม่ commit
- **N+1 query problem** คือ query ในลูปที่ "ดูปกติ" แต่ไม่มี error ให้เห็นเลยจนกว่าจะวัดจริง — พิสูจน์ด้วยตัวเลข 201 queries/27.67ms เทียบกับ 1 query/1.10ms (เร็วขึ้น 25.2 เท่า) แก้ด้วย `JOIN` หรือ `= ANY($1)` แบบ batch

Part ถัดไป (Part 72) จะแนะนำ **Diesel** — ORM อีกตัวของ Rust ecosystem ที่เลือกจุดยืนต่างจาก SQLx ชัดเจน (Rust DSL ที่ generate SQL ให้ แทนเขียน SQL string ด้วยมือ, type-safety ผ่าน schema ที่ generate เป็น Rust type แทน macro ที่ต่อฐานข้อมูลจริงตอน compile) — ทุกแนวคิดเรื่อง connection pool, transaction, migration ที่ Part 70-71 วางพื้นฐานไว้จะยังมีประโยชน์ตรงเมื่อเทียบกับแนวทางของ Diesel เพราะเป็นปัญหาเดียวกันที่ทั้งสอง library ต้องแก้ เพียงแต่เลือกวิธีแก้ต่างกัน

---

**Part ก่อนหน้า:** [Database: เชื่อมต่อ PostgreSQL ด้วย SQLx](part-070-sqlx-postgresql.md) | **Part ถัดไป:** [Diesel ORM เบื้องต้น](part-072-diesel-orm.md)
