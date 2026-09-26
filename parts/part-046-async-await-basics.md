# Part 46: Async/Await เบื้องต้น

> โมดูล: ระดับสูง (Advanced) | ระดับ: สูง | เวลาโดยประมาณ: 260 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้อย่างแม่นยำว่าทำไม **OS thread** (ที่เรียนเต็มรูปแบบใน Part 37-40) ถึง "หนักเกินไป" สำหรับงานบางประเภท
  โดยเฉพาะงาน **I/O-bound** ที่ต้องรอเครือข่าย/ดิสก์/timer จำนวนมาก ๆ พร้อมกัน (เป็นพัน ๆ งาน) และรู้ว่า
  **async/await** คือคำตอบของ Rust สำหรับปัญหานี้ — โมเดล concurrency แบบที่สองที่ต่างจาก thread โดยพื้นฐาน
- อธิบายได้ว่า `async fn` **ไม่ได้รันโค้ดข้างในทันทีที่ถูกเรียก** แต่คืนค่าเป็น **`Future`** — ค่าที่แทน "งานที่จะ
  ให้ผลลัพธ์ในอนาคต" — และเข้าใจว่าไม่มีอะไรเกิดขึ้นจริงจนกว่า Future นั้นจะถูก **poll** โดย executor
- ใช้ `.await` ได้อย่างเข้าใจกลไกจริง: มันคือจุดที่ฟังก์ชัน async **ยอมสละการควบคุม (yield)** กลับไปให้ executor
  ชั่วคราว เพื่อให้งานอื่นได้ทำงานต่อในระหว่างที่ตัวเองยังรอผลลัพธ์ไม่เสร็จ — เข้าใจว่านี่คือ **cooperative
  concurrency** ที่ต่างจาก **preemptive** ของ OS thread โดยสิ้นเชิง
- อธิบายได้ว่าทำไม `async fn main()` เปล่า ๆ ใช้งานไม่ได้ (เจอ error **E0752** จริง) และทำไมต้องมี **runtime/
  executor** มาช่วย "ขับ" (drive) Future เสมอ — เขียน executor ขนาดจิ๋วด้วยมือเองเพื่อพิสูจน์แนวคิดนี้ให้เห็นจริง
  ก่อนที่ Part 48 จะแนะนำ Tokio ซึ่งเป็น runtime ระดับ production
- เข้าใจแนวคิด **state machine** ที่อยู่เบื้องหลัง `async fn`/`.await`: compiler แปลงฟังก์ชัน async ทั้งฟังก์ชัน
  ให้กลายเป็น struct/enum ที่ implement trait `Future` โดยอัตโนมัติ (คู่กับแนวคิด closure desugaring จาก
  Part 24.9) และเชื่อมโยงได้ว่าทำไมปัญหานี้เกี่ยวข้องกับ **self-referential struct** และ **`Pin`** ที่ Part 23
  เกริ่นทิ้งไว้
- เขียนโปรแกรมที่เปรียบเทียบการรัน Future **แบบ sequential** (ทีละตัว) กับ **แบบ concurrent** (พร้อมกันด้วย
  `futures::join!`) เห็นตัวเลขเวลาจริงที่ต่างกันชัดเจน และเข้าใจว่าทำไม Part 48 จะใช้ `tokio::join!`/
  `tokio::spawn` เพื่อประโยชน์แบบเดียวกันในระดับ production

## ความรู้ที่ต้องมีมาก่อน

- **Part 37-40 (Threads, Channels, Mutex/Arc, Send/Sync)**: บทนี้ตั้งต้นจากคำถามที่ Part 37 เกริ่นไว้ตรง ๆ ว่า
  `thread::spawn` สร้าง **OS thread จริง** ที่มีต้นทุนไม่ใช่ศูนย์ (stack memory ระดับ KB-MB, การลงทะเบียนกับ OS
  scheduler) และปิดท้ายด้วยการพูดถึง "async runtime อย่าง Tokio สำหรับงาน I/O-bound" เป็นทางเลือก — Part 46-50
  คือคำตอบเต็มรูปแบบของคำเกริ่นนั้น เราจะใช้ตัวเลขต้นทุนจริงจาก Part 37 (การ spawn thread 10,000 ตัวใช้เวลา
  หลักร้อย ms เทียบกับ loop ธรรมดาที่ใช้เวลาหลักสิบ ns) เป็นจุดเริ่มต้นของการอธิบายว่าทำไมต้องมีโมเดล concurrency
  อีกแบบสำหรับงานที่มีจำนวนมากแต่แต่ละงานเบา — บทนี้จะ**ไม่สอน**เนื้อหาเรื่อง Mutex/channel ซ้ำ แต่จะใช้ความเข้าใจ
  เรื่อง "การรอ", "การบล็อก", และ `Send`/`Sync` เป็นฐานอ้างอิงตลอดบท
- **Part 24 (Closures)**: หัวใจสำคัญที่สุดของบทนี้คือการเทียบ `async` block กับ closure — ทั้งสองเป็น "แม่พิมพ์"
  ที่สร้างค่าไว้ก่อน แล้วค่อย "กระตุ้น" ให้ทำงานทีหลัง (closure กระตุ้นด้วยการเรียก `()`, async block กระตุ้นด้วย
  `.await`) และหัวข้อ 24.9 ที่อธิบายว่า closure จริง ๆ คือ struct ที่ compiler สร้างให้ ก็คือแนวคิดเดียวกันเป๊ะกับ
  ที่บทนี้จะใช้อธิบาย `async fn` ว่าถูกแปลงเป็น state machine โดย compiler
- **Part 23 (Lifetimes ขั้นสูง)**: หัวข้อ 23.9 ของ Part 23 เกริ่นปัญหา **self-referential struct** ไว้ตรง ๆ ว่า
  "พบมากที่สุดในโค้ดเกี่ยวกับ `async`/`Future` ซึ่งคอมไพเลอร์สร้าง state machine ที่เป็น self-referential โดย
  ธรรมชาติ" และเกริ่น `Pin<T>` ว่าเป็นเครื่องมือที่ "การันตีว่าค่าจะไม่ถูก move ไปไหนอีกหลังจากถูก pin ไว้แล้ว" —
  บทนี้จะปิดคำใบ้นั้นให้ครบในระดับแนวคิด (รายละเอียดเต็มรูปแบบของ `Pin`/`Future` trait คือ Part 47)
- Part นี้เป็น**บทแรกของ mini-arc async/Tokio** ในโมดูล 3: Part 46 (async/await พื้นฐาน — บทนี้) → Part 47
  (Futures/executors เจาะกลไกภายใน) → Part 48 (Tokio runtime และ tasks) → Part 49 (Tokio I/O/networking) →
  Part 50 (async synchronization primitives ด้วย `tokio::sync`) คุณยังไม่ต้องรู้จัก Tokio มาก่อนเพื่ออ่านบทนี้ —
  บทนี้จะใช้ crate `futures` (เฉพาะฟังก์ชัน `block_on` และ macro `join!`) เป็น "ตัวขับ" ขั้นต่ำสุดเพื่อสาธิต
  แนวคิดเท่านั้น ไม่ใช่ตัวที่จะใช้ทำงานจริงในโปรเจกต์ — Part 48 จะแนะนำ Tokio ซึ่งเป็น runtime ระดับ production
  ที่ใช้จริงในอุตสาหกรรม

## เนื้อหา

### 46.1 ทวนโจทย์จาก Part 37: OS Thread หนักเกินไปสำหรับอะไร

ก่อนจะเข้าเนื้อหาของบทนี้ ต้องย้อนกลับไปที่คำถามซึ่ง Part 37 เปิดทิ้งไว้แต่ยังไม่ได้ตอบเต็มรูปแบบ: **"ถ้า OS
thread มีต้นทุนสูง แล้วเราจะรันงานพร้อมกันเป็นจำนวนมาก ๆ ได้อย่างไร"**

จำได้จากหัวข้อ 37.2 ว่า `std::thread::spawn()` สร้าง **thread จริงระดับ operating system** — โมเดลที่เรียกว่า
**"1:1 threading"** (หนึ่ง thread ของภาษาตรงกับหนึ่ง thread ของ OS เป๊ะ ๆ) การสร้าง OS thread แต่ละตัวมีต้นทุนจริง
ที่ไม่ใช่ศูนย์: ต้องขอ **stack memory** ก้อนใหม่จาก OS (ปกติหลาย KB ไปจนถึงไม่กี่ MB ต่อ thread) ต้องให้ **OS
scheduler** มาลงทะเบียน thread ใหม่ไว้ และการสลับ (**context switch**) ระหว่าง thread ก็มีต้นทุนของมันเอง — และ
กับดักข้อ 7 ของ Part 37 ก็พิสูจน์ตัวเลขจริงให้เห็นแล้ว: spawn thread 10,000 ตัวเพื่อทำงานที่เบามาก (บวกเลขครั้ง
เดียวต่อ thread) ใช้เวลา **411 ms** เทียบกับ loop ธรรมดาที่ใช้เวลาแค่ **57 นาโนวินาที** — ต่างกันหลาย**ล้านเท่า**

ตัวเลขนั้นสำคัญมากสำหรับบทนี้ เพราะมันนำไปสู่คำถามในสถานการณ์ที่พบบ่อยที่สุดในโลกจริงของ backend/network
programming: **เว็บเซิร์ฟเวอร์ที่ต้องรับ connection พร้อมกันหลักพันหรือหลักหมื่น connection** — สมมติว่าเซิร์ฟเวอร์
หนึ่งตัวต้องจัดการ 10,000 การเชื่อมต่อ HTTP พร้อมกัน แต่ละ connection ส่วนใหญ่ของเวลาคือ **การรอ** (รอข้อมูลจาก
เครือข่ายเข้ามา, รอ database ตอบกลับ, รอไฟล์ถูกอ่านจากดิสก์) ไม่ใช่การคำนวณหนัก ๆ ด้วย CPU

ถ้าเราแก้ปัญหานี้แบบเดียวกับ Part 37 (spawn 1 thread ต่อ 1 connection) จะเกิดอะไรขึ้น:

- **หน่วยความจำ**: 10,000 thread × stack ขนาดหลาย MB ต่อ thread (ค่า default ของ Rust คือ 2 MB ต่อ thread บน
  หลายระบบ) = อาจถึงหลักหมื่น MB (หลาย GB) แค่สำหรับ stack เปล่า ๆ ที่ส่วนใหญ่ **ไม่ได้ใช้เต็มเลย** เพราะ
  connection ส่วนใหญ่กำลัง **นอนรอ** อยู่
- **OS scheduler**: ต้องบริหารจัดการ (context switch) ระหว่าง 10,000 thread ซึ่งส่วนใหญ่ไม่มีงานจริงให้ทำ (แค่รอ)
  — เวลาที่ scheduler ใช้ตัดสินใจว่า "ตาไหนจะได้รัน" กลายเป็น overhead ที่ไม่ได้สร้างประโยชน์อะไรเลย
- **ต้นทุนการสร้าง/ทำลาย**: ทุกครั้งที่ connection ใหม่เข้ามาต้อง spawn thread ใหม่ (ต้นทุนแบบที่กับดักข้อ 7 ของ
  Part 37 พิสูจน์ไว้) และทุกครั้งที่ connection ปิดต้องทำลาย thread นั้นไป

**ปัญหาที่แท้จริงคือ**: OS thread ถูกออกแบบมาให้เหมาะกับงานที่ "มีเนื้องานจริงให้ทำ" (CPU-bound) ไม่ใช่งานที่
"ส่วนใหญ่นอนรอเฉย ๆ" (I/O-bound) — เมื่อ thread ส่วนใหญ่ใช้เวลาแค่ **บล็อกรอ I/O** ต้นทุนของการมี thread เต็มรูป
(stack ใหญ่, ลงทะเบียนกับ OS scheduler เต็มรูปแบบ) กลายเป็นสิ่งที่**เสียเปล่า**เกือบทั้งหมด

#### ทางออก: หน่วยงานที่ "เบากว่า Thread" และ "ยอมสละการควบคุมได้เอง"

แนวคิดของ async/await คือการสร้างหน่วยงานที่เบากว่า thread มาก เรียกว่า **"task"** — task ไม่ใช่ OS thread แต่
เป็นแค่ **ข้อมูล** ที่บอกว่า "งานนี้ทำไปถึงไหนแล้ว" (state) พร้อมกลไกที่ทำให้ task นั้น **หยุดพัก (suspend)** ตัวเอง
ได้เองตอนที่ต้องรอ I/O แล้ว **กลับมาทำงานต่อ (resume)** ได้เมื่อ I/O นั้นพร้อมแล้ว — โดยที่ตัว task เองไม่ต้องมี
OS thread ของตัวเองรอทำงานอยู่เปล่า ๆ ระหว่างพัก

**จำนวนมากแค่ไหน**: task เบากว่า thread มาก (ขนาดในหน่วยความจำระดับสิบ-ร้อยไบต์ต่อ task เทียบกับหลาย MB ต่อ
thread) ทำให้โปรแกรมเดียวสามารถมี task เป็น**หลักแสนหรือหลักล้าน**พร้อมกันได้สบาย ๆ บนเครื่องธรรมดา — สิ่งที่เป็น
ไปไม่ได้เลยถ้าใช้ OS thread หนึ่งตัวต่องาน

**หลักการสำคัญ**: task จำนวนมากเหล่านี้ถูก **multiplex** (จัดสรรให้ทำงานสลับกัน) ลงบน OS thread จำนวน**น้อยกว่า
มาก** (มักเท่ากับหรือใกล้เคียงจำนวน CPU core จริง เช่น 4-16 thread) — เมื่อ task หนึ่งกำลังรอ I/O มันจะ "สละคิว"
ให้ thread ที่มันอยู่ไปรัน task อื่นที่พร้อมทำงานต่อได้ทันที ไม่ต้องเสีย OS thread ทั้งตัวไปกับการนอนรอเฉย ๆ

นี่คือสิ่งที่ Part 37 เกริ่นไว้ตรง ๆ ว่า "Tokio ใช้ thread pool ขนาดเล็กจำนวนคงที่ (worker thread) รัน 'งาน' (task)
น้ำหนักเบาจำนวนมากได้พร้อมกัน แทนที่จะสร้าง OS thread ใหม่สำหรับทุกงาน" — บทนี้และอีก 4 บทถัดไปคือการอธิบายกลไก
เบื้องหลังคำกล่าวนั้นให้ครบทุกขั้น เริ่มจากไวยากรณ์ภาษา (`async`/`.await`) ในบทนี้ ไปจนถึง Tokio runtime จริงใน
Part 48

#### ตารางเปรียบเทียบ: OS Thread vs Async Task

| มิติ | OS Thread (`std::thread::spawn`, Part 37) | Async Task (`async fn`/`.await`, บทนี้) |
|---|---|---|
| ใครจัดสรร CPU ให้ | OS scheduler (preemptive — สลับให้โดยไม่ถามความสมัครใจ) | Executor ในระดับ library (cooperative — ต้อง `.await` เพื่อสละคิวเอง) |
| ขนาดหน่วยความจำต่อหน่วย | หลาย KB ถึงไม่กี่ MB (stack เต็มรูป) | หลักสิบ-ร้อยไบต์ (แค่ state ที่จำเป็น) |
| จำนวนที่รันพร้อมกันได้จริง | หลักพัน (ถูกจำกัดด้วยหน่วยความจำ/OS) | หลักแสนถึงหลักล้าน |
| เหมาะกับงานประเภท | CPU-bound (คำนวณหนัก ใช้ CPU เต็ม ๆ) | I/O-bound (รอเครือข่าย/ดิสก์/timer เป็นส่วนใหญ่) |
| ถูก "ตัดจบกลางคัน" ได้ไหม | ได้ (preemption โดย OS ทุกเมื่อ) | ไม่ได้ — หยุดได้แค่ที่จุด `.await` ที่ตัวเองยินยอมเท่านั้น |
| ใครสร้าง/ทำลาย | OS (ผ่าน syscall) | Executor (แค่จัดการ struct ในหน่วยความจำ ไม่มี syscall) |

แถวสุดท้ายของตารางนี้คือกุญแจสำคัญที่สุดของทั้งบท และจะเป็นหัวข้อหลักของ 46.3 — **async task ไม่มีวันถูกตัดจบ
กลางคันโดยไม่สมัครใจ** ต่างจาก OS thread ที่ OS มีสิทธิ์แทรกและสลับได้ทุกเมื่อ

### 46.2 `async fn`: ประกาศฟังก์ชัน Async — และสิ่งที่มันไม่ได้ทำ

เริ่มจากไวยากรณ์ที่ง่ายที่สุด: การเติมคำว่า `async` หน้า `fn`

```rust
async fn say_hello() {
    println!("สวัสดี จาก async fn!");
}
```

ดูผิวเผินคล้ายฟังก์ชันธรรมดาทุกอย่าง แต่มีความต่างพื้นฐานที่สำคัญที่สุดของทั้งบทซ่อนอยู่: **การเรียก `say_hello()`
ไม่ทำให้ body ข้างในรันเลยแม้แต่นิดเดียว** ลองพิสูจน์ด้วยโค้ดจริง:

```rust
async fn say_hello() {
    println!("สวัสดี จาก async fn!");
}

fn main() {
    let future = say_hello(); // แค่สร้าง Future ยังไม่รัน body เลย
    println!("เรียก say_hello() แล้ว แต่ยังไม่เห็นข้อความ 'สวัสดี' เลย");
    println!("ตัวแปร future มี type ที่ compiler สร้างขึ้นเอง (ซ่อนอยู่หลัง `impl Future<Output = ()>`)");
    drop(future); // future ถูกทิ้งไปโดยไม่เคยถูก poll เลยแม้แต่ครั้งเดียว
    println!("จบ main() แล้ว — 'สวัสดี' ไม่ถูกพิมพ์เลยตลอดการรันโปรแกรมนี้");
}
```

รันจริงได้ผลลัพธ์นี้ (ยืนยันได้ทุกครั้งที่รัน — ไม่ใช่พฤติกรรมไม่แน่นอนแบบ race condition):

```
เรียก say_hello() แล้ว แต่ยังไม่เห็นข้อความ 'สวัสดี' เลย
ตัวแปร future มี type ที่ compiler สร้างขึ้นเอง (ซ่อนอยู่หลัง `impl Future<Output = ()>`)
จบ main() แล้ว — 'สวัสดี' ไม่ถูกพิมพ์เลยตลอดการรันโปรแกรมนี้
```

สังเกตให้ชัด: **คำว่า "สวัสดี จาก async fn!" ไม่ถูกพิมพ์เลยตลอดการรันโปรแกรมนี้** ทั้งที่เราเรียก `say_hello()`
ไปแล้วจริง ๆ ในบรรทัดแรกของ `main` — นี่ไม่ใช่บั๊ก แต่คือพฤติกรรมที่ตั้งใจออกแบบมาแบบนี้ **100%**

#### `async fn` คืนค่าเป็น `Future` ไม่ใช่ผลลัพธ์ตรง ๆ

เมื่อคุณเขียน:

```rust
async fn compute_answer() -> i32 {
    42
}
```

Rust ไม่ได้ตีความมันว่าเป็นฟังก์ชันที่ `-> i32` ตรง ๆ ทั้งที่ signature ดูเหมือนจะบอกแบบนั้น — สิ่งที่ compiler
ทำจริง ๆ คือแปลง signature นี้ให้เทียบเท่ากับ (แนวคิดคร่าว ๆ ยังไม่ใช่ syntax จริงที่เขียนได้ตรง ๆ ในบทนี้ — Part 47
จะเจาะกลไกเต็มรูปแบบ):

```text
fn compute_answer() -> impl Future<Output = i32> {
    // ... struct ที่ compiler สร้างขึ้นเอง ซึ่ง implement Future ...
}
```

พูดให้ตรงที่สุด: **`compute_answer()` คืนค่าเป็นสิ่งที่ implement trait `Future<Output = i32>`** — ไม่ใช่ `i32`
ตรง ๆ เลย ค่านั้นคือ**ตัวแทนของ "งานที่ยังไม่ได้ทำ"** (a computation that will produce a result eventually) เก็บ
ไว้เฉย ๆ ในหน่วยความจำ ยังไม่มีอะไรทำงานจริงจนกว่าจะมีใครมา **"ขับ" (drive)** มันให้ทำงาน

`Future` เป็น trait ที่กำหนดไว้ใน `std::future` มี method หลักตัวเดียวคือ `poll` (รายละเอียดเต็มรูปแบบรวมถึง
signature จริงของ `poll` คือเนื้อหาหลักของ Part 47 — ตอนนี้ให้เข้าใจแค่ระดับแนวคิดว่า **"poll" คือการถามว่า
'พร้อมให้ผลลัพธ์หรือยัง'** ถ้ายังไม่พร้อมจะได้คำตอบว่า `Pending` ถ้าพร้อมแล้วจะได้ `Ready(ผลลัพธ์)`) — สิ่งที่
ต้อง**ไปเรียก `poll` ซ้ำ ๆ จนกว่าจะได้ `Ready`**เรียกว่า **executor** ซึ่งเราจะพบมันในหัวข้อ 46.4

**สรุปให้จำง่าย**: `async fn` คือ "แม่พิมพ์" ที่สร้าง Future ค่าหนึ่งกลับมาทันทีที่ถูกเรียก แต่ Future นั้นไม่ทำ
อะไรเลยด้วยตัวเอง — มันต้องรอให้ **มีใครสักคน** (executor) มา poll มันซ้ำ ๆ ถึงจะได้ทำงานจริง เปรียบได้กับ
ใบสั่งซื้อสินค้า (Future) ที่แค่เขียนขึ้นมาไม่ได้ทำให้สินค้ามาส่งถึงบ้านทันที ต้องมีคนไปดำเนินการตามใบสั่งนั้น
(executor) จริง ๆ ก่อนถึงจะเกิดผล

### 46.3 `.await`: จุดที่ Task ยอม "สละคิว" กลับไปให้ Executor

ทีนี้มาถึงส่วนที่สอง — `.await` คือกลไกที่ทำให้ Future หนึ่งตัว **"ขับ" Future อีกตัวหนึ่งจากข้างใน** ลองดูตัวอย่าง
ที่ใช้ `.await` จริง:

```rust
async fn say_hello() {
    println!("สวัสดี จาก async fn!");
}

async fn greet_twice() {
    say_hello().await; // (1) สร้าง Future จาก say_hello() แล้ว .await มันทันที
    say_hello().await; // (2) เหมือนกัน — เรียกซ้ำอีกครั้ง
}
```

`.await` เขียนต่อท้ายค่าที่ implement `Future` (ในที่นี้คือค่าที่ `say_hello()` คืนมา) — มันทำสามอย่างพร้อมกันในทาง
แนวคิด:

1. **ส่งให้ executor poll** Future นั้น
2. ถ้าได้ `Pending` (ยังไม่พร้อม) — **หยุดการทำงานของฟังก์ชัน async ปัจจุบันตรงจุดนี้ทันที** และคืนการควบคุมกลับ
   ไปให้ executor เพื่อให้ executor ไปทำงานอื่นที่พร้อมกว่าได้ (นี่คือ "สละคิว" หรือ **yield**)
3. เมื่อ executor เห็นว่า Future นั้นพร้อมแล้ว (ผ่านกลไก waker ที่ Part 47 จะเจาะลึก) มันจะ**กลับมาทำงานต่อจากจุด
   ที่หยุดไว้เป๊ะ ๆ** ไม่ใช่เริ่มจากต้นฟังก์ชันใหม่ — พอ Future ข้างในให้ค่า `Ready(ผลลัพธ์)` `.await` ก็จะ
   "แกะ" ผลลัพธ์นั้นออกมาเป็นค่าธรรมดาให้ใช้ต่อได้เลย (เหมือนที่ `?` แกะ `Ok(T)` ออกมาจาก `Result<T, E>` ใน
   Part 12 — ต่างกันแค่บริบทและกลไกภายใน)

#### `.await` คือหัวใจของคำว่า "Cooperative": Task ไม่มีวันถูกแทรกโดยไม่สมัครใจ

นี่คือความต่างเชิงพื้นฐานที่สุดจาก OS thread ที่เรียนมาใน Part 37 — จำได้ว่า OS thread ถูก **preempt** ได้ทุกเมื่อ
โดย OS scheduler โดยที่โค้ดของเราไม่มีสิทธิ์ปฏิเสธเลย (นี่คือเหตุผลที่ต้องมี `Mutex` ใน Part 39 — เพราะ thread
อาจถูกสลับออกไปกลางขั้นตอนที่ยังไม่เสร็จของการแก้ไขข้อมูลร่วม)

**async task ไม่เป็นแบบนั้นเลย**: task หนึ่งจะทำงาน**รวดเดียวไม่มีการหยุดพัก** ตั้งแต่เริ่ม (หรือตั้งแต่ resume
ครั้งล่าสุด) ไปจนถึงจุด `.await` แรกที่มันเจอ (หรือจนจบฟังก์ชันถ้าไม่มี `.await` เลย) — **ไม่มีใครมาแทรกกลางทางได้
โดยที่ task ไม่ยินยอม** compiler เห็น `.await` ทุกจุดตรง ๆ ในโค้ด ต่างจาก preemption ของ OS ที่มองจากมุมโค้ดแล้ว
"เกิดขึ้นตรงไหนก็ได้"

**ความหมายเชิงปฏิบัติที่สำคัญ**: ถ้า async task หนึ่งเขียน loop คำนวณหนัก ๆ โดย**ไม่มี `.await` เลยสักจุด**
task นั้นจะ**ครองสิทธิ์ thread ที่มันรันอยู่ไปตลอดจนกว่าจะจบ loop** — task อื่นทุกตัวที่ถูก multiplex อยู่บน
thread เดียวกันจะ**ต้องรอ**จนกว่า task นี้จะเจอ `.await` หรือจบการทำงาน (เราจะเห็นผลกระทบของเรื่องนี้ชัด ๆ ใน
กับดักที่ 5 ของบทนี้) — นี่คือเหตุผลที่ async runtime อย่าง Tokio (Part 48) เหมาะกับงาน **I/O-bound** เป็นหลัก
ไม่ใช่ **CPU-bound** เพราะงาน CPU-bound หนัก ๆ ที่ไม่มี `.await` จะ "บล็อก" task อื่นทั้งหมดบน thread นั้นไปเลย
(Tokio มีเครื่องมือรับมือกรณีนี้ เช่น `spawn_blocking` — แต่นั่นเป็นเนื้อหา Part 48)

#### เปรียบเทียบ `.await` กับ `.join()` จาก Part 37

ทั้งสองดู "รอ" คล้ายกัน แต่กลไกเบื้องหลังต่างกันโดยสิ้นเชิง — ตารางนี้เทียบให้เห็นชัด:

| | `handle.join()` (Part 37) | `future.await` (บทนี้) |
|---|---|---|
| รอให้ใครทำงานเสร็จ | OS thread อื่น | Future อื่น (อาจเป็น task อื่น หรือแค่ค่าที่ยังไม่พร้อม) |
| ระหว่างรอ thread ปัจจุบันทำอะไร | **บล็อกสนิท** ไม่ทำอะไรเลย (OS อาจเอา CPU ไปให้ thread อื่นแทน แต่ thread ที่เรียก join ค้างอยู่ที่บรรทัดนั้น) | **สละคิว** ให้ executor ไปรัน task อื่นบน thread เดียวกันต่อได้ทันที |
| ใครเป็นคนจัดการการสลับงาน | OS scheduler (นอก process ของเรา) | Executor (โค้ด library ที่รันอยู่ใน process เดียวกัน) |
| เรียกได้ที่ไหน | ที่ไหนก็ได้ในโค้ดธรรมดา | ต้องอยู่ใน `async fn` หรือ `async` block เท่านั้น (ดูหัวข้อ 46.8 กับดักที่ 2) |

### 46.4 ทำไมต้องมี Runtime: `async fn main()` เปล่า ๆ ใช้งานไม่ได้

ตอนนี้เราเห็นแล้วว่า `async fn` คืนค่าเป็น Future ที่ต้องมีใคร "ขับ" มันด้วยการ poll ซ้ำ ๆ — คำถามตามมาตามธรรมชาติ
คือ **แล้วใครเป็นคนขับ Future ตัวแรกของโปรแกรมล่ะ?** ลองคิดแบบไร้เดียงสาก่อน: จะเกิดอะไรขึ้นถ้าเราลองประกาศ
`main` เป็น `async fn` ตรง ๆ?

```rust
async fn main() {
    println!("สวัสดี");
}
```

ลองคอมไพล์โค้ดนี้จริง ๆ:

```
error[E0752]: `main` function is not allowed to be `async`
 --> src/main.rs:1:1
  |
1 | async fn main() {
  | ^^^^^^^^^^^^^^^ `main` function is not allowed to be `async`
```

**เหตุผลที่ Rust ปฏิเสธตรง ๆ ตั้งแต่ compile time**: ถ้า `main` เป็น `async fn` มันจะคืนค่าเป็น Future ตัวหนึ่ง —
แต่ **ตัวโปรแกรม (ระดับ operating system) ไม่รู้จัก `Future` เลย** OS รู้จักแค่ "ฟังก์ชัน `main` ปกติที่ต้องรันจน
จบแล้วคืน exit code" — ไม่มีใคร (ไม่มี OS, ไม่มี Rust runtime อัตโนมัติ) มาคอย poll `Future` ที่ `main` คืนออกมาให้
Rust **ไม่มี runtime ในตัวภาษาที่ทำงานให้อัตโนมัติแบบภาษาอื่น** (จะเปรียบเทียบเรื่องนี้เต็มรูปแบบในหัวข้อ 46.9)
ดังนั้น compiler จึงปฏิเสธตั้งแต่ต้นทาง ก่อนที่จะปล่อยให้เขียนโปรแกรมที่ "ไม่มีวันมีอะไรเกิดขึ้นจริง" ออกมาได้

**สิ่งที่ต้องมีคือ "ตัวขับ" (executor) ที่เขียนด้วยโค้ด Rust ธรรมดา** — มันคือฟังก์ชันที่รับ Future มา แล้ว `poll`
มันซ้ำ ๆ ใน loop จนกว่าจะได้ `Ready` แล้วเคลื่อนที่ไปหาผลลัพธ์นั้น จาก `main()` แบบธรรมดา (ไม่ใช่ `async fn`)
เขียนได้ประมาณนี้ในแนวคิด:

```text
fn main() {
    let my_future = some_async_fn(); // สร้าง Future ไว้ก่อน ยังไม่รัน
    let result = run_executor_until_done(my_future); // ให้ executor ขับจนจบ แล้วได้ผลลัพธ์
    println!("{result:?}");
}
```

`run_executor_until_done` (ชื่อสมมติ) คือสิ่งที่ Part 48 จะให้ Tokio (`#[tokio::main]` หรือ `Runtime::block_on`)
มาเป็นเวอร์ชันที่สมบูรณ์แบบ production-grade ให้ใช้ — แต่บทนี้จะเขียน**เวอร์ชันจิ๋วด้วยตัวเอง** (หัวข้อ 46.6) เพื่อ
พิสูจน์ว่ามันไม่ใช่ "มายากล" อะไรเลย เป็นแค่โค้ดธรรมดาที่ทำสิ่งที่อธิบายไว้ตรง ๆ

#### ทางลัดขั้นต่ำสุดสำหรับบทนี้: `futures::executor::block_on`

ก่อนจะเขียน executor ด้วยตัวเอง มาดูวิธีที่ง่ายที่สุดในการ "ขับ" Future หนึ่งตัวจนจบก่อน — crate `futures`
(crate ทางการที่ทีม Rust async-wg ดูแล ไม่ใช่ Tokio แต่เป็นแค่ชุดเครื่องมือพื้นฐานสำหรับทำงานกับ `Future`) มี
ฟังก์ชัน `futures::executor::block_on` ที่ทำหน้าที่นี้ตรง ๆ:

```rust
use futures::executor::block_on;

async fn say_hello() {
    println!("สวัสดี จาก async fn!");
}

fn main() {
    println!("ก่อนเรียก block_on");
    block_on(say_hello()); // ตอนนี้ future ถูก poll จริง ๆ จนจบ
    println!("หลังเรียก block_on — ตอนนี้ 'สวัสดี' ถูกพิมพ์ไปแล้ว");
}
```

รันจริงได้:

```
ก่อนเรียก block_on
สวัสดี จาก async fn!
หลังเรียก block_on — ตอนนี้ 'สวัสดี' ถูกพิมพ์ไปแล้ว
```

ตอนนี้ "สวัสดี จาก async fn!" ถูกพิมพ์จริงแล้ว — ต่างจากหัวข้อ 46.2 ที่ไม่มีอะไรถูกพิมพ์เลย เพราะครั้งนี้เรามี
`block_on` เป็นตัวขับที่ **poll Future ที่ `say_hello()` คืนมาไปเรื่อย ๆ จนกว่าจะได้ `Ready`** ก่อนที่จะ return
กลับมาให้ `main` ทำงานบรรทัดต่อไป

> **ข้อควรระวังสำคัญ**: `block_on` เป็นเครื่องมือ**สาธิตแนวคิด**ที่ดีมาก แต่ **ไม่ใช่ตัวเลือกที่เหมาะกับโปรเจกต์
> จริง** เพราะมันเป็น executor แบบง่ายที่สุด (single-threaded, ไม่มี I/O reactor สมบูรณ์แบบ, ไม่มี task scheduler
> ที่ซับซ้อน) — บทนี้ใช้มันเพราะสอนแนวคิด `async`/`.await` ได้ถูกต้องครบถ้วน โดยไม่ต้องดึงความซับซ้อนทั้งหมดของ
> Tokio เข้ามาก่อนที่จะถึงเวลาที่เหมาะสม (Part 48) — เมื่อถึง Part 48 ตัวอย่างเดียวกันในบทนี้จะถูกเขียนใหม่ด้วย
> `#[tokio::main]` และ `tokio::join!`/`tokio::spawn` ของจริง

### 46.5 `async` Block: Future แบบไม่มีชื่อ — เปรียบเทียบตรงกับ Closure จาก Part 24

จำได้จาก Part 24 ว่า closure คือ "แม่พิมพ์แบบไม่มีชื่อ" ที่ผลิตค่ากลับมาตอน**ถูกเรียก** (`()`) — `async` block
คือแนวคิดคู่ขนานเป๊ะ ๆ ในโลกของ Future: มันคือ **"แม่พิมพ์แบบไม่มีชื่อที่ผลิต Future"** — Future ที่ผลิตค่ากลับมา
ตอน**ถูก `.await`**

Syntax ของมันคือ `async { ... }` — เขียนตรง ๆ ตรงไหนก็ได้ที่ต้องการ Future โดยไม่ต้องประกาศเป็น `async fn`
แยกไว้ก่อน:

```rust
use futures::executor::block_on;

fn main() {
    // closure: ผลิตค่าตอน "ถูกเรียก" ()
    let add_closure = |a: i32, b: i32| a + b;
    println!("closure สร้างแล้ว ยังไม่ถูกเรียก");
    let sum = add_closure(2, 3);
    println!("closure ถูกเรียกแล้ว ผลลัพธ์ = {sum}");

    // async block: ผลิตค่าตอน "ถูก await"
    let x = 10;
    let y = 20;
    let future = async move {
        println!("async block กำลังรัน (ตอนนี้ถูก poll แล้ว)");
        x + y
    };
    println!("async block สร้างแล้ว (เป็นค่า Future) ยังไม่ถูก await");
    let sum2 = block_on(future);
    println!("async block ถูก await แล้ว ผลลัพธ์ = {sum2}");
}
```

รันจริงได้:

```
closure สร้างแล้ว ยังไม่ถูกเรียก
closure ถูกเรียกแล้ว ผลลัพธ์ = 5
async block สร้างแล้ว (เป็นค่า Future) ยังไม่ถูก await
async block กำลังรัน (ตอนนี้ถูก poll แล้ว)
async block ถูก await แล้ว ผลลัพธ์ = 30
```

สังเกตความคล้ายกันของสองบรรทัด "สร้างแล้ว ยังไม่ถูก..." — ทั้งสองพิสูจน์หลักการเดียวกัน: **การสร้างแม่พิมพ์ไม่ทำให้
โค้ดข้างในรัน ต้องมีการ "กระตุ้น" อย่างชัดเจนเสียก่อน** (`()` สำหรับ closure, `.await` หรือ `block_on(...)` สำหรับ
Future)

#### `move` ใน Async Block: หลักการเดียวกันกับ Part 24.6 และ Part 37.5 เป๊ะ

สังเกตว่าตัวอย่างข้างบนใช้ `async move { ... }` — คำว่า `move` ตรงนี้ทำหน้าที่**เหมือนกันเป๊ะ**กับ `move` ที่ใช้
กับ closure ใน Part 24.6 และกับ `thread::spawn(move || ...)` ใน Part 37.5: มันบอกให้ตัวแปรที่ถูก capture (ในที่นี้
คือ `x` และ `y`) **ถูกยึด ownership เข้าไปในตัว Future ทั้งก้อน** แทนที่จะแค่ยืม (borrow) มาจาก scope รอบตัวมัน

เหตุผลที่ต้องระวังเรื่องนี้เป็นพิเศษในบริบท async: Future ที่สร้างจาก `async` block อาจถูก `.await` ที่จุดใดก็ได้
ในอนาคต (อาจถูกส่งไปเก็บไว้ในตัวแปรก่อน ค่อย await ทีหลัง หรือถูก spawn เป็น task แยก — เรื่องหลังนี้เป็นเนื้อหา
Part 48) ถ้ามันแค่ **ยืม** ตัวแปรจาก scope ที่มันถูกสร้างขึ้นมา แล้ว scope นั้นจบไปก่อนที่ Future จะถูก await จริง
— ก็จะเกิดปัญหาแบบเดียวกับที่ Part 37.5 พิสูจน์ด้วย error **E0373** ตอนพยายามส่ง closure ที่ borrow ตัวแปร local
เข้า `thread::spawn` — หลักการ "ต้องเป็น `'static`" (ไม่ยืมอะไรที่อายุสั้นกว่า Future เอง) ก็ใช้กับ Future ที่จะ
ถูกส่งไปให้ executor จัดการในลักษณะเดียวกัน (รายละเอียดเรื่อง `'static` bound ของ Future ที่ต้อง spawn จริง ๆ
คือเนื้อหา Part 48 — ตอนนี้ให้จำแค่ว่าหลักการเดียวกับ Part 37.5 นำมาใช้ได้ตรง ๆ)

### 46.6 เขียน Executor ขั้นต่ำสุดด้วยตัวเอง: พิสูจน์ว่า `.await` ไม่ใช่มายากล

ถึงเวลาเปิดฝากล่องดูว่า `block_on` ทำอะไรอยู่จริง ๆ — เราจะเขียน **executor ขนาดจิ๋วของตัวเอง** ที่ทำสิ่งเดียวกัน
กับ `futures::executor::block_on` ในระดับแนวคิด (ง่ายกว่ามาก ไม่มีการจัดการ I/O reactor ที่ซับซ้อน — **นี่คือ
เครื่องมือสำหรับสอนเท่านั้น** Part 48 จะให้ Tokio ซึ่งเป็น executor ระดับ production ที่สมบูรณ์กว่านี้มาก)

ก่อนอื่นต้องรู้จัก 3 ชิ้นส่วนสำคัญที่ trait `Future` ใช้งาน (Part 47 จะเจาะกลไกเต็มรูปแบบ — ที่นี่แค่พอให้ใช้งานได้):

- **`Poll<T>`**: enum สองแบบ คือ `Poll::Ready(T)` (พร้อมแล้ว มีผลลัพธ์) และ `Poll::Pending` (ยังไม่พร้อม)
- **`Context`**: ตัวที่ `poll` รับเข้ามาเป็นพารามิเตอร์ ใช้เก็บ **`Waker`** (ตัวที่ Future ใช้ "บอก" executor ว่า
  "ฉันพร้อมให้ poll ใหม่อีกครั้งแล้วนะ" — Part 47 จะอธิบายว่า executor จริงใช้กลไกนี้ยังไง)
- **`Pin<&mut Self>`**: type ของ `self` ใน `poll` (ไม่ใช่ `&mut self` ธรรมดา) — เหตุผลที่ต้องเป็น `Pin` คือหัวข้อ
  46.7 ถัดไปเลย ตอนนี้ให้มองมันเป็น "reference พิเศษที่การันตีว่าค่าจะไม่ถูกย้ายที่อยู่ในหน่วยความจำอีก"

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll, RawWaker, RawWakerVTable, Waker};

// Future แบบง่ายที่สุดที่เขียนมือ: Pending ในการ poll ครั้งแรก แล้ว Ready ในครั้งที่สอง
// จำลอง "จุดที่ต้องหยุดรอ" แบบเดียวกับที่ .await สร้างให้เราโดยอัตโนมัติ
struct YieldOnce {
    yielded: bool,
}

impl Future for YieldOnce {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        if self.yielded {
            Poll::Ready(())
        } else {
            self.yielded = true;
            // บอก executor ว่า "พร้อมให้ poll ใหม่ได้อีกครั้งทันที" (ในสถานการณ์จริงเช่นรอ I/O
            // เสร็จ ตัว driver ของ I/O จะเรียก wake() ตอนข้อมูลพร้อมจริง ๆ ไม่ใช่ทันทีแบบนี้)
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}

async fn counting_task(label: &str) -> u32 {
    println!("{label}: เริ่มทำงาน");
    YieldOnce { yielded: false }.await; // จุดที่ทำให้ async fn นี้ "หยุดพักได้" หนึ่งจุด
    println!("{label}: ทำงานต่อหลัง yield กลับมาแล้ว");
    42
}

// Waker แบบ "ไม่ทำอะไรเลย" (no-op) — ใช้ได้เพราะ tiny_block_on วน poll ซ้ำเองอยู่แล้วโดยไม่รอสัญญาณจริง
fn noop_raw_waker() -> RawWaker {
    fn no_op(_: *const ()) {}
    fn clone_waker(_: *const ()) -> RawWaker {
        noop_raw_waker()
    }
    let vtable = &RawWakerVTable::new(clone_waker, no_op, no_op, no_op);
    RawWaker::new(std::ptr::null(), vtable)
}

/// ตัวขับ Future แบบขั้นต่ำสุดที่เขียนมือเอง — "แค่พอให้เห็นภาพ" ว่า async fn ทำงานได้อย่างไรจริง ๆ
/// เบื้องหลัง executor ของจริง (Part 48: Tokio) ซับซ้อนกว่านี้มาก: มันไม่ spin loop วนถามซ้ำแบบนี้
/// (ซึ่งกิน CPU 100% ไปเปล่า ๆ ระหว่างรอ) แต่ใช้ OS mechanism (epoll/kqueue/IOCP) ทำให้ thread หลับจริง
/// จนกว่า waker จะถูกเรียกจากภายนอกเมื่อ I/O พร้อมจริง ๆ — Part 47 จะเจาะกลไก Future/Waker เต็มรูปแบบ
fn tiny_block_on<F: Future>(future: F) -> F::Output {
    // Box::pin ปักหมุด future ไว้บน heap อย่างปลอดภัย (ไม่ต้องใช้ unsafe เอง) ทำให้ได้ Pin<Box<F>>
    // ที่ poll ต้องการ — เหตุผลที่ poll ต้องการ Pin เลยจะอธิบายในหัวข้อ 46.7
    let mut future = Box::pin(future);

    // unsafe เพียงจุดเดียวในไฟล์นี้: การสร้าง Waker จาก RawWaker ต้องมั่นใจว่า vtable ที่ให้ไป
    // ทำตามสัญญาของ Waker::from_raw จริง ๆ (ที่นี่ทุกฟังก์ชันใน vtable ไม่ทำอะไรเลยโดยตั้งใจ ปลอดภัย)
    let waker = unsafe { Waker::from_raw(noop_raw_waker()) };
    let mut cx = Context::from_waker(&waker);

    loop {
        match future.as_mut().poll(&mut cx) {
            Poll::Ready(value) => return value,
            Poll::Pending => {
                // executor จริงจะให้ thread หลับจนกว่า waker ถูกเรียก แต่ตัวอย่างนี้ทำง่ายสุด:
                // วนถามซ้ำทันที (ใช้ CPU สูงมาก ห้ามทำแบบนี้ในโค้ดจริงเด็ดขาด — สอนแนวคิดเท่านั้น)
                std::hint::spin_loop();
            }
        }
    }
}

fn main() {
    let result = tiny_block_on(counting_task("งาน-A"));
    println!("ผลลัพธ์สุดท้าย: {result}");
}
```

รันจริงได้:

```
งาน-A: เริ่มทำงาน
งาน-A: ทำงานต่อหลัง yield กลับมาแล้ว
ผลลัพธ์สุดท้าย: 42
```

**อ่านลำดับการทำงานทีละก้าว** เพื่อยึดแนวคิด `.await`/`poll` ให้แน่น:

1. `tiny_block_on` เรียก `future.as_mut().poll(&mut cx)` ครั้งแรก — นี่คือการ poll `counting_task("งาน-A")`
   Future ทั้งก้อน
2. body ของ `counting_task` เริ่มรัน: พิมพ์ `"งาน-A: เริ่มทำงาน"` แล้วเจอ `YieldOnce { ... }.await`
3. `.await` ตรงนี้ทำให้ compiler แทรกโค้ดที่ **poll `YieldOnce` นั้นเข้าไปข้างใน** — `YieldOnce::poll` ครั้งแรก
   ตั้ง `yielded = true`, เรียก `wake_by_ref()`, แล้วคืน `Poll::Pending`
4. เพราะ `YieldOnce` คืน `Pending`, `.await` จึงทำให้ **`counting_task` ทั้งฟังก์ชันคืน `Poll::Pending` ออกไปด้วย**
   (สละคิว — หยุดตรงจุดนี้ ยังไม่ทำบรรทัดถัดไปเลย) กลับไปที่ `tiny_block_on`
5. `tiny_block_on` เห็น `Pending` จึง `spin_loop()` แล้ว **poll ใหม่อีกครั้ง** (รอบที่สอง)
6. รอบนี้ `counting_task` **ไม่เริ่มจากบรรทัดแรกใหม่** (ไม่พิมพ์ `"เริ่มทำงาน"` อีกครั้ง!) แต่ไปต่อจากจุดที่ค้างไว้:
   poll `YieldOnce` อีกครั้ง คราวนี้ `yielded` เป็น `true` แล้ว จึงคืน `Poll::Ready(())`
7. `.await` แกะ `Ready(())` ออกมา, body ทำงานต่อ: พิมพ์ `"ทำงานต่อหลัง yield กลับมาแล้ว"` แล้ว return `42`
8. `counting_task` ทั้งก้อนคืน `Poll::Ready(42)` กลับไปให้ `tiny_block_on` ซึ่งได้ผลลัพธ์และ return ออกจาก loop

**ข้อสังเกตที่สำคัญที่สุดจากลำดับนี้**: ในรอบ poll ที่สอง `counting_task` **จำได้**ว่าตัวเองทำงานไปถึงไหนแล้ว
(ไม่พิมพ์ "เริ่มทำงาน" ซ้ำ) — มันต้อง**เก็บ state** ไว้ที่ไหนสักแห่งข้าม poll แต่ละรอบ คำถามคือ **state นั้นเก็บ
อยู่ตรงไหน?** คำตอบคือหัวข้อถัดไป — และนี่คือจุดที่แนวคิด **state machine** เข้ามา

### 46.7 State Machine เบื้องหลัง `async fn`: เชื่อมกับ Closure Desugaring (Part 24.9) และ Self-Referential Struct (Part 23.9)

#### ทวนความจำ: Closure คือ Struct ที่ Compiler สร้างให้ (Part 24.9)

Part 24.9 เปิดฝากล่องของ closure ให้เห็นว่า `|x| x + captured_value` จริง ๆ แล้วไม่ใช่ "เวทมนตร์" อะไรเลย —
compiler แค่สร้าง **struct** ที่มี field เก็บตัวแปรที่ capture มา แล้ว implement trait `Fn`/`FnMut`/`FnOnce`
ให้กับ struct นั้น (ที่มี method `call` เขียน body ของ closure ไว้ข้างใน)

**`async fn`/`.await` ใช้กลไกเดียวกันเป๊ะ ในระดับที่ใหญ่กว่า**: compiler แปลง `async fn` ทั้งฟังก์ชันให้เป็น
**struct หรือ enum** ที่ implement trait `Future` — แต่ต่างจาก closure ตรงที่ struct ของ `async fn` ต้องเก็บ
**ตำแหน่งที่ทำงานไปถึงแล้ว** (เพราะ Future หนึ่งตัวถูก `poll` **หลายครั้ง** ก่อนจะจบ ในขณะที่ closure ธรรมดาถูก
เรียกจบในครั้งเดียว) — โครงสร้างที่เหมาะกับการเก็บ "ตำแหน่งที่ทำงานไปถึงแล้ว" ก็คือ **enum ที่มี 1 variant ต่อ
1 จุดพัก (suspension point)** ในฟังก์ชัน

ลองดูตัวอย่างที่เขียนด้วยมือเพื่อจำลองสิ่งที่ compiler ทำให้กับ `counting_task` จากหัวข้อ 46.6 (ยังใช้ `YieldOnce`
เดิม):

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

struct YieldOnce {
    yielded: bool,
}

impl Future for YieldOnce {
    type Output = ();
    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        if self.yielded {
            Poll::Ready(())
        } else {
            self.yielded = true;
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}

// เวอร์ชันที่เขียนด้วยมือ เทียบเท่ากับ:
//
// async fn counting_task(label: String) -> u32 {
//     println!("{label}: เริ่มทำงาน");
//     YieldOnce { yielded: false }.await;
//     println!("{label}: ทำงานต่อหลัง yield กลับมาแล้ว");
//     42
// }
//
// สังเกตว่า enum มี 1 variant ต่อ "จุดพัก" (suspension point) ในฟังก์ชัน (ที่นี่มีจุดพักเดียวคือ
// ตอน .await ตัว YieldOnce) บวกกับ variant เริ่มต้นและจบ — ตัวแปร label ต้องถูกเก็บไว้ใน struct
// เพราะมันต้อง "มีชีวิตอยู่ข้าม await point" (ใช้ทั้งก่อนและหลัง yield)
enum CountingTaskState {
    Start { label: String },
    WaitingYield { label: String, inner: YieldOnce },
    Done,
}

struct CountingTaskFuture {
    state: CountingTaskState,
}

fn counting_task_manual(label: String) -> CountingTaskFuture {
    CountingTaskFuture {
        state: CountingTaskState::Start { label },
    }
}

impl Future for CountingTaskFuture {
    type Output = u32;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<u32> {
        loop {
            match &mut self.state {
                CountingTaskState::Start { label } => {
                    println!("{label}: เริ่มทำงาน");
                    let label = label.clone();
                    self.state = CountingTaskState::WaitingYield {
                        label,
                        inner: YieldOnce { yielded: false },
                    };
                    // วนกลับไป match ใหม่ทันที (ไม่ return) เพราะยังมีงานให้ทำต่อในรอบ poll เดียวกัน
                }
                CountingTaskState::WaitingYield { label, inner } => {
                    // `inner` เป็น field ของ struct ที่ไม่มี self-reference ใด ๆ (เป็น Unpin เอง
                    // โดย auto trait) จึงสร้าง Pin<&mut YieldOnce> ได้อย่างปลอดภัยด้วย Pin::new
                    // ตรงไปตรงมา ไม่ต้องใช้ unsafe เลยในกรณีนี้
                    match Pin::new(inner).poll(cx) {
                        Poll::Ready(()) => {
                            println!("{label}: ทำงานต่อหลัง yield กลับมาแล้ว");
                            self.state = CountingTaskState::Done;
                            return Poll::Ready(42);
                        }
                        Poll::Pending => return Poll::Pending,
                    }
                }
                CountingTaskState::Done => {
                    panic!("poll ถูกเรียกอีกครั้งหลังจาก Future จบไปแล้ว — ห้ามทำแบบนี้")
                }
            }
        }
    }
}

fn main() {
    use futures::executor::block_on;
    let result = block_on(counting_task_manual("งาน-B".to_string()));
    println!("ผลลัพธ์จากเวอร์ชันมือเขียน: {result}");
}
```

รันจริงได้ผลลัพธ์เหมือนกับเวอร์ชัน `async fn` ในหัวข้อ 46.6 เป๊ะ:

```
งาน-B: เริ่มทำงาน
งาน-B: ทำงานต่อหลัง yield กลับมาแล้ว
ผลลัพธ์จากเวอร์ชันมือเขียน: 42
```

**ย้ำอีกครั้งเพื่อความชัดเจน**: นี่คือ**แนวคิด**ของสิ่งที่ compiler ทำให้ ไม่ใช่โค้ดที่ compiler สร้างจริงตัวต่อ
ตัว (คุณไม่มีวันเห็น source code จริงที่ compiler สร้าง เพราะมันทำงานในระดับ MIR ภายใน ไม่ใช่ source Rust) แต่
โครงสร้าง **"enum ที่มี 1 variant ต่อจุดพัก"** ตรงกับความเป็นจริงของกลไกเบื้องหลัง — คุณไม่จำเป็นต้องเขียนโค้ดแบบ
นี้ด้วยมือเองในชีวิตจริงเลย (แค่เขียน `async fn`/`.await` แบบปกติ compiler จัดการให้หมด) แต่การเห็นมันครั้งหนึ่ง
ช่วยไขข้อสงสัยหลายอย่างที่จะเจอต่อไปได้อย่างมาก

#### ทำไม Local Variable ถึง "มีชีวิตอยู่ข้าม `.await`" ได้: เพราะมันถูกเก็บไว้ใน State

จากตัวอย่างข้างบน สังเกตว่า `label` (ตัวแปรที่มาจาก parameter ของฟังก์ชัน) ต้องถูก**เก็บไว้เป็น field ของทุก
variant ที่ยังต้องใช้มัน** (`Start` และ `WaitingYield` ทั้งคู่มี field `label`) — เหตุผลคือมันถูกใช้**ทั้งก่อน
และหลัง** จุดพัก ในขณะที่ตัวแปรที่ใช้แล้วจบไปก่อนจุดพัก (ไม่มีในตัวอย่างนี้ แต่จะเห็นในหัวข้อ 46.8) ไม่จำเป็นต้อง
ถูกเก็บไว้เลย

นี่คือคำตอบของคำถามที่มือใหม่หลายคนสงสัย: **"ทำไมตัวแปร local ใน async fn ยังใช้ได้หลัง `.await`"** — เพราะมัน
ไม่ใช่ตัวแปร "บน stack" แบบฟังก์ชันธรรมดาที่หายไปทันทีที่ฟังก์ชัน return แต่ถูก **compiler ย้ายไปเก็บเป็น field
ของ struct/enum ที่แทน Future ทั้งก้อน** ตัว struct นั้นมีชีวิตอยู่ตราบใดที่ Future ยังไม่ถูก drop — ข้าม poll
กี่รอบก็ได้ ไม่หายไปไหน

**ผลที่ตามมาอีกข้อ**: ขนาดของ Future (ในหน่วยความจำ) **โตขึ้นตามความซับซ้อนของฟังก์ชัน** — ยิ่งมีตัวแปร local
ที่ต้องมีชีวิตข้าม `.await` หลายจุดมากเท่าไหร่ (โดยเฉพาะถ้ามีหลาย `.await` และตัวแปรใหญ่ ๆ ที่ต้องเก็บไว้ในหลาย
variant พร้อมกัน) struct ที่ compiler สร้างก็ใหญ่ขึ้นตามไปด้วย — เทียบได้ตรงกับที่ Part 24.12 อธิบายไว้สำหรับ
closure ว่า "capture ยิ่งมาก ยิ่งใหญ่" หลักการเดียวกันเป๊ะ เพียงแค่ตอนนี้ใหญ่ขึ้นได้มากกว่าเพราะมีหลาย variant
ที่อาจซ้อนทับกันได้ (compiler จะพยายาม optimize ให้ variant ต่าง ๆ ใช้พื้นที่ซ้อนทับกัน (union-like layout) ให้
มากที่สุดเท่าที่ทำได้ แต่ก็มีขีดจำกัด)

#### ปิดคำใบ้จาก Part 23.9: ทำไม `Future::poll` ต้องรับ `Pin<&mut Self>` ไม่ใช่ `&mut Self` ธรรมดา

ถึงเวลาปิดคำใบ้ที่ Part 23.9 ทิ้งไว้ตรง ๆ ว่า: *"ปัญหา self-referential struct ... พบมากที่สุดในโค้ดเกี่ยวกับ
`async`/`Future` ซึ่งคอมไพเลอร์สร้าง state machine ที่เป็น self-referential โดยธรรมชาติ"*

ทวนจาก Part 23.9: **self-referential struct** คือ struct ที่ field หนึ่ง**ชี้กลับไปยังอีก field ของตัวเอง**
ปัญหาคือถ้า struct ทั้งก้อนถูก **move** ไปอยู่ตำแหน่งความจำอื่น field ที่ชี้กลับเข้าไปข้างในจะยังจำ**ตำแหน่งเดิม**
ไว้ ทำให้ชี้ผิดที่ทันที (dangling) — lifetime ธรรมดาแก้ปัญหานี้ไม่ได้เลยเพราะมันเป็นเรื่อง "ตำแหน่งในหน่วยความจำ"
ไม่ใช่เรื่อง "อายุ"

**ทำไม async state machine เป็น self-referential โดยธรรมชาติ**: ลองนึกภาพฟังก์ชัน async ที่มีบรรทัดแบบนี้:

```text
async fn example() {
    let data = String::from("ข้อมูล");
    let borrowed: &str = &data;       // ยืม data ไว้
    some_other_future().await;         // จุดพัก — ต้องเก็บทั้ง data และ borrowed ไว้ข้าม await นี้
    println!("{borrowed}");            // ใช้ borrowed หลัง await
}
```

เมื่อ compiler แปลงฟังก์ชันนี้เป็น struct ทั้ง `data` และ `borrowed` ต้องถูกเก็บเป็น field ของ struct เดียวกัน (ตาม
หลักการในหัวข้อก่อนหน้า) — แต่ `borrowed` เป็น **reference ที่ชี้ไปยัง `data` ซึ่งเป็น field อื่นของ struct
เดียวกันนั้นเอง** — นี่คือ**นิยามตรงตัว**ของ self-referential struct จาก Part 23.9 ไม่ผิดเพี้ยนเลยแม้แต่นิดเดียว!
และปัญหาเดิมก็เกิดขึ้น: ถ้า Future (struct) ทั้งก้อนถูก **move** (เช่น ถูกส่งจากฟังก์ชันหนึ่งไปอีกฟังก์ชันหนึ่ง,
ถูกใส่ลง `Vec`, หรือถูก `Box::pin` แล้วย้ายกล่องนั้น) `borrowed` ที่จำตำแหน่งเดิมของ `data` ไว้จะกลายเป็น
dangling ทันที

**คำตอบของ Rust คือ `Pin`**: `Pin<P>` (โดยที่ `P` มักเป็น `&mut T` หรือ `Box<T>`) คือ wrapper ที่**การันตีในระดับ
type system** ว่า **ค่าที่มันชี้ไปจะไม่ถูก move ออกจากตำแหน่งนั้นอีกเลย** ตราบใดที่มันยัง pin อยู่ (ยกเว้น type
ที่ implement `Unpin` — type ธรรมดาส่วนใหญ่ที่ไม่มี self-reference จะได้ `Unpin` แบบอัตโนมัติ ซึ่งทำให้ `Pin`
ไม่มีผลจำกัดอะไรเพิ่มกับพวกมันเลย นี่คือเหตุผลที่ตัวอย่างในหัวข้อ 46.7 ใช้ `Pin::new(inner)` แบบตรง ๆ ได้กับ
`YieldOnce` โดยไม่ต้อง unsafe — เพราะ `YieldOnce` ไม่มี self-reference จึงเป็น `Unpin` โดย auto trait)

เมื่อ `poll` รับ `self: Pin<&mut Self>` แทน `&mut self` ธรรมดา มันคือการบอกว่า **"เมื่อคุณเริ่ม poll Future นี้
แล้ว มันจะไม่ถูกย้ายที่อยู่ในหน่วยความจำไปไหนอีกจนกว่าจะจบ"** — การันตีนี้ทำให้ self-reference ภายใน state
machine (เช่น `borrowed` ที่ชี้เข้า `data` ใน field เดียวกัน) **ปลอดภัยที่จะมีอยู่ได้จริง** เพราะตำแหน่งของมัน
จะไม่เปลี่ยนไปกลางทาง — นี่คือเหตุผลที่ `Box::pin(future)` ในหัวข้อ 46.6 สำคัญ: มันย้าย Future ไปอยู่บน heap
ครั้งเดียว แล้ว "ปักหมุด" ไว้ตรงนั้นไม่ให้ขยับอีก ก่อนเริ่ม poll รอบแรก

> **ขอบเขตของบทนี้**: กลไกแบบเต็มของ `Pin`/`Unpin` (การ implement `Future` เองสำหรับ struct ที่ self-referential
> จริง ๆ, `pin_project`, ความแตกต่างระหว่าง `Pin<&mut T>` กับ `Pin<Box<T>>` ในรายละเอียด) เป็นเนื้อหาลึกที่ Part 47
> จะเจาะเต็มรูปแบบ — เป้าหมายของหัวข้อนี้คือแค่ **เข้าใจว่าทำไม `Pin` ต้องมีอยู่** และเชื่อมมันกับปัญหา
> self-referential struct ที่เรียนไปแล้วใน Part 23.9 ให้แน่น ซึ่งเพียงพอสำหรับใช้งาน `async`/`.await` ในระดับ
> ผู้ใช้ทั่วไปโดยไม่ต้องเขียน `Future` ด้วยมือเองเลย (แบบที่ Part 48-50 จะทำกับ Tokio ตลอดทาง)

### 46.8 Sequential กับ Concurrent: ทำไมลำดับของ `.await` สำคัญมาก

ตอนนี้เรามาถึงจุดที่จะเห็นประโยชน์จริงของ async ในทางปฏิบัติ — ลองสร้างฟังก์ชันจำลอง **"fetch ข้อมูลจากเซิร์ฟเวอร์
ปลายทาง"** ที่ต้องรอ I/O (ในที่นี้จำลองด้วย `AsyncDelay` ที่รอตามเวลาที่กำหนด แทนการรอเครือข่ายจริง — Part 49
จะใช้ TCP/HTTP จริงแทนที่จุดนี้):

```rust
use futures::executor::block_on;
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};

// AsyncDelay: จำลอง "รอ I/O เสร็จ" แบบไม่บล็อก thread (ต่างจาก std::thread::sleep ที่บล็อกทั้ง
// thread) — วิธีนี้ยังไม่มีประสิทธิภาพเท่า timer จริงของ Tokio (Part 48 ใช้ OS timer แทนการวนถามซ้ำ)
// แต่แสดงแนวคิดที่ถูกต้อง: poll ซ้ำแบบไม่บล็อก จนกว่าเวลาที่กำหนดจะครบ
struct AsyncDelay {
    deadline: Instant,
}

impl AsyncDelay {
    fn new(duration: Duration) -> Self {
        AsyncDelay {
            deadline: Instant::now() + duration,
        }
    }
}

impl Future for AsyncDelay {
    type Output = ();
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        if Instant::now() >= self.deadline {
            Poll::Ready(())
        } else {
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}

// จำลอง "fetch ข้อมูลจากเซิร์ฟเวอร์ปลายทาง" ที่ต้องรอ I/O (เครือข่าย) เป็นเวลาหนึ่ง ๆ
async fn fetch_data(name: &str, delay_ms: u64) -> String {
    AsyncDelay::new(Duration::from_millis(delay_ms)).await;
    format!("ข้อมูลจาก {name}")
}

async fn run_sequential() {
    let start = Instant::now();

    // .await ทีละตัว: ตัวที่สองจะไม่เริ่มทำงานเลยจนกว่าตัวแรกจะ .await จบสมบูรณ์ก่อน
    let a = fetch_data("service-A", 150).await;
    println!("{a}");

    let b = fetch_data("service-B", 150).await;
    println!("{b}");

    let c = fetch_data("service-C", 150).await;
    println!("{c}");

    println!(
        "แบบ sequential .await (ทีละตัว) ใช้เวลารวม: {:?} (คาดว่า ~450ms เพราะ 150+150+150)",
        start.elapsed()
    );
}

fn main() {
    block_on(run_sequential());
}
```

รันจริงได้:

```
ข้อมูลจาก service-A
ข้อมูลจาก service-B
ข้อมูลจาก service-C
แบบ sequential .await (ทีละตัว) ใช้เวลารวม: 450.068899ms (คาดว่า ~450ms เพราะ 150+150+150)
```

ตัวเลข **450 ms** ตรงกับที่คาดไว้เป๊ะ (150 + 150 + 150) — เหตุผลตรงไปตรงมา: `fetch_data("service-A", 150).await`
ต้อง **จบสมบูรณ์ก่อน** โค้ดบรรทัดถัดไป (`fetch_data("service-B", ...)`) จะได้เริ่มทำงานเลยด้วยซ้ำ (ยังไม่ได้แค่
"รอ" — มันยังไม่ถูก**สร้าง**เลยจนกว่าบรรทัดก่อนจะ await จบ) — นี่คือความหมายตรงตัวของคำว่า **"sequential"**: งาน
ที่สองรอให้งานแรกจบสนิทก่อนถึงจะเริ่ม แม้ว่าในความเป็นจริงงานทั้งสองจะ**ไม่มีความเกี่ยวข้องกันเลย** (ไม่ต้องใช้
ผลลัพธ์ของกันและกัน) จึงไม่มีเหตุผลอะไรที่ต้องรอแบบนี้เลยในทางทฤษฎี

#### ใช้ `futures::join!` รันหลาย Future "พร้อมกัน" บน Task เดียว

ทีนี้ลองใช้ macro `futures::join!` ซึ่งรับ Future หลายตัว แล้ว **poll สลับกันไปมา**จนกว่าทุกตัวจะ `Ready` (คล้าย
กับหลักการเดียวกับ `tiny_block_on` ในหัวข้อ 46.6 แต่ทำกับ Future หลายตัวพร้อมกันในรอบ poll เดียว):

```rust
use futures::executor::block_on;
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};

struct AsyncDelay {
    deadline: Instant,
}

impl AsyncDelay {
    fn new(duration: Duration) -> Self {
        AsyncDelay {
            deadline: Instant::now() + duration,
        }
    }
}

impl Future for AsyncDelay {
    type Output = ();
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        if Instant::now() >= self.deadline {
            Poll::Ready(())
        } else {
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}

async fn fetch_data(name: &str, delay_ms: u64) -> String {
    AsyncDelay::new(Duration::from_millis(delay_ms)).await;
    format!("ข้อมูลจาก {name}")
}

async fn run_concurrent() {
    let start = Instant::now();

    // futures::join! รอหลาย future "พร้อมกัน" บน task เดียว โดยสลับกัน poll ทีละนิด
    // ระหว่างที่ future หนึ่งกำลัง "รอ" (Pending) executor จะไปลอง poll future อื่นต่อได้เลย
    // (ยังไม่ใช่ tokio::join! ของ Part 48 แต่แนวคิดของการซ้อนเวลารอกันเหมือนกันเป๊ะ)
    let (a, b, c) = futures::join!(
        fetch_data("service-A", 150),
        fetch_data("service-B", 150),
        fetch_data("service-C", 150),
    );

    println!("{a}");
    println!("{b}");
    println!("{c}");

    println!(
        "แบบ join! (concurrent) ใช้เวลารวม: {:?} (คาดว่า ~150ms เพราะรอซ้อนกัน ไม่ใช่ 450ms)",
        start.elapsed()
    );
}

fn main() {
    block_on(run_concurrent());
}
```

รันจริงได้:

```
ข้อมูลจาก service-A
ข้อมูลจาก service-B
ข้อมูลจาก service-C
แบบ join! (concurrent) ใช้เวลารวม: 150.023968ms (คาดว่า ~150ms เพราะรอซ้อนกัน ไม่ใช่ 450ms)
```

**150 ms เทียบกับ 450 ms — เร็วขึ้น 3 เท่าเป๊ะ** โดยไม่ต้องแก้อะไรกับ `fetch_data` เองเลยแม้แต่นิดเดียว! เหตุผล
เบื้องหลัง: `join!` สร้าง Future ทั้งสามตัวขึ้นมาพร้อมกัน แล้ว poll **สลับกันไปมา** (poll ตัว A ก่อน, ได้
`Pending` เพราะยังไม่ครบ 150ms, สละคิวไปลอง poll ตัว B, ได้ `Pending` เหมือนกัน เพราะเวลาผ่านไปนิดเดียว, ลอง poll
ตัว C, `Pending` เหมือนกัน, กลับไป poll ตัว A ใหม่ ... วนแบบนี้ไปเรื่อย ๆ) — เพราะทั้งสามตัว **"นอนรอ" (Pending)
พร้อมกัน** ในเวลาเดียวกัน (คนละ Future แต่ share เวลาบนนาฬิกาเดียวกัน) เวลารอของทั้งสามจึง **ซ้อนทับกัน**
(overlap) แทนที่จะบวกกันตรง ๆ — เวลารวมจึงเท่ากับ**เวลาที่นานที่สุด**ในกลุ่ม (150ms ของทุกตัวในตัวอย่างนี้พอดี)
ไม่ใช่ผลรวมของทุกตัว

**นี่คือประโยชน์หลักของ async/await ในทางปฏิบัติ**: เมื่องานหลายงาน **ไม่ต้องพึ่งผลลัพธ์ของกันและกัน** (independent)
และแต่ละงานส่วนใหญ่คือ **การรอ** (I/O-bound) การรันแบบ concurrent ด้วย `.await` ที่ถูกจัดลำดับอย่างเหมาะสม (ผ่าน
`join!` หรือเครื่องมือคล้ายกัน) ทำให้**เวลารวมลดลงมหาศาล** โดยไม่ต้องใช้ OS thread เพิ่มแม้แต่ตัวเดียว — ทั้งหมด
เกิดขึ้นบน **task เดียว, thread เดียว** (ในตัวอย่างนี้คือ thread ที่รัน `block_on`)

> **เกริ่น Part 47/48**: `futures::join!` ที่ใช้ในบทนี้เป็นแค่ตัวอย่างพื้นฐานที่สุดของแนวคิด "รอหลาย Future พร้อม
> กัน" — Part 47 จะอธิบายกลไกเบื้องหลัง `join!`/`select!` แบบเต็มรูปแบบ (การ poll หลาย Future ในรอบเดียวทำงาน
> อย่างไรจริง ๆ ในระดับ `Future` trait) และ Part 48 จะแนะนำ `tokio::join!` (ใช้งานคล้ายกันแต่ทำงานบน Tokio
> runtime จริง) และ **`tokio::spawn`** ซึ่งไปได้ไกลกว่า `join!` อีกขั้น: มันแยกงานออกเป็น **task อิสระ** ที่
> executor อาจเอาไปรันบน **thread อื่นในกลุ่ม worker thread** ได้เลย (ใช้ประโยชน์จาก multi-core CPU จริง ๆ) ไม่ใช่
> แค่สลับกัน poll บน thread เดียวแบบ `join!` — บทนี้ปูพื้นแนวคิด "รันพร้อมกันดีกว่ารันทีละตัว" ให้แน่นก่อน ส่วน
> เครื่องมือระดับ production ที่ทำได้ดีกว่านี้อีกคือหน้าที่ของ Part 48

### 46.9 Borrow และ Ownership ข้าม `.await`: เมื่อ Reference ต้อง "รอด" ผ่านจุดพัก

จุดที่ต้องระวังเป็นพิเศษเมื่อเขียน async คือ: **ตัวแปรที่ borrow มา อาจต้อง "มีชีวิตอยู่" ข้ามจุด `.await`** ซึ่ง
ตามหัวข้อ 46.7 หมายความว่ามันจะถูกเก็บเป็น field ของ state machine ไปด้วย — เรื่องนี้**ส่วนใหญ่ไม่มีปัญหาอะไร**
ถ้า borrow นั้นเป็นแค่ reference ธรรมดา:

```rust
use futures::executor::block_on;

async fn slow_step() {}

async fn sum_with_borrow(numbers: &[i32]) -> i32 {
    let mut total = 0;
    for &n in numbers {
        total += n;
        slow_step().await; // มี reference `numbers` (และตัวแปร local `total`) มีชีวิตอยู่ข้าม await
    }
    total
}

fn main() {
    let data = vec![1, 2, 3, 4, 5];
    let result = block_on(sum_with_borrow(&data));
    println!("ผลรวม: {result}");
}
```

รันจริงได้ `ผลรวม: 15` ปกติ — compile ผ่านสบาย ๆ ไม่มีปัญหาอะไรเลย เพราะ `numbers` (parameter แบบ `&[i32]`) มี
lifetime ที่ผูกกับ Future ทั้งก้อนอยู่แล้ว (Future ที่ `sum_with_borrow` คืนมาจะมี lifetime parameter ที่บอกว่า
"Future นี้ต้องไม่มีชีวิตยาวกว่า `numbers`" — เหมือนกับ struct ที่มี field เป็น reference ต้องมี lifetime
parameter แบบที่เรียนมาตั้งแต่ Part 20) — **นี่คือกรณีปกติทั่วไป** ของการ borrow ข้าม `.await` และไม่ต้องคิดอะไร
มากไปกว่าหลักการ borrow checker ที่เรียนมาตลอดหลักสูตร

#### จุดที่ต้องระวังจริง: เมื่อ Guard/Lock ค้างอยู่ข้าม `.await` แล้วต้องส่ง Future นั้นไปทำงานบน Thread อื่น

ปัญหาที่พบบ่อยในโค้ด async จริงเกิดขึ้นเมื่อค่าที่ borrow มาข้าม `.await` เป็น type ที่**ไม่ใช่ `Send`** (ทวนจาก
Part 40: `Send` คือ "ปลอดภัยที่จะย้ายค่าไปทำงานบน thread อื่น") — ตัวอย่างคลาสสิกที่สุดคือ **`std::sync::
MutexGuard`** จาก Part 39 ซึ่ง**ไม่ implement `Send`** (เหตุผลตรงจุดนี้ลึกกว่าที่ Part 39-40 อธิบายไปเล็กน้อย —
สรุปสั้น ๆ ว่า OS mutex บางแพลตฟอร์มกำหนดว่าต้อง unlock บน thread เดียวกับที่ lock ไว้เท่านั้น) — ลองดูโค้ดที่มี
ปัญหานี้ตรง ๆ:

```rust
use futures::executor::block_on;
use std::sync::Mutex;

fn require_send_future<F: std::future::Future + Send>(_f: F) {}

async fn some_async_op() {}

async fn bad_example(data: &Mutex<i32>) {
    let guard = data.lock().unwrap();
    some_async_op().await; // guard (MutexGuard) ยังมีชีวิตอยู่ข้าม await point นี้
    println!("ค่าปัจจุบัน: {}", *guard);
}

fn main() {
    let data = Mutex::new(5);
    let fut = bad_example(&data);
    require_send_future(fut);
    block_on(bad_example(&data));
}
```

```
error: future cannot be sent between threads safely
  --> src/main.rs:18:25
   |
18 |     require_send_future(fut);
   |                         ^^^ future returned by `bad_example` is not `Send`
   |
   = help: within `impl futures::Future<Output = ()>`, the trait `std::marker::Send` is not implemented for `std::sync::MutexGuard<'_, i32>`
note: future is not `Send` as this value is used across an await
  --> src/main.rs:11:21
   |
10 |     let guard = data.lock().unwrap();
   |         ----- has type `std::sync::MutexGuard<'_, i32>` which is not `Send`
11 |     some_async_op().await; // guard (MutexGuard) ยังมีชีวิตอยู่ข้าม await point นี้
   |                     ^^^^^ await occurs here, with `guard` maybe used later
note: required by a bound in `require_send_future`
  --> src/main.rs:5:49
   |
 5 | fn require_send_future<F: std::future::Future + Send>(_f: F) {}
   |                                                 ^^^^ required by this bound in `require_send_future`
```

**อ่าน error นี้ทีละส่วน** — เห็นคุ้น ๆ ไหม? นี่คือ error class **เดียวกันเป๊ะ**กับที่ Part 40 อธิบายไว้ตอนพยายาม
ส่ง `Rc<RefCell<T>>` ข้าม thread (E0277: `Send` ไม่ implement) เพียงแต่ตอนนี้ตัวที่ไม่ `Send` คือ `MutexGuard`
และบริบทเปลี่ยนจาก "ส่งข้าม thread ตรง ๆ" เป็น "Future ที่**อาจ**ถูกส่งข้าม thread เพราะมันค้าง `MutexGuard` ไว้
ข้าม `.await`":

- **`future returned by bad_example is not Send`** — Future ทั้งก้อนที่ `bad_example` สร้างไม่ implement
  `Send` (ทั้งที่ตัวฟังก์ชันดูปกติทุกอย่าง)
- **`the trait Send is not implemented for MutexGuard<'_, i32>`** — ตัวการที่แท้จริงคือ `MutexGuard` ซึ่งเป็น
  field หนึ่งใน state machine ของ `bad_example` (ตามหลักการหัวข้อ 46.7: ตัวแปรที่มีชีวิตข้าม `.await` ถูกเก็บเป็น
  field) — เพราะ struct ที่มี field ไม่ `Send` แม้แต่ตัวเดียว ก็ทำให้ struct ทั้งก้อนไม่ `Send` ไปด้วย (auto
  trait ที่คำนวณแบบ structural ตามที่ Part 40.8 อธิบายไว้)
- **`future is not Send as this value is used across an await`** — compiler ชี้ตรงจุดที่ `guard` "รอด" ข้าม
  `.await` มา และจะถูกใช้อีกครั้งหลังจากนั้น (`guard maybe used later`)

**เหตุผลว่าทำไมเรื่องนี้สำคัญ**: ในโปรแกรมจริงที่ใช้ Tokio (Part 48) executor แบบ multi-thread จะ**ย้าย task
ข้าม worker thread ได้ตลอดเวลา** ระหว่างจุดพักต่าง ๆ (เพื่อกระจายงานให้ทุก CPU core ทำงานสมดุลกัน) — ถ้า Future
ของ task หนึ่งไม่ `Send` (เพราะค้าง `MutexGuard` ไว้ข้าม `.await`) Tokio จะไม่สามารถย้ายมันได้เลย ซึ่งเป็นข้อจำกัด
ที่ยอมรับไม่ได้สำหรับ multi-thread runtime — **นี่คือเหตุผลที่ compiler ต้องปฏิเสธตั้งแต่ compile time** (ผ่าน
bound `F: Future + Send` ที่ `tokio::spawn` กำหนดไว้ ซึ่งเราจำลองด้วย `require_send_future` ในตัวอย่างข้างบน)
ไม่ปล่อยให้โปรแกรมไป crash หรือ deadlock ตอน runtime แบบภาษาอื่น

**วิธีแก้ที่ถูกต้อง**: ปลด lock ก่อนถึงจุด `.await` เสมอ ไม่ว่าจะด้วยการจำกัด scope ของ `guard` ให้แคบลง (ใช้
`{ }` ครอบเฉพาะส่วนที่ต้องใช้ lock) หรือ clone ค่าที่ต้องใช้ออกมาก่อนแล้วปล่อย lock ทันที:

```rust
use futures::executor::block_on;
use std::sync::Mutex;

async fn some_async_op() {}

async fn good_example(data: &Mutex<i32>) {
    let value_copy = {
        let guard = data.lock().unwrap();
        *guard // อ่านค่าออกมาเป็นสำเนา (i32 เป็น Copy) แล้ว guard ถูก drop ทันทีที่ scope นี้จบ
    }; // <- guard ถูกปล่อยตรงนี้ ก่อนถึง .await ข้างล่างเลย
    some_async_op().await; // ตอนนี้ไม่มี MutexGuard ค้างอยู่ข้าม await แล้ว
    println!("ค่าที่อ่านมา: {value_copy}");
}

fn main() {
    let data = Mutex::new(5);
    block_on(good_example(&data));
}
```

โค้ดนี้ compile ผ่านและเป็น `Send` เพราะ `guard` ถูก drop ไปแล้วตั้งแต่ก่อนถึง `.await` — ไม่มี field ที่ไม่ `Send`
เหลืออยู่ใน state machine ข้ามจุดพักนี้เลย (สำหรับกรณีที่ต้อง lock ข้อมูลแล้ว "รอ" อะไรบางอย่างจริง ๆ ระหว่างที่
ถือ lock ไว้ Part 50 จะแนะนำ `tokio::sync::Mutex` ซึ่งเป็น mutex ที่ออกแบบมาให้ปลอดภัยกับ `.await` โดยเฉพาะ —
`MutexGuard` ของมัน**เป็น** `Send`)

### 46.10 เปรียบเทียบกับภาษาอื่น: JavaScript และ Python

`async`/`await` เป็นคำที่หลายภาษาใช้เหมือนกัน (ยืมไวยากรณ์กันไปมา) แต่ **สิ่งที่เกิดขึ้นเบื้องหลังต่างกันมาก** —
เข้าใจความต่างนี้จะช่วยไม่ให้เอาสมมติฐานจากภาษาอื่นมาใช้กับ Rust แบบผิด ๆ

#### JavaScript: Async/Await ผูกติดกับ Event Loop ที่มีอยู่ในตัวภาษาเสมอ

JavaScript (ทั้งใน browser และ Node.js) มี **event loop สร้างมาให้ในตัว runtime อยู่แล้ว** — ทุกโปรแกรม
JavaScript **มี** event loop ทำงานอยู่เสมอโดยไม่ต้องเลือกหรือติดตั้งอะไรเพิ่ม `async function`/`await` เป็นแค่
syntax sugar ที่ทำงานอยู่**บน**ระบบที่มีอยู่แล้วนี้ นอกจากนี้ JavaScript (ในเธรดหลักของแต่ละ context) เป็น
**single-threaded โดยธรรมชาติของภาษาเอง** — ไม่มี CPU-bound parallelism แบบ multi-thread ในโค้ด JavaScript ปกติ
เลย (มี Web Worker/Worker Threads แยกออกไปเป็นกลไกคนละแบบ) `async`/`await` ใน JS จึงแก้ปัญหา "การรอ I/O โดยไม่
บล็อก UI/main thread" อย่างเดียว ไม่ได้เกี่ยวกับการใช้ CPU หลาย core เลย

#### Python: `asyncio` คือ Runtime ที่ต้องเลือกใช้เอง (คล้าย Rust มากกว่า JS)

Python มี syntax `async def`/`await` ในตัวภาษาเหมือนกัน แต่ **ตัวภาษา Python เองไม่มี event loop สร้างมาให้
อัตโนมัติ** — ต้องใช้ **`asyncio`** (module ในตัว standard library แต่ต้องเรียกใช้งานเอง เช่น
`asyncio.run(main())`) หรือ runtime อื่นเช่น `trio`/`curio` มาเป็นตัวขับ coroutine ที่ `async def` สร้าง — โครงสร้าง
นี้**คล้ายกับ Rust มากกว่าที่คล้าย JavaScript**: ภาษามีแค่ syntax แต่ต้อง "เลือก" runtime มาขับเอง (ต่างจาก Rust
แค่ตรงที่ Python มี `asyncio` เป็นตัวเลือกที่ได้รับการสนับสนุนอย่างเป็นทางการใน standard library ในขณะที่ Rust
ไม่มี runtime ไหนอยู่ใน `std` เลยแม้แต่ตัวเดียว)

#### Rust: ไม่มี Runtime ในตัวภาษาเลย — ต้อง "เลือก" เอาเอง 100%

นี่คือความต่างที่ใหญ่ที่สุดและสำคัญที่สุดที่ต้องเข้าใจให้แน่น: **Rust ไม่มี async runtime อยู่ใน `std` เลย**
`std::future::Future` เป็นแค่ **trait** (สัญญากลางที่ทุก runtime ตกลงใช้ร่วมกัน) — ไม่มี event loop ไม่มี
executor ไม่มี I/O reactor ให้มาโดยอัตโนมัติเหมือน JavaScript หรือมาให้เลือกใช้ใน standard library เหมือน Python
`asyncio` — **คุณต้องเลือก crate ภายนอกมาเป็น runtime เอง 100%** (Tokio เป็นตัวเลือกที่นิยมที่สุดในอุตสาหกรรมและ
เป็นตัวที่หลักสูตรนี้จะสอนใน Part 48 แต่ก็มีตัวเลือกอื่น เช่น `async-std`, `smol`)

**เหตุผลเชิงปรัชญาว่าทำไม Rust เลือกทำแบบนี้**: สอดคล้องกับปรัชญาการออกแบบภาษาที่เห็นมาตลอดหลักสูตร — Rust ไม่
บังคับ garbage collector ให้ทุกโปรแกรม (Part 6-7 ให้ ownership แทน) และก็ไม่บังคับ async runtime ตัวใดตัวหนึ่งให้
ทุกโปรแกรมเช่นกัน โปรแกรม embedded ที่รันบนไมโครคอนโทรลเลอร์ไม่มี OS เลย อาจต้องการ executor ที่เล็กจิ๋วมาก ๆ
(ไม่มี thread pool ไม่มี I/O reactor เต็มรูป) ในขณะที่เว็บเซิร์ฟเวอร์ขนาดใหญ่ต้องการ executor ที่ซับซ้อนเต็มรูป
พร้อม work-stealing scheduler — การแยก **trait (`Future`)** ออกจาก **runtime ที่ทำให้ trait นั้นมีผลจริง**
ทำให้ Rust ครอบคลุมได้ทั้งสองสถานการณ์สุดขั้วนี้ด้วยภาษาเดียวกัน โดยที่ผู้เขียนโค้ดแต่ละคนเลือก runtime ที่เหมาะ
กับงานของตัวเองได้เอง — นี่คือเหตุผลที่บทนี้ต้องเขียน `tiny_block_on` ขึ้นมาเองในหัวข้อ 46.6 (เพื่อพิสูจน์ว่าทำได้
จริง ไม่ใช่ทฤษฎีลอย ๆ) และเหตุผลที่ Part 48 มีอยู่เพื่อสอน Tokio ให้เต็มรูปแบบ

#### ตารางเปรียบเทียบสรุป

| | JavaScript | Python (`asyncio`) | Rust |
|---|---|---|---|
| Runtime มาให้ในตัวภาษา/แพลตฟอร์มหรือไม่ | มี (event loop ในตัว JS engine เสมอ) | ไม่มีในตัวภาษา แต่มีใน standard library (`asyncio`) | **ไม่มีเลยทั้งในภาษาและ `std`** |
| ต้องเลือก runtime เองไหม | ไม่ต้อง (มีให้แล้วเสมอ) | ต้องเรียก `asyncio.run(...)` แต่เป็นตัวเลือกมาตรฐานตัวเดียวที่ใช้กันแทบทั้งหมด | ต้องเลือก crate เอง 100% (Tokio, async-std, smol, หรือเขียนเอง) |
| Threading model พื้นฐาน | Single-threaded (per context) โดยธรรมชาติของภาษา | Single-threaded event loop (มี GIL ผูกอยู่ด้วย) | Executor เลือกได้ทั้ง single-thread หรือ multi-thread work-stealing (Tokio รองรับทั้งคู่) |
| `Future`/`Promise`/coroutine เริ่มทำงานทันทีที่สร้างไหม | Promise เริ่มทำงานทันที (executor function รันทันที) | coroutine **ไม่**เริ่มทำงานจนกว่าจะ `await`/schedule (คล้าย Rust) | **ไม่เริ่มจนกว่าจะถูก `.await`/poll** (ตรงกับหัวข้อ 46.2) |

สังเกตแถวสุดท้าย: จุดที่ Rust กับ Python `asyncio` **เหมือนกัน** (coroutine/Future ไม่ทำงานจนกว่าจะถูกกระตุ้น) แต่
**ต่างจาก JavaScript Promise** ซึ่งฟังก์ชันที่ส่งให้ `new Promise(fn)` รันทันทีตอนสร้าง Promise เลย — ถ้าคุณมี
พื้นฐาน JavaScript มาก่อน นี่คือจุดที่ต้อง**ปรับสมมติฐานใหม่**ให้ตรงกับ Rust (และ Python) ไม่ใช่คาดหวังพฤติกรรม
แบบ Promise

### 46.11 ตัวอย่างโลกจริง: ประกอบหน้าแดชบอร์ดลูกค้าจากสามแหล่งข้อมูล

มาปิดเนื้อหาของบทนี้ด้วยตัวอย่างที่ใหญ่ขึ้นและใกล้เคียงสถานการณ์จริงมากขึ้น — ระบบเว็บที่ต้องประกอบหน้า **แดชบอร์ด
ลูกค้า** จากการเรียกสามไมโครเซอร์วิสที่แยกจากกันโดยสิ้นเชิง: ข้อมูลโปรไฟล์ผู้ใช้, ประวัติการสั่งซื้อ, และสินค้า
แนะนำ — ทั้งสามงาน**ไม่ต้องพึ่งผลลัพธ์ของกันและกันเลย** (ตัวอย่างคลาสสิกของงานที่เหมาะกับการรัน concurrent)

> **หมายเหตุสำคัญ**: ตัวอย่างนี้ยังใช้ `AsyncDelay`/`futures::executor::block_on` เป็นตัวจำลอง เพราะบทนี้ยังไม่ได้
> สอน Tokio — **Part 48 จะเขียนตัวอย่างเดียวกันนี้ใหม่ทั้งหมดด้วย Tokio จริง** (`#[tokio::main]`, `tokio::join!`,
> และในอนาคต Part 49 จะเปลี่ยนจาก `AsyncDelay` จำลองเป็นการเรียก HTTP/TCP จริงด้วย `reqwest`/`tokio::net`) สิ่งที่
> ควรได้จากตัวอย่างนี้คือ**แนวคิดของการออกแบบ** ไม่ใช่เครื่องมือที่ใช้ขับมัน

```rust
use futures::executor::block_on;
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};

struct AsyncDelay {
    deadline: Instant,
}

impl AsyncDelay {
    fn new(duration: Duration) -> Self {
        AsyncDelay {
            deadline: Instant::now() + duration,
        }
    }
}

impl Future for AsyncDelay {
    type Output = ();
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        if Instant::now() >= self.deadline {
            Poll::Ready(())
        } else {
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}

struct UserProfile {
    name: String,
}

struct OrderHistory {
    order_count: u32,
}

struct Recommendations {
    items: Vec<String>,
}

// จำลองการเรียก 3 ไมโครเซอร์วิสที่แยกกันอย่างสิ้นเชิง แต่ละตัวใช้เวลาตอบกลับต่างกัน
async fn fetch_user_profile(user_id: u32) -> UserProfile {
    AsyncDelay::new(Duration::from_millis(120)).await;
    UserProfile {
        name: format!("ผู้ใช้ #{user_id}"),
    }
}

async fn fetch_order_history(user_id: u32) -> OrderHistory {
    AsyncDelay::new(Duration::from_millis(200)).await;
    OrderHistory {
        order_count: user_id % 7 + 1,
    }
}

async fn fetch_recommendations(user_id: u32) -> Recommendations {
    AsyncDelay::new(Duration::from_millis(90)).await;
    Recommendations {
        items: vec![format!("สินค้าแนะนำสำหรับผู้ใช้ #{user_id}")],
    }
}

struct Dashboard {
    profile: UserProfile,
    orders: OrderHistory,
    recommendations: Recommendations,
}

impl Dashboard {
    fn print_summary(&self) {
        let items = self.recommendations.items.join(", ");
        println!(
            "  ชื่อผู้ใช้: {} | จำนวนออเดอร์: {} | คำแนะนำ: {items}",
            self.profile.name, self.orders.order_count
        );
    }
}

// เวอร์ชัน sequential: เรียกทีละงาน แต่ละงานต้องรอให้งานก่อนหน้าจบสมบูรณ์ก่อนจึงเริ่มงานถัดไป
async fn build_dashboard_sequential(user_id: u32) -> Dashboard {
    let profile = fetch_user_profile(user_id).await;
    let orders = fetch_order_history(user_id).await;
    let recommendations = fetch_recommendations(user_id).await;
    Dashboard {
        profile,
        orders,
        recommendations,
    }
}

// เวอร์ชัน concurrent: ยิงสามงานพร้อมกันด้วย join! เพราะทั้งสามงานไม่ต้องพึ่งผลลัพธ์ของกันและกันเลย
async fn build_dashboard_concurrent(user_id: u32) -> Dashboard {
    let (profile, orders, recommendations) = futures::join!(
        fetch_user_profile(user_id),
        fetch_order_history(user_id),
        fetch_recommendations(user_id),
    );
    Dashboard {
        profile,
        orders,
        recommendations,
    }
}

fn main() {
    let start_seq = Instant::now();
    let dashboard_seq = block_on(build_dashboard_sequential(42));
    let seq_elapsed = start_seq.elapsed();
    println!("[sequential]");
    dashboard_seq.print_summary();
    println!("[sequential] ใช้เวลา: {seq_elapsed:?}");

    println!();

    let start_conc = Instant::now();
    let dashboard_conc = block_on(build_dashboard_concurrent(42));
    let conc_elapsed = start_conc.elapsed();
    println!("[concurrent]");
    dashboard_conc.print_summary();
    println!("[concurrent] ใช้เวลา: {conc_elapsed:?}");

    println!();
    println!(
        "สรุป: sequential ~{}ms (120+200+90) เทียบกับ concurrent ~{}ms (max ของทั้งสามคือ 200ms)",
        seq_elapsed.as_millis(),
        conc_elapsed.as_millis()
    );
}
```

รันจริงได้:

```
[sequential]
  ชื่อผู้ใช้: ผู้ใช้ #42 | จำนวนออเดอร์: 1 | คำแนะนำ: สินค้าแนะนำสำหรับผู้ใช้ #42
[sequential] ใช้เวลา: 410.024362ms

[concurrent]
  ชื่อผู้ใช้: ผู้ใช้ #42 | จำนวนออเดอร์: 1 | คำแนะนำ: สินค้าแนะนำสำหรับผู้ใช้ #42
[concurrent] ใช้เวลา: 200.007831ms

สรุป: sequential ~410ms (120+200+90) เทียบกับ concurrent ~200ms (max ของทั้งสามคือ 200ms)
```

**อ่านตัวเลขให้ออก**: เวอร์ชัน sequential ใช้เวลา **410 ms** ตรงกับ 120+200+90 พอดี (บวกกันตรง ๆ เพราะรอทีละตัว)
ในขณะที่เวอร์ชัน concurrent ใช้เวลาแค่ **200 ms** ซึ่งเท่ากับงานที่ **ช้าที่สุด**ในกลุ่ม (คือ `fetch_order_history`
ที่รอ 200ms) — ประหยัดเวลาไปกว่าครึ่ง (200ms เทียบกับ 410ms) โดยไม่ต้องเปลี่ยนตรรกะทางธุรกิจของ `fetch_*` แต่ละ
ตัวเลย เปลี่ยนแค่**วิธีจัดลำดับการ `.await`** เท่านั้น

นี่คือเหตุผลเชิงปฏิบัติที่สำคัญที่สุดว่าทำไม async/await ถึงเป็นเครื่องมือที่มีค่ามากสำหรับระบบที่ต้องเรียกหลาย
บริการภายนอกพร้อมกัน (ซึ่งพบได้ทั่วไปมากในระบบ backend/microservices สมัยใหม่) — ยิ่งมีจำนวนบริการที่ต้องเรียก
มากเท่าไหร่ ผลต่างระหว่าง sequential กับ concurrent ก็ยิ่งมากขึ้นตามไปด้วย (ถ้ามี 10 บริการ แต่ละบริการรอ 200ms
sequential จะใช้เวลา 2000ms เทียบกับ concurrent ที่ยังคงประมาณ 200ms เท่าเดิม — ต่างกัน 10 เท่า!)

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. เอา `async fn` มาใช้ใน `main` ตรง ๆ — เจอ E0752

```rust
async fn main() {
    println!("สวัสดี");
}
```

```
error[E0752]: `main` function is not allowed to be `async`
 --> src/main.rs:1:1
  |
1 | async fn main() {
  | ^^^^^^^^^^^^^^^ `main` function is not allowed to be `async`
```

**เหตุผล**: อธิบายละเอียดแล้วในหัวข้อ 46.4 — `main` เป็นจุดเข้าโปรแกรมที่ OS เรียกโดยตรง มันไม่รู้จัก `Future`
เลย ไม่มีใครมา poll Future ที่ `main` แบบ async จะคืนออกมาให้ **วิธีแก้**: ใช้ `futures::executor::block_on`
(สำหรับตัวอย่างเรียนรู้แบบบทนี้) เขียน executor เอง (หัวข้อ 46.6) หรือใน Part 48 จะใช้ `#[tokio::main]` ซึ่งเป็น
attribute macro ที่แปลง `async fn main()` ให้กลายเป็น `fn main()` ธรรมดาที่สร้าง Tokio runtime แล้วเรียก
`block_on` ให้อัตโนมัติ (เบื้องหลังคือกลไกเดียวกับที่บทนี้อธิบายไว้ทุกขั้น เพียงแค่ห่อด้วย macro ให้สะดวกขึ้น)

### 2. เรียก `.await` นอก `async fn`/`async` Block — เจอ E0728

```rust
async fn say_hello() {
    println!("สวัสดี");
}

fn main() {
    say_hello().await; // .await ใช้ได้แค่ใน async fn/async block เท่านั้น
}
```

```
error[E0728]: `await` is only allowed inside `async` functions and blocks
 --> src/main.rs:7:17
  |
6 | fn main() {
  | --------- this is not `async`
7 |     say_hello().await; // .await ใช้ได้แค่ใน async fn/async block เท่านั้น
  |                 ^^^^^ only allowed inside `async` functions and blocks
```

**เหตุผล**: `.await` ไม่ใช่ operator ธรรมดาที่ใช้ที่ไหนก็ได้ — มันคือ syntax ที่ผูกอยู่กับกลไก state machine
(หัวข้อ 46.7) ซึ่งมีอยู่แค่ในบริบทของฟังก์ชัน/block ที่เป็น `async` เท่านั้น ฟังก์ชันธรรมดาไม่มี "จุดพัก" ให้ยึด
เกาะ **วิธีแก้**: ทำให้ `main` เป็น async แล้วขับด้วย runtime (แต่จะเจอกับดักที่ 1 ต่อ ต้องแก้ทั้งคู่พร้อมกัน:
เปลี่ยนเป็น `fn main() { futures::executor::block_on(say_hello()); }`) หรือครอบด้วย `async` block แล้ว await
มันด้วย runtime แทน

### 3. เรียก `async fn` แล้วลืม `.await` — Compile ผ่านแต่ Warning และไม่มีอะไรเกิดขึ้น

```rust
async fn compute() -> i32 {
    println!("compute() กำลังทำงานจริง");
    42
}

async fn run() {
    compute(); // ลืม .await -> ไม่มีอะไรเกิดขึ้น (ไม่พิมพ์ข้อความข้างในเลย) แค่สร้าง Future แล้วทิ้ง
    println!("run() จบแล้ว");
}

fn main() {
    futures::executor::block_on(run());
}
```

รันจริงได้ (สังเกตว่า **ไม่เห็น** "compute() กำลังทำงานจริง" เลย):

```
warning: unused implementer of `futures::Future` that must be used
 --> src/main.rs:8:5
  |
8 |     compute(); // ลืม .await -> ไม่มีอะไรเกิดขึ้น (ไม่พิมพ์ข้อความข้างในเลย) แค่สร้าง Future แล้วทิ้ง
  |     ^^^^^^^^^
  |
  = note: futures do nothing unless you `.await` or poll them
  = note: `#[warn(unused_must_use)]` (part of `#[warn(unused)]`) on by default

run() จบแล้ว
```

**เหตุผล**: ตรงกับหัวข้อ 46.2 เป๊ะ — `compute()` แค่**สร้าง**ค่า Future คืนมา ไม่ได้รันอะไรเลย และเพราะไม่มีใคร
`.await` มัน มันก็ถูกทิ้งไปเงียบ ๆ (drop) โดยไม่เคยถูก poll แม้แต่ครั้งเดียว — **สาเหตุที่มี warning** (ไม่ใช่
error) เพราะ `Future` trait ถูก mark ด้วย `#[must_use]` ใน standard library (กลไกเดียวกับที่ `Result<T, E>`
ใช้เตือนเมื่อลืมจัดการ `Err` จาก Part 12) **วิธีแก้**: ใส่ `.await` ให้ครบทุกครั้งที่ต้องการให้ Future นั้นทำงาน
จริง — นี่คือกับดักที่พบบ่อยที่สุดในหมู่มือใหม่ที่เพิ่งเริ่มเขียน async เพราะโค้ด **compile ผ่านได้ปกติ** (แค่มี
warning) ทำให้บั๊กแบบ "เงียบ ๆ" นี้หลุดผ่านการตรวจสอบตาไปได้ง่ายกว่า compile error ทั่วไปมาก

### 4. ค้าง `MutexGuard` (หรือค่าที่ไม่ `Send` อื่น ๆ) ไว้ข้าม `.await`

อธิบายละเอียดแล้วในหัวข้อ 46.9 — สรุปสั้น ๆ: ค่าใด ๆ ที่มีชีวิตอยู่ข้ามจุด `.await` จะถูกเก็บเป็น field ของ state
machine (Future) ไปด้วย ถ้าค่านั้นไม่ implement `Send` (เช่น `std::sync::MutexGuard`) จะทำให้ Future ทั้งก้อนไม่
`Send` ไปด้วย ซึ่งจะเป็นปัญหาตอนพยายาม `spawn` มันบน multi-thread executor อย่าง Tokio ใน Part 48 (error ที่เจอ
คือ `future cannot be sent between threads safely` แบบที่แสดงไว้เต็มรูปแบบในหัวข้อ 46.9) **วิธีแก้**: จำกัด scope
ของ lock ให้แคบที่สุด (ปล่อย lock ก่อนถึง `.await` เสมอ) หรือใช้ `tokio::sync::Mutex` (Part 50) ซึ่ง `MutexGuard`
ของมันออกแบบมาให้เป็น `Send` และปลอดภัยที่จะค้างข้าม `.await` ได้โดยเฉพาะ

### 5. เขียน Loop คำนวณหนัก ๆ ใน Async Task โดยไม่มี `.await` เลย — "บล็อก" Task อื่นทั้งหมด

จากหัวข้อ 46.3 เราเรียนไปแล้วว่า async task เป็น cooperative — มันจะไม่ยอมสละคิวจนกว่าจะเจอ `.await` ลองดูปัญหา
นี้ให้เห็นชัด ๆ (ใน production จริงที่ใช้ Tokio หลาย task จะถูก multiplex บน thread เดียวกันได้ ทำให้ปัญหานี้
ส่งผลกระทบชัดกว่าตัวอย่างเดี่ยว ๆ ในบทนี้มาก แต่หลักการเดียวกันสาธิตได้แม้กับ executor เดี่ยว):

```rust
use futures::executor::block_on;
use std::time::Instant;

async fn cpu_heavy_task_without_yield(n: u64) -> u64 {
    // ไม่มี .await เลยสักจุดในฟังก์ชันนี้ — เมื่อ poll ครั้งแรก มันจะรันจนจบรวดเดียว
    // ไม่สละคิวให้ใครเลย ไม่ว่าจะมี task อื่นรออยู่แค่ไหนก็ตาม
    let mut acc: u64 = 0;
    for i in 0..n {
        acc = acc.wrapping_add(i);
    }
    acc
}

fn main() {
    let start = Instant::now();
    let result = block_on(cpu_heavy_task_without_yield(200_000_000));
    println!("ผลลัพธ์ {result} ใช้เวลา {:?} — ตลอดเวลานี้ไม่มี task อื่นแทรกได้เลย", start.elapsed());
}
```

**เหตุผล**: เพราะไม่มี `.await` เลยในฟังก์ชันนี้ การ poll ครั้งแรกจะรัน loop ทั้งหมด (200 ล้านรอบ) จนจบในครั้งเดียว
— ถ้ามี task อื่นรออยู่บน executor เดียวกัน (เช่นในโปรแกรม Tokio จริงที่มีหลาย task ทำงานพร้อมกันบน worker thread
เดียว) task เหล่านั้นจะ**ไม่มีโอกาสได้ทำงานเลยแม้แต่นิดเดียว** จนกว่า loop นี้จะจบ — ต่างจาก OS thread (Part 37)
ที่ OS scheduler จะ preempt ให้ thread อื่นได้ทำงานสลับกันโดยไม่ต้องรอให้ thread ปัจจุบันสมยอม **วิธีแก้**: งาน
CPU-bound หนัก ๆ ไม่ควรรันตรง ๆ ใน async task — Part 48 จะแนะนำ `tokio::task::spawn_blocking` ซึ่งย้ายงานแบบนี้
ไปรันบน thread pool แยกที่ออกแบบมาสำหรับงานที่บล็อกโดยเฉพาะ ไม่ปนกับ task แบบ async ธรรมดา หรือถ้าเป็นไปได้ให้
หา `.await` point มาแบ่ง loop เป็นช่วง ๆ (เช่น `tokio::task::yield_now().await` แทรกเป็นระยะ)

## แบบฝึกหัด (Exercises)

1. **(ง่าย) เขียนและรัน `async fn` แรกของคุณ**: ประกาศ `async fn describe_weather(city: &str) -> String` ที่คืน
   ค่า string บอกว่า `"{city}: อากาศแจ่มใส"` เขียน `main` (แบบ `fn` ธรรมดา ไม่ใช่ `async fn`) ที่เพิ่ม crate
   `futures` (`cargo add futures`) แล้วใช้ `futures::executor::block_on` เรียก `describe_weather("เชียงใหม่")`
   แล้ว print ผลลัพธ์ที่ได้ ทดลองลบ `block_on` ออกแล้วเรียกฟังก์ชันตรง ๆ (ไม่ await ไม่ block_on) สังเกตว่าเกิด
   warning อะไร และข้อความ "แจ่มใส" หายไปจากผลลัพธ์หรือไม่
   *(hint: ทวนหัวข้อ 46.2 และ 46.4 — การเรียก `async fn` เพียว ๆ แค่สร้าง Future ยังไม่รันอะไรเลย)*

2. **(กลาง) เปรียบเทียบ Sequential กับ Concurrent ด้วยมือของคุณเอง**: เขียน `async fn download_file(name: &str,
   size_mb: u64) -> String` ที่จำลองการดาวน์โหลดไฟล์ด้วย `AsyncDelay` แบบในหัวข้อ 46.8 (ให้เวลาแปรผันตาม
   `size_mb` เช่น `size_mb * 20` มิลลิวินาที) เขียนสอง async fn: `download_all_sequential` ที่ `.await` ไฟล์
   4 ไฟล์ทีละไฟล์ (ขนาด 3, 5, 2, 4 MB) และ `download_all_concurrent` ที่ใช้ `futures::join!` ดาวน์โหลดทั้ง 4
   ไฟล์พร้อมกัน วัดเวลาด้วย `Instant`/`.elapsed()` ทั้งสองแบบ แล้ว print เปรียบเทียบ ตรวจสอบว่าเวลาของเวอร์ชัน
   concurrent ใกล้เคียงกับเวลาของไฟล์ที่ใหญ่ที่สุด (5 MB = 100ms) มากกว่าผลรวมทั้งหมด (280ms) หรือไม่
   *(hint: โครงสร้างเหมือนหัวข้อ 46.8 เป๊ะ เปลี่ยนแค่จำนวนและพารามิเตอร์)*

3. **(ยาก) เขียน Future ที่ต้อง Poll หลายครั้งกว่าจะ Ready และสังเกตพฤติกรรม State Machine**: เขียน struct
   `CountdownFuture { remaining: u32 }` ที่ implement `Future<Output = u32>` โดยที่ทุกครั้งที่ถูก poll จะลด
   `remaining` ลง 1 แล้วเรียก `wake_by_ref()` คืน `Pending` ไปเรื่อย ๆ จนกว่า `remaining == 0` จึงคืน
   `Poll::Ready(0)` เขียน `async fn` ที่ `.await` ค่า `CountdownFuture { remaining: 5 }` แล้ว print
   `"เหลืออีก {remaining} รอบ"` **ก่อน**การ poll แต่ละครั้ง (ต้องแก้ `CountdownFuture::poll` ให้ print ก่อนลดค่า)
   แล้วใช้ `tiny_block_on` จากหัวข้อ 46.6 (หรือ `futures::executor::block_on`) ขับมันจนจบ ยืนยันว่าเห็นข้อความ
   พิมพ์ 5 ครั้งก่อนได้ผลลัพธ์สุดท้าย พิสูจน์ด้วยตาตัวเองว่า Future หนึ่งตัวถูก poll ได้หลายรอบจริง ๆ
   *(hint: โครงสร้างเหมือน `YieldOnce` ในหัวข้อ 46.6 แต่เก็บตัวเลขนับถอยหลังแทน bool)*

4. **(ยากมาก / ประยุกต์ใช้งานจริง) ระบบตรวจสอบสุขภาพ Microservices พร้อม Timeout แบบง่าย**: เขียนระบบจำลองที่มี
   5 "เซอร์วิส" (`auth-service`, `payment-service`, `inventory-service`, `notification-service`,
   `analytics-service`) แต่ละตัวมี async fn `health_check(name: &str, response_ms: u64) -> Result<String,
   String>` ที่ใช้ `AsyncDelay` จำลองเวลาตอบสนอง แล้วคืน `Ok(format!("{name}: OK"))` ถ้า `response_ms <= 300`
   หรือ `Err(format!("{name}: ช้าเกินไป ({response_ms}ms)"))` ถ้าเกิน 300ms (ไม่ต้องใช้ timeout จริงจาก timer —
   ให้เช็คค่าตรง ๆ ก่อน await ก็ได้ เพื่อโฟกัสที่โครงสร้าง concurrency) กำหนดเวลาตอบสนองของแต่ละเซอร์วิสเป็น 150,
   450, 200, 600, 100 มิลลิวินาทีตามลำดับ เขียน `async fn check_all_services()` ที่ใช้ `futures::join!` เรียก
   ทั้ง 5 เซอร์วิสพร้อมกัน เก็บผลลัพธ์เป็น `Vec<Result<String, String>>` แล้ว print สรุปว่ามีกี่เซอร์วิส `Ok`
   กี่เซอร์วิส `Err` พร้อมรายชื่อ วัดเวลารวมทั้งหมดด้วย `Instant` แล้วเทียบกับเวอร์ชัน sequential (เรียกทีละตัว)
   ยืนยันว่าเวอร์ชัน concurrent เร็วกว่ามาก (ควรใกล้เคียงกับเวลาของเซอร์วิสที่ช้าที่สุดคือ 600ms ไม่ใช่ผลรวม
   1500ms)
   *(hint: `futures::join!` คืนค่าเป็น tuple ตามลำดับที่ใส่เข้าไป — เก็บผลแต่ละตัวลง `Vec` ด้วยการ destructure
   tuple นั้นออกมาแล้ว `vec![r1, r2, r3, r4, r5]` จากนั้นใช้ `.iter().filter(|r| r.is_ok()).count()` นับจำนวน
   `Ok` ตามที่เรียนมาตั้งแต่ Part 25-26)*

## สรุป

บทนี้เปิดประตูสู่**โมเดล concurrency ที่สองของ Rust** — async/await — โดยเริ่มจากคำถามที่ Part 37 ทิ้งไว้ตรง ๆ:
OS thread มีต้นทุนสูงเกินไปสำหรับงานที่ต้องรอ I/O จำนวนมาก ๆ พร้อมกัน แล้วไล่อธิบายกลไกทั้งหมดที่ทำให้ async/await
เป็นคำตอบของปัญหานั้นได้อย่างครบถ้วนและพิสูจน์ได้ด้วยโค้ดจริงทุกขั้น:

- **`async fn`** ไม่รัน body ทันทีที่ถูกเรียก — มันคืนค่าเป็น **`Future`** ซึ่งเป็นแค่ "ตัวแทนของงานที่จะให้
  ผลลัพธ์ในอนาคต" เราพิสูจน์ด้วยการรันจริงว่าไม่มีอะไรถูกพิมพ์เลยถ้าไม่มีใคร poll Future นั้น
- **`.await`** คือจุดที่ task สละคิวกลับไปให้ executor แบบ**สมัครใจ**เท่านั้น (cooperative) — ต่างจาก OS thread
  ที่ถูก preempt โดย OS ได้ทุกเมื่อ — นี่คือความต่างเชิงพื้นฐานที่สุดระหว่างสองโมเดล concurrency ของ Rust
- Rust **ไม่มี async runtime อยู่ใน `std`** — ต้องมี executor มา poll Future ให้เสมอ เราพิสูจน์ทั้ง error จริง
  E0752 (`async fn main()` ใช้ไม่ได้) และเขียน **executor ขนาดจิ๋วด้วยมือเอง** (`tiny_block_on`) เพื่อยืนยันว่า
  มันไม่ใช่มายากล ก่อนที่ Part 48 จะแนะนำ Tokio ซึ่งเป็น runtime ระดับ production
- **`async` block** คือคู่ขนานของ **closure** (Part 24) ในโลกของ Future — ทั้งสองเป็นแม่พิมพ์ที่ต้อง "กระตุ้น"
  ก่อนถึงจะทำงาน (เรียก `()` สำหรับ closure, `.await` สำหรับ Future)
- **State machine mental model**: compiler แปลง `async fn` เป็น enum/struct ที่ implement `Future` โดย
  อัตโนมัติ — enum มี 1 variant ต่อจุดพัก คู่ขนานกับที่ Part 24.9 อธิบาย closure ว่าเป็น struct ที่ compiler
  สร้างให้ — และนี่คือที่มาของคำตอบเต็มรูปแบบของคำใบ้จาก Part 23.9: async state machine เป็น **self-referential
  struct โดยธรรมชาติ** เพราะตัวแปรที่ borrow กันเองต้องอยู่ในฟิลด์เดียวกัน ซึ่งเป็นเหตุผลที่ `Future::poll`
  ต้องรับ `Pin<&mut Self>` เพื่อการันตีว่าค่าจะไม่ถูก move จนกว่าจะจบ
- **Sequential vs Concurrent**: `.await` ทีละตัวรอกันตรง ๆ (เวลาบวกกัน) ในขณะที่ `futures::join!` สลับกัน poll
  ทำให้เวลารอซ้อนทับกัน (เวลารวม = เวลาที่นานที่สุด ไม่ใช่ผลรวม) — พิสูจน์ด้วยตัวเลขจริง 410ms เทียบกับ 200ms ใน
  ตัวอย่างแดชบอร์ดลูกค้า
- **Borrow ข้าม `.await`** ส่วนใหญ่ไม่มีปัญหา (borrow checker ธรรมดาจัดการให้) แต่ค่าที่ไม่ `Send` (เช่น
  `std::sync::MutexGuard`) ที่ค้างอยู่ข้าม `.await` จะทำให้ Future ทั้งก้อนไม่ `Send` ไปด้วย ซึ่งเป็นปัญหาจริงตอน
  spawn บน multi-thread executor — เราเห็น error จริงและวิธีแก้ (จำกัด scope ของ lock)
- เปรียบเทียบกับ **JavaScript** (มี event loop ในตัวภาษาเสมอ, single-threaded โดยธรรมชาติ) และ **Python
  `asyncio`** (ต้องเลือก runtime เอง คล้าย Rust แต่มีตัวเลือกมาตรฐานให้ใน standard library) — Rust แตกต่างตรงที่
  **ไม่มี runtime ไหนอยู่ใน `std` เลยแม้แต่ตัวเดียว** สอดคล้องกับปรัชญา "ไม่บังคับต้นทุนที่ไม่จำเป็น" แบบเดียวกับ
  ที่ไม่บังคับ garbage collector

บทนี้ตั้งใจให้เห็น**แนวคิด**ทั้งหมดที่จำเป็นสำหรับการใช้งาน async/await ในระดับผู้ใช้ทั่วไป โดยยังไม่ลงรายละเอียด
กลไกภายในของ `Future` trait, `Waker`, และ `Pin` แบบเต็มรูปแบบ — **Part 47 (Futures และ Executors)** จะเจาะกลไก
เบื้องหลังทั้งหมดที่บทนี้แค่แนะนำผิวเผิน (`poll`, `Context`, `Waker` ทำงานอย่างไรจริง ๆ ในระดับ library, การเขียน
`Future`ด้วยมือสำหรับกรณีที่ซับซ้อนกว่านี้, กลไกของ `join!`/`select!` แบบเต็มรูปแบบ) ก่อนที่ **Part 48 (Tokio:
Runtime และ Tasks)** จะแนะนำ runtime ระดับ production ที่ใช้งานจริงในอุตสาหกรรม พร้อมเขียนตัวอย่างแดชบอร์ดลูกค้า
จากหัวข้อ 46.11 ขึ้นมาใหม่ด้วย `tokio::spawn`/`tokio::join!` ของจริง — ทั้งหมดนี้จะนำไปสู่ **Part 49 (Tokio I/O
และ Networking)** ที่เปลี่ยนจาก `AsyncDelay` จำลองเป็นการเชื่อมต่อเครือข่ายจริง และ **Part 50 (Async Channels
และ Synchronization)** ที่เอา channel/Mutex จาก Part 38-39 มาเขียนใหม่ในเวอร์ชันที่ปลอดภัยกับ `.await`

---

**Part ก่อนหน้า:** [Procedural Macros: Derive Macros ขั้นสูง](part-045-proc-macros-derive.md) | **Part ถัดไป:**
[Futures และ Executors](part-047-futures-executors.md)
