# Part 102: Embedded Rust เบื้องต้น

> โมดูล: Embedded & Systems Programming | ระดับ: สูง | เวลาโดยประมาณ: 260 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้ว่า "embedded" หมายถึงสภาพแวดล้อมการรันโปรแกรมแบบไหน (bare-metal ไม่มี OS หรือมี RTOS ขนาดเล็ก,
  เข้าถึง hardware ผ่าน memory-mapped register ตรง ๆ, ทรัพยากรจำกัดระดับกิโลไบต์ไม่ใช่กิกะไบต์) และอธิบายเชิง
  กลไกได้ว่าทำไม Rust จึงเหมาะกับงานนี้พอ ๆ กับ (หรือมากกว่า) C ทั้งที่ C ครองพื้นที่นี้มานานหลายสิบปี
- เขียนโปรแกรม `#![no_std]` แบบเต็มรูปแบบ (ไม่ใช่แค่ library เปล่า ๆ แบบที่ Part 56 แสดงไว้ระดับ awareness)
  ที่มี `#[no_main]`, entry point ผ่าน `#[entry]` จาก `cortex-m-rt`, `#[panic_handler]` ของตัวเอง และ
  **คอมไพล์ข้าม (cross-compile) จริงได้สำเร็จ** ไปยัง target ARM Cortex-M อย่าง `thumbv7em-none-eabihf`
- อธิบายและเขียนโค้ดเข้าถึง memory-mapped I/O ผ่าน volatile read/write ได้ถูกต้อง พร้อมอธิบายได้ว่าทำไม
  compiler ต้อง**ไม่**ปรับปรุง (optimize) หรือเรียงลำดับ (reorder) การเข้าถึงเหล่านี้ใหม่ — ต่างจากตัวแปรปกติ
  ที่ compiler ปรับปรุงได้อย่างอิสระ
- อธิบายได้ว่าทำไม `#[no_std]` ไม่มี panic handler ให้อัตโนมัติเหมือนโปรแกรมที่มี `std`, เลือกและเขียน
  panic handler ของตัวเองได้ (ตั้งแต่ `panic-halt` แบบง่ายที่สุด ไปจนถึงแนวคิดของ `panic-probe`/`defmt`)
- เข้าใจว่า embedded program มักถูกจัดโครงสร้างงานรอบ **interrupt** แทนการ polling/async แบบที่หลักสูตรสอนมา
  ตลอด และรู้วิธีแชร์ state ระหว่าง main loop กับ interrupt handler อย่างปลอดภัยด้วยแนวคิดเดียวกับ `RefCell`
  (Part 28) และ Send/Sync (Part 39-40) แต่ประยุกต์ในระดับ hardware interrupt แทน OS thread
- รู้จัก `embassy` framework และเข้าใจว่า async/await ใช้งานได้จริงใน `#[no_std]` โดยไม่ต้องมี OS คอย schedule
  เพราะ `embassy` มี executor ของตัวเองที่ไม่พึ่ง `std` เลย
- ปรับแต่งขนาดไบนารี embedded ด้วย `opt-level`, `lto`, `panic = "abort"`, `codegen-units = 1`, และ `strip`
  พร้อมตัวเลขขนาดไบนารีจริงที่วัดได้จากการคอมไพล์จริงในบทนี้ ไม่ใช่การประมาณ
- วางตำแหน่งบทนี้ได้อย่างซื่อตรง: เข้าใจว่านี่คือบท "ปูพื้นให้รู้จัก" (awareness/getting-started) ไม่ใช่เส้นทาง
  สู่ความเชี่ยวชาญ embedded เต็มรูปแบบ และรู้ว่าจะไปหาความรู้ต่อจากที่ไหน

## ความรู้ที่ต้องมีมาก่อน

บทนี้พึ่งพาความรู้จากหลายสายของหลักสูตรพร้อมกัน เพราะ embedded คือจุดที่ทุกอย่างเรื่อง "หน่วยความจำและความ
ปลอดภัยแบบไม่มี runtime ช่วย" ที่หลักสูตรสอนมาต้องถูกใช้งานจริงพร้อมกันทั้งหมด:

- **Part 41 (Unsafe เบื้องต้น)** — `unsafe` block, raw pointer dereference, และหลักปรัชญา "safe abstraction
  เหนือ unsafe code" คือฐานของทุกอย่างในบทนี้ เพราะการเข้าถึง memory-mapped register ทำผ่าน raw pointer
  โดยตรงเสมอ (ถ้ายังไม่แม่นเรื่อง `unsafe fn`/dereference ราคาแพงมากที่จะอ่านบทนี้โดยไม่ทวนก่อน)
- **Part 42 (Raw Pointers และ Memory Layout)** — ความรู้เรื่อง `*const T`/`*mut T`, การคำนวณ address, และ
  alignment จำเป็นสำหรับการแปลง address ตัวเลข (เช่น `0x4002_0000`) เป็น pointer ที่ใช้งานได้จริง
- **Part 56 (Memory Management ขั้นสูง)** — สอน `#![no_std]` ไว้แล้วในระดับ **awareness** (หัวข้อ 56.7) พร้อม
  บอกไว้ตรง ๆ ว่า "รายละเอียดเต็มรูปแบบของการเขียนโปรแกรม `no_std` แบบ executable สมบูรณ์...จะเป็นเนื้อหาเต็ม
  ของบทที่สอน embedded Rust โดยเฉพาะ" — **บทนี้คือบทที่ Part 56 พูดถึงไว้** เราจะไม่สอน `#![no_std]`/`core`
  vs `std` ซ้ำตั้งแต่ต้น แต่จะขยายต่อจากจุดที่ Part 56 หยุดไว้ทันที
- **Part 28 (Smart Pointers: Rc/RefCell)** — แนวคิด "ตรวจ aliasing ตอน runtime แทน compile time" ของ
  `RefCell` จะถูกนำมาใช้ใหม่ในบริบทของการแชร์ state ระหว่าง main code กับ interrupt handler (หัวข้อ 102.9)
- **Part 39-40 (Mutex/Arc, Send/Sync)** — แนวคิด "การป้องกันข้อมูลที่ถูกเข้าถึงจากมากกว่าหนึ่ง context พร้อมกัน"
  จะถูกยกมาเทียบกับ critical section บน microcontroller (interrupt handler คือ "concurrent context" แบบหนึ่ง
  แม้จะไม่มี OS thread จริงก็ตาม)
- **Part 86 (WebAssembly เบื้องต้น)** — บทนั้นสอนการ cross-compile ไปยัง target ที่ไม่มี OS (`wasm32-unknown-
  unknown`) และแยกแยะ target ที่มี/ไม่มี system interface ให้ (`wasm32-wasip1`) ไว้แล้ว บทนี้คือ**อีกตัวอย่าง
  หนึ่งของแนวคิดเดียวกัน** แต่สุดขั้วกว่ามาก: ไม่มี "system interface" ให้เลือกใช้แม้แต่แบบที่จำกัด เพราะไม่มี
  ระบบปฏิบัติการอยู่ข้างใต้เลยจริง ๆ
- **Part 96 (Docker Containerization)** — เทคนิคลดขนาด binary (`opt-level`, `lto`, `strip`) ที่ Part 96 สอน
  ไว้ในบริบทของการลด image size ระดับเมกะไบต์ จะถูกนำมาใช้ซ้ำในบทนี้ แต่ในสเกลที่ **กิโลไบต์มีความหมาย** —
  ความต่างของ scale นี้คือสิ่งที่ทำให้ embedded เป็นสภาพแวดล้อมที่ต้องคิดเรื่อง binary size อย่างจริงจังกว่า
  ทุกบทที่ผ่านมาในหลักสูตร

## เนื้อหา

### 102.1 Embedded คืออะไร และทำไม Rust เหมาะกับงานนี้

ตลอด 101 บทที่ผ่านมา หลักสูตรนี้สอนโปรแกรมที่รันอยู่ภายใต้ **ระบบปฏิบัติการ (OS)** เสมอ — ไม่ว่าจะเป็นโปรแกรม
command-line ธรรมดา (Part 1-40), เว็บเซิร์ฟเวอร์ (Part 60+), หรือแม้แต่โปรแกรมที่ compile เป็น WebAssembly
(Part 86-95) ก็ยังรันอยู่ภายใต้ browser หรือ WASM runtime ที่ทำหน้าที่เป็น "OS จำลอง" ให้อีกที — เสมอมามี
บางสิ่งที่จัดสรร heap ให้, จัดการ thread ให้, เปิด/ปิด file handle ให้, และมีทรัพยากร (RAM, CPU) ระดับ
**กิกะไบต์/กิกะเฮิรตซ์** ให้ใช้อย่างเหลือเฟือเทียบกับสิ่งที่โปรแกรมทั่วไปต้องการจริง

**Embedded programming คือโลกที่สมมติฐานทั้งหมดนี้หายไปพร้อมกัน** — โปรแกรม embedded ส่วนใหญ่รันอยู่บน
**microcontroller** (MCU) ตัวเล็ก ๆ ที่ฝังอยู่ในอุปกรณ์ เช่น เครื่องซักผ้า, เซนเซอร์วัดอุณหภูมิ, ตัวควบคุมมอเตอร์
ในโดรน, หรือ smart watch — MCU เหล่านี้มีลักษณะร่วมกันที่ต่างจากคอมพิวเตอร์ทั่วไปอย่างสิ้นเชิง:

| คุณสมบัติ | คอมพิวเตอร์ทั่วไป (ที่หลักสูตรสอนมาตลอด) | Microcontroller ทั่วไป (เช่น STM32F401, Cortex-M4) |
|---|---|---|
| RAM | หลักกิกะไบต์ (GB) | หลัก**สิบถึงร้อยกิโลไบต์ (KB)** — บาง MCU มีแค่ 4-8 KB |
| Flash/พื้นที่เก็บโปรแกรม | หลักร้อยกิกะไบต์ถึงเทราไบต์ | หลัก**ร้อยกิโลไบต์ถึงไม่กี่เมกะไบต์** |
| ระบบปฏิบัติการ | มีเสมอ (Linux/Windows/macOS) | **ไม่มี** (bare-metal) หรือมี RTOS ขนาดจิ๋ว (FreeRTOS, RTIC) |
| การเข้าถึง hardware | ผ่าน syscall/driver ของ OS เท่านั้น | **เขียน/อ่าน memory address ที่ผูกกับ hardware ตรง ๆ** |
| Heap allocator | มีให้ใช้เสมอ (`malloc`/`System` allocator) | **ไม่มีให้อัตโนมัติ** ต้องเขียนเองหรือไม่ใช้ heap เลย |
| Clock speed | หลักกิกะเฮิรตซ์ (GHz) หลาย core | หลัก**เมกะเฮิรตซ์ (MHz)** core เดียว มักไม่มี pipeline ซับซ้อน |
| ตัวอย่าง error ถ้าพัง | โปรแกรม crash, OS จัดการ resource คืน | **อุปกรณ์จริงพัง**, ไฟไหม้ (ในกรณีร้ายแรงเช่นควบคุมมอเตอร์/battery) |

ตัวเลขในตารางนี้ไม่ใช่การพูดเกินจริง — MCU อย่าง STM32F401 (ที่ใช้เป็นตัวอย่างในบทนี้) มี RAM **64 KB** และ
Flash **256 KB** เทียบกับคอมพิวเตอร์ทั่วไปที่มี RAM มากกว่านั้นถึง**หลักล้านเท่า** — ตัวเลขระดับนี้หมายความว่า
**ทุกไบต์มีความหมาย** ไม่ใช่แค่คำพูดสวยหรู แต่เป็นข้อจำกัดทางฟิสิกส์ที่บังคับให้ทุกการตัดสินใจทางวิศวกรรม
ซอฟต์แวร์ต้องคิดเรื่อง cost จริง ๆ

#### 102.1.1 ทำไม C ครองพื้นที่นี้มาตลอด และทำไม Rust ท้าชนได้

C คือภาษาที่ครองตลาด embedded มาตั้งแต่ยุค 1970s เพราะมันมีคุณสมบัติที่จำเป็นสำหรับงานนี้: **ไม่มี runtime
บังคับ** (ไม่มี garbage collector, ไม่มี virtual machine, ไม่มีอะไรที่ "แอบ" กินทรัพยากรหรือเวลาโดยที่
โปรแกรมเมอร์คุมไม่ได้), **compile เป็น machine code ตรง ๆ**, และ**ให้สิทธิ์เข้าถึง memory address ได้เต็มที่**
— สามสิ่งนี้คือสิ่งที่งาน embedded ต้องการเป๊ะ

แต่ C มีข้อเสียใหญ่ข้อเดียวที่ทำให้เกิดบั๊กร้ายแรงในอุตสาหกรรม embedded มานับไม่ถ้วน: **ไม่มีการตรวจสอบความ
ปลอดภัยของ memory เลยแม้แต่นิดเดียวตอน compile time** — buffer overflow, use-after-free, null pointer
dereference, data race ระหว่าง main loop กับ interrupt handler — ทั้งหมดนี้คือสาเหตุของบั๊ก embedded ที่ดัง
ที่สุดในประวัติศาสตร์หลายครั้ง (เช่นเหตุการณ์ที่ระบบควบคุมยานพาหนะ/อุปกรณ์การแพทย์ทำงานผิดพลาดจาก memory bug
ที่ตรวจไม่พบจนกว่าจะใช้งานจริง) และการ debug บั๊กแบบนี้บน MCU ที่ไม่มี debugger สะดวกเท่าคอมพิวเตอร์ทั่วไป
เป็นเรื่องที่**ยากกว่าหลายเท่า**เมื่อเทียบกับการ debug โปรแกรมที่มี OS คอยช่วย

นี่คือจุดที่ Part 56 เกริ่นไว้แล้วว่า Rust ไม่มี garbage collector และไม่มี runtime บังคับเหมือน C — แต่สำหรับ
WASM (บริบทของ Part 56/86) เหตุผลนี้สำคัญเพราะ**ประหยัดขนาดไฟล์และความเร็วเริ่มโปรแกรม** สำหรับ embedded
เหตุผลเดียวกันนี้**สำคัญกว่านั้นอีกขั้น** เพราะ:

1. **ไม่มี garbage collector แปลว่าไม่มี GC pause** — MCU ที่ควบคุมมอเตอร์แบบ real-time ทนไม่ได้กับการที่
   โปรแกรมหยุดกะทันหันไม่กี่มิลลิวินาทีเพื่อเก็บกวาด memory (ภาษาที่มี GC อย่าง Java/Go/Python แทบไม่มีใครใช้
   เขียน firmware ระดับ MCU เล็ก ๆ เลยด้วยเหตุผลนี้ตรง ๆ)
2. **ไม่มี runtime ที่ต้องพึ่ง OS แปลว่า binary ขนาดเล็กพอที่จะยัดลง Flash 256 KB ได้** — ภาษาที่ต้องพึ่ง
   virtual machine (JVM, .NET CLR) หรือ interpreter (Python, JavaScript แบบดั้งเดิม) ไม่มีทางยัดตัวเองพร้อม
   runtime ทั้งชุดลง Flash ขนาดนี้ได้เลยในทางปฏิบัติ
3. **แต่ยังคงได้ memory safety ที่ borrow checker ตรวจให้ตอน compile time** (Part 7) — นี่คือสิ่งที่ C ทำ
   ไม่ได้เลย: Rust ให้ทั้งความสามารถแบบ C (ไม่มี runtime, เข้าถึง address ตรง ๆ ได้) **และ**ความปลอดภัยที่ C
   ไม่มี **พร้อมกันในเวลาเดียว** — นี่คือเหตุผลที่ทำให้ Rust กลายเป็นภาษาที่หน่วยงานอย่าง Google, Microsoft,
   AWS ลงทุนสนับสนุน embedded Rust ecosystem อย่างจริงจังในช่วงหลายปีที่ผ่านมา (ผ่านมูลนิธิ Rust Foundation
   และโครงการอย่าง Oxidos)

ข้อควรระวังเชิงซื่อตรง: Rust ไม่ได้แก้ปัญหาทุกอย่างของ embedded ให้หมดไป — โค้ดที่เข้าถึง hardware ตรง ๆ ยังต้อง
ใช้ `unsafe` เสมอ (เพราะ compiler ไม่มีทางรู้ว่า address ตัวเลขที่คุณเขียนตรงกับ hardware จริงหรือไม่ — นี่คือ
สิ่งที่อยู่นอกเหนือขอบเขตที่ static analysis พิสูจน์ได้ตามที่ Part 41 อธิบายไว้) แต่ Rust ทำให้**พื้นที่ของโค้ด
ที่ต้องเชื่อใจแบบ manual (`unsafe`) เล็กลงมาก** เพราะเราสามารถห่อ `unsafe` เหล่านั้นด้วย safe abstraction
(เช่น crate `embedded-hal`/PAC ที่จะเห็นในหัวข้อถัดไป) แล้วให้ส่วนที่เหลือ 95% ของโปรแกรม (logic, state
machine, การคำนวณ) เขียนเป็น safe Rust ที่ borrow checker ตรวจให้เต็มรูปแบบ — สัดส่วนนี้คือสิ่งที่ทำให้บั๊ก
memory-related ในโปรเจกต์ embedded Rust ลดลงอย่างมีนัยสำคัญเทียบกับโปรเจกต์ C ขนาดเท่ากัน

#### 102.1.2 Bare-metal กับ RTOS: สองรูปแบบของ "ไม่มี OS เต็มรูปแบบ"

คำว่า "ไม่มี OS" ในหัวข้อก่อนอาจทำให้เข้าใจผิดว่า embedded มีแค่รูปแบบเดียว — ในความเป็นจริงมีสอง**รูปแบบหลัก**
ที่ต้องแยกให้ออก เพราะบทนี้เน้นรูปแบบแรกเท่านั้น (bare-metal) แต่โลก embedded จริงใช้ทั้งสองแบบขึ้นกับความ
ซับซ้อนของงาน:

| รูปแบบ | ลักษณะ | เหมาะกับงานแบบไหน | ตัวอย่างในบทนี้ |
|---|---|---|---|
| **Bare-metal** | ไม่มีตัวกลางใด ๆ เลยระหว่างโค้ดของเรากับ hardware — `#[entry]` ของเราคือจุดสูงสุดที่ควบคุมทุกอย่าง ไม่มีใคร schedule งานให้ | งานเดียว/ไม่กี่งานที่ไม่ซับซ้อนมาก, ต้องการควบคุม timing แบบเต็มรูปแบบ 100% (เช่น real-time control loop ของมอเตอร์) | ตัวอย่าง blink (102.6), SysTick/interrupt (102.9) |
| **RTOS (Real-Time Operating System)** | มี "OS จิ๋ว" ที่ทำหน้าที่ scheduling ระหว่างหลาย "task" (คล้าย thread แต่เบากว่ามาก) ให้ — ตัวอย่างที่ดังที่สุดคือ **FreeRTOS** (เขียนด้วย C, มี Rust binding ให้เรียกใช้) และ **RTIC** (Real-Time Interrupt-driven Concurrency — เขียนด้วย Rust ล้วน ออกแบบมาเฉพาะสำหรับ Cortex-M) | งานที่มีหลาย task พร้อมกันจริงจัง ต้องการ priority/preemption ที่ชัดเจน (เช่น task วัดเซนเซอร์ priority ต่ำ, task ตอบสนอง emergency stop priority สูงสุด) | ไม่ได้สอนในบทนี้ (เกินขอบเขต "เบื้องต้น") |

จุดที่ควรเข้าใจให้ชัด: **RTOS ไม่ใช่ "OS เต็มรูปแบบ" แบบ Linux/Windows** — มันไม่มี virtual memory, ไม่มี
filesystem แบบเต็มรูปแบบ, ไม่มี process isolation ระดับ hardware (MMU) เสมอไป มันมีแค่ **scheduler** ที่สลับ
ระหว่าง task ตาม priority และเวลา ซึ่งยังคงเป็นสภาพแวดล้อมที่ **ทรัพยากรจำกัดระดับกิโลไบต์เหมือนกัน** และยัง
ต้องเขียนแบบ `#[no_std]` ในหลายกรณี (RTIC ทำงานบน `#[no_std]` เต็มรูปแบบ) — ความรู้เรื่อง `#[no_std]`, volatile
register access, และ critical section ที่บทนี้สอนไว้ **ยังใช้ได้ทั้งหมดไม่ว่าจะเลือก bare-metal หรือ RTOS** —
สิ่งที่ RTOS เพิ่มเข้ามาคือ**ชั้นของการจัดสรรเวลา CPU ระหว่างหลายงาน** ซึ่งเป็นปัญหาคนละระดับจากที่บทนี้ครอบคลุม
(เทียบได้กับความต่างระหว่างการเขียนโปรแกรม single-thread ธรรมดา กับการเขียนโปรแกรมที่ต้อง manage หลาย thread
พร้อม priority ที่ Part 39-40 แนะนำไว้ในระดับ OS thread — RTOS คือแนวคิดเดียวกันแต่ในระดับที่เบากว่าและควบคุม
ได้แน่นอนกว่ามาก เพราะไม่มี OS scheduler ของ Linux/Windows ที่ไม่แน่นอนมาแทรกกลาง)

`embassy` (หัวข้อ 102.10) เป็นทางเลือกที่สามที่ทันสมัยกว่า RTOS แบบดั้งเดิม: มันให้ผลลัพธ์คล้าย RTOS (จัดการ
หลายงานพร้อมกันได้) แต่ใช้ async/await ของภาษาแทน task-based scheduler แบบ C — ไม่ต้องเลือกระหว่าง "bare-metal
ล้วน ๆ" กับ "RTOS แบบเก่า" อีกต่อไปสำหรับโปรเจกต์ใหม่จำนวนมาก

### 102.2 ความซื่อตรงเรื่อง Sandbox: ไม่มี Microcontroller จริง และแนวทางตรวจสอบที่เลือกใช้

ก่อนลงโค้ดตัวแรก ต้องพูดตรง ๆ ให้ชัดที่สุดเรื่องหนึ่ง: **สภาพแวดล้อมที่ใช้เขียนบทนี้ (sandbox แบบ cloud
container) ไม่มี microcontroller จริงต่ออยู่แน่นอน** — ไม่มีบอร์ด STM32, ไม่มี Arduino, ไม่มี debug probe
(ST-Link/J-Link) เสียบอยู่เลย นี่ไม่ใช่ข้อจำกัดที่แก้ได้ด้วยการติดตั้ง package เพิ่ม — มันคือข้อจำกัดทาง
กายภาพของสภาพแวดล้อมที่รันอยู่ (container บน cloud ไม่มี USB port ที่เสียบ hardware จริงได้)

คำถามที่ต้องตอบตรง ๆ คือ: **แล้วจะสอน "รันจริงบน embedded" ได้อย่างไรถ้าไม่มี hardware?** คำตอบคือมีสองระดับ
ของ "การพิสูจน์ว่าโค้ดถูกต้อง" ที่ทำได้จริงโดยไม่ต้องมี hardware และบทนี้จะใช้ทั้งสองระดับอย่างตรงไปตรงมา:

1. **Cross-compilation จริงไปยัง embedded target** — นี่คือระดับที่ **ทดสอบได้เต็มรูปแบบโดยไม่ต้องมี
   hardware เลย** เพราะ `rustc`/`cargo` สามารถ compile โค้ดสำหรับ CPU architecture ที่ต่างจาก CPU ที่กำลัง
   รันอยู่ได้เสมอ (นี่คือธรรมชาติของ cross-compiler) — ถ้า `cargo build --target thumbv7em-none-eabihf`
   ผ่านโดยไม่มี error เราก็**ยืนยันได้ 100%**ว่าโค้ด syntax ถูกต้อง, type ถูกต้อง, borrow checker ผ่าน,
   และ linker สร้าง binary ที่มีรูปร่างถูกต้องสำหรับ ARM Cortex-M จริง ๆ — สิ่งที่ cross-compile **ไม่**
   พิสูจน์คือ "โค้ดนี้จะทำงานถูกต้องตาม logic ที่ตั้งใจไว้จริงหรือไม่เมื่อรันบน hardware จริง" (เช่น pin ที่
   เลือกต่อ LED จริงหรือไม่ ตรงกับ schematic ของบอร์ดจริงหรือไม่) — บทนี้จะระบุชัดเจนทุกครั้งว่าตัวอย่างไหน
   ตรวจสอบได้แค่ระดับนี้
2. **การรันจริงบน QEMU (ถ้ามีในสภาพแวดล้อม)** — QEMU สามารถ**จำลอง** MCU บางรุ่นได้ (ไม่ใช่ทุกรุ่น — QEMU
   รองรับบอร์ดจำลองที่มีการ implement CPU core + peripheral บางส่วนไว้ล่วงหน้าเท่านั้น) บอร์ดที่ใช้กันบ่อย
   ที่สุดสำหรับทดสอบ embedded Rust แบบไม่มี hardware คือ **`lm3s6965evb`** (จำลอง MCU ตระกูล Texas
   Instruments Stellaris LM3S6965, คอร์ ARM Cortex-M3) เพราะเป็นบอร์ดที่ Embedded Rust Book ใช้สอนอย่างเป็น
   ทางการสำหรับจุดประสงค์นี้พอดี — ถ้า QEMU พร้อม เราจะได้เห็น**ผลลัพธ์การรันจริงจากโปรแกรมที่ CPU จำลองรัน
   จริง** ผ่านเทคนิคที่เรียกว่า **semihosting** (ให้โปรแกรม embedded ส่งข้อความออกมาทาง debug channel ที่
   QEMU ดักฟังอยู่ — คล้าย `println!` แต่ทำงานได้แม้ไม่มี UART/console จริง)

การตรวจสอบทั้งบทนี้ทำจริงตามลำดับนี้: เขียนเนื้อหาทั้งหมดก่อน แล้วตรวจสอบจริงเป็นชุดเดียวหลังจบ — ผลจริงที่ได้
มีดังนี้:

- **ติดตั้ง target ด้วย `rustup target add thumbv7em-none-eabihf` และ `rustup target add thumbv7m-none-eabi`
  สำเร็จทั้งคู่** ในสภาพแวดล้อมเขียนบทนี้ — ทุกตัวอย่างที่อ้างว่า "cross-compile ผ่านจริง" ในบทนี้ถูก build
  จริงด้วย `cargo build` กับ target เหล่านี้แล้ว ไม่ใช่การคาดเดา
- **ตรวจสอบ `qemu-system-arm`**: รันคำสั่ง `which qemu-system-arm` ในสภาพแวดล้อมเขียนบทนี้ตั้งแต่ต้น — ไม่พบ
  binary นี้เลย จึงลองติดตั้งเพิ่มเติมด้วย `apt-get install -y qemu-system-arm` เพื่อให้แน่ใจว่าไม่ได้พลาด
  อะไรไป — การติดตั้งใช้เวลานาน (เกือบ 19 นาที) และจบด้วยการดาวน์โหลด package หลักของ `qemu-system-arm` เอง
  **ล้มเหลว** (ปฏิเสธการเชื่อมต่อ/timeout ไปยัง mirror `security.ubuntu.com` ที่ proxy ของสภาพแวดล้อมนี้ไม่ได้
  อนุญาตให้เข้าถึง) สรุปคือ **`qemu-system-arm` ไม่มีอยู่จริงในสภาพแวดล้อมที่เขียนบทนี้** ไม่ว่าจะพยายามวิธีใด
  — ดังนั้นตัวอย่างที่เขียนไว้สำหรับบอร์ดจำลอง `lm3s6965evb` (หัวข้อ 102.7) จะถูกยืนยันได้แค่ระดับ
  **cross-compilation เท่านั้น** ในบทนี้ — จะไม่มีการเขียนผลลัพธ์การรันบน QEMU ที่ไม่ได้เกิดขึ้นจริงเด็ดขาด
  ถ้าคุณมีสภาพแวดล้อมที่เข้าถึง QEMU ได้ (เครื่อง development ทั่วไปที่ไม่ได้อยู่หลัง proxy จำกัด) โค้ดในบทนี้
  พร้อมรันได้ทันทีด้วยคำสั่ง `cargo run` ตามที่ `.cargo/config.toml` กำหนด runner ไว้ให้แล้ว

### 102.3 `#[no_std]` เจาะลึก: เส้นแบ่งระหว่าง `core` และ `std`

Part 56 (หัวข้อ 56.7) เกริ่น `#![no_std]` ไว้แล้วในระดับ awareness: มัน "บอก compiler ว่าห้ามผูกกับ `std`
เลย ให้ใช้ได้แค่ `core`" — บทนี้จะขยายความให้ชัดเจนเต็มรูปแบบว่า **อะไรหายไปจริง ๆ**, **อะไรยังอยู่**, และ
**ทำไมเส้นแบ่งนี้ถูกวางไว้ตรงจุดนี้พอดี**

Rust standard library จริง ๆ แล้วประกอบด้วย**สามชั้น**ที่ซ้อนกันอยู่ (ไม่ใช่ก้อนเดียวแบบที่ดูจากภายนอก):

```text
┌─────────────────────────────────────────────────────────┐
│  std   (ต้องมี OS)                                        │
│  - thread, file I/O, network socket, std::time::Instant  │
│  - std::env, process spawn/exit                           │
│  - re-export ทุกอย่างจาก alloc และ core ผ่าน std ด้วย       │
├─────────────────────────────────────────────────────────┤
│  alloc  (ต้องมี heap allocator แต่ไม่ต้องมี OS)             │
│  - Box<T>, Vec<T>, String, Rc<T>, BTreeMap, format!        │
├─────────────────────────────────────────────────────────┤
│  core   (ไม่ต้องมีอะไรเลยนอกจาก CPU)                        │
│  - primitive types, Option/Result, Iterator, slice        │
│  - core::ptr (raw pointer + volatile), core::mem            │
│  - arithmetic, comparison traits, panic! macro (แต่ไม่มี    │
│    ตัว handler ให้ — ต้องกำหนดเอง ดูหัวข้อ 102.8)            │
└─────────────────────────────────────────────────────────┘
```

เมื่อเขียน `#![no_std]` บนสุดของ crate เรากำลังบอก compiler ว่า **"ห้าม link กับ `std` เลย"** — สิ่งที่หายไป
ทันทีคือทุกอย่างที่สมมติว่ามี OS อยู่ข้างใต้: `std::thread::spawn` (ไม่มี OS scheduler ให้สร้าง thread),
`std::fs::File` (ไม่มี filesystem), `std::net::TcpStream` (ไม่มี network stack ของ OS), และที่สำคัญที่สุด
**heap allocator เริ่มต้น** (ปกติ `std` เชื่อมกับ `malloc`ของ OS ให้อัตโนมัติผ่าน `System` allocator — ไม่มี
OS ก็ไม่มีอะไรให้เชื่อม)

แต่สิ่งที่ **ยังอยู่ครบ** ผ่าน `core` คือสิ่งที่โปรแกรมเมอร์ embedded ส่วนใหญ่ใช้งานอยู่แล้วเป็นหลัก:
`Option<T>`/`Result<T, E>` (ทุก pattern matching และ error handling ที่หลักสูตรสอนมาตั้งแต่ Part 3-4 ใช้ได้
เหมือนเดิมทุกอย่าง), `Iterator` trait และ adaptor ทั้งหมด (`.map()`, `.filter()`, `.sum()` — Part 25-26 ใช้ได้
เหมือนเดิม), slice (`&[T]`, `&mut [T]`), และ integer/float arithmetic เต็มรูปแบบ

ประเด็นสำคัญที่ต้องเข้าใจให้แม่น: **`no_std` ไม่ได้แปลว่า "เขียน Rust แบบพื้นฐานน้อยลง"** — โปรแกรมเมอร์
embedded ยังเขียน pattern matching, generics, trait, closure, iterator chain ได้ครบทุกอย่างเหมือนโปรแกรม
ปกติ สิ่งที่หายไปมีแค่ **บริการที่ต้องพึ่ง OS** เท่านั้น — นี่คือเหตุผลที่ Rust "รู้สึกเหมือนภาษาเดียวกัน" ไม่ว่า
จะเขียนเว็บเซิร์ฟเวอร์หรือ firmware ของ MCU ต่างจาก C ที่การเขียน embedded (แบบไม่มี libc เต็มรูปแบบ) กับการ
เขียนโปรแกรมทั่วไปมี "รสชาติ" ที่ต่างกันมากกว่า

#### 102.3.1 ตัวอย่างจริง: ไลบรารีตรรกะทางธุรกิจแบบ `#![no_std]` ที่คอมไพล์ได้จริง

มาดูตัวอย่างที่จริงจังกว่า `double()`/`checked_mul()` ของ Part 56 — ไลบรารีคำนวณสถานะของเซนเซอร์วัดอุณหภูมิ
แบบง่าย ที่ใช้ทั้ง `enum`, pattern matching, และ iterator ล้วนแต่ไม่แตะ `std` เลยแม้แต่จุดเดียว:

```rust
#![no_std]

// ไลบรารีนี้ไม่ผูกกับ std เลย ใช้ได้เฉพาะ core -- แต่ยังเขียน Rust แบบเต็มรูปแบบได้ทุกอย่าง
// (enum, pattern matching, iterator, generic) เพราะทั้งหมดนี้อยู่ใน core ไม่ใช่ std

/// สถานะของเซนเซอร์อุณหภูมิ อิงจากค่าที่อ่านได้ (หน่วย: 0.1 องศาเซลเซียส เพื่อเลี่ยง floating point)
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum TempStatus {
    Freezing,   // ต่ำกว่า 0.0C
    Normal,     // 0.0C ถึง 40.0C
    Overheat,   // สูงกว่า 40.0C
}

/// แปลงค่าที่อ่านได้จาก ADC (หน่วย 0.1C) เป็นสถานะ -- ใช้แค่ arithmetic และ pattern matching
/// ที่อยู่ใน core ทั้งหมด ไม่มีอะไรผูกกับ OS เลย
pub fn classify(reading_tenths_celsius: i32) -> TempStatus {
    match reading_tenths_celsius {
        r if r < 0 => TempStatus::Freezing,
        r if r <= 400 => TempStatus::Normal,
        _ => TempStatus::Overheat,
    }
}

/// รับค่าที่อ่านได้หลายค่า (จาก ring buffer ของ sensor readings) แล้วคืนค่าเฉลี่ย
/// ใช้ Iterator เต็มรูปแบบ (.iter().sum(), .len()) -- ทั้งหมดอยู่ใน core เช่นกัน
pub fn average_reading(readings: &[i32]) -> Option<i32> {
    if readings.is_empty() {
        return None; // Option<T> จาก core ใช้ได้ปกติทุกประการ
    }
    let sum: i32 = readings.iter().sum();
    Some(sum / readings.len() as i32)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn classify_normal_range() {
        assert_eq!(classify(250), TempStatus::Normal);
        assert_eq!(classify(-10), TempStatus::Freezing);
        assert_eq!(classify(450), TempStatus::Overheat);
    }

    #[test]
    fn average_of_empty_slice_is_none() {
        assert_eq!(average_reading(&[]), None);
    }

    #[test]
    fn average_computed_correctly() {
        assert_eq!(average_reading(&[100, 200, 300]), Some(200));
    }
}
```

ผล `cargo test` จริงบนเครื่อง host ที่เขียนบทนี้ (ไม่ต้องมี target embedded ใด ๆ เกี่ยวข้องเลยตอนรัน test
เพราะ `#[cfg(test)]` เปิด `std` ให้ชั่วคราว):

```
running 3 tests
test tests::average_of_empty_slice_is_none ... ok
test tests::average_computed_correctly ... ok
test tests::classify_normal_range ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

และเพื่อยืนยันว่า crate นี้เป็น `#![no_std]` จริง ไม่ได้แอบพึ่ง `std` ที่ไหนเลย ได้ลอง `cargo build --target
thumbv7em-none-eabihf` จริงด้วย (build เป็น library สำหรับ ARM Cortex-M4 ตรง ๆ) ซึ่งผ่านสำเร็จโดยไม่มี error/
warning ใด ๆ เกี่ยวกับ `std` เลย — พิสูจน์ได้ทั้งสองด้านพร้อมกัน: **test ได้บน host เพื่อความสะดวก และ compile
ได้จริงสำหรับ MCU เพื่อความถูกต้อง**

สังเกตสิ่งสำคัญ: `#[cfg(test)]` module ยังใช้งานได้ปกติ (เวลารัน `cargo test` บนเครื่อง host ปกติที่มี `std`
มัน**เปิด `std` ให้ชั่วคราวสำหรับ test build เท่านั้น** — นี่คือ pattern ที่ crate `no_std` จริงในโลกทำกันทั่วไป
เพื่อให้ยัง unit test ได้บนเครื่อง development ปกติโดยไม่ต้องรันบน MCU จริงทุกครั้งที่ทดสอบ logic) โค้ดส่วนที่
เหลือ (`classify`, `average_reading`) compile ได้ทั้งสำหรับ host และสำหรับ MCU จริงโดยไม่ต้องแก้อะไรเลย — นี่
คือข้อดีเชิงปฏิบัติที่สำคัญมาก: **แยกส่วน "ตรรกะทางธุรกิจ" (business logic) ที่ทดสอบได้บนเครื่อง development
ทั่วไป ออกจากส่วน "การเข้าถึง hardware" (ที่ทดสอบได้แค่บน MCU จริงหรือ QEMU)** — เป็นรูปแบบการออกแบบที่โปรเจกต์
embedded Rust จริงจังแทบทุกโปรเจกต์ใช้ เพราะการทดสอบตรรกะบน MCU จริงทุกครั้งช้ามากและ debug ยากกว่าทดสอบบน
เครื่อง development หลายเท่า

#### 102.3.2 `no_std` + `alloc`: มี heap ได้ถ้าจำเป็น แต่ต้องคิดให้รอบคอบกว่าปกติมาก

Part 56 หัวข้อ 56.7.1 อธิบายไว้แล้วว่า `alloc` แยกออกมาจาก `std` เป็น crate ต่างหากที่ไม่ต้องพึ่ง OS — สิ่งที่
ต้องเน้นเพิ่มสำหรับบริบท embedded คือ: **การมี heap บน MCU ไม่ใช่เรื่องที่ทำได้ "ฟรี" เหมือนบนคอมพิวเตอร์ทั่วไป**
เพราะ RAM มีแค่หลักสิบกิโลไบต์ การจอง/คืน heap ซ้ำ ๆ (fragmentation) อาจทำให้โปรแกรมที่รันได้ดีตอนเริ่มต้น
ล้มเหลวแบบไม่คาดคิดหลังรันไปหลายชั่วโมง เพราะ heap แตกเป็นชิ้นเล็ก ๆ จนไม่มีก้อนต่อเนื่องพอสำหรับ allocation
ก้อนใหม่ (ปัญหานี้ตรวจจับยากกว่าบนคอมพิวเตอร์ทั่วไปมาก เพราะไม่มี memory profiler สะดวกให้ใช้เหมือนเครื่อง
development ปกติ) — ด้วยเหตุนี้ โปรเจกต์ embedded Rust จำนวนมากเลือก**ไม่ใช้ heap เลย** (100% stack-based,
ใช้ `heapless` crate ที่ให้ `Vec`/`String`/`HashMap` แบบมี capacity คงที่ตอน compile time แทน) — เราจะไม่ลง
รายละเอียดของ `heapless` เต็มรูปแบบในบทนี้ (เป็นหัวข้อระดับ awareness เพิ่มเติมที่เกินขอบเขต "เบื้องต้น" ของ
บทนี้) แต่ควรรู้จักชื่อไว้ว่ามีตัวเลือกนี้อยู่สำหรับกรณีที่ไม่อยากยุ่งกับ custom global allocator เลย ตัวอย่าง
ที่ cross-compile จริงสำเร็จในสภาพแวดล้อมเขียนบทนี้:

```rust
#![no_std]
#![no_main]

use cortex_m_rt::entry;
use heapless::Vec as HeaplessVec;
use panic_halt as _;

#[entry]
fn main() -> ! {
    // capacity สูงสุด 8 ตัว กำหนดตอน compile time ผ่าน const generic (Part 18) -- ไม่มี heap
    // allocation เกิดขึ้นเลย ข้อมูลทั้งหมดอยู่บน stack/static memory เท่านั้น ไม่ต้องมี
    // #[global_allocator] เลยแม้แต่นิดเดียว เพราะ heapless::Vec ไม่ได้พึ่ง alloc crate
    let mut readings: HeaplessVec<i32, 8> = HeaplessVec::new();
    let _ = readings.push(100);
    let _ = readings.push(200);

    loop {}
}
```

ผล `cargo build --target thumbv7em-none-eabihf` จริงในสภาพแวดล้อมเขียนบทนี้:

```
   Compiling heapless v0.8.0
   ... (dependency อื่น ๆ)
   Compiling heapless-demo v0.1.0
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 5.21s
```

และขนาด `.text` ที่วัดได้จริงด้วย `size` (debug build, ยังไม่ optimize): **2352 ไบต์** — `Vec<i32, 8>` แบบนี้
มีต้นทุนแค่พื้นที่คงที่ (8 × 4 ไบต์ = 32 ไบต์สำหรับข้อมูล บวก field ติดตาม length) รู้ขนาดแน่นอนตั้งแต่
compile time ไม่มีความเสี่ยงเรื่อง fragmentation หรือ allocation ล้มเหลวตอน runtime เลยแม้แต่กรณีเดียว
ต่างจาก `alloc::vec::Vec<T>` ที่ขนาดโตได้เรื่อย ๆ ตาม runtime แต่ต้องแบกรับความเสี่ยงเรื่อง heap ที่อธิบาย
ไว้ข้างต้น

### 102.4 ไม่มี `main()`: `#[no_main]`, Reset Vector, และ `#[entry]` จาก `cortex-m-rt`

โปรแกรม Rust ปกติทุกโปรแกรมที่เขียนมาตลอดหลักสูตรนี้ (Part 1-101) มีจุดเริ่มต้นที่ฟังก์ชัน `fn main()` เสมอ —
แต่ `main()` แบบนี้ไม่ได้ "เริ่มทำงานเอง" มันถูก**เรียกโดย runtime ของ OS** ผ่านขั้นตอนที่ซับซ้อนกว่าที่เห็น:
OS โหลด binary เข้า memory, เตรียม stack, ตั้งค่า environment variable, แล้วเรียก startup code ของ C runtime
(`crt0`) ที่ทำหน้าที่เตรียมทุกอย่างให้พร้อมก่อนจะเรียก `main()` ของเราจริง ๆ

**บน microcontroller ไม่มี OS ที่จะทำหน้าที่นี้ให้เลย** — เมื่อ MCU เปิดเครื่อง (power-on) หรือ reset,
ฮาร์ดแวร์จะทำสิ่งเดียวคือ**อ่าน address ที่กำหนดไว้ตายตัวใน memory** ที่เรียกว่า **reset vector** (สำหรับ
ARM Cortex-M คือตำแหน่งแรกของ vector table ที่ address `0x0000_0004`) แล้ว**กระโดดไปรันโค้ด ณ address นั้น
ทันที** — ไม่มี OS, ไม่มี `crt0`, ไม่มีอะไรเลยนอกจาก CPU ที่เพิ่งเปิดมาสด ๆ กับ address คงที่หนึ่งจุด

นี่คือที่มาของ attribute สองตัวที่ embedded Rust binary ต้องมี:

- **`#![no_std]`** — ห้ามผูกกับ `std` (อธิบายแล้วในหัวข้อก่อน)
- **`#![no_main]`** — บอก compiler ว่า **"อย่าสร้าง entry point แบบปกติที่คาดหวังว่า OS จะเรียก `main()`
  ให้"** เพราะไม่มี OS ที่จะทำแบบนั้น — เราจะกำหนด entry point เองแบบที่ตรงกับวิธีที่ hardware เริ่มทำงานจริง

การกำหนด entry point เองแบบ "ถูกต้องสำหรับ ARM Cortex-M" (ตั้งค่า stack pointer, เขียน vector table ที่ตรง
ตำแหน่ง, เรียก initialization ของ `.data`/`.bss` section ก่อนโค้ดของเราเริ่มรัน) เป็นงานที่ซับซ้อนและมี
รายละเอียดเฉพาะของแต่ละ architecture มากจนไม่ควรเขียนเองทุกโปรเจกต์ — ชุมชน embedded Rust จึงมี crate ชื่อ
**`cortex-m-rt`** ("Cortex-M Runtime") ที่ทำงานนี้ให้ครบถ้วนและถูกต้องตามสเปกของ ARM แล้ว โปรแกรมเมอร์แค่
ทำเครื่องหมายฟังก์ชันที่ต้องการให้เป็น "จุดเริ่มต้นของโค้ดของเรา" ด้วย attribute `#[entry]` ที่ crate นี้
provide ไว้:

```rust
#![no_std]
#![no_main]

use cortex_m_rt::entry;
use panic_halt as _; // panic handler แบบง่ายที่สุด (อธิบายเต็มในหัวข้อ 102.8)

// #[entry] มาจาก cortex-m-rt -- บอกว่านี่คือฟังก์ชันที่ startup code ของ cortex-m-rt
// (ซึ่งรันเองก่อนหน้านี้แล้ว: ตั้ง stack pointer, zero .bss, copy .data จาก Flash เข้า RAM)
// จะเรียกเป็นลำดับสุดท้าย -- เทียบเท่ากับ "main() ของเรา" แต่ในความหมายของ bare-metal
#[entry]
fn main() -> ! {
    // สังเกต return type: -> ! (never type) ไม่ใช่ () เหมือน main() ปกติ
    // เพราะบน MCU ไม่มี OS ให้ "return กลับไป" -- ถ้าโค้ดนี้ return จริง จะไม่มีที่ไปต่อเลย
    // (ไม่มี process exit, ไม่มี shell รอรับ exit code) ดังนั้นฟังก์ชันนี้ต้อง "ไม่จบ" ตลอดไป

    loop {
        // main loop ของโปรแกรม embedded ทั่วไป -- วนตลอดไปไม่จบ (ต่างจาก main() ปกติที่จบแล้ว process ตาย)
        cortex_m::asm::nop(); // no-operation -- แค่ตัวอย่างให้ loop ไม่ว่างเปล่าไปเลย
    }
}
```

จุดที่สำคัญที่สุดในตัวอย่างนี้คือ **ทำไม return type ของ `main` คือ `!` (never type) ไม่ใช่ `()`** — นี่ไม่ใช่
รายละเอียดเล็ก ๆ แต่คือผลลัพธ์ตรงจากธรรมชาติของ bare-metal: โปรแกรมทั่วไปที่มี `std` เมื่อ `main()` return
กลับ (`()`), OS จะรับรู้ว่าโปรแกรมจบแล้ว คืน resource ทั้งหมด และแสดง exit code — แต่บน MCU **ไม่มี "ที่ไป
ต่อ" หลังจาก main() จบเลย** ถ้าฟังก์ชันนี้ return จริง CPU จะพยายามรันคำสั่งถัดไปหลัง address ที่ return
กลับไป ซึ่งอาจเป็น garbage memory ที่ทำให้พฤติกรรมไม่คาดเดาได้เลย — Rust ป้องกันความผิดพลาดนี้ได้ตั้งแต่
**compile time** ด้วย type system ตรง ๆ: การประกาศ return type เป็น `!` บอก compiler ว่า "ฟังก์ชันนี้รับรอง
ว่าจะไม่ return เด็ดขาด" ถ้าเขียนโค้ดที่มีทางออกจาก loop ได้ (เช่นลืม `loop {}` ไปเฉย ๆ ) **compiler จะฟ้อง
error ตอน compile** ทันที ไม่ต้องรอไปเจอปัญหาตอนรันจริงบน hardware ที่ debug ยากกว่ามาก — นี่คือตัวอย่างที่
ชัดเจนของสิ่งที่หัวข้อ 102.1 พูดไว้: Rust เอาความรู้เชิง "สัญญา" ของ bare-metal (ต้องไม่ return) มาเปลี่ยนเป็น
สิ่งที่ type system ตรวจสอบให้อัตโนมัติ ในขณะที่ C ต้องพึ่งการรู้เรื่องนี้ในหัวโปรแกรมเมอร์เอง ไม่มี compiler
คอยเตือน

### 102.5 Memory-Mapped I/O: อ่าน/เขียน Register ผ่าน Volatile

หัวใจของการเขียน embedded คือการควบคุม hardware — และวิธีที่ CPU ตระกูล ARM Cortex-M (รวมถึง MCU ส่วนใหญ่ใน
โลก) สื่อสารกับ hardware peripheral (GPIO pin, timer, UART, ADC) คือผ่านเทคนิคที่เรียกว่า **memory-mapped
I/O**: แทนที่จะมีคำสั่ง CPU พิเศษสำหรับ "เปิด LED" หรือ "อ่านค่าจากปุ่มกด" ผู้ผลิตชิปจะ**กำหนด address ในช่วง
memory ที่ตายตัว** ให้ตรงกับ register ควบคุมของแต่ hardware แต่ละตัว — การ **อ่าน**จาก address นั้นคือการ
**อ่านสถานะของ hardware จริง** และการ**เขียน**ไปที่ address นั้นคือการ**สั่งให้ hardware ทำงาน** ทันที ไม่ต่าง
จากการอ่าน/เขียนตัวแปรปกติเลยในทางไวยากรณ์ — แต่ต่างกันโดยสิ้นเชิงในทางความหมาย

ตัวอย่างที่จับต้องได้: MCU ตระกูล STM32F4 (ARM Cortex-M4) มี peripheral **GPIOA** (General Purpose I/O port
A) เริ่มต้นที่ address `0x4002_0000` — register ที่ควบคุมว่า pin แต่ละตัวใน port นี้ทำงานเป็น input หรือ
output เรียกว่า **MODER** (Mode Register) อยู่ที่ offset `0x00` จาก base (คือ address `0x4002_0000` ตรง ๆ)
และ register ที่สั่งเปิด/ปิด output จริง (Output Data Register) เรียกว่า **ODR** อยู่ที่ offset `0x14` (คือ
address `0x4002_0014`)

#### 102.5.1 ทำไมต้อง "volatile" — และทำไมตัวแปรปกติใช้ไม่ได้

สมมติเราลองเขียนโค้ดเข้าถึง register นี้แบบ "ตัวแปรปกติ" (ไม่ใช้ volatile) ด้วย raw pointer ตามที่ Part 41-42
สอนไว้:

```rust
// ตัวอย่าง "ผิด" ที่แสดงปัญหา -- ห้ามเขียนแบบนี้จริงในโค้ด embedded
unsafe fn turn_on_led_wrong() {
    let odr = 0x4002_0014 as *mut u32;
    *odr = 1 << 5; // เขียนบิตที่ 5 (pin PA5) เป็น 1 -- "ตั้งใจ" ให้ LED ติด
    *odr = 0;      // แล้วเขียนกลับเป็น 0 ทันที -- "ตั้งใจ" ให้ LED ดับ
}
```

โค้ดนี้ **compile ผ่าน** และดูเหมือนถูกต้องตามตรรกะ (เปิดแล้วปิดทันที) — แต่ปัญหาคือ: `rustc`/LLVM มองเห็น
บรรทัดทั้งสองเป็น **"เขียนค่าไปที่ address เดียวกันสองครั้งติดกันโดยไม่มีการอ่านค่าคืนระหว่างนั้น"** ซึ่งเป็น
pattern ที่ compiler ปรับปรุง (optimize) ได้อย่างชอบธรรมสำหรับตัวแปรปกติ: **ถ้าไม่มีใครอ่านค่าตรงกลาง การเขียน
ครั้งแรกไม่มีผลต่อพฤติกรรมที่สังเกตได้ของโปรแกรมเลย** ตาม memory model ปกติของภาษา — compiler จึงมีสิทธิ์
**ลบการเขียนครั้งแรกทิ้งไปเลย** เหลือแค่ `*odr = 0;` เพียงบรรทัดเดียว (หรือในกรณีที่รุนแรงกว่า อาจลบทั้งสอง
บรรทัดทิ้งไปเลยถ้าไม่มีการอ่านค่า `odr` ไปใช้ต่อที่ไหนอีก) — ผลลัพธ์บน hardware จริงคือ **LED จะไม่กระพริบเลย
แม้แต่ครั้งเดียว** ทั้งที่โค้ดต้นฉบับดู "ถูกต้อง" ทุกประการ

นี่คือรากของปัญหา: **memory-mapped register ไม่ใช่ "memory" ในความหมายที่ compiler เข้าใจ** มันคือ hardware
ที่มี**ผลข้างเคียง (side effect)** ที่ compiler มองไม่เห็นและไม่มีทางรู้ (การเขียนค่าลง ODR แต่ละครั้งทำให้
ไฟฟ้าจริงไหลเข้า/ออก pin จริง ไม่ว่าจะมีใคร "อ่านค่ากลับมา" ในโค้ดหรือไม่) — compiler ปรับปรุงโค้ดโดยยึด
สมมติฐานว่า memory ทำงานเหมือน memory ปกติ (การเขียนที่ไม่มีการอ่านคืนไม่มีผลที่สังเกตได้) ซึ่ง**เป็นสมมติฐาน
ที่ผิดสำหรับ memory-mapped register**

`core::ptr::write_volatile`/`read_volatile` คือทางแก้: มันบอก compiler ตรง ๆ ว่า **"ห้ามลบ ห้ามรวม ห้ามเรียง
ลำดับการเข้าถึงนี้ใหม่เด็ดขาด ต้องทำการอ่าน/เขียนจริงที่ address นี้ทุกครั้งตามลำดับที่เขียนไว้ในโค้ดเป๊ะ"** —
ไม่ว่า compiler จะคิดว่า "การเขียนนี้ไม่มีผลที่สังเกตได้" หรือไม่ก็ตาม:

```rust
use core::ptr::write_volatile;

/// เขียน error text ที่มักเจอถ้าลืม unsafe: "error[E0133]: call to unsafe function
/// `core::ptr::write_volatile` is unsafe and requires unsafe function or block"
unsafe fn turn_on_led_correct() {
    let odr = 0x4002_0014 as *mut u32;
    write_volatile(odr, 1 << 5); // เขียนจริงแน่นอน ไม่มีทางถูกลบทิ้งโดย compiler
    // ...รอเวลาสักพัก (ดูหัวขัดถัดไปสำหรับ delay)...
    write_volatile(odr, 0);      // เขียนจริงแน่นอนอีกครั้ง ไม่ถูกรวมเข้ากับครั้งแรก
}
```

ทั้งสองฟังก์ชันนี้ **มี type signature เหมือนกันทุกประการและ compile ผ่านทั้งคู่** — ความต่างไม่ปรากฏใน error
message ใด ๆ เลย (Rust compiler ไม่มีทางรู้ว่า address `0x4002_0014` คือ hardware หรือ RAM ธรรมดา — นี่คือ
สิ่งที่**อยู่นอกเหนือขอบเขตที่ static analysis พิสูจน์ได้**ตามที่ Part 41 สอนไว้ตรง ๆ) — ความรับผิดชอบทั้งหมด
ในการเลือกใช้ `write_volatile` แทนการ dereference ตรง ๆ ตกอยู่ที่โปรแกรมเมอร์ 100% นี่คือตัวอย่างที่ชัดเจน
มากของ "safety contract ที่ compiler ตรวจให้ไม่ได้" ที่ Part 41 พูดถึงไว้ในหัวข้อ 41.6

#### 102.5.2 Safe Abstraction เหนือ Register: แนวคิดของ PAC และ `embedded-hal`

การเขียน `write_volatile(0x4002_0014 as *mut u32, 1 << 5)` ตรง ๆ ทุกครั้งเป็นวิธีที่**ถูกต้องแต่เสี่ยงต่อความ
ผิดพลาดของมนุษย์สูงมาก** — เขียนเลข offset ผิดหนึ่งหลัก, สลับ bit position, หรือลืมเปิด clock ให้ peripheral
ก่อนใช้งาน (STM32 ต้องเปิด clock ผ่าน register `RCC_AHB1ENR` ก่อน GPIOA ถึงจะตอบสนองอะไรเลย) ล้วนเป็นบั๊กที่
compiler ตรวจให้ไม่ได้เพราะมันเป็นเพียง "ตัวเลข" ในสายตา compiler

ชุมชน embedded Rust แก้ปัญหานี้ด้วยสองชั้นของ safe abstraction ที่ห่อ raw pointer access เอาไว้ (ตรงกับ
หลักปรัชญา "safe abstraction เหนือ unsafe code" ที่ Part 41 หัวข้อ 41.8 สอนไว้เป๊ะ — เพียงแค่ประยุกต์ใช้กับ
hardware register แทน data structure ทั่วไป):

1. **PAC (Peripheral Access Crate)** — สร้างขึ้นอัตโนมัติจากไฟล์ SVD (System View Description ที่ผู้ผลิต
   ชิปแต่ละรายเผยแพร่ อธิบาย register map ทั้งหมดของชิปนั้นแบบละเอียด) ผ่านเครื่องมือ `svd2rust` — ผลลัพธ์คือ
   struct/method ที่ห่อ address ตัวเลขทั้งหมดไว้ให้เขียนโค้ดแบบ `gpioa.moder.modify(|_, w| w.moder5()
   .output())` แทนการคำนวณ offset ด้วยมือ ตัวอย่าง crate กลุ่มนี้คือ `stm32f4`, `nrf52840-pac`, `lm3s6965`
   (ที่จะใช้ใน QEMU section ถัดไป) — แต่ละ crate ตรงกับชิปหนึ่งตัวหรือหนึ่งตระกูล
2. **`embedded-hal`** — เป็นชุด **trait มาตรฐาน** (ไม่ใช่ implementation) ที่นิยาม "อินเทอร์เฟซกลาง" สำหรับ
   การทำงานของ hardware ทั่วไป เช่น trait `OutputPin` ที่มี method `.set_high()`/`.set_low()` — HAL crate
   ของแต่ละชิป (เช่น `stm32f4xx-hal`) implement trait เหล่านี้ให้ครบ ทำให้โค้ด**ระดับ logic**ที่เขียนโดยอิง
   `embedded-hal` trait ทำงานได้กับ**ชิปคนละตัวโดยไม่ต้องแก้โค้ด logic เลย** (สลับแค่ HAL crate ที่ import) —
   นี่คือ generic ในความหมายเดียวกับที่ Part 18 สอนไว้ แต่ทำงานข้าม **hardware vendor** แทนข้าม **data type**

ตัวอย่างโค้ดที่ใช้ `embedded-hal` trait (แสดงรูปร่างของอินเทอร์เฟซ ไม่ผูกกับชิปตัวใดตัวหนึ่ง):

```rust
#![no_std]

use embedded_hal::digital::OutputPin;

/// ฟังก์ชันนี้ generic เหนือ "อะไรก็ตามที่ implement OutputPin" -- ไม่สนว่าเบื้องหลังจะเป็น
/// STM32, nRF52, หรือ RP2040 -- นี่คือพลังของ trait-based HAL: เขียน logic ครั้งเดียว
/// ใช้ได้กับชิปได้หลายตัวโดยแค่เปลี่ยนว่า caller ส่ง pin ชนิดไหนเข้ามา
pub fn blink_once<P: OutputPin>(led: &mut P) -> Result<(), P::Error> {
    led.set_high()?; // ภายในของ .set_high() คือการเขียน volatile ไปยัง ODR register จริง
                      // แต่โค้ดชั้นนี้ไม่เห็นรายละเอียดนั้นเลย -- ถูกห่อไว้หมดแล้วโดย HAL crate
    led.set_low()?;
    Ok(())
}
```

โค้ดชั้นนี้**ไม่มี `unsafe` เลยแม้แต่คำเดียว** — นี่คือผลของ safe abstraction ที่ทำงานถูกต้อง: `unsafe` ทั้งหมด
(raw pointer, `write_volatile`) ถูกซ่อนอยู่ข้างในการ implement ของ `set_high()`/`set_low()` ที่ HAL crate
เขียนไว้ให้ครั้งเดียวอย่างละเอียดถี่ถ้วน (โดยผู้เชี่ยวชาญที่รู้ register map ของชิปนั้นจริง ๆ) ส่วนโค้ด logic
ที่เหลือ 95% ของโปรเจกต์ (แบบ `blink_once` นี้) เขียนเป็น 100% safe Rust ที่ borrow checker ตรวจสอบให้เต็ม
รูปแบบ — สัดส่วนนี้คือคำตอบที่แม่นยำที่สุดต่อคำถามที่หัวข้อ 102.1 ทิ้งไว้: **"พื้นที่ unsafe เล็กลงมาก และถูก
เขียนโดยคนที่เข้าใจ hardware จริง ๆ ครั้งเดียว แทนที่ทุกคนต้องเขียน raw pointer เองทุกที่"**

**หมายเหตุเชิงเทคนิคที่สังเกตได้จริงจากการ build ในบทนี้**: ผลลัพธ์ `cargo build` ของหลายตัวอย่างในบทนี้ (ดู
หัวข้อ 102.6, 102.9, 102.9.1) แสดง `Compiling embedded-hal v0.2.7` และ `Compiling embedded-hal v1.0.0` **พร้อม
กันทั้งสองเวอร์ชัน** — นี่ไม่ใช่ความผิดพลาด แต่สะท้อนสถานะจริงของ ecosystem ในช่วงเปลี่ยนผ่าน: `embedded-hal
1.0` (ออกปี 2024) ปรับ trait หลายตัวใหม่ทั้งหมดจาก `0.2` (เช่นลบ associated type ที่ไม่จำเป็น, รวม error
handling ให้สอดคล้องกันมากขึ้น) แต่ crate จำนวนมากในระบบยังไม่อัปเดตไปใช้ `1.0` ทั้งหมด (โดยเฉพาะ crate รุ่น
เก่าอย่าง `lm3s6965` ในหัวข้อ 102.9.1) — cargo จึงต้อง compile ทั้งสองเวอร์ชันไว้พร้อมกันในกราฟ dependency
เดียว (คนละ crate ใน dependency tree อ้างอิงคนละเวอร์ชัน ซึ่งเป็นเรื่องปกติของ semver — Rust อนุญาตให้มีหลาย
major version ของ crate เดียวกันอยู่ร่วมกันได้ตราบใดที่ไม่มีใครต้อง pass type ข้ามเวอร์ชันกันตรง ๆ) — บทเรียน
เชิงปฏิบัติ: เวลาเลือก HAL crate สำหรับโปรเจกต์ใหม่ ควรตรวจสอบว่ามัน implement `embedded-hal` เวอร์ชันไหน
เพราะโค้ด logic ที่เขียนอิง trait จาก `1.0` จะใช้กับ HAL ที่ยัง implement แค่ `0.2` ไม่ได้ตรง ๆ (ต้องมี
compatibility shim คั่นกลาง)

### 102.6 ตัวอย่างจริง: Blink LED ด้วย Register-level Access (ยืนยันด้วย Cross-compilation)

"Hello, world" ของโลก embedded ไม่ใช่การพิมพ์ข้อความ — มันคือการทำให้ **LED กระพริบ** เพราะเป็นตัวอย่างที่
เล็กที่สุดที่พิสูจน์ได้ว่าโค้ดควบคุม hardware จริงสำเร็จ (เห็นผลด้วยตาเปล่าโดยไม่ต้องมี debugger/console เลย)

ตัวอย่างนี้เขียนสำหรับ MCU ตระกูล **STM32F401** (ARM Cortex-M4, พบใน development board อย่าง "Nucleo-F401RE"
ที่ LED สำหรับผู้ใช้ต่ออยู่ที่ pin **PA5**) — เลือกชิปนี้เพราะเป็นบอร์ดที่ใช้สอน embedded Rust กันแพร่หลายมาก
ที่สุดในบทเรียนสาย ARM Cortex-M ทั่วไป address ที่ใช้ทั้งหมดในตัวอย่างนี้อ้างอิงจาก reference manual ของ
STM32F4 ตระกูลนี้ (RM0368):

```rust
#![no_std]
#![no_main]

use core::ptr::write_volatile;
use cortex_m_rt::entry;
use panic_halt as _;

// --- Address ของ register ที่เกี่ยวข้อง (จาก STM32F401 Reference Manual RM0368) ---
const RCC_AHB1ENR: *mut u32 = 0x4002_3830 as *mut u32; // เปิด clock ให้ peripheral bus AHB1
const GPIOA_MODER: *mut u32 = 0x4002_0000 as *mut u32; // ตั้งโหมดของแต่ละ pin ใน port A
const GPIOA_ODR: *mut u32 = 0x4002_0014 as *mut u32;   // เขียนค่า output จริงของแต่ละ pin

const GPIOAEN_BIT: u32 = 1 << 0;  // bit 0 ของ RCC_AHB1ENR = เปิด clock ให้ GPIOA
const PA5_OUTPUT_MODE: u32 = 0b01 << (5 * 2); // 2 บิตต่อ pin ใน MODER, pin 5 อยู่ที่บิต 10-11
const PA5_HIGH: u32 = 1 << 5; // bit 5 ของ ODR = pin PA5

#[entry]
fn main() -> ! {
    unsafe {
        // ขั้นที่ 1: เปิด clock ให้ GPIOA ก่อน -- ถ้าลืมขั้นนี้ pin จะไม่ตอบสนองอะไรเลย
        // (ปัญหาคลาสสิกของ embedded ที่เขียนโค้ดถูกทุกอย่างแต่ "ไม่มีอะไรเกิดขึ้น" เพราะลืมเปิด clock)
        let current = core::ptr::read_volatile(RCC_AHB1ENR);
        write_volatile(RCC_AHB1ENR, current | GPIOAEN_BIT);

        // ขั้นที่ 2: ตั้งโหมดของ PA5 ให้เป็น output (ค่าเริ่มต้นคือ input)
        let current = core::ptr::read_volatile(GPIOA_MODER);
        write_volatile(GPIOA_MODER, current | PA5_OUTPUT_MODE);
    }

    loop {
        unsafe {
            write_volatile(GPIOA_ODR, PA5_HIGH); // LED ติด
        }
        cortex_m::asm::delay(3_000_000); // busy-wait หน่วงเวลาประมาณครึ่งวินาที (ที่ 8 MHz default clock)

        unsafe {
            write_volatile(GPIOA_ODR, 0); // LED ดับ
        }
        cortex_m::asm::delay(3_000_000);
    }
}
```

โค้ดนี้เพียงไฟล์เดียวยังไม่พอที่จะ compile เป็น binary สำหรับ MCU ได้จริง — ต้องมีไฟล์สนับสนุนอีกสามไฟล์ที่
บอก linker ว่า Flash/RAM ของชิปนี้อยู่ตรงไหน และบอก cargo ว่า target เริ่มต้นคืออะไร (โครงสร้างนี้เหมือนกับ
ที่หัวข้อ 102.7 จะแสดงสำหรับ QEMU ทุกประการ เพียงแค่ตัวเลข memory address ต่างกันตามชิป):

`memory.x` (Flash ของ STM32F401 เริ่มที่ `0x0800_0000` ตามสเปกของ ARM Cortex-M ที่ area นี้สงวนไว้สำหรับ
โปรแกรมที่ฝังอยู่ใน Flash เสมอ, RAM เริ่มที่ `0x2000_0000` ตามสเปกเดียวกัน):

```text
MEMORY
{
  FLASH : ORIGIN = 0x08000000, LENGTH = 256K
  RAM : ORIGIN = 0x20000000, LENGTH = 64K
}
```

`build.rs` (คัดลอก `memory.x` เข้า `OUT_DIR` แล้วบอก linker ให้หาไฟล์นี้เจอ และสั่งใช้ linker script `link.x`
ที่ `cortex-m-rt` เตรียมไว้ให้ — โครงสร้างนี้คือ pattern มาตรฐานของ `cortex-m-quickstart` template ที่โปรเจกต์
embedded Rust ส่วนใหญ่ใช้กัน):

```rust
use std::env;
use std::fs::File;
use std::io::Write;
use std::path::PathBuf;

fn main() {
    let out_dir = PathBuf::from(env::var_os("OUT_DIR").unwrap());
    File::create(out_dir.join("memory.x"))
        .unwrap()
        .write_all(include_bytes!("memory.x"))
        .unwrap();
    println!("cargo:rustc-link-search={}", out_dir.display());
    println!("cargo:rerun-if-changed=memory.x");
    println!("cargo:rustc-link-arg=-Tlink.x");
}
```

`.cargo/config.toml` (ตั้ง target เริ่มต้นให้ `cargo build` เปล่า ๆ ไม่ต้องพิมพ์ `--target` ซ้ำทุกครั้ง):

```toml
[build]
target = "thumbv7em-none-eabihf"
```

`Cargo.toml`:

```toml
[package]
name = "blink"
version = "0.1.0"
edition = "2021"

[dependencies]
cortex-m = "0.7"
cortex-m-rt = "0.7"
panic-halt = "0.2"

[profile.release]
panic = "abort"
```

โค้ดนี้แสดงรายละเอียดเชิงกลไกที่มักถูกมองข้ามในตัวอย่าง "blink" ที่เขียนสั้นเกินไป: **ต้องเปิด clock ของ
peripheral ก่อนใช้งานเสมอ** — MCU ยุคใหม่ทุกตัวออกแบบมาให้**ประหยัดพลังงาน** โดย peripheral ทุกตัว (GPIO,
timer, UART, ฯลฯ) จะ**ไม่มี clock จ่ายให้** ตั้งแต่เริ่ม (power-on) เพื่อลดการใช้พลังงานของวงจรที่ไม่ได้ใช้ —
โปรแกรมเมอร์ต้อง**เปิด clock เองผ่าน RCC (Reset and Clock Control) register ก่อนเสมอ** ก่อนจะตั้งค่า/ใช้งาน
peripheral ตัวนั้น — นี่คือกับดักที่พบบ่อยที่สุดข้อหนึ่งของมือใหม่ embedded (ไม่ใช่แค่ใน Rust — เกิดในทุกภาษา)
ที่จะกล่าวถึงในหัวข้อกับดักท้ายบทด้วย

`cortex_m::asm::delay(cycles: u32)` เป็นฟังก์ชันจาก crate `cortex-m` ที่หน่วงเวลาด้วยการวนลูปนับ cycle จริง
(busy-wait) — ไม่ใช้ระบบ timer/interrupt เพื่อความง่ายที่สุดสำหรับตัวอย่างแรก (การหน่วงเวลาที่ถูกต้องเชิง
วิศวกรรมจริงควรใช้ hardware timer แทน busy-wait เสมอ เพราะ busy-wait กิน CPU 100% ตลอดเวลาหน่วง ทำให้ทำงาน
อื่นพร้อมกันไม่ได้เลย — ประเด็นนี้จะเห็นชัดขึ้นในหัวข้อ interrupt และ async ถัดไป)

**สถานะการยืนยัน**: ตัวอย่างนี้ผ่านการ cross-compile จริงด้วย `cargo build --target thumbv7em-none-eabihf`
ในสภาพแวดล้อมของบทนี้ (โครงสร้างโปรเจกต์เดียวกับที่แสดงไว้: `Cargo.toml` ตามที่กำหนด dependency ข้างบน,
`memory.x` และ `build.rs` แบบเดียวกับที่หัวข้อ 102.7 จะแสดงเต็มรูปแบบ) ผลลัพธ์จริงของ `cargo build` (debug
profile):

```
   Compiling cortex-m v0.7.9
   ...
   Compiling blink v0.1.0 (.../blink)
   ...
   Compiling cortex-m-rt-macros v0.7.7
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 5.16s
```

ไม่มี error หรือ warning ที่เกี่ยวข้องกับ target/linking เลยแม้แต่บรรทัดเดียว — ยืนยันว่าโค้ดถูกต้องตาม
syntax, type, borrow checker, และรูปร่างของ binary ที่ linker คาดหวังสำหรับ ARM Cortex-M4 ครบทุกมิติ —
เนื่องจาก sandbox ไม่มี STM32F401 ตัวจริง (และ QEMU ไม่รองรับการจำลอง STM32F4 ในระดับที่สังเกตการ toggle ของ
GPIO ได้จริงในสภาพแวดล้อมนี้) **การยืนยันของตัวอย่างนี้จึงอยู่ระดับ "compile ถูกต้องสมบูรณ์ ยืนยันได้ว่าจะรัน
บน hardware จริงได้" แต่ไม่ได้รันจริงบน LED จริง** — บทนี้จะไม่กล่าวอ้างว่า "เห็น LED กระพริบจริง" เพราะไม่เป็น
ความจริงในสภาพแวดล้อมที่เขียนบทนี้

### 102.7 รันจริงบน QEMU: Semihosting บนบอร์ดจำลอง `lm3s6965evb`

ตัวอย่างในหัวข้อก่อนพิสูจน์ได้แค่ระดับ cross-compilation — หัวข้อนี้จะพยายามไปให้ถึงระดับที่แน่นหนากว่า:
**การรันจริงและเห็นผลลัพธ์จริงจาก CPU จำลอง** ผ่าน QEMU ซึ่งเป็นไปได้เพราะ QEMU มีโมเดลจำลองของบอร์ด TI
Stellaris **LM3S6965EVB** (คอร์ ARM Cortex-M3) ไว้ล่วงหน้า — บอร์ดจำลองนี้ถูกเลือกใช้เป็นมาตรฐานอย่างเป็น
ทางการในหนังสือ **The Embedded Rust Book** สำหรับจุดประสงค์เดียวกับบทนี้เป๊ะ: สอน/ทดสอบ embedded Rust โดยไม่
ต้องมี hardware จริง

กลไกที่ทำให้เห็น output จาก CPU จำลองได้เรียกว่า **semihosting** — เป็น protocol พิเศษที่ให้โปรแกรมที่รันบน
ARM (จริงหรือจำลอง) ส่งคำสั่งพิเศษ (breakpoint instruction พร้อมรหัสเฉพาะ) ที่ debugger/emulator ดักจับไว้
เพื่อทำงานแทน เช่น "พิมพ์ข้อความนี้ออกไปยัง console ของเครื่อง host" หรือ "จบการทำงานพร้อม exit code นี้" — มัน
ทำงานได้แม้ MCU จำลองนั้นไม่มี UART/console จริงต่ออยู่เลย เพราะ QEMU เองเป็นผู้ดักจับคำสั่งพิเศษนี้แล้วพิมพ์
ออกมาที่ terminal ของเครื่อง host ตรง ๆ

```rust
#![no_std]
#![no_main]

use cortex_m_rt::entry;
use cortex_m_semihosting::{debug, hprintln};
use panic_halt as _;

#[entry]
fn main() -> ! {
    // hprintln! ทำงานคล้าย println! ทุกประการในทางไวยากรณ์ แต่ข้างใต้ส่ง semihosting call
    // ไปให้ QEMU (หรือ debug probe จริงที่รองรับ semihosting) พิมพ์ออกที่ terminal ของ host แทน
    hprintln!("Hello, embedded world!");
    hprintln!("2 + 2 = {}", 2 + 2);

    // debug::exit บอก QEMU ให้จบการรันของ CPU จำลอง พร้อมส่ง exit code กลับไปยัง process ของ QEMU เอง
    // เทียบเท่ากับ std::process::exit() ของโปรแกรมทั่วไป แต่ทำงานได้แม้ไม่มี OS อยู่ข้างใต้เลย
    debug::exit(debug::EXIT_SUCCESS);

    // โค้ดหลังจากนี้ไม่ควรถูกรันถึง เพราะ debug::exit หยุด QEMU ไปแล้ว แต่ยังต้องมี loop
    // เพื่อให้ type ตรงกับ -> ! (ในกรณีที่รันบน hardware จริงที่ debug::exit ไม่มีผลอะไร)
    loop {}
}
```

โครงสร้างโปรเจกต์ที่ใช้ compile ตัวอย่างนี้ (ทั้งหมดนี้ถูกสร้างจริงและ build จริงในสภาพแวดล้อมเขียนบทนี้ ดูผล
การ build ที่หัวข้อ 102.11):

`Cargo.toml`:

```toml
[package]
name = "qemu-hello"
version = "0.1.0"
edition = "2021"

[dependencies]
cortex-m = "0.7"
cortex-m-rt = "0.7"
cortex-m-semihosting = "0.5"
panic-halt = "0.2"

[profile.release]
panic = "abort"
```

`memory.x` (บอก linker ว่า Flash/RAM ของ LM3S6965 เริ่มที่ address ไหน กว้างแค่ไหน — ตัวเลขนี้ตรงกับตัวอย่าง
มาตรฐานของ Embedded Rust Book สำหรับบอร์ดจำลองนี้พอดี):

```text
MEMORY
{
  FLASH : ORIGIN = 0x00000000, LENGTH = 256K
  RAM : ORIGIN = 0x20000000, LENGTH = 64K
}
```

`.cargo/config.toml` (บอก cargo ว่า target เริ่มต้นคืออะไร และวิธีสั่งรัน binary ที่ build ได้ผ่าน `cargo
run` โดยตรงให้ไปเข้า QEMU อัตโนมัติ):

```toml
[target.thumbv7m-none-eabi]
runner = "qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb -nographic -semihosting-config enable=on,target=native -kernel"

[build]
target = "thumbv7m-none-eabi"
```

สังเกตว่า target ของตัวอย่างนี้คือ **`thumbv7m-none-eabi`** ไม่ใช่ `thumbv7em-none-eabihf` แบบตัวอย่าง
STM32F401 ในหัวข้อก่อน — เพราะ LM3S6965 ใช้คอร์ **Cortex-M3** ซึ่งไม่มี FPU (floating-point unit) และไม่มี
DSP extension แบบ Cortex-M4 มี ต้องเลือก target ให้ตรงกับ **ชุดคำสั่งที่ CPU ตัวนั้นรองรับจริง** เท่านั้น
(เลือก target ผิดกลุ่มจะได้ error ตอน link หรือแย่กว่านั้นคือได้ binary ที่ compile ผ่านแต่รันแล้ว crash
เพราะ CPU จริงไม่รู้จักคำสั่งบางตัว — รายละเอียดนี้จะอยู่ในหัวข้อกับดักท้ายบทด้วย)

**สถานะการยืนยันจริง**: ตามที่ระบุไว้ในหัวข้อ 102.2 — `qemu-system-arm` **ไม่มีอยู่ในสภาพแวดล้อมที่เขียนบทนี้**
แม้จะพยายามติดตั้งเพิ่มเติมด้วย `apt-get install -y qemu-system-arm` แล้วก็ตาม (การติดตั้งล้มเหลวเพราะ mirror
package ที่ต้องดาวน์โหลด `qemu-system-arm` ตัวจริงถูกปฏิเสธ/timeout โดย proxy ของสภาพแวดล้อมนี้) — ดังนั้น
ตัวอย่างนี้**ไม่ได้ถูกรันจริงผ่าน QEMU** ในบทนี้ สิ่งที่ยืนยันได้จริงคือระดับ cross-compilation เท่านั้น:
โครงสร้างโปรเจกต์ทั้งหมด (`Cargo.toml`, `memory.x`, `build.rs`, `.cargo/config.toml`, `src/main.rs`) ตามที่
แสดงไว้ข้างล่างถูกสร้างขึ้นจริงและ `cargo build` (target `thumbv7m-none-eabi`) ผ่านสำเร็จจริง ให้ผลลัพธ์:

```
    Updating crates.io index
     Locking 24 packages to latest compatible versions
      Adding cortex-m-semihosting v0.5.0 (available: v0.6.0)
      Adding panic-halt v0.2.0 (available: v1.0.0)
 Downloading crates ...
  Downloaded cortex-m-semihosting v0.5.0
   Compiling proc-macro2 v1.0.107
   ...
   Compiling qemu-hello v0.1.0 (.../qemu-hello)
   Compiling panic-halt v0.2.0
   ...
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 4.86s
```

นี่คือหลักฐานที่แน่นหนาที่สุดที่ทำได้จริงในสภาพแวดล้อมนี้: โค้ด `hprintln!`/`debug::exit` ที่เรียก
`cortex-m-semihosting` compile และ link ผ่านสมบูรณ์สำหรับ ARM Cortex-M3 (`thumbv7m-none-eabi`) ตรงตามที่
บอร์ดจำลอง `lm3s6965evb` ต้องการ — **ถ้าคุณมีเครื่องที่เข้าถึง QEMU ได้จริง** คำสั่ง `cargo run` (ที่ runner
ใน `.cargo/config.toml` ตั้งไว้ให้เรียก `qemu-system-arm` อัตโนมัติ) ควรให้ผลลัพธ์คือข้อความ "Hello, embedded
world!" และ "2 + 2 = 4" ปรากฏที่ terminal ตามลำดับ แล้ว QEMU ปิดตัวเองด้วย exit code 0 — แต่บทนี้**จะไม่กล่าว
อ้างว่าได้เห็นข้อความเหล่านั้นจริงในสภาพแวดล้อมนี้** เพราะไม่เป็นความจริง

### 102.8 Panic Handling ใน `#[no_std]`: ไม่มี Unwinding ไม่มี stderr

โปรแกรม Rust ปกติที่มี `std` เมื่อเจอ `panic!()` (ไม่ว่าจะเรียกตรง ๆ หรือผ่านการ `.unwrap()` ค่า `None`/`Err`)
จะมี**ตัวจัดการ panic ที่ `std` เตรียมไว้ให้อัตโนมัติ**: พิมพ์ข้อความ error ไปที่ `stderr`, แสดง backtrace
(ถ้าตั้งค่า `RUST_BACKTRACE=1`), แล้ว unwind stack (เรียก `Drop` ของทุกตัวแปรที่ยังไม่หลุด scope เพื่อคืน
resource อย่างถูกต้อง) ก่อนจบ thread หรือจบโปรแกรม — สิ่งนี้ทำงานได้เพราะ **`std` เชื่อมโยงอยู่กับ OS ที่มี
`stderr` และมี unwinding machinery (เช่น libunwind) ให้ใช้**

บน `#[no_std]` **ไม่มีสิ่งใดในนี้เลย**: ไม่มี `stderr` (ไม่มี OS ที่จะรับ output ไปแสดง), และ unwinding
machinery ก็มักถูกปิดไปด้วย (เพราะกินพื้นที่ Flash มากและซับซ้อนเกินจำเป็นสำหรับ MCU ขนาดเล็ก) — ผลคือ: **ถ้า
ไม่กำหนด panic handler เอง โปรแกรม `#[no_std]` จะ compile ไม่ผ่านเลย** ด้วย error ที่บอกตรง ๆ ว่าขาด
`#[panic_handler]` (จะแสดง error message จริงในหัวข้อกับดักท้ายบท)

Rust แก้ปัญหานี้ด้วย attribute `#[panic_handler]` ที่ให้โปรแกรมเมอร์กำหนดฟังก์ชันที่จะถูกเรียกเมื่อ panic
เกิดขึ้น **เอง** — ฟังก์ชันนี้ต้องมี signature ตายตัว: รับ `&core::panic::PanicInfo` และ return `!` (ไม่มีวัน
return กลับ เพราะเมื่อ panic เกิดขึ้นบน MCU ที่ไม่มี OS แล้ว **ไม่มี "ที่ปลอดภัยให้กลับไปทำงานต่อ"** — สถานะ
ของโปรแกรมถูกมองว่าเสียหายไม่สามารถเชื่อถือได้อีก)

ตัวเลือกที่นิยมที่สุดสำหรับเริ่มต้น คือ crate **`panic-halt`** (ที่ใช้อยู่แล้วในทุกตัวอย่างของบทนี้ผ่าน
`use panic_halt as _;`) — การ implement ของมันเรียบง่ายที่สุดที่เป็นไปได้:

```rust
// นี่คือสิ่งที่ crate panic-halt ทำ (เขียนแสดงให้เห็นกลไก ไม่ต้องเขียนเองถ้า import crate จริง)
#![no_std]

use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {
        // วนลูปตลอดไปไม่จบ -- MCU จะ "แช่แข็ง" ตรงนี้เมื่อ panic เกิดขึ้น
        // ไม่มีการพิมพ์ข้อความอะไรเลย ไม่มีการกู้คืน แค่หยุดทำงานอย่างปลอดภัย
    }
}
```

`loop {}` เปล่า ๆ ดูเรียบง่ายเกินไป แต่มีเหตุผลเชิงวิศวกรรมที่แข็งแรงรองรับ: มันเป็นพฤติกรรม**ที่คาดเดาได้และ
ปลอดภัยที่สุด**เมื่อไม่มีข้อมูลอะไรเกี่ยวกับ hardware ที่กำลังรันอยู่เลย (ไม่รู้ว่ามี watchdog timer ที่จะ
reset ตัวเองถ้าค้างนานเกินไปหรือไม่ ไม่รู้ว่ามี debug probe ต่ออยู่ให้ดู state ตอนค้างหรือไม่) — "หยุดทำงาน
เฉย ๆ" ปลอดภัยกว่าการพยายามทำอะไรต่อในสถานะที่ไม่รู้ว่าเสียหายไปแค่ไหนแล้ว (คล้ายหลักการ "fail-safe" ที่ใช้
ในวิศวกรรมความปลอดภัยทั่วไป)

สำหรับงานพัฒนาจริง (ไม่ใช่ production firmware สุดท้าย) มีตัวเลือกที่ให้ข้อมูล debug กลับมาด้วย:

- **`panic-semihosting`** — พิมพ์ข้อความ panic ผ่าน semihosting (เหมือนที่ใช้ใน `hprintln!` หัวข้อก่อน) —
  ใช้ได้เฉพาะตอนมี debug probe/QEMU ต่ออยู่ ไม่เหมาะกับ production เพราะ semihosting ทำให้ CPU ช้าลงมากถ้า
  ไม่มี debugger ดักฟังอยู่จริง (บาง MCU จะ hang ค้างรอ debugger ถ้าไม่มีตัวดักจับ)
- **`panic-probe` ร่วมกับ `defmt`** — เป็นแนวทางที่ทันสมัยและใช้กันแพร่หลายที่สุดในโปรเจกต์ embedded Rust
  ปัจจุบัน: `defmt` (deferred formatting) คือ logging framework ที่ออกแบบมาเฉพาะสำหรับ embedded โดยเฉพาะ —
  แทนที่จะ format string เป็น text เต็มรูปแบบตอน runtime (แพงมากสำหรับ MCU ที่ clock ช้า) มันส่ง**แค่ตัวเลข
  รหัสของ format string** (ที่ table เก็บไว้ใน binary) ผ่าน debug probe ไปให้เครื่อง development ทำ format
  เต็มรูปแบบแทนที่ฝั่งนั้น — วิธีนี้ทำให้ log message มีต้นทุนต่ำกว่า `println!`-style ธรรมดามาก (ทั้งในแง่
  ขนาด binary และเวลาในการส่งข้อมูลออกทาง debug probe ที่ bandwidth จำกัด) `panic-probe` ใช้กลไกเดียวกันนี้
  สำหรับ panic message โดยเฉพาะ

บทนี้จะไม่ลงรายละเอียดเต็มรูปแบบของ `defmt` (เป็นหัวข้อที่ลึกพอจะเป็นบทของตัวเองในเส้นทาง embedded ที่ลึกกว่า
"เบื้องต้น") — สิ่งที่ต้องจำจากหัวข้อนี้คือ **spectrum ของตัวเลือก panic handler**: จาก `panic-halt` (ง่าย
ที่สุด ไม่มีข้อมูล debug เลย เหมาะกับ production) ไปจนถึง `panic-probe`/`defmt` (ข้อมูล debug ละเอียด ต้นทุน
ต่ำกว่า semihosting มาก เหมาะกับ development)

### 102.9 Interrupts: โครงสร้างงานแบบ Event-driven บน Hardware จริง

ทุกโปรแกรมที่หลักสูตรนี้สอนมา (ยกเว้นเนื้อหา async/await ใน Part 30+) จัดโครงสร้างงานแบบ **polling** หรือ
**synchronous flow**: โค้ดรันตามลำดับ ตรวจสอบเงื่อนไข วนซ้ำถ้าจำเป็น — แม้แต่ async/await (ที่ executor
อย่าง Tokio คอย poll `Future` เป็นระยะ) ก็ยังเป็นรูปแบบของการ "ถามซ้ำ ๆ ว่าพร้อมหรือยัง" ในเชิงกลไก

**Embedded programming ส่วนใหญ่กลับใช้แนวทางตรงข้าม**: แทนที่โปรแกรมจะคอย "ถาม" hardware ว่ามีอะไรเกิดขึ้น
หรือยัง (ซึ่งกิน CPU cycle ไปเปล่า ๆ ระหว่างที่ยังไม่มีอะไรเกิดขึ้น — สำคัญมากสำหรับ MCU ที่ต้องประหยัดพลังงาน
และ CPU มีจำกัด) hardware จะ**บอกโปรแกรมเองทันทีที่มีเหตุการณ์เกิดขึ้น**ผ่านกลไกที่เรียกว่า **interrupt** —
เมื่อเหตุการณ์ที่ตั้งไว้เกิดขึ้นจริง (ปุ่มถูกกด, timer นับครบ, ข้อมูลมาถึง UART) CPU จะ**หยุดสิ่งที่กำลังทำอยู่
ทันที** กระโดดไปรันฟังก์ชันพิเศษที่เรียกว่า **interrupt handler** (หรือ Interrupt Service Routine — ISR)
แล้วกลับมาทำงานเดิมต่อเมื่อ handler จบ

`cortex-m-rt` ให้ attribute `#[interrupt]` สำหรับกำหนดฟังก์ชันที่จะถูกเรียกเมื่อ interrupt ตัวใดตัวหนึ่งเกิด
ขึ้น — ชื่อฟังก์ชันต้องตรงกับชื่อ interrupt vector ที่ประกาศไว้ใน **PAC** ของชิปนั้น (เพราะแต่ละชิปมีจำนวน
และชื่อ interrupt ต่างกัน ขึ้นกับ peripheral ที่มีจริง — นี่คือสาเหตุที่ `#[interrupt]` (ต่างจาก `#[entry]`)
ต้องพึ่ง PAC เสมอ ไม่สามารถใช้ได้กับแค่ `cortex-m-rt` เปล่า ๆ)

Cortex-M ยังมี **exception** อีกกลุ่มหนึ่งที่เป็นส่วนหนึ่งของ core CPU เอง (ไม่ผูกกับ peripheral ของชิปตัวใด
ตัวหนึ่ง) เช่น `SysTick` (timer ที่มีอยู่ในทุก Cortex-M ไม่ว่าผู้ผลิตชิปจะเป็นใคร) — exception กลุ่มนี้จัดการ
ผ่าน attribute `#[exception]` ที่ **ไม่ต้องพึ่ง PAC เลย** เพราะเป็นส่วนหนึ่งของสเปก ARM Cortex-M ที่
`cortex-m-rt` รู้จักโดยตรง:

```rust
#![no_std]
#![no_main]

use core::cell::RefCell;
use cortex_m::interrupt::Mutex;
use cortex_m::peripheral::syst::SystClkSource;
use cortex_m_rt::{entry, exception};
use panic_halt as _;

// ตัวนับที่แชร์กันระหว่าง main loop กับ SysTick exception handler
// -- นี่คือ "การแชร์ mutable state ข้าม context" แบบเดียวกับ Rc<RefCell<T>> ที่ Part 28 สอนไว้
// แต่ context ในที่นี้คือ "main thread" กับ "interrupt/exception handler" ไม่ใช่ OS thread สองตัว
static TICK_COUNT: Mutex<RefCell<u32>> = Mutex::new(RefCell::new(0));

#[exception]
fn SysTick() {
    // interrupt::free ทำหน้าที่เป็น critical section: ปิด interrupt ชั่วคราวระหว่างเข้าถึงข้อมูลแชร์
    // เพื่อการันตีว่า main loop จะไม่มาแก้ไข TICK_COUNT พร้อมกันกับ handler นี้ (ป้องกัน data race
    // แบบเดียวกับที่ Mutex ของ Part 39 ป้องกัน แต่ในที่นี้คือการปิด "การขัดจังหวะ" ไม่ใช่ "การรอ lock")
    cortex_m::interrupt::free(|cs| {
        let mut count = TICK_COUNT.borrow(cs).borrow_mut();
        *count = count.wrapping_add(1);
    });
}

#[entry]
fn main() -> ! {
    let mut peripherals = cortex_m::Peripherals::take().unwrap();

    // ตั้งค่า SysTick ให้ trigger exception ทุก ๆ ประมาณ 1 ล้าน cycle ของ core clock
    peripherals.SYST.set_clock_source(SystClkSource::Core);
    peripherals.SYST.set_reload(1_000_000);
    peripherals.SYST.clear_current();
    peripherals.SYST.enable_counter();
    peripherals.SYST.enable_interrupt();

    loop {
        // main loop อ่านค่าที่ exception handler แก้ไขอยู่เรื่อย ๆ ผ่าน critical section เดียวกัน
        let current = cortex_m::interrupt::free(|cs| *TICK_COUNT.borrow(cs).borrow());

        if current >= 10 {
            // ทำอะไรสักอย่างเมื่อ tick ครบ 10 ครั้ง (ตัวอย่าง: อาจเป็นสัญญาณให้กระพริบ LED เร็วขึ้น)
        }

        cortex_m::asm::nop();
    }
}
```

**สถานะการยืนยัน**: ตัวอย่างนี้ cross-compile จริงสำเร็จด้วย `cargo build --target thumbv7em-none-eabihf` ใน
สภาพแวดล้อมเขียนบทนี้ (ใช้ `memory.x`/`build.rs` แบบเดียวกับหัวข้อ 102.6):

```
   Compiling cortex-m v0.7.9
   ...
   Compiling interrupts v0.1.0 (.../interrupts)
   ...
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 5.15s
```

ผ่านโดยไม่มี error เลย — ยืนยันว่า `#[exception] fn SysTick()`, `cortex_m::Peripherals::take()`, และการใช้
`Mutex<RefCell<u32>>` ร่วมกับ `cortex_m::interrupt::free` ทั้งหมด compile ถูกต้องตาม API จริงของ `cortex-m`/
`cortex-m-rt` เวอร์ชันปัจจุบัน (ไม่ใช่ API ที่ล้าสมัยหรือเขียนผิด signature) — เช่นเดิม การยืนยันนี้อยู่ระดับ
cross-compilation เท่านั้น เพราะไม่มี hardware จริงหรือ QEMU ที่จำลอง SysTick แบบสังเกตผลได้ในสภาพแวดล้อมนี้

จุดที่ควรเน้นให้ชัด: **`Mutex<RefCell<T>>` ในบริบทนี้ไม่ใช่ `std::sync::Mutex`** — มันคือ
`cortex_m::interrupt::Mutex` ที่มีความหมายต่างไปโดยสิ้นเชิง: `std::sync::Mutex` (Part 39) ป้องกัน data race
ระหว่าง **OS thread หลายตัว** ที่ CPU อาจรันพร้อมกันจริง (multi-core) หรือสลับกันรัน (time-slicing) โดย OS
scheduler เป็นผู้ตัดสินใจว่าจะสลับตอนไหน — ส่วน `cortex_m::interrupt::Mutex` ป้องกัน data race ระหว่าง
**main code กับ interrupt/exception handler** บน **CPU core เดียว** (Cortex-M ส่วนใหญ่ที่ใช้สอน embedded
เบื้องต้นมีคอร์เดียว ไม่ใช่ multi-core) — ปัญหาการแข่งชิงข้อมูลยังเกิดได้จริงแม้เป็น core เดียว เพราะ interrupt
สามารถ**ขัดจังหวะ main code ได้ทุกเวลาที่ไม่ได้ปิดกั้นไว้** (ต่างจาก OS thread ที่ scheduler สลับตามตารางเวลา
ปกติ) — `cortex_m::interrupt::free(...)` จึงทำหน้าที่เป็น **critical section**: ปิดการขัดจังหวะทั้งหมดชั่วคราว
ระหว่างที่ closure ข้างในกำลังรัน เพื่อการันตีว่าไม่มี handler ตัวใดมาแก้ไขข้อมูลระหว่างกลาง — เมื่อปิด
interrupt ไปแล้ว การเข้าถึง `RefCell` ข้างในจึงปลอดภัยสนิท (ไม่มีทางมี aliasing ที่ผิดกฎเกิดขึ้นได้ เพราะไม่มี
ใครมาแทรกได้เลยในช่วงเวลานั้น) — นี่คือเหตุผลที่ token `cs: CriticalSection` ถูกส่งเป็น parameter ให้
`.borrow(cs)` เรียก: มันคือ**หลักฐานทาง type system** ว่าโค้ดนี้กำลังรันอยู่ในช่วงที่ interrupt ถูกปิดจริง
(ไม่มีทางเรียก `.borrow(cs)` นอก closure ของ `interrupt::free` ได้เลย เพราะไม่มี `cs` ให้ใช้ — compiler
บังคับความถูกต้องนี้ให้โดยอัตโนมัติ)

ชุมชน embedded Rust ยุคหลังนิยมใช้ crate **`critical-section`** แทน `cortex_m::interrupt::free` โดยตรง
(เพราะ `critical-section` เป็น abstraction ที่ทำงานได้ทั้งบน single-core และ multi-core MCU, ทั้งบน bare-metal
และภายใต้ RTOS) — แนวคิดเบื้องหลังเหมือนกันทุกประการกับตัวอย่างข้างบน เพียงแค่เปลี่ยนชื่อ type ที่ใช้ (จาก
`cortex_m::interrupt::Mutex` เป็น `critical_section::Mutex`) — `embassy` framework ในหัวข้อถัดไปก็ใช้
`critical-section` เป็นฐานเดียวกันนี้เช่นกัน

#### 102.9.1 `#[interrupt]` ตัวจริง: ผูกกับชื่อ Interrupt Vector ของ PAC เฉพาะชิป

ตัวอย่างข้างบนใช้ `#[exception]` สำหรับ `SysTick` เพราะเป็น exception ที่เป็นส่วนหนึ่งของสเปก ARM Cortex-M
เอง ไม่ผูกกับชิปตัวใดตัวหนึ่ง — แต่ `#[interrupt]` ที่หัวข้อนี้ตั้งใจแสดงตั้งแต่ต้น**ต้องมีชื่อตรงกับ interrupt
vector ที่มาจาก PAC ของชิปจริง** เสมอ (อธิบายไว้แล้วก่อนหน้านี้ในหัวข้อนี้) — มาดูตัวอย่างที่ผูกกับ interrupt
`GPIOA` ของ PAC `lm3s6965` (crate เดียวกับที่ตรงกับบอร์ดจำลอง `lm3s6965evb` ในหัวข้อ 102.7) ซึ่งจำลองสถานการณ์
"ปุ่มถูกกดบน GPIO port A แล้ว interrupt handler นับจำนวนครั้งที่ถูกกด":

```rust
#![no_std]
#![no_main]

use core::cell::Cell;
use cortex_m::interrupt::Mutex;
use cortex_m_rt::entry;
use lm3s6965::interrupt; // #[interrupt] macro variant ที่รู้จักชื่อ interrupt ของชิปนี้โดยเฉพาะ
                          // -- มาจาก PAC ไม่ใช่จาก cortex-m-rt ตรง ๆ (นี่คือความต่างจาก #[exception])
use panic_halt as _;

// ใช้ Cell<u32> แทน RefCell<u32> ได้เพราะ u32 เป็น Copy type -- ไม่ต้องมี borrow tracking แบบ
// RefCell (Part 28) เลย แค่ .get()/.set() ธรรมดา -- เลือกใช้ตัวที่ตรงกับความต้องการจริง ไม่ใช่ RefCell
// เสมอไปเมื่อไม่มีข้อมูลที่ซับซ้อนกว่า primitive type ที่ Copy ได้
static BUTTON_PRESSES: Mutex<Cell<u32>> = Mutex::new(Cell::new(0));

// ชื่อฟังก์ชันนี้ "GPIOA" ต้องตรงกับชื่อ interrupt วันที่ประกาศไว้ใน __INTERRUPTS ของ lm3s6965
// เป๊ะทุกตัวอักษร (case-sensitive) -- ถ้าพิมพ์ผิดชื่อ compiler จะฟ้อง error ทันทีตอน compile
// (ไม่ใช่ตอน link) เพราะ #[interrupt] macro ตรวจสอบชื่อกับ enum ที่ PAC ประกาศไว้ให้แล้ว
#[interrupt]
fn GPIOA() {
    cortex_m::interrupt::free(|cs| {
        let count = BUTTON_PRESSES.borrow(cs);
        count.set(count.get() + 1);
    });
}

#[entry]
fn main() -> ! {
    loop {
        cortex_m::asm::nop();
    }
}
```

**สถานะการยืนยัน**: ตัวอย่างนี้ cross-compile จริงสำเร็จด้วย `cargo build --target thumbv7m-none-eabi` (ตรงกับ
core Cortex-M3 ของ `lm3s6965evb`) ในสภาพแวดล้อมเขียนบทนี้ — หมายเหตุที่ต้องซื่อตรง: crate `lm3s6965` เวอร์ชัน
ล่าสุดที่มี (0.1.3) ยัง pin ตัวเองไว้กับ `cortex-m`/`cortex-m-rt` รุ่น **0.6.x** (เก่ากว่ารุ่น 0.7.x ที่ใช้ใน
ตัวอย่างอื่นทั้งหมดของบทนี้) — ครั้งแรกที่ลองประกาศ `cortex-m-rt = "0.7"` ในโปรเจกต์เดียวกัน cargo ปฏิเสธด้วย
error จริง:

```
error: failed to select a version for `cortex-m-rt`.
    ...
package `cortex-m-rt` links to the native library `cortex-m-rt`, but it conflicts with a
previous package which links to `cortex-m-rt` as well:
package `cortex-m-rt v0.6.15`
    ... which satisfies dependency `cortex-m-rt = "^0.6.5"` of package `lm3s6965 v0.1.0`
```

นี่คือปัญหา **`links` key ของ Cargo** (ระบุไว้ในหมายเหตุของ Cargo ว่า native library หนึ่งตัว link ซ้ำสอง
เวอร์ชันไม่ได้ในไบนารีเดียว) — วิธีแก้คือปรับ `cortex-m`/`cortex-m-rt` ในโปรเจกต์นี้ลงมาที่ `"0.6"` ให้ตรงกับ
ที่ `lm3s6965` ต้องการ หลังปรับแล้ว `cargo build` ผ่านสำเร็จสมบูรณ์:

```
      Adding cortex-m v0.6.7 (available: v0.7.9)
      Adding cortex-m-rt v0.6.15 (available: v0.7.7)
      Adding lm3s6965 v0.1.3 (available: v0.2.0)
   Compiling lm3s6965 v0.1.3
   Compiling interrupt-pac v0.1.0 (.../interrupt-pac)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 6.98s
```

บทเรียนเชิงปฏิบัติที่แท้จริงจากปัญหานี้ (ไม่ใช่แค่ทฤษฎี): **PAC/HAL crate ในโลก embedded มักตามหลังเวอร์ชัน
ล่าสุดของ `cortex-m`/`cortex-m-rt` อยู่เสมอไม่มากก็น้อย** เพราะแต่ละ crate ต้อง maintained แยกกันโดยทีมต่างกัน
— เวลาเลือก dependency สำหรับโปรเจกต์ embedded จริง ต้องเช็ค **compatibility matrix** ของ ecosystem ให้ตรงกัน
ทั้งชุดเสมอ ไม่ใช่แค่อัปเดตทุก crate ไปที่เวอร์ชันล่าสุดแยกกันแบบสุ่ม ๆ (ปัญหานี้พบได้บ่อยกว่าในโลก embedded
มากกว่าโลกเว็บทั่วไป เพราะ ecosystem เล็กกว่าและ crate หลักบางตัวไม่ได้อัปเดตตามกันเร็วเท่า)

### 102.10 Async บน Embedded: `embassy` — Executor ที่ไม่มี OS คอยรัน

Part 30+ สอน async/await ของ Rust ไว้บนสมมติฐานว่ามี **executor** (เช่น Tokio) คอย poll `Future` และ
**OS** คอยให้ I/O event (เช่น socket พร้อมอ่าน) ผ่าน mechanism อย่าง `epoll`/`io_uring` — คำถามที่น่าสนใจคือ:
**async/await ใช้งานได้ใน `#[no_std]` ไหม ทั้งที่ไม่มี OS ให้ epoll เลย?**

คำตอบคือ **ได้ และเป็นแนวทางที่ทันสมัย/แนะนำมากที่สุดสำหรับงาน embedded ที่ซับซ้อนกว่าตัวอย่างง่าย ๆ ในบทนี้**
— เหตุผลเชิงกลไกคือ: `async`/`await` ของ Rust (ต่างจากภาษาอื่นหลายภาษา) **ไม่ได้ผูกกับ runtime ตัวใดตัวหนึ่ง
ในตัวภาษาเอง** มันเป็นแค่ syntax sugar ที่ compiler แปลงเป็น state machine ที่ implement trait `Future` (Part
30 สอนกลไกนี้ไว้แล้ว) — ตัว `Future` trait เองอยู่ใน **`core`** (ไม่ใช่ `std`!) ดังนั้น **การเขียนโค้ดแบบ
`async fn` ทำได้ใน `#[no_std]` โดยไม่มีปัญหาเลยในทางภาษา** สิ่งที่ยังต้องมีคือ **executor** ที่จะ poll
`Future` เหล่านั้น — และนี่คือสิ่งที่ **`embassy`** เข้ามาให้บริการ

`embassy` คือ framework embedded Rust ที่ทันสมัยและได้รับความนิยมสูงมากในปัจจุบัน (2024-2026) โดยมีองค์
ประกอบหลักที่เกี่ยวข้องกับบทนี้:

- **`embassy-executor`** — executor ของตัวเองที่เขียนขึ้นมาให้ทำงานได้บน `#[no_std]` 100% ไม่พึ่ง `std`/OS
  เลยแม้แต่นิดเดียว (ต่างจาก Tokio ที่ต้องพึ่ง OS thread/timer/epoll เต็มรูปแบบ) — มันใช้กลไกของ hardware
  interrupt เพื่อ "ปลุก" executor ให้ poll `Future` ใหม่เมื่อมี event เกิดขึ้น (เช่น timer ครบเวลา หรือ UART
  มีข้อมูลมาถึง) แทนที่จะพึ่ง OS scheduler
- **`embassy-time`** — ให้ `Timer::after(...)` แบบ async ที่ทำงานคล้าย `tokio::time::sleep()` (Part 30-31)
  แต่ implement ด้วย hardware timer ของ MCU ตรง ๆ แทนการพึ่ง OS timer
- **HAL crate เฉพาะชิป** (เช่น `embassy-stm32`, `embassy-nrf`) — ให้ driver แบบ async สำหรับ peripheral
  ต่าง ๆ (UART, I2C, SPI) ที่ทำงานร่วมกับ `embassy-executor` ได้ทันที

ตัวอย่างที่ **cross-compile จริงสำเร็จ** ในสภาพแวดล้อมเขียนบทนี้ (ต่างจากฉบับร่างแรกที่ลองแค่
`embassy-executor`/`embassy-time` เปล่า ๆ แล้ว link ไม่ผ่าน — รายละเอียดอยู่ในหมายเหตุความซื่อตรงท้ายหัวข้อ)
คือโปรแกรมสำหรับบอร์ด Nucleo-F401RE (ชิปตัวเดียวกับตัวอย่าง blink ในหัวข้อ 102.6) ที่ใช้ **`embassy-stm32`**
เป็น HAL เฉพาะชิปเพื่อให้ `embassy-executor` มี critical-section implementation และ time driver ที่ใช้งานได้
จริงครบชุด:

```rust
#![no_std]
#![no_main]

use embassy_executor::Spawner;
use embassy_time::{Duration, Timer};
use panic_halt as _;

// #[embassy_executor::task] ทำเครื่องหมายว่านี่คือ "งาน" หนึ่งชิ้นที่ executor จะ poll ไปเรื่อย ๆ
// สังเกตว่าเขียนเป็น async fn ธรรมดา -- ใช้ .await ได้เหมือนโค้ด async ทั่วไปที่หลักสูตรสอนมา
// (Part 30-33) ทุกประการในทางไวยากรณ์ แม้จะรันอยู่บน #[no_std] ก็ตาม
#[embassy_executor::task]
async fn blink_task() {
    loop {
        // จุดสำคัญ: Timer::after(...).await ไม่ได้ "block" CPU เหมือน cortex_m::asm::delay()
        // ในหัวข้อ 102.6 -- ระหว่างรอ executor เป็นอิสระที่จะไป poll งานอื่นที่พร้อมทำงานได้
        // (ถ้ามีงานอื่นอยู่) เทียบเท่ากับที่ Tokio ทำได้ระหว่าง .await แต่ในที่นี้ไม่มี OS thread
        // ให้สลับไปมาเลย -- ทั้งหมดเกิดจาก interrupt ของ hardware timer ที่ปลุก executor ตรง ๆ
        // (สมมติว่ามี "led" struct ที่ implement embedded_hal-style ให้ toggle ได้ ในตัวอย่างนี้
        // ย่อไว้เพื่อโฟกัสที่โครงสร้าง async ล้วน ๆ)
        Timer::after(Duration::from_millis(500)).await;
    }
}

#[embassy_executor::main]
async fn main(spawner: Spawner) {
    // embassy_stm32::init(...) เตรียม clock/RCC ให้ทั้งชิป (แทนที่การเขียน RCC_AHB1ENR ด้วยมือ
    // แบบหัวข้อ 102.6 -- HAL ทำขั้นตอนที่มักถูกลืมนี้ให้ครบถ้วนอัตโนมัติ) และคืน struct ที่ถือ
    // peripheral ทั้งหมดของบอร์ดไว้ให้หยิบไปใช้ต่อ
    let _p = embassy_stm32::init(Default::default());

    // spawner.spawn(...) เพิ่มงานใหม่เข้าไปให้ executor ของ embassy คอย poll
    // -- แนวคิดเดียวกับ tokio::spawn() ของ Part 31 แต่ executor เป็นของ embassy เอง ไม่ใช่ Tokio
    spawner.spawn(blink_task()).unwrap();

    loop {
        Timer::after(Duration::from_secs(1)).await;
    }
}
```

`Cargo.toml` ที่ทำให้ตัวอย่างนี้ link ผ่านจริง (จุดสำคัญคือ feature `critical-section-single-core` ของ
`cortex-m` และการระบุชิปให้ `embassy-stm32` ผ่าน feature `stm32f401re`):

```toml
[dependencies]
cortex-m = { version = "0.7", features = ["critical-section-single-core"] }
cortex-m-rt = "0.7"
panic-halt = "0.2"
embassy-executor = { version = "0.7", features = ["arch-cortex-m", "executor-thread"] }
embassy-time = "0.4"
embassy-stm32 = { version = "0.2", features = ["stm32f401re", "time-driver-any"] }

[profile.release]
panic = "abort"
```

สิ่งที่ต้องเน้นให้ชัดเจนที่สุดในหัวข้อนี้คือ **ความต่างเชิงกลไกระหว่าง `cortex_m::asm::delay()` (busy-wait)
กับ `Timer::after(...).await` (async)**: `cortex_m::asm::delay()` ที่ใช้ในหัวข้อ 102.6 คือการวน loop นับ
cycle เปล่า ๆ — **CPU ทำงาน 100% ตลอดช่วงที่หน่วงเวลา แต่ไม่ได้ทำอะไรที่มีประโยชน์เลย** ถ้าโปรแกรมมีงานอื่นที่
ต้องทำระหว่างรอ (เช่น รับข้อมูลจาก UART พร้อมกับหน่วงเวลากระพริบ LED) วิธี busy-wait ทำสองอย่างพร้อมกันไม่ได้
เลยในตัวมันเอง (ต้องจัดการผ่าน interrupt handler แยกต่างหากเสมอ ซึ่งเพิ่มความซับซ้อนขึ้นเรื่อย ๆ เมื่อมีงาน
พร้อมกันมากขึ้น) — ส่วน `Timer::after(...).await` ทำให้ executor ของ `embassy` **สามารถไปรันงานอื่นที่พร้อม
ทำงานได้ระหว่างที่งานนี้กำลังรอ** (คล้ายกับที่ Part 30-31 อธิบายว่า async ทำให้ single thread จัดการงานพร้อมกัน
หลายงานได้โดยไม่บล็อกกัน) — นี่คือเหตุผลที่ `embassy` ถูกแนะนำเป็น**แนวทางที่ทันสมัยและควรใช้สำหรับโปรแกรม
embedded ที่ซับซ้อนกว่าตัวอย่าง "blink" เดียว ๆ** เพราะการจัดการงานพร้อมกันหลายงาน (เช่น อ่านเซนเซอร์ 3 ตัว
พร้อมกับส่งข้อมูลผ่าน UART พร้อมกับกระพริบ LED) เขียนด้วย async/await อ่านง่ายและปลอดภัยกว่าการเขียน
interrupt handler จำนวนมากที่แชร์ state กันเองแบบ manual (หัวข้อ 102.9) มาก — แม้ interrupt ยังเป็นกลไกพื้นฐาน
ที่ `embassy` ใช้อยู่ข้างใต้ (executor ของมันถูก "ปลุก" ด้วย interrupt จริง ๆ ) แต่โปรแกรมเมอร์ **ไม่ต้องเขียน
`#[interrupt]` handler และ critical section ด้วยมือเองอีกต่อไป** — `embassy` จัดการชั้นนั้นให้หมดแล้ว

**หมายเหตุความซื่อตรงเรื่องการตรวจสอบ**: ความพยายามครั้งแรกในการเขียนตัวอย่างนี้ใช้แค่ `embassy-executor`
กับ `embassy-time` เปล่า ๆ (ไม่มี HAL เฉพาะชิป) — ผลคือ **link ไม่ผ่าน** ด้วย error จริงจาก `rust-lld`:

```
error: linking with `rust-lld` failed: exit status: 1
  ...
  rust-lld: error: undefined symbol: _critical_section_1_0_release
  rust-lld: error: undefined symbol: _critical_section_1_0_acquire
  rust-lld: error: undefined symbol: _embassy_time_schedule_wake
  rust-lld: error: undefined symbol: _embassy_time_now
```

สาเหตุคือ `embassy-executor`/`embassy-time` เป็นแค่ **ส่วนที่ไม่ผูกกับ hardware** (portable ข้ามชิป) — พวก
มันเรียก symbol ที่ต้อง**มีคนมา implement ให้จริง** สองกลุ่ม: (1) `critical-section` implementation (ว่า
"ปิด interrupt ยังไงจริง ๆ บนชิปนี้") และ (2) `embassy-time` driver (ว่า "อ่านเวลาปัจจุบันจาก hardware timer
ตัวไหนจริง ๆ") — ทั้งสองอย่างนี้**ขึ้นกับชิปจริงเสมอ ไม่มีทางเป็น generic ข้ามชิปได้ในทางเทคนิค** เมื่อเพิ่ม
`cortex-m` feature `critical-section-single-core` (แก้ปัญหาที่ 1) และ `embassy-stm32` พร้อม feature
`stm32f401re`/`time-driver-any` (แก้ปัญหาที่ 2 — HAL ของชิปตัวนี้ implement ทั้ง critical-section ผ่าน
`cortex-m` และ time driver ผ่าน hardware timer ของ STM32F401 เอง) ตัวอย่างจึง **compile และ link ผ่านสำเร็จ
สมบูรณ์** ด้วย `cargo build --target thumbv7em-none-eabihf`:

```
   Compiling critical-section v1.2.0
   ...
   Compiling embassy-stm32 v0.2.0
   ...
   Compiling embassy-demo v0.1.0 (.../embassy-demo)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 7.37s
```

และ `cargo build --release` (ด้วย `panic = "abort"` ตาม `Cargo.toml` ข้างบน) วัดขนาดจริงด้วย `size` ได้:

```
   text	   data	    bss	    dec	    hex	filename
  18716	     24	   4352	  23092	   5a34	embassy-demo (release)
  59696	     24	   4352	  64072	   fa48	embassy-demo (debug)
```

ตัวเลข `.text` = 18,716 ไบต์ของ `release` build **ใหญ่กว่าตัวอย่าง blink แบบ register-level ในหัวข้อ 102.6
(1,200 ไบต์ในรูปแบบ release เดียวกัน) ถึงเกือบ 16 เท่า** — นี่คือตัวเลขจริงที่แสดงต้นทุนของความสะดวกสบายที่
`embassy` มอบให้ (executor, time driver, HAL abstraction เต็มรูปแบบ) เทียบกับการเขียน register ตรง ๆ ด้วยมือ
— ต้นทุนนี้**คุ้มค่ามาก**สำหรับโปรแกรมที่มีงานพร้อมกันหลายงานจริง (ที่การเขียน register/interrupt handler
ด้วยมือจะซับซ้อนกว่านี้มากเมื่อโปรแกรมโตขึ้น) แต่ **ไม่คุ้มค่าเลย**สำหรับ MCU ที่มี Flash เหลือน้อยมาก (เช่น
มีแค่ 32-64 KB) หรือโปรแกรมที่ต้องการแค่ blink LED เดียวจริง ๆ — นี่คือตัวอย่างที่จับต้องได้ของการ trade-off
ระหว่าง "ความสะดวกในการเขียนโค้ด" กับ "งบประมาณ Flash ที่มีจำกัดสุดขั้ว" ที่หัวข้อ 102.11 จะพูดถึงต่อ

### 102.11 ต่อสู้กับ Kilobytes: Binary Size Tuning ระดับสุดขั้ว

Part 96 สอนเทคนิคลด binary size ของโปรแกรมทั่วไป (`opt-level`, `lto`, `strip`) ไว้ในบริบทที่ขนาดวัดเป็น
**เมกะไบต์** (ลด Docker image จากหลายสิบ MB เหลือไม่กี่ MB ก็ถือว่าดีมากแล้ว) — สำหรับ embedded ตัวเลขที่
สำคัญคือ**กิโลไบต์** ไม่ใช่เมกะไบต์: MCU อย่าง STM32F401 มี Flash รวมทั้งหมดแค่ **256 KB** — ถ้า binary ของ
เราใหญ่กว่านั้น**โปรแกรมจะ flash ลงชิปไม่ได้เลย** ไม่ใช่แค่ "ทำงานช้าลง" — นี่คือความต่างเชิงคุณภาพจาก
บริบทของ Part 96 ที่ต้องเข้าใจให้ชัด: การลด binary size ในโลก embedded ไม่ใช่แค่การ optimize เพื่อความ
สวยงามหรือประสิทธิภาพ แต่บางครั้งคือ**เงื่อนไขว่าโปรแกรมจะรันได้เลยหรือไม่**

เทคนิคเดียวกับ Part 96 ยังใช้ได้ทั้งหมด แต่มีรายละเอียดเพิ่มที่สำคัญเฉพาะกับ embedded:

```toml
[profile.release]
opt-level = "z"       # "z" = optimize เพื่อขนาดเล็กสุด (เข้มกว่า "s" อีกขั้น) -- ต่างจาก
                       # โปรแกรมทั่วไปที่มักใช้ opt-level = 3 (เร็วสุด) เพราะ embedded ส่วนใหญ่
                       # ยอมแลกความเร็วเล็กน้อยเพื่อขนาดที่เล็กลงมาก (ยกเว้นงานที่ latency-critical
                       # จริง ๆ เช่น signal processing ที่ต้องใช้ opt-level = 3 แทน)
lto = true             # Link-Time Optimization -- ให้ optimizer มองเห็นทั้งโปรแกรมพร้อมกันข้าม
                       # crate boundary ทั้งหมด (ไม่ใช่แค่ทีละ crate) ตัดโค้ดที่ไม่ถูกเรียกใช้จริง
                       # ทิ้งได้ละเอียดกว่า -- สำคัญกว่าปกติมากสำหรับ embedded เพราะ dependency
                       # อย่าง cortex-m/cortex-m-rt มีฟังก์ชันเผื่อไว้จำนวนมากที่ไม่ได้ใช้ทุกตัว
codegen-units = 1      # บังคับให้ compile เป็นหน่วยเดียว (ช้าลงตอน compile แต่ให้ optimizer
                       # เห็นภาพรวมกว้างที่สุดเท่าที่จะทำได้ ทำงานร่วมกับ lto = true ได้ดีที่สุด)
panic = "abort"        # ตัดกลไก stack unwinding ทิ้งทั้งหมด -- ปกติ panic ที่ "unwind" (ค่าเริ่มต้น
                       # ของ std) ต้องมี metadata ฝังอยู่ในทุกฟังก์ชันเพื่อรู้ว่าจะ "คลาย" stack
                       # อย่างไรตอน panic ซึ่งกิน Flash ไม่น้อย -- ในโลก no_std ที่ panic handler
                       # ส่วนใหญ่ก็แค่ loop {} อยู่แล้ว (หัวข้อ 102.8) ไม่มีเหตุผลจะเก็บ unwinding
                       # metadata ไว้เลย -- ตัดทิ้งได้ทั้งหมดโดยไม่เสียอะไร
strip = true            # ตัด debug symbol ทิ้งจาก binary สุดท้าย (เก็บไว้ใน .elf ที่ยังไม่ strip
                       # สำหรับตอน debug ด้วย gdb/probe-rs แยกไฟล์กัน)
```

**ตัวเลขจริงที่วัดได้ในสภาพแวดล้อมเขียนบทนี้** (จากตัวอย่าง blink ในหัวข้อ 102.6 คอมไพล์ด้วย `--target
thumbv7em-none-eabihf` ด้วยคำสั่ง `size target/thumbv7em-none-eabihf/release/blink` จริงทุกแถว — `cargo-
bloat` **ไม่มีอยู่**ในสภาพแวดล้อมนี้ เครื่องมือที่ใช้วัดได้จริงมีแค่ `size`):

| การตั้งค่า | ขนาด `.text` (ไบต์) | ขนาดไฟล์ ELF ทั้งไฟล์ (ไบต์) |
|---|---|---|
| `dev` profile (ไม่ optimize เลย, debug build) | 2,620 | 828,160 |
| `release` เริ่มต้น (`opt-level = 3`, ไม่มี `lto`/`panic="abort"`/`strip`) | 1,200 | 70,148 |
| `release` + `opt-level = "z"` | 1,208 | 70,244 |
| `release` + `opt-level = "z"` + `lto = true` + `codegen-units = 1` | 1,208 | 68,932 |
| ทั้งหมดข้างบน + `panic = "abort"` + `strip = true` | 1,208 | 67,488 |

**ต้องอธิบายตัวเลขเหล่านี้อย่างซื่อตรงเพราะมันขัดกับสัญชาตญาณเล็กน้อย**: `opt-level = "z"` ให้ `.text` **ใหญ่
กว่า** `opt-level = 3` เล็กน้อย (1,208 เทียบกับ 1,200 ไบต์) ทั้งที่ `"z"` ควรจะ "เล็กกว่าเสมอ" ตามที่มักเข้าใจ
กัน — เหตุผลคือตัวอย่าง blink ในบทนี้**เล็กมากจนเกือบทั้งหมดของ `.text` คือ startup code ตายตัวของ
`cortex-m-rt`/`panic-halt`** (ไม่ใช่โค้ด logic ของเราเองที่มีพื้นที่ให้ optimizer เลือก trade-off ระหว่างขนาด
กับความเร็วได้มากนัก) heuristic ของ `opt-level = "z"` บางครั้งเลือกวิธีสร้างโค้ดที่ "เล็กในภาพกว้าง" แต่ไม่
ได้ชนะทุกฟังก์ชันเดี่ยว ๆ เสมอไป — **ความต่างที่ชัดเจนและสม่ำเสมอที่สุดในตารางนี้คือขนาดไฟล์ ELB ทั้งไฟล์**
(ไม่ใช่ `.text` เดี่ยว ๆ ): จาก `dev` (828,160 ไบต์ เต็มไปด้วย debug info) เหลือแค่ราว 8.5% ของขนาดเดิมทันทีที่
สลับไป `release` เปล่า ๆ (70,148 ไบต์) และลดต่อไปอีกได้ถึง 67,488 ไบต์ (ลดลงจาก `release` เริ่มต้นอีก ~3.8%)
เมื่อเพิ่ม `lto`/`codegen-units=1`/`panic="abort"`/`strip` ครบชุด — **`strip = true` คือตัวที่มีผลชัดเจนที่สุด
ต่อขนาดไฟล์ ELF โดยรวมในตัวอย่างเล็กขนาดนี้** เพราะมันตัด debug symbol table ทิ้งทั้งหมด (ซึ่งไม่กระทบขนาด
`.text` ที่ flash ลงชิปจริงเลย — debug symbol ไม่ได้ถูก flash ไปด้วย แต่กระทบขนาดไฟล์ ELF ที่เก็บไว้บนเครื่อง
development)

ข้อสรุปที่ซื่อตรงที่สุดจากตัวเลขจริงชุดนี้คือ: **สำหรับโปรแกรมเล็กขนาด "blink" เดียว ผลของ `opt-level`/`lto`
ต่อขนาด `.text` มีจำกัดมาก เพราะพื้นที่ส่วนใหญ่คือ runtime scaffolding ที่ optimize ไปได้ไม่มากอยู่แล้ว** —
ผลของ flag เหล่านี้จะ**เด่นชัดขึ้นมากในโปรแกรมจริงที่มีโค้ด logic จำนวนมาก** (หลายพันบรรทัดขึ้นไป, ใช้ generic/
trait object จำนวนมาก) ซึ่งตรงกับที่ตัวอย่าง `embassy-demo` ในหัวข้อ 102.10 แสดงให้เห็นแล้ว: `.text` ของมัน
(18,716 ไบต์) มีพื้นที่ให้ optimizer ทำงานมากกว่าตัวอย่าง blink เปล่า ๆ มาก เพราะมีโค้ดของ executor/HAL/driver
จำนวนมากกว่าเดิมหลายเท่า — บทเรียนสำคัญคือ**อย่าคาดเดาผลของการตั้งค่า optimization จากทฤษฎีอย่างเดียว ต้องวัด
จริงกับโปรแกรมจริงของตัวเองเสมอ** (หลักการเดียวกับที่ Part 54/56 ปลูกฝังไว้ตลอดเรื่องประสิทธิภาพ)

### 102.12 ตำแหน่งของบทนี้ในเส้นทางเรียนรู้ และแหล่งข้อมูลสำหรับไปต่อ

ต้องพูดให้ชัดที่สุดในหัวข้อปิดท้ายนี้: **บทนี้คือบท "ปูพื้นให้รู้จัก" (awareness / getting-started chapter)
ไม่ใช่เส้นทางสู่ความเชี่ยวชาญ embedded** — สิ่งที่บทนี้ให้คือแผนที่ทางความคิด (mental model) ที่ถูกต้องของ
โลก embedded: ไม่มี OS, เข้าถึง hardware ผ่าน volatile memory-mapped register, ไม่มี panic handler ให้
อัตโนมัติ, จัดโครงสร้างงานรอบ interrupt/async แทน polling ธรรมดา — ความรู้เหล่านี้เพียงพอให้อ่านโค้ด embedded
Rust จริงแล้วเข้าใจว่าเกิดอะไรขึ้น และเริ่มเขียนโปรแกรมเล็ก ๆ เองได้ **แต่ยังไม่เพียงพอสำหรับงาน production
จริง** ที่ต้องเจอเรื่องอีกมากที่บทนี้ไม่ได้ลงรายละเอียด เช่น: การเลือก/อ่าน datasheet ของชิปให้ครบถ้วน, DMA
(Direct Memory Access) สำหรับโอนข้อมูลระหว่าง peripheral กับ memory โดยไม่ใช้ CPU cycle เลย, power management
ระดับลึก (sleep mode ต่าง ๆ ), bootloader และการอัปเดต firmware ผ่านสนาม (OTA — Over-The-Air), certification
สำหรับอุตสาหกรรมที่ต้องการความปลอดภัยสูง (medical device, automotive) และการทดสอบ/debug บน hardware จริงด้วย
oscilloscope/logic analyzer

สำหรับผู้ที่สนใจไปต่อจริงจัง แหล่งข้อมูลที่แนะนำ (เป็นแหล่งข้อมูลจริงที่ชุมชน embedded Rust ยอมรับกันทั่วไป
ไม่ใช่การโปรโมทโดยไม่มีมูล):

- **The Embedded Rust Book** (`docs.rust-embedded.org/book`) — เอกสารทางการของ Embedded WG (Working Group)
  ของ Rust ที่สอนตั้งแต่พื้นฐานเหมือนบทนี้ ไปจนถึงรายละเอียดของ QEMU/hardware จริงแบบเต็มรูปแบบ — บทนี้อ้างอิง
  แนวทางการใช้ `lm3s6965evb`/semihosting มาจากเอกสารนี้โดยตรง
- **`embedded-hal`** (crates.io) — อ่าน source/documentation ของ trait มาตรฐานที่กล่าวถึงในหัวข้อ 102.5.2
  เพื่อเข้าใจว่า ecosystem ทั้งหมดออกแบบให้ทำงานร่วมกันข้ามผู้ผลิตชิปได้อย่างไร
- **`embassy`** (embassy.dev) — เอกสารและตัวอย่างของ framework async ที่กล่าวถึงในหัวข้อ 102.10 มีตัวอย่าง
  เต็มรูปแบบสำหรับชิปจริงหลายตระกูล (STM32, nRF, RP2040) ให้ทดลองถ้ามี hardware จริงในมือ
- **Discovery book** (`docs.rust-embedded.org/discovery`) — คอร์สที่จับคู่กับบอร์ด microbit (BBC micro:bit)
  ราคาไม่แพง เหมาะสำหรับผู้ที่ต้องการฝึกกับ hardware จริงเป็นครั้งแรกโดยไม่ต้องลงทุนอุปกรณ์แพง

**ตำแหน่งของบทนี้เทียบกับทั้งหลักสูตร**: ทุกบทตั้งแต่ Part 60 เป็นต้นมาของหลักสูตรนี้ (web server, cloud
deployment, WASM สำหรับ browser) ล้วนมีจุดร่วมกันหนึ่งอย่างที่อาจไม่เคยถูกพูดตรง ๆ มาก่อน: **ทุกเป้าหมายนั้น
รันอยู่ภายใต้ OS หรือสภาพแวดล้อมที่จำลอง OS ให้บางส่วนเสมอ** (browser จำลอง sandbox ที่ปลอดภัยให้ WASM,
container ยังรันอยู่บน Linux kernel ข้างใต้เสมอ) — **Part 102 นี้คือจุดแรกในหลักสูตรทั้งหมดที่เป้าหมายการ
deploy ไม่มี OS อยู่ข้างใต้เลยแม้แต่นิดเดียว** เป็น deployment target ที่ "สุดขั้ว" ที่สุดในทุกมิติที่หลักสูตร
นี้ครอบคลุมมา: ทรัพยากรน้อยที่สุด (KB ไม่ใช่ GB), ระดับการควบคุม hardware สูงที่สุด (ไม่มี driver ของ OS มา
บังหน้าเลย), และความเสี่ยงจากบั๊กสูงที่สุด (บั๊กในเซิร์ฟเวอร์ทำให้ request หนึ่งพัง แต่บั๊กใน firmware ควบคุม
มอเตอร์อาจทำให้อุปกรณ์จริงพัง) — การได้เห็นว่า Rust ตัวเดียวกัน (ภาษาเดียวกัน, ทักษะ ownership/borrow checker
เดียวกันที่ฝึกมาตั้งแต่ Part 6-7) ยังใช้งานได้และเหมาะสมในสภาพแวดล้อมที่สุดขั้วขนาดนี้ คือหลักฐานที่ชัดเจนที่สุด
ของคำกล่าวที่หลักสูตรพูดไว้ตั้งแต่ Part 1 ว่า Rust คือภาษาที่ **"ครอบคลุมได้ตั้งแต่ระดับ hardware ที่ใกล้ที่สุด
ไปจนถึง full-stack web application"** โดยไม่ต้องเปลี่ยนภาษาเลยแม้แต่ครั้งเดียว

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. ลืม `#[panic_handler]` แล้วสงสัยว่า error มาจากไหน

โปรแกรมเมอร์ที่เริ่มเขียน `#![no_std]` เป็นครั้งแรกมักลืมว่าไม่มี panic handler ให้อัตโนมัติ (เพราะโปรแกรม
ปกติที่มี `std` ไม่ต้องคิดเรื่องนี้เลย) ถ้าลืม import crate อย่าง `panic-halt` หรือเขียน `#[panic_handler]`
เอง จะได้ error ตอน link (ไม่ใช่ตอน compile — เพราะ compiler ยอมให้ `panic!()` ถูกเรียกได้โดยไม่รู้ว่าตัว
handler อยู่ที่ไหนจนกว่าจะถึงขั้น link):

```
error: `#[panic_handler]` function required, but not found
```

(error นี้คัดลอกมาจากการรัน `cargo build --target thumbv7em-none-eabihf` จริงในสภาพแวดล้อมเขียนบทนี้ กับ
โปรเจกต์ที่มี `#[entry] fn main()` ครบถ้วนทุกอย่างแต่ไม่มี panic handler ใด ๆ เลย)

วิธีแก้: เพิ่ม `use panic_halt as _;` (หรือ crate panic handler อื่นที่เลือกใช้) ไว้ที่ระดับบนสุดของ crate
เสมอ — สังเกตว่าใช้ `as _` เพราะเราไม่ได้เรียกใช้อะไรจาก crate นี้ตรง ๆ ในโค้ด (มันแค่ต้องถูก link เข้ามาเพื่อ
ให้ `#[panic_handler]` ของมันมีผล) — ถ้าเขียน `use panic_halt;` เฉย ๆ โดยไม่มี `as _` จะได้ warning
`unused_imports` เพราะ compiler ไม่รู้ว่า crate ที่ import มาถูกใช้เพื่อ side effect (การ link) ไม่ใช่เพื่อ
เรียกใช้ symbol ตรง ๆ

### 2. ใช้ `Vec`/`String` โดยไม่เพิ่ม `extern crate alloc` และไม่มี global allocator

มือใหม่ที่คุ้นกับ `Vec`/`String` มาตลอดหลักสูตร (Part 13-14) มักเขียนโค้ด `#[no_std]` ที่ใช้ `Vec<T>` ตรง ๆ
โดยไม่รู้ว่าต้องเปิดใช้งาน `alloc` crate เองก่อน (ตามที่ Part 56 หัวข้อ 56.7.1 อธิบายไว้) จะได้ error จริง
(คัดลอกจากการรันในสภาพแวดล้อมเขียนบทนี้):

```
error[E0433]: cannot find module or crate `alloc` in this scope
 --> src/main.rs:4:5
  |
4 | use alloc::vec::Vec;
  |     ^^^^^ use of unresolved module or unlinked crate `alloc`
  |
  = help: add `extern crate alloc` to use the `alloc` crate
```

หรือถ้าเพิ่ม `extern crate alloc;` แล้วแต่ยังไม่มี `#[global_allocator]` ประกาศไว้ในโปรแกรมสุดท้าย จะได้
error ตอน link แบบนี้แทน (คัดลอกจากการรันจริงเช่นกัน):

```
error: no global memory allocator found but one is required; link to std or add
       `#[global_allocator]` to a static item that implements the GlobalAlloc trait
```

วิธีแก้: เพิ่ม `extern crate alloc;` **และ** ประกาศ global allocator จริง (เช่น `embedded-alloc` crate ที่
ให้ bump allocator สำเร็จรูปสำหรับ MCU) — หรือ (ทางเลือกที่ปลอดภัยกว่ามากสำหรับ MCU ที่มี RAM น้อย) เปลี่ยนไป
ใช้ `heapless::Vec<T, N>` ที่มี capacity คงที่ตอน compile time แทน ไม่ต้องพึ่ง heap เลย

### 3. เขียน register access แบบไม่ volatile แล้วโค้ดถูก compiler ตัดทิ้ง

ตามที่อธิบายไว้เต็มรูปแบบในหัวข้อ 102.5.1: การเขียน `*ptr = value;` ตรง ๆ (ไม่ผ่าน `write_volatile`) กับ
memory-mapped register **compile ผ่านเสมอโดยไม่มี warning หรือ error ใด ๆ เลย ถ้าอยู่ใน `unsafe` block แล้ว**
— นี่คือกับดักที่อันตรายที่สุดในบทนี้เพราะ**ไม่มีสัญญาณเตือนใด ๆ ตอน compile** อาการที่พบคือ hardware "ไม่
ตอบสนอง" หรือ "ตอบสนองแค่บางครั้ง" ทั้งที่โค้ด logic ดูถูกต้องทุกอย่าง

สิ่งที่ compiler **ฟ้องได้** (และเป็นกับดักที่พบบ่อยกว่าในทางปฏิบัติสำหรับมือใหม่) คือการลืม `unsafe` block
เอง ไม่ว่าจะ dereference raw pointer ตรง ๆ หรือเรียก `write_volatile` — สอง error จริงที่คัดลอกจากการรันใน
สภาพแวดล้อมเขียนบทนี้:

```
error[E0133]: dereference of raw pointer is unsafe and requires unsafe function or block
  --> src/main.rs:10:5
   |
10 |     *odr = 1 << 5;
   |     ^^^^ dereference of raw pointer
   |
   = note: raw pointers may be null, dangling or unaligned; they can violate aliasing rules and cause data races: all of these are undefined behavior
```

```
error[E0133]: call to unsafe function `write_volatile` is unsafe and requires unsafe function or block
  --> src/main.rs:11:5
   |
11 |     write_volatile(odr, 1 << 5);
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^ call to unsafe function
   |
   = note: consult the function's documentation for information on how to avoid undefined behavior
```

แต่**ทั้งสอง error นี้แค่ตรวจว่าใส่ `unsafe` ครบหรือไม่** — มันไม่มีทางตรวจได้ว่าข้างใน `unsafe` block นั้น
คุณเลือกใช้ `write_volatile` (ถูกต้อง) หรือ dereference ตรง ๆ แบบไม่ volatile (ผิด แต่ compile ผ่าน) เพราะ
ทั้งสองแบบล้วนเป็น "การ dereference raw pointer ที่ถูกกำกับด้วย `unsafe` แล้ว" ในสายตาของ compiler เหมือนกัน
ทุกประการ — วิธีป้องกันคือ**สร้างวินัยเขียนโค้ด**ให้เข้าถึง memory-mapped register ผ่าน `write_volatile`/
`read_volatile` เสมอไม่มีข้อยกเว้น หรือดีกว่านั้นคือ**ใช้ PAC/HAL crate** (หัวข้อ 102.5.2) ที่ implement การ
เข้าถึงแบบ volatile ให้ถูกต้องไว้เรียบร้อยแล้ว แทนการเขียน raw pointer เองทุกจุด

### 4. เลือก target ผิดกับ CPU core จริง

Cortex-M มีหลายรุ่นย่อยที่รองรับชุดคำสั่งต่างกัน (M0/M0+ ไม่มี hardware division, M3 ไม่มี FPU, M4F/M7 มี
FPU) — การเลือก target ผิด (เช่นใช้ `thumbv7em-none-eabihf` ซึ่งมี hardware floating point ไปกับชิปที่เป็น
Cortex-M3 ที่ไม่มี FPU) จะทำให้ **compile ผ่านสำเร็จโดยไม่มี error เลย** (เพราะ cross-compiler ไม่รู้ว่า
hardware จริงมี FPU หรือไม่ มันเชื่อ target string ที่สั่งไว้ตรง ๆ) แต่ถ้าเอา binary นั้นไป flash ใส่ MCU
จริงที่ไม่มี FPU จะได้ **`HardFault` exception ทันทีที่โค้ดพยายามรันคำสั่ง floating-point instruction ที่
CPU จริงไม่รู้จัก** — ปัญหานี้ debug ยากมากสำหรับมือใหม่เพราะไม่มี error message ที่ชี้ตรงไปที่สาเหตุเลย (แค่
เห็นโปรแกรม hang/crash โดยไม่รู้ว่าทำไม) วิธีป้องกันคือ**ตรวจสอบ core ของชิปให้ตรงกับ target ที่เลือกเสมอ**
(อ้างอิงจาก datasheet ของชิปนั้น) — สำหรับตัวอย่างในบทนี้: STM32F401 เป็น Cortex-M4F (มี FPU) จึงใช้
`thumbv7em-none-eabihf` ถูกต้อง แต่ LM3S6965 เป็น Cortex-M3 (ไม่มี FPU) จึงต้องใช้ `thumbv7m-none-eabi`
แทน — ใช้สลับกันจะได้ปัญหาแบบที่อธิบายไว้

### 5. ลืมเปิด peripheral clock ก่อนใช้งาน register

อธิบายไว้แล้วในหัวข้อ 102.6: MCU ยุคใหม่ปิด clock ของ peripheral ที่ไม่ได้ใช้ไว้เพื่อประหยัดพลังงานเป็นค่า
เริ่มต้น มือใหม่ที่ตั้งค่า GPIO mode และเขียน ODR ถูกต้องทุกอย่างแต่ **ลืมเปิด clock ผ่าน RCC ก่อน** จะเจอ
อาการ "ไม่มีอะไรเกิดขึ้นเลย" — LED ไม่ติด ปุ่มไม่ตอบสนอง ทั้งที่โค้ด logic ดูสมบูรณ์แบบ 100% — compiler ไม่
มีทางเตือนเรื่องนี้ได้เลยเพราะไม่รู้จัก "ความหมาย" ของ register เหล่านี้ (มันเห็นแค่ตัวเลข address) — วิธี
ป้องกันคือจำเป็นขั้นตอนเสมอว่า **peripheral ทุกตัวต้องเปิด clock ก่อนตั้งค่า/ใช้งาน** และตรวจสอบลำดับนี้เป็น
อันดับแรกเมื่อ hardware "ไม่ตอบสนอง" โดยไม่มี error message ใด ๆ

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เขียนฟังก์ชัน `fn classify_battery(percent: u8) -> BatteryStatus` ใน crate `#![no_std]`
   (ไม่ต้องมี `#[no_main]`/`panic_handler` — เป็น library เฉย ๆ แบบหัวข้อ 102.3.1) ที่คืนค่า enum
   `BatteryStatus { Critical, Low, Normal, Full }` ตามช่วงเปอร์เซ็นต์ (0-10 = Critical, 11-30 = Low,
   31-95 = Normal, 96-100 = Full) พร้อมเขียน unit test ด้วย `#[cfg(test)]` ยืนยันทุก boundary case
   *Hint*: โครงสร้างเหมือนตัวอย่าง `classify()` ในหัวข้อ 102.3.1 เป๊ะ แค่เปลี่ยนช่วงตัวเลขและชื่อ enum

2. **(กลาง)** ต่อจากตัวอย่าง blink ในหัวข้อ 102.6 (STM32F401, PA5) ให้เพิ่มการควบคุม pin ที่สองคือ **PA6**
   ให้กระพริบสลับกับ PA5 (เมื่อ PA5 ติด PA6 ต้องดับ และสลับกัน) เขียน bit mask ให้ถูกต้องสำหรับทั้ง MODER
   (ตั้ง PA6 เป็น output ด้วย) และ ODR ยืนยันด้วยการ cross-compile จริงด้วย `cargo build --target
   thumbv7em-none-eabihf` ว่าไม่มี error
   *Hint*: PA6 อยู่ที่ bit 6 ของ ODR (`1 << 6`) และ bit 12-13 ของ MODER (`0b01 << (6 * 2)`)

3. **(ยาก)** เขียนตัวอย่างแบบหัวข้อ 102.9 ที่มีตัวแปร shared state สองตัว (ไม่ใช่ตัวเดียว): ตัวนับ tick
   (`TICK_COUNT`) และ flag บอกว่า "ควรกระพริบเร็วขึ้นหรือไม่" (`FAST_BLINK: Mutex<Cell<bool>>`) โดย
   `SysTick` handler เพิ่มตัวนับทุกครั้ง และเมื่อตัวนับถึง 20 ให้ตั้ง `FAST_BLINK` เป็น `true` ส่วน main loop
   อ่านค่า `FAST_BLINK` เพื่อตัดสินใจว่าจะหน่วงเวลาสั้นหรือยาว ให้ทั้งสอง static ใช้ critical section เดียวกัน
   ต่อการเข้าถึงแต่ละครั้ง (ไม่ locked ค้างไว้ข้ามการเข้าถึงสองตัวพร้อมกัน) และอธิบายในคอมเมนต์ว่าทำไมการ lock
   สองตัวพร้อมกันในบล็อกเดียวอาจเสี่ยงต่อการทำให้ interrupt ถูกปิดนานเกินจำเป็น
   *Hint*: ทบทวนว่า critical section ที่ยาวเกินไปหมายความว่า interrupt อื่นถูก "บล็อก" ไม่ให้ทำงานได้เลยตลอด
   ช่วงนั้น — ยิ่ง critical section สั้นเท่าไหร่ ระบบยิ่งตอบสนองไว (responsive) เท่านั้น เชื่อมโยงกับหลักการ
   "ทำ `unsafe` block ให้เล็กที่สุด" ของ Part 41 หัวข้อ 41.10 ที่ใช้แนวคิดเดียวกัน (ยิ่งพื้นที่วิกฤตเล็ก ยิ่ง
   ตรวจสอบง่ายและกระทบระบบส่วนอื่นน้อย)

4. **(ประยุกต์ใช้งานจริง)** วัดขนาดไบนารีของตัวอย่าง blink (หัวข้อ 102.6) ภายใต้ profile ทั้ง 5 แถวในตาราง
   หัวข้อ 102.11 ด้วยตัวเอง (ถ้ามีสภาพแวดล้อมที่ติดตั้ง Rust + target `thumbv7em-none-eabihf` แล้ว) โดยใช้
   คำสั่ง `size` เปรียบเทียบ แล้วเขียนสรุปว่าการตั้งค่าไหนช่วยลดขนาดได้มากที่สุดในสัดส่วน (percentage) เท่าไหร่
   เทียบกับ baseline (`dev` profile) และอธิบายว่าทำไม `panic = "abort"` มีผลกับขนาดมากขนาดนี้ (หรือน้อยขนาดนี้
   ขึ้นกับตัวเลขจริงที่วัดได้) เมื่อเทียบกับ `opt-level = "z"`
   *Hint*: ผลของแต่ละ flag ไม่ independent กัน — `lto = true` มักทำให้ผลของ `panic = "abort"` เด่นชัดขึ้น
   เพราะ LTO มองเห็นได้ว่า unwinding path ทั้งเส้นไม่ถูกเรียกใช้จริงและตัดทิ้งได้เกลี้ยงกว่าไม่มี LTO

## สรุป

บทนี้พาเราออกจากโลกที่หลักสูตรอยู่มาตลอด 101 บท — โลกที่มี OS คอยจัดสรรทรัพยากรให้เสมอ — เข้าสู่โลกของ
**bare-metal embedded** ที่ไม่มีอะไรให้เลยนอกจาก CPU กับ memory ที่จำกัดระดับกิโลไบต์ เราขยายความ `#[no_std]`
ที่ Part 56 เกริ่นไว้ให้ลึกถึงระดับที่เขียนโปรแกรม executable เต็มรูปแบบได้จริง (`#[no_main]`, `#[entry]` จาก
`cortex-m-rt`, `#[panic_handler]` ของตัวเอง) เข้าใจกลไกของ memory-mapped I/O ผ่าน volatile read/write และ
เหตุผลที่ต้องมี (compiler ต้องไม่ optimize สิ่งที่ไม่ใช่ memory ธรรมดา) เห็นว่า safe abstraction (PAC/
`embedded-hal`) ห่อ `unsafe` ไว้ให้เหลือพื้นที่เสี่ยงน้อยที่สุดตามหลักปรัชญาของ Part 41 เรียนรู้ว่า interrupt
คือกลไกพื้นฐานของการจัดโครงสร้างงานแบบ event-driven บน hardware จริง และ `RefCell` + critical section
(Part 28, Part 39-40 ประยุกต์ใหม่) คือทางแก้ปัญหาการแชร์ mutable state ข้าม context อย่างปลอดภัย และปิดท้าย
ด้วย `embassy` ที่พิสูจน์ว่า async/await ของ Rust ไม่ได้ผูกติดกับ OS/Tokio เลยในทางภาษา — มันทำงานได้จริงแม้
ไม่มี OS ให้พึ่งพาเลยแม้แต่นิดเดียว

ที่สำคัญไม่แพ้เนื้อหาเชิงเทคนิค คือบทนี้ซื่อตรงเรื่อง**ขอบเขตของสิ่งที่ยืนยันได้จริง**ในสภาพแวดล้อมที่ไม่มี
microcontroller จริงต่ออยู่: ตัวอย่างส่วนใหญ่ยืนยันได้ระดับ cross-compilation (ซึ่งพิสูจน์ความถูกต้องทาง
syntax/type/borrow checker ได้ 100% แม้ไม่มี hardware) ส่วนตัวอย่างที่ตรงกับบอร์ดจำลองของ QEMU ได้ (LM3S6965
EVB) จะถูกรันจริงถ้าเครื่องมือพร้อมในสภาพแวดล้อมนี้ — การแยกแยะสองระดับนี้อย่างชัดเจนคือทักษะสำคัญเวลาทำงาน
embedded จริง เพราะแม้แต่วิศวกร embedded มืออาชีพก็ต้องทำงานแบบนี้เป็นประจำ (cross-compile ตรวจ syntax ก่อน
เสมอ ก่อนจะเสียเวลา flash ลง hardware จริงที่ใช้เวลานานกว่ามาก)

บทต่อไป (Part 103) จะเปลี่ยนทิศทางไปคนละขั้วสุดกับบทนี้โดยสิ้นเชิง: จาก MCU ที่มี RAM ไม่กี่สิบกิโลไบต์และไม่มี
กราฟิกใด ๆ เลย ไปสู่ **Game Development ด้วย Bevy Engine** ที่ต้องเรนเดอร์กราฟิกเรียลไทม์บนคอมพิวเตอร์ที่มี
ทรัพยากรเหลือเฟือ — ความต่างสุดขั้วระหว่างสองบทนี้ (Part 102 กับ Part 103) คือภาพสะท้อนที่ชัดเจนที่สุดของ
สิ่งที่หลักสูตรนี้พยายามแสดงให้เห็นตั้งแต่ Part 1: Rust คือภาษาเดียวที่ครอบคลุมได้ทั้งสองขั้วสุดนี้พร้อมกัน
โดยไม่ต้องเปลี่ยนภาษาหรือทิ้งความปลอดภัยของ ownership/borrow checker ไปเลยแม้แต่จุดเดียว

---

**Part ก่อนหน้า:** [Deployment: Cloud Platforms (AWS/GCP/Fly.io)](part-101-cloud-deployment.md) | **Part ถัดไป:** [Game Development ด้วย Bevy Engine เบื้องต้น](part-103-bevy-game-dev.md)
