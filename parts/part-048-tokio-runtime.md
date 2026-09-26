# Part 48: Tokio: Runtime และ Tasks

> โมดูล: ระดับสูง (Advanced) | ระดับ: สูง | เวลาโดยประมาณ: 260 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- เพิ่ม **Tokio** เข้าโปรเจกต์ได้อย่างถูกต้องด้วย `cargo add tokio --features full` และอธิบายได้ว่าทำไม Tokio
  ถึงออกแบบตัวเองให้พึ่งพา **feature flags** อย่างหนักมาก (เชื่อมกับความเข้าใจเรื่อง `[features]` จาก Part 17/35)
  พร้อมบอกได้ว่าทำไมโค้ด production จริงมักเลือกเปิดเฉพาะ feature ที่ใช้ ไม่ใช้ `full` พร่ำเพรื่อ — และพิสูจน์ด้วย
  ตัวเลขจริงว่าการเลือก feature ให้พอดีลด dependency ที่ต้อง compile ลงได้มากแค่ไหน
- อธิบาย **`#[tokio::main]`** ได้อย่างไม่มีอะไร "เป็นมนตร์ดำ" อีกต่อไป — รู้ว่ามันคือ attribute macro (เชื่อมกับ
  Part 44 ที่สอนกลไก `#[proc_macro_attribute]`) ที่แปลง `async fn main()` เป็น `fn main()` จริงที่รันได้ พร้อม
  เขียนเวอร์ชัน **manual** ที่ไม่ใช้ attribute macule เลย (`Runtime::new().unwrap().block_on(...)`) แล้วพิสูจน์ว่า
  ทั้งสองแบบให้ผลลัพธ์เหมือนกันทุกประการ
- แยกความแตกต่างระหว่าง **runtime flavor** สองแบบหลักของ Tokio ได้ (`multi_thread` ที่เป็นค่าเริ่มต้น เทียบกับ
  `current_thread`) อธิบายได้ว่าแต่ละแบบเหมาะกับงานประเภทไหน และพิสูจน์ด้วยโค้ดจริงว่า task ที่ spawn ไปถูกกระจาย
  ไปทำงานบน OS thread ต่างกันจริงในแบบ `multi_thread` แต่รวมอยู่ที่ thread เดียวใน `current_thread`
- ใช้ **`tokio::spawn()`** สร้าง **task** ใหม่ได้ (ไม่ใช่ OS thread!) เข้าใจว่ามันคือคู่เทียบของ
  `std::thread::spawn()` จาก Part 37 ทุกกระเบียดนิ้ว — คืนค่าเป็น `JoinHandle<T>` ที่ `.await` แทน `.join()`,
  ต้องการ future ที่เป็น `Send + 'static` เหมือนกับที่ `thread::spawn` ต้องการ closure ที่เป็น `Send + 'static`
  (เชื่อมกับ Part 40) และเห็น compile error จริง **E0373** กับ "future cannot be sent between threads safely"
  เมื่อทำผิดกฎ — พร้อมพิสูจน์ด้วยตัวเลขจริงว่า task ของ Tokio **เบากว่า OS thread หลายเท่าตัว** จนสามารถ spawn
  ได้หลักหมื่นตัวพร้อมกันโดยไม่มีปัญหา
- ใช้ **`tokio::time::sleep`** แทน `thread::sleep` ได้อย่างถูกต้อง เข้าใจว่ามันไม่ block OS thread เลยแต่
  "คืน" การควบคุมให้ scheduler ไปทำ task อื่นระหว่างรอ และรัน **`tokio::join!`** เพื่อทำงานหลายอย่างพร้อมกันแทน
  การ `.await` ตามลำดับ พร้อม**วัดเวลาจริง**พิสูจน์ว่าเร็วขึ้นจริงกี่เท่า และใช้ **`tokio::select!`**
  แข่งกันระหว่าง future เพื่อทำ pattern timeout ได้
- ระบุและแก้ **กับดักที่อันตรายที่สุดของ async Rust** ได้: การเรียกโค้ดที่ block (เช่น `std::thread::sleep`
  หรือการคำนวณหนัก ๆ) ตรงในงาน async — อธิบายได้ว่าทำไมมันทำลายจุดประสงค์ทั้งหมดของ async runtime และแก้ด้วย
  **`tokio::task::spawn_blocking()`** ได้อย่างถูกต้อง พร้อมวัดเวลาจริงเห็นความต่างชัด ๆ
- อธิบายกลไก **cancellation** ของ Tokio task ได้อย่างถูกต้องแม่นยำ (ไม่ใช่ความเข้าใจผิดที่พบบ่อยว่า "drop
  `JoinHandle` แล้ว task จะถูกยกเลิกทันที") รู้ว่า `.abort()` คือทางที่ถูกต้องในการยกเลิก task จริง ๆ และเห็นว่า
  `Drop` (เชื่อมกับ Part 6) ของค่าที่ค้างอยู่ใน future ที่ถูกยกเลิกยังทำงานเสมอ ไม่ว่าจะยกเลิกด้วยวิธีไหน
- ประยุกต์ทุกอย่างเขียน **concurrent web scraper จำลอง** เต็มรูปแบบ: spawn หลาย task ทำ "fetch" พร้อมกัน (จำลอง
  I/O ด้วย `tokio::time::sleep` เพราะเนื้อหา networking จริงรอ Part 49) เก็บผลผ่าน `JoinHandle`, วัดความเร็วขึ้น
  เทียบกับแบบ sequential, และเพิ่มขั้นตอน parse ที่ใช้ CPU หนักด้วย `spawn_blocking` ให้เห็น pipeline ที่สมบูรณ์

## ความรู้ที่ต้องมีมาก่อน

- **Part 46 (Async/Await เบื้องต้น)**: บทนี้สมมติว่าคุณอ่าน `async fn`, `.await`, และเข้าใจว่า **future คือค่าที่
  "ยังไม่ได้ทำงาน" จนกว่าจะถูก poll** มาแล้วอย่างแน่น — Tokio เป็นแค่ **ตัวรัน (executor)** ที่ทำหน้าที่ poll
  future เหล่านั้นให้คุณ ไม่มีอะไรเกี่ยวกับ syntax ของ `async`/`await` ที่จะสอนใหม่ในบทนี้
- **Part 47 (Futures และ Executors)**: บทนั้นปิดท้ายด้วยการเขียน executor เองแบบง่าย ๆ (busy-polling หรือใช้
  `Waker` พื้นฐาน) แล้วชี้ให้เห็นข้อจำกัดมหาศาลของมันเทียบกับที่ต้องใช้ใน production จริง (ไม่มี I/O reactor,
  ไม่มี timer, ไม่มี work-stealing scheduler, ไม่ efficient) จบด้วยข้อสรุปว่า **"ไม่มีใครเขียน production
  executor เองจริง ๆ ในทางปฏิบัติ — ใช้ Tokio"** — บทนี้คือคำตอบของประโยคนั้น เราจะไม่เขียน executor เองอีกแล้ว
  แต่จะใช้ Tokio ซึ่งเป็น async runtime ที่ครองความนิยมสูงสุดในวงการ Rust production จริง (ใช้ในโปรเจกต์ขนาดใหญ่
  จำนวนมาก เช่น web server, database driver, message queue client) ทุกแนวคิดที่ Part 47 สอน (`Future`, `Poll`,
  `Waker`, `Pin`) ยังอยู่เบื้องหลัง Tokio ทั้งหมด เพียงแต่ Tokio implement ให้สมบูรณ์และมีประสิทธิภาพสูงแทนเรา
- **Part 37 (Threads พื้นฐาน)** และ **Part 40 (Send, Sync)**: นี่คือคู่เทียบที่สำคัญที่สุดของบทนี้ — เกือบทุก
  concept ของ Tokio ที่จะเรียนมี "คู่แฝด" อยู่ในโลกของ OS thread ที่คุณเรียนมาแล้ว: `tokio::spawn` คู่กับ
  `thread::spawn`, `JoinHandle<T>` (ของ Tokio) คู่กับ `JoinHandle<T>` (ของ `std::thread`), `.await` คู่กับ
  `.join()`, `tokio::time::sleep` คู่กับ `thread::sleep`, และ bound `Send + 'static` ที่ `tokio::spawn` ต้องการ
  ก็คือ bound **เดียวกันเป๊ะ**กับที่ `thread::spawn` ต้องการ (เหตุผลเชิงลึกจาก Part 40 ก็ยังใช้ได้ตรง ๆ) — ถ้า
  Part 37/40 ยังไม่แน่น กลับไปทวนก่อน เพราะบทนี้จะอ้างอิงกลับไปตลอดเวลาแทนที่จะอธิบายซ้ำจากศูนย์
- **Part 44 (Procedural Macros เบื้องต้น)**: `#[tokio::main]` คือ **attribute macro** (`#[proc_macro_attribute]`)
  ตัวเดียวกับที่ Part 44 สอนให้เขียนเอง (ตัวอย่าง `#[log_call]` ในหัวข้อนั้นมี signature
  `fn(TokenStream, TokenStream) -> TokenStream` เหมือนกันเป๊ะ) — บทนี้จะไม่สอนวิธีเขียน `#[tokio::main]` เอง
  (ซับซ้อนกว่าตัวอย่างใน Part 44 มาก) แต่จะอธิบายว่ามันแปลงโค้ดของคุณเป็นอะไรในระดับแนวคิด เพื่อไม่ให้มันดูเหมือน
  "มนตร์ดำที่ทำให้ `main` เป็น `async` ได้ยังไงก็ไม่รู้"
- **Part 12 (Result และ Error Handling)** และ **Part 6 (Ownership)**: `JoinHandle<T>::await` ของ Tokio คืนค่า
  เป็น `Result<T, JoinError>` แบบเดียวกับที่คุณคุ้นเคย และกลไก cancellation ในหัวข้อ 48.10 จะพึ่งพา `Drop` trait
  จาก Part 6 เต็มรูปแบบในการอธิบายว่า "cleanup" หมายถึงอะไรจริง ๆ

## เนื้อหา

### 48.1 จาก Executor ที่เขียนเอง สู่ Tokio: ทำไมต้องมี Async Runtime แบบ Production

ก่อนแตะโค้ดสักบรรทัด ต้องตั้งกรอบความเข้าใจให้ถูกก่อน เพราะคำว่า "runtime" ในบทนี้มีความหมายเฉพาะเจาะจงมาก

จาก Part 47 คุณได้เห็นว่า `Future` ใน Rust เป็นแค่ **ค่าที่อธิบายว่างานควรทำอะไร** — มันไม่ทำอะไรเองจนกว่าจะมี
ใครมา `.poll()` มัน และถ้า `poll` คืนค่า `Poll::Pending` ต้องมีกลไกที่ถูก "ปลุก" กลับมาทีหลังผ่าน `Waker` เมื่อ
งานที่รอนั้นพร้อมแล้ว — Rust เอง (ผ่าน `std`) **ไม่ได้ให้ executor มาด้วย** เป็นการตัดสินใจทางออกแบบที่ตั้งใจ:
ภาษาให้แค่ "ไวยากรณ์" (`async`/`await`) และ "trait" (`Future`) แต่ปล่อยให้ **ecosystem** เลือก runtime ที่เหมาะกับ
งานของตัวเองได้ (ต่างจาก Go ที่ goroutine scheduler ผูกติดกับตัวภาษาโดยตรง ไม่มีให้เลือก)

Part 47 ปิดท้ายด้วยการเขียน executor เองแบบง่าย ๆ เพื่อ**เข้าใจกลไก** แต่ก็ชี้ให้เห็นข้อจำกัดตรงไปตรงมา:
executor ของเราเอง**ไม่มี I/O reactor จริง** (ไม่รู้ว่า socket ไหนพร้อมอ่าน/เขียน), **ไม่มี timer wheel ที่
efficient** (การจำลอง `sleep` ทำได้แค่ busy-loop หรือ thread แยกที่หยาบมาก), และ**ไม่มี work-stealing scheduler**
ที่กระจายงานไปหลาย CPU core อย่างมีประสิทธิภาพ — การเขียนสิ่งเหล่านี้ให้ถูกต้อง 100% และเร็วในระดับ production
เป็นงานวิศวกรรมที่ใช้เวลาเป็นปี ผ่านการทดสอบ/ปรับแต่งจากผู้ใช้งานจริงหลายพันโปรเจกต์ — **ไม่มีทีมพัฒนาโปรแกรมทั่วไป
ทีมไหนเขียน production executor ของตัวเองจริง ๆ** เหมือนกับที่ไม่มีใครเขียน memory allocator ของตัวเองสำหรับ
โปรแกรมทั่วไป (ใช้ของ `std` ที่มีคนทดสอบมาอย่างยาวนานแทน)

**Tokio** คือ crate ที่แก้ปัญหานี้ให้เสร็จสมบูรณ์ — เป็น async runtime ที่ได้รับความนิยมสูงสุดในวงการ Rust
(ไม่ใช่ runtime เดียวที่มีอยู่ในระบบนิเวศ — ยังมี `async-std` และ `smol` เป็นตัวเลือกอื่นที่ออกแบบมาด้วย
เป้าหมายและ API ที่ต่างกันเล็กน้อย แต่ Tokio ครองส่วนแบ่งการใช้งานส่วนใหญ่ในโปรเจกต์ production จริงอย่างชัดเจน
รวมถึงเป็นพื้นฐานของเฟรมเวิร์ก web ยอดนิยมอย่าง `axum`/`hyper` และ database driver จำนวนมาก บทนี้และ Part 49-50
จึงโฟกัสที่ Tokio เพียงตัวเดียวเพื่อความลึกและเพราะมันคือสิ่งที่คุณมีโอกาสเจอในงานจริงมากที่สุด) ประกอบด้วยสาม
ส่วนหลักที่ทำงานร่วมกัน:

1. **Scheduler**: ตัวจัดการว่า task (future ที่ spawn ไว้) ตัวไหนควรถูก poll เมื่อไหร่ บน thread ไหน — รองรับทั้ง
   โมเดล multi-thread แบบ work-stealing และ single-thread ที่เบากว่า (หัวข้อ 48.4)
2. **I/O Reactor**: ตัวเชื่อมกับ OS-level I/O event notification (เช่น `epoll` บน Linux, `kqueue` บน macOS,
   IOCP บน Windows) ทำให้ task ที่รอ socket/file พร้อมสามารถ "หลับ" อย่างมีประสิทธิภาพโดยไม่กิน CPU แล้วถูกปลุกขึ้น
   มาแม่นยำตอนที่พร้อมจริง ๆ (เนื้อหาละเอียดของส่วนนี้คือ Part 49)
3. **Timer**: ระบบ timer ที่ efficient สำหรับ `tokio::time::sleep`, `tokio::time::timeout` และเพื่อนของมัน
   (หัวข้อ 48.6)

สิ่งสำคัญที่ต้องเข้าใจให้ชัดตั้งแต่ต้นบท: **Tokio ไม่ได้เปลี่ยนกฎของภาษา Rust เลยแม้แต่นิดเดียว** — `Future`,
`Poll`, `Waker`, `Pin` ที่เรียนมาจาก Part 46-47 ยังเหมือนเดิมทุกอย่าง สิ่งที่ Tokio ทำคือให้ **implementation
ของ executor ที่สมบูรณ์และเร็ว** มาแทนที่ executor แบบง่าย ๆ ที่เราเขียนเองใน Part 47 — เหมือนกับที่ `Vec<T>`
จาก Part 8 ไม่ได้เปลี่ยนกฎ ownership ของภาษา เพียงแค่ implement โครงสร้างข้อมูลที่ efficient ให้ใช้แทนการจัดการ
raw memory เอง

### 48.2 เพิ่ม Tokio เข้าโปรเจกต์: `cargo add tokio --features full`

เริ่มจากคำสั่งเดียวที่ใช้บ่อยที่สุดตอนหัดเขียน Tokio:

```bash
cargo add tokio --features full
```

รันคำสั่งนี้จริงในโปรเจกต์เปล่า ได้ `Cargo.toml` ที่มี dependency บรรทัดนี้เพิ่มเข้ามา:

```toml
[dependencies]
tokio = { version = "1.53.1", features = ["full"] }
```

(เลขเวอร์ชันจะเปลี่ยนไปตามเวลาที่คุณรันจริง — Tokio อยู่ใน major version 1 มานานมากแล้วและรักษา backward
compatibility อย่างเคร่งครัดตาม semantic versioning ที่ Part 17 สอนไว้ ดังนั้นโค้ดในบทนี้ใช้ได้กับทุก Tokio 1.x)

#### ทำไม Tokio ต้องพึ่งพา Feature Flags หนักมากเป็นพิเศษ

จำได้จาก **Part 17 หัวข้อ 17.12** และ **Part 35** ว่า `[features]` ใน `Cargo.toml` คือกลไกให้ crate หนึ่งเปิด/ปิด
ส่วนของ code ที่ compile ได้ ควบคุมด้วย `#[cfg(feature = "...")]` — Tokio ใช้กลไกนี้**หนักกว่า crate ทั่วไปมาก**
เพราะ Tokio ไม่ใช่แค่ "ไลบรารีเดียว" แต่เป็น**ชุดของ subsystem หลายตัวที่ไม่เกี่ยวข้องกันโดยตรง** มัดรวมอยู่ใน
crate เดียว: runtime/scheduler, TCP/UDP networking, filesystem I/O แบบ async, timer, channel แบบต่าง ๆ
(`tokio::sync`), process spawning, signal handling ฯลฯ — โปรแกรมที่ต้องการแค่ "รัน task พร้อมกันกับ timer"
(เช่นตัวอย่างส่วนใหญ่ในบทนี้) ไม่มีความจำเป็นต้อง compile โค้ดสำหรับ TCP socket หรือ Unix signal handling เข้าไป
ด้วยเลย

ลองดูรายชื่อ feature ทั้งหมดที่ Tokio ประกาศไว้ (เรียกดูจริงจาก `cargo add tokio` โดยไม่ระบุ feature ใด ๆ เพื่อให้
เห็นรายการเต็ม):

```bash
cargo add tokio
```

```
      Adding tokio v1.53.1 to dependencies.
             Features:
             + bytes
             - fs
             - full
             - io-std
             - io-util
             - libc
             - macros
             - mio
             - net
             - parking_lot
             - process
             - rt
             - rt-multi-thread
             - signal
             - signal-hook-registry
             - signal-hook-sys
             - sync
             - test-util
             - time
             - tracing
             - windows-sys
```

(เครื่องหมาย `+` คือ feature ที่เปิดโดย default, `-` คือ feature ที่ต้องเปิดเองถ้าต้องการ) เห็นได้ชัดว่านี่คือ
รายชื่อ feature ที่ยาวกว่า crate ทั่วไปมาก — feature อย่าง `fs` (การอ่าน/เขียนไฟล์แบบ async), `net` (TCP/UDP),
`process` (spawn โปรเซสลูกแบบ async), `signal` (จับ signal ของ OS เช่น Ctrl+C) ต่างเป็นชุดโค้ดที่แยกกันโดย
สิ้นเชิงในทางปฏิบัติ — โปรแกรมส่วนใหญ่ใช้แค่ 3-4 feature จากทั้งหมดนี้

**`full`** คือ feature พิเศษที่เปิด**ทุกอย่างพร้อมกัน** (เขียนไว้ใน `Cargo.toml` ของ Tokio เองว่า `full` เป็นแค่
alias ที่รวม feature อื่นทั้งหมดเข้าด้วยกัน) — สะดวกมากสำหรับตอน**เรียนรู้**เพราะไม่ต้องมานั่งจำว่าตัวอย่างที่กำลัง
ลองต้องการ feature ไหนบ้าง ทุกอย่างพร้อมใช้ตลอดเวลา นี่คือเหตุผลที่บทนี้ (และ Part 49-50) จะใช้ `full` เป็นค่า
เริ่มต้นในตัวอย่างส่วนใหญ่เพื่อไม่ให้ต้องหยุดอธิบาย feature flag ทุกตัวอย่าง

**แต่ในโค้ด production จริง การเปิด `full` แบบพร่ำเพรื่อมีต้นทุนจริงที่วัดได้สองอย่าง**:

1. **เวลา compile นานขึ้น**: feature ที่ไม่ได้ใช้ก็ยังถูก compile เป็น object code (แม้จะไม่ถูกเรียกใช้จริง)
   เพิ่ม dependency ทรานซิทีฟ (transitive dependency) ที่ต้องดึงมาด้วย (เช่น `signal-hook-registry` สำหรับ
   `signal`, `libc`/`socket2` สำหรับ `net`) ยิ่งเปิด feature มาก ยิ่งมี crate ให้ compile มากตามไปด้วย
2. **ขนาด binary ใหญ่ขึ้น**: แม้ dead code elimination ของ compiler (linker-level) จะช่วยตัดโค้ดที่ไม่ได้ถูก
   เรียกจริง ๆ ออกไปได้ในระดับหนึ่ง แต่ก็ไม่สมบูรณ์แบบ 100% เสมอ (โดยเฉพาะโค้ดที่มี trait object หรือ generic
   monomorphization ซับซ้อน) การเปิด feature เกินจำเป็นจึงมีโอกาสทำให้ binary สุดท้ายใหญ่ขึ้นโดยไม่จำเป็น

มาพิสูจน์ข้อแรกด้วยตัวเลขจริง — เปรียบเทียบจำนวน package ที่ต้อง lock/compile ระหว่างเปิด `full` กับเปิดเฉพาะ
feature ที่จำเป็นจริง ๆ สำหรับโปรแกรมที่แค่ต้องการ `#[tokio::main]` และ `tokio::time::sleep`:

```bash
# แบบเปิดทุกอย่าง
cargo add tokio --features full
```

```
    Updating crates.io index
     Locking 24 packages to latest Rust 1.94.1 compatible versions
```

```bash
# แบบเปิดเฉพาะที่ใช้จริง: runtime แบบ multi-thread + attribute macro + timer
cargo add tokio --features rt-multi-thread,macros,time
```

```
    Updating crates.io index
     Locking 7 packages to latest compatible versions
```

ผลลัพธ์จริง: **24 packages เทียบกับ 7 packages** — ต่างกันมากกว่า 3 เท่าสำหรับโปรแกรมที่ทำหน้าที่เดียวกันเป๊ะ
(นี่ยังไม่นับเวลาที่ crate อย่าง `mio` — reactor สำหรับ networking — หรือ `signal-hook-registry` ถูกดึงมาด้วย
ทั้งที่โปรแกรมไม่แตะ network หรือ signal เลยในกรณี `full`) โค้ดที่รันด้วย feature แบบย่อนี้:

```rust
use std::time::Duration;

#[tokio::main]
async fn main() {
    tokio::time::sleep(Duration::from_millis(10)).await;
    println!("ทำงานด้วย feature flag เฉพาะที่จำเป็น: rt-multi-thread + macros + time");
}
```

รันได้ผลลัพธ์ปกติทุกประการ:

```
ทำงานด้วย feature flag เฉพาะที่จำเป็น: rt-multi-thread + macros + time
```

**สรุปแนวปฏิบัติที่แนะนำ**: ตอนเรียนหรือ prototype ใช้ `--features full` ได้เต็มที่เพื่อความสะดวก แต่พอโปรเจกต์
เข้าใกล้ production ควรกลับมาดูว่าจริง ๆ ใช้ feature ไหนบ้าง แล้วเปิดเฉพาะที่จำเป็น — feature ที่ใช้บ่อยที่สุดคือ

| Feature | ใช้ทำอะไร | ต้องการเมื่อ |
|---|---|---|
| `rt` | runtime พื้นฐาน (current-thread) | ต้องมี async runtime อะไรก็ตาม |
| `rt-multi-thread` | runtime แบบ multi-thread (work-stealing) | ใช้ `#[tokio::main]` แบบเริ่มต้น หรือ `Runtime::new()` |
| `macros` | attribute macro `#[tokio::main]` และ `#[tokio::test]` | เกือบทุกโปรแกรม |
| `time` | `tokio::time::sleep`, `timeout`, `interval` | มี delay หรือ timeout ในระบบ |
| `net` | `TcpStream`, `TcpListener`, `UdpSocket` | ทำ networking (Part 49) |
| `io-util`/`io-std` | `AsyncReadExt`/`AsyncWriteExt`, stdin/stdout async | อ่าน/เขียนข้อมูลแบบ async |
| `sync` | `tokio::sync::{Mutex, RwLock, mpsc, oneshot, ...}` | ต้องการ synchronization ข้าม task (Part 50) |
| `fs` | อ่าน/เขียนไฟล์แบบ async | I/O กับ filesystem |
| `process` | spawn โปรเซสลูกแบบ async | รันโปรแกรมภายนอก |
| `signal` | จับ signal ของ OS (`SIGINT`/Ctrl+C ฯลฯ) | graceful shutdown |

นี่คือหลักการเดียวกันกับที่ Part 17/35 สอนเรื่อง optional dependency และ feature ที่ออกแบบมาให้ "จ่ายเฉพาะที่ใช้"
(pay only for what you use) — Tokio เพียงแค่นำหลักการนี้มาใช้ในสเกลที่ใหญ่กว่า crate ทั่วไปมาก เพราะตัวมันเอง
ครอบคลุมงานหลากหลายประเภทไว้ในที่เดียว

### 48.3 `#[tokio::main]`: Attribute Macro ที่ทำให้ `async fn main()` รันได้จริง

ลองเขียนโปรแกรม Tokio ที่สั้นที่สุด:

```rust
#[tokio::main]
async fn main() {
    println!("สวัสดีจาก #[tokio::main] — runtime flavor เริ่มต้นคือ multi_thread");
    let result = async { 1 + 1 }.await;
    println!("ผลลัพธ์จาก async block ที่ await แล้ว: {result}");
}
```

รันจริงได้:

```
สวัสดีจาก #[tokio::main] — runtime flavor เริ่มต้นคือ multi_thread
ผลลัพธ์จาก async block ที่ await แล้ว: 2
```

จุดที่ต้องเข้าใจให้ทะลุ: **`fn main()` ของ Rust ไม่มีทางเป็น `async fn` ได้จริง ๆ** — `main` คือจุดเริ่มต้นของ
โปรแกรมที่ runtime ของภาษา (ในกรณีนี้คือ `std`) เรียกตรง ๆ โดยไม่มี executor ใดมา `.poll()` มันให้ ถ้าคุณเขียน
`async fn main()` เปล่า ๆ โดยไม่มี attribute อะไรกำกับ compiler จะปฏิเสธทันที:

```rust
async fn main() {
    println!("นี่จะไม่ compile");
}
```

```
error[E0752]: `main` function is not allowed to be `async`
 --> src/main.rs:1:1
  |
1 | async fn main() {
  | ^^^^^^^^^^^^^^^^ `main` function is not allowed to be `async`
```

**`#[tokio::main]` คือ attribute macro** (`#[proc_macro_attribute]`) ตัวเดียวกับที่ Part 44 สอนกลไกให้เขียนเอง
มันมี signature ระดับแนวคิดเหมือน `#[log_call]` ในหัวข้อ 44.x เป๊ะ: รับสอง `TokenStream` (argument ของ attribute
เอง เช่น `flavor = "..."` ที่จะเห็นในหัวข้อ 48.4 และตัว item ที่ attribute ติดอยู่ คือฟังก์ชัน `async fn main`
ทั้งก้อน) แล้ว**คืน `TokenStream` ใหม่ที่แทนที่ item เดิม** — สิ่งที่มันทำคือ **สร้าง `fn main()` (แบบธรรมดา ไม่ใช่
`async`) ตัวใหม่ขึ้นมา** ที่ข้างในสร้าง Tokio runtime แล้วเอา `async fn main` เดิมของคุณไปรันบน runtime นั้น
ผ่าน `block_on`

พูดให้เป็นรูปธรรม — โค้ดที่เขียนด้วย `#[tokio::main]`:

```rust
#[tokio::main]
async fn main() {
    println!("สวัสดีจาก async main");
}
```

ขยายออกมาในระดับแนวคิด (ไม่ใช่ token stream จริงที่ macro สร้าง ซึ่งซับซ้อนกว่านี้เล็กน้อยในรายละเอียด เช่น
การตั้งชื่อฟังก์ชันภายในไม่ให้ชนกัน แต่ตรรกะหลักคือแบบนี้เป๊ะ) เป็นสิ่งที่เทียบเท่ากับการเขียนมือเองแบบนี้:

```rust
fn main() {
    tokio::runtime::Runtime::new()
        .unwrap()
        .block_on(async {
            println!("สวัสดีจาก async main");
        })
}
```

มาพิสูจน์ว่าทั้งสองแบบให้ผลลัพธ์**เหมือนกันทุกประการ**จริง ๆ — รันเวอร์ชัน manual แบบเต็ม:

```rust
fn main() {
    let rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async {
        println!("สวัสดีจาก manual Runtime::new().block_on()");
        let result = async { 1 + 1 }.await;
        println!("ผลลัพธ์จาก async block ที่ await แล้ว: {result}");
    });
}
```

```
สวัสดีจาก manual Runtime::new().block_on()
ผลลัพธ์จาก async block ที่ await แล้ว: 2
```

เทียบกับเวอร์ชัน `#[tokio::main]` ในตอนต้นหัวข้อที่ให้:

```
สวัสดีจาก #[tokio::main] — runtime flavor เริ่มต้นคือ multi_thread
ผลลัพธ์จาก async block ที่ await แล้ว: 2
```

โครงสร้างผลลัพธ์เหมือนกันทุกจุด (ต่างกันแค่ข้อความที่เราตั้งใจเขียนต่างกันเพื่อบอกว่ามาจากเวอร์ชันไหน) — พิสูจน์
ว่า `#[tokio::main]` **ไม่ได้ทำอะไรที่วิเศษไปกว่า** การสร้าง `Runtime` แล้วเรียก `.block_on()` ให้เราแบบอัตโนมัติ

**อ่านทีละส่วนของ `Runtime::new().unwrap().block_on(...)`**:

- **`Runtime::new()`** — สร้าง Tokio runtime ตัวใหม่ (ตั้ง scheduler, I/O reactor, timer ให้พร้อมทำงาน) คืนค่า
  เป็น `io::Result<Runtime>` เพราะการสร้าง runtime**อาจล้มเหลวได้จริง**ในระดับ OS (เช่น ระบบสร้าง OS thread ใหม่
  ไม่ได้เพราะ resource เต็ม — เหตุผลเดียวกับที่ `thread::Builder::spawn()` ใน Part 37 คืน `io::Result` เช่นกัน)
  `.unwrap()` จึงจำเป็นถ้าไม่ได้จัดการ error กรณีนี้เอง
- **`.block_on(future)`** — รับ future หนึ่งตัว (ในที่นี้คือ async block ทั้งก้อนของ `main` เดิม) แล้ว **บล็อก
  thread ปัจจุบันไว้** จนกว่า future นั้นจะทำงานจนเสร็จสมบูรณ์ — สังเกตคำว่า "บล็อก" ตรงนี้: `block_on` คือจุด
  เดียวที่ async กับ synchronous โลกมาเชื่อมกัน มันคือสิ่งที่ทำให้ `fn main()` แบบธรรมดา (ที่ synchronous เต็มตัว
  ตามธรรมชาติ) สามารถ "รอ" ผลลัพธ์ของ async code ได้โดยไม่ return ก่อน — เปรียบเทียบได้ตรง ๆ กับ `.join()` ของ
  `std::thread::JoinHandle` ใน Part 37 ที่บล็อก thread ปัจจุบันจนกว่า thread ลูกจะจบ เพียงแต่ `block_on` บล็อก
  รอ**future**ให้จบ ไม่ใช่รอ**thread**

**ทำไมต้องใช้ attribute macro แทนให้ผู้เขียนโค้ด `block_on` เองตลอด**: เพราะ pattern
`Runtime::new().unwrap().block_on(...)` เป็นสิ่งที่ต้องเขียนซ้ำ**ทุกโปรแกรม**ที่ใช้ Tokio (แทบไม่มีข้อยกเว้น) —
`#[tokio::main]` จึงมีไว้เพื่อลด boilerplate นี้ทั้งหมด แล้วให้คุณเขียน `async fn main()` ได้ตรงไปตรงมาที่สุด
เหมือนกับที่ `#[derive(Debug)]` ใน Part 45 ลด boilerplate ของการเขียน `impl Debug` มือเองซ้ำ ๆ ทุก struct — ทั้ง
สองกรณีคือ macro ที่**สร้างโค้ดที่คุณเขียนเองได้อยู่แล้ว** ให้อัตโนมัติ ไม่ใช่มนตร์ดำที่ทำสิ่งที่เป็นไปไม่ได้

### 48.4 Runtime Flavor: `multi_thread` (ค่าเริ่มต้น) เทียบกับ `current_thread`

`#[tokio::main]` มี argument ที่ปรับแต่งได้ ตัวที่สำคัญที่สุดคือ **`flavor`** ซึ่งเลือกว่า scheduler ของ runtime
จะทำงานแบบไหน

#### `multi_thread`: ค่าเริ่มต้น — Work-Stealing Thread Pool

ถ้าไม่ระบุ `flavor` เลย (แบบทุกตัวอย่างที่ผ่านมาในบทนี้) Tokio จะใช้ **`multi_thread`** เป็นค่าเริ่มต้น — สร้าง
**thread pool ขนาดเท่าจำนวน logical CPU core ที่เครื่องมี** (ใช้กลไกเดียวกับ
`thread::available_parallelism()` ที่ Part 37 หัวข้อ 37.2 สอน) แล้วกระจาย task ที่ spawn ไปให้ worker thread
เหล่านั้นทำงานแบบ **work-stealing**: ถ้า worker thread ตัวหนึ่งทำงานของตัวเองหมดแล้วแต่ thread อื่นยังมีงานค้าง
มันจะ "ขโมย" งานจาก thread อื่นมาทำต่อ ทำให้ทุก CPU core ถูกใช้งานอย่างเต็มที่และสมดุลอยู่เสมอ

ลองพิสูจน์ว่า task ที่ spawn ไปจริง ๆ ถูกกระจายไปทำงานบน **OS thread ที่ต่างกัน**:

```rust
use std::collections::HashSet;
use std::sync::{Arc, Mutex};

#[tokio::main] // ค่าเริ่มต้น = multi_thread (work-stealing thread pool)
async fn main() {
    let seen = Arc::new(Mutex::new(HashSet::new()));
    let mut handles = Vec::new();
    for i in 0..50 {
        let seen = Arc::clone(&seen);
        handles.push(tokio::spawn(async move {
            let tid = std::thread::current().id();
            seen.lock().unwrap().insert(format!("{tid:?}"));
            tokio::time::sleep(std::time::Duration::from_millis(5)).await;
            i
        }));
    }
    for h in handles {
        let _ = h.await.unwrap();
    }
    let seen = seen.lock().unwrap();
    println!(
        "multi_thread flavor: task 50 ตัวถูกกระจายไปทำงานบน OS thread ที่ต่างกันจริง {} thread",
        seen.len()
    );
}
```

(โค้ดนี้ใช้ `tokio::spawn`, `JoinHandle`, `Arc<Mutex<T>>` ซึ่งจะอธิบายละเอียดในหัวข้อถัดไป — ตอนนี้แค่โฟกัสที่
ผลลัพธ์ของจำนวน thread) รันจริงบนเครื่องที่ใช้เขียนบทนี้ (4 logical CPU core) ได้:

```
multi_thread flavor: task 50 ตัวถูกกระจายไปทำงานบน OS thread ที่ต่างกันจริง 4 thread
```

task 50 ตัวถูกกระจายไปทำงานบน **4 OS thread จริง** (เท่ากับจำนวน CPU core) — นี่คือหลักฐานที่จับต้องได้ว่า
`multi_thread` flavor ใช้ **thread pool ขนาดคงที่จำนวนน้อย** รัน task น้ำหนักเบาจำนวนมาก แทนที่จะสร้าง OS thread
ใหม่ 1:1 ต่อ 1 task แบบที่ `std::thread::spawn` ทำใน Part 37 — จำนวน thread จริงจะต่างกันไปตามเครื่องที่คุณรัน
(ขึ้นกับจำนวน CPU core) แต่หลักการเดิม: **มัน "ไม่มีวัน" เป็น 50 thread จริง**

#### `current_thread`: Single-Threaded Runtime — เบากว่า เหมาะกับงาน I/O-Bound ล้วน

เลือกได้ด้วยการระบุ `flavor` ใน attribute:

```rust
use std::collections::HashSet;
use std::sync::{Arc, Mutex};

#[tokio::main(flavor = "current_thread")]
async fn main() {
    let seen = Arc::new(Mutex::new(HashSet::new()));
    let mut handles = Vec::new();
    for i in 0..50 {
        let seen = Arc::clone(&seen);
        handles.push(tokio::spawn(async move {
            let tid = std::thread::current().id();
            seen.lock().unwrap().insert(format!("{tid:?}"));
            tokio::time::sleep(std::time::Duration::from_millis(5)).await;
            i
        }));
    }
    for h in handles {
        let _ = h.await.unwrap();
    }
    let seen = seen.lock().unwrap();
    println!(
        "current_thread flavor: task 50 ตัวทำงานบน OS thread เดียวกันทั้งหมด รวม {} thread",
        seen.len()
    );
}
```

```
current_thread flavor: task 50 ตัวทำงานบน OS thread เดียวกันทั้งหมด รวม 1 thread
```

ครั้งนี้ทุก task ทำงานบน **OS thread เดียว** — ไม่มี thread pool เลย มีแค่ thread หลักที่รัน `main` เป็นคนเดียว
ที่ poll ทุก task สลับกันไปเรื่อย ๆ ตามจังหวะที่แต่ละ task ให้การควบคุมกลับคืนมา (เช่นตอน `.await` แล้วยังไม่พร้อม)

**เมื่อไหร่ควรเลือกอะไร** สรุปเป็นตาราง:

| ประเด็น | `multi_thread` (ค่าเริ่มต้น) | `current_thread` |
|---|---|---|
| จำนวน OS thread | เท่าจำนวน logical CPU core (work-stealing) | 1 thread เดียว |
| Overhead ตอนสร้าง runtime | สูงกว่าเล็กน้อย (ต้องสร้าง thread pool) | ต่ำที่สุด — สร้าง thread เดียว |
| ใช้ CPU หลาย core พร้อมกันได้ไหม | ได้ — task คำนวณหนักหลายตัวทำงานคู่ขนานจริงบน core ต่างกัน | ไม่ได้ — ทุก task แข่งกันใช้ core เดียว |
| เหมาะกับงานแบบไหน | งานที่มีทั้ง I/O-bound และอาจมี CPU-bound ปนอยู่บ้าง, web server ที่รับ request จำนวนมาก | งาน I/O-bound ล้วน ๆ ที่ไม่ต้องการ parallelism ระดับ CPU เช่น CLI tool เล็ก ๆ, script อัตโนมัติ, embedded/edge ที่ resource จำกัด |
| Feature ที่ต้องเปิด | `rt-multi-thread` | `rt` (เบากว่า ไม่ต้องมี `rt-multi-thread`) |
| ความซับซ้อนเชิง concurrency | สูงกว่า — ต้องระวังเรื่อง `Send` ข้าม thread เต็มรูปแบบ (หัวข้อ 48.6) | ต่ำกว่า — task ไม่มีวัน "ข้าม thread" เพราะมีแค่ thread เดียว |

**ข้อสังเกตสำคัญที่มักเข้าใจผิด**: `current_thread` ไม่ได้แปลว่า "รันได้แค่ 1 task พร้อมกัน" — มันยังรัน task
จำนวนมาก**พร้อมกันในทางตรรกะ**ได้ตามปกติ (interleaved บน thread เดียว) เพียงแต่**ไม่มีการทำงานคู่ขนานจริงระดับ
CPU** เท่านั้น — สำหรับงานที่ส่วนใหญ่คือการ "รอ" (I/O เช่นรอ network, รอ timer) ไม่ใช่ "คำนวณ" การมีแค่ thread
เดียวไม่เสียเปรียบเลยในทางปฏิบัติ เพราะ CPU core อื่นก็ไม่มีอะไรให้ทำอยู่ดีถ้าทุก task แค่รออะไรสักอย่าง — นี่คือ
เหตุผลที่โปรแกรมขนาดเล็กจำนวนมาก (CLI tool ที่ยิง HTTP request สองสามครั้งแล้วจบ) มักเลือก `current_thread`
เพื่อลด overhead ตอนสตาร์ทโปรแกรมและลด binary size (ไม่ต้องดึง `rt-multi-thread` มา)

### 48.5 `tokio::spawn()`: สร้าง Task ใหม่ — คู่เทียบของ `thread::spawn()`

ตอนนี้มาถึงหัวใจหลักของบทนี้ ฟังก์ชันที่คุณจะใช้บ่อยที่สุดเมื่อเขียนโค้ด Tokio

จำ signature ของ `std::thread::spawn` จาก **Part 37 หัวข้อ 37.2** ได้ไหม:

```rust
pub fn spawn<F, T>(f: F) -> JoinHandle<T>
where
    F: FnOnce() -> T,
    F: Send + 'static,
    T: Send + 'static,
{
    // ...
}
```

`tokio::spawn` มี signature ที่**คล้ายกันมาก** เพียงแค่รับ **future** แทน closure (สมเหตุสมผล เพราะ Tokio ทำงาน
กับ async code ไม่ใช่ synchronous code):

```rust
use std::future::Future;
use tokio::task::JoinHandle;

pub fn spawn<F>(future: F) -> JoinHandle<F::Output>
where
    F: Future + Send + 'static,
    F::Output: Send + 'static,
{
    // การ implement จริงอยู่ใน tokio — ที่นี่แค่แสดง signature เฉย ๆ
    unimplemented!()
}
```

เทียบสองอันนี้ทีละส่วน:

| `std::thread::spawn` (Part 37) | `tokio::spawn` |
|---|---|
| รับ `F: FnOnce() -> T` (closure) | รับ `F: Future` (async block/async fn call) |
| bound `F: Send + 'static` | bound `F: Send + 'static` **เหมือนกันเป๊ะ** |
| bound ค่า return `T: Send + 'static` | bound ค่า return `F::Output: Send + 'static` **เหมือนกันเป๊ะ** |
| คืน `std::thread::JoinHandle<T>` | คืน `tokio::task::JoinHandle<T>` |
| ทำงานบน **OS thread ใหม่หนึ่งตัว** | ทำงานบน **task ใหม่หนึ่งตัว** (ไม่ใช่ OS thread ใหม่!) |

ความต่างที่สำคัญที่สุด — **สิ่งที่ `tokio::spawn` สร้างขึ้นมาไม่ใช่ OS thread เลย** มันคือ **task**: หน่วยงาน
ระดับ "ผู้ใช้" (userspace) ที่ Tokio scheduler จัดการเอง เก็บไว้ใน heap (ผ่าน `Box`-like allocation ภายใน) แล้ว
poll สลับกับ task อื่น ๆ บน worker thread ที่มีอยู่แล้ว (จาก thread pool ที่สร้างตอน runtime เริ่มทำงาน ตามที่
เห็นในหัวข้อ 48.4) — **ไม่มีการขอ stack memory ใหม่จาก OS, ไม่มีการลงทะเบียน thread ใหม่กับ OS scheduler** แบบที่
`thread::spawn` ต้องทำทุกครั้ง

มาดูตัวอย่างพื้นฐานที่สุดก่อน:

```rust
#[tokio::main]
async fn main() {
    let handle: tokio::task::JoinHandle<i32> = tokio::spawn(async {
        println!("task ลูก: กำลังทำงาน");
        tokio::time::sleep(std::time::Duration::from_millis(20)).await;
        println!("task ลูก: ทำงานเสร็จ");
        42
    });

    println!("task หลัก: spawn แล้ว กำลังรอด้วย .await");
    let result: Result<i32, tokio::task::JoinError> = handle.await;
    println!("task หลัก: ได้ผลลัพธ์กลับมา = {result:?}");
}
```

รันจริงได้:

```
task หลัก: spawn แล้ว กำลังรอด้วย .await
task ลูก: กำลังทำงาน
task ลูก: ทำงานเสร็จ
task หลัก: ได้ผลลัพธ์กลับมา = Ok(42)
```

สังเกต pattern ที่เหมือนกับ Part 37 เป๊ะ: `tokio::spawn(...)` คืนค่ากลับมาทันที (ไม่รอ task ทำงานจบ) ส่วน
`handle.await` (แทนที่ `.join()` ของ Part 37) คือจุดที่**รอ**ผลลัพธ์จริง — แต่ต่างจาก `.join()` ที่**บล็อก**
thread ทั้งเส้น `.await` แค่ "คืนการควบคุม" ให้ executor ไปทำงานอื่นระหว่างที่ task ลูกยังไม่เสร็จ (thread ที่รัน
`main` ไม่ได้หยุดนิ่งเฉย ๆ แบบ `.join()` — มันสามารถไปช่วย poll task อื่นได้ถ้ามี)

#### `JoinError`: คู่เทียบของ `thread::Result<T>` จาก Part 37

`handle.await` คืนค่าเป็น `Result<T, JoinError>` — เทียบตรงกับ `thread::Result<T>` ที่ Part 37 หัวข้อ 37.4 สอน
(`Result<T, Box<dyn Any + Send + 'static>>`) แต่ Tokio ให้ error type ที่เจาะจงกว่าคือ `JoinError` ซึ่งครอบคลุม
สองกรณีคือ **panic** และ **cancellation** (จะอธิบายเต็มรูปแบบในหัวข้อ 48.10) มาดูกรณี panic ก่อน — เหมือนกับที่
Part 37 พิสูจน์ว่า panic ใน thread ลูกไม่ทำให้ทั้งโปรแกรม crash Tokio ก็มีกลไกเดียวกันสำหรับ task:

```rust
#[tokio::main]
async fn main() {
    let panicking = tokio::spawn(async {
        panic!("task ลูก panic ตั้งใจ");
    });
    let panic_result = panicking.await;
    println!(
        "ผลลัพธ์จาก task ที่ panic: is_err={} is_panic={}",
        panic_result.is_err(),
        panic_result.as_ref().err().map(|e| e.is_panic()).unwrap_or(false)
    );
}
```

รันจริงได้ (แสดง panic message ที่ Tokio จับไว้บน worker thread ของมันเอง แล้วโปรแกรมยังทำงานต่อได้ปกติ):

```
thread 'tokio-rt-worker' panicked at src/main.rs:4:9:
task ลูก panic ตั้งใจ
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
ผลลัพธ์จาก task ที่ panic: is_err=true is_panic=true
```

เหมือนกับ Part 37: panic ของ task ลูก**ไม่ทำให้ทั้งโปรแกรม crash** — มันถูก catch ไว้ที่ขอบของ task นั้น แล้ว
แปลงเป็น `Err(JoinError)` ที่มี `.is_panic()` ให้เช็คว่าสาเหตุคือ panic (เทียบกับ cancellation) ได้ตรง ๆ
`main` (หรือ task อื่น) ยังทำงานต่อไปได้ปกติทุกประการ ตราบใดที่ไม่ `.unwrap()` ทับ `Err` นั้นซ้ำ (เหตุผลเดียวกับ
กับดักที่ 4 ของ Part 37)

### 48.6 Task เบากว่า OS Thread แค่ไหน: พิสูจน์ด้วยตัวเลขจริง

นี่คือเหตุผลเชิงปฏิบัติที่สำคัญที่สุดว่าทำไม async/Tokio ถึงมีประโยชน์จริง — **task ของ Tokio เบากว่า OS thread
มากจนสามารถสร้างได้เป็นหมื่นเป็นแสนตัวพร้อมกันโดยไม่มีปัญหา** ในขณะที่ Part 37 หัวข้อ "กับดักที่พบบ่อย" ข้อ 7
พิสูจน์ไปแล้วว่า spawn OS thread 10,000 ตัวสำหรับงานเล็ก ๆ ใช้เวลาหลักร้อยมิลลิวินาที (411ms) เพราะ overhead ของ
การสร้าง/ทำลาย thread ระดับ OS

มาพิสูจน์ฝั่ง Tokio ด้วยโจทย์ที่ยากกว่าเดิม — spawn **20,000 task** ที่แต่ละตัว `sleep` 50 มิลลิวินาที (จำลองงาน
I/O-bound ที่ต้อง "รอ" จริง ไม่ใช่แค่บวกเลข) เทียบกับ spawn OS thread ที่ทำงานเดียวกัน (ใช้แค่ 1,000 thread
สำหรับฝั่ง OS thread เพราะ 20,000 OS thread แต่ละตัวขอ stack memory จริงหลาย MB จะใช้ memory มหาศาลจนอาจทำให้
เครื่องทดสอบมีปัญหา — ตัวเลขที่ใหญ่กว่ากันถึง 20 เท่าเช่นนี้ยิ่งทำให้ผลลัพธ์ชัดเจนขึ้นไปอีก):

```rust
use std::time::{Duration, Instant};

#[tokio::main]
async fn main() {
    let n: usize = 20_000;

    // 20,000 tokio task ที่ sleep 50ms พร้อมกัน
    let start = Instant::now();
    let mut handles = Vec::with_capacity(n);
    for i in 0..n {
        handles.push(tokio::spawn(async move {
            tokio::time::sleep(Duration::from_millis(50)).await;
            i
        }));
    }
    let mut total: usize = 0;
    for h in handles {
        total += h.await.unwrap();
    }
    let tokio_elapsed = start.elapsed();
    println!("tokio::spawn {n} tasks sleep 50ms พร้อมกัน: ใช้เวลา {tokio_elapsed:?} (ผลรวม {total})");

    // เทียบ OS thread จำนวนน้อยกว่ามาก (1,000 ตัว)
    let n_os: usize = 1_000;
    let start2 = Instant::now();
    let mut os_handles = Vec::with_capacity(n_os);
    for i in 0..n_os {
        os_handles.push(std::thread::spawn(move || {
            std::thread::sleep(Duration::from_millis(50));
            i
        }));
    }
    let mut total_os: usize = 0;
    for h in os_handles {
        total_os += h.join().unwrap();
    }
    let os_elapsed = start2.elapsed();
    println!("std::thread::spawn {n_os} OS threads sleep 50ms พร้อมกัน: ใช้เวลา {os_elapsed:?} (ผลรวม {total_os})");
}
```

รันจริง (build แบบ `--release`) บนเครื่องที่ใช้เขียนบทนี้ (4 logical CPU core) ได้:

```
tokio::spawn 20000 tasks sleep 50ms พร้อมกัน: ใช้เวลา 68.021536ms (ผลรวม 199990000)
std::thread::spawn 1000 OS threads sleep 50ms พร้อมกัน: ใช้เวลา 118.786072ms (ผลรวม 499500)
```

ตัวเลขนี้น่าทึ่งมาก: **20,000 tokio task** (มากกว่าฝั่ง OS thread ถึง 20 เท่า) ใช้เวลารวมแค่ **68ms** — ใกล้เคียง
กับเวลา `sleep(50ms)` ตามธรรมชาติมาก (ส่วนต่างคือ overhead ของการ spawn/schedule 20,000 task ซึ่งน้อยมาก) ในขณะ
ที่ **1,000 OS thread** (น้อยกว่าฝั่ง tokio 20 เท่า) กลับใช้เวลา **118ms** — มากกว่าเวลา sleep ตามธรรมชาติถึงเกือบ
2.4 เท่า เพราะต้นทุนการสร้าง/ทำลาย OS thread จริงแต่ละตัว (ขอ stack memory จาก OS, ลงทะเบียนกับ scheduler)
สะสมกันจนกลายเป็นค่าใช้จ่ายที่มองเห็นได้ชัดแม้จะมีจำนวนน้อยกว่ามาก

**เหตุผลเชิงลึกที่ทำให้ต่างกันขนาดนี้**: task ของ Tokio ไม่ต้องขอ stack memory ก้อนใหม่จาก OS (มันถูกจัดสรรใน
heap แบบที่ควบคุมได้ภายใน Rust เอง ขนาดพอดีกับ state ที่ future นั้นต้องเก็บจริง ๆ ซึ่งมักเล็กกว่า stack ขนาด
เต็มของ OS thread หลาย MB มาก) ไม่ต้องให้ OS kernel มาลงทะเบียน scheduling entity ใหม่ และการ "สลับงาน" ระหว่าง
task (ตอน `.await` ที่ยังไม่พร้อม) เป็นแค่การสลับ pointer ของ future ที่ต้อง poll ต่อไปในระดับ userspace ล้วน ๆ
— ไม่ต้องผ่าน context switch ระดับ OS ที่มีต้นทุนสูงกว่ามาก นี่คือความหมายที่แท้จริงของคำว่า **"lightweight
task"** ที่มักได้ยินเวลาพูดถึง async runtime

### 48.7 `tokio::spawn` ต้องการ `Send + 'static`: คู่เทียบเป๊ะของ Part 40

ทวนอีกครั้งจาก signature หัวข้อ 48.5: `tokio::spawn` ต้องการ future ที่เป็น **`Send + 'static`** และค่า return
ก็ต้องเป็น **`Send + 'static`** ด้วย — bound เดียวกันเป๊ะกับที่ `thread::spawn` ต้องการจาก Part 37/40 และด้วย
**เหตุผลเดียวกันเป๊ะ**

#### ทำไมต้อง `'static`: Task อาจถูกย้ายไปทำงานบน Worker Thread ไหนก็ได้ เมื่อไหร่ก็ได้

เหตุผลของ `'static` เหมือนกับ Part 37 หัวข้อ 37.5 ทุกประการ: task ที่ spawn ไปมี **อายุที่เป็นอิสระ** จาก scope
ที่สร้างมันขึ้นมา — Tokio scheduler ไม่รู้ว่า task นี้จะถูก poll เมื่อไหร่ (อาจถูก poll ครั้งแรกทันที หรือถูก
"เก็บไว้ก่อน" ถ้า worker thread ทุกตัวไม่ว่างในตอนนั้น) และเมื่อ task ถูก poll มันอาจถูก poll บน **worker thread
ตัวไหนก็ได้** ในกรณี `multi_thread` flavor (แม้จะเป็น task เดิม อาจถูก poll ครั้งแรกบน thread A แล้วครั้งต่อมา
ถูก poll ต่อบน thread B ก็ได้ ถ้า work-stealing scheduler ตัดสินใจย้ายมันไป) — ถ้า future ที่ spawn ไปยืม
reference จาก scope เดิม compiler ไม่มีทางพิสูจน์ได้ว่า reference นั้นจะยังไม่ dangling ตอนที่ task ถูก poll
จริง ๆ

ลองส่ง async block ที่ยืมตัวแปร local โดยไม่ `move`:

```rust
#[tokio::main]
async fn main() {
    let numbers = vec![1, 2, 3];
    let handle = tokio::spawn(async {
        let total: i32 = numbers.iter().sum();
        println!("{total}");
    });
    let _ = handle.await;
}
```

```
error[E0373]: async block may outlive the current function, but it borrows `numbers`, which is owned by the current function
 --> src/main.rs:4:31
  |
4 |     let handle = tokio::spawn(async {
  |                               ^^^^^ may outlive borrowed value `numbers`
5 |         let total: i32 = numbers.iter().sum();
  |                          ------- `numbers` is borrowed here
  |
  = note: async blocks are not executed immediately and must either take a reference or ownership of outside variables they use
help: to force the async block to take ownership of `numbers` (and any other referenced variables), use the `move` keyword
  |
4 |     let handle = tokio::spawn(async move {
  |                                     ++++
```

**นี่คือ E0373 ตัวเดียวกันเป๊ะ**กับที่ Part 37 หัวข้อ 37.5 เจอตอนพยายามส่ง closure ที่ยืมตัวแปร local เข้า
`thread::spawn` — ต่างกันแค่คำว่า "closure" เปลี่ยนเป็น "async block" และ help message ก็แนะนำ `move` เหมือนกัน
เป๊ะ วิธีแก้ก็เหมือนกันเป๊ะ:

```rust
#[tokio::main]
async fn main() {
    let numbers = vec![1, 2, 3];
    let handle = tokio::spawn(async move {
        let total: i32 = numbers.iter().sum();
        println!("{total}");
    });
    let _ = handle.await;
}
```

compile และรันได้ปกติ — `move` ยึด ownership ของ `numbers` เข้าไปในตัว async block ทั้งก้อน ทำให้มันไม่ยืม
อะไรจาก scope ภายนอกอีกต่อไป จึงเป็น `'static` ได้จริง (เหตุผลเชิงลึกเหมือนกับ Part 37 หัวข้อ 37.5 ทุกคำ)

#### ทำไมต้อง `Send`: Task อาจถูก Poll บน Worker Thread ที่ต่างกันจริง

จำได้จากหัวข้อ 48.4 ว่า `multi_thread` flavor มี worker thread หลายตัวทำงานคู่ขนานจริง และ task หนึ่งตัวอาจถูก
work-stealing scheduler ย้ายจาก thread หนึ่งไปยังอีก thread ระหว่างทาง (ทุกครั้งที่ task ถูก poll ใหม่หลังจาก
`.await` คืนค่า `Pending` แล้วถูกปลุกกลับมา มันมีโอกาสถูก poll บน worker thread คนละตัวจากครั้งก่อน) — นี่แปลว่า
**ข้อมูลทั้งหมดที่ future เก็บไว้ข้ามจุด `.await` ต้องปลอดภัยที่จะ "ย้าย" ข้าม thread ได้จริง** ซึ่งคือความหมาย
ของ trait `Send` ที่ Part 40 สอนไว้เต็มรูปแบบ — **เหตุผลเชิงลึกเหมือนกับที่ `thread::spawn` ต้องการ `Send`
ทุกประการ** เพียงแต่ตอนนี้ "การย้ายข้าม thread" ไม่ได้เกิดครั้งเดียวตอน spawn (แบบ OS thread) แต่อาจเกิด**ซ้ำ
หลายครั้ง**ตลอดชีวิตของ task หนึ่งตัว (ทุกครั้งที่มันถูก poll ใหม่)

ลองพยายามใช้ `Rc<T>` (ที่ Part 28 บอกไว้ตรง ๆ ว่าไม่ thread-safe และ Part 37/40 พิสูจน์ไปแล้วว่าไม่ `Send`) ใน
task ที่ spawn:

```rust
use std::rc::Rc;

#[tokio::main]
async fn main() {
    let data = Rc::new(5);
    let handle = tokio::spawn(async move {
        println!("{}", data);
    });
    let _ = handle.await;
}
```

```
error: future cannot be sent between threads safely
 --> src/main.rs:6:18
  |
6 |       let handle = tokio::spawn(async move {
  |  __________________^
7 | |         println!("{}", data);
8 | |     });
  | |______^ future created by async block is not `Send`
  |
  = help: within `{async block@src/main.rs:6:31: 6:41}`, the trait `Send` is not implemented for `Rc<i32>`
note: captured value is not `Send`
 --> src/main.rs:7:24
  |
7 |         println!("{}", data);
  |                        ^^^^ has type `Rc<i32>` which is not `Send`
note: required by a bound in `tokio::spawn`
   --> .../tokio-1.53.1/src/task/spawn.rs:176:21
    |
174 |     pub fn spawn<F>(future: F) -> JoinHandle<F::Output>
    |            ----- required by a bound in this function
175 |     where
176 |         F: Future + Send + 'static,
    |                     ^^^^ required by this bound in `spawn`
```

สังเกตข้อความ error: **"future cannot be sent between threads safely"** — คำเตือนตรงตัวมากว่า future ที่สร้าง
จาก async block นี้ไม่ปลอดภัยที่จะถูกส่งข้าม thread เพราะมันเก็บ `Rc<i32>` ไว้ข้ามจุด `.await`... รอสักครู่
ในตัวอย่างนี้ไม่มี `.await` เลยด้วยซ้ำ แต่ error ยังเกิดขึ้น เพราะ **การตรวจสอบ `Send` ของ compiler มองว่า
future ทั้งก้อน (รวมถึงทุกอย่างที่มันอาจ capture ไว้) ต้องเป็น `Send`** ไม่ว่าจะมีจุด `.await` คั่นระหว่างการใช้
ค่านั้นหรือไม่ — เหตุผลคือ future ตัวนี้ถูกส่งเข้า `tokio::spawn` เพื่อกลายเป็น task ที่มีโอกาสถูก poll ครั้งแรก
บน worker thread ใดก็ได้ตั้งแต่แรกอยู่ดี (ไม่ต้องรอ `.await` คืนค่า `Pending` ก่อน) `note: captured value is
not Send ... has type Rc<i32>` ระบุสาเหตุตรงจุดชัดเจน และ `required by a bound in tokio::spawn` ยืนยันว่านี่คือ
bound `Send` จาก signature ที่เราอ่านไปในหัวข้อ 48.5

**วิธีแก้เหมือนกับ Part 37/40**: เปลี่ยนเป็น `Arc<T>` (thread-safe reference counting) แทน `Rc<T>`:

```rust
use std::sync::Arc;

#[tokio::main]
async fn main() {
    let data = Arc::new(5);
    let handle = tokio::spawn(async move {
        println!("{}", data);
    });
    let _ = handle.await;
}
```

compile และรันได้ปกติ พิมพ์ `5` ออกมา — เพราะ `Arc<i32>` implement `Send` (ใช้ atomic operation สำหรับตัวนับ
reference แทน `Cell` ธรรมดาที่ไม่ thread-safe ของ `Rc`) ตามที่ Part 39/40 อธิบายไว้เต็มรูปแบบ

**บทเรียนสำคัญของหัวข้อนี้**: ทุกอย่างที่คุณเรียนรู้จาก Part 37/40 เรื่อง `Send`/`'static` **ไม่ได้เป็นความรู้
เฉพาะของ OS thread เลย** — มันคือกฎเรื่อง "ความปลอดภัยตอนย้ายข้าม thread" ที่เป็นสากล ใช้ได้กับทุกกลไก
concurrency ใน Rust ไม่ว่าจะเป็น thread ดิบหรือ async runtime อย่าง Tokio ก็ตาม เพราะรากฐานคือ compiler ตรวจสอบ
trait bound แบบเดียวกัน เพียงแต่บริบทของ "การย้าย" ต่างกัน (ย้ายทั้งก้อนตอน spawn ใน thread, ย้ายซ้ำได้หลายครั้ง
ตอน poll ใหม่ใน task)

### 48.8 `.await` ในทางปฏิบัติ: `tokio::time::sleep` เทียบกับ `thread::sleep`

Part 37 หัวข้อ 37.3 ใช้ `thread::sleep(Duration::from_millis(...))` เพื่อ**สาธิต**จังหวะเวลาให้เห็นผลชัดเจน แต่
เตือนไว้ในกับดักข้อ 6 ว่าห้ามใช้เป็นเครื่องมือ synchronize จริงจัง — **`tokio::time::sleep`** คือคู่เทียบฝั่ง
async ของ `thread::sleep` แต่มีความต่างเชิงกลไกที่สำคัญที่สุดของทั้งบทนี้:

```rust
use tokio::time::sleep;
use std::time::Duration;

async fn demo() {
    println!("ก่อน sleep");
    sleep(Duration::from_millis(100)).await; // ไม่ block thread!
    println!("หลัง sleep");
}
```

**`thread::sleep` บล็อก OS thread ทั้งเส้นให้หยุดนิ่งสนิท** — ระหว่างที่ thread กำลัง sleep มันทำอะไรอื่นไม่ได้
เลยจนกว่าเวลาจะหมด (ถ้าเป็น worker thread ของ Tokio ที่ดันไปเรียก `thread::sleep` มันจะพา task อื่น ๆ ที่ควรได้
ทำงานบน worker thread นั้นให้ค้างไปด้วย — นี่คือกับดักสำคัญที่หัวข้อ 48.9 จะพิสูจน์ให้เห็นชัด ๆ) ในทางตรงข้าม
**`tokio::time::sleep(...).await` ไม่บล็อก worker thread เลย** — มันแค่บอก Tokio scheduler ว่า "task นี้ยังไม่
พร้อมทำงานต่อ จะพร้อมอีกทีตอนเวลาผ่านไปเท่านี้" แล้ว**คืนการควบคุมของ worker thread ให้ทันที** ไปทำ task อื่นที่
พร้อมทำงานอยู่ในคิว จนกว่า Tokio timer จะ "ปลุก" (ผ่าน `Waker` จาก Part 47) task นี้กลับมาพอดีเมื่อเวลาผ่านไป
ครบตามที่ตั้งไว้

มาพิสูจน์ความต่างนี้ด้วยตัวเลขจริง — คุณเห็นในหัวข้อ 48.6 ไปแล้วว่า 20,000 task ที่ `tokio::time::sleep` 50ms
พร้อมกันใช้เวลารวมแค่ 68ms (ใกล้เคียงกับเวลา sleep ตามธรรมชาติมาก) — นี่คือหลักฐานที่ชัดเจนที่สุดว่า
`tokio::time::sleep` **ไม่ได้กิน OS thread คนละตัวต่องาน** เพราะถ้ามันบล็อกแบบ `thread::sleep` จริง ระบบจะต้อง
มี OS thread จริง 20,000 ตัวมารอพร้อมกัน (ซึ่งเราเห็นแล้วว่าแค่ 1,000 OS thread ก็ใช้เวลามากกว่านี้เกือบ 2 เท่า)
— แต่ Tokio ทำสิ่งนี้ได้ด้วย **timer เดียว** ที่คอยดูรายการ task ที่รอ deadline ต่างกัน แล้วปลุกให้ถูกต้องตรงเวลา
โดยใช้แค่ worker thread จำนวนน้อย (4 thread ตามจำนวน CPU core) รัน 20,000 task สลับกันเท่านั้น

### 48.9 Concurrent Execution: `tokio::join!` เทียบกับ `.await` ตามลำดับ

โจทย์คลาสสิกที่สุดของ async programming: มีงาน I/O-bound หลายอย่างที่ **ไม่ต้องพึ่งพากัน** ต้องการรอให้ครบทุก
อย่างก่อนไปต่อ — ถ้า `.await` ทีละตัวตามลำดับ เวลารวมจะเป็น**ผลรวม**ของเวลาทุกงาน แต่ถ้าให้ทำงาน**พร้อมกัน**
เวลารวมจะเป็นแค่เวลาของ**งานที่ช้าที่สุด**เท่านั้น

```rust
use std::time::{Duration, Instant};

async fn fetch(name: &str, ms: u64) -> String {
    tokio::time::sleep(Duration::from_millis(ms)).await;
    format!("ผลลัพธ์จาก {name}")
}

#[tokio::main]
async fn main() {
    // แบบ sequential: .await ทีละตัว
    let start = Instant::now();
    let a = fetch("service-a", 100).await;
    let b = fetch("service-b", 100).await;
    let c = fetch("service-c", 100).await;
    let sequential_elapsed = start.elapsed();
    println!("sequential .await ทีละตัว: {sequential_elapsed:?} -> {a}, {b}, {c}");

    // แบบ concurrent ด้วย tokio::join!
    let start2 = Instant::now();
    let (a2, b2, c2) = tokio::join!(
        fetch("service-a", 100),
        fetch("service-b", 100),
        fetch("service-c", 100)
    );
    let joined_elapsed = start2.elapsed();
    println!("tokio::join! พร้อมกัน:            {joined_elapsed:?} -> {a2}, {b2}, {c2}");

    let speedup = sequential_elapsed.as_secs_f64() / joined_elapsed.as_secs_f64();
    println!("อัตราเร็วขึ้นจริง: {speedup:.2}x");
}
```

รันจริงบนเครื่องที่ใช้เขียนบทนี้ (จำลอง 3 งานที่ต้อง "รอ" 100ms เท่ากัน เหมือนที่โจทย์ของบทนี้ตั้งไว้):

```
sequential .await ทีละตัว: 304.000952ms -> ผลลัพธ์จาก service-a, ผลลัพธ์จาก service-b, ผลลัพธ์จาก service-c
tokio::join! พร้อมกัน:            101.61962ms -> ผลลัพธ์จาก service-a, ผลลัพธ์จาก service-b, ผลลัพธ์จาก service-c
อัตราเร็วขึ้นจริง: 2.99x
```

ตัวเลขจริงตรงกับที่คาดไว้เป๊ะ: **sequential ใช้เวลา ~304ms** (เกือบเท่ากับ 3 × 100ms พอดี บวก overhead เล็กน้อย)
ในขณะที่ **`join!` ใช้เวลา ~102ms** (เกือบเท่ากับ 1 × 100ms — เวลาของงานที่ช้าที่สุดเพียงงานเดียว) คิดเป็น
**อัตราเร็วขึ้นจริง 2.99 เท่า** ใกล้เคียงกับตัวเลขทฤษฎี 3 เท่าที่คาดหวังจากการทำ 3 งานพร้อมกันมาก

**อธิบายกลไกที่แท้จริง**: `tokio::join!(fut1, fut2, fut3)` **ไม่ได้ spawn task ใหม่แต่อย่างใด** — มันคือ macro
ที่สร้าง future ตัวใหม่ที่ poll ทั้ง `fut1`, `fut2`, `fut3` **ในการ poll ครั้งเดียวกัน** (วนเรียก `.poll()` ของ
ทุกตัวที่ยังไม่เสร็จทุกครั้งที่ future รวมนี้ถูก poll) แล้วคืนค่าเมื่อ**ทุกตัวเสร็จครบ** — เพราะ future ที่รอ
I/O (เช่น `sleep`) ตอน `.poll()` แล้วยังไม่พร้อมจะคืน `Pending` ทันทีโดยไม่บล็อกอะไรเลย (ตามกลไก `Future`/`Poll`
จาก Part 46-47) การ poll หลาย future แบบสลับกันไปมาในรอบเดียวจึงทำให้ทั้งสามงาน "รอพร้อมกัน" ได้จริงบน task
เดียวเท่านั้น (ไม่ต้องมีหลาย OS thread หรือหลาย task เลยด้วยซ้ำ!) — นี่ต่างจาก `tokio::spawn` ที่สร้าง task
ใหม่จริง ๆ ที่ scheduler ดูแลแยกกัน (`join!` เหมาะกับ future จำนวนไม่มากที่รู้ล่วงหน้าตอน compile time ว่ามีกี่
ตัว ส่วน `tokio::spawn` เหมาะกับงานที่ต้องการให้ scheduler จัดการแบบเป็นอิสระจากกันจริง ๆ หรือจำนวนไม่แน่นอน)

### 48.10 `tokio::select!`: แข่งกันระหว่าง Future — ใครเสร็จก่อนไปก่อน

ในขณะที่ `join!` รอ**ทุกตัว**ให้เสร็จ **`select!`** ทำตรงข้าม — มันรอ**ตัวใดตัวหนึ่ง**ที่เสร็จก่อนแล้วเลิกสนใจ
ตัวที่เหลือทันที (drop ทิ้งไป) นี่คือ pattern พื้นฐานของการทำ **timeout**:

```rust
use std::time::Duration;
use tokio::time::sleep;

async fn slow_operation() -> &'static str {
    sleep(Duration::from_millis(500)).await;
    "ผลลัพธ์จาก slow_operation (กว่าจะเสร็จ)"
}

async fn with_manual_select_timeout() {
    tokio::select! {
        result = slow_operation() => {
            println!("select!: slow_operation เสร็จก่อน -> {result}");
        }
        _ = sleep(Duration::from_millis(100)) => {
            println!("select!: timeout 100ms ถึงก่อน -> ยกเลิกการรอ slow_operation");
        }
    }
}
```

รันจริง (สังเกตว่า `slow_operation` ใช้เวลา 500ms แต่ timeout ตั้งไว้แค่ 100ms):

```
select!: timeout 100ms ถึงก่อน -> ยกเลิกการรอ slow_operation
```

`slow_operation` ใช้เวลา 500ms แต่แขนง `sleep(100ms)` เสร็จก่อน — `select!` จึงเลือกทำงานเฉพาะ block ของแขนงที่
ชนะ (`timeout 100ms ถึงก่อน...`) และ**drop future ของแขนงที่แพ้ทันที** (future ของ `slow_operation()` ที่ยัง
ทำงานไม่จบถูกทิ้งไปเลย ไม่มีทางกลับมาทำงานต่อได้อีก — รายละเอียดผลกระทบของการ drop future กลางทางจะอธิบายเต็ม
รูปแบบในหัวข้อ 48.12)

**Tokio ให้ helper สำเร็จรูปสำหรับ pattern timeout** เพราะมันพบบ่อยมากจนไม่ต้องเขียน `select!` มือเองทุกครั้ง —
**`tokio::time::timeout`**:

```rust
match tokio::time::timeout(Duration::from_millis(100), slow_operation()).await {
    Ok(result) => println!("timeout(): ทำสำเร็จ -> {result}"),
    Err(_) => println!("timeout(): หมดเวลาก่อน slow_operation จะเสร็จ (Elapsed)"),
}
```

```
timeout(): หมดเวลาก่อน slow_operation จะเสร็จ (Elapsed)
```

และถ้าให้เวลามากพอ (มากกว่า 500ms ที่ `slow_operation` ต้องการ):

```rust
match tokio::time::timeout(Duration::from_millis(1000), slow_operation()).await {
    Ok(result) => println!("timeout() (เวลาเยอะพอ): ทำสำเร็จ -> {result}"),
    Err(_) => println!("timeout(): หมดเวลา"),
}
```

```
timeout() (เวลาเยอะพอ): ทำสำเร็จ -> ผลลัพธ์จาก slow_operation (กว่าจะเสร็จ)
```

`tokio::time::timeout(duration, future)` คืนค่าเป็น `Result<T, Elapsed>` — `Ok(T)` ถ้า future เสร็จภายในเวลาที่
กำหนด, `Err(Elapsed)` ถ้าไม่เสร็จทัน (ภายใน implementation จริงของมันคือ `select!` ระหว่าง future ที่ให้มากับ
`sleep(duration)` แบบเดียวกับที่เราเขียนมือเองข้างบนเป๊ะ — เป็นแค่ wrapper ที่กระชับกว่า) ตัว `select!` ดิบยังมี
ประโยชน์เมื่อต้องแข่งกันระหว่าง future มากกว่า 2 ตัว หรือต้องการทำอะไรเฉพาะเจาะจงกับแขนงที่ชนะ/แพ้ต่างกัน (เช่น
รับ message จาก channel หลายอันพร้อมกัน ซึ่งจะเรียนเต็มรูปแบบใน Part 50)

### 48.11 กับดักที่อันตรายที่สุด: Blocking Code ในงาน Async

ถึงจุดที่สำคัญที่สุดของทั้งบทนี้ — เรื่องที่ทำให้โปรแกรม Tokio จำนวนมากในโลกจริง "ช้าอย่างประหลาด" หรือ
"ค้างไม่มีเหตุผล" ทั้งที่โค้ด compile ผ่านและดูปกติทุกอย่าง

#### ทำไมการเรียก Blocking Code ตรงในงาน Async ถึงอันตรายมาก

จำหลักการจากหัวข้อ 48.5-48.6: task ของ Tokio ทำงานคู่กันบน **worker thread จำนวนน้อย** (thread pool ขนาดคงที่)
ที่ scheduler สลับ poll task ต่าง ๆ ไปมาอย่างรวดเร็ว — สิ่งที่ทำให้การสลับนี้ทำงานได้คือ **ทุก task ต้อง "คืน
การควบคุม" กลับให้ scheduler เป็นระยะ ๆ** (ทุกจุด `.await` ที่ยังไม่พร้อมคือจุดคืนการควบคุม) นี่คือความหมายของ
คำว่า **cooperative scheduling** — task ต้อง "ให้ความร่วมมือ" กับ scheduler โดยสมัครใจ ไม่มีใครมาบังคับสลับให้

ถ้า task ตัวหนึ่งเรียกฟังก์ชันที่ **บล็อกจริง ๆ** (เช่น `std::thread::sleep`, การคำนวณหนัก ๆ ที่ใช้เวลานานโดย
ไม่มี `.await` คั่นเลย, หรือ I/O แบบ synchronous ของ `std` เช่น `std::fs::read`) **worker thread ทั้งเส้นที่
กำลังรัน task นั้นจะหยุดนิ่งสนิทไปกับมัน** — ไม่มีทางคืนการควบคุมกลับให้ scheduler ได้เลยจนกว่าฟังก์ชัน
blocking นั้นจะคืนค่า และ**ทุก task อื่นที่ควรถูก poll บน worker thread เดียวกันนั้นจะต้องรอไปด้วย** แม้ว่า
พวกมันจะพร้อมทำงานอยู่แล้วก็ตาม — นี่คือการทำลายจุดประสงค์ทั้งหมดของ async runtime: เราเลือกใช้ async เพื่อให้
worker thread จำนวนน้อยรองรับงานจำนวนมากได้อย่างมีประสิทธิภาพ แต่ blocking call ตรงกลางงาน async ทำให้ worker
thread นั้น "เสีย" ไปทั้งเส้นสำหรับ task เดียว เหมือนย้อนกลับไปมีปัญหาแบบเดียวกับ OS thread ที่ Part 37 เจอ
เพียงแต่แย่กว่า เพราะตอนนี้**task อื่นที่ไม่เกี่ยวข้องเลยก็โดนลากไปด้วย**

มาพิสูจน์ด้วยตัวเลขจริง — ใช้ `current_thread` flavor (worker thread เดียว) เพื่อให้เห็นผลชัดที่สุด (ปัญหา
เดียวกันเกิดใน `multi_thread` ได้เช่นกันถ้า task ที่ blocking มีจำนวนมากพอจะครองทุก worker thread แต่
`current_thread` ทำให้เห็นภาพง่ายกว่า):

```rust
use std::time::{Duration, Instant};

#[tokio::main(flavor = "current_thread")]
async fn main() {
    println!("=== กรณีผิด: เรียก std::thread::sleep (blocking) ตรงในงาน async ===");
    let start = Instant::now();

    let blocking_task = tokio::spawn(async {
        println!("  task A (blocking): เริ่มทำงาน แล้วจะ block worker thread 300ms");
        std::thread::sleep(Duration::from_millis(300)); // ผิด! block ทั้ง worker thread
        println!("  task A (blocking): ทำงานเสร็จ");
    });

    let b_start = Instant::now();
    let quick_task = tokio::spawn(async move {
        tokio::time::sleep(Duration::from_millis(10)).await; // แค่รอสั้น ๆ แบบ non-blocking
        println!(
            "  task B (ควรเสร็จเร็ว): ทำงานเสร็จหลังผ่านไป {:?} จากตอนที่ spawn",
            b_start.elapsed()
        );
    });

    let _ = tokio::join!(blocking_task, quick_task);
    println!("รวมเวลาที่ใช้ (แบบผิด): {:?}", start.elapsed());
}
```

รันจริงได้:

```
=== กรณีผิด: เรียก std::thread::sleep (blocking) ตรงในงาน async ===
  task A (blocking): เริ่มทำงาน แล้วจะ block worker thread 300ms
  task A (blocking): ทำงานเสร็จ
  task B (ควรเสร็จเร็ว): ทำงานเสร็จหลังผ่านไป 311.430401ms จากตอนที่ spawn
รวมเวลาที่ใช้ (แบบผิด): 311.515946ms
```

ดูตัวเลขนี้ให้ดี: **task B ตั้งใจ `sleep` แค่ 10ms** แต่กลับใช้เวลาจริง **311.43ms** ก่อนจะทำงานเสร็จ! เกือบ
เท่ากับเวลาที่ task A ใช้ทั้งหมด (300ms) เหตุผลคือ `current_thread` flavor มี worker thread เดียว — พอ task A
ถูก poll ก่อนแล้วเรียก `std::thread::sleep(300ms)` worker thread เส้นนั้น (เส้นเดียวที่มี!) ก็หยุดนิ่งไป 300ms
เต็ม ๆ ทำให้ task B ที่ควรจะเสร็จใน 10ms **ไม่มีโอกาสได้ถูก poll เลยจนกว่า task A จะคืนการควบคุม** — task B
ที่เขียนโค้ดว่า "รอแค่ 10ms" กลับกลายเป็นรอจริง 300ms กว่า เพราะถูกงานอื่นที่ไม่เกี่ยวข้องบล็อกไว้เฉย ๆ

**นี่คืออันตรายที่แท้จริง**: ในโปรแกรมจริงที่มี worker thread หลายตัว (`multi_thread`) ปัญหานี้อาจไม่แสดงตัว
ทันทีตอนโปรแกรมมีภาระงานน้อย (เพราะยังมี worker thread อื่นว่างอยู่คอย poll task ที่เหลือ) แต่พอ**ปริมาณงานสูง
ขึ้น**จนทุก worker thread มี task ที่ blocking ค้างอยู่ ทั้งระบบจะ "แข็ง" (freeze) พร้อมกันทั้งหมด — เป็นบั๊ก
ประเภทที่มักไม่โผล่ตอนทดสอบเบา ๆ แต่พังหนักตอน production รับ load จริง (คล้ายกับ Heisenbug ที่ Part 37 พูดถึง
แต่เกิดจากสาเหตุที่ต่างกัน)

#### วิธีแก้: `tokio::task::spawn_blocking()`

Tokio เตรียม **blocking thread pool** แยกต่างหากจาก worker thread ปกติไว้โดยเฉพาะ (ขนาดปรับได้ ค่าเริ่มต้นค่อน
ข้างใหญ่ เพราะ thread ในพูลนี้ถูกออกแบบมาให้ "ยอมบล็อกได้" อยู่แล้ว) — **`tokio::task::spawn_blocking(closure)`**
รับ closure ธรรมดา (ไม่ใช่ async!) แล้วส่งไปรันบน thread จากพูลนี้ คืนค่าเป็น `JoinHandle<T>` แบบเดียวกับ
`tokio::spawn` ที่ `.await` ได้ตามปกติ

```rust
use std::time::{Duration, Instant};

#[tokio::main(flavor = "current_thread")]
async fn main() {
    println!("=== กรณีถูก: ใช้ tokio::task::spawn_blocking สำหรับงาน blocking ===");
    let start2 = Instant::now();

    let blocking_task2 = tokio::task::spawn_blocking(|| {
        println!("  task A (spawn_blocking): เริ่มทำงานบน blocking thread pool แยก 300ms");
        std::thread::sleep(Duration::from_millis(300));
        println!("  task A (spawn_blocking): ทำงานเสร็จ");
    });

    let b_start2 = Instant::now();
    let quick_task2 = tokio::spawn(async move {
        tokio::time::sleep(Duration::from_millis(10)).await;
        println!(
            "  task B (ควรเสร็จเร็ว): ทำงานเสร็จหลังผ่านไป {:?} จากตอนที่ spawn",
            b_start2.elapsed()
        );
    });

    let _ = tokio::join!(blocking_task2, quick_task2);
    println!("รวมเวลาที่ใช้ (แบบถูก): {:?}", start2.elapsed());
}
```

รันจริงได้:

```
=== กรณีถูก: ใช้ tokio::task::spawn_blocking สำหรับงาน blocking ===
  task A (spawn_blocking): เริ่มทำงานบน blocking thread pool แยก 300ms
  task B (ควรเสร็จเร็ว): ทำงานเสร็จหลังผ่านไป 11.322974ms จากตอนที่ spawn
  task A (spawn_blocking): ทำงานเสร็จ
รวมเวลาที่ใช้ (แบบถูก): 300.859215ms
```

เทียบให้เห็นชัด ๆ: **task B ใช้เวลาจริงแค่ 11.32ms** (เทียบกับ 311.43ms ในกรณีผิด) — ใกล้เคียงกับ 10ms ที่ตั้งใจ
มากที่สุด เพราะ `std::thread::sleep(300ms)` ของ task A ถูกส่งไปทำงานบน **thread แยกจาก worker thread เดี่ยว
ของ `current_thread` runtime โดยสิ้นเชิง** — worker thread หลักยังว่างอยู่เต็มที่ พร้อม poll task B ได้ทันที
ที่ timer ของมันครบ ไม่ต้องรอ task A เลย สังเกตด้วยว่าลำดับการพิมพ์เปลี่ยนไป: **"task B ทำงานเสร็จ" พิมพ์ก่อน
"task A ทำงานเสร็จ"** ในกรณีที่แก้แล้ว (ตรงข้ามกับกรณีผิดที่ A ต้องพิมพ์เสร็จก่อน B เสมอ) — พิสูจน์ว่า
ทั้งสองงานทำงาน**คู่ขนานกันจริง**แล้ว

**หลักการเลือกใช้**: ใช้ `spawn_blocking` เมื่อมีโค้ดที่**เนื้อแท้แล้วเป็น synchronous/blocking** และไม่มี
เวอร์ชัน async ให้ใช้แทน — เช่น ฟังก์ชันจาก crate ที่เขียนแบบ synchronous ล้วน (บาง crate เข้ารหัส/ถอดรหัสข้อมูล,
crate ประมวลผลภาพ, การเรียก C library ผ่าน FFI จาก Part 43), การคำนวณที่ใช้ CPU หนักเป็นเวลานาน (เช่น hashing
รหัสผ่านแบบ CPU-intensive, การ parse ไฟล์ขนาดใหญ่แบบ synchronous), หรือ I/O แบบ synchronous ของ `std` ที่ยังไม่มี
เวอร์ชัน async ทดแทน — **ไม่ควรใช้ `spawn_blocking` เป็นค่าเริ่มต้นสำหรับทุกอย่าง** เพราะ blocking thread pool
มีต้นทุนของมันเอง (สร้าง/ทำลาย thread จริงคล้าย `std::thread::spawn`) งานที่มีเวอร์ชัน async ให้ใช้อยู่แล้ว
(เช่น I/O ผ่าน `tokio::fs`, `tokio::net` ที่จะเรียนใน Part 49) ควรใช้เวอร์ชัน async โดยตรงเสมอ เพราะเบากว่ามาก
ตามที่พิสูจน์ไว้ในหัวข้อ 48.6

### 48.12 Cancellation: ความเข้าใจที่ถูกต้องเกี่ยวกับการยกเลิก Task

หัวข้อนี้ต้องระมัดระวังเป็นพิเศษ เพราะมี**ความเข้าใจผิดที่พบบ่อยมาก**เกี่ยวกับพฤติกรรมนี้ในหมู่ผู้เขียนโค้ด Tokio
แล้วเราจะพิสูจน์พฤติกรรมที่ถูกต้องด้วยโค้ดจริงทีละกรณี

#### ความเข้าใจผิดที่พบบ่อย: "Drop `JoinHandle` แล้ว Task จะถูกยกเลิกทันที"

หลายคนคิดว่า drop `JoinHandle<T>` (เช่น ไม่เก็บค่าที่ `tokio::spawn` คืนมาเลย หรือเก็บไว้ในตัวแปรที่ scope จบ)
จะทำให้ task ที่มันอ้างอิงถูกยกเลิกทันที เพราะคุ้นเคยกับหลักการ RAII จาก Part 6 ที่ทรัพยากรส่วนใหญ่ถูกคืน/ทำลาย
เมื่อ handle ของมันถูก drop — **แต่นี่ไม่ใช่พฤติกรรมของ `tokio::task::JoinHandle`** มาพิสูจน์ให้เห็นด้วยตากัน:

```rust
use std::time::Duration;

struct CleanupGuard(&'static str);
impl Drop for CleanupGuard {
    fn drop(&mut self) {
        println!("  Drop::drop เรียกทำงาน: cleanup ของ '{}' ทำงานแล้ว", self.0);
    }
}

#[tokio::main]
async fn main() {
    println!("=== Drop JoinHandle ไม่ยกเลิก task — task ทำงานต่อจนจบ (detached) ===");
    let handle = tokio::spawn(async {
        let _guard = CleanupGuard("task-detached");
        tokio::time::sleep(Duration::from_millis(50)).await;
        println!("  task-detached: ทำงานจนจบสมบูรณ์ แม้ handle จะถูก drop ไปแล้ว");
    });
    drop(handle); // แค่ drop handle เฉย ๆ ไม่เรียก .abort()
    tokio::time::sleep(Duration::from_millis(150)).await; // รอให้เห็นว่า task ยังทำงานต่อจริง
}
```

รันจริงได้:

```
=== Drop JoinHandle ไม่ยกเลิก task — task ทำงานต่อจนจบ (detached) ===
  task-detached: ทำงานจนจบสมบูรณ์ แม้ handle จะถูก drop ไปแล้ว
  Drop::drop เรียกทำงาน: cleanup ของ 'task-detached' ทำงานแล้ว
```

พิสูจน์ชัดเจนแล้ว: **task ทำงานจนจบสมบูรณ์** แม้ว่า `handle` จะถูก `drop` ไปตั้งแต่ก่อนที่ task จะเริ่ม sleep
ด้วยซ้ำ — สิ่งที่เกิดขึ้นจริงคือ task กลายเป็น **"detached"** (แยกตัวเป็นอิสระ) มันยังถูก Tokio scheduler
poll ต่อไปจนเสร็จตามปกติทุกอย่าง เพียงแต่ไม่มีใครสามารถ `.await` เพื่อรอผลลัพธ์ หรือเช็คสถานะของมันได้อีกแล้ว
(เพราะ handle ที่เป็นทางเชื่อมเดียวถูกทิ้งไปแล้ว) — นี่คือพฤติกรรมที่ **คล้ายกับ OS thread ที่ไม่ join ใน Part 37
มากกว่าที่คิด**: ทั้งสองกรณี งานยังทำงานต่อไปเป็นอิสระ ต่างกันแค่ที่ **OS thread ที่ไม่ join จะถูกตัดจบไปด้วยถ้า
`main` จบก่อน** (เพราะทั้งโปรแกรมปิด) ในขณะที่ **task ของ Tokio ที่ detached จะทำงานต่อไปจนเสร็จตราบใดที่ runtime
ยังไม่ shutdown** (ในตัวอย่างนี้ `main` รอด้วย `sleep(150ms)` พอดีจนกว่า task จะทำงานจบเอง)

#### วิธียกเลิก Task จริง ๆ: `.abort()`

ถ้าต้องการยกเลิก task จริง ๆ ต้องเรียก **`JoinHandle::abort()`** ตรง ๆ:

```rust
println!("=== .abort() ยกเลิก task จริง — Drop cleanup ยังทำงาน ===");
let handle = tokio::spawn(async {
    let _guard = CleanupGuard("task-aborted");
    tokio::time::sleep(Duration::from_millis(500)).await;
    println!("  task-aborted: บรรทัดนี้ไม่ควรถูกพิมพ์เลย");
});
tokio::time::sleep(Duration::from_millis(20)).await; // ให้ task เริ่มทำงานก่อน
handle.abort();
let result = handle.await;
println!("  ผลลัพธ์หลัง abort: is_cancelled = {}", result.unwrap_err().is_cancelled());
```

รันจริงได้:

```
=== .abort() ยกเลิก task จริง — Drop cleanup ยังทำงาน ===
  Drop::drop เรียกทำงาน: cleanup ของ 'task-aborted' ทำงานแล้ว
  ผลลัพธ์หลัง abort: is_cancelled = true
```

สังเกตสองอย่าง: (1) **บรรทัด "บรรทัดนี้ไม่ควรถูกพิมพ์เลย" ไม่ถูกพิมพ์จริง** — task ถูกยกเลิกก่อนที่จะไปถึงบรรทัด
นั้น (มันค้างอยู่ที่ `sleep(500ms)` พอดีตอนถูก `.abort()` เพราะเราให้เวลาแค่ 20ms ก่อนเรียก) (2) **แต่ `Drop::drop`
ของ `CleanupGuard` ยังทำงาน** — พิมพ์ข้อความ cleanup ออกมาก่อนที่ผลลัพธ์ `is_cancelled = true` จะถูกพิมพ์ด้วยซ้ำ
นี่คือจุดสำคัญที่สุดของ cancellation ใน Rust async: **การยกเลิก future (ไม่ว่าจะผ่าน `.abort()` หรือกรณี
`select!`/`timeout` ที่จะเห็นต่อไป) ทำโดยการ `drop` future นั้นทิ้ง** — และตาม RAII ที่ Part 6 สอนไว้ **ทุกค่า
ที่ implement `Drop` และยังมีชีวิตอยู่ ณ จุดที่ future ถูก drop จะมี `Drop::drop` ของมันถูกเรียกตามปกติทุก
ประการ** เหมือนกับตัวแปร local ธรรมดาที่ scope จบ — นี่คือเหตุผลที่ cancellation ของ Rust async "ปลอดภัย" โดย
ธรรมชาติ: ไม่ว่า task จะถูกยกเลิกกลางทางตรงจุดไหน คุณ**การันตีได้เสมอ**ว่าทรัพยากรที่ถือครองไว้ (lock guard,
file handle, connection ที่ต้องปิด) จะถูกคืน/ปิดอย่างถูกต้องผ่าน `Drop` เช่นเดียวกับที่ Rust การันตีเรื่องนี้
ในโค้ด synchronous ทุกที่มาตั้งแต่ Part 6

`JoinError` ที่ได้จาก `.await` หลัง `.abort()` มี method **`.is_cancelled()`** ให้เช็คสาเหตุตรง ๆ (คู่กับ
`.is_panic()` ที่เห็นในหัวข้อ 48.5) — สามารถแยกแยะได้ว่า task ที่ error กลับมาเพราะ panic หรือเพราะถูกยกเลิก

#### `select!`/`timeout` ก็ Cancel ด้วยกลไกเดียวกัน: Drop Future ที่แพ้การแข่ง

กลับไปดูหัวข้อ 48.10 อีกครั้งด้วยมุมมองใหม่ — ตอนที่ `select!` เลือกแขนงที่ชนะ **แขนงที่แพ้ถูก `drop` ทันที**
ซึ่งใช้กลไก Drop เดียวกันกับ `.abort()` เป๊ะ:

```rust
println!("=== select! drop future ที่แพ้การแข่ง — cleanup ทำงานทันที ===");
let losing_future = async {
    let _guard = CleanupGuard("select-losing-branch");
    tokio::time::sleep(Duration::from_millis(500)).await;
    println!("  select-losing-branch: บรรทัดนี้ไม่ควรถูกพิมพ์เลย");
};
tokio::select! {
    _ = losing_future => {
        println!("  แขนงที่แพ้ทำงานจบ (ไม่ควรเกิด)");
    }
    _ = tokio::time::sleep(Duration::from_millis(30)) => {
        println!("  แขนง timeout ชนะ -> losing_future ถูก drop ทันที");
    }
}
```

รันจริงได้:

```
=== select! drop future ที่แพ้การแข่ง — cleanup ทำงานทันที ===
  Drop::drop เรียกทำงาน: cleanup ของ 'select-losing-branch' ทำงานแล้ว
  แขนง timeout ชนะ -> losing_future ถูก drop ทันที
```

เหมือนกันเป๊ะกับกรณี `.abort()`: บรรทัด "บรรทัดนี้ไม่ควรถูกพิมพ์เลย" ไม่ถูกพิมพ์ (future ของ `losing_future`
ถูก drop ก่อนจะไปถึงจุดนั้น) แต่ `CleanupGuard` ของมัน**ยังถูก drop ตามปกติ** — พิสูจน์ว่าไม่ว่า cancellation
จะเกิดผ่านช่องทางไหน (`.abort()` บน task ที่ spawn ไว้, หรือ `select!`/`timeout` บน future เปล่า ๆ ที่ไม่ได้
spawn) กลไกพื้นฐานเหมือนกันเสมอคือ **"drop future ที่ยังทำงานไม่จบ" และ Drop ของทุกอย่างที่มันถืออยู่จะทำงาน
ตามปกติ**

**สรุปตารางเปรียบเทียบให้ชัดเจน**:

| การกระทำ | Task/Future ถูกยกเลิกไหม | `Drop` ของค่าที่ค้างอยู่ทำงานไหม |
|---|---|---|
| Drop `JoinHandle` เฉย ๆ (ไม่เรียก `.abort()`) | **ไม่** — task ทำงานต่อจนจบแบบ detached | ทำงานตามปกติเมื่อ task จบเอง (ไม่เกี่ยวกับ cancellation) |
| เรียก `handle.abort()` | **ใช่** — task ถูกยกเลิกจริง (ที่จุด `.await` ถัดไปที่มันเจอ) | **ใช่** — ทำงานทันทีตอนที่ future ถูก drop |
| `select!`/`timeout` กับแขนงที่แพ้ | **ใช่** — future ของแขนงที่แพ้ถูก drop ทันที | **ใช่** — ทำงานทันทีตอนที่ future ถูก drop |

**ทำไม Rust ออกแบบเรื่องนี้ต่างจาก OS thread ของ Part 37**: การยกเลิก OS thread จากภายนอกโดยไม่ให้ความร่วมมือ
เป็นสิ่งที่ Part 37 ไม่ได้สอน (และ `std::thread` ก็ไม่มี API แบบนั้นให้ใช้เลย) เพราะการฆ่า OS thread กลางทางใน
ระดับ OS เป็นอันตรายมาก — thread อาจถือ lock อยู่ตอนถูกฆ่า ทำให้ resource ค้างตลอดไป (deadlock) โดยไม่มีทาง
รัน cleanup code ได้เลย แต่ future ของ Rust ต่างออกไปโดยพื้นฐาน: มันเป็นแค่ **ค่าข้อมูลธรรมดาที่ implement
`Future`** — การ "หยุด" มันไม่ต้องพึ่ง OS API พิเศษอะไรเลย แค่ **drop มันทิ้งแบบเดียวกับ drop ค่าอะไรก็ได้** และ
เพราะ Rust การันตี `Drop` ทำงานถูกต้องเสมอไม่ว่าค่าจะถูก drop จากที่ไหน (Part 6) การยกเลิก future จึงปลอดภัยโดย
อัตโนมัติ — **นี่คือข้อได้เปรียบที่แท้จริงของ cancellation ใน async Rust เทียบกับ thread**: ยกเลิกได้อย่าง
ปลอดภัยและมี cleanup รับประกัน ในขณะที่ OS thread ทำแบบนี้ไม่ได้เลยในทางปฏิบัติ

### 48.13 ตัวอย่างใหญ่: Concurrent Web Scraper จำลอง

มาประยุกต์ทุกอย่างที่เรียนมาในบทนี้เขียนโปรแกรมจำลองสถานการณ์จริง — "web scraper" ที่ดึงข้อมูลจากหลาย URL
พร้อมกัน (ใช้ `tokio::time::sleep` จำลอง network I/O เพราะเนื้อหา networking จริงด้วย `reqwest`/`tokio::net`
รอ Part 49) แล้วมีขั้นตอน "parse" ที่ใช้ CPU หนักต่อจากผลลัพธ์ที่ได้:

```rust
use std::time::{Duration, Instant};

#[derive(Debug, Clone)]
struct PageResult {
    url: String,
    bytes: usize,
}

async fn simulated_fetch(url: &str, latency_ms: u64, size: usize) -> PageResult {
    // จำลอง network I/O ด้วย sleep — เนื้อหาจริงจะเรียนใน Part 49
    tokio::time::sleep(Duration::from_millis(latency_ms)).await;
    PageResult { url: url.to_string(), bytes: size }
}

fn cpu_heavy_parse(bytes: usize) -> usize {
    // จำลองงาน parse HTML ที่ใช้ CPU หนัก (ไม่ใช่ I/O) ด้วยการคำนวณวนซ้ำจริง
    let mut acc: u64 = 0;
    for i in 0..(bytes as u64 * 20_000) {
        acc = acc.wrapping_add(i.wrapping_mul(2654435761));
    }
    (acc % 1000) as usize
}

const URLS: [(&str, u64, usize); 5] = [
    ("https://example.com/a", 120, 3),
    ("https://example.com/b", 200, 5),
    ("https://example.com/c", 90, 2),
    ("https://example.com/d", 150, 4),
    ("https://example.com/e", 180, 6),
];

#[tokio::main]
async fn main() {
    println!("=== 1. Sequential scraping (.await ทีละ URL) ===");
    let start = Instant::now();
    let mut sequential_results = Vec::new();
    for (url, latency, size) in URLS {
        sequential_results.push(simulated_fetch(url, latency, size).await);
    }
    let sequential_elapsed = start.elapsed();
    println!("sequential: {sequential_elapsed:?} รวม {} หน้า", sequential_results.len());

    println!("\n=== 2. Concurrent scraping ด้วย tokio::spawn + JoinHandle ===");
    let start2 = Instant::now();
    let mut handles = Vec::new();
    for (url, latency, size) in URLS {
        handles.push(tokio::spawn(simulated_fetch(url, latency, size)));
    }
    let mut concurrent_results = Vec::new();
    for h in handles {
        concurrent_results.push(h.await.unwrap());
    }
    let concurrent_elapsed = start2.elapsed();
    println!("concurrent: {concurrent_elapsed:?} รวม {} หน้า", concurrent_results.len());
    println!(
        "อัตราเร็วขึ้น: {:.2}x",
        sequential_elapsed.as_secs_f64() / concurrent_elapsed.as_secs_f64()
    );

    println!("\n=== 3. เพิ่มขั้น parse (CPU-heavy) ด้วย spawn_blocking ===");
    let start3 = Instant::now();
    let mut parse_handles = Vec::new();
    for page in concurrent_results {
        parse_handles.push(tokio::task::spawn_blocking(move || {
            let score = cpu_heavy_parse(page.bytes);
            (page.url, score)
        }));
    }
    let mut parsed = Vec::new();
    for h in parse_handles {
        parsed.push(h.await.unwrap());
    }
    let parse_elapsed = start3.elapsed();
    for (url, score) in &parsed {
        println!("  parse แล้ว: {url} -> score {score}");
    }
    println!("เวลารวมขั้น parse (บน blocking pool, ไม่บล็อก worker thread): {parse_elapsed:?}");
}
```

รันจริง (build แบบ `--release`) ได้:

```
=== 1. Sequential scraping (.await ทีละ URL) ===
sequential: 747.28573ms รวม 5 หน้า

=== 2. Concurrent scraping ด้วย tokio::spawn + JoinHandle ===
concurrent: 201.091756ms รวม 5 หน้า
อัตราเร็วขึ้น: 3.72x

=== 3. เพิ่มขั้น parse (CPU-heavy) ด้วย spawn_blocking ===
  parse แล้ว: https://example.com/a -> score 0
  parse แล้ว: https://example.com/b -> score 0
  parse แล้ว: https://example.com/c -> score 0
  parse แล้ว: https://example.com/d -> score 0
  parse แล้ว: https://example.com/e -> score 384
เวลารวมขั้น parse (บน blocking pool, ไม่บล็อก worker thread): 427.218µs
```

**อ่านผลลัพธ์ทีละส่วน**:

- **Sequential ใช้ 747ms** — ตรงกับผลรวมของ latency ทั้ง 5 URL (120+200+90+150+180 = 740ms บวก overhead
  เล็กน้อย) ตามที่คาดไว้ เพราะแต่ละ `.await` ต้องรอให้จบก่อนไป URL ถัดไป
- **Concurrent ใช้ 201ms** — ใกล้เคียงกับ latency ของ URL ที่ช้าที่สุดเพียงตัวเดียว (`service e` ที่ 180ms บวก
  overhead การ spawn 5 task) คิดเป็น **อัตราเร็วขึ้นจริง 3.72 เท่า** — พิสูจน์อีกครั้งว่าการ spawn task แยกกัน
  ทำให้ทุก URL ถูก "fetch" พร้อมกันจริง ไม่ใช่รอทีละตัว
- **ขั้น parse ใช้ `spawn_blocking`** ทำให้งานคำนวณหนัก (วนลูปนับแสนรอบต่อ URL) ไม่บล็อก worker thread ของ
  runtime หลักเลย ค่า `score` ที่ได้ไม่มีความหมายพิเศษ (เป็นแค่ผลลัพธ์จากการคำนวณจำลอง) — สิ่งที่สำคัญคือ**เวลา
  รวมของขั้นนี้เล็กมาก (427µs)** เพราะทำงานบน blocking thread pool ที่แยกออกไปต่างหาก คู่ขนานกันได้เต็มที่โดย
  ไม่แย่ง worker thread ที่ยังต้องพร้อม poll task async อื่น ๆ ในระบบ

นี่คือ pipeline ที่สมบูรณ์ของโปรแกรม async ระดับ production จริง: **`tokio::spawn` + `JoinHandle`** สำหรับงาน
I/O-bound ที่ทำพร้อมกันได้ (fetch), **`spawn_blocking`** สำหรับงาน CPU-bound ที่ต้องแยกออกจาก worker thread
(parse) — สองเครื่องมือนี้ใช้คู่กันครอบคลุมโจทย์ประเภทงานส่วนใหญ่ที่ระบบ backend จริงต้องเจอ

### 48.14 ส่งท้าย: ทดสอบโค้ด Async ด้วย `#[tokio::test]`

หัวข้อสุดท้ายก่อนปิดบท — เชื่อมกลับไปที่ **Part 32 (การทดสอบด้วย `#[test]`)** เล็กน้อย เพราะโค้ด async ที่เขียน
ในบทนี้ก็ต้องเขียน test ให้ได้เหมือนโค้ดทั่วไป แต่ `#[test]` ธรรมดาใช้กับ `async fn` ไม่ได้เลย (ด้วยเหตุผลเดียว
กับที่ `async fn main()` เปล่า ๆ compile ไม่ผ่านในหัวข้อ 48.3 — ไม่มี executor มา poll มันให้) Tokio จึงมี
attribute macro คู่แฝดของ `#[tokio::main]` ชื่อ **`#[tokio::test]`** ที่ทำหน้าที่เดียวกัน (สร้าง runtime แล้ว
`block_on` ให้) แต่ใช้กับฟังก์ชัน test แทน:

```rust
async fn double(x: i32) -> i32 {
    tokio::time::sleep(std::time::Duration::from_millis(5)).await;
    x * 2
}

#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn test_double() {
        let result = double(21).await;
        assert_eq!(result, 42);
    }
}
```

รันจริงด้วย `cargo test` ได้:

```
running 1 test
test tests::test_double ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

`#[tokio::test]` มาแทนที่ `#[test]` (ไม่ใช่ใช้คู่กัน) และเหมือนกับ `#[tokio::main]` ตรงที่รับ `flavor` argument
ได้เช่นกัน (`#[tokio::test(flavor = "multi_thread")]` เผื่อ test นั้นต้องการ worker thread หลายตัวจริง ๆ) —
ค่าเริ่มต้นของ `#[tokio::test]` คือ `current_thread` (ต่างจาก `#[tokio::main]` ที่ default เป็น `multi_thread`)
เพราะ test ส่วนใหญ่ไม่ต้องการ parallelism ระดับ CPU และการสร้าง runtime แบบเบาที่สุดต่อ test ช่วยให้ test suite
ทั้งชุดรันเร็วขึ้นเมื่อมี test จำนวนมาก

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. ลืม `.await` — Future ไม่ทำอะไรเลยจนกว่าจะถูก Poll

จาก Part 46-47 คุณรู้แล้วว่า `async fn` เมื่อเรียกจะได้ future กลับมาทันที **ไม่ใช่ผลลัพธ์ของงาน** — ถ้าลืมเขียน
`.await` ต่อท้าย จะไม่มีอะไรเกิดขึ้นจริงเลย:

```rust
async fn write_log(msg: &str) {
    println!("  write_log: เขียน log จริง -> {msg}");
}

#[tokio::main]
async fn main() {
    println!("เรียก write_log(...) โดยไม่ await:");
    write_log("เหตุการณ์สำคัญ"); // ตั้งใจไม่ await
    println!("จบ main แล้ว — สังเกตว่าไม่มีบรรทัด 'เขียน log จริง' เลย");
}
```

compiler เตือนแม้จะยังคอมไพล์ผ่าน:

```
warning: unused implementer of `Future` that must be used
 --> src/main.rs:8:5
  |
8 |     write_log("เหตุการณ์สำคัญ");
  |     ^^^^^^^^^^^^^^^^^^^^^^^^
  |
  = note: futures do nothing unless you `.await` or poll them
  = note: `#[warn(unused_must_use)]` (part of `#[warn(unused)]`) on by default
```

รันจริงได้:

```
เรียก write_log(...) โดยไม่ await:
จบ main แล้ว — สังเกตว่าไม่มีบรรทัด 'เขียน log จริง' เลย
```

พิสูจน์ชัดเจนว่า **`write_log(...)` ที่ไม่มี `.await` ไม่เคยรัน body ของฟังก์ชันเลยแม้แต่บรรทัดเดียว** — มันเพียง
สร้าง future ขึ้นมาลอย ๆ แล้วถูก drop ทิ้งทันทีในบรรทัดถัดไป (เหมือนสร้าง `Vec` แล้วไม่เคยใช้ ต่างกันที่ compiler
เตือนหนักกว่าเพราะ `Future` มี `#[must_use]` ติดไว้) **วิธีแก้**: ตรวจสอบทุก warning `unused implementer of
Future that must be used` อย่างเคร่งครัดเสมอ — มันไม่ใช่ warning ที่เพิกเฉยได้ เพราะแปลว่ามีงานที่ตั้งใจให้ทำ
แต่ไม่ได้ถูกทำจริงเลย ต่างจาก warning เรื่อง unused variable ทั่วไปที่ปกติแค่ทำให้โค้ดรก ไม่ได้ทำให้ logic ผิด

### 2. เรียก Blocking Code ตรงในงาน Async (อธิบายเต็มรูปแบบในหัวข้อ 48.11)

สรุปสั้น ๆ ที่นี่: **`std::thread::sleep`, I/O แบบ synchronous ของ `std` (เช่น `std::fs::read`,
`std::net::TcpStream` ธรรมดา), หรือการคำนวณหนักที่ใช้เวลานานหลายสิบ/ร้อยมิลลิวินาทีโดยไม่มี `.await` คั่นเลย**
ล้วนเป็นอันตรายถ้าเรียกตรงในงาน async — มันบล็อก worker thread ทั้งเส้น ลาก task อื่นที่ไม่เกี่ยวข้องให้ค้าง
ไปด้วย เราพิสูจน์ไปแล้วในหัวข้อ 48.11 ว่า task ที่ตั้งใจรอแค่ 10ms กลับใช้เวลาจริง **311.43ms** เพราะถูก
`std::thread::sleep(300ms)` ของ task อื่นบล็อกไว้ — **วิธีแก้**: ใช้เวอร์ชัน async ที่ Tokio มีให้เสมอถ้ามี
(`tokio::time::sleep` แทน `thread::sleep`, `tokio::fs`/`tokio::net` แทนเวอร์ชัน synchronous ของ `std` — Part 49)
หรือถ้าเป็นโค้ด synchronous ที่หลีกเลี่ยงไม่ได้จริง ๆ (เรียก library ภายนอกที่ไม่มีเวอร์ชัน async, คำนวณ CPU
หนักที่ต้องทำ) ให้ห่อด้วย **`tokio::task::spawn_blocking()`** เสมอ

### 3. `tokio::spawn` โดยไม่มี `Send + 'static` — ใช้ `Rc<T>` หรือยืม Reference ข้าม Task

อธิบายละเอียดแล้วในหัวข้อ 48.7 — สรุปสั้น ๆ: ทุก future ที่ส่งเข้า `tokio::spawn` ต้องเป็น `Send + 'static`
ด้วยเหตุผลเดียวกับ `thread::spawn` จาก Part 37/40 — เจอ **E0373** ถ้ายืม local variable โดยไม่ `move`, เจอ
`"future cannot be sent between threads safely"` ถ้า capture ค่าที่ไม่ `Send` เช่น `Rc<T>` **วิธีแก้**: ใส่
`move` เสมอเมื่อ async block/closure ที่ส่งเข้า `spawn` แตะตัวแปร local ใด ๆ และใช้ `Arc<T>` แทน `Rc<T>` เมื่อ
ต้องแชร์ข้อมูลข้าม task (ยังต้องใช้ `Mutex`/`RwLock` ร่วมด้วยถ้าข้อมูลนั้นต้องถูกแก้ไข — Part 50 จะสอน
`tokio::sync::Mutex` เวอร์ชัน async ของมัน)

### 4. เข้าใจผิดว่า Drop `JoinHandle` = Cancel Task ทันที (อธิบายเต็มรูปแบบในหัวข้อ 48.12)

สรุปสั้น ๆ ที่นี่: การ drop `JoinHandle` เฉย ๆ (ไม่เรียก `.abort()`) **ไม่ยกเลิก task** — task จะทำงานต่อไปจนจบ
แบบ detached เราพิสูจน์ไปแล้วว่า task ที่ handle ของมันถูก drop ไปตั้งแต่ก่อนเริ่ม sleep ก็ยังพิมพ์ข้อความ "ทำงาน
จนจบสมบูรณ์" ออกมาจริง ๆ **วิธีแก้/ข้อควรระวัง**: ถ้าต้องการยกเลิก task จริง ๆ ต้องเก็บ `JoinHandle` ไว้แล้ว
เรียก `.abort()` ตรง ๆ เท่านั้น — ถ้าโค้ดของคุณ spawn task ที่ทำงานต่อเนื่องยาว (เช่น background worker ที่วนรับ
งานจาก channel ไม่รู้จบ) แล้วต้องการปิดมันตอนโปรแกรม shutdown ให้เก็บ `JoinHandle` ไว้เสมอและมีจุด `.abort()`
ที่ชัดเจนในโค้ด shutdown ไม่ใช่แค่ปล่อยให้ handle หลุด scope ไปเฉย ๆ แล้วสมมติว่า task จะหยุดตามไปด้วย

### 5. ใช้ `current_thread` Flavor แล้วแปลกใจว่า Task ไม่ได้ทำงานคู่ขนานจริงระดับ CPU

ผู้เขียนโค้ดที่เคยชินกับ `multi_thread` (ค่าเริ่มต้น) มักแปลกใจเมื่อสลับไปใช้
`#[tokio::main(flavor = "current_thread")]` แล้วพบว่างานคำนวณหนักหลาย task ไม่ได้เร็วขึ้นเลยแม้จะ spawn แยกกัน
— เพราะ `current_thread` มี **worker thread เดียวเท่านั้น** (พิสูจน์ในหัวข้อ 48.4 ว่า task 50 ตัวรวมอยู่ที่ 1
thread เดียวเสมอ) ทุก task คำนวณหนักที่ spawn ไปจะแค่ **สลับกันใช้ CPU core เดียว** ไม่มีการทำงานคู่ขนานจริง
เหมือนใน `multi_thread` ที่กระจายไปหลาย core ได้ **วิธีแก้**: ถ้าโปรแกรมมีงานคำนวณหนักที่ต้องการ parallelism
ระดับ CPU จริง ให้ใช้ `multi_thread` (ค่าเริ่มต้น ไม่ต้องระบุ `flavor` เลย หรือระบุ `flavor = "multi_thread"`
ตรง ๆ ก็ได้) หรือถ้าเป็นงาน CPU-bound แท้ ๆ ควรใช้ `spawn_blocking` (หัวข้อ 48.11) หรือพิจารณา crate อย่าง
`rayon` (ที่ Part 37 กล่าวถึงไว้) แทนการพึ่ง Tokio task ตรง ๆ — `current_thread` เหมาะกับงาน **I/O-bound ล้วน ๆ**
ที่ไม่มีการคำนวณหนักปนอยู่เท่านั้น

### 6. เปิด `--features full` ในโค้ด Production โดยไม่ทบทวนว่าใช้ Feature ไหนจริง

อธิบายละเอียดแล้วในหัวข้อ 48.2 — สรุปสั้น ๆ: `full` สะดวกตอนเรียน แต่ดึง dependency ทรานซิทีฟที่ไม่ได้ใช้เข้ามา
โดยไม่จำเป็น เราพิสูจน์ตัวเลขจริงว่าโปรแกรมเดียวกันต้อง lock **24 packages** ด้วย `full` เทียบกับแค่
**7 packages** ด้วยการเลือกเฉพาะ `rt-multi-thread,macros,time` **วิธีแก้**: ก่อน ship โค้ด production ทบทวน
ว่าโปรแกรมใช้ subsystem ไหนของ Tokio จริง (runtime? timer? networking? filesystem? sync primitive?) แล้วเปิด
เฉพาะ feature ที่จำเป็นตามตารางในหัวข้อ 48.2 — ลด compile time และช่วยให้ dependency tree ของโปรเจกต์ชัดเจนขึ้น
ว่าจริง ๆ ต้องพึ่งพาอะไรบ้าง

### 7. เรียก `tokio::spawn`/`tokio::time::sleep` นอก Runtime Context — Panic ทันที

`tokio::spawn`, `tokio::time::sleep`, และฟังก์ชันอื่น ๆ ของ Tokio ส่วนใหญ่ **ไม่ทำงานแบบ standalone ได้เลย** —
พวกมันต้องมี Tokio runtime ที่กำลังทำงานอยู่ ณ ขณะนั้น (เข้าถึงผ่าน thread-local context ภายในที่ runtime ตั้งไว้
ตอนเริ่มทำงาน) ถ้าเรียกฟังก์ชันเหล่านี้จาก `fn main()` ธรรมดาที่ไม่มี `#[tokio::main]` หรือไม่ได้อยู่ใน
`.block_on(...)` เลย จะไม่ใช่ compile error แต่เป็น **runtime panic**:

```rust
fn main() {
    let _handle = tokio::spawn(async {
        println!("นี่จะไม่ทำงาน");
    });
}
```

รันจริงได้:

```
thread 'main' panicked at src/main.rs:2:19:
there is no reactor running, must be called from the context of a Tokio 1.x runtime
```

ข้อความ **"there is no reactor running, must be called from the context of a Tokio 1.x runtime"** บอกตรง ๆ ว่า
ไม่มี runtime context ให้ `tokio::spawn` ใช้งาน — สาเหตุที่พบบ่อยที่สุดคือลืมใส่ `#[tokio::main]` เหนือ `main`,
หรือเรียกฟังก์ชันของ Tokio จาก thread อื่นที่ spawn ด้วย `std::thread::spawn` (ซึ่งไม่มี runtime context ผูกอยู่
เลย แม้ว่า thread นั้นจะถูกสร้างจากภายในโปรแกรมที่มี Tokio runtime ทำงานอยู่ก็ตาม — runtime context ผูกกับ
thread ที่ runtime สร้างขึ้นเท่านั้น ไม่ได้ลามไปยัง OS thread อื่นที่คุณสร้างเองแยกต่างหาก) **วิธีแก้**: ตรวจสอบ
ว่ามี `#[tokio::main]` (หรือ `Runtime::block_on`) ครอบอยู่จริง และถ้าจำเป็นต้องเรียกโค้ด async จากภายใน
`std::thread::spawn` ให้สร้าง `Runtime` แยกและเรียก `.block_on()` เองภายใน thread นั้น หรือใช้
`tokio::runtime::Handle` ที่ clone มาจาก runtime หลักเพื่อ spawn task เข้าไปในนั้นจากภายนอกได้อย่างถูกต้อง

## แบบฝึกหัด (Exercises)

1. **(ง่าย) แปลง Manual Runtime เป็น `#[tokio::main]`**: เขียนโปรแกรมสองเวอร์ชันที่ทำงานเหมือนกันทุกประการ —
   เวอร์ชันแรกใช้ `fn main()` ธรรมดากับ `tokio::runtime::Runtime::new().unwrap().block_on(...)` เวอร์ชันที่สอง
   ใช้ `#[tokio::main]` ทั้งสองเวอร์ชันต้อง spawn หนึ่ง task ที่ print ข้อความ แล้ว `.await` ผลลัพธ์ก่อนจบ
   โปรแกรม ยืนยันว่า output เหมือนกันทุกตัวอักษร
   *(hint: ดูตัวอย่างในหัวข้อ 48.3 เป็นต้นแบบ — ระวังว่าเวอร์ชัน manual ต้องเรียก `tokio::spawn` ข้างใน
   `block_on` เท่านั้น เพราะ `tokio::spawn` ต้องมี runtime context ที่ทำงานอยู่แล้วจึงเรียกได้)*

2. **(กลาง) วัดความเร็วขึ้นของ `tokio::join!` กับ 5 งาน**: เขียนฟังก์ชัน async ที่จำลองการดึงข้อมูลจาก 5
   "แหล่งข้อมูล" ต่างกัน (ใช้ `tokio::time::sleep` กับ latency ต่างกันแต่ละแหล่ง เช่น 80ms, 120ms, 60ms, 150ms,
   100ms) เขียนสองเวอร์ชัน — sequential (`.await` ทีละตัว) กับ concurrent (`tokio::join!` ทั้ง 5 พร้อมกัน) —
   วัดเวลาจริงด้วย `Instant::now()`/`.elapsed()` แล้ว print อัตราเร็วขึ้นที่วัดได้จริง ยืนยันว่าตัวเลขที่ได้
   ใกล้เคียงกับที่คาดไว้ตามทฤษฎี (เวลา concurrent ควรใกล้เคียงกับ latency ที่มากที่สุดในกลุ่ม ไม่ใช่ผลรวม)
   *(hint: `tokio::join!` รับ future ได้หลายตัวไม่จำกัดแค่ 3 — เขียน `tokio::join!(f1, f2, f3, f4, f5)` ได้ตรง ๆ
   คืนค่าเป็น tuple 5 ตัว)*

3. **(ยาก) แก้กับดัก Blocking แล้ววัดผลก่อน-หลัง**: เขียนโปรแกรมที่ spawn 3 task บน `current_thread` flavor
   — task แรกจำลองงาน CPU หนัก (เขียนลูปคำนวณจริงที่ใช้เวลาประมาณ 200ms ไม่ใช่ `sleep`) เขียนแบบ**ผิด**ก่อน
   (เรียกตรงในงาน async โดยไม่ใช้ `spawn_blocking`) วัดว่า task ที่ 2 และ 3 (ที่แค่ `tokio::time::sleep(20ms)`)
   ใช้เวลานานผิดปกติแค่ไหน จากนั้นแก้ด้วย `tokio::task::spawn_blocking()` แล้ววัดใหม่ เขียนสรุปตัวเลขก่อน-หลัง
   เทียบกัน (ต้องเห็นความต่างชัดเจนแบบเดียวกับหัวข้อ 48.11)
   *(hint: ใช้ลูปคำนวณแบบ `wrapping_add`/`wrapping_mul` เหมือนฟังก์ชัน `cpu_heavy_parse` ในหัวข้อ 48.13 ปรับ
   จำนวนรอบให้ได้เวลาประมาณ 200ms บนเครื่องของคุณเอง อาจต้องลองปรับตัวเลขหลายครั้ง)*

4. **(ยากมาก / ประยุกต์ใช้งานจริง) Concurrent Task Pipeline พร้อม Cancellation**: เขียนโปรแกรมจำลอง "batch job
   processor" ที่มีงาน 10 ชิ้น แต่ละชิ้นต้องผ่าน 2 ขั้นตอน: (ก) "ดึงข้อมูล" (จำลองด้วย `tokio::time::sleep`
   ที่ latency สุ่มระหว่าง 50-300ms) และ (ข) "ประมวลผล" (จำลองด้วย CPU-heavy loop ผ่าน `spawn_blocking`) —
   spawn ทั้ง 10 งานให้ทำงานพร้อมกันตั้งแต่ขั้น (ก) เก็บ `JoinHandle` ทั้งหมดไว้ใน `Vec` เพิ่มเงื่อนไขว่าถ้างาน
   ไหน "ดึงข้อมูล" ใช้เวลาเกิน 200ms ให้ยกเลิกงานนั้นด้วย `tokio::time::timeout` (ไม่ต้องไปถึงขั้น (ข)) ส่วนงาน
   ที่ผ่านขั้น (ก) ทันเวลาให้ไปต่อขั้น (ข) ตามปกติ พิมพ์สรุปท้ายโปรแกรมว่ามีกี่งานเสร็จสมบูรณ์ กี่งานถูกยกเลิก
   เพราะ timeout พร้อมใช้ `CleanupGuard` แบบในหัวข้อ 48.12 ยืนยันว่างานที่ถูก timeout ยกเลิกจริงมี cleanup
   ทำงานถูกต้อง
   *(hint: โครงสร้างหลักคือ vec ของ `tokio::spawn(async move { ... })` ที่ข้างในแต่ละ task เรียก
   `tokio::time::timeout(Duration::from_millis(200), simulated_fetch(...)).await` ก่อน ถ้าได้ `Ok` ค่อยเรียก
   `spawn_blocking` ต่อขั้น (ข) ถ้าได้ `Err` (timeout) ให้ return ค่าที่บอกว่างานนี้ถูกยกเลิกแทน)*

## สรุป

บทนี้พาคุณจากจุดที่ Part 47 ปิดท้ายไว้ ("ไม่มีใครเขียน production executor เอง ใช้ Tokio") มาสู่การใช้งาน
**Tokio** จริงอย่างเป็นระบบ — ตลอดบททั้งหมด เราจงใจดึงคู่เทียบจาก Part 37 (`std::thread`) มาเปรียบตลอดเวลา เพื่อ
แสดงให้เห็นว่าแนวคิดของ concurrency ไม่ได้เปลี่ยนไปเลยระหว่างโลก thread กับโลก async — สิ่งที่เปลี่ยนคือ**กลไก
เบื้องหลัง**ที่ทำให้มันมีประสิทธิภาพต่างกันมาก:

- เพิ่ม Tokio ด้วย `cargo add tokio --features full` ตอนเรียน แต่ในโค้ด production ควรเลือก feature เฉพาะที่
  ใช้จริง (พิสูจน์ด้วยตัวเลขจริง 24 packages เทียบกับ 7 packages) ตามหลักการ feature flag จาก Part 17/35
- `#[tokio::main]` เป็นแค่ attribute macro (เชื่อมกับ Part 44) ที่ขยายเป็น
  `Runtime::new().unwrap().block_on(async { ... })` — ไม่มีอะไรเป็นมนตร์ดำ พิสูจน์ด้วยการเขียนทั้งสองเวอร์ชัน
  แล้วเห็นผลลัพธ์เหมือนกันทุกประการ
- Runtime มีสอง flavor หลัก: `multi_thread` (ค่าเริ่มต้น, work-stealing thread pool, เหมาะกับงานที่อาจมี
  CPU-bound ปนอยู่) กับ `current_thread` (thread เดียว, overhead ต่ำกว่า, เหมาะกับงาน I/O-bound ล้วน)
- `tokio::spawn()` สร้าง **task** (ไม่ใช่ OS thread) คืน `JoinHandle<T>` ที่ `.await` แทน `.join()` — ต้องการ
  `Send + 'static` ด้วยเหตุผลเดียวกับ `thread::spawn` เป๊ะ (เห็น E0373 และ "future cannot be sent between
  threads safely" ที่ตรงกับ E0373/E0277 ของ Part 37/40) และเบากว่า OS thread มากจนพิสูจน์ได้ว่า **20,000 task**
  ทำงานเร็วกว่าแค่ **1,000 OS thread** ในโจทย์เดียวกัน
- `tokio::time::sleep` ไม่บล็อก worker thread เหมือน `thread::sleep` — `tokio::join!` ทำงานหลาย future
  พร้อมกันจริง (พิสูจน์ speedup ~3x จากงาน 3 อย่างที่ latency เท่ากัน) และ `tokio::select!`/`timeout`
  ให้ pattern การแข่งกันระหว่าง future/timeout
- **กับดักที่อันตรายที่สุด**: เรียก blocking code ตรงในงาน async ทำลาย worker thread ทั้งเส้น พิสูจน์ด้วยตัวเลข
  จริงว่า task ที่ควรเสร็จใน 10ms กลับใช้เวลา 311ms เพราะถูกงานอื่นบล็อกไว้ — แก้ด้วย `spawn_blocking()` ที่ใช้
  thread pool แยก ลดเวลาลงมาที่ 11ms
- Cancellation มีความเข้าใจผิดที่พบบ่อย: drop `JoinHandle` **ไม่**ยกเลิก task (มันทำงานต่อจนจบแบบ detached) —
  ต้องเรียก `.abort()` ตรง ๆ หรือให้ `select!`/`timeout` drop future ที่แพ้ — ทั้งสองกรณีนี้ `Drop` ของค่าที่ค้าง
  อยู่ทำงานเสมอ (เชื่อมกับ Part 6) ทำให้ cancellation ปลอดภัยกว่าการพยายามฆ่า OS thread จากภายนอกมาก
- ตัวอย่างใหญ่ปิดท้าย (concurrent web scraper จำลอง) รวมทุกเครื่องมือเข้าด้วยกัน: `tokio::spawn` สำหรับงาน
  I/O-bound พร้อมกัน (speedup 3.72x จากการวัดจริง) และ `spawn_blocking` สำหรับขั้นตอนคำนวณหนักที่ตามมา

บทถัดไป **Part 49 (Tokio: I/O และ Networking)** จะเปลี่ยนจาก `tokio::time::sleep` ที่ใช้ "จำลอง" I/O ตลอดทั้ง
บทนี้ ไปสู่ I/O จริง — เปิด `TcpListener`/`TcpStream` รับ-ส่งข้อมูลผ่าน network จริง อ่าน/เขียนไฟล์แบบ async ด้วย
`tokio::fs`, และเจาะลึก I/O reactor ที่กล่าวถึงไว้ในหัวข้อ 48.1 ว่าทำงานอย่างไรจริง ๆ เบื้องหลังการ poll socket
— ทุกอย่างที่เรียนในบทนี้ (`tokio::spawn`, `JoinHandle`, runtime flavor, `spawn_blocking`, cancellation) จะ
เป็นเครื่องมือพื้นฐานที่ Part 49 พึ่งพาต่อเนื่องไปสร้างโปรแกรม network จริงที่รับการเชื่อมต่อจากหลาย client
พร้อมกันได้อย่างมีประสิทธิภาพ

---

**Part ก่อนหน้า:** [Futures และ Executors](part-047-futures-executors.md) | **Part ถัดไป:** [Tokio: I/O และ Networking](part-049-tokio-networking.md)
