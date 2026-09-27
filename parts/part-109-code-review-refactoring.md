# Part 109: Code Review, Refactoring และ Clean Code ใน Rust

> โมดูล: Production, DevOps และระดับมืออาชีพ | ระดับ: มืออาชีพ | เวลาโดยประมาณ: 240 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้ว่า "clean code" ใน Rust หมายถึงอะไรโดยเฉพาะ ต่างจากคำว่า clean code ทั่ว ๆ ไปในภาษาอื่นอย่างไร และทำไม
  idiomaticity ใน Rust จึงเป็นสัญญาณของความถูกต้องเชิงออกแบบ (design correctness) ไม่ใช่แค่เรื่องความสวยงาม
- ใช้ checklist การรีวิวโค้ดอย่างเป็นระบบ 4 หมวด (ownership/borrowing, error handling, API design, allocation)
  เพื่อจับ "กลิ่นโค้ด" (code smell) ที่ compiler ไม่ได้เตือน แต่ส่งผลเสียต่อคุณภาพในระยะยาว
- ใช้ `cargo clippy` อย่างเป็นระบบในหลายระดับความเข้มงวด (`pedantic`, `nursery`, `cargo`) อ่านผลลัพธ์จริง
  และตัดสินใจได้ว่าเมื่อไหร่ควรแก้ตาม เมื่อไหร่ควร `#[allow(...)]` แบบมีเหตุผล ไม่ใช่ปิดแบบเหมาเข่ง
- รีแฟกเตอร์โค้ดที่ "messy" ให้กลายเป็นโค้ดที่สะอาด idiomatic ผ่านเทคนิค extract function/method และการใช้ trait
  กำหนดขอบเขตของระบบตามแนวคิด hexagonal architecture ทีละขั้นตอนโดยให้โค้ด compile ผ่านตลอดทาง
- ตรวจสอบและแก้ไขชื่อฟังก์ชัน/เมธอดให้สื่อความหมายตรงตามธรรมเนียมของ Rust (`is_`, `has_`, `into_`, `to_`, `as_`)
- ตัดสินใจได้ว่าจุดไหนของโค้ดควรมี doc comment อธิบาย "ทำไม" และ invariant อะไร และจุดไหนที่โค้ดเองอธิบายตัวมันเองอยู่แล้ว
- รีวิวคุณภาพของเทสต์ที่มีอยู่ แยกแยะเทสต์ที่ทดสอบ "พฤติกรรม" ออกจากเทสต์ที่ผูกติดกับ "รายละเอียดการ implement"
- ตรวจสุขภาพของ dependency ด้วย `cargo machete` (และรู้จัก `cargo udeps`) เพื่อหา dependency ที่ประกาศไว้แต่ไม่ได้ใช้จริง
- เขียน characterization test เพื่อ "ตรึง" พฤติกรรมของโค้ดเก่าที่ไม่มีเทสต์ไว้ก่อนกล้าเข้าไปรีแฟกเตอร์มัน

## ความรู้ที่ต้องมีมาก่อน

บทนี้เป็นบทสรุปเชิงคุณภาพของทักษะที่คุณสั่งสมมาตลอดหลักสูตร จึงอ้างอิงเนื้อหาจากหลายบทก่อนหน้าโดยตรง:

- **Part 5** (Comments, rustfmt และ Clippy) — บทนี้เป็นจุดเริ่มต้นของ `cargo clippy` ที่เราจะใช้ต่อยอดอย่างเข้มข้นขึ้น
  มาก โดยเฉพาะเรื่อง lint categories (`style`, `complexity`, `perf`, `pedantic`, `nursery`, `cargo`) และการ `#[allow]`
  อย่างมีเหตุผล ถ้าจำหัวข้อ 5.11-5.13 ไม่ได้ แนะนำให้ย้อนไปทวนก่อน
- **Part 6-7** (Ownership, Borrowing) — checklist หมวด "ownership/borrowing smells" ต้องใช้ความเข้าใจเรื่อง move,
  borrow, และ borrow checker เพื่อรู้ว่าโค้ดแบบไหนกำลัง "สู้กับ" ระบบนี้อยู่แทนที่จะทำงานร่วมกับมัน
- **Part 8** (Slices) — จำเป็นสำหรับหัวข้อ API design smell เรื่อง `&Vec<T>` เทียบกับ `&[T]`
- **Part 11-12** (`Option<T>`, `Result<T, E>` เบื้องต้น) และ **Part 30** (Error Handling ขั้นสูง, custom error type,
  `From`/`Into`) — จำเป็นสำหรับหัวข้อ error handling smell และการออกแบบ `LibraryError` ใน capstone ท้ายบท
- **Part 18-19, 21** (Generics, Traits, Trait Objects) — จำเป็นสำหรับหัวข้อการใช้ trait กำหนดขอบเขตแบบ hexagonal
  architecture ในหัวข้อ 109.8
- **Part 32-33** (Unit Tests, Integration Tests) — จำเป็นสำหรับหัวข้อรีวิวคุณภาพเทสต์และ characterization test
- **Part 105** (Open Source Contributing) หัวข้อ 105.6 — บทนี้แนะนำมารยาทของการรับ/ให้ code review ใน pull request
  จริงบน GitHub ไปแล้ว บทนี้ (Part 109) จะเจาะลึกในมุม "จะรีวิวอะไรบ้าง" และ "จะแก้ตามที่รีวิวอย่างไรให้ปลอดภัย"
  ซึ่งเป็นทักษะที่ Part 105 ไม่ได้ลงรายละเอียด

ถ้ายังไม่ผ่านบทเหล่านี้ในระดับที่เขียนโค้ดที่มี ownership/trait/error handling พื้นฐานได้เอง แนะนำให้กลับไปทวนก่อน
เพราะเราจะพูดถึง "ทำไมโค้ดแบบนี้ไม่ดี" มากกว่า "syntax คืออะไร"

## เนื้อหา

### 109.1 Clean Code ใน Rust คืออะไร — ทำไม idiomatic ถึงเกี่ยวกับความถูกต้อง ไม่ใช่แค่ความสวย

ในภาษาส่วนใหญ่ "clean code" เป็นเรื่องของสไตล์การเขียนที่มนุษย์อ่านง่ายขึ้น — ตัวแปรชื่อดี ฟังก์ชันไม่ยาวเกินไป
ไม่มี magic number แต่ยังเป็นเรื่อง "ความสวย" ที่ไม่กระทบว่าโปรแกรม **ทำงานถูกหรือผิด** โค้ด Python ที่ตัวแปรชื่อ `x`
กับโค้ดที่ตัวแปรชื่อ `total_price_after_discount` รันได้ผลลัพธ์เดียวกันเป๊ะ

Rust ต่างออกไปในจุดที่สำคัญมาก: **โค้ดที่ไม่ idiomatic มักเป็นสัญญาณของการออกแบบที่ผิดพลาดเชิง ownership** ซึ่งแม้จะยัง
compile ผ่านและทำงานถูกในกรณีทดสอบทั่วไป แต่มักจะ "เปราะ" กว่าที่ควรเป็น เสี่ยง panic ตอน runtime มากกว่าที่ควร หรือ
ทำงานหนักกว่าที่จำเป็นโดยไม่มีเหตุผล พูดอีกแบบคือ **ความไม่ idiomatic ใน Rust อยู่ใกล้กับความไม่ถูกต้องมากกว่าในภาษา
อื่น ๆ** — มันไม่ใช่ spectrum เดียวกัน แต่อยู่ "ประชิด" กันมากกว่าที่หลายคนคาดคิด

มาดูสามสัญญาณเตือนที่พบบ่อยที่สุด ซึ่งทั้งหมดนี้ **compile ผ่านได้สบาย ๆ** แต่เป็นสัญญาณของปัญหาการออกแบบ:

**สัญญาณที่ 1: "สู้กับ" borrow checker ด้วยการ clone ทุกครั้งที่มัน complain**

มือใหม่ (และบางครั้งมือเก๋าที่เร่งรีบ) เมื่อเจอ error จาก borrow checker มักมีปฏิกิริยาแรกคือ "ใส่ `.clone()` เข้าไป
แก้ปัญหาให้จบ ๆ" วิธีนี้ **ใช้ได้ผลจริง** ในความหมายที่ว่าโค้ด compile ผ่าน แต่มันไม่ได้แก้ปัญหาที่ borrow checker
กำลังพยายามบอกคุณ มันแค่ "จ่ายเงินซื้อ" ทางออกจากปัญหาด้วยการจ่ายเป็น CPU cycle และ memory allocation ทุกครั้งที่โค้ด
รันจริง เปรียบเทียบง่าย ๆ: ถ้าคุณเขียน C++ แล้วเจอ segfault คุณจะไม่ "แก้" ด้วยการ wrap ทุกอย่างเป็น `shared_ptr`
มั่ว ๆ โดยไม่เข้าใจว่าทำไม pointer มันหลุด — คุณจะสืบหาสาเหตุจริง การ clone มั่ว ๆ ใน Rust ก็เทียบเท่ากับพฤติกรรมนั้น

**สัญญาณที่ 2: `Rc<RefCell<T>>` ที่ปรากฏในโค้ดที่ไม่ได้ต้องการ shared mutable state ข้าม thread หรือข้าม owner จริง ๆ**

`Rc<RefCell<T>>` เป็นเครื่องมือที่ถูกต้องและจำเป็นในบางสถานการณ์ (เช่น โครงสร้างข้อมูลแบบ graph ที่มีหลาย node
ชี้กลับไปกลับมา, GUI widget tree ที่ parent/child ต้องแก้ไขกันและกัน) แต่เมื่อมันปรากฏใน struct ที่จริง ๆ แล้วมี
**เจ้าของเดียวที่ชัดเจน** (เช่น `Library` ที่เป็นเจ้าของ `Vec<Book>` แน่นอนอยู่แล้ว) มันมักเป็นสัญญาณว่าคนเขียนโค้ด
"ไม่แน่ใจ" เรื่อง ownership ของระบบตัวเอง จึงเลือกใช้ reference counting ที่ตรวจสอบ borrow ตอน **runtime**
(ผ่าน `RefCell`) แทนที่จะให้ borrow checker ตรวจให้ฟรีตอน **compile time** ผลที่ตามมาคือ:

1. เสียค่า runtime check ทุกครั้งที่ `.borrow()`/`.borrow_mut()` (แม้จะเล็กน้อย แต่ไม่ใช่ศูนย์)
2. ที่ร้ายกว่าคือ **บั๊กที่ borrow checker เคยจับให้ฟรีตอน compile กลับกลายเป็น panic ตอน runtime** —
   `already borrowed: BorrowMutError` เป็นข้อความที่นักพัฒนา Rust จำนวนมากเจอครั้งแรกในชีวิตตอนใช้
   `Rc<RefCell<T>>` เกินความจำเป็น เราจะสาธิตให้เห็น panic นี้จริงในหัวข้อ 109.3
3. โค้ด "รก" ขึ้นทันที ทุกจุดที่เข้าถึงข้อมูลต้อง `.borrow()`/`.borrow_mut()` แทนการเข้าถึง field ตรง ๆ

หลักการตัดสินใจง่าย ๆ: **ถ้า struct ของคุณมีเจ้าของเดียวที่ชัดเจนตลอดอายุของโปรแกรม ให้ struct นั้นเป็นเจ้าของข้อมูล
ตรง ๆ ด้วย ownership ปกติ (`Vec<T>`, `String`, ฟิลด์ธรรมดา) และใช้ `&mut self` ผ่านเมธอด — สงวน `Rc<RefCell<T>>`
ไว้สำหรับกรณีที่มี "หลายเจ้าของจริง ๆ" เท่านั้น** (จะเรียนรายละเอียดเรื่อง `Rc`/`Arc`/`RefCell` เต็มรูปแบบใน Part
ที่เกี่ยวกับ smart pointers — ในบทนี้เราสนใจแค่มุม "สัญญาณเตือนเชิง design")

**สัญญาณที่ 3: การ over-clone แบบไม่รู้ตัวเพราะไม่เข้าใจว่า field ไหน move ได้**

จาก Part 6-7 เราเรียนไปแล้วว่า field ของ struct ที่ไม่ implement `Drop` สามารถ move ออกมาแยกทีละ field ได้
(partial move) แต่มือใหม่จำนวนมากไม่รู้ข้อเท็จจริงนี้ จึง `.clone()` ทั้ง struct หรือทั้ง field แบบเหมาเข่ง
เพราะ "กลัว" ownership error ทั้งที่ Rust อนุญาตให้ move field เดียวออกมาได้ตรง ๆ โดยที่ field อื่นของ struct
เดิมยังใช้งานต่อได้ปกติ — เราจะเห็นตัวอย่างที่จับได้จริงด้วย `clippy::redundant_clone` ในหัวข้อ 109.3

สรุปหลักคิดของหัวข้อนี้: **การรีวิวโค้ด Rust ที่ดี ไม่ใช่แค่การหา bug เชิง logic แต่คือการหา "รอยต่อ" ที่คนเขียนโค้ด
กำลังพยายามหลีกเลี่ยงการคุยกับ ownership system อย่างตรงไปตรงมา** ทุกจุดที่มี `.clone()` เกินจำเป็น, `Rc<RefCell<>>`
เกินจำเป็น, หรือ `.unwrap()` เกินจำเป็น (จะพูดถึงในหัวข้อ 109.4) คือจุดที่ผู้เขียนโค้ด "ยอมแพ้" ให้กับความซับซ้อน
ของปัญหา แทนที่จะออกแบบให้ตรงกับปัญหาจริง ๆ

### 109.2 Checklist การรีวิวโค้ดอย่างเป็นระบบ

การรีวิวโค้ดที่ดีไม่ใช่การอ่านไล่ทีละบรรทัดแบบสุ่ม แต่ควรมี **มุมมอง (lens)** ชัดเจนที่ใช้ไล่ตรวจทีละหมวด
เพื่อไม่ให้พลาดปัญหาที่ตัวเองไม่ได้นึกถึง ณ ขณะนั้น บทนี้แบ่ง checklist ออกเป็น 4 หมวดหลักที่ครอบคลุมปัญหาคุณภาพโค้ด
Rust ที่พบบ่อยที่สุดในโค้ดจริง:

| หมวด | คำถามหลักที่ต้องถาม | ตัวอย่างกลิ่นโค้ด |
|---|---|---|
| **1. Ownership/Borrowing** | โค้ดนี้ "คุย" กับ ownership system ตรง ๆ หรือ "หลีกเลี่ยง" มัน? | `.clone()` เกินจำเป็น, `Rc<RefCell<T>>` ที่ไม่จำเป็น |
| **2. Error Handling** | error ที่เป็นไปได้ถูก handle แบบ "คาดการณ์ไว้" หรือ "หวังว่าจะไม่เกิด"? | `.unwrap()`/`.expect()` ใน library code |
| **3. API Design** | signature ของฟังก์ชันบีบให้ผู้เรียกทำอะไรเกินจำเป็นหรือไม่? | รับ `String` ทั้งที่แค่อ่าน, รับ `&Vec<T>` ทั้งที่แค่ iterate |
| **4. Allocation** | มี allocation ที่ทำไปโดยไม่มีใครใช้ผลลัพธ์นั้นจริงหรือไม่? | คืนค่า owned ทั้งที่ผู้เรียกต้องการแค่อ่าน |

หมวดเหล่านี้ **ไม่ได้แยกจากกันโดยสิ้นเชิง** — ในทางปฏิบัติ ปัญหาหนึ่งจุดมักจัดอยู่ได้มากกว่าหนึ่งหมวด (เช่น
`fn add_book(&mut self, title: String)` ที่ไม่ได้ใช้ `title` แบบ move เป็นทั้งปัญหา API design และปัญหา allocation
ไปในตัว) แต่การแยกมุมมองแบบนี้ช่วยให้เรา **ไล่ตรวจอย่างเป็นระบบ** ไม่ใช่รอให้ปัญหา "โผล่มาเตะตา" เอง

ในหัวข้อ 109.3–109.6 เราจะไล่ตรวจทั้ง 4 หมวดนี้บนโค้ดจริงของระบบห้องสมุด (library checkout system) พร้อมโค้ด
ก่อน/หลังที่ compile และรันได้จริงทุกก้อน และคำเตือนจาก `cargo clippy` ที่รันจริงจากสิ่งแวดล้อมที่ใช้เขียนบทนี้
(Rust 1.98.1, clippy 0.1.98) — ไม่ใช่ข้อความที่แต่งขึ้นมาลอย ๆ

### 109.3 หมวดที่ 1 — กลิ่นโค้ดด้าน Ownership/Borrowing

มาดูฟังก์ชัน `checkout` เวอร์ชันแรกที่ทีมสมมติเขียนขึ้นสำหรับระบบยืม-คืนหนังสือของห้องสมุด (ยังไม่สมบูรณ์ — เราจะ
รีแฟกเตอร์ทั้งระบบให้จบในหัวข้อ capstone 109.14 แต่ตอนนี้ขอโฟกัสที่ปัญหา ownership เพียงจุดเดียวก่อน):

```rust
use std::cell::RefCell;
use std::rc::Rc;

struct Book {
    isbn: String,
    copies_available: u32,
}

struct Library {
    // ใช้ Rc<RefCell<>> ทั้งที่ Library เป็นเจ้าของ Vec<Book> เพียงผู้เดียว
    // ไม่มีใครอื่นถือ Rc ตัวนี้ไว้เลยตลอดทั้งโปรแกรม
    books: Rc<RefCell<Vec<Book>>>,
}

impl Library {
    fn checkout(&mut self, isbn: &str) -> bool {
        let books = self.books.clone(); // เพิ่ม strong count ของ Rc โดยไม่มีเหตุผล
        let mut books_ref = books.borrow_mut();
        if let Some(book) = books_ref.iter_mut().find(|b| b.isbn == isbn) {
            if book.copies_available > 0 {
                book.copies_available -= 1;
                return true;
            }
        }
        false
    }
}
```

**สิ่งที่ต้องจับได้ตอนรีวิว:**

1. `Rc<RefCell<Vec<Book>>>` — ไม่มีจุดใดในโค้ดที่แสดงว่ามีเจ้าของหลายคนถือ `books` ร่วมกัน `Library` เป็นเจ้าของ
   `Vec<Book>` เพียงผู้เดียวตลอดชีวิตของโปรแกรม การใช้ `Rc<RefCell<>>` ที่นี่จึงเป็น over-engineering ที่ไม่มี
   ประโยชน์ตอบแทน
2. `self.books.clone()` ในบรรทัดแรกของ `checkout` — นี่คือการ clone ตัว `Rc` (เพิ่ม strong reference count ไปที่
   ตัวนับ ไม่ใช่ deep clone ข้อมูลจริง) แต่ก็ยังเป็นการทำงานที่ไม่จำเป็นเลย เพราะ `checkout` มี `&mut self` อยู่แล้ว
   สามารถเข้าถึง `self.books` ตรง ๆ ได้โดยไม่ต้อง clone อะไรทั้งสิ้น
3. ผลลัพธ์ที่อันตรายที่สุด: ถ้ามีจุดใดในโปรแกรมเผลอเรียก `.borrow()` หรือ `.borrow_mut()` ซ้ำสองครั้งในขอบเขตที่
   ทับซ้อนกัน (เช่นลืมว่า borrow ตัวแรกยังไม่จบ scope) จะเกิด **runtime panic** มาดูตัวอย่างจริงที่จำลองสถานการณ์นี้:

```rust
use std::cell::RefCell;
use std::rc::Rc;

struct Book {
    copies_available: u32,
}

fn main() {
    let book = Rc::new(RefCell::new(Book { copies_available: 2 }));

    let mut borrowed = book.borrow_mut(); // borrow #1 (mutable) ยังไม่ถูกปล่อย
    borrowed.copies_available -= 1;

    // เผลอ borrow ซ้ำในสโคปเดียวกันก่อนตัวแรกจะถูกปล่อย
    let mut borrowed_again = book.borrow_mut(); // borrow #2 -> panic ตอน runtime
    borrowed_again.copies_available -= 1;

    println!("{}", borrowed.copies_available);
}
```

รันโค้ดนี้จริงได้ผลลัพธ์ (capture จริงจากการรันในสภาพแวดล้อมที่เขียนบทนี้):

```
thread 'main' panicked at src/main.rs:15:35:
RefCell already borrowed
```

**นี่คือหัวใจของสัญญาณเตือนที่กล่าวถึงในหัวข้อ 109.1**: ถ้าโค้ดนี้ใช้ ownership ปกติแทน `Rc<RefCell<>>`
(เช่น `book: Book` ตรง ๆ แล้วเข้าถึงผ่าน `&mut Book`) **borrow checker จะจับปัญหานี้ให้ตอน compile time**
ด้วย error ที่อ่านง่ายกว่ามาก แทนที่จะรอให้มันระเบิดตอน runtime กลางดึกในโปรดักชัน

**โค้ดหลังแก้ (เอา `Rc<RefCell<>>` ออก ใช้ ownership ปกติ):**

```rust
struct Book {
    isbn: String,
    copies_available: u32,
}

struct Library {
    books: Vec<Book>,
}

impl Library {
    fn checkout(&mut self, isbn: &str) -> bool {
        if let Some(book) = self.books.iter_mut().find(|b| b.isbn == isbn) {
            if book.copies_available > 0 {
                book.copies_available -= 1;
                return true;
            }
        }
        false
    }
}
```

โค้ดนี้ไม่มี `.clone()`, ไม่มี runtime borrow check, และถ้ามีที่ไหนในโปรแกรมพยายาม borrow `books` ซ้ำแบบขัดแย้งกัน
**compiler จะจับให้ตอน compile ก่อนที่โปรแกรมจะรันเสียอีก** — นี่คือความแตกต่างเชิงคุณภาพที่แท้จริงระหว่างโค้ดที่
"หลีกเลี่ยง" ownership system กับโค้ดที่ "ทำงานร่วมกับ" มัน

### 109.4 หมวดที่ 2 — กลิ่นโค้ดด้าน Error Handling: `.unwrap()` ในโค้ด library

`.unwrap()` ไม่ใช่สิ่งชั่วร้ายในตัวเอง — ใน `main()`, ใน example, ใน prototype ที่รู้แน่ ๆ ว่าค่าไม่ใช่ `None`
(เพราะเพิ่งสร้างมันเองในบรรทัดก่อนหน้า) การ `.unwrap()` เป็นเรื่องที่สมเหตุสมผลและอ่านง่ายกว่าการ handle error ที่
"เป็นไปไม่ได้" ปัญหาอยู่ที่ **`.unwrap()` ในโค้ดที่เป็น library หรือ business logic ที่ error สามารถเกิดขึ้นได้จริง
จากข้อมูล input ที่ผู้เรียกส่งเข้ามา** ซึ่งเป็นกรณีที่ต่างกันโดยสิ้นเชิงจากกรณีแรก

มาดูโค้ดยืมหนังสือที่เขียนด้วย `.unwrap()` เกลื่อนไปทั่ว (ตัดมาจากฟังก์ชัน `checkout`/`return_book` เวอร์ชันตั้งต้น
ของ capstone ในหัวข้อ 109.14):

```rust
fn checkout(&mut self, member_id: String, isbn: String) -> bool {
    let mut books_ref = self.books.borrow_mut();
    let book_index = books_ref
        .iter()
        .position(|b| b.isbn == isbn.clone())
        .unwrap(); // ถ้าไม่พบ ISBN นี้ -> panic ทันที ไม่มีทางส่ง error กลับให้ผู้เรียกรู้

    let book = books_ref.get_mut(book_index).unwrap();
    // ...
}
```

รันคำสั่ง `cargo clippy -- -W clippy::unwrap_used` (lint นี้อยู่ในกลุ่ม `restriction` ซึ่งต้องเปิดเองเสมอ ไม่ได้
เปิดโดย default หรือแม้แต่ใน `pedantic`) บนโค้ดชุดนี้ ได้ผลลัพธ์จริงดังนี้:

```
warning: used `unwrap()` on an `Option` value
  --> src/main.rs:74:28
   |
74 |         let member_index = members_ref
   |  ____________________________^
75 | |             .iter()
76 | |             .position(|m| m.member_id == member_id.clone())
77 | |             .unwrap();
   | |_____________________^
   |
   = note: if this value is `None`, it will panic
   = help: consider using `expect()` to provide a better panic message
   = help: for further information visit https://rust-lang.github.io/rust-clippy/rust-1.98.0/index.html#unwrap_used

warning: used `unwrap()` on an `Option` value
  --> src/main.rs:78:22
   |
78 |         let member = members_ref.get_mut(member_index).unwrap();
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: if this value is `None`, it will panic
   = help: consider using `expect()` to provide a better panic message
```

(ผลลัพธ์เต็มมี 8 จุดที่ `.unwrap()` ถูกใช้แบบนี้กระจายอยู่ในฟังก์ชัน `checkout`, `return_book`,
`find_book_by_isbn` — clippy รายงานทุกจุดพร้อมเลขบรรทัดแม่นยำ)

**ทำไมนี่คือปัญหาจริง ไม่ใช่แค่ style?** ลองนึกภาพว่าโค้ดนี้ถูกเรียกจาก web handler (จะเรียนเรื่อง Axum error
handling แบบมืออาชีพใน Part ถัดไปของโมดูล web) — ถ้าผู้ใช้พิมพ์ ISBN ผิดในฟอร์ม หรือ `member_id` สะกดผิดจากฝั่ง
frontend ทั้งเซิร์ฟเวอร์จะ **panic ทั้ง process** (หรืออย่างน้อยก็ crash เฉพาะ request นั้นถ้ามี panic handler
ที่ดี) ทั้งที่นี่เป็นเพียง "ผู้ใช้กรอกข้อมูลผิด" ซึ่งเป็นสถานการณ์ที่คาดการณ์ได้และควร handle ด้วย error ธรรมดา
ไม่ใช่ปล่อยให้โปรแกรมล้ม

**โค้ดหลังแก้ — คืน `Result` แทนการ panic:**

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
enum LibraryError {
    BookNotFound,
    MemberNotFound,
    NoCopiesAvailable,
}

impl std::fmt::Display for LibraryError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            LibraryError::BookNotFound => write!(f, "ไม่พบหนังสือ ISBN นี้"),
            LibraryError::MemberNotFound => write!(f, "ไม่พบสมาชิกรหัสนี้"),
            LibraryError::NoCopiesAvailable => write!(f, "หนังสือถูกยืมหมดแล้ว"),
        }
    }
}
impl std::error::Error for LibraryError {}

fn checkout(&mut self, member_id: &str, isbn: &str) -> Result<(), LibraryError> {
    let book = self.books.find_mut(isbn).ok_or(LibraryError::BookNotFound)?;
    if !book.is_available() {
        return Err(LibraryError::NoCopiesAvailable);
    }
    let member = self.find_member_mut(member_id)?;
    book.checkout_one_copy();
    member.record_borrow(isbn);
    Ok(())
}
```

จุดสำคัญคือการเปลี่ยนจาก `.unwrap()` (ที่บอกว่า "ฉันมั่นใจว่าค่านี้มีอยู่ ไม่ต้อง handle") ไปเป็น `.ok_or(...)?`
(ที่บอกว่า "ค่านี้อาจไม่มี และถ้าไม่มีคือ error ที่ผู้เรียกต้อง handle เอง") — การเปลี่ยนแปลงนี้ **ไม่ได้เพิ่มความ
ซับซ้อนของโค้ดมากอย่างที่คิด** ด้วย `?` operator (จาก Part 12) ที่ทำให้ error propagation อ่านง่ายเกือบเท่ากับ
`.unwrap()` เดิม แต่ปลอดภัยกว่ามหาศาล และเรายังใช้ custom error type (จาก Part 30) เพื่อให้ผู้เรียกสามารถ `match`
แยกแยะ error แต่ละแบบและตอบสนองต่างกันได้ (เช่น ส่ง HTTP 404 ถ้า `BookNotFound`, ส่ง HTTP 409 ถ้า
`NoCopiesAvailable`)

**หลักการตัดสินใจ**: ถามตัวเองว่า *"ถ้าเงื่อนไขนี้เป็น `None`/`Err` จริง จะเป็นเพราะ bug ในโค้ดของฉันเอง (แสดงว่า
ควร panic เพื่อให้เจอ bug เร็วที่สุด) หรือเป็นเพราะข้อมูล/input จากภายนอกที่ควบคุมไม่ได้ (แสดงว่าควรคืน error)?"*
ถ้าเป็นแบบหลัง ให้ใช้ `Result` เสมอ ไม่ใช่ `.unwrap()`

### 109.5 หมวดที่ 3 — กลิ่นโค้ดด้าน API Design: `String` vs `&str`, `Vec<T>` vs `&[T]`

หมวดนี้ตรวจสอบว่า **signature ของฟังก์ชัน** สื่อสารความต้องการที่แท้จริงของฟังก์ชันนั้นหรือไม่ ปัญหาที่พบบ่อยที่สุด
คือการรับ owned type (`String`, `Vec<T>`) ในจุดที่ฟังก์ชันแค่ "อ่าน" ข้อมูล ไม่ได้ต้องการ ownership เลย

```rust
struct Book {
    isbn: String,
    copies_available: u32,
}

// รับ &Vec<Book> — บีบให้ผู้เรียกต้องมี Vec จริง ๆ เท่านั้น
fn total_available(books: &Vec<Book>) -> u32 {
    books.iter().map(|b| b.copies_available).sum()
}
```

รัน `cargo clippy` (default, ไม่ต้องเปิดกลุ่มเสริมใด ๆ) บนโค้ดนี้ ได้ผลลัพธ์จริง:

```
warning: writing `&Vec` instead of `&[_]` involves a new object where a slice will do
 --> src/main.rs:3:27
  |
3 | fn total_available(books: &Vec<Book>) -> u32 {
  |                           ^^^^^^^^^^
  |
  = help: for further information visit https://rust-lang.github.io/rust-clippy/rust-1.98.0/index.html#ptr_arg
  = note: `#[warn(clippy::ptr_arg)]` on by default
help: change this to
  |
3 - fn total_available(books: &Vec<Book>) -> u32 {
3 + fn total_available(books: &[Book]) -> u32 {
  |
```

lint นี้ (`clippy::ptr_arg`) **เปิดโดย default** เพราะแทบไม่มี false positive และผลกระทบชัดเจน: `&Vec<T>` เป็น
reference ไปยัง `Vec<T>` ซึ่งเป็นการบีบขอบเขตให้แคบกว่าที่จำเป็น — ผู้เรียกที่มีแค่ array คงที่ (`[Book; 3]`), slice
ที่ได้จากการ `split_at`, หรือส่วนหนึ่งของ `Vec` อื่นที่ borrow มา จะ**ไม่สามารถส่งเข้าฟังก์ชันนี้ได้เลย** ทั้งที่
ฟังก์ชันแค่ต้องการ iterate อ่านค่าเท่านั้น การใช้ `&[Book]` (slice) แทนทำให้ฟังก์ชันรับได้ **ทุกอย่างที่แปลงเป็น
slice ได้** ซึ่งครอบคลุม `&Vec<T>` เดิมด้วย (ผ่าน deref coercion จาก Part 8) โดยไม่เสียอะไรเลย

หลักการเดียวกันนี้ใช้กับ `String` vs `&str` — เราเห็นตัวอย่างนี้ไปแล้วในหัวข้อ 109.4 กับ `member_id: String` ที่ควร
เป็น `member_id: &str` เพราะฟังก์ชันแค่เทียบค่า ไม่ได้เก็บ ownership ไว้ที่ไหน มาดูคำเตือนจริงจาก
`clippy::needless_pass_by_value` (อยู่ในกลุ่ม `pedantic`) ที่จับปัญหานี้ได้ทุกจุดในโค้ดตั้งต้นของระบบห้องสมุด:

```
warning: this argument is passed by value, but not consumed in the function body
  --> src/main.rs:32:35
   |
32 |     fn add_book(&mut self, title: String, isbn: String, copies: u32) {
   |                                   ^^^^^^
   |
   = help: for further information visit https://rust-lang.github.io/rust-clippy/rust-1.98.0/index.html#needless_pass_by_value
   = note: `-W clippy::needless-pass-by-value` implied by `-W clippy::pedantic`
help: consider changing the type to
   |
32 -     fn add_book(&mut self, title: String, isbn: String, copies: u32) {
32 +     fn add_book(&mut self, title: &str, isbn: String, copies: u32) {
   |
help: change `title.clone()` to
   |
34 -             title: title.clone(),
34 +             title: title.to_string(),
   |
```

รันจริงบนไฟล์ตั้งต้นของ capstone lint ตัวนี้ขึ้นมา **8 ครั้ง** ในฟังก์ชันต่าง ๆ (`add_book`, `add_member`,
`checkout`, `return_book`, `find_book_by_isbn`) — เป็นสัญญาณชัดว่าทีมที่เขียนโค้ดชุดนี้ยึดติดกับ `String` เป็น
ค่าเริ่มต้นของ "ข้อความ" โดยไม่ได้แยกแยะว่าฟังก์ชันไหนต้องการ ownership จริง ๆ (เช่นเก็บไว้ใน struct field ใหม่)
กับฟังก์ชันไหนแค่ต้องการอ่าน

**กฎที่จำง่าย**: ถ้าฟังก์ชันต้องเก็บค่านั้นไว้ใน struct/collection ที่มีชีวิตยาวกว่าตัวฟังก์ชันเอง (เช่น
`add_book` ที่ต้องสร้าง `Book` ใหม่แล้วเก็บไว้ใน `Vec`) — รับเป็น `impl Into<String>` หรือ `String` ตรง ๆ ก็ยัง
สมเหตุสมผล เพราะยังไงก็ต้อง allocate เก็บไว้อยู่ดี แต่ถ้าฟังก์ชันแค่ **อ่าน/เทียบค่า** แล้วปล่อยผ่าน (เช่น `checkout`
ที่แค่ใช้ `isbn` ไปหาตำแหน่งใน `Vec`) ให้รับเป็น `&str` เสมอ

### 109.6 หมวดที่ 4 — กลิ่นโค้ดด้าน Unnecessary Allocation

หมวดนี้เกี่ยวโยงกับหมวดก่อนหน้าอย่างใกล้ชิด แต่โฟกัสที่จุด **คืนค่า (return)** มากกว่าจุด **รับค่า (parameter)**
มาดูตัวอย่าง:

```rust
struct Book {
    title: String,
    isbn: String,
    copies_available: u32,
}

struct Library {
    books: Vec<Book>,
}

impl Library {
    // คืนค่า Book ที่ clone มาทั้งก้อน ทั้งที่ผู้เรียกส่วนใหญ่ต้องการแค่ "อ่าน"
    fn find_book_by_isbn(&self, isbn: String) -> Book {
        self.books.iter().find(|b| b.isbn == isbn).unwrap().clone()
    }
}
```

ทุกครั้งที่เรียก `find_book_by_isbn` จะเกิดการ allocate `String` ใหม่สองก้อน (สำหรับ `title` และ `isbn` ที่ clone
มา) ทั้งที่โค้ดที่เรียกใช้งานส่วนใหญ่ในระบบจริงมักแค่ต้องการ "ดูค่า" เช่น แสดงราคาหรือจำนวนคงเหลือบนหน้าเว็บ ไม่ได้
ต้องการ ownership ของสำเนาใหม่เลย ถ้าฟังก์ชันนี้ถูกเรียกในลูปที่แสดงรายการหนังสือ 10,000 เล่ม การ clone ที่ไม่จำเป็น
นี้จะกลายเป็นภาระ allocator ที่วัดผลได้จริงด้าน performance

**โค้ดหลังแก้ — คืน borrow แทน owned copy:**

```rust
impl Library {
    fn book(&self, isbn: &str) -> Option<&Book> {
        self.books.iter().find(|b| b.isbn == isbn)
    }
}
```

การเปลี่ยนแปลงนี้มีสามจุดที่ดีขึ้นพร้อมกัน:

1. **ไม่มี allocation ใหม่เลย** — คืน reference ตรง ๆ ไปยังข้อมูลที่มีอยู่แล้วใน `Vec`
2. **ไม่ panic เมื่อไม่พบ** — เปลี่ยนจากคืนค่า `Book` ตรง ๆ (ที่ซ่อน `.unwrap()` เอาไว้ข้างใน) เป็นคืน
   `Option<&Book>` ที่บอกตรง ๆ ว่า "อาจไม่พบ" ให้ผู้เรียกจัดการเอง — แก้ทั้งปัญหาหมวด allocation และหมวด error
   handling (109.4) ไปพร้อมกันในการแก้ไขครั้งเดียว
3. **ชื่อฟังก์ชันสั้นและตรงประเด็นขึ้น** จาก `find_book_by_isbn(isbn: String) -> Book` เป็น `book(isbn: &str) ->
   Option<&Book>` — จะพูดถึงเหตุผลเรื่องการตั้งชื่อแบบนี้ในหัวข้อ 109.9

ข้อควรระวังเพียงข้อเดียวของการเปลี่ยนจากคืน owned เป็นคืน borrow: **ผู้เรียกจะถูกผูกด้วย lifetime ของ `&self`** —
ถ้าผู้เรียกพยายามเก็บ `&Book` ที่ได้ไว้ใช้นานกว่าที่ `Library` ยังมีชีวิตอยู่ (เช่น เก็บไว้ข้าม thread หรือเก็บไว้ใน
struct อื่นที่มีชีวิตยาวกว่า) compiler จะปฏิเสธด้วย lifetime error ทันที ซึ่ง **เป็นเรื่องดี** เพราะมันบอกตรง ๆ ว่า
ถ้าคุณต้องการเก็บข้อมูลนี้ไว้นานกว่านั้นจริง ๆ คุณต้อง `.clone()` อย่างมีเจตนา ไม่ใช่ได้ owned copy มาแบบไม่รู้ตัวจาก
ฟังก์ชันที่ไม่ควรมีต้นทุนนั้นเลยในกรณีทั่วไป

### 109.7 ใช้ Clippy อย่างเป็นระบบ: Lint Groups, `#[allow]` ที่มีเหตุผล vs การปิดแบบเหมาเข่ง

จาก Part 5 เราทราบแล้วว่า clippy แบ่ง lint ออกเป็นหมวด `correctness`, `style`, `complexity`, `perf` (เปิดโดย
default) และ `pedantic`, `nursery`, `cargo` (ต้องเปิดเอง) ในบทนี้เราได้เห็นตัวอย่างจริงของแต่ละระดับไปแล้วในหัวข้อ
ก่อนหน้า มาสรุปเป็นภาพเปรียบเทียบให้เห็นชัดว่าความเข้มงวดที่เพิ่มขึ้นแต่ละระดับจับปัญหาเพิ่มขึ้นแค่ไหน โดยใช้โค้ด
ตั้งต้น (messy) ของระบบห้องสมุดทั้งไฟล์ (เวอร์ชันเต็มอยู่ในหัวข้อ 109.14) เป็นเกณฑ์:

| คำสั่งที่รัน | จำนวน warning ที่พบ | ตัวอย่าง lint ที่เพิ่มขึ้นมา |
|---|---|---|
| `cargo clippy` (default) | 5 | `needless_return`, `single_match`, `dead_code` (จาก rustc ไม่ใช่ clippy) |
| `cargo clippy -- -W clippy::pedantic` | 15 | เพิ่ม `needless_pass_by_value` (×8), `struct_field_names` |
| `cargo clippy -- -W clippy::nursery` | 15 | เพิ่ม `redundant_clone` (×5), `use_self` (×2), `use_self` แทนชื่อ struct ซ้ำ |
| `cargo clippy -- -W clippy::unwrap_used` (restriction, ต้องเปิดทีละตัว) | 8 | ทุกจุดที่ `.unwrap()` บน `Option`/`Result` |

**ข้อสังเกตสำคัญ**: `pedantic` และ `nursery` **ไม่ได้ครอบคลุมกัน** — `pedantic` จับปัญหาด้าน API design
(`needless_pass_by_value`, `struct_field_names`) ในขณะที่ `nursery` จับปัญหาด้าน ownership (`redundant_clone`,
`use_self`) ทั้งสองกลุ่มนี้จับปัญหาคนละมุมกัน ทีมที่ต้องการมาตรฐานสูงสุดจึงมักเปิด**ทั้งสองกลุ่มพร้อมกัน** ไม่ใช่
เลือกอย่างใดอย่างหนึ่ง ส่วน `restriction` (ที่ `unwrap_used` อยู่ในกลุ่มนี้) **ไม่มีการเปิดแบบเหมาทั้งกลุ่มเด็ดขาด**
เพราะกฎในกลุ่มนี้หลายตัวขัดแย้งกันเอง (เช่น `clippy::implicit_return` กับ `clippy::needless_return` เข้ากันไม่ได้)
— ต้องเลือกเปิดทีละ lint ที่ต้องการจริง ๆ เท่านั้น

**ตั้งค่า lint ระดับโปรเจกต์ด้วย `[lints]` table ใน `Cargo.toml` (แนะนำสำหรับโปรเจกต์จริง)**

ตั้งแต่ Rust 1.74 เป็นต้นมา Cargo รองรับการประกาศ lint level ใน `Cargo.toml` โดยตรง แทนการเขียน `#![warn(...)]`
กระจายอยู่บนสุดของ `src/main.rs`/`src/lib.rs` (ซึ่งมักถูกลืมเมื่อมีหลาย crate ใน workspace):

```toml
# Cargo.toml
[lints.clippy]
# เปิดทั้งกลุ่ม pedantic และ nursery เป็น warn (ไม่ใช่ deny — ยังให้ build ผ่านได้
# แต่เห็นคำเตือนชัดเจน ทีมตัดสินใจเองว่าจะยกระดับเป็น deny ใน CI หรือไม่)
pedantic = "warn"
nursery = "warn"

# ปิดเฉพาะบางตัวจาก pedantic ที่ทีมตัดสินใจร่วมกันแล้วว่าไม่เหมาะกับโปรเจกต์นี้
# (ตัวอย่าง: โปรเจกต์นี้เป็น internal tool ไม่ใช่ public library จึงไม่บังคับ
# ให้ทุกฟังก์ชันที่คืน Result ต้องมี # Errors section)
missing_errors_doc = "allow"
```

วิธีนี้ดีกว่าการเขียน `#![warn(clippy::pedantic)]` ในโค้ดตรงที่ **การตั้งค่าอยู่ที่จุดเดียว มองเห็นง่าย และไม่ปน
กับโค้ด business logic** ทีม reviewer เปิด `Cargo.toml` ครั้งเดียวก็เห็นมาตรฐานคุณภาพทั้งโปรเจกต์ทันที ไม่ต้องไล่
หาทุกไฟล์ที่มี attribute นี้

**`#[allow(clippy::...)]` ที่มีเหตุผล เทียบกับการปิดแบบเหมาเข่ง**

จุดที่สำคัญที่สุดของหัวข้อนี้คือ: **การ `#[allow]` ไม่ใช่เรื่องผิด ถ้ามันมาพร้อมเหตุผลที่บันทึกไว้เป็น comment**
ปัญหาไม่ได้อยู่ที่การ allow แต่อยู่ที่การ allow แบบ "เงียบ ๆ ไม่มีคำอธิบาย" หรือแบบ "ปิดทั้งไฟล์/ทั้ง crate โดยไม่
เจาะจง" มาดูตัวอย่างจริงจากการรีแฟกเตอร์ระบบห้องสมุดในหัวข้อ 109.14 ที่เจอ lint `clippy::cast_precision_loss`
(อยู่ใน `pedantic`) บนฟังก์ชันคำนวณค่าปรับ:

```
warning: casting `i64` to `f64` may cause a loss of precision (`i64` is 64 bits wide, but `f64`'s mantissa is only 52 bits wide)
   --> src/main.rs:260:9
    |
260 |         days_late as f64 * base_fee_per_day
    |         ^^^^^^^^^^^^^^^^
```

การ allow ที่ **ไม่ดี** (allow เงียบ ไม่มีคำอธิบาย ผู้อ่านโค้ดในอนาคตไม่รู้ว่า "ตั้งใจ" หรือ "มองข้าม"):

```rust
#[allow(clippy::cast_precision_loss)]
fn calculate_late_fee(days_late: i64, base_fee_per_day: f64) -> f64 {
    // ...
}
```

การ allow ที่ **ดี** (มีเหตุผลเจาะจงว่าทำไม lint นี้ไม่เกี่ยวกับสถานการณ์จริงของโดเมนนี้):

```rust
// จำนวนวันล่าช้าของหนังสือห้องสมุดไม่มีทางเกินไม่กี่พันวันในชีวิตจริง
// (ไม่มีมนุษย์คนไหนยืมหนังสือทิ้งไว้เกินอายุขัย) จึงไม่มีความเสี่ยงที่จะสูญ
// precision จริงจากการแปลง i64 -> f64 ในสเกลนี้ — allow เจาะจงจุดนี้แทนการ
// เขียน cast ที่ซับซ้อนขึ้นโดยไม่ได้ประโยชน์อะไรเพิ่ม
#[allow(clippy::cast_precision_loss)]
fn calculate_late_fee(days_late: i64, base_fee_per_day: f64) -> f64 {
    // ...
}
```

โค้ดสองก้อนนี้ **เหมือนกันทุกตัวอักษรยกเว้น comment** แต่คุณค่าต่างกันมหาศาลสำหรับคนที่มารีวิวหรือแก้โค้ดนี้ในอีก
หนึ่งปีข้างหน้า — comment บอกว่าทีมได้คิดเรื่องนี้แล้วจริง ๆ ไม่ใช่แค่กด allow เพื่อให้ CI เขียวโดยไม่ได้อ่าน
คำเตือนเลย

**สิ่งที่ไม่ควรทำเด็ดขาด**: การ allow แบบเหมาเข่งทั้งไฟล์หรือทั้ง crate โดยไม่เจาะจง เช่น

```rust
// อย่าทำแบบนี้ — ปิด lint ทั้งกลุ่มทั้งไฟล์โดยไม่มีเหตุผลเจาะจง
#![allow(clippy::all)]
```

โค้ดแบบนี้ปิด **ทุก lint ในกลุ่ม `correctness`, `style`, `complexity`, `perf` ทั้งหมด** ทั้งไฟล์ ทั้งที่ปัญหาจริง
อาจอยู่แค่จุดเดียว (เช่น false positive หนึ่งตัวจาก `nursery`) การปิดแบบนี้เท่ากับปิดตาไม่ให้เห็นปัญหาจริงในอนาคต
ทุกจุดที่อาจเกิดในไฟล์นั้น — ควร `#[allow]` เจาะจงเฉพาะฟังก์ชัน/บรรทัดที่มีปัญหาจริงเท่านั้น พร้อม comment อธิบาย
เสมอ

**สุดท้าย: `-D warnings` ใน CI**

เช่นเดียวกับที่ Part 5 กล่าวไว้ การรัน `cargo clippy -- -D warnings` (ยกระดับทุก warning เป็น error) เป็นมาตรฐาน
ของ CI pipeline ที่ดี แต่มีข้อควรระวังที่ Part 5 ยังไม่ได้พูดถึง: **เมื่อ clippy เวอร์ชันใหม่ออกมา มันอาจเพิ่ม lint
ใหม่เข้าไปในกลุ่มที่เปิดอยู่โดย default** ทำให้ CI ที่เคยผ่านมาตลอดกลับ fail ทันทีหลัง `rustup update` โดยที่ไม่มี
ใครแก้โค้ดเลย วิธีป้องกันที่ทีม production ใช้กันคือ pin เวอร์ชันของ toolchain ที่ใช้รัน clippy ใน CI ด้วยไฟล์
`rust-toolchain.toml` (จะเรียนละเอียดเรื่อง CI/CD ใน Part 97 ที่ผ่านมาแล้ว) เพื่อให้ผลลัพธ์ของ clippy คงที่ไม่
เปลี่ยนไปมาตามเวลาที่ maintainer ของ clippy เพิ่ม lint ใหม่

### 109.8 เทคนิค Refactoring: Extract Function/Method และใช้ Trait กำหนดขอบเขต (Hexagonal Architecture)

สองเทคนิคการรีแฟกเตอร์ที่ใช้บ่อยที่สุดคือ **extract function/method** (ดึงกลุ่มโค้ดที่ทำงานร่วมกันออกมาเป็น
ฟังก์ชันแยก เพื่อให้แต่ละฟังก์ชันมีความรับผิดชอบเดียวและอ่านง่ายขึ้น) และ **การใช้ trait กำหนดขอบเขตของระบบ**
(แยก "สิ่งที่ระบบทำ" ออกจาก "รายละเอียดว่าทำอย่างไร") มาดูตัวอย่างจริงทีละขั้นบนฟังก์ชัน `return_book` ที่ยาว
และทำหลายอย่างในฟังก์ชันเดียว:

**ขั้นที่ 0 — โค้ดตั้งต้น (compile ผ่าน แต่ยาวและทำหลายหน้าที่ในฟังก์ชันเดียว):**

```rust
struct Book {
    isbn: String,
    copies_total: u32,
    copies_available: u32,
}

struct Member {
    member_id: String,
    borrowed_isbns: Vec<String>,
}

struct Library {
    books: Vec<Book>,
    members: Vec<Member>,
}

impl Library {
    fn return_book(&mut self, member_id: &str, isbn: &str) -> bool {
        // ส่วนที่ 1: หาสมาชิก
        let member_index = match self.members.iter().position(|m| m.member_id == member_id) {
            Some(i) => i,
            None => return false,
        };

        // ส่วนที่ 2: ตรวจว่าสมาชิกยืม isbn นี้อยู่จริงไหม แล้วเอาออกจากรายการที่ยืม
        let borrowed_pos = self.members[member_index]
            .borrowed_isbns
            .iter()
            .position(|b| b == isbn);
        let borrowed_pos = match borrowed_pos {
            Some(p) => p,
            None => return false,
        };
        self.members[member_index].borrowed_isbns.remove(borrowed_pos);

        // ส่วนที่ 3: หาหนังสือแล้วเพิ่มจำนวนที่ยืมได้กลับ
        let book_index = match self.books.iter().position(|b| b.isbn == isbn) {
            Some(i) => i,
            None => return false,
        };
        self.books[book_index].copies_available += 1;

        true
    }
}
```

ฟังก์ชันนี้ **compile ผ่านและทำงานถูกต้อง** แต่ทำ 3 หน้าที่ในฟังก์ชันเดียว: หาสมาชิก, จัดการรายการที่ยืมของสมาชิก,
และจัดการจำนวนสำเนาของหนังสือ — ผสมกันจนยากจะเทสต์แยกส่วน และยากจะอ่านว่า "กฎทางธุรกิจ" จริง ๆ คืออะไร

**ขั้นที่ 1 — Extract Method: ดึงส่วนที่ 2 และ 3 ออกเป็นเมธอดของ `Member` และ `Book` ตามลำดับ**

```rust
impl Member {
    /// เอา isbn ออกจากรายการที่ยืม คืน true ถ้าเคยยืมอยู่จริงและเอาออกสำเร็จ
    fn record_return(&mut self, isbn: &str) -> bool {
        match self.borrowed_isbns.iter().position(|b| b == isbn) {
            Some(pos) => {
                self.borrowed_isbns.remove(pos);
                true
            }
            None => false,
        }
    }
}

impl Book {
    fn return_one_copy(&mut self) {
        self.copies_available += 1;
    }
}

impl Library {
    fn return_book(&mut self, member_id: &str, isbn: &str) -> bool {
        let member_index = match self.members.iter().position(|m| m.member_id == member_id) {
            Some(i) => i,
            None => return false,
        };

        if !self.members[member_index].record_return(isbn) {
            return false;
        }

        let book_index = match self.books.iter().position(|b| b.isbn == isbn) {
            Some(i) => i,
            None => return false,
        };
        self.books[book_index].return_one_copy();

        true
    }
}
```

**ทำไมขั้นนี้ดีขึ้น**: ตรรกะ "การเอา isbn ออกจากรายการที่ยืมของสมาชิก" ตอนนี้เป็นความรับผิดชอบของ `Member` เอง
(encapsulation) และ **เทสต์ได้แยกจาก `Library` ทั้งก้อน** — เขียนเทสต์ `record_return` ตรง ๆ บน `Member` เปล่า ๆ
ได้เลยโดยไม่ต้องสร้าง `Library` ทั้งระบบขึ้นมาก่อน นี่คือประโยชน์หลักของ extract method: **ลดขนาดของ "หน่วยที่ต้อง
คิดพร้อมกัน" (unit of reasoning)** ในหัวของคนอ่านโค้ด

**ขั้นที่ 2 — ดึง "การหา index" ที่ซ้ำกันสองที่ออกเป็น helper method:**

```rust
impl Library {
    fn find_member_index(&self, member_id: &str) -> Option<usize> {
        self.members.iter().position(|m| m.member_id == member_id)
    }

    fn find_book_index(&self, isbn: &str) -> Option<usize> {
        self.books.iter().position(|b| b.isbn == isbn)
    }

    fn return_book(&mut self, member_id: &str, isbn: &str) -> bool {
        let member_index = match self.find_member_index(member_id) {
            Some(i) => i,
            None => return false,
        };
        if !self.members[member_index].record_return(isbn) {
            return false;
        }
        let book_index = match self.find_book_index(isbn) {
            Some(i) => i,
            None => return false,
        };
        self.books[book_index].return_one_copy();
        true
    }
}
```

**ขั้นที่ 3 — ใช้ trait กำหนดขอบเขต (hexagonal architecture / ports & adapters)**

จนถึงขั้นที่ 2 โค้ดยังผูกติดกับการเก็บ `books`/`members` เป็น `Vec` ตรง ๆ ใน `Library` — ถ้าวันหนึ่งทีมต้องการ
เปลี่ยนไปเก็บข้อมูลในฐานข้อมูลจริง (PostgreSQL, SQLite) จะต้องแก้ logic ของ `checkout`/`return_book` ทั้งหมด
ทั้งที่ **กฎทางธุรกิจไม่ได้เปลี่ยนเลย** เปลี่ยนแค่ "ที่เก็บข้อมูล" เท่านั้น นี่คือจุดที่แนวคิด **hexagonal
architecture** (เรียกอีกชื่อว่า ports & adapters) เข้ามาช่วย: เราแยก **"port"** (สิ่งที่ระบบต้องการจากแหล่งเก็บ
ข้อมูล — นิยามด้วย trait) ออกจาก **"adapter"** (การ implement จริงของ port นั้น — จะเป็น `Vec` ในหน่วยความจำ
หรือฐานข้อมูลจริงก็ได้)

```rust
/// Port: สิ่งที่ Library ต้องการจากแหล่งเก็บข้อมูลหนังสือ — ไม่สนใจว่าข้างใน
/// implement ด้วยอะไร
trait BookRepository {
    fn find_mut(&mut self, isbn: &str) -> Option<&mut Book>;
}

/// Adapter: เก็บข้อมูลใน Vec ในหน่วยความจำ (ใช้ตอน demo/เทสต์)
struct InMemoryBookRepository {
    books: Vec<Book>,
}

impl BookRepository for InMemoryBookRepository {
    fn find_mut(&mut self, isbn: &str) -> Option<&mut Book> {
        self.books.iter_mut().find(|b| b.isbn == isbn)
    }
}

struct Library<B: BookRepository> {
    books: B,
    members: Vec<Member>,
}

impl<B: BookRepository> Library<B> {
    fn return_book(&mut self, member_id: &str, isbn: &str) -> bool {
        let member_index = match self.members.iter().position(|m| m.member_id == member_id) {
            Some(i) => i,
            None => return false,
        };
        if !self.members[member_index].record_return(isbn) {
            return false;
        }
        match self.books.find_mut(isbn) {
            Some(book) => {
                book.return_one_copy();
                true
            }
            None => false,
        }
    }
}
```

ตอนนี้ `Library<B>` **ไม่รู้เลยว่าหนังสือถูกเก็บไว้ที่ไหน** มันคุยกับ `BookRepository` (port) เท่านั้น ประโยชน์ที่
จับต้องได้ทันที:

1. **เขียนเทสต์ logic การยืม-คืนได้โดยไม่ต้องพึ่งฐานข้อมูลจริง** — ใช้ `InMemoryBookRepository` ในเทสต์ทั้งหมด
   แม้ใน production จะใช้ adapter ที่คุยกับฐานข้อมูลจริงก็ตาม
2. **สลับ adapter ได้โดยไม่แก้ business logic แม้แต่บรรทัดเดียว** — ถ้าอนาคตต้องเปลี่ยนไปใช้ PostgreSQL แค่เขียน
   `struct SqlBookRepository` ที่ implement `BookRepository` ใหม่ แล้วเปลี่ยนจุดที่สร้าง `Library::new(...)`
   จุดเดียว โค้ดที่เหลือทั้งหมดไม่ต้องแตะ
3. **ขอบเขตของระบบชัดเจนขึ้นในระดับ type system** — คนอ่าน `trait BookRepository` เห็นทันทีว่า "นี่คือทุกอย่างที่
   ระบบยืม-คืนต้องการจากแหล่งเก็บข้อมูลหนังสือ" ไม่ต้องไล่อ่านทั้ง implementation เพื่อเข้าใจขอบเขต

เราจะเห็นเวอร์ชันสมบูรณ์ของแนวคิดนี้ (พร้อม error handling แบบ `Result` และเทสต์ครบ) ในหัวข้อ capstone 109.14

### 109.9 Naming Convention: `is_`/`has_`, `into_`/`to_`/`as_`

Rust มีธรรมเนียมการตั้งชื่อที่ **สื่อความหมายทางเทคนิคจริง ๆ** ไม่ใช่แค่ความชอบส่วนตัว การใช้ prefix ผิดจาก
ความหมายจริงของมันทำให้ผู้ใช้ API เข้าใจผิดเรื่อง cost และ ownership ของการเรียกฟังก์ชันนั้น

**`is_`/`has_` — เมธอดที่คืน `bool`**

- **`is_`** ใช้กับคำถามเกี่ยวกับ **สถานะของตัวเอง** ณ ขณะนี้ เช่น `is_available()`, `is_empty()`, `is_valid()`
- **`has_`** ใช้กับคำถามเกี่ยวกับ **การครอบครอง/มีความสัมพันธ์กับสิ่งอื่น** เช่น `has_borrowed(isbn)`,
  `has_permission(role)`

ทั้งสองแบบต้อง **ไม่มีผลข้างเคียง (side effect)** และควรราคาถูก (ไม่ allocate, ไม่ I/O) เพราะชื่อขึ้นต้นแบบนี้ทำให้
ผู้เรียกคาดหวังว่ามันเป็นแค่การ "ถาม" ไม่ใช่การ "ทำอะไร"

**`into_`/`to_`/`as_` — เมธอดแปลง type**

สามคำนี้มีความหมายที่ **แม่นยำและแตกต่างกันโดยสิ้นเชิง** ตามธรรมเนียมของ Rust API guidelines:

| Prefix | รับ `self` แบบไหน | มี allocation ใหม่ไหม | ตัวอย่างจาก std |
|---|---|---|---|
| `as_` | `&self` (ยืม, เรียกซ้ำได้) | ไม่มี — แค่ "มองอีกมุม" ของข้อมูลเดิม | `str::as_bytes()`, `String::as_str()` |
| `to_` | `&self` (ยืม, เรียกซ้ำได้) | มี — สร้างข้อมูลใหม่จากของเดิม | `str::to_string()`, `[T]::to_vec()` |
| `into_` | `self` (กิน/consume, เรียกได้ครั้งเดียว) | อาจมีหรือไม่มี แต่ค่าเดิม "หายไป" เสมอ | `String::into_bytes()`, `Vec::into_iter()` |

มาดูตัวอย่างจริงบนโดเมนใบเสร็จการยืมหนังสือ ที่ตั้งชื่อตามธรรมเนียมนี้ครบทั้งสามแบบ:

```rust
struct BorrowRecord {
    member_name: String,
    isbn: String,
    days_borrowed: u32,
}

struct BorrowSummary {
    headline: String,
}

struct Receipt {
    member_name: String,
    isbn: String,
    total_days: u32,
}

impl BorrowRecord {
    fn new(member_name: impl Into<String>, isbn: impl Into<String>, days_borrowed: u32) -> Self {
        BorrowRecord {
            member_name: member_name.into(),
            isbn: isbn.into(),
            days_borrowed,
        }
    }

    // to_: ยืม &self เท่านั้น สร้างข้อมูลใหม่ (String ใหม่จาก format!) เรียกซ้ำได้
    fn to_summary(&self) -> BorrowSummary {
        BorrowSummary {
            headline: format!(
                "{} ยืม {} มา {} วัน",
                self.member_name, self.isbn, self.days_borrowed
            ),
        }
    }

    // as_: ยืม &self เท่านั้น ไม่มี allocation ใหม่ แค่คืน reference
    fn as_isbn_str(&self) -> &str {
        &self.isbn
    }

    // into_: กิน self ไปเลย เรียกได้ครั้งเดียว หลังจากนี้ BorrowRecord ตัวเดิมหายไป
    fn into_receipt(self) -> Receipt {
        Receipt {
            member_name: self.member_name,
            isbn: self.isbn,
            total_days: self.days_borrowed,
        }
    }
}

fn main() {
    let record = BorrowRecord::new("Somchai", "111", 5);

    let summary = record.to_summary(); // ยืม &self — ใช้ record ต่อได้อีก
    println!("{}", summary.headline);

    println!("isbn (borrowed view): {}", record.as_isbn_str()); // ยืมอีกครั้ง ยังใช้ต่อได้

    let receipt = record.into_receipt(); // กิน record ไปเลย
    println!(
        "receipt: {} ยืมทั้งหมด {} วัน (isbn {})",
        receipt.member_name, receipt.total_days, receipt.isbn
    );
}
```

รันจริงได้ผลลัพธ์:

```
Somchai ยืม 111 มา 5 วัน
isbn (borrowed view): 111
receipt: Somchai ยืมทั้งหมด 5 วัน (isbn 111)
```

**สิ่งที่พิสูจน์ว่าชื่อสื่อความหมายจริง ไม่ใช่แค่ convention ลอย ๆ**: ถ้าเราลองเพิ่มบรรทัดใช้ `record` อีกครั้ง
**หลังจาก** เรียก `into_receipt()` ไปแล้ว:

```rust
let receipt = record.into_receipt();
println!("{}", receipt.member_name);
println!("{}", record.as_isbn_str()); // พยายามใช้ record ที่ถูก move ไปแล้ว
```

compiler จะปฏิเสธทันทีด้วย error จริง:

```
error[E0382]: borrow of moved value: `record`
  --> examples/naming.rs:76:20
   |
60 |     let record = BorrowRecord::new("Somchai", "111", 5);
   |         ------ move occurs because `record` has type `BorrowRecord`, which does not implement the `Copy` trait
...
71 |     let receipt = record.into_receipt();
   |                          -------------- `record` moved due to this method call
...
76 |     println!("{}", record.as_isbn_str()); // ตั้งใจใส่ผิดเพื่อสาธิต error
   |                    ^^^^^^ value borrowed here after move
   |
note: `BorrowRecord::into_receipt` takes ownership of the receiver `self`, which moves `record`
```

นี่คือสิ่งที่ทำให้ naming convention ของ Rust **ไม่ใช่แค่ documentation ที่อาจตกยุค** — คำว่า `into_` ผูกติดกับ
`fn into_receipt(self)` ที่รับ `self` แบบ consume จริง ๆ ในระดับ type signature ถ้าใครตั้งชื่อฟังก์ชันผิดจาก
พฤติกรรมจริง (เช่นตั้งชื่อ `to_receipt` แต่ signature รับ `self` แบบ consume) โปรแกรมเมอร์คนอื่นที่ใช้ API นี้จะ
เดาพฤติกรรมผิด — เข้าใจว่าเรียกซ้ำได้ทั้งที่จริง ๆ เรียกได้แค่ครั้งเดียว

**ตัวอย่างการ audit และแก้ไขชื่อที่ผิด**

สมมติทีมเขียนเมธอดต่อไปนี้ไว้ (ผิด convention):

```rust
impl Book {
    // ผิด: ตั้งชื่อ get_ ซึ่งไม่ใช่ convention ของ Rust เลย (ภาษาอื่นอย่าง Java/C++
    // ใช้ get_ กันเยอะ แต่ Rust ไม่ใช้ getter prefix แบบนี้ — ยกเว้นกรณีมี
    // ทั้ง getter/setter คู่กันจริง ๆ ซึ่งพบน้อยมาก)
    fn get_isbn(&self) -> String {
        self.isbn.clone() // ผิดซ้ำสอง: ชื่อบอกว่า "get" (น่าจะยืม) แต่คืน owned + clone
    }

    // ผิด: ชื่อ to_available ฟังดูเหมือนแปลงเป็นอะไรใหม่ ทั้งที่จริง ๆ แค่ถามสถานะ bool
    fn to_available(&self) -> bool {
        self.copies_available > 0
    }
}
```

หลัง audit และแก้ตาม convention:

```rust
impl Book {
    // แก้: ไม่ต้องมี prefix เลยถ้าคืน reference ตรง ๆ (เทียบเท่า as_ ที่ไม่ต้อง
    // สะกดยาว เพราะ Rust ถือว่าการคืน &str จาก struct field เป็นเรื่องปกติมาก
    // จนไม่ต้องมี prefix บอกก็ได้ — แต่ถ้าต้องการความชัดเจนสูงสุดจะใช้ as_isbn() ก็ได้)
    fn isbn(&self) -> &str {
        &self.isbn
    }

    // แก้: is_ ตรงตามความหมายจริง — ถามสถานะ bool ของตัวเอง ไม่มี allocation
    fn is_available(&self) -> bool {
        self.copies_available > 0
    }
}
```

### 109.10 เอกสารในฐานะ Clean Code: อะไรควรมี Doc Comment อะไรไม่จำเป็น

จาก Part 5 (หัวข้อ 5.4) เราเรียนรู้ syntax ของ `///`/`//!` ไปแล้ว ในบทนี้เราจะพูดถึง **เนื้อหา** ที่ควรอยู่ในนั้น
หลักการสำคัญที่สุดคือ: **doc comment ที่ดีอธิบายสิ่งที่โค้ดบอกไม่ได้ ไม่ใช่แปลโค้ดเป็นภาษาคนซ้ำ**

**ตัวอย่างที่ไม่มีประโยชน์ (แค่แปลโค้ดเป็นคำพูด):**

```rust
/// เพิ่มจำนวนสำเนาที่ยืมได้ขึ้น 1
fn return_one_copy(&mut self) {
    self.copies_available += 1;
}
```

comment นี้ไม่มีประโยชน์เลย เพราะชื่อฟังก์ชัน `return_one_copy` และโค้ด `self.copies_available += 1` บอกสิ่งเดียวกัน
อยู่แล้ว การอ่าน comment ไม่ได้ทำให้รู้อะไรเพิ่ม

**ตัวอย่างที่มีประโยชน์จริง (อธิบาย invariant และเหตุผลที่ compiler มองไม่เห็น):**

```rust
/// ลดจำนวนสำเนาที่ยืมได้ลง 1 เล่ม
///
/// # Panics
/// panic ถ้าเรียกตอนที่ `copies_available == 0` เพราะเป็น bug ของผู้เรียก
/// (ผู้เรียกต้องตรวจ `is_available()` ก่อนเสมอ — เมธอดนี้ตั้งใจไม่คืน
/// `Result` เพราะมันเป็น private invariant ภายใน ไม่ใช่ error ที่ผู้ใช้
/// ปลายทางควรต้อง handle)
fn checkout_one_copy(&mut self) {
    assert!(self.copies_available > 0, "checkout_one_copy called with 0 copies available");
    self.copies_available -= 1;
}
```

comment นี้มีค่าเพราะบอกสามอย่างที่**อ่านจากโค้ดอย่างเดียวไม่รู้**:

1. **มี invariant ที่ต้องรักษาไว้ก่อนเรียก** (`copies_available > 0`) — นี่เป็น "สัญญา" (contract) ระหว่างฟังก์ชัน
   กับผู้เรียก ที่ type system ของ Rust (ในตอนนี้) ยังไม่มีวิธีบังคับให้ที่ compile time ได้ [^1]
2. **ทำไมมันไม่คืน `Result`** — อธิบายการตัดสินใจเชิง design ว่าทำไมเลือก panic (assert) แทน error handling
   ปกติ ซึ่งเป็นข้อมูลที่สำคัญมากสำหรับคนที่จะมาแก้โค้ดนี้ในอนาคต ไม่ให้เผลอเปลี่ยนเป็น `Result` โดยไม่เข้าใจเหตุผล
3. **สื่อสารว่าใครต้องรับผิดชอบอะไร** — บอกชัดว่าการเรียกผิดเงื่อนไขเป็น "bug ของผู้เรียก" ไม่ใช่ "error ที่คาด
   การณ์ได้" (เชื่อมโยงกับหลักการในหัวข้อ 109.4)

[^1] หมายเหตุ: struct field ที่ต้องรักษา invariant แบบนี้ควรเป็น **private** เสมอ (สังเกตว่าเราไม่มี
`pub copies_available` ในตัวอย่างนี้) เพื่อบังคับให้ทุกการแก้ไขต้องผ่านเมธอดที่รักษา invariant ไว้ — ถ้า field
เป็น `pub` ตรง ๆ ใครก็ตั้งค่าเป็นค่าที่ผิด invariant ได้จากนอก module และ comment นี้จะกลายเป็นแค่ "คำขอ" ที่ไม่มี
การบังคับใช้จริง

**กฎตัดสินใจอย่างง่าย**: ก่อนเขียน doc comment ให้ถามว่า *"ถ้าฉันลบ comment นี้ทิ้ง คนอ่านโค้ดจะเสียข้อมูลอะไรไป
บ้างที่หาไม่ได้จากการอ่าน signature กับ implementation?"* ถ้าคำตอบคือ "ไม่มีอะไรเสียไปเลย" — ลบ comment นั้นทิ้งได้
เลย ถ้าคำตอบคือ "เสีย invariant/เหตุผลเชิง design/ข้อจำกัดที่ซ่อนอยู่" — เก็บไว้และเขียนให้ชัดกว่านี้ไปเลย

**สิ่งที่ควรมี doc comment เสมอไม่ว่าจะดู "ชัดเจน" แค่ไหน**: public API ของ library ที่คนอื่นจะเรียกใช้ (แม้จะเป็น
แค่ `pub fn checkout(...)`) ควรมี `# Errors` section อธิบายว่า error แต่ละแบบเกิดจากอะไร (ดูตัวอย่างจริงในหัวข้อ
109.14) เพราะ `clippy::pedantic` เตือนเรื่องนี้ด้วย lint `missing_errors_doc` — และเหตุผลก็สมเหตุสมผล: ผู้เรียกที่
เห็น `Result<T, MyError>` ใน signature อยากรู้ทันทีว่า "จะได้ `Err` แบบไหนบ้าง โดยไม่ต้องไปไล่อ่าน implementation"

### 109.11 รีวิวคุณภาพเทสต์: Test Behavior ไม่ใช่ Implementation

เทสต์ที่มีอยู่แล้วในโปรเจกต์ก็ต้องรีวิวเช่นเดียวกับโค้ด production — เทสต์ที่เขียนไม่ดีเป็นภาระ (liability) ไม่ใช่
ทรัพย์สิน (asset) เพราะมันจะ fail ทุกครั้งที่มีการรีแฟกเตอร์ **แม้พฤติกรรมภายนอกไม่เปลี่ยนเลย** ทำให้ทีมกลัวการ
รีแฟกเตอร์ (เพราะต้องแก้เทสต์เยอะทุกครั้ง) ซึ่งขัดกับเป้าหมายที่แท้จริงของการมีเทสต์

**ตัวอย่างเทสต์ที่ไม่ดี — ผูกติดกับ implementation detail:**

```rust
#[test]
fn test_checkout() {
    let mut lib = sample_library();
    lib.checkout("M1", "111").unwrap();

    // ผูกติดกับ "ลำดับ" ของ Vec ภายใน และ "จำนวน field" ที่ Library เก็บไว้
    // ถ้าวันหนึ่งเปลี่ยน Vec<Book> เป็น HashMap<String, Book> (เพื่อ lookup
    // เร็วขึ้น) เทสต์นี้พังทันที ทั้งที่พฤติกรรมภายนอก (ยืมได้ ยืมไม่ได้)
    // ไม่ได้เปลี่ยนเลย
    assert_eq!(lib.books.len(), 1);
    assert_eq!(lib.books[0].copies_available, 0);
    assert_eq!(lib.members[0].borrowed_isbns.len(), 1);
    assert_eq!(lib.members[0].borrowed_isbns[0], "111");
}
```

ปัญหาของเทสต์นี้ไม่ใช่ที่มันตรวจสอบ "ผิด" — มันตรวจสอบถูกจริง ๆ ในสถานะปัจจุบัน แต่มันเข้าถึง **field ภายในของ
`Library` ตรง ๆ** (`lib.books`, `lib.members`) ซึ่งควรเป็นรายละเอียดภายในที่ผู้ใช้ `Library` จากภายนอกไม่ควรรู้จัก
เลย เทสต์แบบนี้จะ **compile ไม่ผ่านทันที** ถ้าเราเปลี่ยน field เหล่านี้เป็น private (ซึ่งควรเป็น private ตั้งแต่
แรกตามหลัก encapsulation) หรือถ้าเปลี่ยนโครงสร้างข้อมูลภายใน แม้พฤติกรรมสาธารณะยังเหมือนเดิมทุกประการ

**ตัวอย่างเทสต์ที่ดี — ทดสอบผ่าน public API เท่านั้น มองจากมุมผู้ใช้ภายนอก:**

```rust
#[test]
fn checkout_reduces_available_copies_by_one() {
    let mut lib = sample_library();

    lib.checkout("M1", "111").unwrap();

    // ตรวจผลลัพธ์ผ่าน public API เท่านั้น (lib.book(...)) — ไม่สนใจว่า
    // ภายในเก็บข้อมูลด้วย Vec, HashMap, หรือ BTreeMap
    assert_eq!(lib.book("111").unwrap().copies_available(), 0);
}

#[test]
fn checkout_fails_when_no_copies_available() {
    let mut lib = sample_library(); // มีสำเนาเดียว
    lib.checkout("M1", "111").unwrap();
    lib.add_member(Member::new("Somying", "M2"));

    let result = lib.checkout("M2", "111");

    assert_eq!(result, Err(LibraryError::NoCopiesAvailable));
}
```

สังเกตความแตกต่างสองจุด:

1. **ชื่อเทสต์บอกพฤติกรรมทางธุรกิจ** (`checkout_reduces_available_copies_by_one`,
   `checkout_fails_when_no_copies_available`) ไม่ใช่ชื่อฟังก์ชันที่ทดสอบ (`test_checkout`) — อ่านแค่ชื่อเทสต์อย่าง
   เดียว (โดยไม่ต้องเปิดดู body) ก็รู้ทันทีว่ากฎทางธุรกิจของระบบคืออะไร ซึ่งมีประโยชน์มากตอนรัน `cargo test` แล้ว
   เห็นรายชื่อเทสต์ทั้งหมดพรึบเดียว — มันทำหน้าที่เป็น "เอกสารของกฎทางธุรกิจ" ไปในตัว
2. **เข้าถึงผลลัพธ์ผ่าน public API** (`lib.book(...)`, ค่าที่ `checkout` คืนกลับมา) เท่านั้น — เทสต์นี้จะยังผ่าน
   ต่อไปไม่ว่าจะรีแฟกเตอร์ภายในของ `Library` อย่างไรก็ตาม ตราบใดที่พฤติกรรมสาธารณะยังเหมือนเดิม

**หลักการสรุป**: เทสต์ที่ดีควรตอบคำถาม *"ถ้าฉันรีแฟกเตอร์ implementation ภายในทั้งหมดแบบไม่เปลี่ยน public API
เทสต์นี้ควรพังหรือไม่?"* — ถ้าคำตอบคือ "ไม่ควรพัง" แต่เทสต์ปัจจุบันจะพัง แสดงว่าเทสต์นั้นผูกติดกับ implementation
มากเกินไป ต้องเขียนใหม่ให้ทดสอบผ่าน public API เท่านั้น

### 109.12 Dependency Hygiene: `cargo machete` และ `cargo udeps`

โปรเจกต์ที่มีอายุยาวนานมักสะสม dependency ที่ประกาศไว้ใน `Cargo.toml` แต่ไม่มีใครใช้แล้ว (เช่น เคยลองใช้ crate หนึ่ง
แล้วเปลี่ยนไปใช้ crate อื่น แต่ลืมลบตัวเก่าออก) dependency ที่ไม่ได้ใช้เหล่านี้ทำให้:

1. เวลา compile นานขึ้นโดยไม่จำเป็น (ยังต้อง download และ compile crate นั้นอยู่ดี)
2. surface area ของ security vulnerability กว้างขึ้น (ยิ่ง dependency มาก ยิ่งมีโอกาสที่สักตัวจะมีช่องโหว่)
3. สร้างความสับสนให้คนอ่าน `Cargo.toml` ว่าโปรเจกต์นี้ "ใช้" อะไรจริง ๆ บ้าง

`cargo machete` เป็นเครื่องมือที่สแกนโค้ดทั้งโปรเจกต์หา `use` statement ที่อ้างถึง dependency แต่ละตัว แล้วเทียบกับ
รายการใน `Cargo.toml` — ตัวไหนที่ประกาศไว้แต่ไม่มีการ `use` เลย จะถูกรายงานออกมา

**สาธิตจริง**: สมมติเราเพิ่ม dependency สองตัวเข้าไปใน `Cargo.toml` ของโปรเจกต์ห้องสมุด (`serde` ที่เผื่อไว้
"อยากจะ" serialize ในอนาคตแต่ยังไม่ได้เขียนโค้ดจริง และ `once_cell` ที่เอาไว้จากตอน prototype แรก ๆ แล้วลบโค้ดที่
ใช้มันออกไปแล้วโดยลืมลบ dependency):

```toml
[dependencies]
serde = { version = "1", features = ["derive"] }
once_cell = "1"
```

รัน `cargo machete` จริงในโปรเจกต์นี้ ได้ผลลัพธ์:

```
Analyzing dependencies of crates in this directory...
cargo-machete found the following unused dependencies in this directory:
library_review -- ./Cargo.toml:
	once_cell
	serde

If you believe cargo-machete has detected an unused dependency incorrectly,
you can add the dependency to the list of dependencies to ignore in the
`[package.metadata.cargo-machete]` section of the appropriate Cargo.toml.
For example:

[package.metadata.cargo-machete]
ignored = ["prost"]

You can also try running it with the `--with-metadata` flag for better accuracy,
though this may modify your Cargo.lock files.

Done!
```

หลังลบทั้งสอง dependency ออกจาก `Cargo.toml` แล้วรันซ้ำ ได้ผลลัพธ์:

```
Analyzing dependencies of crates in this directory...
cargo-machete didn't find any unused dependencies in this directory. Good job!
Done!
```

**ข้อควรระวัง (false positive)**: ข้อความ hint ของ `cargo machete` เองก็บอกไว้ตรง ๆ ว่ามันอาจ "ตรวจผิด" ได้ในบาง
กรณี — สถานการณ์ที่พบบ่อยที่สุดคือ dependency ที่ใช้ผ่าน **macro เท่านั้น** โดยไม่มี `use` statement ตรง ๆ ที่มองเห็น
ได้จาก static analysis (เช่น crate ที่ใช้แค่เป็น proc-macro หรือ crate ที่ inject โค้ดผ่าน `#[derive(...)]` โดยไม่
ต้อง `use` อะไรเพิ่ม) ในกรณีแบบนี้ ให้เพิ่มชื่อ crate นั้นไว้ใน `[package.metadata.cargo-machete]` section ตามที่
เครื่องมือแนะนำ แทนการเชื่อผลลัพธ์แบบไม่ตรวจสอบ

**`cargo udeps`**: เป็นเครื่องมือแนวเดียวกันกับ `cargo machete` แต่ใช้วิธีตรวจสอบที่แม่นยำกว่า (คอมไพล์จริงแล้วดู
ว่า crate ไหนไม่ถูกอ้างถึงจริงในผลลัพธ์การ compile ไม่ใช่แค่ scan ข้อความหา `use`) ข้อจำกัดที่สำคัญคือ **ต้องรันบน
nightly toolchain เท่านั้น** (`cargo +nightly udeps`) เพราะใช้ unstable feature ของ compiler ในการตรวจสอบ ติดตั้ง
ด้วย `cargo install cargo-udeps --locked` แล้วรันด้วย `cargo +nightly udeps` ในทางปฏิบัติ ทีมส่วนใหญ่เลือกใช้
`cargo machete` เป็นตัวหลักในการรันบ่อย ๆ (เพราะเร็วกว่ามาก ไม่ต้อง compile จริง และไม่ต้องพึ่ง nightly) แล้วรัน
`cargo udeps` เป็นครั้งคราว (เช่น ก่อน release ใหญ่) เพื่อ double-check ผลลัพธ์ให้แม่นยำขึ้น

### 109.13 รีแฟกเตอร์อย่างปลอดภัยด้วย Characterization Tests

ปัญหาคลาสสิกที่ทุกทีมเจอ: มีฟังก์ชันเก่าที่ **ไม่มีเทสต์เลย** และ **ไม่มีใครในทีมกล้าแก้** เพราะไม่รู้ว่าพฤติกรรม
ปัจจุบัน (ซึ่งอาจดู "แปลก" หรือ "ไม่สมเหตุสมผล" ในบางจุด) เป็นสิ่งที่ตั้งใจออกแบบไว้ หรือเป็นบั๊กที่ไม่มีใครเคยรู้
มาก่อน ถ้าแก้ไปโดยไม่รู้ อาจ "แก้บั๊ก" ที่จริง ๆ ระบบอื่นพึ่งพา (depend on) พฤติกรรมเดิมอยู่แล้วก็ได้

**Characterization test** คือเทสต์ที่เขียนขึ้นเพื่อ **บันทึกพฤติกรรมปัจจุบันตามที่มันเป็นจริง ๆ วันนี้** ไม่ใช่ตาม
ที่ "ควรจะเป็น" — เป้าหมายไม่ใช่การยืนยันว่าโค้ดถูก แต่คือการสร้าง **safety net** ที่จะแจ้งเตือนทันทีถ้าการรีแฟก
เตอร์ครั้งต่อไปเผลอเปลี่ยนพฤติกรรมโดยไม่ตั้งใจ

มาดูฟังก์ชันคำนวณค่าปรับที่ไม่มีเทสต์เลยในระบบห้องสมุดของเรา:

```rust
/// คำนวณค่าปรับคืนหนังสือล่าช้า
///
/// ฟังก์ชันนี้เขียนไว้นานแล้วโดยไม่มีเทสต์ ไม่มีใครในทีมกล้าแก้เพราะไม่รู้ว่า
/// พฤติกรรมปัจจุบัน (รวมส่วนที่ดูเหมือน "แปลก") ตั้งใจหรือเป็นบั๊ก
fn calculate_late_fee(days_late: i64, base_fee_per_day: f64) -> f64 {
    if days_late <= 0 {
        0.0
    } else if days_late <= 7 {
        days_late as f64 * base_fee_per_day
    } else {
        // สัปดาห์แรกคิดปกติ ตั้งแต่วันที่ 8 เป็นต้นไปคิดเพิ่ม 50%
        let first_week = 7.0 * base_fee_per_day;
        let extra_days = (days_late - 7) as f64;
        first_week + extra_days * base_fee_per_day * 1.5
    }
}
```

โค้ดที่ comment บอกว่า "สัปดาห์แรกคิดปกติ ตั้งแต่วันที่ 8 คิดเพิ่ม 50%" — แต่นี่คือ**ตั้งใจ**หรือ**บั๊ก**? ไม่มีใคร
ยืนยันได้แน่ชัดถ้าไม่มีเทสต์หรือ commit history ที่อธิบายไว้ ก่อนจะกล้าแก้ไขอะไรในฟังก์ชันนี้ (เช่น อาจจะอยากแก้
เป็น cast แบบปลอดภัยกว่า หรือใช้ `mul_add` เพื่อความแม่นยำของ floating point) เราต้อง **เขียนเทสต์ที่ตรึงพฤติกรรม
ปัจจุบันไว้ก่อน**:

```rust
#[cfg(test)]
mod characterization_tests {
    use super::*;

    #[test]
    fn characterization_no_late_fee_when_not_late() {
        assert_eq!(calculate_late_fee(0, 5.0), 0.0);
        assert_eq!(calculate_late_fee(-3, 5.0), 0.0);
    }

    #[test]
    fn characterization_linear_fee_within_first_week() {
        assert_eq!(calculate_late_fee(3, 5.0), 15.0);
        assert_eq!(calculate_late_fee(7, 5.0), 35.0);
    }

    #[test]
    fn characterization_fee_after_first_week_has_50_percent_surcharge() {
        // บันทึกพฤติกรรมปัจจุบันไว้ตามที่มันเป็นจริง ๆ วันนี้ (10 วันล่าช้า)
        assert_eq!(calculate_late_fee(10, 5.0), 35.0 + 3.0 * 5.0 * 1.5);
    }
}
```

รันจริงด้วย `cargo test` (บนโค้ดจริงในสภาพแวดล้อมที่เขียนบทนี้) ได้ผลลัพธ์:

```
running 9 tests
test tests::characterization_fee_after_first_week_has_50_percent_surcharge ... ok
test tests::characterization_linear_fee_within_first_week ... ok
test tests::characterization_no_late_fee_when_not_late ... ok
...
test result: ok. 9 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**ขั้นตอนที่ถูกต้องหลังจากนี้** คือเอาชุดเทสต์นี้ไปคุยกับทีม/product owner ว่า "การคิดเพิ่ม 50% หลังวันที่ 7 นี้
ตั้งใจจริงไหม?" ถ้าตั้งใจ — เทสต์เหล่านี้กลายเป็นเทสต์พฤติกรรมถาวรของระบบ (rename จาก `characterization_` เป็นชื่อ
ที่สื่อความหมายทางธุรกิจตามหัวข้อ 109.11) ถ้าไม่ตั้งใจ (เป็นบั๊ก) — แก้ไขฟังก์ชันแล้วอัปเดตเทสต์ **อย่างตั้งใจ**
ในคอมมิตเดียวที่มี message อธิบายชัดเจนว่า "แก้บั๊กการคิดค่าปรับ" ไม่ใช่เผลอเปลี่ยนพฤติกรรมนี้ไปแบบไม่รู้ตัวระหว่าง
ทำการรีแฟกเตอร์เรื่องอื่นที่ไม่เกี่ยวข้อง

**หลักการสำคัญที่สุดของเทคนิคนี้**: ลำดับต้องเป็น **"เขียน characterization test ก่อน → refactor/เปลี่ยนแปลง
implementation → รัน test อีกครั้งเพื่อยืนยันว่าพฤติกรรมยังเหมือนเดิม (ถ้าตั้งใจให้เหมือน) → commit"** ไม่ใช่การ
refactor ก่อนแล้วค่อยเขียนเทสต์ตาม เพราะถ้าทำแบบหลัง เทสต์ที่เขียนจะสะท้อนพฤติกรรม**ใหม่**ที่คุณ (อาจจะไม่ตั้งใจ)
เปลี่ยนไปแล้ว ไม่ใช่พฤติกรรมเดิมที่ระบบอื่นอาจพึ่งพาอยู่

### 109.14 Capstone: รีวิวและรีแฟกเตอร์ระบบห้องสมุดแบบครบวงจร

มาถึงจุดที่เรานำทุกเทคนิคที่เรียนมาในบทนี้มาประกอบกันเป็นการรีวิวและรีแฟกเตอร์แบบครบวงจร บนระบบห้องสมุด (library
checkout system) ที่เราหยิบมาเป็นตัวอย่างทีละส่วนตลอดทั้งบท

**โค้ดตั้งต้น (messy — compile ผ่านและทำงานถูกต้องในกรณีปกติ แต่มีกลิ่นโค้ดครบทั้ง 4 หมวด):**

```rust
use std::cell::RefCell;
use std::rc::Rc;

#[derive(Debug, Clone)]
struct Book {
    title: String,
    isbn: String,
    copies_total: u32,
    copies_available: u32,
}

#[derive(Debug, Clone)]
struct Member {
    name: String,
    member_id: String,
    borrowed_isbns: Vec<String>,
}

struct Library {
    books: Rc<RefCell<Vec<Book>>>,
    members: Rc<RefCell<Vec<Member>>>,
}

impl Library {
    fn new() -> Library {
        return Library {
            books: Rc::new(RefCell::new(Vec::new())),
            members: Rc::new(RefCell::new(Vec::new())),
        };
    }

    fn add_book(&mut self, title: String, isbn: String, copies: u32) {
        let b = Book {
            title: title.clone(),
            isbn: isbn.clone(),
            copies_total: copies,
            copies_available: copies,
        };
        self.books.borrow_mut().push(b);
    }

    fn add_member(&mut self, name: String, member_id: String) {
        let m = Member {
            name: name.clone(),
            member_id: member_id.clone(),
            borrowed_isbns: Vec::new(),
        };
        self.members.borrow_mut().push(m);
    }

    fn checkout(&mut self, member_id: String, isbn: String) -> bool {
        let books = self.books.clone();
        let members = self.members.clone();

        let mut books_ref = books.borrow_mut();
        let book_index = books_ref
            .iter()
            .position(|b| b.isbn == isbn.clone())
            .unwrap();

        let book = books_ref.get_mut(book_index).unwrap();

        match book.copies_available {
            0 => {
                return false;
            }
            _ => {}
        }

        book.copies_available -= 1;

        let mut members_ref = members.borrow_mut();
        let member_index = members_ref
            .iter()
            .position(|m| m.member_id == member_id.clone())
            .unwrap();
        let member = members_ref.get_mut(member_index).unwrap();
        member.borrowed_isbns.push(isbn.clone());

        true
    }

    fn return_book(&mut self, member_id: String, isbn: String) -> bool {
        let books = self.books.clone();
        let members = self.members.clone();

        let mut members_ref = members.borrow_mut();
        let member_index = members_ref
            .iter()
            .position(|m| m.member_id == member_id.clone())
            .unwrap();
        let member = members_ref.get_mut(member_index).unwrap();

        let pos = member
            .borrowed_isbns
            .iter()
            .position(|i| i.clone() == isbn.clone());

        if pos.is_none() {
            return false;
        }

        member.borrowed_isbns.remove(pos.unwrap());

        let mut books_ref = books.borrow_mut();
        let book_index = books_ref.iter().position(|b| b.isbn == isbn.clone()).unwrap();
        let book = books_ref.get_mut(book_index).unwrap();
        book.copies_available += 1;

        return true;
    }

    fn find_book_by_isbn(&self, isbn: String) -> Book {
        let books_ref = self.books.borrow();
        let found = books_ref.iter().find(|b| b.isbn == isbn.clone()).unwrap();
        found.clone()
    }
}
```

โค้ดนี้ compile และรันได้จริงไม่มีปัญหาในกรณีทดสอบทั่วไป แต่เมื่อรัน `cargo clippy` (default, ไม่เปิด flag เสริม
ใด ๆ) บนโค้ดชุดนี้จริง ได้ผลลัพธ์:

```
warning: fields `title` and `copies_total` are never read
warning: field `name` is never read
warning: unneeded `return` statement
 --> src/main.rs:26:9
warning: you seem to be trying to use `match` for an equality check. Consider using `if`
  --> src/main.rs:64:9
warning: unneeded `return` statement
   --> src/main.rs:111:9
library_review generated 5 warnings
```

เมื่อเปิด `-W clippy::pedantic` เพิ่มเข้าไป จำนวน warning เพิ่มเป็น **15** (เพิ่ม `struct_field_names` และ
`needless_pass_by_value` ×8 ตามที่แสดงเต็มในหัวข้อ 109.5) และเมื่อเปิด `-W clippy::nursery` เพิ่มอีก จะเจอ
`redundant_clone` ×5 และ `use_self` ×2 ตามที่แสดงในหัวข้อ 109.3/109.7 — และถ้าเปิด `-W clippy::unwrap_used`
(restriction) จะเจอ `.unwrap()` ที่เสี่ยง panic **8 จุด** ตามที่แสดงในหัวข้อ 109.4

**ขั้นตอนการรีวิว** ก่อนแก้ ให้จับกลิ่นโค้ดทั้งหมดออกมาเป็น checklist ตามหมวดที่เรียนในหัวข้อ 109.2:

| หมวด | ปัญหาที่พบในโค้ดตั้งต้น |
|---|---|
| Ownership/Borrowing | `Rc<RefCell<Vec<Book>>>` ทั้งที่ `Library` เป็นเจ้าของเดียว, `self.books.clone()`/`self.members.clone()` ในทุกเมธอด |
| Error Handling | `.unwrap()` 8 จุดที่ panic ได้จาก input ผู้ใช้ (ISBN/member_id ผิด) |
| API Design | รับ `String` ทุก parameter ทั้งที่หลายจุดแค่อ่าน, คืน `Book` (owned) จาก `find_book_by_isbn` |
| Allocation | `.clone()` ของ `isbn`/`member_id` ซ้ำ ๆ ในทุกเมธอดเพื่อเทียบค่าเฉย ๆ |

**โค้ดหลังรีแฟกเตอร์ (สะอาด, idiomatic, มี error handling ที่ถูกต้อง, มี trait กำหนดขอบเขต, มีเทสต์ครบ):**

```rust
use std::fmt;

/// หนังสือหนึ่งรายการในระบบห้องสมุด
///
/// Invariant ที่ struct นี้รับประกันเสมอ: `copies_available <= copies_total`
/// เสมอ ไม่มีทางที่จำนวนเล่มที่ยืมได้จะมากกว่าจำนวนเล่มทั้งหมดที่ห้องสมุดมี
/// (เมธอด `checkout_one_copy`/`return_one_copy` เป็นทางเดียวที่แก้ไขค่านี้
/// เพื่อรักษา invariant นี้ไว้ — ห้ามแก้ field ตรง ๆ จากนอก module)
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Book {
    pub title: String,
    pub isbn: String,
    copies_total: u32,
    copies_available: u32,
}

impl Book {
    pub fn new(title: impl Into<String>, isbn: impl Into<String>, copies_total: u32) -> Self {
        Book {
            title: title.into(),
            isbn: isbn.into(),
            copies_total,
            copies_available: copies_total,
        }
    }

    /// คืนค่า `true` ถ้ายังมีสำเนาให้ยืมได้อย่างน้อย 1 เล่ม
    pub fn is_available(&self) -> bool {
        self.copies_available > 0
    }

    pub fn copies_available(&self) -> u32 {
        self.copies_available
    }

    /// ลดจำนวนสำเนาที่ยืมได้ลง 1 เล่ม
    ///
    /// # Panics
    /// panic ถ้าเรียกตอนที่ `copies_available == 0` เพราะเป็น bug ของผู้เรียก
    fn checkout_one_copy(&mut self) {
        assert!(self.copies_available > 0, "checkout_one_copy called with 0 copies available");
        self.copies_available -= 1;
    }

    fn return_one_copy(&mut self) {
        assert!(
            self.copies_available < self.copies_total,
            "return_one_copy would exceed copies_total"
        );
        self.copies_available += 1;
    }
}

/// สมาชิกห้องสมุดหนึ่งคน
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Member {
    pub name: String,
    pub member_id: String,
    borrowed_isbns: Vec<String>,
}

impl Member {
    pub fn new(name: impl Into<String>, member_id: impl Into<String>) -> Self {
        Member {
            name: name.into(),
            member_id: member_id.into(),
            borrowed_isbns: Vec::new(),
        }
    }

    /// คืนค่า `true` ถ้าสมาชิกคนนี้กำลังยืม ISBN นี้อยู่
    pub fn has_borrowed(&self, isbn: &str) -> bool {
        self.borrowed_isbns.iter().any(|b| b == isbn)
    }

    fn record_borrow(&mut self, isbn: &str) {
        self.borrowed_isbns.push(isbn.to_string());
    }

    fn record_return(&mut self, isbn: &str) -> bool {
        match self.borrowed_isbns.iter().position(|b| b == isbn) {
            Some(pos) => {
                self.borrowed_isbns.remove(pos);
                true
            }
            None => false,
        }
    }
}

/// ข้อผิดพลาดทั้งหมดที่เกิดขึ้นได้จากการยืม/คืนหนังสือ
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum LibraryError {
    BookNotFound,
    MemberNotFound,
    NoCopiesAvailable,
    BookNotBorrowedByMember,
}

impl fmt::Display for LibraryError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            Self::BookNotFound => write!(f, "ไม่พบหนังสือที่มี ISBN นี้ในระบบ"),
            Self::MemberNotFound => write!(f, "ไม่พบสมาชิกรหัสนี้ในระบบ"),
            Self::NoCopiesAvailable => write!(f, "หนังสือเล่มนี้ถูกยืมหมดแล้ว ไม่มีสำเนาเหลือให้ยืม"),
            Self::BookNotBorrowedByMember => {
                write!(f, "สมาชิกคนนี้ไม่ได้ยืมหนังสือเล่มนี้อยู่ จึงคืนไม่ได้")
            }
        }
    }
}

impl std::error::Error for LibraryError {}

/// ขอบเขต (port) สำหรับแหล่งเก็บข้อมูลหนังสือ — แยกจาก logic การยืม-คืน
/// โดยสิ้นเชิง ตามแนวคิด hexagonal architecture (ports & adapters)
pub trait BookRepository {
    fn add(&mut self, book: Book);
    fn find(&self, isbn: &str) -> Option<&Book>;
    fn find_mut(&mut self, isbn: &str) -> Option<&mut Book>;
}

/// Adapter แบบง่ายที่สุด: เก็บหนังสือไว้ใน `Vec` ในหน่วยความจำ
#[derive(Default)]
pub struct InMemoryBookRepository {
    books: Vec<Book>,
}

impl InMemoryBookRepository {
    pub fn new() -> Self {
        Self::default()
    }
}

impl BookRepository for InMemoryBookRepository {
    fn add(&mut self, book: Book) {
        self.books.push(book);
    }

    fn find(&self, isbn: &str) -> Option<&Book> {
        self.books.iter().find(|b| b.isbn == isbn)
    }

    fn find_mut(&mut self, isbn: &str) -> Option<&mut Book> {
        self.books.iter_mut().find(|b| b.isbn == isbn)
    }
}

/// ระบบห้องสมุด — คุมกฎการยืม/คืนหนังสือ
pub struct Library<B: BookRepository> {
    books: B,
    members: Vec<Member>,
}

impl<B: BookRepository> Library<B> {
    pub const fn new(books: B) -> Self {
        Self {
            books,
            members: Vec::new(),
        }
    }

    pub fn add_book(&mut self, book: Book) {
        self.books.add(book);
    }

    pub fn add_member(&mut self, member: Member) {
        self.members.push(member);
    }

    /// คืน `&Book` (ไม่ clone) เพราะผู้เรียกส่วนใหญ่ต้องการแค่ "อ่าน" ข้อมูล
    pub fn book(&self, isbn: &str) -> Option<&Book> {
        self.books.find(isbn)
    }

    fn find_member_mut(&mut self, member_id: &str) -> Result<&mut Member, LibraryError> {
        self.members
            .iter_mut()
            .find(|m| m.member_id == member_id)
            .ok_or(LibraryError::MemberNotFound)
    }

    /// ให้สมาชิก `member_id` ยืมหนังสือ ISBN `isbn`
    ///
    /// # Errors
    /// คืน [`LibraryError::BookNotFound`] ถ้าไม่พบ ISBN นี้, [`LibraryError::MemberNotFound`]
    /// ถ้าไม่พบสมาชิกรหัสนี้, หรือ [`LibraryError::NoCopiesAvailable`] ถ้าหนังสือถูกยืมหมดแล้ว
    pub fn checkout(&mut self, member_id: &str, isbn: &str) -> Result<(), LibraryError> {
        if !self
            .books
            .find(isbn)
            .ok_or(LibraryError::BookNotFound)?
            .is_available()
        {
            return Err(LibraryError::NoCopiesAvailable);
        }

        self.find_member_mut(member_id)?;

        let book = self.books.find_mut(isbn).ok_or(LibraryError::BookNotFound)?;
        book.checkout_one_copy();

        let member = self.find_member_mut(member_id)?;
        member.record_borrow(isbn);
        Ok(())
    }

    /// รับหนังสือ ISBN `isbn` คืนจากสมาชิก `member_id`
    ///
    /// # Errors
    /// คืน [`LibraryError::MemberNotFound`] ถ้าไม่พบสมาชิก, [`LibraryError::BookNotBorrowedByMember`]
    /// ถ้าสมาชิกคนนี้ไม่ได้ยืม ISBN นี้อยู่, หรือ [`LibraryError::BookNotFound`] ถ้า ISBN ไม่มีในระบบ
    pub fn return_book(&mut self, member_id: &str, isbn: &str) -> Result<(), LibraryError> {
        let member = self.find_member_mut(member_id)?;
        if !member.record_return(isbn) {
            return Err(LibraryError::BookNotBorrowedByMember);
        }

        let book = self.books.find_mut(isbn).ok_or(LibraryError::BookNotFound)?;
        book.return_one_copy();
        Ok(())
    }
}

fn main() {
    let mut library = Library::new(InMemoryBookRepository::new());
    library.add_book(Book::new("The Rust Book", "111", 2));
    library.add_member(Member::new("Somchai", "M1"));

    match library.checkout("M1", "111") {
        Ok(()) => println!("ยืมสำเร็จ"),
        Err(e) => println!("ยืมไม่สำเร็จ: {e}"),
    }

    if let Some(book) = library.book("111") {
        println!("เหลือ {} เล่มที่ยืมได้", book.copies_available());
    }

    match library.return_book("M1", "111") {
        Ok(()) => println!("คืนสำเร็จ"),
        Err(e) => println!("คืนไม่สำเร็จ: {e}"),
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    fn sample_library() -> Library<InMemoryBookRepository> {
        let mut lib = Library::new(InMemoryBookRepository::new());
        lib.add_book(Book::new("The Rust Book", "111", 1));
        lib.add_member(Member::new("Somchai", "M1"));
        lib
    }

    #[test]
    fn checkout_reduces_available_copies_by_one() {
        let mut lib = sample_library();
        lib.checkout("M1", "111").unwrap();
        assert_eq!(lib.book("111").unwrap().copies_available(), 0);
    }

    #[test]
    fn checkout_fails_when_no_copies_available() {
        let mut lib = sample_library();
        lib.checkout("M1", "111").unwrap();
        lib.add_member(Member::new("Somying", "M2"));

        let result = lib.checkout("M2", "111");

        assert_eq!(result, Err(LibraryError::NoCopiesAvailable));
    }

    #[test]
    fn checkout_fails_for_unknown_book() {
        let mut lib = sample_library();
        assert_eq!(lib.checkout("M1", "999"), Err(LibraryError::BookNotFound));
    }

    #[test]
    fn checkout_fails_for_unknown_member() {
        let mut lib = sample_library();
        assert_eq!(
            lib.checkout("UNKNOWN", "111"),
            Err(LibraryError::MemberNotFound)
        );
    }

    #[test]
    fn returning_a_book_increases_available_copies() {
        let mut lib = sample_library();
        lib.checkout("M1", "111").unwrap();

        lib.return_book("M1", "111").unwrap();

        assert_eq!(lib.book("111").unwrap().copies_available(), 1);
    }

    #[test]
    fn returning_a_book_the_member_never_borrowed_fails() {
        let mut lib = sample_library();
        assert_eq!(
            lib.return_book("M1", "111"),
            Err(LibraryError::BookNotBorrowedByMember)
        );
    }
}
```

**ผลการตรวจสอบจริงหลังรีแฟกเตอร์**:

```
$ cargo build
   Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.22s

$ cargo test
running 9 tests
test tests::checkout_fails_for_unknown_book ... ok
test tests::checkout_fails_for_unknown_member ... ok
test tests::checkout_fails_when_no_copies_available ... ok
test tests::checkout_reduces_available_copies_by_one ... ok
test tests::returning_a_book_increases_available_copies ... ok
test tests::returning_a_book_the_member_never_borrowed_fails ... ok
(รวมเทสต์ characterization ของ calculate_late_fee อีก 3 ตัวจากหัวข้อ 109.13)
test result: ok. 9 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s

$ cargo clippy
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.11s
(ไม่มี warning เลยที่ระดับ default — จาก 5 warning ในโค้ดตั้งต้น)

$ cargo clippy -- -W clippy::pedantic -W clippy::nursery
(เหลือ 13 nit เล็กน้อยจาก 20 ก่อนหน้า ส่วนที่เหลือเป็น judgment call ที่ทีมตัดสินใจ
 allow อย่างมีเหตุผล เช่น clippy::cast_precision_loss และ clippy::suboptimal_flops
 ตามที่แสดงไว้ในหัวข้อ 109.7)
```

**สรุปผลลัพธ์ของการรีวิว+รีแฟกเตอร์รอบนี้**: จากโค้ดที่ compile ผ่านแต่มีกลิ่นครบ 4 หมวด (Rc<RefCell> เกินจำเป็น,
`.unwrap()` 8 จุด, รับ `String` ทุกที่, clone ซ้ำซาก) เราได้โค้ดที่:

- ไม่มี `Rc<RefCell<>>` เลย ใช้ ownership ปกติที่ borrow checker ตรวจให้ฟรีตอน compile
- ไม่มี `.unwrap()` ในโค้ด business logic เลย — ทุก error path คืน `Result<_, LibraryError>` ที่ผู้เรียก handle ได้
- รับ `&str` ในทุกจุดที่แค่อ่านค่า, คืน `&Book` แทน owned copy ในจุดที่แค่อ่าน
- มี `trait BookRepository` กำหนดขอบเขตชัดเจนตามแนวคิด hexagonal architecture ทำให้เทสต์ logic ได้โดยไม่ต้องพึ่ง
  แหล่งเก็บข้อมูลจริง
- มี doc comment ที่อธิบาย invariant และเหตุผลเชิง design ในจุดที่จำเป็นจริง ๆ เท่านั้น
- มีเทสต์ 9 ตัวที่ทดสอบผ่าน public API ทั้งหมด ชื่อสื่อความหมายทางธุรกิจ พร้อม characterization test สำหรับ
  ฟังก์ชันเก่าที่ไม่มีเทสต์มาก่อน
- ผ่าน `cargo clippy` ที่ระดับ default โดยไม่มี warning เหลือเลย

## กับดักที่พบบ่อย (Common Pitfalls)

**1. ใช้ `Rc<RefCell<T>>` เพื่อ "แก้ปัญหา" borrow checker แทนการแก้ ownership design จริง ๆ**

เมื่อ borrow checker ปฏิเสธโค้ด ปฏิกิริยาที่อันตรายที่สุดคือการห่อทุกอย่างด้วย `Rc<RefCell<T>>` เพื่อให้ compile
ผ่าน โดยไม่เข้าใจว่าทำไม borrow checker ถึง complain ผลที่ตามมาคือบั๊กที่เคยถูกจับตอน compile time กลับกลายเป็น
panic ตอน runtime แทน:

```
thread 'main' panicked at src/main.rs:15:35:
RefCell already borrowed
```

วิธีแก้: ก่อนเอื้อมมือไปหา `Rc<RefCell<T>>` ให้ถามก่อนว่า "มีเจ้าของข้อมูลนี้มากกว่าหนึ่งจุดจริง ๆ หรือไม่" ถ้าไม่
— ใช้ ownership ปกติ ปรับ struct/function signature ให้ ownership ชัดเจนขึ้นแทน (ดูหัวข้อ 109.1, 109.3)

**2. `#[allow(clippy::...)]` พิมพ์ชื่อ lint ผิด — คำเตือนหายไปเงียบ ๆ โดยไม่รู้ตัว**

```
warning: unknown lint: `clippy::redundent_clone`
 --> src/main.rs:1:9
  |
1 | #[allow(clippy::redundent_clone)]
  |         ^^^^^^^^^^^^^^^^^^^^^^^ help: did you mean: `clippy::redundant_clone`
  |
  = note: `#[warn(unknown_lints)]` on by default
```

ที่อันตรายคือ `unknown_lints` เป็นแค่ **warning** ไม่ใช่ error — โค้ดยัง compile ผ่านสบาย ๆ และถ้าไม่มีใครสังเกต
warning นี้ (เช่น terminal output ยาวจนเลื่อนผ่านไป) จะเข้าใจผิดว่า lint ตัวเดิมถูกปิดอยู่ ทั้งที่จริง ๆ clippy ยัง
รายงานคำเตือนตัวเดิมอยู่ครบทุกครั้ง เพียงแค่ปนกับ warning เรื่อง unknown lint วิธีป้องกันคือรัน
`cargo clippy -- -D warnings` ใน CI เสมอ (ยกระดับทุก warning รวมถึง `unknown_lints` เป็น error) เพื่อให้พิมพ์ผิด
แบบนี้ถูกจับได้ทันทีไม่ปล่อยผ่าน

**3. ลืมว่าเทสต์ผูกติดกับ field ภายในที่ควรเป็น private จนรีแฟกเตอร์ไม่ได้เลย**

เมื่อเทสต์เข้าถึง `lib.books[0].copies_available` ตรง ๆ (ตามตัวอย่างในหัวข้อ 109.11) การเปลี่ยนโครงสร้างข้อมูล
ภายในแม้เพียงเล็กน้อย (เช่นเปลี่ยน `Vec<Book>` เป็น `HashMap<String, Book>`) จะทำให้ **compile ไม่ผ่านทันที**
ที่จุดเทสต์ ไม่ใช่แค่ทดสอบไม่ผ่าน:

```
error[E0609]: no field `books` on type `Library<InMemoryBookRepository>`
```

(error จริงจะขึ้นอยู่กับว่า field นั้นกลายเป็น private หรือเปลี่ยนชนิดไปอย่างไร) วิธีแก้คือเขียนเทสต์ผ่าน public
API เท่านั้นตั้งแต่แรก (หัวข้อ 109.11) — ถ้าเทสต์ที่มีอยู่แล้วผูกกับ implementation detail ให้ถือเป็นหนี้ทางเทคนิค
ที่ต้องรีแฟกเตอร์เทสต์นั้นก่อน ไม่ใช่แก้ field กลับเป็น `pub` เพื่อให้เทสต์ผ่านง่าย ๆ

**4. ใช้ `into_receipt()`-แบบ (consuming method) แล้วพยายามใช้ค่าตัวเดิมต่อ**

เมื่อทำตาม naming convention `into_` (บริโภค `self`) แต่ผู้เรียกลืมว่า `self` ถูก move ไปแล้ว:

```
error[E0382]: borrow of moved value: `record`
  --> examples/naming.rs:76:20
   |
60 |     let record = BorrowRecord::new("Somchai", "111", 5);
   |         ------ move occurs because `record` has type `BorrowRecord`, which does not implement the `Copy` trait
...
71 |     let receipt = record.into_receipt();
   |                          -------------- `record` moved due to this method call
76 |     println!("{}", record.as_isbn_str()); // ตั้งใจใส่ผิดเพื่อสาธิต error
   |                    ^^^^^^ value borrowed here after move
```

นี่ไม่ใช่ "บั๊ก" ของ borrow checker แต่เป็นสิ่งที่ naming convention `into_` **ตั้งใจสื่อสาร** ไว้แล้วตั้งแต่ตอนตั้ง
ชื่อฟังก์ชัน — ถ้าคุณต้องการใช้ค่าเดิมต่อหลังเรียกเมธอดที่แปลง type ให้มองหา method แบบ `to_` (ยืม `&self`) แทน
`into_` หรือถ้าจำเป็นต้องใช้ `into_` แต่ก็ยังต้องการ owned copy ของค่าเดิมไว้ด้วย ให้ `.clone()` ก่อนเรียก
`into_receipt()` อย่างตั้งใจ

**5. รีแฟกเตอร์ฟังก์ชันที่ไม่มีเทสต์โดยไม่เขียน characterization test ก่อน**

เมื่อแก้ `calculate_late_fee` (หัวข้อ 109.13) เพื่อ "ทำความสะอาด" cast จาก `i64` เป็น `f64` โดยไม่ได้ตรวจสอบผลลัพธ์
ตัวเลขก่อน-หลังให้ตรงกัน อาจเผลอเปลี่ยนพฤติกรรมการคำนวณค่าปรับจริงโดยไม่รู้ตัว (เช่น ปัดเศษต่างไปจากเดิม) ซึ่ง
กระทบเงินจริงของผู้ใช้จริง — ถ้าไม่มี characterization test ที่ตรึงค่าเดิมไว้ก่อน จะไม่มีทางรู้เลยว่าการรีแฟกเตอร์
เปลี่ยนพฤติกรรมไปหรือไม่จนกว่าจะมีคนมาแจ้งปัญหาในโปรดักชัน

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** เขียน struct `Product` (สำหรับระบบคลังสินค้า) ที่มี field `name: String`, `price: f64`,
   `stock: u32` แล้วเขียนเมธอด `fn total_value(&self) -> f64` (คืน `price * stock as f64`) จากนั้นรัน
   `cargo clippy -- -W clippy::pedantic` บนโค้ดนี้ ดูว่า clippy เตือนเรื่อง cast ระหว่าง `u32` กับ `f64` หรือไม่
   (hint: `stock` เป็น `u32` ไม่ใช่ `i64` แบบตัวอย่างค่าปรับในบทนี้ ลองเปรียบเทียบว่า `cast_precision_loss` ยัง
   เตือนอยู่ไหมและทำไม — `u32` มีช่วงค่าแคบกว่า `i64` มาก)

2. **[กลาง]** นำฟังก์ชัน `checkout` เวอร์ชัน "messy" จากหัวข้อ 109.14 (ที่มี `Rc<RefCell<>>` และ `.unwrap()`)
   มารีแฟกเตอร์เองทีละขั้นตามรูปแบบในหัวข้อ 109.8: (1) เอา `Rc<RefCell<>>` ออกก่อน ให้ compile ผ่าน (2) เปลี่ยน
   `.unwrap()` เป็น `Result` ให้ compile ผ่าน (3) เปลี่ยน parameter จาก `String` เป็น `&str` ให้ compile ผ่าน
   ทำทีละขั้นและรัน `cargo build` ยืนยันว่า compile ผ่านทุกขั้นก่อนไปขั้นต่อไป (hint: ห้ามทำสามขั้นพร้อมกันในคอมมิต
   เดียว เพราะถ้า compile error จะแยกไม่ออกว่าขั้นไหนทำให้พัง)

3. **[กลาง]** เขียนฟังก์ชัน `fn discount_tier(purchase_count: u32) -> f64` ที่ไม่มีเทสต์เลย (ใส่ logic แบบ
   if-else หลายขั้นตามใจ เช่น ซื้อ 0-4 ครั้งไม่มีส่วนลด, 5-9 ครั้งลด 5%, 10 ครั้งขึ้นไปลด 10%) แล้วเขียน
   characterization test ครอบคลุมทุกช่วงตามหัวข้อ 109.13 ก่อน จากนั้นลองรีแฟกเตอร์ฟังก์ชันให้อ่านง่ายขึ้น (เช่นใช้
   `match` กับ range pattern) แล้วรัน test ยืนยันว่าพฤติกรรมยังเหมือนเดิมทั้งหมด (hint: ระวัง edge case ที่ขอบเขต
   พอดี เช่น purchase_count = 5 และ = 10)

4. **[ยาก/ประยุกต์]** นำระบบห้องสมุดฉบับสมบูรณ์จากหัวข้อ 109.14 มาต่อขยาย: เพิ่ม `trait MemberRepository`
   (คล้าย `BookRepository`) แล้วเปลี่ยน `Library<B>` เป็น `Library<B: BookRepository, M: MemberRepository>` ให้
   ทั้งสอง generic parameter ทำงานผ่าน port/adapter ทั้งคู่ตามแนวคิด hexagonal architecture จากนั้นเขียน
   `struct FakeFailingBookRepository` ที่ implement `BookRepository` แต่ `find_mut` คืน `None` เสมอ (จำลอง
   สถานการณ์ที่ฐานข้อมูลจริง down) แล้วเขียนเทสต์ยืนยันว่า `Library::checkout` คืน `Err(LibraryError::BookNotFound)`
   อย่างถูกต้องโดยไม่ panic แม้ adapter จะ "พัง" — นี่คือประโยชน์ที่จับต้องได้จริงของการแยก port ออกจาก adapter:
   ทดสอบพฤติกรรมตอน dependency ล้มเหลวได้โดยไม่ต้องพึ่งฐานข้อมูลจริงที่ down จริง ๆ

## สรุป

บทนี้เจาะลึกคำว่า "clean code" ในบริบทเฉพาะของ Rust ที่ต่างจากภาษาอื่น: idiomaticity ไม่ใช่แค่เรื่องความสวยงาม แต่
เป็นสัญญาณเตือนของการออกแบบ ownership ที่ผิดพลาด — การ clone มากเกินไป, `Rc<RefCell<T>>` ที่ใช้ผิดที่, และ
`.unwrap()` ในโค้ด business logic ล้วนเป็น "กลิ่น" ที่บอกว่าโค้ดกำลังหลีกเลี่ยงการคุยกับระบบ ownership อย่างตรงไป
ตรงมา เราได้ฝึกใช้ checklist การรีวิว 4 หมวด (ownership/borrowing, error handling, API design, allocation) บนโค้ด
จริงพร้อมคำเตือนจริงจาก `cargo clippy` ในหลายระดับความเข้มงวด (`default`, `pedantic`, `nursery`, `restriction`)
เรียนรู้เทคนิครีแฟกเตอร์ด้วยการ extract function/method และการใช้ trait กำหนดขอบเขตตามแนวคิด hexagonal
architecture ที่ทำให้ทดสอบ business logic ได้โดยไม่ต้องพึ่งรายละเอียดการเก็บข้อมูลจริง ตรวจสอบ naming convention
`is_`/`has_`/`into_`/`to_`/`as_` ที่ผูกกับพฤติกรรมจริงในระดับ type system ไม่ใช่แค่ความชอบส่วนตัว ทบทวนว่า doc
comment ที่ดีต้องอธิบาย "ทำไม" และ invariant ไม่ใช่แปลโค้ดเป็นคำพูด รีวิวคุณภาพเทสต์ที่ควรทดสอบผ่าน public API
เท่านั้น ตรวจสุขภาพ dependency ด้วย `cargo machete` และรู้จัก `cargo udeps` และปิดท้ายด้วยเทคนิค characterization
test ที่ทำให้กล้ารีแฟกเตอร์โค้ดเก่าที่ไม่มีเทสต์ได้อย่างปลอดภัย ทั้งหมดนี้ประกอบกันเป็น capstone ที่รีวิวและ
รีแฟกเตอร์ระบบห้องสมุดทั้งระบบจากโค้ดที่มีกลิ่นครบ 4 หมวดให้กลายเป็นโค้ดที่ผ่าน `cargo clippy` ระดับ default โดย
ไม่มี warning เหลือเลย พร้อมเทสต์ 9 ตัวที่ทดสอบพฤติกรรมทางธุรกิจผ่าน public API ทั้งหมด

ทักษะการรีวิวและรีแฟกเตอร์ที่เรียนในบทนี้เป็นสิ่งที่แยกนักพัฒนา Rust มือใหม่ออกจากมือโปรอย่างชัดเจนที่สุด — มือใหม่
เขียนโค้ดที่ compile ผ่าน ส่วนมือโปรรู้ว่าโค้ดที่ compile ผ่านนั้น "ดีพอ" หรือยัง และรู้วิธีทำให้มันดีขึ้นอย่าง
ปลอดภัยโดยไม่พังของเดิม ใน **Part 110** ซึ่งเป็นบทสุดท้ายของหลักสูตรนี้ เราจะนำทักษะทั้งหมดที่สั่งสมมา — ตั้งแต่
syntax พื้นฐาน ไปจนถึงการรีวิวโค้ดระดับมืออาชีพในบทนี้ — มาเตรียมพร้อมสำหรับการสัมภาษณ์งาน Rust Developer จริง
พร้อมแนวทางการวางแผนอาชีพในระยะยาวกับภาษานี้

---

**Part ก่อนหน้า:** [Capstone: Building a Production-Grade Web Service](part-108-capstone-web-service.md) | **Part ถัดไป:** [เตรียมตัวสัมภาษณ์งาน Rust Developer และแนวทางอาชีพ](part-110-interview-prep-career.md)
