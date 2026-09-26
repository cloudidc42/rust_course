# Part 56: Memory Management ขั้นสูง และ Zero-cost Abstractions

> โมดูล: ระดับสูง (Advanced) | ระดับ: สูง | เวลาโดยประมาณ: 260 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- เดินเรื่องราวหน่วยความจำ**เต็มรูปแบบ**ของโปรแกรม Rust จริงหนึ่งโปรแกรม ตั้งแต่ stack frame แรกที่ `main()`
  เปิดขึ้น ไปจนถึง heap allocation ทุกก้อน การ move ทุกครั้ง และ `Drop` ทุกครั้งที่รัน — พร้อมระบุได้ว่าความรู้
  แต่ละชิ้นมาจาก Part ไหนในหลักสูตร (Part 6, 7, 13, 14, 27, 28, 42) และเห็นภาพรวมทั้งหมดเป็น**หนึ่งโมเดลทาง
  ความคิดเดียว** ไม่ใช่ความรู้แยกส่วนที่ไม่เชื่อมกัน
- **พิสูจน์** (ไม่ใช่แค่ท่องจำ) คำสัญญา "zero-cost abstraction" ที่หลักสูตรพูดถึงมาตั้งแต่ Part 1 ด้วยหลักฐาน
  เชิงประจักษ์จริง 3 ชุด: (1) generic function ที่ monomorphize แล้วให้ machine code เหมือนกับที่เขียนมือ
  100% ตรวจสอบด้วย `objdump`/`nm` บนไบนารีจริง (2) iterator chain ที่ compile เป็น machine code ระดับเดียว
  กับ manual loop ตรวจสอบด้วยวิธีเดียวกัน (3) `const fn` ที่คำนวณค่าตอน compile time แล้ว "หายไปเลย" จาก
  runtime code ทั้งหมด
- วัดต้นทุนจริงของ heap allocation เทียบกับ stack allocation ด้วย `criterion` (Part 54) เป็นตัวเลขจริงที่รัน
  ได้ ไม่ใช่การคาดเดา และอธิบายเชิงกลไกได้ว่าทำไมตัวเลขนั้นถึงออกมาแบบนั้น
- อธิบายระดับ awareness ของเครื่องมือควบคุมหน่วยความจำระดับสุดขั้วสองตัวที่ Rust เปิดให้ทำได้: custom global
  allocator ผ่าน `#[global_allocator]`/`GlobalAlloc` และการเขียนโปรแกรมแบบ `#[no_std]` สำหรับ embedded/kernel
- วัดผลกระทบจริงของ data layout (array-of-structs เทียบกับ struct-of-arrays) ต่อ cache locality ด้วย
  `criterion` เชื่อมกับความรู้เรื่อง padding/alignment จาก Part 42 พร้อมเข้าใจว่า**ไม่มีคำตอบเดียวที่ถูกเสมอ**
  — คำตอบขึ้นกับรูปแบบการเข้าถึงข้อมูลจริงของงานนั้น
- ท่องจำ**เช็คลิสต์ต้นทุนหน่วยความจำ** ที่รวบรวมทุกอย่างที่หลักสูตรสอนมาตั้งแต่ Part 6 ถึง Part 51 ไว้ในที่
  เดียว (อะไร**ฟรีเสมอ**, อะไร**ถูกแต่ไม่ฟรี**, อะไร**แพงจริง**) และนำไปประยุกต์กับตัวอย่างจำลองอนุภาค
  (particle simulation) ที่ปรับปรุงทีละขั้นจนเห็นผลต่างของประสิทธิภาพแบบทวีคูณ

## ความรู้ที่ต้องมีมาก่อน

บทนี้เป็น**บทสังเคราะห์** (synthesis chapter) ที่ปิดสายเนื้อหาระดับระบบ/ประสิทธิภาพของโมดูล 3 (Part 41–56) จึง
มีความรู้ที่ต้องมีมาก่อนมากกว่าปกติ เพราะเราจะดึงทุกเส้นเรื่องที่แยกสอนไว้ตลอดหลักสูตรมาร้อยเข้าด้วยกัน:

- **Part 6 (Ownership และ Move เบื้องต้น)** — กฎ ownership, ความแตกต่างระหว่าง `Copy` กับ move, และคำกล่าวเชิง
  คุณภาพ (qualitative) ที่ Part 6 ทิ้งไว้ว่า "heap allocation แพงกว่า stack" — บทนี้จะเอาคำกล่าวนั้นมา**วัดเป็น
  ตัวเลขจริง**เป็นครั้งแรกในหลักสูตร ด้วย `criterion`
- **Part 18 (Generics เบื้องต้น)** — กลไก **monomorphization** ที่ Part 18 อธิบายด้วยเครื่องมือ `nm` แบบเบา ๆ
  (ดูขนาดไฟล์ symbol) บทนี้จะเจาะลึกกว่านั้นด้วยการอ่าน**เนื้อ assembly จริง** เพื่อพิสูจน์ว่า generic ที่
  monomorphize แล้วให้ machine code แบบเดียวกับที่เขียนมือ และทวนเรื่อง const generics (`Grid<const N: usize>`)
  ที่ Part 18 เกริ่นไว้สั้น ๆ
- **Part 25–26 (Iterators เบื้องต้น/ขั้นสูง)** — iterator chain (`.filter().map().sum()`) ที่ Part 25-26 สอนไว้
  ว่า "เร็วเท่ากับ loop มือ" แต่ไม่เคยพิสูจน์ด้วยเครื่องมือระดับ assembly จริง ๆ ในบทนั้น — บทนี้จะพิสูจน์ให้เห็น
  ด้วยตาตัวเอง
- **Part 27–29 (Smart Pointers: Box, Rc/RefCell, Deref/Drop)** — `Box<T>` คือตัวห่อ heap allocation ที่ชัดเจน
  ที่สุด, `Rc<T>`/`RefCell<T>` คือตัวอย่างของต้นทุนที่ "ถูกแต่ไม่ใช่ศูนย์" (reference counting) ที่บทนี้จะเอามา
  จัดหมวดหมู่ในเช็คลิสต์ต้นทุนรวม
- **Part 42 (Raw Pointers และ Memory Layout)** — ทุกอย่างเรื่อง `size_of`/`align_of`, padding, field reordering,
  และ `#[repr(...)]` ที่ Part 42 สอนไว้อย่างละเอียด บทนี้จะใช้ความรู้นั้นตรง ๆ ในหัวข้อ cache-friendliness และ
  AoS vs SoA — ถ้ายังไม่แม่นเรื่อง alignment/padding ควรกลับไปทวน Part 42 ก่อน
- **Part 51 (Atomics และ Lock-free Programming)** — false sharing, cache line contention, และต้นทุนของ atomic
  operation ภายใต้ contention ที่ Part 51 สอนไว้ จะถูกนำมาใส่ในเช็คลิสต์ต้นทุนรวมของบทนี้เช่นกัน
- **Part 54 (Performance Optimization และ Benchmarking ด้วย criterion)** — บทนี้**ไม่สอน `criterion` ซ้ำ**
  ตั้งแต่ต้น (การติดตั้ง, `benches/`, `[[bench]]`, `harness = false`, `black_box`, `iter_batched`) แต่ใช้ทุก
  เครื่องมือนั้นเป็นฐานตรง ๆ ทันที ถ้ายังไม่แม่นเรื่องการติดตั้งและอ่านผล `criterion` ควรกลับไปทวน Part 54 ก่อน
- ความรู้เสริมที่มีประโยชน์ (ไม่บังคับ): **Part 21 (dyn Trait และ vtable)**, **Part 39 (Mutex/Arc)** — บทนี้จะ
  อ้างอิงข้อสรุปเรื่องต้นทุนของ dynamic dispatch และ atomic reference counting จากทั้งสองบทในหัวข้อเช็คลิสต์
  ต้นทุนรวม (56.10) แต่จะอธิบายซ้ำในระดับที่พอเข้าใจได้แม้ยังไม่เคยอ่านสองบทนั้น

## เนื้อหา

### 56.1 วนกลับไปมองภาพรวม: เรื่องราวหน่วยความจำเต็มรูปแบบของโปรแกรมหนึ่งโปรแกรม

ตลอด 55 บทที่ผ่านมา เราเรียนเรื่องหน่วยความจำเป็นชิ้น ๆ แยกกันไปทีละเรื่อง: Part 6 สอน ownership/move และ
stack/heap เบื้องต้น, Part 13 สอนว่า `Vec<T>` เก็บข้อมูลบน heap และโต growth ยังไง, Part 14 สอนว่า `String`
คือ `Vec<u8>` แบบพิเศษ, Part 27 สอน `Box<T>` เป็นตัวห่อ heap allocation ตัวเดียวที่ชัดเจนที่สุด, Part 28 สอน
`Rc<T>`/`RefCell<T>` สำหรับความเป็นเจ้าของร่วมและ mutation ที่ตรวจตอน runtime, และ Part 42 เจาะลึกที่สุดว่า
ข้อมูลแต่ละก้อน**วางตัวจริง**ในหน่วยความจำอย่างไร (alignment, padding, `repr`) — ทุกเรื่องนี้ถูกสอนแยกกัน เพราะ
ถ้าสอนพร้อมกันหมดตั้งแต่ Part 6 ผู้เรียนจะงงเกินไป

บทนี้คือจุดที่เรา**หยุดแยกส่วน แล้วร้อยทุกเส้นเรื่องเข้าด้วยกัน**เป็นภาพเดียว ลองดูโปรแกรมสมมติที่จำลองระบบ
จัดการ "ห้องแชท" ง่าย ๆ — เลือกโปรแกรมนี้เพราะมันใช้ทุกเครื่องมือที่เรียนมาแล้วพร้อมกันในที่เดียว: `String`
(Part 14), `Vec<T>` (Part 13), `Box<T>` (Part 27), และ `Rc<RefCell<T>>` (Part 28):

```rust
use std::cell::RefCell;
use std::rc::Rc;

// ChatRoom ถือสมาชิกร่วมกันได้หลายคน (Rc) และแต่ละคนแก้ไขสถานะของ ChatRoom ได้ (RefCell)
struct ChatRoom {
    name: String,           // heap allocation แยกก้อน (buffer ของตัวอักษร)
    messages: Vec<String>,  // heap allocation หนึ่งก้อนสำหรับ Vec buffer + อีกก้อนต่อ String ข้างใน
}

struct User {
    username: String,
    room: Rc<RefCell<ChatRoom>>, // ผู้ใช้หลายคนถือ Rc ชี้ไปยัง ChatRoom เดียวกัน
}

impl User {
    fn send(&self, text: &str) {
        // .borrow_mut() ตรวจ aliasing rule ตอน runtime (Part 28) แทน borrow checker ตอน compile time
        let mut room = self.room.borrow_mut();
        room.messages.push(format!("[{}] {}", self.username, text));
    }
}

fn main() {
    // ---- Stack frame ของ main() เปิดขึ้น ----
    let room_data = ChatRoom {
        // "general".to_string() จอง heap buffer 7 ไบต์ (Part 14)
        name: "general".to_string(),
        // Vec::new() ยังไม่จอง heap เลยจนกว่าจะ push ตัวแรก (Part 13)
        messages: Vec::new(),
    };

    // Box::new ย้าย room_data (ที่อยู่บน stack ของ main ในตัวแปรชั่วคราว) เข้าไปบน heap
    // ก้อนใหม่ทันที แล้วคืน Box<ChatRoom> (ตัวชี้ 8 ไบต์บน stack) กลับมา (Part 27)
    let boxed_room: Box<ChatRoom> = Box::new(room_data);

    // Rc::new เอา *ChatRoom (ที่ deref มาจาก Box) มาสร้างเป็น allocation ใหม่อีกก้อนบน heap ที่
    // เก็บทั้งข้อมูล ChatRoom และตัวนับ (strong_count/weak_count) ไว้ด้วยกัน แล้วห่อด้วย RefCell
    // เพื่อให้แก้ไขได้ผ่าน shared reference (Part 28) -- สังเกตว่า *boxed_room ดึงข้อมูลออกจาก Box
    // (ย้าย ownership อีกครั้ง) ก่อนจะเข้าไปเป็นเนื้อในของ allocation ใหม่ของ Rc
    let shared_room = Rc::new(RefCell::new(*boxed_room));
    // ตรงนี้ boxed_room ถูก move เข้าไปแล้ว (ownership หมดสภาพ) ใช้ต่อไม่ได้อีก

    let alice = User {
        username: "alice".to_string(),      // heap อีกก้อนสำหรับ "alice"
        room: Rc::clone(&shared_room),       // แค่ copy ตัวชี้ + เพิ่มตัวนับ ไม่มี heap allocation ใหม่เลย
    };
    let bob = User {
        username: "bob".to_string(),
        room: Rc::clone(&shared_room),
    };

    alice.send("สวัสดีครับ");   // format! จองก้อน String ใหม่, push เข้า Vec (อาจ realloc ถ้า capacity ไม่พอ)
    bob.send("หวัดดีค่ะ");

    {
        // เปิด scope ย่อยเพื่อดู Drop ทำงานตามลำดับ
        let room_ref = shared_room.borrow();
        for msg in room_ref.messages.iter() {
            println!("{msg}");
        }
    } // room_ref (Ref<ChatRoom>) ถูก drop ที่นี่ -- ปลด borrow flag ของ RefCell คืน

    println!(
        "strong_count = {}",
        Rc::strong_count(&shared_room)
    );
} // ท้ายฟังก์ชัน main: bob, alice, shared_room ถูก drop ตามลำดับย้อนกลับ (LIFO ของ stack frame)
  // -- bob.room (Rc) drop ก่อน: strong_count ลดจาก 3 เหลือ 2, ตัวข้อมูลจริงยังไม่ถูกทำลาย
  // -- alice.room (Rc) drop ต่อ: strong_count ลดจาก 2 เหลือ 1
  // -- shared_room (Rc ตัวสุดท้าย) drop: strong_count เหลือ 0 -> เรียก drop ของ RefCell<ChatRoom>
  //    จริง ๆ -> ทำลาย ChatRoom (String name, Vec<String> messages พร้อม String ข้างในทุกตัว)
  //    -> deallocate heap ทุกก้อนที่เกี่ยวข้องกลับคืนให้ allocator ตามลำดับ "จากในสุดออกมานอกสุด"
```

ผลลัพธ์จริง:

```
[alice] สวัสดีครับ
[bob] หวัดดีค่ะ
strong_count = 3
```

#### 56.1.1 อ่านโปรแกรมนี้ด้วย "แผนที่หน่วยความจำ" แบบข้ามทุก Part

สิ่งที่สำคัญที่สุดของหัวข้อนี้ไม่ใช่ตัวโค้ด แต่คือการฝึก**อ่านโค้ด Rust ทุกบรรทัดแล้วเห็นภาพหน่วยความจำที่
เกิดขึ้นจริง**ในหัวไปพร้อมกัน — นี่คือทักษะที่สะสมมาจากทั้ง 55 บทที่ผ่านมา มารวมกันเป็นภาพเดียว:

| ขั้นตอนในโค้ด | เกิดอะไรขึ้นจริงในหน่วยความจำ | สอนไว้ที่ Part ไหน |
|---|---|---|
| `let room_data = ChatRoom { ... }` | struct ทั้งก้อนอยู่บน**stack** ของ `main()` ชั่วคราว (field `name`/`messages` เป็นแค่ "หัว" ที่ชี้ไป heap — ตัว struct เองยังไม่ได้ขึ้น heap) | Part 6 (stack vs heap), Part 13-14 (Vec/String เป็น "หัว" ที่ชี้ไป heap) |
| `"general".to_string()` | จอง heap buffer ใหม่ 7 ไบต์ คัดลอกตัวอักษรเข้าไป | Part 14 |
| `Vec::new()` | **ไม่จอง heap เลย** จนกว่าจะ push ตัวแรก (capacity เริ่มต้น = 0) | Part 13 |
| `Box::new(room_data)` | จอง heap ก้อนใหม่พอดีกับขนาด `ChatRoom`, ย้าย (move) ข้อมูลจาก stack เข้าไป, `room_data` หมดสภาพใช้งานต่อไม่ได้ | Part 27, Part 6 (move semantics) |
| `Rc::new(RefCell::new(*boxed_room))` | จอง heap ก้อนใหม่**อีกก้อน**ที่ใหญ่กว่าตัวข้อมูลเดิม (ต้องมีที่เก็บ `strong_count`/`weak_count` ด้วย) แล้ว deref+move ข้อมูลจาก `Box` เข้าไป — จุดนี้ `Box` เดิมโดน deallocate หลังข้อมูลถูกย้ายออกไปแล้ว | Part 27, Part 28 |
| `Rc::clone(&shared_room)` (x2) | **ไม่มี heap allocation ใหม่เลย** — แค่เพิ่มตัวเลข `strong_count` และคัดลอกตัวชี้ 8 ไบต์บน stack | Part 28 |
| `room.messages.push(format!(...))` | `format!` จอง heap buffer ใหม่สำหรับข้อความ; `.push()` เข้า `Vec` อาจ trigger reallocation ทั้งก้อนถ้า `len == capacity` (Part 13 อธิบายกลยุทธ์ growth แบบ exponential ไว้แล้ว) | Part 13, Part 14 |
| จบ scope ย่อย (`room_ref` หลุด) | `Ref<ChatRoom>` (RAII guard จาก Part 28/53) ถูก `Drop` — ปลด borrow flag คืนใน `RefCell` แต่**ไม่ deallocate อะไร** | Part 28 |
| จบ `main()` | `Drop` ไล่ทำลายจาก stack frame ล่างขึ้นบน (LIFO), `Rc` ตัวสุดท้ายที่ `strong_count` แตะ 0 จึงเรียก `Drop` ของข้อมูลจริงและ deallocate ทุก heap ก้อนที่เหลือ | Part 6 (scope-based drop), Part 27-28 (Drop chain ของ smart pointer) |

ตารางนี้คือ**สิ่งที่ผู้เขียน Rust ที่มีประสบการณ์เห็นในหัวโดยอัตโนมัติ**ทุกครั้งที่อ่านหรือเขียนโค้ด — ไม่ต้อง
เปิด debugger หรือ profiler ก็รู้ล่วงหน้าได้ว่าบรรทัดไหน "แพง" (heap alloc), บรรทัดไหน "ฟรี" (Rc::clone,
การ move ตัวชี้), และข้อมูลจะถูกทำลายที่จุดไหนแน่นอน — ความสามารถนี้คือสิ่งที่ทั้งบทนี้ (และทั้งโมดูล 3) กำลัง
ฝึกให้แม่นยิ่งขึ้น ไปอีกระดับด้วยการ**พิสูจน์**ว่าโมเดลทางความคิดนี้ตรงกับความเป็นจริงของ machine code แค่ไหน
ในหัวข้อ 56.2–56.4

#### 56.1.2 พิสูจน์ลำดับ Drop ให้เห็นด้วยตาตัวเอง

ตารางข้างบนบอกไว้ว่า "`Drop` ไล่ทำลายจาก stack frame ล่างขึ้นบน (LIFO)" แต่คำพูดอย่างเดียวยังไม่หนักแน่นพอ —
มาดูโค้ดเล็ก ๆ ที่พิมพ์ข้อความทุกครั้งที่ `Drop` ทำงาน เพื่อยืนยันลำดับที่แน่นอนทั้งสองมิติที่ต้องแยกให้ออก:
**ลำดับของตัวแปร local หลาย ๆ ตัวใน scope เดียวกัน** กับ **ลำดับของ field หลาย ๆ ตัวภายใน struct เดียวกัน**
— สองมิตินี้มีกฎที่**ต่างกัน**และมักถูกจำสลับกันถ้าไม่เคยเห็นหลักฐานจริง:

```rust
struct Logged(&'static str);

impl Drop for Logged {
    fn drop(&mut self) {
        println!("dropping {}", self.0);
    }
}

struct Outer {
    first: Logged,
    second: Logged,
}

fn main() {
    let _a = Logged("a (local variable ตัวแรก)");
    let _b = Logged("b (local variable ตัวที่สอง)");

    let _outer = Outer {
        first: Logged("outer.first"),
        second: Logged("outer.second"),
    };

    println!("--- จบ main กำลังจะ drop ---");
}
```

ผลลัพธ์จริง:

```
--- จบ main กำลังจะ drop ---
dropping outer.first
dropping outer.second
dropping b (local variable ตัวที่สอง)
dropping a (local variable ตัวแรก)
```

สองกฎที่ยืนยันได้จากผลลัพธ์นี้:

1. **ตัวแปร local ในฟังก์ชันเดียวกัน drop ในลำดับ "ย้อนกลับ" จากที่ประกาศ (reverse declaration order)** —
   `_outer` ประกาศทีหลังสุดจึงถูก drop **ก่อน** ตามด้วย `_b` แล้ว `_a` (ประกาศก่อนสุด drop หลังสุด) — นี่คือ
   กฎ LIFO (Last-In-First-Out) แบบเดียวกับ stack เอง (ตรงกับที่ Part 6 อธิบายไว้ว่า stack frame ทำงานแบบ
   LIFO)
2. **field ภายใน struct เดียวกัน drop ในลำดับ "ตามที่ประกาศ" (declaration order ตรงตัว ไม่ย้อนกลับ)** —
   `outer.first` ถูก drop **ก่อน** `outer.second` ตรงตามลำดับที่เขียนไว้ใน `struct Outer { first, second }`
   ทุกตัวอักษร ไม่ใช่ย้อนกลับแบบตัวแปร local

**ทำไมกฎทั้งสองต่างกัน?** เพราะตัวแปร local คือ "การเปิด stack frame ใหม่ทีละชั้น" (แต่ละ `let` คือการจอง
พื้นที่ใหม่บน stack ต่อจากตัวก่อนหน้า) การ drop ต้อง**ปิดจากชั้นบนสุดลงมา**เสมอ (จะปิดชั้นล่างก่อนชั้นบนไม่ได้
เพราะ stack ทำงานแบบนั้นไม่ได้ทางกายภาพ) แต่ field ภายใน struct เดียวกันไม่ได้เป็น "ชั้น stack" ที่ซ้อนกัน —
มันคือ**หน่วยความจำก้อนเดียวที่มีหลายส่วนอยู่ข้างในพร้อมกัน**ตั้งแต่แรก (ไม่มีลำดับการ "เปิด" ที่ต้องย้อนกลับ)
ดังนั้น Rust จึงเลือก drop field ตามลำดับที่ประกาศตรง ๆ ซึ่งเป็นพฤติกรรมที่**คาดเดาได้และมีเอกสารรับประกัน**
(ระบุไว้ใน Rust Reference ว่าเป็นส่วนหนึ่งของ "drop order" ที่โปรแกรมเมอร์พึ่งพาได้จริง ไม่ใช่รายละเอียด
implementation ที่เปลี่ยนได้ในอนาคต) — ความรู้นี้สำคัญมากเวลาออกแบบ struct ที่มี resource หลายตัวที่ต้อง
ปลดปล่อยตามลำดับที่ถูกต้อง (เช่น connection handle ที่ต้อง flush ข้อมูลก่อนปิด socket จริง)

#### 56.1.3 เปรียบเทียบกับภาษาอื่น: ใครรับผิดชอบเรื่องนี้บ้าง

ก่อนไปพิสูจน์ zero-cost abstraction ต่อ ลองถอยออกมามองภาพกว้างว่าภาษาอื่นจัดการกับ "เรื่องราวหน่วยความจำ
เต็มรูปแบบ" แบบในหัวข้อ 56.1.1 นี้อย่างไร จะช่วยให้เห็นชัดว่าโมเดลของ Rust อยู่ตรงไหนของสเปกตรัม:

- **C/C++**: โปรแกรมเมอร์ต้องเรียก `malloc`/`free` (C) หรือ `new`/`delete` (C++) **เอง**ทุกครั้ง ไม่มี compiler
  คอยตรวจสอบว่าเรียก `free` ครบทุกก้อนหรือไม่ ก็อปปี้ pointer ไปมาได้อย่างอิสระโดยไม่มีการติดตาม ownership เลย
  — ตารางแบบในหัวข้อ 56.1.1 ยังเขียนได้ แต่ต้อง**เขียนด้วยมือและรักษาให้ตรงกับโค้ดจริงเองตลอดเวลา**ไม่มี
  compiler ช่วยยืนยัน ถ้าลืมบรรทัดใดบรรทัดหนึ่ง (ลืม `free`, เรียก `free` ซ้ำสองครั้ง, หรือใช้ pointer หลัง
  `free` ไปแล้ว) จะได้ memory leak หรือ use-after-free ที่ compiler ไม่เตือนอะไรเลย — Rust ย้าย"การตรวจสอบว่า
  ตารางนี้ถูกต้อง"จากความรับผิดชอบของโปรแกรมเมอร์ไปเป็นความรับผิดชอบของ borrow checker แทน (Part 7)
- **Java/C#/Go/Python/JavaScript (ภาษาที่มี Garbage Collector)**: โปรแกรมเมอร์**ไม่ต้องเขียนตารางแบบ 56.1.1
  เองเลย** — GC วิ่งเป็นระยะ ๆ (หรือ trigger ตามเงื่อนไข) เพื่อตรวจสอบว่า object ไหนไม่มีใคร reference ถึงแล้ว
  แล้วเก็บกวาดให้อัตโนมัติ ข้อดีคือเขียนง่ายกว่ามาก ไม่มีทาง use-after-free หรือ double-free ได้เลย (GC เป็น
  ผู้รับผิดชอบทั้งหมด) แต่ข้อเสียคือ **เวลาที่แน่นอนว่า object หนึ่งจะถูกทำลายเมื่อไหร่นั้นทำนายไม่ได้ 100%**
  (GC อาจหยุดโปรแกรมชั่วครู่เพื่อเก็บกวาด เรียกว่า "GC pause" ซึ่งเป็นปัญหาจริงสำหรับงานที่ sensitive ต่อ
  latency เช่น เกมหรือระบบ real-time) และมี**runtime overhead ต่อเนื่อง**ตลอดเวลาที่โปรแกรมรัน (ต้อง track
  ว่า object ไหนยังมีคนใช้อยู่ตลอดเวลา) — ตารางในหัวข้อ 56.1.1 สำหรับภาษาเหล่านี้จะมีคอลัมน์ "จบ scope"
  ว่างเปล่าไปเลย เพราะการทำลาย object ไม่ได้ผูกกับ scope แต่ผูกกับจังหวะที่ GC ทำงาน ซึ่งอาจช้ากว่าตอนที่ตัวแปร
  หลุด scope ไปมากได้
- **Rust**: อยู่ระหว่างสองขั้วนี้อย่างชัดเจน — โปรแกรมเมอร์**ไม่ต้องเรียก free เองเลย** (เหมือนภาษาที่มี GC)
  เพราะ `Drop` ถูกเรียกอัตโนมัติตามกฎ scope/ownership ที่ Part 6-7 สอนไว้ แต่**เวลาที่แน่นอน**ที่ resource
  จะถูกปลดปล่อยนั้น**ทำนายได้ 100% ตั้งแต่ compile time** (เหมือนภาษาที่ไม่มี GC) เพราะมันผูกกับ scope ที่เห็น
  ตรง ๆ ในโค้ด ไม่ใช่จังหวะสุ่มของ garbage collector — นี่คือเหตุผลที่ตารางในหัวข้อ 56.1.1 เขียนคอลัมน์ "จบ
  `main()`" ได้อย่างแม่นยำเป๊ะว่า `Drop` อะไรจะรันตามลำดับไหน **โดยไม่มี runtime cost ของ GC เลยแม้แต่นิดเดียว**
  (ไม่มี background thread ที่ต้องคอย scan heap, ไม่มี GC pause) — ผลลัพธ์คือ Rust ได้ความปลอดภัยของ memory
  management ระดับเดียวกับภาษาที่มี GC (ไม่มี use-after-free/double-free ให้เกิดขึ้นได้เลยในโค้ด safe) โดยที่
  ยังรักษาความสามารถในการทำนายเวลา (deterministic timing) แบบเดียวกับ C/C++ ไว้ครบ — นี่คือสิ่งที่ Part 1
  เกริ่นไว้ว่า Rust "แตกต่างจากภาษาอื่น" และบทนี้คือจุดที่เราเห็นเหตุผลเชิงกลไกเต็มรูปแบบว่าทำไมมันถึงทำได้

### 56.2 พิสูจน์ Zero-cost ข้อที่ 1: Generic Monomorphization ให้ Machine Code เหมือนโค้ดที่เขียนมือ 100%

Part 18 สอนไว้ว่า compiler สร้างสำเนาโค้ดที่เป็นรูปธรรม (concrete) แยกกันสำหรับทุกชนิดข้อมูลที่ generic function
ถูกใช้จริง เรียกว่า **monomorphization** — และยืนยันด้วยเครื่องมือ `nm` ว่ามี symbol แยกกันจริงในไบนารี แต่ Part
18 **ไม่ได้พิสูจน์ว่าสำเนานั้นมี machine code เหมือนกับที่เขียนมือ 100% หรือไม่** มันอาจจะสร้างโค้ดที่ทำงานถูก
แต่ "ช้ากว่า" ก็เป็นไปได้ในทางทฤษฎี — บทนี้คือจุดที่เราจะพิสูจน์ให้เห็นด้วยตาตัวเองว่า**ไม่มีความต่างแม้แต่ไบต์
เดียว**

#### 56.2.1 วิธีตรวจสอบในสภาพแวดล้อมนี้ — หมายเหตุความซื่อตรงเรื่องเครื่องมือ

ก่อนเข้าโค้ด ต้องพูดตรง ๆ ก่อนว่า sandbox ที่ใช้เขียนบทนี้**ไม่มี** `cargo-asm` subcommand ติดตั้งไว้ (ลองรัน
`cargo asm` แล้วได้ error `no such command: asm`) และไม่มีการเชื่อมต่อไปยังบริการเปรียบเทียบ assembly ออนไลน์
อย่าง Compiler Explorer (godbolt.org) เครื่องมือที่มีอยู่จริงและใช้ได้คือ `objdump`/`nm` จาก GNU Binutils
(เวอร์ชัน 2.42) ซึ่ง**เป็นหลักฐานที่แน่นหนากว่าด้วยซ้ำ** เพราะเป็นการอ่าน machine code จาก**ไบนารีที่ compile
และ link จริง**ด้วย `cargo build --release` (opt-level 3 ตาม profile release มาตรฐาน — Part 35/54 สอนไว้แล้ว
ว่า `cargo bench`/`--release` ใช้ profile นี้) ไม่ใช่การอ่าน IR กลาง ๆ ที่อาจถูก optimize เพิ่มอีกชั้นตอน link
time

วิธีการ: เขียนฟังก์ชันคู่เทียบกันไว้ในไฟล์เดียว กำกับทุกฟังก์ชันด้วย `#[no_mangle]` (กันไม่ให้ compiler เปลี่ยน
ชื่อ symbol เป็นชื่อที่ mangle แล้วอ่านยาก) และ `#[inline(never)]` (กันไม่ให้ compiler inline ฟังก์ชันเข้าไปใน
จุดที่เรียกจนไม่เหลือ symbol แยกให้ตรวจ) แล้ว compile ด้วย `cargo build --release` จากนั้นใช้ `nm`/`objdump -d`
อ่าน machine code ของแต่ละ symbol ออกมาเทียบกันตรง ๆ

```rust
// src/bin/asmcheck.rs
use std::ops::Add;

// ฟังก์ชัน generic ธรรมดา ที่ inline(always) เพื่อให้ compiler มีโอกาสยุบมันเข้าไปในตัวที่เรียก
// (ถ้าไม่ inline เข้าไปเลย เราจะเห็นแค่ "call ไปยังอีกฟังก์ชัน" ซึ่งไม่ช่วยพิสูจน์อะไร)
#[inline(always)]
fn generic_add<T: Add<Output = T>>(a: T, b: T) -> T {
    a + b
}

// wrapper ที่ #[no_mangle] + #[inline(never)] -- ตัวนี้แหละที่เราจะเอา symbol ไปเทียบกับ
// specific_add_i32 ด้านล่าง เพราะเนื้อในของมันคือ generic_add<i32> ที่ inline เข้ามาแล้ว
#[no_mangle]
#[inline(never)]
pub fn generic_add_i32(a: i32, b: i32) -> i32 {
    generic_add(a, b)
}

// ฟังก์ชันที่เขียนมือเจาะจง i32 ตรง ๆ ไม่มี generic เกี่ยวข้องเลยแม้แต่นิดเดียว
#[no_mangle]
#[inline(never)]
pub fn specific_add_i32(a: i32, b: i32) -> i32 {
    a + b
}

fn main() {
    println!("generic_add_i32(3, 4)  = {}", generic_add_i32(3, 4));
    println!("specific_add_i32(3, 4) = {}", specific_add_i32(3, 4));
}
```

compile ด้วย `cargo build --release` แล้วดู symbol table ด้วย `nm`:

```
$ nm -C target/release/asmcheck | grep add_i32
00000000000149c0 T generic_add_i32
00000000000149c0 T specific_add_i32
```

**หยุดดูตัวเลขนี้ให้ดี ๆ** — `generic_add_i32` และ `specific_add_i32` มี**address เดียวกันเป๊ะ**
(`0x149c0`) นี่ไม่ใช่แค่ "โค้ดคล้ายกัน" แต่คือ**สองชื่อที่ชี้ไปยังตำแหน่ง machine code จุดเดียวกันในไบนารี**
— linker ตรวจพบว่าทั้งสองฟังก์ชันมี machine code เหมือนกันทุกไบต์ (เทคนิคที่เรียกว่า **Identical Code
Folding — ICF**) จึงยุบให้เหลือชุดคำสั่งเดียวแล้วให้สอง symbol ชี้ไปที่เดียวกัน — ยืนยันด้วย `objdump -d`:

```
$ objdump -d --no-show-raw-insn target/release/asmcheck | grep -A2 "<generic_add_i32>:"
00000000000149c0 <generic_add_i32>:
   149c0:	lea    (%rdi,%rsi,1),%eax
   149c3:	ret
```

instruction เดียว (`lea` ใช้ทำการบวกโดยไม่ต้องผ่าน ALU ธรรมดา — เทคนิคที่ compiler เลือกใช้เองสำหรับการบวก
เลขจำนวนเต็มแบบนี้) แล้ว `ret` — **นี่คือ machine code ที่สั้นและตรงที่สุดที่เป็นไปได้สำหรับ `a + b` บน
`i32`** ไม่มีการเรียกฟังก์ชันซ้อน ไม่มี overhead ของ generic เหลืออยู่แม้แต่ไบต์เดียว เพราะ `objdump` อ่าน
address เดียวกันสำหรับทั้งสอง symbol จึงแสดงผลเหมือนกันเป๊ะไม่ว่าจะสั่งดู `generic_add_i32` หรือ
`specific_add_i32`

**นี่คือหลักฐานที่แน่นหนาที่สุดที่เป็นไปได้สำหรับคำสัญญา "zero-cost abstraction" ในมิติของ generics**: ไม่ใช่
แค่ "compile เร็วพอ ๆ กัน" หรือ "benchmark ออกมาใกล้เคียงกัน" (ซึ่งยังพอมี noise ปนได้ตามที่ Part 54 สอนไว้)
แต่คือ **linker เห็นว่ามันเป็น machine code ชุดเดียวกันตัวต่อตัว** ไม่มีทางที่จะ "ช้ากว่ากันนิดหน่อย" ได้เลย
เพราะมันคือคำสั่ง CPU ชุดเดียวกันที่ถูกรันจริงไม่ว่าจะเรียกผ่านชื่อไหน

#### 56.2.2 หมายเหตุความซื่อตรงเรื่องวิธีวัด: `#[inline(never)]` ไม่ได้ทำให้ผลลัพธ์ผิดเพี้ยนจากการใช้งานจริง

ก่อนไปหัวข้อถัดไป ต้องตอบคำถามที่อาจผุดขึ้นมา: **"ในโค้ดจริงเราไม่เขียน `#[inline(never)]` ครอบทุกฟังก์ชัน
generic แบบนี้ แล้วผลการทดลองนี้ยังเชื่อถือได้อยู่ไหมสำหรับโค้ดที่เขียนกันจริง ๆ ?"** คำตอบคือ **เชื่อถือได้
เต็มร้อย และในความเป็นจริงแล้วผลลัพธ์ในโค้ดจริงจะยิ่ง "ดีกว่า" ตัวอย่างในหัวข้อนี้ด้วยซ้ำ** — เหตุผลคือ
`#[inline(never)]` ในตัวอย่างนี้ถูกใส่ไว้ที่ **wrapper function** (`generic_add_i32`/`specific_add_i32`)
เท่านั้น เพื่อจุดประสงค์เดียว: **บังคับให้แต่ละอันยังเหลือเป็น symbol แยกในไบนารีให้เราชี้ตัวตรวจสอบได้** ไม่ใช่
เพื่อป้องกันการ optimize เนื้อในของมัน — สังเกตว่า `generic_add` (ฟังก์ชัน generic ตัวจริงที่เรียกข้างใน
wrapper) มี `#[inline(always)]` กำกับไว้ต่างหาก และมัน**ถูก inline เข้าไปใน wrapper สำเร็จ**จริง (ถ้าไม่ถูก
inline เราจะเห็น `call` ไปยังอีก symbol ในผลลัพธ์ `objdump` แต่เราไม่เห็นเลย มีแค่ `lea`/`ret` สองคำสั่งตรง ๆ) —
นี่คือหลักฐานว่า **การ inline เกิดขึ้นได้ปกติทุกประการเหมือนโค้ดจริงที่ไม่มี `#[inline(never)]` เลย** เราแค่
ต้อง "ห้าม inline ที่ชั้นนอกสุด" เพื่อให้มี symbol เหลือให้ตรวจ ไม่ได้ห้ามที่ชั้นในซึ่งเป็นจุดที่การพิสูจน์
zero-cost เกิดขึ้นจริง — ในโค้ด production ทั่วไปที่ไม่มี `#[inline(never)]` เลยแม้แต่จุดเดียว compiler มีอิสระ
มากกว่านี้อีกในการ optimize (เช่น อาจ inline `generic_add_i32` เข้าไปในจุดที่เรียกมันต่ออีกชั้นได้เลย ทำให้
เหลือแค่ `lea` ตัวเดียวฝังอยู่ในฟังก์ชันที่ใหญ่กว่า ไม่มี `call`/`ret` เหลือด้วยซ้ำ) — เทคนิค `#[inline(never)]`
ในบทนี้จึงเป็นแค่ **"เครื่องมือสำหรับการวัด" (measurement scaffolding)** ไม่ใช่ส่วนหนึ่งของคำแนะนำให้เขียนโค้ด
จริงแบบนั้น — Part 54 พูดหลักการเดียวกันไว้แล้วสำหรับ `black_box`: มันมีไว้เพื่อ "ป้องกันไม่ให้เครื่องมือวัด
บิดเบือนสิ่งที่กำลังวัด" ไม่ใช่สิ่งที่ต้องใส่ไว้ในโค้ด production จริง

### 56.3 พิสูจน์ Zero-cost ข้อที่ 2: Iterator Chain เทียบกับ Manual Loop

Part 25-26 สอนไว้ว่า `.filter().map().sum()` ควร "เร็วเท่ากับ" การเขียน `for` loop มือเอง เพราะ iterator
adaptor ทุกตัวถูกออกแบบให้ inline และ optimize รวมกันเป็น loop เดียวโดย LLVM (เทคนิคที่เรียกว่า **loop fusion**
ที่ Part 26 เกริ่นชื่อไว้) มาพิสูจน์ด้วยวิธีเดียวกับหัวข้อก่อน:

```rust
#[no_mangle]
#[inline(never)]
pub fn sum_iter_chain(data: &[i64]) -> i64 {
    data.iter().filter(|&&x| x % 2 == 0).map(|&x| x * 3).sum()
}

#[no_mangle]
#[inline(never)]
pub fn sum_manual_loop(data: &[i64]) -> i64 {
    let mut total: i64 = 0;
    for &x in data {
        if x % 2 == 0 {
            total += x * 3;
        }
    }
    total
}
```

ทั้งสองฟังก์ชันทำสิ่งเดียวกันทางตรรกะ: กรองเลขคู่ คูณด้วย 3 แล้วรวมกัน ตัวแรกเขียนด้วย iterator chain (สไตล์ที่
Part 25-26 สนับสนุน) ตัวที่สองเขียนด้วย `for` loop มือเอง compile ด้วย `--release` แล้วดู `objdump -d`:

```
0000000000014a50 <sum_iter_chain>:
   14a64:	movabs $0xffffffffffffffe,%rdx
   ...
   14a80:	mov    (%rdi,%rcx,8),%r9
   14a84:	mov    0x8(%rdi,%rcx,8),%r10
   14a89:	test   $0x1,%r9b
   14a8d:	lea    (%r9,%r9,2),%r9
   14a91:	cmovne %r8,%r9
   14a95:	add    %rax,%r9
   ...
```

```
0000000000014ad0 <sum_manual_loop>:
   14ad8:	movabs $0x1fffffffffffffff,%rax
   ...
   14b10:	mov    (%rdi),%r8
   14b13:	mov    0x8(%rdi),%r9
   14b17:	test   $0x1,%r8b
   14b1b:	lea    (%r8,%r8,2),%r8
   14b1f:	cmovne %rsi,%r8
   14b23:	add    %rax,%r8
   ...
```

**สังเกตแกนกลางของ instruction ทั้งสองชุดให้ดี ๆ** — ทั้งสองฟังก์ชันใช้**เทคนิคระดับ instruction เดียวกันเป๊ะ**
สามจุดสำคัญ:

1. **`lea (%r9,%r9,2),%r9`** — ทั้งคู่ใช้ `lea` (load-effective-address) ทำการคูณด้วย 3 (`r9 + r9*2 = r9*3`)
   โดยไม่ผ่านคำสั่งคูณ (`imul`) เลย เพราะ LLVM รู้ว่า `lea` เร็วกว่าสำหรับตัวคูณคงที่เล็ก ๆ แบบนี้ — ทั้งสอง
   เวอร์ชันได้ optimization ตัวนี้เหมือนกัน
2. **`test $0x1,%r9b` ตามด้วย `cmovne`** — นี่คือ**การกำจัด branch ทิ้งทั้งหมด** เงื่อนไข `if x % 2 == 0`
   ที่ดูเหมือนต้อง jump ในโค้ด Rust ถูก compiler แปลงเป็น**conditional move** (`cmovne` — ย้ายค่าแบบมีเงื่อนไข
   โดยไม่มี branch จริงให้ CPU ต้องทำนายทิศทาง) ทั้งคู่ — เทคนิคนี้เร็วกว่า branch จริงมากสำหรับ pattern ที่
   ทำนายทิศทางได้ยาก (เลขคู่/คี่สุ่ม ๆ) เพราะไม่มี branch misprediction penalty เลย ทั้ง iterator chain และ
   manual loop ได้รับ optimization ระดับนี้เท่ากันทุกประการ
3. **โครงสร้าง loop ที่ประมวลผลหลาย element ต่อรอบ** (loop unrolling) — ทั้งคู่ถูก unroll โดย LLVM เหมือนกัน
   แต่**degree การ unroll ต่างกัน**: `sum_iter_chain` unroll 2 elements/รอบ ส่วน `sum_manual_loop` unroll 4
   elements/รอบ

จุดที่ 3 คือความละเอียดที่ต้องพูดอย่างซื่อตรง: **assembly ทั้งสองชุดไม่ได้ byte-เหมือนกัน 100%** ต่างจากหัวข้อ
56.2 ที่ linker ยุบเป็น symbol เดียวกันเป๊ะ ความต่างของ unrolling degree นี้เป็นผลจาก**heuristic ภายในของ
LLVM's loop unroller** ที่ตัดสินใจจากรูปร่างของ LLVM IR ที่ต่างกันเล็กน้อยระหว่าง iterator adaptor chain กับ
`for` loop ธรรมดา (ตัว iterator ส่ง IR ที่มีการเรียก `next()` ซ้อนกันหลายชั้นก่อนถูก inline รวม ส่วน `for`
loop ธรรมดาให้ IR ที่ตรงไปตรงมากว่าตั้งแต่แรก) — **นี่ไม่ใช่ "ต้นทุนของ abstraction" แต่เป็นความไม่แน่นอนของ
compiler heuristic ที่เกิดได้แม้เขียนโค้ด manual loop สองแบบที่ต่างกันเล็กน้อยก็ตาม**

ข้อสรุปที่ถูกต้องและซื่อตรงที่สุดจากหลักฐานนี้คือ: **ไม่มี "ภาษี" ที่ inherent ต่อการใช้ iterator chain** —
core computation ที่ทำจริง (การคูณ, การกำจัด branch) เหมือนกันทุกประการ ความต่างที่เหลือเป็นแค่ตัวเลข "จำนวน
element ต่อรอบ" ที่ compiler เลือกเอง ซึ่งเป็นรายละเอียดระดับ micro-optimization ที่ไม่คงที่อยู่แล้วแม้จะเทียบ
manual loop สองรูปแบบกันเอง — นี่คือความหมายที่แม่นยำของคำว่า "zero-cost": **ไม่มีค่าใช้จ่ายที่ abstraction
เพิ่มเข้ามาเหนือกว่าที่จำเป็นสำหรับ computation เดียวกัน** ไม่ได้แปลว่า "compiler จะ optimize ทุกอย่างเหมือน
กันเป๊ะเสมอทุกกรณี" ซึ่งเป็นข้อความที่แข็งเกินความจริง

### 56.4 พิสูจน์ Zero-cost ข้อที่ 3: `const fn` — ค่าที่หายไปจาก Runtime อย่างสิ้นเชิง

สองหัวข้อก่อน พิสูจน์ว่า generic/iterator ไม่เสียต้นทุนเทียบกับโค้ดที่เขียนมือ หัวข้อนี้จะพิสูจน์คำสัญญาที่
**แรงกว่านั้นอีกขั้น**: บางการคำนวณไม่ใช่แค่ "เร็วเท่าโค้ดมือ" แต่**ไม่ต้องรันตอน runtime เลยแม้แต่ครั้งเดียว**
เพราะ `rustc` คำนวณให้เสร็จแล้วตั้งแต่ตอน compile — Part 18 เกริ่น const generics ไว้บ้าง (`Grid<const N:
usize>`) แต่ยังไม่ได้พิสูจน์ด้วย assembly ว่า "คำนวณตอน compile time" หมายความว่าอะไรจริง ๆ ในระดับเครื่อง

```rust
const fn fib_const(n: u32) -> u64 {
    let mut a: u64 = 0;
    let mut b: u64 = 1;
    let mut i = 0;
    while i < n {
        let next = a + b;
        a = b;
        b = next;
        i += 1;
    }
    a
}

// เขียน const context (const, static, ขนาด array) -> รับประกันว่าคำนวณตอน compile time เท่านั้น
const FIB_30: u64 = fib_const(30);

#[no_mangle]
#[inline(never)]
pub fn get_fib_30() -> u64 {
    FIB_30
}

// เวอร์ชันที่รับ n เป็นค่าที่รู้แค่ตอน runtime (ไม่ใช่ const context) -- เทียบให้เห็นความต่าง
#[no_mangle]
#[inline(never)]
pub fn get_fib_30_runtime(n: u32) -> u64 {
    fib_const(n)
}
```

ทั้งสองฟังก์ชันเรียก `fib_const` เหมือนกัน แต่ `get_fib_30` เรียกด้วย argument ที่เป็น**ค่าคงที่ตอน compile
time** (`FIB_30` ที่คำนวณจาก `fib_const(30)` ไว้แล้วในบรรทัดที่ประกาศ `const`) ส่วน `get_fib_30_runtime` รับ
`n` เป็น parameter ปกติที่ไม่รู้ค่าจนกว่าจะรันจริง — มาดูว่า `main()` ที่เรียกทั้งสองฟังก์ชันนี้ compile ออกมา
เป็นอะไร:

```
   1490c:	movq   $0xcb228,0x18(%rsp)   ; <- คือค่าจาก get_fib_30() ที่ inline+ยุบไปแล้ว
   ...
   14938:	movl   $0x1e,0x8(%rsp)      ; n = 30 (0x1e) เตรียมส่งเข้า get_fib_30_runtime
   14945:	mov    0x8(%rsp),%edi
   14949:	call   149d0 <get_fib_30_runtime>
```

**นี่คือหลักฐานที่ตรงที่สุดของ "zero runtime cost":** บรรทัด `movq $0xcb228,0x18(%rsp)` คือค่า `0xcb228` =
`832040` ในเลขฐานสิบ (คือ `fib_const(30)` ที่ถูกต้อง) **ถูกเขียนเป็นค่าคงที่ตรง ๆ ลงในคำสั่ง CPU เลย** — ไม่มี
การเรียก `get_fib_30()` เหลืออยู่ในไบนารีแม้แต่จุดเดียว (ลองหา symbol `get_fib_30` ด้วย `nm` จะไม่พบเลย —
มันถูกลบทิ้งไปทั้งฟังก์ชัน เพราะ compiler พิสูจน์ได้ว่ามันคืนค่าคงที่เดียวเสมอ) เทียบกับ
`get_fib_30_runtime(30)` ที่**ยังมี `call` จริง**ไปยังฟังก์ชันที่เต็มไปด้วย loop คำนวณ fibonacci แบบ
iterative จริง ๆ ตอนรัน (ดูได้จาก disassembly ของ `get_fib_30_runtime` ที่มี loop คำนวณเต็มรูปแบบ — ต่างจาก
`get_fib_30` ที่ "หายไปเลย" ทั้งฟังก์ชัน)

**ทำไม `get_fib_30_runtime` ทำแบบเดียวกันไม่ได้ทั้งที่เรียก `fib_const` ซึ่งเป็น const fn เหมือนกัน?** เพราะ
`const fn` ไม่ได้แปลว่า "ทุกครั้งที่เรียกจะคำนวณตอน compile time เสมอ" มันแปลว่า **"สามารถ**ถูกเรียกใน const
context ได้ (ถ้าผู้เรียกใส่มันไว้ใน const context จริง)**"** — ถ้าเรียกด้วย argument ที่รู้ค่าแค่ตอน runtime
(`n: u32` ที่มาจาก parameter ธรรมดา) compiler ก็ยังต้องคำนวณตอน runtime เหมือนฟังก์ชันปกติทุกประการ (นี่คือ
กับดักที่ 4 ในหัวข้อกับดักท้ายบท) — ความ "zero-cost" ของ `const fn` จึงเป็นแบบมีเงื่อนไข: **ฟรีสมบูรณ์เมื่อ
argument ทั้งหมดเป็นค่าคงที่ตอน compile time เท่านั้น** เมื่อเป็นแบบนั้น มันคือรูปแบบ zero-cost ที่แรงที่สุดที่
เป็นไปได้ในภาษาโปรแกรม: **งานคำนวณทั้งหมดถูกทำโดย `rustc` เอง ไม่ใช่โดย CPU ของผู้ใช้ตอนรันโปรแกรมแม้แต่รอบ
เดียว**

### 56.5 ต้นทุนจริงของ Stack กับ Heap Allocation: วัดด้วย `criterion`

สามหัวข้อก่อนพิสูจน์ว่า **abstraction ไม่มีต้นทุนแอบแฝง** — แต่นั่นไม่ได้แปลว่า**ทุกอย่างในโปรแกรม Rust
ไม่มีต้นทุน** Part 6 บอกไว้เป็นคำกล่าวเชิงคุณภาพว่า "heap allocation แพงกว่า stack allocation" แต่ไม่เคยให้
ตัวเลข มาวัดตัวเลขนั้นให้เห็นจริงด้วย `criterion` (ตามวิธีที่ Part 54 สอนไว้ครบแล้ว จะไม่สอนการติดตั้งซ้ำ):

```rust
// benches/alloc_bench.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};

#[derive(Clone, Copy)]
struct Small { a: u64, b: u64, c: u64, d: u64 } // 32 ไบต์ -- ยืนยันด้วย size_of ก่อนรัน

fn work_stack(n: u64) -> u64 {
    let mut acc: u64 = 0;
    for i in 0..n {
        // สร้างค่าบน stack ล้วน ๆ ไม่มีการเรียก allocator เลยแม้แต่ครั้งเดียวตลอด loop
        let s = Small { a: i, b: i + 1, c: i + 2, d: i + 3 };
        acc = acc.wrapping_add(black_box(s).a);
    }
    acc
}

fn work_heap(n: u64) -> u64 {
    let mut acc: u64 = 0;
    for i in 0..n {
        // ทุกรอบ Box::new เรียก allocator (malloc) จองพื้นที่ heap ใหม่ 32 ไบต์
        // แล้วพอ s หลุด scope ก็เรียก deallocator (free) คืนพื้นที่ทันที (Part 6/27)
        let s = Box::new(Small { a: i, b: i + 1, c: i + 2, d: i + 3 });
        acc = acc.wrapping_add(black_box(&s).a);
    }
    acc
}

fn work_vec_preallocated(n: u64) -> u64 {
    // เทียบเพิ่ม: Vec ที่จอง capacity ล่วงหน้าครั้งเดียว (heap alloc ครั้งเดียวทั้ง loop)
    let mut v: Vec<Small> = Vec::with_capacity(n as usize);
    for i in 0..n {
        v.push(Small { a: i, b: i + 1, c: i + 2, d: i + 3 });
    }
    black_box(&v).iter().map(|s| s.a).sum()
}

fn bench_alloc(c: &mut Criterion) {
    let n: u64 = 2000;
    let mut group = c.benchmark_group("stack_vs_heap_alloc_2000_items");
    group.bench_function("stack_local_no_alloc", |b| b.iter(|| work_stack(black_box(n))));
    group.bench_function("box_new_per_item_heap_alloc", |b| b.iter(|| work_heap(black_box(n))));
    group.bench_function("vec_with_capacity_single_alloc", |b| {
        b.iter(|| work_vec_preallocated(black_box(n)))
    });
    group.finish();
}

criterion_group!(benches, bench_alloc);
criterion_main!(benches);
```

รันด้วย `cargo bench --bench alloc_bench` จริงในสภาพแวดล้อมที่เขียนบทนี้ (Linux, `--release`, ไม่มีโหลดอื่น
แทรกขณะรัน) ได้ผลลัพธ์จริงดังนี้ (mean จากช่วงความเชื่อมั่นที่ criterion รายงาน):

```
stack_vs_heap_alloc_2000_items/stack_local_no_alloc
                        time:   [2.0688 µs 2.1094 µs 2.1621 µs]
stack_vs_heap_alloc_2000_items/box_new_per_item_heap_alloc
                        time:   [23.860 µs 24.819 µs 25.973 µs]
stack_vs_heap_alloc_2000_items/vec_with_capacity_single_alloc
                        time:   [5.7539 µs 5.8469 µs 5.9751 µs]
```

แปลงเป็นต้นทุนต่อ element หนึ่งตัว (หาร 2000):

| เวอร์ชัน | เวลาทั้งหมด (2000 items) | เวลาต่อ item | เทียบกับ stack |
|---|---|---|---|
| `stack_local_no_alloc` | ~2.11 µs | ~1.05 ns | 1× (baseline) |
| `vec_with_capacity_single_alloc` | ~5.85 µs | ~2.92 ns | ~2.8× ช้ากว่า |
| `box_new_per_item_heap_alloc` | ~24.8 µs | ~12.4 ns | **~11.8× ช้ากว่า** |

ตัวเลขนี้ยืนยัน**เชิงปริมาณ**สิ่งที่ Part 6 บอกไว้แบบคุณภาพ: **การเรียก heap allocator ต่อ element หนึ่งครั้ง
แพงกว่าการใช้ stack ล้วน ๆ ราวสิบเท่าตัว** สำหรับข้อมูลขนาดเล็กแบบนี้ (32 ไบต์) — เหตุผลเชิงกลไกที่ทำให้ต่าง
กันมากขนาดนี้มีสามข้อรวมกัน:

1. **Bookkeeping ของ allocator** — `malloc`/`free` ต้องค้นหา/อัปเดต free-list ภายในของตัวเอง (โครงสร้างข้อมูล
   ที่ track ว่าหน่วยความจำก้อนไหนว่าง/ไม่ว่าง) ทุกครั้งที่เรียก ต่างจาก stack ที่แค่ขยับ stack pointer ขึ้น/ลง
   หนึ่งค่า (เร็วระดับ 1-2 CPU cycle)
2. **โอกาสเรียก syscall ของระบบปฏิบัติการ** — ถ้า allocator ในเธรดนั้นไม่มีหน่วยความจำว่างพอในก้อนที่ตัวเองถือ
   ไว้แล้ว (thread-local arena) ต้องขอเพิ่มจาก OS ผ่าน syscall (`brk`/`mmap` บน Linux) ซึ่งช้ากว่าคำสั่ง CPU
   ธรรมดาหลายพันเท่า (ถึงแม้ allocator สมัยใหม่จะพยายาม batch การขอเพิ่มไว้ล่วงหน้าเพื่อลดความถี่ของ syscall)
3. **Cache locality ที่แย่กว่า** — ข้อมูลบน heap ที่จองทีละก้อนกระจัดกระจายกันในหน่วยความจำ (ไม่ต่อเนื่องกัน)
   ทำให้ CPU cache ทำงานได้ไม่ดีเท่าข้อมูลบน stack ที่อยู่ติดกันเสมอ (จะเห็นเรื่องนี้ลึกกว่านี้ในหัวข้อ 56.8)

ที่น่าสนใจคือ `vec_with_capacity_single_alloc` อยู่**กึ่งกลาง**: จอง heap แค่**ครั้งเดียว**สำหรับ buffer ทั้ง
ก้อน (ไม่ใช่จองต่อ element เหมือน `Box::new` ในทุกรอบ) จึงเลี่ยงต้นทุนของข้อ 1-2 ไปได้เกือบหมด เหลือแค่ค่าใช้
จ่ายเล็ก ๆ จากการเขียนข้อมูลผ่าน pointer ที่อยู่บน heap (ไม่ได้อยู่ใน register/stack ที่ compiler จัดการให้
โดยตรง) — นี่คือเหตุผลเชิงปฏิบัติที่ Part 54 (และเอกสาร Rust ทั่วไป) แนะนำ **`Vec::with_capacity()` เสมอเมื่อ
รู้ขนาดล่วงหน้า** แทนการ push ทีละตัวโดยไม่จอง capacity หรือแย่กว่านั้นคือ `Box::new` แยกทุกตัว

#### 56.5.1 เจาะดูข้างในกล่องดำ: Allocator สมัยใหม่ทำงานอย่างไรกันแน่

สามข้อในหัวข้อก่อนอธิบาย "ทำไมแพง" ในระดับแนวคิด แต่ allocator ที่มาพร้อม Rust (`System`, ผูกกับ `malloc` ของ
libc บน Linux/macOS หรือของ Windows) และ allocator ทางเลือกอย่าง `jemalloc`/`mimalloc` (หัวข้อ 56.6) ไม่ได้
ทำงานแบบ "ค้นหา free-list เส้นเดียวยาว ๆ" ง่าย ๆ อย่างที่คนมักจินตนาการ — allocator สมัยใหม่ทุกตัวใช้เทคนิค
ร่วมกันสองอย่างที่ควรรู้จักไว้ เพราะเป็นพื้นฐานที่ทำให้เข้าใจว่าทำไมหัวข้อ 56.6 ถึงมี motivation จริงในการสลับ
allocator:

1. **Size classes** — allocator ไม่ได้ค้นหาก้อนที่ "พอดีเป๊ะ" กับขนาดที่ขอทุกครั้ง (ซึ่งจะช้ามากถ้าต้องค้นหา
   ในโครงสร้างข้อมูลขนาดใหญ่) แต่จะปัดขนาดที่ขอขึ้นไปเป็น "class ขนาดมาตรฐาน" ที่กำหนดไว้ล่วงหน้า (เช่น 8, 16,
   32, 64, 128, ... ไบต์) แล้วเก็บ free-list แยกกันสำหรับแต่ละ size class — การขอ allocation ขนาด 20 ไบต์
   (เหมือน `Small` ในหัวข้อ 56.5) จะถูกปัดขึ้นเป็น 32 ไบต์ (class ที่ใกล้ที่สุดที่ไม่เล็กกว่าที่ขอ) แล้วไปหยิบ
   จาก free-list ของ class 32 ไบต์โดยตรง — การค้นหาแบบนี้เร็วมาก (ตาราง lookup ตาม class แทนการค้นหาทั่วไป)
   แต่แลกกับการเสียพื้นที่ไปเล็กน้อยจากการปัดขึ้น (เรียกว่า **internal fragmentation**)
2. **Thread-local caching (thread-local arena)** — allocator สมัยใหม่แทบทุกตัวให้แต่ละ thread มี "คลัง" ของ
   ตัวเองแยกจาก thread อื่น (thread-local arena) เพื่อที่การ allocate/deallocate ส่วนใหญ่**ไม่ต้อง lock อะไร
   เลย**ระหว่าง thread (ถ้าต้อง lock ทุกครั้งที่ allocate ในโปรแกรมที่มีหลาย thread จะกลายเป็นคอขวดใหญ่ทันที
   ตามหลักการ contention ที่ Part 39/51 สอนไว้) — เฉพาะเมื่อ arena ของ thread นั้นหมดจริง ๆ จึงต้องไปขอเพิ่ม
   จาก arena กลางหรือจาก OS ตรง ๆ (คือ syscall ที่ข้อ 2 ของหัวข้อก่อนพูดถึง)

allocator แต่ละตัวต่างกันตรง**รายละเอียดการ implement สองเทคนิคนี้** — `jemalloc` และ `mimalloc` (หัวข้อ 56.6)
ออกแบบ size class และ thread-local arena ให้ tuned ดีกว่า `System` allocator ทั่วไปสำหรับ workload ที่ allocate/
deallocate ถี่มากจากหลาย thread พร้อมกัน (เช่น web server ที่รับ request นับพันต่อวินาที) — นี่คือเหตุผลที่
โปรเจกต์เหล่านี้มักได้ประโยชน์จริงจากการสลับ allocator แม้จะไม่ได้แก้โค้ด logic ของตัวเองแม้แต่บรรทัดเดียว

### 56.6 Custom Allocators: `#[global_allocator]` และ `GlobalAlloc` (ระดับ Awareness)

หัวข้อ 56.5 วัดต้นทุนของ allocator **มาตรฐาน**ที่ Rust ใช้ (ปกติคือ `System` allocator ที่ผูกกับ `malloc`/
`free` ของระบบปฏิบัติการ) แต่ Rust เปิดช่องให้ทำสิ่งที่ภาษาส่วนใหญ่ทำไม่ได้เลย: **เปลี่ยน allocator ของ
โปรแกรมทั้งโปรแกรม** ผ่าน attribute `#[global_allocator]` — หัวข้อนี้เป็นระดับ **awareness** (รู้ว่ามีอยู่และ
ใช้ทำอะไร ไม่ใช่การเรียนรู้เพื่อเขียน production allocator เอง ซึ่งเป็นงานเฉพาะทางมาก)

**ทำไมถึงอยากเปลี่ยน allocator ทั้งโปรแกรม?** สองสถานการณ์หลักที่พบจริง:

1. **Embedded/Kernel ที่ไม่มี heap มาตรฐานของ OS ให้ใช้** — ระบบฝังตัวหลายแบบไม่มี `malloc` ของ libc เลย (ไม่มี
   OS คอยจัดการหน่วยความจำให้) ต้องเขียน allocator ของตัวเองที่จัดการหน่วยความจำ RAM ก้อนที่กำหนดตายตัวไว้
   ล่วงหน้า (จะเจาะลึกกว่านี้ตอนเรียน embedded Rust ในบทหลัง ๆ ของหลักสูตร)
2. **Allocator เฉพาะทางที่เร็วกว่า `System` สำหรับ workload เฉพาะแบบ** — crate อย่าง `jemalloc`
   (`tikv-jemallocator`) หรือ `mimalloc` ออกแบบมาให้จัดการ allocation/deallocation จำนวนมากพร้อมกันข้ามหลาย
   thread ได้เร็วกว่า `System` allocator ทั่วไปในหลายสถานการณ์ (เช่น server ที่รับ request จำนวนมากพร้อมกัน)
   — โปรเจกต์ที่ต้องการประสิทธิภาพสูงสุดมักสลับไปใช้ crate เหล่านี้แทน `System` allocator เริ่มต้น

trait ที่ต้อง implement คือ `GlobalAlloc` ซึ่งมีแค่สอง method บังคับ: `alloc` (ขอหน่วยความจำใหม่) และ
`dealloc` (คืนหน่วยความจำ) — ทั้งคู่เป็น `unsafe fn` เพราะการจัดการ raw memory แบบนี้คือสิ่งที่ Part 42 สอนไว้
ว่า compiler ตรวจสอบให้ไม่ได้เลย ต้องรับผิดชอบเอง 100% ลองเขียน allocator ตัวอย่างที่**นับจำนวนและขนาดของทุก
allocation** (ห่อ `System` allocator เดิมไว้ข้างใน แค่แทรกการนับเข้าไปก่อน) — ตัวอย่างที่ใช้งานได้จริงและ
มีประโยชน์สำหรับ debug memory usage:

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::sync::atomic::{AtomicUsize, Ordering};

struct CountingAllocator;

static ALLOC_COUNT: AtomicUsize = AtomicUsize::new(0);
static ALLOC_BYTES: AtomicUsize = AtomicUsize::new(0);

// unsafe impl เพราะเรากำลังสัญญากับ compiler ว่า allocator นี้ทำตาม "สัญญา" ของ GlobalAlloc
// ครบถ้วน (คืน pointer ที่ align ถูกต้องตาม Layout, ไม่คืน pointer ซ้ำสองครั้งจนกว่าจะ dealloc
// ไปแล้ว ฯลฯ) -- สัญญาแบบเดียวกับ safety invariant ที่ Part 42 สอนไว้ทุกประการ
unsafe impl GlobalAlloc for CountingAllocator {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        ALLOC_COUNT.fetch_add(1, Ordering::Relaxed);
        ALLOC_BYTES.fetch_add(layout.size(), Ordering::Relaxed);
        // ส่งงานจริงต่อให้ System allocator ทำ -- เราแค่ "แทรกตัวนับ" เข้าไปก่อนหน้า ไม่ได้
        // เขียน allocator ตั้งแต่ต้นเอง (การเขียนตั้งแต่ต้นจริง ๆ ต้องจัดการ free-list เอง
        // ซึ่งเป็นงานเฉพาะทางเกินระดับ awareness ของหัวข้อนี้)
        System.alloc(layout)
    }

    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
        System.dealloc(ptr, layout)
    }
}

// #[global_allocator] บอก compiler ว่า "ให้ทุก Box::new, Vec::push, String::from ฯลฯ ในทั้ง
// โปรแกรมนี้ (รวม dependency crate ทุกตัวที่ใช้ heap ด้วย) เรียกผ่าน allocator ตัวนี้แทน System
// ตัวเดิมโดยตรง"
#[global_allocator]
static GLOBAL: CountingAllocator = CountingAllocator;

fn main() {
    let v: Vec<i32> = (0..1000).collect();
    let s = String::from("hello world");
    println!(
        "allocations so far = {}, bytes so far = {}",
        ALLOC_COUNT.load(Ordering::Relaxed),
        ALLOC_BYTES.load(Ordering::Relaxed)
    );
    println!("v.len()={} s={}", v.len(), s);
}
```

ผลลัพธ์จริงจากการรัน (คอมไพล์และรันจริงด้วย `rustc --edition 2021 -O`):

```
allocations so far = 4, bytes so far = 4559
v.len()=1000 s=hello world
```

ตัวเลข "4 allocations" นี้น่าสนใจ — มันไม่ใช่แค่ allocation ของ `v` กับ `s` (2 ครั้ง) แต่รวมถึง allocation
ภายในของ runtime ของ Rust เอง (เช่น buffer ภายในของกลไก `println!`/stdout ที่ต้องจองพื้นที่ก่อนพิมพ์) — นี่
คือประโยชน์จริงของ counting allocator แบบนี้: **มันเผยให้เห็น allocation ที่คุณอาจไม่รู้ตัวว่าเกิดขึ้น**
เพราะมันดักจับ**ทุก** allocation ในโปรแกรม ไม่ใช่แค่ที่คุณเขียนเอง — เทคนิคนี้ใช้ debug memory leak หรือหา
"allocation ที่ไม่จำเป็น" ในโปรแกรมจริงได้ (เชื่อมกับแนวคิด profiling จาก Part 55)

**ข้อจำกัดสำคัญที่ต้องรู้**: ในทั้ง binary crate (รวม dependency ทั้งหมด) **ประกาศ `#[global_allocator]` ได้
แค่ตัวเดียวเท่านั้น** ลองประกาศสองตัวดู:

```rust
use std::alloc::System;

#[global_allocator]
static A: System = System;

#[global_allocator]
static B: System = System;

fn main() {
    println!("hi");
}
```

Error จริงจาก compiler:

```
error: cannot define multiple global allocators
 --> src/main.rs:7:1
  |
4 | static A: System = System;
  | -------------------------- previous global allocator defined here
5 |
6 | #[global_allocator]
  | ------------------- in this attribute macro expansion
7 | static B: System = System;
  | ^^^^^^^^^^^^^^^^^^^^^^^^^^ cannot define a new global allocator
```

เหตุผลที่ต้องมีแค่หนึ่งตัวเท่านั้นชัดเจนในตัว: allocator คือส่วนที่**รับผิดชอบทุกการจอง/คืนหน่วยความจำในทั้ง
กระบวนการ (process)** — ถ้ามีสองตัวพร้อมกัน หน่วยความจำที่ตัวหนึ่งจองแล้วไปคืนกับอีกตัวจะกลายเป็น undefined
behavior ทันที (แต่ละ allocator มักมีโครงสร้างข้อมูลภายในของตัวเองที่ไม่รู้จักกัน) — Rust เลือกป้องกันปัญหานี้
ที่ compile time ไปเลยดีกว่าปล่อยให้เป็นบั๊กที่หายากตอน runtime

#### 56.6.1 ใช้ allocator สำเร็จรูปจริงในโปรเจกต์: ตัวอย่าง `mimalloc`

ในทางปฏิบัติ โปรเจกต์ส่วนใหญ่ที่ต้องการสลับ allocator **ไม่ได้เขียน `GlobalAlloc` implementation เองตั้งแต่ต้น**
(แบบ `CountingAllocator` ในหัวข้อ 56.6) แต่ดึง crate สำเร็จรูปที่มีคนเขียน allocator คุณภาพสูงไว้ให้แล้วมาใช้
ตรง ๆ — `mimalloc` (พัฒนาโดย Microsoft) เป็นตัวอย่างที่ได้รับความนิยมสูงตัวหนึ่ง ลองติดตั้งและใช้งานจริง:

```toml
# Cargo.toml
[dependencies]
mimalloc = { version = "0.1", default-features = false }
```

```rust
use mimalloc::MiMalloc;

// แค่บรรทัดนี้บรรทัดเดียว -- ทุก Box::new, Vec::push, String::from ในทั้งโปรแกรม (รวม
// dependency crate อื่น ๆ ที่ import เข้ามาด้วย) เปลี่ยนไปใช้ mimalloc แทน System ทันที
// ไม่ต้องแก้โค้ด logic ของตัวเองแม้แต่บรรทัดเดียว
#[global_allocator]
static GLOBAL: MiMalloc = MiMalloc;

fn main() {
    let v: Vec<i32> = (0..1000).collect();
    println!("sum = {}", v.iter().sum::<i32>());
}
```

โค้ดนี้ compile และรันผ่านจริง (ทดสอบด้วย `cargo build --release` ในการเขียนบทนี้) ได้ผลลัพธ์ `sum = 499500`
เหมือนกับที่ใช้ `System` allocator ทุกประการ (ผลลัพธ์ทาง logic **ต้องเหมือนกันเป๊ะเสมอ** ไม่ว่าจะใช้ allocator
ตัวไหน — allocator เปลี่ยนแค่ "ประสิทธิภาพของการจอง/คืนหน่วยความจำ" ไม่เคยเปลี่ยน "ความหมายทาง logic" ของ
โปรแกรมเลย นี่คือสัญญาที่ trait `GlobalAlloc` การันตีไว้) — สังเกตว่าการสลับ allocator ทั้งโปรแกรมใช้แค่**สอง
บรรทัด** (`use` และ `#[global_allocator] static GLOBAL: ...`) ไม่ต้องแก้ business logic ที่มีอยู่แล้วแม้แต่
บรรทัดเดียว นี่คือพลังของการที่ Rust ออกแบบให้ allocator เป็น**สิ่งที่เปลี่ยนได้จากภายนอก**แทนที่จะฝังไว้ตายตัว
ในตัวภาษาแบบภาษาส่วนใหญ่ — crate อย่าง `tikv-jemallocator` (ห่อ `jemalloc`) ก็ใช้รูปแบบเดียวกันนี้เป๊ะ ต่างกัน
แค่ชื่อ struct ที่ implement `GlobalAlloc` เท่านั้น

### 56.7 `#[no_std]`: เมื่อไม่มี Standard Library ให้ใช้เลย (ระดับ Awareness)

เนื้อหาทั้งหมดของหลักสูตรจนถึงบทนี้ (และของ Rust โดยทั่วไป) อยู่ในโลกที่มี **standard library** (`std`) ให้ใช้
เสมอ — `std` ให้ heap allocator (ที่หัวข้อ 56.6 พูดถึง), thread, file I/O, network, และอีกมากมายที่ผูกกับ
บริการของระบบปฏิบัติการ แต่มีบริบทหนึ่งที่ **ไม่มี OS ให้ผูกด้วยเลย**: microcontroller ฝังตัว (embedded), เขียน
kernel ของ OS เอง, หรือ bootloader ที่รันก่อน OS จะเริ่มทำงานด้วยซ้ำ — ในบริบทเหล่านี้ `std` ใช้ไม่ได้เลยเพราะ
มันคาดหวังว่ามี OS คอยให้บริการ (memory allocator ที่ผูกกับ syscall, thread ที่ผูกกับ OS scheduler ฯลฯ) ซึ่ง
ไม่มีอยู่จริง

Rust แก้ปัญหานี้ด้วย attribute `#![no_std]` ที่ใส่ไว้บนสุดของ crate — มันบอก compiler ว่า **"ห้ามผูกกับ `std`
เลย ให้ใช้ได้แค่ `core` (ส่วนของ standard library ที่ไม่ต้องพึ่ง OS อะไรเลย เช่น `Option`, `Result`,
`Iterator`, integer/float arithmetic) เท่านั้น"** — ถ้าต้องการ heap allocation (`Box`, `Vec`, `String`) โดยไม่
มี OS ก็ยังทำได้ผ่าน crate `alloc` (ส่วนย่อยของ `std` ที่แยกออกมาต่างหาก) แต่ต้องผูก custom allocator ของ
ตัวเองเข้าไปก่อน (ใช้ `#[global_allocator]` จากหัวข้อ 56.6 นี่แหละ — สองหัวข้อนี้เชื่อมกันโดยตรง เพราะใน
`no_std` world ไม่มี `System` allocator ที่ผูกกับ OS ให้ใช้แล้ว ต้องเขียน allocator เองเสมอถ้าต้องการ heap)

ตัวอย่าง library crate แบบ `no_std` ที่ใช้งานได้จริง (คอมไพล์ผ่านจริงด้วย `cargo check` ในบทนี้):

```rust
#![no_std]

// no_std lib: ไม่ผูกกับ std เลย ใช้ได้เฉพาะ core (และ alloc ถ้าเพิ่ม crate alloc เข้ามาเอง)
// สังเกตว่ายังใช้ Option/checked_mul ได้ปกติ เพราะทั้งสองอยู่ใน core ไม่ใช่ std

pub const fn double(x: i32) -> i32 {
    x * 2
}

pub fn checked_double(x: i32) -> Option<i32> {
    x.checked_mul(2)
}
```

`cargo check` ผ่านสนิทไม่มี warning ใด ๆ เกี่ยวกับ `no_std` เลย — เพราะฟังก์ชันทั้งสองใช้แค่ arithmetic พื้นฐาน
กับ `Option` ซึ่งอยู่ใน `core` อยู่แล้ว **หมายเหตุความซื่อตรง**: ตัวอย่างนี้คอมไพล์เป็น **library** (`rlib`)
เท่านั้น — การสร้างเป็น **executable binary** แบบ `no_std` เต็มรูปแบบต้องเพิ่มอีกสองสิ่งที่ตัวอย่างนี้ยังไม่มี:
`#[panic_handler]` (ฟังก์ชันที่บอกว่าเมื่อ panic เกิดขึ้นแล้วไม่มี `std` คอยจัดการ ให้ทำอะไร เช่น เข้า infinite
loop หรือ reset ตัว microcontroller) และ entry point ระดับต่ำที่ไม่ผ่าน `main()` ปกติของ OS (ผูกกับ
`#[no_main]` และ startup code เฉพาะของแต่ละ target) — สองเรื่องนี้ขึ้นกับ **target ที่ compile ไปด้วย** (เช่น
`thumbv7em-none-eabihf` สำหรับ ARM Cortex-M) ซึ่ง sandbox ที่ใช้เขียนบทนี้ไม่มี target แบบนั้นติดตั้งไว้ให้
ทดสอบจริง — รายละเอียดเต็มรูปแบบของการเขียนโปรแกรม `no_std` แบบ executable สมบูรณ์ (รวม `#[panic_handler]`,
linker script, และการ flash ไปยัง hardware จริง) จะเป็นเนื้อหาเต็มของบทที่สอน embedded Rust โดยเฉพาะในโมดูล
ถัดไปของหลักสูตร — บทนี้ให้แค่รู้จักคำว่า `#[no_std]` และเข้าใจว่ามันแก้ปัญหาอะไร ก็เพียงพอสำหรับระดับ
awareness ที่ตั้งใจไว้

#### 56.7.1 `no_std` + `alloc`: มี Heap ได้โดยไม่ต้องมี OS

จุดที่มักสร้างความสับสนคือ **"`no_std` แปลว่าห้ามใช้ heap เลยหรือไม่?"** — คำตอบคือ**ไม่จำเป็น** ต้องแยกสอง
เรื่องออกจากกันให้ชัด: `std` ผูกกับทั้ง "บริการของ OS" (thread, file, network) **และ** "heap allocation" พร้อม
กัน แต่ Rust แยก**เฉพาะส่วน heap allocation**ออกมาเป็น crate ต่างหากชื่อ `alloc` ที่**ไม่ต้องพึ่ง OS เลย** —
มันแค่ต้องการ**อะไรสักอย่างที่ implement `GlobalAlloc`** (หัวข้อ 56.6) เพื่อรับผิดชอบเรื่องหน่วยความจำ ซึ่งใน
โลก `no_std` มักเป็น allocator ที่เขียนเองแบบจัดการ RAM ก้อนคงที่ที่กำหนดไว้ล่วงหน้า (คล้ายกับ bump allocator
ในแบบฝึกหัดที่ 4) ไม่ใช่ `System` allocator ที่ผูกกับ syscall ของ OS:

```rust
#![no_std]
extern crate alloc; // เปิดใช้ Box/Vec/String แม้ไม่มี std -- ต้องมี #[global_allocator]
                     // ประกาศไว้ที่ไหนสักแห่งในโปรแกรมสุดท้ายเสมอ (หัวข้อ 56.6) ไม่งั้น link ไม่ผ่าน

use alloc::vec::Vec; // Vec มาจาก crate alloc ไม่ใช่ std ในบริบทนี้
use alloc::vec;

pub fn make_doubled(input: &[i32]) -> Vec<i32> {
    // ใช้ Vec ได้ปกติทุกประการ แม้จะไม่มี std เลยก็ตาม -- เพราะ Vec มาจาก alloc ไม่ใช่ std
    input.iter().map(|x| x * 2).collect()
}

pub fn make_zeroed(n: usize) -> Vec<i32> {
    vec![0; n]
}
```

โค้ดนี้แสดงให้เห็นว่า `Vec<T>` (Part 13) ไม่ใช่ของที่ผูกติดกับ `std` โดยเนื้อแท้ — มันอยู่ใน `alloc` ซึ่งต้องการ
แค่ heap allocator ตัวไหนก็ได้ที่ implement `GlobalAlloc` ให้ครบถ้วน ไม่ว่าจะเป็น `System` (ใน binary ปกติที่มี
std) หรือ allocator ที่เขียนเองสำหรับ microcontroller (ใน `no_std` binary) — สิ่งที่**ผูกกับ `std` จริง ๆ**คือ
`String` ในรูปแบบเต็ม (ที่จริง `String` ก็อยู่ใน `alloc` เช่นกัน ใช้ได้ใน `no_std` ได้เหมือน `Vec`), thread,
file I/O, network, และ `std::time::Instant` เป็นต้น — สิ่งเหล่านี้ต้องพึ่งพา OS จริง ๆ จึงอยู่ใน `std` เท่านั้น
ไม่มีทางเลี่ยงได้ในบริบท `no_std` เพราะไม่มี OS ให้เรียกใช้เลย

### 56.8 Cache-Friendliness และ Data Layout: AoS เทียบกับ SoA (วัดจริงด้วย `criterion`)

Part 42 สอนเรื่อง alignment/padding และเทคนิคจัดเรียง field ใหม่เพื่อลดขนาด `struct` ไปแล้วอย่างละเอียด (ตัว
อย่างในบทนั้นลดขนาดได้จาก 32 ไบต์เหลือ 16 ไบต์) แต่ยังไม่ได้ตอบคำถามที่ใหญ่กว่านั้นอีกขั้น: **เมื่อมี `struct`
จำนวนมาก ๆ เก็บอยู่ใน `Vec` (หลักหมื่น-หลักล้านตัว) แล้วโปรแกรมต้องเข้าถึงแค่บาง field ของทุกตัว — การจัดเรียง
ข้อมูลทั้งชุด (ไม่ใช่แค่ struct เดียว) มีผลต่อความเร็วแค่ไหน?** นี่คือคำถามเรื่อง **Array-of-Structs (AoS)
เทียบกับ Struct-of-Arrays (SoA)**

**AoS** คือรูปแบบปกติที่เราเขียนกันมาตลอดหลักสูตร: `Vec<Particle>` ที่แต่ละ `Particle` เก็บทุก field ของตัวมัน
ไว้ติดกัน (เหมือนที่ `Vec<T>` ทำงานมาตั้งแต่ Part 13) **SoA** คือการ**พลิกโครงสร้าง**: แยกแต่ละ field ออกเป็น
`Vec` ของตัวเอง แทนที่จะมี `Vec` ของ struct ตัวเดียว — เมื่อโปรแกรมต้องอ่านแค่ field เดียวจาก particle ทุกตัว
(เช่น สรุปค่า `x` รวมของทุกอนุภาค) SoA ทำให้ CPU **โหลดแค่ข้อมูลที่ต้องใช้จริงเข้า cache** โดยไม่ต้องแบก field
อื่นที่ไม่เกี่ยวมาด้วยเลย ต่างจาก AoS ที่ field อื่นทั้งหมดของ struct ก็ถูกโหลดเข้า cache line เดียวกันไปด้วย
โดยไม่มีประโยชน์:

```rust
// benches/layout_bench.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};

const N: usize = 200_000;

// AoS: struct 32 ไบต์ (8 field x 4 ไบต์) -- ตัวอย่างจริงของ particle simulation ที่มัก
// มี field จำนวนมาก แต่ operation หนึ่ง ๆ ใช้แค่บางฟิลด์
#[derive(Clone, Copy)]
struct ParticleAoS {
    x: f32, y: f32, z: f32,
    vx: f32, vy: f32, vz: f32,
    mass: f32, charge: f32,
}

fn make_aos(n: usize) -> Vec<ParticleAoS> {
    (0..n)
        .map(|i| ParticleAoS {
            x: i as f32, y: 0.0, z: 0.0,
            vx: 1.0, vy: 0.0, vz: 0.0,
            mass: 1.0, charge: 0.0,
        })
        .collect()
}

// SoA: แยกแต่ละ field เป็น Vec ของตัวเอง -- field ที่ไม่ได้ใช้ในงานนี้ (y, z, vx, vy, vz,
// mass, charge) ไม่ต้องถูกโหลดเข้า cache เลยเมื่อเราต้องการแค่ x
struct ParticlesSoA {
    x: Vec<f32>, y: Vec<f32>, z: Vec<f32>,
    vx: Vec<f32>, vy: Vec<f32>, vz: Vec<f32>,
    mass: Vec<f32>, charge: Vec<f32>,
}

fn make_soa(n: usize) -> ParticlesSoA {
    ParticlesSoA {
        x: (0..n).map(|i| i as f32).collect(),
        y: vec![0.0; n], z: vec![0.0; n],
        vx: vec![1.0; n], vy: vec![0.0; n], vz: vec![0.0; n],
        mass: vec![1.0; n], charge: vec![0.0; n],
    }
}

// งานจริงที่วัด: บวก x ของทุก particle เข้าด้วยกัน (ใช้แค่ 1 field จากแปดฟิลด์)
fn sum_x_aos(particles: &[ParticleAoS]) -> f32 {
    particles.iter().map(|p| p.x).sum()
}

fn sum_x_soa(soa: &ParticlesSoA) -> f32 {
    soa.x.iter().sum()
}

fn bench_layout(c: &mut Criterion) {
    let aos = make_aos(N);
    let soa = make_soa(N);
    let mut group = c.benchmark_group("aos_vs_soa_sum_single_field_200k");
    group.bench_function("array_of_structs", |b| b.iter(|| sum_x_aos(black_box(&aos))));
    group.bench_function("struct_of_arrays", |b| b.iter(|| sum_x_soa(black_box(&soa))));
    group.finish();
}

criterion_group!(benches, bench_layout);
criterion_main!(benches);
```

(ยืนยันขนาดจริงก่อนวัดผล ด้วยเทคนิคจาก Part 42: `size_of::<ParticleAoS>() == 32` ไบต์ — 8 field ละ 4 ไบต์
ไม่มี padding เพราะทุก field เป็น `f32` alignment เท่ากันหมด) ผลลัพธ์จริงจาก `cargo bench --bench
layout_bench`:

```
aos_vs_soa_sum_single_field_200k/array_of_structs
                        time:   [163.51 µs 167.36 µs 172.23 µs]
aos_vs_soa_sum_single_field_200k/struct_of_arrays
                        time:   [135.02 µs 136.51 µs 138.14 µs]
```

**SoA เร็วกว่า AoS ประมาณ 18%** (167.36 µs เทียบกับ 136.51 µs) สำหรับงาน "สรุปค่า field เดียวจากทุก element"
— เหตุผลเชิงกลไก: การอ่าน `Vec<ParticleAoS>` แม้จะใช้แค่ `p.x` ก็ต้องโหลดทั้ง 32 ไบต์ของทุก `ParticleAoS`
เข้า cache line (cache line มาตรฐานบน CPU ทั่วไปคือ 64 ไบต์ — พอดีสำหรับ 2 particle ต่อ cache line หนึ่ง
เส้น) ทำให้**ใช้ bandwidth ของ cache ไปกับข้อมูลที่ไม่ได้ใช้ถึง 7/8 ส่วน** (`y`, `z`, `vx`, `vy`, `vz`, `mass`,
`charge` ทั้งหมดถูกโหลดเข้ามาโดยไม่ได้ใช้เลย) ในขณะที่ `Vec<f32>` ของ SoA เก็บค่า `x` ต่อเนื่องกันล้วน ๆ
ทุกไบต์ที่โหลดเข้า cache line ถูกใช้จริง 100% — **นี่คือความเชื่อมโยงตรงกับ Part 42**: มันคือหลักการ "field
reordering เพื่อลด padding" แบบเดียวกัน แต่ยกระดับจาก "ภายใน struct เดียว" ขึ้นไปเป็น "ภายในชุดข้อมูลทั้งหมด"

**ข้อควรระวังที่สำคัญมาก** (ที่หัวข้อ 56.11 จะพิสูจน์ให้เห็นเป็นรูปธรรมยิ่งขึ้น): ผลลัพธ์ 18% นี้เป็นจริง
**เฉพาะกรณีที่ operation ต้องการแค่ 1 field จากหลาย ๆ field เท่านั้น** — ถ้า operation หนึ่งต้องใช้**หลาย
field พร้อมกัน**ต่อ element (เช่น อัปเดตทั้ง `x` และ `vx` ในการคำนวณเดียวกัน) SoA อาจไม่ชนะ AoS เลย หรือถึงกับ
**แพ้** เพราะการเข้าถึงหลาย `Vec` แยกกันพร้อมกันทำให้ CPU ต้อง track memory stream หลายเส้นพร้อมกัน (แต่ละ
`Vec` คือ stream ของตัวเอง) ในขณะที่ AoS เก็บ field ที่เกี่ยวข้องกันไว้ติดกันในก้อนเดียว — **ไม่มีคำตอบเดียวที่
ถูกเสมอสำหรับทุกกรณี** ต้องดูรูปแบบการเข้าถึงข้อมูลจริงของ workload นั้นก่อนตัดสินใจเสมอ (ตามหลักการ "อย่าเดา
วัด" จาก Part 54)

### 56.9 Const Generics และการคำนวณตอน Compile Time: ทวนและขยายจาก Part 18

หัวข้อ 56.4 พิสูจน์ว่า `const fn` ที่เรียกด้วยค่าคงที่หายไปจาก runtime โดยสิ้นเชิง — หัวข้อนี้ทวนแนวคิด **const
generics** ที่ Part 18 เกริ่นไว้สั้น ๆ (`Grid<const N: usize>`) แล้วขยายให้เห็นว่าสองเรื่องนี้ (`const fn` +
const generics) ผสมกันได้อย่างทรงพลัง เพราะ**ขนาด** (ที่เป็น const generic parameter) กับ**การคำนวณ** (ที่เป็น
`const fn`) ทั้งคู่ถูกตรึงไว้ที่ compile time พร้อมกัน — ลองสร้าง `Matrix<const ROWS: usize, const COLS:
usize>` ที่ตรวจสอบมิติผ่าน type system และคำนวณจำนวน element ตอน compile time:

```rust
struct Matrix<const ROWS: usize, const COLS: usize> {
    data: [[f64; COLS]; ROWS],
}

impl<const ROWS: usize, const COLS: usize> Matrix<ROWS, COLS> {
    const fn zero() -> Self {
        Self {
            data: [[0.0; COLS]; ROWS],
        }
    }

    // ฟังก์ชันนี้เป็น const fn -- ถ้าเรียกในบริบท const (เช่นด้านล่าง) จะถูกคำนวณตอน compile
    // time ทั้งหมด ไม่มีการ "คูณเลข" เหลือให้ CPU ทำตอนรันจริงแม้แต่ครั้งเดียว
    const fn element_count() -> usize {
        ROWS * COLS
    }
}

// เรียกใน const context ตรง ๆ -- การคูณ ROWS * COLS เกิดขึ้นที่ compile time เท่านั้น
const IDENTITY_3X3_ELEMENT_COUNT: usize = Matrix::<3, 3>::element_count();

fn main() {
    let m: Matrix<3, 4> = Matrix::zero();
    println!("element_count (compile-time) = {}", IDENTITY_3X3_ELEMENT_COUNT);
    println!("Matrix::<3,4>::element_count() = {}", Matrix::<3, 4>::element_count());
    println!("size_of::<Matrix<3,4>>() = {} bytes", std::mem::size_of::<Matrix<3, 4>>());
    println!("m.data[0][0] = {}", m.data[0][0]);
}
```

ผลลัพธ์จริง:

```
element_count (compile-time) = 9
Matrix::<3,4>::element_count() = 12
size_of::<Matrix<3,4>>() = 96 bytes
m.data[0][0] = 0
```

สังเกตว่า `96 = 3 × 4 × 8` (3 แถว, 4 คอลัมน์, `f64` แถวละ 8 ไบต์) ตรงเป๊ะตามที่คำนวณด้วยมือได้ — นี่คือสิ่งที่
Part 18 พิสูจน์ไว้แล้วว่า `Point<i32>` กับ `Point<f64>` เป็น**ชนิดข้อมูลคนละตัวกันจริง**ในสายตา compiler
(ไม่ใช่แค่ "แม่แบบเดียวกัน") — `Matrix<3, 4>` และ `Matrix<3, 3>` ก็เป็นชนิดข้อมูลคนละตัวกันด้วยหลักการเดียวกัน
เป๊ะ เพียงแต่คราวนี้สิ่งที่ต่างกันคือ**ขนาด** (`ROWS`/`COLS`) ไม่ใช่**ชนิดข้อมูล** (`T`) — และเพราะขนาดถูกตรึง
ไว้ที่ compile time ทั้ง `[[f64; COLS]; ROWS]` จึงเป็น array ที่อยู่บน**stack ล้วน ๆ ไม่มี heap allocation
เลย** (ต่างจาก `Vec<Vec<f64>>` ที่ต้องมี heap allocation ซ้อนกันสองชั้นถ้าเขียนแบบ dynamic-size) — นี่คือ
ประโยชน์เชิงประสิทธิภาพที่จับต้องได้จริงของ const generics: **ขนาดที่รู้ตอน compile time แปลว่าไม่ต้องมี
allocator เข้ามาเกี่ยวข้องเลย** เชื่อมตรงกับหัวข้อ 56.5 ที่พิสูจน์แล้วว่า stack ถูกกว่า heap มาก

#### 56.9.1 Const Generics ก็เป็น Type ที่ต่างกันจริง ไม่ใช่แค่ "ตัวเลขที่ต่างกัน"

หัวข้อ 18.4 (ของ Part 18) พิสูจน์ไว้แล้วว่า `Point<i32>` และ `Point<f64>` เป็นชนิดข้อมูลคนละตัวกันจริง — หลัก
การเดียวกันนี้ใช้ได้กับ const generic parameter ด้วย: `Matrix<3, 4>` และ `Matrix<2, 3>` เป็นชนิดข้อมูลคนละ
ตัวกัน แม้ชื่อ struct จะเหมือนกันก็ตาม ลองผสมกันดูว่าเกิดอะไรขึ้น:

```rust
fn main() {
    let m: Matrix<3, 4> = Matrix::<2, 3>::zero();
    println!("{}", m.data[0][0]);
}
```

Error จริง:

```
error[E0308]: mismatched types
  --> src/main.rs:2:27
   |
 2 |     let m: Matrix<3, 4> = Matrix::<2, 3>::zero();
   |            ------------   ^^^^^^^^^^^^^^^^^^^^^^ expected `3`, found `2`
   |            |
   |            expected due to this
   |
   = note: expected struct `Matrix<3, 4>`
              found struct `Matrix<2, 3>`
```

**Error นี้เกิดตอน compile time เท่านั้น** — ไม่มีทางที่โปรแกรมจะ compile ผ่านแล้วพัง (หรือคำนวณผิดแบบเงียบ ๆ)
ตอนรันจากการผสมขนาด matrix ผิดแบบนี้เลย เทียบกับภาษาที่ตรวจสอบขนาด array/matrix แค่ตอน runtime (เช่น Python
ที่ NumPy ต้องเช็ค `.shape` เอาเองแล้ว raise exception ตอนรันถ้าขนาดไม่ตรงกัน, หรือ Java ที่ array ขนาดต่างกัน
ก็เป็นแค่ `int[][]` เหมือนกันในสายตา type system ไม่มีการแยกแยะขนาดเป็นส่วนหนึ่งของ type เลย) — Rust ยกระดับ
การตรวจสอบนี้ขึ้นไปเป็น**ส่วนหนึ่งของ type system ที่ compiler ตรวจให้ฟรี** ไม่ต้องเขียน runtime check เองเลย
แม้แต่บรรทัดเดียว นี่คือประโยชน์เชิง**ความปลอดภัย**ของ const generics ที่มาควบคู่กับประโยชน์เชิง**ประสิทธิภาพ**
(ไม่มี heap allocation, คำนวณ zero-cost) ที่พูดถึงไปแล้วก่อนหน้า — สองเรื่องนี้ไม่ได้แยกจากกัน แต่เป็นผลพวงจาก
หลักการเดียวกัน: **ยิ่งข้อมูลรู้ตอน compile time มากเท่าไหร่ ยิ่งให้ compiler ช่วยตรวจสอบและ optimize ได้มาก
เท่านั้น** (ภาษาที่มี generic แบบ C++ template ก็ตรวจสอบขนาดแบบนี้ได้เช่นกันผ่าน template parameter ที่เป็น
`size_t`, non-type template parameter — แนวคิดคล้ายกันมาก แต่ syntax และ error message ของ C++ template
มักซับซ้อนกว่ามากในทางปฏิบัติ ซึ่งเป็นเหตุผลหนึ่งที่ const generics ของ Rust ถูกออกแบบให้เรียบง่ายกว่าตั้งแต่
แรก)

### 56.10 Cost Model Checklist: ตารางสรุปต้นทุนที่ต้องพกติดตัวตลอดไป

ทุกหัวข้อก่อนหน้าพิสูจน์ประเด็นแยกกันไปทีละเรื่อง — หัวข้อนี้รวบรวมทุกอย่างที่หลักสูตรสอนมาตั้งแต่ Part 6
จนถึง Part 51 (และที่พิสูจน์เพิ่มในบทนี้) ให้เป็น**เช็คลิสต์เดียว**ที่นักพัฒนา Rust ทุกคนควรมีอยู่ในหัวตลอด
เวลาเวลาเขียนโค้ด — จัดกลุ่มเป็นสามระดับ: **ฟรีเสมอ**, **ถูกแต่ไม่ฟรี**, และ **แพงจริง**

| ระดับต้นทุน | รายการ | เหตุผลเชิงกลไก | สอนไว้ที่ Part ไหน |
|---|---|---|---|
| **ฟรีเสมอ (Always Free)** | Move (`let y = x;` ที่ย้าย ownership) | แค่ copy ตัวชี้/ตัวเลขบน stack ไม่มีการคัดลอกข้อมูลบน heap เลย compiler รู้ตอน compile time ว่า `x` ใช้ไม่ได้อีก | Part 6 |
| | Generic function ที่ monomorphize แล้ว | สร้าง machine code เฉพาะสำหรับแต่ละ concrete type — **พิสูจน์แล้วว่าเหมือนโค้ดเขียนมือ 100%** ด้วย ICF (56.2) | Part 18, บทนี้ 56.2 |
| | Iterator adaptor chain (`.map().filter().sum()`) | LLVM inline+fuse ทุก adaptor เป็น loop เดียว — **พิสูจน์แล้วว่าใช้เทคนิค optimization ระดับเดียวกับ manual loop** (56.3) | Part 25-26, บทนี้ 56.3 |
| | `const fn` ที่เรียกด้วย const context ล้วน | คำนวณเสร็จโดย `rustc` ตอน compile — **หายไปจาก runtime code ทั้งหมด** (56.4) | Part 18 (const generics), บทนี้ 56.4 |
| | Borrow (`&T`/`&mut T`) | เป็นแค่ address บน stack ตรวจสอบความถูกต้องทั้งหมดที่ compile time (borrow checker) ไม่มี runtime check เหลือ | Part 7 |
| **ถูกแต่ไม่ฟรี (Cheap, Not Free)** | `Rc::clone()` | เพิ่มตัวเลข `strong_count` หนึ่งครั้ง (ไม่ใช่ atomic เพราะ `Rc` ใช้แค่ thread เดียว) + copy ตัวชี้ — ถูกกว่า deep clone มากแต่ไม่ใช่ศูนย์ | Part 28 |
| | `Arc::clone()` | เหมือน `Rc::clone()` แต่เพิ่มตัวเลขแบบ **atomic** (Part 51) เพราะต้องปลอดภัยข้าม thread — แพงกว่า `Rc::clone()` เล็กน้อยจากค่าใช้จ่ายของ atomic instruction แต่ยังถูกกว่า deep clone มหาศาล | Part 28, Part 39, Part 51 |
| | `Vec::push()` ที่ capacity ยังพอ | O(1) จริง แต่ยังมีการเขียนข้อมูลลง heap memory (ไม่ใช่ register/stack) — เร็วกว่า reallocation มาก แต่ไม่ใช่ "ฟรี" เหมือน stack local | Part 13, บทนี้ 56.5 |
| | Non-contended atomic operation (`fetch_add` ที่ไม่มี thread อื่นแข่ง) | เร็วกว่า `Mutex` ทั่วไปในกรณีไม่แข่งกัน แต่ยังช้ากว่าคำสั่ง CPU ธรรมดา เพราะต้องรับประกัน memory ordering ข้าม core | Part 51 |
| | AoS ที่ operation ใช้ 1 field จากหลายฟิลด์ | ทำงานถูกต้องเสมอ แต่เสีย cache bandwidth ไปกับ field ที่ไม่ได้ใช้ — **วัดจริงแล้วว่าช้ากว่า SoA ~18%** ในกรณีนี้ (56.8) | Part 42, บทนี้ 56.8 |
| **แพงจริง (Genuinely Expensive)** | Heap allocation ต่อ element (`Box::new` ในทุกรอบ loop) | เรียก allocator เต็มรูปแบบ (bookkeeping + อาจมี syscall) — **วัดจริงแล้วว่าแพงกว่า stack ~11.8×** สำหรับข้อมูลขนาดเล็ก (56.5) | Part 6, Part 27, บทนี้ 56.5 |
| | Deep clone ของ `String`/`Vec<T>` (`.clone()`) | จอง heap buffer ใหม่ทั้งก้อน + คัดลอกข้อมูลทุกไบต์ — ต่างจาก `Copy`/`Rc::clone()` โดยสิ้นเชิง | Part 6, Part 13-14 |
| | Dynamic dispatch ผ่าน `dyn Trait` ที่ทำใน hot loop | เรียกผ่าน vtable (indirect call) — **compiler ไม่สามารถ inline ข้ามขอบ `dyn` ได้** ทำให้เสีย optimization ต่อเนื่อง (loop unrolling, SIMD auto-vectorization) ที่ static dispatch ได้รับ แม้ต้นทุนของ indirect call เองจะเล็ก | Part 21 |
| | Atomic operation ภายใต้ contention จริง (หลาย thread แข่งกันเขียนพร้อมกัน) | เกิด cache line invalidation ข้าม core ซ้ำ ๆ (false sharing ถ้า field อยู่ cache line เดียวกัน) — ช้ากว่า non-contended case มาก และในบางกรณีช้ากว่า `Mutex` เสียอีก | Part 39, Part 51 |
| | `Mutex`/`RwLock` lock ที่มี contention สูง | Thread ที่แย่งกันต้อง block และปลุกใหม่ (context switch ผ่าน OS scheduler) ซึ่งแพงกว่าการหมุนรอ (spin) ของ non-blocking algorithm มากในกรณี contention รุนแรง | Part 39 |
| | Heap allocation แบบสุ่มกระจาย (fragmented) หลาย ๆ ก้อนเล็ก ๆ ที่ต้องเดิน pointer ตาม (pointer chasing) | ไม่ใช่แค่ต้นทุนของการจอง/คืน แต่รวมต้นทุนของ cache miss ตอน**อ่าน**ข้อมูลด้วย เพราะแต่ละก้อนอยู่คนละที่ในหน่วยความจำ — **พิสูจน์เป็นรูปธรรมที่สุดในหัวข้อ 56.11** | Part 27, บทนี้ 56.11 |

**หลักการอ่านตารางนี้ให้ถูกต้อง**: แถวในกลุ่ม "ฟรีเสมอ" คือสิ่งที่**พิสูจน์แล้วด้วยหลักฐานเชิงประจักษ์**ว่าไม่มี
ต้นทุนส่วนเกิน (แถวที่มาจาก 56.2-56.4 ของบทนี้) หรือมาจากการันตีของ type system/borrow checker ที่ compiler
บังคับ (Part 6-7) แถวในกลุ่ม "ถูกแต่ไม่ฟรี" คือสิ่งที่**ควรใช้อย่างอิสระในโค้ดทั่วไป**เพราะต้นทุนต่ำมากเทียบกับ
ประโยชน์ที่ได้ (ไม่ต้องกังวลจนถึงขั้นเลี่ยงมันในโค้ดปกติ) แต่ก็**ไม่ควรใส่ไว้ใน hot loop ที่รันหลักล้านครั้งโดย
ไม่คิด**เพราะต้นทุนสะสมได้ ส่วนกลุ่ม "แพงจริง" คือสิ่งที่ควร**คิดก่อนใช้ใน hot path** และเป็นจุดแรกที่ควรมองหา
เมื่อ profiler (Part 55) ชี้ว่าโปรแกรมช้าตรงไหน — แต่ไม่ได้แปลว่า "ห้ามใช้เด็ดขาด" เพราะบางครั้งความสะดวกและ
ความถูกต้องของโค้ดสำคัญกว่าประสิทธิภาพสูงสุด (หลักการ "อย่า optimize ก่อนวัด" จาก Part 54 ยังใช้ได้เสมอ)

#### ตัวอย่างสั้น ๆ ของแถว "แพงจริง": Static Dispatch เทียบกับ Dynamic Dispatch

แถวหนึ่งในตารางข้างบนพูดถึง `dyn Trait` ที่เสีย "การ inline ต่อเนื่อง" ไป — Part 21 เจาะลึกเรื่อง vtable และ
object safety ไว้เต็มรูปแบบแล้ว บทนี้ขอแค่ยืนยันด้วยโค้ดสั้น ๆ ว่าความหมายของ "เสียการ inline" คืออะไรจริง ๆ:

```rust
trait Shape {
    fn area(&self) -> f64;
}

struct Circle {
    radius: f64,
}

impl Shape for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.radius * self.radius
    }
}

// static dispatch: compiler รู้ concrete type แน่นอนตอน compile time (monomorphize เหมือน
// หัวข้อ 56.2) -> inline area() เข้าไปในลูปได้เต็มที่ ไม่มี call เหลือเลย
fn total_area_static<T: Shape>(shapes: &[T]) -> f64 {
    shapes.iter().map(|s| s.area()).sum()
}

// dynamic dispatch: เรียกผ่าน vtable -- compiler ไม่รู้ concrete type ตอน compile time
// (Box<dyn Shape> อาจเป็น Circle, Square, Triangle ฯลฯ ก็ได้ ตัดสินใจตอน runtime) จึง
// inline area() เข้าไปในนี้ไม่ได้เลย ต้องมี indirect call ผ่าน vtable ทุก element
fn total_area_dyn(shapes: &[Box<dyn Shape>]) -> f64 {
    shapes.iter().map(|s| s.area()).sum()
}

fn main() {
    let circles = vec![Circle { radius: 1.0 }, Circle { radius: 2.0 }];
    println!("static dispatch: {:.4}", total_area_static(&circles));

    let boxed: Vec<Box<dyn Shape>> = vec![
        Box::new(Circle { radius: 1.0 }),
        Box::new(Circle { radius: 2.0 }),
    ];
    println!("dynamic dispatch: {:.4}", total_area_dyn(&boxed));
}
```

ผลลัพธ์จริง (ทั้งสองค่าต้องเท่ากันเสมอ — dispatch แบบไหนไม่เคยเปลี่ยนผลลัพธ์ทาง logic ตามหลักการเดียวกับ
allocator ในหัวข้อ 56.6.1):

```
static dispatch: 15.7080
dynamic dispatch: 15.7080
```

ต้นทุนที่ต่างกันไม่ได้อยู่ที่**ผลลัพธ์** (เหมือนกันเป๊ะ) แต่อยู่ที่**สิ่งที่ compiler ทำได้กับ loop ข้างใน**:
`total_area_static` ถูก compiler มองเห็นตลอดทั้ง loop ว่าเรียก `Circle::area()` ตัวเดียวกันทุกรอบ (เพราะ
monomorphize แล้วรู้ concrete type แน่นอน) จึง inline สูตรคำนวณเข้าไปในลูปได้ตรง ๆ (เปิดโอกาสให้ optimize
ต่อได้อีก เช่น auto-vectorization แบบเดียวกับที่เห็นใน 56.3) — แต่ `total_area_dyn` รับ `&[Box<dyn Shape>]`
ที่แต่ละ element**อาจเป็น concrete type คนละแบบกัน**ได้ (แม้ในตัวอย่างนี้จะมีแค่ `Circle` ก็ตาม แต่ signature
ของฟังก์ชันไม่ได้การันตีอย่างนั้น) — compiler จึงต้อง**เก็บความเป็นไปได้ทั่วไปไว้**และแปลเป็น indirect call
ผ่าน vtable ทุกครั้งจริง ๆ ไม่มีทาง inline ล่วงหน้าได้เลยไม่ว่าจะ optimize level ไหน (เว้นแต่ในบางกรณีที่
compiler ใช้เทคนิค **devirtualization** พิสูจน์ได้ว่า `Box<dyn Shape>` ทุกตัวในบริบทนั้นเป็น `Circle` เท่านั้น
จริง ๆ ซึ่งเกินระดับที่ optimizer ทั่วไปทำได้เสมอ) — นี่คือความหมายที่แท้จริงของ "เสียการ inline ต่อเนื่อง" ที่
ตารางในหัวข้อ 56.10 พูดถึง

#### 56.10.1 ยืนยันแถว "ถูกแต่ไม่ฟรี" ด้วยตัวเลขจริง: `String::clone()` เทียบกับ `Rc::clone()` เทียบกับ `Arc::clone()`

ตารางข้างบนจัด `Rc::clone()` และ `Arc::clone()` ไว้ในกลุ่ม "ถูกแต่ไม่ฟรี" ทั้งคู่ แต่ยังไม่ได้วัดตัวเลขจริงว่า
ทั้งสองต่างจาก deep clone และต่างจากกันเองแค่ไหน — มาปิดช่องว่างนี้ด้วย `criterion` อีกครั้ง เทียบสามแบบใน
สถานการณ์เดียวกัน: `String` ยาว 10,000 ตัวอักษร

```rust
use criterion::{black_box, criterion_group, criterion_main, Criterion};
use std::rc::Rc;
use std::sync::Arc;

fn bench_clone_costs(c: &mut Criterion) {
    let big_string = "x".repeat(10_000);
    let rc_string = Rc::new(big_string.clone());
    let arc_string = Arc::new(big_string.clone());

    let mut group = c.benchmark_group("clone_costs_10k_chars");

    group.bench_function("string_deep_clone", |b| {
        b.iter(|| black_box(black_box(&big_string).clone()))
    });
    group.bench_function("rc_clone", |b| {
        b.iter(|| black_box(Rc::clone(black_box(&rc_string))))
    });
    group.bench_function("arc_clone", |b| {
        b.iter(|| black_box(Arc::clone(black_box(&arc_string))))
    });

    group.finish();
}

criterion_group!(benches, bench_clone_costs);
criterion_main!(benches);
```

ผลลัพธ์จริง:

```
clone_costs_10k_chars/string_deep_clone
                        time:   [82.711 ns 84.078 ns 85.683 ns]
clone_costs_10k_chars/rc_clone
                        time:   [1.3373 ns 1.3578 ns 1.3829 ns]
clone_costs_10k_chars/arc_clone
                        time:   [18.026 ns 18.240 ns 18.468 ns]
```

| วิธี | เวลาเฉลี่ย | เทียบกับ `rc_clone` |
|---|---|---|
| `rc_clone` | ~1.36 ns | 1× (baseline) |
| `arc_clone` | ~18.24 ns | ~13.4× ช้ากว่า |
| `string_deep_clone` | ~84.08 ns | ~61.8× ช้ากว่า |

สามตัวเลขนี้ยืนยันโครงสร้างของตารางเช็คลิสต์ได้อย่างชัดเจนมาก: **deep clone แพงที่สุดในสามแบบอย่างเห็นได้ชัด**
(~62× ของ `Rc::clone()`) เพราะต้องจอง heap buffer ใหม่ทั้งก้อนขนาด 10,000 ไบต์แล้วคัดลอกทุกไบต์จริง ตรงตาม
หลักการ Part 6/14 — ส่วน **`Rc::clone()` ถูกที่สุด**เพราะเป็นแค่การเพิ่มเลขจำนวนเต็มธรรมดา (`strong_count +=
1`) ซึ่งเป็นคำสั่ง CPU แบบไม่ atomic (เพราะ `Rc` สัญญาไว้แล้วว่าใช้ในเธรดเดียวเท่านั้น — Part 28/40 อธิบายไว้
ว่านี่คือเหตุผลที่ `Rc<T>` ไม่ implement `Send`)

**ส่วนที่น่าสนใจที่สุดคือช่องว่างระหว่าง `Rc::clone()` กับ `Arc::clone()`** — ต่างกันถึง **~13.4 เท่า** ทั้งที่
ทั้งคู่ "แค่เพิ่มเลขจำนวนเต็มหนึ่งค่า" เหมือนกันในทางแนวคิด ความต่างนี้มาจากธรรมชาติของ**atomic instruction**
เอง: บน CPU สถาปัตยกรรม x86-64 การเพิ่มค่าแบบ atomic ต้องใช้คำสั่งที่มี prefix `lock` (เช่น `lock xadd`) ซึ่ง
บังคับให้ CPU **หยุดรอให้ cache line ที่เกี่ยวข้องซิงค์กับ cache coherency protocol ของทั้งระบบให้เรียบร้อย
ก่อน**เสมอ — แม้จะไม่มี thread อื่นแข่งอยู่เลย (non-contended, ตรงตามเงื่อนไขของหัวข้อ 56.10 ที่บอกว่า "เร็ว
กว่า Mutex ในกรณีไม่แข่งกัน" — เทียบกับ `Mutex` มันยังเร็วกว่ามาก แต่เทียบกับการเพิ่มค่าแบบไม่ atomic มันก็ยัง
มีค่าใช้จ่ายจริงเสมอ) คำสั่ง `lock`-prefixed พวกนี้มี latency สูงกว่าคำสั่งเพิ่มค่าธรรมดา (`inc`/`add`) อย่าง
มีนัยสำคัญเพราะเหตุผลด้าน hardware ล้วน ๆ ไม่เกี่ยวกับ contention เลย — นี่คือหลักฐานเชิงตัวเลขที่ยืนยัน
คำเตือนของ Part 51 ว่า **"atomic operation มีต้นทุนจริงแม้ไม่มี contention"** ไม่ใช่แค่ทฤษฎีลอย ๆ

ข้อสรุปเชิงปฏิบัติจากตัวเลขทั้งสามนี้: ถ้าโปรแกรมทำงานในเธรดเดียว **`Rc<T>` ควรเป็นตัวเลือกแรกเสมอ** เพราะเร็ว
กว่า `Arc<T>` แบบมีนัยสำคัญ (แม้จะดูเหมือนเล็กน้อยในหน่วย nanosecond แต่ถ้าเรียกใน hot loop หลักล้านครั้งความ
ต่างสะสมได้จริง) และสลับไปใช้ `Arc<T>` ก็ต่อเมื่อ**ต้องแชร์ข้ามเธรดจริง ๆ** (ตามที่ Part 28/39 สอนไว้อยู่แล้ว)
— แต่ไม่ว่าจะเลือกแบบไหน ทั้งสองยังถูกกว่า deep clone แบบ `String::clone()` มหาศาลอยู่ดี ตราบใดที่ข้อมูลที่
ต้องการแชร์นั้น**ไม่ต้องแก้ไขแบบเป็นเจ้าของแยกกันจริง ๆ** (ถ้าต้องแก้ไขแยกกันจริง deep clone คือคำตอบที่ถูก
ต้อง ไม่มีทางเลี่ยง — นี่คือสิ่งที่ Part 28 เตือนไว้แล้วว่า `Rc::clone()`/`.clone()` ธรรมดามีความหมายต่างกัน
โดยสิ้นเชิงแม้ชื่อ method จะเหมือนกัน)

### 56.11 Capstone: Particle Simulation แบบปรับปรุงทีละขั้น — วัดผลสะสมของทุกหลักการในบทนี้

มาปิดบทด้วยตัวอย่างจริงที่รวมทุกหลักการเข้าด้วยกัน: **particle simulation** ที่คำนวณ 1 step ของการอัปเดตตำแหน่ง
อนุภาค (ตาม pattern ที่ใช้จริงในเกม, การจำลองฟิสิกส์, และ scientific computing) เขียนสามเวอร์ชันที่ "ระมัดระวัง
เรื่องหน่วยความจำ" มากขึ้นทีละขั้น แล้ววัดด้วย `criterion` ทุกเวอร์ชัน:

```rust
// benches/capstone_bench.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};

const N: usize = 50_000;
const DT: f32 = 0.016;

#[derive(Clone, Copy)]
struct Particle {
    x: f32, y: f32, vx: f32, vy: f32, mass: f32,
}

fn new_particle(i: usize) -> Particle {
    Particle { x: i as f32, y: 0.0, vx: 0.5, vy: 0.25, mass: 1.0 }
}

// เวอร์ชัน 1: Vec<Box<Particle>> -- จอง heap แยกทีละตัว (สิ่งที่มือใหม่มักเขียนตอนพอร์ตมาจาก
// ภาษาที่ collection เก็บ "reference ไปยัง object" เป็นค่าเริ่มต้นอยู่แล้ว เช่น Java/Python
// ที่ทุกตัวแปรของ object type คือ reference บน hep โดยอัตโนมัติ ไม่มีทางเลือกอื่น)
fn v1_boxed_particles(n: usize) -> Vec<Box<Particle>> {
    (0..n).map(|i| Box::new(new_particle(i))).collect()
}

fn v1_step(particles: &mut [Box<Particle>]) -> f32 {
    let mut total_speed = 0.0;
    for p in particles.iter_mut() {
        p.x += p.vx * DT;
        p.y += p.vy * DT;
        total_speed += (p.vx * p.vx + p.vy * p.vy).sqrt();
    }
    total_speed
}

// เวอร์ชัน 2: Vec<Particle> ธรรมดา (AoS ต่อเนื่องกันในหน่วยความจำก้อนเดียว) -- ไม่มี heap
// alloc ต่อ particle อีกต่อไป มีแค่ heap alloc ก้อนเดียวสำหรับ buffer ทั้ง Vec (Part 13/42)
fn v2_flat_particles(n: usize) -> Vec<Particle> {
    (0..n).map(new_particle).collect()
}

fn v2_step(particles: &mut [Particle]) -> f32 {
    let mut total_speed = 0.0;
    for p in particles.iter_mut() {
        p.x += p.vx * DT;
        p.y += p.vy * DT;
        total_speed += (p.vx * p.vx + p.vy * p.vy).sqrt();
    }
    total_speed
}

// เวอร์ชัน 3: Struct-of-Arrays -- แยก field ที่ step ต้องใช้ (x, y, vx, vy) เป็น Vec แยกกัน
struct ParticlesSoA {
    x: Vec<f32>, y: Vec<f32>, vx: Vec<f32>, vy: Vec<f32>,
}

fn v3_soa_particles(n: usize) -> ParticlesSoA {
    ParticlesSoA {
        x: (0..n).map(|i| i as f32).collect(),
        y: vec![0.0; n], vx: vec![0.5; n], vy: vec![0.25; n],
    }
}

fn v3_step(p: &mut ParticlesSoA) -> f32 {
    let mut total_speed = 0.0;
    for i in 0..p.x.len() {
        p.x[i] += p.vx[i] * DT;
        p.y[i] += p.vy[i] * DT;
        total_speed += (p.vx[i] * p.vx[i] + p.vy[i] * p.vy[i]).sqrt();
    }
    total_speed
}

fn bench_capstone(c: &mut Criterion) {
    let mut group = c.benchmark_group("particle_sim_step_50k");

    // iter_batched: setup (สร้างข้อมูล) ไม่ถูกนับเวลา, routine (การ step + drop ของ batch
    // นั้น) ถูกนับเวลาจริง -- Part 54 สอนเทคนิคนี้ไว้แล้วสำหรับ benchmark ที่ mutate ข้อมูล
    group.bench_function("v1_vec_of_box", |b| {
        b.iter_batched(
            || v1_boxed_particles(N),
            |mut particles| black_box(v1_step(black_box(&mut particles))),
            criterion::BatchSize::LargeInput,
        )
    });
    group.bench_function("v2_flat_vec_aos", |b| {
        b.iter_batched(
            || v2_flat_particles(N),
            |mut particles| black_box(v2_step(black_box(&mut particles))),
            criterion::BatchSize::LargeInput,
        )
    });
    group.bench_function("v3_struct_of_arrays", |b| {
        b.iter_batched(
            || v3_soa_particles(N),
            |mut particles| black_box(v3_step(black_box(&mut particles))),
            criterion::BatchSize::LargeInput,
        )
    });

    group.finish();
}

criterion_group!(benches, bench_capstone);
criterion_main!(benches);
```

ผลลัพธ์จริงจาก `cargo bench --bench capstone_bench` (50,000 particles ต่อ step):

```
particle_sim_step_50k/v1_vec_of_box
                        time:   [592.37 µs 621.96 µs 654.14 µs]
particle_sim_step_50k/v2_flat_vec_aos
                        time:   [52.871 µs 53.450 µs 54.064 µs]
particle_sim_step_50k/v3_struct_of_arrays
                        time:   [67.109 µs 71.974 µs 79.903 µs]
```

#### 56.11.1 อ่านผลลัพธ์อย่างซื่อตรง — รวมทั้งส่วนที่ "ขัดกับสัญชาตญาณ"

| เวอร์ชัน | เวลาเฉลี่ย | เทียบกับ v1 |
|---|---|---|
| v1: `Vec<Box<Particle>>` | ~621.96 µs | 1× (baseline) |
| v2: `Vec<Particle>` (AoS ต่อเนื่อง) | ~53.45 µs | **เร็วขึ้น ~11.6×** |
| v3: Struct-of-Arrays | ~71.97 µs | เร็วขึ้น ~8.6× (แต่**ช้ากว่า v2** ~1.35×) |

**ข้อค้นพบที่ 1 — การก้าวจาก v1 ไป v2 คือก้าวที่ทรงพลังที่สุดในทั้งบท**: แค่เปลี่ยนจาก "heap allocate แยกทุก
particle" เป็น "heap allocate ครั้งเดียวสำหรับทั้งชุด" ทำให้เร็วขึ้น**เกือบ 12 เท่า** — ตัวเลขนี้มาจากสอง
ปัจจัยรวมกัน: (1) ไม่มีการเรียก allocator 50,000 ครั้งอีกต่อไป (เหลือแค่ 1 ครั้งสำหรับ `Vec` buffer ทั้งก้อน
— สอดคล้องกับหัวข้อ 56.5 ที่วัดไว้ว่า per-item heap alloc แพงกว่า preallocated `Vec` มาก) และ (2) การวัดนี้
ใช้ `iter_batched` ที่นับเวลารวมทั้ง**การ drop ของ batch นั้นด้วย** — สำหรับ v1 การ drop หมายถึงเรียก
deallocator แยก 50,000 ครั้ง (ปลด `Box` แต่ละตัว) ในขณะที่ v2/v3 ปลดแค่ 1-4 ก้อนใหญ่ **นี่คือการวัดที่ตรงกับ
ความเป็นจริงของการใช้งาน**: ต้นทุนของการเลือก `Vec<Box<T>>` ไม่ได้จบแค่ตอนสร้าง — มันตามมาหลอกหลอนตอน
collection หลุด scope ด้วยเสมอ

**ข้อค้นพบที่ 2 — SoA (v3) ช้ากว่า AoS (v2) ในกรณีนี้ ทั้งที่หัวข้อ 56.8 พิสูจน์ว่า SoA เร็วกว่า!** นี่ไม่ใช่
ความขัดแย้งในหลักการ แต่เป็น**หลักฐานที่ยืนยันคำเตือนท้ายหัวข้อ 56.8 พอดี**: การ step ของ particle simulation
นี้ต้องใช้ **4 field พร้อมกัน** ต่อ particle หนึ่งตัว (`x`, `y`, `vx`, `vy`) ไม่ใช่แค่ field เดียว — สำหรับ AoS
(v2) ทั้ง 4 field นี้อยู่ติดกันในหน่วยความจำก้อนเดียว (ภายใน 20 ไบต์ของ `Particle` หนึ่งตัว) การอ่านครบทั้ง 4
ค่าจึงมาจาก cache line เดียวกัน (หรือใกล้กันมาก) แต่สำหรับ SoA (v3) ทั้ง 4 field กระจายอยู่ใน **4 `Vec` คนละ
ก้อน** — CPU ต้อง track memory stream 4 เส้นพร้อมกัน (แม้แต่ละเส้นจะต่อเนื่องกันเองก็ตาม) ซึ่งกดดัน prefetcher
และ cache มากกว่าการอ่าน stream เดียวที่มีทุกอย่างที่ต้องใช้ครบในตัว — ส่วนต่างนี้ (~1.35×) เล็กกว่าส่วนต่าง
ระหว่าง v1/v2 มาก (~11.6×) แต่ก็ยังเป็นตัวเลขจริงที่วัดซ้ำได้สม่ำเสมอ ไม่ใช่ noise (ดูจากช่วงความเชื่อมั่นที่
criterion รายงานไม่ overlap กัน)

**บทเรียนสำคัญที่สุดจาก capstone นี้**: ลำดับความสำคัญของการตัดสินใจเรื่อง memory layout ควรเป็น **(1) เลี่ยง
heap allocation แยกทีละ element ก่อนเป็นอันดับแรก** (ผลกระทบมหาศาลอย่างที่เห็น ~11.6×) **แล้วค่อย (2) พิจารณา
AoS เทียบกับ SoA เป็นการปรับจูนขั้นละเอียดทีหลัง** โดยพิจารณาจาก**รูปแบบการเข้าถึงข้อมูลจริงของ operation
นั้น** (field เดียวหรือหลาย field ต่อครั้ง) — และขั้นตอนที่ (2) นี้**ต้องวัดจริงเสมอ ไม่ใช่เลือกตามความเชื่อ
ทั่วไปว่า "SoA เร็วกว่าเสมอ"** เพราะตัวอย่างนี้เองก็พิสูจน์ให้เห็นแล้วว่ามันไม่จริงเสมอไป — นี่คือหลักการ
"อย่าเดา วัด" จาก Part 54 ที่ถูกนำมาใช้จริงจนถึงจุดสุดท้ายของบทนี้

#### 56.11.2 ไม่ใช่แค่ความเร็ว: ผลต่างด้าน "พื้นที่หน่วยความจำที่ใช้" ด้วย

การวัดทั้งหมดในบทนี้โฟกัสที่**เวลา** แต่หลักการจาก Part 42 (เรื่อง `size_of`) ยังใช้วัด**พื้นที่หน่วยความจำที่
ใช้จริง** ของแต่ละเวอร์ชันได้ด้วย ซึ่งเป็นอีกมิติที่สำคัญไม่แพ้ความเร็ว (โดยเฉพาะระบบที่มีข้อจำกัดด้าน RAM เช่น
embedded หรือ mobile) ยืนยันด้วย `size_of` จริง:

```
size_of::<Particle>()      = 20 bytes   (x, y, vx, vy, mass -- ห้า f32 ไม่มี padding)
size_of::<Box<Particle>>() = 8 bytes    (แค่ตัวชี้ -- ตัวข้อมูลจริงอยู่บน heap แยกที่อื่น)
```

คำนวณพื้นที่รวมสำหรับ 50,000 particle ของทั้งสามเวอร์ชัน:

| เวอร์ชัน | พื้นที่ต่อ particle | พื้นที่รวม (50,000 ตัว) | หมายเหตุ |
|---|---|---|---|
| v1: `Vec<Box<Particle>>` | 8 ไบต์ (ใน `Vec`) + 20 ไบต์ (บน heap แยก) + **overhead ของ allocator ต่อก้อน** | ~1,400,000+ ไบต์ (บวก allocator metadata ที่นับไม่ได้ตรง ๆ) | แต่ละ 20 ไบต์ที่จองจริงมักถูก "ปัดขึ้น" เป็น size class ที่ใหญ่กว่า (หัวข้อ 56.5.1) เช่น 32 ไบต์ ทำให้เปลืองพื้นที่เพิ่มอีกจากการปัดขึ้นอย่างเดียว |
| v2: `Vec<Particle>` (AoS) | 20 ไบต์ (ต่อเนื่องในก้อนเดียว) | 1,000,000 ไบต์เป๊ะ | ไม่มี overhead ต่อ element เลย มีแค่ overhead ของ `Vec` เองครั้งเดียว (capacity/len/ptr) |
| v3: SoA (4 `Vec<f32>`) | 16 ไบต์ (4 field ที่ step ใช้ x4 ไบต์) | 800,000 ไบต์ | **น้อยที่สุด** เพราะไม่ต้องเก็บ `mass` ที่ step นี้ไม่ได้ใช้เลย (ตัดออกไปตรง ๆ ตั้งแต่ตอนออกแบบ struct) |

ตารางนี้เผยมุมที่น่าสนใจอีกมุม: **v3 (SoA) ใช้พื้นที่น้อยที่สุด** แม้จะช้ากว่า v2 เล็กน้อยในการวัดความเร็ว
(56.11.1) เพราะการออกแบบ SoA เปิดโอกาสให้ตัดสินใจ**เก็บเฉพาะ field ที่จำเป็นสำหรับ step นั้นจริง ๆ** ได้ง่าย
กว่า AoS (ถ้าเก็บ field `mass` ไว้ด้วยใน SoA ก็แค่เพิ่ม `Vec<f32>` อีกตัว ไม่กระทบ field อื่นเลย ต่างจาก AoS ที่
`mass` ติดอยู่ใน struct เดียวกับทุก field เสมอไม่ว่าจะใช้หรือไม่) — นี่คือ trade-off ที่แท้จริงระหว่าง v2/v3
ในตัวอย่างนี้: **v2 เร็วกว่าเล็กน้อยแต่ v3 ประหยัดพื้นที่กว่า** ไม่มีเวอร์ชันไหน "ชนะทุกมิติ" แบบเบ็ดเสร็จ — การ
เลือกจริงในโปรเจกต์ต้องดูว่าข้อจำกัดของระบบนั้นเป็นเรื่องความเร็วหรือพื้นที่เป็นหลัก (หรือทั้งสองอย่างพร้อมกัน
ถ้าเป็นไปได้) ส่วนที่**ทุกเวอร์ชันของ v1 แพ้ทั้งสองมิติพร้อมกันอย่างชัดเจน** (ช้าที่สุดและกินพื้นที่มากที่สุด
เพราะ overhead ของ allocator ต่อก้อนที่ปัดขึ้นตาม size class) คือเหตุผลที่มันควรเป็นตัวเลือกแรกที่ถูกตัดออกจาก
การพิจารณาเสมอ ไม่ว่าจะให้ความสำคัญกับความเร็วหรือพื้นที่ก็ตาม

## กับดักที่พบบ่อย (Common Pitfalls)

### กับดักที่ 1: เข้าใจผิดว่า "Zero-cost" แปลว่า "ไม่มีต้นทุนอะไรเลยในโปรแกรม"

นี่คือความเข้าใจผิดที่พบบ่อยที่สุดเกี่ยวกับคำว่า "zero-cost abstraction" — มันไม่ได้แปลว่า **"การกระทำทุกอย่าง
ในโปรแกรม Rust ไม่มีต้นทุน"** แต่แปลว่า **"abstraction (เช่น generics, iterator, trait) ไม่เพิ่มต้นทุนใด ๆ
เหนือกว่าที่จำเป็นสำหรับ computation เดียวกันที่เขียนมือ"** — heap allocation ยังแพงเหมือนเดิม (หัวข้อ 56.5
วัดไว้ชัดว่าแพงกว่า stack ~11.8×), deep clone ยังคัดลอกข้อมูลจริงเหมือนเดิม, atomic operation ภายใต้
contention ยังช้าเหมือนเดิม — สิ่งที่ zero-cost รับประกันคือ**การเลือกใช้ generic function แทนโค้ดเฉพาะทาง หรือ
iterator chain แทน manual loop จะไม่ทำให้ช้าลงเลย** ไม่ใช่ "ทุกอย่างในโปรแกรมฟรี" ถ้าเจอใครพูดว่า "Rust
zero-cost แปลว่าเขียนยังไงก็เร็วเท่ากันหมด" นั่นคือความเข้าใจผิดที่ต้องแก้ทันที — ใช้ตารางเช็คลิสต์ในหัวข้อ
56.10 เป็นตัวช่วยจำแนกให้แม่นยำเสมอ

### กับดักที่ 2: เชื่อว่า Struct-of-Arrays เร็วกว่า Array-of-Structs เสมอ

หัวข้อ 56.8 พิสูจน์ว่า SoA เร็วกว่า AoS ~18% สำหรับ workload ที่เข้าถึง field เดียว แต่หัวข้อ 56.11 พิสูจน์ตรง
กันข้ามว่า SoA **ช้ากว่า** AoS ~1.35× สำหรับ workload ที่เข้าถึง 4 field พร้อมกัน — ถ้าใครจำแค่ "SoA เร็วกว่า"
จากที่ไหนมาแล้วรีบไปแปลง data structure ทั้งโปรเจกต์เป็น SoA โดยไม่วัดก่อน อาจได้ผลตรงกันข้ามกับที่ตั้งใจ
วิธีป้องกันคือ**ต้องระบุก่อนเสมอว่า operation ที่จะ optimize เข้าถึงกี่ field ต่อ element** ถ้าเข้าถึง 1-2
field จากหลาย ๆ field → ลองพิจารณา SoA (แล้ววัดยืนยัน) ถ้าเข้าถึงเกือบทุก field พร้อมกันเสมอ → AoS มักชนะหรือ
เสมอกันอยู่แล้ว และ**ต้องวัดด้วย `criterion` เสมอทุกครั้ง**ไม่มีข้อยกเว้น เพราะผลจริงขึ้นกับขนาด struct, จำนวน
element, และสถาปัตยกรรม CPU ที่รันจริงด้วย

### กับดักที่ 3: ประกาศ `#[global_allocator]` มากกว่าหนึ่งตัว (หรือในหลาย crate ของ workspace)

หัวข้อ 56.6 แสดง error จริงของการประกาศ `#[global_allocator]` สองตัวในไฟล์เดียว — แต่กับดักที่ซับซ้อนกว่าและ
พบได้จริงในโปรเจกต์ใหญ่คือ: ถ้า workspace มีหลาย crate และมากกว่าหนึ่ง crate ประกาศ `#[global_allocator]`
ของตัวเอง (เช่น library หนึ่งอยากใช้ `jemalloc` โดยประกาศไว้ใน `lib.rs` ของตัวเอง แล้ว binary crate ที่ import
library นั้นก็ประกาศ allocator ของตัวเองอีกตัวโดยไม่รู้ว่า library ประกาศไว้แล้ว) จะได้ error แบบเดียวกันทันที
ตอน link — **กฎที่ต้องจำ**: `#[global_allocator]` เป็นสิทธิ์ของ**ปลายทางสุดท้าย**เท่านั้น (executable binary
ตัวสุดท้ายที่ผู้ใช้รัน) ไม่ใช่ของ library — ถ้าเขียน library ที่ต้องการแนะนำ allocator เฉพาะทาง ควรทำผ่าน
feature flag ให้ผู้ใช้ปลายทางเลือกเอง ไม่ใช่ประกาศ `#[global_allocator]` ไว้ใน library โดยตรง เพราะจะไปชนกับ
การเลือกของผู้ใช้ปลายทางได้ทันที

### กับดักที่ 4: คิดว่าการเรียก `const fn` จะฟรีเสมอไม่ว่าจะเรียกอย่างไร

หัวข้อ 56.4 พิสูจน์ด้วย assembly ว่า `get_fib_30()` (เรียกด้วยค่าคงที่) หายไปจาก runtime โดยสิ้นเชิง แต่
`get_fib_30_runtime(n)` (เรียกด้วย parameter ธรรมดา) ยังมี `call` และ loop คำนวณจริงเต็มรูปแบบ ทั้งที่เรียก
`const fn` ตัวเดียวกัน — คนที่เข้าใจผิดมักคิดว่าแค่ทำเครื่องหมาย `const fn` ไว้ก็เพียงพอให้ compiler คำนวณ
ล่วงหน้าเสมอไม่ว่าจะเรียกแบบไหน:

```rust
const fn expensive_calculation(n: u64) -> u64 {
    // สมมติว่าซับซ้อนมาก
    (0..n).sum()
}

fn process(user_input: u64) -> u64 {
    // user_input รู้ค่าแค่ตอน runtime -- ต่อให้ expensive_calculation เป็น const fn
    // การเรียกแบบนี้ก็ยังคำนวณตอน runtime เต็มรูปแบบ ไม่มีทางคำนวณล่วงหน้าได้เลย
    // เพราะ compiler ไม่รู้ค่า user_input ตอน compile time
    expensive_calculation(user_input)
}
```

`const fn` เป็นแค่ **"ใบอนุญาตให้เรียกในบริบท const ได้ถ้าผู้เรียกต้องการ"** ไม่ใช่ "คำสั่งบังคับให้คำนวณ
ล่วงหน้าเสมอ" — ประโยชน์ของ zero runtime cost เกิดขึ้นจริงก็ต่อเมื่อ**ผู้เรียกวางมันไว้ใน const context จริง**
(เช่น `const X: u64 = expensive_calculation(100);` หรือใช้กำหนดขนาด array/const generic) ถ้า argument มา
จาก runtime (`user_input`, ค่าที่อ่านจากไฟล์, ค่าที่รับจาก network) มันก็เป็นฟังก์ชันธรรมดาที่ต้องรันจริงเหมือน
ฟังก์ชันทั่วไปทุกประการ

### กับดักที่ 5: เชื่อผลจาก `objdump`/benchmark ที่รันบน debug build โดยไม่ตั้งใจ

หลักฐานทั้งหมดในบทนี้ (ทั้ง assembly และตัวเลข `criterion`) มาจากการ compile ด้วย `cargo build --release`
หรือ `cargo bench` (ที่ Part 54 สอนไว้ว่าใช้ profile `bench` ซึ่ง inherit จาก `release`) เท่านั้น — ถ้าทำ
การทดลองแบบเดียวกันนี้ด้วย debug build ธรรมดา (`cargo build` เปล่า ๆ, ไม่มี `--release`) ผลลัพธ์จะ**ผิดไปคนละ
เรื่อง**ทันที: generic function จะไม่ถูก inline จนเหมือนกันแบบในหัวข้อ 56.2 เลย (debug build ปิด optimization
เกือบทั้งหมด รวม inlining), iterator chain จะช้ากว่า manual loop อย่างเห็นได้ชัด (เพราะ closure/adaptor ที่ไม่
ถูก inline ใน debug build มี overhead ของการเรียกฟังก์ชันจริง ๆ) และตัวเลข `criterion` จาก debug build ไม่มี
ความหมายอะไรเกี่ยวกับความเร็วจริงของโปรแกรม production เลย (Part 54 ย้ำเรื่องนี้ไว้แล้ว) — **ก่อนสรุปผลอะไรก็
ตามเกี่ยวกับ performance ให้ตรวจสอบเสมอว่ากำลังดูผลจาก `--release` build จริง** ไม่ใช่ debug build ที่ปิด
optimization ไว้เกือบหมด

### กับดักที่ 6: ลืมว่า `b.iter_batched()` นับเวลารวมการ `Drop` ของ batch นั้นด้วย

หัวข้อ 56.11 ใช้ `iter_batched` เพื่อแยก "เวลาที่ใช้สร้างข้อมูลทดสอบ" (setup, ไม่ถูกนับ) ออกจาก "เวลาที่ใช้รัน
โค้ดที่ต้องการวัดจริง" (routine, ถูกนับ) ตามที่ Part 54 สอนไว้ — แต่มีรายละเอียดเล็ก ๆ ที่มักถูกมองข้าม: **ถ้า
closure ของ routine รับ ownership ของข้อมูล (`|mut particles| { ... }`) แล้วไม่ได้ return ข้อมูลนั้นออกมา
ข้อมูลนั้นจะถูก `Drop` ณ จุดสิ้นสุดของ closure ซึ่ง**อยู่ภายในเวลาที่ถูกวัด**เสมอ** — สำหรับ v1
(`Vec<Box<Particle>>`) นี่หมายความว่าตัวเลข ~622 µs ที่วัดได้ไม่ได้เป็นแค่ "เวลา iterate + คำนวณ" อย่างเดียว
แต่รวม**เวลาที่ใช้ deallocate 50,000 heap block แยกกันด้วย** — ถ้าตั้งใจจะวัด**แค่**ความเร็วของการ iterate/
คำนวณ (ไม่รวม drop) ต้องเปลี่ยนวิธีเขียน routine ให้ `return` ข้อมูลออกมาแทน (เช่น `|mut particles| {
v1_step(&mut particles); particles }`) เพื่อให้ `Drop` เกิดขึ้น**นอก**closure ที่ criterion จับเวลา (criterion
วัดแค่ระยะเวลาของการ "เรียก routine" ไม่รวมช่วงที่ต้อง drop ค่าที่ routine คืนกลับมา ซึ่งเกิดขึ้นหลังจากเก็บ
ตัวเลขเวลาไปแล้ว) — ทั้งสองวิธีวัด "สิ่งที่ต่างกัน" และ**ทั้งคู่ถูกต้อง** ขึ้นอยู่กับว่าคำถามที่อยากตอบคืออะไร
— บทนี้เลือกวัดแบบรวม drop ไว้ด้วยโดยตั้งใจ (ตามที่ระบุไว้อย่างชัดเจนในหัวข้อ 56.11.1) เพราะการ deallocate
คือส่วนหนึ่งของ "ต้นทุนรวมของการเลือกใช้ `Vec<Box<T>>`" ที่นักพัฒนาจะต้องเจอจริงเสมอเมื่อ collection หลุด scope
— ข้อคิดสำคัญคือ**ต้องรู้ตัวเสมอว่ากำลังวัดอะไรอยู่กันแน่** ไม่ใช่แค่เชื่อตัวเลขที่ `criterion` พิมพ์ออกมาโดย
ไม่เข้าใจว่ามันครอบคลุมขั้นตอนไหนบ้าง

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** เขียนฟังก์ชัน generic `fn max_of<T: PartialOrd + Copy>(a: T, b: T) -> T` (คืนค่าที่มากกว่าระหว่าง
   สองค่า) แล้วสร้าง wrapper `#[no_mangle] #[inline(never)] pub fn max_of_u8(a: u8, b: u8) -> u8` ที่เรียก
   `max_of::<u8>` ข้างใน คู่กับฟังก์ชัน `#[no_mangle] #[inline(never)] pub fn specific_max_u8(a: u8, b: u8) ->
   u8` ที่เขียนด้วย `if a > b { a } else { b }` ตรง ๆ ไม่มี generic เกี่ยวข้อง จากนั้น compile ด้วย `cargo
   build --release` แล้วใช้ `nm -C target/release/<ชื่อไบนารี>` ตรวจสอบว่า address ของทั้งสอง symbol เท่ากัน
   หรือไม่ (ตามวิธีในหัวข้อ 56.2) — ถ้า `nm`/`objdump` ไม่มีในเครื่องของคุณ ให้เขียน `criterion` benchmark
   เทียบสองฟังก์ชันนี้แทน แล้วรายงานว่าเวลาต่างกันภายในช่วงความเชื่อมั่นของกันและกันหรือไม่ (Part 54)
   *Hint*: ผลลัพธ์ที่ควรได้คือทั้งสอง symbol ชี้ไปที่ address เดียวกันเป๊ะ เหมือนตัวอย่าง `generic_add_i32`/
   `specific_add_i32` ในหัวข้อ 56.2 เพราะ `u8` comparison ก็เป็นการคำนวณง่ายพอที่ LLVM จะ inline+ยุบรวมได้แบบ
   เดียวกัน

2. **[กลาง]** เขียน `criterion` benchmark เทียบ `String::clone()` (deep copy) กับ `Rc<String>::clone()`
   (pointer + counter) สำหรับ `String` ที่มีความยาวประมาณ 10,000 ตัวอักษร รันด้วย loop 1,000 รอบต่อการวัดหนึ่ง
   ครั้ง แล้วรายงานอัตราส่วนความเร็วที่วัดได้จริง (คาดหวังว่า `Rc::clone()` ควรเร็วกว่ามาก เพราะไม่มีการคัดลอก
   ตัวอักษรจริงเลย ตรงกับหลักการที่ Part 28 สอนไว้) พร้อมอธิบายว่าทำไมความต่างที่วัดได้ถึงสอดคล้อง (หรือไม่
   สอดคล้อง) กับตัวเลข ~11.8× ที่หัวข้อ 56.5 วัดไว้สำหรับ heap allocation ต่อ element
   *Hint*: ใช้ `Criterion::benchmark_group` กับสอง `bench_function` เหมือนตัวอย่างในหัวข้อ 56.5 ครอบด้วย
   `black_box` ทั้ง input และ output ตามหลักการ Part 54

3. **[ยาก]** ขยาย benchmark ในหัวข้อ 56.8 (AoS vs SoA) ให้มี**เวอร์ชันที่สาม** เรียกว่า **AoSoA** (Array of
   Struct-of-Arrays) — แบ่งข้อมูลเป็น "chunk" ขนาดคงที่ (เช่น 8 particle ต่อ chunk) แล้วภายในแต่ละ chunk เก็บ
   แบบ SoA (คือมี `Vec<ParticleChunk>` ที่ `ParticleChunk` มี field เป็น `[f32; 8]` แยกตาม field เดิม) วัด
   ประสิทธิภาพของการสรุปค่า `x` เทียบกับทั้ง AoS และ SoA เดิม แล้วอธิบายว่า AoSoA พยายามแก้ปัญหาอะไรที่ทั้ง
   AoS และ SoA ล้วน ๆ มีข้อเสีย (คำใบ้: มันคือความพยายามรวมข้อดีของทั้งสองแบบ — cache locality ของ field ที่
   เกี่ยวข้องกัน (ใกล้เคียง AoS) กับการไม่เสีย bandwidth ไปกับ field ที่ไม่ใช้ (ใกล้เคียง SoA))
   *Hint*: เทคนิคนี้ใช้จริงใน game engine และ scientific computing ระดับ production หลายตัว เพราะให้ผลลัพธ์
   ที่มัก "ดีในทั้งสองโลก" โดยไม่ต้องเลือกสุดทางแบบใดแบบหนึ่ง

4. **[ยาก/ประยุกต์]** implement **bump allocator** ของตัวเอง (allocator ที่ไม่มีการ `free` จริงเลย — แค่เลื่อน
   pointer ไปข้างหน้าทุกครั้งที่ `alloc` แล้ว "ลืม" ทุกอย่างพร้อมกันตอนจบ arena) โดย implement `unsafe impl
   GlobalAlloc for BumpAllocator` ที่ถือ buffer ขนาดคงที่ไว้ภายใน (เช่น `UnsafeCell<[u8; 1_000_000]>` กับ
   `AtomicUsize` เก็บตำแหน่งปัจจุบัน) ทำ `dealloc` เป็น no-op (ไม่ทำอะไรเลย) แล้วเขียนโปรแกรมทดสอบเล็ก ๆ ที่
   allocate หลายพันครั้งแล้ววัดความเร็วเทียบกับ `System` allocator ด้วย `criterion` จากนั้นอธิบายเป็นข้อความ
   3-4 บรรทัดว่า trade-off ของ bump allocator แบบนี้เหมาะกับ workload แบบไหน (คำใบ้: คำตอบเกี่ยวข้องกับคำว่า
   "arena allocation pattern" — สถานการณ์ที่ allocate ข้อมูลจำนวนมากแล้วปล่อยทิ้งทั้งชุดพร้อมกันในจังหวะเดียว
   เช่น การประมวลผลหนึ่ง request ของเว็บเซิร์ฟเวอร์ที่ข้อมูลทั้งหมดของ request นั้นไม่จำเป็นอีกต่อไปทันทีที่ตอบ
   กลับเสร็จ) — และอธิบายว่าทำไม bump allocator แบบนี้ถึง**ไม่เหมาะ**เป็น global allocator ของโปรแกรมทั่วไปที่
   รันไปเรื่อย ๆ ไม่มีจุดจบ (คำใบ้: ลองคิดว่าถ้า allocate ไปเรื่อย ๆ โดยไม่มีการ "รีเซ็ต" arena เลย จะเกิด
   อะไรขึ้นกับ buffer ขนาดคงที่ 1,000,000 ไบต์นั้นในที่สุด)

## สรุป

บทนี้คือ**บทปิดของสายเนื้อหาระดับระบบและประสิทธิภาพ**ที่หลักสูตรสร้างมาตั้งแต่ Part 41 — และเป็นบทที่ทำสิ่งที่
ไม่มีบทไหนก่อนหน้าทำมาก่อน: **พิสูจน์**คำสัญญา "zero-cost abstraction" ที่ถูกพูดซ้ำแล้วซ้ำเล่าตั้งแต่ Part 1
ด้วยหลักฐานเชิงประจักษ์จริงสามชุด — generic function ที่ monomorphize แล้ว linker ยุบรวมเป็น machine code
เดียวกันเป๊ะกับโค้ดเขียนมือ (Identical Code Folding), iterator chain ที่ได้รับ optimization ระดับเดียวกับ
manual loop (branchless `cmov`, loop unrolling), และ `const fn` ที่คำนวณเสร็จโดย `rustc` จนหายไปจาก runtime
code โดยสิ้นเชิง — ทั้งสามข้อนี้ไม่ใช่การท่องจำคำโฆษณาอีกต่อไป แต่คือสิ่งที่เราเห็นด้วยตาตัวเองผ่าน `nm` และ
`objdump` บนไบนารีจริง

จากนั้นเราวัด**ต้นทุนที่มีอยู่จริง**ด้วย `criterion` (Part 54) เพื่อคานกับความเข้าใจผิดที่อาจเกิดจากคำว่า
"zero-cost" — heap allocation ต่อ element แพงกว่า stack allocation ถึง ~11.8× สำหรับข้อมูลขนาดเล็ก, การ
จัดเรียงข้อมูลแบบ struct-of-arrays เร็วกว่า array-of-structs ~18% สำหรับ workload ที่เข้าถึง field เดียว แต่
กลับ**ช้ากว่า**ในตัวอย่าง capstone ที่ operation ต้องใช้หลาย field พร้อมกัน — ทุกตัวเลขเหล่านี้มาจากการรัน
จริง ไม่ใช่การประมาณ ตามหลักการ "อย่าเดา วัด" ที่ Part 54 ปลูกฝังไว้ และยังคงเป็นหลักการที่ใช้ได้กับทุกคำถาม
เรื่องประสิทธิภาพไม่มีข้อยกเว้น เราปิดท้ายด้วยเครื่องมือระดับ awareness สองตัว (`#[global_allocator]`/
`GlobalAlloc` สำหรับสลับ allocator ทั้งโปรแกรม และ `#![no_std]` สำหรับโลกที่ไม่มี OS ให้พึ่งพา) ที่เป็นสะพาน
เชื่อมไปยังเนื้อหา embedded/systems programming ที่ลึกกว่านี้ในโมดูลถัดไปของหลักสูตร

**เช็คลิสต์ต้นทุนหน่วยความจำ** ในหัวข้อ 56.10 คือสิ่งที่ควรพกติดตัวไปตลอดการเขียนโค้ด Rust ต่อจากนี้: แยกแยะ
ให้ได้เสมอว่าอะไร**ฟรีเสมอ** (move, generics ที่ monomorphize, iterator chain, `const fn` ในบริบท const,
borrow), อะไร**ถูกแต่ไม่ฟรี** (`Rc`/`Arc::clone`, `Vec::push`, non-contended atomic) และอะไร**แพงจริง**
(heap allocation ต่อ element, deep clone, dynamic dispatch ใน hot loop, atomic ภายใต้ contention) —
ความสามารถในการแยกแยะนี้คือสิ่งที่ทำให้นักพัฒนา Rust เขียนโค้ดที่ทั้ง**ถูกต้อง**และ**เร็ว**โดยไม่ต้องเดา และ
ตัวอย่าง capstone (56.11) ยืนยันบทเรียนที่สำคัญที่สุดของทั้งบท: **การเลือก data structure ระดับพื้นฐาน (เลี่ยง
heap allocation แยกทีละ element) มีผลกระทบมากกว่าการปรับจูนระดับละเอียด (AoS เทียบกับ SoA) อย่างมหาศาล** —
ควรแก้ปัญหาใหญ่ก่อนแล้วค่อยไล่แก้ปัญหาเล็กทีหลัง เสมอ

ตัวเลขอื่น ๆ ที่วัดได้จริงตลอดบทนี้ก็ควรติดตัวไปเป็นสัญชาน มากกว่าจะเป็นแค่ทฤษฎีที่จำไว้เฉย ๆ: `Rc::clone()`
เร็วกว่า `Arc::clone()` ราว 13.4 เท่า (เพราะ atomic instruction มีต้นทุนจริงแม้ไม่มี contention) และทั้งสองเร็ว
กว่า deep clone ของ `String` มหาศาล (61.8× และ 4.6× ตามลำดับ) — ตัวเลขเหล่านี้ไม่ได้มีไว้ให้จำแบบท่องสูตร แต่มี
ไว้ให้ซึมเข้าไปเป็นสัญชาตญาณว่า **การเลือกระหว่าง `Rc<T>`, `Arc<T>`, และ `.clone()` แบบ deep ไม่ใช่แค่ทาง
เลือกด้าน ownership (Part 28) แต่เป็นทางเลือกด้านประสิทธิภาพที่วัดผลต่างได้จริงเสมอ** และเครื่องมือระดับ
awareness สองตัวปิดท้าย (`#[global_allocator]`/`GlobalAlloc` ที่ใช้สลับไปเป็น `mimalloc` ได้จริงด้วยโค้ดแค่
สองบรรทัด และ `#![no_std]` ที่เปิดให้ใช้ `Vec`/`String` ผ่าน crate `alloc` ได้แม้ไม่มี OS เลย) คือหลักฐานว่า
ระดับควบคุมหน่วยความจำของ Rust ไม่ได้จบอยู่แค่ "ปลอดภัยโดยไม่มี garbage collector" แต่ยังเปิดช่องให้ปรับแต่ง
ได้ลึกถึงระดับที่ภาษาส่วนใหญ่ไม่เปิดให้ทำเลยด้วยซ้ำ โดยที่ความปลอดภัยของ type system ยังคุ้มกันอยู่ครบทุกชั้น

ด้วยบทนี้ **โมดูล 3: ระดับสูง (Advanced)** ที่เริ่มต้นจาก Unsafe Rust ใน Part 41 ผ่าน raw pointers, FFI, macro,
async, atomics, design patterns, และ performance/profiling ก็มาถึงจุดปิดที่สมบูรณ์: คุณมีเครื่องมือครบสำหรับ
เข้าใจว่าโปรแกรม Rust ทำงานกับหน่วยความจำและ CPU จริงอย่างไรในทุกระดับ ตั้งแต่ raw byte ไปจนถึง abstraction
ระดับสูงสุด ต่อจากนี้หลักสูตรจะ**เปลี่ยนโหมด**ไปสู่เนื้อหาระดับ**แอปพลิเคชันในชีวิตประจำวัน**มากขึ้น — **Part
57: Serialization: Serde เบื้องต้น** จะเริ่มสอนการแปลงข้อมูล Rust ไปมากับ JSON/format อื่น ๆ ที่เป็นงานที่
โปรแกรมเมอร์ Rust ในโลกจริงทำกันทุกวัน ไม่ต้องกังวลว่าความรู้เรื่อง memory ระดับลึกที่เรียนมาทั้งหมดจะไม่ได้ใช้
อีก — ความเข้าใจเรื่อง ownership, allocation cost, และ zero-cost abstraction ที่แม่นยำนี้จะติดตัวไปเป็นสัญชาน
เบื้องหลังทุกโค้ดที่คุณเขียนต่อจากนี้ แม้จะไม่ต้องพูดถึงมันตรง ๆ ทุกบรรทัดอีกแล้วก็ตาม

---

**Part ก่อนหน้า:** [Profiling Rust Applications](part-055-profiling.md) | **Part ถัดไป:**
[Serialization: Serde เบื้องต้น](part-057-serde-basics.md)
