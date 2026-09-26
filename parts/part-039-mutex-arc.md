# Part 39: Mutex, Arc และ Shared-State Concurrency

> โมดูล: ระดับกลาง (Intermediate) | ระดับ: สูง | เวลาโดยประมาณ: 180 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้ว่า concurrency มีสองแนวทางหลักในการแก้ปัญหา "หลาย thread ต้องทำงานร่วมกัน": **message passing**
  (Part 38 — ไม่มีใครแบ่งข้อมูลกัน ส่งข้อมูลผ่านกันไปมาแทน) กับ **shared-state concurrency** (บทนี้ — ยอมให้
  หลาย thread เข้าถึงข้อมูลก้อนเดียวกันจริง ๆ พร้อมกัน โดยมีกลไก synchronize ควบคุมไม่ให้เกิด data race)
  และรู้ว่าเมื่อไหร่ควรเลือกแนวทางไหน
- ใช้ `Mutex<T>` (mutual exclusion) เพื่อทำ shared mutable state ข้าม thread อย่างปลอดภัย โดยเข้าใจว่ามันคือ
  "`RefCell<T>` เวอร์ชันข้าม thread" — ย้ายจุดตรวจสอบกฎ borrowing จาก runtime counter ธรรมดา (Part 28) มาเป็น
  **OS-level หรือ lightweight lock ที่ปลอดภัยข้าม thread จริง**
- อธิบายได้อย่างละเอียดว่าทำไม `.lock()` คืนค่าเป็น `LockResult<MutexGuard<T>>` (ก็คือ `Result` จาก Part 12) —
  เข้าใจกลไก **mutex poisoning** ที่เกิดขึ้นเมื่อ thread หนึ่ง panic ทั้งที่ยังถือ lock อยู่ และรู้ว่าทำไม
  `.lock().unwrap()` จึงเป็นแนวปฏิบัติที่ใช้กันจริงเมื่อไม่ต้องการกู้คืนจาก poisoning
- ใช้ `MutexGuard<T>` ในฐานะ smart pointer แบบ RAII (Part 6/27-29) ที่ deref ไปยัง `&T`/`&mut T` และปลดล็อก
  อัตโนมัติเมื่อหลุด scope พร้อมรู้ว่าทำไมการถือ guard ไว้นานเกินไปเป็นปัญหาที่ **ร้ายแรงกว่า** การถือ `Ref`/`RefMut`
  นานเกินไปใน Part 28 (เพราะตอนนี้มันบล็อก thread อื่นจริง ไม่ใช่แค่เสี่ยง panic)
- อธิบายได้ว่าทำไม `Rc<T>` (Part 28) ใช้แบ่งข้อมูลข้าม thread ไม่ได้เลย (ตัวนับไม่ thread-safe) และใช้ `Arc<T>`
  (Atomically Reference Counted) ซึ่งเป็นฝาแฝดที่ปลอดภัยข้าม thread ของ `Rc<T>` พร้อมเข้าใจแนวคิดพื้นฐานของ
  "atomic operation" ในระดับที่เพียงพอสำหรับใช้งานจริง (ไม่ต้องลงลึกทฤษฎี lock-free เต็มรูปแบบ ซึ่งเป็นเรื่องของ
  Part 51)
- ประกอบ `Arc<Mutex<T>>` เข้าด้วยกันเพื่อสร้าง shared counter ข้าม thread ที่ถูกต้อง 100% และรู้จัก `RwLock<T>`
  สำหรับสถานการณ์ที่มีการอ่านมากกว่าการเขียนมาก ๆ
- เข้าใจว่า **deadlock** คืออะไร มองเห็นตัวอย่างที่ค้างจริง วินิจฉัยได้ว่าโปรแกรมติด deadlock ได้อย่างไร (ไม่มี
  error, ไม่มี panic, แค่ค้าง) และรู้แนวทางป้องกันมาตรฐาน

## ความรู้ที่ต้องมีมาก่อน

บทนี้เป็นบทที่สามของ mini-arc เรื่อง concurrency และเป็น **คำตอบที่ Part 28 สัญญาไว้ตรง ๆ** — ถ้าคุณจำ Part 28
ได้ไม่แม่นเป็นพิเศษ แนะนำให้กลับไปทวนก่อนอ่านบทนี้ เพราะทุกแนวคิดในบทนี้ถูกสร้างขึ้นมาโดยเทียบกับ Part 28
บรรทัดต่อบรรทัดจริง ๆ:

- **Part 28 (Smart Pointers: Rc\<T\> และ RefCell\<T\>)** — นี่คือความรู้ที่จำเป็นที่สุดสำหรับบทนี้ เพราะบทนี้คือ
  **เวอร์ชันข้าม thread** ของทุกอย่างที่ Part 28 สอน: `RefCell<T>` ให้ interior mutability ผ่านการตรวจ borrow
  rule ด้วยตัวนับที่ไม่ thread-safe ตอน runtime — `Mutex<T>` ทำสิ่งเดียวกันแต่ด้วย lock ที่ thread-safe จริง
  `Rc<T>` ให้เจ้าของร่วมหลายคนด้วยตัวนับธรรมดา — `Arc<T>` ทำสิ่งเดียวกันด้วยตัวนับแบบ atomic ที่ปลอดภัยข้าม
  thread และ `Rc<RefCell<T>>` (pattern ที่พบบ่อยที่สุดใน Part 28) มี `Arc<Mutex<T>>` เป็นฝาแฝดข้าม thread ของมัน
  ตรงเป๊ะ — Part 28 เองก็ทิ้งคำใบ้เรื่องนี้ไว้หลายครั้ง (ในหัวข้อ "กับดักที่พบบ่อย" ที่แสดง error `E0277`
  `Rc<RefCell<i32>> cannot be sent between threads safely` เมื่อพยายามใช้ `Rc` ข้าม thread) — บทนี้คือคำตอบเต็ม
  รูปแบบของคำใบ้นั้น
- **Part 37 (Threads พื้นฐาน)** — ความรู้เรื่อง `std::thread::spawn`, `move` closure, และ `JoinHandle::join()`
  จะถูกใช้ในทุกตัวอย่างของบทนี้ เพราะทุกตัวอย่างต้องสร้างหลาย thread จริงเพื่อสาธิต shared-state concurrency
- **Part 38 (Channels)** — บทนี้เริ่มต้นด้วยการทวนว่า Part 38 แก้ปัญหา "หลาย thread ต้องสื่อสารกัน" ด้วยวิธี
  **ไม่แบ่งข้อมูลกันเลย** (ส่งข้อมูลผ่าน channel แทน) — บทนี้จะแก้ปัญหาแบบเดียวกันด้วยวิธีตรงข้าม คือ **แบ่งข้อมูล
  กันจริง ๆ** อย่างปลอดภัย ทั้งสองแนวทางแก้ปัญหาประเภทเดียวกันได้ คุณควรเข้าใจทั้งคู่เพื่อเลือกใช้ให้ถูกกับสถานการณ์
- **Part 12 (Result\<T, E\> และ Error Handling เบื้องต้น)** — `.lock()` คืนค่าเป็น `LockResult<MutexGuard<T>>`
  ซึ่งก็คือ `Result<MutexGuard<T>, PoisonError<MutexGuard<T>>>` — ถ้าคุณยังไม่คุ้นกับ `Result`, `.unwrap()`,
  หรือ `match` บน `Result` ควรทวน Part 12 ก่อน
- **Part 6 (Ownership เบื้องต้น และ Drop)** — `MutexGuard<T>` ทำงานแบบ RAII (Resource Acquisition Is
  Initialization) เหมือนกับที่ `Drop` trait ทำใน Part 6 — ปลดล็อกอัตโนมัติตอนหลุด scope โดยไม่ต้องเรียกอะไรเอง
- **Part 15 (Collections: HashMap)** — ตัวอย่างระบบธนาคารจำลองท้ายบทใช้ `HashMap<String, f64>` เก็บยอดบัญชี
- **Part 21 (Traits ขั้นสูง)** และ Part 28/37 — คำว่า `Send`/`Sync` ถูกพูดถึงแบบผ่าน ๆ มาตั้งแต่ Part 21/28/37
  แล้ว (โดยเฉพาะ error `E0277` ที่บอกว่า type หนึ่ง "ไม่ implement `Send`") — บทนี้จะใช้คำสองคำนี้ **แบบไม่เป็น
  ทางการ** ต่อไปอีก (แค่พอให้เข้าใจว่า "ปลอดภัยที่จะย้ายข้าม thread" กับ "ปลอดภัยที่จะแบ่งกันใช้ข้าม thread"
  หมายถึงอะไร) — คำนิยามที่เป็นทางการเต็มรูปแบบ (trait `Send`/`Sync` คืออะไรจริง ๆ, compiler ตัดสินใจให้ type
  ไหน implement มันโดยอัตโนมัติอย่างไร, `unsafe impl` คืออะไร) จะเรียนเต็ม ๆ ใน **Part 40** ทันทีหลังบทนี้

## เนื้อหา

### 39.1 ทบทวน: Message Passing แก้ปัญหาอะไร และเหลือปัญหาอะไรไว้

จาก Part 38 เราเรียนรู้ว่า `std::sync::mpsc` (multiple producer, single consumer) แก้ปัญหา "หลาย thread ต้อง
ทำงานร่วมกัน" ด้วยแนวคิดที่ชัดเจนมาก: **อย่าให้สอง thread แบ่งข้อมูลกันเลย** — แทนที่จะให้ทั้งสอง thread เข้าถึง
ตัวแปรตัวเดียวกันพร้อมกัน เราให้ thread หนึ่ง **ส่ง (send)** ค่าที่ตัวเองเป็นเจ้าของทั้งหมดผ่าน channel ไปให้
thread อีกตัว — ownership ของค่านั้นถูก **ย้าย (move)** ไปพร้อมกับข้อความ ไม่มีใครเหลือสิทธิ์เข้าถึงค่านั้นอีก
ฝั่งต้นทาง ผลคือไม่มีทางเกิด data race ได้เลย เพราะไม่มีจุดใดในโปรแกรมที่สอง thread ถือสิทธิ์เข้าถึงข้อมูลก้อน
เดียวกันพร้อมกันแม้แต่ชั่วขณะเดียว

แนวคิดนี้ตรงกับคำขวัญที่โด่งดังของ Go (ภาษาที่เน้น concurrency เหมือนกัน): *"Do not communicate by sharing
memory; instead, share memory by communicating."* — Rust ยืมแนวคิดนี้มาเต็ม ๆ ผ่าน channel เช่นกัน

แต่ message passing ไม่ใช่คำตอบสำหรับทุกสถานการณ์ ลองนึกภาพสถานการณ์เหล่านี้ที่ message passing เริ่มอึดอัด:

1. **สถิติที่ต้องอัปเดตร่วมกันจากหลาย thread พร้อมกัน** เช่น ตัวนับจำนวน request ที่ server ประมวลผลไปแล้ว —
   ถ้าจะใช้ channel คุณต้องมี "thread ผู้รวบรวมผลรวม" แยกออกมาอีกตัวหนึ่งที่คอยรับข้อความ "+1" จากทุก worker
   thread แล้วบวกสะสมเอง ซึ่งเพิ่มความซับซ้อนโดยไม่จำเป็น ถ้าเทียบกับการให้ทุก thread บวกเข้าตัวแปรที่แบ่งกันใช้
   ตรง ๆ ได้เลย
2. **ข้อมูลก้อนใหญ่ที่หลาย thread ต้อง "อ่านเป็นหลัก" พร้อมกัน** (เช่น cache, ตารางค่า configuration ที่โหลด
   มาแล้ว) — การส่งข้อมูลก้อนใหญ่ผ่าน channel ซ้ำ ๆ ให้ทุก thread ที่ต้องการอ่าน สิ้นเปลืองทั้งเวลาและหน่วยความจำ
   เกินความจำเป็นมาก ถ้าเทียบกับการให้ทุก thread "มองไปที่ข้อมูลก้อนเดียวกัน" ได้ตรง ๆ
3. **โครงสร้างข้อมูลที่ธรรมชาติของมันคือ "หลายคนเข้าถึงพร้อมกัน" อยู่แล้ว** เช่นตัวอย่างระบบธนาคารที่จะเจอท้าย
   บทนี้ — บัญชีธนาคารก้อนเดียวถูกทั้งอ่านและแก้ไขจากหลาย ATM (หลาย thread) พร้อมกันโดยธรรมชาติของปัญหาเอง
   ไม่ใช่แค่ "ทางเลือกในการออกแบบ"

สถานการณ์เหล่านี้เหมาะกับแนวทางที่สอง คือ **shared-state concurrency**: ยอมให้หลาย thread เข้าถึงข้อมูลก้อน
เดียวกันจริง ๆ พร้อมกัน แต่ใส่กลไกบางอย่างเข้าไปคุมไม่ให้เกิด data race — นี่คือสิ่งที่บทนี้จะสอน และเครื่องมือ
หลักตัวแรกคือ `Mutex<T>`

สังเกตว่าคำว่า "shared-state" ในที่นี้คือ **สิ่งเดียวกันเป๊ะ** กับที่ Part 28 เรียกว่า "shared mutable state" —
เพียงแค่ตอนนี้ "ผู้แบ่งกันใช้" คือหลาย thread จริง ๆ ไม่ใช่แค่หลายส่วนของโค้ดใน thread เดียว

### 39.2 Mutex\<T\>: RefCell\<T\> เวอร์ชันข้าม Thread

จาก Part 28 เราเรียนรู้ว่า `RefCell<T>` ให้ **interior mutability** — แก้ไขข้อมูลได้ผ่าน `&T` (ไม่ต้องมี
`&mut T`) โดยย้ายการตรวจกฎ borrowing ข้อที่ 1 จาก Part 7 (mutable ได้แค่ตัวเดียว หรือ immutable ได้หลายตัว
ไม่ทั้งสองพร้อมกัน) จาก compile time (borrow checker) มาเป็น runtime (ตัวนับภายในของ `RefCell` เอง) —
`.borrow()`/`.borrow_mut()` คืน `Ref<T>`/`RefMut<T>` ที่ตรวจสอบกฎนี้แบบ dynamic

`Mutex<T>` (`std::sync::Mutex`, ชื่อย่อมาจาก **Mut**ual **Ex**clusion — "การกันไม่ให้เข้าถึงร่วมกัน") ทำสิ่งที่
**concept เดียวกันเป๊ะ** แต่ยกระดับขึ้นไปอีกขั้น: มันให้ interior mutability ที่ **ปลอดภัยข้าม thread จริง** โดย
ใช้ lock ที่ระดับ OS หรือ userspace ที่เบากว่า (ขึ้นกับ platform และ implementation ภายในของ standard library)
แทนตัวนับธรรมดาของ `RefCell<T>`

มาดูวิธีใช้งานพื้นฐานก่อน:

```rust
use std::sync::Mutex;

fn main() {
    let m = Mutex::new(5);

    {
        let mut num = m.lock().unwrap();
        *num += 1;
    } // guard หลุด scope ที่นี่ -> ปลดล็อกอัตโนมัติ

    println!("m = {:?}", m);
}
```

ผลลัพธ์ (รันจริง):

```
m = Mutex { data: 6, poisoned: false, .. }
```

โครงสร้างนี้คล้ายกับ `RefCell<T>` มากจนน่าตกใจ:

| ขั้นตอน | `RefCell<T>` (Part 28, single-thread) | `Mutex<T>` (บทนี้, multi-thread) |
|---|---|---|
| สร้างค่า | `RefCell::new(5)` | `Mutex::new(5)` |
| ขอสิทธิ์แก้ไข | `.borrow_mut()` → `RefMut<T>` | `.lock()` → (ผ่าน `Result`) `MutexGuard<T>` |
| ขอสิทธิ์อ่าน | `.borrow()` → `Ref<T>` | `.lock()` เหมือนกัน (ไม่มี read-only variant แยก — ดู `RwLock<T>` ในหัวข้อ 39.7 สำหรับ multi-reader) |
| แก้ไขค่า | `*ref_mut += 1;` | `*guard += 1;` |
| ปลดล็อก/คืนสิทธิ์ | หลุด scope (Drop) อัตโนมัติ | หลุด scope (Drop) อัตโนมัติ **เหมือนกันทุกประการ** |
| ถ้าละเมิดกฎ | panic (`BorrowError`/`BorrowMutError`) | **ไม่ panic** — thread ที่มาทีหลัง **รอ (block)** จนกว่า lock จะถูกปลด |

จุดที่ต่างที่สุดและสำคัญที่สุดอยู่ที่แถวสุดท้าย: `RefCell<T>` ถูกออกแบบมาให้ใช้ใน **thread เดียว** เท่านั้น
ดังนั้นเมื่อละเมิดกฎ (เช่นเรียก `.borrow_mut()` ซ้อนกันสองครั้ง) มันไม่มีทางเลือกอื่นนอกจาก panic ทันที เพราะไม่มี
"ใครอื่น" ให้รอ — ทุกอย่างเกิดในเธรดเดียวกัน แต่ `Mutex<T>` ถูกออกแบบมาสำหรับสถานการณ์ที่ **มีจริง ๆ หลาย thread
ที่อาจแข่งกันขอสิทธิ์เข้าถึงพร้อมกัน** ดังนั้นเมื่อ thread หนึ่งถือ lock อยู่แล้ว thread อื่นที่มาขอ `.lock()`
ทีหลังจะ **ไม่ panic แต่จะหยุดรอ (block)** อยู่ตรงนั้นจนกว่า thread แรกจะปลดล็อก — นี่คือหัวใจของคำว่า
"mutual exclusion": ณ เวลาใดเวลาหนึ่ง มี**อย่างมากที่สุดหนึ่ง thread**ที่ถือสิทธิ์เข้าถึงข้อมูลข้างในได้ ส่วนคนอื่น
ที่มาทีหลังต้องรอเข้าคิว

### 39.3 ทำไม `.lock()` คืนค่าเป็น `Result`: Mutex Poisoning

สังเกตว่าในตัวอย่างข้างบนเราเขียน `m.lock().unwrap()` — ทำไมต้อง `.unwrap()` ด้วย ในเมื่อ `RefCell::borrow_mut()`
ของ Part 28 ไม่คืน `Result` เลย (มันคืน `RefMut<T>` ตรง ๆ แล้ว panic เองถ้าผิดกฎ)?

คำตอบ: type ของ `.lock()` คือ `LockResult<MutexGuard<T>>` ซึ่งก็คือ type alias ของ
`Result<MutexGuard<T>, PoisonError<MutexGuard<T>>>` (ใช้ `Result` จาก Part 12 ตรง ๆ) — มาดูให้เห็น type ชัด ๆ:

```rust
use std::sync::{Mutex, MutexGuard, LockResult};

fn main() {
    let m = Mutex::new(String::from("hello"));

    // แสดง type ของ .lock() อย่างชัดเจนว่าเป็น LockResult<MutexGuard<T>>
    let result: LockResult<MutexGuard<String>> = m.lock();
    match result {
        Ok(guard) => println!("ได้ lock มา, value = {}", *guard),
        Err(poisoned) => println!("mutex ถูก poison แล้ว: {:?}", poisoned),
    }
}
```

ผลลัพธ์ (รันจริง):

```
ได้ lock มา, value = hello
```

คำถามคือ: `.lock()` จะคืน `Err` ตอนไหน? คำตอบคือเมื่อ **mutex ถูก "poison" (เป็นพิษ)** — สถานการณ์นี้เกิดขึ้น
เมื่อ thread หนึ่ง **panic ขณะที่ยังถือ lock (`MutexGuard`) อยู่** โดยไม่ได้ปลดล็อกก่อน (ซึ่งเกิดขึ้นเองโดย
อัตโนมัติผ่าน mechanism เดียวกับ unwinding ใน Part 30 ตอน panic แต่ปัญหาคือ **ข้อมูลข้างในอาจถูกแก้ไขไปครึ่ง ๆ
กลาง ๆ** ก่อนที่จะ panic) มาดูสถานการณ์นี้เกิดขึ้นจริง:

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let data = Arc::new(Mutex::new(vec![1, 2, 3]));

    let data_clone = Arc::clone(&data);
    let handle = thread::spawn(move || {
        let mut guard = data_clone.lock().unwrap();
        guard.push(4);
        panic!("thread นี้ panic ทั้งที่ยังถือ lock อยู่!");
    });

    // รอ thread จบ แต่มันจะ panic จริง -> join คืน Err
    let join_result = handle.join();
    println!("join_result เป็น Err: {}", join_result.is_err());

    // ตอนนี้ mutex ถูก poison แล้ว
    match data.lock() {
        Ok(guard) => println!("lock สำเร็จ (ไม่ควรเกิด): {:?}", *guard),
        Err(poison_error) => {
            println!("lock คืน Err เพราะ mutex ถูก poison แล้ว");
            // เราสามารถดึงข้อมูลออกมาดูได้ต่อผ่าน into_inner()
            let guard = poison_error.into_inner();
            println!("แต่ข้อมูลข้างในยังอยู่และดูได้: {:?}", *guard);
        }
    };
}
```

ผลลัพธ์ (รันจริง — ข้อความ panic backtrace ตัดบางส่วนเพื่อความกระชับ):

```
thread '<unnamed>' panicked at src/main.rs:11:9:
thread นี้ panic ทั้งที่ยังถือ lock อยู่!
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
join_result เป็น Err: true
lock คืน Err เพราะ mutex ถูก poison แล้ว
แต่ข้อมูลข้างในยังอยู่และดูได้: [1, 2, 3, 4]
```

(หมายเหตุใช้ `RUST_BACKTRACE` เฉพาะโหมด debug และการรัน `panic!` จะพิมพ์ข้อความไปที่ stderr เสมอตาม Part 30 —
โปรแกรมยังคงรันต่อได้ตามปกติหลัง panic เพราะ panic เกิดใน thread ลูก ไม่ใช่ thread `main`)

ทำไม standard library ต้องออกแบบมาแบบนี้ (แทนที่จะปล่อยให้ thread อื่นเข้าถึงข้อมูลต่อไปเงียบ ๆ)? นี่คือ
**trade-off เชิงออกแบบที่ต้องเข้าใจให้ตรงจุด**: เหตุผลคือ **ไม่มีทางรู้ได้แน่นอนว่าข้อมูลข้างใน `Mutex<T>`
ยังอยู่ในสถานะที่ถูกต้อง (consistent state) หรือไม่** หลังจาก thread ที่แก้ไขมันค้าง panic กลางทาง — ในตัวอย่าง
ข้างบน `guard.push(4)` สำเร็จไปแล้วก่อน panic จริง แต่ลองนึกภาพสถานการณ์ที่ซับซ้อนกว่า เช่นฟังก์ชันหนึ่งกำลังแก้
ไขข้อมูลหลายฟิลด์ที่ต้อง**สัมพันธ์กันเสมอ** (invariant) เช่น กำลังย้ายเงินจากบัญชี A ไปบัญชี B — ถ้า panic
เกิดขึ้น**หลังจากหักเงินจาก A แล้ว แต่ก่อนที่จะเพิ่มเงินให้ B** ข้อมูลข้างใน `Mutex` จะอยู่ในสถานะที่เงินหายไป
กลางทาง ถ้า thread ถัดไปที่มา `.lock()` ได้ข้อมูลนี้ไปใช้ต่อแบบไม่รู้ตัวว่ามีอะไรผิดปกติ มันจะทำงานผิดพลาดต่อไป
เรื่อย ๆ โดยไม่มีสัญญาณเตือนใด ๆ เลย — Rust เลือกที่จะ **เตือนอย่างชัดเจนแทนที่จะเงียบ** โดยการ "poison" mutex
ทันทีที่เกิด panic ขณะถือ lock: เจ้าของ lock ตัวถัดไปทุกคนจะได้รับ `Err` เพื่อบังคับให้ผู้เขียนโค้ด **ตัดสินใจเอง**
ว่าจะทำอย่างไรต่อ (เชื่อข้อมูลต่อได้ไหม, กู้คืนอย่างไร, หรือ panic ตามไปด้วยเลย)

ในทางปฏิบัติจริง โปรแกรม Rust ส่วนใหญ่ **ไม่ได้เขียน logic กู้คืนจาก poisoning อย่างละเอียด** เพราะในหลาย ๆ
กรณี ถ้า thread หนึ่ง panic ขณะถือ lock แปลว่ามี bug ร้ายแรงเกิดขึ้นแล้วในโปรแกรม — การที่ thread อื่นจะพยายาม
"เดา" ว่าข้อมูลยังใช้ต่อได้ปลอดภัยแค่ไหนมักไม่คุ้มกับความซับซ้อนที่เพิ่มขึ้น ดังนั้นแนวปฏิบัติที่ใช้กันจริงในโค้ด
จำนวนมาก (รวมถึงโค้ดตัวอย่างส่วนใหญ่ในบทนี้) คือการเขียน **`.lock().unwrap()`** ตรง ๆ — ถ้า mutex ไม่ถูก poison
มันจะได้ `MutexGuard<T>` มาใช้งานตามปกติ แต่ถ้า mutex ถูก poison จริง (แปลว่ามี thread อื่น panic ไปก่อนแล้ว)
`.unwrap()` จะ panic ตามไปด้วยทันที ซึ่งเป็นพฤติกรรมที่ **เข้าใจง่ายและปลอดภัยกว่า** การพยายามฝืนใช้ข้อมูลที่อาจ
เสียหายไปแล้วต่อแบบเงียบ ๆ

ถ้าคุณต้องการกู้คืนจริง ๆ (เช่นในระบบที่ต้องทนต่อความผิดพลาดสูงมาก) `PoisonError<T>` มี method
`.into_inner()`/`.get_ref()`/`.get_mut()` ให้ดึงข้อมูลข้างในออกมาใช้ต่อได้ทั้งที่รู้ตัวว่ามันอาจไม่ consistent
100% — ตามที่เห็นในตัวอย่างข้างบน (`poison_error.into_inner()`)

### 39.4 MutexGuard\<T\>: Smart Pointer แบบ RAII

`m.lock().unwrap()` ในตัวอย่างข้างบนคืนค่าที่ชื่อ `MutexGuard<T>` — เช่นเดียวกับ `Ref<T>`/`RefMut<T>` ของ
`RefCell<T>` ใน Part 28, `Box<T>` ใน Part 27, และ `Rc<T>` ใน Part 28 เอง `MutexGuard<T>` คือ **smart pointer**
อีกตัวหนึ่งในตระกูลเดียวกัน — มันไม่ได้เก็บข้อมูล `T` ไว้ในตัวมันเองตรง ๆ แต่ implement `Deref`/`DerefMut` เพื่อ
"ส่งผ่าน" การเข้าถึงไปยัง `T` ที่อยู่ข้างใน `Mutex<T>` ทำให้เขียน `*guard` หรือเรียก method ของ `T` ผ่าน `guard`
ได้ตรง ๆ เหมือนกับว่ามันคือ `&mut T` เอง

จุดที่ทำให้ `MutexGuard<T>` ทรงพลังและปลอดภัยคือมันใช้ pattern **RAII (Resource Acquisition Is Initialization)**
เต็มรูปแบบผ่าน `Drop` trait ที่เรียนมาจาก Part 6: การขอ lock (`m.lock()`) กับการได้ smart pointer มาถือ
(`MutexGuard<T>`) คือการกระทำเดียวกัน — และเมื่อ `MutexGuard<T>` นั้นหลุด scope (drop) ไม่ว่าจะจากการ return
ปกติ, จบ block `{}`, หรือแม้แต่ตอน panic (unwinding ระหว่างทาง) `Drop::drop` ของมันจะถูกเรียกโดยอัตโนมัติเพื่อ
**ปลดล็อก mutex ทันที** — ผู้เขียนโค้ด**ไม่ต้องเรียก `.unlock()` เองเลย** และที่สำคัญกว่านั้นคือ **ไม่มีทางลืม
ปลดล็อกได้เลย** เพราะ compiler บังคับใช้ `Drop` ให้เสมอ ตรงข้ามกับภาษาที่ใช้ lock แบบ manual (เช่น
`pthread_mutex_lock`/`pthread_mutex_unlock` ใน C) ที่ผู้เขียนโค้ดต้องจำเรียก unlock เองในทุกเส้นทางที่โค้ดอาจ
ออกจากฟังก์ชัน (รวมถึงเส้นทาง error/early return ที่มักถูกมองข้าม) — ถ้าลืมแม้แต่เส้นทางเดียว โปรแกรมจะ deadlock
ตัวเองทันที

มาดูตัวอย่างที่แสดงให้เห็นการ block thread อื่นจริง ๆ เมื่อ lock ถูกถือไว้:

```rust
use std::sync::{Arc, Mutex};
use std::thread;
use std::time::{Duration, Instant};

fn main() {
    let data = Arc::new(Mutex::new(0));

    let data_a = Arc::clone(&data);
    let start = Instant::now();
    let handle_a = thread::spawn(move || {
        let guard = data_a.lock().unwrap();
        // จำลองงานหนักขณะยังถือ guard อยู่ -- นี่คือ "ถือ lock นานเกินไป"
        thread::sleep(Duration::from_millis(300));
        println!("[thread A] ปล่อย lock แล้วที่ {:?}", start.elapsed());
        drop(guard); // ปล่อยชัดเจน (จริง ๆ หลุด scope ก็ปล่อยเหมือนกัน)
    });

    // ให้ thread A เริ่มถือ lock ก่อนแน่ ๆ
    thread::sleep(Duration::from_millis(50));

    let data_b = Arc::clone(&data);
    let handle_b = thread::spawn(move || {
        println!("[thread B] กำลังรอ lock ที่ {:?}", start.elapsed());
        let _guard = data_b.lock().unwrap();
        println!("[thread B] ได้ lock แล้วที่ {:?}", start.elapsed());
    });

    handle_a.join().unwrap();
    handle_b.join().unwrap();
}
```

ผลลัพธ์ (รันจริง — ตัวเลขเวลาอาจเปลี่ยนแปลงเล็กน้อยในแต่ละครั้งที่รันตามธรรมชาติของ scheduler แต่ **ลำดับ
เหตุการณ์และช่วงเวลารอจะเหมือนกันเสมอ**):

```
[thread B] กำลังรอ lock ที่ 50.400022ms
[thread A] ปล่อย lock แล้วที่ 300.265288ms
[thread B] ได้ lock แล้วที่ 300.417766ms
```

สังเกตให้ชัด: thread B เริ่มขอ lock ตั้งแต่นาทีที่ ~50ms แต่ **ต้องรอจนถึง ~300ms** (จนกว่า thread A จะปล่อย
lock) จึงได้ lock มาจริง — นี่คือการพิสูจน์ตรง ๆ ว่า `.lock()` **บล็อก (block)** thread ที่เรียกไว้จนกว่าจะได้
สิทธิ์เข้าถึงจริง ไม่ใช่ล้มเหลวหรือคืนค่าอะไรทันทีแบบ non-blocking

จาก Part 28 คุณเคยเรียนคำแนะนำ "ควรจำกัด scope ของ `.borrow()`/`.borrow_mut()` ให้เล็กที่สุด" เพื่อป้องกัน
`BorrowError`/`BorrowMutError` — คำแนะนำเดียวกันนี้ **ยังใช้ได้เป๊ะกับ `MutexGuard<T>`** แต่มีเหตุผลเพิ่มเข้ามา
อีกข้อที่ **ร้ายแรงกว่าเดิม**:

| ผลที่เกิดจากการถือ guard นานเกินไป | `RefCell<T>` (Part 28, single-thread) | `Mutex<T>` (บทนี้, multi-thread) |
|---|---|---|
| ผลต่อ thread/โค้ดส่วนอื่นในตัวเอง | เสี่ยง panic ถ้าเผลอเรียก `.borrow()`/`.borrow_mut()` ซ้อนกันในโค้ดเส้นทางเดียวกัน | เหมือนกัน (ดูหัวข้อ "กับดักที่พบบ่อย" เรื่อง self-deadlock) |
| ผลต่อผู้อื่นที่ต้องการเข้าถึงข้อมูลเดียวกัน | **ไม่มีผล** เพราะเป็น single-thread — ไม่มี "คนอื่น" ที่รอพร้อมกันได้จริง | **ผู้อื่นทุก thread ที่ต้องการ lock ตัวเดียวกันต้องหยุดรอ (block)** จนกว่า guard จะหลุด scope — ถ้าถือไว้นานเกินจำเป็น (เช่น ทำ I/O ช้า ๆ ขณะถือ lock) ประสิทธิภาพของ**ทั้งระบบ**ตกลงทันที เพราะ thread อื่นทำงานคู่ขนานไม่ได้เลยตราบใดที่ต้องรอ lock ตัวนี้ |

นี่คือเหตุผลที่แนวปฏิบัติมาตรฐานของ Rust คือ **"lock ให้สั้นที่สุด, ทำงานเบาที่สุดเท่าที่ทำได้ขณะถือ lock"** —
ถ้าต้องทำงานหนัก (คำนวณเยอะ, เรียก I/O, เรียก network) ให้ดึงข้อมูลที่ต้องการออกมาจาก guard ก่อน (เช่น
`.clone()` ข้อมูลที่จำเป็นออกมา แล้วปล่อย guard ทันที) แล้วค่อยไปทำงานหนักโดยไม่ถือ lock อยู่ ตัวอย่าง:

```rust
use std::sync::{Arc, Mutex};

fn expensive_computation(value: i32) -> i32 {
    // สมมติว่านี่คืองานที่ใช้เวลานาน
    value * 2
}

fn bad_pattern(shared: &Arc<Mutex<i32>>) -> i32 {
    let guard = shared.lock().unwrap();
    let result = expensive_computation(*guard); // เรียกงานหนักขณะยังถือ lock อยู่ -- ไม่ดี
    result
}

fn good_pattern(shared: &Arc<Mutex<i32>>) -> i32 {
    let value = {
        let guard = shared.lock().unwrap();
        *guard // copy ค่าออกมา แล้ว guard หลุด scope ทันทีตรงนี้
    };
    expensive_computation(value) // ทำงานหนักโดยไม่ถือ lock อยู่เลย -- thread อื่นเข้าถึง shared ได้ระหว่างนี้
}
```

`bad_pattern` ถือ lock ไว้ตลอดเวลาที่ `expensive_computation` กำลังทำงาน (ซึ่งอาจนานมาก) ทำให้ทุก thread อื่นที่
ต้องการ `shared` ตัวเดียวกันต้องรอจนกว่าการคำนวณทั้งหมดจะเสร็จ — ในทางกลับกัน `good_pattern` จำกัด scope ของ
`guard` ด้วย block `{}` ให้ปล่อย lock ทันทีหลังจากคัดลอกค่าที่ต้องการออกมา (ใน `i32` ที่ `Copy` ได้ `*guard`
คือการ copy ค่าออกมาเป็นตัวแปรใหม่บน stack ไม่ใช่การถือ reference เข้าไปในข้อมูลข้างใน `Mutex` อีกต่อไป) แล้ว
ค่อยทำงานหนักโดยไม่มีการถือ lock อยู่เลย — thread อื่นสามารถเข้าถึง `shared` ได้ตามปกติระหว่างที่
`expensive_computation` กำลังทำงานอยู่คู่ขนาน

### 39.5 ทำไมแบ่ง Mutex\<T\> ตรง ๆ ข้าม Thread ไม่ได้

ทีนี้มาถึงคำถามที่นำไปสู่หัวใจของบทนี้: ถ้าเราต้องการให้ **หลาย thread** เข้าถึง `Mutex<T>` ตัวเดียวกันพร้อมกัน
(ซึ่งเป็นเหตุผลทั้งหมดที่เราใช้ `Mutex<T>` ตั้งแต่แรก) เราจะ `move` มันเข้าไปในหลาย thread ได้อย่างไร? ลองเขียน
ตรง ๆ แบบไม่คิดมากดูก่อน — สร้าง counter ตัวเดียว แล้วให้ 3 thread ต่างเพิ่มค่ามันคนละครั้ง:

```rust
use std::sync::Mutex;
use std::thread;

fn main() {
    let counter = Mutex::new(0);
    let mut handles = vec![];

    for _ in 0..3 {
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();
            *num += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Result: {}", *counter.lock().unwrap());
}
```

โค้ดนี้ **compile ไม่ผ่าน**:

```
error[E0382]: borrow of moved value: `counter`
  --> src/main.rs:20:29
   |
 5 |     let counter = Mutex::new(0);
   |         ------- move occurs because `counter` has type `std::sync::Mutex<i32>`, which does not implement the `Copy` trait
...
 8 |     for _ in 0..3 {
   |     ------------- inside of this loop
 9 |         let handle = thread::spawn(move || {
   |                                    ------- value moved into closure here, in previous iteration of loop
...
20 |     println!("Result: {}", *counter.lock().unwrap());
   |                             ^^^^^^^ value borrowed here after move

error: aborting due to 1 previous error

For more information about this error, try `rustc --explain E0382`.
```

นี่คือ error **`E0382`** ตัวเดียวกับที่เจอครั้งแรกใน Part 6 (และเจอซ้ำใน Part 28 ตอนพยายามให้ `Box<Config>` มี
สองเจ้าของ) — เหตุผลตรงไปตรงมาเป๊ะตามกฎ ownership: `move ||` ใน `thread::spawn` **ย้าย ownership** ของ `counter`
เข้าไปใน closure ตัวแรกที่สร้างขึ้นในรอบแรกของ loop ทันที — พอถึงรอบที่สองของ loop `counter` ก็ไม่มีอยู่ให้ใช้อีก
แล้ว (ถูก move ไปแล้ว) ยิ่งไปกว่านั้น บรรทัดสุดท้ายที่พยายามอ่าน `*counter.lock().unwrap()` ใน `main` ก็ใช้ไม่ได้
เช่นกันเพราะ `counter` ถูก move เข้าไปใน thread ไปแล้วตั้งแต่รอบแรก

จุดสำคัญที่ต้องเข้าใจให้ลึกคือ: **นี่ไม่ใช่ปัญหาของ `Mutex<T>` เอง** — `Mutex<T>` ทำงานถูกต้องสมบูรณ์แบบสำหรับ
"ข้อมูลหนึ่งก้อนที่มีเจ้าของหนึ่งคนแต่ต้องแก้ไขได้จากหลายจุด" แต่ปัญหาคือ **การมี "เจ้าของ" ได้แค่คนเดียวเป็น
ข้อจำกัดจากกฎ ownership ของ Rust เอง (Part 6) ไม่ใช่จาก `Mutex` เลย** — `move` เข้า thread ก็คือการโยก ownership
ไปให้อีกคนถือแค่คนเดียวเหมือนโยก ownership ปกติทุกประการ พอ thread แรกเป็นเจ้าของ `counter` ไปแล้ว thread ที่สอง
และสามก็ไม่มีสิทธิ์เป็นเจ้าของร่วมได้อีก — **นี่คือปัญหาเดียวกันเป๊ะที่ Part 28 เจอตอนพยายามให้ `Box<Config>`
มีสอง `Server` เป็นเจ้าของพร้อมกัน** (error `E0382` ตัวเดียวกัน) เพียงแค่เปลี่ยนจาก "หลาย struct เป็นเจ้าของ
ร่วม" มาเป็น "หลาย thread เป็นเจ้าของร่วม"

และคำตอบก็เหมือนกันเป๊ะกับที่ Part 28 เจอ: เราต้องการเครื่องมือที่ให้ **เจ้าของร่วมหลายคนพร้อมกันจริง ๆ** — ใน
Part 28 คำตอบคือ `Rc<T>` แต่คำตอบนั้นใช้กับ thread ไม่ได้ (จะเห็นในหัวข้อถัดไปว่าทำไม) เราจึงต้องการ **ฝาแฝดที่
ปลอดภัยข้าม thread ของ `Rc<T>`** — นั่นคือ `Arc<T>`

### 39.6 ทำไม Rc\<T\> ใช้ข้าม Thread ไม่ได้: กำเนิดของ Arc\<T\>

คำถามต่อไปที่เป็นธรรมชาติมาก: ในเมื่อเรารู้จัก `Rc<T>` จาก Part 28 อยู่แล้วว่ามันให้เจ้าของร่วมหลายคนได้พอดี
ทำไมเราไม่ใช้ `Rc<Mutex<T>>` เลย? ลองดูสิ่งที่เกิดขึ้นจริงถ้าทำแบบนั้น:

```rust
use std::rc::Rc;
use std::sync::Mutex;
use std::thread;

fn main() {
    let counter = Rc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..3 {
        let counter = Rc::clone(&counter);
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();
            *num += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Result: {}", *counter.lock().unwrap());
}
```

โค้ดนี้ **compile ไม่ผ่าน** เช่นกัน แต่ด้วย error คนละตัวจากเดิม:

```
error[E0277]: `Rc<std::sync::Mutex<i32>>` cannot be sent between threads safely
  --> src/main.rs:11:36
   |
11 |         let handle = thread::spawn(move || {
   |                      ------------- ^------
   |                      |             |
   |  ____________________|_____________within this `{closure@src/main.rs:11:36: 11:43}`
   | |                    |
   | |                    required by a bound introduced by this call
12 | |             let mut num = counter.lock().unwrap();
13 | |             *num += 1;
14 | |         });
   | |_________^ `Rc<std::sync::Mutex<i32>>` cannot be sent between threads safely
   |
   = help: within `{closure@src/main.rs:11:36: 11:43}`, the trait `Send` is not implemented for `Rc<std::sync::Mutex<i32>>`
note: required because it's used within this closure
  --> src/main.rs:11:36
   |
11 |         let handle = thread::spawn(move || {
   |                                    ^^^^^^^
note: required by a bound in `spawn`

error: aborting due to 1 previous error

For more information about this error, try `rustc --explain E0277`.
```

นี่คือ **`E0277`** ตัวเดียวกันเป๊ะกับที่ Part 28 และ Part 37 เคยทิ้งคำใบ้ไว้แบบผ่าน ๆ — ตอนนี้คือจุดที่บทนี้
อธิบายเต็มรูปแบบ: `thread::spawn` ต้องการให้ closure ที่ส่งเข้าไป (และทุก type ที่ closure นั้น capture ไว้)
implement trait `Send` — trait ที่บอกว่า **"ปลอดภัยที่จะย้าย (move) ค่า type นี้ข้าม thread ได้"** (นิยามที่เป็น
ทางการเต็ม ๆ รอไว้ Part 40) แต่ **`Rc<T>` ไม่ implement `Send` เลยไม่ว่า `T` จะเป็น type ไหนก็ตาม** — และนี่คือ
เหตุผลเชิงเทคนิคที่ลึกและสำคัญที่สุดของบทนี้ ต้องเข้าใจให้ทะลุ:

#### หัวใจของปัญหา: ตัวนับของ Rc\<T\> เองก็เป็น Shared Mutable State

จาก Part 28 เราเรียนมาว่า `Rc::clone()` ทำสองอย่าง: (1) copy ตัวชี้ และ (2) **เพิ่มตัวนับ `strong_count` ขึ้น
หนึ่ง** — และตอน `Rc<T>` หลุด scope ตัวนับจะ**ลดลงหนึ่ง** เช่นกัน ปัญหาคือ **การเพิ่ม/ลดตัวเลขนี้ ไม่ได้ใช้กลไก
ป้องกัน race condition ใด ๆ เลย** — มันเป็นแค่การบวก/ลบเลขจำนวนเต็มธรรมดา (`count += 1` / `count -= 1`) แบบที่
เร็วที่สุดเท่าที่จะเป็นไปได้ เพราะ `Rc<T>` ถูกออกแบบมาโดยตั้งใจให้ใช้ใน **single-thread เท่านั้น** — ในโลกที่มี
แค่ thread เดียว การบวก/ลบเลขแบบธรรมดาไม่มีทางเกิดปัญหาได้เลย เพราะไม่มีใครมาแทรกกลางทางได้

แต่ถ้าเราเอา `Rc<T>` ไปใช้ข้าม thread จริง ๆ (สมมติว่า compiler ไม่ห้าม) ลองนึกภาพสองเธรดเรียก `Rc::clone()`
พร้อมกันเป๊ะ ๆ (หรือเธรดหนึ่งเรียก `clone` ขณะอีกเธรดกำลัง `drop`) — การบวก/ลบเลขจำนวนเต็มธรรมดาที่ระดับ CPU
จริง ๆ ไม่ใช่การกระทำแบบ "ครั้งเดียวจบ" (atomic) แต่ประกอบด้วยหลายขั้นตอนย่อยซ่อนอยู่: **(1) อ่านค่าปัจจุบันของ
ตัวนับ (2) บวกหนึ่งเข้าไปในค่านั้น (3) เขียนค่าใหม่กลับไปที่ตำแหน่งเดิม** — ถ้าสอง thread ทำสามขั้นตอนนี้สลับกัน
พร้อม ๆ กัน (เช่น ทั้งคู่อ่านค่าเดิมพร้อมกันก่อนที่ใครจะเขียนค่าใหม่ทัน) ผลลัพธ์ที่ได้อาจกลายเป็น "เพิ่มแค่ครั้ง
เดียว" ทั้งที่จริง ๆ ควรเพิ่มสองครั้ง — นี่คือ **data race บนตัวนับของ `Rc<T>` เอง** ซึ่งร้ายแรงกว่าที่คิด: มัน
อาจทำให้ `strong_count` นับผิดจนถึงจุดที่ข้อมูลถูกทำลาย (`drop`) ทั้งที่ยังมีคนอื่นถืออยู่จริง (**use-after-free**
— หนึ่งในบั๊กที่ร้ายแรงที่สุดในโปรแกรมมิ่งระดับต่ำ ที่ Rust สร้างมาเพื่อป้องกันตั้งแต่ต้น) หรือในทางกลับกันอาจ
ทำให้ข้อมูลไม่ถูกทำลายเลยแม้ไม่มีเจ้าของเหลือแล้ว (memory leak)

นี่คือเหตุผลที่ compiler **ปฏิเสธไม่ให้ `Rc<T>` ข้าม thread ได้เลย** ตั้งแต่ compile time — มันไม่ใช่ Rust
"เข้มงวดเกินไป" แต่เป็นการป้องกัน bug ประเภทที่ร้ายแรงและตรวจจับได้ยากที่สุดในโปรแกรมมิ่งแบบ concurrent ตั้งแต่
ก่อนโปรแกรมรันด้วยซ้ำ

#### Arc\<T\>: ฝาแฝดที่ใช้ Atomic Operations

คำตอบคือ `Arc<T>` (`std::sync::Arc`, ย่อมาจาก **A**tomically **R**eference **C**ounted) — API เหมือน `Rc<T>`
ทุกประการ (`Arc::new`, `Arc::clone`, `Arc::strong_count`, ไม่ implement `DerefMut` ด้วยเหตุผลเดียวกัน) แต่ต่าง
กันที่จุดเดียวซึ่งสำคัญที่สุด: **การเพิ่ม/ลดตัวนับใช้ atomic operation แทนการบวก/ลบเลขธรรมดา**

"Atomic operation" ในความหมายที่ใช้งานได้จริงตรงนี้ (ไม่ลงลึกทฤษฎี lock-free เต็มรูปแบบ ซึ่งเป็นเนื้อหาของ
Part 51) คือ: **คำสั่งระดับ CPU ที่รับประกันว่าจะทำงาน "ครบทั้งหมดในครั้งเดียวโดยไม่มีใครมาแทรกกลางทางได้"** —
ฮาร์ดแวร์ (CPU) มีคำสั่งพิเศษ (เช่น `fetch-and-add`, `compare-and-swap` บนสถาปัตยกรรมจริง) ที่ทำ "อ่าน-บวก-เขียน"
ทั้งสามขั้นตอนให้เสร็จเป็นหน่วยเดียวที่แบ่งแยกไม่ได้ (indivisible) จริง ๆ ที่ระดับฮาร์ดแวร์ — ถ้าสอง thread (หรือ
สอง CPU core) พยายามทำ atomic increment บนตัวแปรเดียวกันพร้อมกันเป๊ะ ๆ ฮาร์ดแวร์จะบังคับให้ทำสำเร็จทีละตัวเรียง
ต่อกันเสมอ (ไม่มีทางเกิดการอ่านค่าเก่าซ้อนกันได้) ผลลัพธ์สุดท้ายจึงถูกต้องเสมอไม่ว่า thread จะ schedule มาแบบไหน

จุดที่ควรรู้ (แบบผิวเผินพอสำหรับใช้งานจริง) คือ atomic operation **เร็วกว่า** การใช้ `Mutex` ล็อกทั้งก้อนเพื่อ
ป้องกัน race บนตัวแปรตัวเดียว เพราะมันทำงานที่ระดับคำสั่ง CPU ตัวเดียวโดยตรง ไม่ต้องมีการ "รอคิว" แบบ mutex เต็ม
รูปแบบ — นี่คือเหตุผลที่ `Arc<T>` ไม่ได้ใช้ `Mutex<usize>` มาห่อตัวนับของมันเอง (ซึ่งจะทำงานได้แต่ช้ากว่ามาก) แต่
ใช้ type พิเศษจาก `std::sync::atomic` (เช่น `AtomicUsize`) ที่คอมไพเลอร์และฮาร์ดแวร์ร่วมกันรับประกันความปลอดภัย
ให้โดยตรง

ผลคือ `Arc<T>` **implement `Send`** (และ `Sync` เมื่อ `T: Send + Sync` — รายละเอียดเต็มรออยู่ Part 40) ทำให้
มันย้ายข้าม thread ได้อย่างปลอดภัยจริง ๆ ตามที่ compiler รับประกัน — ลองแก้ตัวอย่างข้างบนด้วย `Arc<T>` แทน
`Rc<T>` ในหัวข้อถัดไปเลย

### 39.7 Arc\<Mutex\<T\>\>: คู่หูคลาสสิกของ Shared-State Concurrency

มาแก้ตัวอย่าง counter ให้ถูกต้องสมบูรณ์ด้วย `Arc<Mutex<T>>` — pattern ที่พบมากที่สุดในโค้ด Rust แบบ
multi-thread ที่ต้องการ shared mutable state:

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    const N_THREADS: usize = 10;
    const INCREMENTS_PER_THREAD: usize = 1000;

    for _ in 0..N_THREADS {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            for _ in 0..INCREMENTS_PER_THREAD {
                let mut num = counter.lock().unwrap();
                *num += 1;
            }
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    let expected = N_THREADS * INCREMENTS_PER_THREAD;
    let actual = *counter.lock().unwrap();
    println!("Result: {actual} (expected: {expected})");
    assert_eq!(actual, expected, "มี increment หายไป! เกิด data race");
    println!("ยืนยันแล้ว: ไม่มี increment หายไปเลยแม้แต่ครั้งเดียว");
}
```

ผลลัพธ์ (รันจริงซ้ำ 5 ครั้งติดกันเพื่อยืนยันว่าไม่ใช่ความบังเอิญ — ทุกครั้งได้ผลลัพธ์ตรงกัน 100%):

```
-- run 1 --
Result: 10000 (expected: 10000)
ยืนยันแล้ว: ไม่มี increment หายไปเลยแม้แต่ครั้งเดียว
-- run 2 --
Result: 10000 (expected: 10000)
ยืนยันแล้ว: ไม่มี increment หายไปเลยแม้แต่ครั้งเดียว
-- run 3 --
Result: 10000 (expected: 10000)
ยืนยันแล้ว: ไม่มี increment หายไปเลยแม้แต่ครั้งเดียว
-- run 4 --
Result: 10000 (expected: 10000)
ยืนยันแล้ว: ไม่มี increment หายไปเลยแม้แต่ครั้งเดียว
-- run 5 --
Result: 10000 (expected: 10000)
ยืนยันแล้ว: ไม่มี increment หายไปเลยแม้แต่ครั้งเดียว
```

ลองไล่ทีละส่วนว่าเกิดอะไรขึ้น:

1. **`Arc::new(Mutex::new(0))`** — สร้าง `Mutex<i32>` บน heap พร้อมห่อด้วย `Arc` ที่มีตัวนับเริ่มต้นที่ 1
2. **`Arc::clone(&counter)`** ในแต่ละรอบของ loop — เพิ่มตัวนับแบบ atomic (ปลอดภัย 100% แม้จะมีหลาย thread
   เรียกพร้อมกันในอนาคต) แล้ว copy ตัวชี้ไปให้แต่ละ thread ถือคนละตัว — **ทั้ง 10 ตัวชี้ที่ได้ชี้ไปยัง
   `Mutex<i32>` ก้อนเดียวกันเป๊ะ** เหมือนที่ `Rc::clone` ทำใน Part 28 เพียงแค่ปลอดภัยข้าม thread เท่านั้น
3. `move ||` — ownership ของ `Arc<Mutex<i32>>` ที่ clone มาถูก move เข้าไปใน closure ของแต่ละ thread จริง ๆ
   (ไม่ใช่ share reference ข้าม thread เพราะทำแบบนั้นไม่ได้ตามกฎ ownership) — แต่สิ่งที่ move เข้าไปคือ **ตัวชี้
   ของ `Arc`** ไม่ใช่ข้อมูล `Mutex<i32>` ตัวจริง ข้อมูลจริงยังอยู่ที่เดิมบน heap ก้อนเดียวและถูกแบ่งกันมองผ่าน
   ตัวชี้หลายตัว
4. ข้างใน closure ของแต่ละ thread: `counter.lock().unwrap()` — แข่งกันขอ lock กับ thread อื่นทั้ง 9 ตัวที่
   เหลือ (และตัวเอง ในรอบถัดไปของ loop ภายในของมันเอง) `Mutex<T>` การันตีว่า ณ ขณะใดขณะหนึ่งมีแค่ thread เดียว
   ที่ได้ `MutexGuard<i32>` ไปแก้ไข `*num += 1` เท่านั้น — thread อื่นที่มาขอพร้อมกันต้อง**รอเข้าคิว**
5. รวมทั้งหมด 10 thread × 1000 ครั้ง = 10,000 การเพิ่มค่า และผลลัพธ์สุดท้ายคือ **10,000 พอดีเป๊ะทุกครั้งที่รัน**
   — พิสูจน์ว่าไม่มี increment ไหนหายไปเลยแม้แต่ครั้งเดียว ทั้งที่มีการแข่งกันขอ lock อย่างหนักหน่วงจริง ๆ

เปรียบเทียบกับตัวอย่าง `Rc<RefCell<T>>`/`Rc<T>` จาก Part 28: `Arc::clone(&counter)` ในบทนี้ทำงาน**แนวคิดเดียวกัน
เป๊ะ**กับ `Rc::clone(&config)` ใน Part 28 — copy ตัวชี้ + เพิ่มตัวนับ ราคาถูก ไม่ deep copy ข้อมูลข้างใน — ต่าง
กันแค่ว่าตัวนับของ `Arc` เป็น **atomic** จึงปลอดภัยเมื่อหลาย thread เรียก `clone`/`drop` พร้อมกันจริง ๆ (ซึ่งเกิด
ขึ้นตลอดในตัวอย่างข้างบน เพราะทุก thread จบงานแล้ว drop `Arc` ของตัวเองในเวลาไล่เลี่ยกัน)

### 39.8 RwLock\<T\>: อ่านพร้อมกันได้ เขียนได้ทีละคน

`Mutex<T>` มีข้อจำกัดหนึ่งที่บางสถานการณ์ทำให้เสียประสิทธิภาพเกินจำเป็น: **มันไม่แยกแยะระหว่าง "อ่าน" กับ
"เขียน" เลย** — แม้ว่า thread สองตัวจะแค่ "อ่าน" ข้อมูลพร้อมกัน (ซึ่งไม่มีความเสี่ยง data race อะไรเลย เพราะไม่มี
ใครแก้ไข) `Mutex<T>` ก็ยังบังคับให้ทำทีละ thread อยู่ดี เพราะมันไม่รู้ว่า `.lock()` ครั้งนี้จะเอาไปอ่านหรือเขียน

`RwLock<T>` (`std::sync::RwLock`, "Read-Write Lock") แก้ปัญหานี้ตรง ๆ — เป็น "`RefCell<T>` ที่ฉลาดกว่า" ในเวอร์
ชันข้าม thread: มัน**แยก method** สำหรับขอสิทธิ์อ่าน (`.read()`) และขอสิทธิ์เขียน (`.write()`) โดยมีกฎที่คุ้น
เคยมาก (ตรงกับกฎข้อที่ 1 ของ borrowing จาก Part 7 เป๊ะ ๆ): **อนุญาตให้มี "ผู้อ่านพร้อมกันได้หลายคน" (`&T` หลาย
ตัว) หรือ "ผู้เขียนได้แค่คนเดียว" (`&mut T` ตัวเดียว) แต่ห้ามทั้งสองอย่างพร้อมกันเด็ดขาด**

```rust
use std::sync::{Arc, RwLock};
use std::thread;

fn main() {
    let data = Arc::new(RwLock::new(vec![1, 2, 3, 4, 5]));

    let mut handles = vec![];

    // สาม thread อ่านพร้อมกันได้
    for id in 0..3 {
        let data = Arc::clone(&data);
        handles.push(thread::spawn(move || {
            let values = data.read().unwrap();
            let sum: i32 = values.iter().sum();
            println!("[reader {id}] sum = {sum}");
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }

    // writer หนึ่งตัว แก้ไขข้อมูล
    {
        let mut values = data.write().unwrap();
        values.push(6);
        println!("[writer] เพิ่มค่าใหม่แล้ว: {:?}", *values);
    }

    let final_values = data.read().unwrap();
    println!("ค่าสุดท้าย: {:?}", *final_values);
}
```

ผลลัพธ์ (รันจริง — ลำดับการพิมพ์ของ reader 0/1/2 อาจสลับกันได้ในแต่ละครั้งที่รัน เพราะทั้งสามทำงานคู่ขนานกันจริง
และ scheduler ของ OS เป็นผู้กำหนดลำดับ ไม่ใช่โปรแกรม):

```
[reader 1] sum = 15
[reader 0] sum = 15
[reader 2] sum = 15
[writer] เพิ่มค่าใหม่แล้ว: [1, 2, 3, 4, 5, 6]
ค่าสุดท้าย: [1, 2, 3, 4, 5, 6]
```

`.read()` คืน `LockResult<RwLockReadGuard<T>>` ที่ implement `Deref` (อ่านได้อย่างเดียว เหมือน `Ref<T>` ของ
`RefCell` ใน Part 28) — หลาย thread เรียก `.read()` พร้อมกันได้โดยไม่ต้องรอกันเลย ตราบใดที่ไม่มีใครถือ
`.write()` guard อยู่ ส่วน `.write()` คืน `RwLockWriteGuard<T>` ที่ implement ทั้ง `Deref` และ `DerefMut`
(แก้ไขได้ เหมือน `RefMut<T>`) — และเหมือนกับ `MutexGuard<T>`, ทั้งสอง guard type นี้ทำงานแบบ RAII ปลดล็อก
อัตโนมัติเมื่อหลุด scope เช่นกัน

#### เมื่อไหร่ควรใช้ RwLock\<T\> แทน Mutex\<T\>

`RwLock<T>` ไม่ใช่ "ดีกว่า `Mutex<T>` เสมอ" — มันมี **overhead ภายในสูงกว่า** `Mutex<T>` เล็กน้อยเสมอ (ต้อง
เก็บบันทึกว่ามี reader กี่คนอยู่ ณ ขณะนั้น เพื่อรู้ว่าเมื่อไหร่จะปล่อยให้ writer เข้าได้) ดังนั้นการเลือกใช้ต้อง
พิจารณาลักษณะงานจริง:

| สถานการณ์ | ควรเลือก | เหตุผล |
|---|---|---|
| อ่านบ่อยกว่าเขียนมาก ๆ (เช่น cache, configuration ที่โหลดมาแล้วแทบไม่เปลี่ยน, ตาราง lookup) | **`RwLock<T>`** | reader หลายคนทำงานคู่ขนานกันได้จริง ไม่ต้องรอคิวทีละคนเหมือน `Mutex` — ได้ throughput สูงขึ้นมากในงานที่อ่านหนัก |
| เขียนบ่อยพอ ๆ กับอ่าน หรือเขียนบ่อยกว่า | **`Mutex<T>`** | ถ้าต้องเขียนบ่อยอยู่แล้ว การอนุญาตให้อ่านพร้อมกันได้ก็ไม่ช่วยลด contention เท่าไหร่ (writer ต้องรอ reader ทุกคนเคลียร์ก่อนอยู่ดี) ในขณะที่ overhead ของ `RwLock<T>` ก็ยังต้องจ่ายอยู่ทุกครั้ง — ใช้ `Mutex<T>` ที่เรียบง่ายและเบากว่าคุ้มค่ากว่า |
| Critical section สั้นมาก (แค่บวกเลข, เทียบค่า) | **`Mutex<T>`** | ความซับซ้อนที่เพิ่มขึ้นของ `RwLock<T>` ไม่คุ้มกับงานที่เบามาก ๆ ขนาดนี้ — เวลาที่ประหยัดได้จากอนุญาตให้อ่านพร้อมกันน้อยกว่า overhead ที่เสียไป |
| ไม่แน่ใจ / เขียน prototype | **`Mutex<T>`** | ง่ายกว่า เข้าใจง่ายกว่า มี method เดียว (`.lock()`) ไม่ต้องแยกคิดเรื่อง read/write — ค่อยเปลี่ยนเป็น `RwLock<T>` ทีหลังถ้า profiling พบว่า read-contention เป็นคอขวดจริง |

กฎทั่วไปที่จำง่าย: **`Mutex<T>` คือค่าเริ่มต้นที่ควรเลือกก่อน** เพราะเรียบง่ายกว่าและพอเพียงสำหรับงานส่วนใหญ่ —
สลับไปใช้ `RwLock<T>` เมื่อวัดจริง (profiling) แล้วพบว่า read-heavy workload กำลังเป็นคอขวดจากการที่ reader
หลายคนต้องรอคิวกันทั้งที่ไม่มีใครแก้ไขข้อมูลจริง ๆ

### 39.9 Deadlock: เมื่อสอง Thread รอกันไปตลอดกาล

ทุกเครื่องมือที่เรียนมาในบทนี้ป้องกัน **data race** ได้อย่างสมบูรณ์ (compiler การันตีเลย) แต่มีปัญหาอีกประเภท
หนึ่งที่ **compiler ตรวจจับให้ไม่ได้เลย** และ Rust ก็ไม่มีข้อยกเว้น: **deadlock** (ติดตาย) — สถานการณ์ที่สอง
thread (หรือมากกว่า) ต่าง**รอ lock ที่อีกฝ่ายถืออยู่**พร้อมกัน โดยไม่มีใครยอมปล่อยก่อน ผลคือทั้งสอง thread
**ค้างตลอดไป** ไม่มีทางหลุดออกมาได้เองเลย

จุดที่ทำให้ deadlock อันตรายเป็นพิเศษคือ **มันไม่ทำให้ compile error และไม่ panic เลย** — โปรแกรมแค่ค้างเงียบ ๆ
ไม่มี error message ให้อ่าน ไม่มี stack trace ให้ดู วิธีสังเกตได้อย่างเดียวคือโปรแกรมไม่ยอมจบสักที (CPU usage
อาจต่ำมากด้วยเพราะ thread ที่ค้างรอ lock มักถูก OS พักไว้ ไม่ได้วนลูปกินซีพียู) — วิธีวินิจฉัยในทางปฏิบัติจริง
คือรันโปรแกรมพร้อมจับเวลา (เช่นด้วยคำสั่ง `timeout` ใน Linux/macOS) ถ้าโปรแกรมไม่จบภายในเวลาที่คาดว่าควรจะจบ
ก็เป็นสัญญาณเตือนแรกว่าอาจ deadlock — เครื่องมือ debugger (เช่น `gdb`) ก็ใช้ดูได้ว่า ณ ขณะที่ค้าง แต่ละ thread
กำลัง block อยู่ที่ instruction ไหน (ถ้าเห็นหลาย thread ค้างอยู่ที่ `.lock()` พร้อมกันตลอด คือสัญญาณ deadlock
ที่ชัดเจน)

มาดูตัวอย่าง deadlock แบบคลาสสิกที่สุด — สอง mutex, สอง thread, ล็อกในลำดับ**สลับกัน**:

```rust
use std::sync::{Arc, Mutex};
use std::thread;
use std::time::Duration;

fn main() {
    let resource_a = Arc::new(Mutex::new("resource A"));
    let resource_b = Arc::new(Mutex::new("resource B"));

    let a1 = Arc::clone(&resource_a);
    let b1 = Arc::clone(&resource_b);
    let thread1 = thread::spawn(move || {
        println!("[thread 1] กำลังล็อก A...");
        let _lock_a = a1.lock().unwrap();
        println!("[thread 1] ล็อก A ได้แล้ว, รอ 100ms แล้วจะล็อก B ต่อ");
        thread::sleep(Duration::from_millis(100));

        println!("[thread 1] กำลังล็อก B...");
        let _lock_b = b1.lock().unwrap(); // ค้างตรงนี้ตลอดไป
        println!("[thread 1] ล็อก B ได้แล้ว (ไม่ควรพิมพ์ถึงตรงนี้)");
    });

    let a2 = Arc::clone(&resource_a);
    let b2 = Arc::clone(&resource_b);
    let thread2 = thread::spawn(move || {
        println!("[thread 2] กำลังล็อก B...");
        let _lock_b = b2.lock().unwrap();
        println!("[thread 2] ล็อก B ได้แล้ว, รอ 100ms แล้วจะล็อก A ต่อ");
        thread::sleep(Duration::from_millis(100));

        println!("[thread 2] กำลังล็อก A...");
        let _lock_a = a2.lock().unwrap(); // ค้างตรงนี้ตลอดไป
        println!("[thread 2] ล็อก A ได้แล้ว (ไม่ควรพิมพ์ถึงตรงนี้)");
    });

    thread1.join().unwrap();
    thread2.join().unwrap();
    println!("โปรแกรมจบแล้ว (ไม่ควรมาถึงตรงนี้เลยถ้าเกิด deadlock จริง)");
}
```

ผลตอนรันจริง (ผู้เขียนรันโค้ดนี้ด้วยคำสั่ง `timeout 3 ./program` เพื่อบังคับให้ process ถูก kill ถ้ายังไม่จบ
ภายใน 3 วินาที — เป็นวิธีวินิจฉัย deadlock ที่ปลอดภัย ไม่ทำให้ terminal ค้างตลอดไปขณะตรวจสอบ):

```
[thread 1] กำลังล็อก A...
[thread 1] ล็อก A ได้แล้ว, รอ 100ms แล้วจะล็อก B ต่อ
[thread 2] กำลังล็อก B...
[thread 2] ล็อก B ได้แล้ว, รอ 100ms แล้วจะล็อก A ต่อ
[thread 1] กำลังล็อก B...
[thread 2] กำลังล็อก A...
```

(หลังจากนั้นโปรแกรม**ไม่พิมพ์อะไรเพิ่มอีกเลย** และ `timeout` สั่ง kill process หลัง 3 วินาที — exit code ที่ได้
คือ **124** ซึ่งเป็นรหัสมาตรฐานที่คำสั่ง `timeout` ใช้บอกว่า "process ถูกฆ่าเพราะหมดเวลา" นี่คือหลักฐานที่ยืนยัน
ได้จริงว่าโปรแกรมค้างจริง ไม่ใช่แค่ทำงานช้า — บรรทัด "ล็อก B ได้แล้ว" ของ thread 1 และ "ล็อก A ได้แล้ว" ของ
thread 2 ไม่ปรากฏเลย รวมถึงบรรทัด "โปรแกรมจบแล้ว" ก็ไม่ปรากฏเช่นกัน พิสูจน์ว่าทั้งสอง thread ค้างอยู่ที่
`.lock()` ตลอดไปจริง ๆ)

มาไล่ดูว่าทำไมถึงเกิด deadlock:

1. `thread1` ล็อก `resource_a` ได้สำเร็จก่อน (ตามลำดับที่มันเขียน: A ก่อน B)
2. `thread2` ล็อก `resource_b` ได้สำเร็จก่อนเช่นกัน (ตามลำดับที่**มันเขียน**: B ก่อน A — สลับกับ `thread1`)
3. ระหว่างที่ทั้งสอง sleep 100ms (จำลองว่ากำลังทำงานอะไรบางอย่างอยู่) ทั้งสองต่างถือ lock ของตัวเองไว้อยู่แล้ว
4. เมื่อครบ 100ms `thread1` พยายามล็อก `resource_b` ต่อ — แต่ `resource_b` ถูก `thread2` ถือไว้อยู่ (และ
   `thread2` ยังไม่ยอมปล่อยจนกว่าจะได้ `resource_a` ก่อน) → `thread1` **ต้องรอ**
5. ในเวลาเดียวกัน `thread2` พยายามล็อก `resource_a` ต่อ — แต่ `resource_a` ถูก `thread1` ถือไว้อยู่ (และ
   `thread1` ยังไม่ยอมปล่อยจนกว่าจะได้ `resource_b` ก่อน) → `thread2` **ต้องรอ**
6. ผลคือ **`thread1` รอ `thread2` และ `thread2` รอ `thread1` พร้อมกันเป๊ะ ๆ** — วงจรปิดของการรอ (circular
   wait) ที่ไม่มีทางคลายออกได้เองเลย เพราะไม่มีใครยอมปล่อย lock ของตัวเองก่อนที่จะได้ lock อีกตัวที่ต้องการ

นี่คือ **นิยามคลาสสิกของ deadlock**: มีเงื่อนไข 4 อย่างเกิดขึ้นพร้อมกัน (Coffman conditions ที่รู้จักกันในวิชา
ระบบปฏิบัติการ) คือ (1) mutual exclusion — resource ถูกถือแบบ exclusive ได้ทีละคน (2) hold and wait — thread
ถือ resource หนึ่งไว้พร้อมรออีกตัว (3) no preemption — ไม่มีใครมาแย่ง lock จาก thread ที่ถืออยู่ได้ (4) circular
wait — มีวงจรปิดของการรอ — ทั้ง 4 ข้อนี้เกิดพร้อมกันในตัวอย่างข้างบนพอดี

#### วิธีป้องกัน: ล็อกตามลำดับเดียวกันเสมอทั้งระบบ

วิธีป้องกัน deadlock ที่ใช้กันมากที่สุดและเข้าใจง่ายที่สุดคือ **กำหนดลำดับการล็อกให้เป็นมาตรฐานเดียวกันทั้ง
ระบบ แล้วบังคับให้ทุก thread ล็อกตามลำดับนั้นเสมอ ไม่มีข้อยกเว้น** — ในตัวอย่างข้างบน ถ้าเราแก้ให้ `thread2`
ล็อก A ก่อน B เหมือน `thread1` (ไม่สลับลำดับกัน) deadlock จะหายไปทันที:

```rust
use std::sync::{Arc, Mutex};
use std::thread;
use std::time::Duration;

fn main() {
    let resource_a = Arc::new(Mutex::new("resource A"));
    let resource_b = Arc::new(Mutex::new("resource B"));

    // ทั้งสอง thread ล็อก A ก่อน B เสมอ (ลำดับเดียวกันทั้งระบบ) -> ไม่มี deadlock
    let a1 = Arc::clone(&resource_a);
    let b1 = Arc::clone(&resource_b);
    let thread1 = thread::spawn(move || {
        let _lock_a = a1.lock().unwrap();
        thread::sleep(Duration::from_millis(50));
        let _lock_b = b1.lock().unwrap();
        println!("[thread 1] ได้ทั้งสอง lock แล้ว");
    });

    let a2 = Arc::clone(&resource_a);
    let b2 = Arc::clone(&resource_b);
    let thread2 = thread::spawn(move || {
        let _lock_a = a2.lock().unwrap();
        thread::sleep(Duration::from_millis(50));
        let _lock_b = b2.lock().unwrap();
        println!("[thread 2] ได้ทั้งสอง lock แล้ว");
    });

    thread1.join().unwrap();
    thread2.join().unwrap();
    println!("โปรแกรมจบแบบปกติ ไม่มี deadlock");
}
```

ผลลัพธ์ (รันจริง — จบได้ปกติทันที ไม่ค้าง):

```
[thread 1] ได้ทั้งสอง lock แล้ว
[thread 2] ได้ทั้งสอง lock แล้ว
โปรแกรมจบแบบปกติ ไม่มี deadlock
```

ทำไมถึงไม่ deadlock แล้ว: ตอนนี้ **ไม่มีทางเกิด circular wait ได้เลย** — สมมติ `thread1` ได้ `resource_a` ไปก่อน
`thread2` ก็ต้องรอ `resource_a` อยู่เฉย ๆ (ยังไม่ได้ถือ `resource_b` เลย) พอ `thread1` ได้ `resource_b` ตามมา
ทำงานเสร็จ ปล่อยทั้งสอง lock แล้ว `thread2` ก็ไล่ตามได้ทั้งสอง lock ตามลำดับได้อย่างราบรื่น — ไม่มีทางที่ทั้งสอง
thread จะ "ถือคนละตัว แล้วรออีกตัวที่อีกฝ่ายถือ" ได้อีกต่อไป เพราะทุกคนพยายามล็อก `resource_a` **ก่อนเสมอ** ไม่
มีใครข้ามไปล็อก `resource_b` ก่อนได้เลย

กฎปฏิบัติจริงที่ใช้กันในระบบขนาดใหญ่คือ: **กำหนดลำดับ (เช่นตาม ID, ตามชื่อตัวแปรเรียงตามตัวอักษร, หรือตาม
ลำดับที่ประกาศในโค้ด) ให้กับทุก lock ที่อาจต้องถูกล็อกพร้อมกันในระบบ แล้ว document ลำดับนั้นให้ทุกคนในทีมรู้
และตรวจสอบผ่าน code review อย่างเคร่งครัด** — Rust ไม่มีกลไกใน type system ที่บังคับเรื่องนี้ให้อัตโนมัติ (ต่าง
จาก data race ที่ compiler ป้องกันให้เต็มรูปแบบ) ดังนั้น **การป้องกัน deadlock ยังคงเป็นความรับผิดชอบของผู้
ออกแบบระบบเสมอ** ไม่ว่าจะเขียนด้วยภาษาไหนก็ตาม — นี่คือหนึ่งในไม่กี่จุดที่ Rust "ไม่ช่วย" คุณโดยอัตโนมัติ
(อีกจุดหนึ่งที่คล้ายกันคือ reference cycle จาก `Rc<RefCell<T>>` ใน Part 28 ที่ทำให้ memory leak แบบ safe ได้)

### 39.10 ตารางเปรียบเทียบเต็มรูปแบบ: Rc\<RefCell\<T\>\> vs Arc\<Mutex\<T\>\>

ถึงจุดนี้เราเห็นภาพครบทั้งสองฝั่งแล้ว มาสรุปเป็นตารางเทียบตรงเพื่อให้เห็นภาพรวมทั้งหมดในที่เดียว — สังเกตว่า
**รูปทรงเชิงแนวคิด (conceptual shape) เหมือนกันทุกประการ** ต่างกันแค่ **กลไกที่อยู่เบื้องหลัง** และ **ต้นทุน**
ที่ต้องจ่าย:

| มิติเปรียบเทียบ | `Rc<RefCell<T>>` (Part 28) | `Arc<Mutex<T>>` (บทนี้) |
|---|---|---|
| ใช้ได้ในสถานการณ์ | Single-thread เท่านั้น | Multi-thread ได้อย่างปลอดภัย |
| ส่วนที่ให้ "เจ้าของร่วมหลายคน" | `Rc<T>` | `Arc<T>` |
| กลไกของตัวนับเจ้าของ | ตัวเลขธรรมดา (`Cell<usize>` ภายใน) เพิ่ม/ลดตรง ๆ ไม่มี synchronize | ตัวเลขแบบ **atomic** (`AtomicUsize` ภายใน) เพิ่ม/ลดด้วยคำสั่ง CPU ที่ไม่ถูกแทรกกลางทางได้ |
| ส่วนที่ให้ "แก้ไขข้อมูลผ่าน shared reference" | `RefCell<T>` | `Mutex<T>` (หรือ `RwLock<T>` ถ้าต้องแยก read/write) |
| กลไกตรวจกฎ "mutable หนึ่ง หรือ immutable หลาย" | ตัวนับ borrow ภายใน ตรวจตอน runtime, **ไม่ thread-safe** | Lock ระดับ OS/lightweight ที่ synchronize ข้าม thread ได้จริง |
| ขอสิทธิ์เขียน | `.borrow_mut()` → `RefMut<T>` | `.lock()` → (ผ่าน `Result`) `MutexGuard<T>` |
| ถ้าละเมิดกฎ (ขอสิทธิ์ซ้อนกัน) | **panic ทันที** (`BorrowError`/`BorrowMutError`) เพราะไม่มีใครให้รอในเธรดเดียว | **thread อื่นรอ (block)** จนกว่า lock จะถูกปลด (ยกเว้นล็อกซ้อนในเธรดเดียวกันเอง ซึ่งจะ deadlock ตัวเอง — ดูกับดักข้อ 3) |
| ปลดล็อก/คืนสิทธิ์ | หลุด scope (RAII/`Drop`) อัตโนมัติ | หลุด scope (RAII/`Drop`) อัตโนมัติ — **เหมือนกันทุกประการ** |
| ต้นทุนของการ clone (`Rc::clone`/`Arc::clone`) | ถูกมาก (เพิ่มเลขธรรมดา) | ถูก แต่ **แพงกว่า `Rc::clone` เล็กน้อย** เพราะ atomic instruction มี overhead มากกว่าการบวกเลขธรรมดาที่ระดับ CPU (ยังคง O(1) และเร็วกว่าการ deep clone ข้อมูลมากอยู่ดี) |
| ต้นทุนของการขอสิทธิ์เขียน | ถูกมาก (แค่เช็ค/ปรับตัวนับ) | **แพงกว่า `.borrow_mut()` เล็กน้อย** เพราะอาจต้องเรียก syscall ของ OS หรือทำ atomic spin ถ้ามี contention จริง |
| ป้องกัน reference cycle (memory leak) ได้ไหม | ไม่ป้องกัน (ต้องใช้ `Weak<T>` จาก Part 29 เอง) | ไม่ป้องกันเช่นกัน (ต้องใช้ `Arc::downgrade`/`Weak<T>` เวอร์ชัน thread-safe เอง) |
| ป้องกัน deadlock ได้ไหม | ไม่เกี่ยวข้อง (single-thread ไม่มี deadlock แบบข้าม thread) | **ไม่ป้องกัน** — ผู้เขียนโค้ดต้องออกแบบลำดับการล็อกให้ถูกต้องเอง (หัวข้อ 39.9) |
| implement `Send`/`Sync` หรือไม่ | **ไม่** (ทั้ง `Rc<T>` และ `RefCell<T>`) | **ใช่** (เมื่อ `T: Send`) — รายละเอียดเต็มใน Part 40 |

บทเรียนที่เป็นรูปธรรมที่สุดจากตารางนี้คือ: **จ่ายต้นทุนของ `Arc`/`Mutex` เฉพาะตอนที่ต้องแบ่งข้อมูลกันข้าม
thread จริง ๆ เท่านั้น** — ถ้าโปรแกรม (หรือแม้แต่แค่บางส่วนของโปรแกรม) ทำงานใน thread เดียว ให้ใช้
`Rc<RefCell<T>>` ต่อไปตามที่เรียนใน Part 28 เพราะมันเบากว่าและง่ายกว่าอย่างมีนัยสำคัญ — การใช้ `Arc<Mutex<T>>`
ในโค้ดที่ไม่มีการแบ่งข้าม thread เลยเป็นการจ่ายต้นทุน atomic operation และ lock overhead โดยไม่ได้ประโยชน์
อะไรกลับมาเลย เป็นตัวอย่างของการ "premature pessimization" (ทำให้โค้ดช้าลงโดยไม่จำเป็นก่อนที่จะรู้ด้วยซ้ำว่า
ต้องการความปลอดภัยข้าม thread จริงหรือไม่)

ในทางกลับกัน ถ้าคุณรู้แน่ชัดตั้งแต่ต้นว่าข้อมูลนี้ **ต้อง** ถูกแบ่งกันใช้จากหลาย thread จริง (เช่นตัวอย่างระบบ
ธนาคารในหัวข้อถัดไป) `Arc<Mutex<T>>` คือคำตอบที่ถูกต้องและปลอดภัยที่สุด — Rust บังคับให้คุณเลือกเครื่องมือที่
ตรงกับความต้องการจริงตั้งแต่ compile time ผ่าน error `E0277` ที่เห็นในหัวข้อ 39.6 พอดี ไม่ปล่อยให้คุณ "ลืม"
ใส่การป้องกันที่จำเป็นได้เลย

### 39.11 ตัวอย่างโลกจริง: ระบบ ATM จำลองด้วย Arc\<Mutex\<HashMap\<String, f64\>\>\>

มาปิดท้ายเนื้อหาด้วยตัวอย่างที่สมบูรณ์และใกล้เคียงงานจริงมากที่สุด: ระบบธนาคารที่มี **หลาย ATM (แทนด้วยหลาย
thread) เข้าถึงบัญชีลูกค้าที่แบ่งกันใช้จริง** (`HashMap<String, f64>` จาก Part 15 ที่เก็บชื่อบัญชีคู่กับยอดเงิน)
พร้อมกันหลายรายการธุรกรรม — เราจะพิสูจน์ด้วยการรันจริงว่า **ยอดเงินรวมทั้งระบบไม่เปลี่ยนแปลงเลย** (ไม่มีเงินหาย
ไม่มีเงินถูกสร้างขึ้นมาจากอากาศ) แม้จะมีการแข่งกันเข้าถึงบัญชีเดียวกันจากหลาย thread อย่างหนักหน่วงก็ตาม —
นี่คือการพิสูจน์ว่าไม่มี update ใดสูญหายไปเลยจาก race condition:

```rust
use std::collections::HashMap;
use std::sync::{Arc, Mutex};
use std::thread;

#[derive(Debug)]
enum TxError {
    AccountNotFound,
    InsufficientFunds,
}

fn withdraw(
    accounts: &Arc<Mutex<HashMap<String, f64>>>,
    account: &str,
    amount: f64,
) -> Result<(), TxError> {
    let mut map = accounts.lock().unwrap();
    let balance = map.get_mut(account).ok_or(TxError::AccountNotFound)?;
    if *balance < amount {
        return Err(TxError::InsufficientFunds);
    }
    *balance -= amount;
    Ok(())
}

fn deposit(accounts: &Arc<Mutex<HashMap<String, f64>>>, account: &str, amount: f64) {
    let mut map = accounts.lock().unwrap();
    if let Some(balance) = map.get_mut(account) {
        *balance += amount;
    }
}

fn total_balance(accounts: &Arc<Mutex<HashMap<String, f64>>>) -> f64 {
    let map = accounts.lock().unwrap();
    map.values().sum()
}

fn main() {
    let mut initial = HashMap::new();
    initial.insert("alice".to_string(), 1000.0);
    initial.insert("bob".to_string(), 1000.0);
    initial.insert("charlie".to_string(), 1000.0);

    let accounts = Arc::new(Mutex::new(initial));
    let starting_total = total_balance(&accounts);
    println!("ยอดรวมตั้งต้น: {starting_total}");

    let mut handles = vec![];

    // จำลอง ATM 6 เครื่อง ทำธุรกรรมพร้อมกัน 200 ครั้งต่อเครื่อง
    // ทุกธุรกรรมคือการโยกเงินระหว่างบัญชี -> ยอดรวมทั้งระบบต้องไม่เปลี่ยน
    let transfers = [
        ("alice", "bob", 10.0),
        ("bob", "charlie", 5.0),
        ("charlie", "alice", 7.0),
        ("bob", "alice", 3.0),
    ];

    for atm_id in 0..6 {
        let accounts = Arc::clone(&accounts);
        let handle = thread::spawn(move || {
            for i in 0..200 {
                let (from, to, amount) = transfers[(atm_id + i) % transfers.len()];
                // ถอนจากบัญชีต้นทาง แล้วฝากเข้าบัญชีปลายทาง
                // ถ้าถอนไม่สำเร็จ (เงินไม่พอ) ก็แค่ข้ามรอบนี้ไป ไม่ฝากด้วย
                if withdraw(&accounts, from, amount).is_ok() {
                    deposit(&accounts, to, amount);
                }
            }
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    let ending_total = total_balance(&accounts);
    println!("ยอดรวมสุดท้าย: {ending_total}");
    assert_eq!(
        starting_total, ending_total,
        "ยอดรวมเปลี่ยนไป! แสดงว่ามี transaction หายหรือถูกนับซ้ำ"
    );
    println!("ยืนยันแล้ว: ยอดรวมเงินในระบบเท่าเดิมทุกบาท ไม่มี update สูญหาย");

    let map = accounts.lock().unwrap();
    let mut names: Vec<&String> = map.keys().collect();
    names.sort();
    for name in names {
        println!("  {name}: {:.2}", map[name]);
    }
}
```

ผลลัพธ์ (รันจริง):

```
ยอดรวมตั้งต้น: 3000
ยอดรวมสุดท้าย: 3000
ยืนยันแล้ว: ยอดรวมเงินในระบบเท่าเดิมทุกบาท ไม่มี update สูญหาย
  alice: 1000.00
  bob: 1600.00
  charlie: 400.00
```

จุดที่ต้องสังเกตให้ชัดหลายจุด:

1. **`withdraw` และ `deposit` แต่ละครั้งขอ `.lock()` แยกกัน** ไม่ใช่ขอ lock ครั้งเดียวแล้วทำทั้งสองอย่างในครั้ง
   เดียว — นี่คือการออกแบบที่ตั้งใจให้ **critical section (ช่วงที่ถือ lock) สั้นที่สุดในแต่ละครั้ง** ตามคำแนะนำ
   ในหัวข้อ 39.4 แต่ก็มีข้อแลกเปลี่ยนที่ต้องรู้: ระหว่างที่ `withdraw` จบไปแล้วแต่ `deposit` ยังไม่เริ่ม มี "ช่วง
   เวลาสั้น ๆ" ที่เงินถูกหักออกจากบัญชีต้นทางไปแล้วแต่ยังไม่เข้าบัญชีปลายทาง — ถ้ามีการอ่านยอดรวม (`total_balance`)
   เข้ามาแทรกกลาง**พอดี**ในช่วงนั้น อาจเห็นยอดรวมที่ดู "หายไปชั่วขณะ" ได้ (แม้จะกลับมาถูกต้องเสมอในระยะยาว) — ใน
   ระบบธนาคารจริงที่ต้องการความถูกต้องแบบ **atomic ข้ามหลายบัญชีในธุรกรรมเดียว** จะต้องออกแบบให้ทั้ง
   withdraw+deposit อยู่ใน critical section เดียวกัน (ล็อกครั้งเดียวคลุมทั้งสองการกระทำ) แต่ตัวอย่างนี้จงใจแยก
   ไว้เพื่อแสดงให้เห็นว่า **ต่อให้แยก ผลลัพธ์สุดท้ายก็ยังถูกต้อง 100%** เพราะทุก field ของ `HashMap` ที่ถูกแก้ไข
   ยังอยู่ภายใต้ `Mutex` ตัวเดียวกันเสมอ ไม่มีทางเกิด lost update ได้เลยไม่ว่า thread จะสลับกันมาแทรกตอนไหน
2. **การใช้ `Result<(), TxError>` ใน `withdraw`** (Part 12) ทำให้ธุรกรรมที่เงินไม่พอ **ไม่ทำให้โปรแกรม panic**
   แต่คืน `Err` ให้ผู้เรียกตัดสินใจ (ในที่นี้แค่ข้ามธุรกรรมนั้นไปเงียบ ๆ ด้วย `if ... .is_ok()`) — สะท้อนสถานการณ์
   จริงที่ธุรกรรมล้มเหลวได้เป็นปกติ ไม่ใช่ข้อผิดพลาดร้ายแรงของระบบ
3. `total_balance` ก็ต้อง `.lock()` เช่นกัน (แม้จะแค่อ่าน) เพราะ `Mutex<T>` ไม่แยก read/write — ถ้าต้องการให้
   การอ่านยอดรวมไม่ block ธุรกรรมอื่นบ่อยเกินไปในระบบที่มีการอ่านสถิติถี่มาก ๆ (เช่น dashboard ที่ refresh
   ทุกวินาที) นี่คือสถานการณ์ตรงแบบที่ `RwLock<T>` จากหัวข้อ 39.8 น่าเอามาแทนที่ `Mutex<T>` ได้พอดี
4. ที่สำคัญที่สุด: **`assert_eq!(starting_total, ending_total, ...)` ผ่านทุกครั้งที่รัน** ทั้งที่มี 6 thread
   แย่งกันแก้ไข `HashMap` เดียวกันจริง ๆ ผ่านการโยกเงินหลายทิศทางรวม 1,200 ธุรกรรม — นี่คือข้อพิสูจน์ที่เป็น
   รูปธรรมที่สุดของบทนี้ว่า `Arc<Mutex<T>>` ป้องกัน data race ได้จริงในสถานการณ์ที่ใกล้เคียงกับงานจริงมากกว่า
   ตัวอย่าง counter เพียว ๆ ในหัวข้อ 39.7

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. พยายาม move `Mutex<T>` ตรง ๆ เข้าหลาย thread — `E0382`

```
error[E0382]: borrow of moved value: `counter`
```

เกิดขึ้นเมื่อพยายาม `move` ตัวแปร `Mutex<T>` เดิม (ไม่ผ่าน `Arc`) เข้า `thread::spawn` ในหลายรอบของ loop ตามที่
เห็นในหัวข้อ 39.5 — ตัวแปรถูกย้าย ownership ไปให้ thread แรกไปแล้ว รอบต่อไปจึงใช้ไม่ได้อีก **วิธีแก้**: ห่อ
`Mutex<T>` ด้วย `Arc<T>` เสมอเมื่อต้องแบ่งให้หลาย thread ถือร่วมกัน แล้ว `Arc::clone(&x)` ก่อนส่งเข้า closure
ของแต่ละ thread — `Arc::clone` เพิ่มตัวนับ ไม่ move ตัวข้อมูลจริง จึงทำให้ทุก thread มี "ตัวชี้ของตัวเอง" ที่
ชี้ไปยัง `Mutex<T>` ก้อนเดียวกันได้โดยไม่ละเมิดกฎ ownership เลย

### 2. ใช้ `Rc<Mutex<T>>` แทน `Arc<Mutex<T>>` — `E0277`

```
error[E0277]: `Rc<std::sync::Mutex<i32>>` cannot be sent between threads safely
= help: within `{closure@...}`, the trait `Send` is not implemented for `Rc<std::sync::Mutex<i32>>`
```

นี่คือความสับสนที่พบบ่อยที่สุดของคนที่มาจาก Part 28 ใหม่ ๆ — `Rc<T>` และ `Arc<T>` มี API เหมือนกันเป๊ะจนสลับกัน
ใช้ผิดได้ง่ายมาก แต่ **`Rc<T>` (ตัวนับธรรมดา ไม่ synchronize) ใช้ข้าม thread ไม่ได้เลยไม่ว่ากรณีใด** ตามที่
อธิบายละเอียดในหัวข้อ 39.6 (ตัวนับของมันเองก็เป็น shared mutable state ที่ไม่ thread-safe) **วิธีแก้**: เปลี่ยน
`use std::rc::Rc;` เป็น `use std::sync::Arc;` และเปลี่ยนทุกจุดที่เรียก `Rc::new`/`Rc::clone` เป็น
`Arc::new`/`Arc::clone` — กฎง่าย ๆ ที่จำได้เสมอ: **ถ้าโค้ดมีการข้าม thread (`thread::spawn`, `async` task ที่
รันบน thread pool ฯลฯ) เกี่ยวข้องเลยแม้แต่นิดเดียว ให้ใช้ `Arc` ไม่ใช่ `Rc`** — ไม่มีข้อยกเว้น

### 3. Self-deadlock: ล็อก `Mutex` ตัวเดียวกันซ้ำในเธรดเดียวกัน

```rust
use std::sync::Mutex;

fn main() {
    let m = Mutex::new(5);

    let _guard1 = m.lock().unwrap();
    println!("ได้ lock ครั้งแรกแล้ว กำลังจะขอ lock ตัวเดิมซ้ำในเธรดเดียวกัน...");

    // เธรดเดียวกันพยายามล็อก Mutex ตัวเดิมเป็นครั้งที่สอง ทั้งที่ _guard1 ยังไม่หลุด scope
    // std::sync::Mutex ไม่ใช่ re-entrant lock -> ค้างตรงนี้ตลอดไป
    let _guard2 = m.lock().unwrap();
    println!("ได้ lock ครั้งที่สองแล้ว (ไม่ควรพิมพ์ถึงตรงนี้เลย)");
}
```

รันจริงด้วย `timeout 3 ./program` แล้วได้ผลว่าโปรแกรม**ค้างและถูก kill หลัง 3 วินาที** (exit code 124) —
พิมพ์ได้แค่บรรทัดแรกเท่านั้น บรรทัด "ได้ lock ครั้งที่สองแล้ว" ไม่ปรากฏเลย เพราะ `std::sync::Mutex` ของ Rust
**ไม่ใช่ re-entrant lock** (ไม่เหมือน mutex บางแบบในบางภาษา/ไลบรารีที่อนุญาตให้เธรดเดียวกันล็อกซ้อนตัวเองได้)
— การเรียก `.lock()` ครั้งที่สองในเธรดเดียวกันขณะที่ `_guard1` จากครั้งแรกยังไม่หลุด scope จะทำให้เธรดรอ lock
ที่ **ตัวเองถืออยู่** ซึ่งไม่มีทางถูกปลดได้เลยเพราะไม่มีใคร (รวมถึงตัวเองในโค้ดบรรทัดถัดไป) จะไปเรียก
`drop(_guard1)` ได้ก่อนที่ `.lock()` ตัวที่สองจะได้ผลลัพธ์กลับมา — เป็น deadlock ที่ง่ายที่สุดที่จะพลาดโดยไม่
ตั้งใจ โดยเฉพาะเมื่อเรียกฟังก์ชันสองตัวที่ต่างก็ `.lock()` `Mutex` ตัวเดียวกันจากภายในฟังก์ชันอีกตัวที่ถือ lock
อยู่แล้ว (เช่น method หนึ่งเรียก method อีกตัวของ struct เดียวกันที่ต่างก็ล็อก field เดียวกัน) **วิธีแก้**: จำกัด
scope ของ guard ให้ปลดล็อกก่อนเรียกฟังก์ชัน/method อื่นที่อาจล็อก `Mutex` ตัวเดียวกันอีกครั้งเสมอ และหลีกเลี่ยง
การออกแบบ API ที่ทำให้ผู้เรียกไม่รู้ว่าฟังก์ชันหนึ่งกำลังล็อกอะไรอยู่บ้างภายใน

### 4. เผลอเรียก `.lock().unwrap()` บน `Mutex` ที่ถูก poison แล้วโดยไม่รู้ตัว

```
thread 'main' panicked at src/main.rs:15:30:
called `Result::unwrap()` on an `Err` value: PoisonError { .. }
```

เมื่อ thread หนึ่ง panic ขณะถือ lock (หัวข้อ 39.3) `Mutex` จะถูก poison — ทุก `.lock()` ที่ตามมาหลังจากนั้น
(จาก thread ไหนก็ตาม รวมถึง thread `main`) จะได้ `Err` กลับมา ถ้าโค้ดเขียน `.lock().unwrap()` เป็นนิสัยไว้ทุก
จุด (ซึ่งเป็นแนวปฏิบัติที่พบบ่อยและสมเหตุสมผลตามที่อธิบายในหัวข้อ 39.3) การ panic นี้จะ **"ลาม" ไปยัง thread
อื่น ๆ ที่พยายามใช้ `Mutex` ตัวเดียวกันต่อทั้งหมด** ทั้งที่จริง ๆ เธรดเหล่านั้นไม่ได้ทำอะไรผิดเลย — บั๊กต้นตอ
จริงเกิดจาก thread แรกที่ panic เท่านั้น แต่ผลกระทบลามไปทั่วระบบ **วิธีแก้**: ถ้าการทำงานของระบบไม่ควรหยุดสนิท
เพียงเพราะ thread หนึ่ง panic (เช่น worker pool ที่ควรทำงานต่อได้แม้ worker ตัวหนึ่งพัง) ให้จัดการ `Err` จาก
`.lock()` อย่างชัดเจนแทนการ `.unwrap()` เสมอไป (เช่นใช้ `poison_error.into_inner()` เพื่อดึงข้อมูลออกมาต่อ หรือ
log แล้ว skip งานนั้นไป) และพิจารณาออกแบบให้ panic ที่อาจเกิดขึ้นได้ (เช่นจาก input ที่ผิดรูปแบบ) ถูกจับด้วย
`Result`/`catch_unwind` **ก่อน** ที่จะไปแตะ `Mutex` แทนที่จะปล่อยให้ panic เกิดขึ้นขณะถือ lock อยู่

### 5. ล็อกในลำดับไม่สอดคล้องกันระหว่าง thread — deadlock แบบ circular wait

ตามที่แสดงในหัวข้อ 39.9 — ถ้าระบบมี `Mutex` มากกว่าหนึ่งตัวที่ต้องล็อกพร้อมกันในบางจุด (เช่นย้ายเงินข้ามสอง
บัญชี ต้องล็อกทั้งบัญชีต้นทางและปลายทาง) และแต่ละจุดในโค้ดล็อกในลำดับที่**ไม่สอดคล้องกัน** (บางที่ล็อก A ก่อน B
บางที่ล็อก B ก่อน A) มีโอกาสเกิด deadlock ได้เสมอไม่ว่าจะทดสอบผ่านกี่ครั้งก็ตาม (deadlock มักเกิดแบบ
**intermittent** คือเกิดเป็นบางครั้งขึ้นกับ timing ของ thread scheduler เท่านั้น ทำให้ตรวจจับได้ยากเป็นพิเศษใน
การทดสอบทั่วไป) **วิธีแก้**: กำหนดลำดับการล็อกที่แน่นอนตายตัวสำหรับทุก `Mutex` ที่อาจต้องถือพร้อมกัน (เช่น
เรียงตาม field ที่ไม่เปลี่ยนแปลง อย่าง account ID) แล้วบังคับให้ทุกจุดในโค้ดล็อกตามลำดับนั้นเสมอ ไม่มีข้อยกเว้น
— ถ้าจำเป็นต้องล็อกสองบัญชีพร้อมกันจริง ๆ ให้เขียนฟังก์ชันกลางที่รับผิดชอบลำดับการล็อกไว้ที่เดียว แล้วให้ทุกจุด
ในระบบเรียกผ่านฟังก์ชันนั้นเท่านั้น อย่าให้แต่ละจุดในโค้ดตัดสินใจลำดับการล็อกเอง

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เขียนโปรแกรมที่สร้าง `Arc<Mutex<Vec<i32>>>` เริ่มต้นเป็น vector ว่าง แล้ว spawn thread จำนวน 5
   ตัว โดยแต่ละ thread `.push()` ค่า `id * 100` (โดย `id` คือเลขลำดับ 0-4 ของ thread นั้น) เข้าไปใน vector
   ที่แบ่งกันใช้ หลัง `join()` ครบทุก thread แล้ว พิมพ์ vector ทั้งหมดออกมา และตรวจสอบด้วย `assert_eq!` ว่า
   ความยาวของ vector เท่ากับ 5 พอดี (คำใบ้: ระวังเรื่องลำดับของสมาชิกใน vector — เพราะ thread ทำงานคู่ขนาน
   ลำดับที่ push เข้าไปจะไม่แน่นอน ให้ `.sort()` ก่อนเทียบผลลัพธ์ถ้าต้องการเช็คเนื้อหาที่แน่นอน)

2. **(กลาง)** ปรับตัวอย่าง shared counter ในหัวข้อ 39.7 ให้เปลี่ยนจาก "เพิ่มค่าเข้า" (`+= 1`) เป็น
   "ลดค่าลง" (`-= 1`) ครึ่งหนึ่งของ thread และ "เพิ่มค่าเข้า" อีกครึ่งหนึ่ง โดยเริ่มต้นค่าที่ 0 — ให้แต่ละ
   thread ทำงาน 1,000 ครั้ง (thread เพิ่มค่าทำ `+= 1` จำนวน 1,000 ครั้ง, thread ลดค่าทำ `-= 1` จำนวน 1,000
   ครั้ง) ทำนายผลลัพธ์สุดท้ายก่อนรัน แล้วรันจริงหลายครั้งเพื่อพิสูจน์ว่าผลลัพธ์ตรงกับที่ทำนายไว้เสมอ ไม่มี
   ครั้งใดที่ผิดเพี้ยนไปจาก data race เลย

3. **(ยาก)** เขียนฟังก์ชัน `transfer(accounts: &Arc<Mutex<HashMap<String, f64>>>, from: &str, to: &str,
   amount: f64) -> Result<(), TxError>` ที่ **ล็อก `Mutex` เพียงครั้งเดียว** (ไม่ใช่สองครั้งแบบ `withdraw`
   แยกจาก `deposit` ในหัวข้อ 39.11) เพื่อทำทั้งการหักเงินจากบัญชีต้นทางและเพิ่มเงินให้บัญชีปลายทางเป็น
   **atomic operation เดียวจริง ๆ** (ไม่มีช่วงเวลาที่เงินหักออกไปแล้วแต่ยังไม่เข้าบัญชีปลายทางที่ผู้อื่นมองเห็น
   ได้เลย) แล้วเขียนโปรแกรมทดสอบที่มีหลาย thread เรียก `transfer` แบบสุ่มพร้อมกันจำนวนมาก (อย่างน้อย 10,000
   ธุรกรรม) แล้วพิสูจน์ด้วย `assert_eq!` ว่ายอดรวมทั้งระบบไม่เปลี่ยนแปลงเลย (คำใบ้: ปัญหาของการล็อกสองบัญชี
   พร้อมกันในฟังก์ชันเดียวคือต้องระวัง deadlock ตามหัวข้อ 39.9 — ถ้าคุณล็อกทั้ง `from` และ `to` ในฟังก์ชันนี้
   ต้องมีกฎลำดับการล็อกที่แน่นอน แม้ในกรณีนี้เพราะทุกอย่างอยู่ใน `Mutex` ตัวเดียว (`HashMap` ก้อนเดียว ไม่ใช่
   `Mutex` แยกต่อบัญชี) ปัญหา deadlock ข้าม `Mutex` หลายตัวจึงไม่เกิดขึ้น — แต่ลองคิดต่อว่าถ้าออกแบบระบบใหม่ให้
   แต่ละบัญชีมี `Mutex` ของตัวเอง (`HashMap<String, Mutex<f64>>` ห่อด้วย `Arc` อีกชั้น) จะต้องระวังเรื่องอะไร
   เพิ่มเติมจากที่เรียนในหัวข้อ 39.9)

4. **(ประยุกต์ใช้งานจริง)** ออกแบบและเขียนระบบ "ตัวจองที่นั่งภาพยนตร์" (คล้ายตัวอย่างระบบจองตั๋วที่ style
   guide แนะนำ) โดยมี `Arc<Mutex<HashMap<u32, bool>>>` เก็บสถานะที่นั่ง (หมายเลขที่นั่ง → จองแล้วหรือไม่) ให้
   มี seat ทั้งหมด 20 ที่นั่ง (หมายเลข 0-19) จำลองลูกค้า 50 คน (50 thread) พยายามจองที่นั่งแบบสุ่มพร้อมกัน โดย
   แต่ละคนจะพยายามจองไปเรื่อย ๆ (สุ่มเลขที่นั่งใหม่) จนกว่าจะจองสำเร็จหนึ่งที่ หรือจนกว่าจะลองครบ 100 ครั้งแล้ว
   ยอมแพ้ (ที่นั่งเต็มหมดพอดี เพราะลูกค้ามากกว่าที่นั่ง) เขียนฟังก์ชัน `try_book(seats: &Arc<Mutex<HashMap<u32,
   bool>>>, seat_number: u32) -> bool` ที่คืน `true` ถ้าจองสำเร็จ (ที่นั่งนั้นยังว่างอยู่และตอนนี้ถูกจองแล้ว)
   หรือ `false` ถ้าที่นั่งนั้นถูกจองไปแล้วก่อนหน้า — หลังจากทุก thread จบงาน ให้พิมพ์สรุปว่ามีลูกค้าจองสำเร็จกี่คน
   (ต้องได้พอดี 20 คน ไม่มากไม่น้อย เพราะมีที่นั่งแค่ 20 ที่) และตรวจสอบว่า**ไม่มีที่นั่งใดถูกจองซ้ำสองครั้ง**
   (ไม่มีลูกค้าสองคนได้ที่นั่งเดียวกัน) — นี่คือการพิสูจน์ว่า `Mutex` ป้องกันปัญหา "double booking" ที่เป็นบั๊ก
   จริงและร้ายแรงมากในระบบจองที่นั่ง/ห้อง/ตั๋วในโลกจริงได้อย่างสมบูรณ์

## สรุป

บทนี้ตอบคำใบ้ที่ Part 28 ทิ้งไว้ทุกจุดอย่างเต็มรูปแบบ: เราเริ่มจากการทวนว่า Part 38 แก้ปัญหา concurrency ด้วย
**message passing** (ไม่แบ่งข้อมูลกันเลย) ส่วนบทนี้แก้ปัญหาประเภทเดียวกันด้วยแนวทางตรงข้าม คือ **shared-state
concurrency** — ยอมให้หลาย thread แบ่งข้อมูลกันจริง ๆ อย่างปลอดภัย

เราเรียนรู้ `Mutex<T>` ในฐานะ **`RefCell<T>` เวอร์ชันข้าม thread**: ให้ interior mutability ผ่าน `.lock()` ที่
คืน `LockResult<MutexGuard<T>>` (`Result` จาก Part 12) — เข้าใจว่าทำไมต้องเป็น `Result`: **mutex poisoning**
เมื่อ thread หนึ่ง panic ขณะถือ lock ทำให้ thread อื่นได้รับคำเตือนผ่าน `Err` แทนการใช้ข้อมูลที่อาจเสียหายไปแล้ว
อย่างเงียบ ๆ และ `.lock().unwrap()` คือแนวปฏิบัติที่ใช้กันจริงเมื่อไม่จำเป็นต้องกู้คืนจาก poisoning — จากนั้นเรา
เห็น `MutexGuard<T>` ในฐานะ smart pointer แบบ RAII (เชื่อมกับ Part 6/27-29) ที่ปลดล็อกอัตโนมัติเมื่อหลุด scope
พร้อมเหตุผลใหม่ที่สำคัญกว่าเดิมว่าทำไมต้องจำกัด scope ของ guard ให้เล็ก: **มันบล็อก thread อื่นจริง ไม่ใช่แค่
เสี่ยง panic แบบ `RefCell` ใน single-thread**

เราเห็นว่าทำไมแบ่ง `Mutex<T>` ตรง ๆ ข้ามหลาย thread ไม่ได้ (`E0382` — กฎ ownership เดิมจาก Part 6) และทำไม
`Rc<T>` จาก Part 28 ก็ใช้แก้ปัญหานี้ไม่ได้เช่นกัน (`E0277` — ตัวนับของมันเองไม่ thread-safe เพราะไม่ได้ใช้
atomic operation) ซึ่งนำไปสู่ `Arc<T>` — ฝาแฝดของ `Rc<T>` ที่ใช้ atomic increment/decrement ที่ระดับ CPU
รับประกันความถูกต้องแม้หลาย thread เรียกพร้อมกันจริง — การรวม `Arc<Mutex<T>>` เข้าด้วยกันคือคู่หูคลาสสิกที่เรา
พิสูจน์ด้วยการรันจริงซ้ำหลายครั้งว่าให้ผลลัพธ์ถูกต้อง 100% ไม่มี increment สูญหายแม้แต่ครั้งเดียว — เราเรียน
`RwLock<T>` สำหรับสถานการณ์ read-heavy ที่อนุญาตให้อ่านพร้อมกันได้หลาย thread และเรียนรู้ **deadlock** อย่าง
ละเอียด (ที่ compiler ตรวจจับให้ไม่ได้เลย) พร้อมตัวอย่างที่ค้างจริงและวิธีป้องกันมาตรฐาน (ล็อกตามลำดับเดียวกัน
ทั้งระบบเสมอ) — ปิดท้ายด้วยตารางเปรียบเทียบ `Rc<RefCell<T>>` กับ `Arc<Mutex<T>>` แบบเต็มรูปแบบ และตัวอย่าง
ระบบธนาคารจำลองที่พิสูจน์ว่า `Arc<Mutex<HashMap<String, f64>>>` รักษาความถูกต้องของยอดเงินรวมได้แม้มีการแข่งขัน
เข้าถึงจากหลาย thread จริงจัง

ตลอดบทนี้เราใช้คำว่า `Send` (ปลอดภัยที่จะย้ายข้าม thread) แบบไม่เป็นทางการซ้ำแล้วซ้ำอีก — เห็น error `E0277`
ที่บอกตรง ๆ ว่า "`Rc<...>` ไม่ implement `Send`" และเห็นว่า `Arc<T>` แก้ปัญหาได้เพราะมัน implement `Send` ได้
จริง — **Part 40 (Send, Sync และความปลอดภัยของ Concurrency)** จะให้คำนิยามที่เป็นทางการเต็มรูปแบบของทั้งสอง
trait นี้: `Send` คืออะไรจริง ๆ ในระดับ type system, `Sync` ต่างจาก `Send` อย่างไร (คำใบ้: `Sync` เกี่ยวกับ
`&T` ที่แบ่งกันใช้ ส่วน `Send` เกี่ยวกับการย้าย `T` ทั้งก้อน), compiler ตัดสินใจให้ type ไหน implement มัน
โดยอัตโนมัติ (auto trait) ด้วยกฎอะไร, และทำไม `Rc<T>`/`RefCell<T>` ไม่ implement ทั้งสองตัวเลยในขณะที่
`Arc<T>`/`Mutex<T>` implement ได้ — คำอธิบายที่เป็นทางการนี้จะทำให้ทุก error message ที่เจอมาตลอดบทนี้และ
Part 28/37 (`E0277 cannot be sent between threads safely`) กลายเป็นเรื่องที่เข้าใจได้ลึกถึงต้นตอ ไม่ใช่แค่
"จำไว้ว่าต้องใช้ `Arc` แทน `Rc`" อีกต่อไป

---

**Part ก่อนหน้า:** [Channels](part-038-channels.md) | **Part ถัดไป:** [Send, Sync และความปลอดภัยของ Concurrency](part-040-send-sync.md)
