# Part 106: Rust Design Patterns สำหรับ Enterprise Applications

> โมดูล: Software Architecture และ Enterprise Patterns | ระดับ: มืออาชีพ | เวลาโดยประมาณ: 300 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายความแตกต่างระหว่าง **"object-level design pattern"** ที่ Part 52-53 สอนไว้ (Builder, Strategy, Observer,
  Newtype, Typestate, RAII) กับ **"application-architecture-level pattern"** ที่บทนี้สอน — และบอกได้ว่าทำไม
  โค้ดเบสที่มีผู้ร่วมเขียนหลายสิบคนและมีอายุหลายปีต้องการ pattern ระดับสถาปัตยกรรมเพิ่มเติม ไม่ใช่แค่ pattern
  ระดับ struct/trait เดี่ยว ๆ
- ออกแบบและเขียน **Hexagonal Architecture (Ports and Adapters)** เต็มรูปแบบ — แยก core business logic ออกจาก
  รายละเอียด infrastructure (ฐานข้อมูล, HTTP, message queue) ผ่าน trait ที่เป็น "port" แล้วให้ concrete type
  ที่เป็น "adapter" implement สัญญานั้น พร้อม refactor โค้ดจาก capstone Part 92-94 ให้ตรงตามแนวทางนี้จริง ทั้ง
  adapter ที่คุย PostgreSQL จริงผ่าน SQLx และ adapter ปลอมในหน่วยความจำสำหรับเทส
- ประยุกต์แนวคิดหลักของ **Domain-Driven Design (DDD)** ในโค้ด Rust จริง: **Value Object** (เช่น `Isbn` ที่
  validate ตอนสร้างเสมอ ต่อยอด newtype จาก Part 27/53), **Entity** (สิ่งที่มี identity คงอยู่ข้ามการเปลี่ยนแปลง)
  และ **Aggregate** (ขอบเขตความสอดคล้อง — consistency boundary — ที่การันตี invariant ของตัวเอง)
- เปรียบเทียบ **Repository pattern แบบ generic ตัวเดียว (`Repository<T, ID>`)** กับ **domain-specific
  repository trait แยกตาม aggregate** ด้วยโค้ดที่ compile ได้จริงทั้งสองแบบ และอธิบาย tradeoff ที่แท้จริงได้
  อย่างเจาะจง ไม่ใช่แค่ท่องจำว่า "อันไหนดีกว่า"
- แยก **Command กับ Query** ออกจากกันอย่างชัดเจนในระดับ type (CQS) และรู้ขอบเขตว่า **CQRS + Event Sourcing**
  คือ "เวอร์ชันที่ใหญ่กว่า" ของแนวคิดเดียวกัน พร้อมความซับซ้อนที่เพิ่มขึ้นมาก — และเมื่อไหร่ที่ความซับซ้อนนั้น
  คุ้มค่าจริง ๆ
- ออกแบบ **Dependency Injection แบบที่เป็น Rust จริง ๆ** (explicit constructor passing — "poor man's DI") แทน
  การมองหา DI framework แบบ Java/C#, รู้จัก crate อย่าง `shaku` และเมื่อไหร่ที่มันคุ้มกับต้นทุนด้าน "มองไม่เห็น
  ว่าอะไรต่อกับอะไร" ที่มันแลกมา
- ออกแบบ **สถาปัตยกรรม error handling ระดับโค้ดเบสใหญ่**: error type ย่อยต่อโมดูล รวมเข้า `AppError` ระดับแอป
  ผ่าน `From` (ต่อยอด Part 30/31/66) พร้อมเทียบกับแนวทาง "enum เดียวยักษ์" อย่างตรงไปตรงมา
- ใช้ **Cargo feature flags** แยก dependency ที่หนักออกจาก consumer ที่ไม่ต้องการมัน (เช่น โมดูล report สำหรับ
  แอดมินที่ดึง dependency หนักเข้ามา) พร้อมพิสูจน์ด้วย `cargo build`/`cargo build --features ...` จริง
- แยกแยะ **test double ทั้งห้าแบบ (dummy/stub/spy/mock/fake)** อย่างถูกต้องตามนิยาม ไม่ใช่ใช้คำเหล่านี้ปนกัน
  แบบที่มักเจอในวงการ พร้อมเขียน property-based test ด้วย `proptest` เพื่อยืนยัน invariant ของ domain logic

## ความรู้ที่ต้องมีมาก่อน

- **Part 52 (Design Patterns ใน Rust: Builder, Strategy, Observer)** และ **Part 53 (Design Patterns ใน Rust:
  Newtype, Typestate, RAII) — จำเป็นที่สุดในฐานะ "ครึ่งแรก" ของหัวข้อ design pattern ทั้งหมด**: บทนี้ไม่สอน
  Builder/Strategy/Observer/Newtype/Typestate/RAII ซ้ำ แต่จะ**อ้างอิงกลับไปใช้ตลอดทั้งบท** โดยเฉพาะ newtype
  pattern (Part 53 หัวข้อ 53.3-53.4) ที่เป็นฐานตรงของ Value Object ในหัวข้อ DDD ของบทนี้ ถ้าจำ "parse, don't
  validate" และเหตุผลสามข้อของ newtype ไม่ได้ ควรย้อนไปทวน Part 53 ก่อนอ่านหัวข้อ 106.3
- **Part 21 (Traits ขั้นสูง)**: trait object (`dyn Trait`), object safety, static เทียบ dynamic dispatch —
  ใช้ตลอดทั้งบทนี้ทุกครั้งที่พูดถึง "port" (trait) กับ "adapter" (concrete type ที่ implement trait นั้น) และ
  ใช้ตัดสินใจว่า `Box<dyn Trait>`/`Arc<dyn Trait>` เหมาะกับ dependency injection แบบไหน
- **Part 27 (Smart Pointers: Box)** และ **Part 53 หัวข้อ newtype**: `Isbn(String)`, `BookId(i64)` ในบทนี้คือ
  การประยุกต์ตรงของ newtype pattern ที่ทั้งสองบทสอนไว้แล้ว
- **Part 30 (Error Handling ขั้นสูง)** และ **Part 31 (thiserror, anyhow)**: การออกแบบ custom error type, `impl
  From`, การ propagate error ด้วย `?` — บทนี้จะขยายไปสู่ "สถาปัตยกรรม error ระดับหลายโมดูล" ที่ Part 30/31 ยัง
  ไม่ได้ลงรายละเอียดขนาดนี้
- **Part 35 (Cargo ขั้นสูง)**: `[features]`, optional dependency, `#[cfg(feature = "...")]` ระดับพื้นฐาน — บทนี้
  หัวข้อ 106.8 จะไปลึกกว่านั้นในบริบทของโค้ดเบสขนาดใหญ่ที่มีทีมหลายทีมใช้ crate เดียวกัน
- **Part 66 (Axum Error Handling)**: `AppError` enum, `impl IntoResponse for AppError`, การรวม error ทุกชนิด
  ให้ตอบ JSON shape เดียวกัน — `AppError` ในบทนี้คือ**วิวัฒนาการต่อ**ของ `AppError` ตัวเดียวกันนี้เมื่อโค้ดเบส
  โตขึ้นจนมีหลายโมดูล
- **Part 70-71 (SQLx: PostgreSQL, Queries, Migrations)**: `PgPool`, `sqlx::query_as`, transaction — จำเป็นสำหรับ
  เข้าใจ adapter ที่คุย PostgreSQL จริงในหัวข้อ 106.2
- **Part 92-94 (Full-Stack Capstone: Backend, Frontend, Integration)**: บทนี้ **refactor สไลซ์จริงของ backend
  Part 92** (ระบบยืม-คืนหนังสือของห้องสมุด, `AppState`, `AppError`, `BookResponse`) ให้เป็นไปตาม hexagonal
  architecture — ถ้าไม่คุ้นกับโครงสร้างโปรเจกต์ของ Part 92 ควรย้อนไปดูก่อน (ไม่จำเป็นต้องอ่านทั้งหมด แค่คุ้นกับ
  `AppState { db: PgPool, .. }` และ `AppError` ที่ Part 92 หัวข้อ 92.4 ต่อยอดจาก Part 66 มา)
- **Part 95 (Testing Web Apps)**: testing pyramid สี่ชั้น, trait-based mocking ผ่าน `BookRepository` trait +
  `InMemoryBookRepository`, `#[sqlx::test]` — บทนี้ใช้ `BookRepository` trait ตัวเดียวกันเป๊ะเป็นจุดตั้งต้นของ
  หัวข้อ hexagonal architecture (106.2) แล้วขยายให้ลึกและกว้างขึ้นในบริบทของสถาปัตยกรรมทั้งระบบ ไม่ใช่แค่เพื่อ
  การเทส
- **Part 32-33 (Testing: Unit Tests, Integration Tests)**: `#[test]`, การ mock ผ่าน trait, ภาพรวม testing
  pyramid เวอร์ชันแรก — ใช้เป็นฐานของหัวข้อ test double hierarchy (106.9) ที่ขยายคำศัพท์ dummy/stub/spy/mock/
  fake ให้แม่นยำขึ้น

## เนื้อหา

### 106.1 จาก Object-Level Pattern สู่ Application-Architecture-Level Pattern

Part 52 และ Part 53 สอน pattern หกแบบที่ทำงานอยู่ใน**ระดับของ struct/trait เดี่ยว ๆ**: Builder ช่วยสร้าง object
ที่มีหลาย field optional, Strategy ช่วยสลับอัลกอริทึมย่อยหนึ่งจุด, Observer ช่วยแจ้งเตือนระหว่าง object, Newtype
ช่วยห่อ type ให้ปลอดภัยขึ้น, Typestate ช่วยบังคับลำดับการเรียก method ผ่าน type system, RAII ช่วยผูก resource
เข้ากับ scope — **ทุก pattern เหล่านี้ตอบคำถามระดับ "struct/function นี้ควรออกแบบยังไง"** ซึ่งเป็นคำถามที่ตอบได้
โดยดูแค่ไฟล์เดียวหรือ module เดียวเป็นหลัก ไม่ต้องมองภาพรวมทั้งระบบ

คำถามที่บทนี้ตอบต่างออกไปโดยพื้นฐาน: **"โค้ดเบสทั้งระบบที่มีคนเขียนหลายสิบคน อายุหลายปี ควรจัดวางโครงสร้าง
อย่างไร เพื่อให้เปลี่ยนแปลงได้อย่างปลอดภัยและทดสอบได้อย่างรวดเร็ว แม้ business logic core จะซับซ้อนขึ้นเรื่อย ๆ
และ infrastructure ที่ต้องคุยด้วย (ฐานข้อมูล, message queue, external API) เปลี่ยนไปตามยุค"** — นี่คือคำถามที่
ตอบไม่ได้ด้วยการดูไฟล์เดียว ต้องมองที่**ความสัมพันธ์ระหว่างโมดูล** (โมดูล A depend on โมดูล B ผ่านอะไร),
**ทิศทางของ dependency** (ใคร depend on ใคร ใครไม่ควร depend on ใคร), และ **ขอบเขตของความรับผิดชอบ** (โมดูลไหน
รู้เรื่อง SQL, โมดูลไหนรู้เรื่อง HTTP, โมดูลไหนรู้แค่กฎทางธุรกิจล้วน ๆ)

ปัญหาที่เกิดขึ้นจริงเมื่อโค้ดเบสโตขึ้นโดยไม่มี pattern ระดับสถาปัตยกรรมช่วย มีลักษณะซ้ำ ๆ กันในหลายทีม:

- **Business logic ปนกับ SQL** จนทดสอบ logic เพียว ๆ ไม่ได้เลยโดยไม่เปิดฐานข้อมูลจริง (แก้ด้วย **Hexagonal
  Architecture** — หัวข้อ 106.2)
- **field ที่ควรมี invariant การันตี (เช่น ISBN ต้องถูกต้องตามรูปแบบเสมอ) กลับเป็น `String` เปล่า ๆ** ที่ validate
  กระจัดกระจายคนละจุดในโค้ด ทำให้ลืม validate บางจุดได้ง่าย (แก้ด้วย **DDD Value Object** — หัวข้อ 106.3)
- **method ของ repository ไม่ตรงกับที่ธุรกิจต้องการจริง** เพราะพยายามยัดทุก entity ให้ใช้ trait กลางตัวเดียว
  (ตัดสินใจ tradeoff ที่ถูกต้องด้วย **Repository pattern เจาะลึก** — หัวข้อ 106.4)
- **service layer เต็มไปด้วย method ที่ทำทั้งอ่านและเขียนปนกันจนตามยาก** ว่า method ไหนมีผลข้างเคียงบ้าง (แก้ด้วย
  **CQS** — หัวข้อ 106.5)
- **สร้าง dependency ใหม่แล้วต้องไล่แก้ constructor ทุกจุดที่เกี่ยวข้อง หรือกลับกัน หา DI container สำเร็จรูปมาใช้
  แล้วดูไม่ออกว่าอะไรต่อกับอะไร** (ตัดสินใจอย่างมีเหตุผลด้วย **DI แบบ Rust** — หัวข้อ 106.6)
- **error จากโมดูล A รั่วรายละเอียดภายในไปถึงโมดูล Z ที่ไม่ควรรู้เรื่องนั้นเลย** หรือ **enum error เดียวยักษ์ที่
  ทุกคนกลัวจะแก้เพราะกระทบทุกที่** (แก้ด้วย **error architecture per-module + From** — หัวข้อ 106.7)
- **ทุก consumer ของ crate ต้อง compile dependency หนัก ๆ ที่ตัวเองไม่ได้ใช้เลย** (แก้ด้วย **feature flags** —
  หัวข้อ 106.8)
- **เทสเรียก "mock" ทุกตัวว่าเหมือนกันหมด ทั้งที่พฤติกรรมต่างกันโดยสิ้นเชิง** ทำให้สื่อสารกันผิดพลาดในทีม (แก้ด้วย
  **test double hierarchy ที่แม่นยำ** — หัวข้อ 106.9)

สังเกตว่า pattern ทั้งหมดในบทนี้**ไม่ได้ขัดแย้งกับ Part 52-53 เลย** — ในทางกลับกัน มันมักถูก**สร้างขึ้นจาก**
pattern ระดับ object เหล่านั้น เช่น Value Object ใน 106.3 คือ newtype ที่มี invariant (Part 53 หัวข้อ 53.4) นำมา
ใช้ในบริบทของ domain model, Repository ใน 106.4 คือ Strategy pattern (Part 52 หัวข้อ 52.7) ที่นำ `dyn Trait`
มาใช้แยก "อะไรที่ต้องทำ" ออกจาก "ทำอย่างไร" ในระดับของการเข้าถึงข้อมูลทั้งก้อน — pattern ระดับสถาปัตยกรรมมักเป็น
**การนำ pattern ระดับ object มาประกอบกันในสเกลที่ใหญ่ขึ้น** ไม่ใช่เครื่องมือชุดใหม่ที่ไม่เกี่ยวข้องกันเลย

### 106.2 Hexagonal Architecture (Ports and Adapters)

#### 106.2.1 ปัญหา: Business Logic ที่ผูกติดกับ SQLx ตรง ๆ

ทวนโค้ดจาก Part 92 หัวข้อ 92.9-92.10 (handler `borrow_book`): logic การยืมหนังสือเขียนอยู่**ในฟังก์ชัน handler
ของ Axum โดยตรง** และเรียก `sqlx::query_scalar!`/`sqlx::query_as!` ตรง ๆ กลางฟังก์ชัน — โครงสร้างแบบนี้ใช้งานได้ดี
สำหรับแอปขนาดเล็กถึงกลาง (Part 92 ทำถูกต้องแล้วสำหรับขนาดของบทนั้น) แต่เมื่อ business logic ซับซ้อนขึ้น (เช่น
ต้องเช็คโควต้าการยืมของสมาชิก, คำนวณค่าปรับ, ส่ง event แจ้งเตือน) ปัญหาสามข้อนี้จะเริ่มปรากฏชัด:

1. **ทดสอบ logic การยืมล้วน ๆ ไม่ได้โดยไม่เปิด PostgreSQL จริง** — ทุก unit test ที่อยากทดสอบ "ยืมหนังสือที่ไม่มี
   สำเนาว่างต้องถูกปฏิเสธ" ต้องแบก `#[sqlx::test]` ทั้งชุดไปด้วย (ช้ากว่าที่ควรมาก ตาม testing pyramid Part 95)
2. **สลับ backend ไม่ได้เลยโดยไม่แก้โค้ด handler** — ถ้าอนาคตต้องย้ายจาก PostgreSQL ไป backend อื่น (หรือแค่
   อยากมี in-memory cache layer คั่นกลาง) ต้องไล่แก้ SQL ที่กระจายอยู่ในทุก handler
3. **ทิศทาง dependency ผิดด้าน** — business logic (สิ่งที่มีค่าทางธุรกิจมากที่สุด ควรอยู่ได้นานที่สุด) กลับ
   **depend on** รายละเอียด infrastructure (SQLx, ที่อาจเปลี่ยนเวอร์ชัน/เปลี่ยนไปเป็นคนละ library ได้ในอนาคต)
   ทั้งที่ควรเป็นตรงกันข้าม

**Hexagonal Architecture** (เรียกอีกชื่อว่า **Ports and Adapters**, เสนอโดย Alistair Cockburn) แก้ปัญหานี้ด้วย
กฎเดียวที่ตรงไปตรงมา: **core business logic ต้อง depend on trait ("port") เท่านั้น ไม่ depend on concrete
infrastructure type ตรง ๆ เด็ดขาด** — ส่วน concrete type ที่คุยกับโลกภายนอกจริง (SQLx, HTTP client, message
queue client) เรียกว่า **"adapter"** ซึ่ง**implement port** นั้น ทิศทางของ dependency พลิกกลับสมบูรณ์: จากที่
core logic เคย depend on SQLx ตรง ๆ กลายเป็น **adapter (SQLx) depend on port (trait) ที่ core logic กำหนดไว้**
ต่างหาก — core logic ไม่รู้จัก SQLx อยู่จริงหรือไม่เลยด้วยซ้ำ

ชื่อ "hexagonal" มาจากภาพประกอบดั้งเดิมที่วาด core logic เป็นรูปหกเหลี่ยมตรงกลาง ล้อมด้วย "port" หลายด้าน (แต่ละ
ด้านของหกเหลี่ยมแทน trait หนึ่งตัว) แล้วมี "adapter" หลายตัวเสียบเข้ากับแต่ละ port จากด้านนอก — เลขหกไม่มี
ความหมายพิเศษ (ไม่ได้แปลว่าต้องมี port หกตัวเป๊ะ) มันแค่สื่อว่า **core อยู่ตรงกลาง ไม่มีด้านไหน "พิเศษ" กว่ากัน**
(ต่างจากภาพ layered architecture แบบดั้งเดิมที่วาดเป็นชั้น ๆ ซ้อนกัน ซึ่งมักทำให้คนเข้าใจผิดว่า database layer
"อยู่ล่างสุด" แปลว่า "สำคัญที่สุด" ทั้งที่ในความเป็นจริง business logic ควรเป็นศูนย์กลางมากกว่า)

#### 106.2.2 Port: `BookRepository` Trait

Part 95 หัวข้อ testing pyramid ได้แนะนำ `BookRepository` trait ไว้แล้วในบริบทของการทำ unit test เร็ว — บทนี้จะ
นำ trait ตัวเดียวกันมาขยายให้สมบูรณ์ขึ้น (เพิ่ม `find_by_isbn` และ `save` ที่ใช้ทั้งตอนสร้างใหม่และตอนอัปเดต) แล้ว
มองมันในฐานะ **"port" ของ hexagonal architecture** อย่างเป็นทางการ ไม่ใช่แค่เครื่องมือช่วยเทสอีกต่อไป:

```rust
// src/ports.rs
use async_trait::async_trait;

// สมมติว่า Book, BookId, Isbn, RepoError มาจาก module อื่นในเนื้อหาถัดไป (106.3, 106.7)
# struct Book;
# #[derive(Clone, Copy)] struct BookId(i64);
# struct Isbn(String);
# #[derive(Debug)] struct RepoError;
# impl std::fmt::Display for RepoError { fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result { write!(f, "repo error") } }
# impl std::error::Error for RepoError {}

/// "Port" ของ Hexagonal Architecture — trait ที่แทน "ความสามารถที่ core logic ต้องการจากโลกภายนอก"
/// core logic (application service ใน commands.rs) คุยกับ trait นี้เท่านั้น ไม่รู้จัก `sqlx`/`PgPool`/
/// HashMap เลยแม้แต่นิดเดียว — "Adapter" (ในโฟลเดอร์ adapters/) คือผู้ implement สัญญานี้ให้จริง
#[async_trait]
pub trait BookRepository: Send + Sync {
    async fn list(&self) -> Result<Vec<Book>, RepoError>;
    async fn find_by_id(&self, id: BookId) -> Result<Option<Book>, RepoError>;
    async fn find_by_isbn(&self, isbn: &Isbn) -> Result<Option<Book>, RepoError>;
    /// บันทึกสถานะปัจจุบันของ `Book` ทั้งก้อน (ใช้ทั้งตอนสร้างใหม่และตอนอัปเดต available_copies)
    async fn save(&self, book: &Book) -> Result<(), RepoError>;
}
```

สังเกตรายละเอียดสำคัญสองจุด: **`#[async_trait]`** ทวนจาก Part 95 — เพราะ Rust (ณ ขณะที่เขียนหลักสูตรนี้) ยังไม่
รองรับ `async fn` ใน trait ที่ต้องใช้เป็น `dyn Trait` object ได้ตรง ๆ โดยไม่มี macro ช่วย (`async fn` ใน trait
ธรรมดา generate เป็น associated type ที่ผูกกับ concrete future type ของแต่ละ implementation ทำให้ trait นั้นไม่
"dyn-compatible" — ทวนแนวคิด object safety จาก Part 21) `async_trait` macro แปลง signature ให้คืน `Pin<Box<dyn
Future<...>>>` แทน ทำให้ trait ยังใช้เป็น `dyn BookRepository`/`Arc<dyn BookRepository>` ได้ — และ **`: Send +
Sync`** บน trait bound ไม่ใช่ตัวเลือก แต่**จำเป็น**สำหรับระบบจริงที่ใช้ Axum/tokio เพราะ `AppState` ที่เก็บ
`Arc<dyn BookRepository>` ต้องถูกส่งข้าม thread ได้เมื่อ handler ถูกเรียกจากหลาย request พร้อมกัน (จะเห็น error
จริงถ้าลืมบวก bound นี้ในหัวข้อกับดักท้ายบท)

#### 106.2.3 Adapter ของจริง: `PgBookRepository`

```rust
// src/adapters/postgres.rs
use async_trait::async_trait;
use sqlx::PgPool;
# use crate::domain::book::{Book, BookId};
# use crate::domain::isbn::Isbn;
# use crate::ports::BookRepository;
# use crate::repo_error::RepoError;

/// row ดิบที่ map ตรงจากตาราง `books` — เก็บ `isbn` เป็น `String` เปล่า ๆ เพราะฐานข้อมูล**ไม่รู้จัก**
/// value object `Isbn` ของเรา (มันเป็นแนวคิดของ domain layer เท่านั้น) การแปลง `String` -> `Isbn` (ที่ตรวจสอบ
/// checksum) เกิดขึ้นตรงขอบของ adapter นี่แหละ — ไม่ใช่ในโค้ด SQL และไม่ใช่ใน core logic
#[derive(sqlx::FromRow)]
struct BookRow {
    id: i64,
    title: String,
    author: String,
    isbn: String,
    total_copies: i32,
    available_copies: i32,
}

impl TryFrom<BookRow> for Book {
    type Error = RepoError;

    fn try_from(row: BookRow) -> Result<Self, Self::Error> {
        let isbn = Isbn::try_from(row.isbn.as_str())
            .map_err(|e| RepoError::Backend(format!("ข้อมูลใน DB มี isbn ที่ไม่ถูกต้อง: {e}")))?;
        Ok(Book {
            id: BookId(row.id),
            title: row.title,
            author: row.author,
            isbn,
            total_copies: row.total_copies,
            available_copies: row.available_copies,
        })
    }
}

/// Adapter "ของจริง" — คุยกับ PostgreSQL ผ่าน `sqlx::PgPool` (Part 70/71) จริง ๆ
/// ใช้ query builder แบบ runtime (`sqlx::query_as::<_, T>(...)`) แทน macro `query_as!` ที่ตรวจสอบ
/// ตอน compile เพราะ macro นั้นต้องเชื่อมต่อฐานข้อมูลจริง (หรือมี `.sqlx/` cache) ตอน `cargo build` — โค้ด
/// ในบทนี้จึงเลือก runtime API เพื่อให้ยืนยันได้ว่า **compile และ implement trait ถูกต้อง** โดยไม่ผูกกับ
/// การมี Postgres รันอยู่จริงตอนสอน (โปรเจกต์จริงที่มี Postgres ให้ใช้ตอน dev สามารถสับไปใช้ `query_as!`
/// เพื่อได้ compile-time SQL checking แบบ Part 92 ได้เลยโดยไม่ต้องเปลี่ยนโครงสร้าง adapter นี้เลย)
pub struct PgBookRepository {
    pub pool: PgPool,
}

#[async_trait]
impl BookRepository for PgBookRepository {
    async fn list(&self) -> Result<Vec<Book>, RepoError> {
        let rows: Vec<BookRow> = sqlx::query_as(
            "SELECT id, title, author, isbn, total_copies, available_copies FROM books ORDER BY id",
        )
        .fetch_all(&self.pool)
        .await?;

        rows.into_iter().map(Book::try_from).collect()
    }

    async fn find_by_id(&self, id: BookId) -> Result<Option<Book>, RepoError> {
        let row: Option<BookRow> = sqlx::query_as(
            "SELECT id, title, author, isbn, total_copies, available_copies FROM books WHERE id = $1",
        )
        .bind(id.0)
        .fetch_optional(&self.pool)
        .await?;

        row.map(Book::try_from).transpose()
    }

    async fn find_by_isbn(&self, isbn: &Isbn) -> Result<Option<Book>, RepoError> {
        let row: Option<BookRow> = sqlx::query_as(
            "SELECT id, title, author, isbn, total_copies, available_copies FROM books WHERE isbn = $1",
        )
        .bind(isbn.as_str())
        .fetch_optional(&self.pool)
        .await?;

        row.map(Book::try_from).transpose()
    }

    async fn save(&self, book: &Book) -> Result<(), RepoError> {
        sqlx::query(
            "INSERT INTO books (id, title, author, isbn, total_copies, available_copies)
             VALUES ($1, $2, $3, $4, $5, $6)
             ON CONFLICT (id) DO UPDATE SET
                title = EXCLUDED.title,
                author = EXCLUDED.author,
                isbn = EXCLUDED.isbn,
                total_copies = EXCLUDED.total_copies,
                available_copies = EXCLUDED.available_copies",
        )
        .bind(book.id.0)
        .bind(&book.title)
        .bind(&book.author)
        .bind(book.isbn.as_str())
        .bind(book.total_copies)
        .bind(book.available_copies)
        .execute(&self.pool)
        .await?;

        Ok(())
    }
}
```

โค้ดนี้คอมไพล์ผ่านจริง (ยืนยันในหัวข้อ "การตรวจสอบ" ท้ายบท) โดยไม่ต้องมี PostgreSQL รันอยู่เลยระหว่าง `cargo
build` — เพราะเลือกใช้ `sqlx::query_as(...)` (runtime API ที่รับ SQL เป็น `&str` ธรรมดา) แทน macro
`sqlx::query_as!(...)` ที่ Part 70-71/92 ใช้ (ที่ตรวจสอบ SQL กับ schema จริงตอน compile) — นี่คือ**tradeoff ที่
ต้องรู้จริง**: macro ให้ compile-time safety ที่แข็งแรงกว่ามาก (พิมพ์ชื่อ column ผิดจะ error ตั้งแต่ `cargo
build`) แต่ต้องมี database (หรือ `.sqlx/` offline cache ที่สร้างจาก database) ตอน build เสมอ — โปรเจกต์จริงที่มี
PostgreSQL ให้ dev เชื่อมต่อได้ตลอดควรใช้ macro แบบ Part 92 เป็นค่าเริ่มต้น ส่วนโค้ดตัวอย่างในบทเรียนที่ต้อง
compile ได้แน่นอนโดยไม่ผูกกับ infrastructure ภายนอก จึงเลือก runtime API แทน — โครงสร้าง adapter ทั้งหมด
**เหมือนกันทุกประการ**ไม่ว่าจะเลือกแบบไหน มีแค่วิธีเขียน query แต่ละคำสั่งที่ต่างกัน

#### 106.2.4 Adapter ปลอมสำหรับเทส: `InMemoryBookRepository`

```rust
// src/adapters/in_memory.rs
use async_trait::async_trait;
use std::collections::HashMap;
use std::sync::Mutex;
# use crate::domain::book::{Book, BookId};
# use crate::domain::isbn::Isbn;
# use crate::ports::BookRepository;
# use crate::repo_error::RepoError;

/// Adapter "ของปลอมที่ใช้งานได้จริง" (fake) — เก็บข้อมูลใน `HashMap` ในหน่วยความจำ ไม่แตะ I/O เลย
/// ใช้เป็น test adapter สำหรับ unit test (เร็ว ไม่ต้องมี Postgres จริง ตาม Part 95 testing pyramid)
/// และใช้สาธิต payoff ของ hexagonal architecture: สลับ adapter นี้เข้า/ออกได้โดย core logic ไม่รู้ตัวเลย
pub struct InMemoryBookRepository {
    books: Mutex<HashMap<i64, Book>>,
}

impl InMemoryBookRepository {
    pub fn new(seed: Vec<Book>) -> Self {
        let books = seed.into_iter().map(|b| (b.id.0, b)).collect();
        InMemoryBookRepository { books: Mutex::new(books) }
    }
}

#[async_trait]
impl BookRepository for InMemoryBookRepository {
    async fn list(&self) -> Result<Vec<Book>, RepoError> {
        let mut books: Vec<Book> = self.books.lock().unwrap().values().cloned().collect();
        books.sort_by_key(|b| b.id.0);
        Ok(books)
    }

    async fn find_by_id(&self, id: BookId) -> Result<Option<Book>, RepoError> {
        Ok(self.books.lock().unwrap().get(&id.0).cloned())
    }

    async fn find_by_isbn(&self, isbn: &Isbn) -> Result<Option<Book>, RepoError> {
        Ok(self.books.lock().unwrap().values().find(|b| b.isbn.as_str() == isbn.as_str()).cloned())
    }

    async fn save(&self, book: &Book) -> Result<(), RepoError> {
        self.books.lock().unwrap().insert(book.id.0, book.clone());
        Ok(())
    }
}
```

สังเกตว่า `InMemoryBookRepository` ใช้ **`std::sync::Mutex`** (จาก Part 39) ไม่ใช่ `tokio::sync::Mutex` — เพราะ
ทุก critical section ในโค้ดนี้ (`.lock().unwrap()...`) จบลง**ก่อน**ที่จะมี `.await` ใด ๆ เกิดขึ้น (ทวนจาก Part 39/
40: `std::sync::MutexGuard` ห้ามถือข้าม `.await` point เด็ดขาด เพราะมันไม่ implement `Send` — ถ้าถือข้ามจะได้
compile error ทันที ซึ่งเป็นกับดักที่พบบ่อยมากเมื่อผสม synchronous lock เข้ากับ async code ที่จะกล่าวถึงในหัวข้อ
กับดักท้ายบท) การใช้ `std::sync::Mutex` ที่ปลดล็อกเร็ว (แค่ช่วง `HashMap` operation) จึงถูกต้องและเบากว่า
`tokio::sync::Mutex` ที่มีต้นทุนสูงกว่าสำหรับ critical section สั้น ๆ แบบนี้

#### 106.2.5 Payoff: ทำไมการแยก Port/Adapter คุ้มค่าจริง

ตอนนี้เรามี `BookRepository` trait (port) หนึ่งตัว กับสอง adapter (`PgBookRepository`, `InMemoryBookRepository`)
ที่ implement มันทั้งคู่ — core business logic (application service ในหัวข้อ 106.5) เขียนโค้ดครั้งเดียวโดยรับ
`&dyn BookRepository` หรือ `Arc<dyn BookRepository>` เท่านั้น แล้วสลับ adapter เข้า/ออกได้อย่างอิสระ **โดยไม่แก้
โค้ด business logic แม้แต่บรรทัดเดียว**:

| สภาพแวดล้อม | Adapter ที่ใช้ | ความเร็ว | ต้องมี Postgres รันอยู่ไหม |
|---|---|---|---|
| Unit test (Part 95 หัวข้อ 95.3) | `InMemoryBookRepository` | < 1ms ต่อเทส | ไม่ |
| Integration test (Part 95 หัวข้อ 95.4) | `PgBookRepository` ผ่าน `#[sqlx::test]` | ~10-300ms ต่อเทส | ใช่ |
| Production จริง | `PgBookRepository` ผ่าน `PgPoolOptions` (Part 92) | ขึ้นกับ query จริง | ใช่ |
| Demo/พัฒนา local แบบเร็ว (ไม่มี DB ติดตั้ง) | `InMemoryBookRepository` | ทันที | ไม่ |

นี่คือ**เหตุผลเชิงปฏิบัติที่จับต้องได้**ของ hexagonal architecture ไม่ใช่แค่ทฤษฎีสถาปัตยกรรมลอย ๆ: ทีมที่เพิ่ม
ฟีเจอร์ใหม่ในกฎการยืมหนังสือ (เช่น "ห้ามยืมถ้ามีค่าปรับค้างจ่ายเกิน 100 บาท") เขียน unit test ผ่าน
`InMemoryBookRepository` ได้ทันที รันจบในเวลาต่ำกว่ามิลลิวินาที ไม่ต้องรอ CI spin up container Postgres เลย
แล้วค่อยเพิ่ม integration test หนึ่งหรือสองตัวผ่าน `#[sqlx::test]` เพื่อยืนยันว่า `PgBookRepository` ตัวจริง
implement สัญญาเดียวกันถูกต้อง (ตาม testing pyramid Part 95 ที่ควรมี unit test เป็นฐานกว้าง e2e เป็นยอดแคบ)

### 106.3 Domain-Driven Design ในทางปฏิบัติ: Value Object, Entity, Aggregate

Domain-Driven Design (DDD) เป็นแนวทางออกแบบซอฟต์แวร์ที่เสนอโดย Eric Evans (หนังสือ *Domain-Driven Design:
Tackling Complexity in the Heart of Software*, 2003) — แนวคิดหลักคือ **โมเดลในโค้ดต้องสะท้อนภาษาและกฎของธุรกิจ
จริงให้ตรงที่สุด** ไม่ใช่แค่ออกแบบตามความสะดวกของฐานข้อมูลหรือ framework บทนี้จะไม่สอน DDD แบบเต็มรูปแบบ (มันเป็น
หนังสือหนา 500 กว่าหน้า มีศัพท์เฉพาะอีกมาก เช่น Bounded Context, Ubiquitous Language, Domain Event) แต่จะโฟกัส
ที่**สามแนวคิดที่ใช้บ่อยที่สุดและนำไปใช้ได้จริงในโค้ด Rust ทันที**: Value Object, Entity, Aggregate

#### 106.3.1 Value Object: `Isbn`

**Value Object** คือสิ่งที่**ไม่มี identity ของตัวเอง** — สองค่าที่ข้อมูลข้างในเหมือนกันทุกประการ **คือค่าเดียวกัน
เสมอ** ไม่ว่าจะถูกสร้างขึ้นจากที่ไหนหรือเมื่อไหร่ (ต่างจาก Entity ในหัวข้อถัดไปที่ต้องแยกแยะด้วย identity แม้
ข้อมูลอื่นจะเหมือนกัน) ตัวอย่างที่เข้าใจง่ายที่สุด: เงิน 100 บาทสองใบต่างกันทางกายภาพ (คนละใบจริง ๆ) แต่ใน
เชิง "มูลค่า" มันคือ "100 บาท" เหมือนกันเป๊ะ — ไม่มีความหมายที่จะถามว่า "ธนบัตรใบนี้กับใบนั้นคือใบเดียวกันไหม"
ในบริบทของระบบบัญชี (สนใจแค่ยอดเงินรวม)

`Isbn` เป็นตัวอย่าง Value Object ที่ตรงไปตรงมา: หนังสือสองเล่มที่มี ISBN เดียวกันคือ "รุ่นพิมพ์เดียวกัน" เสมอ ไม่
สนใจว่าค่า ISBN นั้นถูกสร้างขึ้นมาจากไหน — และที่สำคัญกว่านั้นตามหลัก **"parse, don't validate"** (ทวนจาก Part 53
หัวข้อ 53.4): `Isbn` ต้องรับประกันว่า **การมีอยู่ของค่าเท่ากับการผ่านการตรวจสอบ ISBN-13 checksum มาแล้วเสมอ** ไม่ใช่
`String` เปล่า ๆ ที่ต้อง validate ซ้ำทุกจุดที่ใช้งาน:

```rust
// src/domain/isbn.rs
use std::fmt;

/// Value Object: `Isbn` การันตีว่า "การมีอยู่ของค่า" เท่ากับ "ผ่านการตรวจสอบ ISBN-13 checksum มาแล้วเสมอ"
/// (parse, don't validate — ตาม Part 53 หัวข้อ 53.4) field เป็น private เข้าถึงได้ผ่าน `.as_str()` เท่านั้น
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct Isbn(String);

#[derive(Debug, Clone, PartialEq, Eq)]
pub struct InvalidIsbn(pub String);

impl fmt::Display for InvalidIsbn {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "ISBN ไม่ถูกต้อง: '{}'", self.0)
    }
}

impl std::error::Error for InvalidIsbn {}

impl Isbn {
    pub fn as_str(&self) -> &str {
        &self.0
    }

    fn checksum_valid(digits: &[u32; 13]) -> bool {
        let sum: u32 = digits
            .iter()
            .enumerate()
            .map(|(i, d)| if i % 2 == 0 { *d } else { d * 3 })
            .sum();
        sum % 10 == 0
    }
}

impl TryFrom<&str> for Isbn {
    type Error = InvalidIsbn;

    fn try_from(value: &str) -> Result<Self, Self::Error> {
        let cleaned: String = value
            .chars()
            .filter(|c| !c.is_whitespace() && *c != '-')
            .collect();

        if cleaned.len() != 13 {
            return Err(InvalidIsbn(value.to_string()));
        }

        let mut digits = [0u32; 13];
        for (i, c) in cleaned.chars().enumerate() {
            match c.to_digit(10) {
                Some(d) => digits[i] = d,
                None => return Err(InvalidIsbn(value.to_string())),
            }
        }

        if !Self::checksum_valid(&digits) {
            return Err(InvalidIsbn(value.to_string()));
        }

        Ok(Isbn(cleaned))
    }
}

impl fmt::Display for Isbn {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", self.0)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn accepts_valid_isbn13() {
        // ISBN จริงของหนังสือ "Clean Code" — checksum ถูกต้องตามสูตร ISBN-13
        let isbn = Isbn::try_from("978-0-13-235088-4").unwrap();
        assert_eq!(isbn.as_str(), "9780132350884");
    }

    #[test]
    fn rejects_wrong_length() {
        assert!(Isbn::try_from("12345").is_err());
    }

    #[test]
    fn rejects_bad_checksum() {
        // เปลี่ยนหลักสุดท้ายให้ checksum ไม่ผ่าน
        assert!(Isbn::try_from("9780132350880").is_err());
    }
}
```

การตรวจสอบ checksum ใช้สูตร ISBN-13 มาตรฐานจริง: บวกเลขทุกหลักโดยสลับตัวคูณ 1 และ 3 ไปเรื่อย ๆ (หลักที่ตำแหน่ง
คู่ (0-indexed) คูณด้วย 1, หลักที่ตำแหน่งคี่คูณด้วย 3) แล้วผลรวมต้องหารด้วย 10 ลงตัว — `#[derive(PartialEq, Eq,
Hash)]` บน `Isbn` (ไม่มี `Ord`/`PartialOrd` เพราะ ISBN ไม่มีความหมายในการ "เรียงลำดับมากกว่า-น้อยกว่า") ทำให้
สอง `Isbn` เท่ากันก็ต่อเมื่อ `String` ข้างในเท่ากัน — **นี่คือธรรมชาติของ Value Object โดยตรง**: ความเท่ากัน
ตัดสินจาก "ข้อมูล" ล้วน ๆ ไม่มี concept ของ "คนละ instance กัน แต่ค่าเหมือนกัน" เข้ามาเกี่ยวข้องเลย ต่างจาก
Entity ในหัวข้อถัดไปที่ต้อง**เปรียบเทียบผ่าน identity เท่านั้น** แม้ field อื่นจะต่างกันไปตามเวลา

#### 106.3.2 Entity: `Book`

**Entity** ตรงข้ามกับ Value Object ตรง ๆ: มันคือสิ่งที่**มี identity ที่คงอยู่ตลอดช่วงชีวิตของมัน แม้ field
อื่นจะเปลี่ยนไปตามเวลา** — หนังสือเล่มหนึ่งใน `Book::id` เดียวกันคือ "หนังสือเล่มเดียวกัน" เสมอ ไม่ว่า
`available_copies` ของมันจะเปลี่ยนจาก 3 เป็น 2 เป็น 1 กี่ครั้งก็ตาม (ต่างจาก Value Object ที่ถ้าข้อมูลเปลี่ยน
คือ "กลายเป็นค่าใหม่" ไปเลย ไม่มี concept ของ "ตัวเดิมที่แค่ค่าเปลี่ยน")

```rust
// src/domain/book.rs
use super::isbn::Isbn;
use std::fmt;

/// Newtype (Part 27/53) ที่ทำหน้าที่เป็น identity ของ Entity `Book`
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct BookId(pub i64);

#[derive(Debug, Clone, PartialEq, Eq)]
pub struct BookInvariantViolation(pub String);

impl fmt::Display for BookInvariantViolation {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", self.0)
    }
}

impl std::error::Error for BookInvariantViolation {}

/// Entity: `Book` มี **identity** (`BookId`) ที่คงอยู่ตลอดแม้ field อื่น (`available_copies`) จะเปลี่ยนไป
/// ตามเวลา — สองค่า `Book` ที่มี `id` เดียวกันคือ "หนังสือเล่มเดียวกัน" เสมอ แม้ field อื่นต่างกัน
/// (ต่างจาก Value Object อย่าง `Isbn` ที่สองค่าเท่ากันถ้าข้อมูลข้างในเท่ากัน ไม่สนใจ identity เลย)
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Book {
    pub id: BookId,
    pub title: String,
    pub author: String,
    pub isbn: Isbn,
    pub total_copies: i32,
    pub available_copies: i32,
}

impl Book {
    pub fn new(
        id: BookId,
        title: impl Into<String>,
        author: impl Into<String>,
        isbn: Isbn,
        total_copies: i32,
    ) -> Result<Self, BookInvariantViolation> {
        if total_copies <= 0 {
            return Err(BookInvariantViolation("total_copies ต้องมากกว่า 0".to_string()));
        }
        Ok(Book {
            id,
            title: title.into(),
            author: author.into(),
            isbn,
            total_copies,
            available_copies: total_copies,
        })
    }

    /// การยืมหนึ่งเล่ม — invariant ที่ `Book` การันตีด้วยตัวเอง: `available_copies` ห้ามติดลบเด็ดขาด
    pub fn borrow_one(&mut self) -> Result<(), BookInvariantViolation> {
        if self.available_copies <= 0 {
            return Err(BookInvariantViolation(format!(
                "หนังสือ '{}' ไม่มีสำเนาว่างให้ยืมในขณะนี้",
                self.title
            )));
        }
        self.available_copies -= 1;
        Ok(())
    }

    /// การคืนหนึ่งเล่ม — invariant อีกข้อ: `available_copies` ห้ามเกิน `total_copies`
    pub fn return_one(&mut self) -> Result<(), BookInvariantViolation> {
        if self.available_copies >= self.total_copies {
            return Err(BookInvariantViolation(format!(
                "หนังสือ '{}' มีสำเนาว่างครบ {} เล่มแล้ว คืนซ้ำไม่ได้",
                self.title, self.total_copies
            )));
        }
        self.available_copies += 1;
        Ok(())
    }
}
```

จุดสำคัญที่ทำให้นี่**เป็น DDD Entity จริง ๆ** ไม่ใช่แค่ struct ธรรมดา: **`Book` การันตี invariant ของตัวเองผ่าน
method `borrow_one()`/`return_one()`** ไม่ปล่อยให้โค้ดข้างนอกแก้ `available_copies` ตรง ๆ แล้วเผลอทำให้มันติดลบ
หรือเกิน `total_copies` ได้ — ทวนจาก Part 53 หัวข้อ 53.4: นี่คือหลักการเดียวกับ "encapsulation/invariant
enforcement" ของ newtype แต่ยกระดับขึ้นมาใช้กับ struct ที่มีหลาย field และมี "พฤติกรรม" (behavior) ของตัวเอง ไม่ใช่
แค่ห่อ primitive type เดียว — **นี่คือความแตกต่างเชิงคุณภาพระหว่าง DTO (data transfer object ที่มีแต่ field ไม่มี
พฤติกรรม เช่น `BookResponse` ของ Part 92 ที่ทำหน้าที่ serialize ออกไปเป็น JSON) กับ Entity ของ DDD (ที่มีทั้ง
ข้อมูลและกฎทางธุรกิจติดอยู่ด้วยกัน)** — โค้ดเบสจริงมักมีทั้งสองอย่างอยู่คนละชั้น: DTO อยู่ที่ขอบของระบบ (HTTP
request/response) ส่วน Entity อยู่ใน core domain logic

#### 106.3.3 Aggregate: `Loan` (แนวคิด "Booking")

**Aggregate** คือกลุ่มของ Entity และ/หรือ Value Object ที่ **ต้องเปลี่ยนแปลงไปด้วยกันเป็นหนึ่งเดียวเสมอ เพื่อ
รักษาความสอดคล้อง (consistency) ของข้อมูล** — คำที่ใช้เรียกขอบเขตนี้คือ **consistency boundary**: ทุกการเปลี่ยน
สถานะที่เกี่ยวข้องกับ aggregate ต้องผ่าน **"aggregate root"** (จุดเข้าเดียวของ aggregate นั้น) เท่านั้น ไม่มีทาง
เข้าไปแก้ field ภายในของสมาชิกอื่นใน aggregate ตรง ๆ จากภายนอกได้เลย

ตัวอย่างในบทนี้คือ **`Loan`** (แนวคิดเดียวกับที่เอกสารออกแบบทั่วไปเรียกว่า **"Booking"** — การจองสิทธิ์ใช้
ทรัพยากรหนึ่งหน่วยในช่วงเวลาหนึ่ง ในบริบทห้องสมุดของเราคือ "การยืมหนึ่งครั้ง"): มันมี invariant ของตัวเองที่**
ไม่เกี่ยวข้องกับ invariant ของ `Book` เลย** — `Book` สนใจว่า `available_copies` ต้องอยู่ในช่วงที่ถูกต้อง ส่วน
`Loan` สนใจว่า "ห้ามคืนซ้ำ" และ "เวลาคืนต้องไม่มาก่อนเวลายืม" — สองก้อน invariant นี้แยกกันโดยเจตนา เพราะเป็น
กฎเชิงธุรกิจคนละเรื่องกัน แม้จะเกี่ยวข้องกันในเชิง workflow:

```rust
// src/domain/loan.rs
use super::book::BookId;
use chrono::{DateTime, Utc};
use std::fmt;

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct LoanId(pub i64);

#[derive(Debug, Clone, PartialEq, Eq)]
enum LoanStatus {
    Active,
    Returned,
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub struct LoanInvariantViolation(pub String);

impl fmt::Display for LoanInvariantViolation {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", self.0)
    }
}

impl std::error::Error for LoanInvariantViolation {}

/// Aggregate: `Loan` เป็น **consistency boundary** ของการยืม-คืนหนึ่งครั้ง: ทุกการเปลี่ยนสถานะของ `Loan`
/// ต้องผ่าน method ของมันเอง ไม่มีทางแก้ field ข้างในตรง ๆ จากนอก module ได้เลย (`status` เป็น private)
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Loan {
    pub id: LoanId,
    pub book_id: BookId,
    pub borrower_name: String,
    pub borrowed_at: DateTime<Utc>,
    pub returned_at: Option<DateTime<Utc>>,
    status: LoanStatus,
}

impl Loan {
    pub fn open(
        id: LoanId,
        book_id: BookId,
        borrower_name: impl Into<String>,
        borrowed_at: DateTime<Utc>,
    ) -> Self {
        Loan {
            id,
            book_id,
            borrower_name: borrower_name.into(),
            borrowed_at,
            returned_at: None,
            status: LoanStatus::Active,
        }
    }

    pub fn is_active(&self) -> bool {
        matches!(self.status, LoanStatus::Active)
    }

    /// การเปลี่ยนสถานะเดียวที่ทำได้กับ `Loan` ที่ active — มี invariant สองข้อที่ต้องผ่านทั้งคู่ก่อนสำเร็จ
    pub fn mark_returned(&mut self, at: DateTime<Utc>) -> Result<(), LoanInvariantViolation> {
        if !self.is_active() {
            return Err(LoanInvariantViolation(format!(
                "Loan #{} ถูกคืนไปแล้ว จะคืนซ้ำอีกไม่ได้",
                self.id.0
            )));
        }
        if at < self.borrowed_at {
            return Err(LoanInvariantViolation(
                "เวลาคืน (returned_at) ต้องไม่มาก่อนเวลายืม (borrowed_at)".to_string(),
            ));
        }
        self.returned_at = Some(at);
        self.status = LoanStatus::Returned;
        Ok(())
    }
}
```

สังเกตว่า `status` เป็น **private field** (ไม่มี `pub`) — ทวนจาก Part 9 (field privacy) และ Part 53 หัวข้อ 53.4
(encapsulation ผ่าน newtype/private field): โค้ดนอก module `loan` **ไม่มีทางตั้ง `status` ให้เป็น `Returned`
ตรง ๆ ได้เลย** ต้องผ่าน `mark_returned()` ที่ตรวจ invariant ทั้งสองข้อก่อนเท่านั้น — นี่คือสิ่งที่ทำให้
"Aggregate เป็น consistency boundary" กลายเป็นความจริงที่ compiler บังคับใช้ได้ ไม่ใช่แค่กฎที่หวังว่าทีมจะทำตาม

**ข้อควรระวังสำคัญของ Aggregate design**: หลักการของ DDD บอกว่า **aggregate ควรมีขนาดเล็กที่สุดที่ยังรักษา
invariant ได้ครบ** — เราจงใจ**ไม่**รวม `Book` เข้าไปเป็นส่วนหนึ่งของ `Loan` aggregate (เช่น ทำให้ `Loan` เก็บ
`Book` เต็มก้อนไว้ข้างในแทนแค่ `BookId`) เพราะ `Book` กับ `Loan` มี **rate of change** และ **invariant ที่เป็น
อิสระจากกัน**: หนังสือเล่มเดียวถูกยืมได้หลายครั้งตลอดชีวิตของมัน (หลาย `Loan` อ้างถึง `Book` เดียวกัน) ถ้ารวม
ทั้งสองเป็น aggregate เดียวจะทำให้ทุกครั้งที่ยืม/คืนต้อง lock ทั้ง `Book` และ ประวัติการยืมทั้งหมดไปด้วย ซึ่งเกิน
ความจำเป็นและทำให้ concurrency แย่ลงมากในระบบที่มีคนยืม-คืนพร้อมกันหลายคน — การอ้างถึงกันด้วย **id เท่านั้น**
(`Loan.book_id: BookId` ไม่ใช่ `Loan.book: Book`) ข้าม aggregate boundary คือแนวทางมาตรฐานของ DDD ที่ควรทำเป็น
default เสมอ

### 106.4 Repository Pattern เจาะลึก: Generic เดียว vs Domain-Specific หลายตัว

หัวข้อ 106.2 แนะนำ `BookRepository` trait ในฐานะ port — คำถามที่ทีมสถาปัตยกรรมจริงต้องเจอเมื่อระบบมี aggregate
หลายสิบตัว (`Book`, `Loan`, `Member`, `Fine`, ...) คือ: **ควรเขียน repository trait แยกกันทีละ aggregate (สิบ
trait) หรือเขียน generic trait ตัวเดียวที่ parameterize ด้วย `T`/`ID` แล้วใช้กับทุก aggregate?**

#### 106.4.1 แนวทางที่ 1: Generic `Repository<T, ID>` เดียว

```rust
// src/repository_generic.rs
use async_trait::async_trait;
use std::hash::Hash;
# #[derive(Debug)] pub struct RepoError;
# impl std::fmt::Display for RepoError { fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result { write!(f, "err") } }
# impl std::error::Error for RepoError {}

/// รูปแบบที่ 1 ของ Repository pattern: **generic `Repository<T, ID>` เดียวใช้กับทุก aggregate**
/// ข้อดี: DRY เขียน trait เดียว ใช้ซ้ำได้กับทุก entity ในระบบ
/// ข้อเสียที่เห็นได้จริงจาก bound ด้านล่าง: ต้องบวก `T: Send + Sync + Clone` และ `ID: Eq + Hash + Send + Sync
/// + Copy` เข้าไปเรื่อย ๆ ทุกครั้งที่ implementation ใหม่ต้องการความสามารถเพิ่ม (เช่น in-memory ต้อง `Clone`
/// เพื่อคืนค่าออกจาก `HashMap` ที่ยืมอยู่, ต้อง `Hash` เพื่อใช้เป็น key) — และที่สำคัญกว่านั้น: มันบอกอะไรไม่ได้
/// เลยเกี่ยวกับ query ที่เฉพาะกับ `Book` เช่น "หาจาก isbn" — ต้อง cast/เพิ่ม method นอก trait อยู่ดี
#[async_trait]
pub trait Repository<T, ID>: Send + Sync
where
    T: Send + Sync + Clone,
    ID: Eq + Hash + Send + Sync + Copy,
{
    async fn find(&self, id: ID) -> Result<Option<T>, RepoError>;
    async fn save(&self, entity: &T) -> Result<(), RepoError>;
    async fn delete(&self, id: ID) -> Result<(), RepoError>;
}
```

`InMemoryBookRepository` เดียวกันจากหัวข้อ 106.2 implement trait ตัวนี้ได้ควบคู่กับ `BookRepository` โดยไม่
ขัดแย้งกัน (เพราะเป็นคนละ trait):

```rust
# use async_trait::async_trait;
# use std::collections::HashMap;
# use std::sync::Mutex;
# #[derive(Debug, Clone)] struct Book { id: BookId }
# #[derive(Clone, Copy, PartialEq, Eq, Hash)] struct BookId(i64);
# #[derive(Debug)] struct RepoError;
# impl std::fmt::Display for RepoError { fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result { write!(f, "err") } }
# impl std::error::Error for RepoError {}
# #[async_trait]
# trait Repository<T, ID>: Send + Sync where T: Send + Sync + Clone, ID: Eq + std::hash::Hash + Send + Sync + Copy {
#     async fn find(&self, id: ID) -> Result<Option<T>, RepoError>;
#     async fn save(&self, entity: &T) -> Result<(), RepoError>;
#     async fn delete(&self, id: ID) -> Result<(), RepoError>;
# }
# struct InMemoryBookRepository { books: Mutex<HashMap<i64, Book>> }
/// implement trait generic ตัวเดียวกันด้วย เพื่อเปรียบเทียบให้เห็นจริง — สังเกตว่า `find`/`save`/`delete`
/// ที่นี่ทำหน้าที่ซ้ำกับ method ของ `BookRepository` (domain-specific) เกือบทั้งหมด แต่ชื่อ method เป็นคำกลาง ๆ
/// ที่ไม่บอกอะไรเกี่ยวกับ "หนังสือ" เลย และยังไม่มีทางเรียก "หาจาก isbn" ผ่าน trait นี้ได้เลยแม้แต่ทางเดียว
#[async_trait]
impl Repository<Book, i64> for InMemoryBookRepository {
    async fn find(&self, id: i64) -> Result<Option<Book>, RepoError> {
        Ok(self.books.lock().unwrap().get(&id).cloned())
    }

    async fn save(&self, entity: &Book) -> Result<(), RepoError> {
        self.books.lock().unwrap().insert(entity.id.0, entity.clone());
        Ok(())
    }

    async fn delete(&self, id: i64) -> Result<(), RepoError> {
        self.books.lock().unwrap().remove(&id);
        Ok(())
    }
}
```

#### 106.4.2 แนวทางที่ 2: Domain-Specific Trait แยกตาม Aggregate

แนวทางที่สองคือ `BookRepository` ที่เราเขียนไว้แล้วในหัวข้อ 106.2 — trait เฉพาะของ `Book` ที่มี method ตรงกับ
สิ่งที่ธุรกิจต้องการจริง (`find_by_isbn`, ไม่ใช่ `find` ที่รับ `ID` แบบกลาง ๆ)

#### 106.4.3 ตารางเปรียบเทียบที่ใช้ตัดสินใจได้จริง

| ประเด็น | Generic `Repository<T, ID>` | Domain-specific (`BookRepository`, `LoanRepository`, ...) |
|---|---|---|
| DRY (ไม่เขียนซ้ำ) | สูงมาก — trait เดียวใช้กับทุก aggregate | ต่ำ — ต้องเขียน trait ใหม่ทุก aggregate (แต่ core CRUD เบา ๆ) |
| Trait bound ที่ต้องบวกเพิ่ม | เพิ่มเรื่อย ๆ ตามความต้องการของ implementation ที่หลากหลาย (`Clone`, `Hash`, ...) ที่ aggregate บางตัวอาจไม่อยากมี (เช่น aggregate ที่มี field ขนาดใหญ่ไม่ควร `Clone` พร่ำเพรื่อ) | ไม่มีปัญหานี้เลย — bound กำหนดเฉพาะเจาะจงตาม aggregate นั้น |
| Query ที่ตรงกับธุรกิจ (เช่น `find_by_isbn`, `find_active_loans_for_member`) | ทำไม่ได้ผ่าน trait นี้เลย ต้องเพิ่ม method แยกนอก trait เสมอ (ทำให้ trait generic "ไม่ได้ช่วยอะไร" ในทางปฏิบัติ) | เขียนตรง ๆ ใน trait ได้เลย ชื่อ method สื่อความหมายทางธุรกิจชัดเจน |
| อ่านโค้ดแล้วเข้าใจ business rule | ต้องเดาจากชื่อ `find`/`save`/`delete` กลาง ๆ | เห็นชื่อ method แล้วเข้าใจ business rule ทันที |
| จำนวน trait ในโค้ดเบส | น้อย (1 ตัว) | มาก (1 ตัวต่อ aggregate) — แต่แต่ละตัวเข้าใจง่ายกว่า |
| เหมาะกับ | CRUD ธรรมดาที่ไม่มี query พิเศษเลย, prototype เร็ว ๆ | โค้ดเบส production ที่ aggregate มี query เฉพาะทางจริง (เกือบทุกกรณีจริง) |

**ข้อสรุปที่ใช้ได้จริงในทางปฏิบัติ**: โค้ดเบส enterprise ส่วนใหญ่**ควรเลือก domain-specific trait เป็นค่า
เริ่มต้น** เพราะในความเป็นจริงแทบไม่มี aggregate ไหนที่ต้องการแค่ `find`/`save`/`delete` ล้วน ๆ โดยไม่มี query
พิเศษเลยตลอดชีวิตของโปรเจกต์ — ทันทีที่ต้องเพิ่ม `find_by_isbn` หนึ่ง method นอก trait generic ก็แทบไม่เหลือ
ประโยชน์ของความเป็น "generic" อีกต่อไป (เพราะ core logic ที่เรียก `find_by_isbn` ก็ต้องรู้จัก concrete type ของ
repository นั้นอยู่ดี ไม่ใช่แค่ `Repository<T, ID>` เพียว ๆ) — generic `Repository<T, ID>` มีที่ใช้จริงในกรณี
แคบ ๆ เช่น เป็น **base trait ที่ domain-specific trait เพิ่มเข้ามา** (เช่น `BookRepository: Repository<Book,
BookId>` แล้วเพิ่ม `find_by_isbn` เข้าไปเฉพาะของมัน) เพื่อแบ่งปัน CRUD พื้นฐานที่ซ้ำกันจริง ๆ ระหว่าง aggregate
แต่ก็ยังต้องมี domain-specific trait เป็นสิ่งที่ core logic เรียกใช้จริงอยู่ดี

### 106.5 Command/Query Separation (CQS) และภาพรวมของ CQRS/Event Sourcing

**Command Query Separation (CQS)** เป็นหลักการที่เสนอโดย Bertrand Meyer: **method ทุกตัวควรเป็นได้แค่อย่างใด
อย่างหนึ่งระหว่าง "Command" (เปลี่ยนสถานะ ไม่คืนข้อมูลที่มีความหมายทางธุรกิจ) กับ "Query" (อ่านสถานะ ไม่มีผล
ข้างเคียงเด็ดขาด) ไม่ผสมกันในตัวเดียว** — บทนี้จะนำหลักการนี้มาใช้ในระดับที่ปฏิบัติได้จริง (ไม่ใช่แค่กฎการตั้งชื่อ
method) โดยสร้าง **type ที่ชัดเจนแทน "คำสั่ง" และ "คำถาม"** แทนการมี service ที่มี method จำนวนมากปนกันแบบ
"grab-bag":

```rust
// src/commands.rs
# use crate::domain::book::{Book, BookId};
# use crate::error::AppError;
# use crate::ports::BookRepository;

/// **Command**: ตั้งใจ "เปลี่ยนสถานะของระบบ" — ไม่คืนข้อมูลที่มีความหมายทางธุรกิจ (คืนแค่ผลลัพธ์/error)
pub struct BorrowBookCommand {
    pub book_id: BookId,
}

pub struct ReturnBookCommand {
    pub book_id: BookId,
}

/// **Query**: ตั้งใจ "อ่านสถานะของระบบ" — ห้ามมีผลข้างเคียง (side effect) เด็ดขาด
pub struct ListAvailableBooksQuery {
    pub title_contains: Option<String>,
}

/// Command handler: ยืมหนังสือหนึ่งเล่ม — ไหลผ่าน port `BookRepository` เท่านั้น ไม่รู้จัก SQL/HashMap เลย
pub async fn handle_borrow_book(
    repo: &dyn BookRepository,
    cmd: BorrowBookCommand,
) -> Result<Book, AppError> {
    let mut book = repo
        .find_by_id(cmd.book_id)
        .await?
        .ok_or_else(|| AppError::NotFound(format!("ไม่พบหนังสือ id {}", cmd.book_id.0)))?;

    book.borrow_one().map_err(|e| AppError::Conflict(e.to_string()))?;

    repo.save(&book).await?;
    Ok(book)
}

pub async fn handle_return_book(
    repo: &dyn BookRepository,
    cmd: ReturnBookCommand,
) -> Result<Book, AppError> {
    let mut book = repo
        .find_by_id(cmd.book_id)
        .await?
        .ok_or_else(|| AppError::NotFound(format!("ไม่พบหนังสือ id {}", cmd.book_id.0)))?;

    book.return_one().map_err(|e| AppError::Conflict(e.to_string()))?;

    repo.save(&book).await?;
    Ok(book)
}

/// Query handler: อ่านล้วน ๆ ไม่แก้ไขอะไรเลย
pub async fn handle_list_available_books(
    repo: &dyn BookRepository,
    query: ListAvailableBooksQuery,
) -> Result<Vec<Book>, AppError> {
    let books = repo.list().await?;
    let filtered = books
        .into_iter()
        .filter(|b| b.available_copies > 0)
        .filter(|b| match &query.title_contains {
            Some(needle) => b.title.to_lowercase().contains(&needle.to_lowercase()),
            None => true,
        })
        .collect();
    Ok(filtered)
}
```

ประโยชน์ที่จับต้องได้ของการแยก type แบบนี้ (เทียบกับการมี `BookService` ตัวเดียวที่มี method `borrow_book()`,
`return_book()`, `list_available_books()`, `search_books()`, ... รวมกันหมด): **เห็น "รูปร่าง" ของ input ที่
ต้องการชัดเจนขึ้นมาก** — `BorrowBookCommand { book_id: BookId }` บอกตรง ๆ ว่าฟังก์ชันนี้ต้องการอะไรบ้าง ต่างจาก
`fn borrow_book(&self, book_id: BookId, user_id: Option<UserId>, notify: bool) -> ...` ที่ยิ่งมี parameter มาก
ยิ่งอ่านยาก (ทวนปัญหาเดียวกับที่ Builder pattern แก้ใน Part 52 หัวข้อ 52.2 — แต่คนละบริบท: ที่นี่คือการเรียก
ฟังก์ชันหนึ่งครั้ง ไม่ใช่การสร้าง object ทีละ field) และที่สำคัญกว่านั้น **Command กับ Query ที่เป็น type คนละ
ตัวทำให้ code review เห็นทันทีว่าฟังก์ชันไหน "มีผลข้างเคียง" โดยไม่ต้องเปิดไปดู implementation** — สิ่งที่มีค่า
มากในโค้ดเบสที่คนอื่นต้องอ่านโค้ดที่ตัวเองไม่ได้เขียน (สถานการณ์ปกติในทีมใหญ่)

#### 106.5.1 CQRS และ Event Sourcing: "เวอร์ชันที่ใหญ่กว่า" ของแนวคิดเดียวกัน

**CQRS (Command Query Responsibility Segregation)** เป็นก้าวต่อไปของ CQS ที่ใหญ่กว่ามาก: ไม่ใช่แค่แยก *type*
ของ command/query แต่แยก **model ทั้งชุดที่ใช้เขียนกับที่ใช้อ่านออกจากกันโดยสิ้นเชิง** — เช่น การเขียน (command
side) ไปที่ PostgreSQL แบบ normalized schema เพื่อรักษาความถูกต้อง ส่วนการอ่าน (query side) ไปที่ read model
ที่ denormalize ไว้แล้วในรูปแบบที่ query เร็วที่สุด (อาจเป็นคนละฐานข้อมูลไปเลย เช่น Elasticsearch สำหรับค้นหา)
โดยมีกระบวนการ sync สอง model นี้ให้ตรงกัน (มักทำผ่าน event)

**Event Sourcing** ไปได้ไกลกว่านั้นอีก: แทนที่จะเก็บ **สถานะปัจจุบัน**ของ aggregate ในฐานข้อมูล (แบบที่
`PgBookRepository` ของเราทำ — `UPDATE books SET available_copies = ...`) ระบบเก็บ **ลำดับ event ทุกตัวที่เคย
เกิดขึ้น** (`BookBorrowed { book_id, at }`, `BookReturned { book_id, at }`, ...) แล้ว**คำนวณสถานะปัจจุบันด้วยการ
"replay" event ทั้งหมดใหม่ทุกครั้ง** (หรือ cache ผลลัพธ์ที่ replay แล้วไว้เป็น "snapshot") — ข้อดีมหาศาลคือมี
**audit trail แบบสมบูรณ์ 100%** (รู้ทุกอย่างที่เคยเกิดขึ้นกับ aggregate นั้นตลอดประวัติ ไม่ใช่แค่สถานะล่าสุด) และ
**สามารถสร้าง read model ใหม่ได้ทุกเมื่อโดย replay event เดิม** (ถ้าพบว่า read model แบบเดิมออกแบบผิด ก็ replay
ใหม่เป็นแบบใหม่ได้โดยไม่เสียข้อมูลอะไรเลย)

**ความซับซ้อนที่แลกมาต้องพูดตรง ๆ**: CQRS+Event Sourcing เพิ่มความซับซ้อนของระบบขึ้นมหาศาลจริง ๆ — ต้องออกแบบ
event schema ที่ไม่มีวันเปลี่ยนความหมายเดิม (event ที่เคย commit แล้วต้อง "อ่านได้ตลอดไป" แม้ business logic
เปลี่ยนไปแล้ว), ต้องมี infrastructure สำหรับ event store, ต้องแก้ปัญหา eventual consistency ระหว่าง command
side กับ query side (ผู้ใช้เขียนข้อมูลไปแล้ว query กลับมาอาจยังไม่เห็นข้อมูลใหม่ทันที ถ้า sync ยังไม่เสร็จ), และ
debug ยากขึ้นมาก (บั๊กอาจซ่อนอยู่ใน logic การ replay event ที่ซับซ้อน ไม่ใช่แค่ query ธรรมดา) — **นี่คือหัวข้อที่
ใหญ่พอจะเป็นบททั้งบทของตัวเองได้เลย** จึงอยู่นอกขอบเขตของ subsection นี้โดยเจตนา สิ่งที่ควรจดจำจากบทนี้คือ**เมื่อ
ไหร่ที่มันคุ้มค่าจริง**: ระบบที่ต้องมี **audit trail ที่เป็นข้อบังคับทางกฎหมาย/ธุรกิจ** (เช่น ระบบธนาคาร, ระบบ
สุขภาพ), ระบบที่ **read pattern กับ write pattern ต่างกันมากจริง ๆ** (เขียนน้อยแต่อ่านหนักมากในรูปแบบที่ต่างกัน
โดยสิ้นเชิง), หรือระบบที่ **ต้องรองรับการ "ย้อนเวลา" ไปดูสถานะ ณ จุดใดจุดหนึ่งในอดีต** — ถ้าระบบไม่มีความต้องการ
เฉพาะเหล่านี้ CQS ระดับ type ธรรมดา (ที่หัวข้อนี้สอน) ให้ผลตอบแทนความชัดเจนสูงมากแล้วโดยไม่ต้องแบกความซับซ้อนของ
CQRS+Event Sourcing เลย

### 106.6 Dependency Injection แบบ Rust: Explicit Passing vs `shaku`

นักพัฒนาที่มาจากโลก Java/C# มักมองหา **"DI framework"** (เช่น Spring ของ Java, ASP.NET Core's built-in DI
container ของ C#) ทันทีที่โค้ดเบส Rust โตขึ้นจนมี dependency หลายชั้น — คำตอบที่ตรงไปตรงมาคือ **Rust ไม่มี DI
framework แบบนั้นเป็น mainstream/idiomatic** และนั่นไม่ใช่ข้อจำกัดที่ต้องแก้ไข แต่เป็น**ผลลัพธ์ธรรมชาติของภาษาที่
มี ownership + trait object ที่ทรงพลังพออยู่แล้ว**

#### 106.6.1 "Poor Man's DI": ส่ง Dependency ผ่าน Constructor ตรง ๆ

ทวนจาก Part 21: `Box<dyn Trait>`/`Arc<dyn Trait>` คือ dynamic dispatch ที่ "ประกาศตอน compile ว่ารับ trait
ไหน แต่เลือก concrete implementation ตอน runtime" — นี่คือกลไกเดียวที่จำเป็นสำหรับ dependency injection ใน
ความหมายที่แท้จริง (สลับ implementation ได้โดยโค้ดที่เรียกใช้ไม่ต้องรู้) และ Rust มีมันอยู่แล้วในภาษาโดยไม่ต้อง
มี framework พิเศษมาช่วย:

```rust
// src/lib.rs
use std::sync::Arc;
# mod ports { #[async_trait::async_trait] pub trait BookRepository: Send + Sync {} }

/// "Poor man's DI": ทุก dependency ถูก "ประกอบ" (wire) ไว้ตรงนี้จุดเดียว ผ่าน constructor ธรรมดา ไม่มี
/// container/macro พิเศษใด ๆ มาช่วย — เปิดไฟล์นี้ไฟล์เดียวก็เห็น "อะไรต่อกับอะไร" ครบทั้งระบบ
#[derive(Clone)]
pub struct AppState {
    pub repo: Arc<dyn ports::BookRepository>,
}

impl AppState {
    pub fn new(repo: Arc<dyn ports::BookRepository>) -> Self {
        AppState { repo }
    }
}
```

การ "wire" dependency จริงเกิดขึ้นที่จุดเดียวในโปรแกรม — ปกติคือ `main()` หรือจุดตั้งต้นของเทส:

```rust
# use std::sync::Arc;
# struct AppState;
# impl AppState { fn new(_r: Arc<dyn std::fmt::Debug>) -> Self { AppState } }
// ตอน production: ประกอบ AppState ด้วย adapter จริง (PgBookRepository)
// let pool = PgPoolOptions::new().connect(&database_url).await?;
// let state = AppState::new(Arc::new(PgBookRepository { pool }));

// ตอนเทส: ประกอบ AppState ตัวเดียวกัน แต่สลับ adapter เป็นของปลอม — โค้ด AppState/handler ไม่รู้ตัวเลย
// let state = AppState::new(Arc::new(InMemoryBookRepository::new(seed_books)));
```

นี่คือทั้งหมดของ "dependency injection" ในความหมายที่ Rust ต้องการ: **ไม่มี reflection, ไม่มี annotation
พิเศษ, ไม่มี container ที่ scan โค้ดตอน runtime เพื่อหาว่าอะไรต้อง inject เข้าตรงไหน** — ทุกอย่างเป็น
**explicit function call ธรรมดาที่ compiler ตรวจสอบ type ให้เต็มรูปแบบตั้งแต่ compile time** ถ้าลืมส่ง
dependency ที่จำเป็น จะได้ compile error ทันที (ขาด argument) ไม่ใช่ runtime panic แบบที่ DI container บางตัว
ทำ (เช่น "ไม่พบ implementation ที่ลงทะเบียนไว้สำหรับ interface นี้" ที่ค้นพบตอนแอปสตาร์ทหรือแย่กว่านั้นคือตอน
เรียก endpoint นั้นครั้งแรก)

#### 106.6.2 เมื่อไหร่ `shaku` (หรือ DI container อื่น) อาจคุ้มค่า

`shaku` เป็น crate ที่ทำ **compile-time DI container** สำหรับ Rust — มันใช้ macro (`#[derive(Component)]`,
`module! { ... }`) เพื่อ "ประกาศ" ว่า component ไหนต้องการ dependency อะไร แล้ว generate โค้ด wiring ให้
อัตโนมัติ คล้ายกับที่ framework ของ Java/C# ทำ แต่ตรวจสอบถูกต้องตอน compile (ไม่ใช่ runtime reflection แบบ
Java) — ข้อดีของมันชัดเจนเมื่อ **จำนวน dependency ในระบบมากจริง ๆ** (หลักสิบ component ที่ต่อกันเป็นกราฟซับซ้อน)
เพราะ constructor แบบ "poor man's DI" ที่ต้องส่ง dependency ทุกตัวด้วยมือจะยาวและซ้ำซ้อนมากขึ้นเรื่อย ๆ

แต่ **ต้นทุนที่ต้องจ่ายจริงและมักถูกมองข้าม**: `shaku` เพิ่ม**ชั้นของ "มองไม่เห็นว่าอะไรต่อกับอะไร"** เข้ามา —
เมื่ออ่านโค้ดที่ประกาศ component ด้วย macro แล้วอยากรู้ว่า "ตัวนี้ได้ dependency ตัวไหนมาจริง ๆ ตอน runtime"
ต้องไล่ macro expansion หรือเปิด `module!` block แยกไปดู ไม่เห็นตรง ๆ ในจุดที่ใช้งานแบบ explicit constructor —
นี่คือ **tradeoff ที่ต้องชั่งใจอย่างตรงไปตรงมา**: โค้ดเบสขนาดกลาง-ใหญ่ส่วนใหญ่ (แม้จะมี dependency สิบ-ยี่สิบ
ตัว) ยังจัดการได้ดีด้วย explicit constructor ธรรมดา (อาจแบ่งเป็นหลาย `AppState`/`*Deps` struct ย่อยตามโมดูล เพื่อ
ไม่ให้ constructor เดียวยาวเกินไป) — `shaku` เหมาะกับสถานการณ์เฉพาะที่ **กราฟ dependency ซับซ้อนมากจริง ๆ และ
ทีมคุ้นเคยกับ pattern DI container จากภาษาอื่นมาก่อนแล้ว** (ทำให้ต้นทุนการเรียนรู้ต่ำกว่าทีมที่ไม่คุ้น) — สำหรับ
โค้ดเบสส่วนใหญ่ **"poor man's DI" ควรเป็นค่าเริ่มต้น** เพราะความ "เห็นได้ตรง ๆ ว่าอะไรต่อกับอะไร" (traceability)
มีค่ามากในการ debug และ onboard คนใหม่เข้าทีม — มากกว่าความสะดวกที่ container มอบให้ในระยะสั้น

### 106.7 Error Handling Architecture ที่สเกลได้: Per-Module Error + `From`

Part 66 สอน `AppError` enum เดียวที่ `impl IntoResponse` ครั้งเดียวให้ทุก error ในระบบตอบ JSON shape เดียวกัน —
Part 92 ต่อยอดให้มันรู้จัก `sqlx::Error` ผ่าน `impl From<sqlx::Error> for AppError` คำถามที่บทนี้ตอบคือ:
**เมื่อโค้ดเบสโตจนมีหลายสิบโมดูล (`domain`, `repository`, `payment`, `notification`, ...) ควรให้ `AppError`
enum เดียวรู้จัก error ของทุกโมดูลโดยตรง หรือควรให้แต่ละโมดูลมี error type ของตัวเอง แล้วรวมเข้า `AppError`
ผ่าน `From`?**

#### 106.7.1 ปัญหาของ "Enum เดียวยักษ์"

ถ้า `AppError` เดียวมี variant สำหรับทุกความเป็นไปได้ในทุกโมดูล (`InvalidIsbn`, `BookNotAvailable`,
`LoanAlreadyReturned`, `PaymentDeclined`, `EmailSendFailed`, ...) จะเกิดปัญหาสามข้อที่ทวีความรุนแรงขึ้นตามขนาด
ของทีม:

1. **ทุกโมดูลต้อง depend on `AppError`** แม้จะเป็นโมดูล domain logic เพียว ๆ ที่ไม่ควรรู้จัก HTTP เลย — ทิศทาง
   dependency ผิดด้านแบบเดียวกับปัญหาที่ hexagonal architecture แก้ในหัวข้อ 106.2 แต่คราวนี้เกิดกับ error type
2. **merge conflict บ่อยมาก** — สอง feature ที่พัฒนาพร้อมกันโดยสองทีม ถ้าทั้งคู่ต้องเพิ่ม variant ใหม่ใน enum
   เดียวกัน มีโอกาสสูงที่จะแก้ไฟล์เดียวกันตรงตำแหน่งใกล้กัน (merge conflict ที่ป้องกันได้ถ้าแยกไฟล์กัน)
3. **`match` ที่ exhaustive ต้องแก้ทุกที่ที่มี** เมื่อเพิ่ม variant ใหม่ (ทวนปัญหา breaking change จาก Part 30
   หัวข้อ 30.7 เรื่อง `#[non_exhaustive]`) — enum ที่มี 50 variant จากสิบโมดูลทำให้จุดที่ต้องแก้กระจายไปทั่ว

#### 106.7.2 แนวทางที่สเกลได้: Error ต่อโมดูล + `From` รวมขึ้นบน

โครงสร้างที่ใช้ในหัวข้อ 106.2-106.5 ของบทนี้คือคำตอบอยู่แล้ว: **`DomainError`** (รวม error จาก `Isbn`, `Book`,
`Loan` — รู้จักแค่กฎทางธุรกิจ) และ **`RepoError`** (รู้จักแค่ "หาไม่พบ" กับ "backend มีปัญหา" ไม่รู้จัก
`sqlx::Error` รั่วออกไปถึงชั้นบน) รวมกันเป็น **`AppError`** ผ่าน `impl From` เท่านั้น — **ทิศทางของ dependency
มีทางเดียว**: จากโมดูลย่อยขึ้นไปที่ `AppError`, ไม่มีทางย้อนกลับ:

```rust
// src/domain/errors.rs
# use std::fmt;
# #[derive(Debug)] pub struct InvalidIsbn(pub String);
# impl fmt::Display for InvalidIsbn { fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result { write!(f, "{}", self.0) } }
# impl std::error::Error for InvalidIsbn {}
# #[derive(Debug)] pub struct BookInvariantViolation(pub String);
# impl fmt::Display for BookInvariantViolation { fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result { write!(f, "{}", self.0) } }
# impl std::error::Error for BookInvariantViolation {}
# #[derive(Debug)] pub struct LoanInvariantViolation(pub String);
# impl fmt::Display for LoanInvariantViolation { fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result { write!(f, "{}", self.0) } }
# impl std::error::Error for LoanInvariantViolation {}

/// Error รวมของ "โมดูล domain" เท่านั้น — ไม่รู้จัก HTTP/SQL/anything ภายนอกเลย
#[derive(Debug)]
pub enum DomainError {
    InvalidIsbn(InvalidIsbn),
    BookInvariant(BookInvariantViolation),
    LoanInvariant(LoanInvariantViolation),
}

impl fmt::Display for DomainError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            DomainError::InvalidIsbn(e) => write!(f, "{e}"),
            DomainError::BookInvariant(e) => write!(f, "{e}"),
            DomainError::LoanInvariant(e) => write!(f, "{e}"),
        }
    }
}

impl std::error::Error for DomainError {}

impl From<InvalidIsbn> for DomainError {
    fn from(e: InvalidIsbn) -> Self { DomainError::InvalidIsbn(e) }
}
impl From<BookInvariantViolation> for DomainError {
    fn from(e: BookInvariantViolation) -> Self { DomainError::BookInvariant(e) }
}
impl From<LoanInvariantViolation> for DomainError {
    fn from(e: LoanInvariantViolation) -> Self { DomainError::LoanInvariant(e) }
}
```

```rust
// src/repo_error.rs
use std::fmt;

/// Error ของ "โมดูล repository/persistence" เท่านั้น — ไม่รู้จัก HTTP เลย รู้จักแค่ "หาไม่พบ" กับ
/// "backend มีปัญหา" (ซ่อนรายละเอียดว่าเป็น Postgres หรือ in-memory ไว้เบื้องหลัง `Backend(String)`)
#[derive(Debug)]
pub enum RepoError {
    NotFound,
    Backend(String),
}

impl fmt::Display for RepoError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            RepoError::NotFound => write!(f, "ไม่พบข้อมูลที่ต้องการในแหล่งข้อมูล"),
            RepoError::Backend(msg) => write!(f, "เกิดข้อผิดพลาดจากแหล่งข้อมูล: {msg}"),
        }
    }
}

impl std::error::Error for RepoError {}

impl From<sqlx::Error> for RepoError {
    fn from(err: sqlx::Error) -> Self {
        match err {
            sqlx::Error::RowNotFound => RepoError::NotFound,
            other => RepoError::Backend(other.to_string()),
        }
    }
}
```

```rust
// src/error.rs
# use std::fmt;
# #[derive(Debug)] pub enum DomainError { InvalidIsbn(String), BookInvariant(String), LoanInvariant(String) }
# impl fmt::Display for DomainError { fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result { write!(f, "domain") } }
# #[derive(Debug)] pub enum RepoError { NotFound, Backend(String) }
# impl fmt::Display for RepoError { fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result { write!(f, "repo") } }

/// Error ระดับ "ทั้งแอป" (ต่อยอดแนวคิด `AppError` จาก Part 66/92) — จุดเดียวที่รู้จัก HTTP status code
/// ทุกโมดูลด้านล่าง (`domain`, `repo_error`) **ไม่รู้จัก** `AppError` เลยแม้แต่นิดเดียว — มีแต่ทิศทางเดียว
/// จากโมดูลย่อยขึ้นมาที่นี่ผ่าน `impl From<...>` เท่านั้น (ทวนจาก Part 30/31: `?` desugar เป็น `From::from`)
#[derive(Debug)]
pub enum AppError {
    NotFound(String),
    Validation(String),
    Conflict(String),
    Internal(anyhow::Error),
}

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AppError::NotFound(m) => write!(f, "not found: {m}"),
            AppError::Validation(m) => write!(f, "validation: {m}"),
            AppError::Conflict(m) => write!(f, "conflict: {m}"),
            AppError::Internal(e) => write!(f, "internal: {e}"),
        }
    }
}

impl std::error::Error for AppError {}

impl From<DomainError> for AppError {
    fn from(err: DomainError) -> Self {
        match err {
            DomainError::InvalidIsbn(e) => AppError::Validation(e.to_string()),
            DomainError::BookInvariant(e) => AppError::Conflict(e.to_string()),
            DomainError::LoanInvariant(e) => AppError::Conflict(e.to_string()),
        }
    }
}

impl From<RepoError> for AppError {
    fn from(err: RepoError) -> Self {
        match err {
            RepoError::NotFound => AppError::NotFound("ไม่พบข้อมูลที่ต้องการ".to_string()),
            RepoError::Backend(msg) => AppError::Internal(anyhow::anyhow!(msg)),
        }
    }
}
```

โครงสร้างนี้ทำให้ `commands.rs` (หัวข้อ 106.5) เขียน `repo.find_by_id(...).await?` ได้ตรง ๆ — `?` desugar เป็น
`From::from(RepoError) -> AppError` อัตโนมัติ (ทวนจาก Part 12/30/31) โดย `commands.rs` **ไม่ต้องรู้จัก
`sqlx::Error` เลยแม้แต่นิดเดียว** — ห่วงโซ่การแปลงเป็นสามชั้นที่แต่ละชั้นรู้จักแค่ชั้นที่อยู่ใกล้ตัวเองที่สุด:
`sqlx::Error -> RepoError -> AppError` (ชั้นบนไม่รู้จักรายละเอียดของชั้นล่างเกินความจำเป็น) เทียบกับถ้าใช้ enum
เดียวยักษ์ที่ `AppError` ต้อง `impl From<sqlx::Error> for AppError` ตรง ๆ — วิธีนั้นทำให้ `AppError`
**depend on sqlx โดยตรง** (crate ระดับ core ของแอปต้อง import `sqlx` แค่เพื่อทำ error conversion) ซึ่งขัดกับ
เจตนาของ hexagonal architecture ในหัวข้อ 106.2 ทันที (core ไม่ควรรู้จัก infrastructure เลย)

**กฎที่ใช้ได้จริงในการตัดสินใจ**: ให้แต่ละโมดูลที่มีขอบเขตความรับผิดชอบชัดเจน (domain logic, persistence,
external API client, ...) มี error type ของตัวเองเสมอ **ยกเว้น** กรณีที่โมดูลนั้นเล็กมากจนไม่มี error ที่ต้อง
แยกแยะจริง (เช่น utility function ล้วน ๆ ที่ error มีแค่แบบเดียว) — จำนวน error type ที่เพิ่มขึ้นไม่ใช่ต้นทุนที่
น่ากลัว (แต่ละตัวเล็กและเข้าใจง่าย) เทียบกับต้นทุนของ enum เดียวที่โตจนไม่มีใครอยากแก้อีกต่อไป

### 106.8 Feature Flags ระดับ Cargo สำหรับโค้ดเบสขนาดใหญ่

ทวนจาก Part 35: `[features]` ใน `Cargo.toml` และ optional dependency (`some-crate = { version = "1", optional
= true }`) คือกลไกพื้นฐานที่มีอยู่แล้ว — หัวข้อนี้ประยุกต์กลไกนั้นกับปัญหาจริงของโค้ดเบส enterprise: **โมดูล
เฉพาะทางบางส่วนของระบบ (เช่น การสร้างรายงาน PDF สำหรับแอดมิน) ดึง dependency ที่ "หนัก" เข้ามา (compile time
นานขึ้น, binary size ใหญ่ขึ้น) ทั้งที่ผู้ใช้ crate ส่วนใหญ่ (เช่น service อื่นที่ import แค่ domain logic ไปใช้)
ไม่เคยต้องการมันเลย**

```toml
# Cargo.toml
[dependencies]
# ... dependency หลักของแอป (tokio, sqlx, serde, ...) ...
csv = { version = "1", optional = true }

[features]
# ในตัวอย่างนี้ใช้ `csv` เป็นตัวแทนของ dependency ที่ "หนัก" กว่านี้มากในโลกจริง เช่น crate สร้าง PDF
# อย่าง `printpdf`/`genpdf` — เลือก `csv` เพื่อให้คอมไพล์ตัวอย่างในบทเรียนได้เร็วและไม่ต้องพึ่ง C toolchain
# ใด ๆ แต่กลไก feature flag ที่สาธิตด้านล่างเหมือนกันทุกประการไม่ว่าจะสลับไปใช้ crate หนักตัวไหนจริง ๆ
admin-reports = ["csv"]
```

```rust
// src/reporting.rs
/// โมดูล admin-only ที่ดึง dependency "หนัก" เข้ามา — ทั้งโมดูลถูก `#[cfg(feature = "admin-reports")]`
/// ครอบไว้ (ทวนจาก Part 35 หัวข้อการครอบทั้ง `mod` แทนครอบทีละฟังก์ชัน — อ่านง่ายกว่าและไม่มีจุดที่ลืมแปะ
/// attribute ซ้ำ) — ผู้ใช้ crate นี้ที่ไม่เปิด feature จะไม่ดึง `csv` (หรือ PDF crate ตัวจริง) เข้ามา
/// คอมไพล์เลยแม้แต่บรรทัดเดียว
#[cfg(feature = "admin-reports")]
pub mod admin_reports {
    use crate::domain::book::Book;

    pub fn export_books_csv(books: &[Book]) -> Result<String, csv::Error> {
        let mut wtr = csv::Writer::from_writer(vec![]);
        wtr.write_record(["id", "title", "author", "isbn", "available_copies"])?;
        for b in books {
            wtr.write_record([
                b.id.0.to_string(),
                b.title.clone(),
                b.author.clone(),
                b.isbn.as_str().to_string(),
                b.available_copies.to_string(),
            ])?;
        }
        let bytes = wtr.into_inner().map_err(|e| e.into_error())?;
        Ok(String::from_utf8(bytes).expect("csv writer เขียนเฉพาะ UTF-8"))
    }
}
```

ยืนยันจริงด้วย `cargo build` สองแบบ (ผลลัพธ์จริงจากการตรวจสอบท้ายบท): **ไม่เปิด feature** — `csv` (หรือ PDF
crate ตัวจริง) ไม่ถูกดึงเข้ามา compile เลยแม้แต่บรรทัดเดียว, `Cargo.lock` ไม่มี `csv` อยู่ **เปิด `--features
admin-reports`** — `csv` ถูก compile เข้ามาเพิ่ม (เห็นบรรทัด `Compiling csv v1.4.0` ในผลลัพธ์จริง) และโมดูล
`admin_reports` ใช้งานได้

**เหตุผลเชิงธุรกิจที่ทำให้สิ่งนี้สำคัญในโค้ดเบสจริง**: ถ้า `library-domain` crate ถูก import โดยหลาย service
(เช่น `web-api`, `background-worker`, `admin-cli`) และมีแค่ `admin-cli` ที่ต้องการสร้างรายงาน PDF — ถ้าไม่แยก
feature flag ทุก service (รวมทั้ง `web-api` ที่ไม่เคยสร้าง PDF เลย) จะต้อง compile PDF library หนัก ๆ ไปด้วย
ทุกครั้งที่ build ทำให้ CI ช้าลง, binary ของ `web-api` ใหญ่ขึ้นโดยไม่จำเป็น, และถ้า PDF library มี dependency
ระบบ (เช่นต้องมี font rendering library ของ OS) ทุก environment ที่ build `web-api` ก็ต้องติดตั้งสิ่งเหล่านั้น
ไปด้วยทั้งที่ไม่เคยใช้เลย — feature flag แก้ปัญหานี้ตรงจุดโดยไม่ต้องแยก crate ออกเป็นหลาย crate (ซึ่งเป็นอีก
ทางเลือกหนึ่งที่ overhead สูงกว่าในการดูแล workspace)

### 106.9 Testing Patterns สำหรับ Enterprise: Test Double Hierarchy และ Property-Based Testing

#### 106.9.1 นิยามที่แม่นยำของ Dummy, Stub, Spy, Mock, Fake

คำห้าคำนี้ถูกใช้ปนกันบ่อยมากในวงการ (คนจำนวนมากเรียกทุกอย่างว่า "mock") แต่มีความหมายต่างกันชัดเจนตามที่ Martin
Fowler และ Gerard Meszaros (ผู้เขียน *xUnit Test Patterns*) ให้นิยามไว้ — ตารางสรุปก่อนดูโค้ดจริง:

| ชนิด | นิยาม | ใช้เมื่อ |
|---|---|---|
| **Dummy** | Object ที่ implement trait ไว้แค่ให้ compile ผ่าน **ไม่ถูกเรียกใช้งานจริงเลย** | signature ต้องการ parameter ที่ path ที่ทดสอบไม่แตะเลย |
| **Stub** | คืนค่า "สำเร็จรูป" ตายตัว ไม่มี logic จริงข้างใน ไม่สนใจ input ที่ส่งมา | ต้องการ input คงที่ให้ path อื่นทำงานต่อได้ ไม่สนใจว่า stub เองทำงานถูกไหม |
| **Fake** | Implementation แบบเบา ๆ ที่ **ทำงานได้จริง** (มี logic จริงข้างใน) แต่ไม่เหมาะกับ production (เช่น ช้าเกินไป/ไม่ persist ข้ามการรัน) | ต้องการพฤติกรรมจริงของ dependency แต่ไม่อยากพึ่ง infrastructure จริง |
| **Spy** | ห่อ implementation จริง (มักเป็น fake) แล้ว **บันทึก** ว่าถูกเรียก method ไหนกี่ครั้งด้วย argument อะไร | ต้องการยืนยัน "พฤติกรรมการเรียก" (interaction) ไม่ใช่แค่ผลลัพธ์สุดท้าย |
| **Mock** | มี **ความคาดหวังที่ตั้งไว้ล่วงหน้า** และมี `.verify()` ที่ fail ถ้าพฤติกรรมจริงไม่ตรงกับที่คาดไว้ | ต้องการยืนยันว่า "ต้องถูกเรียกแบบนี้เท่านั้น" ไม่ใช่แค่บันทึกไว้ให้เช็คทีหลัง |

ทั้งหมดคือ struct ธรรมดาที่ implement `BookRepository` trait เดียวกัน (port จากหัวข้อ 106.2) — ความต่างอยู่ที่
**เจตนาและพฤติกรรมภายใน** ไม่ใช่ syntax:

```rust
// tests/test_doubles.rs
use async_trait::async_trait;
# use crate::domain::book::{Book, BookId};
# use crate::domain::isbn::Isbn;
# use crate::ports::BookRepository;
# use crate::repo_error::RepoError;

// ==================== 1. Dummy: implement trait แค่ให้ compile ผ่าน ไม่ถูกเรียกจริง ====================
struct DummyBookRepository;

#[async_trait]
impl BookRepository for DummyBookRepository {
    async fn list(&self) -> Result<Vec<Book>, RepoError> {
        panic!("DummyBookRepository::list ไม่ควรถูกเรียกเลยในเทสนี้")
    }
    async fn find_by_id(&self, _id: BookId) -> Result<Option<Book>, RepoError> {
        panic!("DummyBookRepository::find_by_id ไม่ควรถูกเรียกเลยในเทสนี้")
    }
    async fn find_by_isbn(&self, _isbn: &Isbn) -> Result<Option<Book>, RepoError> {
        panic!("DummyBookRepository::find_by_isbn ไม่ควรถูกเรียกเลยในเทสนี้")
    }
    async fn save(&self, _book: &Book) -> Result<(), RepoError> {
        panic!("DummyBookRepository::save ไม่ควรถูกเรียกเลยในเทสนี้")
    }
}

// ==================== 2. Stub: คืนค่า "สำเร็จรูป" ตายตัว ไม่มี logic จริงข้างใน ====================
struct StubBookRepository;

#[async_trait]
impl BookRepository for StubBookRepository {
    async fn list(&self) -> Result<Vec<Book>, RepoError> {
        Ok(vec![sample_book(1, 3)])
    }
    async fn find_by_id(&self, _id: BookId) -> Result<Option<Book>, RepoError> {
        Ok(Some(sample_book(1, 3))) // คืนค่าเดียวกันไม่ว่า id ที่ขอจะเป็นอะไร
    }
    async fn find_by_isbn(&self, _isbn: &Isbn) -> Result<Option<Book>, RepoError> {
        Ok(Some(sample_book(1, 3)))
    }
    async fn save(&self, _book: &Book) -> Result<(), RepoError> {
        Ok(())
    }
}
```

**Fake** ต่างจาก stub ตรงที่มี logic การค้นหาจริง (ไม่ใช่ค่าคงที่) — โครงสร้างเดียวกันกับ
`InMemoryBookRepository` ในหัวข้อ 106.2 เขียนขึ้นมาใหม่แยกในไฟล์เทสเพื่อเทียบให้เห็นชัด:

```rust
struct FakeBookRepository {
    books: std::sync::Mutex<std::collections::HashMap<i64, Book>>,
}

#[async_trait]
impl BookRepository for FakeBookRepository {
    async fn list(&self) -> Result<Vec<Book>, RepoError> {
        Ok(self.books.lock().unwrap().values().cloned().collect())
    }
    async fn find_by_id(&self, id: BookId) -> Result<Option<Book>, RepoError> {
        Ok(self.books.lock().unwrap().get(&id.0).cloned())
    }
    async fn find_by_isbn(&self, isbn: &Isbn) -> Result<Option<Book>, RepoError> {
        Ok(self.books.lock().unwrap().values().find(|b| b.isbn.as_str() == isbn.as_str()).cloned())
    }
    async fn save(&self, book: &Book) -> Result<(), RepoError> {
        self.books.lock().unwrap().insert(book.id.0, book.clone());
        Ok(())
    }
}
```

**Spy** ห่อ fake ไว้ข้างในแล้วนับจำนวนครั้งที่ `save()` ถูกเรียก — ใช้ยืนยัน "พฤติกรรมการเรียก" ที่ผลลัพธ์
สุดท้ายอย่างเดียวบอกไม่ได้:

```rust
struct SpyBookRepository {
    inner: FakeBookRepository,
    save_call_count: std::sync::Mutex<u32>,
}

#[async_trait]
impl BookRepository for SpyBookRepository {
    async fn list(&self) -> Result<Vec<Book>, RepoError> { self.inner.list().await }
    async fn find_by_id(&self, id: BookId) -> Result<Option<Book>, RepoError> { self.inner.find_by_id(id).await }
    async fn find_by_isbn(&self, isbn: &Isbn) -> Result<Option<Book>, RepoError> { self.inner.find_by_isbn(isbn).await }
    async fn save(&self, book: &Book) -> Result<(), RepoError> {
        *self.save_call_count.lock().unwrap() += 1;
        self.inner.save(book).await
    }
}

#[tokio::test]
async fn spy_records_that_save_was_called_after_successful_borrow() {
    let spy = SpyBookRepository { /* ... */ };
    // (สร้าง spy จริงพร้อม seed หนังสือ id=1 available=3 — ดูโค้ดเต็มในหัวข้อการตรวจสอบท้ายบท)
    let result = handle_borrow_book(&spy, BorrowBookCommand { book_id: BookId(1) }).await;
    assert!(result.is_ok());
    assert_eq!(spy.save_calls(), 1, "borrow สำเร็จต้องเรียก save() เพื่อบันทึก available_copies ใหม่เสมอ");
}
```

**Mock** มีความคาดหวังตั้งไว้ล่วงหน้าและ `.verify()` ที่ fail ถ้าจำนวนครั้งจริงไม่ตรง — นี่คือความต่างสำคัญจาก
spy: spy แค่ *บันทึก* ให้เทสไปเช็คเอง ส่วน mock *ถูกโปรแกรมมาล่วงหน้า* ว่าคาดหวังอะไร:

```rust
struct MockBookRepository {
    canned_book: Book,
    expected_find_calls: u32,
    actual_find_calls: std::sync::Mutex<u32>,
}

impl MockBookRepository {
    fn verify(&self) -> Result<(), String> {
        let actual = *self.actual_find_calls.lock().unwrap();
        if actual != self.expected_find_calls {
            return Err(format!(
                "คาดหวังว่า find_by_id ถูกเรียก {} ครั้ง แต่เรียกจริง {} ครั้ง",
                self.expected_find_calls, actual
            ));
        }
        Ok(())
    }
}

#[async_trait]
impl BookRepository for MockBookRepository {
    async fn list(&self) -> Result<Vec<Book>, RepoError> {
        unimplemented!("เทสนี้ไม่ได้คาดหวังให้เรียก list()")
    }
    async fn find_by_id(&self, _id: BookId) -> Result<Option<Book>, RepoError> {
        *self.actual_find_calls.lock().unwrap() += 1;
        Ok(Some(self.canned_book.clone()))
    }
    async fn find_by_isbn(&self, _isbn: &Isbn) -> Result<Option<Book>, RepoError> {
        unimplemented!("เทสนี้ไม่ได้คาดหวังให้เรียก find_by_isbn()")
    }
    async fn save(&self, _book: &Book) -> Result<(), RepoError> { Ok(()) }
}
```

ผลรันจริงของทั้ง 6 เทส (dummy, stub, fake, spy, mock ทั้งสองแบบ verify ผ่าน/ไม่ผ่าน) ยืนยันในหัวข้อการตรวจสอบ
ท้ายบทว่าผ่านครบทุกตัว — สิ่งที่ควรจดจำคือ **ทั้งห้าแบบทำงานอยู่บน trait เดียวกันเป๊ะ (`BookRepository`)** ไม่ต้อง
มี framework mocking พิเศษมาช่วยเลย เพราะ Rust's trait system แข็งแรงพอที่จะเขียน test double ทุกระดับด้วยมือได้
ตรงไปตรงมา (ต่างจากบางภาษาที่ต้องพึ่ง mocking framework ที่ทำ runtime bytecode manipulation)

#### 106.9.2 Property-Based Testing ด้วย `proptest`: ยืนยัน Invariant ด้วยการสุ่มจำนวนมาก

เทสทั้งหมดในหัวข้อ 106.9.1 เป็น **example-based test**: เขียนกรณีทดสอบทีละกรณีด้วยมือ (input นี้ ควรได้ output
นั้น) — วิธีนี้ดีมากสำหรับพิสูจน์ "กรณีที่คิดถึง" แต่**พลาดกรณีที่ไม่ได้คิดถึงได้ง่าย** โดยเฉพาะ invariant ที่ต้อง
เป็นจริง "ไม่ว่าจะเรียกลำดับไหนก็ตาม" ซึ่งมนุษย์ไล่คิดเองครบทุกกรณีได้ยาก — **Property-Based Testing** (ด้วย
crate `proptest` หรือ `quickcheck`) แก้ปัญหานี้: เขียน **property** (คุณสมบัติที่ต้องเป็นจริงเสมอ) หนึ่งครั้ง
แล้วให้ library **สุ่มสร้าง input จำนวนมาก** (ปกติหลายร้อยเคส) มาทดสอบ property นั้น — ถ้าเจอ input ที่ทำให้
property ล้มเหลว มันจะ**"shrink"** (ลดรูป) input นั้นให้เล็กที่สุดที่ยังทำให้ล้มเหลวอยู่ ก่อนรายงานกลับมา ทำให้
debug ง่ายกว่าเห็น input สุ่มที่ซับซ้อนดิบ ๆ มาก

ในบริบท enterprise ที่ invariant ทางธุรกิจต้องคงอยู่เสมอ (เช่น "จำนวนสำเนาที่ว่างต้องไม่ติดลบและไม่เกินจำนวน
ทั้งหมด ไม่ว่าจะมีคนยืม-คืนสลับกันกี่ครั้งก็ตาม") property-based testing มีค่ามากเป็นพิเศษ — เขียน property เดียว
ทดสอบ `Book::borrow_one()`/`return_one()` จากหัวข้อ 106.3.2:

```rust
// tests/proptest_invariant.rs
use proptest::prelude::*;
# use crate::domain::book::{Book, BookId};
# use crate::domain::isbn::Isbn;

#[derive(Debug, Clone, Copy)]
enum Op { Borrow, Return }

fn sample_book(total: i32) -> Book {
    Book::new(BookId(1), "Domain-Driven Design", "Eric Evans",
        Isbn::try_from("9780132350884").unwrap(), total).unwrap()
}

proptest! {
    /// Invariant ที่ต้องเป็นจริงตลอดเวลาไม่ว่าจะสุ่มเรียก borrow_one()/return_one() ลำดับไหนก็ตาม:
    /// 0 <= available_copies <= total_copies เสมอ — เขียนเป็น property เดียว แทนต้องนั่งไล่คิด
    /// กรณี edge case ทีละเคสด้วยมือ (ยืมจนหมดแล้วคืนซ้ำ, คืนก่อนยืม, สุ่มสลับกันรัว ๆ ฯลฯ)
    #[test]
    fn available_copies_never_leaves_valid_range(
        total in 1i32..20,
        ops in prop::collection::vec(prop_oneof![Just(Op::Borrow), Just(Op::Return)], 0..200)
    ) {
        let mut book = sample_book(total);
        for op in ops {
            match op {
                Op::Borrow => { let _ = book.borrow_one(); }
                Op::Return => { let _ = book.return_one(); }
            }
            // ไม่ว่าผลลัพธ์ของแต่ละ op จะเป็น Ok หรือ Err (ถูกปฏิเสธเพราะ invariant) ค่าที่เหลืออยู่
            // ต้องอยู่ในช่วงที่ถูกต้องเสมอ — ถ้า Book ปล่อยให้หลุดช่วงได้แม้แต่ครั้งเดียว test จะ fail
            prop_assert!(book.available_copies >= 0);
            prop_assert!(book.available_copies <= book.total_copies);
        }
    }

    /// Invariant ของ Isbn value object: ทุกสตริงที่ Isbn::try_from ยอมรับ ต้องมี checksum ผ่านจริง
    /// (ตรวจสอบซ้ำด้วยสูตรเดียวกันแบบ "reference implementation" แยกจากโค้ดจริงในไฟล์ isbn.rs)
    #[test]
    fn accepted_isbn_always_has_valid_checksum(digits in prop::collection::vec(0u32..10, 13..14)) {
        let candidate: String = digits.iter().map(|d| d.to_string()).collect();
        let sum: u32 = digits.iter().enumerate()
            .map(|(i, d)| if i % 2 == 0 { *d } else { d * 3 })
            .sum();
        let reference_says_valid = sum % 10 == 0;

        let parsed = Isbn::try_from(candidate.as_str());
        prop_assert_eq!(parsed.is_ok(), reference_says_valid);
    }
}
```

ผลรันจริง (จากการตรวจสอบท้ายบท): ทั้งสอง property ผ่าน**256 เคสสุ่ม**ต่อ property (ค่า default ของ `proptest`)
โดยแต่ละเคสของ `available_copies_never_leaves_valid_range` สุ่มทั้งจำนวนสำเนาเริ่มต้น (1-19) และลำดับ operation
สูงสุด 200 ครั้ง — เทียบกับการเขียน example-based test ให้ครอบคลุมเท่ากัน จะต้องเขียนกรณีทดสอบด้วยมือหลายสิบเคส
และยังมีโอกาสพลาดลำดับที่ไม่ได้คิดถึง — นี่คือเหตุผลที่ property-based testing ถูกจัดเป็น pattern การทดสอบที่มี
ค่าเฉพาะกับโค้ดเบส enterprise ที่ invariant ต้องถูกรักษาอย่างเข้มงวด (ระบบการเงิน, ระบบจอง/ยืมทรัพยากรที่มีจำนวน
จำกัด, โค้ดที่ทำ state transition ซับซ้อน) มากกว่าจะเป็นแค่ "เทคนิคทดสอบทั่วไปที่ควรใช้ทุกที่" — property-based
test เขียนยากกว่า example-based test เล็กน้อย (ต้องคิดว่า "property อะไรที่เป็นจริงเสมอ" ซึ่งบางครั้งตอบยากกว่า
"input นี้ควรได้ output อะไร") จึงเหมาะเป็นส่วนเสริมของ testing pyramid ไม่ใช่ตัวแทนทั้งหมด

### 106.10 Capstone: ประกอบทุก Pattern เข้าด้วยกันบนสไลซ์ของ Part 92-94

ตอนนี้ทุกชิ้นส่วนที่หัวข้อ 106.2-106.9 สร้างไว้ **คือโค้ดชุดเดียวกันที่ประกอบเป็นสไลซ์จริงของแอปห้องสมุด** ไม่ใช่
ตัวอย่างแยกกันคนละเรื่อง — มาดูภาพรวมว่าทุกอย่างต่อกันอย่างไร โดยเทียบกับโครงสร้างเดิมของ Part 92:

```
โครงสร้างเดิม (Part 92)                     โครงสร้างใหม่ (บทนี้ — hexagonal + DDD)
─────────────────────────                   ──────────────────────────────────────
handlers.rs                                  domain/
  └─ borrow_book() เรียก sqlx::query!           ├─ isbn.rs      (Value Object: Isbn)
     ตรงๆ กลางฟังก์ชัน                          ├─ book.rs      (Entity: Book + invariant)
                                                 ├─ loan.rs      (Aggregate: Loan/Booking)
error.rs                                        └─ errors.rs    (DomainError + From)
  └─ AppError enum เดียว รู้จัก
     sqlx::Error ตรงๆ                        ports.rs          (BookRepository trait — "port")
                                              repo_error.rs     (RepoError — ไม่รู้จัก HTTP)
                                              adapters/
                                                ├─ postgres.rs  (PgBookRepository — "adapter" จริง)
                                                └─ in_memory.rs (InMemoryBookRepository — "adapter" ปลอม)
                                              commands.rs       (Command/Query แยกชัดเจน — CQS)
                                              error.rs          (AppError รับจาก DomainError/RepoError ผ่าน From)
                                              lib.rs            (AppState — poor man's DI)
```

**สิ่งที่พิสูจน์ได้จริงและวัดผลได้จากโครงสร้างใหม่** (ยืนยันด้วยผลรันจริงในหัวข้อการตรวจสอบท้ายบท):

1. **หัวข้อ 106.2 (Hexagonal)**: `PgBookRepository` และ `InMemoryBookRepository` implement `BookRepository`
   trait เดียวกัน compile ผ่านทั้งคู่ — `commands.rs` เขียนครั้งเดียว ใช้ได้กับทั้งสอง adapter โดยไม่ต้องแก้
2. **หัวข้อ 106.3 (DDD)**: `Isbn::try_from("9780132350884")` ผ่าน (ISBN จริงของหนังสือ "Clean Code") และ
   `Isbn::try_from("9780132350880")` (checksum ผิด) ถูกปฏิเสธ — `Book::borrow_one()` ปฏิเสธการยืมเมื่อ
   `available_copies` เป็น 0 — ทั้งสอง invariant ถูกการันตีโดย type เอง ไม่ใช่แค่ comment บอกให้ระวัง
3. **หัวข้อ 106.7 (Error architecture)**: `commands::handle_borrow_book` เขียน `repo.find_by_id(...).await?`
   ตรง ๆ ได้ เพราะ `RepoError` แปลงเป็น `AppError` อัตโนมัติผ่าน `From` — `commands.rs` ไม่มีบรรทัดไหน import
   `sqlx` เลยแม้แต่บรรทัดเดียว (ตรวจสอบได้ด้วย `grep -r sqlx src/commands.rs` ที่ไม่พบผลลัพธ์อะไรเลย)
4. **การสลับ adapter จริงในเทส**: เทส `spy_records_that_save_was_called_after_successful_borrow` (หัวข้อ
   106.9) เรียก `commands::handle_borrow_book` เดียวกันกับที่ production ใช้ แต่ส่ง `SpyBookRepository`
   (ห่อ fake) เข้าไปแทน `PgBookRepository` — ฟังก์ชัน `handle_borrow_book` ไม่มีทางรู้เลยว่าตัวเองถูกเรียกจาก
   เทสหรือจาก production จริง เพราะรับพารามิเตอร์เป็น `&dyn BookRepository` เท่านั้น

นี่คือ **payoff ที่แท้จริงของทุก pattern ในบทนี้รวมกัน**: ไม่ใช่ว่าแต่ละ pattern มีประโยชน์แยกกันเป็นชิ้น ๆ แต่
เมื่อประกอบเข้าด้วยกัน (hexagonal architecture กำหนด "ที่ที่ dependency ควรมองไปทางไหน", DDD กำหนด "โมเดล
ข้อมูลที่ตรงกับธุรกิจและการันตี invariant ของตัวเอง", error architecture กำหนด "ข้อมูลไหลขึ้นบนอย่างไรโดยไม่รั่ว
รายละเอียดเกินจำเป็น", test double hierarchy กำหนด "จะพิสูจน์ความถูกต้องอย่างไรโดยไม่ต้องรอ infrastructure จริง")
— โค้ดเบสทั้งระบบกลายเป็นสิ่งที่**เปลี่ยนแปลงได้อย่างปลอดภัย แม้จะมีคนหลายสิบคนแก้ไขพร้อมกันตลอดหลายปี** ซึ่งเป็น
คำถามตั้งต้นของบทนี้ในหัวข้อ 106.1

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. ลืมแปะ `#[async_trait]` บน `impl` block (แปะไว้แค่บน `trait` definition)

`async_trait` macro ต้องแปะทั้งบน **trait definition** และ **ทุก `impl` block** ของ trait นั้น เพราะมันต้อง
desugar ทั้งสองฝั่งให้ signature ตรงกัน (ทั้งคู่กลายเป็น method ที่คืน `Pin<Box<dyn Future<...>>>` เหมือนกัน) —
ถ้าลืมแปะบน `impl` จะได้ compile error จริงแบบนี้ (ทดสอบจริงจากการลบ `#[async_trait]` ออกจาก `impl
BookRepository for InMemoryBookRepository`):

```
error[E0195]: lifetime parameters or bounds on method `list` do not match the trait declaration
  --> src/adapters/in_memory.rs:28:18
   |
28 |     async fn list(&self) -> Result<Vec<Book>, RepoError> {
   |                  ^ lifetimes do not match method in trait
   |
  ::: src/ports.rs:9:1
   |
 9 | #[async_trait]
   | -------------- this bound might be missing in the impl
10 | pub trait BookRepository: Send + Sync {
11 |     async fn list(&self) -> Result<Vec<Book>, RepoError>;
   |              -----------
   |              |    |
   |              |    this bound might be missing in the impl
   |              lifetimes in impl do not match this method in trait
```

**วิธีแก้**: แปะ `#[async_trait]` บน `impl` block เสมอ ไม่ว่าจะแปะบน trait definition แล้วก็ตาม — ทั้งสองจุดต้อง
มาคู่กันเสมอ ลองนึกเป็นกฎง่าย ๆ: "เห็น `async fn` ใน `impl` ของ trait ที่มี `#[async_trait]` บน trait — ต้องมี
`#[async_trait]` บน `impl` block นั้นด้วยเสมอ ไม่มีข้อยกเว้น"

### 2. ลืม `impl From<SubError> for AppError` แล้วใช้ `?` ตรง ๆ

เมื่อเพิ่มโมดูลย่อยใหม่ที่มี error type ของตัวเอง (ตามแนวทางหัวข้อ 106.7) แล้วลืมเขียน `impl From<...>` ให้
ครบ — `?` operator จะ error ทันทีตอน compile (ทดสอบจริงจากการลบ `impl From<RepoError> for AppError` ออก):

```
error[E0277]: `?` couldn't convert the error to `AppError`
  --> src/commands.rs:26:15
   |
24 |       let mut book = repo
   |  ____________________-
25 | |         .find_by_id(cmd.book_id)
26 | |         .await?
   | |              -^ the trait `From<RepoError>` is not implemented for `AppError`
   | |______________|
   |                this can't be annotated with `?` because it has type `Result<_, RepoError>`
   |
note: `AppError` needs to implement `From<RepoError>`
```

**วิธีแก้**: ทุกครั้งที่เพิ่ม error type ย่อยใหม่ในระบบ (ตามสถาปัตยกรรมหัวข้อ 106.7) ต้องเขียน `impl
From<NewErrorType> for AppError` คู่กันเสมอ — error message ของ compiler บอกตรง ๆ อยู่แล้วว่า "ต้อง implement
`From` ตัวไหน" ทำให้แก้ได้ทันทีโดยไม่ต้องเดา แต่ถ้าลืมจริง ๆ นี่คือสัญญาณที่ดีว่าสถาปัตยกรรม error ของระบบทำงาน
ถูกต้อง (compiler จับปัญหาให้ตั้งแต่ compile time ไม่ใช่รั่วไปถึง production แบบไม่รู้ตัว)

### 3. ส่ง `LoanId`/`BookId` สลับตำแหน่งกัน (Newtype ป้องกันได้ แต่ต้องมี Newtype จริง)

แม้จะใช้ newtype (`BookId`, `LoanId` — ทวนจาก Part 27/53) ป้องกันการสลับ `i64` เปล่า ๆ ไปแล้ว การสลับ **ระหว่าง
สอง newtype ที่หน้าตาคล้ายกัน** ยังเกิดขึ้นได้ถ้าโค้ดสองจุดใช้ตำแหน่ง argument คล้ายกันเกินไป — ข้อดีคือ
compiler จับได้ทันที ไม่ต้องรอ runtime bug (ทดสอบจริงด้วยการส่ง `LoanId` เข้าไปในตำแหน่งที่ `BorrowBookCommand`
ต้องการ `BookId`):

```
error[E0308]: mismatched types
  --> src/bin/broken_e0308.rs:19:68
   |
19 |     let _ = handle_borrow_book(&repo, BorrowBookCommand { book_id: loan_id }).await;
   |                                                                    ^^^^^^^ expected `BookId`, found `LoanId`
```

**วิธีแก้**: ไม่มีอะไรต้อง "แก้" ในความหมายของการเปลี่ยนดีไซน์เลย — นี่คือกับดักที่ยกมาเพื่อ**ย้ำความสำคัญ**ของ
newtype pattern จาก Part 53: ถ้าใช้ `i64` เปล่า ๆ ทั้ง `book_id` และ `loan_id` บั๊กนี้จะ compile ผ่านสมบูรณ์แล้ว
ไปพังตอน runtime แทน — สิ่งที่ต้องระวังจริง ๆ คือ**อย่าลดระดับกลับไปใช้ primitive type เปล่า ๆ เพื่อความสะดวก
ชั่วคราว** เมื่อระบบมี ID หลายชนิดที่ underlying type เดียวกัน

### 4. เปิด Feature Module โดยไม่ครอบ `#[cfg(feature = "...")]` ให้ครบ

ถ้าลืมแปะ `#[cfg(feature = "admin-reports")]` บนโมดูลที่ใช้ optional dependency แต่ยังปล่อยให้โค้ดข้างในเรียก
`csv::...` ตรง ๆ — พอ build แบบไม่เปิด feature (ค่า default) จะพังทันทีเพราะ dependency นั้นไม่ถูกดึงเข้ามาเลย
(ทดสอบจริงด้วยการลบ `#[cfg(feature = "admin-reports")]` ออกจาก `pub mod admin_reports`):

```
error[E0433]: cannot find module or crate `csv` in this scope
  --> src/reporting.rs:10:23
   |
10 |         let mut wtr = csv::Writer::from_writer(vec![]);
   |                       ^^^ use of unresolved module or unlinked crate `csv`
   |
   = help: if you wanted to use a crate named `csv`, use `cargo add csv` to add it to your `Cargo.toml`
```

**วิธีแก้**: ครอบทั้ง `mod` ด้วย `#[cfg(feature = "...")]` แทนครอบทีละฟังก์ชัน (ทวนจาก Part 35) เพื่อไม่ให้มีจุด
ที่ลืมแปะซ้ำ และ**รัน `cargo build` (ไม่เปิด feature ใด ๆ) เป็นส่วนหนึ่งของ CI เสมอ** ไม่ใช่รันแค่ `cargo build
--all-features` — ถ้า CI รันแต่ `--all-features` จะไม่มีวันเจอบั๊กแบบนี้เลยเพราะ feature ถูกเปิดตลอด ทั้งที่
ผู้ใช้ crate จริงจำนวนมากจะ build แบบไม่เปิด feature พิเศษเหล่านี้

### 5. ลืม `Send + Sync` Bound บน Port Trait ที่จะใช้เป็น `Arc<dyn Trait>` ข้าม Thread

Port trait ที่จะถูกเก็บใน `AppState` และแชร์ข้าม request/thread (ผ่าน Axum/tokio) **ต้องมี `Send + Sync` bound**
เสมอ — ถ้าลืม จะไม่ error ตอนนิยาม trait หรือตอน implement เลย (ทุกอย่าง compile ผ่านปกติ) แต่จะ error ทันทีที่
เอาไปใช้จริงในจุดที่ต้องข้าม thread เช่น `tokio::spawn` (ทดสอบจริงด้วยการลบ `Send + Sync` ออกจาก `pub trait
BookRepository: Send + Sync`):

```
error[E0277]: `dyn BookRepository` cannot be shared between threads safely
   --> src/bin/broken_send.rs:9:5
    |
  9 | /     tokio::spawn(async move {
 10 | |         let _ = state.repo.list().await;
 11 | |     });
    | |______^ `dyn BookRepository` cannot be shared between threads safely
    |
    = help: the trait `Sync` is not implemented for `dyn BookRepository`
    = note: required for `Arc<dyn BookRepository>` to implement `Send`
```

**วิธีแก้**: ทุก port trait ที่จะใช้เป็น `Arc<dyn Trait>` ใน `AppState` ของแอป async ควรมี `: Send + Sync` เป็น
super trait bound **ตั้งแต่วันแรกที่นิยาม trait** ไม่ใช่รอให้ compiler ฟ้องตอนหลัง — สังเกตด้วยว่า error message
นี้ปรากฏที่**จุดใช้งาน** (`tokio::spawn`) ไม่ใช่ที่จุดนิยาม trait ทำให้บางครั้งดูเหมือนไม่เกี่ยวกัน ถ้าไม่คุ้นกับ
รูปแบบ error นี้มาก่อนอาจเสียเวลาไล่หาสาเหตุนาน — จำ pattern นี้ไว้: เห็น `` `dyn SomeTrait` cannot be sent/shared
between threads safely `` ให้สงสัย super trait bound ของ `SomeTrait` เป็นอันดับแรกเสมอ

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เพิ่ม method `count_by_author(&self, author: &str) -> Result<usize, RepoError>` เข้าไปใน
   `BookRepository` trait แล้ว implement ให้ครบทั้ง `InMemoryBookRepository` และ `PgBookRepository` (สำหรับ
   `PgBookRepository` ใช้ `SELECT COUNT(*) FROM books WHERE author = $1`) — เขียน unit test ผ่าน
   `InMemoryBookRepository` ที่ seed หนังสือสามเล่ม (ผู้เขียนคนละคนสองเล่ม คนเดียวกันหนึ่งเล่ม) แล้วยืนยันว่า
   `count_by_author` คืนค่าถูกต้อง
   - Hint: เพิ่ม method ใน trait แล้ว compiler จะฟ้องทุก `impl` ที่ยังไม่มี method นี้ให้ตามแก้ทีละที่

2. **(กลาง)** สร้าง Value Object ใหม่ชื่อ `MemberId(String)` ที่การันตีว่ารูปแบบเป็น `"M-"` ตามด้วยเลข 6 หลักเสมอ
   (เช่น `"M-000123"`) ผ่าน `TryFrom<&str>` ตามแนวทาง `Isbn` ในหัวข้อ 106.3.1 จากนั้นสร้าง Entity `Member { id:
   MemberId, name: String, active_loans: u32, max_loans: u32 }` ที่มี method `can_borrow(&self) -> bool` (คืน
   `true` ถ้า `active_loans < max_loans`) และ `record_new_loan(&mut self) -> Result<(), ...>` ที่ปฏิเสธถ้า
   `!self.can_borrow()` — เขียน property-based test (`proptest`) ยืนยันว่า `active_loans` ไม่มีวันเกิน
   `max_loans` ไม่ว่าจะสุ่มเรียก `record_new_loan()` กี่ครั้งก็ตาม
   - Hint: โครงสร้าง property test เหมือนกับ `available_copies_never_leaves_valid_range` ในหัวข้อ 106.9.2
     เกือบทุกจุด แค่เปลี่ยน invariant ที่ตรวจ

3. **(ยาก)** เพิ่ม `LoanRepository` trait (port ใหม่) ที่มี `find_active_by_book(&self, book_id: BookId) ->
   Result<Option<Loan>, RepoError>` และ `save(&self, loan: &Loan) -> Result<(), RepoError>` แล้วเขียน
   application service `handle_return_book_v2` ใหม่ที่ **ใช้ทั้ง `BookRepository` และ `LoanRepository`
   ร่วมกัน**: หา `Loan` ที่ active ของหนังสือเล่มนั้น เรียก `loan.mark_returned(now)`, เรียก
   `book.return_one()`, แล้ว save ทั้งสอง aggregate ผ่านคนละ repository — implement `InMemoryLoanRepository`
   สำหรับเทส แล้วเขียน integration-style test (ยังเป็น unit test เพราะใช้ fake ทั้งคู่) ที่ยืนยันว่าทั้งสอง
   aggregate ถูกอัปเดตถูกต้องพร้อมกัน
   - Hint: นี่คือจุดที่ควรสังเกตว่า "consistency ข้าม aggregate" ต้องจัดการที่ระดับ application service ไม่ใช่
     ผลักภาระให้ aggregate ใดฝ่ายเดียว (ทวนหัวข้อ 106.3.3 เรื่องขอบเขตของ aggregate)

4. **(ยาก/ประยุกต์จริง)** ออกแบบและ implement feature flag `"analytics"` ใหม่ที่เพิ่มโมดูล
   `analytics::compute_borrow_frequency(loans: &[Loan]) -> HashMap<BookId, u32>` (นับว่าหนังสือแต่ละเล่มถูกยืม
   กี่ครั้งจาก `Vec<Loan>`) — เขียนให้ `cargo build` (ไม่เปิด feature) และ `cargo build --features analytics`
   (เปิด feature) ผ่านทั้งคู่ แล้วเขียน mock ที่ตั้งความคาดหวังไว้ล่วงหน้าว่า `LoanRepository::list_all()` ต้อง
   ถูกเรียกพอดี 1 ครั้ง (ตามแนวทาง `MockBookRepository` ในหัวข้อ 106.9.1) สำหรับฟังก์ชัน service ที่ดึง loan
   ทั้งหมดมาคำนวณ analytics แล้วยืนยันด้วย `.verify()` ว่าผ่าน
   - Hint: `analytics` ไม่ควร depend on `sqlx`/`InMemoryBookRepository` เลย — มันควรรับ `&[Loan]` ธรรมดาเข้ามา
     (pure function) แล้วให้ caller เป็นผู้ดึงข้อมูลจาก repository ก่อน — นี่คือการประยุกต์ CQS จากหัวข้อ 106.5
     ที่แยก "การอ่านข้อมูล" ออกจาก "การคำนวณจากข้อมูลที่อ่านมาแล้ว" อีกชั้นหนึ่ง

## สรุป

บทนี้ขยับจาก **pattern ระดับ object เดี่ยว ๆ** (Part 52-53) ไปสู่ **pattern ระดับสถาปัตยกรรมของทั้งแอปพลิเคชัน**
ที่จำเป็นเมื่อโค้ดเบสมีผู้ร่วมเขียนหลายคนและอายุยืนหลายปี: **Hexagonal Architecture** พลิกทิศทาง dependency ให้
core business logic depend on trait ("port") เท่านั้น ทำให้สลับ infrastructure จริง (`PgBookRepository`) กับ
ของปลอมสำหรับเทส (`InMemoryBookRepository`) ได้โดยไม่แก้ core logic แม้แต่บรรทัดเดียว — **DDD** (Value Object,
Entity, Aggregate) ให้โมเดลข้อมูลที่การันตี invariant ทางธุรกิจด้วยตัวเอง ผ่าน type system ไม่ใช่แค่ comment —
**Repository pattern** เจาะลึกให้เห็น tradeoff จริงระหว่าง generic ตัวเดียวกับ domain-specific หลายตัว — **CQS**
แยก command จาก query ให้ชัดในระดับ type พร้อมรู้ขอบเขตว่า CQRS+Event Sourcing คือเวอร์ชันที่ใหญ่กว่าซึ่งมีต้นทุน
สูงกว่ามาก — **DI แบบ Rust** ยืนยันว่า explicit constructor passing เพียงพอสำหรับโค้ดเบสส่วนใหญ่ โดยไม่ต้องพึ่ง
DI container ที่แลกความชัดเจนไป — **Error architecture** ให้ error ต่อโมดูลรวมกันผ่าน `From` แทน enum เดียวยักษ์
— **Feature flags** แยก dependency หนักออกจาก consumer ที่ไม่ต้องการมัน — และ **test double hierarchy +
property-based testing** ให้คำศัพท์และเครื่องมือที่แม่นยำสำหรับพิสูจน์ความถูกต้องของระบบทั้งหมดนี้อย่างรวดเร็ว
โดยไม่ต้องพึ่ง infrastructure จริงเสมอไป

ทุก pattern ในบทนี้ถูกประกอบเป็นสไลซ์เดียวกันของแอปห้องสมุดจาก Part 92-94 ในหัวข้อ 106.10 พิสูจน์ให้เห็นว่า
pattern เหล่านี้**ไม่ใช่ทฤษฎีแยกส่วน**แต่ทำงานร่วมกันเป็นระบบเดียวได้จริง — code ทั้งหมดถูก compile และรันเทสจริง
ยืนยันไว้แล้ว (ดูหัวข้อการตรวจสอบด้านล่าง)

บทถัดไป (**Part 107: Capstone — Building a Production-Grade CLI Tool**) จะนำทุกสิ่งที่เรียนมาตลอดหลักสูตร
(รวมทั้ง pattern ระดับสถาปัตยกรรมจากบทนี้) มาประกอบเป็นโปรเจกต์ capstone สุดท้ายที่เป็น command-line tool ระดับ
production เต็มรูปแบบ — เป็นการปิดหลักสูตรด้วยการลงมือสร้างจริงครบทุกด้าน ตั้งแต่ error handling, testing, ไปจน
ถึงการจัดโครงสร้างโค้ดที่ดูแลรักษาได้ในระยะยาว

---

**Part ก่อนหน้า:** [Contributing to Open Source Rust Projects](part-105-open-source-contributing.md) | **Part ถัดไป:** [Capstone: Building a Production-Grade CLI Tool](part-107-capstone-cli-tool.md)
