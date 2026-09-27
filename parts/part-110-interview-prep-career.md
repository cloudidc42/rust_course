# Part 110: เตรียมตัวสัมภาษณ์งาน Rust Developer และแนวทางอาชีพ

> โมดูล: Production, DevOps และระดับมืออาชีพ | ระดับ: มืออาชีพ | เวลาโดยประมาณ: 240 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- ประเมินตลาดงาน Rust ได้อย่างตรงกับความจริง รู้ว่าควรมองหางานในสายไหน และควรตั้งความคาดหวังอย่างไรให้สมเหตุสมผล
  ไม่โอเวอร์และไม่ท้อแท้เกินไป
- ตอบคำถามสัมภาษณ์เชิงเทคนิคหัวข้อหลักของ Rust ได้อย่างมีโครงสร้าง — ownership/borrowing, `String` vs `&str`,
  ปรัชญาการจัดการ error, trait objects vs generics, concurrency (`Send`/`Sync`), และพื้นฐาน async — พร้อมยกตัวอย่าง
  โค้ดจริงประกอบคำตอบได้ ไม่ใช่แค่ท่องนิยาม
- แก้โจทย์ live coding ที่พบบ่อยในสัมภาษณ์ Rust ได้ 3 ประเภทหลัก: การสร้าง data structure ที่ต้องคิดเรื่อง ownership,
  การเขียน iterator adapter ของตัวเอง, และการแก้ error ของ borrow checker ในโค้ดที่มีบั๊ก
- พูดคุยโจทย์ system design ในสไตล์ที่ผู้สัมภาษณ์สาย Rust คาดหวัง คือการใช้ type system เป็นเครื่องมือ encode invariant
  ของระบบ ไม่ใช่แค่บอกว่า "จะใช้ service อะไรบ้าง"
- สร้าง portfolio, เขียน resume, และตอบคำถามเชิง behavioral ในแบบที่แสดงให้เห็นว่าคุณเข้าใจ trade-off จริง ๆ
  ไม่ใช่แค่ท่องคำศัพท์ Rust
- วางแผนเส้นทางอาชีพระยะยาวในฐานะ Rust developer ได้ พร้อมมีกรอบคิดเรื่อง salary/leveling ที่ไม่ยึดติดกับตัวเลข
  ที่ล้าสมัยเร็ว และมองเห็นภาพรวมของทั้งหลักสูตรที่ได้เรียนมาตั้งแต่ต้นจนจบ

## ความรู้ที่ต้องมีมาก่อน

บทนี้เป็นบทสรุปของ**ทั้งหลักสูตร** จึงอ้างอิงเนื้อหาจากทุกโมดูลเพื่อ "แปล" มันเป็นทักษะที่ใช้ตอบคำถามสัมภาษณ์ได้จริง
โดยเฉพาะ:

- Module 1 (พื้นฐานภาษา): Part 6-7 (Ownership/Borrowing), Part 8 (Slices), Part 12 (Result และ error handling),
  Part 14 (String และการจัดการข้อความ)
- Module 2 (ระดับกลาง): Part 21-22 (Traits/Generics ขั้นสูง), Part 30 (Error handling ขั้นสูง), Part 39-40
  (Mutex/Arc, Send/Sync)
- Module 3 (ระดับสูง): Part 46-48 (Async/Await, Futures, Tokio), Part 52-53 (Design Patterns)
- Module 4 (Web Development): Part 61, 69, 78-79, 81 (HTTP/REST พื้นฐาน, การเลือก framework, REST/GraphQL design,
  Microservices)
- Module 5 (Full-Stack/WASM): ภาพรวมของ WebAssembly และ full-stack Rust
- Module 6 (บทก่อนหน้าในโมดูลเดียวกันนี้): Part 96-101 (Docker, CI/CD, Observability, Security, Deployment),
  Part 102 (Embedded), Part 104 (Blockchain), Part 105 (Open Source), Part 106 (Enterprise Design Patterns),
  Part 107-108 (Capstone CLI/Web Service), Part 109 (Code Review และ Clean Code)

ถ้าคุณอ่านหลักสูตรนี้มาตามลำดับจนถึงบทนี้ คุณมีพื้นฐานเพียงพอแล้ว บทนี้จะไม่สอนหัวข้อใหม่ แต่จะช่วยคุณ "ประกอบ" ความรู้
ที่กระจายอยู่ใน 109 บทที่แล้วให้กลายเป็นคำตอบสัมภาษณ์ที่ชัดเจนและมีน้ำหนัก

## เนื้อหา

### 1. ตลาดงาน Rust ในปัจจุบัน: ความจริงที่ต้องรู้ก่อนเตรียมตัว

ก่อนจะฝึกตอบคำถามสัมภาษณ์สักคำถามเดียว สิ่งที่สำคัญกว่าคือการตั้งความคาดหวังให้ตรงกับความจริง เพราะการเตรียมตัวที่ดี
ที่สุดในโลกก็ช่วยไม่ได้ถ้าคุณเล็งไปผิดตลาด

**ความจริงข้อที่ 1: Rust ไม่ใช่ภาษาที่มีตำแหน่งงานมากเท่า Python, JavaScript, หรือ Java**

ถ้าเปิดเว็บหางานแล้ว filter ด้วยคำว่า "Rust" เทียบกับ "Python" หรือ "JavaScript" คุณจะเห็นความแตกต่างของจำนวนตำแหน่ง
อย่างชัดเจน — เป็นเรื่องจริงที่ควรรับรู้ตรง ๆ ไม่ใช่เรื่องที่ต้องปิดบังหรือกลัว เหตุผลไม่ใช่เพราะ Rust "ไม่ดี"
แต่เพราะ:

- Rust ยังเป็นภาษาที่ค่อนข้างใหม่ในระดับ mainstream (เริ่มเสถียรจริงจังราวปี 2015 ด้วย Rust 1.0) เทียบกับ Python/Java
  ที่มีมาหลายสิบปีและมี codebase สะสมจำนวนมหาศาลที่ต้องมีคนดูแลต่อ
- ระบบเดิม (legacy system) จำนวนมากที่ต้องการคนดูแลยังเขียนด้วยภาษาอื่น การเปลี่ยนมาใช้ Rust ทั้งระบบมักไม่ใช่
  priority อันดับหนึ่งขององค์กรส่วนใหญ่
- Rust มี learning curve ที่สูงกว่าเฉลี่ย (คุณคงสัมผัสได้เองจากการเรียน ownership/borrow checker ใน Part 6-7)
  ทำให้บริษัทจำนวนมากเลือกภาษาที่ทีมเรียนเร็วกว่าสำหรับงานทั่วไปที่ performance ไม่ใช่ปัจจัยชี้ขาด

**ความจริงข้อที่ 2: แต่ demand สำหรับ Rust เป็นของจริงและกำลังเติบโตต่อเนื่อง ไม่ใช่ hype ชั่วคราว**

จุดที่ทำให้ Rust ต่างจากภาษา "กระแสแรงแต่จางเร็ว" อื่น ๆ คือ Rust ถูกนำไปใช้ใน production จริงโดยองค์กรระดับ
infrastructure ที่มีความอ่อนไหวต่อ correctness และ performance สูงมาก เช่น browser engine (Firefox ใช้ Rust
ในหลายส่วนของ Servo/Gecko), cloud infrastructure, และ operating system component — งานเหล่านี้ไม่ได้เลือก Rust
เพราะกระแส แต่เพราะ memory safety ที่ compiler การันตีได้โดยไม่ต้องมี garbage collector (สิ่งที่คุณเรียนไปแล้วอย่าง
ละเอียดใน Part 6-7 และ Part 41-42 เรื่อง unsafe/raw pointer) ตอบโจทย์ปัญหาจริงที่ภาษาอื่นแก้ได้ยาก

**แผนที่ตลาดงาน Rust แบบตรงไปตรงมา แบ่งตามความหนาแน่นของ demand:**

1. **Systems/Infrastructure และ Platform Engineering** — โซนที่ Rust แน่นที่สุด บริษัท cloud infrastructure,
   database engine, networking tool จำนวนมากใช้ Rust เป็นภาษาหลักหรือภาษารองที่สำคัญ เพราะต้องการ performance
   ระดับ C/C++ แต่ไม่อยากแบกความเสี่ยงเรื่อง memory bug ทักษะที่ตรงกับ Part 41-43 (unsafe, raw pointer, FFI),
   Part 51 (atomics), Part 54-56 (performance/profiling) มีค่ามากในโซนนี้
2. **Blockchain/Crypto** — อีกโซนที่หนาแน่นจริง (ตามที่ Part 104 พูดถึง) เพราะหลาย blockchain platform สมัยใหม่
   (เช่นระบบที่ใช้ WASM-based smart contract หรือ validator client) เลือก Rust เป็นภาษาหลักตั้งแต่ต้น ความต้องการ
   correctness แบบสัมบูรณ์ (เงินคนอื่นอยู่ในนั้น) ทำให้ ownership model ของ Rust ตอบโจทย์ได้ดีเป็นพิเศษ
3. **Embedded/IoT** — ตามที่ Part 102 สอนไว้ Rust กำลังเข้ามาแทน C ในงาน embedded มากขึ้นเรื่อย ๆ โดยเฉพาะโปรเจกต์ใหม่
   ที่ยังไม่มี legacy C codebase ผูกอยู่ แต่ต้องยอมรับว่าตลาดนี้เล็กกว่า C/C++ อย่างมากในภาพรวม และองค์กรจำนวนมาก
   ยังคง C ไว้เพราะ toolchain/certification เดิมที่ลงทุนไปแล้ว
4. **WASM/Browser และ Frontend Tooling** — ตามที่เรียนใน Module 5 Rust ผ่าน WebAssembly เข้าไปมีบทบาทในเครื่องมือ
   frontend (bundler, compiler, formatter) จำนวนมากที่ผู้ใช้ไม่รู้ตัวว่าเบื้องหลังเป็น Rust แต่ตำแหน่งงานที่ "เขียน
   Rust สำหรับ WASM โดยตรง" ยังเป็นสัดส่วนเล็กเทียบกับงาน frontend ทั่วไป
5. **Backend Services ทั่วไป** — ตามที่เรียนใน Module 4 (Axum/Actix-web, Part 69) นี่คือโซนที่ **กำลังโตเร็วที่สุด**
   ในเชิงสัดส่วน บริษัท SaaS/startup จำนวนมากขึ้นเลือก Rust สำหรับ backend service ใหม่ ๆ โดยเฉพาะ service ที่ต้อง
   รับ load สูงหรือ latency-sensitive แต่ในภาพรวมตลาดนี้ยังเล็กกว่า Node.js/Go/Java มาก

**ข้อสรุปเชิงกลยุทธ์:** อย่าตั้งเป้าว่า "หางาน Rust" แบบกว้าง ๆ เพราะจะแข่งกับตำแหน่งน้อยกว่าที่ควร ให้มองว่าคุณกำลัง
หางานใน**โดเมนหนึ่ง** (backend, infra, blockchain, embedded) ที่บริษัทนั้นเลือกใช้ Rust เป็นเครื่องมือ ความรู้เรื่อง
โดเมน (เช่น เข้าใจ HTTP, database, distributed systems จาก Module 4) มักสำคัญไม่น้อยกว่าความรู้ Rust เพียว ๆ
และบริษัทจำนวนมากยินดีรับผู้ที่เขียน Rust ได้ระดับกลางแต่เข้าใจโดเมนดี มากกว่าคนที่เขียน Rust เก่งมากแต่ไม่เข้าใจ
ปัญหาทางธุรกิจที่ต้องแก้

### 2. โครงสร้างการสัมภาษณ์เชิงเทคนิค และวิธีเตรียมตัวทีละหัวข้อ

การสัมภาษณ์เชิงเทคนิคสาย Rust ส่วนใหญ่ไม่ได้ถามหัวข้อที่แปลกใหม่ แต่จะถามหัวข้อ**เดิม** ที่ทุกคนเรียนในคอร์สนี้
เพียงแต่คาดหวังว่าคุณจะอธิบายได้ลึกกว่าระดับ "ท่องนิยาม" ส่วนนี้จะยกตัวอย่างคำถามจริงและวางโครงคำตอบที่ "ดี" (strong
answer) ให้ทีละหัวข้อ — คุณควรลองพูดคำตอบเหล่านี้ออกเสียงจริง ๆ ก่อนสัมภาษณ์จริง เพราะการอธิบายด้วยคำพูดล้วน ๆ
ยากกว่าการเขียนโค้ดเงียบ ๆ อย่างมาก

#### 2.1 Ownership และ Borrowing

> **คำถามตัวอย่าง:** "อธิบาย ownership ใน Rust ให้ฟังหน่อย แล้วยกตัวอย่างว่า borrow checker ป้องกันบั๊กแบบไหนได้บ้าง
> ที่ภาษาอื่นป้องกันไม่ได้ตอน compile time"

**โครงคำตอบที่ดี:** เริ่มจากปัญหาที่ ownership แก้ ไม่ใช่เริ่มจากไวยากรณ์ ผู้สัมภาษณ์อยากรู้ว่าคุณเข้าใจ "ทำไม"
ไม่ใช่แค่ "อย่างไร" (ตรงกับที่ Part 6 เน้นย้ำ):

1. ภาษาที่มี garbage collector (Java, Python, Go) แก้ปัญหา memory management โดยเลื่อนการ dealloc ไปที่ runtime
   ซึ่งปลอดภัยแต่มี overhead และไม่การันตีเวลาที่แน่นอน (predictability ต่ำ) ภาษาที่ไม่มี GC (C, C++) เร็วแต่ผู้เขียน
   ต้องจัดการ memory เอง ซึ่งเป็นสาเหตุของบั๊กประเภท use-after-free, double-free, dangling pointer จำนวนมาก
2. Rust เลือกทางที่สาม: ให้ compiler ตรวจสอบกฎ ownership ตอน compile time แทน โดยกฎหลักคือค่าหนึ่งมีเจ้าของได้แค่คน
   เดียวในเวลาหนึ่ง ๆ (move semantics) และการยืม (`&T`/`&mut T`) ต้องเป็นไปตามกฎ "ยืมแบบอ่านได้หลายคน หรือยืมแบบเขียน
   ได้คนเดียว แต่ไม่ปนกัน" — ผลคือไม่มี data race และไม่มี dangling reference ได้ตั้งแต่ compile time โดยไม่ต้องมี
   runtime check เลย (zero-cost)
3. ยกตัวอย่างจริง: บั๊กแบบ use-after-free ในโค้ด C++ ที่คืน pointer ไปยัง local variable ที่ตายไปแล้ว เทียบกับ Rust
   ที่ compiler จะฟ้อง error ทันทีถ้าพยายามทำแบบเดียวกัน (lifetime ของ reference สั้นกว่าค่าที่มันอ้างถึงไม่ได้)

**ตัวอย่างโค้ดที่ควรพูดถึง — error E0382 (use of moved value) ที่ทุกคนเจอตอนเริ่มเรียน Rust:**

```rust
fn takes_ownership(s: String) -> usize {
    s.len()
}

fn main() {
    let name = String::from("Rustacean");
    let len = takes_ownership(name);
    println!("len = {}, name = {}", len, name); // ใช้ name หลัง move
}
```

โค้ดนี้ compile ไม่ผ่าน เพราะ `name` ถูก move เข้าไปใน `takes_ownership` แล้ว การพยายามใช้ `name` อีกครั้งหลังจากนั้น
ทำให้ compiler ฟ้อง:

```
error[E0382]: borrow of moved value: `name`
 --> src/main.rs:9:42
  |
7 |     let name = String::from("Rustacean");
  |         ---- move occurs because `name` has type `String`, which does not implement the `Copy` trait
8 |     let len = takes_ownership(name);
  |                               ---- value moved here
9 |     println!("len = {}, name = {}", len, name);
  |                                          ^^^^ value borrowed here after move
```

**วิธีแก้ที่ควรพูดถึง (และเหตุผลว่าทำไมถึงเลือกวิธีนี้ ไม่ใช่แค่บอกวิธี):** ให้ฟังก์ชันรับ `&str` (ยืม) แทนการรับ
`String` (เป็นเจ้าของ) เพราะฟังก์ชันนี้แค่ "อ่าน" ความยาว ไม่มีความจำเป็นต้องเป็นเจ้าของค่านั้นเลย:

```rust
fn takes_borrow(s: &str) -> usize {
    s.len()
}

fn main() {
    let name = String::from("Rustacean");
    let len = takes_borrow(&name);
    println!("len = {}, name = {}", len, name); // ใช้ name ได้ตามปกติ
}
```

ผลลัพธ์: `len = 9, name = Rustacean` — คำตอบที่ดีของคำถามนี้ควรปิดท้ายด้วยหลักการทั่วไปว่า **"รับ ownership ก็ต่อเมื่อ
ฟังก์ชันต้องเก็บค่านั้นไว้จริง ๆ หรือต้องทำลาย/แปลงค่านั้น ไม่ใช่รับ ownership ทุกครั้งเป็นค่าเริ่มต้น"** — นี่คือ
สัญญาณที่ผู้สัมภาษณ์มองหาว่าคุณคิดเรื่อง API design ด้วย ไม่ใช่แค่ทำให้โค้ด compile ผ่าน

#### 2.2 ความแตกต่างระหว่าง `String` และ `&str`

> **คำถามตัวอย่าง:** "เมื่อไหร่ที่คุณจะเลือกรับ parameter เป็น `String` เมื่อไหร่ที่เลือก `&str` แล้วทำไม struct
> ควรเก็บฟิลด์เป็น `String` เกือบทุกครั้งที่ไม่มี lifetime parameter"

**โครงคำตอบที่ดี:** อธิบายจากมุม memory ก่อน แล้วต่อด้วยกฎการตัดสินใจที่ใช้ได้จริง (ตรงกับ Part 8 และ Part 14):

- `String` เป็น owned, growable, heap-allocated buffer ของ UTF-8 bytes — เจ้าของมันรับผิดชอบ deallocate เมื่อออกจาก
  scope (ผ่าน `Drop`)
- `&str` เป็น "view" หรือ slice เข้าไปดู UTF-8 bytes ที่อยู่ที่อื่น (อาจเป็นส่วนหนึ่งของ `String`, หรือ string
  literal ที่ฝังอยู่ใน binary ตอน compile) — มันไม่ได้เป็นเจ้าของข้อมูล จึงมี lifetime ผูกอยู่กับแหล่งที่มา
- กฎการตัดสินใจ: **ฟังก์ชันที่แค่ "อ่าน" ข้อความ ควรรับ `&str` เสมอ** เพราะ `&String` coerce เป็น `&str` ได้อัตโนมัติ
  (deref coercion) ทำให้ฟังก์ชันรับได้ทั้ง `String` ที่มีอยู่แล้วและ string literal โดยไม่ต้อง allocate เพิ่ม —
  ในทางกลับกัน **struct ที่ต้อง "เป็นเจ้าของ" ข้อมูลระยะยาว (เก็บไว้ใน struct ที่มีชีวิตเป็นอิสระ) ควรเก็บเป็น
  `String`** เพราะถ้าเก็บเป็น `&str` จะต้องมี lifetime parameter ผูกกับ struct ทำให้ struct นั้นใช้ยากขึ้นมาก
  (ผูกกับอายุของข้อมูลต้นทาง) ซึ่งส่วนใหญ่ไม่คุ้มกับ allocation ที่ประหยัดได้

**ตัวอย่างโค้ดที่ควรพูดถึง:**

```rust
struct Product {
    // เก็บเป็น String เพราะ struct เป็นเจ้าของข้อมูลนี้ ต้องมีชีวิตอยู่ได้เองไม่ผูกกับ lifetime ใคร
    name: String,
    sku: String,
}

impl Product {
    fn new(name: &str, sku: &str) -> Self {
        // รับ &str (ยืมชั่วคราว) แล้วค่อย .to_string() เป็นเจ้าของเองตอนสร้าง
        Product {
            name: name.to_string(),
            sku: sku.to_string(),
        }
    }
}

// ฟังก์ชันที่ "แค่อ่าน" ควรรับ &str เสมอ เพราะรับได้ทั้ง String และ &'static str
// โดยไม่ต้องสนใจว่าผู้เรียกเป็นเจ้าของข้อมูลแบบไหน (deref coercion: &String -> &str)
fn shout(text: &str) -> String {
    text.to_uppercase()
}

fn main() {
    let p = Product::new("Mechanical Keyboard", "SKU-001");
    println!("{} ({})", p.name, p.sku);

    let owned = String::from("hello from heap");
    let literal: &str = "hello from binary";

    println!("{}", shout(&owned));
    println!("{}", shout(literal));
}
```

ผลลัพธ์:

```
Mechanical Keyboard (SKU-001)
HELLO FROM HEAP
HELLO FROM BINARY
```

คำตอบที่แข็งแรงมักปิดท้ายด้วยการพูดถึง UTF-8: `&str` และ `String` ทั้งคู่การันตีว่าเป็น valid UTF-8 เสมอ (ต่างจาก
`Vec<u8>` ดิบ ๆ) ซึ่งเป็นเหตุผลที่ indexing ด้วย `s[0]` ทำไม่ได้ตรง ๆ ต้องใช้ `.chars()`, `.bytes()`, หรือ
`.as_bytes()` แทน ตามที่ Part 14 อธิบายไว้ — การพูดถึงจุดนี้ได้แสดงว่าคุณเข้าใจว่าทำไม Rust ถึงออกแบบ string ต่างจาก
ภาษาอื่นที่ index string ด้วย integer ได้ตรง ๆ

#### 2.3 ปรัชญาการจัดการ Error: `Result` vs Panic

> **คำถามตัวอย่าง:** "ในระบบของคุณ เมื่อไหร่ที่คุณจะ `panic!`/`.unwrap()` เมื่อไหร่ที่คุณจะคืน `Result` แล้วทำไม
> library ที่ดีไม่ควร panic เวลาเจอ input ที่ผิด"

**โครงคำตอบที่ดี:** แยกให้ชัดระหว่าง "error ที่คาดเดาได้ (recoverable)" กับ "bug ภายในโปรแกรมเอง
(unrecoverable/invariant violation)" ซึ่งเป็นเส้นแบ่งที่ Part 12 และ Part 30 สอนไว้:

- ใช้ `Result<T, E>` เมื่อ error เกิดจากสภาพแวดล้อมภายนอกที่คาดเดาได้ล่วงหน้า เช่น ไฟล์ไม่พบ, network timeout, สินค้า
  ไม่พอในสต็อก, input ผู้ใช้ไม่ถูกต้อง — สิ่งเหล่านี้ผู้เรียกควรมีโอกาสตัดสินใจว่าจะทำอย่างไรต่อ (retry, แจ้งผู้ใช้,
  ใช้ค่า default) ไม่ควรถูกบังคับให้ crash ทั้งโปรเซส
- ใช้ `panic!`/`.unwrap()`/`.expect()` เมื่อสถานะที่เกิดขึ้นเป็นการละเมิด invariant ที่ควรเป็นไปไม่ได้ถ้าโค้ดถูกต้อง
  (แปลว่าเป็น bug จริง ๆ ไม่ใช่ error จาก input) เช่น index ที่คำนวณมาแล้วควรอยู่ในขอบเขตเสมอแต่ดันไม่ใช่ หรือค่าคงที่
  ที่ประกาศไว้ใน config เอง แต่ parse ไม่ผ่าน
- library ที่ดีไม่ควร panic กับ input จากผู้ใช้ เพราะ library ไม่รู้บริบทของ caller — caller อาจอยู่ใน context ที่
  panic ทำให้เสียหายมาก (เช่น web server ที่ควร fault-isolate แต่ละ request) การคืน `Result` ให้ caller เป็นคน
  เลือกเองว่าจะจัดการ error อย่างไรถึงจะเหมาะกับบริบทของเขา

**ตัวอย่างโค้ดที่ควรพูดถึง (custom error type + `?` operator + การใช้ panic ที่เหมาะสม):**

```rust
use std::fmt;

#[derive(Debug)]
enum OrderError {
    OutOfStock { sku: String, requested: u32, available: u32 },
    InvalidQuantity(u32),
}

impl fmt::Display for OrderError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            OrderError::OutOfStock { sku, requested, available } => write!(
                f,
                "สินค้า {} มีไม่พอ: ขอ {} แต่มีในสต็อก {}",
                sku, requested, available
            ),
            OrderError::InvalidQuantity(q) => write!(f, "จำนวนสั่งซื้อไม่ถูกต้อง: {}", q),
        }
    }
}

impl std::error::Error for OrderError {}

struct Inventory {
    available: u32,
    sku: String,
}

impl Inventory {
    // recoverable error -> คืน Result เพราะผู้เรียกควรตัดสินใจเองว่าจะทำอย่างไรต่อ
    fn reserve(&mut self, qty: u32) -> Result<(), OrderError> {
        if qty == 0 {
            return Err(OrderError::InvalidQuantity(qty));
        }
        if qty > self.available {
            return Err(OrderError::OutOfStock {
                sku: self.sku.clone(),
                requested: qty,
                available: self.available,
            });
        }
        self.available -= qty;
        Ok(())
    }
}

fn process_order(inv: &mut Inventory, qty: u32) -> Result<(), OrderError> {
    // ใช้ ? เพื่อส่ง error ต่อขึ้นไปโดยไม่ต้อง match ซ้ำซ้อน
    inv.reserve(qty)?;
    println!("จองสินค้า {} จำนวน {} หน่วยสำเร็จ", inv.sku, qty);
    Ok(())
}

fn main() {
    let mut inv = Inventory { available: 5, sku: "SKU-001".to_string() };

    match process_order(&mut inv, 3) {
        Ok(()) => println!("order 1 ผ่าน"),
        Err(e) => println!("order 1 ล้มเหลว: {}", e),
    }

    match process_order(&mut inv, 100) {
        Ok(()) => println!("order 2 ผ่าน"),
        Err(e) => println!("order 2 ล้มเหลว: {}", e),
    }

    // ตัวอย่าง "unrecoverable" ที่ควร panic: ละเมิด invariant ภายในโปรแกรมเอง (bug จริง ๆ)
    let config_value: Option<u32> = Some(42);
    let must_exist = config_value.expect("ค่านี้ถูกกำหนดไว้ตายตัวใน const ด้านบน ไม่ควรเป็น None");
    println!("config = {}", must_exist);
}
```

ผลลัพธ์:

```
จองสินค้า SKU-001 จำนวน 3 หน่วยสำเร็จ
order 1 ผ่าน
order 2 ล้มเหลว: สินค้า SKU-001 มีไม่พอ: ขอ 100 แต่มีในสต็อก 2
config = 42
```

คำตอบที่แข็งแรงควรพูดถึงด้วยว่าในโปรเจกต์จริง (ตามที่ Part 31 สอน) มักใช้ `thiserror` สำหรับ error type ของ library
(เพราะ caller ต้อง match กับ variant เฉพาะได้) และใช้ `anyhow` สำหรับ error ที่ระดับ application/binary (ที่แค่ต้อง
propagate และแสดงข้อความ ไม่ต้อง match ย่อย) — การแยกสองกรณีนี้ได้แสดงว่าคุณเข้าใจบริบทการใช้งานจริง ไม่ใช่ใช้
`anyhow::Error` ทุกที่โดยไม่คิด

#### 2.4 Trait Objects vs Generics

> **คำถามตัวอย่าง:** "อธิบายความแตกต่างระหว่าง static dispatch กับ dynamic dispatch ใน Rust แล้วยกตัวอย่างสถานการณ์
> ที่คุณจะเลือก `dyn Trait` แทน generic ทั้งที่ generic เร็วกว่า"

**โครงคำตอบที่ดี:** อธิบาย trade-off สองด้าน ไม่ใช่บอกว่าอันหนึ่ง "ดีกว่า" อีกอันเสมอ (ตรงกับ Part 21-22):

- **Generic + trait bound (static dispatch):** compiler ทำ **monomorphization** คือสร้างโค้ดแยกชุดสำหรับทุก
  concrete type ที่ถูกใช้จริงตอน compile time ผลคือเร็วที่สุด (inline ได้, ไม่มี indirection ผ่าน pointer) แต่แลกมา
  ด้วย **code bloat** (binary ใหญ่ขึ้นถ้ามีหลาย type) และข้อจำกัดว่าแต่ละจุดที่เรียกต้องรู้ type เดียวตอน compile —
  เก็บ `Vec<T>` ที่มีหลาย concrete type ปนกันไม่ได้โดยตรง
- **`dyn Trait` (dynamic dispatch):** ผูก type ตอน runtime ผ่าน vtable (ตาราง function pointer) ทำให้เก็บ
  heterogeneous collection ได้ในตัวแปรเดียว (`Vec<Box<dyn Shape>>`) และ compile เร็วกว่า/binary เล็กกว่าเพราะมีโค้ด
  ชุดเดียว แต่มี overhead เล็กน้อยจากการ lookup ผ่าน pointer indirection ทุกครั้งที่เรียก method และปิดโอกาสการ
  inline ข้าม call
- สถานการณ์จริงที่เลือก `dyn Trait`: เมื่อ**ต้องเก็บ object ต่างชนิดกันในโครงสร้างเดียว** เช่น plugin system ที่โหลด
  handler หลายแบบตอน runtime, หรือ event system ที่มี listener หลายชนิด — ในกรณีนี้ความยืดหยุ่นสำคัญกว่า overhead
  เล็กน้อยของ vtable lookup

**ตัวอย่างโค้ดที่ควรพูดถึง:**

```rust
trait Shape {
    fn area(&self) -> f64;
    fn name(&self) -> &str;
}

struct Circle {
    radius: f64,
}
impl Shape for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.radius * self.radius
    }
    fn name(&self) -> &str {
        "circle"
    }
}

struct Rectangle {
    width: f64,
    height: f64,
}
impl Shape for Rectangle {
    fn area(&self) -> f64 {
        self.width * self.height
    }
    fn name(&self) -> &str {
        "rectangle"
    }
}

// (1) Generic + trait bound: monomorphization ตอน compile time — เร็วสุด แต่ต้องรู้ type เดียวต่อการเรียก
fn describe_generic<T: Shape>(shape: &T) -> String {
    format!("{} มีพื้นที่ {:.2}", shape.name(), shape.area())
}

// (2) dyn Trait: dynamic dispatch ผ่าน vtable — ยืดหยุ่นกว่า เก็บ heterogeneous collection ได้
fn describe_dyn(shape: &dyn Shape) -> String {
    format!("{} มีพื้นที่ {:.2}", shape.name(), shape.area())
}

fn main() {
    let circle = Circle { radius: 2.0 };
    let rect = Rectangle { width: 3.0, height: 4.0 };

    println!("{}", describe_generic(&circle));
    println!("{}", describe_generic(&rect));

    // เก็บ Circle และ Rectangle ปนกันใน Vec เดียวได้ เพราะทุกตัวถูกมองเป็น &dyn Shape เหมือนกัน
    let shapes: Vec<Box<dyn Shape>> = vec![
        Box::new(Circle { radius: 1.0 }),
        Box::new(Rectangle { width: 2.0, height: 5.0 }),
    ];
    for s in &shapes {
        println!("{}", describe_dyn(s.as_ref()));
    }

    let total: f64 = shapes.iter().map(|s| s.area()).sum();
    println!("พื้นที่รวม: {:.2}", total);
}
```

ผลลัพธ์:

```
circle มีพื้นที่ 12.57
rectangle มีพื้นที่ 12.00
circle มีพื้นที่ 3.14
rectangle มีพื้นที่ 10.00
พื้นที่รวม: 13.14
```

ผู้สัมภาษณ์ระดับกลางขึ้นไปมักถามคำถามต่อว่า "trait ทุกตัวทำ `dyn Trait` ได้ไหม" — คำตอบคือไม่ เพราะ trait ต้อง
**dyn compatible** (เดิมเรียกว่า object-safe) เช่น ต้องไม่มี associated function ที่ไม่มี `self` parameter (เพราะ
compiler ไม่รู้ว่าจะสร้าง vtable ให้ type ไหน) ตัวอย่าง error ที่แสดงปัญหานี้:

```rust
trait Cloneable {
    fn make_copy() -> Self; // ไม่มี self parameter
}

struct Widget;
impl Cloneable for Widget {
    fn make_copy() -> Self {
        Widget
    }
}

fn main() {
    let items: Vec<Box<dyn Cloneable>> = Vec::new();
    println!("{}", items.len());
}
```

```
error[E0038]: the trait `Cloneable` is not dyn compatible
  --> src/main.rs:14:28
   |
14 |     let items: Vec<Box<dyn Cloneable>> = Vec::new();
   |                            ^^^^^^^^^ `Cloneable` is not dyn compatible
   |
note: for a trait to be dyn compatible it needs to allow building a vtable
  --> src/main.rs:3:8
   |
 2 | trait Cloneable {
   |       --------- this trait is not dyn compatible...
 3 |     fn make_copy() -> Self;
   |        ^^^^^^^^^ ...because associated function `make_copy` has no `self` parameter
```

วิธีแก้คือเพิ่ม `where Self: Sized` ให้ method ที่ไม่ dyn compatible (แล้ว method นั้นจะใช้ผ่าน `dyn Trait`
ไม่ได้ แต่ trait โดยรวมยังทำเป็น `dyn Trait` ได้สำหรับ method อื่น) หรือออกแบบ trait ใหม่ให้ทุก method มี `&self`
ถ้าคุณตอบจุดนี้ได้ แสดงว่าคุณไม่ได้แค่จำ syntax แต่เข้าใจกลไกภายในของ dynamic dispatch จริง ๆ

#### 2.5 Concurrency: `Send` และ `Sync`

> **คำถามตัวอย่าง:** "`Send` กับ `Sync` ต่างกันอย่างไร แล้วทำไม `Rc<RefCell<T>>` ใช้ข้าม thread ไม่ได้ ทั้งที่
> `Arc<Mutex<T>>` ใช้ได้"

**โครงคำตอบที่ดี:** อธิบายทั้งสอง marker trait แยกกันให้ชัด แล้วต่อด้วยเหตุผลเชิง design ของ `Rc`/`Arc` (ตรงกับ
Part 39-40):

- `Send`: type ที่ implement `Send` สามารถ**ย้าย ownership ข้าม thread boundary**ได้อย่างปลอดภัย (ส่งเข้าไปใน
  closure ที่รันบน thread อื่น)
- `Sync`: type ที่ implement `Sync` สามารถให้**หลาย thread เข้าถึงผ่าน `&T` เดียวกันพร้อมกัน**ได้อย่างปลอดภัย
  (นิยามจริงคือ `T` เป็น `Sync` ก็ต่อเมื่อ `&T` เป็น `Send`)
- ทั้งสอง trait เป็น **auto trait** — compiler ใส่ให้อัตโนมัติถ้าทุก field ภายใน type นั้น implement trait นั้นอยู่
  แล้ว (ไม่ต้องเขียน `impl Send for X {}` เอง ยกเว้นกรณี unsafe พิเศษ) และเป็น **marker trait** ที่ไม่มี method
  ใด ๆ — มันแค่เป็น "ป้าย" ที่ compiler ใช้ตรวจสอบตอน compile time
- `Rc<T>` ใช้ reference counter แบบ**ไม่ atomic** (ธรรมดา ไม่มี lock) เพื่อความเร็วสูงสุดในโลก single-thread — ถ้าสอง
  thread เพิ่ม/ลด counter พร้อมกันโดยไม่มี synchronization จะเกิด data race ทันที ดังนั้น `Rc<T>` จึงถูกออกแบบให้
  **ไม่** implement `Send`/`Sync` เลย ไม่ว่า `T` จะเป็นอะไรก็ตาม
- `Arc<T>` (Atomic Rc) ใช้ atomic operation สำหรับ reference counter ซึ่งปลอดภัยข้าม thread จึงเป็น `Send`/`Sync`
  ได้ (ถ้า `T: Send + Sync`) แต่ atomic operation มี overhead เล็กน้อยกว่า counter ธรรมดา — นี่คือ trade-off ที่
  Rust ให้ผู้เขียนเลือกเอง ไม่ได้บังคับใช้ `Arc` เสมอเพราะ `Rc` เร็วกว่าในบริบท single-thread

**ตัวอย่างโค้ดที่ถูกต้อง — `Arc<Mutex<T>>` ข้าม thread:**

```rust
use std::sync::{Arc, Mutex};
use std::thread;

struct Counter {
    value: u64,
}

fn main() {
    let counter = Arc::new(Mutex::new(Counter { value: 0 }));
    let mut handles = Vec::new();

    for _ in 0..8 {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            // compiler บังคับให้ lock() ก่อนแตะข้อมูล จึงไม่มี data race ได้ตั้งแต่ compile time
            let mut guard = counter.lock().unwrap();
            guard.value += 1;
        });
        handles.push(handle);
    }

    for h in handles {
        h.join().unwrap();
    }

    println!("final value = {}", counter.lock().unwrap().value);
}
```

ผลลัพธ์: `final value = 8`

**ตัวอย่าง error ที่ต้องอธิบายได้ — พยายามส่ง `Rc<RefCell<T>>` ข้าม thread:**

```rust
use std::cell::RefCell;
use std::rc::Rc;
use std::thread;

fn main() {
    let shared = Rc::new(RefCell::new(0));
    let shared2 = Rc::clone(&shared);

    let handle = thread::spawn(move || {
        *shared2.borrow_mut() += 1;
    });

    handle.join().unwrap();
    println!("{}", shared.borrow());
}
```

```
error[E0277]: `Rc<RefCell<i32>>` cannot be sent between threads safely
   --> src/main.rs:10:32
    |
 10 |     let handle = thread::spawn(move || {
    |                                ^^^^^^^ `Rc<RefCell<i32>>` cannot be sent between threads safely
    |
    = help: within `{closure@src/main.rs:10:32: 10:39}`, the trait `Send` is not implemented for `Rc<RefCell<i32>>`
note: required by a bound in `spawn`
```

คำตอบที่แข็งแรงที่สุดคือชี้ให้เห็นว่า**นี่คือบั๊ก data race ที่ถูกจับได้ตอน compile time แทนที่จะเป็น runtime crash
หรือ silent corruption ที่ debug ยากมากในภาษาอื่น** — นี่คือคุณค่าหลักของโมเดล ownership เมื่อขยายไปสู่ concurrency
วิธีแก้คือเปลี่ยนเป็น `Arc<Mutex<T>>` ตามตัวอย่างด้านบน

#### 2.6 พื้นฐาน Async

> **คำถามตัวอย่าง:** "`async`/`await` ใน Rust ทำงานอย่างไรจริง ๆ เบื้องหลัง แล้วทำไมถึงต้องมี executor แยกจาก
> language runtime (ไม่เหมือน goroutine ของ Go ที่มี runtime ในตัว)"

**โครงคำตอบที่ดี:** อธิบายว่า `async fn` ไม่ใช่เวทมนตร์ แต่เป็น syntax sugar เหนือ `trait Future` (ตรงกับ Part
46-48):

- `async fn` เมื่อ compile แล้วจะถูกแปลงเป็น state machine ที่ implement `trait Future` — ค่าที่ได้จากการเรียก
  `async fn` ไม่ใช่ผลลัพธ์สุดท้ายทันที แต่เป็น **future ที่ยังไม่ได้ทำงาน** จนกว่าจะถูก `.await` หรือถูก poll
- `Future::poll` คือ method หลักของ trait นี้ — executor เรียก `poll` ซ้ำ ๆ จนกว่าจะได้ `Poll::Ready(value)` ถ้ายัง
  ไม่พร้อมจะได้ `Poll::Pending` และ future ต้องบอก `Waker` ไว้ว่าจะปลุก executor ให้กลับมา poll ใหม่เมื่อไหร่
  (ผ่าน I/O event, timer, หรือ channel) — นี่คือกลไกที่ทำให้ async ไม่ต้องมี thread ต่อ task (ต่างจาก thread แบบ
  Part 37 ที่ 1 thread ต่อ 1 งาน)
- เหตุผลที่ Rust ไม่มี runtime ผูกกับภาษา (ต่างจาก Go ที่มี goroutine scheduler ฝังอยู่ใน runtime): Rust ต้องการให้
  ใช้งานได้ทั้งใน context ที่มี OS (server) และไม่มี OS (embedded, kernel) การผูก async executor เข้ากับ language
  runtime แบบ Go จะทำให้ Rust ใช้ใน embedded ไม่ได้ (ตรงกับปรัชญาที่ Part 102 พูดถึง) จึงแยก `std::future::Future`
  (trait เปล่า ๆ) ออกจาก executor (Tokio, async-std, embassy) ให้เลือกใช้ตามบริบท

**ตัวอย่างโค้ดที่ควรพูดถึง — `async fn` ธรรมดา ต่อด้วย `Future` ที่เขียนมือ เพื่อพิสูจน์ว่าไม่ใช่เวทมนตร์:**

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

async fn fetch_user_name(id: u32) -> String {
    // ของจริงตรงนี้จะเป็น network I/O (เช่น query ผ่าน sqlx ตาม Part 70-71) จำลองด้วย async fn เฉย ๆ
    format!("user-{}", id)
}

async fn greet(id: u32) -> String {
    let name = fetch_user_name(id).await; // ต้อง .await ทุกครั้งเพื่อ "ขับ" future ให้ทำงาน
    format!("hello, {}", name)
}

// Future ที่เขียนมือเอง เพื่ออธิบายว่า async/await คือ syntax sugar เหนือ trait Future + poll
struct ReadyAfter {
    polled_once: bool,
}

impl Future for ReadyAfter {
    type Output = &'static str;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        if self.polled_once {
            Poll::Ready("done on second poll")
        } else {
            self.polled_once = true;
            // บอก executor ว่า "ยังไม่เสร็จ แต่ให้ poll ฉันใหม่อีกครั้งทันที"
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}

#[tokio::main]
async fn main() {
    let msg = greet(42).await;
    println!("{}", msg);

    let custom = ReadyAfter { polled_once: false }.await;
    println!("custom future result: {}", custom);
}
```

ผลลัพธ์:

```
hello, user-42
custom future result: done on second poll
```

คำตอบระดับสูงมักถูกถามต่อว่า "ทำไม future ต้องใช้ `Pin`" — คำตอบสั้น ๆ ที่ควรพูดได้คือ: state machine ที่ compiler
สร้างจาก `async fn` อาจมี field ที่อ้างถึง field อื่นในตัวมันเอง (self-referential struct) การย้าย (move) struct
แบบนี้ในหน่วยความจำจะทำให้ reference ภายในผิดเพี้ยน `Pin<&mut Self>` คือการันตีว่า memory location ของ future นั้น
จะไม่ถูกย้ายอีกหลังจากถูก poll ครั้งแรก — ตรงกับกลไกที่ Part 47 อธิบายไว้อย่างละเอียด

### 3. Live Coding Interview: ประเภทโจทย์ที่พบบ่อยและวิธีแก้

การสัมภาษณ์แบบ live coding สาย Rust มักไม่ถามโจทย์ algorithm ทั่วไปแบบ LeetCode ตรง ๆ (แม้จะมีบ้าง) แต่ชอบถามโจทย์
ที่ทดสอบว่าคุณ "คิดแบบ Rust" หรือยัง คือคิดเรื่อง ownership/borrowing ไปพร้อมกับ logic ไม่ใช่คิด logic อย่างเดียวแล้ว
ค่อยแก้ error ทีหลังแบบสุ่ม โจทย์ที่พบบ่อยแบ่งได้ 3 ประเภทหลัก

#### 3.1 ประเภทที่ 1: Data Structure ที่ต้องคิดเรื่อง Ownership — LRU Cache

โจทย์คลาสสิกที่ถูกถามบ่อยมากคือ "implement LRU (Least Recently Used) Cache" เพราะมันบังคับให้ผู้สมัครตัดสินใจเรื่อง
ownership ของ key อย่างชัดเจน (ใครเป็นเจ้าของ key ตอนไหน, จะ clone เมื่อไหร่, จะย้ายเมื่อไหร่)

**แนวคิด:** ต้องมีทั้ง lookup แบบเร็ว (ใช้ `HashMap`) และลำดับการใช้งานล่าสุด (เพื่อรู้ว่าจะ evict ตัวไหนก่อนเมื่อ
capacity เต็ม) เวอร์ชันที่ตอบในเวลาจำกัดของสัมภาษณ์ควรเลือกความเรียบง่ายมาก่อน optimal ที่สุด แล้วอธิบายว่าจะ
optimize อย่างไรถ้ามีเวลามากกว่านี้ — นี่คือทักษะสำคัญที่ผู้สัมภาษณ์มองหา ไม่ใช่แค่ความถูกต้อง

```rust
use std::collections::HashMap;

struct LruCache<K, V> {
    capacity: usize,
    map: HashMap<K, V>,
    // เก็บลำดับการใช้งานล่าสุด: ท้าย Vec = ใช้ล่าสุด, หัว Vec = เก่าสุด
    order: Vec<K>,
}

impl<K: std::hash::Hash + Eq + Clone, V> LruCache<K, V> {
    fn new(capacity: usize) -> Self {
        assert!(capacity > 0, "capacity ต้องมากกว่า 0");
        LruCache {
            capacity,
            map: HashMap::new(),
            order: Vec::new(),
        }
    }

    fn touch(&mut self, key: &K) {
        if let Some(pos) = self.order.iter().position(|k| k == key) {
            let k = self.order.remove(pos);
            self.order.push(k);
        }
    }

    fn get(&mut self, key: &K) -> Option<&V> {
        if self.map.contains_key(key) {
            self.touch(key);
            self.map.get(key)
        } else {
            None
        }
    }

    fn put(&mut self, key: K, value: V) {
        if self.map.contains_key(&key) {
            self.map.insert(key.clone(), value);
            self.touch(&key);
            return;
        }

        if self.map.len() >= self.capacity {
            // เจ้าของ (ownership) ของ key ที่เก่าที่สุดถูกโอนออกจาก order เข้าไปใน remove()
            if let Some(oldest) = self.order.first().cloned() {
                self.order.remove(0);
                self.map.remove(&oldest);
            }
        }

        self.order.push(key.clone());
        self.map.insert(key, value);
    }

    fn len(&self) -> usize {
        self.map.len()
    }
}

fn main() {
    let mut cache: LruCache<&str, i32> = LruCache::new(2);
    cache.put("a", 1);
    cache.put("b", 2);
    assert_eq!(cache.get(&"a"), Some(&1)); // "a" ถูกใช้ล่าสุด -> "b" กลายเป็นตัวที่เก่าที่สุด
    cache.put("c", 3); // capacity เต็ม -> ต้อง evict "b" ทิ้ง
    assert_eq!(cache.get(&"b"), None);
    assert_eq!(cache.get(&"a"), Some(&1));
    assert_eq!(cache.get(&"c"), Some(&3));
    assert_eq!(cache.len(), 2);
    println!("LRU cache ทำงานถูกต้อง: len = {}", cache.len());
}
```

ผลลัพธ์: `LRU cache ทำงานถูกต้อง: len = 2`

**สิ่งที่ควรพูดออกมาดัง ๆ ระหว่างเขียน (นี่สำคัญกว่าตัวโค้ดเอง):**

- "ผมเลือก `Vec<K>` สำหรับ order ก่อนเพราะเขียนไวและถูกต้อง แต่ `touch`/`put` มี complexity O(n) ต่อครั้งเพราะต้อง
  scan หา position — ถ้ามีเวลาต่อ ผมจะเปลี่ยนไปใช้ intrusive doubly-linked list เก็บ index (หรือใช้ crate อย่าง
  `linked-hash-map`) เพื่อให้ทุก operation เป็น O(1)"
- "ผม `.clone()` key ตอน `put` เพราะ key ต้องอยู่ทั้งใน `order` และ `map` พร้อมกัน — ถ้า `K` เป็น type ที่ clone แพง
  (เช่น struct ใหญ่) ผมจะเปลี่ยนไปใช้ `Rc<K>` แทนเพื่อแบ่งกันอ้างอิงโดยไม่ copy ข้อมูลจริง"

การพูดถึง trade-off เหล่านี้ออกมาดัง ๆ คือสิ่งที่แยกผู้สมัคร "เขียนโค้ดผ่าน" ออกจากผู้สมัคร "เข้าใจ Rust จริง"

#### 3.2 ประเภทที่ 2: เขียน Iterator Adapter เอง

โจทย์อีกแบบที่พบบ่อยคือให้เขียน iterator adapter ของตัวเอง (คล้าย `.step_by()`, `.chunks()`, `.dedup()`) เพื่อทดสอบ
ว่าคุณเข้าใจ `trait Iterator` และ lazy evaluation จริงหรือไม่ (ตรงกับ Part 25-26)

**โจทย์ตัวอย่าง:** "เขียน iterator adapter `every_nth(n)` ที่คืนค่าตัวที่ 1, (1+n), (1+2n), ... จาก iterator ต้นทาง"

```rust
struct EveryNth<I> {
    inner: I,
    n: usize,
    first: bool,
}

impl<I: Iterator> Iterator for EveryNth<I> {
    type Item = I::Item;

    fn next(&mut self) -> Option<Self::Item> {
        if self.first {
            self.first = false;
            return self.inner.next();
        }
        // ข้าม n-1 ตัวก่อนคืนตัวถัดไป
        for _ in 0..self.n - 1 {
            self.inner.next()?;
        }
        self.inner.next()
    }
}

trait EveryNthExt: Iterator + Sized {
    fn every_nth(self, n: usize) -> EveryNth<Self> {
        assert!(n > 0, "n ต้องมากกว่า 0");
        EveryNth { inner: self, n, first: true }
    }
}

impl<I: Iterator> EveryNthExt for I {}

fn main() {
    let nums = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
    let result: Vec<i32> = nums.into_iter().every_nth(3).collect();
    println!("{:?}", result);
    assert_eq!(result, vec![1, 4, 7, 10]);
    println!("custom iterator adapter ทำงานถูกต้อง");
}
```

ผลลัพธ์:

```
[1, 4, 7, 10]
custom iterator adapter ทำงานถูกต้อง
```

**สิ่งที่ควรอธิบายระหว่างเขียน:**

- "ผมทำ `EveryNthExt` เป็น extension trait แล้ว `impl` ให้กับทุก `I: Iterator` เพื่อให้เรียก `.every_nth(3)` ต่อท้าย
  chain แบบเดียวกับ `.map()`/`.filter()` ทั่วไปได้ — นี่คือ pattern มาตรฐานที่ Rust ใช้ทำ custom adapter"
  (`Iterator` trait เองไม่มี default method ตัวนี้ แต่เรา "เพิ่ม" มันเข้าไปได้ผ่าน trait ใหม่ที่มี `Self: Sized`
  bound — สอดคล้องกับที่ Part 26 อธิบายเรื่อง custom iterator)
- "ทุก adapter ใน std library เป็น **lazy** — ไม่มีการคำนวณอะไรจนกว่าจะมีคนเรียก `.next()` (ผ่าน `.collect()`,
  `for` loop, หรืออื่น ๆ) `EveryNth` ของผมก็เหมือนกัน มันไม่ allocate หรือ loop ล่วงหน้าเลย — ข้อดีคือ chain ยาว ๆ
  ของ adapter ไม่ต้องสร้าง intermediate collection ระหว่างทาง (zero-cost abstraction จริง ๆ)"

#### 3.3 ประเภทที่ 3: แก้ Borrow-Checker Error ในโค้ดที่ให้มา

โจทย์ประเภทนี้ผู้สัมภาษณ์จะให้โค้ดที่ compile ไม่ผ่านมาก่อน แล้วให้คุณอ่าน error message และแก้ให้ถูก — วัดทักษะการ
อ่าน compiler error และความเข้าใจ borrow checker โดยตรง โจทย์คลาสสิกที่สุดคือการพยายามแก้ไข collection ระหว่างที่
กำลัง iterate มันอยู่

**โค้ดที่มีบั๊ก (ตามที่ผู้สัมภาษณ์อาจยื่นให้):**

```rust
use std::collections::HashMap;

fn remove_negative_balances(accounts: &mut HashMap<String, i32>) {
    for (name, balance) in accounts.iter() {
        if *balance < 0 {
            accounts.remove(name);
        }
    }
}

fn main() {
    let mut accounts = HashMap::new();
    accounts.insert("alice".to_string(), 100);
    accounts.insert("bob".to_string(), -50);
    remove_negative_balances(&mut accounts);
    println!("{:?}", accounts);
}
```

Error จริงที่ compiler ให้:

```
error[E0502]: cannot borrow `*accounts` as mutable because it is also borrowed as immutable
 --> src/main.rs:7:13
  |
5 |     for (name, balance) in accounts.iter() {
  |                            ---------------
  |                            |
  |                            immutable borrow occurs here
  |                            immutable borrow later used here
6 |         if *balance < 0 {
7 |             accounts.remove(name);
  |             ^^^^^^^^^^^^^^^^^^^^^ mutable borrow occurs here
```

**วิธีอ่าน error นี้ (สิ่งที่ควรพูดออกมา):** `accounts.iter()` ยืม `accounts` แบบ immutable ตลอดช่วง loop ทั้งหมด
เพราะ `name`/`balance` เป็น reference ที่ยืมมาจากมัน — การเรียก `accounts.remove(name)` ข้างในต้องการยืมแบบ mutable
ในขณะที่ immutable borrow (จาก `.iter()`) ยังไม่จบ ซึ่งขัดกับกฎ "ยืมได้หลายคนแบบอ่านอย่างเดียว หรือยืมได้คนเดียวแบบ
เขียน แต่ไม่ปนกัน" — นี่**ไม่ใช่**ข้อจำกัดที่ไม่มีเหตุผล แต่ป้องกันบั๊กจริง (`HashMap::remove` อาจทำให้ bucket
จัดเรียงใหม่ ทำให้ iterator ที่กำลังเดินอยู่ชี้ผิดตำแหน่งได้ — เทียบเท่ากับ `ConcurrentModificationException` ใน Java
ที่ตรวจจับได้แค่ตอน runtime แต่ Rust จับได้ตอน compile time)

**วิธีแก้ (แยก "อ่าน" กับ "เขียน" ออกจากกันเป็นสองขั้นตอน):**

```rust
use std::collections::HashMap;

fn remove_negative_balances(accounts: &mut HashMap<String, i32>) {
    // ขั้นที่ 1: อ่านอย่างเดียว เก็บ key ที่เข้าเงื่อนไขไว้ใน Vec ใหม่
    let to_remove: Vec<String> = accounts
        .iter()
        .filter(|(_, balance)| **balance < 0)
        .map(|(name, _)| name.clone())
        .collect();

    // ขั้นที่ 2: การยืม immutable จาก .iter() จบไปแล้วตั้งแต่ collect() เสร็จ
    // ตอนนี้ borrow mutable เพื่อ remove ได้อย่างปลอดภัย
    for name in to_remove {
        accounts.remove(&name);
    }
}

fn main() {
    let mut accounts = HashMap::new();
    accounts.insert("alice".to_string(), 100);
    accounts.insert("bob".to_string(), -50);
    remove_negative_balances(&mut accounts);
    println!("{:?}", accounts);
}
```

ผลลัพธ์: `{"alice": 100}`

การพูดแนวทางแก้แบบนี้ ("collect key ที่ต้องแก้ไว้ก่อน แล้วค่อย mutate ทีหลัง") เป็น pattern ที่ใช้ซ้ำได้กับโจทย์แบบ
เดียวกันเกือบทุกแบบที่เกี่ยวกับการแก้ไข collection ระหว่าง iterate — คุ้มค่ามากที่จะจดจำเป็น pattern สำเร็จรูป

### 4. System Design Interview แบบ Rust-Specific

โจทย์ system design ใน Rust interview มีความต่างจากโจทย์ system design ทั่วไป (ที่มักถามแค่ "จะใช้ database อะไร,
จะ scale อย่างไร") ผู้สัมภาษณ์สาย Rust มักถามคำถามต่อว่า **"แล้วคุณจะใช้ type system encode invariant ของ domain
นี้อย่างไร"** — เพราะนี่คือจุดขายหลักของภาษา ถ้าคุณตอบแค่ระดับ "ใช้ Postgres กับ Redis" โดยไม่พูดถึง type-level
design เลย จะดูเหมือนไม่ได้ใช้จุดแข็งของ Rust เลย

> **คำถามตัวอย่าง:** "ออกแบบระบบประมวลผลคำสั่งซื้อ (order processing) ที่ต้องรับประกันว่าคำสั่งซื้อจะไม่ถูก 'จัดส่ง'
> ก่อนที่จะ 'ชำระเงิน' สำเร็จ และห้ามหักสต็อกซ้ำสองครั้งสำหรับคำสั่งซื้อเดียวกัน — จะออกแบบอย่างไรใน Rust"

**โครงคำตอบที่ดี แบ่งเป็นชั้น ๆ ตามลำดับที่ผู้สัมภาษณ์คาดหวัง:**

1. **เริ่มจากถามคำถามกลับเพื่อ scope โจทย์** (สัญญาณที่ดีเสมอในทุก system design interview): ปริมาณ order ต่อวินาที
   ประมาณเท่าไหร่ ต้อง support การ cancel/refund ไหม มีกี่ payment provider เป็น monolith หรือ microservices
   (เชื่อมกับ Part 81)
2. **โมเดล state ของ domain เป็น enum/typestate ก่อนพูดถึง infrastructure ใด ๆ** — นี่คือจุดที่ทำให้คำตอบสาย Rust
   ต่างจากคำตอบทั่วไป: ให้ compiler เป็นคนบังคับลำดับ state transition แทนการเช็ค `if order.status == "paid"`
   ด้วย string/enum ธรรมดาที่ยังพลาดเรียก method ผิดลำดับได้ (runtime bug) ใช้ **typestate pattern** (ตามที่ Part 53
   สอน) แทน:

```rust
use std::marker::PhantomData;

struct Created;
struct Paid;
struct Shipped;

struct Order<State> {
    id: u64,
    total_cents: u64,
    _state: PhantomData<State>,
}

impl Order<Created> {
    fn new(id: u64, total_cents: u64) -> Self {
        Order { id, total_cents, _state: PhantomData }
    }

    // เปลี่ยนสถานะได้ทางเดียว: Created -> Paid เท่านั้น
    fn mark_paid(self, amount_received_cents: u64) -> Result<Order<Paid>, String> {
        if amount_received_cents < self.total_cents {
            return Err(format!(
                "ยอดที่รับ ({}) น้อยกว่ายอดสั่งซื้อ ({})",
                amount_received_cents, self.total_cents
            ));
        }
        Ok(Order { id: self.id, total_cents: self.total_cents, _state: PhantomData })
    }
}

impl Order<Paid> {
    // มีแค่ order ที่ "Paid" แล้วเท่านั้นที่มี method นี้ -> ป้องกันการ ship ก่อนจ่ายเงินตั้งแต่ compile time
    fn ship(self, tracking_code: &str) -> Order<Shipped> {
        println!("order #{} shipped with tracking {}", self.id, tracking_code);
        Order { id: self.id, total_cents: self.total_cents, _state: PhantomData }
    }
}

impl Order<Shipped> {
    fn tracking_summary(&self) -> String {
        format!("order #{} (total {} cents) is on its way", self.id, self.total_cents)
    }
}

fn main() {
    let order = Order::<Created>::new(1, 5000);
    let paid = order.mark_paid(5000).expect("จ่ายเงินครบ");
    let shipped = paid.ship("TH123456789");
    println!("{}", shipped.tracking_summary());

    // ตัวอย่างที่ compiler ป้องกันไว้ (ถ้าเปิดใช้จะ compile ไม่ผ่านโดยตั้งใจ):
    // let bad = Order::<Created>::new(2, 100);
    // bad.ship("X"); // ERROR: no method named `ship` found for struct `Order<Created>`
}
```

ผลลัพธ์:

```
order #1 shipped with tracking TH123456789
order #1 (total 5000 cents) is on its way
```

จุดสำคัญที่ต้องพูดออกมาให้ผู้สัมภาษณ์เห็น: `Order<Created>` และ `Order<Shipped>` เป็นคนละ type กันในมุมของ compiler
แม้จะมี field เหมือนกันหมด — method `.ship()` มีอยู่แค่บน `Order<Paid>` เท่านั้น ทำให้ **การเรียก `.ship()` บน
order ที่ยังไม่จ่ายเงินเป็น compile error ไม่ใช่ runtime bug ที่ต้องมาเขียน unit test คอยจับ** — invariant "ห้าม
ship ก่อนจ่าย" ถูกบังคับโดยตัวภาษาเอง ไม่ต้องพึ่งการ code review หรือ QA มาจับ

3. **ต่อด้วยประเด็น concurrency สำหรับปัญหาการหักสต็อกซ้ำ:** อธิบายว่าจะป้องกัน double-decrement อย่างไรในระดับ
   distributed system — เช่น ใช้ idempotency key ต่อ order (เก็บใน Redis หรือ unique constraint ใน Postgres) หรือ
   ถ้าเป็น in-process ก็ใช้ `Arc<Mutex<Inventory>>`/atomic compare-and-swap (เชื่อมกับ Part 39-40, 51) — แล้วพูดถึง
   ว่าที่ระดับ microservice จริง (Part 81) มักต้องมี distributed transaction pattern เช่น saga pattern เพราะ order
   service กับ inventory service อาจเป็นคนละ process/database กัน
4. **ปิดท้ายด้วยเรื่อง observability และ error handling ระดับ production:** พูดถึงว่าทุก state transition ควร
   emit event/log (เชื่อมกับ Part 98-99 เรื่อง metrics/tracing) เพื่อ debug ปัญหา order ที่ค้างอยู่ใน state ผิดปกติ
   ได้ และ error ของแต่ละ transition ควรเป็น typed error (`OrderError` เหมือนใน section 2.3) ไม่ใช่ string ทั่วไป
   เพื่อให้ caller ตัดสินใจ retry/compensate ได้ถูกต้อง

**สิ่งที่แยกคำตอบ "ดี" จาก "ธรรมดา" ในโจทย์ system design สาย Rust:** คำตอบธรรมดาจะข้ามขั้นที่ 2 ไปเลย (พูดแต่เรื่อง
service/database) ส่วนคำตอบที่ดีจะใช้เวลาส่วนใหญ่อยู่กับขั้นที่ 2 ก่อน แล้วค่อยขยายไปเรื่อง infrastructure — เพราะนี่
คือสิ่งที่บอกผู้สัมภาษณ์ว่าคุณคิดแบบ Rust developer จริง ๆ ไม่ใช่ system designer ทั่วไปที่บังเอิญเขียน Rust

### 5. Portfolio และ Resume สำหรับ Rust Developer

Portfolio ของ Rust developer มีมาตรฐานที่ต่างจากภาษาอื่นเล็กน้อย เพราะชุมชน Rust ค่อนข้างเข้มงวดเรื่อง code quality
และมักตรวจสอบ portfolio ด้วยการอ่านโค้ดจริง ไม่ใช่แค่ดู README

**สิ่งที่ส่งสัญญาณความสามารถจริงให้ผู้ตรวจสอบที่รู้ Rust:**

- **การ contribute open source จริง (ตาม Part 105)** — แม้แค่ PR เล็ก ๆ ที่ merge เข้า crate ที่มีคนใช้จริงบน
  crates.io ก็มีค่ามากกว่าโปรเจกต์ personal ที่ไม่มีใครรีวิว เพราะมันแสดงว่าโค้ดของคุณผ่านการตรวจสอบจากคนอื่นแล้ว —
  ผู้ตรวจสอบสามารถไปดู PR history จริงบน GitHub ได้ทันที ไม่ต้องเชื่อคำบอกเล่า
- **โปรเจกต์ส่วนตัวที่มีเอกสารดีและผ่าน checklist การรีวิวแบบที่ Part 109 สอน** — คือมี `cargo clippy` สะอาด, มี
  test coverage ที่สมเหตุสมผล, มี `README` ที่อธิบาย design decision (ไม่ใช่แค่ "วิธีรัน"), และมี commit history ที่
  อ่านแล้วเข้าใจ evolution ของโค้ด (ไม่ใช่ "wip", "fix", "fix2" ติดกัน 50 commit)
- **การเข้าร่วม contest/challenge อย่าง Advent of Code** — เป็นตัวเลือกที่ commitment ต่ำแต่ยังมีประโยชน์ โดยเฉพาะ
  ถ้าคุณเขียน solution ที่สะอาดและมี benchmark เทียบ approach ต่าง ๆ (เช่นเทียบ `HashMap` vs `Vec` สำหรับโจทย์หนึ่ง)
  มันแสดงความเข้าใจเรื่อง performance ได้ในเวลาสั้น ๆ โดยไม่ต้องมีโปรเจกต์ใหญ่

**สิ่งที่ "ดูเหมือน" มีประโยชน์แต่จริง ๆ ไม่ค่อยสร้างความน่าเชื่อถือ (ควรรู้ตัวไว้):**

- โปรเจกต์ CRUD ง่าย ๆ ที่ทำตาม tutorial แบบเป๊ะ ๆ โดยไม่มีการปรับแต่งหรือคิดต่อเอง — ผู้ตรวจสอบที่เคยอ่าน tutorial
  เดียวกันจะจำได้ทันทีว่าคุณ copy โครงสร้างมา
  หมด ไม่มีหลักฐานว่าคุณเข้าใจ design decision ของตัวเอง
- จำนวนบรรทัดโค้ดหรือจำนวนโปรเจกต์ที่มากแต่ทุกอันไม่มี test, ไม่มี error handling ที่จริงจัง (เต็มไปด้วย
  `.unwrap()` ทุกที่) — ปริมาณไม่ชนะคุณภาพในสายตาคนที่รู้ Rust จริง โปรเจกต์เดียวที่ทำเสร็จสมบูรณ์และดูแลดี มีค่ามากกว่า
  โปรเจกต์ห้าอันที่ทำครึ่ง ๆ กลาง ๆ
- badge/certificate จากคอร์สออนไลน์ที่ไม่มีโค้ดจริงให้ดู — ผู้ตรวจสอบสาย engineering ส่วนใหญ่ให้ความสำคัญกับโค้ดที่
  เขียนเองมากกว่า certificate เสมอ

**คำแนะนำเรื่อง resume โดยเฉพาะ:** ใส่ลิงก์ GitHub และเลือกให้ pin repository ที่ดีที่สุด 2-3 อันไว้บนโปรไฟล์
(ไม่ใช่ทุกอันที่เคยทำ) เขียน bullet point ของแต่ละโปรเจกต์ให้บอก **ผลลัพธ์เชิง technical ที่วัดได้** เช่น "ลด latency
p99 จาก X ms เหลือ Y ms ด้วยการเปลี่ยนจาก `Mutex` เป็น `RwLock`" มากกว่าคำอธิบายกว้าง ๆ ว่า "สร้างเว็บแอปด้วย Rust"
— ตัวเลขที่จับต้องได้และภาษาที่เจาะจงทาง technical คือสิ่งที่ทำให้ resume ของคุณต่างจาก resume ของคนอื่นร้อยคนที่เขียน
"proficient in Rust" เหมือนกัน

### 6. Behavioral และ Non-Technical Interview

คำถาม behavioral สาย Rust มีแพทเทิร์นที่ค่อนข้างซ้ำ ๆ เพราะผู้สัมภาษณ์ส่วนใหญ่รู้ว่าคนที่มาสัมภาษณ์ Rust จำนวนมาก
ย้ายมาจากภาษาอื่น คำตอบที่ดีต้องแสดงให้เห็นว่าคุณเรียนรู้ด้วยความเข้าใจ ไม่ใช่แค่ท่องจำ และคุณมองเห็น trade-off
ของภาษา ไม่ใช่คลั่งไคล้แบบไม่มีเหตุผล

> **คำถามตัวอย่าง:** "คุณเรียน Rust มาจากไหน แล้วอะไรที่ยากที่สุดสำหรับคุณตอนเริ่มเรียน"

**โครงคำตอบที่ดี (ตัวอย่างคำตอบที่แสดง self-awareness จริง):**

> "ผมมาจากพื้นฐาน [ภาษาอื่นที่คุณถนัด] ตอนเริ่มเรียน Rust สิ่งที่ยากที่สุดไม่ใช่ syntax แต่คือการเปลี่ยนวิธีคิดเรื่อง
> การส่งผ่านข้อมูล — ใน [ภาษาเดิม] ผมไม่ต้องคิดว่าใครเป็น 'เจ้าของ' ตัวแปรเลยเพราะ garbage collector จัดการให้
> แต่ Rust บังคับให้ผมคิดเรื่องนี้ตั้งแต่ต้น ช่วงแรกผม fight กับ borrow checker บ่อยมาก จนกระทั่งผมเปลี่ยนมุมมองจาก
> 'ทำไม compiler ไม่ยอมให้ผมทำแบบนี้' เป็น 'compiler กำลังบอกอะไรผมเกี่ยวกับ ownership ของข้อมูลนี้' — พอเปลี่ยนมุม
> คิดแบบนี้ ผมเริ่มเขียนโค้ดที่ compile ผ่านตั้งแต่รอบแรกมากขึ้นเรื่อย ๆ เพราะผมออกแบบ ownership ไว้ในหัวก่อนเขียน
> โค้ดจริง ไม่ใช่เขียนแล้วค่อยแก้ error ทีละอัน"

คำตอบแบบนี้ดีเพราะ (1) เจาะจงและจริงใจ ไม่ใช่คำตอบท่องจำทั่วไป (2) แสดง self-reflection ที่แท้จริง (3) จบด้วยผลลัพธ์
เชิงบวกที่เป็นรูปธรรม (เขียนโค้ด compile ผ่านมากขึ้น) ไม่ใช่แค่บ่นปัญหา

> **คำถามตัวอย่าง:** "เล่าโปรเจกต์ล่าสุดที่คุณใช้ Rust แล้วเจอ trade-off อะไรบ้างที่ต้องตัดสินใจ"

**ตัวอย่างคำตอบที่แสดงความเข้าใจ trade-off แบบที่หลักสูตรนี้สอนมาตลอด (ไม่ใช่แค่ท่องว่า Rust "ดี"):**

> "ตอนเลือก web framework สำหรับโปรเจกต์นั้น ผมเทียบระหว่าง Axum กับ Actix-web (แบบที่เรียนใน Part 69) สุดท้ายเลือก
> Axum เพราะทีมคุ้นกับ ecosystem ของ Tower/Tokio อยู่แล้ว และ Axum ใช้ extractor pattern ที่ทำให้ handler อ่านง่ายกว่า
> สำหรับทีมที่ยังไม่คุ้น async runtime มาก แต่ผมก็ยอมรับกับทีมตรง ๆ ว่า Actix-web มี benchmark ที่เร็วกว่าในบาง
> workload — เราเลือก 'อ่านง่ายและ maintain ได้ในทีม' มากกว่า 'เร็วที่สุดในกระดาษ' เพราะ traffic จริงของระบบเราไม่ได้
> อยู่ในระดับที่ต่างกันของสอง framework นี้จะมีผลกระทบทางธุรกิจจริง — อีกจุดที่ต้องตัดสินใจคือเรื่อง REST vs GraphQL
> (ตาม Part 78-79) เราเลือก REST เพราะ client ของเรามีแค่ mobile app เดียวที่ควบคุม schema เองได้ ถ้ามี client
> หลายแบบที่ควบคุมไม่ได้ (เช่น third-party integration) ผมคงเอียงไปทาง GraphQL มากกว่านี้เพื่อลด over-fetching"

คำตอบนี้แสดงสามอย่างพร้อมกัน: (1) รู้จัก option มากกว่าหนึ่งตัวและเหตุผลของแต่ละตัว (2) ตัดสินใจโดยอิงบริบทจริงของ
ทีม/ธุรกิจ ไม่ใช่ "เพราะมันเจ๋งกว่า" (3) ยอมรับข้อเสียของสิ่งที่เลือกอย่างตรงไปตรงมา — สามอย่างนี้คือสิ่งที่ผู้
สัมภาษณ์ senior มองหาจริง ๆ

> **คำถามตัวอย่าง:** "มีสถานการณ์ไหนที่คุณจะ**ไม่**เลือก Rust สำหรับโปรเจกต์ใหม่ไหม"

คำถามนี้เป็นกับดักที่ท้าทายมาก เพราะผู้สมัครจำนวนมากตอบว่า "ไม่มี Rust ดีที่สุดเสมอ" ซึ่งเป็นคำตอบที่แสดงว่าไม่เข้าใจ
trade-off เลย คำตอบที่ดีควรอิงกับสิ่งที่หลักสูตรนี้เน้นย้ำมาตลอด (Part 69 เรื่องเลือก framework, Part 106 เรื่อง DI
trade-off): "ถ้าทีมต้อง ship MVP ภายในสัปดาห์เดียวเพื่อทดสอบตลาด และทีมไม่มีใครถนัด Rust มาก่อน ผมจะไม่เลือก Rust
เพราะ learning curve จะทำให้ time-to-market ช้าลงมาก ผมจะเลือกภาษาที่ทีม productive อยู่แล้ว แล้วค่อยพิจารณา rewrite
เฉพาะส่วนที่เป็น bottleneck จริงด้วย Rust ทีหลังถ้า MVP พิสูจน์ตัวเองแล้วว่าคุ้มจะลงทุนต่อ" — คำตอบแบบนี้แสดงว่าคุณ
มองภาษาเป็นเครื่องมือ ไม่ใช่ศาสนา

### 7. Take-Home Assignment: ทำอย่างไรให้ได้ผลดีที่สุดในเวลาจำกัด

Take-home assignment สาย Rust ที่พบบ่อยมีสองรูปแบบหลัก ตรงกับสองโปรเจกต์ capstone ที่หลักสูตรนี้สอนไปแล้ว:

1. **CLI tool ขนาดเล็ก** (ตรงกับแนวทางที่ Part 107 สอนไว้ในระดับ capstone เต็มรูปแบบ) เช่น "เขียนโปรแกรม parse
   log file แล้วสรุปสถิติ" หรือ "เขียน tool จัดการ todo list ผ่าน command line"
2. **Web API ขนาดเล็ก** (ตรงกับแนวทางที่ Part 108 สอนไว้ในระดับ capstone เต็มรูปแบบ) เช่น "สร้าง REST API สำหรับจัดการ
   รายการสินค้า พร้อม CRUD พื้นฐาน"

**หลักการจัดสรรเวลาที่สำคัญที่สุด (และเป็นจุดที่ผู้สมัครส่วนใหญ่พลาด):** take-home มักมีเวลาจำกัด (2-6 ชั่วโมงตามที่
ระบุ) ผู้สมัครจำนวนมากเสียเวลาไปกับสองเรื่องที่ "ดูดี" แต่ไม่คุ้มค่ากับ scope ของ assignment:

- **Over-engineering ตั้งแต่ต้น** — สร้าง abstraction layer, trait, dependency injection framework (แบบที่ Part
  106 สอน) สำหรับโปรแกรมที่มีแค่ 3 endpoint ทำให้เวลาส่วนใหญ่หมดไปกับโครงสร้างที่ requirement จริงไม่ต้องการ ผู้
  ตรวจ assignment ระดับ senior จะมองว่านี่คือสัญญาณลบ (ไม่รู้จัก scope) มากกว่าสัญญาณบวก (รู้ pattern เยอะ)
- **พยายาม cover edge case ทุกแบบที่คิดได้** — ใช้เวลาหลายชั่วโมงจัดการ edge case ที่ requirement ไม่ได้ขอ (เช่น
  Unicode normalization ที่ซับซ้อนสำหรับ CLI ที่แค่ต้อง parse log ภาษาอังกฤษ) แทนที่จะทำ requirement หลักให้เสร็จ
  สมบูรณ์และสะอาดก่อน

**สิ่งที่ควร priority ก่อนตามลำดับ (ใช้ได้กับทั้งสองรูปแบบ):**

1. Requirement ที่ระบุไว้ตรง ๆ ต้องถูกต้อง 100% ก่อนอื่นใด — อ่านโจทย์สองรอบ ทำ checklist ของทุกข้อที่ระบุไว้ชัดเจน
   แล้วเช็คให้ครบก่อนจะเริ่มคิดถึงสิ่งที่โจทย์ไม่ได้ขอ
2. โครงสร้างโค้ดที่สะอาดในระดับที่เหมาะกับขนาดงาน (right-sized) — แยก concern พื้นฐาน (เช่น parsing, business
   logic, I/O) ออกจากกันด้วย module/function ที่ชื่อสื่อความหมาย ตาม checklist การรีวิวที่ Part 109 สอน แต่ไม่ต้อง
   ถึงขั้นมี trait abstraction สำหรับทุกอย่างที่ "อาจจะ" ต้องเปลี่ยนในอนาคตที่ไม่มีในโจทย์
3. Error handling ที่สมเหตุสมผล — ใช้ `Result` กับจุดที่ error คาดเดาได้ (ตามหลักการใน section 2.3) ไม่ต้อง cover
   ทุก edge case ที่เป็นไปได้ในทางทฤษฎี แต่ต้องไม่มี `.unwrap()` ที่ทำให้โปรแกรม panic กับ input ปกติที่สมควรจัดการได้
4. Test อย่างน้อยสำหรับ business logic หลัก — ไม่ต้อง 100% coverage แต่ต้องมี test ที่พิสูจน์ว่า requirement หลัก
   ทำงานถูกต้อง เพราะผู้ตรวจ assignment มักรันเทสต์ก่อนอ่านโค้ดเสียอีก
5. README สั้น ๆ ที่บอกวิธีรันและ assumption ที่คุณตั้งไว้เอง (ถ้าโจทย์กำกวมบางจุด) — การเขียน assumption ออกมา
   ตรง ๆ ("ผมสมมติว่า X เพราะโจทย์ไม่ได้ระบุ") ดีกว่าการเดาแบบเงียบ ๆ แล้วหวังว่าผู้ตรวจจะเดาทางเดียวกับคุณ

สรุปสั้น ๆ ของหลักการนี้: **"production sensibility ที่ right-sized"** ไม่ได้แปลว่าทำน้อยที่สุดที่พอผ่าน แต่แปลว่า
เลือกใช้ความพยายามให้ตรงกับสิ่งที่ requirement จริงต้องการ — เหมือนกับที่ capstone ทั้งสองใน Part 107 และ 108
แสดงให้เห็นว่า production-grade ไม่ได้หมายถึงซับซ้อนที่สุดเสมอไป แต่หมายถึง**เหมาะสมกับ scope ของปัญหาที่แก้จริง**

### 8. เงินเดือนและ Leveling: กรอบคิดที่ควรรู้ (ไม่ใช่ตัวเลขตายตัว)

บทนี้จะไม่ระบุตัวเลขเงินเดือนเฉพาะเจาะจง เพราะตัวเลขที่เขียนไว้ตอนนี้จะล้าสมัยเร็วมาก (เงินเฟ้อ, ความต้องการตลาด,
และภูมิภาคที่ต่างกันมากทำให้ตัวเลขเดียวไม่มีความหมาย) — สิ่งที่มีค่ากว่าคือ**กรอบคิด**ที่ใช้ค้นคว้าตัวเลขที่ทันสมัย
และตรงกับตลาดของคุณเองได้ตลอดไป

**กรอบคิดที่ควรใช้หาข้อมูลเงินเดือนของตัวเอง:**

- ใช้แหล่งข้อมูลที่อัปเดตสม่ำเสมอและเจาะจงตามภูมิภาค เช่น survey เงินเดือนของ Stack Overflow ประจำปี (มีหมวด
  "ภาษาที่ได้เงินเดือนสูง" แยกตามภูมิภาค), เว็บรายงานเงินเดือนที่พนักงานกรอกเอง (levels.fyi สำหรับบริษัทเทคขนาดใหญ่,
  Glassdoor/PayScale สำหรับภาพรวมกว้างกว่า), และถ้าเป็นไปได้ให้คุยกับคนในสายงานเดียวกันในตลาดของคุณตรง ๆ (ข้อมูลจาก
  คนจริงในตลาดจริงมักแม่นกว่า aggregate report เสมอ)
- เทียบเงินเดือนตาม **role/level ที่ตรงกัน** ไม่ใช่เทียบตาม "ภาษาที่ใช้" — เพราะ Rust developer ที่ทำงาน backend
  service ควรถูกเทียบกับ backend engineer คนอื่นที่ level เดียวกัน (ไม่ว่าเขาเขียน Go, Java, หรือ Python) ไม่ใช่
  เทียบกับ "Rust developer" เป็น category แยกที่มี premium พิเศษของตัวเอง
- **ข้อสังเกตที่สำคัญและตรงกับความจริงในตลาดปัจจุบัน:** งาน Rust ส่วนใหญ่ (ยกเว้นบางบริษัท infra/blockchain ที่
  หายากและแข่งขันสูงมาก) มักถูก level และจ่ายเงินเดือนในกรอบเดียวกับตำแหน่ง backend/infrastructure engineer ทั่วไป
  ที่ seniority เดียวกัน **ไม่ใช่มี premium พิเศษเพราะ "รู้ Rust" เพียงอย่างเดียว** — สิ่งที่ทำให้เงินเดือนสูงขึ้น
  จริง ๆ คือ seniority, ความรับผิดชอบของตำแหน่ง (impact, scope), และความหายากของ**การผสมทักษะ Rust + โดเมนเฉพาะทาง**
  (เช่น Rust + blockchain, Rust + embedded ที่มี domain knowledge เฉพาะทางประกอบ) มากกว่าการรู้ Rust เพียว ๆ

**การเจรจาต่อรอง (negotiation) ในระดับตระหนักรู้:**

- อย่าให้ตัวเลขก่อนถ้าเลี่ยงได้ — ถามกลับเรื่อง range ของตำแหน่งก่อนเสมอ
- มีตัวเลขในใจที่มาจากการค้นคว้าจริง (ตามกรอบคิดข้างบน) ไม่ใช่ตัวเลขที่คิดขึ้นเอง
- offer ที่ดีไม่ใช่แค่ฐานเงินเดือน — ให้พิจารณา equity/bonus, remote flexibility, และที่สำคัญมากสำหรับสายเทคนิค
  คือคุณภาพของทีมและโอกาสเรียนรู้ (โดยเฉพาะถ้าเป็นงาน Rust แรกของคุณ การได้ทำงานกับทีมที่มี senior Rust developer
  จริง ๆ คอย mentor มีค่ามากกว่าตัวเงินอีกนิดหนึ่งในระยะยาว)
- ยินดีบอกตรง ๆ ว่าต้องการเวลาคิด ไม่มีความจำเป็นต้องตอบรับ offer ทันทีในสายโทรศัพท์

### 9. เส้นทางอาชีพหลังจากงานแรก

งานแรกในสาย Rust เป็นแค่จุดเริ่มต้น ความเชี่ยวชาญจริง ๆ จะสร้างขึ้นมาในหลายปีของการทำงาน ไม่ใช่ทันทีที่จบหลักสูตรนี้
— สิ่งที่หลักสูตรนี้ให้คุณคือ**พื้นฐานที่กว้างและมั่นคงพอที่จะเลือกเส้นทางความเชี่ยวชาญได้เอง** ต่อไปนี้คือเส้นทางที่
พบบ่อยซึ่งหลักสูตรนี้ได้แตะไว้แล้วในระดับพื้นฐานถึงกลาง:

- **Backend/Web Engineering** (Module 4-5) — เส้นทางที่มีตำแหน่งงานมากที่สุดในบรรดาเส้นทางที่เกี่ยวกับ Rust ทั้งหมด
  ความเชี่ยวชาญเชิงลึกในเส้นทางนี้ต่อจากที่เรียนไปคือเรื่อง distributed systems, database internals, และ system
  design ระดับใหญ่ (traffic สูง, multi-region)
- **Infrastructure/Platform/DevOps Engineering** (Module 6 ทั้งโมดูล) — เส้นทางที่ต่อเนื่องโดยตรงจากสิ่งที่เรียนใน
  บทก่อน ๆ ของโมดูลนี้ (Docker, CI/CD, Observability, Security, Cloud deployment) ความเชี่ยวชาญเชิงลึกต่อจากนี้คือ
  เรื่อง distributed systems ระดับ infrastructure จริง (consensus algorithm, storage engine, network protocol)
- **Embedded/IoT** (Part 102) — เส้นทางที่ต้องมีความรู้ hardware ควบคู่กับ Rust ความเชี่ยวชาญต่อจากนี้ต้องลงลึกเรื่อง
  real-time constraint, hardware-specific driver, และมาตรฐานความปลอดภัยของอุตสาหกรรม (เช่น automotive, medical
  device)
- **Blockchain/Crypto** (Part 104) — เส้นทางที่มี demand เฉพาะทางสูงแต่แข่งขันสูงเช่นกัน ความเชี่ยวชาญต่อจากนี้ต้อง
  ลงลึกเรื่อง cryptography, consensus protocol, และ security audit ของ smart contract ซึ่งเป็นสายที่ผิดพลาดมี
  ค่าใช้จ่ายสูงมาก (เงินคนอื่นอยู่ในนั้นจริง ๆ) จึงต้องการความรอบคอบระดับสูงกว่าสายอื่น
- **Compiler/Language Tooling** — เส้นทางขั้นสูงที่หลักสูตรนี้**ไม่ได้สอน**แต่ควรรู้จักไว้ว่ามีอยู่: การทำงานกับ
  Rust compiler เอง (rustc, rust-analyzer), เขียน linter/formatter, หรือทำงานด้าน proc macro/language tooling
  ขั้นสูง เส้นทางนี้ต้องการความเข้าใจ compiler theory, type theory, และการอ่าน source code ของ rustc ซึ่งเป็นทักษะ
  ที่ต้องสร้างเพิ่มเติมจากที่หลักสูตรนี้ให้ไว้อย่างมาก (Part 44-45 เรื่อง proc macro เป็นแค่จุดเริ่มต้นที่ผิวเผิน
  ของโลกนี้เท่านั้น) — เป็นเส้นทางที่คนน้อยมากเดินและใช้เวลาสร้างความเชี่ยวชาญนานที่สุดในบรรดาเส้นทางทั้งหมด

ไม่ว่าจะเลือกเส้นทางไหน สิ่งที่เหมือนกันคือ**ทักษะ Rust พื้นฐานที่แน่นจากหลักสูตรนี้ (ownership, trait system, error
handling, concurrency) เป็นฐานที่ใช้ได้ในทุกเส้นทาง** ส่วนที่ต่างกันคือ domain knowledge เฉพาะทางที่ต้องสร้างเพิ่ม
เอง — และนั่นคือสิ่งที่ใช้เวลาเป็นปีจริง ๆ ไม่ใช่สิ่งที่คอร์สไหนในโลกสอนให้ครบได้ในไม่กี่เดือน

### 10. ทบทวนภาพรวมหลักสูตรทั้งหมด: จาก "เขียน for loop ได้" สู่ "ออกแบบระบบ production ได้"

ก่อนจะปิดหลักสูตรนี้ เรามาย้อนดูเส้นทางทั้งหมดที่คุณเดินผ่านมาตลอด 110 บท เพราะการเห็นภาพรวมนี้ชัด ๆ จะช่วยให้คุณ
พูดถึงตัวเองในสัมภาษณ์ได้อย่างมีโครงสร้าง และช่วยให้คุณรู้ว่าตอนนี้คุณ "ยืนอยู่จุดไหน" ของเส้นทางการเป็น Rust
developer

**Module 0 (Part 1-5): เริ่มต้นใช้งาน Rust** — จุดเริ่มต้นที่ทุกคนต้องผ่าน: ติดตั้ง `rustup`, เข้าใจ `cargo`
เป็นทั้ง build tool และ package manager ในตัวเดียว (ต่างจากภาษาอื่นที่มักแยก build tool กับ package manager),
เขียนตัวแปร ฟังก์ชัน และ control flow พื้นฐาน และเรียนรู้ว่า `rustfmt`/`clippy` ไม่ใช่แค่เครื่องมือเสริม แต่เป็นส่วน
หนึ่งของวัฒนธรรมการเขียนโค้ด Rust ตั้งแต่วันแรก — จุดนี้คือ "เขียน for loop ได้"

**Module 1 (Part 6-20): พื้นฐานภาษา Rust** — นี่คือจุดเปลี่ยนที่สำคัญที่สุดของทั้งหลักสูตร เพราะ Part 6-7
(Ownership/Borrowing) คือสิ่งที่ทำให้ Rust ไม่เหมือนภาษาอื่นที่คุณอาจเคยรู้จัก จากนั้นหลักสูตรค่อย ๆ ขยายไปสู่ slices,
structs, enums/pattern matching, `Option`/`Result` (การจัดการ "ไม่มีค่า" และ "ผิดพลาด" แบบไม่มี null pointer
exception), collections หลัก (`Vec`, `String`, `HashMap`), modules/crates สำหรับจัดระเบียบโค้ดขนาดใหญ่ขึ้น, และปิด
ท้ายด้วย generics/traits/lifetimes เบื้องต้น — จุดนี้คือ "เข้าใจว่าทำไม Rust ถึงปลอดภัยโดยไม่มี garbage collector"

**Module 2 (Part 21-40): ระดับกลาง** — โมดูลนี้ขยายทุกอย่างที่เรียนใน Module 1 ให้ลึกขึ้น: traits ขั้นสูง (trait
object ที่เราใช้ตอบคำถามสัมภาษณ์ในบทนี้), closures และ iterators (ที่ทำให้เราเขียน custom iterator adapter ได้ใน
section 3.2), smart pointers (`Box`, `Rc`, `RefCell`, `Weak`) ที่ขยายโมเดล ownership ให้ยืดหยุ่นขึ้นสำหรับกรณีที่
ownership เดียวไม่พอ, error handling ขั้นสูงที่เราใช้ตอบคำถามสัมภาษณ์ใน section 2.3, testing/documentation ที่เป็น
วินัยสำคัญของงานมืออาชีพ, macro เบื้องต้น, และปิดท้ายด้วย concurrency พื้นฐาน (threads, channels, `Mutex`/`Arc`,
`Send`/`Sync`) ที่เราใช้ตอบคำถามสัมภาษณ์ใน section 2.5 — จุดนี้คือ "เขียนโปรแกรมที่ซับซ้อนได้อย่างถูกต้องและปลอดภัย
จาก race condition"

**Module 3 (Part 41-60): ระดับสูง** — โมดูลนี้พาเราลงลึกไปถึงจุดที่ Rust ต่างจากภาษาระดับสูงอื่นอย่างชัดเจนที่สุด:
unsafe Rust และ raw pointers (เข้าใจว่า "ปลอดภัยโดย default" ไม่ได้แปลว่า "ทำสิ่งที่ต่ำระดับไม่ได้เลย"), FFI สำหรับ
เชื่อมกับ C, procedural macros ขั้นสูง, และที่สำคัญมากคือ async/await, futures, และ Tokio (Part 46-48) ที่เราใช้ตอบ
คำถามสัมภาษณ์ใน section 2.6 — ต่อด้วย atomics/lock-free programming, design patterns แบบ idiomatic Rust,
performance optimization/profiling, serialization ด้วย serde, CLI applications, และ logging/tracing — จุดนี้คือ
"เข้าใจว่า Rust ให้ control ระดับ C ได้เมื่อจำเป็น พร้อมเครื่องมือระดับมืออาชีพสำหรับวัดและปรับ performance จริง"

**Module 4 (Part 61-85): การพัฒนาเว็บแอปพลิเคชัน** — โมดูลที่ใหญ่ที่สุดของหลักสูตร สะท้อนว่านี่คือตลาดงาน Rust ที่
กว้างที่สุดในปัจจุบันตามที่พูดถึงใน section 1 เริ่มจาก HTTP fundamentals, สอง web framework หลัก (Axum, Actix-web)
พร้อมการเปรียบเทียบอย่างตรงไปตรงมาใน Part 69 ที่เราอ้างถึงใน section 6, database integration (SQLx, Diesel,
SeaORM), authentication/authorization ที่จริงจัง (JWT, OAuth2, RBAC), และปิดท้ายด้วยหัวข้อระดับ architecture:
REST API design (Part 78), GraphQL (Part 79), gRPC, microservices (Part 81), message queues, caching, background
jobs, และ API documentation — จุดนี้คือ "สร้างและออกแบบ backend service ที่ผู้ใช้จริงพึ่งพาได้"

**Module 5 (Part 86-95): Full-Stack และ WebAssembly** — โมดูลที่ขยายขอบเขตของ Rust ออกไปนอก backend: WebAssembly
พื้นฐาน, การเชื่อมกับ JavaScript ผ่าน wasm-bindgen, สาม frontend framework (Yew, Leptos, Dioxus), server-side
rendering, และปิดท้ายด้วยโปรเจกต์ full-stack แบบครบวงจร 3 ส่วน (backend, frontend, integration/deployment) พร้อม
testing web application แบบครบวงจร — จุดนี้คือ "เห็นว่า Rust ไม่ได้จำกัดอยู่แค่ backend หรือ systems programming
เท่านั้น"

**Module 6 (Part 96-110): Production, DevOps และระดับมืออาชีพ** — โมดูลปิดท้ายที่เปลี่ยนโฟกัสจาก "เขียนโค้ดให้ทำงาน
ถูก" ไปเป็น "ทำให้ระบบอยู่รอดได้จริงในโลกจริง": containerization ด้วย Docker, CI/CD pipeline, observability
สองแบบ (metrics ด้วย Prometheus, distributed tracing ด้วย OpenTelemetry), security best practices, cloud
deployment, แล้วขยายไปสู่โดเมนเฉพาะทาง (embedded, game dev ด้วย Bevy, blockchain), ปิดท้ายด้วยทักษะที่ทำให้คุณทำงาน
ร่วมกับคนอื่นได้จริง (open source contribution, enterprise design patterns, สองโปรเจกต์ capstone ระดับ production,
code review/refactoring/clean code) และบทนี้ที่แปลทุกอย่างให้เป็นทักษะหางาน — จุดนี้คือ "ออกแบบ, รักษาความปลอดภัย,
สังเกตการณ์ (observe), และ deploy ระบบ Rust ระดับ production ได้ด้วยตัวเอง"

**เส้นทางที่คุณเดินผ่านมา สรุปเป็นภาพเดียว:** จาก "เขียน `for i in 0..10` เป็น" (Part 4) ไปสู่ "อธิบายได้ว่าทำไม
compiler ป้องกัน data race ได้ตั้งแต่ compile time" (Part 40) ไปสู่ "เข้าใจว่า `async`/`await` ไม่ใช่เวทมนตร์แต่เป็น
state machine ที่ implement `Future`" (Part 47-48) ไปสู่ "ออกแบบ REST API และ microservices ที่รับ traffic จริงได้"
(Part 78, 81) ไปสู่ "ทำให้ระบบนั้น deploy ได้จริง, สังเกตการณ์ได้จริง, และปลอดภัยจริง" (Part 96-101) และสุดท้ายไปสู่
"แปลทุกอย่างนี้ให้เป็นทักษะที่ตอบคำถามสัมภาษณ์และวางแผนอาชีพได้" (บทนี้) — นี่คือเส้นทางที่ตาม README.md ของ
หลักสูตรนี้ตั้งเป้าไว้ตั้งแต่บทแรก: จาก "ไม่มีพื้นฐานเลย" สู่ "มืออาชีพและระดับโลก (World-Class Professional)"

**ข้อความปิดท้ายถึงผู้อ่าน:** การจบหลักสูตร 110 บทนี้ไม่ได้แปลว่าคุณ "รู้ Rust หมดแล้ว" — ไม่มีใครรู้ภาษาโปรแกรมภาษา
ไหนหมดจริง ๆ แม้แต่คนที่เขียนภาษานั้นมาสิบปี แต่มันแปลว่าคุณมีพื้นฐานที่ครบและลึกพอที่จะ**เรียนรู้ต่อได้เองอย่างมี
ประสิทธิภาพ** ซึ่งคือทักษะที่สำคัญกว่าความรู้เฉพาะจุดใด ๆ ทรัพยากรที่ควรติดตามต่อจากนี้:

- **The Rust Programming Language (The Book)** ที่ doc.rust-lang.org — เอกสารทางการที่ควรกลับไปอ่านซ้ำเป็นระยะ
  เพราะมันอัปเดตตามภาษาที่เปลี่ยนแปลง และมีรายละเอียดปลีกย่อยที่คอร์สนี้ไม่ได้ครอบคลุมทั้งหมด
- **The Rust Reference** และ **The Rustonomicon** (สำหรับผู้ที่จะลงลึกเรื่อง unsafe Rust ต่อจาก Part 41-43)
- **users.rust-lang.org** (Rust users forum) — ชุมชนที่ตอบคำถามเชิงเทคนิคอย่างละเอียดและเป็นมิตรกับมือใหม่มาก
- **r/rust** บน Reddit — ชุมชนที่แชร์ข่าวสาร โปรเจกต์ใหม่ ๆ และการอภิปรายเชิงลึกเกี่ยวกับทิศทางของภาษา
- **This Week in Rust** (newsletter รายสัปดาห์) — สรุปข่าวสาร RFC ใหม่, crate ที่น่าสนใจ, และบทความจากชุมชนทุก
  สัปดาห์ เป็นวิธีที่มีประสิทธิภาพที่สุดในการติดตามความเคลื่อนไหวของ ecosystem โดยไม่ต้องไล่อ่านทุกที่เอง

ขอแสดงความยินดีที่คุณมาถึงบทสุดท้ายของหลักสูตรที่ครอบคลุมตั้งแต่พื้นฐานที่สุดไปจนถึงหัวข้อระดับมืออาชีพและระดับโลก
อย่างแท้จริง — ตั้งแต่ตัวแปรตัวแรก ไปจนถึงการออกแบบระบบ distributed, การจัดการ concurrency ที่ปลอดภัย, การ deploy
สู่ production, และตอนนี้คือการนำทุกอย่างนั้นไปใช้เริ่มต้นและเติบโตในอาชีพจริง สิ่งที่เหลืออยู่ตอนนี้ไม่ใช่การเรียน
เพิ่มจากคอร์สอีกคอร์ส แต่คือการเอาความรู้นี้ไปลงมือทำจริง ผิดจริง แก้จริง และเติบโตต่อจากที่นี่

## กับดักที่พบบ่อย (Common Pitfalls)

การเตรียมตัวสัมภาษณ์มีกับดักทางความคิดที่พบบ่อยพอ ๆ กับกับดักทาง syntax — ต่อไปนี้คือกับดักที่ตรวจสอบได้จริงจาก
พฤติกรรมและคำตอบที่พบบ่อยในผู้สมัครจริง

- **กับดักที่ 1: ท่องนิยาม ownership/borrowing ได้ แต่อธิบาย "ทำไม" ไม่ได้เมื่อถูกถามต่อ** ผู้สมัครจำนวนมากท่องได้ว่า
  "ownership คือกฎที่ทำให้ค่าหนึ่งมีเจ้าของได้คนเดียว" แต่พอถูกถามต่อว่า "แล้วทำไมภาษาที่มี garbage collector ถึงไม่
  ต้องมีกฎแบบนี้" กลับตอบไม่ได้ วิธีตรวจสอบตัวเองก่อนสัมภาษณ์: ลองอธิบาย ownership ให้เพื่อนที่ไม่รู้ Rust ฟังโดยไม่ใช้
  คำว่า "ownership" หรือ "borrow" เลยแม้แต่คำเดียว ถ้าอธิบายไม่ได้ แสดงว่ายังเข้าใจแค่ระดับคำศัพท์ ไม่ใช่ระดับแนวคิด
  จริง — วิธีแก้คือย้อนกลับไปอ่าน Part 6-7 อีกครั้งโดยเน้นส่วน "ทำไม" มากกว่าส่วน syntax

- **กับดักที่ 2: เข้าใจผิดว่า `unwrap()` ทุกจุดคือสัญญาณของโค้ดแย่ ทั้งที่บางจุดใช้ได้จริง** ผู้สมัครที่เพิ่งเรียนรู้
  เรื่อง error handling มักแก้ไขเกินจำเป็นโดยเปลี่ยน `.unwrap()` ทุกจุดเป็น `Result` แม้ในจุดที่เป็น invariant ที่
  พิสูจน์ได้ว่าไม่ผิดพลาด (เช่น `"42".parse::<i32>().unwrap()` ในโค้ดตัวอย่างที่ literal ถูกกำหนดไว้ตายตัว) การทำ
  แบบนี้ทำให้โค้ดยาวขึ้นโดยไม่ได้อะไรเพิ่ม และแสดงว่ายังไม่เข้าใจเส้นแบ่งระหว่าง recoverable error กับ bug ตามที่
  section 2.3 อธิบายไว้ ให้ฝึกถามตัวเองทุกครั้งก่อนใส่ `Result`: "ถ้าค่านี้เป็น `None`/`Err` จริง ๆ นั่นคือ input
  ผิดปกติที่คาดเดาได้ หรือคือ bug ที่ไม่ควรเกิดขึ้นถ้าโค้ดถูกต้อง"

- **กับดักที่ 3: เลือก `dyn Trait` หรือ generic แบบสุ่มโดยไม่มีเหตุผล แล้วตอบสัมภาษณ์แบบเดา** เมื่อถูกถามว่า "ทำไม
  เลือกแบบนี้" ผู้สมัครจำนวนมากตอบว่า "ผมชินกับแบบนี้" หรือเงียบไป ทั้งที่ในโค้ดจริงเลือกโดยไม่ได้คิด trade-off เลย
  วิธีตรวจสอบตัวเอง: เปิดโปรเจกต์เก่าของตัวเอง หาจุดที่ใช้ `dyn Trait` หรือ `Box<dyn Trait>` ทุกจุด แล้วถามตัวเองว่า
  "ถ้าเปลี่ยนเป็น generic ตรงนี้ได้ไหม แล้วทำไมตอนนั้นเลือกแบบนี้" ถ้าตอบไม่ได้ แสดงว่าตอนเขียนโค้ดไม่ได้ตัดสินใจ
  จริง ๆ ให้กลับไปอ่าน section 2.4 และ Part 21-22 อีกครั้ง

- **กับดักที่ 4: มองว่าคำถาม "ทำไมไม่เลือก Rust" เป็นคำถามหลอกที่ต้องปกป้องภาษา** ตามที่พูดถึงใน section 6 ผู้สมัคร
  จำนวนมากตอบคำถามนี้แบบป้องกันตัว ("Rust ดีที่สุดในทุกกรณี") ซึ่งเป็นสัญญาณลบที่ชัดเจนสำหรับผู้สัมภาษณ์ระดับ senior
  เพราะแสดงว่าไม่เข้าใจ trade-off เชิงธุรกิจ วิธีเตรียมตัว: เขียนรายการสถานการณ์จริงอย่างน้อย 2-3 อย่างที่คุณจะไม่
  เลือก Rust ไว้ล่วงหน้า (เช่น MVP เร่งด่วน, ทีมที่ไม่มีใครถนัด, script one-off ที่รันครั้งเดียว) พร้อมเหตุผลที่อิง
  บริบทจริง ไม่ใช่แค่ "เพราะมันยาก"

- **กับดักที่ 5: เตรียมเฉพาะคำตอบเชิง trivia แต่ไม่เคยลองแก้ borrow-checker error สดในเวลาจำกัด** ผู้สมัครจำนวนมาก
  อ่านคำอธิบาย error ในหนังสือ/บทความจนคุ้น แต่พอเจอ error จริงในสถานการณ์ live coding (ที่มีความเครียดจากเวลาจำกัด
  และมีคนดูอยู่) กลับ freeze เพราะไม่เคยฝึกอ่าน error message แบบ "เปิด terminal จริง อ่านจริง แก้จริง" มาก่อน วิธี
  แก้: จำลองสถานการณ์เจอ error จริงด้วยตัวเองบ่อย ๆ ก่อนสัมภาษณ์ (เขียนโค้ดที่รู้ว่าจะ error แล้วอ่าน error message
  เต็มโดยไม่เปิดเฉลยก่อน) แบบเดียวกับตัวอย่างใน section 3.3

- **กับดักที่ 6: คิดว่าจำนวนโปรเจกต์ใน portfolio สำคัญกว่าคุณภาพ** ตามที่พูดถึงใน section 5 ผู้สมัครจำนวนมากพยายาม
  เพิ่มจำนวนโปรเจกต์ใน GitHub ให้มากที่สุดก่อนสัมภาษณ์ โดยลืมว่าผู้ตรวจสอบที่รู้ Rust จริงจะเปิดอ่านโค้ดจริง ไม่ใช่
  แค่นับจำนวน repository — โปรเจกต์ที่มี `.unwrap()` เกลื่อนและไม่มี test แม้จะมีสิบอัน มีค่าน้อยกว่าโปรเจกต์เดียวที่
  ผ่าน clippy สะอาดและมีเอกสารดี ให้เลือก pin แค่ 2-3 โปรเจกต์ที่ดีที่สุดจริง ๆ

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เลือกหัวข้อสัมภาษณ์เชิงเทคนิคหนึ่งข้อจาก section 2 (ownership, string, error handling, trait
   objects, concurrency, หรือ async) แล้วเขียนคำตอบของตัวเองใหม่ทั้งหมดโดยไม่ดูตัวอย่างในบทนี้ — ต้องมีทั้งคำอธิบาย
   แนวคิดและโค้ดตัวอย่างที่คุณเขียนเอง (ไม่ต้อง copy) จากนั้นลองพูดคำตอบนั้นออกเสียงให้ตัวเองฟังโดยจับเวลาไม่เกิน 3
   นาที — *Hint:* ถ้าพูดไม่ทันหรือติดขัดกลางคำตอบ แสดงว่ายังไม่เข้าใจลึกพอ ให้กลับไปอ่าน section ที่เกี่ยวข้องอีกครั้ง
   ก่อนลองใหม่

2. **(กลาง)** ดัดแปลงโจทย์ live coding ประเภทที่ 3 ใน section 3.3 (แก้ borrow-checker error) ให้เป็นสถานการณ์ใหม่:
   เขียนฟังก์ชันที่พยายาม modify `Vec<T>` ระหว่าง iterate ด้วย `for x in &vec` แล้วพยายาม `.push()` เข้า `vec`
   ตัวเดิมข้างใน loop ให้บันทึก error message จริงที่ได้จาก compiler แล้วเขียนวิธีแก้อย่างน้อย 2 วิธีที่ต่างกัน
   (เช่น collect ค่าที่จะ push ไว้ก่อนแล้ว extend ทีหลัง กับใช้ index-based loop พร้อมจอง capacity ไว้ก่อน) —
   *Hint:* error ที่คาดว่าจะเจอคือ E0502 เหมือนกับตัวอย่างใน section 3.3 แต่บริบทต่างกัน (mutable borrow ชนกับ
   immutable borrow ของ `Vec` เดียวกัน)

3. **(ยาก)** เลือกหนึ่งในสองแบบต่อไปนี้แล้วทำให้เสร็จภายในเวลา 3 ชั่วโมงตามหลักการ "production sensibility แบบ
   right-sized" ใน section 7: (ก) CLI tool ที่อ่านไฟล์ CSV ของรายการสั่งซื้อแล้วสรุปยอดขายรวมต่อสินค้า พร้อม error
   handling ที่เหมาะสมสำหรับไฟล์ที่ format ผิด หรือ (ข) REST API ขนาดเล็กด้วย Axum ที่มี endpoint CRUD สำหรับ "โน้ต"
   (note) พร้อม in-memory storage และ test อย่างน้อย 3 เคสสำหรับ business logic หลัก — เมื่อเสร็จแล้ว ให้เขียน
   README สั้น ๆ ที่บอก assumption ที่คุณตั้งไว้ระหว่างทำ — *Hint:* จับเวลาจริงและหยุดเมื่อครบ 3 ชั่วโมงไม่ว่าจะเสร็จ
   หรือไม่ เพื่อฝึกการจัดสรร priority ภายใต้เวลาจำกัดจริง ๆ ไม่ใช่แค่ทำจนพอใจ

4. **(ยาก/ประยุกต์ใช้งานจริง)** จำลองการสัมภาษณ์ system design แบบใน section 4 ด้วยโจทย์ใหม่: "ออกแบบระบบจองที่นั่ง
   คอนเสิร์ต (concert seat booking) ที่ต้องรับประกันว่าที่นั่งเดียวกันจะไม่ถูกจองซ้ำสองครั้งพร้อมกัน แม้จะมีผู้ใช้
   หลายพันคนพยายามจองพร้อมกันในนาทีเดียว" ให้เขียนคำตอบเป็นเอกสารสั้น ๆ ที่ครอบคลุม (1) การโมเดล state ของที่นั่งด้วย
   type system แบบ typestate หรือ enum ที่เหมาะสม (2) กลยุทธ์ป้องกันการจองซ้ำในระดับ concurrency/distributed system
   (เชื่อมกับ Part 39-40, 51, 81) และ (3) วิธี observe ระบบนี้ใน production (เชื่อมกับ Part 98-99) — *Hint:*
   คำตอบที่ดีควรใช้เวลาส่วนใหญ่กับข้อ (1) ก่อนเหมือนตัวอย่างใน section 4 ไม่ใช่กระโดดไปพูดเรื่อง infrastructure
   ทันที และให้ลองเขียนโค้ด Rust ประกอบข้อ (1) จริง ๆ ให้ compile ผ่านเพื่อพิสูจน์ว่า design ที่เสนอทำได้จริง
   ไม่ใช่แค่พูดในนามธรรม

## สรุป

บทนี้ไม่ได้สอนหัวข้อ Rust ใหม่ แต่เป็นบทที่ทำหน้าที่ "แปล" ทุกอย่างที่คุณเรียนมาตลอด 109 บทก่อนหน้าให้กลายเป็นทักษะ
ที่ใช้หางานและเติบโตในอาชีพได้จริง เราเริ่มจากการตั้งความคาดหวังตลาดงาน Rust ให้ตรงกับความจริง (เล็กกว่า mainstream
แต่เติบโตจริงในโดเมนที่ชัดเจน) ต่อด้วยการฝึกตอบคำถามสัมภาษณ์เชิงเทคนิคหกหัวข้อหลักด้วยคำตอบจริงที่มีโค้ดประกอบและ
ผ่านการ compile ตรวจสอบแล้วทุกตัว ฝึกแก้โจทย์ live coding สามประเภทที่พบบ่อยที่สุด ฝึกพูดคุยโจทย์ system design ใน
สไตล์ที่เน้นการใช้ type system encode invariant ซึ่งเป็นจุดขายหลักของ Rust วางแนวทาง portfolio/resume/behavioral
interview ที่แสดงความเข้าใจ trade-off จริงมากกว่าการท่องจำ ให้กรอบคิดเรื่อง take-home assignment, salary/leveling,
และเส้นทางอาชีพระยะยาว และปิดท้ายด้วยการทบทวนภาพรวมของทั้งหลักสูตรตั้งแต่ Module 0 ถึง Module 6

ถ้าคุณอ่านมาถึงบรรทัดนี้ คุณได้เดินทางผ่านหลักสูตรที่ครอบคลุมตั้งแต่การติดตั้ง Rust ครั้งแรกไปจนถึงการเตรียมตัวเข้าสู่
อาชีพ Rust developer อย่างมืออาชีพแล้ว นี่คือจุดสิ้นสุดของหลักสูตร แต่เป็นจุดเริ่มต้นของการเรียนรู้และเติบโตต่อไป
ในสายอาชีพจริง — ขอให้โชคดีกับการสัมภาษณ์และเส้นทางอาชีพที่กำลังจะเริ่มต้น

หลักสูตรนี้จบลงที่ Part 110 นี้แล้วอย่างสมบูรณ์ ครอบคลุมทั้ง 6 โมดูลตามที่ตั้งเป้าไว้ตั้งแต่ต้นใน README.md — จาก
ผู้ที่ไม่มีพื้นฐานเลย ไปสู่ผู้ที่มีความรู้ครบทั้งด้านภาษา, ระบบ, เว็บ, full-stack, และการทำงานระดับมืออาชีพในโลกจริง
ขอแสดงความยินดีอย่างจริงใจที่คุณเรียนจบหลักสูตรที่มีความครบถ้วนและลึกในระดับที่ไม่ต่างจากหลักสูตรระดับโลกจริง ๆ

---

**Part ก่อนหน้า:** [Code Review, Refactoring และ Clean Code ใน Rust](part-109-code-review-refactoring.md)
