# Part 26: Iterators ขั้นสูง (adapters, custom iterators, performance)

> โมดูล: ระดับกลาง (Intermediate) | ระดับ: กลาง-สูง | เวลาโดยประมาณ: 240 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- ใช้ iterator adaptor ระดับสูงกว่าที่ Part 25 สอนไว้ได้อย่างถูกต้องและรู้ว่าควรเลือกตัวไหนในสถานการณ์ไหน:
  `.flat_map()`, `.flatten()`, `.scan()`, `.step_by()`, `.peekable()`/`.peek()`, `.partition()`, การจัดกลุ่มด้วย
  `.fold()` แบบมือเขียน, และ `.inspect()` สำหรับ debug pipeline โดยไม่แก้ผลลัพธ์
- อธิบายได้ว่า `.fold()` คือ "แม่ของ consuming adaptor ทั้งหมด" — พิสูจน์ด้วยโค้ดจริงว่า `.sum()`, `.count()`,
  `.max()` สามารถเขียนใหม่ด้วย `.fold()` ได้ทั้งหมด (มองปริศนา "70+ methods ได้มาฟรี" จาก Part 25 ในมุมกลับ:
  ไม่ใช่แค่ทุก method เรียก `next()` แต่หลาย method ยังเรียก `.fold()` เป็นแกนกลางร่วมกันด้วย) และใช้
  `.reduce()`, `.try_fold()`, `.try_for_each()`, `.all()`, `.any()`, `.position()`, `.find()`, `.find_map()`
  ได้อย่างถูกต้อง โดยเข้าใจกลไก short-circuit ของ `.try_fold()`/`.try_for_each()` บน `Result`/`Option` ที่ผูกตรง
  กับ `?` operator จาก Part 12
- **เขียน iterator adaptor ของตัวเอง** — struct ที่ห่อ (wrap) iterator ตัวอื่นไว้ข้างใน แล้ว implement
  `Iterator` ให้มันเรียก `.next()` ของตัวที่ห่ออยู่ซ้ำ ๆ พร้อม logic ใหม่ (ตัวอย่าง: `Deduplicate<I>` ที่ข้ามค่า
  ซ้ำติดกัน และ `Batched<I>` ที่รวมกลุ่มเป็นชุดละ N ตัว) — นี่คือทักษะที่ Part 25 (ซึ่งสอนแค่ iterator ที่สร้าง
  ค่าเองจากศูนย์อย่าง `Countdown`) ยังไม่ครอบคลุม
- ใช้ **extension trait pattern** เพิ่ม method ใหม่ (เช่น `.deduplicate()`) ให้กับ **ทุก iterator ที่มีอยู่แล้ว
  ในโลก** ผ่าน trait ที่มี default method + blanket implementation — เข้าใจว่านี่คือกลไกเดียวกันเป๊ะที่ crate
  ชื่อดังอย่าง `itertools` ใช้เพิ่ม method ของตัวเองให้ทุก iterator โดยไม่ต้องแก้ std เลย
- อธิบาย **iterator fusion / zero-cost abstraction** ของ iterator ได้อย่างเจาะลึกกว่า Part 25 — เข้าใจว่า
  compiler รวม adaptor chain ทั้งเส้นให้เป็น loop เดียวได้อย่างไรผ่าน monomorphization + aggressive inlining
  (เชื่อม Part 18/Part 1) และรู้ข้อแลกเปลี่ยนที่แท้จริงของ `Box<dyn Iterator>` (เชื่อม Part 21) เมื่อ
  performance ระดับสุดขีดต้องแลกกับความยืดหยุ่น
- `.collect()` เข้ากับ target ที่ "ฉลาด" กว่า `Vec<T>`/`String`/`HashMap` ธรรมดา — โดยเฉพาะ `Result<Vec<T>, E>`
  และ `Option<Vec<T>>` ที่ short-circuit หยุดทันทีที่เจอ `Err`/`None` ตัวแรก — พร้อมรู้จัก `rayon` ในระดับ
  "รู้ว่ามีและทำอะไรได้" (ไม่เจาะลึก เพราะอยู่นอกขอบเขตของ std) และเขียนโปรแกรมประมวลผล log แบบสมบูรณ์ที่ผสมทุก
  เทคนิคในบทนี้เข้าด้วยกัน

## ความรู้ที่ต้องมีมาก่อน

- **Part 25 (Iterators เบื้องต้น)**: บทนี้คือ**ภาคต่อตรง**ของ Part 25 แบบไม่มีการสอนซ้ำใด ๆ เลย — ต้องแน่นเรื่อง
  นิยามจริงของ `trait Iterator` (`type Item;` + `fn next(&mut self) -> Option<Self::Item>;`), การ desugar ของ
  `for` loop เป็น `while let Some(x) = iter.next()`, ความแตกต่างของ `.iter()`/`.iter_mut()`/`.into_iter()`,
  การเขียน custom iterator แบบพื้นฐาน (`Countdown`), ความแตกต่างของ **consuming adaptor** กับ **iterator
  adaptor**, ความเข้าใจเรื่อง **laziness**, การใช้ `.collect()` เบื้องต้น, adaptor พื้นฐาน (`.filter()`,
  `.map()`, `.enumerate()`, `.zip()`, `.rev()`, `.take()`, `.skip()`, `.chain()`), การเขียน `.map()` เวอร์ชัน
  ของตัวเอง, iterator ที่ไม่มีที่สิ้นสุด, และ marker trait (`DoubleEndedIterator`, `ExactSizeIterator`,
  `FusedIterator`) — ถ้าจุดใดยังไม่แน่น ควรย้อนไปอ่าน Part 25 ก่อน เพราะบทนี้จะอ้างถึงทุกหัวข้อของ Part 25
  ตลอดเวลาโดยไม่อธิบายซ้ำ
- **Part 8 (Slices)** และ **Part 13 (Collections: Vec<T>)**: `.windows()`/`.chunks()` ในหัวข้อ 26.1 เป็น method
  ของ **slice** (`&[T]`) ที่คืน iterator ออกมา — ต้องเข้าใจแนวคิด slice จาก Part 8 และ `Vec<T>` จาก Part 13
  มาก่อน เพื่อเข้าใจว่าทำไม method เหล่านี้อยู่บน `[T]` ไม่ใช่บน `Iterator` โดยตรง
- **Part 12 (Error Handling: Result<T, E>)**: `.try_fold()`, `.try_for_each()`, และการ `.collect()` เป็น
  `Result<Vec<T>, E>` ในหัวข้อ 26.6-26.7 ผูกตรงกับกลไก short-circuit ของ `?` operator ที่เรียนใน Part 12 —
  ถ้ายังไม่แน่นเรื่อง `Result<T, E>`, `?`, และ `From`/`Into` สำหรับแปลง error type ควรทวนก่อน
  (Part 30 จะเจาะลึก custom error type อีกครั้งในอนาคต แต่บทนี้ใช้แค่ `Result` พื้นฐานจาก Part 12 พอ)
- **Part 18 (Generics เบื้องต้น)**: หัวข้อ 26.8 (performance) อ้างกลับไปที่แนวคิด **monomorphization** ที่ Part
  18 สอนไว้ (compiler สร้างโค้ดแยกให้แต่ละ concrete type ตอน compile time) — เป็นกลไกเดียวกันเป๊ะที่ทำให้
  iterator chain ไม่มีต้นทุนเพิ่มจาก generic
- **Part 21 (Traits ขั้นสูง)**: หัวข้อ 26.8 อ้างกลับไปที่ `dyn Trait`/trait object และ dynamic dispatch ที่ Part
  21 สอนไว้ เพื่ออธิบาย `Box<dyn Iterator<Item = T>>` และข้อแลกเปลี่ยนด้าน performance ของมัน
- **Part 22 (Generics ขั้นสูง)**: หัวข้อ 26.3-26.4 (เขียน custom adaptor + extension trait) ใช้ **blanket
  implementation** (`impl<T: Trait1> Trait2 for T`) ที่ Part 22 หัวข้อ 22.9 สอนไว้โดยตรง เป็นกลไกหลักที่ทำให้
  extension trait pattern ใช้ได้กับทุก iterator พร้อมกันในบรรทัดเดียว
- **Part 24 (Closures)**: `.scan()`, `.fold()`, `.reduce()` ทุกตัวรับ closure เป็น parameter และมี trait bound
  เป็น `FnMut`/`FnOnce` ตามหลักการที่ Part 24 สอนไว้แล้ว

## เนื้อหา

### 26.1 ทัวร์ adaptor ที่กว้างขึ้น: เหนือกว่าเซ็ตพื้นฐานของ Part 25

Part 25 แนะนำ adaptor พื้นฐานที่สุด 8 ตัว (`.filter()`, `.map()`, `.enumerate()`, `.zip()`, `.rev()`,
`.take()`, `.skip()`, `.chain()`) ซึ่งครอบคลุมงานส่วนใหญ่ในชีวิตประจำวันได้แล้ว แต่โค้ด production จริงมักเจอ
สถานการณ์ที่ต้องใช้ adaptor ที่ซับซ้อนกว่านั้น มาดูทีละตัว

#### `.flat_map()` และ `.flatten()`: "แบน" ข้อมูลที่ซ้อนกันเป็นชั้น

ลองนึกภาพว่าคุณมี `Vec<Vec<i32>>` (list ของ list) แล้วต้องการวนทุกตัวเลขในทุก list รวมกันเป็น iterator เดียว
แบบ "แบน" (flat):

```rust
fn main() {
    let nested = vec![vec![1, 2, 3], vec![4, 5], vec![6, 7, 8, 9]];

    // .flatten() รับ iterator ของ iterator (หรือของอะไรก็ตามที่ implement IntoIterator)
    // แล้วคืน iterator เดียวที่วนสมาชิกชั้นในทั้งหมดต่อกันเป็นเส้นเดียว
    let flat: Vec<i32> = nested.into_iter().flatten().collect();
    println!("{:?}", flat);
}
```

ผลลัพธ์:

```
[1, 2, 3, 4, 5, 6, 7, 8, 9]
```

`.flatten()` ทำงานได้เพราะ `Item` ของ `nested.into_iter()` คือ `Vec<i32>` ซึ่ง**implement `IntoIterator` เอง**
(ตามที่ Part 25 หัวข้อ 25.7 สอนไว้ว่า `Vec<T>` มี `IntoIterator` ให้ทั้งแบบยึด ownership) — `.flatten()` เพียง
เรียก `.into_iter()` กับสมาชิกแต่ละตัวชั้นนอก แล้วต่อผลลัพธ์ทุกตัวเข้าด้วยกันแบบ lazy (ไม่สร้าง `Vec` กลางเลย
จนกว่าจะถูก `.collect()`)

`.flat_map()` คือ `.map()` ตามด้วย `.flatten()` รวมเป็น method เดียว — มีประโยชน์มากเมื่อ closure ของ `.map()`
คืนค่าเป็น collection แทนค่าเดี่ยว:

```rust
fn main() {
    let sentences = vec!["hello world", "rust is fun"];

    // ถ้าใช้ .map() เฉย ๆ จะได้ Vec<Vec<&str>> (ซ้อนกัน) — ต้อง .flatten() ต่ออีกที
    // .flat_map() ทำสองขั้นตอนนี้ในการเรียกครั้งเดียว
    let words: Vec<&str> = sentences
        .iter()
        .flat_map(|sentence| sentence.split(' '))
        .collect();

    println!("{:?}", words);
}
```

ผลลัพธ์:

```
["hello", "rust", "is", "fun"]
```

**ทำไมสำคัญ**: `sentence.split(' ')` คืน iterator ของ `&str` (ไม่ใช่ `Vec<&str>` — คุ้นเคยจาก Part 14) ถ้าใช้
`.map(|sentence| sentence.split(' '))` เฉย ๆ จะได้ `Item` เป็น `std::str::Split<'_, char>` (ตัว iterator เอง
ไม่ใช่ผลลัพธ์ที่แบนแล้ว) ต้องเรียก `.flatten()` เพิ่มอีกขั้นตอนเสมอ `.flat_map()` รวมสองขั้นตอนที่มักเกิดคู่กัน
นี้ให้เป็น method เดียว อ่านง่ายกว่าและสื่อเจตนาตรงกว่า "map แล้วก็ flatten ทันที"

#### `.scan()`: เหมือน `.map()` แต่มี "ความจำ" ระหว่างแต่ละ item

`.map()` ที่เรียนมาจาก Part 25 แปลงค่าแต่ละตัว**โดยไม่รู้จักตัวก่อนหน้าเลย** — closure ของ `.map()` เห็นแค่ item
ปัจจุบัน ไม่มีทางเข้าถึง state ที่สะสมมาจาก item ก่อนได้ `.scan()` แก้ข้อจำกัดนี้โดยให้ closure ถือ **state ที่
เก็บสะสมได้** (คล้าย accumulator ของ `.fold()` ที่จะเห็นในหัวข้อ 26.2 แต่ `.scan()` ยังเป็น**iterator adaptor**
ที่ lazy อยู่ ไม่ใช่ consuming adaptor):

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    // scan(initial_state, |state, item| -> Option<T>)
    // - state: &mut ตัวสถานะที่เราเลือกชนิดเอง (ในที่นี้คือ running sum ชนิด i32)
    // - คืน Some(x) แปลว่า "ให้ x ออกไปเป็น item ถัดไปของ iterator ผลลัพธ์"
    // - คืน None เมื่อไหร่ ก็หยุด iterator ทั้งเส้นทันที (คล้าย .next() คืน None)
    let running_sums: Vec<i32> = numbers
        .iter()
        .scan(0, |running_total, &n| {
            *running_total += n;
            Some(*running_total)
        })
        .collect();

    println!("{:?}", running_sums);
}
```

ผลลัพธ์:

```
[1, 3, 6, 10, 15]
```

**อ่านทีละส่วน**: `.scan(0, |running_total, &n| { ... })` — argument แรก (`0`) คือ **state เริ่มต้น** closure
รับ `&mut state` (ในตัวอย่างนี้ชื่อ `running_total`) กับ item ปัจจุบัน (`&n` เพราะ `.iter()` ให้ `&i32`) แล้วต้อง
คืน `Option<T>` เสมอ — `*running_total += n` แก้ state สะสมไปเรื่อย ๆ ระหว่าง item, `Some(*running_total)` บอก
ว่า "item ถัดไปของผลลัพธ์คือ running total ตอนนี้" ผลคือได้ **running sum** (ผลรวมสะสม) ของ `numbers` แต่ละจุด

จุดที่ต่างจาก `.fold()` ชัดเจนที่สุด: `.scan()` **ยังเป็น iterator** — chain ต่อด้วย `.filter()`, `.take()`
อะไรก็ได้อีก และยัง lazy อยู่ (ไม่ทำงานจนกว่าจะมี consuming adaptor) ส่วน `.fold()` ขับเคลื่อน iterator จนสุด
ทันทีแล้วคืนค่าสุดท้ายค่าเดียว ไม่ใช่ iterator อีกต่อไป

#### `.step_by()`: ข้าม item ทีละ N

```rust
fn main() {
    let every_third: Vec<i32> = (1..=20).step_by(3).collect();
    println!("{:?}", every_third);
}
```

ผลลัพธ์:

```
[1, 4, 7, 10, 13, 16, 19]
```

`.step_by(n)` เก็บ item แรก แล้ว**ข้าม** `n - 1` ตัวก่อนจะเก็บตัวถัดไปเสมอ (ในที่นี้ `n = 3` แปลว่าเก็บ 1 ตัว
ข้าม 2 ตัว ซ้ำไปเรื่อย ๆ) ใช้บ่อยเมื่อต้องการ "สุ่มตัวอย่างแบบสม่ำเสมอ" จากข้อมูลจำนวนมาก โดยไม่ต้องเขียน
`.enumerate().filter(|(i, _)| i % n == 0).map(|(_, x)| x)` ที่ยาวกว่าและอ่านยากกว่า

#### `.peekable()` / `.peek()`: มองล่วงหน้าโดยไม่กิน item

หนึ่งในข้อจำกัดของ `Iterator` พื้นฐาน (จาก Part 25) คือเรียก `.next()` แล้ว**ไม่มีทางถอยกลับ** — ถ้าอยาก "ดูก่อน
ว่า item ถัดไปคืออะไร" แล้วค่อยตัดสินใจว่าจะกินมันหรือไม่ ต้องใช้ `.peekable()` ห่อ iterator ไว้ก่อน:

```rust
fn main() {
    let tokens = vec!["1", "+", "2", "*", "3"];
    let mut iter = tokens.iter().peekable();

    // .peek() คืน Option<&Item> — ดูค่าถัดไปโดยไม่เดินหน้า iterator เลย
    println!("peek ครั้งแรก: {:?}", iter.peek());
    println!("peek อีกครั้ง (ค่าเดิม เพราะยังไม่ .next()): {:?}", iter.peek());

    // เรียก .next() จริง ๆ ค่อยเดินหน้า
    println!("next(): {:?}", iter.next());
    println!("peek หลัง next(): {:?}", iter.peek());
}
```

ผลลัพธ์:

```
peek ครั้งแรก: Some("1")
peek อีกครั้ง (ค่าเดิม เพราะยังไม่ .next()): Some("1")
next(): Some("1")
peek หลัง next(): Some("+")
```

**กรณีใช้งานจริงที่พบบ่อยที่สุด**: กลุ่ม item ที่มีเงื่อนไข "ต่อเนื่องกันเมื่อไหร่ให้รวมกัน" เช่น parser ง่าย ๆ
ที่ต้องอ่านตัวเลขหลายหลักติดกันให้กลายเป็นเลขเดียว — ต้อง `.peek()` ดูก่อนว่าตัวถัดไปยังเป็นตัวเลขอยู่ไหมก่อน
ตัดสินใจว่าจะ `.next()` กินมันเข้ามารวมด้วยหรือหยุด:

```rust
fn main() {
    let input = "12ab34";
    let mut chars = input.chars().peekable();
    let mut numbers: Vec<String> = Vec::new();

    while let Some(&c) = chars.peek() {
        if c.is_ascii_digit() {
            let mut current = String::new();
            // วนกินตัวเลขติดกันไปเรื่อย ๆ จนกว่า peek() จะไม่ใช่ตัวเลขแล้ว
            while let Some(&d) = chars.peek() {
                if d.is_ascii_digit() {
                    current.push(d);
                    chars.next(); // กินจริง ๆ ตอนนี้
                } else {
                    break;
                }
            }
            numbers.push(current);
        } else {
            chars.next(); // ข้ามตัวอักษรที่ไม่ใช่เลข
        }
    }

    println!("{:?}", numbers);
}
```

ผลลัพธ์:

```
["12", "34"]
```

สังเกตว่าถ้าไม่มี `.peekable()` เราจะไม่มีทางรู้ว่า "ตัวถัดไปยังเป็นเลขอยู่ไหม" ก่อนตัดสินใจ — จะต้องกิน
`.next()` ไปก่อนแล้วค่อยเช็ค ซึ่งถ้าไม่ใช่เลขแล้วจะทำให้ตัวนั้น "หายไป" จาก stream โดยไม่ได้ตั้งใจ `.peekable()`
แก้ปัญหานี้ได้อย่างสมบูรณ์ด้วยการห่อ iterator เดิมไว้แล้วเก็บ "item ที่ดูไปแล้วแต่ยังไม่กิน" ไว้ใน field ภายใน
ตัวมันเอง (`Peekable<I>` เก็บ `Option<I::Item>` ไว้ 1 ช่อง — คล้ายกับที่เราจะเขียน custom adaptor เองในหัวข้อ
26.3)

#### `.windows()` และ `.chunks()`: method ของ slice ไม่ใช่ของ `Iterator` — เชื่อม Part 8/13

`.windows(n)` และ `.chunks(n)` **ไม่ได้อยู่ใน trait `Iterator`** — มันเป็น method ของ **slice** (`[T]`) ที่
เรียนมาแล้วใน Part 8 (Slices) และ Part 13 (`Vec<T>` deref เป็น `&[T]` ได้เสมอ) ทั้งสองคืน **iterator ของ slice
ย่อย ๆ** ทำให้ยังใช้ adaptor ต่อได้ตามปกติ แต่ต่างกันตรงที่ `.windows()` ให้ slice ย่อยที่**เหลื่อมกัน**
(overlapping) ส่วน `.chunks()` ให้ slice ย่อยที่**ไม่เหลื่อมกัน** (non-overlapping):

```rust
fn main() {
    let data = [1, 2, 3, 4, 5];

    println!("=== .windows(3): slice ย่อยขนาด 3 ที่เหลื่อมกัน ===");
    for w in data.windows(3) {
        println!("{:?}", w);
    }

    println!("=== .chunks(2): slice ย่อยขนาด 2 ที่ไม่เหลื่อมกัน ===");
    for c in data.chunks(2) {
        println!("{:?}", c);
    }
}
```

ผลลัพธ์:

```
=== .windows(3): slice ย่อยขนาด 3 ที่เหลื่อมกัน ===
[1, 2, 3]
[2, 3, 4]
[3, 4, 5]
=== .chunks(2): slice ย่อยขนาด 2 ที่ไม่เหลื่อมกัน ===
[1, 2]
[3, 4]
[5]
```

**เชื่อมกับ Part 8**: `.windows(3)` ให้ 3 slice (เพราะข้อมูล 5 ตัว หน้าต่างขนาด 3 เลื่อนได้ 5 - 3 + 1 = 3 ครั้ง)
แต่ละ slice **เหลื่อมกัน** — `[1,2,3]` กับ `[2,3,4]` มี `2` และ `3` ร่วมกัน ใช้บ่อยเมื่อต้องเทียบ item ที่อยู่
ติดกัน (เช่น หาว่าราคาหุ้นวันไหนขึ้นติดกัน 3 วัน) `.chunks(2)` ให้ slice ขนาด 2 ที่**ไม่เหลื่อมกันเลย** (ตัว
สุดท้ายอาจเล็กกว่าที่ขอถ้าจำนวนสมาชิกหารไม่ลงตัว — ในตัวอย่างนี้ตัวสุดท้ายมีแค่ `[5]`) ใช้บ่อยเมื่อต้องแบ่งงาน
เป็นชุด ๆ ประมวลผลทีละชุด

ทั้งสอง method **ต้องเรียกบน slice ที่มีอยู่แล้วในความจำต่อเนื่องกัน** (`&[T]`) เท่านั้น — เป็นเหตุผลว่าทำไมมัน
ไม่ใช่ method ของ `Iterator` ทั่วไป (`Iterator` ไม่รู้ว่า item ของตัวเองเก็บต่อเนื่องในความจำหรือไม่ — อาจมาจาก
`HashMap` ที่กระจายอยู่คนละที่ก็ได้) ในหัวข้อ 26.3 เราจะเห็นว่าถ้าอยากได้ผลแบบ `.chunks()` แต่ทำงานกับ
**iterator อะไรก็ได้** (ไม่จำกัดแค่ slice) ต้องเขียน custom adaptor เอง (`Batched<I>`) เพราะ std ไม่มี method
สำเร็จรูปให้แบบนั้น

#### `.partition()`: แยกเป็นสองกลุ่มในการวนครั้งเดียว

`.filter()` จาก Part 25 กรองแล้วเหลือแค่ตัวที่ผ่านเงื่อนไข — ถ้าต้องการ**ทั้งสองกลุ่ม** (ทั้งที่ผ่านและไม่ผ่าน)
การเรียก `.filter()` สองครั้ง (ครั้งละเงื่อนไขตรงข้ามกัน) จะวนข้อมูลซ้ำสองรอบ `.partition()` ทำทั้งสองอย่างในการ
วนครั้งเดียว:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

    // .partition() เป็น consuming adaptor — วนครั้งเดียว แยกเป็นสอง collection ตามเงื่อนไข
    let (evens, odds): (Vec<i32>, Vec<i32>) =
        numbers.into_iter().partition(|n| n % 2 == 0);

    println!("evens = {:?}", evens);
    println!("odds  = {:?}", odds);
}
```

ผลลัพธ์:

```
evens = [2, 4, 6, 8, 10]
odds  = [1, 3, 5, 7, 9]
```

สังเกตว่า type annotation `(Vec<i32>, Vec<i32>)` **จำเป็น** เหมือนกับ `.collect()` (Part 25 หัวข้อ 25.12) —
`.partition()` เป็น generic method ที่ต้องรู้ target type ทั้งสองฝั่งของ tuple ก่อนจะเลือก
`FromIterator` implementation ที่ถูกต้อง (concept เดียวกับ error "type annotations needed" ที่เรียนไปแล้ว)

#### จัดกลุ่ม (grouping) แบบมือเขียนด้วย `.fold()`: เพราะ `group_by` ไม่มีใน std

หลายภาษา (Python's `itertools.groupby`, หรือ SQL's `GROUP BY`) มี method จัดกลุ่มสำเร็จรูปให้ — **Rust's std
library ไม่มี `.group_by()` ให้ใน `Iterator` เลย** (แม้ crate `itertools` จะมีให้ แต่บทนี้จำกัดอยู่ใน std
เท่านั้นตามที่กำหนดไว้) วิธีจัดกลุ่มมาตรฐานใน Rust คือใช้ `.fold()` (ซึ่งหัวข้อ 26.2 จะอธิบายเจาะลึกว่าทำไม
`.fold()` คือ "แกนกลาง" ของ consuming adaptor ทั้งหมด) สะสมผลลงใน `HashMap<K, Vec<V>>`:

```rust
use std::collections::HashMap;

#[derive(Debug, Clone)]
struct Order {
    customer: String,
    amount: f64,
}

fn main() {
    let orders = vec![
        Order { customer: "Alice".to_string(), amount: 150.0 },
        Order { customer: "Bob".to_string(), amount: 80.0 },
        Order { customer: "Alice".to_string(), amount: 220.0 },
        Order { customer: "Carol".to_string(), amount: 300.0 },
        Order { customer: "Bob".to_string(), amount: 45.0 },
    ];

    // จัดกลุ่ม order ตามชื่อลูกค้า: HashMap<String, Vec<Order>>
    let grouped: HashMap<String, Vec<Order>> = orders.into_iter().fold(
        HashMap::new(),
        |mut acc, order| {
            // .entry().or_insert_with() มาจาก Part 15 (HashMap)
            acc.entry(order.customer.clone())
                .or_insert_with(Vec::new)
                .push(order);
            acc
        },
    );

    // .collect() ที่ HashMap ไม่รักษาลำดับ (Part 15) เราจึงเรียง key ก่อน print เพื่อผลลัพธ์คงที่
    let mut names: Vec<&String> = grouped.keys().collect();
    names.sort();
    for name in names {
        let total: f64 = grouped[name].iter().map(|o| o.amount).sum();
        println!("{name}: {} order(s), รวม {total:.2}", grouped[name].len());
    }
}
```

ผลลัพธ์:

```
Alice: 2 order(s), รวม 370.00
Bob: 2 order(s), รวม 125.00
Carol: 1 order(s), รวม 300.00
```

**อ่านทีละส่วน**: `.fold(HashMap::new(), |mut acc, order| { ... acc })` — เริ่มด้วย `HashMap` เปล่า แต่ละรอบรับ
`acc` (accumulator สะสม) กับ `order` ปัจจุบัน แล้ว **ต้องคืน `acc` กลับออกไปเสมอ** (นี่คือกฎของ `.fold()` ที่จะ
อธิบายเต็มรูปแบบในหัวข้อ 26.2) เราหา (หรือสร้างใหม่ถ้ายังไม่มี) `Vec<Order>` ของลูกค้าคนนั้นด้วย
`.entry().or_insert_with(Vec::new)` แล้ว `.push(order)` เข้าไป — ผลคือ `HashMap` ที่ key คือชื่อลูกค้า, value
คือรายการ order ทั้งหมดของลูกค้านั้น นี่คือ pattern มาตรฐานที่ใช้แทน `group_by` ในโค้ด Rust จริงทุกที่ที่ไม่ได้
ใช้ crate เสริม

#### `.inspect()`: มองเข้าไปใน pipeline โดยไม่แก้ผลลัพธ์เลย

เวลา debug adaptor chain ยาว ๆ บางครั้งอยากรู้ว่า "ค่าตรงจุดนี้ของ chain เป็นอะไร" โดยไม่อยากแก้โค้ดจริงเพื่อ
`println!` ชั่วคราว (แล้วต้องลบออกทีหลัง) `.inspect()` คือ adaptor ที่ทำแบบนั้นโดยเฉพาะ — รับ closure ที่รับ
`&Item` (ยืมอ่านอย่างเดียว) แล้ว**ไม่เปลี่ยนแปลงอะไรเลย** ค่าที่ไหลผ่าน `.inspect()` ออกมาเหมือนก่อนเข้าทุก
ประการ:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    let sum: i32 = numbers
        .iter()
        .inspect(|n| println!("ก่อน filter เห็น: {n}"))
        .filter(|&&n| n % 2 == 0)
        .inspect(|n| println!("หลัง filter เหลือ: {n}"))
        .sum();

    println!("ผลรวม = {sum}");
}
```

ผลลัพธ์:

```
ก่อน filter เห็น: 1
ก่อน filter เห็น: 2
หลัง filter เหลือ: 2
ก่อน filter เห็น: 3
ก่อน filter เห็น: 4
หลัง filter เหลือ: 4
ก่อน filter เห็น: 5
ผลรวม = 6
```

สังเกตลำดับการพิมพ์ — มันสลับกันไปมาระหว่าง `.inspect()` ตัวแรกกับตัวที่สอง **ไม่ใช่**พิมพ์ `.inspect()` แรก
ครบทุกตัวก่อนแล้วค่อยพิมพ์ตัวที่สอง นี่คือหลักฐานตรงจุดของ **laziness** ที่ Part 25 พิสูจน์ไว้แล้ว: แต่ละ item
ไหลผ่าน**ทั้ง chain ทีละตัว** ("pull-based", ไม่ใช่ "batch-based") ก่อนที่ item ถัดไปจะเริ่มไหล — เราจะย้ำเรื่อง
นี้อีกครั้งในหัวข้อ 26.8 ตอนอธิบาย iterator fusion ว่าทำไม `.inspect()` (และ adaptor อื่นทั้งหมด) ไม่ทำให้เกิด
loop แยกหลาย loop ในโค้ดที่ compile ออกมาจริง

### 26.2 `.fold()`: แกนกลางของ consuming adaptor ทั้งหมด

Part 25 หัวข้อ 25.9 พิสูจน์ว่า default method กว่า 70 ตัวของ `Iterator` ได้มาฟรีเพราะทุกตัวสุดท้ายเรียก
`self.next()` เป็นแกนกลาง — หัวข้อนี้จะมองปริศนาเดียวกันจาก**อีกมุมหนึ่ง**: หลาย consuming adaptor ที่ดูเหมือน
ทำงานคนละแบบกัน (`.sum()`, `.count()`, `.max()`) จริง ๆ แล้ว**เขียนด้วยรูปแบบเดียวกัน** คือ "สะสมค่าจาก item
ก่อนหน้าเข้ากับ item ปัจจุบัน ทีละตัว จนกว่า iterator จะหมด" — รูปแบบนี้มีชื่อและมี method ให้เรียกตรง ๆ คือ
`.fold()`

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    // fold(initial, |accumulator, item| -> new_accumulator)
    let sum = numbers.iter().fold(0, |acc, &n| acc + n);
    println!("sum ผ่าน fold = {sum}");

    let product = numbers.iter().fold(1, |acc, &n| acc * n);
    println!("product ผ่าน fold = {product}");

    let count = numbers.iter().fold(0, |acc, _| acc + 1);
    println!("count ผ่าน fold = {count}");

    let max = numbers.iter().fold(i32::MIN, |acc, &n| if n > acc { n } else { acc });
    println!("max ผ่าน fold = {max}");

    // เทียบกับตัวจริงจาก std ว่าให้ผลตรงกันทุกประการ
    println!(
        "เทียบกับ .sum()/.product()/.count()/.max() จริง: {} {} {} {:?}",
        numbers.iter().sum::<i32>(),
        numbers.iter().product::<i32>(),
        numbers.iter().count(),
        numbers.iter().max()
    );
}
```

ผลลัพธ์:

```
sum ผ่าน fold = 15
product ผ่าน fold = 120
count ผ่าน fold = 5
max ผ่าน fold = 5
เทียบกับ .sum()/.product()/.count()/.max() จริง: 15 120 5 Some(5)
```

**อ่านนิยามของ `.fold()` ให้เข้าใจกลไก**:

```
fn fold<B, F>(self, init: B, mut f: F) -> B
where
    F: FnMut(B, Self::Item) -> B,
```

- `init: B` — ค่าเริ่มต้นของ accumulator (ชนิด `B` ซึ่งอาจเป็นชนิดอะไรก็ได้ ไม่ต้องเหมือน `Item`)
- `f: FnMut(B, Self::Item) -> B` — closure ที่รับ **accumulator ปัจจุบัน + item ปัจจุบัน** แล้วคืน
  **accumulator ใหม่**
- ทำงานคร่าว ๆ เหมือน (pseudocode อธิบายหลักการ):

```
fn fold(mut self, init: B, mut f: F) -> B {
    let mut accumulator = init;
    while let Some(item) = self.next() {
        accumulator = f(accumulator, item);
    }
    accumulator
}
```

สังเกตว่านี่คือ**รูปแบบเดียวกันเป๊ะ**กับ pseudocode ของ `.count()` ที่ Part 25 หัวข้อ 25.9 แสดงไว้ (`while let
Some(_) = self.next() { n += 1; }`) เพียงแค่ `.fold()` **ให้เราเลือก accumulator type เองได้** (`B` ใดก็ได้ ไม่
บังคับเป็น `usize` แบบ `.count()`) และ**ให้เราเลือก logic การรวมเองได้** (ผ่าน closure `f`) — เพราะฉะนั้น
`.sum()`, `.product()`, `.count()`, `.max()`, `.min()` ทุกตัวสามารถมองเป็น **`.fold()` เวอร์ชันที่ล็อค
accumulator type และ logic ไว้ตายตัวแล้ว** เท่านั้นเอง (`.sum()` ล็อค `B = Item`, `f = |acc, x| acc + x`;
`.count()` ล็อค `B = usize`, `f = |acc, _| acc + 1`)

นี่คือเหตุผลที่บอกว่า `.fold()` เป็น **"แกนกลางของ consuming adaptor"** — ในทางปฏิบัติ `.fold()` มีพลังมากพอที่
จะเขียน consuming adaptor เกือบทุกตัวขึ้นมาใหม่เองได้ (แม้ std เองจะไม่ได้ implement ทุกตัวด้วย `.fold()`
เพราะบางตัวปรับให้ optimize เฉพาะเจาะจงกว่า แต่ในทาง**ความหมาย** (semantics) ทุกตัวเทียบเท่ากับ `.fold()`
ที่เลือก accumulator และ logic ให้เหมาะกับงานนั้น ๆ)

#### `.reduce()`: เหมือน `.fold()` แต่ไม่ต้องมีค่าเริ่มต้น — ใช้ item แรกเป็นค่าเริ่มต้นให้เอง

ปัญหาหนึ่งของ `.fold()` คือต้องระบุ `init` เสมอ — แต่บางสถานการณ์ "ค่าเริ่มต้นที่เป็นกลาง" ไม่มีความหมายชัดเจน
(เช่น หา max ของ `Vec<String>` — จะใช้ `String` อะไรเป็นค่าเริ่มต้นที่ "เป็นกลาง"?) `.reduce()` แก้ปัญหานี้ด้วย
การใช้ **item แรกของ iterator เป็นค่าเริ่มต้นให้เอง** จึงคืน `Option<Item>` เสมอ (เผื่อกรณี iterator เปล่า — ไม่
มี item แรกให้ใช้เป็นค่าเริ่มต้น):

```rust
fn main() {
    let words = vec!["short", "medium", "extremely long word"];

    // reduce(|acc, item| -> Item) — closure คืนชนิดเดียวกับ Item เสมอ (ต่างจาก fold ที่ B เลือกได้อิสระ)
    let longest = words.iter().reduce(|longest, current| {
        if current.len() > longest.len() {
            current
        } else {
            longest
        }
    });

    println!("{:?}", longest);

    let empty: Vec<&str> = vec![];
    let result = empty.iter().reduce(|a, _| a);
    println!("reduce บน iterator เปล่า: {:?}", result);
}
```

ผลลัพธ์:

```
Some("extremely long word")
reduce บน iterator เปล่า: None
```

**ความแตกต่างที่สำคัญระหว่าง `.fold()` กับ `.reduce()`**:

| ประเด็น | `.fold()` | `.reduce()` |
|---|---|---|
| ต้องระบุค่าเริ่มต้น | ต้องระบุเสมอ (`init: B`) | ไม่ต้อง — ใช้ item แรกให้เอง |
| accumulator type | อิสระ (`B` เป็นชนิดใดก็ได้ ไม่ต้องตรงกับ `Item`) | ต้องเป็นชนิดเดียวกับ `Item` เท่านั้น |
| iterator เปล่า | คืน `init` เดิมกลับไปตรง ๆ (ไม่ error) | คืน `None` (เพราะไม่มี item แรกให้ใช้เป็นค่าเริ่มต้น) |
| ใช้เมื่อไหร่ | ต้องการ accumulator type ต่างจาก `Item` (เช่น สะสมเป็น `String`, `HashMap`, `Vec`) | ต้องการ "รวมค่าชนิดเดียวกันเข้าด้วยกันเรื่อย ๆ" (max, min, concat) และรับได้ว่าอาจไม่มีผลลัพธ์ |

#### `.try_fold()` และ `.try_for_each()`: short-circuit บน `Result`/`Option` — เชื่อม Part 12

`.fold()` ธรรมดามีข้อจำกัดสำคัญ: **มันวนจนสุด iterator เสมอ ไม่มีทาง "หยุดกลางทาง" ได้เลย** แม้ item ตัวที่ 3
จะทำให้เกิด error ที่ทำให้ผลลัพธ์ทั้งหมดไม่มีความหมายแล้วก็ตาม `.fold()` ก็ยังต้องประมวลผล item ที่ 4, 5, ... ที่
เหลือทั้งหมดต่อไปอยู่ดี (สิ้นเปลืองโดยไม่จำเป็น) `.try_fold()` แก้ปัญหานี้: closure ต้องคืน `Result<B, E>` (หรือ
`Option<B>`) และ**หยุดทันทีที่เจอ `Err` (หรือ `None`) ตัวแรก** — เชื่อมตรงกับกลไก short-circuit ของ `?`
operator ที่เรียนใน Part 12:

```rust
fn main() {
    let inputs = vec!["10", "20", "abc", "30"];

    // try_fold: closure คืน Result<B, E> — ถ้าเจอ Err ตัวไหน หยุดทันที ไม่ประมวลผลตัวที่เหลือเลย
    let result: Result<i32, std::num::ParseIntError> =
        inputs.iter().try_fold(0, |acc, s| {
            let n: i32 = s.parse()?; // ? เชื่อม Part 12: ถ้า parse ผิดพลาด คืน Err ออกจาก closure ทันที
            Ok(acc + n)
        });

    println!("{:?}", result);

    let all_valid = vec!["10", "20", "30"];
    let result2: Result<i32, std::num::ParseIntError> =
        all_valid.iter().try_fold(0, |acc, s| Ok(acc + s.parse::<i32>()?));
    println!("{:?}", result2);
}
```

ผลลัพธ์:

```
Err(ParseIntError { kind: InvalidDigit })
Ok(60)
```

**สิ่งที่เกิดขึ้นข้างใน**: พอ `.try_fold()` ประมวลผลถึง `"abc"` (ตัวที่ 3) แล้ว `s.parse()` คืน `Err`, `?` ทำให้
closure คืน `Err` ออกไปทันที **`.try_fold()` หยุดวนต่อทันที ไม่แตะ `"30"` เลย** — พิสูจน์ได้ด้วย `.inspect()`
(จากหัวข้อ 26.1) แปะไว้ก่อน `.try_fold()`:

```rust
fn main() {
    let inputs = vec!["10", "20", "abc", "30"];

    let result: Result<i32, std::num::ParseIntError> = inputs
        .iter()
        .inspect(|s| println!("try_fold กำลังดู: {s}"))
        .try_fold(0, |acc, s| Ok(acc + s.parse::<i32>()?));

    println!("ผลลัพธ์: {:?}", result);
}
```

ผลลัพธ์:

```
try_fold กำลังดู: 10
try_fold กำลังดู: 20
try_fold กำลังดู: abc
ผลลัพธ์: Err(ParseIntError { kind: InvalidDigit })
```

สังเกตว่า **ไม่มี** `"try_fold กำลังดู: 30"` พิมพ์ออกมาเลย — นี่คือ short-circuit ตัวจริง ต่างจาก `.fold()`
ธรรมดาที่จะวนจนครบทุกตัวเสมอไม่ว่าอะไรจะเกิดขึ้นระหว่างทาง

`.try_for_each()` คือรูปแบบง่ายกว่าของ `.try_fold()` สำหรับกรณีที่ไม่ต้องการสะสมค่าอะไรเลย แค่ต้องการ "ทำ side
effect กับทุก item แล้วหยุดทันทีถ้าตัวไหน error":

```rust
fn main() {
    let urls = vec!["ok1", "ok2", "bad", "ok3"];

    fn validate(url: &str) -> Result<(), String> {
        if url == "bad" {
            Err(format!("URL ไม่ถูกต้อง: {url}"))
        } else {
            println!("ตรวจสอบผ่าน: {url}");
            Ok(())
        }
    }

    let result: Result<(), String> = urls.iter().try_for_each(|url| validate(url));
    println!("ผลรวม: {:?}", result);
}
```

ผลลัพธ์:

```
ตรวจสอบผ่าน: ok1
ตรวจสอบผ่าน: ok2
ผลรวม: Err("URL ไม่ถูกต้อง: bad")
```

`"ok3"` ไม่ถูกแตะเลยเช่นกัน — เพราะ `"bad"` ทำให้ `validate()` คืน `Err` ก่อน `.try_for_each()` จะหยุดทันที

#### `.all()` / `.any()`: ตรวจเงื่อนไขทั้งเส้น พร้อม short-circuit ในตัว

`.all()` คืน `true` ถ้า**ทุก** item ผ่านเงื่อนไข, `.any()` คืน `true` ถ้า**มีอย่างน้อยหนึ่ง** item ผ่านเงื่อนไข —
ทั้งสองมี short-circuit ในตัวเองอยู่แล้วโดยธรรมชาติ (ไม่ต้องใช้ `.try_*()`):

```rust
fn main() {
    let numbers = vec![2, 4, 6, 8, 10];

    // .all() หยุดทันทีที่เจอตัวแรกที่ไม่ผ่าน (คืน false ทันที ไม่ต้องเช็คต่อ)
    let all_even = numbers.iter().all(|&n| n % 2 == 0);
    println!("ทุกตัวเป็นเลขคู่: {all_even}");

    // .any() หยุดทันทีที่เจอตัวแรกที่ผ่าน (คืน true ทันที ไม่ต้องเช็คต่อ)
    let has_negative = numbers.iter().any(|&n| n < 0);
    println!("มีเลขลบไหม: {has_negative}");
}
```

ผลลัพธ์:

```
ทุกตัวเป็นเลขคู่: true
มีเลขลบไหม: false
```

พิสูจน์ short-circuit ของ `.any()` ด้วย `.inspect()`:

```rust
fn main() {
    let numbers = vec![1, 3, 5, 4, 7, 9];

    let has_even = numbers
        .iter()
        .inspect(|n| println!("any() กำลังตรวจ: {n}"))
        .any(|&n| n % 2 == 0);

    println!("มีเลขคู่ไหม: {has_even}");
}
```

ผลลัพธ์:

```
any() กำลังตรวจ: 1
any() กำลังตรวจ: 3
any() กำลังตรวจ: 5
any() กำลังตรวจ: 4
มีเลขคู่ไหม: true
```

สังเกตว่า `7` และ `9` ไม่ถูกตรวจเลย — พอเจอ `4` (ตัวแรกที่เป็นเลขคู่) `.any()` คืน `true` ออกไปทันที

#### `.position()`, `.find()`, `.find_map()`: หา item หรือหา index พร้อม short-circuit

```rust
fn main() {
    let names = vec!["Alice", "Bob", "Carol", "Dave"];

    // .position() คืน Option<usize> — index ของ item แรกที่ผ่านเงื่อนไข
    let idx = names.iter().position(|&name| name == "Carol");
    println!("index ของ Carol: {:?}", idx);

    // .find() คืน Option<&Item> — item แรกที่ผ่านเงื่อนไข (ไม่ใช่ index)
    let found = names.iter().find(|&&name| name.starts_with('B'));
    println!("ชื่อแรกที่ขึ้นด้วย B: {:?}", found);

    // .find_map() รวม .find() + .map() — คืนค่าแรกที่ closure คืน Some(...) (ไม่ใช่แค่ตรวจ true/false)
    let scores = vec!["10", "abc", "20", "xyz"];
    let first_valid_number: Option<i32> =
        scores.iter().find_map(|s| s.parse::<i32>().ok());
    println!("เลขแรกที่ parse ผ่าน: {:?}", first_valid_number);
}
```

ผลลัพธ์:

```
index ของ Carol: Some(2)
ชื่อแรกที่ขึ้นด้วย B: Some("Bob")
เลขแรกที่ parse ผ่าน: Some(10)
```

**เปรียบเทียบ `.find()` กับ `.find_map()`**: `.find(predicate)` เทียบเท่ากับ `.filter(predicate).next()` —
ตรวจแค่ `true`/`false` แล้วคืน item ตัวแรกที่ผ่าน `.find_map(f)` เทียบเท่ากับ `.filter_map(f).next()` — closure
คืน `Option<T>` ตรง ๆ (ไม่ใช่ `bool`) แล้ว `.find_map()` คืน**ค่าข้างใน `Some` ตัวแรก**ที่เจอ ข้ามตัวที่คืน `None`
ไปเรื่อย ๆ ในตัวอย่างข้างบน `"10"` parse ผ่านได้ `10` ทันที (`Ok(10).ok() = Some(10)`) จึงเป็นคำตอบโดยไม่ต้องแตะ
`"abc"`, `"20"`, `"xyz"` เลย

### 26.3 เขียน iterator adaptor ของตัวเอง: ห่อ iterator ตัวอื่นไว้ข้างใน

Part 25 สอนให้เขียน custom iterator แบบ `Countdown` ที่ "สร้างค่าขึ้นมาเองจากศูนย์" (ไม่ได้ห่อ iterator ตัวอื่น
เลย — เก็บแค่ `current: u32` เป็น state ภายใน) หัวข้อนี้จะเขียน custom iterator แบบที่**ยากขึ้นอีกขั้น**:
**iterator adaptor ที่ห่อ iterator ตัวอื่นไว้ข้างใน** แล้วเพิ่ม logic ใหม่เข้าไประหว่างทาง — นี่คือรูปแบบเดียวกัน
เป๊ะกับที่ `.map()`, `.filter()`, `.peekable()` ทำงานภายในจริง ๆ (Part 25 หัวข้อ 25.10 เคยเขียน `MyMap` แบบง่าย
ให้ดูมาแล้ว — บทนี้จะไปให้ลึกกว่านั้น)

#### `Deduplicate<I>`: ข้าม item ที่ซ้ำกับตัวก่อนหน้าติด ๆ กัน

โจทย์: เขียน adaptor ที่รับ iterator ใดก็ได้ แล้วคืน iterator ใหม่ที่**ข้ามค่าที่ซ้ำกับตัวก่อนหน้าทันที** (ไม่ใช่
เอาค่าซ้ำทั้งหมดในทั้งเส้นออก — แค่ตัวที่ซ้ำ**ติดกัน**เท่านั้น เหมือน Unix command `uniq`):

```rust
struct Deduplicate<I: Iterator> {
    inner: I,
    last: Option<I::Item>,
}

impl<I: Iterator> Deduplicate<I> {
    fn new(inner: I) -> Self {
        Deduplicate { inner, last: None }
    }
}

impl<I: Iterator> Iterator for Deduplicate<I>
where
    I::Item: PartialEq + Clone,
{
    type Item = I::Item;

    fn next(&mut self) -> Option<I::Item> {
        loop {
            let item = self.inner.next()?; // ? บน Option: ถ้า inner หมดแล้ว (None) คืน None ออกไปทันที
            // ถ้ายังไม่มี "last" (นี่คือ item แรกสุด) หรือ item นี้ไม่เหมือน last ครั้งก่อน -> ให้ item นี้ออกไป
            if self.last.as_ref() != Some(&item) {
                self.last = Some(item.clone());
                return Some(item);
            }
            // ถ้าเหมือน last ครั้งก่อน -> ข้าม (loop กลับไปดึงตัวถัดไปจาก inner ต่อ ไม่ return)
        }
    }
}

fn main() {
    let data = vec![1, 1, 2, 2, 2, 3, 1, 1, 4];
    let deduped = Deduplicate::new(data.into_iter());
    let result: Vec<i32> = deduped.collect();
    println!("{:?}", result);
}
```

ผลลัพธ์:

```
[1, 2, 3, 1, 4]
```

**อ่านทีละส่วน — นี่คือจุดที่ยากขึ้นจริงจากระดับของ `Countdown`**:

- **`struct Deduplicate<I: Iterator> { inner: I, last: Option<I::Item> }`** — struct นี้ **generic บน `I`**
  (ตัว iterator ที่ถูกห่ออยู่ข้างใน) พร้อม bound `I: Iterator` ตั้งแต่ประกาศ struct — สังเกตว่าเราไม่ได้เก็บ
  "ตัวเลขปัจจุบัน" เหมือน `Countdown` แต่เก็บ**ทั้ง iterator ตัวอื่นไว้เป็น field** (`inner: I`) และเก็บ **item
  ตัวล่าสุดที่ปล่อยออกไปแล้ว** (`last: Option<I::Item>`) — นี่คือความแตกต่างสำคัญ: `Countdown` "เป็นแหล่งข้อมูล
  เอง" (source) แต่ `Deduplicate` "ห่อแหล่งข้อมูลอื่นไว้" (adaptor) — คำว่า `I::Item` ในนี้คือการเข้าถึง
  associated type ของ `I` ตามหลักการที่ Part 22 สอนไว้แล้ว (`I::Item` อ่านว่า "ชนิด `Item` ของ `I` โดยเฉพาะ")
- **`impl<I: Iterator> Iterator for Deduplicate<I> where I::Item: PartialEq + Clone`** — สังเกต **bound เพิ่มสอง
  ตัวบน `I::Item`**: `PartialEq` (เพื่อเทียบว่า item ปัจจุบัน "เหมือน" `last` ไหม ด้วย `!=`) และ `Clone` (เพื่อ
  เก็บสำเนาไว้เป็น `last` สำหรับรอบถัดไป โดยไม่ต้องยึด ownership ของค่าที่กำลังจะ `return` ออกไป) — นี่คือ
  where clause ที่ Part 22 หัวข้อ 22.3 สอนไว้ ใช้ตอนที่ bound ซับซ้อนกว่าจะเขียนในมุม `<>` ได้สะดวก
- **`fn next(&mut self) -> Option<I::Item>`** — logic คือ: ดึง item จาก `inner.next()` ด้วย `?` (ใช้ `?` บน
  `Option` ได้เหมือนบน `Result` — เชื่อม Part 12 ที่สอนว่า `?` ใช้กับทั้งสองชนิดได้ตามกฎเดียวกัน — ถ้า `inner`
  หมดแล้ว `None?` จะคืน `None` ออกจาก `next()` ทั้งฟังก์ชันทันที) แล้วเทียบกับ `self.last` — ถ้าไม่เหมือนกัน (
  หรือเป็น item แรกสุดที่ `last` ยังเป็น `None`) ก็อัปเดต `last` แล้ว `return Some(item)` ถ้าเหมือนกัน ก็**ไม่
  return** แต่ `loop` กลับไปดึงตัวถัดไปจาก `inner` ต่อทันที (ข้ามไปเรื่อย ๆ จนกว่าจะเจอตัวที่ไม่ซ้ำ หรือ `inner`
  หมดจริง ๆ)

จุดที่ควรสังเกตอย่างจริงจัง: **`Deduplicate<I>` implement `Iterator` ได้โดยไม่รู้จักเลยว่า `I` คืออะไรกันแน่**
(`Vec::into_iter()`, `Countdown`, หรือ iterator อะไรก็ตาม) — มันรู้แค่ว่า `I: Iterator` และ `I::Item` เทียบกันได้
กับ clone ได้ — เหมือนที่ default method ของ `Iterator` ใน Part 25 ไม่รู้จัก `Countdown` เลยแต่ใช้ได้กับมันได้
ทันที นี่คือพลังของ generic + trait bound ที่ Part 18/22 ปูพื้นไว้: เขียนโค้ดครั้งเดียว ใช้ได้กับ**ทุก**
iterator ที่มี `Item` เทียบกันได้ ไม่ใช่แค่กรณีเดียว

#### `Batched<I>`: รวม item เป็นชุดละ N ตัว (แบบเดียวกับ `.chunks()` แต่ใช้กับ iterator ใดก็ได้ ไม่จำกัดแค่ slice)

หัวข้อ 26.1 บอกไว้ว่า `.chunks()` เป็น method ของ slice เท่านั้น (ต้องมีข้อมูลต่อเนื่องในความจำอยู่แล้ว) ถ้า
อยากได้ผลแบบเดียวกันแต่ใช้กับ **iterator อะไรก็ได้** (เช่น iterator แบบ lazy ที่ยังไม่มีข้อมูลอยู่ในความจำเลย
อย่าง `Countdown` หรือ `(1..)`) ต้องเขียน adaptor เอง:

```rust
struct Batched<I: Iterator> {
    inner: I,
    batch_size: usize,
}

impl<I: Iterator> Batched<I> {
    fn new(inner: I, batch_size: usize) -> Self {
        assert!(batch_size > 0, "batch_size ต้องมากกว่า 0");
        Batched { inner, batch_size }
    }
}

impl<I: Iterator> Iterator for Batched<I> {
    type Item = Vec<I::Item>;

    fn next(&mut self) -> Option<Vec<I::Item>> {
        let mut batch = Vec::with_capacity(self.batch_size);
        for _ in 0..self.batch_size {
            match self.inner.next() {
                Some(item) => batch.push(item),
                None => break, // inner หมดกลางทาง -> หยุดสร้าง batch นี้ (อาจได้ batch เล็กกว่าที่ขอ)
            }
        }
        if batch.is_empty() {
            None // ไม่มีอะไรเหลือเลยแม้แต่ตัวเดียว -> ทั้ง Batched หมดแล้วจริง ๆ
        } else {
            Some(batch) // มีอย่างน้อย 1 ตัว (อาจไม่ครบ batch_size ถ้าเป็น batch สุดท้าย) -> ยังส่งออกไป
        }
    }
}

fn main() {
    let numbers = 1..=10;
    let batches: Vec<Vec<i32>> = Batched::new(numbers, 3).collect();
    println!("{:?}", batches);
}
```

ผลลัพธ์:

```
[[1, 2, 3], [4, 5, 6], [7, 8, 9], [10]]
```

**อ่านทีละส่วน**: `type Item = Vec<I::Item>;` — สังเกตว่า `Item` ของ `Batched<I>` **ไม่ใช่** `I::Item` ตรง ๆ
เหมือน `Deduplicate<I>` แต่เป็น **`Vec<I::Item>`** — item ที่ `Batched` ปล่อยออกมาแต่ละตัวคือ "ชุด" ของ item
เดิม ไม่ใช่ item เดิมตัวเดียว นี่คือตัวอย่างที่แสดงว่า custom adaptor **ไม่จำเป็นต้องคง `Item` ชนิดเดิมไว้เลย**
— เปลี่ยนชนิดของ item ได้อย่างสมบูรณ์ (เหมือนที่ `.map()` เปลี่ยนชนิดได้จาก Part 25) `next()` วนดึง item จาก
`inner` ไม่เกิน `batch_size` ครั้งเก็บไว้ใน `Vec` — ถ้า `inner` หมดกลางทาง (`None`) ก็ `break` ออกจาก loop
ทันที (ทำให้ batch สุดท้ายอาจมีสมาชิกน้อยกว่า `batch_size` เช่นตัวอย่างที่ได้ `[10]` แค่ตัวเดียว) แล้วเช็คว่า
`batch` ว่างหรือไม่ — ถ้าว่างสนิท (`inner` หมดตั้งแต่ก่อนเริ่ม batch นี้เลย) คืน `None` ให้ `Batched` เองหมดด้วย
ถ้ามีอย่างน้อย 1 ตัว ก็ยังปล่อย batch นั้นออกไป

ทั้ง `Deduplicate<I>` และ `Batched<I>` แสดงรูปแบบทั่วไปของการเขียน iterator adaptor เอง: **struct เก็บ inner
iterator ไว้เป็น field (บางครั้งเก็บ state เพิ่มเติมด้วย) แล้ว `next()` เรียก `self.inner.next()` ซ้ำ ๆ (อาจ
มากกว่า 1 ครั้งต่อการเรียก `next()` หนึ่งครั้ง) พร้อม logic ตัดสินใจของตัวเองว่าจะปล่อยอะไรออกไป** — นี่คือ
กลไกเดียวกันเป๊ะที่ `std::iter::Map<I, F>`, `std::iter::Filter<I, P>`, `std::iter::Peekable<I>` ใช้ภายในทั้งหมด
ไม่มีอะไรพิเศษกว่านี้เลย

### 26.4 Extension trait pattern: เพิ่ม `.deduplicate()` ให้ทุก iterator ในโลก

ตอนนี้เรามี `Deduplicate::new(iter)` ใช้งานได้แล้ว แต่การเขียน `Deduplicate::new(data.into_iter())` **ไม่
สามารถ chain ต่อในรูปแบบ method call ปกติได้** (`data.into_iter().deduplicate()` อ่านและ chain ได้ลื่นกว่ามาก
เพราะต่อกับ `.filter()`, `.map()` ในสายเดียวกันได้ทันที) เราต้องการเพิ่ม method `.deduplicate()` ให้กับ **ทุก
iterator ที่มีอยู่แล้วในโลก** โดยไม่ไปแก้ trait `Iterator` ของ std เอง (ซึ่งเป็นไปไม่ได้อยู่แล้ว — เราไม่ได้
เป็นเจ้าของ std) วิธีทำคือ **extension trait pattern**: ประกาศ trait ของเราเองที่มี default method พร้อม
**blanket implementation** ให้กับทุก type ที่ implement `Iterator` — กลไกเดียวกันเป๊ะกับ blanket
implementation ของ `IntoIterator` ที่ Part 25 หัวข้อ 25.6 โชว์ไว้ และตรงกับที่ Part 22 หัวข้อ 22.9 สอนไว้แล้ว

```rust
struct Deduplicate<I: Iterator> {
    inner: I,
    last: Option<I::Item>,
}

impl<I: Iterator> Deduplicate<I> {
    fn new(inner: I) -> Self {
        Deduplicate { inner, last: None }
    }
}

impl<I: Iterator> Iterator for Deduplicate<I>
where
    I::Item: PartialEq + Clone,
{
    type Item = I::Item;

    fn next(&mut self) -> Option<I::Item> {
        loop {
            let item = self.inner.next()?;
            if self.last.as_ref() != Some(&item) {
                self.last = Some(item.clone());
                return Some(item);
            }
        }
    }
}

// --- Extension trait pattern เริ่มจากนี่ ---

// trait ของเราเอง: กำหนดว่า "iterator อะไรก็ตามที่มี Item เทียบกันได้และ clone ได้ ต้องได้ .deduplicate() มาด้วย"
trait DeduplicateExt: Iterator {
    fn deduplicate(self) -> Deduplicate<Self>
    where
        Self: Sized,
        Self::Item: PartialEq + Clone,
    {
        Deduplicate::new(self)
    }
}

// blanket implementation: ให้ทุก type I ที่ implement Iterator ได้ DeduplicateExt มาโดยอัตโนมัติ
impl<I: Iterator> DeduplicateExt for I {}

fn main() {
    let data = vec![1, 1, 2, 2, 2, 3, 1, 1, 4];

    // ใช้งานเหมือนเป็น method มาตรฐานของ std เอง เพราะ blanket impl ให้มากับทุก iterator แล้ว
    let result: Vec<i32> = data.into_iter().deduplicate().collect();
    println!("{:?}", result);

    // chain ต่อกับ adaptor อื่นในสายเดียวกันได้ทันที เพราะ Deduplicate<Self> implement Iterator ปกติ
    let more_data = vec![5, 5, 5, 8, 2, 2, 9, 9, 1];
    let doubled_after_dedup: Vec<i32> = more_data
        .into_iter()
        .deduplicate()
        .map(|n| n * 2)
        .collect();
    println!("{:?}", doubled_after_dedup);
}
```

ผลลัพธ์:

```
[1, 2, 3, 1, 4]
[10, 16, 4, 18, 2]
```

**อ่านทีละส่วน — นี่คือรูปแบบที่ต้องจำให้แม่น เพราะมันคือกลไกจริงที่ `itertools` ใช้**:

- **`trait DeduplicateExt: Iterator { fn deduplicate(self) -> Deduplicate<Self> where Self: Sized, ... }`** —
  ประกาศ trait ใหม่ที่**สืบทอดจาก `Iterator`** (เขียน `: Iterator` หลังชื่อ trait คือ **supertrait bound** —
  บอกว่า "ใครจะ implement `DeduplicateExt` ได้ ต้อง implement `Iterator` ด้วยเสมอ" — concept นี้ Part 21 เกริ่น
  ไว้บ้างแล้วตอนพูดถึง trait ที่มีหลาย bound ซ้อนกัน) เขียน `fn deduplicate(...)` เป็น **default method** ทันที
  ในตัว trait เอง (ไม่ใช่แค่ signature เปล่า ๆ) — ทำให้ type ที่ implement trait นี้**ไม่ต้องเขียน
  `deduplicate()` เองเลย** ได้มาโดยอัตโนมัติจาก default method นี้ตรง ๆ (`Self: Sized` เป็น bound ที่จำเป็นเพราะ
  `deduplicate(self)` รับ `self` แบบ by-value — ถ้า `Self` ไม่การันตีว่ามีขนาดแน่นอน เช่นกรณี `dyn Iterator`
  compiler จะไม่ยอมให้ move มันแบบนี้ได้ — bound นี้จะมีความหมายชัดขึ้นในหัวข้อ 26.8 ตอนพูดถึง
  `Box<dyn Iterator>`)
- **`impl<I: Iterator> DeduplicateExt for I {}`** — นี่คือ **blanket implementation** ตัวจริง: บอกว่า "ทุก
  type `I` ที่ implement `Iterator` แล้ว ให้ implement `DeduplicateExt` ให้ด้วยโดยอัตโนมัติ" — body ของ `impl`
  เป็น `{}` เปล่า ๆ เพราะ `deduplicate()` เป็น default method อยู่แล้วในตัว trait เอง ไม่ต้อง implement เพิ่มอีก
  แม้แต่บรรทัดเดียว
- **ผลลัพธ์**: ตั้งแต่บรรทัดที่ `impl<I: Iterator> DeduplicateExt for I {}` ถูกประกาศ **ทุก iterator ทุกตัวใน
  โปรแกรมนี้** (`Vec::into_iter()`, `Countdown`, `(1..10)`, ทุกอย่าง) จะมี method `.deduplicate()` ให้เรียกใช้
  ทันที เหมือนมันเป็น method มาตรฐานของ std เองมาแต่ต้น — โดยที่ **เราไม่ได้แก้ trait `Iterator` ของ std เลย
  แม้แต่นิดเดียว** เพียงแค่ประกาศ trait ของเราเองแล้วปล่อยให้ blanket implementation ทำงานให้เอง

**ทำไมนี่คือกลไกจริงของ `itertools`**: crate ชื่อดัง `itertools` (ที่หลายโปรเจกต์ Rust production ใช้กันจริง)
ให้ method อย่าง `.unique()`, `.group_by()`, `.chunks()` (เวอร์ชันของมันเอง ที่ใช้กับ iterator ทั่วไปได้ ไม่ใช่
แค่ slice), `.sorted()`, ฯลฯ — ทั้งหมดนี้**ไม่ได้แก้ std เลย** มันประกาศ trait ชื่อ `Itertools` ที่มี
supertrait bound เป็น `Iterator` พร้อม default method นับสิบตัว แล้ว blanket implement ให้กับทุก type ที่
implement `Iterator` ด้วยรูปแบบเดียวกันเป๊ะกับที่เราเพิ่งเขียน `DeduplicateExt` — ต่างกันแค่จำนวน method และ
ความซับซ้อนของ adaptor ที่อยู่ข้างในเท่านั้นเอง ถ้าเข้าใจ `DeduplicateExt` ในตัวอย่างนี้ ก็เข้าใจ**หลักการทั้งหมด**
ที่ `itertools` (และ extension trait pattern ทั่วไปในภาษา Rust) สร้างขึ้นบนแล้ว

### 26.5 `Batched<I>` ผ่าน extension trait เช่นกัน: `.batched()`

ทำแบบเดียวกันกับ `Batched<I>` จากหัวข้อ 26.3 ให้เห็นว่า pattern นี้ใช้ซ้ำได้กับ custom adaptor ตัวไหนก็ได้:

```rust
struct Batched<I: Iterator> {
    inner: I,
    batch_size: usize,
}

impl<I: Iterator> Iterator for Batched<I> {
    type Item = Vec<I::Item>;

    fn next(&mut self) -> Option<Vec<I::Item>> {
        let mut batch = Vec::with_capacity(self.batch_size);
        for _ in 0..self.batch_size {
            match self.inner.next() {
                Some(item) => batch.push(item),
                None => break,
            }
        }
        if batch.is_empty() {
            None
        } else {
            Some(batch)
        }
    }
}

trait BatchedExt: Iterator {
    fn batched(self, batch_size: usize) -> Batched<Self>
    where
        Self: Sized,
    {
        assert!(batch_size > 0, "batch_size ต้องมากกว่า 0");
        Batched { inner: self, batch_size }
    }
}

impl<I: Iterator> BatchedExt for I {}

fn main() {
    // chain เต็มรูปแบบ: range -> filter -> batched -> map -> collect ทั้งหมดในสายเดียว
    let result: Vec<i32> = (1..=20)
        .filter(|n| n % 2 == 0)
        .batched(3)
        .map(|batch: Vec<i32>| batch.iter().sum())
        .collect();

    println!("{:?}", result);
}
```

ผลลัพธ์:

```
[12, 30, 48, 20]
```

**อธิบาย**: `(1..=20).filter(|n| n % 2 == 0)` ได้เลขคู่ 1-20 ทั้งหมด (`2, 4, 6, ..., 20` รวม 10 ตัว) `.batched(3)`
รวมเป็นชุดละ 3: `[2,4,6], [8,10,12], [14,16,18], [20]` (ชุดสุดท้ายมีตัวเดียวเพราะ 10 หารด้วย 3 ไม่ลงตัว) `.map(|batch|
batch.iter().sum())` รวมผลรวมของแต่ละชุด: `2+4+6=12`, `8+10+12=30`, `14+16+18=48`, `20=20` ได้ผลลัพธ์สุดท้าย
`[12, 30, 48, 20]` ตรงกับที่ compile รันจริง สิ่งสำคัญคือรูปแบบของโค้ด (`.filter().batched().map().collect()`)
เป็น chain เดียวยาวที่อ่านลื่นไม่ต่างจาก adaptor มาตรฐานของ std เลย — นี่คือเป้าหมายของ extension trait
pattern: ทำให้ custom adaptor "กลมกลืน" ไปกับ chain ปกติอย่างสมบูรณ์

### 26.6 `.collect()` เข้ากับ target ที่ฉลาดกว่า: `Result<Vec<T>, E>` และ `Option<Vec<T>>`

Part 25 หัวข้อ 25.12 สอน `.collect()` เข้ากับ `Vec<T>`, `String`, `HashMap<K, V>`, `HashSet<T>` — หัวข้อนี้จะ
โชว์ target ที่ทรงพลังกว่านั้นมาก และเป็นที่ผู้เขียนโค้ด Rust มือใหม่มักไม่รู้ว่ามีอยู่: **`.collect()` เป็น
`Result<Vec<T>, E>` จาก iterator ของ `Result<T, E>`** — โดย**หยุดทันทีที่เจอ `Err` ตัวแรก** (short-circuit
เหมือน `.try_fold()`):

```rust
fn main() {
    let inputs = vec!["1", "2", "3", "4", "5"];

    // แต่ละ .parse() คืน Result<i32, ParseIntError> -> iterator ของ Result
    // .collect::<Result<Vec<i32>, _>>() รวมทุก Ok เป็น Vec เดียว หรือคืน Err ตัวแรกทันทีถ้ามีตัวไหนพัง
    let parsed: Result<Vec<i32>, std::num::ParseIntError> =
        inputs.iter().map(|s| s.parse::<i32>()).collect();

    println!("ทุกตัว parse ผ่าน: {:?}", parsed);

    let bad_inputs = vec!["1", "2", "not_a_number", "4", "5"];
    let parsed2: Result<Vec<i32>, std::num::ParseIntError> =
        bad_inputs.iter().map(|s| s.parse::<i32>()).collect();

    println!("มีตัวที่ parse ไม่ผ่าน: {:?}", parsed2);
}
```

ผลลัพธ์:

```
ทุกตัว parse ผ่าน: Ok([1, 2, 3, 4, 5])
มีตัวที่ parse ไม่ผ่าน: Err(ParseIntError { kind: InvalidDigit })
```

**ทำไมสิ่งนี้ทำงานได้ (และทำไมมันน่าประหลาดใจสำหรับมือใหม่)**: `.collect()` ทำงานผ่าน trait `FromIterator`
(ตามที่ Part 25 อธิบายไว้) — และ `Result<T, E>` (สำหรับ `T` ที่ implement `FromIterator` เองด้วย เช่น `Vec<T>`)
**implement `FromIterator<Result<T, E>>` ให้กับตัวเอง** โดย std library ไว้ล่วงหน้าแล้ว logic ภายในของมันคือ
"ลองรวบรวมทุก `Ok` เข้า `Vec` ไปเรื่อย ๆ แต่พอเจอ `Err` ตัวแรก ให้หยุดทันทีและคืน `Err` นั้นออกไปเป็นผลลัพธ์
สุดท้ายของทั้ง `.collect()` เลย" — พิสูจน์ short-circuit นี้ได้ด้วย `.inspect()` เหมือนเดิม:

```rust
fn main() {
    let bad_inputs = vec!["1", "2", "not_a_number", "4", "5"];

    let parsed: Result<Vec<i32>, std::num::ParseIntError> = bad_inputs
        .iter()
        .inspect(|s| println!("กำลัง parse: {s}"))
        .map(|s| s.parse::<i32>())
        .collect();

    println!("ผลลัพธ์: {:?}", parsed);
}
```

ผลลัพธ์:

```
กำลัง parse: 1
กำลัง parse: 2
กำลัง parse: not_a_number
ผลลัพธ์: Err(ParseIntError { kind: InvalidDigit })
```

`"4"` และ `"5"` ไม่ถูก `.inspect()` เห็นเลย — เพราะ `.collect()` (ผ่าน `FromIterator` ของ `Result`) เรียก
`.next()` ของ iterator ต้นทาง (`.map()` chain) น้อยครั้งกว่าจำนวน item ทั้งหมด พอเจอ `Err` มันไม่เรียก
`.next()` ต่ออีกเลย

**pattern เดียวกันใช้กับ `Option<Vec<T>>` ได้เช่นกัน** (สำหรับ iterator ของ `Option<T>`):

```rust
fn main() {
    let raw = vec!["10", "20", "30"];
    let maybe_numbers: Vec<Option<i32>> = raw.iter().map(|s| s.parse::<i32>().ok()).collect();
    println!("{:?}", maybe_numbers);

    // .collect() เป็น Option<Vec<T>>: ถ้าทุกตัวเป็น Some ได้ Some(Vec ของค่าที่แกะออกมาแล้ว)
    // ถ้ามีสักตัวเป็น None ทั้งเส้นได้ None ทันที (short-circuit เหมือนกับ Result)
    let all_present: Option<Vec<i32>> = raw.iter().map(|s| s.parse::<i32>().ok()).collect();
    println!("{:?}", all_present);

    let raw_with_bad = vec!["10", "not_a_number", "30"];
    let with_missing: Option<Vec<i32>> =
        raw_with_bad.iter().map(|s| s.parse::<i32>().ok()).collect();
    println!("{:?}", with_missing);
}
```

ผลลัพธ์:

```
[Some(10), Some(20), Some(30)]
Some([10, 20, 30])
None
```

**ประโยชน์ในทางปฏิบัติ**: pattern นี้มีค่ามากตอนต้อง "validate ทั้ง batch ข้อมูลแล้วเอาผลรวม" — เขียนโค้ด
validate item เดียว (คืน `Result<T, E>` หรือ `Option<T>`) แล้วปล่อยให้ `.collect()` จัดการ short-circuit ทั้งหมด
ให้เอง โดยไม่ต้องเขียน loop มือเปล่าเช็ค error ทีละตัวเลยแม้แต่บรรทัดเดียว — ตัวอย่างที่สมบูรณ์แบบของแนวคิด
"ทำให้ error handling เป็นส่วนหนึ่งของ type system" ที่ Part 12 ปูพื้นไว้ตั้งแต่ต้น

### 26.7 `Box<dyn Iterator>` และการมองข้ามข้อจำกัดของ static dispatch — เชื่อม Part 21

ก่อนพูดถึง performance เต็มรูปแบบในหัวข้อถัดไป มาดูสถานการณ์หนึ่งที่ต้องใช้ `Box<dyn Iterator>` ก่อน:
บางครั้งเราต้องการฟังก์ชันที่**คืน iterator ชนิดต่างกัน** ขึ้นอยู่กับเงื่อนไข (เช่น `if flag { ... } else {
... }` ที่แต่ละฝั่งสร้าง adaptor chain คนละแบบกัน — ชนิดจริงของ `.filter().map()` กับ `.map().filter()` เป็น
**คนละ concrete type กันเลย** แม้จะดูคล้ายกัน) แบบนี้ generic return type ธรรมดาทำไม่ได้ (ฟังก์ชันหนึ่งต้องคืน
type เดียวเสมอ) ต้องใช้ trait object `Box<dyn Iterator<Item = T>>` ตามที่ Part 21 สอนไว้:

```rust
fn make_iterator(include_odd: bool) -> Box<dyn Iterator<Item = i32>> {
    let base = 1..=10;
    if include_odd {
        // สาย if: chain แบบ filter คู่ -> คนละ concrete type กับสาย else
        Box::new(base.filter(|n| n % 2 == 0))
    } else {
        // สาย else: chain แบบ map แล้ว filter -> คนละ concrete type อีกแบบ
        Box::new(base.map(|n| n * 10).filter(|n| n % 3 == 0))
    }
}

fn main() {
    let result1: Vec<i32> = make_iterator(true).collect();
    let result2: Vec<i32> = make_iterator(false).collect();
    println!("{:?}", result1);
    println!("{:?}", result2);
}
```

ผลลัพธ์:

```
[2, 4, 6, 8, 10]
[30, 60, 90]
```

**เหตุผลที่ต้องใช้ `Box<dyn Iterator<Item = i32>>`**: `base.filter(...)` มี concrete type จริงคือ
`std::iter::Filter<std::ops::RangeInclusive<i32>, {closure type}>` ส่วน `base.map(...).filter(...)` มี concrete
type คือ `std::iter::Filter<std::iter::Map<std::ops::RangeInclusive<i32>, {closure type}>, {closure type}>` —
**คนละ type กันโดยสิ้นเชิง** (ทั้งที่ทั้งสองมี `Item = i32` เหมือนกัน) ถ้าฟังก์ชันประกาศ return type แบบ
concrete ตรง ๆ จะ compile ไม่ผ่านเพราะสอง branch ของ `if/else` คืนคนละ type กัน `Box<dyn Iterator<Item = i32>>`
แก้ปัญหานี้ได้เพราะมันซ่อนชนิดจริงไว้หลัง**dynamic dispatch** — ทุก concrete type ที่ implement
`Iterator<Item = i32>` แปลงเป็น `Box<dyn Iterator<Item = i32>>` ได้เหมือนกันหมด (concept ที่ Part 21 สอนไว้
เต็มรูปแบบเรื่อง `dyn Trait`/trait object)

นี่คือเหตุผลที่ `deduplicate()` (หัวข้อ 26.4) และ `batched()` (หัวข้อ 26.5) ต้องมี bound `Self: Sized` — ถ้า
`Self` คือ `dyn Iterator` (ไม่มีขนาดแน่นอนตอน compile time) การ `deduplicate(self)` ที่รับ `self` แบบ by-value
จะทำไม่ได้เลย (ย้ายของที่ไม่รู้ขนาดไปไหนไม่ได้) ต้องเรียกผ่าน reference หรือ `Box` เสมอสำหรับกรณี `dyn
Iterator` — นี่คือข้อจำกัดจริงที่เจอเวลาผสม extension trait pattern กับ trait object เข้าด้วยกัน

ข้อแลกเปลี่ยนของ `Box<dyn Iterator>` คือหัวข้อที่จะอธิบายเต็มรูปแบบต่อไปนี้

### 26.8 Performance เจาะลึก: iterator fusion, monomorphization, และข้อแลกเปลี่ยนของ `dyn Iterator`

นี่คือหัวข้อที่พิสูจน์คำกล่าวอ้างที่ได้ยินบ่อยที่สุดเกี่ยวกับ Rust iterator: **"iterator chain เร็วเท่ากับ (หรือ
เร็วกว่า) loop มือเปล่าที่เขียนเอง"** — Part 25 เกริ่นแนวคิดนี้ไว้สั้น ๆ ผ่านการเขียน `MyMap` ให้ดูว่ามันเป็น
struct ธรรมดา หัวข้อนี้จะอธิบายกลไกที่ทำให้เป็นจริงอย่างละเอียดกว่า

#### ทำไม adaptor chain ไม่ได้แปลว่า "หลาย loop ซ้อนกัน"

มือใหม่หลายคนกังวลว่าโค้ดแบบนี้:

```rust
fn sum_of_even_squares(numbers: &[i32]) -> i32 {
    numbers
        .iter()
        .filter(|&&n| n % 2 == 0)
        .map(|&n| n * n)
        .sum()
}
```

จะ "ช้ากว่า" loop มือเปล่าแบบนี้ เพราะดูเหมือนมี "หลายขั้นตอน" ต่อกัน (filter ต้องวนรอบหนึ่ง, map ต้องวนอีกรอบ,
sum ต้องวนอีกรอบ — รวมเป็น 3 รอบ?):

```rust
fn sum_of_even_squares_manual(numbers: &[i32]) -> i32 {
    let mut total = 0;
    for &n in numbers {
        if n % 2 == 0 {
            total += n * n;
        }
    }
    total
}
```

**ความเข้าใจนี้ผิด** — ทั้งสองฟังก์ชันคอมไพล์ออกมาเป็น**โค้ด machine code ที่เทียบเท่ากันโดยพื้นฐาน** (มีรอบวน
ข้อมูลแค่**รอบเดียว**ทั้งคู่) เหตุผลคือสิ่งที่เรียกว่า **iterator fusion** (บางครั้งเรียก loop fusion) —
กระบวนการนี้เกิดจากการรวมกันของ 3 อย่าง:

1. **แต่ละ adaptor struct มีขนาดเล็กมากและไม่มี virtual dispatch เลย** — `Filter<I, P>` เก็บแค่ `iter: I` กับ
   `predicate: P`, `Map<I, F>` เก็บแค่ `iter: I` กับ `f: F` — ไม่มี pointer ไปยัง vtable, ไม่มี heap allocation,
   ทุกอย่างอยู่บน stack (หรือถูก inline ไปเป็น register โดยตรง)
2. **`next()` ของแต่ละ adaptor เรียก `next()` ของตัวที่ห่ออยู่ข้างในตรง ๆ** — เหมือนที่ `Deduplicate::next()`
   ในหัวข้อ 26.3 เรียก `self.inner.next()` ตรง ๆ — นี่แปลว่าเรียก `.sum()` บน `Map<Filter<Iter<...>, P>, F>`
   สุดท้ายแล้วก็คือการเรียก `next()` ที่ **ซ้อนกันเป็นชั้น ๆ** (`Map::next()` เรียก `Filter::next()` เรียก
   `Iter::next()`) — เขียนเป็น pseudocode:

```
// นี่คือรูปแบบที่ next() ของ chain ทั้งเส้นทำงานจริง (ย่อให้เห็นภาพ)
Map::next(self) {
    loop {
        let filtered = Filter::next(&mut self.iter)?;  // อาจ loop ข้างในถ้า filter ไม่ผ่าน
        return Some((self.f)(filtered));
    }
}
Filter::next(self) {
    loop {
        let item = Iter::next(&mut self.iter)?;
        if (self.predicate)(&item) {
            return Some(item);
        }
        // ไม่ผ่าน predicate -> loop กลับไปดึงตัวถัดไปจาก Iter ต่อทันที (ไม่ return เลย)
    }
}
```

3. **compiler ทำ aggressive inlining** — เพราะทุก struct เล็กและทุก method ไม่มี virtual dispatch (เป็น
   **static dispatch** ล้วน ๆ — รู้ concrete type ของทุกชั้นตั้งแต่ compile time ผ่าน**monomorphization** ที่
   Part 18 สอนไว้: `.filter(predicate)` และ `.map(f)` เป็น generic method ที่ compiler จะสร้างโค้ดเฉพาะเจาะจง
   ให้กับ concrete type ของ `predicate`/`f`/`I` ทุกตัวจริง ๆ ไม่ใช่แค่ประกาศ generic ไว้ลอย ๆ) LLVM (backend
   ของ `rustc`) จึง**inline ทุกชั้นของ `next()` ที่ซ้อนกันให้แบนเป็นโค้ดเส้นเดียว** — ผลลัพธ์สุดท้ายที่ได้จาก
   `sum_of_even_squares()` คือ loop เดียวที่มีแค่การเช็คเงื่อนไข `n % 2 == 0` กับการคูณ `n * n` กับการบวกสะสม —
   **เหมือนกันตัวต่อตัว**กับที่ `sum_of_even_squares_manual()` เขียนไว้ตรง ๆ

**ทำไมนี่คือ "zero-cost abstraction" ตามคำนิยามที่ Part 1 เกริ่นไว้**: หลักการ zero-cost abstraction ของ Rust
(และ C++) คือ "สิ่งที่คุณไม่ได้ใช้ ไม่ต้องจ่ายต้นทุนอะไรเลย และสิ่งที่คุณใช้ ก็ไม่มีทางเขียนโค้ดมือเปล่าให้เร็ว
กว่าที่ compiler ทำให้ได้อีกแล้ว" — iterator chain เป็นตัวอย่างที่ชัดที่สุดของหลักการนี้ในทั้ง standard
library: การเขียนโค้ดในระดับ**นามธรรมสูง** (`.filter().map().sum()` — สื่อเจตนาตรง ๆ ไม่ต้องคิดเรื่อง index,
เงื่อนไขขอบ, หรือ mutable state เอง) ให้ผลลัพธ์ทาง performance **เท่ากับ**การเขียนโค้ด**นามธรรมต่ำ** (loop
มือเปล่า จัดการ index เอง) ทุกประการ ไม่มีการแลกอะไรเลยแม้แต่นิดเดียว — นี่ต่างจากภาษาที่ใช้ virtual
method/interface สำหรับทุก abstraction (เช่น Java's `Stream<T>` ที่มี boxing/unboxing และ virtual call
overhead ในหลายกรณี หรือ Python's generator ที่มี interpreter overhead ทุกการเรียก `next()`) ซึ่งการเพิ่ม
abstraction มักมาพร้อมต้นทุนที่วัดได้จริงเสมอ

#### ข้อแลกเปลี่ยนที่ต้องรู้จริง: `Box<dyn Iterator>` ไม่ fusion เท่าเดิม

หัวข้อ 26.7 แสดง `Box<dyn Iterator<Item = i32>>` ว่าจำเป็นเมื่อต้องคืน iterator คนละ concrete type จาก branch
ต่างกัน — แต่ต้องพูดตรง ๆ ว่านี่ **ไม่ใช่ของฟรี**: การใช้ `dyn Iterator` เปลี่ยนจาก **static dispatch** (ที่
compiler รู้ concrete type ทุกชั้นแน่นอนตั้งแต่ compile time ตามที่อธิบายไว้ข้างบน) ไปเป็น **dynamic dispatch**
(เรียกผ่าน vtable — ตารางของ function pointer ที่ต้อง lookup ตอน runtime ตามที่ Part 21 สอนไว้)

ผลที่ตามมา:

1. **`next()` ของ `dyn Iterator` เรียกผ่าน vtable ทุกครั้ง** — เป็นการเรียก function pointer indirect ไม่ใช่
   การเรียกฟังก์ชันตรง ๆ ที่ compiler inline ได้ — compiler **ไม่สามารถ inline ผ่าน vtable boundary ได้เลย**
   (เพราะไม่รู้ตอน compile time ว่า concrete type ข้างหลัง `dyn Iterator` คืออะไรกันแน่ — นั่นคือความหมายของ
   "dynamic" นั่นเอง)
2. **หมายความว่า chain adaptor ที่อยู่ "ข้างใน" `Box<dyn Iterator>` (เช่น `.filter()`ที่เขียนไว้ก่อนห่อด้วย
   `Box::new()`) ยัง fusion กันเองได้ตามปกติ** (เพราะพวกมันยัง static dispatch กันเองอยู่ข้างในกล่อง) แต่ **จุด
   ที่เรียก `.next()` ของกล่อง `Box<dyn Iterator>` จากภายนอก (เช่นตอน `for` loop หรือ `.collect()` เรียกกับผลลัพธ์
   ที่ `make_iterator()` คืนมา) จะเสียค่า indirect call หนึ่งครั้งต่อการเรียก `.next()` หนึ่งครั้งเสมอ** — ไม่มาก
   (แค่หนึ่ง pointer lookup ต่อ item) แต่ **ไม่ใช่ศูนย์** แบบที่ static dispatch chain ทำได้
3. นอกจากนี้ `Box<dyn Iterator>` ยังมี **heap allocation หนึ่งครั้ง** ตอนสร้าง (`Box::new(...)`) ซึ่ง static
   dispatch chain ไม่มีเลย (ทุกอย่างอยู่บน stack)

**เมื่อไหร่ควรใช้ `Box<dyn Iterator>` แล้ว**: เมื่อความยืดหยุ่นด้าน type (คืน iterator คนละ concrete type จาก
branch ต่างกัน, เก็บ iterator หลายชนิดไว้ใน collection เดียวกัน เช่น `Vec<Box<dyn Iterator<Item = i32>>>`,
หรือส่ง iterator ผ่าน API boundary ที่ไม่อยากประกาศ concrete type ที่ซับซ้อนยาวเป็นบรรทัด) **สำคัญกว่า**
performance ระดับ nanosecond ต่อ item ซึ่งในโค้ด production ส่วนใหญ่ (ที่ไม่ใช่ hot loop ที่ต้องรันหลายล้านครั้ง
ต่อวินาที) ความแตกต่างนี้**วัดไม่ออกเลยในทางปฏิบัติ** — หลักการทั่วไปคือ: **เขียน generic (`impl Iterator<Item
= T>` หรือ generic parameter `I: Iterator<Item = T>`) เป็นค่าเริ่มต้นเสมอเพื่อได้ static dispatch + fusion เต็ม
รูปแบบ แล้วเปลี่ยนไปใช้ `Box<dyn Iterator>` เฉพาะเมื่อ**สถานการณ์บังคับจริง ๆ** (คืนคนละ type จาก branch ต่างกัน
แบบที่ generic function ทำไม่ได้) — ไม่ใช่ใช้ `dyn` เป็นค่าเริ่มต้นแล้วค่อยมาคิดเรื่อง performance ทีหลัง

#### เรื่อง `size_hint()` กับ fusion: ทวนจาก Part 25 ในมุม performance

Part 25 หัวข้อ 25.2 อธิบายว่า `size_hint()` ช่วยให้ `.collect()` เรียก `Vec::with_capacity()` ล่วงหน้าได้ — เมื่อ
รวมกับ fusion ในหัวข้อนี้ จะเห็นภาพครบ: adaptor อย่าง `.map()` และ `.filter()` propagate (ส่งต่อ) `size_hint()`
ของตัวที่ห่ออยู่ข้างในออกไปด้วย (`.map()` คง `size_hint()` เดิมไว้เพราะ map แต่ละ item หนึ่งต่อหนึ่งไม่เปลี่ยน
จำนวน แต่ `.filter()` ทำได้แค่บอก "ขอบบน" เท่าเดิม เพราะไม่รู้ล่วงหน้าว่าจะเหลือกี่ตัวจนกว่าจะเช็คจริง) — นี่คือ
อีกจุดที่ static dispatch ช่วย: compiler รู้ตั้งแต่ compile time ว่า `size_hint()` ของ chain ทั้งเส้นคำนวณ
อย่างไร และ inline การคำนวณนั้นไปพร้อมกับทุกอย่างที่เหลือ — ถ้าเป็น `dyn Iterator` การเรียก `size_hint()` ก็ต้อง
ผ่าน vtable เช่นกัน (แต่เรียกแค่ครั้งเดียวตอนเริ่ม ไม่ใช่ต่อ item เหมือน `next()` เพราะฉะนั้นต้นทุนนี้เล็กกว่า
มาก ไม่ใช่ปัจจัยหลักที่ต้องกังวล)

### 26.9 Rayon: เมื่อ iterator chain เดียวกันขยายไปสู่ parallel iteration (แค่รู้จักไว้ — นอกขอบเขตของบทนี้)

ทุกอย่างที่เรียนมาในบทนี้และ Part 25 เป็น iterator แบบ **sequential** (วนทีละ item บน thread เดียว) — แต่ในโลก
จริง เมื่อข้อมูลมีจำนวนมาก (หลักล้านรายการขึ้นไป) และ CPU มีหลาย core ว่างอยู่ การประมวลผลแบบ sequential ล้วน ๆ
อาจไม่ใช้ hardware ที่มีอยู่ให้เต็มประสิทธิภาพ

crate ชื่อ **`rayon`** (ไม่ใช่ส่วนหนึ่งของ std — ต้องเพิ่มเป็น dependency ใน `Cargo.toml` ตามที่ Part 17 สอนไว้
เรื่อง crate ภายนอก) แก้ปัญหานี้ด้วยการให้ **`.par_iter()`** ที่ทำงาน**เหมือน** `.iter()` ทุกประการในเชิง API —
รับ adaptor chain แบบเดียวกัน (`.filter()`, `.map()`, `.sum()`, ฯลฯ) **แต่กระจายงานไปทำพร้อมกันในหลาย thread
โดยอัตโนมัติ**:

```
// นี่คือตัวอย่าง pseudocode แสดงแนวคิด — ไม่ใช่โค้ดที่ compile ได้ในบทนี้
// เพราะต้องเพิ่ม rayon = "1" ใน Cargo.toml ก่อน (ซึ่งอยู่นอกขอบเขตของบทนี้ที่จำกัดอยู่ที่ std เท่านั้น)
//
// use rayon::prelude::*;
//
// fn sum_of_squares_parallel(numbers: &[i64]) -> i64 {
//     numbers
//         .par_iter()              // เปลี่ยนจาก .iter() เป็น .par_iter() แค่คำเดียว
//         .filter(|&&n| n % 2 == 0)  // adaptor chain เดิมทุกตัว ทำงานเหมือนกันทุกประการ
//         .map(|&n| n * n)
//         .sum()                     // rayon จัดการรวมผลจากหลาย thread ให้เองอัตโนมัติ
// }
```

**สิ่งที่ควรรู้ (แค่ระดับความตระหนัก ไม่ต้องเจาะลึกในบทนี้)**:

- ปรัชญาของ `rayon` คือ **"เปลี่ยน `.iter()` เป็น `.par_iter()` แล้วโค้ด adaptor chain ที่เหลือแทบไม่ต้องแก้เลย"**
  — เพราะ `rayon` implement trait ของตัวเองที่ชื่อคล้ายกัน (`ParallelIterator`) ให้มี method หน้าตาเหมือน
  `Iterator` เกือบทุกตัว (`.filter()`, `.map()`, `.sum()`, `.collect()`, ฯลฯ) ผ่านหลักการเดียวกันกับ extension
  trait pattern ที่เพิ่งเรียนในหัวข้อ 26.4 (แม้ `rayon` จะไม่ได้ implement `std::iter::Iterator` ตรง ๆ เพราะ
  `Iterator` ถูกออกแบบมาสำหรับ sequential โดยธรรมชาติ — `ParallelIterator` เป็น trait คนละตัวที่ตั้งใจให้ API
  หน้าตาคล้ายกันมากที่สุดเพื่อให้ผู้ใช้ย้ายมาใช้ได้ง่าย)
- `rayon` ใช้เทคนิค **work-stealing scheduler** แบ่งงานให้ thread pool ที่มีอยู่โดยอัตโนมัติ — ผู้เขียนโค้ดไม่
  ต้องจัดการ thread เอง ไม่ต้องล็อค mutex เอง (สำหรับ pattern การใช้งานทั่วไปที่ไม่มี shared mutable state
  ระหว่าง item)
- **ข้อควรระวัง**: parallel ไม่ได้แปลว่าเร็วกว่าเสมอ — สำหรับข้อมูลจำนวนน้อย (หลักสิบ-หลักร้อย) หรือ closure ที่
  ทำงานเร็วมาก ต้นทุนของการแบ่งงานข้าม thread (thread synchronization overhead) อาจสูงกว่าเวลาที่ประหยัดได้จาก
  การทำงานพร้อมกัน — ต้อง benchmark จริงก่อนสรุปว่าคุ้มค่าเสมอ
- บทนี้และหลักสูตรทั้งเล่ม (ตามขอบเขตที่กำหนด) **ไม่ลงรายละเอียดการใช้งาน `rayon` เต็มรูปแบบ** (การตั้งค่า
  thread pool, `par_bridge()`, การจัดการ shared state ข้าม thread ที่ปลอดภัยผ่าน `Send`/`Sync`) — จุดสำคัญคือ
  **รู้ว่ามันมีอยู่ และรู้ว่า mental model เดียวกันของ iterator adaptor chain ที่เรียนมาตลอด Part 25-26 ต่อยอด
  ไปสู่โลก parallel ได้โดยไม่ต้องเรียนรู้ API ใหม่ทั้งหมด** — เมื่อถึงจุดที่ต้อง optimize performance ระดับข้อมูล
  ขนาดใหญ่จริง ๆ ในอนาคต ทักษะ iterator chain ที่แน่นจาก Part 25-26 นี้คือรากฐานที่ `rayon` ใช้ต่อยอดตรง ๆ

### 26.10 ตัวอย่างโลกจริงแบบสมบูรณ์: ระบบประมวลผล log

มาผสมทุกเทคนิคในบทนี้เข้าด้วยกันในตัวอย่างเดียว: โปรแกรมประมวลผลไฟล์ log ของเซิร์ฟเวอร์ที่ต้อง (1) แยกส่วนแต่ละ
บรรทัด (2) กรองเอาแต่ log level ที่สนใจ (3) ดึง timestamp ออกมา (4) ลบข้อความซ้ำที่เกิดติดกัน (ใช้
`Deduplicate<I>` ที่เขียนไว้ในหัวข้อ 26.3-26.4) (5) สรุปผลรวม

```rust
use std::collections::HashMap;

// --- ส่วนที่ 1: custom adaptor + extension trait จากหัวข้อ 26.3-26.4 (ใช้ซ้ำในตัวอย่างนี้) ---

struct Deduplicate<I: Iterator> {
    inner: I,
    last: Option<I::Item>,
}

impl<I: Iterator> Iterator for Deduplicate<I>
where
    I::Item: PartialEq + Clone,
{
    type Item = I::Item;

    fn next(&mut self) -> Option<I::Item> {
        loop {
            let item = self.inner.next()?;
            if self.last.as_ref() != Some(&item) {
                self.last = Some(item.clone());
                return Some(item);
            }
        }
    }
}

trait DeduplicateExt: Iterator {
    fn deduplicate(self) -> Deduplicate<Self>
    where
        Self: Sized,
        Self::Item: PartialEq + Clone,
    {
        Deduplicate { inner: self, last: None }
    }
}

impl<I: Iterator> DeduplicateExt for I {}

// --- ส่วนที่ 2: โมเดลข้อมูลของ log entry ---

#[derive(Debug, Clone, PartialEq)]
enum LogLevel {
    Info,
    Warning,
    Error,
}

#[derive(Debug, Clone, PartialEq)]
struct LogEntry {
    timestamp: String,
    level: LogLevel,
    message: String,
}

// แปลง 1 บรรทัดดิบ (รูปแบบ "TIMESTAMP LEVEL message") เป็น LogEntry
// คืน Result เพื่อให้ .collect() ใน parse_all short-circuit ได้ตามหัวข้อ 26.6
fn parse_line(line: &str) -> Result<LogEntry, String> {
    let mut parts = line.splitn(3, ' ');

    let timestamp = parts
        .next()
        .ok_or_else(|| format!("บรรทัดไม่มี timestamp: {line:?}"))?;
    let level_str = parts
        .next()
        .ok_or_else(|| format!("บรรทัดไม่มี log level: {line:?}"))?;
    let message = parts
        .next()
        .ok_or_else(|| format!("บรรทัดไม่มีข้อความ: {line:?}"))?;

    let level = match level_str {
        "INFO" => LogLevel::Info,
        "WARN" => LogLevel::Warning,
        "ERROR" => LogLevel::Error,
        other => return Err(format!("log level ไม่รู้จัก: {other:?}")),
    };

    Ok(LogEntry {
        timestamp: timestamp.to_string(),
        level,
        message: message.to_string(),
    })
}

// parse ทุกบรรทัด แล้ว .collect() เป็น Result<Vec<LogEntry>, String> (หัวข้อ 26.6)
// ถ้าบรรทัดไหน parse ไม่ผ่าน ทั้งฟังก์ชันคืน Err ตัวแรกทันที
fn parse_all(lines: &[&str]) -> Result<Vec<LogEntry>, String> {
    lines.iter().map(|line| parse_line(line)).collect()
}

#[derive(Debug)]
struct LogSummary {
    total_entries: usize,
    error_count: usize,
    warning_count: usize,
    unique_error_messages: Vec<String>,
    first_error_timestamp: Option<String>,
}

// ประมวลผลหลักตามที่โจทย์ต้องการ: กรอง level, ดึง timestamp, dedup ข้อความซ้ำติดกัน, สรุปผล
fn summarize_errors_and_warnings(entries: &[LogEntry]) -> LogSummary {
    // ใช้ .fold() (หัวข้อ 26.2) นับจำนวนแต่ละ level ในการวนครั้งเดียว
    let (error_count, warning_count) = entries.iter().fold((0usize, 0usize), |(err, warn), entry| {
        match entry.level {
            LogLevel::Error => (err + 1, warn),
            LogLevel::Warning => (err, warn + 1),
            LogLevel::Info => (err, warn),
        }
    });

    // กรองเฉพาะ Error, ดึงข้อความ, deduplicate ข้อความซ้ำติดกัน (extension trait จากหัวข้อ 26.4), collect
    let unique_error_messages: Vec<String> = entries
        .iter()
        .filter(|entry| entry.level == LogLevel::Error)
        .map(|entry| entry.message.clone())
        .deduplicate()
        .collect();

    // .find() (หัวข้อ 26.2) หา timestamp ของ error ตัวแรก พร้อม short-circuit
    let first_error_timestamp = entries
        .iter()
        .find(|entry| entry.level == LogLevel::Error)
        .map(|entry| entry.timestamp.clone());

    LogSummary {
        total_entries: entries.len(),
        error_count,
        warning_count,
        unique_error_messages,
        first_error_timestamp,
    }
}

// จัดกลุ่ม log ตาม timestamp (เฉพาะส่วนวันที่ ก่อนตัว 'T') ด้วย .fold() (แบบเดียวกับหัวข้อ 26.1)
fn group_by_date(entries: &[LogEntry]) -> HashMap<String, usize> {
    entries.iter().fold(HashMap::new(), |mut acc, entry| {
        let date = entry
            .timestamp
            .split('T')
            .next()
            .unwrap_or(&entry.timestamp)
            .to_string();
        *acc.entry(date).or_insert(0) += 1;
        acc
    })
}

fn main() {
    let raw_lines = vec![
        "2024-01-01T08:00:00 INFO server started",
        "2024-01-01T08:00:05 INFO listening on port 8080",
        "2024-01-01T08:01:10 WARN slow query detected",
        "2024-01-01T08:01:15 ERROR database connection lost",
        "2024-01-01T08:01:15 ERROR database connection lost",
        "2024-01-01T08:01:16 ERROR database connection lost",
        "2024-01-01T08:02:00 INFO database connection restored",
        "2024-01-02T09:00:00 WARN high memory usage",
        "2024-01-02T09:05:00 ERROR out of memory",
        "2024-01-02T09:05:30 ERROR database connection lost",
    ];

    match parse_all(&raw_lines) {
        Ok(entries) => {
            println!("=== parse สำเร็จทั้งหมด {} บรรทัด ===", entries.len());

            let summary = summarize_errors_and_warnings(&entries);
            println!("\n=== สรุปผล ===");
            println!("จำนวนทั้งหมด: {}", summary.total_entries);
            println!("จำนวน ERROR: {}", summary.error_count);
            println!("จำนวน WARN : {}", summary.warning_count);
            println!("ERROR แรกเกิดเวลา: {:?}", summary.first_error_timestamp);
            println!("ข้อความ ERROR ที่ไม่ซ้ำติดกัน (หลัง deduplicate):");
            for msg in &summary.unique_error_messages {
                println!("  - {msg}");
            }

            let by_date = group_by_date(&entries);
            let mut dates: Vec<&String> = by_date.keys().collect();
            dates.sort();
            println!("\n=== จำนวน log แยกตามวัน ===");
            for date in dates {
                println!("{date}: {} รายการ", by_date[date]);
            }
        }
        Err(e) => {
            println!("parse ล้มเหลว: {e}");
        }
    }

    // ทดสอบ short-circuit ของ parse_all ด้วยข้อมูลที่มีบรรทัดผิดพลาดปนอยู่
    println!("\n=== ทดสอบ parse_all กับข้อมูลที่มีบรรทัดผิดพลาด ===");
    let bad_lines = vec![
        "2024-01-01T08:00:00 INFO server started",
        "2024-01-01T08:00:05 CRITICAL something weird", // level ไม่รู้จัก
        "2024-01-01T08:01:10 WARN slow query detected",
    ];
    match parse_all(&bad_lines) {
        Ok(entries) => println!("parse สำเร็จ {} บรรทัด", entries.len()),
        Err(e) => println!("parse ล้มเหลว (คาดไว้แล้ว): {e}"),
    }
}
```

ผลลัพธ์:

```
=== parse สำเร็จทั้งหมด 10 บรรทัด ===

=== สรุปผล ===
จำนวนทั้งหมด: 10
จำนวน ERROR: 5
จำนวน WARN : 2
ERROR แรกเกิดเวลา: Some("2024-01-01T08:01:15")
ข้อความ ERROR ที่ไม่ซ้ำติดกัน (หลัง deduplicate):
  - database connection lost
  - out of memory
  - database connection lost

=== จำนวน log แยกตามวัน ===
2024-01-01: 7 รายการ
2024-01-02: 3 รายการ

=== ทดสอบ parse_all กับข้อมูลที่มีบรรทัดผิดพลาด ===
parse ล้มเหลว (คาดไว้แล้ว): log level ไม่รู้จัก: "CRITICAL"
```

**เชื่อมทุกอย่างที่ใช้ในตัวอย่างนี้กลับไปยังหัวข้อที่สอนไว้**:

- `parse_all()` ใช้ `.map(parse_line).collect::<Result<Vec<_>, _>>()` — หัวข้อ 26.6 (`.collect()` เข้ากับ
  `Result<Vec<T>, E>` พร้อม short-circuit — พิสูจน์ในผลลัพธ์ท้ายโปรแกรมว่าบรรทัด `"CRITICAL"` ทำให้ทั้งฟังก์ชัน
  คืน `Err` ทันที)
- `summarize_errors_and_warnings()` ใช้ `.fold()` นับ error/warning พร้อมกันในการวนครั้งเดียว — หัวข้อ 26.2
- `.filter().map().deduplicate().collect()` ใน `summarize_errors_and_warnings()` — ใช้ extension trait
  `DeduplicateExt` จากหัวข้อ 26.3-26.4 ตรง ๆ ผสมกับ adaptor พื้นฐานจาก Part 25 ในสายเดียวกัน (สังเกตผลลัพธ์:
  `"database connection lost"` ปรากฏ**สองครั้ง**ใน `unique_error_messages` เพราะระหว่างนั้นมี `"out of
  memory"` แทรกอยู่ — deduplicate เอาแค่ตัวที่ซ้ำ**ติดกัน**ออก ไม่ใช่ตัวที่ซ้ำกันทั้งเส้น ตรงตามที่อธิบายไว้ใน
  หัวข้อ 26.3)
- `.find()` หา error ตัวแรก — หัวข้อ 26.2
- `group_by_date()` ใช้ `.fold()` จัดกลุ่มด้วย `HashMap` — หัวข้อ 26.1 (pattern จัดกลุ่มแบบมือเขียนแทน
  `group_by`)

นี่คือภาพรวมของ "โปรแกรม Rust สไตล์ iterator" ตัวจริงที่ใช้ในโปรเจกต์ production — ไม่มี loop มือเปล่าแม้แต่
บรรทัดเดียวในทั้งฟังก์ชัน (ยกเว้น `for date in dates` ตอน print ผลลัพธ์สุดท้ายซึ่งเป็นแค่การแสดงผล ไม่ใช่ตรรกะ
ประมวลผลข้อมูล) ทุกอย่างประกอบขึ้นจาก adaptor เล็ก ๆ ที่ chain ต่อกันเป็นสายเดียว อ่านจากบนลงล่างตรงกับลำดับที่
ข้อมูลถูกแปลงจริง

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. เรียก `.fold()` แล้วลืมคืน accumulator ออกจาก closure — type mismatch

```rust
fn main() {
    let numbers = vec![1, 2, 3];
    let sum = numbers.iter().fold(0, |acc, &n| {
        acc + n;
    });
    println!("{sum}");
}
```

```
error[E0308]: mismatched types
 --> src/main.rs:3:48
  |
3 |       let sum = numbers.iter().fold(0, |acc, &n| {
  |  ________________________________________________^
4 | |         acc + n;
  | |                - help: remove this semicolon to return this value
5 | |     });
  | |_____^ expected integer, found `()`
```

**เหตุผล**: closure ของ `.fold()` **ต้อง**คืน accumulator ชนิดเดียวกับ `B` เสมอ (`|acc: B, item| -> B`) การใส่
`;` ปิดท้าย `acc + n` ทำให้ expression นั้นกลายเป็น**statement** ที่ให้ค่า `()` (unit type — เรียนใน Part 3)
แทนที่จะเป็น expression ที่คืนค่า `i32` ตรง ๆ — closure body ที่ลงท้ายด้วย `;` จะ**ไม่คืนค่าอะไรเลย** (คืน `()`
เสมอ) ต่างจาก expression เปล่า ๆ ที่บรรทัดสุดท้ายไม่มี `;` ซึ่งค่าของมันจะกลายเป็น return value ของ block
ทั้งหมด (กฎเดียวกับ Part 3 ที่สอนความแตกต่างของ statement/expression) `.fold()` คาดหวัง `i32` แต่ได้ `()` จึงเกิด
type mismatch

**วิธีแก้**: ลบ `;` ออก (หรือใส่ `acc + n` เป็นบรรทัดสุดท้ายแบบ expression):

```rust
fn main() {
    let numbers = vec![1, 2, 3];
    let sum = numbers.iter().fold(0, |acc, &n| acc + n);
    println!("{sum}");
}
```

### 2. เขียน default method ที่รับ `self` by-value บน extension trait โดยไม่มี `Self: Sized` — E0277

```rust
trait CountExt: Iterator {
    fn count_twice(self) -> (usize, usize) {
        let v: Vec<_> = self.collect();
        (v.len(), v.len())
    }
}

impl<I: Iterator> CountExt for I {}

fn main() {
    let v = vec![1, 2, 3];
    let (a, b) = v.into_iter().count_twice();
    println!("{a} {b}");
}
```

```
error[E0277]: the size for values of type `Self` cannot be known at compilation time
 --> src/main.rs:2:20
  |
2 |     fn count_twice(self) -> (usize, usize) {
  |                    ^^^^ doesn't have a size known at compile-time
  |
help: consider further restricting `Self`
  |
2 |     fn count_twice(self) -> (usize, usize) where Self: Sized {
  |                                            +++++++++++++++++
help: function arguments must have a statically known size, borrowed types always have a known size
  |
2 |     fn count_twice(&self) -> (usize, usize) {
  |                    +
```

**เหตุผล**: `fn count_twice(self)` รับ `self` แบบ by-value — การรับ by-value ต้องรู้**ขนาดแน่นอน**ของ `Self`
ตอน compile time เพื่อจอง stack space ให้ถูกต้อง (concept `Sized` จาก Part 22) จุดที่น่าประหลาดใจคือ **error
นี้เกิดขึ้นทันทีตอนประกาศ trait เอง** (ชี้ตรงไปที่บรรทัด 2 ของ `trait CountExt`) **โดยไม่ต้องมี
`Box<dyn Iterator>` หรือ trait object เข้ามาเกี่ยวข้องเลยแม้แต่นิดเดียว** — ต่างจาก generic parameter ทั่วไป
บนฟังก์ชันธรรมดา (เช่น `<I: Iterator>` ที่เรียนใน Part 18) ซึ่ง **implicit ผูก bound `Sized` มาด้วยเสมอ** (ถ้า
ไม่เขียน `?Sized` กำกับไว้เอง) `Self` ใน default method ของ trait **ไม่ได้รับ bound `Sized` ให้ฟรีแบบนั้น**
เพราะ trait หนึ่งอาจถูก implement ให้กับ type ที่ไม่ `Sized` ได้เสมอ (ที่พบบ่อยที่สุดคือ `dyn Trait` ซึ่งไม่รู้
ขนาดแน่นอนตายตัว) compiler จึงต้อง**ป้องกันไว้ก่อนตั้งแต่จุดนิยาม method** ว่า "ถ้าใครเอา trait นี้ไป implement
ให้ type ที่ไม่ `Sized` แล้วเรียก method นี้ ก็จะพังทันที" แทนที่จะรอให้ไปถึงจุดใช้งานจริงก่อนค่อยฟ้อง — นี่คือ
เหตุผลเดียวกันเป๊ะที่หัวข้อ 26.4 ต้องเขียน `deduplicate()` ให้มี bound `Self: Sized` ไว้ตั้งแต่ต้น (เพื่อ
บอก compiler ล่วงหน้าว่า "method นี้ใช้ได้แค่กับ `Self` ที่ `Sized` เท่านั้น" ซึ่งเป็นเงื่อนไขที่ตรงกับ
การรับ `self` by-value อยู่แล้ว) `help` ทั้งสองบรรทัดของ error เสนอทางแก้สองทางไว้ให้ตรง ๆ (เพิ่ม
`where Self: Sized` หรือเปลี่ยนไปรับ `&self` แทน)

**วิธีแก้**: เพิ่ม `Self: Sized` ให้ default method ที่รับ `self` by-value เสมอถ้าคาดว่า trait นี้อาจถูก
implement ให้ type ที่ไม่ `Sized` ด้วย (เป็น practice มาตรฐานสำหรับ extension trait ที่สืบทอดจาก `Iterator`):

```rust
trait CountExt: Iterator {
    fn count_twice(self) -> (usize, usize)
    where
        Self: Sized,
    {
        let v: Vec<_> = self.collect();
        (v.len(), v.len())
    }
}

impl<I: Iterator> CountExt for I {}

fn main() {
    let v = vec![1, 2, 3];
    let (a, b) = v.into_iter().count_twice();
    println!("{a} {b}");
    // การเรียกกับ Box<dyn Iterator> ตรง ๆ ยังไม่ได้อยู่ดี (เพราะ Self: Sized กันไว้ตั้งแต่ trait) —
    // ถ้าต้องใช้กับ dyn Iterator จริง ๆ ต้องออกแบบ method ให้รับ &mut self แทน ไม่ใช่ self by-value
}
```

### 3. `.try_fold()` แล้วลืมว่าค่าที่ได้คืนมาเป็น `Result`/`Option` — ใช้ผลลัพธ์ตรง ๆ โดยไม่ unwrap/match

```rust
fn main() {
    let inputs = vec!["1", "2", "3"];
    let total: i32 = inputs.iter().try_fold(0, |acc, s| Ok(acc + s.parse::<i32>()?));
    println!("{total}");
}
```

```
error[E0308]: mismatched types
 --> src/main.rs:3:22
  |
3 |     let total: i32 = inputs.iter().try_fold(0, |acc, s| Ok(acc + s.parse::<i32>()?));
  |                ---   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected `i32`, found `Result<i32, _>`
  |                |
  |                expected due to this
  |
  = note: expected type `i32`
             found enum `Result<i32, _>`
help: consider using `Result::expect` to unwrap the `Result<i32, _>` value, panicking if the value is a `Result::Err`
  |
3 |     let total: i32 = inputs.iter().try_fold(0, |acc, s| Ok(acc + s.parse::<i32>()?)).expect("REASON");
  |                                                                                     +++++++++++++++++
```

**เหตุผล**: `.try_fold()` **เสมอ**คืนค่าเป็น `R` ที่ closure คืน (ในที่นี้ `Result<i32, ParseIntError>` — compiler
แสดงเป็น `Result<i32, _>` เพราะยังไม่จำเป็นต้องระบุ error type ที่แน่ชัดในจุดที่ฟ้อง error) **ไม่ใช่** `i32`
ตรง ๆ — มันคืน "ผลลัพธ์สุดท้ายที่ห่อด้วย `Result`/`Option`" เพราะการวนอาจ short-circuit กลางทางได้เสมอ (ตามที่
หัวข้อ 26.2 อธิบายไว้) การประกาศ type `let total: i32 = ...` บอก compiler ว่าคาดหวัง `i32` ตรง ๆ แต่สิ่งที่
`.try_fold()` คืนจริงคือ `Result<i32, _>` — เกิด type mismatch ทันที (สังเกตว่า compiler เสนอ `help` ให้ใช้
`.expect(...)` ด้วย แต่นั่นเป็นการแก้แบบ panic เมื่อเจอ `Err` ซึ่งมักไม่ใช่ทางที่ต้องการในโค้ด production จริง —
วิธีที่ถูกต้องกว่าคือ match/handle `Result` ให้ครบทั้งสองกรณีตามที่แสดงด้านล่าง)

**วิธีแก้**: ประกาศ type ให้ตรงกับที่ `.try_fold()` คืนจริง แล้ว unwrap/match ตามความเหมาะสม:

```rust
fn main() {
    let inputs = vec!["1", "2", "3"];
    let total: Result<i32, std::num::ParseIntError> =
        inputs.iter().try_fold(0, |acc, s| Ok(acc + s.parse::<i32>()?));
    match total {
        Ok(sum) => println!("รวมได้: {sum}"),
        Err(e) => println!("parse ผิดพลาด: {e}"),
    }
}
```

### 4. `.collect::<Result<Vec<T>, E>>()` แล้ว error type ของแต่ละ item ไม่ตรงกัน — trait bound ไม่ตรง

```rust
fn main() {
    let inputs = vec!["1", "2", "abc"];
    let result: Result<Vec<i32>, String> =
        inputs.iter().map(|s| s.parse::<i32>()).collect();
    println!("{:?}", result);
}
```

```
error[E0277]: a value of type `Result<Vec<i32>, String>` cannot be built from an iterator over elements of type `Result<i32, ParseIntError>`
 --> src/main.rs:4:49
  |
4 |         inputs.iter().map(|s| s.parse::<i32>()).collect();
  |                                                 ^^^^^^^ value of type `Result<Vec<i32>, String>` cannot be built from `std::iter::Iterator<Item=Result<i32, ParseIntError>>`
  |
help: the trait `FromIterator<Result<_, ParseIntError>>` is not implemented for `Result<Vec<i32>, String>`
      but trait `FromIterator<Result<_, String>>` is implemented for it
  = help: for that trait implementation, expected `String`, found `ParseIntError`
note: the method call chain might not have had the expected associated types
 --> src/main.rs:4:23
  |
2 |     let inputs = vec!["1", "2", "abc"];
  |                  --------------------- this expression has type `Vec<&str>`
3 |     let result: Result<Vec<i32>, String> =
4 |         inputs.iter().map(|s| s.parse::<i32>()).collect();
  |                ------ ^^^^^^^^^^^^^^^^^^^^^^^^^ `Iterator::Item` changed to `Result<i32, ParseIntError>` here
  |                |
  |                `Iterator::Item` is `&&str` here
```

**เหตุผล**: `s.parse::<i32>()` คืน `Result<i32, ParseIntError>` เสมอ — error type คือ `ParseIntError` ตรง ๆ
ตาม type จริงของ `i32::from_str` แต่ตัวแปร `result` ประกาศ error type เป็น `String` — `Result<Vec<i32>, String>`
ต้องการ `FromIterator<Result<i32, String>>` ไม่ใช่ `FromIterator<Result<i32, ParseIntError>>` ที่ iterator นี้ให้
มาจริง ๆ ข้อความ error บอกตรงจุดว่า `` expected `String`, found `ParseIntError` `` — `.collect()` เป็น generic
method (Part 25 หัวข้อ 25.12) ที่ต้องหา `FromIterator` implementation ที่ตรงกับทั้ง `Item` ของ iterator
ต้นทางและ target type ที่ประกาศไว้ ถ้า error type ไม่ตรงกัน ก็ไม่มี implementation ที่ใช้ได้

**วิธีแก้**: แปลง error type ให้ตรงกันด้วย `.map_err()` ก่อน `.collect()` (เชื่อม Part 12 เรื่องการแปลง error
type ด้วย `From`/`Into` หรือ `.map_err()` ตรง ๆ):

```rust
fn main() {
    let inputs = vec!["1", "2", "abc"];
    let result: Result<Vec<i32>, String> = inputs
        .iter()
        .map(|s| s.parse::<i32>().map_err(|e| e.to_string()))
        .collect();
    println!("{:?}", result);
}
```

### 5. เขียน `Deduplicate<I>` แล้วลืม bound `I::Item: PartialEq` — E0369 ตอนเทียบค่า

```rust
struct Deduplicate<I: Iterator> {
    inner: I,
    last: Option<I::Item>,
}

impl<I: Iterator> Iterator for Deduplicate<I> {
    type Item = I::Item;

    fn next(&mut self) -> Option<I::Item> {
        loop {
            let item = self.inner.next()?;
            if self.last.as_ref() != Some(&item) {
                self.last = Some(item);
                return Some(item);
            }
        }
    }
}

fn main() {
    let v = vec![1, 1, 2];
    let d = Deduplicate { inner: v.into_iter(), last: None };
    let result: Vec<i32> = d.collect();
    println!("{:?}", result);
}
```

```
error[E0369]: binary operation `!=` cannot be applied to type `Option<&<I as Iterator>::Item>`
  --> src/main.rs:12:35
   |
12 |             if self.last.as_ref() != Some(&item) {
   |                ------------------ ^^ ----------- Option<&<I as Iterator>::Item>
   |                |
   |                Option<&<I as Iterator>::Item>
   |
help: consider further restricting the associated type
   |
 9 |     fn next(&mut self) -> Option<I::Item> where <I as Iterator>::Item: PartialEq {
   |                                           ++++++++++++++++++++++++++++++++++++++
```

**เหตุผล**: `impl<I: Iterator> Iterator for Deduplicate<I>` ไม่มี bound เพิ่มใด ๆ บน `I::Item` เลย — compiler
**ไม่รู้ล่วงหน้า**ว่า `I::Item` เทียบด้วย `!=` ได้ (เพราะ `Iterator` เองไม่ได้บังคับให้ `Item` ต้อง `PartialEq`
เสมอไป — บาง iterator อาจมี `Item` เป็น type ที่เทียบกันไม่ได้เลยด้วยซ้ำ เช่น closure type หรือ file handle) การ
เขียน `self.last.as_ref() != Some(&item)` จึงถูกปฏิเสธเพราะ compiler ไม่รู้ว่า `I::Item` implement
`PartialEq` หรือไม่ — นี่คือเหตุผลที่หัวข้อ 26.3 ต้องเขียน `where I::Item: PartialEq + Clone` ไว้บน `impl
Iterator for Deduplicate<I>` โดยเฉพาะ (ไม่ใช่บน `struct Deduplicate<I>` ตอนประกาศ เพราะ struct เองยังไม่ต้องเทียบ
ค่าอะไร — เก็บแค่ field ธรรมดา — bound นี้จำเป็นเฉพาะตอน implement `Iterator` ที่ต้องเทียบค่าจริง ๆ)

**วิธีแก้**: เพิ่ม `where I::Item: PartialEq + Clone` บน `impl Iterator for Deduplicate<I>` ตามที่หัวข้อ 26.3
แสดงไว้:

```rust
struct Deduplicate<I: Iterator> {
    inner: I,
    last: Option<I::Item>,
}

impl<I: Iterator> Iterator for Deduplicate<I>
where
    I::Item: PartialEq + Clone,
{
    type Item = I::Item;

    fn next(&mut self) -> Option<I::Item> {
        loop {
            let item = self.inner.next()?;
            if self.last.as_ref() != Some(&item) {
                self.last = Some(item.clone());
                return Some(item);
            }
        }
    }
}

fn main() {
    let v = vec![1, 1, 2];
    let d = Deduplicate { inner: v.into_iter(), last: None };
    let result: Vec<i32> = d.collect();
    println!("{:?}", result);
}
```

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** เขียนฟังก์ชัน `fn running_max(numbers: &[i32]) -> Vec<i32>` ที่ใช้ `.scan()` (หัวข้อ 26.1) คืน
   `Vec<i32>` ของ "ค่าสูงสุดสะสม ณ จุดนั้น" — ทดสอบด้วย `running_max(&[3, 1, 4, 1, 5, 9, 2, 6])` ควรได้
   `[3, 3, 4, 4, 5, 9, 9, 9]` (hint: state ของ `.scan()` คือ "ค่าสูงสุดที่เจอมาแล้วจนถึงตอนนี้" อัปเดตด้วย
   `*state = (*state).max(n)` แล้วคืน `Some(*state)` ทุกรอบ)

2. **[กลาง]** เขียนฟังก์ชัน `fn first_valid_config(raw: &[&str]) -> Option<i32>` ที่ใช้ `.find_map()` (หัวข้อ
   26.2) หาค่าตัวเลขแรกใน `raw` ที่ parse เป็น `i32` ได้**และ**มีค่ามากกว่า 0 เท่านั้น (ตัวที่ parse ไม่ผ่าน หรือ
   parse ผ่านแต่เป็น 0 หรือค่าลบ ให้ข้ามไป) ทดสอบด้วย `first_valid_config(&["abc", "-5", "0", "42", "99"])`
   ควรได้ `Some(42)` (hint: closure ของ `.find_map()` ต้องคืน `Option<i32>` — ใช้ `.parse::<i32>().ok()` แล้ว
   `.filter(|&n| n > 0)` บน `Option` นั้นต่อ (เชื่อม Part 11 ที่สอน `.filter()` บน `Option<T>` ไว้) ก่อนคืนออกจาก
   closure)

3. **[ยาก]** เขียน custom iterator adaptor ชื่อ `RunLengthEncode<I>` ที่รับ iterator ของค่าที่เทียบกันได้
   (`I::Item: PartialEq + Clone`) แล้วคืน iterator ของ `(I::Item, usize)` — จับกลุ่มค่าที่ซ้ำติดกันพร้อมนับ
   จำนวน (คล้าย run-length encoding) จากนั้นเขียน extension trait `RunLengthEncodeExt` ให้เรียกเป็น
   `.run_length_encode()` ได้ ทดสอบด้วย `vec!['a','a','a','b','b','a'].into_iter().run_length_encode()`
   ควรได้ `[('a', 3), ('b', 2), ('a', 1)]` (hint: โครงสร้างคล้าย `Deduplicate<I>` มาก แต่ `next()` ต้องวน "กิน"
   ค่าที่เหมือนกันติดกันทั้งหมดในการเรียกครั้งเดียว ไม่ใช่กินทีละตัวแบบ `Deduplicate` — ต้องใช้เทคนิคคล้าย
   `.peekable()` เก็บค่าที่ "เผลอกิน" มาแล้วแต่ยังไม่ได้ใช้ในรอบถัดไป หรือเก็บ field พิเศษไว้ "buffer" ค่าที่ดึง
   มาเกิน)

4. **[ยาก/ประยุกต์ใช้งานจริง]** ขยายตัวอย่างระบบประมวลผล log ในหัวข้อ 26.10: เขียนฟังก์ชัน
   `fn detect_error_bursts(entries: &[LogEntry], threshold: usize) -> Vec<String>` ที่ใช้ `.windows()` (หัวข้อ
   26.1 — ต้องแปลง `entries` เป็น `Vec` ของ level ก่อน หรือทำงานกับ `entries` ตรง ๆ ผ่าน slice) ตรวจหา
   "ช่วงที่มี ERROR ติดกันอย่างน้อย `threshold` รายการ" แล้วคืน `Vec<String>` ของ timestamp ที่ ERROR burst
   นั้นเริ่มต้น (ใช้ `.enumerate()` ร่วมกับ `.windows(threshold)` เพื่อเช็คว่าทุกตัวใน window เป็น
   `LogLevel::Error` หมดหรือไม่ ด้วย `.all()` จากหัวข้อ 26.2 แล้ว `.map()` ดึง timestamp ของตัวแรกใน window
   นั้นออกมา, สุดท้าย `.collect()`) ทดสอบกับข้อมูลจากหัวข้อ 26.10 ที่มี ERROR สามรายการติดกัน โดยตั้ง
   `threshold = 3` ควรเจอ burst หนึ่งช่วงที่เริ่มต้นด้วย timestamp ของ ERROR ตัวแรกในกลุ่มนั้น (hint: ระวังว่า
   `entries.windows(3)` ต้องเรียกบน `&[LogEntry]` ตรง ๆ ไม่ใช่บน iterator ที่ผ่าน `.filter()` มาแล้ว เพราะ
   `.windows()` เป็น method ของ slice ตามที่อธิบายไว้ในหัวข้อ 26.1 — ต้องดึง index ของ window ที่ผ่านเงื่อนไข
   มาก่อน แล้วค่อย index กลับเข้า `entries` เดิมเพื่อดึง timestamp)

## สรุป

บทนี้ต่อยอดจาก Part 25 เข้าสู่ iterator ขั้นสูงที่โค้ด Rust production จริงใช้กันอย่างหนัก เริ่มจากทัวร์ adaptor
ที่กว้างขึ้น — `.flat_map()`/`.flatten()` แบนข้อมูลที่ซ้อนกันเป็นชั้น, `.scan()` ให้ closure มี "ความจำ" ระหว่าง
item ต่างจาก `.map()`, `.step_by()` ข้าม item เป็นช่วง, `.peekable()`/`.peek()` มองล่วงหน้าโดยไม่กิน item,
`.windows()`/`.chunks()` (method ของ slice ที่คืน iterator เชื่อมกับ Part 8/13), `.partition()` แยกสองกลุ่มใน
การวนครั้งเดียว, การจัดกลุ่มแบบมือเขียนด้วย `.fold()` (เพราะ `group_by` ไม่มีใน std), และ `.inspect()` สำหรับ
debug pipeline โดยไม่แก้ผลลัพธ์

ครึ่งกลางของบทเจาะลึก `.fold()` ในฐานะ **แกนกลางของ consuming adaptor ทั้งหมด** พิสูจน์ว่า `.sum()`,
`.count()`, `.max()` เขียนใหม่ด้วย `.fold()` ได้ทั้งหมด ต่อด้วย `.reduce()` (เหมือน `.fold()` แต่ไม่ต้องมีค่า
เริ่มต้น), `.try_fold()`/`.try_for_each()` (short-circuit บน `Result`/`Option` เชื่อม Part 12), `.all()`/
`.any()`, และ `.position()`/`.find()`/`.find_map()` — ทั้งหมดพิสูจน์ short-circuit ด้วย `.inspect()` ให้เห็น
จริงว่า item ที่เหลือไม่ถูกแตะเลยหลังเจอเงื่อนไขที่ต้องการ

จากนั้นเรียนรู้ทักษะที่สำคัญที่สุดของบท — **เขียน iterator adaptor ของตัวเอง** ที่ห่อ iterator ตัวอื่นไว้ข้างใน
(`Deduplicate<I>` ข้ามค่าซ้ำติดกัน, `Batched<I>` รวมเป็นชุดละ N) ต่างจาก `Countdown` ของ Part 25 ที่สร้างค่าเอง
จากศูนย์ แล้วเรียนรู้ **extension trait pattern** เพิ่ม method `.deduplicate()`/`.batched()` ให้ทุก iterator
ในโลกผ่าน blanket implementation — กลไกเดียวกันเป๊ะที่ crate `itertools` ใช้เพิ่ม method ของตัวเองให้ทุก
iterator โดยไม่แก้ std เลย

บทนี้ปิดท้ายด้วยสามเรื่องสำคัญ: (1) `.collect()` เข้ากับ `Result<Vec<T>, E>`/`Option<Vec<T>>` ที่ short-circuit
หยุดทันทีเจอ `Err`/`None` ตัวแรก — pattern ที่มีประโยชน์มากสำหรับ validate ข้อมูลทั้ง batch, (2) performance
deep dive พิสูจน์ **iterator fusion** ว่า adaptor chain ยาว ๆ compile ออกมาเป็น loop เดียวไม่มีต้นทุนเพิ่มจาก
abstraction เลย ผ่าน static dispatch + monomorphization (เชื่อม Part 18) + aggressive inlining พร้อมข้อ
แลกเปลี่ยนจริงของ `Box<dyn Iterator>` (เชื่อม Part 21) ที่เสีย fusion ไปส่วนหนึ่งเพื่อความยืดหยุ่นด้าน type,
และ (3) การรู้จัก `rayon` ในระดับความตระหนัก — mental model เดียวกันของ adaptor chain ขยายไปสู่ parallel
iteration ได้โดยไม่ต้องเรียนรู้ API ใหม่ทั้งหมด ปิดท้ายด้วยตัวอย่างระบบประมวลผล log ที่ผสมทุกเทคนิคของบท
(custom adaptor + extension trait + `.fold()`/`.find()` + `.collect()` เป็น `Result`) เข้าด้วยกันในโปรแกรมเดียว
ที่ทำงานได้จริง

การเดินทางของ iterator arc จาก Part 25 ถึง Part 26 จบลงตรงนี้ — ตอนนี้คุณมีทักษะครบทั้งการ**ใช้** adaptor ที่มี
อยู่แล้วอย่างคล่องแคล่ว และการ**สร้าง** adaptor ใหม่ของตัวเองเมื่อไม่มีตัวสำเร็จรูปให้ ต่อจากนี้ไปใน **Module 2**
หลักสูตรจะเปลี่ยนโฟกัสไปที่ **Smart Pointers** เริ่มจาก **Part 27: `Box<T>`** — ชนิดข้อมูลที่ทำให้ค่าถูกเก็บบน
heap แทน stack ได้ ซึ่งเป็นกลไกที่บทนี้เพิ่งใช้ไปเองแล้วในหัวข้อ 26.7 ตอนเขียน `Box<dyn Iterator<Item = T>>` —
Part 27 จะอธิบาย `Box<T>` อย่างเป็นทางการตั้งแต่ต้น ทำไมภาษาต้องมีมันอยู่ (recursive type ที่ไม่รู้ขนาดตาย
ตัว, trait object ที่ต้องใช้ pointer เสมอ), และวางรากฐานให้กับ `Rc<T>`/`RefCell<T>` ใน Part 28 ต่อไป

---

**Part ก่อนหน้า:** [Iterators เบื้องต้น](part-025-iterators-basics.md) | **Part ถัดไป:** [Smart Pointers: Box<T>](part-027-smart-pointers-box.md)
