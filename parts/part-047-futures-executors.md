# Part 47: Futures และ Executors

> โมดูล: ระดับสูง (Advanced) | ระดับ: สูง | เวลาโดยประมาณ: 260 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายนิยาม**จริง**ของ trait `Future` ได้ทุกส่วน (`type Output`, `fn poll(self: Pin<&mut Self>, cx: &mut
  Context<'_>) -> Poll<Self::Output>`) และเชื่อมโยงแต่ละส่วนเข้ากับสิ่งที่เรียนมาแล้ว — associated type จาก
  Part 22, ความคล้ายกับ `Iterator::Item` จาก Part 25, และปรัชญา enum แทน exception จาก Part 10/11
- เขียน**custom Future ของตัวเองตั้งแต่ต้น** ได้หลายแบบ ทั้งแบบที่ไม่รอเหตุการณ์ภายนอกเลย (busy-poll) และแบบที่
  รอเหตุการณ์จริงจาก background thread ผ่าน `Waker` พร้อมพิสูจน์ด้วยโค้ดที่รันได้จริงว่า async/await ที่เรียนใน
  Part 46 ไม่ใช่มายากล แต่คือ syntax sugar เหนือกลไกที่เขียนด้วยมือได้ทั้งหมด
- อธิบายได้อย่างแม่นยำว่า `Waker` มีไว้ทำไม เพราะเหตุใด executor ที่ไม่มีกลไกนี้ต้อง busy-loop เปลืองซีพียู
  และเขียนโค้ดที่ใช้ `Waker` ถูกต้อง (รู้ว่าต้อง `.clone()` เก็บไว้ ไม่ใช่เก็บ `&Waker` ตรง ๆ)
- อธิบาย `Pin<T>` ได้ในระดับที่ใช้งานจริง — เชื่อมโยงกลับไปยังปัญหา self-referential struct ที่ Part 23 เกริ่นไว้
  และที่ Part 46 ทีเซอร์ไว้ว่า Future ต้องพึ่งมัน พร้อมอธิบาย `Unpin` auto trait (ตามแนวทาง Part 21/40) ในระดับที่
  "จำได้ทันทีที่เห็นในลายเซ็นฟังก์ชันหรือ error message" โดยไม่ต้องลงรายละเอียด unsafe pinning เต็มรูปแบบ
- **สร้าง executor แบบง่ายที่สุดของตัวเองตั้งแต่ต้น** ที่รันหลาย Future พร้อมกันบน thread เดียว ได้จริง และพิสูจน์
  ด้วยผลลัพธ์การรันจริงว่ามันคือ cooperative multitasking แท้ ๆ ไม่ใช่แค่คำโฆษณา
- ใช้ combinator จาก crate `futures` (เช่น `future::join`, `future::select`) เพื่อรวม Future หลายตัวได้โดยไม่ต้อง
  พึ่ง runtime เต็มรูปแบบอย่าง Tokio และอธิบายได้ว่าทำไมโปรเจกต์จริงแทบไม่มีใครเขียน executor เองเลย — ปูทางสู่
  Part 48 (Tokio)

## ความรู้ที่ต้องมีมาก่อน

บทนี้คือ**ภาคต่อโดยตรง**ของ **Part 46 (Async/Await เบื้องต้น)** — Part 46 สอน syntax `async fn`/`.await` และโมเดล
ทางความคิดแบบ "state machine" (เขียนโค้ดที่ดูเหมือนทำงานทีละบรรทัด แต่ compiler แปลงเป็น state machine ที่หยุด
พักได้ระหว่างทาง) และจงใจ**เลื่อน**รายละเอียดสองเรื่องมาไว้บทนี้ตรง ๆ: (1) นิยามจริงของ trait `Future` ที่ซ่อนอยู่
หลัง syntax `.await`, และ (2) กลไก `Pin` ที่ Part 46 แค่ทีเซอร์ไว้ว่า "state machine พวกนี้ต้องพึ่ง `Pin` แต่จะ
เจาะลึกใน Part 47" — บทนี้คือคำตอบเต็มรูปแบบของทั้งสองเรื่องนั้น ถ้ายังไม่ผ่าน Part 46 หรือจำโมเดล state machine
ไม่ได้แล้ว ควรย้อนไปทวนก่อน เพราะบทนี้จะไม่สอน syntax `async`/`.await` ซ้ำอีกรอบ

นอกจาก Part 46 แล้ว บทนี้ยังอ้างอิงเนื้อหาต่อไปนี้อย่างหนัก:

- **Part 19 (Traits เบื้องต้น)** และ **Part 21-22 (Traits/Generics ขั้นสูง)**: นิยามของ `Future` ใช้
  **associated type** (`type Output`) แบบเดียวกับที่ Part 22 หัวข้อ 22.6 อธิบายไว้ และหัวข้อ 47.2 ของบทนี้จะทำ
  สิ่งเดียวกันกับที่ Part 25 ทำกับ `Iterator` — เดินผ่านนิยาม trait จริงทีละส่วนอย่างละเอียด
- **Part 25 (Iterators เบื้องต้น)**: หัวข้อ 47.2 จะทำการเปรียบเทียบที่ชัดเจนที่สุดของบทนี้ — `Future` เหมือนกับ
  `Iterator` ที่ให้ค่าออกมา "ได้อย่างมากหนึ่งค่า สักวันหนึ่งในอนาคต" ถ้าจำนิยามของ `Iterator::next() ->
  Option<Self::Item>` จาก Part 25 ได้แม่น หัวข้อนี้จะเข้าใจง่ายขึ้นมาก
- **Part 10-11 (Enum, Pattern Matching, Option<T>)**: `Poll<T>` เป็น enum สองแบบ (`Ready(T)`/`Pending`) ที่ใช้
  ปรัชญาการออกแบบเดียวกับ `Option<T>` — แทนความหมาย "ยังไม่มีคำตอบ" ด้วย variant ของ enum ไม่ใช่ null หรือ
  exception
- **Part 23 (Lifetimes ขั้นสูง)**: บทนั้นเกริ่นปัญหา **self-referential struct** ไว้ (struct ที่มี field หนึ่งเก็บ
  reference ชี้ไปยัง field อีกตัวในโครงสร้างเดียวกัน) และบอกว่า "lifetime ธรรมดาแก้ปัญหานี้ไม่ได้" — หัวข้อ 47.7
  ของบทนี้คือคำตอบเต็มรูปแบบว่าโลกจริงแก้ปัญหานี้อย่างไรด้วย `Pin`
- **Part 37 (Threads พื้นฐาน)**: ตัวอย่าง `Delay` future ในหัวข้อ 47.6 ใช้ `thread::spawn` และ `thread::sleep`
  ตรงตามที่เรียนมาแล้วทุกประการ เพื่อสร้างเหตุการณ์ "เวลาผ่านไปจริง" ที่ future ต้องรอ
- **Part 39 (Mutex, Arc)**: `Delay` และ executor ในบทนี้ใช้ `Arc<Mutex<T>>` แชร์สถานะข้าม thread แบบเดียวกับที่
  Part 39 สอนไว้ทุกประการ — ไม่มีเทคนิคใหม่ในส่วนนี้ แค่นำมาประยุกต์กับปัญหาใหม่
- **Part 40 (Send, Sync)**: บทนั้นเกริ่นไว้ว่า `Unpin` เป็น auto trait ตัวที่สามในตระกูลเดียวกับ `Send`/`Sync`
  แต่บอกให้ "รอเรียนเต็มรูปแบบใน Part 46" — Part 46 ทีเซอร์ไว้อีกรอบว่าจะเจาะลึกใน Part 47 — **นี่คือ Part นั้น**
  หัวข้อ 47.7 จะอธิบาย `Unpin` แบบเต็มโดยใช้หลักการ auto trait (structural/recursive) เดียวกับที่ Part 40 สอน
  `Send`/`Sync` ไว้ทุกประการ

## เนื้อหา

### 47.1 ทวนความจำจาก Part 46: โมเดล State Machine และสิ่งที่ถูกเลื่อนมาไว้ที่นี่

Part 46 สอนให้เราเขียนโค้ดแบบนี้ได้แล้ว:

```rust
async fn greet(name: &str) -> String {
    format!("สวัสดี, {name}!")
}
```

และอธิบายว่า `async fn` ไม่ได้ "รันทันที" เหมือนฟังก์ชันปกติ — มันสร้าง**ค่า**ขึ้นมาตัวหนึ่ง (ค่าที่ implement
trait `Future`) ที่ยังไม่ทำอะไรเลยจนกว่าจะถูก `.await` หรือถูกส่งเข้า executor ไปขับเคลื่อน แนวคิด "state machine"
ที่ Part 46 ใช้อธิบายคือ: compiler แปลงโค้ดในบล็อก `async` ให้กลายเป็น struct ที่มีหลาย "สถานะ" ภายใน (คล้าย enum)
— ทุกครั้งที่โค้ดเจอ `.await` มันคือจุดที่ state machine **อาจ**หยุดพักได้ ถ้าสิ่งที่ await ยังไม่พร้อม

ตารางนี้สรุปสิ่งที่ Part 46 ทีเซอร์ไว้ แล้วบอกตรง ๆ ว่าจะเจาะลึกใน Part 47 — บทนี้คือจุดที่ทุกอย่างจะถูกไขให้กระจ่าง:

| สิ่งที่ Part 46 พูดถึง | สถานะตอนนั้น | หัวข้อในบทนี้ที่จะไขให้ครบ |
|---|---|---|
| `async fn` คืนค่าที่ "implement `Future`" | บอกชื่อ trait ไว้ ไม่ได้โชว์นิยามเต็ม | 47.2 |
| `.await` คือจุดที่ state machine "หยุดพักได้" | อธิบายแนวคิดกว้าง ๆ ไม่ได้ลงกลไกจริง | 47.4, 47.9 |
| "ยังไม่พร้อม" ของ Future แทนด้วยอะไร | ไม่ได้พูดถึง `Poll` เลย | 47.3 |
| "executor คือตัวขับเคลื่อน Future ให้ทำงาน" | บอกแค่ว่ามีอยู่ ไม่ได้สอนเขียนเอง | 47.8, 47.10 |
| state machine ต้องใช้ `Pin` เพราะเหตุผลบางอย่าง | ทีเซอร์ตรง ๆ ว่า "จะเจาะลึกใน Part 47" | 47.7 |
| executor "รู้"ได้อย่างไรว่าเมื่อไหร่ควร poll ใหม่ | ไม่ได้พูดถึงเลย | 47.5, 47.6 |

จุดสำคัญที่สุดที่ต้องเข้าใจก่อนเริ่มเนื้อหาใหม่คือ: **ทุกอย่างที่ `async`/`.await` ทำให้อัตโนมัติใน Part 46 ล้วน
เขียนด้วยมือได้ทั้งหมด** ไม่มีส่วนไหนของกลไกนี้ที่เป็น "ของวิเศษที่ compiler แอบทำเบื้องหลังแบบเข้าไม่ถึง" — มันคือ
trait ธรรมดา (`Future`), enum ธรรมดา (`Poll`), และ struct ธรรมดาที่แชร์ข้อมูลข้าม thread ด้วยเทคนิคจาก Part 37-39
ที่เรารู้จักมาแล้วทั้งหมด บทนี้จะเขียนทุกชิ้นส่วนด้วยมือ ทีละชิ้น จนกระทั่งสร้าง executor ของตัวเองที่รันโค้ด async
จริงได้ในตอนจบบท

### 47.2 นิยามจริงของ Trait `Future`

นี่คือนิยามจริงของ `Future` จาก `std::future::Future` (ไม่ใช่ pseudocode — นี่คือของจริงที่คุณ implement ได้ตรง ๆ):

```rust
use std::pin::Pin;
use std::task::{Context, Poll};

trait Future {
    type Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

(นี่คือนิยามที่คัดลอกโครงสร้างมาให้ดูเพื่อการเรียน ไม่ต้องเขียน `trait Future` นี้เอง — เราจะ `impl` ให้กับ
`std::future::Future` ตัวจริงเสมอ)

**อ่านทีละส่วน โดยเชื่อมกับสิ่งที่เรียนมาแล้วทุกจุด:**

**`type Output;`** — คือ **associated type** ตามหลักการที่ Part 22 หัวข้อ 22.6 สอนไว้ (และที่ Part 25 นำมาใช้กับ
`Iterator::Item` อย่างเต็มรูปแบบ) ความหมายคือ "ทุก type ที่ implement `Future` ต้องเลือกชนิดข้อมูลหนึ่งชนิดมาผูกกับ
ชื่อ `Output` — คือชนิดของค่าที่ future ตัวนี้จะให้ออกมา**เมื่อมันเสร็จ**" ตัวอย่าง `greet()` ข้างบนมี
`Output = String` เพราะ `async fn` ที่คืนค่า `String` จะถูกแปลงเป็น future ที่มี `Output` เท่ากับ `String` เสมอ

**นี่คือการเปรียบเทียบที่ทรงพลังที่สุดของทั้งบทนี้**: ลองเทียบนิยามของ `Future` กับนิยามของ `Iterator` ที่เรียนมา
แล้วใน Part 25 หัวข้อ 25.2 แบบเคียงข้างกัน:

```
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}

trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

โครงสร้างเหมือนกันเป๊ะ! **`Future` ก็คือ `Iterator` ที่ให้ค่าออกมาได้อย่างมากแค่หนึ่งค่าเท่านั้น** — `Iterator` เรียก
`next()` ได้เรื่อย ๆ จนกว่าจะได้ `None` ไปตลอดกาล (แทน "ไม่มีสมาชิกเหลือแล้ว") ส่วน `Future` เรียก `poll()` ได้
เรื่อย ๆ จนกว่าจะได้ `Poll::Ready(value)` เพียงครั้งเดียว (แทน "เสร็จแล้ว นี่คือผลลัพธ์") แล้วก็ไม่ควรถูกเรียก
`poll()` ซ้ำอีกต่อไป (ตามธรรมเนียมของ trait — คล้ายกับที่ Part 25 บอกว่า `Iterator` "ไม่ควร" ถูกเรียก `next()` ต่อ
หลังได้ `None` แม้ trait จะไม่ได้ห้ามในระดับ type system ก็ตาม) ถ้าจำได้ว่า `Iterator` เป็น trait ที่ "สอนตัวเองได้
ดีที่สุด" ว่าระบบ trait/associated type ของ Rust ถูกออกแบบมาเพื่อรองรับ pattern แบบนี้ — `Future` ก็คือตัวอย่างที่
สองของ pattern เดียวกันนั่นเอง เพียงแค่เปลี่ยน "จำนวนค่าที่ให้ได้" จาก "ได้หลายค่า" เป็น "ได้อย่างมากหนึ่งค่า"

**`fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;`** — นี่คือ method เดียวที่ต้อง
ประกาศเอง (เหมือน `next()` ของ `Iterator` ที่เป็น method เดียวที่ต้องเขียน ส่วน method อื่นกว่า 30 ตัวของ `Future`
เช่น `.map()`, `.then()` มาจาก crate `futures` ไม่ใช่ trait หลักใน `std`) มีสามส่วนที่ยังไม่คุ้นเคย:

1. **`self: Pin<&mut Self>`** — แปลก! ปกติเราเขียน `&mut self` ไม่ใช่ `self: Pin<&mut Self>` นี่คือจุดที่ `Future`
   ต่างจาก `Iterator` อย่างชัดเจนที่สุด และเป็นเหตุผลที่บทนี้ต้องมีหัวข้อ `Pin` แยกทั้งหัวข้อ (47.7) — ตอนนี้ให้
   จำแค่ว่า "มันคือ `&mut self` เวอร์ชันที่มีการันตีพิเศษเพิ่มเข้ามา" ก่อน แล้วเราจะอธิบายว่าการันตีนั้นคืออะไรและ
   ทำไมจำเป็นในหัวข้อ 47.7 ให้ครบ
2. **`cx: &mut Context<'_>`** — พารามิเตอร์ใหม่ที่ `Iterator::next()` ไม่มี มันคือกล่องที่บรรจุ `Waker` (จะอธิบาย
   เต็มในหัวข้อ 47.5) — future ใช้มันเพื่อ "บอกทางให้ executor รู้ว่าจะแจ้งเตือนตอนไหน" เมื่อ future ยังไม่พร้อม
3. **`Poll<Self::Output>`** — ผลลัพธ์ที่คืนไม่ใช่ `Option<T>` แต่เป็น `Poll<T>` (จะอธิบายเต็มในหัวข้อ 47.3)

ก่อนจะไปลึกกว่านี้ ลองเขียน `Future` แบบที่ง่ายที่สุดที่เป็นไปได้ก่อน — Future ที่**พร้อมทันที**ตั้งแต่ poll ครั้ง
แรก ไม่มีสถานะภายในให้จำ ไม่มีการรอคอยอะไรเลย:

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll, Waker};

struct ReadyNow(i32);

impl Future for ReadyNow {
    type Output = i32;

    fn poll(self: Pin<&mut Self>, _cx: &mut Context<'_>) -> Poll<Self::Output> {
        Poll::Ready(self.0)
    }
}

fn main() {
    let mut fut = ReadyNow(42);

    // Waker::noop() คือ Waker ปลอมที่ไม่ทำอะไรเลยตอน .wake() ถูกเรียก
    // (เป็นของ std เอง stabilize มาตั้งแต่ Rust 1.85 มีไว้ใช้ทดสอบ/สอนโดยเฉพาะ
    //  เพราะ future ตัวนี้ไม่มีทาง return Poll::Pending เลยจึงไม่ต้องพึ่ง Waker จริง)
    let waker = Waker::noop();
    let mut cx = Context::from_waker(waker);

    match Pin::new(&mut fut).poll(&mut cx) {
        Poll::Ready(value) => println!("ได้ผลลัพธ์ทันที: {value}"),
        Poll::Pending => println!("ไม่ควรเกิดกรณีนี้"),
    }
}
```

รันแล้วได้:

```
ได้ผลลัพธ์ทันที: 42
```

สังเกตว่า `Pin::new(&mut fut)` คือวิธี "pin" ค่าบน stack แบบง่ายที่สุด — ใช้ได้เพราะ `ReadyNow` เป็น type ธรรมดาที่
**ไม่มี**ปัญหา self-referential เลย (จะอธิบายว่าทำไมมันปลอดภัยในหัวข้อ 47.7) ตอนนี้แค่จำไว้ว่า "future ทุกตัวต้อง
ถูก pin ก่อนจะเรียก `.poll()` ได้" — และนี่คือสิ่งที่ `.await` ทำให้อัตโนมัติเบื้องหลังใน Part 46 นั่นเอง

### 47.3 `Poll<T>`: พลังของ Enum แทน Blocking หรือ Exception

นิยามจริงของ `Poll<T>` จาก `std::task::Poll` สั้นมาก:

```rust
pub enum Poll<T> {
    Ready(T),
    Pending,
}
```

สองตัวเลือกนี้แทนความหมายตรงไปตรงมา: **`Ready(T)`** คือ "เสร็จแล้ว นี่คือผลลัพธ์" และ **`Pending`** คือ "ยังไม่เสร็จ
ตอนนี้ ลองใหม่ทีหลัง (แต่อย่าถามฉันตอนนี้ ฉันจะบอกเธอเองว่าถามได้เมื่อไหร่ — ผ่าน `Waker`)"

ลองเทียบกับวิธีที่ภาษา/สภาพแวดล้อมอื่นแก้ปัญหา "งานที่ยังไม่เสร็จ" แบบเดียวกัน จะเห็นว่า Rust เลือกทางที่ต่างออกไป
โดยสิ้นเชิง:

- **JavaScript (`Promise`)**: ไม่มี "ถามแล้วได้คำตอบว่ายังไม่เสร็จ" เลย — `Promise` จะ resolve เมื่อพร้อม โดยคุณ
  ผูก callback (`.then()`) ไว้รอ ไม่มีการ "poll" แบบตรง ๆ ที่โค้ดผู้ใช้ทั่วไปมองเห็น
- **C# (`Task<T>`)**: คล้ายกับ `Future` ในเชิงแนวคิด แต่ implementation จริงอิง callback/continuation ภายใน
  runtime ของ .NET ไม่ได้เปิดให้ผู้ใช้ทั่วไปเห็นกลไก "poll" ตรง ๆ เหมือน Rust
- **Go**: ไม่มีแนวคิด future/promise ในภาษาเลย ใช้ goroutine + channel แทน (ความแตกต่างเชิงปรัชญาที่ Part 37-40
  เคยพูดถึง)

สิ่งที่ Rust เลือกทำคือ**เปิดกลไก poll ให้เห็นตรง ๆ ในระดับ type system** และแทนสถานะ "ยังไม่เสร็จ" ด้วย **enum
variant** ไม่ใช่:

- **การ block thread** (แบบ `.join()` ของ `JoinHandle` ใน Part 37) — เพราะ block thread ทิ้งไปเฉย ๆ ระหว่างรอ
  คือการเสียทรัพยากรมหาศาลถ้าต้องรอ future นับพันตัวพร้อมกัน (นี่คือเหตุผลทั้งหมดที่ async มีอยู่ — จะอธิบายเต็มใน
  หัวข้อ 47.12)
- **exception/panic** — เพราะ "ยังไม่เสร็จ" ไม่ใช่ข้อผิดพลาดเลย มันเป็นสถานะปกติที่คาดหวังไว้แล้ว
- **sentinel value** เช่น คืน `-1` หรือ `null` — ปัญหาแบบเดียวกับที่ Part 11 อธิบายไว้ตอนสอน `Option<T>`: ค่า
  sentinel ลืมเช็คได้ ผสมกับค่าจริงได้ ทำให้เกิดบั๊กเงียบ ๆ

**นี่คือปรัชญาการออกแบบเดียวกันเป๊ะกับ `Option<T>`** (Part 11) และ `Iterator::next() -> Option<Self::Item>`
(Part 25): ใช้ enum ที่ compiler บังคับให้ `match` ครบทุก variant แทนการพึ่งพาวินัยของโปรแกรมเมอร์ที่ต้องจำเองว่า
"เช็ค flag นี้ก่อนใช้ค่า" `Poll::Pending` ทำหน้าที่แบบเดียวกับ `None` — บอกความหมาย "ยังไม่มีคำตอบ" ได้อย่างปลอดภัย
โดยไม่ต้องพึ่ง null หรือ exception เลยแม้แต่นิดเดียว

ตารางสรุปการเปรียบเทียบทั้งสามคู่ที่ใช้ปรัชญาเดียวกัน:

| Trait | Method หลัก | ค่าที่คืน | ความหมายของแต่ละ variant |
|---|---|---|---|
| `Iterator` | `next(&mut self)` | `Option<Self::Item>` | `Some(x)` = มีค่าถัดไป, `None` = หมดแล้ว |
| `Future` | `poll(Pin<&mut Self>, &mut Context)` | `Poll<Self::Output>` | `Ready(x)` = เสร็จแล้ว, `Pending` = ยังไม่เสร็จ |
| (Part 12) `Result` | (ทุกฟังก์ชันที่ทำงานได้/ไม่ได้) | `Result<T, E>` | `Ok(x)` = สำเร็จ, `Err(e)` = ล้มเหลว |

### 47.4 เขียน Future มือแรก: `PollCounter` (นับว่า Poll ไปกี่ครั้งแล้วถึงจะพร้อม)

ทีนี้มาเขียน future ที่มีสถานะภายในจริง ๆ — ไม่พร้อมตั้งแต่ poll แรก แต่พร้อมหลัง poll ครบจำนวนครั้งที่กำหนด:

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

struct PollCounter {
    label: &'static str,
    count: u32,
    target: u32,
}

impl PollCounter {
    fn new(label: &'static str, target: u32) -> Self {
        PollCounter { label, count: 0, target }
    }
}

impl Future for PollCounter {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        self.count += 1;
        println!("[{}] poll ครั้งที่ {}/{}", self.label, self.count, self.target);
        if self.count >= self.target {
            Poll::Ready(())
        } else {
            // สำคัญมาก: ต้องบอก executor ว่า "poll ฉันใหม่ได้ทันที" ไม่อย่างนั้นจะไม่มีใคร poll ฉันอีกเลย
            // (จะอธิบายเหตุผลเต็มในหัวข้อ 47.5 — ตอนนี้แค่สังเกตว่ามันมีบรรทัดนี้อยู่)
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}

fn main() {
    use std::task::Waker;

    let mut fut = PollCounter::new("นับรอบ", 4);
    let waker = Waker::noop();
    let mut cx = Context::from_waker(waker);

    loop {
        match Pin::new(&mut fut).poll(&mut cx) {
            Poll::Ready(()) => {
                println!("เสร็จแล้ว!");
                break;
            }
            Poll::Pending => {
                println!("  -> ยังไม่พร้อม ต้อง poll ใหม่");
            }
        }
    }
}
```

รันแล้วได้ผลลัพธ์นี้จริง (ยืนยันด้วยการ compile และรันจริงแล้ว):

```
[นับรอบ] poll ครั้งที่ 1/4
  -> ยังไม่พร้อม ต้อง poll ใหม่
[นับรอบ] poll ครั้งที่ 2/4
  -> ยังไม่พร้อม ต้อง poll ใหม่
[นับรอบ] poll ครั้งที่ 3/4
  -> ยังไม่พร้อม ต้อง poll ใหม่
[นับรอบ] poll ครั้งที่ 4/4
เสร็จแล้ว!
```

สังเกตว่า `main()` ข้างบนคือ**executor แบบง่ายที่สุดที่เป็นไปได้**: ลูป `loop { match ... .poll(...) }` นี่แหละคือ
สิ่งที่ `.await` แปลงโค้ดให้เป็นอัตโนมัติใน Part 46 — ทุกครั้งที่เขียน `some_future.await` ภายในบล็อก `async`
compiler จะสร้างโค้ดที่มีความหมายเทียบเท่ากับ "poll ค่านี้ ถ้า `Ready` ก็ใช้ค่าต่อได้ ถ้า `Pending` ก็หยุด state
machine ตัวเองไว้ตรงนี้ก่อน แล้วคืน `Poll::Pending` ของตัวเองออกไปให้ executor ชั้นบนรับรู้ต่อ" — เราแค่เขียนลูป
นั้นด้วยมือให้เห็นชัด ๆ

แต่สังเกตให้ดี: `PollCounter` ไม่ได้ "รอ" อะไรจากภายนอกจริง ๆ เลย มันแค่นับ ไม่มีเหตุผลอะไรที่ "ต้องรอ" ระหว่าง
poll ครั้งที่ 1 กับครั้งที่ 2 — เราจึงต้องเรียก `cx.waker().wake_by_ref()` เองทุกครั้งที่ยังไม่พร้อม เพื่อบอกทันทีว่า
"poll ฉันใหม่ได้เลย ไม่ต้องรอ" ถ้าลืมเรียกบรรทัดนี้ ลูป `main()` ข้างบนจะยัง poll ต่อได้เพราะมันเป็น busy-loop ที่
poll วนไม่หยุดอยู่แล้ว (ไม่สนใจ wake เลย) แต่ปัญหาจะเห็นชัดทันทีเมื่อเราสร้าง**executor จริงที่ไม่ busy-loop**
(หัวข้อ 47.8) — ถ้า future ไม่เรียก wake จะไม่มีใคร poll มันอีกเลยตลอดไป (ดูกับดักข้อ 4 ท้ายบท ที่พิสูจน์เรื่องนี้
ด้วยการรันจริงจนโปรแกรมค้าง)

รูปแบบ "เรียก `wake_by_ref()` ทันทีแล้วคืน `Pending`" แบบนี้เรียกว่า **busy-polling ที่ทำผ่าน Waker** — มันยังคง
เปลืองซีพียูอยู่ (executor จะ poll วนซ้ำ ๆ เร็วมากโดยไม่มีการรอจริง) เพียงแต่**ไม่บล็อก thread ทิ้งไปเฉย ๆ** ต่างจาก
`thread::sleep` มันใช้ได้ดีกับงานที่ "เสร็จเร็วจริง ๆ ไม่ต้องรอเวลา" อย่าง `PollCounter` แต่ **ไม่ใช่ทางออกที่ถูกต้อง
สำหรับงานที่ต้องรอเวลาจริง** — หัวข้อถัดไปจะอธิบายว่าทำไม

### 47.5 ทำไมต้องมี `Waker`: ปัญหา Busy-Poll และคำตอบของมัน

ลองคิดสถานการณ์นี้: เราต้องเขียน future ที่ "พร้อมหลังจากเวลาผ่านไป 5 วินาที" (เหมือน `tokio::time::sleep` ที่จะ
เจอใน Part 48) แล้วมี executor ที่ดูแล future แบบนี้อยู่ 10,000 ตัวพร้อมกัน (สถานการณ์ปกติมากสำหรับ web server ที่
รับ connection พร้อมกันหลายพันตัว) มีสองทางเลือกกว้าง ๆ:

**ทางเลือกที่ 1 — Busy-poll ทุกตัวไม่หยุด**: executor วนลูปเรียก `.poll()` ทุก future ทั้ง 10,000 ตัวไม่หยุดเลย
ไม่ว่า future นั้นจะมีทางพร้อมเร็ว ๆ นี้หรือไม่ วิธีนี้**ทำงานได้ถูกต้อง** (จะได้ผลลัพธ์ที่ถูกต้องเสมอในที่สุด) แต่
**สิ้นเปลืองซีพียูอย่างมหาศาล** — เรียก `.poll()` future ที่ยังต้องรออีก 4.9 วินาทีนับล้านครั้งโดยไม่มีประโยชน์
อะไรเลย ซีพียูจะร้อนจี๋ทำงาน 100% ตลอดเวลาทั้งที่ไม่มีงานจริงให้ทำ

**ทางเลือกที่ 2 — executor "รู้" ว่าเมื่อไหร่ควร poll ใหม่**: นี่คือทางที่ Rust เลือก และคือหน้าที่ทั้งหมดของ
`Waker` — **สัญญา (contract)** ของ trait `Future` คือ: "เมื่อ future คืน `Poll::Pending` มันมีหน้าที่**ต้อง**จัด
การให้มีใครสักคนเรียก `cx.waker().wake()` (หรือ `wake_by_ref()`) ในอนาคต **ทันทีที่**มันมีโอกาสจะคืบหน้าต่อได้"
ตัวอย่างของ "ใครสักคน" ในสถานการณ์จริง:

- OS timer callback (เมื่อครบเวลาที่ตั้งไว้)
- background thread ที่ทำงานตามที่ future ขอไว้แล้วค่อยแจ้งกลับ (แบบที่จะเขียนในหัวข้อ 47.6)
- OS I/O event notification เช่น epoll/io_uring บอกว่า "socket นี้มีข้อมูลให้อ่านแล้ว" (จะเจอเต็มรูปแบบตอน Tokio
  ใน Part 48-49)
- อีก future หนึ่งที่ future นี้กำลังรออยู่ (แบบ `Ticker` ที่จะเขียนในหัวข้อ 47.9)

เมื่อ executor มีกลไกนี้ มันไม่ต้อง poll future ที่ยัง pending อยู่**เลย**จนกว่า waker ของ future นั้นจะถูกเรียก —
executor จึงนอนรอเฉย ๆ (ไม่กินซีพียู) จนกว่าจะมีอะไรสักอย่างพร้อมคืบหน้าจริง ๆ แล้วค่อย poll เฉพาะตัวที่ถูก wake
เท่านั้น (ไม่ใช่ poll ทุกตัวซ้ำ) นี่คือกลไกที่ทำให้ระบบ async รองรับ connection ได้หลักหมื่นถึงหลักล้านพร้อมกันบน
ทรัพยากรจำกัดได้จริง (จะเห็นตัวเลขที่ชี้ให้เห็นชัดในหัวข้อ 47.12)

**`Context<'_>`** ที่ `poll()` รับเข้ามาคือ "กล่องห่อ" ของ `Waker` — ตอนนี้มันมี field เดียวคือ waker (เหตุผลที่ห่อ
เป็น struct แยกคือเผื่ออนาคตจะเพิ่มข้อมูลอื่นเข้ามาได้โดยไม่ทำลาย signature เดิม — คล้ายกับที่ Part 22 อธิบายเรื่อง
API ที่ต้อง evolve ได้) เรียก `cx.waker()` เพื่อได้ `&Waker` แล้วเรียก `.wake()` (กิน ownership, ใช้ครั้งเดียว) หรือ
`.wake_by_ref()` (ยืม ใช้ซ้ำได้) เพื่อ "บอก executor ว่าฉันคืบหน้าได้แล้ว มา poll ฉันใหม่เถอะ"

### 47.6 Future ที่ใช้ Waker จริง: `Delay` และ Executor แบบเดียว-Future

มาเขียน future ที่**รอเหตุการณ์จากภายนอกจริง** — ใช้ `thread::spawn` (Part 37) สร้าง background thread ที่
`thread::sleep` แล้วค่อยเรียก `waker.wake()` ตอนตื่น:

```rust
use std::future::Future;
use std::pin::Pin;
use std::sync::{Arc, Mutex};
use std::task::{Context, Poll, Waker};
use std::thread;
use std::time::Duration;

/// สถานะที่ future กับ background thread ต้องแชร์กัน (ตาม pattern Arc<Mutex<T>> จาก Part 39)
struct SharedState {
    completed: bool,
    waker: Option<Waker>,
}

/// Future ที่ "พร้อม" หลังจากเวลาที่กำหนดผ่านไปจริง
/// (เทียบเท่ากับ tokio::time::sleep แบบง่ายที่สุดที่เขียนเองได้ทั้งหมด)
struct Delay {
    shared_state: Arc<Mutex<SharedState>>,
}

impl Delay {
    fn new(duration: Duration) -> Self {
        let shared_state = Arc::new(Mutex::new(SharedState {
            completed: false,
            waker: None,
        }));

        let thread_shared_state = Arc::clone(&shared_state);
        thread::spawn(move || {
            thread::sleep(duration);
            let mut state = thread_shared_state.lock().unwrap();
            state.completed = true;
            if let Some(waker) = state.waker.take() {
                waker.wake(); // <-- นี่คือหัวใจของทั้งหัวข้อ: บอก executor ว่า "poll ฉันใหม่ได้แล้ว"
            }
        });

        Delay { shared_state }
    }
}

impl Future for Delay {
    type Output = ();

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let mut state = self.shared_state.lock().unwrap();
        if state.completed {
            Poll::Ready(())
        } else {
            // ต้อง .clone() เก็บไว้เสมอ ห้ามเก็บ &Waker ตรง ๆ (ดูเหตุผลเต็มในกับดักข้อ 3 ท้ายบท)
            state.waker = Some(cx.waker().clone());
            Poll::Pending
        }
    }
}
```

**อ่านทีละส่วน:**

- **`SharedState`** ถูกห่อด้วย `Arc<Mutex<T>>` เพราะมันถูกเข้าถึงจากสอง thread จริง ๆ: thread หลักที่เรียก
  `.poll()` และ background thread ที่จะมาเขียนทับ `completed = true` เมื่อครบเวลา — นี่คือ pattern
  `Arc<Mutex<T>>` แบบเดียวกับ Part 39 ทุกประการ ไม่มีอะไรใหม่ในส่วนนี้เลย
- **`Delay::new()`** สร้าง `SharedState` เริ่มต้น (`completed: false`) แล้ว `thread::spawn` background thread
  ที่ `sleep` ตามเวลาที่กำหนด จากนั้นตั้ง `completed = true` และ**ถ้ามี** `Waker` ถูกเก็บไว้แล้ว (แปลว่า `poll()`
  ถูกเรียกไปแล้วอย่างน้อยหนึ่งครั้งตอนที่ยังไม่พร้อม) ก็เรียก `.wake()` ทันที
- **`poll()`**: ถ้า `completed` เป็น `true` แล้ว คืน `Poll::Ready(())` ทันที ถ้ายังไม่ครบเวลา **เก็บ Waker ของการ
  เรียกครั้งนี้ไว้** (แทนที่ตัวเก่าถ้ามี — เผื่อ future นี้ถูกย้ายไป poll จาก waker คนละตัวในแต่ละรอบ ซึ่งเป็นไปได้
  จริงในระบบที่ซับซ้อน) แล้วคืน `Poll::Pending`

ทีนี้ต้องมีอะไรสักอย่างมา `.poll()` มันให้จบ — มาเขียน **executor แบบง่ายที่สุดที่รันได้แค่ future เดียว** ก่อนจะ
ไปสร้างตัวที่รันหลาย future พร้อมกันในหัวข้อ 47.8:

```rust
use std::sync::Arc;
use std::task::{Wake, Waker};
use std::thread::Thread;

/// Waker ที่ทำหน้าที่ปลุก thread ที่กำลัง park() อยู่ให้ตื่นขึ้นมา poll ใหม่
struct ThreadWaker(Thread);

impl Wake for ThreadWaker {
    fn wake(self: Arc<Self>) {
        self.0.unpark();
    }
}

/// Executor เล็กที่สุดที่เป็นไปได้: รัน future เดียวจนจบบน thread ปัจจุบัน
/// ใช้ thread::park/unpark เป็นกลไก "นอนรอจนกว่าจะถูก wake" แทนการ busy-loop
fn block_on<F: Future + Unpin>(mut future: F) -> F::Output {
    let thread = thread::current();
    let waker = Waker::from(Arc::new(ThreadWaker(thread)));
    let mut cx = Context::from_waker(&waker);

    loop {
        match Pin::new(&mut future).poll(&mut cx) {
            Poll::Ready(value) => return value,
            Poll::Pending => thread::park(), // <-- นอนรอจริง ไม่กินซีพียูเลย จนกว่า wake() จะถูกเรียก
        }
    }
}
```

จุดที่สำคัญที่สุดคือ `std::task::Wake` — trait นี้ (ต่างจาก `Send`/`Sync`/`Unpin`) **ไม่ใช่** auto trait ที่
compiler แจกให้ฟรี มันคือ trait ปกติที่**เราต้อง implement เอง** เพียงแค่ method เดียวคือ `wake(self: Arc<Self>)`
เมื่อ implement `Wake` ให้ type ไหนแล้ว std จะให้ `From<Arc<W>> for Waker` มาแบบ blanket implementation อัตโนมัติ
(ตามหลักการ blanket impl ที่ Part 21 สอนไว้) ทำให้เรียก `Waker::from(Arc::new(ThreadWaker(...)))` เพื่อได้ `Waker`
จริงมาใช้ได้ทันที — นี่คือวิธีสร้าง `Waker` ของตัวเองที่ปลอดภัย (ไม่ต้องแตะ `unsafe`/`RawWaker` เลย ซึ่งเป็นวิธีเดิม
ก่อน Rust 1.51 ที่ซับซ้อนกว่านี้มาก)

ลองรันจริง:

```rust
use std::time::Instant;

fn main() {
    let start = Instant::now();
    println!("เริ่มรอ...");
    block_on(Delay::new(Duration::from_millis(300)));
    println!("รอเสร็จแล้ว หลังจากผ่านไป {:?}", start.elapsed());
}
```

ผลลัพธ์จริง (compile และรันจริงแล้ว):

```
เริ่มรอ...
รอเสร็จแล้ว หลังจากผ่านไป 300.405112ms
```

สังเกตว่าเวลาที่ผ่านไปตรงกับ `300ms` ที่ตั้งไว้พอดี (ไม่ใช่ 0ms แบบ busy-loop และไม่ใช่รอตลอดกาลแบบลืม wake)
พิสูจน์ว่ากลไกทั้งหมดทำงานถูกต้อง: `main()` thread เรียก `block_on` ซึ่ง poll ครั้งแรกได้ `Pending` แล้ว
`thread::park()` (นอนหลับจริง ไม่กินซีพียู — ลองเปิด monitor ซีพียูตอนรันดูได้ จะเห็นว่าไม่มีการใช้งานพุ่งสูงเลย
ระหว่างที่รอ) จนกระทั่ง background thread ตื่นจาก `sleep(300ms)` แล้วเรียก `waker.wake()` ซึ่งไปเรียก
`thread.unpark()` ปลุก `main()` thread ให้ poll ใหม่อีกครั้ง คราวนี้ได้ `Poll::Ready(())` จบการทำงาน

นี่คือคำตอบเต็มรูปแบบของคำถามที่ Part 46 ทีเซอร์ไว้ว่า "executor คือตัวขับเคลื่อน Future ให้ทำงาน" — `block_on`
ข้างบนคือ executor ตัวจริงตัวหนึ่ง (แม้จะรันได้แค่ future เดียว) ที่เขียนด้วยมือทั้งหมด ไม่มีเวทมนตร์ซ่อนอยู่เลย

### 47.7 `Pin<T>`: เกราะป้องกันที่ Future ต้องการจริง ๆ

ตอนนี้ถึงเวลาไขปริศนาที่ค้างมาตั้งแต่ Part 23 และถูกทีเซอร์ซ้ำใน Part 46: **ทำไม `Future::poll` ต้องรับ
`self: Pin<&mut Self>` แทน `&mut self` ธรรมดา?**

#### ทวนปัญหาจาก Part 23: Self-Referential Struct

Part 23 เกริ่นไว้ว่ามีปัญหาคลาสหนึ่งที่ lifetime ธรรมดาแก้ไม่ได้ — **self-referential struct**: struct ที่มี field
หนึ่งเก็บ reference ชี้ไปยัง field อีกตัว**ในโครงสร้างเดียวกัน** ลองดูตัวอย่างที่ทำให้เห็นภาพ (นี่คือ pseudocode
อธิบายแนวคิด ไม่ใช่โค้ดที่ compile ได้ตรง ๆ เพราะ Rust ธรรมดาไม่มีทางเขียนแบบนี้ได้เลยด้วยเหตุผลที่กำลังจะอธิบาย):

```
// pseudocode — Rust ปกติเขียนแบบนี้ไม่ได้! แสดงไว้เพื่ออธิบายแนวคิดเท่านั้น
struct SelfReferential {
    data: String,
    pointer_into_data: &??? str,  // ชี้เข้าไปใน data ของตัวเอง
}
```

ปัญหาคือ: ถ้า `SelfReferential` ถูก**ย้าย** (move) ไปอยู่ที่หน่วยความจำตำแหน่งใหม่ (เช่น ย้ายจาก stack frame หนึ่ง
ไปอีกที่หนึ่งตอน return ค่าออกจากฟังก์ชัน หรือย้ายเข้า `Vec` ตอน push) **`data` จะถูกย้ายไปที่ address ใหม่ด้วย**
แต่ `pointer_into_data` (ถ้ามันมีอยู่จริงได้) จะยังชี้ไปที่ address**เก่า**อยู่ — กลายเป็น dangling pointer ทันที
เพราะ Rust **ไม่มี**กลไก "อัปเดต pointer ภายในให้ตามการย้าย" แบบอัตโนมัติ (ต่างจาก garbage collector บางตัวใน
ภาษาอื่นที่ทำแบบนี้ได้ เรียกว่า "moving GC")

Part 23 บอกไว้แค่ว่า "lifetime ธรรมดาแก้ปัญหานี้ไม่ได้ ... โลกจริงแก้ปัญหานี้ด้วยเทคนิคขั้นสูงกว่านั้น" — คำตอบคือ
`Pin<T>` และตอนนี้เราจะเห็นว่ามันเกี่ยวข้องกับ `Future` อย่างไร

#### ทำไม Future ที่ Compiler สร้างขึ้นถึงเป็น Self-Referential

กลับไปดูโค้ด async จาก Part 46:

```rust
async fn process() {
    let data = String::from("ข้อมูลสำคัญ");
    let slice: &str = &data[0..4];       // ยืม reference จาก data
    some_delay().await;                   // <-- จุด await: state machine อาจหยุดพักตรงนี้
    println!("{slice}");                  // ใช้ slice ต่อ "หลัง" จุด await
}
```

`slice` ถูกยืมมาจาก `data` **ก่อน** จุด `.await` แต่ถูกใช้งานต่อ**หลัง**จุด `.await` — เพื่อให้โค้ดแบบนี้ทำงานได้
compiler ต้องสร้าง state machine ที่**เก็บทั้ง `data` และ `slice` ไว้ในโครงสร้างเดียวกัน** ให้รอดจากจุดพักได้ (ตาม
โมเดล state machine ที่ Part 46 สอนไว้) ลองนึกภาพ (อีกครั้ง — นี่คือ pseudocode อธิบายสิ่งที่ compiler สร้างขึ้น
เบื้องหลังโดยประมาณ ไม่ใช่โค้ดจริงที่เขียนเองได้):

```
// pseudocode: state machine ที่ compiler generate ให้ process() โดยประมาณ
enum ProcessStateMachine {
    Start,
    WaitingOnDelay {
        data: String,
        slice: &??? str,   // ชี้เข้าไปใน `data` ที่อยู่ใน enum ตัวเดียวกันนี้!
        delay: SomeDelayFuture,
    },
    Done,
}
```

**นี่แหละคือ self-referential struct ตัวจริงที่ Part 23 เกริ่นไว้** — เกิดขึ้น**อัตโนมัติ**ทุกครั้งที่มีการยืม
reference จากตัวแปรท้องถิ่นข้าม (across) จุด `.await` และ future ที่เกิดจาก async fn ทุกตัวมีโอกาสมีรูปร่างแบบนี้
ได้เสมอ ถ้า `WaitingOnDelay` ตัวนี้ถูกย้าย (เช่น ย้ายจาก stack ของฟังก์ชันหนึ่งไปเก็บใน `Box` เพื่อส่งให้ executor)
`slice` จะกลายเป็น dangling pointer ทันทีเหมือนตัวอย่าง pseudocode ก่อนหน้า — **เว้นแต่จะมีกลไกบางอย่างการันตีว่า
มันจะไม่ถูกย้ายอีกเลยตั้งแต่ตอนที่เริ่ม poll**

#### คำตอบ: `Pin<P>`

`Pin<P>` คือ wrapper รอบ pointer (`&mut T`, `Box<T>`, ฯลฯ) ที่ให้การันตีนี้: **ถ้า `T` ไม่ใช่ `Unpin` ค่าที่ `P`
ชี้ไปจะไม่ถูกย้ายในหน่วยความจำอีกเลยตราบใดที่มันยังถูก pin อยู่** เพราะ API ของ `Pin` **ไม่มี**ทางที่จะเอาค่า
`T` แบบ by-value ออกมาได้อีก (ยกเว้นผ่าน `unsafe` ที่โปรแกรมเมอร์ต้องรับประกันด้วยตัวเองว่าปลอดภัยจริง) นี่คือ
เหตุผลที่ `Future::poll` ต้องรับ `self: Pin<&mut Self>`: **มันคือการันตีจาก compiler ว่า "ตั้งแต่ poll ครั้งแรก
เป็นต้นไป future ตัวนี้จะไม่ถูกย้ายที่อีกเลยตลอดชีวิตของมัน"** — ซึ่งเป็นการันตีที่ state machine self-referential
จากตัวอย่างข้างบนต้องการพอดี

**จุดที่มือใหม่มักสับสน**: การสร้าง future ขึ้นมาครั้งแรก (`let fut = process();`) **ยังไม่มี**การ
self-reference เกิดขึ้นเลย — self-reference จะถูก "สร้างขึ้นจริง" ก็ตอนที่ poll ไปถึงจุด `.await` แล้วเข้าสู่ state
`WaitingOnDelay` เท่านั้น เพราะฉะนั้น**การย้าย future ที่ยังไม่เคยถูก poll สักครั้งเลยยังปลอดภัยอยู่** (เช่น
`Box::pin(fut)` ย้าย `fut` เข้า heap ครั้งเดียวก่อน poll ครั้งแรก — ปลอดภัยเสมอ) แต่**ห้ามย้ายอีกเลย**หลังจาก
poll ไปแล้วอย่างน้อยหนึ่งครั้ง — นี่คือเหตุผลที่ `poll()` ทุก call ถูกบังคับให้รับผ่าน `Pin` เพื่อปิดประตูไม่ให้
เผลอย้ายได้อีกตั้งแต่ต้น

#### `Unpin`: Auto Trait ตัวที่สาม

Part 40 หัวข้อ 40.2 เกริ่นไว้ว่า `Unpin` เป็น auto trait ตัวที่สามคู่กับ `Send`/`Sync` — ตอนนี้เราพร้อมเข้าใจมันเต็ม
รูปแบบแล้ว โดยใช้หลักการ auto trait แบบเดียวกันเป๊ะกับที่ Part 40 สอน `Send`/`Sync`:

1. **`Unpin` เป็น marker trait** — ไม่มี method เลย เหมือน `Send`/`Sync`
2. **compiler แจกให้อัตโนมัติแบบ structural/recursive** — type ส่วนใหญ่**เป็น** `Unpin` โดยอัตโนมัติ (`i32`,
   `String`, `Vec<T>`, และ struct/enum ที่ประกอบจาก field ที่เป็น `Unpin` ทั้งหมด) เพราะการย้ายมันไปมาใน
   หน่วยความจำ**ไม่มีปัญหาอะไรเลย** ไม่มี field ไหนชี้กลับเข้าไปในตัวมันเอง
3. **type ที่ opt-out ได้ด้วยการมี field พิเศษ** — `std::marker::PhantomPinned` คือ marker field ที่ถ้าใส่เข้าไป
   ใน struct จะทำให้ struct นั้น**ไม่**เป็น `Unpin` (คือ `!Unpin`) ทันที เป็นสัญญาณบอกว่า "struct นี้อาจมีการ
   self-reference เกิดขึ้นได้ ห้ามย้ายมันหลัง pin แล้ว"

**ข่าวดี**: `PollCounter`, `Delay` และ future อื่น ๆ ที่เขียนไปแล้วในบทนี้ทั้งหมดเป็น `Unpin` โดยอัตโนมัติ เพราะไม่มี
field ไหน self-referential เลย — นี่คือเหตุผลที่เราเรียก `Pin::new(&mut fut)` ได้ตรง ๆ แบบง่าย ๆ มาตลอด (ปลอดภัย
เพราะ `Unpin` แปลว่า "ย้ายได้ตามปกติแม้จะถูก pin อยู่ก็ตาม ไม่มีอะไรพัง") **future ที่เกิดจาก `async fn`/`async
block` ต่างหากที่มักจะเป็น `!Unpin`** เมื่อมีการยืม reference ข้ามจุด `.await` แบบตัวอย่าง `process()` ข้างบน — และ
นี่คือเหตุผลที่ `.await` ใน Part 46 ต้อง `Box::pin` หรือ pin บน stack ให้ก่อนเสมอ (สิ่งที่ compiler ทำอัตโนมัติให้
เมื่อคุณเขียน `.await` โดยที่คุณไม่ต้องเห็นเลย)

ลองดูตัวอย่างคลาสสิกที่แสดงการันตีของ `Pin` ให้เห็นเป็นรูปเป็นร่างจริง — struct ที่**ตั้งใจ**เป็น self-referential
โดยใช้ raw pointer (ตัวอย่างมาตรฐานจากเอกสารของ `std::pin` เอง):

```rust
use std::marker::PhantomPinned;
use std::pin::Pin;
use std::ptr::NonNull;

struct Unmovable {
    data: String,
    slice: NonNull<String>, // ชี้กลับไปที่ data ของตัวเอง
    _pin: PhantomPinned,     // ทำให้ Unmovable เป็น !Unpin
}

impl Unmovable {
    fn new(data: String) -> Pin<Box<Self>> {
        let unmovable = Unmovable {
            data,
            slice: NonNull::dangling(),
            _pin: PhantomPinned,
        };
        let mut boxed = Box::pin(unmovable);

        let self_ptr: NonNull<String> = NonNull::from(&boxed.data);
        // ต้องใช้ unsafe เพราะกำลังแก้ field ภายในของสิ่งที่ pin ไปแล้ว
        // แต่ปลอดภัยจริง เพราะเราไม่ได้ "ย้าย" ตัว struct เลย แค่แก้ field ข้างในที่ตำแหน่งเดิม
        unsafe {
            let mut_ref: Pin<&mut Self> = Pin::as_mut(&mut boxed);
            Pin::get_unchecked_mut(mut_ref).slice = self_ptr;
        }
        boxed
    }
}

fn main() {
    let unmovable = Unmovable::new("อยู่ตรงนี้ตลอด".to_string());
    println!("data = {}", unmovable.data);
    println!(
        "slice ชี้กลับไปที่ data ตัวเอง: {}",
        unsafe { unmovable.slice.as_ref() }
    );
    // unmovable มี type เป็น Pin<Box<Unmovable>> — ไม่มีทางดึง Unmovable ออกมาแบบ by-value ได้เลย
    // เพราะ Unmovable ไม่ implement Unpin (มี PhantomPinned) และ Pin ไม่มี safe method
    // ที่คืนค่าแบบ by-value ให้เรียกได้ตรง ๆ นี่คือการันตีที่ `slice` ต้องการพอดี
}
```

รันแล้วได้ (ยืนยันด้วยการ compile และรันจริงแล้ว):

```
data = อยู่ตรงนี้ตลอด
slice ชี้กลับไปที่ data ตัวเอง: อยู่ตรงนี้ตลอด
```

จุดสำคัญ: การสร้าง `Unmovable` ต้องผ่าน `Box::pin` **ก่อน**ที่จะตั้งค่า `slice` ให้ชี้กลับเข้าไปในตัวเอง (สังเกตว่า
สร้างด้วย `NonNull::dangling()` ก่อน แล้วค่อยแก้เป็นค่าจริงหลัง pin แล้ว) — ลำดับนี้สำคัญมาก: **สร้างครั้งแรกยังไม่
self-referential (ย้ายได้ตามปกติ) พอ pin แล้วจึงค่อยสร้าง self-reference ขึ้น (ห้ามย้ายอีกนับจากจุดนี้)** ตรงกับ
หลักการที่อธิบายไว้ข้างบนทุกประการ — future ที่เกิดจาก `async fn` ก็ทำงานตามแบบแผนเดียวกันนี้ เพียงแต่ compiler
เป็นคนเขียน `unsafe` ส่วนนี้ให้เราแทน ไม่ต้องเขียนเองเลย

`Pin` ยังมีรายละเอียดขั้น unsafe อีกมาก (เช่น `pin_project` crate ที่ใช้เขียน combinator ของ future เอง) ซึ่งเกิน
ขอบเขตของบทนี้ — เป้าหมายของหัวข้อนี้คือให้**จำได้ทันทีที่เห็น** `Pin<&mut Self>` ในลายเซ็นฟังก์ชันหรือ
error message ว่ามันกำลังพูดถึงการันตี "จะไม่ถูกย้ายที่" และรู้ว่า `Unpin` คือสิ่งที่บอกว่า type หนึ่งไม่ต้องสนใจ
การันตีนี้เลยเพราะมันปลอดภัยที่จะย้ายอยู่แล้วโดยธรรมชาติ — ระดับความเข้าใจนี้เพียงพอสำหรับการเขียนโค้ด async ใน
โปรเจกต์จริงเกือบทั้งหมด (ดูกับดักข้อ 1-2 ท้ายบทสำหรับ error message จริงที่จะเจอบ่อยที่สุดเมื่อทำงานกับ `Pin`)

### 47.8 การสร้าง Executor ของตัวเอง: หัวใจของ Async Runtime

`block_on` ในหัวข้อ 47.6 รันได้แค่ future เดียว ทีนี้มาสร้าง**executor เต็มรูปแบบ**ที่รันหลาย future (เรียกว่า
"task" เมื่ออยู่ใน executor) พร้อมกันได้ — นี่คือสิ่งที่ Tokio (Part 48) ทำในระดับที่ซับซ้อนกว่านี้มาก แต่หลักการ
พื้นฐานเหมือนกันทุกประการ

**สิ่งที่ executor ต้องทำจริง ๆ มีแค่สามอย่าง**: (1) เก็บชุดของ task ที่ยังทำงานไม่จบไว้ในคิว, (2) หยิบ task ที่
"พร้อมจะถูก poll" ออกมา poll แล้วดูผลลัพธ์ — ถ้า `Pending` ก็เก็บกลับเข้าไปรอ (แต่**ไม่วนกลับมา poll ทันที**
ต่างจาก busy-loop), และ (3) ทำให้ทุก task มี `Waker` ของตัวเองที่เมื่อถูกเรียก จะส่ง task นั้นกลับเข้าคิว
"พร้อมจะถูก poll" อีกครั้ง — เมื่อไม่มี task ไหนถูก wake เลย executor ก็ไม่ต้องทำอะไรเลย (นอนรอเฉย ๆ)

```rust
use std::future::Future;
use std::pin::Pin;
use std::sync::mpsc::{sync_channel, Receiver, SyncSender};
use std::sync::{Arc, Mutex};
use std::task::{Context, Poll, Wake, Waker};

type BoxFuture = Pin<Box<dyn Future<Output = ()> + Send>>;

/// หนึ่ง task คือหนึ่ง future ที่ถูก spawn เข้า executor พร้อมกับช่องทางส่งตัวเองกลับเข้าคิว
struct Task {
    future: Mutex<Option<BoxFuture>>,
    task_sender: SyncSender<Arc<Task>>,
}

/// Task เป็น Waker ของตัวเอง! เมื่อถูก wake มันแค่ส่งตัวเองกลับเข้าคิว "พร้อม poll"
impl Wake for Task {
    fn wake(self: Arc<Self>) {
        self.task_sender
            .send(self.clone())
            .expect("คิวงานเต็มหรือถูกปิดไปแล้ว");
    }
}

#[derive(Clone)]
struct Spawner {
    task_sender: SyncSender<Arc<Task>>,
}

impl Spawner {
    fn spawn(&self, future: impl Future<Output = ()> + Send + 'static) {
        let task = Arc::new(Task {
            future: Mutex::new(Some(Box::pin(future))),
            task_sender: self.task_sender.clone(),
        });
        self.task_sender.send(task).expect("คิวงานเต็มหรือถูกปิดไปแล้ว");
    }
}

struct Executor {
    ready_queue: Receiver<Arc<Task>>,
}

impl Executor {
    fn run(&self) {
        // recv() บล็อกจนกว่าจะมี task ใหม่เข้าคิว "พร้อม poll" — ไม่ใช่ busy-loop
        while let Ok(task) = self.ready_queue.recv() {
            let mut future_slot = task.future.lock().unwrap();
            if let Some(mut future) = future_slot.take() {
                let waker = Waker::from(Arc::clone(&task));
                let mut cx = Context::from_waker(&waker);
                match future.as_mut().poll(&mut cx) {
                    Poll::Ready(()) => {
                        // งานเสร็จแล้ว ไม่ต้องเก็บ future คืน — มันจะถูก drop ไปพร้อม Task
                    }
                    Poll::Pending => {
                        *future_slot = Some(future); // ยังไม่เสร็จ เก็บคืนไว้รอถูก wake
                    }
                }
            }
        }
    }
}

fn new_executor_and_spawner() -> (Executor, Spawner) {
    const MAX_QUEUED_TASKS: usize = 10_000;
    let (task_sender, ready_queue) = sync_channel(MAX_QUEUED_TASKS);
    (Executor { ready_queue }, Spawner { task_sender })
}
```

**อ่านทีละส่วนอย่างละเอียด เพราะนี่คือโค้ดที่สำคัญที่สุดของทั้งบท:**

- **`Mutex<Option<BoxFuture>>` ใน `Task`**: ทำไมต้องมี `Mutex` ทั้งที่ future ถูก poll จาก thread เดียว (thread
  ของ `Executor::run`)? เพราะ **`wake()` อาจถูกเรียกจาก thread อื่นในเวลาเดียวกัน** — เช่น `Delay`'s background
  thread เรียก `waker.wake()` ขณะที่ executor thread กำลัง poll task นั้นอยู่พอดี `Mutex` ป้องกัน data race แบบ
  เดียวกับที่ Part 39 สอนไว้ (ในที่นี้ป้องกันไม่ให้สอง thread แก้ `Option<BoxFuture>` พร้อมกัน) — สังเกตว่านี่คือ
  จุดที่ต่างจาก `block_on` ในหัวข้อ 47.6 ที่ไม่ต้องใช้ `Mutex` รอบ future เพราะรันแค่ future เดียวจาก thread เดียว
  ตลอด ไม่มีการแข่งกันแก้ไข
- **`Arc<Task>`**: `Task` ต้องถูกแชร์ระหว่างสามที่พร้อมกัน — คิวงาน (`SyncSender`), ตัว `Waker` ที่ future ถือไว้
  (อาจอยู่ใน background thread ของ `Delay`), และตัว `Executor::run` เอง `Arc` (Part 28-29) คือคำตอบมาตรฐานสำหรับ
  "หลายเจ้าของ ข้าม thread" แบบเดียวกับที่เรียนมาตลอด Part 37-40
- **`impl Wake for Task`**: นี่คือมุมที่สวยงามที่สุดของ design นี้ — **`Task` เป็น `Waker` ของตัวเอง** เมื่อถูก
  `wake()` มันไม่ต้องรู้อะไรเกี่ยวกับ future ภายในเลย แค่ส่ง `Arc<Task>` ของตัวเองกลับเข้าคิว `ready_queue` แปลว่า
  "ฉันพร้อมถูก poll อีกแล้ว" — executor ที่คอย `recv()` อยู่จะหยิบมัน poll ต่อทันที
- **`sync_channel` (bounded channel จาก Part 38)** ถูกใช้แทน `channel` (unbounded) โดยตั้งใจ — ถ้าคิวงานมี
  ขนาดจำกัด (`MAX_QUEUED_TASKS`) การ `send()` จะบล็อกถ้าคิวเต็ม กลายเป็น **backpressure** อัตโนมัติ (ป้องกันไม่ให้
  task ที่ wake ตัวเองถี่เกินไปทำให้หน่วยความจำระเบิด) — เป็นการตัดสินใจ design ที่ executor จริงต้องคิดถึงเสมอ
- **`drop(spawner)` เพื่อปิด channel**: `Receiver::recv()` คืน `Err` เมื่อทุก `Sender`/`SyncSender` ที่จับคู่กัน
  ถูก drop ไปหมดแล้ว **และ**ไม่มีข้อความเหลือในคิว — เราใช้กลไกนี้เป็นสัญญาณ "ไม่มี task ใหม่จะถูก spawn อีกแล้ว"
  เพื่อให้ `Executor::run()` คืนค่าออกมาได้เมื่อ task ทั้งหมดทำงานจบ (executor จริงอย่าง Tokio ใช้กลไกปิดที่ซับซ้อน
  กว่านี้มาก แต่หลักการพื้นฐานเรื่อง "รู้ว่าเมื่อไหร่ควรหยุด" คล้ายกัน)

### 47.9 `Ticker`: การ "ประกอบ" Future จาก Future อื่น (สิ่งที่ `.await` ทำให้จริง ๆ)

ก่อนพิสูจน์ว่า executor ข้างบนรันหลาย task พร้อมกันได้จริง เราต้องมี future ที่**น่าสนใจกว่า** `Delay` เฉย ๆ — มา
เขียน `Ticker`: future ที่ "tick" (พิมพ์ความคืบหน้า) ทุก ๆ ช่วงเวลาที่กำหนด รวมทั้งหมด `total_ticks` ครั้งแล้วจบ โดย
**ประกอบ**จาก `Delay` ที่เขียนไปแล้วในหัวข้อ 47.6:

```rust
use std::time::Duration;

struct Ticker {
    label: &'static str,
    interval: Duration,
    total_ticks: u32,
    ticks_done: u32,
    current_delay: Option<Delay>,
}

impl Ticker {
    fn new(label: &'static str, interval: Duration, total_ticks: u32) -> Self {
        Ticker {
            label,
            interval,
            total_ticks,
            ticks_done: 0,
            current_delay: None,
        }
    }
}

impl Future for Ticker {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        loop {
            if self.current_delay.is_none() {
                let interval = self.interval;
                self.current_delay = Some(Delay::new(interval));
            }

            // Pin::new ใช้ได้แบบปลอดภัยตรงนี้เพราะ Delay เป็น Unpin โดยอัตโนมัติ (หัวข้อ 47.7)
            let delay = self.current_delay.as_mut().unwrap();
            match Pin::new(delay).poll(cx) {
                Poll::Pending => return Poll::Pending,
                Poll::Ready(()) => {
                    self.current_delay = None;
                    self.ticks_done += 1;
                    println!("[{}] tick {}/{}", self.label, self.ticks_done, self.total_ticks);
                    if self.ticks_done >= self.total_ticks {
                        return Poll::Ready(());
                    }
                    // ยังไม่ครบ วนกลับไปสร้าง Delay รอบใหม่ทันที (ไม่ return Pending ระหว่างนี้)
                }
            }
        }
    }
}
```

จุดที่สำคัญที่สุดของโค้ดนี้คือ **`Pin::new(delay).poll(cx)`** — `Ticker::poll()` เรียก `poll()` ของ `Delay` ที่ตัว
มันเป็นเจ้าของโดยตรง แล้ว**ส่ง `cx` (ที่ได้รับมาจาก executor ชั้นบน) ต่อไปตรง ๆ ให้ `Delay`** นี่คือกลไกที่ทำให้
`Waker` เดินทางจาก executor ผ่าน `Ticker` ไปถึง `Delay` ได้อย่างถูกต้อง (`Delay` จะเก็บ waker ตัวนี้ไว้เอง เพื่อ
เรียก `.wake()` ตอน background thread ของมันตื่น) — **นี่คือสิ่งที่ syntax `.await` ทำให้อัตโนมัติทั้งหมด** ทุก
ครั้งที่เขียน `some_future.await` ภายใน `async fn` compiler จะแปลงมันให้เป็นโค้ดที่มีความหมายเทียบเท่ากับลูป
`match Pin::new(&mut some_future).poll(cx) { Ready(v) => v ต่อได้, Pending => return Poll::Pending }` แบบเดียวกับ
ที่ `Ticker::poll()` เขียนด้วยมือนี่เอง ถ้าเขียน `Ticker` เป็น `async fn` แบบ Part 46 จะได้โค้ดที่ทำงานเหมือนกันเป๊ะ
สั้นกว่ามาก:

```rust
// เทียบเท่ากับ Ticker ข้างบนถ้าเขียนด้วย async/await (แสดงไว้เพื่อเทียบ ไม่ใช่โค้ดที่ใช้ต่อในบทนี้)
async fn ticker_equivalent(label: &'static str, interval: Duration, total_ticks: u32) {
    for tick in 1..=total_ticks {
        Delay::new(interval).await; // <-- บรรทัดนี้แทนลูป Pin::new(...).poll(cx) ทั้งก้อนข้างบน
        println!("[{label}] tick {tick}/{total_ticks}");
    }
}
```

การเห็นทั้งสองเวอร์ชันเคียงข้างกันคือบทพิสูจน์ที่ชัดเจนที่สุดของทั้งบทนี้: **`.await` ไม่ใช่มายากล มันคือ syntax
sugar เหนือลูป `poll()` ที่เขียนด้วยมือได้ทุกตัวอักษร**

### 47.10 การ Desugar `async fn` ด้วยมือแบบเต็มรูปแบบ: จาก Pseudocode สู่โค้ดที่ Compile และรันได้จริง

หัวข้อ 47.7 แสดง pseudocode ของ state machine ที่ compiler สร้างให้ `async fn` ไว้แล้ว แต่บอกไว้ตรง ๆ ว่า "นี่คือ
pseudocode ไม่ใช่โค้ดที่เขียนเองได้" — ทีนี้มาพิสูจน์ให้เห็นว่ามัน**เขียนเองได้จริง 100%** โดยเขียน enum
state machine ตัวจริง (ไม่ใช่ pseudocode อีกต่อไป) ที่ทำงานเทียบเท่ากับ `async fn` นี้ทุกประการ:

```rust
// เป้าหมาย: เขียน enum + impl Future ที่เทียบเท่ากับ async fn นี้แบบเป๊ะ ๆ
//
// async fn greet_slowly(name: String) -> String {
//     Delay::new(Duration::from_millis(200)).await;
//     format!("สวัสดี, {name}!")
// }
```

`async fn` ตัวนี้มีจุด `.await` เดียว แปลว่า state machine ของมันต้องมีอย่างน้อยสามสถานะ: **ยังไม่เริ่ม** (มีแค่
`name` ที่รับมาจาก argument), **กำลังรอ delay** (ต้องเก็บทั้ง `name` ที่ยังต้องใช้ต่อ**และ** `delay` ที่กำลังรอ
พร้อมกัน — สังเกตว่าถ้า `name` ถูกยืมเป็น reference แทนการเก็บทั้งค่า สถานะนี้จะกลายเป็น self-referential ทันที
ตามที่อธิบายไว้ในหัวข้อ 47.7 แต่ในตัวอย่างนี้เราเก็บ `name: String` เป็นเจ้าของตรง ๆ ไม่ใช่ reference จึงไม่มีปัญหา
self-referential เกิดขึ้นเลย — เป็นตัวอย่างที่จงใจเลือกให้เข้าใจโครงสร้าง state machine ก่อน โดยยังไม่ต้องพัวพัน
กับ `unsafe`/`Pin::get_unchecked_mut`), และ **จบแล้ว**:

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::Duration;

enum GreetSlowly {
    Start { name: String },
    WaitingDelay { name: String, delay: Delay },
    Done,
}

impl GreetSlowly {
    fn new(name: String) -> Self {
        GreetSlowly::Start { name }
    }
}

impl Future for GreetSlowly {
    type Output = String;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        loop {
            // ถ้าอยู่ใน state ที่กำลังรอ delay ให้ poll มันก่อน ถ้ายัง Pending ก็คืน Pending ทันที
            if let GreetSlowly::WaitingDelay { delay, .. } = &mut *self {
                match Pin::new(delay).poll(cx) {
                    Poll::Pending => return Poll::Pending,
                    Poll::Ready(()) => {} // delay เสร็จแล้ว ตกลงไปเปลี่ยน state ข้างล่าง
                }
            }

            // ย้าย state ปัจจุบันไปเป็น Done ชั่วคราว (mem::replace) เพื่อดึงข้อมูลข้างในออกมาแบบ by-value
            match std::mem::replace(&mut *self, GreetSlowly::Done) {
                GreetSlowly::Start { name } => {
                    let delay = Delay::new(Duration::from_millis(200));
                    *self = GreetSlowly::WaitingDelay { name, delay };
                }
                GreetSlowly::WaitingDelay { name, .. } => {
                    return Poll::Ready(format!("สวัสดี, {name}!"));
                }
                GreetSlowly::Done => panic!("polled after completion"),
            }
        }
    }
}

fn main() {
    let result = block_on(GreetSlowly::new("Rustacean".to_string()));
    println!("{result}");
}
```

ผลลัพธ์จริง (compile และรันจริงแล้ว):

```
สวัสดี, Rustacean!
```

**อ่านทีละส่วนของ `poll()` อย่างละเอียด เพราะนี่คือคำตอบที่แท้จริงที่สุดของ "state machine คืออะไร":**

- **`loop { ... }` รอบนอก**: ทำไม `poll()` ต้องมีลูปทั้งที่ปกติเราคิดว่า "poll หนึ่งครั้ง = เดินหนึ่งสถานะ"? เพราะ
  เมื่อ state เปลี่ยนจาก `Start` ไป `WaitingDelay` (สร้าง `Delay` ใหม่ที่ยังไม่เคยถูก poll เลย) เราต้อง**poll
  มันทันทีในการเรียกเดียวกัน** ไม่ใช่คืน `Pending` แล้วรอ `.poll()` รอบถัดไปเปล่า ๆ (ถ้าทำแบบนั้นจะไม่มีใครมา
  poll ต่อ เพราะยังไม่มีการเรียก `cx.waker()` ที่ผูกกับ delay ตัวใหม่เลย) — compiler จริงก็ทำแบบเดียวกันนี้ทุก
  ประการ: state machine เดินหน้าไปเรื่อย ๆ ในการ `poll()` เดียวจนกว่าจะเจอจุดที่**คืน Pending จริง ๆ** (จาก
  sub-future ที่ถูก poll แล้วยังไม่พร้อม) หรือจนกว่าจะถึงจุดสิ้นสุด (`Ready`)
- **`if let GreetSlowly::WaitingDelay { delay, .. } = &mut *self`**: ยืม `delay` แบบ mutable จาก enum ปัจจุบัน
  เพื่อ poll มัน — สังเกตว่านี่คือจุดเดียวกันเป๊ะกับที่ `Ticker::poll()` (หัวข้อ 47.9) ทำกับ `current_delay` ของ
  มันเอง เพราะโดยเนื้อแท้แล้ว `Ticker` ก็คือ state machine ที่เขียนด้วยมือแบบเดียวกันนี้ เพียงแต่ไม่ได้ใช้ enum
  ตรง ๆ (ใช้ `Option<Delay>` แทน ซึ่งเทียบเท่ากับ enum สองสถานะ)
- **`std::mem::replace(&mut *self, GreetSlowly::Done)`**: เทคนิคสำคัญที่ต้องรู้เมื่อเขียน state machine ด้วยมือ
  — เราต้อง**ดึงข้อมูลออกจาก enum แบบ by-value** (เอา `String name` ออกมาเป็นเจ้าของตรง ๆ ไม่ใช่ borrow) เพื่อ
  ย้ายมันไปยัง state ถัดไป แต่ Rust ไม่อนุญาตให้ "ดึงค่าออกจาก enum ที่ยัง borrow อยู่" ตรง ๆ (เพราะจะทำให้ enum
  อยู่ในสถานะไม่สมบูรณ์ชั่วขณะ) `mem::replace` แก้ปัญหานี้โดยเขียนค่าใหม่ (`Done`) แทนที่ตำแหน่งเดิมพร้อมกับคืนค่า
  เก่าออกมาเป็น owned value ในขั้นตอนเดียวแบบ atomic (ไม่มีช่วงเวลาที่ enum อยู่ในสถานะไม่สมบูรณ์เลย) — compiler
  จริงใช้เทคนิคเดียวกันนี้เบื้องหลัง (จริง ๆ แล้วซับซ้อนกว่านี้เล็กน้อยในรายละเอียดของการจัดการ layout แต่หลักการ
  "ย้ายข้อมูลออกจาก state เก่าไปสู่ state ใหม่แบบปลอดภัย" เหมือนกันทุกประการ)
- **`GreetSlowly::Done => panic!("polled after completion")`**: ตามธรรมเนียมของ `Future` (เหมือนกับที่ Part 25
  บอกว่า `Iterator` "ไม่ควร" ถูกเรียก `next()` ต่อหลังได้ `None`) — การ poll future ต่อหลังจากมันคืน `Ready` ไป
  แล้วถือเป็นการใช้งานผิด (misuse) ที่ trait ไม่ได้ห้ามในระดับ type system แต่ implementation มีสิทธิ์ทำอะไรก็ได้
  รวมถึง panic เพราะไม่มี "state ถัดไป" ให้เดินต่อแล้วจริง ๆ

การเห็น state machine ตัวนี้เขียนด้วยมือแบบเต็มรูปแบบคือคำตอบสุดท้ายของคำถามที่ Part 46 เปิดไว้ตั้งแต่ต้น: "state
machine ที่ `async fn` สร้างให้คืออะไรกันแน่" — คำตอบคือ**enum ธรรมดาที่มีสถานะเท่ากับจำนวนจุด `.await` บวกหนึ่ง
(สถานะเริ่มต้น) และ `poll()` ที่เดินหน้า state ไปเรื่อย ๆ ในลูป จนกว่าจะเจอ sub-future ที่ยัง `Pending` จริง หรือ
จนกว่าจะถึงจุดสิ้นสุดของฟังก์ชัน** ไม่มีอะไรพิเศษไปกว่านี้เลย — ความซับซ้อนที่เพิ่มขึ้นในโค้ดจริงของ compiler มาจาก
การรองรับหลายจุด `.await` พร้อมกัน, การจัดการ borrow ที่ซับซ้อนกว่า (ซึ่งอาจทำให้ต้องใช้ `Pin` แบบเต็มรูปแบบตามหัวข้อ
47.7), และการ optimize ขนาดของ enum ให้เล็กที่สุด (compiler คำนวณ layout ให้ field ที่ไม่ได้ใช้ร่วมกันในหลาย state
ใช้พื้นที่หน่วยความจำซ้อนกันได้ คล้ายกับ union) — แต่**แนวคิดหลัก**เหมือนกับ `GreetSlowly` ข้างบนทุกประการ

### 47.11 ทางลัดจาก std: `poll_fn`, `ready`, และ `pending`

การเขียน `struct` + `impl Future` เต็มรูปแบบทุกครั้งที่ต้องการ future ง่าย ๆ ตัวหนึ่งดูจะเป็นภาระมากเกินไปสำหรับ
งานเล็ก ๆ — โมดูล `std::future` มีฟังก์ชันสำเร็จรูปสามตัวที่ช่วยลดภาระนี้ได้มาก โดยยังคงเป็น `std` ล้วน ๆ ไม่ต้อง
พึ่ง crate เสริมใด ๆ:

**`std::future::ready(value)`** — สร้าง future ที่ `Poll::Ready(value)` ทันทีตั้งแต่ poll แรก (เทียบเท่ากับ
`ReadyNow` ที่เขียนเองในหัวข้อ 47.2 แต่ไม่ต้องประกาศ struct เอง):

```rust
use std::future;

let value = block_on(future::ready(7));
println!("ready() ให้ผลลัพธ์: {value}"); // ready() ให้ผลลัพธ์: 7
```

**`std::future::poll_fn(closure)`** — สร้าง future จาก closure ที่มีลายเซ็นเหมือน `poll()` ตรง ๆ
(`FnMut(&mut Context<'_>) -> Poll<T>`) โดยไม่ต้องประกาศ struct หรือ `impl Future` เองเลย เหมาะกับ future แบบง่าย ๆ
ที่ไม่มี state ซับซ้อนพอจะคุ้มค่ากับการเขียน struct แยก:

```rust
use std::future;
use std::task::Poll;

let mut calls = 0;
let counting = future::poll_fn(move |cx| {
    calls += 1;
    println!("poll_fn ถูกเรียกครั้งที่ {calls}");
    if calls >= 3 {
        Poll::Ready(calls)
    } else {
        cx.waker().wake_by_ref();
        Poll::Pending
    }
});

let final_count = block_on(counting);
println!("poll_fn จบด้วยค่า: {final_count}");
```

ผลลัพธ์จริง (compile และรันจริงแล้ว):

```
poll_fn ถูกเรียกครั้งที่ 1
poll_fn ถูกเรียกครั้งที่ 2
poll_fn ถูกเรียกครั้งที่ 3
poll_fn จบด้วยค่า: 3
```

สังเกตว่า `poll_fn` ให้เขียน `PollCounter` (หัวข้อ 47.4) แบบเดียวกันได้ในบรรทัดเดียว โดยที่ closure `move |cx| { ...
}` **คือ** `poll()` ตรง ๆ (`calls` ที่ capture ด้วย `move` ทำหน้าที่แทน field `count` ของ struct) — `poll_fn` มี
ประโยชน์มากที่สุดตอนเขียน future ชั่วคราวที่ใช้ครั้งเดียวในโค้ดจริง (เช่นใน test หรือ glue code ระหว่าง API) ไม่คุ้ม
ที่จะแยกเป็น type ของตัวเอง

**`std::future::pending::<T>()`** — สร้าง future ที่คืน `Poll::Pending` **เสมอ** ไม่มีทางเสร็จเลย มีประโยชน์เฉพาะ
ทางมาก ๆ เช่นใช้เป็น "placeholder" ใน `select!` (จะเจอเวอร์ชันเต็มใน Part 48-50) ตอนที่กิ่งหนึ่งไม่มีอะไรให้รอจริง
แต่ต้องมี type ให้ตรงกับกิ่งอื่น ๆ — ไม่ควรใช้ตรง ๆ ใน `block_on` เพราะจะทำให้โปรแกรมค้างตลอดไปเหมือนกับดักข้อ 4
ท้ายบท (เพียงแต่ครั้งนี้เป็นพฤติกรรมที่**ตั้งใจ**ให้เกิดขึ้น ไม่ใช่บั๊ก)

### 47.12 บทพิสูจน์: รันหลาย Future พร้อมกันบน Thread เดียว

ถึงเวลาพิสูจน์ทุกอย่างที่บทนี้อธิบายมา — spawn สาม task ที่แตกต่างกันเข้า executor ที่สร้างในหัวข้อ 47.8 แล้วดูว่า
มันทำงาน**พร้อมกันจริง**บน thread เดียวหรือไม่:

```rust
fn main() {
    let (executor, spawner) = new_executor_and_spawner();

    spawner.spawn(Ticker::new("A", Duration::from_millis(150), 5));
    spawner.spawn(Ticker::new("B", Duration::from_millis(220), 3));
    spawner.spawn(PollCounter::new("C", 4));

    drop(spawner); // ปิด sender เพื่อให้ executor.run() คืนค่าเมื่อ task ทั้งหมดทำงานจบ

    println!("=== เริ่ม executor บน thread เดียว (main thread) ===");
    executor.run();
    println!("=== ทุก task ทำงานจบแล้ว ===");
}
```

**ผลลัพธ์จริงที่ได้จากการ compile และรันจริง** (ไม่ใช่ผลลัพธ์ที่คาดเดา):

```
=== เริ่ม executor บน thread เดียว (main thread) ===
[C] poll ครั้งที่ 1/4
[C] poll ครั้งที่ 2/4
[C] poll ครั้งที่ 3/4
[C] poll ครั้งที่ 4/4
[A] tick 1/5
[B] tick 1/3
[A] tick 2/5
[B] tick 2/3
[A] tick 3/5
[A] tick 4/5
[B] tick 3/3
[A] tick 5/5
=== ทุก task ทำงานจบแล้ว ===
```

(รันทั้งโปรแกรมใช้เวลาจริงประมาณ 0.78 วินาที — ตรงกับที่คาดไว้ เพราะ `A` ต้อง tick ครบ 5 ครั้ง ห่างกันครั้งละ 150ms
= 750ms รวม)

**วิเคราะห์ผลลัพธ์นี้อย่างละเอียด เพราะนี่คือ payoff ของทั้งบท:**

1. **`C` (PollCounter) จบก่อนใครทั้งหมดแทบจะทันที** เพราะมันไม่รอเวลาจริงเลย — ทุกครั้งที่ poll มันเรียก
   `wake_by_ref()` ทันที ส่งตัวเองกลับเข้าคิวใน "รอบถัดไปพอดี" executor จึงวน poll มันจนจบก่อนที่ background
   thread ของ `A`/`B` จะแม้แต่ตื่นครั้งแรกด้วยซ้ำ (150ms/220ms ยังไม่ผ่านไปเลยตอนที่ `C` เสร็จ)
2. **`A` และ `B` สลับกัน tick กันจริง** ตามช่วงเวลาที่ตั้งไว้ (`A` ทุก 150ms, `B` ทุก 220ms) — ไม่ใช่ `A` ทำจน
   จบก่อนแล้วค่อยไปทำ `B` แบบ sequential ทั้งที่โค้ดทั้งหมดรันอยู่บน **thread เดียวเท่านั้น** (thread หลักของ
   `main()` ที่เรียก `executor.run()`) — นี่คือคำจำกัดความของ **cooperative multitasking** ที่ Part 46 พูดถึงไว้
   เป็นแนวคิด และตอนนี้เราเห็นมันทำงานจริงต่อหน้าตา
3. **จำนวน OS thread ทั้งหมดในโปรแกรมนี้**: มี thread หลัก (`main`, รัน executor) บวก thread เบื้องหลังของ
   `Delay` แต่ละตัวที่ `Ticker` สร้างขึ้นชั่วคราว (แค่ `sleep` แล้ว `wake` เท่านั้น ไม่ได้ทำ "งาน" ของ task เลย)
   **งานจริงทั้งหมด (การพิมพ์ tick, การนับ poll, ตรรกะทุกอย่างของ `A`/`B`/`C`) ทำงานอยู่บน thread เดียวกันเท่านั้น
   — thread หลักของ executor** ต่างจาก concurrency แบบ thread ของ Part 37 อย่างสิ้นเชิง ที่ถ้าจะรัน `A`, `B`, `C`
   พร้อมกันแบบ Part 37 ต้อง `thread::spawn` แยกกันสามตัว ใช้ memory เต็ม stack ของแต่ละ thread (ปกติหลัก KB ถึง MB
   ต่อ thread) และมี context-switching cost จาก OS scheduler — แต่ที่นี่ **task ทั้งสามใช้ memory แค่เท่าที่ตัว
   struct ของมันเองใช้ (ไม่กี่สิบไบต์) และสลับกันทำงานโดย executor เอง ไม่ผ่าน OS scheduler เลย**

ความแตกต่างข้อ 3 นี่แหละคือเหตุผลทั้งหมดที่ async concurrency มีอยู่คู่กับ thread concurrency — ถ้าต้องรองรับ
task พร้อมกันหลักสิบ หลักร้อย การใช้ thread ต่อ task (Part 37-40) ยังพอไหว แต่ถ้าต้องรองรับหลักหมื่นถึงหลักล้าน
(เช่น web server ที่รับ connection พร้อมกันจำนวนมาก) การสร้าง OS thread ต่อ connection จะทำให้ระบบล้มก่อนถึงเป้า
ไกลมาก — นี่คือปัญหาที่ async model ถูกออกแบบมาแก้โดยเฉพาะ

### 47.13 Combinator โดยไม่ต้องมี Runtime เต็มรูปแบบ: `futures::future::join`/`select`

executor ที่เขียนเองในหัวข้อ 47.8 มีไว้เพื่อ**เรียน**กลไกเบื้องหลัง ในโค้ดที่ต้องรวม future หลายตัวแบบง่าย ๆ
(ไม่ต้อง spawn เป็น task อิสระ แค่ต้องการ "รอทั้งสองให้จบ" หรือ "รอตัวไหนจบก่อนก็เอา") มี crate มาตรฐานชื่อ
**`futures`** (คนละตัวกับ Tokio — ใช้ได้โดยไม่ต้องพึ่ง runtime เต็มรูปแบบเลย) ที่มี combinator สำเร็จรูปให้ใช้ พร้อม
executor เล็ก ๆ ของตัวเอง (`futures::executor::block_on`) สำหรับรันโค้ดตัวอย่าง/เขียนเทส โดยไม่ต้องติดตั้ง Tokio

เพิ่ม dependency ใน `Cargo.toml`:

```toml
[dependencies]
futures = "0.3"
```

**`future::join`** — รอทั้งสอง future ให้จบ แล้วได้ผลลัพธ์เป็น tuple ของทั้งคู่ (คล้ายกับรอ `JoinHandle` สอง
ตัวจาก Part 37 แต่เป็นเวอร์ชัน async ไม่บล็อก thread):

```rust
use futures::executor::block_on;
use futures::future;

async fn fetch_user() -> &'static str {
    println!("กำลังดึงข้อมูล user...");
    "user-42"
}

async fn fetch_settings() -> &'static str {
    println!("กำลังดึงข้อมูล settings...");
    "dark-mode"
}

fn main() {
    let (user, settings) = block_on(future::join(fetch_user(), fetch_settings()));
    println!("user = {user}, settings = {settings}");
}
```

ผลลัพธ์จริง (compile และรันจริงแล้ว):

```
กำลังดึงข้อมูล user...
กำลังดึงข้อมูล settings...
user = user-42, settings = dark-mode
```

**`future::select`** — รอ future ตัวไหนก็ตามที่จบ**ก่อน**เพียงตัวเดียว คืนผลลัพธ์ของตัวนั้นพร้อม future อีกตัวที่
ยังไม่จบ (เผื่อจะใช้ต่อ) ผลลัพธ์เป็น enum `Either<A, B>` (`Left` = future แรกจบก่อน, `Right` = future ที่สองจบ
ก่อน):

```rust
use futures::executor::block_on;
use futures::future::{select, Either};
use std::time::Duration;
// สมมติว่า Delay ในที่นี้คือ future เดียวกันจากหัวข้อ 47.6 (แค่เปลี่ยน Output เป็น &'static str)

fn main() {
    let fast = Delay::new(Duration::from_millis(100)); // สมมติคืนค่า "หมดเวลา" ตอน Ready
    let slow = Delay::new(Duration::from_millis(500));

    // select ต้องการ future ที่เป็น Unpin ทั้งคู่ — Delay เป็น Unpin โดยอัตโนมัติ (หัวข้อ 47.7)
    // จึงส่งเข้าไปตรง ๆ ได้เลย ไม่ต้อง pin_mut!/Box::pin เพิ่ม
    match block_on(select(fast, slow)) {
        Either::Left((value, _slow_remaining)) => println!("อันเร็วเสร็จก่อน: {value}"),
        Either::Right((value, _fast_remaining)) => println!("อันช้าเสร็จก่อน (ผิดคาด): {value}"),
    }
}
```

ผลลัพธ์จริง (compile และรันจริงแล้ว):

```
อันเร็วเสร็จก่อน: หมดเวลา
```

ตรงตามคาด เพราะ `fast` ตั้งไว้ 100ms ส่วน `slow` ตั้งไว้ 500ms — `select` เลือกตัวที่จบก่อนให้อัตโนมัติ สังเกตว่า
เราไม่ต้องเขียน executor เอง ไม่ต้องจัดการ `Waker` เอง ไม่ต้องกังวลเรื่อง `Pin` เลยแม้แต่นิดเดียวในระดับผู้ใช้
— นี่คือสิ่งที่ crate อย่าง `futures` มีไว้ให้ (จัดการรายละเอียดทั้งหมดที่บทนี้เพิ่งอธิบายไปให้เบื้องหลัง) ซึ่งพา
เราไปสู่หัวข้อสุดท้ายของบทนี้โดยตรง

### 47.14 ทำไมไม่มีใครเขียน Future/Executor เองในโปรดักชันจริง

หลังจากเขียน `Delay`, `Ticker`, `PollCounter`, executor เต็มรูปแบบ และเห็นมันทำงานถูกต้องด้วยตาตัวเองแล้ว คำถาม
ธรรมชาติคือ: "แล้วทำไมโปรเจกต์จริงถึงไม่เขียนแบบนี้เอง แต่ไปพึ่ง Tokio (Part 48) กันหมด?"

คำตอบคือหลักการเดียวกันเป๊ะกับที่ Part 30-31 สอนไว้ตอนอธิบายว่าทำไมโปรเจกต์จริงไม่เขียน error type เองทั้งหมดแต่
หันไปใช้ `thiserror`/`anyhow` — **"ปัญหาที่พบบ่อยมาก ยากที่จะทำให้ถูกต้องและมีประสิทธิภาพจริง ควรใช้ของที่ผ่านการ
พิสูจน์และดูแลโดยทีมที่เชี่ยวชาญมาแล้ว ไม่ใช่เขียนเองใหม่ทุกโปรเจกต์"** executor ที่เขียนในบทนี้พอสำหรับ**การเรียน**
แต่ขาดสิ่งสำคัญหลายอย่างที่ executor โปรดักชันต้องมี:

1. **การ integrate กับ OS async I/O ตัวจริง**: `Delay` ของเราสร้าง **OS thread ใหม่หนึ่งตัวต่อ future หนึ่งตัว**
   ที่ต้องรอเวลา/I/O — ถ้ามี 10,000 connection ที่ต้องรอพร้อมกัน นั่นคือ 10,000 OS thread ซึ่ง**ขัดกับเป้าหมายทั้ง
   หมดของ async** (ที่ต้องการเลี่ยงต้นทุนของ thread ต่อ task ตั้งแต่แรก!) executor โปรดักชันอย่าง Tokio ใช้กลไก
   ระดับ OS โดยตรง — **epoll** บน Linux, **kqueue** บน macOS/BSD, **IOCP** บน Windows, หรือ **io_uring** (Linux
   รุ่นใหม่ที่เร็วกว่า epoll มาก) — ที่ให้ thread จำนวนน้อยมาก (มักเท่ากับจำนวน CPU core) คอย "ฟัง" การเปลี่ยน
   สถานะของ file descriptor นับหมื่นตัวพร้อมกันได้โดย**ไม่ต้อง**มี thread ต่อ descriptor เลย — นี่คือวิศวกรรม
   ระดับระบบปฏิบัติการที่ซับซ้อนมาก เขียนให้ถูกต้องและเร็วจริงยากกว่าที่คิดมหาศาล
2. **Work-stealing scheduler แบบ multi-thread**: Tokio ใช้ thread pool ขนาดเท่าจำนวน CPU core แล้วกระจาย task
   ข้าม thread เหล่านั้นอย่างฉลาด (ถ้า thread หนึ่งว่างแต่อีก thread มี task ล้นคิว มันจะ "ขโมย" งานมาทำ) —
   executor ของเราใน 47.8 เป็น single-thread ล้วน ๆ ทำงานถูกต้องแต่ใช้ CPU core ได้แค่ตัวเดียวเสมอ ไม่ว่าเครื่องจะ
   มีกี่ core ก็ตาม
3. **Timer wheel ที่มีประสิทธิภาพ**: ถ้ามี timer (แบบ `Delay`) เป็นหมื่นตัวพร้อมกัน การเช็คทีละตัวว่าตัวไหนครบเวลา
   แล้วเป็นวิธีที่ช้ามาก runtime จริงใช้โครงสร้างข้อมูลเฉพาะทาง (เช่น hierarchical timer wheel) ให้การจัดการ timer
   จำนวนมากเร็วในระดับ O(1) โดยประมาณ
4. **Cancellation, panic safety, และ edge case อีกมากมาย**: เช่น จะเกิดอะไรขึ้นถ้า future ถูก drop กลางทางก่อนจบ
   (ต้อง cleanup resource ถูกต้อง), จะเกิดอะไรขึ้นถ้า task panic (ไม่ควรทำให้ executor ทั้งตัวล้ม), จะจัดการ
   `Waker` ที่ถูกเรียกซ้ำหลายรอบพร้อมกันจากหลาย thread อย่างไรให้ไม่มี race condition — สิ่งเหล่านี้ทดสอบและแก้ให้
   ถูกต้อง 100% ยากกว่าที่ตัวอย่างในบทนี้แสดงไว้มาก

Tokio (ซึ่งเป็นหัวข้อของ **Part 48** ที่กำลังจะเรียนต่อจากนี้) คือคำตอบของอุตสาหกรรมสำหรับปัญหาทั้งหมดนี้ — เป็น
runtime ที่ผ่านการทดสอบ ปรับแต่งประสิทธิภาพ และใช้งานจริงในโปรดักชันโดยบริษัทนับพันแห่งมาหลายปี ทุกอย่างที่บทนี้
สอน (นิยามของ `Future`, `Poll`, `Waker`, `Pin`) คือ**พื้นฐานที่จำเป็น**ในการอ่าน error message ของ Tokio ให้เข้าใจ
และเขียนโค้ด async ขั้นสูงได้อย่างมั่นใจ (เช่น การเขียน combinator เอง หรือการ debug ปัญหาที่เกี่ยวกับ `Pin`) — แต่
**ไม่มีใครแนะนำให้เขียน executor เองใช้ในโปรดักชันจริง** เว้นแต่มีเหตุผลพิเศษมาก ๆ (เช่น เขียน runtime สำหรับ
embedded system ที่ไม่มี OS เต็มรูปแบบให้ epoll/io_uring ใช้ ซึ่งเป็นกรณีที่หายากและต้องการความเชี่ยวชาญสูงมาก)

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. เรียก `.poll()` ตรง ๆ โดยไม่ Pin ก่อน

```rust
use std::future::Future;
use std::task::{Context, Poll, Waker};

struct ReadyNow(i32);

impl Future for ReadyNow {
    type Output = i32;
    fn poll(self: std::pin::Pin<&mut Self>, _cx: &mut Context<'_>) -> Poll<Self::Output> {
        Poll::Ready(self.0)
    }
}

fn main() {
    let mut fut = ReadyNow(42);
    let waker = Waker::noop();
    let mut cx = Context::from_waker(waker);
    let _ = fut.poll(&mut cx); // ผิด!
}
```

Error จริงจาก compiler:

```
error[E0599]: no method named `poll` found for struct `ReadyNow` in the current scope
  --> examples/ex05_pin_required_fail.rs:20:17
   |
 4 | struct ReadyNow(i32);
   | --------------- method `poll` not found for this struct
...
20 |     let _ = fut.poll(&mut cx);
   |                 ^^^^ method not found in `ReadyNow`
   |
help: consider pinning the expression
   |
20 ~     let mut pinned = std::pin::pin!(fut);
21 ~     let _ = pinned.as_mut().poll(&mut cx);
   |
```

**เหตุผล**: method resolution ของ Rust ลองหา method บน `Self`, `&Self`, `&mut Self` ให้อัตโนมัติ (auto-ref) แต่
**ไม่มีทาง**ลองแปลงเป็น `Pin<&mut Self>` ให้อัตโนมัติ เพราะ `Pin` เป็นแค่ struct ธรรมดา ไม่ใช่ reference ชนิด
พิเศษที่ compiler รู้จัก ต้อง pin เองอย่างชัดเจนก่อนเสมอ — วิธีแก้ตรงตามที่ compiler แนะนำ: ใช้
`Pin::new(&mut fut)` (ถ้า type เป็น `Unpin`) หรือ macro `std::pin::pin!(fut)` (ทำงานได้กับทุก type ไม่ว่า
`Unpin` หรือไม่ pin บน stack)

### 2. Type ไม่ใช่ `Unpin` แล้วพยายามใช้ `Pin::new`

```rust
use std::future::Future;
use std::marker::PhantomPinned;
use std::pin::Pin;
use std::task::{Context, Poll, Waker};

struct NotUnpinFuture {
    _pin: PhantomPinned,
}

impl Future for NotUnpinFuture {
    type Output = ();
    fn poll(self: Pin<&mut Self>, _cx: &mut Context<'_>) -> Poll<Self::Output> {
        Poll::Ready(())
    }
}

fn main() {
    let mut fut = NotUnpinFuture { _pin: PhantomPinned };
    let waker = Waker::noop();
    let mut cx = Context::from_waker(waker);
    let _ = Pin::new(&mut fut).poll(&mut cx); // ผิด!
}
```

Error จริงจาก compiler:

```
error[E0277]: `PhantomPinned` cannot be unpinned
  --> examples/ex06_unpin_bound_fail.rs:22:22
   |
22 |     let _ = Pin::new(&mut fut).poll(&mut cx);
   |             -------- ^^^^^^^^ within `NotUnpinFuture`, the trait `Unpin` is not implemented for `PhantomPinned`
   |             |
   |             required by a bound introduced by this call
   |
   = note: consider using the `pin!` macro
           consider using `Box::pin` if you need to access the pinned value outside of the current scope
note: required because it appears within the type `NotUnpinFuture`
```

**เหตุผล**: `Pin::new` เป็น **safe** constructor เพราะมันมี bound `where P::Target: Unpin` — ถ้า type ไม่ใช่
`Unpin` การ `Pin::new` แบบปลอดภัยจะไม่มีความหมายเลย (มันไม่ได้การันตีอะไรเพิ่มขึ้นจากการห่อ `Pin` รอบ type ที่
อาจถูกย้ายจากภายนอกได้ก่อนหน้านี้แล้ว) ตามที่ compiler แนะนำ ให้ใช้ `Box::pin(fut)` แทน (ย้ายเข้า heap ครั้งเดียว
ก่อนใครจะสร้าง self-reference ได้ ปลอดภัยเสมอไม่ว่า `Unpin` หรือไม่) หรือ `std::pin::pin!(fut)` (pin บน stack แบบ
ปลอดภัยโดยไม่ต้อง heap allocate)

### 3. เก็บ `&Waker` ตรง ๆ โดยไม่ `.clone()`

```rust
struct BadDelay<'w> {
    ready: bool,
    stored_waker: Option<&'w Waker>,
}

impl<'w> Future for BadDelay<'w> {
    type Output = ();
    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        if self.ready {
            Poll::Ready(())
        } else {
            self.stored_waker = Some(cx.waker()); // ผิด!
            Poll::Pending
        }
    }
}
```

Error จริงจาก compiler:

```
error: lifetime may not live long enough
  --> examples/ex08_waker_lifetime_fail.rs:19:13
   |
10 | impl<'w> Future for BadDelay<'w> {
   |      -- lifetime `'w` defined here
...
13 |     fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
   |                                       -- has type `&mut Context<'1>`
...
19 |             self.stored_waker = Some(cx.waker());
   |             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ assignment requires that `'1` must outlive `'w`
```

**เหตุผล**: `cx.waker()` คืน `&Waker` ที่มีอายุผูกกับพารามิเตอร์ `cx: &mut Context<'_>` ของการเรียก `poll()`
ครั้งนี้เท่านั้น — เมื่อ `poll()` return ไป lifetime นั้นก็จบลง แต่เราต้องการเก็บ waker ไว้**ใช้ทีหลัง** (หลังจาก
`poll()` return ไปนานแล้ว เช่นตอน background thread ตื่นค่อยเรียก) การเก็บ reference ที่จะหมดอายุก่อนที่จะใช้จึง
เป็นไปไม่ได้ในหลักการ ownership/lifetime จาก Part 6-7/20/23 ทั้งหมด — นี่คือเหตุผลที่ `Waker` ต้อง implement
`Clone` และเราต้อง **`cx.waker().clone()`** เสมอเมื่อจะเก็บมันไว้ใช้เกินอายุของการเรียก `poll()` ครั้งนั้น (ตามที่
`Delay::poll()` ในหัวข้อ 47.6 ทำไว้ถูกต้องแล้ว)

### 4. ลืมเรียก `.wake()`: บั๊กที่อันตรายที่สุดของทั้งบท — โปรแกรมค้างเงียบ ๆ ตลอดไป

```rust
// เหมือน Delay ทุกอย่าง แต่ "ลืม" เรียก waker.wake() ตอน background thread ทำงานเสร็จ
thread::spawn(move || {
    thread::sleep(duration);
    let mut state = thread_shared_state.lock().unwrap();
    state.completed = true;
    // ลืมเรียก state.waker.take().map(Waker::wake) ตรงนี้!
});
```

โค้ดนี้ **compile ผ่านสมบูรณ์แบบ ไม่มี error หรือ warning เลยแม้แต่ตัวเดียว** — แต่รันแล้วผลลัพธ์จริงคือโปรแกรม
**ค้างตลอดไป** (ทดสอบจริงด้วยการรันภายใต้ `timeout 3s` แล้วยืนยันว่าโปรแกรมไม่จบภายใน 3 วินาทีตามที่คาด แม้ว่า
`Delay` ข้างในตั้งเวลาไว้แค่ 200ms):

```
เริ่มรอ (คาดว่าจะค้างตลอดไปเพราะบั๊ก)...
(ค้างอยู่ตรงนี้ตลอดไป — ไม่มีบรรทัดถัดไปพิมพ์ออกมาเลย)
```

**เหตุผล**: สัญญา (contract) ของ trait `Future` คือ "ถ้าคืน `Pending` ต้องมีคนเรียก waker ในอนาคตแน่ ๆ" —
compiler **ไม่มีทาง**ตรวจสอบสัญญานี้ให้เราได้เลย (มันเป็น semantic ที่อยู่เหนือ type system) ถ้าละเมิดสัญญานี้
(ลืมเรียก wake) executor จะรอ task นั้นตลอดไปโดยไม่มีทางรู้ว่าควร poll มันใหม่เมื่อไหร่ — เพราะไม่เคยมีสัญญาณอะไร
บอกมันเลย นี่คือเหตุผลที่ต้อง**ทดสอบ future ที่เขียนเองอย่างระมัดระวังเป็นพิเศษ** (โดยเฉพาะเส้นทางที่ error/แตกกิ่ง
ทุกเส้นทางต้องเรียก wake ให้ครบ ไม่ใช่แค่เส้นทางปกติ) — บั๊กแบบนี้เป็นสาเหตุอันดับต้น ๆ ที่ทำให้แนะนำให้ใช้
combinator สำเร็จรูปจาก `futures`/Tokio แทนการเขียน `Future` เองเมื่อเป็นไปได้ (ตามหัวข้อ 47.12)

### 5. `thread::spawn` หนึ่งตัวต่อ Future หนึ่งตัว: ไม่ scale (เจอเองใน `Delay`)

`Delay` ที่เขียนในหัวข้อ 47.6 สร้าง OS thread ใหม่**ทุกครั้ง**ที่ถูกสร้างขึ้น ใช้ได้ดีสำหรับตัวอย่างสอนที่มี
`Delay`/`Ticker` แค่ไม่กี่ตัว แต่ถ้านำ pattern นี้ไปใช้ในระบบจริงที่ต้องจัดการ timer หรือ I/O นับพันตัวพร้อมกัน จะ
เจอปัญหาจำนวน thread ระเบิด (thread นับพันตัวกินหน่วยความจำมหาศาลจากขนาด stack ของแต่ละ thread และทำให้ OS
scheduler ทำงานหนักเกินจำเป็น) — ตามที่อธิบายไว้ในหัวข้อ 47.12 นี่คือเหตุผลหลักที่ต้องใช้ runtime อย่าง Tokio ที่
integrate กับ epoll/io_uring/kqueue/IOCP โดยตรง แทนที่จะสร้าง thread ต่อ timer/connection แบบที่บทนี้ทำเพื่อการ
เรียนเท่านั้น — **ห้ามนำ `Delay` เวอร์ชันนี้ไปใช้ในโปรเจกต์จริงเด็ดขาด** ให้ใช้ `tokio::time::sleep` (Part 48)
แทนเสมอ

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** แก้ไข `PollCounter` ให้พิมพ์ "เหลืออีก N ครั้ง" แทน "poll ครั้งที่ X/Y" (คำนวณจาก `target - count`)
   แล้วทดสอบรันด้วยลูป `main()` แบบเดียวกับหัวข้อ 47.4 ยืนยันว่าตัวเลขนับถูกต้อง — โจทย์นี้ฝึกความคุ้นเคยกับการแก้
   state ภายใน `poll()` ผ่าน `mut self: Pin<&mut Self>`

2. **(กลาง)** เขียน `TimeoutTicker` — future ที่ห่อ `Ticker` ไว้อีกชั้น แต่ยอมให้ tick ได้ไม่เกิน `max_duration`
   รวม ถ้าเวลารวมเกินก่อนที่ `Ticker` จะ tick ครบ ให้ future คืน `Poll::Ready(())` ทันที (ยกเลิกกลางทาง) โดยไม่ต้อง
   ใช้ crate `futures` เลย (implement `poll()` เองที่คอยเช็ค `Instant::now()` เทียบกับเวลาเริ่มต้นที่เก็บไว้ตอน
   สร้าง) *Hint*: เก็บ `start: Instant` ไว้ใน struct ตอน `new()` แล้วเช็คใน `poll()` ก่อนจะ delegate ไปยัง
   `Ticker::poll()` ภายใน

3. **(ยาก)** ขยาย executor ในหัวข้อ 47.8 ให้รองรับการนับสถิติ: เพิ่ม `Arc<Mutex<HashMap<&'static str, u32>>>`
   (หรือ `AtomicU32` แยกตัวถ้าต้องการหลีกเลี่ยง Mutex) ที่นับว่าแต่ละ task ถูก `.poll()` ไปกี่ครั้งทั้งหมด แล้ว
   พิมพ์สรุปหลังจาก `executor.run()` จบ เพื่อพิสูจน์ด้วยตัวเลขจริงว่า `PollCounter` (busy-poll) ถูก poll บ่อยกว่า
   `Ticker` (รอ Waker จริง) มากแค่ไหนต่อ "งาน" ที่ทำเสร็จเท่ากัน — เชื่อมกับสิ่งที่อธิบายไว้ในหัวข้อ 47.5 ว่า
   busy-poll เปลืองทรัพยากรกว่า *Hint*: ต้องแก้ `Task` ให้ถือ label หรือ id ของตัวเอง และแก้ `Executor::run()` ให้
   เพิ่มตัวนับก่อนเรียก `.poll()` ทุกครั้ง

4. **(ยาก/ประยุกต์)** เขียน future ของตัวเองชื่อ `RaceFirst<A, B>` ที่ทำงานเหมือน `futures::future::select` แต่
   เขียนเองทั้งหมดโดยไม่พึ่ง crate `futures` เลย — รับสอง future ที่มี `Output` ชนิดเดียวกัน คืนผลลัพธ์ของตัวที่
   `Ready` ก่อน (ถ้าทั้งสองยัง `Pending` ก็คืน `Pending` ต่อไป) ทดสอบด้วย `Delay` สองตัวที่มีเวลาต่างกัน ผ่าน
   `block_on` จากหัวข้อ 47.6 ยืนยันว่าได้ผลลัพธ์จากตัวที่เวลาน้อยกว่าเสมอ *Hint*: `poll()` ของ `RaceFirst` ต้อง
   `Pin::new(&mut self.a).poll(cx)` แล้วเช็คก่อนว่า `Ready` หรือยัง ถ้ายังค่อยลอง `b` ต่อในการเรียกเดียวกัน (สำคัญ:
   ต้อง poll ทั้งสองตัวใน**ทุกครั้ง**ที่ `poll()` ถูกเรียก ไม่ใช่แค่ตัวเดียว ไม่อย่างนั้นอีกตัวจะไม่มีโอกาสได้
   ลงทะเบียน waker ของมันเลย และจะไม่มีวันถูก wake — ย้อนกลับไปดูกับดักข้อ 4 ถ้าไม่แน่ใจว่าทำไมสำคัญ)

## สรุป

บทนี้รื้อกลไกเบื้องหลัง async/await ที่ Part 46 สอนไว้เป็น syntax ออกมาดูทุกชิ้นส่วนจนหมด: **`Future`** คือ trait
ที่มี associated type `Output` และ method เดียวคือ `poll()` — เหมือน `Iterator` ทุกประการ เพียงแต่ให้ค่าออกมาได้
อย่างมากหนึ่งครั้ง **`Poll<T>`** คือ enum สองแบบที่ใช้ปรัชญาเดียวกับ `Option<T>` แทนความหมาย "เสร็จแล้ว" หรือ
"ยังไม่เสร็จ" โดยไม่ต้องพึ่ง blocking หรือ exception **`Waker`** คือกลไกที่ทำให้ executor ไม่ต้อง busy-loop —
future ที่คืน `Pending` มีหน้าที่รับผิดชอบเรียก waker กลับเมื่อพร้อมคืบหน้าต่อได้ **`Pin<T>`** คือการันตีที่
future's compiler-generated state machine ต้องการ เพราะมันอาจเป็น self-referential struct (ปัญหาที่ Part 23
เกริ่นไว้) และ **`Unpin`** คือ auto trait (แบบเดียวกับ `Send`/`Sync` จาก Part 40) ที่บอกว่า type ไหนไม่ต้องสนใจ
การันตีนี้เลยเพราะมันปลอดภัยที่จะย้ายอยู่แล้ว จากนั้นเราสร้าง **executor ของตัวเองตั้งแต่ต้น** ที่รันหลาย future
พร้อมกันบน thread เดียวได้จริง และพิสูจน์ด้วยผลลัพธ์การรันจริงว่ามันคือ cooperative multitasking แท้ ๆ ไม่ใช่แค่
คำโฆษณา ปิดท้ายด้วย combinator จาก crate `futures` (`join`/`select`) ที่ใช้ได้โดยไม่ต้องมี runtime เต็มรูปแบบ และ
เหตุผลที่โปรเจกต์จริงแทบไม่มีใครเขียน `Future`/executor เองเลย — integration กับ epoll/io_uring/kqueue/IOCP,
work-stealing scheduler, timer wheel ที่มีประสิทธิภาพ, และ edge case อีกมากมายที่ต้องทำให้ถูกต้อง 100% คือปัญหาที่
ยากเกินกว่าจะเขียนเองในโปรเจกต์ปกติ — เหตุผลเดียวกันเป๊ะกับที่ Part 30-31 อธิบายไว้ตอน error handling

**Part 48 (Tokio: Runtime และ Tasks)** คือคำตอบของอุตสาหกรรมสำหรับปัญหาทั้งหมดนี้ — runtime async ที่ใช้งานจริง
ในโปรดักชันมากที่สุดในโลก Rust จะนำทุกแนวคิดจากบทนี้ (`Future`, `Poll`, `Waker`, `Pin`) มาใช้งานจริงผ่าน API ระดับ
สูงที่ไม่ต้องเขียน `poll()` เองอีกต่อไป (`tokio::spawn`, `tokio::time::sleep`, `tokio::select!`) แต่ทุกครั้งที่เจอ
error message ที่พูดถึง `Send`, `'static`, หรือ `Unpin` ในโค้ด Tokio — ความเข้าใจจากบทนี้คือกุญแจที่ทำให้อ่าน
error message เหล่านั้นออกและแก้ได้อย่างมั่นใจ ไม่ใช่แค่ "ลองแก้ไปเรื่อย ๆ จนมันหาย"

---

**Part ก่อนหน้า:** [Async/Await เบื้องต้น](part-046-async-await-basics.md) | **Part ถัดไป:** [Tokio: Runtime และ Tasks](part-048-tokio-runtime.md)
