# Part 54: Performance Optimization และ Benchmarking (criterion)

> โมดูล: ระดับสูง (Advanced) | ระดับ: สูง | เวลาโดยประมาณ: 200 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายและปฏิบัติตามกฎเหล็กของงาน performance ได้จริง: **"อย่าเดา วัด"** (don't guess, measure) และอธิบาย
  ได้ว่าทำไมสัญชาตญาณของโปรแกรมเมอร์ (แม้จะมีประสบการณ์มาก) มักผิดพลาดเรื่องความเร็วของโค้ด
- อธิบายได้ว่าทำไมการจับเวลาแบบ naive ด้วย `std::time::Instant` รันครั้งเดียวถึงเชื่อถือไม่ได้ (noise จาก OS
  scheduler, CPU frequency scaling, cache warm-up) และรู้จัก **dead code elimination trap** — กับดักที่
  compiler "optimize" โค้ดที่ผลลัพธ์ไม่ถูกใช้ทิ้งไปทั้งหมดจนได้เวลา "0ns" ที่ไม่มีความหมาย
- ติดตั้งและใช้งาน **`criterion`** crate อย่างถูกต้อง: สร้างไดเรกทอรี `benches/`, ตั้งค่า `[[bench]]` กับ
  `harness = false`, ใช้ macro `criterion_group!`/`criterion_main!`, รันด้วย `cargo bench`, และอ่านผลลัพธ์เชิง
  สถิติ (mean, confidence interval, outlier detection) ที่ criterion คำนวณให้อัตโนมัติ
- ใช้ `std::hint::black_box` (และ `criterion::black_box`) เพื่อป้องกัน compiler "โกง" ด้วยการ optimize
  โค้ดที่กำลังวัดผลทิ้งไปทั้งหมด และอธิบายกลไกที่ทำให้มันทำงานได้
- อ่าน HTML report ของ criterion (`target/criterion/report/index.html`) และใช้กลไก **baseline** เพื่อตรวจจับ
  performance regression เมื่อแก้โค้ด — เชื่อมไปสู่แนวคิด CI/CD ที่จะเรียนเต็มรูปแบบใน **Part 97**
- **วัดจริง** (ไม่ใช่แค่ท่องจำ) ว่าเทคนิค optimization ที่พบบ่อย 5 อย่าง — pre-allocate capacity, หลีกเลี่ยง
  clone ที่ไม่จำเป็น, iterator chain เทียบกับ manual loop, เลือก collection ให้ถูกงาน, และ `#[inline]` hint —
  ให้ผลจริงแค่ไหน ในสถานการณ์แบบไหนที่คุ้มค่า และในสถานการณ์แบบไหนที่ไม่ต่างกันเลย
- เขียน benchmark suite แบบมืออาชีพเทียบ implementation หลายแบบของงานจริง (word frequency counter) และอธิบาย
  เชิงกลไกได้ว่าทำไม implementation ที่เร็วที่สุดถึงเร็ว ไม่ใช่แค่บอกว่า "เร็วกว่า" เฉย ๆ

## ความรู้ที่ต้องมีมาก่อน

- **Part 35 (Cargo ขั้นสูง)**: ต้องเข้าใจ `[profile.release]` อย่างละเอียดมาแล้ว โดยเฉพาะ `opt-level`, `lto`,
  `codegen-units`, และรู้ว่า `cargo bench` ใช้ profile `bench` ที่ inherit ค่าเกือบทั้งหมดมาจาก `release`
  (optimize เต็มที่) — บทนี้ไม่สอน profile ซ้ำ แต่จะใช้ความเข้าใจนั้นเป็นฐานในการอธิบายว่าทำไมตัวเลขที่วัดได้
  จาก `cargo bench` ถึงมีความหมายกับโค้ด production จริง (ต่างจากถ้าไปวัดด้วย debug build ที่ไม่ optimize เลย)
- **Part 17 (Packages, Crates, Workspaces)** และ **Part 32 (Testing: Unit Tests)**: ต้องเข้าใจแนวคิด
  `[dev-dependencies]` มาแล้ว — dependency ที่ใช้ได้แค่ตอน `cargo test`/`cargo bench`/`cargo run` (ไม่ใช่ตอน
  build library ให้คนอื่นใช้จริง) บทนี้จะใส่ `criterion` ไว้ใน `[dev-dependencies]` ด้วยเหตุผลเดียวกันทุก
  ประการกับที่ Part 17 อธิบายไว้
- **Part 33 (Testing: Integration Tests)**: ต้องเข้าใจว่า `tests/` เป็นไดเรกทอรีพิเศษที่ Cargo รู้จักโดย
  อัตโนมัติ (แต่ละไฟล์ compile เป็น crate อิสระ มองเห็นแค่ public API ของ library) — บทนี้จะแสดงให้เห็นว่า
  `benches/` ทำงานตามโครงสร้างแบบเดียวกันเป๊ะ เพียงแต่จุดประสงค์เปลี่ยนจาก "ทดสอบความถูกต้อง" เป็น "วัดความเร็ว"
- **Part 6 (Ownership เบื้องต้น)**: ต้องจำได้ว่า `.clone()` ทำ deep copy จริงและมีต้นทุน (cost) ที่ต้องแลก —
  บทนี้จะเอาแนวคิดนั้นมา**วัดจริง**เป็นตัวเลขที่จับต้องได้ ไม่ใช่แค่พูดลอย ๆ ว่า "clone แพง"
- **Part 13-15 (Collections: `Vec`, `String`, `HashMap`)**: ต้องเข้าใจกลไกภายในของ `Vec`/`String` (capacity,
  การ reallocate แบบ amortized doubling) และของ `HashMap` (hashing, O(1) average lookup เทียบกับ O(n) ของการ
  scan `Vec`) มาแล้ว — บทนี้จะเอากลไกเหล่านั้นมาพิสูจน์ผลกระทบต่อความเร็วจริงด้วยตัวเลข
- **Part 18 (Generics เบื้องต้น)** และ **Part 25-26 (Iterators เบื้องต้น/ขั้นสูง)**: ต้องเข้าใจ monomorphization
  และคำกล่าวอ้างเรื่อง "zero-cost abstraction" ของ iterator ที่ Part 26 อธิบายไว้ในเชิงทฤษฎี — บทนี้จะทดสอบ
  คำกล่าวอ้างนั้นด้วยการวัดจริง ไม่ใช่เชื่อตามที่บอกไว้เฉย ๆ

## เนื้อหา

### 54.1 กฎเหล็กของงาน Performance: "อย่าเดา วัด"

ก่อนจะเรียนเครื่องมือใด ๆ ในบทนี้ ต้องปลูกฝังหลักคิดหนึ่งข้อให้แน่นก่อน เพราะทุกหัวข้อที่เหลือของบทนี้สร้างอยู่บน
หลักคิดนี้ทั้งหมด:

> **ห้ามเชื่อสัญชาตญาณเรื่องความเร็วของโค้ดโดยไม่วัดจริง — ไม่ว่าคุณจะมีประสบการณ์มากแค่ไหนก็ตาม**

นี่ไม่ใช่คำแนะนำทั่วไปแบบลอย ๆ แต่เป็นข้อเท็จจริงที่พิสูจน์ได้ทุกครั้งที่ลองวัดจริง เรามาดูตัวอย่างที่เห็นภาพชัด
ที่สุด: การต่อ string จำนวนมากเข้าด้วยกัน

โปรแกรมเมอร์มือใหม่จำนวนมาก (และมือเก่าบางคนที่ไม่เคยวัดจริง) คิดว่า `format!("{s}{part}")` ในลูปกับ
`s.push_str(part)` ในลูป **ให้ความเร็วใกล้เคียงกัน** เพราะทั้งคู่ "แค่เอาข้อความสองอันมาต่อกัน" ดูเผิน ๆ เหมือน
งานเดียวกัน ต่างกันแค่ syntax ลองมาวัดจริงกันด้วยวิธีที่ง่ายที่สุดก่อน — จับเวลาด้วย `std::time::Instant` ตรง ๆ:

```rust
use std::time::Instant;

fn build_with_format(parts: &[&str]) -> String {
    let mut s = String::new();
    for part in parts {
        s = format!("{s}{part}");
    }
    s
}

fn build_with_push_str(parts: &[&str]) -> String {
    let mut s = String::new();
    for part in parts {
        s.push_str(part);
    }
    s
}

fn main() {
    // สร้างข้อมูลทดสอบ: 3,000 ส่วน ส่วนละ ~12 ตัวอักษร (รวม ~36,000 ตัวอักษร)
    let parts: Vec<&str> = std::iter::repeat("hello-world ").take(3000).collect();

    let start = Instant::now();
    let a = build_with_format(&parts);
    println!("format! loop:  {:?} (len={})", start.elapsed(), a.len());

    let start = Instant::now();
    let b = build_with_push_str(&parts);
    println!("push_str loop: {:?} (len={})", start.elapsed(), b.len());
}
```

รันด้วย `cargo run --release` (สำคัญมาก — ต้องเป็น release build ตาม Part 35 ไม่ใช่ debug build ที่ยังไม่
optimize) ได้ผลลัพธ์จริงบนเครื่องทดสอบ (รันซ้ำ 3 ครั้งเพื่อดูว่าตัวเลขคงที่แค่ไหน):

```
format! loop:  2.195675ms (len=36000)
push_str loop: 12.575µs (len=36000)
format! loop:  2.125163ms (len=36000)
push_str loop: 10.824µs (len=36000)
format! loop:  2.168294ms (len=36000)
push_str loop: 10.275µs (len=36000)
```

ผลลัพธ์ที่ได้คือ **`format!` ในลูปช้ากว่า `push_str` ประมาณ 170-210 เท่า** สำหรับข้อความ 36,000 ตัวอักษร — นี่
ไม่ใช่ความต่างเล็กน้อยแบบที่ "อาจจะไม่ต้องสนใจ" แต่เป็นความต่างระดับสองถึงสามอันดับ (order of magnitude) ที่ถ้า
โค้ดนี้อยู่ใน hot path ของโปรแกรมจริง จะกลายเป็นปัญหา performance ที่ผู้ใช้สัมผัสได้ทันที

**ทำไมความต่างถึงมากขนาดนี้?** จาก Part 13-14 คุณรู้อยู่แล้วว่า `String` เก็บข้อมูลใน buffer บน heap ที่มี
`capacity` ของตัวเอง และ `push_str` จะขยาย buffer แบบ **amortized doubling** (เพิ่ม capacity เป็นสองเท่าเมื่อ
เต็ม) ทำให้จำนวนครั้งที่ต้อง reallocate ทั้งหมดตลอดการต่อ 3,000 ครั้งอยู่ที่ประมาณ `log2(36000) ≈ 16` ครั้งเท่านั้น
— งานโดยรวมคือ **O(n)** เทียบกับความยาวสุดท้าย

ส่วน `format!("{s}{part}")` ทำงานคนละแบบโดยสิ้นเชิง: ทุกครั้งที่เรียก มันจะ **สร้าง buffer ใหม่ทั้งหมด** แล้ว
**copy ข้อความสะสมทั้งหมดที่มีอยู่ (`s`) กลับเข้าไปใหม่** ก่อนต่อ `part` เข้าไปท้ายสุด — ครั้งที่ 1 copy ~12
ตัวอักษร, ครั้งที่ 2 copy ~24 ตัวอักษร, ครั้งที่ 3 copy ~36 ตัวอักษร, ..., ครั้งที่ 3000 copy ~36,000 ตัวอักษร
รวมงาน copy ทั้งหมดคือ `12 + 24 + 36 + ... + 36000` ซึ่งเป็นอนุกรมเลขคณิตที่รวมได้ประมาณ `n²/2` — งานโดยรวมคือ
**O(n²)** เทียบกับความยาวสุดท้าย นี่คือเหตุผลเชิงคณิตศาสตร์ตรง ๆ ที่ทำให้ `format!` ในลูปช้าลงแบบทวีคูณเมื่อ
จำนวนส่วนที่ต่อเพิ่มขึ้น (ลองเพิ่ม `parts` จาก 3,000 เป็น 30,000 ตัว จะพบว่า `push_str` ช้าลงประมาณ 10 เท่า
ตามที่คาด แต่ `format!` จะช้าลงประมาณ **100 เท่า** เพราะเป็น O(n²) — ยิ่งข้อมูลใหญ่ ความต่างยิ่งขยายแบบไม่เป็น
เส้นตรง)

**นี่คือปรัชญาของบทนี้ทั้งบท**: ทุกคำกล่าวอ้างเรื่อง "X เร็วกว่า Y" ที่จะปรากฏต่อจากนี้ **ถูกวัดจริงด้วยเครื่องมือ
ที่เหมาะสม ไม่ใช่แค่อธิบายทฤษฎีแล้วให้เชื่อตาม** — และในหัวข้อถัดไป เราจะเห็นว่าแม้แต่การจับเวลาด้วย `Instant`
แบบข้างบนนี้เอง ก็ยัง**ไม่น่าเชื่อถือพอ**สำหรับงานวัด performance ที่จริงจัง มันให้ภาพกว้าง ๆ ที่ถูกทิศทาง (บอกได้
ว่า `format!` แย่กว่ามาก) แต่ตัวเลขที่แม่นยำระดับที่เอาไปเทียบ implementation ที่ใกล้เคียงกันได้ ต้องใช้เครื่องมือ
ที่ออกแบบมาสำหรับงานนี้โดยเฉพาะ — นั่นคือ `criterion` ที่จะเป็นพระเอกของบทนี้

### 54.2 ทำไม naive timing ด้วย `Instant` เชื่อถือไม่ได้

การจับเวลาด้วย `Instant::now()`/`.elapsed()` แบบหัวข้อ 54.1 มีปัญหาเชิงวิธีวิจัย (methodology) หลายจุดที่ทำให้
ตัวเลขที่ได้ **ไม่น่าเชื่อถือเท่าที่คิด** แม้จะบอกทิศทางถูก (`format!` แย่กว่าจริง) แต่ขนาดของความต่างที่แม่นยำ
ยังมีความคลาดเคลื่อนสูง

#### ปัญหาที่ 1: รันครั้งเดียว ไม่มีการวัดซ้ำเชิงสถิติ

สังเกตจากผลลัพธ์ในหัวข้อ 54.1 ที่รันซ้ำ 3 ครั้ง: `push_str` ได้ `12.575µs`, `10.824µs`, `10.275µs` — ตัวเลข
**ต่างกันในทุกรัน** แม้โค้ดและข้อมูลเหมือนกันทุกประการ ความต่างนี้มาจาก **noise** ของระบบปฏิบัติการและ hardware
ที่ควบคุมไม่ได้:

- **OS scheduler jitter**: ระบบปฏิบัติการอาจสลับไปรัน process อื่นชั่วขณะระหว่างที่โปรแกรมกำลังรันอยู่
  (context switch) ทำให้เวลาที่วัดได้รวม "เวลาที่โปรแกรมไม่ได้ทำงานจริง" เข้าไปด้วยโดยไม่รู้ตัว
- **CPU frequency scaling**: CPU สมัยใหม่ปรับความเร็ว clock ขึ้น-ลงอัตโนมัติตามอุณหภูมิ/การใช้พลังงาน
  (turbo boost, throttling) การรันครั้งแรกอาจเจอ CPU ที่ยังไม่ "boost" ขึ้นไปเต็มที่ ทำให้ช้ากว่าที่ควรจะเป็น
- **Cache warm-up effect**: การรันครั้งแรกของฟังก์ชันหนึ่ง ๆ ข้อมูล/instruction ที่เกี่ยวข้องยังไม่ได้อยู่ใน
  CPU cache (L1/L2/L3) ทำให้ต้องไปอ่านจาก RAM ที่ช้ากว่ามาก การรันซ้ำครั้งต่อ ๆ ไปจะเร็วขึ้นเพราะข้อมูลอยู่ใน
  cache แล้ว — สังเกตว่าผลลัพธ์ข้างบน รันแรกช้าสุด (`2.195675ms`) แล้วค่อย ๆ เร็วขึ้นในรันถัดมา นี่ไม่ใช่
  บังเอิญ

การจับเวลาแค่ครั้งเดียวจึงเหมือนการสุ่มตัวอย่างขนาด 1 จากประชากรที่มี noise — ไม่มีทางรู้ได้เลยว่าตัวเลขที่ได้
คือ "ค่าปกติ" หรือ "ค่าที่โชคดี/โชคร้ายผิดปกติ" เครื่องมือวัด performance ที่ดีต้อง**รันซ้ำหลายครั้งและวิเคราะห์
เชิงสถิติ** (ค่าเฉลี่ย, ส่วนเบี่ยงเบนมาตรฐาน, ตรวจจับ outlier) แทนที่จะเชื่อตัวเลขจากการรันครั้งเดียว

#### ปัญหาที่ 2: Dead Code Elimination — กับดักที่ร้ายแรงกว่า noise

ปัญหาที่ร้ายแรงกว่า noise มาก คือ **compiler อาจ optimize โค้ดที่คุณตั้งใจจะวัดทิ้งไปทั้งหมด** ถ้ามันพิสูจน์ได้
ว่าผลลัพธ์ของการคำนวณนั้น **ไม่ถูกใช้งานที่ไหนต่อเลย** — นี่คือหลักการพื้นฐานของ optimizer ทุกตัว (ไม่ใช่แค่
ของ Rust): ถ้าไม่มีใครสังเกต (observe) ผลลัพธ์ของงานคำนวณหนึ่ง ๆ การคำนวณนั้นก็เท่ากับ **ไม่มีผลต่อพฤติกรรมที่
สังเกตได้ของโปรแกรม** และสามารถลบทิ้งได้อย่างปลอดภัยโดยไม่ทำให้โปรแกรม "ผิด" แม้แต่นิดเดียว

ลองดูตัวอย่างที่ทำให้กับดักนี้เห็นชัดที่สุด — เขียนฟังก์ชันที่ทำงานหนักจริง (บวก square root ของเลข 1 ล้านตัว)
แล้วเรียกมันในลูปโดย**ไม่ใช้ผลลัพธ์ที่ได้เลย**:

```rust
use std::time::Instant;

fn expensive_computation(n: u64) -> f64 {
    let mut sum = 0.0f64;
    for i in 0..n {
        sum += (i as f64).sqrt();
    }
    sum
}

fn main() {
    let start = Instant::now();
    for _ in 0..1000 {
        expensive_computation(1_000_000); // เรียกแล้วไม่เก็บผลลัพธ์ไว้ที่ไหนเลย!
    }
    let elapsed = start.elapsed();
    println!("naive loop (ผลลัพธ์ไม่ถูกใช้เลย): {:?}", elapsed);
}
```

Compile ด้วย `rustc -O` (เทียบเท่า release build) แล้วรัน 3 ครั้ง ได้ผลลัพธ์จริงที่น่าตกใจ:

```
naive loop (ผลลัพธ์ไม่ถูกใช้เลย): 146ns
naive loop (ผลลัพธ์ไม่ถูกใช้เลย): 132ns
naive loop (ผลลัพธ์ไม่ถูกใช้เลย): 108ns
```

**108-146 นาโนวินาที** สำหรับการคำนวณ square root ของเลข 1 ล้านตัว **1,000 รอบ** (รวมแล้วคือ 1 พันล้านครั้งของ
`sqrt` + บวกสะสม) — เป็นไปไม่ได้เลยในทางฟิสิกส์ที่ CPU จะทำงานขนาดนี้ได้ในเวลาระดับนาโนวินาทีเดียว (แค่การอ่าน
RAM หนึ่งครั้งก็ใช้เวลาหลายสิบนาโนวินาทีแล้ว) สิ่งที่เกิดขึ้นจริงคือ **compiler มองเห็นว่า `expensive_computation`
ถูกเรียกแล้วผลลัพธ์ (return value) ไม่ถูกใช้ที่ไหนต่อเลย** (ไม่ถูก `print`, ไม่ถูกเก็บในตัวแปรที่ใช้ต่อ, ไม่ถูก
ส่งกลับจาก `main`) จึงพิสูจน์ได้ว่าการเรียกฟังก์ชันนี้ **ไม่มีผลต่อพฤติกรรมของโปรแกรมเลยแม้แต่นิดเดียว** และลบ
ทั้ง loop กับฟังก์ชันทิ้งไปทั้งหมด — เหลือแค่ลูปเปล่า ๆ (หรือไม่เหลืออะไรเลย) ที่รันจบในเวลาแทบจะเป็นศูนย์

นี่คือ **dead code elimination (DCE) trap** — กับดักที่ทำให้ naive benchmark ให้ผลลัพธ์ที่ "เร็วเกินจริงอย่าง
ไร้สาระ" โดยไม่มีการเตือนใด ๆ จาก compiler เลย โค้ด compile ผ่านปกติ รันได้ปกติ แค่ตัวเลขที่ได้ไม่มีความหมาย
อะไรเกี่ยวกับความเร็วจริงของ `expensive_computation` เลย

**วิธีแก้: `std::hint::black_box`** — ฟังก์ชันในไลบรารีมาตรฐานที่ออกแบบมาเพื่อแก้ปัญหานี้ตรง ๆ มันทำหน้าที่เป็น
"กำแพงทึบ" ที่ compiler **มองไม่ผ่าน**: รับค่าเข้าไปแล้วคืนค่าเดิมออกมา (ทาง logic เหมือน identity function
`x -> x`) แต่ compiler **ไม่รู้ว่าข้างในทำอะไรกับค่านั้นบ้าง** จึงพิสูจน์ไม่ได้ว่าค่าที่ผ่าน `black_box` ไม่ถูก
"สังเกต" จากที่ไหน และไม่กล้าลบการคำนวณที่นำไปสู่ค่านั้นทิ้ง:

```rust
use std::hint::black_box;
use std::time::Instant;

fn expensive_computation(n: u64) -> f64 {
    let mut sum = 0.0f64;
    for i in 0..n {
        sum += (i as f64).sqrt();
    }
    sum
}

fn main() {
    let start = Instant::now();
    for _ in 0..1000 {
        // ครอบทั้ง input และ output ด้วย black_box:
        // - black_box(1_000_000) กันไม่ให้ compiler รู้ค่า n ตอน compile-time แล้วคำนวณล่วงหน้า (constant folding)
        // - black_box(...) รอบนอกกันไม่ให้ compiler พิสูจน์ว่าผลลัพธ์ไม่ถูกใช้ (dead code elimination)
        black_box(expensive_computation(black_box(1_000_000)));
    }
    let elapsed = start.elapsed();
    println!("loop with black_box: {:?}", elapsed);
}
```

รันด้วย `rustc -O` แบบเดียวกัน ได้ผลลัพธ์ที่สมเหตุสมผลกว่าอย่างสิ้นเชิง:

```
loop with black_box: 2.058955595s
loop with black_box: 2.058495925s
loop with black_box: 2.013392891s
```

**~2.0-2.06 วินาที** สำหรับ 1,000 รอบของการคำนวณ `sqrt` 1 ล้านครั้ง — เท่ากับประมาณ 2 มิลลิวินาทีต่อรอบ หรือ
ประมาณ 2 นาโนวินาทีต่อการเรียก `sqrt` หนึ่งครั้ง ซึ่งเป็นตัวเลขที่ **สมเหตุสมผลจริง** สำหรับการคำนวณ floating-point
square root บน CPU สมัยใหม่ (เทียบกับ 108-146 นาโนวินาที**รวม**ในเวอร์ชันที่ไม่มี `black_box` ที่ไม่มีความหมาย
อะไรเลย)

**ข้อสังเกตสำคัญ**: สังเกตว่าตัวเลขจากเวอร์ชันที่มี `black_box` (~2.05 วินาที ในทุกรัน) **นิ่งกว่า**เวอร์ชัน
naive concat ในหัวข้อ 54.1 มาก (ที่แปรผันระหว่าง 10-12µs) เพราะงานนี้ "หนักและสม่ำเสมอ" (เป็น loop คำนวณ CPU
ล้วน ๆ ไม่มี allocation/deallocation แทรก) ทำให้ noise จากปัญหาที่ 1 (scheduler jitter, cache warm-up) มีผล
เป็นสัดส่วนน้อยกว่างานที่ใช้เวลาสั้น ๆ ระดับไมโครวินาที — นี่คือเหตุผลอีกข้อที่ naive timing แบบ single-run ยัง
พอใช้ได้สำหรับงานที่ **ช้ามากจนวัดได้เป็นวินาที** แต่ใช้ไม่ได้เลยสำหรับงานที่เร็วระดับนาโน/ไมโครวินาที ซึ่งเป็น
ระดับความเร็วที่งาน optimization ส่วนใหญ่ต้องเจอจริง ๆ

ทั้งสองปัญหานี้ (noise จากการรันครั้งเดียว และ dead code elimination) คือเหตุผลที่วงการ Rust สร้างเครื่องมือ
เฉพาะทางขึ้นมาแก้ไขทั้งสองปัญหาให้อัตโนมัติ — เครื่องมือนั้นคือ `criterion`

### 54.3 แนะนำ `criterion`: benchmarking ที่ทำถูกต้องตามหลักสถิติ

`criterion` คือ crate benchmarking ที่เป็น **de facto standard** ของวงการ Rust (ไม่ได้อยู่ใน std library แต่
ใช้กันแพร่หลายในระดับที่แทบทุกโปรเจกต์ที่ซีเรียสเรื่อง performance ใช้มัน) มันแก้ปัญหาทั้งสองข้อจากหัวข้อ 54.2
ให้อัตโนมัติ: รันหลายครั้งแล้ววิเคราะห์เชิงสถิติให้ และมี mechanism ป้องกัน dead code elimination ในตัว (ผ่าน
`black_box` ที่ re-export มาให้ใช้ตรง ๆ)

#### 54.3.1 ติดตั้ง: `[dev-dependencies]` ตามที่ Part 17 สอนไว้

`criterion` เป็นเครื่องมือที่ใช้แค่ตอนพัฒนา/วัดผล ไม่ใช่โค้ดที่ต้อง compile เข้าไปใน library ที่แจกจ่ายจริง —
จึงต้องใส่ไว้ใน **`[dev-dependencies]`** ตามหลักการเดียวกันกับที่ Part 17 อธิบายไว้เรื่อง `pretty_assertions`
(ใช้คำสั่งเดียวกับที่เคยใช้ตอนเพิ่ม dev-dependency มาก่อน):

```bash
cargo add criterion --dev --features html_reports
```

คำสั่งนี้เพิ่มบรรทัดต่อไปนี้ลงใน `Cargo.toml` โดยอัตโนมัติ:

```toml
[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }
```

feature `html_reports` เปิดให้ criterion สร้างรายงานเป็นหน้าเว็บ HTML พร้อมกราฟ (จะเห็นในหัวข้อ 54.6) — ถ้าไม่
เปิด feature นี้ criterion จะยังทำงานได้ปกติ (พิมพ์ผลลัพธ์เป็นข้อความในเทอร์มินัล) แค่ไม่สร้างกราฟ HTML ให้

#### 54.3.2 `benches/`: ไดเรกทอรีพิเศษตัวที่สามที่ Cargo รู้จัก

Part 33 สอนไว้ว่า `tests/` เป็นไดเรกทอรีที่ Cargo รู้จักเป็นพิเศษ อยู่ระดับเดียวกับ `src/` ที่ root ของ package
— `benches/` ทำงานตามธรรมเนียมแบบเดียวกันเป๊ะ (เป็นอีกหนึ่งไดเรกทอรีในตระกูลเดียวกับ `tests/` และ `examples/`
ที่ Cargo กำหนดความหมายไว้ล่วงหน้าโดยไม่ต้องประกาศอะไรเพิ่ม):

```
my_project/
├── Cargo.toml
├── src/
│   └── lib.rs        <- ฟังก์ชันที่จะ benchmark ต้องเป็น pub เพื่อให้ไฟล์ใน benches/ เรียกได้
├── tests/             <- (จาก Part 33) integration test — ทดสอบ "ความถูกต้อง" ของ public API
│   └── api_tests.rs
└── benches/           <- ไดเรกทอรีใหม่ของบทนี้ — วัด "ความเร็ว" ของ public API
    └── string_concat_bench.rs
```

**กฎเดียวกับ `tests/` ทุกประการ**: ไฟล์ `.rs` แต่ละไฟล์ที่วางอยู่**โดยตรง**ใต้ `benches/` จะถูก Cargo compile
เป็น **crate อิสระของตัวเอง** ที่มองเห็นได้เฉพาะ **public API** ของ library crate ของคุณ — เหตุผลเชิงเทคนิค
เหมือนกับที่ Part 33 อธิบายไว้เรื่อง integration test: ไฟล์ใน `benches/` ไม่ได้เป็น module ลูกของ `src/lib.rs`
มันคือ crate root ของอีกต้นหนึ่งที่แยกกันสมบูรณ์ ดังนั้นฟังก์ชันที่คุณจะ benchmark **ต้องประกาศเป็น `pub`**
เท่านั้น

#### 54.3.3 `[[bench]]` และ `harness = false`

ต่างจาก `tests/` ที่ Cargo รันไฟล์ทดสอบด้วย **test harness มาตรฐานของ Rust** (ตัวจัดการที่มาจาก `#[test]`
attribute ในตัว `rustc` เอง ไม่ต้องประกาศอะไรเพิ่มใน `Cargo.toml`) ไฟล์ใน `benches/` ที่ใช้ `criterion` ต้อง
**ปิด harness มาตรฐานทิ้ง** เพราะ `criterion` มี harness ของตัวเองที่ทำหน้าที่รันซ้ำหลายครั้ง วิเคราะห์เชิง
สถิติ และพิมพ์รายงาน — ถ้าใช้ harness มาตรฐานของ `rustc` ควบคู่ไปด้วย ทั้งสองระบบจะชนกัน ต้องประกาศ
`[[bench]]` พร้อม `harness = false` ให้ Cargo รู้ว่า "อย่าใช้ built-in test harness กับไฟล์นี้ ให้ `main()`
ของไฟล์นี้เองเป็นตัวควบคุมทั้งหมด":

```toml
[[bench]]
name = "string_concat_bench"   # ต้องตรงกับชื่อไฟล์ benches/string_concat_bench.rs (ไม่มี .rs)
harness = false
```

ถ้าลืมใส่ `harness = false` แล้วรัน `cargo bench` จะเจอ error รูปแบบนี้ (เพราะ `criterion_main!` สร้างฟังก์ชัน
`main()` ของตัวเองไว้แล้ว แต่ built-in harness ของ `rustc` ก็พยายามสร้าง `main()` ของตัวเองด้วย ชนกันตรง ๆ):

```
error[E0152]: duplicate lang item in crate `string_concat_bench`: `start`
```

#### 54.3.4 ตัวอย่างเต็ม: benchmark string concatenation 4 วิธี

มาสร้าง benchmark ที่สมบูรณ์เพื่อวัดตัวอย่างจากหัวข้อ 54.1 ให้แม่นยำกว่า naive `Instant` — คราวนี้เพิ่มอีก 2
วิธีเข้ามาเทียบด้วย (`+` operator และ `push_str` ที่จอง capacity ไว้ล่วงหน้า) รวมเป็น 4 วิธี:

```rust
// src/lib.rs
pub fn build_with_format(parts: &[&str]) -> String {
    let mut s = String::new();
    for part in parts {
        s = format!("{s}{part}");
    }
    s
}

pub fn build_with_add(parts: &[&str]) -> String {
    let mut s = String::new();
    for part in parts {
        s = s + part; // เบื้องหลัง Add<&str> for String เรียก self.push_str(other) แล้วคืน self
    }
    s
}

pub fn build_with_push_str(parts: &[&str]) -> String {
    let mut s = String::new();
    for part in parts {
        s.push_str(part);
    }
    s
}

pub fn build_with_push_str_capacity(parts: &[&str]) -> String {
    let total_len: usize = parts.iter().map(|p| p.len()).sum();
    let mut s = String::with_capacity(total_len);
    for part in parts {
        s.push_str(part);
    }
    s
}
```

```rust
// benches/string_concat_bench.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};
use my_project::{build_with_add, build_with_format, build_with_push_str, build_with_push_str_capacity};

fn bench_string_concat(c: &mut Criterion) {
    // สร้างข้อมูลทดสอบแค่ครั้งเดียวนอก closure ที่ criterion จะเรียกซ้ำ — ไม่อยากให้เวลาสร้างข้อมูลทดสอบ
    // ไปปนกับเวลาที่ต้องการวัดจริง (เวลาของการต่อ string)
    let parts: Vec<&str> = std::iter::repeat("hello-world ").take(3000).collect();

    // benchmark_group รวมหลาย benchmark ที่เทียบกันไว้เป็นกลุ่มเดียว ทำให้ report แสดงเทียบกันในกราฟเดียวได้
    let mut group = c.benchmark_group("string_concat_3000_parts");

    group.bench_function("format!_loop", |b| {
        // b.iter รับ closure ที่จะถูกเรียกซ้ำหลายพันครั้งเพื่อวัดผล — ต้องครอบ input ด้วย black_box
        // เพื่อกัน compiler เห็นว่า parts เป็นค่าคงที่ตอน compile-time แล้ว optimize ทิ้งไปบางส่วน
        b.iter(|| build_with_format(black_box(&parts)))
    });
    group.bench_function("plus_operator_loop", |b| {
        b.iter(|| build_with_add(black_box(&parts)))
    });
    group.bench_function("push_str_no_capacity", |b| {
        b.iter(|| build_with_push_str(black_box(&parts)))
    });
    group.bench_function("push_str_with_capacity", |b| {
        b.iter(|| build_with_push_str_capacity(black_box(&parts)))
    });

    group.finish();
}

criterion_group!(benches, bench_string_concat);
criterion_main!(benches);
```

**อธิบาย macro ทั้งสองตัว**:

- **`criterion_group!(benches, bench_string_concat)`** — สร้างกลุ่มของ benchmark function (ในที่นี้มีแค่
  ตัวเดียวคือ `bench_string_concat` แต่ใส่หลายตัวได้ด้วยการ `criterion_group!(benches, fn1, fn2, fn3)`)
  พร้อมตั้งชื่อกลุ่มว่า `benches` (ชื่อนี้ตั้งเองได้ ไม่ต้องตรงกับอะไร)
- **`criterion_main!(benches)`** — สร้างฟังก์ชัน `main()` ให้ทั้งไฟล์โดยอัตโนมัติ ที่ทำหน้าที่รันทุก
  benchmark ในกลุ่ม `benches` ที่ประกาศไว้ นี่คือ `main()` ตัวเดียวที่ต้องมีในไฟล์นี้ (จึงต้องปิด
  built-in test harness ด้วย `harness = false` ตามหัวข้อ 54.3.3 ไม่ให้มันมาแย่งสร้าง `main()` ซ้ำ)

รันด้วยคำสั่งเดียว:

```bash
cargo bench
```

ผลลัพธ์จริงที่วัดได้ (ตัดเฉพาะส่วนของ benchmark กลุ่มนี้ — คอลัมน์ตัวเลขคือ `[ค่าต่ำสุดของช่วงความเชื่อมั่น
ค่าประมาณกลาง ค่าสูงสุดของช่วงความเชื่อมั่น]`):

```
Benchmarking string_concat_3000_parts/format!_loop
Benchmarking string_concat_3000_parts/format!_loop: Warming up for 3.0000 s
Benchmarking string_concat_3000_parts/format!_loop: Collecting 100 samples in estimated 5.xxxx s
Benchmarking string_concat_3000_parts/format!_loop: Analyzing
string_concat_3000_parts/format!_loop
                        time:   [1.5985 ms 1.6579 ms 1.7423 ms]

Benchmarking string_concat_3000_parts/plus_operator_loop
string_concat_3000_parts/plus_operator_loop
                        time:   [15.467 µs 16.012 µs 16.601 µs]
Found 5 outliers among 30 measurements (16.67%)
  5 (16.67%) high severe

Benchmarking string_concat_3000_parts/push_str_no_capacity
string_concat_3000_parts/push_str_no_capacity
                        time:   [11.039 µs 12.181 µs 13.512 µs]
Found 5 outliers among 30 measurements (16.67%)
  5 (16.67%) high severe

Benchmarking string_concat_3000_parts/push_str_with_capacity
string_concat_3000_parts/push_str_with_capacity
                        time:   [11.347 µs 11.519 µs 11.716 µs]
Found 3 outliers among 30 measurements (10.00%)
  3 (10.00%) high mild
```

> หมายเหตุ: ตัวเลขจริงข้างบนมาจากการรันด้วยค่า config ที่ปรับให้ warm-up/measurement time สั้นลงเพื่อความ
> รวดเร็วตอนทดลอง (ค่า default ของ criterion คือ warm-up 3 วินาที + เก็บตัวอย่างอย่างน้อย 5 วินาทีต่อ
> benchmark ซึ่งแม่นยำกว่าแต่ใช้เวลารันนานกว่ามาก) รูปแบบผลลัพธ์และความหมายของตัวเลขเหมือนกันทั้งสองกรณี

ตัวเลขนี้ **ตรงกับทิศทางที่ naive `Instant` บอกไว้ในหัวข้อ 54.1** (`format!` แย่กว่ามาก) แต่ให้ **ตัวเลขที่แม่นยำ
กว่าและมีช่วงความเชื่อมั่น (confidence interval) ติดมาด้วย** — เห็นชัดเจนว่า:

- `push_str_with_capacity` (11.519 µs) เร็วที่สุด แต่ **ต่างจาก `push_str_no_capacity` (12.181 µs) ไม่มาก**
  (ประมาณ 5%) — สอดคล้องกับที่อธิบายไว้ว่า `push_str` เปล่า ๆ ก็ใช้ amortized doubling อยู่แล้ว การจอง
  capacity ล่วงหน้าช่วยได้แต่ไม่ได้เปลี่ยนความซับซ้อนเชิง algorithm (จะกลับมาขยายความเรื่องนี้ในหัวข้อ 54.7.1)
- `plus_operator_loop` (16.012 µs) ใกล้เคียงกับ `push_str_no_capacity` มาก (ต่างกันไม่ถึง 2x) — สมเหตุสมผล
  เพราะ `Add<&str> for String` เบื้องหลังเรียก `push_str` เช่นกัน (ความต่างเล็กน้อยมาจาก overhead ของการย้าย
  `self` เข้า-ออกจาก `+` operator)
- `format!_loop` (1.6579 ms) ช้ากว่า `push_str_with_capacity` ประมาณ **144 เท่า** (`1658.9 µs ÷ 11.519 µs
  ≈ 144`) — ยืนยันตัวเลขจาก naive timing ในหัวข้อ 54.1 แต่ด้วยความแม่นยำและช่วงความเชื่อมั่นที่ชัดเจนกว่ามาก

#### 54.3.5 `iter_batched`: เมื่อฟังก์ชันที่วัด "เปลี่ยนสภาพ" ข้อมูลของมันเอง

ตัวอย่างทั้งหมดในหัวข้อ 54.3.4 มีจุดร่วมกันหนึ่งอย่าง: ฟังก์ชันที่ถูก benchmark **ไม่ได้เปลี่ยนสภาพ** (mutate)
ข้อมูลที่รับเข้ามา (`build_with_push_str(&parts)` อ่าน `parts` แล้วสร้าง `String` ใหม่ขึ้นมา ไม่ได้แก้ไข `parts`
เดิมเลย) แต่ในโลกจริงมีฟังก์ชันจำนวนมากที่**ทำงานโดยแก้ไขข้อมูลเดิมตรง ๆ** (in-place) เช่น `Vec::sort()`,
`Vec::retain()`, `HashMap::drain()` — ฟังก์ชันแบบนี้เป็นกับดักที่ซ่อนอยู่สำหรับ `b.iter()` แบบธรรมดา

ลองดูตัวอย่าง benchmark การ sort ที่ดู "ปกติ" ทุกประการแต่มีข้อผิดพลาดที่ร้ายแรงซ่อนอยู่:

```rust
use criterion::{black_box, criterion_group, criterion_main, BatchSize, Criterion};

fn shuffled_vec(n: usize) -> Vec<u32> {
    // สร้าง Vec ที่สับเรียงแบบสุ่ม (deterministic pseudo-shuffle ด้วย xorshift เพื่อผลลัพธ์ที่ทำซ้ำได้)
    let mut v: Vec<u32> = (0..n as u32).collect();
    let mut state: u64 = 0xDEADBEEFCAFEBABE;
    for i in (1..v.len()).rev() {
        state ^= state << 13;
        state ^= state >> 7;
        state ^= state << 17;
        let j = (state as usize) % (i + 1);
        v.swap(i, j);
    }
    v
}

fn bench_sort_wrong(c: &mut Criterion) {
    // สร้าง Vec ที่สับเรียงไว้ "แค่ครั้งเดียว" นอก closure
    let mut data = shuffled_vec(10_000);
    c.bench_function("sort_wrong_reused_vec", |b| {
        b.iter(|| {
            data.sort(); // ดูเผิน ๆ เหมือนถูก แต่ data ตัวเดียวกันถูกใช้ซ้ำทุก iteration!
            black_box(&data);
        })
    });
}
```

รันด้วย criterion ได้ผลลัพธ์ที่ **เร็วเกินจริงอย่างน่าสงสัย**:

```
sort_10000_u32/wrong_reused_vec
                        time:   [3.3909 µs 3.4826 µs 3.5836 µs]
```

**3.48 ไมโครวินาที สำหรับการ sort ข้อมูล 10,000 ตัว?** เร็วผิดปกติมากเมื่อเทียบกับความรู้ทั่วไปว่า sorting
algorithm ที่ดีที่สุดมีความซับซ้อน O(n log n) — สิ่งที่เกิดขึ้นจริงคือ: **`b.iter()` เรียก closure ซ้ำหลายร้อย
หรือหลายพันครั้งกับ `data` ตัวเดียวกันไม่เปลี่ยนเลย** รอบแรกที่เรียก `data.sort()` มัน sort ข้อมูลที่สับเรียงไว้
จริง (งานเต็มรูปแบบ) แต่**รอบที่สองเป็นต้นไป `data` เรียงอยู่แล้วจากรอบก่อน!** — `sort()` ของ Rust ใช้อัลกอริทึม
ที่ **รู้จำ pattern ที่เรียงอยู่แล้ว** (adaptive sort ตระกูล merge sort ที่ตรวจจับ "run" ที่เรียงอยู่แล้วและข้าม
งานส่วนนั้นไปได้อย่างรวดเร็ว) ทำให้การ sort ข้อมูลที่เรียงอยู่แล้วใช้เวลาแค่ **O(n)** (ไล่ตรวจสอบครั้งเดียวว่า
เรียงถูกต้องแล้ว) ซึ่งเร็วกว่า O(n log n) ของการ sort ข้อมูลสุ่มจริงมาก — ตัวเลขที่วัดได้ทั้งหมดคือค่าเฉลี่ยของ
"1 ครั้งที่ sort งานหนักจริง" ปนกับ "อีกหลายร้อยครั้งที่แค่ตรวจสอบว่าเรียงแล้ว" ซึ่งไม่ตรงกับสถานการณ์การใช้งาน
จริงเลยแม้แต่นิดเดียว (ในโปรแกรมจริง คุณไม่ได้ sort ข้อมูลเดิมซ้ำ ๆ นับพันครั้ง — คุณ sort ข้อมูล**ใหม่**ที่ยัง
ไม่เรียงทุกครั้ง)

**วิธีแก้: `b.iter_batched()`** — เมธอดของ `Bencher` ที่แยกขั้นตอน **"เตรียมข้อมูล" (setup)** ออกจากขั้นตอน
**"งานที่ต้องการวัดจริง" (routine)** อย่างชัดเจน โดย**ไม่นับเวลาของ setup เข้าไปในผลวัดเลย** — setup ถูกเรียก
ใหม่ทุกรอบ ทำให้แต่ละรอบได้ข้อมูลสดใหม่ (ยังไม่เรียง) เสมอ:

```rust
fn bench_sort_correct(c: &mut Criterion) {
    c.bench_function("sort_correct_iter_batched", |b| {
        b.iter_batched(
            || shuffled_vec(10_000),   // setup: สร้าง Vec สับเรียงใหม่ทุกรอบ (เวลานี้ไม่ถูกนับ)
            |mut d| {                    // routine: รับ ownership ของข้อมูลที่ setup สร้าง แล้ว sort จริง
                d.sort();
                black_box(&d);
            },
            BatchSize::SmallInput,        // hint บอกขนาดงานคร่าว ๆ ให้ criterion ปรับจำนวนรอบให้เหมาะสม
        )
    });
}
```

ผลลัพธ์จริงที่วัดได้คราวนี้:

```
sort_10000_u32/correct_iter_batched
                        time:   [139.99 µs 146.76 µs 153.94 µs]
```

**146.76 ไมโครวินาที — เร็วกว่าเวอร์ชันผิดถึงประมาณ 42 เท่า** (`146.76 ÷ 3.4826 ≈ 42.15`) และเป็นตัวเลขที่
สมเหตุสมผลกว่ามากสำหรับการ sort ข้อมูลสุ่มจริง 10,000 ตัวด้วย O(n log n) algorithm — นี่คือความต่างระดับที่ทำให้
สรุปผลผิดพลาดโดยสิ้นเชิงได้ ถ้าไม่รู้จัก `iter_batched`

**พารามิเตอร์ `BatchSize`** บอก criterion ว่าควรจัดกลุ่ม (batch) การเรียก setup+routine กี่ครั้งต่อหนึ่งการวัด
เวลา (เพื่อลด overhead ของการเรียก setup บ่อยเกินไปสำหรับงานที่เร็วมาก) มีให้เลือกหลักๆ คือ `SmallInput` (ข้อมูล
ขนาดเล็ก setup เร็ว), `LargeInput` (ข้อมูลใหญ่ setup ใช้เวลานาน ควร batch น้อยลง), และ `PerIteration` (บังคับ
เรียก setup ใหม่ทุก iteration เดี่ยว ๆ ไม่ batch เลย — แม่นยำที่สุดแต่มี overhead สูงสุด) เลือกตามลักษณะงานจริง
ของคุณ ถ้าไม่แน่ใจ `SmallInput` เป็นค่าเริ่มต้นที่ปลอดภัยสำหรับกรณีส่วนใหญ่

**กฎการใช้งานจริง**: ทุกครั้งที่ฟังก์ชันที่จะ benchmark **รับ ownership ของ input แล้ว consume หรือ mutate มัน**
(sort in-place, drain, ฟังก์ชันที่ย้าย/บริโภคข้อมูลเข้าไปจนใช้ซ้ำไม่ได้) ให้สงสัยไว้ก่อนว่าต้องใช้ `iter_batched`
แทน `b.iter()` ธรรมดา — สัญญาณเตือนที่ควรระวัง (คล้ายกับ dead code elimination ในหัวข้อ 54.2): **ตัวเลขที่เร็ว
เกินความเป็นไปได้ทางทฤษฎีของ algorithm ที่ใช้** (เช่น sort ได้เร็วกว่า O(n log n) ที่ควรจะเป็นอย่างเห็นได้ชัด)

### 54.4 อ่านผลลัพธ์ของ criterion: สิ่งที่มันทำอัตโนมัติที่ naive timing ทำไม่ได้

ก่อนจะไปต่อ ควรเข้าใจว่าเบื้องหลังตัวเลข `time: [11.347 µs 11.519 µs 11.716 µs]` ที่เห็นในหัวข้อก่อนหน้า
`criterion` ทำอะไรให้อัตโนมัติบ้าง เพื่อแก้ปัญหาที่ 1 จากหัวข้อ 54.2 (noise จากการรันครั้งเดียว):

#### Warm-up phase

ก่อนเริ่มเก็บตัวเลขจริง criterion จะรัน closure ที่ส่งเข้า `b.iter()` ซ้ำไปเรื่อย ๆ เป็นเวลาหนึ่ง (ค่า default
คือ 3 วินาที ปรับได้ด้วย `.warm_up_time(...)`) โดย**ไม่นับผลลัพธ์ช่วงนี้เข้าไปในสถิติเลย** — จุดประสงค์คือให้
ระบบ "อุ่นเครื่อง" ก่อน: ให้ CPU ขึ้น turbo boost เต็มที่ (แก้ปัญหา frequency scaling), ให้ข้อมูล/instruction
ที่เกี่ยวข้องถูกโหลดเข้า CPU cache แล้ว (แก้ปัญหา cache warm-up), และให้ branch predictor ของ CPU "เรียนรู้"
pattern ของโค้ดที่กำลังรันซ้ำ ๆ แล้ว — ทั้งหมดนี้คือปัจจัยที่ทำให้การรันครั้งแรก ๆ (ในตัวอย่าง naive ของหัวข้อ
54.1) ช้ากว่าปกติอย่างเป็นระบบ ไม่ใช่ noise แบบสุ่ม

#### เก็บตัวอย่างจำนวนมาก (samples)

หลัง warm-up เสร็จ criterion จะรัน closure ซ้ำอีกหลายร้อยถึงหลายพันครั้ง (จำนวนรอบขึ้นกับว่า closure หนึ่งครั้ง
ใช้เวลานานแค่ไหน — งานที่เร็วมากอาจถูกจับกลุ่มรันเป็น "batch" หลายครั้งต่อหนึ่ง "sample" เพื่อให้วัดได้แม่นยำ
กว่าการจับเวลาทีละครั้งเดียวที่สั้นเกินกว่าความละเอียดของนาฬิการะบบจะจับได้) แล้วนำเวลาทั้งหมดมาคำนวณสถิติ

#### Confidence interval และ outlier detection

ตัวเลข `[11.347 µs 11.519 µs 11.716 µs]` ไม่ใช่ "ค่าต่ำสุด/เฉลี่ย/สูงสุด" ตรง ๆ แต่คือ **ช่วงความเชื่อมั่น
95%** ที่คำนวณด้วยเทคนิค **bootstrap resampling** (สุ่มเลือกตัวอย่างที่วัดได้มาคำนวณค่าเฉลี่ยซ้ำหลายพันรอบ
เพื่อประมาณว่า "ค่าเฉลี่ยจริง" มีโอกาสอยู่ในช่วงไหนด้วยความมั่นใจ 95%) ตัวเลขกลาง (`11.519 µs`) คือค่าประมาณที่
น่าเชื่อถือที่สุด ส่วนตัวเลขซ้าย-ขวาคือขอบของความไม่แน่นอน — ถ้าช่วงนี้แคบ แสดงว่าการวัดมีความสม่ำเสมอสูง (noise
น้อย) ถ้าช่วงกว้าง แสดงว่าการวัดยังมี noise เยอะอยู่ ควรระวังในการเชื่อตัวเลขกลางมากเกินไป

นอกจากนี้ criterion ยังรายงาน **outlier** ที่ตรวจพบ (เช่น `Found 5 outliers among 30 measurements (16.67%)`)
— ตัวอย่างที่ใช้เวลาผิดปกติไปจากกลุ่มส่วนใหญ่มาก (เช่นเพราะ OS แทรกมา schedule งานอื่นระหว่างวัดพอดี) มันจะ
**ไม่ทิ้งข้อมูลนี้ไปเงียบ ๆ** แต่บอกให้รู้อย่างโปร่งใสว่ามี และใช้เทคนิคทางสถิติที่ทนทานต่อ outlier (robust
statistics) ในการคำนวณช่วงความเชื่อมั่น เพื่อไม่ให้ outlier ไม่กี่ตัวมาทำให้ผลสรุปเพี้ยนไปทั้งหมด

**สรุปสั้น ๆ**: สิ่งที่ naive `Instant` ทำไม่ได้เลยแต่ `criterion` ทำให้อัตโนมัติ คือ (1) warm-up เพื่อขจัด
bias เชิงระบบจากรันแรก ๆ (2) รันซ้ำจำนวนมากพอที่จะคำนวณสถิติได้จริง (3) รายงานความไม่แน่นอนของการวัดอย่าง
ตรงไปตรงมาแทนที่จะให้ตัวเลขเดียวที่ดูแม่นยำกว่าความเป็นจริง (4) ตรวจจับและจัดการ outlier อย่างเป็นระบบ

### 54.5 `black_box`: ป้องกัน compiler "โกง" อย่างเป็นระบบ

หัวข้อ 54.2 แสดงให้เห็นแล้วว่า dead code elimination ทำให้ naive benchmark ให้ผลลัพธ์ที่ผิดพลาดโดยสิ้นเชิงได้
— และวิธีแก้คือ `std::hint::black_box` มาทำความเข้าใจมันให้ลึกกว่านั้นอีกนิด เพราะเวลาเขียน benchmark ด้วย
`criterion` (ที่เห็นในหัวข้อ 54.3.4) คุณยังต้องเรียก `black_box` เองอยู่ดี — `criterion` **ไม่ได้ครอบ
`black_box` ให้อัตโนมัติทุกจุด**

#### `black_box` ทำงานอย่างไรกันแน่

```rust
pub fn black_box<T>(dummy: T) -> T {
    // การ implement จริงใช้เทคนิคระดับ inline assembly เพื่อบอก LLVM ว่า
    // "ค่านี้อาจถูกอ่าน/เขียนจากที่ไหนก็ได้ที่มองไม่เห็น" — ห้าม optimize ข้ามผ่านจุดนี้
}
```

สัญลักษณ์ (signature) ของมันเรียบง่ายมาก: รับค่าประเภท `T` อะไรก็ได้ แล้วคืนค่า `T` แบบเดิม ทาง **logic** มัน
คือ identity function เฉย ๆ แต่ทาง **compiler optimization** มันคือ "กำแพงทึบ": compiler **ไม่รู้**ว่าข้างใน
`black_box` มีการอ่าน/เขียนค่าอะไรเกิดขึ้นบ้าง (ในทาง technical มันบอก LLVM ว่าค่านี้ผ่าน "memory location ที่
มองไม่เห็นจากภายนอก" เข้า-ออก) ทำให้ optimizer **ไม่กล้าสมมติ**ว่าค่านั้นถูกใช้หรือไม่ถูกใช้อีก

ผลที่ตามมาสองทาง:

1. **`black_box(input)` ที่ input** — กัน compiler ไม่ให้รู้ค่าจริงตอน compile-time แล้วคำนวณผลลัพธ์ล่วงหน้า
   (constant folding) เช่นถ้าเขียน `expensive_computation(1_000_000)` ตรง ๆ โดยไม่มี `black_box` ครอบตัวเลข
   `1_000_000` และฟังก์ชันนั้น pure พอที่ LLVM จะพิสูจน์ได้ว่าผลลัพธ์คงที่เสมอ (deterministic ล้วน ๆ ไม่มี I/O)
   ในบางกรณี LLVM อาจคำนวณผลลัพธ์ไว้ล่วงหน้าตอน compile แล้วฝังค่าคงที่นั้นลง binary ตรง ๆ (แม้จะเกิดได้ยากกับ
   loop ขนาดใหญ่แบบนี้ แต่เกิดได้ง่ายกับ input ขนาดเล็ก)
2. **`black_box(output)` ที่ output** — กัน compiler ไม่ให้พิสูจน์ได้ว่าผลลัพธ์ไม่ถูกใช้ที่ไหนต่อ (นี่คือจุดที่
   แก้ dead code elimination ตรง ๆ ตามที่เห็นในหัวข้อ 54.2)

#### พิสูจน์อีกครั้งด้วย object code จริง

ลองตรวจสอบ binary ที่ compile จากเวอร์ชัน naive (ไม่มี `black_box`) ด้วยคำสั่ง `objdump` ของ Unix:

```bash
rustc -O naive.rs -o naive
objdump -d naive --disassemble=expensive_computation
```

```
naive:     file format elf64-x86-64

Disassembly of section .text:
```

**ไม่มีอะไรเลย** — ไม่มีแม้แต่ symbol ของฟังก์ชัน `expensive_computation` ในไฟล์ binary สุดท้าย เพราะมันถูก
inline เข้าไปใน `main` แล้วพบว่าผลลัพธ์ไม่ถูกใช้ จึงลบทั้งฟังก์ชันและการเรียกใช้ทิ้งไปหมดตั้งแต่ก่อนจะถึงขั้นตอน
generate machine code จริง — นี่คือหลักฐานที่ชัดเจนที่สุดว่า naive benchmark ในหัวข้อ 54.2 ไม่ได้วัด "อะไร"
เลยจริง ๆ ไม่ใช่แค่วัดได้ไม่แม่นยำ

#### `criterion::black_box` vs `std::hint::black_box`

เวอร์ชันเก่าของ `criterion` (ก่อน Rust 1.66 ที่เพิ่ม `std::hint::black_box` เข้ามาใน standard library) มี
ฟังก์ชัน `black_box` ของตัวเองที่ re-export ผ่าน `use criterion::black_box;` ปัจจุบัน `criterion::black_box`
ยังใช้ได้อยู่ (เพื่อความเข้ากันได้กับโค้ดเก่า) และภายในก็เรียก `std::hint::black_box` ต่ออยู่ดี — โค้ดตัวอย่าง
ในหัวข้อ 54.3.4 ที่ `use criterion::{black_box, ...}` จึงเทียบเท่ากับการใช้ `std::hint::black_box` ทุกประการ
ทั้งสองแบบใช้แทนกันได้ แต่ในโปรเจกต์ใหม่ ๆ ที่ไม่ได้ผูกกับ `criterion` โดยเฉพาะ (เช่นถ้าจะเขียน micro-benchmark
ง่าย ๆ นอก `criterion`) ควรใช้ `std::hint::black_box` ตรง ๆ เพราะไม่ต้องพึ่ง dependency ภายนอกเลย

#### ข้อควรระวัง: `black_box` ไม่ได้ทำให้โค้ดช้าลงในโปรแกรมจริง

`black_box` เป็นเครื่องมือสำหรับ **benchmark เท่านั้น** — ไม่ควรเอาไปใส่ในโค้ด production จริงเพื่อ "ป้องกัน
optimization" เพราะในโปรแกรมจริง คุณ**ต้องการ**ให้ compiler optimize ให้เต็มที่ (นั่นคือเหตุผลที่มันมีตัวตน
อยู่) การใส่ `black_box` ในโค้ด production จะแค่ **บล็อกการ optimize ที่ควรเกิดขึ้นจริงในโปรแกรมนั้น** ทำให้
โปรแกรมช้าลงโดยไม่ได้ประโยชน์อะไร — ใช้มันแค่ในไฟล์ `benches/` เพื่อจำลองสถานการณ์ที่ผลลัพธ์ "จะถูกใช้จริง"
ในโปรแกรมจริง (ที่ปกติผลลัพธ์จะถูก return, print, เขียนไฟล์, ส่งผ่าน network ฯลฯ ซึ่งเป็นสิ่งที่ compiler
มองไม่ทะลุอยู่แล้วโดยธรรมชาติ — `black_box` แค่จำลองพฤติกรรมนั้นในบริบทของ benchmark ที่ไม่มี "การใช้งานจริง"
ให้เห็น)

### 54.6 HTML Report และการตรวจจับ Performance Regression

#### 54.6.1 HTML Report

เมื่อเปิด feature `html_reports` ไว้ตอนติดตั้ง (หัวข้อ 54.3.1) หลังรัน `cargo bench` เสร็จ criterion จะสร้าง
ไฟล์รายงานเป็นหน้าเว็บ HTML ไว้ที่:

```
target/criterion/report/index.html
```

เปิดไฟล์นี้ด้วย browser จะเห็นหน้าสรุปรวมของทุก benchmark ที่รันไป พร้อมลิงก์เข้าไปดูรายละเอียดของแต่ละกลุ่ม
เช่น `target/criterion/string_concat_3000_parts/report/index.html` ที่แสดง:

- **กราฟ probability density** ของเวลาที่วัดได้ทั้งหมด (เห็นรูปทรงการกระจายตัวของข้อมูล ไม่ใช่แค่ตัวเลขเดียว)
- **กราฟ scatter plot** ของแต่ละ sample ตามลำดับเวลาที่วัด (ช่วยดูว่ามี trend แปลก ๆ เช่นค่อย ๆ ช้าลงระหว่าง
  การวัดหรือไม่ ซึ่งอาจบอกใบ้ถึงปัญหา เช่น thermal throttling ของ CPU)
- **การเทียบกันระหว่าง function ต่าง ๆ ใน benchmark group เดียวกัน** เป็นกราฟแท่งเทียบกัน อ่านง่ายกว่าตัวเลข
  ในเทอร์มินัลมาก โดยเฉพาะเวลามี implementation ให้เทียบมากกว่า 2 แบบ

โครงสร้างไดเรกทอรีของ report:

```
target/criterion/
├── report/
│   └── index.html                          <- หน้าสรุปรวมทุก benchmark
├── string_concat_3000_parts/
│   ├── format!_loop/
│   ├── plus_operator_loop/
│   ├── push_str_no_capacity/
│   ├── push_str_with_capacity/
│   └── report/
│       └── index.html                      <- หน้าเทียบ 4 วิธีในกลุ่มนี้
└── ...
```

#### 54.6.2 Baseline และการตรวจจับ Regression

ความสามารถที่มีประโยชน์จริงมากในทางปฏิบัติคือ criterion **จำผลการรันครั้งก่อนหน้าไว้โดยอัตโนมัติ** (ในไดเรกทอรี
`target/criterion/<benchmark-id>/base/`) แล้วเทียบผลการรันครั้งใหม่กับครั้งก่อนให้ทุกครั้งที่รัน `cargo bench`
ซ้ำ โดยไม่ต้องตั้งค่าอะไรเพิ่มเลย ลองรัน benchmark กลุ่ม prealloc ซ้ำสองครั้งติดกัน (ไม่แก้โค้ดอะไรเลย):

```bash
cargo bench --bench prealloc
```

รันครั้งที่สอง (เทียบกับผลของรันแรกที่เก็บไว้อัตโนมัติ) ให้ผลลัพธ์แบบนี้:

```
vec_push_500000_bigrecord_64bytes/no_with_capacity
                        time:   [11.923 ms 12.356 ms 12.871 ms]
                        change: [-4.1668% -0.0584% +4.8944%] (p = 0.99 > 0.05)
                        No change in performance detected.
```

บรรทัด `change:` คือส่วนสำคัญ — มันเทียบผลรันนี้กับรันก่อนหน้าของ **benchmark เดียวกัน** (ไม่ใช่เทียบกับ
sibling variant อื่นในกลุ่ม!) แล้วบอกว่าเปลี่ยนแปลงไปกี่เปอร์เซ็นต์ พร้อม **p-value** (ค่าที่บอกว่าความ
เปลี่ยนแปลงนี้"มีนัยสำคัญทางสถิติ"หรือเป็นแค่ noise ธรรมดา — โดยทั่วไปถ้า `p > 0.05` แปลว่าความเปลี่ยนแปลงที่
เห็นอาจเป็นแค่ noise เฉย ๆ ยังสรุปไม่ได้ว่าเร็วขึ้น/ช้าลงจริง) ในตัวอย่างนี้ `p = 0.99` (สูงกว่า 0.05 มาก) จึง
สรุปว่า **"No change in performance detected"** ถูกต้องแล้ว เพราะไม่ได้แก้โค้ดอะไรเลยระหว่างสองรัน

**การใช้งานจริง**: ตั้ง baseline ที่มีชื่อไว้ก่อนแก้โค้ด แล้วเทียบหลังแก้:

```bash
# ก่อนแก้โค้ด: บันทึกผลปัจจุบันไว้เป็น baseline ชื่อ "before-refactor"
cargo bench -- --save-baseline before-refactor

# ... แก้โค้ด (เช่น refactor อัลกอริทึมใหม่) ...

# หลังแก้โค้ด: เทียบผลปัจจุบันกับ baseline ที่บันทึกไว้
cargo bench -- --baseline before-refactor
```

ถ้าผลลัพธ์ออกมาเป็น `Performance has regressed.` (แทนที่จะเป็น `No change` หรือ `Performance has improved.`)
นี่คือสัญญาณเตือนที่**วัดได้จริง ไม่ใช่แค่ความรู้สึก**ว่าการแก้โค้ดครั้งนี้ทำให้ช้าลง — มีประโยชน์มากตอน refactor
โค้ดที่อ้างว่า "ไม่กระทบ performance" แต่อยากพิสูจน์จริง ๆ ว่าใช่

**เชื่อมโยงไปข้างหน้า**: กลไก baseline นี้เป็นพื้นฐานสำคัญของการทำ **automated performance regression testing**
ใน CI/CD pipeline — ตั้งให้ทุก pull request รัน `cargo bench` เทียบกับ baseline ของ branch หลัก แล้ว fail
build โดยอัตโนมัติถ้า performance ตกลงเกินเกณฑ์ที่ยอมรับได้ รายละเอียดการตั้ง pipeline แบบนี้ด้วย GitHub
Actions จะได้เรียนเต็มรูปแบบใน **Part 97 (CI/CD Pipeline ด้วย GitHub Actions)**

### 54.7 พิสูจน์เทคนิค Optimization ด้วย Benchmark จริง

ถึงหัวใจของบทนี้ — เทคนิค optimization ที่มักถูกแนะนำกันปากต่อปากในวงการ Rust แต่หลายครั้งไม่มีใครวัดจริงให้ดู
ว่า "ช่วยได้แค่ไหนกันแน่" ในหัวข้อนี้เราจะวัดทุกข้อด้วย `criterion` จริง ๆ

#### 54.7.1 Pre-allocating Capacity: `with_capacity` ช่วยได้แค่ไหนกันแน่?

Part 13-14 สอนไว้ว่า `Vec`/`String` ขยาย capacity แบบ amortized doubling อัตโนมัติเมื่อเต็ม และการจอง
capacity ล่วงหน้าด้วย `Vec::with_capacity`/`String::with_capacity` ช่วยลดจำนวนครั้งที่ต้อง reallocate — คำถาม
คือ **ช่วยได้มากแค่ไหนในทางปฏิบัติ?** คำตอบ (ที่วัดได้จริง) คือ **"ขึ้นอยู่กับขนาดของ element ที่เก็บ"** อย่าง
มีนัยสำคัญ

**กรณีที่ 1: element เล็ก (`u64`, 8 ไบต์) จำนวน 200,000 ตัว**

```rust
fn build_no_capacity(n: u64) -> Vec<u64> {
    let mut v = Vec::new();
    for i in 0..n {
        v.push(i);
    }
    v
}

fn build_with_capacity(n: u64) -> Vec<u64> {
    let mut v = Vec::with_capacity(n as usize);
    for i in 0..n {
        v.push(i);
    }
    v
}
```

ผลลัพธ์จริงจาก criterion:

```
vec_push_200000_u64/no_with_capacity     time: [~199 µs ~207 µs ~211 µs]  (แปรผันระหว่างรัน)
vec_push_200000_u64/with_capacity        time: [~201 µs ~217 µs ~228 µs]  (แปรผันระหว่างรัน)
```

**ผลลัพธ์ที่วัดได้ตรงกันข้ามกับสัญชาตญาณ**: ตัวเลขของทั้งสองวิธีอยู่ในช่วงที่**ทับซ้อนกัน**เมื่อรันซ้ำหลายครั้ง
— บางครั้ง `with_capacity` เร็วกว่านิดหน่อย บางครั้งช้ากว่านิดหน่อย ไม่มีทิศทางที่คงที่ ความต่างอยู่ในระดับ
noise ของการวัด ไม่ใช่ความต่างที่มีนัยสำคัญทางสถิติ

**ทำไม?** เพราะสำหรับ `u64` (8 ไบต์) การขยาย capacity 200,000 ตัวใช้การ reallocate ประมาณ `log2(200000) ≈ 18`
ครั้งเท่านั้น (amortized doubling) และแต่ละครั้ง copy ข้อมูลไม่เกิน 1.6MB (200,000 × 8 ไบต์) — งาน memcpy
ขนาดนี้บน RAM สมัยใหม่ (bandwidth ระดับหลาย GB/s) ใช้เวลาแค่**ไม่กี่ไมโครวินาที**รวมทั้งหมด ในขณะที่งานหลักของ
ลูป (การเรียก `.push()` 200,000 ครั้ง ซึ่งมี bounds check และ capacity check ทุกครั้ง) กิน**เวลาส่วนใหญ่**ของ
ทั้งฟังก์ชันอยู่แล้ว การประหยัด reallocation ไม่กี่ครั้งจึงเป็นสัดส่วนที่เล็กเกินกว่าจะสังเกตได้ชัดเมื่อเทียบ
กับ overhead ของงานหลัก

**กรณีที่ 2: element ใหญ่ (struct 64 ไบต์) จำนวน 500,000 ตัว**

```rust
#[derive(Clone, Copy)]
struct BigRecord {
    data: [u64; 8], // 64 ไบต์ต่อ element
}

fn build_bigrecord_no_capacity(n: u64) -> Vec<BigRecord> {
    let mut v = Vec::new();
    for i in 0..n {
        v.push(BigRecord { data: [i; 8] });
    }
    v
}

fn build_bigrecord_with_capacity(n: u64) -> Vec<BigRecord> {
    let mut v = Vec::with_capacity(n as usize);
    for i in 0..n {
        v.push(BigRecord { data: [i; 8] });
    }
    v
}
```

ผลลัพธ์จริงจาก criterion (คงที่ในหลายรอบการรัน ไม่ทับซ้อนกันเลย):

```
vec_push_500000_bigrecord_64bytes/no_with_capacity     time: [12.070 ms 12.364 ms 12.669 ms]
vec_push_500000_bigrecord_64bytes/with_capacity        time: [2.3286 ms  2.3587 ms  2.3940 ms]
```

**คราวนี้ความต่างชัดเจนและคงที่**: `with_capacity` เร็วกว่าประมาณ **5.2 เท่า** (`12.364 ÷ 2.3587 ≈ 5.24`) —
ช่วงความเชื่อมั่นของทั้งสองวิธี**ไม่ทับซ้อนกันเลย**แม้แต่นิดเดียว ต่างจากกรณี `u64` ที่ทับซ้อนกันหมด

**ทำไมคราวนี้ต่างชัดเจน?** เหตุผลคือ**ต้นทุนการ reallocate แต่ละครั้งแพงขึ้นตามขนาด element**: การ copy
500,000 × 64 ไบต์ = 32MB (ครั้งสุดท้ายที่ใหญ่ที่สุด) ใช้เวลามากกว่าการ copy 1.6MB ของกรณี `u64` อย่างมีนัยสำคัญ
และเมื่อรวมงาน copy จากทุกรอบ reallocation ตลอดการ amortized doubling (ประมาณ 2 เท่าของขนาดสุดท้ายโดยรวม คือ
ราว 64MB ที่ต้องถูก copy ไปมาตลอดกระบวนการสำหรับเวอร์ชันไม่จอง capacity) ต้นทุนนี้ **ใหญ่กว่า overhead ของ
`.push()` เองมาก** จึงกลายเป็นต้นทุนหลักที่ครอบงำเวลารวมทั้งหมด — การจอง `with_capacity` ล่วงหน้าตัดงาน copy
มหาศาลนี้ออกไปเหลือแค่ **0 ครั้ง** (จองพอดีตั้งแต่แรก ไม่ต้องขยายอีกเลย) จึงเห็นผลต่างชัดเจนขนาดนี้

**บทเรียนที่สำคัญกว่าตัวเลข**: การให้คำแนะนำแบบเหมารวมว่า "ใช้ `with_capacity` เสมอเพื่อความเร็ว" นั้น
**ไม่ผิด แต่ไม่สมบูรณ์** — มันเป็น "นิสัยที่ดี ต้นทุนต่ำ (แค่เขียนเพิ่มนิดเดียว) ที่ผลตอบแทนขึ้นอยู่กับสถานการณ์"
ยิ่ง element มีขนาดใหญ่ (หรือมี `Drop` ที่ทำงานหนัก, หรือจำนวนรอบ reallocation สูงกว่าปกติเพราะ pattern การ
เติมข้อมูลไม่สม่ำเสมอ) ยิ่งได้ประโยชน์มาก ในขณะที่ element เล็ก ๆ อาจไม่ต่างเลยในทางปฏิบัติ — **นี่คือเหตุผลที่
ต้องวัดในสถานการณ์ของตัวเองจริง ๆ แทนจะเชื่อคำแนะนำทั่วไปแบบไม่ตรวจสอบ**

#### 54.7.2 หลีกเลี่ยง Clone/Allocation ที่ไม่จำเป็น: `&str` vs `String` param

Part 6 อธิบายไว้ว่า `.clone()` มีต้นทุนต้องแลก — มาวัดจริงว่าต้นทุนนั้นมากแค่ไหน ด้วยการเปรียบเทียบฟังก์ชันที่
ทำงานเหมือนกันทุกประการ ต่างกันแค่ signature: รับ `&str` (ยืม ไม่จัดสรร heap เพิ่ม) เทียบกับรับ `String`
(owned ที่ผู้เรียกซึ่งมีแค่ `&str` ต้อง `.to_string()` ก่อนส่งเข้ามาทุกครั้ง):

```rust
// รับ &str — ไม่มีการจัดสรร heap เพิ่มเลยตอนเรียก ยืมข้อมูลจากผู้เรียกตรง ๆ
fn count_len_borrowed(s: &str) -> usize {
    s.len()
}

fn total_len_borrowed(words: &[&str]) -> usize {
    words.iter().map(|w| count_len_borrowed(w)).sum()
}

// รับ String (owned) — ผู้เรียกที่มีแค่ &str ต้อง .to_string() (clone ข้อมูล) ก่อนส่งเข้ามาทุกครั้ง
fn count_len_owned(s: String) -> usize {
    s.len()
}

fn total_len_owned(words: &[&str]) -> usize {
    words.iter().map(|w| count_len_owned(w.to_string())).sum()
}
```

Benchmark เรียกทั้งสองฟังก์ชันกับ `Vec<&str>` ขนาด 10,000 คำเท่ากัน ผลลัพธ์จริง:

```
total_len_10000_words/borrowed_str_param             time: [3.4686 µs 3.5193 µs 3.5745 µs]
total_len_10000_words/owned_string_param_forces_clone time: [122.31 µs 123.34 µs 124.49 µs]
```

**เวอร์ชันที่รับ `String` owned ช้ากว่าประมาณ 35 เท่า** (`123.34 ÷ 3.5193 ≈ 35.05`) สำหรับงาน 10,000 ครั้ง —
เหตุผลตรงไปตรงมาตามที่ Part 6 อธิบายไว้: `.to_string()` แต่ละครั้ง**จัดสรร heap buffer ใหม่**แล้ว**copy**
ข้อมูลตัวอักษรทั้งหมดเข้าไป ทำ**ทุกครั้ง**ที่เรียกฟังก์ชัน (10,000 ครั้งของการจัดสรร+copy heap) ในขณะที่เวอร์ชัน
`&str` **ไม่จัดสรรอะไรเลยแม้แต่ครั้งเดียว** — ส่งแค่ pointer + length (16 ไบต์บน 64-bit system) ผ่านไปตรง ๆ

**บทเรียนเชิงออกแบบ API**: ถ้าฟังก์ชันไม่ได้ต้องการ**เก็บ** (own) ข้อมูลไว้ใช้ต่อหลังจบฟังก์ชัน (เช่นเก็บไว้ใน
struct, ส่งเข้า thread อื่น, ฯลฯ) การรับ `&str` แทน `String` (หรือ `&[T]` แทน `Vec<T>` โดยทั่วไป) เป็นทางเลือก
ที่ถูกต้องแทบทุกครั้ง — มันให้ผู้เรียกเลือกได้ว่าจะยืมหรือจะ clone (ถ้าผู้เรียกมี `String` อยู่แล้วก็ยืมผ่าน
`&s` ได้โดยไม่มีต้นทุนเพิ่ม แต่ถ้าฟังก์ชันเรียกร้อง `String` ตรง ๆ ผู้เรียกที่มีแค่ `&str` จะ**ไม่มีทางเลือกอื่น
นอกจาก clone เสมอ**) นี่คือเหตุผลที่ Rust idiomatic code มักเห็นฟังก์ชันรับ `&str`/`&[T]` เป็นพารามิเตอร์
มากกว่า owned type เว้นแต่มีเหตุผลชัดเจนว่าต้องเก็บข้อมูลไว้ใช้ต่อจริง ๆ

#### 54.7.3 Iterator Chain vs Manual Loop: พิสูจน์คำกล่าวอ้าง "Zero-Cost"

Part 25-26 อธิบายไว้ว่า iterator adapter ของ Rust เป็น **zero-cost abstraction** — compile ออกมาเป็นโค้ดที่
เร็วเทียบเท่า (ไม่ช้ากว่า) การเขียนลูปมือเอง เพราะ monomorphization (Part 18) ทำให้ทุก closure/adapter ถูก
inline จนไม่เหลือ overhead ของ abstraction เลย มาพิสูจน์คำกล่าวอ้างนี้ด้วยงานจริง — หาผลรวมของกำลังสองของเลข
คู่ในช่วง `0..5_000_000`:

```rust
fn sum_iter(n: u64) -> u64 {
    (0..n).filter(|x| x % 2 == 0).map(|x| x * x).sum()
}

fn sum_loop(n: u64) -> u64 {
    let mut total = 0u64;
    for x in 0..n {
        if x % 2 == 0 {
            total += x * x;
        }
    }
    total
}
```

ผลลัพธ์จริงจาก criterion (`n = 5_000_000`):

```
sum_squares_even_5_000_000/iterator_chain     time: [1.0581 ms 1.1493 ms 1.2329 ms]
sum_squares_even_5_000_000/manual_for_loop    time: [2.6548 ms 2.7364 ms 2.8594 ms]
```

**ผลลัพธ์ที่วัดได้ไม่ใช่แค่ "เท่ากัน" ตามที่คำว่า zero-cost อาจทำให้คาดหวัง — iterator chain กลับ**เร็วกว่า
manual loop ประมาณ 2.4 เท่า** (`2.7364 ÷ 1.1493 ≈ 2.38`) ในการวัดครั้งนี้!** ช่วงความเชื่อมั่นของทั้งสองฝั่ง
ไม่ทับซ้อนกันเลย ยืนยันว่าความต่างนี้ไม่ใช่ noise

**ทำไม iterator เร็วกว่าลูปมือเองในกรณีนี้?** คำอธิบายที่สมเหตุสมผลที่สุด (ตรวจสอบได้ด้วยการดู assembly ที่
generate ออกมา ถ้าอยากลงลึกกว่านี้) คือรูปแบบโค้ดของ iterator chain (`filter` → `map` → `sum`) เป็น pattern
ที่ **LLVM คุ้นเคยและ optimize ได้ดีเป็นพิเศษ** — มันสามารถแปลงงาน "กรองแล้วคำนวณ" ให้เป็นโค้ดแบบ **branchless**
(ใช้ compare + select instruction แทนการ jump ตามเงื่อนไข) และ **vectorize** (ประมวลผลหลายค่าพร้อมกันด้วย SIMD
instruction) ได้ง่ายกว่า เพราะโครงสร้างของ iterator adapter ไม่มี control-flow ที่ซับซ้อนปนเข้ามาให้ optimizer
ต้องวิเคราะห์เยอะ ในขณะที่ลูปมือเขียนที่มี `if` ซ้อนอยู่ข้างในนั้น แม้ pattern ของเงื่อนไข (`x % 2 == 0`) จะ
**ทำนายได้ง่าย**สำหรับ branch predictor ของ CPU (สลับ true/false ทุกตัวเป๊ะ ๆ) แต่ตัว **compiler เองอาจไม่
กล้า vectorize** ลูปที่มี control flow แบบนี้เท่ากับ pattern ของ iterator chain ที่มันจดจำรูปแบบได้ชัดกว่า

**บทเรียนสำคัญที่สุดของหัวข้อนี้ไม่ใช่ตัวเลข 2.4 เท่า** แต่คือ **ผลลัพธ์นี้เป็นเรื่องที่เดาไม่ได้ล่วงหน้าโดย
ไม่วัด** — สัญชาตญาณของโปรแกรมเมอร์จำนวนมาก (โดยเฉพาะที่มาจากภาษาอื่นที่ abstraction มีต้นทุนจริง) คือ "ลูปมือ
เขียนต้องเร็วกว่าหรืออย่างน้อยเท่ากันเสมอ เพราะมัน low-level กว่า" แต่การวัดจริงพิสูจน์ว่า**ไม่จำเป็นต้องเป็น
แบบนั้นเลย** — นี่คือการยืนยันปรัชญาของบทนี้ทั้งบทอีกครั้ง: อย่าเดา วัด และในกรณีนี้ การวัดยังให้ผลลัพธ์ที่
**ดีกว่า**สิ่งที่คำว่า "zero-cost" (ซึ่งแปลตรงตัวว่า "ไม่แพงกว่า") บอกไว้เสียอีก — เป็นไปได้ที่ abstraction
ระดับสูงจะช่วยให้ compiler มองเห็น pattern และ optimize ได้**ดีกว่า**โค้ดที่เขียนมือแบบตรงไปตรงมาด้วยซ้ำ

> ข้อควรระวัง: อย่าตีความผลลัพธ์นี้ว่า "iterator เร็วกว่า loop เสมอ" — บางกรณี (โดยเฉพาะ loop ที่มี logic ซับซ้อน
> จนเขียนเป็น iterator chain ได้ไม่สวยงาม หรือ pattern ที่ LLVM ไม่คุ้นเคย) ผลอาจกลับกันหรือเท่ากันก็ได้ ข้อสรุป
> ที่ปลอดภัยที่สุดคือ: **ทั้งสองแบบมักได้ performance ที่ใกล้เคียงหรือดีกว่ากันแบบไม่แน่นอนทิศทางตายตัว
> ดังนั้นควรเลือกเขียนแบบที่อ่านง่าย/สั้น/สื่อความหมายชัดกว่า (ปกติคือ iterator chain) แล้ววัดเฉพาะจุดที่สงสัย
> จริง ๆ ว่าเป็น bottleneck** (จะเรียนวิธีหา bottleneck จริงใน Part 55)

#### 54.7.4 เลือก Collection ให้ถูกงาน: `Vec` Linear Search vs `HashMap` Lookup

Part 15 อธิบายไว้ว่า `HashMap` ให้ lookup แบบ O(1) โดยเฉลี่ย เทียบกับ `Vec::iter().any()`/`.contains()` ที่เป็น
O(n) — แต่คำถามที่ Part 15 ไม่ได้ตอบคือ **"จุดตัด (crossover point) อยู่ที่ไหน"** เพราะ Big-O บอกแค่ทิศทางของ
การเติบโต ไม่ได้บอก **ค่าคงที่** (constant factor) ที่ซ่อนอยู่ — `HashMap` มี overhead จากการคำนวณ hash ทุก
ครั้งที่ lookup ในขณะที่ `Vec` linear scan ไม่มี overhead แบบนั้นเลย (แค่ไล่เทียบค่าไปเรื่อย ๆ) ทำให้สำหรับ
collection ที่**เล็กมาก** ความเร็วดิบของการ scan ตรง ๆ (ที่ CPU cache-friendly มาก เพราะข้อมูลอยู่ติดกันเป็น
เนื้อเดียว) อาจชนะ overhead ของการ hash ได้

```rust
fn linear_contains(data: &[u32], needle: u32) -> bool {
    data.iter().any(|&x| x == needle)
}

fn hashmap_contains(map: &HashMap<u32, ()>, needle: u32) -> bool {
    map.contains_key(&needle)
}
```

Benchmark ทำ lookup 200 ครั้งต่อรอบ (ครึ่งเจอครึ่งไม่เจอ) เทียบสอง scale: collection ขนาดเล็กมาก (8 ตัว) กับ
ขนาดใหญ่ (10,000 ตัว):

```
lookup_n8_200_queries/vec_linear_search        time: [897.87 ns 910.68 ns 926.24 ns]
lookup_n8_200_queries/hashmap_lookup           time: [1.8796 µs 1.8899 µs 1.9007 µs]

lookup_n10000_200_queries/vec_linear_search    time: [356.71 µs 374.53 µs 396.26 µs]
lookup_n10000_200_queries/hashmap_lookup       time: [2.0383 µs 2.1139 µs 2.2199 µs]
```

ผลลัพธ์แสดงจุดตัดที่ชัดเจนมาก:

| ขนาด Collection | `Vec` linear search | `HashMap` lookup | ผู้ชนะ |
|---|---|---|---|
| n = 8 | 910.68 ns | 1,889.9 ns | **`Vec` เร็วกว่า ~2.08 เท่า** |
| n = 10,000 | 374.53 µs | 2,113.9 ns | **`HashMap` เร็วกว่า ~177 เท่า** |

สำหรับ `n = 8`: **`Vec` linear search เร็วกว่า `HashMap` ถึง 2 เท่า** — ต้นทุนคงที่ของการคำนวณ hash function
(ที่ `HashMap` ต้องทำทุกครั้งไม่ว่า collection จะเล็กแค่ไหน) แพงกว่าการไล่เทียบค่า 8 ตัวตรง ๆ ที่ทั้งหมดอยู่ใน
CPU cache line เดียวหรือสองบรรทัดติดกัน (การ scan ข้อมูลที่ต่อเนื่องกันในหน่วยความจำ = cache-friendly มาก
เทียบกับการกระโดดไปมาตาม hash bucket ที่กระจายอยู่คนละที่ = cache-unfriendly กว่าสำหรับ collection ขนาดเล็ก)

สำหรับ `n = 10,000`: สถานการณ์**กลับกันโดยสิ้นเชิง** — `HashMap` เร็วกว่า `Vec` ถึง **177 เท่า** เพราะต้นทุนของ
`HashMap` ยังคงที่ไม่ขึ้นกับขนาด (ยัง ~2.1 µs เหมือนตอน n=8 เกือบทุกประการ) ในขณะที่ `Vec` linear search ต้อง
ไล่เทียบค่าเฉลี่ยครึ่งหนึ่งของ 10,000 ตัวต่อการ lookup หนึ่งครั้ง (ตาม 200 queries) ทำให้เวลาโตขึ้นเป็นสัดส่วน
ตรงกับขนาด collection ตามที่ Big-O ทำนายไว้เป๊ะ

**บทเรียนสำคัญ**: Big-O เพียงอย่างเดียว**ไม่พอ**สำหรับการตัดสินใจเลือก collection ในโค้ดจริง — ต้องรู้ **ขนาด
ข้อมูลจริงที่จะเจอ** ด้วย ถ้าคุณกำลังเขียนโค้ดที่ lookup ใน collection ขนาดเล็กมาก ๆ เป็นประจำ (เช่น
"รายการ status ที่เป็นไปได้ 5-6 แบบ", "field ที่ valid ในฟอร์ม 10 ช่อง") การใช้ `Vec`/array ตรง ๆ อาจเร็วกว่า
`HashMap` จริง แม้ Big-O จะบอกว่า `HashMap` "ดีกว่า" ก็ตาม — จุดตัดที่แท้จริงสำหรับโปรแกรมของคุณ (อาจเป็น 15,
50, หรือ 200 ขึ้นอยู่กับ hash function ที่ใช้, ขนาด element, และ hardware) **ต้องวัดเอาเองในสถานการณ์ของคุณ**
เพราะขึ้นกับปัจจัยเยอะเกินกว่าจะมีตัวเลขมาตรฐานตายตัวที่ใช้ได้ทุกที่

#### 54.7.5 `#[inline]` Hint: ช่วยได้จริงไหม?

`#[inline]` คือ attribute ที่บอก compiler ว่า "ลองพิจารณา inline ฟังก์ชันนี้เข้าไปในจุดที่เรียกใช้" — แต่ต้อง
เข้าใจให้ชัดตั้งแต่แรกว่ามันเป็นแค่ **hint** (คำแนะนำ) ไม่ใช่คำสั่งบังคับ (มี `#[inline(always)]` ที่ใกล้เคียง
คำสั่งบังคับกว่า แต่ compiler ก็ยังปฏิเสธได้ในบางกรณี เช่นฟังก์ชัน recursive)

**ความจริงที่ควรรู้ก่อน**: **LLVM's inliner** (ตัวตัดสินใจ inline ของ backend ที่ `rustc` ใช้) เก่งมากอยู่แล้ว
โดย default สำหรับฟังก์ชันเล็ก ๆ ที่เรียกจาก**ภายใน crate เดียวกัน** — ไม่ต้องใส่ `#[inline]` เพิ่มเลยก็ตาม เพราะ
มันมองเห็นเนื้อในของฟังก์ชันทั้งหมดอยู่แล้ว (จาก Part 35 หัวข้อ LTO: การ optimize ข้าม crate boundary ต้องพึ่ง
LTO หรือ `#[inline]` แต่ **ภายใน crate เดียวกัน ไม่มีข้อจำกัดแบบนั้นเลย** LLVM inline ได้เต็มที่โดยอัตโนมัติ)

`#[inline]` จึงมีผลจริงชัดเจนที่สุดในสถานการณ์เดียว: **เมื่อฟังก์ชันถูกเรียก "ข้าม crate boundary"** (จาก
library หนึ่งไปอีก library หนึ่ง) **โดยไม่ได้เปิด LTO** — เพราะโดย default `rustc` จะ**ไม่ export** ตัว MIR/LLVM
IR ของฟังก์ชันธรรมดา (ไม่ใช่ generic) ออกไปให้ crate อื่นเห็น ทำให้ crate ที่เรียกใช้ทำได้แค่เรียกผ่าน function
call จริง (ไม่มีทาง inline ได้เลยไม่ว่า optimizer จะฉลาดแค่ไหน) — การใส่ `#[inline]` บอก `rustc` ให้ export
MIR ของฟังก์ชันนั้นออกไปด้วย เปิดโอกาสให้ crate อื่น inline มันเข้าไปได้แม้ไม่เปิด LTO

มาวัดจริงด้วยสองฟังก์ชันที่เหมือนกันทุกประการ ต่างกันแค่ attribute อยู่ใน crate คนละตัวกับที่เรียกใช้:

```rust
// crate "helper" — src/lib.rs

/// ไม่มี #[inline] — โดย default รัสต์จะไม่ export MIR ของฟังก์ชันนี้ข้าม crate (ยกเว้นเปิด LTO)
pub fn calc_noinline(x: u64) -> u64 {
    x.wrapping_mul(2654435761).wrapping_add(1)
}

/// มี #[inline] — บอก compiler ให้ฝัง MIR ไว้ให้ crate อื่น inline ข้ามได้แม้ไม่เปิด LTO
#[inline]
pub fn calc_inline(x: u64) -> u64 {
    x.wrapping_mul(2654435761).wrapping_add(1)
}
```

```rust
// crate "app" (เรียก helper ข้าม crate boundary) — benches/inline_hint.rs
use criterion::black_box;

fn run_noinline(n: u64) -> u64 {
    let mut acc = 0u64;
    for i in 0..n {
        acc = acc.wrapping_add(helper::calc_noinline(black_box(i)));
    }
    acc
}

fn run_inline(n: u64) -> u64 {
    let mut acc = 0u64;
    for i in 0..n {
        acc = acc.wrapping_add(helper::calc_inline(black_box(i)));
    }
    acc
}
```

ผลลัพธ์จริงจาก criterion (`n = 2_000_000` ครั้งของการเรียกข้าม crate):

```
cross_crate_call_2_000_000/without_inline_attribute    time: [698.35 µs 738.48 µs 788.64 µs]
cross_crate_call_2_000_000/with_inline_attribute       time: [676.47 µs 683.26 µs 691.59 µs]
```

`#[inline]` ช่วยได้จริง แต่ **แค่ประมาณ 7.5%** (`738.48 ÷ 683.26 ≈ 1.081`) — ช่วยได้จริงตามทฤษฎี แต่ผลที่ได้
**เล็กกว่าที่หลายคนคาดหวังไว้มาก** เหตุผลคือฟังก์ชันตัวอย่างนี้เล็กมาก (มีแค่ multiply กับ add) ต้นทุนของการ
"เรียกฟังก์ชันจริง" (function call overhead: push return address, jump, ฯลฯ) แม้จะมีอยู่จริงในเวอร์ชันไม่มี
`#[inline]` แต่ก็เล็กมากเมื่อเทียบกับงานอื่นทั้งหมดในลูป (การเข้าถึง `black_box`, การวน loop เอง)

**ข้อสรุปที่ตรงไปตรงมาที่สุด**: `#[inline]` **ไม่ใช่ปุ่มวิเศษที่ทำให้เร็วขึ้นเสมอ** และในโค้ดส่วนใหญ่ (ฟังก์ชัน
ที่เรียกภายใน crate เดียวกัน ซึ่งเป็นกรณีส่วนใหญ่ของโค้ด application ทั่วไป) **ไม่จำเป็นต้องใส่เลย** เพราะ LLVM
จัดการให้ดีอยู่แล้วโดยอัตโนมัติ ควรใส่เฉพาะเมื่อ (1) วัดแล้วเจอว่ามันช่วยจริงในสถานการณ์ของคุณ (เช่นฟังก์ชัน
เล็ก ๆ ที่ export เป็น public API ของ library แล้วถูกเรียกบ่อยมากจาก crate อื่นในลักษณะ hot path) หรือ (2) คุณ
กำลังเขียน library ที่อยากเปิดโอกาสให้ downstream crate inline ฟังก์ชันเล็ก ๆ ของคุณได้โดยไม่ต้องพึ่งให้ผู้ใช้
เปิด LTO เอง — การใส่ `#[inline]` แบบสุ่มไปทุกฟังก์ชันโดยไม่วัดผล มีข้อเสียแฝงด้วย: ฟังก์ชันที่ถูก inline ไป
หลายจุดทำให้ **ขนาด binary ใหญ่ขึ้น** (code bloat) และในบางกรณีอาจทำให้ **instruction cache (icache) ทำงาน
แย่ลง** เพราะโค้ดที่ถูก duplicate ไปหลายจุดกินพื้นที่ icache มากกว่าโค้ดที่เรียกผ่าน function call ครั้งเดียว

### 54.8 Premature Optimization: วัดก่อน แล้วค่อย Profile หา Bottleneck จริง

หลังจากเห็นเทคนิคทั้งหมดในหัวข้อ 54.7 ต้องเน้นย้ำหลักคิดสำคัญอีกข้อที่มักถูกมองข้าม: **การ optimize จุดที่ไม่ใช่
bottleneck จริงของโปรแกรม คือการเสียเวลาเปล่า และบางครั้งยังทำร้ายโค้ดโดยไม่ได้ประโยชน์อะไรเลย**

Donald Knuth เขียนไว้ในบทความชื่อดังปี 1974 ว่า *"premature optimization is the root of all evil"* — คำพูดนี้
มักถูกยกมาใช้แบบผิดบริบท (บางคนใช้เป็นข้อแก้ตัวไม่สนใจ performance เลย) แต่บริบทจริงของ Knuth คือ: โปรแกรมเมอร์
มักเสียเวลามหาศาลไป optimize โค้ดใน**จุดที่ไม่สำคัญ** (ตามสัดส่วนเวลารันจริงของทั้งโปรแกรม) โดยไม่รู้ตัว เพราะ
**เดา**ว่าจุดนั้นคือ bottleneck โดยไม่เคยวัดจริงว่าเวลาส่วนใหญ่ของโปรแกรมไปอยู่ที่ไหนกันแน่

ลองนึกภาพโปรแกรมที่มี 3 ฟังก์ชัน: ฟังก์ชัน A ใช้เวลา 1% ของเวลารันทั้งหมด ฟังก์ชัน B ใช้เวลา 5% และฟังก์ชัน C
ใช้เวลา 94% (เช่นเพราะทำ network request แบบ blocking หรือ query database ที่ไม่มี index) — ถ้าคุณใช้เวลาทั้ง
สัปดาห์ไป optimize ฟังก์ชัน A ให้เร็วขึ้น 10 เท่า (สมมติว่าทำได้จริง) โปรแกรมโดยรวมจะเร็วขึ้นแค่ **0.9%**
เท่านั้น (จาก 1% ลดลงเหลือ 0.1%) — ในขณะที่การไป optimize ฟังก์ชัน C แค่ 20% (ซึ่งอาจทำได้ง่ายกว่าด้วยซ้ำ เช่น
เพิ่ม index ให้ database) จะทำให้โปรแกรมโดยรวมเร็วขึ้นถึง **18.8%** — ผลตอบแทนต่างกันมหาศาลจากความพยายามที่ใกล้
เคียงกัน

**คำถามคือ: แล้วจะรู้ได้อย่างไรว่าฟังก์ชันไหนคือฟังก์ชัน C ของโปรแกรมจริง?** เครื่องมือที่ตอบคำถามนี้เรียกว่า
**profiler** — มันสังเกตโปรแกรมที่กำลังรันจริง (ไม่ใช่ isolated function แบบที่ `criterion` benchmark) แล้ว
รายงานว่าเวลาทั้งหมดถูกใช้ไปที่ฟังก์ชันไหนกี่เปอร์เซ็นต์ ทำให้รู้ได้ว่า**ควร**เอาเครื่องมือของบทนี้ (criterion)
ไปโฟกัสวัด/ปรับปรุงตรงจุดไหนของโปรแกรมจริง แทนที่จะเดาสุ่ม ๆ ว่าโค้ดตรงนี้ "ดูน่าจะช้า"

**ความสัมพันธ์ระหว่างสองบทนี้**: บทนี้ (Part 54) สอน**วิธีวัดว่า "โค้ดชิ้นนี้เร็วแค่ไหน เทียบกับอีกชิ้นนึงเร็ว
กว่ากันแค่ไหน"** — ตอบคำถาม **"เร็วแค่ไหน" (how fast)** ส่วนบทถัดไป **Part 55 (Profiling Rust Applications)**
จะสอนวิธีตอบคำถามที่มาก่อนคำถามนั้นเสมอ: **"เวลาไปอยู่ที่ไหนกันแน่ในโปรแกรมทั้งตัว" (where)** — ลำดับการ
ทำงานที่ถูกต้องในโลกจริงคือ **profile ก่อนเพื่อหาว่าจุดไหนคือ bottleneck จริง แล้วค่อยใช้เทคนิคจาก benchmark
(อย่างบทนี้) ไปวัด/ปรับปรุงเฉพาะจุดนั้น** ไม่ใช่ไล่ optimize ทุกฟังก์ชันในโปรแกรมแบบไม่มีลำดับความสำคัญ

**ข้อเสียอีกข้อของ premature optimization ที่มักถูกมองข้าม**: โค้ดที่ optimize มากเกินจำเป็นมักอ่านยากขึ้น
(เช่นเปลี่ยนจาก iterator chain ที่อ่านง่ายเป็นลูปมือเขียนที่ซับซ้อนกว่า โดยหวังผล performance ที่ (ตามหัวข้อ
54.7.3) อาจไม่ได้ดีขึ้นจริงด้วยซ้ำ) — ถ้าจุดที่ถูก optimize นั้นไม่ใช่ bottleneck จริง คุณจะได้ **โค้ดที่อ่านยาก
ขึ้นโดยไม่ได้ประโยชน์ด้าน performance ที่สังเกตได้จริงเลยแม้แต่นิดเดียว** — เป็นการแลกที่ขาดทุนล้วน ๆ

### 54.9 ตัวอย่างจริงจัง: Word Frequency Counter — เทียบ 4 Implementation

มาปิดท้ายด้วยตัวอย่างที่ประกอบทุกแนวคิดของบทนี้เข้าด้วยกัน — เขียนโปรแกรมนับความถี่ของคำในข้อความยาว (คล้าย
ธีมของ word counter ใน Part 14-15 แต่คราวนี้เราจะ**วัดจริง**ว่า implementation แบบไหนเร็วที่สุด และ**อธิบาย
เชิงกลไก**ว่าทำไมมันถึงเร็ว)

**สถานการณ์**: ข้อความ 300,000 คำ จากคำศัพท์ไม่ซ้ำกัน 300 คำ (แปลว่าแต่ละคำซ้ำโดยเฉลี่ยประมาณ 1,000 ครั้ง —
เหมือนสถานการณ์จริงของข้อความภาษาธรรมชาติที่คำศัพท์ทั่วไปซ้ำกันบ่อยมาก)

#### Implementation 1: Naive — `HashMap::new()` ธรรมดา

```rust
use std::collections::HashMap;

pub fn wf_naive(text: &str) -> HashMap<String, u32> {
    let mut map = HashMap::new();
    for word in text.split_whitespace() {
        *map.entry(word.to_string()).or_insert(0) += 1;
    }
    map
}
```

โค้ดนี้ดูสะอาดและเป็น idiomatic Rust ทั่วไป — แต่มีจุดที่มองข้ามได้ง่าย: **`word.to_string()` ถูกเรียกทุกครั้ง
ที่วนลูป ไม่ว่าคำนั้นจะเคยเจอมาก่อนหรือไม่** เพราะ `HashMap::entry()` ต้องการ **owned key** เป็น argument เสมอ
(ไม่ว่า key นั้นจะมีอยู่ในแมพแล้วหรือไม่ก็ตาม — ถ้ามีอยู่แล้ว key ที่สร้างมาใหม่นี้จะถูกใช้แค่เพื่อ "หา" ตำแหน่ง
ใน map แล้วถูกทิ้งไปเลยหลังจากนั้น การจัดสรร heap เพื่อสร้างมันขึ้นมาจึงสูญเปล่าโดยสิ้นเชิงในกรณีที่ key มีอยู่
แล้ว)

#### Implementation 2: Pre-sized Capacity — `HashMap::with_capacity`

```rust
pub fn wf_with_capacity(text: &str, cap: usize) -> HashMap<String, u32> {
    let mut map: HashMap<String, u32> = HashMap::with_capacity(cap);
    for word in text.split_whitespace() {
        *map.entry(word.to_string()).or_insert(0) += 1;
    }
    map
}
```

เหมือน Implementation 1 ทุกประการ เพิ่มแค่การจอง capacity ล่วงหน้า (`cap = 600` คือสองเท่าของ 300 คำศัพท์ที่
ไม่ซ้ำ เผื่อพื้นที่ตาม load factor ที่ `HashMap` ต้องการ)

#### Implementation 3: Avoid Allocation on Hit — ตรวจก่อนค่อยจัดสรร

```rust
pub fn wf_avoid_alloc_on_hit(text: &str, cap: usize) -> HashMap<String, u32> {
    let mut map: HashMap<String, u32> = HashMap::with_capacity(cap);
    for word in text.split_whitespace() {
        if let Some(count) = map.get_mut(word) {
            // เจอ key นี้อยู่แล้ว — get_mut รับ &str ยืมมาเทียบ ไม่ต้องจัดสรร heap เลย
            *count += 1;
        } else {
            // ไม่เจอ (ครั้งแรกที่พบคำนี้) — จัดสรร heap แค่ตอนนี้เท่านั้น
            map.insert(word.to_string(), 1);
        }
    }
    map
}
```

จุดสำคัญคือ `HashMap::get_mut()` รับ `&str` (ยืม ไม่ต้องเป็น owned) เพื่อ**ค้นหา**ตำแหน่งใน map — ถ้าคำนั้นมี
อยู่แล้ว (ซึ่งเกิดขึ้นเป็นส่วนใหญ่ในสถานการณ์นี้ เพราะ 300,000 คำจากคำศัพท์แค่ 300 คำ แปลว่ามีแค่ 300 ครั้งจาก
300,000 ครั้งเท่านั้นที่เป็นการเจอคำใหม่จริง ๆ) โค้ดจะ**ไม่จัดสรร heap เลย** เรียก `.to_string()` (จัดสรร heap)
ก็เฉพาะกรณี "ไม่เจอ" เท่านั้น ซึ่งในสถานการณ์นี้เกิดขึ้นแค่ 300 ครั้งจาก 300,000 ครั้ง (0.1% ของ iteration
ทั้งหมด)

#### Implementation 4: Collect ก่อนเป็น `Vec<&str>` แล้วค่อยวน (ตั้งใจเพิ่ม allocation ที่ไม่จำเป็น)

```rust
pub fn wf_collect_first_then_loop(text: &str, cap: usize) -> HashMap<String, u32> {
    // ตั้งใจเพิ่มขั้นตอน collect เป็น Vec<&str> ก่อน (allocation กลางทางที่ไม่จำเป็น)
    let words: Vec<&str> = text.split_whitespace().collect();
    let mut map: HashMap<String, u32> = HashMap::with_capacity(cap);
    for word in words {
        if let Some(count) = map.get_mut(word) {
            *count += 1;
        } else {
            map.insert(word.to_string(), 1);
        }
    }
    map
}
```

เหมือน Implementation 3 ทุกประการ (ใช้ logic "avoid alloc on hit" แบบเดียวกัน) แต่เพิ่มขั้นตอน `.collect()`
เป็น `Vec<&str>` ก่อนวนลูป — เพื่อจำลองสถานการณ์ที่พบได้บ่อยในโค้ดจริง: โปรแกรมเมอร์บางคนเคยชินกับการ "เก็บผลลัพธ์
ของ iterator ไว้ในตัวแปรก่อน แล้วค่อยวน" (เผื่อจะใช้ซ้ำ, เผื่อ debug print ได้ง่ายกว่า) โดยไม่รู้ตัวว่าขั้นตอนนี้
เพิ่ม allocation ที่ไม่จำเป็นเข้ามาถ้าไม่ได้ใช้ `words` ซ้ำจริง ๆ (การ `.collect()` เป็น `Vec<&str>` ของ 300,000
ตัวชี้ (fat pointer ตัวละ 16 ไบต์) ต้องจัดสรร heap buffer ขนาด ~4.8MB และเขียนข้อมูลลงไปเต็มก่อน — ทั้งหมดนี้
เป็นงานที่ `split_whitespace()` เดิมทำให้ได้แบบ **lazy** (คำนวณทีละคำตามที่ลูปต้องการ ไม่ต้องเก็บทั้งหมดไว้ก่อน)
อยู่แล้ว)

#### วัดจริงด้วย criterion

```rust
// benches/word_freq.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};

fn bench_word_freq(c: &mut Criterion) {
    let text = generate_corpus(300, 300_000); // 300 คำศัพท์ไม่ซ้ำ, รวม 300,000 คำ
    let cap = 300 * 2;

    let mut group = c.benchmark_group("word_frequency_300000_words");
    group.bench_function("naive_hashmap_new", |b| {
        b.iter(|| my_project::wf_naive(black_box(&text)))
    });
    group.bench_function("hashmap_with_capacity", |b| {
        b.iter(|| my_project::wf_with_capacity(black_box(&text), cap))
    });
    group.bench_function("with_capacity_avoid_alloc_on_hit", |b| {
        b.iter(|| my_project::wf_avoid_alloc_on_hit(black_box(&text), cap))
    });
    group.bench_function("collect_vec_first_then_loop", |b| {
        b.iter(|| my_project::wf_collect_first_then_loop(black_box(&text), cap))
    });
    group.finish();
}

criterion_group!(benches, bench_word_freq);
criterion_main!(benches);
```

ผลลัพธ์จริงที่วัดได้:

```
word_frequency_300000_words/naive_hashmap_new                  time: [15.017 ms 15.674 ms 16.498 ms]
word_frequency_300000_words/hashmap_with_capacity               time: [15.061 ms 15.669 ms 16.405 ms]
word_frequency_300000_words/with_capacity_avoid_alloc_on_hit    time: [9.0538 ms 9.1476 ms 9.2330 ms]
word_frequency_300000_words/collect_vec_first_then_loop         time: [11.428 ms 11.677 ms 11.991 ms]
```

สรุปเป็นตารางเทียบกัน (เรียงจากเร็วสุดไปช้าสุด):

| อันดับ | Implementation | เวลา (ค่าประมาณกลาง) | เทียบกับตัวเร็วสุด |
|---|---|---|---|
| 1 | `with_capacity_avoid_alloc_on_hit` | 9.1476 ms | 1.00x (เร็วสุด) |
| 2 | `collect_vec_first_then_loop` | 11.677 ms | 1.28x ช้ากว่า |
| 3 | `hashmap_with_capacity` | 15.669 ms | 1.71x ช้ากว่า |
| 4 | `naive_hashmap_new` | 15.674 ms | 1.71x ช้ากว่า |

**วิเคราะห์เชิงกลไกว่าทำไมผลลัพธ์ออกมาแบบนี้**:

1. **`hashmap_with_capacity` แทบไม่ต่างจาก `naive_hashmap_new` เลย (15.669 ms vs 15.674 ms)** — ตรงกับสิ่งที่
   อธิบายไว้ในหัวข้อ 54.7.1: การจอง capacity ช่วยเรื่อง **การขยาย `HashMap` เมื่อเต็ม** แต่ในตัวอย่างนี้ map
   สุดท้ายมีแค่ **300 key ที่ไม่ซ้ำกัน** — การ rehash ของ `HashMap` ขนาดเล็กขนาดนี้ (ไม่กี่ครั้งตลอดโปรแกรม)
   มีต้นทุนที่**เล็กจิ๋วมาก**เมื่อเทียบกับต้นทุนที่ครอบงำอยู่จริง (ดูข้อ 2) — นี่คือตัวอย่างที่ชัดเจนว่า
   "pre-allocate capacity" เป็นเทคนิคที่ถูกต้องแต่**ใช้ไม่ถูกจุด**ในกรณีนี้ เพราะปัญหาจริงไม่ได้อยู่ที่การขยาย
   ขนาดของ `HashMap` เลย
2. **ต้นทุนที่ครอบงำอยู่จริงคือ "จำนวนครั้งที่จัดสรร heap"**: `naive_hashmap_new`/`hashmap_with_capacity`
   เรียก `word.to_string()` **ทุกครั้ง**ที่วนลูป (300,000 ครั้ง) แม้ว่า 299,700 ครั้งในนั้น (99.9%) จะเป็นการ
   เจอคำที่มีอยู่แล้วในแมพ (เพราะมีคำศัพท์ไม่ซ้ำแค่ 300 คำ) — การจัดสรร heap 300,000 ครั้งนี้เองคือสิ่งที่กิน
   เวลาส่วนใหญ่ของทั้งสอง implementation
3. **`with_capacity_avoid_alloc_on_hit` เร็วกว่าประมาณ 1.71 เท่า เพราะลดจำนวนการจัดสรร heap จาก 300,000 ครั้ง
   เหลือแค่ 300 ครั้ง** (เจาะจงจัดสรรก็ตอน "ไม่เจอ key" เท่านั้น ซึ่งเกิดขึ้นแค่ครั้งแรกที่พบคำใหม่แต่ละคำ) — นี่
   คือการ optimize ที่ **ตรงจุดจริง** เพราะไปลดสิ่งที่เป็นต้นทุนหลักตรง ๆ (ต่างจาก `with_capacity` ธรรมดาที่ไป
   optimize จุดที่ไม่ใช่ bottleneck)
4. **`collect_vec_first_then_loop` ใช้ logic "avoid alloc on hit" เหมือนกันเป๊ะกับตัวที่เร็วสุด แต่ช้ากว่า
   28%** เพราะเพิ่มขั้นตอน `.collect::<Vec<&str>>()` ที่จัดสรร heap buffer ก้อนใหญ่ (~4.8MB สำหรับ 300,000
   fat pointer) ขึ้นมาโดยไม่ได้ใช้ประโยชน์อะไรเพิ่มเลย (ไม่ได้เอา `words` ไปใช้ซ้ำที่ไหน) — สอนบทเรียนสำคัญว่า
   **การแก้ปัญหาหนึ่งจุด (ลด allocation ตอนใส่ข้อมูลลง HashMap) ไม่ได้แปลว่าโค้ดทั้งฟังก์ชันปราศจาก allocation
   ที่ไม่จำเป็นแล้ว** ต้องมองทั้ง data flow ตั้งแต่ต้นจนจบ ไม่ใช่แค่จุดที่กำลังโฟกัสอยู่

**สรุปบทเรียนของตัวอย่างนี้**: implementation ที่เร็วที่สุดไม่ได้ชนะเพราะ "เทคนิคเดียวที่ขลังที่สุด" แต่ชนะเพราะ
มัน**ระบุถูกว่าต้นทุนหลักที่แท้จริงของงานนี้คือจำนวนครั้งของการจัดสรร heap ไม่ใช่ขนาดของ `HashMap`** แล้วแก้ไข
ปัญหานั้นตรงจุด — นี่คือแก่นของ Part 55 ที่กำลังจะพูดถึงต่อไป (หา bottleneck จริงก่อน) ผสมกับแก่นของบทนี้ (วัด
จริงเพื่อพิสูจน์ว่าการแก้ไขนั้นได้ผลจริงแค่ไหน)

## กับดักที่พบบ่อย (Common Pitfalls)

**1. เขียน micro-benchmark โดยไม่ใช้ `black_box` แล้วได้ผลลัพธ์ "เร็วเกินจริงอย่างไร้สาระ" (Dead Code
Elimination Trap)**

นี่คือกับดักที่ร้ายแรงที่สุดเพราะ**ไม่มี error หรือ warning เตือนเลย** — โค้ด compile ผ่าน รันได้ปกติ แค่ตัวเลข
ที่ได้ไม่มีความหมายอะไร สัญญาณเตือนที่ควรระวังคือ **ตัวเลขที่เร็วเกินความเป็นไปได้ทางฟิสิกส์** เช่นถ้า benchmark
บอกว่างานที่ควรใช้เวลาระดับมิลลิวินาที (มี loop หรือ allocation จำนวนมาก) กลับวัดได้แค่ไม่กี่นาโนวินาที นั่นคือ
สัญญาณชัดเจนว่าโค้ดถูก optimize จนเหลือศูนย์ (หรือใกล้ศูนย์) วิธีป้องกัน: **ครอบทั้ง input ที่ส่งเข้าฟังก์ชันและ
output ที่ได้กลับมาด้วย `black_box` เสมอ** ในทุก closure ที่ส่งเข้า `b.iter()` ของ criterion ไม่มีข้อยกเว้น
แม้จะรู้สึกว่า "ฟังก์ชันนี้มี side effect อยู่แล้ว ไม่น่าถูก optimize ทิ้ง" ก็ตาม เพราะการพิสูจน์ของ compiler
บางครั้งลึกกว่าที่คาดคิด (เช่นเรื่อง loop-idiom recognition ที่แปลงลูปเป็น closed-form formula ที่กล่าวถึงใน
หัวข้อ 54.2)

**2. รัน `cargo bench` โดยไม่ได้ตั้งใจใน debug build (ลืมว่า benchmark ต้อง compile ด้วย optimization)**

`cargo bench` โดย default จะ compile ด้วย profile `bench` ที่ inherit ค่าจาก `release` มา (opt-level = 3
ตามที่ Part 35 อธิบายไว้) ดังนั้นปัญหานี้มักไม่เกิดกับ `cargo bench` ตรง ๆ — แต่เกิดได้บ่อยเมื่อคนพยายามเขียน
benchmark เอง**นอก** `criterion` (เช่นใช้ `Instant` ธรรมดาแบบหัวข้อ 54.1) แล้วรันด้วย `cargo run` ที่ไม่มี
`--release` (compile ด้วย profile `dev`, `opt-level = 0`) — ผลลัพธ์ที่ได้จะ**ช้ากว่าความเป็นจริงหลายเท่า**
(ตามที่ Part 35 หัวข้อ 35.2 แสดงตัวเลขจริงไว้ว่า release build เร็วกว่า debug build ได้หลายเท่าสำหรับโค้ดที่มี
loop/คำนวณหนัก) และแย่กว่านั้นคือ **สัดส่วนความต่างระหว่าง implementation ที่เทียบกันอาจผิดเพี้ยนไปด้วย** เพราะ
debug build ไม่ inline อะไรเลย ทำให้ต้นทุนของ function call/abstraction ที่ปกติถูก optimize ออกไปหมดใน release
build กลับปรากฏเด่นชัดใน debug build จนอาจทำให้สรุปผิดว่า "iterator ช้ากว่า loop มือเขียนมาก" (ซึ่งไม่จริงเลย
ใน release build ตามที่หัวข้อ 54.7.3 แสดงให้เห็น) — **กฎทองคือ: benchmark ทุกครั้งต้องรันบน optimized build
เท่านั้น (`cargo bench` ทำให้อัตโนมัติอยู่แล้ว หรือถ้าเขียนเองต้องใช้ `cargo run --release`/`rustc -O` เสมอ)**

**3. วัดผลบนสภาพแวดล้อมที่มี noise สูงจนตัวเลขไม่น่าเชื่อถือ**

แม้ criterion จะจัดการ noise ได้ดีกว่า naive timing มาก แต่มันไม่สามารถ "แก้" สภาพแวดล้อมที่มี noise สูง
ผิดปกติได้ทั้งหมด — สถานการณ์ที่ทำให้ผลลัพธ์เชื่อถือไม่ได้ ได้แก่: รัน benchmark บนเครื่องที่มีโปรแกรมอื่นทำงาน
หนักพร้อมกัน (browser เปิดหลาย tab, video call, ระบบ backup ทำงานเบื้องหลัง), รันบน **virtualized environment
ที่ share CPU กับ VM อื่น** (เช่น cloud CI runner ราคาถูกที่ share core กับผู้ใช้อื่น — ตัวเลขจาก CI อาจแปรผัน
ได้มากกว่าเครื่องส่วนตัวหลายเท่า), หรือรันบน **laptop ที่ CPU throttle เพราะร้อน** (ทำให้ความเร็วลดลงกลางทาง
การวัด) สัญญาณเตือนที่ควรระวัง: ถ้า criterion รายงาน `Found N outliers` ในสัดส่วนสูงมาก (เช่นเกิน 20-30%
ของทุก benchmark ที่รัน) หรือช่วงความเชื่อมั่น (`[min mid max]`) กว้างผิดปกติเมื่อเทียบกับค่ากลาง ให้สงสัยว่า
สภาพแวดล้อมมี noise สูงเกินกว่าจะเชื่อตัวเลขได้ — วิธีแก้: ปิดโปรแกรมพื้นหลังที่ไม่จำเป็น, รันบนเครื่องเปล่า
(bare-metal) ถ้าเป็นไปได้, และรัน benchmark ซ้ำหลายรอบเพื่อดูว่าผลลัพธ์คงที่ข้ามรันหรือไม่ก่อนจะเชื่อ

**4. เสียเวลา optimize/benchmark จุดที่ไม่ใช่ bottleneck จริงของโปรแกรม (Over-benchmarking Micro-things)**

การเขียน benchmark เปรียบเทียบทุกฟังก์ชันเล็ก ๆ ในโปรแกรมแบบไม่มีลำดับความสำคัญ (เช่นเสียเวลาหลายวันไป
optimize/benchmark ฟังก์ชัน parse string ที่ถูกเรียกแค่ 10 ครั้งตอน startup ของโปรแกรม ในขณะที่ database query
หลักที่ถูกเรียกนับพันครั้งต่อวินาทีไม่เคยถูกวัดเลย) คือการทำผิดลำดับความสำคัญตามที่หัวข้อ 54.8 อธิบายไว้ — ผลที่
ได้คือเสียเวลาพัฒนาไปกับสิ่งที่ผู้ใช้จะไม่สัมผัสได้ถึงความต่างเลย (0.9% เทียบกับ 18.8% ตามตัวอย่างในหัวข้อ 54.8)
และในหลายกรณี โค้ดที่ optimize มากเกินจำเป็นยังอ่านยากขึ้นโดยไม่ได้ผลตอบแทนที่วัดได้จริงในภาพรวม — วิธีแก้:
**profile โปรแกรมทั้งตัวก่อน (Part 55) เพื่อรู้ว่าเวลาส่วนใหญ่ไปอยู่ที่ไหนจริง แล้วเอาเทคนิคของบทนี้ไปโฟกัสวัด/
ปรับปรุงเฉพาะจุดนั้น** ไม่ใช่ไล่เขียน benchmark ให้ทุกฟังก์ชันในโปรแกรมแบบไม่มีเป้าหมาย

**5. เข้าใจผิดว่าบรรทัด `change:` ของ criterion เปรียบเทียบระหว่าง sibling variant ในกลุ่มเดียวกัน**

จากหัวข้อ 54.6.2 บรรทัด `change: [...]` ที่ criterion แสดงเป็นการเทียบผลรัน**ปัจจุบัน**กับผลรัน**ก่อนหน้า**ของ
**benchmark เดียวกัน** (regression detection ข้ามเวลา) **ไม่ใช่**การเทียบระหว่าง variant สองตัวในกลุ่มเดียวกัน
(เช่น `push_str_no_capacity` เทียบกับ `push_str_with_capacity`) — ถ้าอยากเทียบว่า variant ไหนเร็วกว่ากันใน
การรันครั้งเดียว ต้องดูตัวเลข `time: [...]` ของแต่ละ variant แล้วเอามาเทียบกันเอง (หรือดูจากกราฟเทียบใน HTML
report ที่ทำให้เห็นง่ายกว่า) การเข้าใจผิดจุดนี้อาจทำให้สรุปผลผิดพลาด เช่นเห็น `No change in performance
detected` แล้วเข้าใจผิดว่า "สอง variant นี้เร็วเท่ากัน" ทั้งที่จริง ๆ มันหมายความว่า "variant นี้ (ตัวเดียว) ไม่
ได้เปลี่ยนไปจากที่รันครั้งก่อน" เท่านั้น

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** สร้างโปรเจกต์ใหม่ เพิ่ม `criterion` เป็น dev-dependency ตามหัวข้อ 54.3.1 แล้วเขียน benchmark
   เปรียบเทียบการหาผลรวมของเลข `1..=1_000_000` ด้วยสองวิธี: `(1..=1_000_000u64).sum()` กับ manual `for` loop
   ที่บวกสะสมเอง รัน `cargo bench` แล้วดูว่าผลลัพธ์ตรงกับที่คาดไว้หรือไม่ (Hint: ผลรวมของเลขต่อเนื่องมี closed
   form formula ทางคณิตศาสตร์ `n(n+1)/2` — สังเกตว่า LLVM มีโอกาสตรวจจับ pattern นี้แล้ว optimize ทั้ง loop
   ให้เหลือแค่การคำนวณครั้งเดียวหรือไม่ ถ้าตัวเลขที่วัดได้เร็วผิดปกติทั้งสองวิธี ให้สงสัยเรื่องนี้ก่อน)

2. **[กลาง]** เขียนฟังก์ชันสองตัวที่ตรวจสอบว่าตัวเลขหนึ่งเป็นจำนวนเฉพาะ (prime) หรือไม่ — ตัวแรกเช็คหารด้วย
   ทุกตัวเลขตั้งแต่ 2 ถึง `n-1`, ตัวที่สองเช็คหารแค่ถึง `sqrt(n)` เท่านั้น (อัลกอริทึมที่มีความซับซ้อนต่ำกว่า
   ตามทฤษฎี) เขียน benchmark เทียบทั้งสองแบบสำหรับการเช็คจำนวนเฉพาะขนาดใหญ่ (เช่นเลขระดับ 10 หลัก) แล้ว
   ตรวจสอบว่าความต่างของความซับซ้อนทางทฤษฎีสะท้อนออกมาเป็นความต่างของเวลาที่วัดได้จริงมากน้อยแค่ไหน (Hint:
   อย่าลืม `black_box` ทั้ง input และ output ตามหัวข้อ 54.5)

3. **[ยาก]** จากตัวอย่าง `Vec` linear search vs `HashMap` lookup ในหัวข้อ 54.7.4 ที่แสดงจุดตัดระหว่าง n=8
   (Vec ชนะ) กับ n=10,000 (HashMap ชนะ) — เขียน benchmark ที่ทดสอบหลายขนาด (เช่น n = 4, 8, 16, 32, 64, 128,
   256) เพื่อหา**จุดตัดที่แม่นยำกว่า**บนเครื่องของคุณเอง (ขนาดไหนคือจุดที่ทั้งสองวิธีให้เวลาใกล้เคียงกันที่สุด)
   แล้วลองเปลี่ยนประเภทของ key จาก `u32` เป็น `String` (ที่การเทียบค่าและ hash แพงกว่า) ดูว่าจุดตัดเปลี่ยนไป
   ทางไหน อธิบายเชิงกลไกว่าทำไม (Hint: การเทียบ `String` ต้องเทียบทุกตัวอักษร ไม่ใช่แค่ 4 ไบต์แบบ `u32` — ทั้ง
   การ hash และการเทียบค่าตอน linear scan จะแพงขึ้นทั้งคู่ แต่แพงขึ้นในสัดส่วนที่ต่างกันหรือไม่?)

4. **[ยาก/ประยุกต์ใช้งานจริง]** สมมติคุณกำลังพัฒนาระบบตรวจสอบ order ซ้ำในระบบขายของออนไลน์ (e-commerce) — มี
   list ของ `order_id: String` ที่ต้องเช็คว่า order ใหม่ที่เข้ามาซ้ำกับ order เก่าหรือไม่ ระบบมี order สะสมอยู่
   แล้วประมาณ 500,000 รายการ และต้องเช็ค order ใหม่เข้ามาประมาณ 100 รายการต่อวินาที เขียน 3 implementation:
   (a) เก็บใน `Vec<String>` เช็คด้วย `.contains()`, (b) เก็บใน `HashSet<String>` เช็คด้วย `.contains()`, (c)
   เก็บใน `HashSet<u64>` โดย hash `order_id` ด้วย `std::hash::Hash` เองก่อนเก็บ (ลด cost การเทียบ string ยาว ๆ
   ทุกครั้ง แลกกับความเสี่ยงเรื่อง hash collision ที่ต้องพิจารณา) วัดทั้งสามแบบด้วย workload ที่จำลองสถานการณ์
   จริง (500,000 รายการเดิม, เช็ค order ใหม่ 100 รายการที่ครึ่งซ้ำครึ่งไม่ซ้ำ) แล้วเขียนสรุปว่าจะเลือก
   implementation ไหนไปใช้จริง พร้อมให้เหตุผลทั้งด้าน performance ที่วัดได้และด้าน correctness/ความเสี่ยงที่
   ต้องแลก (โดยเฉพาะข้อ (c) ที่เพิ่มความเสี่ยงเรื่อง hash collision ที่ (a) และ (b) ไม่มี)

## สรุป

บทนี้สอนหลักคิดที่สำคัญที่สุดข้อเดียวของงาน performance ทั้งหมด: **อย่าเดา วัด** — สัญชาตญาณเรื่องความเร็วของ
โค้ดผิดพลาดได้ง่ายกว่าที่คิด (`format!` ในลูปช้ากว่า `push_str` ถึง 144 เท่า, iterator chain เร็วกว่า manual
loop ได้ถึง 2.4 เท่าในบางกรณี, `with_capacity` ช่วยได้ 5 เท่าสำหรับ element ใหญ่แต่แทบไม่ต่างเลยสำหรับ element
เล็ก) — ตัวเลขเหล่านี้ล้วนวัดได้จริงและขัดหรือเกินความคาดหมายเบื้องต้นในหลายกรณี พิสูจน์ว่าการเดาไม่พอสำหรับงาน
ที่ซีเรียสเรื่อง performance

เราเรียนรู้ว่าการจับเวลาด้วย `Instant` ตรง ๆ มีปัญหาสองข้อ: **noise** จากการรันครั้งเดียว (แก้ด้วยการรันซ้ำและ
วิเคราะห์เชิงสถิติ — สิ่งที่ `criterion` ทำให้อัตโนมัติ) และ **dead code elimination** ที่ compiler อาจลบโค้ด
ที่ผลลัพธ์ไม่ถูกใช้ทิ้งไปทั้งหมด (แก้ด้วย `black_box` ที่ทำหน้าที่เป็นกำแพงทึบกัน compiler มองทะลุ) เราติดตั้ง
และใช้งาน `criterion` อย่างครบวงจร ตั้งแต่ `benches/` (ไดเรกทอรีพิเศษคู่กับ `tests/` จาก Part 33), `[[bench]]
harness = false`, ไปจนถึงการอ่าน HTML report และใช้ baseline ตรวจจับ performance regression (ที่เชื่อมไปสู่
CI/CD ใน Part 97)

ที่สำคัญที่สุด เราได้**วัดจริง**เทคนิค optimization ที่พบบ่อย 5 อย่างด้วย `criterion` จริง แทนที่จะเชื่อคำแนะนำ
ทั่วไปแบบไม่ตรวจสอบ — บางเทคนิคให้ผลตามคาด (หลีกเลี่ยง clone ที่ไม่จำเป็น: เร็วขึ้น 35 เท่า) บางเทคนิคให้ผลที่
ซับซ้อนกว่าที่คิด (`with_capacity`: ช่วยมากหรือน้อยขึ้นกับขนาด element) และบางเทคนิคให้ผลที่ขัดกับสัญชาตญาณโดย
สิ้นเชิง (`#[inline]` ช่วยได้จริงแต่น้อยกว่าที่คาด, iterator chain เร็วกว่า manual loop ในบางกรณี) — ทั้งหมดนี้
คือเหตุผลที่ต้องวัดในสถานการณ์ของตัวเองจริง ไม่ใช่เชื่อกฎทั่วไปแบบตายตัว

สุดท้าย เราปิดท้ายด้วยคำเตือนสำคัญเรื่อง **premature optimization**: การ optimize จุดที่ไม่ใช่ bottleneck จริง
ของโปรแกรมเป็นการเสียเวลาเปล่า — เครื่องมือของบทนี้ (`criterion`) ตอบคำถาม **"เร็วแค่ไหน"** ได้ดีเยี่ยม แต่ยัง
ตอบไม่ได้ว่า **"เวลาไปอยู่ที่ไหนกันแน่ในโปรแกรมทั้งตัว"** — นั่นคือหน้าที่ของเครื่องมือที่เรียกว่า **profiler**
ซึ่งจะเป็นหัวข้อหลักของ **Part 55 (Profiling Rust Applications)** ที่จะสอนวิธีใช้ `perf` และ `flamegraph` เพื่อ
หา bottleneck จริงของโปรแกรมทั้งตัว ก่อนจะย้อนกลับมาใช้เทคนิคการวัดแบบละเอียดของบทนี้ไปโฟกัสที่จุดนั้นโดยเฉพาะ
— สองบทนี้คือเครื่องมือคู่กันที่ตอบคำถามคนละมุมของงาน performance เดียวกัน: **Part 55 บอกว่า "ที่ไหน" ส่วน
Part 54 บอกว่า "เร็วแค่ไหน"**

---

**Part ก่อนหน้า:** [Design Patterns ใน Rust (Newtype, Typestate, RAII)](part-053-design-patterns-2.md) | **Part ถัดไป:** [Profiling Rust Applications](part-055-profiling.md)
