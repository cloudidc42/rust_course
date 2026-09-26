# Part 40: Send, Sync และความปลอดภัยของ Concurrency

> โมดูล: ระดับกลาง (Intermediate) | ระดับ: กลาง | เวลาโดยประมาณ: 220 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้อย่างแม่นยำว่า **compiler รู้ได้อย่างไร** ว่าโค้ด concurrency ชิ้นหนึ่ง "ปลอดภัย" หรือ "ไม่ปลอดภัย"
  ก่อนที่โปรแกรมจะรันด้วยซ้ำ — ปิดวงคำถามที่เปิดไว้ตั้งแต่ Part 37 และ Part 39
- อธิบายความหมายที่แท้จริงของ trait `Send` (ปลอดภัยที่จะ**ย้าย**ข้ามเธรด) และ `Sync` (ปลอดภัยที่จะ**แชร์
  reference**ข้ามเธรด) พร้อมบอกความสัมพันธ์เชิงคณิตศาสตร์ระหว่างสองสิ่งนี้ได้ (`T: Sync` ก็ต่อเมื่อ `&T: Send`)
- อธิบายกลไกที่แท้จริง (ไม่ใช่แค่ท่องจำ) ว่าทำไม `Rc<T>` ไม่ใช่ `Send` และทำไม `RefCell<T>` ไม่ใช่ `Sync` —
  ทั้งสองมีสาเหตุเดียวกันคือ counter ภายในที่ไม่ใช่ atomic
- อธิบายได้ว่า auto trait ถูกคำนวณโดย compiler แบบ**อัตโนมัติและเป็น structural/recursive** อย่างไร
  (ดูจาก field ทุกตัวของ struct/enum) และรู้จักวิธี opt-out ด้วย `PhantomData` หรือ negative impl
- อ่านลายเซ็นเต็มของ `std::thread::spawn` ได้ทุกส่วน (`FnOnce`, `Send`, `'static`) และอธิบายได้ว่าทำไม
  compiler ต้องบังคับ bound แต่ละตัวเป๊ะ ๆ แบบนี้
- อธิบาย "final boss" ของ concurrency arc ทั้งหมด: ทำไม `Arc<Mutex<T>>` จึงเป็นทั้ง `Send` และ `Sync`
  พร้อมกัน และนำความรู้นี้ไปออกแบบ type ของตัวเองที่ปลอดภัยสำหรับ multi-thread ได้จริง

## ความรู้ที่ต้องมีมาก่อน

- **Part 21 (Traits ขั้นสูง)** — หัวข้อ 21.7 เกริ่นเรื่อง **auto trait** ไว้สั้น ๆ ตอนอธิบาย error E0225 ว่า
  `Send`/`Sync` เป็น trait พิเศษที่ไม่มี method จึงรวมกับ `dyn Trait` อื่นได้ไม่จำกัด — บทนี้จะขยายความเต็มรูปแบบ
- **Part 23 (Lifetimes ขั้นสูง)** — แนวคิด `'static` lifetime ที่จะกลับมาเป็นส่วนหนึ่งของ bound ใน `thread::spawn`
- **Part 24 (Closures)** — ความแตกต่างของ `Fn`, `FnMut`, `FnOnce` และการ capture ตัวแปรแบบ `move`
- **Part 27–29 (Smart Pointers: `Box<T>`, `Rc<T>`/`RefCell<T>`, `Weak<T>`/`Cow<T>`)** — โดยเฉพาะกลไก
  reference counting ของ `Rc<T>` และ interior mutability ของ `RefCell<T>` ที่บทนี้จะอธิบายว่า "ทำไม" มันไม่ปลอดภัย
  ข้ามเธรด
- **Part 37 (Threads พื้นฐาน)** — การใช้ `thread::spawn` และ error E0373 ตอนพยายาม capture reference
  ที่ไม่ใช่ `'static`
- **Part 38 (Channels)** — การส่งข้อมูลข้ามเธรดผ่าน `mpsc`
- **Part 39 (Mutex, Arc และ Shared-State Concurrency)** — การใช้ `Arc<Mutex<T>>` จริง และ error E0277
  ตอนพยายามส่ง `Rc<T>` ข้ามเธรด (บทนี้จะอธิบาย**กลไกที่แท้จริง**เบื้องหลัง error นั้น)

## เนื้อหา

### 40.1 ทวนความจำ: compiler จับบั๊ก concurrency ได้อย่างไร (ปิดวง Part 37–39)

ตลอดสามบทที่แล้ว เราเขียนโปรแกรม concurrency จริงหลายตัว และ compiler ก็ปฏิเสธโค้ดที่ "ดูเหมือนจะทำงานได้"
อยู่หลายครั้ง สองครั้งที่สำคัญที่สุดคือ:

1. **Part 37**: ตอนพยายามสร้าง thread ใหม่ด้วย closure ที่ capture reference ของตัวแปรใน stack frame ปัจจุบัน
   (ไม่ใช่ `move`) compiler ปฏิเสธด้วย error **E0373** ("closure may outlive the current function") เพราะ
   thread ใหม่**อาจ**มีชีวิตอยู่นานกว่าตัวแปรต้นฉบับ — ถ้า thread หลักจบฟังก์ชันไปก่อน ตัวแปรจะถูก drop
   ทิ้งไปทั้งที่ thread ลูกยังอ้างถึงอยู่ นี่คือ **dangling reference ข้ามเธรด**
2. **Part 39**: ตอนพยายามส่ง `Rc<T>` เข้าไปใน `thread::spawn` compiler ปฏิเสธด้วย error **E0277**
   ("`Rc<i32>` cannot be sent between threads safely") — ตอนนั้นเราอธิบายแค่ระดับ "จำไว้ว่า `Rc` ใช้ข้ามเธรด
   ไม่ได้ ให้ใช้ `Arc` แทน" โดยยังไม่ได้ลงกลไกจริงว่า **ทำไม**

คำถามที่ค้างอยู่คือ: compiler รู้เรื่องนี้ได้อย่างไร? มันไม่ได้รันโปรแกรมแล้วเจอ data race ตอน runtime
เหมือนภาษาอื่น ๆ (เช่น Go ที่ต้องพึ่ง `go run -race` หรือ C++ ที่ data race เป็น undefined behavior ที่เจอเอาตอน
crash) — Rust ปฏิเสธโค้ดพวกนี้ตั้งแต่ตอน `cargo check` **ก่อน**โปรแกรมจะรันด้วยซ้ำ

คำตอบคือ: `thread::spawn` มี **trait bound** กำกับอยู่ในลายเซ็นของมันเอง และ bound เหล่านั้นคือ trait สองตัวที่
เป็นหัวใจของบทนี้ — **`Send`** และ **`Sync`** ทุก error ที่เราเจอใน Part 37 (ผ่าน `'static`) และ Part 39
(ผ่าน `Send`) ล้วนมาจากการที่ compiler ตรวจ bound พวกนี้แบบ **static analysis ที่ compile time** ล้วน ๆ
ไม่มี runtime cost เลยแม้แต่นาโนวินาทีเดียว — นี่คือแก่นของคำว่า **"fearless concurrency"** ที่ทีม Rust ใช้พูดถึง
ภาษาตัวเองมาตลอด: ไม่ใช่เพราะ Rust "ห้ามเขียนโค้ดผิด" แต่เพราะ **ระบบ type ผลักดันให้บั๊ก concurrency
กลายเป็น compile error แทนที่จะเป็น runtime bug ที่จับยาก**

บทนี้จะรื้อกลไกทั้งหมดออกมาดูทีละชั้น เริ่มจากนิยามของ auto trait ที่ Part 21 เกริ่นไว้

### 40.1.1 มุมมองเทียบกับภาษาอื่น: ทำไม Rust ตรวจได้ตอน compile time ทั้งที่ภาษาอื่นตรวจได้แค่ตอน runtime

ก่อนลงรายละเอียดกลไกของ `Send`/`Sync` ลองเทียบกับวิธีที่ภาษาอื่นจัดการปัญหาเดียวกันดูสักครู่ จะช่วยให้เห็นว่า
สิ่งที่ Rust ทำอยู่นั้น**ไม่ธรรมดา**ขนาดไหน:

- **C++**: `std::thread`, `std::mutex` มีให้ใช้ แต่ไม่มีกลไกบังคับใด ๆ ในระดับ type ที่ป้องกันไม่ให้คุณส่ง
  pointer หรือ reference ที่ไม่ thread-safe ข้าม thread ได้ — ถ้าลืมใส่ `std::lock_guard` หรือแชร์ object
  ที่มี internal state ไม่ thread-safe ข้าม thread โปรแกรมจะ**compile ผ่านได้ปกติ**แล้ว data race จะเกิดขึ้น
  ที่ runtime แบบสุ่ม ๆ ทำให้ debug ยากมาก (บั๊กแบบนี้เรียกกันว่า "Heisenbug" — สังเกตอาการเปลี่ยนไปทุกครั้งที่
  รัน) ต้องพึ่งเครื่องมือแยกอย่าง `ThreadSanitizer` (`-fsanitize=thread`) มาช่วยตรวจตอน runtime และตรวจได้แค่
  path ของโค้ดที่ถูก execute จริงในการรันครั้งนั้นเท่านั้น ไม่ครอบคลุมทุก path เหมือน static analysis
- **Java**: มี keyword `synchronized` และ class `java.util.concurrent` ที่ช่วยจัดการ แต่การลืมใส่
  `synchronized` ตรง critical section ก็ยังคง**compile ผ่านได้ปกติ**เช่นกัน — JVM ไม่มีทางรู้ตอน compile
  time ว่า object ไหนถูกแชร์ข้าม thread โดยไม่มีการป้องกัน ต้องอาศัยทั้ง code review และเครื่องมือ static
  analysis เสริม (เช่น FindBugs/SpotBugs) ที่เป็น**เครื่องมือแยก ไม่ใช่ส่วนหนึ่งของภาษา**
- **Go**: มี goroutine และ channel ที่ทำให้เขียน concurrency สั้นและง่ายกว่า Rust มาก แต่ก็มีคำเตือนที่
  โด่งดังของทีม Go เองว่า "Do not communicate by sharing memory; instead, share memory by communicating"
  — เป็นแค่**คำแนะนำเชิงวัฒนธรรม**ไม่ใช่กฎที่ compiler บังคับ Go มี `go run -race` (race detector) ที่ตรวจ
  data race ได้ดีมาก แต่ก็ยังเป็นเครื่องมือที่ต้อง**รันโปรแกรมจริงและ path นั้นต้องถูก execute**ถึงจะเจอ
  ไม่ใช่การตรวจแบบ static ที่ครอบคลุมทุก path ที่เป็นไปได้เหมือนที่ Rust ทำ

สิ่งที่ Rust ทำแตกต่างออกไปโดยพื้นฐาน: `Send`/`Sync` ไม่ใช่ "เครื่องมือเสริม" หรือ "ธรรมเนียมการเขียนโค้ดที่ดี"
แต่เป็น **ส่วนหนึ่งของระบบ type ที่ compiler ตรวจบังคับทุกครั้งที่ compile** — ไม่ว่า path ของโค้ดนั้นจะถูก
execute จริงหรือไม่ก็ตาม (static analysis ครอบคลุม 100% ของโค้ดที่เขียน ไม่ใช่แค่ path ที่ test case ไปเจอ)
นี่คือเหตุผลที่คำว่า "fearless concurrency" ไม่ใช่คำโฆษณาเกินจริง — มันคือผลลัพธ์ตรงจากการออกแบบ type system
ให้ auto trait สองตัวนี้เป็นส่วนหนึ่งของทุกลายเซ็นฟังก์ชันที่เกี่ยวกับ thread โดยอัตโนมัติ

### 40.2 Auto Trait คืออะไร (ทวนและขยายจาก Part 21)

ใน Part 21 หัวข้อ 21.7 เราเห็นข้อความนี้ตอนอธิบาย error E0225:

> "auto-traits like `Send` and `Sync` are traits that have special properties"

**Auto trait** คือ trait พิเศษกลุ่มเล็ก ๆ ที่มีคุณสมบัติต่างจาก trait ทั่วไปสามข้ออย่างชัดเจน:

1. **ไม่มี method เลยแม้แต่ตัวเดียว** — มันเป็น **marker trait** แบบเดียวกับ `Copy` ที่เราเรียนใน Part 19
   หัวข้อ 19.12 (trait ที่ไม่มี behavior อะไรให้ implement แค่ "ป้ายกำกับ" ว่า type นี้มีคุณสมบัติอะไรบางอย่าง)
2. **compiler เป็นคน implement ให้อัตโนมัติ** — คุณไม่ต้องเขียน `impl Send for MyType {}` เอง (และปกติก็เขียน
   ไม่ได้ด้วย เพราะ `Send`/`Sync` เป็น `unsafe trait` — การ implement เองต้องเขียน `unsafe impl` ซึ่งหมายความว่า
   คุณกำลังรับประกันด้วยตัวเองว่ามันปลอดภัยจริง สิ่งที่ compiler ตรวจให้ไม่ได้อีกต่อไป)
3. **คำนวณจาก field ภายในแบบ structural/recursive** — type ผสม (struct, enum, tuple) จะเป็น auto trait
   นั้นโดยอัตโนมัติ **ก็ต่อเมื่อทุก field ภายในเป็น auto trait นั้นด้วย** (รายละเอียดเต็มอยู่หัวข้อ 40.8)

Rust มี auto trait หลักอยู่สามตัวใน standard library: `Send`, `Sync`, และ `Unpin` (ตัวหลังเกี่ยวกับ async/await
ซึ่งจะเรียนใน Part 46) บทนี้โฟกัสที่สองตัวแรกซึ่งเป็นหัวใจของ concurrency ทั้งหมด

นิยามของ trait ทั้งสองใน standard library จริง ๆ นั้นสั้นมาก (ไม่มี method):

```rust
// นี่คือรูปแบบนิยามจริงของ Send/Sync ใน std::marker (คัดลอกโครงมาให้ดู ไม่ต้อง implement เอง)
// pub unsafe auto trait Send {}
// pub unsafe auto trait Sync {}
```

สังเกตคำว่า `unsafe auto trait` — `auto` บอกว่าคอมไพเลอร์แจกให้อัตโนมัติ ส่วน `unsafe` บอกว่าถ้าใครจะ
implement เอง (เช่นทีมงาน std เอง ตอนเขียน `Arc<T>` หรือ `Mutex<T>`) ต้องรับผิดชอบด้วยตัวเองว่าไม่ทำให้เกิด
undefined behavior จริง ๆ (คำสำคัญ `auto trait` ในไวยากรณ์นี้ยังเป็น **unstable feature**
`#![feature(auto_traits)]` สำหรับผู้ใช้ทั่วไป — เฉพาะ std เท่านั้นที่นิยาม auto trait ใหม่ได้ในปัจจุบัน
ผู้ใช้งานทั่วไปแค่ "ใช้" `Send`/`Sync` ที่มีอยู่แล้ว หรือ opt-out ด้วยเทคนิคที่จะเห็นในหัวข้อ 40.9)

(สำหรับ `Unpin` — auto trait ตัวที่สามที่เอ่ยถึงไปตอนต้นหัวข้อนี้ — ยังไม่ต้องสนใจตอนนี้ มันเกี่ยวข้องกับ
การ "ย้ายตำแหน่งในหน่วยความจำ" ของ type ที่ implement `Future` โดยเฉพาะ ซึ่งเป็นแนวคิดที่จะเข้าใจได้เต็มที่
ก็ต่อเมื่อเรียน async/await ใน Part 46 เท่านั้น — ตอนนี้ให้จำแค่ว่ามันเป็น auto trait ตัวที่สามที่มีอยู่จริง
เพื่อจะไม่แปลกใจถ้าเจอชื่อมันโผล่มาใน error message ของโค้ด async ในอนาคต)

### 40.3 `Send`: ปลอดภัยที่จะ "ย้าย" ข้ามเธรด

นิยามที่ต้องจำให้แม่นคือ:

> **`T: Send`** หมายความว่า การโอนความเป็นเจ้าของ (ownership) ของค่าชนิด `T` จาก thread หนึ่งไปยังอีก thread
> หนึ่งเป็นเรื่องปลอดภัย

สังเกตคำว่า **"โอนความเป็นเจ้าของ"** — นี่คือ**การย้าย (move)** ไม่ใช่การแชร์ (`Sync` ในหัวข้อถัดไปคือการแชร์)
เมื่อค่าถูก move ข้ามเธรด มันจะมี**เจ้าของเดียว**อยู่เสมอ (ตรงตามกฎ ownership จาก Part 6) — ไม่มีสองเธรดถือ
ค่าเดียวกันพร้อมกัน จึงไม่มีทางเกิด data race จากการ**เข้าถึงพร้อมกัน**ได้ (เพราะมีแค่เธรดเดียวที่เข้าถึงได้ ณ
เวลาใดก็ตาม) แต่ก็ไม่ได้แปลว่าการ move ข้ามเธรดจะปลอดภัย**เสมอไปโดยอัตโนมัติ** — ถ้า type ภายในมี
mechanism ที่พึ่งพา "ฉันอยู่ใน thread เดิมเสมอ" (เช่น non-atomic counter ที่คาดหวังว่าจะไม่มีใครแก้พร้อมกัน
จากเธรดอื่น) การย้ายอาจทำให้เกิดปัญหาที่ subtle กว่านั้น — นี่คือเหตุผลที่ `Rc<T>` ถึงไม่ใช่ `Send` ทั้งที่
"ย้ายทั้งก้อน" ดูเหมือนจะปลอดภัย

**ข่าวดี**: type ส่วนใหญ่ในโปรแกรม Rust ปกติเป็น `Send` โดยอัตโนมัติ เพราะมันเป็น **auto trait** — ถ้าคุณสร้าง
struct ขึ้นมาจาก field ที่เป็น `Send` ทั้งหมด (ซึ่ง type พื้นฐานอย่าง `i32`, `f64`, `bool`, `String`,
`Vec<T>` ที่ `T: Send`, `Box<T>` ที่ `T: Send` ล้วนเป็น `Send`) struct ของคุณก็เป็น `Send` โดยไม่ต้องเขียน
อะไรเพิ่มเลย

```rust
// ตัวช่วยสำหรับ "พิสูจน์" ที่ compile time ว่า type ใด ๆ เป็น Send/Sync หรือไม่
// ถ้า T ไม่ตรงกับ bound compiler จะปฏิเสธ compile ทันที (เทคนิคมาตรฐานที่ใช้ตรวจ auto trait)
fn assert_send<T: Send>() {}
fn assert_sync<T: Sync>() {}

struct Order {
    id: u32,
    customer_name: String,
    items: Vec<String>,
}

fn main() {
    assert_send::<Order>();
    assert_sync::<Order>();
    assert_send::<i32>();
    assert_sync::<i32>();
    assert_send::<String>();
    assert_sync::<String>();
    assert_send::<Vec<i32>>();
    assert_sync::<Vec<i32>>();
    println!("Order เป็นทั้ง Send และ Sync โดยอัตโนมัติ");
}
```

ผลลัพธ์:

```
Order เป็นทั้ง Send และ Sync โดยอัตโนมัติ
```

`assert_send::<T>()` และ `assert_sync::<T>()` เป็น**เทคนิคมาตรฐาน**ที่ใช้กันทั่วไปในโค้ด production และแม้ใน
test suite ของ std เอง — มันคือฟังก์ชัน generic เปล่า ๆ ที่มี bound `T: Send` หรือ `T: Sync` กำกับไว้ ถ้า
เรียก `assert_send::<Order>()` แล้ว compile ผ่าน แปลว่า `Order: Send` เป็นจริง ถ้า compile ไม่ผ่าน (จะเห็นใน
หัวข้อถัดไป) แปลว่าไม่เป็น `Send` — เราจะใช้เทคนิคนี้ตลอดทั้งบทเพื่อ "พิสูจน์" ให้เห็นจริงว่า type ไหนเป็นหรือไม่
เป็น `Send`/`Sync` โดยไม่ต้องเขียนโปรแกรม multi-thread เต็มรูปแบบทุกครั้ง

### 40.4 ทำไม `Rc<T>` ไม่ใช่ `Send` (คำตอบที่แม่นยำที่ Part 39 ยังไม่ได้ลงลึก)

ทวนจาก Part 28: `Rc<T>` ("Reference Counted") คือ smart pointer ที่อนุญาตให้มีเจ้าของหลายคน (`multiple
ownership`) โดยเก็บ **ตัวนับจำนวนการอ้างอิง (reference count)** ไว้ในหน่วยความจำที่ heap ควบคู่กับข้อมูล ทุกครั้ง
ที่เรียก `Rc::clone(&rc)` ตัวนับจะ**เพิ่มขึ้นหนึ่ง** และทุกครั้งที่ `Rc<T>` ตัวใดตัวหนึ่งถูก drop ตัวนับจะ
**ลดลงหนึ่ง** — เมื่อตัวนับถึงศูนย์ ข้อมูลจริงบน heap จะถูกปล่อยคืน

จุดสำคัญที่ Part 28 ไม่ได้เน้นมาก่อน (เพราะตอนนั้นยังไม่มี context เรื่อง thread เลย) คือ **ตัวนับนี้เป็นแค่
`Cell<usize>` ธรรมดา — จำนวนเต็มไม่มีเครื่องหมายชนิดปกติที่สุด ไม่ใช่ atomic integer** การเพิ่ม/ลดค่าทำผ่าน
operation ธรรมดาแบบ "อ่านค่าปัจจุบัน → บวกหรือลบหนึ่ง → เขียนค่ากลับ" (read-modify-write) ซึ่ง**ไม่ได้เป็น
operation เดียวที่แบ่งแยกไม่ได้ (atomic)** ในระดับ CPU

ลองนึกภาพสถานการณ์นี้ ถ้า Rust "อนุญาต" ให้ `Rc<T>` เป็น `Send` ได้ (ซึ่งจริง ๆ มันไม่อนุญาต แต่เราสมมติเพื่อ
เข้าใจเหตุผล): มี `Rc<T>` สองตัวที่ clone มาจากตัวเดียวกัน (ตัวนับ = 2) ตัวหนึ่งอยู่ใน thread A อีกตัวอยู่ใน
thread B ถ้าทั้งสอง thread เกิด drop ค่าของตัวเอง**พร้อมกันในเวลาเดียวกันจริง ๆ** (concurrent) ทั้งคู่จะ:

1. อ่านค่าตัวนับปัจจุบัน (สมมติว่าทั้งคู่อ่านเห็นค่า `2` พร้อมกัน เพราะยังไม่มีใครเขียนค่าใหม่ทันเวลา)
2. คำนวณค่าใหม่เป็น `2 - 1 = 1`
3. เขียนค่า `1` กลับเข้าไปในตัวนับ — **ทั้งสอง thread เขียน `1` ทับกัน**

ผลลัพธ์คือตัวนับกลายเป็น `1` ทั้งที่ควรจะเป็น `0` (เพราะมีการ drop เกิดขึ้นจริงสองครั้ง) ลองดูลำดับเวลาแบบ
ละเอียดที่สุดว่าเกิดอะไรขึ้นจริง ๆ ในระดับ instruction (นี่คือสิ่งที่**อาจ**เกิดขึ้นได้ ถ้า Rust ยอมให้ทำแบบนี้
— ในความเป็นจริง compiler ปฏิเสธไม่ให้ compile ตั้งแต่แรก):

| เวลา | Thread A (drop ตัวที่ 1) | Thread B (drop ตัวที่ 2) | ค่าตัวนับจริงบน heap |
|---|---|---|---|
| t0 | — | — | 2 |
| t1 | อ่านค่าตัวนับ ได้ `2` | — | 2 |
| t2 | — | อ่านค่าตัวนับ ได้ `2` (ยังไม่เห็นการเขียนจาก A) | 2 |
| t3 | คำนวณ `2 - 1 = 1` | — | 2 |
| t4 | — | คำนวณ `2 - 1 = 1` | 2 |
| t5 | เขียนค่า `1` กลับ | — | 1 |
| t6 | — | เขียนค่า `1` กลับ (ทับค่าที่ A เขียนไปแล้ว!) | **1** (ควรเป็น 0) |

สังเกตที่ t6: ทั้งสอง thread ต่างคำนวณจากค่าตั้งต้นเดียวกัน (`2` ที่อ่านไปคนละครั้งใน t1 และ t2) แล้วต่างเขียน
ผลลัพธ์ `1` ทับกันคนละรอบ ทั้งที่ในความเป็นจริงมีการ drop เกิดขึ้น**สองครั้ง**ซึ่งควรทำให้ตัวนับลดลงสองครั้ง
(`2 → 1 → 0`) — นี่คือสิ่งที่เรียกว่า **lost update** (การอัปเดตหนึ่งครั้งถูก "กลืน" หายไปโดยไม่มีผลอะไรเลย)
ซึ่งเป็นรูปแบบคลาสสิกที่สุดของ data race ในระบบที่มีตัวนับร่วม (shared counter) ไม่ว่าจะเป็นตัวนับ reference
count ของ `Rc<T>` หรือตัวนับ borrow flag ของ `RefCell<T>` ในหัวข้อ 40.7 ก็เกิดจาก pattern เดียวกันนี้เป๊ะ
ตัวนับที่ผิดพลาดแบบนี้นำไปสู่หายนะสองแบบ:

- **Memory leak**: ถ้าตัวนับ "สูงกว่าความจริง" (เหมือนตัวอย่างข้างบนที่ควรเป็น 0 แต่ค้างที่ 1) ข้อมูลบน heap
  จะไม่ถูกปล่อยคืนเลย ทั้งที่ไม่มีใครใช้งานมันแล้ว
- **Double free (อันตรายกว่ามาก)**: ถ้าเกิดกรณีตรงกันข้าม ตัวนับ "ต่ำกว่าความจริง" จนถึง 0 เร็วเกินไปทั้งที่ยัง
  มี `Rc<T>` ตัวอื่นถืออยู่จริง หน่วยความจำจะถูกปล่อยคืนไปให้ระบบใช้ซ้ำ ทั้งที่ยังมีตัวชี้ (pointer) อีกตัวที่ยัง
  "เชื่อ" ว่าข้อมูลยังอยู่ — พอตัวนั้นพยายามอ่าน/เขียนข้อมูลต่อ จะกลายเป็น **use-after-free**
  ซึ่งเป็นช่องโหว่ความปลอดภัยระดับร้ายแรงที่สุดในภาษาระดับต่ำ (ตรงประเภทเดียวกับที่ CVE จำนวนมากใน C/C++
  มาจาก)

นี่คือคำตอบที่แม่นยำและเป็นกลไกจริง (ไม่ใช่แค่ "กฎที่ต้องจำ") ว่าทำไม `Rc<T>` ไม่ใช่ `Send`: **เพราะตัวนับ
ภายในของมันไม่ใช่ atomic การแก้ไขพร้อมกันจากสองเธรดจะเกิด race condition ที่ทำให้ตัวนับผิดพลาด** compiler
ไม่จำเป็นต้องรอให้เกิด race จริงถึงจะรู้ — มันรู้ตั้งแต่ตอนที่ std ประกาศ `Rc<T>` ว่า**ไม่ implement** `Send`
ให้ (คือใน std ประกาศชัดเจนว่า `impl<T> !Send for Rc<T> {}` ในทางความหมาย — จะเห็นวิธีเขียนแนวคิดนี้ในหัวข้อ
40.9) ผลคือทุกครั้งที่มีคนพยายามส่ง `Rc<T>` เข้า `thread::spawn` compiler จะปฏิเสธทันทีที่ compile time

มาดู error จริงที่เกิดขึ้น (เวอร์ชันเต็ม ตรงกับที่ Part 39 พบแบบย่อ):

```rust
use std::rc::Rc;
use std::thread;

fn main() {
    let data = Rc::new(42);
    let handle = thread::spawn(move || {
        println!("{}", data);
    });
    handle.join().unwrap();
}
```

Error จริง (compile ด้วย `rustc --edition 2021`):

```
error[E0277]: `Rc<i32>` cannot be sent between threads safely
 --> src/main.rs:6:32
  |
6 |       let handle = thread::spawn(move || {
  |                    ------------- ^------
  |                    |             |
  |  __________________|_____________within this `{closure@src/main.rs:6:32: 6:39}`
  | |                  |
  | |                  required by a bound introduced by this call
7 | |         println!("{}", data);
8 | |     });
  | |_____^ `Rc<i32>` cannot be sent between threads safely
  |
  = help: within `{closure@src/main.rs:6:32: 6:39}`, the trait `Send` is not implemented for `Rc<i32>`
note: required because it's used within this closure
 --> src/main.rs:6:32
  |
6 |     let handle = thread::spawn(move || {
  |                                ^^^^^^^
note: required by a bound in `spawn`
```

ตอนนี้เราสามารถอ่าน error นี้ได้ครบทุกบรรทัดแบบมีความหมาย:

- **`the trait `Send` is not implemented for `Rc<i32>`**` — นี่คือประโยคหลัก: `Rc<i32>` ไม่ใช่ `Send`
  เพราะเหตุผลเรื่องตัวนับที่ไม่ใช่ atomic ที่อธิบายไปข้างต้น
- **`required because it's used within this closure`** — closure ที่ส่งให้ `thread::spawn` **capture**
  ตัวแปร `data` เข้ามาข้างใน ทำให้ตัวมันเองก็ "ไม่ใช่ `Send`" ไปด้วย (ตาม auto trait rule ในหัวข้อ 40.8 —
  closure ก็เป็น struct ที่ไม่มีชื่อ ถูกคำนวณ `Send`/`Sync` แบบ structural เหมือนกัน)
- **`required by a bound in `spawn`** — `thread::spawn` เขียน bound `F: Send` ไว้ในลายเซ็นของตัวเอง (จะ
  เห็นเต็ม ๆ ในหัวข้อ 40.10) นี่คือจุดที่ compiler ใช้ตรวจสอบ

### 40.5 `Sync`: ปลอดภัยที่จะ "แชร์ reference" ข้ามเธรด

นิยามของ `Sync`:

> **`T: Sync`** หมายความว่า การมี **`&T`** (reference ที่ไม่ใช่เจ้าของ) อยู่ในมือของหลาย ๆ thread
> **พร้อมกัน**เป็นเรื่องปลอดภัย

สังเกตความแตกต่างจาก `Send` ให้ชัด: `Send` พูดถึงการ**ย้ายเจ้าของ** (มีเจ้าของเดียวเสมอ ไม่มีการแชร์)
ส่วน `Sync` พูดถึงการ**แชร์ reference** (หลายเธรดถือ `&T` ชี้ไปที่ค่าเดียวกัน**พร้อมกันจริง ๆ**) นี่คือสอง
สถานการณ์ที่แตกต่างกันโดยสิ้นเชิงในเชิง memory safety และเป็นเหตุผลที่ Rust ต้องมี**สอง trait แยกกัน** ไม่ใช่
trait เดียว

ทำไมการแชร์ `&T` ข้ามเธรดถึงอาจไม่ปลอดภัย ทั้งที่ `&T` เป็น**immutable reference**? คำตอบคือ **interior
mutability** จาก Part 28/29 — type บางชนิด (`Cell<T>`, `RefCell<T>`, `Rc<T>`) แม้จะให้ `&T` มาจากภายนอก
แต่ข้างในสามารถ**เปลี่ยนแปลงค่าได้จริง**ผ่านกลไกที่ซ่อนอยู่ (เช่น `RefCell::borrow_mut()` ที่ใช้แค่ `&self`
ไม่ใช่ `&mut self`) ถ้าสอง thread ถือ `&RefCell<T>` ตัวเดียวกันพร้อมกัน แล้วต่างฝ่ายต่างเรียก `borrow_mut()`
พร้อมกัน — เกิด race condition ทันที (รายละเอียดกลไกเต็ม ๆ อยู่หัวขัดถัดไป)

### 40.6 ความสัมพันธ์ที่แม่นยำระหว่าง `Send` และ `Sync`

นี่คือสูตรที่สำคัญที่สุดของบทนี้ ควรจำให้แม่น:

> **`T` เป็น `Sync` ก็ต่อเมื่อ `&T` เป็น `Send`**

พูดเป็นภาษาธรรมดา: "การแชร์ `T` ข้ามเธรด (`Sync`)" กับ "การย้าย `&T` ข้ามเธรด (`Send` ของ reference)"
คือ**สิ่งเดียวกัน** ฟังดูอาจงงตอนแรก แต่ลองไล่เหตุผล: การที่หลาย thread จะ "แชร์" `&T` พร้อมกันได้ ก็คือการที่
คุณสามารถ**สร้าง `&T` ตัวใหม่แล้วส่ง (move) มันไปให้ thread อื่นได้เรื่อย ๆ** — ซึ่งก็คือนิยามของ "`&T`
เป็น `Send`" นั่นเอง! สอง statement นี้จึงเทียบเท่ากันทางตรรกะ และ standard library ก็นิยาม `Sync` จริง ๆ
ด้วยสูตรนี้เป๊ะ (ใน libcore มี blanket implementation ประมาณ `unsafe impl<T: ?Sized> Sync for &T where T:
Sync {}` และความสัมพันธ์ผูกกันแบบนี้เกิดจากวิธีคำนวณ auto trait ของ reference type)

มาดูตัวอย่างที่พิสูจน์ความสัมพันธ์นี้ให้เห็นจริงทั้งสองทาง

**ทางที่ 1 — `T: Sync` ⇒ `&T: Send` (ตัวอย่างที่คอมไพล์ผ่าน):**

```rust
use std::sync::Mutex;

fn assert_send<T: Send>() {}

fn main() {
    // Mutex<i32> เป็น Sync ดังนั้น &Mutex<i32> ต้องเป็น Send ตามความสัมพันธ์ T: Sync <=> &T: Send
    assert_send::<&Mutex<i32>>();
    println!("&Mutex<i32> เป็น Send เพราะ Mutex<i32> เป็น Sync");
}
```

ผลลัพธ์: `&Mutex<i32> เป็น Send เพราะ Mutex<i32> เป็น Sync` — คอมไพล์ผ่านเพราะ `Mutex<T>` เป็น `Sync`
(กลไกอธิบายในหัวข้อ 40.11)

**ทางที่ 2 — `T` ไม่ใช่ `Sync` ⇒ `&T` ไม่ใช่ `Send` (ตัวอย่างที่คอมไพล์ไม่ผ่าน):**

```rust
use std::cell::RefCell;

fn assert_send<T: Send>() {}

fn main() {
    // RefCell<i32> ไม่ใช่ Sync ดังนั้น &RefCell<i32> ต้องไม่ใช่ Send ด้วย
    assert_send::<&RefCell<i32>>();
}
```

Error จริง:

```
error[E0277]: `&RefCell<i32>` cannot be sent between threads safely
 --> src/main.rs:7:19
  |
7 |     assert_send::<&RefCell<i32>>();
  |                   ^^^^^^^^^^^^^ `&RefCell<i32>` cannot be sent between threads safely
  |
  = help: the trait `Sync` is not implemented for `RefCell<i32>`
  = note: required for `&RefCell<i32>` to implement `Send`
help: consider removing the leading `&`-reference
```

สังเกตบรรทัด `= help: the trait \`Sync\` is not implemented for \`RefCell<i32>\`` ร่วมกับ
`= note: required for \`&RefCell<i32>\` to implement \`Send\`` — นี่คือ compiler เขียนความสัมพันธ์
`Sync ⇔ &T: Send` ให้เห็นตรง ๆ ใน error message เลย! มันไม่ได้แค่บอกว่า "`&RefCell<i32>` ไม่ใช่ `Send`"
แบบเดี่ยว ๆ แต่บอกเหตุผลย้อนไปถึงต้นตอ (`RefCell<i32>` ไม่ใช่ `Sync`) ตรงตามสูตรที่เราเพิ่งพิสูจน์ทุกคำ

### 40.7 ทำไม `RefCell<T>` เป็น `Send` แต่ไม่ใช่ `Sync` (คำตอบที่แม่นยำ — เชื่อม Part 28 เข้ากับ Part 39)

นี่คือจุดที่เชื่อมทุกอย่างในหลักสูตรเข้าด้วยกัน ทวนจาก Part 28: `RefCell<T>` ให้ **interior mutability** —
คุณสามารถ mutate ค่าภายในได้แม้ตัวแปรภายนอกเป็น `&RefCell<T>` (ไม่ใช่ `&mut`) เพราะ `RefCell<T>` ตรวจกฎ
"borrow เดียวในแต่ละครั้ง" (exactly one `&mut` borrow, or many `&` borrows) **เอง ที่ runtime** แทนที่
borrow checker จะตรวจให้ที่ compile time (สลับ compile-time check เป็น runtime check — นี่คือคำจำกัดความของ
interior mutability ที่ Part 28 สอนไว้)

กลไกภายในของ `RefCell<T>` คือมันเก็บ **ตัวนับจำนวนการ borrow ปัจจุบัน** ไว้ใน field ที่ชื่อประมาณ
`borrow: Cell<BorrowFlag>` — ทุกครั้งที่เรียก `.borrow()` ตัวนับจะเพิ่มขึ้น (นับ shared borrow) ทุกครั้งที่
เรียก `.borrow_mut()` ตัวนับจะถูกตั้งเป็นค่าพิเศษที่หมายถึง "มี exclusive borrow อยู่" และทุกครั้งที่ `Ref`/
`RefMut` (ค่าที่ได้จาก `.borrow()`/`.borrow_mut()`) ถูก drop ตัวนับจะถูกปรับกลับ — สังเกตว่า**นี่คือ
pattern เดียวกันเป๊ะกับตัวนับของ `Rc<T>` ในหัวข้อ 40.4**: อ่านค่า → คำนวณ → เขียนกลับ และก็**ไม่ใช่
atomic** เช่นกัน (เพราะ `RefCell<T>` ถูกออกแบบมาให้เป็น**ตัวที่เร็วที่สุด**สำหรับ single-thread โดยเฉพาะ —
atomic operation แม้จะไม่ได้ช้ามาก แต่ก็ยังช้ากว่า plain integer operation อย่างมีนัยสำคัญเมื่อเรียกบ่อยขนาดนี้
Part 51 เรื่อง Atomics จะเจาะรายละเอียดต้นทุนนี้เพิ่ม)

ผลที่ตามมาแยกเป็นสองกรณีตามสอง trait ของเราพอดี:

- **`RefCell<T>` เป็น `Send`** (เมื่อ `T: Send`) — เพราะการ**ย้าย** `RefCell<T>` ทั้งก้อนไปอีก thread ไม่มี
  ปัญหาอะไรเลย มันจะมีเจ้าของเดียวเสมอ (thread ปลายทาง) ตัวนับ borrow ก็ยังโดนแก้ไขจาก thread เดียวเท่านั้น
  ตลอดชีวิตของมัน ไม่มีทางเกิด race
- **`RefCell<T>` ไม่ใช่ `Sync`** — เพราะการ**แชร์** `&RefCell<T>` ให้สอง thread ถือพร้อมกัน แล้วให้ทั้งคู่
  เรียก `.borrow_mut()` พร้อมกันได้ (ซึ่ง type signature ของ `.borrow_mut()` คือ `&self` ไม่ใช่ `&mut self`
  — จึงเรียกจาก `&RefCell<T>` ที่แชร์กันได้จริง!) จะทำให้ตัวนับ borrow ภายในถูกแก้ไขพร้อมกันจากสอง thread —
  **race condition แบบเดียวกันเป๊ะกับตัวนับของ `Rc<T>`** ผลลัพธ์ที่แย่ที่สุดคือตัวนับผิดพลาดจนกฎ "borrow
  เดียวในแต่ละครั้ง" พังไปด้วย เช่นสอง thread อาจได้ `RefMut` (exclusive mutable access) พร้อมกันทั้งคู่
  ทั้งที่กฎ aliasing ของ Rust (Part 7) ห้ามมี `&mut` สองตัวชี้ข้อมูลเดียวกันพร้อมกันเด็ดขาด — เมื่อกฎนี้พัง
  จะเปิดช่องให้เกิด **undefined behavior** เต็มรูปแบบ (compiler จะ optimize โค้ดโดยสมมติว่ากฎ aliasing เป็น
  จริงเสมอ ถ้ามันไม่จริงจริง ๆ ผลลัพธ์ที่ได้จะคาดเดาไม่ได้เลย)

มาพิสูจน์ทั้งสองข้อด้วยโค้ดจริง:

```rust
use std::cell::RefCell;

fn assert_send<T: Send>() {}

fn main() {
    // RefCell<i32> เป็น Send: ย้ายทั้งก้อนไปเธรดอื่นได้ ไม่มีปัญหา (เป็นเจ้าของแค่เธรดเดียวเสมอ)
    assert_send::<RefCell<i32>>();
    println!("RefCell<i32> เป็น Send");
}
```

ผลลัพธ์: `RefCell<i32> เป็น Send` — คอมไพล์ผ่าน ยืนยันข้อแรก

```rust
use std::cell::RefCell;

fn assert_sync<T: Sync>() {}

fn main() {
    assert_sync::<RefCell<i32>>();
}
```

Error จริง (ยืนยันข้อสอง):

```
error[E0277]: `RefCell<i32>` cannot be shared between threads safely
 --> src/main.rs:6:19
  |
6 |     assert_sync::<RefCell<i32>>();
  |                   ^^^^^^^^^^^^ `RefCell<i32>` cannot be shared between threads safely
  |
  = help: the trait `Sync` is not implemented for `RefCell<i32>`
  = note: if you want to do aliasing and mutation between multiple threads, use `std::sync::RwLock` instead
```

สังเกตบรรทัดสุดท้ายที่ compiler ใจดีบอกทางออกให้เลย: **"ถ้าอยากทำ aliasing และ mutation ข้ามหลาย thread
ให้ใช้ `std::sync::RwLock` แทน"** — `RwLock<T>` คือ "`RefCell<T>` เวอร์ชัน thread-safe" (ใช้ atomic
operation แทนตัวนับธรรมดา และมี blocking behavior แบบ `Mutex<T>` แต่แยก reader/writer lock) เราจะพบ
`RwLock<T>` อีกครั้งใน Part 51 (Atomics และ Lock-free Programming) — ตอนนี้ให้จำแค่ว่า
**"`RefCell<T>` (single-thread) มี `Mutex<T>`/`RwLock<T>` (multi-thread) เป็นคู่หูที่แก้ปัญหา atomic
ให้เรียบร้อยแล้ว"** ซึ่งเป็นเหตุผลตรงที่ Part 39 ต้องเปลี่ยนจาก `RefCell<T>` (Part 28) มาเป็น `Mutex<T>`
ทันทีที่ข้ามไปสู่บริบท multi-thread — ไม่ใช่เรื่องบังเอิญ แต่เป็นผลตรงจากกฎ `Sync` ที่เราพิสูจน์ในหัวข้อนี้เป๊ะ ๆ

### 40.8 Auto Trait คำนวณอย่างไร: กฎ structural และ recursive แบบเต็ม

ตอนนี้เรามาดูกฎการคำนวณแบบเป็นทางการ compiler ใช้กฎนี้กับ**ทุก struct, enum, และ tuple** ที่คุณเขียนขึ้นมา
โดยไม่ต้องเขียน `impl Send`/`impl Sync` เองแม้แต่ครั้งเดียว:

> **struct/enum/tuple ชนิด `T` จะเป็น `Send` (หรือ `Sync`) โดยอัตโนมัติ ก็ต่อเมื่อ field ทุกตัวภายใน `T`
> เป็น `Send` (หรือ `Sync`) ทั้งหมด** — เป็นการตรวจแบบ **recursive**: field ที่เป็น struct ซ้อนอยู่ข้างใน
> ก็ต้องถูกตรวจ field ของมันเองต่อไปอีกที จนกว่าจะถึง type พื้นฐานที่ std กำหนดไว้ตรง ๆ (เช่น `i32: Send +
> Sync`, `*const T: !Send + !Sync`, `Rc<T>: !Send`, ฯลฯ)

นี่คือเหตุผลที่ `Order` ในหัวข้อ 40.3 เป็น `Send + Sync` แบบไม่ต้องเขียนอะไรเพิ่ม — field ทั้งสาม (`u32`,
`String`, `Vec<String>`) ล้วนเป็น `Send + Sync` ทั้งหมด ทำให้ `Order` ก็เป็นไปด้วย ในทางกลับกัน มาดูตัวอย่าง
ที่ struct มี field เพียง**หนึ่งตัว**ที่ไม่ใช่ `Send` — ผลคือ struct ทั้งก้อน**ไม่ใช่ `Send`** ทันที (กฎ "ทุก
field ต้องผ่าน" คือ AND ทางตรรกะ ตัวเดียวไม่ผ่านคือทั้งก้อนไม่ผ่าน):

```rust
use std::rc::Rc;

fn assert_send<T: Send>() {}

struct Cache {
    id: u32,
    shared_data: Rc<Vec<String>>,
}

fn main() {
    assert_send::<Cache>();
}
```

Error จริง:

```
error[E0277]: `Rc<Vec<String>>` cannot be sent between threads safely
  --> src/main.rs:11:19
   |
11 |     assert_send::<Cache>();
   |                   ^^^^^ `Rc<Vec<String>>` cannot be sent between threads safely
   |
   = help: within `Cache`, the trait `Send` is not implemented for `Rc<Vec<String>>`
note: required because it appears within the type `Cache`
  --> src/main.rs:5:8
   |
 5 | struct Cache {
   |        ^^^^^
note: required by a bound in `assert_send`
```

สังเกตบรรทัด `note: required because it appears within the type \`Cache\`` — นี่คือ compiler กำลังไล่ตาม
กฎ recursive ให้เห็นตรง ๆ: มันหา field `shared_data: Rc<Vec<String>>` เจอว่าไม่ใช่ `Send` แล้วสรุปว่า
`Cache` (ที่มี field นี้อยู่ภายใน) ก็ไม่ใช่ `Send` ไปด้วย ต่อให้ field อื่นอย่าง `id: u32` เป็น `Send`
สมบูรณ์แบบก็ตาม — field เดียวที่พังก็พอทำให้ทั้ง struct พังไปด้วย

ประเด็นสำคัญที่ต้องเน้น: **นี่ไม่ใช่ error ที่เกิดจากการ "ลืมเขียน `impl Send`"** เพราะคุณไม่มีทางเขียน
`impl Send for Cache {}` เองได้อยู่แล้ว (ยกเว้นด้วย `unsafe impl` ซึ่งจะพูดถึงในหัวข้อถัดไป) — มันคือ error
ที่บอกว่า **โครงสร้างข้อมูลของคุณเองมี field ที่ไม่ปลอดภัยสำหรับ concurrency ซ่อนอยู่** ทางแก้ที่ถูกต้อง
ในสถานการณ์จริงคือเปลี่ยน field จาก `Rc<Vec<String>>` เป็น `Arc<Vec<String>>` (ถ้าต้องการ multiple
ownership ข้ามเธรด) — โครงสร้าง auto trait จะคำนวณใหม่ให้อัตโนมัติทันทีที่ field เปลี่ยน โดยไม่ต้องแก้อะไรที่
ระดับ `Cache` เองเลย:

```rust
use std::sync::Arc;

fn assert_send<T: Send>() {}
fn assert_sync<T: Sync>() {}

struct Cache {
    id: u32,
    shared_data: Arc<Vec<String>>,
}

fn main() {
    // แค่เปลี่ยน Rc -> Arc, compiler คำนวณ Send/Sync ใหม่ให้อัตโนมัติทันที ไม่ต้องแก้ Cache เลย
    assert_send::<Cache>();
    assert_sync::<Cache>();
    println!("Cache เป็นทั้ง Send และ Sync แล้ว หลังเปลี่ยนเป็น Arc");
}
```

นี่คือพลังของการที่ `Send`/`Sync` เป็น auto trait: มันทำให้ concurrency safety ของโปรแกรมทั้งหมด
**ประกอบขึ้นจากล่างขึ้นบน (compositional)** โดยอัตโนมัติ — คุณไม่ต้องไล่ทวน struct ทุกตัวที่ใช้ `Cache`
เพื่อเช็คว่ายัง `Send` อยู่ไหม compiler ทำให้หมดทั้ง dependency graph ในโปรแกรมคุณ

กฎ structural นี้ใช้กับ **`enum`** แบบเดียวกันเป๊ะ — ต่างกันแค่ตรงที่ enum ต้องดูทุก field ของ**ทุก
variant**รวมกัน (ไม่ใช่แค่ variant ที่ถูกใช้งานจริงตอนนั้น เพราะ enum หนึ่งตัวสามารถเป็นได้ทุก variant
ตลอดชีวิตของมัน compiler จึงต้องตรวจแบบ conservative ครอบคลุมทุกความเป็นไปได้):

```rust
fn assert_send<T: Send>() {}

enum Message {
    Text(String),
    Number(i32),
}

fn main() {
    assert_send::<Message>();
    println!("Message เป็น Send");
}
```

ผลลัพธ์: `Message เป็น Send` — ทั้งสอง variant (`String` และ `i32`) เป็น `Send` ทั้งคู่ enum จึงเป็น `Send`
ไปด้วย แต่ถ้ามี variant เดียวที่มี field ไม่ใช่ `Send` แอบอยู่ (แม้จะเป็น variant ที่ "ไม่ได้ใช้บ่อย" ก็ตาม)
enum ทั้งก้อนก็เสีย `Send` ไปเหมือนกับกรณี struct:

```rust
use std::rc::Rc;

fn assert_send<T: Send>() {}

enum BadMessage {
    Text(String),
    Cached(Rc<String>),
}

fn main() {
    assert_send::<BadMessage>();
}
```

Error จริง:

```
error[E0277]: `Rc<String>` cannot be sent between threads safely
  --> src/main.rs:11:19
   |
11 |     assert_send::<BadMessage>();
   |                   ^^^^^^^^^^ `Rc<String>` cannot be sent between threads safely
   |
   = help: within `BadMessage`, the trait `Send` is not implemented for `Rc<String>`
note: required because it appears within the type `BadMessage`
  --> src/main.rs:5:6
   |
 5 | enum BadMessage {
   |      ^^^^^^^^^^
note: required by a bound in `assert_send`
```

ข้อคิดเชิงออกแบบที่ได้จากตัวอย่างนี้: ถ้าคุณมี enum ขนาดใหญ่ที่รวม variant หลากหลายไว้ด้วยกัน (เช่น enum
สำหรับ message queue หรือ event system) การมี variant เดียวที่ผ่าน `Rc<T>` เข้ามาโดยไม่ตั้งใจ จะทำให้
enum ทั้งตัว "ติดโรค" ไม่ใช่ `Send` ไปด้วย ต่อให้ variant อื่น ๆ ปลอดภัยทุกตัวก็ตาม — เวลาออกแบบ enum ที่
ตั้งใจให้ส่งข้าม thread ได้ (เช่น เป็น message type ของ channel จาก Part 38) ควรตรวจทุก variant ให้เป็น
`Send` ตั้งแต่ต้น ด้วยเทคนิค `assert_send::<T>()` ก่อนจะนำไปใช้งานจริง

### 40.9 Opt-out จาก auto trait: เมื่อ field ปลอดภัย แต่ type ไม่ควรเป็น Send/Sync

บางครั้ง field ทั้งหมดของ type เป็น `Send`/`Sync` ตามกฎ แต่**ตัว type เองไม่ควรเป็น** เพราะ invariant
ด้าน thread-safety ที่ compiler มองไม่เห็นจาก field เฉย ๆ — ตัวอย่างคลาสสิกคือ wrapper รอบ handle ของ
OS resource บางชนิด (เช่น handle ของ GUI บางระบบที่ผูกกับ thread ที่สร้างมันขึ้นมาเท่านั้น) หรือ type ที่
ทีมพัฒนา std ใช้ทำ FFI (Foreign Function Interface — จะเรียนใน Part 43) ที่ผูกกับ library ภาษา C ซึ่งไม่
thread-safe

Rust มีสองวิธีสำหรับ opt-out จาก auto trait นี้:

**วิธีที่ 1 (nightly-only, ยังไม่ stable): `impl !Send for MyType {}`** — เขียนตรง ๆ ว่า "ห้าม auto-derive
`Send` ให้ type นี้" ต้องเปิด unstable feature `#![feature(negative_impls)]` ก่อน (ปัจจุบันยังไม่ stabilize
ใน Rust edition 2021/2024) วิธีนี้เป็นวิธีที่ std เองใช้ประกาศว่า `Rc<T>` ไม่ใช่ `Send`/`Sync` ภายใน
source code จริง ๆ ของมัน (`impl<T: ?Sized> !Send for Rc<T> {}`) แต่สำหรับโค้ดของผู้ใช้ทั่วไปบน stable
Rust วิธีนี้ยังใช้ไม่ได้

**วิธีที่ 2 (stable, ใช้งานได้จริงตอนนี้): แอบใส่ field ที่รู้อยู่แล้วว่าไม่ใช่ `Send`/`Sync` ผ่าน
`PhantomData`** — ทวนจาก Part 22 ว่า `PhantomData<T>` คือ marker type ที่ไม่กิน memory จริง (`zero-sized
type`) ใช้บอก compiler ว่า "type นี้เกี่ยวข้องกับ `T` ในเชิง type-level" โดยไม่ต้องมีค่า `T` จริงเก็บอยู่
เทคนิคคือใส่ `PhantomData<*const ()>` เป็น field หนึ่ง — เพราะ **raw pointer (`*const T`, `*mut T`)
ไม่ใช่ `Send` และไม่ใช่ `Sync` ตามกฎของ std** (เหตุผล: raw pointer ไม่มีการันตีอะไรเรื่อง aliasing หรือ
lifetime เลย ปล่อยให้เป็นความรับผิดชอบของโค้ด `unsafe` ทั้งหมด — Part 41 จะลงรายละเอียด) เมื่อ
`PhantomData<*const ()>` ไม่ใช่ `Send`/`Sync` กฎ structural ในหัวข้อ 40.8 ก็ทำให้ struct ที่มี field นี้
**ไม่ใช่ `Send`/`Sync` ไปด้วยโดยอัตโนมัติ** — ได้ผลลัพธ์เดียวกับ `impl !Send` แต่ใช้ได้บน stable Rust:

```rust
use std::marker::PhantomData;

fn assert_send<T: Send>() {}

// PhantomData<*const ()> ทำให้ compiler คำนวณว่า type นี้ไม่ใช่ Send/Sync
// เพราะ raw pointer (*const T) ไม่ใช่ Send และไม่ใช่ Sync ไปตามกฎ auto trait เชิงโครงสร้าง
struct NotThreadSafe {
    value: i32,
    _marker: PhantomData<*const ()>,
}

fn main() {
    assert_send::<NotThreadSafe>();
}
```

Error จริง:

```
error[E0277]: `*const ()` cannot be sent between threads safely
  --> src/main.rs:13:19
   |
13 |     assert_send::<NotThreadSafe>();
   |                   ^^^^^^^^^^^^^ `*const ()` cannot be sent between threads safely
   |
   = help: within `NotThreadSafe`, the trait `Send` is not implemented for `*const ()`
note: required because it appears within the type `PhantomData<*const ()>`
note: required because it appears within the type `NotThreadSafe`
  --> src/main.rs:7:8
   |
 7 | struct NotThreadSafe {
   |        ^^^^^^^^^^^^^
note: required by a bound in `assert_send`
```

แต่ `value: i32` ยังใช้งานได้ตามปกติทุกอย่างในเธรดเดียว — `PhantomData` ไม่กระทบ runtime behavior เลยแม้
แต่นิดเดียว มันแค่เปลี่ยนผลการคำนวณ auto trait ที่ compile time:

```rust
use std::marker::PhantomData;

struct NotThreadSafe {
    value: i32,
    _marker: PhantomData<*const ()>,
}

impl NotThreadSafe {
    fn new(value: i32) -> Self {
        NotThreadSafe { value, _marker: PhantomData }
    }
    fn get(&self) -> i32 {
        self.value
    }
}

fn main() {
    let x = NotThreadSafe::new(10);
    println!("{}", x.get());
}
```

ผลลัพธ์: `10` — ทำงานปกติทุกอย่างในเธรดเดียว เพียงแต่ตอนนี้ compiler จะปฏิเสธทันทีถ้ามีใครพยายามส่ง
`NotThreadSafe` เข้า `thread::spawn`

เทคนิคนี้**ไม่ได้ลึกมากในระดับที่ต้องใช้บ่อย** สำหรับตอนนี้ให้จำแค่ระดับ "รู้จักว่ามีอยู่" — มันจะสำคัญมากขึ้น
เมื่อเราเข้าสู่ Part 41 (Unsafe Rust) ที่คุณจะเริ่มเขียน type ที่ห่อ raw pointer เอง และต้องตัดสินใจด้วยตัวเอง
ว่า type นั้นควรเป็น `Send`/`Sync` หรือไม่ (บางครั้งถึงกับต้องทำตรงข้าม คือ**เพิ่ม** `Send`/`Sync` ให้ type
ที่มี raw pointer อยู่ข้างใน ด้วย `unsafe impl Send for MyType {}` เมื่อคุณมั่นใจ 100% ว่ามันปลอดภัยจริง
ทั้งที่ field ไม่ผ่านกฎ auto trstatic — นี่คือเหตุผลที่ `Send`/`Sync` ต้องเป็น `unsafe trait`)

### 40.10 ลายเซ็นเต็มของ `thread::spawn`: ตอนนี้อ่านได้ครบทุกส่วนแล้ว

ใน Part 37 เราใช้ `thread::spawn` มาแล้วหลายครั้งโดยยังไม่เห็นลายเซ็นเต็มของมัน ตอนนี้เรามีความรู้พอที่จะอ่าน
มันได้ครบทุกตัวอักษร ลายเซ็นจริงจาก standard library คือ:

```rust
use std::thread::JoinHandle;

// นี่คือลายเซ็นจริงของ std::thread::spawn (ตรงกับ standard library)
// เขียนใหม่ในตัวอย่างนี้เพื่อให้ compile ได้จริงและตรวจ syntax ได้ครบ
fn spawn<F, T>(f: F) -> JoinHandle<T>
where
    F: FnOnce() -> T,
    F: Send + 'static,
    T: Send + 'static,
{
    let _ = f;
    unimplemented!()
}

fn main() {
    // เรียกด้วย closure จริงเพื่อให้ compiler ตรวจ type inference และ bound ทั้งหมดผ่านจริง
    // (ไม่ execute เพราะ body คือ unimplemented!() แต่ syntax/type-check ผ่าน 100%)
    let _typecheck_only = || {
        let _handle: JoinHandle<i32> = spawn(|| 5);
    };
}
```

มาแยกอธิบายทุก bound ทีละส่วน — ตอนนี้แต่ละคำมีความหมายเต็มที่เชื่อมกับสิ่งที่เรียนมาทั้งหมด:

- **`F: FnOnce() -> T`** — closure ที่ส่งเข้าไปต้อง**เรียกได้อย่างน้อยหนึ่งครั้ง** (ทวนจาก Part 24: `FnOnce`
  คือ trait ที่กว้างที่สุดในสามตัว `Fn`/`FnMut`/`FnOnce` เพราะ closure ที่ implement `Fn` หรือ `FnMut`
  ก็ implement `FnOnce` ได้เสมอ) เหตุผลที่ต้องเป็น `FnOnce` (ไม่ใช่ `Fn` หรือ `FnMut`) คือ thread ใหม่จะ
  เรียก closure นี้**แค่ครั้งเดียวเท่านั้น**ตลอดชีวิตของ thread นั้น (thread หนึ่งรันฟังก์ชันเดียวจบแล้วก็จบ
  ชีวิต) การเรียกร้อง `FnOnce` เท่านั้น (แทน `Fn` ที่เข้มงวดกว่า) ทำให้ closure สามารถ**ย้าย
  ownership**ของสิ่งที่ capture มาออกไปใช้ได้เต็มที่ (เช่น `move` ค่าออกจาก closure ไปคืนเป็น return value
  `T`) ซึ่งจำเป็นมากในบริบท thread — ถ้าบังคับเป็น `Fn` closure จะเรียกซ้ำได้หลายครั้ง แต่ก็ห้าม consume
  ค่าที่ capture มา ซึ่งจำกัดการใช้งานเกินจำเป็น
- **`F: Send`** — closure เองต้องเป็น `Send` เพราะ closure ก็คือ struct ที่ไม่มีชื่อ (unnamed struct) ที่
  compiler สร้างขึ้นเก็บทุกตัวแปรที่ capture มาไว้เป็น field (ทวนจาก Part 24) กฎ auto trait ในหัวข้อ 40.8
  จึงใช้กับมันตรง ๆ: **closure จะเป็น `Send` ก็ต่อเมื่อทุกอย่างที่มัน capture มาเป็น `Send` ทั้งหมด** —
  นี่คือ bound ที่ปฏิเสธ `Rc<T>` ในหัวข้อ 40.4 พอดี เพราะ closure ที่ capture `Rc<T>` มาไม่ใช่ `Send` ตามกฎ
  นี้เอง
- **`T: Send`** — **ค่าที่ return ออกมา**จาก closure (เก็บไว้ให้ดึงออกมาผ่าน `handle.join()`) ก็ต้องเป็น
  `Send` ด้วย เพราะค่านั้นถูกสร้างขึ้นใน thread ลูก แต่ต้องถูก**ย้าย**กลับมาที่ thread หลักตอนเรียก `.join()`
  — เป็นการ "ย้ายข้ามเธรด" อีกครั้งหนึ่ง (คนละทิศทางกับ closure) จึงต้องใช้ bound `Send` เหมือนกัน
- **`F: 'static`** (และ `T: 'static`) — closure (และค่าที่มันสร้าง) ต้องไม่มี reference ที่อ้างถึงข้อมูลที่
  อาจถูก drop ไปก่อน ทวนจาก Part 23: `'static` ไม่ได้แปลว่า "อยู่ตลอดโปรแกรม" เสมอไป แต่แปลว่า **"ไม่มี
  lifetime constraint ที่ผูกกับ scope ใด scope หนึ่งโดยเฉพาะ"** — ข้อมูลที่ `move` เข้าไปใน closure ทั้งก้อน
  (owned data เช่น `String`, `Vec<T>`, หรือค่าที่ clone มา) ถือว่าเป็น `'static` เพราะไม่มีใครอื่นเป็นเจ้าของ
  มันพร้อมกัน จึงไม่มีทาง dangling ได้ — แต่ borrowed reference ที่มี lifetime ผูกกับตัวแปร local ใน
  ฟังก์ชันที่เรียก `spawn` (เช่น `&local_var`) จะไม่ผ่าน bound นี้ เพราะ thread ใหม่**อาจ**ทำงานนานกว่าฟังก์ชัน
  ที่เรียกมันอยู่ (thread หลักอาจ return ไปก่อน ตัวแปร local ถูก drop ก่อน thread ลูกยังรันไม่จบ) — นี่คือ
  ต้นตอของ error E0373 ที่เจอใน Part 37 พอดี

สรุปเป็นประโยคเดียว: **`thread::spawn` รับ closure ที่ (ก) เรียกได้แค่ครั้งเดียว (ข) ทุกอย่างที่มันถือครอง
ย้ายข้ามเธรดได้อย่างปลอดภัย และ (ค) ไม่มี reference ห้อยที่อาจ dangling ระหว่างที่ thread ยังรันอยู่** —
ลายเซ็นเดียวนี้ครอบคลุมทั้งสามมิติของความปลอดภัย concurrency (memory safety ของ ownership, memory safety
ของ data race, และ memory safety ของ lifetime) พร้อมกันหมดในบรรทัดเดียว นี่คือเหตุผลที่วิศวกร Rust มักพูดว่า
"ถ้า code ของคุณ compile ผ่าน `thread::spawn` ได้ มันมักจะไม่มีบั๊ก concurrency แบบคลาสสิกที่เจอในภาษาอื่นเลย"

**`JoinHandle<T>` เองก็เป็น `Send`/`Sync` ตามกฎเดียวกัน**

สิ่งที่ `thread::spawn` คืนกลับมาคือ `JoinHandle<T>` — handle ที่ใช้เรียก `.join()` เพื่อรอ thread จบและ
ดึงค่า return กลับมา สังเกตว่า `JoinHandle<T>` เองก็เป็น struct ธรรมดาที่ต้องผ่านกฎ auto trait เหมือนกับทุก
type อื่น มันเก็บ handle ของ OS thread ไว้ภายใน (ผ่าน layer ของระบบปฏิบัติการที่ออกแบบมาให้ปลอดภัยข้าม
thread โดยธรรมชาติ) ทำให้ `JoinHandle<T>` เป็นทั้ง `Send` และ `Sync` เสมอ ไม่ว่า `T` จะเป็นอะไรก็ตาม
(เพราะ `JoinHandle<T>` ไม่ได้เก็บค่า `T` ไว้ตรง ๆ ตั้งแต่ต้น — มันแค่รู้วิธี "รอและดึงค่า" ออกมาตอน `.join()`
เท่านั้น) ซึ่งเป็นเหตุผลที่ Part 37 สามารถเก็บ `Vec<JoinHandle<T>>` แล้วส่งต่อไปมาระหว่างฟังก์ชันได้อย่าง
อิสระโดยไม่มี compile error เรื่อง `Send`/`Sync` โผล่มาให้แก้เลย:

```rust
use std::thread::JoinHandle;

fn assert_send<T: Send>() {}
fn assert_sync<T: Sync>() {}

fn main() {
    assert_send::<JoinHandle<i32>>();
    assert_sync::<JoinHandle<i32>>();
    println!("JoinHandle<i32> เป็นทั้ง Send และ Sync");
}
```

### 40.11 `Arc<Mutex<T>>` คือทั้ง `Send` และ `Sync`: "final boss" ที่ทำให้ทุกอย่างเข้าที่

ตอนนี้เรามีความรู้พอที่จะอธิบาย pattern ที่ใช้บ่อยที่สุดใน Rust concurrency — `Arc<Mutex<T>>` — แบบเจาะ
กลไกได้ครบทุกชั้น มาแยกวิเคราะห์สองชั้นของมันทีละตัว

**ชั้นนอก: `Arc<T>` แก้ปัญหา `Send` ของ `Rc<T>` ด้วย atomic reference counting**

ทวนจาก Part 29 (Weak/Cow) และ Part 39: `Arc<T>` ("**A**tomically **R**eference **C**ounted") มี API
เหมือน `Rc<T>` ทุกอย่าง (clone เพื่อเพิ่มเจ้าของ, drop เพื่อลดจำนวน) **ยกเว้นกลไกภายในของตัวนับ** — `Arc<T>`
ใช้ **atomic integer** (เช่น `AtomicUsize`) แทน `Cell<usize>` ธรรมดา atomic operation เป็น operation
ระดับ CPU ที่**การันตีว่าแบ่งแยกไม่ได้ (indivisible)** แม้จะมีหลาย CPU core พยายามแก้ไขค่าเดียวกันพร้อมกันจริง ๆ
— ฮาร์ดแวร์เองมีกลไกป้องกันไม่ให้สอง core อ่าน-เขียนทับกันแบบที่เกิดกับตัวนับธรรมดาของ `Rc<T>` ในหัวข้อ 40.4
เพราะฉะนั้น `Arc<T>` จึงแก้ปัญหา `Send` ได้ตรงจุด: **การย้าย/clone `Arc<T>` ข้ามเธรดปลอดภัย เพราะตัวนับที่
ทุกเธรดแก้ไขร่วมกันเป็น atomic** (std ประกาศ `unsafe impl<T: Send + Sync> Send for Arc<T> {}` — สังเกตว่า
ต้องมี `T: Sync` ด้วย ไม่ใช่แค่ `T: Send` เหตุผลจะชัดในหัวข้อถัดไป)

**ชั้นใน: `Mutex<T>` แก้ปัญหา `Sync` ด้วยการล็อก (locking)**

ทวนจาก Part 39: `Mutex<T>` ("**Mut**ual **Ex**clusion") ให้ **interior mutability ที่ปลอดภัยข้ามเธรด**
โดยใช้ `.lock()` เพื่อขอสิทธิ์เข้าถึงข้อมูลภายในแบบ**เธรดเดียวในแต่ละเวลา (exclusive access)** — ถ้ามีอีก
เธรดถือ lock อยู่แล้ว เธรดที่เรียก `.lock()` จะ**block (หยุดรอ)** จนกว่า lock จะถูกปล่อย นี่คือกลไกที่แก้
ปัญหาของ `RefCell<T>` ในหัวข้อ 40.7 ได้ตรงจุด: **`RefCell<T>` ตรวจกฎ borrow แบบ non-blocking (ถ้าผิดกฎก็
panic ทันที) และใช้ counter ที่ไม่ atomic — ส่วน `Mutex<T>` ตรวจกฎแบบ blocking (รอจนกว่าจะปลอดภัย) และใช้
sychronization primitive ที่ OS/hardware การันตีความถูกต้องข้ามเธรดจริง ๆ** ผลคือ `Mutex<T>` เป็น `Sync`
(เมื่อ `T: Send`) เพราะการแชร์ `&Mutex<T>` ให้หลายเธรดแล้วให้ทุกเธรดแย่งกันเรียก `.lock()` เป็นเรื่องปลอดภัย
โดยออกแบบ — ไม่มีทางสอง thread ได้ exclusive access พร้อมกันเด็ดขาด เพราะ lock ทำหน้าที่**บังคับให้ต้องรอ
คิว**เสมอ

**รวมสองชั้นเข้าด้วยกัน**

`Arc<Mutex<T>>` จึงแก้ปัญหาคนละครึ่งอย่างพอดิบพอดี:

- `Arc<...>` (ชั้นนอก) → แก้ปัญหา **"หลายเธรดเป็นเจ้าของร่วมกันได้อย่างไร"** (multiple ownership ข้ามเธรด
  ด้วย atomic refcount) — นี่คือมิติของ **`Send`**
- `Mutex<T>` (ชั้นใน) → แก้ปัญหา **"หลายเธรดแก้ไขข้อมูลเดียวกันได้อย่างปลอดภัยอย่างไร"** (exclusive mutable
  access ผ่าน lock) — นี่คือมิติของ **`Sync`**

ผลลัพธ์คือ `Arc<Mutex<T>>` (เมื่อ `T: Send`) เป็นทั้ง **`Send` และ `Sync`** พร้อมกัน — ส่งไปเธรดใหม่ได้
(`Send`) และแชร์ให้หลายเธรดถือ `&Arc<Mutex<T>>` พร้อมกันได้ (`Sync`) พิสูจน์ด้วยโค้ด:

```rust
use std::sync::{Arc, Mutex};

fn assert_send<T: Send>() {}
fn assert_sync<T: Sync>() {}

fn main() {
    // Arc<Mutex<i32>> เป็นทั้ง Send และ Sync เพราะ i32: Send
    assert_send::<Arc<Mutex<i32>>>();
    assert_sync::<Arc<Mutex<i32>>>();
    println!("Arc<Mutex<i32>> เป็นทั้ง Send และ Sync");
}
```

และนี่คือโปรแกรมเต็มที่ Part 39 เคยแสดง มาดูอีกครั้งพร้อมความเข้าใจกลไกครบทุกชั้นแล้ว:

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..10 {
        // Arc::clone ปลอดภัยข้ามเธรด เพราะ atomic refcount (มิติ Send ของ Arc<T>)
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            for _ in 0..1000 {
                // .lock() ปลอดภัยเมื่อมีหลายเธรดเรียกพร้อมกัน เพราะ blocking mutual exclusion (มิติ Sync ของ Mutex<T>)
                let mut num = counter.lock().unwrap();
                *num += 1;
            }
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("ผลรวมสุดท้าย: {}", *counter.lock().unwrap());
    assert_eq!(*counter.lock().unwrap(), 10_000);
}
```

ผลลัพธ์ (ยืนยันด้วย `assert_eq!` ในโค้ดเอง):

```
ผลรวมสุดท้าย: 10000
```

ทุกครั้งที่รันจะได้ `10000` เป๊ะ (ไม่ใช่ค่าสุ่มที่คลาดเคลื่อนแบบที่จะเกิดถ้าไม่มี `Mutex` ป้องกัน — ลองนึกภาพ
ถ้าใช้ `Rc<RefCell<i32>>` แทน มันจะไม่ผ่าน compile time ด้วยซ้ำ (ทั้งสองไม่ใช่ `Send`) ดังนั้นในภาษา Rust
คุณ**ไม่มีทาง**ได้ผลลัพธ์ที่คลาดเคลื่อนแบบ `9987` หรือ `10003` จาก race condition นี้เลย — เพราะโค้ดที่มี
โอกาสเกิด race แบบนี้จะไม่ผ่าน compile ตั้งแต่แรก นี่คือความหมายที่แท้จริงของ "fearless concurrency"
ที่พูดถึงตั้งแต่หัวข้อ 40.1

### 40.11.1 กับดักคลาสสิก: `Arc<RefCell<T>>` ไม่ใช่คำตอบ (แม้จะดูเหมือนใช่)

ก่อนไปดูตารางเปรียบเทียบ มีความเข้าใจผิดหนึ่งข้อที่พบบ่อยมากพอที่ต้องแยกพูดเป็นหัวข้อของตัวเอง: นักพัฒนา
ที่มาจาก Part 28 (ที่ใช้ `Rc<RefCell<T>>` คล่องมือแล้ว) มักคิดว่าแค่เปลี่ยน `Rc` เป็น `Arc` (แต่ยังใช้
`RefCell` อยู่ข้างใน) ก็พอแล้วสำหรับ multi-thread — คือใช้ `Arc<RefCell<T>>` ทั้งที่ควรเป็น
`Arc<Mutex<T>>` มาดูว่าเกิดอะไรขึ้นเมื่อลองพิสูจน์ด้วย `assert_sync`:

```rust
use std::cell::RefCell;
use std::sync::Arc;

fn assert_sync<T: Sync>() {}

fn main() {
    assert_sync::<Arc<RefCell<i32>>>();
}
```

Error จริง:

```
error[E0277]: `RefCell<i32>` cannot be shared between threads safely
 --> src/main.rs:7:19
  |
7 |     assert_sync::<Arc<RefCell<i32>>>();
  |                   ^^^^^^^^^^^^^^^^^ `RefCell<i32>` cannot be shared between threads safely
  |
  = help: the trait `Sync` is not implemented for `RefCell<i32>`
  = note: if you want to do aliasing and mutation between multiple threads, use `std::sync::RwLock` instead
  = note: required for `Arc<RefCell<i32>>` to implement `Sync`
note: required by a bound in `assert_sync`
```

ไม่น่าแปลกใจ — `Arc<T>` เป็น `Sync` ก็ต่อเมื่อ `T: Send + Sync` (ตามหัวข้อ 40.11) และ `RefCell<i32>` ไม่ใช่
`Sync` (ตามหัวข้อ 40.7) ดังนั้น `Arc<RefCell<i32>>` จึงไม่ใช่ `Sync` ไปด้วย — แต่สิ่งที่น่าประหลาดใจกว่าคือ
**`Arc<RefCell<i32>>` ไม่ใช่ `Send` ด้วยเช่นกัน!**

```
error[E0277]: `RefCell<i32>` cannot be shared between threads safely
 --> src/main.rs:7:19
  |
7 |     assert_send::<Arc<RefCell<i32>>>();
  |                   ^^^^^^^^^^^^^^^^^ `RefCell<i32>` cannot be shared between threads safely
  |
  = help: the trait `Sync` is not implemented for `RefCell<i32>`
  = note: required for `Arc<RefCell<i32>>` to implement `Send`
```

สังเกตบรรทัดสุดท้าย `required for \`Arc<RefCell<i32>>\` to implement \`Send\`` — นี่คือหลักฐานตรงจาก
compiler ที่ยืนยันสิ่งที่พูดไว้ท้ายหัวข้อ 40.11: **`Arc<T>` ประกาศ `unsafe impl<T: Send + Sync> Send for
Arc<T> {}`** (สังเกตว่า bound คือ `T: Send + Sync` ทั้งคู่ ไม่ใช่แค่ `T: Send`) เหตุผลคือ `Arc<T>` ให้สิทธิ์
`&T` ผ่าน `Deref` ได้จากหลายเธรดพร้อมกันเสมอ (เพราะเป็นแก่นของการมี "เจ้าของร่วม") ดังนั้นการที่ `Arc<T>`
เองจะ `Send` ได้ (ย้ายจาก thread หนึ่งไปอีก thread ที่อาจถือ clone ตัวอื่นของ `Arc` เดียวกันอยู่แล้ว) ก็ต้อง
มั่นใจว่า `T` นั้น `Sync` ด้วย ไม่ใช่แค่ `Send` — ผลคือ `Arc<RefCell<T>>` **ไม่มีประโยชน์อะไรเลยในบริบท
multi-thread** มันจะถูก compiler ปฏิเสธตั้งแต่ก้าวแรกที่พยายามส่งหรือแชร์ข้าม thread ทางแก้เดียวคือเปลี่ยน
`RefCell<T>` เป็น `Mutex<T>` (หรือ `RwLock<T>`) เท่านั้น — ไม่มีทางลัดอื่น

### 40.12 ตารางเปรียบเทียบ: `Rc` vs `Arc` vs `RefCell` vs `Mutex`

ตารางนี้รวบรวมทั้ง smart pointer arc (Part 27–29) และ concurrency arc (Part 37–39) ของหลักสูตรทั้งหมดเข้า
เป็นภาพเดียว — ควรกลับมาดูตารางนี้ทุกครั้งที่ลังเลว่าจะเลือก type ไหนในสถานการณ์จริง:

| Type | `Send`? | `Sync`? | เหตุผลเชิงกลไก |
|---|---|---|---|
| `Rc<T>` | **ไม่** | **ไม่** | ตัวนับ refcount เป็น `Cell<usize>` ธรรมดา ไม่ใช่ atomic — สอง thread แก้ไขพร้อมกันจะเกิด race ทำตัวนับผิดพลาด (ดูหัวข้อ 40.4) เพราะไม่ใช่ `Send` การแชร์ `&Rc<T>` ก็ไม่ปลอดภัยไปด้วย (ถึงจะไม่ได้เกี่ยวกับ mutation ตรง ๆ แต่ std เลือก conservative ปฏิเสธทั้งคู่เพื่อความเรียบง่ายของ mental model) |
| `Arc<T>` | **ใช่** ถ้า `T: Send + Sync` | **ใช่** ถ้า `T: Send + Sync` | ตัวนับ refcount เป็น atomic integer — แก้ไขพร้อมกันจากหลายเธรดปลอดภัยโดยฮาร์ดแวร์การันตี (ดูหัวข้อ 40.11) ต้องการ `T: Sync` ด้วยเพราะ `&Arc<T>` ที่แชร์หลายเธรดให้ `&T` แบบแชร์ต่อไปได้ (ผ่าน `Deref`) |
| `RefCell<T>` | **ใช่** ถ้า `T: Send` | **ไม่** | ย้ายทั้งก้อนไปเธรดเดียวปลอดภัย (มีเจ้าของเดียวเสมอ) แต่ตัวนับ borrow ภายในไม่ใช่ atomic — แชร์ `&RefCell<T>` ให้สองเธรดเรียก `.borrow_mut()` พร้อมกันจะพังกฎ aliasing (ดูหัวข้อ 40.7) |
| `Mutex<T>` | **ใช่** ถ้า `T: Send` | **ใช่** ถ้า `T: Send` | ใช้ synchronization primitive ของ OS (futex/atomic) ควบคุมการเข้าถึงแบบ exclusive และ blocking — แชร์ `&Mutex<T>` ให้หลายเธรดปลอดภัยเพราะ lock บังคับคิวเสมอ (ดูหัวข้อ 40.11) สังเกตว่า `Mutex<T>` ไม่ต้องการ `T: Sync` เลย (แค่ `T: Send` พอ) เพราะตัว lock เองทำหน้าที่สร้างการเข้าถึงแบบเธรดเดียวให้อยู่แล้ว ไม่ต้องพึ่ง `T: Sync` ของข้อมูลภายใน |

จุดที่น่าสังเกตเป็นพิเศษ: `Mutex<T>` ต้องการแค่ `T: Send` (ไม่ต้อง `T: Sync`) ในการเป็น `Sync` ของตัวมันเอง
— นี่ฟังดูขัดกับสัญชาตญาณตอนแรก แต่เข้าใจได้ทันทีถ้านึกภาพว่า `Mutex<T>` ทำหน้าที่ "เปลี่ยนการเข้าถึงแบบ
`Sync` (หลายเธรดถือ `&T` พร้อมกัน) ให้กลายเป็นการเข้าถึงแบบ `Send` ทีละเธรด (ผ่าน lock)" — มันคือ**เครื่องมือ
แปลง `Send` เป็น `Sync`** โดยตัวมันเอง จึงไม่จำเป็นต้องพึ่ง `T: Sync` อยู่แล้วเป็นทุนเดิม (แต่ก็ยังต้องพึ่ง
`T: Send` เพราะข้อมูลข้างในอาจถูกย้ายข้ามเธรดผ่านการ lock/unlock สลับกันไปมา)

ตัวอย่างที่พิสูจน์ประเด็นนี้ได้ชัดเจนที่สุดคือการเอา `RefCell<T>` (ที่พิสูจน์ไปแล้วในหัวข้อ 40.7 ว่า **เป็น
`Send` แต่ไม่ใช่ `Sync`**) มาห่อด้วย `Mutex` อีกชั้น:

```rust
use std::cell::RefCell;
use std::sync::Mutex;

fn assert_send<T: Send>() {}
fn assert_sync<T: Sync>() {}

fn main() {
    // RefCell<i32> เป็น Send แต่ไม่ใช่ Sync (หัวข้อ 40.7)
    // แต่ Mutex<RefCell<i32>> เป็นทั้ง Send และ Sync! เพราะ Mutex<T> ต้องการแค่ T: Send เท่านั้น ไม่ต้อง T: Sync
    assert_send::<Mutex<RefCell<i32>>>();
    assert_sync::<Mutex<RefCell<i32>>>();
    println!("Mutex<RefCell<i32>> เป็นทั้ง Send และ Sync แม้ RefCell<i32> เองจะไม่ใช่ Sync ก็ตาม");
}
```

ผลลัพธ์: `Mutex<RefCell<i32>> เป็นทั้ง Send และ Sync แม้ RefCell<i32> เองจะไม่ใช่ Sync ก็ตาม` — นี่คือ
หลักฐานที่ชัดที่สุดว่า `Mutex<T>` ไม่ได้ "รอ" ให้ `T` เป็น `Sync` อยู่แล้วถึงจะยอมเป็น `Sync` เอง มันสร้าง
`Sync` ให้เองจากศูนย์ด้วย lock ของตัวมันเองล้วน ๆ (ต่างจาก `Arc<T>` ที่**ต้องพึ่ง** `T: Sync` อยู่แล้วถึงจะ
เป็น `Sync` ได้ ตามหัวข้อ 40.11.1) นี่คือความแตกต่างเชิงกลไกที่สำคัญระหว่าง "ตัวห่อที่ให้ยืม `&T` ออกไป
ตรง ๆ" (`Arc<T>`) กับ "ตัวห่อที่ควบคุมการเข้าถึงทั้งหมดด้วยตัวเอง" (`Mutex<T>`)

### 40.13 ตัวอย่างจริง: ออกแบบ Counter ที่ปลอดภัยสำหรับทั้งเธรดเดียวและหลายเธรด

มาดูตัวอย่างสุดท้ายที่รวมทุกอย่างในบทนี้เข้าด้วยกัน สมมติเราต้องออกแบบ counter อย่างง่ายสำหรับระบบนับจำนวน
คำสั่งซื้อที่เข้ามา (order counter) — เริ่มจาก type ธรรมดาที่สุดก่อน:

```rust
struct RawCounter {
    count: i32,
}

impl RawCounter {
    fn new() -> Self {
        RawCounter { count: 0 }
    }

    fn increment(&mut self) {
        self.count += 1;
    }

    fn get(&self) -> i32 {
        self.count
    }
}
```

`RawCounter` มี field เดียวคือ `count: i32` — ตาม gฎ auto trait ในหัวข้อ 40.8 มันเป็นทั้ง `Send` **และ**
`Sync` โดยอัตโนมัติ (เพราะ `i32: Send + Sync`) มาพิสูจน์และใช้งานแบบเธรดเดียวก่อน (ย้าย ownership ทั้งก้อน
ไปเธรดใหม่แค่เธรดเดียว):

```rust
use std::thread;

struct RawCounter {
    count: i32,
}

impl RawCounter {
    fn new() -> Self {
        RawCounter { count: 0 }
    }

    fn increment(&mut self) {
        self.count += 1;
    }

    fn get(&self) -> i32 {
        self.count
    }
}

fn assert_send<T: Send>() {}
fn assert_sync<T: Sync>() {}

fn main() {
    assert_send::<RawCounter>();
    assert_sync::<RawCounter>();

    let mut counter = RawCounter::new();
    let handle = thread::spawn(move || {
        for _ in 0..1000 {
            counter.increment();
        }
        counter.get()
    });

    let result = handle.join().unwrap();
    println!("ผลลัพธ์จากเธรดเดียว: {}", result);
    assert_eq!(result, 1000);
}
```

ผลลัพธ์: `ผลลัพธ์จากเธรดเดียว: 1000` — ถูกต้องทุกครั้ง เพราะ **ownership ของ `counter` ถูกย้าย (`move`)
ไปให้ thread ลูกทั้งก้อนเพียงครั้งเดียว** ไม่มีเธรดอื่นแตะต้องมันอีก ทั้ง thread หลักหลังจากนี้ก็ไม่มีสิทธิ์
เข้าถึง `counter` แล้ว (borrow checker บังคับ ownership rule แบบเดียวกับ Part 6) — นี่คือกรณีที่ `Send`
เพียงอย่างเดียวก็เพียงพอแล้ว ไม่ต้องพึ่ง `Mutex` เลย เพราะไม่มีการแชร์เกิดขึ้น

**แต่ถ้าต้องการให้หลายเธรดช่วยกันนับพร้อมกัน** (เช่น มี worker เธรด 8 ตัวรับคำสั่งซื้อพร้อมกันแล้วต้องอัปเดต
ตัวเลขรวมตัวเดียวกัน) — สถานการณ์นี้ต้องการ**การแชร์** (`Sync`) และ**การย้าย ownership ร่วมกันได้หลายที่**
(`Send` ของตัวห่อที่รองรับ multiple ownership) พร้อมกัน สังเกตให้ดี: แม้ `RawCounter` เป็น `Sync` เองอยู่แล้ว
ตามหัวข้อ 40.8 (เพราะ field เป็น `i32`) แต่ `Sync` ของ `RawCounter` เพียงอย่างเดียว**ไม่ได้แปลว่าคุณ mutate
มันจากหลายเธรดพร้อมกันได้** — `Sync` แค่บอกว่า "การถือ `&RawCounter` จากหลายเธรดพร้อมกันปลอดภัย" แต่การ
เรียก `.increment()` ต้องใช้ `&mut self` — และกฎ aliasing ของ Rust (Part 7) ห้ามมี `&mut` สองตัวชี้ข้อมูล
เดียวกันพร้อมกันเด็ดขาดอยู่แล้ว ต่อให้ type เป็น `Sync` ก็ตาม (borrow checker จะปฏิเสธความพยายามแชร์ `&mut
RawCounter` ให้หลายเธรดตั้งแต่ต้นอยู่ดี ไม่ต้องพึ่ง auto trait เลยด้วยซ้ำ) — นี่คือจุดสำคัญที่มักเข้าใจผิด:
**`Send`/`Sync` ของ type เองไม่ได้แก้ปัญหาเรื่อง interior mutability ให้คุณ** มันแค่บอกว่า "ย้าย/แชร์ type
นี้ข้ามเธรดปลอดภัยแค่ไหน" ส่วนการจะทำ mutation ร่วมกันได้จริงยังต้องพึ่ง**interior mutability ที่ปลอดภัย**
(`Mutex<T>`) ห่อไว้ข้างนอกอีกชั้นหนึ่งอยู่ดี — พึ่ง `Arc<Mutex<RawCounter>>`:

```rust
use std::sync::{Arc, Mutex};
use std::thread;

struct RawCounter {
    count: i32,
}

impl RawCounter {
    fn new() -> Self {
        RawCounter { count: 0 }
    }

    fn increment(&mut self) {
        self.count += 1;
    }

    fn get(&self) -> i32 {
        self.count
    }
}

fn main() {
    let counter = Arc::new(Mutex::new(RawCounter::new()));
    let mut handles = vec![];

    for _ in 0..8 {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            for _ in 0..500 {
                let mut guard = counter.lock().unwrap();
                guard.increment();
            }
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    let final_count = counter.lock().unwrap().get();
    println!("ผลรวมจากหลายเธรด: {}", final_count);
    assert_eq!(final_count, 4000);
}
```

ผลลัพธ์: `ผลรวมจากหลายเธรด: 4000` (8 เธรด × 500 ครั้ง) ถูกต้องทุกครั้งที่รัน — `counter.lock()` คืนค่า
`MutexGuard<RawCounter>` ที่ implement `Deref`/`DerefMut` ไปยัง `RawCounter` (ทวนแนวคิด `Deref` จาก
Part 27) ทำให้เรียก `.increment()` (ที่ต้องการ `&mut self`) ผ่าน `guard` ได้ตรง ๆ โดย `Mutex` เป็นคน
การันตีว่า ณ เวลาใดเวลาหนึ่งมีแค่ `guard` เดียวเท่านั้นที่ถืออยู่ (exclusive access) — จึงไม่มีทางเกิด
`&mut` สองตัวพร้อมกันจริง ๆ ที่ runtime แม้ผิวเผินจะดูเหมือนมีหลายเธรดพยายาม "mutate พร้อมกัน" ก็ตาม

**สรุปสิ่งที่ต้องเอาไปใช้จริง**: `RawCounter` เป็น `Send + Sync` โดยอัตโนมัติเพราะ field เป็น primitive
แต่นั่น**ไม่พอ**สำหรับ multi-thread mutation — ต้องใช้ `Arc<Mutex<T>>` เสมอเมื่อต้องการ **(1) เป็นเจ้าของ
ร่วมกันได้จากหลายเธรด (`Arc`)** และ **(2) แก้ไขข้อมูลได้อย่างปลอดภัยจากหลายเธรด (`Mutex`)** — ถ้าต้องการ
แค่ข้อ (1) อย่างเดียวโดยไม่มีการแก้ไข (read-only sharing) `Arc<T>` เปล่า ๆ ก็พอแล้วไม่ต้องมี `Mutex`
(เช่น config ที่โหลดครั้งเดียวแล้วอ่านอย่างเดียวตลอดอายุโปรแกรม) แต่ถ้ามีการแก้ไขแม้แต่นิดเดียวจากมากกว่า
หนึ่งเธรด ต้องมี `Mutex`/`RwLock` ห่อไว้เสมอ ไม่มีทางเลี่ยง

### 40.14 Send/Sync ในทางปฏิบัติอื่น ๆ: channels และ trait object ข้ามเธรด

ปิดท้ายเนื้อหาด้วยการโยงกลับไปยังสองจุดสำคัญของหลักสูตรที่ยังไม่ได้พูดถึงตรง ๆ ในบทนี้ — channel จาก
Part 38 และ trait object จาก Part 21

**`mpsc::Sender<T>`/`Receiver<T>` เป็น `Send`/`Sync` เมื่อไหร่?**

ทวนจาก Part 38: `mpsc::channel::<T>()` คืนคู่ `(Sender<T>, Receiver<T>)` สำหรับส่งข้อมูลข้าม thread
ผ่านคิวข้อความ ทั้งสอง type นี้ก็ถูกคำนวณ `Send`/`Sync` ด้วยกฎเดียวกันทั้งหมด — และเพราะ**ทั้งคู่ต้องออกแบบมา
ให้ใช้ข้าม thread โดยธรรมชาติอยู่แล้ว** (นั่นคือจุดประสงค์ทั้งหมดของ channel) std จึงimplement ให้ทั้ง
`Sender<T>` และ `Receiver<T>` เป็น `Send` (เมื่อ `T: Send`) — และ `Sender<T>` ยังเป็น `Sync` ด้วย เพราะ
ออกแบบให้ clone แล้วแจกจ่ายให้หลาย thread ส่งเข้า queue เดียวกันได้พร้อมกันอย่างปลอดภัย (multiple producer
ตามชื่อ "mpsc" — multiple producer, single consumer)

```rust
use std::sync::mpsc;
use std::thread;

fn assert_send<T: Send>() {}
fn assert_sync<T: Sync>() {}

fn main() {
    assert_send::<mpsc::Sender<i32>>();
    assert_sync::<mpsc::Sender<i32>>();

    let (tx, rx) = mpsc::channel::<i32>();
    let handle = thread::spawn(move || {
        tx.send(42).unwrap();
    });
    handle.join().unwrap();
    println!("ได้รับค่า: {}", rx.recv().unwrap());
}
```

ผลลัพธ์: `ได้รับค่า: 42` — ตอนนี้เราเข้าใจแล้วว่าทำไม Part 38 ใช้ `tx.clone()` แจกให้หลาย thread ได้อย่าง
ปลอดภัย (`Sender<T>: Sync` ทำให้แชร์ reference ได้ และ `Sender<T>: Clone` ทำให้แต่ละ thread มีสำเนาของ
ตัวเองสำหรับย้ายเข้าไปในตัวเองผ่าน `move`)

**`Box<dyn FnOnce() + Send + 'static>`: trait object ที่ส่งข้าม thread ได้**

ทวนจาก Part 21 หัวข้อ 21.7: `dyn Trait + Send` compile ผ่านได้เพราะ `Send` เป็น auto trait ไม่มี vtable
ของตัวเอง ตอนนี้เราเห็นการใช้งานจริงของ pattern นี้แล้ว — สถานการณ์ที่พบบ่อยมากคือการสร้าง **job queue**
หรือ **thread pool** ที่ต้องเก็บ closure หลายตัว (แต่ละตัวมี type ที่ไม่เหมือนกันในระดับ concrete type
เพราะ closure แต่ละตัวคือ struct ที่ไม่มีชื่อของตัวเอง) ไว้ใน collection เดียว แล้วส่งแต่ละตัวไปรันใน
thread อื่นทีละตัว — วิธีแก้คือห่อด้วย `Box<dyn FnOnce() + Send + 'static>` (trait object ที่รวม
`FnOnce()` เข้ากับ auto trait `Send` และ `'static` พร้อมกัน):

```rust
use std::thread;

struct Job(Box<dyn FnOnce() + Send + 'static>);

fn run_job(job: Job) {
    thread::spawn(move || {
        (job.0)();
    })
    .join()
    .unwrap();
}

fn main() {
    let job = Job(Box::new(|| {
        println!("job กำลังรันอยู่ในเธรดใหม่");
    }));
    run_job(job);
}
```

ผลลัพธ์: `job กำลังรันอยู่ในเธรดใหม่` — โครงสร้าง `Box<dyn FnOnce() + Send + 'static>` นี้คือหัวใจของ
thread pool ทุกตัวที่เขียนด้วย Rust (รวมถึง crate ที่มีชื่อเสียงอย่าง `rayon` และ `tokio`) — มันคือคำตอบ
สุดท้ายของคำถามที่เปิดไว้ใน Part 21 ว่า "แล้วเมื่อไหร่จะได้ใช้ `dyn Trait + Send` จริง ๆ สักที" ตอนนี้ตอบได้
เต็มปากแล้วว่า: **ทุกครั้งที่ต้องเก็บ closure/trait object ต่างชนิดกันไว้ใน collection เดียว แล้วต้องการันตี
ว่ามันย้ายข้าม thread ได้อย่างปลอดภัยด้วย**

### 40.15 สรุปกลไกทั้งหมดเป็นแผนภาพการตัดสินใจเดียว

ก่อนเข้ากับดัก ลองรวมทุกอย่างที่เรียนมาในบทนี้ให้เป็น**แผนภาพการตัดสินใจ (decision flow)** เดียวที่ใช้ได้จริง
เวลาต้องเลือก smart pointer สำหรับข้อมูลที่จะแชร์ในโปรแกรม — ไล่ตามคำถามทีละข้อ:

```text
คำถามที่ 1: ข้อมูลนี้ต้องมีเจ้าของมากกว่าหนึ่งคนหรือไม่ (multiple ownership)?
  ├─ ไม่ต้อง → ใช้ ownership ปกติ หรือ Box<T> ถ้าต้องเก็บบน heap (Part 27) ไม่ต้องอ่านต่อ
  └─ ต้องมีหลายเจ้าของ → ไปคำถามที่ 2

คำถามที่ 2: ข้อมูลนี้จะถูกใช้ข้ามมากกว่าหนึ่ง thread หรือไม่?
  ├─ ไม่ข้าม thread (single-thread เท่านั้น) → ใช้ Rc<T> (Part 28) — เร็วกว่า ไม่มีต้นทุน atomic
  └─ ข้าม thread → ต้องใช้ Arc<T> เท่านั้น (Rc<T> จะไม่ Send/Sync ตามหัวข้อ 40.4) → ไปคำถามที่ 3

คำถามที่ 3: ข้อมูลภายในต้องถูกแก้ไข (mutate) หลังจากสร้างแล้วหรือไม่?
  ├─ ไม่ต้องแก้ไข (อ่านอย่างเดียวตลอดชีวิต) → Arc<T> เปล่า ๆ พอแล้ว ไม่ต้องมี Mutex/RefCell เลย
  └─ ต้องแก้ไขได้ → ไปคำถามที่ 4

คำถามที่ 4: การแก้ไขนั้นเกิดจาก thread เดียวหรือหลาย thread พร้อมกัน?
  ├─ single-thread เท่านั้น (ไม่ข้าม thread ณ จุดที่แก้ไข) → RefCell<T> ห่อไว้ข้างใน Rc<T> เพียงพอ
  └─ หลาย thread พร้อมกัน → ต้องใช้ Mutex<T> หรือ RwLock<T> ห่อไว้ข้างใน Arc<T> เท่านั้น
       (RefCell<T> จะไม่ Sync — ดูหัวข้อ 40.7 และกับดัก Arc<RefCell<T>> ในหัวข้อ 40.11.1)

คำถามที่ 5 (ถ้าไปถึงจุดนี้ คือใช้ Mutex/RwLock): อ่านบ่อยกว่าเขียนมากหรือไม่ (read-heavy)?
  ├─ อ่าน ๆ เขียน ๆ พอ ๆ กัน หรือไม่แน่ใจ → ใช้ Mutex<T> (เรียบง่ายกว่า พฤติกรรมคาดเดาง่ายกว่า)
  └─ อ่านบ่อยกว่าเขียนมาก (เช่น cache, สถิติ) → พิจารณา RwLock<T> (Part 51) แทน เพื่อให้อ่านพร้อมกันได้
       หลาย thread โดยไม่ต้อง block กันเอง
```

แผนภาพนี้คือสิ่งที่รวมเอา Part 27–29 (smart pointers) และ Part 37–40 (concurrency) ทั้งหมดของหลักสูตรมา
รวมกันเป็นเครื่องมือเดียวที่หยิบมาใช้ได้ทันทีในงานจริง — ทุกครั้งที่ลังเลว่าจะเลือก type ไหน ให้ไล่คำถามทั้ง 5
ข้อนี้ตามลำดับ คำตอบที่ได้จะตรงกับ type ที่ compiler ยอมรับเสมอ (เพราะแผนภาพนี้สร้างขึ้นจากกฎ `Send`/`Sync`
ที่ compiler ใช้ตรวจจริง ไม่ใช่แค่ heuristic ที่คาดเดา)

## กับดักที่พบบ่อย (Common Pitfalls)

**1. คิดว่า error "cannot be sent between threads safely" คือ error ทั่วไปที่แก้ด้วยการ `.clone()` เพิ่ม**

หลายคนเจอ error นี้ครั้งแรกแล้วลองแก้แบบสุ่ม เช่น clone ตัวแปรเพิ่มก่อนส่งเข้า `thread::spawn` ทั้งที่ต้นเหตุ
จริงคือ type ที่ใช้ (`Rc<T>`) ไม่ได้ implement `Send` เลย ไม่ว่าจะ clone กี่ครั้งก็ไม่ช่วย:

```rust
use std::rc::Rc;
use std::thread;

fn main() {
    let data = Rc::new(vec![1, 2, 3]);
    let data2 = data.clone(); // clone ก่อนส่ง — ยังไม่ช่วยอะไร เพราะ clone ยังเป็น Rc<Vec<i32>> เหมือนกัน
    let handle = thread::spawn(move || {
        println!("{:?}", data2);
    });
    handle.join().unwrap();
}
```

Error ที่ได้จะเหมือนเดิมทุกอย่าง (แค่เปลี่ยนชื่อ type ภายในเป็น `Vec<i32>`):

```
error[E0277]: `Rc<Vec<i32>>` cannot be sent between threads safely
  |
  = help: within `{closure@...}`, the trait `Send` is not implemented for `Rc<Vec<i32>>`
note: required because it's used within this closure
note: required by a bound in `spawn`
```

**วิธีแก้ที่ถูกต้อง**: เปลี่ยน `Rc<T>` เป็น **`Arc<T>`** ตั้งแต่ต้น (ไม่ใช่ clone มากขึ้น) — `Arc::clone`
ใช้ atomic refcount ที่ปลอดภัยข้ามเธรดตามหัวข้อ 40.11 การแก้ที่ต้นเหตุคือเปลี่ยน type ไม่ใช่เปลี่ยนจำนวนครั้ง
ที่เรียก `.clone()`

**2. เข้าใจผิดว่า field ที่ "ดูปลอดภัย" ทำให้ struct เป็น `Send` เสมอ ทั้งที่มี field ซ่อนที่ไม่ใช่**

```rust
use std::rc::Rc;
use std::thread;

struct Report {
    title: String,       // Send + Sync
    total_sales: f64,    // Send + Sync
    cached_summary: Rc<String>, // ไม่ Send! ซ่อนอยู่ท่ามกลาง field ที่ดูปลอดภัยทั้งหมด
}

fn main() {
    let report = Report {
        title: "Q3".to_string(),
        total_sales: 100.0,
        cached_summary: Rc::new("summary".to_string()),
    };
    thread::spawn(move || {
        // ใช้ทั้งสาม field ในโค้ด ทำให้ closure ต้อง capture cached_summary (Rc<String>) เข้ามาด้วย
        println!("{} - {} - {}", report.title, report.total_sales, report.cached_summary);
    });
}
```

Error:

```
error[E0277]: `Rc<String>` cannot be sent between threads safely
  --> src/main.rs:14:19
   |
14 |       thread::spawn(move || {
   |       ------------- ^------
   |       |             |
   |  _____|_____________within this `{closure@src/main.rs:14:19: 14:26}`
   | |     |
   | |     required by a bound introduced by this call
15 | |         println!("{} - {} - {}", report.title, report.total_sales, report.cached_summary);
16 | |     });
   | |_____^ `Rc<String>` cannot be sent between threads safely
   |
   = help: within `{closure@src/main.rs:14:19: 14:26}`, the trait `Send` is not implemented for `Rc<String>`
note: required because it's used within this closure
note: required by a bound in `spawn`
```

**ข้อสังเกตสำคัญเพิ่มเติม**: ถ้า closure ข้างในใช้แค่ `report.title` เพียง field เดียว (ไม่แตะ
`cached_summary` เลย) โค้ดนี้จะ **compile ผ่านได้ปกติ!** เพราะ Rust 2021 edition มีฟีเจอร์ **disjoint
closure capture** — closure จะ capture มาเฉพาะ field ที่มันใช้จริงเท่านั้น ไม่ใช่ทั้ง struct ทั้งก้อน
เสมอไป (ทวนจาก Part 24) ดังนั้นถ้าไม่ได้แตะ field ที่เป็น `Rc<T>` เลย closure ก็จะไม่ capture มันมา และ
`Send` ก็จะไม่มีปัญหาอะไรเลย — นี่คือเหตุผลที่ตัวอย่างข้างบนต้องใช้ `report.cached_summary` ในการ
`println!` ด้วย เพื่อบังคับให้ closure capture field ที่มีปัญหาเข้ามาจริง ๆ ให้เห็น error ชัด ๆ (แต่ในโค้ด
จริงคุณไม่ควรพึ่งพา "ความบังเอิญที่ไม่ได้แตะ field ที่มีปัญหา" แบบนี้เป็นทางแก้ — มันคือ time bomb ที่จะ
ระเบิดทันทีที่มีคนแก้โค้ดให้ไปแตะ field นั้นในอนาคต ทางแก้ที่ถูกต้องเสมอคือเปลี่ยน `Rc<T>` เป็น `Arc<T>`)

จำกฎจากหัวข้อ 40.8 ให้แม่น: **แค่ field เดียวที่ไม่ใช่ `Send` ก็ทำให้ struct ทั้งก้อนไม่ใช่ `Send`** ไม่ว่า
field อื่นจะปลอดภัยกี่ตัวก็ตาม เวลาเจอ error แบบนี้ ให้ไล่หา field ที่เป็น `Rc<T>`/`RefCell<T>` (หรือ type
ที่มี `Rc`/`RefCell` ซ่อนอยู่ข้างในอีกที) แล้วเปลี่ยนเป็น `Arc<T>`/`Mutex<T>` ตามความเหมาะสม

**3. คิดว่า `Sync` แปลว่า "mutate จากหลายเธรดพร้อมกันได้เลย" — สับสนระหว่าง "แชร์ปลอดภัย" กับ "mutate ได้"**

ตามที่อธิบายในหัวข้อ 40.13: `T: Sync` บอกแค่ว่า **การถือ `&T` จากหลายเธรดพร้อมกันปลอดภัย** ไม่ได้บอกว่า
คุณ mutate ผ่าน `&T` นั้นได้ ถ้า `T` ไม่มี interior mutability เลย (ไม่มี `Cell`/`RefCell`/`Mutex` ซ่อนอยู่)
`&T` ก็ mutate ไม่ได้อยู่แล้วโดยธรรมชาติ (ต้องใช้ `&mut T` ซึ่ง borrow checker ไม่ยอมให้แชร์หลายตัวพร้อมกัน
อยู่แล้ว) `Sync` เพียงอย่างเดียวจึงไม่เคย "ปลดล็อก" การ mutate ข้ามเธรดให้เลย มันบอกแค่ความปลอดภัยของการ
แชร์ **read-only reference** เท่านั้น ถ้าต้องการ mutate ร่วมกันข้ามเธรดจริง ๆ ต้องมี `Mutex<T>`/`RwLock<T>`
(interior mutability แบบ thread-safe) เสมอ ไม่มีทางลัด

**4. ลืมว่า `Mutex<T>::lock()` return `Result` — และ deadlock ตัวเองด้วยการ lock ซ้อน (double-lock)
ในเธรดเดียวกัน**

```rust
use std::sync::Mutex;

fn main() {
    let m = Mutex::new(5);

    let guard1 = m.lock().unwrap();
    println!("guard1: {}", *guard1);

    // ถ้าพยายาม lock() ซ้ำในเธรดเดียวกันโดยที่ guard1 ยังไม่ถูก drop:
    // let guard2 = m.lock().unwrap(); // <-- จะ deadlock ค้างตรงนี้ตลอดกาล (ไม่มี panic ไม่มี error message ใด ๆ)
    drop(guard1); // ต้อง drop guard เดิมก่อน ถึงจะ lock() ซ้ำได้อย่างปลอดภัย
    let guard2 = m.lock().unwrap();
    println!("guard2: {}", *guard2);
}
```

`std::sync::Mutex` ของ Rust (ต่างจาก `Mutex` บางภาษาที่เป็น *reentrant*) **ไม่อนุญาตให้เธรดเดียวกัน lock
ซ้ำสองครั้งโดยไม่ปล่อยตัวแรกก่อน** — ถ้าทำแบบนั้นจะเกิด **deadlock ทันที** (เธรดค้างตลอดกาล ไม่มี panic ไม่มี
error message เตือน เพราะ deadlock ไม่ใช่สิ่งที่ type system ตรวจจับได้ — มันเป็นปัญหาด้าน logic ที่ compiler
มองไม่เห็น ต้องระมัดระวังด้วยตัวเองเสมอ) วิธีป้องกันคือจำกัด scope ของ `MutexGuard` ให้แคบที่สุด (ใช้ block
`{ }` ครอบ หรือเรียก `drop(guard)` ทันทีที่ใช้เสร็จ) เพื่อให้ lock ถูกปล่อยเร็วที่สุด และหลีกเลี่ยงการเรียก
`.lock()` ซ้อนกันในฟังก์ชันเดียวกันโดยไม่จำเป็น

**5. คิดว่า `Send`/`Sync` implies กันไปมา (คิดว่า `Send` แล้วต้อง `Sync` เสมอ หรือกลับกัน)**

`Send` และ `Sync` เป็น**อิสระจากกันโดยสมบูรณ์** — มีทั้งสี่กรณีที่เป็นไปได้จริง และ `RefCell<T>` ก็คือตัวอย่าง
ที่ชัดที่สุดของกรณี "`Send` แต่ไม่ `Sync`" (หัวข้อ 40.7) ในทางตรงกันข้าม ก็มี type ที่ "`Sync` แต่ไม่ `Send`"
ได้เช่นกัน (พบได้ยากกว่า แต่มีอยู่จริงใน ecosystem เช่น type ที่ผูกกับ thread เดิมตอนสร้าง แต่ยอมให้อ่านค่า
จากเธรดอื่นผ่าน reference ได้) อย่าเดาความสัมพันธ์เอง — ใช้เทคนิค `assert_send::<T>()` /
`assert_sync::<T>()` จากหัวข้อ 40.3 ตรวจให้แน่ใจเสมอเมื่อไม่ชัวร์

**6. คิดว่าแค่เปลี่ยน `Rc` เป็น `Arc` พอแล้ว โดยไม่ต้องเปลี่ยน `RefCell` เป็น `Mutex` ด้วย**

ตามที่พิสูจน์ไว้ในหัวข้อ 40.11.1 — `Arc<RefCell<T>>` **ไม่ใช่ทั้ง `Send` และ `Sync`** เพราะ `Arc<T>: Send`
ต้องการ `T: Send + Sync` (ไม่ใช่แค่ `T: Send`) และ `RefCell<T>` ไม่ผ่านเงื่อนไข `Sync` เลย เวลา migrate โค้ด
จาก single-thread (Part 28: `Rc<RefCell<T>>`) ไปเป็น multi-thread ต้องเปลี่ยน**ทั้งสองชั้นพร้อมกัน**เสมอ —
`Rc` → `Arc` (แก้ปัญหา `Send` ของตัวนับ) และ `RefCell` → `Mutex` (แก้ปัญหา `Sync` ของ interior mutability)
เปลี่ยนแค่ชั้นเดียวจะยังคง compile ไม่ผ่านอยู่ดี

**7. ลืมว่า `.lock().unwrap()` panic ได้จริง เมื่อ `Mutex` ถูก "poison" จาก thread ที่ panic ขณะถือ lock**

Part 39 สอนให้เขียน `counter.lock().unwrap()` เป็นสูตรสำเร็จ แต่ไม่ได้อธิบายว่า `.unwrap()` นั้น**panic ได้
จริง**ในสถานการณ์หนึ่ง: ถ้ามี thread ใด thread หนึ่งเกิด panic **ขณะที่ยังถือ lock อยู่** (ไม่ได้ปล่อยตามปกติ)
Rust จะตั้งสถานะ `Mutex` นั้นเป็น **"poisoned"** ทันที เพื่อเตือนว่าข้อมูลภายในอาจอยู่ในสถานะที่ไม่สมบูรณ์
(เพราะ thread ที่ panic อาจแก้ไขข้อมูลไปครึ่งทางแล้วยังไม่เสร็จ) — การเรียก `.lock()` ครั้งต่อไปจาก thread
ไหนก็ตามจะได้ `Err(PoisonError)` กลับมาแทน `Ok(MutexGuard)` และถ้าเขียน `.unwrap()` ต่อท้ายแบบไม่ระวัง โปรแกรม
ก็จะ panic ตามไปด้วย:

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let data = Arc::new(Mutex::new(0));

    let data2 = Arc::clone(&data);
    let handle = thread::spawn(move || {
        let _guard = data2.lock().unwrap();
        panic!("เธรดนี้ล่มขณะยังถือ lock อยู่!");
    });

    // เธรดลูก panic ขณะถือ lock -> join() จะได้ Err กลับมา (thread panicked)
    let _ = handle.join();

    // ตอนนี้ Mutex กลาย "poisoned" แล้ว: การ .lock() ครั้งถัดไปจะได้ Err ไม่ใช่ panic
    let lock_result = data.lock();
    match lock_result {
        Ok(_) => println!("ล็อกได้ปกติ (ไม่ควรเกิดในตัวอย่างนี้)"),
        Err(poison_error) => {
            println!("lock ถูก poison แล้ว: {}", poison_error);
            let guard = poison_error.into_inner();
            println!("ยังดึงข้อมูลออกมาดูได้ผ่าน into_inner(): {}", *guard);
        }
    }
}
```

ผลลัพธ์:

```
thread '<unnamed>' panicked at src/main.rs:10:9:
เธรดนี้ล่มขณะยังถือ lock อยู่!
lock ถูก poison แล้ว: poisoned lock: another task failed inside
ยังดึงข้อมูลออกมาดูได้ผ่าน into_inner(): 0
```

จุดสำคัญ: `PoisonError` **ไม่ได้ทำให้ข้อมูลหายไป** — มันแค่เป็นการเตือนเชิง defensive ว่า "ข้อมูลนี้*อาจ*
ไม่สมบูรณ์ ให้ตรวจสอบก่อนใช้" คุณยังดึงข้อมูลออกมาดูได้เสมอผ่าน `poison_error.into_inner()` (ในตัวอย่างนี้
ข้อมูลจริง ๆ ไม่ได้เสียหายอะไรเลย เพราะ panic เกิดขึ้น**หลัง**ได้ lock แต่**ก่อน**จะแก้ไขข้อมูลอะไร) สำหรับ
โปรแกรม production ที่ต้องทนทานต่อ panic ของ thread ใด thread หนึ่งโดยไม่ทำให้ทั้งโปรแกรมล้ม ควร handle
`PoisonError` อย่างตั้งใจ (เช่นด้วย `match` หรือ `.unwrap_or_else()`) แทนที่จะใช้ `.unwrap()` แบบเดารวด — 
`.unwrap()` เหมาะกับตัวอย่างการเรียนในบทนี้และ Part 39 ที่ต้องการความกระชับ แต่ในโค้ด production ที่ยอมรับ
ไม่ได้ให้ทั้งโปรแกรม crash เพราะ thread เดียว panic ควร handle ให้ครบ

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เขียน struct ชื่อ `Product` ที่มี field `name: String`, `price: f64`, และ `quantity: u32`
   ใช้เทคนิค `assert_send::<Product>()` และ `assert_sync::<Product>()` จากหัวข้อ 40.3 พิสูจน์ว่ามันเป็นทั้ง
   `Send` และ `Sync` โดยไม่ต้องเขียนอะไรเพิ่ม จากนั้นลองเพิ่ม field ใหม่ `tags: Rc<Vec<String>>` เข้าไป แล้ว
   สังเกตว่า `assert_send::<Product>()` compile ไม่ผ่านอีกต่อไป — อ่าน error message ที่ได้ แล้วอธิบายด้วย
   คำพูดของตัวเองว่าทำไม field เดียวถึงทำให้ทั้ง struct เสีย `Send` ไป (hint: กฎ AND เชิงโครงสร้างจากหัวข้อ
   40.8)

2. **(กลาง)** เขียนโปรแกรมที่มี `struct BankAccount { balance: i64 }` พร้อม method `deposit(&mut self,
   amount: i64)` และ `withdraw(&mut self, amount: i64) -> bool` (คืน `false` ถ้าเงินไม่พอ) ใช้ `Arc<Mutex<
   BankAccount>>` สร้างบัญชีเดียวที่ใช้ร่วมกัน แล้วสร้าง 20 เธรด แต่ละเธรด deposit 100 บาทซ้ำ 50 ครั้ง เธรดที่
   join กันหมดแล้วต้องได้ยอดรวม `20 * 100 * 50 = 100,000` เป๊ะทุกครั้งที่รัน (hint: โครงสร้างคล้ายตัวอย่าง
   `RawCounter` ในหัวข้อ 40.13 แต่ใช้ deposit แทน increment)

3. **(ยาก)** ลองเขียน struct ที่ชื่อ `SessionCache` ที่ภายในมี `data: HashMap<String, String>` และตั้งใจ
   ห่อด้วย `Rc<RefCell<HashMap<String, String>>>` (ใช้แบบเดียวกับที่เรียนใน Part 28) แล้วพยายามแชร์
   `SessionCache` นี้ให้ 4 เธรดอ่าน/เขียนพร้อมกันผ่าน `thread::spawn` สังเกต error ที่เกิดขึ้น (จะมีมากกว่า
   หนึ่ง error ซ้อนกัน) จากนั้นแก้ไขให้ compile ผ่านและทำงานถูกต้องโดยเปลี่ยนเป็น `Arc<Mutex<HashMap<String,
   String>>>` — อธิบายว่าทำไมต้องเปลี่ยน**ทั้งสองชั้น** (ทั้ง `Rc`→`Arc` และ `RefCell`→`Mutex`) ไม่ใช่แค่ชั้น
   เดียว (hint: ตารางเปรียบเทียบในหัวข้อ 40.12 อธิบายเหตุผลของแต่ละชั้นแยกกันไว้แล้ว)

4. **(ยาก/ประยุกต์ใช้งานจริง)** ออกแบบระบบ "ตัวนับการเข้าชมหน้าเว็บแบบ read-heavy" (page view counter):
   มี `Arc<RwLock<HashMap<String, u64>>>` เก็บคู่ `(page_url, view_count)` สร้าง 4 เธรดที่ทำหน้าที่**อ่าน**
   สถิติบ่อย ๆ (`.read()`) และ 1 เธรดที่ทำหน้าที่**เพิ่มค่า**เวลามีการเข้าชมใหม่ (`.write()`) ให้รันจนกว่า
   เธรดเขียนจะอัปเดตครบ 1,000 ครั้ง แล้วให้เธรดอ่านทั้งหมด join และพิมพ์ผลรวมสุดท้าย จากนั้นตอบคำถามเชิงทฤษฎี
   (ไม่ต้องเขียนโค้ดพิสูจน์): เพราะเหตุใด `RwLock<T>` จึงเหมาะกับสถานการณ์นี้มากกว่า `Mutex<T>` ทั้งที่ทั้งคู่
   เป็น `Sync` เหมือนกัน (hint: คิดถึงความแตกต่างระหว่าง "อ่านพร้อมกันได้หลายเธรด" กับ "เขียนได้ทีละเธรด" ที่
   กล่าวถึงสั้น ๆ ในหัวข้อ 40.7 — รายละเอียดเต็มจะเรียนใน Part 51)

5. **(โบนัส/ประยุกต์ใช้งานจริง)** สร้าง mini "task queue" ของตัวเอง โดยมี `struct TaskQueue { jobs: Vec<Box<
   dyn FnOnce() + Send + 'static>> }` พร้อม method `push` (รับ closure ใดก็ได้ที่ตรง bound แล้วห่อเก็บไว้)
   และ `run_all` (ดึงทุก job ออกมาแล้วส่งให้ `thread::spawn` รันแยกกันคนละ thread แล้ว join รอให้ครบทุกตัว)
   ทดสอบด้วยการ push closure 3 ตัวที่ต่างชนิดกันโดยสิ้นเชิง (เช่น ตัวหนึ่ง capture `String`, อีกตัว capture
   `i32`, อีกตัว capture `Vec<u8>`) แล้วอธิบายว่าทำไมต้องใช้ `Box<dyn FnOnce() + Send + 'static>` แทนที่จะใช้
   generic `Vec<F>` ธรรมดา (hint: ทวนความแตกต่างระหว่าง static dispatch กับ dynamic dispatch จาก Part 21 —
   `Vec<T>` ต้องมี concrete type เดียวกันทุก element แต่ closure แต่ละตัวคือ type ที่ไม่ซ้ำกันเสมอ)

## สรุป

บทนี้ปิดวงคำถามใหญ่ที่เปิดไว้ตลอด Part 37–39: **compiler รู้ได้อย่างไรว่าโค้ด concurrency ปลอดภัยหรือไม่**
คำตอบคือ auto trait สองตัว — `Send` (ปลอดภัยที่จะย้ายข้ามเธรด) และ `Sync` (ปลอดภัยที่จะแชร์ reference ข้าม
เธรด) ที่ compiler คำนวณให้อัตโนมัติแบบ structural/recursive จาก field ทุกตัวของทุก type ในโปรแกรม เราเห็น
กลไกที่แม่นยำว่าทำไม `Rc<T>` ไม่ใช่ `Send` (ตัวนับ refcount ไม่ใช่ atomic) ทำไม `RefCell<T>` เป็น `Send`
แต่ไม่ใช่ `Sync` (ตัวนับ borrow มีปัญหาเดียวกันเป๊ะ) และทำไม `Arc<Mutex<T>>` แก้ปัญหาทั้งสองมิติได้พร้อมกัน
(`Arc` แก้ `Send` ด้วย atomic refcount, `Mutex` แก้ `Sync` ด้วย blocking lock) พร้อมทั้งอ่านลายเซ็นเต็มของ
`thread::spawn` ได้ครบทุกส่วนเป็นครั้งแรก

**บทนี้คือจุดปิดของ Module 2 (ระดับกลาง) ทั้งโมดูล** — ก่อนจบ ลองมองภาพรวมของเส้นทางที่เดินมาตั้งแต่ Part 21:

- **Part 21–23 (Type system ขั้นสูง)**: เรียนรู้ trait object และ `dyn Trait`, generic ขั้นสูงพร้อม
  `PhantomData`, และ lifetime ขั้นสูง — วางรากฐานความเข้าใจ type system ของ Rust ให้ลึกกว่าระดับพื้นฐาน
- **Part 24–26 (Functional-style programming)**: closures (`Fn`/`FnMut`/`FnOnce`) และ iterator ที่ทำให้
  เขียนโค้ดสไตล์ functional ได้อย่างมีประสิทธิภาพเทียบเท่า loop มือเขียน (zero-cost abstraction)
- **Part 27–29 (Smart Pointers)**: `Box<T>` สำหรับ heap allocation เดี่ยว, `Rc<T>`/`RefCell<T>` สำหรับ
  multiple ownership และ interior mutability ในบริบทเธรดเดียว, `Weak<T>`/`Cow<T>` สำหรับกรณีขั้นสูงกว่านั้น
  — บทนี้แสดงให้เห็นว่าทำไม pointer เหล่านี้ถูกจำกัดไว้แค่เธรดเดียว
- **Part 30–31 (Error Handling ขั้นสูง)**: custom error type, `From`/`Into`, `thiserror`/`anyhow` สำหรับ
  จัดการ error แบบมืออาชีพในโปรเจกต์จริง
- **Part 32–34 (Testing และ Documentation)**: unit test, integration test, และ rustdoc สำหรับสร้างโค้ดที่
  ทั้งถูกต้องและมีเอกสารกำกับ
- **Part 35 (Cargo ขั้นสูง)**: features, profiles, workspace จริงสำหรับโปรเจกต์ขนาดใหญ่
- **Part 36 (Macros)**: `macro_rules!` สำหรับลดโค้ดซ้ำซ้อนด้วย metaprogramming ระดับ declarative
- **Part 37–40 (Concurrency arc)**: threads พื้นฐาน → channels (message passing) → `Mutex`/`Arc`
  (shared-state) → **`Send`/`Sync`** (บทนี้ ที่อธิบายกลไกทั้งหมดที่ทำให้สามบทก่อนหน้าปลอดภัยจริง)

ทั้ง 20 บทของ Module 2 รวมกันคือสิ่งที่เปลี่ยนคุณจาก "คนที่เขียน Rust ให้ compile ผ่าน" ไปเป็น "คนที่เข้าใจ
**ว่าทำไม** Rust ออกแบบมาแบบนี้" — ทุก error message ที่เจอไม่ใช่อุปสรรค แต่เป็นหน้าต่างที่เปิดให้เห็นกลไก
ความปลอดภัยที่ compiler ทำงานให้แบบเงียบ ๆ อยู่เบื้องหลังตลอดเวลา

**เช็คลิสต์สั้น ๆ ที่ควรติดตัวไปใช้ในงานจริงหลังจบบทนี้:**

- ก่อนออกแบบ struct/enum ที่จะใช้ข้าม thread ให้ตรวจด้วย `assert_send::<T>()` / `assert_sync::<T>()`
  (หัวข้อ 40.3) ก่อนเขียนโค้ด multi-thread จริง จะช่วยจับปัญหาได้เร็วกว่าเจอ error ตอนเขียนโปรแกรมเต็มรูปแบบ
  ไปแล้วมาก
- เจอ error "cannot be sent between threads safely" ให้ไล่หา `Rc<T>` (เปลี่ยนเป็น `Arc<T>`) และเจอ
  "cannot be shared between threads safely" ให้ไล่หา `RefCell<T>`/`Cell<T>` (เปลี่ยนเป็น `Mutex<T>`/
  `RwLock<T>`) — สอง error message นี้บอกต้นเหตุตรง ๆ เสมอ ไม่ต้องเดา
- เปลี่ยนจาก single-thread ไปสู่ multi-thread ต้องเปลี่ยน**ทั้งสองชั้น**พร้อมกันเสมอ (`Rc`→`Arc` และ
  `RefCell`→`Mutex`) — เปลี่ยนแค่ชั้นเดียวจะยังคง compile ไม่ผ่าน (หัวข้อ 40.11.1)
- `Send`/`Sync` ของ type ไม่ได้แก้ปัญหา mutation ให้อัตโนมัติ — ยังต้องมี interior mutability ที่ปลอดภัย
  (`Mutex`/`RwLock`) เสมอเมื่อมีการแก้ไขข้อมูลจากมากกว่าหนึ่ง thread (หัวข้อ 40.13)
- ใช้แผนภาพการตัดสินใจในหัวข้อ 40.15 ทุกครั้งที่ลังเลว่าจะเลือก `Rc`/`Arc`/`RefCell`/`Mutex`/`RwLock` ตัวไหน

**Module 3 (ระดับสูง)** เริ่มต้นที่ **Part 41: Unsafe Rust เบื้องต้น** — จุดที่คุณจะได้เรียนรู้ว่า
เมื่อไหร่และทำไมนักพัฒนา Rust ถึงต้อง "ปิด" การตรวจสอบบางอย่างของ compiler ด้วย keyword `unsafe`,
กฎ 5 ข้อที่ต้องรับผิดชอบเองเมื่อเข้าสู่โลก unsafe, และความเชื่อมโยงตรงกับเนื้อหาบทนี้ — `PhantomData<*const
()>` ที่เพิ่งเห็นในหัวข้อ 40.9 จะกลายเป็นเครื่องมือที่ใช้บ่อยขึ้นมากตอนออกแบบ type ที่ห่อ raw pointer เอง
และการตัดสินใจว่า type ของคุณควรเป็น `Send`/`Sync` หรือไม่จะกลายเป็นสิ่งที่**คุณต้องรับผิดชอบเอง**ผ่าน
`unsafe impl` แทนที่จะให้ compiler คำนวณให้อัตโนมัติเหมือนที่ผ่านมาตลอด 40 บทนี้

---

**Part ก่อนหน้า:** [Mutex, Arc และ Shared-State Concurrency](part-039-mutex-arc.md) | **Part ถัดไป:**
[Unsafe Rust เบื้องต้น](part-041-unsafe-basics.md)
