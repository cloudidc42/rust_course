# Part 28: Smart Pointers: Rc<T> และ RefCell<T>

> โมดูล: Smart Pointers | ระดับ: สูง | เวลาโดยประมาณ: 180 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้ว่าทำไมบางสถานการณ์ (เช่น กราฟ/ต้นไม้ที่โหนดหนึ่งถูกอ้างอิงจากหลายที่ หรือข้อมูล configuration ที่ใช้ร่วมกัน
  หลายส่วนของโปรแกรมโดยไม่รู้ล่วงหน้าว่าใครจะใช้งานมันเป็นคนสุดท้าย) ต้องการ **เจ้าของหลายคนพร้อมกันจริง ๆ** ไม่ใช่แค่
  "เจ้าของหนึ่งคน ยืมได้หลายคน" แบบที่ ownership/borrowing (Part 6-7) และ `Box<T>` (Part 27) รองรับ
- ใช้ `Rc<T>` (Reference Counted) เพื่อสร้างสถานการณ์ที่ค่าหนึ่งมีเจ้าของร่วมได้หลายคนอย่างปลอดภัย พร้อมอธิบายกลไก
  reference counting ที่อยู่เบื้องหลังได้อย่างละเอียด (`Rc::new`, `Rc::clone`, `Rc::strong_count`)
- แยกแยะได้อย่างแม่นยำว่า `Rc::clone()` เป็นการ copy "ตัวชี้ + เพิ่มตัวนับ" ที่ **ราคาถูกมาก** ไม่ใช่การ deep copy ข้อมูล
  แบบ `.clone()` ของ `String`/`Vec<T>` ที่เรียนมาจาก Part 6/13
- อธิบายได้ว่าทำไม `Rc<T>` ให้สิทธิ์แค่อ่าน (shared immutable access) เท่านั้น และทำไมสิ่งนี้จึงนำไปสู่ความต้องการ
  interior mutability
- ใช้ `RefCell<T>` เพื่อทำ **interior mutability** — แก้ไขข้อมูลได้แม้ผ่าน reference หรือ binding ที่ดูเหมือน immutable
  — โดยเข้าใจว่ากฎ borrowing เดิมจาก Part 7 (mutable หนึ่งตัว **หรือ** immutable หลายตัว ไม่ทั้งสองพร้อมกัน) ยังถูกบังคับใช้
  อยู่เหมือนเดิมทุกประการ เพียงแค่ย้ายจุดตรวจสอบจาก **compile time ไป runtime**
- อ่านและเข้าใจ panic จริงที่เกิดจากการละเมิดกฎของ `RefCell<T>` (`BorrowError`/`BorrowMutError`) พร้อมรู้วิธีจัด scope
  ของ `.borrow()`/`.borrow_mut()` ให้ถูกต้องเพื่อป้องกันมัน
- ใช้ pattern `Rc<RefCell<T>>` ซึ่งเป็น pattern ที่พบบ่อยที่สุดใน Rust สำหรับ "หลายเจ้าของที่แก้ไขข้อมูลร่วมกันได้"
  แบบไม่มี thread เข้ามาเกี่ยวข้อง และรู้ว่ามันคือ "เวอร์ชัน single-thread" ของ `Arc<Mutex<T>>` ที่จะเรียนใน Part 39
- อธิบายได้ว่า reference cycle ที่เกิดจาก `Rc<RefCell<T>>` ชี้กลับไปมาระหว่างกันคือ **memory leak ที่ปลอดภัย** (safe แต่
  รั่วจริง) หนึ่งในไม่กี่วิธีที่ Rust ยอมให้เกิดขึ้นได้ทั้งที่ compile ผ่านและไม่มี `unsafe` เลย

## ความรู้ที่ต้องมีมาก่อน

บทนี้ต่อยอดจากความรู้สะสมมาหลายบทพร้อมกัน โดยเฉพาะ:

- **Part 6 (Ownership เบื้องต้น)**: กฎที่ว่าค่าหนึ่ง ๆ มีเจ้าของได้เพียง**หนึ่งคน**ในเวลาเดียวกัน และ `.clone()` ของ
  `String`/`Vec<T>` คือการ **deep copy** ที่จองหน่วยความจำใหม่ทั้งก้อน — บทนี้จะเปรียบเทียบ `.clone()` แบบนั้นกับ
  `Rc::clone()` ที่ทำงานต่างกันโดยสิ้นเชิง แม้ชื่อ method จะเหมือนกันก็ตาม
- **Part 7 (Borrowing และ References)**: กฎเหล็ก 2 ข้อของ borrowing ที่คุณเพิ่งเรียนมาอย่างละเอียด — โดยเฉพาะ **กฎข้อที่ 1**
  ("mutable ได้แค่หนึ่งตัว หรือ immutable ได้หลายตัว แต่ไม่ทั้งสองพร้อมกัน") ที่ compiler บังคับใช้ตอน compile time
  ผ่าน borrow checker บทนี้จะแสดงให้เห็นว่า **กฎเดียวกันนี้เป๊ะ ๆ** ยังถูกบังคับใช้อยู่ใน `RefCell<T>` เพียงแค่ย้าย
  "ผู้ตรวจสอบ" จาก compiler ไปเป็น runtime counter ภายในของ `RefCell<T>` เอง — ถ้าคุณยังไม่แน่นเรื่องกฎข้อที่ 1 จาก
  Part 7 ควรกลับไปทบทวนก่อนเริ่มบทนี้ เพราะทุกอย่างในบทนี้สร้างอยู่บนความเข้าใจกฎนั้น
- **Part 27 (Smart Pointers: Box<T>)**: `Box<T>` สอนให้เรารู้จัก smart pointer ตัวแรก — วิธีเก็บข้อมูลบน heap พร้อม
  ownership แบบเดี่ยว (single ownership) เหมือน owned value ทั่วไป และ trait `Deref`/`DerefMut` ที่ทำให้ `Box<T>`
  ใช้งานได้เหมือน `T` โดยตรง บทนี้จะเริ่มต้นจากคำถามว่า **"ถ้า `Box<T>` มีเจ้าของได้แค่คนเดียว แล้วถ้าเราต้องการเจ้าของ
  มากกว่าหนึ่งคนจริง ๆ ล่ะ จะทำอย่างไร?"** ซึ่งเป็นจุดที่ `Rc<T>` เข้ามาตอบคำถามนี้โดยตรง
- ความรู้เรื่อง **struct, method (`impl`), `Option<T>`, `Result<T, E>`** จาก Part ก่อน ๆ จะถูกใช้ประกอบตัวอย่างตลอดบท
- Part นี้เพียงแค่ **แตะ** เรื่อง thread (`std::thread::spawn`) สั้น ๆ ในหัวข้อกับดัก เพื่อโยงไปยัง **Part 39
  (Mutex, Arc และ Shared-State Concurrency)** ซึ่งเป็นเวอร์ชัน "ปลอดภัยข้าม thread" ของ pattern ที่เราจะเรียนในบทนี้
  — คุณยังไม่จำเป็นต้องรู้เรื่อง thread มาก่อนเพื่ออ่านบทนี้ให้เข้าใจ

ถ้าคุณจำ**คำใบ้**ที่ Part 23 (เรื่อง lifetime ขั้นสูง) ทิ้งไว้ได้ — ที่บอกว่าบางสถานการณ์แบบ self-referential หรือ
shared-ownership ซับซ้อนเกินกว่าที่ lifetime-based borrowing ล้วน ๆ จะแก้ได้อย่างสวยงาม และแนะนำว่า `Rc`/`RefCell`
คือทางเลือกหนึ่ง — บทนี้คือคำตอบแบบเต็ม ๆ ของคำใบ้นั้น

## เนื้อหา

### 28.1 ทบทวน: เมื่อ Ownership เดี่ยวและ Borrowing ไม่พอ

จาก Part 6-7 เราเรียนมาว่า Rust มีกฎเหล็กว่า **ค่าหนึ่ง ๆ มีเจ้าของได้แค่หนึ่งคนในเวลาเดียวกัน** ส่วน "การใช้งานร่วมกัน"
ระหว่างหลายส่วนของโปรแกรมทำผ่าน **borrowing** — เจ้าของตัวจริงยังมีอยู่แค่คนเดียว ส่วนคนอื่น ๆ ได้แค่ "ยืม" reference
ไปใช้ชั่วคราว (มี lifetime ผูกอยู่กับเจ้าของเสมอ) แล้วจาก Part 27 เราเห็นว่า `Box<T>` ก็ยังอยู่ในกรอบเดียวกันนี้ —
`Box<T>` มีเจ้าของได้แค่คนเดียว เพียงแค่ข้อมูลที่มันชี้ไปอยู่บน heap แทน stack เท่านั้นเอง

ระบบนี้ทำงานได้ดีมากในสถานการณ์ส่วนใหญ่ เพราะโปรแกรมจริงจำนวนมากมี "โครงสร้างความเป็นเจ้าของ" ที่ชัดเจนแบบต้นไม้
(tree-like ownership hierarchy) — เช่น `struct House` เป็นเจ้าของ `Vec<Room>` ซึ่งแต่ละ `Room` เป็นเจ้าของ
`Vec<Furniture>` ของตัวเอง ไม่มีใครแบ่งกันใช้ ทุกอย่างมีเจ้าของเดียวชัดเจนตลอดสาย

แต่ในโลกจริงมีสถานการณ์ที่ **ไม่ได้เป็นต้นไม้แบบนั้น** — บางค่าจำเป็นต้องมี**เจ้าของหลายคนพร้อมกันจริง ๆ** โดยที่
"เจ้าของ" ทุกคนมีสิทธิ์เท่ากัน ไม่มีใครเป็น "เจ้าตัวจริง" ที่คนอื่นแค่ยืม และที่สำคัญคือ **เราไม่รู้ล่วงหน้าตอน compile
time ว่าใครจะเป็นคนสุดท้ายที่ยังใช้ค่านั้นอยู่** — ถ้ารู้ล่วงหน้า เราจะใช้ lifetime-based borrowing (Part 20) ได้สบาย ๆ
แต่ถ้าไม่รู้ (เพราะขึ้นกับ logic ตอน runtime เช่น ผู้ใช้กดปุ่มไหนก่อน หรือ thread ไหนจบงานก่อน) lifetime ธรรมดาจะ
เขียนไม่ได้เลย เพราะ compiler ต้องพิสูจน์ lifetime ได้อย่างแน่นอนตั้งแต่ compile time

### 28.2 ปัญหาที่ต้องการเจ้าของร่วมอย่างแท้จริง

ลองดูตัวอย่างที่เป็นรูปธรรม: ระบบที่มี object `Config` ก้อนเดียว (การตั้งค่าของระบบ) ที่ต้องถูกใช้โดย `Server`
หลายตัวพร้อมกัน — `Server` แต่ละตัวไม่ใช่ "เจ้าของแค่คนเดียวที่คนอื่นยืม" แต่ทุกตัว**ควรเป็นเจ้าของ `Config` เท่า ๆ กัน**
เพราะไม่มีใครรู้ว่า server ตัวไหนจะถูก drop ก่อน — ถ้า `Server` ตัวหนึ่งตายไปแต่ตัวอื่นยังใช้ `Config` อยู่
`Config` ก็ต้องยังมีชีวิตอยู่ต่อ และจะถูกทำลายจริง ๆ ก็ต่อเมื่อ**ไม่มี `Server` ตัวใดใช้งานมันอีกแล้วเท่านั้น**

#### ทำไม lifetime-based borrowing (Part 20) ไม่พอ

ก่อนจะลองใช้ `Box<T>` ลองมาดูก่อนว่าทำไมเราไม่ใช้ **reference ธรรมดาพร้อม lifetime** (`&'a Config`) จาก Part 7/20
แก้ปัญหานี้ไปเลย — ในเมื่อ `Server` แค่ต้องการ "อ่าน" `Config` เท่านั้น ฟังดูเหมือนน่าจะพอ:

```rust
struct Config {
    max_connections: u32,
}

struct Server<'a> {
    name: String,
    config: &'a Config,
}

fn create_servers(config: &Config) -> Vec<Server> {
    vec![
        Server { name: "server-1".to_string(), config },
        Server { name: "server-2".to_string(), config },
    ]
}

fn main() {
    let servers;
    {
        let config = Config { max_connections: 100 };
        servers = create_servers(&config);
    } // config หมด scope ตรงนี้ แต่ servers ยังถือ &Config ไปยังมันอยู่!

    println!("{}", servers[0].name);
}
```

โค้ดนี้ **compile ไม่ผ่าน**:

```
error[E0597]: `config` does not live long enough
  --> src/main.rs:21:34
   |
20 |         let config = Config { max_connections: 100 };
   |             ------ binding `config` declared here
21 |         servers = create_servers(&config);
   |                                  ^^^^^^^ borrowed value does not live long enough
22 |     }
   |     - `config` dropped here while still borrowed
23 |
24 |     println!("{}", servers[0].name);
   |                    ------- borrow later used here
```

นี่คือ `E0597` ตัวเดียวกับที่เจอครั้งแรกใน Part 7 (dangling reference ระดับ block scope) — ปัญหาคือ `config` เป็น
ตัวแปร local ที่มีชีวิตอยู่แค่ในบล็อก `{}` เท่านั้น แต่ `servers` (ที่ `create_servers` คืนออกมา) ยังต้องมีชีวิตอยู่
**ต่อไปหลังจากนั้น** เพื่อพิมพ์ `servers[0].name` — lifetime ของ `servers` (ซึ่งถือ `&Config` อยู่ภายใน) ต้อง
"สั้นกว่าหรือเท่ากับ" lifetime ของ `config` เสมอตามกฎ borrowing แต่โค้ดนี้ต้องการให้ `servers` มีชีวิตอยู่**นานกว่า**
`config` ซึ่งเป็นไปไม่ได้เลยด้วยกฎ borrowing ล้วน ๆ — compiler พิสูจน์ไม่ได้แน่นอนตอน compile time ว่า
lifetime จะเรียงลำดับกันถูกต้อง เพราะในความเป็นจริงมันเรียงลำดับผิดจริง ๆ

นี่คือแก่นของปัญหาที่กล่าวไว้ตอนต้นบท: **เราไม่รู้ล่วงหน้าว่าใครจะเป็นคนใช้ `config` เป็นคนสุดท้าย** — ถ้าการ
ออกแบบโปรแกรมทำให้ `Server` ที่ยืม `Config` มา "มีชีวิตยาวกว่า" ตัวแปร `Config` ต้นฉบับ (เช่นถูกส่งกลับออกจาก
ฟังก์ชัน หรือถูกเก็บไว้ใน collection ที่มีอายุยาวกว่า scope ที่สร้าง `Config`) lifetime-based borrowing จะไม่มีทาง
compile ผ่านได้เลย ไม่ว่าจะพยายามเขียน lifetime annotation ยังไงก็ตาม เพราะมันสะท้อนความจริงที่ขัดแย้งกันเอง:
`Config` ต้อง**ตายก่อน** ตาม scope ที่เขียนไว้ แต่ก็ต้อง**มีชีวิตอยู่นานกว่า** ผู้ที่ยืมมันไปด้วยในเวลาเดียวกัน

ทางแก้ที่ถูกต้องคือเปลี่ยนคำถามจาก "ใครยืมข้อมูลจากใคร (และใครมีชีวิตยาวกว่า)" เป็น **"ให้ทุกคนเป็นเจ้าของร่วมกัน
เลย โดยไม่มีใครต้องพึ่ง lifetime ของใคร"** — นี่คือสิ่งที่ `Rc<T>` ตอบโจทย์ได้ตรงที่สุด เพราะข้อมูลจะมีชีวิตอยู่
ตราบใดที่ยังมีเจ้าของเหลืออยู่อย่างน้อยหนึ่งคน ไม่ขึ้นกับ scope ของใครคนใดคนหนึ่งอีกต่อไป

#### ลองใช้ Box<T>: เจ้าของเดี่ยวก็ยังไม่พอ

ทีนี้ลองเปลี่ยนมาใช้ `Box<T>` (ความรู้จาก Part 27) แบบตรงไปตรงมาดูบ้าง เพื่อดูว่าปัญหาคืออะไรเมื่อเปลี่ยนจาก
"ยืม" มาเป็น "เป็นเจ้าของ" แทน:

```rust
struct Config {
    max_connections: u32,
    timeout_secs: u32,
}

struct Server {
    name: String,
    config: Box<Config>,
}

fn main() {
    let config = Box::new(Config { max_connections: 100, timeout_secs: 30 });

    let server1 = Server { name: String::from("server-1"), config };
    let server2 = Server { name: String::from("server-2"), config };

    println!("{} กับ {}", server1.name, server2.name);
}
```

โค้ดนี้ **compile ไม่ผ่าน** ด้วย error ที่เราคุ้นเคยจาก Part 6 อยู่แล้ว:

```
error[E0382]: use of moved value: `config`
  --> src/main.rs:15:60
   |
12 |     let config = Box::new(Config { max_connections: 100, timeout_secs: 30 });
   |         ------ move occurs because `config` has type `Box<Config>`, which does not implement the `Copy` trait
13 |
14 |     let server1 = Server { name: String::from("server-1"), config };
   |                                                            ------ value moved here
15 |     let server2 = Server { name: String::from("server-2"), config };
   |                                                            ^^^^^^ value used here after move
   |
note: if `Config` implemented `Clone`, you could clone the value
  --> src/main.rs:1:1
   |
 1 | struct Config {
   | ^^^^^^^^^^^^^ consider implementing `Clone` for this type
```

เหตุผลตรงไปตรงมามาก: `Box<Config>` คือ owned value เดี่ยว ๆ ตัวหนึ่ง — พอ `server1` รับ `config` เข้าไปในฟิลด์ของมัน
ownership ของ `config` ก็ถูก **move** เข้าไปอยู่ใน `server1` ทั้งหมด ตัวแปร `config` เดิมจึงใช้ต่อไม่ได้อีกเลย
ตรงตามกฎ ownership ข้อแรกที่เราเรียนมาตั้งแต่ Part 6 ทุกประการ — `Box<T>` ไม่มีทางฝ่าฝืนกฎนี้ได้ เพราะมันคือ owned
value ปกติ เพียงแค่ข้อมูลอยู่บน heap เท่านั้น

compiler เองก็แนะนำ hint ไว้ว่า "ถ้า `Config` implement `Clone` คุณสามารถ clone ค่านั้นได้" — แต่ลองคิดดูดี ๆ ว่า
`.clone()` แบบ deep copy จะแก้ปัญหาได้จริงไหม ถ้าเราให้ `server1` และ `server2` ถือ **`Config` คนละก้อนที่หน้าตา
เหมือนกัน** (deep copy สองชุดแยกกันจริง ๆ บน heap):

```rust
#[derive(Clone)]
struct Config {
    max_connections: u32,
    timeout_secs: u32,
}

struct Server {
    name: String,
    config: Config, // ไม่ต้องใช้ Box ด้วยซ้ำ เพราะ deep clone แล้ว
}

fn main() {
    let config = Config { max_connections: 100, timeout_secs: 30 };

    let server1 = Server { name: String::from("server-1"), config: config.clone() };
    let server2 = Server { name: String::from("server-2"), config: config.clone() }; // clone อีกชุด

    println!("{}: max={}", server1.name, server1.config.max_connections);
    println!("{}: max={}", server2.name, server2.config.max_connections);
}
```

โค้ดนี้ compile ผ่านและทำงานถูกต้อง — แต่มัน **ไม่ตรงกับความต้องการจริงเลย** ในหลายสถานการณ์: ถ้าระบบต้องเปลี่ยนค่า
`max_connections` ตอน runtime (เช่น ผู้ดูแลระบบสั่งลดจำนวน connection สูงสุดกลางอากาศ) การแก้ไข `config` ของ `server1`
จะ**ไม่มีผลกับ `server2` เลย** เพราะทั้งสองถือข้อมูลคนละก้อนกันอย่างสิ้นเชิงหลัง deep clone แล้ว นี่คือบั๊กทาง logic
ที่ร้ายแรง — เราต้องการให้ทั้งสองมองเห็น "config ตัวเดียวกัน" จริง ๆ ไม่ใช่ "สำเนาที่เหมือนกันตอนสร้าง แต่แยกกันตอนแก้ไข"

นี่คือปัญหาที่แท้จริงที่บทนี้จะแก้: **เราต้องการค่าหนึ่งก้อนที่มีเจ้าของร่วมกันหลายคน โดยที่ทุกคนมองเห็นข้อมูล
ก้อนเดียวกันจริง ๆ (ไม่ใช่สำเนา) และค่านั้นจะถูกทำลายก็ต่อเมื่อไม่มีเจ้าของเหลืออยู่เลยสักคน** — นี่คือสิ่งที่
`Rc<T>` (ตัวย่อของ "Reference Counted") ถูกออกแบบมาเพื่อแก้โดยตรง

#### ตัวอย่างที่สอง: กราฟ/ต้นไม้ที่โหนดหนึ่งมีผู้ปกครองมากกว่าหนึ่งคน

ปัญหา "ต้องการเจ้าของร่วม" ไม่ได้เกิดขึ้นแค่กับ configuration เท่านั้น — มันเกิดขึ้นบ่อยมากในโครงสร้างข้อมูลแบบ
กราฟหรือต้นไม้ที่**ไม่ใช่ต้นไม้บริสุทธิ์** (pure tree ที่ทุกโหนดมีผู้ปกครองแค่หนึ่งคน) ลองนึกภาพระบบจัดการทีมงาน
ในบริษัท ที่พนักงานคนหนึ่งอาจถูก "ยืมตัว" ไปทำงานในหลายทีมพร้อมกันจริง ๆ (เช่น ผู้เชี่ยวชาญด้านฐานข้อมูลที่ถูกดึง
ไปช่วยทั้งทีม Backend และทีม Data Platform ในเวลาเดียวกัน) — พนักงานคนนี้ไม่ได้ "เป็นของ" ทีมใดทีมหนึ่งแต่เพียงผู้
เดียว แต่ถูก**อ้างอิงจากหลายทีมพร้อมกันจริง ๆ** เหมือนโหนดหนึ่งในกราฟที่มีเส้นเชื่อมเข้ามาจาก "ผู้ปกครอง" มากกว่า
หนึ่งคน:

```rust
use std::rc::Rc;

struct Employee {
    name: String,
    role: String,
}

struct Team {
    team_name: String,
    members: Vec<Rc<Employee>>,
}

fn main() {
    let specialist = Rc::new(Employee {
        name: "คุณนิดา".to_string(),
        role: "Database Specialist".to_string(),
    });
    println!("strong_count ตอนสร้าง specialist = {}", Rc::strong_count(&specialist));

    let backend_team = Team {
        team_name: "Backend Team".to_string(),
        members: vec![Rc::clone(&specialist)],
    };
    println!(
        "strong_count หลัง Backend Team ยืมตัว specialist = {}",
        Rc::strong_count(&specialist)
    );

    let data_team = Team {
        team_name: "Data Platform Team".to_string(),
        members: vec![Rc::clone(&specialist)],
    };
    println!(
        "strong_count หลัง Data Platform Team ยืมตัว specialist ด้วย = {}",
        Rc::strong_count(&specialist)
    );

    for team in [&backend_team, &data_team] {
        for member in &team.members {
            println!("{}: มี {} ({})", team.team_name, member.name, member.role);
        }
    }

    drop(backend_team);
    println!(
        "strong_count หลัง Backend Team ยุบทีม = {}",
        Rc::strong_count(&specialist)
    );
}
```

ผลลัพธ์ (รันจริง):

```
strong_count ตอนสร้าง specialist = 1
strong_count หลัง Backend Team ยืมตัว specialist = 2
strong_count หลัง Data Platform Team ยืมตัว specialist ด้วย = 3
Backend Team: มี คุณนิดา (Database Specialist)
Data Platform Team: มี คุณนิดา (Database Specialist)
strong_count หลัง Backend Team ยุบทีม = 2
```

จุดที่ต้องสังเกตให้ชัด: `backend_team` และ `data_team` **ไม่มีความสัมพันธ์ทาง ownership กันเลยโดยตรง** — ไม่มี
ใครเป็น "เจ้าของ" อีกฝั่ง แต่ทั้งคู่ต่างก็เป็นเจ้าของ `specialist` (`Employee`) คนเดียวกันร่วมกันอย่างถูกต้องตามกฎ
ผ่าน `Rc::clone` เมื่อ `backend_team` ถูกยุบ (`drop(backend_team)`) `specialist` **ไม่ได้ถูกทำลายไปด้วย** เพราะ
`data_team` ยังอ้างอิงถึงอยู่ — ตัวนับแค่ลดลงจาก 3 เหลือ 2 เท่านั้น ตรงตามสัญชาตญาณที่ถูกต้องของ "หลายผู้ปกครอง
ใช้ทรัพยากรร่วมกัน": ทรัพยากรจะหายไปก็ต่อเมื่อผู้ปกครองทุกคนเลิกใช้งานมันแล้วเท่านั้น ไม่ใช่ทันทีที่ผู้ปกครองคนแรก
เลิกใช้

นี่คือรูปแบบเดียวกันเป๊ะ ๆ กับตัวอย่าง `Config`/`Server` ก่อนหน้า เพียงแค่เปลี่ยนบริบทให้เห็นภาพแบบ "กราฟที่มีหลาย
ผู้ปกครอง" ชัดเจนขึ้น — ทั้งสองตัวอย่างคือรูปแบบเดียวกันของปัญหาเดียวกัน: **ค่าหนึ่งก้อนที่ต้องมีเจ้าของมากกว่าหนึ่ง
คนพร้อมกันจริง ๆ โดยไม่รู้ล่วงหน้าว่าใครจะเลิกใช้เป็นคนสุดท้าย**

### 28.3 Rc<T>: Reference Counted Smart Pointer

`Rc<T>` (`std::rc::Rc`) คือ smart pointer ที่เก็บข้อมูล type `T` ไว้บน heap เหมือน `Box<T>` แต่มีความแตกต่างสำคัญ
ที่สุดคือ: **`Rc<T>` อนุญาตให้มีเจ้าของหลายคนพร้อมกันได้** โดยภายในมันเก็บ **ตัวนับ (counter)** ไว้ด้วยว่า ณ ขณะนี้
มี "เจ้าของ" กี่คนกำลังถืออยู่ ทุกครั้งที่มีการ `Rc::clone()` ตัวนับนี้จะเพิ่มขึ้นหนึ่ง และทุกครั้งที่เจ้าของตัวหนึ่ง
หมด scope (ถูก drop) ตัวนับจะลดลงหนึ่ง — ข้อมูลจริงบน heap จะถูกทำลายก็ต่อเมื่อ **ตัวนับลดลงมาถึง 0** เท่านั้น
(หมายความว่าไม่มีเจ้าของคนไหนเหลืออยู่เลย)

ชื่อ "Rc" ย่อมาจาก **Reference Counted** ตรงตามหลักการนี้เป๊ะ ๆ — ชื่อของมันบอกความหมายของมันตรงตัวที่สุด

มาแก้ตัวอย่าง `Config`/`Server` ด้วย `Rc<T>`:

```rust
use std::rc::Rc;

struct Config {
    max_connections: u32,
    timeout_secs: u32,
}

struct Server {
    name: String,
    config: Rc<Config>, // เปลี่ยนจาก Box<Config> เป็น Rc<Config>
}

fn main() {
    let config = Rc::new(Config { max_connections: 100, timeout_secs: 30 });
    println!("strong_count หลังสร้าง config = {}", Rc::strong_count(&config));

    let server1 = Server { name: "server-1".to_string(), config: Rc::clone(&config) };
    println!("strong_count หลัง server1 ถือ config = {}", Rc::strong_count(&config));

    let server2 = Server { name: "server-2".to_string(), config: Rc::clone(&config) };
    println!("strong_count หลัง server2 ถือ config = {}", Rc::strong_count(&config));

    println!(
        "{}: max_connections={}, timeout={}",
        server1.name, server1.config.max_connections, server1.config.timeout_secs
    );
    println!(
        "{}: max_connections={}, timeout={}",
        server2.name, server2.config.max_connections, server2.config.timeout_secs
    );

    drop(server1);
    println!("strong_count หลัง drop server1 = {}", Rc::strong_count(&config));

    drop(server2);
    println!("strong_count หลัง drop server2 = {}", Rc::strong_count(&config));
}
```

ผลลัพธ์ (รันจริง):

```
strong_count หลังสร้าง config = 1
strong_count หลัง server1 ถือ config = 2
strong_count หลัง server2 ถือ config = 3
server-1: max_connections=100, timeout=30
server-2: max_connections=100, timeout=30
strong_count หลัง drop server1 = 2
strong_count หลัง drop server2 = 1
```

มาไล่ดูทีละจุดว่าเกิดอะไรขึ้น:

1. **`Rc::new(Config { ... })`** — สร้างค่า `Config` บน heap พร้อมกับ **ตัวนับ (strong count)** เริ่มต้นที่ 1
   (ตัวแปร `config` เป็นเจ้าของคนแรกและคนเดียวตอนนี้)
2. **`Rc::clone(&config)`** — นี่คือหัวใจสำคัญของหัวข้อนี้: `Rc::clone` **ไม่ได้สร้างข้อมูล `Config` ใหม่บน heap
   เลย** มันแค่ (1) copy ตัวชี้ (pointer) ที่ชี้ไปยัง `Config` เดิมก้อนเดียวกัน และ (2) เพิ่มตัวนับขึ้นหนึ่ง —
   นั่นคือทำไม `Rc::strong_count` เพิ่มจาก 1 เป็น 2 เป็น 3 ทุกครั้งที่มี `Server` ตัวใหม่มาถือ `config`
3. **`server1.config` และ `server2.config` ชี้ไปที่ `Config` ก้อนเดียวกันจริง ๆ** — ถ้าใครแก้ไขค่าได้ (ยังไม่ได้ในตอนนี้
   เพราะ `Rc<T>` ให้แค่ immutable access ตามหัวข้อ 28.5 ที่จะเห็นต่อไป) ทุกคนจะเห็นการเปลี่ยนแปลงนั้นเหมือนกัน
4. **`drop(server1)`** — เมื่อ `server1` ถูกทำลาย ฟิลด์ `config: Rc<Config>` ของมันก็ถูกทำลายไปด้วย ซึ่งหมายความว่า
   ตัวนับของ `Rc<Config>` **ลดลงหนึ่ง** (จาก 3 เหลือ 2) — แต่ **ข้อมูล `Config` จริงบน heap ยังไม่ถูกทำลาย** เพราะยังมี
   เจ้าของเหลืออยู่ 2 คน (ตัวแปร `config` ใน `main` และ `server2`)
5. **`drop(server2)`** — ตัวนับลดลงอีกหนึ่ง (เหลือ 1) — ยังไม่ถูกทำลาย เพราะตัวแปร `config` ใน `main` ยังถือมันอยู่
6. เมื่อ `main` จบ (จบบรรทัดสุดท้ายของฟังก์ชัน) ตัวแปร `config` ก็หมด scope เช่นกัน ตัวนับจะลดลงเป็น **0**
   ถึงตอนนั้น `Config` บน heap จะถูก drop จริง ๆ

จะสังเกตได้ว่า **`Rc<T>` ไม่ได้ทำลายกฎ ownership จาก Part 6 เลย** มันแค่ขยายความหมายของ "เจ้าของ" จาก "หนึ่งคนเท่านั้น"
เป็น "หนึ่งกลุ่มคนที่ถือ `Rc<T>` ที่ชี้ไปยังข้อมูลเดียวกัน" — เมื่อสมาชิกกลุ่มลดลงเหลือ 0 คน กฎ "เจ้าของหมด scope
แล้วค่าถูก drop" (กฎข้อ 3 จาก Part 6) ก็ยังถูกบังคับใช้อยู่เหมือนเดิม เพียงแค่ "เจ้าของ" ในที่นี้ไม่ได้นับเป็นคนเดียว
แต่นับทั้งกลุ่มพร้อมกัน

#### ภาพในหัวสำหรับ Rc<T>: heap allocation ที่มีตัวนับติดอยู่

เพื่อให้เห็นภาพชัดเจนว่าทำไม `Rc::clone` ถึงราคาถูกและทำไมทุก `Rc` ที่ clone มาจากกันถึงชี้ไปที่ตำแหน่งเดียวกัน
เป๊ะ ๆ ลองนึกภาพ memory layout ของ `Rc<Config>` ในตัวอย่างข้างบน (เทียบกับภาพ memory layout ของ `String` และ
reference ที่เคยเห็นใน Part 7):

```
                     heap allocation ก้อนเดียว (จัดสรรครั้งเดียวตอน Rc::new)
                     ┌─────────────────────────────────────┐
                     │ strong_count: 3   (จำนวนเจ้าของ)     │
                     │ weak_count:   0   (จะเจอใน Part 29)  │
                     │ value: Config { max_connections:100 }│
                     └─────────────────────────────────────┘
                                ▲       ▲       ▲
                                │       │       │
   ตัวแปร config (บน stack) ────┘       │       │
   server1.config (บน stack) ───────────┘       │
   server2.config (บน stack) ───────────────────┘
```

สังเกตว่า `Rc<Config>` แต่ละตัว (`config`, `server1.config`, `server2.config`) เป็นแค่ **ตัวชี้เล็ก ๆ บน stack**
ที่ชี้ไปยัง heap allocation **ก้อนเดียวกัน** ก้อนนั้นเก็บ 3 อย่างรวมกัน: (1) `strong_count` ตัวเลขบอกจำนวนเจ้าของ
ที่ยังมีชีวิตอยู่ (2) `weak_count` ตัวเลขอีกตัวที่เกี่ยวกับ `Weak<T>` ซึ่งจะเจอใน Part 29 (ตอนนี้เป็น 0 เสมอถ้ายัง
ไม่ได้ใช้ `Weak`) และ (3) `value` คือข้อมูล `Config` จริงที่ทุกคนแบ่งกันใช้ — เมื่อเรียก `Rc::clone()` สิ่งที่
เกิดขึ้นคือ **สร้างตัวชี้ใหม่อีกตัวบน stack ที่ชี้ไปยัง heap allocation ก้อนเดิม** พร้อมกับเพิ่มเลข
`strong_count` ขึ้นหนึ่ง — ไม่มีการจัดสรร heap ใหม่เลยแม้แต่ byte เดียว ต่างจาก `.clone()` ของ `String` ที่ต้อง
จัดสรร heap ก้อนใหม่ทั้งก้อนอย่างสิ้นเชิงตามภาพที่เคยเห็นใน Part 7

#### `Rc::clone(&x)` เขียนแบบไหนก็ได้ แต่มีความหมายต่างกัน

ในทางเทคนิค คุณสามารถเขียน `config.clone()` (แบบ method call ผ่าน `.`) แทน `Rc::clone(&config)` ได้เหมือนกัน เพราะ
`Rc<T>` implement trait `Clone` และ auto-deref/method resolution จะหา method `clone` เจอทั้งสองแบบ ผลลัพธ์ที่ได้
เหมือนกันทุกประการ — แต่**สาย Rust ที่เขียนโค้ดแบบ idiomatic นิยมเขียน `Rc::clone(&config)` แบบ associated function
เสมอ** เพื่อสื่อความหมายให้ชัดเจนต่อคนอ่านโค้ดว่า **"นี่คือการเพิ่มตัวนับ ไม่ใช่การ deep copy"** — ถ้าเขียน
`config.clone()` เฉย ๆ คนอ่านโค้ด (โดยเฉพาะคนที่ยังไม่คุ้นกับ `Rc`) อาจเข้าใจผิดว่ากำลัง deep copy ข้อมูลทั้งก้อน
เหมือน `.clone()` ของ `String`/`Vec<T>` ทั่วไป ซึ่งนำไปสู่ความสับสนที่หัวข้อถัดไปจะอธิบายอย่างละเอียด

#### เปรียบเทียบ Reference Counting ของ Rust กับภาษาอื่น

แนวคิด "นับจำนวนผู้อ้างอิง แล้วทำลายข้อมูลเมื่อนับถึง 0" ไม่ใช่เรื่องใหม่ในวงการโปรแกรมมิ่งเลย หลายภาษาใช้แนวคิด
นี้เป็นส่วนหนึ่งของการจัดการหน่วยความจำมานานแล้ว แต่ Rust มีจุดที่ต่างจากภาษาอื่นอย่างชัดเจน:

| ภาษา | กลไก reference counting | ใครควบคุมว่าใช้ตอนไหน | ป้องกัน reference cycle ได้หรือไม่ |
|---|---|---|---|
| **C++** | `std::shared_ptr<T>` (ตัวนับเป็น atomic โดย default เพื่อความปลอดภัยข้าม thread) | ผู้เขียนโค้ดเลือกใช้เอง (เหมือน `Rc`/`Arc` ของ Rust) แทน raw pointer หรือ `unique_ptr` | ไม่ป้องกัน — ต้องใช้ `std::weak_ptr` เองเพื่อตัด cycle เหมือน `Weak<T>` ของ Rust |
| **Python** | ทุก object ถูกนับ reference โดยอัตโนมัติเสมอ (ไม่มีทางเลือกอื่น) | ไม่มีทางเลือก — ทุก object ทำงานแบบนี้เป็น default ของภาษา | **ป้องกันได้บางส่วน** — มี cycle detector (garbage collector เสริม) ทำงานเป็นระยะเพื่อตาม "เก็บกวาด" cycle ที่หลุดจาก reference counting ธรรมดา |
| **Swift** | ARC (Automatic Reference Counting) — compiler แทรกโค้ดเพิ่ม/ลดตัวนับให้อัตโนมัติทุกจุด | อัตโนมัติทั้งหมด ผู้เขียนไม่ต้องเรียก clone/retain เอง แต่ต้องรู้จัก `weak`/`unowned` เพื่อตัด cycle เอง | ไม่ป้องกันอัตโนมัติ — ต้องประกาศ `weak var` เองเหมือนต้องเลือกใช้ `Weak<T>` |
| **Java / Go / C#** | ไม่ใช้ reference counting เลย ใช้ **garbage collector แบบ tracing** (ไล่ตาม object graph ทั้งหมดเป็นระยะ) | อัตโนมัติทั้งหมด ไม่มี concept ของ "ตัวนับ" ให้เห็นในโค้ดเลย | **ป้องกันได้เต็มรูปแบบ** — tracing GC มองเห็น cycle ได้ตามธรรมชาติ เพราะไม่ได้ตัดสินใจจาก "ตัวนับ" แต่ไล่ดูว่ามีใครเข้าถึง object นั้นได้จาก root จริงหรือไม่ |
| **Rust** | `Rc<T>` (single-thread) / `Arc<T>` (multi-thread, Part 39) — ตัวนับธรรมดา ไม่มี background thread ใดมาช่วยตรวจ cycle เลย | ผู้เขียนโค้ดเลือกใช้เองอย่างชัดเจน (ต้อง `use` และเรียก `Rc::new`/`Rc::clone` เอง) | **ไม่ป้องกันเลย** — เหมือน C++/Swift ต้องใช้ `Weak<T>` (Part 29) ตัด cycle ด้วยตัวเอง ไม่มี background GC มาช่วย |

จุดที่ทำให้ Rust ต่างจากภาษาที่มี garbage collector (Java/Go/Python) อย่างมีนัยสำคัญคือ **Rust ไม่มี runtime ที่
วิ่งอยู่เบื้องหลังคอยไล่ตรวจ object graph เป็นระยะเลย** — การเพิ่ม/ลดตัวนับของ `Rc<T>` เกิดขึ้น ณ จุดที่ `clone`/`drop`
ถูกเรียกเท่านั้น (deterministic เต็มที่ ไม่มี "หยุดโลกกะทันหัน" แบบที่ garbage collector บางแบบทำ ที่เรียกว่า "GC
pause") นี่คือเหตุผลที่ Rust ถูกเลือกใช้ในงานที่ต้องการ latency ต่ำและคาดเดาได้แน่นอน (เช่น เกม, ระบบ real-time,
embedded) แต่ก็ต้องแลกมาด้วยการที่ผู้เขียนโค้ดต้องรับผิดชอบเรื่อง reference cycle ด้วยตัวเองเต็ม ๆ ไม่มีใครมา
"เก็บกวาด" ให้แบบที่ Python ทำได้บางส่วน

### 28.4 Rc::clone ไม่ใช่ Deep Clone: ข้อแตกต่างที่สำคัญที่สุดในบทนี้

นี่คือจุดที่มือใหม่สับสนบ่อยที่สุดเมื่อเริ่มใช้ `Rc<T>` — **ชื่อ method เหมือนกัน (`clone`) แต่ความหมายและ cost
ต่างกันโดยสิ้นเชิง** ขึ้นอยู่กับว่าเรียกกับ type ไหน:

```rust
use std::rc::Rc;

fn main() {
    // .clone() บน String: deep copy — จองหน่วยความจำใหม่บน heap อีกก้อน แล้ว copy ตัวอักษรทั้งหมดไปไว้
    let s1 = String::from("hello");
    let s2 = s1.clone();
    println!("s1 = {s1}, s2 = {s2}");
    println!("s1 address: {:p}, s2 address: {:p}", s1.as_ptr(), s2.as_ptr());

    // Rc::clone บน Rc<String>: copy แค่ตัวชี้ + เพิ่มตัวนับ — ไม่จองหน่วยความจำใหม่เลย
    let a = Rc::new(String::from("hello"));
    let b = Rc::clone(&a);
    println!("a = {a}, b = {b}");
    println!("a address: {:p}, b address: {:p}", Rc::as_ptr(&a), Rc::as_ptr(&b));
}
```

ผลลัพธ์ (รันจริง — ตำแหน่ง address อาจเปลี่ยนไปในแต่ละครั้งที่รัน แต่ **รูปแบบความสัมพันธ์จะเหมือนกันเสมอ**):

```
s1 = hello, s2 = hello
s1 address: 0x55d8bb1fad60, s2 address: 0x55d8bb1fad80
a = hello, b = hello
a address: 0x55d8bb1faaf0, b address: 0x55d8bb1faaf0
```

สังเกตความแตกต่างที่ชี้ชัดที่สุด: **`s1.as_ptr()` และ `s2.as_ptr()` เป็น address คนละตำแหน่งกัน** (`...ad60` vs
`...ad80`) เพราะ `.clone()` ของ `String` จองหน่วยความจำใหม่แยกก้อนจริง ๆ บน heap — แก้ไข `s2` จะไม่กระทบ `s1` เลย
แม้แต่นิดเดียว ในทางกลับกัน **`Rc::as_ptr(&a)` และ `Rc::as_ptr(&b)` เป็น address ตำแหน่งเดียวกันเป๊ะ ๆ**
(`...aaf0` ทั้งคู่) เพราะ `Rc::clone(&a)` ไม่ได้จองหน่วยความจำใหม่เลย — มันแค่สร้างตัวชี้อีกตัวที่ชี้ไปยัง `String`
เดิมก้อนเดียวกัน พร้อมเพิ่มตัวนับขึ้นหนึ่ง

สรุปเป็นตารางเปรียบเทียบให้เห็นภาพชัดที่สุด:

| ลักษณะ | `.clone()` บน `String`/`Vec<T>` (Part 6/13) | `Rc::clone(&x)` บน `Rc<T>` |
|---|---|---|
| จองหน่วยความจำใหม่บน heap หรือไม่ | **จอง** — ก้อนใหม่แยกจากเดิมสมบูรณ์ | **ไม่จอง** — ชี้ไปที่ก้อนเดิมเป๊ะ ๆ |
| Cost (ความเร็ว) | ขึ้นกับขนาดข้อมูล (ยิ่งข้อมูลใหญ่ ยิ่งช้า) | คงที่เสมอ (O(1)) — แค่ copy pointer + เพิ่มตัวเลขหนึ่งตัว |
| แก้ไขสำเนาหนึ่งกระทบอีกสำเนาหรือไม่ | **ไม่กระทบ** — เป็นข้อมูลคนละก้อนกันจริง ๆ | **กระทบ** (ถ้าแก้ไขได้ ผ่าน `RefCell` ในหัวข้อถัดไป) เพราะชี้ไปที่ก้อนเดียวกัน |
| สิ่งที่เพิ่มขึ้นหลัง clone | ก้อนข้อมูลใหม่บน heap | แค่ตัวเลข "จำนวนเจ้าของ" (`strong_count`) |

**นี่คือเหตุผลที่ต้องระวังให้มากเป็นพิเศษ**: ถ้าคุณเห็นโค้ดที่มี `.clone()` เกลื่อนไปทั่ว แล้วสมมติไปเลยว่า "โค้ดนี้
ต้อง slow เพราะ clone เยอะ" — คุณอาจเข้าใจผิดถ้าตัวแปรเหล่านั้นเป็น `Rc<T>` เพราะ `Rc::clone` ราคาถูกมาก (แค่เพิ่ม
เลขจำนวนเต็มหนึ่งตัวและ copy pointer) ในทางกลับกัน ถ้าคุณคาดหวังว่าการ `.clone()` จะได้สำเนาที่ **แก้ไขแยกจากต้นฉบับ
ได้อย่างอิสระ** แล้วดันเป็น `Rc<T>` คุณจะพบว่าการแก้ไขผ่านสำเนาหนึ่งกระทบอีกสำเนาด้วยเสมอ (ดูตัวอย่างในหัวข้อ
"กับดักที่พบบ่อย" ข้อ 1 ท้ายบทที่จะสาธิตความสับสนนี้ให้เห็นภาพชัดเจนขึ้นอีก)

### 28.5 ข้อจำกัดของ Rc<T>: ให้แค่ Shared Immutable Access

ทีนี้มาถึงคำถามสำคัญ: ในตัวอย่าง `Config`/`Server` ข้างบน ถ้าเราต้องการ **แก้ไข** `max_connections` ผ่าน `server1`
แล้วให้ `server2` มองเห็นการเปลี่ยนแปลงนั้นด้วย (เพราะทั้งคู่ถือ `Rc` ที่ชี้ไปยังก้อนเดียวกัน) จะทำได้ไหม?

ลองดูสิ่งที่เกิดขึ้นเมื่อพยายามแก้ไขค่าผ่าน `Rc<T>` โดยตรง:

```rust
use std::rc::Rc;

fn main() {
    let shared = Rc::new(5);
    *shared += 1;
    println!("{shared}");
}
```

โค้ดนี้ **compile ไม่ผ่าน**:

```
error[E0594]: cannot assign to data in an `Rc`
 --> src/main.rs:5:5
  |
5 |     *shared += 1;
  |     ^^^^^^^^^^^^ cannot assign
  |
  = help: trait `DerefMut` is required to modify through a dereference, but it is not implemented for `Rc<i32>`
```

error message บอกเหตุผลตรงตัวมาก: **`Rc<T>` implement เฉพาะ trait `Deref` (อ่านได้อย่างเดียว) แต่ไม่ implement
`DerefMut` (แก้ไขได้)** — จำได้จาก Part 27 ว่า `Deref` คือสิ่งที่ทำให้ `Box<T>` ใช้งานเหมือน `T` ได้โดยตรง ส่วน
`DerefMut` คือเวอร์ชันที่อนุญาตให้แก้ไขค่าผ่าน dereference ได้ (`Box<T>` implement ทั้งสองตัว เพราะมันมีเจ้าของ
เดี่ยว การแก้ไขจึงปลอดภัยเสมอ) แต่ **`Rc<T>` implement แค่ `Deref` เท่านั้น** เพราะเหตุผลเชิง safety ที่สำคัญมาก:

> ถ้า `Rc<T>` ยอมให้แก้ไขข้อมูลผ่าน `&mut T` ได้ตรง ๆ ทั้งที่มีเจ้าของหลายคนพร้อมกัน (`strong_count > 1`) จะเกิด
> สถานการณ์ที่ **หลายคนมี `&T` (immutable reference จาก `Rc` ตัวอื่น ๆ) พร้อมกับมีคนหนึ่งกำลังแก้ไขผ่าน `&mut T`
> ในเวลาเดียวกัน** — นี่คือการละเมิด**กฎข้อที่ 1 ของ borrowing จาก Part 7 ตรง ๆ** (ห้ามมี `&mut T` พร้อมกับ `&T`
> ตัวอื่นในเวลาเดียวกัน) ซึ่งเป็นเงื่อนไขของ data race ที่ Rust ต้องป้องกันไม่ให้เกิดขึ้นได้เลย

ยังมีอีกวิธีหนึ่งที่ standard library เปิดช่องให้ลองแก้ไขผ่าน `Rc<T>` ได้ — method `Rc::get_mut()` ซึ่งคืนค่า
`Option<&mut T>` (ให้ `&mut T` มาจริง ๆ **ถ้า** เงื่อนไขปลอดภัยครบ) มาดูว่ามันทำงานอย่างไร:

```rust
use std::rc::Rc;

fn main() {
    let mut a = Rc::new(10);
    let b = Rc::clone(&a);
    println!("count = {}", Rc::strong_count(&a));

    match Rc::get_mut(&mut a) {
        Some(v) => println!("ได้ &mut มา แก้ไขเป็น: {v}"),
        None => println!(
            "ไม่สามารถแก้ไขได้ เพราะมีเจ้าของร่วมมากกว่า 1 คน (strong_count = {})",
            Rc::strong_count(&a)
        ),
    }
    println!("b = {b}");

    drop(b);
    println!("หลัง drop b, count = {}", Rc::strong_count(&a));
    match Rc::get_mut(&mut a) {
        Some(v) => {
            *v += 100;
            println!("ตอนนี้มีเจ้าของเดียว แก้ไขได้: {v}");
        }
        None => println!("ยังแก้ไม่ได้"),
    }
}
```

ผลลัพธ์ (รันจริง):

```
count = 2
ไม่สามารถแก้ไขได้ เพราะมีเจ้าของร่วมมากกว่า 1 คน (strong_count = 2)
b = 10
หลัง drop b, count = 1
ตอนนี้มีเจ้าของเดียว แก้ไขได้: 110
```

`Rc::get_mut()` **ไม่ panic และไม่ error ตอน compile** — มันแค่คืน `None` อย่างสุภาพเมื่อ `strong_count` มากกว่า 1
(หมายความว่ามีเจ้าของอื่นที่อาจถือ `&T` ไปใช้อยู่ ณ ขณะนั้น การให้ `&mut T` ออกไปจะไม่ปลอดภัย) และคืน `Some(&mut T)`
มาให้จริง ๆ ก็ต่อเมื่อ**เหลือเจ้าของแค่คนเดียวเท่านั้น** (`strong_count == 1`) เพราะตอนนั้นการันตีได้ว่าไม่มีใครถือ
`&T` ไปใช้พร้อมกันแน่นอน การแก้ไขจึงปลอดภัย 100% — สังเกตว่านี่คือการตรวจสอบกฎข้อที่ 1 ของ borrowing (Part 7)
**แบบมีเงื่อนไข** โดยใช้ตัวนับที่มีอยู่แล้วเป็นตัวชี้วัดความปลอดภัย

แต่ `Rc::get_mut()` มีข้อจำกัดที่ทำให้มันใช้แก้ปัญหาของเราไม่ได้จริง: **ในสถานการณ์ที่เราต้องการ (หลายเจ้าของแก้ไข
ข้อมูลร่วมกันได้อยู่แล้ว) `strong_count` จะมากกว่า 1 อยู่เสมอ** — นั่นแปลว่า `Rc::get_mut()` จะคืน `None` ตลอดเวลา
ในสถานการณ์จริงที่เราสนใจ มันใช้ได้ดีแค่ในกรณีพิเศษที่ต้องการ "แก้ไขครั้งสุดท้ายก่อนแจกจ่ายให้คนอื่นถือต่อ" เท่านั้น
ไม่ใช่กรณี "หลายคนแก้ไขพร้อมกันตลอดเวลา" ที่เราต้องการจริง ๆ

นี่คือจุดที่ทำให้เราต้องหาเครื่องมือใหม่ — เครื่องมือที่จะยอมให้แก้ไขข้อมูลได้ **แม้จะมีเจ้าของ/ผู้ยืมหลายคนพร้อมกัน**
โดยยังคงความปลอดภัยของกฎข้อที่ 1 ไว้อยู่ เพียงแค่เปลี่ยนวิธีตรวจสอบจาก "ห้ามเด็ดขาดตอน compile time" เป็น
**"ตรวจสอบตอน runtime แทน"** — เครื่องมือนั้นคือ `RefCell<T>`

### 28.6 Interior Mutability และ RefCell<T>

**Interior mutability** (การแก้ไขค่าภายในได้ แม้ตัวมันเองดูเหมือน immutable จากภายนอก) คือแนวคิดหลักของหัวข้อนี้
ปกติแล้ว Rust มีกฎง่าย ๆ ว่า: ถ้าคุณมี `&T` (immutable reference หรือ binding แบบไม่มี `mut`) คุณจะแก้ไขข้อมูล
ที่มันชี้ไปไม่ได้เลย — กฎนี้ถูกบังคับใช้ตอน compile time โดย borrow checker เสมอมา (ตามที่เรียนจาก Part 7)

`RefCell<T>` (`std::cell::RefCell`) ทำลายกฎนี้ในทางที่ **ปลอดภัย** — มันยอมให้คุณแก้ไขข้อมูลภายในได้ **แม้ผ่าน `&T`
หรือตัวแปรที่ไม่ได้ประกาศด้วย `mut`** โดยการย้ายการตรวจสอบกฎ borrowing ทั้งหมด **จาก compile time (borrow checker)
ไป runtime (ตัวนับภายในของ `RefCell` เอง)**

มาดูตัวอย่างที่แสดงพลังของ interior mutability ให้เห็นชัด ๆ:

```rust
use std::cell::RefCell;

struct Counter {
    count: RefCell<i32>,
}

impl Counter {
    fn new() -> Self {
        Counter { count: RefCell::new(0) }
    }

    fn increment(&self) {
        // สังเกต: รับ &self (immutable) ไม่ใช่ &mut self! แต่ยังแก้ไขค่าภายในได้
        let mut count = self.count.borrow_mut();
        *count += 1;
    }

    fn get(&self) -> i32 {
        *self.count.borrow()
    }
}

fn main() {
    let counter = Counter::new(); // สังเกต: ไม่ต้องมี `mut` เลยด้วยซ้ำ!
    counter.increment();
    counter.increment();
    counter.increment();
    println!("count = {}", counter.get());
}
```

ผลลัพธ์ (รันจริง):

```
count = 3
```

จุดที่ต้องเน้นให้เห็นชัดที่สุดคือ **2 อย่างที่ดู "ผิดกฎ" จาก Part 3/7 แต่กลับ compile และรันได้ถูกต้อง**:

1. `let counter = Counter::new();` — **ไม่มี `mut`** ตามกฎ Part 3 (`let` เป็น immutable โดย default) ตัวแปร `counter`
   นี้ไม่สามารถถูก reassign หรือแก้ไขฟิลด์โดยตรงได้เลยตามปกติ
2. `fn increment(&self)` — รับ `&self` (immutable reference ไปยัง `Counter`) **ไม่ใช่ `&mut self`** ตามกฎ Part 7
   ฟังก์ชันนี้ไม่มีสิทธิ์แก้ไขฟิลด์ของ `Counter` ผ่าน `self` ตรง ๆ เลย

แต่ `count.field` ภายในกลับถูกแก้ไขสำเร็จ 3 ครั้ง เพราะ **ฟิลด์ `count` ไม่ได้เป็น `i32` ตรง ๆ แต่เป็น `RefCell<i32>`**
— `RefCell<T>` คือ "ภาชนะพิเศษ" ที่ยอมให้แก้ไขข้อมูล `T` ที่อยู่ภายในได้ **แม้ตัว `RefCell<T>` เองถูกเข้าถึงผ่าน `&T`
เท่านั้น** เพราะการแก้ไขข้อมูลจริง ๆ ไม่ได้ทำผ่าน mutable reference ของ Rust ตามปกติ แต่ทำผ่าน **method พิเศษของ
`RefCell`** ที่จะอธิบายต่อไปนี้

#### `.borrow()` และ `.borrow_mut()`: หัวใจของ RefCell<T>

`RefCell<T>` มี 2 method หลักที่ใช้เข้าถึงข้อมูลภายใน:

| Method | คืนค่า type | ความหมาย | คล้ายกับ |
|---|---|---|---|
| `.borrow()` | `Ref<'_, T>` | ขอสิทธิ์ **อ่าน** ข้อมูลภายในชั่วคราว | `&T` (immutable reference) |
| `.borrow_mut()` | `RefMut<'_, T>` | ขอสิทธิ์ **แก้ไข** ข้อมูลภายในชั่วคราว | `&mut T` (mutable reference) |

`Ref<'_, T>` และ `RefMut<'_, T>` ไม่ใช่ reference ธรรมดา แต่เป็น **struct ที่ทำหน้าที่เป็น "ใบยืม" (guard)** —
มันทำงานเหมือน `&T`/`&mut T` ได้ทุกอย่างผ่าน `Deref`/`DerefMut` (auto-deref ทำงานให้ตามปกติจาก Part 7/27) แต่มี
สิ่งพิเศษเพิ่มเข้ามา: **ตอนที่มันถูกสร้างขึ้น (เรียก `.borrow()`/`.borrow_mut()`) มันจะไปเพิ่มตัวนับภายในของ
`RefCell` และตอนที่มันถูก drop (หมด scope) มันจะไปลดตัวนับนั้นกลับ** — พูดง่าย ๆ คือ `Ref`/`RefMut` คือ "ตัวติดตาม
ว่าใครกำลังยืมอยู่บ้าง" ที่ `RefCell` ใช้แทนสิ่งที่ borrow checker เคยทำให้ตอน compile time

ภายใน `RefCell<T>` มีตัวนับ (ในทางความหมาย ไม่ใช่ implementation จริง) ที่ทำงานคล้ายกฎข้อที่ 1 จาก Part 7 เป๊ะ ๆ:

- เรียก `.borrow()` ได้**หลายครั้งพร้อมกัน**ตราบใดที่ไม่มี `.borrow_mut()` ตัวใดมีชีวิตอยู่ ณ ขณะนั้น (เหมือน `&T`
  หลายตัวพร้อมกันได้ใน Part 7)
- เรียก `.borrow_mut()` ได้**ทีละครั้งเท่านั้น** และต้องไม่มี `.borrow()`/`.borrow_mut()` ตัวอื่นมีชีวิตอยู่เลย
  (เหมือน `&mut T` ต้องมีแค่ตัวเดียวและห้ามมี `&T` อื่นซ้อนใน Part 7)

**ประโยคที่สำคัญที่สุดของบทนี้คือ: กฎที่ `RefCell<T>` บังคับใช้คือกฎเดียวกันเป๊ะ ๆ กับกฎข้อที่ 1 ของ borrowing จาก
Part 7 (mutable ได้แค่หนึ่งตัว หรือ immutable ได้หลายตัว แต่ไม่ทั้งสองพร้อมกัน) — สิ่งเดียวที่เปลี่ยนไปคือ "ผู้ตรวจสอบ"
กฎนี้ ไม่ใช่ตัวกฎเอง** — borrow checker ตรวจตอน compile time โดยดูโค้ดล้วน ๆ (static analysis) ส่วน `RefCell`
ตรวจตอน runtime โดยดูตัวนับจริง ๆ ที่เปลี่ยนแปลงไปตามการทำงานของโปรแกรม (dynamic/runtime check)

### 28.7 กฎ Borrowing ของ RefCell ที่ตรวจตอน Runtime: BorrowError / BorrowMutError

เมื่อกฎถูกย้ายจาก compile time ไป runtime สิ่งที่เปลี่ยนไปคือ **รูปแบบของการแจ้งเตือนเมื่อละเมิดกฎ** — borrow checker
ที่ compile time จะทำให้โค้ด **compile ไม่ผ่านเลย** (คุณไม่มีทางรันโค้ดที่ผิดกฎได้ตั้งแต่แรก) แต่ `RefCell<T>` ที่
runtime จะทำให้โค้ด **compile ผ่านสนิท** แล้วไป **panic ตอนรันจริง** เมื่อละเมิดกฎ — นี่คือ **trade-off ที่สำคัญที่สุด**
ของ `RefCell<T>` ที่ต้องเข้าใจให้ลึก:

> **คุณแลกความปลอดภัยที่การันตีได้ตอน compile time กับความยืดหยุ่นที่เขียนโค้ดแบบที่ borrow checker ปกติจะไม่ยอมให้
> compile ผ่าน** — แต่ความรับผิดชอบในการไม่ละเมิดกฎก็ตกมาอยู่ที่ตัวคุณเองเต็ม ๆ เพราะ compiler ไม่ช่วยจับให้แล้ว
> ถ้าคุณเขียนผิด โปรแกรมจะ **compile ผ่านแต่ panic ตอนรัน** ซึ่งอาจเกิดขึ้นเฉพาะบางเงื่อนไข (เช่น เฉพาะ path ของ
> โค้ดบางเส้นทางที่ไม่ค่อยถูกทดสอบ) — อันตรายกว่า compile error ตรงที่มันอาจหลุดรอดไปถึง production ได้ถ้า
> test coverage ไม่ครอบคลุมพอ

มาดู panic จริงที่เกิดขึ้นเมื่อละเมิดกฎ ลองถือ `.borrow_mut()` สองตัวพร้อมกัน:

```rust
use std::cell::RefCell;

fn main() {
    let cell = RefCell::new(5);

    let borrow1 = cell.borrow_mut();
    println!("borrow1 = {borrow1}");
    let borrow2 = cell.borrow_mut(); // ❌ borrow1 ยังมีชีวิตอยู่ (ยังไม่หมด scope)

    println!("{} {}", borrow1, borrow2);
}
```

โค้ดนี้ **compile ผ่านสนิท** (ไม่มี error ใด ๆ จาก compiler เลย!) แต่เมื่อรันจริงจะ **panic** ทันทีที่ถึงบรรทัด
`cell.borrow_mut()` ตัวที่สอง:

```
thread 'main' panicked at src/main.rs:8:24:
RefCell already borrowed
stack backtrace:
   ...
note: Some details are omitted, run with `RUST_BACKTRACE=full` for a verbose backtrace.
```

(ข้อความ panic ข้างบนคือข้อความจริงที่ได้จากการรันโค้ดนี้ ในบางเวอร์ชันของ Rust ก่อนหน้าอาจแสดงเป็น
`already borrowed: BorrowMutError` แทน — ทั้งสองรูปแบบสื่อความหมายเดียวกัน: การเรียก `.borrow_mut()` ตัวที่สอง
ล้มเหลวเพราะภายในมีการตรวจสอบและคืนค่า `Err(BorrowMutError)` จาก `try_borrow_mut()` แล้ว `.borrow_mut()` ก็แค่เรียก
`try_borrow_mut().expect(...)` ต่ออีกชั้น ซึ่ง `.expect()` panic ทันทีเมื่อได้ `Err`)

ทีนี้ลองอีกกรณี: ถือ `.borrow_mut()` ไว้ก่อน แล้วพยายามเรียก `.borrow()` (ขอสิทธิ์อ่าน) ขณะที่ยังมี exclusive
borrow ค้างอยู่:

```rust
use std::cell::RefCell;

fn main() {
    let cell = RefCell::new(5);

    let borrow_mut1 = cell.borrow_mut();
    println!("borrow_mut1 = {borrow_mut1}");
    let borrow2 = cell.borrow(); // ❌ borrow_mut1 ยังมีชีวิตอยู่

    println!("{} {}", borrow_mut1, borrow2);
}
```

ผลลัพธ์เมื่อรันจริง:

```
thread 'main' panicked at src/main.rs:8:24:
RefCell already mutably borrowed
stack backtrace:
   ...
```

สังเกตว่า **ข้อความ panic ต่างกันเล็กน้อย** ตามลำดับการละเมิด: กรณีแรก (`.borrow_mut()` ซ้อน `.borrow_mut()`) ภายใน
`RefCell` คืนค่า type `BorrowMutError` (ความหมาย: "ขอ mutable borrow ไม่ได้ เพราะมีการยืม — ไม่ว่าจะ mutable หรือ
immutable — ค้างอยู่แล้ว") ส่วนกรณีที่สอง (`.borrow_mut()` ค้างไว้ แล้วเรียก `.borrow()`) คืนค่า type `BorrowError`
(ความหมาย: "ขอ immutable borrow ไม่ได้ เพราะมี mutable borrow ค้างอยู่แล้ว") — ทั้งสอง type นี้คือชนิดของ error ที่
`RefCell::try_borrow()` และ `RefCell::try_borrow_mut()` คืนออกมาเวลาที่กฎถูกละเมิด และ `.borrow()`/`.borrow_mut()`
(ไม่มี `try_`) ก็แค่เรียก `try_` แล้ว `.expect()` ทันทีถ้าได้ `Err` — panic ที่เห็นข้างบนคือผลจาก `.expect()` นั่นเอง

ถ้าคุณต้องการ **จัดการกรณีที่อาจ borrow ไม่สำเร็จโดยไม่ให้โปรแกรม panic** ให้ใช้ `try_borrow()`/`try_borrow_mut()`
ที่คืน `Result<Ref<'_, T>, BorrowError>` / `Result<RefMut<'_, T>, BorrowMutError>` แล้วจัดการ `Err` เองด้วย `match`
เหมือน `Result<T, E>` ทั่วไปที่เรียนมาจากบทก่อน ๆ — แต่ในโค้ดส่วนใหญ่ที่ตรรกะการยืม/แก้ไขถูกออกแบบมาอย่างรัดกุม
คุณมักไม่จำเป็นต้องใช้ `try_` เพราะการเกิด panic ในกรณีนี้มักหมายถึง **บั๊ก** ในการออกแบบ scope ของ `.borrow()`
มากกว่าเป็นสถานการณ์ปกติที่ควรรองรับแบบ graceful

#### เทียบให้เห็นชัด: borrow checker (compile time) vs RefCell (runtime)

ตารางนี้เทียบแนวคิดของ Part 7 กับแนวคิดของหัวข้อนี้แบบเคียงข้างกัน เพื่อยืนยันอีกครั้งว่า **กฎที่ถูกบังคับใช้คือกฎ
เดียวกัน** เปลี่ยนแค่ "ใครตรวจ" และ "ตรวจตอนไหน":

| แง่มุม | `&T` / `&mut T` (Part 7) | `Ref<'_, T>` / `RefMut<'_, T>` จาก `RefCell<T>` |
|---|---|---|
| ใครตรวจกฎ "mutable หนึ่งตัว หรือ immutable หลายตัว" | Borrow checker ของ `rustc` | ตัวนับภายในของ `RefCell` เอง |
| ตรวจตอนไหน | **Compile time** — ก่อนโปรแกรมรันด้วยซ้ำ | **Runtime** — ตอนที่ `.borrow()`/`.borrow_mut()` ถูกเรียกจริง |
| ถ้าละเมิดกฎจะเกิดอะไรขึ้น | โปรแกรม **compile ไม่ผ่านเลย** (`E0499`, `E0502` จาก Part 7) | โปรแกรม compile ผ่าน แต่ **panic ตอนรัน** (`BorrowError`/`BorrowMutError`) |
| ต้นทุนตอน runtime | **ไม่มีเลย** (zero-cost — ตรวจจบไปแล้วตั้งแต่ compile time) | มีเล็กน้อย (เช็ค/อัปเดตตัวนับทุกครั้งที่ยืม) |
| เขียนโค้ดที่ผ่าน borrow checker ปกติไม่ได้ (เพราะ compiler มองไม่เห็นว่าปลอดภัยจริง) แต่ตัวเราการันตีได้เองว่าปลอดภัย | ทำไม่ได้ — compiler ตัดสินสุดท้าย | **ทำได้** — นี่คือเหตุผลหลักที่มีอยู่ของ `RefCell<T>` |

แถวสุดท้ายคือใจความสำคัญที่สุด: บางครั้ง **คุณรู้ (จาก logic ของโปรแกรม) ว่าโค้ดปลอดภัยแน่นอน แต่ borrow checker
พิสูจน์แบบ static analysis ไม่ได้** เพราะมันวิเคราะห์แค่โครงสร้างของโค้ด ไม่รู้ตรรกะเชิง runtime (เช่นในตัวอย่าง
`Department`/`Budget` ที่ `.borrow_mut()` ของแต่ละ `Department.spend()` ไม่มีทางซ้อนกันจริงเพราะเรียกทีละครั้ง
ตามลำดับ แต่ borrow checker ธรรมดาไม่มีทางรู้เรื่องนี้ได้เลยถ้าไม่มี `RefCell` มาช่วย) — `RefCell<T>` คือทางออก
สำหรับสถานการณ์แบบนี้: ให้ compiler "ยกมือยอมแพ้" แล้วผลักภาระการตรวจสอบไปที่ runtime แทน โดยยังคงความปลอดภัยของ
กฎเดิมไว้ครบถ้วน (ไม่มี undefined behavior เกิดขึ้นได้เลย แค่ต้องแลกด้วยความเสี่ยงที่จะ panic ถ้าตรรกะที่คุณ "รู้"
ว่าปลอดภัยนั้นดันผิดจริง ๆ)

#### วิธีป้องกัน: จำกัด scope ของ Ref/RefMut ให้แคบที่สุด

หลักการป้องกันปัญหานี้ที่สำคัญที่สุดคือ: **ให้ `Ref`/`RefMut` หมด scope (ถูก drop) โดยเร็วที่สุดเท่าที่ทำได้** —
ใช้ block `{}` ครอบเฉพาะส่วนที่ต้องยืม แล้วให้มันหมดอายุก่อนที่จะยืมอีกครั้ง (หรือเรียกฟังก์ชันอื่นที่อาจยืมซ้อน):

```rust
use std::cell::RefCell;

fn main() {
    let cell = RefCell::new(5);

    {
        let mut value = cell.borrow_mut();
        *value += 10;
    } // <- value (RefMut) หมด scope ตรงนี้ ปล่อย borrow_mut แล้ว

    let value = cell.borrow(); // ✅ ตอนนี้ borrow_mut ปล่อยไปแล้ว เรียก .borrow() ได้ปลอดภัย
    println!("value = {value}");
}
```

หลักการนี้จะปรากฏอีกครั้งในตัวอย่างสมบูรณ์ท้ายบท (หัวข้อ 28.11) ซึ่งจะแสดงบั๊กจริงที่เกิดจากไม่จำกัด scope
และวิธีแก้ด้วยเทคนิคเดียวกันนี้

### 28.8 Rc<RefCell<T>>: คอมโบที่ทรงพลังที่สุด

ตอนนี้เราได้เครื่องมือสองตัวที่เติมเต็มกันอย่างสมบูรณ์แบบ:

- **`Rc<T>`** ให้ **เจ้าของหลายคน (multiple ownership)** ได้ แต่ให้แค่สิทธิ์**อ่าน**อย่างเดียว
- **`RefCell<T>`** ให้ **แก้ไขข้อมูลได้แม้ผ่าน `&T`** แต่ยังเป็น owned value เดี่ยว ๆ (มีเจ้าของได้แค่คนเดียวเหมือนปกติ)

เมื่อนำสองตัวมาซ้อนกันเป็น **`Rc<RefCell<T>>`** เราจะได้ทั้งสองอย่างพร้อมกัน: **หลายเจ้าของ ที่ทุกคนแก้ไขข้อมูล
ที่ใช้ร่วมกันได้จริง** — นี่คือ pattern ที่พบเจอบ่อยที่สุดในโค้ด Rust จริงเวลาต้องการ "shared mutable state"
แบบไม่มี thread เข้ามาเกี่ยวข้อง

ลองดูตัวอย่างที่เป็นรูปธรรม: หลายแผนกในบริษัทใช้งบประมาณ (`Budget`) ก้อนเดียวกัน โดยที่ทุกแผนกต้อง **อ่านงบที่เหลือ**
และ **หักงบเมื่อใช้เงิน** ได้จริง ๆ (ไม่ใช่แค่อ่าน):

```rust
use std::cell::RefCell;
use std::rc::Rc;

struct Budget {
    total: f64,
    spent: f64,
}

impl Budget {
    fn remaining(&self) -> f64 {
        self.total - self.spent
    }

    fn spend(&mut self, amount: f64) -> Result<(), String> {
        if amount > self.remaining() {
            return Err(format!(
                "งบไม่พอ: เหลือ {:.2} แต่ต้องใช้ {:.2}",
                self.remaining(),
                amount
            ));
        }
        self.spent += amount;
        Ok(())
    }
}

struct Department {
    name: String,
    budget: Rc<RefCell<Budget>>, // เจ้าของร่วม (Rc) ของงบที่แก้ไขได้ (RefCell)
}

impl Department {
    fn spend(&self, amount: f64) {
        // สังเกต: spend(&self) ไม่ใช่ &mut self! แต่แก้ไขงบที่ใช้ร่วมกันได้ผ่าน RefCell
        match self.budget.borrow_mut().spend(amount) {
            Ok(()) => println!("{}: ใช้เงินไป {:.2} สำเร็จ", self.name, amount),
            Err(e) => println!("{}: {}", self.name, e),
        }
    }
}

fn main() {
    let shared_budget = Rc::new(RefCell::new(Budget { total: 1000.0, spent: 0.0 }));

    let marketing = Department {
        name: "Marketing".to_string(),
        budget: Rc::clone(&shared_budget),
    };
    let engineering = Department {
        name: "Engineering".to_string(),
        budget: Rc::clone(&shared_budget),
    };

    marketing.spend(400.0);
    engineering.spend(500.0);
    marketing.spend(300.0); // งบเหลือแค่ 100 ตอนนี้ ไม่พอสำหรับ 300 -> ควร fail

    println!("งบที่เหลือทั้งบริษัท: {:.2}", shared_budget.borrow().remaining());
    println!(
        "จำนวนแผนกที่ถือ budget นี้ (strong_count) = {}",
        Rc::strong_count(&shared_budget)
    );
}
```

ผลลัพธ์ (รันจริง):

```
Marketing: ใช้เงินไป 400.00 สำเร็จ
Engineering: ใช้เงินไป 500.00 สำเร็จ
Marketing: งบไม่พอ: เหลือ 100.00 แต่ต้องใช้ 300.00
งบที่เหลือทั้งบริษัท: 100.00
จำนวนแผนกที่ถือ budget นี้ (strong_count) = 3
```

สังเกตให้ดีว่านี่คือสิ่งที่เป็นไปไม่ได้เลยด้วยเครื่องมือที่เรียนมาก่อนหน้านี้: `marketing` และ `engineering` เป็น
`Department` **คนละตัวกันสองตัว** ไม่มีความสัมพันธ์ทาง ownership กันเลยโดยตรง แต่ทั้งคู่ **แก้ไขข้อมูล `Budget`
ก้อนเดียวกันจริง ๆ** — เมื่อ `marketing` ใช้เงินไป 400 แล้ว `engineering` เรียก `.spend()` การหักเงินของ
`marketing` มีผลกระทบจริงต่อค่าที่ `engineering` เห็น (งบเหลือ 600 ตอนที่ engineering ใช้ ไม่ใช่ 1000 เต็ม) และ
ตอนที่ `marketing` พยายามใช้เงินครั้งที่สอง งบที่เหลือก็สะท้อนผลจากการใช้เงินของทั้งสองแผนกรวมกันแล้ว ทั้งหมดนี้เกิด
ขึ้นได้เพราะ `Rc<RefCell<Budget>>` ทำให้ทั้งสอง `Department` ชี้ไปยัง `Budget` ก้อนเดียวกันจริง ๆ (เหมือน `Rc<T>`
ธรรมดา) และแต่ละคนก็แก้ไขมันได้จริงผ่าน `.borrow_mut()` ของ `RefCell` (ซึ่งกฎ "หนึ่งคนแก้ไขทีละครั้ง" ก็ยังถูก
บังคับใช้อยู่ — สังเกตว่า `.borrow_mut()` ในเมธอด `spend` จะหมด scope ทันทีที่บรรทัด `match` จบ เพราะมันเป็น
temporary value ที่ไม่ถูกผูกกับตัวแปรใด ๆ จึงไม่มีปัญหา borrow ซ้อนเกิดขึ้นเลยในตัวอย่างนี้)

#### เชื่อมโยงไปข้างหน้า: Arc<Mutex<T>> คือเวอร์ชัน Thread-Safe ของ Pattern นี้

`Rc<RefCell<T>>` ที่เพิ่งเรียนนี้ใช้งานได้ดีมาก **แต่เฉพาะในโปรแกรม single-thread เท่านั้น** — `Rc<T>` ไม่ implement
trait `Send`/`Sync` (จะเรียนเต็ม ๆ ใน Part 40) เพราะตัวนับภายในของมันเพิ่ม/ลดโดยไม่มีการ synchronize ข้าม thread
เลย ถ้าสอง thread เพิ่ม/ลดตัวนับพร้อมกันจริง ๆ (ไม่ผ่าน lock ใด ๆ) จะเกิด data race บนตัวนับเองได้ — และในทำนอง
เดียวกัน `RefCell<T>` ก็ตรวจสอบ borrow rule ด้วยตัวนับที่ไม่ thread-safe เช่นกัน

ใน **Part 39 (Mutex, Arc และ Shared-State Concurrency)** คุณจะเจอ **`Arc<Mutex<T>>`** ซึ่งเป็น "ฝาแฝดที่ปลอดภัย
ข้าม thread" ของ pattern นี้เป๊ะ ๆ:

| Pattern | ใช้ได้กับ | ตัวนับ/lock thread-safe หรือไม่ | หลักการเหมือนกันอย่างไร |
|---|---|---|---|
| `Rc<RefCell<T>>` | Single-thread เท่านั้น | ไม่ (ตัวนับธรรมดา, เร็วกว่า) | `Rc` = หลายเจ้าของ, `RefCell` = แก้ไขได้ผ่าน `&T` โดยตรวจ borrow ตอน runtime |
| `Arc<Mutex<T>>` | Multi-thread ได้ปลอดภัย | ใช่ (`Arc` ใช้ atomic operations, `Mutex` ใช้ OS-level lock) | `Arc` = หลายเจ้าของข้าม thread, `Mutex` = แก้ไขได้ผ่าน `&T` โดย lock ให้ thread อื่นรอ |

โครงสร้างความคิดเบื้องหลังทั้งสองคู่นี้**เหมือนกันเป๊ะ ๆ**: "smart pointer สำหรับหลายเจ้าของ" ห่อด้วย "container
สำหรับแก้ไขข้อมูลผ่าน shared reference" — ต่างกันแค่ว่าต้องรับมือกับ thread หลายตัวพร้อมกันหรือไม่เท่านั้น ถ้าคุณเข้าใจ
`Rc<RefCell<T>>` แน่นในบทนี้ การเรียน `Arc<Mutex<T>>` ใน Part 39 จะรู้สึกคุ้นเคยมากเพราะเป็นแนวคิดเดียวกันที่ขยายให้
ทำงานข้าม thread ได้อย่างปลอดภัย

### 28.9 Reference Cycles: วิธีเดียวที่ Rust ยอมให้ Memory รั่วแบบ Safe

มีข้อเสียหนึ่งของ `Rc<T>` ที่ต้องเข้าใจให้ลึกก่อนใช้งานจริง: **ระบบ ownership ของ Rust ทั้งหมด (Part 6-7) ถูกออกแบบ
มาเพื่อป้องกัน use-after-free, double-free, และ data race — แต่ไม่ได้ถูกออกแบบมาเพื่อป้องกัน "memory leak" (การไม่
ปล่อยหน่วยความจำที่ไม่ใช้แล้ว) อย่างสมบูรณ์แบบ** และ `Rc<T>` คือช่องทางที่ทำให้ leak เกิดขึ้นได้ **แบบ safe 100%**
(compile ผ่าน ไม่ต้องมี `unsafe` เลยสักตัว) — สถานการณ์นี้เกิดขึ้นเมื่อมี **reference cycle** (วงจรอ้างอิงกันไปมา)

ลองดูตัวอย่างขั้นต่ำที่สุดที่แสดงปัญหานี้: สอง `Node` ที่ชี้กลับไปมาถึงกันเองผ่าน `Rc<RefCell<...>>`

```rust
use std::cell::RefCell;
use std::rc::Rc;

struct Node {
    value: i32,
    next: RefCell<Option<Rc<Node>>>,
}

fn main() {
    let a = Rc::new(Node { value: 1, next: RefCell::new(None) });
    println!("a strong_count เริ่มต้น = {}", Rc::strong_count(&a));

    let b = Rc::new(Node { value: 2, next: RefCell::new(Some(Rc::clone(&a))) });
    println!("a strong_count หลัง b ชี้มาที่ a = {}", Rc::strong_count(&a));
    println!("b strong_count เริ่มต้น = {}", Rc::strong_count(&b));

    // สร้าง cycle จริง ๆ: ให้ a ชี้กลับไปที่ b ด้วย (ก่อนหน้านี้ b ชี้ไปที่ a อยู่แล้ว)
    *a.next.borrow_mut() = Some(Rc::clone(&b));
    println!("b strong_count หลัง a ชี้กลับไปที่ b = {}", Rc::strong_count(&b));
    println!("a strong_count สุดท้าย = {}", Rc::strong_count(&a));

    println!("a.value = {}, b.value = {}", a.value, b.value);
}
```

ผลลัพธ์ (รันจริง):

```
a strong_count เริ่มต้น = 1
a strong_count หลัง b ชี้มาที่ a = 2
b strong_count เริ่มต้น = 1
b strong_count หลัง a ชี้กลับไปที่ b = 2
a strong_count สุดท้าย = 2
a.value = 1, b.value = 2
```

ไล่ดูสถานะตัวนับทีละจุด: `a` มี `strong_count = 2` (ตัวแปร `a` ใน `main` เป็นเจ้าของหนึ่ง + `b.next` ถือ `Rc`
ไปยัง `a` อีกหนึ่ง) และ `b` ก็มี `strong_count = 2` เช่นกัน (ตัวแปร `b` ใน `main` + `a.next` ถือ `Rc` ไปยัง `b`)
ทั้งสองชี้ถึงกันเป็นวงจรสมบูรณ์แล้ว ณ จุดนี้ (ถ้าลองเรียก `Rc::weak_count(&a)` ดูตอนนี้ จะได้ 0 เพราะเรายังไม่ได้
ใช้ `Weak<T>` เลยในตัวอย่างนี้ — `weak_count` จากภาพ memory layout ในหัวข้อ 28.3 จะเริ่มไม่เป็น 0 ก็ต่อเมื่อมีคน
เรียก `Rc::downgrade()` ซึ่งเป็นเนื้อหาของ Part 29)

ทีนี้ลองนึกภาพว่าถ้า `main` จบลง ตัวแปร `a` และ `b` หมด scope ทั้งคู่จะเกิดอะไรขึ้น:

1. `a` (ตัวแปรใน `main`) หมด scope → `strong_count` ของ `Node` ที่ `a` เดิมชี้ไปลดลงจาก 2 เหลือ **1**
   (เพราะยังมี `b.next` ถือ `Rc` ไปยังมันอยู่)
2. `b` (ตัวแปรใน `main`) หมด scope → `strong_count` ของ `Node` ที่ `b` เดิมชี้ไปลดลงจาก 2 เหลือ **1**
   (เพราะยังมี `a.next` ถือ `Rc` ไปยังมันอยู่)
3. **ไม่มีตัวนับตัวใดลดลงถึง 0 เลย!** เพราะ `Node` ทั้งสองก้อนยัง "ค้ำ" กันและกันอยู่ผ่าน field `next` ของกันเอง
   ตลอดไป — ไม่มีจุดใดในโปรแกรมที่จะทำให้ `strong_count` ของทั้งคู่ลดลงมาถึง 0 ได้อีกต่อไปเลย เพราะไม่มี "เจ้าของ
   จากภายนอก" เหลืออยู่ที่จะไป drop มันได้ — วงจรกลายเป็น "เกาะ" ที่ลอยอยู่บน heap ตลอดไปโดยไม่มีใครเข้าถึงได้จาก
   ภายนอกอีก และก็ไม่มีใครทำลายมันได้เช่นกัน

**นี่คือ memory leak ที่แท้จริง** — หน่วยความจำของ `Node` สองก้อนนี้จะไม่ถูกคืนให้ระบบปฏิบัติการเลยตลอดอายุของ
โปรแกรม (ยกเว้นตอนโปรแกรมทั้งตัวจบและ OS เคลียร์หน่วยความจำทั้งหมดของ process ให้ ซึ่งไม่ใช่เรื่องที่ Rust runtime
ทำให้)

#### ทำไม Rust "ยอม" ให้เกิดเรื่องนี้ได้ ทั้งที่พยายามป้องกัน bug ด้าน memory มาตลอด

คำถามที่สมเหตุสมผลมาก: ถ้า Rust เข้มงวดเรื่อง memory safety มากถึงขนาดนี้ ทำไมยังยอมให้ leak แบบนี้เกิดขึ้นได้?
คำตอบคือ **memory leak ไม่ใช่ memory *unsafety*** — leak หมายถึงหน่วยความจำที่ยังใช้งานได้ปกติ (ไม่มี
use-after-free, ไม่มี dangling pointer, ไม่มี data race) แต่แค่**ไม่ถูกปล่อยคืนเมื่อไม่ต้องการใช้แล้ว** ซึ่งเป็น
ปัญหาด้าน "ประสิทธิภาพ/การจัดการทรัพยากร" ไม่ใช่ปัญหาด้าน "ความปลอดภัย" (safety) ในความหมายที่ Rust รับประกัน —
สัญญาที่ Rust ให้ไว้คือ **"safe Rust จะไม่มี undefined behavior"** ไม่ใช่ **"safe Rust จะไม่มีการรั่วของ
หน่วยความจำเลย"** — สองอย่างนี้เป็นคนละสัญญากัน และ `Rc<T>` ที่สร้าง cycle ได้คือตัวอย่างที่ชัดเจนที่สุดของช่องว่าง
ระหว่างสัญญาทั้งสองนี้

พูดอีกแบบ: **การตรวจจับ reference cycle แบบทั่วไปเป็นปัญหาที่ยากมาก (ในทางทฤษฎีคอมพิวเตอร์ใกล้เคียงกับปัญหาที่
ตัดสินไม่ได้ทั่วไปในหลายกรณี) และ Rust เลือกที่จะไม่พยายามพิสูจน์เรื่องนี้ตอน compile time เลย** เพื่อให้ compiler
ยังคงทำงานได้เร็วและระบบ ownership ยังเรียบง่ายพอที่จะเข้าใจได้ — ภาษาที่มี garbage collector (เช่น บางส่วนของ
Python ที่มี cycle detector, หรือ Java/Go) มักมีกลไกพิเศษตรวจจับ cycle ตอน runtime แล้วเก็บกวาดให้ (ซึ่งมี
runtime cost) แต่ Rust เลือกเส้นทาง zero-cost abstraction (ไม่มี garbage collector) จึงต้อง**ผลักความรับผิดชอบ
ในการป้องกัน cycle มาให้ผู้เขียนโค้ดเอง** เป็นการแลกเปลี่ยนที่ตั้งใจทำ (deliberate trade-off) ไม่ใช่ข้อบกพร่อง
ของภาษา

#### ทางแก้: Weak<T> (เนื้อหาของ Part 29)

วิธีแก้ปัญหา reference cycle ที่ถูกต้องคือการใช้ **`Weak<T>`** — smart pointer ที่ "ชี้ไปยังข้อมูลได้เหมือน `Rc<T>`
แต่ไม่นับเป็นเจ้าของ" (ไม่เพิ่ม `strong_count`) พูดง่าย ๆ คือ `Weak<T>` คือการยอมรับว่า "ฉันสนใจข้อมูลนี้ แต่ฉันไม่ใช่
เจ้าของมัน และฉันโอเคที่มันอาจถูกทำลายไปแล้วเมื่อฉันมาเช็คทีหลัง" — ถ้าในตัวอย่างข้างบนเราให้ `next` ของ `Node`
เก็บ `Weak<Node>` แทน `Rc<Node>` (สำหรับความสัมพันธ์แบบ "ชี้กลับ" หรือ "child ไปยัง parent" ที่ไม่ควรนับเป็น
ownership จริง) วงจรจะไม่เกิดขึ้นอีกเลย เพราะ `strong_count` ของแต่ละฝั่งจะลดลงถึง 0 ได้จริงตามลำดับที่ตัวแปร
หมด scope

รายละเอียดทั้งหมดของ `Weak<T>` — วิธีสร้าง (`Rc::downgrade`), วิธีใช้ (`.upgrade()` ที่คืน `Option<Rc<T>>`), และ
รูปแบบการออกแบบ tree/graph ที่ใช้ `Rc` สำหรับความสัมพันธ์ "เจ้าของจริง" (เช่น parent เป็นเจ้าของ child) ผสมกับ
`Weak` สำหรับความสัมพันธ์ "แค่ชี้ถึง" (เช่น child ชี้กลับไปยัง parent) จะเป็นเนื้อหาหลักของ **Part 29 (Smart
Pointers: Weak<T>, Cow<T>)** ที่ต่อจากบทนี้โดยตรง — ตอนนี้แค่จำไว้ว่า **ทุกครั้งที่คุณออกแบบโครงสร้างข้อมูลด้วย
`Rc<RefCell<T>>` ที่มีการอ้างอิงไปมาระหว่างกันในหลายทิศทาง ให้ตั้งคำถามเสมอว่า "ทิศทางไหนคือ ownership จริง ๆ
(ควรเป็น `Rc`) และทิศทางไหนแค่ต้องการ 'รู้จัก' กัน (ควรเป็น `Weak`)"** — คำถามนี้คือกุญแจสำคัญที่จะป้องกัน
reference cycle ตั้งแต่ตอนออกแบบ

### 28.10 Cell<T>: ลูกพี่ลูกน้องที่เบากว่าของ RefCell<T>

ก่อนไปตัวอย่างสรุปท้ายบท ควรรู้จัก **`Cell<T>`** (`std::cell::Cell`) ไว้สั้น ๆ เพราะเป็นญาติสนิทของ `RefCell<T>`
ที่พบเจอได้บ้างในโค้ดจริง — `Cell<T>` ก็ทำ interior mutability เหมือนกัน แต่มีข้อจำกัดและข้อดีที่ต่างจาก
`RefCell<T>` อย่างชัดเจน:

| ลักษณะ | `Cell<T>` | `RefCell<T>` |
|---|---|---|
| Type ที่ใช้งานได้ | ต้อง implement `Copy` เท่านั้น (หรือใช้ `.replace()`/`.take()` กับ type อื่นได้บ้าง) | ใช้กับ type ใดก็ได้ |
| วิธีเข้าถึงข้อมูล | `.get()` (คืนค่าทั้งก้อนแบบ copy), `.set(v)` (เขียนทับค่าทั้งก้อน) | `.borrow()`/`.borrow_mut()` (คืน "ใบยืม" ที่ผูกกับข้อมูลจริงชั่วคราว) |
| มีการตรวจสอบ borrow rule ตอน runtime หรือไม่ | **ไม่มี** — เพราะไม่มีการ "ยืม" เกิดขึ้นเลย มีแต่การ copy ค่าเข้า/ออกทั้งก้อน | **มี** — ตรวจสอบผ่านตัวนับตามที่อธิบายไว้ในหัวข้อ 28.7 |
| Cost ตอน runtime | ต่ำกว่า (ไม่มี branch ตรวจสอบเลย) | สูงกว่าเล็กน้อย (ต้องเช็ค/อัปเดตตัวนับทุกครั้งที่ยืม) |
| อาจ panic ตอนรันได้หรือไม่ | **ไม่มีทาง panic เลย** | **panic ได้** ถ้าละเมิดกฎ borrowing |

เหตุผลที่ `Cell<T>` ไม่ต้องตรวจสอบ borrow rule ตอน runtime เลยคือ **มันไม่ได้ให้ reference ไปยังข้อมูลภายในออกมา
เลยสักครั้ง** — `.get()` คืนค่า **สำเนา** (copy) ของข้อมูลออกมาตรง ๆ และ `.set()` **เขียนทับข้อมูลทั้งก้อน** ในครั้ง
เดียว ไม่มีช่วงเวลาใดที่จะมี "ใบยืม" (reference) ไปยังข้อมูลภายในเดินทางอยู่ในโปรแกรมเลย — เมื่อไม่มี reference
ที่อาจ "ค้าง" อยู่ ก็ไม่มีทางเกิดสถานการณ์ที่ reference สองตัวขัดแย้งกันได้เลยตั้งแต่ต้น (สอดคล้องกับกฎข้อที่ 1
จาก Part 7 แบบอัตโนมัติ เพราะไม่มี "&T ที่มีชีวิตอยู่นาน" ให้ขัดแย้งกับใครได้เลย)

```rust
use std::cell::Cell;

struct Stats {
    hits: Cell<u32>,
}

fn main() {
    let stats = Stats { hits: Cell::new(0) };
    stats.hits.set(stats.hits.get() + 1);
    stats.hits.set(stats.hits.get() + 1);
    stats.hits.set(stats.hits.get() + 1);
    println!("hits = {}", stats.hits.get());
}
```

ผลลัพธ์ (รันจริง):

```
hits = 3
```

กฎการเลือกใช้ที่จำง่าย: **ถ้าข้อมูลภายในเป็น type เล็ก ๆ ที่ `Copy` ได้ (เช่น `i32`, `bool`, `f64`) และคุณแค่ต้องการ
อ่าน/เขียนค่าทั้งก้อนแบบอะตอมมิก (ไม่แยกอ่านบางส่วนแก้บางส่วน) ให้ใช้ `Cell<T>`** เพราะเร็วกว่าและไม่มีความเสี่ยง
panic เลย **แต่ถ้าข้อมูลภายในเป็น struct ใหญ่ที่ต้องแก้ไขบางฟิลด์ผ่าน method ที่รับ `&mut self`, หรือเป็น `Vec<T>`
ที่ต้อง `.push()`/`.pop()` ทีละส่วน ให้ใช้ `RefCell<T>`** เพราะ `Cell<T>` ไม่มีทางให้ `&mut T` ออกมาให้เรียก method
แบบนั้นได้เลย บทนี้จะโฟกัสที่ `RefCell<T>` เป็นหลักเพราะเป็นตัวที่ใช้บ่อยกว่ามากในโค้ดจริง ส่วน `Cell<T>` แค่ต้อง
รู้จักไว้เผื่อเจอในโค้ดคนอื่นหรือในกรณีที่เหมาะสมจริง ๆ

### 28.11 ตัวอย่างจบครบวงจร: ระบบบัญชีธนาคารที่ใช้ร่วมกันหลายตู้ ATM

มาปิดเนื้อหาทั้งหมดของบทนี้ด้วยตัวอย่างที่ครบวงจรที่สุด: ระบบบัญชีธนาคารหนึ่งบัญชี ที่ถูกเข้าถึงจาก **ตู้ ATM หลายตู้**
พร้อมกัน (จำลองการที่ผู้ใช้คนเดียวไปกดเงินจากตู้ ATM คนละสาขาในเวลาใกล้เคียงกัน) — แต่ละตู้ ATM ไม่ได้เป็นเจ้าของ
บัญชีนั้นแต่เพียงผู้เดียว และทุกตู้ต้องเห็นยอดคงเหลือที่อัปเดตล่าสุดตรงกันเสมอ

#### เวอร์ชันที่ถูกต้อง

```rust
use std::cell::RefCell;
use std::rc::Rc;

#[derive(Debug)]
struct Account {
    owner: String,
    balance: f64,
    history: Vec<String>,
}

impl Account {
    fn new(owner: &str, initial: f64) -> Self {
        Account {
            owner: owner.to_string(),
            balance: initial,
            history: vec![format!("เปิดบัญชีด้วยยอด {:.2} บาท", initial)],
        }
    }

    fn deposit(&mut self, amount: f64) {
        self.balance += amount;
        self.history.push(format!("ฝาก {:.2} บาท", amount));
    }

    fn withdraw(&mut self, amount: f64) -> Result<(), String> {
        if amount > self.balance {
            return Err(format!("ยอดไม่พอ: มี {:.2} แต่ถอน {:.2}", self.balance, amount));
        }
        self.balance -= amount;
        self.history.push(format!("ถอน {:.2} บาท", amount));
        Ok(())
    }
}

struct AtmTerminal {
    location: String,
    account: Rc<RefCell<Account>>, // ทุกตู้ถือ Rc ที่ชี้ไปยังบัญชีเดียวกัน แก้ไขได้ผ่าน RefCell
}

impl AtmTerminal {
    fn deposit(&self, amount: f64) {
        self.account.borrow_mut().deposit(amount);
        println!("[{}] ฝากเงิน {:.2} บาทสำเร็จ", self.location, amount);
    }

    fn withdraw(&self, amount: f64) {
        match self.account.borrow_mut().withdraw(amount) {
            Ok(()) => println!("[{}] ถอนเงิน {:.2} บาทสำเร็จ", self.location, amount),
            Err(e) => println!("[{}] ถอนเงินไม่สำเร็จ: {}", self.location, e),
        }
    }

    fn print_balance(&self) {
        let acc = self.account.borrow();
        println!(
            "[{}] ยอดคงเหลือของ {}: {:.2} บาท",
            self.location, acc.owner, acc.balance
        );
    }
}

fn main() {
    let account = Rc::new(RefCell::new(Account::new("สมชาย", 1000.0)));

    let atm_siam = AtmTerminal { location: "สาขาสยาม".to_string(), account: Rc::clone(&account) };
    let atm_asoke = AtmTerminal { location: "สาขาอโศก".to_string(), account: Rc::clone(&account) };

    atm_siam.deposit(500.0);
    atm_asoke.withdraw(300.0);
    atm_siam.print_balance();
    atm_asoke.print_balance();

    println!("จำนวนผู้ถือ Rc ตอนนี้ = {}", Rc::strong_count(&account));
    println!("ประวัติทั้งหมด: {:?}", account.borrow().history);
}
```

ผลลัพธ์ (รันจริง):

```
[สาขาสยาม] ฝากเงิน 500.00 บาทสำเร็จ
[สาขาอโศก] ถอนเงิน 300.00 บาทสำเร็จ
[สาขาสยาม] ยอดคงเหลือของ สมชาย: 1200.00 บาท
[สาขาอโศก] ยอดคงเหลือของ สมชาย: 1200.00 บาท
จำนวนผู้ถือ Rc ตอนนี้ = 3
ประวัติทั้งหมด: ["เปิดบัญชีด้วยยอด 1000.00 บาท", "ฝาก 500.00 บาท", "ถอน 300.00 บาท"]
```

สังเกตว่า `atm_asoke.print_balance()` เห็นยอด **1200** ซึ่งรวมผลของการฝากเงิน 500 ที่ `atm_siam` ทำไปก่อนหน้านั้น
ทั้ง ๆ ที่ `atm_asoke` ไม่ได้เรียก `.deposit()` เองเลย — นี่คือพลังของ `Rc<RefCell<T>>` ที่ทำให้ทั้งสองตู้มองเห็น
"บัญชีเดียวกันจริง ๆ" ไม่ใช่สำเนาคนละก้อน (`strong_count = 3` เพราะมีเจ้าของ 3 คน: ตัวแปร `account` ใน `main`,
`atm_siam`, และ `atm_asoke`)

เพื่อให้เห็นภาพว่ายอดเงินและตัวนับเปลี่ยนแปลงไปตามลำดับการเรียกใช้งานอย่างไร มาไล่ทีละบรรทัดเป็นตารางกัน:

| ลำดับ | การกระทำ | balance หลังทำ | strong_count |
|---|---|---|---|
| 1 | `Account::new("สมชาย", 1000.0)` | 1000.00 | 1 (แค่ตัวแปร `account`) |
| 2 | สร้าง `atm_siam` ด้วย `Rc::clone(&account)` | 1000.00 | 2 |
| 3 | สร้าง `atm_asoke` ด้วย `Rc::clone(&account)` | 1000.00 | 3 |
| 4 | `atm_siam.deposit(500.0)` | 1500.00 | 3 (ไม่เปลี่ยน — deposit ไม่กระทบตัวนับ) |
| 5 | `atm_asoke.withdraw(300.0)` | 1200.00 | 3 |
| 6 | `atm_siam.print_balance()` (แค่อ่าน) | 1200.00 | 3 |
| 7 | `atm_asoke.print_balance()` (แค่อ่าน) | 1200.00 | 3 |

สังเกตว่า **คอลัมน์ `strong_count` กับคอลัมน์ `balance` เปลี่ยนแปลงจากสาเหตุที่ต่างกันโดยสิ้นเชิง**: `strong_count`
เปลี่ยนเฉพาะตอนที่มีการสร้าง/ทำลาย `Rc` (ตอนสร้าง `atm_siam`/`atm_asoke` เท่านั้นในตารางนี้) ในขณะที่ `balance`
เปลี่ยนเฉพาะตอนที่มีการเรียก `.deposit()`/`.withdraw()` ผ่าน `.borrow_mut()` ของ `RefCell` — สองอย่างนี้เป็นกลไก
คนละชั้นกันอย่างสิ้นเชิง: **`Rc` จัดการ "ใครเป็นเจ้าของ" ส่วน `RefCell` จัดการ "ข้อมูลถูกแก้ไขเมื่อไหร่"** ตรงตาม
หลักการที่สรุปไว้ในหัวข้อ 28.12

#### เวอร์ชันที่มีบั๊ก: BorrowError จากการ Borrow ซ้อนโดยไม่ตั้งใจ

ทีนี้มาดูบั๊กที่พบได้บ่อยมากในโค้ดจริงที่ใช้ `RefCell<T>`: การเรียก method หนึ่งที่ถือ `.borrow_mut()` ไว้อยู่
แล้วเผลอเรียก method อีกตัวที่ต้อง `.borrow()`/`.borrow_mut()` ซ้อนเข้าไปอีก **ก่อนที่ borrow ตัวแรกจะหมดอายุ**:

```rust
use std::cell::RefCell;
use std::rc::Rc;

struct Account {
    owner: String,
    balance: f64,
}

impl Account {
    fn withdraw(&mut self, amount: f64) -> Result<(), String> {
        if amount > self.balance {
            return Err(format!("ยอดไม่พอ: มี {:.2} แต่ถอน {:.2}", self.balance, amount));
        }
        self.balance -= amount;
        Ok(())
    }
}

struct AtmTerminal {
    location: String,
    account: Rc<RefCell<Account>>,
}

impl AtmTerminal {
    fn print_balance(&self) {
        let acc = self.account.borrow();
        println!("[{}] ยอดคงเหลือของ {}: {:.2} บาท", self.location, acc.owner, acc.balance);
    }

    // เวอร์ชันมีบั๊ก: ถือ borrow_mut() ค้างไว้ในตัวแปร acc ยัง "มีชีวิตอยู่" ตอนเรียก print_balance
    fn withdraw_and_log_buggy(&self, amount: f64) {
        let mut acc = self.account.borrow_mut(); // borrow_mut ตัวที่ 1 เริ่มมีชีวิต ยังไม่ถูก drop
        match acc.withdraw(amount) {
            Ok(()) => {
                self.print_balance(); // ❌ print_balance เรียก .borrow() ขณะที่ acc (borrow_mut) ยังไม่ถูกปล่อย
            }
            Err(e) => println!("[{}] ถอนเงินไม่สำเร็จ: {}", self.location, e),
        }
    }
}

fn main() {
    let account = Rc::new(RefCell::new(Account { owner: "สมชาย".to_string(), balance: 1000.0 }));
    let atm = AtmTerminal { location: "สาขาสยาม".to_string(), account: Rc::clone(&account) };

    atm.withdraw_and_log_buggy(200.0);
}
```

โค้ดนี้ **compile ผ่านสนิท** — ไม่มี warning หรือ error จาก compiler เลยสักตัว เพราะในมุมมองของ borrow checker
`self.account.borrow_mut()` แค่คืนค่า `RefMut<Account>` ธรรมดาที่ผูกกับตัวแปร `acc` ตามกฎ ownership ปกติทุกอย่าง
ไม่มีอะไรผิดกฎ Part 6-7 เลย — **ปัญหาอยู่ที่ runtime ล้วน ๆ** เมื่อรันจริง:

```
thread 'main' panicked at src/main.rs:26:32:
RefCell already mutably borrowed
stack backtrace:
   0: __rustc::rust_begin_unwind
   1: core::panicking::panic_fmt
   2: core::cell::panic_already_mutably_borrowed::do_panic::runtime
   3: core::cell::panic_already_mutably_borrowed::do_panic
   4: core::cell::panic_already_mutably_borrowed
   5: core::cell::RefCell<T>::borrow
   6: AtmTerminal::print_balance
   7: AtmTerminal::withdraw_and_log_buggy
   8: main
note: Some details are omitted, run with `RUST_BACKTRACE=full` for a verbose backtrace.
```

โปรแกรม**ล้ม (panic) ตอนรันจริง** ไม่ใช่ตอน compile — สังเกตจาก stack trace ว่าจุดที่ panic คือ
`RefCell::borrow` ที่ถูกเรียกจาก `print_balance` ซึ่งถูกเรียกมาจาก `withdraw_and_log_buggy` อีกที: `acc` (จาก
`.borrow_mut()`) ยังมีชีวิตอยู่ (ยังไม่หมด scope เพราะ `match acc.withdraw(amount)` ยังไม่จบ) ในขณะที่
`print_balance` พยายามเรียก `.borrow()` ซ้อนเข้าไป — เข้าเงื่อนไข "มี `RefMut` ค้างอยู่ แล้วมีคนขอ `Ref` เพิ่ม"
ตรงตามที่อธิบายไว้ในหัวข้อ 28.7 เป๊ะ ๆ

#### เวอร์ชันที่แก้ไขแล้ว: จำกัด Scope ของ borrow_mut ด้วย Block

วิธีแก้คือจำกัด scope ของ `.borrow_mut()` ให้แคบที่สุด โดยห่อมันไว้ใน block `{}` แยกออกมา แล้วให้ `RefMut` หมดอายุ
(ถูก drop) **ก่อน** ที่จะเรียก `print_balance` ซึ่งต้องการ `.borrow()`:

```rust
use std::cell::RefCell;
use std::rc::Rc;

struct Account {
    owner: String,
    balance: f64,
}

impl Account {
    fn withdraw(&mut self, amount: f64) -> Result<(), String> {
        if amount > self.balance {
            return Err(format!("ยอดไม่พอ: มี {:.2} แต่ถอน {:.2}", self.balance, amount));
        }
        self.balance -= amount;
        Ok(())
    }
}

struct AtmTerminal {
    location: String,
    account: Rc<RefCell<Account>>,
}

impl AtmTerminal {
    fn print_balance(&self) {
        let acc = self.account.borrow();
        println!("[{}] ยอดคงเหลือของ {}: {:.2} บาท", self.location, acc.owner, acc.balance);
    }

    // เวอร์ชันแก้แล้ว: จำกัด scope ของ borrow_mut() ด้วย block {} ให้จบก่อนเรียก print_balance
    fn withdraw_and_log_fixed(&self, amount: f64) {
        let result = {
            let mut acc = self.account.borrow_mut(); // borrow_mut อยู่แค่ในขอบเขตของ block นี้
            acc.withdraw(amount)
        }; // <- RefMut ถูก drop ตรงนี้พอดี ปล่อย borrow_mut ไปแล้วแน่นอน

        match result {
            Ok(()) => self.print_balance(), // ✅ ตอนนี้ borrow_mut ปล่อยไปแล้ว เรียก .borrow() ได้ปลอดภัย
            Err(e) => println!("[{}] ถอนเงินไม่สำเร็จ: {}", self.location, e),
        }
    }
}

fn main() {
    let account = Rc::new(RefCell::new(Account { owner: "สมชาย".to_string(), balance: 1000.0 }));
    let atm = AtmTerminal { location: "สาขาสยาม".to_string(), account: Rc::clone(&account) };

    atm.withdraw_and_log_fixed(200.0);
    atm.withdraw_and_log_fixed(2000.0); // ยอดไม่พอ ควรได้ error message แทน panic
}
```

ผลลัพธ์ (รันจริง — ไม่ panic แล้ว):

```
[สาขาสยาม] ยอดคงเหลือของ สมชาย: 800.00 บาท
[สาขาสยาม] ถอนเงินไม่สำเร็จ: ยอดไม่พอ: มี 800.00 แต่ถอน 2000.00
```

จุดสำคัญที่ทำให้เวอร์ชันนี้ถูกต้อง: `acc.withdraw(amount)` (บรรทัดสุดท้ายในบล็อก) เป็น**นิพจน์** (expression) ที่
ค่าที่ได้ (`Result<(), String>`) ถูกส่งออกจาก block ไปเก็บที่ `result` — แต่ตัวแปร `acc` เอง (ซึ่งเป็น `RefMut`)
**ไม่ได้ถูกส่งออกไปด้วย** มันหมด scope และถูก drop ทันทีที่ block `{}` ปิด ทำให้ `.borrow_mut()` ถูกปล่อยเรียบร้อย
**ก่อน** ที่โค้ดจะไปถึงบรรทัด `match result` ที่เรียก `self.print_balance()` ต่อ — เมื่อถึงตรงนั้น ไม่มี `RefMut`
เหลืออยู่แล้วเลย `self.print_balance()` จึงเรียก `.borrow()` ได้อย่างปลอดภัยแน่นอน

หลักการที่ควรจำจากตัวอย่างนี้ไปใช้ตลอด: **เขียนโค้ดที่ใช้ `RefCell<T>` ให้ทุก `.borrow()`/`.borrow_mut()` มีช่วงชีวิต
สั้นที่สุดเท่าที่เป็นไปได้เสมอ** — ยิ่ง scope ของ `Ref`/`RefMut` แคบเท่าไหร่ ยิ่งลดโอกาสที่มันจะไป "ค้าง" ทับกับ
การยืมอื่นที่เกิดขึ้นในฟังก์ชันที่ถูกเรียกซ้อนกันลงไปอีกชั้น (เหมือนที่เกิดขึ้นในเวอร์ชันบั๊กข้างบน)

### 28.12 สรุปเปรียบเทียบ: เลือกใช้ Smart Pointer ตัวไหนดี

ก่อนไปหัวข้อกับดัก ควรมีตารางสรุปไว้อ้างอิงเร็ว ๆ ว่าแต่ละสถานการณ์ควรเลือกใช้เครื่องมือตัวไหนระหว่าง `Box<T>`
(Part 27), `Rc<T>`, `RefCell<T>`, และ `Rc<RefCell<T>>` ที่เรียนมาทั้งหมดในบทนี้:

| สถานการณ์ | เครื่องมือที่เหมาะสม | เหตุผล |
|---|---|---|
| เก็บข้อมูลบน heap เจ้าของเดียว (เช่น recursive type จาก Part 27) | `Box<T>` | ไม่ต้องมีเจ้าของร่วม ไม่ต้องแก้ไขผ่าน shared reference — เรียบง่ายและเร็วที่สุด |
| ข้อมูลต้องมีเจ้าของหลายคนพร้อมกัน แต่ **ไม่ต้องแก้ไข** หลังสร้างแล้ว (read-only ตลอด) | `Rc<T>` | ได้ shared ownership โดยไม่มี overhead ของการตรวจ borrow rule ตอน runtime เลย |
| ข้อมูลมีเจ้าของเดียว (`&mut self` เข้าถึงได้ตามปกติ) แต่ต้องแก้ไขได้แม้ผ่าน `&self`/binding แบบไม่ `mut` | `RefCell<T>` (หรือ `Cell<T>` ถ้าเป็น type ที่ `Copy` ได้และแก้ทีละค่าทั้งก้อน) | ได้ interior mutability โดยไม่ต้องมีเจ้าของร่วมเลย |
| ข้อมูลต้องมีเจ้าของหลายคน **และ** ทุกคนต้องแก้ไขมันได้จริง (shared mutable state, single-thread) | `Rc<RefCell<T>>` | รวมข้อดีของทั้งสองแบบ — pattern ที่ใช้บ่อยที่สุดสำหรับ shared mutable state แบบไม่มี thread |
| เหมือนข้อบนแต่ต้องทำงานข้ามหลาย thread จริง ๆ | `Arc<Mutex<T>>` (Part 39) | `Rc`/`RefCell` ไม่ thread-safe เลย ต้องใช้เวอร์ชันที่ synchronize ข้าม thread โดยเฉพาะ |
| โครงสร้างมีการอ้างอิงกลับไปมา (เช่น child ชี้กลับไป parent) และต้องการเลี่ยง reference cycle | `Weak<T>` ผสมกับ `Rc<T>` (Part 29) | ตัดวงจร ownership โดยยังคง "รู้จัก" กันไว้ได้ |

หลักการจำง่าย ๆ ที่สรุปทั้งบทนี้ได้ในประโยคเดียว: **`Rc<T>` ตอบคำถาม "ใครเป็นเจ้าของบ้าง" (หลายคนได้) ส่วน
`RefCell<T>` ตอบคำถาม "แก้ไขได้ตอนไหน" (ตรวจตอน runtime แทน compile time) — เมื่อสถานการณ์ต้องการคำตอบทั้งสอง
คำถามพร้อมกัน คุณก็ประกบทั้งสองเข้าด้วยกันเป็น `Rc<RefCell<T>>`**

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. เข้าใจผิดว่า Rc::clone() จะได้สำเนาที่แก้ไขแยกจากต้นฉบับ

มือใหม่จำนวนมากที่คุ้นเคยกับ `.clone()` แบบ deep copy ของ `String`/`Vec<T>` (Part 6/13) มักคาดหวังผิด ๆ ว่า
`Rc::clone()` จะให้ "สำเนาที่เป็นอิสระจากกัน" เหมือนกัน — แต่ในความจริง `Rc::clone()` ให้แค่ตัวชี้ตัวที่สองไปยัง
**ข้อมูลก้อนเดียวกัน**:

```rust
use std::cell::RefCell;
use std::rc::Rc;

fn main() {
    let original = Rc::new(RefCell::new(vec![1, 2, 3]));
    let shared_copy = Rc::clone(&original); // ดูเหมือน "copy" แต่จริง ๆ คือชี้ไปที่ข้อมูลเดียวกัน

    shared_copy.borrow_mut().push(4); // แก้ไขผ่าน shared_copy

    println!("original = {:?}", original.borrow());     // [1, 2, 3, 4] — เปลี่ยนตามไปด้วย!
    println!("shared_copy = {:?}", shared_copy.borrow()); // [1, 2, 3, 4]
}
```

ผลลัพธ์จริง:

```
original = [1, 2, 3, 4]
shared_copy = [1, 2, 3, 4]
```

ถ้าความตั้งใจจริง ๆ คือต้องการสำเนาที่เป็นอิสระ (แก้ไขแยกกันได้) ต้องใช้ `.clone()` บนข้อมูลที่ **อยู่ภายใน** `Rc`
โดยตรง เช่น `(*original.borrow()).clone()` เพื่อ deep clone `Vec<i32>` ออกมาก่อน แล้วค่อยห่อด้วย `Rc::new(RefCell::new(...))`
เป็นคนละก้อนใหม่จริง ๆ — วิธีป้องกันปัญหานี้ที่ดีที่สุดคือ **เขียน `Rc::clone(&x)` แบบ associated function เสมอ**
(ไม่ใช่ `x.clone()`) เพื่อเตือนตัวเองและคนอ่านโค้ดตลอดว่านี่คือการเพิ่มตัวนับ ไม่ใช่การ deep copy

### 2. พยายามแก้ไขข้อมูลผ่าน Rc<T> โดยตรง

```rust
use std::rc::Rc;

fn main() {
    let shared = Rc::new(5);
    *shared += 1;
    println!("{shared}");
}
```

```
error[E0594]: cannot assign to data in an `Rc`
 --> src/main.rs:5:5
  |
5 |     *shared += 1;
  |     ^^^^^^^^^^^^ cannot assign
  |
  = help: trait `DerefMut` is required to modify through a dereference, but it is not implemented for `Rc<i32>`
```

`Rc<T>` implement แค่ `Deref` (อ่านได้) ไม่ implement `DerefMut` (แก้ไขได้) โดยตั้งใจ — เพราะการยอมให้แก้ไขได้ตรง ๆ
ทั้งที่มีเจ้าของหลายคนจะเปิดช่องให้เกิด data race ทันที วิธีแก้คือห่อข้อมูลภายในด้วย `RefCell<T>` (หรือ `Cell<T>`
ถ้าเป็น type ที่ `Copy` ได้) กลายเป็น `Rc<RefCell<T>>` ตามที่เรียนในหัวข้อ 28.8 แล้วแก้ไขผ่าน `.borrow_mut()` แทน

### 3. Borrow ซ้อน borrow_mut (หรือ borrow_mut ซ้อน borrow_mut) จนเกิด Panic ตอนรัน

```rust
use std::cell::RefCell;

fn main() {
    let cell = RefCell::new(5);
    let borrow1 = cell.borrow_mut();
    let borrow2 = cell.borrow_mut(); // ❌ borrow1 ยังไม่หมด scope
    println!("{} {}", borrow1, borrow2);
}
```

```
thread 'main' panicked at src/main.rs:6:24:
RefCell already borrowed
```

และถ้าเป็น `borrow_mut()` ค้างอยู่ แล้วมีคนเรียก `.borrow()` ซ้อนเข้ามา จะได้ panic คนละข้อความ (แต่สาเหตุเดียวกัน):

```
thread 'main' panicked at src/main.rs:6:24:
RefCell already mutably borrowed
```

**นี่คือความเสี่ยงหลักของ `RefCell<T>` ที่ต้องระวังตลอดเวลา** — compiler ไม่ช่วยจับให้เหมือน borrow checker ปกติ
วิธีป้องกันที่ดีที่สุดคือจำกัด scope ของ `Ref`/`RefMut` ให้แคบที่สุดเท่าที่เป็นไปได้เสมอ (ใช้ block `{}` ครอบ)
ตามที่แสดงไว้ในหัวข้อ 28.7 และ 28.11 และหลีกเลี่ยงการเรียก method อื่นของ `self`/struct เดียวกันในขณะที่ยังถือ
`Ref`/`RefMut` ของ field ใน struct นั้นค้างอยู่ ถ้าจำเป็นต้องเรียก ให้แน่ใจว่า borrow เดิมถูกปล่อยไปก่อนแล้วจริง ๆ

### 4. ลืมว่า Rc<T> และ RefCell<T> ใช้ข้าม Thread ไม่ได้

```rust
use std::cell::RefCell;
use std::rc::Rc;
use std::thread;

fn main() {
    let shared = Rc::new(RefCell::new(0));
    let handle = thread::spawn(move || {
        *shared.borrow_mut() += 1;
    });
    handle.join().unwrap();
}
```

```
error[E0277]: `Rc<RefCell<i32>>` cannot be sent between threads safely
 --> src/main.rs:7:32
  |
7 |       let handle = thread::spawn(move || {
  |                    ------------- ^------
  |                    |             |
  |  __________________|_____________within this `{closure@src/main.rs:7:32: 7:39}`
  | |                  |
  | |                  required by a bound introduced by this call
8 | |         *shared.borrow_mut() += 1;
9 | |     });
  | |_____^ `Rc<RefCell<i32>>` cannot be sent between threads safely
  |
  = help: within `{closure@src/main.rs:7:32: 7:39}`, the trait `Send` is not implemented for `Rc<RefCell<i32>>`
note: required because it's used within this closure
note: required by a bound in `spawn`
```

`thread::spawn` ต้องการ closure ที่ implement `Send` (ปลอดภัยที่จะย้ายไปทำงานบน thread อื่น) แต่ `Rc<T>` ไม่
implement `Send` เลย (ตัวนับของมันไม่ thread-safe) — compiler จับข้อผิดพลาดนี้ได้ตั้งแต่ compile time ซึ่งเป็น
ตัวอย่างที่ดีมากว่าระบบ trait ของ Rust (`Send`/`Sync` ที่จะเรียนใน Part 40) ทำงานร่วมกับ ownership system เพื่อ
ป้องกันบั๊กข้าม thread ได้ตั้งแต่ก่อนโปรแกรมรันด้วยซ้ำ วิธีแก้คือเปลี่ยนไปใช้ **`Arc<Mutex<T>>`** (Part 39) ซึ่ง
ออกแบบมาให้ thread-safe โดยเฉพาะ — **ห้ามพยายามใช้ `unsafe` เพื่อ "บังคับ" ให้ `Rc<RefCell<T>>` ทำงานข้าม thread
เด็ดขาด** เพราะตัวนับที่ไม่ synchronize จะทำให้เกิด data race บนตัวนับเองได้จริง (undefined behavior)

### 5. สร้าง Reference Cycle จน Memory รั่วโดยไม่รู้ตัว

การออกแบบ struct ที่มี `Rc<RefCell<T>>` ชี้ไปมาระหว่างกันในหลายทิศทาง (เช่น parent ชี้ไปยัง child และ child ชี้
กลับไปยัง parent ด้วย `Rc` ทั้งสองทาง) จะทำให้ `strong_count` ของทั้งสองฝั่งไม่มีทางลดลงถึง 0 ได้เลย ตามที่อธิบาย
ในหัวข้อ 28.9 — compiler **ไม่มี error หรือ warning ใด ๆ ให้เห็นเลย** เพราะนี่ไม่ใช่ปัญหาด้าน memory safety
(ไม่มี undefined behavior เกิดขึ้น) แต่เป็นปัญหาด้าน resource management ล้วน ๆ วิธีป้องกันคือตั้งคำถามกับตัวเอง
เสมอเมื่อออกแบบโครงสร้างที่มีการอ้างอิงสองทาง: **"ทิศทางไหนคือ ownership จริง (`Rc`) และทิศทางไหนแค่ 'รู้จัก'
กัน (ควรเป็น `Weak`)"** — รายละเอียดวิธีใช้ `Weak<T>` แก้ปัญหานี้จะอยู่ใน Part 29

### 6. ใช้ .unwrap() ตรงตัวกับ Rc::get_mut() แล้วแปลกใจว่า panic

```rust
use std::rc::Rc;

fn main() {
    let a = Rc::new(10);
    let b = Rc::clone(&a); // strong_count = 2 แล้ว
    let mut a = a;
    let value = Rc::get_mut(&mut a).unwrap(); // ❌ panic! เพราะ get_mut คืน None (count > 1)
    *value += 1;
    println!("{b}");
}
```

`Rc::get_mut()` คืน `Option<&mut T>` และจะเป็น `None` เสมอเมื่อมีเจ้าของมากกว่า 1 คน (`strong_count > 1`) ถ้าเผลอ
`.unwrap()` มันตรง ๆ โดยไม่เช็คก่อน โปรแกรมจะ panic ด้วยข้อความมาตรฐานของ `Option::unwrap()` ("called
`Option::unwrap()` on a `None` value") ซึ่งไม่ได้บอกสาเหตุที่แท้จริงชัดเจนเท่า panic ของ `RefCell` เลย — ถ้าคุณ
พบว่าตัวเองต้องใช้ `Rc::get_mut()` บ่อย ๆ ในสถานการณ์ที่มีเจ้าของมากกว่าหนึ่งอยู่เป็นปกติ นั่นคือสัญญาณว่าคุณควร
ใช้ `RefCell<T>` (หรือ `Cell<T>`) ห่อข้อมูลไว้แทน เพราะ `Rc::get_mut()` ถูกออกแบบมาสำหรับกรณีพิเศษที่ค่อนข้างแคบ
(ตอนที่การันตีได้ว่าเหลือเจ้าของเดียวแล้วเท่านั้น) ไม่ใช่เครื่องมือหลักสำหรับ "หลายเจ้าของแก้ไขข้อมูลร่วมกัน"

### 7. Borrow ค้างไว้ตลอดการวนลูปโดยไม่รู้ตัว แล้วพยายาม borrow_mut ซ้อนข้างใน

กับดักนี้พบบ่อยมากในทางปฏิบัติ เพราะดูเผิน ๆ เหมือนไม่มี `.borrow()`/`.borrow_mut()` ซ้อนกันเลย:

```rust
use std::cell::RefCell;

fn main() {
    let numbers = RefCell::new(vec![1, 2, 3]);

    for n in numbers.borrow().iter() {
        if *n == 2 {
            numbers.borrow_mut().push(99); // ❌ borrow() ของ .iter() ยังไม่ปล่อยตลอดการวนลูป
        }
    }

    println!("{:?}", numbers.borrow());
}
```

```
thread 'main' panicked at src/main.rs:8:21:
RefCell already borrowed
```

ต้นเหตุของปัญหาคือ `numbers.borrow()` ในบรรทัด `for n in numbers.borrow().iter()` สร้างค่า `Ref<Vec<i32>>` ขึ้นมา
ก้อนหนึ่ง แล้ว `.iter()` ก็สร้าง iterator ที่ยืม (borrow ในความหมายของ Rust ทั่วไป จาก Part 7) ค่า `Ref` นั้นต่อไป
ตลอดทั้ง loop — พูดง่าย ๆ คือ **`Ref` ตัวนี้มีชีวิตอยู่ตลอดการวนลูปทั้งหมด ไม่ได้หมดอายุทันทีหลังบรรทัดแรกอย่างที่
ตาเปล่ามองแล้วอาจเข้าใจผิด** เพราะ Rust ต้องรักษาให้ iterator ใช้งานได้ตลอด loop ค่าที่มันยืมมาจึงต้องมีชีวิตอยู่
ตลอดไปด้วย — เมื่อโค้ดข้างในพยายามเรียก `.borrow_mut()` ซ้อนเข้าไปอีก (เพื่อ `.push()` สมาชิกใหม่) จึงชนกับ borrow
ที่ยังไม่ปล่อยจาก `.iter()` ทันที

วิธีแก้ที่ตรงไปตรงมาที่สุดคือ **ทำสำเนาของข้อมูลที่จะวนลูปออกมาก่อน** (ถ้าข้อมูลไม่ใหญ่เกินไป) เพื่อให้ `Ref`
หมดอายุทันทีหลัง copy จบ ไม่ต้องมีชีวิตอยู่ตลอด loop อีกต่อไป:

```rust
use std::cell::RefCell;

fn main() {
    let numbers = RefCell::new(vec![1, 2, 3]);

    let snapshot: Vec<i32> = numbers.borrow().clone(); // Ref หมด scope ทันทีหลังบรรทัดนี้
    for n in snapshot {
        if n == 2 {
            numbers.borrow_mut().push(99); // ✅ ไม่มี Ref ค้างอยู่แล้ว ปลอดภัย
        }
    }

    println!("{:?}", numbers.borrow());
}
```

ผลลัพธ์จริง: `[1, 2, 3, 99]` — ไม่ panic อีกต่อไป เพราะ `snapshot` เป็น `Vec<i32>` ธรรมดาที่ไม่ได้ผูกกับ `RefCell`
เลย การวนลูปบน `snapshot` จึงไม่เกี่ยวข้องกับตัวนับของ `numbers` เลยแม้แต่นิดเดียว ข้อแลกเปลี่ยนของวิธีนี้คือต้อง
เสีย cost ของการ clone ข้อมูล — ถ้าข้อมูลใหญ่มากและ clone ไม่คุ้ม อีกทางเลือกคือ เก็บ index ที่ต้องแก้ไขไว้ในตัวแปร
แยกระหว่างวนลูปอ่าน แล้วมาวนลูปแก้ไขทีหลัง**หลังจาก** borrow แรกหมดอายุไปแล้วอย่างสมบูรณ์

## แบบฝึกหัด (Exercises)

1. **โจทย์ระดับง่าย**: เขียน struct `Playlist` ที่มีฟิลด์ `name: String` และ `songs: Rc<Vec<String>>` (สมมติว่า
   playlist หลายรายการสามารถ "อ้างอิง" ไปยัง `Vec<String>` เพลงชุดเดียวกันได้ เช่น "เพลงฮิตประจำสัปดาห์" ที่ถูกใส่
   ไว้ในหลาย playlist พร้อมกัน) เขียน `main()` ที่สร้าง `Rc<Vec<String>>` หนึ่งชุด แล้วสร้าง `Playlist` สองรายการ
   ที่ใช้ `Rc::clone()` ชี้ไปยังชุดเพลงเดียวกัน จากนั้นพิมพ์ `Rc::strong_count()` ออกมาดูก่อนและหลังสร้างแต่ละ
   playlist เพื่อยืนยันว่าตัวนับเปลี่ยนตามที่คาดไว้จริง
   (hint: โครงสร้างเหมือนตัวอย่าง `Config`/`Server` ในหัวข้อ 28.3 เป๊ะ ๆ แค่เปลี่ยนชนิดข้อมูล)

   โครงเริ่มต้น (ไม่ใช่เฉลยเต็ม แค่ช่วยตั้งต้น):
   ```rust
   use std::rc::Rc;

   struct Playlist {
       name: String,
       songs: Rc<Vec<String>>,
   }

   fn main() {
       let weekly_hits = Rc::new(vec![
           "เพลงที่ 1".to_string(),
           "เพลงที่ 2".to_string(),
       ]);
       println!("strong_count เริ่มต้น = {}", Rc::strong_count(&weekly_hits));

       let playlist_a = Playlist { name: "ของ A".to_string(), songs: Rc::clone(&weekly_hits) };
       // TODO: พิมพ์ strong_count หลังสร้าง playlist_a

       let playlist_b = Playlist { name: "ของ B".to_string(), songs: Rc::clone(&weekly_hits) };
       // TODO: พิมพ์ strong_count หลังสร้าง playlist_b

       println!("{}: {:?}", playlist_a.name, playlist_a.songs);
       println!("{}: {:?}", playlist_b.name, playlist_b.songs);
   }
   ```

2. **โจทย์ระดับกลาง**: ต่อยอดจากข้อ 1 — เปลี่ยน `Playlist` ให้ใช้ `Rc<RefCell<Vec<String>>>` แทน `Rc<Vec<String>>`
   เพื่อให้ playlist ใดก็ได้สามารถ `.push()` เพลงใหม่เข้าไปในชุดเพลงที่ใช้ร่วมกันได้ (ผ่าน `.borrow_mut()`) เขียน
   เมธอด `add_song(&self, title: &str)` ให้ `Playlist` แล้วทดสอบว่าเมื่อ playlist ตัวแรกเพิ่มเพลง playlist ตัวที่
   สองก็ต้องเห็นเพลงใหม่นั้นด้วย (พิมพ์เนื้อหาทั้งหมดของทั้งสอง playlist ออกมาเทียบกัน) — สังเกตว่าเมธอด
   `add_song` รับ `&self` ไม่ใช่ `&mut self` แต่ยังแก้ไขข้อมูลที่ใช้ร่วมกันได้จริง
   (hint: โครงสร้างเหมือนตัวอย่าง `Department`/`Budget` ในหัวข้อ 28.8)

   โครงเริ่มต้น (ไม่ใช่เฉลยเต็ม แค่ช่วยตั้งต้น):
   ```rust
   use std::cell::RefCell;
   use std::rc::Rc;

   struct Playlist {
       name: String,
       songs: Rc<RefCell<Vec<String>>>,
   }

   impl Playlist {
       fn add_song(&self, title: &str) {
           // TODO: push ชื่อเพลงใหม่เข้าไปใน self.songs ผ่าน .borrow_mut()
           todo!()
       }
   }

   fn main() {
       let shared_songs = Rc::new(RefCell::new(vec!["เพลงที่ 1".to_string()]));

       let playlist_a = Playlist { name: "ของ A".to_string(), songs: Rc::clone(&shared_songs) };
       let playlist_b = Playlist { name: "ของ B".to_string(), songs: Rc::clone(&shared_songs) };

       playlist_a.add_song("เพลงใหม่จาก A");

       println!("{}: {:?}", playlist_a.name, playlist_a.songs.borrow());
       println!("{}: {:?}", playlist_b.name, playlist_b.songs.borrow());
   }
   ```

3. **โจทย์ระดับยาก**: เขียนโปรแกรมที่ตั้งใจทำให้เกิด panic แบบ `BorrowMutError`/`BorrowError` ขึ้นมาให้เห็นจริง
   ๆ ด้วยตัวเอง (ไม่ใช่ copy จากบทนี้ตรง ๆ แต่ออกแบบสถานการณ์ของตัวเอง เช่น struct `Inventory` ที่มีเมธอด
   `restock(&self)` และ `report(&self)` ที่ทั้งคู่ต้องเข้าถึง `RefCell<Vec<Item>>` เดียวกัน) จากนั้นแก้บั๊กนั้นด้วย
   เทคนิคจำกัด scope ของ `.borrow_mut()`/`.borrow()` ด้วย block `{}` ตามที่เรียนในหัวข้อ 28.7/28.11 แล้วรันซ้ำเพื่อ
   ยืนยันว่าไม่ panic อีกต่อไป — เขียนอธิบายสั้น ๆ (3-5 บรรทัด) ว่าทำไม compiler ไม่จับ error นี้ให้ตั้งแต่แรก
   ทั้งที่มันคือการละเมิดกฎ borrowing แบบเดียวกับที่ Part 7 สอนมา
   (hint: ให้เมธอดหนึ่งถือ `.borrow_mut()` ไว้ในตัวแปรที่ยังไม่ drop แล้วเรียกอีกเมธอดซ้อนเข้าไปข้างใน เหมือน
   `withdraw_and_log_buggy` ในหัวข้อ 28.11)

4. **โจทย์ระดับประยุกต์ใช้งานจริง**: สร้างระบบ "ห้องแชท" (chat room) ขนาดเล็ก — เขียน struct `ChatRoom` ที่มี
   `messages: RefCell<Vec<String>>` แล้วสร้าง struct `User` ที่มีฟิลด์ `name: String` และ `room: Rc<RefCell<ChatRoom>>`
   (ผู้ใช้หลายคนถือ `Rc` ไปยังห้องแชทเดียวกัน) เขียนเมธอด `User::send(&self, text: &str)` ที่เพิ่มข้อความ (พร้อม
   ชื่อผู้ส่ง) ลงใน `messages` ของห้อง และเมธอด `User::read_all(&self)` ที่พิมพ์ข้อความทั้งหมดในห้องออกมา สร้าง
   ผู้ใช้ 3 คนที่ถือห้องแชทเดียวกัน ให้แต่ละคนส่งข้อความคนละ 1-2 ข้อความสลับกัน แล้วให้ทุกคนเรียก `read_all()`
   เพื่อยืนยันว่าทุกคนเห็นข้อความของทุกคนตรงกัน สุดท้ายให้ลองจำลอง reference cycle แบบง่าย ๆ โดยเพิ่มฟิลด์
   `owner: RefCell<Option<Rc<User>>>` เข้าไปใน `ChatRoom` แล้วให้ `ChatRoom` ถือ `Rc<User>` ของผู้ใช้คนแรกที่สร้าง
   ห้อง (ในขณะที่ `User` ก็ถือ `Rc<RefCell<ChatRoom>>` อยู่แล้ว) พิมพ์ `Rc::strong_count()` ของทั้งสองฝั่งก่อนและ
   หลังจบโปรแกรม แล้วอธิบายว่าทำไมตัวเลขนั้นถึงไม่มีวันลดลงถึง 0 แม้ตัวแปรทั้งหมดใน `main()` จะหมด scope ไปแล้ว
   (hint: ข้อสุดท้ายนี้จะบังคับให้ `User` implement บางอย่างเพื่อให้ `ChatRoom` ถือ `Rc<User>` ได้ — ลองคิดดูว่า
   ต้องปรับโครงสร้างยังไงให้ compile ผ่าน และคำตอบสุดท้ายควรจะสอดคล้องกับสิ่งที่อธิบายไว้ในหัวข้อ 28.9 พอดี)

   โครงเริ่มต้น (ไม่ใช่เฉลยเต็ม แค่ช่วยตั้งต้นสำหรับส่วนห้องแชทพื้นฐาน ยังไม่รวมส่วน reference cycle):
   ```rust
   use std::cell::RefCell;
   use std::rc::Rc;

   struct ChatRoom {
       messages: RefCell<Vec<String>>,
   }

   struct User {
       name: String,
       room: Rc<RefCell<ChatRoom>>,
   }

   impl User {
       fn send(&self, text: &str) {
           // TODO: เพิ่มข้อความ (พร้อมชื่อผู้ส่ง) ลงใน self.room.borrow().messages
           todo!()
       }

       fn read_all(&self) {
           // TODO: พิมพ์ข้อความทั้งหมดในห้อง
           todo!()
       }
   }

   fn main() {
       let room = Rc::new(RefCell::new(ChatRoom { messages: RefCell::new(Vec::new()) }));

       let alice = User { name: "Alice".to_string(), room: Rc::clone(&room) };
       let bob = User { name: "Bob".to_string(), room: Rc::clone(&room) };

       alice.send("สวัสดีครับ");
       bob.send("สวัสดีค่ะ");

       alice.read_all();
       bob.read_all();
   }
   ```

## สรุป

บทนี้เริ่มจากคำถามที่ Part 27 ทิ้งไว้: **"ถ้า `Box<T>` มีเจ้าของได้แค่คนเดียว แล้วถ้าเราต้องการเจ้าของหลายคนพร้อม
กันจริง ๆ ล่ะ?"** — เราเห็น error `E0382` จริงที่เกิดขึ้นเมื่อพยายามให้ `Box<T>` มีสองเจ้าของ และเรียนรู้ว่า
`Rc<T>` แก้ปัญหานี้ได้ด้วยการเพิ่ม **reference counting**: `Rc::new` สร้างค่าพร้อมตัวนับเริ่มต้นที่ 1,
`Rc::clone` เพิ่มตัวนับโดย**ไม่ deep copy ข้อมูล**เลย (ต่างจาก `.clone()` ของ `String`/`Vec<T>` อย่างสิ้นเชิง —
ความแตกต่างที่สำคัญที่สุดของบทนี้), และข้อมูลจะถูกทำลายจริงก็ต่อเมื่อ `strong_count` ลดลงถึง 0 เท่านั้น

จากนั้นเราเจอข้อจำกัดของ `Rc<T>`: มันให้แค่ **shared immutable access** — `*shared += 1` compile ไม่ผ่านด้วย
`E0594` เพราะ `Rc<T>` ไม่ implement `DerefMut` โดยตั้งใจ (เพื่อป้องกัน data race) นี่คือจุดที่นำไปสู่ **interior
mutability** ผ่าน `RefCell<T>` — เครื่องมือที่ย้ายการตรวจสอบกฎ borrowing ข้อที่ 1 จาก Part 7 (mutable หนึ่งตัว
หรือ immutable หลายตัว ไม่ทั้งสองพร้อมกัน) **จาก compile time (borrow checker) ไป runtime (ตัวนับภายในของ
RefCell)** — `.borrow()`/`.borrow_mut()` ทำงานเหมือน `&T`/`&mut T` แต่ตรวจสอบผ่านค่าจริงตอนโปรแกรมรัน ถ้าละเมิด
กฎจะได้ panic จริง (`RefCell already borrowed` สำหรับ `BorrowMutError`, `RefCell already mutably borrowed`
สำหรับ `BorrowError`) ไม่ใช่ compile error — trade-off ที่ต้องจำไว้เสมอคือ **ความยืดหยุ่นที่เพิ่มขึ้นแลกมาด้วย
ความรับผิดชอบที่ต้องจัด scope ของ borrow ให้ถูกต้องด้วยตัวเอง**

การรวม `Rc<T>` กับ `RefCell<T>` เป็น **`Rc<RefCell<T>>`** คือ pattern ที่ทรงพลังและพบบ่อยที่สุดในโค้ด Rust
สำหรับ "หลายเจ้าของที่แก้ไขข้อมูลร่วมกันได้จริง" แบบไม่มี thread — เราเห็นตัวอย่างงบประมาณที่ใช้ร่วมกันหลายแผนก
และระบบ ATM ที่หลายตู้เข้าถึงบัญชีเดียวกันได้จริง รวมทั้งเห็นบั๊ก `BorrowError` จริงที่เกิดจากการเรียก method
ซ้อนกันโดยไม่ระวัง scope ของ `.borrow_mut()` และวิธีแก้ด้วยการจำกัด scope ด้วย block `{}` — เราทิ้งท้ายด้วยสอง
เรื่องสำคัญ: **reference cycle** ที่ทำให้ `Rc<RefCell<T>>` รั่วหน่วยความจำได้แบบ safe 100% (ไม่มี `unsafe`
ไม่มี undefined behavior แต่ก็ไม่ปล่อยหน่วยความจำคืนเช่นกัน) ซึ่งจะแก้ด้วย `Weak<T>` ใน Part 29 และ **`Cell<T>`**
ซึ่งเป็นทางเลือกที่เบากว่า `RefCell<T>` สำหรับ type ที่ `Copy` ได้และไม่ต้องการ borrow rule ตรวจสอบเลย

ที่สำคัญที่สุด: **`Rc<RefCell<T>>` คือเวอร์ชัน single-thread ของ `Arc<Mutex<T>>`** ที่จะเรียนใน Part 39 — แนวคิด
เบื้องหลังทั้งสองคู่เหมือนกันเป๊ะ ๆ (smart pointer สำหรับหลายเจ้าของ ห่อด้วย container สำหรับแก้ไขข้อมูลผ่าน
shared reference) เพียงแค่ `Arc`/`Mutex` ต้องรับมือกับหลาย thread พร้อมกันด้วย จึงต้องใช้ atomic operations และ
OS-level lock แทนตัวนับธรรมดา — ถ้าคุณเข้าใจบทนี้แน่น การเรียน concurrency ใน Part 39 จะง่ายขึ้นมากเพราะเป็น
การขยายแนวคิดเดิมที่คุ้นเคยแล้ว ไม่ใช่แนวคิดใหม่ทั้งหมด

ใน **Part 29 (Smart Pointers: Weak<T>, Cow<T>)** เราจะกลับไปแก้ปัญหา reference cycle ที่ทิ้งไว้ในหัวข้อ 28.9
อย่างเต็มรูปแบบด้วย `Weak<T>` (`Rc::downgrade`, `.upgrade()`) พร้อมออกแบบโครงสร้าง tree/graph ที่ถูกต้อง (parent
เป็นเจ้าของ child ด้วย `Rc` แต่ child ชี้กลับไปยัง parent ด้วย `Weak` เพื่อไม่ให้เกิด cycle) และจะแนะนำ `Cow<T>`
("Clone on Write") ซึ่งเป็น smart pointer อีกแบบที่ช่วยเลี่ยงการ clone ข้อมูลที่ไม่จำเป็นในสถานการณ์ที่ข้อมูล
ส่วนใหญ่ไม่ถูกแก้ไข — ปิดท้ายมินิซีรีส์ Smart Pointers (Part 27-29) ที่เริ่มจาก `Box<T>` ตัวที่เรียบง่ายที่สุด
ไปจนถึงเครื่องมือที่ซับซ้อนและทรงพลังที่สุดสำหรับจัดการ ownership ในสถานการณ์ที่ระบบ ownership พื้นฐานจาก
Part 6-7 เพียงอย่างเดียวไม่เพียงพอ

---

**Part ก่อนหน้า:** [Smart Pointers: Box<T>](part-027-smart-pointers-box.md) | **Part ถัดไป:** [Smart Pointers: Weak<T>, Cow<T>](part-029-smart-pointers-weak-cow.md)
