# Part 100: Security Best Practices ใน Rust

> โมดูล: Production, DevOps และระดับมืออาชีพ | ระดับ: มืออาชีพ | เวลาโดยประมาณ: 280 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้อย่างแม่นยำว่า memory safety ที่ Rust ให้มา**ป้องกันอะไรจริง ๆ** (buffer overflow, use-after-free,
  double-free, data race ใน safe code) **และไม่ป้องกันอะไร** (logic bug, SQL injection, broken
  authentication, บั๊กใน `unsafe` code, ปัญหา supply-chain) — แก้ความเข้าใจผิดที่พบบ่อยที่สุดข้อหนึ่งในวงการ
  Rust ที่ว่า "Rust ปลอดภัย" แปลว่าโปรแกรมที่เขียนด้วย Rust ปลอดภัยจากทุกอย่างโดยอัตโนมัติ
- ระบุจุดที่ `unsafe` block เปิดช่องให้เกิดบั๊กด้านความปลอดภัยได้จริง พิสูจน์ด้วยโค้ดที่ compile ผ่านแต่มี
  out-of-bounds read จริงผ่าน raw pointer arithmetic และวัดปริมาณ `unsafe` code ทั้ง dependency tree ของ
  โปรเจกต์จริงด้วย `cargo geiger`
- ตรวจสอบ dependency ของโปรเจกต์หา known vulnerability ด้วย `cargo audit` (RustSec Advisory Database) และ
  บังคับ policy ระดับองค์กร (license compliance, banned crate, duplicate version) ด้วย `cargo deny` — พร้อม
  ผนวกทั้งสองเป็น CI step ที่รันทุก pull request
- แยกแยะและป้องกัน injection สามคลาส: SQL injection (ทวนจาก Part 70), command injection (ใหม่ — เมื่อโปรแกรม
  เรียก `std::process::Command`), และ path traversal (ใหม่ — เมื่อโปรแกรมเสิร์ฟไฟล์ตาม path จาก request)
  พร้อมพิสูจน์ทั้งเวอร์ชันที่มีช่องโหว่และเวอร์ชันที่ปลอดภัยด้วยโค้ดที่รันได้จริง
- เก็บ secret ในหน่วยความจำอย่างถูกวิธีด้วย `secrecy`/`zeroize` (ป้องกันไม่ให้ secret รั่วผ่าน `Debug`/log
  โดยไม่ตั้งใจ) ตั้งค่า TLS ด้วย `rustls` ให้ Axum server จริง และเพิ่ม security header
  (`Content-Security-Policy`, `X-Content-Type-Options`, `Strict-Transport-Security`, `X-Frame-Options`)
  ผ่าน middleware
- นำ checklist ความปลอดภัยที่รวมทุกหัวข้อของบทนี้ไปตรวจสอบระบบ capstone จริงจาก Part 92-94 อย่างเป็นระบบ —
  ยืนยันว่าอะไรมีมาตรการป้องกันอยู่แล้ว (พร้อมอ้างอิงว่ามาจากบทไหน) และแก้ช่องโหว่จริงที่พบระหว่างตรวจ

## ความรู้ที่ต้องมีมาก่อน

บทนี้เป็น**บทสังเคราะห์ (synthesis chapter)** — Part 100 จาก 110 บทของหลักสูตร มันไม่ได้สอนเทคนิคใหม่แบบแยก
ส่วนอย่างเดียว แต่รวบรวมเนื้อหาด้านความปลอดภัยที่กระจายอยู่ทั่วทั้งหลักสูตรเข้าเป็นภาพเดียว แล้วเพิ่มเนื้อหาใหม่ที่
ยังไม่ได้สอนมาก่อนเข้าไปในจุดที่ยังขาด ดังนั้นรายการความรู้ที่ต้องมีมาก่อนจึงยาวกว่าปกติมาก:

- **Part 6 (Ownership เบื้องต้น)** และ **Part 7 (Borrowing และ References)**: กลไก ownership/borrow checker ที่
  เป็นฐานของทุกการันตีด้าน memory safety ที่หัวข้อ 100.1 จะนำมาสรุปให้แม่นยำอีกครั้ง
- **Part 39 (Mutex, Arc และ Shared-State Concurrency)** และ **Part 40 (Send, Sync และความปลอดภัยของ
  Concurrency)**: เหตุผลที่ safe Rust ป้องกัน data race ได้ตอน compile time ผ่าน trait `Send`/`Sync`
- **Part 41 (Unsafe Rust เบื้องต้น)**, **Part 42 (Raw Pointers และ Memory Layout)**, **Part 43 (FFI: การเชื่อมต่อ
  กับ C)**: ห้าสิ่งที่ `unsafe` ปลดล็อก, raw pointer arithmetic, และขอบเขตของ Undefined Behavior — หัวข้อ 100.2
  จะไม่สอนกลไกพื้นฐานซ้ำ แต่มองผ่านเลนส์ความปลอดภัยแทน
- **Part 35 (Cargo ขั้นสูง)**: `cargo install`, `cargo tree`, และ `cargo audit` เบื้องต้น — บทนั้นแนะนำ
  `cargo audit` ไว้แล้วและบอกไว้ตรง ๆ ว่า "จะพูดถึง security practice แบบเจาะจงและ workflow การจัดการช่องโหว่
  แบบเต็มรูปแบบใน Part 100" ซึ่งคือบทนี้
- **Part 65 (Axum: Middleware (tower, tower-http))**: `tower::Layer`, `RequestBodyLimitLayer`,
  `TimeoutLayer`, `CorsLayer` — หัวข้อ 100.11/100.12/100.13 ต่อยอดกลไก middleware นี้ตรง ๆ
- **Part 70 (SQLx: การเชื่อมต่อ PostgreSQL)**: bind parameter และการพิสูจน์ SQL injection ด้วยโค้ดจริง — หัวข้อ
  100.4 ทวนสั้น ๆ แล้วขยายไปที่ command injection/path traversal ที่เป็นปัญหาแบบเดียวกันในบริบทต่างกัน
- **Part 74 (JWT Authentication)**, **Part 75 (Session และ OAuth2)**, **Part 76 (Authorization และ RBAC)**:
  algorithm confusion, password hashing ด้วย argon2, CSRF/session fixation/OAuth2 state parameter, RBAC —
  หัวข้อ 100.6 รวมเป็น checklist อ้างอิงกลับไปทั้งสามบทโดยไม่อธิบายกลไกซ้ำ
- **Part 78 (RESTful API Design)** และ **Part 83 (Caching ด้วย Redis)**: rate limiting design/implementation —
  หัวข้อ 100.8 อ้างอิงกลับไปในฐานะมาตรการป้องกัน denial-of-service ที่ทำไว้แล้ว
- **Part 92-94 (Full-Stack Project ทั้ง 3 บท)**: ระบบห้องสมุด/ยืม-คืนหนังสือที่เป็น capstone เต็มรูปแบบ — หัวข้อ
  100.14 นำระบบนี้มาตรวจสอบความปลอดภัยทั้งระบบเป็นบทสรุปปิดท้าย

## เนื้อหา

### 100.1 ขอบเขตของ "Memory Safety": มันคือความปลอดภัยแค่ส่วนหนึ่ง ไม่ใช่ทั้งหมด

ประโยค "Rust ปลอดภัย" ถูกพูดถึงบ่อยมากจนกลายเป็นสโลแกน แต่ประโยคนี้กำกวมและนำไปสู่ความเข้าใจผิดที่พบบ่อยมากใน
ทีมที่เพิ่งย้ายมาใช้ Rust: การคิดว่าเลือกใช้ Rust แล้วโปรแกรมจะ "ปลอดภัย" โดยอัตโนมัติในทุกความหมาย ก่อนจะไปลึก
กว่านี้ ต้องแยกคำสองคำที่มักถูกใช้แทนกันแต่ความหมายไม่เหมือนกัน:

- **Memory safety** คือคุณสมบัติที่จำกัดเฉพาะการเข้าถึงหน่วยความจำ: โปรแกรมจะไม่มีทางอ่าน/เขียนหน่วยความจำที่
  "ไม่ควรเข้าถึง" ตามกฎของภาษา (นอกขอบเขตของ allocation, หลังจากถูกปล่อยไปแล้ว, ก่อนถูก initialize)
- **Security** (ความปลอดภัยของระบบ) คือคุณสมบัติที่กว้างกว่ามาก ครอบคลุมทุกอย่างที่ทำให้ระบบทำงานตาม
  "เจตนาที่ตั้งใจ" แม้จะมี input ที่เป็นปฏิปักษ์ (adversarial input) เข้ามา — authentication ที่ถูกต้อง, authorization
  ที่ถูกต้อง, ข้อมูลที่ validate แล้ว, dependency ที่ไม่มีช่องโหว่ที่รู้จัก, การเข้ารหัสข้อมูลระหว่างทาง ฯลฯ

**Memory safety เป็นสับเซ็ตหนึ่งของ security** — เป็นสับเซ็ตที่สำคัญมากเพราะบั๊ก memory safety (buffer
overflow, use-after-free) คือต้นเหตุของช่องโหว่ความปลอดภัยระดับร้ายแรงจำนวนมากในซอฟต์แวร์ที่เขียนด้วย C/C++
มาหลายสิบปี (Microsoft และ Google รายงานตรงกันว่าราว 70% ของช่องโหว่ที่แก้ไปในผลิตภัณฑ์ของตนมีต้นเหตุจาก
memory safety) แต่มันเป็นแค่ "สับเซ็ตหนึ่ง" — ไม่ใช่ทั้งหมด

#### สิ่งที่ Rust's Ownership Model การันตีให้จริง ๆ (ทวนจาก Part 6/7/39/40 อย่างแม่นยำ)

Part 6 อธิบายไว้ว่าค่าทุกตัวมี**เจ้าของ (owner)** เดียว และเมื่อ owner ออกจาก scope ค่าจะถูก drop ทันที — Part 7
เพิ่ม reference (`&T`/`&mut T`) ที่ borrow checker ตรวจสอบตอน compile time ว่า **"immutable reference หลายตัว
พร้อมกันได้ หรือ mutable reference ได้แค่ตัวเดียว ไม่ผสมกันทั้งสองแบบพร้อมกัน"** สองกฎนี้ (ownership เดียว +
borrow rule) ทำงานร่วมกันแล้วการันตีสี่อย่างนี้ **สำหรับ safe Rust code (ไม่มี `unsafe` เลย)**:

| การันตี | กลไกที่ทำให้เกิดขึ้น | เทียบกับ C/C++ |
|---|---|---|
| ไม่มี **buffer overflow** (อ่าน/เขียนเกินขอบเขตของ array/slice) | ทุกการเข้าถึง index ผ่าน `[]` มีการตรวจ bound รันไทม์ (panic ถ้าเกิน), slice เก็บ length ไว้เสมอ | `array[i]` ไม่มีการตรวจ bound ใด ๆ — เขียนเกินขอบเขตได้เงียบ ๆ |
| ไม่มี **use-after-free** (ใช้ค่าที่ถูก drop/free ไปแล้ว) | borrow checker ปฏิเสธไม่ให้ reference มีชีวิตอยู่นานกว่าค่าที่มันชี้ไป (lifetime) | pointer ที่เหลืออยู่หลัง `free()` ไม่มีอะไรห้ามใช้ต่อ |
| ไม่มี **double-free** (ปล่อย memory ก้อนเดียวกันสองครั้ง) | มีแค่ owner เดียวที่รับผิดชอบ drop เสมอ, move semantics ทำให้ owner เดิมใช้ค่าต่อไม่ได้หลัง move | เรียก `free()` สองครั้งกับ pointer เดิมได้ ไม่มีอะไรห้าม |
| ไม่มี **data race** ใน safe code (สอง thread เข้าถึงหน่วยความจำเดียวกันพร้อมกัน อย่างน้อยหนึ่งฝั่งเขียน โดยไม่มี synchronization) | trait `Send`/`Sync` (Part 40) ที่ compiler ตรวจสอบเอง ปฏิเสธการส่งค่าที่ไม่ `Send` ข้าม thread และปฏิเสธ shared mutable state ที่ไม่ผ่าน `Mutex`/`Arc` (Part 39) | ไม่มีการตรวจสอบใด ๆ ตอน compile — ต้องพึ่ง discipline ของโปรแกรมเมอร์ล้วน ๆ |

ข้อสังเกตสำคัญ: ทั้งสี่แถวนี้คือ**การันตีที่ compiler พิสูจน์ให้ทั้งหมดก่อนโปรแกรมจะแม้แต่ compile ผ่าน** — ไม่ใช่
สิ่งที่ต้องไปรัน test แล้วหวังว่าจะจับได้ นี่คือความหมายที่แม่นยำของคำว่า "Rust ปลอดภัยด้าน memory" และเป็น
เหตุผลที่ภาษานี้ถูกเลือกใช้ในโปรเจกต์ระดับ critical infrastructure เพิ่มขึ้นเรื่อย ๆ (Linux kernel, Android,
Windows components, Cloudflare edge)

#### สิ่งที่ Rust ไม่ได้การันตีให้เลย — และทำไมถึงสำคัญที่ต้องพูดตรง ๆ

รายการต่อไปนี้คือสิ่งที่ **ownership/borrow checker ไม่มีส่วนเกี่ยวข้องเลย** เพราะมันอยู่นอกขอบเขตของ "การเข้าถึง
หน่วยความจำถูกต้องหรือไม่" โดยสิ้นเชิง — บทนี้ทั้งบทที่เหลือคือการอุดช่องว่างเหล่านี้ด้วยเครื่องมือ/แนวทางที่ถูกต้อง:

1. **Logic bug** — โค้ด compile ผ่าน, ไม่มีปัญหา memory เลย, แต่ตรรกะทางธุรกิจผิด (เช่น เช็คสิทธิ์ผิดเงื่อนไข,
   คำนวณราคาผิด) borrow checker ไม่มีทางรู้ว่า "ตรรกะที่ถูกต้อง" ควรเป็นอย่างไร เพราะมันไม่ใช่สิ่งที่ type
   system เข้าใจ
2. **SQL injection** — string ที่ประกอบเป็น SQL query ก็เป็นค่า `String` ที่ ownership/borrow ถูกต้องสมบูรณ์
   แบบ 100% แต่ความหมายของมันตอนถูกส่งไปตีความเป็น SQL นั้นผิดพลาดได้เต็มที่ (Part 70 พิสูจน์ไว้แล้วด้วยโค้ดจริง
   — ทวนในหัวข้อ 100.4)
3. **Broken authentication/authorization** — ระบบเช็ค JWT algorithm ผิด, ลืมเช็ค `403`, มี CSRF ช่องโหว่ — ทั้งหมด
   นี้คือโค้ดที่ compile ผ่านสมบูรณ์และไม่มีบั๊ก memory เลยแม้แต่จุดเดียว (Part 74-76 ทวนใน 100.6)
4. **บั๊ก memory safety ภายใน `unsafe` code** — นี่คือจุดที่คนเข้าใจผิดบ่อยที่สุด: การันตีทั้งสี่ข้อในตารางข้างบน
   มีเงื่อนไข **"สำหรับ safe Rust code เท่านั้น"** — ทันทีที่มี `unsafe` block แม้แต่บล็อกเดียว โปรแกรมเมอร์
   กำลังบอก compiler ว่า "ฉันรับผิดชอบพิสูจน์ความถูกต้องเอง แทนที่จะให้ compiler พิสูจน์ให้" (หัวข้อ 100.2
   พิสูจน์ด้วยโค้ดจริงว่าบั๊กแบบนี้เกิดขึ้นได้จริงและ compile ผ่านสนิท)
5. **Supply-chain risk** — ถึง ownership model ของโค้ดที่คุณเขียนเองจะสมบูรณ์แบบทุกบรรทัด แต่ dependency ที่
   ดึงมาจาก crates.io อาจมีช่องโหว่ที่ประกาศไว้แล้ว (RustSec advisory) หรือแม้แต่เป็น crate ที่ตั้งใจแฝง
   โค้ดอันตราย (หัวข้อ 100.3)

ข้อสรุปของหัวข้อนี้ที่ต้องจำไว้ตลอดบทที่เหลือ: **"เขียนด้วย Rust" ไม่ใช่ security checklist ที่ทำให้เสร็จในตัว
เอง** มันคือฐานที่แข็งแกร่งกว่าภาษาที่ไม่มี ownership model มาก (ตัดปัญหาทั้งคลาสของ memory corruption ออกไป
โดยอัตโนมัติ) แต่ยังเหลืองานอีกมากที่ต้องทำต่อด้วยความรอบคอบเหมือนภาษาอื่น ๆ — บทนี้คือแผนที่ของงานที่เหลือนั้น

#### เปรียบเทียบกับภาษาอื่น: ตำแหน่งของ Rust ในสเปกตรัมของ Memory Safety

การจัดวางภาษาต่าง ๆ ตามแนวทางจัดการหน่วยความจำ (ที่ Part 6 หัวข้อ 6.1 เกริ่นไว้แล้วว่ามีสามแนวทางหลัก) ช่วยให้
เห็นภาพว่า Rust อยู่ตรงไหนในสเปกตรัมนี้ และทำไมมันถึงได้ทั้ง memory safety **และ** performance ระดับเดียวกับ
C/C++ พร้อมกัน ซึ่งเป็นสิ่งที่ก่อนหน้านี้ถูกมองว่าต้องเลือกอย่างใดอย่างหนึ่งเท่านั้น:

| ภาษา | แนวทางจัดการหน่วยความจำ | Memory Safety | ราคาที่ต้องจ่าย |
|---|---|---|---|
| C/C++ | Manual (`malloc`/`free`, `new`/`delete`) | ไม่การันตีเลย — ทุกบรรทัดมีความเสี่ยง use-after-free/buffer overflow เท่ากันหมด | ไม่มี runtime overhead แต่ผู้เขียนโค้ดรับภาระตรวจสอบเองทั้งหมด 100% |
| Java/C#/Python/Go | Garbage Collection | Memory safe (ไม่มี use-after-free/double-free เพราะ GC ไม่ปล่อย memory จนกว่าจะไม่มีใครอ้างถึงแล้ว) | GC pause ที่คาดเดาเวลาไม่ได้แน่นอน, memory overhead จากการเก็บ metadata ของ GC, ไม่มีการควบคุม timing ของการปล่อย memory ที่ละเอียด |
| Rust | Ownership + Borrow Checker (ตรวจตอน compile time) | Memory safe เทียบเท่า GC สำหรับ safe code (ตารางหัวข้อก่อนหน้า) **บวก** ป้องกัน data race ที่ GC ภาษาอื่นไม่ได้ป้องกันให้ | ต้องเรียนรู้ ownership/lifetime (เส้นทางการเรียนรู้ที่บทนี้ก่อนหน้าทั้งหมดปูมาให้) แต่ **ไม่มี runtime overhead เลย** เพราะทุกอย่างตรวจสอบเสร็จตอน compile |

แถวที่สำคัญที่สุดของตารางนี้คือแถว Rust: มันคือภาษาแรกที่ใช้กันแพร่หลายจริงในระดับ production ที่ได้ทั้ง memory
safety (แถวเดียวกับ Java/Go) **และ** ไม่มี runtime overhead (แถวเดียวกับ C/C++) พร้อมกัน — เหตุผลที่ทำได้คือ
การตรวจสอบทั้งหมดเกิดขึ้น**ตอน compile time** (ผ่านการวิเคราะห์ lifetime/ownership ที่ Part 6/7 สอนไว้) ไม่ใช่
ตอน runtime แบบ garbage collector ที่ต้องมี background process คอยตรวจสอบและปล่อย memory ระหว่างโปรแกรมรันอยู่
จริง — นี่คือเหตุผลเชิงเทคนิคที่แท้จริงเบื้องหลังคำว่า "zero-cost abstraction" ที่มักถูกพูดถึงคู่กับ Rust

### 100.2 `unsafe` และภาระที่ผู้เขียนโค้ดรับไว้เอง: มุมมองด้านความปลอดภัย

Part 41 อธิบายไว้แล้วว่า `unsafe` block ปลดล็อกได้แค่ห้าอย่าง (dereference raw pointer, เรียก `unsafe fn`,
อ่าน/เขียน `static mut`, implement `unsafe trait`, เข้าถึง field ของ `union`) และ **ไม่ได้ปิดกฎของ Rust ข้อ
อื่นเลย** (borrow checker, move semantics ยังทำงานเหมือนเดิมทุกประการ) — สิ่งที่หัวข้อนี้เพิ่มเข้ามาคือการมอง
ทั้งห้าอย่างนี้ผ่าน**เลนส์ความปลอดภัยโดยตรง**: ทุกจุดที่มี `unsafe` คือจุดที่การันตีในตารางหัวข้อ 100.1 **ไม่มี
ผลบังคับอีกต่อไป** — compiler เชื่อคำสัญญาที่โปรแกรมเมอร์ให้ไว้โดยไม่ตรวจสอบซ้ำ ถ้าคำสัญญานั้นผิด ผลลัพธ์คือ
Undefined Behavior ที่ Part 41.7 อธิบายไว้ว่า "ไม่พังทันทีเสมอไป" — และนั่นคือเหตุผลที่มันอันตรายกว่า error ที่
crash ทันที เพราะบั๊กจะไม่ถูกจับได้ในการทดสอบทั่วไป

#### พิสูจน์ด้วยโค้ดจริง: Out-of-Bounds Read ผ่าน Raw Pointer Arithmetic ที่ Compile ผ่านสมบูรณ์

โค้ดต่อไปนี้จำลองสถานการณ์ที่พบได้จริง: ฟังก์ชันอ่านค่าจาก buffer ของ sensor reading โดยใช้ raw pointer
arithmetic (`.add()`) แทนการเข้าถึงผ่าน index ปกติ (`buf[index]`) — เหตุผลที่คนทำแบบนี้จริงในโค้ด production
มักเป็นเรื่อง performance (เลี่ยง bound check ซ้ำในลูปที่ hot path) แต่ถ้าทำโดยไม่ตรวจสอบ bound เองก่อน ผลลัพธ์
คือช่องโหว่ที่ compiler ไม่มีทางจับได้เลย:

```rust
struct SensorReading {
    id: u32,
    celsius: f32,
}

fn read_at_offset(buf: &[SensorReading], index: usize) -> (u32, f32) {
    // เขียนโดยตั้งใจให้ "ดูเหมือน" ปลอดภัย: ใช้ .as_ptr() แล้วบวก offset เอง
    // ไม่มีการเช็ค `index < buf.len()` เลย -- ต่างจาก buf[index] ที่ Rust จะ panic ให้ทันที
    let ptr = buf.as_ptr();
    unsafe {
        // *** บั๊กจริง: ไม่ตรวจ bound ก่อนใช้ .add() ***
        let elem_ptr = ptr.add(index);
        let reading = &*elem_ptr;
        (reading.id, reading.celsius)
    }
}

fn main() {
    let readings = vec![
        SensorReading { id: 1, celsius: 21.5 },
        SensorReading { id: 2, celsius: 22.0 },
        SensorReading { id: 3, celsius: 19.8 },
    ];

    println!("=== อ่านค่าในขอบเขตปกติ (index 0..3) ===");
    for i in 0..3 {
        let (id, c) = read_at_offset(&readings, i);
        println!("index {i}: id={id} celsius={c}");
    }

    println!("\n=== อ่านค่าเกินขอบเขตจริง (index 3, 4, 50, 10000) ===");
    println!("readings.len() = {} (index ที่ valid คือ 0..=2 เท่านั้น)", readings.len());
    for i in [3usize, 4, 50, 10_000] {
        let (id, c) = read_at_offset(&readings, i);
        println!("index {i}: id={id} celsius={c}   <-- อ่าน memory ที่ไม่ได้เป็นของ Vec นี้เลย");
    }
}
```

โค้ดนี้ **compile ผ่านสมบูรณ์ ไม่มี warning เลยแม้แต่ตัวเดียว** — รันจริงแล้วได้ผลลัพธ์นี้ (capture จากการรันจริง
บนเครื่อง Linux x86_64):

```
=== อ่านค่าในขอบเขตปกติ (index 0..3) ===
index 0: id=1 celsius=21.5
index 1: id=2 celsius=22
index 2: id=3 celsius=19.8

=== อ่านค่าเกินขอบเขตจริง (index 3, 4, 50, 10000) ===
readings.len() = 3 (index ที่ valid คือ 0..=2 เท่านั้น)
index 3: id=131729 celsius=0   <-- อ่าน memory ที่ไม่ได้เป็นของ Vec นี้เลย
index 4: id=0 celsius=0   <-- อ่าน memory ที่ไม่ได้เป็นของ Vec นี้เลย
index 50: id=0 celsius=0   <-- อ่าน memory ที่ไม่ได้เป็นของ Vec นี้เลย
index 10000: id=0 celsius=0   <-- อ่าน memory ที่ไม่ได้เป็นของ Vec นี้เลย
```

สังเกตสิ่งที่สำคัญที่สุดของผลลัพธ์นี้: **โปรแกรมไม่ crash เลยแม้แต่ครั้งเดียว** ทั้งที่ index 10000 อยู่ไกลเกิน
ขอบเขตของ allocation อย่างชัดเจน — นี่คือ Undefined Behavior ในทางปฏิบัติ: มันไม่ได้แปลว่า "ต้อง crash" มันแปลว่า
**"ผลลัพธ์เป็นอะไรก็ได้ ไม่มีการันตีอะไรเลย"** ในการรันนี้บังเอิญได้ค่า `0`/garbage กลับมาเงียบ ๆ แทนที่จะ crash
— ถ้าค่า `id`/`celsius` ที่ได้ถูกนำไปใช้ต่อในตรรกะทางธุรกิจ (เช่น ใช้ตัดสินใจว่าจะแจ้งเตือนหรือไม่) ระบบจะทำงาน
ผิดพลาดแบบเงียบ ๆ โดยไม่มี error, ไม่มี panic, ไม่มีสัญญาณอะไรเตือนเลยว่ามีบั๊กอยู่ — ต่างจากถ้าใช้ `buf[index]`
ธรรมดา ที่ Rust จะ panic ทันทีด้วยข้อความ `index out of bounds` ให้เห็นชัดเจนตั้งแต่การทดสอบครั้งแรก

**ข้อคิดที่ต้องจำ**: ช่องโหว่ประเภทนี้ (out-of-bounds read ผ่าน `unsafe`) ในภาษาอื่นที่ไม่มี ownership model
(เช่น C) ก็เขียนได้ง่ายพอ ๆ กันและพบได้จริงเป็นสาเหตุของช่องโหว่จำนวนมาก — Rust ไม่ได้ทำให้ปัญหานี้ "หายไป" มัน
แค่ทำให้ปัญหานี้**ถูกจำกัดอยู่ในพื้นที่ที่ประกาศไว้ชัดเจนด้วยคำว่า `unsafe`** ซึ่งเป็นข้อได้เปรียบใหญ่มากตอน
ตรวจสอบโค้ด (code review/audit): ทีมสามารถ `grep -rn "unsafe"` ทั้ง codebase แล้วรู้ได้แน่นอนว่าต้องตรวจสอบ
เฉพาะจุดไหนเป็นพิเศษ — เทียบกับ C/C++ ที่ทุกบรรทัดมีความเสี่ยงแบบนี้เท่ากันหมด ไม่มีทาง grep หา "จุดเสี่ยง" ได้
เลย

#### วินัยของการใช้ `unsafe`: Safe Abstraction, `# Safety` Doc, และ Miri (ทวนจาก Part 41.8/41.11)

Part 41 หัวข้อ 41.8 สอนหลักการ **safe abstraction เหนือ unsafe code** ไว้แล้ว (ห่อ `unsafe` ทั้งหมดไว้ในฟังก์ชัน
เล็ก ๆ ที่ตรวจสอบ invariant เองให้ครบ แล้ว expose แค่ safe API ออกไปให้โค้ดส่วนอื่นเรียกโดยไม่ต้องรู้เลยว่าข้างใน
มี `unsafe`) และ Part 41.11 แนะนำ **Miri** (interpreter ที่ตรวจจับ Undefined Behavior แบบที่ compiler ปกติจับ
ไม่ได้) ไว้เป็นเครื่องมือตรวจสอบ — โค้ด `read_at_offset` ข้างบนถ้ารันผ่าน `cargo +nightly miri run` จะถูกตรวจจับ
ทันทีว่าเป็น out-of-bounds access แม้ในการรันจริงบนเครื่องจะไม่ crash ก็ตาม (Miri จำลอง memory model เข้มงวดกว่า
hardware จริง เพื่อจับ UB ที่ hardware "บังเอิญ" ไม่พังให้เห็น)

หลักปฏิบัติที่ควรทำเป็นมาตรฐานสำหรับทุกทีมที่มี `unsafe` ในโปรเจกต์:

1. **ทำ `unsafe` block ให้เล็กที่สุดเท่าที่จะทำได้** (Part 41.10) — ยิ่ง block เล็ก ยิ่งตรวจสอบ invariant ได้
   ครบและง่ายขึ้นตอน code review
2. **เขียน `# Safety` doc comment ทุกครั้งที่มี `unsafe fn` เป็น public API** ระบุ precondition ที่ผู้เรียก
   ต้องรับผิดชอบเอง (Part 41 กับดักข้อ 2)
3. **รัน Miri เป็นประจำ** สำหรับโค้ดที่มี `unsafe` เยอะ (ไม่ใช่แค่ตอน debug บั๊กที่สงสัยแล้ว) — ทีม production
   จริงหลายทีมตั้งให้ Miri รันเป็น CI job แยกสำหรับ crate ที่มี `unsafe`
4. **ลดปริมาณ `unsafe` ในโค้ดของตัวเองให้น้อยที่สุด** — ถ้ามี safe alternative ที่ทำงานได้ผลลัพธ์เดียวกัน (เช่น
   ใช้ `buf.get(index)` ที่คืน `Option` แทน raw pointer arithmetic) ให้เลือก safe เสมอ ยกเว้นวัดผลจริงแล้วว่า
   performance ต่างกันอย่างมีนัยสำคัญและจำเป็นต้องแลก

#### วัดปริมาณ `unsafe` ทั้ง Dependency Tree ด้วย `cargo geiger`

ปัญหาหนึ่งที่โค้ด application เขียนเองอาจไม่มี `unsafe` เลยแม้แต่บรรทัดเดียว แต่ dependency ที่ดึงมาใช้ (และ
dependency ของ dependency อีกหลายชั้น) อาจมี `unsafe` เต็มไปหมด — โปรเจกต์จริงที่ใช้ crate จำนวนมาก (เช่น Axum
stack ทั้งชุด) จึงมีพื้นที่ `unsafe` ที่ต้อง "เชื่อใจ" อยู่มากกว่าที่มองจากโค้ดของตัวเองมาก `cargo-geiger`
(ติดตั้งเวอร์ชัน 0.13.0 ตอนเขียนบทนี้) คือเครื่องมือที่สแกนทั้ง dependency graph แล้วนับจำนวน `unsafe` usage
จริงในแต่ละ crate ให้เห็นภาพรวม:

```bash
cargo install cargo-geiger --locked
cargo geiger
```

รันกับโปรเจกต์ทดสอบที่มี dependency ใกล้เคียงกับ stack ของ Part 92 (`axum`, `tokio`, `serde`, `regex`) ได้
ผลลัพธ์จริงบางส่วน (ตัดกราฟที่ยาวมากออกเพื่อความกระชับ — dependency graph เต็มมี 91 crate):

```
Metric output format: x/y
    x = unsafe code used by the build
    y = total unsafe code found in the crate

Symbols:
    :) = No `unsafe` usage found, declares #![forbid(unsafe_code)]
    ?  = No `unsafe` usage found, missing #![forbid(unsafe_code)]
    !  = `unsafe` usage found

Functions  Expressions  Impls  Traits  Methods  Dependency

0/0        0/0          0/0    0/0     0/0      ?  audit_demo 0.1.0
0/0        0/0          0/0    0/0     0/0      ?  ├── axum 0.8.9
0/0        0/0          0/0    0/0     0/0      ?  │   ├── axum-core 0.5.6
40/40      780/826      12/14  1/1     16/20    !  │   │   ├── bytes 1.12.1
...
0/0        0/0          0/1    0/0     0/0      ?  ├── regex 1.13.1
8/8        997/997      5/5    1/1     86/86    !  │   ├── aho-corasick 1.1.5
...
26/30      2387/3011    110/119 3/3     109/139  !  └── tokio 1.53.1

173/301    12809/17056  256/342 21/23   469/718
```

อ่านตัวเลขแถวสุดท้าย (ผลรวมทั้งกราฟ): มี `unsafe` function ที่ถูกใช้จริง 173 จาก 301 ที่มีอยู่ทั้งหมด, มี
`unsafe` expression 12,809 จาก 17,056 — ตัวเลขเหล่านี้**ไม่ได้แปลว่าโปรเจกต์นี้ไม่ปลอดภัย** (crate อย่าง `bytes`,
`tokio` มี `unsafe` มากเพราะเป็น low-level building block ที่ implement safe abstraction ไว้ให้แล้ว ตรงตาม
หลักการ Part 41.8) แต่มันบอกว่า**พื้นที่ที่ต้อง "เชื่อใจ" ว่าไม่มีบั๊กมีขนาดใหญ่แค่ไหน** — มีประโยชน์มากตอนต้อง
ตัดสินใจว่า dependency ตัวไหนควรได้รับความสนใจตรวจสอบเป็นพิเศษ (โดยเฉพาะ crate ที่มี `unsafe` เยอะ**และ**ไม่ค่อย
มีคนดูแล/ไม่ค่อยมี test coverage) หรือใช้เป็นเกณฑ์เปรียบเทียบระหว่าง crate สองตัวที่ทำงานเหมือนกันแต่ตัวหนึ่งมี
`unsafe` น้อยกว่าอย่างมีนัยสำคัญ

### 100.3 Dependency และ Supply-Chain Security

Part 35 หัวข้อ 35.20 แนะนำ `cargo audit` ไว้เบื้องต้นแล้วว่ามันเทียบ `Cargo.lock` กับ RustSec Advisory Database
— หัวข้อนี้ลงรายละเอียดทั้งหมดพร้อมรันจริง แล้วเพิ่ม `cargo deny` ที่ยังไม่เคยสอนมาก่อน

#### ความเสี่ยงของ Supply Chain: ทำไมเรื่องนี้ถึงสำคัญกว่าที่คิด

โปรเจกต์ Rust จริงแทบทุกโปรเจกต์ดึง dependency มาจาก crates.io เป็นจำนวนมาก (โปรเจกต์ Axum เล็ก ๆ ก็ดึงมา
มากกว่า 90-100 crate ทางอ้อมอย่างที่หัวข้อก่อนแสดงให้เห็น) โค้ดที่รันจริงในโปรดักชันจึงประกอบด้วยโค้ดที่ทีมเขียน
เองเป็นสัดส่วนน้อยมากเทียบกับโค้ดของคนอื่นที่ดึงมาใช้ทั้งหมด ความเสี่ยงจากจุดนี้มีอย่างน้อยสามรูปแบบที่ควรรู้จัก
ในระดับความตระหนัก (awareness) แม้จะไม่มีทางป้องกันได้ 100%:

1. **Known vulnerability ที่ยังไม่อัปเดต** — crate เวอร์ชันที่ใช้อยู่มีช่องโหว่ที่ถูกค้นพบและประกาศแก้ไปแล้ว
   (มี patch เวอร์ชันใหม่ให้แล้ว) แต่โปรเจกต์ยังไม่ได้ upgrade — นี่คือกรณีที่ `cargo audit` ตรวจจับได้ตรง ๆ
2. **Typosquatting** — crate ที่ตั้งชื่อให้คล้ายกับ crate ที่นิยมใช้กันมาก (สะกดผิดเล็กน้อย เช่นสลับตัวอักษร หรือ
   ใช้ underscore แทน hyphen) เพื่อหลอกให้คนพิมพ์ผิดตอน `cargo add` แล้วดึง crate อันตรายเข้ามาโดยไม่ตั้งใจ —
   crates.io มีระบบตรวจสอบและมาตรการรับมือเรื่องนี้อยู่ แต่ก็ยังเป็นความเสี่ยงที่เกิดขึ้นจริงในทุก package
   ecosystem (npm, PyPI ก็เจอปัญหาแบบเดียวกัน) วิธีป้องกันที่ตรงไปตรงมาที่สุดคือตรวจชื่อ crate ให้ถูกต้องเป๊ะ
   ทุกครั้งก่อน `cargo add` และดูจำนวน download/maintainer ที่ crates.io แสดงไว้เป็นสัญญาณเสริม
3. **Dependency ที่ถูกยึดครองหรือแฝงโค้ดอันตราย (compromised dependency)** — เมนเทนเนอร์บัญชีถูกแฮ็ก หรือ
   maintainer เดิมขายสิทธิ์ดูแล crate ให้คนอื่นที่แฝงโค้ดที่ไม่พึงประสงค์เข้าไปในเวอร์ชันถัดมา — เหตุการณ์แบบนี้
   เกิดขึ้นจริงในหลาย package ecosystem crates.io ลดความเสี่ยงนี้ด้วยกลไกหลายชั้น (ownership transfer ต้องมี
   การยืนยัน, yanking เวอร์ชันที่มีปัญหาไม่ให้ resolve ใหม่ได้อีก) แต่มาตรการที่ทีมพัฒนาทำได้เองคือ **pin
   เวอร์ชันที่ตรวจสอบแล้วผ่าน `Cargo.lock` ที่ commit เข้า git เสมอ** (ไม่ปล่อยให้ `cargo build` ดึงเวอร์ชันใหม่
   ที่ยังไม่ตรวจสอบเข้ามาเองแบบเงียบ ๆ) และอัปเดต dependency แบบตั้งใจเป็นรอบ ๆ พร้อมรีวิว changelog แทนการ
   `cargo update` แบบไม่ดูอะไรเลย

ทั้งสามข้อนี้คือเหตุผลที่ supply-chain security กลายเป็นสาขาหนึ่งของ security ที่แยกออกมาต่างหากในช่วงสิบปีที่
ผ่านมา ไม่ใช่แค่ปัญหาเฉพาะของ Rust — แต่ Rust ecosystem มีเครื่องมือที่ทำงานร่วมกับ Cargo ได้ดีเป็นพิเศษเพราะ
`Cargo.lock` ที่ล็อกทุกเวอร์ชันแบบ deterministic อยู่แล้วเป็นฐานที่ดีมากสำหรับเครื่องมือตรวจสอบพวกนี้

#### `cargo audit`: ตรวจ Known Vulnerability จริงกับ RustSec Advisory Database

ทวนจาก Part 35: ติดตั้งด้วย `cargo install cargo-audit --locked` แล้วรันแค่ `cargo audit` ในโฟลเดอร์โปรเจกต์
— มันจะ clone/อัปเดต advisory database จาก GitHub มาไว้ในเครื่อง แล้วเทียบทุก crate+เวอร์ชันใน `Cargo.lock`

รันจริงกับโปรเจกต์ที่ตั้งใจ pin `time = "0.1.45"` (เวอร์ชันเก่ามากที่มีช่องโหว่ที่รู้จักแล้ว) ไว้ใน
`Cargo.toml`:

```
$ cargo audit
    Fetching advisory database from `https://github.com/RustSec/advisory-db.git`
      Loaded 1271 security advisories (from /root/.cargo/advisory-db)
    Updating crates.io index
    Scanning Cargo.lock for vulnerabilities (71 crate dependencies)
Crate:     time
Version:   0.1.45
Title:     Potential segfault in the time crate
Date:      2020-11-18
ID:        RUSTSEC-2020-0071
URL:       https://rustsec.org/advisories/RUSTSEC-2020-0071
Severity:  6.2 (medium)
Solution:  Upgrade to >=0.2.23

error: 1 vulnerability found!
```

ผลลัพธ์บอกทุกอย่างที่ทีมต้องรู้เพื่อแก้ปัญหา: crate ไหน, เวอร์ชันไหน, ปัญหาคืออะไร (segfault จาก race condition
ตอนอ่าน environment variable ข้าม thread), รุนแรงแค่ไหน (CVSS score 6.2 = medium), และแก้ยังไง (upgrade ไป
`>=0.2.23`) — `cargo audit` คืน **exit code ที่ไม่ใช่ 0** เมื่อเจอช่องโหว่ (สำคัญมากสำหรับ CI: `error: 1
vulnerability found!` ทำให้ CI step ที่รันคำสั่งนี้ fail ทันที ไม่ต้อง parse output เอง) แก้ปัญหาโดย
เปลี่ยนเป็น `time = "0.3"` ใน `Cargo.toml` แล้วรันซ้ำ:

```
$ cargo audit
    Fetching advisory database from `https://github.com/RustSec/advisory-db.git`
      Loaded 1271 security advisories (from /root/.cargo/advisory-db)
    Updating crates.io index
    Scanning Cargo.lock for vulnerabilities (17 crate dependencies)
```

ไม่มีข้อความ error ใด ๆ และ exit code เป็น `0` — สังเกตว่าจำนวน dependency ลดจาก 71 เหลือ 17 ด้วย (เวอร์ชัน
`0.3` ของ `time` ไม่ดึง `wasi`/`libc` เพิ่มเข้ามาแบบที่เวอร์ชัน `0.1` เก่าทำ) นี่คือผลพลอยได้ที่พบได้บ่อย: การ
อัปเดต dependency ให้เป็นเวอร์ชันปัจจุบันมักลด dependency graph ให้เล็กลงไปด้วย ไม่ใช่แค่แก้ช่องโหว่

**ข้อควรรู้เรื่องขอบเขต**: `cargo audit` ตรวจได้แค่ช่องโหว่ที่**ถูกประกาศไว้แล้ว**ใน RustSec Advisory Database
— ถ้า dependency มีบั๊ก memory safety ที่ยังไม่มีใครค้นพบ/รายงาน (zero-day) `cargo audit` จะผ่านสนิทโดยไม่มี
สัญญาณเตือนอะไรเลย เพราะฉะนั้น **ผลลัพธ์ "ผ่าน" ของ `cargo audit` ไม่ได้แปลว่า dependency ทุกตัวปลอดภัย 100%**
มันแปลว่า "ไม่มีช่องโหว่ที่รู้จักแล้วในตอนนี้" เท่านั้น — ต้องรันซ้ำเป็นประจำ (ไม่ใช่รันครั้งเดียวตอนสร้าง
โปรเจกต์) เพราะ advisory database มีรายการใหม่เพิ่มขึ้นทุกวัน dependency ที่ผ่านวันนี้อาจมีช่องโหว่ถูกประกาศ
พรุ่งนี้

#### `cargo deny`: Policy ที่กว้างกว่า — License, Banned Crate, Duplicate Version, Registry Source

`cargo audit` โฟกัสเฉพาะเรื่อง known vulnerability — `cargo deny` (เวอร์ชัน 0.20.2 ตอนเขียนบทนี้) ทำงานกว้าง
กว่านั้นมาก โดยตรวจสอบ**policy ขององค์กร**สี่ด้านพร้อมกันในการรันครั้งเดียว:

- **`advisories`** — ตรวจ known vulnerability เหมือน `cargo audit` (ใช้ RustSec database เดียวกัน)
- **`licenses`** — ตรวจว่า license ของทุก dependency (รวม transitive dependency) อยู่ในรายการที่ทีม
  legal/บริษัทอนุญาตให้ใช้หรือไม่ — สำคัญมากสำหรับซอฟต์แวร์ commercial ที่ไม่สามารถใช้ dependency ที่มี license
  แบบ copyleft (เช่น GPL) ปนเข้ามาได้โดยไม่รู้ตัว
- **`bans`** — แบน crate เฉพาะเจาะจงที่ทีมตัดสินใจไม่ใช้ (เช่น crate ที่ unmaintained แล้ว หรือมี alternative ที่
  ดีกว่า) และเตือนเมื่อพบ dependency เวอร์ชันซ้ำซ้อนหลายเวอร์ชันในกราฟเดียวกัน (ต่อยอด `cargo tree -d` จาก Part
  35 แต่ทำเป็น policy ที่ enforce ได้จริงใน CI)
- **`sources`** — ตรวจว่า dependency ทุกตัวมาจาก registry ที่เชื่อถือได้ (crates.io) เท่านั้น ไม่มี dependency
  ที่ดึงมาจาก git repository หรือ registry อื่นที่ไม่ได้รับอนุญาตแฝงเข้ามา

ตั้งค่า policy ผ่านไฟล์ `deny.toml` ที่ root ของโปรเจกต์ (ต่างจาก `cargo audit` ที่ไม่ต้องมีไฟล์ config เลย —
`cargo deny` ออกแบบมาให้ปรับ policy ได้ละเอียดกว่า):

```toml
# deny.toml -- ตัวอย่าง policy สำหรับ cargo-deny

[graph]
all-features = false

[advisories]
# ใช้ RustSec advisory database เดียวกับ cargo-audit
db-path = "~/.cargo/advisory-db"
ignore = []

[licenses]
# อนุญาตเฉพาะ license ที่ทีม legal ตรวจสอบแล้วว่าใช้ในโปรเจกต์ commercial ได้ปลอดภัย
allow = [
    "MIT",
    "Apache-2.0",
    "Apache-2.0 WITH LLVM-exception",
    "BSD-2-Clause",
    "BSD-3-Clause",
    "Unicode-3.0",
    "Zlib",
]
confidence-threshold = 0.8

[bans]
multiple-versions = "warn"
wildcards = "deny"
deny = [
    # ตัวอย่าง crate ที่ทีมตัดสินใจแบนเพราะ unmaintained/มีปัญหาที่รู้จัก
    { name = "openssl", reason = "ทีมนี้ใช้ rustls เท่านั้น ไม่ผูกกับ OpenSSL bindings" },
]

[sources]
unknown-registry = "deny"
unknown-git = "deny"
allow-registry = ["https://github.com/rust-lang/crates.io-index"]
```

รันด้วย `cargo deny check` กับโปรเจกต์เดิม (ก่อนแก้ `time` และก่อนเพิ่ม `license` field ใน `Cargo.toml`) —
ผลลัพธ์จริงตัดมาเฉพาะส่วนสำคัญ:

```
$ cargo deny check
warning[no-license-field]: license expression was not specified in manifest for crate 'audit_demo = 0.1.0'
 ├ audit_demo v0.1.0

error[unlicensed]: audit_demo = 0.1.0 is unlicensed
 ├ audit_demo v0.1.0 (*)

warning[duplicate]: found 2 duplicate entries for crate 'wasi'
   ┌─ Cargo.lock:64:1
   │
64 │ ╭ wasi 0.10.0+wasi-snapshot-preview1 registry+https://github.com/rust-lang/crates.io-index
65 │ │ wasi 0.11.1+wasi-snapshot-preview1 registry+https://github.com/rust-lang/crates.io-index
   ├ wasi v0.10.0+wasi-snapshot-preview1
     └── time v0.1.45
         └── audit_demo v0.1.0
   ├ wasi v0.11.1+wasi-snapshot-preview1
     └── mio v1.2.3
         └── tokio v1.53.1
             └── audit_demo v0.1.0

error[vulnerability]: Potential segfault in the time crate
 ┌─ Cargo.lock:55:1
55 │ time 0.1.45 registry+https://github.com/rust-lang/crates.io-index
   │ security vulnerability detected
   ├ ID: RUSTSEC-2020-0071
   ├ Solution: Upgrade to >=0.2.23 (try `cargo update -p time`)

advisories FAILED, bans ok, licenses FAILED, sources ok
```

(exit code จริงของคำสั่งนี้คือ `5` — ไม่ใช่ `0`) สรุปบรรทัดสุดท้ายบอกผลของทั้งสี่ policy พร้อมกัน:
`advisories FAILED` (เจอช่องโหว่ตัวเดียวกับที่ `cargo audit` เจอ), `licenses FAILED` (ไม่มี `license` field ใน
`Cargo.toml`), `bans ok`, `sources ok` — ข้อสังเกตที่มีประโยชน์มาก: `cargo deny` เห็น**duplicate version ของ
`wasi`** ที่เกิดจากการ pin `time` เวอร์ชันเก่าไว้ (เหตุผลเดียวกับที่หัวข้อ `cargo audit` ข้างบนสังเกตว่า upgrade
`time` ลด dependency graph ลง) — นี่คือตัวอย่างที่ policy สี่ด้านของ `cargo deny` เชื่อมโยงกันจริงในทางปฏิบัติ
ไม่ได้แยกจากกันเป็นเอกเทศ

แก้ปัญหาโดยเพิ่ม `license = "MIT"` เข้าไปใน `[package]` ของ `Cargo.toml` และเปลี่ยน `time` เป็น `"0.3"` แล้วรัน
ซ้ำ:

```
$ cargo deny check
warning[license-not-encountered]: license was not encountered
   ┌─ deny.toml:18:6
18 │     "BSD-2-Clause",
   │      ━━━━━━━━━━━━ unmatched license allowance
(และอีกหนึ่ง warning แบบเดียวกันสำหรับ "Zlib")

advisories ok, bans ok, licenses ok, sources ok
```

ครั้งนี้ exit code เป็น `0` — เหลือแค่ `warning` สองบรรทัด (บอกว่า allow list มี license สองตัวที่ไม่มี
dependency ตัวไหนใช้จริง ไม่ใช่ปัญหา แค่ข้อสังเกตว่า allow list กว้างกว่าที่ใช้จริง) ทั้งสี่ policy ผ่านหมด:
`advisories ok, bans ok, licenses ok, sources ok`

### 100.4 ผนวก `cargo audit`/`cargo deny` เข้า CI Pipeline

ทีม production จริงไม่รันคำสั่งเหล่านี้ด้วยมือเป็นครั้ง ๆ — Part 97 (CI/CD Pipeline ด้วย GitHub Actions) ตั้ง
pipeline พื้นฐานไว้แล้ว (test, `cargo clippy`, `cargo fmt --check`) สิ่งที่บทนี้เพิ่มเข้าไปคือ **job ตรวจสอบ
ความปลอดภัยของ dependency** ที่รันทุกครั้งที่มี pull request เข้ามา เพื่อจับช่องโหว่/dependency ที่ผิด policy
**ก่อน**ที่จะถูก merge เข้า `main`:

```yaml
# .github/workflows/ci.yml -- เพิ่มเข้าไปในไฟล์ pipeline ที่ Part 97 ตั้งไว้ ในฐานะ job ใหม่
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  # ... job อื่น ๆ ที่ Part 97 สร้างไว้แล้ว (test, clippy, fmt) ...

  security-audit:
    name: Dependency Security Audit
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable

      - name: Cache cargo registry
        uses: actions/cache@v4
        with:
          path: |
            ~/.cargo/registry
            ~/.cargo/git
          key: ${{ runner.os }}-cargo-audit-${{ hashFiles('**/Cargo.lock') }}

      - name: Install cargo-audit
        run: cargo install cargo-audit --locked

      - name: Run cargo audit
        run: cargo audit

      - name: Install cargo-deny
        run: cargo install cargo-deny --locked

      - name: Run cargo deny
        run: cargo deny check
```

สองจุดที่ต้องรู้ก่อนเอาไปใช้จริง:

1. **การ compile `cargo-audit`/`cargo-deny` เองทุกครั้งที่ CI รันช้ามาก** (ทั้งสองตัวใช้เวลาหลายนาทีตอน compile
   ครั้งแรกอย่างที่พิสูจน์ให้เห็นในหัวข้อก่อน) — `actions/cache` ข้างบนช่วยได้บ้าง (cache แค่ registry ไม่ใช่
   binary ที่ compile แล้ว) ทีมที่ต้องการความเร็วกว่านี้มักเปลี่ยนไปใช้ GitHub Action สำเร็จรูปที่มี binary
   pre-built ให้แล้ว เช่น `rustsec/audit-check` (สำหรับ `cargo audit`) และ `EmbarkStudios/cargo-deny-action`
   (สำหรับ `cargo deny` — เขียนโดยทีมเดียวกับที่ดูแล `cargo-deny` เอง) แทนการ `cargo install` เองทุกครั้ง
2. **`cargo audit`/`cargo deny check` คืน exit code ที่ไม่ใช่ 0 เมื่อเจอปัญหา** (พิสูจน์ให้เห็นแล้วในหัวข้อก่อน
   — `error: 1 vulnerability found!` และ exit code `5`) ซึ่งทำให้ GitHub Actions step นี้ fail และบล็อก
   pull request จากการ merge โดยอัตโนมัติ (ถ้าตั้ง branch protection rule ให้ต้องผ่าน status check นี้ก่อน) —
   นี่คือกลไกเดียวกับที่ทำให้ `cargo test`/`cargo clippy` บล็อก PR ได้ใน pipeline ที่ Part 97 สร้างไว้ ไม่มี
   อะไรพิเศษเพิ่มเติมที่ต้องเขียนเอง

### 100.5 Input Validation และ Injection: จาก SQL ถึง Shell Command ถึง File Path

Part 70 พิสูจน์ไว้แล้วด้วยโค้ดจริงว่าการ `format!()` ค่าจาก input ผู้ใช้เข้าไปใน SQL string โดยตรงเปิดช่องให้
เกิด SQL injection ได้ — หัวข้อนี้ทวนหลักการนั้นสั้น ๆ ก่อนขยายไปที่ command injection และ path traversal ซึ่ง
เป็น**ปัญหาเชิงโครงสร้างเดียวกันทุกประการ** เพียงแค่เปลี่ยนบริบทจาก "SQL parser ตีความ string" เป็น "shell
ตีความ string" หรือ "filesystem ตีความ path"

#### ทวนจาก Part 70: SQL Injection และเหตุผลที่ Bind Parameter ป้องกันได้ 100%

หลักการที่ Part 70 หัวข้อ 70.6 พิสูจน์ไว้: การต่อ string เข้า SQL ตรง ๆ

```rust
let unsafe_sql = format!("SELECT isbn, title FROM books WHERE isbn = '{}'", user_input);
```

ทำให้ input ที่มีเครื่องหมายคำพูดหรือ SQL keyword ปนอยู่ (เช่น `nonexistent' OR '1'='1`) เปลี่ยนความหมายของ
query ทั้งหมด — ในขณะที่ bind parameter (`$1` ผ่าน `.bind()` หรือ `sqlx::query!`) ทำให้ PostgreSQL รับค่าทั้ง
สตริงเป็น **"ข้อมูลตัวเดียว"** เสมอ ไม่มีทางถูกตีความเป็น syntax ได้ไม่ว่าจะมีอักขระอะไรอยู่ในนั้น — กลไกนี้คือ
"หัวใจ" ของการป้องกัน injection ทุกรูปแบบที่จะพูดถึงต่อจากนี้: **แยกให้ชัดว่าอะไรคือ "คำสั่ง/syntax" กับอะไรคือ
"ข้อมูล" แล้วส่งข้อมูลผ่านช่องทางที่ระบบไม่มีทางตีความเป็น syntax ได้เลย**

#### Command Injection: เมื่อโปรแกรม Rust เรียก Shell ผ่าน `std::process::Command`

โปรแกรมที่ต้องเรียกโปรแกรมภายนอก (`ffmpeg`, `git`, `imagemagick`, สคริปต์ shell ที่มีอยู่แล้ว) ใช้
`std::process::Command` — ปัญหาเกิดขึ้นเมื่อ**ส่วนหนึ่งของคำสั่งมาจาก input ที่ไม่น่าเชื่อถือ แล้วถูกส่งผ่าน
shell ที่ตีความ metacharacter** (`;`, `&&`, `|`, `` ` ``, `$()`) — เป็นปัญหาเดียวกับ SQL injection เป๊ะ เพียงแค่
เปลี่ยนตัวตีความจาก SQL parser เป็น `sh`

```rust
use std::process::Command;

/// เวอร์ชัน "ไม่ปลอดภัย": ต่อ input ผู้ใช้เข้าไปในสตริงคำสั่งแล้วส่งให้ shell ตีความ
/// เทียบเท่ากับการ format! SQL string เองที่ Part 70 พิสูจน์ไว้แล้วว่าเป็น SQL injection --
/// ที่นี่ shell metacharacter (`;`, `&&`, `|`, `` ` ``, `$()`) เข้ามาแทนเครื่องหมายคำพูดของ SQL
fn run_unsafe(user_supplied_name: &str) -> String {
    let shell_command = format!("echo Hello, {}", user_supplied_name);
    let output = Command::new("sh")
        .arg("-c")
        .arg(shell_command)
        .output()
        .expect("failed to run shell command");
    String::from_utf8_lossy(&output.stdout).to_string()
}

/// เวอร์ชันปลอดภัย: ส่ง input เป็น "argument" หนึ่งตัวตรง ๆ ให้ process ใหม่ ไม่ผ่าน shell เลย --
/// เทียบเท่ากับการใช้ bind parameter ($1) ของ SQLx ที่ Part 70 สอนไว้: ค่าที่รับมาถูกมองเป็น
/// "ข้อมูล" หนึ่งชิ้นเท่านั้น ไม่มีทางถูกตีความเป็น syntax ของ shell ได้เลยไม่ว่าจะมีอักขระอะไรอยู่ในนั้น
fn run_safe(user_supplied_name: &str) -> String {
    let output = Command::new("echo")
        .arg("Hello,")
        .arg(user_supplied_name)
        .output()
        .expect("failed to run echo command");
    String::from_utf8_lossy(&output.stdout).to_string()
}

fn main() {
    // input ปกติ -- ทั้งสองเวอร์ชันให้ผลลัพธ์เหมือนกัน
    let normal_input = "Alice";
    println!("=== input ปกติ: {normal_input:?} ===");
    println!("run_unsafe: {}", run_unsafe(normal_input).trim_end());
    println!("run_safe:   {}", run_safe(normal_input).trim_end());

    // input ที่มี shell metacharacter ปนอยู่ -- จำลองค่าที่มาจาก field ในฟอร์ม/query string
    let tricky_input = "Alice; echo INJECTED-EXTRA-COMMAND";
    println!("\n=== input ที่มี shell metacharacter: {tricky_input:?} ===");
    println!("run_unsafe ผลลัพธ์จริง:\n{}", run_unsafe(tricky_input));
    println!("run_safe ผลลัพธ์จริง:\n{}", run_safe(tricky_input));
}
```

รันจริงได้ผลลัพธ์นี้:

```
=== input ปกติ: "Alice" ===
run_unsafe: Hello, Alice
run_safe:   Hello, Alice

=== input ที่มี shell metacharacter: "Alice; echo INJECTED-EXTRA-COMMAND" ===
run_unsafe ผลลัพธ์จริง:
Hello, Alice
INJECTED-EXTRA-COMMAND

run_safe ผลลัพธ์จริง:
Hello, Alice; echo INJECTED-EXTRA-COMMAND
```

ผลลัพธ์นี้พิสูจน์กลไกให้เห็นตรงกันข้ามกันสองแบบ: `run_unsafe` ส่ง string ที่ประกอบเสร็จ (`"echo Hello, Alice;
echo INJECTED-EXTRA-COMMAND"`) ให้ `sh -c` ตีความ — `sh` เห็น `;` เป็นตัวแบ่งคำสั่ง เลยรันสองคำสั่งจริง
(`echo Hello, Alice` แล้วต่อด้วย `echo INJECTED-EXTRA-COMMAND`) ในขณะที่ `run_safe` ส่ง `"Alice; echo
INJECTED-EXTRA-COMMAND"` ทั้งก้อนเป็น**หนึ่ง argument** ให้ process `echo` ตรง ๆ (ไม่มี shell มาตีความเลย) —
`echo` แค่พิมพ์ argument ที่ได้รับออกมาตามตัวอักษร รวมเครื่องหมาย `;` ด้วย ไม่มีการรันคำสั่งที่สองเกิดขึ้นเลย

**หลักการที่ต้องจำ**: `Command::new(prog).arg(a).arg(b)` (เรียก executable ตรง ๆ พร้อม argument list แยกเป็น
ชิ้น ๆ) กับ `Command::new("sh").arg("-c").arg(format!(...))` (ประกอบ string เดียวแล้วให้ shell parse) เป็นคนละ
กลไกกันโดยสิ้นเชิงในระดับ OS — แบบแรกส่ง argument ไปให้ process ใหม่ผ่าน `execve()` (หรือเทียบเท่า) แบบตรง ๆ
ไม่มี shell คั่นกลางเลย จึงไม่มีทางที่ metacharacter จะถูกตีความ **กฎคือ: ถ้าไม่มีความจำเป็นต้องใช้ shell
feature จริง ๆ (pipe, wildcard expansion, variable substitution ของ shell เอง) ให้เรียก executable ตรง ๆ ผ่าน
`Command::new` เสมอ ไม่ผ่าน `sh -c` เลย** — ถ้าจำเป็นต้องผ่าน shell จริง ๆ (บางกรณีเลี่ยงไม่ได้) ต้อง validate/
escape input อย่างเข้มงวดก่อนเสมอ ซึ่งทำได้ยากกว่าการเลี่ยง shell ไปเลยมาก

#### Path Traversal: เมื่อ Path จาก Request หลุดออกจากขอบเขตที่ตั้งใจ

ปัญหาคลาสเดียวกันอีกครั้ง แต่คราวนี้ตัวตีความคือ **filesystem** — โปรแกรมที่เสิร์ฟไฟล์ตาม path ที่มาจาก request
(เช่น `/files/{filename}`) ถ้าเอา `filename` มาต่อกับ base directory ตรง ๆ โดยไม่ตรวจสอบ ก็เปิดช่องให้ `../`
เดินขึ้นไปนอก base directory ได้ ตามความหมายมาตรฐานของ path resolution ในทุก OS/ภาษา — Part 94 หัวข้อ 94.3 ใช้
`tower_http::services::ServeDir` ในการเสิร์ฟไฟล์ static ของ frontend ไปแล้ว ในฐานะ single-origin serving strategy
ของ capstone project แต่ยังไม่ได้อธิบายลึกว่า `ServeDir` ป้องกันปัญหานี้ได้อย่างไรกันแน่ — หัวข้อนี้พิสูจน์ให้เห็น
ทั้งสองด้าน

**เวอร์ชันที่ไม่ปลอดภัย** (เขียน handler อ่านไฟล์เองโดยไม่ผ่าน `ServeDir`) เทียบกับ**เวอร์ชันที่ปลอดภัย**
(`canonicalize` + ตรวจว่าผลลัพธ์ยังอยู่ในขอบเขต):

```rust
use std::path::{Path, PathBuf};

/// เวอร์ชัน "ไม่ปลอดภัย": เอา path ที่ผู้ใช้ส่งมาต่อเข้ากับ base directory ตรง ๆ ด้วย `Path::join`
/// โดยไม่ตรวจสอบอะไรเลย -- `Path::join` ของ Rust (เหมือนกับทุกภาษา) ให้ `..` เดินขึ้นไปนอก
/// base directory ได้ตามความหมายมาตรฐานของ filesystem path เสมอ
fn naive_resolve(base_dir: &Path, user_supplied_path: &str) -> PathBuf {
    base_dir.join(user_supplied_path)
}

/// เวอร์ชันปลอดภัย: canonicalize ผลลัพธ์ที่ได้ (แปลง `..`/symlink ให้เป็น absolute path ที่ resolve
/// จริงแล้ว) จากนั้นตรวจว่า path ที่ resolve ได้ยัง "อยู่ภายใน" canonicalized base_dir หรือไม่ --
/// ถ้าหลุดออกไปนอก base_dir ให้ปฏิเสธทันที ไม่เปิดไฟล์เด็ดขาด
fn safe_resolve(base_dir: &Path, user_supplied_path: &str) -> Result<PathBuf, String> {
    let candidate = base_dir.join(user_supplied_path);
    let canonical_base = base_dir
        .canonicalize()
        .map_err(|e| format!("base_dir ไม่ถูกต้อง: {e}"))?;
    let canonical_candidate = candidate
        .canonicalize()
        .map_err(|e| format!("ไม่พบไฟล์หรือ path ไม่ถูกต้อง: {e}"))?;

    if canonical_candidate.starts_with(&canonical_base) {
        Ok(canonical_candidate)
    } else {
        Err(format!(
            "ปฏิเสธ: {canonical_candidate:?} อยู่นอกขอบเขตของ {canonical_base:?}"
        ))
    }
}
```

รันจริงกับโฟลเดอร์ `./public/hello.txt` (ไฟล์ที่ตั้งใจให้เสิร์ฟได้) และไฟล์ `./secret_outside.txt` ที่อยู่
**นอก** `public/` (จำลองไฟล์ที่ไม่ควรเข้าถึงได้จาก path ที่เสิร์ฟให้สาธารณะ):

```
=== 1) naive_resolve: ต่อ path ตรง ๆ ไม่ตรวจสอบอะไรเลย ===
request path = "hello.txt" -> resolved = "./public/hello.txt"
  อ่านไฟล์ได้: Ok("This is a public file. Fine to serve.")

request path = "../secret_outside.txt" -> resolved = "./public/../secret_outside.txt"
  อ่านไฟล์ได้: Ok("OUTSIDE_MARKER: this file lives OUTSIDE the public/ directory and must never be served.")

=== 2) safe_resolve: canonicalize + starts_with ตรวจก่อนเปิดไฟล์เสมอ ===
request path = "hello.txt" -> อนุญาต, อ่านได้: "This is a public file. Fine to serve."

request path = "../secret_outside.txt" -> ปฏิเสธ: "/.../pathtrav_demo/secret_outside.txt" อยู่นอกขอบเขตของ "/.../pathtrav_demo/public"
```

ผลลัพธ์ยืนยันตรงตามที่วิเคราะห์ไว้: `naive_resolve` เปิดไฟล์ที่อยู่นอก `public/` ได้จริงเพราะ `Path::join` ไม่รู้
เรื่อง "ขอบเขต" อะไรเลย มันแค่ทำ path arithmetic ตามความหมายมาตรฐาน — `safe_resolve` ปฏิเสธคำขอเดียวกันได้ถูกต้อง
เพราะ `canonicalize()` resolve `..` ให้เป็น absolute path จริงก่อน แล้ว `starts_with()` เทียบว่า path ที่ resolve
ได้ยังอยู่ใต้ base directory หรือไม่

**เปรียบเทียบกับ `tower_http::services::ServeDir`** (ที่ Part 94 ใช้จริง): `ServeDir` implement การตรวจสอบแบบ
เดียวกับ `safe_resolve` ให้อัตโนมัติอยู่แล้ว (เป็นเหตุผลที่ Part 94 เลือกใช้มันแทนการเขียน handler อ่านไฟล์เอง
ตรง ๆ) พิสูจน์ด้วยการรัน Axum server จริงที่ `.fallback_service(ServeDir::new("./public"))` แล้วยิง path
traversal หลายรูปแบบเข้าไปด้วย `curl`:

```
--- normal file ---
$ curl -sS -i http://127.0.0.1:9911/hello.txt
HTTP/1.1 200 OK
content-type: text/plain
content-length: 38

This is a public file. Fine to serve.

--- raw ../ traversal attempt ---
$ curl -sS -i "http://127.0.0.1:9911/../secret_outside.txt"
HTTP/1.1 404 Not Found
content-length: 0

--- url-encoded ..%2f traversal attempt ---
$ curl -sS -i "http://127.0.0.1:9911/..%2Fsecret_outside.txt"
HTTP/1.1 404 Not Found
content-length: 0

--- double-encoded traversal attempt ---
$ curl -sS -i "http://127.0.0.1:9911/%2e%2e%2fsecret_outside.txt"
HTTP/1.1 404 Not Found
content-length: 0
```

ทั้งสามความพยายามหลบหนีขอบเขต (raw `../`, URL-encoded `..%2F`, double-encoded `%2e%2e%2f`) ถูก `ServeDir`
ปฏิเสธด้วย `404 Not Found` เหมือนกันหมด ในขณะที่ไฟล์ปกติ (`hello.txt`) ตอบ `200 OK` พร้อมเนื้อหาถูกต้อง —
`ServeDir` normalize path ที่ได้จาก URL ก่อนแปลงเป็น filesystem path เสมอ (รวม decode URL encoding ก่อน
ตรวจสอบ ไม่ใช่ตรวจสอบ string ดิบที่ยังเข้ารหัสอยู่ ซึ่งเป็นจุดที่ implementation ที่เขียนเองมักพลาด) **หลักการที่
ต้องจำ**: ทุกครั้งที่ต้องเสิร์ฟไฟล์ตาม path จาก request ให้ใช้ `tower_http::services::ServeDir`/`ServeFile`
เสมอแทนการเขียน handler อ่านไฟล์เอง — ถ้าจำเป็นต้องเขียนเอง (เช่น logic การเข้าถึงไฟล์ซับซ้อนกว่าการเสิร์ฟตรง
ๆ) ต้อง `canonicalize()` + `starts_with()` ตรวจก่อนเปิดไฟล์ทุกครั้งไม่มีข้อยกเว้น ตามรูปแบบของ `safe_resolve`
ข้างบน

### 100.6 Secrets Management ในหน่วยความจำ: `secrecy` และ `zeroize`

หลักการพื้นฐาน "ห้าม commit secret เข้า source control" ถูกพูดถึงมาตลอดหลักสูตร (Part 74 กับดักข้อ 4, Part 94
หัวข้อ 94.4 กับรูปแบบ `.env.example` ที่ commit ได้จริงเพราะมีแต่ placeholder ไม่มี secret จริงเลย) — หัวข้อนี้
เพิ่มมุมที่ยังไม่ได้พูดถึง: **secret ที่อ่านจาก environment variable เข้ามาแล้ว ถูกเก็บใน "หน่วยความจำ" อย่าง
ถูกวิธีหรือไม่** ปัญหาของการเก็บ secret เป็น `String` ธรรมดามีสามข้อ:

1. **`Debug`/log พิมพ์ค่าออกมาโดยไม่ตั้งใจ** — ถ้า struct ที่มี field เป็น `String` ของ secret ถูก
   `#[derive(Debug)]` แล้วมีจุดใดจุดหนึ่งใน codebase ที่ `tracing::debug!("{:?}", config)` (เพื่อ debug เหตุผล
   อื่น) secret จะถูกพิมพ์ออกไปที่ log แบบเงียบ ๆ — log มักถูกเก็บไว้นานและมีคนเข้าถึงได้มากกว่าที่คาดไว้
2. **ค่าเดินทางผ่านหลายที่ในหน่วยความจำโดยไม่มีร่องรอย** — `String` ธรรมดา clone ได้ง่าย ส่งผ่านฟังก์ชันได้โดย
   compiler ไม่มีสัญญาณเตือนว่า "ค่านี้ควรถูกจับตามองเป็นพิเศษ"
3. **ค่าไม่ถูกล้างออกจากหน่วยความจำตอน drop** — `String`/`Vec<u8>` ปกติแค่คืน memory กลับ allocator ตอน drop
   โดยไม่ล้างเนื้อหาก่อน (เพราะไม่จำเป็นสำหรับข้อมูลทั่วไป) ค่าเก่ายังเหลืออยู่ใน memory page นั้นจนกว่าจะถูก
   เขียนทับด้วยข้อมูลอื่น

crate `secrecy` และ `zeroize` แก้ปัญหาทั้งสามข้อนี้แบบเจาะจง:

#### `secrecy::SecretString`: ห่อ Secret ให้ `Debug` ไม่รั่วค่าจริง

```rust
use secrecy::{ExposeSecret, SecretString};

#[derive(Debug)]
struct LoginRequest {
    #[allow(dead_code)]
    username: String,
    password: SecretString,
}

#[derive(Debug)]
struct ServiceConfig {
    api_key: SecretString,
    #[allow(dead_code)]
    timeout_secs: u64,
}

fn authenticate(req: &LoginRequest) -> bool {
    // ต้องเรียก .expose_secret() อย่างตั้งใจเพื่อดึงค่าจริงออกมาใช้งาน (เช่น ส่งให้ argon2::verify_password
    // ตามที่ Part 74 สอนไว้) -- การต้องเรียกเมธอดชื่อนี้ตรงๆ ทำให้จุดที่ secret ถูก "แกะ" ออกมาใช้จริง
    // มองเห็นได้ชัดเจนตอน code review ต่างจาก String ธรรมดาที่ค่าโผล่ได้ทุกที่โดยไม่มีสัญญาณเตือนอะไรเลย
    req.password.expose_secret() == "correct-horse-battery-staple"
}

fn main() {
    let req = LoginRequest {
        username: "phutjirakul".to_string(),
        password: SecretString::from("correct-horse-battery-staple".to_string()),
    };
    let config = ServiceConfig {
        api_key: SecretString::from("sk-live-abcdef0123456789".to_string()),
        timeout_secs: 30,
    };

    println!("=== พิสูจน์ว่า Debug/{{:?}} ของ SecretString ไม่รั่วค่าจริง ===");
    println!("println!(\"{{:?}}\", req)    => {:?}", req);
    println!("println!(\"{{:?}}\", config) => {:?}", config);

    println!("\n=== ค่าจริงยังใช้งานได้ปกติผ่าน .expose_secret() (ต้องเรียกอย่างตั้งใจ) ===");
    println!("authenticate(&req) = {}", authenticate(&req));
}
```

รันจริงได้ผลลัพธ์นี้ (ยืนยันด้วย crate `secrecy` เวอร์ชัน 0.10):

```
=== พิสูจน์ว่า Debug/{:?} ของ SecretString ไม่รั่วค่าจริง ===
println!("{:?}", req)    => LoginRequest { username: "phutjirakul", password: SecretBox<str>([REDACTED]) }
println!("{:?}", config) => ServiceConfig { api_key: SecretBox<str>([REDACTED]), timeout_secs: 30 }

=== ค่าจริงยังใช้งานได้ปกติผ่าน .expose_secret() (ต้องเรียกอย่างตั้งใจ) ===
authenticate(&req) = true
```

สังเกตว่า `username` (ข้อมูลที่ไม่ลับ) ยังพิมพ์ค่าจริงออกมาปกติ (`"phutjirakul"`) แต่ `password`/`api_key` พิมพ์
เป็น `SecretBox<str>([REDACTED])` เสมอไม่ว่าจะเรียก `{:?}` กี่ครั้งก็ตาม — `SecretString` (ที่จริงคือ type alias
ของ `SecretBox<str>` ใน secrecy 0.10) implement `Debug` เองแบบ**ตั้งใจ hardcode ให้พิมพ์ `[REDACTED]` เสมอ
ไม่สนใจค่าจริงข้างในเลย** และไม่ implement `Display` ให้เลย (ป้องกันการใช้ `{}` โดยไม่ตั้งใจ) วิธีเดียวที่จะได้
ค่าจริงออกมาคือเรียก `.expose_secret()` ตรง ๆ ซึ่งชื่อเมธอดก็บอกตรง ๆ ว่า "กำลังแกะความลับออกมา" ทำให้จุดที่ค่า
จริงถูกใช้งานมองเห็นได้ชัดเจนตอน code review (ค้นด้วย `grep -rn "expose_secret"` ทั้ง codebase ได้ทันที ตรงกับ
หลักการเดียวกับที่หัวข้อ 100.2 ใช้ `grep "unsafe"` ตรวจสอบจุดเสี่ยง)

#### `zeroize`: ล้างเนื้อหาในหน่วยความจำจริงตอน Drop

`secrecy` ใช้ `zeroize` เป็น dependency ภายในอยู่แล้ว (เมื่อ `SecretBox` ถูก drop มันเรียก `zeroize()` ล้าง
เนื้อหาก่อนคืน memory ให้ allocator โดยอัตโนมัติ) แต่บางสถานการณ์ต้องเรียก `zeroize` ตรง ๆ เอง เช่นตอนจัดการ raw
byte buffer ของ key ที่ไม่ได้ห่อด้วย `secrecy`:

```rust
use zeroize::Zeroize;

fn main() {
    let mut raw_key_bytes: Vec<u8> = b"super-secret-session-key-32bytes".to_vec();
    println!("ก่อน zeroize(): {:?}", raw_key_bytes);
    raw_key_bytes.zeroize();
    println!("หลัง zeroize(): {:?}", raw_key_bytes);
}
```

ผลลัพธ์จริง:

```
ก่อน zeroize(): [115, 117, 112, 101, 114, 45, 115, 101, 99, 114, 101, 116, 45, 115, 101, 115, 115, 105, 111, 110, 45, 107, 101, 121, 45, 51, 50, 98, 121, 116, 101, 115]
หลัง zeroize(): []
```

สังเกตว่า `Vec<u8>::zeroize()` ไม่ได้แค่เขียนศูนย์ทับข้อมูล แต่ยัง `truncate` ความยาวเป็น 0 ด้วย (ผลลัพธ์คือ `[]`
ไม่ใช่ `[0, 0, 0, ...]`) — สิ่งที่สำคัญกว่าตัวเลขที่เห็นคือ**สิ่งที่ compiler ปกติจะเลี่ยงไม่ทำ**: การเขียนโค้ดที่
"แค่เขียนค่าทับตัวแปรแล้วไม่ใช้ต่อ" มีความเสี่ยงที่ compiler optimizer จะเห็นว่าค่าที่เขียนทับไม่มีผลต่อ output
ของโปรแกรม แล้ว**ตัดการเขียนทับนั้นออกไปเลยตอน optimize** (เพราะมันดู "ไม่มีประโยชน์" ในมุมของ compiler) —
`zeroize` ป้องกันปัญหานี้ด้วยการใช้ [`core::ptr::write_volatile`](https://doc.rust-lang.org/core/ptr/fn.write_volatile.html)
ภายใน ซึ่งบอก compiler ว่า "ห้าม optimize การเขียนนี้ออกไปเด็ดขาด ต้องเขียนจริงเสมอ" — นี่คือความต่างสำคัญ
ระหว่าง `zeroize()` กับการเขียน `for b in bytes.iter_mut() { *b = 0; }` เอง ซึ่ง**ไม่การันตี**ว่าจะไม่ถูก
optimize ออกไปในบาง build configuration

**เมื่อไหร่ควรใช้อะไร**: ใช้ `secrecy::SecretString`/`SecretBox<T>` ห่อค่าที่เป็น secret ตั้งแต่จุดที่อ่านเข้ามา
(เช่น อ่าน `JWT_SECRET`/`DATABASE_PASSWORD` จาก environment variable) จนถึงจุดที่ใช้งานจริง เพื่อได้ทั้งการ
ป้องกัน `Debug` รั่วและการล้างหน่วยความจำตอน drop (ผ่าน `zeroize` ที่ทำงานอยู่ข้างใน) โดยไม่ต้องเรียก `zeroize`
เองเลย — ใช้ `zeroize` ตรง ๆ เฉพาะกรณีที่จัดการ raw buffer ที่ไม่ได้ผ่าน `secrecy` (เช่น derive session key ด้วย
ตัวเองแล้วต้องล้าง buffer ชั่วคราวทันทีที่ใช้เสร็จ)

### 100.7 Checklist รวม: Authentication และ Authorization

หัวข้อนี้ไม่สอนกลไกใหม่ — Part 74-76 อธิบายกลไกเชิงลึกไว้ครบแล้ว สิ่งที่หัวข้อนี้ทำคือรวมเป็น **checklist ที่ใช้
ตรวจสอบระบบจริงได้ทันที** พร้อมอ้างอิงกลับไปยังบทที่อธิบายกลไกแต่ละข้อ:

| # | รายการตรวจสอบ | อ้างอิง | สิ่งที่ต้องยืนยัน |
|---|---|---|---|
| 1 | Password hashing | Part 74.6 | ใช้ `argon2` (Argon2id) เท่านั้น — **ห้ามเก็บ/เทียบ password แบบ plaintext เด็ดขาด** |
| 2 | JWT algorithm pinning | Part 74 กับดัก 1 | pin algorithm ที่ยอมรับไว้ตายตัวตอน `decode` (`Validation::new(Algorithm::HS256)`) — ไม่เชื่อ `alg` จาก header ของ token เอง ป้องกัน algorithm confusion/`alg: none` |
| 3 | JWT secret strength | Part 74 กับดัก 6 | secret ของ HS256 ต้องสุ่มและยาวพอ (อย่างน้อย 32 byte) ไม่ใช่ string ที่เดาง่าย |
| 4 | Session cookie attributes | Part 75.4 | `HttpOnly` (กัน JavaScript อ่าน), `Secure` (ส่งผ่าน HTTPS เท่านั้น), `SameSite` (ป้องกัน CSRF พื้นฐาน) ตั้งครบทั้งสามค่า |
| 5 | Session fixation | Part 75.6 | เรียก `cycle_id()` ทุกครั้งที่ privilege เปลี่ยน (login สำเร็จ, เปลี่ยน role) |
| 6 | CSRF protection | Part 75.7 | `SameSite=Lax`/`Strict` เป็นการป้องกันพื้นฐาน + CSRF token แบบ synchronizer pattern เป็น defense-in-depth สำหรับ browser เก่า |
| 7 | OAuth2 state parameter | Part 75.16 | `state` parameter สุ่มจริงทุกครั้งที่สร้าง authorization URL และตรวจสอบตรงกันทุกครั้งที่ callback กลับมา ป้องกัน CSRF บน OAuth2 redirect flow |
| 8 | OAuth2 PKCE | Part 75.15 | ใช้ PKCE (`code_challenge`/`code_verifier`) เสมอสำหรับ Authorization Code Flow |
| 9 | RBAC enforcement | Part 76.4-76.6 | ทุก endpoint ที่ต้องจำกัดสิทธิ์มีการตรวจ role/permission จริงในเส้นทางที่ execute จริง (middleware หรือ handler) ไม่ใช่แค่ซ่อนปุ่มฝั่ง frontend |
| 10 | 403 vs 404 | Part 76.7 | ตัดสินใจอย่างมีเหตุผลระหว่างบอกผู้ใช้ว่า "resource มีอยู่แต่ไม่มีสิทธิ์" (403) กับ "ไม่รู้ว่า resource มีอยู่จริงหรือไม่" (404) ตามความอ่อนไหวของข้อมูล |

การตรวจสอบตาม checklist นี้ไม่ใช่การอ่านทฤษฎีซ้ำ — มันคือการเปิดโค้ดจริงของระบบขึ้นมาไล่ทีละแถว แล้วยืนยันว่า
แต่ละข้อมีการ implement จริงหรือไม่ (หัวข้อ 100.14 ท้ายบทจะทำแบบนี้กับระบบ capstone จาก Part 92-94 อย่างเป็น
รูปธรรม)

### 100.8 TLS/HTTPS: เข้ารหัสข้อมูลระหว่างทางด้วย `rustls`

ทุกหัวข้อก่อนหน้านี้ป้องกันปัญหาที่เกิด**ในตัวแอปพลิเคชัน** — TLS ป้องกันปัญหาคนละชั้น: **ข้อมูลที่เดินทางระหว่าง
client กับ server ผ่าน network** หากไม่เข้ารหัส (HTTP ธรรมดา) ข้อมูลทุกอย่างที่ส่งผ่านไป (password ตอน login,
session cookie, JWT token, ข้อมูลส่วนตัว) เดินทางเป็น plaintext ที่ใครก็ตามที่อยู่ระหว่างทางของ network (เช่น
Wi-Fi network เดียวกัน, router ตัวกลาง, ISP) อ่านได้โดยตรงถ้าดักฟัง traffic ได้ — Part 75 หัวข้อ 75.4 พูดถึง
เรื่องนี้ไปแล้วในบริบทของ cookie attribute `Secure` (ที่บอก browser ว่า "ส่ง cookie นี้ผ่าน HTTPS เท่านั้น") แต่
ยังไม่ได้สอนวิธีตั้งค่า TLS ให้ server จริง

#### ทำไมเลือก `rustls` แทน OpenSSL Bindings

Rust ecosystem มีสอง TLS backend หลักที่ใช้กันแพร่หลาย: `native-tls` (ผูกกับ library TLS ของระบบปฏิบัติการ —
บน Linux ส่วนใหญ่คือ OpenSSL ผ่าน binding) และ `rustls` (implement TLS protocol ทั้งหมดเป็น Rust ล้วน ๆ ไม่พึ่ง
C library เลย) ความต่างที่สำคัญที่สุดในมุมความปลอดภัย:

- **`native-tls`/OpenSSL bindings**: ตัว parser ของ TLS protocol เอง (ส่วนที่ซับซ้อนและมีประวัติช่องโหว่มาก
  ที่สุด) เขียนด้วย C — ช่องโหว่ระดับ critical ของ OpenSSL อย่าง **Heartbleed** (CVE-2014-0160, 2014) เกิดจาก
  buffer over-read แบบเดียวกับที่หัวข้อ 100.2 พิสูจน์ให้เห็นในตัวอย่าง `unsafe` Rust — ต่างกันที่ OpenSSL ไม่มี
  ownership model มาช่วยจำกัดพื้นที่เสี่ยงเลย ทุกบรรทัดมีความเสี่ยงแบบนี้เท่ากันหมด ไม่ใช่แค่ในจุดที่ประกาศ
  `unsafe` ไว้ชัดเจน
- **`rustls`**: implement TLS state machine, certificate parsing, cipher suite ทั้งหมดเป็น safe Rust (มี
  `unsafe` น้อยมาก ส่วนใหญ่กระจุกอยู่ที่ FFI เข้า crypto primitive library ระดับล่างเท่านั้น) — คลาสของ
  ช่องโหว่แบบ Heartbleed (buffer over-read ใน parser) **ถูกตัดออกไปทั้งคลาสโดยอัตโนมัติจาก ownership model**
  ตามหลักการที่หัวข้อ 100.1 อธิบายไว้ (ไม่มี `unsafe`/raw pointer ในส่วน parsing logic เลย)

นี่คือ "เหตุผลที่ใช้ Rust" ที่จับต้องได้จริง ไม่ใช่แค่คำกล่าวลอย ๆ — TLS parser คือโค้ดที่ประมวลผล input ที่มา
จาก network โดยตรง (ผู้โจมตีควบคุมได้เต็มที่) ทำให้เป็นเป้าที่ช่องโหว่ memory safety ส่งผลกระทบร้ายแรงที่สุด
`rustls` ถูก audit อย่างเข้มงวดและใช้งานจริงในโปรเจกต์ระดับ production จำนวนมาก (AWS, Cloudflare, Firefox บาง
ส่วน) รวมถึงเป็น TLS backend ที่ `reqwest` (Part 75 หัวข้อ 75.14) และ `sqlx-cli` (Part 70 หัวข้อ 70.5) เลือกเป็น
ค่า default ในหลักสูตรนี้ทั้งคู่อยู่แล้ว

#### ตั้งค่า Axum Server ให้รองรับ TLS จริงด้วย `axum-server` + `rustls`

`axum_server` (crate แยกจาก `axum` เอง) ให้ `bind_rustls` ที่รับ `RustlsConfig` แทน `axum::serve` ปกติที่ Part
92 ใช้ (ซึ่งเป็น plain HTTP) — ตัวอย่างนี้สร้าง self-signed certificate ด้วย `rcgen` ตอน runtime เพื่อสาธิต (ใน
production ต้องใช้ certificate จาก Certificate Authority จริง เช่นผ่าน Let's Encrypt/ACME หรือ certificate ที่
cloud provider ออกให้ตามที่ Part 101 จะกล่าวถึง):

```rust
use axum::routing::get;
use axum::Router;
use axum_server::tls_rustls::RustlsConfig;

async fn handler() -> &'static str {
    "hello over TLS"
}

#[tokio::main]
async fn main() {
    // สร้าง self-signed certificate จริงตอน runtime ด้วย rcgen (ใช้เพื่อสาธิตเท่านั้น --
    // production ใช้ certificate จาก CA จริง เช่น Let's Encrypt ผ่าน ACME)
    let cert = rcgen::generate_simple_self_signed(vec!["127.0.0.1".to_string(), "localhost".to_string()])
        .expect("สร้าง self-signed cert ไม่สำเร็จ");
    let cert_pem = cert.cert.pem();
    let key_pem = cert.key_pair.serialize_pem();

    // axum-server ต่อยอด rustls ตรง ๆ (ไม่ใช้ OpenSSL bindings เลย) -- โหลด cert/key จาก PEM ในหน่วยความจำ
    let config = RustlsConfig::from_pem(cert_pem.into_bytes(), key_pem.into_bytes())
        .await
        .expect("โหลด TLS config ไม่สำเร็จ");

    let app = Router::new().route("/", get(handler));

    let addr: std::net::SocketAddr = "127.0.0.1:9933".parse().unwrap();
    println!("tls_demo ฟังอยู่ที่ https://127.0.0.1:9933 (rustls, self-signed cert)");
    axum_server::bind_rustls(addr, config)
        .serve(app.into_make_service())
        .await
        .unwrap();
}
```

`Cargo.toml` ที่ตรงกัน:

```toml
[dependencies]
axum = "0.8.9"
axum-server = { version = "0.8.0", features = ["tls-rustls"] }
rcgen = "0.13"
tokio = { version = "1", features = ["full"] }
```

พิสูจน์ด้วย TLS handshake จริงผ่าน `curl -k` (flag `-k` แค่บอกให้ curl ไม่ปฏิเสธ self-signed certificate ที่ไม่
มี CA ที่ browser รู้จักรับรอง — ไม่เกี่ยวกับการเข้ารหัส การเข้ารหัสยังเกิดขึ้นเต็มรูปแบบเหมือนเดิมทุกประการ):

```
$ curl -k -sS -v https://127.0.0.1:9933/
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384 / X25519 / id-ecPublicKey
*  subject: CN=rcgen self signed cert
*  issuer: CN=rcgen self signed cert
* using HTTP/2
> GET / HTTP/2
< HTTP/2 200
hello over TLS
```

ผลลัพธ์ยืนยัน TLS handshake สมบูรณ์จริง: negotiate เป็น **TLSv1.3** ด้วย cipher suite
`TLS_AES_256_GCM_SHA384` และ key exchange แบบ `X25519` (elliptic curve ที่ทันสมัย) — server ตอบด้วย HTTP/2 (ที่
มาพร้อมกับการเปิด TLS โดยอัตโนมัติผ่าน ALPN negotiation) และ response body `hello over TLS` ถูกส่งกลับมาถูกต้อง
ทั้งหมดผ่านช่องทางที่เข้ารหัสแล้ว **หลักการที่ต้องจำ**: ทุก endpoint ที่รับ credential (password, token, cookie
session) ต้องรันผ่าน HTTPS เท่านั้นในโปรดักชัน — cookie attribute `Secure` ที่ Part 75 สอนไว้จะไม่มีความหมายเลย
ถ้า server ไม่มี TLS ให้ browser เชื่อมต่อผ่านตั้งแต่ต้น

### 100.9 Denial-of-Service Awareness: ทวนมาตรการที่มีอยู่แล้วในหลักสูตร

หัวข้อนี้ไม่มีโค้ดใหม่ — เป็นการรวม**มุมมองความปลอดภัย**ให้กับมาตรการที่ Part 65/78/83 implement ไว้แล้วด้วย
เหตุผลอื่น (performance, resource management) แต่มาตรการเดียวกันนี้**คือมาตรการป้องกัน denial-of-service ที่
สำคัญที่สุด** เพราะ availability (ระบบยังใช้งานได้) เป็นเสาหลักหนึ่งของ security ควบคู่กับ confidentiality
(ข้อมูลไม่รั่ว) และ integrity (ข้อมูลไม่ถูกแก้ไขผิดพลาด)

| มาตรการ | อ้างอิง | ทำไมสำคัญด้าน DoS |
|---|---|---|
| `RequestBodyLimitLayer` | Part 65.5 | จำกัดขนาด request body ป้องกันคำขอที่ body ใหญ่ผิดปกติกิน memory/bandwidth ของ server จนกระทบผู้ใช้คนอื่น |
| `TimeoutLayer` | Part 65.5 | จำกัดเวลาที่ request หนึ่งค้างได้นานสุด ป้องกัน connection ที่ทำงานช้าผิดปกติ (ตั้งใจหรือไม่ตั้งใจก็ตาม) จับ worker thread/connection slot ไว้นานเกินจำเป็นจนคำขออื่นรอไม่ได้ |
| Rate limiting (token bucket) | Part 78.9, Part 83.7 | จำกัดจำนวนคำขอต่อหน่วยเวลาต่อ client ป้องกันคำขอปริมาณมากผิดปกติจาก client เดียวจนใช้ resource ของ server เกินสัดส่วนที่ผู้ใช้คนอื่นควรได้รับ |

สามมาตรการนี้ทำงานร่วมกันเป็นชั้น: `RequestBodyLimitLayer`/`TimeoutLayer` ป้องกันคำขอ**ตัวเดียว**ที่ผิดปกติ
(ใหญ่เกินไปหรือช้าเกินไป) ส่วน rate limiting ป้องกัน**ปริมาณ**คำขอที่ผิดปกติจาก client เดียว — ทั้งสามอย่างเป็น
`tower::Layer`/middleware ที่ต่อเข้า Router เดียวกันได้ตามหลักการ Part 65 หัวข้อ 65.13 (capstone middleware
stack) ไม่มีอะไรต้องเขียนเพิ่มถ้า Part 65/78/83 implement ไว้ครบแล้ว — สิ่งที่ต้องทำในเชิง audit คือ**ยืนยันว่า
ทุก endpoint สาธารณะที่ไม่ต้อง authentication (จึงเรียกได้ไม่จำกัดจากใครก็ได้) มีมาตรการเหล่านี้ครอบอยู่จริง**
ไม่ใช่แค่ endpoint ที่ต้อง login เท่านั้น เพราะ endpoint สาธารณะคือเป้าที่ถูกใช้ทำ DoS ได้ง่ายที่สุด (ไม่ต้อง
สมัครบัญชี ไม่ต้องผ่าน authentication ใด ๆ ก่อนเรียก)

### 100.10 Security Headers: ป้องกันชั้นที่ Browser ทำให้แทน

HTTP response header บางตัวสั่งการ browser ให้บังคับใช้นโยบายความปลอดภัยเพิ่มเติมกับหน้าเว็บที่โหลดมา — ต่างจาก
มาตรการก่อนหน้าที่ทำงานฝั่ง server ทั้งหมด, security header เป็นกลไกที่**server สั่ง แต่ browser เป็นผู้บังคับ
ใช้จริง** สี่ตัวที่สำคัญที่สุด:

| Header | ทำหน้าที่อะไร |
|---|---|
| `X-Content-Type-Options: nosniff` | ห้าม browser "เดา" (sniff) MIME type ของ response เองถ้า `Content-Type` ที่ server ส่งมาดูไม่ตรงกับเนื้อหา — ป้องกันสถานการณ์ที่ browser ตีความไฟล์ที่ควรเป็นข้อมูลธรรมดา (เช่น ไฟล์ที่ผู้ใช้อัปโหลด) เป็น HTML/JavaScript แล้วรันมันแทน |
| `X-Frame-Options: DENY` | ห้ามหน้าเว็บนี้ถูกฝังอยู่ใน `<iframe>` ของหน้าเว็บอื่น (ค่า `DENY`) หรืออนุญาตเฉพาะจาก origin เดียวกัน (`SAMEORIGIN`) — ป้องกัน clickjacking (การซ้อนหน้าเว็บที่มองไม่เห็นทับปุ่มของหน้าเว็บอื่นเพื่อหลอกให้คลิก) |
| `Strict-Transport-Security` (HSTS) | สั่ง browser ให้จดจำว่า domain นี้ต้องเข้าถึงผ่าน HTTPS เท่านั้นเป็นเวลา `max-age` วินาทีที่กำหนด — ครั้งถัด ๆ ไปแม้ผู้ใช้พิมพ์ `http://` เอง browser จะเปลี่ยนเป็น `https://` ให้อัตโนมัติก่อนส่ง request จริงด้วยซ้ำ (ป้องกันการดักจับ connection แรกที่ยังเป็น plaintext) |
| `Content-Security-Policy` (CSP) | กำหนดว่าหน้าเว็บนี้อนุญาตให้โหลด resource (script, style, image, ฯลฯ) จาก origin ไหนได้บ้าง — ค่า `default-src 'self'` แปลว่า "อนุญาตเฉพาะ resource จาก origin เดียวกับหน้าเว็บนี้เท่านั้น" เป็นชั้นป้องกันเสริมสำหรับสถานการณ์ที่ script ที่ไม่ได้ตั้งใจหลุดเข้ามาในหน้าเว็บ (เช่นผ่านช่องโหว่ XSS ที่จุดอื่น) ให้มันโหลด resource เพิ่มจากที่อื่นไม่ได้ |

#### เพิ่ม Security Header ผ่าน Axum Middleware (ต่อยอด Part 65)

`tower_http::set_header::SetResponseHeaderLayer` (feature `set-header` ของ `tower-http`) เติม response header
ให้ทุก response ที่ผ่าน Router — ใช้หลักการเดียวกับ `CorsLayer`/`TraceLayer` ที่ Part 65 สอนไว้ทุกประการ (เป็น
`tower::Layer` ที่ห่อ Service เดิมให้กลายเป็น Service ใหม่ที่มีพฤติกรรมเพิ่มเข้ามา):

```rust
use axum::http::HeaderValue;
use axum::routing::get;
use axum::Router;
use tower_http::set_header::SetResponseHeaderLayer;

async fn handler() -> &'static str {
    "hello"
}

fn build_app() -> Router {
    Router::new().route("/", get(handler)).layer(
        tower::ServiceBuilder::new()
            .layer(SetResponseHeaderLayer::overriding(
                axum::http::header::X_CONTENT_TYPE_OPTIONS,
                HeaderValue::from_static("nosniff"),
            ))
            .layer(SetResponseHeaderLayer::overriding(
                axum::http::header::HeaderName::from_static("x-frame-options"),
                HeaderValue::from_static("DENY"),
            ))
            .layer(SetResponseHeaderLayer::overriding(
                axum::http::header::STRICT_TRANSPORT_SECURITY,
                HeaderValue::from_static("max-age=63072000; includeSubDomains"),
            ))
            .layer(SetResponseHeaderLayer::overriding(
                axum::http::header::CONTENT_SECURITY_POLICY,
                HeaderValue::from_static("default-src 'self'"),
            )),
    )
}
```

`Cargo.toml` ต้องเปิด feature `set-header` ของ `tower-http` ตรง ๆ (ไม่ได้เปิดมาโดยอัตโนมัติแม้จะเปิด feature
อื่นแล้ว — กับดักที่พบบ่อยข้อ 2 ท้ายบทจะพิสูจน์ error ที่เกิดถ้าลืม):

```toml
[dependencies]
axum = "0.8.9"
tokio = { version = "1", features = ["full"] }
tower-http = { version = "0.7.1", features = ["set-header"] }
```

พิสูจน์ด้วยการรัน server จริงแล้ว `curl -i` (แสดง response header ทั้งหมด):

```
$ curl -sS -i http://127.0.0.1:9922/
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-security-policy: default-src 'self'
strict-transport-security: max-age=63072000; includeSubDomains
x-frame-options: DENY
x-content-type-options: nosniff
content-length: 5
date: Sun, 27 Sep 2026 05:28:13 GMT

hello
```

ทั้งสี่ header ปรากฏใน response จริงครบทุกตัว **ข้อควรระวังเรื่อง `Strict-Transport-Security`**: header นี้มี
ความหมายเฉพาะเมื่อ response มาจาก HTTPS จริง (ตามหัวข้อ 100.8) — การส่ง `Strict-Transport-Security` ผ่าน HTTP
ธรรมดา (ไม่มี TLS) จะถูก browser **เมิน** โดยสิ้นเชิงตามข้อกำหนดของ HSTS เอง (ป้องกันไม่ให้ attacker ที่ควบคุม
HTTP connection ที่ไม่เข้ารหัสสั่ง downgrade HSTS policy ได้) เพราะฉะนั้น header นี้มีประโยชน์จริงก็ต่อเมื่อ
deploy ผ่าน HTTPS แล้วเท่านั้น — ตั้งไว้ตั้งแต่ตอนพัฒนาไม่มีผลเสีย แต่ต้องรอ deploy จริงผ่าน TLS ก่อนมันจะมีผล

### 100.11 Capstone Security Audit: ตรวจสอบระบบห้องสมุดจาก Part 92-94 ทั้งระบบ

ถึงเวลานำทุกอย่างที่ผ่านมาทั้งบทมาใช้งานจริง — นำ checklist ที่ประกอบจากหัวข้อ 100.1-100.10 ไปตรวจสอบระบบ
capstone จริง (ระบบห้องสมุด/ยืม-คืนหนังสือจาก Part 92-94) อย่างเป็นระบบ ทีละหัวข้อ

#### สิ่งที่ตรวจสอบแล้วพบว่า**มีมาตรการป้องกันอยู่แล้ว** (ยืนยันพร้อมอ้างอิง)

| หัวข้อตรวจสอบ | สถานะ | อ้างอิง |
|---|---|---|
| SQL injection | ✅ ป้องกันแล้ว | ทุก query ใน `routes/*.rs` ใช้ `sqlx::query!`/`query_as!` ที่ bind parameter ผ่าน `$1, $2, ...` เสมอ (Part 92 สืบทอดจาก Part 70 โดยตรง ไม่มีจุดใด `format!()` SQL string เลย) |
| Password hashing | ✅ ป้องกันแล้ว | `auth::register`/`auth::login` เรียก `argon2::Argon2::default().hash_password(...)`/`.verify_password(...)` (Part 92 สืบทอด Part 74.6) ไม่มีการเทียบ plaintext ที่ไหนเลย |
| JWT algorithm pinning | ✅ ป้องกันแล้ว | `decode_access_token` ปักหมุด `Validation::new(Algorithm::HS256)` ตรง ๆ (Part 92 สืบทอด Part 74.10) |
| RBAC enforcement | ✅ ป้องกันแล้ว | endpoint สร้างหนังสือ (`POST /api/v1/books`) ตรวจ `current_user.is_admin()` ในตัว handler ก่อน insert เสมอ (Part 92 สืบทอด Part 76.6) |
| CORS scope | ✅ ป้องกันแล้ว (แนวโน้มดีตั้งแต่ต้น) | `build_app` ตั้ง `allow_origin` เป็น `frontend_origin` ที่เจาะจง (parse จาก environment variable) ไม่ใช่ `Any`/`CorsLayer::permissive()` — ตรงกับหลักการ "จำกัดให้แคบที่สุดที่จำเป็น" ตาม Part 65 หัวข้อ CorsLayer |
| Race condition ตอนยืมหนังสือ | ✅ ป้องกันแล้ว | ใช้ transaction + atomic conditional update ตาม Part 92 หัวข้อ 92.x (พิสูจน์ด้วยการยิง 10 request พร้อมกันจริงไปแล้วในบทนั้น) — ไม่ใช่ security bug โดยตรงแต่เป็น integrity guarantee ที่เกี่ยวข้อง |

#### ช่องโหว่จริงที่พบระหว่างตรวจสอบ และการแก้ไข

**ช่องโหว่ที่ 1: `JWT_SECRET` มีค่า default ที่ fallback แบบเงียบ ๆ ถ้าไม่ได้ตั้ง environment variable**

`main.rs` ของทั้ง Part 92 และ Part 94 มีโค้ดนี้:

```rust
let jwt_secret = std::env::var("JWT_SECRET")
    .unwrap_or_else(|_| "dev-only-secret-change-me-in-production".to_string());
```

ปัญหา: ถ้าลืมตั้ง `JWT_SECRET` ตอน deploy จริง (เช่น ลืมตั้งใน environment variable ของ deployment platform)
แอปจะ**รันต่อไปได้เงียบ ๆ** โดยใช้ secret ที่เป็น string คงที่ที่**พิมพ์อยู่ใน source code ที่เปิดสาธารณะทุกคนอ่าน
ได้** (Part 94 หัวข้อ Exercises ข้อ 1 ระบุปัญหานี้ไว้เป็นแบบฝึกหัดให้ผู้เรียนไปแก้เอง — บทนี้แก้ให้เห็นเป็น
ตัวอย่างที่สมบูรณ์) ผลคือใครก็ตามที่รู้ค่า string นี้ (ซึ่งอยู่ใน source code ที่เผยแพร่แล้ว) สามารถสร้าง JWT
token ที่ signature ตรวจผ่านได้เอง — เทียบเท่ากับไม่มี authentication เลย

**การแก้ไข**: เปลี่ยนให้ **fail-fast ทันทีตอน startup** ถ้าไม่ได้ตั้ง `JWT_SECRET` (ไม่ fallback ไปใช้ค่า
default ที่ไม่ปลอดภัยเด็ดขาด) พร้อมตรวจความยาวขั้นต่ำ และห่อด้วย `secrecy::SecretString` ทันทีตั้งแต่จุดที่อ่าน
เข้ามา ตามหลักการหัวข้อ 100.6:

```rust
use secrecy::{ExposeSecret, SecretString};

/// AppState เวอร์ชันแก้ไข -- jwt_secret เปลี่ยนจาก Arc<str> เป็น Arc<SecretString>
#[derive(Clone)]
struct AppState {
    jwt_secret: std::sync::Arc<SecretString>,
    // ... field อื่น ๆ (db: PgPool) เหมือนเดิมตาม Part 92
}

impl std::fmt::Debug for AppState {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.debug_struct("AppState")
            .field("jwt_secret", &self.jwt_secret)
            .finish()
    }
}

/// เวอร์ชันที่แก้แล้ว: fail-fast ทันทีตอน startup ถ้าไม่ได้ตั้ง JWT_SECRET จริง ๆ
/// และคืนค่าเป็น SecretString ทันทีที่อ่านจาก environment (ไม่มีช่วงที่เป็น String เปล่าเดินทางอยู่เลย)
fn load_jwt_secret_secure() -> SecretString {
    let raw = std::env::var("JWT_SECRET").unwrap_or_else(|_| {
        eprintln!("FATAL: ต้องตั้ง environment variable JWT_SECRET ก่อนเริ่มเซิร์ฟเวอร์ (ไม่มีค่า default ที่ปลอดภัยพอสำหรับ production)");
        std::process::exit(1);
    });
    if raw.len() < 32 {
        eprintln!("FATAL: JWT_SECRET สั้นเกินไป ({} ตัวอักษร) ต้องมีอย่างน้อย 32 ตัวอักษร", raw.len());
        std::process::exit(1);
    }
    SecretString::from(raw)
}
```

พิสูจน์ทั้งสามสถานการณ์ด้วยการรันจริง:

```
=== case 1: ไม่ได้ตั้ง JWT_SECRET เลย ===
FATAL: ต้องตั้ง environment variable JWT_SECRET ก่อนเริ่มเซิร์ฟเวอร์ (ไม่มีค่า default ที่ปลอดภัยพอสำหรับ production)
exit code: 1

=== case 2: ตั้ง JWT_SECRET สั้นเกินไป ===
FATAL: JWT_SECRET สั้นเกินไป (5 ตัวอักษร) ต้องมีอย่างน้อย 32 ตัวอักษร
exit code: 1

=== case 3: ตั้ง JWT_SECRET ถูกต้อง ===
AppState แบบ Debug: AppState { jwt_secret: SecretBox<str>([REDACTED]) }
ค่าจริงที่ใช้เซ็น JWT จริง (ยาว 54 ตัวอักษร): ใช้งานได้ปกติผ่าน .expose_secret()
exit code: 0
```

การแก้ไขนี้เปลี่ยนความล้มเหลวจาก "เงียบและอันตราย" (แอปรันต่อได้ด้วย secret ที่ไม่ปลอดภัย) เป็น "ดังและปลอดภัย"
(แอปหยุดทำงานทันทีพร้อมข้อความชัดเจนว่าต้องแก้อะไร) — และผลพลอยได้จากการห่อด้วย `SecretString`: ถ้ามีจุดอื่นใน
codebase ที่ `tracing::debug!("state = {:?}", state)` เพื่อ debug เหตุผลอื่น (เช่น debug เรื่อง `db` pool) จะไม่มี
ความเสี่ยงที่ `jwt_secret` รั่วออกไปที่ log โดยไม่ตั้งใจอีกต่อไป

**ช่องโหว่ที่ 2: ไม่มี security header เลยในทุก response**

ตรวจสอบ `build_app` ของ Part 92/94 พบว่า stack มีแค่ `CorsLayer` และ `TraceLayer` — ไม่มี
`SetResponseHeaderLayer` ตัวไหนเลยตามหัวข้อ 100.10 **การแก้ไข**: เพิ่มเข้าไปในจุดเดียวกับที่ประกอบ layer อื่น ๆ
อยู่แล้ว (`build_app` ใน `lib.rs`) ตามรูปแบบที่หัวข้อ 100.10 พิสูจน์ไว้แล้วว่าทำงานจริง — ต่อ
`SetResponseHeaderLayer` ทั้งสี่ตัวเข้ากับ `.layer(cors).layer(TraceLayer::new_for_http())` ที่มีอยู่แล้ว โดย
ไม่กระทบ endpoint หรือ business logic ใดเลย เพราะเป็น middleware ที่ทำงานกับทุก response แบบเดียวกันหมด
ไม่ขึ้นกับ route

**ช่องโหว่ที่ 3 (ระดับ architecture, ไม่ใช่บั๊ก): ไม่มี TLS ในเวอร์ชันที่ verify ได้ในบทนี้**

Part 92-94 รันผ่าน plain HTTP ตลอด (`axum::serve` ไม่ใช่ `axum_server::bind_rustls`) ซึ่ง**สมเหตุสมผลสำหรับ
สภาพแวดล้อม dev/capstone-ทดสอบในเครื่อง** ที่ Part 92-94 ตั้งใจไว้ (frontend/backend รันในเครื่องเดียวกันผ่าน
`127.0.0.1`) แต่เมื่อ deploy จริง (ตามที่ Part 101 จะสอนต่อ) **ต้องมี TLS อยู่ระหว่าง client กับ server เสมอ** —
ในทางปฏิบัติจริง TLS termination มักทำที่ load balancer/reverse proxy ของ cloud provider (ไม่ใช่ที่ตัว Axum
process เอง) แต่หลักการ `rustls` ที่หัวข้อ 100.8 พิสูจน์ไว้ก็ใช้ได้เหมือนกันถ้าเลือก terminate TLS ที่ตัวแอปเอง
โดยตรง — ไม่ว่าจะเลือกสถาปัตยกรรมไหน **ข้อสรุปของการ audit ข้อนี้คือบันทึกไว้ชัดเจนเป็นข้อกำหนดก่อน deploy** ไม่
ใช่ปัญหาที่ต้องแก้ในโค้ดของ capstone project ตอนนี้

การ audit ทั้งสามข้อนี้แสดงรูปแบบที่ใช้ตรวจสอบระบบจริงได้ทุกระบบ ไม่จำกัดแค่ capstone project นี้: **ไล่
checklist ทีละข้อ อ่านโค้ดจริงเพื่อยืนยัน (ไม่ใช่เดา), อ้างอิงกลับไปยังบทที่อธิบายกลไก, และแก้ไขช่องโหว่ที่พบด้วย
การพิสูจน์ก่อน-หลังเหมือนที่ทำมาตลอดทั้งบทนี้**

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. ใช้ `SecretString`/`SecretBox` กับ `{}` (Display) ตรง ๆ แทน `.expose_secret()`

```rust
fn broken_log(req: &LoginRequest) {
    println!("logging in with password: {}", req.password);
}
```

```
error[E0277]: `SecretBox<str>` doesn't implement `std::fmt::Display`
  --> src/main.rs:56:46
   |
56 |     println!("logging in with password: {}", req.password);
   |                                         --   ^^^^^^^^^^^^ `SecretBox<str>` cannot be formatted with the default formatter
   |
   = help: the trait `std::fmt::Display` is not implemented for `SecretBox<str>`
   = note: in format strings you may be able to use `{:?}` (or {:#?} for pretty-print) instead
```

`secrecy` ตั้งใจไม่ implement `Display` ให้เลย (ต่างจาก `Debug` ที่ implement ให้แต่ hardcode พิมพ์
`[REDACTED]` เสมอ) — compiler error นี้**คือฟีเจอร์ ไม่ใช่บั๊ก**: มันบังคับให้ทุกจุดที่ต้องใช้ค่าจริงต้องเรียก
`.expose_secret()` อย่างตั้งใจเสมอ ทำให้ grep หาจุดที่ secret ถูกแกะออกมาใช้ได้ครบทุกจุดจริง ๆ วิธีแก้: เรียก
`req.password.expose_secret()` เมื่อต้องการค่าจริง (เช่น ส่งให้ `argon2::verify_password`) ไม่ใช่พยายามพิมพ์
log ค่า secret เลยไม่ว่าจะผ่านช่องทางไหน

### 2. เปิด `tower-http` แต่ลืมเปิด feature `set-header`

```rust
use tower_http::set_header::SetResponseHeaderLayer;
```

```
error[E0432]: unresolved import `tower_http::set_header`
 --> src/main.rs:4:17
  |
4 | use tower_http::set_header::SetResponseHeaderLayer;
  |                 ^^^^^^^^^^ could not find `set_header` in `tower_http`
  |
note: found an item that was configured out
 --> tower-http-0.7.1/src/lib.rs:214:9
  |
213 | #[cfg(feature = "set-header")]
  |       ---------------------- the item is gated behind the `set-header` feature
214 | pub mod set_header;
```

`tower-http` แยก module ทุกตัวเป็น feature flag ของตัวเองทั้งหมด (เหมือนที่ Part 65 อธิบายไว้สำหรับ `cors`/
`trace`/`timeout`/`limit`) — `set-header` ไม่ได้ถูกเปิดมาพร้อมกับ feature อื่นเลย ต้องเปิดเอง:
`tower-http = { version = "0.7.1", features = ["set-header"] }`

### 3. เวอร์ชันของ `axum-server`/`rcgen` เปลี่ยน field/type signature บ่อย

```rust
let key_pem = cert.signing_key.serialize_pem();
```

```
error[E0609]: no field `signing_key` on type `CertifiedKey`
  --> src/main.rs:16:24
   |
16 |     let key_pem = cert.signing_key.serialize_pem();
   |                        ^^^^^^^^^^^ unknown field
   |
   = note: available fields are: `cert`, `key_pair`
```

และหลังแก้เป็น `cert.key_pair` แล้ว การเรียก `axum_server::bind_rustls("...".parse().unwrap(), config)` แบบไม่
ระบุ type อาจเจอ error อีกชั้น:

```
error[E0283]: type annotations needed
  --> src/main.rs:26:47
   |
26 |     axum_server::bind_rustls("127.0.0.1:9933".parse().unwrap(), config)
   |                                               ^^^^^ cannot infer type of the type parameter `F` declared on the method `parse`
   |
   = note: cannot satisfy `_: Address`
```

crate ระดับ infrastructure อย่าง `rcgen`/`axum-server` เปลี่ยน field name และ generic bound ระหว่างเวอร์ชันบ่อย
กว่า crate ระดับ application ทั่วไป — วิธีแก้ที่แนะนำเสมอ: อย่า copy โค้ดจาก tutorial เก่าตรง ๆ ให้เปิด
documentation ของเวอร์ชันที่ระบุไว้จริงใน `Cargo.lock` (ผ่าน `cargo doc --open -p rcgen` หรือดูที่
docs.rs พร้อมระบุเวอร์ชัน) และระบุ type ให้ compiler ชัดเจนเมื่อ error บอกว่า "type annotations needed"
(ในกรณีนี้คือประกาศ `let addr: std::net::SocketAddr = "...".parse().unwrap();` แยกออกมาก่อน)

### 4. ลืมใส่ `license` field ใน `Cargo.toml` ทำให้ `cargo deny check` fail ที่ license check แม้ dependency ทุกตัวผ่านหมด

```
warning[no-license-field]: license expression was not specified in manifest for crate 'audit_demo = 0.1.0'
 ├ audit_demo v0.1.0

error[unlicensed]: audit_demo = 0.1.0 is unlicensed
 ├ audit_demo v0.1.0 (*)
```

`cargo deny` ตรวจ license ของ**ทุก crate ในกราฟรวมถึง package ของตัวเอง** — ถ้า `Cargo.toml` ไม่มี `license`
field (เป็นเรื่องปกติมากสำหรับโปรเจกต์ที่เพิ่งสร้างด้วย `cargo new` และยังไม่ตั้งใจ publish) `licenses` check
จะ fail ทันทีแม้ dependency ทั้งหมดจะผ่านหมด วิธีแก้: เพิ่ม `license = "MIT"` (หรือ license ที่ตรงกับนโยบายของ
ทีม) เข้าไปใน `[package]` ของ `Cargo.toml`

### 5. เข้าใจผิดว่า `cargo audit`/`cargo deny check` "ผ่านหมด" แปลว่าโค้ดปลอดภัย 100%

ไม่มี compiler error สำหรับข้อนี้เพราะเป็นความเข้าใจผิดเชิงแนวคิด แต่เป็นกับดักที่พบบ่อยมากพอจะบันทึกไว้: ผลลัพธ์
"ผ่าน" ของทั้งสองเครื่องมือบอกได้แค่ **"ไม่มีช่องโหว่ที่ถูกประกาศไว้แล้วใน advisory database ณ ตอนที่รัน"**
(หัวข้อ 100.3) ไม่ได้แปลว่าไม่มี zero-day, ไม่ได้แปลว่าโค้ดของทีมเองไม่มี logic bug/SQL injection/broken auth
(หัวข้อ 100.1) และไม่ได้แปลว่า `unsafe` code ในโปรเจกต์ (ถ้ามี) ไม่มี Undefined Behavior (หัวข้อ 100.2) — ต้อง
รันทั้งสองเครื่องมือเป็นประจำ (ไม่ใช่ครั้งเดียว) และมองเป็นแค่**หนึ่งชั้น**ในหลาย ๆ ชั้นของ security ที่บทนี้
รวบรวมไว้ ไม่ใช่ชั้นเดียวที่พอแล้ว

### 6. Fallback ไปใช้ secret ค่า default แบบเงียบ ๆ เมื่อไม่ได้ตั้ง environment variable

```rust
let jwt_secret = std::env::var("JWT_SECRET")
    .unwrap_or_else(|_| "dev-only-secret-change-me-in-production".to_string());
```

โค้ดแบบนี้ (ที่พบจริงในเวอร์ชันเริ่มต้นของ Part 92/94 ตามที่หัวข้อ 100.14 ตรวจพบ) compile ผ่านสมบูรณ์ ไม่มี
warning ไม่มี error — และนั่นคือปัญหา: มันทำให้แอป**รันต่อได้เงียบ ๆ ในโปรดักชันโดยใช้ secret ที่รู้กันทั่วไป**
ถ้าลืมตั้ง environment variable ก่อน deploy วิธีแก้คือ fail-fast ทันทีตอน startup (`std::process::exit(1)`
พร้อมข้อความชัดเจน) แทนการ fallback แบบเงียบ ๆ ตามที่หัวข้อ 100.14 พิสูจน์ไว้ — หลักการทั่วไป: **environment
variable ที่เป็น secret ไม่ควรมีค่า default ที่ "ใช้งานได้" เลย ไม่ว่ากรณีไหน** ค่า default ที่ปลอดภัยที่สุด
สำหรับ secret คือ "ไม่มี" (บังคับให้ตั้งเองเสมอ)

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** สร้างโปรเจกต์ Rust ใหม่ เพิ่ม dependency `time = "0.1.45"` เข้าไปตรง ๆ ใน `Cargo.toml` แล้วรัน
   `cargo audit` จริง — ยืนยันว่าเจอ `RUSTSEC-2020-0071` เหมือนที่บทนี้พิสูจน์ไว้ จากนั้นแก้เป็น `time = "0.3"`
   แล้วรันซ้ำเพื่อยืนยันว่าผ่านสะอาด (exit code 0) — Hint: ใช้ `echo $?` ทันทีหลังรันคำสั่งเพื่อดู exit code จริง

2. **(กลาง)** เขียนฟังก์ชัน Rust ที่รับ hostname เป็น `&str` แล้วเรียก `ping -c 1 <hostname>` สองแบบ: แบบที่ผ่าน
   `sh -c` (ต่อ string เอง) และแบบที่เรียก `Command::new("ping")` ตรง ๆ พร้อม `.arg("-c").arg("1").arg(hostname)`
   — ทดสอบด้วย hostname ปกติ (เช่น `"127.0.0.1"`) ก่อน แล้วทดสอบด้วย string ที่มี `; echo test-marker` ปนอยู่ —
   สังเกตความต่างของผลลัพธ์ตามรูปแบบที่หัวข้อ 100.5 พิสูจน์ไว้กับ `echo` — Hint: ระวังว่า `ping` ต้องมี argument
   ที่ถูกต้องครบ ไม่ใช่แค่ hostname เฉย ๆ

3. **(ยาก)** เขียน `AppConfig` struct ที่มี field `database_password: String`, `api_key: String`, และ
   `max_connections: u32` — เปลี่ยนสอง field แรกเป็น `secrecy::SecretString` แล้ว derive `Debug` ให้ struct
   ทั้งก้อน พิสูจน์ด้วยการ `println!("{:?}", config)` ว่า field ที่เป็น `SecretString` ไม่โผล่ค่าจริง ในขณะที่
   `max_connections` ยังโผล่ปกติ จากนั้นเขียนฟังก์ชันที่ต้องใช้ค่าจริงของ `api_key` (เช่น ส่งเป็น header ไปยัง
   HTTP request จำลอง) เพื่อพิสูจน์ว่ายังใช้งานได้ผ่าน `.expose_secret()` — Hint: ต้อง `use
   secrecy::ExposeSecret` ก่อนเรียกเมธอดนี้

4. **(ยากมาก/ประยุกต์ใช้งานจริง)** ทำการ security audit เต็มรูปแบบกับโปรเจกต์ของคุณเอง (หรือโปรเจกต์ capstone
   จาก Part 92-94 ที่คัดลอกโค้ดมาสร้างใหม่ตามที่บทนั้นบอกว่าทำได้จริง) ตามลำดับนี้: (ก) รัน `cargo audit` และ
   `cargo deny check` จริง แก้ทุกปัญหาที่เจอจนผ่านสะอาดทั้งคู่ (ข) เพิ่ม `SetResponseHeaderLayer` ทั้งสี่ตัวจาก
   หัวข้อ 100.10 เข้าไปใน `build_app` จริง แล้วยืนยันด้วย `curl -i` ว่า header ปรากฏครบ (ค) แก้จุดที่อ่าน
   `JWT_SECRET`/secret อื่นให้ fail-fast แทนการ fallback แบบเงียบ ๆ ตามหัวข้อ 100.14 พร้อมห่อด้วย
   `secrecy::SecretString` (ง) เพิ่ม job `security-audit` เข้า CI pipeline ของโปรเจกต์จริง (ถ้ามี GitHub
   Actions workflow อยู่แล้วจาก Part 97) — บันทึกทุกขั้นตอนพร้อมผลลัพธ์จริงก่อน/หลังแก้แต่ละจุด ตามรูปแบบที่
   บทนี้ทำให้เห็นตลอดทั้งบท — Hint: ทำทีละข้อและ commit แยกกันในแต่ละขั้น จะตรวจสอบและ rollback ง่ายกว่าทำทีเดียว
   ทั้งหมด

## สรุป

บทนี้ไม่ได้สอนเทคนิคใหม่แยกส่วนเหมือนบทอื่น ๆ ในหลักสูตร — มันคือบทสังเคราะห์ที่รวบรวมทุกอย่างด้านความปลอดภัยที่
กระจายอยู่ทั่วทั้งหลักสูตรตั้งแต่ Part 6 จนถึง Part 94 เข้าเป็นภาพเดียว แล้วเติมเนื้อหาที่ยังขาดเข้าไปในจุดที่
สำคัญที่สุด สิ่งที่ต้องจำที่สุดจากบทนี้คือข้อสรุปของหัวข้อ 100.1: **memory safety ที่ Rust ให้มา (buffer
overflow/use-after-free/double-free/data race ที่ป้องกันได้ตอน compile time) เป็นสับเซ็ตหนึ่งของ security ที่
กว้างกว่ามาก** — logic bug, SQL/command injection, path traversal, broken authentication/authorization, บั๊ก
ใน `unsafe` code, และความเสี่ยงจาก supply chain ล้วนเป็นสิ่งที่ ownership model ไม่มีส่วนเกี่ยวข้องเลย และต้อง
ใช้เครื่องมือ/แนวปฏิบัติที่ถูกต้องเจาะจงกับแต่ละปัญหา — `cargo audit`/`cargo deny` สำหรับ dependency,
`secrecy`/`zeroize` สำหรับ secret ในหน่วยความจำ, `rustls` สำหรับข้อมูลระหว่างทาง, bind parameter/argument list
ที่ถูกต้องสำหรับ injection ทุกคลาส, และ checklist ที่อ้างอิงกลับไปยัง Part 74-76 สำหรับ authentication/
authorization — ปิดท้ายด้วยการนำทุกอย่างมาตรวจสอบระบบ capstone จริงจาก Part 92-94 จนพบและแก้ช่องโหว่จริงสองจุด
(secret fallback ที่ไม่ปลอดภัย และ security header ที่ขาดไป) พิสูจน์ว่า checklist นี้ใช้งานได้จริงกับโค้ดจริง
ไม่ใช่แค่ทฤษฎี

Part ถัดไป (**Part 101: Deployment: Cloud Platforms (AWS/GCP/Fly.io)**) จะนำระบบ capstone ที่ผ่านการตรวจสอบ
ความปลอดภัยจากบทนี้ไป deploy จริงบน cloud platform — คำถามเรื่อง TLS termination, environment variable/secret
management ของแต่ละ platform, และ network policy ที่บทนั้นจะเจอ ล้วนต่อยอดจากหลักการที่บทนี้วางไว้ตรงทุกจุด

---

**Part ก่อนหน้า:** [Observability: Distributed Tracing ด้วย OpenTelemetry](part-099-observability-opentelemetry.md) | **Part ถัดไป:** [Deployment: Cloud Platforms (AWS/GCP/Fly.io)](part-101-cloud-deployment.md)
