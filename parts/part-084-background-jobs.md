# Part 84: Background Jobs และ Task Queues

> โมดูล: การพัฒนาเว็บแอปพลิเคชัน (Web Development) | ระดับ: สูง | เวลาโดยประมาณ: 240 นาที

## เป้าหมายของบทเรียน

- อธิบายได้ว่าทำไมงานบางอย่าง (ส่งอีเมล, สร้างรายงาน, ประมวลผลรูปภาพ) ไม่ควรรันอยู่ใน request/response path โดยตรง และวัดผลกระทบด้าน latency ได้จริงด้วยตัวเลข
- ใช้ `tokio::spawn` แบบ fire-and-forget ได้อย่างถูกต้อง และอธิบายข้อจำกัดของมันได้ชัดเจนว่าทำไมไม่พอสำหรับงานที่สำคัญ
- ออกแบบและสร้าง job queue อย่างง่ายด้วย Redis (`LPUSH`/`BRPOP`) ที่ทำงานได้จริง แยก enqueue (handler) ออกจาก worker (process ที่ประมวลผล) ได้อย่างสมบูรณ์
- Implement retry พร้อม exponential backoff และ dead-letter handling สำหรับ job ที่ล้มเหลว โดยไม่ทำให้งานหายหรือ retry ไม่มีที่สิ้นสุด
- เข้าใจปัญหา idempotency ของ background job ที่เกิดจาก at-least-once delivery และเขียนโค้ดป้องกันผลข้างเคียงซ้ำได้จริง
- สร้าง scheduled/recurring job ด้วย `tokio-cron-scheduler` และรู้ข้อจำกัดของ in-process scheduler เทียบกับ OS-level cron หรือ dedicated scheduler service
- ประกอบทุกส่วนเป็นระบบ booking-confirmation-email แบบ end-to-end ที่ enqueue, worker, retry, idempotency, และ scheduled cleanup ทำงานร่วมกันจริง

## ความรู้ที่ต้องมีมาก่อน

- **Part 46-50 (Async/Await, Futures, Tokio Runtime, Networking, Sync)**: บทนี้ใช้ `tokio::spawn`, `JoinHandle`, `tokio::time::sleep`, และ `tokio::select!`/timeout เป็นพื้นฐานตลอดทั้งบท โดยเฉพาะ **Part 48** ที่อธิบายไว้ว่า drop `JoinHandle` ไม่ได้ยกเลิก task (task จะกลายเป็น "detached" และทำงานต่อจนจบตราบใดที่ runtime ยังไม่ถูก drop) — บทนี้จะเอาความเข้าใจนั้นมาขยายต่อว่า "runtime ถูก drop" หมายถึงอะไรจริง ๆ ในบริบทของ process ที่ปิดหรือ restart
- **Part 57-58 (Serde)**: เราจะ serialize/deserialize job เป็น JSON เพื่อเก็บลง Redis list ด้วย `#[derive(Serialize, Deserialize)]` แบบเดียวกับที่เรียนมา
- **Part 62-66 (Axum)**: บทนี้ใช้ระบบจองตั๋ว (booking system) ต่อจากที่สร้างมาตั้งแต่ Part 63 เป็น domain ตัวอย่าง — สมมติว่ามี handler `POST /bookings` อยู่แล้ว
- **Part 70-71 (SQLx + PostgreSQL)**: ใช้เป็นแหล่งอ้างอิงเมื่อพูดถึงการบันทึกสถานะ "ส่งอีเมลไปแล้วหรือยัง" แบบ persistent ในฐานข้อมูลจริง (ทางเลือกคู่กับ Redis)
- **Part 78 (REST API Design)** หัวข้อ Idempotency-Key: บทนี้นำแนวคิดเดียวกัน (ป้องกันผลข้างเคียงซ้ำจาก request/การประมวลผลที่อาจเกิดซ้ำ) มาประยุกต์ใช้กับ background job แทน HTTP request
- **Part 82 (Message Queues)** และ **Part 83 (Caching ด้วย Redis)**: บทนี้ตัดกับ Part 82 ตรงแนวคิด "at-least-once delivery" และ "dead-letter queue" แต่ใช้ในบริบทต่างกันโดยสิ้นเชิง (อธิบายในหัวข้อ 84.2) และใช้ Redis instance/แนวคิดเดียวกับ Part 83 เป็น backend ของ job queue

## เนื้อหา

### 84.1 ทำไม Background Jobs ถึงจำเป็น: วัด Latency จริงจากระบบจองตั๋ว

ย้อนกลับไปที่ระบบจองตั๋วที่เราสร้างมาตั้งแต่ Part 63: มี handler `POST /bookings` ที่รับข้อมูลการจอง บันทึกลงฐานข้อมูล แล้วตอบ `201 Created` กลับไป ทีนี้ทีมธุรกิจขอเพิ่ม requirement ใหม่: **หลังจากจองสำเร็จ ต้องส่งอีเมลยืนยันให้ลูกค้าทันที**

วิธีที่ตรงไปตรงมาที่สุดคือเรียก API ของผู้ให้บริการอีเมล (เช่น SendGrid, AWS SES, Postmark) จากภายใน handler เดียวกันเลย ก่อนตอบ response กลับไป โค้ดหน้าตาแบบนี้:

```rust
use std::time::Instant;

// จำลอง handler ของ POST /bookings ที่ส่งอีเมลยืนยัน "แบบ synchronous"
// อยู่ในเส้นทางเดียวกับ request/response โดยตรง
async fn call_email_provider(should_fail: bool, latency_ms: u64) -> Result<(), String> {
    tokio::time::sleep(std::time::Duration::from_millis(latency_ms)).await;
    if should_fail {
        Err("email provider timeout: connection reset after 30s".to_string())
    } else {
        Ok(())
    }
}

fn ts() -> String {
    chrono::Local::now().format("%H:%M:%S%.3f").to_string()
}

#[tokio::main]
async fn main() {
    println!("[{}] client ส่ง POST /bookings เข้ามา", ts());
    let start = Instant::now();

    // 1) บันทึก booking ลงฐานข้อมูล (จำลองว่าเร็ว ~5ms)
    tokio::time::sleep(std::time::Duration::from_millis(5)).await;
    println!("[{}] บันทึก booking ลง DB สำเร็จ (5ms)", ts());

    // 2) ส่งอีเมลยืนยัน "รอผลลัพธ์ตรงนี้เลย" ก่อนตอบ response กลับไป
    println!("[{}] เริ่มเรียก email provider (synchronous)...", ts());
    match call_email_provider(false, 1200).await {
        Ok(()) => println!("[{}] ส่งอีเมลสำเร็จ", ts()),
        Err(e) => println!("[{}] ส่งอีเมลล้มเหลว: {e}", ts()),
    }

    let elapsed = start.elapsed();
    println!(
        "[{}] ตอบ response 201 Created กลับ client (รวมเวลาทั้งหมด {:.0}ms)",
        ts(),
        elapsed.as_secs_f64() * 1000.0
    );
}
```

รันจริงแล้วได้ผลลัพธ์นี้ (capture ตรงจากการรัน ไม่ได้แต่ง):

```
[01:58:34.749] client ส่ง POST /bookings เข้ามา
[01:58:34.755] บันทึก booking ลง DB สำเร็จ (5ms)
[01:58:34.755] เริ่มเรียก email provider (synchronous)...
[01:58:35.956] ส่งอีเมลสำเร็จ
[01:58:35.956] ตอบ response 201 Created กลับ client (รวมเวลาทั้งหมด 1207ms)
```

สังเกตตัวเลข: การบันทึกลง DB ใช้เวลาแค่ ~6ms แต่การเรียก email provider (ในตัวอย่างนี้จำลอง latency ที่ 1200ms ซึ่งใกล้เคียงกับ latency จริงของ SMTP/HTTP API ของผู้ให้บริการอีเมลหลายเจ้าเมื่อเจอ cold start หรือ rate limit) ทำให้ request ทั้งก้อนใช้เวลารวม **1207ms** — ลูกค้าต้องรอเกือบ 1.2 วินาทีเพื่อให้ browser ได้ response แค่เพราะ "อีเมล" ซึ่งไม่ใช่ส่วนสำคัญของธุรกรรมการจองตั๋วเลย

ปัญหานี้มีสองมุม:

1. **Latency ที่ผู้ใช้ไม่ควรต้องรับผล**: การจองตั๋วสำเร็จแล้วจริง ๆ ตั้งแต่ DB commit เสร็จ (~6ms) ส่วนที่เหลือ (~1200ms) คือการรออีเมลที่ไม่ได้เปลี่ยนผลลัพธ์ของการจองเลย
2. **Failure domain ที่ผสานกันโดยไม่จำเป็น**: ถ้า email provider ล่มหรือ timeout (ลองเปลี่ยน `should_fail` เป็น `true` แล้วจะเห็นว่าโปรแกรมพิมพ์ "ส่งอีเมลล้มเหลว" แต่ยังตอบ 201 ต่อไปได้เพราะเราไม่ได้ `?` ค่า error — แต่ถ้าโค้ดจริงเขียนแบบ `call_email_provider(...).await?` ใน handler ที่คืน `Result<StatusCode, AppError>` ตามที่ Part 66 สอนไว้ การจองที่**สำเร็จแล้วจริงในฐานข้อมูล**จะถูกตอบเป็น error กลับไปให้ลูกค้า หรือแย่กว่านั้นคือ request timeout ทั้งที่การจองสำเร็จไปแล้ว)

นี่คือประเด็นสำคัญที่สุดของบทนี้: **งานที่ไม่ใช่ส่วนสำคัญของ transaction หลัก (ส่งอีเมล, สร้าง PDF ใบเสร็จ, sync ข้อมูลไปยัง analytics, resize รูปภาพ) ไม่ควรอยู่ใน critical path ของ request** ทั้งในมุม latency และในมุม failure isolation คำตอบคือต้อง **defer** งานเหล่านี้ให้ไปทำงาน "เบื้องหลัง" (background) แทน

### 84.2 Background Jobs ต่างจาก Message Queues (Part 82) อย่างไร

ก่อนไปต่อ ต้องแยกให้ชัดว่าบทนี้ (Background Jobs) กับ Part 82 (Message Queues) พูดถึงปัญหาคนละแบบ แม้เครื่องมือที่ใช้ (Redis, retry, DLQ) จะหน้าตาคล้ายกันมาก:

| มุมมอง | Message Queue (Part 82) | Background Job (บทนี้) |
|---|---|---|
| ขอบเขต | สื่อสาร**ข้ามระบบ/ข้ามบริการ** (เช่น booking service ส่ง event ให้ notification service, inventory service, analytics service) | งานภายใน**แอปพลิเคชันเดียว** ที่แค่ไม่อยากให้บล็อก request/response |
| ผู้ผลิต/ผู้บริโภค | หลาย service ที่ deploy แยกกัน อาจเขียนด้วยภาษาต่างกัน | โค้ดเดียวกัน (หรือ crate เดียวกัน) แค่รันเป็นอีก process/task |
| ตัวอย่าง | "booking.confirmed" event ที่ inventory service, notification service, analytics service ต่าง subscribe ไปทำงานของตัวเอง | "ส่งอีเมลยืนยันหลัง booking นี้" ซึ่งเป็นงานเดียว รู้ผู้รับตายตัว |
| ความซับซ้อนที่ต้องแลก | ต้องมี message schema/contract ระหว่างทีม, versioning, ผู้บริโภคหลายราย | ไม่ต้องมี contract ข้ามทีม แค่ enqueue/dequeue ภายในโค้ดตัวเอง |

พูดง่าย ๆ คือ **message queue คือการสื่อสารระหว่างระบบ** ส่วน **background job คือการเลื่อนงานออกจาก request path ภายในระบบเดียว** ทั้งสองแนวคิดสามารถใช้ Redis เป็น backend ได้เหมือนกัน (และถ้าระบบคุณมี Redis สำหรับ message queue อยู่แล้วจาก Part 82 หรือใช้เป็น cache จาก Part 83 ก็สามารถใช้ instance เดียวกันทำ job queue ได้เลยโดยไม่ต้องตั้ง infrastructure ใหม่) แต่การออกแบบ (จำนวนผู้บริโภค, ความเข้มงวดของ schema, ใครเป็นเจ้าของ) ต่างกันโดยพื้นฐาน บทนี้จะโฟกัสที่กรณีง่ายกว่า: **ผู้ผลิตและผู้บริโภคของ job คือแอปเดียวกัน** เช่นเดียวกับ concept ของ "at-least-once delivery" และ "dead-letter queue" ที่ Part 82 สอนไว้ในบริบทข้ามระบบ — บทนี้จะนำแนวคิดเดียวกันมาใช้กับ job ภายในแอปเดียว ซึ่งเรียบง่ายกว่ามากเพราะไม่ต้องกังวลเรื่อง schema evolution ข้ามทีมหรือ consumer group หลายฝ่าย

### 84.3 วิธีที่ง่ายที่สุด: `tokio::spawn` แบบ Fire-and-Forget

วิธีแรกที่นึกถึงได้ทันทีคือใช้ `tokio::spawn` ที่เรียนมาตั้งแต่ Part 48 — spawn task ส่งอีเมลไว้ แล้วตอบ response กลับไปโดยไม่รอ:

```rust
use std::time::Instant;

fn ts() -> String {
    chrono::Local::now().format("%H:%M:%S%.3f").to_string()
}

// จำลอง handler ที่ตอบ response กลับไปก่อน แล้วค่อย tokio::spawn งานส่งอีเมล
// "ทิ้งไว้" โดยไม่เก็บ JoinHandle (fire-and-forget)
#[tokio::main]
async fn main() {
    let start = Instant::now();
    println!("[{}] client ส่ง POST /bookings เข้ามา", ts());
    tokio::time::sleep(std::time::Duration::from_millis(5)).await;
    println!("[{}] บันทึก booking ลง DB สำเร็จ (5ms)", ts());

    // spawn แล้วไม่เก็บ handle เลย -> งานนี้ "detached" ทันที (อ้างอิง Part 48)
    tokio::spawn(async {
        println!("[{}] (background task) เริ่มส่งอีเมลยืนยัน...", ts());
        tokio::time::sleep(std::time::Duration::from_millis(1200)).await;
        println!("[{}] (background task) ส่งอีเมลสำเร็จ!", ts());
    });

    println!(
        "[{}] ตอบ response 201 Created กลับ client (รวมเวลา {:.0}ms) — ไม่รอ task ส่งอีเมล",
        ts(),
        start.elapsed().as_secs_f64() * 1000.0
    );

    // main() จบตรงนี้ -> #[tokio::main] จะ drop runtime ทันทีหลัง main คืนค่า
}
```

ผลลัพธ์จริงจากการรัน:

```
[01:58:41.283] client ส่ง POST /bookings เข้ามา
[01:58:41.290] บันทึก booking ลง DB สำเร็จ (5ms)
[01:58:41.290] ตอบ response 201 Created กลับ client (รวมเวลา 6ms) — ไม่รอ task ส่งอีเมล
```

สังเกตให้ดี ๆ: **บรรทัด "(background task) เริ่มส่งอีเมลยืนยัน..." และ "(background task) ส่งอีเมลสำเร็จ!" ไม่ถูกพิมพ์ออกมาเลยแม้แต่บรรทัดแรก!** เกิดอะไรขึ้น?

จาก **Part 48** เราเรียนมาแล้วว่า drop `JoinHandle` (แบบที่เกิดขึ้นเมื่อไม่เก็บค่าที่ `tokio::spawn` คืนมา) **ไม่ได้ยกเลิก task** — task ที่ spawn ไว้จะกลายเป็น "detached" และทำงานต่อไปเรื่อย ๆ อย่างเป็นอิสระ ตราบใดที่ **runtime ยังไม่ถูก drop** นี่คือกุญแจสำคัญที่ต้องเพิ่มเข้าไปในความเข้าใจ: `#[tokio::main]` สร้าง `Runtime` ขึ้นมาแล้ว `block_on(main_body)` เมื่อ future ของ `main()` (ในที่นี้คือทุกอย่างในฟังก์ชัน `main` ของเรา) `.await` จบ ตัว macro จะ **drop `Runtime` นั้นทันที** และการ drop `Runtime` จะทำให้ task ที่ยังทำงานอยู่ทั้งหมดถูกยกเลิกไปด้วย ไม่ว่าจะ detached หรือไม่ก็ตาม — เพราะไม่มี runtime เหลือให้ poll มันต่อแล้ว ในตัวอย่างนี้ task ที่ spawn ไว้ยังไม่ทันได้ถูก scheduler หยิบไป poll รอบแรกเลยด้วยซ้ำ (เพราะ `main` เขียน `println!` บรรทัดสุดท้ายเสร็จเร็วกว่าที่ scheduler จะสลับไปที่ task ใหม่) จึงไม่มีข้อความใดจาก background task ปรากฏออกมาเลย

นี่คือข้อจำกัดที่ร้ายแรงที่สุดของ `tokio::spawn` แบบ fire-and-forget เมื่อใช้ในบริบทของ web server:

1. **ไม่มี persistence**: ถ้า process ของเว็บเซิร์ฟเวอร์ปิดหรือ restart (deploy ใหม่, crash, container ถูก kill โดย orchestrator) งานที่ spawn ไว้แต่ยังไม่เสร็จจะหายไปทันที ไม่มีทางกู้กลับมาได้ ต่างจากตัวอย่างข้างบนที่ทั้งโปรแกรมปิดตัวเองปกติ — ในเว็บเซิร์ฟเวอร์จริง แม้ process จะยังรันต่อไปเรื่อย ๆ (ไม่ปิดทันทีแบบ `main()` ในตัวอย่าง) แต่ **ทุกครั้งที่ deploy ใหม่หรือ restart** งานเบื้องหลังที่ค้างอยู่ก็จะหายไปเหมือนกัน
2. **ไม่มี retry**: ถ้า `call_email_provider` ล้มเหลว (provider timeout) เราไม่มีกลไกให้ลองใหม่เลย — job นั้นก็หายไปเงียบ ๆ ไม่มีใครรู้
3. **ไม่มี visibility/tracking**: ไม่มีทางรู้เลยว่ามีงานค้างอยู่กี่งาน งานไหนสำเร็จ งานไหนล้มเหลว ถ้าลูกค้าโทรมาถามว่า "ทำไมไม่ได้รับอีเมลยืนยัน" ทีม support จะไม่มีข้อมูลอะไรเลยที่จะตรวจสอบได้

สำหรับงานที่ "ถ้าหายไปก็ไม่เป็นไรจริง ๆ" (เช่น log analytics event ที่ sample ไว้แค่ 1%) `tokio::spawn` แบบนี้ก็ยังใช้ได้ดีเพราะเรียบง่ายและเร็ว แต่สำหรับงานที่มีความสำคัญทางธุรกิจอย่างการส่งอีเมลยืนยันการจอง เราต้องการระบบที่:

- เก็บ job ไว้ใน storage ที่อยู่ทน (persistent) ไม่หายไปพร้อม process
- มี retry logic ที่ควบคุมได้
- ตรวจสอบสถานะได้ว่างานไหนทำสำเร็จ/ล้มเหลว

นี่คือที่มาของ **job queue** จริง ๆ

### 84.4 เลือกเครื่องมือ: `apalis` หรือสร้าง Queue เองด้วย Redis

ในโลก Rust มี crate ชื่อ **`apalis`** ที่เป็น background-job framework โดยเฉพาะ รองรับ backend หลายแบบทั้ง Redis, PostgreSQL (ผ่าน `apalis-sql`), SQLite และ in-memory มี abstraction สำหรับ job (`Job` trait), worker pool, retry policy, และ storage backend ให้พร้อม ถ้าโปรเจกต์มีความซับซ้อนสูง (หลาย job type, ต้องการ concurrency control ต่อ worker, ต้องการ UI dashboard) `apalis` เป็นตัวเลือกที่ดีมาก เพราะไม่ต้องเขียน retry/backoff/serialization เองทั้งหมด และ backend Postgres ของมันต่อยอดจากโครงสร้างที่ Part 70-71 สอนไว้ได้ทันที (ใช้ตาราง job แทน list ใน Redis)

อย่างไรก็ตาม บทนี้เลือก **สร้าง job queue เองด้วย Redis (`LPUSH`/`BRPOP`)** เป็นเครื่องมือหลักในการสอน ด้วยเหตุผลดังนี้:

1. **โปร่งใสสมบูรณ์**: `LPUSH`/`BRPOP` เป็น primitive ของ Redis ที่เข้าใจง่ายมาก — `LPUSH` ดันค่าเข้าหัว list, `BRPOP` ดึงค่าจากท้าย list แบบบล็อกรอถ้า list ว่าง (First-In-First-Out พอดี) ไม่มี abstraction ซ่อนอยู่ ทำให้เห็นตรง ๆ ว่า "job queue" ที่จริงแล้วก็คือ data structure ง่าย ๆ ตัวหนึ่งที่ครอบด้วย logic การ retry/tracking ที่เราเขียนเอง ซึ่งเหมาะกับเป้าหมายการสอนของบทนี้มากกว่าอาศัย framework ที่ซ่อน mechanism ไว้
2. **ต่อยอดจาก Part 83 ได้ทันที**: ถ้าระบบมี Redis อยู่แล้วสำหรับ caching (Part 83) ก็ใช้ instance เดียวกันเป็น job queue ได้เลยโดยไม่ต้องเพิ่ม infrastructure ใหม่ (แต่ในระบบจริงที่มี load สูง ควรแยก Redis instance ของ cache กับ queue ออกจากกัน เพราะพฤติกรรมการใช้หน่วยความจำและ eviction policy ต่างกันมาก — cache ต้องการ eviction แบบ LRU ได้ ส่วน queue ห้าม evict ข้อมูลทิ้งเด็ดขาด)
3. **ไม่ผูกกับ API ของ framework ที่เปลี่ยนเวอร์ชันบ่อย**: `LPUSH`/`BRPOP` เป็น command ของ Redis ที่คงที่มาหลายสิบปี ไม่มีความเสี่ยงเรื่อง breaking change ของ framework

โครงสร้างพื้นฐานของ Redis-based queue ที่เราจะสร้าง:

- **Enqueue**: serialize job เป็น JSON ด้วย `serde_json` แล้ว `LPUSH` เข้า Redis list ที่ชื่อ `jobs:booking_confirmation`
- **Dequeue**: worker เรียก `BRPOP jobs:booking_confirmation <timeout>` ซึ่งจะบล็อกรอจนกว่าจะมีงานเข้ามา (หรือ timeout) แล้วดึงงานจาก**ท้าย** list ออกมา (เพราะ `LPUSH` ดันเข้า**หัว** list ผลคือ FIFO — งานที่เข้าก่อนจะถูกประมวลผลก่อน)
- **Retry**: ถ้าประมวลผลล้มเหลว worker จะลองใหม่ในลูปเดิม (in-process retry) พร้อม backoff โดยไม่ต้อง `LPUSH` กลับเข้า queue จนกว่าจะครบจำนวนครั้งที่กำหนด
- **Dead-letter**: ถ้า retry ครบจำนวนแล้วยังไม่สำเร็จ ให้ `LPUSH` เข้า list อีกตัวชื่อ `jobs:booking_confirmation:failed` เพื่อรอการตรวจสอบด้วยมือ

Dependencies ที่ใช้ตลอดบทนี้ (เวอร์ชันที่ทดสอบจริงแล้วว่า compile และรันได้กับ Rust edition 2021):

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
redis = { version = "0.27", features = ["tokio-comp"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
chrono = { version = "0.4", features = ["clock"] }
uuid = { version = "1", features = ["v4", "serde"] }
tokio-cron-scheduler = "0.13"
```

### 84.5 นิยาม Job และ Enqueue จาก Handler

เริ่มจากนิยาม struct ของ job ที่เป็น `Serialize`/`Deserialize` ตามที่เรียนมาจาก Part 57:

```rust
use chrono::Local;
use redis::AsyncCommands;
use serde::{Deserialize, Serialize};

pub const REDIS_URL: &str = "redis://127.0.0.1:6379/";
pub const QUEUE_KEY: &str = "jobs:booking_confirmation";
pub const FAILED_KEY: &str = "jobs:booking_confirmation:failed";

pub fn ts() -> String {
    Local::now().format("%H:%M:%S%.3f").to_string()
}

// payload ของ job — ข้อมูลที่ worker ต้องใช้ทำงานจริง
#[derive(Debug, Serialize, Deserialize, Clone)]
pub struct SendBookingConfirmationEmail {
    pub booking_id: String,
    pub email: String,
}

// envelope ที่ครอบ payload พร้อม metadata สำหรับการ retry/tracking
#[derive(Debug, Serialize, Deserialize, Clone)]
pub struct EnqueuedJob {
    pub id: String,
    pub job_type: String,
    pub payload: SendBookingConfirmationEmail,
    pub attempts: u32,
    pub max_attempts: u32,
    pub enqueued_at: String,
}

pub async fn get_conn() -> redis::aio::MultiplexedConnection {
    let client = redis::Client::open(REDIS_URL).expect("invalid redis url");
    client
        .get_multiplexed_async_connection()
        .await
        .expect("cannot connect to redis")
}

pub async fn enqueue(conn: &mut redis::aio::MultiplexedConnection, job: &EnqueuedJob) {
    let payload = serde_json::to_string(job).expect("serialize job");
    let _: i64 = conn.lpush(QUEUE_KEY, payload).await.expect("LPUSH failed");
}

/// จำลองการเรียก email provider API จริง ๆ ที่มี latency และอาจ fail ได้
pub async fn call_email_provider(should_fail: bool, latency_ms: u64) -> Result<(), String> {
    tokio::time::sleep(std::time::Duration::from_millis(latency_ms)).await;
    if should_fail {
        Err("email provider timeout: connection reset after 30s".to_string())
    } else {
        Ok(())
    }
}
```

สังเกตว่า `EnqueuedJob` มี field `attempts` และ `max_attempts` ติดตัวมาด้วย — นี่คือสิ่งที่ต่างจากการ spawn task แบบเดิม: **job ถือ state ของตัวเองไว้ได้** ทำให้ worker รู้ว่าควร retry ได้อีกกี่ครั้ง เราแยก `job_type` เป็น `String` เผื่ออนาคตมี job หลายประเภทอยู่ใน queue เดียวกัน (ในระบบจริงมักจะแยก queue ต่อ job type หรือใช้ enum ครอบ payload หลายแบบ แต่บทนี้เจาะเฉพาะ job ประเภทเดียวเพื่อความชัดเจน)

ต่อไปคือ handler ที่ enqueue job แทนการส่งอีเมลตรง ๆ:

```rust
use uuid::Uuid;
// สมมติว่า mod ด้านบน (SendBookingConfirmationEmail, EnqueuedJob, enqueue, get_conn, ts) มาจาก lib.rs

#[tokio::main]
async fn main() {
    let start = std::time::Instant::now();
    println!("[{}] client ส่ง POST /bookings เข้ามา", ts());

    tokio::time::sleep(std::time::Duration::from_millis(5)).await;
    let booking_id = "BK-9001".to_string();
    println!("[{}] บันทึก booking {booking_id} ลง DB สำเร็จ", ts());

    let job = EnqueuedJob {
        id: Uuid::new_v4().to_string(),
        job_type: "send_booking_confirmation_email".to_string(),
        payload: SendBookingConfirmationEmail {
            booking_id: booking_id.clone(),
            email: "customer@example.com".to_string(),
        },
        attempts: 0,
        max_attempts: 3,
        enqueued_at: ts(),
    };

    let mut conn = get_conn().await;
    enqueue(&mut conn, &job).await;
    println!(
        "[{}] enqueue job {} เข้า Redis list '{}' สำเร็จ (LPUSH)",
        ts(), job.id, QUEUE_KEY
    );

    println!(
        "[{}] ตอบ response 201 Created กลับ client (รวมเวลา {:.1}ms) — ยังไม่ได้ส่งอีเมลจริงเลย",
        ts(),
        start.elapsed().as_secs_f64() * 1000.0
    );
}
```

รันจริง (มี Redis รันอยู่ที่ `127.0.0.1` พอร์ตที่กำหนดใน `REDIS_URL`) ได้ผลลัพธ์:

```
[01:58:53.272] client ส่ง POST /bookings เข้ามา
[01:58:53.278] บันทึก booking BK-9001 ลง DB สำเร็จ
[01:58:53.280] enqueue job 74ef2479-ab5d-4ff9-b5e3-20b8e2bd26de เข้า Redis list 'jobs:booking_confirmation' สำเร็จ (LPUSH)
[01:58:53.280] ตอบ response 201 Created กลับ client (รวมเวลา 7.5ms) — ยังไม่ได้ส่งอีเมลจริงเลย
```

เทียบกับ 1207ms ในหัวข้อ 84.1 ตอนนี้ handler ตอบกลับใน **7.5ms** — เร็วขึ้นกว่า 160 เท่า และไม่ว่า email provider จะล่มหรือช้าแค่ไหน ก็ไม่มีทางกระทบ latency ของ request นี้อีกต่อไป เพราะ handler ทำแค่ `LPUSH` ข้อมูลเข้า Redis เท่านั้น (ซึ่งเร็วมากเพราะเป็น in-memory operation) ไม่ได้เรียก email provider เลยด้วยซ้ำ

### 84.6 Worker Process: ดึงงานด้วย `BRPOP`

ทีนี้ต้องมีอีกฝั่งที่ดึงงานจาก queue ไปประมวลผลจริง นั่นคือ **worker** — โดยหลักการแล้วควรเป็น **process แยกต่างหาก** จาก web server (รันด้วย `cargo run --bin worker` เป็นอีก binary หรือ deploy เป็นอีก container/pod ก็ได้) เพื่อให้การประมวลผล job ไม่แย่ง CPU/memory กับการรับ HTTP request และเพื่อให้ scale แต่ละส่วนแยกกันได้ (เช่น ถ้ามี booking เข้ามาถี่มากในบางช่วงเวลา ก็เพิ่มจำนวน worker process ได้โดยไม่ต้องแตะ web server เลย)

```rust
use redis::AsyncCommands;
// สมมติว่า EnqueuedJob, get_conn, call_email_provider, ts, QUEUE_KEY มาจาก lib.rs

#[tokio::main]
async fn main() {
    println!("[{}] worker เริ่มทำงาน รอ job จาก '{}' (BRPOP)", ts(), QUEUE_KEY);
    let mut conn = get_conn().await;

    // BRPOP บล็อกรอสูงสุด 10 วินาที ถ้าไม่มีงานเข้ามาก็ปล่อยให้ worker จบ (สำหรับสาธิตในบทนี้ —
    // ในระบบจริง worker จะวนลูป BRPOP ไปเรื่อย ๆ ไม่มีที่สิ้นสุด)
    let result: Option<(String, String)> = conn.brpop(QUEUE_KEY, 10.0).await.expect("BRPOP failed");

    match result {
        Some((_key, raw_job)) => {
            let job: EnqueuedJob = serde_json::from_str(&raw_job).expect("deserialize job failed");
            println!(
                "[{}] worker ดึงงาน job_id={} booking_id={} ออกมาจากคิวได้ (นี่คือคนละ process จาก handler ที่ enqueue!)",
                ts(), job.id, job.payload.booking_id
            );

            println!("[{}] worker เริ่มเรียก email provider จริง...", ts());
            match call_email_provider(false, 800).await {
                Ok(()) => println!(
                    "[{}] ส่งอีเมลยืนยันไปที่ {} สำเร็จ (booking_id={})",
                    ts(), job.payload.email, job.payload.booking_id
                ),
                Err(e) => println!("[{}] ส่งอีเมลล้มเหลว: {e}", ts()),
            }
        }
        None => println!("[{}] ไม่มีงานเข้ามาภายใน timeout", ts()),
    }
}
```

เพื่อพิสูจน์ว่า handler กับ worker เป็นคนละ process ที่แยกกันจริง ๆ ให้รัน `enqueue` แล้วปล่อยให้จบไปเลย (process ของมันตายไปแล้วสมบูรณ์) จากนั้นรอ 4 วินาที ค่อยรัน `worker` เป็น process ใหม่มาดึงงาน:

```bash
$ ./enqueue
$ sleep 4
$ ./worker
```

ผลลัพธ์จริงจากการรันตามลำดับนี้:

```
=== รัน enqueue (handler) ===
[01:58:53.272] client ส่ง POST /bookings เข้ามา
[01:58:53.278] บันทึก booking BK-9001 ลง DB สำเร็จ
[01:58:53.280] enqueue job 74ef2479-ab5d-4ff9-b5e3-20b8e2bd26de เข้า Redis list 'jobs:booking_confirmation' สำเร็จ (LPUSH)
[01:58:53.280] ตอบ response 201 Created กลับ client (รวมเวลา 7.5ms) — ยังไม่ได้ส่งอีเมลจริงเลย

=== รอ 4 วินาที (จำลองว่า handler process นี้จบไปแล้ว, client ได้ response ไปแล้ว) ===

=== รัน worker (คนละ process) มาดึงงานไปทำ ===
[01:58:57.284] worker เริ่มทำงาน รอ job จาก 'jobs:booking_confirmation' (BRPOP)
[01:58:57.286] worker ดึงงาน job_id=74ef2479-ab5d-4ff9-b5e3-20b8e2bd26de booking_id=BK-9001 ออกมาจากคิวได้ (นี่คือคนละ process จาก handler ที่ enqueue!)
[01:58:57.286] worker เริ่มเรียก email provider จริง...
[01:58:58.088] ส่งอีเมลยืนยันไปที่ customer@example.com สำเร็จ (booking_id=BK-9001)
```

ดู timestamp ให้ดี: handler จบงานไปตั้งแต่ `01:58:53.280` — process ของมันปิดตัวลงสมบูรณ์แล้ว ไม่มี memory หรือ task ใด ๆ ค้างอยู่ในนั้นเลย แล้ว **4 วินาทีต่อมา** worker (binary คนละตัว, process คนละตัว, เริ่มที่ `01:58:57.284`) มาดึง job ตัวเดียวกัน (`job_id=74ef2479-...`) ออกจาก Redis ได้ และประมวลผลจนสำเร็จที่ `01:58:58.088` — นี่คือหลักฐานที่ชัดเจนว่า **job ถูกเก็บไว้ใน Redis อย่างทนทาน (persistent) ไม่ได้ผูกติดกับ process หรือ memory ของ handler เลย** ต่างจาก `tokio::spawn` ในหัวข้อ 84.3 อย่างสิ้นเชิง ที่ต่อให้ process เดิมยังไม่ทันปิดตัว งานก็หายไปได้ตั้งแต่ runtime ถูก drop

### 84.7 Retry พร้อม Exponential Backoff

Email provider ในโลกจริงไม่ได้ทำงานสำเร็จ 100% เสมอไป — บางครั้ง timeout, บางครั้ง rate limit ชั่วคราว การ retry ทันทีซ้ำ ๆ ติดกันอาจทำให้ปัญหาแย่ลง (ยิ่ง provider โอเวอร์โหลดหนักขึ้น) จึงต้องเว้นระยะเวลาให้นานขึ้นในแต่ละครั้งที่ retry ซึ่งเรียกว่า **exponential backoff**

```rust
// จำลอง worker ที่ประมวลผล job เดียว ซึ่ง email provider "timeout" 2 ครั้งแรก
// แล้วสำเร็จในครั้งที่ 3 -- ใช้ exponential backoff ระหว่างการ retry แต่ละครั้ง
#[tokio::main]
async fn main() {
    let max_attempts = 4u32;
    let job_id = "job-7c3a";
    let booking_id = "BK-9002";

    println!("[{}] worker ดึงงาน job_id={job_id} booking_id={booking_id} ออกจากคิว", ts());

    let mut attempt = 0u32;
    loop {
        attempt += 1;
        println!("[{}] พยายามส่งอีเมล ครั้งที่ {attempt}/{max_attempts} ...", ts());

        // จำลองว่า provider timeout ในสองครั้งแรก แล้วสำเร็จตั้งแต่ครั้งที่ 3
        let should_fail = attempt < 3;
        match call_email_provider(should_fail, 300).await {
            Ok(()) => {
                println!(
                    "[{}] ส่งอีเมลสำเร็จในการพยายามครั้งที่ {attempt} (booking_id={booking_id})",
                    ts()
                );
                break;
            }
            Err(e) => {
                if attempt >= max_attempts {
                    println!(
                        "[{}] ครบ {max_attempts} ครั้งแล้วยังไม่สำเร็จ ({e}) -> ส่งเข้า dead-letter",
                        ts()
                    );
                    break;
                }
                // exponential backoff: 300ms, 600ms, 1200ms, ...
                let backoff_ms = 300u64 * 2u64.pow(attempt - 1);
                println!("[{}] ล้มเหลว: {e} -> รอ backoff {backoff_ms}ms ก่อน retry ครั้งถัดไป", ts());
                tokio::time::sleep(std::time::Duration::from_millis(backoff_ms)).await;
            }
        }
    }
}
```

ผลลัพธ์จริงจากการรัน:

```
[01:59:04.942] worker ดึงงาน job_id=job-7c3a booking_id=BK-9002 ออกจากคิว
[01:59:04.942] พยายามส่งอีเมล ครั้งที่ 1/4 ...
[01:59:05.244] ล้มเหลว: email provider timeout: connection reset after 30s -> รอ backoff 300ms ก่อน retry ครั้งถัดไป
[01:59:05.545] พยายามส่งอีเมล ครั้งที่ 2/4 ...
[01:59:05.846] ล้มเหลว: email provider timeout: connection reset after 30s -> รอ backoff 600ms ก่อน retry ครั้งถัดไป
[01:59:06.448] พยายามส่งอีเมล ครั้งที่ 3/4 ...
[01:59:06.748] ส่งอีเมลสำเร็จในการพยายามครั้งที่ 3 (booking_id=BK-9002)
```

ดู interval ระหว่างแต่ละครั้งที่ล้มเหลว: ครั้งที่ 1 ล้มเหลวที่ `05.244` แล้วรอ **300ms** พอดี ก่อนพยายามครั้งที่ 2 ที่ `05.545` (`05.545 - 05.244 = 301ms` ตรงตามที่คำนวณไว้ บวก overhead เล็กน้อยจาก scheduler) จากนั้นล้มเหลวอีกที่ `05.846` แล้วรอ **600ms** ก่อนพยายามครั้งที่ 3 ที่ `06.448` (`06.448 - 05.846 = 602ms`) — สูตร `base_ms * 2^(attempt-1)` ทำให้ระยะเวลารอเพิ่มเป็นสองเท่าทุกครั้งที่ retry (300ms → 600ms → 1200ms → ...) ซึ่งช่วยลดโอกาสที่ worker หลายตัวจะยิง request ไปที่ provider พร้อมกันถี่เกินไปตอนที่ provider กำลังมีปัญหาอยู่แล้ว (ในระบบจริงระดับ production มักจะเพิ่ม **jitter** แบบสุ่มเข้าไปในค่า backoff ด้วย เพื่อป้องกันปัญหา "thundering herd" ที่ worker หลายตัว retry พร้อมกันเป๊ะ ๆ ถ้าล้มเหลวพร้อมกัน แต่บทนี้ขอเน้นแค่หลักการ exponential backoff แบบพื้นฐานก่อน)

สังเกตว่าโค้ดนี้ **ไม่ได้เอา job ออกจากคิวแล้วใส่กลับเข้าไปใหม่ทุกครั้งที่ retry** — เป็นการ retry แบบ in-process loop ภายใน worker ตัวเดียวกัน ข้อดีคือเรียบง่ายและเห็น backoff ตรงไปตรงมา ข้อเสียคือ worker ตัวนั้นจะถูก "จอง" ไว้กับ job นี้ตลอดช่วง backoff ทำให้ไม่ไปหยิบ job อื่นมาทำระหว่างนั้น ถ้าต้องรองรับ throughput สูงมาก การออกแบบที่ดีกว่าคือใช้ Redis sorted set (`ZADD`) เก็บ job ที่รอ retry พร้อม timestamp ที่ "พร้อมจะ retry" เป็น score แล้วมี process แยกที่คอยเช็คและย้ายกลับเข้า `LPUSH` เมื่อถึงเวลา (คล้ายกับ delayed queue) วิธีนี้ทำให้ worker ตัวอื่นยังหยิบ job ใหม่ ๆ ไปทำได้ระหว่างที่ job หนึ่งกำลังรอ backoff อยู่ — เป็นการต่อยอดที่ผู้อ่านที่สนใจงานปริมาณสูงสามารถทำเพิ่มได้ แต่นอกเหนือขอบเขตของบทนี้

### 84.8 Dead-Letter: เมื่อ Retry ครบแล้วยังไม่สำเร็จ

ถ้า retry ไปเรื่อย ๆ ไม่มีที่สิ้นสุดสำหรับ job ที่ล้มเหลวอย่างถาวร (เช่น อีเมลผิด, provider ปิดบริการถาวร) จะทำให้ worker ติดอยู่กับ job นั้นตลอดไปและ resource รั่วไหล คำตอบคือกำหนด `max_attempts` และเมื่อครบจำนวนแล้วยังไม่สำเร็จ ให้ย้าย job นั้นไปเก็บไว้ใน **dead-letter list** แยกต่างหาก เพื่อให้ทีมตรวจสอบด้วยมือทีหลัง (แนวคิดเดียวกับ dead-letter queue ของ Part 82 แต่ใช้กับ job ในแอปเดียว ไม่ใช่ message ข้ามระบบ):

```rust
use redis::AsyncCommands;
use serde_json::json;

#[tokio::main]
async fn main() {
    let job = EnqueuedJob {
        id: "job-dead-001".to_string(),
        job_type: "send_booking_confirmation_email".to_string(),
        payload: SendBookingConfirmationEmail {
            booking_id: "BK-9003".to_string(),
            email: "unreachable@example.com".to_string(),
        },
        attempts: 0,
        max_attempts: 3,
        enqueued_at: ts(),
    };

    println!(
        "[{}] worker ดึงงาน job_id={} booking_id={} ออกจากคิว",
        ts(), job.id, job.payload.booking_id
    );

    let mut attempt = 0u32;
    let mut last_error = String::new();
    loop {
        attempt += 1;
        println!("[{}] พยายามส่งอีเมล ครั้งที่ {attempt}/{} ...", ts(), job.max_attempts);
        match call_email_provider(true, 200).await {
            Ok(()) => {
                println!("[{}] ส่งอีเมลสำเร็จ (ไม่ควรเกิดในตัวอย่างนี้)", ts());
                return;
            }
            Err(e) => {
                last_error = e.clone();
                if attempt >= job.max_attempts {
                    println!("[{}] ล้มเหลว: {e} -> ครบ {} ครั้งแล้ว หยุด retry", ts(), job.max_attempts);
                    break;
                }
                let backoff_ms = 200u64 * 2u64.pow(attempt - 1);
                println!("[{}] ล้มเหลว: {e} -> รอ backoff {backoff_ms}ms", ts());
                tokio::time::sleep(std::time::Duration::from_millis(backoff_ms)).await;
            }
        }
    }

    // ย้ายเข้า dead-letter list พร้อม metadata สำหรับตรวจสอบด้วยมือ
    let mut conn = get_conn().await;
    let dead_letter_entry = json!({
        "job": job,
        "final_error": last_error,
        "attempts_made": attempt,
        "failed_at": ts(),
    });
    let _: i64 = conn
        .lpush(FAILED_KEY, dead_letter_entry.to_string())
        .await
        .expect("LPUSH to dead-letter failed");
    println!(
        "[{}] ย้าย job_id={} เข้า dead-letter list '{}' แล้ว (รอทีมตรวจสอบด้วยมือ)",
        ts(), job.id, FAILED_KEY
    );

    let stored: Vec<String> = conn.lrange(FAILED_KEY, 0, -1).await.expect("LRANGE failed");
    println!("[{}] ตรวจสอบ dead-letter list ตอนนี้มี {} รายการ", ts(), stored.len());
}
```

ผลลัพธ์จริงจากการรัน:

```
[01:59:06.757] worker ดึงงาน job_id=job-dead-001 booking_id=BK-9003 ออกจากคิว
[01:59:06.757] พยายามส่งอีเมล ครั้งที่ 1/3 ...
[01:59:06.959] ล้มเหลว: email provider timeout: connection reset after 30s -> รอ backoff 200ms
[01:59:07.161] พยายามส่งอีเมล ครั้งที่ 2/3 ...
[01:59:07.362] ล้มเหลว: email provider timeout: connection reset after 30s -> รอ backoff 400ms
[01:59:07.763] พยายามส่งอีเมล ครั้งที่ 3/3 ...
[01:59:07.963] ล้มเหลว: email provider timeout: connection reset after 30s -> ครบ 3 ครั้งแล้ว หยุด retry
[01:59:07.965] ย้าย job_id=job-dead-001 เข้า dead-letter list 'jobs:booking_confirmation:failed' แล้ว (รอทีมตรวจสอบด้วยมือ)
[01:59:07.966] ตรวจสอบ dead-letter list ตอนนี้มี 1 รายการ
```

และเมื่อตรวจสอบ Redis โดยตรงด้วย `redis-cli LRANGE jobs:booking_confirmation:failed 0 -1` จะเห็น JSON ที่เก็บไว้จริง:

```
{"attempts_made":3,"failed_at":"01:59:07.965","final_error":"email provider timeout: connection reset after 30s","job":{"attempts":0,"enqueued_at":"01:59:06.757","id":"job-dead-001","job_type":"send_booking_confirmation_email","max_attempts":3,"payload":{"booking_id":"BK-9003","email":"unreachable@example.com"}}}
```

ข้อมูลนี้เพียงพอสำหรับทีม support หรือ engineer ที่ตรวจสอบทีหลัง: รู้ว่า booking ไหนได้รับผลกระทบ (`BK-9003`), error สุดท้ายคืออะไร, พยายามไปกี่ครั้ง, และล้มเหลวเมื่อไหร่ — สามารถเขียน admin endpoint หรือ CLI tool ที่อ่านจาก `FAILED_KEY` แล้วให้เลือก "ลอง enqueue ใหม่" (`LPUSH` กลับเข้า `QUEUE_KEY` พร้อม reset `attempts` เป็น 0) หรือ "ปิด case ทิ้ง" (`LREM` ออกจาก dead-letter list) ได้ตามความเหมาะสม จุดสำคัญคือ **job ที่ล้มเหลวถาวรไม่หายไปเงียบ ๆ เหมือนตอนใช้ `tokio::spawn`** แต่ถูกเก็บไว้ให้ตรวจสอบได้เสมอ

### 84.9 Job Idempotency: ป้องกันผลข้างเคียงจากการประมวลผลซ้ำ

Queue ที่เราสร้างด้วย `BRPOP` มีคุณสมบัติ **at-least-once delivery** เหมือนกับ message queue ใน Part 82 — หมายความว่า **job อาจถูกประมวลผลมากกว่าหนึ่งครั้งได้** ตัวอย่างสถานการณ์จริง: worker `BRPOP` ดึง job ออกมาจากคิวสำเร็จ (job หายจากคิวแล้ว ณ จุดนี้) แล้วเริ่มเรียก email provider จนส่งอีเมลสำเร็จ แต่**ก่อน**ที่ worker จะบันทึกผลหรือทำ cleanup ใด ๆ ต่อ — worker process ดันแครช (OOM, container ถูก kill, network partition) ถ้าเรามีระบบ monitoring ที่คอย re-enqueue job ที่ "ดูเหมือนไม่มีคนทำต่อ" (เช่น pattern "reliable queue" ที่ย้าย job ไปไว้ใน "processing list" ชั่วคราวระหว่างทำงาน แล้วถ้า worker ไม่ ack ภายในเวลาที่กำหนดก็ย้ายกลับเข้า queue หลัก) job ตัวเดิมก็จะถูกส่งไปให้ worker ตัวใหม่ประมวลผล**ซ้ำ**

ปัญหาคือ **การส่งอีเมลไม่ใช่การกระทำที่ idempotent โดยธรรมชาติ** — ส่งอีเมลสองครั้งก็คือลูกค้าได้รับอีเมลสองฉบับ (ต่างจาก database update ที่ set ค่าเดิมซ้ำก็ยังได้ผลลัพธ์เหมือนกัน) มาดูปัญหานี้แบบจับได้จริง:

```rust
const BOOKING_ID: &str = "BK-9004";

// จำลอง redelivery: worker ประมวลผล job สำเร็จ ส่งอีเมลไปแล้ว แต่ worker ดันแครช
// (หรือ ack หลุด) "ก่อน" ที่จะลบ job ออกจากคิว -- ระบบคิวแบบ at-least-once (เหมือน
// message queue ใน Part 82) จะส่ง job ตัวเดิมมาให้ประมวลผล "ซ้ำ" อีกครั้ง
async fn send_without_idempotency_check(label: &str) {
    println!("[{}] ({label}) เรียก email provider ส่งอีเมลยืนยัน booking {BOOKING_ID}...", ts());
    call_email_provider(false, 150).await.unwrap();
    println!("[{}] ({label}) ส่งอีเมลสำเร็จ! -- ลูกค้าจะได้รับอีเมลนี้", ts());
}
```

รันสองครั้งจำลอง "การประมวลผลซ้ำ" ได้ผลลัพธ์:

```
=== กรณีที่ 1: ไม่มีการเช็ค idempotency (ปัญหาจริง) ===
[01:59:15.245] worker ประมวลผล job ครั้งที่ 1 (ก่อน crash)
[01:59:15.246] (รอบที่ 1) เรียก email provider ส่งอีเมลยืนยัน booking BK-9004...
[01:59:15.400] (รอบที่ 1) ส่งอีเมลสำเร็จ! -- ลูกค้าจะได้รับอีเมลนี้
[01:59:15.400] (จำลอง) worker แครชก่อนลบ job ออกจากคิว -> คิวคิดว่า job ยังไม่เสร็จ
[01:59:15.400] queue ส่ง job ตัวเดิมมาให้ worker คนใหม่ประมวลผลซ้ำ
[01:59:15.400] (รอบที่ 2 (ซ้ำ!)) เรียก email provider ส่งอีเมลยืนยัน booking BK-9004...
[01:59:15.551] (รอบที่ 2 (ซ้ำ!)) ส่งอีเมลสำเร็จ! -- ลูกค้าจะได้รับอีเมลนี้
>>> ผลลัพธ์: ลูกค้าได้รับอีเมลยืนยัน booking เดียวกัน 2 ฉบับ — ปัญหาจริงแม้จะเล็ก
```

นี่คือปัญหาที่ไม่ทำให้ระบบล่มหรือข้อมูลเสียหาย แต่สร้างความรำคาญและความไม่น่าเชื่อถือให้กับลูกค้า (ลูกค้าอาจสงสัยว่าตัวเองถูก "จองซ้ำ" หรือ "เก็บเงินซ้ำ" ทั้งที่ไม่ได้เกิดขึ้นจริง) วิธีแก้ที่ตรงไปตรงมาที่สุดคือ **บันทึกสถานะ "ส่งอีเมลไปแล้ว" ไว้ที่ใดที่หนึ่งที่ทุก worker เข้าถึงได้ร่วมกัน แล้วเช็คก่อนส่งทุกครั้ง** เช่นเดียวกับแนวคิด Idempotency-Key ที่ Part 78 สอนไว้สำหรับ HTTP request — ในที่นี้เราใช้ Redis (ต่อยอดจาก Part 83) เพราะเป็น infrastructure ที่มีอยู่แล้วในระบบและรองรับ atomic check-and-set ผ่านคำสั่ง `SET key value NX` (ตั้งค่าได้ก็ต่อเมื่อ key ยังไม่มีอยู่ — atomic ในตัวเอง ไม่มี race condition ระหว่างเช็คกับตั้งค่า):

```rust
async fn send_with_idempotency_check(conn: &mut redis::aio::MultiplexedConnection, label: &str) {
    let key = format!("email_sent:{BOOKING_ID}");
    // SET key value NX -> ตั้งค่าได้ก็ต่อเมื่อ key ยังไม่มีอยู่ (atomic check-and-set)
    let acquired: bool = redis::cmd("SET")
        .arg(&key)
        .arg("1")
        .arg("NX")
        .query_async::<Option<String>>(conn)
        .await
        .expect("SET NX failed")
        .is_some();

    if !acquired {
        println!(
            "[{}] ({label}) พบว่า key '{key}' ถูกตั้งไปแล้ว -> ข้ามการส่งอีเมล (ป้องกันส่งซ้ำ)",
            ts()
        );
        return;
    }

    println!("[{}] ({label}) key '{key}' ยังไม่มี -> เรียก email provider จริง", ts());
    call_email_provider(false, 150).await.unwrap();
    println!("[{}] ({label}) ส่งอีเมลสำเร็จ และบันทึกสถานะ 'already sent' แล้ว", ts());
}
```

ผลลัพธ์จริงเมื่อรันซ้ำสองครั้งเหมือนเดิม:

```
=== กรณีที่ 2: เช็คสถานะ 'already sent' ก่อนส่งทุกครั้ง (แก้ปัญหา) ===
[01:59:15.554] worker ประมวลผล job ครั้งที่ 1 (ก่อน crash)
[01:59:15.555] (รอบที่ 1) key 'email_sent:BK-9004' ยังไม่มี -> เรียก email provider จริง
[01:59:15.706] (รอบที่ 1) ส่งอีเมลสำเร็จ และบันทึกสถานะ 'already sent' แล้ว
[01:59:15.706] (จำลอง) worker แครชก่อนลบ job ออกจากคิว -> คิวส่ง job ตัวเดิมมาซ้ำอีกครั้ง
[01:59:15.706] (รอบที่ 2 (ซ้ำ!)) พบว่า key 'email_sent:BK-9004' ถูกตั้งไปแล้ว -> ข้ามการส่งอีเมล (ป้องกันส่งซ้ำ)
>>> ผลลัพธ์: ลูกค้าได้รับอีเมลยืนยันแค่ 1 ฉบับ แม้ job จะถูกประมวลผล 2 ครั้ง
```

สังเกตว่ารอบที่ 2 ไม่ได้เรียก `call_email_provider` เลย (ไม่มี latency 150ms เกิดขึ้น สังเกตจาก timestamp ที่ห่างกันแค่ ~0ms) — worker "ประมวลผล" job ซ้ำจริง แต่ **ผลข้างเคียงที่มองเห็นได้ (ลูกค้าได้รับอีเมล) เกิดขึ้นแค่ครั้งเดียว** ซึ่งคือเป้าหมายของ idempotency: ไม่ได้ห้ามการประมวลผลซ้ำ (บางครั้งห้ามไม่ได้จริง ๆ ในระบบแบบ distributed) แต่ทำให้ **ผลลัพธ์สุดท้ายเหมือนกับประมวลผลครั้งเดียว**

ข้อสังเกตเชิงปฏิบัติสองข้อ: (1) ในตัวอย่างนี้ตั้งค่า key แบบไม่มี TTL (permanent) ซึ่งในระบบจริงมักตั้ง `EX` (expire) ไว้ด้วย เช่น 7 วัน เพื่อไม่ให้ Redis เก็บ key เหล่านี้ค้างอยู่ตลอดไปโดยไม่จำเป็น (booking ที่เก่ากว่านั้นไม่มีทางถูก retry ซ้ำอีกแล้ว) — คำสั่งจริงจะเป็น `SET key val NX EX 604800`, (2) ทางเลือกอื่นแทน Redis คือบันทึกสถานะ `email_sent_at: Option<DateTime<Utc>>` เป็นคอลัมน์ในตาราง `bookings` เองผ่าน SQLx (Part 70) แล้วใช้ `UPDATE bookings SET email_sent_at = now() WHERE id = $1 AND email_sent_at IS NULL` (ใช้ `WHERE ... IS NULL` เป็นตัวเช็ค-แล้ว-ตั้งค่าแบบ atomic เหมือนกับ `SET NX` ของ Redis) วิธีนี้เหมาะกับกรณีที่อยากให้สถานะนี้อยู่ในฐานข้อมูลหลักเพื่อ query ร่วมกับข้อมูล booking ได้ง่ายกว่า ขึ้นอยู่กับว่าระบบมี Redis อยู่แล้วหรือไม่และต้องการ query สถานะนี้รวมกับข้อมูลอื่นแค่ไหน

### 84.10 Scheduled และ Recurring Jobs: `tokio-cron-scheduler`

จนถึงตอนนี้ job ทุกตัวถูก enqueue จาก event ที่เกิดขึ้น (booking ถูกสร้าง) แต่มีงานอีกประเภทที่ไม่ได้ผูกกับ event ใด ๆ เลย ต้องรันตามตารางเวลาซ้ำ ๆ เช่น **งานตอนกลางคืนที่ release booking ที่ยังไม่จ่ายเงินและเลย expiry มาแล้ว** (คืน seat ให้คนอื่นจองได้) หรือ **อีเมลสรุปประจำวัน** งานแบบนี้ไม่ได้มาจากการ enqueue โดยตรง แต่ต้องมีตัวจับเวลาคอยสั่งงานตามรอบ

Crate `tokio-cron-scheduler` ให้ scheduler ที่รันอยู่ใน process เดียวกับแอปได้เลย โดยรับ cron expression 6 ช่อง (มีวินาทีด้วย ต่างจาก cron ของ Unix ที่มี 5 ช่อง):

```rust
use tokio_cron_scheduler::{Job, JobScheduler};

// จำลอง "งานตอนกลางคืน" ที่ต้องรันซ้ำตามตาราง (ปกติจะเป็น "0 0 3 * * *" = ตีสามทุกวัน
// เพื่อ release booking ที่ยังไม่ได้จ่ายเงินและเลย expiry มาแล้ว) แต่ในตัวอย่างนี้ใช้
// "ทุก 3 วินาที" เพื่อให้เห็นผลจริงภายในเวลาสาธิตสั้น ๆ
#[tokio::main]
async fn main() {
    println!("[{}] เริ่ม in-process scheduler (tokio-cron-scheduler)", ts());
    let mut sched = JobScheduler::new().await.expect("create scheduler failed");

    let job = Job::new_async("1/3 * * * * *", |_uuid, _l| {
        Box::pin(async move {
            println!(
                "[{}] [cron] เริ่มงาน cleanup: ค้นหา booking ที่ unpaid และเลย expiry แล้ว",
                ts()
            );
            // จำลองการ query DB + release ที่นั่ง (Part 70/71)
            tokio::time::sleep(std::time::Duration::from_millis(50)).await;
            println!(
                "[{}] [cron] release booking ที่หมดเวลาแล้ว 2 รายการ (BK-8001, BK-8002)",
                ts()
            );
        })
    })
    .expect("invalid cron expression");

    sched.add(job).await.expect("add job failed");
    sched.start().await.expect("start scheduler failed");

    // ปล่อยให้ scheduler รันอยู่ประมาณ 8 วินาที เพื่อให้เห็นการรันซ้ำ 2-3 รอบ
    tokio::time::sleep(std::time::Duration::from_secs(8)).await;
    println!("[{}] จบการสาธิต shutdown scheduler", ts());
    let _ = sched.shutdown().await;
}
```

ผลลัพธ์จริงจากการรัน:

```
[01:59:22.382] เริ่ม in-process scheduler (tokio-cron-scheduler)
[01:59:22.885] [cron] เริ่มงาน cleanup: ค้นหา booking ที่ unpaid และเลย expiry แล้ว
[01:59:22.936] [cron] release booking ที่หมดเวลาแล้ว 2 รายการ (BK-8001, BK-8002)
[01:59:25.391] [cron] เริ่มงาน cleanup: ค้นหา booking ที่ unpaid และเลย expiry แล้ว
[01:59:25.442] [cron] release booking ที่หมดเวลาแล้ว 2 รายการ (BK-8001, BK-8002)
[01:59:28.397] [cron] เริ่มงาน cleanup: ค้นหา booking ที่ unpaid และเลย expiry แล้ว
[01:59:28.448] [cron] release booking ที่หมดเวลาแล้ว 2 รายการ (BK-8001, BK-8002)
[01:59:30.385] จบการสาธิต shutdown scheduler
```

จะเห็นว่างานถูกยิงซ้ำทุกประมาณ 2.5 วินาที (`1/3 * * * * *` หมายถึง "ทุกวินาทีที่หารด้วย 3 ลงตัว" ซึ่ง scheduler จะ align ตาม wall-clock second จริง ไม่ใช่นับ 3 วินาทีจากตอนที่ `start()` ถูกเรียก — จึงเห็นรอบแรกยิงหลัง start แค่ ~0.5 วินาที เพราะตอน start เป็นวินาทีที่ 22 ซึ่งใกล้กับวินาทีถัดไปที่หารด้วย 3 ลงตัว) ในการใช้งานจริงสำหรับ cleanup ตอนกลางคืน จะเปลี่ยน cron expression เป็น `"0 0 3 * * *"` (วินาที 0, นาที 0, ชั่วโมง 3 ของทุกวัน คือตีสามตรง)

**ข้อจำกัดที่ต้องพูดตรง ๆ**: in-process scheduler แบบนี้มีจุดอ่อนสำคัญคือ **มันผูกอยู่กับ lifecycle ของ process** ถ้า process ของแอปพลิเคชัน restart พอดีในช่วงเวลาที่ควรจะรัน (เช่น deploy ใหม่ตอนตี 2:59 แล้ว container ใหม่ยังไม่ทันขึ้นตอนตี 3:00) งาน cleanup ของคืนนั้นจะไม่ถูกรันเลย และไม่มีใครรู้ด้วยว่ามันไม่ถูกรัน (ไม่มี error, ไม่มี log ใด ๆ เพราะ process ที่ควรจะรันมันไม่ได้อยู่ในสถานะที่จะรันได้) ยิ่งไปกว่านั้น ถ้าแอปพลิเคชัน deploy เป็นหลาย instance (เพื่อ load balancing) **ทุก instance จะรัน cron job ของตัวเองพร้อมกัน** ทำให้งาน cleanup ถูกรันซ้ำหลายครั้งในเวลาเดียวกันโดยไม่ได้ตั้งใจ (ในตัวอย่างนี้ผลจะไม่ร้ายแรงเพราะ `UPDATE ... WHERE status = 'unpaid' AND expiry < now()` เป็น idempotent อยู่แล้ว แต่ถ้าเป็นงานอย่าง "ส่งอีเมลสรุปประจำวัน" การรันซ้ำหลาย instance จะทำให้ลูกค้าได้รับอีเมลซ้ำเหมือนปัญหาในหัวข้อ 84.9)

ทางเลือกที่ทนทานกว่าสำหรับ production:

1. **OS-level cron หรือ Kubernetes CronJob**: ให้ orchestrator ที่อยู่ "นอก" lifecycle ของแอปเป็นคนสั่งงานตามตาราง แล้วรันเป็น one-shot process/container แยก วิธีนี้ไม่มีปัญหาเรื่อง process restart พอดีเวลา เพราะ cron/CronJob จะรันใหม่ตามรอบถัดไปเสมอไม่ว่า container ก่อนหน้าจะเป็นอย่างไร และแยก concern ระหว่าง "แอปที่รับ request" กับ "งานตามตาราง" ออกจากกันชัดเจน
2. **Dedicated scheduler service ตัวเดียว + distributed lock**: ถ้าจำเป็นต้องมี in-process scheduler จริง ๆ (เช่นต้องการ logic ที่ผูกกับ state ในหน่วยความจำของแอป) ให้ใช้ distributed lock (เช่น Redis `SET key val NX EX <ttl>` แบบเดียวกับหัวข้อ 84.9) เพื่อให้แน่ใจว่ามีแค่ instance เดียวที่ "ชนะ" การรันงานในรอบนั้น ๆ instance อื่นเช็คแล้วเจอ lock ก็ข้ามไป

บทนี้แสดง `tokio-cron-scheduler` เพื่อให้เห็นว่าการ schedule งานใน Rust ทำได้อย่างไรในทางเทคนิค และเหมาะกับสถานการณ์ที่แอปมี instance เดียว หรือกรณีที่ผลของการรันซ้ำไม่ร้ายแรง (idempotent อยู่แล้ว) — แต่สำหรับงานที่สำคัญและรันในระบบที่มีหลาย instance ควรพิจารณาสองทางเลือกข้างต้นแทน

### 84.11 Capstone: ระบบ Booking-Confirmation-Email แบบ End-to-End

ตอนนี้รวมทุกส่วนเข้าด้วยกันเป็นระบบเดียว: handler ที่ enqueue job, worker ที่ retry พร้อม backoff และเช็ค idempotency, และ scheduled cleanup job — ทั้งหมดรันพร้อมกันจริงในโปรแกรมเดียว (จำลองการแยก process ด้วย `tokio::spawn` หลาย task เพื่อให้สาธิตในบทเดียวได้ ในระบบจริงส่วน handler กับ worker ควรเป็น binary/deployment แยกกันตามที่อธิบายไว้ในหัวข้อ 84.6):

```rust
use redis::AsyncCommands;
use tokio_cron_scheduler::{Job, JobScheduler};
use uuid::Uuid;

async fn run_handler(booking_id: &str) {
    println!("[{}] [handler] client ส่ง POST /bookings booking_id={booking_id}", ts());
    tokio::time::sleep(std::time::Duration::from_millis(5)).await;
    println!("[{}] [handler] บันทึก booking ลง DB สำเร็จ", ts());

    let job = EnqueuedJob {
        id: Uuid::new_v4().to_string(),
        job_type: "send_booking_confirmation_email".to_string(),
        payload: SendBookingConfirmationEmail {
            booking_id: booking_id.to_string(),
            email: "customer@example.com".to_string(),
        },
        attempts: 0,
        max_attempts: 4,
        enqueued_at: ts(),
    };
    let mut conn = get_conn().await;
    let payload = serde_json::to_string(&job).unwrap();
    let _: i64 = conn.lpush(QUEUE_KEY, payload).await.unwrap();
    println!(
        "[{}] [handler] enqueue job {} สำเร็จ -> ตอบ 201 Created กลับ client ทันที",
        ts(), job.id
    );
}

async fn run_worker() {
    println!("[{}] [worker] เริ่มทำงาน รองาน...", ts());
    let mut conn = get_conn().await;
    loop {
        let result: Option<(String, String)> = conn.brpop(QUEUE_KEY, 5.0).await.expect("BRPOP failed");
        let Some((_key, raw)) = result else {
            println!("[{}] [worker] ไม่มีงานใหม่ภายใน timeout -> worker หยุดทำงาน", ts());
            break;
        };
        let job: EnqueuedJob = serde_json::from_str(&raw).unwrap();
        println!(
            "[{}] [worker] ดึงงาน job_id={} booking_id={} มาประมวลผล",
            ts(), job.id, job.payload.booking_id
        );

        // เช็ค idempotency ก่อนส่งอีเมลทุกครั้ง (หัวข้อ 84.9)
        let idem_key = format!("email_sent:{}", job.payload.booking_id);
        let acquired: bool = redis::cmd("SET")
            .arg(&idem_key).arg("1").arg("NX")
            .query_async::<Option<String>>(&mut conn)
            .await.unwrap().is_some();
        if !acquired {
            println!(
                "[{}] [worker] booking_id={} ถูกส่งอีเมลไปแล้ว (idempotency key มีอยู่แล้ว) -> ข้าม",
                ts(), job.payload.booking_id
            );
            continue;
        }

        // retry พร้อม exponential backoff (หัวข้อ 84.7-84.8)
        let mut attempt = 0u32;
        loop {
            attempt += 1;
            println!(
                "[{}] [worker] พยายามส่งอีเมล ครั้งที่ {attempt}/{} booking_id={}",
                ts(), job.max_attempts, job.payload.booking_id
            );
            let should_fail = attempt < 2; // จำลอง provider timeout ครั้งแรก
            match call_email_provider(should_fail, 250).await {
                Ok(()) => {
                    println!(
                        "[{}] [worker] ส่งอีเมลสำเร็จ booking_id={} (ครั้งที่ {attempt})",
                        ts(), job.payload.booking_id
                    );
                    break;
                }
                Err(e) => {
                    if attempt >= job.max_attempts {
                        println!(
                            "[{}] [worker] ครบ {} ครั้ง ยังไม่สำเร็จ: {e} -> เข้า dead-letter",
                            ts(), job.max_attempts
                        );
                        // ปลด idempotency key ออกเพื่อให้ retry รอบใหม่ (จาก DLQ) ส่งอีเมลได้จริง
                        let _: i64 = conn.del(&idem_key).await.unwrap();
                        break;
                    }
                    let backoff_ms = 250u64 * 2u64.pow(attempt - 1);
                    println!("[{}] [worker] ล้มเหลว: {e} -> backoff {backoff_ms}ms", ts());
                    tokio::time::sleep(std::time::Duration::from_millis(backoff_ms)).await;
                }
            }
        }
    }
}

async fn run_scheduler() {
    let mut sched = JobScheduler::new().await.expect("create scheduler failed");
    let job = Job::new_async("1/4 * * * * *", |_uuid, _l| {
        Box::pin(async move {
            println!("[{}] [cron] cleanup job: release booking ที่ unpaid เกิน expiry", ts());
        })
    }).unwrap();
    sched.add(job).await.unwrap();
    sched.start().await.unwrap();
    tokio::time::sleep(std::time::Duration::from_secs(9)).await;
    let _ = sched.shutdown().await;
}

#[tokio::main]
async fn main() {
    let worker = tokio::spawn(run_worker());
    let scheduler = tokio::spawn(run_scheduler());

    tokio::time::sleep(std::time::Duration::from_millis(300)).await;
    run_handler("BK-CAP-1").await;

    let _ = tokio::join!(worker, scheduler);
    println!("[{}] จบการสาธิต capstone", ts());
}
```

ผลลัพธ์จริงจากการรันทั้งระบบพร้อมกัน:

```
[01:59:38.884] [worker] เริ่มทำงาน รองาน...
[01:59:39.186] [handler] client ส่ง POST /bookings booking_id=BK-CAP-1
[01:59:39.192] [handler] บันทึก booking ลง DB สำเร็จ
[01:59:39.200] [worker] ดึงงาน job_id=b1acf39c-58d3-45c2-a950-ae20fdcf9bb3 booking_id=BK-CAP-1 มาประมวลผล
[01:59:39.200] [worker] พยายามส่งอีเมล ครั้งที่ 1/4 booking_id=BK-CAP-1
[01:59:39.201] [handler] enqueue job b1acf39c-58d3-45c2-a950-ae20fdcf9bb3 สำเร็จ -> ตอบ 201 Created กลับ client ทันที
[01:59:39.460] [worker] ล้มเหลว: email provider timeout: connection reset after 30s -> backoff 250ms
[01:59:39.711] [worker] พยายามส่งอีเมล ครั้งที่ 2/4 booking_id=BK-CAP-1
[01:59:39.962] [worker] ส่งอีเมลสำเร็จ booking_id=BK-CAP-1 (ครั้งที่ 2)
[01:59:41.390] [cron] cleanup job: release booking ที่ unpaid เกิน expiry
[01:59:45.062] [worker] ไม่มีงานใหม่ภายใน timeout -> worker หยุดทำงาน
[01:59:45.401] [cron] cleanup job: release booking ที่ unpaid เกิน expiry
[01:59:47.888] จบการสาธิต capstone
```

มีรายละเอียดที่น่าสังเกตหนึ่งจุด: บรรทัด `[handler] enqueue job ... สำเร็จ` ที่ `39.201` ปรากฏขึ้น **หลัง** บรรทัด `[worker] ดึงงาน ... มาประมวลผล` ที่ `39.200` ทั้งที่ตามลำดับเหตุผลแล้ว การ `LPUSH` (จบใน `run_handler`) ต้องเกิดก่อนการ `BRPOP` จะดึงงานออกมาได้เสมอ — สิ่งที่เกิดขึ้นคือคำสั่ง `LPUSH` ใน Redis เสร็จสมบูรณ์ไปแล้วจริง (นั่นคือเหตุผลที่ worker ดึงงานออกมาได้) แต่ `println!` บรรทัดที่รายงานผลใน `run_handler` ถูกพิมพ์ทีหลัง เพราะ scheduler ของ tokio สลับไปรัน task ของ worker ก่อนที่ `run_handler` จะได้กลับมาทำงานบรรทัดถัดไปจาก `.await` — นี่คือบทเรียนสำคัญเชิงระบบ (ไม่ใช่แค่เชิง syntax): **ลำดับของ log ที่พิมพ์ออกมาไม่ได้สะท้อนลำดับเหตุการณ์จริงของระบบเสมอไปในโปรแกรม concurrent/distributed** ถ้าต้องการลำดับเหตุการณ์ที่แม่นยำ ต้องอ้างอิงจาก timestamp หรือ sequence number ที่บันทึกไว้ ณ จุดเกิดเหตุจริง ไม่ใช่จุดที่ log ถูกพิมพ์ ซึ่งเป็นเหตุผลที่ทุกตัวอย่างในบทนี้ใช้ `ts()` แนบไปกับทุกบรรทัดเสมอ

นอกจากนี้ยังเห็นว่า cron cleanup job (`[cron] cleanup job: ...`) รันแทรกอยู่ระหว่างที่ worker กำลังทำงานอยู่ (ที่ `41.390` และ `45.401`) แสดงให้เห็นว่าทั้งสาม component — handler, worker, scheduler — ทำงานเป็นอิสระจากกันอย่างแท้จริงภายใน runtime เดียว: handler enqueue เสร็จแล้วก็จบไป (ในระบบจริงคือ HTTP response ถูกส่งกลับไปแล้ว), worker วนลูปรอ-ประมวลผล-รอ-... ไปเรื่อย ๆ จนกว่าจะไม่มีงานเข้ามาภายใน timeout ที่กำหนด, และ scheduler ก็ยิงงานตามตารางเวลาของตัวเองโดยไม่สนใจว่า handler หรือ worker กำลังทำอะไรอยู่ — นี่คือภาพรวมของระบบ background job ที่ทำงานได้จริงในเชิง production แม้จะย่อส่วนลงมาให้รันในโปรแกรมเดียวเพื่อการสาธิตก็ตาม

## กับดักที่พบบ่อย (Common Pitfalls)

**1. ลืม `use redis::AsyncCommands;` แล้วเรียก `.lpush()`/`.brpop()` ไม่ได้**

เมธอดอย่าง `.lpush()`, `.brpop()`, `.lrange()` มาจาก trait `AsyncCommands` ที่ต้อง import เข้ามาก่อนจะเรียกใช้ผ่าน connection object ได้ — ถ้าลืม จะได้ compiler error แบบนี้จริง ๆ:

```
error[E0599]: no method named `lpush` found for mutable reference `&mut MultiplexedConnection` in the current scope
  --> src/lib.rs:39:23
   |
39 |     let _: i64 = conn.lpush(QUEUE_KEY, payload).await.expect("LPUSH failed");
   |                       ^^^^^
   |
   = help: items from traits can only be used if the trait is in scope
help: trait `AsyncCommands` which provides `lpush` is implemented but not in scope; perhaps you want to import it
   |
 1 + use redis::AsyncCommands;
   |
```

Rust compiler ค่อนข้างช่วยเราได้ดีในกรณีนี้ — error message บอกตรง ๆ เลยว่าต้อง import trait ไหน วิธีแก้ก็ตรงตาม suggestion: เพิ่ม `use redis::AsyncCommands;` ที่หัวไฟล์ นี่คือดีไซน์ทั่วไปของ Rust ที่ methods บน external types (เช่น `MultiplexedConnection` ของ crate `redis`) มักถูกให้มาผ่าน trait แยก เพื่อให้ผู้ใช้เลือก import ได้เฉพาะที่ต้องการ ไม่ยัดทุก method ลงใน struct หลักตรง ๆ

**2. Job schema เปลี่ยนแปลงระหว่างที่ job เก่ายังอยู่ในคิว**

Job queue ที่ backed ด้วย Redis list เก็บ job เป็น JSON ที่ persist ข้าม deploy ได้ — ถ้า deploy โค้ดใหม่ที่เพิ่ม field บังคับ (ไม่ใช่ `Option<T>`) ลงใน struct ของ job แต่ยังมี job เก่าที่ enqueue ไว้ตั้งแต่ก่อน deploy ค้างอยู่ใน queue worker ตัวใหม่จะ deserialize job เก่าไม่ผ่าน:

```
Err(Error("missing field `email`", line: 1, column: 90))
```

นี่ไม่ใช่ compiler error แต่เป็น runtime error จาก `serde_json::from_str` ที่พังตอน deserialize เพราะ JSON เก่าไม่มี field `email` ที่ struct เวอร์ชันใหม่ต้องการ วิธีป้องกัน: (1) เพิ่ม field ใหม่เป็น `Option<T>` พร้อม `#[serde(default)]` เสมอเมื่อจะแก้ schema ของ job ที่อาจมีของเก่าค้างอยู่ในคิว ไม่ใช่เพิ่มเป็น field บังคับตรง ๆ, (2) ทำ "drain queue ก่อน deploy" เป็นขั้นตอนมาตรฐานสำหรับ breaking change (ปล่อยให้ worker เคลียร์คิวจนหมดก่อนแล้วค่อย deploy โค้ดใหม่), หรือ (3) จับ error การ deserialize แล้วย้าย job ที่ deserialize ไม่ผ่านไปเข้า dead-letter list ทันที (เหมือนหัวข้อ 84.8) แทนที่จะให้ worker panic และตายไปทั้ง process

**3. Worker เชื่อมต่อ Redis คนละ endpoint กับ handler (หรือ Redis ยังไม่ขึ้น)**

ถ้า `REDIS_URL` ที่ handler กับ worker ใช้ไม่ตรงกัน (เช่น handler ต่อ production Redis แต่ worker ยังตั้ง URL แบบ default ไว้ตอน dev) worker จะไม่เห็น job ที่ handler enqueue ไว้เลย และจะดูเหมือนว่า "queue ทำงานได้แต่ไม่มี job เข้ามา" ซึ่ง debug ยาก ถ้าเชื่อมต่อไม่ได้เลย (Redis ไม่ได้รันอยู่ที่ port ที่ระบุ) จะได้ error ตรง ๆ แบบนี้:

```
เชื่อมต่อไม่ได้: Connection refused (os error 111)
```

วิธีป้องกัน: อ่าน `REDIS_URL` จาก environment variable เดียวกันทั้ง handler และ worker (ผ่าน config ที่ share กัน) ไม่ hardcode ค่าไว้คนละที่ และเพิ่ม health check ตอน worker start ที่ทดลอง `PING` ไปที่ Redis ก่อนเข้าลูป `BRPOP` เพื่อ fail-fast ตั้งแต่ startup ถ้าต่อไม่ได้ แทนที่จะปล่อยให้ worker "ดูเหมือนทำงานอยู่" ทั้งที่ไม่เคยเชื่อมต่อสำเร็จ

**4. เข้าใจผิดว่า drop `JoinHandle` ของ `tokio::spawn` จะทำให้ job ยังทำงานต่อได้แม้ web server restart**

ต่อเนื่องจากความเข้าใจผิดที่ Part 48 อธิบายไว้ (drop `JoinHandle` ไม่ยกเลิก task ทันที — task ยัง "detached" ทำงานต่อได้) หลายคนสรุปผิดต่อไปว่า "ถ้าอย่างนั้นก็ปลอดภัยแล้ว ปล่อย spawn ไปเลยไม่ต้องมี queue" แต่อย่างที่พิสูจน์ในหัวข้อ 84.3 ด้วยผลลัพธ์จริง ("background task" ไม่ได้พิมพ์ข้อความใดออกมาเลยแม้แต่บรรทัดแรก) — **การ detached หมายถึง "ไม่ถูกยกเลิกโดยการ drop handle" เท่านั้น ไม่ใช่ "รอดจาก process restart"** เมื่อ `Runtime` ทั้งตัวถูก drop (ซึ่งเกิดขึ้นทุกครั้งที่ process ปิดตัว ไม่ว่าจะปิดแบบปกติ, crash, หรือถูก orchestrator สั่ง restart) task ที่ detached อยู่ทั้งหมดจะถูกยกเลิกไปด้วยเสมอ ไม่มีข้อยกเว้น วิธีแก้คือสิ่งที่บทนี้สอนมาตลอด: เก็บ job ไว้ใน storage ที่อยู่นอก process (Redis, database) ก่อนเริ่มประมวลผล ไม่ใช่เก็บไว้แค่ในหน่วยความจำของ task ที่ spawn ไว้

**5. Retry loop ไม่มี `max_attempts` หรือคำนวณ backoff ผิดจนกลายเป็น infinite/near-zero wait**

ถ้าลืมเช็ค `attempt >= max_attempts` ก่อน retry loop จะวนไม่มีที่สิ้นสุดสำหรับ job ที่ล้มเหลวถาวร (เช่น อีเมลผิดฟอร์แมต) ทำให้ worker ติดอยู่กับ job นั้นตลอดไปและ job อื่นในคิวไม่ได้ถูกประมวลผล อีกกรณีที่พบบ่อยคือคำนวณ backoff ผิด เช่น เขียน `base_ms * attempt` (linear) ทั้งที่ตั้งใจจะทำ exponential (`base_ms * 2u64.pow(attempt - 1)`) ทำให้ backoff โตช้าเกินไปเมื่อ provider มีปัญหาต่อเนื่องยาวนาน หรือลืม cap ค่าสูงสุดของ backoff (เช่น "ไม่ควรรอเกิน 5 นาทีต่อครั้งไม่ว่า attempt จะเยอะแค่ไหน") จนกรณี `attempt` สูงมาก ๆ ทำให้ `2u64.pow(attempt - 1)` overflow หรือรอนานเกินสมควร ควรเขียนแบบมี cap เสมอ เช่น `let backoff_ms = (base_ms * 2u64.pow(attempt.min(10) - 1)).min(300_000);`

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** แก้ไขตัวอย่าง `worker.rs` ในหัวข้อ 84.6 ให้วนลูป `BRPOP` ไม่มีที่สิ้นสุด (ลบเงื่อนไข timeout ออก ใช้ `brpop` กับ timeout เป็น `0.0` ซึ่งหมายถึง "รอตลอดไป" ตาม semantic ของ Redis) แล้วเพิ่ม `println!` นับจำนวน job ที่ประมวลผลสำเร็จสะสมตั้งแต่ worker เริ่มทำงาน — hint: เก็บตัวแปร `let mut processed_count = 0u64;` ไว้นอกลูป แล้วเพิ่มค่าทุกครั้งที่ประมวลผลสำเร็จ

2. **(กลาง)** เพิ่ม field `last_error: Option<String>` และ `next_retry_at: Option<String>` เข้าไปใน struct `EnqueuedJob` แล้วแก้ worker ให้บันทึกค่าเหล่านี้ (ผ่าน `serde_json::to_string` แล้ว `LPUSH` กลับเข้า queue เป็น job ตัวใหม่ที่มี `attempts` เพิ่มขึ้น 1) แทนการ retry แบบ in-process loop เหมือนหัวข้อ 84.7 — วิธีนี้ทำให้ worker หยิบ job อื่นไปทำระหว่างที่ job นี้รอ backoff ได้ hint: ต้องมีอีก process/task ที่คอยเช็คว่า job ที่ `LPUSH` กลับมาถึงเวลา `next_retry_at` แล้วหรือยัง ก่อนจะให้ worker ตัวใดหยิบไปประมวลผลจริง (หรือจะทำง่าย ๆ ก่อนคือ worker เช็คเองตอนหยิบ job มาว่ายังไม่ถึงเวลาก็ `LPUSH` กลับเข้าไปท้ายคิวเฉย ๆ แล้ว `continue` ก็ได้)

3. **(ยาก)** เขียน integration test (อ้างอิงแนวทางจาก Part 33) ที่ยืนยันว่าระบบ idempotency ในหัวข้อ 84.9 ทำงานถูกต้องจริง: spawn worker เป็น `tokio::spawn`, enqueue job เดียวกัน (`booking_id` เดิม) สองครั้งติดกันเข้า queue, รอให้ worker ประมวลผลทั้งสอง job เสร็จ, แล้ว assert ว่า counter ที่จำลอง "จำนวนครั้งที่เรียก email provider จริง" (ใช้ `Arc<AtomicU64>` แบบที่เรียนจาก Part 51) มีค่าเท่ากับ 1 ไม่ใช่ 2 — hint: ต้อง `.await` ให้ worker มีเวลาประมวลผลทั้งสอง job ก่อน assert (เช่นด้วย `tokio::time::sleep` สั้น ๆ หรือ poll ค่า counter จนกว่าจะนิ่ง) และต้อง `redis-cli DEL email_sent:<booking_id>` ก่อนเริ่ม test ทุกครั้งเพื่อไม่ให้ state จาก test รอบก่อนรั่วไหลมา

4. **(ยาก/ประยุกต์)** ขยาย capstone ในหัวข้อ 84.11 ให้ dead-letter job (จากหัวข้อ 84.8) ถูกนำกลับมา retry ใหม่โดยอัตโนมัติได้ผ่าน scheduled job อีกตัว: เขียน cron job ที่รันทุก 1 นาที (ในตัวอย่างสาธิตใช้ทุก 5 วินาทีเพื่อดูผลเร็ว) คอยเช็ค `jobs:booking_confirmation:failed`, ถ้ามี entry ที่ `failed_at` เก่ากว่า threshold ที่กำหนด (เช่น 10 วินาทีในตัวอย่างสาธิต) ให้ deserialize กลับเป็น `EnqueuedJob`, reset `attempts` เป็น 0, แล้ว `LPUSH` กลับเข้า `jobs:booking_confirmation` อีกครั้ง พร้อมลบ entry นั้นออกจาก dead-letter list ด้วย `LREM` — hint: ต้องระวังไม่ให้ retry job เดิมซ้ำไม่มีที่สิ้นสุดถ้า provider ยังล่มอยู่ ควรมี field เพิ่ม เช่น `dlq_retry_count` เพื่อจำกัดจำนวนครั้งที่จะดึงจาก dead-letter กลับมา retry อัตโนมัติ ก่อนต้องให้คนตรวจสอบด้วยมือจริง ๆ

## สรุป

บทนี้เริ่มจากปัญหาที่วัดผลได้จริง: การส่งอีเมลแบบ synchronous ใน request handler ทำให้ response ช้าลงกว่า 1 วินาทีและผูก failure ของ email provider เข้ากับความสำเร็จของการจองตั๋วโดยไม่จำเป็น จากนั้นสำรวจ `tokio::spawn` แบบ fire-and-forget ซึ่งแก้ปัญหา latency ได้ แต่พิสูจน์ให้เห็นด้วยผลลัพธ์จริงว่ามันไม่ทนต่อการที่ process ปิดตัว (ต่อยอดความเข้าใจเรื่อง detached task จาก Part 48 ว่า detach ไม่ได้แปลว่ารอดจาก runtime shutdown) จึงนำไปสู่การสร้าง job queue เองด้วย Redis `LPUSH`/`BRPOP` ซึ่งเป็น primitive ที่โปร่งใสและต่อยอดจาก infrastructure ของ Part 83 ได้ทันที — เราสร้างระบบครบวงจร: enqueue job จาก handler, worker แยก process ที่พิสูจน์การ decoupling ด้วย timestamp จริง, retry พร้อม exponential backoff ที่แสดง interval การรอจริง, dead-letter list สำหรับ job ที่ล้มเหลวถาวร, idempotency check ที่ป้องกันผลข้างเคียงซ้ำจาก at-least-once delivery (เชื่อมกับแนวคิด Idempotency-Key จาก Part 78), และ scheduled job ด้วย `tokio-cron-scheduler` พร้อมข้อจำกัดที่ต้องรู้เมื่อนำไปใช้จริงกับระบบที่มีหลาย instance ทุกตัวอย่างถูก compile และรันจริงเพื่อยืนยันว่าใช้งานได้ ไม่ใช่แค่โค้ดที่ดูถูกต้องบนกระดาษ

จาก background job ที่จัดการงานภายในแอปเดียวในบทนี้ Part ถัดไปจะเปลี่ยนโฟกัสไปที่การทำให้ API ของเราสื่อสารกับผู้ใช้ภายนอก (frontend team, third-party integrator) ได้ชัดเจนขึ้นผ่านเอกสารที่เครื่องอ่านได้และมนุษย์อ่านเข้าใจ — **OpenAPI/Swagger ด้วย `utoipa`** ซึ่งจะสร้าง schema เอกสารจาก type และ handler ที่เราเขียนไว้แล้วโดยตรง

---

**Part ก่อนหน้า:** [Caching ด้วย Redis](part-083-redis-caching.md) | **Part ถัดไป:** [API Documentation ด้วย OpenAPI/Swagger (utoipa)](part-085-openapi-utoipa.md)
