# Part 29: Smart Pointers: Weak<T>, Cow<T>, Interior Mutability Patterns

> โมดูล: ระดับกลาง (Intermediate) | ระดับ: กลาง | เวลาโดยประมาณ: 200 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้ว่า **memory leak จากวงจรอ้างอิง** (reference cycle) ที่ Part 28 ทิ้งปริศนาไว้ตอนจบเกิดขึ้นได้อย่างไร และ
  พิสูจน์ด้วยโค้ดจริงว่า `Rc<RefCell<T>>` สองตัวที่ชี้กลับไปกลับมาทำให้ `strong_count` ไม่มีวันถึงศูนย์ ทำให้ destructor
  ไม่ถูกเรียกเลยแม้ตัวแปรทั้งหมดจะหมด scope ไปแล้ว
- แยกแยะแนวคิด **strong reference** กับ **weak reference** ได้อย่างชัดเจนในเชิงปรัชญา ("ฉันต้องให้ข้อมูลนี้มีชีวิตอยู่"
  เทียบกับ "ฉันแค่อยากรู้ว่าข้อมูลนี้ยังอยู่ไหม แต่ไม่ได้ยืนยันว่ามันต้องอยู่เพราะฉัน") และเชื่อมโยงกับกลไกการนับ
  reference ของ `Rc<T>` ที่เรียนมาจาก Part 28
- ใช้ `Rc::downgrade()` เพื่อสร้าง `Weak<T>` จาก `Rc<T>` และใช้ `Weak::upgrade()` เพื่อพยายามแปลงกลับเป็น `Rc<T>`
  อย่างปลอดภัย พร้อมอธิบายได้ว่าทำไม `upgrade()` ต้องคืนค่าเป็น `Option<Rc<T>>` โดยเชื่อมกับปรัชญาการออกแบบ `Option<T>`
  จาก **Part 11** ตรง ๆ — `None` ไม่ใช่ error แต่คือคำตอบที่ถูกต้องเมื่อข้อมูลถูกทำลายไปแล้ว
- ออกแบบโครงสร้างข้อมูลแบบ **tree/parent-child** ที่ถูกต้องด้วยแนวคิด "เจ้าของกับผู้ยืม" — ลูกเป็นเจ้าของโดยพ่อแม่ผ่าน
  `Rc<RefCell<T>>` (ความสัมพันธ์แบบ "เป็นเจ้าของ") ส่วนพ่อแม่ถูกอ้างอิงกลับโดยลูกผ่าน `Weak<RefCell<T>>` (ความสัมพันธ์
  แบบ "รู้จักแต่ไม่ได้เป็นเจ้าของ") และพิสูจน์ด้วย `strong_count()`/`weak_count()` ว่าการออกแบบแบบนี้ไม่มี memory leak
- อธิบายความแตกต่างระหว่าง `Weak<T>` ที่ upgrade แล้วได้ `None` อย่างปลอดภัย กับ dangling pointer ใน C/C++ ที่นำไปสู่
  undefined behavior ได้ว่าทำไม Rust ถึง "ปลอดภัยกว่า" โดยไม่ต้องมี garbage collector คอยตรวจสอบตอน runtime
- ใช้ `Cow<T>` (Clone on Write) เพื่อเขียนฟังก์ชันที่ **หลีกเลี่ยงการ allocate ข้อมูลใหม่โดยไม่จำเป็น** — คืนค่าแบบ
  borrowed เมื่อไม่มีอะไรต้องแก้ไข และคืนค่าแบบ owned เฉพาะเมื่อจำเป็นต้องแก้ไขข้อมูลจริง ๆ พร้อมใช้ `.to_mut()` และ
  `.into_owned()` ได้อย่างถูกต้อง
- เข้าใจภาพรวมทั้งหมดของ "ตระกูล interior mutability" (`Cell<T>`, `RefCell<T>`, `Mutex<T>`/`RwLock<T>`) ในฐานะ
  spectrum ของเครื่องมือที่แลก tradeoff กันคนละแบบ และเลือกใช้ smart pointer ที่ถูกต้อง (`Box<T>`, `Rc<T>`,
  `RefCell<T>`, `Weak<T>`, `Cow<T>`) ให้เหมาะกับสถานการณ์ได้อย่างมีเหตุผล ไม่ใช่แค่จำ syntax

## ความรู้ที่ต้องมีมาก่อน

บทนี้เป็น **บทปิดของ mini-arc "Smart Pointers"** ที่เริ่มจาก Part 27 และต่อยอดจาก Part 28 อย่างหนักมาก แนะนำให้ทวน
สองบทนี้ให้แน่นก่อนอ่านบทนี้ เพราะทุกหัวข้อในบทนี้อ้างอิงกลับไปที่นั่นตรง ๆ:

- **Part 27 (Smart Pointers: `Box<T>`)**: แนวคิดพื้นฐานว่า smart pointer คือ struct ที่ทำตัวเหมือน pointer (ผ่าน
  trait `Deref`/`DerefMut`) แต่มี "พฤติกรรมเพิ่มเติม" ติดมาด้วย และ `Box<T>` คือ smart pointer ที่ simple ที่สุด — เป็น
  เจ้าของข้อมูลบน heap แบบ **เจ้าของเดียว (single ownership)** ไม่มีการนับ reference ใด ๆ เลย
- **Part 28 (Smart Pointers: `Rc<T>` และ `RefCell<T>`)**: นี่คือความรู้ที่บทนี้ต้องพึ่งพามากที่สุด โดยเฉพาะ:
  - **`Rc<T>` (Reference Counted)**: การมี "เจ้าของร่วมกันหลายคน" กับข้อมูลเดียวกันบน heap ด้วยการนับจำนวน owner
    (`strong_count`) — ข้อมูลจะถูก drop ก็ต่อเมื่อ `strong_count` ลดลงถึง 0 เท่านั้น และ `Rc::clone()` ไม่ได้ copy
    ข้อมูลจริง แค่เพิ่มตัวเลขนับ (เป็นการ clone ที่ราคาถูกมาก คนละแบบกับ `.clone()` ของ `String`/`Vec`)
  - **`RefCell<T>` และ interior mutability**: การได้ `&mut T` ออกมาจาก `&T` ที่ดูเหมือน immutable ได้ ด้วยการเลื่อน
    การตรวจสอบกฎ borrowing จาก **compile time ไปเป็น runtime** — `.borrow()` คืน `Ref<T>` (เหมือน `&T`), `.borrow_mut()`
    คืน `RefMut<T>` (เหมือน `&mut T`) และถ้าละเมิดกฎ (เช่น borrow_mut สองครั้งซ้อนกัน) โปรแกรมจะ **panic ตอน runtime**
    ไม่ใช่ compile error
  - **ทำไมต้องรวม `Rc<RefCell<T>>` เข้าด้วยกัน**: `Rc<T>` เดี่ยว ๆ ให้แค่ `&T` เท่านั้น (อ่านได้อย่างเดียว แก้ไขไม่ได้
    เพราะมีเจ้าของร่วมหลายคน) การห่อ `RefCell<T>` ไว้ข้างในทำให้ "แก้ไขข้อมูลที่มีเจ้าของร่วมกันได้" ซึ่งเป็น pattern
    ที่พบบ่อยมากในโครงสร้างข้อมูลแบบ graph/tree
  - **ปัญหาที่ Part 28 ทิ้งไว้ตอนจบ**: เมื่อสอง node ใช้ `Rc<RefCell<T>>` ชี้กลับไปกลับมาหากัน (a ชี้ไป b, b ชี้กลับมา a)
    `strong_count` ของทั้งคู่จะติดอยู่ที่อย่างน้อย 1 ตลอดไปแม้ตัวแปรภายนอกทั้งหมดจะหมด scope แล้ว — เกิด **memory leak**
    ที่ `Rc<T>` ป้องกันไม่ได้ด้วยตัวเอง — บทนี้จะแก้ปัญหานี้เป็นเรื่องแรกด้วย `Weak<T>`
  - **`Cell<T>`**: Part 28 แนะนำสั้น ๆ ว่าเป็น interior mutability สำหรับ type ที่ implement `Copy` เท่านั้น ผ่าน
    `.get()`/`.set()` โดยไม่มี borrow checking เลยแม้แต่ตอน runtime (เพราะไม่มีการคืน reference ออกมา) — บทนี้จะเอามา
    จัดวางในภาพรวมของ "ตระกูล interior mutability" ทั้งหมด
- **Part 11 (`Option<T>` และ Null Safety)**: หัวใจสำคัญของ `Weak::upgrade()` คือมันคืนค่า `Option<Rc<T>>` — ปรัชญา
  เดียวกับที่ Part 11 สอนไว้ทุกประการ: การเข้าถึงข้อมูลที่ "อาจจะไม่มีอยู่แล้ว" ต้องถูกบังคับให้เขียนโค้ดจัดการทั้งสอง
  กรณี (`Some`/`None`) อย่างชัดเจนตั้งแต่ compile time ไม่มีทางลืมเช็คได้เหมือน `null` ในภาษาอื่น
- **Part 6-7 (Ownership และ Borrowing)**: แนวคิดพื้นฐานเรื่อง "ใครเป็นเจ้าของข้อมูล" และ "ใครแค่ยืมสิทธิ์เข้าถึง" ที่
  `Weak<T>` เอาไปประยุกต์ใช้ในระดับที่ลึกกว่าเดิม — `Weak<T>` คือรูปแบบหนึ่งของ "การยืมแบบพิเศษ" ที่ทำงานได้แม้ owner
  จะถูก drop ไปแล้วก็ตาม (ต่างจาก `&T` ปกติที่ borrow checker การันตีว่าข้อมูลต้นทางต้องยังมีชีวิตอยู่เสมอ)
- **Part 9 (Structs)**: ตัวอย่างใหญ่ท้ายบทใช้ struct ที่มี field เป็น `Rc<RefCell<T>>` และ `Weak<RefCell<T>>` ผสมกัน
  สไตล์เดียวกับที่เรียนมาตั้งแต่ Part 9

## เนื้อหา

### 29.1 ทวนปัญหาจาก Part 28: วงจรอ้างอิงที่ทำให้ Memory Leak

ก่อนแก้ปัญหา เราต้อง**เห็นปัญหาให้ชัดด้วยตาตัวเองก่อน** ลองเขียนโครงสร้าง `Node` ที่แต่ละตัวมี field `next` เป็น
`RefCell<Option<Rc<Node>>>` — ออกแบบให้แต่ละ node ชี้ไปยัง node ถัดไปได้ (และแก้ไขได้ด้วย เพราะห่อด้วย `RefCell`)
แล้วสร้างสอง node ที่ชี้กลับไปกลับมาหากันโดยตั้งใจ:

```rust
use std::cell::RefCell;
use std::rc::Rc;

#[derive(Debug)]
struct Node {
    name: String,
    next: RefCell<Option<Rc<Node>>>,
}

impl Drop for Node {
    fn drop(&mut self) {
        // ถ้า destructor นี้ถูกเรียก แสดงว่า memory ของ Node ถูกปล่อยจริง
        println!("กำลัง drop Node: {}", self.name);
    }
}

fn main() {
    let a = Rc::new(Node {
        name: "A".to_string(),
        next: RefCell::new(None),
    });

    let b = Rc::new(Node {
        name: "B".to_string(),
        next: RefCell::new(None),
    });

    // สร้างวงจร: a.next -> b, b.next -> a
    *a.next.borrow_mut() = Some(Rc::clone(&b));
    *b.next.borrow_mut() = Some(Rc::clone(&a));

    println!("a strong_count = {}", Rc::strong_count(&a));
    println!("b strong_count = {}", Rc::strong_count(&b));
} // a, b หมด scope ตรงนี้ แต่ Drop ไม่ถูกเรียกเลย เพราะ strong_count ไม่ถึง 0
```

ผลลัพธ์ที่ได้จากการรันจริง:

```
a strong_count = 2
b strong_count = 2
```

สังเกตให้ดี: **ไม่มีข้อความ "กำลัง drop Node" ปรากฏออกมาเลยแม้แต่บรรทัดเดียว** แม้ว่าโปรแกรมจะออกจากฟังก์ชัน `main` ไป
แล้วก็ตาม (ซึ่งเป็นจุดที่ `a` และ `b` ควรจะหมด scope และถูก drop ตามกฎ ownership ปกติที่เรียนมาตั้งแต่ Part 6) นี่คือ
หลักฐานที่จับได้ตรง ๆ ว่า **เกิด memory leak จริง**

ทำไมถึงเป็นแบบนี้? มาไล่ดูตัวเลข `strong_count` กัน:

- ก่อนสร้างวงจร: `a` มี strong_count = 1 (แค่ตัวแปร `a` เอง), `b` ก็เช่นกัน = 1
- หลัง `*a.next.borrow_mut() = Some(Rc::clone(&b))`: `b` ถูก clone อีกครั้ง (ครั้งนี้ไปอยู่ใน `a.next`) ทำให้
  `b` strong_count = 2
- หลัง `*b.next.borrow_mut() = Some(Rc::clone(&a))`: เช่นกัน `a` strong_count = 2

ตอนนี้แต่ละ node มี **สอง strong reference ชี้มาหามัน**: หนึ่งจากตัวแปรใน `main` (`a`, `b`) และอีกหนึ่งจากอีก node
(`b.next` ชี้มาที่ `a`, `a.next` ชี้มาที่ `b`) เมื่อ `main` จบลง ตัวแปร `a` และ `b` หมด scope จริง แต่นั่น**ลด
strong_count ลงได้แค่ 1 หน่วยต่อตัว** เหลือ strong_count = 1 ทั้งคู่ — ไม่ถึง 0 เพราะยังมีอีก node หนึ่งถืออ้างอิงไว้อยู่
เสมอ! และ node ที่ถืออ้างอิงนั้นก็ไม่มีวันถูก drop เองด้วยเหตุผลเดียวกัน — **ทั้งสองต่างถือชีวิตของอีกฝ่ายไว้ พร้อมกับ
ไม่มีใครมีชีวิตของตัวเองเหลืออยู่ให้ปล่อยได้เลย** นี่คือธรรมชาติของ reference cycle: มันสร้าง "ห่วงโซ่ที่เลี้ยงตัวเอง"
ที่ reference counting เพียงอย่างเดียวไม่มีทางตัดออกได้ ไม่ว่าจะรอนานแค่ไหนก็ตาม — memory ก้อนนี้จะรั่วไหลอยู่แบบนั้น
ไปตลอดอายุของโปรแกรม (จนกว่า process จะจบการทำงานแล้ว OS คืน memory ทั้งหมดกลับไปให้)

นี่คือจุดที่ Part 28 ทิ้งปริศนาไว้ให้เราแก้ในบทนี้ — ปัญหาไม่ได้อยู่ที่ `Rc<T>` "ผิด" หรือ `RefCell<T>` "ผิด" ปัญหาอยู่ที่
**เราใช้ `Rc<T>` (strong reference) ผิดจุด** ในความสัมพันธ์ที่ไม่ควรมีทั้งสองฝั่งเป็น "เจ้าของ" ซึ่งกันและกัน — คำตอบคือ
เครื่องมือใหม่ในบทนี้ที่เรียกว่า `Weak<T>`

#### ภาพในหัวสำหรับวงจรอ้างอิง: ทำไม `strong_count` ไม่มีวันถึงศูนย์

ลองวาดภาพ allocation บน heap ของ `a` และ `b` ออกมาให้เห็นชัด ๆ (คล้ายไดอะแกรมที่ Part 7 ใช้อธิบาย reference):

```
ก่อนจบ main (ทั้ง a, b ยังเป็นตัวแปรที่ยัง scope อยู่):

   ตัวแปร a ────────┐                       ตัวแปร b ────────┐
                     ▼                                       ▼
              ┌─────────────────┐                    ┌─────────────────┐
              │ Node "A"        │                    │ Node "B"        │
              │ strong_count = 2│◀───────────────┐    │ strong_count = 2│◀───┐
              │ next: ──────────┼───────────────▶│    │ next: ──────────┼───┘
              └─────────────────┘                └────└─────────────────┘

หลัง main จบ (ตัวแปร a, b หมด scope แล้ว — เส้นจากตัวแปรถูกตัดออก):

              ┌─────────────────┐                    ┌─────────────────┐
              │ Node "A"        │                    │ Node "B"        │
              │ strong_count = 1│◀───────────────┐    │ strong_count = 1│◀───┐
              │ next: ──────────┼───────────────▶│    │ next: ──────────┼───┘
              └─────────────────┘                └────└─────────────────┘
                    ▲ ยังมี strong reference จาก b.next ชี้มาอยู่ (ไม่ใช่ 0!)
```

สังเกตว่าเส้นลูกศรจาก `a.next` ไปยัง `b` และจาก `b.next` ไปยัง `a` **ไม่ได้ถูกตัดออกไปเลยแม้ตัวแปรภายนอกทั้งคู่จะ
หมด scope แล้วก็ตาม** — เพราะเส้นเหล่านี้คือ strong reference ที่ node หนึ่งถือไว้เพื่อชี้ไปยังอีก node หนึ่ง ไม่ใช่
เส้นที่มาจากตัวแปรใน `main` เลย ทั้งสอง node จึงเหลือ `strong_count = 1` ค้างอยู่ตลอดไป **ไม่ใช่ 0** — และเพราะไม่มี
`strong_count` ตัวใดถึง 0 เลย destructor ของทั้งคู่จึงไม่ถูกเรียกเลยแม้แต่ตัวเดียว นี่คือรากของปัญหาที่มองเห็นได้ชัด
ที่สุดในรูปภาพ: **ไม่มีจุดเริ่มต้นให้เริ่ม "แก้" วงจรได้เลยจากภายในระบบนับ reference เพียงอย่างเดียว** ต้องมีเครื่องมือ
ภายนอกมาตัดวงจรนี้ ซึ่งคือ `Weak<T>` ที่เรากำลังจะเรียนในหัวข้อถัดไป

### 29.2 Strong Reference vs Weak Reference: สองปรัชญาที่ต่างกันโดยสิ้นเชิง

ก่อนดู syntax ของ `Weak<T>` เราต้องเข้าใจ **แนวคิดเบื้องหลัง** ให้แน่นก่อน เพราะนี่คือสิ่งที่จะช่วยให้คุณตัดสินใจได้ถูก
ทุกครั้งว่า "ควรใช้ `Rc<T>` (strong) หรือ `Weak<T>` (weak)" ในความสัมพันธ์แบบใหม่ ๆ ที่คุณจะเจอในอนาคต

ลองเปรียบเทียบสองประโยคนี้:

> **Strong reference**: "ฉันต้องการให้ข้อมูลนี้ยังมีชีวิตอยู่ ตราบใดที่ฉันยังต้องใช้มัน — ถ้าฉันยังอยู่ ข้อมูลนี้ต้อง
> ยังอยู่ด้วยเสมอ"

> **Weak reference**: "ฉันแค่อยากรู้ว่าข้อมูลนี้ยังอยู่ไหมเวลาที่ฉันอยากใช้มัน แต่ฉันไม่ได้ยืนยันว่าข้อมูลนี้ต้องอยู่
> เพราะฉัน — ถ้ามันถูกทำลายไปแล้วโดยคนอื่น (เพราะไม่มีใครต้องการมันอีกแล้ว) ฉันก็ยอมรับความจริงนั้นได้ ไม่ถือเป็นข้อผิดพลาด"

`Rc<T>` (strong reference) ทุกตัวที่ถูก clone ออกมา **มีสิทธิ์ "โหวต" ให้ข้อมูลยังมีชีวิตอยู่** — ตราบใดที่มี strong
reference เหลืออยู่แม้แต่ 1 ตัว ข้อมูลจะไม่ถูก drop เด็ดขาด (`Rc::strong_count()` คือจำนวนโหวตทั้งหมดที่ยังค้างอยู่)
ส่วน `Weak<T>` **ไม่มีสิทธิ์โหวตเลย** — การมี `Weak<T>` อยู่กี่ตัวก็ตาม ไม่มีผลต่อการที่ข้อมูลจะถูก drop หรือไม่แม้แต่
นิดเดียว มันเป็นแค่ "ผู้สังเกตการณ์" ที่คอยถามได้ว่า "ตอนนี้ข้อมูลยังอยู่ไหม" เท่านั้น

กลับไปที่ตัวอย่างวงจรอ้างอิงในหัวข้อ 29.1: ปัญหาที่แท้จริงคือ **ทั้ง `a` และ `b` "โหวต" ให้กันและกันมีชีวิตอยู่พร้อมกัน**
— `a` บอกว่า "ฉันต้องการให้ `b` มีชีวิตอยู่" และ `b` ก็บอกกลับว่า "ฉันต้องการให้ `a` มีชีวิตอยู่" เช่นกัน ทั้งสองฝ่ายจึง
"ล็อก" กันและกันไว้ตลอดไป ไม่มีใครยอมปล่อยก่อน — ถ้าเราเปลี่ยนความสัมพันธ์ฝั่งหนึ่งให้เป็นแค่ "การรับรู้" (weak) แทนที่
จะเป็น "การโหวต" (strong) วงจรล็อกกันนี้จะขาดออกทันที เพราะจะมีฝ่ายหนึ่งที่ไม่ได้ถูกใครโหวตให้มีชีวิตอยู่นานเกินความ
จำเป็นอีกต่อไป

ตัวอย่างที่เข้าใจง่ายที่สุดในโลกจริงคือความสัมพันธ์แบบ **พ่อแม่-ลูก (parent-child)**: พ่อแม่ "เป็นเจ้าของ" ลูก
ในความหมายที่ว่าตราบใดที่พ่อแม่ยังดูแลอยู่ ลูกก็ยังอยู่ในความรับผิดชอบนั้น (ความสัมพันธ์แบบ strong/ownership จาก
พ่อแม่ไปยังลูก) แต่ลูกไม่ได้ "เป็นเจ้าของ" พ่อแม่กลับ — ลูกแค่ "รู้ว่าใครคือพ่อแม่ของตัวเอง" เพื่อใช้ตอบคำถามเวลาจำเป็น
(เช่น "ผู้จัดการของฉันคือใคร") ถ้าพ่อแม่หายไป (ถูก drop เพราะไม่มีใครอื่นถืออ้างอิงไว้แล้ว) ความสัมพันธ์แบบลูก-ไปหา-พ่อแม่
ก็ควรจะ "รู้ตัว" ว่าพ่อแม่ไม่อยู่แล้ว ไม่ใช่ไปยึดชีวิตของพ่อแม่ไว้ไม่ให้ตายตลอดกาลเพียงเพราะลูกยังอ้างอิงถึงอยู่ — นี่คือ
เหตุผลที่ความสัมพันธ์ **child → parent ควรเป็น `Weak<T>` เสมอ** ในขณะที่ **parent → child ควรเป็น `Rc<T>` (strong)**

#### แนวคิด Strong/Weak ไม่ใช่เรื่องแปลกใหม่ — ภาษาอื่นก็เจอปัญหาเดียวกัน

ปัญหา reference cycle ที่เกิดจากการนับ reference (reference counting) ไม่ใช่ปัญหาที่มีแค่ใน Rust — ภาษาใดก็ตามที่ใช้
กลไก reference counting ในการจัดการ memory จะเจอปัญหานี้เหมือนกันหมด และต้องมีแนวคิด "weak reference" แบบเดียวกันนี้
เพื่อแก้ไข:

| ภาษา | กลไก reference counting | คำที่ใช้เรียก weak reference | หมายเหตุ |
|---|---|---|---|
| **Rust** | `Rc<T>`/`Arc<T>` — นับแบบ manual ที่ compiler ช่วยตรวจสอบ | `Weak<T>` (`Rc::downgrade()`/`Weak::upgrade()`) | `upgrade()` คืน `Option<Rc<T>>` ให้ตรวจสอบตอน compile time เสมอ |
| **Swift** | Automatic Reference Counting (ARC) — compiler แทรกโค้ดนับ reference ให้อัตโนมัติ | `weak` (ต้องเป็น `Optional` เสมอ) และ `unowned` (ไม่ใช่ `Optional` แต่ crash ถ้าเข้าถึงหลังข้อมูลตายไปแล้ว) | `weak` ของ Swift ทำงานคล้าย `Weak<T>` ของ Rust มากที่สุด — บังคับเช็ค `nil` ก่อนใช้เสมอ |
| **C++ (ผ่าน `<memory>`)** | `std::shared_ptr<T>` — นับแบบ atomic เพื่อความปลอดภัยข้าม thread | `std::weak_ptr<T>` (ต้องเรียก `.lock()` ซึ่งคืน `shared_ptr` ที่อาจเป็น `nullptr` ได้) | แนวคิดเหมือนกับ Rust เป๊ะ ๆ (`lock()` ก็คือ `upgrade()`) แต่ compiler C++ ไม่บังคับให้เช็ค `nullptr` ก่อนใช้เหมือน Rust บังคับเช็ค `Option` |
| **Python** | Reference counting + **cycle detector แบบ garbage collector เสริม** | ไม่มีแนวคิด weak reference แบบตรง ๆ ในภาษา แต่มี module `weakref` ให้ใช้เอง | Python แก้ปัญหา cycle ด้วยวิธีอื่น: มี garbage collector คอยเดินหา "กลุ่มวัตถุที่ไม่มีใครจากภายนอกอ้างอิงถึงเลย" เป็นระยะ ๆ แล้วเก็บกวาดทั้งกลุ่มพร้อมกัน — เป็นแนวทางที่ต่างจาก Rust/Swift/C++ โดยสิ้นเชิง (runtime cost สูงกว่า แต่ผู้เขียนโค้ดไม่ต้องคิดเรื่อง weak reference เองเลย) |

จุดที่ทำให้ Rust ต่างจาก Swift และ C++ ชัดเจนที่สุดคือ **`Option<Rc<T>>` ที่ `.upgrade()` คืนมาถูกบังคับให้ตรวจสอบตอน
compile time เสมอ** (ตามหลักการ `Option<T>` จาก Part 11) — ใน C++ แม้ `std::weak_ptr::lock()` จะคืน `shared_ptr` ที่
อาจเป็น `nullptr` ได้เหมือนกัน แต่ **ไม่มีอะไรบังคับให้โปรแกรมเมอร์เช็ค `nullptr` ก่อนใช้เลย** ถ้าลืมเช็คแล้วเผลอเรียก
method ผ่าน pointer ที่เป็น `nullptr` จะได้ undefined behavior ทันที — ในขณะที่ Rust compiler จะไม่ยอมให้ code
compile ผ่านเลยถ้าคุณพยายามดึงค่าออกจาก `Option` โดยไม่ผ่านการตรวจสอบก่อน (ตามที่เห็นในกับดักข้อ 1 และ 2 ของบทนี้)

### 29.3 `Rc::downgrade()` และ `Weak::upgrade()`: สร้างและใช้งาน Weak Reference

Rust ให้เราสร้าง `Weak<T>` จาก `Rc<T>` ที่มีอยู่แล้วด้วยฟังก์ชัน `Rc::downgrade()` — ฟังก์ชันนี้**ไม่เพิ่ม**
`strong_count` เลย แต่จะเพิ่ม **`weak_count`** ขึ้นมาแทน (เราจะพูดถึงตัวนับนี้แบบละเอียดในหัวข้อ 29.5) เมื่อได้ `Weak<T>`
มาแล้ว เราไม่สามารถเข้าถึงข้อมูลข้างในได้ตรง ๆ (ไม่มี `Deref` ให้เหมือน `Rc<T>`) เราต้อง **"อัพเกรด" มันกลับเป็น
`Rc<T>` ก่อนเสมอ** ด้วย method `.upgrade()` ซึ่งคืนค่าเป็น `Option<Rc<T>>`:

```rust
use std::rc::{Rc, Weak};

fn main() {
    let strong = Rc::new(String::from("ข้อมูลสำคัญ"));
    let weak: Weak<String> = Rc::downgrade(&strong);

    println!(
        "strong_count = {}, weak_count = {}",
        Rc::strong_count(&strong),
        Rc::weak_count(&strong)
    );

    match weak.upgrade() {
        Some(rc) => println!("upgrade สำเร็จ: {rc}"),
        None => println!("ข้อมูลถูกทำลายไปแล้ว"),
    }

    drop(strong); // strong reference ตัวสุดท้ายถูกปล่อย -> ข้อมูลถูก deallocate

    match weak.upgrade() {
        Some(rc) => println!("upgrade สำเร็จ: {rc}"),
        None => println!("upgrade ล้มเหลว: ข้อมูลถูกทำลายไปแล้ว"),
    }
}
```

ผลลัพธ์:

```
strong_count = 1, weak_count = 1
upgrade สำเร็จ: ข้อมูลสำคัญ
upgrade ล้มเหลว: ข้อมูลถูกทำลายไปแล้ว
```

นี่คือหัวใจสำคัญที่สุดของ `Weak<T>` ทั้งระบบ — ลองไล่ทีละขั้น:

1. **`Rc::downgrade(&strong)`** สร้าง `Weak<String>` ตัวใหม่ โดยไม่เพิ่ม `strong_count` เลย (ยังคงเป็น 1) แต่เพิ่ม
   `weak_count` เป็น 1
2. เรียก `weak.upgrade()` ครั้งแรก **ในขณะที่ `strong` ยังมีชีวิตอยู่** — ข้อมูลยังไม่ถูกทำลาย จึงได้ `Some(rc)` กลับมา
   `rc` ตัวนี้คือ `Rc<String>` ตัวใหม่ที่แยกออกมาจริง ๆ (upgrade สำเร็จหนึ่งครั้ง หมายถึง `strong_count` เพิ่มขึ้น
   ชั่วคราวเป็น 2 ระหว่างที่ `rc` มีชีวิตอยู่ในขอบเขตของ `match` แล้วลดกลับเป็น 1 ทันทีที่ `rc` หมด scope หลัง match
   จบลง)
3. เรียก **`drop(strong)`** ตรง ๆ — นี่คือการบังคับให้ strong reference ตัวสุดท้ายถูกปล่อยทันที (ปกติ `drop()` จะถูก
   เรียกอัตโนมัติตอนหมด scope แต่เราเรียกมันเองตรงนี้เพื่อให้เห็นผลทันทีในโค้ดตัวอย่าง) — เมื่อ `strong_count` ลดลง
   ถึง 0 ข้อมูล `String` ข้างในถูก **deallocate จริง** ทันที
4. เรียก `weak.upgrade()` ครั้งที่สอง **หลังจากข้อมูลถูกทำลายไปแล้ว** — คราวนี้ได้ `None` กลับมา ไม่ใช่ error, ไม่ใช่
   panic, ไม่ใช่ undefined behavior — เป็นแค่ค่าปกติที่บอกตรง ๆ ว่า "ข้อมูลที่แกอยากได้ไม่มีอยู่แล้วนะ"

#### เชื่อมกับปรัชญา `Option<T>` จาก Part 11 โดยตรง

สังเกตว่า signature ของ `.upgrade()` คือ:

```rust
fn upgrade(&self) -> Option<Rc<T>>
```

นี่คือการนำแนวคิด `Option<T>` จาก **Part 11** มาใช้ในบริบทที่ตรงเป้าที่สุดเท่าที่จะเป็นไปได้ — ใน Part 11 เราเรียนว่า
Rust ไม่มี `null` เพราะการเข้าถึงข้อมูลที่ "อาจจะไม่มีอยู่" ต้องถูกบังคับให้ผู้เขียนโค้ดจัดการทั้งสองกรณีอย่างชัดเจน
ตั้งแต่ compile time ไม่มีทางลืมเช็คได้เหมือนภาษาที่มี `null`/`nil`/`None` แบบไม่มีการันตี — `Weak::upgrade()` คือ
สถานการณ์ที่เข้ากับหลักการนี้เป๊ะ ๆ:

- **`Some(rc)`**: ข้อมูลยังมีชีวิตอยู่จริง ณ ขณะที่เรียก `.upgrade()` — คุณได้ `Rc<T>` ตัวใหม่ที่ **เพิ่ม** `strong_count`
  ขึ้นชั่วคราว (เท่ากับ "การโหวต" ให้ข้อมูลนี้มีชีวิตอยู่ต่อไปตราบใดที่ `rc` ตัวนี้ยังไม่ถูก drop)
- **`None`**: ข้อมูลถูกทำลายไปแล้ว (`strong_count` ลงถึง 0 ไปก่อนหน้านี้แล้ว) — คุณ**ต้อง**เขียนโค้ดจัดการกรณีนี้เสมอ
  compiler จะไม่ยอมให้คุณดึง `Rc<T>` ออกมาตรง ๆ โดยไม่ผ่าน pattern matching หรือ combinator (`.map()`, `.unwrap_or()`
  ฯลฯ) เหมือนที่เรียนมาแล้วใน Part 11 ทุกประการ

ข้อดีที่สำคัญมากคือ **compiler บังคับให้คุณคิดถึงกรณี "ข้อมูลอาจไม่อยู่แล้ว" ตั้งแต่ตอนเขียนโค้ด** — คุณไม่มีวันลืมเช็ค
กรณีนี้ไปโดยไม่ตั้งใจ ต่างจากภาษาที่ใช้ raw pointer/reference ที่ dangling pointer อาจถูกใช้งานไปโดยไม่มีการเตือนใด ๆ
เลยจนกว่าโปรแกรมจะ crash (หรือแย่กว่านั้นคือทำงานผิดแบบเงียบ ๆ โดยไม่ crash เลย) เราจะเจาะลึกเรื่องนี้อีกครั้งในหัวข้อ
29.6

#### ทำไม `Weak<T>` ไม่ implement `Deref` เหมือน `Rc<T>`

มือใหม่หลายคนสงสัยว่าทำไม `Weak<T>` ไม่ทำตัวเหมือน `Rc<T>` ที่เรียก method ผ่าน `.` ได้ตรง ๆ (auto-deref ตามที่เรียน
มาจาก Part 7) แต่ต้องเรียก `.upgrade()` ก่อนเสมอ — คำตอบเชิงออกแบบคือ **`Deref` ใน Rust ถูกออกแบบให้เป็น operation
ที่ "ต้องสำเร็จเสมอ ไม่มีทางล้มเหลว"** (ดูจาก signature ของ trait `Deref::deref(&self) -> &Self::Target` ที่ไม่คืน
`Option` หรือ `Result` เลย) — ถ้า `Weak<T>` implement `Deref` ได้ มันจะต้องคืน `&T` ตรง ๆ โดยไม่มีทางบอกผู้เรียกได้ว่า
"ข้อมูลนี้อาจไม่มีอยู่แล้วนะ" ซึ่งขัดกับธรรมชาติของ `Weak<T>` ที่ **ไม่การันตีว่าข้อมูลจะยังมีชีวิตอยู่เลย** โดยตรง —
การบังคับให้ต้องเรียก `.upgrade()` ที่คืน `Option<Rc<T>>` เสียก่อนคือวิธีเดียวที่ทำให้ compiler ยังคงรักษาการันตี
"ไม่มี dangling reference หลุดออกมาได้เลย" ไว้ได้อย่างสมบูรณ์ — ถ้า `Weak<T>` ยอมให้ deref ตรง ๆ ได้ Rust ก็จะสูญเสีย
การันตีข้อนี้ไปทันที และกลายเป็นเหมือน dangling pointer ในภาษาอื่นตามที่อธิบายในหัวข้อ 29.6 นั่นเอง

### 29.4 แก้ปัญหาวงจรจริง: Parent/Child Tree ด้วย `Rc<RefCell<T>>` + `Weak<RefCell<T>>`

ตอนนี้เรามีเครื่องมือครบแล้ว มาแก้ปัญหาจากหัวข้อ 29.1 กันจริง ๆ ด้วยการออกแบบใหม่ตามหลักการจากหัวข้อ 29.2: ทิศทาง
**parent → child** (พ่อแม่ถือลูก) ใช้ `Rc<RefCell<T>>` (strong, เป็นเจ้าของ) ส่วนทิศทาง **child → parent** (ลูกอ้างอิง
กลับไปพ่อแม่) ใช้ `Weak<RefCell<T>>` (weak, แค่รู้จัก ไม่ได้เป็นเจ้าของ)

```rust
use std::cell::RefCell;
use std::rc::{Rc, Weak};

#[derive(Debug)]
struct Node {
    value: i32,
    parent: RefCell<Weak<Node>>,      // child -> parent: weak (ไม่เป็นเจ้าของ)
    children: RefCell<Vec<Rc<Node>>>, // parent -> child: strong (เป็นเจ้าของ)
}

fn main() {
    // สร้าง leaf ก่อน โดยยังไม่มี parent (Weak::new() คือ weak reference ที่ "ว่างเปล่า" ตั้งแต่แรก
    // ไม่ต้องมี Rc<T> ให้อ้างอิงอยู่ก่อนก็สร้างได้ — upgrade() จะได้ None เสมอจนกว่าจะกำหนดค่าใหม่)
    let leaf = Rc::new(Node {
        value: 3,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![]),
    });

    println!(
        "leaf parent = {:?}",
        leaf.parent.borrow().upgrade().map(|n| n.value)
    );
    println!(
        "leaf strong = {}, weak = {}",
        Rc::strong_count(&leaf),
        Rc::weak_count(&leaf)
    );

    // สร้าง branch ที่มี leaf เป็นลูก (strong reference จาก branch ไปหา leaf)
    let branch = Rc::new(Node {
        value: 5,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![Rc::clone(&leaf)]),
    });

    // ผูก leaf.parent ให้ชี้กลับไปที่ branch แบบ weak
    *leaf.parent.borrow_mut() = Rc::downgrade(&branch);

    println!(
        "leaf parent = {:?}",
        leaf.parent.borrow().upgrade().map(|n| n.value)
    );

    println!(
        "branch strong = {}, weak = {}",
        Rc::strong_count(&branch),
        Rc::weak_count(&branch)
    );
    println!(
        "leaf strong = {}, weak = {}",
        Rc::strong_count(&leaf),
        Rc::weak_count(&leaf)
    );
    println!("branch มีลูกทั้งหมด {} node", branch.children.borrow().len());
}
```

ผลลัพธ์:

```
leaf parent = None
leaf strong = 1, weak = 0
leaf parent = Some(5)
branch strong = 1, weak = 1
leaf strong = 2, weak = 0
branch มีลูกทั้งหมด 1 node
```

มาอ่านตัวเลขให้ครบทุกจุด:

- **ก่อนสร้าง `branch`**: `leaf` ยังไม่มี parent (`upgrade()` คืน `None` เพราะ `Weak::new()` เป็น weak reference ที่
  "ไม่ได้ชี้ไปที่ใครเลย" ตั้งแต่ต้น) และ `leaf` มี `strong = 1` (แค่ตัวแปร `leaf` เอง), `weak = 0` (ยังไม่มีใครสร้าง
  `Weak<Node>` ไปหามันเลย)
- **หลังสร้าง `branch`**: `branch.children` ถูกกำหนดเป็น `vec![Rc::clone(&leaf)]` ตอนสร้าง ทำให้ `leaf` strong_count
  เพิ่มเป็น 2 ทันที (`leaf` ตัวแปรเอง + `branch.children[0]`) — แต่ในผลลัพธ์ข้างบนตอนนี้ยังไม่ print ตัวเลขนี้ออกมา
  จนกว่าจะถึงบรรทัดสุดท้าย
- **หลัง `*leaf.parent.borrow_mut() = Rc::downgrade(&branch)`**: สร้าง `Weak<Node>` ตัวใหม่ชี้ไปที่ `branch` แล้วเก็บไว้
  ใน `leaf.parent` — บรรทัดนี้ **เพิ่ม `weak_count` ของ `branch` เป็น 1** แต่ **ไม่แตะ `strong_count` ของ `branch`
  เลย** (ยังเป็น 1 อยู่ — แค่ตัวแปร `branch` เอง) เพราะนี่คือใจความสำคัญที่สุด: **ลูก (`leaf`) ไม่ได้ "โหวต" ให้พ่อแม่
  (`branch`) มีชีวิตอยู่เลย**
- ผลลัพธ์บรรทัดสุดท้ายแสดงว่า `leaf.parent.upgrade()` ทำงานได้จริง — เพราะ `branch` **ยังมีชีวิตอยู่จริง**
  (`strong_count = 1` มากกว่า 0) ทำให้ upgrade สำเร็จได้ `Some(5)` (`value` ของ branch)

**จุดที่สำคัญที่สุดของการออกแบบนี้**: ถ้าเราลอง drop `branch` ตัวแปรทิ้งไป (สมมติเขียน `drop(branch);` ก่อน `main` จบ)
`strong_count` ของ `branch` จะลดลงถึง 0 ทันที (เพราะไม่มี strong reference อื่นเหลืออยู่เลย — `leaf.parent` เป็นแค่
weak) ทำให้ `branch` ถูก deallocate จริง แล้วถ้า `leaf.parent.borrow().upgrade()` ถูกเรียกอีกครั้งหลังจากนั้น จะได้
`None` กลับมาอย่างปลอดภัย **ไม่มี memory leak เกิดขึ้นเลย** ต่างจากตัวอย่างในหัวข้อ 29.1 อย่างสิ้นเชิง — นี่คือรูปแบบ
มาตรฐานที่ใช้กันจริงในโค้ด Rust สำหรับโครงสร้างข้อมูลแบบ tree ที่ต้องการ back-reference กลับไปยัง parent (เช่น DOM
tree ใน browser engine, AST ใน compiler, GUI widget tree เป็นต้น)

#### กรณีใช้งานจริงอีกแบบของ `Weak<T>`: Observer / Event Bus Pattern

parent-child tree ไม่ใช่สถานการณ์เดียวที่ต้องใช้ `Weak<T>` — อีกรูปแบบที่พบบ่อยมากในโปรแกรมจริงคือ **Observer
pattern** (หรือ event bus / pub-sub): มี "ผู้ส่ง event" (publisher) ที่ต้องเก็บรายชื่อ "ผู้รับ event" (listener/
observer) ไว้เพื่อแจ้งเตือนเมื่อมีเหตุการณ์เกิดขึ้น แต่ **publisher ไม่ควรเป็นเจ้าของ listener เหล่านั้นเลย** —
listener ควรมีอายุการใช้งานที่กำหนดโดยส่วนอื่นของโปรแกรม (เช่น UI component ที่ถูกสร้าง/ทำลายไปตามการใช้งานจริงของ
ผู้ใช้) ถ้า publisher ถือ `Rc<Listener>` (strong) ไว้ตลอดไป listener นั้นจะไม่มีวันถูก drop เลยแม้ผู้ใช้จะปิด UI
component นั้นไปแล้วก็ตาม — เป็นการรั่วไหลของ memory ในอีกรูปแบบหนึ่งที่ไม่ใช่ cycle ตรง ๆ แต่มีสาเหตุเดียวกันคือ
"ใช้ strong reference ในความสัมพันธ์ที่ไม่ควรเป็นเจ้าของ"

```rust
use std::cell::RefCell;
use std::rc::{Rc, Weak};

struct Logger {
    label: String,
    log: RefCell<Vec<String>>,
}

impl Logger {
    fn new(label: &str) -> Rc<Logger> {
        Rc::new(Logger {
            label: label.to_string(),
            log: RefCell::new(Vec::new()),
        })
    }

    fn on_event(&self, message: &str) {
        self.log
            .borrow_mut()
            .push(format!("[{}] {}", self.label, message));
    }
}

struct EventBus {
    listeners: RefCell<Vec<Weak<Logger>>>,
}

impl EventBus {
    fn new() -> Self {
        EventBus {
            listeners: RefCell::new(Vec::new()),
        }
    }

    // สมัครรับ event แบบ weak — EventBus จะไม่ยึดชีวิต Logger ไว้เลย
    fn subscribe(&self, logger: &Rc<Logger>) {
        self.listeners.borrow_mut().push(Rc::downgrade(logger));
    }

    // ส่ง event ไปยังทุก listener ที่ยังมีชีวิตอยู่ พร้อมเก็บกวาดตัวที่ตายไปแล้วออกไปด้วย
    fn publish(&self, message: &str) {
        self.listeners.borrow_mut().retain(|weak_logger| {
            match weak_logger.upgrade() {
                Some(logger) => {
                    logger.on_event(message);
                    true // ยังมีชีวิตอยู่ เก็บไว้ใน list ต่อไป
                }
                None => false, // ตายไปแล้ว ตัดออกจาก list เลย
            }
        });
    }
}

fn main() {
    let bus = EventBus::new();

    let logger_a = Logger::new("A");
    bus.subscribe(&logger_a);

    {
        let logger_b = Logger::new("B");
        bus.subscribe(&logger_b);

        bus.publish("เริ่มระบบ");
        println!("จำนวน listener ตอนนี้: {}", bus.listeners.borrow().len());
    } // logger_b หมด scope ที่นี่ ถูก drop จริง เพราะ EventBus ไม่ได้ถือ strong reference ไว้เลย

    bus.publish("หลัง logger_b ตายไปแล้ว");
    println!("จำนวน listener หลังเก็บกวาด: {}", bus.listeners.borrow().len());

    println!("{:?}", logger_a.log.borrow());
}
```

ผลลัพธ์:

```
จำนวน listener ตอนนี้: 2
จำนวน listener หลังเก็บกวาด: 1
["[A] เริ่มระบบ", "[A] หลัง logger_b ตายไปแล้ว"]
```

ประเด็นสำคัญที่สุดของตัวอย่างนี้อยู่ที่ `EventBus::subscribe()` — มันใช้ `Rc::downgrade(logger)` ไม่ใช่
`Rc::clone(logger)` ทำให้ `EventBus` **ไม่มีสิทธิ์โหวตให้ `Logger` มีชีวิตอยู่เลย** เมื่อ `logger_b` หมด scope
(ในตัวอย่างคือปิด block `{ }` ที่ครอบมันไว้) มันถูก drop จริงทันที ไม่ต้องรอให้ `EventBus` ปล่อยก่อนแบบที่จะเกิดขึ้น
ถ้าใช้ `Rc::clone` ผิดจุด — ฟังก์ชัน `publish()` ยังฉลาดพอที่จะ **ตรวจและเก็บกวาด weak reference ที่ตายไปแล้วออกจาก
`listeners` ไปในตัวเดียวกัน** ด้วย `.retain()` ที่คืน `false` ให้กับ listener ที่ `upgrade()` ไม่สำเร็จ — ทำให้
`Vec<Weak<Logger>>` ไม่โตขึ้นเรื่อย ๆ แบบไม่มีที่สิ้นสุดด้วย (ถ้าไม่เก็บกวาด แม้จะไม่มี memory leak ของตัว `Logger`
เอง แต่ก็ยังเปลืองที่ของ `Weak<T>` ที่ไม่มีประโยชน์แล้วสะสมอยู่ใน `Vec` ไปเรื่อย ๆ) — นี่คือรูปแบบที่ใช้กันจริงใน
GUI framework และระบบ event-driven จำนวนมากที่เขียนด้วย Rust

### 29.5 `strong_count()` vs `weak_count()`: ใครควบคุมชีวิต-ความตายของข้อมูลจริง ๆ

ตอนนี้เราเห็นทั้งสองตัวนับทำงานแล้ว มาสรุปความหมายที่แม่นยำของแต่ละตัว:

| ตัวนับ | นับอะไร | มีผลต่อการ deallocate ข้อมูลหรือไม่ |
|---|---|---|
| `Rc::strong_count(&rc)` | จำนวน `Rc<T>` (strong reference) ทั้งหมดที่ชี้ไปยังข้อมูลชุดนี้ ณ ขณะนั้น | **มีผลโดยตรง** — ข้อมูล `T` (ค่าจริงที่คุณเก็บไว้) จะถูก drop ทันทีที่ตัวเลขนี้ลดลงถึง 0 |
| `Rc::weak_count(&rc)` | จำนวน `Weak<T>` ทั้งหมดที่ยังไม่ถูก drop ที่ชี้ไปยังข้อมูลชุดนี้ ณ ขณะนั้น | **ไม่มีผลต่อการ drop ค่า `T` เลย** — ไม่ว่าจะมี `Weak<T>` เหลืออยู่กี่ตัวก็ตาม |

กฎที่ต้องจำให้แม่นคือ: **ค่า `T` ที่แท้จริงจะถูก drop (destructor ถูกเรียก) ทันทีที่ `strong_count` ลดลงถึง 0 — โดย
ไม่สนใจ `weak_count` เลยแม้แต่นิดเดียว** แม้ตอนนั้นจะยังมี `Weak<T>` เหลืออยู่ 100 ตัวก็ตาม ค่าข้างในก็ถูกทำลายไปตามปกติ

คำถามที่ตามมาคือ "แล้วถ้ามี `Weak<T>` เหลืออยู่หลังจากค่าถูก drop ไปแล้ว จะเกิดอะไรขึ้นตอนที่ใช้ `Weak<T>` นั้นต่อ?"
นี่คือรายละเอียดเชิงลึกที่ทำให้ `Weak<T>` ปลอดภัยกว่า raw pointer ทุกประการ: internally `Rc<T>` และ `Weak<T>` ไม่ได้
ชี้ไปที่ค่า `T` ตรง ๆ แต่ชี้ไปที่ **allocation กลาง (มักเรียกว่า `RcBox`/control block)** ที่เก็บทั้ง `strong_count`,
`weak_count`, และค่า `T` ไว้ด้วยกันเป็นก้อนเดียว เมื่อ `strong_count` ลงถึง 0:

1. ค่า `T` ข้างในถูก **drop ทันที** (destructor ถูกเรียก, memory ของค่า `T` เองถูกคืน — เช่นถ้า `T` คือ `String` ก็คือ
   heap buffer ของ `String` นั้นถูกปล่อย)
2. แต่ **ตัว allocation กลางเอง** (ที่เก็บตัวเลขนับทั้งสองตัว) **ยังไม่ถูกปล่อยทันที** ถ้ายังมี `weak_count > 0` อยู่ —
   มันจะถูกเก็บไว้เฉย ๆ ในสถานะ "ค่า `T` ตายไปแล้ว แต่กล่องยังอยู่" จนกว่า `weak_count` จะลดลงถึง 0 ด้วย (คือ
   `Weak<T>` ทุกตัวถูก drop ไปหมดแล้ว) ตัว allocation กลางถึงจะถูกปล่อยคืนให้ระบบจริง ๆ

การออกแบบแบบนี้ **ฉลาดมาก** เพราะทำให้ `Weak::upgrade()` ตรวจสอบ `strong_count` ได้อย่างปลอดภัยเสมอ ไม่ว่าจะเรียกกี่
ครั้งก็ตาม โดยไม่ต้องกังวลว่า memory ตรงนั้นจะถูกคนอื่นเอาไปใช้ซ้ำแล้ว (เพราะตัว allocation กลางยังถูกจองไว้อยู่จนกว่า
`Weak<T>` ทุกตัวจะหมดไปด้วย) — นี่คือกลไกที่ทำให้ `upgrade()` **ไม่มีวันเป็น undefined behavior** ไม่ว่าจะเรียกตอนไหน
ก็ตาม จะได้ `Some` (ถ้าค่ายังอยู่) หรือ `None` (ถ้าค่าตายไปแล้ว) เท่านั้น ไม่มีทางที่สามเกิดขึ้นได้เลย

### 29.6 เปรียบเทียบกับ Dangling Pointer ใน C/C++: ทำไม Rust ปลอดภัยกว่า

เพื่อให้เห็นความแตกต่างชัดที่สุด ลองนึกภาพโค้ด C++ ที่ทำสิ่งคล้าย ๆ กันด้วย raw pointer:

```cpp
// ตัวอย่าง C++ (ไม่ใช่ Rust — แสดงเพื่อเปรียบเทียบเท่านั้น)
struct Node {
    int value;
    Node* parent; // raw pointer ธรรมดา ไม่มีการนับ reference ใด ๆ เลย
};

Node* make_child(Node* parent_ptr) {
    Node* child = new Node{1, parent_ptr};
    return child;
}

int main() {
    Node* parent = new Node{5, nullptr};
    Node* child = make_child(parent);

    delete parent;      // ปล่อย memory ของ parent ไปแล้ว
    parent = nullptr;   // ตัวแปร parent เป็น nullptr แล้ว แต่...

    std::cout << child->parent->value; // 💥 child->parent ยังชี้ไปที่ memory เดิมที่ถูก delete ไปแล้ว!
    // นี่คือ undefined behavior — อาจ crash, อาจอ่านค่าขยะโดยไม่ crash, อาจทำงานถูกบางครั้งบางที
    // ไม่มีทางรู้ล่วงหน้าได้เลยว่าจะเกิดอะไรขึ้น และ compiler ไม่เตือนอะไรเลยทั้งตอน compile และตอน run
}
```

ปัญหาของโค้ดนี้คือ `child->parent` เป็น **dangling pointer** — มันยังเก็บ**ที่อยู่ (address)** ของ memory ที่ถูก
`delete` ไปแล้วไว้ แต่ไม่มีกลไกใดเลยที่จะรู้ตัวว่า memory ตรงนั้น "ตายไปแล้ว" การอ่านค่าผ่าน pointer นี้คือ
**undefined behavior** ตามนิยามของภาษา C++ ซึ่งหมายความว่าคอมไพเลอร์และ runtime **ไม่มีข้อผูกมัดใด ๆ** ว่าโปรแกรม
จะทำอะไร — อาจดูเหมือนทำงานถูกในบางครั้ง (ถ้า memory ตรงนั้นบังเอิญยังไม่ถูกเขียนทับ) แต่จะพังแบบสุ่มในบางครั้ง
(ถ้า memory ตรงนั้นถูก allocator เอาไปใช้ซ้ำให้ข้อมูลอื่นแล้ว) — บั๊กประเภทนี้เป็นสาเหตุของช่องโหว่ความปลอดภัยจำนวนมาก
ในโลกจริง (use-after-free เป็นหนึ่งในประเภทช่องโหว่ที่ถูกใช้โจมตี software ที่เขียนด้วย C/C++ บ่อยที่สุด)

เทียบกับ `Weak<T>` ใน Rust: `Weak<T>` **ไม่ได้เก็บ address ของข้อมูลแบบดิบ ๆ** แต่เก็บ pointer ไปยัง allocation กลาง
ที่มีตัวนับ `strong_count`/`weak_count` อยู่เสมอ (ตามที่อธิบายในหัวข้อ 29.5) — เมื่อคุณเรียก `.upgrade()` มันจะ **ตรวจ
`strong_count` ก่อนเสมอ**:

- ถ้า `strong_count > 0`: ข้อมูลยังมีชีวิตอยู่จริง ปลอดภัยที่จะคืน `Rc<T>` ใหม่กลับมา (`Some`)
- ถ้า `strong_count == 0`: ข้อมูลตายไปแล้ว **การันตีว่าจะไม่คืนอะไรที่เป็นอันตราย** — คืน `None` แทน ไม่มีการอ่านค่า
  `T` ที่ตายไปแล้วเลยแม้แต่ byte เดียว

ผลลัพธ์คือ **ไม่มีทางเกิด use-after-free ผ่าน `Weak<T>` ได้เลยไม่ว่ากรณีใด** — สิ่งที่แย่ที่สุดที่เกิดขึ้นได้คือคุณได้
`None` กลับมาแล้วต้องเขียนโค้ดจัดการกรณีนั้น (ซึ่ง compiler บังคับให้คุณทำอยู่แล้วผ่าน `Option<T>` ตามที่อธิบายใน
หัวข้อ 29.3) — จาก **undefined behavior ที่อาจนำไปสู่ security vulnerability ในภาษาอื่น** กลายเป็นแค่ **ค่า `None`
ที่ตรวจสอบได้อย่างปลอดภัย 100% ตอน compile time ว่าคุณจัดการมันแล้ว** นี่คือตัวอย่างที่ชัดเจนที่สุดอย่างหนึ่งว่าทำไม
Rust ถึงทำได้ทั้ง "ควบคุม memory เองแบบละเอียด (manual memory management)" และ "ปลอดภัยเท่าภาษาที่มี garbage
collector" ไปพร้อมกัน — โดยไม่ต้องมี garbage collector คอยเดินตรวจ memory ตอน runtime แม้แต่นิดเดียว

### 29.7 `Cow<T>`: Clone on Write คืออะไร และทำไมต้องมี

เปลี่ยนเรื่องไปดูเครื่องมืออีกตัวที่ไม่เกี่ยวกับปัญหา reference cycle เลย แต่ยังอยู่ในตระกูล "smart pointer ที่ช่วย
จัดการเรื่อง ownership ให้ฉลาดขึ้น" — `Cow<T>` (ย่อจาก **Clone on Write**) อยู่ใน module `std::borrow`

มาดูปัญหาที่ `Cow<T>` แก้ก่อน: สมมติคุณต้องเขียนฟังก์ชันที่รับ `&str` เข้ามา **ส่วนใหญ่คืนค่ากลับไปแบบไม่มีการแก้ไข
อะไรเลย** แต่ **บางครั้ง** ต้องแก้ไขข้อมูลนั้นก่อนคืนกลับ (เช่นฟังก์ชัน sanitize ข้อความ, ฟังก์ชัน escape ตัวอักษร
พิเศษ, ฟังก์ชันตัด whitespace ที่เกินความจำเป็น) — คุณมีตัวเลือกอยู่สองแบบ **ที่ไม่ดีทั้งคู่** ถ้าไม่รู้จัก `Cow<T>`:

1. **คืน `String` (owned) เสมอ**: ใช้งานง่าย compile ผ่านชัวร์ แต่ **ต้อง allocate + copy ข้อมูลใหม่ทุกครั้ง** แม้ใน
   กรณีส่วนใหญ่ (ที่ไม่มีอะไรต้องแก้ไขเลย) ก็ตาม — เสีย performance ไปโดยไม่จำเป็นเลย ยิ่งถ้าฟังก์ชันนี้ถูกเรียก
   บ่อยมาก ๆ (เช่นใน hot path ของโปรแกรมประมวลผลข้อความจำนวนมาก) ความสูญเปล่านี้จะสะสมเป็นก้อนใหญ่มาก
2. **คืน `&str` (borrowed) เสมอ**: เร็วในกรณีที่ไม่ต้องแก้ไข แต่ **เป็นไปไม่ได้เลย** ในกรณีที่ต้องแก้ไขข้อมูลจริง —
   ถ้าคุณ trim หรือ replace ตัวอักษรจนได้ข้อมูลใหม่ที่ต่างจาก input เดิม คุณไม่มี `&str` ที่ชี้ไปยังข้อมูลใหม่นั้นให้
   คืนกลับได้เลย (มันเป็นข้อมูลที่สร้างขึ้นมาใหม่ ไม่มี owner เดิมให้ borrow มาจาก)

`Cow<T>` คือคำตอบที่แก้ปัญหานี้ได้อย่างสมบูรณ์แบบ — มันคือ `enum` ที่มีสองรูปแบบ:

```rust
// นิยามจริงใน std::borrow (ย่อให้เข้าใจง่าย)
enum Cow<'a, B: ?Sized + ToOwned + 'a> {
    Borrowed(&'a B),
    Owned(<B as ToOwned>::Owned),
}
```

สำหรับ `Cow<'a, str>` (ที่พบบ่อยที่สุด) จะมีสองรูปแบบที่เข้าใจง่ายกว่านี้:

- **`Cow::Borrowed(&'a str)`**: เก็บแค่ reference ไปยังข้อมูลต้นทาง — **ไม่มีการ allocate ใหม่เลย** ใช้ได้ตอนที่ไม่
  ต้องแก้ไขข้อมูล
- **`Cow::Owned(String)`**: เก็บ `String` ที่เป็นเจ้าของข้อมูลจริง — allocate ใหม่ ใช้ตอนที่ต้องแก้ไข/สร้างข้อมูลใหม่

ชื่อ "Clone on Write" สื่อถึงหลักการสำคัญ: **ถ้าไม่ต้องเขียน (write/modify) ก็ไม่ต้อง clone เลย** — clone (การ
allocate ข้อมูลใหม่) จะเกิดขึ้น**ก็ต่อเมื่อจำเป็นต้องแก้ไขข้อมูลจริง ๆ เท่านั้น** ซึ่งตรงกับสถานการณ์ที่เรากำลังพูดถึง
เป๊ะ ๆ: "ส่วนใหญ่ไม่ต้องแก้ไข บางครั้งต้องแก้ไข"

### 29.8 ตัวอย่างจริง: ฟังก์ชัน Sanitize ข้อความด้วย `Cow<str>`

มาเขียนฟังก์ชันจริงที่ตัด whitespace หัว-ท้ายและแทน tab ด้วย space — ถ้าข้อความ "สะอาด" อยู่แล้ว (ไม่มี whitespace
เกิน ไม่มี tab) จะคืนค่าแบบ borrowed ทันทีโดยไม่ allocate อะไรเลย แต่ถ้าต้องแก้ไขจริง จะคืนแบบ owned:

```rust
use std::borrow::Cow;

fn sanitize(input: &str) -> Cow<'_, str> {
    let trimmed = input.trim();

    if trimmed.contains('\t') {
        // มี tab ที่ต้องแทนด้วย space จริง ๆ -> ต้อง allocate String ใหม่
        Cow::Owned(trimmed.replace('\t', " "))
    } else if trimmed.len() != input.len() {
        // มี whitespace หัว/ท้ายที่ต้องตัดออก -> ต้อง allocate เช่นกัน
        Cow::Owned(trimmed.to_string())
    } else {
        // ไม่มีอะไรต้องแก้เลย -> คืน borrowed ตรง ๆ ไม่ allocate เลยแม้แต่ byte เดียว
        Cow::Borrowed(input)
    }
}

fn main() {
    let clean = "hello";
    let dirty = "  hello\tworld  ";

    match sanitize(clean) {
        Cow::Borrowed(s) => println!("ไม่มีการ clone เลย: {s:?}"),
        Cow::Owned(s) => println!("ต้อง clone: {s:?}"),
    }

    match sanitize(dirty) {
        Cow::Borrowed(s) => println!("ไม่มีการ clone เลย: {s:?}"),
        Cow::Owned(s) => println!("ต้อง clone: {s:?}"),
    }
}
```

ผลลัพธ์:

```
ไม่มีการ clone เลย: "hello"
ต้อง clone: "hello world"
```

สังเกตว่า `sanitize(clean)` คืนค่า `Cow::Borrowed(input)` กลับมา **ตรง ๆ โดยไม่มีการ allocate memory ก้อนใหม่บน heap
เลยแม้แต่ byte เดียว** — เป็นแค่การส่ง reference ไปยัง string literal `"hello"` เดิมกลับออกไปในรูปแบบที่ signature
ของฟังก์ชันยอมรับ (`Cow<str>` แทน `&str` ตรง ๆ) ในขณะที่ `sanitize(dirty)` ต้องสร้าง `String` ใหม่จริง ๆ เพราะข้อมูล
เปลี่ยนไปจากต้นฉบับ (ตัด whitespace และแทน tab)

**จุดที่สำคัญที่สุดในเชิง performance**: ถ้าฟังก์ชันนี้ถูกเรียกนับล้านครั้งในโปรแกรมประมวลผลข้อความขนาดใหญ่ (เช่น
log parser, text pipeline, ระบบ validate input จากผู้ใช้) และในทางปฏิบัติจริง **ข้อความส่วนใหญ่มักจะ "สะอาด" อยู่แล้ว**
(ไม่มี whitespace เกิน ไม่มี tab) การใช้ `Cow<str>` จะทำให้ **ส่วนใหญ่ของการเรียกฟังก์ชันนี้ไม่มีการ allocate memory
เลย** ต่างจากการออกแบบที่คืน `String` เสมอซึ่งจะ allocate ทุกครั้งไม่ว่าจำเป็นหรือไม่ — นี่คือการประหยัด CPU cycle
และ heap allocation จำนวนมากในโปรแกรมจริงที่ทำงานหนักด้าน throughput

### 29.9 `.to_mut()` และ `.into_owned()`: แปลง `Cow<T>` ไปมาอย่างถูกวิธี

`Cow<T>` มี method สำคัญสองตัวที่ควรรู้จักให้แม่น:

- **`.to_mut(&mut self) -> &mut B`**: ถ้า `Cow` เป็น `Owned` อยู่แล้ว คืน `&mut` ไปยังข้อมูลนั้นตรง ๆ แต่ถ้าเป็น
  `Borrowed` อยู่ **จะ clone ข้อมูลนั้นให้กลายเป็น `Owned` ก่อน (นี่คือจังหวะที่ "clone on write" เกิดขึ้นจริง)** แล้ว
  คืน `&mut` ไปยัง `Owned` ตัวใหม่นั้น — เหมาะกับสถานการณ์ที่คุณต้องการแก้ไขข้อมูลใน `Cow` แบบ "แก้ไขก็ต่อเมื่อจำเป็น"
- **`.into_owned(self) -> <B as ToOwned>::Owned`**: แปลง `Cow<T>` เป็นค่า owned เสมอไม่ว่าจะเป็น `Borrowed` หรือ
  `Owned` อยู่ก่อน (ถ้าเป็น `Borrowed` จะ clone ให้ ณ จุดนี้เลย ถ้าเป็น `Owned` อยู่แล้วก็คืนค่านั้นตรง ๆ โดยไม่ clone
  ซ้ำ) — เหมาะกับสถานการณ์ที่คุณต้องการ "ปิดประเด็นเรื่อง borrow/owned" แล้วอยากได้ `String`/`Vec<T>` ที่เป็นเจ้าของ
  แน่นอนไปใช้งานต่อ

```rust
use std::borrow::Cow;

fn ensure_exclaim(input: &str) -> Cow<'_, str> {
    let mut cow: Cow<str> = Cow::Borrowed(input);
    if !input.ends_with('!') {
        cow.to_mut().push('!'); // ถ้ายังเป็น Borrowed จะ clone เป็น Owned ให้ก่อน แล้วค่อย push
    }
    cow
}

fn main() {
    let a = ensure_exclaim("hello");  // ไม่มี ! -> ต้อง clone แล้วเติม !
    let b = ensure_exclaim("hello!"); // มี ! อยู่แล้ว -> ไม่ clone เลย คืน Borrowed ตรง ๆ
    println!("{a}");
    println!("{b}");

    let owned_string: String = a.into_owned(); // แปลงเป็น String แน่นอน (a เป็น Owned อยู่แล้ว ไม่ clone ซ้ำ)
    println!("{owned_string}");
}
```

ผลลัพธ์:

```
hello!
hello!
hello!
```

สังเกตว่าทั้ง `a` และ `b` แสดงผลเหมือนกัน (`"hello!"`) แต่ **เบื้องหลังต่างกันโดยสิ้นเชิง**: `b` ไม่มีการ allocate
memory ใหม่เลยตลอดทั้งฟังก์ชัน (ยังเป็น `Cow::Borrowed("hello!")` ตรง ๆ) ในขณะที่ `a` ต้อง clone ข้อมูลตอนเรียก
`.to_mut()` ก่อนจะ `push('!')` ได้ (เพราะ `Cow::Borrowed` ไม่มี `&mut` ให้แก้ไขตรง ๆ ได้ — ข้อมูลต้นทางอาจถูกใช้ที่อื่น
อยู่ก็ได้ การแก้ไขมันตรง ๆ ผ่าน reference จะขัดกับกฎ borrowing ที่เรียนมาตั้งแต่ Part 7)

นี่คือสิ่งที่ทำให้ `Cow<T>` "ฉลาด" กว่าการคืน `String` ตรง ๆ เสมอ — โค้ดที่เรียกใช้ `ensure_exclaim` **ไม่ต้องรู้เลย
ว่าเบื้องหลังมีการ allocate หรือไม่** (interface เดียวกันทั้งสองกรณี คือ `Cow<str>` ที่ deref เป็น `&str` ได้เสมอ
ผ่าน trait `Deref`) แต่ตัวฟังก์ชันเองก็ยัง**ฉลาดพอที่จะเลี่ยง allocate เมื่อไม่จำเป็น**ได้อย่างสมบูรณ์แบบ นี่คือแก่น
ของแนวคิด "Clone on Write" — clone จะเกิดขึ้นก็ต่อเมื่อมีการ "write" (แก้ไข) เกิดขึ้นจริงเท่านั้น ไม่ใช่ก่อนหน้านั้น

#### `Cow<T>` ในโลกจริง: ตัวอย่างจาก Standard Library

`Cow<T>` ไม่ใช่เครื่องมือที่ใช้แค่ในตัวอย่างสอน — standard library ของ Rust เองก็ใช้ `Cow<T>` ในหลายจุดที่ตรงกับ
สถานการณ์ "ส่วนใหญ่ไม่ต้องแก้ไข บางครั้งต้องแก้ไข" เป๊ะ ๆ ที่เราเพิ่งเรียนมา สองตัวอย่างที่พบบ่อยที่สุดคือ
`Path::to_string_lossy()` และ `String::from_utf8_lossy()`:

```rust
use std::borrow::Cow;
use std::path::Path;

fn main() {
    let path = Path::new("/home/user/report.txt");
    let display: Cow<str> = path.to_string_lossy();
    match &display {
        Cow::Borrowed(_) => println!("to_string_lossy: borrowed (path เป็น valid UTF-8 อยู่แล้ว)"),
        Cow::Owned(_) => println!("to_string_lossy: owned (ต้องแปลง byte ที่ไม่ valid UTF-8)"),
    }
    println!("{display}");

    let valid_bytes = b"hello";
    let s1 = String::from_utf8_lossy(valid_bytes);
    match &s1 {
        Cow::Borrowed(_) => println!("from_utf8_lossy (valid): borrowed"),
        Cow::Owned(_) => println!("from_utf8_lossy (valid): owned"),
    }

    let invalid_bytes = vec![0x68, 0x69, 0xFF, 0xFE]; // มี byte ที่ไม่ valid UTF-8 ปนอยู่
    let s2 = String::from_utf8_lossy(&invalid_bytes);
    match &s2 {
        Cow::Borrowed(_) => println!("from_utf8_lossy (invalid): borrowed"),
        Cow::Owned(_) => println!("from_utf8_lossy (invalid): owned"),
    }
    println!("{s2}");
}
```

ผลลัพธ์:

```
to_string_lossy: borrowed (path เป็น valid UTF-8 อยู่แล้ว)
/home/user/report.txt
from_utf8_lossy (valid): borrowed
from_utf8_lossy (invalid): owned
hi��
```

(สองตัวอักษรสุดท้ายของบรรทัดสุดท้ายคือ "replacement character" (U+FFFD, `�`) ที่ `from_utf8_lossy` ใส่แทนที่ byte
`0xFF` และ `0xFE` ที่ไม่ valid UTF-8 แต่ละตัวโดยอัตโนมัติ — นี่คือเหตุผลที่ชื่อฟังก์ชันมีคำว่า "lossy": ข้อมูลต้นฉบับ
บางส่วนสูญหายไปจริง ถูกแทนด้วยตัวอักษรพิเศษนี้แทน)

เหตุผลที่ทั้งสองฟังก์ชันนี้เลือกคืน `Cow<str>` ไม่ใช่ `String` ตรง ๆ คือ: **กรณีส่วนใหญ่ในโลกจริง path หรือ byte
sequence ที่ได้มามักเป็น valid UTF-8 อยู่แล้ว** (โดยเฉพาะบนระบบปฏิบัติการที่ใช้ UTF-8 เป็นมาตรฐาน) การบังคับให้
allocate `String` ใหม่ทุกครั้งที่เรียกฟังก์ชันเหล่านี้ (ซึ่งมักถูกเรียกบ่อยมากในโค้ดที่ทำงานกับไฟล์/path จำนวนมาก)
จะเป็นการสิ้นเปลืองที่ไม่จำเป็นในกรณีส่วนใหญ่ — `Cow<str>` ทำให้ทั้งสองฟังก์ชันคืนค่าแบบ borrowed ได้ทันทีในกรณีปกติ
และคืนแบบ owned เฉพาะกรณีพิเศษ (มี byte แปลกปลอมที่ต้อง "lossy" แปลงเป็น `�` แทน) เท่านั้น — ตรงกับปรัชญาของ
`Cow<T>` ที่เราเรียนมาทั้งหมดในหัวข้อ 29.7-29.9 อย่างสมบูรณ์

### 29.10 ตารางเปรียบเทียบ Smart Pointer ทั้งหมดในโมดูลนี้

ตอนนี้เราเรียนจบ smart pointer หลัก ๆ ที่ใช้บ่อยที่สุดใน Rust ครบทุกตัวแล้ว (ยกเว้น `Arc<T>`/`Mutex<T>` ที่เป็น
เวอร์ชัน thread-safe ซึ่งจะเรียนเต็ม ๆ ใน **Part 39**) มาสรุปเป็นตารางเปรียบเทียบเพื่อให้เห็นภาพรวมทั้งหมดในที่เดียว:

| Smart Pointer | เจ้าของข้อมูล | ตำแหน่งข้อมูล | เข้าถึงแบบไหน | Thread-safe? | ใช้เมื่อไหร่ |
|---|---|---|---|---|---|
| `Box<T>` (Part 27) | เจ้าของเดียว (single owner) | heap | `&T` / `&mut T` ตามปกติ (ผ่าน `Deref`/`DerefMut`) | ได้ (ถ้า `T: Send`) แต่ไม่มีการแชร์เจ้าของ | ต้องการเก็บข้อมูลบน heap แบบมีเจ้าของเดียวชัดเจน (เช่น recursive type, trait object, ข้อมูลขนาดใหญ่ที่ไม่ต้องการ copy บ่อย ๆ) |
| `Rc<T>` (Part 28) | เจ้าของร่วมกันหลายคน (shared ownership) นับด้วย `strong_count` | heap | `&T` เท่านั้น (อ่านอย่างเดียว) | **ไม่** ปลอดภัยข้าม thread (ตัวนับไม่ atomic) | ข้อมูลเดียวกันต้องมีเจ้าของหลายคนพร้อมกันในโปรแกรม single-threaded (เช่น graph, tree ที่หลาย node ต้องอ้างอิงถึงกัน) |
| `RefCell<T>` (Part 28) | ไม่เกี่ยวกับ ownership โดยตรง — ให้ interior mutability | heap หรือ stack ก็ได้ (มักใช้คู่กับ `Rc<T>`) | `&T`/`&mut T` ที่ตรวจกฎ borrowing **ตอน runtime** (panic ถ้าผิดกฎ) | **ไม่** ปลอดภัยข้าม thread | ต้องการแก้ไขข้อมูลผ่าน reference ที่ดูเหมือน immutable (เช่นข้างใน `Rc<T>`) |
| `Weak<T>` (บทนี้) | **ไม่เป็นเจ้าของเลย** — ไม่นับใน `strong_count` | ชี้ไปยัง allocation เดียวกับ `Rc<T>`/`Arc<T>` ต้นทาง | ต้อง `.upgrade()` เป็น `Option<Rc<T>>` ก่อนใช้เสมอ | ขึ้นกับว่า downgrade มาจาก `Rc<T>` หรือ `Arc<T>` | แก้ปัญหา reference cycle — ใช้กับความสัมพันธ์ที่ "รู้จักกัน" แต่ไม่ควร "เป็นเจ้าของกัน" เช่น child → parent |
| `Cow<T>` (บทนี้) | อาจเป็นเจ้าของ (`Owned`) หรือแค่ยืม (`Borrowed`) แล้วแต่สถานการณ์ | ขึ้นกับ variant — `Borrowed` ไม่ allocate เลย, `Owned` allocate บน heap | เหมือน `&T` เสมอผ่าน `Deref` ไม่ว่าจะเป็น variant ไหน | ขึ้นกับ `T` (โดยทั่วไปไม่มีปัญหา thread-safety เพราะไม่มีการนับ reference ใด ๆ) | ฟังก์ชันที่ "ส่วนใหญ่ไม่ต้องแก้ไขข้อมูล บางครั้งต้องแก้ไข" — หลีกเลี่ยง allocation ที่ไม่จำเป็นในกรณีส่วนใหญ่ |

**หลักในการเลือกใช้แบบสรุปสั้น ๆ**:

- ต้องการ "เจ้าของเดียว บน heap" → `Box<T>`
- ต้องการ "เจ้าของร่วมกันหลายคน" (single-threaded) → `Rc<T>`
- ต้องการ "แก้ไขข้อมูลผ่าน reference ที่ดูเหมือนอ่านอย่างเดียว" → `RefCell<T>` (มักใช้คู่กับ `Rc<T>` เป็น `Rc<RefCell<T>>`)
- ต้องการ "อ้างอิงกลับ โดยไม่อยากให้มันมีผลต่อการมีชีวิตของข้อมูลต้นทาง" (แก้ปัญหา cycle) → `Weak<T>`
- ต้องการ "คืนค่าที่บางครั้ง borrow บางครั้งต้อง clone แล้วแต่สถานการณ์" (ประหยัด allocation) → `Cow<T>`
- ต้องการทุกอย่างข้างบนแต่ต้องใช้ข้าม thread หลาย thread พร้อมกัน → รอพบใน **Part 39** กับ `Arc<T>` (Atomic
  Rc — เวอร์ชันของ `Rc<T>` ที่ตัวนับเป็น atomic ปลอดภัยข้าม thread) และ `Mutex<T>`/`RwLock<T>` (เวอร์ชันของ
  `RefCell<T>` ที่ตรวจ borrowing ข้าม thread ได้ปลอดภัย)

### 29.11 Interior Mutability Spectrum: `Cell<T>`, `RefCell<T>`, และ `Mutex<T>`/`RwLock<T>`

ก่อนจะไปดูตัวอย่างใหญ่ปิดท้าย เรามาปิดภาพรวมของ **"ตระกูล interior mutability"** ทั้งหมดที่กระจายอยู่ในบทที่ 28 และ
29 นี้ให้เห็นเป็นภาพเดียวกัน คำถามที่ทั้งตระกูลนี้พยายามตอบคือคำถามเดียวกันหมด: **"ฉันมี `&T` (reference ที่ดูเหมือน
อ่านได้อย่างเดียว) แต่ฉันอยากแก้ไขข้อมูลข้างในมันได้ — ทำอย่างไรให้ทำได้โดยไม่ละเมิดกฎ ownership/borrowing ที่ Rust
การันตีไว้?"**

คำตอบไม่ได้มีแค่หนึ่งเดียว เพราะสถานการณ์การใช้งานต่างกัน ทำให้ tradeoff ที่ต้องการก็ต่างกันไปด้วย — นี่คือเหตุผลที่
Rust มีเครื่องมือหลายตัวสำหรับปัญหาเดียวกันนี้ ไม่ใช่ตัวเดียวที่ตอบโจทย์ทุกกรณี:

```rust
use std::cell::{Cell, RefCell};
use std::sync::Mutex;

fn main() {
    // Cell<T>: ใช้กับ T ที่เป็น Copy ล้วน ๆ ไม่มีการยืม/คืนค่าอ้างอิงออกมาเลย
    let counter = Cell::new(0);
    counter.set(counter.get() + 1);
    println!("Cell counter = {}", counter.get());

    // RefCell<T>: ใช้กับ T ใดก็ได้ ตรวจ borrow rule ตอน runtime แทน compile time
    let list = RefCell::new(vec![1, 2, 3]);
    list.borrow_mut().push(4);
    println!("RefCell list = {:?}", list.borrow());

    // Mutex<T>: เหมือน RefCell แต่ปลอดภัยข้าม thread ได้ (ใช้ .lock() แทน .borrow_mut())
    let shared = Mutex::new(0);
    {
        let mut guard = shared.lock().unwrap();
        *guard += 10;
    }
    println!("Mutex value = {}", *shared.lock().unwrap());
}
```

ผลลัพธ์:

```
Cell counter = 1
RefCell list = [1, 2, 3, 4]
Mutex value = 10
```

มาดูเหตุผลเชิงลึกว่าทำไมแต่ละตัวถึงมี tradeoff ต่างกัน:

| เครื่องมือ | ใช้กับ type แบบไหน | วิธีตรวจกฎ borrowing | คืน reference ออกมาได้ไหม | ปลอดภัยข้าม thread ไหม | Overhead ตอน runtime |
|---|---|---|---|---|---|
| **`Cell<T>`** | ต้องเป็น `Copy` เท่านั้น (จริง ๆ ใช้กับ non-`Copy` ได้ผ่าน `.replace()`/`.take()` แต่ที่ใช้บ่อยที่สุดคือกับ `Copy`) | **ไม่ตรวจเลย** เพราะไม่มีการคืน reference ออกมาให้ต้องดูแลอายุ — ใช้ `.get()` (คืนค่า copy ใหม่) และ `.set()` (เขียนทับค่าทั้งหมด) เท่านั้น | **ไม่ได้** — ไม่มี `.borrow()`/`.borrow_mut()` ให้เลย | ไม่ | **เกือบเป็นศูนย์ (zero-cost)** — แค่ copy ค่าเข้า-ออก ไม่มีการตรวจสอบใด ๆ ตอน runtime เลย |
| **`RefCell<T>`** | type ใดก็ได้ (ไม่ต้อง `Copy`) | ตรวจ **ตอน runtime** ด้วยตัวนับ borrow ภายใน — panic ถ้าละเมิดกฎ (borrow_mut ซ้อนกัน หรือ borrow_mut ทับ borrow) | **ได้** — `.borrow()` คืน `Ref<T>`, `.borrow_mut()` คืน `RefMut<T>` (ทั้งสอง deref เป็น `&T`/`&mut T` ได้) | ไม่ (ตัวนับไม่ใช่ atomic) | เบามาก (แค่เพิ่ม/ลดตัวเลขนับ + เช็คเงื่อนไข) แต่ไม่เป็นศูนย์เหมือน `Cell<T>` |
| **`Mutex<T>`/`RwLock<T>`** (Part 39) | type ใดก็ได้ | ตรวจ **ตอน runtime ข้าม thread จริง** ด้วย OS-level lock — thread อื่น**รอ (block)** จนกว่า lock จะถูกปล่อย ไม่ panic แต่รอ | **ได้** — `.lock()` คืน `MutexGuard<T>` (deref เป็น `&mut T`) | **ใช่ ปลอดภัยข้าม thread เต็มรูปแบบ** | สูงกว่าทั้งสองตัวข้างบนมาก (ต้องมี synchronization primitive จริงระดับ OS/hardware) |

จุดที่น่าสนใจที่สุดคือ **ทั้งสามตัวนี้เรียงลำดับกันเป็น spectrum ของ "ยอมเสีย performance เพิ่มขึ้น เพื่อได้ความสามารถ
เพิ่มขึ้น"**:

- `Cell<T>` เร็วที่สุด (zero-cost) แต่ **จำกัดเฉพาะ type ที่ copy ได้ง่าย** และไม่มีทางยืม reference ออกมาแก้ไขบางส่วน
  ของข้อมูลได้เลย (ต้องเขียนทับทั้งค่าเสมอผ่าน `.set()`) — เหมาะกับ counter, flag, ค่าตัวเลขเดี่ยว ๆ
- `RefCell<T>` ยืดหยุ่นกว่ามาก (ใช้กับ type ซับซ้อนอย่าง `Vec<T>`, `HashMap<K, V>`, struct ใดก็ได้) แต่ **ต้องยอมรับ
  ความเสี่ยงที่โปรแกรมจะ panic ตอน runtime** ถ้าโค้ดละเมิดกฎ borrowing (ตามที่เรียนมาจาก Part 28) — เหมาะกับ
  single-threaded program ที่ต้องการ interior mutability กับข้อมูลซับซ้อน
- `Mutex<T>`/`RwLock<T>` ยืดหยุ่นเท่า `RefCell<T>` แต่เพิ่มความสามารถ **ปลอดภัยข้าม thread จริง** — แลกมาด้วย
  overhead ที่สูงกว่า (ต้องมี lock ระดับ OS) และเปลี่ยนพฤติกรรมจาก "panic ถ้าละเมิดกฎ" เป็น "รอ (block) จนกว่า lock
  จะว่าง" ซึ่งเป็นพฤติกรรมที่เหมาะกับ concurrent program มากกว่า — นี่คือสิ่งที่ **Part 39** จะเจาะลึกให้เต็มรูปแบบ

**ทำไม Rust ไม่รวมทั้งสามตัวเป็นตัวเดียว?** เพราะ tradeoff ที่ต่างกันมีความหมายจริงในโลกการเขียนโปรแกรม — ถ้าคุณ
เขียนโปรแกรม single-threaded ธรรมดาแล้วต้องการแค่ counter ตัวเดียว การบังคับให้ใช้ `Mutex<T>` (ที่มี overhead ของ
OS lock) จะเป็นการสิ้นเปลืองที่ไม่จำเป็นเลย ในทางกลับกัน ถ้าคุณเขียนโปรแกรม multi-threaded การใช้ `RefCell<T>`
(ที่ตัวนับไม่ atomic) จะเป็นอันตรายจริง — Rust เลือกให้คุณ **"จ่ายเฉพาะในสิ่งที่คุณใช้"** (pay only for what you
use) ซึ่งเป็นหลักการ zero-cost abstraction แบบเดียวกับที่เราเห็นมาตลอดหลักสูตรนี้ ตั้งแต่เรื่อง ownership/borrowing
ใน Part 6-7 จนถึงตระกูล smart pointer ทั้งหมดในบทที่ 27-29 นี้

#### `Cell<T>` กับ Type ที่ไม่ใช่ `Copy`: `.take()`, `.replace()`, `.into_inner()`

หัวข้อ 29.11 (และ Part 28) บอกว่า `Cell<T>` "ใช้กับ type ที่เป็น `Copy` เท่านั้น" — ข้อความนี้ถูกแค่ครึ่งเดียว ในความ
จริง `Cell<T>` ใช้กับ type ใดก็ได้ (ไม่บังคับ `Copy`) เพียงแต่ **ไม่มี method ที่คืน reference เข้าไปข้างในให้เลย**
(ไม่มี `.borrow()`/`.borrow_mut()` แบบ `RefCell<T>`) — สำหรับ type ที่ไม่ใช่ `Copy` จะใช้งานผ่าน method ที่ "ย้าย
ค่าออกมาทั้งก้อน" แทน:

- **`.take(&self) -> T`** (ต้องการ `T: Default`): ดึงค่าปัจจุบันออกมาเป็นเจ้าของ แล้วแทนที่ค่าข้างใน `Cell` ด้วย
  `T::default()`
- **`.replace(&self, new: T) -> T`**: ดึงค่าปัจจุบันออกมาเป็นเจ้าของ แล้วแทนที่ด้วยค่าใหม่ที่กำหนด
- **`.into_inner(self) -> T`**: กิน `Cell<T>` ทั้งตัว (ต้องการ ownership เต็ม ไม่ใช่ `&self`) แล้วคืนค่า `T` ข้างในออก
  มาตรง ๆ

```rust
use std::cell::Cell;

fn main() {
    let cell = Cell::new(String::from("hello"));
    let taken = cell.take(); // เอาค่าออกมา แล้วปล่อยค่า default (String::new()) ไว้แทนข้างใน cell
    println!("taken = {taken}");
    println!("cell เหลืออยู่ (หลัง into_inner) = {:?}", cell.into_inner());
}
```

ผลลัพธ์:

```
taken = hello
cell เหลืออยู่ (หลัง into_inner) = ""
```

สังเกตว่าไม่มีจุดใดในโค้ดนี้ที่ได้ `&String` หรือ `&mut String` ออกมาเลยแม้แต่ตอนเดียว — ทุกการเข้าถึงเป็นการ "ย้าย
ค่าทั้งก้อนเข้า-ออก" เท่านั้น (`.take()` ย้ายค่าออกมาเป็นเจ้าของเต็ม ๆ แล้วใส่ค่า default กลับเข้าไปแทน) นี่คือ
เหตุผลที่ `Cell<T>` ไม่ต้องมีการตรวจสอบกฎ borrowing ใด ๆ เลยตอน runtime (ไม่มี `.borrow()` ที่อาจ panic ถ้าเรียกซ้อน
ผิดจังหวะแบบ `RefCell<T>`) — เพราะไม่มี reference ที่มี "อายุการใช้งาน" ให้ต้องดูแลเลยตั้งแต่ต้น เป็นเหตุผลเดียวกับที่
ทำให้ `Cell<T>` เร็วกว่า `RefCell<T>` ตามตารางเปรียบเทียบก่อนหน้านี้

### 29.12 ตัวอย่างใหญ่: ระบบ Org Chart (โครงสร้างองค์กร) ด้วย `Rc<RefCell<T>>` + `Weak<RefCell<T>>`

มาปิดบทด้วยตัวอย่างจริงที่รวมทุกอย่างที่เรียนมาในบทนี้เข้าด้วยกัน — ระบบจัดการโครงสร้างองค์กร (org chart) ที่แต่ละ
`Employee` มี **ผู้ใต้บังคับบัญชา (subordinates)** ที่ตัวเองเป็นเจ้าของอย่างชัดเจน (ใช้ `Rc<RefCell<T>>` — ความ
สัมพันธ์แบบ "ฉันดูแลคนพวกนี้") และมี **ผู้จัดการ (parent)** ที่ตัวเองแค่รู้จัก ไม่ได้เป็นเจ้าของ (ใช้
`Weak<RefCell<T>>` — ความสัมพันธ์แบบ "ฉันรู้ว่าใครดูแลฉัน" ที่ไม่ควรไปยึดชีวิตของผู้จัดการไว้)

สังเกตว่าเราไม่จำเป็นต้องห่อ field `parent`/`subordinates` ด้วย `RefCell<...>` **ซ้อนเข้าไปอีกชั้น** เหมือนตัวอย่าง
`Node` ในหัวข้อ 29.4 เพราะคราวนี้เราเลือกห่อ **ทั้ง struct `Employee` ไว้ใน `RefCell<Employee>` ตั้งแต่ต้น** (ผ่าน
`Rc<RefCell<Employee>>`) — การมี `RefCell` อยู่รอบนอกแบบนี้ทำให้ field ข้างในทุกตัวแก้ไขได้ผ่าน `.borrow_mut()`
เดียวอยู่แล้ว ไม่ต้องมี `RefCell` ซ้อนกันหลายชั้นให้ยุ่งยาก — นี่คือสองแนวทางออกแบบที่ใช้กันจริง เลือกใช้ตามความ
เหมาะสมของโครงสร้างข้อมูล

```rust
use std::cell::RefCell;
use std::rc::{Rc, Weak};

struct Employee {
    name: String,
    parent: Weak<RefCell<Employee>>,        // ผู้จัดการ — รู้จัก แต่ไม่ได้เป็นเจ้าของ
    subordinates: Vec<Rc<RefCell<Employee>>>, // ผู้ใต้บังคับบัญชา — เป็นเจ้าของจริง
}

// พิสูจน์ว่า Employee ถูก drop จริงตอนไม่มีใครถืออ้างอิงอยู่แล้ว (ไม่มี memory leak)
impl Drop for Employee {
    fn drop(&mut self) {
        println!("Employee ถูก drop: {}", self.name);
    }
}

impl Employee {
    fn new(name: &str) -> Rc<RefCell<Employee>> {
        Rc::new(RefCell::new(Employee {
            name: name.to_string(),
            parent: Weak::new(), // ยังไม่มีผู้จัดการตอนสร้าง
            subordinates: Vec::new(),
        }))
    }
}

/// เพิ่ม `sub` เข้าเป็นผู้ใต้บังคับบัญชาของ `manager`
/// ทิศทาง manager -> sub เป็น strong (Rc::clone), ทิศทาง sub -> manager เป็น weak (Rc::downgrade)
fn add_subordinate(manager: &Rc<RefCell<Employee>>, sub: &Rc<RefCell<Employee>>) {
    sub.borrow_mut().parent = Rc::downgrade(manager);
    manager.borrow_mut().subordinates.push(Rc::clone(sub));
}

/// หาชื่อผู้จัดการของพนักงานคนหนึ่ง — คืน None ถ้าเป็นตำแหน่งสูงสุด (ไม่มีผู้จัดการ) หรือถ้าผู้จัดการถูก drop ไปแล้ว
fn find_manager_name(emp: &Rc<RefCell<Employee>>) -> Option<String> {
    emp.borrow().parent.upgrade().map(|m| m.borrow().name.clone())
}

/// นับจำนวนผู้ใต้บังคับบัญชาทั้งหมด รวมทางอ้อม (ลูกของลูกด้วย) แบบ recursive
fn total_team_size(manager: &Rc<RefCell<Employee>>) -> usize {
    let subs = &manager.borrow().subordinates;
    let mut count = subs.len();
    for sub in subs {
        count += total_team_size(sub);
    }
    count
}

fn main() {
    let ceo = Employee::new("สมชาย (CEO)");
    let cto = Employee::new("สมหญิง (CTO)");
    let dev1 = Employee::new("วิชัย (Developer)");
    let dev2 = Employee::new("มานี (Developer)");

    add_subordinate(&ceo, &cto);
    add_subordinate(&cto, &dev1);
    add_subordinate(&cto, &dev2);

    println!("ผู้จัดการของ dev1 คือ: {:?}", find_manager_name(&dev1));
    println!("ผู้จัดการของ ceo คือ: {:?}", find_manager_name(&ceo));
    println!("ทีมทั้งหมดใต้ ceo (รวมทางอ้อม): {}", total_team_size(&ceo));

    println!(
        "ceo    strong={}, weak={}",
        Rc::strong_count(&ceo),
        Rc::weak_count(&ceo)
    );
    println!(
        "cto    strong={}, weak={}",
        Rc::strong_count(&cto),
        Rc::weak_count(&cto)
    );
    println!(
        "dev1   strong={}, weak={}",
        Rc::strong_count(&dev1),
        Rc::weak_count(&dev1)
    );

    drop(dev1);
    drop(dev2);
    drop(cto);
    drop(ceo);
    println!("ออกจาก main แล้ว — สังเกตว่า Employee ทุกตัวถูก drop จริง ไม่มี memory leak");
}
```

ผลลัพธ์:

```
ผู้จัดการของ dev1 คือ: Some("สมหญิง (CTO)")
ผู้จัดการของ ceo คือ: None
ทีมทั้งหมดใต้ ceo (รวมทางอ้อม): 3
ceo    strong=1, weak=1
cto    strong=2, weak=2
dev1   strong=2, weak=0
Employee ถูก drop: สมชาย (CEO)
Employee ถูก drop: สมหญิง (CTO)
Employee ถูก drop: วิชัย (Developer)
Employee ถูก drop: มานี (Developer)
ออกจาก main แล้ว — สังเกตว่า Employee ทุกตัวถูก drop จริง ไม่มี memory leak
```

มาอ่านตัวเลขแต่ละบรรทัดให้เข้าใจครบ:

- **`find_manager_name(&dev1)` คืน `Some("สมหญิง (CTO)")`**: `dev1.parent.upgrade()` สำเร็จเพราะ `cto` (ผู้จัดการ
  ของ `dev1`) ยังมีชีวิตอยู่จริงตอนนั้น — เห็นการใช้งาน `Option<T>` จาก Part 11 ที่ผูกกับ `upgrade()` ตรง ๆ ตามที่
  อธิบายไว้ในหัวข้อ 29.3
- **`find_manager_name(&ceo)` คืน `None`**: `ceo.parent` ยังเป็น `Weak::new()` (ไม่มีใครกำหนดผู้จัดการให้ตั้งแต่ต้น
  เพราะ CEO คือตำแหน่งสูงสุดแล้ว) — `.upgrade()` บน `Weak::new()` จะได้ `None` เสมอ นี่คือกรณีปกติที่ไม่ใช่ error เลย
- **`total_team_size(&ceo)` คืน `3`**: เดิน recursive ผ่าน `ceo.subordinates` (มี `cto`) แล้วเดินต่อไปยัง
  `cto.subordinates` (มี `dev1`, `dev2`) รวมทั้งหมด 1 + 2 = 3 คน
- **ตัวเลข `strong`/`weak`**: `ceo` มี `strong = 1` (แค่ตัวแปร `ceo` เอง — ไม่มีใครเป็นผู้จัดการของ CEO เลยจึงไม่มีใคร
  clone strong reference มาเพิ่ม) และ `weak = 1` (เพราะ `cto.parent` ถือ `Weak` ชี้กลับมาที่ `ceo`) ส่วน `cto` มี
  `strong = 2` (ตัวแปร `cto` เอง + ที่ `ceo.subordinates` ถืออยู่) และ `weak = 2` (จาก `dev1.parent` และ
  `dev2.parent` ที่ชี้กลับมาที่ `cto` ทั้งคู่)
- **บรรทัดสุดท้ายที่สำคัญที่สุด**: หลังเรียก `drop()` ไล่ทุกตัวแปรจนหมด **ข้อความ "Employee ถูก drop" ปรากฏขึ้นครบ
  ทั้ง 4 คน** — พิสูจน์ได้ชัดเจนว่า**ไม่มี memory leak เกิดขึ้นเลย** ต่างจากตัวอย่างในหัวข้อ 29.1 ที่ไม่มีข้อความ
  drop ปรากฏออกมาสักบรรทัดเดียว — นี่คือผลลัพธ์ตรงจากการออกแบบความสัมพันธ์ parent-child ให้ถูกทิศทาง (strong ไปทาง
  เดียว, weak กลับทาง) ตามหลักการที่อธิบายไว้ในหัวข้อ 29.2 และ 29.4

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. เข้าถึง field ผ่าน `Weak<T>` ตรง ๆ โดยไม่ upgrade ก่อน

`Weak<T>` **ไม่ implement `Deref`** เหมือน `Rc<T>` — มันเป็นแค่ "ตัวชี้ที่ปลอดภัย" ไปยัง allocation กลาง ไม่ใช่ตัวชี้
ไปยังข้อมูล `T` ที่ใช้งานได้ตรง ๆ ถ้าคุณพยายามเข้าถึง field หรือเรียก method ผ่าน `Weak<T>` โดยตรง compiler จะฟ้อง
ทันที:

```rust
use std::rc::{Rc, Weak};

struct Data {
    value: i32,
}

fn main() {
    let strong = Rc::new(Data { value: 42 });
    let weak: Weak<Data> = Rc::downgrade(&strong);

    // พยายามเข้าถึง field ผ่าน Weak ตรง ๆ โดยไม่ upgrade ก่อน
    println!("{}", weak.value);
}
```

```
error[E0609]: no field `value` on type `std::rc::Weak<Data>`
  --> src/main.rs:12:25
   |
12 |     println!("{}", weak.value);
   |                         ^^^^^ unknown field
```

**วิธีแก้**: เรียก `.upgrade()` ก่อนเสมอ แล้วจัดการทั้งกรณี `Some`/`None` ให้ครบ:

```rust
if let Some(rc) = weak.upgrade() {
    println!("{}", rc.value);
} else {
    println!("ข้อมูลถูกทำลายไปแล้ว");
}
```

### 2. เรียก `.unwrap()` บนผลลัพธ์ของ `.upgrade()` แล้ว panic ตอน runtime

การเขียน `weak.upgrade().unwrap()` เป็นการบอก compiler ว่า "ฉันมั่นใจ 100% ว่าข้อมูลนี้ยังอยู่แน่นอน ถ้าไม่จริงให้
โปรแกรม crash ไปเลย" — ซึ่งมักเป็นสมมติฐานที่ผิดในโค้ดจริง เพราะจุดประสงค์หลักของ `Weak<T>` คือการรองรับสถานการณ์ที่
ข้อมูลอาจถูกทำลายไปแล้วโดยไม่มีการเตือนล่วงหน้า:

```rust
use std::rc::{Rc, Weak};

fn main() {
    let weak: Weak<i32>;
    {
        let strong = Rc::new(100);
        weak = Rc::downgrade(&strong);
    } // strong หมด scope ตรงนี้ ข้อมูลถูกทำลายไปแล้ว

    let value = weak.upgrade().unwrap(); // panic เพราะ upgrade คืน None
    println!("{value}");
}
```

```
thread 'main' panicked at src/main.rs:10:32:
called `Option::unwrap()` on a `None` value
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

**วิธีแก้**: ใช้ `match`, `if let`, หรือ combinator อย่าง `.map()`/`.unwrap_or()`/`.unwrap_or_else()` เพื่อจัดการ
กรณี `None` อย่างมีเหตุผลตามบริบทของโปรแกรม (เช่นข้ามการประมวลผล node นั้นไปเงียบ ๆ, log คำเตือน, หรือคืนค่า default)
แทนการสมมติว่า `Some` เสมอ

### 3. คืน `Cow::Borrowed` ที่อ้างอิงไปยังตัวแปร local — Lifetime ไม่ตรงกัน

`Cow<'a, str>` มี lifetime parameter `'a` ที่ผูกกับข้อมูลต้นทาง — ถ้าคุณสร้าง `String` ขึ้นมาใหม่ในฟังก์ชัน (เป็น
ตัวแปร local ที่จะถูก drop เมื่อฟังก์ชันจบ) แล้วพยายามคืน `Cow::Borrowed` ที่อ้างอิงกลับไปยังตัวแปรนั้น จะเจอปัญหา
เดียวกับการคืน dangling reference ที่เรียนมาจาก Part 7-8:

```rust
use std::borrow::Cow;

fn broken(input: &str) -> Cow<str> {
    let local = input.to_uppercase(); // String ที่สร้างขึ้นใน scope นี้เท่านั้น
    Cow::Borrowed(&local) // ❌ ยืม reference ไปยัง local ที่กำลังจะถูก drop
}

fn main() {
    let result = broken("hi");
    println!("{result}");
}
```

```
error[E0515]: cannot return value referencing local variable `local`
 --> src/main.rs:5:5
  |
5 |     Cow::Borrowed(&local) // ❌ ยืม reference ไปยัง local ที่กำลังจะถูก drop
  |     ^^^^^^^^^^^^^^------^
  |     |             |
  |     |             `local` is borrowed here
  |     returns a value referencing data owned by the current function
```

**วิธีแก้**: ถ้าข้อมูลถูกสร้างขึ้นใหม่ในฟังก์ชัน (ไม่ได้ borrow มาจาก input เดิม) ต้องคืนแบบ `Cow::Owned(local)`
เสมอ — ใช้ `Cow::Borrowed` ได้เฉพาะตอนที่ reference ที่คุณจะคืนกลับ **มาจาก parameter ที่รับเข้ามาโดยตรง** เท่านั้น
(หรือจากข้อมูลอื่นที่มี lifetime ยาวพอ ๆ กัน) นี่คือกฎเดียวกับเรื่อง lifetime ที่เรียนมาตั้งแต่ Part 20/23 เพียงแค่
มาปรากฏในบริบทของ `Cow<T>`

### 4. สลับทิศทาง Strong/Weak โดยไม่ตั้งใจ — สร้างวงจรกลับมาอีกครั้งทั้งที่ตั้งใจจะแก้

ความเข้าใจผิดที่พบบ่อยมากคือคิดว่า "แค่มี `Weak<T>` อยู่ในโครงสร้างที่ไหนสักที่ก็เพียงพอที่จะป้องกัน cycle แล้ว"
— ความจริงคือ **ทิศทาง** ต้องถูกด้วย ถ้าใช้ `Rc::clone()` (strong) ผิดจุดแทนที่จะใช้ `Rc::downgrade()` (weak) วงจร
ก็ยังเกิดขึ้นเหมือนเดิม แม้ว่าคุณจะ `use std::rc::Weak;` ไว้ในโค้ดแล้วก็ตาม:

```rust
use std::cell::RefCell;
use std::rc::Rc;

struct Node {
    name: String,
    parent: RefCell<Option<Rc<Node>>>, // ❌ ควรเป็น Weak<Node> ไม่ใช่ Rc<Node>
    children: RefCell<Vec<Rc<Node>>>,
}

impl Drop for Node {
    fn drop(&mut self) {
        println!("drop: {}", self.name);
    }
}

fn main() {
    let parent = Rc::new(Node {
        name: "parent".to_string(),
        parent: RefCell::new(None),
        children: RefCell::new(vec![]),
    });

    let child = Rc::new(Node {
        name: "child".to_string(),
        parent: RefCell::new(Some(Rc::clone(&parent))), // ใช้ Rc::clone แทน Rc::downgrade โดยไม่ตั้งใจ
        children: RefCell::new(vec![]),
    });

    parent.children.borrow_mut().push(Rc::clone(&child));

    println!("parent strong_count = {}", Rc::strong_count(&parent));
    // strong_count ของ parent คือ 2 (ตัวแปร parent เอง + ที่ child.parent ถืออยู่)
    // เมื่อ main จบ ทั้งสอง Rc นี้จะไม่ถูก drop เลย เพราะอ้างวนกัน (ไม่มีข้อความ "drop:" ปรากฏเลย)
}
```

รันแล้วได้:

```
parent strong_count = 2
```

**ไม่มีข้อความ `drop: parent` หรือ `drop: child` ปรากฏออกมาเลยแม้แต่บรรทัดเดียว** — เป็น memory leak แบบเดียวกับ
หัวข้อ 29.1 เป๊ะ ๆ เพียงแค่ครั้งนี้ field ชื่อ `parent` "ดูเหมือน" ถูกออกแบบให้เป็น back-reference แล้ว แต่ type ที่
เลือกใช้ (`Rc<Node>` แทน `Weak<Node>`) ยังผิดอยู่ **วิธีแก้**: ตรวจสอบทุกครั้งว่า field ที่ตั้งใจให้เป็น "แค่รู้จัก
ไม่ได้เป็นเจ้าของ" ต้องมี type เป็น `Weak<T>` จริง ๆ และตอนกำหนดค่าต้องใช้ `Rc::downgrade()` ไม่ใช่ `Rc::clone()`
— ยิ่งดีกว่านั้นคือตั้งชื่อ field/ตัวแปรให้สื่อความหมายชัดเจน (เช่น `parent: Weak<...>` ไม่ใช่แค่ `back_ref`) เพื่อ
ให้ code reviewer และตัวเราเองในอนาคตจับผิดได้ง่ายขึ้น

### 5. เรียก `.borrow_mut()` ซ้อนกันสองครั้งพร้อมกัน — ทวนกฎ `RefCell<T>` จาก Part 28

เนื่องจากบทนี้ใช้ `RefCell<T>` ควบคู่กับ `Rc<T>`/`Weak<T>` อย่างหนักในตัวอย่าง org chart และ tree จึงควรทวนกับดัก
คลาสสิกจาก Part 28 อีกครั้ง เพราะพบได้บ่อยเป็นพิเศษเมื่อโค้ดมีการเดิน tree แบบ recursive ที่อาจ borrow ค่าเดียวกัน
ซ้อนกันโดยไม่ตั้งใจ:

```rust
use std::cell::RefCell;

fn main() {
    let data = RefCell::new(vec![1, 2, 3]);

    let first_borrow = data.borrow_mut();
    let second_borrow = data.borrow_mut(); // ❌ ยืมแบบ mutable ซ้อนกันตอน runtime

    println!("{:?} {:?}", first_borrow, second_borrow);
}
```

```
thread 'main' panicked at src/main.rs:7:30:
RefCell already borrowed
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

**วิธีแก้**: ทำให้ scope ของ `Ref`/`RefMut` ที่ได้จาก `.borrow()`/`.borrow_mut()` **สั้นที่สุดที่จำเป็น** — ปิด scope
ของมันด้วย block `{ }` ทันทีที่ใช้งานเสร็จ หรือดึงค่าที่ต้องใช้ออกมาเก็บเป็นตัวแปรใหม่ก่อน (เช่น `.clone()` ค่าที่
ต้องใช้ต่อ) แล้วปล่อย `Ref`/`RefMut` ตัวเดิมไปก่อนที่จะ borrow ครั้งใหม่ — ในตัวอย่าง `total_team_size()` จากหัวข้อ
29.12 จะสังเกตว่าเราเขียน `let subs = &manager.borrow().subordinates;` แล้วจบการใช้ `.borrow()` แค่บรรทัดเดียว
ก่อนจะเข้า loop ที่เรียก `total_team_size(sub)` แบบ recursive ซึ่งจะไป `.borrow()` ค่าของ node อื่น (คนละตัวกับ
`manager`) — ไม่มีการ borrow ค่าเดียวกันซ้อนกันเลย

### 6. ลืม import `Weak<T>` หรือสับสนระหว่าง `std::rc::Weak` กับ `std::sync::Weak`

`Weak<T>` ไม่ได้อยู่ใน prelude (ชุด type ที่ Rust import ให้อัตโนมัติทุกไฟล์) เหมือน `Option<T>`/`Result<T, E>` —
ต้อง `use std::rc::Weak;` เองเสมอ (หรือ `use std::sync::Weak;` ถ้าใช้คู่กับ `Arc<T>` ที่จะเรียนใน Part 39) ถ้าลืม
import compiler จะฟ้อง:

```rust
use std::rc::Rc;

fn main() {
    let strong = Rc::new(5);
    let weak: Weak<i32> = Rc::downgrade(&strong);
    println!("{:?}", weak.upgrade());
}
```

```
error[E0425]: cannot find type `Weak` in this scope
 --> src/main.rs:5:15
  |
5 |     let weak: Weak<i32> = Rc::downgrade(&strong);
  |               ^^^^ not found in this scope
  |
help: consider importing one of these structs
  |
1 + use std::rc::Weak;
  |
1 + use std::sync::Weak;
  |
```

สังเกตว่า compiler แนะนำให้เลือกได้ **สองทาง**: `std::rc::Weak` (คู่กับ `Rc<T>` — single-threaded อย่างที่เรียนมา
ทั้งบทนี้) และ `std::sync::Weak` (คู่กับ `Arc<T>` — เวอร์ชัน thread-safe ที่จะเรียนเต็ม ๆ ใน **Part 39**) ทั้งสองมี
API หน้าตาเหมือนกันทุกประการ (`.upgrade()` คืน `Option<Arc<T>>` แทน `Option<Rc<T>>`) เพราะแนวคิดเบื้องหลังเหมือนกัน
เป๊ะ ๆ ต่างกันแค่ว่าตัวนับ reference เป็น atomic (ปลอดภัยข้าม thread) หรือไม่เท่านั้น **วิธีแก้**: เลือก import
ให้ตรงกับว่าโค้ดส่วนนั้นใช้ `Rc<T>` หรือ `Arc<T>` — ถ้าใช้ `Rc<T>` (แบบในบทนี้) ต้อง `use std::rc::Weak;` เท่านั้น
ห้ามผสมกันเด็ดขาด เพราะ `Rc::downgrade()` คืน `std::rc::Weak<T>` ไม่ใช่ `std::sync::Weak<T>` (compiler จะฟ้อง type
mismatch ถ้าประกาศ type ผิดฝั่ง)

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** โค้ดในกับดักข้อ 4 (`Node` ที่ใช้ `Rc<Node>` ผิดจุดสำหรับ field `parent`) มี memory leak เพราะ type ผิด
   ให้แก้ไข struct `Node` ให้ field `parent` เป็น `Weak<Node>` ที่ถูกต้อง ปรับจุดที่กำหนดค่า `child.parent` ให้ใช้
   `Rc::downgrade(&parent)` แทน `Rc::clone(&parent)` แล้วรันโปรแกรมใหม่ ตรวจสอบว่าครั้งนี้ข้อความ `drop: parent` และ
   `drop: child` ปรากฏออกมาทั้งคู่หรือไม่

   (hint: field `parent` ต้องเปลี่ยนจาก `RefCell<Option<Rc<Node>>>` เป็น `RefCell<Option<Weak<Node>>>`
   — ลองเทียบกับโครงสร้าง `Node` ในหัวข้อ 29.4 ที่ทำถูกตั้งแต่ต้น)

   เฉลยแบบย่อ: เปลี่ยน `parent: RefCell<Option<Rc<Node>>>` เป็น `parent: RefCell<Option<Weak<Node>>>` และเปลี่ยน
   `RefCell::new(Some(Rc::clone(&parent)))` เป็น `RefCell::new(Some(Rc::downgrade(&parent)))` — หลังแก้แล้วรัน
   จะเห็นข้อความ `drop: child` และ `drop: parent` ปรากฏขึ้นตอนจบ `main` ครบทั้งคู่

2. **[ง่าย-กลาง]** ใช้โครงสร้าง `Employee` จากหัวข้อ 29.12 เขียนฟังก์ชัน `fn depth(emp: &Rc<RefCell<Employee>>) ->
   usize` ที่คำนวณ "ระดับความลึก" ของพนักงานคนหนึ่งในองค์กร (CEO มีระดับความลึก = 0, ลูกน้องตรงของ CEO = 1, ลูกน้อง
   ของลูกน้องนั้น = 2 และเป็นแบบนี้ต่อไป) โดยเดินขึ้นไปทาง `parent` ด้วย recursive function จนกว่า `.upgrade()` จะ
   คืน `None` (แสดงว่าถึงตำแหน่งสูงสุดแล้ว) เขียน `main` ที่สร้างองค์กร 3 ระดับแล้วเรียก `depth()` กับพนักงานแต่ละคน
   เพื่อพิสูจน์ว่าได้ค่าถูกต้อง

   (hint: โครงสร้าง recursive จะคล้ายกับ `total_team_size()` ในหัวข้อ 29.12 มาก แต่เดินขึ้น (`parent`) แทนเดินลง
   (`subordinates`) — เงื่อนไขจบ (base case) คือตอนที่ `.upgrade()` คืน `None`)

   เฉลยแบบย่อ:
   ```rust
   fn depth(emp: &Rc<RefCell<Employee>>) -> usize {
       match emp.borrow().parent.upgrade() {
           Some(parent) => 1 + depth(&parent),
           None => 0,
       }
   }
   ```

3. **[กลาง-ยาก]** เขียนฟังก์ชัน `fn escape_html(input: &str) -> Cow<str>` ที่แทนตัวอักษรพิเศษในภาษา HTML
   (`<` เป็น `&lt;`, `>` เป็น `&gt;`, `&` เป็น `&amp;`) — ถ้า input ไม่มีตัวอักษรพิเศษเหล่านี้เลย ให้ฟังก์ชันคืนค่า
   แบบ `Cow::Borrowed` ตรง ๆ โดยไม่ allocate อะไรเลย แต่ถ้ามีตัวอักษรพิเศษอย่างน้อยหนึ่งตัว ให้คืนค่าแบบ `Cow::Owned`
   ที่ผ่านการ escape ครบถ้วนแล้ว ทดสอบด้วย input สองแบบ (ไม่มีตัวอักษรพิเศษ กับมีตัวอักษรพิเศษ) แล้วดูว่า `Cow`
   ที่ได้เป็น variant ไหนในแต่ละกรณี

   (hint: เช็คก่อนด้วย `.chars().any(...)` ว่ามีตัวอักษรพิเศษหรือไม่ ถ้าไม่มีให้ return ทันทีแบบ `Cow::Borrowed`
   ตัด shortcut ออกไปก่อนที่จะเริ่ม build `String` ใหม่ เพื่อไม่ต้อง allocate อะไรเลยในกรณีที่ไม่จำเป็น)

   เฉลยแบบย่อ:
   ```rust
   fn escape_html(input: &str) -> Cow<'_, str> {
       if input.chars().any(|c| c == '<' || c == '>' || c == '&') {
           let mut result = String::with_capacity(input.len());
           for c in input.chars() {
               match c {
                   '<' => result.push_str("&lt;"),
                   '>' => result.push_str("&gt;"),
                   '&' => result.push_str("&amp;"),
                   other => result.push(other),
               }
           }
           Cow::Owned(result)
       } else {
           Cow::Borrowed(input)
       }
   }
   ```

4. **[ยาก/ประยุกต์ใช้งานจริง]** ต่อยอดจากระบบ org chart ในหัวข้อ 29.12: เขียนฟังก์ชัน `fn remove_subordinate(manager:
   &Rc<RefCell<Employee>>, target_name: &str)` ที่ลบพนักงานที่มีชื่อตรงกับ `target_name` ออกจาก `subordinates` ของ
   `manager` (ใช้ `Vec::retain()` เพื่อกรองออกตามชื่อ) จากนั้นเขียน `main` ที่สร้าง `ceo` กับ `cto` เชื่อมความสัมพันธ์
   กันด้วย `add_subordinate`, พิมพ์ `Rc::strong_count(&cto)` ก่อนลบ, เรียก `remove_subordinate(&ceo, "CTO")`,
   แล้วพิมพ์ `Rc::strong_count(&cto)` อีกครั้งหลังลบ พิสูจน์ด้วยตัวเลขว่า strong_count ลดลงจริงหลังจากลบออกจาก
   `subordinates` (เพราะ `ceo` เลิกเป็นเจ้าของ `cto` ผ่าน strong reference นั้นแล้ว) — ถ้าเพิ่ม `impl Drop for
   Employee` เข้าไปด้วย ลองสังเกตว่าข้อความ drop ของ `cto` ปรากฏขึ้นตอนไหนกันแน่ระหว่างการรันโปรแกรม

   (hint: `Vec::retain(|e| e.borrow().name != target_name)` จะเก็บไว้เฉพาะสมาชิกที่ closure คืน `true` — สมาชิกที่
   ชื่อตรงกับ `target_name` จะถูกดึงออกจาก `Vec` และ `Rc` ตัวนั้นภายใน `Vec` ก็จะถูก drop ไปด้วย ทำให้ strong_count
   ของพนักงานที่ถูกลบลดลง 1 ทันที)

   เฉลยแบบย่อ:
   ```rust
   fn remove_subordinate(manager: &Rc<RefCell<Employee>>, target_name: &str) {
       manager
           .borrow_mut()
           .subordinates
           .retain(|e| e.borrow().name != target_name);
   }
   ```
   ผลลัพธ์ที่คาดหวัง: strong_count ของ `cto` ลดจาก 2 (ตัวแปร `cto` เอง + ที่ `ceo.subordinates` ถืออยู่) เป็น 1
   (แค่ตัวแปร `cto` เอง) ทันทีหลังเรียก `remove_subordinate` — ถ้าไม่มีตัวแปร `cto` เหลืออยู่นอกฟังก์ชันนี้อีกแล้ว
   (เช่นถ้าถูกเรียกจากฟังก์ชันอื่นที่ไม่ได้เก็บตัวแปรไว้) strong_count จะลดถึง 0 และ `Employee` ตัวนั้นจะถูก drop
   ทันทีในบรรทัดที่ `.retain()` ทำงาน

## สรุป

บทนี้คือ**บทปิดของ mini-arc "Smart Pointers"** ที่เริ่มต้นจาก `Box<T>` ใน Part 27 ต่อด้วย `Rc<T>`/`RefCell<T>` ใน
Part 28 และมาจบลงที่บทนี้ด้วยการแก้ปัญหาที่ Part 28 ทิ้งปริศนาไว้: **`Weak<T>`** คือคำตอบของ reference cycle —
ด้วยการแยกความสัมพันธ์ออกเป็นสองแบบอย่างชัดเจนตามความหมายจริง คือ **strong reference** ("ฉันต้องการให้ข้อมูลนี้
มีชีวิตอยู่") กับ **weak reference** ("ฉันแค่อยากรู้ว่าข้อมูลนี้ยังอยู่ไหม แต่ไม่ยึดชีวิตมันไว้") — `Rc::downgrade()`
สร้าง `Weak<T>` จาก `Rc<T>` โดยไม่เพิ่ม `strong_count`, และ `Weak::upgrade()` คืน `Option<Rc<T>>` ที่เชื่อมตรงกับ
ปรัชญา `Option<T>` จาก Part 11: `None` ไม่ใช่ error แต่คือคำตอบที่ปลอดภัยและถูกต้องเมื่อข้อมูลถูกทำลายไปแล้ว —
ปลอดภัยกว่า dangling pointer ใน C/C++ อย่างสิ้นเชิงโดยไม่ต้องพึ่ง garbage collector เลยแม้แต่นิดเดียว

เรายังได้เรียนรู้เครื่องมือที่แยกจากปัญหา cycle โดยสิ้นเชิงแต่สำคัญไม่แพ้กัน คือ **`Cow<T>`** (Clone on Write) —
เครื่องมือที่ช่วยให้ฟังก์ชันคืนค่าแบบ borrowed (ไม่ allocate) ในกรณีส่วนใหญ่ที่ไม่ต้องแก้ไขข้อมูล และคืนค่าแบบ owned
(allocate ใหม่) เฉพาะกรณีที่จำเป็นต้องแก้ไขจริง ๆ ผ่าน `.to_mut()`/`.into_owned()` — เป็นการหลีกเลี่ยง allocation
ที่ไม่จำเป็นซึ่งส่งผลจริงต่อ performance ในโปรแกรมที่ประมวลผลข้อความจำนวนมาก

ปิดท้ายด้วยการมองภาพรวมทั้งหมดของตระกูล **interior mutability** (`Cell<T>` → `RefCell<T>` → `Mutex<T>`/`RwLock<T>`)
ในฐานะ spectrum ของ tradeoff ที่ต่างกัน — ยิ่งต้องการความยืดหยุ่นและความปลอดภัยข้าม thread มากขึ้น ก็ต้องยอมรับ
overhead ที่สูงขึ้นตามไปด้วย ตามหลัก "จ่ายเฉพาะในสิ่งที่คุณใช้" ที่เป็นแก่นของ Rust ทั้งภาษา และตัวอย่างใหญ่ท้ายบท
(ระบบ org chart) ก็รวมทุกแนวคิดในบทนี้และบท 27-28 เข้าด้วยกันเป็นโครงสร้างข้อมูลจริงที่ใช้งานได้ พร้อมพิสูจน์ด้วย
`strong_count()`/`weak_count()` และ `impl Drop` ว่าการออกแบบถูกต้องจริง ไม่มี memory leak หลงเหลืออยู่เลย

ด้วยบทนี้ mini-arc ของ smart pointer ทั้งหมด (`Box<T>`, `Rc<T>`, `RefCell<T>`, `Weak<T>`, `Cow<T>`) ก็ครบสมบูรณ์แล้ว
— คุณมีเครื่องมือครบมือสำหรับจัดการ ownership ที่ซับซ้อนกว่ากรณีพื้นฐาน ไม่ว่าจะเป็นข้อมูลที่มีเจ้าของร่วมกันหลายคน,
โครงสร้างแบบ graph/tree ที่มี back-reference, หรือฟังก์ชันที่ต้องการประหยัด allocation ให้มากที่สุด

ในบทถัดไป **Part 30: Error Handling ขั้นสูง** เราจะเปลี่ยนโฟกัสไปที่การจัดการ error ในระดับที่ลึกกว่าที่เรียนไว้ตอน
`Result<T, E>` เบื้องต้นใน Part 12 มาก — คุณจะได้เรียนการออกแบบ **custom error type** ของตัวเอง, การใช้ trait
`From`/`Into` เพื่อแปลง error ระหว่างกันได้อย่างไร้รอยต่อผ่าน operator `?`, และการสร้าง **error chain** ที่เก็บ
"สาเหตุที่แท้จริง" ของ error ไว้ครบทุกชั้น ซึ่งเป็นพื้นฐานสำคัญก่อนจะไปเรียนรู้ crate `thiserror` และ `anyhow` ที่ใช้
กันจริงในโปรเจกต์ระดับ production ใน Part 31

---

**Part ก่อนหน้า:** [Smart Pointers: Rc<T> และ RefCell<T>](part-028-smart-pointers-rc-refcell.md) | **Part ถัดไป:** [Error Handling ขั้นสูง](part-030-error-handling-advanced.md)
