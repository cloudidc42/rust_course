# Part 81: Microservices Architecture ด้วย Rust

> โมดูล: การพัฒนาเว็บแอปพลิเคชัน (Web Development) | ระดับ: มืออาชีพ | เวลาโดยประมาณ: 300 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้อย่างตรงไปตรงมาว่า**อะไรเปลี่ยนไปจริง ๆ** เมื่อแตก monolith ออกเป็น microservices (deployment independence, การเลือกเทคโนโลยี/scale แยกต่อ service) เทียบกับ**สิ่งที่ต้องแลกมา** (network call overhead, ความซับซ้อนของ distributed systems, ภาระงาน operations ที่เพิ่มขึ้น) — และรู้ว่า microservices เป็น**เครื่องมือแก้ปัญหาองค์กร/scaling** ไม่ใช่ "แนวทางที่ดีกว่าเสมอ"
- แตกโดเมนห้องสมุด/ระบบจองที่ใช้ตลอดหลักสูตรออกเป็น service จริงตามหลัก **data ownership** (แต่ละ service เป็นเจ้าของฐานข้อมูลตัวเอง ไม่มีการแชร์ตารางข้าม service) พร้อมอธิบายเหตุผลว่าทำไมการแชร์ database ข้าม service จึงสร้างความผูกติดกันแบบเดียวกับ monolith ขึ้นมาใหม่
- เลือกวิธีสื่อสารระหว่าง service ได้ถูกต้องระหว่าง **synchronous (gRPC จาก Part 80)** กับ **asynchronous messaging (ที่ Part 82 จะสอนเต็มรูปแบบ)** ตามลักษณะของงานจริง
- สร้างระบบสอง service ที่**รันได้จริง**: `catalog-service` (gRPC server ด้วย Tonic + SQLx/PostgreSQL) และ `booking-service` (REST API ด้วย Axum ที่เรียก `catalog-service` ผ่าน generated Tonic client) — พร้อมพิสูจน์ด้วย `curl` จริงที่ทำให้เกิดการเรียกข้าม service จริงทั้งสาย
- ส่งต่อ identity ข้าม service ด้วย RS256 JWT (ต่อจาก Part 74) ผ่าน gRPC metadata (ต่อจาก Part 80) ให้ service ปลายทางตรวจสอบสิทธิ์ได้เองโดยไม่ต้องเรียกกลับไปยัง auth service
- ใส่ resilience pattern จริง (timeout, retry with exponential backoff) ให้กับการเรียกข้าม service พร้อมรู้จัก circuit breaker ในระดับแนวคิด และรู้ตำแหน่งของ correlation ID / distributed tracing ในระบบแบบนี้ (ต่อจาก Part 60, ปูทางไป Part 99)
- อธิบาย Saga pattern สำหรับปัญหา "transaction ข้าม service ทำไม่ได้" (ต่อจาก Part 70) ด้วยตัวอย่าง reserve-then-confirm-or-release จริง และประเมินได้ว่าเมื่อไหร่ระบบของคุณควรอยู่แบบ monolith (Part 62-78) เมื่อไหร่ควรแตกเป็น microservices

## ความรู้ที่ต้องมีมาก่อน

- **Part 17 (Packages, Crates, Workspaces)**: บทนี้ใช้แนวคิด Cargo workspace เป็น **ขั้นแรกที่ควรทำก่อนแตกเป็น microservices จริง** — โมดูลาไรซ์โค้ดให้ดีภายใน workspace เดียวก่อน แล้วค่อยแตกเป็น service จริงทีหลังถ้ามีเหตุผลที่หนักแน่นพอ (หัวข้อ 81.1 จะอธิบายลำดับนี้ละเอียด)
- **Part 60 (Logging และ Tracing เบื้องต้น)**: correlation ID / request ID ที่บทนั้นสอนไว้ภายใน process เดียว จะถูกขยายในหัวข้อ 81.8 ให้ไหลข้าม service boundary ผ่าน gRPC metadata — เป็นรากฐานสำคัญของหัวข้อ distributed tracing เต็มรูปแบบที่ Part 99 จะสอน
- **Part 61-66 (HTTP Fundamentals, Axum พื้นฐานถึง Error Handling)**: `booking-service` ในบทนี้คือ Axum REST API ตัวหนึ่งที่สร้างตาม pattern ทั้งหมดที่บทเหล่านี้สอนไว้ (routing, extractors, state, error handling) เพียงแต่ handler ของมันเรียกออกไปยัง service อื่นด้วย
- **Part 65 (Axum Middleware)**: `TimeoutLayer` ที่บทนั้นสอนไว้สำหรับฝั่ง REST จะถูกเทียบกับ timeout configuration ของ Tonic client ในหัวข้อ 81.7
- **Part 70 (SQLx และ PostgreSQL)**: `catalog-service` เก็บข้อมูลหนังสือผ่าน SQLx จริง และหัวข้อ 81.9 จะอธิบายว่าทำไม `transaction` เดียวที่ Part 70 สอนไว้ (`pool.begin()`/`commit()`/`rollback()`) ใช้ครอบคลุมการดำเนินการที่ข้าม service ไม่ได้อีกต่อไป
- **Part 74 (JWT Authentication)**: หัวข้อ 81.5 ใช้แนวคิด **RS256 asymmetric signing** ที่บทนั้นแนะนำไว้เต็มรูปแบบ — private key อยู่ที่ auth-service (สมมติ) เท่านั้น ส่วน `booking-service`/`catalog-service` มีแค่ public key สำหรับตรวจสอบ
- **Part 80 (gRPC ด้วย Tonic)**: บทนี้คือบทต่อยอดโดยตรง — `.proto`, `tonic-prost-build`, การ implement server trait, interceptor สำหรับ auth ผ่าน metadata ทั้งหมดใช้ pattern เดียวกับ Part 80 ทุกประการ ถ้ายังไม่แน่นเรื่องกลไกพื้นฐานของ gRPC ควรกลับไปทวน Part 80 ก่อน เพราะบทนี้จะไม่อธิบายกลไกพื้นฐานซ้ำ
- **บทนี้เป็นจุดเริ่มต้นของโมดูล "Distributed Systems กับ Rust"** — Part 82 จะสอน message queue (RabbitMQ/Kafka) เพื่อเติมเต็มด้าน asynchronous ที่หัวข้อ 81.3 แค่ foreshadow ไว้ และ Part 96 ขึ้นไปจะสอน Docker/CI-CD/observability ที่ทำให้แนวทางในบทนี้ใช้งานได้จริงในโลก production

## หมายเหตุเรื่องการตรวจสอบเนื้อหา (สำคัญ — อ่านก่อนเริ่ม)

ทุกตัวอย่างโค้ดของทั้งสอง service ในบทนี้ผู้เขียน **build และรันจริงพร้อมกันทั้งสองตัว** ด้วย scratch project แยกไว้นอก repo ของหลักสูตร (ไม่กระทบไฟล์ใด ๆ ในคอร์สนี้เลย) โดยมี PostgreSQL จริงรันอยู่ในเครื่องเดียวกัน (ต่อเนื่องมาจาก Part 70) และ `protoc` (`libprotoc 3.21.12`) ติดตั้งไว้แล้ว (ต่อเนื่องมาจาก Part 80) — ทั้ง `curl` transcript, log output ที่มี request ID ตรงกันข้าม service, และลำดับเวลาของ retry/backoff ที่ปรากฏในบทนี้**คือผลลัพธ์ที่รันจริงแล้วคัดลอกมา** รวมถึงการทดสอบที่จงใจ **kill process ของ `catalog-service` กลางที่ `booking-service` กำลังยิง request อยู่จริง** แล้ว restart มันขึ้นมาใหม่ เพื่อพิสูจน์ว่า retry-with-backoff ทำงานได้จริงในสถานการณ์ที่จำลองมาจากการ deploy จริง ไม่ใช่แค่คำอธิบายทฤษฎี

## เนื้อหา

### 81.1 Monolith vs Microservices: อะไรเปลี่ยนไปจริง ๆ (และทำไมต้องระมัดระวังเรื่อง hype)

#### ก่อนอื่นใด: microservices ไม่ใช่ "แนวทางที่ดีกว่า" โดยอัตโนมัติ

ต้องพูดตรง ๆ ตั้งแต่บรรทัดแรกของบทนี้ เพราะเป็นความเข้าใจผิดที่พบบ่อยและสร้างความเสียหายจริงในหลายทีม: **microservices ไม่ใช่เป้าหมายในตัวเอง และไม่ใช่ "best practice" ที่ทุกระบบควรทำตาม** มันคือ**เครื่องมือแก้ปัญหาเฉพาะเจาะจง** สองปัญหาหลัก:

1. **ปัญหาองค์กร (organizational)** — เมื่อทีมวิศวกรใหญ่ขึ้นจนหลายทีมต้อง deploy โค้ดส่วนของตัวเองพร้อมกันโดยไม่รอทีมอื่น monolith เดียวที่ทุกคนต้อง merge/deploy ร่วมกันกลายเป็นคอขวด (ทีม A อยากปล่อยฟีเจอร์ใหม่วันนี้ แต่ต้องรอทีม B แก้ bug ในโค้ดของ B ให้เสร็จก่อน เพราะทั้งคู่อยู่ใน binary เดียวกัน)
2. **ปัญหา scaling ที่ไม่เท่ากัน (differential scaling)** — เมื่อบางส่วนของระบบต้องรับโหลดสูงกว่าส่วนอื่นมาก ๆ (เช่น endpoint ค้นหาหนังสือถูกเรียกหนักกว่า endpoint จัดการโปรไฟล์ผู้ใช้ 1000 เท่า) การ scale ทั้ง monolith ขึ้นพร้อมกันเพื่อรองรับแค่ส่วนเดียวเป็นการสิ้นเปลือง resource มาก

ระบบจริงจำนวนมาก **ไม่มี**ปัญหาทั้งสองข้อนี้ — ทีมมีวิศวกรไม่กี่คนถึงไม่กี่สิบคน ทุกส่วนของระบบมีโหลดใกล้เคียงกัน ไม่มีเหตุผลอะไรที่ต้องแตกเป็น microservices เลย และ**ระบบ monolith ที่ออกแบบดีตาม Part 62-78 ยังคงเป็นตัวเลือกที่ถูกต้องกว่าสำหรับระบบส่วนใหญ่ในโลก** — บริษัทเทคโนโลยีที่ประสบความสำเร็จจำนวนมาก (Shopify ในยุคแรก, StackOverflow เป็นตัวอย่างคลาสสิกที่มักถูกอ้างถึง) รันระบบหลักด้วย monolith เดียวมาเป็นเวลานานมากแม้จะมี traffic สูงมากก็ตาม เพราะปัญหาที่พวกเขาเจอไม่ใช่ปัญหาที่ microservices แก้ได้ดีกว่า

#### สิ่งที่**เปลี่ยนจริง**เมื่อแตกเป็น microservices — ทั้งด้านดีและด้านที่ต้องแลก

| มิติ | Monolith (Part 62-78) | Microservices (บทนี้) |
|---|---|---|
| **Deployment** | deploy ครั้งเดียว ทั้งระบบขึ้นพร้อมกัน (หรือลงพร้อมกันถ้า deploy พลาด) | แต่ละ service deploy แยกอิสระ — ทีม A ปล่อยเวอร์ชันใหม่ของ `catalog-service` ได้โดยไม่กระทบ `booking-service` เลย ถ้า contract (`.proto`) ไม่เปลี่ยน |
| **เทคโนโลยี/ภาษา** | ทั้งระบบใช้ stack เดียวกัน (compiler เดียว, dependency version เดียวกันทั้งหมด) | แต่ละ service เลือก stack ของตัวเองได้ (แต่บทนี้ยังใช้ Rust ทั้งสองฝั่ง เพราะจุดสนใจของหลักสูตรนี้คือ Rust) |
| **Scaling** | scale ทั้ง process พร้อมกันเสมอ (แม้จะมีแค่บางส่วนที่โหลดสูง) | scale แต่ละ service ตามโหลดของมันเองได้ — เพิ่ม instance ของ `catalog-service` โดยไม่ต้องแตะ `booking-service` เลย |
| **การเรียกข้ามโมดูล** | function call ธรรมดาในหน่วยความจำเดียวกัน — เร็ว (nanosecond), ไม่มีทางล้มเหลวเพราะ "เครือข่าย" | ต้องผ่านเครือข่ายจริงเสมอ (gRPC/HTTP) — ช้าลง (millisecond ขึ้นไป), **ล้มเหลวได้จากเหตุผลที่ไม่เกี่ยวกับ logic เลย** (packet loss, service ปลายทางล่ม, DNS ไม่ resolve) |
| **Transaction** | `BEGIN`/`COMMIT`/`ROLLBACK` เดียวครอบคลุมทั้งการดำเนินการ (Part 70) | ไม่มี transaction เดียวที่ครอบคลุมข้าม service ได้อีกต่อไป (หัวข้อ 81.9) |
| **Debugging** | stack trace เดียว เห็นทั้งเส้นทางการทำงานในที่เดียว | ต้องตามรอย request ข้าม log ของหลาย service (หัวข้อ 81.8) — ถ้าไม่มี correlation ID ดี ๆ จะ debug ยากขึ้นมาก |
| **Operational overhead** | 1 process ให้ monitor/deploy/scale | N process ให้ monitor/deploy/scale (ต้องมีเครื่องมือรองรับ — Part 96+) |

ตารางนี้คือเหตุผลที่ต้องพูดตรง ๆ ว่า **การแตกเป็น microservices ไม่ได้ "ฟรี"** — คุณแลก simplicity ของ monolith กับ independence ที่ได้มา ถ้าไม่มีปัญหาที่ independence นั้นมาแก้ คุณกำลังจ่ายต้นทุน (network overhead, distributed complexity, operational burden) โดยไม่ได้ผลตอบแทนอะไรกลับมาเลย

#### ขั้นแรกที่ควรทำก่อนแตกเป็น service จริง: Cargo Workspace จาก Part 17

นี่คือจุดที่ **Part 17 (Cargo Workspaces)** กลับมาเชื่อมกับบทนี้อย่างเป็นรูปธรรม ก่อนจะแตกโค้ดออกเป็น service คนละ process คนละ deployment กันจริง ๆ **ขั้นแรกที่ราคาถูกกว่ามากและควรทำก่อนเสมอ** คือ**โมดูลาไรซ์โค้ดภายใน monolith เดียวให้ดีก่อน** ด้วย Cargo workspace ตามที่ Part 17 สอนไว้:

```text
library_system/                  <- workspace root (มีแค่ [workspace], ไม่มี [package])
├── Cargo.toml
├── catalog/                     <- package: โดเมนหนังสือ/สต๊อก ล้วน ๆ (ไม่มี HTTP/gRPC เลย)
│   ├── Cargo.toml
│   └── src/lib.rs               <- pub struct Book, pub fn check_availability(...), ...
├── booking/                     <- package: โดเมนการจอง ล้วน ๆ
│   ├── Cargo.toml
│   └── src/lib.rs               <- pub struct Booking, pub fn create_booking(...), ...
└── api/                         <- package: binary เดียวที่รัน Axum, เรียก catalog + booking
    ├── Cargo.toml                  เป็น path dependency ตรง ๆ (function call ในหน่วยความจำเดียว)
    └── src/main.rs
```

สังเกตว่าโครงสร้างนี้**แยกขอบเขตโดเมนไว้ชัดเจนแล้ว** (module `catalog` ไม่รู้จักรายละเอียดภายในของ `booking` เลย คุยกันผ่าน public function/trait เท่านั้น) แต่**ยังเป็น binary เดียว, deploy ครั้งเดียว, transaction เดียวได้อยู่** — นี่คือสถานะที่ระบบส่วนใหญ่ควรอยู่ให้นานที่สุดเท่าที่จะทำได้ เพราะได้ประโยชน์ด้าน "ขอบเขตชัดเจน แยกความรับผิดชอบได้" ของ microservices มาบางส่วนแล้ว โดยยังไม่ต้องแลกกับ network overhead หรือ distributed transaction เลย

**สัญญาณที่บอกว่าถึงเวลาแตก module เหล่านี้ออกเป็น service จริง** (ไม่ใช่แค่ "อยากลองของใหม่"):

- มีหลายทีมที่ต้อง deploy ส่วนของตัวเองแยกจากกันจริง ๆ โดยไม่รอทีมอื่น และ deploy ร่วมกันเริ่มเป็นคอขวดที่วัดผลกระทบได้จริง (ไม่ใช่แค่ "รำคาญ")
- โหลดของแต่ละโมดูลต่างกันมากจนการ scale พร้อมกันทั้งหมดสิ้นเปลือง resource อย่างมีนัยสำคัญ (วัดเป็นตัวเลขได้ ไม่ใช่แค่คาดเดา)
- ต้องการใช้เทคโนโลยี/ภาษาต่างกันสำหรับบางส่วนของระบบด้วยเหตุผลที่หนักแน่น (เช่น ML model serving ที่ Python ecosystem แข็งแรงกว่ามาก)

หัวข้อ 81.10 ท้ายบทจะกลับมาขยายเช็คลิสต์นี้ให้ครบถ้วนยิ่งขึ้นอีกครั้งหลังจากที่คุณเห็นภาพเต็มของสิ่งที่ต้องแลกแล้ว

#### ตัวอย่าง Workspace ที่โมดูลาไรซ์แล้ว: โค้ดจริงก่อนแตกเป็น Service

เพื่อไม่ให้หัวข้อ workspace ข้างบนเป็นแค่ไดอะแกรมลอย ๆ มาดูโค้ดจริงของสถานะ "monolith ที่โมดูลาไรซ์ดีแล้ว" ก่อนตัดสินใจแตกเป็น service — สมมติทีมยังไม่มีเหตุผลหนักแน่นพอที่จะแตก `catalog`/`booking` เป็น process แยกกัน (ตามเช็คลิสต์หัวข้อ 81.1) แต่ต้องการขอบเขตความรับผิดชอบที่ชัดเจนไว้ก่อน:

```rust
// library_system/catalog/src/lib.rs
// package "catalog" -- โดเมนหนังสือ/สต๊อกล้วน ๆ ไม่รู้จัก HTTP หรือ gRPC เลย
use std::collections::HashMap;
use std::sync::Mutex;

#[derive(Debug, Clone)]
pub struct Book {
    pub id: i64,
    pub title: String,
    pub available_copies: i32,
}

#[derive(Debug)]
pub enum CatalogError {
    NotFound(i64),
    NoCopiesAvailable(i64),
}

/// ขอบเขตความรับผิดชอบของ module นี้ชัดเจนเหมือน service จริงทุกประการ -- แค่ยังเป็น
/// function call ในหน่วยความจำเดียวกัน ไม่ใช่ network call
pub struct Catalog {
    books: Mutex<HashMap<i64, Book>>,
}

impl Catalog {
    pub fn new() -> Self {
        Self { books: Mutex::new(HashMap::new()) }
    }

    pub fn add_book(&self, book: Book) {
        self.books.lock().unwrap().insert(book.id, book);
    }

    pub fn check_availability(&self, book_id: i64) -> Result<bool, CatalogError> {
        let books = self.books.lock().unwrap();
        let book = books.get(&book_id).ok_or(CatalogError::NotFound(book_id))?;
        Ok(book.available_copies > 0)
    }

    pub fn reserve_copy(&self, book_id: i64) -> Result<(), CatalogError> {
        let mut books = self.books.lock().unwrap();
        let book = books.get_mut(&book_id).ok_or(CatalogError::NotFound(book_id))?;
        if book.available_copies <= 0 {
            return Err(CatalogError::NoCopiesAvailable(book_id));
        }
        book.available_copies -= 1;
        Ok(())
    }
}

impl Default for Catalog {
    fn default() -> Self {
        Self::new()
    }
}
```

```rust
// library_system/booking/src/lib.rs
// package "booking" -- โดเมนการจองล้วน ๆ พึ่งพา "catalog" ผ่าน path dependency
// (ตาม Part 17 หัวข้อ 17.7) ไม่ใช่ network call
use catalog::{Catalog, CatalogError};
use std::collections::HashMap;
use std::sync::Mutex;

#[derive(Debug, Clone)]
pub struct Booking {
    pub id: u64,
    pub book_id: i64,
    pub user: String,
}

pub struct BookingService<'a> {
    catalog: &'a Catalog,
    bookings: Mutex<HashMap<u64, Booking>>,
    next_id: Mutex<u64>,
}

impl<'a> BookingService<'a> {
    pub fn new(catalog: &'a Catalog) -> Self {
        Self {
            catalog,
            bookings: Mutex::new(HashMap::new()),
            next_id: Mutex::new(1),
        }
    }

    pub fn create_booking(&self, book_id: i64, user: &str) -> Result<Booking, CatalogError> {
        // เรียก "catalog" ตรง ๆ ในหน่วยความจำเดียวกัน -- ยังเป็น function call ธรรมดา
        // ไม่ต้องมี retry/timeout/metadata forwarding เลย เพราะไม่มีเครือข่ายมาเกี่ยวข้อง
        self.catalog.reserve_copy(book_id)?;

        let mut next_id = self.next_id.lock().unwrap();
        let id = *next_id;
        *next_id += 1;

        let booking = Booking { id, book_id, user: user.to_string() };
        self.bookings.lock().unwrap().insert(id, booking.clone());
        Ok(booking)
    }
}
```

สังเกตว่า `BookingService::create_booking` ทำสิ่งเดียวกันเชิงหน้าที่กับ `create_booking` handler ของ `booking-service` ในหัวข้อ 81.4 ทุกประการ (เรียก catalog เพื่อ reserve ก่อน แล้วบันทึก booking) **แต่ไม่ต้องมี**: การแนบ metadata, retry with backoff, correlation ID ข้าม process, หรือ Saga compensation เลยแม้แต่นิดเดียว — เพราะ `self.catalog.reserve_copy(book_id)` เป็น function call ที่**ไม่มีทางล้มเหลวเพราะเครือข่าย**และ**อยู่ในหน่วยความจำ/transaction scope เดียวกัน**ได้ถ้าต้องการ (ห่อทั้งสองการเรียกด้วย mutex/transaction เดียวกันได้ตรง ๆ) — นี่คือสิ่งที่ "หายไป" ทันทีที่ตัดสินใจแตกเป็น service จริงตามหัวข้อ 81.4 และเป็นเหตุผลที่ต้องมั่นใจจริง ๆ ว่าคุ้มค่าตามเช็คลิสต์หัวข้อ 81.1/81.10 ก่อนตัดสินใจแตก

#### Strangler Fig Pattern: แตกออกทีละส่วน ไม่ใช่รื้อทั้งระบบพร้อมกัน

ถ้าตัดสินใจแล้วว่าต้องแตก monolith ที่โมดูลาไรซ์ไว้ (แบบโค้ดข้างบน) ออกเป็น service จริง **ไม่ควรทำทั้งระบบพร้อมกันในครั้งเดียว** — แนวทางที่ปลอดภัยกว่าและใช้กันแพร่หลายในทีมจริงเรียกว่า **Strangler Fig pattern** (ชื่อมาจากต้นไม้ปรสิตที่ค่อย ๆ เติบโตพันรอบต้นไม้เดิมจนแทนที่มันไปทีละน้อย โดยที่ต้นไม้เดิมยังมีชีวิตอยู่ตลอดกระบวนการ):

1. เลือก module เดียวที่มีเหตุผลหนักแน่นที่สุดตามเช็คลิสต์หัวข้อ 81.1 (ในระบบตัวอย่างนี้คือ `catalog`) แตกออกเป็น service ก่อน
2. ให้ monolith เดิมเรียก service ใหม่นี้ผ่าน network (gRPC) **แทนที่จะเรียก function ในหน่วยความจำแบบเดิม** — ส่วนอื่นของ monolith (`booking` และส่วนที่เหลือ) ยังทำงานเหมือนเดิมทุกประการ ไม่ต้องแก้อะไร
3. สังเกตผลกระทบจริง (latency เพิ่มขึ้นแค่ไหน, operations ยากขึ้นแค่ไหน) ก่อนตัดสินใจแตก module ถัดไป
4. ทำซ้ำทีละ module จนกว่าจะถึงจุดที่เหมาะสม (ซึ่ง**ไม่จำเป็นต้องเป็น "แตกทุกอย่างจนไม่มี monolith เหลือเลย"** — ระบบจริงจำนวนมากจบลงที่สถานะ "มี core monolith ใหญ่ + service เฉพาะทางไม่กี่ตัวที่มีเหตุผลชัดเจนจริง ๆ" ซึ่งเป็นจุดที่สมดุลดีที่สุดสำหรับหลายทีม)

ข้อดีของแนวทางนี้คือ**ลดความเสี่ยง**ลงมาก — ถ้าแตก `catalog` ออกมาแล้วพบว่ามีปัญหาที่คาดไม่ถึง (เช่น latency เพิ่มขึ้นเกินที่รับได้) ยังแก้ไข/ย้อนกลับได้โดยกระทบแค่ส่วนเดียวของระบบ ไม่ใช่ทั้งระบบพร้อมกัน — บทนี้ (หัวข้อ 81.4 เป็นต้นไป) แสดงผลลัพธ์ของขั้นตอนที่ 1-2 ข้างบนเสร็จสมบูรณ์แล้วสำหรับ `catalog-service` เพียงตัวเดียว โดยยังไม่ได้แตก `user-service` ออกมาเต็มรูปแบบ (มีแค่ `mint_token` เป็นเครื่องมือจำลองในหัวข้อ 81.4 — แบบฝึกหัดข้อ 4 ท้ายบทให้ลองทำขั้นตอนนี้ต่อจริง)

### 81.2 แตกโดเมนออกเป็น Service: หลักการ Data Ownership

#### ขอบเขตของ service ในระบบตัวอย่างของบทนี้

หลักสูตรนี้ใช้โดเมน "ห้องสมุด/ระบบจอง" มาตลอด (Part 70, 74, 80) มาถึงบทนี้ เราจะแตกมันออกเป็นสาม service ตามขอบเขตทางธุรกิจที่ชัดเจน:

```text
                        ┌─────────────────────┐
   client (เว็บ/มือถือ) │                      │
   ──── HTTP/JSON ─────▶│   booking-service    │  (Axum REST API)
                        │   (เจ้าของ: booking) │
                        └──────────┬───────────┘
                                   │ gRPC (synchronous)
                                   │ ตรวจสอบ availability + reserve
                                   ▼
                        ┌──────────────────────┐
                        │   catalog-service     │  (Tonic gRPC server)
                        │   (เจ้าของ: books)    │
                        └──────────┬───────────┘
                                   │ SQLx
                                   ▼
                          PostgreSQL: catalog DB
                          (ตาราง books เท่านั้น)

   booking-service มีฐานข้อมูล/storage ของตัวเองแยกต่างหาก (ตาราง bookings)
   ไม่แชร์กับ catalog DB เลยแม้แต่ตารางเดียว

              ┌──────────────────────┐
              │    user-service       │  (สมมติ — ไม่ implement เต็มในบทนี้)
              │  (เจ้าของ: users,      │  ต่อยอดจาก Part 74-76 (JWT/RBAC)
              │   ออก JWT ด้วย         │  ออก JWT ที่ service อื่นตรวจสอบเอง
              │   RS256 private key)   │  ด้วย public key (หัวข้อ 81.5)
              └──────────────────────┘
```

สาม service นี้แบ่งความรับผิดชอบตามหลัก **domain-driven boundary** ไม่ใช่แบ่งตาม "ชนิดของเทคโนโลยี" หรือ "ขนาดของโค้ด":

- **`catalog-service`** — เจ้าของข้อมูล **หนังสือ/สต๊อก** ทั้งหมด: ชื่อ, ผู้เขียน, จำนวนที่มี, จำนวนที่เหลือให้จอง มีหน้าที่ตอบคำถาม "หนังสือเล่มนี้มีเหลือไหม" และ "reserve/release สิทธิ์การจองให้หน่อย" เท่านั้น — **ไม่รู้จักแนวคิด "การจอง" ของผู้ใช้เลย** (ไม่มีตาราง `bookings`, ไม่รู้ว่าใครจองอะไรไปบ้าง)
- **`booking-service`** — เจ้าของข้อมูล **การจอง** ของผู้ใช้แต่ละคน: ใครจองเล่มไหน เมื่อไหร่ สถานะอะไร — **ไม่รู้จักรายละเอียดของหนังสือเลย** (ไม่มีตาราง `books`, ไม่รู้ราคา/ผู้เขียน) มันแค่ถือ `book_id` ไว้เป็นตัวชี้ แล้วถาม `catalog-service` เมื่อต้องรู้รายละเอียด
- **`user-service`** (สมมติ ต่อยอดจาก Part 74-76) — เจ้าของข้อมูล **ผู้ใช้และสิทธิ์** ทั้งหมด: username, password hash, role — เป็นตัวเดียวที่มี private key สำหรับออก JWT (ตามแนวคิด RS256 ของ Part 74) service อื่นทั้งหมดมีแค่ public key

#### หลักการที่สำคัญที่สุดของหัวข้อนี้: แต่ละ Service เป็นเจ้าของฐานข้อมูลตัวเอง — ห้ามแชร์ตารางข้าม Service

นี่คือกฎที่**สำคัญกว่าเรื่องเทคนิคอื่นใดในบทนี้**และเป็นกฎที่ทีมจริงจำนวนมากละเมิดโดยไม่รู้ตัว จนสุดท้ายได้ระบบที่เรียกตัวเองว่า "microservices" แต่มีความผูกติดกันแบบ monolith ซ่อนอยู่:

> **ห้าม service สองตัวเชื่อมต่อไปยังฐานข้อมูลเดียวกันแล้ว query ตารางของกันและกันตรง ๆ เด็ดขาด** — ทุกการเข้าถึงข้อมูลของ service หนึ่งจาก service อื่น ต้องผ่าน API ที่ service เจ้าของข้อมูลเปิดให้เท่านั้น (ในบทนี้คือ gRPC)

ทำไมกฎนี้ถึงสำคัญมาก มาดูสิ่งที่เกิดขึ้นถ้าละเมิดมัน: สมมติ `booking-service` เชื่อมต่อ `catalog_service` database ตรง ๆ แล้ว `SELECT available_copies FROM books WHERE id = $1` เอง (ข้าม gRPC ไปเลย เพราะ "เร็วกว่า ง่ายกว่า"):

- **`catalog-service` เปลี่ยน schema ตาราง `books` ไม่ได้อีกต่อไปโดยไม่ประสานกับทีม `booking-service`** — ทั้งที่ในทางทฤษฎี `catalog-service` ควรเป็นเจ้าของ schema นี้แต่เพียงผู้เดียว นี่คือความผูกติดกัน (coupling) แบบเดียวกับที่ monolith มี เพียงแต่ซ่อนอยู่ในฐานข้อมูลแทนที่จะอยู่ใน function call ตรง ๆ — **แย่กว่าเดิมด้วยซ้ำ** เพราะ compiler ไม่ช่วยตรวจให้เหมือนตอนเป็น monolith (เปลี่ยนชื่อ column ใน `catalog-service` แล้ว `cargo build` ผ่านสนิท แต่ `booking-service` จะพังตอน runtime โดยไม่มีสัญญาณเตือนล่วงหน้าเลย)
- **`catalog-service` เสียการควบคุม business logic ของตัวเอง** — ถ้า `booking-service` ทำ `UPDATE books SET available_copies = available_copies - 1` ตรง ๆ เอง logic การตรวจสอบ/validation ที่ `catalog-service` ควรทำ (เช่น ห้ามลดต่ำกว่า 0, ห้าม reserve เกิน total_copies) จะถูก bypass ไปเลย เพราะ `booking-service` เขียน SQL เองโดยไม่ผ่าน logic ของเจ้าของข้อมูลจริง
- **Scale แยกกันไม่ได้อีกต่อไป** — ทั้งสอง service แข่งกันใช้ connection pool เดียวกันของฐานข้อมูลเดียวกัน โหลดสูงของฝั่งหนึ่งกระทบอีกฝั่งได้ทันที (ข้อดีเรื่อง independent scaling ที่ควรได้จากการแตก service หายไปเลย)

#### ทำไม `user-service` ในภาพนี้ถึงไม่มีลูกศรตรงไปยัง `booking-service`/`catalog-service`

สังเกตในไดอะแกรมข้างบนว่า `user-service` ถูกวาดแยกออกมาโดยไม่มีลูกศรเชื่อมกับอีกสอง service เลย — นี่คือความตั้งใจ ไม่ใช่การละเลย: `user-service` **ไม่ถูกเรียกโดยตรงจาก `booking-service`/`catalog-service` เลยแม้แต่ครั้งเดียว** ในสถาปัตยกรรมของบทนี้ ต่างจากที่หลายคนคาดไว้ตอนแรก ("ก็ต้องเช็คกับ user-service ว่า user คนนี้มีจริงไหมสิ") — เหตุผลย้อนกลับไปที่หัวข้อ 81.5 ที่จะอธิบายเต็มรูปแบบ: `user-service` มีหน้าที่**ออก JWT เท่านั้น** (ตอน login) หลังจากนั้น**ทุก service ที่ต้องรู้ว่าใครเป็นคนเรียก ตรวจสอบได้เองจาก JWT ที่ client ส่งมาโดยตรง** ผ่าน public key ที่มีอยู่แล้วในเครื่อง (RS256 ตามที่ Part 74 สอนไว้) โดย**ไม่ต้องเรียกกลับไปถาม `user-service` อีกเลย**

นี่คือความแตกต่างที่สำคัญเชิงสถาปัตยกรรม: `client` เรียก `user-service` **แค่ครั้งเดียวตอน login** เพื่อได้ JWT มา จากนั้น `client` แนบ JWT นั้นไปกับทุก request ที่ยิงไปยัง `booking-service` เอง (ไม่ใช่ให้ `booking-service` ไปถาม `user-service` แทน) — ลูกศรที่ควรมีจริง ๆ ในไดอะแกรมคือ `client → user-service` (สำหรับ login เพียงครั้งเดียว) แยกออกจากลูกศร `client → booking-service` (สำหรับทุก request หลังจากนั้น) — `booking-service` และ `catalog-service` จึงเป็นอิสระจาก `user-service` อย่างสมบูรณ์ในเชิง runtime (ถ้า `user-service` ล่มไปกลางดึก ผู้ใช้ที่ login ไว้ก่อนแล้วยังจองหนังสือต่อได้ตามปกติ เพราะ JWT ของพวกเขายังตรวจสอบผ่าน public key ได้เหมือนเดิม — มี **แค่ผู้ใช้ใหม่ที่ยัง login ไม่ได้เท่านั้น**ที่กระทบ) นี่คือประโยชน์เชิงความทนทานของระบบ (resilience) ที่ได้มาโดยตรงจากการเลือก RS256 asymmetric signing ตั้งแต่ Part 74 — แบบฝึกหัดข้อ 4 ท้ายบทให้ลอง implement `user-service` เต็มรูปแบบและวัดผลกระทบนี้ด้วยตัวเลขจริง

สรุปเป็นหลักการเดียว: **"database เดียวที่ใช้ร่วมกันหลาย service" คือ monolith ที่ปลอมตัวมาเป็น microservices** — มันมีต้นทุนทั้งหมดของ distributed system (network, deploy แยก, ops ซับซ้อนขึ้น) โดยไม่ได้ประโยชน์เรื่อง decoupling ที่ควรได้เลย เพราะทุก service ยังผูกติดกับ schema เดียวกันอยู่ดี — ในบทนี้ `catalog-service` เชื่อมต่อ database ชื่อ `catalog_service` เท่านั้น และ `booking-service` **ไม่มี connection string ไปยังฐานข้อมูลนั้นอยู่ในโค้ดเลยแม้แต่บรรทัดเดียว** ทุกครั้งที่ต้องรู้เรื่องหนังสือ มันต้องเรียกผ่าน gRPC เท่านั้น — นี่คือสิ่งที่หัวข้อ 81.4 จะสร้างให้เห็นจริง

### 81.3 การสื่อสารระหว่าง Service: เลือก Synchronous หรือ Asynchronous

เมื่อ service สองตัวต้องคุยกัน มีสองแนวทางหลักที่ต้องเข้าใจความแตกต่างให้ชัดก่อนเลือกใช้:

#### Synchronous: gRPC (Part 80) — เมื่อผู้เรียก "ต้องรอคำตอบ"

**ใช้เมื่อ**: ผู้เรียกต้องการคำตอบ**เดี๋ยวนี้**เพื่อไปทำงานต่อ — ไม่มีคำตอบ งานต่อไปทำไม่ได้เลย ตัวอย่างที่ชัดที่สุดในระบบของบทนี้: **`booking-service` เรียก `catalog-service` เพื่อเช็คว่าหนังสือมีเหลือไหม ก่อนจะยืนยันการจอง** — นี่ต้อง**รอคำตอบ**เสมอ เพราะถ้าไม่รอ `booking-service` จะไม่รู้เลยว่าควรสร้าง booking หรือไม่ (ตอบ client กลับไปว่า "จองสำเร็จ" ทั้งที่ยังไม่รู้ว่ามีเหลือจริงไหม เป็นความผิดพลาดที่ยอมรับไม่ได้)

ลักษณะสำคัญของ synchronous call:

- ผู้เรียก**block รอ**จนกว่าจะได้คำตอบ (หรือ timeout/error)
- ผู้เรียกต้องมี **fallback plan ทันที** ถ้าเรียกไม่สำเร็จ (retry? ยกเลิก operation? แจ้ง error ให้ client?) — หัวข้อ 81.7 จะลงรายละเอียด
- เหมาะกับ**คำขอที่มีผลลัพธ์ทันที** — "ตรวจสอบ", "อ่านข้อมูล", "ทำธุรกรรมที่ต้องรู้ผลก่อนไปต่อ"

#### Asynchronous: Message Queue (Part 82 จะสอนเต็มรูปแบบ) — เมื่อผู้เรียก "ไม่ต้องรอ"

**ใช้เมื่อ**: ผู้เรียกแค่ต้องการ **"บอกให้รู้ว่าเกิดอะไรขึ้น"** โดยไม่จำเป็นต้องรอให้ผู้รับทำงานเสร็จก่อนไปต่อ ตัวอย่างในระบบของบทนี้ (ที่บทนี้จะแค่ **foreshadow ไว้ ไม่ implement เต็ม** เพราะเป็นหน้าที่ของ Part 82): เมื่อ `booking-service` สร้าง booking สำเร็จแล้ว มันอาจต้อง **publish event "booking created"** ให้ `notification-service` (สมมติ) ไปส่งอีเมลยืนยันให้ผู้ใช้ — `booking-service` **ไม่จำเป็นต้องรอ**ให้อีเมลถูกส่งจริงก่อนตอบ client ว่า "จองสำเร็จ" เลย เพราะการส่งอีเมลไม่ใช่เงื่อนไขที่ทำให้การจองสำเร็จหรือไม่สำเร็จ

ลักษณะสำคัญของ asynchronous messaging:

- ผู้เรียก (publisher) **ส่งข้อความแล้วไปต่อทันที** ไม่ต้องรอผู้รับ (consumer) ประมวลผลเสร็จ
- ผู้รับอาจประมวลผลข้อความนั้น**ช้ากว่า**ตอนที่ส่งมาก (วินาที นาที หรือชั่วโมง ขึ้นกับ queue) — ระบบต้อง "ยอมรับ" ความล่าช้านี้ได้ในเชิง business logic
- ทนต่อการที่ผู้รับ**ล่มชั่วคราว**ได้ดีกว่า synchronous มาก เพราะ message queue เก็บข้อความรอไว้ให้ ผู้รับกลับมาออนไลน์เมื่อไหร่ก็ไปดึงมาประมวลผลต่อได้ (ต่างจาก gRPC synchronous ที่ถ้าปลายทางล่ม การเรียกครั้งนั้น**fail ทันที**)
- เหมาะกับ **event ที่ "แจ้งให้รู้"** — "สร้างแล้ว", "อัปเดตแล้ว", "ลบแล้ว" ที่ผู้รับหลายตัวอาจสนใจ (notification-service, analytics-service, audit-log-service ต่างก็ subscribe event เดียวกันได้โดยไม่ต้องให้ `booking-service` รู้จักพวกเขาเลย)

#### ตารางสรุปการเลือกใช้

| | Synchronous (gRPC, Part 80) | Asynchronous (Message Queue, Part 82) |
|---|---|---|
| ผู้เรียกต้องรอคำตอบไหม | ต้องรอ | ไม่ต้องรอ |
| ใช้เมื่อ | ต้องรู้ผลลัพธ์ก่อนไปทำงานต่อ | แค่ต้องการแจ้งให้รู้ว่ามีอะไรเกิดขึ้น |
| ตัวอย่างในบทนี้ | `booking-service` เช็ค availability กับ `catalog-service` ก่อนยืนยันจอง | `booking-service` publish "booking created" ให้ `notification-service` (Part 82) |
| พฤติกรรมถ้าปลายทางล่ม | การเรียกครั้งนั้น fail ทันที (ต้องมี retry/circuit breaker — หัวข้อ 81.7) | ข้อความรอใน queue จนกว่าปลายทางกลับมา (ทนทานกว่ามาก) |
| Coupling ระหว่างผู้เรียก/ผู้รับ | รู้จักกันตรง ๆ (ผู้เรียกต้องรู้ endpoint ของผู้รับ) | หลวมกว่า (ผู้ส่งไม่ต้องรู้ว่าใครกำลัง subscribe อยู่บ้าง) |

บทนี้จะสร้างฝั่ง synchronous (หัวข้อ 81.4-81.9) ให้ทำงานได้จริงทั้งระบบ ส่วนฝั่ง asynchronous จะเป็นเพียงการวางกรอบความคิดไว้ให้ — **Part 82 จะกลับมาเติมเต็มส่วนนี้เต็มรูปแบบด้วย RabbitMQ/Kafka จริง**

#### ทำไมไม่ใช้ REST ธรรมดาระหว่าง `booking-service` กับ `catalog-service` ไปเลย

คำถามที่มักเกิดขึ้นตอนนี้: ทั้ง `booking-service` และ `catalog-service` เป็น Rust ทั้งคู่ ทำไมไม่ให้ `booking-service` ยิง REST/JSON ธรรมดาไปหา `catalog-service` (เปิด Axum เป็น server ฝั่ง `catalog-service` ด้วยเลย) แทนที่จะต้องเรียนรู้ gRPC/`.proto` เพิ่ม — คำตอบย้อนกลับไปที่หลักการของ Part 80 หัวข้อ 80.1 ตรง ๆ: **REST เหมาะกับสถานการณ์ที่คุณไม่ได้ควบคุม client** ในขณะที่การเรียกระหว่าง `booking-service` กับ `catalog-service` คือสถานการณ์ที่**คุณควบคุมทั้งสองฝั่งเอง** เต็มร้อยเปอร์เซ็นต์ — ตารางนี้สรุปผลต่างที่จับต้องได้จริงถ้าเลือกผิดทาง:

| ผลกระทบ | ถ้าเลือก REST/JSON ระหว่าง service | ถ้าเลือก gRPC/Protobuf (ที่บทนี้ใช้) |
|---|---|---|
| Contract ระหว่างสอง service | ตกลงกันด้วยเอกสาร/ความเข้าใจร่วม (`{"book_id": ..., "available": ...}`) — compiler ไม่ช่วยตรวจให้เลยว่าทั้งสองฝั่งเข้าใจ field ตรงกัน | ตกลงกันด้วย `.proto` ไฟล์เดียว — เปลี่ยน field ผิดจะ compile error ทันทีที่ฝั่งที่ยัง generate จาก `.proto` เก่าไม่ตรงกับที่ใช้จริง (ยังต้องระวังเรื่อง rolling deploy ตามกับดักข้อ 6 อยู่ดี แต่ปลอดภัยกว่า REST มาก) |
| ขนาดข้อมูลบน wire ต่อการเรียกหลักพัน/หมื่นครั้งต่อวินาที | ใหญ่กว่า (มี field name ซ้ำ ๆ ทุกครั้งตามที่ Part 80 หัวข้อ 80.1 วัดจริงไว้ ~53% ใหญ่กว่า) | เล็กกว่ามาก — สำคัญเมื่อ service คุยกันถี่มากภายในระบบเดียวกัน |
| Multiplexing เมื่อเรียกพร้อมกันหลาย RPC ไปปลายทางเดียวกัน | ขึ้นกับ HTTP/1.1 connection pooling ของ client library ที่ใช้ (มักจำกัด concurrent connection ต่อ host) | ได้ HTTP/2 multiplexing มาโดยอัตโนมัติ (พิสูจน์จริงด้วยตัวเลขไว้แล้วใน Part 80 หัวข้อ 80.1 — เร็วขึ้นประมาณ 3 เท่าเมื่อยิง 3 RPC พร้อมกัน) |
| Streaming (เช่น RPC ที่ต้องส่งข้อมูลต่อเนื่อง) | ต้องประกอบเองด้วย WebSocket/SSE/chunked transfer ซึ่ง Axum รองรับได้แต่ไม่ built-in เป็นส่วนหนึ่งของ protocol | มีในตัวโดย design (server/client/bidirectional streaming ตาม Part 80 หัวข้อ 80.6) |

**ข้อยกเว้นที่ REST ยังชนะ**: ถ้า service ปลายทางที่ต้องคุยด้วยเป็น service ภายนอกที่คุณไม่ได้เขียน/ควบคุม (third-party API) หรือทีมที่ดูแล service นั้นไม่ถนัด gRPC (เช่นทีมนั้นถนัด REST ทีมเดียวและไม่มีเหตุผลจะเปลี่ยน) การฝืนใช้ gRPC อาจเพิ่มความยุ่งยากมากกว่าประโยชน์ที่ได้ — หลักการเลือกจึงยังเป็นหลักการเดียวกับ Part 80: **ควบคุมทั้งสองฝั่งได้เต็มที่และคุยกันถี่ ใช้ gRPC, ควบคุมได้ฝั่งเดียวหรือคุยกันไม่ถี่ ใช้ REST**

### 81.4 สร้างระบบจริง: `catalog-service` และ `booking-service`

ถึงเวลาลงมือสร้างจริง — เราจะสร้างสอง Rust project แยกกันสมบูรณ์ (คนละ `Cargo.toml`, คนละ binary, คนละ process) ที่คุยกันผ่าน gRPC ตาม pattern ของ Part 80

#### โครงสร้างโปรเจกต์

```text
catalog-service/
├── Cargo.toml
├── build.rs
├── proto/
│   └── catalog.proto
└── src/
    └── main.rs

booking-service/
├── Cargo.toml
├── build.rs
├── proto/
│   └── catalog.proto        <- สำเนาของไฟล์เดียวกัน (ดูหมายเหตุด้านล่าง)
└── src/
    ├── main.rs
    └── bin/
        └── mint_token.rs     <- เครื่องมือช่วยทดสอบ จำลอง auth-service (Part 74/75)
```

**หมายเหตุเรื่องการซ้ำไฟล์ `.proto`**: ในตัวอย่างสองไบนารีเล็ก ๆ ของบทนี้ เราคัดลอกไฟล์ `catalog.proto` ไว้ทั้งสองฝั่งเพื่อความง่าย — ในระบบจริงที่มีหลาย service คุยผ่าน `.proto` เดียวกัน แนวทางที่ดีกว่าคือเก็บไฟล์ `.proto` ไว้ใน **crate กลางเดียว** ภายใน Cargo workspace (ตาม Part 17) ที่ทั้งฝั่ง server และ client `include!` จากที่เดียวกัน หรือเก็บไว้ใน repository แยกที่ทุก service ดึงมาใช้ผ่าน build step ของตัวเอง — ไม่ว่าจะเลือกแนวทางไหน หลักการที่ต้องคงไว้คือ **`.proto` ต้องมี "แหล่งความจริงเดียว" (single source of truth)** ไม่ให้สอง service มี schema เพี้ยนไปจากกันโดยไม่รู้ตัว

#### `catalog.proto`: Contract ระหว่างสอง Service

```protobuf
syntax = "proto3";

package catalog.v1;

service CatalogService {
  rpc GetBook(GetBookRequest) returns (Book);
  rpc CheckAvailability(CheckAvailabilityRequest) returns (AvailabilityResponse);
  rpc ReserveCopy(ReserveCopyRequest) returns (ReserveCopyResponse);
  rpc ReleaseCopy(ReleaseCopyRequest) returns (ReleaseCopyResponse);
}

message Book {
  int64 id = 1;
  string title = 2;
  string author = 3;
  int32 total_copies = 4;
  int32 available_copies = 5;
}

message GetBookRequest {
  int64 id = 1;
}

message CheckAvailabilityRequest {
  int64 book_id = 1;
}

message AvailabilityResponse {
  int64 book_id = 1;
  bool available = 2;
  int32 available_copies = 3;
}

message ReserveCopyRequest {
  int64 book_id = 1;
  string booking_id = 2;
}

message ReserveCopyResponse {
  bool success = 1;
  string message = 2;
}

message ReleaseCopyRequest {
  int64 book_id = 1;
  string booking_id = 2;
}

message ReleaseCopyResponse {
  bool success = 1;
}
```

สังเกตว่า RPC ทั้ง 4 ตัวสะท้อนหน้าที่ของ `catalog-service` ตรงตามหลักการหัวข้อ 81.2 ทุกประการ: **`GetBook`**/**`CheckAvailability`** เป็นการอ่านข้อมูล ส่วน **`ReserveCopy`**/**`ReleaseCopy`** คือคู่ operation ที่ทำหน้าที่เป็น "จอง" กับ "คืนสิทธิ์" — `ReleaseCopy` นี้เองที่จะเป็นกลไก compensating action ของ Saga pattern ในหัวข้อ 81.9 `ReserveCopyRequest`/`ReleaseCopyRequest` มี field `booking_id` (เป็น `string` เพราะฝั่ง `booking-service` ใช้ `Uuid`) แนบไปด้วยเสมอ เพื่อให้ `catalog-service` ผูก log/audit trail ของการ reserve แต่ละครั้งกับ booking ที่เกี่ยวข้องได้ (แม้ `catalog-service` จะไม่เก็บรายละเอียดของ booking นั้นไว้เองก็ตาม — มันแค่ผ่าน id นี้ไปเป็นข้อมูล metadata สำหรับ logging)

#### `catalog-service`: gRPC Server + SQLx

`Cargo.toml` (เวอร์ชันที่ตรวจสอบจริงจาก crates.io ณ วันที่เขียนบทนี้ ด้วยการรัน `cargo add` ในโปรเจกต์ทดลองจริง — เหมือนที่ Part 80 ทำไว้):

```toml
[package]
name = "catalog-service"
version = "0.1.0"
edition = "2021"

[dependencies]
prost = "0.14.4"
tokio = { version = "1", features = ["full"] }
tonic = "0.14.6"
tonic-prost = "0.14.6"
async-trait = "0.1"
sqlx = { version = "0.8", features = ["runtime-tokio", "postgres"] }
jsonwebtoken = { version = "9", features = ["use_pem"] }
serde = { version = "1", features = ["derive"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }

[build-dependencies]
tonic-prost-build = "0.14.6"
```

`build.rs` — เหมือน pattern ของ Part 80 หัวข้อ 80.3 ทุกประการ เพียงแต่เปิด `build_client(false)` เพราะ `catalog-service` **ไม่ต้องมี client code เลย** (มันเป็นฝ่ายรับสายเท่านั้น ไม่เคยเรียก gRPC ออกไปหาใคร):

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    tonic_prost_build::configure()
        .build_server(true)
        .build_client(false)
        .compile_protos(&["proto/catalog.proto"], &["proto"])?;
    Ok(())
}
```

`src/main.rs` เต็มรูปแบบ:

```rust
pub mod catalog {
    tonic::include_proto!("catalog.v1");
}

use catalog::catalog_service_server::{CatalogService, CatalogServiceServer};
use catalog::{
    AvailabilityResponse, Book, CheckAvailabilityRequest, GetBookRequest, ReleaseCopyRequest,
    ReleaseCopyResponse, ReserveCopyRequest, ReserveCopyResponse,
};
use jsonwebtoken::{decode, Algorithm, DecodingKey, Validation};
use serde::{Deserialize, Serialize};
use sqlx::{FromRow, PgPool};
use tonic::{transport::Server, Request, Response, Status};
use tracing::Instrument;

#[derive(Debug, Serialize, Deserialize, Clone)]
struct Claims {
    sub: String,
    role: String,
    exp: usize,
}

#[derive(Debug, FromRow)]
struct BookRow {
    id: i64,
    title: String,
    author: String,
    total_copies: i32,
    available_copies: i32,
}

impl From<BookRow> for Book {
    fn from(row: BookRow) -> Self {
        Book {
            id: row.id,
            title: row.title,
            author: row.author,
            total_copies: row.total_copies,
            available_copies: row.available_copies,
        }
    }
}

/// catalog-service เป็นเจ้าของข้อมูลหนังสือทั้งหมด -- ไม่มี service อื่นใดใน
/// ระบบที่ query ตาราง `books` ตรง ๆ เลย ทุกคนต้องผ่าน gRPC ตัวนี้เท่านั้น
/// (หลักการ "data ownership" ที่หัวข้อ 81.2 อธิบายไว้)
pub struct CatalogStore {
    pool: PgPool,
}

fn extract_request_id<T>(request: &Request<T>) -> String {
    request
        .metadata()
        .get("x-request-id")
        .and_then(|v| v.to_str().ok())
        .unwrap_or("unknown")
        .to_string()
}

#[tonic::async_trait]
impl CatalogService for CatalogStore {
    async fn get_book(&self, request: Request<GetBookRequest>) -> Result<Response<Book>, Status> {
        let request_id = extract_request_id(&request);
        let id = request.into_inner().id;
        let span = tracing::info_span!("get_book", request_id = %request_id, book_id = id);
        async move {
            tracing::info!("get_book เริ่มทำงาน");
            let row = sqlx::query_as::<_, BookRow>(
                "SELECT id, title, author, total_copies, available_copies FROM books WHERE id = $1",
            )
            .bind(id)
            .fetch_optional(&self.pool)
            .await
            .map_err(|e| Status::internal(format!("database error: {e}")))?;

            match row {
                Some(row) => Ok(Response::new(Book::from(row))),
                None => Err(Status::not_found(format!("ไม่พบหนังสือ id={id}"))),
            }
        }
        .instrument(span)
        .await
    }

    async fn check_availability(
        &self,
        request: Request<CheckAvailabilityRequest>,
    ) -> Result<Response<AvailabilityResponse>, Status> {
        let request_id = extract_request_id(&request);
        let book_id = request.into_inner().book_id;
        let span =
            tracing::info_span!("check_availability", request_id = %request_id, book_id);
        async move {
            tracing::info!("ตรวจสอบความพร้อมของหนังสือ (synchronous call จาก booking-service)");
            let row = sqlx::query_as::<_, (i32,)>(
                "SELECT available_copies FROM books WHERE id = $1",
            )
            .bind(book_id)
            .fetch_optional(&self.pool)
            .await
            .map_err(|e| Status::internal(format!("database error: {e}")))?;

            match row {
                Some((available,)) => {
                    tracing::info!(available_copies = available, "ตรวจสอบสำเร็จ");
                    Ok(Response::new(AvailabilityResponse {
                        book_id,
                        available: available > 0,
                        available_copies: available,
                    }))
                }
                None => Err(Status::not_found(format!("ไม่พบหนังสือ id={book_id}"))),
            }
        }
        .instrument(span)
        .await
    }

    async fn reserve_copy(
        &self,
        request: Request<ReserveCopyRequest>,
    ) -> Result<Response<ReserveCopyResponse>, Status> {
        let request_id = extract_request_id(&request);
        let inner = request.into_inner();
        let span = tracing::info_span!(
            "reserve_copy",
            request_id = %request_id,
            book_id = inner.book_id,
            booking_id = %inner.booking_id
        );
        async move {
            tracing::info!("พยายาม reserve หนังสือให้ booking");
            // atomic ที่ระดับแถวเดียวในฐานข้อมูลของ catalog-service เอง (local transaction
            // แบบที่ Part 70 สอน) -- นี่คือขอบเขตสุดท้ายของ "transaction เดียว" ในระบบนี้
            // เพราะ booking-service อยู่คนละฐานข้อมูล คนละ process กันโดยสิ้นเชิง
            let result = sqlx::query(
                "UPDATE books SET available_copies = available_copies - 1 \
                 WHERE id = $1 AND available_copies > 0",
            )
            .bind(inner.book_id)
            .execute(&self.pool)
            .await
            .map_err(|e| Status::internal(format!("database error: {e}")))?;

            if result.rows_affected() == 1 {
                tracing::info!("reserve สำเร็จ");
                Ok(Response::new(ReserveCopyResponse {
                    success: true,
                    message: "reserved".to_string(),
                }))
            } else {
                tracing::warn!("reserve ล้มเหลว: ไม่มี copy ว่างเหลือแล้ว");
                Ok(Response::new(ReserveCopyResponse {
                    success: false,
                    message: "no copies available".to_string(),
                }))
            }
        }
        .instrument(span)
        .await
    }

    async fn release_copy(
        &self,
        request: Request<ReleaseCopyRequest>,
    ) -> Result<Response<ReleaseCopyResponse>, Status> {
        let request_id = extract_request_id(&request);
        let inner = request.into_inner();
        let span = tracing::info_span!(
            "release_copy",
            request_id = %request_id,
            book_id = inner.book_id,
            booking_id = %inner.booking_id
        );
        async move {
            // compensating action ของ Saga (หัวข้อ 81.9) -- คืน copy กลับให้ระบบเมื่อ
            // booking-service ทำ local step ของตัวเองไม่สำเร็จหลังจาก reserve ไปแล้ว
            tracing::warn!("compensating action: คืน copy กลับ (release)");
            sqlx::query(
                "UPDATE books SET available_copies = LEAST(available_copies + 1, total_copies) \
                 WHERE id = $1",
            )
            .bind(inner.book_id)
            .execute(&self.pool)
            .await
            .map_err(|e| Status::internal(format!("database error: {e}")))?;

            Ok(Response::new(ReleaseCopyResponse { success: true }))
        }
        .instrument(span)
        .await
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    tracing_subscriber::fmt()
        .with_env_filter(tracing_subscriber::EnvFilter::from_default_env())
        .init();

    let database_url = std::env::var("DATABASE_URL")
        .unwrap_or_else(|_| "postgres://postgres:postgres@127.0.0.1:5432/catalog_service".to_string());
    let pool = PgPool::connect(&database_url).await?;

    // ตาราง `books` นี้เป็นของ catalog-service เท่านั้น -- booking-service ไม่มีสิทธิ์
    // เชื่อมต่อฐานข้อมูลนี้เลยแม้แต่ connection string ก็ไม่ควรมี
    sqlx::query(
        "CREATE TABLE IF NOT EXISTS books (
            id BIGINT PRIMARY KEY,
            title TEXT NOT NULL,
            author TEXT NOT NULL,
            total_copies INT NOT NULL,
            available_copies INT NOT NULL
        )",
    )
    .execute(&pool)
    .await?;

    sqlx::query(
        "INSERT INTO books (id, title, author, total_copies, available_copies)
         VALUES
            (1, 'Rust in Action', 'Tim McNamara', 3, 3),
            (2, 'Zero To Production In Rust', 'Luca Palmieri', 1, 1)
         ON CONFLICT (id) DO NOTHING",
    )
    .execute(&pool)
    .await?;

    let public_key_path =
        std::env::var("JWT_PUBLIC_KEY_PATH").unwrap_or_else(|_| "../keys/public.pem".to_string());
    let public_key_pem = std::fs::read(&public_key_path)?;
    let decoding_key = DecodingKey::from_rsa_pem(&public_key_pem)?;

    let store = CatalogStore { pool };

    // Auth interceptor: catalog-service ตรวจสอบลายเซ็นของ JWT ด้วย public key ของตัวเอง
    // โดยตรง -- ไม่มีการเรียกกลับไปยัง auth-service (หรือ booking-service) เพื่อยืนยันตัว
    // ผู้ใช้ซ้ำเลยแม้แต่ครั้งเดียว ตรงตามแนวคิด RS256 asymmetric signing ที่ Part 74 สอนไว้
    let interceptor = move |req: Request<()>| -> Result<Request<()>, Status> {
        let token = req
            .metadata()
            .get("authorization")
            .and_then(|v| v.to_str().ok())
            .and_then(|v| v.strip_prefix("Bearer "))
            .ok_or_else(|| Status::unauthenticated("ไม่พบ authorization metadata"))?;

        let mut validation = Validation::new(Algorithm::RS256);
        validation.set_required_spec_claims(&["exp", "sub"]);
        let claims = decode::<Claims>(token, &decoding_key, &validation)
            .map_err(|e| Status::unauthenticated(format!("token ไม่ถูกต้อง: {e}")))?
            .claims;

        tracing::debug!(
            sub = %claims.sub,
            role = %claims.role,
            "catalog-service ตรวจสอบ JWT ผ่าน public key เองสำเร็จ (ไม่เรียกกลับ auth-service)"
        );
        Ok(req)
    };

    let addr = "0.0.0.0:50061".parse()?;
    tracing::info!("catalog-service (gRPC) กำลังฟังที่ {addr}");

    Server::builder()
        .add_service(CatalogServiceServer::with_interceptor(store, interceptor))
        .serve(addr)
        .await?;

    Ok(())
}
```

**อธิบายจุดที่ต่างจาก Part 80 อย่างมีนัยสำคัญ**:

- **SQLx จริง แทน `Mutex<HashMap>`** — ตรงตามที่ Part 80 หัวข้อ 80.4 บอกไว้ว่า "สลับจาก in-memory ไปเป็น SQLx ทำได้โดยไม่กระทบ signature ของ trait ที่ generate มาจาก `.proto` เลยแม้แต่บรรทัดเดียว" นี่คือการพิสูจน์ข้อความนั้นด้วยโค้ดจริง — `reserve_copy`/`release_copy` ใช้ `UPDATE ... WHERE ... AND available_copies > 0` เป็นเทคนิค **atomic conditional update ระดับแถวเดียว** ที่ PostgreSQL การันตีความถูกต้องให้แม้มีหลาย request มาชนกันพร้อมกัน (ไม่ต้องใช้ `SELECT` แล้ว `UPDATE` แยกสองคำสั่งซึ่งจะมี race condition)
- **`extract_request_id`** อ่าน metadata `x-request-id` ที่ `booking-service` แนบมาให้ (หัวข้อ 81.8) แล้วผูกเข้ากับทุก `tracing::info_span!` ของ RPC นั้น — ทำให้ log ของ `catalog-service` มี field เดียวกันกับ log ของ `booking-service` สำหรับ request เดียวกัน
- **Interceptor ที่ตรวจ JWT ด้วย public key ของตัวเอง** — ตรงตามแนวคิด RS256 ของ Part 74 ที่บอกว่า service ปลายทางตรวจสอบได้เองโดยไม่ต้องเรียกกลับไปยังผู้ออก token (หัวข้อ 81.5 จะขยายความเรื่องนี้)

#### `booking-service`: Axum REST API + Tonic Client

`Cargo.toml` (เวอร์ชันที่ตรวจสอบจริงเช่นกัน):

```toml
[package]
name = "booking-service"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = "0.8.9"
jsonwebtoken = { version = "11.1.0", features = ["rust_crypto"] }
prost = "0.14.4"
serde = { version = "1.0.229", features = ["derive"] }
serde_json = "1.0.151"
tokio = { version = "1.53.1", features = ["full"] }
tonic = "0.14.6"
tonic-prost = "0.14.6"
tracing = "0.1.44"
tracing-subscriber = { version = "0.3.23", features = ["env-filter"] }
uuid = { version = "1.26.1", features = ["v4", "serde"] }

[build-dependencies]
tonic-prost-build = "0.14.6"
```

สังเกตว่า `booking-service` ใช้ `jsonwebtoken` เวอร์ชัน **11.1.0** (ต่างจาก `catalog-service` ที่ resolve มาเป็น 9.3.1 เพราะ dependency graph คนละชุดกัน) พร้อมเปิด feature `rust_crypto` ตามกับดักที่ Part 74 หัวข้อ 74.3 อธิบายไว้ (jsonwebtoken 11 ขึ้นไปต้องเลือก crypto backend เองไม่เช่นนั้น panic ตอน runtime) — **นี่คือตัวอย่างจริงของสิ่งที่เกิดขึ้นเสมอในระบบ microservices: แต่ละ service มี dependency tree เป็นของตัวเอง อาจ resolve เวอร์ชันของ crate เดียวกันต่างกันได้ โดยไม่กระทบกันเลย** เพราะทั้งสองฝั่งคุยกันผ่าน `.proto` contract ไม่ใช่ผ่าน Rust type ที่ต้อง compile ร่วมกัน

`build.rs` — เปิด `build_client(true)` แทน เพราะ `booking-service` เป็นฝ่ายเรียกออกไป ไม่ได้เป็น server ของ `CatalogService`:

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    tonic_prost_build::configure()
        .build_server(false)
        .build_client(true)
        .compile_protos(&["proto/catalog.proto"], &["proto"])?;
    Ok(())
}
```

`src/main.rs` เต็มรูปแบบ:

```rust
pub mod catalog {
    tonic::include_proto!("catalog.v1");
}

use axum::{
    extract::{FromRequestParts, Path, State},
    http::{header, request::Parts, StatusCode},
    response::IntoResponse,
    routing::{get, post},
    Json, Router,
};
use catalog::catalog_service_client::CatalogServiceClient;
use catalog::{CheckAvailabilityRequest, ReleaseCopyRequest, ReserveCopyRequest};
use jsonwebtoken::{decode, Algorithm, DecodingKey, Validation};
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::sync::{Arc, Mutex};
use std::time::Duration;
use tonic::transport::Channel;
use tonic::{Code, Status};
use tracing::Instrument;
use uuid::Uuid;

#[derive(Debug, Serialize, Deserialize, Clone)]
struct Claims {
    sub: String,
    role: String,
    exp: usize,
}

#[derive(Debug, Clone, Serialize)]
struct Booking {
    id: Uuid,
    book_id: i64,
    user_sub: String,
    status: String,
}

/// booking-service เก็บ booking ของตัวเองใน storage ของตัวเองล้วน ๆ (ในบทนี้ใช้
/// in-memory เพื่อโฟกัสที่การสื่อสารข้าม service เป็นหลัก -- ในระบบจริงจะเป็น Postgres
/// อีกฐานข้อมูลหนึ่งที่แยกจาก catalog-service เด็ดขาด ตาม pattern SQLx ของ Part 70)
/// ไม่มีทางใดที่ booking-service จะ query ตาราง `books` ของ catalog-service ตรง ๆ ได้เลย
#[derive(Clone)]
struct AppState {
    catalog_client: CatalogServiceClient<Channel>,
    bookings: Arc<Mutex<HashMap<Uuid, Booking>>>,
    jwt_public_key: Arc<DecodingKey>,
}

struct CurrentUser {
    claims: Claims,
    raw_token: String,
}

fn unauthorized(message: &str) -> (StatusCode, Json<serde_json::Value>) {
    (
        StatusCode::UNAUTHORIZED,
        Json(serde_json::json!({"error": {"code": "UNAUTHENTICATED", "message": message}})),
    )
}

// custom extractor ตาม pattern เดียวกับ CurrentUser ของ Part 74 หัวข้อ 74.9 ทุกประการ
// เพียงแต่ตรวจสอบด้วย public key (RS256) แทน HS256 secret -- booking-service ไม่มีสิทธิ์
// "ออก" token ใหม่เลย มีแค่สิทธิ์ "ตรวจสอบ" เท่านั้น เหมือนกับ catalog-service
impl FromRequestParts<AppState> for CurrentUser {
    type Rejection = (StatusCode, Json<serde_json::Value>);

    async fn from_request_parts(
        parts: &mut Parts,
        state: &AppState,
    ) -> Result<Self, Self::Rejection> {
        let header_value = parts
            .headers
            .get(header::AUTHORIZATION)
            .and_then(|v| v.to_str().ok())
            .ok_or_else(|| unauthorized("ไม่พบ Authorization header"))?;

        let token = header_value
            .strip_prefix("Bearer ")
            .ok_or_else(|| unauthorized("รูปแบบ Authorization header ไม่ถูกต้อง"))?;

        let mut validation = Validation::new(Algorithm::RS256);
        validation.set_required_spec_claims(&["exp", "sub"]);
        let claims = decode::<Claims>(token, &state.jwt_public_key, &validation)
            .map_err(|e| unauthorized(&format!("token ไม่ถูกต้อง: {e}")))?
            .claims;

        Ok(CurrentUser {
            claims,
            raw_token: token.to_string(),
        })
    }
}

#[derive(Debug, Deserialize)]
struct CreateBookingRequest {
    book_id: i64,
    #[serde(default)]
    simulate_local_failure: bool,
}

/// แนบ JWT (forward ต่อจาก client ตามแนวคิด "ส่งต่อ identity" ในหัวข้อ 81.5) และ
/// correlation id (หัวข้อ 81.8) ไปเป็น gRPC metadata ก่อนยิงออกไปยัง catalog-service
fn attach_metadata<T>(request: &mut tonic::Request<T>, raw_token: &str, request_id: &str) {
    request.metadata_mut().insert(
        "authorization",
        format!("Bearer {raw_token}").parse().unwrap(),
    );
    request
        .metadata_mut()
        .insert("x-request-id", request_id.parse().unwrap());
}

fn is_retryable(status: &Status) -> bool {
    matches!(status.code(), Code::Unavailable | Code::DeadlineExceeded)
}

fn status_to_http(status: Status) -> (StatusCode, Json<serde_json::Value>) {
    let code = match status.code() {
        Code::NotFound => StatusCode::NOT_FOUND,
        Code::Unauthenticated => StatusCode::UNAUTHORIZED,
        Code::Unavailable | Code::DeadlineExceeded => StatusCode::SERVICE_UNAVAILABLE,
        _ => StatusCode::INTERNAL_SERVER_ERROR,
    };
    (
        code,
        Json(serde_json::json!({"error": {"code": format!("{:?}", status.code()), "message": status.message()}})),
    )
}

/// wrapper ทั่วไปสำหรับเรียก gRPC พร้อม retry แบบ exponential backoff -- ใช้กับ RPC ที่
/// เป็น idempotent เท่านั้น (เช่น CheckAvailability ที่แค่ "อ่าน") ตามหลักการที่หัวข้อ 81.7
/// อธิบายไว้: retry ปลอดภัยกับ error ที่บอกว่า "เชื่อมต่อไม่ได้ตอนนี้" (Unavailable/
/// DeadlineExceeded) เท่านั้น ไม่ retry error อื่น (เช่น NotFound, Unauthenticated) เพราะ
/// การ retry ซ้ำ ๆ ไม่ทำให้ error เหล่านั้นหายไป
async fn call_with_retry<F, Fut, T>(
    request_id: &str,
    op_name: &str,
    max_attempts: u32,
    mut f: F,
) -> Result<T, Status>
where
    F: FnMut() -> Fut,
    Fut: std::future::Future<Output = Result<T, Status>>,
{
    let mut attempt = 0u32;
    let mut delay = Duration::from_millis(200);

    loop {
        attempt += 1;
        match f().await {
            Ok(value) => {
                if attempt > 1 {
                    tracing::info!(
                        "[{request_id}] {op_name} สำเร็จหลังจาก retry (ครั้งที่ {attempt}/{max_attempts})"
                    );
                }
                return Ok(value);
            }
            Err(status) if is_retryable(&status) && attempt < max_attempts => {
                tracing::warn!(
                    "[{request_id}] {op_name} ล้มเหลว code={:?} (ครั้งที่ {attempt}/{max_attempts}) -- retry ใน {delay:?}",
                    status.code()
                );
                tokio::time::sleep(delay).await;
                delay *= 2;
            }
            Err(status) => {
                tracing::error!("[{request_id}] {op_name} ล้มเหลวถาวร (ครั้งที่ {attempt}): {status}");
                return Err(status);
            }
        }
    }
}

async fn create_booking(
    State(state): State<AppState>,
    current_user: CurrentUser,
    Json(req): Json<CreateBookingRequest>,
) -> Result<(StatusCode, Json<Booking>), (StatusCode, Json<serde_json::Value>)> {
    // correlation id ที่เกิด ณ "ขอบ" ของระบบ (edge) -- booking-service เป็นจุดแรกที่รับ
    // request จาก client ตรง ๆ จึงเป็นจุดที่ "ให้กำเนิด" request_id เดียวที่จะไหลต่อไปยัง
    // catalog-service ผ่าน metadata ตามหัวข้อ 81.8
    let request_id = Uuid::new_v4().to_string();
    let span = tracing::info_span!(
        "handle_booking_request",
        request_id = %request_id,
        book_id = req.book_id
    );

    async move {
        tracing::info!(user = %current_user.claims.sub, "ได้รับคำขอจองหนังสือ");

        let mut client = state.catalog_client.clone();

        // ---- ก้าวที่ 1: ตรวจสอบความพร้อมแบบ synchronous ผ่าน gRPC ----
        // booking-service "ต้องรอคำตอบ" ก่อนจะไปต่อได้ -- นี่คือเหตุผลที่ใช้ gRPC แบบ
        // synchronous ไม่ใช่ async message queue (หัวข้อ 81.3)
        let availability = call_with_retry(&request_id, "CheckAvailability", 4, || {
            let mut client = client.clone();
            let request_id = request_id.clone();
            let token = current_user.raw_token.clone();
            let book_id = req.book_id;
            async move {
                let mut grpc_req = tonic::Request::new(CheckAvailabilityRequest { book_id });
                attach_metadata(&mut grpc_req, &token, &request_id);
                client.check_availability(grpc_req).await
            }
        })
        .await
        .map_err(status_to_http)?
        .into_inner();

        if !availability.available {
            tracing::warn!("หนังสือไม่มีเหลือให้จอง (available_copies=0)");
            return Err((
                StatusCode::CONFLICT,
                Json(serde_json::json!({"error": {"code": "NOT_AVAILABLE", "message": "หนังสือเล่มนี้ถูกจองครบแล้ว"}})),
            ));
        }

        // ---- ก้าวที่ 1 ของ Saga: reserve ที่ catalog-service (local transaction ของฝั่งนั้น) ----
        let booking_id = Uuid::new_v4();
        let mut reserve_req = tonic::Request::new(ReserveCopyRequest {
            book_id: req.book_id,
            booking_id: booking_id.to_string(),
        });
        attach_metadata(&mut reserve_req, &current_user.raw_token, &request_id);

        let reserve_resp = client
            .reserve_copy(reserve_req)
            .await
            .map_err(status_to_http)?
            .into_inner();

        if !reserve_resp.success {
            tracing::warn!(reason = %reserve_resp.message, "reserve ล้มเหลวที่ catalog-service");
            return Err((
                StatusCode::CONFLICT,
                Json(serde_json::json!({"error": {"code": "RESERVE_FAILED", "message": reserve_resp.message}})),
            ));
        }
        tracing::info!(%booking_id, "reserve สำเร็จที่ catalog-service");

        // ---- ก้าวที่ 2 ของ Saga: local step ของ booking-service เอง ----
        // ถ้าขั้นตอนนี้ล้มเหลว ต้อง "compensate" ด้วยการเรียก ReleaseCopy คืนให้
        // catalog-service เพราะไม่มี distributed transaction ให้ rollback ข้าม service
        // ได้อีกต่อไป (หัวข้อ 81.9) -- ในตัวอย่างนี้ควบคุมการล้มเหลวด้วย flag
        // simulate_local_failure เพื่อพิสูจน์ compensation path ได้แน่นอน
        if req.simulate_local_failure {
            tracing::error!("จำลอง local failure ของ booking-service หลัง reserve สำเร็จแล้ว -- ต้อง compensate");

            let mut release_req = tonic::Request::new(ReleaseCopyRequest {
                book_id: req.book_id,
                booking_id: booking_id.to_string(),
            });
            attach_metadata(&mut release_req, &current_user.raw_token, &request_id);

            match client.release_copy(release_req).await {
                Ok(_) => tracing::info!("compensating action สำเร็จ: คืน copy ให้ catalog-service แล้ว"),
                Err(e) => tracing::error!(
                    "compensating action ล้มเหลว!! ต้องมีระบบ reconcile/dead-letter แยกต่างหาก: {e}"
                ),
            }

            return Err((
                StatusCode::INTERNAL_SERVER_ERROR,
                Json(serde_json::json!({"error": {
                    "code": "BOOKING_FAILED",
                    "message": "จองไม่สำเร็จ (จำลอง failure) -- ระบบได้คืนสิทธิ์ให้ catalog-service แล้ว"
                }})),
            ));
        }

        let booking = Booking {
            id: booking_id,
            book_id: req.book_id,
            user_sub: current_user.claims.sub.clone(),
            status: "confirmed".to_string(),
        };
        state
            .bookings
            .lock()
            .unwrap()
            .insert(booking_id, booking.clone());

        tracing::info!(%booking_id, "บันทึก booking สำเร็จในฐานข้อมูลของ booking-service เอง");
        Ok((StatusCode::CREATED, Json(booking)))
    }
    .instrument(span)
    .await
}

async fn get_booking(
    State(state): State<AppState>,
    _current_user: CurrentUser,
    Path(id): Path<Uuid>,
) -> impl IntoResponse {
    match state.bookings.lock().unwrap().get(&id) {
        Some(booking) => (StatusCode::OK, Json(serde_json::to_value(booking).unwrap())).into_response(),
        None => (
            StatusCode::NOT_FOUND,
            Json(serde_json::json!({"error": {"code": "NOT_FOUND", "message": "ไม่พบ booking นี้"}})),
        )
            .into_response(),
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    tracing_subscriber::fmt()
        .with_env_filter(tracing_subscriber::EnvFilter::from_default_env())
        .init();

    let public_key_path =
        std::env::var("JWT_PUBLIC_KEY_PATH").unwrap_or_else(|_| "../keys/public.pem".to_string());
    let public_key_pem = std::fs::read(&public_key_path)?;
    let jwt_public_key = Arc::new(DecodingKey::from_rsa_pem(&public_key_pem)?);

    // Service discovery แบบง่ายที่สุด: environment variable ชี้ตรงไปที่ที่อยู่ของ
    // catalog-service (หัวข้อ 81.6) -- connect_lazy() ไม่เปิด TCP connection จริงจนกว่าจะมี
    // การเรียกครั้งแรก และจะพยายามเชื่อมต่อใหม่ให้เองในทุกครั้งที่ก่อนหน้าเชื่อมต่อไม่ติด
    // ซึ่งเป็นกลไกที่ทำให้ retry+backoff ในหัวข้อ 81.7 ใช้งานได้จริงเมื่อ catalog-service
    // เพิ่ง restart กลับมา
    let catalog_addr = std::env::var("CATALOG_SERVICE_ADDR")
        .unwrap_or_else(|_| "http://127.0.0.1:50061".to_string());
    let channel = Channel::from_shared(catalog_addr)?
        .timeout(Duration::from_secs(2)) // gRPC client timeout ต่อการเรียกหนึ่งครั้ง (หัวข้อ 81.7)
        .connect_lazy();
    let catalog_client = CatalogServiceClient::new(channel);

    let state = AppState {
        catalog_client,
        bookings: Arc::new(Mutex::new(HashMap::new())),
        jwt_public_key,
    };

    let app = Router::new()
        .route("/bookings", post(create_booking))
        .route("/bookings/{id}", get(get_booking))
        .with_state(state);

    let addr = std::env::var("BOOKING_SERVICE_ADDR").unwrap_or_else(|_| "0.0.0.0:8080".to_string());
    let listener = tokio::net::TcpListener::bind(&addr).await?;
    tracing::info!("booking-service (REST) กำลังฟังที่ {addr}");
    axum::serve(listener, app).await?;

    Ok(())
}
```

**เครื่องมือช่วยทดสอบ `mint_token`** (จำลอง auth-service ของ Part 74/75 — ในระบบจริง `booking-service`/`catalog-service` จะไม่มีสิทธิ์เข้าถึง private key นี้เลย มันควรอยู่แค่ที่ `user-service`/auth-service เท่านั้น เราแยกมันเป็น binary ต่างหากในบทนี้เพื่อความสะดวกในการทดสอบ ไม่ใช่โค้ดที่ควร deploy จริง):

```rust
/// เครื่องมือช่วยทดสอบ: จำลอง auth-service (Part 74/75) ที่ครอบครอง RS256 private
/// key และมีหน้าที่ "ออก" JWT เท่านั้น -- booking-service และ catalog-service ในบทนี้
/// ไม่มีสิทธิ์เข้าถึง private key ตัวนี้เลย มีแค่ public key สำหรับตรวจสอบ
use jsonwebtoken::{encode, Algorithm, EncodingKey, Header};
use serde::{Deserialize, Serialize};
use std::time::{SystemTime, UNIX_EPOCH};

#[derive(Debug, Serialize, Deserialize)]
struct Claims {
    sub: String,
    role: String,
    iat: usize,
    exp: usize,
}

fn main() {
    let args: Vec<String> = std::env::args().collect();
    let private_key_path = args
        .get(1)
        .cloned()
        .unwrap_or_else(|| "../keys/private.pem".to_string());
    let sub = args.get(2).cloned().unwrap_or_else(|| "user-1".to_string());
    let role = args.get(3).cloned().unwrap_or_else(|| "member".to_string());

    let private_pem = std::fs::read(&private_key_path).expect("อ่าน private key ไม่สำเร็จ");
    let encoding_key =
        EncodingKey::from_rsa_pem(&private_pem).expect("parse private key ไม่สำเร็จ");

    let now = SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .unwrap()
        .as_secs() as usize;
    let claims = Claims {
        sub,
        role,
        iat: now,
        exp: now + 3600,
    };

    let token = encode(&Header::new(Algorithm::RS256), &claims, &encoding_key)
        .expect("เซ็น token ไม่สำเร็จ");
    println!("{token}");
}
```

#### รันจริงทั้งสอง Service พร้อมกัน แล้วยิง `curl` ให้เกิดการเรียกข้าม Service จริง

เตรียม key pair RS256 ด้วย `openssl` (ตาม pattern เดียวกับ Part 74 หัวข้อ 74.4) เก็บไว้ในโฟลเดอร์ `keys/` ที่ทั้งสองโปรเจกต์เข้าถึงได้ร่วมกัน:

```bash
mkdir -p keys
openssl genrsa -out keys/private.pem 2048
openssl rsa -in keys/private.pem -pubout -out keys/public.pem
```

สร้างฐานข้อมูลสำหรับ `catalog-service`:

```bash
sudo -u postgres psql -c "CREATE DATABASE catalog_service;"
```

**เทอร์มินัลที่ 1** — รัน `catalog-service`:

```bash
cd catalog-service
RUST_LOG=info \
DATABASE_URL="postgres://postgres:postgres@127.0.0.1:5432/catalog_service" \
JWT_PUBLIC_KEY_PATH="../keys/public.pem" \
cargo run
```

ผลลัพธ์จริงจากการรัน:

```text
   Compiling catalog-service v0.1.0 (.../catalog-service)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 49.61s
     Running `target/debug/catalog-service`
2026-09-27T02:00:21.223378Z  INFO catalog_service: catalog-service (gRPC) กำลังฟังที่ 0.0.0.0:50061
```

**เทอร์มินัลที่ 2** — รัน `booking-service` (ในเวลาเดียวกัน คนละ process กันจริง):

```bash
cd booking-service
RUST_LOG=info \
JWT_PUBLIC_KEY_PATH="../keys/public.pem" \
CATALOG_SERVICE_ADDR="http://127.0.0.1:50061" \
BOOKING_SERVICE_ADDR="0.0.0.0:8080" \
cargo run --bin booking-service
```

ผลลัพธ์จริง:

```text
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 35.52s
     Running `target/debug/booking-service`
2026-09-27T02:00:33.626574Z  INFO booking_service: booking-service (REST) กำลังฟังที่ 0.0.0.0:8080
```

ทั้งสอง process รันอยู่**พร้อมกันจริง** คนละพอร์ต (`booking-service` ที่ 8080 เป็น REST, `catalog-service` ที่ 50061 เป็น gRPC) — ตอนนี้ mint token ด้วยเครื่องมือที่เขียนไว้:

```bash
cd booking-service
TOKEN=$(cargo run --bin mint_token -- ../keys/private.pem user-42 member 2>/dev/null)
echo "$TOKEN"
```

ยิง `curl` จริงไปที่ `booking-service` (REST) ซึ่งจะ**เรียก `catalog-service` ผ่าน gRPC ภายในของมันเองโดยอัตโนมัติ**:

```bash
curl -s -X POST http://127.0.0.1:8080/bookings \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"book_id": 2}' -w "\nHTTP_STATUS:%{http_code}\n"
```

**ผลลัพธ์จริงจากการรัน**:

```text
{"id":"c3725c28-5c9e-4f16-9b1f-607004219016","book_id":2,"user_sub":"user-42","status":"confirmed"}
HTTP_STATUS:201
```

`booking-service` ตอบ `201 Created` กลับมาพร้อม `booking_id` ที่สร้างขึ้นจริง — นี่คือหลักฐานว่าเส้นทางเต็ม **client → `booking-service` (REST) → `catalog-service` (gRPC) → PostgreSQL → กลับมาที่ `booking-service` → client** ทำงานสำเร็จจริงทั้งสาย ลองยิงจองเล่มเดิมอีกครั้ง (เล่ม id=2 มี `total_copies=1` เท่านั้น จึงควรเต็มแล้ว):

```bash
curl -s -X POST http://127.0.0.1:8080/bookings \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"book_id": 2}' -w "\nHTTP_STATUS:%{http_code}\n"
```

```text
{"error":{"code":"NOT_AVAILABLE","message":"หนังสือเล่มนี้ถูกจองครบแล้ว"}}
HTTP_STATUS:409
```

`booking-service` ถาม `catalog-service` แล้วรู้ว่าเล่มนี้ไม่มีเหลือจริง ๆ (ผ่าน `CheckAvailability` gRPC call จริง) แล้วตอบ `409 Conflict` กลับไปโดย**ไม่ต้อง reserve เลย** — พิสูจน์ว่า synchronous call ในหัวข้อ 81.3 ทำงานตามที่ออกแบบไว้

ลองไม่แนบ token เลยดูว่า auth ทำงานถูกต้อง:

```bash
curl -s -X POST http://127.0.0.1:8080/bookings \
  -H "Content-Type: application/json" -d '{"book_id": 1}' -w "\nHTTP_STATUS:%{http_code}\n"
```

```text
{"error":{"code":"UNAUTHENTICATED","message":"ไม่พบ Authorization header"}}
HTTP_STATUS:401
```

และดึง booking ที่สร้างไว้กลับมาดูด้วย `GET`:

```bash
curl -s http://127.0.0.1:8080/bookings/c3725c28-5c9e-4f16-9b1f-607004219016 \
  -H "Authorization: Bearer $TOKEN"
```

```text
{"book_id":2,"id":"c3725c28-5c9e-4f16-9b1f-607004219016","status":"confirmed","user_sub":"user-42"}
```

ทั้งหมดนี้คือระบบสอง service ที่รันจริง คุยกันข้าม process จริงผ่าน gRPC จริง — ไม่ใช่ mock หรือ diagram บนกระดาษ

#### สรุปคำสั่งทั้งหมดไว้ในที่เดียว (Quick Reference)

เพื่อให้กลับมาลองรันซ้ำได้ง่ายโดยไม่ต้องไล่หาคำสั่งกระจายอยู่ทั้งหัวข้อ นี่คือลำดับคำสั่งทั้งหมดตั้งแต่ต้นจนจบ รวบรวมไว้ในที่เดียว (ต้องเปิดหลายเทอร์มินัลพร้อมกันตามที่ระบุ):

```bash
# --- เตรียมการหนึ่งครั้ง ---
mkdir -p keys
openssl genrsa -out keys/private.pem 2048
openssl rsa -in keys/private.pem -pubout -out keys/public.pem
sudo -u postgres psql -c "CREATE DATABASE catalog_service;"

# --- เทอร์มินัลที่ 1: catalog-service ---
cd catalog-service
RUST_LOG=info \
  DATABASE_URL="postgres://postgres:postgres@127.0.0.1:5432/catalog_service" \
  JWT_PUBLIC_KEY_PATH="../keys/public.pem" \
  cargo run

# --- เทอร์มินัลที่ 2: booking-service ---
cd booking-service
RUST_LOG=info \
  JWT_PUBLIC_KEY_PATH="../keys/public.pem" \
  CATALOG_SERVICE_ADDR="http://127.0.0.1:50061" \
  BOOKING_SERVICE_ADDR="0.0.0.0:8080" \
  cargo run --bin booking-service

# --- เทอร์มินัลที่ 3: client (ทดสอบ) ---
cd booking-service
TOKEN=$(cargo run --bin mint_token -- ../keys/private.pem user-42 member 2>/dev/null)

# จองสำเร็จ (เล่ม id=2 มี total_copies=1)
curl -s -X POST http://127.0.0.1:8080/bookings \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"book_id": 2}' -w "\nHTTP_STATUS:%{http_code}\n"

# จองซ้ำ -- ควรได้ 409 เพราะเล่มเต็มแล้ว
curl -s -X POST http://127.0.0.1:8080/bookings \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"book_id": 2}' -w "\nHTTP_STATUS:%{http_code}\n"

# ไม่แนบ token -- ควรได้ 401
curl -s -X POST http://127.0.0.1:8080/bookings \
  -H "Content-Type: application/json" -d '{"book_id": 1}' -w "\nHTTP_STATUS:%{http_code}\n"

# จำลอง local failure -- ควรได้ 500 พร้อม compensating action (หัวข้อ 81.9)
curl -s -X POST http://127.0.0.1:8080/bookings \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"book_id": 1, "simulate_local_failure": true}' -w "\nHTTP_STATUS:%{http_code}\n"

# --- ทดสอบ retry+backoff จริง (หัวข้อ 81.7) ---
# 1) หา pid ของ catalog-service แล้ว kill -9 ทันที
# 2) ยิง curl ตัวใดตัวหนึ่งข้างบนทันทีหลัง kill (ในเทอร์มินัลที่ 3)
# 3) restart catalog-service ทันที (ในเทอร์มินัลที่ 1) ก่อนที่ retry ครบ 4 ครั้ง
#    -- สังเกต log ทั้งสองฝั่งว่า request_id เดียวกันปรากฏในทั้งสอง log stream
#    และ curl ได้รับคำตอบสำเร็จหลังจาก retry แม้ catalog-service เพิ่งตายไปกลางทาง

# --- ล้างข้อมูลหลังทดสอบเสร็จ ---
sudo -u postgres psql -c "DROP DATABASE catalog_service;"
```

สังเกตว่าสคริปต์นี้คือลำดับคำสั่งเดียวกันเป๊ะกับที่ใช้ตรวจสอบเนื้อหาของบทนี้จริงทุกขั้นตอน (ตามที่ระบุไว้ในหมายเหตุต้นบท) — คัดลอกไปรันตามได้ทันทีโดยไม่ต้องแก้อะไรเพิ่ม (นอกจากพาธของ `keys/` และ `catalog-service`/`booking-service` ให้ตรงกับโครงสร้างโฟลเดอร์จริงของคุณ)

### 81.5 Cross-Service Authentication: ส่งต่อ Identity ด้วย RS256 JWT

หัวข้อนี้ขยายความจากโค้ดในหัวข้อ 81.4 ว่า**ทำไม**การเลือก RS256 (ต่อจาก Part 74) ถึงเหมาะกับสถานการณ์นี้พอดี

#### ปัญหา: ผู้ใช้ login ที่ไหน ใครต้องตรวจสอบตัวตนซ้ำบ้าง

ในระบบ monolith เดียว (Part 62-78) การตรวจสอบ JWT ทำครั้งเดียวตอนรับ request เข้ามา (ผ่าน extractor แบบ `CurrentUser` ของ Part 74) แล้ว handler ทุกตัวก็ใช้ผลลัพธ์นั้นต่อได้เลยในหน่วยความจำเดียวกัน — แต่ในระบบที่แตกเป็น service แล้ว เมื่อ `booking-service` เรียกต่อไปยัง `catalog-service` เกิดคำถามว่า: **`catalog-service` ควรรู้ไหมว่าใครเป็นคนสั่งการดำเนินการนี้? ต้องตรวจสอบตัวตนซ้ำไหม?**

มีสองตัวเลือกที่เป็นไปได้ (และตัวเลือกที่แย่กว่าที่ควรหลีกเลี่ยง):

1. **(แย่)** `catalog-service` เชื่อ `booking-service` แบบไม่มีเงื่อนไข (ไม่ตรวจสอบอะไรเลย) — ถ้า `booking-service` ถูกแฮก หรือมี bug ที่เรียก `catalog-service` แบบผิด ๆ `catalog-service` จะไม่มีทางรู้เลยว่าการเรียกนั้นควรถูกปฏิเสธหรือไม่
2. **(แย่กว่า)** `catalog-service` เรียกกลับไปถาม auth-service ทุกครั้งว่า token นี้ valid ไหม — ทำงานได้ แต่เพิ่ม network round-trip อีกชั้น (และเพิ่มจุดที่ล้มเหลวได้อีกจุดหนึ่ง — ถ้า auth-service ล่ม ทุก service ในระบบพังหมด แม้จะไม่เกี่ยวกับ auth เลยก็ตาม)
3. **(ที่บทนี้ใช้)** `catalog-service` ตรวจสอบลายเซ็นของ JWT **ด้วยตัวเอง** ผ่าน public key ที่มีอยู่แล้วในเครื่อง — ไม่ต้องเรียกกลับไปหาใครเลย

#### ทำไม RS256 ถึงทำให้ตัวเลือกที่ 3 เป็นไปได้จริง

นี่คือจุดที่แนวคิด **asymmetric signing** จาก Part 74 หัวข้อ 74.4 กลับมามีความหมายเต็มรูปแบบ: **private key** อยู่ที่ auth-service (`user-service` ในภาพหัวข้อ 81.2) เท่านั้น ที่เดียว — เป็นตัวเดียวที่ "ออก" token ใหม่ได้ ส่วน **public key** ถูกแจกจ่ายไปให้ทุก service ที่ต้อง "ตรวจสอบ" token (ในบทนี้คือ `booking-service` และ `catalog-service` ทั้งคู่มี `keys/public.pem` ก๊อปปี้เดียวกัน) — เพราะคุณสมบัติทาง cryptography ของ RSA (พิสูจน์ไว้แล้วด้วยโค้ดจริงใน Part 74 หัวข้อ 74.4) **การมี public key ไม่ได้ให้ความสามารถในการปลอมสร้าง token ใหม่เลย** ดังนั้นการแจก public key ไปให้หลาย service จึงปลอดภัย — แต่ละ service ตรวจสอบได้เองโดยไม่ต้องเรียกกลับไปยังผู้ออก token แม้แต่ครั้งเดียว

ระบบในหัวข้อ 81.4 ทำสิ่งนี้จริง: ทั้ง `booking-service` (ใน `CurrentUser::from_request_parts`) และ `catalog-service` (ใน auth interceptor) ต่าง**อ่านไฟล์ `keys/public.pem` เดียวกันแยกกันคนละ process** แล้วเรียก `jsonwebtoken::decode` ตรวจสอบลายเซ็นด้วยตัวเองทั้งคู่ — ไม่มีจุดใดในระบบที่เรียก "auth-service" กลับไปเลยแม้แต่ครั้งเดียว (เพราะในตัวอย่างนี้เราไม่ได้ implement `user-service` เต็มรูปแบบ แค่จำลองการออก token ด้วย `mint_token` bin — แต่หลักการเดียวกันนี้ใช้ได้กับ auth-service จริงเต็มรูปแบบทุกประการ)

#### การ Forward Claims ผ่าน gRPC Metadata

จุดที่ต้องสังเกตให้ชัดในโค้ดหัวข้อ 81.4 คือฟังก์ชัน `attach_metadata`:

```rust
fn attach_metadata<T>(request: &mut tonic::Request<T>, raw_token: &str, request_id: &str) {
    request.metadata_mut().insert(
        "authorization",
        format!("Bearer {raw_token}").parse().unwrap(),
    );
    request
        .metadata_mut()
        .insert("x-request-id", request_id.parse().unwrap());
}
```

`booking-service` **ไม่ได้สร้าง token ใหม่เพื่อเรียก `catalog-service`** — มันแค่**ส่งต่อ (forward) token เดิมที่ client ส่งมาให้มันนั่นแหละ** ไปเป็น gRPC metadata (ตาม pattern ของ Part 80 หัวข้อ 80.8) นี่คือแนวคิดสำคัญของการส่งต่อ identity ข้าม service: **`catalog-service` เห็น claims เดียวกันเป๊ะกับที่ `booking-service` เห็น** (`sub: "user-42"`, `role: "member"`) — มันจึงรู้**ว่าใครเป็นคนสั่งการดำเนินการนี้จริง ๆ** ไม่ใช่แค่รู้ว่า "`booking-service` เรียกมา" เฉย ๆ ซึ่งเปิดทางให้ `catalog-service` ทำ **authorization ที่ granular ระดับผู้ใช้ปลายทางได้** (เช่น ในระบบจริงอาจเช็คว่า `role` เป็น `admin` ก่อนอนุญาตให้ดู `available_copies` ของหนังสือบางประเภท) โดยไม่ต้องพึ่งพา `booking-service` ให้ตรวจสอบให้ครบถ้วนทุกกรณีแทนตัวเอง — เป็นการกระจายความรับผิดชอบด้าน authorization ให้ตรงกับ service ที่เป็นเจ้าของ resource นั้นจริง ๆ ตามหลักการหัวข้อ 81.2

### 81.6 Service Discovery: ภาพรวมระดับที่ต้องรู้

ในหัวข้อ 81.4 `booking-service` หาที่อยู่ของ `catalog-service` ด้วยวิธีที่**ง่ายที่สุดเท่าที่จะเป็นไปได้**:

```rust
let catalog_addr = std::env::var("CATALOG_SERVICE_ADDR")
    .unwrap_or_else(|_| "http://127.0.0.1:50061".to_string());
```

**environment variable ที่ config ไว้ตรง ๆ** — นี่คือการตัดสินใจที่ตั้งใจ ไม่ใช่ทางลัดที่มองข้าม: บทนี้มีแค่สอง service รันบนเครื่องเดียวกัน วิธีนี้ตรงไปตรงมาและเข้าใจง่ายที่สุด แต่ในระบบที่มี service จำนวนมากขึ้น รันบนหลายเครื่อง/หลาย container ที่ IP เปลี่ยนไปมา วิธีนี้จะเริ่มมีข้อจำกัดชัดเจน — ต้องรู้จักภาพรวมของตัวเลือกอื่น ๆ ที่มีในโลกจริงด้วย แม้บทนี้จะไม่ implement ทุกตัวเต็มรูปแบบก็ตาม:

| แนวทาง | หลักการ | เหมาะกับ | ข้อจำกัด |
|---|---|---|---|
| **Hardcoded config/env var** (ที่บทนี้ใช้) | ที่อยู่ของแต่ละ service ถูกกำหนดตรง ๆ ผ่าน environment variable หรือไฟล์ config | ระบบเล็ก จำนวน service น้อย ที่อยู่ไม่เปลี่ยนบ่อย (หรือ dev environment) | ต้องแก้ config ทุกครั้งที่ service ย้ายที่อยู่ ไม่ scale เมื่อจำนวน service เพิ่มมาก |
| **DNS-based discovery** | แต่ละ service มีชื่อ DNS ของตัวเอง (เช่น `catalog-service.internal`) ที่ resolve ไปยัง IP จริงโดยระบบ DNS ภายใน — โค้ดเรียกผ่านชื่อ ไม่ใช่ IP ตรง ๆ | ระบบขนาดกลาง ที่ IP เปลี่ยนได้แต่โครงสร้าง network ค่อนข้างนิ่ง | ต้องมี DNS server ภายในที่ update record ให้ทันเวลาเมื่อ service restart/ย้ายเครื่อง |
| **Service registry** (Consul, etcd) | แต่ละ service "ลงทะเบียน" ที่อยู่ของตัวเองไว้กับ registry กลางตอนเริ่มทำงาน (และ "ถอนทะเบียน" เมื่อหยุด) ผู้เรียกถาม registry ว่า service ที่ต้องการอยู่ที่ไหนตอนนี้ | ระบบขนาดใหญ่ที่ instance ของแต่ละ service ขึ้น/ลงบ่อย ต้องการ health check ในตัว | ต้องดูแล registry เองเป็น infrastructure เพิ่มเติม (จุดเดียวที่ถ้าล่ม กระทบทั้งระบบ ต้องทำให้ registry เองมี high availability) |
| **Container orchestrator** (Kubernetes) | orchestrator จัดการ service discovery ให้ "ฟรี" ผ่าน internal DNS + load balancing ในตัว — เรียกชื่อ Service ของ Kubernetes ได้ตรง ๆ โดยไม่ต้องรู้ IP ของ Pod แต่ละตัวเลย | ระบบที่ deploy บน Kubernetes อยู่แล้ว (หลักสูตรนี้จะแตะ Kubernetes ในโมดูล Docker/CI-CD ที่ Part 96 ขึ้นไป) | ต้องมี Kubernetes cluster ให้ดูแล ซึ่งเป็นภาระ operations ระดับหนึ่งในตัวเอง |

**หลักการเลือกที่ควรจำ**: ยิ่งระบบมีจำนวน service มากขึ้นและ instance เปลี่ยนที่อยู่บ่อยขึ้น ยิ่งต้องการ discovery mechanism ที่เป็นอัตโนมัติมากขึ้น — แต่**ไม่มีเหตุผลที่ต้องรีบไปทำ service registry เต็มรูปแบบตั้งแต่วันแรก** ถ้าระบบยังมีแค่สอง-สาม service env var ตรง ๆ (แบบที่บทนี้ทำ) ก็เพียงพอและเข้าใจง่ายกว่ามาก — ค่อยขยับไปใช้ DNS/registry/orchestrator เมื่อจำนวน service และความถี่ในการเปลี่ยนที่อยู่มันคุ้มกับความซับซ้อนที่เพิ่มขึ้นจริง ๆ

#### DNS-based Discovery หน้าตาเป็นอย่างไรในโค้ด Rust

เพื่อให้เห็นภาพว่า DNS-based discovery ต่างจาก hardcoded env var ของบทนี้อย่างไรในระดับโค้ด (ไม่ implement เต็มในระบบของบทนี้ เพราะยังไม่มีเหตุผลตามเช็คลิสต์หัวข้อ 81.1/81.10 ที่จะต้องใช้) — ความต่างที่สำคัญที่สุดคือ**ที่อยู่ที่โค้ดถืออยู่เป็นชื่อ ไม่ใช่ IP** และปล่อยให้ DNS resolver (ของระบบปฏิบัติการหรือของ container orchestrator) แปลงชื่อเป็น IP ให้ในเวลาที่เรียกใช้จริง:

```rust
use std::net::ToSocketAddrs;

/// แนวคิด DNS-based discovery: resolve ชื่อ service เป็น IP ตอนเรียกใช้จริง
/// ไม่ใช่ IP ตายตัวที่ config ไว้ล่วงหน้าแบบ CATALOG_SERVICE_ADDR ของบทนี้
fn resolve_catalog_service() -> std::io::Result<Vec<std::net::SocketAddr>> {
    // "catalog-service.internal:50061" คือชื่อที่ DNS ภายในองค์กร (หรือ Kubernetes
    // internal DNS) ต้อง resolve ให้ -- ไม่ใช่ IP ที่ hardcode ไว้เลย
    let addrs: Vec<_> = "catalog-service.internal:50061".to_socket_addrs()?.collect();
    Ok(addrs)
}
```

สังเกตว่าโค้ดฝั่งเรียกใช้ (`booking-service`) **ไม่ต้องรู้เลยว่า `catalog-service` มีกี่ instance หรืออยู่ที่ IP ไหนบ้าง** — มันรู้แค่ชื่อ `catalog-service.internal` เท่านั้น การ resolve จริงถูกผลักภาระไปให้ DNS infrastructure ทำแทน ซึ่งเป็นข้อดีเวลา `catalog-service` restart แล้วได้ IP ใหม่ (เช่น container ใหม่ในสภาพแวดล้อมที่ IP ไม่คงที่) โค้ดฝั่งเรียกใช้ไม่ต้องแก้อะไรเลยเพราะมันอ้างถึงชื่อเสมอ ไม่ใช่ IP — Tonic รองรับการต่อ `Channel` ด้วยชื่อ host แบบนี้ได้ตรง ๆ อยู่แล้ว (ผ่าน `Channel::from_shared("http://catalog-service.internal:50061")`) โดยไม่ต้อง resolve ด้วยมือแบบโค้ดตัวอย่างข้างบน — โค้ดข้างบนแค่แสดงให้เห็นกลไกที่ซ่อนอยู่เบื้องหลังเท่านั้น

#### Health Check Endpoint: รากฐานที่ Service Discovery ทุกแบบต้องพึ่งพา

ไม่ว่าจะเลือก discovery mechanism แบบไหน (env var, DNS, registry, orchestrator) มีองค์ประกอบหนึ่งที่ทุกแบบต้องพึ่งพาเหมือนกัน: **วิธีตรวจสอบว่า service instance นั้น "พร้อมรับ request จริง" หรือยัง** — ไม่ใช่แค่ process ยังไม่ตาย แต่ต้องพร้อมทำงานจริง (เช่น connection pool ไปฐานข้อมูลเชื่อมต่อสำเร็จแล้ว) เพิ่ม endpoint นี้เข้าไปใน `booking-service` และ `catalog-service` ของหัวข้อ 81.4 ได้ตรง ๆ:

```rust
// เพิ่มเข้าไปใน Router ของ booking-service (หัวข้อ 81.4)
async fn healthz() -> &'static str {
    "ok"
}

// ...
let app = Router::new()
    .route("/healthz", get(healthz))
    .route("/bookings", post(create_booking))
    .route("/bookings/{id}", get(get_booking))
    .with_state(state);
```

ฝั่ง `catalog-service` (gRPC) ทำแนวคิดเดียวกันได้ด้วย `tonic-health` (crate แยกจากทีม Tonic ที่ implement gRPC Health Checking Protocol มาตรฐานให้) — บทนี้ไม่ implement เต็มเพื่อไม่ให้เนื้อหายาวเกินสโคป แต่หลักการเดียวกันคือมี RPC พิเศษ (`grpc.health.v1.Health/Check`) ที่ตอบกลับทันทีว่า service พร้อมทำงานไหม

**ทำไม health check endpoint นี้ถึงเชื่อมโยงกับหัวข้อนี้โดยตรง**: container orchestrator อย่าง Kubernetes ใช้ endpoint แบบนี้เป็น **readiness probe** — มันจะเรียก `/healthz` (หรือ gRPC health check) เป็นระยะ แล้ว**ไม่ส่ง traffic ไปยัง instance ที่ยังตอบไม่สำเร็จ** ซึ่งคือ**หนึ่งในสิ่งที่ orchestrator "ให้มาฟรี"** ตามที่ตารางข้างบนกล่าวถึง — มันทำหน้าที่ discovery + health-aware routing ให้พร้อมกันในตัว โดยที่ทีมไม่ต้องเขียน registry เองเลย นี่คือเหตุผลสำคัญที่ทำให้ Kubernetes เป็นตัวเลือกที่คุ้มค่ามากในระบบที่มี service จำนวนมากพอ (แม้จะมีต้นทุนด้าน operations ในการดูแล cluster เองก็ตาม)

### 81.7 Resilience: Timeout, Retry with Backoff, และ Circuit Breaker

การเรียกข้าม service ผ่านเครือข่าย**ล้มเหลวได้ด้วยเหตุผลที่ไม่เกี่ยวกับ logic เลย** — packet loss, service ปลายทางกำลัง restart (เช่นตอน deploy เวอร์ชันใหม่), เครือข่ายช้าผิดปกติชั่วคราว — ระบบที่ดีต้องรับมือกับสถานการณ์เหล่านี้อย่างมีสติ ไม่ใช่แค่ปล่อยให้ error โผล่ไปหา client ทันทีโดยไม่ลองอะไรก่อน

#### Timeout: อย่ารอไม่มีที่สิ้นสุด

**Part 65** สอน `TimeoutLayer` ของ `tower-http` ไว้สำหรับฝั่ง REST ที่จำกัดเวลาที่ request ค้างได้นานสุด — ฝั่ง gRPC client ก็มีแนวคิดเดียวกัน ตั้งค่าได้ตรงที่ `Channel`:

```rust
let channel = Channel::from_shared(catalog_addr)?
    .timeout(Duration::from_secs(2)) // ทุกการเรียกผ่าน channel นี้ timeout ที่ 2 วินาที
    .connect_lazy();
```

**เหตุผลที่ timeout สำคัญมาก**: ถ้าไม่ตั้ง timeout เลย และ `catalog-service` ค้าง (เช่น connection pool ของ SQLx หมด กำลังรอ connection คืนมา) `booking-service` จะ**block รอไม่มีกำหนด** ซึ่งหมายความว่า thread/task ของ `booking-service` ที่กำลังจัดการ request นั้นก็ถูกกักไว้ไม่มีกำหนดเช่นกัน — ถ้ามีหลาย request ลักษณะนี้พร้อมกัน `booking-service` เองก็จะเริ่มมีปัญหา resource exhaustion ตามไปด้วย (แม้ตัวมันเองไม่มี bug อะไรเลยก็ตาม) **timeout คือการยอมรับว่า "รอไม่คุ้ม" แล้วปล่อยให้ error handling เข้ามาทำงานต่อ** ดีกว่าค้างไม่มีกำหนด

#### Retry with Exponential Backoff: พิสูจน์ด้วยการ Kill/Restart Service จริง

ในโค้ดหัวข้อ 81.4 มีฟังก์ชัน `call_with_retry` ที่ห่อการเรียก `CheckAvailability` (ซึ่งเป็น**การอ่านข้อมูลอย่างเดียว จึงเป็น idempotent** — เรียกซ้ำกี่ครั้งก็ไม่เปลี่ยนผลลัพธ์ของระบบ ปลอดภัยที่จะ retry) ด้วย retry แบบ **exponential backoff**: รอ 200ms ก่อน retry ครั้งที่ 1, รอ 400ms ก่อนครั้งที่ 2, รอ 800ms ก่อนครั้งที่ 3 (เพิ่มเป็นสองเท่าทุกครั้ง) — เหตุผลของการเพิ่มเวลารอแบบทวีคูณ (ไม่ใช่รอเวลาเท่ากันทุกครั้ง) คือ**ถ้า service ปลายทางกำลังโอเวอร์โหลดอยู่ การยิง retry ถี่ ๆ ทันทีจะซ้ำเติมปัญหาให้แย่ลง** การเว้นช่วงเพิ่มขึ้นทุกครั้งให้เวลา service ปลายทางได้ "หายใจ" และมีโอกาสฟื้นตัวมากขึ้น

**มาพิสูจน์กันด้วยสถานการณ์จริง**: จำลอง deploy `catalog-service` เวอร์ชันใหม่ (kill process เดิมทิ้งแล้ว restart) ขณะที่ `booking-service` กำลังมี request เข้ามาพอดี:

```bash
# เทอร์มินัลที่ 3: kill catalog-service กลางอากาศ
kill -9 <catalog-service-pid>

# เทอร์มินัลที่ 4: ยิง request ทันทีหลัง kill (catalog-service ยังไม่ทันขึ้นใหม่)
curl -s -X POST http://127.0.0.1:8080/bookings \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"book_id": 2}' -w "\nHTTP_STATUS:%{http_code}\n"

# เทอร์มินัลที่ 1: restart catalog-service ทันที (จำลองว่า deploy เวอร์ชันใหม่เสร็จแล้ว)
cd catalog-service && RUST_LOG=info DATABASE_URL="..." JWT_PUBLIC_KEY_PATH="../keys/public.pem" cargo run
```

**ผลลัพธ์จริงจาก log ของ `booking-service`** (คัดลอกมาตรง ๆ จากการรันจริง — สังเกต `request_id` เดียวกัน `972e6872-34c3-45ff-a1bb-30613c501dae` ตลอดทั้ง sequence):

```text
2026-09-27T02:02:10.600145Z  INFO handle_booking_request{request_id=972e6872-... book_id=2}: booking_service: ได้รับคำขอจองหนังสือ user=user-42
2026-09-27T02:02:10.601017Z  WARN handle_booking_request{request_id=972e6872-... book_id=2}: booking_service: [972e6872-...] CheckAvailability ล้มเหลว code=Unavailable (ครั้งที่ 1/4) -- retry ใน 200ms
2026-09-27T02:02:10.803502Z  WARN handle_booking_request{request_id=972e6872-... book_id=2}: booking_service: [972e6872-...] CheckAvailability ล้มเหลว code=Unavailable (ครั้งที่ 2/4) -- retry ใน 400ms
2026-09-27T02:02:11.209785Z  INFO handle_booking_request{request_id=972e6872-... book_id=2}: booking_service: [972e6872-...] CheckAvailability สำเร็จหลังจาก retry (ครั้งที่ 3/4)
2026-09-27T02:02:11.209869Z  WARN handle_booking_request{request_id=972e6872-... book_id=2}: booking_service: หนังสือไม่มีเหลือให้จอง (available_copies=0)
```

และ log ของ `catalog-service` ที่เพิ่ง restart ขึ้นมา (เวลา `02:02:10.844204Z` คือตอนเริ่มฟังใหม่ — สังเกตว่ามันรับ request แรกที่สำเร็จได้ที่ `02:02:11.207071Z` ซึ่งตรงกับ log ฝั่ง `booking-service` ที่บอกว่า retry ครั้งที่ 3 สำเร็จ):

```text
2026-09-27T02:02:10.844204Z  INFO catalog_service: catalog-service (gRPC) กำลังฟังที่ 0.0.0.0:50061
2026-09-27T02:02:11.207071Z  INFO check_availability{request_id=972e6872-... book_id=2}: catalog_service: ตรวจสอบความพร้อมของหนังสือ (synchronous call จาก booking-service)
2026-09-27T02:02:11.208838Z  INFO check_availability{request_id=972e6872-... book_id=2}: catalog_service: ตรวจสอบสำเร็จ available_copies=0
```

นี่คือหลักฐานจริงว่า: **ครั้งที่ 1 ล้มเหลว** (`catalog-service` ยังตายอยู่ — error code `Unavailable` ตรงกับที่ `is_retryable` ตรวจจับ) **รอ 200ms → ครั้งที่ 2 ล้มเหลวอีก** (`catalog-service` ยังไม่ทันขึ้น) **รอ 400ms → ครั้งที่ 3 สำเร็จ** (`catalog-service` ขึ้นมาทันเวลาพอดี ที่ `02:02:10.844204Z`) — client ที่ยิง `curl` ไม่รู้เลยว่าเบื้องหลังมีการ retry เกิดขึ้น มันได้รับคำตอบที่ถูกต้อง (`409 NOT_AVAILABLE` เพราะเล่มนั้นหมดจริง — ระบบยังทำงานถูกต้องตาม business logic แม้จะมี retry เกิดขึ้นก่อนหน้า) โดยรอเพิ่มขึ้นแค่ประมาณ 600 มิลลิวินาที **ถ้าไม่มี retry เลย** request นี้จะ fail ทันทีตั้งแต่ครั้งแรกที่ `catalog-service` ยังไม่ทันขึ้นมา ทั้งที่จริง ๆ แค่รออีกไม่ถึงวินาทีก็สำเร็จได้แล้ว — นี่คือมูลค่าที่จับต้องได้ของ retry+backoff ในสถานการณ์ deploy จริง

**ข้อควรระวังที่สำคัญมาก (ย้อนกลับไปดูเงื่อนไข `is_retryable`)**: ฟังก์ชัน `call_with_retry` ในบทนี้ retry แค่ error code `Unavailable`/`DeadlineExceeded` เท่านั้น — **ไม่ retry** `NotFound`, `Unauthenticated`, หรือ error อื่น เพราะ retry ไม่ทำให้ error เหล่านั้นหายไปเลย (ถ้า token ผิด ยิงซ้ำ 10 ครั้งก็ยัง `Unauthenticated` เหมือนเดิม) — และสำคัญกว่านั้นคือ**เราเรียก `call_with_retry` กับ `CheckAvailability` เท่านั้น ไม่ใช่กับ `ReserveCopy`** เพราะ `ReserveCopy` **ไม่ใช่ idempotent อย่างปลอดภัย 100%** ในตัวอย่างนี้ — ถ้า `booking-service` ยิง `ReserveCopy` ไปแล้ว `catalog-service` ลด `available_copies` สำเร็จ แต่ response หายไปกลางทาง (network partition ระหว่างส่ง response กลับ) `booking-service` จะเห็นเป็น error แล้วถ้า retry ซ้ำแบบไม่ระมัดระวัง อาจเกิดการ reserve สองครั้งจริงทั้งที่ตั้งใจแค่ครั้งเดียว — นี่คือเหตุผลที่ในระบบจริงที่ต้องการ retry การเขียนข้อมูล (ไม่ใช่แค่การอ่าน) จำเป็นต้องออกแบบให้ operation นั้น idempotent จริง ๆ ก่อน (เช่น ใช้ `booking_id` เดียวกันเป็น idempotency key ให้ `catalog-service` ตรวจสอบว่าเคย reserve ด้วย id นี้ไปแล้วหรือยัง ก่อนจะลด `available_copies` ซ้ำ) ซึ่งเป็นรายละเอียดที่ลึกกว่าสโคปของบทนี้

#### Circuit Breaker: เมื่อ Retry อย่างเดียวไม่พอ

Retry with backoff แก้ปัญหา**ความล้มเหลวชั่วคราว**ได้ดี (เช่นสถานการณ์ deploy ข้างบน ที่ service กลับมาภายในไม่กี่ร้อยมิลลิวินาที) แต่มีสถานการณ์ที่ retry เพียงอย่างเดียว**ทำให้สถานการณ์แย่ลง**: ถ้า `catalog-service` **ล่มจริงจัง**เป็นเวลานาน (ไม่ใช่แค่กำลัง restart) การที่ `booking-service` ยังคง retry ทุก request ที่เข้ามาซ้ำ ๆ (แต่ละ request รอ 200+400+800ms ก่อนจะ fail ในที่สุด) จะทำให้:

- **`booking-service` เองก็ช้าลงมาก** เพราะทุก request ที่ต้องเช็ค availability ต้องรอ retry ครบ 4 ครั้งก่อนจะรู้ว่า fail (แทนที่จะ fail ทันทีตั้งแต่ต้น)
- **`catalog-service` ที่กำลังพยายามฟื้นตัว (เช่น restart แล้วกำลัง warm-up) ถูกถล่มด้วย retry จากทุก request ที่เข้ามาพร้อมกัน** ทำให้ฟื้นตัวได้ยากขึ้นไปอีก (thundering herd problem)

**Circuit breaker pattern** แก้ปัญหานี้ด้วยแนวคิดเหมือนเบรกเกอร์ไฟฟ้าจริง: หลังจากเห็น error จากปลายทางเดียวกันติดต่อกันเกิน threshold ที่กำหนด (เช่น 5 ครั้งใน 10 วินาที) circuit breaker จะ **"เปิด" (open)** และ**หยุดส่ง request ไปยังปลายทางนั้นทันทีโดยไม่ลองเรียกเลย** (ตอบ error ทันทีจาก client-side เอง) เป็นเวลาหนึ่ง ทำให้ปลายทางที่กำลังลำบากได้พักไม่ต้องรับโหลดเพิ่ม จากนั้นหลังจากเวลาที่กำหนด circuit breaker จะ**"ปิดครึ่งหนึ่ง" (half-open)** ลองส่ง request ทดสอบไปสองสามครั้ง — ถ้าสำเร็จก็ **"ปิด" (closed)** กลับสู่สภาวะปกติ ถ้ายังล้มเหลวก็เปิดต่อไป

บทนี้จะ**ไม่ implement circuit breaker เต็มรูปแบบเข้าไปในระบบจริง** (เป็นแนวคิดที่ลึกพอจะเป็นบทของตัวเองได้ และต้องออกแบบ state transition ให้ทนทานจริงจังกว่าที่สโคปของบทนี้ควรลงรายละเอียด) แต่เพื่อให้เห็นภาพว่า state machine ของมันหน้าตาเป็นอย่างไรในระดับแนวคิด (ไม่ใช่แค่คำอธิบายเป็นข้อความ) นี่คือ sketch ของ state machine นั้นเป็นโค้ด Rust ที่ compile ได้ถูกต้องตามหลักภาษา แต่**ยังไม่ได้เดินสายเข้ากับ `call_with_retry` ของหัวข้อ 81.4 จริง** (ทิ้งไว้เป็นทิศทางสำหรับศึกษาต่อ ไม่ใช่โค้ดที่ทดสอบรันจริงในบทนี้):

```rust
use std::time::{Duration, Instant};

#[derive(Debug, Clone, Copy, PartialEq)]
enum CircuitState {
    Closed,                 // ปกติ -- ปล่อยทุก request ผ่านไปเรียกปลายทางจริง
    Open { opened_at: Instant },   // เปิด -- ปฏิเสธทันทีโดยไม่เรียกปลายทางเลย
    HalfOpen,               // ทดสอบ -- ปล่อย request ทดสอบผ่านไปดูว่าปลายทางฟื้นหรือยัง
}

struct CircuitBreaker {
    state: CircuitState,
    consecutive_failures: u32,
    failure_threshold: u32,   // เปิด circuit เมื่อ fail ติดกันครบจำนวนนี้
    open_duration: Duration,  // เปิดค้างไว้นานแค่ไหนก่อนลองเป็น half-open
}

impl CircuitBreaker {
    /// เรียกก่อนจะยิง request จริงทุกครั้ง -- true แปลว่าอนุญาตให้เรียกปลายทางได้
    fn allow_request(&mut self) -> bool {
        match self.state {
            CircuitState::Closed => true,
            CircuitState::Open { opened_at } => {
                if opened_at.elapsed() >= self.open_duration {
                    self.state = CircuitState::HalfOpen;
                    true // อนุญาต request ทดสอบหนึ่งครั้ง
                } else {
                    false // ยังอยู่ในช่วงเปิด -- ปฏิเสธทันทีโดยไม่เรียกปลายทางเลย
                }
            }
            CircuitState::HalfOpen => true,
        }
    }

    fn record_success(&mut self) {
        self.consecutive_failures = 0;
        self.state = CircuitState::Closed; // ทดสอบผ่าน -- กลับสู่สภาวะปกติ
    }

    fn record_failure(&mut self) {
        self.consecutive_failures += 1;
        if self.state == CircuitState::HalfOpen || self.consecutive_failures >= self.failure_threshold {
            self.state = CircuitState::Open { opened_at: Instant::now() };
        }
    }
}
```

สังเกตว่า `allow_request` คือจุดที่ต่างจาก retry-with-backoff อย่างชัดเจน: retry (หัวข้อก่อนหน้า) **ยังพยายามเรียกปลายทางจริงทุกครั้ง** เพียงแต่เว้นช่วงเวลาระหว่างครั้ง ส่วน circuit breaker ที่อยู่ในสถานะ `Open` จะ**ไม่เรียกปลายทางเลยแม้แต่ครั้งเดียว** จนกว่าจะครบเวลา `open_duration` — นี่คือกลไกที่ปกป้อง service ปลายทางที่กำลังลำบากจากการถูกถล่มด้วย request ซ้ำ ๆ ระหว่างที่มันพยายามฟื้นตัวอยู่ ในระบบจริงมักนำ `CircuitBreaker` แบบนี้ไปประกอบเป็น `tower::Layer` ที่ครอบ `Channel` ของ Tonic client อีกชั้น (เพราะ Tonic client ก็เป็น `tower::Service` ตัวหนึ่งตามที่ Part 80 อธิบายไว้ — สามารถประกอบ layer ซ้อนกันได้ด้วย `tower::ServiceBuilder` แบบเดียวกับที่ Part 65 ใช้ประกอบ middleware หลายตัวเข้าด้วยกันฝั่ง REST) ทำให้ `call_with_retry` (retry) และ circuit breaker (ตัดการเรียกทั้งหมดชั่วคราว) ทำงานร่วมกันเป็นชั้น ๆ ได้ — ไม่ต้องเขียนทั้งสองอย่างปนกันในฟังก์ชันเดียว

สำหรับการใช้งานจริง ไม่จำเป็นต้องเขียน state machine นี้เองเสมอไป — ecosystem ของ Rust มีเครื่องมือสำเร็จรูปให้เลือก: **`tower::retry`** ให้ `Layer`/`Policy` สำหรับ retry ที่ประกอบเข้ากับ `tower::Service` ได้ตรง ๆ (แทนการเขียน `call_with_retry` เองแบบหัวข้อ 81.4) ส่วน circuit breaker มี crate เฉพาะทางอย่าง `failsafe` หรือการประกอบ `tower::Layer` เองตาม state machine ข้างบน — ทั้งหมดนี้คือทิศทางที่ควรไปศึกษาต่อเมื่อระบบโตขึ้นจนจำนวน service และความเสี่ยงของ cascading failure คุ้มค่ากับความซับซ้อนที่เพิ่มขึ้น (ตามเช็คลิสต์แบบเดียวกับหัวข้อ 81.1/81.10)

### 81.8 Observability ข้าม Service Boundary: Correlation ID ที่ไหลผ่านทุก Service

**Part 60** สอนไว้ว่า `tracing::info_span!` กับ `#[instrument]` ช่วยผูก `request_id` เข้ากับทุก log message ภายใน**process เดียว** ได้อย่างไร — ปัญหาที่เกิดขึ้นทันทีเมื่อแตกเป็น microservices คือ: **`request_id` ที่ `booking-service` สร้างขึ้น ไม่มีทางเดินทางไปถึง `catalog-service` เองได้โดยอัตโนมัติ** เพราะทั้งสองเป็นคนละ process กันสิ้นเชิง ไม่มีหน่วยความจำร่วมกันเลย — ถ้าไม่ทำอะไรเพิ่ม log ของ `catalog-service` จะมีแต่ span ของตัวเองที่ไม่เชื่อมโยงกับ request ต้นทางที่ `booking-service` เห็นเลย

#### วิธีแก้: ส่ง Request ID เดียวกันผ่าน gRPC Metadata

ดูโค้ดหัวข้อ 81.4 อีกครั้ง: `booking-service` สร้าง `request_id` **หนึ่งครั้งที่จุดแรกที่รับ request จาก client** (ที่ "ขอบ" ของระบบ — edge):

```rust
let request_id = Uuid::new_v4().to_string();
let span = tracing::info_span!(
    "handle_booking_request",
    request_id = %request_id,
    book_id = req.book_id
);
```

แล้วส่ง `request_id` เดียวกันนี้ไปเป็น gRPC metadata `x-request-id` ทุกครั้งที่เรียก `catalog-service` (ผ่าน `attach_metadata` ที่เห็นในหัวข้อ 81.5) ฝั่ง `catalog-service` อ่านค่านี้กลับมาผ่าน `extract_request_id` แล้วผูกมันเข้ากับ `tracing::info_span!` ของตัวเองด้วย field ชื่อเดียวกัน (`request_id`) — ผลลัพธ์คือ**ทั้งสอง process มี field ชื่อเดียวกัน ค่าเดียวกัน** สำหรับ request เดียวกัน แม้จะเป็นคนละ log stream กันโดยสิ้นเชิง

#### พิสูจน์จริงด้วย Log จริงจากทั้งสอง Service

จากการรันจริงในหัวข้อ 81.4 (การจองเล่ม id=2 ที่สำเร็จครั้งแรก) นี่คือ log ของ `booking-service`:

```text
2026-09-27T02:00:47.723334Z  INFO handle_booking_request{request_id=0f4fcdec-123a-4c00-bbfc-8ac090d76321 book_id=2}: booking_service: ได้รับคำขอจองหนังสือ user=user-42
2026-09-27T02:00:47.733078Z  INFO handle_booking_request{request_id=0f4fcdec-123a-4c00-bbfc-8ac090d76321 book_id=2}: booking_service: reserve สำเร็จที่ catalog-service booking_id=c3725c28-5c9e-4f16-9b1f-607004219016
2026-09-27T02:00:47.733144Z  INFO handle_booking_request{request_id=0f4fcdec-123a-4c00-bbfc-8ac090d76321 book_id=2}: booking_service: บันทึก booking สำเร็จในฐานข้อมูลของ booking-service เอง booking_id=c3725c28-5c9e-4f16-9b1f-607004219016
```

และนี่คือ log ของ `catalog-service` ที่บันทึกไว้**ในเวลาใกล้เคียงกันมาก (millisecond เดียวกัน)** สำหรับ request เดียวกัน:

```text
2026-09-27T02:00:47.726196Z  INFO check_availability{request_id=0f4fcdec-123a-4c00-bbfc-8ac090d76321 book_id=2}: catalog_service: ตรวจสอบความพร้อมของหนังสือ (synchronous call จาก booking-service)
2026-09-27T02:00:47.727730Z  INFO check_availability{request_id=0f4fcdec-123a-4c00-bbfc-8ac090d76321 book_id=2}: catalog_service: ตรวจสอบสำเร็จ available_copies=1
2026-09-27T02:00:47.729578Z  INFO reserve_copy{request_id=0f4fcdec-123a-4c00-bbfc-8ac090d76321 book_id=2 booking_id=c3725c28-5c9e-4f16-9b1f-607004219016}: catalog_service: พยายาม reserve หนังสือให้ booking
2026-09-27T02:00:47.732377Z  INFO reserve_copy{request_id=0f4fcdec-123a-4c00-bbfc-8ac090d76321 book_id=2 booking_id=c3725c28-5c9e-4f16-9b1f-607004219016}: catalog_service: reserve สำเร็จ
```

สังเกต `request_id=0f4fcdec-123a-4c00-bbfc-8ac090d76321` **ปรากฏเหมือนกันเป๊ะทั้งสองไฟล์ log ของสอง process ที่แยกกันสิ้นเชิง** — ถ้าผู้ใช้รายงานว่า "การจองของฉันช้า/ผิดพลาด" และมี `request_id` นี้ติดมา (เช่นผ่าน response header ที่เพิ่มเข้าไปได้ในระบบจริง) วิศวกรที่ debug สามารถ `grep 0f4fcdec` ในทั้งสอง log stream (แม้จะเก็บอยู่คนละไฟล์ คนละเครื่องกันในระบบจริง) แล้วเห็น**เส้นทางเต็มของ request นั้นข้าม service boundary ได้ทันที** — นี่คือความแตกต่างเชิงคุณภาพระหว่าง "ระบบที่ debug ได้" กับ "ระบบที่ debug ไม่ได้เลย" เมื่อมีหลาย service เกี่ยวข้อง

#### ทางไปสู่ Distributed Tracing เต็มรูปแบบ (Part 99)

สิ่งที่ทำไปข้างบนนี้คือ **"correlation ID แบบพื้นฐาน"** — เพียงพอสำหรับการ `grep` ข้าม log file ด้วยมือ แต่ในระบบที่มี service จำนวนมาก (สิบ ๆ ตัว) และ RPC หลายชั้นซ้อนกัน การ `grep` ด้วยมือไม่สเกล — นี่คือจุดที่ **OpenTelemetry** (มาตรฐานเปิดสำหรับ distributed tracing ที่ **Part 99** จะสอนเต็มรูปแบบ) เข้ามาเสริม: มันขยายแนวคิด "request_id เดียวไหลผ่านทุก service" ที่เราทำด้วยมือในบทนี้ ให้เป็น **"trace" ที่ประกอบจากหลาย "span"** อย่างเป็นมาตรฐาน (แต่ละ RPC เป็นหนึ่ง span, พ่อ-ลูกของ span เชื่อมกันด้วย trace context ที่ไหลผ่าน metadata แบบเดียวกับที่เราทำ) พร้อมเครื่องมือ visualize เป็น timeline ให้เห็นภาพรวมทั้งเส้นทางในเว็บ UI เดียว (เช่น Jaeger, Grafana Tempo) แทนการ `grep` ข้ามไฟล์ด้วยมือ — สิ่งที่บทนี้ทำคือรากฐานความคิดเดียวกัน เพียงแต่ยังไม่ใช้ library มาตรฐานหรือเครื่องมือ visualize เท่านั้น

### 81.9 Data Consistency ข้าม Service: ปัญหา Transaction ที่ทำไม่ได้อีกต่อไป และ Saga Pattern

#### ทวนปัญหาจาก Part 70: Transaction เดียวครอบคลุมได้แค่ในฐานข้อมูลเดียว

**Part 70** หัวข้อ 70.9 สอนไว้ว่า `pool.begin()` / `tx.commit()` / `tx.rollback()` การันตีว่าการดำเนินการหลายคำสั่ง SQL **ภายในฐานข้อมูลเดียวกัน** จะสำเร็จทั้งหมดหรือไม่สำเร็จเลย (atomicity) — ตัวอย่างคลาสสิกในบทนั้นคือ "ยืมหนังสือ: ลดจำนวนที่เหลือ + สร้าง record การยืม ต้องสำเร็จทั้งคู่หรือไม่สำเร็จเลย" ซึ่งทำได้เพราะทั้งสองคำสั่งอยู่ใน `pool` เดียวกัน ฐานข้อมูลเดียวกัน

**ปัญหาที่เกิดขึ้นทันทีในระบบของบทนี้**: การจองหนึ่งครั้งเกี่ยวข้องกับ**สองฐานข้อมูลที่แยกจากกันสิ้นเชิง** — `catalog-service` reserve หนังสือในฐานข้อมูลของมันเอง (`catalog_service`) และ `booking-service` บันทึก booking ในฐานข้อมูล/storage ของมันเอง (แยกกันคนละ process, คนละ connection pool, คนละเครื่องก็เป็นไปได้) **ไม่มีทางเรียก `pool.begin()` ตัวเดียวที่ครอบคลุมทั้งสองฐานข้อมูลนี้ได้เลย** — mechanism ของ Part 70 หยุดทำงานตรงเส้นแบ่งของ service พอดี

ทำไมทำ distributed transaction แบบ two-phase commit (2PC) ข้าม service ไม่ใช่คำตอบที่ดีในทางปฏิบัติ: มันต้องการให้ทั้งสอง service "lock" resource ของตัวเองรอกันจนกว่าทุกฝ่ายจะพร้อม commit พร้อมกัน — ถ้า service ใดฝ่ายหนึ่งช้าหรือค้าง ฝ่ายอื่นก็ต้อง lock resource ค้างรอไปด้วย (ลดความพร้อมใช้งานของทั้งระบบลงอย่างมาก) และการ implement 2PC ให้ทนทานจริงในโลกที่ network ไม่น่าเชื่อถือ (ตามที่หัวข้อ 81.7 อธิบายไว้) ซับซ้อนกว่าที่คุ้มค่าในกรณีส่วนใหญ่มาก

#### Saga Pattern: ลำดับของ Local Transaction เล็ก ๆ พร้อม Compensating Action

**Saga pattern** แก้ปัญหานี้ด้วยแนวคิดที่ต่างออกไปโดยสิ้นเชิง: แทนที่จะพยายามทำ transaction เดียวใหญ่ข้าม service ให้แตกการดำเนินการทั้งหมดเป็น **ลำดับของ local transaction เล็ก ๆ** ที่แต่ละอันอยู่ภายใน service เดียว (จึงใช้ transaction ปกติของ Part 70 ได้ตามเดิม) — ถ้าขั้นตอนใดขั้นตอนหนึ่งในลำดับล้มเหลว **ต้องมี "compensating action" (การดำเนินการชดเชย) ที่ย้อนผลของขั้นตอนก่อนหน้าที่สำเร็จไปแล้ว** เพื่อให้ระบบกลับสู่สถานะที่สมเหตุสมผล — สังเกตว่านี่**ไม่ใช่ rollback แบบ database transaction** (ที่ไม่มีอะไรเกิดขึ้นเลยถ้า rollback) แต่เป็น**การดำเนินการใหม่ที่ตั้งใจย้อนผลของการดำเนินการเดิม** (ต่างกันตรงที่ observer ภายนอกอาจเห็น "สถานะกลาง" ชั่วครู่ก่อนจะถูกชดเชย)

#### Saga ของระบบจองในบทนี้: Reserve-then-Confirm-or-Release

โค้ดในหัวข้อ 81.4 คือ implementation จริงของ Saga สองขั้นตอนสำหรับ flow การจอง:

```text
ก้าวที่ 1 (local transaction ที่ catalog-service):
   ReserveCopy(book_id, booking_id)
   → UPDATE books SET available_copies = available_copies - 1
     WHERE id = $1 AND available_copies > 0

ก้าวที่ 2 (local transaction ที่ booking-service):
   บันทึก booking ใหม่ใน storage ของ booking-service
   → ถ้าสำเร็จ: จบ saga ด้วยสถานะ "confirmed" (happy path)
   → ถ้าล้มเหลว: ต้องเรียก compensating action ของก้าวที่ 1

Compensating action (ชดเชยก้าวที่ 1):
   ReleaseCopy(book_id, booking_id)
   → UPDATE books SET available_copies = LEAST(available_copies + 1, total_copies)
     WHERE id = $1
```

**Happy path** (ทดสอบจริงในหัวข้อ 81.4): `ReserveCopy` สำเร็จ → บันทึก booking สำเร็จ → จบ saga ปกติ

**Compensation path** (ทดสอบจริงด้วย flag `simulate_local_failure`): `ReserveCopy` สำเร็จ (หนังสือถูกลดจำนวนไปแล้วจริงในฐานข้อมูลของ `catalog-service`) แต่ก้าวที่ 2 ล้มเหลว (จำลองด้วย flag) → `booking-service` เรียก `ReleaseCopy` เพื่อคืนสิทธิ์กลับ — นี่คือผลลัพธ์จริงจากการทดสอบ:

```text
คำขอที่ 1: POST /bookings {"book_id": 1, "simulate_local_failure": true}
ผลลัพธ์: {"error":{"code":"BOOKING_FAILED","message":"จองไม่สำเร็จ (จำลอง failure) -- ระบบได้คืนสิทธิ์ให้ catalog-service แล้ว"}}
HTTP_STATUS:500
```

log ของ `booking-service` ระหว่างการทดสอบนี้:

```text
2026-09-27T02:01:16.606357Z  INFO handle_booking_request{request_id=c51e35c6-... book_id=1}: booking_service: ได้รับคำขอจองหนังสือ user=user-42
2026-09-27T02:01:16.612669Z  INFO handle_booking_request{request_id=c51e35c6-... book_id=1}: booking_service: reserve สำเร็จที่ catalog-service booking_id=d594c3cb-75f8-4a85-bb5b-0114ae9c56dd
2026-09-27T02:01:16.612731Z ERROR handle_booking_request{request_id=c51e35c6-... book_id=1}: booking_service: จำลอง local failure ของ booking-service หลัง reserve สำเร็จแล้ว -- ต้อง compensate
2026-09-27T02:01:16.616376Z  INFO handle_booking_request{request_id=c51e35c6-... book_id=1}: booking_service: compensating action สำเร็จ: คืน copy ให้ catalog-service แล้ว
```

**คำขอที่ 2 ทันทีหลังจากนั้น** (จองเล่มเดิม `book_id=1` แบบปกติ ไม่จำลอง failure) **สำเร็จ** ด้วย `HTTP_STATUS:201` — นี่คือหลักฐานว่า compensating action ทำงานจริง: หนังสือที่ถูก reserve ไปแล้วแต่ saga ไม่จบสมบูรณ์ **ถูกคืนสิทธิ์กลับเข้าระบบจริง** ทำให้คนอื่นจองต่อได้ตามปกติ ไม่ได้ "ค้าง" อยู่ในสถานะ reserve ตลอดไปโดยไม่มีใครรู้

#### ข้อจำกัดที่ต้องเข้าใจอย่างตรงไปตรงมา: นี่คือจุดเริ่มต้นของหัวข้อใหญ่ ไม่ใช่ทั้งหมด

ตัวอย่างในบทนี้ทำให้เห็น**หลักการ**ของ Saga ชัดเจน แต่ยังขาดหลายเรื่องที่ระบบ production จริงต้องมี ถ้าต้องการนำแนวคิดนี้ไปใช้งานจริงจัง:

- **compensating action เองก็ล้มเหลวได้** — ในโค้ดหัวข้อ 81.4 ถ้าเรียก `ReleaseCopy` แล้ว fail (เช่น `catalog-service` ล่มพอดีตอนนั้น) โค้ดแค่ log error ไว้ (`tracing::error!("compensating action ล้มเหลว!! ...")`) แล้วปล่อยผ่าน — ระบบจริงต้องมีกลไก **retry การ compensate เองซ้ำ** หรือระบบ **reconcile/dead-letter queue** ที่ตรวจพบและแก้ไข inconsistency แบบนี้ในภายหลัง (มักทำเป็น background job ที่ตรวจสอบ "booking ที่ reserve ไว้นานเกินไปแต่ไม่มี confirmed booking คู่กัน" แล้วสั่ง release ซ้ำ)
- **ระหว่างที่ saga ยังไม่จบ มีช่วงเวลาที่ระบบอยู่ในสถานะ "กลาง ๆ"** — ระหว่าง `ReserveCopy` สำเร็จ กับก้าวถัดไปยังไม่จบ หนังสือเล่มนั้นถูกลดจำนวนไปแล้วจริง แม้ booking จะยังไม่ confirmed — ถ้ามีคนอื่นมาเช็ค availability พอดีช่วงนี้ จะเห็นจำนวนที่ลดไปแล้วทั้งที่ booking ยังไม่สำเร็จจริง (เรียกว่า "eventual consistency" — ระบบจะ consistent ในที่สุด แต่ไม่ใช่ทันทีเหมือน transaction เดียว) การออกแบบระบบต้องยอมรับและออกแบบรอบสภาวะนี้อย่างมีสติ ไม่ใช่สมมติว่าจะไม่เกิดขึ้น
- **ลำดับของ Saga ที่ซับซ้อนกว่าสองขั้นตอน** ต้องมี state machine ที่ชัดเจนกว่านี้ (มักใช้ pattern เรียกว่า **Saga orchestration** ที่มี "ผู้ควบคุม" กลางคอยสั่งแต่ละขั้นและ compensate ตามลำดับที่ถูกต้อง หรือ **Saga choreography** ที่แต่ละ service ฟัง event ของกันและกันแล้วตัดสินใจเอง — ทั้งสองแนวทางนี้มักต้องพึ่งพา message queue แบบที่ Part 82 จะสอน มากกว่า synchronous call ตรง ๆ แบบบทนี้)

#### แนวคิด Reconciliation Job: แก้ Inconsistency ที่ Compensating Action เองก็ล้มเหลว

ประเด็นแรกที่กล่าวถึงข้างบน ("compensating action เองก็ล้มเหลวได้") สำคัญพอที่จะขยายเป็นภาพร่างโค้ดให้เห็นแนวทางแก้ไข — แม้บทนี้จะไม่ implement เต็มรูปแบบ (ต้องมี scheduler/background worker ที่ Part 82 หรือโมดูล operations ในบทหลัง ๆ จะเกี่ยวข้องด้วย) แต่หลักการนี้สำคัญพอที่ทุกทีมที่ใช้ Saga pattern ต้องมี:

```rust
/// background job ที่รันเป็นระยะ (เช่นทุก 5 นาที) เพื่อค้นหา booking ที่อยู่ในสถานะ
/// "ค้าง" ผิดปกติ -- reserve ไปแล้วที่ catalog-service แต่ไม่มี booking ที่ confirmed
/// คู่กันในระบบของ booking-service เอง (แปลว่า saga ไม่จบสมบูรณ์และ compensating
/// action ครั้งก่อนอาจล้มเหลวไปด้วย)
async fn reconcile_stuck_reservations(
    catalog_client: &mut CatalogServiceClient<Channel>,
    bookings: &Mutex<HashMap<Uuid, Booking>>,
    stuck_booking_ids: Vec<(i64, String)>, // (book_id, booking_id) ที่สงสัยว่าค้าง
) {
    for (book_id, booking_id) in stuck_booking_ids {
        let confirmed_exists = bookings
            .lock()
            .unwrap()
            .values()
            .any(|b| b.id.to_string() == booking_id && b.status == "confirmed");

        if !confirmed_exists {
            tracing::warn!(
                book_id, booking_id = %booking_id,
                "พบ reservation ที่ค้าง -- ไม่มี booking ที่ confirmed คู่กัน กำลัง compensate ซ้ำ"
            );
            let mut release_req = tonic::Request::new(ReleaseCopyRequest {
                book_id,
                booking_id: booking_id.clone(),
            });
            // ในระบบจริง background job แบบนี้มักมี "system identity" ของตัวเอง
            // (ไม่ใช่ JWT ของผู้ใช้ปลายทาง) สำหรับเรียก service อื่น
            match catalog_client.release_copy(release_req).await {
                Ok(_) => tracing::info!(booking_id = %booking_id, "reconcile สำเร็จ"),
                Err(e) => tracing::error!(booking_id = %booking_id, "reconcile ล้มเหลวอีกครั้ง: {e}"),
            }
        }
    }
}
```

สังเกตว่า job นี้ทำหน้าที่เป็น **"ตาข่ายรองรับ" (safety net) ชั้นสุดท้าย** สำหรับ inconsistency ที่หลุดรอดจาก compensating action ปกติไป — มันไม่ได้แทนที่การ compensate ทันทีในหัวข้อ 81.4 (ที่ยังจำเป็นต้องมีเพื่อแก้ปัญหาให้เร็วที่สุดในกรณีส่วนใหญ่) แต่เป็นกลไกสำรองสำหรับกรณีที่แม้แต่การ compensate ทันทีนั้นก็ล้มเหลวไปด้วย — หลักการออกแบบที่สำคัญคือ**ต้องมีทาง "ตรวจจับ" สถานะที่ไม่สอดคล้องกันได้เสมอ** (ในตัวอย่างนี้คือเช็คว่า reservation มี booking ที่ confirmed คู่กันจริงไหม) ไม่ใช่แค่หวังว่า compensating action ครั้งแรกจะไม่ล้มเหลว

สรุปหัวข้อนี้อย่างตรงไปตรงมา: **Saga pattern ที่แสดงในบทนี้คือจุดเริ่มต้นของหัวข้อใหญ่ที่เรียกว่า "distributed data management" ไม่ใช่ framework ที่ครบถ้วนสมบูรณ์** — สิ่งที่สำคัญที่สุดที่ต้องนำไปคือ**หลักการ**: เมื่อไม่มี transaction เดียวให้พึ่งพาได้อีกต่อไป ทุกขั้นตอนที่เปลี่ยนแปลง state ต้อง**คิดล่วงหน้าไว้เสมอว่า "ถ้าขั้นตอนถัดไปล้มเหลว จะย้อนขั้นตอนนี้อย่างไร"** ตั้งแต่ตอนออกแบบ ไม่ใช่มาคิดตอน incident เกิดขึ้นจริงในระบบ production

### 81.10 เมื่อไหร่ควรเลือก Microservices จริง ๆ (และเมื่อไหร่ไม่ควร)

หลังจากเห็นภาพเต็มของทั้งประโยชน์และต้นทุนแล้ว มาสรุปเป็นเช็คลิสต์ที่ใช้ตัดสินใจได้จริง — ไม่ใช่แค่ "เพราะบทเรียนสอนไว้" แต่ต้องมีเหตุผลที่หนักแน่นระดับองค์กร:

#### เช็คลิสต์: อยู่ Monolith ต่อไปดีกว่าถ้า...

- ทีมวิศวกรมีขนาดเล็กถึงกลาง (ไม่กี่คนถึงไม่กี่สิบคน) ที่ยังประสานงานกันได้สะดวกในโค้ดฐานเดียว
- ทุกส่วนของระบบมีโหลด/รูปแบบการใช้งานใกล้เคียงกัน ไม่มีส่วนใดที่ต้อง scale ต่างจากส่วนอื่นอย่างมีนัยสำคัญ
- ยังไม่มีความจำเป็นต้อง deploy แต่ละส่วนแยกจากกันอิสระ (deploy ทั้งระบบพร้อมกันยังไม่เป็นคอขวดที่วัดผลกระทบได้จริง)
- ทีมยังไม่มีความพร้อมด้าน operations สำหรับดูแลหลาย service พร้อมกัน (monitoring, logging รวมศูนย์, CI/CD ต่อ service, service mesh ฯลฯ)
- ระบบยังอยู่ในช่วง product-market fit ที่ requirement เปลี่ยนเร็ว — การแตก service ตายตัวเกินไปตอนที่ domain boundary ยังไม่ชัดเจน มักทำให้ต้องรื้อ boundary ใหม่บ่อย ๆ ซึ่งยากกว่าการรื้อ module ภายใน monolith เดียวมาก

ในกรณีเหล่านี้ **Cargo workspace ที่โมดูลาไรซ์ดี** ตามหัวข้อ 81.1 คือคำตอบที่คุ้มค่ากว่ามาก — ได้ขอบเขตความรับผิดชอบที่ชัดเจนโดยไม่ต้องแลกกับ network overhead หรือ distributed complexity เลย

#### เช็คลิสต์: มีเหตุผลพอที่จะแตกเป็น Microservices ถ้า...

- **มีหลายทีมที่ต้อง deploy อิสระจากกันจริง** และการรอ deploy พร้อมกันเป็นคอขวดที่วัดผลกระทบได้ (เวลาที่เสียไป, feature ที่ delay)
- **มีความต้องการ scale ที่ต่างกันชัดเจนระหว่างส่วนต่าง ๆ ของระบบ** วัดเป็นตัวเลขได้จริง (เช่น endpoint หนึ่งรับ traffic 100 เท่าของอีก endpoint)
- **มีเหตุผลทางเทคนิคที่หนักแน่นในการใช้ stack ต่างกันสำหรับบางส่วน** (ไม่ใช่แค่ "อยากลองของใหม่")
- **ทีมมีความพร้อมด้าน operations แล้ว** (หรือกำลังจะลงทุนสร้างความพร้อมนั้น — ตรงกับที่ **Part 96 ขึ้นไป** ของหลักสูตรนี้จะสอน Docker, CI/CD, observability tooling ที่ทำให้การดูแลหลาย service เป็นไปได้จริงในทางปฏิบัติ)
- **domain boundary ของระบบชัดเจนและนิ่งพอสมควรแล้ว** (มักมาจากการอยู่ในสถานะ "monolith ที่โมดูลาไรซ์ดีด้วย Cargo workspace" มาสักระยะ จนเห็นขอบเขตที่แท้จริงชัดเจน ไม่ใช่เดาเอาตั้งแต่วันแรก)

**คำแนะนำเชิงปฏิบัติสุดท้ายของบทนี้**: ถ้าไม่แน่ใจว่าระบบของคุณอยู่ในกรณีไหน **ให้เอียงไปทาง monolith ที่โมดูลาไรซ์ดีเสมอ** เพราะต้นทุนของการแตกเป็น microservices ก่อนเวลาที่เหมาะสม (premature decomposition) สูงกว่าต้นทุนของการอยู่กับ monolith นานเกินไปแล้วค่อยแตกทีหลังมาก — การรวม service กลับเป็น monolith ทำได้ยากกว่าการแตก monolith ที่โมดูลาไรซ์ดีออกเป็น service มากนัก เพราะ boundary ที่ผิดพลาดมักฝังลึกเข้าไปในการออกแบบ database schema, API contract, และวิธีคิดของทีมไปแล้ว

#### ต้นทุนด้าน Operations ที่ต้องนับเป็นรายการจริง ไม่ใช่แค่ความรู้สึก "ยุ่งยากขึ้น"

เพื่อให้เช็คลิสต์ข้างบนจับต้องได้มากกว่าความรู้สึกเชิงคุณภาพ ("ดูแลยากขึ้น") มาดูรายการงาน operations ที่**เพิ่มขึ้นจริงเป็นตัวเลข**เมื่อมี service เพิ่มขึ้นแต่ละตัว เทียบระหว่างระบบสอง service ของบทนี้กับ monolith เดียว:

| งาน Operations | Monolith 1 ตัว | 2 Service (บทนี้) | N Service |
|---|---|---|---|
| Deployment pipeline ที่ต้องดูแล | 1 | 2 (แยกกันอิสระ) | N |
| ฐานข้อมูล/connection pool ที่ต้อง monitor | 1 | 2 (`catalog_service` + storage ของ `booking-service`) | N |
| จุดที่ต้องตั้ง alert เมื่อ error rate สูงผิดปกติ | 1 | 2 | N |
| จุดที่ health check/readiness probe ต้องตั้งค่า | 1 | 2 | N |
| ความสัมพันธ์ (dependency) ที่ต้องรู้ว่า "ถ้า X ล่ม จะกระทบอะไรบ้าง" | 0 (ไม่มีการเรียกข้ามระบบ) | 1 คู่ (`booking-service` → `catalog-service`) | เพิ่มแบบไม่เป็นเส้นตรง (จำนวนคู่ที่เป็นไปได้โตเร็วกว่าจำนวน service เอง) |
| key/secret ที่ต้อง distribute อย่างปลอดภัย (เช่น public key สำหรับ JWT) | 0 (auth logic อยู่ใน process เดียว) | ต้อง distribute public key ไปยังทั้งสอง service | ต้อง distribute ไปยังทุก service ที่ต้องตรวจสอบ JWT |

แถวสุดท้ายของตารางคือประเด็นที่มักถูกมองข้าม: **ความสัมพันธ์ระหว่าง service โตเร็วกว่าจำนวน service เอง** — ระบบที่มี 3 service มีความสัมพันธ์ที่เป็นไปได้สูงสุด 3 คู่ แต่ระบบที่มี 10 service มีได้ถึง 45 คู่ (`n × (n-1) / 2`) นี่คือเหตุผลเชิงตัวเลขที่ทำให้ทีมที่แตก service มากเกินไปโดยไม่มีเหตุผลหนักแน่นพอ เจอปัญหา "ไม่รู้ว่าอะไรกระทบอะไรบ้าง" เร็วกว่าที่คาดไว้มาก — และเป็นเหตุผลที่ observability (หัวข้อ 81.8) และ resilience pattern (หัวข้อ 81.7) ไม่ใช่ "ของแต่งเพิ่ม" แต่เป็น**สิ่งจำเป็นที่ต้นทุนโตตามจำนวน service** ตารางนี้เองคือเหตุผลที่ **Part 96 ขึ้นไป** ของหลักสูตรต้องมีอยู่ — เพื่อให้เครื่องมือ (Docker, CI/CD, observability platform) มาช่วยจัดการรายการงานเหล่านี้อย่างเป็นระบบ แทนที่จะทำด้วยมือทีละ service ซึ่งไม่สเกลเมื่อจำนวน service เพิ่มขึ้น

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. แชร์ฐานข้อมูลเดียวกันข้าม Service "เพราะง่ายกว่า"

นี่คือกับดักที่อธิบายไว้เต็มรูปแบบในหัวข้อ 81.2 แต่ต้องย้ำอีกครั้งเพราะพบบ่อยมากในทีมที่เริ่มทำ microservices ใหม่ ๆ: เห็นว่า `booking-service` ต้องรู้ราคา/ชื่อหนังสือบ่อย ๆ แล้วคิดว่า "connect ไปที่ database ของ `catalog-service` ตรง ๆ เลยง่ายกว่า ไม่ต้องผ่าน gRPC ให้ยุ่งยาก" — ผลลัพธ์คือได้ระบบที่มีต้นทุนทั้งหมดของ distributed system (deploy แยก, network, ops ซับซ้อนขึ้น) แต่**ไม่ได้ประโยชน์เรื่อง decoupling เลย** เพราะทั้งสอง service ยังผูกติดกับ schema เดียวกันอยู่ดี **วิธีแก้**: ทุกการเข้าถึงข้อมูลข้าม domain boundary ต้องผ่าน API ที่เจ้าของข้อมูลเปิดให้เท่านั้น แม้จะดูช้ากว่า/ยุ่งยากกว่าในระยะสั้น

### 2. ลืม Forward Authorization Metadata แล้วสงสัยว่าทำไม gRPC Call ถูกปฏิเสธ

ถ้าลืมเรียก `attach_metadata` ก่อนส่ง gRPC request (หรือลืมแนบ header `Authorization` ตั้งแต่ REST request แรกที่ยิงเข้า `booking-service`) จะได้ error ที่ดูเหมือน "ทำไม request ปกติ ๆ ถึง fail" ถ้าไม่อ่าน error message ให้ครบ — ทดสอบจริงในบทนี้ (ไม่แนบ `Authorization` header เข้า `booking-service` เลย):

```text
{"error":{"code":"UNAUTHENTICATED","message":"ไม่พบ Authorization header"}}
HTTP_STATUS:401
```

และถ้า `booking-service` ตรวจสอบผ่านแล้ว แต่ลืม forward metadata ต่อไปยัง `catalog-service` (สมมติลบ `attach_metadata` ทิ้งไปในโค้ด) `catalog-service` จะปฏิเสธด้วย error แบบเดียวกับที่ Part 80 หัวข้อ 80.8 พิสูจน์ไว้แล้ว:

```text
gRPC error: code=Unauthenticated message="ไม่พบ authorization metadata"
```

**วิธีแก้**: ตรวจสอบทุกจุดที่มีการเรียกข้าม service ว่า metadata (ทั้ง `authorization` และ `x-request-id`) ถูกแนบไปด้วยเสมอ — วิธีที่ปลอดภัยกว่าการเรียก `attach_metadata` แยกทุกจุดในโค้ด คือรวมมันเข้าไปเป็น interceptor ฝั่ง client (Tonic รองรับ client-side interceptor ได้เหมือน server-side ตามที่ Part 80 สอนไว้) เพื่อไม่ให้มีจุดใดในโค้ดที่เผลอลืมแนบไปได้

### 3. Retry Operation ที่ไม่ใช่ Idempotent อย่างไม่ระมัดระวัง

ถ้าใครลอกฟังก์ชัน `call_with_retry` ไปห่อ `ReserveCopy` ด้วย (ไม่ใช่แค่ `CheckAvailability`) โดยไม่คิดถึงผลที่ตามมา จะเจอปัญหาที่ตรวจจับยากมาก: สมมติ `catalog-service` ลด `available_copies` สำเร็จ แล้ว response หายไปกลางทาง (เช่น connection ถูกตัดพอดีตอนส่งกลับ) — `booking-service` เห็นเป็น error (`Unavailable` หรือ timeout) แล้ว retry ตามที่ตั้งไว้ → `catalog-service` ลด `available_copies` **ซ้ำอีกครั้ง** ทั้งที่ตั้งใจ reserve แค่ครั้งเดียว ผลคือหนังสือถูก "จอง" ไปสองสิทธิ์ทั้งที่มีการจองจริงครั้งเดียว **วิธีแก้**: retry เฉพาะ operation ที่เป็นอ่านข้อมูลอย่างเดียว (เช่น `CheckAvailability`) หรือ operation ที่ออกแบบให้ idempotent จริง ๆ ด้วย idempotency key (เช่นให้ `catalog-service` ตรวจสอบ `booking_id` ที่แนบมาก่อนว่าเคย reserve ไปแล้วหรือยัง ถ้าเคยแล้วให้ตอบ success กลับไปตรง ๆ โดยไม่ลดจำนวนซ้ำ) — บทนี้จงใจ**ไม่**ห่อ `ReserveCopy` ด้วย retry เพื่อหลีกเลี่ยงปัญหานี้โดยตรง

### 4. ใช้ `Channel::connect()` (Eager) แทน `connect_lazy()` แล้ว Service เริ่มทำงานผิดลำดับไม่ได้

ถ้าเปลี่ยนโค้ดหัวข้อ 81.4 จาก `.connect_lazy()` เป็น `.connect().await?` (การเชื่อมต่อแบบ eager ที่พยายามเปิด TCP connection จริงทันที) แล้ว start `booking-service` **ก่อน** ที่ `catalog-service` จะพร้อมรับการเชื่อมต่อ (ลำดับการ deploy/start ที่เกิดขึ้นจริงได้บ่อยมาก โดยเฉพาะตอน container orchestrator กำลัง roll out พร้อมกันหลาย service) จะได้ error ทันทีตั้งแต่ `booking-service` เริ่มทำงาน:

```text
Error: transport error

Caused by:
    tcp connect error: Connection refused (os error 111)
```

`booking-service` จะ**ไม่ยอมขึ้นมาเลย** ทั้งที่จริง ๆ ปัญหาแค่ "ยังไม่ทันเวลา" ไม่ใช่ configuration ผิด **วิธีแก้**: ใช้ `connect_lazy()` เสมอสำหรับ client ที่เรียก service อื่นซึ่งอาจไม่พร้อมตอน start (ตามที่บทนี้ทำ) — มันจะไม่พยายามเชื่อมต่อจริงจนกว่าจะมีการเรียกใช้ครั้งแรก และพยายามเชื่อมต่อใหม่ให้เองในทุกครั้งที่ครั้งก่อนเชื่อมต่อไม่ติด (ซึ่งเป็นกลไกเดียวกันที่ทำให้ retry+backoff ในหัวข้อ 81.7 ใช้งานได้จริงเมื่อ `catalog-service` restart กลับมา)

### 5. เข้าใจผิดว่า `jsonwebtoken` เวอร์ชันเดียวกันต้องใช้ในทุก Service

ถ้าคาดหวังว่า `catalog-service` และ `booking-service` ต้องใช้ `jsonwebtoken` เวอร์ชันเดียวกันเพราะ "คุยกันด้วย JWT เดียวกัน" จะแปลกใจเมื่อเห็นว่าจริง ๆ ทั้งสองฝั่ง resolve คนละเวอร์ชันได้ (ในบทนี้ `catalog-service` ได้ 9.3.1 ส่วน `booking-service` ได้ 11.1.0) — **นี่ไม่ใช่บั๊ก** เพราะทั้งสองฝั่งคุยกันผ่าน **JWT string** (ข้อความ base64url ธรรมดาตามมาตรฐาน RFC 7519 ที่ Part 74 สอนไว้) ไม่ใช่ผ่าน Rust type ที่ต้อง compile ร่วมกันเหมือนตอนเป็น monolith เดียว — ตราบใดที่ทั้งสองฝั่งใช้ RS256 กับ key pair เดียวกันถูกต้อง เวอร์ชันของ library ที่ใช้ทำงานภายในแต่ละ service ไม่จำเป็นต้องตรงกันเลย **ข้อควรระวังจริง**: jsonwebtoken เวอร์ชัน 11 ขึ้นไปต้องเปิด feature `rust_crypto` หรือ `aws_lc_rs` เอง (ตามกับดักที่ Part 74 หัวข้อ 74.3 อธิบายไว้) ไม่เช่นนั้นจะ panic ตอน runtime พร้อม error:

```text
thread 'main' panicked at .../jsonwebtoken-11.1.0/src/crypto/mod.rs:124:40:

Could not automatically determine the process-level CryptoProvider from jsonwebtoken crate features.
```

ต้องตรวจสอบ feature นี้เป็นพิเศษในทุก service ที่ใช้ `jsonwebtoken` เวอร์ชันใหม่ เพราะแต่ละ service resolve dependency ของตัวเองอิสระจากกันจริง — ความผิดพลาดในเวอร์ชัน/feature ของ service หนึ่งไม่ทำให้ `cargo build` ของ service อื่น fail ตามไปด้วยเลย ต้องทดสอบแต่ละ service แยกกันจริงจัง

### 6. แก้ไข `.proto` แล้ว Deploy Service ปลายทางก่อน โดยไม่คิดถึง Service ที่ยังใช้เวอร์ชันเก่าอยู่

ในระบบ monolith เดียว การเปลี่ยน struct หนึ่งตัวกระทบทุกจุดที่ใช้มันทันทีตอน `cargo build` (compiler จับให้เห็นหมดในที่เดียว) แต่ในระบบ microservices **`booking-service` และ `catalog-service` deploy แยกกันอิสระ** — ระหว่างการ deploy เวอร์ชันใหม่ (rolling deployment) มักมีบางช่วงเวลาที่ **`booking-service` เวอร์ชันเก่า (สร้างจาก `.proto` เวอร์ชันเก่า) ยังคุยกับ `catalog-service` เวอร์ชันใหม่ (สร้างจาก `.proto` เวอร์ชันใหม่) อยู่** — ถ้าทีมแก้ `.proto` โดยเปลี่ยน field number ที่มีอยู่แล้ว (ไม่ใช่แค่เพิ่ม field ใหม่) ตามกับดักที่ Part 80 หัวข้อ 80.2 เตือนไว้แล้ว ("เปลี่ยนเลข field number ทีหลัง = breaking change ทันที") การเรียกข้าม service ระหว่างสองเวอร์ชันนี้จะได้ข้อมูลผิดเพี้ยนโดยไม่มี error ให้เห็นเลย (เพราะ Protobuf ไม่ throw error เมื่อ decode ผิด tag — มันแค่อ่านค่าผิดไปเงียบ ๆ ตามกลไก wire format ที่ Part 80 อธิบายไว้)

**วิธีแก้**: ทุกการเปลี่ยน `.proto` ในระบบ microservices ต้องทำตามกฎ backward/forward compatibility ที่ Part 80 หัวข้อ 80.2 พิสูจน์ไว้แล้วอย่างเคร่งครัดกว่าตอนเป็น monolith มาก เพราะ**ไม่มีทางบังคับให้ทุก service deploy พร้อมกันเป๊ะได้จริงในทางปฏิบัติ** — เพิ่ม field ใหม่ได้เสมอ (ใช้ tag number ใหม่) แต่**ห้ามเปลี่ยนความหมาย/type ของ tag number เดิม และห้ามนำ tag number ที่เลิกใช้แล้วมาใช้ซ้ำ** (ใช้ `reserved` keyword กันไว้) นี่คือเหตุผลที่ทำให้ schema evolution ของ Protobuf (ไม่ใช่แค่ "มีก็ดี") เป็นคุณสมบัติที่**จำเป็นจริง**สำหรับระบบ microservices ไม่ใช่แค่ความสะดวกเสริม

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เพิ่ม RPC ใหม่ `ListBooks` (server streaming ตาม pattern ของ Part 80 หัวข้อ 80.6) ให้ `catalog-service` ที่ส่งหนังสือทั้งหมดในระบบกลับมาเป็น stream แล้วเพิ่ม endpoint `GET /books` ใน `booking-service` ที่เรียก RPC นี้แล้วรวบรวมผลลัพธ์เป็น JSON array ตอบ client
   - *Hint*: โครงสร้างเหมือน `ListBooks` ใน Part 80 หัวข้อ 80.6.2 ทุกประการ ฝั่ง client เรียก `.into_inner()` ได้ `Streaming<Book>` แล้ววนอ่านด้วย `stream.message().await` จนกว่าจะได้ `None`

2. **(กลาง)** เพิ่ม client-side interceptor ให้ `booking-service` (ตาม pattern ของ Part 80 หัวข้อ 80.8 ที่เป็น server-side interceptor — คราวนี้ให้ทำฝั่ง client) ที่แนบ `authorization` และ `x-request-id` metadata ให้อัตโนมัติทุก RPC โดยไม่ต้องเรียก `attach_metadata` แยกในแต่ละจุดของโค้ดเหมือนในบทนี้
   - *Hint*: `Interceptor` trait รับ `Request<()>` ก่อนส่งออกจริง ต้องหาวิธีส่ง `raw_token`/`request_id` เข้าไปในตัว interceptor (มักทำผ่าน `tokio::task_local!` เพราะค่าเหล่านี้ต่างกันไปตาม request ที่กำลังประมวลผล ไม่ใช่ค่าคงที่แบบ auth token เดียวที่ Part 80 สาธิตไว้)

3. **(ยาก)** implement idempotency key เต็มรูปแบบให้ `ReserveCopy` ของ `catalog-service` (ตามที่กับดักข้อ 3 ท้ายบทกล่าวถึง) — เพิ่มตาราง `reservations (booking_id TEXT PRIMARY KEY, book_id BIGINT, created_at TIMESTAMPTZ)` แล้วแก้ `reserve_copy` ให้ตรวจสอบก่อนว่า `booking_id` นี้เคย reserve ไปแล้วหรือยัง ถ้าเคยแล้วให้ตอบ `success: true` กลับไปตรง ๆ โดยไม่ลด `available_copies` ซ้ำ จากนั้นลองห่อ `ReserveCopy` ด้วย `call_with_retry` ที่มีอยู่แล้ว แล้วพิสูจน์ด้วยการจำลอง network failure (เช่น ปิด response กลางทางด้วยการ kill connection) ว่า retry ไม่ทำให้ `available_copies` ลดผิดพลาด
   - *Hint*: ใช้ `INSERT ... ON CONFLICT (booking_id) DO NOTHING` แล้วเช็ค `rows_affected()` ก่อนตัดสินใจว่าจะรัน `UPDATE books` จริงหรือไม่ — ทั้งสองคำสั่งต้องอยู่ใน transaction เดียวกันของ `catalog-service` (`pool.begin()`) เพื่อไม่ให้เกิด race condition ระหว่างสองคำสั่งนี้เอง

4. **(ยาก / ประยุกต์ใช้งานจริง)** เพิ่ม `user-service` ที่สามที่ implement เต็มรูปแบบตามแนวคิด Part 74-76 (signup/login, hash password ด้วย argon2, ออก JWT ด้วย RS256 private key จริง ผ่าน HTTP endpoint) แทนการใช้ `mint_token` bin ที่เป็นเครื่องมือทดสอบในบทนี้ — จากนั้นแก้ `booking-service` ให้เรียก `user-service` ผ่าน endpoint `POST /login` (เป็น synchronous call ธรรมดา ไม่ใช่ gRPC ก็ได้ เพราะเป็น public-facing API ตามที่ Part 80 หัวข้อ 80.1 อธิบายไว้ว่า REST เหมาะกับ client ที่ไม่ได้ควบคุมทั้งสองฝั่ง) แล้ววัด latency ของ flow เต็ม (login → booking) เทียบกับตอนที่ `booking-service` ตรวจสอบ JWT เองโดยไม่ต้องเรียก `user-service` เลย เพื่อยืนยันด้วยตัวเลขจริงว่าทำไม RS256 asymmetric signing (ที่ไม่ต้องเรียกกลับ auth-service ตอนตรวจสอบ) ถึงมีค่าจริงในเชิง performance
   - *Hint*: `user-service` implement ตาม pattern ของ Part 74 หัวข้อ 74.6-74.10 ทุกประการ เพียงแต่ private key ต้องอยู่ที่นี่เท่านั้น ห้าม `booking-service`/`catalog-service` เข้าถึงเด็ดขาด — ใช้ `std::time::Instant` วัดเวลาแบบเดียวกับที่ Part 80 หัวข้อ 80.1 วัด multiplexing เพื่อให้ได้ตัวเลขจริงมาเทียบ ไม่ใช่แค่คาดเดา

## สรุป

บทนี้เริ่มจากการวางกรอบความคิดที่ตรงไปตรงมาที่สุดเรื่อง microservices: **มันคือเครื่องมือแก้ปัญหาองค์กร/scaling ที่เฉพาะเจาะจง ไม่ใช่ default ที่ควรใช้เสมอ** — และ Cargo workspace จาก Part 17 คือขั้นแรกที่ราคาถูกกว่ามากที่ควรทำก่อนแตกเป็น service จริง จากนั้นเราแตกโดเมนห้องสมุด/การจองออกเป็น `catalog-service` และ `booking-service` ตามหลัก **data ownership** ที่แต่ละ service เป็นเจ้าของฐานข้อมูลตัวเองเท่านั้น สร้างทั้งสองให้รันได้จริงพร้อมกัน คุยกันผ่าน gRPC (ต่อจาก Part 80) ที่มีการส่งต่อ identity ด้วย RS256 JWT (ต่อจาก Part 74) ผ่าน metadata ให้ทั้งสองฝั่งตรวจสอบสิทธิ์ได้เองโดยไม่ต้องเรียกกลับไปยัง auth service เราพิสูจน์ resilience pattern (timeout, retry with exponential backoff) ด้วยการ kill/restart service จริงระหว่างการทดสอบ พิสูจน์ correlation ID ที่ไหลข้าม service boundary ด้วย log จริงที่มี request ID ตรงกัน และจบด้วย Saga pattern สำหรับปัญหา distributed transaction ผ่านตัวอย่าง reserve-then-confirm-or-release ที่ทำงานได้จริงทั้ง happy path และ compensation path

สิ่งที่บทนี้ **ไม่ได้ทำ** ก็สำคัญไม่แพ้กัน: ฝั่ง asynchronous messaging ที่หัวข้อ 81.3 แค่วางกรอบความคิดไว้ — **Part 82 (Message Queues: RabbitMQ/Kafka Integration)** จะเติมเต็มส่วนนี้เต็มรูปแบบ ให้เห็นว่าเมื่อไหร่ event-driven communication เหมาะกว่า synchronous call ที่บทนี้ใช้ และการทำงานร่วมกับ synchronous gRPC ที่เราสร้างในบทนี้จะออกแบบให้ประกอบกันอย่างไรในระบบจริง

---

**Part ก่อนหน้า:** [gRPC ด้วย Tonic](part-080-grpc-tonic.md) | **Part ถัดไป:** [Message Queues: RabbitMQ/Kafka Integration](part-082-message-queues.md)
