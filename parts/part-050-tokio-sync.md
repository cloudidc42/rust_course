# Part 50: Async Channels และ Synchronization (tokio::sync)

> โมดูล: ระดับสูง (Advanced) | ระดับ: สูง | เวลาโดยประมาณ: 240 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้อย่างละเอียดว่าทำไม `std::sync::Mutex::lock()` (Part 39) เป็นตัวเลือกที่ **อันตราย** เมื่อใช้ข้างใน
  async task ที่รันอยู่บน Tokio worker thread — เชื่อมโยงตรงกับคำเตือนเรื่อง "blocking call ข้างใน async"
  ที่ Part 48 วางไว้ และเห็นด้วยการรันจริงว่าการถือ `std::sync::MutexGuard` ข้าม `.await` point ทำให้เกิด
  อันตรายได้ **สองแบบที่ต่างกัน**: บางครั้ง compiler ปฏิเสธไม่ให้ compile ตั้งแต่ต้น บางครั้ง compile ผ่านแต่
  กลายเป็น **deadlock จริง** ที่ค้างตลอดไป
- ใช้ `tokio::sync::Mutex` ได้อย่างถูกต้อง — เข้าใจว่า `.lock()` ของมันเป็น **async method** (`.lock().await`)
  ที่ **คืน task กลับให้ scheduler** ระหว่างรอ lock แทนการบล็อก OS thread ทั้งเส้น และรีเมค shared counter จาก
  Part 39 ให้ใช้ `Arc<tokio::sync::Mutex<T>>` กับ `tokio::spawn` แทน `std::thread::spawn` โดยพิสูจน์ด้วยการรันจริง
  ว่าให้ผลลัพธ์ถูกต้อง 100%
- ใช้ **กฎการตัดสินใจ** ที่สำคัญที่สุดของบทนี้ได้อย่างมั่นใจ: เมื่อไหร่ควรใช้ `std::sync::Mutex` เดิม (เมื่อ
  critical section สั้นและไม่มี `.await` ข้างใน — ใช้ได้ดีแม้อยู่ใน async code และเร็วกว่าด้วยซ้ำ) เมื่อไหร่ต้อง
  เปลี่ยนไปใช้ `tokio::sync::Mutex` (เมื่อจำเป็นต้องถือ lock ข้าม `.await` point จริง ๆ)
- ใช้ `tokio::sync::RwLock` เป็นฝาแฝด async ของ `std::sync::RwLock` (Part 39) พร้อม `.read().await`/
  `.write().await`
- ใช้ `tokio::sync::mpsc` เป็นฝาแฝด async ของ `std::sync::mpsc` (Part 38) — เข้าใจว่า `.send().await` ทำให้เกิด
  **backpressure จริงแบบไม่บล็อก thread** เมื่อ bounded channel เต็ม และรีเมค worker pool/pipeline แบบ Part 38
  ให้ใช้ async channel กับ `tokio::spawn`
- ใช้ `tokio::sync::oneshot` สำหรับรูปแบบ "ถาม-ตอบครั้งเดียว" ระหว่าง task — และเข้าใจว่านี่คือคำตอบเต็มรูปแบบของ
  คำใบ้ที่ Part 38 ทิ้งไว้เรื่อง "reply channel ฝังอยู่ในข้อความ"
- ใช้ `tokio::sync::broadcast` เพื่อกระจายข้อความหนึ่งชิ้นไปให้ **หลาย receiver พร้อมกัน** — และเข้าใจว่านี่คือ
  เครื่องมือที่ถูกต้องระดับ production สำหรับ pattern "chat server broadcast" ที่ Part 49 ร่างไว้แบบง่าย ๆ ด้วย
  `Arc<Mutex<Vec<Sender>>>`
- ใช้ `tokio::sync::watch` สำหรับสถานการณ์ที่ผู้รับสนใจแค่ **ค่าล่าสุด** เท่านั้น (เช่น configuration update)
- ใช้ `tokio::sync::Semaphore` เพื่อจำกัดจำนวนงานที่ทำพร้อมกันสูงสุด (เช่น "connection พร้อมกันได้ไม่เกิน N")
  พร้อมเข้าใจว่า permit ที่ได้จาก `.acquire()` ทำงานแบบ RAII (Part 6) เหมือนกับ `MutexGuard`
- เลือกใช้ primitive ที่ถูกต้องได้เองจากตารางเปรียบเทียบ `std::sync` เทียบกับ `tokio::sync` และปิดท้ายด้วยการ
  สร้าง chat server เวอร์ชัน "production-quality" จริงที่ใช้ทั้ง `broadcast` และ `Semaphore` ร่วมกัน

## ความรู้ที่ต้องมีมาก่อน

บทนี้คือบทปิดของ mini-arc เรื่อง async/Tokio (Part 46-50) และเป็น**คำตอบเต็มรูปแบบ**ของคำใบ้หลายจุดที่บทก่อน ๆ
ในหลักสูตรทิ้งไว้ ถ้าคุณยังไม่แม่นเนื้อหาต่อไปนี้ แนะนำให้กลับไปทวนก่อน เพราะบทนี้จะไม่สอนพื้นฐานพวกนั้นซ้ำอีก:

- **Part 39 (Mutex, Arc และ Shared-State Concurrency)** — นี่คือความรู้ที่จำเป็นที่สุดสำหรับบทนี้ บทนี้คือ
  **เวอร์ชัน async-native** ของทุกอย่างที่ Part 39 สอน: `std::sync::Mutex`/`RwLock`/`Arc` ทำงานถูกต้องสมบูรณ์
  สำหรับ OS thread แต่พฤติกรรม **"บล็อก (block) thread ที่เรียก จนกว่าจะได้ lock"** ของมันกลายเป็นปัญหาใหม่
  ทันทีที่ย้ายมาอยู่ในโลก async — บทนี้จะอธิบายว่าทำไม และแนะนำเครื่องมือที่ออกแบบมาสำหรับโลก async โดยเฉพาะ
  คุณต้องจำ `MutexGuard<T>` แบบ RAII, มโนภาพของ critical section, และ deadlock จาก Part 39 ได้แม่นเพื่อเทียบ
  กับเนื้อหาบทนี้บรรทัดต่อบรรทัด
- **Part 38 (Channels)** — บทนี้คือเวอร์ชัน async-native ของ `std::sync::mpsc` เช่นกัน: `Sender`/`Receiver`,
  `.send()`/`.recv()`, multiple producer, bounded channel กับ backpressure — ทุกแนวคิดเหล่านี้กลับมาอีกครั้งใน
  รูปแบบ `.await` ได้ และบทนี้จะ**แก้คำใบ้ที่ Part 38 ทิ้งไว้ตรง ๆ**: ตอนที่ Part 38 พูดถึง pattern "ฝัง reply
  channel ไว้ในข้อความ" มันบอกไว้ชัดว่า `tokio::sync::oneshot` คือเครื่องมือระดับ production สำหรับ pattern นั้น
  — บทนี้คือจุดที่คำใบ้นั้นถูกอธิบายเต็มรูปแบบพร้อมโค้ดจริง
- **Part 48 (Tokio: Runtime และ Tasks)** — บทนี้ใช้ `tokio::spawn`, `#[tokio::main]`, และแนวคิดเรื่อง **worker
  thread pool ของ Tokio runtime** ตรง ๆ ในทุกตัวอย่าง คุณต้องเข้าใจมาก่อนว่า async task ไม่ได้รันแต่ละตัวบน
  OS thread ของตัวเอง แต่ถูก **สลับ (interleave)** กันรันบน worker thread จำนวนจำกัดที่ executor คุมอยู่ — และ
  ต้องเข้าใจคำเตือนเรื่อง **"blocking call ข้างใน async ทำให้ worker thread ทั้งเส้นค้าง ทำงานอย่างอื่นไม่ได้เลย
  ตราบใดที่ blocking call นั้นยังไม่จบ"** ที่ Part 48 อธิบายไว้ — บทนี้คือจุดที่คำเตือนนั้นถูกนำมาปะทะกับ
  `std::sync::Mutex::lock()` ตรง ๆ เพื่อแสดงให้เห็นว่ามันคือ blocking call ประเภทหนึ่งเป๊ะ ๆ
- **Part 49 (Tokio: I/O และ Networking)** — บทนี้ใช้ `TcpListener`/`TcpStream`, `tokio::io::AsyncReadExt`/
  `AsyncWriteExt`, และ `tokio::select!` ต่อยอดตรง ๆ ในตัวอย่างเซิร์ฟเวอร์แชท และเป็นคำตอบเต็มรูปแบบของ pattern
  "broadcast แชทให้ทุก client" ที่ Part 49 ร่างไว้แบบง่าย ๆ ด้วย `Arc<Mutex<Vec<Sender>>>` (เก็บ list ของ
  sender ไว้ในโครงสร้างที่ต้องล็อกเองทุกครั้งที่มีคนเข้า/ออกห้อง และต้องวนส่งข้อความให้ทุกคนเอง) — บทนี้จะแสดง
  ให้เห็นว่า `tokio::sync::broadcast` แก้ปัญหาเดียวกันได้สะอาดกว่ามากแค่ไหน
- **Part 6 (Ownership เบื้องต้น และ Drop)** — permit ที่ได้จาก `Semaphore::acquire()` และ guard ที่ได้จาก
  `tokio::sync::Mutex::lock()` ทำงานแบบ RAII เหมือนกับที่ Part 6 สอนเรื่อง `Drop` ทุกประการ — ปล่อยคืนอัตโนมัติ
  เมื่อหลุด scope
- **Part 12 (Result\<T, E\>)** — ทุก method แบบ async ในบทนี้ (`.lock().await`, `.recv().await`,
  `.send().await`, `.acquire().await`) ยังคงคืนค่าที่เกี่ยวข้องกับ `Result`/`Option` เหมือนฝาแฝด `std::sync`
  ของมัน ต้องคุ้นกับการ `match`/`?` มาก่อน

## เนื้อหา

### 50.1 ทวนภาพรวม: mini-arc เรื่อง Async/Tokio กำลังจะปิดครบ

ก่อนเข้าเนื้อหาหลัก มาทวนเส้นทางที่เดินมาตลอด 4 บทที่แล้วกันก่อน เพื่อให้เห็นว่าบทนี้อยู่ตรงไหนของภาพใหญ่:

- **Part 46** สอน `async`/`.await` เบื้องต้น — วิธีเขียนโค้ดที่ "รอ" ได้โดยไม่บล็อก thread
- **Part 47** เปิดฝา "เครื่องยนต์" ข้างใน — `Future` trait, `poll()`, executor คืออะไรจริง ๆ
- **Part 48** แนะนำ **Tokio** ในฐานะ runtime ระดับ production — `#[tokio::main]`, `tokio::spawn`, และ
  แนวคิดสำคัญที่สุดที่บทนี้จะใช้ต่อ: **worker thread pool ที่ executor ใช้สลับกันรัน task หลายพันตัว** พร้อม
  คำเตือนว่า **blocking call ข้างใน async task เป็นอันตรายร้ายแรง** เพราะมันกินสิทธิ์ worker thread ทั้งเส้น
  ไปตลอดเวลาที่มันบล็อก ทำให้ task อื่นทุกตัวที่ควรจะได้รันบน thread เดียวกันนั้นต้องหยุดชะงักไปด้วย
- **Part 49** ใช้ Tokio ทำ I/O และ networking จริง — TCP/UDP server ที่รับหลาย connection พร้อมกัน และร่าง
  แนวคิด "chat server ที่กระจายข้อความให้ทุกคน" แบบง่าย ๆ ด้วยการเก็บ list ของ sender ไว้ใน
  `Arc<Mutex<Vec<Sender>>>` (ยืมเครื่องมือจาก Part 38/39 มาใช้ไปก่อน โดยที่ยังไม่มีเครื่องมือที่เหมาะสมกว่านั้น
  ให้ใช้)

สังเกตคำว่า **"ยืมเครื่องมือจาก Part 38/39 มาใช้ไปก่อน"** — นี่คือประเด็นสำคัญที่สุดที่บทนี้จะเจาะลึก: เครื่องมือ
จาก Part 38/39 (`std::sync::mpsc`, `std::sync::Mutex`, `std::sync::Arc` — ทุกตัวถูกออกแบบมาสำหรับโลกของ
**OS thread ล้วน ๆ**) ยังคง **compile ผ่านได้** เมื่อนำมาใช้ในโค้ด async แต่การใช้งานบางรูปแบบกลับกลายเป็น
อันตรายที่ร้ายแรงกว่าที่คิด และบางรูปแบบก็ยังใช้ได้ดีอยู่ไม่มีปัญหาอะไรเลย — บทนี้จะสอนให้แยกแยะสองกรณีนี้ให้ได้
อย่างแม่นยำ ก่อนแนะนำชุดเครื่องมือใหม่ทั้งชุดจาก `tokio::sync` ที่ออกแบบมาสำหรับโลก async โดยเฉพาะ ปิดท้ายด้วย
การนำ `tokio::sync::broadcast` มาแก้ pattern ที่ Part 49 ร่างไว้อย่างสมบูรณ์

### 50.2 ปัญหาหลัก: `std::sync::Mutex::lock()` เป็น Blocking Call

จาก Part 39 เราเรียนมาว่า `Mutex<T>::lock()` **บล็อก (block) thread ที่เรียกไว้จนกว่าจะได้ lock** — ถ้า thread
อื่นถือ lock อยู่แล้ว thread ที่เรียก `.lock()` จะหยุดทำงานอย่างสมบูรณ์ (ไม่กิน CPU แบบ busy-loop, OS จะเอา
thread นั้นออกจากคิวการรันจนกว่า lock จะว่าง) จนกว่าจะได้สิทธิ์เข้าถึงจริง — ในโลกของ OS thread ล้วน ๆ (Part
37-39) นี่ไม่ใช่ปัญหาเลย เพราะแต่ละ thread มี **stack และตัวตนของตัวเองแยกจากกันเด็ดขาด** การที่ thread หนึ่ง
หยุดรออยู่ไม่กระทบ thread อื่นแม้แต่นิดเดียว — OS scheduler จะสลับไปรัน thread อื่นที่พร้อมทำงานแทนโดยอัตโนมัติ

แต่ Part 48 สอนไปแล้วว่าโลกของ async **ไม่ทำงานแบบนั้น**: Tokio runtime ไม่ได้สร้าง OS thread ใหม่ให้ทุก task
ที่ `tokio::spawn` ขึ้นมา — มันสร้าง **worker thread pool ขนาดจำกัด** (ปกติเท่ากับจำนวน CPU core) แล้วให้
executor **สลับ (interleave)** การรัน task หลายพัน หลายหมื่นตัวบน worker thread จำนวนน้อยนั้นซ้ำไปซ้ำมา — กลไก
การสลับนี้ทำงานได้เพราะทุก task ที่เขียนถูกต้องจะ **"คืนสิทธิ์การรัน" (yield)** ให้ executor เองตรง `.await`
point ทุกจุดที่ยังไม่มีผลลัพธ์พร้อม แทนที่จะยึด thread ไว้ทั้งหมด

นี่คือจุดที่ปัญหาเกิด: `std::sync::Mutex::lock()` **ไม่รู้จักแนวคิดของ "การคืนสิทธิ์การรันให้ executor"** เลย
มันเป็นฟังก์ชันแบบ synchronous ธรรมดาที่เขียนมาก่อนที่ async/await จะมีในภาษา Rust ด้วยซ้ำ — เมื่อมันบล็อก มัน
**บล็อกที่ระดับ OS thread จริง ๆ** ไม่ใช่ "บล็อกที่ระดับ task" — และ worker thread ที่ถูกบล็อกอยู่นั้นก็คือ
worker thread เส้นเดียวกันที่ executor ใช้รัน task **อื่น ๆ** อีกนับพันตัวอยู่ด้วย ผลคือ: **ตราบใดที่ `.lock()`
ยังไม่ได้ lock กลับมา task อื่นทุกตัวที่ถูกกำหนดให้รันบน worker thread เส้นนั้นจะไม่ได้ทำงานเลยแม้แต่นิดเดียว**
— นี่คือ **"blocking call ข้างใน async" ตัวอย่างที่ตรงเป๊ะที่สุด** ที่ Part 48 เตือนไว้ (เทียบกับ `.await` ปกติ
ที่คืนสิทธิ์การรันให้ executor ทันทีที่ยังไม่มีผลลัพธ์พร้อม)

มาดูเปรียบเทียบให้เห็นความต่างชัด ๆ ก่อนลงรายละเอียดในหัวข้อต่อไป:

| | `std::sync::Mutex::lock()` | `.await` ปกติ (เช่น `tokio::time::sleep(...).await`) |
|---|---|---|
| เมื่อยังไม่พร้อม | **บล็อก OS thread ทั้งเส้น** ไม่ทำอะไรอื่นได้เลยจนกว่าจะพร้อม | **คืนสิทธิ์การรันให้ executor ทันที** (`Poll::Pending`) แล้ว executor ไปรัน task อื่นบน thread เดียวกันต่อได้ |
| ผลต่อ task อื่นบน worker thread เดียวกัน | **หยุดชะงักทั้งหมด** ตราบใดที่ยัง `.lock()` ไม่จบ | **ไม่กระทบเลย** — executor สลับไปรัน task อื่นได้ปกติ |
| เหมาะกับ | โค้ด OS thread ล้วน ๆ (Part 37-39) | โค้ด async ทุกชนิดที่รันบน Tokio (หรือ executor async อื่น) |

หัวข้อถัดไปจะแสดงให้เห็นว่าปัญหานี้แสดงตัวออกมาในโค้ดจริงได้สองรูปแบบ — และรูปแบบที่สองอันตรายกว่าที่คิดมาก

### 50.3 การถือ `std::sync::MutexGuard` ข้าม `.await` Point: อันตรายสองระดับ

ลองนึกภาพเขียน async function ที่ล็อก `std::sync::Mutex` แล้ว "เผลอ" `.await` อะไรบางอย่างในระหว่างที่ยังถือ
`MutexGuard` อยู่ (ยังไม่หลุด scope) — จาก Part 39 เราเรียนมาว่า `MutexGuard<T>` เป็น RAII guard ที่ปลดล็อก
อัตโนมัติตอนหลุด scope เท่านั้น มันไม่รู้อะไรเกี่ยวกับ async เลย ลองเขียนโค้ดแบบนี้ดูตรง ๆ:

```rust
use std::sync::{Arc, Mutex};
use std::time::Duration;

async fn increment_and_wait(data: Arc<Mutex<i32>>) {
    let mut guard = data.lock().unwrap();
    *guard += 1;
    // ★ guard ยังไม่ถูก drop ที่นี่ -- มันจะ "อยู่ข้าม" await point ต่อไปนี้ไปด้วย
    tokio::time::sleep(Duration::from_millis(10)).await;
    println!("value = {}", *guard);
}

#[tokio::main]
async fn main() {
    let data = Arc::new(Mutex::new(0));
    let handle = tokio::spawn(increment_and_wait(data));
    handle.await.unwrap();
}
```

โค้ดนี้ **compile ไม่ผ่าน** เมื่อรันด้วย `cargo build` จริง:

```
error: future cannot be sent between threads safely
   --> src/main.rs:15:31
    |
 15 |     let handle = tokio::spawn(increment_and_wait(data));
    |                               ^^^^^^^^^^^^^^^^^^^^^^^^ future returned by `increment_and_wait` is not `Send`
    |
    = help: within `impl Future<Output = ()>`, the trait `Send` is not implemented for `std::sync::MutexGuard<'_, i32>`
note: future is not `Send` as this value is used across an await
   --> src/main.rs:8:51
    |
  5 |     let mut guard = data.lock().unwrap();
    |         --------- has type `std::sync::MutexGuard<'_, i32>` which is not `Send`
...
  8 |     tokio::time::sleep(Duration::from_millis(10)).await;
    |                                                   ^^^^^ await occurs here, with `mut guard` maybe used later
note: required by a bound in `tokio::spawn`
   --> .../tokio-1.53.1/src/task/spawn.rs:176:21
    |
174 |     pub fn spawn<F>(future: F) -> JoinHandle<F::Output>
    |            ----- required by a bound in this function
175 |     where
176 |         F: Future + Send + 'static,
    |                     ^^^^ required by this bound in `spawn`
```

**อ่าน error นี้ให้ทะลุ เพราะมันเชื่อมทุกอย่างที่เรียนมาก่อนหน้านี้เข้าด้วยกัน:**

- `tokio::spawn` (Part 48) กำหนดไว้ว่า `F: Future + Send + 'static` — future ที่ส่งเข้าไปต้อง implement
  `Send` (Part 39/40: ปลอดภัยที่จะย้ายข้าม thread ได้) เพราะ Tokio runtime แบบ multi-thread (ค่าเริ่มต้นของ
  `#[tokio::main]`) อาจย้าย task ไปรันบน worker thread คนละเส้นได้ทุกครั้งที่มันถูก poll ใหม่ — เพื่อให้ Tokio
  ทำแบบนั้นได้อย่างปลอดภัย **ทุกอย่างที่ future เก็บอยู่ข้างในตัวมันเอง ณ ทุกจุดที่มันอาจถูกพักไว้ (คือทุก
  `.await` point) ต้อง `Send`**
- **`std::sync::MutexGuard<'_, T>` ถูกออกแบบมาให้ `Send` ไม่ได้โดยเจตนา** — นี่คือกฎที่ standard library กำหนด
  มาโดยตั้งใจ เหตุผลจริง ๆ คือ mutex บางระบบปฏิบัติการมีแนวคิด "thread ที่ปลดล็อกต้องเป็น thread เดียวกันกับ
  ที่ล็อกไว้" (เช่น pthread mutex บางโหมด) การอนุญาตให้ `MutexGuard` ย้ายข้าม thread ได้จะขัดกับสมมติฐานนี้ใน
  บาง platform — ผลคือ compiler รู้ตั้งแต่ compile time ว่า **type นี้ห้ามอยู่ในค่าที่จะถูกย้ายข้าม thread**
- เมื่อ `guard` (ตัวแปรของ type ที่ `Send` ไม่ได้) ยังมีชีวิตอยู่ **ข้าม** `.await` point (สังเกต comment
  `with 'mut guard' maybe used later` — compiler วิเคราะห์ตรง ๆ ว่าตัวแปรนี้ยัง "อาจถูกใช้อีก" หลัง await ที่นี่)
  Rust ต้องเก็บ `guard` ไว้เป็นส่วนหนึ่งของ **state ข้างในของ future ที่ `increment_and_wait` สร้างขึ้นมา**
  (เพราะ future ต้องเก็บทุกตัวแปร local ที่ยังมีชีวิตอยู่ตอนถูกพัก — Part 47 อธิบายไว้ว่า future คือ state
  machine) — พอ `MutexGuard` เป็นส่วนหนึ่งของ state ของ future ทั้งก้อน ก็แปลว่า **future ทั้งก้อนนั้นก็
  `Send` ไม่ได้ไปด้วย** ตาม transitively (ถ้าข้างในมี field ที่ `Send` ไม่ได้แม้แต่ field เดียว type ทั้งก้อน
  ก็ `Send` ไม่ได้)
- `tokio::spawn` ต้องการ future ที่ `Send` แต่ future ของเราไม่ `Send` — compiler จึงปฏิเสธ

**นี่คือข่าวดีที่ต้องเข้าใจให้ถูกจุด**: compiler **จับปัญหานี้ให้ก่อนที่โปรแกรมจะรันด้วยซ้ำ** ในกรณีที่คุณใช้
`tokio::spawn` (หรือ runtime แบบ multi-thread ที่ต้องการ `Send`) — นี่คือความปลอดภัยแบบเดียวกับที่ Rust ให้มา
ตลอดทั้งหลักสูตร: **บั๊กที่ภาษาอื่นต้องรอให้เกิด race condition จริงในโปรดักชันก่อนจะรู้ตัว Rust จับให้เห็นตั้งแต่
ตอน compile**

แต่ข่าวร้ายคือ **ไม่ใช่ทุกสถานการณ์ที่ compiler จะจับให้ได้แบบนี้** — ถ้าคุณไม่ได้ผ่าน `tokio::spawn` เลย (เช่น
`.await` โค้ดที่ถือ guard ข้ามอยู่ตรง ๆ ใน task เดียวกันโดยไม่มีการ spawn คั่นกลาง หรือ join ด้วย `tokio::join!`
ที่ไม่ต้องการ bound `Send` เพราะทุกอย่างยังอยู่ใน task เดียวกันเสมอ) โค้ดจะ **compile ผ่านได้สนิท** ทั้งที่ยัง
ถือ `MutexGuard` ข้าม `.await` อยู่เหมือนเดิม — และนี่คือจุดที่อันตรายที่ **ร้ายแรงกว่าการ compile ไม่ผ่านมาก**
เพราะโปรแกรมจะรันได้ปกติเป็นส่วนใหญ่ แต่แอบซ่อนความเป็นไปได้ของ **deadlock ถาวร** ไว้ ซึ่งจะแสดงให้เห็นจริงใน
หัวข้อถัดไป

### 50.4 พิสูจน์ Deadlock จริง: เมื่อ `std::sync::Mutex` ข้าม `.await` Compile ผ่านแต่ค้างตลอดไป

มาสร้างสถานการณ์ที่โค้ดถือ `std::sync::MutexGuard` ข้าม `.await` **โดยที่ compile ผ่านสนิท** (หลีกเลี่ยงเงื่อนไข
`Send` ของ `tokio::spawn` ด้วยการรันสอง task พร้อมกันผ่าน `tokio::join!` ในฟังก์ชันเดียวกันแทน) แล้วดูว่าเกิด
อะไรขึ้นจริงตอนรัน — เราจะใช้ runtime แบบ **`current_thread`** (มี worker thread เดียวเท่านั้น) เพื่อให้เห็นภาพ
ปัญหาชัดที่สุด (แม้ปัญหานี้เกิดได้แม้ใน runtime แบบ multi-thread เช่นกัน ถ้า worker thread ที่เหลือทั้งหมดกำลัง
ยุ่งอยู่พอดี เพียงแต่โอกาสเกิดต่ำกว่าเพราะมี thread อื่นสำรอง):

```rust
use std::sync::{Arc, Mutex};
use std::time::Duration;
use tokio::time::Instant;

#[tokio::main(flavor = "current_thread")]
async fn main() {
    let start = Instant::now();
    let data = Arc::new(Mutex::new(0));

    let data_a = Arc::clone(&data);
    let start_a = start;
    let task_a = async move {
        let mut guard = data_a.lock().unwrap();
        *guard += 1;
        println!(
            "[task A] ได้ lock แล้วที่ {:?} กำลังจะ .await ขณะยังไม่ปล่อย lock...",
            start_a.elapsed()
        );
        // ★ ถือ guard ข้าม await point นี้ตรง ๆ -- compile ผ่านเพราะไม่ได้ผ่าน tokio::spawn
        tokio::time::sleep(Duration::from_millis(200)).await;
        println!(
            "[task A] ทำงานต่อหลัง await ที่ {:?} ปล่อย lock ตรงนี้ (ไม่ควรพิมพ์ถึงตรงนี้เลย)",
            start_a.elapsed()
        );
    };

    let data_b = Arc::clone(&data);
    let start_b = start;
    let task_b = async move {
        tokio::time::sleep(Duration::from_millis(50)).await; // ให้ task A ได้ lock ไปก่อนแน่ ๆ
        println!(
            "[task B] กำลังขอ lock (เรียก .lock() แบบ blocking) ที่ {:?}...",
            start_b.elapsed()
        );
        let guard = data_b.lock().unwrap(); // ★ blocking call บน worker thread เดียวที่มีอยู่ทั้งหมด
        println!(
            "[task B] ได้ lock แล้ว: {} ที่ {:?} (ไม่ควรพิมพ์ถึงตรงนี้เลย)",
            *guard,
            start_b.elapsed()
        );
    };

    tokio::join!(task_a, task_b);
    println!("จบโปรแกรม (ไม่ควรพิมพ์ถึงตรงนี้เลย)");
}
```

โค้ดนี้ **compile ผ่านสนิท** (ไม่มี error เรื่อง `Send` เลย เพราะ `tokio::join!` รัน future ทั้งสองตัวภายใน
task เดียวกันเสมอ ไม่ต้องย้ายข้าม thread) — แต่รันจริงด้วย `timeout 3 ./program` (ใช้ `timeout` แบบเดียวกับที่
Part 39 ใช้พิสูจน์ self-deadlock) แล้วได้ผลว่า **โปรแกรมค้างและถูก kill หลัง 3 วินาที (exit code 124)**:

```
[task A] ได้ lock แล้วที่ 2.769µs กำลังจะ .await ขณะยังไม่ปล่อย lock...
[task B] กำลังขอ lock (เรียก .lock() แบบ blocking) ที่ 51.298477ms...
```

พิมพ์ได้แค่สองบรรทัดนี้เท่านั้น — บรรทัด `"[task A] ทำงานต่อหลัง await..."`, `"[task B] ได้ lock แล้ว..."`, และ
`"จบโปรแกรม..."` **ไม่ปรากฏเลยแม้แต่บรรทัดเดียว** โปรแกรมค้างตลอดไปจริง ๆ มาวิเคราะห์ทีละขั้นว่าทำไมถึงเกิดแบบนี้
(runtime แบบ `current_thread` มี **worker thread เดียวเท่านั้น** ที่ใช้ขับเคลื่อน task ทุกตัว รวมถึง executor
loop เองด้วย):

1. `tokio::join!` เริ่ม poll `task_a` ก่อน — `task_a` ล็อก `data_a.lock().unwrap()` สำเร็จ (ไม่มีใครถืออยู่)
   ได้ `guard` มา แล้วพิมพ์บรรทัดแรก จากนั้นเจอ `.await` ของ `sleep(200ms)` — นี่คือ `.await` ปกติ (ไม่ใช่
   blocking call) จึงคืน `Poll::Pending` กลับไปให้ `join!` ทันที **โดยที่ `guard` ยังมีชีวิตอยู่** (เก็บอยู่ใน
   state ของ future `task_a` ที่ถูกพักไว้ ตามที่อธิบายในหัวข้อ 50.3)
2. `join!` เห็น `task_a` ยัง `Pending` จึงไป poll `task_b` ต่อ — `task_b` เจอ `sleep(50ms)` ก่อน คืน
   `Pending` เช่นกัน (ยังไม่ทำอะไรกับ lock เลยตอนนี้)
3. Executor เห็นทั้งสอง future `Pending` พร้อมกัน จึงพัก thread ไปรอ **timer event** (กลไกภายในของ Tokio ตามที่
   Part 47/48 อธิบาย ไม่ใช่ busy-loop) จนกว่า timer ตัวใดตัวหนึ่งจะครบกำหนด
4. เมื่อ timer 50ms ของ `task_b` ครบก่อน (เร็วกว่า 200ms ของ `task_a`) executor ปลุก `task_b` ขึ้นมา poll ต่อ —
   `task_b` ผ่าน `.await` ของ sleep ไปแล้ว มาถึงบรรทัด `println!("[task B] กำลังขอ lock...")` แล้วเรียก
   `data_b.lock().unwrap()` — **นี่คือ blocking call จริง ไม่ใช่ `.await`** เพราะ `std::sync::Mutex::lock()`
   ไม่มีคำว่า `async` เลย มันจะ**บล็อกเธรดปัจจุบันตรงนั้นทันที**จนกว่าจะได้ lock
5. แต่ lock ยังถูก `guard` ของ `task_a` ถืออยู่ (ยังไม่หลุด scope เพราะ `task_a` ยังไม่ได้ถูก poll ต่อให้ผ่าน
   จุด `.await` ของมันไปจนถึงท้ายฟังก์ชันเลย) — `task_b` จึงต้องรอ
6. **ตรงนี้คือหัวใจของ deadlock**: การรอของ `task_b` ที่ข้อ 5 ไม่ใช่การ `.await` ที่คืนสิทธิ์การรันให้ executor
   — มันคือการบล็อก **thread ปัจจุบันเลย** (thread เดียวที่มีอยู่ทั้งระบบในโหมด `current_thread`) และ thread
   เส้นนี้ก็คือ thread เดียวกันเป๊ะที่ต้อง**นำ `task_a` กลับมา poll ต่อเพื่อให้มันเดินผ่าน `.await` ของ
   `sleep(200ms)` แล้วไปถึงจุดที่ `guard` หลุด scope และปลดล็อก** — แต่ thread เส้นนั้นกำลัง**ค้างอยู่ข้างใน
   `data_b.lock()` ของ `task_b` เองอยู่พอดี** ไม่มีทางกลับไปรัน executor loop เพื่อ poll `task_a` ได้อีกเลย
7. ผลคือ: `task_a` รอ timer 200ms ที่จะครบกำหนดจริงในไม่ช้า (มันครบแน่ ๆ ที่ระดับ OS timer) แต่**ไม่มีใครไปเช็ค
   ผลลัพธ์ของ timer นั้นแทน `task_a` ได้เลย** เพราะ thread เดียวที่มีอยู่ถูกยึดอยู่ในการรอ lock ที่ไม่มีวันถูก
   ปลดแล้ว (เพราะคนที่จะปลดคือ `task_a` เอง ซึ่งก็รอ thread เส้นเดียวกันนี้อยู่พอดี) — วงจรปิดสมบูรณ์: **circular
   wait ระดับเดียวกับที่ Part 39 อธิบายไว้ในหัวข้อ deadlock แบบ lock-ordering เป๊ะ ๆ เพียงแต่ตอนนี้ "ผู้เล่น" คือ
   task สอง task ที่แบ่ง thread เดียวกัน ไม่ใช่ thread สองตัวที่แบ่ง lock สองตัว**

**นี่คือเหตุผลที่การถือ `std::sync::MutexGuard` ข้าม `.await` point อันตรายกว่า plain blocking call ธรรมดา
(ที่ Part 48 เตือนไว้) อีกขั้นหนึ่ง**: blocking call ธรรมดา (เช่นเผลอเรียก `std::thread::sleep` ข้างใน async
fn) ทำให้ worker thread ช้าลงหรือค้าง **ชั่วคราว** จนกว่า call นั้นจะจบไปเอง แต่การถือ `std::sync::Mutex` ข้าม
`.await` สามารถสร้าง**สถานการณ์ที่ไม่มีวันจบเองได้เลย** ถ้า task ที่ต้องมาปลดล็อกดันเป็น task ที่ถูกกันไม่ให้
ได้รันโดยตัว blocking call นั้นเอง — วงจรที่ปิดสนิทแบบนี้ไม่มีทางคลายออกเองได้ไม่ว่าจะรอนานแค่ไหนก็ตาม

### 50.5 `tokio::sync::Mutex`: Async-Aware Mutex

คำตอบของ Tokio คือ `tokio::sync::Mutex<T>` — API หน้าตาคล้าย `std::sync::Mutex<T>` มากจนแทบจะสลับกันใช้แบบ
เขียนโค้ดผิวเผินได้ (ยังมี `MutexGuard`, ยังใช้ `Arc` แบ่งกันข้าม task ได้เหมือนกัน) แต่มีความต่างที่สำคัญที่สุด
อยู่จุดเดียว: **`.lock()` ของมันเป็น `async fn`** — เรียกแล้วต้อง `.await` เสมอ (`.lock().await` ไม่ใช่
`.lock()` เฉย ๆ)

ความต่างเชิงพฤติกรรมที่อยู่หลัง `.await` นั้นคือหัวใจทั้งหมดของบทนี้: เมื่อ lock ไม่ว่าง `tokio::sync::Mutex`
**ไม่บล็อก OS thread เลย** — มันคืน `Poll::Pending` กลับให้ executor (แบบเดียวกับ `.await` ปกติทุกตัวที่เรียน
มาใน Part 46-49) ทำให้ executor สามารถสลับไปรัน task **อื่น** บน worker thread เดียวกันได้ทันที จนกว่า lock
จะว่างแล้ว task นี้จะถูกปลุกขึ้นมาทำงานต่อ — **ไม่มีการยึด worker thread ไว้เฉย ๆ ระหว่างรอเลยแม้แต่วินาที
เดียว**

มารีเมค shared counter จาก Part 39 (หัวข้อ 39.7 ที่ใช้ `Arc<std::sync::Mutex<T>>` กับ `std::thread::spawn`) ให้
เป็นเวอร์ชัน async เต็มรูปแบบด้วย `Arc<tokio::sync::Mutex<T>>` กับ `tokio::spawn`:

```rust
use std::sync::Arc;
use tokio::sync::Mutex;

#[tokio::main]
async fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..3 {
        let counter = Arc::clone(&counter);
        let handle = tokio::spawn(async move {
            let mut num = counter.lock().await; // ★ .lock() เป็น async -- ต้อง .await
            *num += 1;
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.await.unwrap();
    }

    println!("Result: {}", *counter.lock().await);
}
```

ผลลัพธ์ (รันจริง):

```
Result: 3
```

**อธิบายทีละส่วน เทียบกับ Part 39.7 บรรทัดต่อบรรทัด:**

- `Arc::clone(&counter)` — เหมือนเดิมทุกประการกับ Part 39 (Part 39 หัวข้อ 39.6-39.7: `Arc<T>` ให้เจ้าของร่วม
  หลายตัวด้วย atomic reference counting ที่ปลอดภัยข้าม thread — ตอนนี้ก็ยังปลอดภัยข้าม task/thread เหมือนเดิม
  ไม่มีอะไรเปลี่ยนในส่วนนี้เลย เพราะปัญหาเรื่อง ownership ร่วมเป็นปัญหาคนละชั้นกับปัญหาเรื่อง blocking)
- `tokio::spawn(async move { ... })` แทน `thread::spawn(move || { ... })` — สร้าง **task** (Part 48) แทน OS
  thread จริง — นี่คือความต่างเดียวที่สำคัญในระดับ "การสร้างหน่วยงานคู่ขนาน"
- `counter.lock().await` แทน `counter.lock().unwrap()` — สังเกตว่า **ไม่มี `.unwrap()` ตรงนี้เลย** เพราะ
  `tokio::sync::Mutex::lock()` **คืนค่า `MutexGuard<T>` ตรง ๆ ไม่ผ่าน `Result`** — มันไม่มีแนวคิด **mutex
  poisoning** แบบ `std::sync::Mutex` (Part 39.3) เพราะ Tokio เลือกออกแบบให้เรียบง่ายกว่า: ถ้า task ที่ถือ lock
  panic ขณะถือ lock อยู่ Tokio จะปลดล็อกให้ (ไม่ poison) แล้วให้ task ถัดไปได้ lock ไปใช้ต่อตามปกติ — trade-off
  นี้เลือกความง่ายในการใช้งานมากกว่าการเตือนอย่างเข้มงวดแบบ `std::sync::Mutex` (เป็นทางเลือกเชิงออกแบบที่ต่างกัน
  จริง ไม่ใช่ "จำนวนน้อยกว่าเพราะยังไม่ทำ" — ผู้ใช้งานที่ต้องการ semantics แบบ poisoning เข้มงวดยังสามารถเลือก
  `std::sync::Mutex` ต่อไปได้ในกรณีที่ critical section ไม่มี `.await`)
- ผลลัพธ์ `Result: 3` ถูกต้อง 100% เหมือนกับที่ Part 39 พิสูจน์ไว้กับ `std::sync::Mutex` — ยืนยันว่า
  `tokio::sync::Mutex` ป้องกัน data race ได้จริงเหมือนกันทุกประการ เพียงแค่เปลี่ยนกลไกการรอจาก "บล็อก OS
  thread" เป็น "คืนสิทธิ์ให้ executor" เท่านั้น

### 50.6 กฎการตัดสินใจ: เมื่อไหร่ใช้ `std::sync::Mutex`, เมื่อไหร่ใช้ `tokio::sync::Mutex`

ถึงจุดนี้ผู้เรียนจำนวนมากจะสรุปแบบเข้าใจผิดที่พบบ่อยที่สุดในหัวข้อนี้ทั้งหมด: **"ในเมื่อ std::sync::Mutex
อันตรายในโค้ด async แบบนี้ ก็แปลว่าต้องใช้ tokio::sync::Mutex ทุกครั้งที่เขียนโค้ด async สินะ"** — ข้อสรุปนี้
**ผิด** และเป็น overcorrection ที่ทำให้โค้ด async จำนวนมากช้าลงโดยไม่จำเป็น กฎที่ถูกต้องคือ:

> **ปัญหาไม่ได้อยู่ที่ "อยู่ในโค้ด async หรือไม่" — ปัญหาอยู่ที่ "critical section (ช่วงที่ถือ lock) มี `.await`
> point อยู่ข้างในหรือไม่"**

ถ้า critical section **สั้นและไม่มี `.await` เลยแม้แต่จุดเดียว** (แค่แก้ค่าตัวเลข, push เข้า `Vec`, อ่าน field
ธรรมดา) `std::sync::Mutex` **ยังใช้ได้ดีและเหมาะสมกว่าด้วยซ้ำ** แม้อยู่ในโค้ด async ก็ตาม — เพราะเวลาที่ lock
ถูกถือจริง ๆ นั้นสั้นมาก (แค่ไม่กี่ nanosecond ถึง microsecond สำหรับการอ่าน/เขียนตัวเลขหรือ push ข้อมูลเข้า
`Vec`) จึงแทบไม่มีโอกาสที่ task อื่นจะต้องรอนานพอจนกระทบ throughput ของระบบเลย และ `std::sync::Mutex`
**มี overhead ต่อการล็อกที่ต่ำกว่า `tokio::sync::Mutex`** (เพราะไม่ต้องมีกลไก register/wake กับ async
executor เลย มันเป็น lock แบบ synchronous ล้วน ๆ ที่เบากว่า)

ในทางกลับกัน ถ้าจำเป็นต้อง **`.await` อะไรบางอย่างขณะยังถือ lock อยู่จริง ๆ** (เช่น เขียนไฟล์แบบ async, ส่งผ่าน
network, หรือรอ lock ตัวอื่นที่เป็น async) นั่นคือสถานการณ์เดียวที่ **ต้อง** เปลี่ยนไปใช้ `tokio::sync::Mutex`
— เพราะการถือ `std::sync::MutexGuard` ข้าม `.await` แบบนั้นคือสถานการณ์อันตรายที่หัวข้อ 50.3-50.4 พิสูจน์ให้
เห็นแล้วว่านำไปสู่ compile error หรือ deadlock ได้จริง

มาดูทั้งสองสถานการณ์เทียบกันในโค้ดเดียว เพื่อให้เห็นการตัดสินใจที่ถูกต้องแบบรูปธรรม:

```rust
use std::sync::{Arc, Mutex as StdMutex};
use tokio::sync::Mutex as TokioMutex;
use tokio::time::{sleep, Duration};

// กรณีที่ 1: critical section สั้นมาก ไม่มี .await ข้างใน -> std::sync::Mutex ใช้ได้ดี แม้อยู่ใน async fn
async fn record_hit(counter: &Arc<StdMutex<u64>>) {
    let mut n = counter.lock().unwrap();
    *n += 1;
} // ปล่อยทันที -- ไม่มี await ระหว่างถือ lock เลย จึงไม่มีทาง block worker thread นาน

// กรณีที่ 2: ต้องถือ lock ข้าม .await จริง (จำลองงาน async ขณะยังบันทึก log อยู่) -> ต้องใช้ tokio::sync::Mutex
async fn append_log(log: &Arc<TokioMutex<Vec<String>>>, line: String) {
    let mut buffer = log.lock().await;
    // จำลองงาน async ที่ต้องทำขณะยังถือ lock (เช่นรอ I/O จริงของปลายทาง)
    sleep(Duration::from_millis(5)).await;
    buffer.push(line);
}

#[tokio::main]
async fn main() {
    let counter = Arc::new(StdMutex::new(0u64));
    let mut handles = vec![];
    for _ in 0..100 {
        let counter = Arc::clone(&counter);
        handles.push(tokio::spawn(async move {
            record_hit(&counter).await;
        }));
    }
    for h in handles {
        h.await.unwrap();
    }
    println!(
        "นับ hit ด้วย std::sync::Mutex ใน async task: {}",
        *counter.lock().unwrap()
    );

    let log = Arc::new(TokioMutex::new(Vec::new()));
    let mut handles2 = vec![];
    for i in 0..5 {
        let log = Arc::clone(&log);
        handles2.push(tokio::spawn(async move {
            append_log(&log, format!("บันทึกที่ {i}")).await;
        }));
    }
    for h in handles2 {
        h.await.unwrap();
    }
    let final_log = log.lock().await;
    println!("จำนวนบรรทัด log ทั้งหมด (ผ่าน tokio::sync::Mutex): {}", final_log.len());
}
```

ผลลัพธ์ (รันจริง):

```
นับ hit ด้วย std::sync::Mutex ใน async task: 100
จำนวนบรรทัด log ทั้งหมด (ผ่าน tokio::sync::Mutex): 5
```

ทั้งสองกรณีให้ผลลัพธ์ถูกต้อง 100% — สังเกตว่า `record_hit` ถูกเรียกจาก async task 100 ตัวพร้อมกันจริง แต่ใช้
`std::sync::Mutex` โดยไม่มีปัญหาอะไรเลย เพราะ **critical section ของมัน (`*n += 1;`) ไม่มี `.await` แม้แต่
จุดเดียว** — lock ถูกถือแค่ชั่วครู่เดียวเท่านั้นแล้วปล่อยทันที ในขณะที่ `append_log` **ต้อง** ใช้
`tokio::sync::Mutex` เพราะ critical section ของมันมี `sleep(...).await` (แทนงาน I/O จริงในโลกจริง เช่นเขียน
ไฟล์แบบ async หรือส่งผ่าน network connection ที่แบ่งกันใช้) อยู่ข้างในขณะยังถือ `buffer` ค้างอยู่

จำกฎนี้ให้แม่น เพราะจะใช้ตัดสินใจซ้ำหลายครั้งตลอดบทที่เหลือ: **ดูที่ critical section ไม่ใช่ดูที่บริบทรอบข้าง
— ถ้าไม่มี `.await` ข้างในระหว่างถือ lock ให้ใช้ `std::sync::Mutex` ต่อไป ถ้ามี ให้เปลี่ยนเป็น
`tokio::sync::Mutex`**

### 50.7 `tokio::sync::RwLock`: Multiple Readers แบบ Async

Part 39 หัวข้อ 39.8 แนะนำ `std::sync::RwLock<T>` สำหรับสถานการณ์ read-heavy (อ่านบ่อยกว่าเขียนมาก ๆ) —
อนุญาตให้หลาย thread ถือ **read lock** พร้อมกันได้ (ผ่าน `.read()`) แต่ **write lock** (ผ่าน `.write()`) ต้อง
ผู้ถือเพียงคนเดียวเท่านั้นและกันทุกคนอื่นออกไปทั้งหมด (ทั้ง reader และ writer อื่น) `tokio::sync::RwLock<T>`
ทำสิ่งเดียวกันเป๊ะสำหรับโลก async — เพียงแค่ `.read()`/`.write()` กลายเป็น `async fn` ที่ต้อง `.await`
(`.read().await`/`.write().await`) และใช้กฎการตัดสินใจเดียวกับหัวข้อ 50.6: ถ้า critical section ของการอ่าน/
เขียนมี `.await` ข้างใน ต้องใช้ `tokio::sync::RwLock` แทน `std::sync::RwLock`

```rust
use std::collections::HashMap;
use std::sync::Arc;
use tokio::sync::RwLock;
use tokio::time::{sleep, Duration, Instant};

#[tokio::main]
async fn main() {
    let start = Instant::now();
    let mut initial = HashMap::new();
    initial.insert("timeout_ms".to_string(), 500u64);
    let config = Arc::new(RwLock::new(initial));

    let mut readers = vec![];
    for id in 1..=4u64 {
        let config = Arc::clone(&config);
        readers.push(tokio::spawn(async move {
            sleep(Duration::from_millis(10 * id)).await;
            let map = config.read().await; // ★ read lock -- หลาย task ถือพร้อมกันได้
            println!(
                "[reader {id}] อ่านค่าที่ {:?}: timeout_ms = {:?}",
                start.elapsed(),
                map.get("timeout_ms")
            );
        }));
    }

    let config_w = Arc::clone(&config);
    let writer = tokio::spawn(async move {
        sleep(Duration::from_millis(25)).await;
        let mut map = config_w.write().await; // ★ write lock -- กันทุกคนอื่นออกทั้งหมด
        println!(
            "[writer] กำลังแก้ไขค่าที่ {:?} (readers อื่นต้องรอจนกว่าจะปล่อย write lock)",
            start.elapsed()
        );
        sleep(Duration::from_millis(50)).await; // จำลองงานที่ใช้เวลาขณะถือ write lock
        *map.get_mut("timeout_ms").unwrap() = 1000;
        println!("[writer] แก้ไขเสร็จที่ {:?}", start.elapsed());
    });

    for r in readers {
        r.await.unwrap();
    }
    writer.await.unwrap();

    let final_map = config.read().await;
    println!("ค่าสุดท้าย: timeout_ms = {:?}", final_map.get("timeout_ms"));
}
```

ผลลัพธ์ (รันจริง — ตัวเลขเวลาอาจขยับเล็กน้อยในแต่ละครั้งที่รันตามธรรมชาติของ scheduler แต่ **ลำดับเหตุการณ์จะ
เหมือนกันเสมอ**):

```
[reader 1] อ่านค่าที่ 11.304153ms: timeout_ms = Some(500)
[reader 2] อ่านค่าที่ 21.474394ms: timeout_ms = Some(500)
[writer] กำลังแก้ไขค่าที่ 26.680496ms (readers อื่นต้องรอจนกว่าจะปล่อย write lock)
[writer] แก้ไขเสร็จที่ 78.446227ms
[reader 4] อ่านค่าที่ 78.507911ms: timeout_ms = Some(1000)
[reader 3] อ่านค่าที่ 78.520016ms: timeout_ms = Some(1000)
ค่าสุดท้าย: timeout_ms = Some(1000)
```

สังเกตให้ชัด: `reader 1` (10ms) และ `reader 2` (20ms) อ่านค่าเก่า (`500`) ได้ทันที **ก่อน**ที่ writer จะเริ่ม
ทำงานที่ 25ms — แต่ `reader 3` (30ms) และ `reader 4` (40ms) ควรจะพร้อมอ่านตั้งแต่ตอนนั้น กลับต้อง**รอจนถึง
~78ms** (จนกว่า writer จะปล่อย write lock) จึงได้อ่านค่า และเห็นค่าใหม่ (`1000`) ทั้งคู่ — นี่คือการพิสูจน์ตรง ๆ
ว่า write lock **กันทุก read lock ออกไปทั้งหมด** ตราบใดที่ยังไม่ถูกปล่อย ตรงกับพฤติกรรมที่ Part 39.8 สอนไว้กับ
`std::sync::RwLock` ทุกประการ เพียงแค่ตอนนี้การ "รอ" ของ reader 3/4 คือการคืนสิทธิ์การรันให้ executor
(`.await`) ไม่ใช่การบล็อก OS thread — task อื่นที่ไม่เกี่ยวข้องกับ `config` เลยยังทำงานคู่ขนานได้สบายในระหว่าง
ที่ reader 3/4 กำลัง "รอ" อยู่

**ข้อควรระวังอีกจุดที่ควรรู้ไว้เกี่ยวกับ `tokio::sync::RwLock`**: มันไม่มี method ให้ "ยกระดับ" จาก read lock
เป็น write lock ตรง ๆ ในที่เดียว (ไม่มี `upgradeable_read()` แบบที่บางไลบรารีภาษาอื่นมี) — ถ้าโค้ดต้องอ่านค่า
ก่อนเพื่อตัดสินใจว่าจะเขียนหรือไม่ (เช่น pattern "get-or-compute" ในหัวข้อแบบฝึกหัดท้ายบท) ต้องปล่อย read lock
ก่อนแล้วขอ write lock ใหม่แยกกันเสมอ ซึ่งเปิดช่องให้เกิด **race ระหว่างสองขั้นตอน** ได้ (task อื่นอาจแซงเข้ามา
เขียนค่าไปแล้วระหว่างที่ปล่อย read lock กับขอ write lock) — ต้อง design ให้ทนต่อสถานการณ์นี้เสมอ (เช่นเช็คซ้ำ
อีกครั้งหลังได้ write lock มาแล้ว ก่อนคำนวณค่าใหม่) นอกจากนี้ `tokio::sync::RwLock` (เหมือนกับ
`std::sync::RwLock` ใน Part 39.8) **ไม่ได้การันตีความเป็นธรรม (fairness) ระหว่าง reader กับ writer อย่างเข้มงวด**
ในทุก platform — ถ้ามี reader มาขอ read lock ถี่มากอย่างต่อเนื่องไม่หยุด writer ที่รออยู่อาจถูก "แซง" ไปเรื่อย ๆ
(เรียกว่า **writer starvation**) แม้ implementation ของ Tokio จะพยายามลดปัญหานี้ให้น้อยที่สุดก็ตาม ถ้าระบบมีการ
เขียนที่สำคัญมากและต้องมั่นใจว่าจะได้ทำงานภายในเวลาที่คาดการณ์ได้ ควรพิจารณาออกแบบให้จำกัดความถี่ของการอ่านหรือ
ใช้เครื่องมืออื่นเสริม (เช่น `Semaphore` ควบคุมจำนวน reader พร้อมกัน) แทนการพึ่ง `RwLock` เพียงอย่างเดียว

### 50.8 `tokio::sync::mpsc`: Async Channel กับ Backpressure แบบไม่บล็อก Thread

Part 38 สอน `std::sync::mpsc` ในฐานะเครื่องมือหลักของ message-passing concurrency — `tx.send(value)`,
`rx.recv()`, multiple producer ผ่าน `.clone()`, และ bounded channel (`mpsc::sync_channel`) ที่ทำ
**backpressure**: ถ้า channel เต็ม `.send()` จะ**บล็อก** thread ผู้ส่งจนกว่า consumer จะรับข้อความบางส่วนไปก่อน

`tokio::sync::mpsc` ทำสิ่งเดียวกันเป๊ะสำหรับโลก async: `tx.send(value).await` และ `rx.recv().await` เป็น
`async fn` ทั้งคู่ — ความต่างที่สำคัญที่สุดคือ **`tokio::sync::mpsc::channel(capacity)` เป็น bounded channel
เสมอ** (ต้องระบุ capacity ตอนสร้างเสมอ ไม่มีเวอร์ชัน unbounded ให้เขียนเผลอลืม cap แบบ `std::sync::mpsc::channel()`
— ถ้าต้องการ unbounded จริง ๆ ต้องเรียก `tokio::sync::mpsc::unbounded_channel()` แยกชื่อไปเลยเพื่อให้เห็นชัดใน
โค้ดว่าตั้งใจไม่จำกัด) และเมื่อ channel เต็ม `.send().await` จะ **คืนสิทธิ์การรันให้ executor** ระหว่างรอที่ว่าง
แทนการบล็อก OS thread — นี่คือ **backpressure แบบ async-native**: producer ที่เร็วเกินไปจะถูกหน่วงให้ช้าลง
โดยอัตโนมัติเหมือนเดิม แต่ไม่มี worker thread ไหนถูกยึดไว้เฉย ๆ ระหว่างที่รอเลย

มารีเมค worker pool จาก Part 38 (หัวข้อ 38.8 ที่ใช้ `Arc<Mutex<Receiver<T>>>` ให้ worker หลาย thread แบ่ง
`Receiver` ตัวเดียวกัน) ให้เป็นเวอร์ชัน async — และนี่คือจุดที่กฎการตัดสินใจจากหัวข้อ 50.6 กลับมาใช้งานจริงแบบ
เป๊ะที่สุด: เราต้องใช้ **`tokio::sync::Mutex`** (ไม่ใช่ `std::sync::Mutex`) มาห่อ `Receiver` เพราะ critical
section ที่จะแบ่ง `Receiver` กันใช้นั้น**มี `.recv().await` อยู่ข้างในตรง ๆ**:

```rust
use std::sync::Arc;
use tokio::sync::{mpsc, Mutex as TokioMutex};
use tokio::time::{sleep, Duration};

struct Job {
    id: u32,
    payload: i32,
    work_ms: u64,
}

async fn worker(id: usize, rx: Arc<TokioMutex<mpsc::Receiver<Job>>>) {
    loop {
        // ★ ต้องใช้ tokio::sync::Mutex เพราะ .recv().await อยู่ข้างในขณะยังถือ guard
        let job = {
            let mut guard = rx.lock().await;
            guard.recv().await
        }; // ปล่อย lock ทันทีหลังได้ job (หรือ None) มา -- ก่อนเริ่มประมวลผลจริง

        match job {
            Some(job) => {
                println!("[worker {id}] กำลังประมวลผล job #{}", job.id);
                sleep(Duration::from_millis(job.work_ms)).await;
                println!(
                    "[worker {id}] เสร็จ job #{} -> ผลลัพธ์ = {}",
                    job.id,
                    job.payload * 2
                );
            }
            None => {
                println!("[worker {id}] channel ปิดแล้ว เลิกทำงาน");
                break;
            }
        }
    }
}

#[tokio::main]
async fn main() {
    let (tx, rx) = mpsc::channel::<Job>(4); // bounded -> genuine backpressure
    let rx = Arc::new(TokioMutex::new(rx));

    let mut worker_handles = vec![];
    for id in 1..=3 {
        let rx = Arc::clone(&rx);
        worker_handles.push(tokio::spawn(worker(id, rx)));
    }

    for i in 1..=9u32 {
        let job = Job {
            id: i,
            payload: i as i32,
            work_ms: 50,
        };
        println!("[producer] กำลังส่ง job #{i} (ถ้า channel เต็ม .send().await จะรอที่นี่)");
        tx.send(job).await.unwrap();
    }
    drop(tx); // ปิด channel -> workers จะได้ None แล้วเลิกงาน

    for h in worker_handles {
        h.await.unwrap();
    }
    println!("[main] ประมวลผลครบทุก job แล้ว");
}
```

ผลลัพธ์ (รันจริง — ลำดับการสลับของ worker แต่ละตัวไม่แน่นอนเหมือนกับ multiple producer ใน Part 38.5 แต่การันตี
ว่าทุก job ถูกประมวลผลครบและ backpressure ทำงานจริง):

```
[producer] กำลังส่ง job #1 (ถ้า channel เต็ม .send().await จะรอที่นี่)
[producer] กำลังส่ง job #2 (ถ้า channel เต็ม .send().await จะรอที่นี่)
[producer] กำลังส่ง job #3 (ถ้า channel เต็ม .send().await จะรอที่นี่)
[producer] กำลังส่ง job #4 (ถ้า channel เต็ม .send().await จะรอที่นี่)
[producer] กำลังส่ง job #5 (ถ้า channel เต็ม .send().await จะรอที่นี่)
[worker 1] กำลังประมวลผล job #1
[producer] กำลังส่ง job #6 (ถ้า channel เต็ม .send().await จะรอที่นี่)
[worker 2] กำลังประมวลผล job #2
[producer] กำลังส่ง job #7 (ถ้า channel เต็ม .send().await จะรอที่นี่)
[worker 3] กำลังประมวลผล job #3
[producer] กำลังส่ง job #8 (ถ้า channel เต็ม .send().await จะรอที่นี่)
[worker 3] เสร็จ job #3 -> ผลลัพธ์ = 6
[worker 3] กำลังประมวลผล job #4
...
[worker 1] channel ปิดแล้ว เลิกทำงาน
[worker 2] channel ปิดแล้ว เลิกทำงาน
[worker 3] channel ปิดแล้ว เลิกทำงาน
[main] ประมวลผลครบทุก job แล้ว
```

(ผลลัพธ์ตัดบางบรรทัดกลางเพื่อความกระชับ — รันจริงได้ครบทั้ง 9 job เสมอ) สังเกตว่า producer ส่ง job #1-5 ได้
ทันทีตั้งแต่ต้น (เพราะ 3 worker เริ่มดึงงานออกจาก channel เกือบจะพร้อมกับที่ producer เริ่มส่ง ทำให้มีที่ว่างใน
บัฟเฟอร์ขนาด 4 อยู่เรื่อย ๆ) แต่หลังจากนั้น producer เริ่ม **สลับกับ** worker ที่กำลังประมวลผล — นี่คือ
backpressure ที่ทำงานจริง: ถ้า worker ประมวลผลช้ากว่าที่ producer ส่ง (ในตัวอย่างนี้ `work_ms: 50` ต่อ job)
`.send().await` ของ producer จะเริ่ม "รอ" (คืนสิทธิ์การรันให้ executor) ทุกครั้งที่บัฟเฟอร์ 4 ช่องเต็มพอดี ไม่มี
job ไหนสะสมอยู่ใน memory แบบไม่มีเพดานเหมือนที่ Part 38.7 เตือนไว้เรื่อง unbounded channel เลย

จุดที่ควรสังเกตให้ดีเป็นพิเศษคือ **บล็อก `{ let mut guard = rx.lock().await; guard.recv().await }`** — เรา
จงใจจำกัด scope ของ `guard` ให้เล็กที่สุดด้วย block `{}` (เทคนิคเดียวกับ `good_pattern` ใน Part 39.4) เพื่อให้
lock ถูกปล่อย**ทันที**หลังจากได้ `job` (หรือ `None`) มา **ก่อน**ที่จะเริ่ม `sleep(job.work_ms).await` ที่จำลอง
งานหนัก — ถ้าลืมจำกัด scope แบบนี้ (ถือ `guard` ค้างไว้ตลอดการประมวลผล job) worker ตัวอื่นจะแบ่ง `Receiver`
กันใช้ไม่ได้เลย เพราะต้องรอ worker ตัวแรกประมวลผล job เสร็จก่อนเสมอ ทำให้ worker pool ทำงานแบบ **serial** (ทีละ
ตัว) ทั้งที่ตั้งใจให้ทำงานคู่ขนานกัน 3 ตัว — นี่คือตัวอย่างที่จับต้องได้ของกฎ "critical section ให้สั้นที่สุด"
จาก Part 39.4 ที่ยังคงสำคัญเท่าเดิมในโลก async

**สำหรับกรณีที่ต้องการ channel แบบไม่จำกัดขนาดจริง ๆ** (ยอมรับความเสี่ยงเรื่อง memory ไม่มีเพดานตามที่ Part
38.7 เตือนไว้ เพื่อแลกกับการไม่มีวันบล็อกฝั่งส่งเลย) `tokio::sync::mpsc` มีฟังก์ชันแยกชื่อไปเลยคือ
`mpsc::unbounded_channel()` — สังเกตว่า `UnboundedSender::send()` **ไม่ต้อง `.await`** (ต่างจาก
`Sender::send()` ของ bounded channel) เพราะไม่มีทางที่การส่งจะ "ต้องรอ" ได้เลยไม่ว่ากรณีใด:

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::unbounded_channel::<i32>();

    // unbounded_send ไม่ต้อง .await เลย -- ไม่มีบัฟเฟอร์ให้เต็ม จึงไม่มีทางต้องรอ (แลกกับไม่มี backpressure)
    for i in 1..=5 {
        tx.send(i).unwrap();
    }
    drop(tx);

    let mut total = 0;
    while let Some(v) = rx.recv().await {
        total += v;
    }
    println!("รวมค่าทั้งหมด: {total}");
}
```

ผลลัพธ์ (รันจริง):

```
รวมค่าทั้งหมด: 15
```

การที่ชื่อฟังก์ชันต้องเขียนยาวขึ้นเป็น `unbounded_channel()` (ไม่ใช่แค่ `channel()` โดยไม่ระบุ capacity แบบ
`std::sync::mpsc::channel()` ใน Part 38.2) เป็นการออกแบบที่ตั้งใจ: **บังคับให้ผู้เขียนโค้ดต้องเลือกอย่างชัดเจน
ว่าจะรับความเสี่ยงเรื่อง unbounded memory หรือไม่ ไม่ให้เผลอใช้แบบไม่จำกัดขนาดไปโดยไม่ตั้งใจ** เหมือนที่อาจ
เกิดขึ้นได้ง่ายกับ `std::sync::mpsc::channel()` ที่ไม่มี bound เป็นค่าเริ่มต้น

### 50.9 `tokio::sync::oneshot`: ถาม-ตอบครั้งเดียวระหว่าง Task

Part 38 หัวข้อท้ายบททิ้งคำใบ้ไว้ตรง ๆ ว่า pattern "ฝัง reply channel ไว้ในข้อความที่ส่ง" (ให้ผู้รับตอบกลับหา
ผู้ส่งเจาะจงคนเดียวได้ ไม่ใช่ตอบกลับผ่าน channel รวมที่ทุกคนแย่งกันอ่าน) มีชื่อเรียกว่า **"oneshot channel"**
เพราะแต่ละ channel ถูกใช้ตอบกลับแค่ครั้งเดียวแล้วทิ้งไปเลย และ `tokio` มี type ที่ออกแบบมาสำหรับ pattern นี้
โดยเฉพาะ — บทนี้คือจุดที่คำใบ้นั้นถูกอธิบายเต็มรูปแบบพร้อมโค้ดจริง

`tokio::sync::oneshot::channel::<T>()` คืน `(Sender<T>, Receiver<T>)` เหมือน `mpsc` แต่มีข้อจำกัดที่ตั้งใจให้
เข้มงวดกว่ามาก: **ส่งได้แค่ครั้งเดียวเท่านั้น** — `Sender<T>::send(self, value: T)` รับ `self` โดย value (ไม่ใช่
`&self`) ทำให้เรียกซ้ำสองครั้งไม่ได้เลยในระดับ type system (เรียกครั้งแรก `self` ก็ถูก move ไปกินแล้ว) และ
**ไม่ implement `Clone`** ทั้ง `Sender` และ `Receiver` — สื่อความหมายผ่าน type ตรง ๆ ว่านี่คือ "ที่อยู่ตอบกลับ
แบบใช้แล้วทิ้ง" ไม่ใช่ channel ทั่วไปที่ใครจะ subscribe เพิ่มก็ได้

Pattern ที่ใช้ `oneshot` บ่อยที่สุดคือ **"actor pattern"**: มี task หนึ่งตัวเป็นเจ้าของข้อมูลบางอย่างแต่เพียง
ผู้เดียว (ไม่ต้องใช้ `Mutex` เลย เพราะไม่มีใครแบ่งข้อมูลนั้นตรง ๆ — สอดคล้องกับปรัชญา message passing ของ Part
38.1) ส่วน task อื่นที่ต้องการ "ถามคำถาม" กับมันจะส่ง request ผ่าน `mpsc` (เพราะมีคนถามได้หลายคน) โดย **ฝัง
`oneshot::Sender` ของตัวเองไว้ในข้อความ request นั้นด้วย** เพื่อให้ actor ตอบกลับหาคนที่ถามคนนั้นได้เจาะจง:

```rust
use std::collections::HashMap;
use tokio::sync::{mpsc, oneshot};

struct Request {
    query: String,
    respond_to: oneshot::Sender<String>, // ★ "ที่อยู่ตอบกลับ" เฉพาะของ request นี้เท่านั้น
}

async fn database_actor(mut rx: mpsc::Receiver<Request>) {
    let mut store: HashMap<String, String> = HashMap::new();
    store.insert("alice".to_string(), "1000 บาท".to_string());
    store.insert("bob".to_string(), "500 บาท".to_string());

    while let Some(req) = rx.recv().await {
        let answer = store
            .get(&req.query)
            .cloned()
            .unwrap_or_else(|| "ไม่พบข้อมูล".to_string());
        // ส่งคำตอบกลับไปยังผู้ถามคนนี้เท่านั้น ผ่าน oneshot -- ใช้ได้ครั้งเดียวแล้วทิ้ง
        let _ = req.respond_to.send(answer);
    }
    println!("[database_actor] ไม่มีผู้ส่งคำขอเหลือแล้ว จบการทำงาน");
}

async fn query(tx: &mpsc::Sender<Request>, name: &str) -> String {
    let (respond_to, response) = oneshot::channel();
    tx.send(Request {
        query: name.to_string(),
        respond_to,
    })
    .await
    .unwrap();
    response
        .await
        .unwrap_or_else(|_| "ผู้ตอบหายไปก่อนตอบ".to_string())
}

#[tokio::main]
async fn main() {
    let (tx, rx) = mpsc::channel(8);
    let actor_handle = tokio::spawn(database_actor(rx));

    let names = ["alice", "bob", "charlie"];
    let mut handles = vec![];
    for name in names {
        let tx = tx.clone();
        handles.push(tokio::spawn(async move {
            let result = query(&tx, name).await;
            println!("ยอดของ {name}: {result}");
        }));
    }
    drop(tx);
    for h in handles {
        h.await.unwrap();
    }
    actor_handle.await.unwrap();
}
```

ผลลัพธ์ (รันจริง):

```
ยอดของ alice: 1000 บาท
ยอดของ bob: 500 บาท
ยอดของ charlie: ไม่พบข้อมูล
[database_actor] ไม่มีผู้ส่งคำขอเหลือแล้ว จบการทำงาน
```

**อธิบายจุดสำคัญ:**

- `database_actor` เป็นเจ้าของ `store: HashMap<...>` แต่เพียงผู้เดียวตลอดทั้งโปรแกรม ไม่มี `Mutex` ห่อมันเลย —
  เพราะไม่มี task อื่นเข้าถึง `store` ตรง ๆ ได้เลย ทุกการเข้าถึงต้อง "ถาม" ผ่าน message เท่านั้น (ปรัชญา
  message-passing เต็มรูปแบบตาม Part 38.1) นี่คือทางเลือกที่มักสะอาดกว่า `Arc<Mutex<HashMap<...>>>` ในหลาย
  สถานการณ์ เพราะไม่มีความเสี่ยง deadlock จาก lock เลยแม้แต่นิดเดียว (ไม่มี lock ให้ deadlock ตั้งแต่แรก)
- ทุกครั้งที่ `query()` ถูกเรียก มันสร้าง `oneshot::channel()` **ใหม่** ขึ้นมาเฉพาะสำหรับ request ครั้งนั้นครั้ง
  เดียว — เปรียบเหมือนการเขียนซองจดหมายพร้อมที่อยู่ตอบกลับที่ใช้ครั้งเดียวแล้วทิ้งแนบไปกับคำถามทุกครั้ง
- `response.await` — ฝั่งผู้ถามรอคำตอบด้วยการ `.await` ตรง ๆ บน `oneshot::Receiver<T>` (มัน implement
  `Future<Output = Result<T, RecvError>>` ให้มาโดยตรง) — `Err` เกิดขึ้นก็ต่อเมื่อฝั่ง `Sender` ถูก drop ไปโดยไม่
  ได้ `.send()` เลย (เช่น actor panic หรือปิดตัวไปก่อนตอบ) ซึ่งจัดการด้วย `.unwrap_or_else(...)` ได้ตรงไปตรงมา
  ตามหลัก error handling จาก Part 12
- สาม request (alice, bob, charlie) วิ่งเข้า `database_actor` **ผ่าน `mpsc` เดียวกัน** (multiple producer จาก
  Part 38.5) แต่คำตอบของแต่ละคนกลับไปหา**เฉพาะ task ที่ถามคนนั้น**เท่านั้น ไม่มีทางที่ task ของ `alice` จะได้
  คำตอบของ `bob` ไปผิดตัวได้เลย เพราะ type system รับประกันไว้ตั้งแต่ตอน compile ว่า `oneshot::Receiver<String>`
  ตัวหนึ่งผูกกับ `oneshot::Sender<String>` ที่จับคู่กันมาตัวเดียวเท่านั้น

### 50.10 `tokio::sync::broadcast`: หนึ่งข้อความ หลาย Receiver

นี่คือเครื่องมือที่จะแก้ปัญหาที่ Part 49 ร่างไว้แบบง่าย ๆ ตอนสร้าง chat server: ตอนนั้นเรายังไม่มีเครื่องมือ
async ที่เหมาะสมสำหรับ "กระจายข้อความหนึ่งชิ้นให้ทุก client ที่เชื่อมต่ออยู่พร้อมกัน" จึงต้อง**ยืม** `Mutex`
(Part 39) กับ `mpsc::Sender` (Part 38) มาประกอบกันเป็น `Arc<Mutex<Vec<Sender<String>>>>` — เก็บ list ของ
sender ของทุก client ไว้ในโครงสร้างเดียว แล้วเวลาจะส่งข้อความให้ทุกคน ต้องล็อก list นั้น วนส่งให้ทุก sender
ทีละตัว และคอยลบ sender ของ client ที่หลุดการเชื่อมต่อไปแล้วออกเองด้วย

มาดูโค้ดจริงของแนวทางแบบนั้นก่อน เพื่อเห็นภาระที่มันสร้างขึ้นชัด ๆ:

```rust
use std::sync::Arc;
use tokio::sync::{mpsc, Mutex as TokioMutex};
use tokio::time::{sleep, Duration};

type Subscribers = Arc<TokioMutex<Vec<mpsc::Sender<String>>>>;

async fn broadcast_to_all(subs: &Subscribers, msg: String) {
    let mut list = subs.lock().await;
    // ต้องวน .send().await ทีละคนเอง เขียน error handling เอง และลบคนที่ channel ปิดไปแล้วเอง
    let mut i = 0;
    while i < list.len() {
        if list[i].send(msg.clone()).await.is_err() {
            list.remove(i); // ผู้รับหลุดไปแล้ว -- ต้องจัดการลบเอง ไม่มีใครทำให้
        } else {
            i += 1;
        }
    }
}

#[tokio::main]
async fn main() {
    let subs: Subscribers = Arc::new(TokioMutex::new(Vec::new()));

    let mut readers = vec![];
    for id in 1..=3 {
        let (tx, mut rx) = mpsc::channel::<String>(8);
        subs.lock().await.push(tx);
        readers.push(tokio::spawn(async move {
            while let Some(msg) = rx.recv().await {
                println!("[reader {id}] ได้รับ: {msg}");
            }
        }));
    }

    broadcast_to_all(&subs, "สวัสดีทุกคน".to_string()).await;
    sleep(Duration::from_millis(20)).await;

    // ต้องปิดทุก subscriber ด้วยการเคลียร์ list เอง (drop sender ทั้งหมดด้วยมือ)
    subs.lock().await.clear();
    for r in readers {
        r.await.unwrap();
    }
    println!("จบตัวอย่างแบบเก่า: ต้องจัดการ list ของ Sender เอง, ต้องล็อกทุกครั้งที่ broadcast, ต้องลบคนหลุดเอง");
}
```

ผลลัพธ์ (รันจริง):

```
[reader 1] ได้รับ: สวัสดีทุกคน
[reader 3] ได้รับ: สวัสดีทุกคน
[reader 2] ได้รับ: สวัสดีทุกคน
จบตัวอย่างแบบเก่า: ต้องจัดการ list ของ Sender เอง, ต้องล็อกทุกครั้งที่ broadcast, ต้องลบคนหลุดเอง
```

โค้ดนี้ทำงานได้จริง (และในหัวข้อ decision rule ของบทนี้ก็ยืนยันแล้วว่าเหตุผลที่ต้องใช้ `tokio::sync::Mutex`
แทน `std::sync::Mutex` ตรงนี้ถูกต้อง เพราะ critical section มี `.send().await` ข้างใน) แต่สังเกตภาระที่ผู้เขียน
โค้ดต้องแบกไว้เอง: ต้องล็อก list ทุกครั้งที่จะ broadcast, ต้องวน loop ส่งให้ทุกคนเอง, ต้องจัดการ error/ลบคนที่
หลุดการเชื่อมต่อไปแล้วเอง, และต้องคิดเรื่อง critical section/`.await` ให้ถูกต้องเองทุกจุด — งานเหล่านี้**ไม่ใช่
ตรรกะทางธุรกิจของ chat server เลย มันเป็นแค่ "งานท่อประปา" (plumbing) ที่ควรมีเครื่องมือสำเร็จรูปทำให้**

`tokio::sync::broadcast` คือเครื่องมือสำเร็จรูปนั้น: `broadcast::channel::<T>(capacity)` คืน `(Sender<T>,
Receiver<T>)` เหมือน `mpsc` แต่มีความหมายต่างไปจากเดิมโดยสิ้นเชิง — **แต่ละ `Receiver` ที่ subscribe ไว้จะได้
รับสำเนาของ "ทุกข้อความที่ส่งหลังจากตัวเอง subscribe" ครบทุกข้อความ ไม่ใช่แค่ใครมาถึงก่อนได้ไป (แบบ `mpsc`)**
— `Sender<T>` implement `Clone` **และ** มี method `.subscribe()` ที่สร้าง `Receiver<T>` ตัวใหม่ได้ทุกเมื่อ
โดยไม่ต้องล็อกอะไรเองเลย (กลไก fan-out ภายในของ `broadcast` จัดการให้ทั้งหมด):

```rust
use tokio::sync::broadcast;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let (tx, _rx) = broadcast::channel::<String>(16);

    let mut readers = vec![];
    for id in 1..=3 {
        let mut rx = tx.subscribe(); // สมัครสมาชิกใหม่เมื่อไหร่ก็ได้ ไม่ต้องล็อก list เอง
        readers.push(tokio::spawn(async move {
            while let Ok(msg) = rx.recv().await {
                println!("[reader {id}] ได้รับ: {msg}");
            }
            println!("[reader {id}] sender ปิดแล้ว เลิกฟัง");
        }));
    }

    // ส่งครั้งเดียว -- ทุกคนที่ subscribe ไว้ได้สำเนาของตัวเอง ไม่ต้องล็อก ไม่ต้อง loop เอง
    tx.send("สวัสดีทุกคน".to_string()).unwrap();
    sleep(Duration::from_millis(20)).await;

    drop(tx); // ปิด channel -> ทุก receiver จะได้ Err(Closed) แล้วเลิก loop เอง
    for r in readers {
        r.await.unwrap();
    }
}
```

ผลลัพธ์ (รันจริง):

```
[reader 1] ได้รับ: สวัสดีทุกคน
[reader 2] ได้รับ: สวัสดีทุกคน
[reader 3] ได้รับ: สวัสดีทุกคน
[reader 1] sender ปิดแล้ว เลิกฟัง
[reader 2] sender ปิดแล้ว เลิกฟัง
[reader 3] sender ปิดแล้ว เลิกฟัง
```

ทั้งสามคนได้รับข้อความ **"สวัสดีทุกคน"** ครบทุกคน (นี่คือความหมายของคำว่า "broadcast" — ต่างจาก `mpsc` ที่ถ้ามี
หลาย receiver แย่งกันอ่าน channel เดียวกัน ข้อความหนึ่งชิ้นจะไปถึงแค่คนเดียวที่แย่งได้ก่อนเท่านั้น) และสังเกตว่า
**เราไม่ต้องเขียนโค้ดล็อกอะไรเองเลยแม้แต่บรรทัดเดียว** — เทียบกับตัวอย่างแบบเก่าที่ต้องเขียน `Arc<Mutex<Vec<...>>>`
กับ loop จัดการเองทั้งหมด `tx.send()` ตัวเดียวก็ทำหน้าที่กระจายให้ทุกคนที่ subscribe ไว้เรียบร้อยแล้ว — นี่คือ
เหตุผลที่ `tokio::sync::broadcast` เป็นเครื่องมือระดับ production ที่ถูกต้องสำหรับ pattern นี้ ไม่ใช่แค่
"ทางลัดที่สะดวกกว่า" แต่เป็น **การใช้ type ที่ตรงกับความหมายของปัญหาโดยตรง** ตั้งแต่การออกแบบ

**สิ่งที่ควรรู้เพิ่มเติมเกี่ยวกับ `broadcast::Receiver::recv()`** — มันคืน
`Result<T, broadcast::error::RecvError>` ที่มีสอง variant สำคัญ (ต่างจาก `mpsc` ที่มีแค่ "ปิดแล้วหรือยัง"):

- **`RecvError::Closed`** — sender ทุกตัวถูก drop แล้ว (ความหมายเดียวกับ `mpsc` ที่เคยเรียนมา)
- **`RecvError::Lagged(n)`** — receiver ตัวนี้ **อ่านไม่ทัน** ข้อความที่ถูกส่งมาเร็วเกินไป จน buffer ภายใน
  (ขนาดตาม `capacity` ที่ระบุตอนสร้าง channel) ล้นและ**ข้อความเก่าที่สุด `n` ข้อความถูกทิ้งไปแล้ว** — นี่คือ
  trade-off ที่ `broadcast` เลือก: มันไม่ยอมให้ผู้ส่งต้องรอ (ไม่มี backpressure แบบ `mpsc` bounded) แต่ยอมทิ้ง
  ข้อความเก่าของ receiver ที่ตามไม่ทันแทน — เหมาะกับสถานการณ์แบบ chat/log ที่ "พลาดข้อความเก่าไปบ้างยังพอรับได้
  ดีกว่าทำให้ทุกคนช้าลงเพราะรอคนช้าที่สุด" โค้ด production ที่ดีควร `match` แยก `Lagged` ออกมาต่างหาก (แจ้งเตือน
  หรือ log ไว้) แทนการมองว่ามันเป็น error ทั่วไปเหมือน `Closed`

### 50.11 `tokio::sync::watch`: เก็บแค่ค่าล่าสุด

`tokio::sync::watch` แก้ปัญหาที่ต่างจาก `broadcast` ตรงจุดสำคัญจุดเดียว: **`watch` เก็บแค่ค่าล่าสุดเพียงค่า
เดียว ไม่ใช่ประวัติทุกข้อความ** — ถ้าผู้ส่งส่งค่าใหม่หลายครั้งติดกันเร็ว ๆ ก่อนที่ผู้รับจะทันอ่าน ผู้รับจะเห็น
**แค่ค่าล่าสุดค่าเดียว** ค่ากลาง ๆ ที่ถูกส่งมาก่อนหน้าจะถูก "เขียนทับ" หายไปเลย ไม่มีสิ่งที่เรียกว่า "ตกข้อความ"
แบบ `Lagged` ของ `broadcast` เพราะโดยธรรมชาติของ `watch` แล้ว **ไม่มีใครสัญญาว่าจะให้เห็นค่าทุกค่าที่เคยส่งไป
ตั้งแต่ต้น** มันสัญญาแค่ว่า "คุณจะเห็นค่าที่ใหม่ที่สุดเสมอเมื่อคุณพร้อมอ่าน"

นี่คือเครื่องมือที่เหมาะกับสถานการณ์ **configuration update / state broadcast** ที่ผู้รับสนใจแค่ **สถานะปัจจุบัน**
ไม่สนใจ "ประวัติการเปลี่ยนแปลง" เลย เช่น "ค่าจำกัดจำนวน connection ตอนนี้คือเท่าไหร่" — ถ้าค่าถูกปรับ 3 ครั้ง
ติดกันเร็ว ๆ ก่อนที่ทุก task จะทันอ่าน สิ่งที่ทุก task ต้องการรู้จริง ๆ คือ **ค่าสุดท้าย** เท่านั้น ไม่ใช่ค่ากลาง
ทางที่ผ่านมา:

```rust
use tokio::sync::watch;
use tokio::time::{sleep, Duration};

#[derive(Debug, Clone)]
struct Config {
    max_connections: u32,
}

async fn watcher(id: &'static str, mut rx: watch::Receiver<Config>) {
    loop {
        // .changed() รอจนกว่าค่าจะเปลี่ยนจากครั้งก่อนที่ตัวเองอ่าน (ไม่ใช่ "ทุกครั้งที่ส่ง")
        if rx.changed().await.is_err() {
            println!("[{id}] ผู้ส่งปิดแล้ว เลิกฟัง");
            break;
        }
        let cfg = rx.borrow().clone();
        println!("[{id}] เห็นค่าใหม่: max_connections = {}", cfg.max_connections);
    }
}

#[tokio::main]
async fn main() {
    let (tx, rx1) = watch::channel(Config {
        max_connections: 100,
    });
    let rx2 = tx.subscribe();

    let h1 = tokio::spawn(watcher("watcher-1", rx1));
    let h2 = tokio::spawn(watcher("watcher-2", rx2));

    sleep(Duration::from_millis(30)).await;
    // ส่งค่าใหม่ 3 ครั้งติดกันเร็ว ๆ โดยไม่มี .await คั่นเลย -- watcher ที่ยังไม่ทันอ่านจะเห็นแค่ค่า "ล่าสุด" เท่านั้น
    for n in [200, 150, 300] {
        tx.send(Config { max_connections: n }).unwrap();
    }

    sleep(Duration::from_millis(50)).await;
    drop(tx);
    let _ = tokio::join!(h1, h2);
}
```

ผลลัพธ์ (รันจริง):

```
[watcher-2] เห็นค่าใหม่: max_connections = 300
[watcher-1] เห็นค่าใหม่: max_connections = 300
[watcher-2] ผู้ส่งปิดแล้ว เลิกฟัง
[watcher-1] ผู้ส่งปิดแล้ว เลิกฟัง
```

นี่คือจุดที่ต้องสังเกตให้ทะลุ: เราส่งค่าไป **3 ค่า** (200, 150, 300) แต่ทั้ง `watcher-1` และ `watcher-2` พิมพ์
บรรทัด **"เห็นค่าใหม่"" แค่ครั้งเดียว** และเห็นแค่ **300** (ค่าสุดท้าย) เท่านั้น — ค่า 200 และ 150 **ไม่ปรากฏ
เลยแม้แต่บรรทัดเดียว** เพราะทั้งสาม `tx.send()` ถูกเรียกติดกันโดยไม่มี `.await` คั่นระหว่างกลาง (จึงไม่มีโอกาส
ให้ executor สลับไปปลุก `watcher-1`/`watcher-2` ขึ้นมาอ่านค่าระหว่างทางเลย) พอ `watcher` ทั้งสองตัวถูกปลุกขึ้นมา
จริง ๆ (หลัง `sleep(30ms)` ของ main จบ) สิ่งที่มันเห็นในช่อง `watch` คือ **ค่าที่เขียนทับล่าสุดเท่านั้น** — นี่
คือความหมายตรงตัวของชื่อ "watch": มันคือหน้าต่างที่มองเห็น **สถานะปัจจุบัน** ของสิ่งที่กำลัง "เฝ้ามอง" อยู่ ไม่ใช่
inbox ที่เก็บทุกข้อความที่เคยส่งผ่านมาแบบ `mpsc`/`broadcast`

### 50.12 `tokio::sync::Semaphore`: จำกัดจำนวนงานที่ทำพร้อมกัน

`Semaphore` (ตัวนับสัญญาณ) เป็นเครื่องมือคลาสสิกจากทฤษฎี concurrency ทั่วไป (ไม่ได้เกิดกับ Rust หรือ Tokio
เป็นครั้งแรก — มีมาตั้งแต่ยุค operating system เริ่มต้น) ที่ `tokio::sync` นำมาทำเป็นเวอร์ชัน async: มันคือ
"ตั๋วจำนวนจำกัด" (`permits`) จำนวน N ใบ — งานไหนต้องการทำงานต้องขอตั๋วก่อนด้วย `.acquire().await` ถ้าตั๋วยัง
เหลืออยู่ก็ได้ไปทันที ถ้าตั๋วหมด `.acquire().await` จะ **คืนสิทธิ์การรันให้ executor** (ไม่บล็อก thread) จนกว่า
จะมีใครคืนตั๋วกลับมา — เมื่อคืนแล้วงานที่รออยู่ตัวถัดไปจะได้ตั๋วไปทำงานต่อ

สังเกตว่า `Semaphore` **ไม่ได้ป้องกัน data race แบบ `Mutex`** — มันไม่ได้ห่อข้อมูลอะไรไว้ข้างในเลย มันแค่จำกัด
**จำนวนงานที่ทำพร้อมกันได้สูงสุด** เท่านั้น (เช่น "connection ไปยัง database พร้อมกันได้ไม่เกิน 10", "เรียก
API ภายนอกพร้อมกันได้ไม่เกิน 5 ครั้ง" เพื่อไม่ให้ปลายทางล้น) — ใช้งานร่วมกับ `Mutex`/`RwLock` ได้แต่แก้ปัญหา
คนละแบบกัน

`Semaphore::acquire()` คืน `SemaphorePermit<'_>` (หรือ `OwnedSemaphorePermit` ถ้าใช้ `.acquire_owned()` กับ
`Arc<Semaphore>` เพื่อให้ permit เป็นเจ้าของ `Arc` ของตัวเอง ย้ายข้าม task ได้สะดวกกว่า) ที่ทำงานแบบ **RAII
เหมือนกับ `MutexGuard` ทุกประการ** (เชื่อมกับ `Drop` trait จาก Part 6) — เมื่อ permit หลุด scope มันจะ **คืน
ตั๋วกลับให้ semaphore โดยอัตโนมัติ** ไม่ต้องเรียกอะไรเองเลย:

```rust
use std::sync::Arc;
use tokio::sync::Semaphore;
use tokio::time::{sleep, Duration};

async fn fetch_resource(id: u32, semaphore: Arc<Semaphore>) {
    let _permit = semaphore.acquire().await.unwrap(); // รอ "ตั๋ว" ถ้าตอนนี้เต็มโควต้าแล้ว
    println!(
        "[request {id}] ได้ตั๋วแล้ว กำลังทำงาน... (เหลือตั๋วว่าง: {})",
        semaphore.available_permits()
    );
    sleep(Duration::from_millis(100)).await; // จำลอง network call
    println!("[request {id}] ทำงานเสร็จ กำลังคืนตั๋ว");
} // _permit หลุด scope ที่นี่ -> Drop -> คืนตั๋วให้ semaphore โดยอัตโนมัติ (RAII, Part 6)

#[tokio::main]
async fn main() {
    let semaphore = Arc::new(Semaphore::new(3)); // อนุญาตแค่ 3 คำขอพร้อมกันสูงสุด
    let mut handles = vec![];
    for id in 1..=8u32 {
        let semaphore = Arc::clone(&semaphore);
        handles.push(tokio::spawn(fetch_resource(id, semaphore)));
    }
    for h in handles {
        h.await.unwrap();
    }
    println!("ทำงานครบ 8 คำขอแล้ว (ไม่เกิน 3 พร้อมกันตลอดเวลา)");
}
```

ผลลัพธ์ (รันจริง — ลำดับ id ที่ได้ตั๋วอาจสลับกันไปมาตามธรรมชาติของ scheduler เพราะ 8 task แข่งกันขอตั๋วพร้อมกัน
แต่ **จำนวนตั๋วที่เหลือจะไม่ติดลบและไม่เกิน 3 ที่ทำงานพร้อมกันเลย**):

```
[request 1] ได้ตั๋วแล้ว กำลังทำงาน... (เหลือตั๋วว่าง: 2)
[request 2] ได้ตั๋วแล้ว กำลังทำงาน... (เหลือตั๋วว่าง: 1)
[request 4] ได้ตั๋วแล้ว กำลังทำงาน... (เหลือตั๋วว่าง: 0)
[request 4] ทำงานเสร็จ กำลังคืนตั๋ว
[request 6] ได้ตั๋วแล้ว กำลังทำงาน... (เหลือตั๋วว่าง: 0)
[request 2] ทำงานเสร็จ กำลังคืนตั๋ว
[request 7] ได้ตั๋วแล้ว กำลังทำงาน... (เหลือตั๋วว่าง: 0)
[request 1] ทำงานเสร็จ กำลังคืนตั๋ว
[request 8] ได้ตั๋วแล้ว กำลังทำงาน... (เหลือตั๋วว่าง: 0)
[request 3] ได้ตั๋วแล้ว กำลังทำงาน... (เหลือตั๋วว่าง: 0)
[request 6] ทำงานเสร็จ กำลังคืนตั๋ว
[request 5] ได้ตั๋วแล้ว กำลังทำงาน... (เหลือตั๋วว่าง: 0)
[request 7] ทำงานเสร็จ กำลังคืนตั๋ว
[request 5] ทำงานเสร็จ กำลังคืนตั๋ว
[request 3] ทำงานเสร็จ กำลังคืนตั๋ว
ทำงานครบ 8 คำขอแล้ว (ไม่เกิน 3 พร้อมกันตลอดเวลา)
```

สังเกตว่า `available_permits()` ไม่เคยติดลบและไม่เคยแสดงว่ามีมากกว่า 3 งานทำงานพร้อมกันเลย — เมื่อ 3 request
แรก (ในที่นี้คือ 1, 2, 4 ตามลำดับที่แข่งกันได้ตั๋วก่อน) ได้ตั๋วครบ 3 ใบแล้ว (`available_permits() == 0`)
request ที่เหลือทุกตัว (3, 5, 6, 7, 8) ต้องรอที่ `.acquire().await` จนกว่าจะมีใครสักคนคืนตั๋วก่อน — และทันทีที่
`request 4` ทำงานเสร็จคืนตั๋ว `request 6` ก็ได้ตั๋วไปทำงานต่อทันที ตั๋วจึงหมุนเวียนไปเรื่อย ๆ จนกว่าทั้ง 8
request จะเสร็จครบ โดยที่**ไม่มี worker thread ไหนถูกยึดไว้เฉย ๆ ระหว่างที่ request 3/5/6/7/8 กำลัง "รอตั๋ว"
อยู่เลย** เพราะ `.acquire().await` เป็น async เช่นเดียวกับ primitive อื่น ๆ ทั้งหมดในบทนี้

**`Semaphore` ยังมี method ที่มีประโยชน์อีกสองตัวที่ควรรู้จักไว้**: `.add_permits(n)` เพิ่มจำนวนตั๋วระหว่างทาง
ได้ (เช่นเมื่อ resource จริงที่กำลัง "จำลอง" ด้วย permit เพิ่มขึ้นจริง ๆ ระหว่างที่โปรแกรมกำลังรันอยู่) และ
`.close()` ปิด semaphore อย่างถาวร — ทุก `.acquire().await` ที่กำลังรออยู่หรือเรียกมาใหม่หลังจากนั้นจะได้
`Err(AcquireError)` กลับมาทันที (คล้ายกับการปิด channel ที่เรียนมาก่อนหน้านี้ในบทนี้ — เป็นสัญญาณแบบเดียวกันว่า
"เลิกให้บริการแล้วอย่างถาวร"):

```rust
use std::sync::Arc;
use tokio::sync::Semaphore;

#[tokio::main]
async fn main() {
    let semaphore = Arc::new(Semaphore::new(1));
    println!("ตั๋วเริ่มต้น: {}", semaphore.available_permits());

    semaphore.add_permits(2); // เพิ่มตั๋วระหว่างทางได้ (เช่นเมื่อ resource เพิ่มขึ้นจริง)
    println!("หลัง add_permits(2): {}", semaphore.available_permits());

    let permit = semaphore.acquire().await.unwrap();
    println!("ได้ตั๋วมา 1 ใบ เหลือ: {}", semaphore.available_permits());
    drop(permit);
    println!("คืนตั๋วแล้ว เหลือ: {}", semaphore.available_permits());

    semaphore.close(); // ปิดสำหรับตลอดไป -- ไม่มีใครขอตั๋วเพิ่มได้อีก
    let result = semaphore.acquire().await;
    match result {
        Ok(_) => println!("ไม่ควรได้ตั๋วอีกหลังปิดแล้ว"),
        Err(e) => println!("ขอตั๋วหลัง close() แล้ว: {e}"),
    }
}
```

ผลลัพธ์ (รันจริง):

```
ตั๋วเริ่มต้น: 1
หลัง add_permits(2): 3
ได้ตั๋วมา 1 ใบ เหลือ: 2
คืนตั๋วแล้ว เหลือ: 3
ขอตั๋วหลัง close() แล้ว: semaphore closed
```

`.close()` มีประโยชน์มากในสถานการณ์ **graceful shutdown**: เมื่อระบบต้องการปิดตัวอย่างสุภาพ (ไม่รับ connection
ใหม่อีกแล้ว แต่ปล่อยให้ connection ที่ทำงานอยู่ทำงานให้จบก่อน) การเรียก `semaphore.close()` ทำให้ทุก task ที่
กำลังรอ `.acquire().await` อยู่ (เช่น connection ใหม่ที่เพิ่งมา) ได้รับ `Err` และเลิกรอทันที โดยไม่กระทบ permit
ที่ถูกถือครองอยู่แล้วก่อนหน้านั้นเลยแม้แต่นิดเดียว

### 50.13 Synchronous vs Asynchronous `send`: จุดที่สับสนบ่อยที่สุดของ `tokio::sync`

ก่อนไปดูตารางเปรียบเทียบรวบยอด มีจุดสำคัญจุดหนึ่งที่ผู้เรียนสับสนกันบ่อยมากจนควรแยกมาพูดให้ชัดเจนต่างหาก:
**ไม่ใช่ทุก method ของ `tokio::sync` ที่ต้อง `.await`** — มีแค่บาง method เท่านั้นที่เป็น `async fn` จริง ๆ
ส่วนที่เหลือเป็นฟังก์ชัน synchronous ธรรมดาที่ทำงานจบในตัวทันที ทำไมถึงต่างกัน? คำตอบเชื่อมกับกลไกภายในของแต่
ละ primitive ตรง ๆ: **method ไหนที่ "อาจต้องรอ" จริง ๆ (อาจถูกพักได้) จะเป็น `async fn` — method ไหนที่ "ทำเสร็จ
ทันทีเสมอไม่มีทางต้องรอ" จะเป็นฟังก์ชันธรรมดา**

```rust
use tokio::sync::{broadcast, mpsc, oneshot, watch};

#[tokio::main]
async fn main() {
    // mpsc::Sender::send ต้อง .await เพราะ bounded channel อาจทำให้ต้องรอที่ว่าง (backpressure)
    let (tx1, mut rx1) = mpsc::channel::<i32>(1);
    tx1.send(1).await.unwrap();
    println!("mpsc: send ต้อง .await -> ส่งสำเร็จ, ได้รับ: {:?}", rx1.recv().await);

    // oneshot::Sender::send ไม่ต้อง .await -- มันแค่เขียนค่าลงที่เดียวและปลุก receiver ทันที ไม่มีทางรอ
    let (tx2, rx2) = oneshot::channel::<i32>();
    let send_result: Result<(), i32> = tx2.send(2); // ไม่มี .await เลย
    println!("oneshot: send ไม่ต้อง .await -> ผลลัพธ์การส่ง: {:?}", send_result);
    println!("oneshot: ได้รับ: {:?}", rx2.await);

    // broadcast::Sender::send ก็ไม่ต้อง .await -- มันแค่เขียนลง ring buffer แล้วปลุกทุก receiver ทันที
    let (tx3, mut rx3) = broadcast::channel::<i32>(4);
    let send_result3 = tx3.send(3); // ไม่มี .await เลย คืน Result<usize, SendError<T>> (usize = จำนวน receiver ที่ยังเปิดอยู่)
    println!("broadcast: send ไม่ต้อง .await -> ส่งถึง {:?} receiver", send_result3);
    println!("broadcast: ได้รับ: {:?}", rx3.recv().await);

    // watch::Sender::send ก็ไม่ต้อง .await เช่นกัน -- แค่เขียนทับค่าล่าสุด
    let (tx4, rx4) = watch::channel::<i32>(0);
    let send_result4 = tx4.send(4); // ไม่มี .await เลย
    println!("watch: send ไม่ต้อง .await -> ผลลัพธ์การส่ง: {:?}", send_result4);
    println!("watch: ค่าปัจจุบัน: {:?}", *rx4.borrow());
}
```

ผลลัพธ์ (รันจริง):

```
mpsc: send ต้อง .await -> ส่งสำเร็จ, ได้รับ: Some(1)
oneshot: send ไม่ต้อง .await -> ผลลัพธ์การส่ง: Ok(())
oneshot: ได้รับ: Ok(2)
broadcast: send ไม่ต้อง .await -> ส่งถึง Ok(1) receiver
broadcast: ได้รับ: Ok(3)
watch: send ไม่ต้อง .await -> ผลลัพธ์การส่ง: Ok(())
watch: ค่าปัจจุบัน: 4
```

**อธิบายเหตุผลของแต่ละตัว ทีละ primitive:**

- **`mpsc::Sender::send()` ต้อง `.await`** — เพราะ `tokio::sync::mpsc::channel(capacity)` เป็น bounded channel
  เสมอ (หัวข้อ 50.8) ถ้าบัฟเฟอร์เต็มพอดี การส่งจะต้อง**รอ**จนกว่า consumer จะดึงข้อความออกไปก่อนสักชิ้นหนึ่ง —
  มันเป็น operation ที่ "อาจต้องรอ" ได้จริง จึงต้องเป็น `async fn`
- **`oneshot::Sender::send()` ไม่ต้อง `.await`** — เพราะ oneshot มี "ช่อง" ให้เก็บค่าได้แค่ 1 ค่าเท่านั้นและ
  ไม่มีแนวคิดเรื่อง "เต็มแล้วต้องรอ" เลย (มันถูกออกแบบมาให้ใช้ครั้งเดียว ณ compile time อยู่แล้วตามหัวข้อ 50.9)
  การส่งจึงจบทันทีเสมอ ไม่มีทางต้องรอ — สังเกตว่ามันคืน `Result<(), T>` แบบ synchronous ตรง ๆ (`Err(T)` เกิดขึ้น
  ถ้า `Receiver` ถูก drop ไปแล้ว ก็คืนค่าเดิมกลับมาให้ ตามหลักการเดียวกับ `SendError<T>` ของ `std::sync::mpsc`
  ใน Part 38.2 ที่ไม่ยอมให้ค่าที่ส่งไม่สำเร็จสูญหายไปเฉย ๆ)
- **`broadcast::Sender::send()` ไม่ต้อง `.await`** — เพราะ `broadcast` เลือก trade-off แบบ "ไม่มี backpressure
  เลย" (หัวข้อ 50.10): ถ้าบัฟเฟอร์ภายในเต็ม มันจะทิ้งข้อความเก่าที่สุดออกไปแทนการรอ (ทำให้ receiver ที่ตามไม่ทัน
  ได้ `Lagged(n)` ในครั้งถัดไปที่ `.recv()`) — เพราะไม่มีทางต้องรอ มันจึงเป็นฟังก์ชันธรรมดา คืน
  `Result<usize, SendError<T>>` ที่ `usize` บอกจำนวน receiver ที่ยัง subscribe อยู่ ณ ขณะส่ง (มีประโยชน์เผื่อ
  อยากรู้ว่ามีใครฟังอยู่จริงหรือไม่)
- **`watch::Sender::send()` ไม่ต้อง `.await`** — เพราะ `watch` เก็บได้แค่ค่าเดียวเสมอ (หัวข้อ 50.11) การส่งคือ
  การ "เขียนทับ" ค่าเดิมตรง ๆ ไม่มีบัฟเฟอร์ให้เต็มเลย จึงไม่มีทางต้องรอเช่นกัน

จำกฎนี้ไว้แทนการท่องจำเป็นตัว ๆ: **ถามตัวเองว่า "operation นี้มีทางที่จะต้องรอจริงไหม (เช่น บัฟเฟอร์เต็ม, ยังไม่
มีข้อมูล)"** — ถ้ามี มันจะเป็น `async fn` ที่ต้อง `.await` ถ้าไม่มีทางเกิดขึ้นได้เลยไม่ว่ากรณีใด มันจะเป็น
ฟังก์ชันธรรมดา และลืม `.await` ตรงจุดที่ต้องมี (ตามที่เตือนไว้ในกับดักที่พบบ่อยข้อ 4) หรือเผลอเขียน `.await`
ตรงจุดที่ไม่มีให้ (ซึ่งจะเจอ compile error `no method named 'await' found` ทันที เพราะ type นั้นไม่ implement
`Future`) เป็นความผิดพลาดที่พบบ่อยมากตอนสลับไปมาระหว่าง primitive ต่าง ๆ ในบทนี้

### 50.14 ตารางเปรียบเทียบ: `std::sync` เทียบกับ `tokio::sync`

รวบทุกอย่างที่เรียนมาในบทนี้เป็นตารางเดียว เพื่อใช้อ้างอิงเร็ว ๆ เวลาต้องเลือก primitive ในโค้ดจริง:

| แนวคิด | `std::sync` (Part 38-39, OS thread) | `tokio::sync` (บทนี้, async task) | เมื่อไหร่ใช้ตัวไหน |
|---|---|---|---|
| Mutual exclusion | `std::sync::Mutex<T>` — `.lock()` **บล็อก OS thread** | `tokio::sync::Mutex<T>` — `.lock().await` **คืนสิทธิ์ให้ executor** | ใช้ `std` ถ้า critical section ไม่มี `.await`; ใช้ `tokio` ถ้ามี |
| Multiple readers | `std::sync::RwLock<T>` — `.read()`/`.write()` บล็อก | `tokio::sync::RwLock<T>` — `.read().await`/`.write().await` | กฎเดียวกับ Mutex |
| Message passing (1 คิว) | `std::sync::mpsc` — `.send()`/`.recv()` บล็อก, `sync_channel` มี bound | `tokio::sync::mpsc` — `.send().await`/`.recv().await`, bound เสมอ (หรือ `unbounded_channel()`) | ใช้ `tokio` เสมอในโค้ด async ที่ต้องคุยข้าม task |
| เจ้าของร่วมข้าม thread/task | `std::sync::Arc<T>` — atomic reference counting | **`Arc<T>` ตัวเดียวกัน ใช้ได้ทั้งสองโลก** | ไม่มีเวอร์ชัน async แยก เพราะ `Arc::clone` ไม่มี `.await` และไม่บล็อกอยู่แล้ว |
| ถาม-ตอบครั้งเดียว | *(ไม่มีในตัว — ต้องประกอบ `mpsc` เองแบบ hacky)* | `tokio::sync::oneshot` — `Sender<T>`/`Receiver<T>` ใช้ได้ครั้งเดียว | คำตอบเต็มรูปแบบของ pattern "reply channel" จาก Part 38 |
| กระจายให้หลายผู้รับ (ทุกข้อความ) | *(ไม่มีในตัว — ต้องประกอบ `Arc<Mutex<Vec<Sender>>>` เอง)* | `tokio::sync::broadcast` — `.subscribe()` ได้หลายตัว ทุกตัวได้ครบทุกข้อความ (หรือ `Lagged` ถ้าตามไม่ทัน) | คำตอบเต็มรูปแบบของ pattern "chat broadcast" จาก Part 49 |
| กระจายแค่ค่าล่าสุด | *(ไม่มีในตัว)* | `tokio::sync::watch` — `.changed().await`/`.borrow()`, เก็บแค่ค่าล่าสุด | configuration update, state ที่สนใจแค่ "ปัจจุบัน" |
| จำกัดจำนวนงานพร้อมกัน | *(ไม่มีในตัว — ต้องประกอบด้วย `Mutex<usize>`/`Condvar` เอง)* | `tokio::sync::Semaphore` — `.acquire().await` คืน permit แบบ RAII | จำกัด concurrent connection/API call/resource ใด ๆ |

**เส้นแบ่งที่สำคัญที่สุดในตารางนี้ที่ควรจำไว้ตลอด**: แถวบนสองแถว (`Mutex`/`RwLock`) มีทั้งสองเวอร์ชันให้เลือกตาม
สถานการณ์ (กฎการตัดสินใจจากหัวข้อ 50.6) ส่วนแถวที่เหลือ **สามแถวสุดท้ายของ `tokio::sync`** (`oneshot`,
`broadcast`, `watch`, `Semaphore`) **ไม่มีคู่เทียบใน `std::sync` เลย** — ไม่ใช่เพราะมันเป็นแค่ "เวอร์ชัน async"
ของอะไรบางอย่างที่มีอยู่แล้ว แต่เพราะมันคือ primitive ใหม่ที่ออกแบบมาสำหรับรูปแบบปัญหาที่พบบ่อยเป็นพิเศษในโลก
async/concurrent จนสมควรมี type สำเร็จรูปให้ใช้เลย

### 50.15 ตัวอย่างจริง: Chat Server เวอร์ชัน Production-Quality (`broadcast` + `Semaphore`)

ถึงเวลานำทุกอย่างที่เรียนมารวมกันเป็นตัวอย่างเดียวที่สมบูรณ์ — สร้าง chat server ผ่าน TCP จริง (ต่อยอดจาก
`TcpListener`/`TcpStream` ของ Part 49) ที่แก้ปัญหาทั้งสองข้อที่ Part 49 ร่างไว้แบบง่าย ๆ ให้ถูกต้องแบบ
production:

1. **กระจายข้อความให้ทุก client** ด้วย `tokio::sync::broadcast` (หัวข้อ 50.10) แทน
   `Arc<Mutex<Vec<Sender>>>` แบบเดิม
2. **จำกัดจำนวน connection พร้อมกันสูงสุด** ด้วย `tokio::sync::Semaphore` (หัวข้อ 50.12) — ฟีเจอร์ที่ Part 49
   ยังไม่มีเครื่องมือให้ทำเลย

```rust
use std::net::SocketAddr;
use std::sync::Arc;
use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use tokio::net::{TcpListener, TcpStream};
use tokio::sync::{broadcast, Semaphore};
use tokio::time::{sleep, Duration, Instant};

const MAX_CONNECTIONS: usize = 2;

async fn handle_client(
    socket: TcpStream,
    addr: SocketAddr,
    tx: broadcast::Sender<String>,
    semaphore: Arc<Semaphore>,
    start: Instant,
) {
    // ขอ "ตั๋วเชื่อมต่อ" ก่อนทำงานอะไรเลย -- ถ้าตั๋วหมด รอที่นี่จนกว่าจะมีคนคืนตั๋ว
    let _permit = semaphore.acquire().await.expect("semaphore closed");
    println!(
        "[server] {addr} เชื่อมต่อสำเร็จ (ได้ตั๋วแล้ว) ที่ {:?} (เหลือตั๋วว่าง: {})",
        start.elapsed(),
        semaphore.available_permits()
    );

    let mut rx = tx.subscribe();
    let (reader, mut writer) = socket.into_split();
    let mut lines = BufReader::new(reader).lines();

    let welcome = format!("*** {addr} เข้าห้องแชทแล้ว ***\n");
    let _ = tx.send(welcome);

    loop {
        tokio::select! {
            line = lines.next_line() => {
                match line {
                    Ok(Some(text)) => {
                        let msg = format!("[{addr}] {text}\n");
                        let _ = tx.send(msg);
                    }
                    Ok(None) => break, // client ปิดการเชื่อมต่อ (EOF)
                    Err(_) => break,
                }
            }
            received = rx.recv() => {
                match received {
                    Ok(msg) => {
                        if writer.write_all(msg.as_bytes()).await.is_err() {
                            break;
                        }
                    }
                    Err(broadcast::error::RecvError::Lagged(n)) => {
                        eprintln!("[server] {addr} ตกข้อความไป {n} ข้อความ (อ่านไม่ทัน)");
                    }
                    Err(broadcast::error::RecvError::Closed) => break,
                }
            }
        }
    }

    let goodbye = format!("*** {addr} ออกจากห้องแชทแล้ว ***\n");
    let _ = tx.send(goodbye);
    println!(
        "[server] {addr} หลุดการเชื่อมต่อที่ {:?} (ตั๋วถูกคืนอัตโนมัติตอน _permit หลุด scope)",
        start.elapsed()
    );
    // _permit หลุด scope ที่นี่ -> Drop -> คืนตั๋วให้ semaphore โดยอัตโนมัติ (RAII จาก Part 6)
}

async fn run_server(
    listener: TcpListener,
    tx: broadcast::Sender<String>,
    semaphore: Arc<Semaphore>,
    start: Instant,
) {
    loop {
        match listener.accept().await {
            Ok((socket, peer)) => {
                let tx = tx.clone();
                let semaphore = Arc::clone(&semaphore);
                tokio::spawn(handle_client(socket, peer, tx, semaphore, start));
            }
            Err(e) => {
                eprintln!("[server] accept error: {e}");
                break;
            }
        }
    }
}
```

**อธิบายทีละกลไก:**

- `semaphore.acquire().await` เป็นบรรทัด**แรกสุด**ของ `handle_client` — งานทุกอย่างที่ตามมา (subscribe เข้า
  broadcast, ประกาศ welcome, วน select loop) **จะไม่เกิดขึ้นเลยจนกว่าจะได้ตั๋ว** นี่คือการบังคับ "เพดานจำนวน
  chat session ที่กำลังทำงานจริงพร้อมกัน" ที่แม่นยำมาก — connection ที่ 3 (เมื่อ `MAX_CONNECTIONS = 2`) จะเชื่อม
  TCP สำเร็จก่อน (เพราะ `listener.accept()` ไม่รู้จัก semaphore เลย มันแค่รับ connection ตาม TCP backlog) แต่
  `handle_client` ของมันจะ **"แช่แข็ง" อยู่ที่ `.acquire().await`** จนกว่า client ตัวใดตัวหนึ่งในสองตัวแรกจะออก
  จากห้องไปก่อน
- `tx.subscribe()` เรียกได้อย่างอิสระทุกครั้งที่ client ใหม่เข้ามา ไม่ต้องล็อกอะไรเองเหมือนเวอร์ชันเก่าในหัวข้อ
  50.10 เลย
- `tokio::select!` (จาก Part 49) เป็นหัวใจของ loop — รอสองอย่างพร้อมกัน: **(1)** อ่านบรรทัดใหม่จาก client
  ตัวเอง (`lines.next_line()`) แล้วส่งเข้า broadcast ให้ทุกคน หรือ **(2)** รับข้อความที่คนอื่น broadcast มา
  (`rx.recv()`) แล้วเขียนกลับไปให้ client ตัวเอง — `select!` จะทำงานทันทีที่อย่างใดอย่างหนึ่งพร้อมก่อน ไม่ต้อง
  สร้าง task แยกสองตัวสำหรับอ่าน/เขียนก็ทำงานคู่ขนานได้ในเวลาเดียวกัน
- `RecvError::Lagged(n)` ถูก match แยกออกมาต่างหาก (ตามที่อธิบายในหัวข้อ 50.10) — ถ้า client คนใดคนหนึ่งอ่าน
  ข้อความช้าเกินไปจนบัฟเฟอร์ (16 ข้อความในตัวอย่างนี้) ล้น เซิร์ฟเวอร์จะแค่แจ้งเตือนใน log แล้ว**ทำงานต่อ**ไม่
  ปิดการเชื่อมต่อทิ้งไปเฉย ๆ

ต่อด้วยส่วนจำลอง client เพื่อพิสูจน์ว่าทั้งระบบทำงานถูกต้องจริง — สาม client (Alice, Bob, Carol) เชื่อมต่อ
ห่างกันเล็กน้อย โดยที่ `MAX_CONNECTIONS = 2` หมายความว่า Carol (คนที่ 3) ต้องรอคิว:

```rust
async fn run_client(
    name: &'static str,
    addr: SocketAddr,
    connect_delay_ms: u64,
    messages: Vec<&'static str>,
    start: Instant,
) {
    sleep(Duration::from_millis(connect_delay_ms)).await;
    println!("[{name}] กำลังเชื่อมต่อที่ {:?}", start.elapsed());
    let stream = TcpStream::connect(addr).await.expect("connect failed");
    println!("[{name}] เชื่อมต่อ TCP สำเร็จที่ {:?}", start.elapsed());
    let (reader, mut writer) = stream.into_split();
    let mut lines = BufReader::new(reader).lines();

    // task ย่อยสำหรับอ่านข้อความที่ broadcast มาถึง แล้วพิมพ์ออกมา
    let reader_task = tokio::spawn(async move {
        while let Ok(Some(line)) = lines.next_line().await {
            println!("  [{name} ได้รับ] {line}");
        }
    });

    for msg in messages {
        sleep(Duration::from_millis(80)).await;
        let _ = writer.write_all(format!("{msg}\n").as_bytes()).await;
    }

    sleep(Duration::from_millis(150)).await;
    drop(writer); // ปิดการเชื่อมต่อฝั่งนี้ -> reader ฝั่ง server จะเจอ EOF แล้วปล่อยตั๋วคืน
    let _ = reader_task.await;
    println!("[{name}] ออกจากห้องแล้วที่ {:?}", start.elapsed());
}

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let start = Instant::now();
    let listener = TcpListener::bind("127.0.0.1:0").await?;
    let addr = listener.local_addr()?;
    println!("[server] เริ่มฟังที่ {addr} (จำกัดสูงสุด {MAX_CONNECTIONS} การเชื่อมต่อพร้อมกัน)");

    let (tx, _rx) = broadcast::channel::<String>(16);
    let semaphore = Arc::new(Semaphore::new(MAX_CONNECTIONS));

    tokio::spawn(run_server(listener, tx.clone(), Arc::clone(&semaphore), start));

    sleep(Duration::from_millis(50)).await; // ให้ server พร้อมก่อน

    let c1 = tokio::spawn(run_client("Alice", addr, 0, vec!["สวัสดีทุกคน"], start));
    let c2 = tokio::spawn(run_client("Bob", addr, 20, vec!["หวัดดีครับ"], start));
    // MAX_CONNECTIONS = 2 -- Carol ต้องรอ "ตั๋ว" จนกว่า Alice หรือ Bob จะออกจากห้องก่อน
    let c3 = tokio::spawn(run_client("Carol", addr, 40, vec!["มาสายนิดหน่อย"], start));

    let _ = tokio::join!(c1, c2, c3);

    sleep(Duration::from_millis(100)).await;
    println!("[main] จบการทดสอบที่ {:?}", start.elapsed());
    Ok(())
}
```

ผลลัพธ์ (รันจริงบน `127.0.0.1` ด้วย port ที่ OS สุ่มเลือกให้ผ่าน `"127.0.0.1:0"` — ตัวเลขเวลา/พอร์ตจะขยับ
เล็กน้อยในแต่ละครั้งที่รันตามธรรมชาติ แต่ **ลำดับเหตุการณ์และการบังคับ `MAX_CONNECTIONS` จะเหมือนกันเสมอ**):

```
[server] เริ่มฟังที่ 127.0.0.1:41427 (จำกัดสูงสุด 2 การเชื่อมต่อพร้อมกัน)
[Alice] กำลังเชื่อมต่อที่ 52.692721ms
[Alice] เชื่อมต่อ TCP สำเร็จที่ 52.995758ms
[server] 127.0.0.1:51776 เชื่อมต่อสำเร็จ (ได้ตั๋วแล้ว) ที่ 53.098557ms (เหลือตั๋วว่าง: 1)
  [Alice ได้รับ] *** 127.0.0.1:51776 เข้าห้องแชทแล้ว ***
[Bob] กำลังเชื่อมต่อที่ 73.435098ms
[Bob] เชื่อมต่อ TCP สำเร็จที่ 73.568321ms
[server] 127.0.0.1:51784 เชื่อมต่อสำเร็จ (ได้ตั๋วแล้ว) ที่ 73.62749ms (เหลือตั๋วว่าง: 0)
  [Alice ได้รับ] *** 127.0.0.1:51784 เข้าห้องแชทแล้ว ***
  [Bob ได้รับ] *** 127.0.0.1:51784 เข้าห้องแชทแล้ว ***
[Carol] กำลังเชื่อมต่อที่ 92.812979ms
[Carol] เชื่อมต่อ TCP สำเร็จที่ 93.052751ms
  [Bob ได้รับ] [127.0.0.1:51776] สวัสดีทุกคน
  [Alice ได้รับ] [127.0.0.1:51776] สวัสดีทุกคน
  [Alice ได้รับ] [127.0.0.1:51784] หวัดดีครับ
  [Bob ได้รับ] [127.0.0.1:51784] หวัดดีครับ
[server] 127.0.0.1:51776 หลุดการเชื่อมต่อที่ 285.465228ms (ตั๋วถูกคืนอัตโนมัติตอน _permit หลุด scope)
[server] 127.0.0.1:51796 เชื่อมต่อสำเร็จ (ได้ตั๋วแล้ว) ที่ 285.576235ms (เหลือตั๋วว่าง: 0)
  [Carol ได้รับ] *** 127.0.0.1:51796 เข้าห้องแชทแล้ว ***
  [Carol ได้รับ] [127.0.0.1:51796] มาสายนิดหน่อย
[Alice] ออกจากห้องแล้วที่ 285.727059ms
  [Bob ได้รับ] *** 127.0.0.1:51776 ออกจากห้องแชทแล้ว ***
  [Bob ได้รับ] *** 127.0.0.1:51796 เข้าห้องแชทแล้ว ***
  [Bob ได้รับ] [127.0.0.1:51796] มาสายนิดหน่อย
[server] 127.0.0.1:51784 หลุดการเชื่อมต่อที่ 306.968934ms (ตั๋วถูกคืนอัตโนมัติตอน _permit หลุด scope)
  [Carol ได้รับ] *** 127.0.0.1:51784 ออกจากห้องแชทแล้ว ***
[Bob] ออกจากห้องแล้วที่ 307.10111ms
[server] 127.0.0.1:51796 หลุดการเชื่อมต่อที่ 325.406911ms (ตั๋วถูกคืนอัตโนมัติตอน _permit หลุด scope)
[Carol] ออกจากห้องแล้วที่ 325.486735ms
[main] จบการทดสอบที่ 426.981069ms
```

มาไล่อ่านผลลัพธ์นี้ให้ทะลุ เพราะมันพิสูจน์ทั้งสองกลไกพร้อมกันในการรันครั้งเดียว:

1. **Alice** (delay 0ms) และ **Bob** (delay 20ms) เชื่อมต่อและได้ตั๋วทันที (เหลือตั๋วว่าง 1 แล้ว 0 ตามลำดับ) —
   ทั้งสองคนเห็นข้อความ "เข้าห้องแชท" และข้อความทักทายของกันและกันครบถูกต้อง (พิสูจน์ว่า `broadcast` กระจาย
   ข้อความให้ทุก subscriber จริง)
2. **Carol** (delay 40ms) เชื่อมต่อ TCP สำเร็จที่ ~93ms (สังเกตว่า TCP handshake ผ่านได้ปกติ เพราะ `Semaphore`
   ไม่เกี่ยวกับ `listener.accept()` เลย) แต่สังเกตว่า **ไม่มีบรรทัด `"[server] ... เชื่อมต่อสำเร็จ (ได้ตั๋วแล้ว)"`
   สำหรับ Carol ปรากฏขึ้นเลยในตอนนั้น** — เธอ "ค้าง" อยู่ที่ `.acquire().await` เพราะตั๋วทั้ง 2 ใบถูก Alice/Bob
   ถือครองอยู่เต็มพอดี
3. เมื่อ **Alice ออกจากห้องที่ ~285ms** (ปิด `writer` ทำให้ server เจอ EOF, `_permit` ของ Alice หลุด scope
   คืนตั๋ว) **ทันทีบรรทัดต่อไปคือ Carol ได้ตั๋วแล้ว** ("127.0.0.1:51796 เชื่อมต่อสำเร็จ (ได้ตั๋วแล้ว) ที่
   285.576235ms") — ห่างจากตอน Alice ออกไปแค่เสี้ยว millisecond เท่านั้น พิสูจน์ตรง ๆ ว่า Carol รอตั๋วอยู่จริง
   และได้ตั๋วทันทีที่มีคนคืน (ไม่ใช่ต้องรอ polling หรือ timeout อะไรเลย)
4. หลังจากนั้น Carol เข้าห้องได้ตามปกติ ส่งข้อความของตัวเอง ("มาสายนิดหน่อย") และ Bob (ซึ่งยังอยู่ในห้อง) ก็ได้
   รับทั้งประกาศ "เข้าห้อง" และข้อความของ Carol ครบถูกต้อง — พิสูจน์ว่าระบบทำงานถูกต้องสมบูรณ์ทั้งสองกลไกร่วมกัน:
   **broadcast กระจายข้อความให้ทุกคนที่อยู่ในห้อง ณ ขณะนั้น และ semaphore ควบคุมว่า "อยู่ในห้อง ณ ขณะนั้น" ได้
   ไม่เกิน 2 คนตลอดเวลา**

เทียบกับ pattern `Arc<Mutex<Vec<Sender>>>` ที่ Part 49 ร่างไว้: เวอร์ชันนี้ **ไม่มี `Mutex` เกี่ยวข้องกับการ
กระจายข้อความเลยแม้แต่จุดเดียว** (มีแต่ `broadcast::Sender::clone()`/`.subscribe()`/`.send()` ที่ปลอดภัยในตัว
เองอยู่แล้ว) และได้ฟีเจอร์ "จำกัดจำนวน connection พร้อมกัน" เพิ่มมาแบบ**เกือบไม่ต้องเขียนโค้ดเพิ่มเลย** (แค่
`Semaphore::new(N)`, `.acquire().await` หนึ่งบรรทัด, และปล่อยให้ RAII จัดการคืนตั๋วให้เอง) — นี่คือความหมายของ
คำว่า "production-quality": ไม่ใช่แค่ "ทำงานได้" แต่ **ใช้ type ที่ตรงกับความหมายของปัญหา** ทำให้โค้ดทั้งสั้น
กว่า อ่านง่ายกว่า และปลอดภัยกว่าเวอร์ชันร่างแบบง่าย ๆ ในทุกมิติ

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. ถือ `std::sync::MutexGuard` ข้าม `.await` แล้ว `tokio::spawn` — compile ไม่ผ่าน

```
error: future cannot be sent between threads safely
= help: within `impl Future<Output = ()>`, the trait `Send` is not implemented for `std::sync::MutexGuard<'_, i32>`
note: future is not `Send` as this value is used across an await
```

ตามที่พิสูจน์ในหัวข้อ 50.3 — เกิดเมื่อ async function ล็อก `std::sync::Mutex` แล้วมี `.await` point ตามมาโดยที่
`MutexGuard` ยังไม่หลุด scope และ future นั้นถูกส่งเข้า `tokio::spawn` (ซึ่งกำหนด bound `Send`) **วิธีแก้**:
เปลี่ยนเป็น `tokio::sync::Mutex` แล้วใช้ `.lock().await` แทน — หรือถ้าเป็นไปได้ ให้ปรับโครงสร้างโค้ดให้ปล่อย
`guard` ก่อนถึง `.await` เสมอ (คัดลอกค่าที่ต้องการออกมาก่อน แบบเดียวกับ `good_pattern` ใน Part 39.4) ถ้า
critical section ไม่จำเป็นต้อง `.await` จริง ๆ

### 2. ถือ `std::sync::MutexGuard` ข้าม `.await` โดยไม่ผ่าน `tokio::spawn` — compile ผ่านแต่ Deadlock จริง

ตามที่พิสูจน์ด้วยการรันจริงในหัวข้อ 50.4 (`timeout 3 ./program` แล้วได้ exit code 124) — ถ้าโค้ดหลีกเลี่ยง
เงื่อนไข `Send` ของ `tokio::spawn` ได้ (เช่นรันสอง future พร้อมกันด้วย `tokio::join!`/`select!` ในฟังก์ชัน
เดียวกัน) compiler จะ**ไม่มีทางเตือนได้เลย** ทั้งที่ยังมีความเสี่ยง deadlock อยู่จริง ถ้า task ที่ถือ
`std::sync::MutexGuard` ข้าม `.await` ไปพักอยู่ และ task อื่นที่ใช้ worker thread เส้นเดียวกัน (โดยเฉพาะใน
runtime แบบ `current_thread` หรือเมื่อ worker thread อื่นกำลังยุ่งอยู่พอดี) มาเรียก `.lock()` แบบ blocking บน
`Mutex` ตัวเดียวกันนั้น **วิธีแก้**: กฎง่าย ๆ ที่ใช้ได้เสมอไม่ว่า compiler จะจับให้หรือไม่ — **ห้ามถือ
`std::sync::MutexGuard` ข้าม `.await` point เด็ดขาด ไม่มีข้อยกเว้น** ถ้าจำเป็นต้อง `.await` ขณะยังต้องรักษา
สถานะที่แบ่งกันใช้อยู่ ให้เปลี่ยนไปใช้ `tokio::sync::Mutex` ทันที อย่ารอให้ compiler เตือนก่อนเพราะบางกรณี
มันเตือนไม่ได้

### 3. Overcorrection: เปลี่ยนทุก `Mutex` เป็น `tokio::sync::Mutex` "เผื่อไว้" ทั้งที่ไม่จำเป็น

โค้ดที่เขียนแบบนี้ **compile ผ่านและทำงานถูกต้อง** แต่ **ช้าลงโดยไม่จำเป็น**:

```rust
// ไม่ผิด แต่ "เกินความจำเป็น" ถ้า critical section ไม่มี .await เลย
use tokio::sync::Mutex;
async fn record_hit(counter: &std::sync::Arc<Mutex<u64>>) {
    let mut n = counter.lock().await; // เพิ่ม overhead ของ async lock ทั้งที่ไม่จำเป็น
    *n += 1;
}
```

ตามกฎการตัดสินใจในหัวข้อ 50.6 — ถ้า critical section สั้นและไม่มี `.await` ข้างใน `std::sync::Mutex` ทำงานได้
ดีกว่าและเบากว่า **วิธีแก้**: ก่อนเปลี่ยนไปใช้ `tokio::sync::Mutex` ทุกครั้ง ให้ถามตัวเองก่อนเสมอว่า **"ข้างใน
critical section นี้มี `.await` อยู่จริงหรือไม่"** — ถ้าไม่มี ให้ใช้ `std::sync::Mutex` ต่อไป ไม่ต้องเปลี่ยน
เพราะ "อยู่ในโค้ด async" เฉย ๆ

### 4. ลืม `.await` หลัง `.lock()`/`.send()`/`.recv()`/`.acquire()` ของ `tokio::sync`

```
error[E0277]: `impl Future<Output = MutexGuard<'_, i32>>` is not a future that resolves to `MutexGuard<'_, i32>`
```

(หรือในหลายกรณี compiler จะเตือนแบบตรงกว่านั้นด้วย warning `unused implementer of 'Future' that must be used`)
เกิดเพราะทุก method ของ `tokio::sync` ที่มีชื่อเหมือน `std::sync` (`.lock()`, `.send()`, `.recv()`) เป็น
**`async fn`** ทั้งหมด — เรียกแล้วได้ `impl Future<...>` กลับมาเฉย ๆ ถ้าไม่ `.await` มันจะไม่ทำอะไรเลย (ตาม Part
46 ที่สอนว่า future ที่ไม่ถูก poll จะไม่ขยับ) **วิธีแก้**: ตรวจสอบทุกจุดที่ port โค้ดจาก `std::sync` มาเป็น
`tokio::sync` ว่าเติม `.await` ครบทุก method call แล้ว — เป็นความผิดพลาดที่พบบ่อยมากตอน migrate โค้ดเก่า

### 5. ใช้ `tokio::sync::mpsc::Receiver` เดียวกันจากหลาย task โดยตรง

```
error[E0382]: use of moved value: `rx`
```

`tokio::sync::mpsc::Receiver<T>` (เหมือน `std::sync::mpsc::Receiver<T>`) **ไม่ implement `Clone`** เพราะ mpsc
คือ "single consumer" ตามชื่อ — ถ้าต้องการให้หลาย task แบ่ง `Receiver` ตัวเดียวกันดึงงานออกไปกัน (worker pool
pattern จากหัวข้อ 50.8) ต้องห่อด้วย `Arc<tokio::sync::Mutex<Receiver<T>>>` (**ไม่ใช่** `std::sync::Mutex`
เพราะ critical section มี `.recv().await` อยู่ข้างใน ตามกฎหัวข้อ 50.6) **วิธีแก้**: ใช้ pattern จากหัวข้อ 50.8
ตรง ๆ หรือถ้าความหมายจริง ๆ ของปัญหาคือ "ทุกคนต้องได้ข้อความเดียวกันครบทุกคน" (ไม่ใช่ "แบ่งงานกันทำ") ให้ใช้
`tokio::sync::broadcast` แทนตั้งแต่แรก เพราะนั่นคือ primitive ที่ตรงกับความหมายนั้นโดยตรง

### 6. เข้าใจผิดว่า `broadcast::Receiver::recv()` คืน `Ok`/`Err` แบบเดียวกับ `mpsc` ทุกประการ

โค้ดที่ `match` แบบนี้จะพลาดกรณี lag:

```rust
// พลาด: มองว่า Err ทุกแบบคือ "ปิดแล้ว" เหมือน mpsc — จะ panic/หยุดทำงานทั้งที่ sender ยังส่งต่อได้อยู่
// while let Ok(msg) = rx.recv().await { ... } // ออกจาก loop ทันทีที่ Lagged เกิดขึ้นครั้งแรก ทั้งที่ยังไม่ปิดจริง
```

`broadcast::Receiver::recv()` มี `Err` สองความหมายต่างกันโดยสิ้นเชิง (หัวข้อ 50.10): `Closed` (จบจริง) กับ
`Lagged(n)` (แค่ตกข้อความไปบางส่วน ยังรับต่อได้ปกติ) **วิธีแก้**: `match` แยกทั้งสอง variant เสมอเหมือนในตัวอย่าง
chat server ของหัวข้อ 50.15 — จัดการ `Lagged` ด้วยการ log/แจ้งเตือนแล้ว **ทำงานต่อ** (loop กลับไป `.recv()`
ใหม่) ไม่ใช่ `break`/panic ออกจาก loop ไปเหมือนเจอ `Closed`

### 7. ลืม `drop()` ตัวต้นฉบับของ `mpsc::Sender` — channel ไม่ปิดแม้ clone ทุกตัวจะถูก drop ไปแล้ว

นี่คือ pitfall เดียวกันเป๊ะกับที่ Part 38.5 เตือนไว้สำหรับ `std::sync::mpsc` — และยังคงเป็นกับดักที่พบบ่อยที่สุด
อันหนึ่งใน `tokio::sync::mpsc` เช่นกัน เพราะกลไก "channel จะปิดก็ต่อเมื่อ `Sender` ทุกตัวถูก drop จนหมดสิ้น"
เหมือนกันทุกประการ:

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel::<i32>(8);

    let mut worker_handles = vec![];
    for id in 1..=3 {
        let tx_clone = tx.clone();
        worker_handles.push(tokio::spawn(async move {
            for i in 0..2 {
                tx_clone.send(id * 10 + i).await.unwrap();
            }
        }));
    }
    // ★ ลืม drop(tx) ตัวต้นฉบับ -- channel จะไม่ปิดแม้ clone ทุกตัวจะถูก drop ไปแล้วก็ตาม

    for h in worker_handles {
        h.await.unwrap();
    }

    while let Some(v) = rx.recv().await {
        println!("ได้รับ: {v}");
    }
    println!("loop จบแล้ว (ไม่ควรพิมพ์ถึงตรงนี้เลย)");
}
```

รันจริงด้วย `timeout 3 ./program` แล้วพิมพ์ค่าได้ครบทั้ง 6 ค่าจริง (`10, 11, 20, 21, 30, 31` ตามลำดับที่แต่ละ
worker แข่งกันส่ง) แต่หลังจากนั้น**ค้างตลอดไปและถูก kill (exit code 124)** — บรรทัด `"loop จบแล้ว..."` ไม่
ปรากฏเลย เพราะแม้ `tx_clone` ของทั้ง 3 worker จะถูก drop อัตโนมัติไปแล้วตอน task จบ (หลัง `worker_handles`
ทุกตัว `.await` เสร็จ) **`tx` ตัวต้นฉบับใน `main` ก็ยังมีชีวิตอยู่** (ไม่เคยถูก `.clone()` ทิ้งไปไหน แค่ยังไม่ถูก
ใช้ต่อเฉย ๆ) — channel จึงยังไม่ปิด และ `rx.recv().await` ที่เหลืออยู่จะ**คืนสิทธิ์การรันให้ executor รอไปเรื่อย
ๆ อย่างไม่มีที่สิ้นสุด** (ต่างจาก Part 38.5 ที่ `for received in rx` แบบ OS thread บล็อก thread ทั้งเส้นไปเลย
— ในเวอร์ชัน async นี้ executor ยังทำงานปกติ แค่ task นี้ตัวเดียวไม่มีวันคืบหน้าต่อ) **วิธีแก้**: เหมือนกับ
Part 38.5 ทุกประการ — `drop(tx);` ตัวต้นฉบับทันทีหลังจาก `.clone()` ให้ทุก worker ครบแล้ว (หรือให้ `tx` ตัว
ต้นฉบับหลุด scope ไปเองก่อนถึงจุดที่เริ่มวน `rx.recv().await`) เพื่อให้ channel ปิดได้จริงเมื่อ `Sender` ทุกตัว
(รวมตัวต้นฉบับ) หมดอายุลงจริง ๆ

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เขียนโปรแกรมที่มี `Arc<tokio::sync::Mutex<Vec<i32>>>` เริ่มต้นเป็น vector ว่าง แล้ว `tokio::spawn`
   จำนวน 5 task โดยแต่ละ task `.push()` ค่า `id * 100` (id คือเลขลำดับ 0-4) เข้าไปใน vector ที่แบ่งกันใช้ —
   หลัง `.await` ครบทุก `JoinHandle` แล้ว พิมพ์ vector ออกมาและตรวจด้วย `assert_eq!` ว่าความยาวเท่ากับ 5 พอดี
   (คำใบ้: โครงสร้างนี้เหมือน exercise ข้อ 1 ของ Part 39 ทุกประการ เพียงแค่เปลี่ยน `std::thread::spawn` เป็น
   `tokio::spawn` และเปลี่ยน `std::sync::Mutex` เป็น `tokio::sync::Mutex` — สังเกตว่าในกรณีนี้ critical
   section (`.push()`) ไม่มี `.await` เลย ลองตอบด้วยว่าจริง ๆ ควรใช้ `std::sync::Mutex` หรือ `tokio::sync::Mutex`
   ตามกฎในหัวข้อ 50.6 แล้วลองเขียนทั้งสองแบบเทียบกัน)

2. **(กลาง)** ปรับตัวอย่าง worker pool ในหัวข้อ 50.8 ให้แต่ละ job มี field เพิ่มเติมคือ `oneshot::Sender<i32>`
   สำหรับส่งผลลัพธ์ (`payload * 2`) กลับไปให้ผู้ส่ง job นั้นโดยเฉพาะ (รวม pattern ของหัวข้อ 50.8 กับ 50.9 เข้า
   ด้วยกัน) แล้วเขียนฟังก์ชัน `submit_job(tx: &mpsc::Sender<Job>, payload: i32) -> i32` ที่ส่ง job เข้า pool
   แล้ว `.await` รอผลลัพธ์กลับมาโดยตรง (คล้าย `query()` ในหัวข้อ 50.9) ทดสอบด้วยการเรียก `submit_job` แบบ
   concurrent จากหลาย task พร้อมกัน (อย่างน้อย 10 ครั้ง) แล้วตรวจว่าผลลัพธ์ที่ได้กลับมาตรงกับ `payload * 2` ของ
   แต่ละครั้งที่ส่งไปเสมอ ไม่มีทางสลับคำตอบของกันและกันได้เลย

3. **(ยาก)** เขียนระบบ "cache ที่มีอายุ" (TTL cache) แบบง่าย ๆ ที่ใช้ `tokio::sync::RwLock<HashMap<String,
   (String, tokio::time::Instant)>>` เก็บ key-value พร้อมเวลาที่บันทึกไว้ — เขียนฟังก์ชัน `get_or_compute
   (cache: &Arc<RwLock<...>>, key: &str, ttl: Duration, compute: impl Fn() -> String) -> String` ที่ตรวจสอบ
   ก่อนว่า key นี้มีอยู่ใน cache หรือไม่ (ใช้ **read lock** ก่อนเสมอสำหรับการตรวจสอบนี้) ถ้ามีและยังไม่หมด TTL
   ให้คืนค่าที่ cache ไว้ทันที ถ้าไม่มีหรือหมด TTL แล้ว ให้ขอ **write lock** เพื่อคำนวณค่าใหม่ด้วย `compute()`
   แล้วบันทึกลง cache ก่อนคืนค่า จำลอง 20 task เรียก `get_or_compute` พร้อมกันด้วย key ซ้ำกันไม่กี่ตัว แล้วนับ
   ว่า `compute()` ถูกเรียกไปกี่ครั้งจริง (คำใบ้: ระวังปัญหา "thundering herd" — ถ้าหลาย task เห็น cache miss
   พร้อมกันเป๊ะ ๆ ก่อนที่ใครจะทันเขียนค่าใหม่ลงไป ทุก task อาจเรียก `compute()` ซ้ำกันหมด ลองคิดว่าจะป้องกัน
   ปัญหานี้อย่างไรด้วยเครื่องมือที่เรียนมาในบทนี้ เช่น `tokio::sync::Mutex` หรือ `tokio::sync::Semaphore`
   ประกอบเพิ่มเข้าไป)

4. **(ประยุกต์ใช้งานจริง)** ขยายตัวอย่าง chat server ในหัวข้อ 50.15 ให้รองรับ **"ห้องแชทหลายห้อง"** (multiple
   chat rooms) — ให้แต่ละห้องมี `broadcast::Sender<String>` และ `Semaphore` ของตัวเอง (เก็บอยู่ใน
   `HashMap<String, RoomState>` โดย `RoomState` เก็บทั้งสองอย่างนี้) ผู้ใช้ที่เชื่อมต่อเข้ามาต้องส่งชื่อห้องที่
   ต้องการเข้าเป็นบรรทัดแรกก่อน (เช่น `"JOIN general"`) จากนั้นข้อความทั้งหมดที่ตามมาจะถูก broadcast แค่ภายใน
   ห้องนั้นเท่านั้น ไม่รั่วไปห้องอื่น ให้แต่ละห้องมี `MAX_CONNECTIONS` ของตัวเองที่ตั้งค่าต่างกันได้ (เช่นห้อง
   "general" รับได้ 5 คน ห้อง "vip" รับได้ 2 คน) เขียนโปรแกรมทดสอบที่จำลองผู้ใช้เชื่อมต่อเข้าสองห้องพร้อมกันหลาย
   คน แล้วพิสูจน์ด้วยการพิมพ์ log ว่า **ข้อความจากห้อง "general" ไม่มีทางไปถึงผู้ใช้ในห้อง "vip" เลย** และการ
   จำกัดจำนวนคนต่อห้องทำงานถูกต้องอย่างอิสระจากกัน (คำใบ้: `HashMap<String, RoomState>` ที่ใช้ค้นหา/สร้างห้อง
   ตอนมีคนเข้าใหม่ต้องมีตัวป้องกัน concurrent modification ของมันเองด้วย — ต้องเลือกระหว่าง `std::sync::Mutex`
   กับ `tokio::sync::Mutex` สำหรับ `HashMap` ชั้นนอกนี้อีกชั้นหนึ่ง ลองพิจารณาว่า critical section ของการ
   "ค้นหาหรือสร้างห้องใหม่" มี `.await` อยู่ข้างในหรือไม่ตามกฎหัวข้อ 50.6)

## สรุป

บทนี้ปิดท้าย mini-arc เรื่อง async/Tokio (Part 46-50) ด้วยการตอบคำถามที่ค้างอยู่ตลอดมา: **เครื่องมือ
synchronization จาก Part 38-39 (`std::sync::Mutex`/`RwLock`/`mpsc`) ยังใช้ได้ในโค้ด async หรือไม่** — คำตอบคือ
**"ใช้ได้ แต่ต้องรู้ขีดจำกัดของมันให้แม่น"**: `std::sync::Mutex::lock()` เป็น **blocking call** ที่ยึด OS
worker thread ทั้งเส้นไว้ระหว่างรอ ตรงกับคำเตือนเรื่อง "blocking ข้างใน async" ที่ Part 48 วางไว้พอดี — เรา
พิสูจน์ด้วยการรันจริงว่าการถือ `MutexGuard` ข้าม `.await` point อันตรายได้สองระดับ: บางครั้ง compiler จับได้
ตั้งแต่ compile time (error `Send`) บางครั้งจับไม่ได้เลยและนำไปสู่ **deadlock จริงที่ค้างตลอดไป**
(`timeout` exit code 124) ซึ่งร้ายแรงกว่า blocking call ธรรมดามาก เพราะมันปิดวงจรที่ไม่มีทางคลายออกเองได้

เราเรียน `tokio::sync::Mutex`/`RwLock` เป็นฝาแฝด async ของ Part 39 ที่ `.lock().await`/`.read().await`/
`.write().await` **คืนสิทธิ์ให้ executor** แทนการบล็อก thread และวาง **กฎการตัดสินใจ** ที่สำคัญที่สุดของบทนี้:
ใช้ `std::sync::Mutex` เมื่อ critical section สั้นไม่มี `.await` (เร็วกว่าและใช้ได้ดีแม้ในโค้ด async) ใช้
`tokio::sync::Mutex` เมื่อต้องถือ lock ข้าม `.await` จริง ๆ เท่านั้น — จากนั้นเรียน `tokio::sync::mpsc` เป็น
ฝาแฝด async ของ Part 38 ที่ `.send().await` ให้ backpressure จริงแบบไม่บล็อก thread และรีเมค worker pool
pattern โดยใช้กฎการตัดสินใจข้อนี้พิสูจน์ตัวเองผ่านการต้องใช้ `tokio::sync::Mutex` ห่อ `Receiver` ที่แบ่งกันใช้

สามหัวข้อถัดมาคือ primitive ใหม่ที่ไม่มีคู่เทียบใน `std::sync` เลย: `tokio::sync::oneshot` แก้คำใบ้ "reply
channel" ของ Part 38 อย่างสมบูรณ์ด้วย pattern ถาม-ตอบครั้งเดียวแบบ actor, `tokio::sync::broadcast` แก้คำใบ้
"chat broadcast" ของ Part 49 อย่างสมบูรณ์ด้วยการกระจายข้อความให้หลาย subscriber โดยไม่ต้องเขียน
`Arc<Mutex<Vec<Sender>>>` เองเลย, และ `tokio::sync::watch` สำหรับสถานการณ์ที่สนใจแค่ค่าล่าสุด — ปิดท้ายด้วย
`tokio::sync::Semaphore` ที่ทำงานแบบ RAII เหมือน `MutexGuard` (เชื่อมกับ Part 6) สำหรับจำกัดจำนวนงานพร้อมกัน
และตัวอย่างใหญ่ที่รวมทุกอย่างเข้าด้วยกัน: chat server เวอร์ชัน production-quality ที่พิสูจน์ด้วยการรันจริงผ่าน
TCP ว่า `broadcast` กระจายข้อความถูกต้องและ `Semaphore` บังคับเพดานจำนวน connection ได้จริง

ด้วยบทนี้ mini-arc เรื่อง async/Tokio (Part 46-50) ก็จบครบสมบูรณ์: จาก `async`/`.await` เบื้องต้น ผ่าน
`Future`/executor ภายใน ผ่าน Tokio runtime/tasks/networking จนถึงชุดเครื่องมือ synchronization ที่ออกแบบมา
สำหรับโลก async โดยเฉพาะ — **Part 51 (Atomics และ Lock-free Programming)** จะพาลงไปให้ลึกกว่านี้อีกขั้นหนึ่ง
กลับไปที่โลกของ OS thread เดิม (Part 37-40) เพื่อตอบคำถามที่ Part 39 ทิ้งไว้แบบผิวเผิน (ตอนอธิบายว่า `Arc<T>`
ใช้ atomic increment/decrement ที่ "เร็วกว่า Mutex เพราะทำงานที่ระดับคำสั่ง CPU ตัวเดียวโดยตรง") — Part 51
จะอธิบายว่า atomic operation ทำงานอย่างไรจริง ๆ ที่ระดับ CPU, `std::sync::atomic::{AtomicUsize, AtomicBool,
...}` ใช้งานอย่างไร, memory ordering (`Relaxed`, `Acquire`, `Release`, `SeqCst`) คืออะไรและทำไมสำคัญ, และ
เขียนโครงสร้างข้อมูล lock-free เบื้องต้นได้เอง — เป็นการปิดวง concurrency ทั้งหมดของโมดูล 3 ด้วยความเข้าใจใน
ระดับที่ลึกที่สุดว่าเครื่องมือ synchronization ทุกตัวที่เรียนมา (`Mutex`, `RwLock`, `Arc`, และ `tokio::sync`
ทั้งหมดในบทนี้) ถูกสร้างขึ้นมาจากอะไรที่ระดับล่างที่สุดกันแน่

---

**Part ก่อนหน้า:** [Tokio: I/O และ Networking](part-049-tokio-networking.md) | **Part ถัดไป:** [Atomics และ Lock-free Programming](part-051-atomics-lockfree.md)
