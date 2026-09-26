# Part 42: Raw Pointers และ Memory Layout

> โมดูล: ระดับสูง (Advanced) | ระดับ: สูง | เวลาโดยประมาณ: 240 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายความแตกต่างที่แม่นยำระหว่าง raw pointer (`*const T`/`*mut T`) กับ reference (`&T`/`&mut T`)
  ได้ทั้งสี่มิติ: การันตีความ valid, กฎ aliasing, ความเป็นไปได้ที่จะเป็น null, และการติดตาม lifetime
- สร้าง raw pointer จาก reference ได้อย่างปลอดภัย และรู้ว่า**การสร้าง**กับ**การ dereference** เป็นการ
  กระทำที่ต่างกันโดยสิ้นเชิงในสายตาของ compiler — มีแค่การ dereference เท่านั้นที่ต้อง `unsafe`
- ใช้ pointer arithmetic (`.add()`, `.sub()`, `.offset()`) เดินทางผ่านหน่วยความจำแบบ raw ได้อย่างถูกต้อง
  และอธิบายได้ว่าทำไม `.add(1)` ถึงเลื่อนไป "หนึ่ง element" ไม่ใช่ "หนึ่งไบต์"
- แปลง raw pointer กลับไปเป็น reference ได้อย่างปลอดภัย พร้อมท่องและตรวจสอบ **safety invariant ทั้งสี่ข้อ**
  ที่ต้องเป็นจริงเสมอก่อนแปลงกลับ (non-null, aligned, initialized, ไม่มี alias ที่ขัดแย้งกัน)
- อธิบายความแตกต่างของ `#[repr(Rust)]`, `#[repr(C)]`, `#[repr(packed)]`, และ `#[repr(transparent)]`
  ได้ทั้งในเชิง layout จริงที่วัดได้ด้วย `size_of`/`align_of` และเชิง "ทำไมถึงมีสี่แบบนี้" ในบริบทใช้งานจริง
- คำนวณและอธิบายเรื่อง **alignment** และ **padding** ได้ พร้อมนำเทคนิคจัดเรียง field ใหม่ไปลดขนาด struct
  ในโปรแกรมจริงได้ทันที และรู้จัก `mem::transmute` ในระดับ "รู้ว่ามันมีอยู่และอันตรายแค่ไหน" มากกว่าจะใช้บ่อย ๆ

## ความรู้ที่ต้องมีมาก่อน

- **Part 41 (Unsafe Rust เบื้องต้น)** — นี่คือ**พื้นฐานตรง**ของบทนี้ เราจะใช้ทุกอย่างที่ Part 41 ปูไว้:
  พลังทั้ง 5 ข้อที่ `unsafe` block ปลดล็อกให้ (หนึ่งในนั้นคือ "dereference raw pointer" ซึ่งเป็นหัวใจของ
  บทนี้ทั้งบท), แนวคิด **safe abstraction** (การห่อ `unsafe` ไว้ข้างในแล้วเปิด API ที่ปลอดภัย 100% ให้ผู้ใช้
  ภายนอก), และตัวอย่าง `split_at_mut` ที่ Part 41 แสดงแบบเบา ๆ โดยตั้งใจ**เลื่อนความลึกเรื่อง raw pointer**
  มาไว้ที่บทนี้โดยเฉพาะ — ถ้าคุณยังไม่แม่นเรื่อง 5 พลังของ `unsafe` และ pattern safe abstraction ควรกลับไป
  ทวน Part 41 ก่อน เพราะบทนี้จะไม่อธิบายสองเรื่องนั้นซ้ำอีกรอบ
- **Part 7 (References และ Borrowing)** — กฎ aliasing ที่เข้มงวดของ Rust (`&mut T` ต้องไม่มี reference อื่น
  ชี้ไปที่เดียวกันพร้อมกันเด็ดขาด, `&T` มีได้หลายตัวพร้อมกันแต่ต้องไม่มี `&mut T` ปนอยู่) คือกฎเดียวกันที่บทนี้
  จะเทียบให้เห็นว่า **raw pointer ไม่ถูกบังคับด้วยกฎนี้เลย** — ต้องเข้าใจกฎเดิมให้แม่นก่อนถึงจะเห็นความต่างชัด
- **Part 18 (Generics ขั้นสูง)** — `MyVec<T>` ที่จะสร้างในหัวข้อ 42.12 เป็น generic type เต็มรูปแบบ ต้องใช้
  ความเข้าใจเรื่อง type parameter, `where` clause โดยนัย (ผ่าน trait bound ของ `Layout::array::<T>`) และ
  การเขียน `impl<T> ... for MyVec<T>`
- **Part 27–29 (Smart Pointers)** — `Box<T>` (Part 27) คือตัวอย่างของ "raw pointer ที่ห่อไว้ให้ปลอดภัยแล้ว"
  ที่ชัดเจนที่สุดใน std เอง บทนี้จะแสดงให้เห็นว่าเบื้องหลัง `Box<T>`, `Vec<T>`, และ `Rc<T>` (Part 28) ล้วน
  ใช้ raw pointer + manual memory management กันทั้งนั้น เราแค่ยังไม่เคยเห็นมันแบบเปิดเผยมาก่อน
- ตัวแปรพื้นฐาน `std::mem::size_of`/`align_of` ที่เกริ่นผ่าน ๆ ไปในบางบทก่อนหน้า — บทนี้จะอธิบายอย่างเป็น
  ระบบเต็มรูปแบบเป็นครั้งแรก

## เนื้อหา

### 42.1 ทวนความจำจาก Part 41: จุดที่เราค้างไว้

Part 41 เปิดประตูสู่โลก `unsafe` ด้วย 5 พลังที่ compiler ปลดล็อกให้เมื่ออยู่ใน `unsafe` block/fn:
dereference raw pointer, เรียก unsafe function/method, เข้าถึง/แก้ไข mutable static variable, implement
unsafe trait, และเข้าถึง field ของ union พร้อมกันนั้นได้เห็นตัวอย่าง `split_at_mut` แบบเบา ๆ ที่พา raw
pointer เข้ามาแค่พอให้เห็นว่า "มันมีอยู่และจำเป็นสำหรับสร้าง safe abstraction บางแบบที่ borrow checker
เพียงอย่างเดียวทำไม่ได้" แต่ตั้งใจ**ไม่ลงรายละเอียด**เรื่อง raw pointer เอง — เก็บไว้ให้บทนี้ทั้งบท

คำถามที่ Part 41 เปิดไว้แต่ยังไม่ตอบคือ: raw pointer คืออะไรกันแน่? มันต่างจาก reference ที่เราใช้มาตั้งแต่
Part 6–8 อย่างไรในระดับกลไก ไม่ใช่แค่ "แบบเดียวกันแต่ต้องเขียน `unsafe`" และเมื่อเราต้องเขียนโค้ดที่ยุ่งกับ
หน่วยความจำ raw ๆ แบบนี้ Rust จัดการเรื่อง "หน้าตา" ของข้อมูลในหน่วยความจำ (memory layout) อย่างไร — เพราะ
ถ้าจะเดินไปมาในหน่วยความจำด้วย pointer arithmetic เอง ก็ต้องรู้ให้แน่ชัดว่าแต่ละ field ของ struct วางอยู่ตรง
ไหน ห่างกันกี่ไบต์ นี่คือสองคำถามหลักที่บทนี้จะตอบให้ครบ

### 42.2 Raw Pointer คืออะไร: `*const T` และ `*mut T`

Rust มี pointer type สองแบบใหญ่ ๆ ที่ดูคล้ายกันบนหน้าจอ แต่มีความหมายต่างกันโดยสิ้นเชิงสำหรับ compiler:

- **Reference**: `&T` (shared) และ `&mut T` (mutable) — ที่เราใช้มาตลอดตั้งแต่ Part 6
- **Raw pointer**: `*const T` (ชี้แบบอ่านได้) และ `*mut T` (ชี้แบบเขียนได้) — พระเอกของบทนี้

Syntax คล้ายกันมาก (`&` เทียบกับ `*`) แต่สิ่งที่ compiler รับประกันให้ต่างกันแบบสิ้นเชิง reference ทุกตัว
ที่คุณสร้างได้ใน Rust ที่ compile ผ่าน **ต้องผ่านการตรวจของ borrow checker** ก่อนเสมอ (Part 7) — มันจึง
รับประกันได้ 100% ว่า reference นั้น**ไม่เป็น null, ชี้ไปยังข้อมูลที่ valid เสมอตลอดช่วง lifetime ของมัน,
และไม่มี alias ที่ขัดกับกฎ exclusive/shared borrow** ราคาที่ต้องแลกคือ compiler ต้องปฏิเสธโค้ดบางแบบที่
"จริง ๆ ก็ปลอดภัย" แต่พิสูจน์ด้วย static analysis ไม่ได้ (เช่น `split_at_mut` เวอร์ชัน safe ล้วนที่ Part 41
พยายามเขียนแล้วเจอ error)

Raw pointer คือ**ทางออกที่ Rust เปิดไว้ให้**สำหรับสถานการณ์แบบนั้น — มันคือ pointer แบบเดียวกับที่ C/C++ ใช้
กันมาตลอด: **แค่ที่อยู่ในหน่วยความจำ ไม่มีการรับประกันอะไรเพิ่มเติมจาก compiler เลย** ตารางเปรียบเทียบนี้สรุป
ความต่างทั้งสี่มิติที่สำคัญที่สุด:

| คุณสมบัติ | `&T` / `&mut T` (Reference) | `*const T` / `*mut T` (Raw Pointer) |
|---|---|---|
| **การันตีความ valid** | รับประกันเสมอ (ไม่ null, ชี้ข้อมูลที่มีอยู่จริง, ไม่ dangling) ตรวจโดย borrow checker ที่ compile time | **ไม่รับประกันอะไรเลย** — อาจ dangle, อาจชี้ไปยังหน่วยความจำที่ไม่ valid ก็ได้ ต้องรับผิดชอบเอง |
| **Null ได้หรือไม่** | เป็น null ไม่ได้เด็ดขาด (นี่คือเหตุผลที่ `Option<&T>` มีขนาดเท่ากับ `&T` เปล่า ๆ — null pointer optimization) | **เป็น null ได้** ผ่าน `ptr::null()`/`ptr::null_mut()` ต้องเช็คเองด้วย `.is_null()` |
| **กฎ Aliasing** | `&mut T` ต้องเป็น**หนึ่งเดียว**เท่านั้นในขณะที่มันยัง active (ห้ามมี `&T`/`&mut T` อื่นชี้ที่เดียวกันพร้อมกัน) | **อนุญาตให้ alias ได้เสมอ** — สร้าง `*mut T` หลายตัวชี้ตำแหน่งเดียวกันพร้อมกันได้อย่างไม่มีข้อจำกัดจาก type system |
| **การติดตาม Lifetime** | มี lifetime (`'a`) ที่ compiler ติดตามและตรวจสอบให้เต็มรูปแบบตลอดทั้งโปรแกรม | **ไม่มี lifetime ติดตามเลย** — pointer อยู่ได้นานเท่าไหร่ก็ได้ในสายตา compiler แม้ข้อมูลจริงจะถูกทำลายไปแล้ว |
| **ต้อง `unsafe` ตอนไหน** | ไม่ต้องเลย ใช้งานปกติทุกที่ | ต้อง `unsafe` เฉพาะตอน**dereference**เท่านั้น (หัวข้อถัดไป) |
| **Trait ที่ implement ให้อัตโนมัติ** | `Send`/`Sync` คำนวณตาม `T` (Part 40) | **ไม่ใช่ `Send` และไม่ใช่ `Sync`** ไม่ว่า `T` จะเป็นอะไร (เพราะ aliasing ที่ไม่มีการควบคุมทำให้ compiler ไม่กล้ารับประกันความปลอดภัยข้ามเธรดให้เลย) |

แถวสุดท้ายของตารางเชื่อมกลับไปที่ Part 40 ได้ตรง ๆ: raw pointer ทุกตัว (`*const T`/`*mut T`) **ไม่เป็น
`Send` และไม่เป็น `Sync` ไม่ว่า `T` จะเป็นชนิดอะไรก็ตาม** เพราะกฎ aliasing ที่ raw pointer เปิดกว้างให้นั้น
ทำให้ compiler ไม่สามารถพิสูจน์ความปลอดภัยข้ามเธรดแบบ structural ได้เลย (auto trait คำนวณจาก field แบบ
recursive ตามหัวข้อ 40.8 แต่ raw pointer เป็นจุด "ตัดวงจร" ของการคำนวณนั้นเสมอ) — นี่คือเหตุผลที่ type ที่
ห่อ raw pointer ไว้ภายใน เช่น `MyVec<T>` ที่เราจะเขียนในหัวข้อ 42.12 ต้อง**ประกาศ `Send`/`Sync` ด้วยตัวเอง
ผ่าน `unsafe impl`** (เทคนิคที่ Part 40 หัวข้อ 40.9 เกริ่นไว้) ถ้าต้องการให้ใช้ข้ามเธรดได้จริง

มาดูวิธีเขียน type ทั้งสองแบบให้ชัดเจนก่อนเข้าเนื้อหาลึกกว่านี้:

```rust
fn main() {
    let x: i32 = 42;

    // Reference ธรรมดา -- ผ่าน borrow checker เต็มรูปแบบ
    let ref_to_x: &i32 = &x;

    // Raw pointer -- แค่ "ที่อยู่" ไม่มีการรับประกันอะไรจาก type system
    let const_ptr: *const i32 = &x as *const i32;

    // *mut T ต้องมาจาก &mut T (หรือจาก allocator ตรง ๆ อย่างในหัวข้อ 42.12)
    let mut y: i32 = 10;
    let mut_ptr: *mut i32 = &mut y as *mut i32;

    println!("ref_to_x = {}", ref_to_x);
    println!("const_ptr (แสดงเป็น address) = {:p}", const_ptr);
    println!("mut_ptr (แสดงเป็น address)   = {:p}", mut_ptr);
}
```

โค้ดนี้ compile และรันผ่านทั้งหมด — สังเกตว่าเราสร้าง raw pointer ได้โดย**ไม่ต้องเขียน `unsafe` เลยแม้แต่คำ
เดียว** นี่คือประเด็นสำคัญที่หัวข้อถัดไปจะขยายความให้ชัดเจนขึ้นไปอีก

#### 42.2.1 เปรียบเทียบกับภาษาอื่น: ทำไมภาษาระดับสูงถึงไม่มี Raw Pointer แบบนี้

ก่อนไปต่อ ลองถอยออกมามองภาพกว้างสักครู่ว่าภาษาอื่น ๆ ที่คุณอาจคุ้นเคยจัดการกับแนวคิด "pointer" อย่างไร จะ
ช่วยให้เห็นว่าสิ่งที่ Rust ทำอยู่นั้นอยู่ตรงไหนของสเปกตรัม:

- **C/C++**: มี pointer แบบเดียวกับ raw pointer ของ Rust เป๊ะ (`int*`, `int* const`) และเป็น pointer type
  **เดียว**ที่มีให้ใช้เลย — ไม่มี concept ของ "reference ที่ borrow checker ตรวจสอบ" แยกออกมาต่างหาก (C++
  มี `int&` ก็จริง แต่ compiler ไม่ตรวจ lifetime/aliasing ให้เหมือน Rust) ผลคือโปรแกรมเมอร์ C/C++ ต้องจำ
  กฎความปลอดภัยทั้งหมดไว้ในหัวเอง ไม่มี compiler ช่วยตรวจ dangling pointer, aliasing violation, หรือ
  use-after-free ให้เลยแม้แต่กรณีพื้นฐานที่สุด — นี่คือที่มาของสถิติที่โด่งดังว่า memory safety bug
  (buffer overflow, use-after-free, double-free) เป็นสาเหตุของช่องโหว่ความปลอดภัยส่วนใหญ่ในซอฟต์แวร์ที่เขียน
  ด้วย C/C++ มาหลายทศวรรษ Rust แก้ปัญหานี้ด้วยการ**แยก pointer เป็นสองระดับ**: reference (ปลอดภัยแต่จำกัด)
  สำหรับโค้ด 95%+ ของโปรแกรมทั่วไป และ raw pointer (เหมือน C แต่ต้องประกาศ `unsafe` ชัดเจน) สำหรับ 5% ที่
  เหลือที่จำเป็นจริง ๆ
- **Python/JavaScript**: **ไม่มี concept ของ pointer ที่ผู้ใช้เข้าถึงได้เลย** ทุกตัวแปรที่ไม่ใช่ primitive
  คือ reference ไปยัง object บน heap โดยอัตโนมัติ (เรียกกันว่า "everything is a reference" ใน object
  model ของทั้งสองภาษา) แต่ผู้ใช้**ไม่มีทาง**เอา "ที่อยู่" นั้นออกมาทำ arithmetic เองได้ — ภาษาซ่อนกลไก
  หน่วยความจำทั้งหมดไว้หลัง garbage collector และไม่เปิดช่องให้ยุ่งกับ raw memory เด็ดขาด นี่คือเหตุผลที่
  Python/JavaScript เขียนง่ายกว่ามากในงานทั่วไป แต่**ไม่สามารถ**เอาไปเขียน operating system kernel, device
  driver, หรือ high-performance data structure ระดับต่ำได้เลย เพราะไม่มีทางควบคุม memory layout ได้ตรง ๆ
- **Java/Go**: มี reference (object reference ใน Java, pointer ใน Go ที่เขียนด้วย `*T`) แต่ทั้งสองภาษาไม่
  อนุญาตให้ทำ pointer arithmetic เด็ดขาด (Go ห้ามชัดเจน — pointer ใน Go ใช้ได้แค่ dereference กับส่งผ่าน
  ฟังก์ชัน ทำ `.add()`/`.offset()` ไม่ได้เลยนอกจาก import `unsafe` package ที่ Go เตือนไว้ตรง ๆ ว่า "ใช้แล้ว
  ไม่รับประกันความเข้ากันได้ข้าม Go version") ทั้งคู่พึ่งพา garbage collector จัดการหน่วยความจำให้ทั้งหมด
  ไม่เปิดช่องให้ manual allocation ตรง ๆ แบบที่ Rust ทำได้ผ่าน `std::alloc`

จุดที่ Rust แตกต่างจากทุกภาษาข้างบนคือ: **มันให้ทั้งสองโลกพร้อมกัน** — เขียนโค้ด 95% ด้วย reference ที่
ปลอดภัยเทียบเท่า (หรือดีกว่า) Java/Go/Python ในเรื่อง memory safety โดยไม่มี runtime cost ของ garbage
collector เลย และเมื่อจำเป็นต้องลงไปควบคุมหน่วยความจำแบบละเอียดจริง ๆ (เขียน library ระดับ system, custom
allocator, หรือ data structure ประสิทธิภาพสูงอย่าง `MyVec<T>` ในหัวข้อ 42.12) ก็เปิดทางให้ทำได้เต็มรูปแบบ
เทียบเท่า C — เพียงแค่ต้องประกาศ `unsafe` ให้ชัดเจนว่ากำลังออกจากพื้นที่ที่ compiler รับประกันความปลอดภัย
ให้แล้ว

### 42.3 การสร้าง Raw Pointer เป็นเรื่อง Safe — มีแค่การ Dereference ที่ต้อง `unsafe`

นี่คือ**นัยที่มักถูกเข้าใจผิด**บ่อยที่สุดสำหรับคนที่เพิ่งเจอ raw pointer เป็นครั้งแรก: หลายคนคิดว่า "raw
pointer = ของอันตราย ต้องเขียน `unsafe` ตลอดเวลาที่ยุ่งกับมัน" แต่ความจริงแล้ว Rust แบ่งเส้นไว้ชัดเจนกว่านั้น
มาก — **การสร้าง (create) raw pointer เป็นการกระทำที่ปลอดภัยเสมอ ไม่ว่าจะสร้างจากอะไรก็ตาม** เหตุผลเชิง
ออกแบบคือ: การมี "ค่า" ที่เป็นแค่ตัวเลขแทนที่อยู่ในหน่วยความจำอยู่เฉย ๆ **ไม่สามารถทำให้เกิด undefined
behavior ได้เลย** — ต่อให้มันเป็น null, dangling, หรือชี้ไปยังหน่วยความจำที่ไม่มีอยู่จริงก็ตาม ตราบใดที่ยังไม่
มีใคร**อ่านหรือเขียนผ่านมัน** ค่านั้นก็เป็นแค่ข้อมูลธรรมดาชิ้นหนึ่งไม่ต่างจาก `u64` เปล่า ๆ

สิ่งที่เป็นอันตรายจริง ๆ คือ**การ dereference** — การใช้ `*ptr` เพื่ออ่านหรือเขียนค่าที่ pointer ชี้ไปที่
ตรงนั้นแหละที่ compiler ไม่มีทางรับประกันความถูกต้องให้ได้อีกต่อไป (ไม่มี borrow checker คอยตรวจ) และตรงนั้น
คือจุดเดียวที่ Rust บังคับให้เขียน `unsafe` block

มาดูตัวอย่างที่แสดงเส้นแบ่งนี้ให้เห็นชัด ๆ — สร้าง raw pointer หลายตัวได้อย่างอิสระโดยไม่ต้อง `unsafe` เลย:

```rust
fn main() {
    let a = 1;
    let b = 2;
    let mut c = 3;

    // สร้าง raw pointer จาก reference หลายรูปแบบ -- ทุกบรรทัดนี้ "ปลอดภัย" ในสายตา compiler
    let p1: *const i32 = &a;              // จาก &T ตรง ๆ (coercion อัตโนมัติ)
    let p2: *const i32 = &b as *const i32; // จาก &T ผ่าน `as` แบบชัดเจน
    let p3: *mut i32 = &mut c as *mut i32; // จาก &mut T
    let p4: *mut i32 = &mut c as *mut i32; // สร้างตัวที่สองชี้ไปที่ c เดิม -- ทำได้! (raw pointer alias ได้)

    // สร้าง raw pointer ที่ "ไม่มีข้อมูลจริงรองรับ" ก็ยังปลอดภัย ตราบใดที่ไม่ dereference
    let dangling: *const i32 = std::ptr::null();
    let random_number = 0x1234_5678_usize;
    let from_number = random_number as *const i32; // แปลงจากตัวเลขล้วน ๆ ก็ยังปลอดภัย!

    println!("p1={:p} p2={:p} p3={:p} p4={:p}", p1, p2, p3, p4);
    println!("dangling.is_null() = {}", dangling.is_null());
    println!("from_number (แค่ค่าที่อยู่ ไม่ได้ dereference) = {:p}", from_number);
    // ไม่มี unsafe block เลยแม้แต่บรรทัดเดียวในฟังก์ชันนี้ -- และมัน compile ผ่านจริง!
}
```

โค้ดนี้ compile และรันผ่านโดยไม่มี `unsafe` เลยแม้แต่คำเดียว จุดที่ควรสังเกตเป็นพิเศษคือบรรทัด `p3`/`p4`:
เราสร้าง `*mut i32` **สองตัว**ที่ชี้ไปยัง `c` ตัวเดียวกันพร้อมกัน — ถ้าเป็น `&mut i32` สองตัวแบบนี้ compiler
จะปฏิเสธทันทีด้วย error E0499 ("cannot borrow `c` as mutable more than once at a time") ตามกฎจาก Part 7
แต่กับ raw pointer ไม่มีกฎ aliasing แบบนั้นบังคับเลย — สร้างได้อิสระ **นี่คือ "ราคา" ของอำนาจที่มากขึ้น**:
คุณได้ความยืดหยุ่นที่ borrow checker ไม่อนุญาต แต่ก็ต้องแบกรับผิดชอบ**เองทั้งหมด**ว่าการใช้ pointer ที่ alias
กันแบบนี้จะไม่ทำให้เกิดปัญหาจริงตอน dereference (ดูหัวข้อกับดักที่ 3 ท้ายบท)

ตอนนี้มาดูสิ่งที่เกิดขึ้นทันทีที่เราลองเพิ่มการ dereference เข้าไปโดยไม่มี `unsafe` block ครอบ:

```rust
fn main() {
    let x = 5;
    let p: *const i32 = &x as *const i32;
    println!("{}", *p); // ลอง dereference ตรง ๆ โดยไม่มี unsafe
}
```

Error จริง (compile ด้วย `rustc --edition 2021`):

```
error[E0133]: dereference of raw pointer is unsafe and requires unsafe function or block
 --> src/main.rs:4:20
  |
4 |     println!("{}", *p); // ลอง dereference ตรง ๆ โดยไม่มี unsafe
  |                    ^^ dereference of raw pointer
  |
  = note: raw pointers may be null, dangling or unaligned; they can violate aliasing rules and cause data races: all of these are undefined behavior
```

ข้อความ `= note` ท้าย error สรุปเหตุผลทั้งหมดที่เราคุยกันในหัวข้อ 42.2 ไว้ในบรรทัดเดียว: null, dangling,
unaligned, aliasing/data race — ทั้งสี่อย่างนี้คือสิ่งที่ borrow checker เคยรับประกันให้กับ reference แต่ raw
pointer ไม่มีการรับประกันเลย ดังนั้น**การอ่านหรือเขียนผ่านมัน** จึงต้องประกาศด้วย `unsafe` ว่า "ฉันตรวจสอบ
เองแล้วว่าทั้งสี่ข้อนี้เป็นจริง ไม่ใช่ปัญหา ในกรณีนี้"

### 42.4 Dereferencing: การอ่านและเขียนผ่าน Raw Pointer

เมื่อรู้แล้วว่าต้องใส่ `unsafe` ตอนไหน มาดูวิธีอ่าน (`*ptr`) และเขียน (`*ptr = value`) แบบเต็มรูปแบบ:

```rust
fn main() {
    let mut score = 75;

    let read_ptr: *const i32 = &score as *const i32;
    let write_ptr: *mut i32 = &mut score as *mut i32;

    // อ่านค่าผ่าน raw pointer -- ต้องอยู่ใน unsafe block
    let current = unsafe { *read_ptr };
    println!("อ่านค่าปัจจุบัน: {}", current);

    // เขียนค่าใหม่ผ่าน raw pointer -- ก็ต้อง unsafe เช่นกัน
    unsafe {
        *write_ptr = 100;
    }
    println!("score หลังเขียนผ่าน raw pointer: {}", score);

    // ทำหลายการกระทำใน unsafe block เดียวได้ตามปกติ
    unsafe {
        *write_ptr += 5;
        *write_ptr *= 2;
    }
    println!("score หลังคำนวณเพิ่มเติม: {}", score);
}
```

ทดสอบรันแล้วได้ผลลัพธ์จริงตามลำดับ:

```
อ่านค่าปัจจุบัน: 75
score หลังเขียนผ่าน raw pointer: 100
score หลังคำนวณเพิ่มเติม: 210
```

สังเกตว่าทั้ง `read_ptr` (อ่าน `*const i32` ที่จริง ๆ อยู่ในที่เดียวกันกับ `score`) และ `write_ptr` ต่างชี้ไป
ยังตัวแปรตัวเดียวกัน — เมื่อเราเขียนผ่าน `write_ptr` แล้วอ่านค่า `score` ตรง ๆ (ไม่ผ่าน pointer) ก็เห็นค่าที่
เปลี่ยนไปด้วยเหมือนกัน เพราะมันคือหน่วยความจำก้อนเดียวกันเป๊ะ ไม่มีการ copy ข้อมูลใด ๆ เกิดขึ้นระหว่างทาง —
raw pointer ก็เป็นแค่ "ที่อยู่" เหมือนเดิม การ dereference คือการบอกให้ CPU ไปอ่าน/เขียนที่ address นั้น
ตรง ๆ

ข้อสังเกตสำคัญอีกจุด: `unsafe` block **ไม่ได้แปลว่าโค้ดข้างในจะปิดการตรวจ type ทั้งหมด** — compiler ยังตรวจ
type ให้ครบถ้วนเหมือนเดิมทุกอย่าง (`*write_ptr = 100` ต้องเป็น `i32` เท่านั้น ใส่ `String` ไม่ได้) `unsafe`
ปิดแค่ **5 การตรวจเฉพาะที่ Part 41 ระบุไว้เท่านั้น** — การ dereference raw pointer เป็นหนึ่งในนั้น ไม่ใช่
"ประตูปิดการตรวจทั้งหมด" อย่างที่บางคนเข้าใจผิด

### 42.5 Pointer Arithmetic: เดินทางในหน่วยความจำด้วย `.add()`, `.sub()`, `.offset()`

จุดแข็งของ raw pointer ที่ reference ทำไม่ได้เลยคือ**การเลื่อนตำแหน่ง**ไปยัง element อื่นในหน่วยความจำต่อ
เนื่องกัน (เช่น array หรือ slice) โดยไม่ต้องผ่าน indexing operator `[]` ปกติ Rust มี method หลักสามตัวสำหรับ
ทำสิ่งนี้บน raw pointer:

- **`.add(count)`** — เลื่อนไปข้างหน้า `count` "ตัว" (`count: usize`)
- **`.sub(count)`** — เลื่อนถอยหลัง `count` "ตัว" (`count: usize`)
- **`.offset(count)`** — เลื่อนไปทางใดก็ได้ตาม `count: isize` (บวกคือไปหน้า ลบคือถอยหลัง — รวมสองตัวข้าง
  บนเข้าเป็นตัวเดียว)

ทั้งสามตัวเป็น **unsafe method** ทั้งหมด (ต้องเรียกใน `unsafe` block) และประเด็นที่สำคัญที่สุดที่ต้องเข้าใจ
ให้แม่นคือ: **หน่วยที่เลื่อนคือ "จำนวน element ของ `T`" ไม่ใช่ "จำนวนไบต์"** — pointer มันรู้ว่าตัวมันชี้ไปที่
type อะไร (จาก generic parameter `T` ของมันเอง) จึงคำนวณให้เองว่าต้องเลื่อนกี่ไบต์จริง ๆ โดยเอา
`count * size_of::<T>()` เข้าไปคำนวณให้อัตโนมัติ — เราไม่ต้องคำนวณไบต์เองเลย

ตัวอย่างที่แสดงให้เห็นทั้งเรื่องนี้และเปรียบเทียบกับวิธี safe จาก Part 13/25:

```rust
fn main() {
    let scores: [i32; 5] = [10, 20, 30, 40, 50];
    let ptr: *const i32 = scores.as_ptr(); // as_ptr() ให้ raw pointer ไปยัง element แรก (safe เพราะแค่สร้าง)

    println!("size_of::<i32>() = {}", std::mem::size_of::<i32>());

    // วนอ่านค่าด้วย pointer arithmetic ผ่าน .add(offset)
    for i in 0..scores.len() {
        unsafe {
            let element_ptr = ptr.add(i); // เลื่อนไป i "ตัว" ไม่ใช่ i "byte"
            println!("scores[{}] ผ่าน raw pointer = {}", i, *element_ptr);
        }
    }

    // .offset() รับ isize เลื่อนได้ทั้งบวกและลบในเมธอดเดียว
    unsafe {
        let last = ptr.offset(4);
        println!("offset(4) = {}", *last);
        let back_to_first = last.offset(-4);
        println!("offset(-4) จากตัวสุดท้าย = {}", *back_to_first);
    }

    // .sub() สำหรับเลื่อนถอยหลังล้วน ๆ
    let end_ptr = unsafe { ptr.add(scores.len() - 1) };
    unsafe {
        let prev = end_ptr.sub(1);
        println!("sub(1) จากตัวสุดท้าย = {}", *prev);
    }

    // เปรียบเทียบผลลัพธ์กับวิธี safe: iterator (Part 25) และ indexing ปกติ (Part 13)
    let sum_safe: i32 = scores.iter().sum();
    let sum_unsafe: i32 = {
        let mut total = 0;
        for i in 0..scores.len() {
            total += unsafe { *ptr.add(i) };
        }
        total
    };
    println!("sum_safe = {}, sum_unsafe = {}", sum_safe, sum_unsafe);
    assert_eq!(sum_safe, sum_unsafe);
}
```

ผลลัพธ์จริงจากการรัน:

```
size_of::<i32>() = 4
scores[0] ผ่าน raw pointer = 10
scores[1] ผ่าน raw pointer = 20
scores[2] ผ่าน raw pointer = 30
scores[3] ผ่าน raw pointer = 40
scores[4] ผ่าน raw pointer = 50
offset(4) = 50
offset(-4) จากตัวสุดท้าย = 10
sub(1) จากตัวสุดท้าย = 40
sum_safe = 150, sum_unsafe = 150
```

สังเกตว่า `size_of::<i32>()` คือ 4 ไบต์ แต่เราเรียก `.add(1)` ไม่ใช่ `.add(4)` เพื่อเลื่อนไปตัวถัดไป — นี่คือ
สิ่งที่ยืนยันข้อความสำคัญของหัวข้อนี้: **pointer คำนวณไบต์ให้เราอัตโนมัติจาก `size_of::<T>()`** เบื้องหลังแล้ว
ถ้าเปลี่ยน `T` เป็น type ที่ใหญ่กว่า เช่น `i64` (8 ไบต์) โค้ด `.add(i)` ตัวเดิมก็ยังใช้ตรรกะ "เลื่อนไป i ตัว"
เหมือนกันทุกตัวอักษร ไม่ต้องแก้เลขคูณ 8 เองที่ไหนเลย — นี่คือความสะดวกที่ raw pointer ให้เมื่อเทียบกับการ
คำนวณ offset ไบต์เองแบบดิบ ๆ ในภาษาที่ต่ำกว่า Rust

**ข้อควรระวังสำคัญ**: ทั้ง `.add()`, `.sub()`, และ `.offset()` มี safety invariant ของตัวเองว่า**ผลลัพธ์
ต้องยังอยู่ในขอบเขตของ allocation เดียวกัน** (หรือเลยไปได้อีกแค่ 1 ตัวสำหรับ "one-past-the-end" pointer ที่
ใช้เทียบเงื่อนไข loop เท่านั้น ห้าม dereference ตำแหน่งนั้น) การเรียก `.add()` เกินขอบเขตของ array/allocation
เดิม (เช่น `.add(1000)` บน array ที่มี 5 ตัว) ถือเป็น undefined behavior ทันที **แม้จะยังไม่ได้ dereference
ก็ตาม** — ต่างจากการสร้าง raw pointer จาก address ธรรมดาในหัวข้อ 42.3 ที่ปลอดภัยเสมอ นี่คือข้อยกเว้นที่ควร
จำไว้: การคำนวณ pointer arithmetic ที่เกินขอบเขต allocation เป็น UB ได้เองโดยไม่ต้อง dereference เนื่องจาก
compiler ใช้ข้อมูลนี้ไปกับการ optimize code ในภาษา LLVM IR ระดับล่างมาก

#### 42.5.1 เทียบ Pointer กันตรง ๆ: `ptr::eq` และตัวดำเนินการ `==`

อีกการกระทำหนึ่งที่ raw pointer ทำได้แต่ reference ทำไม่สะดวกเท่าคือการ**เทียบว่า pointer สองตัวชี้ไปที่
"ตำแหน่ง" เดียวกันในหน่วยความจำหรือไม่** (ไม่ใช่เทียบว่า "ค่า" ที่ชี้ไปเท่ากันหรือไม่ — สองเรื่องนี้ต่างกัน
โดยสิ้นเชิง) Rust มี `std::ptr::eq()` และ operator `==` บน raw pointer ที่ทำสิ่งนี้ให้ตรง ๆ:

```rust
fn main() {
    let a = 5;
    let b = 5;

    let pa: *const i32 = &a;
    let pb: *const i32 = &b;
    let pa2: *const i32 = &a;

    println!("pa == pa2 (ptr::eq): {}", std::ptr::eq(pa, pa2));
    println!("pa == pb  (ptr::eq): {}", std::ptr::eq(pa, pb));
    // เทียบกับ == ธรรมดาบน raw pointer -- ก็เทียบ address เหมือนกัน (PartialEq ของ raw pointer คือเทียบ address)
    println!("pa == pa2 (==): {}", pa == pa2);
    println!("pa == pb  (==): {}", pa == pb);

    // เทียบ address โดยไม่สนใจ "ค่า" ที่ pointer ชี้ไป -- a และ b มีค่าเท่ากัน (5 == 5) แต่คนละตำแหน่ง
    assert_eq!(a, b);
    assert!(!std::ptr::eq(pa, pb));
}
```

ผลลัพธ์จริง:

```
pa == pa2 (ptr::eq): true
pa == pb  (ptr::eq): false
pa == pa2 (==): true
pa == pb  (==): false
```

ข้อสังเกตสำคัญ: `a` และ `b` มี**ค่า**เท่ากัน (ทั้งคู่เป็น `5`) แต่ `pa`/`pb` เทียบกันได้ `false` เพราะทั้งสอง
ชี้ไปยัง**ตำแหน่ง**คนละที่กันในหน่วยความจำ (`a` กับ `b` เป็นตัวแปรคนละตัว แม้จะมีค่าตรงกันก็ตาม) — นี่คือ
ความแตกต่างระหว่าง **identity** (คือตัวเดียวกันไหม ตรวจด้วย `ptr::eq`) กับ **equality** (คือค่าเท่ากันไหม
ตรวจด้วย `==` บน `T` เอง ผ่าน trait `PartialEq` ที่ Part 19 สอนไว้) การเทียบ identity แบบนี้มีประโยชน์จริง
ในสถานการณ์เช่น: ตรวจว่า reference สองตัวที่ผ่านเข้ามาในฟังก์ชันชี้ไปยัง object เดียวกันหรือไม่ (สำคัญมาก
ตอนเขียน algorithm ที่ต้องป้องกันการประมวลผลซ้ำ หรือตรวจ self-reference ในโครงสร้างข้อมูลแบบ graph) — และ
มันคือกลไกเดียวกันเป๊ะที่ `Rc::ptr_eq()` (Part 28) ใช้ตรวจว่า `Rc<T>` สองตัวชี้ไปยัง allocation เดียวกัน
หรือไม่ ไม่ใช่แค่มีค่าเท่ากัน

### 42.6 Null Pointer: `ptr::null()`, `ptr::null_mut()`, และ `.is_null()`

จากตารางเปรียบเทียบในหัวข้อ 42.2 — raw pointer เป็น**null ได้**ในขณะที่ reference เป็นไม่ได้เด็ดขาด
Rust มีฟังก์ชันสร้าง null pointer ให้ตรง ๆ ใน `std::ptr`:

```rust
use std::ptr;

fn safe_read(p: *const i32) -> Option<i32> {
    if p.is_null() {
        None
    } else {
        // SAFETY: ตรวจแล้วว่าไม่ null ก่อนหน้านี้ และในตัวอย่างนี้เราควบคุมที่มาของ pointer เอง
        // ว่ามันชี้ไปยัง i32 ที่ valid จริง (ผ่านมาจาก &x โดยตรงในฝั่ง caller)
        Some(unsafe { *p })
    }
}

fn main() {
    let n: *const i32 = ptr::null();
    let nm: *mut i32 = ptr::null_mut();
    println!("n.is_null() = {}", n.is_null());
    println!("nm.is_null() = {}", nm.is_null());

    match safe_read(n) {
        Some(v) => println!("อ่านได้ค่า {}", v),
        None => println!("pointer เป็น null ไม่อ่าน"),
    }

    let x = 42;
    let p: *const i32 = &x;
    match safe_read(p) {
        Some(v) => println!("อ่านได้ค่า {}", v),
        None => println!("pointer เป็น null ไม่อ่าน"),
    }
}
```

ผลลัพธ์จริง:

```
n.is_null() = true
nm.is_null() = true
pointer เป็น null ไม่อ่าน
อ่านได้ค่า 42
```

ฟังก์ชัน `safe_read` ในตัวอย่างนี้แสดง**รูปแบบมาตรฐาน**ที่ควรใช้ทุกครั้งก่อน dereference raw pointer ที่มี
โอกาสเป็น null: **เช็ค `.is_null()` ก่อนเสมอ** แล้วค่อยเข้า `unsafe` block เฉพาะกรณีที่ไม่ null เท่านั้น
สังเกตด้วยว่า comment `// SAFETY: ...` ที่แนบมากับทุก `unsafe` block ในบทนี้ไม่ใช่แค่ธรรมเนียมสวย ๆ — มันคือ
เอกสารที่บันทึกไว้ว่า "ทำไมโค้ดตรงนี้ถึงปลอดภัยจริง" สำหรับคนอ่านโค้ดในอนาคต (รวมถึงตัวเราเองในอีกหกเดือน
ข้างหน้า) นี่คือ convention ที่ทีม Rust เองก็ใช้ทั่วทั้ง compiler และ std source code

ที่สำคัญคือ **การเช็ค `.is_null()` เพียงอย่างเดียวไม่ได้พิสูจน์ว่า pointer นั้น "ปลอดภัย" ที่จะ
dereference 100%** — มันแค่ตัดกรณีที่แย่ที่สุดกรณีหนึ่งออกไป (null pointer dereference ซึ่งในภาษาอื่นมักทำ
ให้โปรแกรม crash แบบ segfault) แต่ pointer ที่ไม่ null ก็ยัง**dangling ได้** (ชี้ไปยังหน่วยความจำที่ถูกคืน
กลับไปแล้ว) หรือ**ไม่ align** ได้เหมือนกัน — `.is_null()` เป็นแค่**หนึ่งใน**การตรวจที่ต้องทำ ไม่ใช่การตรวจ
ทั้งหมด เราจะเห็นรายการตรวจแบบครบทั้งชุดในหัวข้อถัดไป

### 42.7 แปลงกลับไปเป็น Reference: `&*ptr` และ Safety Invariant ทั้งสี่ข้อ

เราเห็นทางไป **reference → raw pointer** มาแล้วในหัวข้อ 42.2–42.3 (`as *const T`/`as *mut T` — ปลอดภัย
เสมอ) ทีนี้มาดูทางกลับ **raw pointer → reference** ซึ่งเขียนด้วย `&*ptr` (สำหรับ `&T`) หรือ `&mut *ptr`
(สำหรับ `&mut T`) — สังเกตว่าตอนนี้ความ unsafe ย้ายไปอยู่ที่ฝั่ง**dereference** (`*ptr` ส่วนที่อยู่ข้างใน)
ไม่ใช่ที่ตัวดำเนินการ `&`/`&mut` ข้างนอก ทั้งนี้เพราะการหยิบ reference ออกมาจาก raw pointer ก็คือการยืนยัน
ว่า "ข้อมูลที่ pointer นี้ชี้ไปมีอยู่จริงและถูกต้องครบถ้วนพร้อมให้ borrow checker กลับมาดูแลต่อ" ซึ่งเป็น
การยืนยันที่ compiler ตรวจให้ไม่ได้ ต้องประกาศด้วย `unsafe` เหมือนการ dereference ทั่วไป

```rust
fn main() {
    let mut value = 100;

    // ทางไป: reference -> raw pointer (SAFE เสมอ ไม่ต้อง unsafe)
    let raw: *mut i32 = &mut value as *mut i32;

    // ทางกลับ: raw pointer -> reference (ต้อง unsafe เพราะต้องรับประกัน invariant ด้วยตัวเอง)
    // SAFETY:
    // 1. raw ไม่ใช่ null (มาจาก &mut value โดยตรง เห็นชัดจาก scope นี้)
    // 2. raw ถูก align อย่างถูกต้องสำหรับ i32 (มาจาก &mut value โดยตรง ไม่ได้มาจาก packed struct)
    // 3. raw ชี้ไปยัง i32 ที่ initialized แล้วจริง (คือ value ที่ประกาศไว้ข้างบน)
    // 4. ไม่มี reference หรือ raw pointer อื่นที่ยัง alive ชี้ไปที่ value ในขณะนี้ (raw ตัวนี้ตัวเดียว)
    let back: &mut i32 = unsafe { &mut *raw };
    *back += 1;
    println!("value = {}", value);

    let shared_raw: *const i32 = &value as *const i32;
    // SAFETY: เงื่อนไขเดียวกับข้างบน แต่เป็นฝั่ง shared reference (ไม่ต้องกังวลเรื่อง exclusivity)
    let back_shared: &i32 = unsafe { &*shared_raw };
    println!("back_shared = {}", back_shared);
}
```

ผลลัพธ์จริง:

```
value = 101
back_shared = 101
```

โค้ดนี้ compile และรันได้อย่างถูกต้อง แต่สิ่งที่สำคัญที่สุดของหัวข้อนี้ไม่ใช่โค้ดที่ compile ผ่าน — คือ**เหตุผล
ที่ทำให้มันปลอดภัยจริง** ซึ่งสรุปเป็น **safety invariant สี่ข้อที่ต้องเป็นจริงทั้งหมดพร้อมกัน** ทุกครั้งที่
แปลง raw pointer กลับไปเป็น reference (ไม่ว่าจะเป็น `&*ptr` หรือ `&mut *ptr`):

1. **Non-null**: pointer ต้องไม่เป็น null — reference เป็น null ไม่ได้เด็ดขาดตามที่เห็นในหัวข้อ 42.2
2. **Properly aligned**: pointer ต้องชี้ไปยัง address ที่ align ถูกต้องสำหรับ `T` (ดูหัวข้อ 42.9) —
   reference ที่ไม่ align จะเป็น undefined behavior แม้จะไม่เคย dereference เลย (ดูกับดักที่ 2 ท้ายบท)
3. **Points to a valid, initialized `T`**: หน่วยความจำที่ pointer ชี้ไปต้องมีค่าของ type `T` ที่
   ก่อสร้างสมบูรณ์แล้วอยู่จริง (ไม่ใช่หน่วยความจำที่ยังไม่ initialize หรือ garbage เก่าที่เหลือจากการใช้งาน
   ครั้งก่อน)
4. **ไม่มี alias ที่ขัดแย้งกัน**: สำหรับ `&mut *ptr` โดยเฉพาะ — ในขณะที่ reference ที่ได้ยัง "มีชีวิต" อยู่
   (ยัง alive ตาม lifetime ของมัน) **ต้องไม่มี reference หรือ raw pointer ตัวอื่นเข้าถึงข้อมูลชิ้นเดียวกันนี้
   เลย** (ไม่ว่าจะอ่านหรือเขียน) — สำหรับ `&*ptr` (shared) เงื่อนไขผ่อนลงมาเป็น: ต้องไม่มี `&mut`/`*mut`
   ตัวอื่นเขียนทับข้อมูลนี้ในขณะที่ shared reference ยัง alive

ข้อที่ 4 คือข้อที่**ละเมิดง่ายที่สุดและอันตรายที่สุด**เพราะมันไม่ทำให้ compiler ฟ้อง error ใด ๆ เลย (ดูกับดัก
ที่ 3 ท้ายบท) — compiler จาวาไปเชื่อ**เต็มที่**ว่า `&mut T` ที่คุณส่งต่อไปให้ฟังก์ชันอื่นนั้นไม่มี alias
ใด ๆ เหลืออยู่ (กฎ "no-alias" นี้คือสิ่งที่ทำให้ LLVM optimizer ของ Rust ทำงานได้ดีกว่า C ในหลายกรณี เพราะมัน
สามารถสมมติกฎนี้ไว้แล้ว optimize ได้อย่างมั่นใจ) — ถ้าคุณละเมิดข้อ 4 โดยไม่ตั้งใจ ผลลัพธ์ที่ได้อาจถูกต้องบน
เครื่องคุณตอนนี้ แต่**ไม่รับประกันว่าจะถูกต้องเสมอ**บน optimization level อื่น หรือ compiler version อื่น
เพราะมันคือ undefined behavior ทางเทคนิคแล้ว ไม่ใช่แค่ "บั๊กที่ยังไม่เจอ"

#### 42.7.1 ข้อ "Initialized" ในทางปฏิบัติ: `MaybeUninit<T>`

Safety invariant ข้อที่ 3 (pointer ต้องชี้ไปยัง `T` ที่ initialized แล้วจริง) ฟังดูตรงไปตรงมา แต่คำถามที่
ตามมาคือ: แล้วก่อนที่ข้อมูลจะ initialized ล่ะ เราจะมี pointer ชี้ไปยังพื้นที่หน่วยความจำที่ "ยังไม่มีค่า"
ได้อย่างไรโดยไม่ละเมิดกฎของ Rust ที่ปกติบังคับว่าตัวแปรทุกตัวต้องมีค่าตั้งแต่ประกาศ (Part 3) — คำตอบคือ
`std::mem::MaybeUninit<T>` ซึ่งเราใช้ไปแล้วโดยไม่รู้ตัวใน `MyVec<T>` (ผ่าน `alloc::alloc` ที่คืนหน่วยความจำ
ดิบมาโดยไม่มีค่า `T` ใด ๆ อยู่ข้างในเลย) มาดูตัวอย่างสั้น ๆ ที่แสดงวงจรชีวิตแบบเต็มของมัน:

```rust
use std::mem::MaybeUninit;

fn main() {
    // MaybeUninit<T> คือ "กล่องเปล่า" ขนาดเท่ากับ T แต่ยังไม่มีค่า T ที่ valid อยู่ข้างในจริง
    let mut slot: MaybeUninit<i32> = MaybeUninit::uninit();

    // ก่อนเขียนค่า -- ตำแหน่งนี้ "ยังไม่ initialized" การอ่านผ่าน assume_init() ตอนนี้จะเป็น UB ทันที
    let ptr: *mut i32 = slot.as_mut_ptr();

    // SAFETY: ptr มาจาก slot.as_mut_ptr() โดยตรง ไม่ null แน่นอน และ i32 ไม่มี invariant พิเศษต้องรักษา
    // (ตัวเลขจำนวนเต็มธรรมดารับ bit pattern อะไรก็ได้) การเขียนทับตำแหน่งที่ยัง uninitialized ด้วย ptr::write
    // ปลอดภัยเสมอ เพราะไม่มีค่าเก่าที่ valid ให้ drop ก่อน (เหมือนกับที่ push() ใน MyVec<T> ทำ)
    unsafe {
        std::ptr::write(ptr, 42);
    }

    // ตอนนี้ initialized แล้วจริง -- เรียก assume_init() ได้อย่างปลอดภัย
    // SAFETY: เพิ่งเขียนค่า 42 ที่ valid ลงไปด้วย ptr::write ข้างบนเป็นขั้นตอนสุดท้ายก่อนหน้านี้พอดี
    let value = unsafe { slot.assume_init() };
    println!("value = {}", value);
}
```

ผลลัพธ์จริง: `value = 42` — สิ่งที่สำคัญที่สุดในตัวอย่างนี้คือชื่อ method `assume_init()` เอง: คำว่า
**"assume"** (สมมติ) บอกความหมายตรงตัวเลย — **มันไม่ได้ตรวจสอบอะไรให้เลยว่า initialized จริงหรือไม่**
มันเป็นแค่การที่คุณ**ยืนยัน**กับ compiler ว่า "ฉันมั่นใจว่าตำแหน่งนี้มีค่าที่ valid แล้วจริง ๆ" — ถ้าเรียก
`assume_init()` ก่อนเขียนค่าจริง (ลบบรรทัด `ptr::write` ออก) โค้ดจะยัง compile ผ่านเหมือนเดิมทุกอย่าง แต่
จะกลายเป็น undefined behavior ทันทีตอนรัน เพราะค่าที่อ่านออกมาจะเป็น garbage ที่ไม่มีความหมาย (หรือแย่กว่า
นั้นถ้า `T` เป็น type ที่ซับซ้อนกว่า เช่น `enum` ที่มี invalid discriminant) `MaybeUninit<T>` คือเครื่องมือ
มาตรฐานที่ std ใช้ภายในเองสำหรับสร้าง buffer ที่ยังไม่ initialize (เช่นเดียวกับที่ `Vec::with_capacity`
ทำภายใน) — และเป็นเครื่องมือหลักสำหรับโจทย์ `RingBuffer<T, N>` ในแบบฝึกหัดที่ 4 ท้ายบทด้วย

### 42.8 Memory Layout เจาะลึก: `repr(Rust)`, `repr(C)`, `repr(packed)`, `repr(transparent)`

ทีนี้เรามาถึงคำถามที่สองของบทนี้: เมื่อเราจะเดินไปมาในหน่วยความจำด้วย raw pointer เอง (เหมือนในหัวข้อ
42.5 และตัวอย่างเต็มรูปแบบในหัวข้อ 42.12) เราต้องรู้ให้แน่ชัดว่า struct หนึ่งตัววางตัวอยู่ในหน่วยความจำ
อย่างไร — field แต่ละตัวห่างกันกี่ไบต์ เรียงตามลำดับที่เขียนในซอร์สโค้ดหรือไม่ Rust ตอบคำถามนี้ผ่าน
**attribute `#[repr(...)]`** ที่กำกับไว้บน struct/enum แต่ละตัว — บทนี้จะพาดูสี่รูปแบบหลักที่ใช้บ่อยที่สุด

#### 42.8.1 `#[repr(Rust)]` — ค่า default ที่ compiler จัดการให้เอง

ถ้าไม่เขียน `#[repr(...)]` เลย struct ของคุณจะใช้ `repr(Rust)` โดยอัตโนมัติ — นี่คือ layout ที่**ไม่ระบุตาย
ตัว (unspecified)** อย่างเป็นทางการ compiler **มีสิทธิ์เต็มที่**ที่จะจัดเรียง field ใหม่ทั้งหมดให้ได้ขนาดที่
เล็กที่สุดหรือมี performance ดีที่สุดเท่าที่เป็นไปได้ — **ไม่จำเป็นต้องตรงกับลำดับที่คุณเขียนในซอร์สโค้ดเลย**
มาดูตัวอย่างจริงที่พิสูจน์เรื่องนี้:

```rust
use std::mem::{align_of, size_of};

// ลำดับ field แบบสลับขนาดไปมา (u8, u64, u8, u32, u8)
#[derive(Debug)]
struct MixedOrder {
    a: u8,
    b: u64,
    c: u8,
    d: u32,
    e: u8,
}

// ลำดับ field ที่จัดมือให้ใหญ่ไปเล็ก (u64, u32, u8, u8, u8)
#[derive(Debug)]
struct HandOrdered {
    b: u64,
    d: u32,
    a: u8,
    c: u8,
    e: u8,
}

fn main() {
    println!("size_of::<MixedOrder>()  = {}", size_of::<MixedOrder>());
    println!("size_of::<HandOrdered>() = {}", size_of::<HandOrdered>());
    println!("align_of::<MixedOrder>() = {}", align_of::<MixedOrder>());
    println!(
        "naive sum ของ field MixedOrder (1+8+1+4+1) = {}",
        1 + 8 + 1 + 4 + 1
    );
}
```

ผลลัพธ์จริงจากการรัน (วัดด้วย `rustc 1.94.1`, สถาปัตยกรรม x86_64):

```
size_of::<MixedOrder>()  = 16
size_of::<HandOrdered>() = 16
align_of::<MixedOrder>() = 8
naive sum ของ field MixedOrder (1+8+1+4+1) = 15
```

สังเกตผลลัพธ์นี้ให้ดี: **`MixedOrder` (ที่เราเขียน field สลับขนาดไปมาแบบไม่เป็นระเบียบ) ได้ขนาดเท่ากับ
`HandOrdered` (ที่เราจัดมือให้เรียงใหญ่ไปเล็กแล้ว) เป๊ะ — ทั้งคู่ 16 ไบต์** นี่ไม่ใช่ความบังเอิญ — มันคือ
เพราะ **`repr(Rust)` จัดเรียง field ใหม่ให้เราโดยอัตโนมัติอยู่แล้ว ไม่ว่าเราจะเขียนลำดับในซอร์สโค้ดแบบไหนก็
ตาม** compiler มองเห็นว่าวิธีจัดที่ดีที่สุดคือเอา `u64` ไว้ก่อน (ต้องการ alignment 8), ตามด้วย `u32`
(alignment 4), แล้วปิดท้ายด้วย `u8` สามตัวติดกัน (alignment 1 ไม่ต้องมี padding ระหว่างกันเลย) แล้วมันก็
จัดแบบนี้ให้ **ทั้งสอง struct** โดยไม่สนใจว่าเราเขียนลำดับอะไรมา — นี่คือความหมายของคำว่า "unspecified
layout, compiler-optimized": คุณไม่ควรพึ่งพาลำดับ field เพื่อคาดเดา layout จริงเลยกับ `repr(Rust)`

ข้อสำคัญที่ต้องเน้น: **ห้ามใช้ `repr(Rust)` (default) เพื่อสื่อสารกับโค้ดภาษาอื่น** (เช่น C) เพราะ layout
ของมัน**ไม่ได้ถูกกำหนดเป็นสัญญาที่คงที่** — แม้แต่ compiler version ถัดไปก็อาจจัดเรียงต่างจากตอนนี้ได้ (ไม่มี
การันตีความคงที่ข้าม compiler version ด้วยซ้ำ ถึงแม้ในทางปฏิบัติจะค่อนข้าง stable สำหรับ struct แบบง่าย ๆ)
สำหรับ FFI (Part 43 บทถัดไป) ต้องใช้ `repr(C)` เท่านั้น

#### 42.8.2 `#[repr(C)]` — Layout ที่ compatible กับภาษา C

`#[repr(C)]` เปลี่ยนกฎทั้งหมด: มันบอก compiler ว่า **"ห้ามจัดเรียง field ใหม่ ให้เรียงตามลำดับที่เขียนใน
ซอร์สโค้ดเป๊ะ ๆ เหมือนที่ C compiler ทำ"** — field แต่ละตัวจะถูกวางตามลำดับ พร้อม padding ที่แทรกเข้ามาเมื่อ
จำเป็น (ตามกฎ alignment ในหัวข้อ 42.9) แต่**ไม่มีการสลับลำดับเด็ดขาด**

```rust
use std::mem::size_of;

#[repr(Rust)]
struct RustLayout {
    a: u8,
    b: u64,
    c: u8,
}

#[repr(C)]
struct CLayout {
    a: u8,
    b: u64,
    c: u8,
}

fn main() {
    println!("size_of::<RustLayout>() = {}", size_of::<RustLayout>());
    println!("size_of::<CLayout>()    = {}", size_of::<CLayout>());
}
```

ผลลัพธ์จริง:

```
size_of::<RustLayout>() = 16
size_of::<CLayout>()    = 24
```

นี่คือผลลัพธ์ที่น่าสนใจมาก — **`CLayout` ใหญ่กว่า `RustLayout` ถึง 8 ไบต์ ทั้งที่มี field ชุดเดียวกันเป๊ะ**
(`u8`, `u64`, `u8`) เหตุผลคือ `repr(Rust)` มีสิทธิ์จัดเรียงใหม่เป็น `b, a, c` (u64 ก่อน แล้วตาม u8 สองตัว
ติดกันได้เลยไม่ต้องมี padding คั่น) แต่ `repr(C)` **ต้องคงลำดับ `a, b, c` ตามที่เขียนไว้** ผลคือ: `a`
(1 ไบต์) ตามด้วย padding 7 ไบต์เพื่อให้ `b` (ต้องการ align 8) เริ่มที่ offset ที่หาร 8 ลงตัว, แล้ว `b`
(8 ไบต์), แล้ว `c` (1 ไบต์) ตามด้วย padding อีก 7 ไบต์ท้าย struct เพื่อให้ขนาดรวมหาร alignment ของทั้ง
struct (8) ลงตัว — รวมเป็น 1+7+8+1+7 = 24 ไบต์

นี่คือเหตุผลเชิงปฏิบัติที่สำคัญที่สุดของ `repr(C)`: **มันคือ layout เดียวที่การันตีความ compatible กับ
struct ภาษา C ตัวต่อตัว** ซึ่งจำเป็นอย่างยิ่งสำหรับ FFI — ถ้าคุณกำลังจะส่ง struct ข้ามไปให้ C library ใช้งาน
(ผ่าน `extern "C"` ที่ Part 43 จะสอนเต็มรูปแบบ) struct นั้น**ต้อง**เป็น `#[repr(C)]` เท่านั้น ไม่ใช่
`repr(Rust)` เด็ดขาด เพราะ C compiler คาดหวัง field ตามลำดับที่ประกาศเป๊ะ ๆ (พร้อม padding rule เดียวกันกับ
ที่เราคำนวณข้างบน) — ถ้าใช้ `repr(Rust)` แล้วส่งข้าม FFI จะเกิดความหมายผิดพลาดทันที เพราะฝั่ง Rust กับฝั่ง C
"เข้าใจ" ตำแหน่งของแต่ละ field ต่างกัน แม้จะเป็น struct ที่ "ดูเหมือน" ตรงกันทุกตัวอักษรก็ตาม (จะเห็นตัวอย่าง
เต็มรูปแบบใน Part 43)

#### 42.8.3 `#[repr(packed)]` — ไม่มี Padding เลย พร้อมอันตรายที่ต้องระวัง

`#[repr(packed)]` (มักเขียนคู่กับ `repr(C)` เป็น `#[repr(C, packed)]`) บอก compiler ว่า **"ห้ามแทรก padding
เลยแม้แต่ไบต์เดียว วาง field ติดกันสนิทเสมอ"** ผลคือ struct มีขนาดเล็กที่สุดเท่าที่จะเป็นไปได้ (เท่ากับ
sum ของขนาด field ทั้งหมดเป๊ะ ไม่มีส่วนเกิน) แต่ต้องแลกมาด้วยราคาที่หนักมาก:

```rust
use std::mem::{align_of, size_of};

#[repr(C)]
struct Normal {
    a: u8,
    b: u32,
}

#[repr(C, packed)]
struct Packed {
    a: u8,
    b: u32,
}

fn main() {
    println!(
        "size_of::<Normal>() = {}, align_of::<Normal>() = {}",
        size_of::<Normal>(),
        align_of::<Normal>()
    );
    println!(
        "size_of::<Packed>() = {}, align_of::<Packed>() = {}",
        size_of::<Packed>(),
        align_of::<Packed>()
    );

    let p = Packed { a: 1, b: 0xAABBCCDD };
    // ห้ามอ้างอิง &p.b ตรง ๆ เพราะ field นี้ไม่ align -- ต้องอ่านผ่าน raw pointer + read_unaligned เท่านั้น
    let b_ptr: *const u32 = std::ptr::addr_of!(p.b);
    let b_val = unsafe { b_ptr.read_unaligned() };
    println!("b_val (อ่านผ่าน read_unaligned) = {:#X}", b_val);
}
```

ผลลัพธ์จริง:

```
size_of::<Normal>() = 8, align_of::<Normal>() = 4
size_of::<Packed>() = 5, align_of::<Packed>() = 1
b_val (อ่านผ่าน read_unaligned) = 0xAABBCCDD
```

`Normal` มี 8 ไบต์ (a=1 ไบต์ + padding 3 ไบต์ + b=4 ไบต์ เพราะ `u32` ต้องการ align 4) แต่ `Packed` มีแค่
5 ไบต์เป๊ะ (1+4 ไม่มี padding เลย) — ประหยัดได้ 3 ไบต์ต่อ struct ฟังดูดี แต่สังเกตว่า `align_of::<Packed>()`
กลายเป็น **1** ทั้งที่ field `b` เป็น `u32` ซึ่งปกติต้องการ align 4 — นี่คือ**อันตรายที่แท้จริง**ของ
`repr(packed)`: field `b` ของ `Packed` **ไม่ align ตามที่ประเภทของมันต้องการอีกต่อไป** เพราะมันอาจถูกวางไว้
ที่ offset ใดก็ได้ (ในตัวอย่างนี้คือ offset 1 ซึ่งไม่หาร 4 ลงตัว)

ผลที่ตามมาคือ **คุณสร้าง `&u32` ธรรมดา ๆ ไปชี้ที่ field `b` ตรง ๆ ไม่ได้อีกแล้ว** — ถ้าลองทำ compiler จะ
ปฏิเสธด้วย error ทันที (ดูกับดักที่ 2 ท้ายบท) เพราะการมี reference ที่ไม่ align เป็น undefined behavior
เสมอไม่ว่าจะ dereference มันหรือไม่ก็ตาม วิธีเข้าถึง field แบบนี้ได้อย่างปลอดภัยจริง ๆ คือต้องผ่าน raw
pointer เท่านั้น พร้อมใช้ `.read_unaligned()`/`.write_unaligned()` (แทน `*ptr`/`.read()`/`.write()` ธรรมดาที่
สมมติว่าข้อมูล align อยู่แล้ว) — เมธอดทั้งสองนี้ออกแบบมาเฉพาะสำหรับอ่าน/เขียนข้อมูลที่ไม่ align โดยไม่พึ่งพา
CPU instruction ที่ต้องการ alignment (บางสถาปัตยกรรมจะ crash ทันทีถ้าใช้ instruction ปกติกับข้อมูลไม่ align
— ดูหัวข้อ 42.9)

ด้วยความยุ่งยากและอันตรายระดับนี้ **`repr(packed)` ควรใช้เฉพาะกรณีจำเป็นจริง ๆ เท่านั้น** เช่นตอนต้องอ่าน
binary format ที่มีมาตรฐานกำหนดไว้ตายตัวแบบ byte-exact (เช่น network protocol header, ไฟล์ format บางชนิด)
ที่ไม่มี padding อยู่แล้วในตัว — สำหรับ struct ทั่วไปที่ต้องการแค่ประหยัดหน่วยความจำ ควรใช้เทคนิค**จัดเรียง
field ใหม่**ในหัวข้อ 42.10 แทน ซึ่งได้ผลประหยัดพื้นที่โดยไม่ต้องเจอปัญหา alignment เลย

#### 42.8.4 `#[repr(transparent)]` — Wrapper ที่มี Layout เหมือน Field เดียวข้างในทุกประการ

`#[repr(transparent)]` ใช้ได้กับ struct ที่มี**field ที่ไม่ใช่ zero-sized เพียงตัวเดียว**เท่านั้น (field
อื่น ๆ ถ้ามีต้องเป็น zero-sized type เช่น `PhantomData<T>`) มันการันตีว่า struct นี้จะมี `size_of`,
`align_of`, และ ABI (application binary interface — วิธีที่ค่าถูกส่งผ่าน function call ในระดับต่ำ) **ตรง
กับ field ข้างในทุกประการ 100%** — เหมือนกับว่า wrapper ตัวนี้ "ไม่มีอยู่จริง" ในสายตาของ layout เลย

```rust
use std::mem::{align_of, size_of};

#[repr(transparent)]
struct Meters(f64);

fn main() {
    println!("size_of::<Meters>()  = {}", size_of::<Meters>());
    println!("size_of::<f64>()     = {}", size_of::<f64>());
    println!("align_of::<Meters>() = {}", align_of::<Meters>());
    println!("align_of::<f64>()    = {}", align_of::<f64>());
    assert_eq!(size_of::<Meters>(), size_of::<f64>());
    assert_eq!(align_of::<Meters>(), align_of::<f64>());
}
```

ผลลัพธ์จริง:

```
size_of::<Meters>()  = 8
size_of::<f64>()     = 8
align_of::<Meters>() = 8
align_of::<f64>()    = 8
```

`Meters` และ `f64` มีขนาดและ alignment เท่ากันเป๊ะ — นี่คือความเชื่อมโยงตรงกับ **newtype pattern จาก Part
21**: เราใช้ newtype (`struct Meters(f64)`) เพื่อสร้าง type ใหม่ที่ type system แยกแยะจาก `f64` เปล่า ๆ ได้
(ป้องกันการเอา `Meters` ไปบวกกับ `Feet` โดยไม่ได้ตั้งใจ เป็นต้น) แต่โดย default (`repr(Rust)`) Rust**ไม่
การันตี**ว่า layout ของ newtype นี้จะเหมือนกับ field ข้างในเป๊ะ (ในทางปฏิบัติมันมักจะเหมือนอยู่แล้วสำหรับ
struct ที่มี field เดียว แต่ไม่ใช่ "สัญญา" ที่เป็นทางการ) `#[repr(transparent)]` คือการ**ยืนยันด้วยสัญญา
ทางการ**ว่า layout จะเหมือนกันแน่นอน 100% ซึ่งสำคัญมากในสองบริบท:

1. **FFI** (Part 43): เมื่อฟังก์ชัน C คาดหวัง `double` (เทียบเท่า `f64`) แต่คุณอยากส่ง `Meters` เข้าไปแทน
   เพื่อความปลอดภัยด้าน type บนฝั่ง Rust — `repr(transparent)` การันตีว่าการส่งแบบนี้ปลอดภัยแน่นอนใน ABI
   ระดับ binary เพราะ compiler รับประกันว่ามันคือ `f64` เปี๊ยบทุกไบต์
2. **การใช้ raw pointer ข้าม type**: ถ้าคุณมี `*const f64` และอยากตีความมันเป็น `*const Meters` (หรือกลับ
   กัน) ด้วย `as` cast — `repr(transparent)` การันตีว่าการตีความแบบนี้ปลอดภัย เพราะ layout เหมือนกันเป๊ะ
   ทุกไบต์ ไม่มีความเสี่ยงเรื่อง padding หรือ field เพิ่มเติมที่ซ่อนอยู่

### 42.9 Alignment: ทำไม CPU ถึงสนใจว่าข้อมูลอยู่ "ตรงตำแหน่ง" หรือไม่

**Alignment** ของ type หนึ่ง คือข้อกำหนดว่า address ของมันในหน่วยความจำ**ต้องหารด้วยเลขนี้ลงตัว** เช่น
`u32` มี alignment 4 หมายความว่า `u32` ทุกตัวต้องอยู่ที่ address ที่เป็นพหุคูณของ 4 (เช่น 0, 4, 8, 12, ...
ไม่ใช่ 1, 2, 3, 5, ...) เราวัดค่านี้ได้ผ่าน `std::mem::align_of::<T>()`:

```rust
use std::mem::align_of;

struct Empty;

fn main() {
    println!("align_of::<u8>()    = {}", align_of::<u8>());
    println!("align_of::<u16>()   = {}", align_of::<u16>());
    println!("align_of::<u32>()   = {}", align_of::<u32>());
    println!("align_of::<u64>()   = {}", align_of::<u64>());
    println!("align_of::<i128>()  = {}", align_of::<i128>());
    println!("align_of::<f64>()   = {}", align_of::<f64>());
    println!("align_of::<bool>()  = {}", align_of::<bool>());
    println!("align_of::<char>()  = {}", align_of::<char>());
    println!("align_of::<&i32>()  = {}", align_of::<&i32>());
    println!("align_of::<Empty>() = {}", align_of::<Empty>());
}
```

ผลลัพธ์จริง (x86_64):

```
align_of::<u8>()    = 1
align_of::<u16>()   = 2
align_of::<u32>()   = 4
align_of::<u64>()   = 8
align_of::<i128>()  = 16
align_of::<f64>()   = 8
align_of::<bool>()  = 1
align_of::<char>()  = 4
align_of::<&i32>()  = 8
align_of::<Empty>() = 1
```

สังเกตกฎที่เห็นได้จากตัวเลขนี้: **สำหรับ primitive type ส่วนใหญ่ alignment เท่ากับขนาดของมันเอง** (`u32`
ขนาด 4 ไบต์ align 4, `u64` ขนาด 8 ไบต์ align 8) และ `align_of::<&i32>()` เท่ากับ 8 เพราะ reference บน
สถาปัตยกรรม 64-bit ก็คือ pointer ขนาด 8 ไบต์ (เป็น "ที่อยู่" ที่ align เท่ากับขนาดตัวมันเองเช่นกัน) ส่วน
`Empty` (struct ไม่มี field เลย) มี alignment 1 (ค่าต่ำสุดที่เป็นไปได้ เพราะไม่มีข้อกำหนดพิเศษอะไรเลย)

**ทำไม alignment ถึงสำคัญ?** เหตุผลมาจากวิธีที่ CPU เข้าถึงหน่วยความจำจริง ๆ ในระดับฮาร์ดแวร์ CPU ไม่ได้
อ่านหน่วยความจำทีละไบต์ — มันอ่านเป็น**บล็อก** (เช่น 8 ไบต์ต่อครั้งสำหรับ 64-bit bus) ถ้าข้อมูลที่ต้องการ
วางอยู่ตรง "ขอบ" ของบล็อกพอดี (คือ align ถูกต้อง) CPU อ่านมันได้ด้วย**การเข้าถึงหน่วยความจำครั้งเดียว** แต่
ถ้าข้อมูลนั้นคาบเกี่ยวสองบล็อก (misaligned) CPU **บางตัว**ต้องทำการอ่านสองครั้งแล้วเอาผลมาประกอบกันเอง — ช้า
กว่าอย่างมีนัยสำคัญ และที่ร้ายแรงกว่าความช้าคือ: **บนสถาปัตยกรรมบางตัว (เช่น ARM รุ่นเก่าบางรุ่น หรือโหมด
strict alignment บางแบบ) การเข้าถึงข้อมูลที่ไม่ align ไม่ใช่แค่ช้า — มันคือ hardware fault ที่ทำให้โปรแกรม
crash ทันที** ไม่ใช่แค่ performance penalty เหมือนบน x86 ทั่วไปเท่านั้น

นี่คือเหตุผลที่ Rust (และ LLVM ที่ Rust ใช้เป็น backend) ถือว่า **การสร้าง reference ที่ไม่ align เป็น
undefined behavior เสมอ ไม่ว่าสถาปัตยกรรมที่รันจริงจะทนได้หรือไม่ก็ตาม** — เพราะโค้ดต้อง compile ให้ทำงาน
ถูกต้องได้บน**ทุก**สถาปัตยกรรมที่ Rust รองรับ ไม่ใช่แค่เครื่องที่คุณกำลังทดสอบอยู่ตอนนี้ compiler จะสมมติ
เสมอว่า reference ทุกตัวถูก align มาถูกต้องแล้ว และอาจ optimize โค้ดโดยอิงกับสมมติฐานนี้ (เช่น เลือก CPU
instruction แบบที่เร็วที่สุดสำหรับข้อมูล aligned) — ถ้าสมมติฐานนี้ผิดจริง ผลลัพธ์คาดเดาไม่ได้เลย

### 42.10 Padding: ไบต์ที่มองไม่เห็นแต่มีอยู่จริง และเทคนิคจัดเรียง Field ลดขนาด

**Padding** คือไบต์เปล่า ๆ ที่ compiler แทรกเข้าไประหว่าง field (หรือท้าย struct) เพื่อให้ field ถัดไป
(หรือ struct ตัวถัดไปในกรณีเป็น array ของ struct) ได้ alignment ที่ถูกต้อง — เราเห็นตัวอย่างนี้ไปแล้วใน
`repr(C)` ของหัวข้อ 42.8.2 (`CLayout` ที่ได้ 24 ไบต์ ทั้งที่ field รวมกันมีแค่ 10 ไบต์ — ส่วนต่าง 14 ไบต์คือ
padding ทั้งหมด) กฎทั่วไปคือ: **alignment ของ struct ทั้งก้อนเท่ากับ alignment ที่มากที่สุดในหมู่ field
ของมัน** และ padding จะถูกแทรกเท่าที่จำเป็นเพื่อให้ทุก field เริ่มต้นที่ offset ที่ align ถูกต้องสำหรับตัว
มันเอง รวมถึงแทรกท้าย struct เพื่อให้ขนาดรวมทั้งก้อนหาร alignment ของ struct ลงตัวด้วย (จำเป็นเวลามี array
ของ struct นี้ต่อ ๆ กัน เพื่อให้ตัวที่สองก็ align ถูกต้องเช่นกัน)

มาดูตัวอย่างที่แสดงผลของการจัดเรียง field ใหม่แบบเห็นได้ชัดเจนที่สุด — ใช้ `#[repr(C)]` เพื่อบังคับให้
compiler เรียงตามลำดับที่เขียนจริง (เพื่อโชว์ผลของการจัดเรียงมือเอง ไม่ให้ `repr(Rust)` แอบจัดให้อัตโนมัติ
เหมือนหัวข้อ 42.8.1):

```rust
use std::mem::size_of;

// สลับ u64 กับ bool ไปมา -- ทำให้ต้องแทรก padding หลายช่วงเพื่อให้ u64/u32 ได้ alignment ที่ถูกต้อง
#[repr(C)]
struct Interleaved {
    flag1: bool,
    id: u64,
    flag2: bool,
    count: u32,
    flag3: bool,
}

// จัดกลุ่ม field ขนาดใหญ่ (alignment สูง) ไว้ก่อน แล้วค่อยตามด้วย field เล็ก ๆ รวมกันไว้ท้าย
#[repr(C)]
struct Grouped {
    id: u64,
    count: u32,
    flag1: bool,
    flag2: bool,
    flag3: bool,
}

fn main() {
    let naive_sum = 1 + 8 + 1 + 4 + 1; // bool=1, u64=8, bool=1, u32=4, bool=1
    println!("naive sum ของ field ทั้งหมด = {}", naive_sum);
    println!("size_of::<Interleaved>() (repr(C)) = {}", size_of::<Interleaved>());
    println!("size_of::<Grouped>()     (repr(C)) = {}", size_of::<Grouped>());
}
```

ผลลัพธ์จริง:

```
naive sum ของ field ทั้งหมด = 15
size_of::<Interleaved>() (repr(C)) = 32
size_of::<Grouped>()     (repr(C)) = 16
```

ตัวเลขนี้คือหัวใจของหัวข้อนี้ทั้งหมด: **แค่จัดเรียง field ใหม่ (field ชุดเดียวกันเป๊ะ ไม่มีการเพิ่มหรือลด
อะไร) `Interleaved` ใช้พื้นที่ 32 ไบต์ ในขณะที่ `Grouped` ใช้แค่ 16 ไบต์ — ลดลงครึ่งหนึ่งเป๊ะ!** มาไล่ดูว่า
padding มาจากไหนใน `Interleaved`:

| offset | field | ขนาด | หมายเหตุ |
|---|---|---|---|
| 0 | `flag1: bool` | 1 | — |
| 1–7 | *(padding)* | 7 | ต้องรอให้ `id` เริ่มที่ offset ที่หาร 8 ลงตัว |
| 8–15 | `id: u64` | 8 | offset 8 หาร 8 ลงตัว ✓ |
| 16 | `flag2: bool` | 1 | — |
| 17–19 | *(padding)* | 3 | ต้องรอให้ `count` เริ่มที่ offset ที่หาร 4 ลงตัว |
| 20–23 | `count: u32` | 4 | offset 20 หาร 4 ลงตัว ✓ |
| 24 | `flag3: bool` | 1 | — |
| 25–31 | *(padding ท้าย struct)* | 7 | ให้ขนาดรวม (32) หาร alignment ของ struct (8) ลงตัว |

รวม padding ทั้งหมด 7+3+7 = 17 ไบต์ บวกกับ field จริง 15 ไบต์ เท่ากับ 32 ไบต์พอดี ในขณะที่ `Grouped` เอา
`id` (align 8) ไว้ก่อนสุด (offset 0 หาร 8 ลงตัวเลย ไม่ต้อง padding), ตามด้วย `count` (align 4, offset 8 หาร
4 ลงตัวเลย), แล้วปิดท้ายด้วย `bool` สามตัวติดกันที่ offset 12, 13, 14 (ไม่ต้อง padding ระหว่างกันเพราะ
align 1 ทั้งหมด) เหลือ offset 15 ว่างพอดี 1 ไบต์ ให้ขนาดรวมเป็น 16 ซึ่งหาร 8 ลงตัวพอดีโดยไม่ต้องเติม padding
ท้ายเพิ่มเลย

**นี่คือเทคนิคการ optimize หน่วยความจำที่ใช้งานได้จริงในโปรเจกต์จริง**: เวลาออกแบบ struct ที่จะถูกสร้างเป็น
จำนวนมาก ๆ (เช่น struct ที่แทน entity นับล้านตัวใน game engine, หรือ record นับล้านแถวในระบบประมวลผลข้อมูล)
การจัดเรียง field จาก**alignment สูงไปต่ำ** (ปกติคือขนาดใหญ่ไปเล็ก) สามารถลดขนาดรวมของโปรแกรมได้อย่างมี
นัยสำคัญ — ในตัวอย่างข้างบนคือลดลงครึ่งหนึ่งโดยไม่ต้องเสียอะไรเลย (ไม่มีความเสี่ยงแบบ `repr(packed)` เพราะ
ทุก field ยังคง align ถูกต้องตามปกติ 100% เหมือนเดิม) ข้อสังเกตสำคัญ: เทคนิคนี้มีผลชัดเจนกับ `repr(C)` และ
struct ที่ต้อง fix ลำดับ (เช่น FFI) — ส่วน `repr(Rust)` (default) นั้น compiler ทำสิ่งนี้ให้เราอัตโนมัติอยู่
แล้วอย่างที่เห็นในหัวข้อ 42.8.1 (`MixedOrder` กับ `HandOrdered` ได้ขนาดเท่ากันแม้ลำดับต่างกัน) แต่การเข้าใจ
กลไกนี้ให้ลึกยังมีประโยชน์เสมอ เพราะช่วยให้อ่านและคาดเดา behavior ของ `#[repr(C)]` ได้แม่นยำ ซึ่งจำเป็น
อย่างยิ่งเมื่อเข้าสู่ Part 43 (FFI)

### 42.11 `std::mem::transmute`: รู้จักไว้ แต่ควรหลีกเลี่ยง

`std::mem::transmute::<T, U>(value)` คือการ**ตีความ bit pattern ดิบ ๆ ของ `T` ใหม่เป็น `U` ตรง ๆ** โดยไม่
มีการแปลงค่าใด ๆ เกิดขึ้นเลย (ต่างจาก `as` cast ที่แปลง**ค่า** เช่น `3.7_f64 as i32` ได้ `3` เพราะมันตัด
ทศนิยมออกจริง ๆ — `transmute` ไม่ทำแบบนั้น มันแค่เอาไบต์ดิบชุดเดิมมา "แปะป้าย" เป็น type ใหม่)

นี่คือ**หนึ่งในการกระทำที่อันตรายที่สุดในภาษา Rust ทั้งหมด** เพราะมันข้ามระบบ type safety ไปเต็มรูปแบบ —
compiler ไม่ตรวจสอบความสมเหตุสมผลของการแปลงให้เลยนอกจาก**ขนาด**ของทั้งสอง type ต้องเท่ากันเป๊ะ (ถ้าไม่เท่า
กันจะเป็น compile error ทันที ดูกับดักที่ 4 ท้ายบท) แต่ต่อให้ขนาดเท่ากัน การตีความ bit pattern ผิดความหมาย
ก็ยังสร้างปัญหาร้ายแรงได้ เช่น transmute ตัวเลขปกติไปเป็น enum ที่มี invalid discriminant, หรือ transmute
`&T` ไปเป็น `&mut T` (ละเมิดกฎ aliasing ทันทีโดยไม่มีการเตือนใด ๆ), หรือ transmute ไปเป็น type ที่มี
invariant ภายในที่ต้องตรวจสอบ (เช่น `&str` ต้องเป็น valid UTF-8 เสมอ — transmute ไบต์สุ่ม ๆ ไปเป็น `&str`
โดยไม่ตรวจ UTF-8 ก่อนคือ undefined behavior ทันที)

มาดูตัวอย่างเดียวที่เป็นการใช้งานที่ "ถูกต้องปลอดภัย" — การตีความ bit pattern ของตัวเลขระหว่าง type ที่มี
ขนาดเท่ากันเป๊ะ พร้อมเทียบกับวิธีที่ปลอดภัยกว่าซึ่งควรใช้แทนในสถานการณ์จริง:

```rust
fn main() {
    // ตัวอย่าง "ปลอดภัย" ของ transmute: u32 <-> [u8; 4] ที่มีขนาดเท่ากันเป๊ะ (4 ไบต์ทั้งคู่)
    let n: u32 = 0x11223344;
    let bytes: [u8; 4] = unsafe { std::mem::transmute(n) };
    println!("bytes = {:02X?}", bytes);

    // วิธีปลอดภัยกว่าที่ทำสิ่งเดียวกันได้ โดยไม่ต้องใช้ transmute หรือ unsafe เลย -- ควรใช้แทนเสมอ
    let bytes_safe: [u8; 4] = n.to_ne_bytes();
    println!("bytes_safe = {:02X?}", bytes_safe);
    assert_eq!(bytes, bytes_safe);

    let back: u32 = unsafe { std::mem::transmute(bytes) };
    let back_safe = u32::from_ne_bytes(bytes_safe);
    assert_eq!(back, back_safe);

    // ตัวอย่างการตีความ bit pattern ของ f32 เป็น u32 (บิตต่อบิต ไม่ใช่การแปลงค่าเชิงตัวเลข)
    let f: f32 = 1.5;
    let bits_transmute: u32 = unsafe { std::mem::transmute(f) };
    let bits_safe: u32 = f.to_bits(); // วิธีที่ std เตรียมให้ ปลอดภัยกว่าและอ่านง่ายกว่า
    println!(
        "bits_transmute = {:#010X}, bits_safe = {:#010X}",
        bits_transmute, bits_safe
    );
    assert_eq!(bits_transmute, bits_safe);
}
```

ผลลัพธ์จริง (แม้จะ compile ผ่านและรันได้ถูกต้อง แต่สังเกต compiler warning ด้านล่าง):

```
bytes = [44, 33, 22, 11]
bytes_safe = [44, 33, 22, 11]
bits_transmute = 0x3FC00000, bits_safe = 0x3FC00000
```

ทุก `assert_eq!` ผ่านหมด ยืนยันว่าทั้งสองวิธีให้ผลลัพธ์เดียวกัน แต่ที่น่าสนใจคือ **compiler เวอร์ชันปัจจุบัน
(`rustc 1.94.1`) ฉลาดพอที่จะเตือนเราตรง ๆ ว่าไม่ควรใช้ `transmute` ในทุกกรณีข้างบนนี้**:

```
warning: unnecessary transmute
 --> src/main.rs:4:35
  |
4 |     let bytes: [u8; 4] = unsafe { std::mem::transmute(n) };
  |                                   -------------------^^^
  |                                   |
  |                                   help: replace this with: `u32::to_ne_bytes`
  |
  = help: there's also `to_le_bytes` and `to_be_bytes` if you expect a particular byte order
  = note: `#[warn(unnecessary_transmutes)]` on by default
```

lint `unnecessary_transmutes` นี้คือหลักฐานที่ชัดเจนที่สุดว่า**แนวทางที่ทีม Rust แนะนำคือ: มองหาทางเลือกที่
ปลอดภัยกว่าก่อนเสมอ** ทางเลือกที่ปลอดภัยกว่าซึ่งครอบคลุมกรณีใช้งานส่วนใหญ่ที่คนอยากใช้ `transmute` ได้แก่:

- **`as`**: สำหรับแปลงค่าตัวเลขระหว่าง numeric type (ไม่ใช่ตีความ bit pattern แต่แปลงค่าจริง ๆ ตามกฎที่
  Part ต้น ๆ ของหลักสูตรสอนไว้)
- **`.to_ne_bytes()` / `.from_ne_bytes()`** (และ `to_le_bytes`/`to_be_bytes`/`from_le_bytes`/
  `from_be_bytes`): สำหรับแปลงตัวเลขเป็น/จาก array ของไบต์ พร้อมควบคุม byte order ได้ชัดเจน (native, little
  endian, big endian) ซึ่ง `transmute` ทำไม่ได้เลยเพราะมันขึ้นกับ native representation ของเครื่องเสมอ
- **`.to_bits()` / `::from_bits()`**: สำหรับ `f32`/`f64` โดยเฉพาะ ตีความ bit pattern เป็น/จาก integer แบบ
  ปลอดภัยและมี method ให้ตรงตัว
- **`TryFrom`/`From`**: สำหรับแปลงระหว่าง custom type ที่ต้องการ validate ความถูกต้องของข้อมูลระหว่างทาง

**ข้อสรุปของหัวข้อนี้**: `mem::transmute` เป็นเครื่องมือที่**ควรรู้ว่ามีอยู่และเข้าใจว่ามันทำอะไร** (เพราะจะ
เจอมันในโค้ด unsafe บางแห่งที่ไม่มีทางเลือกอื่นจริง ๆ เช่นในงาน FFI ขั้นสูงหรือ low-level optimization บาง
ประเภท) แต่**ไม่ควรเป็นตัวเลือกแรกที่นึกถึง**ในโค้ดทั่วไป — ทุกครั้งที่มือกำลังจะพิมพ์ `transmute` ให้หยุด
ถามตัวเองก่อนว่า "มี safe alternative ไหมที่ทำสิ่งเดียวกันได้" เกือบทุกครั้งคำตอบคือมี

### 42.12 ตัวอย่างจริง: สร้าง Growable Buffer (`MyVec<T>`) ด้วย Raw Pointer

ตอนนี้เรามีทุกเครื่องมือที่จำเป็นแล้ว: การสร้าง/dereference raw pointer, pointer arithmetic, และความเข้าใจ
memory layout — มาประกอบทุกอย่างเข้าด้วยกันเป็นตัวอย่างที่สมบูรณ์และสมจริง: การสร้าง**growable buffer ของ
เราเอง** ที่ทำงานคล้าย `std::vec::Vec<T>` ในระดับพื้นฐาน (`push`, `pop`, `get`, และ automatic growth) โดย
จัดการหน่วยความจำเองทั้งหมดผ่าน `std::alloc::alloc`/`realloc`/`dealloc` แทนที่จะพึ่ง `Vec<T>` ของ std เลย

คำถามที่คนมักถามคือ "ทำไมต้องเขียนเองในเมื่อ `Vec<T>` มีให้ใช้อยู่แล้ว" — คำตอบสำหรับบทเรียนนี้ไม่ใช่ "เพื่อ
ใช้งานจริงแทน `Vec<T>`" (ในโปรเจกต์จริงควรใช้ `Vec<T>` ของ std เสมอ มันผ่านการทดสอบและ optimize มาอย่างดี
กว่าโค้ดที่เราเขียนเองมาก) แต่เพื่อ**เข้าใจว่า `Vec<T>` (และโครงสร้างข้อมูลแบบ dynamic อื่น ๆ ที่ Part 27
เคยกล่าวถึงผ่าน ๆ) ทำงานอย่างไรข้างใน** — ความเข้าใจนี้จะช่วยให้อ่าน error message เกี่ยวกับ ownership/
capacity ของ `Vec<T>` ได้ลึกซึ้งขึ้น และเป็นพื้นฐานสำคัญสำหรับการเขียน custom data structure ของตัวเองใน
งานจริงที่ `Vec<T>` มาตรฐานไม่ตอบโจทย์ (เช่น ring buffer, arena allocator, หรือ SIMD-aligned buffer)

```rust
use std::alloc::{self, Layout};
use std::ptr::{self, NonNull};

/// MyVec<T> คือ growable buffer แบบง่าย ๆ ที่จำลองกลไกภายในของ std::vec::Vec<T>
/// เพื่อสอน raw pointer navigation แบบครบวงจร: การจอง/คืนหน่วยความจำเอง (manual allocation),
/// pointer arithmetic, และการ dereference ทั้งหมดถูกซ่อนอยู่หลัง safe API เพื่อไม่ให้ผู้ใช้
/// ภายนอกต้องยุ่งกับ unsafe เลยแม้แต่นิดเดียว (safe abstraction pattern จาก Part 41)
struct MyVec<T> {
    ptr: NonNull<T>, // raw pointer ที่การันตีว่าไม่ null ด้วย type เอง (NonNull<T> จาก std::ptr)
    len: usize,      // จำนวน element ที่ใช้งานจริงอยู่ตอนนี้ (initialized แล้วทุกตัว)
    cap: usize,      // จำนวน element ที่จองพื้นที่ไว้แล้วทั้งหมด (อาจมากกว่า len)
}

impl<T> MyVec<T> {
    fn new() -> Self {
        MyVec {
            ptr: NonNull::dangling(), // ยังไม่ได้จองหน่วยความจำจริง เพราะ cap = 0 (ยังไม่มีการ alloc)
            len: 0,
            cap: 0,
        }
    }

    fn layout_for(cap: usize) -> Layout {
        Layout::array::<T>(cap).expect("layout คำนวณไม่ได้ (ขนาดเกิน isize::MAX)")
    }

    /// ขยายพื้นที่: ถ้า cap = 0 จองใหม่ตั้งแต่ 4 ตัว ไม่งั้นเพิ่มเป็นสองเท่า (amortized growth --
    /// เทคนิคเดียวกับที่ Vec<T> ของ std ใช้ เพื่อให้ต้นทุนเฉลี่ยของ push ยังเป็น O(1) ในระยะยาว)
    fn grow(&mut self) {
        let new_cap = if self.cap == 0 { 4 } else { self.cap * 2 };
        let new_layout = Self::layout_for(new_cap);

        let new_ptr = if self.cap == 0 {
            // SAFETY: new_layout มีขนาด > 0 เสมอเพราะ new_cap >= 4 และ T ไม่ใช่ zero-sized ในตัวอย่างนี้
            unsafe { alloc::alloc(new_layout) }
        } else {
            let old_layout = Self::layout_for(self.cap);
            // SAFETY:
            // 1. self.ptr มาจาก allocator ตัวเดียวกันเสมอ (alloc::alloc/realloc เท่านั้น ไม่เคยมาจากที่อื่น)
            // 2. old_layout ตรงกับ layout ที่ใช้ตอน allocate ครั้งก่อนเป๊ะ (คำนวณจาก self.cap เดิม)
            // 3. new_layout.size() > 0 เสมอ (new_cap คูณต่อจาก cap เดิมที่ไม่ใช่ 0)
            unsafe { alloc::realloc(self.ptr.as_ptr() as *mut u8, old_layout, new_layout.size()) }
        };

        // ตรวจ null ทันทีตามข้อกำหนดของ GlobalAlloc: allocation ล้มเหลวได้จริง (out of memory)
        let new_ptr = match NonNull::new(new_ptr as *mut T) {
            Some(p) => p,
            None => alloc::handle_alloc_error(new_layout),
        };

        self.ptr = new_ptr;
        self.cap = new_cap;
    }

    fn push(&mut self, value: T) {
        if self.len == self.cap {
            self.grow();
        }
        // SAFETY:
        // 1. self.ptr ชี้ไปยัง block หน่วยความจำที่จองไว้ขนาด self.cap element เสมอ (invariant ของ struct นี้)
        // 2. self.len < self.cap ตรวจแล้วข้างบน (grow ถ้าเต็มไปแล้ว) จึง add(self.len) ยังอยู่ในขอบเขตที่จองไว้
        // 3. ตำแหน่งนี้ยังไม่มีค่าที่ valid (uninitialized) การเขียนทับด้วย write() จึงไม่ไป drop ค่าเก่าที่ไม่มีอยู่จริง
        unsafe {
            ptr::write(self.ptr.as_ptr().add(self.len), value);
        }
        self.len += 1;
    }

    fn pop(&mut self) -> Option<T> {
        if self.len == 0 {
            return None;
        }
        self.len -= 1;
        // SAFETY: ตำแหน่ง self.len (หลังลดแล้ว) เป็น element ที่ initialized จริงอยู่ในขอบเขตที่จองไว้
        // ptr::read จะ "ย้ายค่าออกมา" โดยไม่ไป double-drop ตำแหน่งเดิม (ตำแหน่งนั้นถือว่า "ไม่มีค่า" อีกต่อไป
        // ตาม logical invariant ที่เราดูแลเองผ่าน self.len)
        Some(unsafe { ptr::read(self.ptr.as_ptr().add(self.len)) })
    }

    fn get(&self, index: usize) -> Option<&T> {
        if index >= self.len {
            return None;
        }
        // SAFETY: index < self.len <= self.cap ตรวจแล้วข้างบน ตำแหน่งนี้ initialized และอยู่ในขอบเขตที่จองไว้แน่นอน
        Some(unsafe { &*self.ptr.as_ptr().add(index) })
    }

    fn len(&self) -> usize {
        self.len
    }
}

impl<T> Drop for MyVec<T> {
    fn drop(&mut self) {
        // ต้อง pop ทุกตัวออกก่อน เพื่อให้ T::drop ถูกเรียกครบทุก element (สำคัญมากถ้า T เก็บ heap
        // allocation ของตัวเอง เช่น String หรือ Vec -- ถ้าลืมขั้นนี้จะเกิด memory leak ทันที)
        while self.pop().is_some() {}
        if self.cap != 0 {
            let layout = Self::layout_for(self.cap);
            // SAFETY: self.ptr มาจาก alloc::alloc/realloc ด้วย layout เดียวกันนี้เป๊ะ และเพิ่งเคลียร์
            // element ทั้งหมดไปแล้วข้างบน จึงคืนหน่วยความจำได้อย่างปลอดภัย
            unsafe {
                alloc::dealloc(self.ptr.as_ptr() as *mut u8, layout);
            }
        }
    }
}

fn main() {
    let mut v: MyVec<String> = MyVec::new();
    v.push("หนึ่ง".to_string());
    v.push("สอง".to_string());
    v.push("สาม".to_string());
    println!("len หลัง push 3 ตัว = {}", v.len());

    for i in 0..v.len() {
        println!("v[{}] = {:?}", i, v.get(i));
    }

    let popped = v.pop();
    println!("pop() ได้ = {:?}, len เหลือ = {}", popped, v.len());

    // ทดสอบ growth หลายครั้งเพื่อยืนยันว่า realloc ทำงานถูกต้อง (เดิม cap=4 -> เพิ่มเป็น 8 -> 16 โดยอัตโนมัติ)
    let mut nums: MyVec<i32> = MyVec::new();
    for i in 0..10 {
        nums.push(i);
    }
    println!("nums.len() = {}", nums.len());
    for i in 0..nums.len() {
        print!("{} ", nums.get(i).unwrap());
    }
    println!();

    // v และ nums ถูก drop ตอนจบ main -- Drop::drop ของเราคืนหน่วยความจำให้ allocator ครบทุก block
}
```

ผลลัพธ์จริงจากการรัน (ตรวจแล้วว่าไม่มี memory leak หรือ crash ใด ๆ):

```
len หลัง push 3 ตัว = 3
v[0] = Some("หนึ่ง")
v[1] = Some("สอง")
v[2] = Some("สาม")
pop() ได้ = Some("สาม"), len เหลือ = 2
nums.len() = 10
0 1 2 3 4 5 6 7 8 9
```

มาไล่ดูจุดที่ raw pointer ถูกใช้งานทั้งหมดในตัวอย่างนี้ ทีละจุด:

- **`NonNull<T>`** — เป็น wrapper รอบ `*mut T` จาก `std::ptr` ที่การันตีด้วย type ว่าค่าข้างในไม่เป็น null
  เด็ดขาด (ถ้าพยายามสร้างจาก pointer ที่เป็น null จะได้ `None` กลับมาจาก `NonNull::new`) — มันคือตัวอย่าง
  ของการใช้ type system มาช่วยลดพื้นที่ที่ต้องรับผิดชอบด้วย `unsafe` เอง (แทนที่จะต้องเช็ค `.is_null()`
  ด้วยตัวเองทุกครั้งที่ใช้ `self.ptr`) — เป็นเทคนิค safe abstraction ตรงตาม pattern ที่ Part 41 สอนไว้
- **`grow()`** — ใช้ `alloc::alloc`/`alloc::realloc` ตรง ๆ (unsafe function ตามพลังข้อ 2 ของ Part 41) เพื่อ
  จองและขยายหน่วยความจำ พร้อมคำนวณ `Layout` ที่ตรงกับ `size_of::<T>()`/`align_of::<T>()` ให้ถูกต้องเสมอ
  ผ่าน `Layout::array::<T>()` — นี่คือจุดที่ความเข้าใจเรื่อง alignment จากหัวข้อ 42.9 มาบรรจบกับการจัดการ
  หน่วยความจำจริง: allocator ต้องรู้ทั้งขนาดและ alignment ที่ต้องการ ไม่ใช่แค่ขนาดเพียงอย่างเดียว
- **`push()`** — ใช้ `ptr::write()` (ไม่ใช่ `*ptr = value` ธรรมดา) เพราะตำแหน่งที่เขียนยัง**ไม่มีค่าเดิม
  ที่ valid อยู่** (`uninitialized memory`) — ถ้าใช้ `*ptr = value` ธรรมดา Rust จะพยายาม drop ค่าเก่าที่
  ตำแหน่งนั้นก่อนเขียนค่าใหม่ทับ (ตามกฎ `Drop` ปกติ) ซึ่งจะเป็น undefined behavior ทันทีเพราะไม่มีค่าเก่า
  ที่ valid ให้ drop จริง ๆ — `ptr::write()` เขียนทับแบบ "ดิบ" โดยไม่พยายาม drop อะไรก่อนเลย ถูกออกแบบมา
  เฉพาะสำหรับกรณีนี้
- **`pop()`** — ใช้ `ptr::read()` เพื่อ "ย้าย" ค่าออกจากตำแหน่งในหน่วยความจำโดยไม่ไป drop ตำแหน่งนั้นซ้ำ
  (ทำสำเนา bit pattern ออกมาเป็นค่าใหม่ที่ caller เป็นเจ้าของ แล้วถือว่าตำแหน่งเดิม "ไม่มีค่าที่ valid"
  อีกต่อไปในทางตรรกะ แม้ไบต์เดิมจะยังอยู่ในหน่วยความจำก็ตาม) — ตรงกับกลไก **move semantics** ที่ Part 6
  สอนไว้ แต่คราวนี้เราต้องทำเองด้วยมือในระดับ raw memory
- **`get()`** — ใช้ pointer arithmetic (`.add(index)`) ตรงตามหัวข้อ 42.5 แล้วแปลงกลับเป็น `&T` ด้วย `&*ptr`
  ตรงตามหัวข้อ 42.7 พร้อมตรวจ safety invariant ทั้งสี่ข้อผ่าน comment SAFETY ที่แนบไว้
- **`Drop::drop()`** — เรียก `pop()` วนจนหมดก่อน (เพื่อให้ `T::drop` ของทุก element ถูกเรียกครบ ป้องกัน
  memory leak ถ้า `T` เป็น type ที่มี heap allocation เอง เช่น `String`) แล้วค่อยคืน block หน่วยความจำทั้ง
  ก้อนด้วย `alloc::dealloc` — ลำดับนี้สำคัญมาก: **ต้อง drop element ก่อนคืน memory เสมอ** สลับลำดับไม่ได้

#### 42.12.1 `MyVec<T>` เชื่อมโยงกับ `Box<T>`, `Vec<T>`, และ `Rc<T>` อย่างไร

ตอนนี้เราสร้าง `MyVec<T>` ครบวงจรแล้ว ควรใช้เวลาสักครู่มองย้อนกลับไปที่ smart pointer ทั้งสามตัวจาก Part
27–29 เพื่อเห็นภาพว่าพวกมันสร้างจากส่วนผสมเดียวกันนี้เป๊ะ เพียงแค่ปรับรายละเอียดตาม use case ของตัวเอง:

- **`Box<T>`** (Part 27) คือรุ่นที่**เรียบง่ายที่สุด**: ภายในมันเก็บแค่ `Unique<T>` (คล้าย `NonNull<T>` ที่
  เราใช้ แต่มี metadata เพิ่มเติมสำหรับบอก compiler ว่ามันเป็นเจ้าของแบบ exclusive) ชี้ไปยัง**หนึ่ง**ค่าบน
  heap ไม่มีเรื่อง `len`/`cap` หรือ growth เพราะ `Box<T>` เก็บได้แค่ค่าเดียวเท่านั้น `Drop` ของมันก็ทำสิ่ง
  เดียวกับ `MyVec<T>::drop()` แบบย่อ: drop ค่าข้างในก่อน แล้วเรียก `dealloc` คืนหน่วยความจำ — มันคือ
  `MyVec<T>` เวอร์ชันที่ `cap` ถูกตรึงไว้ที่ 1 เสมอนั่นเอง
- **`Vec<T>`** ของ std เองคือสิ่งที่ `MyVec<T>` **จำลองมา**เกือบทั้งหมด: field ภายในของมันคือ `RawVec<T>`
  (ที่ห่อ `ptr`, `cap` ไว้อีกชั้น) บวกกับ `len` — ตรงกับ struct `MyVec<T>` ของเราทุกประการในเชิงแนวคิด
  ความแตกต่างหลักคือ std ปรับแต่ง growth strategy, error handling, และ edge case (เช่น zero-sized type)
  ให้ครอบคลุมกว่ามาก แต่กลไกหลัก (`alloc`/`realloc`/`dealloc` + `ptr::write`/`ptr::read` + pointer
  arithmetic) เหมือนกันเป๊ะกับที่เราเขียนในหัวข้อ 42.12
- **`Rc<T>`** (Part 28) คือ `Box<T>` ที่เพิ่ม**reference count** เข้ามาอีกชั้น (ตามที่ Part 40 หัวข้อ 40.4
  อธิบายกลไกตัวนับไว้แล้ว) — ภายในมันจอง allocation หนึ่งก้อนที่ใหญ่กว่า `T` เพียวๆ เพราะต้องเก็บทั้ง
  `strong_count`, `weak_count`, และ `T` ไว้ด้วยกัน (เรียกกันว่า `RcBox<T>`) การเข้าถึง field พวกนี้ก็ทำผ่าน
  raw pointer arithmetic แบบเดียวกับที่เราเห็นในบทนี้ทั้งหมด

ข้อคิดสำคัญที่ได้จากการเปรียบเทียบนี้: **`unsafe`/raw pointer ไม่ใช่ "ทางลัดที่แย่" ที่ควรหลีกเลี่ยงเสมอ
ไป** — มันคือฐานรากที่ทำให้ std เขียน abstraction ระดับสูงที่ทั้งปลอดภัยและมีประสิทธิภาพสูงสุดให้เราใช้งาน
ได้แบบไม่ต้องคิดเรื่อง memory เองเลยในโค้ดทั่วไป สิ่งที่ต่างกันระหว่างโค้ดของ std กับโค้ดทั่วไปคือ**ปริมาณ
การตรวจสอบและทดสอบ**ที่ทุ่มเทให้กับ `unsafe` block เพียงไม่กี่บรรทัดนั้น (ทั้ง code review อย่างเข้มงวด,
fuzzing, และการรัน Miri ตรวจ UB อย่างสม่ำเสมอ) — ปรัชญา safe abstraction จาก Part 41 ที่ว่า "เขียน
`unsafe` ให้น้อยที่สุด ห่อให้แน่นที่สุด" นี่แหละคือสิ่งที่ std ทำมาตลอดตั้งแต่ `Box<T>` จนถึง `Vec<T>`

### 42.13 ตารางสรุป: เมื่อไหร่ควรใช้ Raw Pointer จริง ๆ

ก่อนเข้าสู่หัวข้อกับดัก มาสรุปภาพรวมว่าในโค้ดจริงเราควรใช้ raw pointer เมื่อไหร่ และหลีกเลี่ยงเมื่อไหร่:

| สถานการณ์ | ควรใช้อะไร |
|---|---|
| โค้ดทั่วไปในโปรแกรม Rust ปกติ | `&T`/`&mut T` เสมอ — raw pointer ไม่จำเป็นเลย |
| สร้าง safe abstraction ที่ borrow checker พิสูจน์ไม่ได้ (เช่น `split_at_mut`, `MyVec<T>`) | raw pointer ภายใน implementation แต่เปิด API แบบ safe ให้ผู้ใช้ภายนอก (Part 41 pattern) |
| FFI กับ C library | raw pointer + `#[repr(C)]` (Part 43) |
| ทำ binary parsing/serialization ที่ต้องคุม layout เป๊ะ | `#[repr(C)]` หรือ `#[repr(packed)]` (ถ้าจำเป็นจริง) ร่วมกับ raw pointer |
| ต้องการ optimize ขนาด struct | จัดเรียง field ใหม่ (หัวข้อ 42.10) — **ไม่ต้องใช้ raw pointer เลย** |
| แปลงตัวเลข/bit pattern ระหว่าง type | `as`, `to_ne_bytes`/`from_ne_bytes`, `to_bits`/`from_bits` — **หลีกเลี่ยง `transmute`** |

## กับดักที่พบบ่อย (Common Pitfalls)

### กับดักที่ 1: Dangling Raw Pointer — Pointer ที่ยังอยู่แต่ข้อมูลหายไปแล้ว

เพราะ raw pointer ไม่มี lifetime ให้ compiler ติดตาม (ตามตารางเปรียบเทียบในหัวข้อ 42.2) มันจึง**เป็นไปได้
เสมอ**ที่จะสร้าง raw pointer ชี้ไปยังข้อมูลที่ไม่มีอยู่จริงอีกแล้ว โดย compile ผ่านสนิท:

```rust
fn dangling() -> *const i32 {
    let x = 5;
    &x as *const i32 // คืน raw pointer ที่ชี้ไปยัง x ซึ่งกำลังจะถูก drop ทันทีที่ฟังก์ชันนี้ return
} // <- x ถูกทำลายที่นี่ (stack frame ของฟังก์ชันถูกเก็บกลับคืน)

fn main() {
    let p = dangling();
    unsafe {
        // ต่อไปนี้คือ undefined behavior จริง: p ชี้ไปยังหน่วยความจำที่ไม่ใช่ของ x แล้ว
        println!("ค่าที่ได้ (ไม่ควรเชื่อถือ ค่านี้คือ UB): {}", *p);
    }
}
```

น่าสังเกตว่า **`rustc` เวอร์ชันปัจจุบันฉลาดพอที่จะเตือนกรณีนี้ที่ตรงเป๊ะ** ด้วย lint ใหม่:

```
warning: function returns a dangling pointer to dropped local variable `x`
 --> src/main.rs:5:5
  |
3 | fn dangling() -> *const i32 {
  |                  ---------- return type is `*const i32`
4 |     let x = 5;
  |         - local variable `x` is dropped at the end of the function
5 |     &x as *const i32
  |     --^^^^^^^^^^^^^^
  |     |
  |     dangling pointer created here
  |
  = note: a dangling pointer is safe, but dereferencing one is undefined behavior
  = note: `#[warn(dangling_pointers_from_locals)]` on by default
```

สังเกตข้อความ `= note: a dangling pointer is safe, but dereferencing one is undefined behavior` — ตรง
ตามหัวข้อ 42.3 เป๊ะ: **การสร้างมันปลอดภัย แต่การใช้มันไม่ปลอดภัย** lint นี้ (`dangling_pointers_from_locals`)
เป็น**เครื่องมือช่วยเหลือ**ที่ compiler ให้มา แต่ไม่ใช่การรับประกัน — มันจับได้แค่กรณีที่เห็นได้ชัดในซอร์ส
โค้ดฟังก์ชันเดียวเท่านั้น (local variable ที่ return address ออกไปตรง ๆ) กรณีที่ dangling pointer เกิดจาก
โค้ดที่ซับซ้อนกว่านั้น (เช่น ผ่านหลายฟังก์ชัน หรือมาจากการ `free`/`dealloc` ข้อมูลไปแล้วแต่ pointer เก่ายัง
ถูกเก็บไว้ใช้ต่อ) จะไม่มี warning เตือนเลย **วิธีป้องกันที่แท้จริงคือมีวินัยเรื่อง lifetime ของข้อมูลที่
pointer ชี้ไปเสมอ** ไม่ว่า compiler จะเตือนหรือไม่ก็ตาม — ถ้าเป็นไปได้ ให้ raw pointer มี lifetime ที่สั้น
ที่สุดเท่าที่จำเป็น และไม่เก็บมันไว้ข้ามช่วงที่ข้อมูลต้นทางอาจถูกทำลาย

### กับดักที่ 2: Misaligned Access — สร้าง Reference ไปยังข้อมูลที่ไม่ Align

จากหัวข้อ 42.8.3 (`repr(packed)`) — การพยายามสร้าง reference ธรรมดาไปยัง field ที่ไม่ align จะถูก compiler
ปฏิเสธทันที (ไม่ใช่แค่ warning แต่เป็น hard error):

```rust
#[repr(C, packed)]
struct Packed {
    a: u8,
    b: u32,
}

fn main() {
    let p = Packed { a: 1, b: 2 };
    let r: &u32 = &p.b; // พยายามสร้าง reference ตรง ๆ ไปยัง field ที่ไม่ align
    println!("{}", r);
}
```

Error จริง:

```
error[E0793]: reference to field of packed struct is unaligned
 --> src/main.rs:9:19
  |
9 |     let r: &u32 = &p.b; // พยายามสร้าง reference ตรง ๆ ไปยัง field ที่ไม่ align
  |                   ^^^^
  |
  = note: this struct is 1-byte aligned, but the type of this field may require higher alignment
  = note: creating a misaligned reference is undefined behavior (even if that reference is never dereferenced)
  = help: copy the field contents to a local variable, or replace the reference with a raw pointer and use `read_unaligned`/`write_unaligned` (loads and stores via `*p` must be properly aligned even when using raw pointers)
```

สังเกตว่า compiler เองระบุ**ทางแก้ไว้ให้ตรง ๆ** ในบรรทัด `= help`: มีสองทางเลือก — (1) copy ค่าไปเก็บใน
ตัวแปร local ก่อน (เช่น `let b = p.b;` ซึ่งเป็นการ copy ค่าดิบ ไม่ใช่การสร้าง reference จึงไม่ผิดกฎ
alignment) หรือ (2) ใช้ raw pointer พร้อม `.read_unaligned()`/`.write_unaligned()` ตามที่แสดงในหัวข้อ
42.8.3 — ข้อความ `even if that reference is never dereferenced` ย้ำประเด็นสำคัญ: **แค่การมี reference ที่
ไม่ align อยู่เฉย ๆ ก็เป็น UB แล้ว ไม่ต้องรอให้ dereference ก่อน** ต่างจากกรณี raw pointer ที่ dangling ใน
กับดักที่ 1 (ที่การสร้างปลอดภัย แต่การ dereference ไม่ปลอดภัย) — สำหรับ reference ที่ misaligned การสร้าง
เองก็ผิดกฎไปแล้วทันที

### กับดักที่ 3: ละเมิดกฎ Aliasing โดยการให้ `&mut T` กับ Raw Pointer อยู่ร่วมกัน

นี่คือกับดักที่**อันตรายที่สุด**ในบรรดาทั้งหมด เพราะมันไม่ทำให้เกิด compile error หรือแม้แต่ warning เลย
สักครั้ง — compiler ปล่อยผ่านสนิท แต่โค้ดกลับละเมิด safety invariant ข้อที่ 4 จากหัวข้อ 42.7 (ห้ามมี
reference/pointer อื่นเข้าถึงข้อมูลเดียวกันขณะที่ `&mut T` ยัง alive):

```rust
fn main() {
    let mut value = 10;
    let m: &mut i32 = &mut value;
    let raw: *mut i32 = m as *mut i32; // สร้าง raw pointer จาก &mut ที่ยัง "อยู่" ตัวเดิม

    unsafe {
        *raw += 1; // เขียนผ่าน raw pointer ขณะที่ m (&mut) ก็ยังถือ borrow นี้อยู่ในตัวแปรเดียวกัน
    }
    *m += 1; // ใช้ m ต่อ -- ทั้งสองบรรทัดนี้เข้าถึงข้อมูลเดียวกันจาก "เส้นทาง" คนละเส้น
    println!("value = {}", value);
}
```

โค้ดนี้ compile ผ่านโดยไม่มี warning เลยแม้แต่ตัวเดียว และรันได้ผลลัพธ์ `value = 12` ตรงตามที่คาดไว้ (10 → 11
ผ่าน `raw` → 12 ผ่าน `m`) บนคอมไพเลอร์และ optimization level ที่ทดสอบ — **แต่นี่ไม่ได้แปลว่าโค้ดนี้ถูกต้อง**
สิ่งที่เกิดขึ้นจริงคือ: การสร้าง `raw` จาก `m` แล้วใช้ `raw` เขียนข้อมูล จากนั้นกลับมาใช้ `m` ต่อ คือรูปแบบที่
ละเมิดสมมติฐาน "no-alias" ที่ compiler ใช้กับ `&mut T` เสมอ (หัวข้อ 42.7) — LLVM optimizer ของ Rust
**อนุญาตให้ตัวเองสมมติได้เต็มที่**ว่า `&mut i32` อย่าง `m` ไม่มีใครมาแก้ไขข้อมูลเดียวกันจากทางอื่นเลยตลอด
ช่วงที่มันยัง alive — สมมติฐานนี้ผิดในโค้ดข้างบน (เพราะ `raw` แก้ไขข้อมูลเดียวกันระหว่างทาง) ทำให้เกิด
undefined behavior ในทางเทคนิค แม้ผลลัพธ์ตัวเลขที่เห็นจะ "ดูถูกต้อง" ก็ตาม

ผลที่ตามมาของ UB แบบนี้คือ**ไม่แน่นอน**: บน optimization level อื่น (`-O` ต่างระดับ), compiler version อื่น
ในอนาคต หรือแม้แต่การเรียงลำดับ instruction ต่างกันเล็กน้อยรอบ ๆ โค้ดนี้ ก็อาจทำให้ผลลัพธ์ที่ได้ต่างออกไป
โดยไม่มีสัญญาณเตือนใด ๆ ล่วงหน้า — นี่คือธรรมชาติของ undefined behavior ที่แตกต่างจาก bug ปกติ: มันไม่
"พัง" ให้เห็นทันทีเสมอไป แต่ทำให้โค้ดอยู่ในสถานะที่**ไม่มีการรับประกัน**อะไรเลยจากภาษา

**วิธีป้องกัน**: กฎง่าย ๆ ที่ควรยึดถือคือ **ห้ามให้ `&mut T` และ raw pointer ที่ชี้ข้อมูลเดียวกันถูกใช้งาน
สลับกันไปมาในช่วงเวลาเดียวกันเด็ดขาด** — ถ้าจำเป็นต้องแปลง `&mut T` เป็น raw pointer เพื่อทำ pointer
arithmetic หรือส่งเข้า FFI ให้ใช้ raw pointer นั้น**อย่างเดียว**ตลอดช่วงที่ต้องการ แล้วค่อยกลับไปใช้ reference
เดิมทีหลัง (ไม่สลับใช้ทั้งสองแบบพร้อมกัน) เครื่องมือ **Miri** (interpreter สำหรับตรวจ undefined behavior ที่
มาพร้อม toolchain แบบ nightly ผ่าน `cargo +nightly miri run`) เป็นเครื่องมือที่ทีม Rust แนะนำให้ใช้ตรวจโค้ด
`unsafe` ทุกครั้งก่อนเชื่อว่ามันถูกต้องจริง เพราะมันจำลอง Stacked/Tree Borrows model (โมเดลที่ Rust ใช้นิยาม
กฎ aliasing อย่างเป็นทางการ) และจะรายงานการละเมิดแบบนี้ออกมาชัดเจน แม้ผลลัพธ์ตัวเลขปกติจะดู "ถูกต้อง" ก็ตาม

### กับดักที่ 4: ใช้ `transmute` ผิดขนาด — Compile Error ที่ป้องกันความผิดพลาดร้ายแรงกว่า

ข่าวดีเรื่องหนึ่งของ `mem::transmute` คือ compiler **บังคับ**ให้ทั้งสอง type มีขนาดเท่ากันเป๊ะเสมอ (เป็น
หนึ่งในไม่กี่การตรวจสอบที่ยังเหลืออยู่) — ถ้าไม่เท่ากันจะเป็น compile error ทันที ไม่ปล่อยให้กลายเป็น UB
ที่รันแล้วค่อยพัง:

```rust
fn main() {
    let n: u32 = 10;
    // พยายาม transmute จาก u32 (4 ไบต์) ไปเป็น u64 (8 ไบต์) -- ขนาดไม่เท่ากัน
    let m: u64 = unsafe { std::mem::transmute(n) };
    println!("{}", m);
}
```

Error จริง:

```
error[E0512]: cannot transmute between types of different sizes, or dependently-sized types
 --> src/main.rs:4:27
  |
4 |     let m: u64 = unsafe { std::mem::transmute(n) };
  |                           ^^^^^^^^^^^^^^^^^^^
  |
  = note: source type: `u32` (32 bits)
  = note: target type: `u64` (64 bits)
```

ข้อควรจำจากกับดักนี้คือ **การตรวจขนาดเป็นแค่การตรวจสอบเดียวที่ compiler ทำให้** — มันตรวจแค่ "จำนวนไบต์เท่า
กันไหม" **ไม่ได้ตรวจ**ว่าการตีความ bit pattern นั้น**สมเหตุสมผล**หรือไม่เลย ตัวอย่างที่ compile ผ่านสนิทแต่
เป็นอันตรายกว่าเดิมคือการ transmute ไปเป็น enum ที่มี invalid discriminant (เช่น มี enum ที่ประกาศไว้ว่ามี
แค่ 3 ค่าที่เป็นไปได้ แต่ transmute ตัวเลข `99` ที่ไม่ตรงกับค่าใดเลยเข้าไป) หรือ transmute `bool` จากไบต์ที่
ไม่ใช่ `0`/`1` (ซึ่ง `bool` ใน Rust ต้องมีแค่สองค่า bit pattern ที่ valid เท่านั้น) กรณีเหล่านี้ **compile
ผ่านได้แม้จะเป็น undefined behavior เต็มรูปแบบตอนรัน** — นี่คือเหตุผลที่หัวข้อ 42.11 เน้นย้ำว่าควรมองหา
alternative ที่ปลอดภัยกว่าเสมอ แทนที่จะพึ่งพา "การตรวจขนาด" เพียงอย่างเดียวเป็นเครื่องยืนยันความถูกต้อง

### กับดักที่ 5: ลืมตรวจ Null ก่อน Dereference

กับดักพื้นฐานที่สุดแต่ก็ยังพบได้บ่อยเมื่อทำงานกับ raw pointer ที่มาจากแหล่งภายนอก (เช่นค่าที่ได้จาก FFI
หรือจาก allocator เมื่อ allocation ล้มเหลว) คือการลืมเช็ค `.is_null()` ก่อน dereference — ต่างจาก
`Option<T>` ที่ Part 9 สอนไว้ ซึ่งบังคับให้ต้อง `match`/`unwrap`/`?` อย่างชัดเจนก่อนใช้ค่าข้างใน raw pointer
**ไม่มีการบังคับอะไรแบบนั้นเลย** — `*ptr` compile ผ่านได้เสมอไม่ว่า `ptr` จะ null หรือไม่:

```rust
fn get_value_dangerous(ptr: *const i32) -> i32 {
    unsafe { *ptr } // ไม่เช็ค null ก่อนเลย -- ถ้า ptr เป็น null จะ crash (segmentation fault) ตอนรัน
}
```

ฟังก์ชันนี้ compile ผ่านสนิทไม่มี warning ใด ๆ (เพราะ `unsafe` บอก compiler ไว้แล้วว่า "ฉันรับผิดชอบเอง")
แต่ถ้าเรียกด้วย `get_value_dangerous(std::ptr::null())` โปรแกรมจะ crash ทันทีตอนรัน (ต่างจาก UB แบบเงียบ ๆ
ในกับดักที่ 3 — null pointer dereference มักจะทำให้ crash แบบเห็นชัดในทางปฏิบัติ เพราะ address 0 มักไม่ได้
ถูก map ไว้ในหน่วยความจำจริงของโปรเซส แต่นี่ก็ยังเป็น undefined behavior ในทางเทคนิค ไม่ใช่ "พฤติกรรมที่
รับประกัน" ว่าจะ crash เสมอบนทุกสถาปัตยกรรม) วิธีป้องกันคือทำตาม pattern ในหัวข้อ 42.6 เสมอ: เช็ค
`.is_null()` ก่อนเข้า `unsafe` block ทุกครั้งที่ pointer อาจมาจากแหล่งที่ไม่รับประกันว่าไม่ null (โดยเฉพาะ
ค่าที่ผ่าน FFI มา — Part 43 จะเจาะเรื่องนี้ลึกกว่านี้อีก)

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เขียนฟังก์ชัน `fn swap_via_raw_ptr<T: Copy>(a: *mut T, b: *mut T)` ที่สลับค่าของสองตำแหน่ง
   ในหน่วยความจำโดยใช้ raw pointer และ `unsafe { *a }`/`unsafe { *b }` ล้วน ๆ (ไม่ใช้ `std::mem::swap`)
   ทดสอบกับตัวแปร `i32` สองตัว แล้วพิสูจน์ว่าค่าสลับกันจริงหลังเรียกฟังก์ชัน
   *Hint*: ต้องเก็บค่าตัวหนึ่งไว้ในตัวแปร local ชั่วคราวก่อน ไม่งั้นจะเขียนทับค่าเดิมหายไปก่อนอ่านเสร็จ —
   ลำดับคือ อ่าน `*a` เก็บไว้ → เขียน `*b` ทับ `*a` → เขียนค่าที่เก็บไว้ทับ `*b`

2. **(กลาง)** กำหนด struct สามแบบต่อไปนี้ (ทั้งหมดมี field ชุดเดียวกัน: `bool`, `f64`, `bool`, `u16`,
   `bool`) ให้เขียนโปรแกรมพิมพ์ `size_of` ของทั้งสามแบบด้วย `#[repr(C)]` กำกับทุกตัว แล้ววิเคราะห์ (เขียน
   คำอธิบายเป็น comment) ว่า struct ไหนใช้พื้นที่น้อยที่สุดและทำไม พร้อมจัดเรียง field ของตัวที่สามให้ได้
   ขนาดเล็กที่สุดเท่าที่เป็นไปได้:
   - Struct A: `bool, f64, bool, u16, bool` (ลำดับตามที่กำหนด)
   - Struct B: `f64, u16, bool, bool, bool` (จัดใหญ่ไปเล็ก)
   - Struct C: ลำดับที่คุณคิดว่าดีที่สุด (ให้ลองคำนวณด้วยมือก่อนรันเพื่อยืนยัน)

3. **(ยาก)** เพิ่ม method `fn insert(&mut self, index: usize, value: T)` ให้กับ `MyVec<T>` จากหัวข้อ 42.12
   ที่แทรกค่าเข้าไปที่ตำแหน่ง `index` โดยต้อง**เลื่อน element ทั้งหมดที่อยู่หลัง `index` ไปทางขวาหนึ่งตำแหน่ง
   ก่อน** (ใช้ `ptr::copy` หรือเขียนวน loop เองด้วย pointer arithmetic ก็ได้) แล้วค่อยเขียนค่าใหม่ลงไปที่
   ตำแหน่งที่ว่างลง อย่าลืมเรียก `grow()` ก่อนถ้า `len == cap` เหมือนเดิม เขียน SAFETY comment อธิบายทุก
   `unsafe` block ที่เพิ่มเข้ามาให้ครบ
   *Hint*: ศึกษา `std::ptr::copy` (ต่างจาก `copy_nonoverlapping` ตรงที่ region ต้นทางและปลายทางซ้อนกันได้
   อย่างปลอดภัย ซึ่งจำเป็นมากสำหรับการเลื่อน element ในที่เดิม)

4. **(ยาก/ประยุกต์)** ออกแบบ `RingBuffer<T, const N: usize>` (fixed-size circular buffer ที่ใช้ const
   generic จาก Part 18) ที่เก็บข้อมูลใน `[MaybeUninit<T>; N]` ภายใน พร้อม `head`/`tail`/`len` สำหรับติดตาม
   ตำแหน่ง ให้มี method `push_back(&mut self, value: T) -> Result<(), T>` (คืน `Err(value)` ถ้าเต็มแล้ว)
   และ `pop_front(&mut self) -> Option<T>` ที่ใช้ raw pointer + `ptr::write`/`ptr::read` เข้าถึง element
   ภายใน `MaybeUninit<T>` แต่ละตัวโดยตรง (ไม่ใช้ `Vec<T>` เลย) พร้อมเขียน `Drop` implementation ที่ drop
   เฉพาะ element ที่ initialized จริงเท่านั้น (ระหว่าง `head` ถึง `tail` ตามจำนวน `len`)
   *Hint*: `MaybeUninit<T>::as_mut_ptr()` ให้ raw pointer ไปยังพื้นที่หน่วยความจำโดยไม่สนใจว่า initialized
   หรือยัง — โจทย์นี้รวมทุกอย่างที่เรียนในบทนี้เข้าด้วยกัน: layout ของ array คงที่ (const generic ขนาดตายตัว
   ไม่ต้อง allocate เอง), pointer arithmetic แบบวงกลม (`(index + 1) % N`), และ safety invariant ที่ต้อง
   รักษาด้วยตัวเองทั้งหมด

## สรุป

บทนี้เปิดกล่องที่ Part 41 ตั้งใจปิดไว้ให้ดูสั้น ๆ — **raw pointer** และ **memory layout** สองเรื่องที่เป็น
รากฐานของงาน low-level ทั้งหมดใน Rust เราเห็นความแตกต่างที่แม่นยำระหว่าง `*const T`/`*mut T` กับ `&T`/
`&mut T` ในสี่มิติ (validity, null-ability, aliasing, lifetime tracking) และเข้าใจนัยสำคัญที่มักถูกมองข้าม:
**การสร้าง raw pointer ปลอดภัยเสมอ มีแค่การ dereference เท่านั้นที่ต้อง `unsafe`** เราเรียนรู้การเดินทางใน
หน่วยความจำด้วย pointer arithmetic (`.add()`/`.sub()`/`.offset()`) ที่คำนวณขนาดไบต์ให้อัตโนมัติจาก
`size_of::<T>()`, การแปลงกลับไปเป็น reference พร้อม**safety invariant สี่ข้อ**ที่ต้องรักษาให้ครบ (non-null,
aligned, initialized, no conflicting alias) และปิดท้ายด้วยการเจาะลึก memory layout ทั้งสี่แบบ
(`repr(Rust)`, `repr(C)`, `repr(packed)`, `repr(transparent)`) พร้อมตัวเลขจริงที่วัดได้ — รวมถึงเทคนิคจัด
เรียง field ที่ลดขนาด struct ลงได้ถึงครึ่งหนึ่งในตัวอย่างจริงที่เราทดสอบ (32 ไบต์ → 16 ไบต์) ตัวอย่าง
`MyVec<T>` ในหัวข้อ 42.12 ประกอบทุกเครื่องมือเข้าด้วยกันเป็นโครงสร้างข้อมูลที่ทำงานได้จริง สะท้อนให้เห็นว่า
`Vec<T>`, `Box<T>`, และสารพัด smart pointer ที่เราใช้กันมาตลอดหลักสูตร (Part 27–29) ล้วนสร้างจากส่วนผสม
เดียวกันนี้ทั้งหมดที่เบื้องหลัง — เพียงแค่ห่อไว้ให้ปลอดภัยแล้วเท่านั้น

ประเด็นที่ควรติดตัวไปใช้ในงานจริงมากที่สุดจากบทนี้: **raw pointer เป็นเครื่องมือสำหรับสร้าง safe
abstraction ไม่ใช่เครื่องมือสำหรับ API สาธารณะ** — เขียน `unsafe` ให้น้อยที่สุด เก็บมันไว้ในฟังก์ชันเล็ก ๆ
ที่ตรวจสอบได้ง่าย แนบ `// SAFETY:` comment อธิบายทุกครั้งไม่มีข้อยกเว้น และมองหา safe alternative ก่อนเสมอ
เมื่อคิดจะใช้ `transmute` หรือ `repr(packed)` — ทั้งสองอย่างนี้มีเหตุผลให้มีอยู่จริง แต่ไม่ใช่ตัวเลือกแรกที่
ควรนึกถึง

ตอนนี้เรามีเครื่องมือครบสำหรับก้าวเข้าสู่หัวข้อที่ raw pointer และ `repr(C)` ถูกใช้งานอย่างเข้มข้นที่สุด:
**Part 43: FFI (Foreign Function Interface)** — การเชื่อมต่อโค้ด Rust กับ library ที่เขียนด้วยภาษา C
โดยตรง ผ่าน `extern "C"`, `#[no_mangle]`, และเครื่องมืออย่าง `bindgen` — ทุกอย่างที่เราเรียนในบทนี้ (โดย
เฉพาะ `repr(C)` และ safety invariant ของการแปลง pointer กลับเป็น reference) จะกลายเป็นเครื่องมือที่ใช้งาน
จริงทุกวันในบทนั้น

---

**Part ก่อนหน้า:** [Unsafe Rust เบื้องต้น](part-041-unsafe-basics.md) | **Part ถัดไป:**
[FFI: การเชื่อมต่อกับ C](part-043-ffi-c.md)
