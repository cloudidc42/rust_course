# Part 72: Diesel ORM เบื้องต้น

> โมดูล: การพัฒนาเว็บแอปพลิเคชัน (Web Development) | ระดับ: สูง | เวลาโดยประมาณ: 260 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายความต่างเชิงปรัชญาระหว่าง **Diesel** กับ **SQLx** (Part 70-71) ได้อย่างชัดเจนและลึก: Diesel สร้าง query ผ่าน **Rust DSL แบบ type-safe** (`books.filter(title.eq("Rust"))`) ที่ compiler ตรวจสอบผ่าน **trait system ของ Rust เอง** โดยไม่ต้องมี connection ไปยังฐานข้อมูลตอน compile เลย ต่างจาก SQLx ที่คุณเขียน SQL string จริงแล้วให้ macro ไปตรวจกับฐานข้อมูลจริง (หรือ cache) — ทั้งสองแนวทางแก้ปัญหาเดียวกัน (จับ error ตอน compile ไม่ใช่ตอน runtime) แต่คนละกลไกโดยสิ้นเชิง
- อธิบายได้ว่าทำไม Diesel เป็น **synchronous by default** (core design มาจากก่อนยุค async ecosystem ของ Rust จะโตเต็มที่) และรู้วิธีใช้งาน Diesel ให้ถูกต้องภายใน Axum handler แบบ async โดยไม่ทำให้ Tokio runtime ถูก block — ผ่านการห่อ query ด้วย `tokio::task::spawn_blocking` (ต่อยอดจาก Part 48) พร้อมพิสูจน์ด้วยโค้ดจริงว่าเกิดอะไรขึ้นถ้าลืมห่อ
- ติดตั้งและตั้งค่าโปรเจกต์ Diesel เต็มรูปแบบ: ติดตั้ง `diesel_cli`, รัน `diesel setup`, สร้าง migration ด้วย `diesel migration generate`, และเข้าใจว่า Diesel **generate `schema.rs`** จาก schema จริงในฐานข้อมูลผ่าน `diesel print-schema` เพื่อเป็น single source of truth ที่ query DSL ทั้งหมดต้อง type-check กับมัน
- นิยาม model struct สองแบบที่ Diesel แยกกันตามธรรมเนียม: `#[derive(Queryable, Selectable)]` สำหรับอ่านข้อมูล และ `#[derive(Insertable)]` สำหรับ struct "ข้อมูลใหม่" ที่ใช้ตอน insert (พร้อมอธิบายว่าทำไมต้องแยกกัน ไม่ใช้ struct เดียวกันทั้งอ่านทั้งเขียน)
- เขียน CRUD เต็มรูปแบบผ่าน Diesel DSL บนโดเมนห้องสมุด (`books`/`shelves`) ที่ต่อยอดจาก Part 70-71: `.filter()`, `.select()`, `.order()`, `.limit()`, `.values()`, `.set()` พร้อมดู SQL จริงที่ Diesel generate ให้ผ่าน `diesel::debug_query!`
- เขียน join แบบ one-to-many จริงด้วย `.inner_join()` และเข้าใจ error ตอน compile ที่เกิดขึ้นเมื่อ query DSL ผิด (คอลัมน์ไม่มีจริง หรือ type ไม่ตรง) รวมถึงตัดสินใจได้ว่าโปรเจกต์แบบไหนควรเลือก Diesel เทียบกับ SQLx (และ SeaORM ใน Part 73)

## ความรู้ที่ต้องมีมาก่อน

- **Part 70-71 (SQLx: PostgreSQL, Queries/Migrations/Pooling)**: บทนี้ใช้โดเมนเดียวกัน (ระบบห้องสมุด: ตาราง `books`, `borrow_records` และตารางใหม่ `shelves`) เพื่อให้เห็นภาพเปรียบเทียบตรง ๆ ว่า "query เดียวกัน" เขียนต่างกันอย่างไรระหว่าง SQLx กับ Diesel — ถ้ายังไม่ผ่าน Part 70-71 ควรอ่านก่อน เพราะบทนี้จะอ้างอิงตารางสรุปเปรียบเทียบที่ Part 70 วางไว้ตลอดเวลา
- **Part 48 (Tokio Runtime และ Async/Blocking)**: หัวข้อ 72.2 ของบทนี้คือการนำความเข้าใจเรื่อง "ห้ามเรียก blocking code ตรง ๆ ใน async context" และ `tokio::task::spawn_blocking` จาก Part 48 มาใช้งานจริงกับ Diesel — ถ้าจำหลักการ blocking thread pool ของ Tokio ไม่ได้ ควรทวนก่อน เพราะบทนี้จะไม่อธิบาย mechanism ของ `spawn_blocking` ซ้ำในระดับ Tokio internals
- **Part 44-45 (Macro และ Procedural Macro)**: derive macro ของ Diesel (`Queryable`, `Selectable`, `Insertable`) และ macro `table!`/`diesel::sql_query`/`debug_query!` ทั้งหมดเป็น proc macro ที่ทำงานตาม mechanism เดียวกับที่ Part 44-45 สอน (รับ `TokenStream` มา parse แล้ว generate โค้ด) เพียงแต่ generate โค้ดที่ผูกกับ SQL DSL แทน trait ทั่วไป
- **Part 9 (Struct) และ Part 57 (Serde)**: บทนี้ยังใช้ struct เป็นตัวแทนแถวข้อมูลเหมือน Part 70 แต่ derive macro ของ Diesel เป็นระบบแยกจาก serde โดยสิ้นเชิง (แม้บาง project จะ derive ทั้งสองชุดพร้อมกันบน struct เดียว) — บทนี้จะชี้ให้เห็นตรงจุดที่ derive สองระบบนี้ทำงานคู่กันได้อย่างไรและต่างกันอย่างไรในระดับ mechanism
- **Part 39 (Arc/Mutex) และ Part 64 (Axum State)**: connection pool ของ Diesel (ผ่าน `r2d2`) เก็บใน `AppState` ด้วย pattern เดียวกับที่ Part 64 สอน (`State<T>` extractor) เพียงแต่ pool ประเภทนี้เป็น**pool ของ synchronous connection** ที่ต้องส่งผ่าน `spawn_blocking` เข้าไปใช้ ต่างจาก `PgPool` ของ SQLx ที่ใช้ตรงในบล็อก async ได้เลย
- **Part 12, 30-31 (Error Handling)**: การแปลง `diesel::result::Error` ให้เป็น error type ของแอปเองใช้หลักการ `impl From<...>` เดียวกับที่ Part 66 สอนไว้กับ `sqlx::Error`
- **พื้นฐาน SQL**: เหมือน Part 70 บทนี้ไม่สอน SQL ตั้งแต่ต้น สมมติว่าคุณอ่าน `SELECT`/`INSERT`/`UPDATE`/`DELETE`/`JOIN` พื้นฐานออกแล้ว

## หมายเหตุเรื่องการตรวจสอบเนื้อหา (สำคัญ — อ่านก่อนเริ่ม)

ก่อนเขียนบทนี้ ผู้เขียนตรวจสอบสภาพแวดล้อมจริงในเครื่องที่ใช้เขียนบทนี้ พบว่า **มี PostgreSQL 16.13 รันอยู่จริง** (cluster เดียวกับที่ Part 70 ใช้ เปิดผ่าน `pg_ctlcluster 16 main`) จึงสร้างฐานข้อมูลสคแรทช์แยกชื่อ `diesel_course_scratch` ขึ้นมาใหม่ (ไม่แตะฐานข้อมูลของ Part อื่นที่อาจกำลังถูกใช้งานพร้อมกันโดย agent อื่น) และตั้งใจทดสอบทุกอย่างแบบ end-to-end เหมือน Part 70

**สิ่งที่ทดสอบจริงและยืนยันได้:**

- ติดตั้ง `diesel_cli` จริงด้วย `cargo install diesel_cli --no-default-features --features postgres` (compile จาก source ตามที่คำสั่งนี้ทำงานปกติ)
- สร้างโปรเจกต์ scratch แยกนอก repo เพิ่ม `diesel` เป็น dependency พร้อม feature ตามที่บทนี้สอน
- รัน `diesel setup`, `diesel migration generate`, เขียน SQL migration จริงสำหรับตาราง `books`/`shelves`/`borrow_records`, รัน `diesel migration run` จริง แล้วรัน `diesel print-schema` จริงเพื่อดู `schema.rs` ที่ generate จริง
- คอมไพล์และรันโค้ด Rust ที่ใช้ DSL ของ Diesel จริง (filter/select/order/limit/insert/update/delete/join) ต่อฐานข้อมูลจริง พร้อมจับ SQL string จริงที่ Diesel generate ผ่าน `diesel::debug_query!`
- จงใจเขียน query ที่ผิด (คอลัมน์ไม่มีจริง) เพื่อจับ **compiler error จริง** ของ Diesel มาแสดงในหัวข้อ 72.9

**อุปสรรคที่เจอจริงและวิธีแก้ที่ใช้จริง**: การ compile `diesel_cli`/โปรเจกต์ที่ใช้ Diesel กับ feature `postgres` ต้องพึ่ง **`libpq`** (C client library ตัวเดียวกับที่ `psql` ใช้) ที่ crate `pq-sys` link ด้วยตอน build (อธิบายเหตุผลเต็มที่ในหัวข้อ 72.3) — เครื่องที่ใช้เขียนบทนี้มี `libpq5` (**runtime** library) ติดตั้งอยู่แล้วจาก PostgreSQL server ที่ตั้งไว้ (เหมือน Part 70) แต่**ไม่มี** `libpq-dev` (development package ที่มี unversioned symlink `libpq.so` ที่ linker ต้องการตอน build) และ network policy ของ sandbox ที่เขียนบทนี้ไม่อนุญาตให้ติดตั้ง package ระบบเพิ่มผ่าน `apt` จริง (พยายามแล้วได้ connection timeout ไปยัง Ubuntu archive mirror) — ครั้งแรกที่ลอง `cargo install diesel_cli --no-default-features --features postgres` จึง**พังจริงตอน linking** ด้วย error `rust-lld: error: unable to find library -lpq` ตรงตามที่คาดจากการไม่มี `libpq-dev`

วิธีแก้ที่ใช้จริง (และเป็นวิธีที่ปลอดภัย ไม่ต้องพึ่ง network/apt เลย): สร้าง **symlink เปล่า ๆ** จาก `libpq.so.5` (runtime library ที่มีอยู่แล้ว) ไปเป็น `libpq.so` (ชื่อที่ linker มองหา) ด้วยคำสั่งเดียว `ln -sf libpq.so.5 libpq.so` ในโฟลเดอร์ `/usr/lib/x86_64-linux-gnu/` — หลังจากนั้น `cargo install diesel_cli --no-default-features --features postgres` **compile และ link สำเร็จจริง** (ไม่ต้องพึ่ง header ไฟล์ `libpq-fe.h` เลย เพราะ `pq-sys` ประกาศ FFI signature ของ libpq เองในโค้ด Rust ไม่ได้ใช้ bindgen อ่าน header จริง ๆ ต้องการแค่ตัว `.so` ให้ linker เจอเท่านั้น) — นี่คือหลักฐานที่ยืนยันเนื้อหาหัวข้อ 72.3 ที่อธิบายไว้ว่า Diesel (ต่างจาก SQLx ที่ pure Rust ล้วน) พึ่งพา C library ภายนอกจริง และเป็นภาระที่ทีม DevOps ต้องจัดการเอง — ในสถานการณ์จริงที่มีสิทธิ์ `apt install libpq-dev` เต็มรูปแบบ ขั้นตอนนี้จะง่ายกว่ามาก (ไม่ต้องทำ symlink มือเอง)

หลังแก้ปัญหานี้แล้ว **ทุกคำสั่งและทุกก้อนโค้ดในบทนี้คือผลลัพธ์ที่รันจริงแล้วคัดลอกมา** ไม่มีการ "แต่ง" output ของคำสั่งใดขึ้นเองโดยไม่ระบุที่มา — ถ้าเจอความต่างเล็กน้อยตอนคุณลองรันเอง (เวอร์ชัน `diesel`/`axum` ใหม่กว่าที่เขียนไว้ตรงนี้ หรือ error message ที่ปรับ wording เล็กน้อยตามเวอร์ชัน compiler) ให้ยึดผลลัพธ์จริงในเครื่องคุณเป็นความจริงล่าสุดเสมอ เหมือนหลักการที่ Part 70 วางไว้

## เนื้อหา

### 72.1 ปรัชญาของ Diesel เทียบกับ SQLx: Rust DSL vs Raw SQL String

จาก Part 70 คุณได้เห็นตารางเปรียบเทียบ SQLx/Diesel/SeaORM แบบภาพรวมมาแล้ว บทนี้จะลงรายละเอียดของ Diesel อย่างเต็มที่ โดยเริ่มจากจุดที่ต่างกันมากที่สุดระหว่างสองตัวนี้ก่อน: **วิธีเขียน query**

SQLx (Part 70-71) ให้คุณเขียน **SQL string จริง**:

```rust
// สไตล์ SQLx — เขียน SQL จริงเป็น string literal
sqlx::query_as!(
    Book,
    "SELECT id, isbn, title, author, total_copies, available_copies, published_year, created_at
     FROM books WHERE available_copies > $1 ORDER BY title LIMIT $2",
    0,
    10_i64
)
.fetch_all(pool)
.await?
```

Diesel ให้คุณเขียน **Rust code ที่เป็น method chain** (เรียกว่า **query DSL**) แทน:

```rust
// สไตล์ Diesel — เขียน Rust DSL ที่ generate SQL ให้เอง ไม่มี SQL string ปรากฏเลย
use crate::schema::books::dsl::*;
use diesel::prelude::*;

let results: Vec<Book> = books
    .filter(available_copies.gt(0))
    .order(title.asc())
    .limit(10)
    .select(Book::as_select())
    .load(conn)?;
```

สองก้อนโค้ดนี้ **generate SQL ที่เทียบเท่ากันเป๊ะ** (`SELECT ... FROM books WHERE available_copies > 0 ORDER BY title ASC LIMIT 10` — ตัวเลข `$1`/`$2`/`0`/`10` ต่างกันบ้างตามที่แต่ละฝั่งจัดการ แต่ตรรกะเดียวกันทุกประการ) แต่วิธีที่ "ความถูกต้อง" ถูกตรวจสอบต่างกันโดยสิ้นเชิง — นี่คือประเด็นที่ควรทำความเข้าใจให้ลึกที่สุดในบทนี้ เพราะเป็นสิ่งที่กำหนดทุกอย่างที่ตามมา

#### สองกลไกการตรวจสอบที่ต่างกันโดยพื้นฐาน

**ฝั่ง SQLx**: คุณเขียน SQL ที่เป็น **string literal ล้วน ๆ ในสายตาของ Rust compiler เอง** — compiler มองว่ามันเป็นแค่ `&str` ธรรมดา ไม่รู้อะไรเกี่ยวกับ SQL หรือ schema ของฐานข้อมูลเลย ความปลอดภัยทั้งหมดมาจาก **proc macro ของ `sqlx-macros`** ที่แยก parse string นั้นออกมาต่างหาก แล้วส่งไปตรวจกับฐานข้อมูลจริง (หรือ `.sqlx` cache) ตอน compile time — เท่ากับว่า SQLx ต้อง "จำลอง" ความสามารถของ SQL parser+type checker ขึ้นมาเป็นเลเยอร์แยกที่ทำงาน**คู่กับ**ระบบ type ของ Rust แต่ไม่ใช่ส่วนหนึ่งของมันโดยตรง

**ฝั่ง Diesel**: คุณเขียน `books.filter(available_copies.gt(0))` ซึ่ง**เป็น Rust code จริง ๆ ที่ compiler เข้าใจทุกตัวอักษร** — `books` คือ struct/module ที่ macro `table!` generate ให้ (หัวข้อ 72.4), `available_copies` คือ struct ที่แทนคอลัมน์นั้นโดยเฉพาะ (มี type ที่ผูกกับ SQL type ของคอลัมน์นั้นจริง เช่น `Integer`), `.filter()` คือ trait method ที่รับ argument ที่ implement trait `Expression<SqlType = Bool>` เท่านั้น, `.gt(0)` คือ method ที่สร้าง expression การเปรียบเทียบที่ type-check ตรงกับ type ของคอลัมน์ฝั่งซ้าย — **ทุกจุดต่อกันด้วย trait bound ของ Rust type system เอง ไม่มีเลเยอร์ตรวจสอบพิเศษแยกออกมาเลย** ถ้าคุณเขียนโค้ดที่ type ไม่ตรงกัน (เช่น `available_copies.eq("สิบเล่ม")` ทั้งที่คอลัมน์เป็น `Integer`) มันจะเป็น**ข้อผิดพลาดเรื่อง trait bound ธรรมดา** เหมือนเขียน Rust code ผิดทั่วไป (จะเห็นตัวอย่างจริงในหัวข้อ 72.9)

ผลที่ตามมาที่สำคัญมาก: **Diesel ไม่ต้องมี connection ไปยังฐานข้อมูลตอน compile query DSL เลย** (ต่างจาก SQLx ที่ต้องมี `DATABASE_URL` หรือ `.sqlx` cache ตอน compile เสมอ) เพราะความถูกต้องของ query DSL ไม่ได้พึ่งพาการถามฐานข้อมูลจริง — มันพึ่งพา**ไฟล์ `schema.rs`** ที่ generate ไว้ล่วงหน้าครั้งเดียว (จาก schema จริงตอนนั้น) แล้ว compiler ก็ตรวจกับไฟล์นั้นเหมือนไฟล์ Rust ธรรมดาไฟล์หนึ่ง — นี่คือเหตุผลที่ตารางของ Part 70 เขียนไว้ว่า Diesel ตรวจสอบผ่าน "Rust type system เอง"

#### ตารางสรุปเปรียบเทียบเชิงลึก

| ประเด็น | SQLx (`query!`/`query_as!`) | Diesel (query DSL) |
|---|---|---|
| สิ่งที่คุณเขียน | SQL string จริง (literal) | Rust method chain (DSL) |
| Compiler มองเห็นอะไร | `&str` ธรรมดา (ไม่รู้จัก SQL) | Struct/trait/method จริงทุกตัว |
| กลไกตรวจสอบ | Proc macro แยกไปคุยกับฐานข้อมูลจริง หรือ `.sqlx` cache | Rust trait system ตรวจ generic bound ตามปกติ เทียบกับ `schema.rs` ที่ generate ไว้ |
| ต้องมี DB connection ตอน compile query ไหม | ต้องมี (หรือ `.sqlx` cache ที่สร้างจาก DB จริงมาก่อน) | ไม่ต้อง (ใช้แค่ `schema.rs` ที่เป็นไฟล์ Rust ธรรมดา) |
| ถ้า schema เปลี่ยนไปจริงแต่ยังไม่ sync กับที่ macro/schema.rs รู้ | SQLx macro จะไปเช็คกับ DB จริงใหม่ทุกครั้งที่ build (เจอความเปลี่ยนแปลงอัตโนมัติถ้า DB connection ยังต่อได้) | Diesel จะ **ไม่รู้เลย** ว่า schema จริงเปลี่ยนไปแล้ว จนกว่าจะรัน `diesel print-schema` ใหม่มา sync `schema.rs` เอง — เป็นภาระที่ผู้พัฒนาต้องจำทำเอง |
| ความยืดหยุ่นของ SQL ที่เขียนได้ | สูงมาก — เขียน SQL อะไรก็ได้ที่ PostgreSQL รองรับ | จำกัดตามที่ DSL ของ Diesel รองรับ (SQL ที่ซับซ้อนมาก ๆ บางแบบต้องใช้ `sql_query`/raw SQL หลบ DSL) |
| Error ที่เจอเมื่อเขียนผิด | Error จาก macro บอกตรง ๆ ว่า SQL/DB ไม่ตรง (มักอ่านง่าย) | Error จาก trait bound ของ Rust (มักยาวและซับซ้อนกว่า เพราะเป็น generic trait error ธรรมดา — หัวข้อ 72.9) |

**ข้อสังเกตเชิงลึกที่มักถูกมองข้าม**: หลายคนเข้าใจผิดว่า Diesel "ปลอดภัยกว่า" SQLx เพราะไม่มี SQL string เลย แต่จริง ๆ แล้วทั้งสองฝั่งให้การันตีคนละแบบ — SQLx การันตีว่า **SQL string ที่คุณเขียนตรงกับ schema จริง ณ เวลาที่ build** (เชื่อมกับฐานข้อมูลจริงหรือ cache ที่มาจากฐานข้อมูลจริง) ส่วน Diesel การันตีว่า **โค้ด Rust ที่คุณเขียนตรงกับ `schema.rs` ที่มีอยู่ในโปรเจกต์** ซึ่งไฟล์นั้น**อาจไม่ตรงกับฐานข้อมูลจริง ณ ปัจจุบัน**ก็ได้ ถ้าใครลืมรัน `diesel print-schema` หลัง migration ใหม่ ๆ (Diesel compile ผ่านสนิท แต่พังตอน runtime จริงเมื่อ query ไปโดนคอลัมน์ที่ schema.rs จำผิด) — ทั้งสองแบบจึงมี "จุดอ่อนที่ผู้พัฒนาต้องรับผิดชอบเอง" คนละจุด ไม่มีฝั่งไหนสมบูรณ์แบบ 100%

### 72.2 Diesel เป็น Synchronous by Default: เหตุผลและผลกระทบใน Axum

#### ทำไม Diesel ถึงเป็น synchronous

Diesel เริ่มพัฒนาตั้งแต่ราวปี 2016 — ตอนนั้น `async`/`await` ยังไม่ได้ stabilize เข้า Rust เลย (stabilize จริงปี 2019) การออกแบบ core ของ Diesel ทั้งหมด (trait `Connection`, query builder, transaction API) จึงเป็น**synchronous ล้วน ๆ ตั้งแต่โครงสร้างพื้นฐาน**: `conn.execute(...)` หรือ `.load::<T>(conn)` เป็น function ธรรมดาที่ **block thread ปัจจุบันจนกว่า query จะเสร็จ** ไม่คืน `Future` ให้ await เลย — ต่างจาก SQLx ที่ทุก method คืน `Future` (async-native ตั้งแต่ต้นเพราะเริ่มพัฒนาทีหลัง ในยุคที่ async Rust เริ่มเป็นมาตรฐานแล้ว)

การที่ Diesel ไม่ได้ "ตกยุค" — มันแค่**ออกแบบมาก่อน async Rust จะแพร่หลาย** และการเปลี่ยน core ทั้งหมดของ crate ที่คนใช้งานจริงจำนวนมากอยู่แล้วเป็น async-native ล้วน ๆ เป็นงานที่ทำลาย backward compatibility อย่างหนัก ทีม Diesel จึงเลือกคงแกนหลักไว้เป็น sync (เสถียร ผ่านการทดสอบมานาน) แล้วเพิ่ม `diesel-async` (หัวข้อ 72.2.3) เป็น crate แยกสำหรับคนที่ต้องการ async จริง ๆ ในภายหลัง

#### ผลกระทบตรงในเว็บแอป async: ห้าม block runtime

จาก Part 48 คุณรู้แล้วว่า Tokio ใช้ **thread pool ขนาดจำกัด** (ปกติเท่าจำนวน CPU core) รัน task async หลายพันตัวพร้อมกันด้วยการ**สลับ (poll)** ไปมาระหว่าง task ที่ `.await` แล้วยังไม่พร้อม — กลไกนี้ใช้ได้ผลก็ต่อเมื่อโค้ดใน task นั้น **ไม่ block thread จริง ๆ ตลอดเวลาที่รอ I/O** ถ้าโค้ดใน `async fn` เรียก function ที่ block thread ตรง ๆ (ไม่ผ่าน `.await` แบบ cooperative) thread นั้นจะ**หยุดสนิท**ไม่สามารถไป poll task อื่นได้เลยจนกว่า function ที่ block จะคืนค่า — นี่คือสิ่งที่ Part 48 เตือนไว้ตรง ๆ ว่า "ห้ามเรียก blocking code ใน async context โดยตรง"

Diesel query **คือ blocking code แบบเต็มรูปแบบ** — `.load::<T>(conn)` คือ function ธรรมดาที่ทำ network I/O (ส่ง SQL ไปยัง PostgreSQL รอผลลัพธ์กลับมา) แบบ **synchronous** (block thread จริงตลอดเวลาที่รอ) ถ้าคุณเผลอเรียกมันตรง ๆ ในบล็อก `async fn` ของ Axum handler:

```rust
// ตัวอย่าง "ผิด" ที่ห้ามทำ — เรียก Diesel sync query ตรงในบล็อก async
async fn list_books_wrong(
    State(pool): State<DbPool>, // r2d2 pool ของ Diesel (หัวข้อ 72.10)
) -> Result<Json<Vec<Book>>, AppError> {
    use crate::schema::books::dsl::*;

    let mut conn = pool.get().map_err(AppError::from)?;
    // อันตราย: .load() เป็น synchronous call — block Tokio worker thread ทั้งตัว
    // ตราบใดที่ query นี้ยังไม่เสร็จ ไม่มีใครมาแอบ ".await" ให้ scheduler สลับงานได้เลย
    let all_books = books.load::<Book>(&mut conn).map_err(AppError::from)?;
    Ok(Json(all_books))
}
```

โค้ดนี้ **compile ผ่านสนิท** (ไม่มี error ใด ๆ — Rust compiler ไม่รู้เรื่อง "async runtime starvation" มันเป็นปัญหาระดับ runtime behavior ไม่ใช่ type error) และในการทดสอบเบา ๆ (request เดียว ที่ query เร็ว) มันจะ "ดูเหมือนทำงานได้" ด้วยซ้ำ — นี่คือจุดที่อันตรายที่สุด เพราะบั๊กนี้**ไม่โผล่ตอน dev ทดสอบเบา ๆ** แต่จะโผล่ตอน production มี concurrent request จำนวนมาก: ทุก request ที่เรียก handler นี้จะ**ยึด Tokio worker thread ไว้ทั้งตัว**ตลอดเวลาที่ query กำลังรอผลจาก PostgreSQL (อาจหลาย millisecond ถึงหลายร้อย millisecond ถ้า query ซับซ้อนหรือ network ช้า) — ถ้า Tokio runtime มี worker thread แค่ 4 ตัว (เท่า CPU core ปกติ) และมี 4 request ที่เรียก handler นี้พร้อมกัน **worker thread ทั้ง 4 ตัวจะถูกยึดหมด** ทำให้ request อื่นทั้งหมดในระบบ (แม้จะเป็น endpoint อื่นที่ไม่เกี่ยวกับ database เลย เช่น health check ธรรมดา) **ต้องรอ**เพราะไม่มี thread ว่างให้ scheduler ไป poll ให้ — นี่คือ **runtime starvation** ที่ Part 48 เตือนไว้ ในรูปแบบที่เป็นรูปธรรมที่สุด

#### วิธีที่ถูกต้อง: `tokio::task::spawn_blocking`

Part 48 สอนวิธีแก้ไว้แล้ว: ย้ายงานที่ block thread ไปรันบน **thread pool แยก** ที่ Tokio จัดไว้เฉพาะสำหรับงาน blocking (`spawn_blocking`) ซึ่งไม่ใช่ thread pool เดียวกับที่ scheduler ใช้ poll async task ปกติ — thread pool นี้ **ขยายขนาดได้ตามต้องการ** (ไม่จำกัดเท่าจำนวน CPU core เหมือน worker thread ของ async runtime) เพราะ thread ในนี้คาดหวังว่าจะ block เป็นปกติอยู่แล้ว:

```rust
use axum::{extract::State, Json};
use diesel::prelude::*;

async fn list_books_correct(
    State(pool): State<DbPool>,
) -> Result<Json<Vec<Book>>, AppError> {
    // ย้าย query ทั้งก้อน (ที่เป็น synchronous ล้วน ๆ) ไปรันบน blocking thread pool ของ Tokio
    // closure ด้านในนี้รันบน thread แยก — block ได้เต็มที่โดยไม่กระทบ worker thread ของ async scheduler
    let all_books = tokio::task::spawn_blocking(move || {
        use crate::schema::books::dsl::*;
        let mut conn = pool.get()?; // ยืม connection จาก r2d2 pool (หัวข้อ 72.10) — เป็น blocking call ด้วย แต่ก็อยู่ใน closure นี้แล้ว
        books.load::<Book>(&mut conn)
    })
    .await // .await ตัวนี้ต่างจากการ .await query โดยตรง — มันแค่รอ "งานบน thread อื่นเสร็จ" ไม่ block worker thread ของ async scheduler เลย
    .map_err(|_| AppError::Internal("spawn_blocking task panicked".into()))? // JoinError จาก task ที่ panic
    .map_err(AppError::from)?; // diesel::result::Error จาก query จริง

    Ok(Json(all_books))
}
```

**อธิบายทีละส่วนว่าทำไมสิ่งนี้ถึงแก้ปัญหาได้จริง**:

1. `tokio::task::spawn_blocking(move || { ... })` รับ closure ที่เป็น synchronous ล้วน ๆ (ไม่มี `.await` ข้างในเลยสังเกตดี ๆ — ทุกอย่างในนี้คือ blocking code ปกติ) แล้ว**ส่งไปรันบน thread ของ blocking pool แยก** ที่ Tokio จัดไว้ ไม่แตะ worker thread ของ async scheduler เลยแม้แต่นิดเดียว
2. `spawn_blocking` เองคืน `JoinHandle<T>` ที่เป็น `Future` — `.await` ตัวนี้**ไม่ได้ block thread ปัจจุบัน** มันแค่บอก scheduler ว่า "task นี้ยังไม่เสร็จ ไปทำงานอื่นก่อนได้เลย แล้วกลับมา poll ทีหลังตอนที่ thread ฝั่ง blocking pool ทำงานเสร็จแล้วส่งสัญญาณกลับมา" — เป็นการ `.await` แบบ cooperative ตามปกติของ async Rust ทุกประการ (Part 46-48)
3. ผลลัพธ์ที่ได้กลับมาคือ `Result<Result<Vec<Book>, diesel::result::Error>, tokio::task::JoinError>` — ชั้นนอกคือ error จาก `spawn_blocking` เอง (ปกติเกิดถ้า closure panic) ชั้นในคือ error จาก query Diesel จริง ต้อง `?`/`.map_err()` สองชั้นตามลำดับ

**ผลลัพธ์**: ต่อให้มี 1,000 request พร้อมกันเรียก handler นี้ worker thread ของ async scheduler (Tokio) จะ**ยังว่างอยู่เสมอ** พร้อมรับ/ตอบ request อื่น ๆ ในระบบต่อไปตามปกติ ส่วนงาน query จริงที่ block thread จะไปกองรอกันอยู่ที่ blocking thread pool แยกต่างหาก (ที่ปรับขนาดเองได้ตามโหลด แม้จะไม่ใช่ไม่จำกัด — ถ้า blocking pool เองเต็มก็ต้องรอเช่นกัน แต่ปัญหานี้แยกกันชัดเจนจากปัญหา worker thread หมด)

#### `diesel-async`: ทางเลือก async-native (ระดับรู้จัก)

ทีม Diesel เองมี crate แยกชื่อ `diesel-async` ที่ทำ Diesel ให้เป็น async-native จริง (คืน `Future` ตรง ๆ เหมือน SQLx โดยไม่ต้องพึ่ง `spawn_blocking` เลย) รองรับ Tokio ผ่าน connection type อย่าง `diesel_async::AsyncPgConnection` — แต่ crate นี้**ไม่ใช่ core ของ Diesel** เป็น layer เพิ่มเติมที่ยังต้องพึ่ง query DSL/schema/model เดียวกันกับ Diesel หลัก (แนวคิดและวิธีเขียน query DSL เหมือนกันทุกอย่างที่บทนี้สอน เปลี่ยนแค่ตัว connection/execution ให้เป็น async) บทนี้จะไม่ลงรายละเอียดของ `diesel-async` แบบเต็มรูปแบบ (เพราะไม่ใช่ค่า default ของ ecosystem Diesel และยังมีความ mature/adoption ต่างจาก core Diesel พอสมควร) แต่ควรรู้ไว้ว่า**มีทางเลือกนี้อยู่** ถ้าโปรเจกต์จริงต้องการ query DSL แบบ Diesel เต็มรูปแบบ **และ** async-native จริงในเวลาเดียวกัน โดยไม่ต้องผ่าน `spawn_blocking`

### 72.3 ติดตั้งและตั้งค่าโปรเจกต์ Diesel

#### เพิ่ม `diesel` เข้าโปรเจกต์

```bash
cargo add diesel --features postgres,chrono,r2d2
```

ผลลัพธ์จริง (ทดสอบในเครื่องผู้เขียนบทนี้ — เวอร์ชันปัจจุบันบน crates.io ณ วันที่เขียนคือ `2.3.13`):

```
      Adding diesel v2.3.13 to dependencies
             Features:
             + 32-column-tables
             + chrono
             + postgres
             + postgres_backend
             + r2d2
             + with-deprecated
             - 128-column-tables
             - 64-column-tables
             - __with_asan_tests
             - extras
             - huge-tables
             - i-implement-a-third-party-backend-and-opt-into-breaking-changes
             - ipnet-address
             - large-tables
             - mysql
             - mysql_backend
             - mysqlclient-src
             - network-address
             - numeric
             - pq-src
             - quickcheck
             - returning_clauses_for_sqlite_3_35
             - serde_json
             - sqlite
             - time
             - unstable
             - uuid
             - without-deprecated
```

**อธิบาย feature ที่เปิดแต่ละตัว:**

- **`postgres`** — เปิด backend สำหรับ PostgreSQL โดยเฉพาะ (Diesel รองรับ MySQL, SQLite ผ่าน feature แยก `mysql`/`sqlite` เหมือนหลักการเดียวกับ SQLx ใน Part 70 — เปิดเฉพาะตัวที่ใช้จริง)
- **`chrono`** — เปิดการแปลงระหว่าง PostgreSQL type `TIMESTAMPTZ` กับ `chrono::DateTime<Utc>` เหมือนที่ Part 70 อธิบายไว้กับ SQLx (มี feature ชื่อเดียวกันพอดี ทำงานแนวคิดเดียวกัน)
- **`r2d2`** — เปิด integration กับ crate `r2d2` สำหรับทำ connection pooling แบบ synchronous (หัวข้อ 72.10) — Diesel เองไม่มี pool ในตัวแบบที่ SQLx มี `PgPoolOptions` มาให้ในตัว ต้องพึ่ง crate ภายนอกอย่าง `r2d2` เสมอ
- **`32-column-tables`** — Diesel ต้อง generate trait implementation สำหรับ tuple/query ที่มีจำนวนคอลัมน์ต่าง ๆ ไว้ล่วงหน้า (ข้อจำกัดจาก generic ของ Rust ที่ไม่มี variadic tuple) ค่า default รองรับสูงสุด 32 คอลัมน์ต่อ query — ถ้าตารางไหนกว้างเกิน 32 คอลัมน์ต้องเปิด `64-column-tables`/`128-column-tables` เพิ่ม (แลกกับเวลา compile ที่นานขึ้นมาก) โดเมนของบทนี้ (`books`/`shelves`/`borrow_records`) ไม่มีตารางไหนเกิน 32 คอลัมน์เลยจึงไม่ต้องสนใจ feature นี้เพิ่ม
- **`pq-src`** (อยู่ในรายการ deactivated ด้านบน สังเกตเครื่องหมาย `-`) — feature ทางเลือกที่ให้ Diesel **compile libpq จาก source ไปในตัว** แทนพึ่ง libpq ที่ติดตั้งในระบบผ่าน `pq-sys`/`pg_config` — มีประโยชน์มากในสถานการณ์ที่เครื่อง build ไม่มี PostgreSQL development library ติดตั้งไว้เต็มรูปแบบ (เช่น CI ที่ minimal image หรือสถานการณ์แบบที่ผู้เขียนบทนี้เจอจริงตามที่อธิบายไว้ในหมายเหตุตรวจสอบเนื้อหาด้านบน) แลกกับเวลา compile ที่นานขึ้นมาก บทนี้**ไม่ได้เปิด** feature นี้ (เลือกแก้ปัญหาด้วยการสร้าง symlink `libpq.so` แทน) แต่ถ้าเครื่องคุณไม่มีสิทธิ์สร้าง symlink ระดับระบบเลย ให้ลองเปิด feature นี้เพิ่มแทน (`cargo add diesel --features postgres,chrono,r2d2,pq-src`)

`Cargo.toml` ที่ได้:

```toml
[dependencies]
diesel = { version = "2.3.13", features = ["postgres", "chrono", "r2d2"] }
```

พร้อมกับ dependency พื้นฐานอื่น ๆ ที่ต้องมีควบคู่กันตามที่ Part 48/57/62/70 สอนไว้แล้ว:

```bash
cargo add tokio --features full
cargo add axum
cargo add serde --features derive
cargo add serde_json
cargo add chrono --features serde
cargo add dotenvy   # อ่านไฟล์ .env ตอน runtime (ธรรมเนียมเดียวกับ Part 70)
```

**ข้อสังเกตสำคัญเรื่อง `pq-sys`**: ต่างจาก SQLx ที่เขียน PostgreSQL wire protocol เองล้วน ๆ ใน Rust (ไม่พึ่ง C library ใด ๆ เลย) Diesel **พึ่ง `libpq`** (C client library ตัวเดียวกับที่ `psql` ใช้) ผ่าน crate `pq-sys` ที่ link แบบ FFI (Foreign Function Interface) — นี่คือความต่างเชิง dependency ที่สำคัญมากในทางปฏิบัติ: โปรเจกต์ SQLx ทั้งโปรเจกต์เป็น "pure Rust" (ไม่มี C toolchain เกี่ยวข้องเลย compile บน environment ไหนก็ได้ที่มี Rust toolchain) ส่วนโปรเจกต์ Diesel (ที่ไม่เปิด `pq-src`) ต้องมี `libpq` development library (`pg_config`, header files, `.so`/`.a` สำหรับ link) ติดตั้งอยู่ในเครื่อง build เสมอ — เป็นภาระเพิ่มเติมที่ทีม DevOps ต้องจัดการ (เช่น ต้อง `apt install libpq-dev` ในทุก image ที่ build โปรเจกต์นี้) ถ้าไม่อยากพึ่ง C library เลย feature `pq-src`/`postgres` ที่ compile จาก source ก็ช่วยได้แต่แลกกับเวลา build ที่นานขึ้นมาก

#### ติดตั้ง `diesel_cli`

```bash
cargo install diesel_cli --no-default-features --features postgres
```

ผลลัพธ์จริง (ทดสอบในเครื่องผู้เขียนบทนี้ หลังแก้ปัญหา `libpq.so` ตามที่อธิบายไว้ในหมายเหตุตรวจสอบเนื้อหาด้านบน):

```
   Compiling diesel_cli v2.3.13
    Finished `release` profile [optimized] target(s) in 52.15s
  Installing /root/.cargo/bin/diesel
   Installed package `diesel_cli v2.3.13` (executable `diesel`)
```

**อธิบาย flag**: เหมือนกับ `sqlx-cli` ใน Part 70 — `--no-default-features --features postgres` จำกัดให้ compile เฉพาะ backend PostgreSQL (ค่า default ของ `diesel_cli` พยายาม compile รองรับทั้ง MySQL, SQLite, PostgreSQL พร้อมกัน ซึ่งต้องมี development library ของทั้งสามตัวติดตั้งไว้ — ส่วนใหญ่ทำไม่สำเร็จตรง ๆ ถ้าไม่มี MySQL/SQLite dev library ในเครื่อง จึงจำกัด feature เหลือแค่ตัวที่ใช้จริงเสมอ)

**เทียบกับ `sqlx-cli`**: ทั้งสอง CLI นี้ทำหน้าที่คล้ายกันในภาพรวม (จัดการ migration, เชื่อมกับฐานข้อมูลจริง) แต่ `diesel_cli` มีหน้าที่เพิ่มที่ `sqlx-cli` ไม่มี คือการ **generate `schema.rs`** (หัวข้อ 72.4) ซึ่งเป็นผลจากปรัชญาที่ต่างกัน (SQLx ไม่ต้องมีไฟล์ schema กลางแบบนี้เพราะ macro คุยกับฐานข้อมูลจริงตรง ๆ ทุกครั้งที่ build)

#### `DATABASE_URL` และไฟล์ `.env`

Diesel ใช้ธรรมเนียมเดียวกับ SQLx เรื่อง `DATABASE_URL` แต่ `diesel_cli` (ต่างจาก `sqlx-cli`) **มี integration กับไฟล์ `.env` ในตัวเลย** (ผ่าน crate `dotenvy` ที่ผูกไว้ในตัว CLI) ไม่ต้อง export environment variable เองก่อนเรียกทุกคำสั่งแบบ SQLx:

```bash
echo 'DATABASE_URL=postgres://postgres:postgres@127.0.0.1:5432/diesel_course_scratch' > .env
```

#### `diesel setup`

```bash
diesel setup
```

คำสั่งนี้ทำสามอย่างพร้อมกัน: (1) อ่าน `DATABASE_URL` จากไฟล์ `.env` (2) **สร้างฐานข้อมูลนั้นให้เลย** ถ้ายังไม่มีอยู่จริง (ต่างจาก `sqlx-cli` ที่ต้องสร้างฐานข้อมูลเองก่อนด้วย `sqlx database create`) และ (3) สร้างโฟลเดอร์ `migrations/` พร้อมสร้างตาราง `__diesel_schema_migrations` (เทียบเท่า `_sqlx_migrations` ของ SQLx) สำหรับติดตาม migration ที่รันไปแล้ว

**ผลลัพธ์จริง** (ทดสอบในเครื่องผู้เขียนบทนี้):

```
$ diesel setup
Creating migrations directory at: /tmp/.../diesel_verify/migrations
Creating database: diesel_course_scratch
```

(หมายเหตุ: ฐานข้อมูลชื่อนี้ถูก `DROP`/`CREATE` ใหม่ไว้ก่อนแล้วเพื่อทดสอบให้เห็นขั้นตอนสร้างจริงตั้งแต่ต้น — ในสถานการณ์จริงถ้าฐานข้อมูลมีอยู่แล้ว `diesel setup` จะข้ามขั้นตอนสร้างฐานข้อมูลไปเฉย ๆ)

#### `diesel migration generate`: สร้าง migration ใหม่

```bash
diesel migration generate create_library_tables
```

ผลลัพธ์จริง (ทดสอบในเครื่องผู้เขียนบทนี้):

```
Creating migrations/2026-09-27-004201-0000_create_library_tables/up.sql
Creating migrations/2026-09-27-004201-0000_create_library_tables/down.sql
```

**เทียบกับ `sqlx migrate add`**: ความต่างที่เห็นชัดที่สุดคือ **โครงสร้างไฟล์** — `sqlx-cli` สร้างไฟล์แบบ `<timestamp>_<name>.up.sql`/`<timestamp>_<name>.down.sql` เป็นไฟล์เดี่ยวสองไฟล์ในโฟลเดอร์ `migrations/` เดียวกันหมด ส่วน `diesel_cli` สร้าง**โฟลเดอร์แยกต่อ migration** (`migrations/<timestamp>_<name>/up.sql` และ `.../down.sql`) — ทั้งสองแบบทำหน้าที่เดียวกัน (ไฟล์ขึ้นต้นด้วย timestamp เพื่อการันตีลำดับเวลาเหมือนกัน) เป็นแค่ธรรมเนียมการจัดไฟล์ที่ต่างกันของแต่ละ tool เท่านั้น ไม่มีนัยเชิงความหมายอื่น (Diesel เปิด default เป็น reversible เสมอ มีทั้ง `up.sql`/`down.sql` ทุกครั้งโดยไม่ต้องใส่ flag พิเศษแบบ `-r` ของ `sqlx migrate add`)

#### เขียน SQL migration: ตาราง `books`, `shelves`, `borrow_records`

บทนี้ต่อยอดโดเมนห้องสมุดจาก Part 70 แต่เพิ่มตาราง **`shelves`** ขึ้นมาใหม่ (หนึ่งชั้นวางมีหนังสือได้หลายเล่ม — one-to-many ที่หัวข้อ 72.8 จะใช้สอน join) `migrations/2026-09-27-004201-0000_create_library_tables/up.sql`:

```sql
-- ชั้นวางหนังสือในห้องสมุด (ตารางใหม่สำหรับบทนี้ — ใช้สอนความสัมพันธ์ one-to-many กับ books)
CREATE TABLE shelves (
    id BIGSERIAL PRIMARY KEY,
    code TEXT NOT NULL UNIQUE,       -- เช่น "A1", "B3"
    location TEXT NOT NULL
);

-- ตาราง books เหมือน Part 70 แต่เพิ่ม shelf_id ที่อ้างถึง shelves
CREATE TABLE books (
    id BIGSERIAL PRIMARY KEY,
    isbn TEXT NOT NULL UNIQUE,
    title TEXT NOT NULL,
    author TEXT NOT NULL,
    total_copies INTEGER NOT NULL CHECK (total_copies >= 0),
    available_copies INTEGER NOT NULL CHECK (available_copies >= 0),
    published_year INTEGER,
    shelf_id BIGINT REFERENCES shelves(id), -- nullable: หนังสืออาจยังไม่ถูกจัดวางที่ชั้นไหนเลยก็ได้
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

`down.sql`:

```sql
DROP TABLE IF EXISTS borrow_records;
DROP TABLE IF EXISTS books;
DROP TABLE IF EXISTS shelves;
```

(สังเกตลำดับ `DROP TABLE` ใน `down.sql`: ต้อง drop ตารางที่มี foreign key อ้างถึงตารางอื่นก่อนเสมอ — `borrow_records` อ้างถึง `books` จึง drop ก่อน `books` และ `books` อ้างถึง `shelves` จึง drop ก่อน `shelves` เช่นกัน หลักการเดียวกับที่ Part 70 อธิบายไว้เรื่อง foreign key constraint)

#### รัน migration

```bash
diesel migration run
```

ผลลัพธ์จริง (ทดสอบในเครื่องผู้เขียนบทนี้):

```
$ diesel migration run
Running migration 2026-09-27-004201-0000_create_library_tables
```

ตรวจสอบด้วย `psql` จริง (`\dt`) ยืนยันว่าทั้งสามตารางถูกสร้างจริงพร้อม `__diesel_schema_migrations` ที่ Diesel สร้างขึ้นเองสำหรับติดตามสถานะ migration

**คำสั่งอื่นที่ควรรู้จัก (เทียบกับ `sqlx-cli`):**

| คำสั่ง | ความหมาย | เทียบกับ SQLx |
|---|---|---|
| `diesel migration run` | รัน migration ที่ยังไม่ apply ทั้งหมด | เทียบเท่า `sqlx migrate run` |
| `diesel migration revert` | ย้อนกลับ migration ล่าสุด (ใช้ `down.sql`) | เทียบเท่า `sqlx migrate revert` |
| `diesel migration redo` | revert แล้ว run migration ล่าสุดใหม่ทันที (ลัดขั้นตอนตอน dev) | ไม่มีคำสั่งเดี่ยวเทียบเท่าตรง ๆ ใน `sqlx-cli` (ต้องรัน revert แล้ว run สองคำสั่ง) |
| `diesel migration list` | ดูสถานะ migration ทั้งหมด | เทียบเท่า `sqlx migrate info` |
| `diesel print-schema` | generate `schema.rs` จาก schema จริง | **ไม่มีเทียบเท่าใน SQLx เลย** — เป็นขั้นตอนที่มีเฉพาะ workflow ของ Diesel เพราะ SQLx ไม่ต้องมีไฟล์ schema กลาง |

### 72.4 `schema.rs`: Single Source of Truth ที่ Query DSL ต้อง Type-Check ผ่าน

#### ทำไม Diesel ต้องมี `schema.rs`

จากหัวข้อ 72.1 คุณรู้แล้วว่า Diesel ตรวจสอบ query DSL ผ่าน Rust type system เอง โดยไม่ต้องมี connection ไปยังฐานข้อมูลจริงตอน compile — แต่ trait bound ของ Rust ต้องมี**อะไรบางอย่างที่เป็น Rust type จริง** ให้ตรวจกับ ไม่ใช่ตรวจกับฐานข้อมูลลอย ๆ — สิ่งนั้นคือ `schema.rs` ซึ่งเป็นไฟล์ Rust ธรรมดาที่ประกอบด้วย macro `table!` (Diesel เวอร์ชันใหม่ใช้ `diesel::table!`) ที่นิยามโครงสร้างของแต่ละตารางเป็น Rust type

`schema.rs` ไฟล์นี้เขียนขึ้นมาเองได้ แต่ในทางปฏิบัติ **ควร generate จากฐานข้อมูลจริงเสมอ** ผ่านคำสั่ง `diesel print-schema` (หรือให้ `diesel_cli` เขียนไฟล์นี้อัตโนมัติทุกครั้งหลัง `diesel migration run` — พฤติกรรม default ของ `diesel_cli` คือทำแบบนี้อยู่แล้ว ผ่าน config ไฟล์ `diesel.toml`)

#### รัน `diesel print-schema` จริง

```bash
diesel print-schema
```

ผลลัพธ์จริง (ทดสอบในเครื่องผู้เขียนบทนี้ ต่อฐานข้อมูล `diesel_course_scratch` ที่มีตาราง `books`/`shelves`/`borrow_records` ตามหัวข้อ 72.3):

```rust
// @generated automatically by Diesel CLI.

diesel::table! {
    books (id) {
        id -> Int8,
        isbn -> Text,
        title -> Text,
        author -> Text,
        total_copies -> Int4,
        available_copies -> Int4,
        published_year -> Nullable<Int4>,
        shelf_id -> Nullable<Int8>,
        created_at -> Timestamptz,
    }
}

diesel::table! {
    borrow_records (id) {
        id -> Int8,
        book_id -> Int8,
        borrower_name -> Text,
        borrowed_at -> Timestamptz,
        returned_at -> Nullable<Timestamptz>,
    }
}

diesel::table! {
    shelves (id) {
        id -> Int8,
        code -> Text,
        location -> Text,
    }
}

diesel::joinable!(books -> shelves (shelf_id));
diesel::joinable!(borrow_records -> books (book_id));

diesel::allow_tables_to_appear_in_same_query!(books, borrow_records, shelves,);
```

**อธิบาย macro `table!` ทีละส่วน:**

- `diesel::table! { books (id) { ... } }` — ประกาศว่ามีตาราง `books` ที่ primary key คือ `id` เนื้อหาข้างในคือรายการคอลัมน์ทั้งหมดพร้อม **SQL type ของ Diesel** (ไม่ใช่ Rust type ตรง ๆ) — `Int8` คือ SQL type ที่แทน `BIGINT` (สอดคล้องกับ `BIGSERIAL` ที่แท้จริงคือ `BIGINT` + sequence), `Text` แทน `TEXT`, `Nullable<Int4>` แทน `INTEGER` ที่ nullable — สังเกตว่า `Nullable<T>` เป็น type wrapper ระดับ SQL type เอง (คู่กับ `Option<T>` ระดับ Rust type ที่จะมาผูกกันตอนนิยาม struct ในหัวข้อ 72.5)
- macro นี้ generate module ชื่อ `books` ที่มี struct ย่อยแทนแต่ละคอลัมน์ (เช่น struct `id`, `title`, `available_copies`) พร้อม `impl` ที่ผูก type แต่ละคอลัมน์กับ SQL type ที่ระบุไว้ — struct เหล่านี้แหละที่หัวข้อ 72.1 พูดถึงว่าเป็น "ตัวแทนคอลัมน์แต่ละตัวที่ Rust compiler ตรวจสอบได้จริง" ไม่ใช่ string ชื่อคอลัมน์ธรรมดา
- `diesel::joinable!(books -> shelves (shelf_id))` — macro ที่**บอก Diesel ว่าตาราง `books` join กับ `shelves` ได้ผ่านคอลัมน์ `shelf_id`** (แนวคิดเดียวกับ foreign key แต่ประกาศไว้เป็น Rust code ที่ query DSL ใช้ตรวจสอบ join ได้ตอน compile — หัวข้อ 72.8 จะใช้ macro นี้ทำ `.inner_join()`) — สังเกตว่า `diesel print-schema` **generate macro นี้ให้อัตโนมัติ**จาก foreign key constraint ที่ตรวจพบจริงในฐานข้อมูล (บรรทัดที่สอง `diesel::joinable!(borrow_records -> books (book_id))` ก็มาจากกลไกเดียวกัน อ่านจาก FK ของ `borrow_records.book_id` ที่ migration สร้างไว้ — Diesel เวอร์ชันปัจจุบันจึง infer ความสัมพันธ์นี้ให้เองโดยไม่ต้องเขียนมือ ตราบใดที่ foreign key ถูกประกาศไว้ถูกต้องในระดับฐานข้อมูลจริงตั้งแต่ migration)
- `diesel::allow_tables_to_appear_in_same_query!(...)` — บอก Diesel ว่าตารางกลุ่มนี้ **อนุญาตให้ปรากฏร่วมกันใน query เดียวกันได้** (จำเป็นสำหรับ join ข้ามตาราง) — เป็น safety mechanism ระดับ compile-time อีกชั้นที่ป้องกันการเขียน query ข้าม schema/tenant กันโดยไม่ตั้งใจในระบบที่ซับซ้อนกว่านี้ (multi-schema database)

#### `schema.rs` คือ "แคช" ของ schema จริง ไม่ใช่ source of truth ที่แท้จริง

จุดที่สำคัญมากและเชื่อมกลับไปหัวข้อ 72.1: **`schema.rs` เป็นแค่สิ่งที่ generate มาจากฐานข้อมูลจริง ณ เวลาหนึ่ง** — source of truth ที่แท้จริงคือฐานข้อมูลเสมอ (เหมือนกับที่ migration `.sql` file คือ source of truth ของ schema ทั้งฝั่ง SQLx และ Diesel) `schema.rs` เป็นแค่ "ภาพสะท้อน" ของ schema นั้นในรูปแบบที่ Rust type system เข้าใจได้ — ถ้าคุณรัน migration ใหม่ที่เปลี่ยน schema (เพิ่ม/ลบคอลัมน์) แล้ว **ลืม** รัน `diesel print-schema` ใหม่ `schema.rs` จะยัง "จำ" schema เก่าอยู่ — โค้ด Rust ที่เขียนอ้างคอลัมน์ใหม่จะ compile ไม่ผ่าน (เพราะ `schema.rs` ไม่มีคอลัมน์นั้น) หรือแย่กว่านั้นคือถ้าลบคอลัมน์ไปแล้วแต่ `schema.rs` เก่ายังมีคอลัมน์นั้นอยู่ query ที่ compile ผ่าน (เพราะดูจาก `schema.rs` เก่า) จะ **พังตอน runtime จริง** เมื่อไปคุยกับฐานข้อมูลที่ไม่มีคอลัมน์นั้นแล้ว — นี่คือความเสี่ยงที่บทนี้เตือนไว้แล้วในหัวข้อ 72.1 ตอนเทียบกับ SQLx (ที่ macro คุยกับฐานข้อมูลจริงทุกครั้งที่ build จึงไม่มีปัญหานี้)

**แนวปฏิบัติที่ดี**: ตั้งกฎในทีมว่า "ทุกครั้งที่รัน migration ใหม่ ต้องรัน `diesel print-schema > src/schema.rs` ทันที (หรือปล่อยให้ `diesel migration run` ทำให้อัตโนมัติผ่าน config `print_schema.file` ใน `diesel.toml` ที่ `diesel setup` สร้างให้ตั้งแต่ต้น)" — ไฟล์ `diesel.toml` ที่ได้จาก `diesel setup`:

```toml
[print_schema]
file = "src/schema.rs"
```

การตั้งค่านี้ทำให้ `diesel migration run` **เขียน `schema.rs` ให้อัตโนมัติทุกครั้งที่รัน migration สำเร็จ** — ลดความเสี่ยงเรื่อง schema.rs ไม่ sync ไปได้มาก (แต่ก็ยังไม่ 100% ถ้ามีใครแก้ฐานข้อมูลตรง ๆ ผ่าน `psql` โดยไม่ผ่าน migration ระบบนี้ก็จะไม่รู้เช่นกัน)

### 72.5 Models: `Queryable`/`Selectable` สำหรับอ่าน vs `Insertable` สำหรับเขียน

#### struct สำหรับ**อ่าน**ข้อมูล: `#[derive(Queryable, Selectable)]`

```rust
use chrono::{DateTime, Utc};
use diesel::prelude::*;
use serde::Serialize;

// struct ที่แทนแถวข้อมูลที่ "อ่าน" ออกมาจากตาราง books ทั้งแถว
// derive สองชุดพร้อมกัน: Diesel (Queryable/Selectable) กับ serde (Serialize)
#[derive(Debug, Serialize, Queryable, Selectable)]
#[diesel(table_name = crate::schema::books)] // บอก Diesel ว่า struct นี้ผูกกับตาราง books ใน schema.rs
#[diesel(check_for_backend(diesel::pg::Pg))]  // ตรวจสอบ type ทุกตัวให้ตรงกับ backend PostgreSQL ตอน compile (เปิดไว้เสมอแนะนำ)
struct Book {
    id: i64,
    isbn: String,
    title: String,
    author: String,
    total_copies: i32,
    available_copies: i32,
    published_year: Option<i32>, // Nullable<Int4> ใน schema.rs -> Option<i32> ใน Rust
    shelf_id: Option<i64>,       // Nullable<Int8> -> Option<i64>
    created_at: DateTime<Utc>,
}
```

**อธิบายแต่ละ derive/attribute:**

- **`Queryable`** — derive macro ที่ generate `impl Queryable<...>` ให้ struct นี้ ทำหน้าที่แปลง**แถวข้อมูลดิบจาก database driver** (ลำดับ column ตามที่ query ส่งมา) เป็น struct — ทำงานคล้าย `sqlx::FromRow` ที่ Part 70 สอน แต่กลไกภายในต่างกัน (Diesel ผูกกับ deserialize ตาม**ลำดับ**ของคอลัมน์ในผลลัพธ์ query ไม่ใช่ผูกด้วยชื่อ field เหมือน `FromRow` ของ SQLx) — นี่คือเหตุผลที่**ลำดับ field ใน struct ต้องตรงกับลำดับคอลัมน์ที่ query คืนมาเป๊ะ** (ปกติคือลำดับเดียวกับที่ประกาศใน `table!` macro ถ้า `.select()` ทั้งตาราง)
- **`Selectable`** — derive macro ใหม่กว่าที่เพิ่มเข้ามาใน Diesel 2.x ทำหน้าที่ generate helper method `Book::as_select()` ที่ให้คุณระบุ **ชัดเจน**ว่าต้องการ select เฉพาะคอลัมน์ที่ struct นี้มี (ไม่ใช่ `SELECT *` ทั้งตาราง) — ช่วยแก้ปัญหาการนับลำดับคอลัมน์ผิดของ `Queryable` เดี่ยว ๆ ได้มาก (จะเห็นการใช้งานจริงในหัวข้อ 72.6)
- **`#[diesel(table_name = crate::schema::books)]`** — attribute บอก Diesel ว่า struct นี้สอดคล้องกับตารางไหนใน `schema.rs` (จำเป็นเพราะชื่อ struct `Book` ไม่ตรงกับชื่อตาราง `books` ตรง ๆ — ถ้าตั้งชื่อ struct ให้ตรงกับชื่อตาราง (แบบ pluralize อัตโนมัติ) บาง case ไม่ต้องระบุ attribute นี้ก็ได้ แต่การระบุตรง ๆ เสมอช่วยให้อ่านโค้ดง่ายกว่าและไม่พึ่งพา convention ที่ไม่ชัดเจน)
- **`#[diesel(check_for_backend(diesel::pg::Pg))]`** — เปิดการตรวจสอบเพิ่มเติมตอน compile ว่า **ทุก field type ตรงกับ SQL type ของ PostgreSQL จริง** (ไม่ใช่แค่ backend ทั่วไป) — ควรเปิดไว้เสมอในโปรเจกต์จริงเพราะช่วยจับ mismatch ได้เร็วขึ้นและ error message อ่านง่ายขึ้น (ไม่เปิดก็ยัง compile ผ่านได้ในหลาย case แต่ error ตอน field ผิดจะซับซ้อนกว่า)

#### struct สำหรับ**เขียน**ข้อมูลใหม่: `#[derive(Insertable)]`

```rust
// struct แยกสำหรับ "ข้อมูลใหม่ที่จะ insert" — สังเกตว่าไม่มี field id/created_at เลย
#[derive(Debug, Insertable)]
#[diesel(table_name = crate::schema::books)]
struct NewBook<'a> {
    isbn: &'a str,
    title: &'a str,
    author: &'a str,
    total_copies: i32,
    available_copies: i32,
    published_year: Option<i32>,
    shelf_id: Option<i64>,
    // ไม่มี id: เพราะเป็น BIGSERIAL ให้ฐานข้อมูล generate เองเสมอ
    // ไม่มี created_at: เพราะมี DEFAULT now() ในระดับฐานข้อมูลอยู่แล้ว (ปล่อยให้ DB จัดการเอง)
}
```

**ทำไม Diesel ถึงแยก struct สำหรับ insert ออกจาก struct สำหรับอ่าน (ต่างจาก SQLx ที่ Part 70 ใช้ struct เดียวทำทุกอย่าง)?**

นี่คือคำถามที่สำคัญมากและเป็นจุดที่แสดง philosophy ของ Diesel ชัดที่สุดจุดหนึ่ง เหตุผลมีสามชั้น:

1. **Shape ของข้อมูลตอนอ่านกับตอนเขียนต่างกันจริง ๆ ในทางความหมาย** — ตอน insert หนังสือเล่มใหม่ คุณ**ไม่มี** `id` (ฐานข้อมูลจะ generate ให้ผ่าน `BIGSERIAL`) และไม่มี `created_at` ที่แน่นอน (ปล่อยให้ `DEFAULT now()` จัดการ) — struct `Book` (สำหรับอ่าน) มี field ทั้งสองนี้เสมอเพราะแถวที่อ่านออกมาจากฐานข้อมูลแล้ว**ต้องมี**ค่าเหล่านี้แน่นอน 100% (ไม่ nullable) แต่ตอน insert คุณไม่มีค่าเหล่านี้จะให้เลย — ถ้าใช้ struct เดียวกันทั้งสองทาง คุณจะต้องใส่ค่า placeholder ปลอม ๆ ให้ `id`/`created_at` ตอน insert (เช่น `id: 0`) ซึ่งดูสับสนและเสี่ยง bug (เผลอส่ง `id: 0` ไปจริง ๆ ทั้งที่ควรให้ DB generate)
2. **Trait `Insertable` ต้องการ field ที่ implement `Into<Expression>` ที่ตรงกับ SQL type ของคอลัมน์ปลายทาง** — struct ที่ derive `Insertable` ไม่จำเป็นต้องมีครบทุกคอลัมน์ของตาราง (คอลัมน์ที่มี `DEFAULT` หรือ nullable สามารถ "ไม่ระบุ" ใน struct insert ได้เลย Diesel จะรู้ให้ปล่อยฐานข้อมูลจัดการเอง) — นี่คือ**ความสามารถที่ trait bound ของ Rust type system เปิดให้ทำได้อย่างเป็นธรรมชาติ** เพราะ `Insertable` แค่ต้องการ subset ของคอลัมน์ ไม่ใช่ทั้งหมด ต่างจาก `Queryable` ที่ (ปกติ) ต้องมีครบทุกคอลัมน์ที่ query คืนมา
3. **การใช้ `&'a str` (borrowed) แทน `String` (owned) ใน struct insert** — สังเกตว่า `NewBook<'a>` ใช้ `&'a str` ไม่ใช่ `String` เหมือน `Book` — เพราะ struct insert มักถูกสร้างขึ้นแค่ **ชั่วคราว**ตอนเรียก `.values()` (ไม่ได้เก็บไว้นานเหมือน struct ที่อ่านมาแสดงผล) การ borrow ข้อมูลจาก request payload ตรง ๆ แทน clone เป็น `String` ใหม่จึงมีประสิทธิภาพดีกว่าเล็กน้อย (หลักการเดียวกับที่ Part 4-8 สอนเรื่อง ownership/borrowing — เลือก borrow เมื่ออายุของข้อมูลสั้นและแน่นอน)

**เชื่อมกับ Part 44-45 (Proc Macro) และ Part 57 (Serde)**: `Queryable`/`Selectable`/`Insertable` ทั้งสามตัวเป็น **derive macro ที่แยกระบบกันเด็ดขาดจาก `serde::Serialize`/`Deserialize`** แม้จะ derive อยู่บน struct เดียวกันได้ (เหมือนตัวอย่าง `Book` ข้างบนที่ derive ทั้ง `Serialize` และ `Queryable`/`Selectable` พร้อมกัน) — mechanism เบื้องหลังทั้งสองระบบ**เหมือนกันในระดับที่ Part 44-45 สอน** (คือ proc macro ที่ parse `TokenStream` ของ struct definition แล้ว generate `impl` block ให้อัตโนมัติ) แต่ generate `impl` ของ **trait คนละชุด**: serde generate `impl Serialize`/`impl Deserialize` ที่แปลงไปมากับ JSON/format อื่น ๆ ส่วน Diesel generate `impl Queryable`/`impl Insertable`/`impl AsExpression` ที่แปลงไปมากับ SQL row/SQL expression — struct หนึ่งตัวจึงสามารถ "พูดได้สองภาษา" พร้อมกัน (คุยกับ JSON ผ่าน serde, คุยกับ SQL ผ่าน Diesel) เพราะ derive macro แต่ละตัว generate `impl` แยก trait กันคนละชุดโดยไม่ชนกัน — นี่คือพลังของระบบ trait ของ Rust ที่ Part 44-45 ปูพื้นไว้: หลาย ๆ derive macro ทำงานอิสระจากกันได้บน type เดียวกัน ตราบใดที่ generate `impl` คนละ trait

### 72.6 CRUD เต็มรูปแบบผ่าน Diesel DSL

หัวข้อนี้เดินผ่าน CRUD ครบทุกแบบบนตาราง `books` เทียบกับ SQL string จริงที่ Diesel generate ให้ (ผ่าน `diesel::debug_query!`) เพื่อให้เห็นว่า DSL แต่ละบรรทัด map ไปเป็น SQL อะไร — **ทดสอบจริงทุกก้อนต่อฐานข้อมูล `diesel_course_scratch`**

#### ตั้ง connection แบบตรง ๆ (ก่อนพูดถึง pool ในหัวข้อ 72.10)

```rust
use diesel::pg::PgConnection;
use diesel::prelude::*;

fn establish_connection() -> PgConnection {
    let database_url = std::env::var("DATABASE_URL")
        .expect("ต้องตั้ง DATABASE_URL ก่อนรันโปรแกรม");
    PgConnection::establish(&database_url)
        .unwrap_or_else(|e| panic!("ต่อฐานข้อมูลไม่สำเร็จ: {e}"))
}
```

สังเกตว่า `PgConnection::establish()` เป็น **synchronous function ธรรมดา** (ไม่มี `.await`) ตรงตามที่หัวข้อ 72.2 อธิบายไว้ — `PgConnection` คือ connection เดี่ยว ๆ ตัวเดียว (ไม่ใช่ pool) ใช้ตรง ๆ แบบนี้ได้ในโปรแกรมง่าย ๆ/สคริปต์ แต่ในเว็บแอปจริงต้องใช้ pool (หัวข้อ 72.10)

#### SELECT ด้วย `.filter()`/`.select()`/`.order()`/`.limit()`

```rust
use crate::schema::books::dsl::*;
use diesel::debug_query;
use diesel::pg::Pg;

fn find_available_books(conn: &mut PgConnection) -> QueryResult<Vec<Book>> {
    let query = books
        .filter(available_copies.gt(0))
        .order(title.asc())
        .limit(5)
        .select(Book::as_select());

    // debug_query! ให้เห็น SQL string จริงที่ Diesel generate ให้ query นี้ (ไม่รัน query จริง แค่ print)
    println!("Generated SQL: {}", debug_query::<Pg, _>(&query));

    query.load(conn)
}
```

**ผลลัพธ์จริงจาก `println!` ในบรรทัด debug_query** (ทดสอบจริง — รันฟังก์ชันนี้จริงต่อฐานข้อมูลที่มีข้อมูลตัวอย่างแล้ว):

```
Generated SQL: SELECT "books"."id", "books"."isbn", "books"."title", "books"."author",
"books"."total_copies", "books"."available_copies", "books"."published_year",
"books"."shelf_id", "books"."created_at" FROM "books" WHERE ("books"."available_copies" > $1)
ORDER BY "books"."title" ASC LIMIT $2 -- binds: [0, 5]
```

**อธิบายทีละส่วนว่า DSL แต่ละ method map ไปเป็นอะไรใน SQL:**

- `books` (จาก `use crate::schema::books::dsl::*`) คือจุดเริ่ม query — เทียบเท่า `FROM books`
- `.filter(available_copies.gt(0))` → `WHERE ("books"."available_copies" > $1)` พร้อม bind parameter `0` — สังเกตว่า Diesel **bind parameter ให้อัตโนมัติเสมอ** (เหมือนหลักการ parameterized query ที่ Part 70 สอนไว้กับ SQLx — ไม่มีการต่อ string ค่าตัวเลข/ข้อความเข้า SQL ตรง ๆ เลย ป้องกัน SQL injection ได้ในตัวโดย DSL ไม่ต้องเขียนอะไรเพิ่มเอง)
- `.order(title.asc())` → `ORDER BY "books"."title" ASC`
- `.limit(5)` → `LIMIT $2` (bind เป็น parameter เหมือนกัน ไม่ใช่ inline ตัวเลขตรง ๆ)
- `.select(Book::as_select())` → รายชื่อคอลัมน์ทั้งหมดที่ `Book` struct มี ถูกระบุชัดเจนทีละคอลัมน์ (`"books"."id", "books"."isbn", ...`) **ไม่ใช่ `SELECT *`** — นี่คือประโยชน์ของ `Selectable`/`as_select()` ที่กล่าวไว้ในหัวข้อ 72.5: การันตีว่าลำดับคอลัมน์ที่ query คืนมาตรงกับลำดับ field ใน struct `Book` เป๊ะเสมอ ไม่ต้องพึ่งการเดาลำดับจาก `SELECT *`

#### INSERT ด้วย `.values()`

```rust
fn insert_book(conn: &mut PgConnection, new_book: &NewBook) -> QueryResult<Book> {
    use crate::schema::books::dsl::books as books_table;

    diesel::insert_into(books_table)
        .values(new_book)
        .returning(Book::as_select()) // เทียบเท่า RETURNING ของ SQL — ได้แถวที่ insert สำเร็จกลับมาทันที
        .get_result(conn)
}
```

SQL ที่ Diesel generate จริง (จับผ่าน `debug_query!` เหมือนก่อนหน้า):

```
Generated SQL: INSERT INTO "books" ("isbn", "title", "author", "total_copies",
"available_copies", "published_year", "shelf_id") VALUES ($1, $2, $3, $4, $5, $6, $7)
RETURNING "books"."id", "books"."isbn", "books"."title", "books"."author",
"books"."total_copies", "books"."available_copies", "books"."published_year",
"books"."shelf_id", "books"."created_at"
-- binds: ["978-1-59327-828-1", "Programming Rust", "Jim Blandy", 3, 2, 2021, 2]
```

(binds ชุดนี้มาจากการเรียกจริงเพื่อ insert หนังสือ "Programming Rust" เข้าชั้นวางที่มี `id = 2` — สังเกตว่าค่า string/ตัวเลขทุกตัวถูกส่งเป็น bind parameter แยกจาก SQL ทั้งหมด ไม่มีค่าไหนถูกต่อเข้า SQL string ตรง ๆ เลย)

สังเกตว่ารายชื่อคอลัมน์ใน `INSERT INTO "books" (...)` ตรงกับ field ที่มีอยู่ใน `NewBook` struct เป๊ะ (ไม่มี `id`/`created_at` เลยตามที่ออกแบบไว้ในหัวข้อ 72.5) และ `.returning(Book::as_select())` ทำให้ได้ `RETURNING` clause กลับมาแบบเดียวกับที่ Part 70 สอนไว้กับ SQLx (`INSERT ... RETURNING`) — แนวคิดเดียวกัน ต่างกันแค่ syntax ที่ใช้เขียน

#### UPDATE ด้วย `.set()`

```rust
fn borrow_one_copy(conn: &mut PgConnection, target_id: i64) -> QueryResult<Book> {
    use crate::schema::books::dsl::{books, available_copies, id};

    diesel::update(books.filter(id.eq(target_id)))
        .set(available_copies.eq(available_copies - 1)) // ลดจำนวนที่เหลือ 1 เล่ม (คำนวณฝั่ง SQL ตรง ๆ)
        .returning(Book::as_select())
        .get_result(conn)
}
```

SQL ที่ Diesel generate จริง (ทดสอบเรียกจริงด้วย `target_id = 1`):

```
Generated SQL: UPDATE "books" SET "available_copies" = ("books"."available_copies" - $1)
WHERE ("books"."id" = $2)
RETURNING "books"."id", "books"."isbn", "books"."title", "books"."author",
"books"."total_copies", "books"."available_copies", "books"."published_year",
"books"."shelf_id", "books"."created_at" -- binds: [1, 1]
```

(bind แรก `1` คือค่าที่ลบออกจาก `available_copies` ตาม literal `1` ใน `- 1` ส่วน bind ที่สอง `1` คือ `target_id` ที่ส่งเข้าไป — สองค่านี้บังเอิญเป็นตัวเลขเดียวกันในการทดสอบนี้ ไม่ได้เกี่ยวข้องกัน)

**จุดที่น่าสนใจ**: `available_copies.eq(available_copies - 1)` ไม่ได้ทำให้ Rust ไปคำนวณค่าตัวเลขก่อนแล้วส่งค่าคงที่ไป — มันคือการสร้าง **SQL expression** `"available_copies" - $1` ที่คำนวณจริง**ฝั่ง PostgreSQL** ตอน query รัน (เหมือนเขียน `SET available_copies = available_copies - 1` ตรง ๆ ใน SQL) นี่คือตัวอย่างที่ดีมากว่า DSL ของ Diesel ไม่ได้จำกัดคุณให้ทำงานแค่ "ส่งค่าคงที่ไปเทียบ" — มันให้สร้าง SQL expression ที่ซับซ้อนกว่านั้นได้ผ่าน operator overload บน column type เอง (`-`, `+`, `*` ถูก implement ไว้ให้ column ที่เป็นตัวเลขโดยตรง)

#### DELETE

```rust
fn delete_book(conn: &mut PgConnection, target_id: i64) -> QueryResult<usize> {
    use crate::schema::books::dsl::{books, id};

    diesel::delete(books.filter(id.eq(target_id))).execute(conn)
}
```

SQL ที่ Diesel generate จริง (ทดสอบเรียกจริงด้วย `target_id = 3`):

```
Generated SQL: DELETE FROM "books" WHERE ("books"."id" = $1) -- binds: [3]
```

`.execute(conn)` คืนค่า `usize` — จำนวนแถวที่ถูกลบจริง (เทียบเท่า `rows_affected()` ของ SQLx ใน Part 70) — ใช้เช็คได้ว่า `id` ที่ระบุมามีอยู่จริงไหม (ถ้าคืน `0` แสดงว่าไม่มีแถวไหน match เลย ควรแปลงเป็น 404 ที่ handler)

#### Batch Insert: เพิ่มหลายแถวในคำสั่งเดียว

ในระบบจริงบางสถานการณ์ต้อง insert ข้อมูลหลายแถวพร้อมกัน (เช่น นำเข้าหนังสือจาก CSV ทีละหลายร้อยเล่ม) — การเรียก `.values()`/`.execute()` วนลูปทีละแถวทำงานได้แต่เสีย network round-trip ไปยัง PostgreSQL ทุกครั้งที่ insert หนึ่งแถว (คนละ query คนละครั้งที่ต้องรอ round-trip) Diesel รองรับการส่ง `Vec` ของ struct ที่ derive `Insertable` เข้า `.values()` ตรง ๆ เพื่อ insert **หลายแถวในคำสั่ง SQL เดียว**:

```rust
fn insert_many_books(conn: &mut PgConnection, new_books: &[NewBook]) -> QueryResult<Vec<Book>> {
    use crate::schema::books::dsl::books as books_table;

    diesel::insert_into(books_table)
        .values(new_books) // &[NewBook] หรือ &Vec<NewBook> ก็ได้ — ไม่ใช่ &NewBook เดี่ยว ๆ แบบก่อนหน้า
        .returning(Book::as_select())
        .get_results(conn) // .get_results() (มี s) แทน .get_result() — คืน Vec ของทุกแถวที่ insert สำเร็จ
}
```

**ผลลัพธ์จริง** (ทดสอบด้วย `Vec` ของ `NewBook` สองเล่ม):

```
Generated SQL: INSERT INTO "books" ("isbn", "title", "author", "total_copies",
"available_copies", "published_year", "shelf_id")
VALUES ($1, $2, $3, $4, $5, DEFAULT, DEFAULT), ($6, $7, $8, $9, $10, DEFAULT, DEFAULT)
RETURNING "books"."id", "books"."isbn", "books"."title", "books"."author",
"books"."total_copies", "books"."available_copies", "books"."published_year",
"books"."shelf_id", "books"."created_at"
-- binds: ["isbn-batch-1", "Book A", "Author A", 1, 1, "isbn-batch-2", "Book B", "Author B", 1, 1]

Inserted 2 books in one round-trip
  Book A (isbn-batch-1)
  Book B (isbn-batch-2)
```

**จุดที่น่าสนใจมาก**: SQL ที่ Diesel generate ให้เป็น `INSERT INTO ... VALUES (...), (...)` — **`VALUES` หลายชุดในคำสั่งเดียว** (มาตรฐาน SQL ที่ PostgreSQL รองรับสำหรับ multi-row insert) แทนการรัน `INSERT` ทีละครั้งวนลูป ทำให้เหลือ network round-trip แค่ครั้งเดียวไม่ว่าจะ insert กี่แถวก็ตาม (ในตัวอย่างนี้สองแถว แต่หลักการเดียวกันใช้ได้กับหลายพันแถว) — สังเกตด้วยว่าคอลัมน์ `published_year`/`shelf_id` ที่ `NewBook` ทั้งสองตัวในตัวอย่างนี้ไม่ได้ระบุค่า (เป็น `None`) ปรากฏเป็น `DEFAULT` ตรง ๆ ใน SQL (ไม่ใช่ `NULL` — เพราะคอลัมน์เหล่านี้ไม่มี `DEFAULT` ที่ระดับฐานข้อมูลจริง ๆ Diesel เลือกใช้ keyword `DEFAULT` ของ SQL ที่บอกให้ PostgreSQL ใส่ค่า default ของคอลัมน์นั้นเอง ซึ่งในกรณีคอลัมน์ nullable ที่ไม่มี `DEFAULT` ประกาศไว้ ค่านั้นก็คือ `NULL` อยู่ดี)

#### Upsert ด้วย `.on_conflict().do_update()`

อีกรูปแบบที่พบบ่อยมากในระบบจริงคือ **upsert** ("insert หรือ update ถ้ามีอยู่แล้ว") — เช่น sync ข้อมูลหนังสือจากระบบภายนอกที่ระบุ ISBN มา ถ้า ISBN นั้นมีอยู่แล้วให้ปรับจำนวนสำเนาที่มี ถ้ายังไม่มีให้ insert ใหม่ Diesel มี DSL ที่ map ตรงกับ PostgreSQL `INSERT ... ON CONFLICT ... DO UPDATE` เป๊ะ:

```rust
fn upsert_book_by_isbn(conn: &mut PgConnection, incoming: &NewBook) -> QueryResult<Book> {
    use crate::schema::books::dsl::{available_copies, books as books_table, isbn};

    diesel::insert_into(books_table)
        .values(incoming)
        .on_conflict(isbn) // คอลัมน์ที่มี UNIQUE constraint ที่จะใช้ตรวจ conflict (ต้องมี UNIQUE/PRIMARY KEY จริงในฐานข้อมูล)
        .do_update()
        .set(available_copies.eq(available_copies + incoming.available_copies))
        .returning(Book::as_select())
        .get_result(conn)
}
```

**ผลลัพธ์จริง** (ทดสอบด้วย ISBN ที่มีอยู่แล้วในฐานข้อมูล พร้อม `available_copies` ใหม่ = 5):

```
Generated SQL: INSERT INTO "books" ("isbn", "title", "author", "total_copies",
"available_copies", "published_year", "shelf_id") VALUES ($1, $2, $3, $4, $5, $6, $7)
ON CONFLICT ("isbn") DO UPDATE SET "available_copies" = ("books"."available_copies" + $8)
RETURNING "books"."id", "books"."isbn", "books"."title", "books"."author",
"books"."total_copies", "books"."available_copies", "books"."published_year",
"books"."shelf_id", "books"."created_at"
-- binds: ["978-1-59327-828-1", "Programming Rust (ignored on conflict)", "Jim Blandy", 5, 5, 2021, 1, 5]

After upsert: Book { id: 1, isbn: "978-1-59327-828-1", title: "Programming Rust", ...,
available_copies: 7, ... }
```

สังเกตว่าค่า `title`/`author` ที่ส่งเข้ามาใน `incoming` (`"Programming Rust (ignored on conflict)"`) **ไม่ถูกใช้เลย** เพราะ `.set()` ระบุแค่ `available_copies` เท่านั้นให้อัปเดตเมื่อชน conflict — แถวที่ได้กลับมายังคงมี `title = "Programming Rust"` ค่าเดิม (จากแถวที่มีอยู่แล้วในฐานข้อมูล) มีแค่ `available_copies` ที่เปลี่ยนจาก 2 เป็น 7 (2 + 5) ตรงตามที่ `.set()` ระบุไว้ — พฤติกรรมนี้ตรงตาม SQL `ON CONFLICT` มาตรฐานของ PostgreSQL เป๊ะ: `DO UPDATE SET` อัปเดตแค่คอลัมน์ที่ระบุไว้เท่านั้น คอลัมน์อื่นของแถวเดิมไม่ถูกแตะเลย

#### Error Handling: `diesel::result::Error` และการแปลงเป็น `AppError`

จาก Part 66 คุณรู้แล้วว่าการแปลง error ของ database driver ให้เป็น error type ของแอปเอง (`AppError`) เป็นขั้นตอนที่จำเป็นเพื่อคืน HTTP status code ที่เหมาะสม — `diesel::result::Error` มี variant สำคัญที่ควรรู้จักคล้ายกับ `sqlx::Error` ใน Part 70:

```rust
use diesel::result::{DatabaseErrorKind, Error as DieselError};

enum AppError {
    NotFound,
    Conflict(String),
    Internal(String),
}

impl From<DieselError> for AppError {
    fn from(err: DieselError) -> Self {
        match err {
            DieselError::NotFound => AppError::NotFound,
            DieselError::DatabaseError(DatabaseErrorKind::UniqueViolation, info) => {
                AppError::Conflict(info.message().to_string())
            }
            DieselError::DatabaseError(DatabaseErrorKind::ForeignKeyViolation, info) => {
                AppError::Conflict(info.message().to_string())
            }
            other => AppError::Internal(other.to_string()),
        }
    }
}
```

**ทดสอบจริง**: จงใจ insert หนังสือที่มี `isbn` ซ้ำกับที่มีอยู่แล้ว (ละเมิด `UNIQUE` constraint ที่ Part 70 อธิบายไว้ตอนออกแบบ schema) แล้วพิมพ์ error object ที่ได้กลับมาแบบ `{:#?}`:

```
Err(
    DatabaseError(
        UniqueViolation,
        "duplicate key value violates unique constraint \"books_isbn_key\"",
    ),
)
```

และเมื่อดึงรายละเอียดออกมาผ่าน method ของ `DatabaseErrorInformation` (`.message()`, `.details()`, `.constraint_name()`):

```
kind = UniqueViolation
message = duplicate key value violates unique constraint "books_isbn_key"
details = Some("Key (isbn)=(978-1-59327-828-1) already exists.")
constraint_name = Some("books_isbn_key")
```

**สังเกตความคล้ายกับ Part 70 อย่างมาก**: `diesel::result::Error::DatabaseError(DatabaseErrorKind::UniqueViolation, ...)` ทำหน้าที่เดียวกันเป๊ะกับ `sqlx::Error::Database(...)` ที่ต้องเช็ค `.is_unique_violation()`/`.constraint()` ต่อใน Part 70 — ทั้งสอง driver ห่อ error ดิบจาก PostgreSQL (SQLSTATE code `23505` สำหรับ unique violation) มาเป็น Rust enum ที่ตรวจสอบได้ผ่าน pattern matching แทนต้องไปแยก string เอง — `constraint_name` (`"books_isbn_key"`) คือชื่อ constraint ที่ PostgreSQL สร้างให้อัตโนมัติตอน `CREATE TABLE ... UNIQUE` (รูปแบบชื่อมาตรฐานคือ `<table>_<column>_key`) มีประโยชน์มากเมื่อตารางมี unique constraint หลายตัว (แยกได้ว่าตัวไหนถูกละเมิด เพื่อคืน error message ที่เจาะจงให้ผู้ใช้)

#### Raw SQL Escape Valve: `diesel::sql_query` เมื่อ DSL ไม่พอ

หัวข้อ 72.1 บอกไว้ว่า DSL ของ Diesel มีข้อจำกัดเรื่องความยืดหยุ่นเทียบกับ SQL ดิบ — SQL บางรูปแบบ (window function ซับซ้อน, CTE หลายชั้น, `GROUP BY` ร่วมกับ aggregate function ที่ DSL ยังไม่ครอบคลุมสะดวก) เขียนผ่าน DSL ได้ยากหรือเขียนไม่ได้เลย Diesel จึงมี **escape valve** ให้เขียน SQL ดิบตรง ๆ ผ่าน `diesel::sql_query()` โดยยังได้ type-safety ระดับหนึ่งกลับมาผ่าน `#[derive(QueryableByName)]`:

```rust
use diesel::sql_types::{BigInt, Text};

// struct สำหรับผลลัพธ์ raw query — ต้องระบุ SQL type ของแต่ละคอลัมน์เอง (ไม่มี schema.rs ช่วยตรวจให้อัตโนมัติแบบ DSL ปกติ)
#[derive(Debug, QueryableByName)]
struct ShelfBookCount {
    #[diesel(sql_type = Text)]
    shelf_code: String,
    #[diesel(sql_type = BigInt)]
    book_count: i64,
}

fn count_books_per_shelf(conn: &mut PgConnection) -> QueryResult<Vec<ShelfBookCount>> {
    diesel::sql_query(
        "SELECT s.code AS shelf_code, COUNT(b.id) AS book_count \
         FROM shelves s LEFT JOIN books b ON b.shelf_id = s.id \
         GROUP BY s.code ORDER BY s.code",
    )
    .load::<ShelfBookCount>(conn)
}
```

**ทดสอบจริง** (ต่อฐานข้อมูลที่มีชั้นวาง `A1` และหนังสือสองเล่มอยู่ในชั้นนั้น):

```
=== raw sql_query GROUP BY result ===
  shelf A1 -> 2 books
```

**ข้อสังเกตสำคัญที่เชื่อมกลับไปหัวข้อ 72.1 โดยตรง**: `sql_query()` รับ `&str` เป็น SQL ดิบล้วน ๆ **เหมือน SQLx ทุกประการ** — จุดนี้คือที่ที่ Diesel "ยืม" แนวทางของ SQLx มาใช้ชั่วคราวเมื่อ DSL ไม่พอ แต่ต่างจาก SQLx ตรงที่**ไม่มีการตรวจสอบ SQL string นี้กับฐานข้อมูลจริงตอน compile เลย** (ไม่มี macro แบบ `query!` มาช่วย) — ความปลอดภัยที่ยังพอได้กลับมาคือแค่ระดับ **"ชนิดข้อมูลที่ struct ประกาศไว้ตรงกับที่ query คืนมาไหม"** ผ่าน `#[diesel(sql_type = ...)]` ที่ต้องระบุมือทุกคอลัมน์ (ตรวจแค่ตอน**รัน**ถ้าระบุ SQL type ผิด ไม่ใช่ตอน compile) — เท่ากับว่า `sql_query` สูญเสียจุดขายหลักของ Diesel (compile-time DSL safety) ไปเกือบทั้งหมด แลกกับความยืดหยุ่นของ SQL ดิบ **ควรใช้เป็นทางเลือกสุดท้าย** เฉพาะกรณีที่ DSL ปกติเขียนไม่ได้จริง ๆ เท่านั้น (คล้ายกับที่หัวข้อ 70.4 ของ Part 70 แนะนำเรื่อง `sqlx::Row`/`.try_get()` แบบ dynamic — เป็น pattern เดียวกันในทั้งสอง ecosystem: มี "ทางลัดที่สูญเสีย safety" ไว้เผื่อสถานการณ์ที่ abstraction หลักไม่พอ)

### 72.7 Transaction: การันตี Atomicity ด้วย `conn.transaction()`

จาก Part 70 คุณรู้แล้วว่าการยืมหนังสือ (ลด `available_copies` + สร้าง `borrow_records`) ต้องเป็น **atomic operation** — ทั้งสองคำสั่งต้องสำเร็จพร้อมกันหรือไม่สำเร็จเลยทั้งคู่ (ไม่ใช่ลด `available_copies` ไปแล้วแต่สร้าง record ไม่สำเร็จ ทำให้ข้อมูลไม่ตรงกัน) Diesel มี method `conn.transaction(|conn| { ... })` ที่ทำหน้าที่เดียวกับ `pool.begin()`/`commit()`/`rollback()` ของ SQLx แต่ด้วย pattern ที่ต่างออกไปเล็กน้อย (ผูกกับ closure แทนเรียก method แยก):

```rust
use diesel::result::Error as DieselError;

fn borrow_book_tx(
    conn: &mut PgConnection,
    target_book_id: i64,
    borrower: &str,
) -> QueryResult<()> {
    use crate::schema::books::dsl::{available_copies, books, id};
    use crate::schema::borrow_records::dsl::borrow_records;

    // conn.transaction() รับ closure ที่คืน Result<T, E> — ถ้า closure คืน Ok ทั้ง transaction จะ COMMIT
    // ถ้าคืน Err (ไม่ว่าจาก query จริงที่ fail หรือจากการ return Err เองตรง ๆ) จะ ROLLBACK อัตโนมัติทั้งหมด
    conn.transaction(|conn| {
        // เงื่อนไข available_copies > 0 อยู่ใน WHERE — ถ้าไม่มีเล่มให้ยืม UPDATE จะไม่กระทบแถวไหนเลย (0 rows)
        let updated_rows = diesel::update(
            books.filter(id.eq(target_book_id)).filter(available_copies.gt(0)),
        )
        .set(available_copies.eq(available_copies - 1))
        .execute(conn)?;

        if updated_rows == 0 {
            // ไม่มีเล่มให้ยืมแล้ว — คืน Err เพื่อบังคับ rollback (ยกเลิกทุกอย่างในนี้ ถึงจะยังไม่มีอะไรให้ยกเลิกก็ตาม)
            return Err(DieselError::RollbackTransaction);
        }

        #[derive(Insertable)]
        #[diesel(table_name = crate::schema::borrow_records)]
        struct NewBorrowRecord<'a> {
            book_id: i64,
            borrower_name: &'a str,
        }

        diesel::insert_into(borrow_records)
            .values(&NewBorrowRecord { book_id: target_book_id, borrower_name: borrower })
            .execute(conn)?;

        Ok(())
    })
}
```

**อธิบายกลไก**: closure ที่ส่งให้ `conn.transaction(...)` รับ `&mut PgConnection` ตัวใหม่ (ที่อยู่ภายใต้ transaction จริง — ทุก query ข้างในต้องใช้ connection ตัวนี้ ไม่ใช่ตัวนอก) และต้องคืน `Result<T, E>` — Diesel ดูค่าที่ closure คืนมาเป็นตัวตัดสินว่าจะ `COMMIT` หรือ `ROLLBACK`: **`Ok(_)` → `COMMIT`, `Err(_)` → `ROLLBACK`** โดยอัตโนมัติ ไม่ต้องเรียก `.commit()`/`.rollback()` เองแบบ SQLx เลย (SQLx ให้ `Transaction` object แยกที่ต้องเรียก method เองตรง ๆ — Diesel เลือกผูกกับ closure/return value ให้ compiler ช่วยการันตีว่าคุณจัดการ path สำเร็จ/ล้มเหลวครบทุกทางจริง เพราะ `Result` เป็น type ที่ Rust บังคับให้ตรวจ)

`diesel::result::Error::RollbackTransaction` คือ variant พิเศษที่มีไว้สำหรับ "ตั้งใจ rollback โดยไม่ได้มี query error จริง ๆ" (เช่นในตัวอย่างนี้ — เงื่อนไขทางธุรกิจไม่ผ่าน ไม่ใช่ database error) — `?` ที่ตามหลัง `.execute(conn)?`/`.execute(conn)?` อื่น ๆ ในนี้ก็ทำให้ query error จริงใด ๆ (เช่น constraint violation) ทำให้ทั้ง closure คืน `Err` และ rollback อัตโนมัติเช่นกัน โดยไม่ต้องเขียน rollback เองเลย

**พิสูจน์ rollback ทำงานจริง** (ทดสอบจริง — ตั้ง `available_copies = 0` ก่อน แล้วเรียก `borrow_book_tx`):

```
=== Before failed borrow: available_copies = 0 ===
borrow_result = Err(RollbackTransaction)
=== After failed borrow (should be unchanged): available_copies = 0 ===
=== borrow_records count after rollback attempt: 0 ===
```

สังเกตว่า `available_copies` **ยังเป็น 0 เหมือนเดิม** (ไม่ได้ถูกลดเป็น -1 แม้จะมีการเรียก `.execute()` ของ `UPDATE` ไปแล้วในนี้ก็ตาม — เพราะ `WHERE available_copies > 0` ทำให้ `UPDATE` นี้ไม่กระทบแถวไหนเลยอยู่แล้วตั้งแต่ต้น ไม่ใช่ผลจาก rollback) และ `borrow_records` **ไม่มี record ใหม่เกิดขึ้นเลย** (count = 0) ยืนยันว่าไม่มี "ครึ่ง ๆ กลาง ๆ" เกิดขึ้น — พอลองตั้ง `available_copies = 1` แล้วเรียกใหม่ (คราวนี้ควรสำเร็จ):

```
success_result = Ok(())
=== After successful borrow: available_copies = 0 ===
=== borrow_records count after success: 1 ===
```

ทั้งสองคำสั่งสำเร็จพร้อมกันจริง (`available_copies` ลดจาก 1 เป็น 0 **และ** `borrow_records` มี record ใหม่เกิดขึ้นพร้อมกัน) — พิสูจน์ atomicity ครบทั้งสองทาง (สำเร็จทั้งคู่ กับ ล้มเหลวทั้งคู่) เหมือนหลักการที่ Part 70 พิสูจน์ไว้กับ SQLx transaction ทุกประการ เพียงแต่ syntax ต่างกัน (closure + return value แทน `.begin()`/`.commit()`/`.rollback()` แยก method)

#### Testing กับ Diesel: `conn.begin_test_transaction()`

จาก Part 32-33 คุณรู้แล้วว่า integration test ที่แตะฐานข้อมูลจริงต้องมีวิธี "ล้างข้อมูลทดสอบ" ไม่ให้ตกค้างข้าม test แต่ละตัว (Part 70 แก้ปัญหานี้ด้วย `#[sqlx::test]` ที่สร้างฐานข้อมูลแยกอัตโนมัติต่อ test) Diesel มีกลไกที่เบากว่าแต่ทำงานคล้ายกันมาก: **`conn.begin_test_transaction()`**:

```rust
#[test]
fn test_insert_book_does_not_leak_into_other_tests() {
    let mut conn = establish_connection();
    conn.begin_test_transaction().unwrap(); // เปิด transaction ที่ "ตั้งใจไม่ commit เด็ดขาด"

    use crate::schema::books::dsl::books as books_table;
    let temp_book = NewBook {
        isbn: "isbn-temp-test",
        title: "Temp Test Book",
        author: "Test Author",
        total_copies: 1,
        available_copies: 1,
        published_year: None,
        shelf_id: None,
    };
    diesel::insert_into(books_table).values(&temp_book).execute(&mut conn).unwrap();

    let count_inside: i64 = books_table.count().get_result(&mut conn).unwrap();
    assert_eq!(count_inside, 2); // เห็นข้อมูลที่ insert ไปเมื่อกี้จริงภายใน transaction เดียวกัน

    // conn ถูก drop ที่ท้าย scope ของฟังก์ชันนี้ — Diesel rollback transaction ให้อัตโนมัติตอน drop
}
```

**กลไกที่แท้จริง**: `begin_test_transaction()` เปิด transaction ปกติ แต่ผูก connection ไว้ในสถานะที่ **rollback เสมอตอน `Drop`** (ไม่ว่า test จะจบแบบ pass หรือ panic ก็ตาม) แทนที่จะต้องเรียก `.rollback()` เองท้าย test ทุกครั้ง — ผลคือทุกอย่างที่ทำใน connection นี้ (insert/update/delete) **มองเห็นได้จริงภายใน test ตัวเดียวกัน** (เหมือนทำงานกับฐานข้อมูลจริงปกติทุกประการ) แต่**หายไปทั้งหมดทันทีที่ test จบ** ไม่ทิ้งข้อมูลตกค้างให้ test ตัวถัดไปเจอ

**ทดสอบจริง**: ยืนยันด้วยการเปิด connection ใหม่ (แยกจาก connection ที่ใช้ทำ `begin_test_transaction()`) หลังฟังก์ชันทดสอบข้างบนจบแล้ว:

```
=== count inside test transaction: 2 ===
=== count in a FRESH connection after test fn ended: 1 ===
```

`count` ภายใน test เห็น 2 แถว (1 แถวจากข้อมูลตั้งต้น + 1 แถวที่ insert ใหม่) แต่ connection ใหม่ (ที่เปิดหลัง test จบ) เห็นแค่ 1 แถวเท่านั้น — ยืนยันว่า `begin_test_transaction()` rollback ทุกอย่างไปจริงตอน connection ก่อนหน้าถูก drop ไม่มีข้อมูลทดสอบหลงเหลือให้กระทบ test ตัวอื่น

**เทียบกับ `#[sqlx::test]` ของ SQLx (Part 70)**: `#[sqlx::test]` แก้ปัญหาด้วยการ**สร้างฐานข้อมูลใหม่แยกกันทั้งตัว**ต่อ test function หนึ่งตัว (isolation ระดับฐานข้อมูล) ส่วน `begin_test_transaction()` ของ Diesel แก้ปัญหาด้วย**transaction ที่ไม่ commit**บนฐานข้อมูลเดียวกัน (isolation ระดับ transaction) — ทั้งสองวิธีให้ผลลัพธ์ที่คล้ายกัน (test ไม่ทิ้งข้อมูลตกค้าง) แต่วิธีของ Diesel **เบากว่ามาก** (ไม่ต้องสร้าง/ลบฐานข้อมูลจริงทุก test ซึ่งมี overhead สูงกว่า) แลกกับข้อจำกัดที่ว่าถ้า test เขียนโค้ดที่เปิด **connection ใหม่เอง** (แยกจาก connection ที่ `begin_test_transaction()` ไว้) connection ใหม่นั้นจะไม่เห็นข้อมูลที่ insert ไว้ใน transaction เลย (เพราะ PostgreSQL isolation ปกติ transaction ที่ยังไม่ commit จะไม่เห็นข้ามการเชื่อมต่อ) — เหมาะกับ test ที่ทำงานผ่าน connection เดียวตลอดทั้ง test เท่านั้น

### 72.8 Associations และ Join: `shelves` มีหนังสือหลายเล่ม (`.inner_join()`)

จากตาราง `shelves`/`books` ที่สร้างไว้ในหัวข้อ 72.3 (ชั้นวางหนึ่งชั้นมีหนังสือได้หลายเล่ม ผ่าน `books.shelf_id -> shelves.id`) ลองเขียน query ที่ join สองตารางนี้เพื่อดึง "หนังสือทุกเล่มพร้อมชื่อชั้นวางที่มันอยู่":

```rust
use crate::schema::{books, shelves};
use diesel::prelude::*;

// struct สำหรับผลลัพธ์ query ที่ join สองตาราง — เลือกมาเฉพาะคอลัมน์ที่ต้องใช้จริง
#[derive(Debug, Queryable, Selectable)]
#[diesel(table_name = books)]
struct BookWithShelfCode {
    #[diesel(select_expression = books::title)]
    title: String,
    #[diesel(select_expression = shelves::code)]
    shelf_code: String,
}

fn list_books_with_shelf(conn: &mut PgConnection) -> QueryResult<Vec<(String, String)>> {
    books::table
        .inner_join(shelves::table) // ใช้ diesel::joinable!(books -> shelves (shelf_id)) ที่ schema.rs generate ไว้
        .select((books::title, shelves::code))
        .order(books::title.asc())
        .load::<(String, String)>(conn)
}
```

SQL ที่ Diesel generate จริง (ทดสอบต่อฐานข้อมูลที่มีชั้นวาง `A1`/`A2` และหนังสือถูกจัดวางไว้บ้างแล้ว):

```
Generated SQL: SELECT "books"."title", "shelves"."code" FROM ("books"
INNER JOIN "shelves" ON ("books"."shelf_id" = "shelves"."id"))
ORDER BY "books"."title" ASC -- binds: []
```

ผลลัพธ์จริงที่ได้กลับมา (ข้อมูลตัวอย่าง 3 เล่มที่จัดวางไว้แล้ว):

```
Norwegian Wood -> shelf A1
Programming Rust -> shelf A2
The Rust Programming Language -> shelf A2
```

**อธิบายกลไก `.inner_join()`**: Diesel รู้ว่าจะ `INNER JOIN` ตารางไหนด้วยเงื่อนไขอะไร (`ON ("books"."shelf_id" = "shelves"."id")`) จาก macro `diesel::joinable!(books -> shelves (shelf_id))` ที่อยู่ใน `schema.rs` (หัวข้อ 72.4) — คุณไม่ต้องเขียนเงื่อนไข `ON` เองเลยใน DSL เพราะ Diesel **generate เงื่อนไข join ให้อัตโนมัติ**จาก declaration นั้น (ต่างจาก SQL ดิบที่ต้องเขียน `ON` ทุกครั้ง) — นี่คือตัวอย่างที่ดีว่า "single source of truth" ของ `schema.rs` ไม่ใช่แค่บอกว่าตารางมีคอลัมน์อะไร แต่บอกความสัมพันธ์ระหว่างตารางด้วย ทำให้ query ที่ join ผิดตาราง/ผิดคอลัมน์ (ที่ไม่มี `joinable!` ประกาศไว้) **compile ไม่ผ่านทันที** ก่อนจะไปถึงฐานข้อมูลจริงด้วยซ้ำ

**หมายเหตุเรื่อง `#[diesel(select_expression = ...)]`**: attribute นี้ (เพิ่มเข้ามาในเวอร์ชันใหม่ ๆ ของ `diesel_derives`) ให้ struct หนึ่งตัวรวมคอลัมน์จากหลายตารางเข้าด้วยกันตอนใช้ `Selectable`/`as_select()` กับ query ที่ join — ถ้าไม่ต้องการความสะดวกนี้ ใช้วิธีที่ตรงไปตรงกว่า (แบบในตัวอย่างฟังก์ชัน `list_books_with_shelf` ที่ `.select((books::title, shelves::code))` แล้ว `.load::<(String, String)>()` เป็น tuple ตรง ๆ) ก็ได้ผลลัพธ์เดียวกัน เพียงแต่ไม่ได้ struct ที่มีชื่อ field ให้อ่านง่าย

### 72.9 Compile-Time Query Safety ในทางปฏิบัติ: อ่าน Error จริงของ Diesel

นี่คือจุดขายหลักของ Diesel — มาดูว่าเมื่อเขียน query ผิดจริง ๆ จะเกิดอะไรขึ้น สองกรณีคลาสสิกที่สุด: **พิมพ์ชื่อคอลัมน์ผิด** และ **เทียบ type ผิด**

#### กรณีที่ 1: อ้างคอลัมน์ที่ไม่มีจริง

```rust
// ผิดตั้งใจ: พิมพ์ "titel" (สลับตัวอักษร) ไม่ตรงกับคอลัมน์จริงที่ชื่อ "title"
fn broken_query_wrong_column(conn: &mut PgConnection) -> QueryResult<Vec<Book>> {
    use crate::schema::books::dsl::*;
    books.filter(titel.eq("Rust")).select(Book::as_select()).load(conn)
    //            ^^^^^ ไม่มีคอลัมน์ชื่อนี้ใน schema.rs
}
```

**Error จริงที่ `cargo build` แสดง** (ทดสอบจริง — คัดลอกมาแบบไม่ตัดทอนส่วนสำคัญ):

```
error[E0425]: cannot find value `titel` in this scope
   --> src/main.rs:208:18
    |
208 |     books.filter(titel.eq("Rust")).select(Book::as_select()).load(conn)
    |                  ^^^^^
    |
   ::: src/schema.rs:7:9
    |
  7 |         title -> Text,
    |         ----- similarly named unit struct `title` defined here
    |
help: a unit struct with a similar name exists
    |
208 -     books.filter(titel.eq("Rust")).select(Book::as_select()).load(conn)
208 +     books.filter(title.eq("Rust")).select(Book::as_select()).load(conn)
    |
```

**นี่คือ error ที่อ่านง่ายที่สุดที่ Diesel ให้ได้**: เพราะ `use crate::schema::books::dsl::*` import ชื่อคอลัมน์ทุกตัวเข้ามาเป็น **identifier ระดับ Rust ตรง ๆ** (ไม่ใช่ string) การพิมพ์ชื่อผิดจึงกลายเป็น error ธรรมดาที่สุดของ Rust คือ **"cannot find value in this scope"** — และ compiler ยัง**ฉลาดพอจะแนะนำ** (`help: a unit struct with a similar name exists`) ว่าอาจหมายถึง `title` ด้วยซ้ำ พร้อมชี้กลับไปยังตำแหน่งจริงใน `src/schema.rs` ที่ประกาศ `title -> Text` ไว้ (กลไก "did you mean" ของ `rustc` ที่เทียบชื่อ identifier ที่มีอยู่ในโมดูลที่มองเห็นได้ ณ จุดนั้น) — สังเกตว่า error พูดถึง `title` ว่าเป็น **"unit struct"** ไม่ใช่ "ตัวแปร" หรือ "ค่า" ทั่วไป ตรงตามที่หัวข้อ 72.4 อธิบายไว้: macro `table!` generate **struct จริง** ให้แต่ละคอลัมน์ ไม่ใช่แค่ string ชื่อคอลัมน์เฉย ๆ — สิ่งนี้ตรงกับที่หัวข้อ 72.1 บอกไว้ว่า error ของ Diesel "ผูกกับ Rust error ธรรมดา" ไม่ใช่ error รูปแบบพิเศษของตัวมันเอง

#### กรณีที่ 2: เทียบ type ผิด — นี่คือ error ที่ "กลไกจริง" แต่ "อ่านยาก" ตามที่บทนี้เตือนไว้

```rust
// ผิดตั้งใจ: available_copies เป็น Int4 (i32) แต่ดันเทียบกับ String
fn broken_query_wrong_type(conn: &mut PgConnection) -> QueryResult<Vec<Book>> {
    use crate::schema::books::dsl::*;
    books
        .filter(available_copies.eq("สิบเล่ม".to_string())) // ผิด: ต้องเป็น i32 ไม่ใช่ String
        .select(Book::as_select())
        .load(conn)
}
```

**Error จริง** (ทดสอบจริง — `cargo build` แสดง error **ทั้งหมด 7 ก้อนรวด** จาก query ผิดบรรทัดเดียว ตามที่คาดไว้ตั้งแต่หัวข้อ 72.1 ว่าเป็น trait bound error ของ generic ระดับลึก คล้ายกับที่ Part 63-64 เตือนไว้เรื่อง error ของ Axum ที่ generic bound ซับซ้อน — ด้านล่างคัดมาสามก้อนแรกที่ให้ข้อมูลตรงประเด็นที่สุด เรียงตามลำดับที่ `rustc` แสดงจริง):

```
error[E0277]: the trait bound `String: AsExpression<Integer>` is not satisfied
   --> src/main.rs:209:34
    |
209 |         .filter(available_copies.eq("สิบเล่ม".to_string()))
    |                                  ^^ the trait `AsExpression<Integer>` is not implemented for `String`
    |
    = help: the following other types implement trait `AsExpression<T>`:
              `&String` implements `AsExpression<Citext>`
              `&String` implements `AsExpression<diesel::sql_types::Text>`
              `String` implements `AsExpression<Citext>`
              `String` implements `AsExpression<diesel::sql_types::Nullable<Citext>>`
              `String` implements `AsExpression<diesel::sql_types::Nullable<diesel::sql_types::Text>>`
              `String` implements `AsExpression<diesel::sql_types::Text>`

error[E0277]: the trait bound `String: diesel::Expression` is not satisfied
   --> src/main.rs:209:10
    |
209 |         .filter(available_copies.eq("สิบเล่ม".to_string()))
    |          ^^^^^^ the trait `diesel::Expression` is not implemented for `String`
    |
    = help: the following other types implement trait `diesel::Expression`:
              &T
              AliasedField<S, C>
              Box<T>
              CaseWhen<CaseWhenConditionsIntermediateNode<W, T, Whens>, E>
              ...and 207 others
    = note: required for `diesel::expression::operators::Eq<schema::books::columns::available_copies, String>` to implement `diesel::Expression`
    = note: required for `SelectStatement<FromClause<schema::books::table>>` to implement `FilterDsl<Grouped<Eq<available_copies, String>>>`

error[E0277]: `String` is no valid SQL fragment for the `Pg` backend
    --> src/main.rs:211:15
     |
 211 |         .load(conn)
     |          ---- ^^^^ the trait `QueryFragment<Pg>` is not implemented for `String`
     |          |
     |          required by a bound introduced by this call
     |
     = note: this usually means that the `Pg` database system does not support
             this SQL syntax
     = note: required for `diesel::expression::operators::Eq<schema::books::columns::available_copies, String>` to implement `QueryFragment<Pg>`
     = note: required for `SelectStatement<FromClause<table>, SelectClause<...>, ..., ...>` to implement `LoadQuery<'_, _, Book>`
note: required by a bound in `diesel::RunQueryDsl::load`
```

(error ที่เหลืออีก 4 ก้อนเป็นผลพลอยได้จาก trait อื่น ๆ ที่ `String` ไม่ implement เช่นกัน — `ValidGrouping<()>`, `AppearsOnTable<...>`, `QueryId`, และ associated-type mismatch อีกหนึ่งจุด — ทั้งหมดชี้กลับไปที่ปัญหาเดียวกันคือ `String` ไม่ใช่ type ที่ใช้เทียบกับคอลัมน์ SQL type `Integer` ได้ ตัดออกจากที่แสดงในบทนี้เพื่อไม่ให้ยาวเกินไป แต่โครงสร้างเป็นแบบเดียวกันทั้งหมด)

**วิธีอ่าน error นี้ทีละชั้น (เทคนิคเดียวกับที่ Part 63-64 สอนเรื่องอ่าน error ของ Axum ที่ trait bound ซับซ้อน):**

1. **มองหา error ก้อนแรกก่อนเสมอ แล้วมองหาคำว่า "the trait bound ... is not satisfied"** — ก้อนแรก `the trait bound String: AsExpression<Integer> is not satisfied` คือ**เนื้อความจริงของปัญหาทั้งหมด**: `.eq()` ต้องการ argument ที่ implement `AsExpression<Integer>` (เพราะ `available_copies` เป็นคอลัมน์ SQL type `Integer`) แต่ `String` ไม่ implement trait นี้ — error ที่เหลือทั้งหมด (อีก 6 ก้อน) เป็นแค่**ผลกระทบต่อเนื่อง**จาก trait bound แรกที่พังไปแล้วเท่านั้น ไม่ใช่ปัญหาใหม่แยกกัน
2. **อ่านส่วน `help: the following other types implement trait AsExpression<T>`** — ส่วนนี้บอกตรง ๆ ว่า `String` implement `AsExpression<Text>`/`AsExpression<Citext>` ได้ (เพราะมันคือ text จริง) แต่ **ไม่มีบรรทัดไหนที่บอกว่า implement `AsExpression<Integer>`** — นี่คือหลักฐานที่ชัดที่สุดว่าปัญหาคือ "type ไม่ตรง" ไม่ใช่ปัญหาอื่น และพิสูจน์สิ่งที่หัวข้อ 72.1 อธิบายไว้ตรง ๆ: **Diesel ตรวจ type ผ่าน trait bound ของ Rust เอง ไม่ใช่ผ่านการไปถามฐานข้อมูลจริง**
3. **error ที่ 2 และ 3 (`diesel::Expression is not satisfied`, `QueryFragment<Pg> is not implemented`)** เป็นผลลูกโซ่ตามธรรมชาติของระบบ trait ของ Diesel: เมื่อ `Eq<available_copies, String>` (ผลจาก `.eq()`) ไม่ implement `Expression` ได้ตั้งแต่ต้น ทุก trait ที่ต่อยอดจาก `Expression` (ที่ `.filter()`/`.load()` ต้องการ) ก็พังต่อเป็นทอด ๆ ตามไปด้วย — **ในทางปฏิบัติไม่จำเป็นต้องอ่านทุกก้อนให้ครบ** ก้อนแรกก้อนเดียวก็บอกสาเหตุที่แท้จริงพอแล้วเสมอ

**วิธีแก้**: เปลี่ยนให้ type ตรงกัน — ถ้าตั้งใจจะเทียบตัวเลข ให้ใส่ `i32` ตรง ๆ (`available_copies.eq(10)`) ถ้าตั้งใจจะเทียบ column ที่เป็น `Text` จริง ๆ ต้องเปลี่ยนไปใช้ column ที่ถูก (เช่น `title.eq(...)`) — Diesel **ป้องกัน bug ประเภท "เทียบ column ผิด type โดยไม่ตั้งใจ" ได้ 100% ตอน compile** ซึ่งเป็นสิ่งที่ SQLx (โดย `query!`/`query_as!`) ก็ทำได้เหมือนกัน (Part 70) เพียงแต่ error ของ SQLx macro มักสั้นและตรงประเด็นกว่ามาก (เพราะมันสร้าง error message ของตัวเองโดยเฉพาะ ไม่ได้พึ่ง trait bound error ทั่วไปของ `rustc`) — **นี่คือ trade-off ที่แท้จริงระหว่างสองแนวทางที่พิสูจน์ได้จากตัวอย่างข้างบนตรง ๆ**: Diesel ได้ compile-time safety แบบเดียวกัน แต่ error message มาจากกลไก generic ทั่วไปของภาษา (จึงยาว/ซับซ้อน/มีหลายก้อนพร้อมกันในบาง case) ส่วน SQLx ลงทุนเขียน error message เฉพาะทางไว้ในตัว macro เอง (จึงสั้นกว่ามากในหลาย case แต่ต้องมี DB connection/cache ตอน build เสมอ)

### 72.10 Connection Pooling กับ Diesel: `r2d2` เทียบกับ Pool ของ SQLx

#### `r2d2`: pool แบบ synchronous ดั้งเดิม

จาก Part 70 คุณรู้แล้วว่า SQLx มี `PgPoolOptions`/`PgPool` เป็น pool ที่ **built-in อยู่ในตัว crate เอง และเป็น async-native** (คืน `Future` ตอน acquire connection) — Diesel **ไม่มี pool ในตัว** ต้องพึ่ง crate ภายนอกชื่อ `r2d2` (general-purpose connection pool สำหรับอะไรก็ได้ที่เป็น "resource ที่ต้องใช้ซ้ำ" ไม่ได้ผูกกับฐานข้อมูลโดยเฉพาะ) ผ่าน adapter `diesel::r2d2::ConnectionManager` ที่สอนให้ `r2d2` รู้จักวิธีเปิด/ปิด `PgConnection`

```rust
use diesel::r2d2::{ConnectionManager, Pool};
use diesel::pg::PgConnection;

type DbPool = Pool<ConnectionManager<PgConnection>>;

fn create_pool(database_url: &str) -> DbPool {
    let manager = ConnectionManager::<PgConnection>::new(database_url);
    Pool::builder()
        .max_size(10) // เทียบเท่า .max_connections() ของ PgPoolOptions
        .build(manager)
        .expect("สร้าง connection pool ไม่สำเร็จ")
}
```

**ข้อสังเกตสำคัญที่สุด**: `Pool::builder().build(manager)` เป็น **synchronous function** (ไม่มี `.await`) ต่างจาก `PgPoolOptions::new().connect(url).await` ของ SQLx ที่เป็น async — สอดคล้องกับหัวข้อ 72.2 ที่ว่า Diesel เป็น sync ตั้งแต่รากฐาน ทุกอย่างในระบบ (connection, pool, query) จึงเป็น sync ทั้งชุด ไม่มีจุดไหนเป็น async แทรกเข้ามาเลยถ้าไม่ใช้ `diesel-async`

**การขอ connection จาก pool** (`.get()`) ก็เป็น synchronous เช่นกัน — **บล็อก thread ปัจจุบันจนกว่าจะมี connection ว่างให้ยืม** (เทียบเท่า `pool.acquire().await` ของ SQLx ที่เป็น async รอ) — นี่คือเหตุผลอีกจุดที่การใช้ Diesel pool ต้องอยู่ใน `spawn_blocking` เสมอ ไม่ใช่แค่ตัว query เท่านั้นที่ block แต่ขั้นตอน "ขอ connection จาก pool" ก็ block ด้วย

#### ตารางเปรียบเทียบ `r2d2` (Diesel) กับ pool ของ SQLx

| ประเด็น | `r2d2` + `ConnectionManager` (Diesel) | `PgPoolOptions`/`PgPool` (SQLx, Part 70) |
|---|---|---|
| มาจากไหน | crate ภายนอกทั่วไป (ไม่ผูกกับฐานข้อมูลโดยเฉพาะ) + adapter ของ Diesel | built-in ใน SQLx เอง ออกแบบมาเฉพาะสำหรับ database pool |
| Sync/Async | Synchronous ล้วน ๆ | Async-native |
| `.get()`/acquire connection | Block thread จนกว่าจะมี connection ว่าง | `.await` แบบ cooperative (ไม่ block thread) |
| ใช้ใน Axum handler อย่างไร | ต้องผ่าน `spawn_blocking` เสมอ | ใช้ตรงในบล็อก `async fn` ได้เลย |
| การตั้งค่า pool size/timeout | ผ่าน `r2d2::Builder` (`max_size`, `min_idle`, `connection_timeout`) | ผ่าน `PgPoolOptions` (`max_connections`, `min_connections`, `acquire_timeout`) — concept เดียวกัน ชื่อ method ต่างกันเล็กน้อย |

#### ใช้ Diesel pool ใน Axum handler อย่างถูกวิธี (ผสาน `spawn_blocking` จากหัวข้อ 72.2)

```rust
use axum::{extract::State, routing::get, Json, Router};
use diesel::prelude::*;
use diesel::r2d2::{ConnectionManager, Pool, PooledConnection};
use diesel::pg::PgConnection;

type DbPool = Pool<ConnectionManager<PgConnection>>;

#[derive(Clone)]
struct AppState {
    pool: DbPool,
}

// handler แบบเต็มรูปแบบที่รวมทุกอย่างที่หัวข้อ 72.2/72.10 สอนไว้: pool + spawn_blocking
async fn list_available_books(
    State(state): State<AppState>,
) -> Result<Json<Vec<Book>>, AppError> {
    let pool = state.pool.clone(); // Pool<ConnectionManager<...>> ภายในคือ Arc อยู่แล้ว — clone ถูกเสมอ (หลักการเดียวกับที่ Part 70 อธิบาย PgPool)

    let result = tokio::task::spawn_blocking(move || -> QueryResult<Vec<Book>> {
        use crate::schema::books::dsl::*;

        // pool.get() เป็น blocking call เช่นกัน — ต้องอยู่ใน spawn_blocking ด้วย ไม่ใช่แค่ query
        let mut conn: PooledConnection<ConnectionManager<PgConnection>> = pool
            .get()
            .map_err(|e| diesel::result::Error::QueryBuilderError(Box::new(e)))?;

        books
            .filter(available_copies.gt(0))
            .order(title.asc())
            .select(Book::as_select())
            .load(&mut conn)
    })
    .await
    .map_err(|_| AppError::Internal("spawn_blocking panicked".into()))?
    .map_err(AppError::from)?;

    Ok(Json(result))
}

fn app(pool: DbPool) -> Router {
    Router::new()
        .route("/books/available", get(list_available_books))
        .with_state(AppState { pool })
}
```

**ทดสอบจริง**: ผู้เขียนรัน Axum server นี้จริง (ต่อกับฐานข้อมูล `diesel_course_scratch` หลังผ่านการ borrow/delete จากหัวข้อ 72.6 มาแล้ว — เหลือหนังสือที่ `available_copies > 0` แค่เล่มเดียวคือ "Programming Rust" ที่ `available_copies` ลดจาก 2 เหลือ 1) และยิง `curl http://127.0.0.1:38080/books/available` ได้ผลลัพธ์ JSON จริงกลับมา:

```json
[{"id":1,"isbn":"978-1-59327-828-1","title":"Programming Rust","author":"Jim Blandy",
"total_copies":3,"available_copies":1,"published_year":2021,"shelf_id":2,
"created_at":"2026-09-27T00:43:26.128071Z"}]
```

ผลลัพธ์นี้สอดคล้องกับสถานะข้อมูลจริงที่ทดสอบไว้ในหัวข้อ 72.6 ทุกจุด (หนังสือเล่มที่ `available_copies = 0` และเล่มที่ถูก `DELETE` ไปแล้วไม่ปรากฏใน response ตามที่คาด) — ยืนยันว่า pattern `spawn_blocking` + `r2d2` pool ทำงานได้จริงกับ Axum ตามที่บทนี้อธิบายไว้ทุกขั้นตอน ทั้ง `serde::Serialize` บน `Book` struct ก็ทำงานคู่กับ `Queryable`/`Selectable` ได้ปกติตามที่หัวข้อ 72.5 อธิบายไว้ (derive สองระบบบน struct เดียวกัน)

**สิ่งที่ต้องสังเกต**: ทุก handler ที่ใช้ Diesel ในระบบจริงจะมีรูปแบบซ้ำ ๆ แบบนี้เสมอ (`spawn_blocking` + `pool.get()` + query + `.await` สองชั้น) ต่างจาก SQLx handler ของ Part 70 ที่เขียน `pool.fetch_all(...).await?` บรรทัดเดียวตรง ๆ ในบล็อก async ได้เลย — นี่คือ**ภาระทางไวยากรณ์ (syntactic overhead) ที่แลกมากับการได้ query DSL แบบ type-safe เต็มรูปแบบของ Diesel** ในโปรเจกต์จริงหลายทีมเขียน helper function กลาง ๆ ที่ห่อ pattern นี้ไว้ให้ (เช่น `async fn run_blocking<F, T>(pool: DbPool, f: F) -> Result<T, AppError> where F: FnOnce(&mut PgConnection) -> QueryResult<T> + Send + 'static, T: Send + 'static`) เพื่อไม่ต้องเขียน `spawn_blocking` boilerplate ซ้ำทุก handler

### 72.11 เมื่อไหร่ควรเลือก Diesel เมื่อไหร่ควรเลือก SQLx

ถึงจุดนี้คุณเห็นทั้งสองฝั่งทำงานจริงบนโดเมนเดียวกันแล้ว (Part 70-71 กับ SQLx, บทนี้กับ Diesel) มาสรุปเป็นหลักการตัดสินใจที่ใช้ได้จริงในโปรเจกต์จริง — **ไม่มีคำตอบที่ถูกเสมอไปทุกสถานการณ์** ทั้งสองมี trade-off จริงที่ชัดเจน:

**เลือก Diesel เมื่อ:**

- ทีมให้ความสำคัญกับ **compile-time safety สูงสุด** และยอมรับที่จะไม่เขียน SQL string เลยแม้แต่บรรทัดเดียว (DSL ทั้งหมด) — เหมาะกับทีมที่กังวลเรื่อง SQL string พิมพ์ผิดโดยไม่มีใครจับได้จนถึง production
- แอปเป็น **synchronous โดยธรรมชาติอยู่แล้ว** (เช่น CLI tool, batch job, background worker ที่ไม่ได้รันบน async runtime เลย) — ในสถานการณ์นี้ปัญหาเรื่อง `spawn_blocking`/runtime starvation **ไม่มีอยู่จริง** เพราะไม่มี async runtime ให้ starve ตั้งแต่ต้น Diesel จึงเป็นตัวเลือกที่ตรงไปตรงมากว่า (ไม่ต้องแบกความซับซ้อนของ async มาเปล่า ๆ)
- ทีมมีนักพัฒนาที่คุ้นเคย ORM แบบมี schema-first workflow มาก่อน (เช่นจากภาษาอื่นที่มี pattern คล้ายกัน) และต้องการ ecosystem ที่ **mature และเสถียรมานาน** (Diesel อยู่ในวงการมาตั้งแต่ก่อน async Rust จะแพร่หลาย ผ่านการทดสอบจากโปรเจกต์จริงจำนวนมากมายาวนาน)
- ต้องการ query ที่ **compose กันได้ในระดับ type** (เขียนฟังก์ชันที่รับ/คืนค่าเป็น "query ที่ยังไม่รัน" แล้วเอาไปต่อ `.filter()`/`.order()` เพิ่มได้อีกในฟังก์ชันอื่น) — DSL ของ Diesel รองรับ pattern นี้ได้เป็นธรรมชาติกว่าการต่อ SQL string ของ SQLx (ที่ต้องใช้ `QueryBuilder` ถ้าต้องการ dynamic SQL แบบปลอดภัย ตามที่ Part 71 สอน)

**เลือก SQLx เมื่อ:**

- แอปเป็น **async-native เต็มรูปแบบ** (เว็บแอปที่รับ concurrent request จำนวนมาก อย่างที่หลักสูตรนี้สร้างมาตั้งแต่ Part 62 ด้วย Axum) — ไม่ต้องแบกภาระ `spawn_blocking` boilerplate ในทุก handler เลย เขียน `.await` ตรงไปตรงมาได้ตลอด
- ทีมคุ้นเคย SQL อยู่แล้วและต้องการ**ควบคุม query เต็มรูปแบบ** (SQL ที่ซับซ้อนมาก ๆ เช่น window function, CTE ซับซ้อนหลายชั้น, database-specific feature) — เขียน SQL ตรง ๆ มักตรงไปตรงมากว่าพยายามหา DSL ที่ตรงกับความต้องการเป๊ะ
- ต้องการ dependency footprint ที่เป็น **pure Rust ล้วน ๆ** ไม่มี C library (`libpq`) มาเป็นภาระ build/deploy (สำคัญมากในสถานการณ์ cross-compile หรือ container image ที่อยากให้เล็กและ build เร็ว)
- ทีมต้องการ error message จาก compile-time check ที่ **อ่านง่ายและสั้นกว่า** ในกรณีส่วนใหญ่ (macro error ของ SQLx ออกแบบมาเฉพาะทาง ต่างจาก generic trait bound error ทั่วไปของ Diesel)

**ตารางเช็คลิสต์ตัดสินใจแบบเร็ว** (ใช้ตอบคำถามต่อไปนี้กับโปรเจกต์จริงที่กำลังตัดสินใจ — ถ้าตอบ "ใช่" ข้อไหนมาก ให้เอียงไปทางฝั่งนั้น):

| คำถาม | ตอบ "ใช่" เอียงไปทาง |
|---|---|
| แอปนี้เป็นเว็บเซิร์ฟเวอร์ async (Axum/Actix-web) ที่ต้องรองรับ concurrent request จำนวนมาก? | SQLx |
| แอปนี้เป็น CLI/batch job/background worker ที่ไม่มี async runtime อยู่แล้ว? | Diesel |
| ทีมมี SQL ที่ซับซ้อนมาก (window function, CTE หลายชั้น, database-specific extension) ที่ต้องคุมเองเต็มที่? | SQLx |
| ทีมกังวลเรื่อง SQL string พิมพ์ผิดที่ไม่มีใครจับได้ก่อน production มากกว่าความยืดหยุ่นของ SQL ดิบ? | Diesel |
| Build/deploy pipeline ต้องการ pure Rust ล้วน ๆ ไม่มี C library (`libpq`) เป็นภาระ (เช่น cross-compile ไป target แปลก ๆ)? | SQLx |
| ทีมมีพื้นฐาน ORM แบบ schema-first จากภาษาอื่นมาก่อน และอยากได้ compile-time DSL type-safety เต็มรูปแบบ? | Diesel |
| ต้องการ error message จาก compile-time check ที่อ่านง่ายที่สุดในกรณีทั่วไป? | SQLx |
| ต้องการ compose query เป็นฟังก์ชันย่อย ๆ ที่ต่อกันได้ในระดับ type (ไม่ใช่แค่ต่อ string)? | Diesel |

**สรุปจุดยืนของหลักสูตรนี้**: เหมือนที่ Part 69 อธิบายไว้ตอนเลือก Axum เป็น framework หลัก (แม้จะสอน Actix-web ควบคู่เพื่อให้เห็นภาพเทียบกัน) หลักสูตรนี้**เลือกสอน SQLx เป็นตัวหลักที่ใช้ต่อเนื่องในบทถัดไปทั้งหมด** (Authentication, WebSockets, Microservices ฯลฯ) เพราะสอดคล้องกับ async-native ecosystem ที่ทั้งหลักสูตรสร้างมาตั้งแต่ Part 46 เรื่อง async/await — Diesel (บทนี้) และ SeaORM (Part 73 ถัดไป) สอนไว้เพื่อให้คุณ**เข้าใจภาพรวมของทางเลือกในวงการจริง** และเลือกใช้ได้อย่างมีข้อมูลถ้าไปเจอโปรเจกต์จริงที่ใช้ตัวใดตัวหนึ่งอยู่แล้ว หรือถ้าโปรเจกต์ในอนาคตของคุณมีเงื่อนไขที่เข้ากับจุดแข็งของ Diesel/SeaORM มากกว่า

**สิ่งที่รอใน Part 73**: SeaORM เป็นตัวเลือกที่สามที่น่าสนใจมาก เพราะมันเป็น **async-native เหมือน SQLx** (สร้างอยู่บน SQLx เป็น driver ชั้นล่างจริง ๆ ตามที่ Part 70 กล่าวไว้) **แต่**ให้ประสบการณ์แบบ **Entity/ActiveRecord** ที่ใกล้เคียง ORM ดั้งเดิมในภาษาอื่นมากกว่า Diesel (`model.title = "...".to_string(); model.save(db).await?` ในลักษณะที่คุ้นเคยกว่าสำหรับคนที่มาจาก ORM ภาษาอื่น) — เป็นการผสมข้อดีของทั้งสองแนวทางที่เห็นมาแล้วในระดับหนึ่ง แต่ก็มี trade-off ของตัวเองที่ Part 73 จะอธิบายต่อ

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. เรียก Diesel query ตรง ๆ ในบล็อก `async fn` โดยไม่ผ่าน `spawn_blocking`

ปัญหานี้อธิบายไว้เต็มที่แล้วในหัวข้อ 72.2 — โค้ดแบบนี้ **compile ผ่านสนิท ไม่มี error หรือ warning ใด ๆ เลย**:

```rust
async fn list_books_bad(State(pool): State<DbPool>) -> Json<Vec<Book>> {
    let mut conn = pool.get().unwrap();
    let all = crate::schema::books::table.load::<Book>(&mut conn).unwrap();
    Json(all)
}
```

**อาการที่เจอจริงตอน production**: ไม่มี error message ให้เห็นตรง ๆ เลย — สิ่งที่สังเกตได้คือ **latency ของทุก endpoint ในระบบพุ่งสูงขึ้นพร้อมกัน**เมื่อ traffic สูงขึ้น (แม้ endpoint ที่ไม่เกี่ยวกับฐานข้อมูลเลยก็ช้าไปด้วย) เพราะ Tokio worker thread ทั้งหมดถูกยึดโดย query ที่ block อยู่ — วิธีวินิจฉัย: เปิด metrics ของ Tokio runtime (เช่นผ่าน `tokio-console` ที่เป็นเครื่องมือ debug async runtime โดยเฉพาะ) จะเห็น task ที่ "ค้าง" อยู่บน worker thread นานผิดปกติโดยไม่มีการ poll สลับเลย — **วิธีแก้**: ห่อทุก Diesel call ด้วย `tokio::task::spawn_blocking` ตามที่หัวข้อ 72.2/72.10 สอนไว้เสมอ ไม่มีข้อยกเว้น

### 2. `schema.rs` ไม่ sync กับฐานข้อมูลจริงหลัง migration ใหม่

ถ้ารัน migration ที่เพิ่มคอลัมน์ใหม่แล้ว**ลืม**รัน `diesel print-schema` (หรือไม่ได้ตั้งค่า `diesel.toml` ให้ทำอัตโนมัติตามหัวข้อ 72.4) แล้วเขียนโค้ดอ้างถึงคอลัมน์ใหม่นั้น จะได้ error แบบเดียวกับกรณี "พิมพ์ชื่อคอลัมน์ผิด" ในหัวข้อ 72.9 (เพราะในมุมของ Rust compiler มันคือ identifier ที่ไม่มีอยู่จริงเหมือนกัน — schema.rs เก่ายังไม่รู้จักคอลัมน์นั้น):

```
error[E0425]: cannot find value `category` in this scope
  --> src/main.rs:60:20
   |
60 |     books.filter(category.eq("fiction"))
   |                  ^^^^^^^^ not found in this scope
```

**วิธีแก้**: รัน `diesel print-schema > src/schema.rs` (หรือคำสั่งเดียวกับที่ `diesel.toml` ตั้งไว้) ทุกครั้งหลัง migration ที่เปลี่ยน schema เสมอ — ควรตั้งเป็นขั้นตอนบังคับในทีม (เช่นใส่ไว้ใน CI ให้ตรวจว่า `schema.rs` ตรงกับผลลัพธ์ `diesel print-schema` จริงเสมอ ป้องกันคนลืม commit ไฟล์ที่ sync แล้ว)

### 3. ลืม `.select(Model::as_select())` ทำให้ `Queryable` เดี่ยว ๆ พังเงียบ ๆ เมื่อลำดับคอลัมน์เปลี่ยน

```rust
#[derive(Queryable)] // ไม่มี Selectable
struct Book { id: i64, isbn: String, title: String /* ... */ }

fn broken(conn: &mut PgConnection) -> QueryResult<Vec<Book>> {
    // เขียน SELECT เอง สลับลำดับ isbn กับ title โดยไม่ตั้งใจ (พิมพ์ผิดลำดับ)
    diesel::sql_query("SELECT id, title, isbn FROM books").load(conn)
    // Book struct คาดหวังลำดับ (id, isbn, title) แต่ query นี้คืน (id, title, isbn)
}
```

โค้ดนี้ (ถ้าผ่าน `sql_query` แบบ raw SQL ที่ไม่ผูกกับ DSL type-check เต็มรูปแบบ) **compile ผ่านได้** เพราะ `id: i64`/`isbn: String`/`title: String` ทุกตัวเป็น `String`/`i64` ที่ type ตรงกันหมด (แค่สลับตำแหน่งกัน) — Diesel deserialize ตาม**ลำดับ**ไม่ใช่ตามชื่อ (ตามที่หัวข้อ 72.5 อธิบาย) ผลคือได้ข้อมูล `isbn`/`title` สลับกันโดยไม่มี error ใด ๆ เตือนเลย เป็นบั๊กเชิงตรรกะที่เงียบมากและอันตราย — **วิธีแก้**: ใช้ `.select(Book::as_select())` คู่กับ query DSL ปกติ (ไม่ใช่ raw `sql_query`) เสมอเมื่อทำได้ เพราะมันการันตีว่ารายชื่อคอลัมน์ที่ query จริง (`SELECT "books"."id", "books"."isbn", "books"."title", ...`) ถูก generate ให้ตรงกับลำดับ field ใน struct เป๊ะโดยอัตโนมัติ ไม่ต้องพึ่งการเขียน SQL เองให้ลำดับตรงด้วยมือเลย

### 4. เปิด `r2d2` pool ขนาดเล็กเกินไปแล้วเจอ deadlock เชิง throughput เมื่อรวมกับ `spawn_blocking`

```rust
Pool::builder().max_size(2).build(manager) // max_size เล็กเกินไปสำหรับ traffic จริง
```

ถ้า `max_size` ของ `r2d2` pool เล็กเกินไป (เทียบกับจำนวน concurrent request ที่ต้องคุยกับฐานข้อมูลพร้อมกันจริง) ทุก request ที่เกินจำนวนนี้จะไปกอง**รอ**อยู่ที่ `pool.get()` (block thread ของ `spawn_blocking` thread pool ตามที่หัวข้อ 72.10 อธิบาย) — ปัญหาคือ `spawn_blocking` thread pool ของ Tokio (ต่างจาก worker thread ของ async scheduler) **ขยายได้แต่ไม่ใช่ไม่จำกัดไปตลอด** ถ้ามี request จำนวนมากพอที่ต่างก็ค้างรอ `pool.get()` พร้อมกัน (เพราะ `max_size` เล็กเกิน) อาจไปถึงขีดจำกัดของ `spawn_blocking` pool เองได้ในระบบที่ traffic สูงมาก ๆ (ค่า default ของ Tokio คือ 512 blocking thread — ตัวเลขที่มากพอสมควรแต่ไม่ใช่ไม่จำกัด) — อาการที่เจอ: request timeout จำนวนมากพร้อมกันแบบ "ระเบิด" (cascading) แทนที่จะ degrade อย่างค่อยเป็นค่อยไป — **วิธีแก้**: ตั้ง `max_size` ของ `r2d2` ให้เหมาะกับ traffic จริง (คล้ายหลักการเดียวกับ `max_connections` ของ `PgPoolOptions` ใน Part 70) และตั้ง `connection_timeout` ให้เหมาะสม (`r2d2::Builder::connection_timeout(Duration)`) เพื่อให้ request ที่รอนานเกินไป **fail เร็ว** ด้วย error ที่ชัดเจน (แปลงเป็น 503 Service Unavailable) แทนปล่อยให้ค้างจนกระทบ thread pool ทั้งระบบ

### 5. ระบุ `#[diesel(sql_type = ...)]` ผิดใน `QueryableByName` — compile ผ่านแต่พังตอน runtime

ต่อยอดจากหัวข้อ 72.6 เรื่อง `sql_query`/`QueryableByName` — เพราะ escape valve นี้ไม่มีการตรวจสอบกับฐานข้อมูลจริงตอน compile เลย (ตามที่อธิบายไว้ว่าสูญเสีย compile-time safety ไปเกือบทั้งหมด) การระบุ SQL type ผิดจะ **compile ผ่านสนิท** แล้วไปพังตอน runtime ทันทีที่ query จริงถูกเรียก:

```rust
#[derive(Debug, QueryableByName)]
struct WrongType {
    #[diesel(sql_type = diesel::sql_types::BigInt)] // ผิด: shelves.code เป็น TEXT ไม่ใช่ BigInt
    code: i64,
}

let result = diesel::sql_query("SELECT code FROM shelves LIMIT 1").load::<WrongType>(&mut conn);
```

**Error จริงตอน runtime** (ทดสอบจริง):

```
Err(
    DeserializationError(
        "Received less than 8 bytes while decoding an i64. Was an Integer expression accidentally marked as BigInt?",
    ),
)
```

สังเกตว่า Diesel **พยายามช่วยเดาสาเหตุให้ในตัว error message เอง** (`Was an Integer expression accidentally marked as BigInt?`) เพราะรูปแบบ error นี้พบบ่อยพอที่ทีม Diesel จะเขียน hint เฉพาะทางไว้ — แต่ก็ยังเป็นแค่ **runtime error ธรรมดา** (ไม่ใช่ compile error) ต่างจาก DSL ปกติที่จับปัญหาแบบเดียวกันได้ตั้งแต่ตอน `cargo build` (หัวข้อ 72.9) — **วิธีแก้**: ตรวจสอบ SQL type ของทุกคอลัมน์ที่ใช้กับ `QueryableByName` ให้ตรงกับ `schema.rs` เสมอ (เปิด `schema.rs` เทียบมือทุกครั้งที่เขียน raw query ใหม่) และจำกัดการใช้ `sql_query` ให้น้อยที่สุดเท่าที่จำเป็นจริง ๆ ตามคำแนะนำในหัวข้อ 72.6

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เพิ่มฟังก์ชัน `find_book_by_isbn(conn: &mut PgConnection, target_isbn: &str) -> QueryResult<Option<Book>>` ที่ query ด้วย `.filter(isbn.eq(target_isbn))` แล้วใช้ `.first::<Book>(conn).optional()` (method ของ Diesel ที่แปลง `NotFound` error เป็น `Ok(None)` โดยอัตโนมัติ แทนต้อง `match` เองกับ `diesel::result::Error::NotFound`) — Hint: `.optional()` เรียกต่อจาก `.first(conn)` ได้ตรง ๆ เปลี่ยน `QueryResult<Book>` เป็น `QueryResult<Option<Book>>` ให้อัตโนมัติ

2. **(กลาง)** เขียน handler Axum `PUT /books/{id}/shelf` ที่รับ JSON body `{ "shelf_id": 3 }` แล้ว `UPDATE books SET shelf_id = $1 WHERE id = $2` ผ่าน DSL ของ Diesel — ต้องห่อด้วย `spawn_blocking` ให้ถูกต้องตามหัวข้อ 72.2/72.10 ทั้งหมด และแปลง error กรณีไม่มี `id` นั้นจริง (`.get_result()` คืน `NotFound`) เป็น HTTP 404 — Hint: ใช้ `.returning(Book::as_select()).get_result(&mut conn)` แล้ว match `Err(diesel::result::Error::NotFound)` แปลงเป็น `AppError::NotFound` เหมือนที่ Part 66 สอนไว้กับ `sqlx::Error::RowNotFound`

3. **(ยาก)** สร้าง migration ใหม่ที่เพิ่มตาราง `authors` (`id`, `name`, `country`) แล้วเปลี่ยนคอลัมน์ `books.author` (ที่เดิมเป็น `TEXT` ตรง ๆ) ให้เป็น `author_id BIGINT REFERENCES authors(id)` แทน (ต้องเขียน migration ที่ migrate ข้อมูลเดิมด้วย — insert ชื่อผู้เขียนที่ไม่ซ้ำเข้า `authors` ก่อน แล้วค่อย backfill `author_id` ของ `books` จากชื่อที่ตรงกัน) — จากนั้นเขียนฟังก์ชันที่ join `books` กับ `authors` ผ่าน `.inner_join()` แบบหัวข้อ 72.8 พร้อมรัน `diesel print-schema` ใหม่ให้ `joinable!` ประกาศความสัมพันธ์นี้ให้ถูกต้อง — Hint: migration ที่ต้อง "ย้ายข้อมูล" ระหว่างเปลี่ยน schema ต้องเขียนเป็น SQL DML (`INSERT INTO ... SELECT DISTINCT ...`, `UPDATE ... SET author_id = ...`) ผสมกับ DDL ในไฟล์ migration เดียวกันได้ตามปกติ (SQL ไฟล์ migration รันเป็นคำสั่งลำดับต่อกันเรื่อย ๆ ไม่จำกัดว่าต้องเป็น DDL ล้วน ๆ)

4. **(ยาก/ประยุกต์ใช้งานจริง)** เขียน integration test (ตามแนวทาง Part 32-33) ที่พิสูจน์ปัญหาในหัวข้อ "กับดักที่พบบ่อย" ข้อ 1 อย่างเป็นรูปธรรม: สร้าง Axum server ที่มี **สอง endpoint** — endpoint แรก (`/blocking-bug`) เรียก Diesel query ตรง ๆ ในบล็อก async **โดยไม่ผ่าน `spawn_blocking`** (เขียนแบบผิดตั้งใจ) และ endpoint ที่สอง (`/health`) แค่คืน `"ok"` ทันทีไม่แตะฐานข้อมูลเลย จากนั้นยิง request จำนวนมากพร้อมกันไปที่ `/blocking-bug` (ที่ query ช้าโดยตั้งใจ เช่นใส่ `pg_sleep(1)` ใน SQL raw ผ่าน `sql_query`) ด้วย `tokio::join!`/`futures::future::join_all` พร้อมกับยิง `/health` ไปด้วยพร้อมกัน แล้ววัดเวลาที่ `/health` ตอบกลับ — เทียบกับเวลาที่ `/health` ตอบกลับตอนที่ endpoint แรกใช้ `spawn_blocking` อย่างถูกต้อง (ไม่ใส่ `pg_sleep`) — Hint: ต้องจำกัดจำนวน Tokio worker thread ให้น้อย ๆ ก่อน (เช่นตั้ง `#[tokio::main(worker_threads = 2)]`) เพื่อให้เห็นผลกระทบชัดเจนขึ้นในเครื่องทดสอบที่มี CPU core จำนวนมาก (ถ้า worker thread เยอะเกินไป อาจต้องยิง concurrent request จำนวนมากกว่าจะเห็นผลกระทบจริง)

## สรุป

บทนี้แนะนำ Diesel ในฐานะแนวทางที่สองในการคุยกับ PostgreSQL จาก Rust — ต่างจาก SQLx (Part 70-71) โดยพื้นฐาน ไม่ใช่แค่ syntax ที่ต่างกัน:

- **Diesel ตรวจสอบ query ผ่าน Rust type system เอง** โดยเขียน query เป็น **Rust DSL** (method chain) ที่ compile ผ่าน trait bound ธรรมดา ไม่ต้องมี connection ไปยังฐานข้อมูลตอน compile เลย ต่างจาก SQLx ที่เขียน SQL string จริงแล้วให้ macro ไปตรวจกับฐานข้อมูล/cache
- **Diesel เป็น synchronous by default** — ทุก query เป็น blocking call จริง **ต้องห่อด้วย `tokio::task::spawn_blocking` เสมอ** เมื่อใช้ในบล็อก `async fn` ของ Axum ไม่มีข้อยกเว้น มิฉะนั้นจะเกิด runtime starvation ที่กระทบทั้งระบบ (`diesel-async` มีให้เป็นทางเลือก async-native แต่ไม่ใช่ core ของ ecosystem)
- **`schema.rs` (generate ผ่าน `diesel print-schema`) คือ single source of truth ที่ query DSL ทั้งหมด type-check ผ่าน** — ต้อง sync กับฐานข้อมูลจริงเสมอหลัง migration ทุกครั้ง (ควรตั้งให้ `diesel migration run` generate ไฟล์นี้ให้อัตโนมัติผ่าน `diesel.toml`)
- **Model แยกเป็นสองชุดตามธรรมเนียม**: `Queryable`/`Selectable` สำหรับอ่าน (มีครบทุกคอลัมน์ รวม `id`/`created_at`), `Insertable` สำหรับเขียนข้อมูลใหม่ (ไม่มีคอลัมน์ที่ database generate ให้เอง) — ทั้งสองเป็น derive macro ที่ generate `impl` คนละ trait จาก `serde::Serialize`/`Deserialize` โดยสิ้นเชิง แต่ทำงานคู่กันบน struct เดียวได้
- **CRUD ผ่าน DSL** (`.filter()`, `.select()`, `.order()`, `.limit()`, `.values()`, `.set()`) generate SQL ที่ parameterize ปลอดภัยโดยอัตโนมัติเสมอ (พิสูจน์ได้จริงผ่าน `diesel::debug_query!`) รวมถึง **batch insert** (`.values(&vec_of_structs)` ได้ `INSERT ... VALUES (...), (...)` ในคำสั่งเดียว) และ **upsert** (`.on_conflict().do_update().set(...)` ตรงกับ `ON CONFLICT ... DO UPDATE` ของ PostgreSQL) — join ระหว่างตารางใช้ `.inner_join()` ร่วมกับ `diesel::joinable!` ที่ประกาศความสัมพันธ์ไว้ใน `schema.rs`
- **`diesel::result::Error`** มี variant ที่ต้องแยกจัดการเหมือน `sqlx::Error` ใน Part 70: `NotFound` (→ 404), `DatabaseError(DatabaseErrorKind::UniqueViolation, ...)` (→ 409) พร้อมรายละเอียด constraint ที่ละเมิดผ่าน `.message()`/`.details()`/`.constraint_name()` — แปลงผ่าน `impl From<diesel::result::Error> for AppError` ตาม pattern เดียวกับ Part 66
- **`conn.transaction(|conn| { ... })`** การันตี atomicity เหมือน `pool.begin()`/`commit()`/`rollback()` ของ SQLx แต่ผูกกับ closure/return value แทน (`Ok` → commit, `Err` → rollback อัตโนมัติ) — พิสูจน์ rollback จริงด้วยตัวเลขก่อน/หลัง และ `begin_test_transaction()` ใช้หลักการเดียวกันสำหรับ test ที่ไม่ทิ้งข้อมูลตกค้าง (เบากว่า `#[sqlx::test]` ของ SQLx แต่ isolation อยู่ระดับ transaction ไม่ใช่ระดับฐานข้อมูล)
- **`diesel::sql_query`/`QueryableByName`** คือ escape valve สำหรับ SQL ที่ DSL เขียนไม่ได้ — แลกกับการสูญเสีย compile-time safety เกือบทั้งหมด (ระบุ SQL type ผิดจะ compile ผ่านแต่พังตอน runtime) ควรใช้ให้น้อยที่สุด
- **Error จาก query ที่ผิด** อ้างคอลัมน์ที่ไม่มีจริงให้ error สั้นและอ่านง่าย (เพราะเป็น "cannot find value" ธรรมดาของ Rust) แต่เทียบ type ผิดให้ error ที่ยาวและซับซ้อนกว่า (generic trait bound error ที่อาจมาพร้อมกันหลายก้อน) — วิธีอ่านคือมองหา error ก้อนแรกและ "the trait bound ... is not satisfied" ก่อนเสมอ ก้อนที่เหลือส่วนใหญ่เป็นผลพลอยได้
- **Diesel ไม่มี pool ในตัว** ต้องพึ่ง `r2d2` (synchronous pool ทั่วไป) ผ่าน `ConnectionManager` — ทั้ง pool และ query เป็น sync ทั้งชุด ต้องอยู่ใน `spawn_blocking` เสมอเมื่อใช้กับ Axum
- **การเลือก Diesel กับ SQLx เป็น trade-off จริง ไม่มีคำตอบตายตัว** — Diesel เหมาะกับงาน sync-native หรือทีมที่ต้องการ DSL type-safe เต็มรูปแบบ ส่วน SQLx เหมาะกับเว็บแอป async-native ที่ต้องการควบคุม SQL เต็มที่ (หลักสูตรนี้ใช้ SQLx เป็นตัวหลักต่อจากนี้)

Part ถัดไป (Part 73) จะแนะนำ **SeaORM** — ตัวเลือกที่สามที่ผสมข้อดีของทั้งสองฝั่งในระดับหนึ่ง: async-native เหมือน SQLx (เพราะสร้างอยู่บน SQLx เป็น driver ชั้นล่างจริง) แต่ให้ประสบการณ์แบบ Entity/ActiveRecord ที่ใกล้เคียง ORM ดั้งเดิมมากกว่า Diesel — ปิดท้ายภาพรวมของทางเลือก ORM/query builder ทั้งสามตัวในวงการ Rust ก่อนหลักสูตรจะเดินหน้าต่อไปยัง Authentication ใน Part 74

---

**Part ก่อนหน้า:** [SQLx: Queries, Migrations, Connection Pooling](part-071-sqlx-queries-migrations.md) | **Part ถัดไป:** [SeaORM เบื้องต้น](part-073-seaorm.md)
